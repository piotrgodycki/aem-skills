---
title: Style System Implementation
impact: MEDIUM-HIGH
impactDescription: Proper Style System usage enables reusable component variations without code duplication — incorrect patterns create QA nightmares and author confusion
tags: style-system, css, scss, bem, policy, template, layout-styles, display-styles
---

## Style System Implementation

The Style System allows authors to select visual variations for components. Selected styles inject CSS classes into the component's outer wrapper div, configured via content policies on editable templates.

---

### 1. How It Works

```
1. Define style groups in component policy (Template Editor → Structure → Policy)
2. Each style maps an author-facing name to one or more CSS classes
3. Authors select styles in the content toolbar (paintbrush icon)
4. AEM injects the CSS class on the component's OUTER WRAPPER div
```

---

### 2. Style Types

#### Layout Styles — Single Selection

Affect the entire component's rendition. Only **one** layout style active at a time.

Examples: "Card", "Hero", "Horizontal", "Full Width"

```
Group: Layout
Type: Single selection
Options:
  "Default"      → (no class)
  "Card"         → cmp-teaser--card
  "Hero"         → cmp-teaser--hero
  "Horizontal"   → cmp-teaser--horizontal
```

#### Display Styles — Multi Selection

Minor visual variations. **Multiple** can be active simultaneously.

Examples: "Primary Color", "Underline", "Dark Background", "No Padding"

```
Group: Decoration
Type: Multi selection
Options:
  "Underline"     → cmp-title--underline
  "Primary Color"  → cmp-title--primary-color
  "Centered"       → cmp-title--centered
```

---

### 3. HTML Output

```html
<!-- No style applied -->
<div class="title aem-GridColumn aem-GridColumn--default--12">
    <div class="cmp-title">
        <h2 class="cmp-title__text">Hello World</h2>
    </div>
</div>

<!-- Layout style "Hero" + display style "Underline" applied -->
<div class="title cmp-title--hero cmp-title--underline aem-GridColumn aem-GridColumn--default--12">
    <div class="cmp-title">
        <h2 class="cmp-title__text">Hello World</h2>
    </div>
</div>
```

**Important:** Style classes go on the **outer wrapper** (`div.title`), not on the BEM block (`.cmp-title`). Your CSS must account for this.

---

### 4. Policy Configuration

#### XML Policy Definition

```xml
<!-- /conf/myproject/settings/wcm/policies/myproject/components/teaser/branded -->
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="Branded Teaser Policy"
    sling:resourceType="wcm/core/components/policy/policy">
    <cq:styleGroups jcr:primaryType="nt:unstructured">
        <item0
            jcr:primaryType="nt:unstructured"
            cq:styleGroupLabel="Layout">
            <cq:styles jcr:primaryType="nt:unstructured">
                <item0
                    jcr:primaryType="nt:unstructured"
                    cq:styleClasses="cmp-teaser--card"
                    cq:styleId="card"
                    cq:styleLabel="Card"/>
                <item1
                    jcr:primaryType="nt:unstructured"
                    cq:styleClasses="cmp-teaser--hero"
                    cq:styleId="hero"
                    cq:styleLabel="Hero"/>
                <item2
                    jcr:primaryType="nt:unstructured"
                    cq:styleClasses="cmp-teaser--horizontal"
                    cq:styleId="horizontal"
                    cq:styleLabel="Horizontal"/>
            </cq:styles>
        </item0>
        <item1
            jcr:primaryType="nt:unstructured"
            cq:styleGroupLabel="Background">
            <cq:styles jcr:primaryType="nt:unstructured">
                <item0
                    jcr:primaryType="nt:unstructured"
                    cq:styleClasses="cmp-teaser--dark-bg"
                    cq:styleId="dark-bg"
                    cq:styleLabel="Dark Background"/>
                <item1
                    jcr:primaryType="nt:unstructured"
                    cq:styleClasses="cmp-teaser--brand-bg"
                    cq:styleId="brand-bg"
                    cq:styleLabel="Brand Background"/>
            </cq:styles>
        </item1>
    </cq:styleGroups>
</jcr:root>
```

#### Policy Mapping on Template

```xml
<!-- /conf/myproject/settings/wcm/templates/page/policies/jcr:content/root/responsivegrid/teaser -->
<teaser
    jcr:primaryType="nt:unstructured"
    cq:policy="/conf/myproject/settings/wcm/policies/myproject/components/teaser/branded"/>
```

---

### 5. CSS Organization Pattern

#### Base Styles (No Style System Class)

```scss
// Default component appearance — always applied
.cmp-teaser {
  margin-bottom: $spacing-lg;
}

.cmp-teaser__title {
  font-family: $font-family-heading;
  font-size: 1.5rem;
  margin-bottom: $spacing-sm;
}

.cmp-teaser__description {
  color: $color-text-secondary;
  line-height: 1.6;
}

.cmp-teaser__action-link {
  display: inline-block;
  padding: $spacing-sm $spacing-md;
  background: $brand-primary;
  color: white;
  text-decoration: none;
  border-radius: 4px;
}
```

#### Layout Style Variations

