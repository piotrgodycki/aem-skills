---
title: Touch UI Page Editor Customization
impact: HIGH
impactDescription: Page editor extensions control how authors interact with components — incorrect patterns break editing workflows
tags: page-editor, layers, toolbar, overlays, granite-author, events, authoring, editor
---

## Touch UI Page Editor Customization

The AEM page editor is a layered system with distinct frames, overlays, and extension points. All customizations go in `/apps` using overlays or clientlib hooks.

### Page Editor Architecture

The editor separates concerns into two frames:

- **Content Frame**: Renders the actual page in an iframe. Independent CSS/JS scope prevents conflicts.
- **Editor Frame**: Houses the toolbar, side panel, overlays, and authoring controls outside the iframe.

**Overlays** are visual elements positioned over components in the content frame, enabling transparent interaction (selection, drag-and-drop, resize).

### Granite.author Namespace

The `Granite.author` namespace is the central API for page editor scripting.

#### Key Sub-namespaces

| Namespace | Purpose |
|-----------|---------|
| `Granite.author.edit` | Core editing: Dialog, Overlay, Toolbar, EditableActions |
| `Granite.author.design` | Design/policy mode: Dialog, Layer, Overlay |
| `Granite.author.ui` | UI abstractions: Dialog, Overlay, Toolbar |
| `Granite.author.responsive` | Responsive layout: Layer, Overlay, Toolbar |
| `Granite.author.ContentFrame` | Content frame management, loading editables |
| `Granite.author.EditorFrame` | Editor frame controls |
| `Granite.author.DialogFrame` | Dialog frame operations |
| `Granite.author.editables` | Editable content management |
| `Granite.author.editableHelper` | Utilities for editables |
| `Granite.author.selection` | Selection management |
| `Granite.author.overlayManager` | Overlay lifecycle management |
| `Granite.author.pageInfo` | Page information handling |
| `Granite.author.pageDesign` | Page design operations |
| `Granite.author.components` | Component management |

#### Key Methods

```javascript
// Initialize editor (called internally)
Granite.author.init(config, useApplyDefaults);

// Load page information from server
Granite.author.loadPageInfo(cache); // returns Promise

// Load page design
Granite.author.loadPageDesign(); // returns Promise

// Load all available components for the page
Granite.author.loadComponents();
```

#### Granite.author.Component

```javascript
// Represents an AEM component resource
var component = new Granite.author.Component(componentConfig);
component.getPath();          // component resource path
component.getResourceType();  // sling:resourceType
component.getTitle();         // component title
component.getGroup();         // component group
component.getDropTarget(id);  // drop target config
component.toHtml();           // HTMLElement representation
```

#### Granite.author.edit.Dialog

```javascript
// Represents a component dialog opened on an Editable
var dialog = new Granite.author.edit.Dialog(editable);
dialog.getConfig();        // dialog configuration
dialog.getRequestData();   // additional request data
dialog.onOpen();           // fires when dialog opens
dialog.onClose();          // fires when dialog closes
dialog.onReady();          // fires when dialog is ready
dialog.onSuccess();        // fires after successful submission
// Triggers "cq-persistence-after-update" event on success
```

### Page Editor Events

Listen for editor lifecycle events on the `document` or `channel`:

| Event | When Fired |
|-------|------------|
| `cq-editor-loaded` | Editor initialization complete |
| `cq-layer-activated` | A layer (mode) becomes active |
| `cq-content-frame-loaded` | Content iframe finishes loading |
| `cq-editables-loaded` | All editables are loaded and ready |
| `cq-page-info-loaded` | Page information loaded from server |
| `cq-page-design-loaded` | Page design information loaded |
| `cq-overlay-click` | Author clicks a component overlay |
| `cq-overlay-hover` | Author hovers over a component overlay |
| `cq-overlay-dblclick` | Author double-clicks a component overlay |
| `cq-persistence-before-update` | Before a content persistence operation |
| `cq-persistence-after-update` | After a content persistence operation |
| `cq-sidepanel-*` | Side panel interaction events |
| `cq-history-*` | Undo/redo history events |

