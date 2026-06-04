# Backtesting Engine Design

A backtesting engine simulates strategy performance on historical data. For quant developers, building the engine correctly is as important as the strategy itself — a realistic backtester prevents overfitting and surprises in production. This chapter covers event-driven architecture, realistic fill modeling, and validation methodology.

---

## 1. Event-Driven vs. Vectorized Backtesting

### 1.1 Vectorized Backtesting

Operates on arrays of prices directly. Simple, fast, but cannot model realistic execution:

```python
# Vectorized: entire series at once
returns   = prices.pct_change()
signal    = (fast_ma > slow_ma).astype(int)        # 1 or 0
positions = signal.shift(1)                         # shift to avoid look-ahead
strategy_returns = positions * returns
```

**Pros:** 100x faster than event-driven. Useful for initial alpha research.

**Cons:**
- No realistic fill modeling (assumes fills at close price).
- No partial fills, queue position, or market impact.
- Cannot model latency or order management.
- Dangerous for HFT strategies — results are meaningless.

### 1.2 Event-Driven Backtesting

Processes events sequentially, one at a time, in chronological order. Strategy code is nearly identical to live execution code:

```
MarketDataEvent → Strategy → OrderEvent → SimulatedExchange → FillEvent → Portfolio
```

**Pros:** Realistic execution modeling, reusable strategy logic, explicit latency simulation.

**Cons:** Slower (100-1000x). More complex to implement correctly.

**Rule:** Use vectorized for alpha discovery. Switch to event-driven before any capital commitment.

---

## 2. Event-Driven Architecture

### 2.1 Event Types

```python
from dataclasses import dataclass, field
from enum import Enum, auto
from typing import Literal

class EventType(Enum):
    MARKET_DATA = auto()
    ORDER       = auto()
    FILL        = auto()
    CANCEL      = auto()

@dataclass
class MarketDataEvent:
    type:   EventType = EventType.MARKET_DATA
    ts_ns:  int       = 0
    symbol: str       = ""
    bid:    float     = 0.0
    ask:    float     = 0.0
    bid_sz: int       = 0
    ask_sz: int       = 0
    last:   float     = 0.0

@dataclass
class OrderEvent:
    type:     EventType                       = EventType.ORDER
    ts_ns:    int                             = 0
    symbol:   str                             = ""
    side:     Literal['B', 'S']               = 'B'
    order_type: Literal['LIMIT', 'MARKET']    = 'LIMIT'
    price:    float                           = 0.0
    qty:      int                             = 0
    order_id: str                             = ""

@dataclass
class FillEvent:
    type:      EventType                      = EventType.FILL
    ts_ns:     int                            = 0
    symbol:    str                            = ""
    side:      Literal['B', 'S']              = 'B'
    fill_price: float                         = 0.0
    fill_qty:  int                            = 0
    order_id:  str                            = ""
    commission: float                         = 0.0
```

### 2.2 Core Engine Loop

```python
import heapq
from collections import deque

class BacktestEngine:
    def __init__(self, strategy, exchange_sim, portfolio):
        self.strategy     = strategy
        self.exchange     = exchange_sim
        self.portfolio    = portfolio
        self._event_queue = deque()

    def run(self, market_data: list[MarketDataEvent]):
        for md_event in market_data:
            # 1. Deliver market data to exchange sim (for fill matching)
            fills = self.exchange.on_market_data(md_event)
            for fill in fills:
                self._event_queue.append(fill)

            # 2. Deliver market data to strategy
            order_events = self.strategy.on_market_data(md_event)
            for order in order_events:
                self._event_queue.append(order)

            # 3. Process all queued events
            while self._event_queue:
                event = self._event_queue.popleft()
                if event.type == EventType.ORDER:
                    self.exchange.on_order(event)
                elif event.type == EventType.FILL:
                    self.portfolio.on_fill(event)
                    self.strategy.on_fill(event)
```

---

## 3. Simulated Exchange: Realistic Fill Modeling

This is where most backtests fail. Overly optimistic fill assumptions inflate returns.

### 3.1 Limit Order Fill Logic

