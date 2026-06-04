# 03 — Order Types & Order Book Anatomy

## The Order Book

Foundation of all exchange-traded markets. An order book is a list of all outstanding buy and sell orders, sorted by price.

```
ORDER BOOK — BTC/USDC

ASKS (sell orders) — ascending price
Price      Size       Total
47,250     0.50       0.50
47,200     1.20       1.70
47,150     0.80       2.50
47,100     2.10       4.60

── SPREAD ── (47,100 ask - 47,050 bid = $50)

47,050     1.50       1.50  ← Best Bid
47,000     3.20       4.70
46,950     0.90       5.60
46,900     4.10       9.70

BIDS (buy orders) — descending price
```

### Key Concepts

**Best Bid**: highest price anyone is willing to pay right now  
**Best Ask (Offer)**: lowest price anyone is willing to sell right now  
**Spread**: ask - bid. The cost of immediacy. Market makers earn this.  
**Mid Price**: (bid + ask) / 2. Reference price.  
**Depth**: total volume available at each price level  

```
Spread % = (ask - bid) / mid × 100
BTC spread of $50 at $47,075 mid = 0.106%
```

Tight spread = liquid market. Wide spread = illiquid market.

### Level 1 vs Level 2 vs Level 3

| Level | Data | What you see |
|---|---|---|
| L1 | Best bid/ask only | NBBO: top of book |
| L2 | All price levels and sizes | Full depth, anonymized |
| L3 | Individual orders per level | Each order with ID (rare, mostly HFT direct) |

Most exchanges provide L2 via WebSocket. L3 data gives you flow toxicity signals.

---

## Order Types

### 1. Market Order

Execute immediately at best available price. Guaranteed fill, not guaranteed price.

```
BTC order book:
  Ask: 47,100 (2.10 BTC), 47,150 (0.80), 47,200 (1.20)

Market Buy 3 BTC:
  2.10 BTC filled @ 47,100
  0.80 BTC filled @ 47,150
  0.10 BTC filled @ 47,200
  VWAP: (2.10×47100 + 0.80×47150 + 0.10×47200) / 3 = $47,120
```

Market order "walks the book" — eats through levels until filled. On thin books, can cause massive slippage.

**When to use**: when you need immediate execution regardless of price. Speed > price precision.

**For HFT**: market orders almost never used. Market maker can't use them — you'd pay the spread every time.

### 2. Limit Order

Execute at specified price or better. Guaranteed price, not guaranteed fill.

```
Limit Buy BTC @ 47,000:
  - If ask ≤ 47,000: fills immediately (limit acts like market)
  - If ask > 47,000: rests in book as passive order, waits
  - Never pays more than 47,000
```

Limit orders REST in the book = provide liquidity = earn the spread (or pay no spread).

**Maker vs Taker**:
- **Maker**: you place a limit order that rests → you "make" liquidity. Usually get rebate or lower fee.
- **Taker**: you place order that fills immediately → you "take" liquidity. Pay higher fee.

**Fee example (Binance BTC/USDC)**:
```
Spot:
  Taker fee: 0.10%
  Maker fee: 0.08% (or rebate)
  
VIP tier 9 (market maker):
  Taker fee: 0.015%
  Maker fee: -0.005% (get paid to make markets)
```

### 3. Stop Order (Stop-Loss)

Becomes a market order when trigger price is reached. Used to limit losses.

```
You hold 1 BTC long @ 47,000. Set stop @ 46,000.
Price drops to 46,000 → stop triggers → market sell fires
→ fills at best available price (might be 45,950 in fast market = slippage)
```

**Stop order risk**: in fast markets, stop triggers but fills much worse. Called **slippage on stop**. Common in crypto flash crashes.

### 4. Stop-Limit Order

Like stop, but triggers a **limit** order instead of market order.

```
Stop-Limit: stop @ 46,000, limit @ 45,800
Price drops to 46,000 → limit sell @ 45,800 placed
If price gaps below 45,800 → order does NOT fill (protection from slippage)
Risk: might not fill at all → position stays open
```

**When to use**: when you want to limit slippage on stops, but accept risk of non-fill.

### 5. Post-Only Order

Limit order that cancels if it would execute immediately (i.e., guarantees maker status).

```
Post-only Limit Buy @ 47,050 (current ask = 47,100):
  47,050 < 47,100 → would rest in book → accepted as maker ✓
  
Post-only Limit Buy @ 47,150 (current ask = 47,100):
  47,150 > 47,100 → would cross immediately → CANCELLED ✗
```

Market makers use this religiously. Never accidentally pay taker fees.

### 6. Immediate-or-Cancel (IOC)

Execute what you can right now, cancel the rest.

```
IOC Buy 5 BTC:
  Ask: 47,100 (2.10), 47,150 (0.80)
  Available at market: 2.90 BTC
  Result: 2.90 BTC filled, remaining 2.10 BTC CANCELLED
```

Used when you want to sweep available liquidity without resting an order in the book.

### 7. Fill-or-Kill (FOK)

Execute the entire order immediately or cancel entirely. No partial fills.

```
FOK Buy 5 BTC @ 47,200:
  Available through 47,200: 4.60 BTC (< 5 BTC needed)
  Result: ENTIRE order cancelled — zero fills
```

Used for large block trades where partial fill changes your strategy.

### 8. Good-Till-Cancelled (GTC)

Order rests in book until filled or you cancel it. Default behavior on most exchanges.

### 9. Good-Till-Date (GTD)

Like GTC but with expiry timestamp.

### 10. Iceberg / Hidden Order

