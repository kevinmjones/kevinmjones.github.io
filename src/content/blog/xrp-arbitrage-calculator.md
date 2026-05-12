---
title: "Building a browser-only XRP arbitrage scanner"
description: >-
  A read-only TypeScript/Vite calculator for ranking XRP cross-exchange spreads
  by depth-weighted VWAP after taker fees, with live spot books and a first pass
  at funding-rate basis.
pubDate: 2026-05-12
tags: [trading, arbitrage, typescript, crypto, agents]
---

I wanted a quick answer to a narrow question:

> At a given trade size, where is XRP cheapest to buy and most expensive to sell right now?

Most exchange screens do not answer that. They show top-of-book prices. That is fine if you are moving tiny size. It is misleading once you care about $10k or $50k notional, because the trade walks the book. Fees matter too. A 12 bp gross spread can turn into a negative trade once you pay taker fees on both legs.

So I built a small read-only calculator: [xrp-arbitrage](https://kevinmjones.github.io/xrp-arbitrage/).

Code is here: [github.com/kevinmjones/xrp-arbitrage](https://github.com/kevinmjones/xrp-arbitrage).

## What it does

The app polls public REST order books in the browser, normalizes each venue into the same shape, calculates VWAP for a selected notional, then ranks every buy-venue / sell-venue pair by net basis points after taker fees.

The important part is that it does not use the best ask and best bid as the trade price. It walks the book by quote notional.

For a $10,000 buy, the calculator consumes asks until it has spent $10,000. If the first level has $2,500 of capacity and the next level has enough remaining size, it takes the first level fully and the second level partially. Average price is total quote spent divided by XRP filled.

Same idea on the sell side, using bids.

```text
buy_cost_per_xrp  = buy_vwap  * (1 + buy_taker_bps  / 10_000)
sell_recv_per_xrp = sell_vwap * (1 - sell_taker_bps / 10_000)
net_edge_bps      = (sell_recv_per_xrp - buy_cost_per_xrp)
                    / buy_cost_per_xrp * 10_000
```

That one formula is the app.

## Why browser-only

This is a discovery layer, not a trading bot.

No API keys. No balances. No order placement. No backend. Nothing to secure except the code itself. You can open the page, let it poll public endpoints, and see whether there is anything worth investigating.

That constraint kept the build honest. The app is TypeScript, Vite, vanilla DOM, Tailwind, native `fetch`, `AbortController`, and Vitest. No React. No database. No server.

The UI is one control panel and two tables:

- Spot spreads, ranked by net bps
- Funding / basis, showing perp funding rates and mark/index basis where the venue exposes enough data

## The annoying parts

The math is simple. The adapters are not.

Every venue returns a slightly different shape:

- Kraken wraps books under legacy symbols like `XXRPZUSD`.
- Coinbase uses tuple levels with an order count.
- Gemini uses objects instead of tuples.
- Bitfinex combines bids and asks in one array and marks asks with negative size.
- OKX and Bybit wrap useful data a few layers down.
- Binance is clean structurally, but CORS makes direct browser access unreliable.

The useful design decision was to make adapters boring and isolated. Each one exports a parser and a fetcher. Everything downstream consumes this canonical shape:

```ts
interface OrderBook {
  venue: string;
  symbol: string;
  quote: Quote;
  bids: [number, number][];
  asks: [number, number][];
  venueTs: number | null;
  fetchedAt: number;
}
```

Once a venue becomes an `OrderBook`, the rest of the app does not care where it came from.

## Freshness matters

A stale profitable row is worse than no row. The app tracks local fetch time per venue, shows age in seconds, flags books after 2x the polling interval, and hides spread rows after 4x.

Single venue failures do not break the table. If Coinbase fails and Kraken, Bitstamp, and Bitfinex are still fresh, the ranker keeps going with what it has.

That sounds obvious until you watch public exchange APIs and public CORS proxies behave like public exchange APIs and public CORS proxies.

## Agentic build notes

This was also a useful test case for agentic coding. The PRD was specific enough to hand off: file structure, adapter contract, acceptance criteria, build sequence, fixture tests, and explicit non-goals.

The agents did well because the boundaries were tight:

- Implement VWAP first and prove it with fixtures.
- Keep venue quirks inside adapter modules.
- Add one venue at a time.
- Do not add execution.
- Do not add auth.
- Do not add WebSockets.

That last set of constraints mattered. Without it, a simple scanner would have turned into half a trading system before the first table rendered.

## Current state

The page is live at [kevinmjones.github.io/xrp-arbitrage](https://kevinmjones.github.io/xrp-arbitrage/).

It currently includes:

- Tier 1 spot venues: Kraken, Coinbase, Bitstamp, Gemini, Bitfinex
- Global venues behind the app's feature flags/proxy path where needed
- Configurable notional presets: $1k, $10k, $50k
- Fee overrides by venue
- Freshness and stale-row handling
- Funding-rate scan for Binance, OKX, and Bybit XRP perpetuals
- Unit tests for VWAP, ranking, adapters, funding parsers, and UI behavior

The next useful step is not execution. It is better market context: stablecoin basis, venue reliability scoring, more robust index-price handling for OKX basis, and probably a tiny historical buffer so I can tell the difference between a real recurring edge and a one-poll mirage.

For now, it answers the original question quickly. At size, after fees, where is the edge?
