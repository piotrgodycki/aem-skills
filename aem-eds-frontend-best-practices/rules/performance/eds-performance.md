---
title: EDS Performance Optimization
impact: CRITICAL
impactDescription: EDS targets 100 Lighthouse score — performance mistakes directly degrade Core Web Vitals and user experience
tags: eds, performance, lighthouse, lcp, cls, fid, lazy-loading, fonts, images
---

## EDS Performance Optimization

Edge Delivery Services is engineered for perfect Lighthouse scores. The architecture automatically handles many performance optimizations, but block developers must follow specific patterns to maintain this performance.

### Built-in Performance Features

EDS provides these optimizations automatically — don't reimplement them:

- **Block auto-loading**: CSS/JS loaded only when a block appears on the page
- **Lazy loading of images**: Below-fold images get `loading="lazy"` automatically
- **Eager loading of LCP image**: First visible image is loaded eagerly
- **CSS/JS splitting**: Each block's assets are isolated and loaded on demand
- **Minimal HTML**: Server delivers clean, semantic HTML with no framework overhead
- **CDN delivery**: All assets served from edge locations globally

### Script Loading Tiers

EDS has three tiers of script execution:

```
scripts/aem.js      → Runtime utilities (loaded first, inline)
scripts/scripts.js  → Site-wide JS (loaded on DOMContentLoaded)
scripts/delayed.js  → Deferred scripts (loaded 3s after page load)
```

#### What goes in `delayed.js`

Everything non-essential for initial render:

```javascript
// delayed.js — loaded ~3 seconds after page load
// Analytics, tracking, chat widgets, social embeds

// Load analytics
const script = document.createElement('script');
script.src = 'https://analytics.example.com/tracker.js';
script.async = true;
document.head.appendChild(script);

// Initialize chat widget
window.addEventListener('scroll', () => {
  // Load chat on first interaction
}, { once: true });
```

### LCP (Largest Contentful Paint) Optimization

The LCP element is typically the hero image or heading. Critical rules:

```javascript
// In your block's decorate function:
export default function decorate(block) {
  const img = block.querySelector('img');

  // EDS handles eager/lazy automatically based on position
  // Do NOT manually set loading="lazy" on above-fold images
  // Do NOT add loading="eager" — EDS does this for LCP candidates

  // DO: keep DOM manipulation minimal to avoid layout shifts
  // DO: avoid wrapping images in extra containers unless necessary
}
```

**Incorrect — blocking LCP:**

```javascript
export default async function decorate(block) {
  // Fetching data before rendering blocks LCP
  const response = await fetch('/api/data.json');
  const data = await response.json();

  // Heavy DOM manipulation delays paint
  block.innerHTML = '';
  data.items.forEach(item => {
    const div = document.createElement('div');
    div.innerHTML = `<h2>${item.title}</h2><p>${item.description}</p>`;
    block.appendChild(div);
  });
}
```

**Correct — non-blocking enhancement:**

```javascript
export default function decorate(block) {
  // Enhance existing DOM — don't replace it
  block.querySelector('.title')?.classList.add('animated');

  // Defer non-critical work
  const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        // Load additional content when visible
        loadExtraContent(entry.target);
        observer.unobserve(entry.target);
      }
    });
  });
  block.querySelectorAll('.lazy-section').forEach(el => observer.observe(el));
}
```

### CLS (Cumulative Layout Shift) Prevention

```css
/* Reserve space for images to prevent layout shift */
.block.hero {
  .image-wrapper {
    aspect-ratio: 16 / 9;
    width: 100%;
  }

  /* Don't use auto height on containers with dynamic content */
  min-height: 400px;
}
```

**Incorrect:**

```javascript
// Inserting elements that shift layout
export default function decorate(block) {
  const banner = document.createElement('div');
  banner.textContent = 'Special offer!';
  block.prepend(banner); // Shifts everything down = CLS
}
```

**Correct:**

```javascript
// Use existing DOM structure, add classes for styling
export default function decorate(block) {
  const firstRow = block.querySelector(':scope > div:first-child');
  firstRow?.classList.add('banner'); // Style with CSS, no layout shift
}
```

