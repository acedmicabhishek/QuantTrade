# 12 — Market Simulation: Multi-Agent Exchange with Low-Latency Strategy Submission

Build a synthetic exchange where dozens of bots trade stocks and derivatives, prices emerge naturally from their order flow, and you can plug in your own strategy to compete against them in simulated time.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        SIMULATION CLOCK                          │
│          Advances in discrete ticks (or wall-clock time)         │
└───────────────────────────┬─────────────────────────────────────┘
                            │ tick events
          ┌─────────────────▼──────────────────┐
          │           EVENT BUS                 │
          │  (asyncio Queue / priority queue)   │
          └──┬──────────────┬──────────────────┘
             │              │
  ┌──────────▼───┐   ┌──────▼──────────────────────────────────┐
  │  BOT AGENTS  │   │           EXCHANGE CORE                  │
  │              │   │  ┌──────────────────────────────────┐   │
  │ MarketMaker  │   │  │  Order Book (per instrument)     │   │
  │ Momentum     │   │  │  - Bid heap (max-heap)           │   │
  │ MeanRevert   │   │  │  - Ask heap (min-heap)           │   │
  │ NoiseTrader  │   │  │  - Price-time priority           │   │
  │ FundTrader   │   │  └──────────────┬───────────────────┘   │
  │ OptionBot    │   │                 │ matched trades         │
  └──────────────┘   │  ┌──────────────▼───────────────────┐   │
         │           │  │  Matching Engine                  │   │
  ┌──────▼────────┐  │  │  - Fill logic (partial/full)     │   │
  │  USER STRATEGY│  │  │  - Trade tape (time & sales)     │   │
  │  (your code)  │  │  └──────────────┬───────────────────┘   │
  │  via Strategy │  │                 │                        │
  │  API          │  │  ┌──────────────▼───────────────────┐   │
  └───────────────┘  │  │  Risk Manager                    │   │
                     │  │  - Per-agent position limits     │   │
                     │  │  - Margin for derivatives        │   │
                     │  └──────────────────────────────────┘   │
                     └─────────────────────────────────────────┘
                                        │
                     ┌──────────────────▼──────────────────────┐
                     │           MARKET DATA FEED               │
                     │  Last price, OHLCV, order book snapshot  │
                     │  Greeks feed (for options instruments)   │
                     └─────────────────────────────────────────┘
```

---

## 2. Core Data Structures

### 2.1 Order

```python
# sim/core/order.py
from dataclasses import dataclass, field
from enum import Enum, auto
import time

class Side(Enum):
    BUY  = auto()
    SELL = auto()

class OrderType(Enum):
    MARKET = auto()
    LIMIT  = auto()
    STOP   = auto()    # converts to market when stop price hit
    IOC    = auto()    # immediate-or-cancel
    FOK    = auto()    # fill-or-kill

class OrderStatus(Enum):
    PENDING   = auto()
    OPEN      = auto()
    PARTIAL   = auto()
    FILLED    = auto()
    CANCELLED = auto()
    REJECTED  = auto()

@dataclass
class Order:
    order_id   : int
    agent_id   : str
    instrument : str
    side       : Side
    order_type : OrderType
    quantity   : int
    price      : float          # limit price; ignored for MARKET
    stop_price : float = 0.0
    timestamp  : int   = field(default_factory=lambda: time.time_ns())
    filled_qty : int   = 0
    status     : OrderStatus = OrderStatus.PENDING

    @property
    def remaining(self):
        return self.quantity - self.filled_qty

    @property
    def is_active(self):
        return self.status in (OrderStatus.OPEN, OrderStatus.PARTIAL)
```

### 2.2 Trade (Fill)

```python
# sim/core/trade.py
from dataclasses import dataclass

@dataclass
class Trade:
    trade_id      : int
    instrument    : str
    buyer_id      : str
    seller_id     : str
    buyer_order_id : int
    seller_order_id: int
    price         : float
    quantity      : int
    timestamp     : int   # nanoseconds
```

### 2.3 Limit Order Book

```python
# sim/core/orderbook.py
import heapq
import time
from collections import defaultdict
from typing import List, Optional, Tuple
from .order import Order, Side, OrderType, OrderStatus
from .trade import Trade

