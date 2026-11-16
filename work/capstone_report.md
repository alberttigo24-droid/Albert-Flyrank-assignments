# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Albert Michael Tigo
- **Lane:** Lane 2 — Refresh / Content Opportunity Scoring
- **Repository:** https://github.com/alberttigo24-droid/Albert-Flyrank-assignments
- **Dataset:** FlyRank anonymized starter snapshot, 30,000 content rows
- **Random seed:** 42

## 0. Abstract

This project asks whether observable content and search signals can help an editor prioritize pages for refresh review. It uses the 30,000-row anonymized FlyRank starter snapshot, with one row per pseudonymous content item and 32 pseudonymous clients. A transparent stale-and-visible rule is compared with logistic regression, a decision tree, and a random forest using a client-holdout evaluation where available. The committed pipeline report records a random-forest ROC AUC of 0.750 and Precision@50 of 0.740, compared with 0.627 and 0.240 for the rule baseline; the overall declining-label rate is 0.542. These results are decision-support evidence on a cross-sectional snapshot, not proof that refreshing a page will improve traffic and not a validated forecast of future decline.

## 1. Problem framing

Editors have more pages than they can manually inspect. The unit of analysis is one content item. The system outputs a ranked review queue, a score, a suggested action, confidence band, and reason codes. An editor uses it to decide which pages to inspect first; the system does not publish or rewrite pages automatically.

A false positive costs editorial review time. A false negative may leave a potentially useful page unreviewed. The ranking is therefore a triage aid, not an automated decision.

## 2. Data safety and contract

The starter snapshot contains 30,000 rows × 44 source columns and 32 pseudonymous clients. The target used by the reference pipeline is `is_declining_label`, derived from `trend_direction == "down"`; the data dictionary reports 16,262 positive rows (54.2%).

Pseudonymous `content_id` and `client_id` are identifiers for joining and grouping only, never predictive features. The model feature list excludes `trend_direction` and `trend_pct`, because they directly define or encode the label. It also excludes client and content IDs, provider/model names, and product-decision flags.

**Important temporal limitation:** the available starter data is a single trailing-90-day snapshot. Several model inputs (including 90-day impressions/clicks and activity totals) overlap the 30-day comparison window used to derive the decline label. Therefore this experiment is a contemporaneous classification/ranking exercise, not a clean past-to-future prediction. A genuine forecast requires a feature window ending before a separately held-out future label window, ideally from the daily warehouse facts.

No raw client names, domains, URLs, titles, or search queries are used or published.

## 3. Baseline

The ML-07 baseline is intentionally readable: flag pages at least 180 days since their last update and with at least 500 impressions in the observed 90-day window. Its score is `stale × visible × impressions_90d`; its reason code is `stale_and_visible`, and its action is `review_refresh`.

The saved baseline notebook observed 17 actionable pages among 30,000. For model comparison, the reference training script evaluates baseline scores on the same held-out rows as each model.

## 4. Model and methodology

The reference pipeline prepares features, creates the baseline, trains logistic regression, a depth-limited decision tree, and a random forest, then exports a ranked queue. The random forest uses 200 trees, maximum depth 10, minimum leaf size 25, balanced subsampling, and random seed 42. Categorical variables are one-hot encoded; numeric missing values are handled by the reference feature builder.

The reference model feature list contains observed search, content-age, activity, CTR, position, engagement and scroll signals, plus selected content categories. It excludes the label fields and pseudonymous IDs. The target is a proxy for observed decline in the snapshot, not a verified editorial outcome.

The training script attempts a client-level holdout (about 20% of clients) and falls back to a stratified row split if the client holdout does not contain both target classes. The committed report states that the split used was `client_holdout`. This protects against direct row-level mixing across clients, but does not solve the temporal overlap described above.

## 5. Results

The committed pipeline report records the following comparison. Precision@50 is measured on the held-out test set; the base rate is shown to contextualize it.

| Method | ROC AUC | Average precision | Precision@50 | Recall | F1 |
|---|---:|---:|---:|---:|---:|
| Rule baseline | 0.627 | 0.468 | 0.240 | — | — |
| Decision tree | 0.742 | 0.575 | 0.540 | 0.716 | 0.634 |
| Logistic regression | 0.700 | 0.522 | 0.400 | 0.567 | 0.566 |
| Random forest | 0.750 | 0.618 | 0.740 | 0.744 | 0.640 |

The random forest is selected by Precision@50. Its Precision@50 is 0.198 higher than the baseline (0.740 versus 0.240) on the reported evaluation. The positive-label base rate is 0.542, so the baseline's top-50 precision is below the overall positive rate; the random forest's is above it.

