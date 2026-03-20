---
title: Core Components Frontend Patterns
impact: CRITICAL
impactDescription: Core Components are the foundation of AEM sites — incorrect customization breaks upgrades and voids support
tags: core-components, bem, css, proxy-component, htl, sling-model, delegation, customization
---

## Core Components Frontend Patterns

AEM Core Components provide production-ready, extensible components. Their CSS classes follow BEM naming with a `cmp-` namespace and are considered the stable API for styling. Never modify Core Components directly — always proxy.

---

### 1. BEM Naming Convention

**Pattern**: `cmp-{blockname}__{element}--{modifier}`

```html
<!-- Tabs component — full BEM structure -->
<div class="cmp-tabs">
  <ol class="cmp-tabs__tablist" role="tablist">
    <li class="cmp-tabs__tab cmp-tabs__tab--active" role="tab" aria-selected="true">Tab 1</li>
    <li class="cmp-tabs__tab" role="tab">Tab 2</li>
  </ol>
  <div class="cmp-tabs__tabpanel cmp-tabs__tabpanel--active" role="tabpanel">Content 1</div>
  <div class="cmp-tabs__tabpanel" role="tabpanel" hidden>Content 2</div>
</div>

<!-- Teaser component -->
<div class="cmp-teaser">
  <div class="cmp-teaser__image">
    <img class="cmp-image__image" src="..." alt="..." loading="lazy"/>
  </div>
  <div class="cmp-teaser__content">
    <h2 class="cmp-teaser__title">
      <a class="cmp-teaser__title-link" href="/en/page.html">Title</a>
    </h2>
    <div class="cmp-teaser__description"><p>Description text</p></div>
    <div class="cmp-teaser__action-container">
      <a class="cmp-teaser__action-link" href="/en/page.html">Read More</a>
    </div>
  </div>
</div>
```

### Key BEM Rules

- **Block**: Independent entity (`cmp-tabs`, `cmp-teaser`, `cmp-image`)
- **Element**: Part of a block, uses `__` separator (`cmp-tabs__tab`)
- **Modifier**: State/variation, uses `--` separator (`cmp-tabs__tab--active`)
- **Namespace**: Always `cmp-` prefix to avoid conflicts
- **No `--is-` prefix**: Core Components use `--active`, NOT `--is-active`

---

### 2. Stable CSS Classes (API)

These classes are safe to target — they won't change between Core Component versions:

| Component | Block Class | Key Elements |
|-----------|------------|--------------|
| Accordion | `cmp-accordion` | `__item`, `__header`, `__button`, `__panel` |
| Breadcrumb | `cmp-breadcrumb` | `__list`, `__item`, `__item-link` |
| Button | `cmp-button` | `__text`, `__icon` |
| Carousel | `cmp-carousel` | `__item`, `__action`, `__indicator` |
| Container | `cmp-container` | — |
| Image | `cmp-image` | `__image`, `__title` |
| List | `cmp-list` | `__item`, `__item-link`, `__item-title` |
| Navigation | `cmp-navigation` | `__group`, `__item`, `__item-link` |
| Search | `cmp-search` | `__form`, `__input`, `__results` |
| Tabs | `cmp-tabs` | `__tablist`, `__tab`, `__tabpanel` |
| Teaser | `cmp-teaser` | `__image`, `__content`, `__title`, `__description` |
| Text | `cmp-text` | — (content is direct children) |
| Title | `cmp-title` | `__text`, `__link` |

---

### 3. Proxy Component Pattern (Required)

Never modify Core Component code. Create a proxy component:

#### Minimal Proxy (.content.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    jcr:primaryType="cq:Component"
    jcr:title="Title"
    jcr:description="Displays a page title or custom title"
    sling:resourceSuperType="core/wcm/components/title/v3/title"
    componentGroup="My Project - Content"/>
