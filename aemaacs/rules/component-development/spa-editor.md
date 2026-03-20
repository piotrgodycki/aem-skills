---
title: "AEM SPA Editor Development Patterns"
impact: "High — affects architecture, authoring experience, and long-term maintainability of SPA-based AEM projects"
tags: [spa, react, angular, spa-editor, universal-editor, headless, content-fragments, deprecated, migration]
---

# AEM SPA Editor Development Patterns

> **Deprecation Notice**: Adobe deprecated the SPA Editor with AEM as a Cloud Service release 2025.01. No further enhancements or SDK updates will be made. Existing implementations remain supported (P1/P2 issues and security fixes only). All new projects must use the **Universal Editor** or **Content Fragment Editor**. There is no direct migration path from SPA Editor to Universal Editor due to fundamental architectural differences.

This rule covers patterns for teams **maintaining existing** SPA Editor projects and provides guidance for eventual migration.

---

## 1. SPA Editor Architecture

The SPA Editor mediates between AEM and the SPA through JSON-based communication rather than traditional HTML rendering.

**Core principles:**
- The SPA is always in charge of its own display; the editor is isolated from the SPA itself
- AEM always owns the site structure; content authors control page hierarchy
- Content is delivered as a JSON page model via Sling Model Exporter
- Component mapping connects AEM resource types to SPA components

**Editing lifecycle (8 steps):**
1. Editor loads; SPA instantiates in a separate iframe
2. SPA fetches JSON content model and renders components with `data-cq-data-path` attributes
3. Editor detects rendered elements and generates authoring overlays
4. Author clicks an overlay to access the editing toolbar
5. Editor persists changes via POST request to AEM
6. Updated JSON model is retrieved and dispatched through DOM events
7. SPA re-renders affected components
8. Authoring overlays refresh

**Known limitations** (will not be resolved due to deprecation):
- No Target mode, ContextHub, inline image editing, undo/redo, page diff, developer mode, or AEM Launches support

---

## 2. React SPA SDK

### Required Packages

```json
{
  "dependencies": {
    "@adobe/aem-react-editable-components": "~1.0.4",
    "@adobe/aem-spa-component-mapping": "~1.0.5",
    "@adobe/aem-spa-page-model-manager": "~1.0.3"
  }
}
```

| Package | Purpose |
|---------|---------|
| `@adobe/aem-spa-page-model-manager` | Fetches and manages the JSON page model from AEM |
| `@adobe/aem-spa-component-mapping` | Maps AEM resource types to SPA components |
| `@adobe/aem-react-editable-components` | Provides `MapTo`, `withModel`, `Page`, `Container`, `ResponsiveGrid`, and editing wrappers |

### ModelManager Initialization

```javascript
// src/index.js
import { ModelManager, Constants } from '@adobe/aem-spa-page-model-manager';

ModelManager.initialize().then((pageModel) => {
  ReactDOM.render(
    <App
      cqChildren={pageModel[Constants.CHILDREN_PROP]}
      cqItems={pageModel[Constants.ITEMS_PROP]}
      cqItemsOrder={pageModel[Constants.ITEMS_ORDER_PROP]}
      cqPath={ModelManager.rootPath}
      locationPathname={window.location.pathname}
    />,
    document.getElementById('page')
  );
});
```

### App Component with Page Model

```javascript
import { Page, withModel } from '@adobe/aem-react-editable-components';

class App extends Page {
  // Page base class handles rendering child components from the model
}

export default withModel(App);
```

---

## 3. MapTo Pattern with EditConfig

The `MapTo` function is the glue between AEM resource types and SPA components. The `EditConfig` object controls authoring behavior.

### Correct: Complete MapTo with EditConfig

```javascript
import React, { Component } from 'react';
import { MapTo } from '@adobe/aem-react-editable-components';

// Define EditConfig with both emptyLabel and isEmpty
const ImageEditConfig = {
  emptyLabel: 'Image',
  isEmpty: function(props) {
    return !props || !props.src || props.src.trim().length < 1;
  }
};

class Image extends Component {
  render() {
    return this.props.src ? (
      <img src={this.props.src} alt={this.props.alt || ''} />
    ) : null;
  }
}

// MapTo binds the React component to the AEM resource type
MapTo('my-app/components/content/image')(Image, ImageEditConfig);
```

