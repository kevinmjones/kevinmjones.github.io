---
title: "KLAX26 progress: a multi-source running max for the intraday lock"
description: >-
  A follow-up on KLAX26. The intraday lock now feeds on the leakage-gated max across MADIS, NWS, and ASOS one-minute observations instead of a single MADIS-only running max. A bounded isolation backtest shows the model sees the temperature sooner, holds fewer already-passed buckets, and eliminates stale-mode calls. Still paper-only.
pubDate: 2026-06-01
tags: [trading, weather, kalshi, forecasting, agents]
draft: false
---

A while back I wrote up [building KLAX26](/blog/klax26-kalshi-lax-temperature), a leakage-safe research system for Kalshi's recurring **KXHIGHLAX** market on the LAX daily high temperature. The rule that shaped the whole thing still holds:

> The NWS CLI LAX report is the label. If another source disagrees, the other source is evidence, not truth.

This post is a follow-up on one specific change to the intraday path. It is still paper-only. There is no live trading.

The short version: the intraday lock used to feed on a single observation source, and that source lagged. I changed it to feed on the max across every available source, leakage-gated. The model now sees the temperature sooner. A bounded backtest says that helps, but only along the axis I actually changed.

## The bug was a lagging single feed

On 2026-06-01 I watched the live model call sit at 70–71°F while two other feeds had already moved. The ASOS tape showed 72°F. The NWS feed showed 71.6°F. The model was holding buckets the temperature had already passed.

The cause was mundane. The running max fed to the intraday lock was MADIS-only, and MADIS-for-KLAX had stalled at about 70°F. Nothing was broken in the model. It was being fed a stale floor.

That is worse than it sounds, because of how this contract settles. The CLI high can only be greater than or equal to any valid reading taken during the day. So if any leakage-gated source already shows 72°F, the settled high is at least 72°F, and every bucket with a ceiling below 72 is dead. A lagging single feed makes the model keep those dead buckets alive and trade off stale data.

## The fix: max over every source available by decision_time

I added a canonical leakage-safe function, `multi_source_running_max`, in `src/features/best_running_max.py`. The running max at a decision time is now:

```text
MAX over all sources of every reading where
  observation_time <= decision_time
  AND available_at  <= decision_time
```

The sources are MADIS HFMETAR, NWS `api.weather.gov`, and ASOS one-minute. The same `available_at` leakage gate from the original build applies to all of them; a reading published after the simulated decision time is not visible.

There are two guards on top:

- Physical sanity bounds. Readings outside `[18, 115]°F` are dropped before the max.
- A MADIS-first tie-break that preserves tenths-of-a-degree-Celsius precision when two sources report the same Fahrenheit value.

The whole thing is revertible at runtime with no code change. Setting the obs-source env var back to MADIS-only restores the prior behavior, which is what the baseline arm of the backtest uses.

## Why this is safe by construction

This is the part I want to state plainly, because it is the reason I trusted the change before I had numbers.

Taking the max over more valid, leakage-gated readings can only raise the running max toward the true high. The settled CLI value is greater than or equal to every valid reading taken that day, so a higher running max from an additional source is never an overshoot past settlement. And adding sources cannot introduce a more-stale floor than MADIS-only, because the old MADIS value is still in the set being maxed over.

So the change is monotone-safe: it can move the running max up toward the truth, never down below where MADIS-only already had it. The backtest is there to measure how much that helps, not to check whether it is dangerous.

## The backtest is an isolation experiment

I wanted to measure the obs-source change and nothing else, so the harness (`scripts/backtest_obs_sources.py`) is deliberately narrow.

For each settled contract day:

1. Fit the production stack walk-forward on settled dates strictly before that day.
2. Hold the pre-day base PMF **fixed**.
3. Vary **only** the observed running max fed to the intraday lock.

Two arms:

- Baseline: `madis` only.
- Candidate: `madis, nws, asos_1min`.

Then score the post-lock posterior against the settled winning bucket at hourly decision times from 08:00 to 17:00 PT.

Holding the base PMF fixed is the whole point. It means any difference between the arms comes from one thing: when and how high each arm sees the temperature. It is not a full-system claim.

## Results on the extended window

The extended window runs 2026-04-17 to 2026-06-01: 43 scored days, 430 decision points. The numbers below are from the saved eval artifact, baseline vs. candidate.

- Mean multi-class Brier: 0.796 → 0.718 (about −9.8%)
- Mode-bucket hit-rate: 34.2% → 46.3% (+12.1 points)
- Stale-mode events: 77 → 0 (eliminated)
- MADIS lagged the cross-source max at 373 of 430 decision points (87%)

A **stale-mode event** is the failure I saw live: the post-lock mode bucket has a ceiling already below the best-available running max. The headline call is a bucket the day has already passed. The baseline had 77 of these in the window. The candidate has none.

The window was only this long because I imported about five extra weeks of KLAX NWS history from an internal NWS-history importer (the "weatherman" service). Before that, NWS coverage did not stretch back far enough to score this many days.

## Caveats, stated up front

This is a narrow result and I do not want it read as more than it is.

- The base PMF was held fixed to isolate the obs-source effect. A fully intraday-refit base would move both arms together; it would not change the relative comparison, but it does mean these absolute Brier numbers are not the numbers a fully live system would post.
- The improvement comes purely from seeing the temperature sooner and higher. That is real and it matters for this contract, but it is one axis.
- This is still all paper-only research. There is no proven real-market alpha.
- The MADIS-KLAX ingest coverage gap is the actual underlying problem. The multi-source max papers over it by routing around the stall. The gap itself is still worth fixing at the source.

## Adjacent fix in the same push

One small thing shipped alongside this. The Kalshi orderbook parser was storing empty top-of-book. Persistence was still parsing the legacy `orderbook.yes`/`orderbook.no` shape, but Kalshi now returns depth under an `orderbook_fp` key. So the parser saw nothing and the dashboard fell back to trades.

I updated the parser to read `orderbook_fp`, keeping the legacy path as a fallback. The dashboard reads real top-of-book again.

## What's next

The narrow list:

1. Fold NWS into the running max used by the morning briefing and daily paper-trader paths too. Those still run on ASOS plus MADIS only.
2. Investigate the MADIS ingest gap at the source instead of routing around it.
3. Keep moving toward a multi-week paper run with realistic fills.

The lesson is the same one from the first post. The model was fine. The problem was what it was allowed to see, and how late it saw it. For a one-degree market, seeing the temperature one feed sooner is not a detail. It is the difference between holding a live bucket and holding a dead one.
