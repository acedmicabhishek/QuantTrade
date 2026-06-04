# Monitoring, Alerting, and Kill Switches

A trading system without robust monitoring is a liability. Production quant systems require real-time observability of latency, fill rates, positions, and P&L — plus automated circuit breakers that halt trading before small problems become large losses. This chapter covers the instrumentation a QD builds and operates.

---

## 1. Key Metrics to Monitor

### 1.1 Execution Metrics

| Metric | Alert Threshold | Meaning |
|--------|----------------|---------|
| Order round-trip latency (p99) | > 500 µs | Processing or network degradation |
| Fill rate | < 80% of expected | Quotes too far from market |
| Cancel-to-fill ratio | > 10:1 | Over-quoting, potential exchange warning |
| Order reject rate | > 1% | Fat finger check firing or session issues |
| Message send rate | > 80% of throttle limit | Risk of exchange-imposed rate limit |

### 1.2 Risk Metrics

| Metric | Alert Threshold | Meaning |
|--------|----------------|---------|
| Gross position (notional) | > 80% of limit | Approaching position limit |
| Net delta exposure | > limit | Directional risk accumulating |
| Drawdown (intraday) | > 50% of daily loss limit | Strategy underperforming |
| P&L velocity | > -$X/min for 5min | Rapid loss — possible runaway |
| Inventory half-life deviation | > 2x expected | Market making inventory not recycling |

### 1.3 System Metrics

| Metric | Alert Threshold | Meaning |
|--------|----------------|---------|
| Feed handler sequence gaps | > 0 in 5min | Market data reliability issue |
| Strategy CPU usage | > 70% on isolated core | Near compute saturation |
| Ring buffer utilization | > 50% | Risk of backpressure / drops |
| Order gateway TCP reconnects | > 0 | Connection instability |
| NIC queue drops | > 0 | DPDK ring overflow |

---

## 2. Instrumentation Architecture

### 2.1 Metrics Pipeline

```
Trading System (C++)
       │ lock-free stats counters (atomic increment)
       ▼
Stats Aggregator (background thread)
       │ publish every 100ms
       ▼
Prometheus Exporter (HTTP endpoint)
       │ scrape every 15s
       ▼
Prometheus TSDB ──► Grafana (dashboards)
                                │ alert rules
                                ▼
                         AlertManager ──► PagerDuty / Slack
```

The key constraint: **instrumentation must not affect trading latency**. Use lock-free atomic counters in the hot path; aggregate on a background thread.

### 2.2 Lock-Free Stats Counter

```cpp
struct alignas(64) TradingMetrics {
    std::atomic<uint64_t> orders_sent{0};
    std::atomic<uint64_t> orders_filled{0};
    std::atomic<uint64_t> orders_cancelled{0};
    std::atomic<uint64_t> orders_rejected{0};
    std::atomic<int64_t>  net_position{0};
    std::atomic<int64_t>  gross_pnl_cents{0};

    // Latency histogram (bucket = microseconds, 0-1000)
    std::atomic<uint64_t> latency_hist[1001]{};

    void record_fill(char side, int qty, int64_t pnl_cents) {
        ++orders_filled;
        net_position.fetch_add(side == 'B' ? qty : -qty,
                                std::memory_order_relaxed);
        gross_pnl_cents.fetch_add(pnl_cents, std::memory_order_relaxed);
    }

    void record_latency_us(uint64_t us) {
        size_t bucket = std::min(us, (uint64_t)1000);
        ++latency_hist[bucket];
    }
};
```

### 2.3 Prometheus Exposition Format

Background thread publishes metrics over HTTP:

