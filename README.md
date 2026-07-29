# The Virtue of Complexity in Return Prediction — Replication & Critique

Do heavily overparameterized models genuinely predict market returns, or is the
apparent signal repackaged volatility-timed momentum? This repo replicates the
"virtue of complexity" result of Kelly, Malamud & Zhou (2024), builds on the
Gu–Kelly–Xiu (2020) machine-learning-in-asset-pricing foundation, and stress-tests
the KMZ result against the three published critiques — Nagel (2025), Buncic (2025),
and Cartea, Jin & Shi (2025) — to document which findings survive under which
specifications.

**Status:** in progress (started July 2026). [KMZ replication: in progress ·
GKX: planned · critiques: planned]

## What's here
- `kmz/` — replication of KMZ (2024): random Fourier features of the 15
  Goyal–Welch predictors, ridge/ridgeless regression across the complexity
  spectrum, out-of-sample R² and Sharpe-vs-complexity curves.
- `gkx/` — scaled replication of Gu–Kelly–Xiu (2020) on the authors' published
  characteristics panel.
- `debate/` — each critique re-run as a counter-specification against the KMZ pipeline.
- `note/` — the research note adjudicating the debate.
- `SPEC.md` — frozen implementation specifications drawn directly from the papers.

## Data
Datasets are not committed. To reproduce:
- Goyal–Welch predictors — Amit Goyal's website (2024 update).
- GKX characteristics panel — Dacheng Xiu's website.
See `SPEC.md` for exact sources and construction.