---
title: AEM Accessibility (WCAG 2.1 AA) and SEO Patterns
impact: HIGH
impactDescription: Accessibility failures expose legal risk and exclude users — SEO misconfigurations reduce organic traffic and search visibility
tags: accessibility, wcag, aria, a11y, seo, structured-data, json-ld, sitemap, meta-tags, core-web-vitals, hreflang, canonical
---

## AEM Accessibility and SEO Patterns

AEM Core Components are built with WCAG 2.1 AA compliance in mind, but correct usage and custom component development require deliberate accessibility and SEO practices.

---

### 1. WCAG 2.1 AA Compliance in AEM Components

AEM Core Components provide semantic HTML, ARIA attributes, keyboard navigation, and screen reader support out of the box. Custom components must meet the same standard.

**Key WCAG 2.1 AA requirements for AEM developers:**

| Criterion | Requirement | AEM Implementation |
|-----------|-------------|--------------------|
| 1.1.1 Non-text Content | Alt text for images | Image component `alt` field; enforce via dialog validation |
| 1.3.1 Info and Relationships | Semantic structure | Use heading levels, lists, tables correctly in HTL |
| 2.1.1 Keyboard | All functionality via keyboard | Test Tab/Enter/Escape on custom JS components |
| 2.4.1 Bypass Blocks | Skip navigation mechanism | Add skip link in page component header |
| 2.4.6 Headings and Labels | Descriptive headings | Heading hierarchy enforced by template policies |
| 4.1.2 Name, Role, Value | Programmatic name for UI controls | ARIA attributes on custom interactive elements |

---

### 2. ARIA Roles and Landmarks in HTL Templates

**Correct -- semantic landmarks in the page component HTL:**

```html
<!-- page body.html -->
<header role="banner" class="cmp-page__header">
    <a href="#main-content" class="cmp-page__skip-link">Skip to main content</a>
    <sly data-sly-resource="${'header' @ resourceType='myproject/components/header'}" />
</header>

<nav role="navigation" aria-label="Main Navigation" class="cmp-page__nav">
    <sly data-sly-resource="${'navigation' @ resourceType='core/wcm/components/navigation/v2/navigation'}" />
</nav>

<main id="main-content" role="main" class="cmp-page__main">
    <sly data-sly-resource="${'root' @ resourceType='wcm/foundation/components/responsivegrid'}" />
</main>

<footer role="contentinfo" class="cmp-page__footer">
    <sly data-sly-resource="${'footer' @ resourceType='myproject/components/footer'}" />
</footer>
```

**Incorrect -- missing landmarks and skip navigation:**

```html
<!-- No skip link, no landmark roles, no aria-labels -->
<div class="header">
    <sly data-sly-resource="${'header' @ resourceType='myproject/components/header'}" />
</div>
<div class="content">
    <sly data-sly-resource="${'root' @ resourceType='wcm/foundation/components/responsivegrid'}" />
</div>
<div class="footer">
    <sly data-sly-resource="${'footer' @ resourceType='myproject/components/footer'}" />
</div>
```

**Skip link CSS:**

```css
.cmp-page__skip-link {
    position: absolute;
    left: -9999px;
    top: auto;
    width: 1px;
    height: 1px;
    overflow: hidden;
}

.cmp-page__skip-link:focus {
    position: fixed;
    top: 0;
    left: 0;
    width: auto;
    height: auto;
    padding: 0.5rem 1rem;
    background: #000;
    color: #fff;
    z-index: 10000;
}
```

---

### 3. Accessible Dialog Authoring

Every dialog field must have `fieldLabel` and, where appropriate, `fieldDescription` to ensure screen reader users can author content effectively.

**Correct -- accessible dialog fields:**

```xml
<image
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/include"
    path="core/wcm/components/image/v3/image/cq:dialog/content/items/tabs/items/asset"/>

<alt
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="Alternative Text"
    fieldDescription="Describe the image for screen readers. Leave empty only for decorative images."
    name="./alt"
    required="{Boolean}true"/>

<isDecorative
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
    fieldDescription="Check if the image is purely decorative and should be hidden from assistive technology"
    name="./isDecorative"
    text="Image is decorative"
    value="{Boolean}true"/>
```

**Incorrect -- missing labels and descriptions:**

```xml
<!-- Screen readers cannot identify the purpose of these fields -->
<alt
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    name="./alt"/>

<heading
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    name="./headingLevel">
    <!-- Missing fieldLabel — author has no idea what this select controls -->
</heading>
```

---

### 4. Image Alt Text Enforcement

Enforce alt text through dialog validation and Sling Model logic.

**Sling Model alt text handling:**

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = CustomImage.class,
       resourceType = "myproject/components/image")
public class CustomImageImpl implements CustomImage {

    @ValueMapValue(injectionStrategy = InjectionStrategy.OPTIONAL)
    private String alt;

    @ValueMapValue(injectionStrategy = InjectionStrategy.OPTIONAL)
    private boolean isDecorative;