```python
from prometheus_client import start_http_server, Gauge, Histogram, Counter
import time

# Define metrics
order_latency = Histogram('order_rtt_us', 'Order round-trip latency in µs',
                            buckets=[10, 50, 100, 200, 500, 1000, 5000])
net_position  = Gauge('net_position_shares', 'Net position in shares', ['symbol'])
fill_counter  = Counter('fills_total', 'Total fills', ['symbol', 'side'])
pnl_gauge     = Gauge('gross_pnl_usd', 'Gross P&L in USD')

start_http_server(8080)  # Prometheus scrapes :8080/metrics

def publish_from_c_metrics(metrics: dict):
    order_latency.observe(metrics['last_latency_us'])
    net_position.labels(symbol=metrics['symbol']).set(metrics['net_pos'])
    pnl_gauge.set(metrics['pnl_cents'] / 100.0)
```

---

## 3. Kill Switches

A kill switch immediately halts all trading activity. Every production system needs multiple layers of kill switches, each faster than the last.

### 3.1 Kill Switch Hierarchy

```
Level 1: Strategy-level kill (per-strategy, software)
         Trigger: Strategy's own risk checks fail
         Action: Stop generating new orders, cancel open orders
         Latency: < 1 ms

Level 2: Risk engine kill (cross-strategy, software)
         Trigger: Portfolio-level limits breached
         Action: Cancel all open orders on all strategies
         Latency: < 5 ms

Level 3: Gateway kill (hardware/software, all orders)
         Trigger: Operator command or circuit breaker
         Action: Drop TCP connections to exchanges
         Latency: < 10 ms

Level 4: Exchange-side kill (FAK/FAR, Mass Cancel)
         Trigger: Mass Cancel on Sight (MCoS) if gateway drops
         Action: Exchange cancels all open orders on connection drop
         Latency: 1-50 ms (exchange processing)
```

### 3.2 Software Kill Switch Implementation

```cpp
class KillSwitch {
public:
    void arm()   { active_.store(true, std::memory_order_release); }
    void disarm() { active_.store(false, std::memory_order_release); }
    bool is_active() const { return active_.load(std::memory_order_acquire); }

    // Called in hot path — must be < 5ns
    bool check_and_block() const {
        return active_.load(std::memory_order_relaxed);
    }

private:
    alignas(64) std::atomic<bool> active_{false};
};

// In strategy engine — hot path
void StrategyEngine::on_signal(const Signal& sig) {
    if (kill_switch_.check_and_block()) return;   // < 5ns
    // ... generate order ...
}
```

### 3.3 Pre-Trade Risk Checks

Every order must pass inline pre-trade checks before reaching the gateway:

```cpp
class PreTradeRiskChecker {
public:
    struct Limits {
        int64_t max_order_qty       = 10000;
        int64_t max_position_notional = 1'000'000;  // $1M
        int64_t max_order_rate_per_sec = 500;
        double  max_price_deviation_bps = 50;       // 50bps from mid
    };

    enum class Result { PASS, REJECT_QTY, REJECT_NOTIONAL,
                         REJECT_RATE, REJECT_PRICE, REJECT_KILL_SWITCH };

    Result check(const Order& order, double mid_price,
                  int64_t current_position) {
        if (kill_switch_.is_active())
            return Result::REJECT_KILL_SWITCH;

        if (order.qty > limits_.max_order_qty)
            return Result::REJECT_QTY;

        double notional = order.qty * order.price;
        if (std::abs(current_position * order.price) + notional
            > limits_.max_position_notional)
            return Result::REJECT_NOTIONAL;

        // Rate limit: token bucket
        if (!rate_limiter_.allow())
            return Result::REJECT_RATE;

        // Price sanity: not more than N bps from mid
        double dev = std::abs(order.price - mid_price) / mid_price * 10000;
        if (dev > limits_.max_price_deviation_bps)
            return Result::REJECT_PRICE;

        return Result::PASS;
    }

private:
    KillSwitch&  kill_switch_;
    Limits       limits_;
    TokenBucket  rate_limiter_;
};
```

### 3.4 Token Bucket Rate Limiter

