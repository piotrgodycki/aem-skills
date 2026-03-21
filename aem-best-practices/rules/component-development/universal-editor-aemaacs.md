---
title: Universal Editor for AEMaaCS
impact: HIGH
impactDescription: Universal Editor is the strategic replacement for Page Editor and SPA Editor — incorrect instrumentation causes broken editing, missing fields, and container failures
tags: universal-editor, data-aue, instrumentation, cors, containers, content-types, remote-spa, extension-points
---

## Universal Editor for AEMaaCS

The Universal Editor (UE) is Adobe's next-generation visual editor that replaces both the Page Editor (for traditional AEM) and the SPA Editor (deprecated). It works with any frontend framework via CORS-based instrumentation using `data-aue-*` attributes.

---

### 1. Architecture

```
Browser (unifiedshell.adobe.com)
  ├── Universal Editor Shell
  │     ├── Properties Panel (right rail)
  │     ├── Content Tree
  │     └── Preview iframe
  │           └── Your page (any origin, any framework)
  │                 └── data-aue-* attributes → tell UE what's editable
  │
  └── Universal Editor Service (Adobe-hosted)
        ├── Reads content via AEM API
        ├── Writes content via AEM API
        └── Handles CORS between UE and AEM
```

**Key difference from Page Editor:**
- Page Editor: page runs inside AEM author, editor overlays injected via server-side
- Universal Editor: page runs on any origin (AEM, Vercel, etc.), editor communicates via API

---

### 2. Connection Setup

#### Meta Tags

Every UE-editable page needs these meta tags in `<head>`:

```html
<head>
    <!-- Required: tell UE where content lives -->
    <meta name="urn:auecon:aemconnection" content="aem:https://author-p12345-e67890.adobeaemcloud.com"/>

    <!-- Required: Universal Editor CORS library -->
    <script src="https://universal-editor-service.experiencecloud.live/corslib/LATEST" async></script>
</head>
```

For HTL pages:

```html
<head data-sly-use.clientlib="core/wcm/components/commons/v1/templates/clientlib.html">
    <meta name="urn:auecon:aemconnection"
          content="aem:${properties['ue-connection'] || 'aem:https://author-p12345-e67890.adobeaemcloud.com'}"/>
    <script src="https://universal-editor-service.experiencecloud.live/corslib/LATEST" async></script>
</head>
```

---

### 3. Instrumentation Attributes

#### Core Attributes

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `data-aue-resource` | Content path (what to edit) | `urn:aemconnection:/content/mysite/en/jcr:content/root/hero` |
| `data-aue-type` | Content type (how to edit) | `reference`, `richtext`, `text`, `container` |
| `data-aue-prop` | Property name | `jcr:title`, `fileReference`, `text` |
| `data-aue-label` | Display name in UE panel | `Hero Title` |
| `data-aue-model` | Component model definition path | `myproject/components/hero` |
| `data-aue-filter` | Allowed children for containers | `myproject/filters/section` |
| `data-aue-behavior` | Container behavior | `component` |

#### Basic Component Instrumentation

```html
<!-- HTL component with UE instrumentation -->
<div class="cmp-hero"
     data-aue-resource="urn:aemconnection:${resource.path}"
     data-aue-type="component"
     data-aue-label="Hero"
     data-aue-model="myproject/components/hero">

    <h1 data-aue-prop="jcr:title"
        data-aue-type="text"
        data-aue-label="Title">
        ${properties.jcr:title}
    </h1>

    <div data-aue-prop="text"
         data-aue-type="richtext"
         data-aue-label="Description">
        ${properties.text @ context='html'}
    </div>

    <img data-aue-prop="fileReference"
         data-aue-type="media"
         data-aue-label="Hero Image"
         src="${properties.fileReference}"/>
</div>
```

---

### 4. Content Types

| Type | Widget | Use Case |
|------|--------|----------|
| `text` | Inline text editing | Headings, labels, short text |
| `richtext` | Rich text editor | Body text, formatted content |
| `media` | Asset picker | Images, videos, documents |
| `reference` | Content reference picker | Content Fragments, pages |
| `component` | Component properties panel | Full component editing |
| `container` | Drag-and-drop zone | Paragraph systems, layout containers |
| `boolean` | Toggle | Checkboxes, feature flags |
| `number` | Number input | Quantities, dimensions |
| `date-time` | Date/time picker | Dates, scheduling |
| `select` | Dropdown | Predefined options |
| `tag` | Tag picker | AEM tags |
| `aem-content` | AEM content browser | Pages, XFs |

---

### 5. Container Instrumentation

#### Editable Container (Paragraph System Equivalent)

```html
<div class="cmp-container"
     data-aue-resource="urn:aemconnection:${resource.path}"
     data-aue-type="container"
     data-aue-label="Main Content"
     data-aue-filter="myproject/filters/main-content"
     data-aue-behavior="component">

    <!-- Child components rendered here -->
    <sly data-sly-resource="${'par' @ resourceType='wcm/foundation/components/parsys'}"/>
</div>
```

#### Component Filter Definition

```json
// /apps/myproject/filters/main-content
{
    "id": "main-content",
    "components": [
        { "title": "Text", "id": "myproject/components/text", "icon": "Text" },
        { "title": "Image", "id": "myproject/components/image", "icon": "Image" },
        { "title": "Teaser", "id": "myproject/components/teaser", "icon": "ViewCard" },
        { "title": "Columns", "id": "myproject/components/columns", "icon": "ViewColumn" }
    ]
}
```

---

### 6. Component Model Definitions

Define the properties panel for each component:

