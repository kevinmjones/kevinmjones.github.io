---
title: "What Kevin's weather computer actually does"
description: "A plain-English explanation of the KLAX26 weather-market system: how it forecasts LAX high-temperature buckets, compares them to Kalshi prices, and stays in paper mode."
pubDate: 2026-06-10
tags: [weather, kalshi, forecasting, paper-trading, family-guide]
---

KLAX26 is a weather-market research system. It watches Los Angeles airport weather, predicts the official daily high-temperature bucket, compares that prediction to market prices, and paper-trades only when the numbers look meaningfully different.

No real money is at risk.

## The game it plays

Kalshi has markets where people can trade yes-or-no contracts on real events. One recurring question is:

> How hot will it get at Los Angeles International Airport today?

The possible answers are split into boxes, such as:

- 70°F or less
- 71°F to 72°F
- 73°F to 74°F
- 75°F to 76°F
- 77°F or more

Each box has a price between 1 cent and 99 cents. If you buy a box for 25 cents and the official temperature lands in that box, it pays $1. If it misses, you lose the 25 cents.

At the end of the day, the National Weather Service publishes the official high temperature for the airport. That official number decides which box won.

## How a 25-cent box becomes a dollar

The math is simple:

```text
Buy the 73°F to 74°F box for $0.25.
If the official high is 74°F, the contract pays $1.00.
If the official high is outside the box, the contract pays $0.00.
```

The hard part is not the payout math. The hard part is estimating the probability of each box better than the crowd does.

## So what did Kevin build?

A computer program that tries to predict the official number and spot moments when the crowd's prices look wrong. It works in four steps.

### It watches the weather constantly

Every few minutes, it collects airport weather readings: temperature, clouds, wind, and official forecasts. It also uses years of Los Angeles weather history.

### It makes its own forecast

The system does not produce just one number. It estimates odds for every box:

```text
70°F or less: 10%
71°F to 72°F: 40%
73°F to 74°F: 30%
75°F to 76°F: 15%
77°F or more: 5%
```

Los Angeles is tricky because morning ocean fog can decide the whole day. If the marine layer burns off early, the airport can warm quickly. If it lingers, the high can stall several degrees lower.

### It compares its odds to the prices

If the program thinks a box has a 40% chance of winning and the market sells it for 20 cents, that may be a bargain.

If the prices already match the forecast, it does nothing. Most of the time, doing nothing is the right move.

### It grades its own homework

Every evening, it checks the official number, scores the forecast, writes a report card, and studies what went wrong. When it finds a specific weakness, Kevin fixes that weakness and tests the change against history.

## The whole machine on one napkin

The system flow looks like this:

```text
airport thermometer
+ government forecasts
+ years of history
        ↓
forecast brain
        ↓
odds for every temperature box
        ↓
compare against market prices
        ↓
paper bet or pass
        ↓
nightly report card
```

That feedback loop is the project. The machine is useful only if it is honest about its misses.

## Why the fog matters so much

Same city, same season, six degrees apart: that can be decided almost entirely by whether the ocean fog burns off before lunch.

That one question moves more money on the market than almost anything else. The program watches morning cloud reports the way a farmer watches the sky.

## A real example from this week

One clear morning, the program predicted a cooler day than the crowd. Its favorite box was 71°F to 72°F, while the market leaned toward 75°F to 76°F.

The temperature ended up in the middle. The miss exposed two real problems:

- Some weather readings were arriving about 40 minutes late.
- On fog-free mornings, the system guessed about a degree too cool because it leaned too hard on a recent cool stretch.

Both problems got fixed the same day. It now receives readings within minutes, and it uses the government's blended forecast as the starting point on days that break the recent pattern.

## What a bargain looks like

For each box, the system compares two numbers:

- What the program thinks the box is worth.
- What people are charging for the box.

A box is interesting only when the program's estimate is meaningfully higher than the market price.

Example:

```text
Box: 71°F to 72°F
Program estimate: 40 cents of value
Market price: 20 cents
Decision: possible paper trade
```

Most boxes, most days, are priced about right. On those days the system passes. A day with zero bets is counted as a good day, not a boring one.

## The part that matters: it is play money

The system trades with pretend dollars, which traders call paper trading. It keeps an honest ledger of every imaginary bet, win, and loss exactly as if it were real.

The program is built so it cannot place a real-money trade. That mode does not exist in the code on purpose.

The idea is simple: prove it works on paper first, with months of honest records. Only with that proof would real money, in small and strictly limited amounts, ever be considered.

## Questions you might be wondering

### Is this gambling?

It is closer to insurance or farming decisions than a casino. In a casino, the odds are fixed against you. Here, someone who genuinely predicts weather better than the crowd can have an edge. Kalshi is a legal, U.S.-regulated exchange.

Right now, though, it is all pretend money.

### Couldn't you just read the weather forecast like everyone else?

Everyone does. That is the point. To beat the crowd, you need something extra: faster data, better math on the fog, and the discipline to act only when the numbers genuinely disagree with the price.

Computers are good at that kind of patience.

### What does Kevin actually do all day?

He is the mechanic, not the driver. The system runs itself. Kevin studies its report cards, finds blind spots, and upgrades it, often by directing AI assistants that test improvements against years of history before anything goes live.

### What if it is wrong?

It is wrong often. Every forecaster is. The goal is not to be right every day. The goal is to be slightly less wrong than the crowd over hundreds of days.

The system also has safety rules. If its data feeds go quiet or its confidence is low, it refuses to bet at all.