### Font Loading

```css
/* styles/fonts.css — loaded with site styles */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom-regular.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap; /* Show fallback immediately, swap when loaded */
}

@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom-bold.woff2') format('woff2');
  font-weight: 700;
  font-style: normal;
  font-display: swap;
}
```

**Best practices for fonts:**
- Use `font-display: swap` — prevents invisible text during load
- Use WOFF2 format only — smallest file size, universal browser support
- Self-host fonts in `/fonts/` — avoid third-party CDN latency
- Limit to 2-3 font files — each file blocks rendering slightly
- Preload critical fonts in `head.html`:

```html
<link rel="preload" href="/fonts/custom-regular.woff2" as="font" type="font/woff2" crossorigin>
```

### Image Optimization

EDS serves images through its CDN with automatic optimization. Follow these rules:

1. **Use `<picture>` and responsive images** — EDS generates these from authored content
2. **Don't load images in JavaScript** — let the HTML/CSS handle it
3. **Use SVG for icons** — place in `/icons/`, reference by name
4. **Avoid background images in CSS for content images** — invisible to lazy loading
5. **Keep image dimensions in HTML** — prevents CLS

### DOM Manipulation Rules

```javascript
// GOOD: Add classes to existing elements
block.querySelector('.content')?.classList.add('highlighted');

// GOOD: Minor restructuring
const items = [...block.querySelectorAll(':scope > div')];
const list = document.createElement('ul');
items.forEach(item => {
  const li = document.createElement('li');
  li.append(...item.childNodes);
  list.appendChild(li);
});
block.textContent = '';
block.appendChild(list);

// BAD: Fetching and replacing all content
block.innerHTML = await (await fetch('/fragment.html')).text();

// BAD: Creating complex component trees
block.innerHTML = `
  <div class="wrapper">
    <div class="inner">
      <div class="content-area">
        ${/* deeply nested template */}
      </div>
    </div>
  </div>
`;
```

### Third-Party Scripts

Never load third-party scripts in block JS or `scripts.js`. Always use `delayed.js`:

```javascript
// delayed.js
function loadThirdParty() {
  // Google Tag Manager
  const gtm = document.createElement('script');
  gtm.src = `https://www.googletagmanager.com/gtm.js?id=GTM-XXXX`;
  gtm.async = true;
  document.head.appendChild(gtm);
}

loadThirdParty();
```

### Performance Checklist

- [ ] No `async/await` in `decorate()` for above-fold blocks
- [ ] No `fetch()` calls blocking initial render
- [ ] Images use natural HTML — not loaded via JavaScript
- [ ] Third-party scripts in `delayed.js` only
- [ ] Fonts use `font-display: swap` and WOFF2 format
- [ ] CSS uses `aspect-ratio` to reserve image/video space
- [ ] No unnecessary DOM nesting (3 levels max for block content)
- [ ] `IntersectionObserver` used for below-fold enhancements
- [ ] No inline `<style>` or `<script>` tags in block JS
- [ ] Block CSS scoped to `.block.<name>` — no global selectors

---

### Real User Monitoring (RUM) — The True Performance Metric

**Lighthouse is a synthetic lab test. RUM is what matters.**

EDS has built-in Real User Monitoring (Operational Telemetry) that captures field data from real visitors. This is the authoritative source for performance optimization — not Lighthouse.

#### Why RUM Over Lighthouse

| Aspect | Lighthouse (Lab) | EDS RUM (Field) |
|--------|-----------------|-----------------|
| Network | Simulated throttle | Real connections worldwide |
| Device | Emulated Moto G4 | Actual devices (low-end Android, iPhones) |
| Sample | 1 cold-cache load | Thousands of real visits |
| SEO impact | None | CrUX field data drives search ranking |
| Caching | Always cold cache | Mix of warm/cold |

Google uses **CrUX (Chrome User Experience Report)** — field data — for ranking. A Lighthouse 100 means nothing if real users on 3G in India experience 5s LCP.

#### EDS RUM Data Collection

EDS automatically samples real user visits and collects:
- **Core Web Vitals**: LCP, CLS, INP, TTFB
- **Page load timing**: DNS, TCP, TLS, server response, DOM processing
- **Custom checkpoints**: block-specific events you define

```javascript
// Add a custom RUM checkpoint in your block
import { sampleRUM } from '../../scripts/aem.js';

