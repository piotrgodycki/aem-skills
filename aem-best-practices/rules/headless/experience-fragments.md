---
title: Experience Fragments — Reusable Composed Experiences
impact: HIGH
impactDescription: Experience Fragments enable reusable, layout-complete content sections across pages and channels; incorrect use leads to performance degradation and editorial complexity
tags: experience-fragments, xf, personalization, target, live-copy, core-components, caching, sdi, reuse
---

## Experience Fragments — Reusable Composed Experiences

Experience Fragments (XF) are groups of one or more AEM components — **including layout** — that form a self-contained, reusable experience. Unlike Content Fragments (structured data, no presentation), XFs carry both content and visual design and are rendered server-side.

---

### XF vs Content Fragments: When to Use Which

| Criterion | Experience Fragment | Content Fragment |
|-----------|-------------------|-----------------|
| **Content + Layout** | Yes — components in a parsys | No — structured data only |
| **Channel-neutral data** | No — tied to HTML rendering | Yes — JSON / GraphQL delivery |
| **Authoring paradigm** | Page Editor (drag & drop) | Form-based CF Editor |
| **Reuse pattern** | Same rendered block on many pages | Same data rendered differently per channel |
| **Target integration** | Export as HTML/JSON offer | Export as JSON offer |
| **Typical use cases** | Header, footer, promo banner, nav | Article body, product specs, FAQ, author bio |

**Rule of thumb**: if the consumer needs **data** (SPA, mobile app, third-party system), use a Content Fragment. If the consumer needs a **ready-to-render HTML block** reused across AEM pages or exported to Target, use an Experience Fragment.

---

### XF Architecture

Experience Fragments are stored under `/content/experience-fragments/<project>/` and based on **editable templates only**.

```
/content/experience-fragments/mysite/en/
  promo-banner/
    jcr:content
      sling:resourceType = cq/experience-fragments/components/xfpage
    master/                    ← root variation (always exists)
      jcr:content/
        root/
          responsivegrid/
            teaser_1           ← components with layout
            text_1
    web-variation/             ← named variation
    email-variation/           ← variation with email-safe markup
    social-variation/          ← social media card
```

#### Template Requirements

Templates must satisfy one of:
1. `sling:resourceType` inherits from `cq/experience-fragments/components/xfpage` **and** template name begins with `experience-fragments`.
2. Allowed templates configured on the XF folder via folder properties regex:
   ```
   /conf/(.*)/settings/wcm/templates/experience-fragment(.*)?
   ```
   This approach survives upgrades and is recommended.

---

### Building Blocks

Building Blocks are reusable component groupings inside an Experience Fragment.

1. Select multiple components in the editor.
2. Choose **Convert to building block** from the toolbar.
3. The block is saved and available in the same fragment or across all fragments (filter by **Local** or **All**).

Building blocks allow consistent sub-sections (e.g., a "price card" used in multiple promo XF variations) without duplicating component configuration.

---

### Variations

Every XF has a **master** variation. Additional variations are created for different channels or layouts.

#### Variation Types

| Type | Description | Rendering |
|------|-------------|-----------|
| **Web** (default) | Standard AEM page rendering | Full HTML with CSS/JS |
| **Plain HTML** | Stripped-down markup via `.plain.html` selector | Minimal HTML for third-party embedding |
| **Email** | Email-safe HTML (inline styles, table layout) | Compatible with ESP rendering engines |
| **Social** | Open Graph / social card format | Extracts image + text for Facebook, Pinterest |
| **Custom** | Project-specific templates | As defined by template |

#### Creating Variations

```
Experience Fragment Console → select XF → Create → Variation
  Title:     "Email — Holiday Promo"
  Template:  experience-fragment-email-template
```

Variations may **share content and components** with the master or diverge entirely.

#### Live-Copy Variations

A variation can be created as a **live copy** of the master. Changes to master propagate automatically (MSM inheritance). Break inheritance on individual components when a variation needs to diverge.

---

### Rendering XFs on Pages

#### Using data-sly-resource (HTL include)

```html
<!-- Include the master variation of an XF -->
<sly data-sly-resource="${'experience-fragment-path'
    @ resourceType='cq/experience-fragments/components/xfpage'}" />
```

#### Using the Experience Fragment Core Component

The recommended approach uses the **Experience Fragment component** (Core Component):

- **Resource type**: `core/wcm/components/experiencefragment/v2/experiencefragment`
- Author selects the XF path and variation in the component dialog.
- The component resolves the correct language-copy XF path when used with localizedRoot (see language copies below).

```xml
<!-- Component node in page content -->
<experiencefragment
    jcr:primaryType="nt:unstructured"
    sling:resourceType="core/wcm/components/experiencefragment/v2/experiencefragment"
    fragmentVariationPath="/content/experience-fragments/mysite/en/header/master" />
```

