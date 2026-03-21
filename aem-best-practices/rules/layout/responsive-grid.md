---
title: Responsive Grid & Layout
impact: MEDIUM
impactDescription: Correct grid and breakpoint configuration ensures consistent responsive behavior across devices
tags: responsive, grid, breakpoints, layout-container, template, mobile, columns, nesting
---

## Responsive Grid & Layout

AEM's responsive grid is a 12-column CSS grid system with configurable breakpoints. Layout is controlled via editable templates, Layout Containers, and content policies.

---

### 1. Default Breakpoints

| Breakpoint | Max Width | Class Prefix | Typical Devices |
|-----------|-----------|--------------|-----------------|
| Phone | 768px | `aem-GridColumn--phone--` | Mobile phones |
| Tablet | 1200px | `aem-GridColumn--tablet--` | Tablets, small laptops |
| Desktop | 1200px+ | `aem-GridColumn--default--` | Desktops, large screens |

Adobe recommends **no more than 3 breakpoints** (including default). More breakpoints add exponential QA complexity with diminishing returns.

---

### 2. Grid CSS Classes

#### Basic Grid HTML Structure

```html
<!-- 12-column grid container -->
<div class="aem-Grid aem-Grid--12 aem-Grid--default--12">

  <!-- 8 columns on desktop, 6 on tablet, 12 on phone -->
  <div class="aem-GridColumn aem-GridColumn--default--8 aem-GridColumn--tablet--6 aem-GridColumn--phone--12">
    <!-- Main content area -->
  </div>

  <!-- 4 columns on desktop, 6 on tablet, 12 on phone -->
  <div class="aem-GridColumn aem-GridColumn--default--4 aem-GridColumn--tablet--6 aem-GridColumn--phone--12">
    <!-- Sidebar -->
  </div>
</div>
```

#### Column Span Classes

```
aem-GridColumn--default--1   through  aem-GridColumn--default--12
aem-GridColumn--tablet--1    through  aem-GridColumn--tablet--12
aem-GridColumn--phone--1     through  aem-GridColumn--phone--12
```

#### Offset and Hide Classes

```html
<!-- Offset: push column to the right -->
<div class="aem-GridColumn aem-GridColumn--default--6 aem-GridColumn--offset--default--3">
  <!-- Centered 6-column block with 3-column offset -->
</div>

<!-- Hide at specific breakpoints -->
<div class="aem-GridColumn aem-GridColumn--default--4 aem-GridColumn--phone--hide">
  <!-- Hidden on mobile -->
</div>
```

#### Newline/Clear Classes

```html
<!-- Force a new row -->
<div class="aem-GridColumn aem-GridColumn--default--12 aem-GridColumn--default--newline">
  <!-- Starts on a new row -->
</div>
```

---

### 3. Breakpoint Configuration

#### At Template Type Level (Recommended)

```
/conf/<project>/settings/wcm/template-types/<type>/structure/jcr:content/cq:responsive/breakpoints
```

```xml
<breakpoints jcr:primaryType="nt:unstructured">
    <phone
        jcr:primaryType="nt:unstructured"
        title="Phone"
        width="{Long}768"/>
    <tablet
        jcr:primaryType="nt:unstructured"
        title="Tablet"
        width="{Long}1200"/>
</breakpoints>
```

#### Custom Breakpoints

If the default breakpoints don't match your design system:

```xml
<!-- Example: phone/tablet/laptop/desktop (4 breakpoints — use cautiously) -->
<breakpoints jcr:primaryType="nt:unstructured">
    <phone
        jcr:primaryType="nt:unstructured"
        title="Phone"
        width="{Long}576"/>
    <tablet
        jcr:primaryType="nt:unstructured"
        title="Tablet"
        width="{Long}768"/>
    <laptop
        jcr:primaryType="nt:unstructured"
        title="Laptop"
        width="{Long}1200"/>
</breakpoints>
```

**Important:** Custom breakpoints require matching CSS. The `grid.less/grid.scss` must generate classes for your custom breakpoint names.

#### Generating Custom Grid CSS

```scss
// grid.scss — generate grid classes for custom breakpoints
$grid-columns: 12;

// Generate column classes per breakpoint
@each $bp-name, $bp-width in (phone: 576px, tablet: 768px, laptop: 1200px) {
  @media (max-width: $bp-width) {
    @for $i from 1 through $grid-columns {
      .aem-GridColumn--#{$bp-name}--#{$i} {
        width: #{($i / $grid-columns * 100) + '%'};
      }
    }
    .aem-GridColumn--#{$bp-name}--hide {
      display: none !important;
    }
  }
}
```

---

### 4. Layout Container

The Layout Container is a responsive grid component that enables drag-and-drop authoring.

#### Container Configuration

```xml
<!-- Component definition for Layout Container proxy -->
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0"
    jcr:primaryType="cq:Component"
    jcr:title="Layout Container"
    sling:resourceSuperType="wcm/foundation/components/responsivegrid"
    componentGroup="My Project - Structure"/>
```

**Layout property:**
```
layout = responsiveGrid  (on the container's jcr:content node)
```

#### Container Nesting