class LimitOrderBook:
    """
    Price-time priority order book.
    Bids: max-heap  (negate price so Python's min-heap acts as max-heap)
    Asks: min-heap
    """

    def __init__(self, instrument: str):
        self.instrument = instrument
        self._bids: list  = []   # heap of (-price, timestamp, order)
        self._asks: list  = []   # heap of ( price, timestamp, order)
        self._orders: dict = {}  # order_id → Order
        self._trade_counter = 0
        self.trades: List[Trade] = []
        self.last_price: Optional[float] = None

    # ── public API ────────────────────────────────────────────────

    def add_order(self, order: Order) -> List[Trade]:
        order.status = OrderStatus.OPEN
        self._orders[order.order_id] = order

        if order.order_type == OrderType.MARKET:
            return self._match_market(order)
        elif order.order_type == OrderType.LIMIT:
            fills = self._match_limit(order)
            if order.remaining > 0 and order.status != OrderStatus.FILLED:
                self._insert_resting(order)
            return fills
        elif order.order_type == OrderType.IOC:
            fills = self._match_limit(order)
            if order.remaining > 0:
                order.status = OrderStatus.CANCELLED
            return fills
        elif order.order_type == OrderType.FOK:
            if not self._can_fill_fully(order):
                order.status = OrderStatus.REJECTED
                return []
            return self._match_limit(order)
        return []

    def cancel_order(self, order_id: int) -> bool:
        order = self._orders.get(order_id)
        if order and order.is_active:
            order.status = OrderStatus.CANCELLED
            return True
        return False

    @property
    def best_bid(self) -> Optional[float]:
        self._prune(self._bids, negate=True)
        return -self._bids[0][0] if self._bids else None

    @property
    def best_ask(self) -> Optional[float]:
        self._prune(self._asks, negate=False)
        return self._asks[0][0] if self._asks else None

    @property
    def spread(self) -> Optional[float]:
        if self.best_bid and self.best_ask:
            return self.best_ask - self.best_bid
        return None

    def book_snapshot(self, depth=5) -> dict:
        """Return top N levels on each side."""
        bids, asks = defaultdict(int), defaultdict(int)
        for (neg_p, _, o) in self._bids:
            if o.is_active:
                bids[round(-neg_p, 2)] += o.remaining
        for (p, _, o) in self._asks:
            if o.is_active:
                asks[round(p, 2)] += o.remaining
        return {
            "bids": sorted(bids.items(), reverse=True)[:depth],
            "asks": sorted(asks.items())[:depth],
        }

    # ── private matching ──────────────────────────────────────────

    def _match_market(self, aggressor: Order) -> List[Trade]:
        heap = self._asks if aggressor.side == Side.BUY else self._bids
        negate = aggressor.side == Side.SELL
        return self._drain_heap(aggressor, heap, negate)

    def _match_limit(self, aggressor: Order) -> List[Trade]:
        if aggressor.side == Side.BUY:
            heap, negate = self._asks, False
            crossable = lambda ask_p: ask_p <= aggressor.price
        else:
            heap, negate = self._bids, True
            crossable = lambda neg_p: (-neg_p) >= aggressor.price
        return self._drain_heap(aggressor, heap, negate, crossable)

    def _drain_heap(self, aggressor, heap, negate, crossable=None):
        fills = []
        while heap and aggressor.remaining > 0:
            self._prune(heap, negate)
            if not heap:
                break
            top_key = heap[0][0]
            price   = -top_key if negate else top_key
            if crossable and not crossable(top_key):
                break
            _, _, resting = heapq.heappop(heap)
            if not resting.is_active:
                continue
            fill_qty  = min(aggressor.remaining, resting.remaining)
            fill_price = resting.price   # resting order sets the price
            self._apply_fill(aggressor, resting, fill_qty, fill_price, fills)
            if resting.remaining > 0:
                key = (-resting.price, resting.timestamp, resting) if negate \
                      else (resting.price, resting.timestamp, resting)
                heapq.heappush(heap, key)
        return fills

    def _apply_fill(self, aggressor, resting, qty, price, fills):
        aggressor.filled_qty += qty
        resting.filled_qty   += qty
        aggressor.status = OrderStatus.FILLED if aggressor.remaining == 0 else OrderStatus.PARTIAL
        resting.status   = OrderStatus.FILLED if resting.remaining == 0   else OrderStatus.PARTIAL
        self.last_price  = price
        self._trade_counter += 1
        t = Trade(
            trade_id       = self._trade_counter,
            instrument     = self.instrument,
            buyer_id       = aggressor.agent_id if aggressor.side == Side.BUY else resting.agent_id,
            seller_id      = resting.agent_id   if aggressor.side == Side.BUY else aggressor.agent_id,
            buyer_order_id = aggressor.order_id if aggressor.side == Side.BUY else resting.order_id,
            seller_order_id= resting.order_id   if aggressor.side == Side.BUY else aggressor.order_id,
            price    = price,
            quantity = qty,
            timestamp= time.time_ns(),
        )
        self.trades.append(t)
        fills.append(t)

    def _insert_resting(self, order: Order):
        if order.side == Side.BUY:
            heapq.heappush(self._bids, (-order.price, order.timestamp, order))
        else:
            heapq.heappush(self._asks, (order.price, order.timestamp, order))

    def _prune(self, heap, negate):
        while heap and not heap[0][2].is_active:
            heapq.heappop(heap)

    def _can_fill_fully(self, order: Order) -> bool:
        available = 0
        if order.side == Side.BUY:
            for (p, _, o) in sorted(self._asks):
                if o.is_active and p <= order.price:
                    available += o.remaining
        else:
            for (neg_p, _, o) in sorted(self._bids):
                if o.is_active and (-neg_p) >= order.price:
                    available += o.remaining
        return available >= order.quantity
```

---

## 3. Exchange Core

```python
# sim/exchange.py
import time
import itertools
from typing import Dict, List, Optional
from .core.order import Order, Side, OrderType, OrderStatus
from .core.orderbook import LimitOrderBook
from .core.trade import Trade

