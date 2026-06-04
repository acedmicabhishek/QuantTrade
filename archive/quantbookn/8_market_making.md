# Market Making: Inventory Control and Optimal Quote Placement

Market makers post passive limit orders on both sides of the book to capture the bid-ask spread. The risk is **adverse selection** (being picked off by informed flow) and **inventory accumulation** (accumulating a directional position as the market moves against you). This chapter covers the Avellaneda-Stoikov model and its production implementation.

---

## 1. The Avellaneda-Stoikov Model

### 1.1 Model Assumptions

- Mid-price follows pure diffusion: $dS_t = \sigma dW_t$ (no drift — market maker is indifferent to direction)
- Order arrivals follow Poisson process with intensity that decays with distance from mid:
  $$\lambda(\delta) = A e^{-\kappa \delta}$$
  where $\kappa$ is order book depth and $A$ is order flow activity.
- Market maker has constant absolute risk aversion (CARA) utility: $U = -e^{-\gamma W}$

### 1.2 Reservation Price

The market maker holds inventory $q$. The **reservation price** is the mid-price adjusted for inventory risk:
$$r(s, q, t) = s - q \gamma \sigma^2 (T - t)$$

- $q > 0$ (long): reservation price below mid → skew quotes down to sell inventory.
- $q < 0$ (short): reservation price above mid → skew quotes up to buy inventory.
- At $T$ (session end): reservation price = mid, as time-to-risk-horizon shrinks to zero.

### 1.3 Optimal Half-Spreads

Optimal distance of bid/ask quotes from the reservation price:
$$\delta^* = \frac{1}{\gamma} \ln\left(1 + \frac{\gamma}{\kappa}\right)$$

Optimal bid and ask prices:
$$p^{\text{ask}} = r + \delta^*$$
$$p^{\text{bid}} = r - \delta^*$$

Total spread = $2\delta^*$. This is symmetric around the reservation price, not around mid.

**Interpretation:** As $\kappa \to \infty$ (very deep book), $\delta^* \to 1/\kappa$ (spread shrinks). As $\gamma \to 0$ (risk-neutral), $\delta^* \to 1/\kappa$.

---

## 2. Implementation

### 2.1 Core Quote Calculator

```python
import numpy as np
from dataclasses import dataclass

@dataclass
class MMParams:
    sigma: float     # mid-price volatility (per second)
    gamma: float     # risk aversion parameter
    kappa: float     # order book depth parameter
    A: float         # order arrival intensity at mid
    T: float         # session length in seconds (e.g. 23400 for 6.5h)

class AvellanedaStoikovMM:
    def __init__(self, params: MMParams):
        self.p = params
        self._delta_star = (1.0 / params.gamma) * np.log(1.0 + params.gamma / params.kappa)

    def reservation_price(self, mid: float, inventory: int,
                           time_remaining: float) -> float:
        return mid - inventory * self.p.gamma * self.p.sigma**2 * time_remaining

    def optimal_quotes(self, mid: float, inventory: int,
                        time_remaining: float) -> tuple[float, float]:
        """Returns (bid_price, ask_price)."""
        r = self.reservation_price(mid, inventory, time_remaining)
        return r - self._delta_star, r + self._delta_star

    def expected_fill_rate(self, delta: float) -> float:
        """Expected fills per second at a given distance from mid."""
        return self.p.A * np.exp(-self.p.kappa * delta)
```

### 2.2 Parameter Calibration

Parameters must be calibrated from historical tick data:

```python
def calibrate_mm_params(trades: np.ndarray, timestamps: np.ndarray,
                          book_snapshots: list) -> MMParams:
    """
    trades: array of trade prices
    timestamps: trade timestamps in seconds
    book_snapshots: list of (bid, ask, bid_size, ask_size) at each tick
    """
    # Volatility: realized vol from trade prices
    returns = np.diff(np.log(trades))
    dt = np.mean(np.diff(timestamps))
    sigma = np.std(returns) / np.sqrt(dt)

    # Order arrival intensity: count trades per unit time vs distance from mid
    mids = [(s[0] + s[1]) / 2 for s in book_snapshots]
    spreads = [(s[1] - s[0]) / 2 for s in book_snapshots]  # half-spread

    avg_half_spread = np.mean(spreads)
    avg_trade_rate  = len(trades) / (timestamps[-1] - timestamps[0])

    # Fit A and kappa: lambda = A * exp(-kappa * delta)
    # At delta = 0: lambda(0) = A (rough estimate)
    A = avg_trade_rate / 2   # per side
    kappa = 1.0 / avg_half_spread   # inverse of typical half-spread

    return MMParams(sigma=sigma, gamma=0.1, kappa=kappa, A=A, T=23400.0)
```

**Note:** `gamma` (risk aversion) is not directly observable — tune it by backtest, targeting desired max inventory bounds.

### 2.3 Inventory Management

Unconstrained Avellaneda-Stoikov can allow unbounded inventory in trending markets. Add hard limits:

