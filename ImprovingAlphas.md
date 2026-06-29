## Improving Alphas

When you build an alpha on BRAIN, the raw signal is rarely submission-ready. Improving an alpha means raising its risk-adjusted performance while keeping its trading cost, concentration, and risk exposures within acceptable bounds. The main levers are **smoothing** (decay), **conditional trading** (`trade_when`), **outlier control** (winsorization/truncation), **normalization** (`rank`, `zscore`, `scale`), and **neutralization** (`group_neutralize`, `regression_neut`). Below we define the key metrics and the operators used to move them.

---

### ▶ Turnover

Turnover of an alpha measures its daily trading activity — how much of the book is bought and sold each day relative to how much is held.

$$
{\rm Turnover} = \frac{\text{Dollar Trading Value}}{\text{Booksize}} = \frac{\text{Value Traded}}{\text{Value Held}}
$$

Equivalently, if $w_{i,t}$ is the dollar weight of instrument $i$ on day $t$, turnover is the average absolute change in weights:

$$
{\rm Turnover}_t = \frac{\sum_i \left| w_{i,t} - w_{i,t-1} \right|}{\sum_i \left| w_{i,t} \right|}
$$

High turnover means high transaction costs, which erode realized returns. There are several ways to reduce it.

**(1) Increase decay.** Applying (or increasing) a linear decay smooths the signal over a lookback window, so weights change more slowly day-to-day.

$$
{\rm increase\ decay} \implies {\rm reduces\ turnover}
$$

The relevant operator is `ts_decay_linear`, a weighted moving average that puts more weight on recent days:

$$
f(x, d) = \frac{\sum_{k=0}^{d-1} (d-k)\, x_{t-k}}{\sum_{k=0}^{d-1} (d-k)} = \frac{d\,x_t + (d-1)\,x_{t-1} + \cdots + 1\cdot x_{t-d+1}}{d + (d-1) + \cdots + 1}
$$

A larger window $d$ produces a smoother, lower-turnover signal at the cost of responsiveness.

```
alpha = ts_decay_linear(close/open - 1, 20);
```

**(2) Conditional trading with `trade_when`.** Instead of re-trading at every market open, only update the position when a meaningful event occurs. `trade_when(condition, alpha, exit)` keeps the previous position while `condition` is false, takes the new `alpha` value when it is true, and closes the position when `exit > 0`. Here `adv20` is the average daily volume over the past 20 days:

```
event = volume > adv20;
alpha = news_cap;
trade_when(event, alpha, -1)
```

This reduces the number of trading days per instrument, directly cutting turnover.

**(3) Filter small changes with `hump`.** The `hump` operator suppresses position changes below a threshold $h$ (a fraction of book), so the alpha only rebalances when the move is large enough to be worth the cost:

$$
\texttt{hump}(x, h):\quad w_t =
\begin{cases}
w_{t-1}, & \left| x_t - w_{t-1} \right| \le h \cdot \text{booksize} \\
x_t, & \text{otherwise}
\end{cases}
$$

```
alpha = hump(rank(-returns), 0.01);
```

---

### ▶ Fitness

Fitness combines returns, turnover, and Sharpe into a single quality score. It is defined as:

$$
\text{Fitness} = \text{Sharpe} \, \sqrt{\frac{\left| {\rm returns} \right|}{\max\!\left[{\rm turnover},\, 0.125\right]}}
$$

where the Sharpe ratio annualizes the daily mean return over its volatility:

$$
\text{Sharpe} = \sqrt{252}\,\frac{\overline{r}}{\sigma_r}
$$

Good alphas generally have high fitness. You can improve fitness by **increasing Sharpe (or returns)** and **reducing turnover**. Note the $\max[\cdot, 0.125]$ floor: once turnover drops below 12.5%, further reductions no longer boost fitness, so over-decaying past that point only sacrifices signal. The passing requirement for fitness on the BRAIN platform is **greater than 1.0**.

---

### ▶ Sharpe and Returns

The Sharpe ratio is the primary measure of risk-adjusted performance. On BRAIN the in-sample passing threshold is typically **Sharpe > 1.25**. Two ways to raise it:

- **Reduce noise** — smoothing (`ts_decay_linear`) and normalization (`rank`, `zscore`) lower volatility without necessarily lowering mean return.
- **Reduce unwanted risk** — neutralization removes systematic exposures that add volatility but no edge (see below).

---

### ▶ Normalization Operators

Raw signals often have skewed, scale-dependent distributions. Normalizing them cross-sectionally makes the alpha robust to outliers and comparable across instruments.