class Exchange:
    """Central exchange. Maintains one order book per instrument."""

    def __init__(self):
        self._books: Dict[str, LimitOrderBook] = {}
        self._order_counter = itertools.count(1)
        self._agents: Dict[str, "Agent"] = {}
        self.tape: List[Trade] = []          # global trade tape

    def register_instrument(self, symbol: str):
        self._books[symbol] = LimitOrderBook(symbol)

    def register_agent(self, agent):
        self._agents[agent.agent_id] = agent
        agent.exchange = self

    def submit_order(self,
                     agent_id   : str,
                     instrument : str,
                     side       : Side,
                     order_type : OrderType,
                     quantity   : int,
                     price      : float = 0.0,
                     stop_price : float = 0.0) -> Order:

        t0 = time.time_ns()
        book = self._books.get(instrument)
        if book is None:
            raise ValueError(f"Unknown instrument: {instrument}")

        order = Order(
            order_id   = next(self._order_counter),
            agent_id   = agent_id,
            instrument = instrument,
            side       = side,
            order_type = order_type,
            quantity   = quantity,
            price      = price,
            stop_price = stop_price,
            timestamp  = t0,
        )

        fills = book.add_order(order)
        self.tape.extend(fills)

        # Notify the agents involved in each fill
        for trade in fills:
            for aid in (trade.buyer_id, trade.seller_id):
                if aid in self._agents:
                    self._agents[aid].on_fill(trade)

        # Record simulated latency (nanoseconds)
        order._latency_ns = time.time_ns() - t0
        return order

    def cancel_order(self, order_id: int, instrument: str) -> bool:
        book = self._books.get(instrument)
        return book.cancel_order(order_id) if book else False

    def market_data(self, instrument: str) -> dict:
        book = self._books.get(instrument)
        if not book:
            return {}
        return {
            "instrument" : instrument,
            "last_price" : book.last_price,
            "best_bid"   : book.best_bid,
            "best_ask"   : book.best_ask,
            "spread"     : book.spread,
            "book"       : book.book_snapshot(),
        }
```

---

## 4. Agent Base Class

Every participant — bot or user strategy — inherits from this.

```python
# sim/agents/base.py
from abc import ABC, abstractmethod
from ..core.order import Side, OrderType
from ..core.trade import Trade

class Agent(ABC):
    def __init__(self, agent_id: str, cash: float = 100_000.0):
        self.agent_id  = agent_id
        self.cash      = cash
        self.positions : dict = {}   # instrument → net quantity (signed)
        self.pnl       : float = 0.0
        self.exchange  = None        # injected by Exchange.register_agent

    # ── helpers ──────────────────────────────────────────────────

    def buy(self, instrument, qty, price=0.0,
            order_type=OrderType.LIMIT):
        return self.exchange.submit_order(
            self.agent_id, instrument,
            Side.BUY, order_type, qty, price)

    def sell(self, instrument, qty, price=0.0,
             order_type=OrderType.LIMIT):
        return self.exchange.submit_order(
            self.agent_id, instrument,
            Side.SELL, order_type, qty, price)

    def market_buy(self, instrument, qty):
        return self.exchange.submit_order(
            self.agent_id, instrument,
            Side.BUY, OrderType.MARKET, qty)

    def market_sell(self, instrument, qty):
        return self.exchange.submit_order(
            self.agent_id, instrument,
            Side.SELL, OrderType.MARKET, qty)

    def md(self, instrument) -> dict:
        return self.exchange.market_data(instrument)

    def position(self, instrument) -> int:
        return self.positions.get(instrument, 0)

    # ── callbacks (override as needed) ───────────────────────────

    def on_fill(self, trade: Trade):
        """Called by exchange whenever one of our orders is matched."""
        instrument = trade.instrument
        if trade.buyer_id == self.agent_id:
            self.positions[instrument] = self.positions.get(instrument, 0) + trade.quantity
            self.cash -= trade.price * trade.quantity
        else:
            self.positions[instrument] = self.positions.get(instrument, 0) - trade.quantity
            self.cash += trade.price * trade.quantity

    # ── abstract interface ────────────────────────────────────────

    @abstractmethod
    def on_tick(self, tick: int, market_data: dict):
        """Called once per simulation tick. Submit orders here."""
        ...
```

---

## 5. Bot Agents — Market Participants

### 5.1 Noise Trader (Random Activity)

Creates realistic order flow "noise" — the foundation of price movement.

```python
# sim/agents/noise_trader.py
import random
from .base import Agent
from ..core.order import OrderType

class NoiseTrader(Agent):
    """Randomly buys and sells. No strategy. Provides liquidity noise."""

    def __init__(self, agent_id, instruments, cash=50_000):
        super().__init__(agent_id, cash)
        self.instruments = instruments
        self.trade_prob  = 0.3    # 30% chance to act per tick

    def on_tick(self, tick, market_data):
        for inst in self.instruments:
            if random.random() > self.trade_prob:
                continue
            md = market_data.get(inst, {})
            last = md.get("last_price") or 100.0
            side_fn = self.buy if random.random() < 0.5 else self.sell
            qty   = random.randint(1, 10)
            # Submit slightly off mid so it often rests in the book
            offset = random.uniform(-0.5, 0.5)
            price  = round(last + offset, 2)
            side_fn(inst, qty, price)
```

### 5.2 Market Maker

Posts tight two-sided quotes. Profits from the bid-ask spread.

```python
# sim/agents/market_maker.py
import random
from .base import Agent
from ..core.order import OrderType, Side

