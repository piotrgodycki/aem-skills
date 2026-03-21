---
title: Vanilla JavaScript Patterns for EDS Block Development
impact: HIGH
impactDescription: EDS blocks rely on vanilla JS exclusively — using correct patterns directly impacts performance, maintainability, and Lighthouse scores
tags: eds, vanilla-js, dom-manipulation, events, async, performance, blocks, patterns
---

## Vanilla JavaScript Patterns for EDS Block Development

EDS blocks use vanilla JavaScript exclusively — no frameworks, no jQuery, no build step. Every pattern here is chosen for EDS block context: scoped DOM manipulation inside a `decorate(block)` function, performance-first loading, and modern browser targets.

---

### 1. DOM Selection and Traversal

#### querySelector vs querySelectorAll

Always scope queries to the block element, never to `document`:

**Correct — scoped to block:**

```javascript
export default function decorate(block) {
  const heading = block.querySelector('h2');
  const images = block.querySelectorAll('picture');
  const firstLink = block.querySelector('a');
}
```

**Incorrect — querying the whole document:**

```javascript
export default function decorate(block) {
  // Matches elements outside this block instance
  const heading = document.querySelector('h2');
  const images = document.querySelectorAll('picture');
}
```

#### :scope Selector for Direct Children

Use `:scope` to select only direct children of the block, which is essential when blocks contain nested structures:

**Correct — direct children only:**

```javascript
export default function decorate(block) {
  // Select only the top-level row divs, not nested ones
  const rows = block.querySelectorAll(':scope > div');
  const firstRow = block.querySelector(':scope > div:first-child');

  rows.forEach((row) => {
    const cols = row.querySelectorAll(':scope > div');
    cols.forEach((col, i) => {
      col.classList.add(`col-${i + 1}`);
    });
  });
}
```

**Incorrect — selects all nested divs too:**

```javascript
export default function decorate(block) {
  // This grabs every div at every level inside the block
  const rows = block.querySelectorAll('div');
}
```

#### Traversal Methods

Use `closest()`, `matches()`, and sibling/parent properties for efficient DOM navigation:

```javascript
export default function decorate(block) {
  block.addEventListener('click', (e) => {
    // Walk up from click target to find the clickable row
    const row = e.target.closest('.card-row');
    if (!row) return;

    // Check if the clicked element is a specific type
    if (e.target.matches('a.cta')) {
      // Handle CTA click
    }
  });

  // Sibling traversal for sequential layouts
  const hero = block.querySelector('.hero-content');
  const caption = hero?.nextElementSibling;
  const wrapper = hero?.parentElement;
}
```

#### NodeList vs HTMLCollection (Static vs Live)

`querySelectorAll` returns a static `NodeList` — safe to iterate while modifying the DOM. `getElementsBy*` returns a live `HTMLCollection` — it changes as the DOM changes, causing bugs during iteration.

**Correct — static NodeList, safe to modify during iteration:**

```javascript
export default function decorate(block) {
  const paragraphs = block.querySelectorAll('p');
  paragraphs.forEach((p) => {
    if (!p.textContent.trim()) {
      p.remove(); // Safe — NodeList doesn't change
    }
  });
}
```

**Incorrect — live HTMLCollection mutates during loop:**

```javascript
export default function decorate(block) {
  const divs = block.getElementsByTagName('div');
  // BUG: as you remove items, the collection shrinks and indices shift
  for (let i = 0; i < divs.length; i++) {
    divs[i].remove();
  }
}
```

#### Spreading NodeList for Array Methods

`NodeList` supports `forEach` but not `map`, `filter`, `find`, etc. Spread into an array when you need those:

```javascript
export default function decorate(block) {
  const rows = [...block.querySelectorAll(':scope > div')];

  const textRows = rows.filter((row) => !row.querySelector('picture'));
  const imageRow = rows.find((row) => row.querySelector('picture'));

  const labels = rows.map((row) => row.querySelector('h3')?.textContent ?? '');
}
```

---

### 2. DOM Manipulation (Performance-First)

#### DocumentFragment for Batch Insertions

When adding multiple elements, use a `DocumentFragment` to batch them into a single DOM write:

**Correct — single reflow with fragment:**

```javascript
export default function decorate(block) {
  const items = ['Feature A', 'Feature B', 'Feature C'];
  const list = document.createElement('ul');
  const fragment = document.createDocumentFragment();

  items.forEach((text) => {
    const li = document.createElement('li');
    li.textContent = text;
    fragment.append(li);
  });

  list.append(fragment); // Single DOM insertion
  block.append(list);
}
```

