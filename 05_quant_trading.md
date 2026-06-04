# 05 — Quantitative Trading

Quant trading = use mathematical models and systematic rules to make trading decisions, rather than discretionary judgment.

Goal: find alpha (excess return), size it appropriately, execute efficiently.

---

## Alpha

**Alpha** = return in excess of what the risk exposure (beta) would predict.

```
Return = α + β₁ × market + β₂ × factor₂ + ... + ε

Alpha = α (what's left after explaining all systematic risk)
```

Finding alpha means finding a signal that predicts future price movement better than random chance.

### Alpha Decay

Alpha erodes over time as:
1. More participants discover and trade on the signal
2. Market becomes more efficient at incorporating the information
3. Regime change makes historical relationship invalid

Alpha horizon: how long does the signal predict prices?
- HFT signals: microseconds to seconds
- Stat arb signals: minutes to hours
- Factor signals: days to months

### Alpha Sources in Crypto

| Source | Type | Horizon |
|---|---|---|
| Order flow imbalance | Microstructure | Seconds |
| Cross-exchange price lag | Arb | Milliseconds-seconds |
| Funding rate extremes | Carry/mean reversion | Hours |
| On-chain accumulation | Fundamental | Days-weeks |
| Liquidation level proximity | Technical | Hours |
| Sentiment (social, news) | Alternative data | Hours-days |
| Momentum (cross-section) | Price | Days-months |
| Mean reversion (intraday) | Price | Minutes-hours |

---

## Signal Construction

A signal is a quantitative measure that predicts price direction, magnitude, or both.

### Step 1: Feature Engineering

Raw data → features:
```python
# Example: Funding rate Z-score signal
import pandas as pd
import numpy as np

def funding_zscore(funding_rate, lookback=720):  # 720 periods = 10 days at 8h intervals
    rolling_mean = funding_rate.rolling(lookback).mean()
    rolling_std = funding_rate.rolling(lookback).std()
    zscore = (funding_rate - rolling_mean) / rolling_std
    return zscore
```

### Step 2: Signal Evaluation

```python
# Information Coefficient (IC): Spearman correlation between signal and forward returns
from scipy.stats import spearmanr

def information_coefficient(signal, forward_returns):
    # Remove NaNs
    mask = ~(np.isnan(signal) | np.isnan(forward_returns))
    ic, p_value = spearmanr(signal[mask], forward_returns[mask])
    return ic, p_value

# ICIR = IC / std(IC) — risk-adjusted signal strength
def ic_information_ratio(ic_series):
    return ic_series.mean() / ic_series.std() * np.sqrt(252)
```

**Good signal thresholds**:
- |IC| > 0.02: shows some skill
- |IC| > 0.05: good signal
- |IC| > 0.10: excellent (rare)
- ICIR > 0.3: worth trading

### Step 3: Signal Combination

Multiple weak signals combined = stronger signal (diversification in signal space).

```python
# Simple equal-weight combination
combined_signal = (signal1 + signal2 + signal3) / 3

# Optimized weighting (maximize Sharpe on training period)
from scipy.optimize import minimize

def neg_sharpe(weights, signals, returns):
    port_signal = (signals * weights).sum(axis=1)
    # rank-normalize
    ranked = port_signal.rank(pct=True) - 0.5
    pnl = ranked * returns
    return -pnl.mean() / pnl.std()
```

---

## Strategy Types

### 1. Market Making

Quote both bid and ask, earn spread. Requires:
- Fast execution (cancel before adverse moves)
- Inventory management
- Spread wider than fees + adverse selection

See [07_hft_low_latency.md](07_hft_low_latency.md) for implementation.

### 2. Statistical Arbitrage (Stat Arb)

Trade mean-reverting spread between correlated assets.

**Pairs Trading**:
```python
# Find cointegrated pair: ETH and BTC often cointegrated
from statsmodels.tsa.stattools import coint

def find_pairs(prices_df):
    pairs = []
    n = len(prices_df.columns)
    for i in range(n):
        for j in range(i+1, n):
            _, pvalue, _ = coint(prices_df.iloc[:,i], prices_df.iloc[:,j])
            if pvalue < 0.05:  # cointegrated at 5%
                pairs.append((prices_df.columns[i], prices_df.columns[j], pvalue))
    return sorted(pairs, key=lambda x: x[2])

# Spread: residual from regression
def hedge_ratio(y, x):
    from numpy.linalg import lstsq
    model = lstsq(x.values.reshape(-1,1), y.values, rcond=None)
    return model[0][0]

# Signal: z-score of spread
# Entry: |z| > 2, Exit: |z| < 0.5
```