class MarketMaker(Agent):
    """
    Continuously posts bid and ask around the fair value.
    Adjusts quotes based on inventory to avoid directional exposure.
    """

    def __init__(self, agent_id, instrument, cash=500_000,
                 spread=0.10, order_qty=20, max_inventory=200):
        super().__init__(agent_id, cash)
        self.instrument    = instrument
        self.half_spread   = spread / 2
        self.order_qty     = order_qty
        self.max_inventory = max_inventory
        self._active_orders = {}

    def on_tick(self, tick, market_data):
        # Cancel stale quotes first
        for oid in list(self._active_orders):
            self.exchange.cancel_order(oid, self.instrument)
        self._active_orders.clear()

        md   = market_data.get(self.instrument, {})
        last = md.get("last_price") or 100.0
        inv  = self.position(self.instrument)

        # Skew quotes based on inventory (Avellaneda-Stoikov style)
        inv_skew = (inv / self.max_inventory) * self.half_spread
        bid = round(last - self.half_spread - inv_skew, 2)
        ask = round(last + self.half_spread - inv_skew, 2)

        if bid > 0 and abs(inv) < self.max_inventory:
            o1 = self.buy(self.instrument,  self.order_qty, bid)
            o2 = self.sell(self.instrument, self.order_qty, ask)
            self._active_orders[o1.order_id] = o1
            self._active_orders[o2.order_id] = o2
```

### 5.3 Momentum Trader

Chases price trends using a short vs. long EMA crossover.

```python
# sim/agents/momentum_bot.py
from collections import deque
from .base import Agent
from ..core.order import OrderType

class MomentumBot(Agent):
    """Buys on upward EMA crossover, sells on downward crossover."""

    def __init__(self, agent_id, instrument, cash=200_000,
                 fast=10, slow=30, qty=15):
        super().__init__(agent_id, cash)
        self.instrument = instrument
        self.fast, self.slow = fast, slow
        self.qty = qty
        self._prices = deque(maxlen=slow + 1)
        self._last_signal = 0   # +1 = long, -1 = short, 0 = flat

    def _ema(self, prices, span):
        if len(prices) < span:
            return None
        k, ema = 2/(span+1), prices[0]
        for p in list(prices)[1:]:
            ema = p * k + ema * (1 - k)
        return ema

    def on_tick(self, tick, market_data):
        md   = market_data.get(self.instrument, {})
        last = md.get("last_price")
        if not last:
            return
        self._prices.append(last)

        fast_ema = self._ema(list(self._prices)[-self.fast:], self.fast)
        slow_ema = self._ema(list(self._prices), self.slow)
        if fast_ema is None or slow_ema is None:
            return

        signal = 1 if fast_ema > slow_ema else -1

        if signal != self._last_signal:
            pos = self.position(self.instrument)
            if signal == 1 and pos <= 0:
                self.market_buy(self.instrument, self.qty + abs(pos))
            elif signal == -1 and pos >= 0:
                self.market_sell(self.instrument, self.qty + abs(pos))
            self._last_signal = signal
```

### 5.4 Mean Reversion Bot

Buys when price dips below N-day average, sells when it rises above.

```python
# sim/agents/mean_revert_bot.py
from collections import deque
from .base import Agent

class MeanReversionBot(Agent):
    def __init__(self, agent_id, instrument, cash=200_000,
                 window=20, z_threshold=1.5, qty=10):
        super().__init__(agent_id, cash)
        self.instrument  = instrument
        self.window      = window
        self.z_threshold = z_threshold
        self.qty         = qty
        self._prices     = deque(maxlen=window)

    def on_tick(self, tick, market_data):
        md   = market_data.get(self.instrument, {})
        last = md.get("last_price")
        if not last:
            return
        self._prices.append(last)
        if len(self._prices) < self.window:
            return

        prices = list(self._prices)
        mean   = sum(prices) / len(prices)
        std    = (sum((p - mean)**2 for p in prices) / len(prices)) ** 0.5
        if std == 0:
            return

        z_score = (last - mean) / std
        pos     = self.position(self.instrument)

        if z_score < -self.z_threshold and pos <= 0:
            self.market_buy(self.instrument, self.qty)
        elif z_score > self.z_threshold and pos >= 0:
            self.market_sell(self.instrument, self.qty)
        elif abs(z_score) < 0.3 and pos != 0:
            # Exit when back at mean
            if pos > 0:
                self.market_sell(self.instrument, abs(pos))
            else:
                self.market_buy(self.instrument, abs(pos))
```

### 5.5 Fundamental Trader

Has a "true value" belief and trades toward it — acts as a gravity pull on price.

```python
# sim/agents/fundamental_trader.py
import random
from .base import Agent
from ..core.order import OrderType

