# Capstone Report — CTR / Engagement Opportunity Scoring

- **Author:** Ayesha Asif
- **Lane:** CTR / Engagement Opportunity Scoring
- **Repo:** https://github.com/ayesha-asif1/flyrank-ml-internship
- **Date:** 28 August 2026

## 0. Abstract

Among 176,738 pages in a March 2026 search performance dataset, this study asks which pages rank well but under-capture the clicks expected for their position — and whether that gap can be scored reliably enough to prioritize a content team's review queue. Using Google Search Console data filtered to reliable rows (36.7% of the raw partition), I built a transparent baseline that compares each page's actual click-through rate to the expected rate for its position bucket, weighted by impression volume to separate real signals from low-traffic flukes. I also trained a Random Forest regressor under a leakage-checked, grouped train/test split to test whether a learned model could out-predict this rule using position, impression volume, and day-of-week features. The honest result: the model only marginally beat a no-signal baseline (R²=0.0051 vs ≈0), showing that these features explain very little of CTR on their own — the missing signal is most likely page title and content quality, which weren't available in this dataset. The output is a ranked, reason-coded action queue — with roughly 35% of the portfolio flagged as high-priority metadata review candidates — intended as decision-support for a content team's triage, not an automated or causal verdict.

## 1. Problem framing

**Unit of analysis:** One page (content_hash_id), aggregated across all of March 2026.

**Output:** A ranked score (confidence_weighted_score) with an accompanying reason code (high_position_low_ctr, mid_position_low_ctr, low_confidence_signal, monitor_only, on_par_or_above).

**Decision this supports:** An SEO strategist or content editor doing weekly/monthly triage needs to know which of hundreds of thousands of pages deserve a metadata or content review first. Rewriting titles/meta descriptions is cheap, but reviewing every page manually is not — the decision this work supports is: where should limited review time go first?

**Action a human takes:** Open a flagged page, verify the reason code manually (check title/meta, check for cannibalization, check page age), and decide whether to rewrite metadata, do a content refresh, or leave it for now.

