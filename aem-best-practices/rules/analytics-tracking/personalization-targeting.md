---
title: Personalization and Targeting
impact: HIGH
impactDescription: Correct personalization architecture prevents caching conflicts, avoids layout flicker, ensures privacy compliance, and delivers measurable uplift without degrading page performance
tags: personalization, targeting, contexthub, adobe-target, segments, audiences, experience-fragments, caching, privacy, a-b-testing
---

## Personalization and Targeting

AEM Cloud Service provides two complementary personalization engines: ContextHub (native AEM segmentation and client-side context) and Adobe Target (advanced testing, AI-driven optimization, and enterprise audience management). Frontend developers must understand both to implement performant, cache-friendly, privacy-respecting personalized experiences.

---

### ContextHub Architecture in AEM Cloud Service

ContextHub is a client-side JavaScript framework for storing, manipulating, and presenting context data. It enables content personalization by evaluating visitor context against defined segments.

#### Enabling ContextHub on Pages

Include the ContextHub component in the page template's `<head>` section via HTL:

```html
<!-- In the page component's head.html or customheaderlibs.html -->
<sly data-sly-resource="${'contexthub' @ resourceType='granite/contexthub/components/contexthub'}"/>
```

ContextHub segments are stored under `/conf/<site>/settings/wcm/segments`. The toolbar appears in Preview mode on the author instance when enabled.

#### ContextHub Stores

Stores are client-side data containers that persist context data and expose it via a JavaScript API. Each store is backed by a **store candidate** (a registered JavaScript class).

**Built-in store types:**

| Store Candidate | Purpose | Key Data |
|---|---|---|
| `granite.profile` | Current user profile | `displayName`, `path`, `avatar`, `authorizableId` |
| `contexthub.geolocation` | Browser geolocation (lat/lng) | `latitude`, `longitude` (requires HTTPS on Chrome 50+) |
| `contexthub.surferinfo` | Client environment details | `browser.family`, `os.name`, `device`, `viewport.width`, `viewport.height` |
| `contexthub.datetime` | Date and time | `year`, `month`, `day`, `hour`, `minute` |
| `contexthub.generic-jsonp` | External data via JSONP/JSON | Custom — configured per service |
| `aem.segmentation` | Resolved ContextHub segments | Automatically populated by SegmentEngine |
| `granite.emulators` | Device emulator profiles | Resolution, orientation, device capabilities |

> **Important:** Adobe warns: "Do not use [sample stores] directly" in production. Adapt them for your project requirements.

#### Accessing ContextHub Data from Frontend JavaScript

```javascript
// Wait for all stores to be ready
ContextHub.eventing.on(ContextHub.Constants.EVENT_ALL_STORES_READY, function() {

  // Retrieve a specific store
  var profileStore = ContextHub.getStore('profile');
  var geoStore = ContextHub.getStore('geolocation');

  // Read data from a store
  var displayName = profileStore.getItem('displayName');
  var latitude = geoStore.getItem('latitude');
  var longitude = geoStore.getItem('longitude');

  // Read entire store tree
  var allProfileData = profileStore.getTree();

  // Update store data
  profileStore.setItem('customProperty', 'value');

  // Remove data
  profileStore.removeItem('customProperty');
});
```

**Listening for store changes:**

```javascript
var geoStore = ContextHub.getStore('geolocation');

// Bind to data update events
geoStore.eventing.on(ContextHub.Constants.EVENT_DATA_UPDATE, function(event, data) {
  console.log('Geolocation updated:', data);
  updatePersonalizedContent(data);
}, 'my-geo-handler');

// One-time listener
geoStore.eventing.once(ContextHub.Constants.EVENT_DATA_UPDATE, function(event, data) {
  initializeMap(data);
}, 'my-geo-init');

// Remove listener
geoStore.eventing.off(ContextHub.Constants.EVENT_DATA_UPDATE, 'my-geo-handler');
```

**Key ContextHub constants:**

