# 10 — Basic Finance: Arbitrage, Options & Pricing Models

A ground-up reference for the core ideas every quant trader needs before touching derivatives.

---

## 1. Arbitrage

**Arbitrage** is the simultaneous purchase and sale of equivalent assets in different markets to profit from a price discrepancy — with zero net investment and zero risk.

### The No-Arbitrage Principle

Almost every pricing formula in finance rests on this single idea:

> *If two portfolios have identical future payoffs in every possible state of the world, they must have the same price today.*

If they don't, a trader can buy the cheap one, sell the expensive one, pocket the difference now, and have zero net exposure going forward. Markets quickly close such gaps.

### Types of Arbitrage

| Type | Mechanism |
|---|---|
| **Pure / Risk-Free** | Identical asset priced differently on two exchanges. Buy low, sell high simultaneously. |
| **Statistical Arb** | Two assets are historically correlated. Trade the spread when it deviates beyond a threshold. |
| **Triangular Arb** | FX: A→B→C→A exploits inconsistent cross rates. |
| **Merger Arb** | Buy target, short acquirer after announcement; profit if deal closes at stated price. |
| **Convertible Arb** | Long convertible bond, short underlying equity to isolate the embedded optionality. |

### Simple Example

AAPL trades at **$150.00** on NASDAQ and **$150.30** on a regional exchange.

- Buy 1000 shares on NASDAQ: −$150,000  
- Sell 1000 shares on regional: +$150,300  
- **Risk-free profit: $300** (minus transaction costs)

Real arb opportunities vanish in milliseconds. HFT firms exist primarily to capture them.

---

## 2. Options — Overview

An **option** is a contract that gives the buyer the *right, but not the obligation*, to buy or sell an underlying asset at a predetermined price (the **strike**, K) on or before a specified date (the **expiry**, T).

The seller (writer) of the option takes on the corresponding obligation and receives a **premium** upfront.

### Key Terms

| Term | Symbol | Meaning |
|---|---|---|
| Underlying price | S | Current market price of the asset |
| Strike price | K | Price at which option can be exercised |
| Time to expiry | T | Years remaining until expiration |
| Risk-free rate | r | Continuously compounded, e.g. US T-bill rate |
| Volatility | σ | Annualised standard deviation of returns |
| Premium / Price | C or P | Cost to buy the option today |

---

## 3. Call Options and Put Options

### Call Option

A **call** gives the buyer the right to **buy** the underlying at strike K.

**Payoff at expiry:**

```
Payoff_call = max(S_T − K, 0)
```

- If `S_T > K`: in-the-money (ITM) — you exercise and profit from the difference.
- If `S_T < K`: out-of-the-money (OTM) — you let it expire; loss is capped at the premium paid.

**Example:** Buy a call on TSLA, K = $250, premium = $8.  
- TSLA expires at $270 → payoff = $20, net profit = $20 − $8 = **$12**  
- TSLA expires at $240 → payoff = $0, net loss = **−$8**

---

### Put Option

A **put** gives the buyer the right to **sell** the underlying at strike K.

**Payoff at expiry:**

```
Payoff_put = max(K − S_T, 0)
```

- If `S_T < K`: ITM — you can sell above market price.
- If `S_T > K`: OTM — expires worthless.

**Example:** Buy a put on SPY, K = $500, premium = $6.  
- SPY expires at $480 → payoff = $20, net profit = $20 − $6 = **$14**  
- SPY expires at $510 → payoff = $0, net loss = **−$6**

---

### Put-Call Parity (No-Arbitrage Relationship)

For European options on a non-dividend-paying stock:

```
C − P = S − K · e^(−rT)
```

This is enforced purely by no-arbitrage. If it breaks, you can construct a riskless profit by:
- Buying the cheap side (e.g., synthetic via call+bond)
- Selling the expensive side (e.g., actual stock+put)

---

## 4. European vs American Options

| Feature | European | American |
|---|---|---|
| Exercise timing | **Only at expiry** | **Any time up to expiry** |
| Pricing complexity | Lower (closed-form BSM) | Higher (needs numerical methods) |
| Where traded | Index options (SPX), most FX | Individual equity options (AAPL, TSLA) |
| Early exercise premium | None | Positive for deep ITM cases |

### When is Early Exercise Optimal?

For American **calls** on non-dividend-paying stocks: **never** early — you're better off selling the call (which retains time value) than exercising it.

For American **puts** or **calls on dividend-paying stocks**: early exercise can be optimal when:
- Deep ITM put: the interest earned on K outweighs remaining time value.
- Call just before a large dividend: capturing the dividend may exceed time value lost.

---

## 5. The Binomial Option Pricing Model

The binomial model discretises time into N steps. At each step the stock price either goes **up** by factor u or **down** by factor d.

### One-Step Setup

```
         S·u  (probability q, risk-neutral)
S
         S·d  (probability 1−q, risk-neutral)
```

**Risk-neutral probability:**

```
q = (e^(rΔt) − d) / (u − d)
```