export default function decorate(block) {
  // Track when a video starts playing
  block.querySelector('video')?.addEventListener('play', () => {
    sampleRUM('video-play', { source: block.dataset.blockName });
  });
}
```

#### RUM-Driven Optimization Workflow

1. **Measure** — Collect RUM for 7+ days via EDS dashboard
2. **Identify** — Find worst pages by p75 LCP/CLS/INP
3. **Diagnose** — Use Lighthouse/DevTools on those specific pages
4. **Fix** — Apply targeted optimizations
5. **Validate** — Confirm improvement in next 7-day RUM window

Access RUM data at: `https://www.aem.live/tools/rum-explorer`

---

### TTFB Target: Below 800ms

Time to First Byte should be under 800ms. EDS achieves this through edge-first architecture:

```
TTFB = DNS + TCP/QUIC + TLS + CDN Edge Response
     = ~50ms (CDN hit) to ~300ms (CDN miss → origin)
```

EDS sites typically achieve 50-150ms TTFB because content is served from CDN edge. If you see TTFB > 800ms, investigate:
- CDN cache miss rate (check `X-Cache` header)
- AEM Author publish latency
- Geographic distance to nearest edge node

#### HTTP/3 (QUIC)

EDS CDN supports HTTP/3 automatically. Benefits:
- **0-RTT connections** — eliminates TCP+TLS handshake
- **No head-of-line blocking** — independent stream multiplexing
- **Better on mobile** — handles WiFi↔cellular switches gracefully

With HTTP/3, the EDS pattern of many small per-block CSS/JS files is actually optimal — multiplexing handles concurrent requests efficiently.

---

### `<head>` Element Order (capo.js)

EDS manages `<head>` via `head.html`. The order of elements matters significantly:

```html
<!-- head.html — optimal order -->

<!-- 1. Meta charset MUST be first (within first 1024 bytes) -->
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">

<!-- 2. Preconnect to required origins -->
<link rel="preconnect" href="https://fonts.googleapis.com" crossorigin>

<!-- 3. Preload critical resources -->
<link rel="preload" href="/fonts/brand.woff2" as="font" type="font/woff2" crossorigin>

<!-- 4. Global styles (render-blocking — keep minimal) -->
<link rel="stylesheet" href="/styles/styles.css">

<!-- 5. Scripts (defer by default in EDS) -->
<script src="/scripts/aem.js" type="module"></script>
<script src="/scripts/scripts.js" type="module"></script>

<!-- 6. SEO / social meta (doesn't affect rendering) -->
<!-- Injected from metadata block -->

<!-- 7. Favicon -->
<link rel="icon" href="/icons/favicon.ico">
```

