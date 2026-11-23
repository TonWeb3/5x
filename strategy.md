# Strategy: Polymarket BTC "Up or Down" 5-minute — Latency Arbitrage

This document describes how the bot decides to trade. It trades a binary option:
*"Will BTC be higher at the end of this 5-minute window than at the window's open?"*
The winning side pays **$1.00** per share, the loser $0.

Series **10684** (`btc-up-or-down-5m`). Windows are aligned to exact 5-minute
boundaries, so the local window clock and the market's `endDate` agree.

---

## 0. The idea

A historical study found that the obvious indicator model (Monte Carlo + RSI/MACD/HA)
had **no predictive edge** over a trivial baseline — *"is spot already above the
window's open right now?"* Whoever is winning at the checkpoint usually wins, and that
persistence is fully visible to, and priced by, the market.

The only edge left is **latency**: acting on a Binance spot move *before* Polymarket's
thin book reprices it. The strategy is built around that one idea.

---

## 1. The pipeline

Every 500 ms ([`update_loop`](main.py)):

```
Binance spot ─► fair_prob_up ─► EV vs Polymarket price ─► decide_ev ─► size (%/fixed) ─► execute
(fast feed)     (closed-form    (edge = fair − ask;       (EV + min-    (% or flat $  (FOK,
                GBM, continuous  book stale?)              prob + time)  of balance)   paper/live)
                time decay)
```

A trade fires only when the model's **fast** fair probability beats the market's
(possibly stale) price by a margin.

---

## 2. Fair probability — closed-form, continuous time
[`fair_prob_up`](bot/indicators.py). The probability that price closes above the
window's open is a closed-form GBM expression:

```
sd        = σ · √t
z         = (ln(strike / spot) − (μ − ½σ²)·t) / sd
prob_up   = ½·erfc(z / √2)            # = 1 − Φ(z)
```

where **`t` is the remaining time in fractional minutes** and `μ, σ` are the realized
per-**minute** log-return drift and vol, measured from **1-minute** candles over a
300-candle (5-hour) lookback ([`realized_drift_vol`](bot/indicators.py)).

**Time is continuous — this is the heart of the 5m port.** The 15-minute version
quantized the horizon into whole 5-minute steps (`n = ceil(minutes_left / 5)`). On a
5-minute market that expression is *always 1*, whether 4:59 or 0:05 remains: the model
would price a full five minutes of volatility into a position with seconds left, never
gain conviction, and the latency edge would never appear. Scaling by `√t` instead makes
conviction tighten smoothly as the window drains, which is exactly what a 5m market
needs.

The model is, in essence, *"is spot above the open, given the volatility still to come?"*

