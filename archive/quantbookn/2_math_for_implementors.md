# Derivatives Math and Stochastic Calculus

Quantitative finance relies heavily on modeling continuous-time uncertainty. This chapter provides a rigorous overview of stochastic calculus, derivative pricing theory, and numerical simulation techniques tailored for engineers who are comfortable with calculus, linear algebra, and differential equations.

---

## 1. Stochastic Calculus Foundations

In deterministic physics, we model motion using standard ordinary differential equations (ODEs). In financial markets, asset prices exhibit unpredictable, random behavior. To model this, we introduce stochastic calculus.

### 1.1 The Wiener Process (Brownian Motion)

A Wiener process (or standard Brownian motion) $W_t$ is a continuous-time stochastic process that serves as the fundamental building block of continuous randomness. A stochastic process $W = \{W_t : t \ge 0\}$ is a Wiener process if:
1. $W_0 = 0$ almost surely.
2. It has **independent increments**: for any $0 \le t_1 < t_2 < \dots < t_n$, the increments $W_{t_2} - W_{t_1}, W_{t_3} - W_{t_2}, \dots, W_{t_n} - W_{t_{n-1}}$ are mutually independent.
3. It has **stationary Gaussian increments**: for any $0 \le s < t$, the increment $W_t - W_s$ is normally distributed with mean $0$ and variance $t-s$:
   $$W_t - W_s \sim \mathcal{N}(0, t-s)$$
4. The paths of $W_t$ are continuous with probability 1.

