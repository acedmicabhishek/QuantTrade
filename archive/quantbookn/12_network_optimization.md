# Network Optimization and Co-location

In HFT, network latency is a primary competitive differentiator. This chapter covers physical co-location, kernel bypass networking, hardware timestamping, and OS-level tuning to push round-trip latency below 10 microseconds.

---

## 1. Co-location

Co-location (colo) places your trading servers physically inside the exchange data center. Eliminating the WAN hop typically reduces latency from 5-50ms to 1-10µs.

### 1.1 Proximity Tiers

| Tier | Distance from Matching Engine | Typical RTT | Notes |
|------|-------------------------------|-------------|-------|
| Co-lo (same cage) | < 5m | 1-3 µs | Cross-connect fee applies |
| Co-lo (nearby cage) | < 50m | 3-10 µs | Standard co-lo |
| Proximity hosting | Same building | 10-50 µs | Cheaper, less competitive |
| Remote DC | 1-100 km | 100 µs - 1 ms | WAN hop |

### 1.2 Cross-Connect

A cross-connect is a physical fiber patch cable between your cage and the exchange's network switch. Order from the data center's MMR (Meet-Me Room). Specify:
- **Fiber type:** Single-mode (SMF) for distances > 100m; multi-mode (MMF) within cage
- **Port type:** 10GbE, 25GbE, or 40GbE depending on exchange offering
- **Redundancy:** Two cross-connects to two exchange switches (A/B feeds)

### 1.3 NIC Selection

Standard NICs use the kernel network stack, adding 10-50µs of processing overhead. For HFT, use:

| NIC | Technology | RTT Reduction | Notes |
|-----|-----------|---------------|-------|
| Solarflare SFN8522 | OpenOnload kernel bypass | 2-5 µs | Industry standard |
| Xilinx Alveo U25 | FPGA + kernel bypass | < 1 µs | Programmable datapath |
| Exablaze ExaNIC V5 | Hardware timestamping | < 200 ns | Used for line arbitration |
| Mellanox ConnectX-6 | RDMA / DPDK | 1-3 µs | Also used for storage |

---

## 2. Kernel Bypass: DPDK

DPDK (Data Plane Development Kit) bypasses the Linux kernel TCP/IP stack entirely. Your application polls the NIC directly using busy-wait, eliminating interrupt overhead.

### 2.1 Architecture

```
Standard Linux Network Stack:
  NIC → Interrupt → Kernel TCP/IP → Socket → User App
  Latency: 10-50 µs

DPDK:
  NIC → Poll Mode Driver (PMD) → User App (via DPDK API)
  Latency: 1-3 µs
```

### 2.2 DPDK Initialization

```c
#include <rte_eal.h>
#include <rte_ethdev.h>
#include <rte_mbuf.h>

#define RX_RING_SIZE 1024
#define TX_RING_SIZE 1024
#define NUM_MBUFS    8192
#define MBUF_CACHE   250

int main(int argc, char** argv) {
    // Initialize EAL (Environment Abstraction Layer)
    int ret = rte_eal_init(argc, argv);
    if (ret < 0) rte_exit(EXIT_FAILURE, "EAL init failed\n");

    uint16_t port_id = 0;

    // Create memory pool for packet buffers
    struct rte_mempool* mbuf_pool = rte_pktmbuf_pool_create(
        "MBUF_POOL", NUM_MBUFS, MBUF_CACHE, 0,
        RTE_MBUF_DEFAULT_BUF_SIZE, rte_socket_id()
    );

    // Configure port
    struct rte_eth_conf port_conf = {0};
    rte_eth_dev_configure(port_id, 1, 1, &port_conf);

    // Setup RX/TX queues
    rte_eth_rx_queue_setup(port_id, 0, RX_RING_SIZE,
                            rte_eth_dev_socket_id(port_id), NULL, mbuf_pool);
    rte_eth_tx_queue_setup(port_id, 0, TX_RING_SIZE,
                            rte_eth_dev_socket_id(port_id), NULL);

    rte_eth_dev_start(port_id);

    // Main polling loop (no sleep, no interrupt — pure busy-wait)
    struct rte_mbuf* bufs[32];
    while (1) {
        uint16_t n = rte_eth_rx_burst(port_id, 0, bufs, 32);
        for (uint16_t i = 0; i < n; i++) {
            process_packet(bufs[i]);
            rte_pktmbuf_free(bufs[i]);
        }
    }
}
```

**DPDK requires:** Huge pages enabled, NIC bound to DPDK PMD driver (not kernel driver), dedicated isolated cores.

### 2.3 Solarflare OpenOnload

OpenOnload is a kernel bypass solution that intercepts standard BSD socket calls — your existing code compiles unchanged, but the socket is implemented in user-space:

```bash
# Install OpenOnload driver
# Application links against libefvi/libonload
LD_PRELOAD=libonload.so ./your_trading_app
```

OpenOnload advantages over DPDK:
- **Drop-in:** No code changes to existing socket-based apps.
- **EF_VI API:** Optional low-level API for maximum performance.
- **Automatic fallback:** Non-time-critical connections still use kernel.

```cpp
// Using Solarflare EF_VI (lowest-level, < 1 µs)
#include <etherfabric/vi.h>
#include <etherfabric/pd.h>

ef_driver_handle dh;
ef_pd pd;
ef_vi vi;

ef_driver_open(&dh);
ef_pd_alloc_by_name(&pd, dh, "eth0", EF_PD_DEFAULT);
ef_vi_alloc_from_pd(&vi, dh, &pd, dh,
                    -1, -1, -1, NULL, -1, EF_VI_FLAGS_DEFAULT);

// Post RX buffer
ef_vi_receive_post(&vi, dma_addr, rx_id);

// Poll for received events (busy-wait)
ef_event evs[16];
int n = ef_eventq_poll(&vi, evs, 16);
```

