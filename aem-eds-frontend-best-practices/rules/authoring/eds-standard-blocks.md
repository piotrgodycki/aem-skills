---
title: AEM Edge Delivery Services Standard Blocks & Components Reference
impact: HIGH
impactDescription: >
  Comprehensive reference for all standard EDS blocks, their HTML structure, JavaScript decorate
  patterns, CSS classes, Universal Editor component models, and block collection blocks. Essential
  for any developer building or customizing AEM EDS sites with the XWalk boilerplate.
tags:
  - eds
  - blocks
  - components
  - universal-editor
  - xwalk
  - boilerplate
  - block-collection
  - content-modeling
---

# AEM Edge Delivery Services — Standard Blocks & Components

This document covers every standard block and component available in the AEM EDS XWalk boilerplate
and the AEM Block Collection. Each entry includes purpose, HTML output, JS decorate pattern, CSS
classes, component model JSON, and common options/variants.

---

## 1. Default Content (No Block Needed)

Default content is any semantic HTML that does not belong to a named block. It renders inside a
`.default-content-wrapper` div within a `.section` container.

### What renders without a block

| Authored element       | Rendered HTML                                         |
|------------------------|-------------------------------------------------------|
| Heading (level 1-6)    | `<h1>` through `<h6>`                                 |
| Paragraph              | `<p>text</p>`                                         |
| Bold / Italic          | `<strong>` / `<em>`                                   |
| Link                   | `<a href="...">text</a>`                              |
| Image                  | `<picture><source .../><img .../></picture>`           |
| Unordered list         | `<ul><li>...</li></ul>`                                |
| Ordered list           | `<ol><li>...</li></ol>`                                |
| Button (strong link)   | `<p class="button-container"><a class="button">...</a></p>` |

Images are always rendered as full `<picture>` elements with multiple `<source>` children for
responsive formats and resolutions. AEM never outputs a bare `<img>` tag.

### Default content wrapper

```html
<div class="section">
  <div class="default-content-wrapper">
    <h2>Heading</h2>
    <p>Paragraph of text.</p>
    <p><picture><source .../><img src="..." alt="..."/></picture></p>
    <p class="button-container">
      <a href="/page" class="button primary">Call to action</a>
    </p>
  </div>
</div>
```

### Universal Editor component definitions for default content

These are registered in `component-definition.json` under the **Default Content** group:

```jsonc
// Text
{
  "title": "Text",
  "id": "text",
  "plugins": {
    "xwalk": {
      "page": {
        "resourceType": "core/franklin/components/text/v1/text",
        "template": {}
      }
    }
  }
}

// Title
{
  "title": "Title",
  "id": "title",
  "plugins": {
    "xwalk": {
      "page": {
        "resourceType": "core/franklin/components/title/v1/title",
        "template": { "model": "title" }
      }
    }
  }
}

// Image
{
  "title": "Image",
  "id": "image",
  "plugins": {
    "xwalk": {
      "page": {
        "resourceType": "core/franklin/components/image/v1/image",
        "template": {}
      }
    }
  }
}

// Button
{
  "title": "Button",
  "id": "button",
  "plugins": {
    "xwalk": {
      "page": {
        "resourceType": "core/franklin/components/button/v1/button",
        "template": { "model": "button" }
      }
    }
  }
}
```

### Default content models

```json
[
  {
    "id": "title",
    "fields": [
      {
        "component": "text",
        "name": "title",
        "label": "Title"
      },
      {
        "component": "select",
        "name": "titleType",
        "label": "Title Type",
        "options": [
          { "name": "h1", "value": "h1" },
          { "name": "h2", "value": "h2" },
          { "name": "h3", "value": "h3" },
          { "name": "h4", "value": "h4" },
          { "name": "h5", "value": "h5" },
          { "name": "h6", "value": "h6" }
        ]
      }
    ]
  },
  {
    "id": "image",
    "fields": [
      {
        "component": "reference",
        "name": "image",
        "label": "Image",
        "multi": false
      },
      {
        "component": "text",
        "name": "imageAlt",
        "label": "Alt Text"
      }
    ]
  },
  {
    "id": "button",
    "fields": [
      {
        "component": "aem-content",
        "name": "link",
        "label": "Link"
      },
      {
        "component": "text",
        "name": "linkText",
        "label": "Text"
      },
      {
        "component": "text",
        "name": "linkTitle",
        "label": "Title"
      },
      {
        "component": "select",
        "name": "linkType",
        "label": "Type",
        "options": [
          { "name": "default", "value": "" },
          { "name": "primary", "value": "primary" },
          { "name": "secondary", "value": "secondary" }
        ]
      }
    ]
  }
]
```

---

## 2. Sections and Section Metadata

### Section container

Every page is divided into sections. Horizontal rules (`---`) in authored content create section
breaks. The EDS framework wraps each section in:

```html
<main>
  <div class="section">
    <div class="default-content-wrapper"><!-- default content --></div>
    <div class="hero-wrapper"><!-- hero block --></div>
  </div>
  <div class="section highlight">
    <div class="default-content-wrapper"><!-- more content --></div>
  </div>
</main>
```

Each block inside a section also gets a wrapper div with the class `<blockname>-wrapper`.

### Section Metadata

Section Metadata is a special key-value table placed at the end of a section. It attaches
`data-*` attributes (and CSS classes via the `Style` property) to the containing `.section` div.

| Property   | Effect                                                        |
|------------|---------------------------------------------------------------|
| `Style`    | Converted to CSS classes on the `.section` element            |
| Any other  | Converted to `data-<property-name>` attribute on `.section`   |

For example, a section metadata table with `Style = highlight` produces:

```html
<div class="section highlight">
```

A property `Background = dark` produces:

```html
<div class="section" data-background="dark">
```

### Section definition (component-definition.json)

```json
{
  "title": "Section",
  "id": "section",
  "plugins": {
    "xwalk": {
      "page": {
        "resourceType": "core/franklin/components/section/v1/section",
        "template": {
          "model": "section"
        }
      }
    }
  }
}
```

### Section model (component-models.json)

```json
{
  "id": "section",
  "fields": [
    {
      "component": "text",
      "name": "name",
      "label": "Section Name",
      "description": "The label shown for this section in the Content Tree"
    },
    {
      "component": "multiselect",
      "name": "style",
      "label": "Style",
      "options": [
        { "name": "Highlight", "value": "highlight" }
      ]
    }
  ]
}
```

