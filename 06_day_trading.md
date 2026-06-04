# 06 — Day Trading

Day trading = open and close positions within the same trading session. No overnight holds. Faster feedback loop than swing trading but higher transaction cost drag.

This chapter bridges discretionary intuition and systematic rules.

---

## Why Day Trading Crypto is Different from Equities

| Factor | Equities (US) | Crypto |
|---|---|---|
| Market hours | 6.5 hours/day | 24/7/365 |
| Volatility | 1-2% daily typical | 2-10% daily typical |
| Leverage available | 2-4x retail | Up to 125x (Binance) |
| Funding rate | None (borrowing cost) | Every 8h |
| Liquidity | Deep on large caps | Thin on altcoins |
| News events | Market hours only | Any time |
| Circuit breakers | Halts trading | None |
| Tax events | Capital gains | Every trade is taxable event |

Crypto's 24/7 nature is both opportunity (can trade overnight sessions) and danger (can't sleep).

---

## Market Sessions

Even though crypto trades 24/7, volume and volatility cluster by timezone:

```
UTC Times:
  00:00-08:00: Asia session (Tokyo/Shanghai/Singapore)
    Characteristics: often trend continuation, lower volume
    
  08:00-17:00: Europe/London session
    Characteristics: picks up volume, overlaps with Asia close
    
  13:00-22:00: US session (overlaps with Europe 13:00-17:00)
    Most volatile: US open overlap = highest volume window
    Key times: 13:30-16:00 UTC = US equity open (correlated to risk)
    
  22:00-00:00: Quiet period before Asia opens
    Often used for accumulation by large players
```

**Monthly patterns**:
- Month end: institutional rebalancing
- Options expiry (last Friday of month): large open interest → pin risk
- Halving cycle: 4-year Bitcoin supply cycle narrative

---

## Technical Analysis

