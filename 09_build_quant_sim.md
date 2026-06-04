# 09 — Building Your Quant Simulator

This chapter ties everything together: the theory from files 01-08 + the existing C++ simulation infrastructure in this repo.

---

## What This Repo Already Has

```
/Users/ace/Documents/collage/HFT/
├── src/
│   ├── matching.cpp        → Price-time priority matching engine
│   ├── orderbook.cpp       → Order book (add/cancel/modify + best bid/ask)
│   ├── engine.cpp          → Core simulation loop
│   ├── strategy.cpp        → Strategy interface
│   ├── market.cpp          → Market state, price feeds
│   ├── feed_replay.cpp     → Historical feed replay
│   ├── itch_parser.cpp     → ITCH 5.0 binary protocol parser
│   ├── benchmark.cpp       → Latency benchmarking
│   └── gui_main.cpp        → ImGui visualization
├── include/hft_simulator/
│   ├── orderbook.hpp       → Lock-free order book
│   ├── matching.hpp        → Matching engine interface
│   ├── order.hpp           → Order types (limit, market, IOC, FOK)
│   ├── strategy.h          → Strategy base class
│   ├── risk.hpp            → Risk manager
│   ├── metrics.hpp         → Sharpe, drawdown, PnL metrics
│   └── latency_hist.hpp    → Latency histogram
├── python/quantsim/
│   ├── strategy.py         → Python strategy API
│   ├── backtester.py       → Backtest runner
│   ├── analytics.py        → Performance analytics
│   └── dashboard.py        → Visualization
└── examples/
    ├── market_maker.py     → Market making strategy example
    ├── stat_arb.py         → Statistical arbitrage example
    ├── signal_strategy.py  → Signal-based strategy
    └── twap.py             → TWAP execution algorithm
```

You have a solid foundation. The C++ core handles microsecond-level simulation. Python wraps it for strategy research.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   Python Layer                       │
│  Strategy research + backtesting + analytics         │
│  quantsim.backtester / quantsim.strategy             │
└────────────────┬────────────────────────────────────┘
                 │ pybind11 bindings (python/bindings.cpp)
┌────────────────▼────────────────────────────────────┐
│                   C++ Core                           │
│  Order book + Matching engine + Engine loop          │
│  Latency benchmarks + Risk manager                   │
└─────────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────┐
│                   Data Sources                       │
│  ITCH feed replay / CSV / WebSocket (add this)       │
└─────────────────────────────────────────────────────┘
```

---

## Building and Running

```bash
# Build
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)

# Run simulation
./build/run_simulation

# Run benchmark (measure order book + matching latency)
./build/run_benchmark

# Python interface
pip install -e .
python run_sim.py

# GUI
./build/hft_gui
```

---

## Writing Your First Strategy (Python)

The cleanest starting point. The Python API exposes everything you need.

```python
# examples/my_strategy.py
from quantsim import strategy, backtester
import numpy as np

class MomentumStrategy(strategy.Strategy):
    """
    Buy when short EMA > long EMA, sell otherwise.
    Simplest trend-following strategy to verify the pipeline.
    """
    
    def __init__(self, fast=20, slow=60):
        super().__init__()
        self.fast = fast
        self.slow = slow
        self.prices = []
    
    def on_market_data(self, event):
        """Called on every order book update."""
        mid = (event.best_bid + event.best_ask) / 2
        self.prices.append(mid)
        
        if len(self.prices) < self.slow:
            return  # not enough data yet
        
        prices_arr = np.array(self.prices[-self.slow:])
        fast_ema = self._ema(prices_arr[-self.fast:])
        slow_ema = self._ema(prices_arr)
        
        if fast_ema > slow_ema:
            self.target_position = 1.0  # full long
        else:
            self.target_position = -1.0  # full short
    
    def on_fill(self, fill):
        """Called when your order fills."""
        self.position += fill.qty if fill.is_buy else -fill.qty
    
    def _ema(self, prices, span=None):
        span = span or len(prices)
        alpha = 2 / (span + 1)
        ema = prices[0]
        for p in prices[1:]:
            ema = p * alpha + ema * (1 - alpha)
        return ema