#### Key Engineering Intuition: Nowhere Differentiable Paths
Although $W_t$ is continuous everywhere, it is **nowhere differentiable** in the classical sense. The limit:
$$\lim_{\Delta t \to 0} \frac{W_{t+\Delta t} - W_t}{\Delta t} \sim \lim_{\Delta t \to 0} \frac{\mathcal{N}(0, \Delta t)}{\Delta t} \sim \lim_{\Delta t \to 0} \mathcal{N}\left(0, \frac{1}{\Delta t}\right)$$
blows up as $\Delta t \to 0$. Therefore, classical calculus ($df = f'(x)dx$) fails, and we must define a new integration theory (Itô calculus) where $(dW_t)^2 = dt$.

---

### 1.2 Itô's Lemma

Let $X_t$ be an Itô drift-diffusion process satisfying the Stochastic Differential Equation (SDE):
$$dX_t = \mu(t, X_t) dt + \sigma(t, X_t) dW_t$$
where $\mu(t, X_t)$ is the drift coefficient and $\sigma(t, X_t)$ is the diffusion coefficient (volatility).

If $f(t, x)$ is a twice-differentiable function of $x$ and once-differentiable of $t$, then the differential of the stochastic process $f(t, X_t)$ is given by **Itô's Lemma**:
$$df(t, X_t) = \left( \frac{\partial f}{\partial t} + \mu(t, X_t) \frac{\partial f}{\partial x} + \frac{1}{2} \sigma(t, X_t)^2 \frac{\partial^2 f}{\partial x^2} \right) dt + \sigma(t, X_t) \frac{\partial f}{\partial x} dW_t$$

#### Derivation Sketch (Taylor Series Expansion)
Expand $df$ using a second-order Taylor series:
$$df = \frac{\partial f}{\partial t} dt + \frac{\partial f}{\partial x} dX_t + \frac{1}{2} \frac{\partial^2 f}{\partial x^2} (dX_t)^2 + \frac{1}{2} \frac{\partial^2 f}{\partial t^2} (dt)^2 + \frac{\partial^2 f}{\partial t \partial x} dt \, dX_t$$
Substitute $dX_t = \mu dt + \sigma dW_t$:
$$(dX_t)^2 = (\mu dt + \sigma dW_t)^2 = \mu^2 (dt)^2 + 2\mu\sigma dt \, dW_t + \sigma^2 (dW_t)^2$$
Under Itô calculus rules, we drop terms of order higher than $dt$. The multiplication rules are:
- $dt \cdot dt = 0$
- $dt \cdot dW_t = 0$
- $dW_t \cdot dW_t = dt$ (due to the quadratic variation of Brownian motion accumulating deterministically over time).

Substituting these rules back into the Taylor expansion yields Itô's Lemma. The term $\frac{1}{2}\sigma^2 \frac{\partial^2 f}{\partial x^2} dt$ is called the **Itô correction term**, which has no analogue in classical calculus.

---

### 1.3 Geometric Brownian Motion (GBM)

The standard model for stock price dynamics is Geometric Brownian Motion (GBM):
$$dS_t = \mu S_t dt + \sigma S_t dW_t$$
where $S_t$ is the stock price, $\mu$ is the constant expected rate of return (drift), and $\sigma$ is the constant volatility.

#### Solving the GBM SDE
Let $f(S_t) = \ln(S_t)$. Let us apply Itô's Lemma:
- $\frac{\partial f}{\partial t} = 0$
- $\frac{\partial f}{\partial S} = \frac{1}{S_t}$
- $\frac{\partial^2 f}{\partial S^2} = -\frac{1}{S_t^2}$

Applying the lemma:
$$d(\ln S_t) = \left( 0 + (\mu S_t)\frac{1}{S_t} + \frac{1}{2}(\sigma S_t)^2 \left(-\frac{1}{S_t^2}\right) \right) dt + (\sigma S_t) \frac{1}{S_t} dW_t$$
$$d(\ln S_t) = \left( \mu - \frac{1}{2}\sigma^2 \right) dt + \sigma dW_t$$

Integrating both sides from $0$ to $t$:
$$\ln S_t - \ln S_0 = \left( \mu - \frac{1}{2}\sigma^2 \right) t + \sigma W_t$$
Exponentiating both sides gives the analytical solution for $S_t$:
$$S_t = S_0 \exp\left( \left(\mu - \frac{1}{2}\sigma^2\right)t + \sigma W_t \right)$$

This shows that $S_t$ is **log-normally distributed**: $\ln(S_t/S_0) \sim \mathcal{N}\left( (\mu - 0.5\sigma^2)t, \sigma^2 t \right)$.

---

## 2. The Black-Scholes-Merton Framework

The Black-Scholes-Merton model prices European options by constructing a risk-free replicating portfolio containing the underlying asset and a derivative contract.

### 2.1 The Black-Scholes PDE Derivation

Assume a market with:
1. A risky asset $S_t$ following GBM: $dS_t = \mu S_t dt + \sigma S_t dW_t$
2. A risk-free bond $B_t$ growing at rate $r$: $dB_t = r B_t dt$

Let $V(S, t)$ be the price of a derivative on $S$. By Itô's Lemma:
$$dV = \left( \frac{\partial V}{\partial t} + \mu S \frac{\partial V}{\partial S} + \frac{1}{2}\sigma^2 S^2 \frac{\partial^2 V}{\partial S^2} \right) dt + \sigma S \frac{\partial V}{\partial S} dW_t$$

Construct a portfolio $\Pi$ consisting of short 1 unit of the derivative and long $\Delta$ units of the stock:
$$\Pi = -V + \Delta S$$
The change in portfolio value over an infinitesimal step $dt$ is:
$$d\Pi = -dV + \Delta dS$$
Substitute the expressions for $dV$ and $dS$:
$$d\Pi = -\left[ \left( \frac{\partial V}{\partial t} + \mu S \frac{\partial V}{\partial S} + \frac{1}{2}\sigma^2 S^2 \frac{\partial^2 V}{\partial S^2} \right) dt + \sigma S \frac{\partial V}{\partial S} dW_t \right] + \Delta (\mu S dt + \sigma S dW_t)$$
Combine the deterministic ($dt$) and stochastic ($dW_t$) terms:
$$d\Pi = \left( -\frac{\partial V}{\partial t} - \mu S \frac{\partial V}{\partial S} - \frac{1}{2}\sigma^2 S^2 \frac{\partial^2 V}{\partial S^2} + \Delta \mu S \right) dt + \sigma S \left( \Delta - \frac{\partial V}{\partial S} \right) dW_t$$

To make the portfolio **risk-free**, we choose a hedge ratio $\Delta$ that eliminates the stochastic $dW_t$ term:
$$\Delta = \frac{\partial V}{\partial S}$$
This is known as **delta-hedging**. The portfolio change becomes completely deterministic:
$$d\Pi = \left( -\frac{\partial V}{\partial t} - \frac{1}{2}\sigma^2 S^2 \frac{\partial^2 V}{\partial S^2} \right) dt$$

Under the assumption of **no arbitrage**, any risk-free portfolio must earn the risk-free rate of return $r$. Therefore:
$$d\Pi = r \Pi dt \implies d\Pi = r \left( -V + \frac{\partial V}{\partial S} S \right) dt$$

Equating the two expressions for $d\Pi$:
$$\left( -\frac{\partial V}{\partial t} - \frac{1}{2}\sigma^2 S^2 \frac{\partial^2 V}{\partial S^2} \right) dt = r \left( -V + S\frac{\partial V}{\partial S} \right) dt$$

Rearranging terms yields the famous **Black-Scholes Partial Differential Equation**:
$$\frac{\partial V}{\partial t} + r S \frac{\partial V}{\partial S} + \frac{1}{2}\sigma^2 S^2 \frac{\partial^2 V}{\partial S^2} - r V = 0$$

> [!IMPORTANT]
> The drift parameter $\mu$ does not appear in the Black-Scholes PDE. This implies that the pricing of derivatives is independent of investors' risk preferences (drift rate $\mu$), giving rise to **risk-neutral valuation**.

---

### 2.2 Analytical Option Pricing Formulas

For a European Call Option with strike price $K$ and expiration time $T$, the boundary condition at $t = T$ is $V(S, T) = \max(S_T - K, 0)$. Solving the PDE with this boundary condition yields:
$$C(S, t) = S_t N(d_1) - K e^{-r(T-t)} N(d_2)$$

For a European Put Option (payoff $\max(K - S_T, 0)$):
$$P(S, t) = K e^{-r(T-t)} N(-d_2) - S_t N(-d_1)$$

Where:
$$d_1 = \frac{\ln(S_t / K) + \left(r + \frac{1}{2}\sigma^2\right)(T-t)}{\sigma \sqrt{T-t}}$$
$$d_2 = d_1 - \sigma \sqrt{T-t}$$
And $N(x)$ is the cumulative standard normal distribution function:
$$N(x) = \frac{1}{\sqrt{2\pi}} \int_{-\infty}^{x} e^{-\frac{z^2}{2}} dz$$

---

## 3. The Option Greeks

The Greeks measure the sensitivity of the option's price to various market parameters. They are essential tools for quantitative risk management.

| Greek | Mathematical Definition | Formula (European Call) | Interpretation |
| :--- | :--- | :--- | :--- |
| **Delta ($\Delta$)** | $\frac{\partial V}{\partial S}$ | $N(d_1)$ | Change in option price per unit change in stock price. Used for delta-hedging. |
| **Gamma ($\Gamma$)** | $\frac{\partial^2 V}{\partial S^2}$ | $\frac{N'(d_1)}{S \sigma \sqrt{T-t}}$ | Sensitivity of Delta to stock price. Measures hedging stability. |
| **Vega ($\nu$)** | $\frac{\partial V}{\partial \sigma}$ | $S \sqrt{T-t} N'(d_1)$ | Change in option price per 1% change in volatility. |
| **Theta ($\Theta$)** | $\frac{\partial V}{\partial t}$ | $-\frac{S N'(d_1) \sigma}{2\sqrt{T-t}} - r K e^{-r(T-t)} N(d_2)$ | Time decay of the option. Usually negative as options lose value over time. |
| **Rho ($\rho$)** | $\frac{\partial V}{\partial r}$ | $K (T-t) e^{-r(T-t)} N(d_2)$ | Sensitivity to interest rates. |

Here $N'(x) = \frac{1}{\sqrt{2\pi}} e^{-\frac{x^2}{2}}$ is the standard normal probability density function (PDF).

---

## 4. Volatility Surfaces

In the Black-Scholes model, volatility $\sigma$ is assumed to be constant. In real markets, if you extract $\sigma$ from observed option prices using the BSM formula, you will find that $\sigma$ varies by **Strike ($K$)** and **Time to Maturity ($T$)**.

```
Implied Volatility (σ)
    ▲
    │         \             /   <-- Volatility Smile (Common in Equities/FX)
    │          \           /
    │           \_________/
    │
    └─────────────────────────────► Strike (K)
```

- **Volatility Smile/Skew:** Options that are deep in-the-money or out-of-the-money often trade at higher implied volatilities than at-the-money options. A skew represents asymmetric downside risk pricing (skewed left in equities due to fear of crashes).
- **Term Structure:** Implied volatility changes as maturity increases, reflecting the market's view of how uncertainty will unfold.
- **Advanced Models:** To address this discrepancy, modern desks use models like **Heston (stochastic volatility)** or **SABR** to calibrate to the market volatility surface.

---

## 5. Monte Carlo Simulation

When a derivative is path-dependent (e.g., Asian options where payoff depends on the average price over time, or Barrier options), analytical BSM solutions do not exist. We use Monte Carlo methods instead.

### 5.1 Euler-Maruyama Discretization

To simulate paths of GBM $dS_t = r S_t dt + \sigma S_t dW_t$ under the risk-neutral measure, we discretize time into $N$ steps of size $\Delta t = T / N$.
Letting $x_t = \ln(S_t)$, we know that $dx_t = (r - 0.5\sigma^2)dt + \sigma dW_t$. The exact update formula is:
$$S_{t+\Delta t} = S_t \exp\left( \left(r - \frac{1}{2}\sigma^2\right)\Delta t + \sigma \sqrt{\Delta t} Z \right)$$
where $Z \sim \mathcal{N}(0, 1)$ is a standard normal random variable.

### 5.2 Python Monte Carlo Implementation

Below is a vector-based simulation function to price an Asian average-strike Call option:

```python
import numpy as np

def price_asian_call(S0, K, r, sigma, T, steps, paths):
    dt = T / steps
    # Pre-generate random normals
    Z = np.random.standard_normal((steps, paths))
    
    # Accumulate log paths
    log_drifts = (r - 0.5 * sigma**2) * dt + sigma * np.sqrt(dt) * Z
    log_paths = np.zeros((steps + 1, paths))
    log_paths[0, :] = np.log(S0)
    log_paths[1:, :] = np.log(S0) + np.cumsum(log_drifts, axis=0)
    
    # Exponentiate to get stock price paths
    S = np.exp(log_paths)
    
    # Calculate Asian path-dependent payoff: Mean along time axis
    average_prices = np.mean(S[1:, :], axis=0)
    payoffs = np.maximum(average_prices - K, 0)
    
    # Discount back to present value
    price = np.exp(-r * T) * np.mean(payoffs)
    standard_error = np.exp(-r * T) * np.std(payoffs) / np.sqrt(paths)
    
    return price, standard_error
```

### 5.3 Variance Reduction Techniques

Monte Carlo error decays slowly at a rate of $\mathcal{O}(1/\sqrt{M})$ where $M$ is the number of simulation paths. To speed up convergence, we use variance reduction:

1. **Antithetic Variates:** For every path generated using random draws $\{Z_i\}$, we simultaneously generate a path using $\{-Z_i\}$. Since $\text{Cov}(S(Z), S(-Z)) < 0$, the average of the two paths has significantly lower variance.
2. **Control Variates:** If we want to price a complex option $Y$, we select a similar option $X$ that has a known analytic price $\mathbb{E}[X]$. The estimator is modified to:
   $$Y^* = Y - \beta (X - \mathbb{E}[X])$$
   where choosing $\beta = \frac{\text{Cov}(Y, X)}{\text{Var}(X)}$ minimizes the variance of the estimator.

---

Next Chapter: [Market Microstructure & Order Book Mechanics](file:///Users/ace/Documents/collage/HFT/quantbookn/3_market_microstructure.md)