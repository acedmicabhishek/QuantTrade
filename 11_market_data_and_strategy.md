# 11 — Market Data: Capture, Analysis & Strategy Building

How to pull historical data for NSE/BSE, clean it, extract signals, and turn those signals into a systematic strategy.

---

## 1. What Data Do You Actually Need?

Before fetching anything, know what your strategy requires.

| Data Type | Granularity | Use Case |
|---|---|---|
| **OHLCV** | Daily / hourly / minute | Trend, momentum, mean reversion |
| **Tick / Trade data** | Every trade | Microstructure, HFT |
| **Order book (L2)** | Snapshot / event | Market making, impact modelling |
| **Fundamentals** | Quarterly | Factor investing (P/E, ROE, debt) |
| **Corporate events** | Date-stamped | Earnings, dividends, splits, bonus |
| **Index weights** | Periodic | Reconstitution trades, beta hedging |
| **Options chain** | Daily snapshot | Implied vol, put-call ratio, PCR |

For most retail and mid-frequency quant strategies, **daily OHLCV + corporate actions** is the starting point.

---

## 2. Data Sources for NSE and BSE

### Free Sources

| Source | What You Get | Access Method |
|---|---|---|
| **NSE India official** | EOD prices, F&O data, indices | nseindia.com — download CSV manually or scrape |
| **BSE India official** | EOD prices, bulk deals, board meetings | bseindia.com |
| **yfinance (Yahoo Finance)** | Daily OHLCV, adjusted prices | Python library |
| **nsepy** | NSE EOD + derivatives historical data | Python library (unmaintained but still works) |
| **jugaad-trader** | NSE live + historical | Python library, actively maintained |
| **Stooq** | Daily data for NSE symbols | Via pandas-datareader |
| **Alpha Vantage** | Intraday + daily, some Indian tickers | REST API, free tier = 25 req/day |

### Paid / Professional Sources

| Source | Notes |
|---|---|
| **Zerodha Kite Connect API** | Intraday tick + historical, INR 2000/month, official broker API |
| **Upstox API** | Similar to Kite, free historical with demat account |
| **True Data** | Clean NSE/BSE tick data, used by serious quants |
| **Global Data Feed (GDF)** | Tick data, popular with Indian algo traders |
| **Refinitiv / Bloomberg** | Institutional grade, expensive |

For learning and strategy development, **yfinance + Kite Connect** covers 95% of use cases.

---

## 3. Fetching Data — Practical Code

### 3.1 Daily OHLCV via yfinance

NSE symbols use the `.NS` suffix; BSE symbols use `.BO`.

```python
import yfinance as yf
import pandas as pd

# Single stock — Reliance Industries (NSE)
reliance = yf.download("RELIANCE.NS", start="2020-01-01", end="2024-12-31")
print(reliance.head())

# Multiple stocks at once
tickers = ["RELIANCE.NS", "TCS.NS", "INFY.NS", "HDFCBANK.NS", "TATAMOTORS.NS"]
data = yf.download(tickers, start="2020-01-01", end="2024-12-31")

# NIFTY 50 index
nifty = yf.download("^NSEI", start="2020-01-01", end="2024-12-31")
```

Output columns: `Open, High, Low, Close, Adj Close, Volume`

Always use **Adj Close** for return calculations — it accounts for dividends and splits.

### 3.2 Intraday Data via Kite Connect

```python
from kiteconnect import KiteConnect
import datetime

kite = KiteConnect(api_key="your_api_key")
kite.set_access_token("your_access_token")

# Fetch 15-minute candles for RELIANCE
data = kite.historical_data(
    instrument_token=738561,   # RELIANCE NSE token
    from_date=datetime.date(2024, 1, 1),
    to_date=datetime.date(2024, 3, 31),
    interval="15minute"
)

df = pd.DataFrame(data)
df.set_index("date", inplace=True)
```

### 3.3 NSE Options Chain Snapshot