class FundamentalTrader(Agent):
    """
    Knows the 'fair value'. Buys when market price is below it,
    sells when above. Simulates institutional value investors.
    """

    def __init__(self, agent_id, instrument, fair_value, cash=1_000_000,
                 threshold=0.5, qty=25, noise_std=0.2):
        super().__init__(agent_id, cash)
        self.instrument  = instrument
        self.fair_value  = fair_value
        self.threshold   = threshold
        self.qty         = qty
        self.noise_std   = noise_std  # uncertainty in fair value estimate

    def on_tick(self, tick, market_data):
        # Fair value drifts slightly each tick (simulates earnings revisions)
        self.fair_value *= (1 + random.gauss(0, 0.0001))

        md    = market_data.get(self.instrument, {})
        last  = md.get("last_price")
        if not last:
            return

        perceived_fv = self.fair_value + random.gauss(0, self.noise_std)
        gap = perceived_fv - last

        if gap > self.threshold:
            self.buy(self.instrument, self.qty,
                     round(last + 0.05, 2), OrderType.LIMIT)
        elif gap < -self.threshold:
            self.sell(self.instrument, self.qty,
                      round(last - 0.05, 2), OrderType.LIMIT)
```

### 5.6 Options Bot

Prices options via BSM and submits quotes. Creates a live options market.

```python
# sim/agents/options_bot.py
import math
from scipy.stats import norm
from .base import Agent
from ..core.order import OrderType

class OptionsMarketMaker(Agent):
    """
    Quotes call and put options using BSM.
    Instrument naming: 'RELIANCE_CE_2500_JAN25'
    """

    def __init__(self, agent_id, underlying, strike, expiry_years,
                 option_type, r=0.065, sigma=0.25, cash=500_000):
        super().__init__(agent_id, cash)
        self.underlying   = underlying
        self.strike       = strike
        self.expiry_years = expiry_years   # decreases each tick
        self.option_type  = option_type    # 'call' or 'put'
        self.r, self.sigma = r, sigma
        self.instrument   = f"{underlying}_{option_type.upper()}_{strike}"

    def bsm_price(self, S, K, T, r, sigma, option_type):
        if T <= 0:
            if option_type == "call":
                return max(S - K, 0)
            return max(K - S, 0)
        d1 = (math.log(S/K) + (r + 0.5*sigma**2)*T) / (sigma*math.sqrt(T))
        d2 = d1 - sigma * math.sqrt(T)
        if option_type == "call":
            return S * norm.cdf(d1) - K * math.exp(-r*T) * norm.cdf(d2)
        return K * math.exp(-r*T) * norm.cdf(-d2) - S * norm.cdf(-d1)

    def on_tick(self, tick, market_data):
        md  = market_data.get(self.underlying, {})
        S   = md.get("last_price")
        if not S:
            return

        self.expiry_years = max(0.001, self.expiry_years - 1/(252*390))
        fair = self.bsm_price(S, self.strike, self.expiry_years,
                              self.r, self.sigma, self.option_type)

        spread = max(0.05, fair * 0.02)   # 2% spread, min 5 paise
        bid = round(fair - spread/2, 2)
        ask = round(fair + spread/2, 2)

        if bid > 0:
            self.buy( self.instrument, 10, bid)
            self.sell(self.instrument, 10, ask)
```

---

## 6. Simulation Engine

```python
# sim/simulation.py
import time
import random
from typing import List
from .exchange import Exchange
from .agents.base import Agent

class Simulation:
    """
    Runs the market simulation for N ticks.
    Each tick:
      1. Advance the simulation clock.
      2. Compute current market data snapshot.
      3. Call on_tick() for every agent (shuffled to avoid order bias).
      4. Process any stop orders triggered by new prices.
      5. Record OHLCV bar.
    """

    def __init__(self, exchange: Exchange, tick_interval_ms: float = 100.0):
        self.exchange         = exchange
        self.tick_interval_ms = tick_interval_ms
        self.tick             = 0
        self._ohlcv: dict     = {}   # instrument → list of bars

    def run(self, num_ticks: int, realtime: bool = False):
        agents = list(self.exchange._agents.values())

        for t in range(num_ticks):
            self.tick = t
            t_start   = time.time_ns()

            # Market data snapshot for all instruments
            all_md = {
                inst: self.exchange.market_data(inst)
                for inst in self.exchange._books
            }

            # Each agent acts — randomise order to avoid position bias
            random.shuffle(agents)
            for agent in agents:
                try:
                    agent.on_tick(t, all_md)
                except Exception as e:
                    print(f"[WARN] Agent {agent.agent_id} raised: {e}")

            # Process stop orders
            self._check_stops(all_md)

            # Record OHLCV
            self._record_ohlcv(all_md)

            # Optionally sleep to simulate real time
            if realtime:
                elapsed_ms = (time.time_ns() - t_start) / 1e6
                sleep_ms   = max(0, self.tick_interval_ms - elapsed_ms)
                time.sleep(sleep_ms / 1000)

        return self._build_results()

    def _check_stops(self, all_md):
        for inst, book in self.exchange._books.items():
            last = all_md.get(inst, {}).get("last_price")
            if not last:
                continue
            for order in list(book._orders.values()):
                if order.is_active and order.stop_price > 0:
                    triggered = (
                        (order.side.name == "BUY"  and last >= order.stop_price) or
                        (order.side.name == "SELL" and last <= order.stop_price)
                    )
                    if triggered:
                        order.stop_price = 0
                        order.order_type = __import__(
                            'sim.core.order', fromlist=['OrderType']
                        ).OrderType.MARKET
                        book.add_order(order)

    def _record_ohlcv(self, all_md):
        for inst, md in all_md.items():
            last = md.get("last_price")
            if not last:
                continue
            bars = self._ohlcv.setdefault(inst, [])
            if bars:
                bar = bars[-1]
                bar["high"]  = max(bar["high"], last)
                bar["low"]   = min(bar["low"],  last)
                bar["close"] = last
                bar["volume"]+= sum(t.quantity for t in self.exchange._books[inst].trades[-5:])
            else:
                bars.append({"open": last, "high": last,
                             "low": last, "close": last, "volume": 0})

    def _build_results(self) -> dict:
        results = {"ohlcv": self._ohlcv, "agents": {}}
        for aid, agent in self.exchange._agents.items():
            results["agents"][aid] = {
                "cash"      : round(agent.cash, 2),
                "positions" : agent.positions,
                "pnl"       : round(agent.pnl, 2),
            }
        return results