```cpp
class TokenBucket {
public:
    TokenBucket(double rate_per_sec, double burst)
        : rate_(rate_per_sec), tokens_(burst), max_(burst),
          last_ts_(steady_clock::now()) {}

    bool allow() {
        auto now     = steady_clock::now();
        double secs  = duration<double>(now - last_ts_).count();
        last_ts_     = now;
        tokens_      = std::min(max_, tokens_ + rate_ * secs);

        if (tokens_ >= 1.0) {
            tokens_ -= 1.0;
            return true;
        }
        return false;
    }

private:
    using steady_clock = std::chrono::steady_clock;
    using duration     = std::chrono::duration<double>;
    double rate_, tokens_, max_;
    std::chrono::steady_clock::time_point last_ts_;
};
```

---

## 4. Circuit Breakers

Automated circuit breakers trigger kill switches when thresholds are exceeded without human intervention.

### 4.1 Loss-Based Circuit Breaker

```python
class LossCircuitBreaker:
    def __init__(self, daily_limit: float, interval_limit: float,
                  interval_seconds: int = 300, kill_switch=None):
        self.daily_limit    = daily_limit
        self.interval_limit = interval_limit
        self.interval_secs  = interval_seconds
        self.kill_switch    = kill_switch
        self.daily_pnl      = 0.0
        self.interval_pnl   = 0.0
        self.interval_start = time.time()

    def on_pnl_update(self, pnl_delta: float):
        now = time.time()
        if now - self.interval_start > self.interval_secs:
            self.interval_pnl   = 0.0
            self.interval_start = now

        self.daily_pnl    += pnl_delta
        self.interval_pnl += pnl_delta

        if self.daily_pnl < -self.daily_limit:
            self._trigger("Daily loss limit exceeded: "
                           f"{self.daily_pnl:.2f} < {-self.daily_limit:.2f}")

        if self.interval_pnl < -self.interval_limit:
            self._trigger(f"Interval loss limit exceeded: "
                           f"{self.interval_pnl:.2f} in {self.interval_secs}s")

    def _trigger(self, reason: str):
        self.kill_switch.arm()
        log_critical(f"CIRCUIT BREAKER TRIGGERED: {reason}")
        alert_ops(reason)
```

### 4.2 Stale Price Detection

If the market data feed goes stale (no updates for > N milliseconds), cancel all orders immediately:

```cpp
class StalePriceGuard {
public:
    explicit StalePriceGuard(int64_t max_age_ns, KillSwitch& ks)
        : max_age_ns_(max_age_ns), kill_switch_(ks) {}

    void on_tick(int64_t ts_ns) {
        last_tick_ns_.store(ts_ns, std::memory_order_release);
    }

    void check(int64_t now_ns) {
        int64_t last = last_tick_ns_.load(std::memory_order_acquire);
        if (now_ns - last > max_age_ns_) {
            kill_switch_.arm();
            // Log: "Feed stale for " + (now_ns - last) + "ns"
        }
    }

private:
    int64_t                max_age_ns_;
    std::atomic<int64_t>   last_tick_ns_{0};
    KillSwitch&            kill_switch_;
};
```

---

## 5. Post-Trade Reconciliation

End-of-day reconciliation catches any discrepancy between internal position records and exchange-reported positions.

```python
def reconcile_positions(internal: dict[str, int],
                          exchange: dict[str, int]) -> list[str]:
    """Compare internal vs exchange positions. Return list of breaks."""
    breaks = []
    all_symbols = set(internal) | set(exchange)

    for sym in sorted(all_symbols):
        int_pos  = internal.get(sym, 0)
        exch_pos = exchange.get(sym, 0)
        if int_pos != exch_pos:
            breaks.append(
                f"{sym}: internal={int_pos}, exchange={exch_pos}, "
                f"diff={exch_pos - int_pos}"
            )
    return breaks
```

Any break must be resolved before next-day trading opens. Common causes:
- Execution report received after internal state reset
- Partial fill not fully processed before position snapshot
- Fat finger cancel creating phantom fill

---

Next Chapter: [ML Infrastructure & Feature Pipelines](15_ml_infrastructure.md)