### Correct: EditConfig for a Text component

```javascript
const TextEditConfig = {
  emptyLabel: 'Text',
  isEmpty: function(props) {
    return !props || !props.text || props.text.trim().length < 1;
  }
};

MapTo('my-app/components/content/text')(Text, TextEditConfig);
```

### Incorrect: Missing isEmpty in EditConfig

```javascript
// WRONG — without isEmpty, the editor cannot show a placeholder
// when the component has no content, making it invisible and unclickable
const ImageEditConfig = {
  emptyLabel: 'Image'
  // Missing isEmpty — authors cannot interact with unconfigured components
};
```

### Incorrect: Hardcoded resource type mismatch

```javascript
// WRONG — resource type string must exactly match the AEM component's
// sling:resourceType, including the app name prefix
MapTo('components/content/image')(Image, ImageEditConfig);
// Should be: MapTo('my-app/components/content/image')(...)
```

### Incorrect: Forgetting to export or import mapped components

```javascript
// WRONG — MapTo registration only works if the file is actually imported
// somewhere in the application bundle. An unmapped component renders
// as an empty placeholder in the editor.

// components/Image.js
MapTo('my-app/components/content/image')(Image, ImageEditConfig);
// But never imported in App.js or index.js — registration never executes
```

**Anti-pattern: Using MapTo without EditConfig.** Always provide both `emptyLabel` and `isEmpty`. Without them, authors see blank areas they cannot interact with in the editor.

---

## 4. ResponsiveGrid and Container Components

The `ResponsiveGrid` (Layout Container) is the primary drag-and-drop zone that allows authors to add and arrange components.

### Using ResponsiveGrid in a Page

```javascript
import { Page, MapTo, withComponentMappingContext } from '@adobe/aem-react-editable-components';

class AppPage extends Page {
  // The Page base class automatically renders child components
  // including ResponsiveGrid containers from the page model
}

MapTo('my-app/components/structure/page')(
  withComponentMappingContext(AppPage)
);
```

### Using ResponsiveGrid Directly

```javascript
import { ResponsiveGrid } from '@adobe/aem-react-editable-components';

function Home() {
  return (
    <div className="Home">
      <ResponsiveGrid
        pagePath="/content/my-app/us/en/home"
        itemPath="root/responsivegrid"
      />
    </div>
  );
}
```

- `pagePath` points to the AEM page resource (e.g., `/content/my-app/us/en/home`)
- `itemPath` maps to the container node in the JCR (e.g., `root/responsivegrid` resolves to `.../jcr:content/root/responsivegrid`)

### Responsive Grid CSS

Import AEM's grid SCSS to support layout editing:

```scss
@import '~@adobe/aem-react-editable-components/dist/aem-grid/aem-grid';
```

### Composite Components with Container

Use `withMappable` and `Container` to create composite components (e.g., cards combining image + text):

```javascript
import { Container, withMappable } from '@adobe/aem-react-editable-components';

export const AEMCard = withMappable(Container, {
  resourceType: 'my-app/components/imagecard'
});
```

### Incorrect: Hardcoding content inside a container area

```javascript
// WRONG — defeats the purpose of an editable container
function Home() {
  return (
    <div className="Home">
      <ResponsiveGrid pagePath="/content/my-app/us/en/home" itemPath="root/responsivegrid" />
      {/* Do not hardcode content that should be author-managed */}
      <div className="hardcoded-banner">Buy now!</div>
    </div>
  );
}
```

---

## 5. ModelManager Initialization and Page Model

### Page Model JSON Structure

AEM delivers content as JSON with these key properties:

| Property | Purpose |
|----------|---------|
| `:type` | AEM resource type identifier |
| `:children` | Hierarchical child page resources |
| `:items` | Nested content resources within containers |
| `:itemsOrder` | Ordered list maintaining component sequence |
| `:path` | Content path for page-level items |
| `:hierarchyType` | Structural classification (currently `page`) |

### Page Component Meta Properties

Configure these on the AEM page component to control SPA behavior:

| Property | Purpose |
|----------|---------|
| `cq:datatype="JSON"` | Enables Sling Model JSON endpoint |
| `cq:pagemodel_root_url` | Root model URL (critical for child pages to find the app root) |
| `cq:pagemodel_router` | Enable/disable the built-in ModelRouter |
| `cq:pagemodel_route_filters` | Comma-separated regex patterns for routes the ModelRouter should ignore |

### Communication Setup

The `cq.authoring.pagemodel.messaging` client library must be included for editor communication. Add it via template policy or `customfooterlibs.html`, restricted to the page editor context only:

```html
<!-- customfooterlibs.html -->
<sly data-sly-use.clientLib="${'/libs/granite/sightly/templates/clientlib.html'}">
  <sly data-sly-test="${wcmmode.edit || wcmmode.preview}"
       data-sly-call="${clientLib.all @ categories='cq.authoring.pagemodel.messaging'}" />
</sly>
```

### Correct: Async initialization for Remote SPA

```javascript
import { ModelManager } from '@adobe/aem-spa-page-model-manager';

// Use initializeAsync when the SPA does not need to block rendering
// on model availability (typical for Remote SPA pattern)
ModelManager.initializeAsync();
```

### Incorrect: Initializing ModelManager multiple times

```javascript
// WRONG — ModelManager is a singleton; calling initialize() more than
// once causes unpredictable behavior
ModelManager.initialize().then(/* ... */);
// Later in another module:
ModelManager.initialize().then(/* ... */); // Do NOT do this
```

---

## 6. Remote SPA Pattern

The Remote SPA pattern allows a SPA hosted outside AEM to remain editable in the AEM author environment.

### Environment Configuration

```bash
# .env.development
REACT_APP_HOST_URI=http://localhost:4502
REACT_APP_USE_PROXY=true
REACT_APP_AUTH_METHOD=basic
REACT_APP_BASIC_AUTH_USER=admin
REACT_APP_BASIC_AUTH_PASS=admin
```

### Proxy Setup for Development

Use `http-proxy-middleware` to route AEM requests through the dev server, avoiding CORS issues:

```javascript
// src/setupProxy.js
const { createProxyMiddleware } = require('http-proxy-middleware');

module.exports = function(app) {
  app.use(
    ['/content', '/graphql', '/.model.json'],
    createProxyMiddleware({
      target: process.env.REACT_APP_HOST_URI,
      changeOrigin: true,
      auth: `${process.env.REACT_APP_BASIC_AUTH_USER}:${process.env.REACT_APP_BASIC_AUTH_PASS}`
    })
  );
};
```

### Editable Components in Remote SPA

Remote SPAs use `EditableComponent` wrapper with explicit `pagePath` and `itemPath`:

```javascript
import { EditableComponent, MapTo } from '@adobe/aem-react-editable-components';

const RESOURCE_TYPE = 'my-app/components/text';

const EditConfig = {
  emptyLabel: 'Text',
  isEmpty: (props) => !props || !props.text || props.text.trim().length < 1,
  resourceType: RESOURCE_TYPE
};

const EditableText = (props) => (
  <EditableComponent config={EditConfig} {...props}>
    <WrappedText componentProperties={props} />
  </EditableComponent>
);

MapTo(RESOURCE_TYPE)(EditableText);
```

### Using Fixed Editable Components

```javascript
import EditableText from './editable/EditableText';
import EditableImage from './editable/EditableImage';

function Home() {
  return (
    <div className="Home">
      <EditableTitle
        pagePath="/content/my-app/us/en/home"
        itemPath="title"
      />
      <ResponsiveGrid
        pagePath="/content/my-app/us/en/home"
        itemPath="root/responsivegrid"
      />
    </div>
  );
}
```

### Incorrect: Missing CORS configuration on AEM

```text
# WRONG — Remote SPA will fail to load models without CORS on AEM.
# You must configure an AEM CORS OSGi configuration allowing the
# Remote SPA's origin, or use a proxy during development.
```

### Incorrect: Using relative image paths in Remote SPA

```javascript
// WRONG — images with relative paths break when loaded through AEM Editor
<img src="/static/media/logo.png" />

// CORRECT — use absolute URLs via environment variable
<img src={`${process.env.REACT_APP_PUBLIC_URI}/static/media/logo.png`} />
```

---

## 7. SPA Routing with AEM Page Model