```python
class SimulatedExchange:
    def __init__(self, latency_ns: int = 10_000,
                  queue_position_model: str = 'back'):
        self.latency_ns = latency_ns     # simulated round-trip latency
        self.open_orders: dict[str, OrderEvent] = {}
        self.queue_model = queue_position_model

    def on_order(self, order: OrderEvent):
        # Simulate network + processing latency
        effective_ts = order.ts_ns + self.latency_ns
        order.ts_ns  = effective_ts
        self.open_orders[order.order_id] = order

    def on_market_data(self, md: MarketDataEvent) -> list[FillEvent]:
        fills = []
        for oid, order in list(self.open_orders.items()):
            if order.ts_ns > md.ts_ns:
                continue  # order arrived after this tick

            fill = self._try_fill(order, md)
            if fill:
                fills.append(fill)
                del self.open_orders[oid]
        return fills

    def _try_fill(self, order: OrderEvent,
                   md: MarketDataEvent) -> FillEvent | None:
        if order.order_type == 'MARKET':
            fill_price = md.ask if order.side == 'B' else md.bid
            return self._make_fill(order, fill_price, order.qty, md.ts_ns)

        if order.side == 'B' and order.price >= md.ask:
            # Buy limit: fills at ask (aggressive cross)
            fill_price = md.ask
            if self.queue_model == 'back':
                # Assume we're at back of queue — only fill if market
                # traded through our price (last trade < bid)
                if md.last < order.price:
                    return self._make_fill(order, fill_price, order.qty, md.ts_ns)
            else:
                return self._make_fill(order, fill_price, order.qty, md.ts_ns)

        if order.side == 'S' and order.price <= md.bid:
            fill_price = md.bid
            if self.queue_model == 'back':
                if md.last > order.price:
                    return self._make_fill(order, fill_price, order.qty, md.ts_ns)
            else:
                return self._make_fill(order, fill_price, order.qty, md.ts_ns)

        return None

    def _make_fill(self, order, price, qty, ts) -> FillEvent:
        commission = qty * price * 0.0001  # 1bps commission
        return FillEvent(ts_ns=ts, symbol=order.symbol, side=order.side,
                          fill_price=price, fill_qty=qty,
                          order_id=order.order_id, commission=commission)
```

**Queue position model:** `'back'` is the conservative default — assume you're last in queue, fill only when market trades through your price. `'pro_rata'` models CME-style allocation.

### 3.2 Partial Fills

For strategies trading large sizes relative to book depth, model partial fills:

```python
def _try_fill_with_depth(self, order: OrderEvent,
                          md: MarketDataEvent) -> FillEvent | None:
    available_size = md.ask_sz if order.side == 'B' else md.bid_sz
    fill_qty = min(order.qty, available_size)

    if fill_qty == 0:
        return None

    fill_price = md.ask if order.side == 'B' else md.bid
    # Update remaining quantity on order
    order.qty -= fill_qty

    if order.qty > 0:
        # Partial fill — leave order open
        self.open_orders[order.order_id] = order

    return self._make_fill(order, fill_price, fill_qty, md.ts_ns)
```

### 3.3 Market Impact Modeling

For larger orders, incorporate the square-root market impact model:

```python
def market_impact_bps(qty: int, adv: float, sigma: float,
                       eta: float = 0.1) -> float:
    """
    Almgren-Chriss linear market impact.
    qty:   order size in shares
    adv:   average daily volume
    sigma: daily volatility
    Returns impact in bps.
    """
    participation = qty / adv
    return eta * sigma * np.sqrt(participation) * 10000
```

Apply impact as slippage on top of quoted price during fill simulation.

---

## 4. Look-Ahead Bias Prevention

Most common and most damaging error in backtesting.

### 4.1 Sources of Look-Ahead Bias

1. **Using future data for normalization:** Computing z-score using the full-sample mean/std.
2. **Point-in-time data errors:** Using today's `shares_outstanding` for a company in 2015.
3. **Survivorship bias:** Testing only on companies that still exist today (winners).
4. **Lookahead in feature engineering:** Computing today's Sharpe using tomorrow's returns.

### 4.2 Enforcing Correct Data Access