| Constant | Value | Purpose |
|---|---|---|
| `EVENT_NAMESPACE` | `"ch"` | Event namespace |
| `EVENT_ALL_STORES_READY` | `"all-stores-ready"` | All stores initialized |
| `EVENT_STORE_UPDATED` | `"store-updated"` | A store's data changed |
| `EVENT_DATA_UPDATE` | (store-level) | Specific store data updated |

**Core store methods:**

| Method | Description |
|---|---|
| `getItem(key)` | Retrieve a value by key |
| `setItem(key, value, options)` | Set a value |
| `removeItem(key, options)` | Delete a value |
| `getTree(includeInternals)` | Get the full data tree |
| `getKeys(includeInternals)` | List all keys |
| `addReference(key, anotherKey)` | Create a data reference |
| `clean()` | Clear all data |
| `reset(keepRemainingData)` | Reset to initial state |
| `pauseEventing()` / `resumeEventing()` | Control event dispatch |

#### ContextHub Segments and the Segment Engine

Segments define visitor groups based on store data. The SegmentEngine evaluates all registered segments against the current context and returns those that resolve to `true`.

```javascript
// Get all resolved (active) segments for the current visitor
var resolvedSegments = ContextHub.SegmentEngine.SegmentManager.getResolvedSegments();

resolvedSegments.forEach(function(segment) {
  console.log('Active segment:', segment.getName(), segment.getPath());
});
```

Segments are authored in the AEM Segment Editor at **Tools > Personalization > ContextHub Segments**. Each segment references store properties and evaluates conditions using comparison operators and AND/OR grouping.

#### ContextHub UI Module (Toolbar and Debugging)

The ContextHub toolbar appears in Preview mode and displays UI modules grouped into modes.

**Built-in UI module types:**

| Module Type | Linked Store | Description |
|---|---|---|
| `contexthub.browserinfo` | `surferinfo` | Browser and OS information |
| `contexthub.datetime` | `datetime` | Editable date/time picker |
| `contexthub.location` | `geolocation` | Interactive Google Map |
| `contexthub.screen-orientation` | `emulators` | Device orientation selector |
| `contexthub.tagcloud` | `tagcloud` | Page tag statistics |
| `granite.profile` | `profile` | User profile with edit capability |

**Debugging ContextHub:**

- Enable debug mode via OSGi configuration (`com.adobe.granite.contexthub`) or CRXDE
- Use the Adobe Granite ContextHub OSGi service for logging control
- Silent mode suppresses all debug output globally
- Set `com.adobe.granite.contexthub.show_ui=true` to force toolbar display

#### Creating Custom ContextHub Stores

When built-in stores are insufficient, create custom store candidates by extending a base store class.

```javascript
// File: /apps/mysite/clientlibs/clientlib-contexthub/js/weather-store.js
// Client library category: contexthub.store.weather

;(function() {
  'use strict';

  // 1. Define the store candidate constructor
  var WeatherStore = function() {};

  // 2. Inherit from a base store class
  ContextHub.Utils.inheritance.inherit(WeatherStore, ContextHub.Store.PersistedStore);

  // 3. Override init to fetch weather data
  WeatherStore.prototype.init = function(name, config) {
    // Call parent init
    WeatherStore.__superClass.init.call(this, name, config);

    // Fetch weather data based on geolocation
    var geoStore = ContextHub.getStore('geolocation');
    if (geoStore) {
      var lat = geoStore.getItem('latitude');
      var lon = geoStore.getItem('longitude');
      if (lat && lon) {
        this.fetchWeather(lat, lon);
      }
    }
  };

  WeatherStore.prototype.fetchWeather = function(lat, lon) {
    var self = this;
    fetch('/api/weather?lat=' + lat + '&lon=' + lon)
      .then(function(response) { return response.json(); })
      .then(function(data) {
        self.setItem('temperature', data.temperature);
        self.setItem('condition', data.condition);
        self.setItem('city', data.city);
      });
  };

  // 4. Register the store candidate
  ContextHub.Utils.storeCandidates.registerStoreCandidate(
    WeatherStore,
    'contexthub.weather',  // Store type identifier
    0                       // Priority (0 = default)
  );
})();
```