**Incorrect — causes reflow on every iteration:**

```javascript
export default function decorate(block) {
  const items = ['Feature A', 'Feature B', 'Feature C'];
  const list = document.createElement('ul');
  block.append(list); // Attached to DOM too early

  items.forEach((text) => {
    const li = document.createElement('li');
    li.textContent = text;
    list.append(li); // Reflow on each append because list is already in the DOM
  });
}
```

#### textContent vs innerHTML vs innerText

| Property | Use When | Parses HTML? | XSS Risk | Performance |
|---|---|---|---|---|
| `textContent` | Setting/reading plain text | No | None | Fastest |
| `innerHTML` | Setting static trusted markup | Yes | **Yes** | Slower (parses) |
| `innerText` | Rarely — reads visible text only | No | None | Slowest (triggers reflow) |

**Correct — textContent for plain text:**

```javascript
const label = document.createElement('span');
label.textContent = userProvidedText; // Safe even with malicious input
```

**Correct — innerHTML for static trusted markup:**

```javascript
// Only with hardcoded strings, never user input
block.querySelector('.icon-container').innerHTML = `
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24">
    <path d="M12 2L2 22h20L12 2z"/>
  </svg>
`;
```

**Incorrect — innerHTML with user content (XSS vulnerability):**

```javascript
// NEVER do this — user content could contain <script> tags or event handlers
const name = getUserInput();
block.innerHTML = `<h2>Welcome, ${name}</h2>`;
```

#### classList API

Use `classList` methods instead of string manipulation on `className`:

```javascript
export default function decorate(block) {
  const card = block.querySelector('.card');

  card.classList.add('active', 'highlighted');  // Add multiple classes
  card.classList.remove('loading');             // Remove a class
  card.classList.toggle('expanded');            // Toggle on/off
  card.classList.replace('old-class', 'new');   // Atomic replace

  if (card.classList.contains('featured')) {
    // Conditional logic based on class presence
  }
}
```

**Incorrect — string manipulation on className:**

```javascript
// Fragile, error-prone, and harder to read
card.className = card.className.replace('old-class', 'new');
card.className += ' active';
```

#### dataset API for Data Attributes

Use `dataset` to read and write `data-*` attributes cleanly:

```javascript
export default function decorate(block) {
  // Read: data-slide-index="3" -> block.dataset.slideIndex
  const index = parseInt(block.dataset.slideIndex ?? '0', 10);

  // Write
  block.dataset.state = 'loaded';
  block.dataset.itemCount = '5';
  // Results in: data-state="loaded" data-item-count="5"
}
```

#### createElement + append vs innerHTML

Use `createElement` when elements need event listeners or dynamic content. Use `innerHTML` for static structural markup.

**Correct — createElement for interactive elements:**

```javascript
export default function decorate(block) {
  const button = document.createElement('button');
  button.textContent = 'Load More';
  button.classList.add('load-more');
  button.addEventListener('click', () => loadNextPage(block));
  block.append(button);
}
```

**Correct — innerHTML for static structure (then query for event targets):**

```javascript
export default function decorate(block) {
  const wrapper = document.createElement('div');
  wrapper.classList.add('tabs-wrapper');
  wrapper.innerHTML = `
    <div class="tab-list" role="tablist">
      <button role="tab" aria-selected="true">Tab 1</button>
      <button role="tab" aria-selected="false">Tab 2</button>
    </div>
    <div class="tab-panels"></div>
  `;

  // Attach events after setting innerHTML
  wrapper.querySelectorAll('[role="tab"]').forEach((tab) => {
    tab.addEventListener('click', () => switchTab(tab, wrapper));
  });

  block.append(wrapper);
}
```

#### Avoid Forced Reflows — Read Then Write

Interleaving DOM reads and writes forces the browser to recalculate layout repeatedly. Batch all reads first, then all writes.

**Correct — batch reads then writes:**

```javascript
export default function decorate(block) {
  // READ phase — gather all measurements
  const cards = [...block.querySelectorAll('.card')];
  const heights = cards.map((card) => card.offsetHeight);
  const maxHeight = Math.max(...heights);

  // WRITE phase — apply changes
  cards.forEach((card) => {
    card.style.height = `${maxHeight}px`;
  });
}
```

**Incorrect — interleaved reads and writes (layout thrashing):**

```javascript
export default function decorate(block) {
  const cards = block.querySelectorAll('.card');
  cards.forEach((card) => {
    const height = card.offsetHeight;        // READ — forces layout
    card.style.height = `${height + 20}px`;  // WRITE — invalidates layout
    // Next iteration's READ forces another layout recalc
  });
}
```