The top reported random-forest features include days with impressions (0.1578), log impressions (0.1282), average position (0.1090), and content age (0.0955). These are plausible activity/visibility signals, but because the snapshot windows overlap the target window, their importance must not be interpreted as evidence of advance warning.

**Reproducibility note:** these numbers are transcribed from the repository's committed `outputs/model_report.md`. The raw `model_results.json` and prediction CSV are generated artifacts and were not present in the repository when this report was prepared. Re-run the pipeline to regenerate them before treating this report as a fresh run.

## 6. Limitations and honest framing

1. The target is a proxy derived from a short-term trend category, not a human-labelled refresh success outcome.
2. The single-snapshot feature windows overlap the label-construction period. This prevents a future-prediction claim.
3. The starter slice has 30,000 rows and 32 pseudonymous clients; results may not generalize to the full warehouse, other clients, or later periods.
4. A client holdout tests transfer to held-out clients within this snapshot, not transfer across time.
5. Feature importance is associative, not causal. It does not show that changing a feature or refreshing a page will improve performance.
6. The queue is generated using model scores plus a baseline score and heuristic reason codes. Confidence bands are operational triage labels, not calibrated probabilities of editorial success.
7. The report's metrics should be regenerated from a clean run; generated predictions and raw data should not be committed.

The correct claim is: **on the reported held-out split of this anonymized snapshot, the random forest ranked the proxy-declining label more effectively at the top 50 than the transparent rule baseline.** It does not establish that refresh actions cause recovery or that the model predicts future Google rankings.

## 7. Ranked recommendations and action playbook

Use the exported `outputs/refresh_queue.csv` as a review queue. Start with the highest-ranked items, then verify context manually.

- `refresh_and_review_ctr`: inspect title/snippet alignment and search intent; check impressions, clicks, CTR and position before editing.
- `refresh_and_review_engagement`: inspect content usefulness, page experience and whether visitors find the answer.
- `refresh`: compare the page with current intent and competing coverage; update only where the editor finds a substantive gap.
- `expand_and_refresh`: consider missing coverage or thin content, but verify the page's purpose before expanding it.
- `monitor`: retain in monitoring rather than spending immediate editorial effort.

For every page, confirm the current URL and content manually in the authorized editorial system. Do not infer page identity from pseudonymous IDs. Do not automatically change titles, publish content, or delete pages based only on the score.

## 8. Reproducibility

From a fresh clone:

```bash
pip install -r requirements.txt
python scripts/run_all.py
```

The reference pipeline writes processed features and predictions under `data/processed/` and reports/queues under `outputs/`. Random seed: 42. The dataset is the anonymized starter CSV already supplied by the repository. Generated datasets and per-page queues are not to be committed. The source report and scripts are committed; regenerate metrics locally before making a new performance claim.

## 9. Acknowledgments and data credit

Built on the FlyRank ML Internship anonymized starter dataset and teaching pipeline. Data and program credit: [FlyRank](https://flyrank.ai).

---

## ML-12 — Five-minute demo outline

1. **0:00–0:40 — Problem:** explain the editorial triage problem and why a ranked queue is useful.
2. **0:40–1:20 — Data and safety:** show the 30,000-row anonymized snapshot; explain pseudonymous IDs and excluded label fields.
3. **1:20–2:10 — Baseline:** explain the 180-day + 500-impression rule and its reason code.
4. **2:10–3:20 — Model and validation:** compare baseline, logistic regression, decision tree and random forest; call out the client-holdout split and temporal-overlap limitation.
5. **3:20–4:20 — Recommendations:** show action labels and explain how an editor reviews a page.
6. **4:20–5:00 — Limitations:** stress that this is a triage aid, not causal evidence or a future forecast.

## Social-post cut

I built a content-refresh prioritization experiment using FlyRank's anonymized search dataset (30,000 pages). A transparent stale-and-visible rule was compared with three classifiers; the saved evaluation report records Random Forest Precision@50 of 0.74 versus 0.24 for the rule baseline. The key lesson was not just model performance: time-window alignment matters, and this single-snapshot experiment is a decision-support ranking exercise—not proof that refreshing content causes traffic recovery. Paper and reproducible code: https://github.com/alberttigo24-droid/Albert-Flyrank-assignments

## Employer-facing summary

I built an explainable content-refresh prioritization pipeline on a 30,000-row anonymized search dataset, including a rule baseline, three classifiers, client-holdout evaluation, reason codes and a ranked editorial review queue. The committed pipeline report records Random Forest ROC AUC 0.750 and Precision@50 0.740, compared with baseline Precision@50 0.240. I also documented the temporal-overlap limitation and framed the output as human-reviewed decision support rather than a future-performance or causal claim.
