# Tick Data Pipelines and Storage

A quantitative developer's primary raw material is market data. This chapter covers the full stack: sourcing raw tick data, normalizing it into a consistent schema, storing it efficiently for backtesting and research, and building real-time ingestion pipelines for live trading.

---

## 1. Data Sources and Feed Types

### 1.1 Data Hierarchy

| Level | Name | Contents | Latency |
|-------|------|----------|---------|
| L1 | Top of Book | Best bid/ask + size | < 1ms |
| L2 | Market Depth | Full price levels, sizes | < 1ms |
| L3 | Order-by-Order | Individual order add/cancel/execute | < 1ms |
| Trades | Time & Sales | Executed trades with aggressor side | < 1ms |
| Reference | Instrument Ref | Symbol, lot size, tick size, expiry | Daily |

L3 data (ITCH feed) is required for accurate order book reconstruction and microstructure features (OFI, book imbalance). L2 is sufficient for strategy backtesting.

### 1.2 Feed Sources

**Consolidated Tape (SIP):** OPRA/CTA/UTP feeds aggregate data across all US exchange venues. Cheapest, but introduces ~1-5ms latency from consolidation. Suitable for daily/intraday research.

**Direct Exchange Feeds:** Nasdaq TotalView-ITCH, NYSE OpenBook Ultra, CME MDP 3.0. Lowest latency, one venue only. Required for microstructure research and HFT.

**Historical Vendors:** Refinitiv Tick History, Bloomberg BTCA, Databento, Polygon.io, Tardis.dev. Normalized schemas with good API coverage. Tardis provides raw binary ITCH/MDP3 replays for realistic feed handler testing.

---

## 2. Data Normalization

Raw feeds arrive in exchange-specific formats. Normalize to a canonical internal schema before storage:

### 2.1 Canonical Schemas

```python
from dataclasses import dataclass
from enum import IntEnum

class Side(IntEnum):
    BID = 0
    ASK = 1

@dataclass(slots=True)
class Trade:
    ts_exchange_ns: int    # exchange-stamped nanoseconds since epoch
    ts_recv_ns:     int    # local receive timestamp
    symbol:         str
    price:          float
    size:           int
    aggressor:      Side   # which side was the taker

@dataclass(slots=True)
class BookLevel:
    price:  float
    size:   int
    count:  int            # number of orders at this level (if available)

@dataclass(slots=True)
class BookSnapshot:
    ts_ns:   int
    symbol:  str
    bids:    list[BookLevel]   # sorted descending by price
    asks:    list[BookLevel]   # sorted ascending by price
    seq:     int               # sequence number for gap detection
```

### 2.2 Corporate Action Adjustment

Price continuity breaks at dividends, splits, and spin-offs. For backtesting, adjust historical prices using **back-adjusted** (multiply all historical prices by cumulative adjustment factor):

```python
def apply_split_adjustment(prices: np.ndarray, dates: np.ndarray,
                             splits: list[tuple]) -> np.ndarray:
    """
    splits: list of (date, ratio) e.g. (2022-08-25, 20.0) for AAPL 20:1 split
    Adjusts all prices before split date backward.
    """
    adjusted = prices.copy().astype(float)
    for split_date, ratio in sorted(splits, reverse=True):
        mask = dates < split_date
        adjusted[mask] /= ratio
    return adjusted
```

---

## 3. Storage Formats

### 3.1 Parquet (Research / Batch Processing)

Apache Parquet is the standard for quant research workloads: columnar storage, excellent compression, native support in pandas/polars/DuckDB/Spark.

```python
import polars as pl

# Write tick data to partitioned Parquet
def write_tick_parquet(trades: list[Trade], path: str, symbol: str, date: str):
    df = pl.DataFrame({
        "ts_exchange_ns": [t.ts_exchange_ns for t in trades],
        "ts_recv_ns":     [t.ts_recv_ns     for t in trades],
        "price":          [t.price          for t in trades],
        "size":           [t.size           for t in trades],
        "aggressor":      [int(t.aggressor) for t in trades],
    })
    df.write_parquet(
        f"{path}/{symbol}/{date}/trades.parquet",
        compression="zstd",
        statistics=True,
    )

# Query efficiently — predicate pushdown reads only needed row groups
def read_price_range(path: str, symbol: str, date: str,
                      start_ns: int, end_ns: int) -> pl.DataFrame:
    return pl.scan_parquet(f"{path}/{symbol}/{date}/trades.parquet") \
             .filter(pl.col("ts_exchange_ns").is_between(start_ns, end_ns)) \
             .collect()
```

**Partitioning strategy:** `symbol / date / trades.parquet`. Enables fast reads by date range without scanning the whole dataset.

### 3.2 kdb+ / q (Production Time-Series)

kdb+ is the industry standard for high-frequency tick data storage. Its columnar, in-memory architecture with on-disk HDB (historical database) allows nanosecond-precision queries over years of data.

```q
// Define trade schema in q
trade:([] time:`timestamp$(); sym:`symbol$(); price:`float$(); size:`int$(); side:`char$())

// Upsert from C++ feed handler via kdb+ IPC (port 5000)
// In production, use kdb+ tickerplant (TP) → real-time subscriber → HDB saver