**Cost of a wrong call:** A false positive (flagging a page that's actually fine — e.g., an informational query where low CTR is expected) wastes editor time. A false negative (missing a real opportunity) leaves free, already-earned visibility uncaptured. Neither is catastrophic, which is why this is framed as decision-support, not automation.

**Why data/ML helps:** With 176,738 unique pages, manual review of every page is infeasible. A transparent rule turns this into a ranked, explainable queue. An ML model was also tested to see if a learned model could meaningfully out-predict this rule — it could not, which is itself a useful, honest finding (Section 6).

## 2. Data safety

**Data used:** FlyRank/internship-warehouse (Hugging Face, gated), table fact_content_daily_performance, March 2026 partition (2026-03-01 to 2026-03-31). Rows filtered to gsc_data_available == True (3,611,061 of 9,841,378 rows, 36.7%), then aggregated to one row per unique page (176,738 pages).

**Columns deliberately excluded:**
- gsc_clicks as a model feature — it is the numerator of the CTR label itself (CTR = clicks/impressions); including it would leak the answer directly.
- gsc_sum_position — redundant with gsc_avg_position and harder to interpret.
- All GA4/session/AI-referral columns (ga4_pageviews, ga4_sessions, sessions_organic/direct/referral/social/paid/ai, ai_chatgpt/perplexity/gemini/copilot/claude/meta/other, scroll_events) — these describe on-page behavior that can only be recorded after a click has already occurred, making them causally downstream of the CTR outcome being predicted.
- client_hash_id, content_hash_id, report_date — used only for grouping, filtering, and identifying rows, never as model inputs (raw IDs don't generalize to new pages/clients).

**Leakage risks considered:** A structural check was run on every candidate feature (is it known before or only after a click happens?), followed by a correlation check as a secondary signal — not the primary test, since a feature can be leaky with a weak correlation (observed directly with gsc_clicks, which had only 0.10 correlation with CTR despite being mathematically part of its formula). A before/after comparison of the model under a naive random split versus a grouped split confirmed the expected direction of leakage risk (naive R²=0.0062 vs grouped R²=0.0051), though the gap was smaller than expected given how weak the overall signal already is.

**Client-identifying details:** None. All client_hash_id and content_hash_id values are pre-hashed by the dataset provider. No client names, domains, URLs, or private queries appear anywhere in this repo's work/ folder.

## 3. Baseline

**The rule:** For each page, expected_ctr = the average CTR of all pages sharing the same position_bucket (top_3, top_10, top_20, beyond_20). opportunity_gap = expected_ctr − actual_ctr. To avoid letting low-traffic flukes dominate the ranking, this gap is weighted by traffic volume: confidence_weighted_score = opportunity_gap × log(1 + total_impressions).

**Why it's a fair comparison:** It uses the exact same data and the same label (CTR) as the ML model, and it establishes what a simple, transparent, position-relative comparison can achieve before adding model complexity.

**Numbers:** Evaluated as a no-signal mean-CTR baseline for direct comparison with the regression model: R² ≈ 0.0000, MAE = 0.005485.

Note: R²/MAE shown here are for the mean-CTR reference point used to evaluate the ML model in Section 5; the rule-based score itself is a ranking, not a regressor, so it is evaluated qualitatively via the reason-code breakdown in Section 7 of the notebook."

## 4. Model / analysis

**Method:** Random Forest Regressor, chosen over Linear Regression and a single Decision Tree because an earlier honest-split test (Week 4) showed both underperforming or overfitting, while Random Forest's averaging across trees reduces overfitting and can capture non-linear position–CTR relationships.

**Exact feature list:** gsc_impressions, log_impressions (log1p transform of impressions, to compress a heavily skewed distribution), gsc_avg_position, day_of_week, position_bucket (one-hot encoded categorical version of position).

**Left out on purpose:** gsc_clicks, gsc_sum_position, and all GA4/session/AI-referral columns (see Section 2 for reasoning).

**Target/proxy definition:** CTR = total_clicks / total_impressions, computed at the page level (summed across March, not averaged as daily ratios, to avoid distortion from low-volume days).

## 5. Evaluation

**Split:** Grouped by content_hash_id (GroupShuffleSplit, 80/20), not time-aware. This was chosen because the task is cross-sectional (comparing pages to each other), not forecasting; the real risk was the same page's multiple daily rows leaking into both train and test, which a grouped split eliminates (confirmed: zero content_hash_id overlap between train and test).

**Metrics, model vs baseline, same split:**

| Approach | R² | MAE |
|---|---|---|
| Mean-CTR baseline | ≈0.0000 | 0.005485 |
| Random Forest | 0.0051 | 0.005436 |

**Error analysis:** The largest individual errors occurred where actual CTR was 1.0 (a single click on a single impression) — a statistical fluke the model cannot learn to predict. Mean absolute error was lowest in the beyond_20 bucket (0.0024) and highest in top_3 (0.0085): low-ranked content has consistently near-zero CTR and is easy to predict, while top-ranked content has much higher CTR variance that these features don't explain.

## 6. Interpretation

**What the model found:** Permutation importance shows gsc_avg_position dominates (importance ≈0.0114), followed by log_impressions (≈0.0033) and gsc_impressions (≈0.0022). day_of_week and position_bucket contribute almost nothing once raw position is available — position_bucket is derived from gsc_avg_position, so it adds no new information once the raw value is already in the model.

**Negative result, stated plainly:** The model only marginally beats a no-signal baseline (R²=0.0051). This means position, impression volume, and day of week together explain roughly 0.5% of why CTR varies across pages. This is a well-understood "no effect" rather than a failure of technique: the likely actual drivers of CTR (title text, meta description quality, query intent, competing search results) simply were not available as features in this dataset.

**Surprise:** The gap between the naive (leaky) split and the honest grouped split was smaller than expected (R²=0.0062 vs 0.0051) — leakage inflates results less when the underlying learnable signal is already this weak.

## 7. Recommendation

**Ranked actions:**
1. Prioritize the 62,586 pages flagged high_position_low_ctr (35.4% of the portfolio) for metadata review first — highest-leverage, lowest-effort fix, since these pages already rank well.
2. Treat the 33,148 monitor_only pages (beyond_20 position) as lower priority — position itself is the likely limiter, not metadata.
3. Hold off on the 33,532 low_confidence_signal pages (median 3 impressions) until more data accumulates.
4. Pair every metadata rewrite with a 30–60 day re-check against a fresh run of this pipeline, to confirm the fix actually worked.

**How an editor would use this tomorrow:** Open the ranked queue (baseline_action_score.csv), start from the top, open each flagged page, verify the reason code manually (check the current title/meta, check for cannibalizing pages, check publish date), then decide whether to rewrite, refresh, or leave it.

**Confidence and limits, stated explicitly:** This flags where to look, not why a page underperforms — a human must investigate each case. No causal claims are made. Findings are observational, based on one month of one dataset, and should not be generalized without re-validation. The rule-based score, not the ML model, is recommended for deployment, since the model's improvement is too small to trust for individual-page predictions while being far less transparent.

## 8. Reproducibility

**Environment:** Google Colab (free tier), Python 3. Key libraries: pandas, numpy, scikit-learn (GroupShuffleSplit, RandomForestRegressor, permutation_importance), huggingface_hub, matplotlib.

**Random seed:** random_state=42 used throughout (train/test split and Random Forest).

**Data access:** Requires a Hugging Face read token for the gated dataset FlyRank/internship-warehouse, stored as a Colab Secret named HF_TOKEN.

**To re-run from a fresh clone:**
1. Add HF_TOKEN as a Colab Secret.
2. Run work/notebooks/w03_data_contract.ipynb top to bottom.
3. Run work/notebooks/w03_feature_leakage_check.ipynb top to bottom.
4. Run work/notebooks/w04_baseline_score.ipynb top to bottom — produces baseline_action_score.csv.
5. Run work/notebooks/w05_model.ipynb top to bottom — produces the Random Forest comparison numbers.
6. Run work/notebooks/w06_validation_audit.ipynb top to bottom — reproduces the naive-vs-grouped split comparison.
7. Run work/notebooks/w07_action_playbook.ipynb top to bottom — produces final exports in work/outputs/.
8. Run work/notebooks/capstone.ipynb top to bottom — regenerates all charts/tables used in this report.

Each notebook is self-contained (reloads and rebuilds data/features independently), since Colab free-tier sessions do not persist variables or files across separate runtime sessions.

**Evaluation transparency:** The grouped train/test split is built and verified (zero content_hash_id overlap) directly in work/notebooks/w05_model.ipynb, and the resulting metrics (R², MAE) are printed in that notebook's output and reproduced in work/outputs/model_vs_baseline_results.csv — so the "evaluated once, on an honest split" claim is checkable from the repo, not taken on faith.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset. [https://flyrank.ai](https://flyrank.ai)