**Using requestAnimationFrame for deferred DOM writes:**

```javascript
export default function decorate(block) {
  // Defer visual updates to next frame to avoid blocking current paint
  requestAnimationFrame(() => {
    block.querySelectorAll('.card').forEach((card) => {
      card.classList.add('animate-in');
    });
  });
}
```

---

### 3. Event Handling

#### addEventListener with Options

Always use the options object form for clarity. Key options:

```javascript
export default function decorate(block) {
  // once: auto-removes after first invocation
  block.querySelector('.intro-video').addEventListener('click', () => {
    loadVideoPlayer(block);
  }, { once: true });

  // passive: tells browser this won't call preventDefault (enables optimizations)
  block.addEventListener('touchstart', handleTouch, { passive: true });

  // capture: listen during capture phase (parent fires before child)
  block.addEventListener('focus', handleFocus, { capture: true });
}
```

#### Event Delegation

Use a single listener on the block instead of one per child. Essential for dynamically added content and for reducing memory usage:

**Correct — event delegation:**

```javascript
export default function decorate(block) {
  block.addEventListener('click', (e) => {
    const tab = e.target.closest('[role="tab"]');
    if (tab) {
      switchTab(tab, block);
      return;
    }

    const closeBtn = e.target.closest('.close-button');
    if (closeBtn) {
      closePanel(block);
      return;
    }
  });
}
```

**Incorrect — individual listeners on each element:**

```javascript
export default function decorate(block) {
  // Creates N listeners instead of 1 — wasteful and breaks for dynamic content
  block.querySelectorAll('[role="tab"]').forEach((tab) => {
    tab.addEventListener('click', () => switchTab(tab, block));
  });
}
```

Note: individual listeners are acceptable for a small, fixed number of elements (2-3 buttons). Delegation is better when there are many elements or when content changes dynamically.

#### Passive Event Listeners for Scroll and Touch

Scroll and touch listeners block the browser compositor thread unless marked passive. Always use passive for these:

```javascript
export default function decorate(block) {
  // Passive scroll listener — cannot call preventDefault, but enables smooth scrolling
  window.addEventListener('scroll', () => {
    const scrolled = window.scrollY > 100;
    block.classList.toggle('scrolled', scrolled);
  }, { passive: true });

  // If you MUST call preventDefault (rare), explicitly set passive: false
  block.querySelector('.custom-scroll').addEventListener('wheel', (e) => {
    e.preventDefault();
    customScrollLogic(e);
  }, { passive: false });
}
```

#### AbortController for Cleaning Up Listeners

Use `AbortController` to remove multiple listeners at once — cleaner than tracking individual references:

```javascript
export default function decorate(block) {
  const controller = new AbortController();
  const { signal } = controller;

  window.addEventListener('resize', () => handleResize(block), { signal });
  window.addEventListener('scroll', () => handleScroll(block), { signal, passive: true });
  document.addEventListener('keydown', (e) => handleKeys(e, block), { signal });

  // Clean up all listeners at once (e.g., when block is removed)
  block.dataset.cleanup = 'true';
  const observer = new MutationObserver(() => {
    if (!document.contains(block)) {
      controller.abort(); // Removes all three listeners
      observer.disconnect();
    }
  });
  observer.observe(block.parentElement, { childList: true });
}
```

#### Custom Events for Block-to-Block Communication

Use Custom Events on `document` to coordinate between blocks:

```javascript
// In the filter block
export default function decorate(block) {
  block.addEventListener('change', (e) => {
    const filter = e.target.value;
    document.dispatchEvent(new CustomEvent('filter:change', {
      detail: { category: filter },
    }));
  });
}

// In the card-list block
export default function decorate(block) {
  document.addEventListener('filter:change', (e) => {
    const { category } = e.detail;
    block.querySelectorAll('.card').forEach((card) => {
      card.hidden = category !== 'all' && card.dataset.category !== category;
    });
  });
}
```

---

### 4. Async Patterns (Without Blocking LCP)

#### Why NOT to Use async decorate() for Above-Fold Blocks

EDS waits for `decorate()` to resolve before marking the block as loaded. An `async` decorate with `await` delays LCP paint for above-fold content.

**Incorrect — async decorate blocks rendering:**

```javascript
// This delays page paint until the fetch completes
export default async function decorate(block) {
  const resp = await fetch('/api/data.json');
  const data = await resp.json();
  renderCards(block, data);
}
```

**Correct — fire and forget, or use IntersectionObserver:**