# Run backtest
if __name__ == "__main__":
    bt = backtester.Backtester(
        data_path="data/BTC_USDT_ticks.parquet",
        strategy=MomentumStrategy(fast=20, slow=60),
        initial_capital=100_000,
        transaction_cost_bps=5,
    )
    results = bt.run()
    
    print(f"Sharpe: {results.sharpe:.2f}")
    print(f"Max Drawdown: {results.max_drawdown:.1%}")
    print(f"Total Return: {results.total_return:.1%}")
    results.plot_equity_curve()
```

---

## Writing a C++ Strategy

For strategies that need sub-millisecond execution:

```cpp
// src/my_strategy.cpp
#include "hft_simulator/strategy.h"
#include "hft_simulator/orderbook.hpp"
#include "hft_simulator/order.hpp"

class MarketMakingStrategy : public Strategy {
    double target_spread_bps_ = 2.0;
    double max_inventory_ = 1.0;  // 1 BTC max position
    double inventory_ = 0.0;
    
public:
    void on_book_update(const OrderBook& book) override {
        double mid = book.mid_price();
        double spread = target_spread_bps_ / 10000.0 * mid;
        
        // Inventory skew (Avellaneda-Stoikov)
        double gamma = 0.5;
        double vol_estimate = 0.01;  // replace with rolling vol
        double skew = gamma * vol_estimate * inventory_;
        
        double reservation = mid - skew;
        double bid_price = reservation - spread / 2;
        double ask_price = reservation + spread / 2;
        
        // Cancel existing quotes, place new ones
        cancel_all_orders();
        
        if (std::abs(inventory_) < max_inventory_) {
            place_limit_order(Side::BUY,  bid_price, 0.01);  // 0.01 BTC
            place_limit_order(Side::SELL, ask_price, 0.01);
        }
    }
    
    void on_fill(const Fill& fill) override {
        inventory_ += fill.is_buy ? fill.qty : -fill.qty;
        log_pnl(fill);
    }
};
```

---

## Adding Crypto Data

The existing `feed_replay.cpp` handles ITCH format (equities). To use real crypto data:

### Option 1: CSV/Parquet Historical Data

```python
# Download historical tick data
# Sources:
#   - Tardis.dev (paid, best quality, $100-500/month)
#   - Binance Data portal (free, OHLCV only, limited tick)
#   - Kaiko (institutional, $$$)
#   - CoinAPI (REST download, free tier available)

import ccxt
import pandas as pd

def download_binance_ohlcv(symbol="BTC/USDT", timeframe="1m", since="2024-01-01"):
    exchange = ccxt.binance()
    since_ms = exchange.parse8601(since)
    all_ohlcv = []
    
    while True:
        ohlcv = exchange.fetch_ohlcv(symbol, timeframe, since=since_ms, limit=1000)
        if not ohlcv:
            break
        all_ohlcv.extend(ohlcv)
        since_ms = ohlcv[-1][0] + 1
        if since_ms > exchange.milliseconds():
            break
    
    df = pd.DataFrame(all_ohlcv, columns=["timestamp", "open", "high", "low", "close", "volume"])
    df["timestamp"] = pd.to_datetime(df["timestamp"], unit="ms")
    return df.set_index("timestamp")
```

### Option 2: Live WebSocket Feed

```python
# Create a feed adapter that transforms Binance WebSocket 
# into the format expected by the C++ engine

import asyncio
import websockets
import json
from python.bindings import FeedHandler  # C++ extension

async def binance_feed_to_engine(symbol="BTCUSDT"):
    handler = FeedHandler()  # connects to C++ order book
    url = f"wss://stream.binance.com:9443/ws/{symbol.lower()}@depth20@100ms"
    
    async with websockets.connect(url) as ws:
        async for raw in ws:
            msg = json.loads(raw)
            # Convert Binance format → internal format → C++ book update
            for price, qty in msg["b"]:
                handler.update_bid(float(price), float(qty))
            for price, qty in msg["a"]:
                handler.update_ask(float(price), float(qty))
            handler.tick()  # trigger strategy on_book_update
