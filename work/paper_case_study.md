# Case study: ordering a review queue for a content team

Framing material for the paper's abstract and introduction. The deployed paper URL is still a placeholder (`submission/paper_url.txt`), so this is written to stand on its own. The claim ladder is applied explicitly at the end, because the difference between what this work observed and what it decided is the whole point of the case.

## The real problem

A content team at FlyRank manages thousands of pages across dozens of client sites. An editor reviews a few pages per cycle. That is the binding constraint. Not the data, not the tooling, the editor's calendar.

The team already collects the ingredients. Search Console gives impressions, clicks, average position, and the number of days a page was active. All of it is there. What was missing is an ordering: a defensible answer to the question of which pages deserve a look first.

The absence matters more than it sounds. Without an order, the queue gets built by whatever sort is easiest, and sorting by impressions descending is the easiest sort there is. That approach puts the largest pages at the top whether or not anything is wrong with them, and it systematically buries small pages that are failing. The alternative is an editor's memory, which is a real signal but not one anybody can write down, hand to a colleague, or check against six weeks later.

So the problem is a triage problem. A large population of pages, a small reviewing capacity, and no defensible order. The population here is 151,981 page-level rows across the first half of March 2026, of which 77,400 have enough volume to carry a label.

## Why machine learning fits here

The honest framing is decision-support, not prediction. The model does not forecast what will happen to a page in April. It takes a large population of pages, orders it by a measured relationship to an outcome that has already occurred inside the same month, and hands a person a shorter list to read.

That is a lower bar than forecasting, and it should be stated as a lower bar. A model that ranks a queue well is useful even when it is wrong often, as long as its top of the list is enriched relative to the population it is drawn from. A model that predicts next quarter badly is useless for this job regardless of its AUC.

What machine learning buys over a hand-written rule is coverage and honesty about its own limits. A rule can be written, and a rule is easy to defend, but rules accumulate and nobody audits which ones still fire on anything. Fitting the relationship to the data at least produces coefficients that can be inspected, and it produces error bars, which is what lets you say a pattern is real and another one is not.

The cost is real too. Five features and one decision date means the fitted coefficients describe March 2026 specifically. They are not a law. Any feature added or removed invalidates every coefficient sign, which is why the reason codes shipped with the queue are generated from the fitted values rather than written by hand.

## What the lane actually found

Grouped AUC 0.5902 against 0.5000 for random ranking, and P@50 0.4800 against 0.4080 for random ordering, on `GroupKFold(5)` split by client so that no client appears in both train and test. The model clears random ranking. It clears it by 0.09 AUC, and the five folds range from 0.4666 to 0.6547 with a standard deviation of 0.0646, so one fold is below chance. I describe that as modest and I would describe any larger version of that sentence as unsupported.

The validation split did more work than the model. The same estimator and the same five features score AUC 0.6262 on a random row split. Grouping by client gives back 0.036 of that, which is the model partly recognizing clients rather than recognizing decline. For contrast, adding a single label-derived column reaches AUC 1.0000 and stays there under a client holdout, which is the useful reminder: grouping does not protect against a feature that is derived from the label. Only keeping that column out of the feature set does.

Three measured results, in the order I would defend them.

Low CTR is the signal that held. Split at the median CTR of 0.00135, pages below it decline at 0.5072 and pages above it at 0.3804, against a base rate of 0.4438. `low_ctr_more_decline` is the only one of the five reason codes whose assigned group finishes above the base rate. Its measured lift is 0.0634. Note what this is not: on permutation importance, `f_ctr` scores 0.0602 and `f_active_days` scores 0.0596, with standard deviations of 0.0026 and 0.0015. Those overlap within one SD, so the two are tied rather than ranked, and the defensible claim is that CTR produced the only reason code with a measured lift, not that it is the model's strongest lever.

