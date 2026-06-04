# Risk Management and Portfolio Theory

Risk management in quantitative trading serves two roles: it prevents catastrophic loss (protecting survival) and optimizes capital allocation to maximize long-term geometric returns. This chapter details risk metrics, mathematical modeling of extreme loss, position-sizing theory, and operational system safety.

---

## 1. Value at Risk (VaR) and Expected Shortfall (CVaR)

Let $L$ be the loss of a portfolio over a specific time horizon $\Delta t$. We represent $L$ as a random variable.

### 1.1 Value at Risk (VaR)

The Value at Risk at confidence level $c \in (0, 1)$ (typically $95\%$ or $99\%$) is the smallest loss threshold that will not be exceeded with probability $c$:
$$\text{VaR}_c = \inf \{ l \in \mathbb{R} : P(L > l) \le 1 - c \}$$

```
Probability Density
      ▲
      │             c (e.g. 95% Confidence)
      │      ┌──────────────────────────────────┐   1 - c
      │      │                                  │  ┌──────┐
      │  ____▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│  ▒▒▒▒▒▒
      └──────┴──────────────────────────────────┴──┴──────► Loss
                                            VaR_c
```

#### Method 1: Parametric (Delta-Normal) VaR
Assume portfolio returns follow a normal distribution $\mathcal{N}(\mu_p, \sigma_p^2)$. If the portfolio value is $V_0$, the VaR is:
$$\text{VaR}_c = -(\mu_p + z_{1-c} \sigma_p) \cdot V_0$$
where $z_{1-c}$ is the $(1-c)$-percentile of the standard normal distribution (e.g., $z_{0.01} \approx -2.33$ for $99\%$ confidence).
- *Limitation:* Financial return distributions exhibit fat tails (leptokurtosis), meaning extreme losses occur far more frequently than predicted by a normal distribution.

#### Method 2: Historical Simulation
Does not assume a parametric distribution. Instead, we take the vector of historical returns, apply them to the current portfolio weights to simulate historical losses, sort them, and select the $(1-c)$ percentile.
- *Limitation:* It assumes the future will behave like the past and cannot model scenarios that have not occurred historically.

#### Method 3: Monte Carlo VaR
Simulate thousands of paths for the underlying risk factors using stochastic differential equations (e.g., GBM, jump-diffusion), compute the portfolio value at the end of the horizon for each path, and find the percentile loss.

---

### 1.2 Expected Shortfall (Conditional VaR / CVaR)

VaR is not a **coherent risk measure** because it violates the axiom of subadditivity (the VaR of a combined portfolio can theoretically be greater than the sum of individual VaRs under non-normal distributions). Furthermore, VaR is blind to the severity of losses in the tail beyond the threshold.

Expected Shortfall (CVaR) resolves this by measuring the expected loss given that the loss exceeds the VaR threshold:
$$\text{CVaR}_c = \mathbb{E}[L \mid L \ge \text{VaR}_c]$$

For continuous distributions, this can be written as:
$$\text{CVaR}_c = \frac{1}{1-c} \int_c^1 \text{VaR}_u \, du$$

CVaR is subadditive, coherent, and quantifies the expected impact of extreme "black swan" tail events.

---

## 2. Risk-Adjusted Performance Metrics

To compare strategies, we must normalize raw returns by the risk taken to generate them.

- **Sharpe Ratio ($SR$):** Measures excess return per unit of total risk (standard deviation):
  $$SR = \frac{\mathbb{E}[R_p - R_f]}{\sigma_p}$$
  where $R_p$ is portfolio return and $R_f$ is the risk-free rate.
- **Sortino Ratio:** Focuses only on downside risk, ignoring upside volatility:
  $$\text{Sortino} = \frac{\mathbb{E}[R_p - R_f]}{\sigma_{\text{down}}}$$
  where $\sigma_{\text{down}} = \sqrt{\mathbb{E}[(\min(R_p - R_f, 0))^2]}$.
- **Calmar Ratio:** Compares annual return to maximum drawdown:
  $$\text{Calmar} = \frac{\text{Annualized Return}}{\text{Maximum Drawdown}}$$
- **Information Ratio ($IR$):** Measures active return relative to a benchmark:
  $$IR = \frac{\mathbb{E}[R_p - R_b]}{\text{std}(R_p - R_b)}$$
  where $R_b$ is the benchmark return.

---

## 3. Position Sizing: The Kelly Criterion