### Section filters (component-filters.json)

```json
[
  {
    "id": "main",
    "components": ["section"]
  },
  {
    "id": "section",
    "components": [
      "text",
      "image",
      "button",
      "title",
      "hero",
      "cards",
      "columns",
      "fragment"
    ]
  }
]
```

The `main` filter controls what can be added at the top level (only sections). The `section`
filter controls which blocks/components authors can insert inside a section.

---

## 3. Standard Boilerplate Blocks

### 3.1 Hero

**Purpose**: Full-width banner with background image, heading text, and optional CTA.

**HTML output**:

```html
<div class="hero-wrapper">
  <div class="hero block" data-block-name="hero">
    <div>
      <div>
        <picture><source .../><img src="..." alt="..."/></picture>
      </div>
    </div>
    <div>
      <div>
        <h1>Heading text</h1>
        <p>Supporting copy</p>
        <p class="button-container"><a href="..." class="button">CTA</a></p>
      </div>
    </div>
  </div>
</div>
```

**JS decorate pattern**: The hero block in the AEM Block Collection ships with an empty `hero.js`
(0 bytes). The hero is styled entirely with CSS. The image is positioned absolutely behind the
text content.

**CSS classes**:
- `.hero-wrapper` — outer wrapper (no max-width, no padding)
- `.hero` — block element with `position: relative; min-height: 300px`
- `.hero picture` — `position: absolute; inset: 0; z-index: -1`
- `.hero img` — `object-fit: cover; width: 100%; height: 100%`
- `.hero h1` — text color set to `var(--background-color)` for contrast

**Key CSS** (from block collection):

```css
.hero-container .hero-wrapper {
  max-width: unset;
  padding: 0;
}

.hero {
  position: relative;
  padding: 40px 24px;
  min-height: 300px;
}

.hero picture {
  position: absolute;
  z-index: -1;
  inset: 0;
}

.hero img {
  object-fit: cover;
  width: 100%;
  height: 100%;
}
```

**Component model** (`blocks/hero/_hero.json`):

```json
{
  "definitions": [
    {
      "title": "Hero",
      "id": "hero",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block",
            "template": {
              "name": "Hero",
              "model": "hero"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "hero",
      "fields": [
        {
          "component": "reference",
          "valueType": "string",
          "name": "image",
          "label": "Image",
          "multi": false
        },
        {
          "component": "text",
          "valueType": "string",
          "name": "imageAlt",
          "label": "Alt",
          "value": ""
        },
        {
          "component": "richtext",
          "name": "text",
          "value": "",
          "label": "Text",
          "valueType": "string"
        }
      ]
    }
  ],
  "filters": []
}
```

> The `image` + `imageAlt` fields use **field collapse**: because `imageAlt` matches
> the pattern `<fieldname>Alt`, EDS collapses them into a single `<img src="..." alt="...">`.

---

### 3.2 Columns

**Purpose**: Flexible multi-column layout. Each column can contain text, images, buttons.

**HTML output**:

```html
<div class="columns-wrapper">
  <div class="columns block columns-2-cols" data-block-name="columns">
    <div>
      <div class="columns-img-col">
        <picture>...</picture>
      </div>
      <div>
        <h3>Column 2 heading</h3>
        <p>Column 2 text</p>
      </div>
    </div>
  </div>
</div>
```

**JS decorate** (`columns.js`):

```javascript
export default function decorate(block) {
  const cols = [...block.firstElementChild.children];
  block.classList.add(`columns-${cols.length}-cols`);

  [...block.children].forEach((row) => {
    [...row.children].forEach((col) => {
      const pic = col.querySelector('picture');
      if (pic) {
        const picWrapper = pic.closest('div');
        if (picWrapper && picWrapper.children.length === 1) {
          picWrapper.classList.add('columns-img-col');
        }
      }
    });
  });
}
```

**CSS classes**:
- `.columns` — block element
- `.columns-2-cols`, `.columns-3-cols`, etc. — auto-added based on column count
- `.columns-img-col` — applied to columns containing only an image

**Component model** (`blocks/columns/_columns.json`):

```json
{
  "definitions": [
    {
      "title": "Columns",
      "id": "columns",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/columns/v1/columns",
            "template": {
              "columns": "2",
              "rows": "1"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "columns",
      "fields": [
        {
          "component": "text",
          "valueType": "number",
          "name": "columns",
          "value": "",
          "label": "Columns"
        },
        {
          "component": "text",
          "valueType": "number",
          "name": "rows",
          "value": "",
          "label": "Rows"
        }
      ]
    }
  ],
  "filters": [
    {
      "id": "columns",
      "components": ["column"]
    },
    {
      "id": "column",
      "components": ["text", "image", "button", "title"]
    }
  ]
}
```

> Columns use the dedicated resourceType `core/franklin/components/columns/v1/columns`
> (not the generic block resourceType). Filters define that each `column` can hold
> text, image, button, and title components.

---

### 3.3 Cards

**Purpose**: Grid of cards, each with an optional image and rich text body.

**HTML output** (after decoration):

```html
<div class="cards-wrapper">
  <div class="cards block" data-block-name="cards">
    <ul>
      <li>
        <div class="cards-card-image">
          <picture>...</picture>
        </div>
        <div class="cards-card-body">
          <h3>Card title</h3>
          <p>Card description</p>
        </div>
      </li>
      <!-- more <li> items -->
    </ul>
  </div>
</div>
```

**JS decorate** (`cards.js`):

```javascript
import { createOptimizedPicture } from '../../scripts/aem.js';

export default function decorate(block) {
  const ul = document.createElement('ul');
  [...block.children].forEach((row) => {
    const li = document.createElement('li');
    while (row.firstElementChild) li.append(row.firstElementChild);
    [...li.children].forEach((div) => {
      if (div.children.length === 1 && div.querySelector('picture')) {
        div.className = 'cards-card-image';
      } else {
        div.className = 'cards-card-body';
      }
    });
    ul.append(li);
  });
  ul.querySelectorAll('picture > img').forEach((img) => {
    img.closest('picture').replaceWith(
      createOptimizedPicture(img.src, img.alt, false, [{ width: '750' }]),
    );
  });
  block.textContent = '';
  block.append(ul);
}
```