```scss
// Card layout
.cmp-teaser--card {
  .cmp-teaser {
    border: 1px solid $color-border;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
    transition: box-shadow 0.2s;

    &:hover {
      box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
    }
  }

  .cmp-teaser__content {
    padding: $spacing-md;
  }

  .cmp-teaser__image img {
    aspect-ratio: 16 / 9;
    object-fit: cover;
  }
}

// Hero layout
.cmp-teaser--hero {
  .cmp-teaser {
    position: relative;
    min-height: 60vh;
    display: flex;
    align-items: flex-end;
  }

  .cmp-teaser__image {
    position: absolute;
    inset: 0;
    img { width: 100%; height: 100%; object-fit: cover; }
    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(transparent 40%, rgba(0, 0, 0, 0.7));
    }
  }

  .cmp-teaser__content {
    position: relative;
    color: white;
    padding: $spacing-xl;
    max-width: 600px;
  }

  .cmp-teaser__title { font-size: 3rem; color: white; }
}

// Horizontal layout
.cmp-teaser--horizontal {
  .cmp-teaser {
    display: flex;
    flex-direction: row;
    gap: $spacing-md;

    @include respond-to(phone) {
      flex-direction: column;
    }
  }

  .cmp-teaser__image {
    flex: 0 0 40%;
    img { aspect-ratio: 4 / 3; object-fit: cover; }
  }

  .cmp-teaser__content {
    flex: 1;
    display: flex;
    flex-direction: column;
    justify-content: center;
  }
}
```

#### Display Style Variations

```scss
// Dark background (works with any layout)
.cmp-teaser--dark-bg {
  .cmp-teaser {
    background-color: $color-bg-dark;
  }
  .cmp-teaser__title,
  .cmp-teaser__description {
    color: white;
  }
}

// Brand background
.cmp-teaser--brand-bg {
  .cmp-teaser {
    background-color: $brand-primary;
  }
  .cmp-teaser__title,
  .cmp-teaser__description,
  .cmp-teaser__action-link {
    color: white;
  }
  .cmp-teaser__action-link {
    background: white;
    color: $brand-primary;
  }
}
```

#### Layout + Display Combinations

```scss
// Hero with dark background — specific combination override
.cmp-teaser--hero.cmp-teaser--dark-bg {
  .cmp-teaser__image::after {
    background: linear-gradient(transparent 20%, rgba(0, 0, 0, 0.85));
  }
}

// Card with brand background — specific combination
.cmp-teaser--card.cmp-teaser--brand-bg {
  .cmp-teaser {
    border-color: $brand-primary;
  }
}
```

---

### 6. Common Style Patterns

#### Title Component Styles

```scss
// Underline decoration
.cmp-title--underline {
  .cmp-title__text::after {
    display: block;
    width: 84px;
    padding-top: 8px;
    content: '';
    border-bottom: 2px solid $brand-primary;
  }
}

// Centered title
.cmp-title--centered {
  .cmp-title { text-align: center; }
  .cmp-title__text::after { margin: 8px auto 0; }
}

// Display size overrides
.cmp-title--display-large .cmp-title__text { font-size: 3.5rem; }
.cmp-title--display-small .cmp-title__text { font-size: 1.25rem; }
```

#### Container Styles

```scss
// Full-width background
.cmp-container--full-width {
  .cmp-container {
    max-width: none;
    padding: $spacing-xl $spacing-lg;
  }
}

// Narrow content container
.cmp-container--narrow {
  .cmp-container {
    max-width: 720px;
    margin: 0 auto;
  }
}

// Background variations
.cmp-container--bg-light { .cmp-container { background: $color-bg-light; } }
.cmp-container--bg-dark {
  .cmp-container { background: $color-bg-dark; }
  .cmp-title__text, .cmp-text { color: white; }
}
```

---

### 7. Migration from Design Dialog to Style System

```
Design Dialog (Classic/Legacy)        →  Style System (Modern)
cq:design_dialog → per-page config   →  Content Policy → per-template config

Steps:
1. Identify design dialog options that are visual variations
2. Create Style System groups and CSS classes
3. Configure policies on templates
4. Remove design dialog dependency
5. Re-author existing pages (select styles)
```

---

### 8. Best Practices

1. **Decouple author labels from CSS**: Author sees "Primary", CSS uses `.cmp-component--primary-color`
2. **Scope styles tightly**: Always scope to the target component
3. **Target BEM classes**: Use `.cmp-title__text` not `h2`
4. **Minimize options**: Only expose variations allowed by brand standards
5. **Use semantic naming**: Name CSS classes for purpose, not appearance (`.cmp-teaser--featured`, not `.cmp-teaser--blue`)
6. **Document combinations**: Test and document which layout + display combinations are valid
7. **Reuse policies**: Same policy can be shared across templates

---

### 9. Anti-Patterns

#### Exposing Too Many Style Options

```
// WRONG — 5 layout x 4 background x 3 alignment = 60 combinations to QA
// Authors confused by irrelevant options

// CORRECT — curated set of 2-3 layout options + 1-2 display options
// Only expose what the design system actually uses
```

#### Targeting HTML Elements

```scss
// WRONG — fragile, breaks on Core Component updates
.cmp-title--underline h2 { border-bottom: 2px solid blue; }

// CORRECT — target BEM class
.cmp-title--underline .cmp-title__text { border-bottom: 2px solid $brand-primary; }
```

#### Style Names That Match CSS Class Names

```
// WRONG — author sees "cmp-teaser--card" (implementation detail)
// CORRECT — author sees "Card Layout" (semantic label)
```

#### Display Styles That Conflict

```scss
// WRONG — both set background color, one wins arbitrarily
// "Light Background" + "Dark Background" — which one wins?

// CORRECT — put mutually exclusive styles in a Layout (single-select) group
// Or document that only one background style should be selected
```

#### Mixing Layout Concerns into Display Styles

```scss
// WRONG — display style changes layout
.cmp-teaser--highlighted {
  display: flex;
  flex-direction: row; // This is a layout change, not display
}

// CORRECT — layout changes go in Layout group (single-select)
// Display styles only modify colors, borders, shadows, etc.
```