The age heuristic is backwards. The assumption going in was that older pages decay more, which is a reasonable editorial prior and a common refresh rule. Measured, the oldest quartile declines at 0.3707, the lowest of the four, below the newest quartile at 0.4074 and below the base rate. The pattern is not monotone; it peaks in the second quartile at 0.5458. The `stale_high_traffic` reason code was dropped and assigned to zero pages. A prior that runs backwards in the data is worth reporting precisely because it was plausible before the measurement, and an editor may still choose to refresh a page for reasons this label does not capture. That is a legitimate editorial decision. It is not a measured decline signal, and the write-up should keep those two things separate.

Nothing cleared the bar for an automated action. The best measured lift any pattern achieved over the base rate is 0.0634, on a label whose own base rate is 0.4438. That is not enough evidence to tell an editor to change a page. So no reason code in the shipped queue is assigned a refresh action, and the whole queue is a reading order. `low_activity_concentration` is the clearest case: its coefficient is +0.6803, so more active days associate with more decline, and pages active on 10 or fewer days decline at 0.1251 against 0.4948 for pages active on all 15. It was filed, measured, shown in the figure, and explicitly not converted into an action.

The model also did not beat the rule I wrote by hand in Week 4. That rule scores P@50 0.4360 on the same grouped folds against the model's 0.4800. The 0.0440 difference sits inside the rule's own fold spread of 0.20 to 0.58. I report those as unseparated. A metric whose baseline swings 0.38 between folds cannot support a 0.0440 claim, and publishing the comparison without the spread would misrepresent my own work.

## What this means for the team

A ranked queue with the reason attached, and a written list of things that must never be automated from it.

The reason code is the part that changes the workflow. A score on its own is a number an editor has to trust or ignore, and ignoring is the rational response to an unexplained number. `low_ctr_more_decline` at a 0.5072 measured decline rate tells an editor what to open the page and look for, and the accompanying checklist (last update date, position trend, query intent, topical currency, whether it was already refreshed) tells them what to check once it is open. The queue is 77,400 rows, so the practical artifact is the top few hundred with reasons attached.

The no-go list is as much a deliverable as the queue. Never auto-delete, auto-noindex, or auto-redirect on this score, at an AUC of 0.5902. Never auto-refresh or auto-rewrite, because the output is a reading order. Never act on the 74,581 frame rows that fell below the 100-impression threshold, which the model never saw. Never read the score as a prediction of future decline; it is a within-month ordering. Never attach a client-facing risk or health label to it.

The value is in the process rather than in the score. A ranked queue, a reason code per row, a human review checklist, and honest limits is something an editor can use on Monday. A better AUC is not, because the gap between 0.5902 and 0.63 would not change what a person does with a page.

## Claim ladder applied

The words in this document, mapped to the evidence behind them. This is the section to keep honest under review.

| Claim | Evidence held | Words this work uses |
|---|---|---|
| Grouped AUC 0.5902 clears random ranking at 0.5000 | cross-validated, grouped by client, out-of-fold | "the model ranks above random ordering" |
| Low CTR pages decline more, 0.5072 vs 0.3804 | measured split on 77,400 labeled rows | "low CTR is associated with higher decline in this data" |
| Oldest quartile declines least, 0.3707 | measured quartile split, non-monotonic | "the age prior is not supported by this label" |
| `f_ctr` tied with `f_active_days` | permutation importance, overlapping SDs | "tied within one SD, not ranked" |
| Model and Week-4 rule unseparated at P@50 | same folds, difference inside fold spread | "unseparated, not a win" |
| Best pattern lift is 0.0634 | measured against the 0.4438 base rate | "no pattern clears the bar for an automated action" |
| Queue is a reading order | design decision following from the above | "decision-support: a reading order, not a prediction" |

Banned in this document, and why: "causes", "will improve", "proves", and any phrasing that turns the within-month label into a future forecast. There is no intervention here, no matched design, and no time-aware split. One decision date, one month, one label. The strongest causal verb available would be "associated with", and the paper should use it.

The one prior belief this work reversed is worth keeping in the introduction rather than burying in results. The team expected age to be a decline signal and CTR to be secondary. Measurement gave the opposite ordering on both counts, and the age result ran backwards. Reporting a prior that the data contradicted is more useful to a reader than reporting only the patterns that confirmed the plan.