**Rank** maps each value to its cross-sectional percentile in $[0,1]$:

$$
\texttt{rank}(x_i) = \frac{1}{N}\sum_{j=1}^{N} \mathbf{1}\left[ \text{if } x_j \le x_i \right]
$$

This is the most common transform — it is fully robust to outliers because only the ordering matters.

**Z-score** centers and standardizes the signal cross-sectionally:

$$
\texttt{zscore}(x_i) = \frac{x_i - \mu_x}{\sigma_x}
$$

**Scale** rescales the book so the sum of absolute weights equals a target $a$ (default 1), controlling gross exposure:

$$
\texttt{scale}(x, a) = a \cdot \frac{x_i}{\sum_j \left| x_j \right|}
$$

---

### ▶ Outlier Control: Winsorization and Truncation

Extreme values can dominate the book and create concentration risk. **Winsorization** clips the signal at $\pm s$ standard deviations from the mean:

$$
\texttt{winsorize}(x, s):\quad x_i \mapsto \min \big(\max(x_i,\ \mu - s\sigma),\ \mu + s\sigma\big)
$$

**Truncation** caps the maximum weight any single instrument can hold to a fraction $m$ of the book, improving diversification:

$$
\left| w_i \right| \le m \cdot \text{booksize}
$$

```
alpha = truncate(rank(signal), 0.05);   /* no position exceeds 5% of book */
```

---

### ▶ Neutralization Operators

Neutralization removes systematic exposures (to the market, a sector, or a risk factor) so the alpha bets only on the residual idiosyncratic signal. This usually lowers volatility and raises Sharpe.

**Group neutralization** `group_neutralize` subtracts the group mean so each group (e.g. sector, industry) nets to zero:

$$
\mathrm{GN}(x, g)_i = x_i - \frac{1}{\left| G_i \right|}\sum_{j \in G_i} x_j
$$

```
alpha = group_neutralize(rank(-returns), sector);
```

**Regression neutralization** `regression_neut` removes the component of the alpha explained by one or more risk factors $f$ by regressing the signal on them and keeping the residual $\varepsilon$:

$$
x = \beta f + \varepsilon, \qquad RN(x, f) = \varepsilon = x - \beta f
$$

This is how you make an alpha market-neutral, beta-neutral, or factor-neutral.

---

### ▶ Putting It Together — A Worked Example

A typical "improved" alpha chains several of these operators: build a raw signal, normalize it, neutralize unwanted exposures, smooth to control turnover, and trade conditionally.

```
raw     = -ts_delta(close, 5) / close;          /* 5-day reversal signal     */
norm    = rank(winsorize(raw, 4));              /* clip outliers, then rank   */
neutral = group_neutralize(norm, sector);       /* remove sector exposure     */
smooth  = ts_decay_linear(neutral, 10);         /* reduce turnover            */
event   = volume > adv20;                        /* only trade on liquid days  */
alpha   = trade_when(event, smooth, -1);
```

Each step targets a specific metric:

| Step | Operator | Metric improved |
|------|----------|-----------------|
| Outlier control | `winsorize` | Drawdown, stability |
| Normalization | `rank` | Sharpe (lower noise) |
| Neutralization | `group_neutralize` | Sharpe (less risk) |
| Smoothing | `ts_decay_linear` | Turnover ↓, Fitness ↑ |
| Conditional trade | `trade_when` | Turnover ↓ |

---

### ▶ Drawdown

Drawdown measures the largest peak-to-trough decline in the alpha's cumulative PnL over the backtest. If $C_t$ is the cumulative PnL at time $t$, the drawdown at $t$ is the drop from the running peak:

$$
{\rm DD}_t = \frac{\max_{\tau \le t} C_\tau - C_t}{\max_{\tau \le t} C_\tau}
$$

and the headline figure is the **maximum drawdown** over the whole period:

$$
{\rm MaxDD} = \max_{t}\ {\rm DD}_t = \max_{t}\left[\frac{\max_{\tau \le t} C_\tau - C_t}{\max_{\tau \le t} C_\tau}\right]
$$

A large drawdown signals an alpha that suffers prolonged, painful losing stretches even if its long-run Sharpe is acceptable. On BRAIN the in-sample limit is typically **MaxDrawdown < 50%**. Ways to reduce it:

- **Outlier control** (`winsorize`, `truncate`) caps the damage from any single instrument or extreme day.
- **Neutralization** (`group_neutralize`, `regression_neut`) removes systematic exposures that cause correlated, simultaneous losses.
- **Smoothing** (`ts_decay_linear`) dampens whipsaw losses from a noisy signal.