**Base store classes available for extension:**

| Class | Persistence | Use Case |
|---|---|---|
| `ContextHub.Store.SessionStore` | In-memory only | Temporary session data |
| `ContextHub.Store.PersistedStore` | Based on ContextHub config | General-purpose persistent store |
| `ContextHub.Store.JSONPStore` | External JSONP with caching | Third-party API integration |
| `ContextHub.Store.PersistedJSONPStore` | Persisted + JSONP | Cached external data |

---

### Adobe Target Integration

Adobe Target enables A/B testing, multivariate testing, experience targeting, and AI-driven personalization. AEM Cloud Service integrates with Target through IMS (Identity Management System) authentication.

#### Configuration Prerequisites

1. **IMS Configuration** — S2S OAuth (JWT is deprecated). Configure at **Tools > Cloud Services > Adobe Target**
2. **Adobe Launch (Experience Platform Tags)** — Required for client-side Target delivery. Install the Adobe Target and Adobe ContextHub extensions.
3. **Cloud Configuration** — Applied at `/conf/<tenant>/settings/cloudconfigs/target/`. Requires Tenant ID and Client Code (often identical for AEM CS customers).

#### at.js vs Adobe Experience Platform Web SDK (Alloy)

| Feature | at.js | Web SDK (Alloy) |
|---|---|---|
| Library | `at.js 2.x` standalone | `alloy.js` unified SDK |
| Scope | Target only | Target + Analytics + Audience Manager + more |
| A4T (Analytics for Target) | Requires SDID stitching | Natively supported, no stitching |
| Pre-hiding snippet ID | `at-body-style` | `alloy-prehiding` |
| Render methods | `applyOffer()`, `applyOffers()` | `applyPropositions` (supports set, replace, append) |
| On-device decisioning | Supported | Not currently supported |
| SPA support | `triggerView()` | Automatic via `xdm` properties |
| Migration | Legacy | Recommended for new implementations |

> **Critical:** The Web SDK is **not compatible** with the at.js pre-hiding snippet. Change the snippet ID during migration.

**at.js function mapping to Web SDK:**

```javascript
// at.js — fetch and apply offers
adobe.target.getOffer({
  mbox: 'hero-banner',
  success: function(offer) {
    adobe.target.applyOffer({ offer: offer });
  }
});

// Web SDK equivalent
alloy('sendEvent', {
  decisionScopes: ['hero-banner'],
  renderDecisions: true
});
```

#### Target Component and Experience Targeting

In AEM's page editor, authors use the **Target component** as a container for personalized content. The authoring workflow has three steps:

1. **Create** — Add experiences with targeted components and offers
2. **Target** — Map experiences to audience segments; set traffic allocation for A/B tests
3. **Goals & Settings** — Configure activity timing, priority, and success metrics (Conversion, Revenue, Engagement)

Only **Experience Fragment** components can be directly targeted in the Page Editor. Other component types must be converted via "Convert to experience fragment variation."

#### Offer Delivery Patterns

```javascript
// Pattern 1: Automatic rendering (recommended for most cases)
alloy('sendEvent', {
  renderDecisions: true,  // SDK renders offers automatically
  xdm: {
    web: {
      webPageDetails: {
        viewName: 'product-detail'  // SPA view name
      }
    }
  }
});

// Pattern 2: Manual rendering (for custom placement)
alloy('sendEvent', {
  decisionScopes: ['hero-banner', 'sidebar-promo'],
  renderDecisions: false
}).then(function(result) {
  var propositions = result.propositions || [];
  propositions.forEach(function(proposition) {
    proposition.items.forEach(function(item) {
      if (item.schema === 'https://ns.adobe.com/personalization/html-content-item') {
        document.querySelector('#hero').innerHTML = item.data.content;
      }
    });
  });
  // Notify Target that propositions were displayed
  alloy('applyPropositions', { propositions: propositions });
});
```

#### Visual Experience Composer (VEC) with AEM

The VEC allows marketers to create Target activities visually by overlaying changes on the live page. Requirements for AEM pages:

- Pages must be publicly accessible (or accessible via VEC Helper browser extension)
- at.js or Web SDK must be loaded on the page
- Consistent DOM structure and stable CSS selectors (avoid dynamic IDs)
- Content Security Policy must allow Target's iframe embedding

#### A/B Tests and Multivariate Tests

**A/B Tests** compare two or more experiences with configurable traffic allocation:

```
Experience A (Control):  40% traffic
Experience B (Variant):  40% traffic
Experience C (Variant):  20% traffic
```

**Multivariate Tests (MVT)** test combinations of changes across multiple locations on a page simultaneously.

Both require at least one goal metric. Activity types depend on the Target tenant configuration — if `xt_only` is enabled, only XT activities are available.

#### Target with AEM Experience Fragments

Experience Fragments are the modern replacement for legacy offers. Export workflow:

1. Create Experience Fragments using the **Web Variation** template
2. Apply Cloud Configuration to the fragment's folder (specifies Target workspace and format)
3. Export via the Experience Fragments console: **Select > Export to Adobe Target**

**Supported export formats:**

| Format | Use Case |
|---|---|
| HTML (default) | Web and hybrid delivery |
| JSON | Headless content delivery |
| HTML & JSON | Combined support |

**Key considerations:**
- Publish fragments before exporting — media assets remain in AEM, only references transfer to Target
- Folder-level cloud configuration cascades to child fragments
- Edits must be made in AEM, then re-exported (unlike native Target offers)
- Experience Fragment offers work across all activity types including AI-powered activities (Auto-Allocate, Auto-Target, Automated Personalization)

---

### Experience Targeting (XT) in AEM

Experience Targeting maps specific content experiences to defined audience segments. It is simpler than A/B testing — each audience sees exactly one predetermined experience.

#### Activities and Offers

Activities are organized by **brands** in the Activities console (**Personalization > Activities**). Each activity contains:

- One or more **experiences**, each with targeted content (offers)
- **Audience mappings** that link experiences to segments
- **Duration settings** (start/end dates)
- **Priority levels** (Low, Normal, High) to resolve conflicts when multiple activities target the same content area

**Targeting engine choice:**
- **AEM (ContextHub)** — Segments resolved entirely client-side
- **Adobe Target** — Segments resolved server-side by Target, then delivered to client

#### Audience Definitions (Segments)

An audience (called a "segment" in ContextHub) is a class of visitors defined by specific criteria.

**ContextHub segments** support traits based on:
- Store property comparisons (profile attributes, geolocation, browser info, datetime)
- AND/OR boolean grouping
- Script-based evaluation for complex logic

**Adobe Target audiences** support rule-based definitions:
- Mobile (device type, vendor, screen dimensions)
- Operating system
- Browser type and version
- Custom mbox parameters
- Site pages (URL matching)
- Visitor profile parameters
- Traffic sources (referrer)

Multiple rules in Target audiences combine with boolean AND logic.

#### Default vs Targeted Content Rendering

```
Page Request
  |
  ├── Is targeting active for this component?
  |     ├── NO → Render default content
  |     └── YES → Evaluate segments/audiences
  |               ├── Segment A matches → Render Experience A offer
  |               ├── Segment B matches → Render Experience B offer
  |               └── No segment matches → Render Default experience
```

The Default experience is mandatory and acts as the fallback. Targeted content is identified in the editor by a dotted border. Authors simulate different personas via the ContextHub toolbar in Preview mode.

---

### Frontend Patterns for Personalization

#### Client-Side Personalization (ContextHub + JavaScript)

The most common AEM-native approach. Page HTML is delivered with default content, then JavaScript evaluates context and swaps content on the client.

