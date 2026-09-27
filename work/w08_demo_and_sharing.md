# Week 8: demo outline and shareable cuts

Three sections. The first is roughly what I would say out loud in five minutes. The second is a post I would actually publish rather than sit on. The third is three sentences for a recruiter email or a first call.

Everything here comes out of the March 2026 Search Console slice. Every number is re-measured in the notebook that produced it, and the split design is named wherever a number appears, because the same model scores very differently depending on how you split it.

## 1. Five-minute demo outline

### The problem I was solving

The content team owns thousands of pages per client. One editor gets through a handful per cycle. The team already had Search Console data: impressions, clicks, average position, active days.

What they did not have was an order. Every queue I had seen was either a spreadsheet sorted by impressions descending, which puts the biggest pages on top whether or not anything is wrong with them, or an editorial judgment call that nobody could write down and nobody could check later.

So the question was narrow. Of the pages this editor has to look at anyway, which ones deserve the look first?

### The method

One decision date, 2026-03-15. March is split in half. Features are computed over 2026-03-01 through 03-15. The outcome is measured over 03-16 through 03-31.

The label is whether a page's impressions fell in the second half. That is a within-month proxy, not a forecast of future decline. I want to be upfront about that, because everything downstream rests on it and it is the weakest assumption in the whole design.

Five features: log impressions, CTR, average position, active days, log age in days. Logistic regression, no scaler, `max_iter=500`.

A page needs at least 100 first-half impressions to earn a label. That leaves 77,400 labeled pages out of a 151,981-row feature frame, so 50.9% of the frame. This is a high-traffic queue. I did not solve the problem for thin pages, and I am not going to pretend the demo covers them.

Validation is `GroupKFold(5)` grouped by `client_hash_id`, so no client sits in both train and test. This mattered more than I expected. The same model on a random row split scores AUC 0.6262. Grouping gave 0.036 of that back, which turns out to be client memorization rather than generalization. One grouped fold lands at 0.4666, below chance. I quote the grouped number everywhere and I would push back on anyone quoting the other one.

### The chart

`work/figures/queue_decline_profile.png` plots decline rate by score decile, with the base rate of 0.4438 drawn as a dashed reference line. Decile 1 sits at 0.1848. Decile 10 sits at 0.5567. The bars are near-flat across the middle, with deciles 5 through 9 all landing between 0.4952 and 0.5532. The ordering does most of its work in the first two deciles and then coasts.

Two honesty notes on this figure, and the notebook prints both of them next to the image. The bars are in-sample, so the figure is a consistency check on the ordering rather than evidence about predictive power. And the gap between 0.1848 and 0.5567 sounds bigger than it is once you put it next to the base rate line. A base rate of 0.4438 means 44% of these pages decline with no model involved at all, so what the model contributes is the distance from that line rather than the raw spread between the bars.

If someone asks how good the ordering actually is, the out-of-fold answer is the top decile declining at 0.5195 against a base rate of 0.4438, a lift of 0.0757. That is the number to use, not the decile 1 to decile 10 spread.

### The honest result

Grouped AUC 0.5902, against 0.5000 for random ranking. Grouped P@50 0.4800, against 0.4080 for random ordering. Both clear their reference point. Neither clears it by much.

Here is the part I have to say out loud rather than bury. The model does not beat my own Week-4 rule. That rule scores P@50 0.4360 on the same grouped folds. The gap is 0.0440, and the rule's per-fold values swing from 0.20 to 0.58. The gap sits inside the comparator's own fold-to-fold noise. I am calling them unseparated, not claiming a win. A 0.0440 difference on a metric whose baseline moves 0.38 between folds is not a result.

Two other things the data did to me, and I would rather report these than have them found later.

The age story is backwards. I started from a reasonable-sounding assumption that old pages decay more. The oldest quartile declines at 0.3707, which is the lowest of the four quartiles, below the newest at 0.4074 and below the base rate of 0.4438. The pattern is not monotone anywhere. It peaks in the second quartile at 0.5458. I killed the `stale_high_traffic` reason code and assigned it to zero pages, because a heuristic that runs backwards in the data is worse than no heuristic at all.

The CTR split is the one that held. Split the labeled set at the median CTR of 0.00135. Below the median, decline runs at 0.5072. Above it, 0.3804. That is a 0.1268 gap on 77,400 rows, and `low_ctr_more_decline` is the only one of the five reason codes whose assigned group finishes above the base rate.