    @Override
    public String getAlt() {
        return alt;
    }

    @Override
    public boolean isDecorative() {
        return isDecorative;
    }
}
```

**Correct HTL rendering:**

```html
<sly data-sly-use.image="com.myproject.models.CustomImage">
    <img class="cmp-image__image"
         src="${image.src}"
         data-sly-test="${!image.decorative}"
         alt="${image.alt}"
         loading="lazy"
         width="${image.width}"
         height="${image.height}" />
    <!-- Decorative image: empty alt, role="presentation" -->
    <img class="cmp-image__image"
         src="${image.src}"
         data-sly-test="${image.decorative}"
         alt=""
         role="presentation"
         loading="lazy" />
</sly>
```

---

### 5. Heading Hierarchy Management

Prevent heading level skips across components using template policies and the Title component's heading level selector.

**Template policy configuration (restrict available heading levels):**

```xml
<!-- /conf/myproject/settings/wcm/policies/myproject/components/title/policy_main_content -->
<jcr:root jcr:primaryType="nt:unstructured"
    jcr:title="Main Content Title Policy"
    sling:resourceType="wcm/core/components/policy/policy"
    type="h2"
    allowedTypes="[h2,h3,h4]"/>
<!-- Restrict to h2-h4 in main content area; h1 reserved for page title -->
```

**Correct -- one h1 per page, sequential hierarchy:**

```html
<h1 class="cmp-title__text">Page Title</h1>           <!-- One per page -->
  <h2 class="cmp-title__text">Section Title</h2>       <!-- Direct child -->
    <h3 class="cmp-title__text">Subsection Title</h3>  <!-- No level skipped -->
```

**Incorrect -- skipped heading levels:**

```html
<h1>Page Title</h1>
  <h3>Subsection</h3>  <!-- Skipped h2 — violates WCAG 1.3.1 -->
    <h6>Detail</h6>    <!-- Skipped h4, h5 — confusing for screen readers -->
```

---

### 6. Focus Management in Custom Components

Interactive components (modals, accordions, tabs) must manage keyboard focus.

```javascript
// Modal focus trap pattern
function trapFocus(modal) {
    const focusableSelectors = 'a[href], button:not([disabled]), input, select, textarea, [tabindex]:not([tabindex="-1"])';
    const focusableElements = modal.querySelectorAll(focusableSelectors);
    const firstFocusable = focusableElements[0];
    const lastFocusable = focusableElements[focusableElements.length - 1];

    // Move focus into the modal
    firstFocusable.focus();

    modal.addEventListener('keydown', function(e) {
        if (e.key === 'Tab') {
            if (e.shiftKey && document.activeElement === firstFocusable) {
                e.preventDefault();
                lastFocusable.focus();
            } else if (!e.shiftKey && document.activeElement === lastFocusable) {
                e.preventDefault();
                firstFocusable.focus();
            }
        }
        if (e.key === 'Escape') {
            closeModal(modal);
        }
    });
}
```

---

### 7. SEO: Page Properties for Meta Tags

AEM page properties map to critical SEO meta tags via the page component HTL.

**Correct -- page component head section with full SEO tags:**

```html
<!-- customheaderlibs.html or head.html -->
<sly data-sly-use.page="com.adobe.cq.wcm.core.components.models.Page">

    <!-- Page title: Use pageTitle, fall back to jcr:title -->
    <title>${page.title || currentPage.title}</title>

    <!-- Meta description -->
    <meta name="description"
          content="${currentPage.description}"
          data-sly-test="${currentPage.description}" />

    <!-- Canonical URL -->
    <link rel="canonical"
          href="${page.canonicalLink || currentPage.path @ extension='html', scheme='https'}" />

    <!-- Open Graph tags -->
    <meta property="og:title" content="${currentPage.properties['og:title'] || currentPage.title}" />
    <meta property="og:description" content="${currentPage.properties['og:description'] || currentPage.description}" />
    <meta property="og:image"
          content="${currentPage.properties['og:image']}"
          data-sly-test="${currentPage.properties['og:image']}" />
    <meta property="og:type" content="website" />
    <meta property="og:url" content="${page.canonicalLink}" />

    <!-- Robots directive -->
    <meta name="robots"
          content="${currentPage.properties.robotsTag || 'index, follow'}" />
</sly>
```

**Incorrect -- hardcoded or missing SEO tags:**

```html
<!-- Missing canonical, og tags, and robots directives -->
<title>My Website</title>
<!-- Hardcoded title — ignores page properties authored by content editors -->
```

---

### 8. Structured Data / JSON-LD

Inject structured data via HTL and Sling Models for rich search results.

**Correct -- JSON-LD in the page component:**

```html
<sly data-sly-use.schema="com.myproject.models.SchemaModel">
    <script type="application/ld+json"
            data-sly-test="${schema.jsonLd}">
        ${schema.jsonLd @ context='unsafe'}
    </script>
