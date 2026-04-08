# Presentation Script — Momentum Sector Rotation Strategy
### FIN452 Quantitative Trading Project

---

## Opening

"My strategy is momentum-driven sector rotation across the eleven S&P 500 Select Sector SPDR ETFs, benchmarked against SPY. The core idea: sectors trending strongly today tend to keep trending. Every day I measure each sector's momentum, group them into regimes, and go long on only the strongest-trending cluster. Three components make this work — a Kalman filter, k-means clustering, and a yield curve risk-off switch."

---

## Section: Strategy — Description

"The strategy runs daily. A Kalman filter extracts a noise-filtered trend velocity from each sector's price series. K-means clusters those velocities — the cluster with the highest average velocity is the Bull cluster and gets a LONG signal. Everything else sits flat."

"When the FRED 10Y-2Y spread drops below −0.25%, the yield curve override fires and the entire portfolio goes to cash. No cluster signal runs, no positions taken."

"Execution: signal at day T's close → executed at day T+1's open. Long-only, equal-weighted among Bull cluster sectors."

---

## Section: Strategy — Rationale

"Each component solves a specific problem. The Kalman filter handles noise — raw daily returns get dominated by single-day shocks that don't represent real trend changes. Without denoising, sectors bounce in and out of the Bull cluster on noise, not signal, and turnover spikes."

"K-means handles regime identification. Rather than ranking sectors and picking the top N with a fixed threshold, k-means finds natural breakpoints in the velocity distribution that shift dynamically as market conditions change."

"The yield curve switch avoids a second forecasting problem. Instead of trying to predict which sectors hold up in a recession — which is harder than the momentum signal itself — I simply exit the market entirely when recession probability is elevated."

---

## Section: Strategy — Research

"All three design choices are grounded in peer-reviewed work. Moskowitz and Grinblatt (1999) showed sector-level momentum is statistically significant and concentrated on the long side — supporting long-only implementation. Faber (2010) tested relative-strength sector rotation back to the 1920s and found it profitable across all lookback windows. Estrella and Mishkin (1998) documented the yield curve's near-perfect historical track record as a leading recession indicator."

---

## Section: Model Implementation — Data (Table 1)

*[Point to Table 1 on screen]*

"I pull daily OHLC data from Yahoo Finance for all twelve symbols and the FRED T10Y2Y series. Table 1 shows coverage. Two sectors have shorter histories — XLC launched June 2018, XLRE October 2015. Before those dates they're simply absent from the clustering universe and k-means runs on the available subset."

"Training is June 2000 through December 2024 — 24.5 years covering the dot-com bust, the GFC, COVID, and the 2022 rate cycle. Test is January 2025 through March 2026, fully sealed during development."

---

## Section: Model Implementation — Indicators

"The Kalman filter is a local level-plus-slope state-space model tracking two hidden states per sector: the price level and the velocity — the rate of change of that level. Every new price updates both estimates proportional to how surprising it was relative to the prior."

"The key parameter is Q₂, the velocity process noise. Small Q₂ = slow-reacting, long-memory velocity. Large Q₂ = fast-reacting. This gets optimized jointly with k."

"Critically, I use `dlmFilter` — the forward-only causal pass — not `dlmSmooth`, which uses future data to refine past estimates. That would be look-ahead bias."

---

## Section: Model Implementation — Signals (Figure 1)

*[Point to Figure 1 — heatmap]*

"Figure 1 shows how frequently each sector was selected into the Bull cluster by year under default parameters. Warmer blue = selected more often. You can see Tech dominating in bull years, Utilities and Staples appearing more during stress periods — the signal is picking up economically interpretable patterns, not random noise."

"Below that is the decision flowchart — shows exactly how a daily price becomes a position: Kalman filter → yield curve check → k-means → execute at next open."

---

## Section: Model Implementation — Trades

"Return type depends on what the position was doing that day. Entry days — FLAT to LONG — earn Open-to-Close: bought at open, held to close. Hold days — LONG to LONG — earn Close-to-Close: already held from prior close. Exit days — LONG to FLAT — earn Close-to-Open: sold at next morning's open, capturing only the overnight gap."

"Transaction costs are 0.05% one-way on each unit of weight change, applied as |Δweight| × 0.0005. All results shown gross and net."

---

## Section: Model Implementation — Training Period / Optimization (Figure 2)

*[Point to Figure 2 — 3D Omega surface]*

"Two free parameters: k (number of clusters, 2–6) and Q₂ (velocity noise, 12 log-spaced values from 0.00001 to 0.1). That's 60 combinations evaluated in parallel."

"I optimize for Omega ratio, not cumulative return. Cumulative return would just reward whichever parameters kept me most invested during the 2009–2021 bull market. Omega measures whether the strategy generates more dollar-weighted upside than downside at zero return — it accounts for the full return distribution."

"Figure 2 is the Omega surface across all 60 combos. What you want is a broad plateau, not a sharp isolated spike. A spike means the optimum is fragile and specific to one historical stretch. A plateau means the strategy works similarly across a range of parameter values — that's robustness."

"The optimizer selects the best k and Q₂ from this surface. Those values are frozen and never touched again once the test period begins."

---

## Section: Training Period Results (Table 4 / Figure 3)

*[Point to Table 4 and Figure 3]*

"Table 4 and Figure 3 are in-sample results — the model was fit on this data so these are presented for calibration and transparency, not as a forward-looking forecast."

