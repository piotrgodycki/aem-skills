---
title: Coral UI 3 & Granite UI Frontend Framework
impact: HIGH
impactDescription: Touch UI authoring relies entirely on Coral UI 3 and Granite UI — incorrect usage breaks dialogs, editor extensions, and author experience
tags: coral-ui, granite-ui, javascript, frontend, web-components, clientlib, foundation
---

## Coral UI 3 & Granite UI Frontend Framework

AEM Cloud Service Touch UI is built on two frontend layers: **Coral UI 3** (Adobe's Web Components library) and **Granite UI** (server-side Sling components that render Coral markup). All authoring interfaces — dialogs, consoles, page editor — use these frameworks.

### Coral UI 3 Architecture

Coral UI 3 uses the Web Components Custom Elements specification. Components are HTML custom elements (e.g., `<coral-alert>`, `<coral-select>`, `<coral-dialog>`).

#### Instantiating Coral Components from JavaScript

```javascript
// Constructor approach
var alert = new Coral.Alert();
alert.header.innerHTML = "Notice";
alert.content.innerHTML = "This is an alert message.";
alert.variant = "info";
document.body.appendChild(alert);

// DOM createElement approach
var select = document.createElement("coral-select");
document.body.appendChild(select);
```

**Critical**: Self-closing tags are NOT supported. Always use end tags:
```html
<!-- WRONG -->
<coral-icon icon="edit" />

<!-- CORRECT -->
<coral-icon icon="edit"></coral-icon>
```

#### Coral.commons.ready()

In polyfilled browsers, custom elements upgrade asynchronously. Use `Coral.commons.ready()` to ensure components are upgraded before accessing properties:

```javascript
var el = document.createElement("div");
el.innerHTML = '<coral-numberinput></coral-numberinput>';
var numberInput = el.firstElementChild;
document.body.appendChild(el);

Coral.commons.ready(el, function() {
    // Safe to use component API now
    numberInput.value = 10;
});
```

**Workarounds without ready()**:
- Set values via attributes in markup: `<coral-progress value="100"></coral-progress>`
- Use `setAttribute()`: `el.setAttribute("value", "100");`

#### Coral Component Properties

Every Coral 3 component property has a JavaScript getter/setter. This replaces Coral 2's `data-*` attribute pattern:

```javascript
// Coral 3 — direct property access
var dialog = new Coral.Dialog();
dialog.header.innerHTML = "Title";
dialog.variant = "warning";
dialog.closable = "on";

// Properties also reflect as attributes
dialog.setAttribute("variant", "error");
```

#### Coral Event System

Coral components fire standard DOM events. Listen with `addEventListener` or jQuery:

```javascript
// Coral Select change event
document.querySelector("coral-select").addEventListener("change", function(e) {
    console.log("Selected value:", e.target.value);
});

// Coral Dialog events
dialog.addEventListener("coral-overlay:open", function() { /* opened */ });
dialog.addEventListener("coral-overlay:close", function() { /* closed */ });
dialog.addEventListener("coral-overlay:beforeopen", function(e) {
    e.preventDefault(); // cancel opening
});
```

#### Content Zones

Coral components organize content into named zones:

```html
<coral-alert>
    <coral-alert-header>INFO</coral-alert-header>
    <coral-alert-content>Alert message here</coral-alert-content>
</coral-alert>
```

Access zones via JavaScript: `alert.header.innerHTML = "Updated";`

#### Coral.Collection Interface

Multi-item components expose a standard collection API:

```javascript
var select = document.querySelector("coral-select");
select.items.add({ value: "new", content: { textContent: "New Item" } });
select.items.remove(select.items.first());
select.items.getAll(); // array of all items
```

#### Coral Base Utilities

| Namespace | Purpose |
|-----------|---------|
| `Coral.commons` | `ready()`, utility functions |
| `Coral.i18n` | Internationalization |
| `Coral.transform` | Built-in value transformers (number, boolean, string) |
| `Coral.validate` | Validation functions |
| `Coral.property` | Property descriptor utilities |
| `Coral.keys` | Keyboard event handling |

### Granite UI Client-Side JavaScript

#### Key Namespaces

| Object | Purpose |
|--------|---------|
| `Granite` | Root namespace for all Granite client-side APIs |
| `Granite.HTTP` | HTTP utility methods (URL encoding, context path handling) |
| `Granite.I18n` | Client-side internationalization (`Granite.I18n.get("key")`) |
| `Granite.author` | Page editor authoring API (components, editables, layers) |
| `Granite.UI.Foundation` | Foundation UI utilities |

#### Granite.HTTP

```javascript
// Get context path
var contextPath = Granite.HTTP.getContextPath();

// Externalize a URL (prepend context path)
var url = Granite.HTTP.externalize("/content/mysite/page.html");

// Encode path
var encoded = Granite.HTTP.encodePath("/content/my site/page");
```

#### Granite.I18n

```javascript
// Simple translation
var label = Granite.I18n.get("Save");

// Translation with placeholder
var msg = Granite.I18n.get("Showing {0} of {1} items", [count, total]);

// Variant (disambiguation)
var label = Granite.I18n.get("Save", null, "button label");
```

#### Granite.UI.Foundation.Utils

```javascript
// Debounce function calls
var debouncedSave = Granite.UI.Foundation.Utils.debounce(save, 300);

// Reverse array iteration (like Array.every but reversed)
Granite.UI.Foundation.Utils.everyReverse(items, function(item) {
    return item.isValid;
});

// Process HTML for safe DOM injection (prevents duplicate JS/CSS)
var safeHtml = Granite.UI.Foundation.Utils.processHtml(rawHtml, ".selector", callback);
```

#### Foundation Vocabulary (Events & Patterns)

Granite UI uses a "vocabulary" system — CSS classes and data attributes that define behavior:

| Vocabulary | Purpose |
|------------|---------|
| `foundation-content-loaded` | Event fired when new content is injected into DOM |
| `foundation-contentloaded` | Event fired when dialog content finishes loading |
| `foundation-field-change` | Event fired when a form field value changes |
| `foundation-validation` | Validation API for form fields |
| `foundation-form-submitted` | Event fired after form AJAX submission |
| `foundation-selections` | Selection management API |
| `foundation-collection` | Collection/list management API |
| `foundation-toggleable` | Show/hide toggling |

#### AdaptTo API Pattern

Granite UI exposes APIs through an adapter pattern:

```javascript
// Get selection API for a collection element
var selectionAPI = $(myCollection).adaptTo("foundation-selections");
var count = selectionAPI.getCount();

// Get validation API
var validationAPI = $(myField).adaptTo("foundation-validation");
```

#### Foundation Registry

Register custom validators, adapters, and other extensions:

```javascript
$(window).adaptTo("foundation-registry").register(
    "foundation.validation.validator", {
    selector: "[data-myapp-maxlength]",
    validate: function(el) {
        var max = parseInt(el.getAttribute("data-myapp-maxlength"), 10);
        if (isNaN(max)) return;
        if (el.value.length > max) {
            return "Maximum " + max + " characters allowed.";
        }
    }
});
```

### Foundation ClientLib Categories

| Category | Purpose |
|----------|---------|
| `coralui3` | Coral UI 3 library (CSS + JS) |
| `granite.ui.coral.foundation` | Granite UI Foundation for Coral 3 |
| `granite.ui.foundation` | Legacy Granite Foundation (includes Coral 3 compatibility) |
| `granite.ui.foundation.admin` | Admin console foundation |
| `cq.authoring.dialog` | Loaded for ALL component dialogs |
| `cq.authoring.editor.sites.page` | Page editor core |
| `cq.authoring.editor.sites.page.hook` | Hook category for page editor extensions |
| `cq.shared` | Shared namespace — **must be included on all content pages** |

### Coral 3 vs Coral 2 Key Differences

| Aspect | Coral 2 | Coral 3 |
|--------|---------|---------|
| **Architecture** | jQuery plugins + `data-*` attrs | Web Components Custom Elements |
| **Initialization** | `data-init="NumberInput"` | `<coral-numberinput>` auto-upgrades |
| **Properties** | Separate data object API | Direct JS getters/setters on element |
| **Clientlib** | `coralui2` | `coralui3` |
| **Granite clientlib** | `granite.ui.foundation` | `granite.ui.coral.foundation` |
| **Resource types** | `granite/ui/components/foundation/...` | `granite/ui/components/coral/foundation/...` |
| **Layout** | Separate layout node + container | Direct component (tabs, accordion) |
| **Configuration** | `layoutConfig` child node | `parentConfig` child node |
| **Validation** | `$.validator` (deprecated) | `foundation-validation` |

**Cloud Service**: Only Coral 3 / `granite/ui/components/coral/foundation/` resource types are supported. Never use Coral 2 resource types.

### Pitfalls

- Do NOT use Coral 2 resource types (`granite/ui/components/foundation/...`) in Cloud Service
- Do NOT use self-closing tags for Coral custom elements
- Do NOT access Coral component properties before `Coral.commons.ready()` in polyfilled browsers
- Do NOT mix `coralui2` and `coralui3` clientlibs on the same page
- Do NOT forget `cq.shared` clientlib category on content pages — causes JS errors
- Do NOT use `config` as the name for RTE configuration nodes — breaks for non-admin users
