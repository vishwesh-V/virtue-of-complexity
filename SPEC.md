# SPEC.md — Frozen Implementation Specifications
*virtue-of-complexity · last updated September 24, 2026*

Two replication specs live here: **KMZ (Phase 2, COMPLETE)** and **GKX (Phase 1, IN PROGRESS)**.

**Project status:** KMZ replicated · Buncic + Nagel + Cartea critiques done · research note drafted · GKX build underway.

---

## KMZ REPLICATION SPEC — Kelly, Malamud & Zhou (2024)  COMPLETE

All items below confirmed against the paper's empirical-design section (§V.A–C). Page refs are to
the SSRN version (Swiss Finance Institute RP 21-90): printed pp. 39–42 = PDF pp. 41–44; Table I on
printed p. 50. Journal of Finance (2024) equivalent: pp. 487–490, Table I on p. 495.

### DATA
- Source: Goyal–Welch predictors, monthly, from Amit Goyal's file (Data2025.xlsx, "Monthly" tab),
  which runs 1871–2025. Ratios are PRE-COMPUTED simple (non-log) columns — no construction needed.
- The 15 predictors used (file column → clean name): d/p→dp, d/y→dy, e/p→ep, d/e→de, b/m→bm,
  svar, ntis, tbl, lty, ltr, tms, dfy, dfr, infl, + lagged market return (ret.shift(1)) = the 15th.
- NO log transform on predictors. Confirmed against paper p.42: the only transformation is
  volatility-standardization (Stage A); "none of our findings are sensitive to variations in how
  standardizations are implemented."
- infl: use as-is (GWZ date convention, do not add extra lag); not load-bearing per paper.
- Usable sample after dropna: 1926-12 → 2025-12, 1189 rows (predictors + contemporaneous excess_ret).