```javascript
// Option 1: Non-blocking async (no await at top level)
export default function decorate(block) {
  // Show skeleton/placeholder immediately
  block.classList.add('loading');

  fetch('/api/data.json')
    .then((resp) => resp.json())
    .then((data) => {
      renderCards(block, data);
      block.classList.remove('loading');
    });
}

// Option 2: Lazy init for below-fold blocks
export default function decorate(block) {
  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) {
      observer.disconnect();
      initBlock(block);
    }
  });
  observer.observe(block);
}
```

#### IntersectionObserver for Lazy Initialization

Defer heavy work until the block is actually visible:

```javascript
export default function decorate(block) {
  // Minimal setup — runs immediately
  block.classList.add('carousel');

  // Heavy setup — deferred until visible
  const observer = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        observer.disconnect();
        initCarousel(block); // Load slides, set up timers, bind events
      }
    });
  }, { rootMargin: '200px' }); // Start loading slightly before visible

  observer.observe(block);
}
```

#### Dynamic import() for Heavy Libraries

Load large dependencies only when needed:

```javascript
export default function decorate(block) {
  block.querySelector('.play-button')?.addEventListener('click', async () => {
    // Only loads the video player library when user clicks play
    const { default: initPlayer } = await import('./video-player.js');
    initPlayer(block);
  }, { once: true });
}
```

#### requestIdleCallback for Non-Urgent Work

Schedule non-critical work during browser idle periods:

```javascript
export default function decorate(block) {
  // Critical: render visible content immediately
  renderVisibleCards(block);

  // Non-critical: preload next page of results during idle time
  requestIdleCallback(() => {
    prefetchNextPage(block);
  }, { timeout: 2000 }); // Ensure it runs within 2 seconds max
}
```

#### queueMicrotask for Batching

Use `queueMicrotask` to defer work to the end of the current microtask queue — runs before the next paint:

```javascript
export default function decorate(block) {
  let pendingUpdate = false;

  function scheduleUpdate() {
    if (pendingUpdate) return;
    pendingUpdate = true;
    queueMicrotask(() => {
      updateLayout(block);
      pendingUpdate = false;
    });
  }

  // Multiple calls in the same tick batch into one update
  block.querySelectorAll('.item').forEach(() => scheduleUpdate());
}
```

#### fetch with AbortController

Cancel in-flight requests when they are no longer needed:

```javascript
export default function decorate(block) {
  let controller;

  async function search(query) {
    // Abort previous request
    controller?.abort();
    controller = new AbortController();

    try {
      const resp = await fetch(`/search?q=${encodeURIComponent(query)}`, {
        signal: controller.signal,
      });
      const results = await resp.json();
      renderResults(block, results);
    } catch (err) {
      if (err.name !== 'AbortError') throw err;
      // AbortError is expected when cancelling — ignore it
    }
  }

  block.querySelector('input').addEventListener('input', (e) => {
    search(e.target.value);
  });
}
```

#### Promise.allSettled for Parallel Non-Critical Fetches

When fetching multiple independent data sources, use `Promise.allSettled` so one failure does not break everything:

```javascript
export default function decorate(block) {
  const observer = new IntersectionObserver(async (entries) => {
    if (!entries[0].isIntersecting) return;
    observer.disconnect();

    const results = await Promise.allSettled([
      fetch('/api/testimonials.json').then((r) => r.json()),
      fetch('/api/stats.json').then((r) => r.json()),
      fetch('/api/partners.json').then((r) => r.json()),
    ]);

    const [testimonials, stats, partners] = results.map((r) =>
      r.status === 'fulfilled' ? r.value : null
    );

    // Render whatever succeeded — gracefully handle failures
    if (testimonials) renderTestimonials(block, testimonials);
    if (stats) renderStats(block, stats);
    if (partners) renderPartners(block, partners);
  });

  observer.observe(block);
}
```

---

### 5. Modern JS Features (Supported in EDS)

EDS targets modern evergreen browsers — no IE11 polyfills needed. Use these features freely.

#### Optional Chaining and Nullish Coalescing

```javascript
export default function decorate(block) {
  // Optional chaining — short-circuits to undefined if any part is null/undefined
  const altText = block.querySelector('picture img')?.alt;
  const href = block.querySelector('a')?.href;
  block.querySelector('.subtitle')?.classList.add('styled');

  // Nullish coalescing — fallback only for null/undefined (not '' or 0)
  const heading = block.querySelector('h2')?.textContent ?? 'Default Title';
  const columns = parseInt(block.dataset.columns ?? '3', 10);
}
```

**Incorrect — using || instead of ?? (treats '' and 0 as falsy):**

