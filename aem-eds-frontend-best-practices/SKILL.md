---
name: aem-eds-frontend-best-practices
description: AEM Edge Delivery Services (EDS / Franklin / Helix) frontend. Use when building or reviewing EDS blocks, scripts.js/styles.css, content models, Universal Editor block config, or tuning EDS performance (RUM, LCP, TTFB). Vanilla JS and modern CSS only — no build step, no frameworks. For server-rendered AEM components use aem-best-practices instead.
---

# AEM Edge Delivery Services — Frontend Best Practices

Production-grade patterns for Edge Delivery Services, written for a senior frontend engineer. Each reference file carries the correct pattern, the incorrect one beside it, and the anti-patterns that cost you a 100 Lighthouse score.

EDS serves JS and CSS **as-is with zero build step**: vanilla ES modules and native modern CSS only. No SCSS, no preprocessor, no framework (React/Vue/Svelte). Modern CSS — custom properties, nesting, container queries, `:has()`, `@layer` — covers what preprocessors used to. Performance is the product: the 100/100/100/100 Lighthouse target drives every decision here.

## How to use this skill

This is reference, not a sequence. Open the one or two files that bear on the task **before** writing code, apply the pattern, then move on. Match the boilerplate's existing idiom when it conflicts with an example. A new block needs `block-development/`, not the whole skill.

## Reference map

**Block development** — reach when writing or editing a block, or core `scripts.js`/`styles.css`:
- `rules/block-development/eds-block-development.md` — block structure, `decorate()`, the three-phase loading model (E-L-D), auto-blocking
- `rules/block-development/css-best-practices.md` — custom properties, nesting, container queries, `:has()`, `color-mix()`, `@layer`, CSS without a build
- `rules/block-development/vanilla-js-patterns.md` — DOM building, events, async, module patterns, common components
- `rules/block-development/eds-content-modeling.md` — block table structure, modelling content authors can edit, JSON content
- `rules/block-development/web-components.md` — custom elements, Shadow DOM, slots inside blocks
- `rules/block-development/service-workers.md` — offline, caching strategies, background sync
- `rules/block-development/edge-compute.md` — Cloudflare Workers, auth, A/B at the edge, API proxy
- `rules/block-development/third-party-integrations.md` — analytics, chat, payment, consent, the facade pattern for deferred loading
- `rules/block-development/eds-experimentation.md` — experimentation plugin, A/B tests, metrics
- `rules/block-development/eds-forms-sheets.md` — forms block, spreadsheet-backed data, submission handling
- `rules/block-development/eds-testing.md` — unit tests for logic, Playwright for blocks, linting

**Authoring & content** — reach when shaping how authors work or configuring Universal Editor:
- `rules/authoring/content-driven-development.md` — content-first workflow, document authoring, modelling from the doc
- `rules/authoring/universal-editor.md` — Universal Editor block config, component models/definitions/filters
- `rules/authoring/eds-standard-blocks.md` — standard Adobe blocks (cards, columns, hero…) with UE models
- `rules/authoring/eds-sidekick-seo.md` — Sidekick, metadata, SEO, structured data

**Multi-site & performance** — reach when scaling across sites or chasing Core Web Vitals:
- `rules/multi-tenant/eds-multi-site.md` — repoless, multi-site, theming across sites
- `rules/performance/eds-performance.md` — RUM, TTFB, HTTP/3, capo.js, Speculation Rules, bfcache, consent via GTM
- `rules/performance/eds-cdn-configuration.md` — CDN config, caching, push invalidation, origin selection
