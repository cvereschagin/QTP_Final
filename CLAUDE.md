# FIN452 Quantitative Trading Project — CLAUDE.md

## Project Overview

Single-file Quarto project implementing a **Kalman Filter + K-Means Clustering + Yield Curve Sector Rotation** strategy across S&P 500 Select Sector SPDRs. All analysis lives in one file:

**`momentum_sector_rotation.qmd`** → renders to self-contained `momentum_sector_rotation.html`

To render:
```bash
quarto render momentum_sector_rotation.qmd
```

---

## Symbols

| Role | Symbol |
|---|---|
| Benchmark | SPY |
| Sectors (11) | XLB, XLC, XLE, XLF, XLI, XLK, XLP, XLRE, XLV, XLU, XLY |

**Known launch dates** (no data before these):
- XLC — June 18, 2018
- XLRE — October 7, 2015

---

## Date Ranges

| Period | Range |
|---|---|
| Training | June 2000 – December 2024 |
| Testing | January 2025 – March 2026 |
| Download buffer | TRAIN_START − 30 days |

---

## Strategy Summary

1. **Kalman Filter** (`dlmFilter`, NOT `dlmSmooth`) applied to each sector's daily adjusted close → extracts `kal_level` and `kal_velocity` (noise-filtered trend slope)
2. **K-Means Clustering** (base R `kmeans()`) on `kal_velocity` only (single feature, `k` clusters) at each month-end — cluster with highest mean `kal_velocity` = "Bull cluster"
3. **Yield Curve Overlay** (FRED `T10Y2Y`): if 10Y-2Y spread < -0.25%, override cluster signal → LONG XLP, XLU, XLV (defensives) only
4. **Signal**: sectors in Bull cluster (or defensives if yield curve inverted) → LONG equal weight; others → FLAT
5. **Execution**: signal at month-end Close_T → execute at Open_{T+1} → earn Open-to-Close return; hold until next month-end
6. **Transaction cost**: 0.05% one-way per unit of weight change
7. **Long-only, monthly rebalancing**: no short positions; ~12 rebalances/year

---

## Key Design Decisions

### Look-Ahead Bias Safeguards
- Use `dlmFilter()` only, never `dlmSmooth()`
- Yield curve forward-filled with `zoo::na.locf()` using only past observations
- All return calculations use `dplyr::lag()` (backward-looking)
- Parameter optimization (`k` and `Q`) on training data exclusively
- Test set untouched until Section 9

### Return Types
| Type | Formula | Use |
|---|---|---|
| C2C (Close-to-Close) | `adjusted / lag(adjusted) - 1` | SPY benchmark |
| O2C (Open-to-Close) | `close / open - 1` | **Strategy return** (entered at open) |
| C2O (Close-to-Open) | `open / lag(close) - 1` | Overnight gap reference |

### Transaction Costs
- 0.05% one-way (`TC_RATE = 0.0005`)
- Applied as `|weight_change| × TC_RATE`
- Results shown gross and net

---

## Architecture — Chunk Execution Order

Chunks must run in this strict sequence (Quarto executes top-to-bottom):

```
setup                          → constants, packages
data_download                  → raw_prices via tq_get(); yc_raw via FRED [cache: TRUE]
data_quality                   → coverage table (Table 1)
data_split                     → sector_train/test, spy_train/test, yc_filled [FIREWALL]
helper_functions               → run_kalman(), omega_ratio(), generate_signals_clustered(),
                                  build_portfolio(), compute_metrics()
apply_kalman_train             → kalman_train, returns_train, spy_ret_train [cache: TRUE]
default_ranking_inspection     → train_signals_default → Figure 1 (heatmap)
param_grid_setup               → param_grid (16 combos: k=2:5 × Q=4 values)
parallel_optimization          → opt_results [cache: TRUE — slow]
surface_plot_omega             → Figure 2 (plotly 3D surface)
optimal_params                 → OPTIMAL_K, OPTIMAL_Q
zscored_metrics                → Table 3
refit_optimal                  → kalman_train_opt, train_signals_opt, portfolio_train,
                                  train_combined [cache: TRUE]
train_metrics_table            → Table 4
velocity_animation             → Figure 2 (animated bar chart, Bull cluster highlighted)
train_cum_chart                → Figure 3
build_test_features            → kalman_full_opt [cache: TRUE]
test_clustering_and_portfolio  → test_signals, portfolio_test, test_combined
test_metrics_table             → Table 5
cum_return_chart               → Figure 4 (gross vs net vs SPY)
pa_charts                      → Figure 5 (charts.PerformanceSummary)
drawdown_chart                 → Figure 6 (chart.Drawdown)
spy_chart                      → Figure 7 (chartSeries + overlays)
monthly_holdings               → Table 6
rolling_correlation            → Figure 8 (plotly)
stress_test_computations       → 3 stress scenarios
stress_metrics_table           → Table 7
stress_cum_chart               → Figure 9
session_info
```