One correction to how I would phrase that, because the demo will get quoted. CTR is not cleanly the top feature. On permutation importance `f_ctr` is 0.0602 and `f_active_days` is 0.0596, with standard deviations of 0.0026 and 0.0015. They overlap inside one SD, so they are tied, and ranking one above the other would be overclaiming. `f_avg_position` at 0.0010 is indistinguishable from zero. The claim I can defend is that low CTR is the only reason code with a measured lift, not that it is the strongest lever in the model.

### The recommendation

Look at low-CTR pages first. Do not automate anything.

Every action in the shipped queue is a look. None of them is a refresh. The best measured lift any of my patterns has over the base rate is 0.0634, and that does not justify telling an editor to change a page. The model is a reading order, and a person still reads the page.

## 2. Social post

Roughly 150 words. One finding, one method sentence, the limits stated up front rather than buried in a reply thread.

> I built a ranked review queue for a content team with thousands of pages and one editor. Here's what it actually showed.
>
> Five features from the first half of March 2026 (impressions, CTR, position, active days, age), logistic regression, validated with GroupKFold by client so no client leaks across the split. Label: did the page lose impressions in the second half.
>
> It clears random ranking, modestly. Grouped AUC 0.5902 vs 0.5000. P@50 0.4800 vs 0.4080. It does not beat the hand-written rule I started from, and I'd rather say that here than have someone find it in the notebook.
>
> What held: split on median CTR and decline goes 0.5072 below vs 0.3804 above. What didn't: my "old pages decay more" hunch. Oldest quartile declines least, at 0.3707.
>
> So it ships as a reading order, not a decision. Nothing automated.
>
> Limits: one month, one decision date, pages with 100+ first-half impressions only, 38 clients.
>
> [link-to-repo]

## 3. Employer-facing three-sentencer

For a recruiter screen or a first call, where the goal is to be accurate in thirty seconds and give them something specific to ask about.

> I built a ranked review queue with per-page reason codes for a content team whose editor could only look at a few pages per cycle, with a leakage audit and a validation split I trust enough to publish the negative results from.
>
> It runs on the FlyRank internship warehouse cache at `month=2026-03`: 151,981 page-level rows in the two-week feature window rolled up from 9,841,378 daily Search Console observations, 38 pseudonymized clients contributing labeled pages, and 77,400 labeled pages after requiring at least 100 first-half impressions.
>
> The model clears random ranking but not by much (grouped AUC 0.5902 vs 0.5000, P@50 0.4800 vs 0.4080) and it does not separate from my own rule baseline at 0.4360, so the honest headline is that low CTR is the only reason code with a measured lift over the 0.4438 base rate and the "old pages decline more" heuristic I started with is measured backwards, with the oldest quartile declining least at 0.3707.

## Numbers used, with the design attached to each

Worth keeping in one place so nobody has to go spelunking, and so a number without its split design gets caught.

| Number | Value | Design it belongs to |
|---|---|---|
| Base rate | 0.4438 | labeled rows only (77,400 of 151,981, 50.9% of frame) |
| Grouped AUC | 0.5902 | `GroupKFold(5)` by `client_hash_id`; folds 0.4666 to 0.6547, SD 0.0646 |
| Grouped P@50 | 0.4800 | ranked within fold, same grouped split |
| Random-ranking P@50 | 0.4080 | same folds, shuffled order |
| Week-4 rule P@50 | 0.4360 | same folds, fits nothing; folds 0.20 to 0.58 |
| Model minus rule | +0.0440 | inside the rule's fold spread, unseparated |
| Random row-split AUC | 0.6262 | contrast only; 0.036 of it is client memorization |
| Out-of-fold top-decile rate | 0.5195 | grouped split; lift +0.0757 over base rate |
| Low-CTR decline rate | 0.5072 | below median CTR of 0.00135 |
| High-CTR decline rate | 0.3804 | at or above median CTR |
| Oldest quartile decline rate | 0.3707 | Q4, n=18,476; pattern peaks in Q2 at 0.5458 |
| Best reason-code lift | +0.0634 | `low_ctr_more_decline`; not enough for a refresh action |

Three things in that table deserve a note before anyone quotes them.

The 0.6262 random row split is not a result, it is a warning. The same estimator on the same features with a seeded shuffle gives 0.5641 on one design and 0.6262 on another purely from row order.

151,981 is a page-level count for the two-week feature window, not a count of daily observations. The March fact table holds 9,841,378 daily rows, and 3,611,061 of them survive the `gsc_data_available` filter. Getting this wrong in a public post would be a small lie with a big denominator attached.

38 is the count of clients contributing labeled pages, not the count of clients in the warehouse. There are 44 in the feature window and 104 in the clients dimension, and they are very unevenly sized, with a median of 485 labeled rows per client and a maximum of 17,610.
