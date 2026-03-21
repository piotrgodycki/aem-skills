---
title: DOM Manipulation Patterns for AEM
impact: MEDIUM
impactDescription: Inefficient DOM operations cause layout thrashing, memory leaks, and poor Core Web Vitals in AEM pages with many components
tags: vanilla-js, dom, performance, mutation-observer, document-fragment, event-delegation, raf
---

## DOM Manipulation Patterns for AEM

Efficient DOM patterns for AEM vanilla JS components — batch operations, avoid layout thrashing, manage memory, and integrate with AEM's component lifecycle.

---

### 1. DocumentFragment for Batch Rendering

```typescript
// Build DOM off-screen, then insert once
function renderSearchResults(container: HTMLElement, results: SearchResult[]) {
  const fragment = document.createDocumentFragment();

  results.forEach(result => {
    const li = document.createElement('li');
    li.className = 'cmp-search__result';
    li.innerHTML = `
      <a class="cmp-search__result-link" href="${escapeHtml(result.path)}">
        <h3 class="cmp-search__result-title">${escapeHtml(result.title)}</h3>
        <p class="cmp-search__result-desc">${escapeHtml(result.description)}</p>
      </a>
    `;
    fragment.appendChild(li);
  });

  // Single DOM operation — one reflow
  container.replaceChildren(fragment);
}

// Incorrect — causes N reflows:
function renderResultsBad(container: HTMLElement, results: SearchResult[]) {
  container.innerHTML = ''; // reflow 1
  results.forEach(result => {
    container.innerHTML += `<li>...</li>`; // reflow per iteration!
  });
}
```

---

### 2. Template Element Cloning

For repeated structures, clone templates instead of parsing HTML strings:

```typescript
// Create template once
const cardTemplate = document.createElement('template');
cardTemplate.innerHTML = `
  <div class="cmp-card">
    <img class="cmp-card__image" src="" alt="" loading="lazy" />
    <div class="cmp-card__body">
      <h3 class="cmp-card__title"></h3>
      <p class="cmp-card__description"></p>
      <a class="cmp-card__link" href="">Read More</a>
    </div>
  </div>
`;

function createCard(data: CardData): HTMLElement {
  const card = cardTemplate.content.firstElementChild!.cloneNode(true) as HTMLElement;

  card.querySelector<HTMLImageElement>('.cmp-card__image')!.src = data.image;
  card.querySelector<HTMLImageElement>('.cmp-card__image')!.alt = data.imageAlt;
  card.querySelector('.cmp-card__title')!.textContent = data.title;
  card.querySelector('.cmp-card__description')!.textContent = data.description;
  card.querySelector<HTMLAnchorElement>('.cmp-card__link')!.href = data.link;

  return card;
}
```

---

### 3. Event Delegation

Single listener on a parent instead of N listeners on children:

```typescript
export function initTabs(root: HTMLElement) {
  const tablist = root.querySelector<HTMLElement>('.cmp-tabs__tablist');
  if (!tablist) return;

  const controller = new AbortController();

  // One listener handles all tabs
  tablist.addEventListener('click', (e: Event) => {
    const tab = (e.target as HTMLElement).closest<HTMLElement>('.cmp-tabs__tab');
    if (!tab) return;

    const index = Array.from(tablist.children).indexOf(tab);
    activateTab(root, index);
  }, { signal: controller.signal });

  // Keyboard navigation
  tablist.addEventListener('keydown', (e: KeyboardEvent) => {
    const tab = (e.target as HTMLElement).closest<HTMLElement>('.cmp-tabs__tab');
    if (!tab) return;

    const tabs = Array.from(tablist.querySelectorAll<HTMLElement>('.cmp-tabs__tab'));
    const index = tabs.indexOf(tab);

    let newIndex = index;
    if (e.key === 'ArrowRight') newIndex = (index + 1) % tabs.length;
    else if (e.key === 'ArrowLeft') newIndex = (index - 1 + tabs.length) % tabs.length;
    else return;

    e.preventDefault();
    activateTab(root, newIndex);
    tabs[newIndex].focus();
  }, { signal: controller.signal });

  return () => controller.abort();
}

function activateTab(root: HTMLElement, index: number) {
  const tabs = root.querySelectorAll<HTMLElement>('.cmp-tabs__tab');
  const panels = root.querySelectorAll<HTMLElement>('.cmp-tabs__tabpanel');

  tabs.forEach((tab, i) => {
    tab.setAttribute('aria-selected', String(i === index));
    tab.tabIndex = i === index ? 0 : -1;
  });

  panels.forEach((panel, i) => {
    panel.hidden = i !== index;
  });
}
```

---

