# C++ Low-Latency Engineering Patterns

In high-frequency quantitative finance, microseconds are the unit of measurement. A strategy with high statistical accuracy will lose money if its execution engine is slow and gets front-run by competitors. This chapter covers the C++ engineering techniques that bring execution latency into the sub-microsecond regime.

---

## 1. Cache Locality and Cache Line Alignment

CPUs fetch data in **64-byte cache lines** from RAM into L1/L2/L3 caches. Accessing L1 cache takes $\approx 1$ ns; accessing main memory (RAM) takes $\approx 60$ ns (a cache miss).

- **Data Locality:** Use contiguous memory layouts. Avoid nested pointers or node-based structures (`std::list`, `std::map`) on critical paths. Prefer `std::vector` or pre-allocated flat arrays.
- **Cache Line Alignment:** Align structure members to cache line boundaries to avoid **false sharing** — where multiple cores write to different variables on the same cache line, forcing cache invalidation overhead:

```cpp
struct alignas(64) MarketUpdate {
    double price;
    uint32_t volume;
    char symbol[8];
    // Padding automatically added to fill 64 bytes
};
```

- **Padding to separate hot/cold data:** Place frequently-read fields together in the first cache line; rarely-touched fields in a second struct.
- **Prefetching:** Use `__builtin_prefetch(ptr, 0, 3)` to hint the CPU to load the next element of a loop body into cache before it's needed.

---

## 2. Lock-Free Ring Buffers (SPSC Queue)

Classical mutex locks (`std::mutex`) in execution paths are prohibited — locking triggers OS thread context switches, which take several microseconds. Use lock-free data structures with atomic memory orderings instead.

Single-Producer Single-Consumer (SPSC) lock-free ring buffer for passing market data ticks to a strategy:

```cpp
#include <atomic>
#include <vector>
#include <optional>

template<typename T, size_t Capacity>
class LockFreeQueue {
public:
    static_assert((Capacity & (Capacity - 1)) == 0, "Capacity must be a power of 2");

    bool push(const T& item) {
        const size_t write_idx = write_index_.load(std::memory_order_relaxed);
        const size_t read_idx  = read_index_.load(std::memory_order_acquire);

        if ((write_idx - read_idx) == Capacity) {
            return false; // full
        }

        buffer_[write_idx & (Capacity - 1)] = item;
        write_index_.store(write_idx + 1, std::memory_order_release);
        return true;
    }

    std::optional<T> pop() {
        const size_t read_idx  = read_index_.load(std::memory_order_relaxed);
        const size_t write_idx = write_index_.load(std::memory_order_acquire);

        if (read_idx == write_idx) {
            return std::nullopt; // empty
        }

        T item = buffer_[read_idx & (Capacity - 1)];
        read_index_.store(read_idx + 1, std::memory_order_release);
        return item;
    }

private:
    alignas(64) std::atomic<size_t> write_index_{0};
    alignas(64) std::atomic<size_t> read_index_{0};
    T buffer_[Capacity];
};
```

**Memory ordering rules:**
- `relaxed`: no synchronization, just atomicity. Used for the local index load where ordering doesn't matter.
- `acquire`: ensures no reads/writes after this point are reordered before it (pairs with `release`).
- `release`: ensures no reads/writes before this point are reordered after it.

For MPMC (multi-producer, multi-consumer) scenarios, consider `folly::MPMCQueue` or a CAS-based queue, accepting higher complexity.

---

## 3. Memory Pools and Placement `new`

Dynamically allocating via `malloc`/`new` during live trading is dangerous — the heap manager searches for free blocks and may trigger system calls. Pre-allocate at startup using a **memory pool**:

```cpp
#include <new>
#include <vector>

class OrderMemoryPool {
public:
    explicit OrderMemoryPool(size_t count) {
        storage_.resize(count * sizeof(Order));
        for (size_t i = 0; i < count; ++i) {
            free_list_.push_back(storage_.data() + i * sizeof(Order));
        }
    }

    template<typename... Args>
    Order* allocate(Args&&... args) {
        if (free_list_.empty()) return nullptr;
        void* raw = free_list_.back();
        free_list_.pop_back();
        return ::new (raw) Order(std::forward<Args>(args)...);
    }

    void deallocate(Order* ptr) {
        ptr->~Order();
        free_list_.push_back(reinterpret_cast<void*>(ptr));
    }

private:
    std::vector<char> storage_;
    std::vector<void*> free_list_;
};
```

