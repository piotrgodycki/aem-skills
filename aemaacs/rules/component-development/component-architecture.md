---
title: AEM Component Architecture Patterns & Best Practices
impact: HIGH
impactDescription: Poor component architecture leads to content migration nightmares, broken upgrades, and unmaintainable codebases
tags: components, core-components, proxy, hierarchy, versioning, experience-fragments, content-fragments, container, decoration-tag, wcm-mode, page-component
---

## AEM Component Architecture Patterns

Practical patterns and anti-patterns for component development on AEM as a Cloud Service. Understanding the component hierarchy and following the proxy pattern prevents costly content migrations and ensures smooth Core Components upgrades.

---

### 1. Component Hierarchy: Foundation -> Core -> Proxy -> Custom

AEM components follow a layered inheritance model using `sling:resourceSuperType`:

```
Foundation Components (deprecated, do NOT use)
  └── /libs/wcm/foundation/components/

Core Components (maintained by Adobe, auto-updated on AEMaaCS)
  └── /libs/core/wcm/components/

Proxy Components (your project's thin wrappers)
  └── /apps/myproject/components/

Custom Components (project-specific, extend proxy or build from scratch)
  └── /apps/myproject/components/
```

**Correct -- proxy component pointing to Core Component:**

```xml
<!-- /apps/myproject/components/title/.content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="cq:Component"
    jcr:title="Title"
    jcr:description="Displays a page heading"
    sling:resourceSuperType="core/wcm/components/title/v3/title"
    componentGroup="My Project - Content"/>
```

**Incorrect -- content referencing versioned Core Component directly:**

```xml
<!-- NEVER do this in content nodes -->
<jcr:root
    jcr:primaryType="nt:unstructured"
    sling:resourceType="core/wcm/components/title/v2/title"/>
<!-- When Adobe releases v3, you must migrate ALL content nodes -->
```

**Correct -- content references unversioned proxy:**

```xml
<jcr:root
    jcr:primaryType="nt:unstructured"
    sling:resourceType="myproject/components/title"/>
<!-- To upgrade to v3, only change sling:resourceSuperType in the proxy -->
```

**Key rule:** Content `sling:resourceType` must NEVER contain a version number. Only the proxy component's `sling:resourceSuperType` should reference a versioned path.

---

### 2. Component Groups and Organization

Organize components into logical groups that appear in the AEM editor's component browser:

```
My Project - Content        (text, image, title, teaser, list)
My Project - Structure      (container, tabs, accordion, carousel)
My Project - Commerce       (product teaser, cart, checkout)
My Project - Forms          (form container, text input, submit)
My Project - Hidden         (components not for direct authoring)
```

**Proxy component group assignment:**

```xml
<!-- Set componentGroup on the proxy component node -->
<jcr:root
    jcr:primaryType="cq:Component"
    jcr:title="Teaser"
    sling:resourceSuperType="core/wcm/components/teaser/v2/teaser"
    componentGroup="My Project - Content"/>
```

**Pro tip:** Use `.hidden` suffix or set `componentGroup=""` for components that should only be included programmatically (via HTL `data-sly-resource`) and never dragged from the component browser.

---

### 3. Container Components vs Content Components

Understanding this distinction is fundamental to AEM component architecture.

#### Content Components
Render specific content (text, image, title). They do NOT allow child components.

```html
<!-- Content component: renders its own content, no children -->
<div class="cmp-title" data-cmp-data-layer='${title.data.json}'>
    <h2 class="cmp-title__text">${title.text}</h2>
</div>
```

#### Container Components
Accept child components via a parsys (paragraph system). They define layout and structure.

```html
<!-- Container component with responsive grid -->
<div class="cmp-container">
    <div class="aem-Grid aem-Grid--12">
        <sly data-sly-resource="${resource.path @ resourceType='wcm/foundation/components/responsivegrid'}" />
    </div>
</div>
```

**Core container components and their purposes:**