```python
class InventoryConstrainedMM(AvellanedaStoikovMM):
    def __init__(self, params: MMParams, max_inventory: int,
                  inventory_skew_factor: float = 0.5):
        super().__init__(params)
        self.max_inventory = max_inventory
        self.skew_factor   = inventory_skew_factor

    def optimal_quotes(self, mid: float, inventory: int,
                        time_remaining: float) -> tuple[float, float] | None:
        # Hard block: stop quoting one side at inventory limits
        if inventory >= self.max_inventory:
            # Only quote ask (want to sell to reduce inventory)
            r = self.reservation_price(mid, inventory, time_remaining)
            return None, r + self._delta_star

        if inventory <= -self.max_inventory:
            r = self.reservation_price(mid, inventory, time_remaining)
            return r - self._delta_star, None

        # Soft skew: widen spread on the side that increases inventory
        bid, ask = super().optimal_quotes(mid, inventory, time_remaining)

        # Penalty proportional to how close we are to limits
        fill_ratio = abs(inventory) / self.max_inventory
        extra_skew  = self.skew_factor * fill_ratio * self._delta_star

        if inventory > 0:
            bid -= extra_skew   # push bid lower → less likely to fill on buy side
        else:
            ask += extra_skew   # push ask higher → less likely to fill on sell side

        return bid, ask
```

---

## 3. Order Management in Production

### 3.1 Quote Lifecycle

Each market making cycle:
1. **Receive tick**: new mid-price from feed handler.
2. **Compute quotes**: `optimal_quotes(mid, inventory, time_remaining)`.
3. **Check existing orders**: compare new quotes to live resting orders.
4. **Cancel + Replace**: if new quote differs by more than `reprice_threshold` from live order.
5. **Send new orders**: via OUCH (see Ch 11).

```python
REPRICE_THRESHOLD = 0.0001  # 1 basis point

class MMOrderManager:
    def __init__(self, gateway, quote_calc, lot_size: int = 100):
        self.gateway    = gateway
        self.calc       = quote_calc
        self.lot_size   = lot_size
        self.live_bid   = None  # (order_id, price)
        self.live_ask   = None
        self.inventory  = 0

    def on_tick(self, mid: float, time_remaining: float):
        new_bid, new_ask = self.calc.optimal_quotes(mid, self.inventory, time_remaining)

        # Cancel/replace bid if moved significantly
        if self.live_bid is not None:
            if new_bid is None or abs(new_bid - self.live_bid[1]) > REPRICE_THRESHOLD:
                self.gateway.cancel(self.live_bid[0])
                self.live_bid = None

        if self.live_ask is not None:
            if new_ask is None or abs(new_ask - self.live_ask[1]) > REPRICE_THRESHOLD:
                self.gateway.cancel(self.live_ask[0])
                self.live_ask = None

        # Place new orders
        if new_bid is not None and self.live_bid is None:
            oid = self.gateway.limit_order('B', self.lot_size, new_bid)
            self.live_bid = (oid, new_bid)

        if new_ask is not None and self.live_ask is None:
            oid = self.gateway.limit_order('S', self.lot_size, new_ask)
            self.live_ask = (oid, new_ask)

    def on_fill(self, order_id: str, side: str, qty: int):
        if side == 'B':
            self.inventory += qty
            self.live_bid   = None
        else:
            self.inventory -= qty
            self.live_ask   = None
```

### 3.2 Cancel Rate Management

Exchanges monitor cancel-to-fill ratios. Excessive cancels → warnings, fees, or ban. Keep reprice threshold above tick size × 2 to avoid unnecessary cancels.

---

## 4. Risk Controls Specific to Market Making

### 4.1 Adverse Selection Detection (VPIN)

**Volume-synchronized Probability of Informed Trading (VPIN)** flags when order flow becomes directional (informed traders picking off your quotes):

$$\text{VPIN} = \frac{\sum_{i=1}^{n} |V_i^B - V_i^S|}{\sum_{i=1}^{n} V_i}$$

where $V_i^B$ and $V_i^S$ are buy and sell volumes in bucket $i$ (equal-volume time buckets, not equal-time).

```python
def compute_vpin(trades: list[tuple[str, float]], bucket_vol: float) -> float:
    """trades: list of (side, volume) tuples. Returns VPIN over last n buckets."""
    buckets = []
    cur_buy = cur_sell = cur_total = 0.0

    for side, vol in trades:
        remaining = vol
        while remaining > 0:
            space = bucket_vol - cur_total
            fill  = min(remaining, space)
            if side == 'B':
                cur_buy += fill
            else:
                cur_sell += fill
            cur_total += fill
            remaining -= fill
            if cur_total >= bucket_vol:
                buckets.append(abs(cur_buy - cur_sell))
                cur_buy = cur_sell = cur_total = 0.0

    if not buckets:
        return 0.0
    return sum(buckets) / (len(buckets) * bucket_vol)
```

When VPIN exceeds threshold (~0.7), widen spreads or pause quoting.

### 4.2 Stale Quote Protection

Quoted prices become stale when feed handler is delayed. Always track time since last mid update:

```python
MAX_QUOTE_AGE_NS = 5_000_000  # 5ms

def is_quote_stale(last_mid_ts_ns: int) -> bool:
    now = time.time_ns()
    return (now - last_mid_ts_ns) > MAX_QUOTE_AGE_NS
```

If stale: cancel all live orders immediately, do not place new ones until feed resumes.

---

Next Chapter: [Execution Algorithms](9_execution_algos.md)
