# 04 — Market Microstructure

Market microstructure = study of how prices form, how trades happen, and the mechanics of exchange.

This is the most important theory for building a realistic simulator.

---

## Price Discovery

**Price discovery** = the process by which information gets incorporated into prices.

When new information arrives (earnings, news, hack, regulatory decision), it doesn't instantly reflect in price. Informed traders trade against it, moving price toward "fundamental value." This takes time — milliseconds to minutes depending on market.

### Price Discovery Across Venues

In crypto, BTC trades on 50+ exchanges. Which one "leads" price discovery?

```
Typical flow:
  News event → Binance futures market reacts first (most liquid, most sophisticated traders)
  → Binance spot
  → OKX spot
  → Coinbase
  → smaller exchanges (lag 100ms-5s)
```

**Price discovery arbitrage**: buy on lagging exchange, sell on leading exchange as price converges. Millisecond window.

---

## Market Maker Economics

Market makers quote both bid and ask continuously. They earn the spread but bear inventory risk and adverse selection risk.

### Avellaneda-Stoikov Model (Core Market Making Math)

The fundamental market making model. Market maker quotes around a **reservation price** that accounts for inventory.

```
Reservation price:
  r(s, q, t) = s - q * γ * σ² * (T - t)

Where:
  s = current mid price
  q = current inventory (signed: positive = long)
  γ = risk aversion parameter
  σ = price volatility
  T - t = time remaining

Bid and ask:
  r_bid = r - spread/2
  r_ask = r + spread/2

Optimal spread:
  δ* = γ * σ² * (T - t) + (2/γ) * ln(1 + γ/κ)
  
Where κ = order arrival rate sensitivity to spread
```

**Key insight**: the model tells you to **skew quotes away from inventory**. Long too much → quote lower bid, higher ask to attract sellers.

### Spread Must Cover Costs

```
Minimum viable spread:
  ≥ fee_taker (cost when someone hits your quote)
  + expected adverse selection loss
  + inventory financing cost
  
If spread < these costs → market making unprofitable
```

---

## Adverse Selection

The most dangerous risk for market makers. Arises from **information asymmetry**.

**Scenario**: You're market making BTC. A trader hits your ask (buys from you).

- If trader is uninformed (noise trader) → price bounces back → you profit from spread
- If trader is informed (has alpha) → price keeps going up → you now hold short BTC at bad price → loss

The question is: **what fraction of flow is informed?**

### Kyle (1985) Lambda

Linear price impact model. Kyle's lambda measures adverse selection from order flow.

```
Δp = λ × Q

Where:
  Δp = price change
  Q = signed order flow (+ = buy, - = sell)
  λ = Kyle's lambda (price impact per unit flow)
```

High λ → informed trading → price moves a lot per unit flow → MM quotes wider spread.

### VPIN (Volume-Synchronized Probability of Informed Trading)

Real-time toxicity measure. Splits trades into buy/sell-initiated buckets. High VPIN = imbalanced flow = informed trading likely = expect volatility and spread widening.

```
VPIN = |V_buy - V_sell| / V_total

Range [0, 1]. Near 0 = balanced flow. Near 1 = one-directional = toxic.
```

Easley et al. showed VPIN predicted the 2010 Flash Crash.

---

## Market Impact Models

When you trade, you move the price. This cost is **market impact**.

### Temporary vs Permanent Impact

```
Temporary impact: price moves up while you're buying, reverts after you stop
  → cost of liquidity consumption, not information

Permanent impact: price moves up and stays there
  → your trade was informed, or you moved the fundamental
```

**Total impact**:
```
Total impact = temporary + permanent
             = η * Q^α + γ * Q^β

Typical values: α ≈ 0.5, β ≈ 1.0, η/γ calibrated to market
```

### Square Root Law (Empirical)

Most robust empirical market impact finding:

```
Expected impact = σ * √(Q / ADV)

Where:
  σ = daily volatility
  Q = trade size
  ADV = average daily volume
  
Constant ≈ 1 (varies by market)
```

**Example**: want to buy $1M of BTC, ADV = $500M, σ = 2%/day

```
Impact = 2% × √(1M / 500M) = 2% × 0.045 = 0.09%
At BTC price $47,000: cost ≈ $42 per BTC
```

Multiply by size → total slippage cost of execution.

### Almgren-Chriss: Optimal Execution

How to liquidate a large position optimally? Trade-off: execute fast (less market risk, more impact) vs execute slow (less impact, more market risk).

```
Optimal execution trajectory minimizes:
  E[total cost] + λ × Var[total cost]

Solution: TWAP is approximately optimal when impact is linear in rate
With risk aversion: front-load execution (trade more early)
```

