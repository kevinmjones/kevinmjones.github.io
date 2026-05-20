---
title: "Building KLAX26: a leakage-safe weather market research system"
description: >-
  KLAX26 models Kalshi's recurring LAX high-temperature market from the settlement source up: NWS CLI labels, provenance-aware weather data, calibrated probability buckets, synthetic strategy tests, and a paper-only trading path.
pubDate: 2026-05-20
tags: [trading, weather, kalshi, forecasting, agents]
---

I spent a day building a research system for one very specific market: Kalshi's recurring **KXHIGHLAX** contract, which asks what the high temperature at Los Angeles International Airport will be.

That sounds like a weather project. It is not, really. It is a settlement project.

The contract does not settle from Weather Underground, the airport display board, the current METAR screen, or the highest one-minute observation I can scrape. It settles from the **NWS Daily Climate Report for KLAX**. Everything else is an input, a cross-check, or a temptation to leak future information into a backtest.

So KLAX26 starts with one rule:

> The NWS CLI LAX report is the label. If another source disagrees, the other source is evidence, not truth.

That rule shaped the whole system.

## The market

KXHIGHLAX is a daily bucket market. For a given date, Kalshi lists contracts like "68 or below," "69 to 70," "71 to 72," and so on. One bucket pays $1. The rest pay $0.

The naive version of the project would be: forecast tomorrow's high, pick the closest bucket, trade if the price looks wrong.

The useful version is: model the full probability distribution over buckets, compare that distribution to market prices, and refuse to trade when the settlement mechanics are too close to a boundary.

That last clause matters. If I buy a YES position on the 69–70 bucket, the difference between 70°F and 71°F is not small. It is the entire contract.

## Build from the tape backward

The first useful artifact was not a model. It was a data lake with provenance.

KLAX26 currently tracks:

- NWS CLI LAX reports: 6,384 rows across about 9.4 years
- Kalshi market metadata: 1,934 markets
- Kalshi trades: more than 142,000 rows, with a parser bug currently blocking use of the price field
- ASOS one-minute observations from IEM: 305,899 rows
- MADIS HFMETAR observations: backfill in progress
- NWP forecasts through Herbie/NBM/GFS paths: early backfill in progress

Every row is supposed to carry the boring fields that keep research honest: `source_url`, `retrieved_at`, `available_at`, and `raw_payload_hash`.

The most important one is `available_at`. Every feature takes a `decision_time` and can only read rows where `available_at <= decision_time`. There is a dedicated leakage test for this. It is not optional. If a feature accidentally sees a report that was published after the simulated trade time, the backtest is fake.

That discipline caught several design shortcuts before they could become impressive-looking charts.

## The label audit paid for itself

The settlement parser now agrees with Kalshi's populated `expiration_value` field on 260 of 261 settled events: 99.6% agreement.

The one mismatch was 2025-07-04. Every archived CLILAX version I checked reported a maximum of 74°F. Kalshi's analytics field carried 73°F. The relevant bucket still covered both values, so this did not change a contract outcome, but it proved the point: Kalshi's analytics field is a cross-check, not the source of truth.

There was also a less dramatic but more operationally annoying discovery: a 62-day window from 2025-10-31 through 2026-01-12 where Kalshi's `expiration_value` field is blank in the raw JSON. The CLI parser makes that survivable. Without it, a label audit would quietly inherit a vendor-side data hole.

## The model got simpler

The first forecaster ladder had four ideas:

1. Climatology: what does LAX usually do around this day of year?
2. Recency weighting: count recent years more heavily.
3. Residual correction: learn predictable bias in the climatology.
4. Isotonic calibration: map raw probabilities back to observed frequencies.

The latest ablation removed one of them.

Recency weighting helped when it stood alone. But once the residual model and isotonic calibrator were on top, recency added variance more than signal. Across four matched on/off tests, it improved mean Brier by only 0.29%, below the threshold I set for keeping a layer.

So the current production stack is the simpler one:

```text
IsotonicCalibrator(
  ResidualModel(
    Climatology(v2-flat)
  )
)
```

That stack scored Brier **0.7286** across 322 contract dates, versus **0.9890** for the raw climatology baseline used in the report comparison. The residual model is doing the heavy lifting. Calibration is doing the trust work: expected calibration error fell roughly 5x in the report run, from 0.0444 to 0.0085.