```javascript
// personalization.js — client-side content swap based on ContextHub segments
(function() {
  'use strict';

  function applyPersonalization() {
    var resolvedSegments = ContextHub.SegmentEngine.SegmentManager.getResolvedSegments();
    var segmentNames = resolvedSegments.map(function(s) { return s.getName(); });

    // Swap hero banner based on segment
    var heroEl = document.querySelector('[data-personalization="hero"]');
    if (!heroEl) return;

    if (segmentNames.indexOf('returning-customer') > -1) {
      heroEl.setAttribute('data-active-variant', 'returning');
      showVariant(heroEl, 'returning');
    } else if (segmentNames.indexOf('high-value-prospect') > -1) {
      heroEl.setAttribute('data-active-variant', 'high-value');
      showVariant(heroEl, 'high-value');
    }
    // else: default content remains visible
  }

  function showVariant(container, variantId) {
    var variants = container.querySelectorAll('[data-variant]');
    variants.forEach(function(v) {
      v.style.display = v.getAttribute('data-variant') === variantId ? '' : 'none';
    });
  }

  ContextHub.eventing.on(ContextHub.Constants.EVENT_ALL_STORES_READY, applyPersonalization);
})();
```

#### Server-Side Personalization

AEM supports server-side personalization through Sling mechanisms:

- **Sling Dynamic Include (SDI)** — Replaces component includes with SSI/ESI directives, enabling per-component caching strategies
- **Sling Resource Merger** — Overlays resource definitions based on context
- **Custom Sling Filters/Processors** — Java-based request processing that can alter content before delivery

SDI is the most relevant for frontend developers as it affects the page markup structure.

#### Hybrid Approach (Recommended for Performance)

Combine server-side static content delivery with client-side personalization for dynamic elements:

```html
<!-- Page delivered from CDN/Dispatcher with default content -->
<section class="hero" data-personalization="hero">
  <!-- Default content (always rendered server-side) -->
  <div data-variant="default">
    <h1>Welcome to Our Site</h1>
  </div>

  <!-- Variant content (hidden by default, shown via JS) -->
  <div data-variant="returning" style="display: none;">
    <h1>Welcome Back!</h1>
  </div>

  <div data-variant="high-value" style="display: none;" >
    <h1>Exclusive Offer for You</h1>
  </div>
</section>
```

This pattern keeps the page cacheable while allowing personalization. The tradeoff is that all variant HTML is delivered to every visitor.

#### Flicker Prevention

Flicker (Flash of Original Content) occurs when default content is briefly visible before personalized content loads.

**at.js pre-hiding snippet (place before at.js in `<head>`):**

```html
<style id="at-body-style">
  body { opacity: 0 !important; }
</style>
<script>
  // Auto-remove after 3 seconds if at.js fails to load
  (function(wait) {
    setTimeout(function() {
      var style = document.getElementById('at-body-style');
      if (style) { style.parentNode.removeChild(style); }
    }, wait);
  })(3000);
</script>
```

**Web SDK (Alloy) pre-hiding snippet:**

```html
<script>
  !function(e,a,n,t){
    if(e.alloy)return;
    var i=e.setTimeout(function(){a.getElementById(n)&&(a.getElementById(n).style="")},t);
    e.addEventListener("load",function(){i&&(clearTimeout(i),i=null)});
    var o=a.createElement("style");
    o.id=n;o.innerHTML="body { opacity: 0 !important }";
    a.head.appendChild(o);
  }(window,document,"alloy-prehiding",3000);
</script>
```

> **Important:** at.js and Web SDK pre-hiding snippets use different style IDs (`at-body-style` vs `alloy-prehiding`) and are **not interchangeable**.

**Optimized pre-hiding (recommended):**

Instead of hiding the entire `body`, hide only personalization containers:

```css
/* Target only elements that will be personalized */
#hero-banner, .promo-rail, [data-target-mbox] {
  opacity: 0 !important;
}
```

This preserves Core Web Vitals metrics (LCP, CLS) by keeping non-personalized content visible immediately.

#### Loading States and Fallback Content

```javascript
// Show skeleton/loading state while personalization resolves
function initPersonalizedComponent(selector, timeoutMs) {
  var el = document.querySelector(selector);
  if (!el) return;

  // Add loading state
  el.classList.add('is-loading');

  var timeout = setTimeout(function() {
    // Timeout: show default content
    el.classList.remove('is-loading');
    el.classList.add('is-default');
    console.warn('Personalization timeout for', selector);
  }, timeoutMs || 2000);

  return {
    resolve: function(html) {
      clearTimeout(timeout);
      el.innerHTML = html;
      el.classList.remove('is-loading');
      el.classList.add('is-personalized');
    },
    fallback: function() {
      clearTimeout(timeout);
      el.classList.remove('is-loading');
      el.classList.add('is-default');
    }
  };
}
```

