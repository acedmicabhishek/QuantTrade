# QuantTrade — Complete Knowledge Base

Fast-paced docs for quant trading, crypto, Web3, HFT, low-latency systems, and building your own quant simulator.

## Files

| File | What it covers |
|---|---|
| [01_crypto_fundamentals.md](01_crypto_fundamentals.md) | Blockchain, Bitcoin, Ethereum, tokens, wallets, CEX/DEX |
| [02_web3.md](02_web3.md) | Web3 stack, smart contracts, DeFi, AMM, MEV, Layer 2 |
| [03_order_types.md](03_order_types.md) | Every order type, order book anatomy, bid/ask/spread |
| [04_market_microstructure.md](04_market_microstructure.md) | Price discovery, market makers, slippage, adverse selection |
| [05_quant_trading.md](05_quant_trading.md) | Alpha, signals, backtesting, Sharpe, position sizing |
| [06_day_trading.md](06_day_trading.md) | Technical analysis, momentum, scalping, risk management |
| [07_hft_low_latency.md](07_hft_low_latency.md) | Latency budget, FPGA, kernel bypass, lock-free, co-location |
| [08_crypto_hft.md](08_crypto_hft.md) | Crypto-specific HFT: feeds, arb, mempool, gas, DEX bots |
| [09_build_quant_sim.md](09_build_quant_sim.md) | How to extend this repo for crypto quant simulation |

## Reading Order

**Complete beginner:** 01 → 02 → 03 → 04 → 06 → 05 → 07 → 08 → 09

**Have trading background:** 03 → 04 → 05 → 07 → 08 → 09

**Want to build immediately:** 03 → 04 → 09 → everything else as reference

## The Main Repo

This guide lives inside the [HFT QuantSim](../README.md) project — a C++ + Python HFT simulator with:
- Lock-free order book (`include/hft_simulator/orderbook.hpp`)
- Matching engine (`src/matching.cpp`)
- Strategy framework (`python/quantsim/strategy.py`)
- Backtester (`python/quantsim/backtester.py`)
