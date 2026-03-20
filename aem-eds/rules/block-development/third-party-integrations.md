---
title: Third-Party Service Integration with EDS
impact: HIGH
impactDescription: Third-party scripts are the #1 cause of poor Lighthouse scores — incorrect loading patterns destroy LCP, CLS, and INP on otherwise fast EDS sites
tags: eds, third-party, analytics, crm, chat, payment, social, consent, gtm, facade-pattern, delayed-js, performance
---

## Third-Party Service Integration with EDS

Every third-party script is a performance tax. EDS achieves 100 Lighthouse because it starts with zero dependencies. The loading strategy (eager, lazy, delayed) determines whether a third-party integration helps or hurts.

---

### 1. Loading Strategy

```
Page Load Timeline:
├── Eager (LCP-critical)     → scripts.js   — only critical above-fold functionality
├── Lazy (below-fold)        → IntersectionObserver — loads when scrolled into view
└── Delayed (non-critical)   → delayed.js   — loads 3-5s after page load
```

| Integration | Loading | Reason |
|------------|---------|--------|
| Analytics (GA4, Adobe) | **Delayed** | Not needed for user experience |
| Tag Manager (GTM) | **Delayed** | Loads all tags; zero LCP impact |
| Chat widgets | **Delayed** | User won't chat in first 3 seconds |
| Social embeds | **Lazy** | Only when scrolled into view |
| Maps | **Lazy** | Heavy; only load when visible |
| Video (YouTube, Vimeo) | **Lazy** | Facade pattern until interaction |
| Payment (Stripe) | **Lazy** | Only on checkout/form pages |
| CRM forms | **Lazy** | Only when form is visible |
| Cookie consent | **Delayed** (via GTM) | GTM consent mode |
| A/B testing | **Edge** or **Delayed** | Edge preferred (no flicker) |

---

### 2. Analytics Integration

#### Google Analytics 4 (via GTM in delayed.js)

```javascript
// scripts/delayed.js
export default function loadDelayed() {
  // Load GTM — handles GA4, consent, all tags
  const gtmId = 'GTM-XXXXXXX';
  window.dataLayer = window.dataLayer || [];
  window.dataLayer.push({
    'gtm.start': Date.now(),
    event: 'gtm.js',
  });

  const script = document.createElement('script');
  script.src = `https://www.googletagmanager.com/gtm.js?id=${gtmId}`;
  script.async = true;
  document.head.append(script);
}
```

#### Adobe Analytics (via Alloy/Web SDK)

```javascript
// scripts/delayed.js
function loadAdobeAnalytics() {
  const script = document.createElement('script');
  script.src = 'https://cdn1.adoberesources.net/alloy/2.19.0/alloy.min.js';
  script.async = true;
  script.onload = () => {
    window.alloy('configure', {
      datastreamId: 'YOUR_DATASTREAM_ID',
      orgId: 'YOUR_ORG_ID@AdobeOrg',
    });
    window.alloy('sendEvent', {
      xdm: {
        web: {
          webPageDetails: {
            name: document.title,
            URL: window.location.href,
          },
        },
      },
    });
  };
  document.head.append(script);
}
```

#### Custom RUM Events (EDS Built-In)

```javascript
// Use EDS RUM for core metrics — no third-party needed
// sampleRUM is available globally in EDS
window.sampleRUM('conversion', {
  source: 'newsletter-signup',
  target: window.location.pathname,
});
```

---

### 3. Chat Widgets

Always load in `delayed.js` — users never need chat in the first 3 seconds:

```javascript
// scripts/delayed.js
function loadIntercom() {
  window.intercomSettings = {
    api_base: 'https://api-iam.intercom.io',
    app_id: 'YOUR_APP_ID',
  };

  const script = document.createElement('script');
  script.src = 'https://widget.intercom.io/widget/YOUR_APP_ID';
  script.async = true;
  document.head.append(script);
}

