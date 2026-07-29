# SPEC.md — Frozen Implementation Specifications
*virtue-of-complexity · last updated July 28, 2026*

Two replication specs live here: **GKX (Phase 1)** and **KMZ (Phase 2)**.

---

## KMZ REPLICATION SPEC — Kelly, Malamud & Zhou (2024)

All items below confirmed against the paper's empirical-design section (pp. 42–44).

### DATA
- Source: 15 Goyal–Welch (2008) predictors, monthly, 1926–2020, from Amit Goyal's file (2024 update).
- The exact 15 (paper's mnemonics): **dfy, infl, svar, de, lty, tms, tbl, dfr, dp, dy, ltr, ep, b/m, ntis** (14 predictors) **+ one lag of the market return** (the 15th).
  - Note: `de` (dividend-payout, log D12 − log E12) IS included. Do not confuse with a separate d/e.
- Construction from raw ingredients (log form where applicable):
  - dp = log(D12) − log(Index) (dividend-price ratio)
  - dy = log(D12) − log(lagged Index) (dividend yield)
  - ep = log(E12) − log(Index) (earnings-price ratio)
  - de = log(D12) − log(E12) (dividend-payout ratio)
  - tms = lty − tbl (term spread = long-term yield - T-bill rate)
  - dfy = BAA − AAA (default yield spread)
  - dfr = corpr − ltr (default return spread = corporate bond return - government bond return)
  - svar, b/m, ntis, tbl, lty, ltr = direct columns (stock variance, book-to-market, net equity issuance)
  - infl = direct column, **GWZ date convention** — the price data underlying the inflation stat 
    is treated as time-t information; do NOT add your own extra lag. (Paper: results essentially
    unchanged if infl is excluded entirely — so it's not load-bearing if it fights you.) (inflation)
  - lagged market return = CRSP value-weighted; confirm excess vs. total in the paper's data note.

### STANDARDIZATION (two stages — both backward-looking, no look-ahead)
- **Stage 1 — raw predictors & returns, before RFF:**
  - Predictors: **expanding-window** historical standard deviation (high persistence).
  - Returns: **trailing 12-month** standard deviation (faster-moving vol), using the *uncentered*
    second moment (not demeaned — mean monthly returns too noisy in short windows).
  - Warm-up: require 36 months for initial predictor standardization → usable sample starts 1930.
- **Stage 2 — the RFFs themselves:** after generating features, volatility-standardize the
  training-sample RFFs and the out-of-sample RFF by their std in the *training window*, before regression.

### RANDOM FOURIER FEATURES
- Per draw i: S_i,t = [ sin(γ · ω_i′ G_t) , cos(γ · ω_i′ G_t) ], with ω_i ~ i.i.d. N(0, I₁₅).
- Each draw → a sin/cos PAIR, so P = 10,000 features means 5,000 ω draws.
- Bandwidth γ = 2 (results insensitive to γ).
- G_t = the 15×1 standardized predictor vector at month t.

### REGRESSION & COMPLEXITY GRID
- Ridge / ridgeless regression, **NO INTERCEPT**. 🎯 (Paper excludes it; a constant gets shrunk to
  irrelevance. This is the exact choice **Buncic (2025)** attacks — Phase-3 counter-spec: restore intercept.)
- Complexity: P from 2 to 12,000; ridge shrinkage log₁₀(z) from −3 to 3; complexity c = P/T.
- Training windows: T ∈ {12, 60, 120} months (rolling). T=12 is the headline case.
- Recursive OOS: for each t ∈ {T, …, 1091}, fit on trailing T obs, forecast β̂′S_t,
  timing return = β̂′S_t · R_{t+1}.

### METRICS
- OOS R² = 1 − (OOS forecast-error variance / OOS realized-return variance). 🎯
  ⚠ NOTE: this differs from GKX's zero-forecast denominator below — do not cross-contaminate.
- Sharpe: uses the *centered* standard deviation in the denominator.
- 🎯 Aggregation across random draws is the OTHER thing Buncic attacks — record exactly how you
  average across ω draws so the Phase-3 counter-spec is a clean swap.

### TARGET TO REPRODUCE
- OOS market-timing Sharpe improvement ≈ **0.47/year, t ≈ 3.0**, 1926–2020 — *despite* OOS
  predictive R² being substantially negative for most models. That paradox (negative R², positive
  Sharpe) is the paper's central empirical claim; reproducing it = Phase 2 done.

### ⚠ DECISIONS TO FREEZE
- Exact P grid points to actually run (can't do all 12,000 integers — pick ~40 log-spaced).
- Which T to lead with (suggest T=12 for the headline curve, then confirm with T=60).
- ridge z grid resolution.

### DONE =
- VoC curves (OOS R² and Sharpe vs. P) overlaid on the paper's, qualitative match incl. T=12.
- Discrepancies logged; short write-up posted.

---

## GKX REPLICATION SPEC — Gu, Kelly & Xiu (2020)

### UNIVERSE
- Source: Xiu dataset (authors' own characteristics panel, 1957–2021).
- Filter: top 1,000 by market equity (mvel1) each month.
  ⚠ DECIDE: hard top-1,000, or NYSE-size-breakpoint equivalent? (write down which and why)
- Note: GKX use ~30,000 stocks with no price/share-code filters (paper §2.1); our restriction is the
  scaled-down choice — expect lower R², weaker ML edge (documented, expected).

### SAMPLE
- Full: 1957–2021 as provided.
- ⚠ DECIDE: our window (suggested 1990–2021: modern market, 32 years, faster iteration).

### CHARACTERISTICS (~20 of the 94)
- ⚠ DECIDE the list. Selection rule: top features by importance from GKX Figure 5 / Table 4 so
  results are comparable — roughly: mom12m, mom1m, mom36m, chmom, indmom, mvel1, dolvol, turn,
  retvol, idiovol, beta, maxret, ill, bm, ep, sp, agr, nincr, std_turn, zerotrade.
- Cross-check every name against the Supplemental Material definitions.
- Missing values: cross-sectional median, then rank-transform.
- Transform: each characteristic rank-normalized cross-sectionally into [-1, 1] each month (paper §2.1).
  ⚠ CONFIRM via the AAPL sanity check whether Xiu's file is already ranked or raw — decides whether
  you apply this transform or not.

### MACRO INTERACTIONS
- ⚠ DECIDE: include 8 GW macro predictors × characteristics (full spec), or characteristics-only
  (scaled). Suggested: characteristics-only first; add interactions only if time allows.

### TARGET
- Monthly excess return (RET − RF from French library); lags pre-adjusted per readme — **DO NOT re-lag**.

### CV / SAMPLE-SPLITTING
- GKX scheme: recursive expanding training window + fixed validation window, refit annually
  (18y train / 12y val / rest OOS in the original).
- Ours: same recursive structure scaled, e.g. 12y train / 6y val / rest OOS, refit annually.
- PLUS purged & embargoed splits within validation per López de Prado Ch. 7 (embargo ≥ 1 month).
- ⚠ DECIDE and write down: exact split years before running anything.

### MODELS (3 families)
- Elastic net (linear benchmark): grid over alpha, l1_ratio.
- LightGBM: small depth/leaves/lr/estimators grid — write it here once chosen.
- NN: feed-forward, 2–3 hidden layers (GKX NN2–NN3 spirit: 32-16 / 32-16-8), ReLU, batch norm,
  Adam, early stopping on validation; ensemble over ≥5 seeds (paper does 10).

### METRICS
- OOS R² GKX-style: 1 − Σ(r − r̂)² / Σ r²  ← denominator uses **ZERO forecast**, not historical mean
  (paper eq. 3.15; #1 replication error). ⚠ Differs from KMZ's R² above.
- Decile portfolios: sort on prediction monthly, equal-weighted long-short D10−D1; report
  mean/vol/Sharpe vs. paper Table 7.

### DONE =
- Results table side-by-side with paper Table 1 & Table 7 analogues.
- Half-page note: where numbers differ and why (universe, characteristic subset, window).
- Reproducible run script.
