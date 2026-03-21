---
title: Custom Dialog Widget and Field Development
impact: HIGH
impactDescription: Custom widgets extend authoring capabilities beyond out-of-the-box Granite UI fields — must be built correctly for stability and maintainability
tags: dialog, custom-widget, granite-ui, coral-ui, clientlib, validation, dialog-events, content-transformer, content-policy, sling-resourceSuperType
---

## Custom Dialog Widget and Field Development

When standard Granite UI fields are insufficient, you can create custom field components, extend existing ones, hook into dialog lifecycle events, and implement custom validation and save logic.

### Creating a Custom Granite UI Field Component

A custom field is a Sling component with a `sling:resourceType`, rendered via JSP or HTL, that outputs Coral UI markup.

#### Component Structure

```
/apps/myproject/components/form/
└── colorswatchfield/
    ├── .content.xml            (component definition)
    ├── render.jsp              (field rendering)
    └── clientlibs/
        ├── .content.xml        (clientlib definition)
        ├── js.txt
        ├── css.txt
        ├── js/
        │   └── colorswatchfield.js
        └── css/
            └── colorswatchfield.css
```

#### Component Definition (.content.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="sling:Folder"
    sling:resourceSuperType="granite/ui/components/coral/foundation/form/field"
    jcr:title="Color Swatch Field"
    componentGroup=".hidden"/>
```

The `sling:resourceSuperType` of `granite/ui/components/coral/foundation/form/field` provides the base field rendering framework.

#### JSP Rendering (render.jsp)

```jsp
<%@include file="/libs/granite/ui/global.jsp" %>
<%@page session="false"
    import="org.apache.commons.lang3.StringUtils,
            com.adobe.granite.ui.components.AttrBuilder,
            com.adobe.granite.ui.components.Config,
            com.adobe.granite.ui.components.Field,
            com.adobe.granite.ui.components.Tag" %>
<%

    Config cfg = cmp.getConfig();
    ValueMap vm = (ValueMap) request.getAttribute(Field.class.getName());
    String value = vm.get("value", String.class);

    Tag tag = cmp.consumeTag();
    AttrBuilder attrs = tag.getAttrs();
    cmp.populateCommonAttrs(attrs);

    String name = cfg.get("name", String.class);
    boolean required = cfg.get("required", false);
    String[] swatches = cfg.get("swatches", String[].class);

    attrs.add("name", name);
    attrs.add("value", value);
    attrs.addClass("myproject-colorswatchfield");

    if (required) {
        attrs.add("aria-required", "true");
    }

    String validations = StringUtils.join(cfg.get("validation", new String[0]), " ");
    attrs.add("data-validation", validations);
%>
<div class="myproject-colorswatchfield-wrapper">
    <input type="hidden" <%= attrs.build() %>>
    <div class="myproject-swatches" role="radiogroup">
        <% if (swatches != null) {
            for (String swatch : swatches) {
                String selected = swatch.equals(value) ? " is-selected" : "";
        %>
        <button type="button"
                class="myproject-swatch<%= selected %>"
                data-value="<%= xssAPI.encodeForHTMLAttr(swatch) %>"
                style="background-color: <%= xssAPI.encodeForHTMLAttr(swatch) %>"
                role="radio"
                aria-checked="<%= swatch.equals(value) %>">
        </button>
        <% }} %>
    </div>
</div>
```

**Key rendering concepts:**
- `cmp.getConfig()` reads field properties from the dialog XML node
- `request.getAttribute(Field.class.getName())` retrieves the stored value
- `cmp.consumeTag()` and `cmp.populateCommonAttrs()` handle standard Granite field attributes
- `xssAPI` provides XSS-safe encoding for output
- The hidden `<input>` stores the actual value for form submission

#### HTL Alternative (render.html)

For HTL-based custom fields, use a companion Sling Model:

```html
<sly data-sly-use.field="com.myproject.core.models.ColorSwatchField"/>
<div class="myproject-colorswatchfield-wrapper">
    <input type="hidden"
           name="${field.name}"
           value="${field.storedValue}"
           class="myproject-colorswatchfield"
           data-validation="${field.validation}"/>
    <div class="myproject-swatches" role="radiogroup">
        <sly data-sly-list.swatch="${field.swatches}">
            <button type="button"
                    class="myproject-swatch ${swatch == field.storedValue ? 'is-selected' : ''}"
                    data-value="${swatch}"
                    style="background-color: ${swatch}"
                    role="radio"
                    aria-checked="${swatch == field.storedValue}">
            </button>
        </sly>
    </div>