```json
// /apps/myproject/components/hero/_cq_dialog/.content.xml → replaced by model
// For UE, define component model:

// /apps/myproject/models/hero
{
    "id": "myproject/components/hero",
    "fields": [
        {
            "component": "text",
            "name": "jcr:title",
            "label": "Title",
            "valueType": "string",
            "required": true
        },
        {
            "component": "richtext",
            "name": "text",
            "label": "Description",
            "valueType": "string"
        },
        {
            "component": "aem-content",
            "name": "fileReference",
            "label": "Image",
            "valueType": "string",
            "picker": "asset"
        },
        {
            "component": "select",
            "name": "variant",
            "label": "Variant",
            "valueType": "string",
            "options": [
                { "name": "Default", "value": "default" },
                { "name": "Full Width", "value": "full-width" },
                { "name": "Centered", "value": "centered" }
            ]
        },
        {
            "component": "boolean",
            "name": "hideOnMobile",
            "label": "Hide on Mobile",
            "valueType": "boolean"
        }
    ]
}
```

---

### 7. Remote SPA with Universal Editor

Edit a React/Next.js app hosted externally:

```jsx
// React component with UE instrumentation
import { useUniversalEditor } from '@adobe/universal-editor-react';

export default function HeroComponent({ title, text, image, resourcePath }) {
  return (
    <section
      className="hero"
      data-aue-resource={`urn:aemconnection:${resourcePath}`}
      data-aue-type="component"
      data-aue-label="Hero"
      data-aue-model="myproject/components/hero"
    >
      <h1
        data-aue-prop="jcr:title"
        data-aue-type="text"
        data-aue-label="Title"
      >
        {title}
      </h1>

      <div
        data-aue-prop="text"
        data-aue-type="richtext"
        data-aue-label="Description"
        dangerouslySetInnerHTML={{ __html: text }}
      />

      {image && (
        <img
          data-aue-prop="fileReference"
          data-aue-type="media"
          data-aue-label="Image"
          src={image}
          alt={title}
        />
      )}
    </section>
  );
}
```

---

### 8. CORS Configuration

AEM must allow CORS from the Universal Editor origin:

```json
// com.adobe.granite.cors.impl.CORSPolicyImpl~ue.cfg.json
{
    "alloworigin": [
        "https://experience.adobe.com",
        "https://universal-editor-service.experiencecloud.live"
    ],
    "allowedpaths": [
        "/content/.*",
        "/conf/.*",
        "/apps/.*"
    ],
    "allowCredentials": true,
    "maxage:Integer": 86400,
    "allowedmethods": [
        "GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"
    ],
    "allowedheaders": [
        "Content-Type",
        "Authorization",
        "X-Requested-With",
        "X-CSRF-Token"
    ]
}
```

---

### 9. UE vs Page Editor Migration

| Feature | Page Editor | Universal Editor |
|---------|------------|-----------------|
| Hosting | AEM author only | Any origin |
| Framework | HTL only (or SPA SDK) | Any (HTL, React, Next.js, etc.) |
| Instrumentation | `cq:editConfig` XML nodes | `data-aue-*` HTML attributes |
| Component definition | `cq:dialog` XML | JSON model definitions |
| Container | `parsys` / Layout Container | `data-aue-type="container"` |
| InPlace editing | `cq:inplaceEditing` config | `data-aue-type="text/richtext"` |
| Drop targets | `cq:dropTargets` | `data-aue-type="media"` |

#### Migration Steps

```
1. Keep existing cq:dialog for Page Editor backward compatibility
2. Add data-aue-* attributes to HTL templates
3. Create component model JSON definitions
4. Configure CORS for Universal Editor
5. Test in both editors during transition
6. Remove cq:editConfig once fully migrated
```

---

### 10. Extension Points

#### UI Extensions (App Builder)

```javascript
// Custom panel in Universal Editor
import { register } from '@adobe/uix-guest';

const init = async () => {
  const guestConnection = await register({
    id: 'myproject-ue-extension',
    methods: {
      panel: {
        get() {
          return [
            {
              id: 'seo-panel',
              title: 'SEO Analysis',
              url: '/seo-panel.html'
            }
          ];
        }
      }
    }
  });
};

init().catch(console.error);
```

---

### 11. Anti-Patterns

#### Mixing Page Editor and UE Instrumentation

```html
<!-- WRONG — both systems active causes conflicts -->
<div data-aue-type="component" data-aue-resource="..."
     cq:editConfig="...">

<!-- CORRECT — use one system per component
     During migration, UE attributes are ignored by Page Editor
     and cq:editConfig is ignored by UE, so both can coexist -->
```

#### Missing CORS Configuration

```
// WRONG — UE can't communicate with AEM
// Error: "Cross-Origin Request Blocked"
// Symptoms: editor loads but can't read/write content

// CORRECT — configure CORS policy for UE origins (see section 8)
```

#### Hardcoded Connection URLs

```html
<!-- WRONG — hardcoded author URL breaks across environments -->
<meta name="urn:auecon:aemconnection" content="aem:https://author-p12345-e67890.adobeaemcloud.com"/>

<!-- CORRECT — use context-aware configuration or Sling Externalizer -->
<meta name="urn:auecon:aemconnection" content="aem:${caconfig.ueConnectionUrl}"/>
```

#### Over-Instrumenting Non-Editable Elements

```html
<!-- WRONG — making computed/derived content editable -->
<span data-aue-prop="readTime" data-aue-type="text">5 min read</span>
<!-- This is calculated, not authored — don't instrument it -->

<!-- CORRECT — only instrument authored properties -->
```