**Option value today:**

```
V = e^(−rΔt) · [q · V_u + (1−q) · V_d]
```

Where `V_u` and `V_d` are option values at the up and down nodes.

### Standard Parameterisation (Cox-Ross-Rubinstein)

```
u = e^(σ√Δt)
d = 1/u = e^(−σ√Δt)
Δt = T/N
```

### Multi-Step Recombining Tree

For N steps, the stock price at node (i steps up, j steps down) is:

```
S_{i,j} = S · u^i · d^j
```

Work backwards from expiry: compute payoffs at all terminal nodes, then discount back step by step using the risk-neutral probability.

### Example — 2-Step Binomial Call

```
S = 100,  K = 100,  r = 5%,  σ = 20%,  T = 1 year,  N = 2
Δt = 0.5
u = e^(0.20 · √0.5) = 1.1519
d = 1/u = 0.8681
q = (e^(0.05·0.5) − 0.8681) / (1.1519 − 0.8681) = 0.5765
```

Terminal nodes (t=1):
```
S_uu = 100 · 1.1519² = 132.68  → C_uu = max(132.68−100, 0) = 32.68
S_ud = 100 · 1.1519 · 0.8681 = 100.00 → C_ud = 0
S_dd = 100 · 0.8681²  = 75.36  → C_dd = 0
```

Step back to t=0.5:
```
C_u = e^(−0.05·0.5) · [0.5765·32.68 + 0.4235·0] = 18.37
C_d = e^(−0.05·0.5) · [0.5765·0    + 0.4235·0] = 0
```

Step back to t=0:
```
C = e^(−0.05·0.5) · [0.5765·18.37 + 0.4235·0] = 10.30
```

**Call price ≈ $10.30**

As N → ∞, the binomial model converges to BSM.

---

## 6. The Black-Scholes-Merton (BSM) Model

### Assumptions

1. The stock follows geometric Brownian motion: `dS = μS dt + σS dW`
2. No dividends, no transaction costs, continuous trading
3. Constant r and σ over the life of the option
4. No arbitrage

### Closed-Form Formulas

**European Call:**

```
C = S · N(d1) − K · e^(−rT) · N(d2)
```

**European Put:**

```
P = K · e^(−rT) · N(−d2) − S · N(−d1)
```

Where:

```
d1 = [ ln(S/K) + (r + σ²/2) · T ] / (σ · √T)

d2 = d1 − σ · √T
```

And `N(·)` is the **cumulative standard normal distribution function**:

```
N(x) = P(Z ≤ x),   Z ~ N(0,1)
```

### Interpretation of Terms

| Term | Meaning |
|---|---|
| `S · N(d1)` | Expected value of receiving the stock, conditional on exercise |
| `K · e^(−rT) · N(d2)` | Present value of paying the strike, weighted by exercise probability |
| `N(d2)` | Risk-neutral probability that the option expires ITM |
| `d1` | Captures moneyness adjusted for drift and convexity |

### The Greeks

Sensitivities that every options trader monitors:

| Greek | Formula | Meaning |
|---|---|---|
| **Delta** Δ | `N(d1)` for call; `N(d1)−1` for put | Change in option price per $1 move in S |
| **Gamma** Γ | `N'(d1) / (S·σ·√T)` | Rate of change of delta |
| **Theta** Θ | `−[S·N'(d1)·σ/(2√T)] − rKe^(−rT)·N(d2)` | Time decay per day |
| **Vega** ν | `S·√T·N'(d1)` | Change per 1% move in σ |
| **Rho** ρ | `K·T·e^(−rT)·N(d2)` | Change per 1% move in r |

Where `N'(x) = e^(−x²/2) / √(2π)` is the standard normal PDF.

---

## 7. Worked BSM Examples

### Example A — Call Option Pricing

```
S = 100,  K = 105,  r = 5%,  σ = 25%,  T = 0.5 years
```

Step 1 — compute d1 and d2:
```
d1 = [ln(100/105) + (0.05 + 0.25²/2)·0.5] / (0.25·√0.5)
   = [−0.04879 + 0.04063] / 0.17678
   = −0.00816 / 0.17678
   = −0.04616

d2 = −0.04616 − 0.17678 = −0.22294
```

Step 2 — look up / compute N(d1) and N(d2):
```
N(−0.04616) ≈ 0.4816
N(−0.22294) ≈ 0.4118
```

Step 3 — plug in:
```
C = 100·0.4816 − 105·e^(−0.025)·0.4118
  = 48.16 − 105·0.9753·0.4118
  = 48.16 − 42.19
  = $5.97
```

### Example B — Put Option via Put-Call Parity

Using the same inputs:
```
P = C − S + K·e^(−rT)
  = 5.97 − 100 + 105·e^(−0.025)
  = 5.97 − 100 + 102.41
  = $8.38
```

### Example C — Implied Volatility

You observe a call trading at $7.50 (above our theoretical $5.97). The market is pricing in higher σ than 25%. The σ that makes BSM output $7.50 is called **implied volatility (IV)**.