The practical translation: when the model says 70%, I want that to mean something close to 70%, not "high, probably." Trading off a miscalibrated probability is just numerology with a price chart.

## The risk is at the bucket edge

This market has a specific failure mode: being right about the weather but wrong about the settlement bucket.

KLAX ASOS observations and the final CLI high are related but not identical. The station samples continuously, the climate report uses its own storage and reporting path, and public feeds can round or lag in ways that matter at one-degree boundaries.

The system therefore carries an M1 boundary safeguard. It computes the distance from the current observed max to the nearest bucket edge:

- inside 0.5 × δ: refuse the trade
- between 0.5 × δ and δ: downsize
- outside δ: allow the strategy to proceed

The current operational δ is **1.5°F**. The empirical residual from the available data is closer to 1.0°F, but the IEM one-minute stream is whole-degree Fahrenheit. That makes the empirical number a lower bound, not a license to be precise. Until MADIS tenths-Celsius coverage proves otherwise, the floor stays.

A real paper-trade briefing artifact showed the safeguard doing exactly what it should. The model had a sharp distribution around 69–72°F, but the observed max sat on a strike edge. The top recommendation was refused; the next two were downsized. That is not a missed trade. That is the system declining to pretend a one-degree cliff is smooth.

## Strategies are mostly hypotheses

KLAX26 currently has nine strategy families sketched or wired:

- expected-value bucket trades
- convergence trades
- Dutch-book basket checks
- late-day residual trades
- CLI/METAR reconciliation
- forecast latency
- maker-style quoting
- cross-market comparison
- regime overlay

Six fire in the synthetic harness. Three are scaffolds waiting on diagnostics.

The latest parameter sweep was useful mostly because it killed enthusiasm. The Dutch-book strategy was consistently negative across tested parameters. That is not a tuning problem; it needs to be removed or re-derived. Convergence has a cliff: a two-cent parameter change flips mean P&L from negative to positive, which is exactly the kind of fragile behavior that disappears when fees, spreads, and real queues show up.

Three strategies survived the stricter synthetic criterion of positive `mean - std` with coefficient of variation below 1.0: EV bucket, maker-style quoting, and regime overlay. Even there, the caveat is loud: the P&L is synthetic-market P&L. It validates plumbing and sensitivity, not alpha.

## What is real, and what is not yet real

Real:

- Settlement source is identified and independently parsed.
- The Kalshi series ticker is verified.
- The label audit is strong enough to trust the CLI parser over Kalshi's convenience field.
- The data lake has provenance and decision-time gates.
- The forecaster beats a flat six-bucket prior and the tested climatology baseline on Brier.
- Calibration materially improves probability honesty.
- The paper-trader path refuses to run unless `TRADING_MODE=paper`.

Not real yet:

- There is no live trading.
- There is no proven real-market alpha.
- Phase 5 P&L is still synthetic.
- The Kalshi trade parser needs repair before historical trades can support an honest real-tape backtest.
- MADIS and NWP backfills are still in flight.
- Several strategy families are placeholders with diagnostics still missing.

The difference matters. A trading project can look finished long before it has answered the only question that matters: would this have made money against the actual market, after fees, spreads, fills, and mistakes?

KLAX26 is not there. It is closer to being able to ask that question honestly.

## The next useful work

The roadmap is narrow now:

1. Fix the Kalshi trade parser so `yes_price_cents` is usable, then repair existing rows.
2. Replace synthetic order books with real Kalshi trade/order-book replay where available.
3. Finish MADIS and NWP backfills, then re-run forecaster evaluation at multiple decision times.
4. Remove or rewrite Dutch-book logic.
5. Run the paper trader for at least 30 days with realistic fills and daily reconciliation.

Only after that would live trading even be a topic. And even then, it needs explicit approval, exposure caps, and a kill switch. The code is intentionally hostile to accidental live execution.

That is the broader lesson from this project so far: the hard part is not making a model that produces probabilities. The hard part is building a system that knows what it was allowed to know, knows what will actually settle the contract, and refuses to trade when the uncertainty is concentrated exactly where the payout changes discontinuously.

For a one-degree weather market, that is the whole game.