```

#### Proxy with Dialog Override

```
/apps/myproject/components/title/
├── .content.xml                    # sling:resourceSuperType → core title
├── _cq_dialog/
│   └── .content.xml                # Custom dialog additions (merged via Sling Resource Merger)
└── title.html                      # Optional HTL override
```

#### Complete Proxy Project Structure

```
/apps/myproject/components/
├── title/            → core/wcm/components/title/v3/title
├── text/             → core/wcm/components/text/v2/text
├── image/            → core/wcm/components/image/v3/image
├── teaser/           → core/wcm/components/teaser/v2/teaser
├── list/             → core/wcm/components/list/v4/list
├── tabs/             → core/wcm/components/tabs/v1/tabs
├── accordion/        → core/wcm/components/accordion/v1/accordion
├── carousel/         → core/wcm/components/carousel/v1/carousel
├── container/        → core/wcm/components/container/v1/container
├── button/           → core/wcm/components/button/v2/button
├── navigation/       → core/wcm/components/navigation/v2/navigation
├── breadcrumb/       → core/wcm/components/breadcrumb/v3/breadcrumb
└── page/             → core/wcm/components/page/v3/page
```

---

### 4. CSS Customization Strategies

#### Strategy 1: BEM Class Targeting (Recommended)

```scss
// Target stable BEM classes — survives Core Component upgrades
.cmp-teaser__title {
  font-size: 1.5rem;
  font-family: $font-family-heading;
  margin-bottom: $spacing-sm;
}

.cmp-teaser__description {
  color: $color-text-secondary;
  line-height: 1.6;
}

.cmp-teaser__action-link {
  @include button-primary;
}
```

#### Strategy 2: Style System Integration

```scss
// Style System applies classes to the outer wrapper div
.cmp-teaser--hero {
  .cmp-teaser__image {
    height: 60vh;
    overflow: hidden;
  }
  .cmp-teaser__content {
    position: absolute;
    bottom: 0;
    color: white;
    padding: $spacing-lg;
  }
}

.cmp-teaser--card {
  border: 1px solid $color-border;
  border-radius: 8px;
  overflow: hidden;
  .cmp-teaser__content {
    padding: $spacing-md;
  }
}
```

#### Strategy 3: Context-Based Scoping

```scss
// Scope by container context
.cmp-container--sidebar {
  .cmp-teaser {
    // Teasers inside sidebar render smaller
    .cmp-teaser__title { font-size: 1rem; }
    .cmp-teaser__image img { aspect-ratio: 1; }
  }
}
```

**Incorrect — targeting HTML elements:**

```scss
/* WRONG — fragile, breaks if Core Component HTML changes */
.title h2 { color: red; }
div > p { margin: 0; }
.teaser a:first-child { font-weight: bold; }

/* CORRECT — target BEM classes */
.cmp-title__text { color: red; }
.cmp-text p { margin: 0; }
.cmp-teaser__title-link { font-weight: bold; }
```

---

### 5. Sling Model Delegation Pattern

Extend behavior while inheriting all Core Component functionality:

```java
@Model(
    adaptables = SlingHttpServletRequest.class,
    adapters = Title.class,
    resourceType = "myproject/components/title"
)
public class CustomTitle implements Title {

    @Self
    @Via(type = ResourceSuperType.class)
    private Title delegate; // Core Component's Title model

    @ValueMapValue(name = "analyticsId")
    @Default(values = "")
    private String analyticsId;

    // Override specific methods
    @Override
    public String getText() {
        String text = delegate.getText();
        return StringUtils.isNotBlank(text) ? text : "Default Title";
    }

    // Delegate all other methods
    @Override
    public String getType() { return delegate.getType(); }

    @Override
    public String getLinkURL() { return delegate.getLinkURL(); }

    @Override
    public boolean isLinkDisabled() { return delegate.isLinkDisabled(); }

    @Override
    public String getId() { return delegate.getId(); }

    @Override
    @JsonProperty("dataLayer")
    public ComponentData getData() { return delegate.getData(); }

    // Custom method (accessible in HTL)
    public String getAnalyticsId() { return analyticsId; }
}
```

---

### 6. Core Component JS Hooks and Data Attributes

Core Components use `data-cmp-is` for JavaScript initialization:

```html
<div class="cmp-tabs" data-cmp-is="tabs">
  <!-- JS hooks: data-cmp-hook-tabs__tab, data-cmp-hook-tabs__tabpanel -->
