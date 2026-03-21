---
title: Edge Delivery Services Block Development
impact: CRITICAL
impactDescription: EDS is Adobe's recommended approach for new high-performance sites — correct patterns are essential
tags: edge-delivery, eds, franklin, helix, blocks, vanilla-js, universal-editor
---

## Edge Delivery Services Block Development

Edge Delivery Services (EDS) delivers high-performance sites using plain HTML, CSS, and vanilla JavaScript. Content is authored via Universal Editor or Google Docs and delivered through a global CDN. No frameworks allowed.

### Project Structure (XWalk Boilerplate)

```
project-root/
├── blocks/                           # Custom block components
│   ├── columns/
│   │   ├── columns.js
│   │   └── columns.css
│   ├── hero/
│   │   ├── hero.js
│   │   └── hero.css
│   └── teaser/
│       ├── _teaser.json              # Block definition + model + filters
│       ├── teaser.js
│       └── teaser.css
├── models/                           # Shared content models
│   └── _section.json
├── scripts/
│   ├── aem.js                        # EDS runtime utilities
│   ├── scripts.js                    # Site-wide JavaScript
│   └── delayed.js                    # Deferred scripts (analytics, etc.)
├── styles/
│   ├── styles.css                    # Global styles
│   └── fonts.css                     # Font definitions
├── head.html                         # Global <head> content
├── fstab.yaml                        # Connects to AEM Author
├── component-definition.json         # Compiled block definitions
├── component-models.json             # Compiled content models
└── component-filters.json            # Compiled block filters
```

### Block JavaScript Pattern

Each block exports a single `decorate` function:

```javascript
export default function decorate(block) {
  // block = the DOM element for this block instance
  // Add semantic classes, restructure DOM, add event listeners

  block.querySelector('picture')?.classList.add('image-wrapper');
  block.querySelector('h1,h2,h3,h4,h5,h6')?.classList.add('title');

  block.querySelectorAll('p').forEach((p) => {
    if (p.innerHTML?.trim().startsWith('Terms:')) {
      p.classList.add('terms');
    }
  });

  block.querySelector('.button')?.addEventListener('mouseover', () => {
    block.querySelector('.image')?.classList.add('zoom');
  });
}
```

### Block CSS Pattern

```css
/* Scoped to the block */
.block.teaser {
  position: relative;
  height: 500px;

  .image-wrapper {
    position: absolute;
    inset: 0;
  }

  .content {
    position: relative;
    padding: 2rem;
  }

  /* Variant via block options */
  &.side-by-side {
    display: flex;
    height: auto;

    .image-wrapper { flex: 1; }
    .content { flex: 1; }
  }
}
```

### Block Definition JSON (`_teaser.json`)

```json
{
  "definitions": [{
    "title": "Teaser",
    "id": "teaser",
    "plugins": {
      "xwalk": {
        "page": {
          "resourceType": "core/franklin/components/block/v1/block",
          "template": {
            "name": "Teaser",
            "model": "teaser",
            "classes": ""
          }
        }
      }
    }
  }],
  "models": [{
    "id": "teaser",
    "fields": [
      { "component": "reference", "name": "image", "label": "Image", "valueType": "string" },
      { "component": "text", "name": "imageAlt", "label": "Alt text", "valueType": "string" },
      { "component": "richtext", "name": "textContent_text", "label": "Text", "valueType": "string" },
      { "component": "aem-content", "name": "textContent_cta", "label": "CTA", "valueType": "string" },
      { "component": "text", "name": "textContent_ctaText", "label": "CTA label", "valueType": "string" }
    ]
  }],
  "filters": []
}
```

### Field Grouping & Collapse

- **Element Grouping**: Fields with matching prefixes (`textContent_text`, `textContent_cta`) render in the same container `<div>`
- **Field Collapse**: `image` + `imageAlt` collapse into a single `<img>` element

### Block Options (Variants)

Define using `classes` as the field name in the model:

```json
{
  "component": "select",
  "name": "classes",
  "value": "",
  "label": "Layout",
  "valueType": "string",
  "options": [
    { "name": "Default", "value": "" },
    { "name": "Side-by-side", "value": "side-by-side" }
  ]
}
```

Selected values become CSS classes: `<div class="block teaser side-by-side">`

### Detecting Options in JS

```javascript
function getOptions(block) {
  return [...block.classList].filter((c) => !['block', 'teaser'].includes(c));
}

if (getOptions(block).includes('side-by-side')) {
  // Apply side-by-side specific DOM changes
}
```

### Development Workflow

1. Create block folder with JS + CSS + optional JSON
2. `npm run lint` for validation
3. Husky pre-commit hook compiles JSON fragments into `component-*.json`
4. Push to branch, preview with `?ref=branch-name`

### Key Principles

- **No frameworks** — vanilla JavaScript and CSS only
- **Performance first** — minimal DOM manipulation, lazy loading built-in
- **Semantic HTML** — block HTML generated from content models
- **Auto-loading** — CSS/JS loaded only when block is present on page
- **Git-based deployment** — push to main = deploy to production

### Pitfalls

- Excessive DOM manipulation breaks Universal Editor authoring
- Do NOT use React, Next.js, SASS, or any framework
- Block names must be lowercase and hyphenated
- JSON syntax errors caught by `npm run lint:js` — always lint before commit
- Don't add script/link tags manually — EDS auto-loads block assets