```javascript
// BUG: if textContent is '' (empty string), this falls back to 'Default'
const heading = block.querySelector('h2')?.textContent || 'Default Title';

// BUG: if data-columns="0", this becomes 3
const columns = parseInt(block.dataset.columns || '3', 10);
```

#### Destructuring for Cleaner Block Processing

```javascript
export default function decorate(block) {
  // Destructure row structure
  const rows = [...block.querySelectorAll(':scope > div')];
  const [headerRow, ...contentRows] = rows;

  // Destructure columns within a row
  contentRows.forEach((row) => {
    const [imageCol, textCol] = [...row.querySelectorAll(':scope > div')];
    imageCol?.classList.add('card-image');
    textCol?.classList.add('card-body');
  });
}
```

#### Template Literals for Dynamic Content

```javascript
function buildCard(title, description, href) {
  const card = document.createElement('div');
  card.classList.add('card');
  card.innerHTML = `
    <h3>${title}</h3>
    <p>${description}</p>
    ${href ? `<a href="${href}" class="cta">Learn More</a>` : ''}
  `;
  return card;
}
```

Note: only use template literals in `innerHTML` with trusted data (content from AEM, not user input).

#### Array Methods for Block Data Processing

```javascript
export default function decorate(block) {
  const rows = [...block.querySelectorAll(':scope > div')];

  // map: transform rows into data objects
  const items = rows.map((row) => {
    const cols = [...row.querySelectorAll(':scope > div')];
    return {
      image: cols[0]?.querySelector('picture'),
      title: cols[1]?.querySelector('h3')?.textContent,
      link: cols[1]?.querySelector('a')?.href,
    };
  });

  // filter: keep only items with images
  const withImages = items.filter((item) => item.image);

  // find: get the first featured item
  const featured = items.find((item) => item.title?.includes('Featured'));

  // some / every: check conditions
  const hasLinks = items.some((item) => item.link);
  const allHaveImages = items.every((item) => item.image);

  // flatMap: flatten nested structures
  const allLinks = rows.flatMap((row) => [...row.querySelectorAll('a')]);
}
```

#### Set and Map for Efficient Lookups

```javascript
export default function decorate(block) {
  // Set for unique values and O(1) lookup
  const activeCategories = new Set();
  block.querySelectorAll('.filter.active').forEach((f) => {
    activeCategories.add(f.dataset.category);
  });

  block.querySelectorAll('.card').forEach((card) => {
    card.hidden = !activeCategories.has(card.dataset.category);
  });

  // Map for key-value associations (preserves insertion order, any key type)
  const tabPanels = new Map();
  block.querySelectorAll('[role="tab"]').forEach((tab, i) => {
    const panel = block.querySelectorAll('[role="tabpanel"]')[i];
    tabPanels.set(tab, panel);
  });

  tabPanels.forEach((panel, tab) => {
    tab.addEventListener('click', () => activatePanel(panel));
  });
}
```

#### Loop Styles — When to Use Each

```javascript
export default function decorate(block) {
  const items = [...block.querySelectorAll('.item')];

  // for...of — best for iteration when you need break/continue
  for (const item of items) {
    if (item.classList.contains('stop')) break;
    item.classList.add('processed');
  }

  // forEach — best for simple side effects with no break needed
  items.forEach((item) => item.classList.add('styled'));

  // for loop — best when you need the index for logic (not just labeling)
  for (let i = 0; i < items.length; i++) {
    items[i].style.animationDelay = `${i * 100}ms`;
  }
}
```

#### Logical Assignment Operators

```javascript
export default function decorate(block) {
  // ??= assign only if null or undefined
  block.dataset.page ??= '1';

  // ||= assign only if falsy
  let label = block.querySelector('.label');
  label ||= createDefaultLabel();

  // &&= assign only if truthy
  let config = getBlockConfig(block);
  config &&= { ...config, initialized: true };
}
```

#### structuredClone for Deep Copying

```javascript
// Deep copy complex objects without JSON.parse(JSON.stringify()) quirks
const originalConfig = { filters: [{ name: 'size', values: ['S', 'M'] }] };
const clonedConfig = structuredClone(originalConfig);
clonedConfig.filters[0].values.push('L'); // Does not affect original
```

---

### 6. Common EDS Block Patterns

#### Decorating Tables into Semantic HTML

EDS delivers block content as a table-like div structure. Decorate it into semantic markup:

```javascript
export default function decorate(block) {
  const rows = [...block.querySelectorAll(':scope > div')];
  const [headerRow, ...bodyRows] = rows;

  const table = document.createElement('table');
  table.classList.add('data-table');

  // Build thead
  const thead = document.createElement('thead');
  const headerCells = [...headerRow.querySelectorAll(':scope > div')];
  thead.innerHTML = `<tr>${headerCells.map((c) => `<th>${c.textContent}</th>`).join('')}</tr>`;
  table.append(thead);

  // Build tbody
  const tbody = document.createElement('tbody');
  const fragment = document.createDocumentFragment();

  bodyRows.forEach((row) => {
    const tr = document.createElement('tr');
    const cells = [...row.querySelectorAll(':scope > div')];
    cells.forEach((cell) => {
      const td = document.createElement('td');
      td.textContent = cell.textContent;
      tr.append(td);
    });
    fragment.append(tr);
  });

  tbody.append(fragment);
  table.append(tbody);

  block.textContent = ''; // Clear original content
  block.append(table);
}
```

#### Building Picture Elements with Sources

Create responsive images with proper source sets:

```javascript
function createOptimizedPicture(src, alt = '', eager = false, breakpoints = [{ media: '(min-width: 600px)', width: 750 }, { width: 400 }]) {
  const picture = document.createElement('picture');

  breakpoints.forEach((bp) => {
    const source = document.createElement('source');
    if (bp.media) source.setAttribute('media', bp.media);
    source.setAttribute('type', 'image/webp');
    source.setAttribute('srcset', `${src}?width=${bp.width}&format=webply&optimize=medium`);
    picture.append(source);
  });

  const img = document.createElement('img');
  img.setAttribute('src', `${src}?width=${breakpoints[breakpoints.length - 1].width}&format=webply&optimize=medium`);
  img.setAttribute('alt', alt);
  img.setAttribute('loading', eager ? 'eager' : 'lazy');
  picture.append(img);

  return picture;
}
```

Note: EDS provides `createOptimizedPicture` in `aem.js` — use the built-in version when available.

#### Accessible Tab Interface

