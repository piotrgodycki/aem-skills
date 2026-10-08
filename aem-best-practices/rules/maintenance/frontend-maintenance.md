---
title: Frontend Maintenance (ClientLibs, Bundles, CSS Debt)
impact: MEDIUM
impactDescription: Unmanaged ClientLib and bundle growth degrades Core Web Vitals and makes CSS changes unpredictable in a mature AEM front-end.
tags: clientlibs, ui-frontend, webpack, css-debt, bundle, performance, maintenance
---

## Frontend Maintenance

Maintaining the `ui.frontend` + ClientLibs layer of an existing AEM project: keeping bundles lean, CSS predictable, and the Core Component markup contract intact. Scope every change via `spec-driven-development.md`; verify against RUM/CWV, not vibes.

### 1. Audit the Front-End Build First

- Identify the pipeline: `ui.frontend` Webpack → `aem-clientlib-generator` → ClientLib categories. Confirm `webpack.common/dev/prod` and the clientlib mapping (`clientlib.config.js`).
- Map ClientLib categories and their dependency/embed graph — which categories load on which templates (`policies`, page `clientlibs`).
- Measure the baseline: `webpack-bundle-analyzer`, number of ClientLibs on a page, render-blocking CSS/JS, current RUM p75 for LCP/CLS/INP.

### 2. Keep Markup Contracts Intact

CSS targets Core Component BEM markup. When upgrading Core Components or editing HTL, re-verify selectors.

**Correct — target stable BEM hooks:**
```css
/* Survives Core Component patch upgrades */
.cmp-teaser__title { font-size: var(--font-size-l); }
```

**Incorrect — brittle structural selectors:**
```css
/* Breaks the moment markup nesting changes in a CC upgrade */
.teaser > div > div:first-child h3 { font-size: 24px; }
```

### 3. Manage CSS Debt

- Prefer extending the existing SCSS domain structure (`frontend/scss-domain-structure.md`) over bolting on a new global stylesheet.
- Remove dead CSS — tie selectors to components still in the component group; delete styles for retired components.
- Consolidate duplicated custom properties into the design-token layer; don't introduce a parallel token set.
- If the project uses Tailwind, respect the existing prefix/BEM coexistence strategy (`frontend/tailwind-aem.md`) — don't mix conventions.

### 4. Bundle Hygiene

- Code-split per feature; load heavy JS in a lazy/`delayed` ClientLib category, not the core category.
- Dedupe dependencies after any npm change (`npm ls <pkg>`, dedupe duplicate React/lodash copies).
- Keep immutable, versioned ClientLib URLs for long cache TTLs (`performance/cloud-performance.md`).
- Progressive enhancement: AEM ships server-rendered HTL; JS augments it. Don't convert a component to client-rendered to "modernise" it.

### 5. Verify Against Field Data

- Rebuild, deploy to a lower env, compare bundle sizes vs baseline.
- Check CWV with RUM field data (what Google ranks), not just a one-off Lighthouse run.
- Visual-diff representative templates; confirm no FOUC/CLS introduced.

### AEM 6.5 / Classic differences
- Same `ui.frontend` → ClientLib pipeline works on 6.5; the difference is delivery (no Adobe CDN — see `performance/cdn-configuration.md` 6.5 notes).
- ClientLib minification can be handled by the HTML Library Manager (`htmllibmanager`) on 6.5 instead of (or alongside) the Webpack build.
- RUM: EDS-style RUM is not built in; use your own analytics/CrUX for field data.

### Anti-Patterns
- Brittle structural CSS selectors instead of BEM hooks.
- Adding a new global stylesheet instead of extending the existing SCSS domains.
- Loading heavy JS in the core (eager) ClientLib category.
- Shipping duplicate dependency copies after an npm change.
- Converting server-rendered HTL components to client-rendered for no measured benefit.
- Validating only with lab Lighthouse while RUM p75 regresses.
