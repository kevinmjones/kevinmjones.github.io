---
title: "FenixFire: Kevin built a careful trading robot"
description: "A plain-English guide to FenixFire, a paper-only stock-trading system where the clever part suggests trades and a strict risk rulebook decides what is allowed."
pubDate: 2026-06-10
tags: [trading, paper-trading, risk, automation, family-guide]
---

FenixFire is a practice-mode trading system. It studies the stock market each morning, suggests a small number of trades, and lets a strict rulebook decide whether any suggestion is safe enough to place.

The most important design choice is simple:

> The clever part never touches the money.

Right now, it uses pretend money only. Real trading is switched off.

## Three desks, one strict boss

Think of FenixFire as a tiny trading firm with three desks.

### The Analyst suggests

Every morning, the analyst scans for stocks that are unusually busy. Then it watches the first part of the trading day.

If a stock breaks above its early range with conviction, the analyst writes a suggestion:

```text
Buy this.
Here is my exit if I am wrong.
Here is my target if I am right.
```

That is all it can do. It cannot place an order.

### The Risk Boss decides

A separate, unbendable rulebook runs about 20 checks.

It asks questions like:

- Is the bet small enough?
- Is too much money already in play?
- Has the system lost twice in a row today?
- Is it too close to the end of the trading day?
- Does the trade already include its safety exits?

One failed check means automatic no.

The risk boss is intentionally not clever. It is ordinary rule-based code, which is the point. Smart software can be confidently wrong. A dumb, strict rulebook is harder to sweet-talk.

### The Clerk acts

Only after a trade passes the risk boss does the clerk place it with a safety bracket already attached:

- A stop-loss order sells automatically if the trade goes wrong.
- A profit target sells automatically if the trade goes right.
- A daily shutdown sells anything still open before the market closes.

The robot never holds stocks overnight.

## Where information comes from, and where orders go

Market prices come in from data providers and are saved locally. That lets the robot rehearse against past days as well as watch the current day.

Orders have one guarded path:

```text
market data + history
        ↓
analyst suggestion
        ↓
risk boss approval or veto
        ↓
clerk places bracket order
        ↓
practice broker and dashboard
```

Every suggestion, veto, fill, and shutdown is written into a permanent diary. If the program crashes and restarts, it reads the diary first, compares its books to the broker's books, and refuses to trade if anything looks off.

## What a trading day looks like

### Before 9:30 AM: wake up and sanity check

The system reads its diary, reconciles its books, and builds the day's shortlist of unusually busy stocks.

### 9:30 to 9:35 AM: watch, do not touch

The market open is chaotic. The robot observes the first five minutes to learn each stock's opening range.

### 9:35 to 11:00 AM: the hunting window

If a stock breaks out of its opening range on heavy volume, the analyst suggests a trade. The risk boss votes. Approved trades go in with their safety brackets.

Most suggestions get vetoed. A quiet day is normal.

### All day: babysitting

The clerk checks every open position once a minute. It asks whether the stop triggered, the target hit, or anything drifted out of sync with the broker.

### 3:50 to 3:55 PM: pack up

Anything still open gets sold. The robot always sleeps with empty pockets.

### After close: write the diary

The system records the final profit or loss, reconciles again, and updates the dashboard for review.

## Five safety rules that matter

### Practice money only

Real trading is hard-wired off. Turning it on would require Kevin to flip a switch, hold a valid login, and personally write a sign-off file no program is allowed to create.

### Small bets, always

Each trade risks a fixed sliver of the account. Hard caps prevent too much money from being in play at once or piled into one stock. No borrowed money is allowed.

### Every trade wears a bracket

No position exists without an automatic exit below it and above it. Losses are pre-decided and small.

### Silence means no

When human approval is enabled, ideas wait in a dashboard queue with a countdown. If nobody approves in time, the answer is no.

### The big red switch stays big and red

One kill switch cancels every order, sells everything, and halts trading. It stays tripped through restarts until a human investigates and types `RESET`.

## Why the dashboard says $5,205

The practice account started with a pretend $5,000. Over a month of rehearsed trading days, it made and lost small amounts like a bank statement:

- Pretend deposit: $5,000.00
- Wins across nine winning days: +$329.14
- Losses across losing days: -$124.02
- Current practice account: $5,205.12

The "P&L today: -$9.38" tile describes the most recent day. The +$205.12 describes the whole month. Both can be true at once.

The honest small print: 24 practice trades is far too few to prove the strategy works. The system stamps reports with that warning and refuses to consider real money until there is much more evidence.

## Questions parents ask

### Is it using real money right now?

No. Every trade is simulated with practice money. The real-money pathway exists only as a locked door.

### Could the AI go rogue and bet everything?

No. The AI part can only write suggestions. The component that can place orders is the rulebook checker, which rejects anything oversized, late, unprotected, or repeated after losses.

### What happens if the computer crashes mid-trade?

Everything important is written to disk when it happens. On restart, the robot reads its diary, compares its records with the broker's, adopts any orphaned positions with a protective stop, and halts if the books do not match.

### Is it making money?

In practice mode, it turned $5,000 into $5,205 over the sample described above. That is encouraging and statistically meaningless. The next step is months more paper trading plus historical testing.

### What's with the name?

A phoenix rises from the ashes carefully and on a schedule, which is roughly the vibe: ambitious, but wearing five seatbelts.