#### Localization-Aware Resolution

The XF Core Component supports automatic locale resolution. Given:

```
fragmentVariationPath = /content/experience-fragments/mysite/en/header/master
current page          = /content/mysite/de/products
```

The component resolves to `/content/experience-fragments/mysite/de/header/master` automatically, falling back to the configured path if the localized version does not exist.

---

### Live Copy and Language Copies

Experience Fragments integrate with AEM Multi-Site Manager (MSM) and language copy workflows:

- **Language copies**: create XFs under locale folders (`/en/`, `/de/`, `/fr/`). The XF Core Component auto-resolves the correct locale.
- **Live copies**: XF variations can inherit from master via MSM. Cancel inheritance per component to localize individual blocks.
- **Translation workflow**: XFs are translatable via AEM translation framework; initiate from the XF console or project.

```
/content/experience-fragments/mysite/
  en/
    header/master/
    footer/master/
  de/                           ← language copy
    header/master/              ← translated version
    footer/master/
  fr/
    header/master/
    footer/master/
```

---

### Export to Adobe Target

Experience Fragments can be exported to Adobe Target as **HTML offers** or **JSON offers** for personalization and A/B testing.

#### Prerequisites

1. Adobe Target Cloud Configuration applied to the XF folder or individual XF.
2. XF and its assets must be **published** (or use "Export without publishing" for preview).

#### Export Flow

1. XF Console → select fragment → **Export to Adobe Target**.
2. Choose: **Export without publishing** or **Publish** (publishes + exports).
3. The XF appears as an **offer** in Adobe Target.

#### Link Rewriting

During export, internal links are rewritten to use the publish domain via the `publishLink()` method. For advanced cases (Sling Mappings, Dispatcher redirects, `sling:alias`), implement a **Custom Link Rewriter Provider**:

```java
@Component(service = ExperienceFragmentLinkRewriterProvider.class)
public class MyLinkRewriterProvider implements ExperienceFragmentLinkRewriterProvider {

    @Override
    public String rewriteLink(String link, String tag, String attribute) {
        // Rewrite internal links to external CDN URLs
        if (link.startsWith("/content/dam/")) {
            return "https://cdn.example.com" + link;
        }
        return null; // null = use default rewriting
    }

    @Override
    public boolean shouldRewrite(ExperienceFragmentVariation variation) {
        return true;
    }

    @Override
    public int getPriority() {
        return 100; // higher value = higher precedence
    }
}
```

#### Plain HTML Selector

The `.plain.html` selector renders a stripped-down HTML version of the XF, suitable for third-party systems. The Sling Rewriter pipeline at `/libs/experience-fragments/config/rewriter/experiencefragments` controls:

- `allowedCssClasses`: regex of CSS classes to retain.
- `allowedTags`: whitelist of HTML tags (defaults: `html`, `head`, `body`, `div`, `p`, `span`, `img`, `link`, `script`, headings).

---

### Frontend Patterns

#### Header and Footer as XF

The most common XF pattern: shared header/footer across all pages.

```
/content/experience-fragments/mysite/en/
  site-header/
    master/                    ← global navigation, logo, search
  site-footer/
    master/                    ← footer links, copyright, social icons
```

Include via the XF Core Component in the page template's **structure** (locked) container:

```xml
<!-- Editable template structure -->
<header jcr:primaryType="nt:unstructured"
    sling:resourceType="core/wcm/components/experiencefragment/v2/experiencefragment"
    fragmentVariationPath="/content/experience-fragments/mysite/en/site-header/master" />

<!-- Page content (editable by authors) -->
<root jcr:primaryType="nt:unstructured"
    sling:resourceType="wcm/foundation/components/responsivegrid" />

<footer jcr:primaryType="nt:unstructured"
    sling:resourceType="core/wcm/components/experiencefragment/v2/experiencefragment"
    fragmentVariationPath="/content/experience-fragments/mysite/en/site-footer/master" />
```

#### Promo Banners

Create time-limited promotional XFs and insert them via the XF component. Use Target integration for personalized banner selection.

#### Shared Navigation

Multi-level navigation authored as an XF. Use the Navigation Core Component **inside** the XF for automatic page-tree-based menu generation.

---

### Style Variations and CSS Scoping

XF content is rendered inline on the host page, so CSS from the page and the XF share the same DOM.

#### CSS Scoping Strategies

1. **BEM namespacing**: prefix all XF classes with a unique namespace.
   ```css
   .xf-promo-banner { ... }
   .xf-promo-banner__title { ... }
   .xf-promo-banner__cta { ... }
   ```