</div>
```

**Extend JS behavior:**

```javascript
// Listen for Core Component initialization events
(function() {
    'use strict';

    const DATA_ATTRIBUTE = 'data-cmp-is';

    // Hook into the tabs component lifecycle
    function onDocumentReady() {
        const tabsElements = document.querySelectorAll(`[${DATA_ATTRIBUTE}="tabs"]`);
        tabsElements.forEach((tabs) => {
            // Add custom behavior after Core Component initializes
            const observer = new MutationObserver((mutations) => {
                mutations.forEach((mutation) => {
                    if (mutation.attributeName === 'class') {
                        trackTabChange(mutation.target);
                    }
                });
            });
            tabs.querySelectorAll('[data-cmp-hook-tabs="tabpanel"]').forEach((panel) => {
                observer.observe(panel, { attributes: true });
            });
        });
    }

    if (document.readyState !== 'loading') {
        onDocumentReady();
    } else {
        document.addEventListener('DOMContentLoaded', onDocumentReady);
    }
})();
```

---

### 7. Component Versioning and Upgrades

```
Version naming: core/wcm/components/{name}/v{N}/{name}
  v1 → v2 → v3 (breaking changes between major versions)

Upgrade strategy:
  1. Check Core Components release notes
  2. Update proxy sling:resourceSuperType to new version
  3. Test all component instances
  4. Update any custom HTL overrides
  5. Verify Style System classes still apply correctly
```

**Correct proxy update:**

```xml
<!-- Upgrade title from v2 to v3 -->
<jcr:root
    sling:resourceSuperType="core/wcm/components/title/v3/title"
    .../>
<!-- Previously: core/wcm/components/title/v2/title -->
```

---

### 8. Image Component Responsive Behavior

```html
<!-- Core Image with web-optimized delivery -->
<div class="cmp-image" data-cmp-is="image"
     data-cmp-src="/content/dam/mysite/hero.coreimg{.width}.jpeg"
     data-cmp-widths="[320,480,640,800,1024,1280,1920]"
     data-cmp-lazy
     data-cmp-lazy-threshold="0">
    <img class="cmp-image__image"
         src="/content/dam/mysite/hero.coreimg.jpeg/1920.jpeg"
         srcset="..."
         sizes="100vw"
         alt="Hero"
         loading="lazy"/>
</div>
```

**CSS for responsive image display:**

```scss
.cmp-image {
  &__image {
    width: 100%;
    height: auto;
    display: block; // Remove inline-block gap
  }
}

// Constrained image (within text)
.cmp-text .cmp-image {
  max-width: 100%;
  margin: $spacing-md 0;
}
```

---

### 9. Anti-Patterns

#### Modifying Core Component Code

```
// WRONG — editing core component files directly
/libs/core/wcm/components/title/v3/title/title.html ← NEVER modify

// CORRECT — proxy and override in /apps
/apps/myproject/components/title/title.html ← override via proxy
```

#### Targeting HTML Elements

```scss
// WRONG — will break when Core Component updates HTML structure
.teaser > div > a { color: red; }
h2 { font-size: 2rem; } // Affects all h2s on the page

// CORRECT — BEM classes are the stable API
.cmp-teaser__title-link { color: red; }
.cmp-title__text { font-size: 2rem; }
```

#### Forking Core Components

```
// WRONG — copying entire Core Component and modifying
/apps/myproject/components/image/ (copy of core image with edits)
// Lose all future updates, security patches, bug fixes

// CORRECT — proxy + HTL override + Sling Model delegation
```

#### Ignoring Data Layer Compatibility

```java
// WRONG — custom model without ComponentData support
@Model(adapters = Title.class, resourceType = "myproject/components/title")
public class CustomTitle implements Title {
    @Override
    public ComponentData getData() { return null; } // Breaks ACDL
}

// CORRECT — delegate getData() to Core Component
@Override
public ComponentData getData() { return delegate.getData(); }
```
