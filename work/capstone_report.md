# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Sukuna310
- **Lane:** Refresh / Content Opportunity Scoring (ranked review queue, decision-support only)
- **Repo:** https://github.com/Sukuna310/MY-Flyrank-ml-internship-starter
- **Deployed paper:** https://sukuna310.github.io/MY-Flyrank-ml-internship-starter/
- **Date:** November 2026 (analysis window: March 2026)

> This report mirrors the deployed paper and the executed notebook
> `work/notebooks/capstone.ipynb`. Every stable number here was re-measured in that notebook
> from the local warehouse cache and checked against the paper (receipts cell); the two
> comparator P@50s and numbers derived from them are disclosed as version-sensitive in §8.

## 1. Problem framing

**Decision:** an editor opens a queue on the morning of 2026-03-15 and asks *"which pages do I
look at first, and why?"* Unit of analysis: one pseudonymized page (client × content) in the
15-day feature window. Output: a ranked reading order with a per-page reason code and a
look-only action. The human action is a review, never an automated change.

**Cost of a wrong call:** a wasted editor-hour (the binding constraint is one editor, a few
reviews per cycle, 77,400 labeled pages). That is why nothing destructive is automated from
the score — the best lift among the queue's reason-code patterns over the base rate is +0.0634, far too weak to justify
telling an editor to change a page.

**Why ML at all:** a plain impressions-descending sort puts the largest pages on top whether
or not anything is wrong, and buries small failing pages. Five feature-window signals,
tangled, scored against an observed within-month outcome give a defensible order. The claim
is decision-support, not prediction.

## 2. Data safety

- **Used:** FlyRank internship warehouse, `month=2026-03` (mid-panel month), cached locally at
  `work/outputs/warehouse_cache/` (gitignored). Tables: `fact_content_daily_performance`
  (9,841,378 daily rows → 3,611,061 with `gsc_data_available IS TRUE`), `dim_content`
  (creation date only), `dim_clients` (coverage check only).
- **Excluded, deliberately:** GA4 columns (three-valued availability flag, zero-filled early
  history — a second, messier measurement system); the `_sample` table (the panel's last
  month — developing label logic there would peek at the test window); product flags
  (health/priority scores must not be rebuilt as features); `trend_direction` / `trend_pct`
  (label-derived in the starter data — never features).
- **Leakage controls:** feature window 2026-03-01→03-15 strictly before outcome window
  2026-03-16→03-31; a deliberate leak test (adding the outcome-window sum drives quick AUC to
  1.0000; removing it restores 0.6262) is run and printed in the capstone notebook.
  Pseudonymous hash IDs are used for grouping/joining/splitting only, never as features.
- **Public-safe:** no client names, client domains/URLs, or private queries anywhere in `work/` —
  only pseudonymized hashes, counts, cuts, and rates.

## 3. Baseline

The Week-4 hand rule (`work/notebooks/w04_baseline_score.ipynb`, re-scored in w06/w07 and the
capstone): +2 if position ≤ 5 and CTR below the median of that subset; else +1 if
log-impressions and log-age both ≥ their medians; else +1 if log-impressions ≥ median and CTR
< median. It fits nothing, uses only feature-window columns, and is scored on the **same
grouped folds** as the model: **P@50 = 0.4360** (fold spread 0.20–0.58), base rate 0.4438.
Fair comparison: same rows, same folds, same metric; the rule is a comparator, never a feature.

## 4. Model / analysis

`LogisticRegression(max_iter=500, C=1.0)` — no scaler, the exact w03/w05/w06 estimator. A
linear model's coefficients are readable reason codes; an ensemble would score the same
ranking task without them.

**Feature list (5, unchanged since w03):** `f_log_impressions`, `f_ctr`, `f_avg_position`,
`f_active_days`, `f_log_age_days` — all knowable on the decision date.

**Label (one sentence):** `is_visibility_loss = 1` when a page's second-half March impressions
are below its first-half March impressions, computed only for pages with ≥100 first-half
impressions — an observed within-month proxy, not a future-window forecast. A further 140
threshold-eligible pages had no outcome-window row and are unlabeled (77,540 → 77,400), so the
labeled set conditions on being observed after the decision date — disclosed, not silently dropped.

Frame: 151,981 rows; labeled: 77,400 (50.9%); 38 clients; base rate 0.4438.

## 5. Evaluation

**Split:** `GroupKFold(5)` by `client_hash_id` — no client in both train and test (one client
holds 22.8% of rows, so a random row split would let the model memorize clients). A seeded
random row split (`StratifiedKFold(3, shuffle=True, random_state=0)`) is quoted for contrast
only: AUC 0.6262 random vs **0.5902 grouped** — 0.036 of the random-split number was client
memorisation, not generalisation.

**Same-split comparison (base rate 0.4438):**

| Method | AUC | P@50 |
|---|---|---|
| Random ordering (reference, recorded) | 0.5000 | 0.4080 |
| Week-4 hand rule (recorded) | — | 0.4360 |
| Logistic regression | **0.5902** | **0.4800** |

P@50 is ranked **within each fold** (fold-model scores are not on one scale). In the recorded
run the +0.0440 gap sits inside the rule's 0.20–0.58 fold spread; a fresh run gives 0.4320 /
0.4520, +0.0280 and 0.20–0.60 (§8) → **unseparated, not a win** either way.