Shows only a small visible quantity. When visible portion fills, next chunk becomes visible.

```
Iceberg Sell 100 BTC, visible=5 BTC:
  Book shows: 47,100 | 5.00 BTC
  Buyer buys 5 BTC → next 5 BTC appears → repeat until 100 BTC sold
```

Institutions use icebergs to hide order size and reduce market impact. Detecting iceberg orders is a quant signal.

### 11. Trailing Stop

Stop price trails market price by fixed amount or percentage.

```
Trailing Stop: current price 47,000, trail = 500 USD
  Price rises to 48,000 → stop trails to 47,500
  Price rises to 50,000 → stop trails to 49,500
  Price drops to 49,500 → triggers → market sell
```

Locks in profits while letting winners run.

### 12. TWAP Order (Time-Weighted Average Price)

Algorithm that splits large order into slices over time, targeting TWAP benchmark.

```
Buy 100 BTC over 1 hour → roughly equal slices every minute → ~1.67 BTC/min
```

Reduces market impact. Used by institutions for large positions.

### 13. VWAP Order (Volume-Weighted Average Price)

Like TWAP but sized proportionally to historical volume profile (trade more when market trades more).

### 14. Reduce-Only Order

Can only reduce, not increase, your position. Common in futures/perps.

```
You're long 2 BTC. Reduce-only sell 1 BTC:
  Fills normally, closes 1 BTC of long
  
You're long 2 BTC. Non-reduce sell 3 BTC:
  If reduce-only set: REJECTED (would flip to short)
```

Used to ensure stops don't accidentally flip your position direction.

---

## Crypto-Specific Order Types

### Conditional Orders
Trigger order when specific condition met (price, time, indicator). Not available on all exchanges.

### Bracket Order
Entry + profit target + stop-loss in one order group. Common in retail platforms.

### TWAP/VWAP Execution Algorithms
Available on Binance, OKX as built-in algos. Same concept as institutional equities.

### Perpetuals Specific: Funding-Aware Orders
Some advanced platforms let you set orders that consider funding rate in execution decision.

---

## Order Flow: Full Lifecycle

```
1. ORDER CREATED
   Client submits order via API/WebSocket
   
2. VALIDATION
   Exchange checks: valid price, sufficient balance, rate limits, position limits
   
3. ORDER BOOK INSERTION (limit) or IMMEDIATE MATCH (market)
   
4. MATCHING
   Matching engine checks if order can fill against resting orders
   
5. FILL
   Partial fill: order remains with reduced quantity
   Full fill: order removed from book
   No fill: order rests (limit) or cancelled (market/IOC/FOK)
   
6. NOTIFICATION
   Exchange sends fill report via WebSocket
   
7. POSITION UPDATE
   Your balance/position updated
```

Latency breakdown (CEX HFT):
```
Client → Exchange API gateway:  0.1-50ms (depends on co-location)
API gateway → matching engine:  0.01-1ms (internal)
Matching engine → response:     0.01-0.1ms
Response → client:              same as step 1
Total round-trip:               1-100ms typical, <1ms co-located
```

---

## Order Book Dynamics (What to Watch)

### Spread Compression/Expansion
- Tight spread → high liquidity, low volatility expected
- Widening spread → uncertainty, volatility incoming, or market maker pulling quotes

### Book Imbalance
```
Bid depth at top N levels / Ask depth at top N levels

Imbalance = (bid_vol - ask_vol) / (bid_vol + ask_vol)
Range: [-1, +1]

High positive → strong buying pressure → price likely to rise
High negative → strong selling pressure → price likely to fall
```

This is a real-time signal. Predictive over 1-10 second horizon.

### Order Book Spoofing
Place large order to create false impression of support/resistance → cancel before it fills → fool algorithmic traders.

Illegal in equities, common in crypto. Detecting spoof orders is a quant signal.

Signs: large orders appear/disappear within milliseconds, price doesn't move when they appear.

### Quote Stuffing
Flood exchange with orders and cancellations to slow down competitors' systems. Used in equity HFT (controversial). Rare in crypto due to rate limits.

---

## Bid-Ask Spread Decomposition

Spread has two components:

**1. Inventory cost**: market maker holds unwanted inventory. Wider spread = compensation for this risk.

**2. Adverse selection**: some traders have better information. Spread protects market maker from informed flow.

```
Total spread = inventory component + adverse selection component

In efficient markets:
  Random order → MM expects zero profit
  Informed order → MM expects to lose
  
MM sets spread wide enough so: profits from uninformed > losses from informed
```

**Order flow toxicity**: proportion of informed vs uninformed orders. High toxicity = MM pulls quotes or widens spread.

**PIN (Probability of Informed Trading)**: classic measure of adverse selection. Estimated from trade initiation patterns.

---

## Crypto vs Equity Order Books

| Feature | Crypto CEX | US Equities |
|---|---|---|
| Maker-taker model | Yes | Usually |
| Hidden orders | Exchange-specific | Yes (dark pools) |
| Lot sizes | Usually 0.001-1 BTC | Usually 1 share |
| Min price increment (tick) | Often fractional | $0.01 |
| Market hours | 24/7/365 | 9:30-16:00 ET |
| Settlement | Immediate (within exchange) | T+1 |
| Order book depth | Public | Public (NMS) |
| Venue fragmentation | Extreme (50+ exchanges) | Extreme (NYSE, NASDAQ, CBOE, dark pools) |

---

## Next

Read [04_market_microstructure.md](04_market_microstructure.md) for how prices form, market maker economics, and measuring market quality.
