---
title: Alpine.js for AEM Components
impact: HIGH
impactDescription: Alpine.js adds reactive interactivity directly in HTL markup with zero build step — ideal for traditional AEM where full React/Preact is overkill
tags: alpine, alpinejs, htl, aem, reactive, progressive-enhancement, lightweight, x-data, x-bind
---

## Alpine.js for AEM Components

Alpine.js (~17KB) brings reactive behavior directly into HTML attributes — no build step, no virtual DOM, no component compilation. It works naturally with AEM's server-rendered HTL because you add behavior to existing markup instead of replacing it.

---

### 1. When to Use Alpine.js in AEM

| Criterion | Alpine.js | Preact/React | Vanilla JS |
|-----------|-----------|-------------|------------|
| Interactive islands in HTL pages | Best fit | Good (more setup) | Verbose |
| Build step required | No (CDN or ClientLib) | Yes (Webpack) | Optional |
| Learning curve | Minimal (HTML attributes) | Moderate (JSX, hooks) | Low |
| Bundle size | ~17KB (single file) | 3-40KB + app code | 0KB |
| Headless / SPA Editor | No | React only | No |
| Server-side rendering | Not needed (enhances HTL) | Needed for hydration | Not needed |
| Complex state management | Limited (x-store) | React Context / Signals | Manual |

**Rule**: Use Alpine.js when you need reactive behavior on HTL-rendered pages without a build pipeline. If you already have Webpack set up with React/Preact, stick with those.

---

### 2. Installation in AEM

**Option A — via ClientLib (recommended for production):**

```
ui.apps/src/main/content/jcr_root/apps/mysite/clientlibs/clientlib-alpine/
├── .content.xml
├── js.txt
└── js/
    └── alpine.min.js
```

```xml
<!-- .content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0"
  jcr:primaryType="cq:ClientLibraryFolder"
  categories="[mysite.alpine]"
  allowProxy="{Boolean}true"/>
```

```
# js.txt
#base=js
alpine.min.js
```

**Load with `defer` in HTL page template:**

```html
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html" />
<sly data-sly-call="${clientlib.js @ categories='mysite.alpine'}" />
```

**Option B — CDN (dev/prototyping only):**

```html
<script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
```

---

### 3. Basic Patterns with HTL

**Accordion:**

```html
<!-- accordion.html -->
<sly data-sly-use.model="com.mysite.models.AccordionModel" />
<div class="cmp-accordion" x-data="{ active: null }">
  <sly data-sly-list.item="${model.items}">
    <div class="cmp-accordion__item">
      <button
        class="cmp-accordion__header"
        @click="active = active === ${itemList.index} ? null : ${itemList.index}"
        :aria-expanded="active === ${itemList.index}"
        type="button">
        ${item.title}
        <span class="cmp-accordion__icon" :class="{ 'cmp-accordion__icon--open': active === ${itemList.index} }"></span>
      </button>
      <div
        class="cmp-accordion__panel"
        x-show="active === ${itemList.index}"
        x-collapse>
        <sly data-sly-resource="${item.resource}" />
      </div>
    </div>
  </sly>
</div>
```

**Tabs:**

```html
<!-- tabs.html -->
<sly data-sly-use.model="com.mysite.models.TabsModel" />
<div class="cmp-tabs" x-data="{ activeTab: 0 }">
  <ol class="cmp-tabs__tablist" role="tablist">
    <sly data-sly-list.tab="${model.tabs}">
      <li
        class="cmp-tabs__tab"
        :class="{ 'cmp-tabs__tab--active': activeTab === ${tabList.index} }"
        @click="activeTab = ${tabList.index}"
        @keydown.arrow-right="activeTab = Math.min(activeTab + 1, ${tabList.count - 1})"
        @keydown.arrow-left="activeTab = Math.max(activeTab - 1, 0)"
        role="tab"
        :aria-selected="activeTab === ${tabList.index}"
        :tabindex="activeTab === ${tabList.index} ? 0 : -1">
        ${tab.title}
      </li>
    </sly>
  </ol>
  <sly data-sly-list.tab="${model.tabs}">
    <div
      class="cmp-tabs__tabpanel"
      x-show="activeTab === ${tabList.index}"
      role="tabpanel">
      <sly data-sly-resource="${tab.resource}" />
    </div>
  </sly>
</div>
```

---

### 4. Search with API Integration

```html
<!-- search.html -->
<div class="cmp-search"
  x-data="{
    query: '',
    results: [],
    loading: false,
    async search() {
      if (this.query.length < 3) { this.results = []; return; }
      this.loading = true;
      try {
        const res = await fetch('/bin/mysite/search?q=' + encodeURIComponent(this.query));
        const data = await res.json();
        this.results = data.results || [];
      } catch (e) {
        console.error('Search failed', e);
        this.results = [];
      } finally {
        this.loading = false;
      }
    }
  }">
  <form class="cmp-search__form" @submit.prevent="search()">
    <input
      class="cmp-search__input"
      type="search"
      x-model="query"
      @input.debounce.300ms="search()"
      placeholder="Search..." />
  </form>

  <div class="cmp-search__loading" x-show="loading">Searching...</div>

  <ul class="cmp-search__results" x-show="results.length > 0">
    <template x-for="result in results" :key="result.path">
      <li class="cmp-search__result">
        <a :href="result.path" x-text="result.title"></a>
        <p x-text="result.description"></p>
      </li>
    </template>
  </ul>

  <p class="cmp-search__empty"
    x-show="!loading && query.length >= 3 && results.length === 0">
    No results found.
  </p>
</div>
```