**Cross-Exchange Arb**:
```python
# Price of BTC differs across exchanges
# If Coinbase price > Binance price by > fees + transfer cost → buy on Binance, sell on Coinbase
# Caveat: transfer time may be too slow. Must have pre-funded accounts on both sides.
```

**Triangular Arbitrage** (within one exchange):
```
BTC/USDT * USDT/ETH * ETH/BTC ≠ 1 → circular arb opportunity
If product < 1 - fees: buy BTC → sell for USDT → buy ETH → sell ETH for BTC → profit
```

### 3. Momentum

Assets that went up continue going up (and vice versa). Works across all timeframes.

**Time-series momentum (TSMOM)**:
```python
# If 12-month return > 0: go long, else go short
def tsmom_signal(prices, lookback=252):
    return prices.pct_change(lookback).apply(np.sign)
```

**Cross-sectional momentum**:
```python
# Rank assets by past return. Long top decile, short bottom decile.
def cross_mom_signal(returns_df, lookback=30):
    past_returns = returns_df.rolling(lookback).sum()
    ranks = past_returns.rank(axis=1, pct=True)
    return (ranks - 0.5) * 2  # center at 0
```

Momentum in crypto is strong at:
- 1-day (intraday trend following)
- 1-week (weekly momentum)
- 1-3 months (asset allocation momentum)

Mean reversion at:
- Sub-minute (microstructure reversion)
- Intraday (opening gap reversion)

### 4. Carry / Funding Rate Arbitrage

Perpetual futures have funding rate. If consistently positive: short perp, buy spot → earn funding.

```python
# Funding rate arbitrage
# Long BTC spot, Short BTC perpetual
# Earn funding when rate positive (longs pay shorts)

def carry_signal(funding_rate, threshold=0.01/100):  # 0.01% per 8h = 10.95% annualized
    return funding_rate > threshold

# Risk: spot-perp basis can move against you
# Net P&L = funding_earned - spot_hedge_cost - transaction_fees - basis_change
```

### 5. Mean Reversion

Prices revert to mean after deviation. Works best in range-bound markets.

**Bollinger Bands**:
```python
def bollinger_signal(prices, window=20, n_std=2):
    rolling_mean = prices.rolling(window).mean()
    rolling_std = prices.rolling(window).std()
    upper = rolling_mean + n_std * rolling_std
    lower = rolling_mean - n_std * rolling_std
    
    signal = pd.Series(0, index=prices.index)
    signal[prices < lower] = 1   # oversold, buy
    signal[prices > upper] = -1  # overbought, sell
    return signal
```

### 6. Trend Following / CTA Style

Medium-term momentum. Hold until signal reverses.

```python
def trend_signal(prices, fast=20, slow=60):
    fast_ma = prices.ewm(span=fast).mean()
    slow_ma = prices.ewm(span=slow).mean()
    return np.sign(fast_ma - slow_ma)
```

Crypto CTAs: AHL, Brevan Howard crypto, Coinbase Asset Management all run these.

---

## Portfolio Construction

Signal → position sizing → portfolio.

### Kelly Criterion

Optimal fraction of capital to risk on each bet:

```
Kelly fraction f* = (p × b - q) / b

Where:
  p = probability of win
  q = probability of loss = 1 - p
  b = odds (win/loss ratio)

Example: 55% win rate, 1:1 payoff
  f* = (0.55 × 1 - 0.45) / 1 = 0.10 → bet 10% of bankroll
```

Full Kelly is too aggressive in practice. Most quant funds use **half-Kelly** or **quarter-Kelly**.

**Continuous version** for a signal with Sharpe ratio S:
```
Kelly fraction = S² / σ

Where σ = annualized volatility
```

### Volatility Targeting

Size positions to target constant risk (not constant notional).

```python
def vol_target_size(signal, returns, target_vol=0.20, lookback=30):
    realized_vol = returns.rolling(lookback).std() * np.sqrt(252)
    scale = target_vol / realized_vol
    return signal * scale.clip(0.1, 3)  # cap at 3x, floor at 0.1x
```

**Why**: a $100K position in low-vol asset carries same risk as $10K position in high-vol asset. Equal-notional portfolios have unequal risk.

### Risk Parity

Weight assets by inverse volatility so each contributes equal risk.

