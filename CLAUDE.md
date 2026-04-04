# FIN452 Quantitative Trading Project — CLAUDE.md

## Project Overview

Single-file Quarto project implementing a **Kalman Filter + K-Means Sector Rotation** strategy across S&P 500 Select Sector SPDRs. All analysis lives in one file:

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
| Download buffer | TRAIN_START − 200 days (indicator warm-up) |

---

## Strategy Summary

1. **Kalman Filter** (`dlmFilter`, NOT `dlmSmooth`) applied to each sector's daily adjusted close → extracts `kal_level` and `kal_velocity` (trend direction signal)
2. **Technical indicators** computed via TTR: RSI-14, MACD histogram (Kalman-smoothed → `macd_kal`), Bollinger Band z-score, 1m/3m/6m momentum returns
3. **K-means clustering** via `tidyclust` on 7 normalized features — cluster with highest mean `kal_velocity` = "Bull cluster"
4. **Signal**: sectors in Bull cluster → LONG (equal weight); others → FLAT
5. **Execution**: signal at Close_T → execute at Open_{T+1} → earn Open-to-Close return of T+1
6. **Transaction cost**: 0.05% one-way per unit of weight change
7. **Long-only**: no short positions

---

## Key Design Decisions

### Look-Ahead Bias Safeguards
- Use `dlmFilter()` only, never `dlmSmooth()`
- `step_normalize()` fits on training data only (embedded in workflow; applied to test via `augment()`)
- Rolling returns use `dplyr::lag()` (backward-looking)
- Test set untouched until Section 9
- Parameter optimization on training data exclusively

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
setup                     → constants, packages, tidymodels_prefer()
data_download             → raw_prices via tq_get() [cache: TRUE]
data_quality              → coverage table
data_split                → sector_train/test, spy_train/test [FIREWALL]
helper_functions          → run_kalman(), omega_ratio(), generate_signals(), build_portfolio(), compute_metrics()
compute_indicators_fn     → compute_indicators()
apply_kalman_train        → kalman_train [cache: TRUE]
apply_indicators_train    → features_train, cluster_input_train [cache: TRUE]
recipe_definition         → cluster_recipe
kmeans_default_fit        → km_fit_default (k=3)
cluster_inspection        → centroids, BULL_CLUSTER_DEFAULT
identify_bull_cluster     → BULL_CLUSTER_DEFAULT
cluster_heatmap           → Figure 1
returns_computation       → returns_train, spy_ret_train
default_signals_and_portfolio → portfolio_train_default
param_grid_setup          → param_grid (20 combos)
parallel_optimization     → opt_results [cache: TRUE — slow]
surface_plot_omega        → Figure 2 (plotly 3D)
optimal_params            → OPTIMAL_K, OPTIMAL_Q
zscored_metrics           → Table 3
refit_optimal             → km_fit_opt, BULL_CLUSTER_OPT [cache: TRUE]
train_portfolio_opt       → portfolio_train, returns_train_opt, spy_ret_train, train_combined
train_metrics_table       → Table 4
train_cum_chart           → Figure 3
build_test_features       → cluster_input_test [cache: TRUE]
test_clustering_and_portfolio → test_assignments, test_signals, portfolio_test, test_combined
test_metrics_table        → Table 5
cum_return_chart          → Figure 4
pa_charts                 → Figure 5 (charts.PerformanceSummary)
drawdown_chart            → Figure 6 (chart.Drawdown)
spy_chart                 → Figure 7 (chartSeries + overlays)
monthly_holdings          → Table 6
rolling_correlation       → Figure 8 (plotly)
stress_test_computations  → 3 stress scenarios
stress_metrics_table      → Table 7
stress_cum_chart          → Figure 9
session_info
```

---

## Key Global Objects

| Object | Type | Description |
|---|---|---|
| `SYMBOLS` / `SECTOR_SYMBOLS` | character vec | All 12 / just 11 sectors |
| `FEATURE_COLS` | character vec | 7 clustering features |
| `OPTIMAL_K` / `OPTIMAL_Q` | scalar | Best params from grid search |
| `BULL_CLUSTER_OPT` | factor level | Cluster ID for LONG signal |
| `km_fit_opt` | workflow | Frozen trained model (apply to test) |
| `portfolio_train` / `portfolio_test` | tibble | Daily net returns + n_positions |
| `train_combined` / `test_combined` | tibble | Strategy + SPY returns merged |

---

## R Package Dependencies

```r
# Install if missing:
install.packages(c(
  "tidyquant", "TTR", "xts", "quantmod", "PerformanceAnalytics",
  "dlm", "tidymodels", "tidyclust",
  "doParallel", "foreach",
  "plotly", "ggplot2", "patchwork", "scales",
  "dplyr", "tidyr", "purrr", "lubridate", "tibble", "zoo",
  "knitr", "kableExtra"
))
```

Minimum versions:
- `tidyclust` >= 0.1.2 (k_means() and augment() API)
- Quarto >= 1.3 (native Mermaid rendering)

---

## Common Pitfalls Documented in the Paper

Nine explicit `callout-warning` boxes in Section 5.7:
1. Survivorship bias
2. Look-ahead bias
3. Data mining / parameter snooping
4. Storytelling / narrative fitting
5. Transaction costs
6. Adjusted price look-ahead
7. Short selling constraints
8. Outliers (winsorized at 1st/99th pct for clustering features)
9. Backtests ≠ scientific experiments

---

## Caching Notes

Add `#| cache: true` to slow chunks for faster re-renders:
- `data_download` — Yahoo Finance API calls
- `apply_kalman_train` — Kalman on 11 × 6,500 days
- `apply_indicators_train` — TTR indicators
- `parallel_optimization` — 20-combination parallel grid
- `refit_optimal` — full pipeline with optimal params
- `build_test_features` — Kalman + indicators on full history

Clear cache with: `quarto render momentum_sector_rotation.qmd --cache-refresh`

---

## Rendering Checklist

Before submission, verify:
- [ ] `quarto render momentum_sector_rotation.qmd` completes without errors
- [ ] Both Mermaid diagrams render in browser (Section 3 and Section 5.3)
- [ ] 3D Omega surface is interactive (Section 7)
- [ ] `charts.PerformanceSummary()` renders (Section 9)
- [ ] `chartSeries()` with overlays renders (Section 9)
- [ ] HTML opens offline (confirms `embed-resources: true` works)
- [ ] Training and Testing metrics tables both present
- [ ] Gross vs net returns shown in Figure 4
