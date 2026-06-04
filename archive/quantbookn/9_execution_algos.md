# Execution Algorithms

Execution algorithms (algos) break large parent orders into smaller child orders to minimize market impact and track a benchmark price. A QD implements and maintains these algos in the execution engine. This chapter covers TWAP, VWAP, Implementation Shortfall, and Smart Order Routing.

---

## 1. Why Execution Algorithms Exist

A large market order for 100,000 shares moves the price against you — market impact. Execution algos trade off:
- **Market impact** (moving price by trading too fast)
- **Timing risk** (price moving against you while trading slowly)

The goal: find the optimal trading schedule that minimizes total execution cost.

---

## 2. TWAP — Time-Weighted Average Price

TWAP splits the parent order into equal-sized slices over equal time intervals. It tracks the time-weighted average price of the benchmark period.

### 2.1 Implementation

```python
import time
from dataclasses import dataclass

@dataclass
class TWAPParams:
    symbol:       str
    total_qty:    int
    side:         str            # 'B' or 'S'
    start_time:   float          # Unix timestamp
    end_time:     float          # Unix timestamp
    num_slices:   int = 10

class TWAPAlgo:
    def __init__(self, params: TWAPParams, gateway):
        self.p            = params
        self.gateway      = gateway
        self.slice_qty    = params.total_qty // params.num_slices
        self.interval     = (params.end_time - params.start_time) / params.num_slices
        self.slices_sent  = 0
        self.filled_qty   = 0
        self.next_fire_ts = params.start_time

    def on_tick(self, ts: float, bid: float, ask: float):
        if ts < self.next_fire_ts:
            return
        if self.slices_sent >= self.p.num_slices:
            return

        remaining_slices = self.p.num_slices - self.slices_sent
        remaining_qty    = self.p.total_qty - self.filled_qty

        # Last slice: send all remaining to avoid underfill
        qty = remaining_qty if self.slices_sent == self.p.num_slices - 1 \
              else self.slice_qty

        price = ask if self.p.side == 'B' else bid
        self.gateway.limit_order(self.p.symbol, self.p.side, qty, price)

        self.slices_sent  += 1
        self.next_fire_ts += self.interval

    def on_fill(self, qty: int):
        self.filled_qty += qty
```

**Use case:** TWAP is appropriate when you have no view on intraday price direction and want to minimize timing risk with predictable execution.

---

## 3. VWAP — Volume-Weighted Average Price

VWAP tracks the volume-weighted average price of the market. The algo sends more volume during historically high-volume periods (open, close) and less during low-volume periods (midday).

### 3.1 Volume Profile Construction

Build an intraday volume profile from historical data:

```python
import polars as pl
import numpy as np

def build_volume_profile(trades: pl.DataFrame,
                          bucket_seconds: int = 300,
                          lookback_days: int = 20) -> np.ndarray:
    """
    Returns normalized volume profile: fraction of daily volume per bucket.
    trades: columns [ts_ns, symbol, size]
    """
    df = trades.with_columns([
        (pl.col("ts_ns") // (bucket_seconds * 1_000_000_000))
            .alias("bucket")
    ])

    profile = (
        df.group_by("bucket")
          .agg(pl.col("size").sum().alias("vol"))
          .sort("bucket")
    )

    volumes = profile["vol"].to_numpy().astype(float)
    return volumes / volumes.sum()   # normalize to fractions

class VWAPAlgo:
    def __init__(self, symbol: str, total_qty: int, side: str,
                  volume_profile: np.ndarray, bucket_seconds: int,
                  start_bucket: int, gateway):
        self.symbol          = symbol
        self.total_qty       = total_qty
        self.side            = side
        self.profile         = volume_profile
        self.bucket_secs     = bucket_seconds
        self.start_bucket    = start_bucket
        self.gateway         = gateway
        self.filled_qty      = 0
        self.current_bucket  = start_bucket

    def on_bucket_start(self, bucket_idx: int, bid: float, ask: float):
        idx = bucket_idx - self.start_bucket
        if idx < 0 or idx >= len(self.profile):
            return

        target_participation = self.profile[idx]
        qty = int(self.total_qty * target_participation)
        qty = max(qty, 1)

        price = ask if self.side == 'B' else bid
        self.gateway.limit_order(self.symbol, self.side, qty, price)
        self.current_bucket = bucket_idx

    def on_fill(self, qty: int):
        self.filled_qty += qty
```

**Variance:** VWAP target is achievable only if actual volume profile matches historical profile. Divergence (e.g., surprise news event causing volume spike) will cause VWAP tracking error.

---

## 4. Implementation Shortfall (IS)

IS minimizes the difference between the **decision price** (price when you decided to trade) and the **actual average fill price**. It explicitly models the tradeoff between market impact and timing risk.

### 4.1 Almgren-Chriss Model

The optimal trading trajectory minimizes:
$$\text{IS} = \underbrace{\eta \sum_{k=1}^{N} v_k^2}_{\text{market impact}} + \underbrace{\lambda \sigma^2 \sum_{k=1}^{N} q_k^2 \Delta t}_{\text{timing risk}}$$

where:
- $v_k$ = shares traded in interval $k$
- $q_k$ = remaining inventory after interval $k$
- $\eta$ = linear market impact coefficient
- $\lambda$ = risk aversion
- $\sigma$ = volatility

