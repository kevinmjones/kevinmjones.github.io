# kevinmjones.github.io

Personal portfolio + blog. Astro 5, deployed to GitHub Pages via Actions.

## Local dev

```bash
npm install
npm run dev      # http://localhost:4321
npm run build    # outputs to dist/
```

## Adding a blog post

Drop a markdown file into `src/content/blog/<slug>.md` with frontmatter:

```yaml
---
title: "Post title"
description: "One-paragraph summary"
pubDate: 2026-04-30
tags: [tag1, tag2]
---
```

Push to `master` and the GitHub Action builds and deploys.

## Stack

- [Astro 5](https://astro.build) (static, content collections, MDX)
- IBM Plex Sans (body), JetBrains Mono (code) via Google Fonts
- Light/dark theme toggle (CSS custom properties + `localStorage`)
- Shiki syntax highlighting
- RSS at `/rss.xml`, sitemap at `/sitemap-index.xml`
