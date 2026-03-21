---
title: Component Dialog Development
impact: HIGH
impactDescription: Dialog quality directly affects author experience and content accuracy — incorrect patterns cause broken fields, confusing UX, and content entry errors
tags: dialog, granite-ui, coral, xml, multifield, authoring, validation, tabs, fields, design-dialog
---

## Component Dialog Development

AEM component dialogs use Granite UI (Coral 3) resource types. Dialogs are defined as XML in `_cq_dialog/.content.xml` and provide the author editing experience for components.

---

### 1. Dialog Types

| Type | Location | Purpose |
|------|----------|---------|
| `_cq_dialog` | Edit dialog | Author content editing (per-instance) |
| `_cq_design_dialog` | Design/Policy dialog | Template-level configuration |
| `_cq_editConfig` | Edit config | Drag-and-drop, inplace editing, toolbar |

---

### 2. Granite UI Field Types (Coral 3)

Always use `granite/ui/components/coral/foundation/...`:

#### Text Input Fields

```xml
<!-- Single-line text -->
<title
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="Title"
    fieldDescription="The component title"
    name="./jcr:title"
    required="{Boolean}true"
    maxlength="{Long}80"
    emptyText="Enter a title"/>

<!-- Multi-line text -->
<description
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textarea"
    fieldLabel="Description"
    fieldDescription="A brief description (max 200 chars)"
    name="./description"
    maxlength="{Long}200"
    rows="{Long}3"/>

<!-- Number field -->
<maxItems
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/numberfield"
    fieldLabel="Maximum Items"
    fieldDescription="The maximum number of items to display"
    name="./maxItems"
    min="{Long}1"
    max="{Long}100"
    step="{Long}1"
    value="{Long}10"/>
```

#### Selection Fields

```xml
<!-- Dropdown select -->
<alignment
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Alignment"
    name="./alignment">
    <items jcr:primaryType="nt:unstructured">
        <left jcr:primaryType="nt:unstructured" text="Left" value="left" selected="{Boolean}true"/>
        <center jcr:primaryType="nt:unstructured" text="Center" value="center"/>
        <right jcr:primaryType="nt:unstructured" text="Right" value="right"/>
    </items>
</alignment>

<!-- Checkbox (boolean) -->
<hideOnMobile
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
    fieldDescription="Hide this component on mobile devices"
    name="./hideOnMobile"
    text="Hide on Mobile"
    value="{Boolean}true"
    uncheckedValue="{Boolean}false"/>

<!-- Radio group -->
<variant
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/radiogroup"
    fieldLabel="Variant"
    name="./variant"
    vertical="{Boolean}true">
    <items jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" text="Default" value="default" checked="{Boolean}true"/>
        <compact jcr:primaryType="nt:unstructured" text="Compact" value="compact"/>
        <expanded jcr:primaryType="nt:unstructured" text="Expanded" value="expanded"/>
    </items>
</variant>
```

#### Picker Fields

```xml
<!-- Path picker -->
<linkURL
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
    fieldLabel="Link URL"
    fieldDescription="Internal page or asset path"
    name="./linkURL"
    rootPath="/content"
    filter="hierarchyNotFile"/>

<!-- Date picker -->
<publishDate
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/datepicker"
    fieldLabel="Publish Date"
    name="./publishDate"
    type="datetime"
    displayedFormat="YYYY-MM-DD HH:mm"/>

<!-- Color picker -->
<backgroundColor
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/colorfield"
    fieldLabel="Background Color"
    name="./backgroundColor"
    showSwatches="{Boolean}true"
    showProperties="{Boolean}false">
    <items jcr:primaryType="nt:unstructured">
        <white jcr:primaryType="nt:unstructured" value="#ffffff"/>
        <light jcr:primaryType="nt:unstructured" value="#f5f5f5"/>
        <dark jcr:primaryType="nt:unstructured" value="#333333"/>
        <brand jcr:primaryType="nt:unstructured" value="#0066cc"/>
    </items>
</backgroundColor>

<!-- Tag picker -->
<tags
    jcr:primaryType="nt:unstructured"
    sling:resourceType="cq/gui/components/common/admin/tagspicker"
    fieldLabel="Tags"
    name="./cq:tags"
    rootPath="/content/cq:tags/myproject"
    multiple="{Boolean}true"/>
```

#### Hidden and Include Fields

