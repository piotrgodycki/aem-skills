---
title: Touch UI Dialog ClientLib Patterns
impact: HIGH
impactDescription: Dialog JavaScript controls field visibility, validation, and dynamic behavior — critical for author-friendly dialogs
tags: dialog, clientlib, cq.authoring.dialog, showhide, validation, foundation-contentloaded, dialog-ready, coral-events
---

## Touch UI Dialog ClientLib Patterns

Custom dialog behavior — field show/hide, validation, dynamic options — is implemented via client libraries with specific authoring categories.

### Authoring ClientLib Categories

| Category | Loaded When | Use For |
|----------|-------------|---------|
| `cq.authoring.dialog` | Every component dialog opens | Dialog field customization, show/hide, validation |
| `cq.authoring.editor.sites.page` | Page editor loads | Core editor dependency |
| `cq.authoring.editor.sites.page.hook` | Page editor loads | Page editor extensions, custom actions |
| `cq.authoring.page` | Page authoring context | Page-level authoring customizations |

### Dialog ClientLib Structure

```
/apps/myproject/clientlibs/
└── clientlib-dialog/
    ├── .content.xml
    ├── js/
    │   └── dialog.js
    └── js.txt
```

```xml
<!-- .content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="cq:ClientLibraryFolder"
    categories="[cq.authoring.dialog]"
    allowProxy="{Boolean}true"/>
```

```
#base=js
dialog.js
```

### Dialog Lifecycle Events

#### dialog-ready Event

Fired when a dialog DOM is loaded and ready:

```javascript
(function($, Coral) {
    "use strict";

    $(document).on("dialog-ready", function() {
        // Dialog is loaded — fields are accessible
        var dialog = $(".cq-Dialog");
        // Initialize custom behavior
    });
})(jQuery, Coral);
```

#### foundation-contentloaded Event

Fired when new content is injected into the DOM (dialogs, panels). More general than `dialog-ready`:

```javascript
(function($) {
    "use strict";

    $(document).on("foundation-contentloaded", function(e) {
        // New content loaded — could be dialog, panel, or other content
        var container = e.target;
        // Initialize fields within the loaded container
    });
})(jQuery);
```

RTE inplace editing starts by default on `foundation-contentloaded`.

### Show/Hide Pattern

The most common dialog customization — show or hide fields based on another field's value.

#### Using Coral Select Change Event

```javascript
(function($, Coral) {
    "use strict";

    $(document).on("dialog-ready", function() {
        // Find the controlling select field
        var $select = $("[name='./layoutType']");
        if ($select.length === 0) return;

        // Initial state
        toggleFields($select);

        // Listen for changes — Coral Select fires native "change" event
        $select.on("change", function() {
            toggleFields($(this));
        });
    });

    function toggleFields($select) {
        var value = $select.val();
        var $dialog = $select.closest(".cq-Dialog");

        // Show/hide fields based on selection
        var $gridFields = $dialog.find(".grid-options");
        var $listFields = $dialog.find(".list-options");

        if (value === "grid") {
            $gridFields.show();
            $listFields.hide();
        } else {
            $gridFields.hide();
            $listFields.show();
        }
    }
})(jQuery, Coral);
```

#### Using granite:class for CSS Hooks

Add `granite:class` to dialog fields as hooks for JavaScript:

```xml
<!-- In dialog XML -->
<layoutType
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Layout Type"
    name="./layoutType"
    granite:class="myproject-layout-select">
    <items jcr:primaryType="nt:unstructured">
        <grid jcr:primaryType="nt:unstructured" text="Grid" value="grid"/>
        <list jcr:primaryType="nt:unstructured" text="List" value="list"/>
    </items>
</layoutType>
<columns
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/numberfield"
    fieldLabel="Columns"
    name="./columns"
    granite:class="myproject-grid-columns"
    wrapperClass="grid-options"/>
```

```javascript
(function($) {
    "use strict";

    $(document).on("dialog-ready", function() {
        var $select = $(".myproject-layout-select coral-select");
        if ($select.length === 0) return;

        handleVisibility($select);
        $select.on("change", function() {
            handleVisibility($(this));
        });
    });

    function handleVisibility($select) {
        var value = $select.val();
        $(".grid-options").toggle(value === "grid");
    }
})(jQuery);
```

### Checkbox Toggle Pattern

```javascript
(function($) {
    "use strict";

    $(document).on("dialog-ready", function() {
        var $checkbox = $("[name='./enableFeature']");
        if ($checkbox.length === 0) return;

        function toggle() {
            var checked = $checkbox.prop("checked") ||
                          $checkbox.attr("checked") === "true";
            $(".feature-options").toggle(checked);
        }

        toggle();
        // Coral Checkbox fires "change" event
        $checkbox.on("change", function() {
            toggle();
        });
    });
})(jQuery);
```

### Dialog Validation

#### Foundation Validation API

Granite UI uses `foundation-validation` which replicates HTML5 Constraint Validation API:

```javascript
// Register a custom validator
$(window).adaptTo("foundation-registry").register(
    "foundation.validation.validator", {
    // Selector matching fields to validate
    selector: "[data-validation='myproject-url']",
    // Return error message string on failure, undefined on success
    validate: function(el) {
        var value = el.value;
        if (!value) return; // skip empty (use required for that)

        try {
            new URL(value);
        } catch (e) {
            return "Please enter a valid URL.";
        }
    }
});
```

#### Custom Validation with Error Display

```javascript
$(window).adaptTo("foundation-registry").register(
    "foundation.validation.validator", {
    selector: "[data-validation='myproject-max-words']",
    validate: function(el) {
        var max = parseInt(el.getAttribute("data-max-words"), 10);
        if (isNaN(max)) return;
        var wordCount = el.value.trim().split(/\s+/).length;
        if (wordCount > max) {
            return "Maximum " + max + " words allowed. Current: " + wordCount;
        }
    },
    // Custom error UI (optional)
    show: function(el, message, ctx) {
        $(el).closest(".coral-Form-fieldwrapper")
            .find(".coral-Form-fielderror").remove();
        $(el).after('<coral-icon icon="alert" class="coral-Form-fielderror" ' +
            'title="' + message + '"></coral-icon>');
    },
    clear: function(el, ctx) {
        $(el).closest(".coral-Form-fieldwrapper")
            .find(".coral-Form-fielderror").remove();
    }
});
```

#### Using validation Property in Dialog XML

```xml
<urlField
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="URL"
    name="./url"
    validation="[myproject-url]"/>
```

Built-in validators include `foundation.jcr.name` (valid JCR node name).

#### Suppress Validation UI

```html
<input data-foundation-validation-ui="none" />
```

### Hide Conditions (Server-Side)

Use `granite:hide` property for server-side conditional field visibility based on component policies:

```xml
<!-- Hide field when template policy disables a feature -->
<childPagesOption
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Child Pages"
    name="./source"
    granite:hide="${cqDesign.disableChildren}"/>
```

Expression syntax:
- `${cqDesign.myProperty}` — Hide when property is truthy
- `${!cqDesign.myProperty}` — Hide when property is falsy
- `${cqDesign.prop1 == 'value'}` — Equality comparison
- `${cqDesign.prop1 && cqDesign.prop2}` — Boolean logic

The `cqDesign` variable reads from the component's content policy. Properties are set in the design dialog / policy dialog.

### Custom Granite UI Field Components

Create reusable custom fields with their own rendering and behavior:

1. Create component at `/apps/myproject/components/form/myfield/`
2. Set `sling:resourceType` extending a base Granite field
3. Add `init.jsp` for value initialization and `render.jsp` for rendering
4. Optionally add a clientlib for client-side behavior

```xml
<!-- Using a custom field in dialog -->
<myCustomField
    jcr:primaryType="nt:unstructured"
    sling:resourceType="myproject/components/form/myfield"
    fieldLabel="Custom Field"
    name="./customValue"/>
```

**Note**: Granite UI form field server-side rendering requires JSP. HTL is used for content components; dialog field rendering still uses JSP internally.

### Granite UI Form Field Resource Types

All under `granite/ui/components/coral/foundation/form/`:

| Resource Type | Purpose |
|---------------|---------|
| `textfield` | Single-line text input |
| `textarea` | Multi-line text input |
| `numberfield` | Numeric input |
| `checkbox` | Boolean checkbox |
| `radiogroup` | Radio button group |
| `select` | Dropdown select |
| `pathfield` | Path browser/picker |
| `datepicker` | Date/time picker |
| `colorfield` | Color picker |
| `fileupload` | File upload |
| `hidden` | Hidden field |
| `switch` | Toggle switch |
| `password` | Password input |
| `multifield` | Repeatable field group |
| `fieldset` | Field grouping |
| `userpicker` | User/group selector |
| `autocomplete` | Autocomplete input |
| `nestedcheckboxlist` | Hierarchical checkboxes |
| `buttongroup` | Grouped buttons |
| `advancedselect` | Advanced selection with status |
| `actionfield` | Field with action button |

### Pitfalls

- Do NOT use `$(document).ready()` for dialog code — dialogs load after page ready. Use `dialog-ready` or `foundation-contentloaded`
- Do NOT forget to check if fields exist before attaching listeners — dialog JS runs for ALL dialogs when using `cq.authoring.dialog` category
- Do NOT use complex CSS selectors for validation — use flat selectors like `[data-validation]` for performance
- `granite:class` is the correct way to add CSS hooks — not the `class` property
- `wrapperClass` adds classes to the field's wrapper div — useful for show/hide targeting
- Coral events (e.g., `change` on `coral-select`) are standard DOM events — use `addEventListener` or jQuery `.on()`
- Foundation validation validators run in LIFO order — last registered runs first
- Use `required="{Boolean}true"` on field for required validation — do not implement required checks in custom validators