```python
import requests
import pandas as pd

def fetch_nse_option_chain(symbol="NIFTY"):
    url = f"https://www.nseindia.com/api/option-chain-indices?symbol={symbol}"
    headers = {
        "User-Agent": "Mozilla/5.0",
        "Accept": "application/json",
        "Referer": "https://www.nseindia.com"
    }
    session = requests.Session()
    session.get("https://www.nseindia.com", headers=headers)
    response = session.get(url, headers=headers)
    data = response.json()
    records = data["records"]["data"]
    rows = []
    for r in records:
        for opt_type in ["CE", "PE"]:
            if opt_type in r:
                row = r[opt_type]
                row["optionType"] = opt_type
                row["strikePrice"] = r["strikePrice"]
                rows.append(row)
    return pd.DataFrame(rows)

oc = fetch_nse_option_chain("NIFTY")
```

---

## 4. Cleaning and Preparing the Data

Raw data is never ready to trade on. These steps are non-negotiable.

### 4.1 Adjust for Corporate Actions

```python
# yfinance Adj Close already handles this.
# If using raw data, manually adjust:

def adjust_for_split(df, split_date, split_ratio):
    df.loc[df.index < split_date, ["Open","High","Low","Close"]] /= split_ratio
    df.loc[df.index < split_date, "Volume"] *= split_ratio
    return df
```

### 4.2 Handle Missing Data

```python
# Check for gaps
df = df.asfreq("B")        # Business day frequency
missing = df[df["Close"].isna()]
print(f"Missing days: {len(missing)}")

# Forward-fill short gaps (holidays, data outages)
df["Close"] = df["Close"].fillna(method="ffill")

# Drop if gap is too long (suspicious data)
df = df.dropna(subset=["Close"])
```

### 4.3 Detect and Handle Outliers

```python
# Flag suspicious single-day returns
returns = df["Adj Close"].pct_change()
threshold = 0.25   # 25% single-day move is suspicious for large caps

outliers = returns[returns.abs() > threshold]
print("Suspect returns:\n", outliers)

# Winsorise rather than drop — preserves time series continuity
from scipy.stats import mstats
df["returns_clean"] = mstats.winsorize(returns.dropna(), limits=[0.01, 0.01])
```

### 4.4 Compute Log Returns

Prefer log returns over simple returns for statistical analysis:

```
r_t = ln(P_t / P_{t-1})
```

```python
df["log_ret"] = np.log(df["Adj Close"] / df["Adj Close"].shift(1))
df["simple_ret"] = df["Adj Close"].pct_change()
```

Log returns are:
- Additive over time (multi-period returns = sum of daily log returns)
- Approximately normally distributed
- Symmetric (no lower bound at -100%)

---

## 5. Exploratory Data Analysis (EDA)

Before building any strategy, understand the data's statistical properties.

### 5.1 Distribution of Returns

```python
import matplotlib.pyplot as plt
from scipy import stats

returns = df["log_ret"].dropna()

print(f"Mean daily return : {returns.mean():.4f}")
print(f"Std dev (daily)   : {returns.std():.4f}")
print(f"Annualised return : {returns.mean()*252:.2%}")
print(f"Annualised vol    : {returns.std()*np.sqrt(252):.2%}")
print(f"Skewness          : {returns.skew():.3f}")
print(f"Kurtosis (excess) : {returns.kurtosis():.3f}")

# Indian large caps typically show:
# - slight negative skew (crash asymmetry)
# - fat tails (excess kurtosis > 3) — BSM assumption of normality is violated
```

### 5.2 Autocorrelation (Is There Serial Predictability?)

```python
from statsmodels.stats.stattools import durbin_watson
from statsmodels.graphics.tsaplots import plot_acf

plot_acf(returns, lags=30)
plt.title("Autocorrelation of RELIANCE returns")
plt.show()

dw = durbin_watson(returns.dropna())
print(f"Durbin-Watson stat: {dw:.3f}")
# ~2 means no autocorrelation; <2 positive serial correlation
```

### 5.3 Stationarity Check (ADF Test)

A strategy built on a non-stationary series will backtest well but fail live.

```python
from statsmodels.tsa.stattools import adfuller

result = adfuller(df["Adj Close"].dropna())
print(f"ADF Statistic: {result[0]:.4f}")
print(f"p-value      : {result[1]:.4f}")
# p > 0.05 → non-stationary (price levels always are)

result_ret = adfuller(returns.dropna())
print(f"Returns ADF p: {result_ret[1]:.4f}")
# p < 0.05 → stationary ✓ (returns usually are)
```

---

## 6. Technical Indicators — Implementation

