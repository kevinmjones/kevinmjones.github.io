# Blog visual style guide — KLAX26 report system

Source reference: `/home/kevin/Documents/Projects/klax26/reports/performance_dashboard_2026-05-20.html` plus the v3 executive/interpretive reports.

## Design thesis

The blog should read like a field report, not a plain article: dark analytical canvas, large editorial serif headlines, compact mono metadata, dashboard cards, honest caveats, and visual system diagrams before long prose.

## Core motifs

- **Dark operations room:** near-black page, fixed faint grid, low-contrast panels.
- **Editorial + terminal typography:** Fraunces for large narrative headlines and numeric drama; JetBrains Mono for body, labels, tables, code, and metadata.
- **Numbered sections:** big amber section counters with hairline rules.
- **Dashboard cards:** bordered dark panels with tiny uppercase labels, serif numbers, colored status strips.
- **Ticker tape:** compact horizontal metadata row using slashes and small caps.
- **Honesty hooks:** dashed amber callouts for caveats, caveat-first summaries, and “what is not proven yet.”
- **System maps:** simple node/arrow diagrams; no stock imagery.
- **Quant visuals:** inline bars, gauges, cards, tables, and timeline/progress rows.

## Palette

```css
--ink: #0a0c0f;        /* page background */
--ink-2: #0e1116;      /* header / dark panels */
--paper: #13161b;      /* cards */
--paper-2: #181c23;    /* raised cards */
--paper-3: #1e232b;    /* deep cell surfaces */
--rule: #2a2f38;       /* major borders */
--rule-soft: #1f242c;  /* panel borders */
--fg: #ece7da;         /* warm foreground */
--fg-dim: #a39d8f;     /* readable secondary */
--fg-faint: #6c6960;   /* labels */
--amber: #e8a23a;      /* primary accent */
--teal: #5fb6ad;       /* secondary/technical */
--green: #86b06b;      /* good/success */
--red: #d96157;        /* negative/warning */
--rose: #c98b9c;       /* alternate series */
--warn: #f0c04a;       /* caution */
```

Light mode is intentionally deprecated for blog visuals. The site may keep a theme toggle, but the report aesthetic is dark-first.

## Typography

- Load fonts:
  - `Fraunces` variable: expressive serif, headline/numbers only.
  - `JetBrains Mono`: all body, nav, labels, code.
- Headline treatment:
  - 48–72px desktop, weight 300, tight line-height.
  - Italic `<em>` words in amber.
- Body:
  - 14–15px mono, 1.7 line-height, max-width around 74ch.
- Labels:
  - 9–11px mono, uppercase, letter spacing `.18em–.32em`.
- Numerals:
  - Fraunces for big metric values; tabular mono for tables.

## Page architecture

1. **Blog index**
   - Masthead with title, short mission copy, right-side report date/count metadata.
   - Ticker row with post count, latest publish date, tags, “static GitHub Pages.”
   - Featured/latest post as a wide report card.
   - Remaining posts as grid cards, each with a mini system diagram or metric strip.

2. **Blog post**
   - Report masthead:
     - eyebrow: `KEVINMJONES / FIELD NOTE / <topic>`
     - large serif title, with optional amber italic words handled by CSS if title contains `<em>` in template data
     - metadata block: date, reading-time proxy, tags.
   - Ticker tape:
     - slug, published date, primary tag, mode/status.
   - Visual briefing before prose:
     - stat/gauge cards
     - system pipeline diagram
     - honesty hook / caveat card
   - Article body:
     - numbered `h2` headings
     - panel-styled blockquotes
     - report-style tables
     - code in dark bordered blocks
   - Footer/backlink as quiet report footer.

## Components

### Gauge card

Use for metrics, project facts, and outcomes.

- Border: `1px solid var(--rule-soft)`
- Top 2px accent strip
- Label: uppercase mono
- Value: Fraunces 32–42px
- Subtext: dim mono

### System map

Use CSS grid/flex nodes.

- `.map-node`: dark panel, label/value split
- `.map-arrow`: mono arrow `→`
- On mobile, stack vertically.

### Honesty hook

Used for caveats and constraints.

- Dashed amber border
- Small uppercase amber label
- Body text with bold facts only where necessary

### Inline bar / mini chart

For visualizing before/after, score deltas, progress.

- Use CSS bars instead of canvas where possible.
- Amber = primary; teal = comparison; green/red = outcome.

## Content refactor rules

- Do not turn blog posts into image-heavy marketing pages.
- Every visual must communicate a real fact from the post.
- Keep caveats visible, especially for trading posts.
- Use first-person prose below the visual layer.
- Never imply live trading or real P&L where the source only supports synthetic/backtest/paper results.

## Current implementation notes

The Astro blog templates implement this style globally:

- `src/pages/blog/index.astro`
- `src/pages/blog/[...slug].astro`
- `src/styles/global.css`

Post-specific briefing cards are controlled in `src/pages/blog/[...slug].astro` by slug-keyed metadata. Current posts:

- `klax26-kalshi-lax-temperature`
- `xrp-arbitrage-calculator`
- `reverse-engineering-a-qidi-3d-printer`
