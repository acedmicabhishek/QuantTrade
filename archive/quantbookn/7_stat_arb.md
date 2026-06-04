# Statistical Arbitrage & Mean Reversion: Implementation

This chapter covers the implementation of mean reversion and momentum strategies. The focus is on production-ready code — parameter estimation, signal generation, and position sizing — not mathematical derivation (see Ch 2 for the underlying stochastic calculus).

---

## 1. Mean Reversion and Cointegration

Mean reversion strategies assume a spread between assets returns to a long-term average. The QD's job: detect cointegration, estimate OU parameters, generate entry/exit signals, and size positions correctly.

### 1.1 The Ornstein-Uhlenbeck (OU) Process

The spread $X_t$ between two cointegrated assets follows:
$$dX_t = \theta (\mu - X_t) dt + \sigma dW_t$$

Parameters:
- $\mu$: long-term mean (the "fair value" of the spread)
- $\theta$: mean reversion speed (higher = faster reversion)
- $\sigma$: spread volatility

**Half-life of mean reversion** (time for spread to revert halfway to mean):
$$t_{1/2} = \frac{\ln 2}{\theta}$$

A half-life under 5 minutes is suitable for intraday strategies; under 1 day for daily strategies.

### 1.2 Parameter Estimation (AR(1) via OLS)

Discretize the OU SDE and run OLS regression on historical spread data:

```python
import numpy as np
from scipy.stats import linregress

def estimate_ou_params(spread: np.ndarray, dt: float = 1.0) -> dict:
    """
    Estimate OU parameters from spread time series.
    dt: time step in same units as desired half-life output.
    """
    x = spread[:-1]
    y = spread[1:]
    
    slope, intercept, r_value, p_value, stderr = linregress(x, y)
    
    theta = (1 - slope) / dt
    mu    = intercept / (theta * dt)
    sigma = np.std(y - (slope * x + intercept)) / np.sqrt(dt)
    
    half_life = np.log(2) / theta
    
    return {
        "theta": theta,
        "mu": mu,
        "sigma": sigma,
        "half_life": half_life,
        "r_squared": r_value**2,
    }
```

### 1.3 Cointegration Testing: Engle-Granger

Two price series $P^A$, $P^B$ are cointegrated if their residual $X_t = P^A_t - \beta P^B_t$ is stationary (I(0)):

```python
from statsmodels.regression.linear_model import OLS
from statsmodels.tsa.stattools import adfuller
import statsmodels.api as sm

def test_cointegration_eg(price_a: np.ndarray, price_b: np.ndarray,
                           significance: float = 0.05) -> dict:
    """Engle-Granger two-step cointegration test."""
    # Step 1: OLS regression
    X = sm.add_constant(price_b)
    result = OLS(price_a, X).fit()
    beta   = result.params[1]
    spread = price_a - beta * price_b
    
    # Step 2: ADF test on spread
    adf_stat, p_value, _, _, crit_values, _ = adfuller(spread, autolag='AIC')
    
    return {
        "beta": beta,
        "spread": spread,
        "adf_stat": adf_stat,
        "p_value": p_value,
        "is_cointegrated": p_value < significance,
    }
```

### 1.4 Johansen Test for N-Asset Portfolios

For portfolios with $N > 2$ assets, use the Johansen test to find multiple cointegrating vectors simultaneously:

```python
from statsmodels.tsa.vector_ar.vecm import coint_johansen

def johansen_cointegration(prices: np.ndarray, det_order: int = 0,
                            k_ar_diff: int = 1) -> dict:
    """
    prices: (T, N) array of N price series.
    Returns cointegrating vectors (each column is a portfolio weight vector).
    """
    result = coint_johansen(prices, det_order, k_ar_diff)
    
    # Trace test: number of cointegrating relationships at 5% significance
    n_coint = np.sum(result.lr1 > result.cvt[:, 1])
    
    # Eigenvectors = cointegrating vectors (columns)
    vectors = result.evec[:, :n_coint]
    
    return {
        "n_cointegrating_vectors": n_coint,
        "vectors": vectors,           # shape (N, n_coint)
        "eigenvalues": result.eig[:n_coint],
        "trace_stat": result.lr1,
        "crit_values_95": result.cvt[:, 1],
    }
```

### 1.5 Signal Generation: Z-Score Entry/Exit

Once the spread is identified and OU parameters estimated, generate signals from the spread's z-score:

```python
class MeanReversionSignal:
    def __init__(self, entry_z: float = 2.0, exit_z: float = 0.5,
                 stop_z: float = 4.0):
        self.entry_z = entry_z
        self.exit_z  = exit_z
        self.stop_z  = stop_z
        self.position = 0  # -1, 0, 1

    def update(self, spread: float, mu: float, sigma_spread: float) -> int:
        """Returns: +1 (long spread), -1 (short spread), 0 (flat)."""
        z = (spread - mu) / sigma_spread

        if self.position == 0:
            if z < -self.entry_z:
                self.position = 1   # spread below mean → long
            elif z > self.entry_z:
                self.position = -1  # spread above mean → short

        elif self.position == 1:
            if z > -self.exit_z or z < -self.stop_z:
                self.position = 0   # exit or stop-loss

        elif self.position == -1:
            if z < self.exit_z or z > self.stop_z:
                self.position = 0

        return self.position
```