---

## Key Global Objects

| Object | Type | Description |
|---|---|---|
| `SYMBOLS` / `SECTOR_SYMBOLS` | character vec | All 12 / just 11 sectors |
| `DEFAULT_K` / `K_GRID` | scalar / int vec | Default k=3; grid search 2:5 |
| `YC_THRESHOLD` | scalar | -0.0025 (−0.25%) inversion trigger |
| `DEFENSIVE_SECTORS` | character vec | c("XLP","XLU","XLV") |
| `OPTIMAL_K` / `OPTIMAL_Q` | scalar | Best params from grid search |
| `yc_filled` | tibble | FRED T10Y2Y, forward-filled to all dates |
| `kalman_train_opt` / `kalman_full_opt` | tibble | Kalman output with OPTIMAL_Q |
| `train_signals_opt` / `test_signals` | tibble | cols: symbol, date, signal, cluster, kal_velocity, yield_spread, yc_inverted |
| `portfolio_train` / `portfolio_test` | tibble | Daily port_ret_gross, port_ret_net, n_positions |
| `train_combined` / `test_combined` | tibble | Strategy + SPY returns merged |

---

## R Package Dependencies

```r
# Install if missing:
install.packages(c(
  "tidyquant", "xts", "PerformanceAnalytics",
  "dlm",
  "doParallel", "foreach",
  "plotly", "ggplot2", "scales",
  "dplyr", "tidyr", "purrr", "lubridate", "tibble", "zoo",
  "knitr", "kableExtra"
))
```

**Removed from original**: `TTR`, `tidymodels`, `tidyclust`, `quantmod`, `patchwork`
(k-means now uses base R `kmeans()`; no recipe/workflow infrastructure needed)

Minimum versions:
- Quarto >= 1.3 (native Mermaid rendering)

---

## Common Pitfalls Documented in the Paper

Nine explicit `callout-warning` boxes in Section 5.7:
1. Survivorship bias
2. Look-ahead bias
3. Data mining / parameter snooping (2 params: `k` and `Q`; 16 combos)
4. Storytelling / narrative fitting
5. Transaction costs
6. Adjusted price look-ahead
7. Short selling constraints
8. Outliers (Kalman filter absorbs outliers before clustering; no winsorization)
9. Backtests ≠ scientific experiments

---

## Caching Notes

Add `#| cache: true` to slow chunks for faster re-renders:
- `data_download` — Yahoo Finance + FRED API calls
- `apply_kalman_train` — Kalman on 11 × ~6,500 days
- `parallel_optimization` — 16-combination parallel grid
- `refit_optimal` — full pipeline with optimal params
- `build_test_features` — Kalman on full train+test history

Clear cache with: `quarto render momentum_sector_rotation.qmd --cache-refresh`

---

## Rendering Checklist

Before submission, verify:
- [ ] `quarto render momentum_sector_rotation.qmd` completes without errors
- [ ] Both Mermaid diagrams render in browser (Section 3 and Section 5.4)
- [ ] 3D Omega surface is interactive (Section 7) — axes: k (clusters) × log10(Q)
- [ ] Velocity animation renders with "Bull Cluster (LONG)" legend (Section 8)
- [ ] `charts.PerformanceSummary()` renders (Section 9)
- [ ] HTML opens offline (confirms `embed-resources: true` works)
- [ ] Training and Testing metrics tables both present (Tables 4 and 5)
- [ ] Gross vs net returns shown in Figure 4
