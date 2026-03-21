---
title: Web Components for AEM
impact: HIGH
impactDescription: Custom Elements provide framework-free encapsulated components that integrate with AEM's HTL rendering and Coral UI authoring
tags: web-components, custom-elements, shadow-dom, slots, coral-ui, aem, htm, dialog
---

## Web Components for AEM

Web Components (Custom Elements + Shadow DOM) provide encapsulated, framework-free components that work with AEM's HTL server rendering, Coral UI authoring dialogs, and existing ClientLib pipeline.

---

### 1. Basic Custom Element for AEM

```typescript
// ui.frontend/src/components/site-accordion.ts
class SiteAccordion extends HTMLElement {
  private items: Array<{ header: HTMLElement; panel: HTMLElement }> = [];

  connectedCallback() {
    this.setupItems();
    this.addEventListener('click', this.handleClick.bind(this));
    this.addEventListener('keydown', this.handleKeydown.bind(this));
  }

  disconnectedCallback() {
    // Cleanup handled by removing element from DOM
  }

  private setupItems() {
    const headers = this.querySelectorAll<HTMLElement>('[data-accordion-header]');
    headers.forEach((header, index) => {
      const panel = header.nextElementSibling as HTMLElement;
      if (!panel) return;

      header.setAttribute('role', 'button');
      header.setAttribute('tabindex', '0');
      header.setAttribute('aria-expanded', index === 0 ? 'true' : 'false');
      panel.setAttribute('role', 'region');
      panel.hidden = index !== 0;

      this.items.push({ header, panel });
    });
  }

  private handleClick(e: Event) {
    const header = (e.target as HTMLElement).closest('[data-accordion-header]');
    if (header) this.toggle(header as HTMLElement);
  }

  private handleKeydown(e: KeyboardEvent) {
    const header = (e.target as HTMLElement).closest('[data-accordion-header]');
    if (!header) return;
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      this.toggle(header as HTMLElement);
    }
  }

  private toggle(header: HTMLElement) {
    const item = this.items.find(i => i.header === header);
    if (!item) return;

    const isExpanded = header.getAttribute('aria-expanded') === 'true';
    header.setAttribute('aria-expanded', String(!isExpanded));
    item.panel.hidden = isExpanded;
  }
}

customElements.define('site-accordion', SiteAccordion);
```

**HTL usage:**

```html
<site-accordion class="cmp-accordion">
  <sly data-sly-list.item="${model.items}">
    <h3 data-accordion-header class="cmp-accordion__header">${item.title}</h3>
    <div class="cmp-accordion__panel">
      <sly data-sly-resource="${item.resource}" />
    </div>
  </sly>
</site-accordion>
```

---

### 2. Shadow DOM with AEM Styles

Use Shadow DOM for style encapsulation, with CSS custom properties for theming:

```typescript
// ui.frontend/src/components/site-tooltip.ts
const template = document.createElement('template');
template.innerHTML = `
  <style>
    :host {
      position: relative;
      display: inline-block;
    }
    .tooltip {
      position: absolute;
      bottom: 100%;
      left: 50%;
      transform: translateX(-50%);
      padding: var(--tooltip-padding, 8px 12px);
      background: var(--tooltip-bg, #333);
      color: var(--tooltip-color, #fff);
      border-radius: var(--tooltip-radius, 4px);
      font-size: var(--tooltip-font-size, 14px);
      white-space: nowrap;
      opacity: 0;
      pointer-events: none;
      transition: opacity 0.2s;
    }
    :host(:hover) .tooltip,
    :host(:focus-within) .tooltip {
      opacity: 1;
    }
  </style>
  <slot></slot>
  <div class="tooltip" role="tooltip"><slot name="content"></slot></div>
`;

class SiteTooltip extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this.shadowRoot!.appendChild(template.content.cloneNode(true));
  }

  static get observedAttributes() {
    return ['text'];
  }

  attributeChangedCallback(name: string, _old: string, value: string) {
    if (name === 'text') {
      const tooltip = this.shadowRoot!.querySelector('.tooltip')!;
      tooltip.textContent = value;
    }
  }
}

customElements.define('site-tooltip', SiteTooltip);
```

**HTL:**

```html
<site-tooltip text="Click to learn more">
  <a href="${model.link}">Help</a>
</site-tooltip>
```

**Theme via CSS custom properties (no Shadow DOM piercing needed):**

```css
/* In site theme CSS */
site-tooltip {
  --tooltip-bg: var(--color-primary);
  --tooltip-color: var(--color-on-primary);
  --tooltip-radius: var(--border-radius-sm);
}
```

---

### 3. Slots for HTL Content Projection

```typescript
// ui.frontend/src/components/site-card.ts
const cardTemplate = document.createElement('template');
cardTemplate.innerHTML = `
  <style>
    :host { display: block; }
    .card {
      border: 1px solid var(--card-border, #e0e0e0);
      border-radius: var(--card-radius, 8px);
      overflow: hidden;
    }
    .card__media { width: 100%; }
    .card__body { padding: var(--card-padding, 16px); }
    .card__footer {
      padding: var(--card-padding, 16px);
      border-top: 1px solid var(--card-border, #e0e0e0);
    }
    ::slotted(img) { width: 100%; display: block; }
  </style>
  <div class="card">
    <div class="card__media"><slot name="media"></slot></div>
    <div class="card__body"><slot></slot></div>
    <div class="card__footer"><slot name="footer"></slot></div>
  </div>
`;

class SiteCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this.shadowRoot!.appendChild(cardTemplate.content.cloneNode(true));
  }
}

customElements.define('site-card', SiteCard);
```

**HTL:**

```html
<site-card>
  <img slot="media" src="${model.image}" alt="${model.imageAlt}" loading="lazy" />
  <h3>${model.title}</h3>
  <p>${model.description}</p>
  <a slot="footer" href="${model.link}">Read More</a>
</site-card>
```

---

### 4. Coral UI Interop (Authoring Dialogs)

Web Components used inside AEM authoring dialogs alongside Coral UI:

```typescript
// ui.frontend/src/authoring/color-palette-field.ts
class ColorPaletteField extends HTMLElement {
  private input!: HTMLInputElement;

  connectedCallback() {
    // Create hidden input for form submission
    this.input = document.createElement('input');
    this.input.type = 'hidden';
    this.input.name = this.getAttribute('name') || '';
    this.input.value = this.getAttribute('value') || '';
    this.appendChild(this.input);

    // Render color swatches
    const colors = (this.getAttribute('colors') || '').split(',');
    const palette = document.createElement('div');
    palette.className = 'color-palette';

    colors.forEach(color => {
      const swatch = document.createElement('button');
      swatch.type = 'button';
      swatch.className = 'color-palette__swatch';
      swatch.style.backgroundColor = color.trim();
      swatch.setAttribute('data-color', color.trim());
      if (color.trim() === this.input.value) {
        swatch.classList.add('color-palette__swatch--selected');
      }
      palette.appendChild(swatch);
    });

    palette.addEventListener('click', (e) => {
      const swatch = (e.target as HTMLElement).closest('[data-color]') as HTMLElement;
      if (!swatch) return;
      this.input.value = swatch.dataset.color!;
      palette.querySelectorAll('.color-palette__swatch--selected')
        .forEach(s => s.classList.remove('color-palette__swatch--selected'));
      swatch.classList.add('color-palette__swatch--selected');

      // Trigger Granite UI validation
      this.input.dispatchEvent(new Event('change', { bubbles: true }));
    });

    this.appendChild(palette);
  }
}

customElements.define('color-palette-field', ColorPaletteField);
```

---

### 5. Form-Associated Custom Elements

Custom elements that participate in HTML forms (including AEM dialogs):

```typescript
class SiteRating extends HTMLElement {
  static formAssociated = true;
  private internals: ElementInternals;

  constructor() {
    super();
    this.internals = this.attachInternals();
  }

  connectedCallback() {
    const max = Number(this.getAttribute('max')) || 5;
    const value = Number(this.getAttribute('value')) || 0;

    for (let i = 1; i <= max; i++) {
      const star = document.createElement('button');
      star.type = 'button';
      star.textContent = i <= value ? '★' : '☆';
      star.className = 'cmp-rating__star';
      star.addEventListener('click', () => this.setValue(i));
      this.appendChild(star);
    }

    this.internals.setFormValue(String(value));
  }

  private setValue(val: number) {
    this.internals.setFormValue(String(val));
    const stars = this.querySelectorAll('.cmp-rating__star');
    stars.forEach((star, i) => {
      star.textContent = i < val ? '★' : '☆';
    });
  }
}

customElements.define('site-rating', SiteRating);
```

---

### Anti-Patterns

- **Shadow DOM for everything** — only use when style encapsulation is truly needed; it blocks AEM global styles
- **Not using `disconnectedCallback`** — event listeners on external elements leak memory
- **Extending built-in elements** — `class MyButton extends HTMLButtonElement` has poor Safari support; use autonomous custom elements
- **Heavy initialization in `constructor`** — don't access attributes or DOM in constructor; wait for `connectedCallback`
- **Not registering idempotently** — `customElements.define` throws if called twice; guard with `if (!customElements.get('my-el'))`
- **Ignoring Coral UI lifecycle in dialogs** — Coral initializes asynchronously; wait for `Coral.commons.ready()` before interacting