```xml
<!-- Hidden field (store metadata) -->
<componentType
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/hidden"
    name="./componentType"
    value="hero"/>

<!-- Include another dialog fragment -->
<shared
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/include"
    path="/apps/myproject/components/shared/dialog-fields/link-settings"/>
```

---

### 3. Dialog XML Template (Full Structure)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:granite="http://www.adobe.com/jcr/granite/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="Hero Component"
    sling:resourceType="cq/gui/components/authoring/dialog"
    extraClientlibs="[myproject.authoring]"
    helpPath="https://docs.example.com/hero">
    <content
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/container">
        <items jcr:primaryType="nt:unstructured">
            <tabs
                jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/tabs"
                maximized="{Boolean}true">
                <items jcr:primaryType="nt:unstructured">
                    <!-- Tab 1: Content -->
                    <content
                        jcr:primaryType="nt:unstructured"
                        jcr:title="Content"
                        sling:resourceType="granite/ui/components/coral/foundation/container"
                        margin="{Boolean}true">
                        <items jcr:primaryType="nt:unstructured">
                            <columns
                                jcr:primaryType="nt:unstructured"
                                sling:resourceType="granite/ui/components/coral/foundation/fixedcolumns"
                                margin="{Boolean}true">
                                <items jcr:primaryType="nt:unstructured">
                                    <column
                                        jcr:primaryType="nt:unstructured"
                                        sling:resourceType="granite/ui/components/coral/foundation/container">
                                        <items jcr:primaryType="nt:unstructured">
                                            <!-- Fields here -->
                                            <title
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                                                fieldLabel="Title"
                                                name="./jcr:title"
                                                required="{Boolean}true"/>
                                            <text
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="cq/gui/components/authoring/dialog/richtext"
                                                fieldLabel="Description"
                                                name="./text"
                                                useFixedInlineToolbar="{Boolean}true"/>
                                        </items>
                                    </column>
                                </items>
                            </columns>
                        </items>
                    </content>
                    <!-- Tab 2: Settings -->
                    <settings
                        jcr:primaryType="nt:unstructured"
                        jcr:title="Settings"
                        sling:resourceType="granite/ui/components/coral/foundation/container"
                        margin="{Boolean}true">
                        <items jcr:primaryType="nt:unstructured">
                            <columns
                                jcr:primaryType="nt:unstructured"
                                sling:resourceType="granite/ui/components/coral/foundation/fixedcolumns"
                                margin="{Boolean}true">
                                <items jcr:primaryType="nt:unstructured">
                                    <column
                                        jcr:primaryType="nt:unstructured"
                                        sling:resourceType="granite/ui/components/coral/foundation/container">
                                        <items jcr:primaryType="nt:unstructured">
                                            <variant
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/select"
                                                fieldLabel="Variant"
                                                name="./variant">
                                                <items jcr:primaryType="nt:unstructured">
                                                    <default jcr:primaryType="nt:unstructured" text="Default" value="default"/>
                                                    <fullWidth jcr:primaryType="nt:unstructured" text="Full Width" value="fullWidth"/>
                                                </items>
                                            </variant>
                                        </items>
                                    </column>
                                </items>
                            </columns>
                        </items>
                    </settings>
                </items>
            </tabs>
        </items>
    </content>
</jcr:root>
```

---

### 4. Multifield Patterns

#### Simple Multifield (Single Value)

```xml
<tags
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
    fieldLabel="Tags"
    fieldDescription="Add keyword tags">
    <field
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
        name="./tags"/>
</tags>
```

#### Composite Multifield (Multiple Values Per Row)

```xml
<links
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
    fieldLabel="Links"
    fieldDescription="Add navigation links"
    composite="{Boolean}true">
    <field
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/container"
        name="./links">
        <items jcr:primaryType="nt:unstructured">
            <linkURL
                jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
                fieldLabel="URL"
                name="linkURL"
                rootPath="/content"/>
            <linkText
                jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                fieldLabel="Text"
                name="linkText"/>
            <openInNewTab
                jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
                text="Open in new tab"
                name="openInNewTab"
                value="{Boolean}true"/>
        </items>
    </field>
</links>
```

**Sling Model for composite multifield:**

```java
@ChildResource(name = "links")
private List<Resource> links;

public List<LinkItem> getLinks() {
    if (links == null) return Collections.emptyList();
    return links.stream().map(r -> {
        ValueMap vm = r.getValueMap();
        return new LinkItem(
            vm.get("linkURL", ""),
            vm.get("linkText", ""),
            vm.get("openInNewTab", false)
        );
    }).collect(Collectors.toList());
}
```

---

### 5. Field Validation

```xml
<!-- Required field -->
<title name="./jcr:title" required="{Boolean}true"/>