**Error analysis:** one grouped fold lands below chance (0.4666; fold SD 0.0646). The ordering
does most of its work in the top two score deciles and then coasts (Figure 1, in-sample
consistency check only). Healthy-looking pages in the queue (position ≤ 5 and CTR ≥ median;
15,736 rows) decline at 0.3479 — *below* base rate — so the queue's head is not hiding winners
and must never trigger an auto-refresh.

## 6. Interpretation

- **Low CTR is the signal that held:** below-median CTR (0.00135) pages decline 0.5072 vs
  0.3804 above — the largest measured gap, and the only reason code whose assigned group
  clears the base rate. Permutation importance ties CTR (0.0602 ± 0.0026) with active days
  (0.0596 ± 0.0015) within one SD — tied, not ranked (w06).
- **The age heuristic ran backwards:** the oldest quartile declines *least* (0.3707), the
  pattern peaks in Q2 (0.5458), and the standardized age coefficient is negative (−0.2028).
  `stale_high_traffic` was dropped — assigned to zero pages. A plausible prior that fails
  measurement is reported, not hidden.
- **The activity signal is the opposite of the prior:** more active days goes with *more*
  decline (coef +0.6803). Pages active ≤10 days decline 0.1251 vs 0.4948 for 15/15-day pages;
  79.2% of labeled pages sit at 15/15, so the cut is absolute (≤10), and the code is
  monitor-only, never a refresh reason.
- **Position is a snapshot, not a trend:** `f_avg_position` permutation importance
  0.0010 ± 0.0017 — indistinguishable from zero. It prompts a look, not a finding.

## 7. Recommendation

Ranked playbook for the editor (full version: paper §6, `w07_action_playbook.ipynb`):

1. **Look at low-CTR pages first** — check query-intent match, not a verdict. *Directional.*
2. **Check position trend by eye** — the feature is a 15-day average, not a slope. *Low.*
3. **Monitor high-volume pages** — standalone lift only +0.0191. *Weak.*
4. **Do not refresh low-activity pages** — they decline least; the negative result is the
   finding. *Measured.*
5. **Do not auto-decide anything from this score** — best lift +0.0634 cannot carry a
   destructive action. *Strong — the number says so.*

**No-go list:** no auto-delete/noindex/redirect; no auto-refresh/rewrite/republish; no action
below the 100-impression threshold; no reading the score as a prediction; no refresh of
healthy-looking pages; no client-facing "risk"/"health" grade. Monitoring triggers (base-rate
band, AUC drift > 1 fold SD, top-decile lift decay) are in w07 §4.

## 8. Reproducibility

```bash
git clone https://github.com/Sukuna310/MY-Flyrank-ml-internship-starter
cd MY-Flyrank-ml-internship-starter
pip install -r requirements.txt
# one-time: request access to FlyRank/internship-warehouse on HuggingFace (gated, instant),
# then export month=2026-03 + the two dims to these EXACT gitignored filenames:
#   work/outputs/warehouse_cache/fact_2026-03.parquet
#   work/outputs/warehouse_cache/dim_content.parquet
#   work/outputs/warehouse_cache/dim_clients.parquet
# (access rules in skills/flyrank/flyrank-data/SKILL.md; token in env/Colab Secret, never a cell)
pip install nbconvert nbformat ipykernel   # notebook execution tooling, not in requirements.txt
jupyter nbconvert --to notebook --execute --inplace work/notebooks/capstone.ipynb
```

- **Seeds:** random-split contrast `StratifiedKFold(3, shuffle=True, random_state=0)`; P@k
  random ranking `np.random.RandomState(0)`; `GroupKFold(5)` is deterministic given the groups.
  The estimator has no `random_state`; the `RANDOM_STATE = 42` / tree / forest values belong
  to the unedited reference pipeline in `scripts/`, not to this capstone.
- **Environment:** dependency floors in `requirements.txt` at repo root (not exact pins);
  executed with pandas 3.0.5, numpy 2.5.2, scikit-learn 1.9.0, duckdb 1.5.5 (Python 3.12),
  printed by the notebook's setup cell. Run from the repo root or `work/notebooks` — Colab
  would need a manual ROOT/CACHE override because `find_root()` relies on the local tree.
- **Reproducibility is measured:** the capstone re-runs the frame SQL and compares
  feature-by-feature. The frame is not bit-for-bit stable (`f_avg_position` is a parallel
  DuckDB AVG; ~4e-14 float noise) — every reported number is orders of magnitude above that
  floor. The receipts cell checks all headline numbers against the paper (tolerance 5e-4).
- **Version sensitivity (honest spread):** every model metric, the base rate, all counts and all
  pattern rates reproduce exactly. The two *comparator* P@50s move by ~0.02 on a fresh run:
  random ordering depends on the seeded permutation, and the three-level hand rule depends on
  `np.argsort` tie-breaking at the top-50 cut. Both comparators stay below the model's 0.4800
  under either measurement, so the conclusion is unchanged — the third decimal of a comparator
  is not a result. This is documented in the notebook's §4 note and the paper's §7.4.
- **Artifacts:** `work/outputs/action_playbook_queue.csv` (77,400 rows, gitignored),
  `work/outputs/capstone_metrics.json`, `work/figures/queue_decline_profile.png`,
  `work/figures/reason_code_breakdown.png`. The reference pipeline in `scripts/` is unedited.

---

**Claims checklist:** observed / measured / directional / decision-support language throughout ·
base rate 0.4438 reported next to every P@50 · no causal claims · no "predicted Google's
algorithm" · no client-identifying details · stable numbers match a fresh re-run; the two comparators are disclosed as version-sensitive (receipts cell).

**Data credit:** Built on the [FlyRank ML Internship dataset](https://flyrank.ai).
