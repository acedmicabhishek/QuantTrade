# Quantitative Developer Handbook

A field guide for engineers transitioning into quantitative developer roles at trading firms, hedge funds, and market makers. Emphasis on production implementation over theory: C++, systems design, exchange protocols, and the math you need to build — not just understand.

---

## Who This Is For

**Quantitative Developers (QD):** Bridge between research and production. You build the backtesting simulators, execution engines, market data feed handlers, and low-latency infrastructure. You are a C++/Python engineer who needs to understand enough math to implement it correctly and enough market structure to know when your system is misbehaving.

This is not a QR (Quantitative Researcher) book. It minimizes derivations and maximizes working code.

---

## Structure

### Part I — Foundations

| # | Chapter | Core Topics |
|---|---------|-------------|
| 1 | [The QD Mindset & Role](1_qd_mindset.md) | Role taxonomy, market participants, engineering vs. finance mental models |
| 2 | [Math for Implementors](2_math_for_implementors.md) | GBM, Itô's Lemma, Black-Scholes, Greeks, Monte Carlo — code-first |
| 3 | [Microstructure & Matching Engine Internals](3_microstructure_matching.md) | LOB mechanics, price-time priority, market impact, adverse selection, VPIN |

### Part II — Systems Core

| # | Chapter | Core Topics |
|---|---------|-------------|
| 4 | [C++ Low-Latency Engineering](4_cpp_low_latency.md) | Cache locality, SPSC lock-free queues, memory pools, core pinning, SIMD, huge pages |
| 5 | [Tick Data Pipelines & Storage](5_data_pipelines.md) | Feed types, normalization schemas, Parquet/kdb+/TimescaleDB, real-time ingestion, data quality |
| 6 | [Backtesting Engine Design](6_backtesting_engine.md) | Event-driven vs. vectorized, fill simulation, look-ahead bias prevention, purged cross-validation |

### Part III — Strategy Implementation

| # | Chapter | Core Topics |
|---|---------|-------------|
| 7 | [Statistical Arbitrage & Mean Reversion](7_stat_arb.md) | OU process, Engle-Granger/Johansen cointegration, z-score signals, Kalman hedge ratio |
| 8 | [Market Making: Inventory Control](8_market_making.md) | Avellaneda-Stoikov model, reservation price, optimal spreads, VPIN adverse selection, stale quote protection |
| 9 | [Execution Algorithms](9_execution_algos.md) | TWAP, VWAP volume profiling, Almgren-Chriss IS, POV, Smart Order Routing, dark pools |

### Part IV — Exchange Connectivity

| # | Chapter | Core Topics |
|---|---------|-------------|
| 10 | [FIX Protocol: Session & Application Layer](10_fix_protocol.md) | Session state machine, message types (NOS/ER/OCR), sequence recovery, QuickFIX/N, order state machine |
| 11 | [Binary Protocols: ITCH, OUCH, SBE](11_binary_protocols.md) | ITCH struct binding, feed handler, line arbitration, OUCH order entry, CME SBE, hardware timestamps |
| 12 | [Network Optimization & Co-location](12_network_optimization.md) | Colo tiers, cross-connect, DPDK, Solarflare OpenOnload/EF_VI, PTP timestamping, OS kernel tuning |

### Part V — Production Operations

| # | Chapter | Core Topics |
|---|---------|-------------|
| 13 | [Risk Engine: Pre-Trade & Real-Time](13_risk_engine.md) | VaR, CVaR, Kelly criterion, position limits, pre-trade checks, operational safeguards |
| 14 | [Monitoring, Alerting & Kill Switches](14_monitoring.md) | Metrics pipeline, Prometheus/Grafana, pre-trade risk, kill switch hierarchy, circuit breakers, reconciliation |
| 15 | [ML Infrastructure & Feature Pipelines](15_ml_infrastructure.md) | Feature engineering (OFI, imbalance, frac diff), triple barrier labeling, model serving, online learning |

---

## Suggested Reading Paths

### "I want to crack a QD interview"
→ Ch 1, 2, 3, 4, 6, 10, 11

### "I want to build a backtester from scratch"
→ Ch 5, 6, 7, 8

### "I want to understand HFT system architecture"
→ Ch 4, 5, 10, 11, 12, 14

### "I want to implement my first market making strategy"
→ Ch 3, 6, 7, 8, 13, 14

### "I want to build exchange connectivity"
→ Ch 10, 11, 12

---

## Prerequisites

- **Strong:** C++17, Python 3, linear algebra, probability theory
- **Helpful:** OS internals (memory model, threading), network fundamentals (TCP/UDP)
- **Not Required:** Stochastic calculus (Ch 2 covers what you need), finance background

---

*15 chapters. All code is production-oriented. Math is present where it directly informs implementation.*