**CSS classes**:
- `.cards` — block element
- `.cards ul` — the card grid (typically CSS Grid or Flexbox)
- `.cards-card-image` — image wrapper per card
- `.cards-card-body` — text content wrapper per card

**Component model** (`blocks/cards/_cards.json`):

```json
{
  "definitions": [
    {
      "title": "Cards",
      "id": "cards",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block",
            "template": {
              "name": "Cards",
              "filter": "cards"
            }
          }
        }
      }
    },
    {
      "title": "Card",
      "id": "card",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block/item",
            "template": {
              "name": "Card",
              "model": "card"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "card",
      "fields": [
        {
          "component": "reference",
          "valueType": "string",
          "name": "image",
          "label": "Image",
          "multi": false
        },
        {
          "component": "richtext",
          "name": "text",
          "value": "",
          "label": "Text",
          "valueType": "string"
        }
      ]
    }
  ],
  "filters": [
    {
      "id": "cards",
      "components": ["card"]
    }
  ]
}
```

> Cards is a **container block**. It defines two component definitions: `Cards` (the container,
> using `block/v1/block` with a filter) and `Card` (the item, using `block/v1/block/item` with
> a model). The filter ensures only `card` items can be added inside the cards container.

---

### 3.4 Tabs

**Purpose**: Tabbed content panels. Each tab has a label and associated content panel.

**HTML output** (after decoration):

```html
<div class="tabs-wrapper">
  <div class="tabs block" data-block-name="tabs">
    <div class="tabs-list" role="tablist">
      <button class="tabs-tab" id="tab-first" role="tab"
              aria-controls="tabpanel-first" aria-selected="true">First</button>
      <button class="tabs-tab" id="tab-second" role="tab"
              aria-controls="tabpanel-second" aria-selected="false">Second</button>
    </div>
    <div class="tabs-panel" id="tabpanel-first" role="tabpanel"
         aria-labelledby="tab-first" aria-hidden="false">
      <div>Panel 1 content</div>
    </div>
    <div class="tabs-panel" id="tabpanel-second" role="tabpanel"
         aria-labelledby="tab-second" aria-hidden="true">
      <div>Panel 2 content</div>
    </div>
  </div>
</div>
```

**JS decorate** (`tabs.js`):

```javascript
import { toClassName } from '../../scripts/aem.js';

export default async function decorate(block) {
  const tablist = document.createElement('div');
  tablist.className = 'tabs-list';
  tablist.setAttribute('role', 'tablist');

  const tabs = [...block.children].map((child) => child.firstElementChild);
  tabs.forEach((tab, i) => {
    const id = toClassName(tab.textContent);

    // decorate tabpanel
    const tabpanel = block.children[i];
    tabpanel.className = 'tabs-panel';
    tabpanel.id = `tabpanel-${id}`;
    tabpanel.setAttribute('aria-hidden', !!i);
    tabpanel.setAttribute('aria-labelledby', `tab-${id}`);
    tabpanel.setAttribute('role', 'tabpanel');

    // build tab button
    const button = document.createElement('button');
    button.className = 'tabs-tab';
    button.id = `tab-${id}`;
    button.innerHTML = tab.innerHTML;
    button.setAttribute('aria-controls', `tabpanel-${id}`);
    button.setAttribute('aria-selected', !i);
    button.setAttribute('role', 'tab');
    button.setAttribute('type', 'button');
    button.addEventListener('click', () => {
      block.querySelectorAll('[role=tabpanel]').forEach((panel) => {
        panel.setAttribute('aria-hidden', true);
      });
      tablist.querySelectorAll('button').forEach((btn) => {
        btn.setAttribute('aria-selected', false);
      });
      tabpanel.setAttribute('aria-hidden', false);
      button.setAttribute('aria-selected', true);
    });
    tablist.append(button);
    tab.remove();
  });

  block.prepend(tablist);
}
```

**CSS classes**:
- `.tabs` — block element
- `.tabs-list` — tab button container (`role="tablist"`)
- `.tabs-tab` — individual tab button (`role="tab"`)
- `.tabs-panel` — content panel (`role="tabpanel"`)

**Component model** (`blocks/tabs/_tabs.json`):

Tabs are typically modeled as section-based containers where each section represents a tab panel.
The tab label comes from the first element of each section.

```json
{
  "definitions": [
    {
      "title": "Tabs",
      "id": "tabs",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block",
            "template": {
              "name": "Tabs",
              "filter": "tabs"
            }
          }
        }
      }
    },
    {
      "title": "Tab Item",
      "id": "tab-item",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block/item",
            "template": {
              "name": "Tab Item",
              "model": "tab-item"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "tab-item",
      "fields": [
        {
          "component": "text",
          "name": "label",
          "label": "Tab Label",
          "valueType": "string"
        },
        {
          "component": "richtext",
          "name": "content",
          "label": "Content",
          "valueType": "string"
        }
      ]
    }
  ],
  "filters": [
    {
      "id": "tabs",
      "components": ["tab-item"]
    }
  ]
}
```

> **Alternative approach**: Tabs can also be modeled as sections with section metadata where
> each section becomes a tab panel using `resourceType: "core/franklin/components/section/v1/section"`.

---

### 3.5 Accordion

**Purpose**: Stack of collapsible label/content pairs using `<details>/<summary>` elements.

**HTML output** (after decoration):

```html
<div class="accordion-wrapper">
  <div class="accordion block" data-block-name="accordion">
    <details>
      <summary>Question or label</summary>
      <div class="accordion-item-body">
        <p>Answer or content</p>
      </div>
    </details>
    <!-- more <details> items -->
  </div>
</div>
```

**JS decorate** (`accordion.js`):

```javascript
export default function decorate(block) {
  [...block.children].forEach((row) => {
    const label = row.children[0];
    const body = row.children[1];

    const details = document.createElement('details');
    const summary = document.createElement('summary');
    summary.className = 'accordion-item-label';
    summary.append(...label.childNodes);
    details.append(summary);

    body.className = 'accordion-item-body';
    details.append(body);

    row.replaceWith(details);
  });
}
```

**CSS classes**:
- `.accordion` — block element
- `.accordion-item-label` — the `<summary>` element
- `.accordion-item-body` — the expandable content area