This is the theory behind institutional TWAP/VWAP algorithms.

---

## Liquidity Measurement

### Bid-Ask Spread
Simplest liquidity measure. Tight = liquid. But only measures top of book.

### Market Depth
Total quantity within X% of mid price. Deeper = more liquid.

```
Depth at 0.1% = sum of all bids within 0.1% below mid + asks within 0.1% above mid
```

### Amihud Illiquidity Ratio

```
ILLIQ = (1/T) × Σ |r_t| / Vol_t

Where:
  r_t = daily return
  Vol_t = daily dollar volume

High ILLIQ → price moves a lot per dollar traded → illiquid
```

Used for liquidity risk premia in factor models.

### Realized Spread and Price Impact

After a trade:
```
Realized spread = 2 * q_t * (p_t - m_{t+τ})

Where:
  q_t = +1 (buy) or -1 (sell)
  p_t = trade price
  m_{t+τ} = mid price τ time later
```

If realized spread > 0 on average → MM is profitable → spread is more than adverse selection cost.

**Decomposition**:
```
Quoted spread = Realized spread + Price impact

Where:
  Realized spread = profit to MM (spread component)
  Price impact = adverse selection (information component)
```

---

## Order Flow Imbalance (OFI)

Predictor of short-term price movement.

```
OFI = ΔBid_size - ΔAsk_size

Where changes are measured each tick.
ΔBid_size > 0: more buyers joining bid → bullish
ΔAsk_size > 0: more sellers joining ask → bearish

OFI = net buying pressure in the order book
```

Studies show OFI explains ~70% of short-term price changes in liquid markets. This is a core HFT alpha signal.

---

## Transaction Costs in Crypto

Total cost of a trade:

```
Total cost = Exchange fee + Spread cost + Market impact + Opportunity cost + Funding cost
```

### Exchange Fees (Binance example)
```
Spot taker: 0.10%
Spot maker: 0.08%
Futures taker: 0.05%
Futures maker: 0.02%
VIP 9 maker: -0.005% (rebate)
```

With BNB discount: 25% off all fees.

### Spread Cost
```
Half-spread = (ask - bid) / 2
For market order: you pay full half-spread
For limit order: you earn half-spread (as maker)
```

### Funding (Perpetuals)
```
Funding cost every 8 hours = funding_rate × position_value
Annual cost at 0.01% per 8h = 0.01% × 3/day × 365 = 10.95%/year
```

Very significant for carry strategies. Positive funding = longs pay shorts.

### Network Fees (DEX/On-chain)
```
Transaction fee = gas_used × base_fee + tip
Typical ETH swap: $2-50
Typical Arbitrum swap: $0.10-1
Typical Solana swap: ~$0.00025
```

---

## Price Impact in Crypto vs Equities

Crypto is unique:
1. **Venue fragmentation**: same asset on 50+ exchanges. Arbitrageurs keep prices aligned but create cross-venue dynamics.
2. **No circuit breakers**: crypto doesn't halt trading. Flash crashes go to zero bids.
3. **Liquidation feedback loops**: forced liquidations amplify price moves.
4. **Thin books on alt coins**: small-cap tokens have minimal depth. $50K buy = 5-10% move.

### Liquidation Cascade Effect

```
Large sell → price falls → overleveraged longs hit margin call
→ exchange auto-liquidates at market → price falls more
→ more liquidations → feedback loop
```

This is why crypto flash crashes are fast and deep. Historical examples:
- March 2020: BTC -50% in 2 days (COVID)  
- May 2021: BTC -55% in 1 month (China ban)
- Nov 2022: -25% in 1 day (FTX collapse)

---

## Tick Size and Lot Size

**Tick size**: minimum price increment. Small tick = more price levels = wider spread in ticks but tighter in dollars.

**Lot size**: minimum order quantity.

```
BTC/USDT on Binance:
  Tick size: $0.01
  Min lot: 0.00001 BTC
  
ETH/USDT on Binance:
  Tick size: $0.01
  Min lot: 0.00010 ETH
```

For HFT: tick size determines how fine-grained your quote placement can be.

---

## Information Efficiency

**Efficient Market Hypothesis (EMH)**:
- Weak form: price reflects all historical prices
- Semi-strong form: price reflects all public information
- Strong form: price reflects all information (including private)

Crypto markets are weak-form efficient on major pairs. Semi-strong efficiency is questionable (news events often predictable, on-chain data underutilized). Strong form clearly not true (insiders, private block data).

**Implication**: alpha from price patterns is hard. Alpha from information edges (on-chain data, order flow, cross-venue) is more durable.

---

## Next

Read [05_quant_trading.md](05_quant_trading.md) for how to build systematic strategies on top of this microstructure understanding.