Use [capo.js](https://github.com/rviscomi/capo.js) Chrome extension to audit your `<head>` order. Misplaced elements (e.g., CSS after JS, preconnect after the resource request) directly increase LCP.

---

### Cookie Consent via GTM (Best Practice)

Cookie consent is the #1 performance killer when loaded incorrectly. **Load consent management through GTM in `delayed.js`.**

```javascript
// delayed.js — loaded ~3s after page load
function loadGTM() {
  // Set consent defaults BEFORE GTM loads
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}

  gtag('consent', 'default', {
    'analytics_storage': 'denied',
    'ad_storage': 'denied',
    'ad_user_data': 'denied',
    'ad_personalization': 'denied',
    'wait_for_update': 500
  });

  // Load GTM (which loads CMP as a GTM tag)
  const gtm = document.createElement('script');
  gtm.src = 'https://www.googletagmanager.com/gtm.js?id=GTM-XXXX';
  gtm.async = true;
  document.head.appendChild(gtm);
}

loadGTM();
```

**Why this pattern:**
- Zero impact on LCP — GTM loads 3s after page
- Consent banner appears after content is interactive
- GTM consent mode gates all tracking tags
- CMP managed in GTM — no code deploys for consent config changes

**Anti-patterns:**
- Loading CMP in `scripts.js` or `head.html` — blocks rendering
- Loading CMP independently from GTM — two scripts instead of one
- Consent wall that prevents content rendering — use non-blocking banner

---

### Speculation Rules & Back/Forward Cache (bfcache)

#### Speculation Rules API

Speculation Rules allow the browser to prefetch or prerender likely next navigations, making them feel instant:

```html
<!-- In head.html or injected by scripts.js -->
<script type="speculationrules">
{
  "prerender": [{
    "where": {
      "and": [
        { "href_matches": "/*" },
        { "not": { "href_matches": "/logout" } },
        { "not": { "selector_matches": "[data-no-prerender]" } }
      ]
    },
    "eagerness": "moderate"
  }],
  "prefetch": [{
    "where": { "href_matches": "/*" },
    "eagerness": "moderate"
  }]
}
</script>
```

**Eagerness levels:**
- `immediate` — speculate as soon as rules are observed
- `eager` — earlier than moderate, later than immediate
- `moderate` — speculate on hover (200ms) — **recommended for most EDS sites**
- `conservative` — speculate on pointer down (click start)

```javascript
// Add speculation rules dynamically in scripts.js
function addSpeculationRules() {
  if (!HTMLScriptElement.supports?.('speculationrules')) return;

  const rules = document.createElement('script');
  rules.type = 'speculationrules';
  rules.textContent = JSON.stringify({
    prerender: [{
      where: { href_matches: '/*' },
      eagerness: 'moderate',
    }],
  });
  document.head.appendChild(rules);
}

addSpeculationRules();
```

#### Back/Forward Cache (bfcache)

bfcache stores a complete snapshot of the page in memory when the user navigates away. Pressing Back/Forward restores it instantly (0ms load).

**EDS sites are bfcache-eligible by default** because they use simple HTML/CSS/JS. But these patterns break bfcache:

```javascript
// ❌ BREAKS bfcache
window.addEventListener('unload', () => { ... });
// Use 'pagehide' instead

// ❌ BREAKS bfcache
Cache-Control: no-store
// Use: Cache-Control: max-age=300

// ❌ BREAKS bfcache
window.opener  // pages opened with window.open
```

**Correct — bfcache-friendly patterns:**

```javascript
// ✅ Use pagehide + persisted check
window.addEventListener('pagehide', (event) => {
  if (event.persisted) {
    // Page is going into bfcache — clean up, don't prevent it
  }
});

// ✅ Restore state on bfcache restore
window.addEventListener('pageshow', (event) => {
  if (event.persisted) {
    // Restored from bfcache — refresh dynamic content
    updateTimestamps();
    refreshUserState();
  }
});

// ✅ Send analytics on bfcache restore (counts as new pageview)
window.addEventListener('pageshow', (event) => {
  if (event.persisted) {
    sampleRUM('pageview'); // Re-track the view
  }
});
```

**Test bfcache eligibility:**
- Chrome DevTools > Application > Back/forward cache > "Test back/forward cache"
- Lists all reasons preventing bfcache if any

---

### Pitfalls

- `async` decorate functions delay LCP — keep `decorate()` synchronous for above-fold blocks
- `innerHTML` replacement destroys Universal Editor instrumentation attributes
- Third-party scripts in `scripts.js` block page interactivity (FID/INP)
- Missing `aspect-ratio` on media containers causes CLS
- Loading too many font weights/styles increases page weight
- Background images in CSS bypass lazy loading — use `<img>` elements
- Optimizing for Lighthouse 100 while ignoring RUM field data — field data is what Google ranks
- TTFB > 800ms on CDN miss — check cache hit rate and AEM publish latency
- `unload` event listeners prevent bfcache — use `pagehide` instead
- Cookie consent in `scripts.js` instead of `delayed.js` — blocks interactivity