### ModelRouter

The `ModelRouter` module (enabled by default) synchronizes client-side navigation with AEM page models:

- Intercepts `pushState` and `replaceState` calls to fetch corresponding page models
- Pre-fetches models for linked pages
- Can be disabled via `cq:pagemodel_router=disabled` on the page component

### React Router Integration

```javascript
import { BrowserRouter, Route } from 'react-router-dom';
import { ModelManager } from '@adobe/aem-spa-page-model-manager';
import { Page } from '@adobe/aem-react-editable-components';

function App() {
  return (
    <BrowserRouter>
      <Route
        path="*"
        render={(routeProps) => (
          <Page
            cqPath={routeProps.location.pathname}
            // ... other model props
          />
        )}
      />
    </BrowserRouter>
  );
}
```

### Controlling Route Depth

Configure `structureDepth` on the AEM Navigation component to control how many levels of child page models are included in the JSON export. This affects the routes available for client-side navigation without additional server requests.

### Incorrect: Static routes that bypass ModelManager

```javascript
// WRONG — hardcoded routes prevent authors from creating new pages
<Route path="/about" component={AboutPage} />
<Route path="/contact" component={ContactPage} />

// CORRECT — use dynamic routing driven by the AEM page model
// so that new pages created by authors automatically appear
<Route path="*" render={(props) => <Page cqPath={props.location.pathname} />} />
```

### Incorrect: Disabling ModelRouter without alternative

```javascript
// WRONG — disabling ModelRouter without implementing your own model
// fetching logic means child pages will not load their content
// cq:pagemodel_router=disabled (on the page component)
// ... and no custom fetch logic in the SPA
```

---

## 8. SPA + Content Fragment Integration

SPAs can consume Content Fragments via two approaches:

### Approach A: GraphQL Persisted Queries (Headless)

```javascript
import AEMHeadless from '@adobe/aem-headless-client-js';

const aemHeadlessClient = new AEMHeadless({
  serviceURL: process.env.REACT_APP_HOST_URI,
  endpoint: '/content/cq:graphql/my-app/endpoint.json'
});

async function fetchAdventures() {
  const { data } = await aemHeadlessClient.runPersistedQuery(
    'my-app/adventures-all'
  );
  return data.adventureList.items;
}
```

### Approach B: Content Fragment Delivery OpenAPI

The newer OpenAPI-based approach provides REST endpoints for Content Fragment delivery without requiring GraphQL.

### Approach C: Embedded in SPA Editor Page Model

Content Fragments can be exposed through the page model when an AEM Content Fragment component is placed on the page. The SPA component receives CF data as props through the standard MapTo mechanism.

### Guidance

- For **existing SPA Editor projects**, Content Fragments accessed through page model components maintain the editor integration
- For **headless-first** approaches, use GraphQL persisted queries or the OpenAPI delivery APIs
- Do **not** mix both approaches for the same content in the same view; choose one delivery mechanism per component

---

## 9. Angular SPA SDK

The Angular SDK follows equivalent patterns with Angular-specific syntax.

### Required Packages

```json
{
  "dependencies": {
    "@adobe/aem-angular-editable-components": "~1.0.3",
    "@adobe/aem-spa-component-mapping": "~1.0.5",
    "@adobe/aem-spa-page-model-manager": "~1.0.3"
  }
}
```

### Module Setup

```typescript
// app.module.ts
import { SpaAngularEditableComponentsModule } from '@adobe/aem-angular-editable-components';
import { BrowserTransferStateModule } from '@angular/platform-browser';

@NgModule({
  imports: [
    BrowserModule,
    BrowserTransferStateModule,
    SpaAngularEditableComponentsModule,
    AppRoutingModule
  ],
  declarations: [AppComponent],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

### ModelManager Initialization in Angular

```typescript
// app.component.ts
import { ModelManager } from '@adobe/aem-spa-page-model-manager';
import { Constants } from '@adobe/aem-angular-editable-components';

export class AppComponent {
  ngOnInit() {
    ModelManager.initialize().then(this.updateData.bind(this));
  }

  private updateData(model: any) {
    this.items = model[Constants.ITEMS_PROP];
    this.itemsOrder = model[Constants.ITEMS_ORDER_PROP];
    this.path = model[Constants.PATH_PROP];
  }
}
```

### MapTo in Angular

```typescript
import { MapTo } from '@adobe/aem-angular-editable-components';

