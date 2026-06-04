# 07 — HFT & Low Latency Systems

High-frequency trading (HFT) = strategies that hold positions for milliseconds to seconds, relying on speed as a competitive advantage.

This is the domain your C++ simulator is built for.

---

## What HFT Actually Is

Common misconceptions:
- "HFT is front-running" → some forms are, most are legal market making
- "HFT hurts retail" → HFT market makers generally tighten spreads (benefit) but may extract from institutional flow
- "It's all bots going crazy" → it's mathematical models making discrete, well-reasoned decisions in microseconds

**HFT categories**:
| Type | Strategy | Hold Time |
|---|---|---|
| Electronic market making | Quote bid/ask, earn spread | Microseconds-seconds |
| Latency arbitrage | Price lags between venues | Microseconds-milliseconds |
| Statistical arbitrage | Mean reversion, correlated assets | Seconds-minutes |
| Low-latency directional | Momentum, OFI signals | Milliseconds-seconds |
| Flash liquidity | IOC sweeps, liquidity taking | Microseconds |

---

## Latency Taxonomy

Latency is measured at every stage:

```
                     Wire latency
                    ┌─────────────┐
Client ────────────────────────────── Exchange
  │    (network propagation)           │
  │                                    │
  │  Application latency               │  Matching engine
  │  (your code: signal → order)       │  latency (exchange internal)
  │                                    │
  ╰──────────────── Round trip ────────╯

Total RTT = wire × 2 + exchange_internal + your_application
```

### Wire Latency

Speed of light in fiber: ~200,000 km/s (0.67c due to refractive index).

```
New York to Chicago: ~1,200 km
Speed of light (fiber): ~5ms one way
Microwave (line of sight): ~4ms one way
Hollow-core fiber (experimental): ~3.3ms one way

NY to London: ~70ms fiber
NY to Tokyo: ~130ms fiber
```

Real HFT firms have spent hundreds of millions on microwave networks (Virtu, Jump Trading) to shave 1ms off the NY-Chicago route.

### Co-location

Most exchanges offer co-location: you put your servers **inside the exchange's data center**, meters from the matching engine.

```
Co-located: 10-100 microsecond RTT
Same city, not co-located: 1-5 ms RTT
Cross-country: 50-150 ms RTT

Co-location fee: $5,000-100,000/month depending on exchange and cabinet size
```

At 50 microseconds RTT vs 5ms RTT: the fast player has 100x latency advantage.

### Kernel Latency

Even with co-location, your OS adds latency:

```
Default Linux TCP stack: 50-200 microseconds per syscall
With kernel bypass: < 1 microsecond

Kernel bypass options:
  DPDK: bypass kernel, process packets in userspace at line rate
  Solarflare OpenOnload: kernel bypass via user-space networking
  Mellanox ConnectX + VMA: RDMA-based kernel bypass
  FPGA: eliminate CPU entirely for critical path
```

---

## System Architecture

### Critical Path

The **critical path** is the sequence of operations that determines how fast you can respond to a market event:

```
Market data arrives
  → Parse raw bytes (market data feed handler)
  → Update order book
  → Run signal computation
  → Risk check
  → Generate order
  → Serialize order
  → Send to exchange
  
Each step adds latency. Optimize ruthlessly only on critical path.
```