**Strike:** the model uses the **Binance** 5m candle open (which shares the market's
boundary, so it *is* the window's open). Measuring the Binance move against the Binance
open cancels the Binance/Chainlink feed offset. Settlement is scored separately against
the **Chainlink** open (§7).

---

## 3. The edge — fair vs the market
[`update_loop`](main.py). Polymarket's prices are normalized to an implied
probability, then:

```
edge_up = fair_up − market_implied_up
```

A positive edge means the book hasn't yet repriced the move our fast feed already
sees. **That gap is the entire thesis.** The book is re-fetched every tick (the market
*identity* is cached for 10s — it only changes once per window — but the prices and
order books are not).

### The price we compare against is the ASK
Entry prices are read as the **best ask** straight off the order book — the price we
actually *pay*.

This was previously sourced from CLOB `/price?side=buy`, which despite the name returns
the best **bid** (verified live: `side=buy` → 0.84 while the book's best ask was 0.85).
EV was therefore computed against the price we'd *sell* at, overstating every edge by the
full spread. On a $1 contract a 1¢ spread is a quarter of the 4¢ EV threshold — enough to
turn a real edge into a leak, and far less forgiving at 5m than at 15m. Exits and
mark-to-market use the **bid** (what we'd receive), which was already correct.

### Tick cadence is the real constraint
This is a latency strategy, so the tick period *is* the strategy. Measured end-to-end it
is now **~0.5s** (median) — the configured `POLL_INTERVAL_MS` floor — down from ~5.6s with
occasional 70s stalls. That is ~594 looks at the book per 5-minute window, an 11× gain,
reached in three steps:

1. **Pooled HTTP** — every request used to build a fresh client and pay a TLS handshake.
2. **CLOB book over WebSocket** — the order books are now *pushed* ([`ClobBookStream`](bot/ws_data.py)),
   not polled. Polling `/book` was the dominant per-tick cost. A full snapshot plus level
   deltas is maintained in memory and read at zero network cost; REST `/book` remains only
   as a fallback while the socket (re)connects or the market rolls.
3. **Chainlink off the critical path** — its RPC round-trip is seconds and its WS only
   emits on `AnswerUpdated` (minutes apart), so it was silently dragging every tick to the
   slow fallback. It is now refreshed in its own 2s loop; the trading tick just reads the
   last value. It only marks the open and settles trades, neither of which needs to be fast.

---

## 4. Entry decision — Underdog Reversal (Strategies A + D + C)
[`decide_underdog_reversal`](bot/engines.py). Targets extreme underdog mispricings
(shares priced **$\le$ `MAX_UNDERDOG_PRICE`**, default **7¢** / $0.07) offering asymmetric
14:1+ payout potential on mean-reversion.

A trade is taken only if **all** pass:

1. **Underdog Price Gate:** Ask price $\le$ `MAX_UNDERDOG_PRICE` (default 0.07 / 7¢).
2. **Strategy D (Volatility Cone):** Distance to strike $Z = \frac{|S - K|}{S \cdot \sigma_{1m} \cdot \sqrt{t}} \le$ `MAX_SIGMA_DISTANCE` (default 2.0$\sigma$). Ensures the strike is statistically reachable within normal volatility before expiry.
3. **Strategy A (Model Dislocation & EV):**
   - Fair continuous-time probability (GBM) $\ge$ `MIN_UNDERDOG_PROB` (default 0.10 / 10%).
   - Expected Value $\text{EV} = \text{fair} - \text{VWAP} \ge$ `MIN_UNDERDOG_EV` (default 0.02 / 2¢).
4. **Strategy C (Binance Momentum Reversal Hook):**
   - For UP underdog: Binance spot momentum must be positive ($>$ 0), confirming price has bounced.
   - For DOWN underdog: Binance spot momentum must be negative ($<$ 0), confirming price has dropped.
   - Prevents catching falling knives before the reversal starts.

### Strategy B: Convexity Scalper / Early Exit (TP/SL)
When `TP_SL_ENABLED` is true:
- **Take-Profit (`TAKE_PROFIT_CENTS`):** Denominated in **¢ per share** (default 25¢). If the contract reprices from 6¢ to 25¢ on the spot bounce, the bot sells immediately to bank a **~4× (+316%) profit** without waiting for full expiry resolution.
- **Stop-Loss (`STOP_LOSS_PCT`):** Denominated in **% of the stake risked** (default 50%). If the contract loses 50% of the initial stake, it sells to cut loss.

---

## 5. Sizing — always the configured risk per trade
[`execute_trade`](main.py). The stake (dollars put at risk) is set by `RISK_TYPE`:
**`percent`** stakes `RISK_VALUE`% of the current balance; **`fixed`** stakes a flat
`RISK_VALUE` dollars. The stake is **exactly that** — never trimmed to fit the book and
never scaled up. If the ask side cannot absorb the whole stake, the trade is *skipped*
rather than taken at a smaller size (`thin_book`).

### The fill is priced by walking the book
The entry price is the blended **VWAP** of walking the ask levels to fill the full stake
([`walk_asks`](bot/data.py)), not the best ask. Taking `stake / best_ask` assumes the
entire order clears at the top of the book; on a thin 5m book that overstates both the
share count and the edge, and it does so *worst* on the trades that look most attractive.
The **EV gate in §4 runs on this same VWAP** — an "edge" that exists only for the first
few shares at the top of the book is not an edge for the size actually traded. Exits and
mark-to-market symmetrically walk the **bids** ([`walk_bids`](bot/data.py)) for what a
sale would really return.

---

## 6. Execution
[`execute_trade`](main.py). One position at a time. **Paper:** debits the simulated
balance. **Live:** a slippage-capped marketable Fill-Or-Kill BUY on the CLOB
(limit = quote + `CLOB_MAX_SLIPPAGE`). Optional close-and-flip on a strong opposite
signal ([`maybe_flip_position`](main.py), `FLIP_ENABLED`, default off).

### Stale-feed safety
Every entry requires a **provably fresh** Binance spot price (age ≤ `MAX_SPOT_AGE_S`,
default 5s). A silently-stalled socket — connection alive, messages stopped — freezes the
spot price; the market keeps moving, so the gap between "fair" and the book grows without
bound and *a dead feed looks exactly like a huge latency edge*. Rather than fire into that
phantom edge at maximum confidence, entries halt until a fresh price arrives (the dashboard
shows a red "Spot stale" banner). Deliberately, we do **not** substitute the Chainlink /
Polymarket price here — that is the slow feed the book is already priced off, so comparing
it to the book measures nothing. Open positions still settle normally while spot is stale.

---

## 7. Exits & resolution
[`update_trades`](main.py). Positions are held to the 5-minute expiry and resolved
robustly: authoritative Polymarket settlement first, settlement-price-vs-strike
fallback once the window is over, and a 2-minute grace before voiding (paper stake
refunded) so a trade can never stay stuck open across two consecutive windows.

Settlement mirrors Polymarket: the window's **Chainlink open** is snapshotted at the
boundary (within 10s, or the window is simply not traded), and the **Chainlink close**
is frozen the instant the window ends. Open and close come from the same feed, so no
cross-feed offset can flip a near-the-money result.

---

## 8. Low-Latency Execution Engine (Python-Rust Native Pipeline)

In 5-minute binary options, trade execution latency is critical. Standard Python Web3 and `eth_account` libraries incur 50ms–150ms of overhead constructing EIP-712 structured data and calculating secp256k1 elliptic curve signatures.

To eliminate this bottleneck, the bot incorporates the native **`rust_clob`** PyO3 extension from high-frequency market-making architecture:

1. **Native EIP-712 & Solady ECDSA Signing (`k256`):**
   - Order hash computation and recoverable secp256k1 signing execute in C/Assembly-speed Rust in **~40–160 microseconds (0.04ms–0.16ms)**, over 500× faster than pure Python.
   - Pre-computes and caches EIP-712 domain separators and type hashes for standard and neg-risk exchanges.
2. **Persistent HTTP Keep-Alive Connection Pool (`ureq`):**
   - Submits orders over a warmed keep-alive HTTP/1.1 connection pool directly in Rust (`max_idle_connections: 20`), avoiding TLS re-handshakes and HTTP/2 framing overhead.
   - Computes L2 HMAC credentials (`POLY_SIGNATURE`) in Rust.
3. **GIL-Released Network I/O (`py.allow_threads`):**
   - The Python Global Interpreter Lock is released during cryptographic signing and HTTP POST I/O, preventing thread contention on the asyncio event loop.
4. **Pre-Warmed Token Metadata (`warm_token`):**
   - Pre-populates token tick sizes, fee structures, and exchange configurations in memory, eliminating preliminary GET requests before order submission.
5. **C-Accelerated Fallback (`coincurve`):**
   - If Python fallback is triggered, `coincurve` (libsecp256k1) monkey-patches `Account.unsafe_sign_hash` for sub-millisecond fallback execution.

---

## 9. Logging

[`logs/signals.csv`](logs/) records one row per tick — the fair/market probabilities,
the edge, and the exact decision/blocking reason — so you can see what the bot did and
why. Run in **paper mode** first and confirm EV-positive trades actually appear against
the real book before risking capital; if they don't, the book isn't slow and the
latency edge isn't there.

---

## 10. Tunable parameters

| Setting | Default | Meaning |
|---------|---------|---------|
| `CANDLE_WINDOW_MINUTES` | 5 | Market window length. |
| `POLL_INTERVAL_MS` | 500 | How fast we see the book move (bounds the capturable staleness). |
| `MAX_UNDERDOG_PRICE` | 0.07 | Max ask price to consider (7¢ / $0.07). |
| `MIN_UNDERDOG_PROB` | 0.10 | Min model fair probability to enter (10%). |
| `MIN_UNDERDOG_EV` | 0.02 | Min expected value (fair − ask) to enter (2¢). |
| `MAX_SIGMA_DISTANCE` | 2.0 | Max Z-score distance to strike (Strategy D Volatility Cone). |
| `MOMENTUM_FILTER_ENABLED` | true | Require Binance spot momentum confirmation (Strategy C). |
| `TAKE_PROFIT_CENTS` | 25 | Scalp exit target in cents per share (Strategy B, 25¢). |
| `STOP_LOSS_PCT` | 50 | Stop-loss % of stake risked. |
| `MAX_SPOT_AGE_S` | 5 | Halt entries if the Binance spot price is older than this. |
| `VOL_LOOKBACK_1M` | 300 | 1m candles used for realized per-minute drift/vol. |
| `RISK_TYPE` / `RISK_VALUE` | percent / 10 | Stake sizing: % of balance, or flat $ (exact, never trimmed). |
| `MIN_BOOK_LIQUIDITY_USD` | 20.0 | Skip if the ask side can't absorb the full stake. |
| `CLOB_MAX_SLIPPAGE` | 0.02 | Live-order limit buffer. |

---

*This is not financial advice. The edge is unproven — the latency thesis is unverified
until measured live. A 5-minute window gives the book less time to be wrong and gives
you less time to be filled; it is a harder game than the 15m, not an easier one. Use at
your own risk; live mode trades real funds.*