```

---

## Extending the Order Book for Crypto

The existing order book (`include/hft_simulator/orderbook.hpp`) is designed for equities. Crypto differences to handle:

### 1. Floating Point Prices

Crypto prices have many decimal places (0.00001234 BTC). Use fixed-point with sufficient precision:

```cpp
// Instead of double price:
// Store as integer with fixed decimal places
// price_int = price_double * 1e8 (8 decimal places)
using Price = int64_t;  // satoshi-precision
const int64_t PRICE_MULTIPLIER = 100000000;  // 1e8

Price to_price(double p) { return static_cast<Price>(p * PRICE_MULTIPLIER); }
double from_price(Price p) { return static_cast<double>(p) / PRICE_MULTIPLIER; }
```

### 2. Perpetual Futures Accounting

Add funding rate tracking:

```cpp
struct PerpAccount {
    double position;       // net position in BTC
    double entry_value;    // dollar value at entry
    double unrealized_pnl;
    double funding_paid;   // cumulative funding
    
    void apply_funding(double funding_rate, double mark_price) {
        double payment = funding_rate * position * mark_price;
        funding_paid += payment;
        unrealized_pnl -= payment;
    }
};
```

### 3. Multi-Symbol Support

Track multiple BTC/ETH/SOL pairs simultaneously:

```cpp
struct MultiBookEngine {
    std::unordered_map<std::string, OrderBook> books;
    
    void on_update(const std::string& symbol, const BookDelta& delta) {
        books[symbol].apply(delta);
        signal_engine_.on_update(symbol, books[symbol]);
    }
};
```

---

## Complete Crypto Market Making Example

Putting it all together:

```python
# examples/crypto_mm.py
import asyncio
import numpy as np
from collections import deque
from quantsim.strategy import CryptoStrategy

class CryptoMarketMaker(CryptoStrategy):
    """
    Market making strategy for crypto perpetuals.
    Quotes symmetric spread around skewed mid price.
    Manages inventory via quote asymmetry.
    """
    
    def __init__(self, 
                 target_spread_bps=2.5,
                 max_inventory_usd=5000,
                 vol_lookback=100,
                 risk_aversion=0.5):
        super().__init__()
        self.target_spread = target_spread_bps / 10000
        self.max_inv = max_inventory_usd
        self.gamma = risk_aversion
        self.returns = deque(maxlen=vol_lookback)
        self.last_mid = None
    
    def on_book_update(self, book):
        mid = book.mid
        if mid is None:
            return
        
        # Update vol estimate
        if self.last_mid:
            self.returns.append((mid - self.last_mid) / self.last_mid)
        self.last_mid = mid
        
        vol = np.std(self.returns) if len(self.returns) > 10 else 0.001
        
        # Compute inventory in dollars
        inv_usd = self.position * mid
        inv_ratio = inv_usd / self.max_inv  # [-1, 1]
        
        # Skew reservation price away from inventory
        inv_skew = self.gamma * vol * inv_ratio * mid
        reservation = mid - inv_skew
        
        # Spread: min of target or volatility-scaled
        half_spread = max(
            self.target_spread * mid / 2,
            vol * mid * 0.5  # at least 0.5 × per-tick vol
        )
        
        bid = reservation - half_spread
        ask = reservation + half_spread
        
        # Don't quote if inventory maxed
        quote_size = 0.001  # 0.001 BTC per quote
        if inv_usd > self.max_inv * 0.8:
            bid_size = 0         # don't buy more
            ask_size = quote_size
        elif inv_usd < -self.max_inv * 0.8:
            bid_size = quote_size
            ask_size = 0         # don't sell more
        else:
            bid_size = quote_size
            ask_size = quote_size
        
        # Cancel + requote
        self.cancel_all()
        if bid_size > 0:
            self.place_limit("BUY", bid, bid_size)
        if ask_size > 0:
            self.place_limit("SELL", ask, ask_size)
    
    def on_fill(self, fill):
        delta = fill.qty if fill.is_buy else -fill.qty
        self.position += delta
        self.realized_pnl += fill.realized_pnl