**Parameter tuning:** entry z-score trades off frequency vs. edge quality. Higher entry threshold = fewer but higher-quality trades. Use walk-forward optimization (see Ch 6) to avoid overfitting.

### 1.6 Spread Decay: Kalman Filter for Online Parameter Tracking

Static OU parameters drift as market regimes change. Track $\beta$ (hedge ratio) online with a Kalman filter:

```python
class KalmanSpreadTracker:
    """Tracks time-varying hedge ratio beta using Kalman filter."""
    def __init__(self, delta: float = 1e-5, Vt: float = 1e-3):
        self.delta = delta        # state transition noise
        self.Vt    = Vt           # observation noise
        self.beta  = np.zeros(2)  # [intercept, slope]
        self.P     = np.eye(2)    # state covariance
        self.R     = 0.0

    def update(self, price_a: float, price_b: float) -> float:
        F = np.array([[1.0, price_b]])  # observation matrix

        # Predict
        Q = self.delta / (1 - self.delta) * np.eye(2)
        self.P = self.P + Q

        # Update
        innovation = price_a - F @ self.beta
        S = F @ self.P @ F.T + self.Vt
        K = self.P @ F.T / S                # Kalman gain
        self.beta = self.beta + K.flatten() * innovation
        self.P    = (np.eye(2) - np.outer(K, F)) @ self.P

        spread = price_a - self.beta[1] * price_b - self.beta[0]
        return spread
```

---

## 2. Momentum and Trend Following

### 2.1 Exponential Moving Average and MACD

```python
class EMACalculator:
    def __init__(self, fast_period: int = 12, slow_period: int = 26):
        self.alpha_fast = 2.0 / (fast_period + 1)
        self.alpha_slow = 2.0 / (slow_period + 1)
        self.ema_fast: float | None = None
        self.ema_slow: float | None = None

    def update(self, price: float) -> float | None:
        if self.ema_fast is None:
            self.ema_fast = price
            self.ema_slow = price
            return None

        self.ema_fast = self.alpha_fast * price + (1 - self.alpha_fast) * self.ema_fast
        self.ema_slow = self.alpha_slow * price + (1 - self.alpha_slow) * self.ema_slow

        return self.ema_fast - self.ema_slow  # MACD
```

### 2.2 Kalman Filter Trend Tracker

Treats true trend as latent state variable; more principled than ad-hoc MA:

```python
class KalmanTrendFilter:
    """
    State equation:       x_t = x_{t-1} + w_t,   w_t ~ N(0, Q)
    Measurement equation: y_t = x_t    + v_t,   v_t ~ N(0, R)
    """
    def __init__(self, Q: float = 1e-5, R: float = 0.01):
        self.Q = Q   # process noise (trend evolution uncertainty)
        self.R = R   # measurement noise (price observation uncertainty)
        self.x: float | None = None  # state estimate
        self.P: float = 1.0          # state covariance

    def update(self, price: float) -> float:
        if self.x is None:
            self.x = price
            return price

        # Predict
        P_pred = self.P + self.Q

        # Update
        K      = P_pred / (P_pred + self.R)   # Kalman gain
        self.x = self.x + K * (price - self.x)
        self.P = (1 - K) * P_pred

        return self.x
```

---

## 3. Backtesting Considerations for Stat Arb

### 3.1 Look-Ahead Bias

Most dangerous error in stat arb backtests:
- Never use future prices to compute the hedge ratio $\beta$ for a past period.
- Use only data available at the signal time — trailing windows only.
- Apply cointegration tests using expanding or rolling windows.

### 3.2 Regime Stability Check

Before deploying, check OU parameter stability over rolling windows:

```python
def rolling_ou_stability(spread: np.ndarray, window: int = 252) -> pd.DataFrame:
    """Check OU parameter stability — unstable params = fragile strategy."""
    records = []
    for i in range(window, len(spread)):
        params = estimate_ou_params(spread[i - window:i])
        records.append({"t": i, **params})
    return pd.DataFrame(records)
```

If `theta` or `half_life` varies more than 50% across rolling windows, the spread is not stable enough for live deployment.

### 3.3 Transaction Costs and Fill Modeling

Stat arb requires realistic cost modeling:
- **Bid-ask spread:** pay half-spread on each leg on entry and exit.
- **Market impact:** for larger sizes, use square-root law: $\Delta P \approx \sigma \sqrt{Q/ADV}$.
- **Borrow cost:** short legs require securities lending; include daily borrow rate.

```python
def net_pnl(gross_pnl: float, n_trades: int, avg_spread_bps: float,
             avg_notional: float) -> float:
    cost_per_trade = avg_notional * avg_spread_bps * 1e-4
    return gross_pnl - n_trades * cost_per_trade * 2  # entry + exit
```

---

Next Chapter: [Market Making Implementation](8_market_making.md)