function loadZendesk() {
  const script = document.createElement('script');
  script.id = 'ze-snippet';
  script.src = 'https://static.zdassets.com/ekr/snippet.js?key=YOUR_KEY';
  script.async = true;
  document.head.append(script);
}
```

---

### 4. Social Media Embeds (Facade Pattern)

Never load embed scripts eagerly. Use a facade (placeholder) that loads the real embed on interaction:

```javascript
// blocks/youtube/youtube.js
export default function decorate(block) {
  const link = block.querySelector('a');
  const videoUrl = new URL(link.href);
  const videoId = videoUrl.searchParams.get('v') || videoUrl.pathname.split('/').pop();

  // Create facade (static thumbnail) — zero JS cost
  const facade = document.createElement('div');
  facade.className = 'youtube-facade';
  facade.innerHTML = `
    <img src="https://i.ytimg.com/vi/${videoId}/maxresdefault.jpg"
         alt="Play video" loading="lazy" width="640" height="360">
    <button class="youtube-play" aria-label="Play video">
      <svg viewBox="0 0 68 48" width="68" height="48">
        <path d="M66.52 7.74c-.78-2.93-2.49-5.41-5.42-6.19C55.79.13 34 0 34 0S12.21.13 6.9 1.55C3.97 2.33 2.27 4.81 1.48 7.74.06 13.05 0 24 0 24s.06 10.95 1.48 16.26c.78 2.93 2.49 5.41 5.42 6.19C12.21 47.87 34 48 34 48s21.79-.13 27.1-1.55c2.93-.78 4.64-3.26 5.42-6.19C67.94 34.95 68 24 68 24s-.06-10.95-1.48-16.26z" fill="red"/>
        <path d="M45 24L27 14v20" fill="white"/>
      </svg>
    </button>
  `;

  // Load real iframe only on click
  facade.addEventListener('click', () => {
    const iframe = document.createElement('iframe');
    iframe.src = `https://www.youtube-nocookie.com/embed/${videoId}?autoplay=1`;
    iframe.allow = 'autoplay; encrypted-media';
    iframe.allowFullscreen = true;
    iframe.loading = 'lazy';
    iframe.style.cssText = 'position:absolute;top:0;left:0;width:100%;height:100%;border:0';

    facade.replaceWith(iframe);
  }, { once: true });

  block.textContent = '';
  block.append(facade);
}
```

---

### 5. Map Integrations

Load only when scrolled into view:

```javascript
// blocks/map/map.js
export default function decorate(block) {
  const address = block.textContent.trim();
  block.textContent = '';

  // Placeholder until visible
  const placeholder = document.createElement('div');
  placeholder.className = 'map-placeholder';
  placeholder.textContent = 'Loading map...';
  placeholder.style.cssText = 'aspect-ratio:16/9;background:#f0f0f0;display:grid;place-items:center';
  block.append(placeholder);

  // Load map library when block enters viewport
  const observer = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        observer.disconnect();
        loadMap(block, address);
      }
    });
  }, { rootMargin: '200px' }); // Start loading 200px before visible

  observer.observe(block);
}

async function loadMap(container, address) {
  // Dynamic import of map library
  const script = document.createElement('script');
  script.src = `https://maps.googleapis.com/maps/api/js?key=YOUR_KEY&callback=initMap`;
  script.async = true;

  window.initMap = () => {
    const map = new google.maps.Map(container.querySelector('.map-placeholder'), {
      zoom: 15,
      center: { lat: 0, lng: 0 },
    });

    const geocoder = new google.maps.Geocoder();
    geocoder.geocode({ address }, (results, status) => {
      if (status === 'OK') {
        map.setCenter(results[0].geometry.location);
        new google.maps.Marker({ map, position: results[0].geometry.location });
      }
    });
  };

  document.head.append(script);
}
```

---

### 6. Payment Integration

```javascript
// blocks/checkout/checkout.js — Stripe Elements
export default async function decorate(block) {
  // Only load Stripe on checkout pages
  const stripe = await loadStripe();

  const elements = stripe.elements();
  const cardElement = elements.create('card', {
    style: {
      base: {
        fontSize: '16px',
        color: '#333',
        fontFamily: 'var(--body-font-family)',
      },
    },
  });

  const form = document.createElement('form');
  const cardContainer = document.createElement('div');
  cardContainer.id = 'card-element';

  const submitBtn = document.createElement('button');
  submitBtn.type = 'submit';
  submitBtn.textContent = 'Pay Now';

  form.append(cardContainer, submitBtn);
  block.textContent = '';
  block.append(form);

  cardElement.mount('#card-element');

  form.addEventListener('submit', async (e) => {
    e.preventDefault();
    submitBtn.disabled = true;

    const { token, error } = await stripe.createToken(cardElement);
    if (error) {
      showError(error.message);
      submitBtn.disabled = false;
    } else {
      await processPayment(token.id);
    }
  });
}