```

---

## 7. User Strategy API — Plug In Your Own Code

This is the interface a user submits. Inherit `UserStrategy`, implement `on_tick`, done.

```python
# sim/agents/user_strategy.py
from .base import Agent
from ..core.order import OrderType, Side
from ..core.trade import Trade
from typing import Optional

class UserStrategy(Agent):
    """
    Base class for user-submitted strategies.

    Available inside on_tick:
      self.buy(instrument, qty, price)           → submit limit buy
      self.sell(instrument, qty, price)          → submit limit sell
      self.market_buy(instrument, qty)           → market buy
      self.market_sell(instrument, qty)          → market sell
      self.md(instrument)                        → market data dict
      self.position(instrument)                  → current net position
      self.cash                                  → available cash
      tick                                       → current tick number
      market_data                                → all instruments snapshot

    market_data[instrument] keys:
      last_price, best_bid, best_ask, spread,
      book: {"bids": [(price, qty)...], "asks": [(price, qty)...]}
    """

    def on_fill(self, trade: Trade):
        super().on_fill(trade)   # updates cash and positions
        self.on_my_fill(trade)   # hook for user to override

    def on_my_fill(self, trade: Trade):
        """Override to react to your own fills."""
        pass

    def on_tick(self, tick: int, market_data: dict):
        raise NotImplementedError("Implement on_tick in your strategy")
```

### Example User Strategy — RSI Reversal

```python
# my_strategy.py  ← this is what you write and submit
from collections import deque
from sim.agents.user_strategy import UserStrategy

class MyRSIStrategy(UserStrategy):

    def __init__(self):
        super().__init__(agent_id="my_strategy", cash=500_000)
        self.instrument = "RELIANCE"
        self._prices    = deque(maxlen=15)
        self._active_orders = []

    def _rsi(self, prices, period=14):
        if len(prices) < period + 1:
            return None
        prices = list(prices)
        deltas = [prices[i] - prices[i-1] for i in range(1, len(prices))]
        gains  = [max(d, 0) for d in deltas]
        losses = [abs(min(d, 0)) for d in deltas]
        avg_gain = sum(gains[-period:])  / period
        avg_loss = sum(losses[-period:]) / period
        if avg_loss == 0:
            return 100
        rs = avg_gain / avg_loss
        return 100 - 100 / (1 + rs)

    def on_tick(self, tick, market_data):
        md   = market_data.get(self.instrument, {})
        last = md.get("last_price")
        if not last:
            return

        self._prices.append(last)
        rsi = self._rsi(self._prices)
        if rsi is None:
            return

        pos = self.position(self.instrument)

        # Oversold — buy
        if rsi < 30 and pos == 0 and self.cash > last * 50:
            order = self.buy(self.instrument, 50, round(last + 0.10, 2))
            self._active_orders.append(order)
            print(f"[{tick:5d}] BUY  50 @ {last:.2f}  RSI={rsi:.1f}")

        # Overbought — sell
        elif rsi > 70 and pos > 0:
            order = self.sell(self.instrument, pos, round(last - 0.10, 2))
            self._active_orders.append(order)
            print(f"[{tick:5d}] SELL {pos} @ {last:.2f}  RSI={rsi:.1f}")

    def on_my_fill(self, trade):
        print(f"  → FILL {trade.quantity} @ ₹{trade.price:.2f}")
```

---

## 8. Running the Full Simulation

```python
# run_simulation.py
from sim.exchange import Exchange
from sim.simulation import Simulation
from sim.agents.noise_trader import NoiseTrader
from sim.agents.market_maker import MarketMaker
from sim.agents.momentum_bot import MomentumBot
from sim.agents.mean_revert_bot import MeanReversionBot
from sim.agents.fundamental_trader import FundamentalTrader
from sim.agents.options_bot import OptionsMarketMaker
from my_strategy import MyRSIStrategy   # ← user-submitted strategy
import pandas as pd
import numpy as np

# ── 1. Create exchange and instruments ─────────────────────────
exchange = Exchange()
exchange.register_instrument("RELIANCE")
exchange.register_instrument("TCS")
exchange.register_instrument("RELIANCE_CALL_2500")   # options

# ── 2. Seed the order book with initial quotes ──────────────────
def seed_book(exchange, instrument, mid_price, levels=5):
    """Place initial resting limit orders so the book is not empty."""
    from sim.core.order import Order, Side, OrderType
    import itertools
    ctr = itertools.count(10_000)
    for i in range(1, levels + 1):
        bid = Order(next(ctr), "SEED", instrument, Side.BUY,
                    OrderType.LIMIT, 100, round(mid_price - i * 0.05, 2))
        ask = Order(next(ctr), "SEED", instrument, Side.SELL,
                    OrderType.LIMIT, 100, round(mid_price + i * 0.05, 2))
        exchange._books[instrument].add_order(bid)
        exchange._books[instrument].add_order(ask)

