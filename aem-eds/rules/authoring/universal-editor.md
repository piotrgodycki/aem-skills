---
title: Universal Editor Instrumentation
impact: HIGH
impactDescription: Universal Editor replaces the deprecated SPA Editor — correct instrumentation is required for new AEM projects
tags: universal-editor, data-attributes, authoring, spa-editor-replacement
---

## Universal Editor Instrumentation

The Universal Editor is Adobe's replacement for the deprecated SPA Editor (deprecated 2025.01). It works with any web framework via HTML data attributes — no SDK lock-in required.

### SPA Editor Deprecation Notice

The SPA Editor is deprecated with no migration path to Universal Editor. Affected SDKs (feature-frozen):
- `@adobe/aem-react-editable-components`
- `@adobe/aem-spa-component-mapping`
- `@adobe/aem-spa-page-model-manager`

**Existing projects**: Continue using SPA Editor (P1/P2 bug fixes only)
**New projects**: Use Universal Editor

### Data Attributes

| Attribute | Purpose | When Required |
|-----------|---------|---------------|
| `data-aue-resource` | URN to the AEM resource being edited | Always |
| `data-aue-prop` | JCR property to update | For in-context editing |
| `data-aue-type` | Editing interface type | Always with `data-aue-prop` |
| `data-aue-filter` | Allowed components/RTE features | Optional |
| `data-aue-label` | Display label in editor | Optional |
| `data-aue-model` | Model for properties panel | Optional |

### Item Types (`data-aue-type` values)

- `text` — Simple text editing
- `richtext` — Rich text with formatting toolbar
- `media` — Asset picker (images/videos)
- `container` — Paragraph system for child components
- `component` — Movable/deletable component
- `reference` — Content/Experience Fragment reference

### Instrumentation Example

```html
<div data-aue-resource="urn:aem:/content/mysite/page/jcr:content/root/teaser"
     data-aue-type="component"
     data-aue-label="Teaser">
  <h2 data-aue-prop="jcr:title"
      data-aue-type="text">Hello World</h2>
  <div data-aue-prop="description"
       data-aue-type="richtext">
    <p>Some content</p>
  </div>
  <img data-aue-prop="fileReference"
       data-aue-type="media"
       src="/content/dam/image.jpg" alt="Alt text"/>
</div>
```

### Container Example

```html
<div data-aue-resource="urn:aem:/content/mysite/page/jcr:content/root"
     data-aue-type="container"
     data-aue-filter="allowed-components"
     data-aue-label="Main Content">
  <!-- Child components rendered here -->
</div>
```

### Key Differences from SPA Editor

| SPA Editor (Deprecated) | Universal Editor |
|--------------------------|------------------|
| React/Angular SDK required | Any framework or vanilla JS |
| `MapTo()` component mapping | `data-aue-*` HTML attributes |
| `ModelManager` initialization | No SDK initialization |
| Tight coupling to AEM SDK | Decoupled architecture |
| Feature-frozen | Actively developed |
| Template Editor, Style System | Limited equivalents (in progress) |

### Best Practices

1. **Use semantic `data-aue-label` values** — authors see these in the editor
2. **Set `data-aue-type` correctly** — wrong type gives wrong editing interface
3. **Use URN format for resources** — `urn:aem:/content/...`
4. **Keep DOM structure stable** — excessive JS DOM manipulation can break authoring
5. **Test in Universal Editor** — verify all editable fields work in authoring mode

### Pitfalls

- No direct migration path from SPA Editor — architecture is fundamentally different
- Template Editor and Style System equivalents don't exist yet in Universal Editor
- Responsive Grid features have no Universal Editor equivalent yet
- DOM manipulation in JS must not remove/replace elements with `data-aue-*` attributes