```

---

## Backtesting Pipeline

```python
# Full backtesting workflow
from quantsim.backtester import Backtester, BacktestConfig
from quantsim.analytics import TearSheet

config = BacktestConfig(
    data_path="data/BTCUSDT_perp_2024_ticks.parquet",
    initial_capital=100_000,
    transaction_costs={
        "maker_bps": 0,           # post-only
        "taker_bps": 5,           # 0.05%
        "market_impact_factor": 0.5,
    },
    slippage_model="half_spread",  # conservative: fill at half-spread away from mid
    funding_rate_data="data/BTCUSDT_funding_2024.csv",
    start="2024-01-01",
    end="2024-06-01",
)

bt = Backtester(strategy=CryptoMarketMaker(), config=config)
results = bt.run()

# Generate tearsheet
sheet = TearSheet(results)
sheet.print_summary()
# Output:
#   Sharpe Ratio (annualized): 2.34
#   Sortino Ratio: 3.12
#   Max Drawdown: -4.2%
#   Daily Turnover: 8.3x
#   Win Rate: 54.3%
#   Avg Trade Duration: 4.2s
#   Total Trades: 18,234
sheet.plot()  # equity curve, drawdown, PnL heatmap
```

---

## Signal Research Workflow

```python
# How to research a new signal: step by step

import pandas as pd
import numpy as np
from quantsim.analytics import IC, ICIR, information_ratio

# 1. Load tick data
ticks = pd.read_parquet("data/BTCUSDT_ticks.parquet")

# 2. Compute features
ticks["bid_vol"] = ticks[["bid_qty_0","bid_qty_1","bid_qty_2","bid_qty_3","bid_qty_4"]].sum(axis=1)
ticks["ask_vol"] = ticks[["ask_qty_0","ask_qty_1","ask_qty_2","ask_qty_3","ask_qty_4"]].sum(axis=1)
ticks["imbalance"] = (ticks["bid_vol"] - ticks["ask_vol"]) / (ticks["bid_vol"] + ticks["ask_vol"])

# 3. Compute forward returns (various horizons)
for h in [10, 30, 60, 300]:  # seconds
    ticks[f"fwd_ret_{h}s"] = ticks["mid"].pct_change(h).shift(-h)

# 4. Evaluate signal at each horizon
for h in [10, 30, 60, 300]:
    ic = IC(ticks["imbalance"], ticks[f"fwd_ret_{h}s"])
    print(f"IC at {h}s: {ic:.4f}")

# Output:
# IC at 10s: 0.0823
# IC at 30s: 0.0612
# IC at 60s: 0.0341
# IC at 300s: 0.0089
# → signal decays fast, best used at 10-30s horizon