seed_book(exchange, "RELIANCE", 2480.0)
seed_book(exchange, "TCS",      3900.0)

# ── 3. Register bots ────────────────────────────────────────────
bots = [
    # 3 market makers per stock
    MarketMaker("MM_REL_1", "RELIANCE", spread=0.10, order_qty=30),
    MarketMaker("MM_REL_2", "RELIANCE", spread=0.15, order_qty=20),
    MarketMaker("MM_TCS_1", "TCS",      spread=0.12, order_qty=25),

    # Noise traders
    NoiseTrader("NOISE_1", ["RELIANCE", "TCS"], cash=100_000),
    NoiseTrader("NOISE_2", ["RELIANCE"],        cash=80_000),
    NoiseTrader("NOISE_3", ["TCS"],             cash=60_000),

    # Momentum bots
    MomentumBot("MOM_REL", "RELIANCE", fast=8, slow=21, qty=20),
    MomentumBot("MOM_TCS", "TCS",      fast=5, slow=20, qty=15),

    # Mean reversion bots
    MeanReversionBot("MR_REL",  "RELIANCE", window=20, z_threshold=1.5),
    MeanReversionBot("MR_TCS",  "TCS",      window=15, z_threshold=2.0),

    # Fundamental traders — pull price toward fair value
    FundamentalTrader("FUND_REL", "RELIANCE", fair_value=2500.0, qty=30),
    FundamentalTrader("FUND_TCS", "TCS",      fair_value=3950.0, qty=20),

    # Options market maker
    OptionsMarketMaker("OPT_MM_1", "RELIANCE", strike=2500,
                       expiry_years=0.083, option_type="call"),

    # User strategy
    MyRSIStrategy(),
]

for bot in bots:
    exchange.register_agent(bot)

# ── 4. Run ──────────────────────────────────────────────────────
sim     = Simulation(exchange, tick_interval_ms=0)   # 0 = max speed
results = sim.run(num_ticks=5000)

# ── 5. Analyse results ──────────────────────────────────────────
print("\n=== AGENT P&L ===")
for aid, stats in sorted(results["agents"].items()):
    pos_str = ", ".join(f"{k}:{v}" for k,v in stats["positions"].items() if v != 0)
    print(f"{aid:<20} Cash: ₹{stats['cash']:>12,.2f}  Pos: {pos_str or 'flat'}")

# Build OHLCV DataFrame for RELIANCE
rel_bars = pd.DataFrame(results["ohlcv"]["RELIANCE"])
print(f"\nRELIANCE — {len(rel_bars)} ticks")
print(rel_bars.tail(10))

# Price path stats
prices = rel_bars["close"].dropna()
returns = np.log(prices / prices.shift(1)).dropna()
print(f"\nSimulated annualised vol: {returns.std() * np.sqrt(252*390):.2%}")
```

---

## 9. Low-Latency Design

### 9.1 Measuring Simulated Latency

Every order in the exchange carries a `_latency_ns` field populated at submission time. Use it to benchmark your strategy's reaction speed.

```python
# After simulation
all_orders = [
    o for book in exchange._books.values()
    for o in book._orders.values()
    if hasattr(o, "_latency_ns")
]
latencies_us = [o._latency_ns / 1000 for o in all_orders]
print(f"Median order latency : {sorted(latencies_us)[len(latencies_us)//2]:.1f} µs")
print(f"99th pct latency     : {sorted(latencies_us)[int(len(latencies_us)*0.99)]:.1f} µs")
```

### 9.2 Async Strategy Interface

For strategies that react to events rather than polling, use the async interface.

```python
# sim/agents/async_strategy.py
import asyncio
from .base import Agent
from ..core.trade import Trade

class AsyncStrategy(Agent):
    """
    Event-driven strategy: reacts to market data pushes instead of polling.
    Simulates WebSocket-based algo deployment on Kite Connect.
    """

    def __init__(self, agent_id, cash=500_000):
        super().__init__(agent_id, cash)
        self._event_queue = asyncio.Queue()
        self._running     = False

    async def start(self):
        self._running = True
        while self._running:
            event = await asyncio.wait_for(self._event_queue.get(), timeout=1.0)
            if event["type"] == "tick":
                await self.handle_tick(event["tick"], event["market_data"])
            elif event["type"] == "fill":
                await self.handle_fill(event["trade"])

    def on_tick(self, tick, market_data):
        self._event_queue.put_nowait({
            "type": "tick", "tick": tick, "market_data": market_data
        })

    def on_fill(self, trade: Trade):
        super().on_fill(trade)
        self._event_queue.put_nowait({"type": "fill", "trade": trade})

    async def handle_tick(self, tick, market_data):
        raise NotImplementedError

    async def handle_fill(self, trade):
        pass