</div>
```

```java
@Model(adaptables = SlingHttpServletRequest.class)
public class ColorSwatchField {

    @Self
    private SlingHttpServletRequest request;

    private Config config;
    private String storedValue;

    @PostConstruct
    protected void init() {
        config = new Config(request.getResource());
        ValueMap vm = (ValueMap) request.getAttribute(Field.class.getName());
        storedValue = vm != null ? vm.get("value", String.class) : null;
    }

    public String getName() {
        return config.get("name", String.class);
    }

    public String getStoredValue() {
        return storedValue;
    }

    public String[] getSwatches() {
        return config.get("swatches", new String[0]);
    }

    public String getValidation() {
        String[] validations = config.get("validation", new String[0]);
        return String.join(" ", validations);
    }
}
```

#### Using the Custom Field in a Dialog

```xml
<brandColor
    jcr:primaryType="nt:unstructured"
    sling:resourceType="myproject/components/form/colorswatchfield"
    fieldLabel="Brand Color"
    name="./brandColor"
    required="{Boolean}true"
    swatches="[#003366,#0066CC,#FF6600,#333333,#FFFFFF]"/>
```

### Registering Custom Field ClientLibs

Custom field behavior and styling require a dedicated clientlib:

```xml
<!-- /apps/myproject/components/form/colorswatchfield/clientlibs/.content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="cq:ClientLibraryFolder"
    categories="[cq.authoring.dialog]"
    allowProxy="{Boolean}true"/>
```

**ClientLib category choices:**

| Category | When Loaded | Best For |
|----------|-------------|----------|
| `cq.authoring.dialog` | Every dialog opens | Widely reused custom fields |
| `cq.authoring.dialog.all` | All dialogs | Global dialog extensions |
| Component-specific (via `extraClientlibs`) | Only when that component's dialog opens | Heavy or single-use fields |

For component-specific loading, set `extraClientlibs` on the dialog root:

```xml
<jcr:root ...
    sling:resourceType="cq/gui/components/authoring/dialog"
    extraClientlibs="[myproject.colorswatchfield]">
```

### Custom Field Validation

#### Client-Side Validation

Register a validator using the `foundation-validation` API:

```javascript
(function($, Coral) {
    "use strict";

    // Register a named validator
    var registry = $(window).adaptTo("foundation-registry");

    registry.register("foundation.validation.validator", {
        // Matches fields with data-validation="myproject.hexcolor"
        selector: "[data-validation='myproject.hexcolor']",
        validate: function(el) {
            var value = el.value || $(el).val();
            if (value && !/^#([0-9A-Fa-f]{3}|[0-9A-Fa-f]{6})$/.test(value)) {
                return Granite.I18n.get("Please enter a valid hex color (e.g., #FF6600)");
            }
            return null; // null = valid
        }
    });

    // Validator with async check
    registry.register("foundation.validation.validator", {
        selector: "[data-validation='myproject.uniqueslug']",
        validate: function(el) {
            var value = $(el).val();
            if (!value) return null;

            var result = null;
            $.ajax({
                url: "/bin/myproject/check-slug",
                data: { slug: value },
                async: false, // synchronous for validation
                success: function(data) {
                    if (!data.available) {
                        result = Granite.I18n.get("This slug is already in use");
                    }
                }
            });
            return result;
        }
    });

})(jQuery, Coral);
```

**Usage in dialog XML:**

```xml
<slug
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="URL Slug"
    name="./slug"
    validation="myproject.uniqueslug"/>
```

#### Server-Side Validation

Implement a `SlingPostProcessor` for server-side validation on save:

```java
@Component(service = SlingPostProcessor.class)
public class DialogFieldValidator implements SlingPostProcessor {