# 5. Build strategy around strongest signal horizon
# 6. Backtest → evaluate Sharpe, turnover, drawdown
```

---

## Adding Crypto Features Roadmap

What to add to the existing repo for full crypto support:

**Short term** (1-2 weeks):
- [ ] WebSocket feed handler (Binance L2 order book stream)
- [ ] Floating-point price precision fix (satoshi-level)
- [ ] Multiple symbol support in engine
- [ ] Binance REST API order management

**Medium term** (1-2 months):
- [ ] Perpetual funding rate accounting
- [ ] Cross-exchange order book consolidation
- [ ] Liquidation feed monitoring
- [ ] Walk-forward backtesting framework
- [ ] Transaction cost model with exchange fees + spread

**Advanced** (3-6 months):
- [ ] FPGA simulation model (ideal for bench comparison)
- [ ] On-chain data integration (funding arb signals)
- [ ] MEV simulation (fork mainnet + simulate)
- [ ] Options pricing (Black-Scholes/binomial for crypto options)
- [ ] Portfolio-level risk (VaR, CVaR across positions)

---

## Debugging and Validation

### Sanity Checks

Before trusting any backtest result:

```python
def validate_backtest(results):
    checks = []
    
    # 1. No lookahead: signals must be computed before the bar they trade
    checks.append(("No lookahead", results.signal_lag_correct))
    
    # 2. No weekend anomalies (for crypto, every day is valid)
    daily_counts = results.trades.groupby(results.trades.index.date).size()
    checks.append(("Even daily activity", daily_counts.std() / daily_counts.mean() < 2))
    
    # 3. Transaction costs applied
    checks.append(("TC applied", results.gross_pnl > results.net_pnl))
    
    # 4. Realistic fill prices
    checks.append(("Fill within spread", results.fill_prices_within_spread))
    
    # 5. P&L attribution makes sense
    pnl_from_fees = results.total_fees
    pnl_from_trading = results.gross_pnl
    checks.append(("Gross PnL > fees", pnl_from_trading > 0))
    
    for name, passed in checks:
        print(f"{'✓' if passed else '✗'} {name}")
```

### Performance Profiling

```bash
# Profile C++ components
perf record -g ./build/run_simulation
perf report

# Or use callgrind for detailed instruction-level profiling
valgrind --tool=callgrind ./build/run_simulation
kcachegrind callgrind.out.*

# For Python layer
python -m cProfile -o profile.out run_sim.py
python -m pstats profile.out
```

---

## Final Architecture: Production-Ready Crypto Quant System

```
┌──────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                                │
│  Binance WS → Feed Handler → Normalized Book Store               │
│  CryptoQuant API → On-chain signal feed                          │
│  Funding rate feed → Carry signal                                 │
└────────────────────────┬─────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────┐
│                        SIGNAL LAYER                               │
│  Order flow imbalance signal (30s horizon)                        │
│  Cross-exchange basis signal                                      │
│  Funding rate Z-score signal                                      │
│  On-chain exchange flow signal (daily)                            │
└────────────────────────┬─────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────┐
│                      STRATEGY LAYER                               │
│  Market making (C++, millisecond)                                 │
│  Stat arb (Python, second-minute)                                 │
│  Carry (Python, 8h rebalance)                                     │
└────────────────────────┬─────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────┐
│                       RISK LAYER                                  │
│  Position limits per symbol                                       │
│  Daily drawdown limit → auto-shutdown                             │
│  Exchange exposure limit                                          │
│  Correlation-adjusted notional limits                             │
└────────────────────────┬─────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────┐
│                    EXECUTION LAYER                                │
│  Order management (C++, lock-free OMS)                           │
│  Binance API (WebSocket order entry for low latency)             │
│  Post-only limit orders (maker rebate)                            │
└──────────────────────────────────────────────────────────────────┘
```

Start simple: one strategy, one symbol, paper trading. Validate every layer before adding complexity.

---

## Resources

**Books** (worth reading):
- *Algorithmic Trading and DMA* — Barry Johnson (market microstructure bible)
- *High-Frequency Trading* — Aldridge (accessible overview)
- *Advances in Financial Machine Learning* — de Prado (ML for finance, correct backtesting)
- *Options, Futures, and Other Derivatives* — Hull (derivatives fundamentals)
- *Market Microstructure Theory* — O'Hara (academic but foundational)
- *Flash Boys* — Lewis (narrative, not technical, but useful context)

**Papers**:
- Avellaneda & Stoikov (2008): *High-frequency trading in a limit order book* — market making model
- Almgren & Chriss (2001): *Optimal execution of portfolio transactions* — execution model
- Kyle (1985): *Continuous auctions and insider trading* — price impact model
- Glosten & Milgrom (1985): *Bid, ask and transaction prices* — adverse selection model

**Online**:
- Quantopian lecture series (free, archived on YouTube)
- QuantLib documentation
- CME Group education
- Binance Academy (crypto-specific)

Go build. Good luck.