For lock-free allocation, use a slab allocator with atomic free-list head (CAS-based pop/push). Avoid `std::deque` for the free list; use a flat ring instead to keep pool operations O(1) with no dynamic allocation.

---

## 4. Thread Affinity and Core Pinning

OS schedulers migrate threads between cores, destroying cache warmth and introducing scheduling jitter. Pin trading threads to dedicated, isolated cores:

```cpp
#include <pthread.h>
#include <thread>

void pin_thread_to_core(std::thread& th, int core_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(core_id, &cpuset);
    pthread_setaffinity_np(th.native_handle(), sizeof(cpu_set_t), &cpuset);
}
```

**Linux kernel parameters** for dedicated trading cores:
```
isolcpus=2,3          # Prevent OS from scheduling anything on cores 2-3
nohz_full=2,3         # Disable timer interrupts on isolated cores
rcu_nocbs=2,3         # Remove RCU callbacks from isolated cores
irqaffinity=0,1       # Route all hardware IRQs away from trading cores
```

Set in `/etc/default/grub` as `GRUB_CMDLINE_LINUX` and rebuild grub.

---

## 5. SIMD Vectorization

For computationally intensive paths (e.g., batch Greeks computation, feature scoring over order book snapshots), use AVX-512 intrinsics to process 8 doubles in a single instruction:

```cpp
#include <immintrin.h>

// Compute dot product of two 8-element double arrays using AVX-512
double dot_product_avx512(const double* a, const double* b, size_t n) {
    __m512d sum = _mm512_setzero_pd();
    for (size_t i = 0; i < n; i += 8) {
        __m512d va = _mm512_loadu_pd(a + i);
        __m512d vb = _mm512_loadu_pd(b + i);
        sum = _mm512_fmadd_pd(va, vb, sum);
    }
    return _mm512_reduce_add_pd(sum);
}
```

Ensure input arrays are 64-byte aligned (`alignas(64)`) for `_mm512_load_pd` (aligned load) vs `_mm512_loadu_pd` (unaligned, 1-2 cycle penalty).

---

## 6. Huge Pages

Default 4KB OS memory pages mean TLB (Translation Lookaside Buffer) misses on large working sets. Allocate order books and ring buffers on 2MB huge pages to reduce TLB pressure:

```cpp
#include <sys/mman.h>

void* alloc_huge_page(size_t size) {
    void* ptr = mmap(nullptr, size,
                     PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                     -1, 0);
    if (ptr == MAP_FAILED) return nullptr;
    return ptr;
}
```

Enable huge pages:
```bash
echo 512 > /proc/sys/vm/nr_hugepages   # reserve 512 x 2MB = 1GB
```

---

## 7. HFT System Architecture: Component Overview

```
              Exchange Market Data Feed
                         │ (Multicast UDP)
                         ▼
            ┌─────────────────────────┐
            │      Feed Handler       │  Parses ITCH binary, line arbitration
            └────────────┬────────────┘
                         │
                         ▼
            ┌─────────────────────────┐
            │   Order Book Builder    │  Maintains L2/L3 state, computes OFI
            └────────────┬────────────┘
                         │
                         ▼
            ┌─────────────────────────┐
            │    Strategy Engine      │  Evaluates signals, emits order intents
            └────────────┬────────────┘
                         │
                         ▼
            ┌─────────────────────────┐
            │   Pre-Trade Risk Unit   │  Verifies position limits, notional bounds
            └────────────┬────────────┘
                         │
                         ▼
            ┌─────────────────────────┐
            │    Execution Gateway    │  Serializes to FIX/OUCH, writes TCP socket
            └─────────────────────────┘
```

Each component passes data downstream via SPSC queues (Ch 4.2). The feed handler and order book builder run on isolated cores (Ch 4.4). All heap allocation for orders happens at startup via memory pools (Ch 4.3).

---

Next Chapter: [Tick Data Pipelines & Storage](5_data_pipelines.md)