    @Override
    public void process(SlingHttpServletRequest request, List<Modification> changes) throws Exception {
        String resourceType = request.getParameter("sling:resourceType");

        // Only process specific component types
        if (!"myproject/components/card".equals(resourceType)) {
            return;
        }

        String slug = request.getParameter("./slug");
        if (slug != null && !slug.matches("^[a-z0-9-]+$")) {
            throw new PersistenceException("Slug must contain only lowercase letters, numbers, and hyphens");
        }
    }
}
```

### Wrapping Coral UI Components

Extend an existing Coral 3 component by overlaying its Granite UI resource type:

```xml
<!-- /apps/myproject/components/form/enhancedselect/.content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="sling:Folder"
    sling:resourceSuperType="granite/ui/components/coral/foundation/form/select"
    jcr:title="Enhanced Select"/>
```

The `sling:resourceSuperType` inherits all behavior from the base select. Override only the rendering script you need:

```jsp
<%@include file="/libs/granite/ui/global.jsp" %>
<%-- Delegate to parent rendering first --%>
<sling:include resourceType="granite/ui/components/coral/foundation/form/select"/>
<%-- Add custom wrapper markup or data attributes --%>
<script>
    // Enhance the last rendered coral-select
    Coral.commons.ready(function() {
        var select = document.querySelector("coral-select[name='${resource.valueMap['name']}']");
        if (select) {
            select.classList.add("myproject-enhanced-select");
        }
    });
</script>
```

### Extending Existing Granite UI Fields

To add behavior to a standard field without creating a completely new component, use the `sling:resourceSuperType` pattern:

```
/apps/myproject/components/form/
└── pathfieldwithpreview/
    ├── .content.xml
    │   (sling:resourceSuperType = granite/ui/components/coral/foundation/form/pathfield)
    └── clientlibs/
        └── js/
            └── pathfieldpreview.js   (adds preview on path selection)