// Query last 1 minute of AAPL trades
select from trade where sym=`AAPL, time > .z.p - 0D00:01:00

// Compute 1-second OHLCV bars
select open:first price, high:max price, low:min price,
       close:last price, volume:sum size
  by sym, 1000000000 xbar time  // 1 second = 1e9 ns
  from trade where date = .z.d
```

### 3.3 TimescaleDB (Open Source Alternative)

For teams without kdb+ licenses, TimescaleDB (PostgreSQL extension) provides time-series optimizations:

```sql
-- Create hypertable partitioned by time
CREATE TABLE trades (
    ts_ns       BIGINT NOT NULL,
    symbol      VARCHAR(16) NOT NULL,
    price       DOUBLE PRECISION NOT NULL,
    size        INTEGER NOT NULL,
    aggressor   SMALLINT NOT NULL
);

SELECT create_hypertable('trades', 'ts_ns',
                          chunk_time_interval => 86400000000000);  -- 1 day in ns

-- Create compression policy (run nightly)
SELECT add_compression_policy('trades', INTERVAL '7 days');

-- Fast range query with index on (symbol, ts_ns)
CREATE INDEX ON trades (symbol, ts_ns DESC);
```

### 3.4 Format Comparison

| Format | Write Speed | Read Speed | Compression | Query Language | Best For |
|--------|------------|-----------|-------------|----------------|----------|
| Parquet | Medium | High | Excellent | SQL/Python | Batch research |
| kdb+ | Highest | Highest | Good | q | Production, HFT |
| TimescaleDB | Medium | Medium | Good | SQL | Open-source prod |
| HDF5 | Medium | High | Good | Python h5py | Numpy/ML arrays |

---

## 4. Real-Time Ingestion Pipeline

### 4.1 Pipeline Architecture

```
Exchange Feed (UDP multicast)
        │
        ▼
 Feed Handler (C++)          ← Ch 4: parses ITCH binary
        │ SPSC queue
        ▼
 Normalizer (C++)            ← converts to canonical schema
        │ shared memory / IPC
        ▼
 ┌──────────────────────────────────┐
 │                                  │
 ▼                                  ▼
Live Strategy Engine          Tick Data Writer
(latency-critical)            (kdb+ tickerplant)
```

The normalizer fans out to two consumers: the strategy engine (latency-critical path, zero-copy) and the tick data writer (durability path, can tolerate 1-10ms lag).

### 4.2 kdb+ Tickerplant Integration

The kdb+ tickerplant (TP) is the standard real-time logging component. Feed it via IPC from the C++ normalizer:

```cpp
// kdb+ IPC connection from C++ (using KDB+ C API)
#include "k.h"

class KDBWriter {
public:
    KDBWriter(const char* host, int port) {
        conn_ = khpu(host, port, "user:pass");
    }

    void write_trade(const Trade& t) {
        K row = knk(5,
            ktj(-KJ, t.ts_exchange_ns),
            ks(const_cast<char*>(t.symbol.c_str())),
            kf(t.price),
            ki(t.size),
            kc(t.aggressor == Side::BID ? 'B' : 'S')
        );
        k(conn_, ".u.upd", ks("trade"), row, (K)0);
        r0(row);
    }

private:
    int conn_;
};
```

### 4.3 Backpressure and Gap Handling

If the writer falls behind (disk I/O, network congestion):
1. Use a bounded ring buffer between normalizer and writer.
2. On buffer full: drop data to tick writer (never drop to strategy).
3. Log sequence gaps; fill from recovery feed if available.
4. Alert ops when gap rate exceeds threshold.

---

## 5. Data Quality and Validation

### 5.1 Automated Checks

Run these on every daily data load:

```python
def validate_tick_data(df: pl.DataFrame, symbol: str, date: str) -> list[str]:
    issues = []

    # Timestamp monotonicity
    if not df["ts_exchange_ns"].is_sorted():
        issues.append("Non-monotonic timestamps")

    # Price sanity: no zero or negative prices
    if (df["price"] <= 0).any():
        issues.append("Non-positive prices detected")

    # Price continuity: flag >10% moves between consecutive ticks
    price_changes = df["price"].pct_change().abs()
    if (price_changes > 0.10).any():
        issues.append(f"Large price jump: max={price_changes.max():.2%}")

    # Volume sanity
    if (df["size"] <= 0).any():
        issues.append("Non-positive trade sizes")

    # Coverage: at least 50% of expected trading hours
    expected_ns = 6.5 * 3600 * 1e9
    actual_range = df["ts_exchange_ns"].max() - df["ts_exchange_ns"].min()
    if actual_range < 0.5 * expected_ns:
        issues.append(f"Sparse data: only {actual_range/1e9/3600:.1f}h covered")

    return issues
```

### 5.2 Outlier Filtering

Apply NBBO (National Best Bid and Offer) bounds filtering before research:

```python
def filter_outliers(trades: pl.DataFrame, rolling_window: int = 1000,
                     z_threshold: float = 10.0) -> pl.DataFrame:
    """Remove trades more than z_threshold std devs from rolling mean."""
    rolling_mean = trades["price"].rolling_mean(rolling_window)
    rolling_std  = trades["price"].rolling_std(rolling_window)
    z_scores = (trades["price"] - rolling_mean) / rolling_std
    return trades.filter(z_scores.abs() < z_threshold)
```

---

Next Chapter: [Backtesting Engine Design](6_backtesting_engine.md)