# Example async strategy
class FastScalper(AsyncStrategy):
    async def handle_tick(self, tick, market_data):
        md  = market_data.get("RELIANCE", {})
        bid = md.get("best_bid")
        ask = md.get("best_ask")
        if bid and ask and (ask - bid) > 0.20:
            # Spread is wide — jump inside to capture
            inside_bid = round(bid + 0.05, 2)
            inside_ask = round(ask - 0.05, 2)
            self.buy( "RELIANCE", 5, inside_bid)
            self.sell("RELIANCE", 5, inside_ask)

    async def handle_fill(self, trade):
        print(f"SCALPER FILL: {trade.quantity} @ ₹{trade.price:.2f}")
```

### 9.3 Critical Path Optimisations (for the Matching Engine)

In Python, the above is fast enough for research (~100k orders/second). For production-grade speed:

| Technique | What it gives you |
|---|---|
| Replace `list` heaps with C++ `std::priority_queue` via pybind11 | 10–100× faster matching |
| Lock-free order book using atomic CAS operations | Eliminates GIL contention in multi-thread |
| Pre-allocated order pool (object pool pattern) | Zero malloc on hot path |
| Binary protocol (FIX/ITCH) over loopback socket | Realistic network latency simulation |
| `time.time_ns()` → RDTSC counter | Sub-nanosecond timestamp resolution |
| `asyncio` → `uvloop` | 2–4× faster event loop on Linux |

See [07_hft_low_latency.md](07_hft_low_latency.md) for the full C++ implementation of the matching engine this simulator can use as its backend.

---

## 10. Evaluating Your Strategy Against the Bots

After the simulation runs, compute these metrics for your agent specifically:

```python
def eval_user_strategy(exchange, agent_id, instrument):
    agent = exchange._agents[agent_id]
    trades = [t for t in exchange.tape
              if t.buyer_id == agent_id or t.seller_id == agent_id]

    if not trades:
        print("No trades executed.")
        return

    # Realised P&L from completed round-trips
    pnl_series = []
    pos, cost_basis = 0, 0.0
    for t in sorted(trades, key=lambda x: x.timestamp):
        if t.buyer_id == agent_id:
            cost_basis = (cost_basis * pos + t.price * t.quantity) / (pos + t.quantity) if pos >= 0 else cost_basis
            pos += t.quantity
        else:
            realised = (t.price - cost_basis) * t.quantity
            pnl_series.append(realised)
            pos -= t.quantity

    md     = exchange.market_data(instrument)
    last_p = md.get("last_price", 0)
    unrealised = (last_p - cost_basis) * pos if pos > 0 else (cost_basis - last_p) * abs(pos)

    print(f"\n=== Strategy Report: {agent_id} ===")
    print(f"Total trades          : {len(trades)}")
    print(f"Realised P&L          : ₹{sum(pnl_series):,.2f}")
    print(f"Unrealised P&L        : ₹{unrealised:,.2f}")
    print(f"Remaining cash        : ₹{agent.cash:,.2f}")
    print(f"Open position         : {agent.position(instrument)} shares")
    if pnl_series:
        wins = sum(1 for p in pnl_series if p > 0)
        print(f"Win rate              : {wins/len(pnl_series):.1%}")
        print(f"Avg profit per trade  : ₹{sum(pnl_series)/len(pnl_series):,.2f}")

eval_user_strategy(exchange, "my_strategy", "RELIANCE")
```

---

## 11. Project File Layout

```
market_sim/
├── sim/
│   ├── __init__.py
│   ├── exchange.py             ← Exchange, order routing
│   ├── simulation.py           ← Simulation engine, tick loop
│   ├── core/
│   │   ├── order.py            ← Order dataclass, enums
│   │   ├── orderbook.py        ← LimitOrderBook (heap-based)
│   │   └── trade.py            ← Trade dataclass
│   └── agents/
│       ├── base.py             ← Agent base class
│       ├── noise_trader.py
│       ├── market_maker.py     ← Avellaneda-Stoikov inventory skew
│       ├── momentum_bot.py     ← EMA crossover
│       ├── mean_revert_bot.py  ← Z-score reversion
│       ├── fundamental_trader.py
│       ├── options_bot.py      ← BSM quoting
│       ├── user_strategy.py    ← Base class for user code
│       └── async_strategy.py  ← Event-driven async base
├── my_strategy.py              ← YOUR STRATEGY GOES HERE
├── run_simulation.py           ← Wire everything together
└── analysis/
    ├── pnl_report.py
    └── price_visualiser.py
```

---

## Summary

```
Order Book     → Heap-based price-time priority matching; handles LIMIT, MARKET, IOC, FOK, STOP
Exchange       → Routes orders, fires fill callbacks, exposes market data
Bots           → Noise (random), MarketMaker (spread), Momentum (EMA), MeanRevert (z-score),
                 Fundamental (fair value gravity), OptionsBot (BSM quoting)
User API       → Inherit UserStrategy, implement on_tick — 5 lines minimum to go live
Low Latency    → _latency_ns on every order, async event-driven interface, C++ upgrade path
Derivatives    → Options priced via BSM in real time; instrument naming encodes type/strike
Evaluation     → Realised P&L, win rate, open position tracked per agent post-simulation
```

The bots collectively produce realistic microstructure: a tight spread from market makers, price trends from momentum bots, mean reversion pressure, and a fair-value anchor. Your strategy competes in that ecosystem — if it profits against them, it has a shot in the real market.