```

```javascript
// pathfieldpreview.js
(function($, Coral) {
    "use strict";

    $(document).on("dialog-ready", function() {
        $(".myproject-pathfield-preview").each(function() {
            var $wrapper = $(this);
            var $pathfield = $wrapper.find("foundation-autocomplete");

            $pathfield.on("change", function() {
                var path = $(this).val();
                if (path) {
                    var previewUrl = path + ".thumb.319.319.png";
                    $wrapper.find(".preview-image").attr("src", previewUrl).show();
                }
            });
        });
    });

})(jQuery, Coral);
```

### Dialog Event Lifecycle

AEM Touch UI dialogs fire specific events during their lifecycle. Hook into these in your clientlib:

```javascript
(function($, Coral) {
    "use strict";

    // 1. DIALOG-READY: Fired when dialog DOM is loaded and interactive
    //    Use for: field initialization, binding event handlers, DOM manipulation
    $(document).on("dialog-ready", function(event) {
        var $dialog = $(event.target).closest(".cq-Dialog");
        if (!$dialog.length) return;

        // Initialize custom fields
        initializeCustomFields($dialog);
    });

    // 2. FOUNDATION-CONTENTLOADED: Fired when new content is injected into DOM
    //    Use for: re-initializing fields after dynamic content loads (e.g., multifield add)
    $(document).on("foundation-contentloaded", function(event) {
        var container = event.target;
        initializeCustomFields($(container));
    });

    // 3. CORAL-OVERLAY:OPEN: Fired when a Coral overlay (dialog) opens
    //    Use for: early access before dialog-ready, animation hooks
    $(document).on("coral-overlay:open", ".cq-Dialog", function(event) {
        // Dialog overlay is now visible
    });

    // 4. CORAL-OVERLAY:CLOSE: Fired when dialog closes
    //    Use for: cleanup, removing event handlers, resetting state
    $(document).on("coral-overlay:close", ".cq-Dialog", function(event) {
        cleanupCustomFields($(this));
    });

    // 5. SUBMIT / PRE-SAVE: Intercept form submission
    //    Use for: transforming values, additional validation, blocking save
    $(document).on("submit", ".cq-Dialog form", function(event) {
        var $form = $(this);
        var isValid = performCustomValidation($form);
        if (!isValid) {
            event.preventDefault();
            event.stopPropagation();
            return false;
        }
        // Transform values before save
        transformFieldValues($form);
    });

    function initializeCustomFields($container) {
        // Find and initialize custom fields within the container
        $container.find(".myproject-custom-field").each(function() {
            // Initialization logic
        });
    }

    function cleanupCustomFields($dialog) {
        // Remove event handlers, clear intervals, etc.
    }

    function performCustomValidation($form) {
        // Return true if valid, false to block save
        return true;
    }

    function transformFieldValues($form) {
        // Modify hidden fields, combine values, etc.
    }

})(jQuery, Coral);
```

**Event firing order on dialog open:**
1. `coral-overlay:open` (overlay becomes visible)
2. `foundation-contentloaded` (content loaded into DOM)
3. `dialog-ready` (fields are interactive)

**Event firing order on dialog save:**
1. Form `submit` event
2. `coral-overlay:close` (on success)

### Custom Dialog Actions (Save Hooks)

#### Pre-Save Transformation

Transform dialog values before they are persisted to JCR:

```javascript
(function($) {
    "use strict";

    $(document).on("dialog-ready", function() {
        var $form = $(".cq-Dialog form");

        // Intercept the Granite UI form submission
        $form.off("submit.custom").on("submit.custom", function(e) {
            // Combine first + last name into a computed field
            var firstName = $form.find("[name='./firstName']").val();
            var lastName = $form.find("[name='./lastName']").val();
            var $fullName = $form.find("[name='./fullName']");
            if (!$fullName.length) {
                $fullName = $("<input>", {
                    type: "hidden",
                    name: "./fullName"
                }).appendTo($form);
            }
            $fullName.val(firstName + " " + lastName);

            // Slugify a title into a URL-safe value
            var title = $form.find("[name='./jcr:title']").val();
            var $slug = $form.find("[name='./slug']");
            if ($slug.length && !$slug.val()) {
                $slug.val(title.toLowerCase().replace(/[^a-z0-9]+/g, "-"));
            }
        });
    });

})(jQuery);
```

#### Post-Save Processing (Server-Side)

Use an `EventHandler` or `ResourceChangeListener` to process saved dialog data:

```java
@Component(
    service = EventHandler.class,
    property = {
        EventConstants.EVENT_TOPIC + "=org/apache/sling/api/resource/Resource/CHANGED",
        EventConstants.EVENT_FILTER + "=(path=/content/mysite/*)"
    }
)
public class DialogPostSaveHandler implements EventHandler {

    @Override
    public void handleEvent(Event event) {
        String path = (String) event.getProperty("path");
        // Process the saved component data
    }
}
```

### Content Transformer Patterns

Transform dialog values between authoring and storage using a `SlingPostProcessor`:

```java
@Component(service = SlingPostProcessor.class)
public class DialogContentTransformer implements SlingPostProcessor {

    private static final Logger LOG = LoggerFactory.getLogger(DialogContentTransformer.class);

    @Override
    public void process(SlingHttpServletRequest request, List<Modification> changes) throws Exception {
        // Check if this is a component dialog save
        String contentPath = request.getResource().getPath();
        if (!contentPath.startsWith("/content/")) {
            return;
        }

        // Transform markdown to HTML before saving
        String description = request.getParameter("./description");
        if (description != null && description.contains("**")) {
            ResourceResolver resolver = request.getResourceResolver();
            Resource resource = resolver.getResource(contentPath);
            if (resource != null) {
                ModifiableValueMap props = resource.adaptTo(ModifiableValueMap.class);
                if (props != null) {
                    String html = convertMarkdownToHtml(description);
                    props.put("descriptionHtml", html);
                    changes.add(Modification.onModified(contentPath + "/descriptionHtml"));
                }
            }
        }
    }