---

### 5. Global State with Alpine.store

Share state across multiple components on the page:

```html
<!-- In page head or ClientLib -->
<script>
  document.addEventListener('alpine:init', () => {
    Alpine.store('cart', {
      items: JSON.parse(localStorage.getItem('mysite:cart') || '[]'),
      get count() { return this.items.length; },
      get total() { return this.items.reduce((s, i) => s + i.price * i.qty, 0); },
      add(product) {
        const existing = this.items.find(i => i.sku === product.sku);
        if (existing) { existing.qty++; }
        else { this.items.push({ ...product, qty: 1 }); }
        this.persist();
      },
      remove(sku) {
        this.items = this.items.filter(i => i.sku !== sku);
        this.persist();
      },
      persist() {
        localStorage.setItem('mysite:cart', JSON.stringify(this.items));
      }
    });

    Alpine.store('auth', {
      user: null,
      isLoggedIn: false,
      async checkSession() {
        try {
          const res = await fetch('/bin/mysite/session');
          const data = await res.json();
          this.user = data.user || null;
          this.isLoggedIn = !!this.user;
        } catch { /* not logged in */ }
      }
    });
  });
</script>
```

**Usage in any component:**

```html
<!-- Cart badge in header -->
<span class="cmp-header__cart-badge"
  x-data
  x-text="$store.cart.count"
  x-show="$store.cart.count > 0">
</span>

<!-- Product card with add-to-cart -->
<sly data-sly-use.model="com.mysite.models.ProductCard" />
<div class="cmp-product-card" x-data>
  <h3>${model.title}</h3>
  <p>${model.formattedPrice}</p>
  <button
    @click="$store.cart.add({ sku: '${model.sku}', title: '${model.title}', price: ${model.price} })">
    Add to Cart
  </button>
</div>
```

---

### 6. Alpine.js Plugins for AEM

```html
<!-- Load plugins before Alpine core -->
<script defer src="/etc.clientlibs/mysite/clientlibs/clientlib-alpine/js/collapse.min.js"></script>
<script defer src="/etc.clientlibs/mysite/clientlibs/clientlib-alpine/js/intersect.min.js"></script>
<script defer src="/etc.clientlibs/mysite/clientlibs/clientlib-alpine/js/alpine.min.js"></script>
```

| Plugin | Use Case | AEM Example |
|--------|----------|-------------|
| `x-collapse` | Smooth accordion/dropdown animation | Accordion panels |
| `x-intersect` | Lazy load on scroll into view | Below-fold images, analytics tracking |
| `x-mask` | Input formatting | Phone number, credit card fields |
| `@alpinejs/focus` | Focus trap | Modal dialogs |
| `@alpinejs/persist` | Persist state to localStorage | Theme preference, cart |

```html
<!-- Lazy image with x-intersect -->
<div x-data="{ loaded: false }" x-intersect.once="loaded = true">
  <img
    :src="loaded ? '${model.imageSrc}' : ''"
    :loading="loaded ? 'eager' : 'lazy'"
    alt="${model.imageAlt}"
    width="${model.imageWidth}"
    height="${model.imageHeight}" />
</div>
```

---

### 7. Modal / Dialog Pattern

```html
<!-- modal.html -->
<div class="cmp-modal"
  x-data="{ open: false }"
  @keydown.escape.window="open = false">

  <button @click="open = true" class="cmp-modal__trigger">
    ${model.triggerLabel}
  </button>

  <div class="cmp-modal__backdrop"
    x-show="open"
    x-transition:enter="cmp-modal__backdrop--entering"
    x-transition:leave="cmp-modal__backdrop--leaving"
    @click="open = false">
  </div>

  <div class="cmp-modal__content"
    x-show="open"
    x-transition
    x-trap.noscroll="open"
    role="dialog"
    aria-modal="true"
    @click.stop>
    <button @click="open = false" class="cmp-modal__close" aria-label="Close">×</button>
    <sly data-sly-resource="${model.contentResource}" />
  </div>
</div>
```

---

### 8. Author Mode Considerations

Alpine.js behavior during AEM author editing:

```html
<sly data-sly-use.wcm="com.day.cq.wcm.commons.WCMUtils" />
<div class="cmp-tabs"
  x-data="{ activeTab: 0 }"
  data-sly-test="${!wcmmode.edit}">
  <!-- Interactive tabs in publish mode -->
</div>
<div class="cmp-tabs cmp-tabs--edit-mode"
  data-sly-test="${wcmmode.edit}">
  <!-- All panels visible in edit mode for authoring -->
  <sly data-sly-list.tab="${model.tabs}">
    <div class="cmp-tabs__tabpanel">
      <sly data-sly-resource="${tab.resource}" />
    </div>
  </sly>
</div>
```

---

### Anti-Patterns

- **Complex app logic in x-data attributes** — extract to reusable Alpine.data() components or Alpine.store() for anything beyond ~10 lines
- **Alpine.js for SPA routing** — Alpine doesn't do client-side routing; use React/Next.js for SPAs
- **Loading Alpine before HTL renders** — use `defer`; Alpine auto-initializes on DOMContentLoaded
- **Not escaping HTL in x-data** — use `@ context='scriptString'` for dynamic values in Alpine attributes
- **Mixing Alpine and jQuery** — both manipulate DOM; pick one
- **Heavy state in x-data** — for complex cross-component state, use Alpine.store(); for truly complex apps, use React
- **No fallback for JS-disabled users** — critical content should be visible in the HTL without Alpine