```css
/* Loading state styles */
.is-loading {
  min-height: 200px;
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s ease-in-out infinite;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
```

---

### Caching and Personalization Conflicts

Personalized content and aggressive caching are inherently in tension. Understanding the AEM Cloud Service caching layers is essential.

#### AEM Cloud Service Caching Layers

```
Browser Cache → CDN (Fastly) → Dispatcher (Apache) → AEM Publish
```

**Default CDN behavior:**
- Responses with `Cache-Control: private` are NOT cached at CDN
- Responses setting cookies are NOT cached at CDN
- Public content: `Cache-Control: public, max-age=600, immutable`
- Authenticated content: `Cache-Control: private, max-age=600, immutable`

#### Dispatcher Caching with Personalized Content

**Rule: Never cache personalized HTML at the Dispatcher or CDN level.** Instead, cache the generic page and personalize on the client.

For mixed pages (static shell + dynamic components), use **Sling Dynamic Include (SDI)**:

```xml
<!-- OSGi configuration: org.apache.sling.dynamicinclude.Configuration -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    jcr:primaryType="sling:OsgiConfig"
    include-filter.config.enabled="{Boolean}true"
    include-filter.config.path="/content"
    include-filter.config.resource-types="[mysite/components/personalized-hero]"
    include-filter.config.include-type="SSI"
    include-filter.config.selector="nocache"
    include-filter.config.ttl=""/>
```

**Dispatcher rules for SDI:**

```
/rules {
    /0009 {
        /glob "*.nocache.html*"
        /type "deny"
    }
}

/cache {
    /enableTTL "1"
}
```

**Apache configuration for SSI:**

```apache
LoadModule include_module libexec/apache2/mod_include.so

<Directory /path/to/docroot>
    Options FollowSymLinks Includes
    AddOutputFilter INCLUDES .html
</Directory>
```

With SDI, the page shell is cached while personalized components are fetched fresh (or with a short TTL) on every request.

#### TTL-Based Caching Strategies

| Content Type | Recommended TTL | Strategy |
|---|---|---|
| Fully static pages | 600s+ (CDN default) | Standard caching |
| Pages with personalized components | Cache page shell; exclude personalized parts via SDI | SSI/ESI |
| Personalized API responses | 0–60s or `no-cache` | Short TTL or uncached |
| Client libraries (JS/CSS) | `immutable` / 30 days | Long-lived content-hash URLs |
| Experience Fragment offers | Managed by Target CDN | Target-controlled |

Use `stale-while-revalidate` to serve stale content while refreshing in the background:

```
Cache-Control: public, max-age=300, stale-while-revalidate=60
```

#### CDN-Friendly Personalization Patterns

**Pattern 1: Client-side personalization (preferred)**
Cache the full page at CDN with default content. JavaScript personalizes after page load.

**Pattern 2: Edge-side personalization (ESI)**
CDN evaluates ESI tags and fetches personalized fragments from origin. Requires CDN ESI support.

**Pattern 3: API-driven personalization**
Page loads from CDN cache. JavaScript calls a personalization API (Target, custom endpoint) for dynamic content. Offers are delivered as JSON and rendered client-side.

```javascript
// API-driven pattern with Target Web SDK
alloy('sendEvent', {
  renderDecisions: false,
  decisionScopes: ['homepage-hero']
}).then(function(result) {
  // Result comes from Target edge, not AEM Dispatcher
  var propositions = result.propositions;
  renderPersonalizedContent(propositions);
});
```

---

### Consent and Privacy

#### Opt-In / Opt-Out Patterns for Targeting

Personalization libraries (at.js, Web SDK) must not fire until the visitor grants consent.