Technical indicators are transformations of price/volume that attempt to encode market state.

### 6.1 Moving Averages

```python
df["SMA_20"]  = df["Adj Close"].rolling(20).mean()
df["SMA_50"]  = df["Adj Close"].rolling(50).mean()
df["EMA_20"]  = df["Adj Close"].ewm(span=20, adjust=False).mean()
```

**Golden Cross / Death Cross** — when 50-day MA crosses 200-day MA. A classic (and heavily arbitraged) signal.

### 6.2 Relative Strength Index (RSI)

```
RS  = (avg gain over N days) / (avg loss over N days)
RSI = 100 − 100 / (1 + RS)
```

```python
def compute_rsi(series, period=14):
    delta = series.diff()
    gain  = delta.clip(lower=0).rolling(period).mean()
    loss  = (-delta.clip(upper=0)).rolling(period).mean()
    rs    = gain / loss
    return 100 - 100 / (1 + rs)

df["RSI_14"] = compute_rsi(df["Adj Close"])
# RSI > 70 → overbought; RSI < 30 → oversold
```

### 6.3 MACD (Moving Average Convergence Divergence)

```
MACD Line   = EMA(12) − EMA(26)
Signal Line = EMA(9) of MACD Line
Histogram   = MACD Line − Signal Line
```

```python
ema12 = df["Adj Close"].ewm(span=12, adjust=False).mean()
ema26 = df["Adj Close"].ewm(span=26, adjust=False).mean()
df["MACD"]        = ema12 - ema26
df["MACD_signal"] = df["MACD"].ewm(span=9, adjust=False).mean()
df["MACD_hist"]   = df["MACD"] - df["MACD_signal"]
```

### 6.4 Bollinger Bands

```
Middle Band = SMA(20)
Upper Band  = SMA(20) + 2 × std(20)
Lower Band  = SMA(20) − 2 × std(20)
%B = (Price − Lower) / (Upper − Lower)
```

```python
window = 20
df["BB_mid"]   = df["Adj Close"].rolling(window).mean()
df["BB_std"]   = df["Adj Close"].rolling(window).std()
df["BB_upper"] = df["BB_mid"] + 2 * df["BB_std"]
df["BB_lower"] = df["BB_mid"] - 2 * df["BB_std"]
df["BB_pct"]   = (df["Adj Close"] - df["BB_lower"]) / (df["BB_upper"] - df["BB_lower"])
```

### 6.5 Average True Range (ATR) — Volatility Measure

```
TR  = max(High−Low, |High−Prev Close|, |Low−Prev Close|)
ATR = EMA(TR, 14)
```

```python
high_low   = df["High"] - df["Low"]
high_close = (df["High"] - df["Adj Close"].shift()).abs()
low_close  = (df["Low"]  - df["Adj Close"].shift()).abs()
tr         = pd.concat([high_low, high_close, low_close], axis=1).max(axis=1)
df["ATR_14"] = tr.ewm(span=14, adjust=False).mean()
```

ATR is critical for **position sizing** — size inversely proportional to ATR so every trade risks the same dollar amount.

---

## 7. Building a Strategy — Step by Step

### The Strategy Blueprint

```
1. HYPOTHESIS   → Why should this edge exist?
2. SIGNAL       → What indicator triggers entry / exit?
3. FILTER       → Conditions that must be true before trading
4. SIZING       → How many shares per trade?
5. RISK RULES   → Stop-loss, max drawdown, position limits
6. BACKTEST     → Simulate on historical data
7. VALIDATE     → Walk-forward, out-of-sample, transaction costs
```

---

## 8. Strategy Example 1 — NIFTY 50 Momentum (Trend Following)

**Hypothesis:** NIFTY 50 trends. A breakout above the recent high indicates continuing momentum.

**Signal:** 
- Enter LONG when Close crosses above its 50-day high.
- Exit when Close falls below its 20-day low.