```
Page
└── Root Container (12 columns)
    ├── Header Component (span 12)
    ├── Main Container (span 8)     ← Inner grid resets to 12 columns
    │   ├── Hero (span 12)          ← 12 of inner container = 8 of outer
    │   ├── Text (span 6)
    │   └── Image (span 6)
    └── Sidebar Container (span 4)  ← Inner grid resets to 12 columns
        ├── Navigation (span 12)
        └── Banner (span 12)
```

**Rules for nesting:**
- All nested containers must use `layout = responsiveGrid`
- Don't mix `layout = simple` with `responsiveGrid` in a hierarchy
- Inner containers get their own 12-column grid (not subdivisions of parent)

---

### 5. Template-Level Layout Policy

Control grid behavior through template policies:

```xml
<!-- Policy definition: /conf/myproject/settings/wcm/policies/wcm/foundation/components/responsivegrid/default -->
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="Default Grid Policy"
    sling:resourceType="wcm/foundation/components/responsivegrid">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <breakpoints jcr:primaryType="nt:unstructured">
            <phone jcr:primaryType="nt:unstructured" title="Phone" width="{Long}768"/>
            <tablet jcr:primaryType="nt:unstructured" title="Tablet" width="{Long}1200"/>
        </breakpoints>
    </cq:responsive>
</jcr:root>
```

---

### 6. Editable Template Structure

```
/conf/myproject/settings/wcm/templates/page-template/
├── structure/                    # Fixed structure (non-editable by authors)
│   └── jcr:content/
│       ├── root/                 # Root Layout Container (responsivegrid)
│       │   ├── header            # Locked header component
│       │   ├── responsivegrid    # Unlocked area for authors
│       │   └── footer            # Locked footer component
│       └── cq:responsive/
│           └── breakpoints/      # Breakpoint definitions
├── initial/                      # Default content for new pages
│   └── jcr:content/root/
│       └── responsivegrid/
│           └── text              # Pre-populated text component
├── policies/                     # Policy mappings
│   └── jcr:content/root/
│       └── @cq:policy            # Reference to policy node
└── thumbnail.png                 # Template icon
```

---

### 7. Responsive Images with Grid

The Core Image component handles responsive images automatically:

```html
<!-- Core Image with responsive behavior -->
<div class="cmp-image" data-cmp-is="image"
     data-cmp-widths="[320,480,640,800,1024,1280,1600]">
  <img src="/content/dam/mysite/hero.coreimg.jpeg/1920.hero.jpeg"
       srcset="/content/dam/mysite/hero.coreimg.320.jpeg 320w,
               /content/dam/mysite/hero.coreimg.640.jpeg 640w,
               /content/dam/mysite/hero.coreimg.1280.jpeg 1280w"
       sizes="(max-width: 768px) 100vw, 50vw"
       loading="lazy" alt="Hero image"/>
</div>
```

**Configure widths in the Image component policy:**

```
Available widths: 320, 480, 640, 800, 1024, 1280, 1600, 1920
```

---

### 8. Fixed-Width Layouts

For content that shouldn't be full-width:

```scss
// Container with max-width constraint
.cmp-container--fixed {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 $spacing-md;

  @include respond-to(phone) {
    padding: 0 $spacing-sm;
  }
}
```

---

### 9. Best Practices

1. Define breakpoints at the **template type level** — not per-template
2. Use **no more than 3 breakpoints** total (phone, tablet, desktop)
3. Use a **12-column grid** (standard)
4. Test layout at **all breakpoints** during development
5. Use the **Core Image component** for automatic responsive images
6. Layout mode in the page editor lets authors resize components per breakpoint
7. Use **content policies** to control grid behavior per template
8. Define **max-width constraints** in CSS, not in grid configuration

---

### 10. Anti-Patterns

#### Breakpoints Per Template Instead of Template Type

```
// WRONG — inconsistent breakpoints across templates
/conf/myproject/settings/wcm/templates/homepage/structure/.../breakpoints
/conf/myproject/settings/wcm/templates/article/structure/.../breakpoints (different values!)

// CORRECT — define once at the template type level
/conf/myproject/settings/wcm/template-types/page/structure/.../breakpoints
```

#### More Than 3 Breakpoints

```xml
<!-- WRONG — 5 breakpoints = exponential QA burden -->
<breakpoints>
    <xs width="{Long}375"/>
    <sm width="{Long}576"/>
    <md width="{Long}768"/>
    <lg width="{Long}1024"/>
    <xl width="{Long}1200"/>
</breakpoints>

<!-- CORRECT — 2 breakpoints (3 with default) -->
<breakpoints>
    <phone width="{Long}768"/>
    <tablet width="{Long}1200"/>
</breakpoints>
```

#### Mixing Layout Types in Nested Containers

```
// WRONG — simple layout inside responsiveGrid
Root Container (layout=responsiveGrid)
  └── Inner Container (layout=simple)  ← breaks grid behavior

// CORRECT — consistent layout type
Root Container (layout=responsiveGrid)
  └── Inner Container (layout=responsiveGrid)
```

#### CSS Media Queries Not Matching AEM Breakpoints

```scss
// WRONG — CSS breakpoints don't match AEM grid breakpoints
@media (max-width: 640px) { }  // AEM grid breaks at 768px

// CORRECT — match CSS to AEM breakpoints
@media (max-width: 768px) { }  // Matches phone breakpoint
@media (max-width: 1200px) { } // Matches tablet breakpoint
```
