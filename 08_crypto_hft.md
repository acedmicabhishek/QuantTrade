# 08 — Crypto HFT

Crypto HFT differs from equities HFT in several important ways:
- No single dominant venue → fragmentation is an opportunity
- WebSocket APIs instead of binary UDP protocols → higher latency floor
- On-chain settlement creates unique latency dimension (DEX)
- Perpetual futures with funding rate → carry strategies
- Manipulation (spoofing, wash trading) is widespread

---

## CEX vs DEX HFT

| Dimension | CEX (Binance/OKX) | DEX (Uniswap/dYdX) |
|---|---|---|
| Latency | 1-50ms round-trip | Block time (2-12s on ETH, 400ms on Solana) |
| API | WebSocket/REST | JSON-RPC or custom |
| Order types | Full (limit/market/IOC) | On-chain tx or off-chain orderbook (dYdX) |
| Front-running risk | Exchange policy | MEV (guaranteed) |
| Co-location | Available | Irrelevant (on-chain) |
| Fee structure | Taker/maker, tiered | Gas + protocol fee (fixed % of swap) |
| Counterparty risk | Exchange (FTX risk!) | Smart contract (code risk) |
| Leverage | Up to 125x | Protocol-specific |

For HFT purposes: **CEX is the primary venue**. DEX HFT = MEV strategies (see [02_web3.md](02_web3.md)).

---

## CEX Market Data

### WebSocket Order Book

Most crypto exchanges publish:
- **Full snapshot**: complete order book on subscription
- **Incremental updates**: only changes since last message

```python
# Reconstruct order book from WebSocket stream
import json
import websockets
import asyncio
from sortedcontainers import SortedDict

class OrderBook:
    def __init__(self):
        self.bids = SortedDict(lambda x: -x)  # descending price
        self.asks = SortedDict()               # ascending price
    
    def apply_update(self, bids_update, asks_update):
        for price, qty in bids_update:
            if qty == 0:
                self.bids.pop(float(price), None)
            else:
                self.bids[float(price)] = float(qty)
        for price, qty in asks_update:
            if qty == 0:
                self.asks.pop(float(price), None)
            else:
                self.asks[float(price)] = float(qty)
    
    @property
    def best_bid(self):
        return next(iter(self.bids)) if self.bids else None
    
    @property
    def best_ask(self):
        return next(iter(self.asks)) if self.asks else None
    
    @property 
    def mid(self):
        if self.best_bid and self.best_ask:
            return (self.best_bid + self.best_ask) / 2
        return None
    
    @property
    def spread(self):
        if self.best_bid and self.best_ask:
            return self.best_ask - self.best_bid
        return None

async def subscribe_binance_orderbook(symbol="BTCUSDT", depth=20):
    url = f"wss://stream.binance.com:9443/ws/{symbol.lower()}@depth{depth}@100ms"
    book = OrderBook()
    
    async with websockets.connect(url) as ws:
        async for msg in ws:
            data = json.loads(msg)
            book.apply_update(data['b'], data['a'])
            
            # Compute signal
            imbalance = compute_imbalance(book, levels=5)
            yield book, imbalance

def compute_imbalance(book, levels=5):
    bid_sizes = [v for v in list(book.bids.values())[:levels]]
    ask_sizes = [v for v in list(book.asks.values())[:levels]]
    bid_vol = sum(bid_sizes)
    ask_vol = sum(ask_sizes)
    if bid_vol + ask_vol == 0:
        return 0
    return (bid_vol - ask_vol) / (bid_vol + ask_vol)
```

### WebSocket Trade Feed

Real-time trade prints (executed orders). Use for OFI calculation.

```python
async def subscribe_binance_trades(symbol="BTCUSDT"):
    url = f"wss://stream.binance.com:9443/ws/{symbol.lower()}@aggTrade"
    
    async with websockets.connect(url) as ws:
        async for msg in ws:
            data = json.loads(msg)
            yield {
                "price": float(data["p"]),
                "qty": float(data["q"]),
                "is_buyer_maker": data["m"],  # True = sell hit bid, False = buy hit ask
                "time": data["T"]
            }
```

### REST vs WebSocket

| Use case | Use |
|---|---|
| Order management | REST (place, cancel, query) |
| Real-time book | WebSocket |
| Real-time trades | WebSocket |
| Account balance | REST (on demand) |
| Historical data | REST |

**WebSocket latency** on Binance from EU/US: typically 10-80ms. From Singapore (co-location): 1-5ms.

---

## Exchange APIs (Binance Reference)

### Authentication