The Kelly Criterion determines the optimal fraction of capital to allocate to a trade to maximize the long-term log growth rate of wealth.

### 3.1 Derivation (Binary Bet)
Let $W_0$ be initial wealth. You bet a fraction $f$ of wealth on a coin flip.
- Win with probability $p$: payoff is $f \cdot b$ (odds are $b$-to-1).
- Lose with probability $q = 1-p$: loss is $f$.

Your wealth after one bet is:
$$W_1 = W_0 (1 + b f) \quad \text{with probability } p$$
$$W_1 = W_0 (1 - f) \quad \text{with probability } q$$

After $N$ independent trials, wealth is:
$$W_N = W_0 (1 + b f)^{pN} (1 - f)^{qN}$$
We want to maximize the expected growth rate $g(f) = \mathbb{E}[\ln(W_N / W_0)] / N$:
$$g(f) = p \ln(1 + b f) + q \ln(1 - f)$$
To find the optimal allocation $f^*$, take the derivative with respect to $f$ and set to zero:
$$g'(f) = \frac{p b}{1 + b f} - \frac{q}{1 - f} = 0$$
$$p b (1 - f) = q (1 + b f) \implies p b - p b f = q + q b f$$
Substitute $q = 1-p$:
$$f^* = \frac{p b - (1-p)}{b} = \frac{p(b + 1) - 1}{b}$$

#### The Continuous Version
For an asset with expected return $\mu$ and volatility $\sigma$, the continuous Kelly allocation is:
$$f^* = \frac{\mu - r}{\sigma^2}$$

> [!WARNING]
> While Kelly maximizes asymptotic wealth, using full Kelly ($f^*$) leads to highly volatile paths and a $50\%$ chance of experiencing a $50\%$ drawdown. In practice, institutions use **fractional Kelly** (e.g., $0.5 \cdot f^*$ or $0.25 \cdot f^*$) to scale down exposure and smooth the growth curve.

---

## 4. Modern Portfolio Theory (MPT) and Risk Budgeting

### 4.1 Mean-Variance Optimization
Modern Portfolio Theory (Markowitz) formalizes diversification. For a portfolio of $N$ assets with weight vector $w$, expected asset returns $\mu$, and covariance matrix $\Sigma$:
- Portfolio expected return: $\mu_p = w^T \mu$
- Portfolio variance: $\sigma_p^2 = w^T \Sigma w$

To find the minimum variance portfolio for a target return $\mu_0$, we solve:
$$\min_w w^T \Sigma w \quad \text{subject to } w^T \mu = \mu_0, \ w^T \mathbf{1} = 1$$
This is a standard quadratic programming problem.

### 4.2 Risk Budgeting (Risk Parity)
Mean-variance optimization often yields highly concentrated portfolios if one asset has slightly higher historical return. **Risk Parity** allocates capital based on risk contributions.
The marginal risk contribution of asset $i$ to portfolio volatility $\sigma_p$ is:
$$\text{MRC}_i = \frac{\partial \sigma_p}{\partial w_i} = \frac{(\Sigma w)_i}{\sigma_p}$$
The absolute risk contribution of asset $i$ is:
$$\text{RC}_i = w_i \frac{(\Sigma w)_i}{\sigma_p}$$
A risk parity portfolio budgets weights such that risk contributions are equalized across all assets:
$$\text{RC}_i = \text{RC}_j \quad \forall i, j$$

---

## 5. System Safety and Operational Risk Controls

In high-frequency systems, mathematical risk management must be complemented by hard operational limits implemented directly inside the execution engine.

- **Pre-Trade Risk Checks:** Every outgoing order must pass validation checks before being serialized and sent to the exchange. If a check fails, the order is blocked immediately.
  - **Notional Limit Checks:** Rejects orders exceeding a maximum value.
  - **Fat-Finger Checks:** Blocks orders with prices deviating too far from the current market mid-price.
  - **Size Checks:** Restricts maximum order size to prevent accidental market disruption.
- **Dynamic Limits:**
  - **Drawdown Circuit Breakers:** Suspends trading for a strategy if its intraday loss exceeds a threshold.
  - **Message Rate Limits:** Restricts the number of messages (orders, cancels) sent per second to prevent exchange throttling penalties or runaway loops.
- **The Kill Switch:** An out-of-band administrative signal that immediately cancels all resting orders and flattens outstanding positions.

---

Next Chapter: [Machine Learning & Signal Processing in Finance](file:///Users/ace/Documents/collage/HFT/quantbookn/6_machine_learning.md)