Non-critical path (don't optimize aggressively):
- Logging
- P&L calculation
- Risk monitoring dashboards
- Position accounting

### Threading Model

```
Recommended for HFT:

Thread 1 (critical path, pinned to core 0):
  Receive market data → process → send orders
  NO system calls. No allocations. No logging.
  
Thread 2 (risk monitor, pinned to core 1):
  Read positions (lock-free) → check limits
  Alert only — never block thread 1
  
Thread 3 (logging, lower priority):
  Consume from lock-free ring buffer
  Write to disk/network
  
Thread 4 (order management):
  Track fills, update positions
  Never touch critical path during operation
```

### CPU Affinity and NUMA

```bash
# Pin thread to specific core
taskset -c 2 ./hft_system

# In C++:
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(2, &cpuset);
pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset);

# NUMA topology: threads should be on same socket as NIC
# Cross-NUMA memory access adds 50-100ns per access
numactl --cpunodebind=0 --membind=0 ./hft_system
```

---

## Lock-Free Data Structures

Locks (mutexes) are catastrophic for latency. Under lock contention, threads wait → microseconds to milliseconds of delay.

### Single-Producer Single-Consumer Ring Buffer

```cpp
template<typename T, size_t Capacity>
class SPSCQueue {
    static_assert((Capacity & (Capacity - 1)) == 0, "must be power of 2");
    
    alignas(64) std::atomic<size_t> head_{0};  // producer writes
    alignas(64) std::atomic<size_t> tail_{0};  // consumer reads
    T buffer_[Capacity];
    
public:
    bool push(const T& item) {
        size_t h = head_.load(std::memory_order_relaxed);
        size_t next = (h + 1) & (Capacity - 1);
        if (next == tail_.load(std::memory_order_acquire)) return false; // full
        buffer_[h] = item;
        head_.store(next, std::memory_order_release);
        return true;
    }
    
    bool pop(T& item) {
        size_t t = tail_.load(std::memory_order_relaxed);
        if (t == head_.load(std::memory_order_acquire)) return false; // empty
        item = buffer_[t];
        tail_.store((t + 1) & (Capacity - 1), std::memory_order_release);
        return true;
    }
};
```

This queue is used in your repo's order management between threads.

### Compare-and-Swap (CAS)

Atomic primitive for lock-free algorithms:

```cpp
// CAS loop: modify shared state without mutex
std::atomic<int> counter{0};

void increment() {
    int expected = counter.load();
    while (!counter.compare_exchange_weak(expected, expected + 1)) {
        // retry with new expected value
    }
}
```

CAS = single CPU instruction on x86 (LOCK CMPXCHG). Atomic and fast.

### False Sharing

When two threads write to different variables that live in the same cache line (64 bytes), they "thrash" each other's caches.

```cpp
// BAD: head and tail in same cache line
struct QueueBad {
    size_t head;  // thread 1
    size_t tail;  // thread 2  ← same cache line as head
};

// GOOD: aligned to separate cache lines
struct QueueGood {
    alignas(64) size_t head;  // thread 1: owns this cache line
    alignas(64) size_t tail;  // thread 2: separate cache line
};
```

False sharing can add 100-200ns per access. Critical to avoid on hot variables.

---

## Order Book Implementation

### Price Level Storage

Two main approaches:

**Array-indexed** (fastest, what your repo uses):
```cpp
// Fixed price range, index = (price - base) / tick_size
// O(1) update, O(1) lookup
// Memory: large but cache-resident for top-of-book
```

**Map-based** (flexible but slower):
```cpp
// std::map<Price, Level> bids (sorted by price)
// O(log n) update, O(log n) lookup
// Good for dynamic price ranges
```

**Hybrid**: use array for top N levels, overflow to map.

### Key Performance Numbers (Target)

```
Order book update: < 100 nanoseconds
Signal computation: < 500 nanoseconds
Order generation: < 100 nanoseconds
Serialization: < 1 microsecond
Total critical path: < 2 microseconds (software only, co-located)
```

### Memory Layout

Cache line = 64 bytes. Pack data accordingly.

```cpp
// BAD: scattered in memory
struct Order {
    OrderID id;       // 8 bytes
    std::string symbol; // 32+ bytes, heap allocated
    Price price;      // 8 bytes
    Quantity qty;     // 8 bytes
    bool is_buy;      // 1 byte + 7 padding
    // total: 57+ bytes + string heap
};

// GOOD: compact, stack allocated, cache-friendly
struct Order {
    uint64_t id;       // 8
    uint64_t price;    // 8 (fixed point, price × 100)
    uint64_t qty;      // 8 (quantity × 100)
    uint32_t symbol_id;// 4 (lookup table, not string)
    bool is_buy;       // 1
    uint8_t padding[3];// 3
    // total: 32 bytes = half cache line
};
```

---

## Market Data Feed Handling

### Feed Types (Exchanges)

| Protocol | Type | Latency | Used by |
|---|---|---|---|
| ITCH (NASDAQ) | Binary, UDP multicast | Lowest | Equities |
| OUCH (NASDAQ) | Binary, TCP | Low | Equity order entry |
| FIX/FAST | Text-ish, TCP | Medium | Most exchanges |
| WebSocket + JSON | Text, TCP | High | Crypto exchanges |
| WebSocket + MsgPack | Binary, TCP | Medium | Some crypto |
| Custom binary | Varies | Very low | Co-lo feeds |

**Your repo includes an ITCH parser** (`src/itch_parser.cpp`). ITCH is the gold standard for market data performance.

For crypto, most exchanges use WebSocket. This is significantly slower than ITCH but unavoidable unless you co-locate and use exchange's FIX endpoint.

### Feed Processing

```cpp
// Typical feed handler structure
void on_market_data(const uint8_t* data, size_t len) {
    // 1. Parse message type (single byte lookup)
    MessageType type = static_cast<MessageType>(data[0]);
    
    // 2. Dispatch to handler (function pointer table, O(1))
    handlers_[static_cast<int>(type)](data + 1, len - 1);
}

void on_add_order(const uint8_t* data, size_t len) {
    // 3. Parse fields — no dynamic allocation
    OrderID id = *(uint64_t*)(data);
    Price price = *(uint64_t*)(data + 8);
    Quantity qty = *(uint64_t*)(data + 16);
    Side side = *(bool*)(data + 24) ? Side::BUY : Side::SELL;
    
    // 4. Update book — O(1)
    book_.add_order(id, price, qty, side);
    
    // 5. Trigger signal
    signal_engine_.on_book_update(book_);
}
```

---

## FPGA in HFT

Field-Programmable Gate Arrays — reconfigurable hardware.

```
FPGA vs CPU:
  CPU: general purpose, sequential (mostly), 1-10ns per operation
  FPGA: parallel pipeline, nanosecond-level, no OS overhead
  
FPGA market data handler:
  Receive packet → parse header → dispatch → update order book: 10-50 nanoseconds
  
CPU equivalent: 500-2000 nanoseconds
```

**FPGA use cases in HFT**:
- Market data feed handler (parse UDP packets)
- Order entry (generate TCP packet)
- Risk checks (simple limits: position size, order rate)
- Simple arbitrage logic (if price A > price B + cost → fire order)

**Not worth for**: complex ML models, signal computation, strategy logic with state.

FPGA vendors: Xilinx (AMD), Intel (Altera). Used by: Virtu, Jane Street, Optiver, Citadel.

---

## Latency Budget

When designing an HFT system, allocate your total budget:

```
Total target: 10 microseconds (co-located, software only)

  Network receive (NIC → CPU, with kernel bypass): 1 μs
  Message parsing:                                0.1 μs
  Order book update:                              0.1 μs
  Signal computation:                             0.5 μs
  Risk check:                                     0.2 μs
  Order generation:                               0.1 μs
  Network transmit (CPU → NIC):                   1 μs
  Exchange matching engine:                       5 μs
  ─────────────────────────────────────────────────────
  Total (one-way):                               ~8 μs
  Round-trip confirmation:                      ~16 μs
```

With FPGA: can compress the non-exchange parts to 1-2 μs.

---

## Benchmarking Your System

```cpp
// High-resolution timing on Linux
#include <time.h>

uint64_t now_ns() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (uint64_t)ts.tv_sec * 1000000000ULL + ts.tv_nsec;
}

// RDTSC (CPU timestamp counter) — even faster, but not portable
inline uint64_t rdtsc() {
    uint32_t lo, hi;
    __asm__ __volatile__ ("rdtsc" : "=a"(lo), "=d"(hi));
    return (uint64_t)hi << 32 | lo;
}

// Your benchmark target: see src/benchmark.cpp
// run: ./run_benchmark → latency histogram in nanoseconds
```

---

## Jitter

Average latency is less important than **tail latency**. A strategy that requires 10μs to respond needs P99 < 10μs, not just average < 10μs.

**Sources of jitter**:
- OS scheduling: other processes interrupt your thread
- Memory allocation: `new`/`malloc` → heap traversal → variable time
- Garbage collection: Java/Python completely unusable for latency-critical paths
- Page faults: memory not in TLB/cache → OS page walk
- IRQ handling: network interrupts
- Thermal throttling: CPU downclock when hot

**Mitigations**:
```
mlockall(MCL_CURRENT | MCL_FUTURE)  // lock all memory in RAM, no page faults
CPU isolation (isolcpus kernel param)  // dedicate cores to your process
Real-time kernel (PREEMPT_RT)  // deterministic scheduling
Huge pages (HugeTLB)  // reduce TLB misses
DPDK/OpenOnload  // kernel bypass
Pre-allocate all memory  // no dynamic allocation on hot path
```

---

## Order Management System (OMS)

The OMS tracks all open orders, fills, and positions. It must be:
- Fast (cancel/modify orders quickly)
- Correct (every fill must update position)
- Safe (pre-trade risk checks before every order)

```cpp
// Minimal OMS state
struct OMS {
    std::unordered_map<OrderID, Order> live_orders;  // indexed for O(1) cancel
    Position position;                               // current net position
    double pnl;                                      // marked P&L
    
    void on_fill(const Fill& fill) {
        live_orders.erase(fill.order_id);
        position.update(fill);
        pnl += fill.realized_pnl();
    }
};
```

---

## Risk Management in HFT

Pre-trade risk checks (must be < 1μs):
```
1. Position limit: |new_position| < max_position
2. Order rate limit: orders_per_second < max
3. Notional limit: order_value < max_notional
4. Price sanity: |price - mid| < n × spread (prevent fat-finger)
5. Total P&L circuit breaker: daily_loss < max_daily_loss → halt
```

If any check fails: cancel all open orders, stop sending.

---

## Your Repo's Architecture

```
src/matching.cpp         → Core matching engine (price-time priority)
include/hft_simulator/orderbook.hpp → Lock-free order book
src/engine.cpp           → Trading engine (signal → order lifecycle)
src/benchmark.cpp        → Latency benchmarking tool
src/itch_parser.cpp      → ITCH 5.0 binary feed parser
python/quantsim/         → Python layer for strategy research/backtesting
```

For crypto HFT, you'd add:
- WebSocket feed handler (Boost.Beast or uWebSockets)
- Exchange-specific message encoding (Binance/OKX JSON → your internal format)
- Multi-exchange arbitrage logic

---

## Next

Read [08_crypto_hft.md](08_crypto_hft.md) for crypto-specific HFT mechanics.
