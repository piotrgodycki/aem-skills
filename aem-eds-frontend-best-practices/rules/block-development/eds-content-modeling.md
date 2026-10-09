## EDS Content Modeling & Block Definitions

Block definitions, content models, and filters are defined as JSON fragments in `_<blockname>.json` files within each block folder. These are compiled into root-level `component-definition.json`, `component-models.json`, and `component-filters.json` by the Husky pre-commit hook.

> **Content-first, not field-first.** Model the block to match how authors naturally structure content, then write decoration code against it — even if that means more complex JS. Developer convenience is secondary to author experience (see `authoring/content-driven-development.md`). Pick the canonical model *before* listing fields.

### The 4 Canonical Block Models

Almost every EDS block maps to one of four shapes. Choosing the right one up front prevents author-hostile structures and brittle decoration code.

| Model | Shape | Use when | Examples |
|---|---|---|---|
| **Standalone** | One block, a fixed set of fields (one "instance") | The block is a single self-contained unit | Hero, Teaser, Embed, Quote |
| **Collection** | A repeating list of uniform items | Authors add N of the same thing | Cards, Carousel, Gallery, Accordion items |
| **Configuration** | A block whose rows are key/value settings that control dynamic output | The block *queries/renders* other content from settings | Blog listing, Search, Feed, Filtered index |
| **Auto-Blocked** | No author-created table — JS synthesises the block from plain content or a URL | A recognisable pattern (a bare video URL, a tab heading sequence) should "just work" | Video/YouTube embed, auto-tabs |

**Model-selection heuristics:**
- Repeating, uniform content → **Collection** (one container, rows = items). Never add a "header row" to a Collection — that's a Standalone concern.
- Fixed, named slots → **Standalone**.
- Rows are *settings* (e.g. `tag: news`, `limit: 10`) rather than rendered content → **Configuration**. ⚠️ Configuration blocks render content authors didn't type — document the contract and defaults carefully.
- Authors shouldn't have to create a block at all → **Auto-Blocked** (decorate in `scripts.js`/`buildAutoBlocks`). Use sparingly; only for unambiguous patterns.

**Incorrect — Collection modelled as Standalone:** giving a Cards block 6 fixed `card1Title`, `card1Image`, `card2Title`… fields. Authors can't add a 7th card; the model fights reality. Model it as a repeating collection instead.

### JSON Fragment Structure (`_<blockname>.json`)

Each fragment contains up to three sections:

```json
{
  "definitions": [...],
  "models": [...],
  "filters": [...]
}
```

### Block Definition

Tells the Universal Editor what the block is and how to create it:

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
  }]
}
```

- `title` — Display name in Universal Editor
- `id` — Unique identifier, matches the block folder name
- `template.name` — Block name as it appears in the DOM
- `template.model` — References a model `id` from the models array
- `template.classes` — Default block option value (empty = no variant)

### Content Model

Defines the editable fields for a block:

```json
{
  "models": [{
    "id": "teaser",
    "fields": [
      {
        "component": "reference",
        "name": "image",
        "label": "Image",
        "valueType": "string"
      },
      {
        "component": "text",
        "name": "imageAlt",
        "label": "Alt Text",
        "valueType": "string"
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
        "label": "CTA",
        "valueType": "string"
      },
      {
        "component": "text",
        "name": "textContent_ctaText",
        "label": "CTA Label",
        "valueType": "string"
      }
    ]
  }]
}
```

### Component Types

| Component | Purpose | Value Type |
|-----------|---------|------------|
| `text` | Single-line text input | `string` / `number` |
| `textarea` | Multi-line plain text | `string` |
| `richtext` | Rich text with formatting | `string` |
| `reference` | Asset picker (images, binaries) | `string` |
| `aem-content` | AEM content/page reference | `string` |
| `aem-content-fragment` | Content Fragment reference | `string` |
| `aem-experience-fragment` | Experience Fragment reference | `string` |
| `aem-tag` | AEM tag picker | `string` |
| `select` | Dropdown, predefined options | `string` / `number` |
| `multiselect` | Multiple selection | `string` (comma-separated) |
| `checkbox-group` | Multiple checkboxes | `string` |
| `radio-group` | Mutually-exclusive radios | `string` / `number` / `boolean` |
| `boolean` | Toggle switch | `boolean` |
| `number` | Numeric input | `number` |
| `date-time` | Date and time picker | `string` (ISO 8601) |
| `container` | Groups fields into a nested object | — |
| `tab` | Visual grouping tab (no data) | — |

**`valueType` constraints:** `aem-*`, `reference`, `richtext`, `textarea` are always `string`. `boolean` must be `boolean`. `number`/`date-time` default to their natural type. A mismatch between `component` and `valueType` is the most common reason a field silently fails to persist.

**Field-level validation** (Universal Editor enforces before save):
```json
{ "component": "text", "name": "zip", "label": "ZIP",
  "valueType": "string", "required": true,
  "validation": { "pattern": "^\\d{5}$", "patternMessage": "5 digits" } }
