# Machine Learning and Signal Processing in Finance

Applying machine learning to financial modeling is notoriously difficult. Unlike computer vision or natural language processing, financial markets have an extremely low signal-to-noise ratio (SNR) and are highly non-stationary. This chapter covers the mathematical frameworks, specialized feature engineering, labeling, validation, and reinforcement learning techniques developed specifically for quantitative finance.

---

## 1. Feature Engineering: Stationarity vs. Memory

In traditional machine learning, we assume that the training data and test data are drawn from the same underlying stationary distribution. Financial asset prices $P_t$, however, are non-stationary ($I(1)$).

To feed prices into standard models, quants historically computed integer returns:
$$R_t = \frac{P_t - P_{t-1}}{P_{t-1}}$$
While $R_t$ is stationary ($I(0)$), it has **no memory**: it completely removes the long-term price level and historical trend.

### 1.1 Fractional Differentiation

To balance this trade-off, we use **Fractional Differentiation** (introduced by Marcos López de Prado). It allows us to differentiate a time series by a real number $d \in (0, 1)$, removing the unit root (achieving stationarity) while preserving historical memory.

The fractional difference operator is defined using the binomial expansion of $(1 - B)^d$, where $B$ is the backshift operator ($B X_t = X_{t-1}$):
$$(1 - B)^d = \sum_{k=0}^{\infty} (-1)^k \binom{d}{k} B^k = \sum_{k=0}^{\infty} \omega_k B^k$$
where the weights $\omega_k$ are computed recursively:
$$\omega_0 = 1, \quad \omega_k = -\omega_{k-1} \frac{d - k + 1}{k}$$

The fractionally differentiated series $X_t^{(d)}$ is:
$$X_t^{(d)} = \sum_{k=0}^{\infty} \omega_k P_{t-k}$$

We select the minimum value of $d$ that passes ADF stationarity tests, preserving the maximum amount of historical price history.

---

### 1.2 Microstructural Features

For high-frequency models, features are built directly from order book dynamics:

1. **Order Flow Imbalance (OFI):** Measures the net supply and demand changes at the best bid and ask.
   Let $P_b(t)$ and $Q_b(t)$ be the best bid price and quantity. Let $P_a(t)$ and $Q_a(t)$ be the best ask price and quantity.
   $$\text{OFI}_t = \Delta Q_b(t) - \Delta Q_a(t)$$
   where:
   $$\Delta Q_b(t) = \begin{cases} Q_b(t), & \text{if } P_b(t) > P_b(t-1) \\ Q_b(t) - Q_b(t-1), & \text{if } P_b(t) = P_b(t-1) \\ 0, & \text{if } P_b(t) < P_b(t-1) \end{cases}$$
   $$\Delta Q_a(t) = \begin{cases} Q_a(t), & \text{if } P_a(t) < P_a(t-1) \\ Q_a(t) - Q_a(t-1), & \text{if } P_a(t) = P_a(t-1) \\ 0, & \text{if } P_a(t) > P_a(t-1) \end{cases}$$
2. **Bid-Ask Volume Imbalance:**
   $$\text{Imbalance}_t = \frac{Q_b(t) - Q_a(t)}{Q_b(t) + Q_a(t)}$$

---

## 2. Advanced Labeling: The Triple-Barrier Method

Standard machine learning models often label observations using fixed-horizon returns (e.g., classifying if $P_{t+h} - P_t > 0$). This approach is unrealistic because:
- It ignores stop-loss levels that would have triggered during the holding period.
- Volatility is constant over the horizon, which is false in real markets.

The **Triple-Barrier Method** sets three barriers based on dynamic volatility:
1. **Upper Barrier:** Horizontal line at $+U_t$, representing a profit target (Take Profit).
2. **Lower Barrier:** Horizontal line at $-D_t$, representing a stop-loss limit.
3. **Vertical Barrier:** Vertical line at $t + H$, representing expiration of the trade.

```
Price
  ▲
  │              Take Profit Barrier (+U_t)
  │    ┌──────────────────────────────────┐
  │   /                                   │
  │  / ◄── Asset Price Path               │
  │ /                                     │
  ├─ ◄── Trade Entry (t)                  │
  │ \                                     │
  │  \                                    │
  │   \                                   │
  │    └──────────────────────────────────┴──────► Time
                 Stop Loss Barrier (-D_t) |
                                      Expiration (t + H)
```

### Label Assignment
Let the path hit one of the barriers first:
- Label $= 1$ if the path hits the upper barrier first.
- Label $= -1$ if the path hits the lower barrier first.
- Label $= 0$ if the path hits the vertical barrier first (no clear trend).

We scale the horizontal barriers $U_t$ and $D_t$ using a rolling estimate of standard deviation (volatility) to ensure they adapt to market regimes.

---

## 3. Validation: Purging and Embargoing

Standard $K$-Fold Cross Validation fails in finance because financial observations are highly correlated over time.

If an observation $t_i$ overlaps in time with another observation $t_j$ (e.g., two trades that are held simultaneously), information from the training set leaks into the testing set, leading to severe overfitting.

```
TRAIN SET                              TEST SET
[========== overlapping data ==========] | [========== leaked data ==========]
                                       |
                                Purge Window
```

### López de Prado's Validation Pipeline
1. **Purging:** Remove from the training set any labels whose evaluation window overlaps with the test set.
2. **Embargoing:** Because of autoregressive noise, historical data *after* the test set can also contain leaked information. We apply an embargo window (typically $1\%$ of the dataset length) directly after the test set, removing those training instances.

---

## 4. Unsupervised Learning: Principal Component Analysis (PCA)

Principal Component Analysis (PCA) is widely used to extract structural factors from yield curves (interest rates) or equity portfolios.

Let $X$ be a matrix of normalized returns for $N$ assets. The covariance matrix is:
$$\Sigma = \frac{1}{M} X^T X$$
Using eigenvalue decomposition:
$$\Sigma = W \Lambda W^T$$
where:
- $W$ is the matrix of eigenvectors (principal component loadings).
- $\Lambda$ is the diagonal matrix of eigenvalues, representing the variance explained by each component.

In Fixed Income, the first three principal components explain over $95\%$ of yield curve movements:
1. **PC1 (Level):** A parallel shift in rates across all maturities.
2. **PC2 (Slope):** A twist where short-term and long-term rates move in opposite directions.
3. **PC3 (Curvature):** A bending of the medium-term rates relative to the wings.

---

## 5. Reinforcement Learning for Optimal Execution

Optimal execution is the task of selling (or buying) a large block of shares $Q$ over a fixed time horizon $T$ while minimizing transaction costs and market impact.

This can be formulated as a Markov Decision Process (MDP):
- **State ($s_t$):** Current inventory $q_t$, time remaining $T - t$, and current order book spread/depth.
- **Action ($a_t$):** The volume to submit as limit or market orders in the next interval.
- **Reward ($r_t$):** The execution value relative to a benchmark price (e.g., VWAP or arrival price), minus a penalty for inventory risk:
  $$r_t = (P_t - P_{\text{arrival}}) \cdot a_t - \phi q_t^2$$
  where $\phi$ is the inventory aversion parameter.

Algorithms like Deep Q-Networks (DQN) and Proximal Policy Optimization (PPO) are trained on simulated historical limit order book data to learn optimal dynamically adjusted execution trajectories.

---

Next Chapter: [High-Frequency System Architecture & C++ Low-Latency Design](file:///Users/ace/Documents/collage/HFT/quantbookn/7_tools_and_architecture.md)