```python
def risk_parity_weights(returns_df, lookback=60):
    vols = returns_df.rolling(lookback).std()
    inv_vol = 1 / vols
    weights = inv_vol.div(inv_vol.sum(axis=1), axis=0)
    return weights
```

---

## Backtesting

Testing strategy on historical data. Most important — and most error-prone — step.

### Backtest Pipeline

```python
# Minimal correct backtest structure
def backtest(signals, prices, transaction_cost_bps=10):
    """
    signals: pd.DataFrame, each row is desired position (-1 to 1)
    prices: pd.DataFrame, same index/columns
    """
    # Compute position changes
    position_changes = signals.diff().abs().sum(axis=1)
    
    # Daily returns = position(t) × return(t+1)
    # CRITICAL: use next period return, NOT same period (lookahead bias!)
    returns = prices.pct_change()
    portfolio_returns = (signals.shift(1) * returns).sum(axis=1)  # shift(1) = use yesterday's signal
    
    # Transaction costs
    tc = position_changes * (transaction_cost_bps / 10000)
    
    net_returns = portfolio_returns - tc
    return net_returns
```

### Backtest Biases (Ways You'll Fool Yourself)

**1. Lookahead Bias** (most common, most dangerous)
Using information at time t that wouldn't be available until t+1.

```python
# BAD: uses today's close to generate today's signal, then also trades at today's close
signal = compute_signal(prices)  
returns = signal * prices.pct_change()  # WRONG: trades same bar signal was computed

# GOOD: generate signal from yesterday's data, trade today's open
signal = compute_signal(prices.shift(1))
returns = signal * prices.pct_change()  # or next open return
```

**2. Survivorship Bias**
Testing only on assets that exist today. Excludes delisted coins (died → 100% loss). 

For crypto: include coins that got delisted/died in your universe.

**3. Overfitting / Data Snooping**
Testing 1000 parameter combinations, reporting best result. 

Rule: if you try N parameter sets, require Sharpe√N to account for multiple testing.

**4. Transaction Cost Underestimation**
Backtests use mid-price. Reality: you trade at bid (if selling) or ask (if buying). Plus market impact on larger trades.

Use realistic cost model:
```
cost_per_trade = spread/2 + market_impact + exchange_fee
For crypto: typically 5-30 bps total depending on size and exchange
```

**5. Liquidity Assumption**
Assuming you can trade at any time at market price. Reality: large orders move market, some altcoins have hours of zero volume.

**6. Stationarity Assumption**
Signal that worked 2019-2021 may fail post-2022 (different regime). Always test on out-of-sample periods.

---

## Performance Metrics

### Sharpe Ratio (Most Important)

```
Sharpe = (Mean Return - Risk Free Rate) / Std(Returns) × √(periods_per_year)

Annualized Sharpe for daily returns:
  Sharpe = daily_mean / daily_std × √252

For minute-level: × √(252 × 390) for equity, × √(252 × 1440) for crypto
```

Thresholds:
- < 0.5: not worth trading
- 0.5-1.0: acceptable with low cost
- 1.0-2.0: good
- > 2.0: excellent (rare and probably overfit)
- > 3.0: suspect (check for lookahead bias)

### Sortino Ratio

Like Sharpe but penalizes only downside volatility.

```
Sortino = (Mean Return - Target Return) / Downside Deviation × √252

Where downside deviation = std of returns below target (usually 0)
```

Better for asymmetric strategies.

### Maximum Drawdown

```
Max Drawdown = (Peak - Trough) / Peak

Calmar Ratio = Annual Return / |Max Drawdown|
```

Anything worse than -20% drawdown is hard to recover from psychologically and practically.

### Win Rate and Profit Factor

```
Win Rate = # winning trades / total trades
Profit Factor = gross profit / gross loss

A strategy with 40% win rate can still be very profitable if average win >> average loss
```

### Turnover

```
Annual turnover = sum of |position changes| / average_capital

High turnover → high costs → need high gross alpha to survive
```

For crypto stat arb at daily frequency: typically 2-10x turnover.

---

## Walk-Forward Optimization

Correct way to optimize and test:

```
Full dataset: [Train1][Val1][Train2][Val2]...[Test]
                 ↓      ↓
              Optimize  Pick best params
                        
Walk-forward: slide window forward, refit periodically
Out-of-sample (Test): NEVER touch until final evaluation
```

**Embargo period**: exclude data immediately after train → avoid microstructure contamination.

---

## Next

Read [06_day_trading.md](06_day_trading.md) for discretionary and semi-systematic day trading techniques.