```python
import numpy as np
import pandas as pd
import yfinance as yf

# Fetch NIFTY 50
df = yf.download("^NSEI", start="2015-01-01", end="2024-12-31")
df["Close"] = df["Adj Close"]

# Signals
df["high_50"] = df["Close"].rolling(50).max().shift(1)
df["low_20"]  = df["Close"].rolling(20).min().shift(1)

df["entry_signal"] = df["Close"] > df["high_50"]   # breakout up
df["exit_signal"]  = df["Close"] < df["low_20"]    # breakdown

# Position: 1 = long, 0 = flat
position = 0
positions = []
for i, row in df.iterrows():
    if position == 0 and row["entry_signal"]:
        position = 1
    elif position == 1 and row["exit_signal"]:
        position = 0
    positions.append(position)

df["position"] = positions
df["strategy_ret"] = df["position"].shift(1) * df["Close"].pct_change()
df["buyhold_ret"]  = df["Close"].pct_change()

# Cumulative returns
df["strategy_cum"] = (1 + df["strategy_ret"].fillna(0)).cumprod()
df["buyhold_cum"]  = (1 + df["buyhold_ret"].fillna(0)).cumprod()

print(f"Strategy CAGR : {df['strategy_cum'].iloc[-1]**(1/9):.2%}")
print(f"Buy&Hold CAGR : {df['buyhold_cum'].iloc[-1]**(1/9):.2%}")
```

---

## 9. Strategy Example 2 — Reliance vs TCS Pairs Trade (Mean Reversion)

**Hypothesis:** RELIANCE and TCS are both NIFTY 50 heavyweights correlated with the broader market. Their spread mean-reverts when it deviates.

**Signal:**
- Compute z-score of the price spread.
- Enter when z-score > +2 (sell RELIANCE, buy TCS) or < −2 (buy RELIANCE, sell TCS).
- Exit when z-score reverts toward 0.

```python
import yfinance as yf
import numpy as np
import pandas as pd
from statsmodels.regression.linear_model import OLS
from statsmodels.tools import add_constant

# Fetch data
tickers = ["RELIANCE.NS", "TCS.NS"]
raw = yf.download(tickers, start="2019-01-01", end="2024-12-31")["Adj Close"].dropna()
rel = raw["RELIANCE.NS"]
tcs = raw["TCS.NS"]

# OLS hedge ratio: RELIANCE = β·TCS + α + ε
X = add_constant(tcs)
model = OLS(rel, X).fit()
beta = model.params["TCS.NS"]
print(f"Hedge ratio β = {beta:.4f}")

# Spread and z-score
spread = rel - beta * tcs
window = 60   # 60-day rolling z-score
z_score = (spread - spread.rolling(window).mean()) / spread.rolling(window).std()

# Signals
signal = pd.Series(0, index=z_score.index)
signal[z_score >  2] = -1   # short RELIANCE, long TCS
signal[z_score < -2] =  1   # long RELIANCE, short TCS
signal[z_score.abs() < 0.5] = 0  # exit zone

# Spread P&L (long +1 unit of spread = long REL, short β TCS)
spread_ret = spread.pct_change()
strategy_ret = signal.shift(1) * spread_ret

sharpe = strategy_ret.mean() / strategy_ret.std() * np.sqrt(252)
print(f"Pairs Sharpe Ratio: {sharpe:.2f}")
```

---

## 10. Strategy Example 3 — RSI Mean Reversion on HDFC Bank

**Hypothesis:** HDFC Bank is a large, liquid stock. Extreme oversold readings on daily RSI recover quickly.

**Rules:**
- RSI(14) < 30 and price is above 200-day SMA (uptrend filter) → BUY
- Exit after 5 days, or if RSI > 60, or if stop-loss of −3% hit

```python
import yfinance as yf
import numpy as np
import pandas as pd

df = yf.download("HDFCBANK.NS", start="2015-01-01", end="2024-12-31")
df["Close"] = df["Adj Close"]

# Indicators
def rsi(series, period=14):
    delta = series.diff()
    gain  = delta.clip(lower=0).rolling(period).mean()
    loss  = (-delta.clip(upper=0)).rolling(period).mean()
    return 100 - 100 / (1 + gain/loss)

df["RSI"]    = rsi(df["Close"])
df["SMA200"] = df["Close"].rolling(200).mean()

# Backtest
trades = []
in_trade = False
entry_price, entry_date, hold_days = 0, None, 0

for date, row in df.iterrows():
    if in_trade:
        hold_days += 1
        ret = (row["Close"] - entry_price) / entry_price
        if hold_days >= 5 or row["RSI"] > 60 or ret < -0.03:
            trades.append({"entry": entry_date, "exit": date, "return": ret})
            in_trade = False
    else:
        if (row["RSI"] < 30) and (row["Close"] > row["SMA200"]) and not pd.isna(row["RSI"]):
            in_trade = True
            entry_price = row["Close"]
            entry_date  = date
            hold_days   = 0

results = pd.DataFrame(trades)
if not results.empty:
    print(f"Total trades     : {len(results)}")
    print(f"Win rate         : {(results['return'] > 0).mean():.2%}")
    print(f"Avg return/trade : {results['return'].mean():.2%}")
    print(f"Total return     : {(1 + results['return']).prod() - 1:.2%}")
```