```
`TextValidation` (`pattern`, `minLength`, `maxLength`) applies to `text`; `NumberValidation` (`minimum`, `maximum`) to `number`; `required` to any field.

**Conditional fields** — show a field only when another has a given value:
```json
{ "component": "text", "name": "externalUrl", "label": "External URL",
  "condition": { "===": [ { "var": "linkType" }, "external" ] } }
```

### Field Naming — Grouping & Collapse

Field names control how content renders into block HTML.

#### Element Grouping (Prefix Convention)

Fields sharing a prefix before `_` render in the same container `<div>`:

```json
{ "name": "textContent_text", ... },
{ "name": "textContent_cta", ... },
{ "name": "textContent_ctaText", ... }
```

All three render inside a single `<div>` in the block's HTML.

#### Field Collapse

An image field + its alt text field collapse into a single `<img>` element:

```json
{ "component": "reference", "name": "image", ... },
{ "component": "text", "name": "imageAlt", ... }
```

Renders as: `<img src="..." alt="...">` (not two separate elements).

**Naming rule**: The alt field must be named `{imageFieldName}Alt` (e.g., `image` → `imageAlt`).

### Block Options (Variants)

Add a `select` field named `classes` to enable block variants:

```json
{
  "component": "select",
  "name": "classes",
  "value": "",
  "label": "Layout",
  "valueType": "string",
  "options": [
    { "name": "Default", "value": "" },
    { "name": "Side-by-side", "value": "side-by-side" },
    { "name": "Side-by-side Left", "value": "side-by-side left" }
  ]
}
```

Selected values become CSS classes on the block: `<div class="block teaser side-by-side">`

Multiple space-separated values in a single option create multiple CSS classes.

### Block Filters

Control which blocks are allowed in containers:

```json
{
  "filters": [{
    "id": "teaser-filter",
    "components": ["text", "image", "button"]
  }]
}
```

### Section Model (`models/_section.json`)

Controls which blocks are allowed in page sections:

```json
{
  "filters": [{
    "id": "section",
    "components": ["text", "image", "teaser", "columns", "hero"]
  }]
}
```

### Compilation Flow

1. Author edits `blocks/teaser/_teaser.json`
2. On `git commit`, Husky pre-commit hook runs
3. All `_*.json` fragments are merged into:
   - `component-definition.json`
   - `component-models.json`
   - `component-filters.json`
4. These root-level files are what Universal Editor reads

### Best Practices

1. **One JSON fragment per block** — `_<blockname>.json` inside the block folder
2. **Use prefixes for grouping** — fields that belong together share a prefix (`textContent_*`)
3. **Name image alt fields correctly** — `{image}Alt` for automatic collapse
4. **Always include `label`** on all fields — shown to authors in Universal Editor
5. **Use `tab` components** for complex blocks with many fields
6. **Keep `classes` options minimal** — each variant needs CSS and QA coverage
7. **Run `npm run lint`** — catches JSON syntax errors before commit

### Pitfalls

- Mismatched `model` reference between definition and model `id` — block won't be editable
- Forgetting the `Alt` suffix on image alt fields — breaks field collapse
- JSON syntax errors — caught by lint but break the build if missed
- Adding fields without corresponding JS/CSS handling — renders empty elements
- Editing `component-*.json` root files directly — overwritten by pre-commit hook