```javascript
// Consent-aware initialization with Web SDK
alloy('configure', {
  datastreamId: 'YOUR_DATASTREAM_ID',
  orgId: 'YOUR_ORG_ID',
  defaultConsent: 'pending'  // Block all data collection until consent
});

// When visitor grants consent
function onConsentGranted() {
  alloy('setConsent', {
    consent: [{
      standard: 'Adobe',
      version: '2.0',
      value: {
        collect: { val: 'y' },
        personalize: { content: { val: 'y' } }
      }
    }]
  });
}

// When visitor declines
function onConsentDenied() {
  alloy('setConsent', {
    consent: [{
      standard: 'Adobe',
      version: '2.0',
      value: {
        collect: { val: 'n' },
        personalize: { content: { val: 'n' } }
      }
    }]
  });
}
```

**at.js opt-in pattern:**

```javascript
// Defer Target until consent is given
adobe.optIn.fetchPermissions(function() {
  if (adobe.optIn.isApproved(adobe.OptInCategories.TARGET)) {
    adobe.target.getOffer({ /* ... */ });
  } else {
    // Show default content, no tracking
    showDefaultContent();
  }
});
```

#### GDPR / Cookie Consent Integration

- ContextHub stores can use in-memory persistence (`SessionStore`) instead of cookies or localStorage when consent is not granted
- Target's `deviceIdLifetime` and `secureOnly` cookie settings should match your consent policy
- Use the `targetMigrationEnabled` option during at.js-to-Web-SDK migration to handle existing cookies
- Configure Data Streams in Adobe Experience Platform to respect consent signals

#### Anonymous vs Authenticated Targeting

| Scenario | Approach |
|---|---|
| Anonymous visitors | Cookie-based identification, ContextHub session stores, behavioral segments |
| Authenticated visitors | Profile store with CRM data, cross-device targeting via ECID |
| Consent pending | Default content only, no personalization cookies, `defaultConsent: 'pending'` |
| Consent denied | Suppress all personalization, use `SessionStore` only, no third-party calls |

---

### Anti-Patterns

**Personalization on every component.** Evaluating segments and swapping content across dozens of components per page creates excessive DOM manipulation and JavaScript execution. Limit personalization to 2-3 high-impact areas (hero, primary CTA, key promotion rail).

**Synchronous Target calls blocking render.** Never make synchronous XHR calls to Target before allowing the page to paint. Use `renderDecisions: true` with the Web SDK or asynchronous `getOffer()` with at.js. Synchronous calls block the main thread and destroy Time to Interactive.

**Caching personalized HTML at CDN.** Serving user-specific HTML from the CDN means one visitor's personalized content is shown to all subsequent visitors until cache expires. Cache only generic/default content at the CDN layer. Use `Cache-Control: private` or `no-store` for any response containing personalized markup.

**No fallback content.** If the personalization service is slow or unavailable, visitors see a blank space or loading spinner indefinitely. Always render default content server-side and progressively enhance with personalized content. Enforce JavaScript timeouts (2-3 seconds) before falling back.

**Testing only in author mode.** ContextHub toolbar behavior on author differs from publish. Personalization libraries (at.js, Web SDK) typically load only on publish. Always test personalization flows on the publish tier with Dispatcher caching active. Use the ContextHub Preview mode on author only for initial content verification.

**Ignoring CLS (Cumulative Layout Shift).** Swapping content after page load without reserving space causes layout shifts. Reserve dimensions for personalized containers using CSS `min-height` or aspect-ratio boxes. Use the `opacity` pre-hiding approach rather than `display: none` to maintain layout stability.

**Mixing at.js and Web SDK on the same page.** These libraries are not compatible. Their pre-hiding snippets conflict, and running both creates race conditions. Choose one library and migrate fully. Page-by-page migration is supported using `targetMigrationEnabled`.

**Hardcoding segment logic in JavaScript.** Duplicating segment evaluation in custom JS instead of using ContextHub SegmentEngine leads to drift between authored segments and runtime behavior. Use `ContextHub.SegmentEngine.SegmentManager.getResolvedSegments()` as the single source of truth.