</sly>
```

**Sling Model that generates safe JSON-LD:**

```java
@Model(adaptables = SlingHttpServletRequest.class)
public class SchemaModel {

    @ScriptVariable
    private Page currentPage;

    @OSGiService
    private Externalizer externalizer;

    public String getJsonLd() {
        JsonObject schema = new JsonObject();
        schema.addProperty("@context", "https://schema.org");
        schema.addProperty("@type", "WebPage");
        schema.addProperty("name", currentPage.getTitle());
        schema.addProperty("description", currentPage.getDescription());
        schema.addProperty("url", externalizer.publishLink(
            currentPage.getPath() + ".html"));

        // Breadcrumb structured data
        // Organization structured data
        // Article structured data (for blog pages)

        return schema.toString(); // Gson handles JSON escaping
    }
}
```

**Important:** Using `context='unsafe'` is acceptable here only because the JSON is built entirely server-side with proper escaping via Gson. Never pass author-entered HTML through `context='unsafe'`.

---

### 9. AEM Built-in Sitemap

AEM Cloud Service provides automatic XML sitemap generation.

**Enable sitemap in page properties:**

1. Navigate to page properties > Advanced tab
2. Enable "Sitemap" generation
3. AEM generates `/content/mysite/en.sitemap.xml` automatically

**OSGi configuration for sitemap scheduler:**

```json
{
    "scheduler.expression": "0 0 2 * * ?",
    "include": ["/content/mysite/en"],
    "exclude": ["/content/mysite/en/internal"],
    "changefreq.default": "weekly",
    "priority.default": "0.5"
}
```

**Dispatcher rule to expose sitemap at domain root:**

```
# Rewrite /sitemap.xml to the AEM-generated path
RewriteRule ^/sitemap\.xml$ /content/mysite/en.sitemap.xml [PT,L]
```

---

### 10. Hreflang for Multi-Language Sites

**Correct -- hreflang link elements via the page component:**

```html
<sly data-sly-use.languageNav="com.adobe.cq.wcm.core.components.models.LanguageNavigation"
     data-sly-list.lang="${languageNav.items}">
    <link rel="alternate"
          hreflang="${lang.language}"
          href="${lang.url @ extension='html', scheme='https'}" />
</sly>
<link rel="alternate" hreflang="x-default"
      href="${defaultLanguagePage @ extension='html', scheme='https'}" />
```

**Incorrect -- missing x-default or inconsistent URLs:**

```html
<!-- Missing x-default fallback -->
<link rel="alternate" hreflang="en" href="https://www.example.com/en/page.html" />
<link rel="alternate" hreflang="de" href="https://www.example.com/de/page.html" />
<!-- No x-default — search engines don't know which is the fallback -->
```

---

### 11. Robots.txt and Dispatcher Redirects

**Robots.txt via Dispatcher:**

```
# dispatcher/src/conf.d/rewrites/rewrite.rules
RewriteRule ^/robots\.txt$ /content/dam/mysite/robots.txt [PT,L]
```

**SEO-friendly redirects in Dispatcher (301 for permanent moves):**

```
# Redirect old URLs to preserve link equity
RewriteRule ^/old-page\.html$ /content/mysite/en/new-page.html [R=301,L]

# Trailing slash normalization
RewriteRule ^(.+)/$ $1 [R=301,L]

# Force lowercase URLs
RewriteMap lowercase int:tolower
RewriteRule ^(.*)$ ${lowercase:$1} [R=301,L]
```

---

### 12. Accessibility Testing in AEM

**Automated testing tools:**

| Tool | Purpose |
|------|---------|
| aXe DevTools | In-browser WCAG 2.1 AA audit |
| Lighthouse Accessibility | Score and actionable recommendations |
| WAVE | Visual feedback on accessibility errors |
| pa11y CI | Automated a11y testing in CI/CD pipeline |
| AEM Content Analyzer | Built-in AEM content quality checks |

**pa11y integration in the frontend pipeline:**

```json
{
    "scripts": {
        "test:a11y": "pa11y-ci --config .pa11yci.json"
    }
}
```

```json
{
    "defaults": {
        "standard": "WCAG2AA",
        "timeout": 30000
    },
    "urls": [
        "http://localhost:4502/content/mysite/en.html?wcmmode=disabled",
        "http://localhost:4502/content/mysite/en/contact.html?wcmmode=disabled"
    ]
}
```

---

### Pitfalls

- **Never skip alt text** on meaningful images -- enforce via required dialog fields
- **Never skip heading levels** (h1 > h3) -- use template policies to restrict available levels
- **Never omit landmark roles** on page regions -- screen readers depend on them
- **Never hardcode page titles** in HTL -- use page properties for SEO flexibility
- **Never omit canonical URLs** -- causes duplicate content penalties
- **Never use `context='unsafe'`** with author-entered content for JSON-LD -- build JSON server-side
- **Always include hreflang x-default** on multi-language sites
- **Test with keyboard only** -- if you cannot Tab to it, it fails WCAG 2.1.1