```javascript
export default function decorate(block) {
  const rows = [...block.querySelectorAll(':scope > div')];
  const tabList = document.createElement('div');
  tabList.setAttribute('role', 'tablist');

  const panelContainer = document.createElement('div');
  panelContainer.classList.add('tab-panels');

  rows.forEach((row, i) => {
    const [labelCol, contentCol] = [...row.querySelectorAll(':scope > div')];
    const id = `tab-${i}`;

    // Tab button
    const tab = document.createElement('button');
    tab.setAttribute('role', 'tab');
    tab.setAttribute('aria-selected', i === 0 ? 'true' : 'false');
    tab.setAttribute('aria-controls', `panel-${id}`);
    tab.setAttribute('id', id);
    tab.setAttribute('tabindex', i === 0 ? '0' : '-1');
    tab.textContent = labelCol.textContent;
    tabList.append(tab);

    // Panel
    const panel = document.createElement('div');
    panel.setAttribute('role', 'tabpanel');
    panel.setAttribute('id', `panel-${id}`);
    panel.setAttribute('aria-labelledby', id);
    panel.hidden = i !== 0;
    panel.append(...contentCol.childNodes);
    panelContainer.append(panel);
  });

  // Keyboard navigation
  tabList.addEventListener('keydown', (e) => {
    const tabs = [...tabList.querySelectorAll('[role="tab"]')];
    const current = tabs.indexOf(e.target);
    let next;

    if (e.key === 'ArrowRight') next = (current + 1) % tabs.length;
    else if (e.key === 'ArrowLeft') next = (current - 1 + tabs.length) % tabs.length;
    else if (e.key === 'Home') next = 0;
    else if (e.key === 'End') next = tabs.length - 1;
    else return;

    e.preventDefault();
    tabs[next].focus();
    tabs[next].click();
  });

  // Tab activation
  tabList.addEventListener('click', (e) => {
    const tab = e.target.closest('[role="tab"]');
    if (!tab) return;

    tabList.querySelectorAll('[role="tab"]').forEach((t) => {
      t.setAttribute('aria-selected', 'false');
      t.setAttribute('tabindex', '-1');
    });
    tab.setAttribute('aria-selected', 'true');
    tab.setAttribute('tabindex', '0');

    panelContainer.querySelectorAll('[role="tabpanel"]').forEach((p) => { p.hidden = true; });
    panelContainer.querySelector(`#panel-${tab.id}`).hidden = false;
  });

  block.textContent = '';
  block.append(tabList, panelContainer);
}
```

#### Accordion / Disclosure Pattern

```javascript
export default function decorate(block) {
  const rows = [...block.querySelectorAll(':scope > div')];

  const accordion = document.createElement('div');
  accordion.classList.add('accordion');

  rows.forEach((row, i) => {
    const [labelCol, contentCol] = [...row.querySelectorAll(':scope > div')];

    const details = document.createElement('details');
    if (i === 0) details.open = true; // First item open by default

    const summary = document.createElement('summary');
    summary.textContent = labelCol.textContent;

    const content = document.createElement('div');
    content.classList.add('accordion-content');
    content.append(...contentCol.childNodes);

    details.append(summary, content);
    accordion.append(details);
  });

  block.textContent = '';
  block.append(accordion);
}
```

#### Carousel / Slider Without Libraries

```javascript
export default function decorate(block) {
  const slides = [...block.querySelectorAll(':scope > div')];
  let currentIndex = 0;

  const track = document.createElement('div');
  track.classList.add('carousel-track');

  slides.forEach((slide, i) => {
    slide.classList.add('carousel-slide');
    slide.setAttribute('aria-hidden', i !== 0 ? 'true' : 'false');
    track.append(slide);
  });

  const nav = document.createElement('div');
  nav.classList.add('carousel-nav');
  nav.innerHTML = `
    <button class="prev" aria-label="Previous slide">&lsaquo;</button>
    <button class="next" aria-label="Next slide">&rsaquo;</button>
  `;

  function goToSlide(index) {
    currentIndex = (index + slides.length) % slides.length;
    track.style.transform = `translateX(-${currentIndex * 100}%)`;
    slides.forEach((slide, i) => {
      slide.setAttribute('aria-hidden', i !== currentIndex ? 'true' : 'false');
    });
  }

  nav.addEventListener('click', (e) => {
    const btn = e.target.closest('button');
    if (!btn) return;
    if (btn.classList.contains('prev')) goToSlide(currentIndex - 1);
    if (btn.classList.contains('next')) goToSlide(currentIndex + 1);
  });

  // Swipe support
  let startX = 0;
  track.addEventListener('touchstart', (e) => { startX = e.touches[0].clientX; }, { passive: true });
  track.addEventListener('touchend', (e) => {
    const diff = startX - e.changedTouches[0].clientX;
    if (Math.abs(diff) > 50) {
      goToSlide(currentIndex + (diff > 0 ? 1 : -1));
    }
  }, { passive: true });

  block.textContent = '';
  block.append(track, nav);
}
```

#### Modal / Dialog Using the `<dialog>` Element

```javascript
export default function decorate(block) {
  const triggerText = block.querySelector('h3')?.textContent ?? 'Open';
  const content = block.querySelector(':scope > div:last-child');

  const trigger = document.createElement('button');
  trigger.textContent = triggerText;
  trigger.classList.add('dialog-trigger');

  const dialog = document.createElement('dialog');
  dialog.classList.add('block-dialog');
  dialog.innerHTML = `
    <form method="dialog">
      <button class="dialog-close" aria-label="Close">&times;</button>
    </form>
    <div class="dialog-body"></div>
  `;
  dialog.querySelector('.dialog-body').append(...content.childNodes);

  trigger.addEventListener('click', () => dialog.showModal());

  // Close on backdrop click
  dialog.addEventListener('click', (e) => {
    if (e.target === dialog) dialog.close();
  });

  block.textContent = '';
  block.append(trigger, dialog);
}
```

#### Responsive Navigation Menu

```javascript
export default function decorate(block) {
  const links = [...block.querySelectorAll('a')];

  const nav = document.createElement('nav');
  nav.setAttribute('aria-label', 'Main navigation');

  const toggle = document.createElement('button');
  toggle.classList.add('nav-toggle');
  toggle.setAttribute('aria-expanded', 'false');
  toggle.setAttribute('aria-controls', 'nav-menu');
  toggle.setAttribute('aria-label', 'Toggle navigation');
  toggle.innerHTML = '<span class="hamburger"></span>';

  const menu = document.createElement('ul');
  menu.id = 'nav-menu';
  menu.classList.add('nav-menu');
  menu.setAttribute('role', 'list');

  links.forEach((link) => {
    const li = document.createElement('li');
    li.append(link);
    menu.append(li);
  });

  toggle.addEventListener('click', () => {
    const expanded = toggle.getAttribute('aria-expanded') === 'true';
    toggle.setAttribute('aria-expanded', String(!expanded));
    menu.classList.toggle('open', !expanded);
  });

  // Close menu when clicking outside
  document.addEventListener('click', (e) => {
    if (!nav.contains(e.target) && menu.classList.contains('open')) {
      toggle.setAttribute('aria-expanded', 'false');
      menu.classList.remove('open');
    }
  });

  nav.append(toggle, menu);
  block.textContent = '';
  block.append(nav);
}
```

#### Debounce and Throttle (No Libraries)

```javascript
// Debounce — wait until calls stop for `delay` ms
function debounce(fn, delay = 200) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