Wrap all data access with a timestamp gate:

```python
class PointInTimeDataStore:
    def __init__(self, data: pl.DataFrame):
        # data must have columns: [ts_ns, symbol, ...features...]
        self._data = data.sort("ts_ns")

    def get_features(self, symbol: str, as_of_ns: int,
                      lookback: int) -> pl.DataFrame:
        """Returns only data strictly before as_of_ns."""
        return (
            self._data
            .filter(
                (pl.col("symbol") == symbol) &
                (pl.col("ts_ns") < as_of_ns)   # strict less-than
            )
            .tail(lookback)
        )
```

### 4.3 Cross-Validation: Purging and Embargoes

Standard k-fold CV leaks when training and test sets overlap in a time-series context (due to autocorrelation). Use **Purged K-Fold**:

```python
class PurgedKFold:
    """
    Purge training samples whose labels overlap with the test period.
    Add an embargo period after the test window to prevent leakage.
    """
    def __init__(self, n_splits: int = 5, embargo_pct: float = 0.01):
        self.n_splits    = n_splits
        self.embargo_pct = embargo_pct

    def split(self, X, pred_times: np.ndarray, eval_times: np.ndarray):
        n = len(X)
        fold_size = n // self.n_splits
        embargo   = int(n * self.embargo_pct)

        for i in range(self.n_splits):
            test_start = i * fold_size
            test_end   = test_start + fold_size
            test_idx   = np.arange(test_start, test_end)

            # Purge: remove training samples whose eval_time overlaps test
            purge_mask = eval_times >= pred_times[test_start]
            purge_mask &= pred_times <= eval_times[test_end - 1]

            train_idx = np.concatenate([
                np.arange(0, test_start),
                np.arange(test_end + embargo, n)
            ])
            train_idx = train_idx[~purge_mask[train_idx]]

            yield train_idx, test_idx
```

---

## 5. Performance Metrics

```python
import pandas as pd

def compute_metrics(returns: pd.Series, risk_free_rate: float = 0.05) -> dict:
    daily_rf = risk_free_rate / 252
    excess   = returns - daily_rf

    sharpe   = excess.mean() / excess.std() * np.sqrt(252)

    # Calmar ratio
    cumulative    = (1 + returns).cumprod()
    rolling_max   = cumulative.cummax()
    drawdown      = (cumulative - rolling_max) / rolling_max
    max_drawdown  = drawdown.min()
    calmar        = returns.mean() * 252 / abs(max_drawdown)

    # Sortino (downside deviation)
    downside_std = returns[returns < daily_rf].std() * np.sqrt(252)
    sortino      = (returns.mean() * 252 - risk_free_rate) / downside_std

    # Win rate and profit factor
    wins   = returns[returns > 0]
    losses = returns[returns < 0]
    profit_factor = wins.sum() / abs(losses.sum()) if len(losses) else np.inf

    return {
        "annualized_return": returns.mean() * 252,
        "annualized_vol":    returns.std() * np.sqrt(252),
        "sharpe":            sharpe,
        "sortino":           sortino,
        "calmar":            calmar,
        "max_drawdown":      max_drawdown,
        "win_rate":          len(wins) / len(returns),
        "profit_factor":     profit_factor,
    }
```

### 5.1 Walk-Forward Validation

Never optimize on the full history. Use rolling out-of-sample windows:

```python
def walk_forward_test(strategy_factory, data: pd.DataFrame,
                       train_window: int = 252,
                       test_window: int  = 63) -> pd.Series:
    """
    Returns out-of-sample returns only.
    train_window, test_window: in trading days.
    """
    all_oos_returns = []
    n = len(data)

    for start in range(0, n - train_window - test_window, test_window):
        train = data.iloc[start : start + train_window]
        test  = data.iloc[start + train_window : start + train_window + test_window]

        # Fit parameters on train, evaluate on test
        strategy = strategy_factory()
        strategy.fit(train)
        oos_returns = strategy.backtest(test)
        all_oos_returns.append(oos_returns)

    return pd.concat(all_oos_returns)
```

---

Next Chapter: [Statistical Arbitrage & Mean Reversion](7_stat_arb.md)