<!-- Pattern validation (regex) -->
<email
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="Email"
    name="./email"
    validation="email"/>

<!-- Custom validation regex -->
<zipCode
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="ZIP Code"
    name="./zipCode"
    validation="^[0-9]{5}(-[0-9]{4})?$"/>

<!-- Min/max for numbers -->
<count
    sling:resourceType="granite/ui/components/coral/foundation/form/numberfield"
    name="./count"
    min="{Long}1"
    max="{Long}50"
    required="{Boolean}true"/>

<!-- Max length for text -->
<summary
    sling:resourceType="granite/ui/components/coral/foundation/form/textarea"
    name="./summary"
    maxlength="{Long}300"/>
```

---

### 6. Conditional Field Visibility (Show/Hide)

```xml
<!-- Trigger field -->
<variant
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Variant"
    name="./variant"
    granite:class="cq-dialog-dropdown-showhide"
    cq-dialog-dropdown-showhide-target=".variant-options">
    <items jcr:primaryType="nt:unstructured">
        <simple jcr:primaryType="nt:unstructured" text="Simple" value="simple"/>
        <advanced jcr:primaryType="nt:unstructured" text="Advanced" value="advanced"/>
    </items>
</variant>

<!-- Conditional field — shown only when variant = "advanced" -->
<advancedOptions
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/fieldset"
    jcr:title="Advanced Options"
    granite:class="hide variant-options"
    showhidetargetvalue="advanced">
    <items jcr:primaryType="nt:unstructured">
        <customCss
            jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
            fieldLabel="Custom CSS Class"
            name="./customCssClass"/>
    </items>
</advancedOptions>
```

**Requires the showhide clientlib:**

```xml
<!-- In the dialog root -->
extraClientlibs="[cq.authoring.dialog.all]"
```

---

### 7. Granite UI Layout Types

| Layout | Use Case |
|--------|----------|
| `fixedcolumns` | **Required** wrapper — prevents full-width field stretching |
| `tabs` | Multiple content sections |
| `accordion` | Collapsible sections within a tab |
| `wizard` | Multi-step configuration (rarely used) |
| `well` | Visual grouping with background |

---

### 8. Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Node names | lowerCamelCase | `sortOrder`, `maxItems` |
| Property names | lowerCamelCase | `fieldDescription`, `fieldLabel` |
| Field `name` property | `./lowerCamelCase` | `./jcr:title`, `./maxItems` |
| Tab titles | Title Case | "Content", "Settings", "Advanced" |
| Field labels | Title Case, descriptive | "Maximum Items" |
| Field descriptions | Sentence case, contextual | "The maximum number of items to display" |

---

### 9. Anti-Patterns

#### Using Coral 2 Resource Types

```xml
<!-- WRONG — Coral 2 (deprecated) -->
<field sling:resourceType="granite/ui/components/foundation/form/textfield"/>

<!-- CORRECT — Coral 3 -->
<field sling:resourceType="granite/ui/components/coral/foundation/form/textfield"/>
```

#### Missing Fixed Columns Wrapper

```xml
<!-- WRONG — fields stretch to full dialog width -->
<items>
    <title sling:resourceType=".../form/textfield" name="./title"/>
</items>

<!-- CORRECT — wrap in fixedcolumns -->
<columns sling:resourceType="granite/ui/components/coral/foundation/fixedcolumns" margin="{Boolean}true">
    <items>
        <column sling:resourceType="granite/ui/components/coral/foundation/container">
            <items>
                <title sling:resourceType=".../form/textfield" name="./title"/>
            </items>
        </column>
    </items>
</columns>
```

#### Missing Field Labels and Descriptions

```xml
<!-- WRONG — no context for authors -->
<field sling:resourceType=".../form/textfield" name="./title"/>

<!-- CORRECT — always provide label and description -->
<field sling:resourceType=".../form/textfield"
       fieldLabel="Title"
       fieldDescription="The main heading displayed above the content"
       name="./title"/>
```

#### Instructive Labels

```xml
<!-- WRONG — labels that instruct -->
<field fieldLabel="Select the alignment"/>
<field fieldLabel="Enter the title"/>
<field fieldLabel="Choose a color"/>

<!-- CORRECT — descriptive labels -->
<field fieldLabel="Alignment"/>
<field fieldLabel="Title"/>
<field fieldLabel="Background Color"/>
```