### 4. Avoiding Layout Thrashing

```typescript
// Incorrect — read/write interleaving causes forced reflows:
function resizeBad(elements: HTMLElement[]) {
  elements.forEach(el => {
    const width = el.offsetWidth;   // READ → forces layout
    el.style.height = `${width}px`; // WRITE → invalidates layout
    // Next iteration: READ forces layout again!
  });
}

// Correct — batch reads, then batch writes:
function resizeGood(elements: HTMLElement[]) {
  // Phase 1: READ all
  const widths = elements.map(el => el.offsetWidth);

  // Phase 2: WRITE all
  elements.forEach((el, i) => {
    el.style.height = `${widths[i]}px`;
  });
}

// Best — use requestAnimationFrame for writes:
function resizeBest(elements: HTMLElement[]) {
  const widths = elements.map(el => el.offsetWidth);

  requestAnimationFrame(() => {
    elements.forEach((el, i) => {
      el.style.height = `${widths[i]}px`;
    });
  });
}
```

---

### 5. MutationObserver for Dynamic Content

Watch for AEM author-mode changes or lazy-loaded content:

```typescript
export function observeComponentChanges(
  root: HTMLElement,
  selector: string,
  callback: (added: HTMLElement[], removed: HTMLElement[]) => void
) {
  const observer = new MutationObserver(mutations => {
    const added: HTMLElement[] = [];
    const removed: HTMLElement[] = [];

    mutations.forEach(mutation => {
      mutation.addedNodes.forEach(node => {
        if (node instanceof HTMLElement) {
          if (node.matches(selector)) added.push(node);
          added.push(...Array.from(node.querySelectorAll<HTMLElement>(selector)));
        }
      });
      mutation.removedNodes.forEach(node => {
        if (node instanceof HTMLElement) {
          if (node.matches(selector)) removed.push(node);
          removed.push(...Array.from(node.querySelectorAll<HTMLElement>(selector)));
        }
      });
    });

    if (added.length || removed.length) {
      callback(added, removed);
    }
  });

  observer.observe(root, { childList: true, subtree: true });
  return () => observer.disconnect();
}

// Usage — auto-init new components added by author:
observeComponentChanges(document.body, '.cmp-accordion', (added, removed) => {
  added.forEach(el => initAccordion(el));
  removed.forEach(el => {
    const cleanup = cleanups.get(el);
    if (cleanup) {
      cleanup();
      cleanups.delete(el);
    }
  });
});
```

---

### 6. Memory Leak Prevention

```typescript
// Common leak: storing references to removed DOM nodes
class ComponentManager {
  private components = new WeakMap<HTMLElement, () => void>();

  register(el: HTMLElement, cleanup: () => void) {
    // WeakMap — allows GC when element is removed from DOM
    this.components.set(el, cleanup);
  }

  destroy(el: HTMLElement) {
    const cleanup = this.components.get(el);
    if (cleanup) {
      cleanup();
      this.components.delete(el);
    }
  }
}

// Common leak: closures holding DOM references
function initCarousel(root: HTMLElement) {
  const controller = new AbortController();
  let timer: ReturnType<typeof setInterval>;

  timer = setInterval(() => {
    // Verify element is still in DOM
    if (!root.isConnected) {
      clearInterval(timer);
      controller.abort();
      return;
    }
    advanceSlide(root);
  }, 5000);

  return () => {
    clearInterval(timer);
    controller.abort();
  };
}
```

---

### 7. XSS-Safe HTML Generation

```typescript
// utils/dom.ts
export function escapeHtml(str: string): string {
  const div = document.createElement('div');
  div.textContent = str;
  return div.innerHTML;
}

// For complex templates, use tagged template literals:
export function html(strings: TemplateStringsArray, ...values: unknown[]): string {
  return strings.reduce((result, str, i) => {
    const value = i < values.length ? escapeHtml(String(values[i])) : '';
    return result + str + value;
  }, '');
}

// Usage:
const card = html`
  <div class="cmp-card">
    <h3>${userTitle}</h3>
    <p>${userDescription}</p>
  </div>
`;
```

---

### Anti-Patterns

- **`innerHTML +=` in loops** — causes N reflows and destroys event listeners; use DocumentFragment
- **Reading layout properties in write loops** — interleaved reads/writes cause layout thrashing
- **Event listeners without cleanup** — use AbortController; return cleanup function from init
- **Strong references to DOM nodes** — use WeakMap/WeakRef for caches; check `isConnected` in timers
- **`document.getElementById` in tight loops** — cache references outside the loop
- **Unsanitized user content in innerHTML** — always escape or use textContent
- **querySelectorAll without scope** — always query from the component root, not `document`