TA is controversial in academia (weak form efficiency suggests it shouldn't work). In practice, TA signals work because enough market participants believe in them → self-fulfilling prophecy. The signal isn't in the pattern per se, but in the collective behavior around the pattern.

### Candlestick Charts

```
     │       ← high (wick)
   ┌─┴─┐     ← open/close (body)
   │   │     (green = close > open = bullish)
   └─┬─┘     (red = close < open = bearish)
     │       ← low (wick)
```

**Key single-candle patterns**:
- **Doji**: open ≈ close, long wicks. Indecision.
- **Hammer**: small body, long lower wick. Rejection of lower prices. Bullish at support.
- **Shooting star**: small body, long upper wick. Rejection of higher prices. Bearish at resistance.
- **Marubozu**: full body, no wicks. Strong directional move.

**Multi-candle patterns**:
- **Engulfing**: second candle fully engulfs first. Strong reversal signal.
- **Morning star / Evening star**: 3-candle reversal at extremes.

### Moving Averages

Trend identification tools. Lag by nature.

```
Simple MA: SMA(n) = (p₁ + p₂ + ... + pₙ) / n

Exponential MA: EMA(t) = price(t) × α + EMA(t-1) × (1-α)
  where α = 2 / (n + 1)

Golden Cross: 50 SMA crosses above 200 SMA → bullish
Death Cross: 50 SMA crosses below 200 SMA → bearish
```

Key levels watched by crowd:
- 50 EMA, 100 EMA, 200 EMA (daily chart = $50K watchers)
- 20 EMA (shorter-term trend traders)

### RSI (Relative Strength Index)

Momentum oscillator, range [0, 100].

```python
def rsi(prices, n=14):
    delta = prices.diff()
    gain = delta.clip(lower=0)
    loss = (-delta).clip(lower=0)
    avg_gain = gain.rolling(n).mean()
    avg_loss = loss.rolling(n).mean()
    rs = avg_gain / avg_loss
    return 100 - (100 / (1 + rs))
```

- RSI > 70: overbought (not necessarily sell signal!)
- RSI < 30: oversold (not necessarily buy signal!)
- Divergence: price makes new high but RSI doesn't → weakening momentum

**Crypto caveat**: in strong trends, RSI can stay "overbought" for weeks (2021 BTC ran from 40 to 69k with RSI persistently above 70). Don't fade RSI in trending markets.

### MACD (Moving Average Convergence Divergence)

```
MACD line = EMA(12) - EMA(26)
Signal line = EMA(9) of MACD
Histogram = MACD - Signal

Signal: MACD crosses above signal = bullish
        MACD crosses below signal = bearish
        Histogram zero-cross = entry/exit
```

### Bollinger Bands

```
Middle band = 20 SMA
Upper band = 20 SMA + 2σ
Lower band = 20 SMA - 2σ

Squeeze: bands narrow → volatility compressed → explosive move coming
Walk the band: price hugging upper/lower = strong trend, not reversal
```

### Volume

**Volume confirms price**: large move with large volume = genuine. Large move with tiny volume = potentially fake.

```
Volume SMA: compare current volume vs 20-day average
On-Balance Volume (OBV): running total of +/- volume by day direction
  → OBV diverges from price = leading indicator
```

### Support & Resistance

Levels where price has repeatedly reversed. Why they work: traders place orders (buy limit at support, sell limit at resistance) → becomes self-fulfilling.

**Key levels in crypto**:
- Round numbers: $40,000, $50,000, $100,000 BTC
- Previous all-time high/low
- 50% and 61.8% Fibonacci retracements
- Volume profiles: price levels where most volume traded (VPVR)

---

## Day Trading Strategies

### 1. Momentum / Trend Following (Intraday)

Trade in direction of established move. Enter on pullback, exit on momentum exhaustion.

```
Setup:
  15m chart: strong uptrend (price above 20 EMA, 20 EMA above 50 EMA)
  1m chart: RSI pulls back to 40-50 (retest of momentum)
  Entry: break above previous 1m candle high
  Stop: below swing low on 1m
  Target: 1:2 or 1:3 risk/reward, or trail with 5 EMA
```

**Crypto specific**: Bitcoin/Ethereum often lead altcoins. If BTC breaks out: alts follow with 5-30 min lag. Trade alts on BTC momentum signal.

### 2. Opening Range Breakout (adapted for crypto)

In crypto: no official open. Use a "session open" (e.g., 00:00 UTC, or 08:00 UTC Europe open, or 13:30 UTC US open).

```
Setup:
  Mark high and low of first 30 minutes after "open"
  If price breaks above high: buy (stop below range low)
  If price breaks below low: sell (stop above range high)
  
Works best when first-30-min range is relatively narrow (shows indecision before breakout)
```

### 3. Mean Reversion (Fade Extremes)

After large fast move, price often reverts. High-risk, high-reward.

```
Setup:
  Price moves 3%+ in 15 minutes on no fundamental news
  RSI > 80 or < 20 on 5m chart
  Volume declining on the move (exhaustion)
  Counter-trend entry with tight stop above/below extreme
  Target: 50% retracement or VWAP
```

**Critical**: never fade with leverage in trending crypto market. This strategy gets destroyed in strong trends. Use only in range-bound conditions.

### 4. VWAP Trading

VWAP (Volume-Weighted Average Price) = average price weighted by volume. Institutions use VWAP as benchmark.

```
VWAP = Σ(price × volume) / Σ(volume)
     (calculated from session start)
```

**Signal**:
- Price above VWAP = bullish intraday bias (institutions are in profit if they bought at VWAP)
- Price below VWAP = bearish intraday bias
- Price reclaims VWAP from below = bullish signal
- Price fails to reclaim VWAP = bearish continuation

### 5. Liquidation-Based Trading

Monitor open interest and funding rate. When open interest is very high with positive funding (many overleveraged longs), short squeeze/cascade liquidation is likely.

```
Signal:
  Funding rate > 0.05% per 8h (high leveraged long)
  Open interest at or near historical high
  Price testing support level
  
Setup: short on break of support, stop above recent high
Target: liquidation cascade to next major support
```

Tools: Bybt.com, Coinglass.com for liquidation heatmaps and OI data.

---

## Risk Management for Day Traders

The most important chapter. Bad risk management kills accounts.

### Position Sizing

**Fixed risk per trade** (best method):

```
Position size = (Account × Risk_per_trade%) / Distance_to_stop

Example:
  Account: $10,000
  Risk per trade: 1% = $100
  Entry: $47,000
  Stop: $46,500 ($500 below)
  
  Position = $100 / $500 = 0.2 BTC
  If stop hits: lose exactly $100 = 1%
```

Never risk more than 1-2% per trade.

### Daily Loss Limit

Set maximum daily loss (e.g., 3-5% of account). If hit: stop trading for the day.

```
After 3 losses: your brain is impaired. Decision quality degrades.
Revenge trading = biggest account killer in retail day trading.
```

### Risk/Reward Ratio

Only take trades with ≥ 1:2 risk/reward (risk 1% to make 2%).

```
Win rate 50%, R:R 1:2:
  10 trades: 5 wins (2% each) + 5 losses (-1% each) = +5% net
  
Win rate 40%, R:R 1:3:
  10 trades: 4 wins (3% each) + 6 losses (-1% each) = +6% net
  
You can be wrong 60% of the time and still profit with good R:R!
```

### Leverage in Crypto

Available up to 125x on Binance futures. This is a trap.

```
10x leverage: 10% adverse move = 100% liquidation of position
20x leverage: 5% adverse move = 100% liquidation
100x leverage: 1% adverse move = 100% liquidation

BTC daily volatility ≈ 3-5%
Most retail 100x trades liquidated within hours
```

Recommended for beginners: 1-3x max. Professional day traders: 5-10x max in specific setups.

### Correlation

Don't hold 5 "different" crypto positions — they all go up/down together during risk-on/risk-off.

```
BTC/ETH correlation: usually 0.85-0.95
BTC/altcoin correlation: 0.60-0.90

"Diversified" 5-alt portfolio during crash: all -40% simultaneously
Real diversification: need negatively correlated assets
```

---

## Psychology

Trading psychology is underestimated. Systematic rules exist partly to override emotional decisions.

**Common emotional errors**:
- Holding losers too long (loss aversion: losses hurt 2x more than equivalent gains feel good)
- Cutting winners too early (taking profits before target to "secure" gain)
- Revenge trading after a loss
- FOMO — entering too late into a move
- Overconfidence after wins — increasing size at peak

**Fix**: trade rules mechanically. Keep a journal. Review losing trades without emotion.

**The edge is in execution, not the setup**. You can have a great strategy and blow up if you ignore your stops or double down on losers.

---

## Tools for Day Traders

| Tool | Purpose |
|---|---|
| TradingView | Charting (best for crypto) |
| Coinglass | Liquidation heatmaps, OI, funding |
| CryptoQuant | On-chain flows |
| Bybit/Binance | Exchange with good UI |
| Bookmap | Depth-of-market visualization |
| Glassnode | On-chain analytics |
| Alternative.me/Fear & Greed | Sentiment index |

---

## Journal Template

Every trade should be logged:

```
Date/Time:
Asset + direction:
Timeframe:
Setup type:
Entry price:
Stop price:
Target price:
Rationale (why this trade?):
Result:
P&L ($):
Notes (what did I do well/badly?):
```

Review weekly. Identify patterns in your errors.

---

## Next

Read [07_hft_low_latency.md](07_hft_low_latency.md) to understand how professional firms operate at microsecond speeds.