---

## 11. Performance Metrics — What to Measure

Never judge a strategy by return alone.

```python
def strategy_metrics(returns, risk_free=0.06):
    """
    returns: pandas Series of daily returns
    risk_free: annualised risk-free rate (Indian ~6% for T-bills)
    """
    rf_daily = risk_free / 252

    sharpe = (returns.mean() - rf_daily) / returns.std() * np.sqrt(252)

    # Sortino — only penalises downside vol
    downside = returns[returns < rf_daily].std()
    sortino  = (returns.mean() - rf_daily) / downside * np.sqrt(252)

    # Max Drawdown
    cum = (1 + returns.fillna(0)).cumprod()
    rolling_max = cum.cummax()
    drawdown    = (cum - rolling_max) / rolling_max
    max_dd      = drawdown.min()

    # Calmar Ratio = CAGR / |Max Drawdown|
    n_years = len(returns) / 252
    cagr    = cum.iloc[-1] ** (1/n_years) - 1
    calmar  = cagr / abs(max_dd)

    return {
        "CAGR"          : f"{cagr:.2%}",
        "Sharpe"        : f"{sharpe:.2f}",
        "Sortino"       : f"{sortino:.2f}",
        "Max Drawdown"  : f"{max_dd:.2%}",
        "Calmar Ratio"  : f"{calmar:.2f}",
        "Avg Daily Ret" : f"{returns.mean():.4%}",
        "Daily Volatility": f"{returns.std():.4%}",
    }
```

| Metric | What it tells you | Good threshold |
|---|---|---|
| **Sharpe Ratio** | Return per unit of total risk | > 1.0 is acceptable, > 2.0 is strong |
| **Sortino Ratio** | Return per unit of downside risk | > 1.5 |
| **Max Drawdown** | Worst peak-to-trough loss | Depends on strategy; < 20% for daily |
| **Calmar Ratio** | CAGR divided by max drawdown | > 1.0 |
| **Win Rate** | % of trades that profit | Meaningless without avg win/loss ratio |

---

## 12. Backtesting Pitfalls (NSE/BSE Specific)

These mistakes will make a losing strategy look like a winner.

### Survivorship Bias

The NIFTY 50 today is not the same 50 stocks as in 2010. Companies get removed after they crash. If you only backtest current index constituents, you exclude all the stocks that went to zero.

**Fix:** Use point-in-time index membership data (available from NSE archives).

### Look-Ahead Bias

Using data in your signal that you wouldn't have known at the time of the trade.

```python
# WRONG — uses today's closing price to generate today's signal
df["signal"] = df["Close"] > df["SMA_50"]

# CORRECT — shift by 1 so signal is generated at close, trade executes next open
df["signal"] = (df["Close"] > df["SMA_50"]).shift(1)
```

### Transaction Costs on Indian Markets

| Cost | NSE Equity | NSE F&O |
|---|---|---|
| Brokerage | 0.01–0.03% or flat ₹20/order | ₹20/order (flat) |
| STT | 0.1% on sell side (delivery) | 0.0125% on sell (futures) |
| Exchange charges | 0.00335% | 0.0019% |
| SEBI charges | 0.0001% | 0.0001% |
| GST on brokerage | 18% of brokerage | 18% |
| Stamp duty | 0.015% on buy | 0.002% |
| **Effective round-trip** | **~0.25–0.5%** | **~0.05%** |

For a strategy that trades daily, 0.5% per round trip = ~125% per year in costs alone. This is why high-frequency trading in delivery equity is nearly impossible for small capital.