IV is found numerically (Newton-Raphson, bisection). In Python:

```python
from scipy.optimize import brentq
from scipy.stats import norm
import numpy as np

def bsm_call(S, K, T, r, sigma):
    d1 = (np.log(S/K) + (r + 0.5*sigma**2)*T) / (sigma*np.sqrt(T))
    d2 = d1 - sigma*np.sqrt(T)
    return S*norm.cdf(d1) - K*np.exp(-r*T)*norm.cdf(d2)

def implied_vol(market_price, S, K, T, r):
    f = lambda sigma: bsm_call(S, K, T, r, sigma) - market_price
    return brentq(f, 1e-6, 10.0)

iv = implied_vol(7.50, 100, 105, 0.5, 0.05)
print(f"Implied Volatility: {iv:.2%}")   # → ~29.3%
```

---

## 8. Using This Knowledge in Real Trades

### 8.1 Arbitrage in Practice

- **Statistical pairs trading:** Model the spread between two correlated assets (e.g., GLD vs GDX). When the z-score of the spread exceeds ±2σ, enter a mean-reversion trade. Exit at convergence. This is the backbone of many quant L/S equity funds.
- **Options arbitrage:** If put-call parity breaks (net of bid-ask), you can lock in a risk-free spread. In practice this requires fast execution — it's dominated by market makers.

### 8.2 Delta Hedging (Market Making)

A market maker who sells a call can **delta-hedge** by buying `Δ = N(d1)` shares of the underlying. The position is then neutral to small moves in S. Profit comes from collecting the bid-ask spread, not directional bets.

As S moves, delta changes (gamma risk), so the hedge must be rebalanced dynamically. The cost of rebalancing is your **gamma P&L**, and it's offset by **theta decay** you collect as the option writer.

```
Daily P&L of a delta-hedged short option ≈ Θ − ½·Γ·(ΔS)²
```

If realised volatility < implied volatility, the theta you collect exceeds the gamma losses → **short vol is profitable**. This is the core P&L identity for volatility traders.

### 8.3 Volatility Trading

- **Long Straddle:** Buy call + put at same strike. Profits if the underlying makes a large move in either direction. You're long gamma and short theta. Use when you expect a big event (earnings, Fed decision) and IV is cheap.
- **Short Strangle:** Sell OTM call + OTM put. Collect premium. Profit if the stock stays range-bound. You're short gamma. Use when IV is elevated relative to your realised vol forecast.
- **Vol Surface Arbitrage:** Compare IV across strikes (skew) and maturities (term structure). A quant model can identify mispricings and construct spread trades to isolate them.

### 8.4 The Quant Workflow

```
1. SIGNAL
   Identify an edge: mean reversion, momentum, vol misprice, factor exposure.
   Backtest with realistic transaction costs.

2. SIZING
   Kelly Criterion or fractional Kelly to size positions relative to edge and variance.
   f* = edge / odds = (expected return) / (variance)

3. HEDGING
   Strip out unwanted risks using Greeks.
   Delta-hedge directional exposure. Vega-hedge if you want pure gamma bets.

4. EXECUTION
   Minimise market impact. Use VWAP/TWAP algos for equity.
   For options: work limit orders, don't cross the spread on illiquid strikes.

5. RISK MANAGEMENT
   Monitor Greeks in real time. Set hard stop-losses on vega and delta.
   Watch for regime change (vol spikes invalidate short-vol strategies instantly).
```

### 8.5 Key Rules Every Quant Learns the Hard Way

| Rule | Why it matters |
|---|---|
| **IV ≠ realised vol** | The difference (vol risk premium) is the edge in options. Track it constantly. |
| **Gamma kills short vol positions** | A large move wipes out weeks of theta. Always know your max loss. |
| **Put-call parity is a constraint** | If your model prices calls and puts inconsistently, it has a bug. |
| **Binomial ≠ BSM for American options** | Never price early-exercise options with the BSM closed form. Use trees or finite differences. |
| **Liquidity trumps edge** | A 30-cent theoretical edge on an illiquid option with a 40-cent spread is not a trade. |

---

## Summary

```
Arbitrage       → No-free-lunch principle that underlies all pricing
Options         → Asymmetric payoffs: right without obligation
Call / Put      → Buy: pay premium, gain leverage. Write: collect premium, take risk.
European/Amer   → Exercise timing determines pricing method
Binomial        → Discrete tree; handles American options; converges to BSM
BSM             → Closed-form for European options; C = S·N(d1) − Ke^(−rT)·N(d2)
Greeks          → Delta, Gamma, Theta, Vega — your real-time risk dashboard
Trading edge    → Mispricings in vol surface, statistical divergences, arb spreads
```

The BSM model is a lens, not a truth. Real markets have skew, jumps, and stochastic vol. But understanding BSM rigorously is the prerequisite to knowing *why* more advanced models (Heston, SABR, local vol) exist and where they improve on it.