---

## 3. Hardware Timestamping

Software `clock_gettime()` has 100-500ns jitter from OS scheduling. Hardware timestamping captures the exact time the NIC DMA'd the packet, before any software processing.

### 3.1 Linux SO_TIMESTAMPING

Enable hardware timestamps on a socket:

```cpp
#include <linux/net_tstamp.h>
#include <sys/socket.h>

int enable_hw_timestamps(int sock_fd) {
    int flags = SOF_TIMESTAMPING_RX_HARDWARE |
                SOF_TIMESTAMPING_RAW_HARDWARE |
                SOF_TIMESTAMPING_OPT_CMSG;
    return setsockopt(sock_fd, SOL_SOCKET, SO_TIMESTAMPING,
                      &flags, sizeof(flags));
}

// Retrieve timestamp from cmsg ancillary data
uint64_t get_hw_timestamp(struct msghdr* msg) {
    for (struct cmsghdr* cmsg = CMSG_FIRSTHDR(msg);
         cmsg != NULL;
         cmsg = CMSG_NXTHDR(msg, cmsg)) {
        if (cmsg->cmsg_level == SOL_SOCKET &&
            cmsg->cmsg_type  == SCM_TIMESTAMPING) {
            struct timespec* ts = (struct timespec*)CMSG_DATA(cmsg);
            // ts[2] = hardware timestamp (raw)
            return ts[2].tv_sec * 1'000'000'000ULL + ts[2].tv_nsec;
        }
    }
    return 0;
}
```

### 3.2 PTP / IEEE 1588 Time Synchronization

Hardware timestamps are only meaningful if the NIC clock is synchronized to exchange time (UTC). Use PTP (Precision Time Protocol) for sub-microsecond synchronization:

```bash
# Install linuxptp
apt install linuxptp

# Sync NIC hardware clock to GPS/PTP grandmaster
ptp4l -i eth0 -m --slave_only --tx_timestamp_timeout 40

# Sync system clock from PTP hardware clock
phc2sys -s eth0 -c CLOCK_REALTIME -m -w
```

Achievable accuracy: < 100ns to exchange clock with hardware PTP support. Without PTP, use GPS-disciplined oscillator (GPSDO) for independent time reference.

---

## 4. Linux OS Tuning

### 4.1 Kernel Boot Parameters

```bash
# /etc/default/grub — GRUB_CMDLINE_LINUX additions:

isolcpus=2,3,4,5        # Isolate cores 2-5 for trading threads
nohz_full=2,3,4,5       # Disable scheduler tick on isolated cores
rcu_nocbs=2,3,4,5       # No RCU callbacks on isolated cores
irqaffinity=0,1         # Route all hardware IRQs to cores 0-1
intel_idle.max_cstate=0 # Disable C-states (latency from deep sleep)
processor.max_cstate=0
idle=poll               # Busy-wait idle (eliminates C-state wake latency)
transparent_hugepage=never  # Disable THP (unpredictable page faults)
skew_tick=1             # Stagger timer interrupts across cores
```

### 4.2 Interrupt Affinity

Route NIC interrupts away from trading cores:

```bash
# Bind NIC RX interrupt to core 0 only
echo 1 > /proc/irq/$(cat /proc/interrupts | grep eth0 | awk '{print $1}' | tr -d ':')/smp_affinity
```

### 4.3 CPU Frequency Scaling

Disable frequency scaling — variable CPU speed adds latency jitter:

```bash
cpupower frequency-set -g performance
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

### 4.4 NUMA Awareness

For multi-socket servers, ensure all memory allocations occur on the same NUMA node as the NIC and the trading cores:

```cpp
#include <numa.h>

// Allocate on NUMA node 0 (where NIC and cores 2-5 are)
void* buf = numa_alloc_onnode(buffer_size, 0);

// Pin thread to node 0
numa_run_on_node(0);
```

Allocating memory on node 1 while the NIC is on node 0 adds ~100ns per access (QPI/UPI interconnect crossing).

---

## 5. Latency Measurement and Profiling

### 5.1 rdtsc-Based Timing

For sub-microsecond measurement inside tight loops, use the CPU's Time Stamp Counter:

```cpp
static inline uint64_t rdtsc() {
    uint32_t lo, hi;
    __asm__ volatile ("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}

// Calibrate: cycles per nanosecond (measure against clock_gettime)
double cycles_per_ns = 3.2;  // typical for 3.2 GHz CPU

// Measure
uint64_t t0 = rdtsc();
// ... code to measure ...
uint64_t t1 = rdtsc();
double elapsed_ns = (t1 - t0) / cycles_per_ns;
```

`rdtscp` (serializing variant) prevents instruction reordering across the measurement boundary.

### 5.2 Latency Distribution

Always measure latency distribution, not just mean. P99/P999 spikes kill strategies:

```cpp
struct LatencyHistogram {
    std::array<uint64_t, 10000> buckets_{};  // 1ns per bucket, 0-10µs range
    uint64_t overflow_ = 0;

    void record(uint64_t latency_ns) {
        if (latency_ns < buckets_.size())
            ++buckets_[latency_ns];
        else
            ++overflow_;
    }

    double percentile(double p) const {
        uint64_t total = std::accumulate(buckets_.begin(), buckets_.end(), 0ULL)
                         + overflow_;
        uint64_t target = static_cast<uint64_t>(total * p);
        uint64_t cum = 0;
        for (size_t i = 0; i < buckets_.size(); ++i) {
            cum += buckets_[i];
            if (cum >= target) return static_cast<double>(i);
        }
        return static_cast<double>(buckets_.size()); // overflow
    }
};
```

---

Next Chapter: [Risk Engine](13_risk_engine.md)