**Component model** (`blocks/accordion/_accordion.json`):

```json
{
  "definitions": [
    {
      "title": "Accordion",
      "id": "accordion",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block",
            "template": {
              "name": "Accordion",
              "filter": "accordion"
            }
          }
        }
      }
    },
    {
      "title": "Accordion Item",
      "id": "accordion-item",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block/item",
            "template": {
              "name": "Accordion Item",
              "model": "accordion-item"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "accordion-item",
      "fields": [
        {
          "component": "text",
          "name": "label",
          "label": "Label",
          "valueType": "string"
        },
        {
          "component": "richtext",
          "name": "body",
          "label": "Body",
          "valueType": "string"
        }
      ]
    }
  ],
  "filters": [
    {
      "id": "accordion",
      "components": ["accordion-item"]
    }
  ]
}
```

---

### 3.6 Carousel / Slider

**Purpose**: Rotating content display with navigation controls and slide indicators.

**HTML output** (after decoration):

```html
<div class="carousel-wrapper">
  <div class="carousel block" id="carousel-1" role="region"
       aria-roledescription="Carousel" data-block-name="carousel">
    <div class="carousel-slides-container">
      <ul class="carousel-slides">
        <li class="carousel-slide"><!-- slide content --></li>
        <li class="carousel-slide"><!-- slide content --></li>
      </ul>
      <div class="carousel-navigation-buttons">
        <button type="button" class="slide-prev" aria-label="Previous Slide"></button>
        <button type="button" class="slide-next" aria-label="Next Slide"></button>
      </div>
    </div>
    <nav aria-label="Carousel Slide Controls">
      <ol class="carousel-slide-indicators">
        <li class="carousel-slide-indicator" data-target-slide="0">
          <button type="button" aria-label="Show Slide 1 of 2"></button>
        </li>
        <li class="carousel-slide-indicator" data-target-slide="1">
          <button type="button" aria-label="Show Slide 2 of 2"></button>
        </li>
      </ol>
    </nav>
  </div>
</div>
```

**CSS classes**:
- `.carousel` — block element with `role="region"`
- `.carousel-slides-container` — wraps the slide track and nav buttons
- `.carousel-slides` — `<ul>` containing slides
- `.carousel-slide` — individual slide `<li>`
- `.carousel-navigation-buttons` — prev/next button container
- `.slide-prev`, `.slide-next` — navigation buttons
- `.carousel-slide-indicators` — dot/indicator nav
- `.carousel-slide-indicator` — individual indicator

**Component model** (`blocks/carousel/_carousel.json`):

```json
{
  "definitions": [
    {
      "title": "Carousel",
      "id": "carousel",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block",
            "template": {
              "name": "Carousel",
              "filter": "carousel"
            }
          }
        }
      }
    },
    {
      "title": "Carousel Slide",
      "id": "carousel-slide",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block/item",
            "template": {
              "name": "Carousel Slide",
              "model": "carousel-slide"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "carousel-slide",
      "fields": [
        {
          "component": "reference",
          "valueType": "string",
          "name": "image",
          "label": "Image",
          "multi": false
        },
        {
          "component": "text",
          "name": "imageAlt",
          "label": "Alt Text",
          "valueType": "string"
        },
        {
          "component": "richtext",
          "name": "text",
          "label": "Content",
          "valueType": "string"
        }
      ]
    }
  ],
  "filters": [
    {
      "id": "carousel",
      "components": ["carousel-slide"]
    }
  ]
}
```

---

### 3.7 Teaser

**Purpose**: Promotional block combining image, heading, body text, and CTA links. Commonly used
for feature highlights, article teasers, and marketing content.

**HTML output**:

```html
<div class="teaser-wrapper">
  <div class="teaser block" data-block-name="teaser">
    <div>
      <div>
        <picture><source .../><img src="..." alt="..."/></picture>
      </div>
    </div>
    <div>
      <div>
        <h2>Teaser title</h2>
        <p>Description text</p>
        <p class="button-container"><a href="..." class="button">Learn more</a></p>
      </div>
    </div>
  </div>
</div>
```

**JS decorate pattern**: Apply semantic CSS classes and optional interactivity.

```javascript
export default function decorate(block) {
  const [imageWrapper, contentWrapper] = block.children;
  if (imageWrapper) imageWrapper.classList.add('teaser-image');
  if (contentWrapper) contentWrapper.classList.add('teaser-content');

  // Optional: add hover effects, animations, etc.
}
```

**Common options** (via `classes` field):
- `side-by-side` — image and text side by side instead of stacked
- `left` / `right` — image position

**Component model** (`blocks/teaser/_teaser.json`):

```json
{
  "definitions": [
    {
      "title": "Teaser",
      "id": "teaser",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block",
            "template": {
              "name": "Teaser",
              "model": "teaser"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "teaser",
      "fields": [
        {
          "component": "reference",
          "valueType": "string",
          "name": "image",
          "label": "Image",
          "multi": false
        },
        {
          "component": "text",
          "valueType": "string",
          "name": "imageAlt",
          "label": "Image Alt Text",
          "required": true
        },
        {
          "component": "richtext",
          "name": "textContent_text",
          "label": "Text",
          "valueType": "string"
        },
        {
          "component": "aem-content",
          "name": "textContent_cta",
          "label": "CTA Link"
        },
        {
          "component": "text",
          "name": "textContent_ctaText",
          "label": "CTA Label",
          "valueType": "string"
        },
        {
          "component": "select",
          "name": "classes",
          "value": "",
          "label": "Teaser Style",
          "valueType": "string",
          "options": [
            { "name": "Default", "value": "" },
            { "name": "Side-by-side", "value": "side-by-side" }
          ]
        }
      ]
    }
  ],
  "filters": []
}
```

> The teaser uses **element grouping** via the `textContent_` prefix — all fields with that
> prefix are rendered into a single `<div>` in the HTML output. It also uses **field collapse**
> for `image`/`imageAlt` and `textContent_cta`/`textContent_ctaText`.

---

### 3.8 Fragment

**Purpose**: Embed another AEM page (or content fragment) inline. The referenced page's content
is fetched, decorated, and injected into the current page.

**HTML output**: The fragment block replaces itself with the content of the referenced page.

**JS decorate** (`fragment.js`):