The optimal closed-form solution gives a trading schedule:

```python
def almgren_chriss_schedule(total_qty: float, T: float, n_slices: int,
                              sigma: float, eta: float,
                              lambda_risk: float) -> np.ndarray:
    """
    Returns array of trade sizes (one per time slice).
    T: total trading horizon in seconds
    sigma: per-second volatility
    eta: market impact coefficient
    lambda_risk: risk aversion (higher = trade faster)
    """
    dt     = T / n_slices
    kappa  = np.sqrt(lambda_risk * sigma**2 / eta)

    # Optimal trajectory
    times   = np.arange(n_slices + 1) * dt
    q       = total_qty * np.sinh(kappa * (T - times)) / np.sinh(kappa * T)

    # Trade sizes (differences in inventory)
    trades  = -np.diff(q)
    return trades

# Usage:
schedule = almgren_chriss_schedule(
    total_qty=10000,
    T=3600,          # 1 hour
    n_slices=12,     # 5-minute buckets
    sigma=0.0001,    # 1bps/second vol
    eta=0.01,
    lambda_risk=1e-6
)
```

**Calibration:**
- `sigma`: use realized vol from tick data (Ch 5)
- `eta`: regress market impact from historical executions: $\Delta P \propto \eta v$
- `lambda_risk`: tune to desired risk tolerance; higher = more aggressive schedule

### 4.2 IS vs VWAP vs TWAP

| Algo | Optimizes | Best When | Weakness |
|------|-----------|-----------|----------|
| TWAP | Time execution | No vol prediction | Ignores volume shape |
| VWAP | Volume participation | Vol profile stable | Tracks market not IS |
| IS   | Decision price | Directional view | Requires impact calibration |
| POV  | Market participation | Opportunistic | Unpredictable finish time |

---

## 5. Percentage of Volume (POV)

POV sends orders as a fixed percentage of observed market volume. No schedule — the algo adapts to actual market activity in real time:

```python
class POVAlgo:
    def __init__(self, symbol: str, total_qty: int, side: str,
                  target_pct: float, gateway):
        self.symbol      = symbol
        self.total_qty   = total_qty
        self.side        = side
        self.target_pct  = target_pct   # e.g. 0.10 = 10% of volume
        self.gateway     = gateway
        self.filled_qty  = 0
        self.market_vol  = 0            # observed market volume this interval

    def on_trade(self, trade_size: int):
        """Called for every market trade on this symbol."""
        self.market_vol += trade_size

    def on_interval(self, bid: float, ask: float):
        """Called at end of each measurement interval (e.g. every second)."""
        if self.filled_qty >= self.total_qty:
            return

        target_qty = int(self.market_vol * self.target_pct)
        target_qty = min(target_qty, self.total_qty - self.filled_qty)

        if target_qty > 0:
            price = ask if self.side == 'B' else bid
            self.gateway.limit_order(self.symbol, self.side, target_qty, price)

        self.market_vol = 0   # reset for next interval
```

---

## 6. Smart Order Routing (SOR)

Smart Order Routing (SOR) splits a single order across multiple venues to minimize total execution cost by accessing the best available liquidity.

### 6.1 Venue Selection

```python
@dataclass
class VenueSnapshot:
    venue_id:     str
    bid:          float
    ask:          float
    bid_size:     int
    ask_size:     int
    fee_bps:      float      # taker fee in basis points
    rebate_bps:   float      # maker rebate in basis points
    latency_us:   float      # estimated round-trip latency

def sor_allocate(venues: list[VenueSnapshot], qty: int,
                  side: str) -> list[tuple[str, int, float]]:
    """
    Returns list of (venue_id, qty, price) allocations.
    Minimizes effective price including fees.
    """
    # Sort venues by effective price (price ± fee)
    def effective_price(v: VenueSnapshot) -> float:
        if side == 'B':
            return v.ask * (1 + v.fee_bps * 1e-4)
        else:
            return v.bid * (1 - v.fee_bps * 1e-4)

    sorted_venues = sorted(venues, key=effective_price,
                            reverse=(side == 'S'))
    allocations = []
    remaining   = qty

    for venue in sorted_venues:
        if remaining <= 0:
            break
        available = venue.ask_size if side == 'B' else venue.bid_size
        alloc     = min(remaining, available)
        price     = venue.ask if side == 'B' else venue.bid
        allocations.append((venue.venue_id, alloc, price))
        remaining -= alloc

    return allocations
```

### 6.2 Dark Pool Integration

Dark pools (off-exchange ATS venues) offer price improvement but lower fill probability. Ping dark pools first with IOC (immediate-or-cancel) orders; route remainder to lit venues:

```python
def dark_first_sor(dark_venues: list[VenueSnapshot],
                    lit_venues: list[VenueSnapshot],
                    qty: int, side: str, gateway) -> None:
    # 1. Sweep dark pools with IOC
    for dark in dark_venues:
        available = dark.ask_size if side == 'B' else dark.bid_size
        if available > 0:
            price = dark.ask if side == 'B' else dark.bid
            gateway.ioc_order(dark.venue_id, side, min(qty, available), price)

    # 2. Route remainder to lit venues (fills from IOC callbacks update qty)
    # Remainder handled in on_fill callback
```

---

Next Chapter: [FIX Protocol Deep Dive](10_fix_protocol.md)