**Fix:** Always subtract realistic transaction costs from every trade in your backtest.

### Circuit Breakers and Illiquidity

NSE applies 5%, 10%, 20% circuit breakers on individual stocks. Strategies that rely on exiting at a specific price may be unable to do so.

Always filter for minimum daily volume (e.g., > ₹5 crore turnover) before including a stock in your universe.

---

## 13. Walk-Forward Validation

A single backtest is not enough. Use walk-forward to simulate live deployment.

```
Total data: 2015–2024  (10 years)

Training window : 3 years   → optimise parameters
Testing window  : 1 year    → out-of-sample evaluation
Step forward    : 1 year    → repeat

Folds:
  Train 2015–2017 → Test 2018
  Train 2016–2018 → Test 2019
  Train 2017–2019 → Test 2020
  ...and so on
```

```python
def walk_forward_test(df, strategy_fn, train_years=3, test_years=1):
    results = []
    start_year = df.index.year.min()
    end_year   = df.index.year.max()

    for fold_start in range(start_year, end_year - train_years - test_years + 2):
        train_end  = fold_start + train_years
        test_end   = train_end + test_years

        train = df[df.index.year <  train_end]
        test  = df[(df.index.year >= train_end) & (df.index.year < test_end)]

        # optimise on train, evaluate on test
        best_params = strategy_fn.optimise(train)
        test_perf   = strategy_fn.evaluate(test, best_params)
        results.append({"period": f"{train_end}–{test_end}", **test_perf})

    return pd.DataFrame(results)
```

If out-of-sample performance is consistently positive across folds, the strategy has a genuine edge. If it collapses out-of-sample, the backtest was overfit.

---

## 14. Putting It All Together — Strategy Development Workflow

```
Step 1 — UNIVERSE SELECTION
  Pick liquid stocks: NIFTY 50, NIFTY 100, or sector index.
  Filter: market cap > ₹5000 Cr, avg daily volume > ₹20 Cr.

Step 2 — DATA ACQUISITION
  yfinance for free exploration.
  Kite Connect / True Data for production.
  Adjust for splits, bonuses, dividends.

Step 3 — EDA
  Check return distributions, autocorrelation, volatility clustering.
  Identify regime shifts (COVID crash, 2022 rate hikes).

Step 4 — HYPOTHESIS GENERATION
  "Large-cap IT stocks exhibit momentum over 3-month windows."
  "Mid-cap stocks revert after gap-down opens above 200 SMA."
  The hypothesis must have an economic rationale, not just a pattern.

Step 5 — INDICATOR / SIGNAL CONSTRUCTION
  Implement with .shift(1) to prevent look-ahead bias.
  Keep it simple — 1–2 signals beat over-engineered systems.

Step 6 — BACKTEST
  Include transaction costs (use 0.1% round-trip for daily NSE strategies).
  Compute Sharpe, Sortino, max drawdown, Calmar.

Step 7 — STRESS TEST
  How did it perform in March 2020 (COVID crash)?
  How did it perform in 2022 (rate hike selloff)?
  If it blew up in a known crisis, it will blow up in the next one.

Step 8 — WALK-FORWARD VALIDATION
  Confirm the edge is not parameter-overfitted.

Step 9 — PAPER TRADE
  Run the strategy live without real money for 1–3 months.
  Compare live signals with backtest predictions.

Step 10 — DEPLOY
  Use Kite Connect / Upstox API for automated order placement.
  Monitor in real time. Kill switch ready.
```

---

## Summary

```
Data sources     → yfinance (free), Kite Connect (intraday), True Data (tick)
Symbols          → RELIANCE.NS, TCS.NS, ^NSEI for NIFTY 50
Cleaning         → Adjust prices, handle gaps, check stationarity
EDA              → Return distribution, autocorrelation, ADF test
Indicators       → SMA, EMA, RSI, MACD, Bollinger Bands, ATR
Strategies       → Momentum (breakout), mean reversion (pairs/RSI), carry
Performance      → Sharpe > 1, max drawdown acceptable, Calmar > 1
Pitfalls         → Survivorship bias, look-ahead bias, ignoring costs
Validation       → Walk-forward, out-of-sample, paper trading
```

Data without a hypothesis is noise. A hypothesis without data is speculation. The edge lives in the disciplined intersection of both.