2. **Style System**: apply style classes via the AEM Style System on the XF component itself to toggle visual variants without separate XF variations.

3. **Container scoping**: wrap XF output in a scoped container class.
   ```html
   <div class="xf-scope xf-scope--header">
     <!-- XF content renders here -->
   </div>
   ```

4. **Avoid `!important`**: XF styles should not need `!important`; if they do, the CSS architecture needs refactoring.

---

### Caching Considerations

#### Dispatcher Caching

XFs rendered on pages are part of the page HTML and cached with the page. When the XF changes, **all pages including it must be invalidated**.

#### Sling Dynamic Include (SDI)

To cache XF sections independently from the page, use SDI to replace the XF include with an SSI/ESI directive:

```
# OSGi Configuration — org.apache.sling.dynamicinclude.Configuration
include-filter.config.enabled=true
include-filter.config.path=/content
include-filter.config.resource-types=core/wcm/components/experiencefragment/v2/experiencefragment
include-filter.config.include-type=SSI
include-filter.config.selector=nocache
include-filter.config.ttl=300
include-filter.config.required_header=Server-Agent=Communique-Dispatcher
```

Apache vhost:
```apache
<Directory "${DOCROOT}">
    Options FollowSymLinks Includes
    AddOutputFilter INCLUDES .html
</Directory>
```

Dispatcher `dispatcher.any`:
```
/rules {
    /0001 { /glob "*.nocache.html" /type "deny" }
}
```

This enables:
- The page is cached as a static HTML file with an SSI include tag.
- The XF fragment is fetched separately on each request (or cached with its own TTL).
- XF changes invalidate only the XF fragment cache, not every page.

#### TTL Strategy

| Component | Recommended TTL | Rationale |
|-----------|----------------|-----------|
| Global header/footer XF | 300–600 s | Changes infrequently; short TTL for safety |
| Promo banner XF | 60–300 s | May change for time-sensitive campaigns |
| Page body | Standard Dispatcher rules | Invalidated on content publish |

> **AEM Cloud Service note**: ESI is **not** supported with the out-of-the-box Fastly CDN. Use SSI via Apache `mod_include` or JSI (JavaScript Include) as alternatives.

---

### Access Control

Write access to Experience Fragments requires membership in the `experience-fragments-editors` group. Read access follows standard AEM ACL rules.

---

### Best Practices

1. **Reuse across pages via the XF Core Component** — never copy-paste XF markup manually.
2. **One XF per concern**: separate header, footer, promo, and navigation into distinct XFs.
3. **Use language copy folders** (`/en/`, `/de/`) and let the XF component auto-resolve locale.
4. **Cross-site XF sharing**: place shared XFs under a common project root (e.g., `/content/experience-fragments/shared/`) and reference from multiple sites.
5. **Keep XFs lightweight** — avoid embedding heavy client-side libraries or complex JS inside XFs (load once at page level).
6. **Use SDI for independently cached XFs** — especially global header/footer on high-traffic sites.
7. **Publish XF before publishing pages that reference it** — avoids 404 for XF include on publish tier.
8. **Limit XF per page to 3–5** — each SDI-included XF adds an SSI sub-request; too many degrade TTFB.
9. **Use Building Blocks** for reusable sub-sections within XFs rather than duplicating components.
10. **Test Target exports** in a staging environment — link rewriting edge cases surface only after export.

### Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| **XF for everything** | Every page becomes a mosaic of SSI includes; debugging and cache invalidation become unmanageable | Use XFs only for genuinely shared content (header, footer, promos) |
| **XF with complex JS** | JS initialisation runs per-include, conflicts with page-level scripts, breaks SPA hydration | Load JS at page level; keep XFs markup-only or with minimal inline scripts |
| **Too many XFs on one page** (10+) | Each SSI/ESI sub-request adds latency; TTFB degrades linearly | Consolidate into fewer, larger XFs or inline rarely-changing content |
| **XF instead of CF for headless** | XF output is HTML, not structured data; cannot be consumed by native apps or SPAs cleanly | Use Content Fragments + GraphQL for headless delivery |
| **No CSS scoping** | XF styles bleed into the page or vice versa, causing visual regressions | Apply BEM namespacing or container scoping to all XF CSS |
| **Skipping locale folders** | XF component cannot auto-resolve translations; editors must manually wire each locale | Mirror the site locale folder structure under `/content/experience-fragments/` |
| **Editing XF in Target** | Target only holds a snapshot; changes in Target are lost on re-export from AEM | Always author in AEM, export to Target; treat Target copy as read-only |
| **Deeply nested XFs** (XF inside XF inside XF) | Multiplied SSI requests, unpredictable cache invalidation, hard to debug | Keep XF nesting to one level maximum |