    private String convertMarkdownToHtml(String markdown) {
        // Simple bold conversion; use a library for full markdown
        return markdown.replaceAll("\\*\\*(.*?)\\*\\*", "<strong>$1</strong>");
    }
}
```

### Custom Content Policy Fields

Content policies (design dialogs) follow the same Granite UI patterns but are configured in `_cq_design_dialog`:

```xml
<!-- _cq_design_dialog/.content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="Card Policy"
    sling:resourceType="cq/gui/components/authoring/dialog">
    <content jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/container">
        <items jcr:primaryType="nt:unstructured">
            <tabs jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/tabs"
                maximized="{Boolean}true">
                <items jcr:primaryType="nt:unstructured">
                    <properties
                        jcr:primaryType="nt:unstructured"
                        sling:resourceType="granite/ui/components/coral/foundation/container"
                        jcr:title="Policy Settings"
                        margin="{Boolean}true">
                        <items jcr:primaryType="nt:unstructured">
                            <columns
                                jcr:primaryType="nt:unstructured"
                                sling:resourceType="granite/ui/components/coral/foundation/fixedcolumns">
                                <items jcr:primaryType="nt:unstructured">
                                    <column
                                        jcr:primaryType="nt:unstructured"
                                        sling:resourceType="granite/ui/components/coral/foundation/container">
                                        <items jcr:primaryType="nt:unstructured">
                                            <allowedColors
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
                                                composite="{Boolean}true"
                                                fieldLabel="Allowed Brand Colors">
                                                <field
                                                    jcr:primaryType="nt:unstructured"
                                                    sling:resourceType="granite/ui/components/coral/foundation/container"
                                                    name="./allowedColors">
                                                    <items jcr:primaryType="nt:unstructured">
                                                        <label
                                                            jcr:primaryType="nt:unstructured"
                                                            sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                                                            fieldLabel="Label"
                                                            name="./label"/>
                                                        <value
                                                            jcr:primaryType="nt:unstructured"
                                                            sling:resourceType="granite/ui/components/coral/foundation/form/colorfield"
                                                            fieldLabel="Color"
                                                            name="./value"/>
                                                    </items>
                                                </field>
                                            </allowedColors>
                                            <maxItems
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/numberfield"
                                                fieldLabel="Maximum Cards"
                                                fieldDescription="Limit number of cards displayed"
                                                name="./maxItems"
                                                min="{Long}1"
                                                max="{Long}20"
                                                step="{Long}1"/>
                                            <disableLinks
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
                                                text="Disable link fields in edit dialog"
                                                name="./disableLinks"
                                                value="{Boolean}true"/>
                                        </items>
                                    </column>
                                </items>
                            </columns>
                        </items>
                    </properties>
                    <!-- Include the standard style system tab -->
                    <styletab
                        jcr:primaryType="nt:unstructured"
                        sling:resourceType="granite/ui/components/coral/foundation/include"
                        path="/mnt/overlay/cq/gui/components/authoring/dialog/style/tab_design/styletab"/>
                </items>
            </tabs>
        </items>
    </content>
</jcr:root>
```

**Reading policy values in a Sling Model:**

```java
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class CardComponent {

    @ScriptVariable
    private Style currentStyle;

    public int getMaxItems() {
        return currentStyle.get("maxItems", 10);
    }

    public boolean isLinksDisabled() {
        return currentStyle.get("disableLinks", false);
    }
}
```

### Dialog Extension Points Summary

| Extension Point | Mechanism | Use Case |
|----------------|-----------|----------|
| Custom field component | `sling:resourceType` + JSP/HTL | New field types not in Granite UI |
| Field override | `sling:resourceSuperType` | Extend existing field behavior |
| Client-side validation | `foundation-registry` validator | Input format enforcement |
| Server-side validation | `SlingPostProcessor` | Data integrity checks |
| Pre-save transform | Form `submit` event handler | Compute/combine values |
| Post-save processing | `EventHandler` / `ResourceChangeListener` | Side effects (indexing, notifications) |
| Conditional rendering | `granite:rendercondition` | Role-based or context-based field visibility |
| Dynamic options | DataSource servlet | Database/API-driven select options |
| Policy fields | `_cq_design_dialog` | Template-level constraints |
| Dialog-specific clientlib | `extraClientlibs` property | Heavy JS only for one component |