```javascript
// Example: Listen for layer changes
$(document).on("cq-layer-activated", function(event) {
    var layerName = event.layer;
    if (layerName === "Edit") {
        // Edit mode activated
    }
});

// Example: Listen for editable selection
$(document).on("cq-editables-loaded", function() {
    // All editables are ready for interaction
});
```

### Layer System

The page editor uses layers to provide different editing modes. Each layer controls what overlays, toolbars, and interactions are available.

#### Built-in Layers

| Layer | Purpose |
|-------|---------|
| **Edit** | Default authoring — component selection, editing, drag-and-drop |
| **Layout** | Responsive grid layout editing (resize, reorder) |
| **Preview** | Content preview without editing chrome |
| **Annotate** | Add review annotations to components |
| **Developer** | Developer information overlay (component paths, types) |
| **Targeting** | Adobe Target integration for personalization |
| **MSM** | Multi Site Manager — highlights live copy/inheritance status |

#### Creating Custom Layers

Reference implementation: `/libs/wcm/msm/content/touch-ui/authoring/editor/js/msm.Layer.js`

A custom layer must:
1. Define a Layer class extending `Granite.author.Layer`
2. Register with the `Granite.author.LayerManager`
3. Provide overlay and toolbar implementations

GitHub example: `aem-authoring-new-layer-mode` (Adobe Marketing Cloud)

### Extending the Component Toolbar

When a component is selected in the editor, a toolbar appears with actions. Add custom actions:

```javascript
// Custom toolbar action (conceptual pattern)
// Register via clientlib with category cq.authoring.editor.sites.page.hook
(function($, channel) {
    "use strict";

    // Listen for toolbar rendering
    channel.on("cq-layer-activated", function(event) {
        if (event.layer === "Edit") {
            // Custom toolbar logic here
        }
    });
})(jQuery, jQuery(document));
```

GitHub example: `aem-authoring-extension-toolbar-screenshot` (Adobe Marketing Cloud)

### Custom In-Place Editors

Create editors for direct content editing (no dialog). A custom in-place editor must implement:

- `setUp()` — Initialize the editor
- `tearDown()` — Clean up
- Register via `editor.register`
- Connect to component resource types via `cq:inplaceEditing` config

GitHub example: `aem-authoring-extension-inplace-editor` (Adobe Marketing Cloud)

### Extending the Side Panel (Asset Browser)

Add custom asset sources or filter predicates to the side panel asset browser:

- Implement `com.day.cq.commons.predicate.AbstractNodePredicate` on the server side
- Register as an asset finder source

GitHub example: `aem-authoring-extension-assetfinder-flickr` (Adobe Marketing Cloud)

### Customization Approaches

#### Clientlib Hook Pattern

Create a clientlib that hooks into the page editor:

```xml
<!-- /apps/myproject/clientlibs/editor-hook/.content.xml -->
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="cq:ClientLibraryFolder"
    categories="[cq.authoring.editor.sites.page.hook]"
    dependencies="[cq.authoring.editor.sites.page]"/>
```

JavaScript in this clientlib runs when the page editor loads.

#### Overlay Pattern

Overlay `/libs` resources under `/apps` to modify editor behavior:

```
/apps/cq/gui/components/authoring/editors/...  (overlays /libs equivalent)
```

Use the Sling Resource Merger for minimal diffs.

### Pitfalls

- Do NOT modify `/libs` directly — overlays in `/apps` only
- Do NOT rely on internal editor APIs that may change between AEM versions
- Do NOT add heavy JavaScript to `cq.authoring.editor.sites.page.hook` — it loads on every page edit
- Do NOT mix editor-frame JavaScript with content-frame JavaScript — they run in separate frames
- Page editor clientlibs load in the editor frame, NOT in the content iframe
- Content frame has its own independent CSS/JS scope — no conflicts with editor