@Component({
  selector: 'app-image',
  template: `<img [src]="src" [alt]="alt" />`
})
export class ImageComponent {
  @Input() src: string;
  @Input() alt: string;
}

const ImageEditConfig = {
  emptyLabel: 'Image',
  isEmpty: (props: any) => !props || !props.src || props.src.trim().length < 1
};

MapTo('my-angular-app/components/image')(ImageComponent, ImageEditConfig);
```

### Page Component in Angular Template

```html
<aem-page
  [cqPath]="path"
  [cqItems]="items"
  [cqItemsOrder]="itemsOrder">
</aem-page>
```

---

## 10. Migration Considerations: SPA Editor to Universal Editor

### Current State (Post-Deprecation)

| Aspect | SPA Editor (Deprecated) | Universal Editor (Recommended) |
|--------|------------------------|-------------------------------|
| **SDK** | AEM-specific SDK required (`@adobe/aem-react-editable-components`, etc.) | No AEM-specific SDK; uses `corlib.js` with HTML annotations |
| **Frameworks** | React and Angular only | Any web framework |
| **Hosting** | Typically AEM-hosted or Remote SPA pattern | Fully decoupled, hosted anywhere |
| **Content types** | Pages only | Pages and Content Fragments natively |
| **AEM expertise** | Java/Sling Models required | Minimal AEM knowledge needed |
| **AEM versions** | AEM 6.5+ and Cloud Service | AEM 6.5 (2024.11.05+) and Cloud Service |

### Migration Strategy

There is **no automated migration path**. Adobe recommends:

1. **Keep existing SPA Editor sites running** — Adobe continues P1/P2 and security support
2. **Adopt Universal Editor for all new pages, sections, or sites** — even within an existing project
3. **Plan incremental migration** — replace SPA Editor pages one section at a time as business needs arise
4. **Do not start new SPA Editor projects** — the SDKs are in permanent feature freeze

### What Changes in Universal Editor

```html
<!-- Universal Editor uses HTML annotations instead of SDK wrappers -->
<div
  itemscope
  itemtype="https://experience.adobe.com/aue/content/page"
  itemid="urn:aemconnection:/content/my-site/home">
  <h1
    itemprop="title"
    itemtype="text">
    Page Title
  </h1>
</div>
```

Key differences:
- No `MapTo`, no `EditConfig`, no `ModelManager` — all replaced by semantic HTML attributes
- No framework lock-in — works with React, Angular, Vue, Svelte, or plain HTML
- Content can be AEM pages or Content Fragments, addressed by URN
- The `corlib.js` script handles all editor communication

### Practical Guidance for Existing SPA Projects

**Do:**
- Pin your SPA SDK dependency versions; do not expect new features
- Document all MapTo mappings and EditConfig patterns for future migration reference
- Begin evaluating Universal Editor for upcoming sections or microsites
- Keep AEM Core Components updated for security patches

**Do not:**
- Upgrade SPA SDK versions expecting improvements — the SDKs are frozen
- Build new features that deepen SPA Editor coupling (e.g., custom editor plugins)
- Delay planning; start a Universal Editor proof-of-concept to understand the differences
- Attempt to run both SPA Editor and Universal Editor on the same page simultaneously

---

## Quick Reference: Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Missing `isEmpty` in EditConfig | Authors cannot see or click empty components | Always implement `isEmpty` returning `true` when required props are absent |
| Mismatched resource type string | Component never renders in editor | Verify string matches `sling:resourceType` exactly |
| MapTo file never imported | Registration code never executes | Import all mapped component files in your app entry point |
| Multiple `ModelManager.initialize()` calls | Race conditions and duplicate model fetches | Initialize once in `index.js`; use singleton pattern |
| Hardcoded routes in SPA | Authors cannot create or reorder pages | Use dynamic routing driven by the page model |
| Relative asset URLs in Remote SPA | Broken images when rendered in AEM editor | Use absolute URLs with environment variable prefix |
| Starting a new SPA Editor project in 2025+ | Building on a deprecated, frozen platform | Use Universal Editor instead |