| Component | Purpose | Frontend Consideration |
|-----------|---------|----------------------|
| Container | Generic wrapper with layout policies | Generates `aem-Grid` classes for responsive grid |
| Tabs | Tabbed content panels | Requires JS for tab switching, ARIA roles |
| Accordion | Collapsible panels | Requires JS for expand/collapse, ARIA |
| Carousel | Rotating content panels | Requires JS for slide navigation, pause controls |

**Layout Container and the Responsive Grid:**

The Layout Container generates CSS grid classes that frontend must handle:

```css
/* AEM responsive grid classes -- loaded from /libs/clientlibs/granite/jquery/ui */
.aem-Grid { display: flex; flex-wrap: wrap; }
.aem-GridColumn--default--12 { width: 100%; }
.aem-GridColumn--default--6 { width: 50%; }
.aem-GridColumn--default--4 { width: 33.33%; }

/* Breakpoint-specific (configured in template policy) */
@media (max-width: 768px) {
    .aem-GridColumn--phone--12 { width: 100%; }
}
```

---

### 4. Experience Fragments and Content Fragments in Frontend

#### Experience Fragments

Pre-authored page sections (headers, footers, modals) that render as standard HTML. Frontend treats them like any other component output.

```html
<!-- Including an Experience Fragment in HTL -->
<sly data-sly-resource="${'experiencefragment' @ resourceType='cq/experience-fragments/editor/components/experiencefragment'}" />

<!-- The output is standard HTML that your CSS targets normally -->
<div class="xf-content-height">
    <div class="cmp-experiencefragment cmp-experiencefragment--footer">
        <!-- Fragment content renders here -->
    </div>
</div>
```

**Frontend tip:** Experience Fragment content is server-rendered and included inline. No special JavaScript handling is needed. Style them using the standard BEM classes.

#### Content Fragments

Structured, headless content that can be consumed via:

1. **Content Fragment Component** -- renders in traditional AEM pages
2. **GraphQL API** -- for headless/SPA consumption
3. **Assets HTTP API** -- REST-based access

```javascript
// Fetching Content Fragment data via GraphQL (headless pattern)
const query = `{
    articleByPath(_path: "/content/dam/mysite/articles/intro") {
        item {
            title
            body { html }
            heroImage { ... on ImageRef { _path width height } }
        }
    }
}`;

const response = await fetch(
    'https://publish-p12345-e67890.adobeaemcloud.com/graphql/execute.json/mysite/article',
    { headers: { 'Content-Type': 'application/json' } }
);
```

---

### 5. Component Versioning and Migration

Core Components use semantic versioning: `v1`, `v2`, `v3`. Major versions indicate breaking changes in markup, dialog, or data model.

**Version upgrade process (via proxy pattern):**

```xml
<!-- Step 1: Current proxy pointing to v2 -->
<jcr:root
    sling:resourceSuperType="core/wcm/components/image/v2/image"
    componentGroup="My Project - Content"/>

<!-- Step 2: Update proxy to point to v3 -->
<jcr:root
    sling:resourceSuperType="core/wcm/components/image/v3/image"
    componentGroup="My Project - Content"/>
```

**Frontend impact of version upgrades:**

| Change Type | Frontend Action Required |
|-------------|------------------------|
| New CSS class names | Update selectors in SCSS |
| Changed HTML structure | Update JS selectors and CSS |
| New data attributes | Update JS component initialization |
| New dialog fields | Usually no frontend change needed |

**Correct -- check Core Components changelog before upgrading:**
```
# Review release notes at:
# https://github.com/adobe/aem-core-wcm-components/releases
# Each release documents markup changes per component
```

**Incorrect -- upgrading version without checking markup changes:**
```css
/* Your CSS breaks silently because v3 changed cmp-image__image to cmp-image__asset */
.cmp-image__image { object-fit: cover; }  /* Works in v2, broken in v3 */
```

---

### 6. Embed Component and oEmbed

The Embed Core Component supports three modes: URL (oEmbed), Embeddable, and HTML.