```javascript
import { decorateMain } from '../../scripts/scripts.js';
import { loadSections } from '../../scripts/aem.js';

export async function loadFragment(path) {
  if (path && path.startsWith('/') && !path.startsWith('//')) {
    const resp = await fetch(`${path}.plain.html`);
    if (resp.ok) {
      const main = document.createElement('main');
      main.innerHTML = await resp.text();

      // reset base path for media to fragment base
      const resetAttributeBase = (tag, attr) => {
        main.querySelectorAll(`${tag}[${attr}^="./media_"]`).forEach((elem) => {
          elem[attr] = new URL(
            elem.getAttribute(attr), new URL(path, window.location)
          ).href;
        });
      };
      resetAttributeBase('img', 'src');
      resetAttributeBase('source', 'srcset');

      decorateMain(main);
      await loadSections(main);
      return main;
    }
  }
  return null;
}

export default async function decorate(block) {
  const link = block.querySelector('a');
  const path = link ? link.getAttribute('href') : block.textContent.trim();
  const fragment = await loadFragment(path);
  if (fragment) {
    const fragmentSection = fragment.querySelector(':scope .section');
    if (fragmentSection) {
      block.closest('.section').classList.add(...fragmentSection.classList);
      block.closest('.fragment').replaceWith(...fragment.childNodes);
    }
  }
}
```

**Component model** (`blocks/fragment/_fragment.json`):

```json
{
  "definitions": [
    {
      "title": "Fragment",
      "id": "fragment",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block",
            "template": {
              "name": "Fragment",
              "model": "fragment"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "fragment",
      "fields": [
        {
          "component": "aem-content",
          "name": "reference",
          "label": "Reference"
        }
      ]
    }
  ],
  "filters": []
}
```

---

### 3.9 Video / Embed

**Purpose**: Embed YouTube, Vimeo, Twitter/X content, or direct MP4 videos.

The block collection provides two variants:
- **Embed** — broader: YouTube, Vimeo, Twitter, and generic iframes
- **Video** — focused: YouTube, Vimeo, and native MP4 with autoplay/background support

**HTML output** (after decoration — YouTube example):

```html
<div class="embed block embed-youtube embed-is-loaded" data-block-name="embed">
  <div style="left: 0; width: 100%; height: 0; position: relative; padding-bottom: 56.25%;">
    <iframe src="https://www.youtube.com/embed/VIDEO_ID?rel=0&v=VIDEO_ID"
      style="border: 0; top: 0; left: 0; width: 100%; height: 100%; position: absolute;"
      allow="autoplay; fullscreen; picture-in-picture; encrypted-media"
      allowfullscreen scrolling="no" title="Content from Youtube" loading="lazy">
    </iframe>
  </div>
</div>
```

**CSS classes**:
- `.embed` / `.video` — block element
- `.embed-youtube`, `.embed-vimeo`, `.embed-twitter` — provider-specific classes
- `.embed-is-loaded` — added after iframe loads
- `.embed-placeholder` / `.video-placeholder` — click-to-play thumbnail wrapper
- `.embed-placeholder-play` / `.video-placeholder-play` — play button overlay

**Video block options**:
- `autoplay` class — auto-plays when scrolled into view (muted, looped, no controls)

**Key features**:
- Lazy loading via IntersectionObserver
- Optional click-to-play with poster image (when a `<picture>` is present)
- Respects `prefers-reduced-motion` media query
- 16:9 aspect ratio via `padding-bottom: 56.25%`

**Component model** (`blocks/embed/_embed.json`):

```json
{
  "definitions": [
    {
      "title": "Embed",
      "id": "embed",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block",
            "template": {
              "name": "Embed",
              "model": "embed"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "embed",
      "fields": [
        {
          "component": "text",
          "name": "url",
          "label": "URL",
          "valueType": "string",
          "description": "YouTube, Vimeo, or other embeddable URL"
        },
        {
          "component": "reference",
          "name": "poster",
          "label": "Poster Image",
          "multi": false,
          "description": "Optional poster image for click-to-play"
        }
      ]
    }
  ],
  "filters": []
}
```

---

### 3.10 Table

**Purpose**: Render data in a semantic HTML `<table>` with optional header row.

**HTML output** (after decoration):

```html
<div class="table-wrapper">
  <div class="table block" data-block-name="table">
    <table>
      <thead>
        <tr><th scope="col">Name</th><th scope="col">Value</th></tr>
      </thead>
      <tbody>
        <tr><td>Row 1</td><td>Data 1</td></tr>
        <tr><td>Row 2</td><td>Data 2</td></tr>
      </tbody>
    </table>
  </div>
</div>
```

**JS decorate** (`table.js`):

```javascript
function buildCell(rowIndex) {
  const cell = rowIndex
    ? document.createElement('td')
    : document.createElement('th');
  if (!rowIndex) cell.setAttribute('scope', 'col');
  return cell;
}

export default async function decorate(block) {
  const table = document.createElement('table');
  const thead = document.createElement('thead');
  const tbody = document.createElement('tbody');
  const header = !block.classList.contains('no-header');
  if (header) table.append(thead);
  table.append(tbody);

  [...block.children].forEach((child, i) => {
    const row = document.createElement('tr');
    if (header && i === 0) thead.append(row);
    else tbody.append(row);
    [...child.children].forEach((col) => {
      const cell = buildCell(header ? i : i + 1);
      cell.innerHTML = col.innerHTML;
      row.append(cell);
    });
  });
  block.innerHTML = '';
  block.append(table);
}
```

**CSS classes**:
- `.table` — block element
- Block option: `no-header` — skips the `<thead>` and renders all rows as `<td>`

---

### 3.11 Header

**Purpose**: Site header with branding, navigation, and tools. Loaded asynchronously from a
dedicated `nav` page.

**How it works**:
1. The header block reads `<meta name="nav">` to find the navigation page path (default: `/nav`)
2. Fetches `{navPath}.plain.html` via XHR
3. Decorates the fragment and injects it into the `<header>` element

**Expected nav page structure** (three sections):
- Section 1 (`.nav-brand`) — Logo and site name
- Section 2 (`.nav-sections`) — Main navigation links
- Section 3 (`.nav-tools`) — Search, login, profile

**JS decorate pattern**:

```javascript
import { loadFragment } from '../fragment/fragment.js';
import { getMetadata } from '../../scripts/aem.js';

export default async function decorate(block) {
  const navMeta = getMetadata('nav');
  const navPath = navMeta
    ? new URL(navMeta, window.location).pathname
    : '/nav';
  const fragment = await loadFragment(navPath);

  block.textContent = '';
  const nav = document.createElement('nav');
  nav.id = 'nav';

  while (fragment.firstElementChild) {
    nav.append(fragment.firstElementChild);
  }

  // Decorate nav sections: .nav-brand, .nav-sections, .nav-tools
  const classes = ['brand', 'sections', 'tools'];
  classes.forEach((c, i) => {
    const section = nav.children[i];
    if (section) section.classList.add(`nav-${c}`);
  });

  // Mobile hamburger menu
  const hamburger = document.createElement('div');
  hamburger.classList.add('nav-hamburger');
  hamburger.innerHTML = '<button type="button" aria-label="Menu"></button>';
  hamburger.addEventListener('click', () => {
    const expanded = nav.getAttribute('aria-expanded') === 'true';
    nav.setAttribute('aria-expanded', !expanded);
  });
  nav.prepend(hamburger);
  nav.setAttribute('aria-expanded', 'false');

  block.append(nav);
}
```

**CSS classes**:
- `.header` — block element (bound to `<header>` element)
- `#nav` — the `<nav>` element
- `.nav-brand` — logo/brand section
- `.nav-sections` — main navigation
- `.nav-tools` — utility links
- `.nav-hamburger` — mobile menu toggle

> **Important**: The `nav` page must be published separately. Publishing other pages does not
> update the header. This allows global header changes without full cache purges.

---

### 3.12 Footer

**Purpose**: Site footer loaded asynchronously from a dedicated `footer` page.

**How it works**: Same pattern as header — reads `<meta name="footer">` (default: `/footer`),
fetches `{footerPath}.plain.html`, and decorates.

**Recommended footer structure** (using Columns block):
- Left column: Promotional content (image + text)
- Middle column: Navigation links
- Right column: Social media links
- Bottom row: Copyright notice

**JS decorate pattern**:

```javascript
import { loadFragment } from '../fragment/fragment.js';
import { getMetadata } from '../../scripts/aem.js';

export default async function decorate(block) {
  const footerMeta = getMetadata('footer');
  const footerPath = footerMeta
    ? new URL(footerMeta, window.location).pathname
    : '/footer';
  const fragment = await loadFragment(footerPath);

  block.textContent = '';
  if (fragment) {
    const footer = document.createElement('div');
    while (fragment.firstElementChild) footer.append(fragment.firstElementChild);
    block.append(footer);
  }
}
```

**CSS classes**:
- `.footer` — block element (bound to `<footer>` element)

---

### 3.13 Breadcrumb

**Purpose**: Display hierarchical navigation path.

**HTML output** (after decoration):

```html
<div class="breadcrumb-wrapper">
  <div class="breadcrumb block" data-block-name="breadcrumb">
    <nav aria-label="Breadcrumb">
      <ol>
        <li><a href="/">Home</a></li>
        <li><a href="/products">Products</a></li>
        <li aria-current="page">Current Page</li>
      </ol>
    </nav>
  </div>
</div>
```

**JS decorate pattern**: Typically constructs the breadcrumb from page metadata, the URL path,
or a content-index JSON.

---

### 3.14 Search

**Purpose**: Site search with query input, results display, and URL parameter synchronization.

**Key features**:
- Minimum 3 characters to trigger search
- Syncs search term with `?q=` URL parameter
- Fetches from `/query-index.json` (configurable)
- Highlights matching terms with `<mark>` tags
- ARIA live region for screen reader announcements
- Prioritizes matches in headers over body content

**HTML output** (after decoration):

```html
<div class="search-wrapper">
  <div class="search block" data-block-name="search">
    <div class="search-input-wrapper">
      <input type="search" aria-label="Search" placeholder="Search...">
      <span class="icon icon-search"></span>
    </div>
    <ul class="search-results" aria-live="polite">
      <li>
        <a href="/page">
          <h3>Result Title</h3>
          <p>Description with <mark>matching</mark> term</p>
        </a>
      </li>
    </ul>
  </div>
</div>
```

---

## 4. Block Collection Blocks