A useful companion metric is the **Calmar-style ratio**, return over max drawdown:

$$
{\rm Calmar} = \frac{\text{Annualized Return}}{{\rm MaxDD}}
$$

---

### ▶ Margin and IC

**Margin** measures how much profit the alpha extracts per dollar traded — essentially profitability per unit of turnover. It is usually quoted in basis points (bps):

$$
{\rm Margin} = \frac{\text{Total PnL}}{\text{Total Dollars Traded}} = \frac{\text{Mean returns}}{\text{Turnover}}\times 10^{4}\ \text{(bps)}
$$

A low margin (a few bps) means transaction costs can easily wipe out the edge; a healthy margin gives the alpha room to survive realistic slippage. Reducing turnover (decay, `trade_when`, `hump`) raises margin directly, since the same PnL is earned with fewer trades.

**Information Coefficient (IC)** measures the predictive power of the signal — the cross-sectional correlation between the alpha's forecast on day $t$ and the realized forward returns on day $t+1$:

$$
{\rm IC}_t = {\rm corr}\!\left(\text{alpha}_{i,t},\ r_{i,t+1}\right)
$$

The headline **mean IC** is the average over all days, and its stability is captured by the **information ratio of the IC**:

$$
\overline{\rm IC} = \frac{1}{T}\sum_{t=1}^{T} {\rm IC}_t,
\qquad
{\rm IR}_{\rm IC} = \frac{\overline{\rm IC}}{\sigma_{\rm IC}}
$$

A small but *consistent* positive IC (e.g. 0.02–0.05 with high IR) is often more valuable than a large but erratic one. The `rank`-based version (Spearman IC, i.e. `ts_corr` of ranks) is the most robust to outliers.

---

### ▶ Sub-Universe Sharpe

The headline Sharpe is computed on the full trading universe, but an alpha can earn most of its performance from a handful of large, liquid names while doing nothing — or losing — on the rest. The **sub-universe Sharpe** checks robustness by re-computing Sharpe on a restricted universe (typically the smaller / less liquid subset, e.g. names *outside* the top-tier liquidity bucket):

$$
{\rm Sharpe}_{\rm sub} = \sqrt{252}\,\frac{\overline{r}^{\,\rm sub}}{\sigma_r^{\rm sub}},
\qquad U_{\rm sub} \subset U_{\rm full}
$$

BRAIN reports the ratio of sub-universe to full-universe Sharpe as a consistency check:

$$
\rho_{\rm sub} = \frac{{\rm Sharpe}_{\rm sub}}{{\rm Sharpe}_{\rm full}}
$$

A value close to 1 means the edge generalizes across the universe; a value near 0 (or negative) means the alpha is concentrated in a few names and is fragile. To improve it, make the signal universe-agnostic: normalize cross-sectionally (`rank`, `zscore`), neutralize by group (`group_neutralize`), and avoid letting a few large-cap names dominate the book (`truncate`).

---

### ▶ Correlation to Existing Alphas

A new alpha only adds value if it is **not** already captured by alphas you (or the platform) hold. BRAIN therefore checks the correlation of your alpha's daily PnL against the existing pool:

$$
\rho_{a,b} = \frac{{\rm Cov}\!\left({\rm PnL}_a,\ {\rm PnL}_b\right)}{\sigma_{{\rm PnL}_a}\,\sigma_{{\rm PnL}_b}}
$$

The submission constraint is usually that the **maximum** pairwise correlation against the existing set stays below a threshold (commonly **< 0.7**):

$$
\max_{b\,\in\,{\rm pool}}\ \rho_{a,b} < 0.7
$$

Two alphas that are individually good but highly correlated contribute almost the same bet; combining low-correlation alphas is what diversifies risk and grows the overall portfolio Sharpe. To *lower* correlation to an existing pool:

- Use a **different data field or signal family** (e.g. fundamentals vs. price-volume vs. news).
- Apply a **different neutralization** (sector vs. market vs. factor) so the residual exposure differs.
- Change the **horizon** (`ts_delta`/decay window) so the alpha trades on a different time scale.
- Combine signals so the new alpha's residual — after regressing out the correlated component (`regression_neut`) — carries the fresh information.

The portfolio benefit of adding a low-correlation alpha follows the standard two-asset combination: for equal-volatility alphas combined equally,

$$
\sigma_{\rm combined} = \sigma\sqrt{\frac{1 + \rho_{a,b}}{2}}
$$

so the lower the correlation $\rho_{a,b}$, the greater the volatility reduction — and the higher the combined Sharpe.