```html
<!-- Embed component output for YouTube oEmbed -->
<div class="cmp-embed" data-cmp-is="embed">
    <div class="cmp-embed__wrapper">
        <iframe src="https://www.youtube.com/embed/..."
                allow="autoplay; encrypted-media"
                allowfullscreen></iframe>
    </div>
</div>
```

**Frontend considerations:**
```css
/* Make embeds responsive */
.cmp-embed__wrapper {
    position: relative;
    padding-bottom: 56.25%; /* 16:9 aspect ratio */
    height: 0;
    overflow: hidden;
}

.cmp-embed__wrapper iframe,
.cmp-embed__wrapper video {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}
```

**Security tip:** The Embed component's HTML mode allows arbitrary markup. Frontend should sanitize or scope styles to prevent CSS leaks:

```css
/* Scope embed styles to prevent global pollution */
.cmp-embed [data-cmp-embed-type="html"] {
    all: initial; /* Reset inherited styles */
}
```

---

### 7. Dynamic Components (Computed resourceType)

Use `data-sly-resource` with dynamically computed resource types for flexible component rendering:

```html
<!-- Dynamic resource type based on content property -->
<sly data-sly-use.resolver="com.myproject.core.models.ComponentResolver">
    <sly data-sly-resource="${'content' @ resourceType=resolver.resourceType}" />
</sly>

<!-- Conditional component rendering based on content type -->
<sly data-sly-test.isVideo="${properties.mediaType == 'video'}">
    <sly data-sly-resource="${'media' @ resourceType='myproject/components/video'}" />
</sly>
<sly data-sly-test="${!isVideo}">
    <sly data-sly-resource="${'media' @ resourceType='myproject/components/image'}" />
</sly>
```

---

### 8. Component Placeholder and Empty State Patterns

Authors need visual feedback when a component has no content. AEM provides built-in placeholder handling.

```html
<!-- HTL placeholder pattern using Core Component conventions -->
<sly data-sly-use.title="com.adobe.cq.wcm.core.components.models.Title">
    <div class="cmp-title"
         data-cmp-is="title"
         data-sly-test="${title.text}">
        <h2 class="cmp-title__text">${title.text}</h2>
    </div>
    <!-- Show placeholder only in edit/preview mode when empty -->
    <sly data-sly-test="${!title.text && wcmmode.edit}">
        <div class="cq-placeholder" data-emptytext="Title"></div>
    </sly>
</sly>
```

**Correct -- placeholder visible only in edit mode:**
```html
<sly data-sly-test="${wcmmode.edit && !model.hasContent}">
    <div class="cq-placeholder" data-emptytext="${component.title}"></div>
</sly>
```

**Incorrect -- placeholder visible on publish:**
```html
<!-- This renders an empty div on publish, causing layout issues -->
<div class="cq-placeholder" data-emptytext="Click to configure"></div>
```

**Frontend CSS for placeholders:**
```css
/* Style placeholders only in authoring context */
.cq-placeholder {
    padding: 1rem;
    background: rgba(0, 0, 0, 0.04);
    border: 1px dashed #ccc;
    text-align: center;
    color: #999;
    font-style: italic;
}

/* Hide on publish (defense-in-depth) */
body:not(.aem-AuthorLayer-Edit) .cq-placeholder {
    display: none;
}
```

---

### 9. Decoration Tag Customization

AEM wraps components in a decoration `<div>` for editor functionality. You can control this behavior.

#### Default Behavior

```html
<!-- data-sly-resource without options: NO wrapper div -->
<sly data-sly-resource="child"></sly>
<!-- Output: <p>Hello World</p> -->

<!-- Editable component: wrapper div added automatically -->
<!-- Output: <div class="component-class">Hello World</div> -->
```

#### Customizing the Decoration Tag

**Via HTL attributes:**
```html
<!-- Custom element and CSS class -->
<sly data-sly-resource="${'child' @ decorationTagName='article', cssClassName='my-component'}" />
<!-- Output: <article class="my-component">Hello World</article> -->

<!-- Force decoration on -->
<sly data-sly-resource="${'child' @ decoration=true}" />

<!-- Force decoration off (use cautiously -- breaks editing) -->
<sly data-sly-resource="${'child' @ decoration=false}" />
```