The [AEM Block Collection](https://github.com/adobe/aem-block-collection) is the EDS equivalent
of AEM Core Components. To be included, a block must be used on more than half of all AEM
projects. The blocks provide reusable content structures — you are expected to customize the CSS
and JS for your project.

### Available blocks

| Block         | Type         | Description                                           |
|---------------|--------------|-------------------------------------------------------|
| **Accordion** | Block        | Collapsible content sections using `<details>`        |
| **Cards**     | Block        | Card grid with image + text per card                  |
| **Carousel**  | Block        | Rotating slides with navigation and indicators        |
| **Columns**   | Block        | Multi-column responsive layouts                       |
| **Embed**     | Block        | YouTube, Vimeo, Twitter, generic iframe embeds        |
| **Footer**    | Auto-loaded  | Site footer from dedicated page                       |
| **Form**      | Block        | Input controls (deprecated)                           |
| **Fragment**  | Block        | Inline content from another page                      |
| **Header**    | Auto-loaded  | Site header/nav from dedicated page                   |
| **Hero**      | Block        | Full-width banner with background image               |
| **Modal**     | Autoblock    | Overlay popup triggered by link patterns              |
| **Quote**     | Block        | Styled quotation with attribution                     |
| **Search**    | Block        | Query input with filtered results                     |
| **Table**     | Block        | Semantic HTML table from block content                |
| **Tabs**      | Block        | Tabbed content panels with ARIA roles                 |
| **Video**     | Block        | YouTube, Vimeo, MP4 with autoplay/background support  |

### How to import from the block collection

The block collection is open source at `github.com/adobe/aem-block-collection`. To use a block:

1. **Copy the block folder** from the collection into your project's `blocks/` directory
2. **Adapt the CSS** to match your project's design system
3. **Adapt the JS** if you need different DOM structure or behavior
4. **Create the `_blockname.json`** model/definition/filter for Universal Editor integration
5. **Add the block ID to your section filter** in `component-filters.json`

```bash
# Example: copy the tabs block
cp -r aem-block-collection/blocks/tabs ./blocks/tabs

# Then create blocks/tabs/_tabs.json with your model definition
```

The primary value of block collection blocks is the **content structure** they provide, not the
styling. All CSS and JS should be customized per project.

### Quote block (additional example from collection)

**Component model**:

```json
{
  "definitions": [
    {
      "title": "Quote",
      "id": "quote",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/block/v1/block",
            "template": {
              "name": "Quote",
              "model": "quote",
              "quote": "<p>Think, McFly! Think!</p>",
              "author": "Biff Tannen"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "quote",
      "fields": [
        {
          "component": "richtext",
          "name": "quote",
          "value": "",
          "label": "Quote",
          "valueType": "string"
        },
        {
          "component": "text",
          "valueType": "string",
          "name": "author",
          "label": "Author",
          "value": ""
        }
      ]
    }
  ],
  "filters": []
}
```

### Modal block (autoblock)

The modal is an **autoblock** — it does not require an author to create a table. Instead, links
matching a specific pattern (e.g., `#modal-name`) are automatically converted into modal triggers
during the `buildAutoBlocks()` phase in `scripts.js`.

---

## 5. Universal Editor Component Models — Complete Reference

### 5.1 Field component types

| Component Type           | Description                                           | Value Type          |
|--------------------------|-------------------------------------------------------|---------------------|
| `text`                   | Single-line text input                                | `string`            |
| `textarea`               | Multi-line plain text                                 | `string`            |
| `richtext`               | Rich text with HTML formatting                        | `string`            |
| `number`                 | Numeric input                                         | `number`            |
| `boolean`                | Toggle switch                                         | `boolean`           |
| `select`                 | Single-option dropdown                                | `string`            |
| `multiselect`            | Multi-option dropdown                                 | `string[]` (array)  |
| `checkbox-group`         | Multiple checkboxes                                   | `string[]`          |
| `radio-group`            | Radio button group                                    | `string`            |
| `reference`              | AEM asset picker (images/assets only)                 | `string`            |
| `aem-content`            | AEM content/page picker (any resource)                | `string`            |
| `aem-tag`                | AEM tag picker                                        | `string`            |
| `aem-content-fragment`   | Content Fragment picker                               | `string`            |
| `aem-experience-fragment`| Experience Fragment picker                            | `string`            |
| `date-time`              | Date and/or time picker                               | `date` / `date-time`|
| `container`              | Groups nested fields; supports `multi` for repeating  | (composite)         |
| `tab`                    | Visual tab separator in the properties panel          | (UI only)           |

### 5.2 Field properties

| Property      | Type        | Required | Description                                         |
|---------------|-------------|----------|-----------------------------------------------------|
| `component`   | string      | Yes      | Field type (see table above)                        |
| `name`        | string      | Yes      | Property name (maps to JCR property)                |
| `label`       | string      | Yes      | Display label in Universal Editor                   |
| `description` | string      | No       | Help text below the field                           |
| `value`       | any         | No       | Default value                                       |
| `valueType`   | string      | No       | `string`, `string[]`, `number`, `date`, `boolean`   |
| `required`    | boolean     | No       | Whether the field is mandatory                      |
| `readOnly`    | boolean     | No       | Read-only field                                     |
| `hidden`      | boolean     | No       | Hidden from the UI                                  |
| `multi`       | boolean     | No       | Enable multi-value (repeating) fields               |
| `options`     | array       | No       | Choices for select/multiselect/radio/checkbox        |
| `condition`   | object      | No       | Conditional visibility rules                        |
| `validation`  | object      | No       | Validation constraints                              |

> **Important**: Underscores (`_`) are not permitted in field `name` values when using `aem` or
> `xwalk` plugins, EXCEPT for element grouping prefixes (e.g., `textContent_title`).

### 5.3 Content modeling techniques

#### Field collapse

When field names follow the pattern `<base>` + `<Suffix>`, EDS collapses them into a single
HTML element:

| Suffix      | Effect                                           |
|-------------|--------------------------------------------------|
| `Alt`       | Becomes `alt` attribute on `<img>`               |
| `Text`      | Becomes link text for `<a>`                      |
| `Title`     | Becomes `title` attribute                        |
| `Type`      | Determines element type (e.g., `h1` vs `h2`)    |
| `MimeType`  | Determines rendering behavior                    |

Example: `image` + `imageAlt` collapses to `<img src="..." alt="...">`.

#### Element grouping

Fields sharing a prefix separated by `_` are grouped into a single `<div>`:

```json
{
  "name": "teaserText_title",
  "name": "teaserText_titleType",
  "name": "teaserText_cta",
  "name": "teaserText_ctaText"
}
```

This renders as a single cell containing the title and CTA, simplifying CSS styling.

#### Type inference

EDS automatically infers how to render values:
- Values referencing assets with `image/*` MIME types render as `<picture><img>`
- Values starting with `https?://` render as `<a>` links
- Values starting with block elements (`<p>`, `<h1>`) preserve rich text formatting
- The `classes` field is treated as block variant/option CSS classes

### 5.4 Block types in component definitions

| resourceType                                          | Usage                         |
|-------------------------------------------------------|-------------------------------|
| `core/franklin/components/text/v1/text`               | Default text component        |
| `core/franklin/components/title/v1/title`             | Default title (heading)       |
| `core/franklin/components/image/v1/image`             | Default image                 |
| `core/franklin/components/button/v1/button`           | Default button/link           |
| `core/franklin/components/section/v1/section`         | Section container             |
| `core/franklin/components/columns/v1/columns`         | Columns layout                |
| `core/franklin/components/block/v1/block`             | Generic named block           |
| `core/franklin/components/block/v1/block/item`        | Item within a container block |

### 5.5 Key-value blocks

For configuration-style blocks (like Metadata or Section Metadata), set `"key-value": true` in
the template:

```json
{
  "template": {
    "name": "Metadata",
    "model": "page-metadata",
    "key-value": true
  }
}
```

### 5.6 Container blocks vs simple blocks

**Simple blocks** have a single model — all fields render as rows:

```json
{
  "template": {
    "name": "Hero",
    "model": "hero"
  }
}
```

**Container blocks** have a filter instead of (or in addition to) a model — they accept child
items:

```json
{
  "template": {
    "name": "Cards",
    "filter": "cards"
  }
}
```

Each child item uses `block/v1/block/item` as its resourceType and has its own model.

---

## 6. Section Model Example

### Complete `models/_section.json`

This file is stored at `models/_section.json` in the project root (not inside a block folder).
It defines the section container, its metadata model, and the filter controlling which blocks
authors can add.

```json
{
  "definitions": [
    {
      "title": "Section",
      "id": "section",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/franklin/components/section/v1/section",
            "template": {
              "model": "section"
            }
          }
        }
      }
    }
  ],
  "models": [
    {
      "id": "section",
      "fields": [
        {
          "component": "text",
          "name": "name",
          "label": "Section Name",
          "description": "The label shown for this section in the Content Tree"
        },
        {
          "component": "multiselect",
          "name": "style",
          "label": "Style",
          "valueType": "array",
          "options": [
            { "name": "Highlight", "value": "highlight" },
            { "name": "Dark", "value": "dark" },
            { "name": "Full Width", "value": "full-width" },
            { "name": "Centered", "value": "centered" }
          ]
        },
        {
          "component": "reference",
          "name": "background",
          "label": "Background Image",
          "multi": false
        },
        {
          "component": "select",
          "name": "backgroundColor",
          "label": "Background Color",
          "valueType": "string",
          "options": [
            { "name": "Default", "value": "" },
            { "name": "Light Gray", "value": "bg-light-gray" },
            { "name": "Dark", "value": "bg-dark" },
            { "name": "Brand", "value": "bg-brand" }
          ]
        }
      ]
    }
  ],
  "filters": [
    {
      "id": "section",
      "components": [
        "text",
        "image",
        "button",
        "title",
        "hero",
        "cards",
        "columns",
        "tabs",
        "accordion",
        "carousel",
        "teaser",
        "fragment",
        "embed",
        "video",
        "table",
        "quote",
        "search"
      ]
    }
  ]
}
```

### How section styles map to CSS

The `style` multiselect values become CSS classes on the `.section` element:

```css
/* Section style variants */
.section.highlight {
  background-color: var(--highlight-background-color);
}

.section.dark {
  background-color: var(--dark-color);
  color: var(--background-color);
}

.section.full-width > div {
  max-width: unset;
  padding: 0;
}

.section.centered {
  text-align: center;
}
```

### Page metadata model

The `page-metadata` model is a reserved ID that adds fields to the page properties panel:

```json
{
  "id": "page-metadata",
  "fields": [
    {
      "component": "text",
      "valueType": "string",
      "name": "jcr:title",
      "label": "Title"
    },
    {
      "component": "text",
      "valueType": "string",
      "name": "jcr:description",
      "label": "Description"
    },
    {
      "component": "text",
      "valueType": "string",
      "name": "keywords",
      "multi": true,
      "label": "Keywords"
    }
  ]
}
```

Auto-mapped AEM properties: `title`, `description`, `robots`, `canonical`, `keywords`,
`cq-tags`, `published-time`.

---

## 7. Block Options (Variants) Quick Reference

Block options let authors choose visual variants via a `classes` field in the model. The selected
values become CSS classes on the block's root element.

### Single-select option

```json
{
  "component": "select",
  "name": "classes",
  "value": "",
  "label": "Style",
  "valueType": "string",
  "options": [
    { "name": "Default", "value": "" },
    { "name": "Wide", "value": "wide" },
    { "name": "Compact", "value": "compact" }
  ]
}
```

Result: `<div class="block hero wide">` when "Wide" is selected.

### Multi-select option

```json
{
  "component": "multiselect",
  "name": "classes",
  "value": [],
  "label": "Options",
  "valueType": "array",
  "options": [
    { "name": "Side-by-side", "value": "side-by-side" },
    { "name": "Image Left", "value": "left" },
    { "name": "Image Right", "value": "right" }
  ]
}
```

Result: `<div class="block teaser side-by-side left">` when both are selected.

### Default option on creation

Set the default in the block definition template:

```json
{
  "template": {
    "name": "Teaser",
    "model": "teaser",
    "classes": "side-by-side"
  }
}
```

### Detecting options in JS

```javascript
export default function decorate(block) {
  const options = [...block.classList].filter(
    (c) => !['block', 'teaser'].includes(c)
  );
  if (options.includes('side-by-side')) {
    // apply side-by-side layout
  }
}
```

---

## 8. JSON Fragment File Convention

In the XWalk boilerplate, each block has a JSON fragment file at
`blocks/<blockname>/_<blockname>.json`. This file contains three arrays:

```json
{
  "definitions": [ /* component definitions */ ],
  "models": [ /* component models */ ],
  "filters": [ /* component filters */ ]
}
```

A **Husky pre-commit hook** automatically compiles all `_*.json` fragment files into the
root-level `component-definition.json`, `component-models.json`, and `component-filters.json`
files on each `git commit`. You should never edit the root-level files directly.

Section-level configuration is stored at `models/_section.json` following the same structure.

### Build command

```bash
# The pre-commit hook runs automatically, but you can also trigger manually:
npm run build:json
```

---

## 9. Block Development Workflow Summary

1. **Create the block folder**: `blocks/<blockname>/`
2. **Create the JSON fragment**: `blocks/<blockname>/_<blockname>.json` with definitions, models,
   filters
3. **Add the block to the section filter**: Update `models/_section.json` to include the new
   block ID in the section filter's components array
4. **Commit** — the Husky hook compiles JSON files automatically
5. **Preview in Universal Editor** — use `?ref=<branch>` to test on a feature branch
6. **Create content** — add the block to a page via Universal Editor
7. **Implement CSS**: `blocks/<blockname>/<blockname>.css`
8. **Implement JS** (if needed): `blocks/<blockname>/<blockname>.js` with
   `export default function decorate(block) { ... }`
9. **Lint**: `npm run lint`
10. **Merge to main** — block goes live

### Best practices

- Prefer CSS-only styling over JavaScript DOM manipulation
- When JS is needed, minimize DOM changes to avoid breaking Universal Editor instrumentation
- Use `moveInstrumentation()` from `scripts.js` when moving/replacing elements to preserve
  `data-aue-*` attributes
- Scope all CSS selectors to the block class: `.block.hero { ... }`
- Use CSS nesting for clean, maintainable stylesheets
- Keep block names lowercase, hyphenated (e.g., `my-block`)
- Each block's JS is loaded as an ESM module — use `import`/`export`