```python
import hmac, hashlib, time
import requests

API_KEY = "your_key"
SECRET = "your_secret"

def sign(params: dict) -> str:
    query = "&".join(f"{k}={v}" for k, v in sorted(params.items()))
    return hmac.new(SECRET.encode(), query.encode(), hashlib.sha256).hexdigest()

def place_limit_order(symbol, side, price, qty):
    endpoint = "https://api.binance.com/api/v3/order"
    params = {
        "symbol": symbol,
        "side": side,         # BUY or SELL
        "type": "LIMIT",
        "timeInForce": "GTC",
        "price": str(price),
        "quantity": str(qty),
        "timestamp": int(time.time() * 1000)
    }
    params["signature"] = sign(params)
    headers = {"X-MBX-APIKEY": API_KEY}
    return requests.post(endpoint, params=params, headers=headers).json()

def cancel_order(symbol, order_id):
    endpoint = "https://api.binance.com/api/v3/order"
    params = {"symbol": symbol, "orderId": order_id, "timestamp": int(time.time() * 1000)}
    params["signature"] = sign(params)
    headers = {"X-MBX-APIKEY": API_KEY}
    return requests.delete(endpoint, params=params, headers=headers).json()
```

### Rate Limits (Critical for HFT)

Binance limits:
```
Order rate limit: 10 orders/second, 100,000 orders/day
Request weight: each endpoint costs 1-50 "weight" per request
  Weight limit: 1200 per minute
  
Exceeding limits → HTTP 429 (rate limit) or 418 (IP ban)
```

For HFT: use WebSocket order management (not REST) for order placement. Binance supports order placement via WebSocket User Data Stream.

---

## Crypto HFT Strategies

### 1. Cross-Exchange Spot Arbitrage

Price of BTC differs across exchanges. Buy on cheaper, sell on more expensive.