function loadStripe() {
  return new Promise((resolve) => {
    const script = document.createElement('script');
    script.src = 'https://js.stripe.com/v3/';
    script.onload = () => resolve(window.Stripe('pk_live_YOUR_KEY'));
    document.head.append(script);
  });
}
```

---

### 7. Consent Management (GDPR/CCPA)

```javascript
// scripts/delayed.js — GTM Consent Mode
export default function loadDelayed() {
  // Initialize consent defaults BEFORE GTM loads
  window.dataLayer = window.dataLayer || [];
  function gtag() { window.dataLayer.push(arguments); }

  // Default: deny all (GDPR regions)
  gtag('consent', 'default', {
    ad_storage: 'denied',
    analytics_storage: 'denied',
    functionality_storage: 'denied',
    personalization_storage: 'denied',
    security_storage: 'granted', // always allowed
    wait_for_update: 500, // ms to wait for CMP
  });

  // Load GTM
  loadGTM('GTM-XXXXXXX');

  // GTM manages the consent banner UI and updates consent state
  // When user accepts: gtag('consent', 'update', { analytics_storage: 'granted' })
}

function loadGTM(gtmId) {
  window.dataLayer.push({ 'gtm.start': Date.now(), event: 'gtm.js' });
  const script = document.createElement('script');
  script.src = `https://www.googletagmanager.com/gtm.js?id=${gtmId}`;
  script.async = true;
  document.head.append(script);
}
```

---

### 8. Search Integration

```javascript
// blocks/search/search.js — Algolia InstantSearch
export default async function decorate(block) {
  const input = document.createElement('input');
  input.type = 'search';
  input.placeholder = 'Search...';
  input.className = 'search-input';

  const results = document.createElement('div');
  results.className = 'search-results';
  results.hidden = true;

  block.textContent = '';
  block.append(input, results);

  let algolia = null;

  // Load Algolia only when user focuses the search input
  input.addEventListener('focus', async () => {
    if (!algolia) {
      algolia = await loadAlgolia();
    }
  }, { once: true });

  // Debounced search
  let debounceTimer;
  input.addEventListener('input', () => {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(async () => {
      if (!algolia || input.value.length < 2) {
        results.hidden = true;
        return;
      }
      const hits = await algolia.search(input.value);
      renderResults(results, hits);
      results.hidden = false;
    }, 300);
  });
}

async function loadAlgolia() {
  const { default: algoliasearch } = await import(
    'https://cdn.jsdelivr.net/npm/algoliasearch@4/dist/algoliasearch-lite.esm.browser.js'
  );
  const client = algoliasearch('APP_ID', 'SEARCH_API_KEY');
  return client.initIndex('mysite_content');
}
```

---

### 9. Anti-Patterns

#### Loading Scripts Eagerly

```javascript
// WRONG — third-party in head or scripts.js
// <script src="https://www.googletagmanager.com/gtm.js?id=GTM-XXX"></script>
// Impact: +200-500ms to LCP

// CORRECT — always in delayed.js or lazy-loaded
// See loading strategy table in section 1
```

#### Not Using Facade Pattern

```html
<!-- WRONG — YouTube iframe loads immediately (1MB+ resources) -->
<iframe src="https://www.youtube.com/embed/VIDEO_ID"></iframe>

<!-- CORRECT — facade loads on click (0 KB until interaction) -->
<!-- See section 4 for implementation -->
```

#### Blocking LCP with Consent Banner

```javascript
// WRONG — consent banner renders before page content
document.addEventListener('DOMContentLoaded', () => {
  showConsentBanner(); // Shifts layout, blocks LCP
});

// CORRECT — consent via GTM in delayed.js
// Banner appears after page is interactive, no LCP impact
```

#### Multiple Tag Managers

```javascript
// WRONG — loading GTM + Adobe Launch + segment.io
// Each adds 50-200ms and 50-200KB

// CORRECT — pick ONE tag manager (GTM recommended)
// Route all integrations through it
```

#### Direct SDK Loading Instead of Tag Manager

```html
<!-- WRONG — loading each SDK directly -->
<script src="https://www.google-analytics.com/analytics.js"></script>
<script src="https://connect.facebook.net/en_US/fbevents.js"></script>
<script src="https://snap.licdn.com/li.lms-analytics/insight.min.js"></script>

<!-- CORRECT — load via GTM, which handles consent, loading order, and deduplication -->
```