## Valuation ratios
*Is the market cheap or expensive relative to fundamentals? High values generally imply higher
expected future returns (you're buying in at a low price).*

| Clean name | File column | Description |
|---|---|---|
| `dp` | `d/p` | **Dividend–price ratio** — annual dividends ÷ price. Dividend received per dollar of price. |
| `dy` | `d/y` | **Dividend yield** — dividends ÷ *last month's* price. Nearly identical to dp, just a lagged-price denominator. |
| `ep` | `e/p` | **Earnings–price ratio** — earnings ÷ price. The inverse of the P/E ratio. |
| `de` | `d/e` | **Dividend-payout ratio** — dividends ÷ earnings. Fraction of earnings paid out as dividends (a corporate-behavior signal, not a cheapness signal). |
| `bm` | `b/m` | **Book-to-market** — aggregate book value ÷ market value. The classic value signal. |

## Interest-rate / bond signals
*The rates environment and the shape of the yield curve.*

| Clean name | File column | Description |
|---|---|---|
| `tbl` | `tbl` | **T-bill rate** — the short-term risk-free rate (3-month Treasury). |
| `lty` | `lty` | **Long-term government bond yield.** |
| `ltr` | `ltr` | **Long-term government bond return** — realized return, not the yield. |
| `tms` | `tms` | **Term spread** — lty − tbl, the slope of the yield curve. An inverted curve (negative tms) is a well-known recession predictor. |

## Credit-risk signals
*How nervous the market is about corporate default — a business-cycle "fear gauge."*

| Clean name | File column | Description |
|---|---|---|
| `dfy` | `dfy` | **Default yield spread** — BAA − AAA corporate bond yields. Wider = more default fear. |
| `dfr` | `dfr` | **Default return spread** — corporate bond return − government bond return. |

## Other

| Clean name | File column | Description |
|---|---|---|
| `svar` | `svar` | **Stock variance** — realized variance of recent market returns (a volatility measure). |
| `ntis` | `ntis` | **Net equity issuance** — net stock issued vs. bought back. Heavy issuance has historically predicted *lower* returns (firms issue when their stock is expensive). |
| `infl` | `infl` | **Inflation.** |
| `mktret_lag` | `ret.shift(1)` | **Lagged market return** — last month's market return; captures short-term momentum/reversal. This is the 15th predictor, built by shifting `ret` forward one row. ⚠ See the timing deviation below. |

---

### TARGET
- Monthly log excess return: excess_ret = log(1+ret) − log(1+Rfree).
- ⚠ MINOR DEVIATION: KMZ use the "monthly excess return" (simple), and a timing strategy's
  realized return is simple. Log vs simple differs by roughly R²/2 per month — small at monthly
  frequency; kept as is.
- ⚠ DEVIATION (conservative, kept): row t holds month-t predictors and targets R_{t+1}, but
  `mktret_lag = ret.shift(1)` puts month t−1's return in row t. KMZ's "one lag of the market
  return" (fn. 33) is month t's return, known at the end of month t. Our model therefore sees the
  market return one month later than KMZ's — never look-ahead, but it drops the most recent
  return, likely lowering Sharpe somewhat. The Nagel momentum benchmark uses the same convention
  (trailing window t−12…t−1), so the KMZ-vs-momentum comparison is like for like, but both omit
  month t's return.
- ⚠ DEVIATION: file's `ret` = CRSP-calc S&P500 total return; KMZ specify CRSP value-weighted index.
  Close proxy (Goyal's standard series), documented as a minor deviation.
- Alignment (paper p.44): training pairs (R_t, S_{t-1}); forecast β'S_t predicts R_{t+1}. Built in
  pipeline via excess_ret.shift(-1). Verified by 3-row alignment unit test (PASSED).

### STANDARDIZATION (two stages — both backward-looking, no look-ahead)
Not full z-score: **divide by std, do NOT subtract the mean.**
- **Stage 1 — raw predictors & returns, before RFF:**
  - Predictors: **expanding-window** historical standard deviation (high persistence).
  - Returns: **trailing 12-month** standard deviation (faster-moving vol), using the *uncentered*
    second moment (not demeaned — mean monthly returns too noisy in short windows).
  - ⚠ DEVIATION (resolved, kept): KMZ use the vol-standardized return as both the forecast target
    and the return the timing strategy earns (Table I caption; fn. 44, SSRN p. 49; "Max Loss" in
    standard-deviation units; alphas vs the vol-standardized market). Our pipeline computes
    `target_std` but only uses it in the warm-up dropna (where it removes nothing extra — the
    36-month predictor warm-up is longer); the ridge is **trained on raw next-month excess returns**
    and the timing return uses raw returns. Kept deliberately: KMZ report their conclusions are
    "entirely unaffected" by raw vs standardized returns (fn. 44; Internet Appendix §VI). We rely on
    their robustness check and have not verified it ourselves. Effect: raw returns give
    high-volatility periods (e.g. the 1930s) more weight in both fitting and Sharpe, so our Sharpe
    levels are not directly comparable to KMZ's.
  - All `.shift(1)`-ed for causality (month t uses only data through t−1).
  - Warm-up: predictor standardization needs history to stabilize; the first ~36 standardized
    months are NaN and drop → **effective analysis sample starts ~1930** (matches KMZ). Usable: 1152 rows.
  - Leakage test PASSED (truncation-recompute unchanged).
- **Stage 2 — the RFFs themselves:** after generating features, volatility-standardize the
  training-sample RFFs and the out-of-sample RFF by their std in the *training window*, before regression.

### RANDOM FOURIER FEATURES
- Per draw i: S_i,t = [ sin(γ · ω_i′ G_t) , cos(γ · ω_i′ G_t) ], with ω_i ~ i.i.d. N(0, I₁₅).
- Each draw → a sin/cos PAIR, so P = 10,000 features means 5,000 ω draws.
- Column layout: sin/cos **interleaved** ([sin ω₁, cos ω₁, sin ω₂, cos ω₂, …]) and ω draws generated
  so draw j is independent of the total count. The nested sweep takes the first P columns of one
  P_max matrix, so any first-P slice is exactly P/2 full pairs, identical for any P_max.
  (Fixed Sep 2026: the original [all sines | all cosines] layout gave every model with P ≤ P_max/2
  sine-only features. All results below are from the fixed code; P must be even.)
- Bandwidth γ = 2 (results insensitive to γ).
- G_t = the 15×1 standardized predictor vector at month t.

### REGRESSION & COMPLEXITY GRID
- Ridge estimator (paper p.18): β̂(z) = ( z·I + (1/T)·Σ SₜSₜ' )⁻¹ · (1/T)·Σ SₜRₜ₊₁.
  Penalty is **z·I**; Gram and cross-moment both divided by T.
- Implemented in the **dual form** (solve T×T, not P×P) — exact identity, makes P=12,000 fast.
- Ridge / ridgeless regression, **NO INTERCEPT**. (Paper excludes it; a constant gets shrunk to
  irrelevance. This is the exact choice **Buncic (2025)** attacks — Phase-3 counter-spec: restore intercept.)
- Complexity: P from 2 to 12,000; ridge shrinkage log₁₀(z) from −3 to 3; complexity c = P/T.
- Training windows: T ∈ {12, 60, 120} months (rolling). T=12 is the headline case.
- Recursive OOS: for each t ∈ {T, …, N}, fit on trailing T obs, forecast β̂′S_t,
  timing return = β̂′S_t · R_{t+1}.

### METRICS
- OOS R² (ours) = 1 − Σ(R − f)² / Σ R², i.e. benchmarked against a **zero forecast** — same
  formula as GKX's R² below.
  ⚠ MINOR DEVIATION: KMZ fn. 40 (SSRN p. 42) define R² as 1 − Var(forecast error) / Var(realized
  return), i.e. mean-removed. With forecasts averaging ≈ 0, ours ≈ theirs × Var(R)/(Var(R) + mean²)
  ≈ theirs × 0.99 — same sign, negligible size difference. Sharpe does not use R², so no Sharpe or
  critique result is affected.
- Sharpe: uses the *centered* standard deviation in the denominator; annualized ×√12.
- ⚠ LIMITATION: all Sharpe ratios (pipeline and critiques) are raw, not adjusted for static market
  exposure. KMZ also report alpha / information ratio vs the (vol-standardized) market (Table I:
  IR 0.31, t=2.9), since a mostly-long strategy earns the equity premium. We do not compute this.
- 🎯 Aggregation across random draws is the OTHER thing Buncic attacks — per-draw results are
  stored (kmz_sweep_raw.parquet for the pipeline; buncic_base_raw / buncic_int_raw.parquet for
  the Buncic runs).

### FROZEN CHOICES (resolved)
- P grid actually run: [2, 8, 32, 128, 512, 2000, 6000, 12000] (log-spaced).
- Led with T=12 (headline; T=60/120 not run). z grid: log₁₀(z) ∈ {−3,…,3}.
- Draws: **20 (final for this replication**, all notebooks; KMZ use 1,000).

### RESULT (reproduced)
- Single curve (z=1e-3, 20-draw mean):

  | P | 2 | 8 | 32 | 128 | 512 | 2,000 | 6,000 | 12,000 |
  |---|---|---|---|---|---|---|---|---|
  | c = P/T | 0.17 | 0.67 | 2.7 | 11 | 43 | 167 | 500 | 1000 |
  | Sharpe | −0.025 | 0.062 | 0.016 | 0.044 | 0.076 | 0.096 | 0.125 | 0.147 |
  | OOS R² | −0.188 | −2.179 | −0.660 | −0.126 | −0.041 | −0.023 | −0.017 | −0.015 |

  Sharpe rises steadily from c≈3 to 0.147 at c=1000 (not monotone below c≈3). R² negative
  throughout, trough −2.18 at c≈0.67 (double descent near the interpolation boundary),
  recovering to −0.015.
- Shrinkage family: heavy shrinkage lifts Sharpe; best **0.28** at P=2,000, z=10³. At z=10³ Sharpe
  peaks near c≈167 and eases to 0.24 at c=1000.
- **Paradox reproduced: negative R², positive Sharpe.** Qualitative match = Phase 2 done.
- Like-for-like vs paper (Table I, T=12, c=1000, z=10³): **0.24 vs KMZ 0.47.** Differences:
  (1) raw vs vol-standardized returns, (2) S&P 500 total return vs CRSP value-weighted index,
  (3) 20 vs 1,000 draws, (4) lagged market return at t−1 rather than t; plus sample end 2025 vs
  2020 and log vs simple returns. Best-tuned in our grid: 0.28 (P=2,000, z=10³).
- Caveat: with 20 draws, differences of ~±0.02 Sharpe are within draw noise. Over ~95 years a
  Sharpe of 0.15 has t ≈ 0.15·√95 ≈ 1.4 — not individually significant.

---

## PHASE 3 — CRITIQUE COUNTER-SPECS  COMPLETE
All three reuse the KMZ engine (same data, standardization, seeds `42·100000 + d`, 20 draws,
T=12, z=1e-3). Each notebook's no-intercept / zero-noise baseline reproduces the pipeline exactly.

### Buncic (2025) — `debate/Buncic/buncic.ipynb`
- **Counter-spec 1 (intercept):** ridge with an **unpenalized intercept**, primal solve
  ((P+1)×(P+1)), so the grid is capped at P ≤ 512 (c ≤ 43).

  | P (c) | 2 (0.17) | 8 (0.67) | 32 (2.7) | 128 (11) | 512 (43) |
  |---|---|---|---|---|---|
  | Sharpe, no intercept (KMZ) | −0.025 | 0.062 | 0.016 | 0.044 | 0.076 |
  | Sharpe, intercept | 0.064 | 0.059 | 0.052 | 0.078 | 0.102 |

  Intercept raises Sharpe at 4 of 5 levels (P=8 is a tie within noise), most at low complexity.
  The simplest intercept model (P=2, 0.064) gets ~85% of the no-intercept model at c≈43 — most of
  the low-complexity handicap is the missing intercept. The intercept model still rises with c
  (0.052 → 0.102 from c≈3 to 43). R² is worse with the intercept at every level.
- **Counter-spec 2 (aggregation):** distribution of single-draw Sharpes for the **intercept model**
  (`buncic_int_raw`). 10–90% band ≈ −0.05 to +0.17 at c ≤ 3, narrowing to 0.06–0.19 at c≈43.
- Limitation: intercept not tested above c≈43 (a dual-form intercept via within-window demeaning
  would extend it to P=12,000).
- Limitation: an intercept tilts the average forecast positive, i.e. adds a static long-market
  position. Without an alpha-vs-market regression we cannot say how much of the intercept's Sharpe
  lift is timing vs plain market exposure. Supported claim: "restoring the intercept raises Sharpe
  at low complexity." Not supported: "the complexity benefit is an artifact of the missing intercept."
- Context: KMZ fn. 35 (SSRN p. 40) report that adding a constant has no effect on Table I — but that
  is at c=1000, z=10³ (heavy shrinkage). Our test is z=10⁻³, c ≤ 43, where the intercept matters.
  Different regimes, not a contradiction.

### Nagel (2025) — `debate/Nagel/nagel.ipynb`
- **Counter-spec:** vol-timed momentum, no predictors/RFF: position_t = mean(ret_{t−12..t−1}) /
  uncentered std(ret_{t−12..t−1}), earning the same next-month return as KMZ (1,140 months).
- Momentum Sharpe **0.283** vs KMZ P=12,000: 0.145 (single draw), 0.147 (20-draw mean).
- **Spanning regression** (KMZ single-draw returns on momentum returns, Newey–West 12 lags):
  corr 0.21, R² 0.04, β t=2.21, alpha t=0.77, alpha information ratio 0.088 (vs 0.145).
  Momentum accounts for ~40% of KMZ's Sharpe; the residual is insignificant, but so is KMZ's raw
  Sharpe over this sample, so the test has little power. Single draw, single P.

### Cartea, Jin & Shi (2025) — `debate/Cartea/cartea.ipynb`
- **Counter-spec:** add N(0, σ²) noise to the standardized predictors before building RFFs
  (noise drawn once per draw), σ ∈ {0, 0.25, 0.5, 1.0}; full P grid.

  | P | 2 | 8 | 32 | 128 | 512 | 2,000 | 6,000 | 12,000 |
  |---|---|---|---|---|---|---|---|---|
  | σ = 0 | −0.025 | 0.062 | 0.016 | 0.044 | 0.076 | 0.096 | 0.125 | 0.147 |
  | σ = 0.25 | 0.009 | 0.007 | −0.017 | 0.002 | 0.008 | 0.059 | 0.077 | 0.069 |
  | σ = 0.5 | 0.008 | 0.023 | −0.006 | −0.002 | 0.028 | 0.011 | −0.001 | −0.025 |
  | σ = 1.0 | −0.035 | −0.025 | 0.023 | −0.010 | −0.018 | −0.032 | −0.003 | 0.012 |

  Mild noise (σ=0.25) roughly halves the high-complexity Sharpe but keeps the upward slope above
  c≈43; σ ≥ 0.5 eliminates it (values scatter around zero).

---

## GKX REPLICATION SPEC — Gu, Kelly & Xiu (2020)  🔨 IN PROGRESS

### UNIVERSE (FROZEN)
- Source: Xiu dataset (authors' own characteristics panel, 1957–2021), dachxiu.chicagobooth.edu.
- Filter: **hard top-1,000 by market equity (mvel1) each month.** Rationale: liquid, tractable,
  avoids microcap distortions that inflate paper results.
- Note: GKX use ~30,000 stocks with no price/share-code filters (paper §2.1); our restriction is the
  scaled-down choice — expect lower R², weaker ML edge (documented, expected).

### SAMPLE (FROZEN)
- **1990–2021.** Modern market, 32 years, faster iteration (vs. the full 1957–2021).

### CHARACTERISTICS (FROZEN — ~20 of the 94)
Balanced to mirror where GKX Figure 5 / Table 4 feature importance concentrates
(momentum, liquidity, volatility). **Verify every name against actual datashare columns on load;
substitute nearest equivalent if a name is absent (e.g. retvol↔rvar_mean, idiovol↔rvar_ff3).**

- Momentum/reversal (5): `mom12m, mom1m, mom6m, chmom, indmom`
- Liquidity/frictions (5): `mvel1, dolvol, turn, ill, baspread`
- Risk/volatility (4): `retvol, idiovol, beta, maxret`
- Value (3): `bm, ep, sp`
- Investment/profitability (3): `agr, roaq, ni`

- Missing values: cross-sectional median imputation.
- Transform: each characteristic rank-normalized cross-sectionally into [-1, 1] each month (paper §2.1).
  ⚠ CONFIRM via the AAPL sanity check (permno 14593) whether Xiu's file is already ranked or raw —
  decides whether you apply this transform or not. (readme convention: `ret_t = chars_{t-1}`.)

### MACRO INTERACTIONS (FROZEN)
- **Characteristics-only.** No macro×characteristic interactions in the scaled version.

### TARGET (FROZEN)
- Monthly excess return (RET − RF from Ken French 3-factor library).
- **Lags are pre-adjusted per readme (`ret_t = chars_{t-1}`) — DO NOT re-lag.**

### CV / SAMPLE-SPLITTING (FROZEN)
- Recursive, refit annually: **train 1990–2001 / validate 2002–2007 / OOS 2008–2021.**
- PLUS purged & embargoed splits within validation per López de Prado Ch. 7 (embargo ≥ 1 month).

### MODELS (3 families)
- Elastic net (linear benchmark): grid over alpha, l1_ratio.
- LightGBM: small depth/leaves/lr/estimators grid — write it here once chosen.
- NN: feed-forward, 2–3 hidden layers (GKX NN2–NN3 spirit: 32-16 / 32-16-8), ReLU, batch norm,
  Adam, early stopping on validation; ensemble over ≥5 seeds (paper does 10).

### METRICS
- OOS R² GKX-style: 1 − Σ(r − r̂)² / Σ r²  ← denominator uses **ZERO forecast**, not historical mean
  (paper eq. 3.15; #1 replication error). Same formula as the KMZ R² above.
- Decile portfolios: sort on prediction monthly, equal-weighted long-short D10−D1; report
  mean/vol/Sharpe vs. paper Table 7.

### DATA HANDLING (Colab)
- Download 4GB zip to LOCAL `/content/` (NOT Drive — only ~0.3GB Drive free after KMZ).
- Extract, filter to the 20 chars + 1990–2021 + top-1000, save ONLY the small filtered parquet to Drive.
- Five sanity checks before modeling: zip integrity, row count (tens of millions) + date range
  (1957→2021), stocks-per-month plot, AAPL (permno 14593) value check + raw-vs-ranked determination.

### DONE =
- Results table side-by-side with paper Table 1 & Table 7 analogues.
- Half-page note: where numbers differ and why (universe, characteristic subset, window).
- Reproducible run script.