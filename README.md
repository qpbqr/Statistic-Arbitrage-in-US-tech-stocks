# Statistical Arbitrage (ETF & PCA Residual + OU Mean Reversion)

A practical stat-arb prototype on a fixed US equity universe (mostly tech).  
Core idea: **remove common risk**, trade **idiosyncratic mean reversion**.

## Core Logic (Essentials)

### 1) Build “idiosyncratic residuals”
Daily log returns.

1. **Factor-neutralize** each stock using ETFs:
   - regress on `SPY` (market) and `XLK` (tech)
   - residual = stock return − (beta_SPY · SPY + beta_XLK · XLK)

2. **Residual PCA cleanup**
   - standardize residuals
   - PCA with `N_COMPONENTS=2`
   - remove PCA-explained common residual component
   - remaining part is treated as **idiosyncratic residual** `eps`

### 2) Mean-reversion signal (OU-style)
Maintain a cumulative state per stock:
- `x_t = x_{t-1} + eps_t`

Fit an AR(1) on the last `OU_WINDOW=63` days of `x` to approximate OU behavior:
- keep names whose **half-life** is in `[1, 60]` days

Signal:
- `s = (x - mu) / sigma_eq`  (standardized deviation from OU mean)

### 3) Trading rules
- **Dynamic entry threshold** per stock: 80th percentile of historical `|s|`, clipped to `[1.0, 2.5]`
- **Enter**
  - `s > entry_thres` → short
  - `s < -entry_thres` → long
- **Exit** (symmetric): close when `|s|` falls back near the mean (`EXIT_THRES=0.75`)
- **Stop loss**: close if `|s| > 3.3`

### 4) Execution modes (two implementations)
- **Continuous rebalancing**: equal-weight among active signals daily (costs set to 0 in this experiment)
- **Fixed-unit trading**: discrete trades on signal changes, ~$50k per position (scaled with NAV)
  - includes a **net exposure cap**
  - optional transaction cost (3 bps) on traded notional

## What’s Auxiliary
A **Kalman-filter beta** version is included only as a side experiment to estimate factor betas dynamically.  
In the shown run, it underperformed the simple rolling-window OLS betas.

## Notes / Caveats (important)
- Uses **Open** prices from Yahoo Finance (`yfinance`).
- Static ticker list ⇒ survivorship and selection bias possible.
- Continuous mode has **no transaction cost model** in current runs.