**Via component node properties:**
```xml
<!-- /apps/myproject/components/card/.content.xml -->
<jcr:root
    jcr:primaryType="cq:Component"
    jcr:title="Card"
    sling:resourceSuperType="core/wcm/components/teaser/v2/teaser">

    <!-- Custom decoration tag configuration -->
    <cq:htmlTag
        jcr:primaryType="nt:unstructured"
        cq:tagName="article"
        class="card-wrapper"/>
</jcr:root>
```

**Correct -- consistent wrapper across all modes:**
```
The wrapper element should NOT differ between WCM modes (edit, preview, publish).
CSS and JavaScript must work identically regardless of the mode.
```

**Incorrect -- mode-dependent wrapper:**
```html
<!-- DO NOT conditionally change the wrapper tag based on wcmmode -->
<!-- This causes CSS/JS to break in different modes -->
```

#### Suppressing Decoration

Use `cq:noDecoration` when a component should render without any wrapper:

```xml
<jcr:root
    jcr:primaryType="cq:Component"
    cq:noDecoration="{Boolean}true"/>
```

**Warning:** Components without decoration tags cannot be edited inline in the page editor. Only use this for non-editable included components.

---

### 10. WCM Mode Detection in Frontend

AEM runs in different modes that affect rendering and behavior.

| Mode | Context | `wcmmode` Value |
|------|---------|----------------|
| Edit | Page editor, component editing | `edit` |
| Preview | Author preview | `preview` |
| Design | Template editor (deprecated in Touch UI) | `design` |
| Disabled | Publish instance, `?wcmmode=disabled` | `disabled` / not set |

#### HTL Mode Detection

```html
<!-- Show content only in edit mode -->
<div data-sly-test="${wcmmode.edit}" class="author-hint">
    <p>Author: Remember to set the hero image.</p>
</div>

<!-- Hide authoring UI elements on publish -->
<sly data-sly-test="${wcmmode.disabled}">
    <script src="/path/to/analytics.js" async></script>
</sly>

<!-- Common pattern: edit OR preview -->
<sly data-sly-test="${wcmmode.edit || wcmmode.preview}">
    <div class="component-placeholder">Configure this component</div>
</sly>
```

#### JavaScript Mode Detection

```javascript
// Client-side WCM mode detection
const WCMMode = {
    get current() {
        // URL parameter takes precedence
        const param = new URLSearchParams(window.location.search).get('wcmmode');
        if (param) return param;

        // Cookie set by AEM editor (values: 'edit' or 'preview')
        const match = document.cookie.match(/wcmmode=([^;]+)/);
        return match ? match[1] : 'disabled';
    },

    get isAuthor() {
        return this.current === 'edit' || this.current === 'preview';
    },

    get isPublish() {
        return this.current === 'disabled';
    },

    get isEdit() {
        return this.current === 'edit';
    }
};

// Usage: guard third-party scripts from author mode
if (WCMMode.isPublish) {
    loadGTM();
    loadChatWidget();
}

// Usage: enhance components only on publish
if (WCMMode.isPublish) {
    initCarousels();   // Auto-play on publish
    initLazyLoading(); // Lazy load on publish
}
```

---

### 11. Component Includes and Nesting Depth Limits

AEM has a default Sling include depth limit of **15 levels** (`SlingMainServlet` recursion limit). Deeply nested components will silently stop rendering.

**Correct -- flat component architecture:**
```
Page
  └── Container (depth 1)
       ├── Hero (depth 2)
       ├── Tabs (depth 2)
       │    ├── Tab Panel (depth 3)
       │    │    └── Text (depth 4)
       │    └── Tab Panel (depth 3)
       └── Footer (depth 2)
```

**Incorrect -- excessive nesting:**
```
Page > Container > Accordion > Tab > Container > Accordion > Container > ...
<!-- At depth 15+, components silently fail to render -->
<!-- No error in logs, content just disappears -->
```

