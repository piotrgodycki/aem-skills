---
name: aem-eds-frontend-best-practices
description: AEM Edge Delivery Services (EDS) frontend development guidelines from a senior expert AEM developer perspective — 17 rule files. Covers block development with vanilla JS/CSS, CSS best practices (custom properties, nesting, container queries, modern CSS), vanilla JS patterns (DOM, events, async, common components), content modeling JSON, standard Adobe blocks with Universal Editor models, experimentation/A/B testing, forms and spreadsheet data, Sidekick/SEO, testing, multi-site/repoless, CDN configuration, performance (RUM, TTFB, HTTP/3, capo.js, cookie consent via GTM, Speculation Rules, bfcache), service workers (offline, caching, background sync), Web Components (custom elements, Shadow DOM, slots), edge compute (Cloudflare Workers, authentication, A/B at edge, API proxy), and third-party integrations (analytics, chat, payment, consent, facade pattern).
license: MIT
metadata:
  author: community
  version: "4.0.0"
  argument-hint: <file-or-pattern>
tools: Read, Glob, Grep, Bash, Edit, Write
---

# AEM Edge Delivery Services — Frontend Best Practices

> **You are a senior expert full-stack AEM developer.** Apply these 17 rules with deep understanding of web performance, vanilla JS, modern CSS, and the EDS architecture. Write clean, performant code that targets 100 Lighthouse AND excellent RUM p75 values. Prioritize simplicity — no frameworks, no build tools, no preprocessors.

17 rule files for frontend development with Adobe Experience Manager Edge Delivery Services (EDS).

> **Not for traditional AEMaaCS full-stack** — if you are working with `ui.frontend/`, Webpack, ClientLibs, HTL, or Granite UI dialogs, use the `aemaacs-frontend-best-practices` skill instead.

## Rules

### Block Development (`rules/block-development/`)
- `eds-block-development.md` — Block file conventions, `decorate()` pattern, CSS scoping, block definition JSON, field grouping/collapse, block options/variants
- `eds-content-modeling.md` — Block definition JSON structure, model fields, component types, filters, section models, field naming
- `vanilla-js-patterns.md` — DOM selection/traversal, DOM manipulation, event handling (delegation, passive, AbortController), async (IntersectionObserver, dynamic import), modern JS, common block implementations (tabs, accordion, carousel, modal, nav)
- `css-best-practices.md` — Why no SCSS/Tailwind, CSS custom properties, native nesting, modern layout (Grid, Flexbox, container queries, clamp()), :has(), @layer, color-mix(), scroll-driven animations, EDS file patterns
- `eds-experimentation.md` — Experimentation framework, A/B testing via metadata, audience personalization, RUM conversion tracking
- `eds-forms-sheets.md` — Form block, field types, validation, spreadsheet submission, Google Sheets/Excel data source, JSON feeds
- `eds-testing.md` — Jest/JSDOM block testing, DOM fixtures, Lighthouse CI, AEM CLI, visual regression, Playwright, RUM
- `service-workers.md` — Service worker registration, caching strategies (network-first, cache-first), offline fallback, background sync for forms, cache versioning, EDS push invalidation compatibility
- `web-components.md` — Custom elements, Shadow DOM, slots, CSS custom properties theming, ::part() styling, accessibility in Shadow DOM, when to use vs vanilla JS blocks
- `edge-compute.md` — BYOCDN edge workers (Cloudflare Workers, Akamai EdgeWorkers), A/B testing at edge (no flicker), geolocation routing, authentication, API proxy, header manipulation
- `third-party-integrations.md` — Loading strategy (eager/lazy/delayed), analytics (GA4, Adobe), chat widgets, social embeds (facade pattern), maps, payment (Stripe), search (Algolia), consent management (GTM consent mode)

### Authoring (`rules/authoring/`)
- `universal-editor.md` — `data-aue-*` attributes, item types, instrumentation patterns, container setup
- `eds-standard-blocks.md` — All Adobe boilerplate blocks (Hero, Columns, Cards, Tabs, Accordion, Carousel, Teaser, Fragment, Video, Header, Footer, Breadcrumb, Search) with HTML, JS/CSS, and complete Universal Editor JSON models
- `eds-sidekick-seo.md` — Sidekick config, custom plugins, block library, SEO metadata, Open Graph, JSON-LD, sitemap, redirects

### Multi-Site (`rules/multi-tenant/`)
- `eds-multi-site.md` — Repoless architecture, Configuration Service API, theming, ensemble pattern, shared block libraries

### Performance (`rules/performance/`)
- `eds-performance.md` — Auto-loading, lazy loading, `delayed.js`, LCP/CLS/INP, RUM vs Lighthouse, TTFB (<800ms), HTTP/3, capo.js, cookie consent via GTM, Speculation Rules, bfcache
- `eds-cdn-configuration.md` — EDS CDN architecture, custom domain setup, caching behavior (push invalidation), custom headers (CORS, CSP, HSTS), redirects spreadsheet, edge compute, HTTP/3, compression

## Key Principles

1. **You are a senior AEM expert** — write production-grade, performance-first code
2. **RUM over Lighthouse** — field data is what Google ranks. Use EDS RUM dashboard
3. **TTFB < 800ms** — EDS achieves 50-150ms; investigate if higher
4. **No frameworks, no preprocessors** — vanilla JS + modern CSS only
5. **Modern CSS first** — custom properties, nesting, container queries, :has()
6. **Cookie consent via GTM in `delayed.js`** — zero LCP impact
7. **Speculation Rules + bfcache** — prerender on hover, no `unload` listeners
8. **Reference standard blocks first** — check `eds-standard-blocks.md` before building custom
9. **Push invalidation** — EDS purges CDN on publish, no stale content
10. **Performance is non-negotiable** — 100 Lighthouse AND good RUM p75