// Throttle — run at most once per `limit` ms
function throttle(fn, limit = 100) {
  let lastCall = 0;
  let timer;
  return (...args) => {
    const now = Date.now();
    const remaining = limit - (now - lastCall);
    clearTimeout(timer);
    if (remaining <= 0) {
      lastCall = now;
      fn(...args);
    } else {
      timer = setTimeout(() => {
        lastCall = Date.now();
        fn(...args);
      }, remaining);
    }
  };
}

// Usage in a block
export default function decorate(block) {
  const input = block.querySelector('input[type="search"]');

  // Debounced search — waits for user to stop typing
  input.addEventListener('input', debounce((e) => {
    performSearch(e.target.value, block);
  }, 300));

  // Throttled scroll handler — runs at most every 100ms
  window.addEventListener('scroll', throttle(() => {
    updateStickyHeader(block);
  }, 100), { passive: true });
}
```

---

### 7. Anti-Patterns for EDS

Every item below will degrade performance, break EDS conventions, or introduce security vulnerabilities.

#### Never Use jQuery

```javascript
// WRONG — jQuery is not available and not needed
$('.block').find('.card').addClass('active');

// CORRECT — vanilla JS equivalent
block.querySelectorAll('.card').forEach((card) => card.classList.add('active'));
```

#### No Framework Patterns

```javascript
// WRONG — virtual DOM, state management, component lifecycle
class CardComponent {
  constructor() { this.state = { active: false }; }
  setState(newState) { this.state = { ...this.state, ...newState }; this.render(); }
  render() { /* re-render entire component */ }
}

// CORRECT — direct DOM manipulation
function toggleCard(card) {
  card.classList.toggle('active');
}
```

#### Never Use innerHTML with User Content

```javascript
// WRONG — XSS vulnerability
const searchTerm = new URLSearchParams(window.location.search).get('q');
block.innerHTML = `<p>Results for: ${searchTerm}</p>`;

// CORRECT — textContent is safe
const p = document.createElement('p');
p.textContent = `Results for: ${searchTerm}`;
block.append(p);
```

#### No Synchronous XHR

```javascript
// WRONG — blocks the main thread
const xhr = new XMLHttpRequest();
xhr.open('GET', '/api/data.json', false); // synchronous
xhr.send();

// CORRECT — async fetch
const resp = await fetch('/api/data.json');
const data = await resp.json();
```

#### Never Use document.write, with, or eval

```javascript
// ALL WRONG — never use these
document.write('<script src="tracker.js"></script>');
with (config) { doSomething(value); }
eval('const x = ' + jsonString);

// CORRECT alternatives
const script = document.createElement('script');
script.src = 'tracker.js';
document.head.append(script);

const { value } = config;
doSomething(value);

const x = JSON.parse(jsonString);
```

#### No Unnecessary Polyfills

EDS targets modern evergreen browsers. Do not polyfill features that already have full support:

```javascript
// WRONG — these polyfills are unnecessary
import 'core-js/features/promise';
import 'whatwg-fetch';
import 'intersection-observer';
import '@webcomponents/custom-elements';

// CORRECT — use native APIs directly
const observer = new IntersectionObserver(callback);
const resp = await fetch(url);
const result = await Promise.allSettled(promises);
```

#### Always Use DocumentFragment for Loops

```javascript
// WRONG — causes reflow on every append
export default function decorate(block) {
  const list = document.createElement('ul');
  block.append(list);
  items.forEach((item) => {
    const li = document.createElement('li');
    li.textContent = item;
    list.append(li); // DOM write inside loop
  });
}

// CORRECT — batch with fragment, or build before attaching
export default function decorate(block) {
  const list = document.createElement('ul');
  const fragment = document.createDocumentFragment();

  items.forEach((item) => {
    const li = document.createElement('li');
    li.textContent = item;
    fragment.append(li);
  });

  list.append(fragment);
  block.append(list); // Single DOM insertion
}

// ALSO CORRECT — build the list before appending to the DOM
export default function decorate(block) {
  const list = document.createElement('ul');

  items.forEach((item) => {
    const li = document.createElement('li');
    li.textContent = item;
    list.append(li); // Safe — list is not yet in the DOM
  });

  block.append(list); // Single DOM insertion
}
```