**The catch**: execution requires pre-funded accounts on both sides (can't transfer between exchanges fast enough).

```python
class CrossExchangeArb:
    def __init__(self, exchange_a_book, exchange_b_book, min_profit_bps=2.0):
        self.min_profit = min_profit_bps / 10000
    
    def check_arb(self, book_a, book_b, fee_a, fee_b):
        # Buy on A, sell on B
        if book_a.best_ask and book_b.best_bid:
            cost = book_a.best_ask * (1 + fee_a)
            revenue = book_b.best_bid * (1 - fee_b)
            profit_pct = (revenue - cost) / cost
            
            if profit_pct > self.min_profit:
                return {
                    "direction": "buy_A_sell_B",
                    "profit_pct": profit_pct,
                    "buy_price": book_a.best_ask,
                    "sell_price": book_b.best_bid
                }
        
        # Buy on B, sell on A
        if book_b.best_ask and book_a.best_bid:
            cost = book_b.best_ask * (1 + fee_b)
            revenue = book_a.best_bid * (1 - fee_a)
            profit_pct = (revenue - cost) / cost
            
            if profit_pct > self.min_profit:
                return {
                    "direction": "buy_B_sell_A",
                    "profit_pct": profit_pct,
                    "buy_price": book_b.best_ask,
                    "sell_price": book_a.best_bid
                }
        return None
```

**Typical opportunity**: 1-10 bps. After fees (10-20 bps round-trip), often nothing left. Need VIP fee tiers.

### 2. Spot-Futures Basis Trading

Spot price and futures price should converge at expiry. If futures trade at premium to spot: sell futures, buy spot.

```
Basis = futures_price - spot_price
Expected basis at expiry = 0

If basis > carrying cost (fees + interest): sell futures + buy spot → convergence profit
If basis < 0 (backwardation): buy futures + sell spot → convergence profit
```

**Perpetual version**: funding rate arbitrage
```
If funding_rate > 0: long spot + short perp → earn funding every 8h
If funding_rate < 0: short spot + long perp → earn funding every 8h

Risk: basis can widen before converging. Position must survive marked-to-market swings.
Annualized yield typically: 5-40% depending on market sentiment
```

### 3. Market Making on Crypto CEX

Quote bid and ask, earn spread. Requires:
- Fast cancel (before adverse price moves)
- Inventory management (don't get stuck holding too much in one direction)
- Smart quoting (asymmetric around inventory)

```python
class CryptoMarketMaker:
    def __init__(self, symbol, target_spread_bps=2.0, max_inventory_usd=10000):
        self.symbol = symbol
        self.target_spread = target_spread_bps / 10000
        self.max_inv = max_inventory_usd
        self.inventory = 0.0  # in BTC
    
    def compute_quotes(self, mid_price, inventory_btc, volatility):
        # Avellaneda-Stoikov inspired: skew based on inventory
        gamma = 0.5  # risk aversion
        inv_skew = gamma * volatility * inventory_btc
        
        reservation_price = mid_price - inv_skew
        half_spread = max(self.target_spread * mid_price / 2, 
                         volatility * 0.1)  # floor spread at 10% of vol
        
        bid = reservation_price - half_spread
        ask = reservation_price + half_spread
        
        return bid, ask
    
    def on_fill(self, side, qty, price):
        if side == "BUY":
            self.inventory += qty
        else:
            self.inventory -= qty
```

### 4. Statistical Arbitrage (Short-Term)

Same as explained in [05_quant_trading.md](05_quant_trading.md), but at second-to-minute frequency.

**Crypto stat arb pairs** with consistent cointegration:
- BTC/ETH
- ETH/ETH-staking token (stETH)
- Spot/futures basis
- BTC/WBTC (wrapped bitcoin on Ethereum)
- Same asset cross-exchange (Binance BTC vs Coinbase BTC)

### 5. Liquidation Sniping

Monitor exchange liquidation feeds. When large liquidation occurs: price moves sharply. Predict the cascade, trade ahead.

```python
# Binance provides liquidation feed
# wss://fstream.binance.com/ws/!forceOrder@arr

async def monitor_liquidations():
    url = "wss://fstream.binance.com/ws/!forceOrder@arr"
    async with websockets.connect(url) as ws:
        async for msg in ws:
            data = json.loads(msg)
            order = data["o"]
            symbol = order["s"]
            side = order["S"]      # which side was liquidated
            qty = float(order["q"])
            price = float(order["ap"])  # avg fill price
            
            usd_value = qty * price
            if usd_value > 1_000_000:  # $1M+ liquidation
                # Large long liquidated → expect more selling → consider short
                # Large short liquidated → expect more buying → consider long
                yield {"symbol": symbol, "side": side, "usd_value": usd_value}
```

---

## On-Chain HFT (MEV)

If trading on DEXes, you're competing with MEV searchers.

### MEV Bot Architecture

```
1. Subscribe to mempool (Ethereum pending transactions)
2. Decode pending transaction (which DEX, which tokens, size)
3. Simulate result (use eth_call on pending state)
4. Compute your profit opportunity
5. Construct bundle (your tx + target tx)
6. Send to Flashbots relay (off-chain auction)
7. If your bid wins → included in next block
```

Tools for MEV:
- **Flashbots MEV-Share**: receive hints about pending txs in exchange for profit share
- **Blocknative**: mempool monitoring service
- **Ethers.js** / **Alloy (Rust)**: simulate and construct transactions

### Arbitrage Bot (Simple DEX-to-DEX)

```python
# Pseudocode — actual implementation needs web3.py + careful gas management
async def find_dex_arb(token_a, token_b, amount):
    # Get quotes from multiple DEXes
    price_uniswap = await uniswap.get_output(token_a, token_b, amount)
    price_sushiswap = await sushiswap.get_output(token_a, token_b, amount)
    
    best_buy = min(price_uniswap, price_sushiswap)  # which is cheaper to buy
    best_sell = max(price_uniswap, price_sushiswap)  # which pays more
    
    profit = best_sell - best_buy - gas_cost - protocol_fees
    if profit > 0:
        # Execute: buy on cheaper DEX, sell on expensive DEX
        # Use flash loan if needed (no capital requirement)
        execute_arb(...)
```

### Reality of MEV Competition

MEV is extremely competitive. Top bots (Jaredfromsubway, Uncle in disguise) extract millions per day. Breaking in requires:
1. Fast node (< 50ms to mempool broadcast)
2. Efficient simulation
3. Smart bidding strategy
4. Backrunning opportunities (lower competition than front-running)

---

## Data Infrastructure for Crypto HFT

### What Data You Need

| Data | Frequency | Source |
|---|---|---|
| Order book L2 | Tick (10-100ms) | Exchange WebSocket |
| Trades | Tick | Exchange WebSocket |
| Funding rate | Every 8h | REST API |
| Open interest | Per minute | REST API |
| Liquidations | Real-time | Exchange WebSocket |
| OHLCV (candles) | 1m, 5m, 1h, 1d | REST API |
| On-chain flows | Per block (~12s) | Node / Glassnode |
| Sentiment | Hourly | CryptoQuant, Santiment |

### Data Storage

```
Raw tick data:
  Format: Parquet or binary columnar
  Typical daily BTC/USDT L2 data: 5-20 GB/day (full depth, 100ms)
  
Backtesting:
  Need 1-3 years of tick data
  Compressed: ~1-5 TB for major pairs
  
Tools:
  InfluxDB: time-series, good for OHLCV
  Arctic (Man Group): excellent for tick data, built on MongoDB
  Parquet + DuckDB: fast analytical queries on historical ticks
  KDB+/q: industry standard for HFT time-series (expensive license)
```

### Normalizing Multi-Exchange Data

Different exchanges have different:
- Precision (decimal places)
- Timestamp formats (ms vs μs)
- Symbol naming (BTC-USDT vs BTCUSDT vs BTC/USDT)
- Order update semantics (snapshot vs delta)

Build a normalization layer:
```python
@dataclass
class NormalizedBookUpdate:
    exchange: str          # "binance", "okx", "kraken"
    symbol: str            # normalized: "BTC-USDT"
    timestamp_ns: int      # nanoseconds since epoch
    bids: list[tuple[float, float]]  # [(price, qty), ...]
    asks: list[tuple[float, float]]
    is_snapshot: bool      # full snapshot vs delta update
```

---

## Latency Optimization for Crypto

### Python Limitations

Python's GIL and interpreted nature cap performance. For HFT:

```
Python async/websocket round-trip: 10-100ms
Python order book update: 1-10ms
Python signal computation: 0.1-10ms

Acceptable for: research, position-size trading, strategies with second-level signals
Not acceptable for: market making on major pairs, sub-second arbitrage
```

### C++ Binance Integration Sketch

```cpp
// Using uWebSockets or libwebsockets for WebSocket
// JSON parsing: simdjson (fastest JSON parser, 2.5 GB/s)

#include <simdjson.h>

void handle_depth_update(const std::string_view& raw_json) {
    simdjson::ondemand::parser parser;
    auto doc = parser.iterate(raw_json);
    
    auto bids = doc["b"].get_array();
    for (auto bid : bids) {
        auto arr = bid.get_array();
        double price = std::stod(std::string(arr.at(0).get_string().value()));
        double qty = std::stod(std::string(arr.at(1).get_string().value()));
        
        if (qty == 0.0) {
            order_book_.remove_bid(price);
        } else {
            order_book_.update_bid(price, qty);
        }
    }
    
    signal_engine_.on_update(order_book_);
}
```

**simdjson** parses JSON at 2.5+ GB/s vs Python's json module at ~50 MB/s = 50x faster.

---

## Risk Management Specific to Crypto

### Exchange Counterparty Risk

**FTX (2022)**: $8B missing. All funds on exchange: gone.

Mitigations:
1. Never hold more on exchange than you need for current strategy
2. Withdraw profits regularly to cold wallet
3. Diversify across exchanges
4. Monitor exchange health (Nansen, exchange reserve flows, social media)
5. Use exchanges with proof-of-reserves (Kraken, Coinbase)

### Smart Contract Risk (DEX)

Code bugs → total loss. Major exploits 2022-2023: $3B+.

Mitigations:
1. Use audited protocols only (Uniswap, Aave, Compound)
2. Time-locked or multi-sig admin keys
3. Cap amount per protocol
4. Monitor exploit alerts (Harpie, Forta, community Discord)

### Liquidation Risk

On leveraged positions: if margin ratio drops below maintenance margin, exchange liquidates.

```
Binance perpetual liquidation:
  Maintenance margin rate: 0.40% for BTC (tier 1)
  
  At 10x leverage:
    Initial margin: 10%
    Maintenance: 0.40%
    Liquidation price (long): entry × (1 - 1/leverage + maintenance_rate)
    
  Example: Long BTC at $47,000, 10x leverage
    Liquidation at: 47,000 × (1 - 0.1 + 0.004) = 47,000 × 0.904 = $42,488
```

---

## Practical Starting Point

If building a crypto HFT system from scratch:

**Phase 1** (week 1-2): Data infrastructure
- Subscribe to Binance WebSocket (order book + trades)
- Store normalized tick data
- Build order book reconstruction + validation

**Phase 2** (week 3-4): Signal research
- Compute OFI, imbalance, spread signals
- Backtest on stored tick data
- Measure IC at 10s, 30s, 1m, 5m horizons

**Phase 3** (month 2): Paper trading
- Connect to exchange test API (Binance testnet)
- Implement order management
- Run strategy in simulation with real market data

**Phase 4** (month 3+): Live trading
- Start with tiny size ($100-1000)
- Monitor execution quality vs backtest
- Scale size only after proving live performance

---

## Next

Read [09_build_quant_sim.md](09_build_quant_sim.md) to see how to connect all this knowledge into the existing C++ + Python simulator in this repo.