**Pro tip:** If you need deeply nested structures, consider flattening with Experience Fragments or restructuring the component model.

---

### 12. Page Component Structure

The page component is the foundation of every AEM page. Understanding its HTL structure is essential for frontend.

```
/apps/myproject/components/page/
├── .content.xml                 # sling:resourceSuperType="core/wcm/components/page/v3/page"
├── customheaderlibs.html        # Injected into <head> -- CSS, preloads, meta tags
├── customfooterlibs.html        # Injected before </body> -- JS, analytics
└── body.html                    # (optional) Override body structure
```

**customheaderlibs.html -- CSS, fonts, and critical resources:**

```html
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html">
    <!-- Load site CSS in <head> -->
    <sly data-sly-call="${clientlib.css @ categories='myproject.base'}" />
</sly>

<!-- Preload critical fonts -->
<link rel="preload" href="/etc.clientlibs/myproject/clientlibs/clientlib-base/resources/fonts/brand.woff2"
      as="font" type="font/woff2" crossorigin="anonymous" />

<!-- Preconnect to third-party origins -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="dns-prefetch" href="https://www.googletagmanager.com" />
```

**customfooterlibs.html -- JavaScript and deferred resources:**

```html
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html">
    <!-- Load site JS before </body> -->
    <sly data-sly-call="${clientlib.js @ categories='myproject.base'}" />
</sly>

<!-- Deferred third-party scripts -->
<sly data-sly-test="${wcmmode.disabled}">
    <script async src="https://www.googletagmanager.com/gtm.js?id=GTM-XXXX"></script>
</sly>
```

**Correct -- CSS in header, JS in footer:**
```html
<!-- customheaderlibs.html: CSS only (render-blocking is acceptable for CSS) -->
<sly data-sly-call="${clientlib.css @ categories='myproject.base'}" />

<!-- customfooterlibs.html: JS only (non-blocking) -->
<sly data-sly-call="${clientlib.js @ categories='myproject.base'}" />
```

**Incorrect -- loading everything in header:**
```html
<!-- customheaderlibs.html -->
<sly data-sly-call="${clientlib.all @ categories='myproject.base'}" />
<!-- This loads JS in <head>, blocking page rendering -->
```

---

### 13. Error Handling in Components

Robust frontend components must handle missing or malformed content gracefully.

**HTL defensive patterns:**

```html
<!-- Guard against null model -->
<sly data-sly-use.card="com.myproject.core.models.Card"
     data-sly-test.hasCard="${card.ready}">

    <div class="cmp-card">
        <!-- Guard against missing image -->
        <sly data-sly-test="${card.imageSrc}">
            <img class="cmp-card__image"
                 src="${card.imageSrc}"
                 alt="${card.imageAlt || card.title}"
                 loading="lazy" />
        </sly>

        <!-- Guard against empty title with fallback -->
        <h3 class="cmp-card__title">
            ${card.title || 'Untitled'}
        </h3>

        <!-- Guard against missing link -->
        <sly data-sly-test="${card.linkURL}">
            <a class="cmp-card__link" href="${card.linkURL}">
                ${card.linkText || 'Read More'}
            </a>
        </sly>
    </div>
</sly>

<!-- Fallback for edit mode when model fails -->
<sly data-sly-test="${!hasCard && wcmmode.edit}">
    <div class="cq-placeholder" data-emptytext="Card - please configure"></div>
</sly>
```

**JavaScript error handling for component initialization:**

```javascript
// Defensive component initialization
document.querySelectorAll('[data-cmp-is="carousel"]').forEach(element => {
    try {
        const config = JSON.parse(element.dataset.cmpConfig || '{}');
        new Carousel(element, config);
    } catch (error) {
        console.error('Carousel initialization failed:', error, element);
        // Don't break other components -- continue the loop
    }
});
```

**Pro tip:** Never let one broken component crash the entire page's JavaScript. Always wrap component initialization in try/catch blocks and use `querySelectorAll` loops instead of assuming a single instance.