"Figure 3 shows cumulative returns versus SPY over the full training period. The flat sections in the strategy line are the yield curve inversion periods — the strategy stepped entirely out of the market during those windows. You can see this most clearly around 2006–2007 and 2022–2023."

---

## Section: Performance — Out-of-Sample (Table 5 / Figure 4)

*[Point to Table 5 and Figure 4]*

"This is the actual test. January 2025 through March 2026 — 15 months the model never saw during development."

"The Kalman filter state carries forward naturally from training. I concatenate the full price history and run the filter once sequentially — the test-period velocity estimates are informed by all 24.5 years of training history, with no refitting."

"Figure 4 shows three lines: strategy gross, strategy net, and SPY. The gap between gross and net is the transaction cost drag. Table 5 gives the full metrics breakdown."

"One honest caveat: 15 months is a short window. One dominant regime during this period — and 2025 had a significant tariff-shock drawdown — can move these numbers substantially in either direction. I'll come back to this in the stress tests."

---

## Section: Daily Returns & Drawdown (Figures 5a, 5b)

*[Point to Figure 5a and 5b]*

"Figure 5a is daily returns — green for gains, red for losses. This gives a sense of the volatility profile and whether returns are symmetric or skewed."

"Figure 5b is drawdown versus SPY. The depth and duration of drawdowns relative to the benchmark matters — a strategy that trails SPY in bull markets but with meaningfully shallower drawdowns has real portfolio utility as a complement to passive exposure."

---

## Section: Daily Sector Holdings (Figure 6)

*[Point to Figure 6 — tile chart]*

"Figure 6 is a tile chart of daily holdings across the test period. Each row is a sector, each column is a day, blue means LONG. This is where you can see the rotation actually happening — some sectors held consistently, others cycling in and out as momentum shifts."

"This also shows the yield curve cash periods as horizontal white bands across all sectors simultaneously."

---

## Section: Rolling Correlation (Figure 8)

*[Point to Figure 8]*

"Figure 8 is the 60-day rolling correlation between the strategy and SPY. This matters for portfolio context — low or negative correlation during market stress means the strategy is providing genuine diversification, not just replicating SPY with higher fees. A strategy that drops in lockstep with SPY is adding no value beyond the index."

---

## Section: Stress Tests (Table 7 / Figures 9a–c)

*[Point to Table 7 and Figures 9a, 9b, 9c]*

"Three stress tests probe whether the results are fragile."

"First — double transaction costs to 0.10% one-way on the test period. If the edge collapses under higher friction, the strategy is marginal. Figure 9a shows the comparison."

"Second — exclude the GFC (2008–2009) from the training backtest. If training-period performance depends on that one episode, it's a crisis-alpha story not a general momentum story. Figure 9b."

"Third — exclude COVID (February–June 2020) from training for the same reason. Figure 9c."

"Table 7 summarizes all five scenarios side by side — training baseline, test baseline, 2× TC, exclude GFC, exclude COVID. If the metrics hold up across all five, the strategy is demonstrating real robustness."

---

## Section: Learnings — Summary

"To summarize the model: Kalman filter extracts noise-filtered trend velocity, k-means clusters sectors into momentum regimes daily, yield curve exits the market during inversion. Two parameters optimized on training data via Omega ratio grid search, frozen for the test period. No refitting, no look-ahead."

---

## Section: Learnings — Limitations

"The yield curve threshold is fixed at −0.25% and the inversion-to-recession lag historically ranges from 6 to 18 months — the switch may fire too early or linger too long relative to actual market drawdowns."

"K-means draws hard cluster boundaries. Sectors near the boundary can flip on small velocity changes, generating unnecessary trades. Gaussian Mixture Models with probabilistic membership would reduce this."

"Parameters are static — optimized once, never updated. Walk-forward re-optimization on rolling windows would be more realistic for a live implementation."

"And the test window is 15 months. That is not enough to make strong statistical claims. Multiple full market cycles are needed."

---

## Section: Learnings — Future Work

"Five extensions I'd prioritize."

"Walk-forward backtesting — re-optimize k and Q₂ quarterly rather than once."

"Soft clustering — Gaussian Mixture Models with position sizing by Bull cluster membership probability."

"Realized volatility as a second clustering feature — right now Q₂ handles temporal noise in the velocity estimate, but the observation noise R is fixed at 1.0 across all sectors, so the model applies uniform noise assumptions to XLU and XLE equally despite their very different return dispersions. Adding 20-day realized vol as a second feature would let the cluster geometry separate clean uptrends from erratic ones with similar mean velocity. This would require z-scoring both features before clustering, and the optimal Q₂ would likely shift upward since some noise-filtering responsibility transfers explicitly to the volatility dimension."

"Additional macro filters — credit spreads, ISM PMI, or Fed funds futures alongside the yield curve."

"Volatility scaling — size positions inversely to realized volatility as in Moskowitz, Ooi & Pedersen (2012) to stabilize portfolio risk across regimes."

---

## Closing

"The strategy is grounded in published sector momentum research, implemented with proper look-ahead controls, and evaluated on a fully out-of-sample test period. The limitations are real and acknowledged — short test window, fixed parameters, hard cluster assignments. The future work section points to specific, tractable extensions rather than vague improvements."

---

*Script end. Each section header matches the document structure — scroll through the HTML in order as you speak.*
