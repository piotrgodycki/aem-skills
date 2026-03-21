---
title: Advanced Dialog Field Patterns
impact: HIGH
impactDescription: Advanced field types enable rich authoring experiences for images, tags, fragments, colors, dates, and dynamic selections
tags: dialog, pathfield, fileupload, tagfield, colorpicker, datepicker, multifield-composite, autocomplete, datasource, rendercondition, experience-fragment, content-fragment
---

## Advanced Dialog Field Patterns

Beyond basic text, select, and checkbox fields, AEM provides specialized Granite UI components for complex authoring scenarios. This guide covers advanced field configurations for AEM Cloud Service.

### Pathfield Configuration

Use `granite/ui/components/coral/foundation/form/pathfield` (NOT the deprecated `pathbrowser`):

```xml
<pagePath
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
    fieldLabel="Page Path"
    fieldDescription="Select a content page"
    name="./pagePath"
    rootPath="/content/mysite"
    filter="hierarchyNotFile"
    emptyText="Select a page"/>
```

**Key pathfield properties:**

| Property | Type | Description |
|----------|------|-------------|
| `rootPath` | String | Root path for browsing (e.g., `/content/mysite`) |
| `filter` | String | Filter type: `hierarchy`, `hierarchyNotFile`, `nosystem` |
| `emptyText` | String | Placeholder text |
| `droppable` | String | CSS selector for drop target (e.g., `[name='./pagePath']`) |
| `multiple` | Boolean | Allow multiple path selection |
| `forceSelection` | Boolean | Require selection from browser (no manual typing) |
| `pickerSrc` | String | Custom picker source URL |
| `suggestionSrc` | String | Custom suggestion source URL |
| `deleteHint` | Boolean | Send delete hint when field is cleared (default: true) |

#### Pagefield (CQ-specific)

For page-only selection, use the CQ pagefield which restricts to `cq:Page` nodes:

```xml
<linkURL
    jcr:primaryType="nt:unstructured"
    sling:resourceType="cq/gui/components/coral/common/form/pagefield"
    fieldLabel="Link"
    fieldDescription="Select a content page"
    name="./linkURL"
    rootPath="/content"/>
```

### Image / File Upload Fields

The `cq/gui/components/authoring/dialog/fileupload` resource type provides drag-and-drop image handling:

```xml
<file
    jcr:primaryType="nt:unstructured"
    sling:resourceType="cq/gui/components/authoring/dialog/fileupload"
    class="cq-droptarget"
    fieldLabel="Image Asset"
    fieldDescription="Drag an image from the asset finder or click to browse"
    fileNameParameter="./fileName"
    fileReferenceParameter="./fileReference"
    mimeTypes="[image/gif,image/jpeg,image/png,image/tiff,image/svg+xml]"
    name="./file"
    title="Upload Image"
    allowUpload="{Boolean}false"
    uploadUrl="${suffix.path}"
    autoStart="{Boolean}false"
    useHTML5="{Boolean}true"/>
```

**Critical fileupload properties:**

| Property | Type | Description |
|----------|------|-------------|
| `class` | String | Set to `cq-droptarget` to enable drag-and-drop from asset finder |
| `fileNameParameter` | String | Property to store filename (e.g., `./fileName`) |
| `fileReferenceParameter` | String | Property to store DAM reference (e.g., `./fileReference`) |
| `mimeTypes` | String[] | Allowed MIME types array |
| `allowUpload` | Boolean | Allow direct upload (`false` = DAM reference only) |
| `uploadUrl` | String | Upload endpoint URL |
| `autoStart` | Boolean | Auto-start upload on file selection |
| `useHTML5` | Boolean | Use HTML5 file upload |
| `sizeLimit` | Long | Maximum file size in bytes |

**Pattern: Image field that only accepts DAM references (no upload):**

```xml
<image
    jcr:primaryType="nt:unstructured"
    sling:resourceType="cq/gui/components/authoring/dialog/fileupload"
    allowUpload="{Boolean}false"
    class="cq-droptarget"
    fileNameParameter="./fileName"
    fileReferenceParameter="./fileReference"
    mimeTypes="[image/gif,image/jpeg,image/png,image/svg+xml]"
    name="./file"
    title="Drag image here"/>
```

### Tag Picker Field

Use the dedicated tagfield component for AEM tag selection:

```xml
<tags
    jcr:primaryType="nt:unstructured"
    sling:resourceType="cq/gui/components/coral/common/form/tagfield"
    fieldLabel="Tags"
    fieldDescription="Select content tags"
    name="./cq:tags"
    multiple="{Boolean}true"
    rootPath="/content/cq:tags/mysite"/>
```

**Alternate autocomplete-based tag pattern** (for older compatibility):

```xml
<tags
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/autocomplete"
    fieldLabel="Tags"
    name="./cq:tags"
    multiple="{Boolean}true"
    emptyText="Enter tags">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="cq/gui/components/common/datasources/tags"/>
    <values
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/foundation/form/autocomplete/tags"/>
    <options
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/foundation/form/autocomplete/list"/>
</tags>
```

### Experience Fragment Picker

Use the dedicated XF picker field from core components:

```xml
<fragmentVariationPath
    jcr:primaryType="nt:unstructured"
    sling:resourceType="cq/experience-fragments/editor/components/xffield"
    fieldLabel="Experience Fragment"
    fieldDescription="Select an experience fragment variation"
    name="./fragmentVariationPath"
    filter="folderOrVariant"
    propertyFilter="cq:xfShowInEditor"
    variant="web"
    emptyText="Select an experience fragment"/>
```

**XF field properties:**

| Property | Type | Description |
|----------|------|-------------|
| `filter` | String | `folderOrVariant` to show folders and variations |
| `propertyFilter` | String | `cq:xfShowInEditor` to filter visible XFs |
| `variant` | String | `web`, `email`, `social`, etc. |

Load the XF validator clientlib in the dialog root:

```xml
<jcr:root ...
    extraClientlibs="[cq.xf.editor.picker.validator]">
```

### Content Fragment Picker

Use the CF picker component:

```xml
<fragmentPath
    jcr:primaryType="nt:unstructured"
    sling:resourceType="dam/cfm/components/cfpicker"
    fieldLabel="Content Fragment"
    fieldDescription="Select a content fragment"
    name="./fragmentPath"
    rootPath="/content/dam/mysite/fragments"/>
```

Populate element and variation selects dynamically:

```xml
<elementNames
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Element"
    name="./elementNames"
    emptyText="Select element">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="dam/cfm/components/datasources/elements"/>
</elementNames>

<variationName
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Variation"
    name="./variationName"
    emptyText="Master">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="dam/cfm/components/datasources/variations"/>
</variationName>
```

### Color Picker

Use `granite/ui/components/coral/foundation/form/colorfield`:

```xml
<color
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/colorfield"
    fieldLabel="Background Color"
    fieldDescription="Select or enter a color value"
    name="./backgroundColor"
    value="#ffffff"
    showSwatches="{Boolean}true"
    showProperties="{Boolean}true"
    showDefaultColors="{Boolean}true"
    autogenerateColors="shades"/>
```

**Color field with predefined swatches:**

```xml
<brandColor
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/colorfield"
    fieldLabel="Brand Color"
    name="./brandColor"
    showSwatches="{Boolean}true"
    showProperties="{Boolean}false"
    showDefaultColors="{Boolean}false"
    autogenerateColors="off">
    <items jcr:primaryType="nt:unstructured">
        <color1
            jcr:primaryType="nt:unstructured"
            value="#003366"/>
        <color2
            jcr:primaryType="nt:unstructured"
            value="#0066CC"/>
        <color3
            jcr:primaryType="nt:unstructured"
            value="#FF6600"/>
        <color4
            jcr:primaryType="nt:unstructured"
            value="#333333"/>
    </items>
</brandColor>
```

**Colorfield properties:**

| Property | Type | Description |
|----------|------|-------------|
| `showSwatches` | Boolean | Show color swatch palette |
| `showProperties` | Boolean | Show color property inputs (hex/RGB) |
| `showDefaultColors` | Boolean | Show system default colors |
| `autogenerateColors` | String | `off`, `shades`, `tints` — auto-generate palette |

### Date/Time Picker with Constraints

```xml
<eventDate
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/datepicker"
    fieldLabel="Event Date"
    fieldDescription="Select event date and time"
    name="./eventDate"
    type="datetime"
    displayedFormat="YYYY-MM-DD HH:mm"
    valueFormat="YYYY-MM-DDTHH:mm:ss.000Z"
    minDate="2024-01-01T00:00:00.000"
    maxDate="2030-12-31T23:59:59.000"
    displayTimezoneMessage="{Boolean}true"
    typeHint="Date"
    required="{Boolean}true"/>
```

**Datepicker type values:**

| `type` Value | Behavior |
|-------------|----------|
| `date` | Date only |
| `datetime` | Date and time |
| `time` | Time only |

**Key properties:**

| Property | Type | Description |
|----------|------|-------------|
| `displayedFormat` | String | Display format (e.g., `YYYY-MM-DD`, `DD/MM/YYYY`) |
| `valueFormat` | String | Storage format |
| `minDate` | String | Minimum selectable date (ISO 8601) |
| `maxDate` | String | Maximum selectable date (ISO 8601) |
| `displayTimezoneMessage` | Boolean | Show timezone info |
| `typeHint` | String | Set to `Date` for proper JCR date storage |

### Composite Multifield (Nested Fields)

Composite multifield stores child items as sub-nodes with multiple properties:

```xml
<slides
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
    composite="{Boolean}true"
    fieldLabel="Slides">
    <field
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/container"
        name="./slides">
        <items jcr:primaryType="nt:unstructured">
            <image
                jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
                fieldLabel="Image"
                name="./image"
                rootPath="/content/dam"
                filter="hierarchyNotFile"/>
            <title
                jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                fieldLabel="Title"
                name="./title"
                required="{Boolean}true"/>
            <description
                jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/form/textarea"
                fieldLabel="Description"
                name="./description"/>
            <linkURL
                jcr:primaryType="nt:unstructured"
                sling:resourceType="cq/gui/components/coral/common/form/pagefield"
                fieldLabel="Link"
                name="./linkURL"
                rootPath="/content"/>
        </items>
    </field>
</slides>
```

**Critical composite multifield rules:**

- Set `composite="{Boolean}true"` on the multifield node
- The `field` child must be a `container` with `name` matching the property path (e.g., `./slides`)
- Child field names use relative paths (e.g., `./title` NOT `./slides/title`)
- Data is stored as child nodes: `slides/item0/title`, `slides/item1/title`, etc.
- Access in HTL with `${resource.children}` iteration or Sling Model `@ChildResource`

**Sling Model for composite multifield:**

```java
@Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class CarouselModel {

    @ChildResource
    private List<SlideItem> slides;

    @Model(adaptables = Resource.class, defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
    public interface SlideItem {
        @ValueMapValue
        String getTitle();

        @ValueMapValue
        String getDescription();

        @ValueMapValue
        String getImage();

        @ValueMapValue
        String getLinkURL();
    }
}
```

### Autocomplete / Suggest Field

```xml
<suggestion
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/autocomplete"
    fieldLabel="Search Term"
    name="./searchTerm"
    multiple="{Boolean}false"
    icon="search"
    emptyText="Type to search...">
    <options
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/form/autocomplete/list"
        src="/apps/myproject/datasources/suggestions{.offset,limit}.html{+value}"/>
</autocomplete>
```

The `src` property uses a Sling servlet or resource that returns JSON options for typeahead suggestions.

### Select with Dynamic Options (DataSource)

Instead of hardcoding `<items>` in select fields, use a `datasource` child node to populate options dynamically:

```xml
<country
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Country"
    name="./country"
    emptyText="Select a country">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="myproject/components/datasources/countries"/>
</country>
```

See the companion rule file `dialog-datasources.md` for full DataSource servlet implementation details.

### Conditional Field Rendering with granite:rendercondition

Render conditions evaluate server-side whether a field or tab should be rendered at all (not just hidden — completely excluded from the HTML):

```xml
<adminOnlyField
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="Admin Override"
    name="./adminOverride">
    <granite:rendercondition
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/renderconditions/privilege"
        path="${requestPathInfo.suffix}"
        privileges="[rep:write,crx:replicate]"/>
</adminOnlyField>
```

**Built-in render condition types:**

| Resource Type | Purpose |
|---------------|---------|
| `.../renderconditions/privilege` | Check user JCR privileges |
| `.../renderconditions/simple` | Evaluate an EL expression |
| `.../renderconditions/or` | Logical OR of child conditions |
| `.../renderconditions/and` | Logical AND of child conditions |
| `.../renderconditions/not` | Negate a child condition |

**EL expression-based condition:**

```xml
<granite:rendercondition
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/renderconditions/simple"
    expression="${requestPathInfo.suffix != '/content/dam'}"/>
```

**Custom render condition** (Sling servlet):

```java
@Component(service = Servlet.class, property = {
    "sling.servlet.resourceTypes=myproject/renderconditions/hasfeature",
    "sling.servlet.methods=GET"
})
public class HasFeatureRenderCondition extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
        Config cfg = new Config(request.getResource());
        String featureFlag = cfg.get("feature", String.class);

        boolean conditionMet = checkFeature(featureFlag);

        request.setAttribute(
            "com.adobe.granite.ui.components.rendercondition.RenderCondition",
            new SimpleRenderCondition(conditionMet)
        );
    }
}
```

Usage in dialog:

```xml
<granite:rendercondition
    jcr:primaryType="nt:unstructured"
    sling:resourceType="myproject/renderconditions/hasfeature"
    feature="dark-mode"/>
```

### Field Wrapper Patterns

#### Well (Visual Grouping)

```xml
<settingsWell
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/well">
    <items jcr:primaryType="nt:unstructured">
        <heading
            jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/heading"
            text="Advanced Settings"
            level="{Long}3"/>
        <field1 .../>
        <field2 .../>
    </items>
</settingsWell>
```

#### Fieldset

```xml
<contactInfo
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/fieldset"
    jcr:title="Contact Information">
    <items jcr:primaryType="nt:unstructured">
        <name .../>
        <email .../>
        <phone .../>
    </items>
</contactInfo>
```

#### Accordion Within Dialogs

```xml
<accordion
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/accordion"
    variant="default"
    margin="{Boolean}true">
    <items jcr:primaryType="nt:unstructured">
        <section1
            jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/container"
            jcr:title="Basic Settings">
            <items jcr:primaryType="nt:unstructured">
                <field1 .../>
            </items>
        </section1>
        <section2
            jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/container"
            jcr:title="Advanced Settings">
            <items jcr:primaryType="nt:unstructured">
                <field2 .../>
            </items>
        </section2>
    </items>
</accordion>
```

### Tab Navigation with Icons

Add icons to tabs using `granite:class` and Coral icon attributes:

```xml
<tabs
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/tabs"
    maximized="{Boolean}true">
    <items jcr:primaryType="nt:unstructured">
        <properties
            jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/container"
            jcr:title="Properties"
            granite:icon="properties"
            margin="{Boolean}true">
            <items jcr:primaryType="nt:unstructured">
                <!-- fields -->
            </items>
        </properties>
        <styles
            jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/container"
            jcr:title="Styles"
            granite:icon="brush"
            margin="{Boolean}true">
            <items jcr:primaryType="nt:unstructured">
                <!-- fields -->
            </items>
        </styles>
    </items>
</tabs>
```

Common Coral icons for tabs: `properties`, `brush`, `gears`, `image`, `link`, `text`, `code`, `globe`, `visibility`, `data`.

### Responsive Dialog Layouts

Use `granite/ui/components/coral/foundation/fixedcolumns` with column widths for multi-column layouts:

```xml
<columns
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/fixedcolumns"
    margin="{Boolean}true">
    <items jcr:primaryType="nt:unstructured">
        <column1
            jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/container">
            <items jcr:primaryType="nt:unstructured">
                <leftField .../>
            </items>
        </column1>
        <column2
            jcr:primaryType="nt:unstructured"
            sling:resourceType="granite/ui/components/coral/foundation/container">
            <items jcr:primaryType="nt:unstructured">
                <rightField .../>
            </items>
        </column2>
    </items>
</columns>
```

### Asset Reference with Rendition Selection

Combine a pathfield for DAM asset selection with a select for rendition:

```xml
<assetRef
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
    fieldLabel="Asset"
    name="./fileReference"
    rootPath="/content/dam"
    filter="hierarchyNotFile"
    emptyText="Select a DAM asset"/>

<rendition
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Rendition"
    name="./rendition"
    emptyText="Original">
    <items jcr:primaryType="nt:unstructured">
        <original
            jcr:primaryType="nt:unstructured"
            text="Original"
            value="original"/>
        <web
            jcr:primaryType="nt:unstructured"
            text="Web Optimized"
            value="cq5dam.web.1280.1280"/>
        <thumbnail
            jcr:primaryType="nt:unstructured"
            text="Thumbnail"
            value="cq5dam.thumbnail.319.319"/>
    </items>
</rendition>
```

For dynamic rendition lists, use a datasource servlet that reads available renditions from the selected DAM asset.

### Complete Advanced Dialog Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:granite="http://www.adobe.com/jcr/granite/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="Advanced Card Component"
    sling:resourceType="cq/gui/components/authoring/dialog"
    extraClientlibs="[myproject.dialog.advanced]">
    <content
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/container">
        <items jcr:primaryType="nt:unstructured">
            <tabs
                jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/tabs"
                maximized="{Boolean}true">
                <items jcr:primaryType="nt:unstructured">
                    <content
                        jcr:primaryType="nt:unstructured"
                        sling:resourceType="granite/ui/components/coral/foundation/container"
                        jcr:title="Content"
                        granite:icon="text"
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
                                            <image
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="cq/gui/components/authoring/dialog/fileupload"
                                                allowUpload="{Boolean}false"
                                                class="cq-droptarget"
                                                fieldLabel="Card Image"
                                                fileNameParameter="./fileName"
                                                fileReferenceParameter="./fileReference"
                                                mimeTypes="[image/jpeg,image/png,image/svg+xml]"
                                                name="./file"/>
                                            <tags
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="cq/gui/components/coral/common/form/tagfield"
                                                fieldLabel="Tags"
                                                name="./cq:tags"
                                                multiple="{Boolean}true"/>
                                            <publishDate
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/datepicker"
                                                fieldLabel="Publish Date"
                                                name="./publishDate"
                                                type="datetime"
                                                displayedFormat="YYYY-MM-DD HH:mm"
                                                typeHint="Date"/>
                                            <bgColor
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/colorfield"
                                                fieldLabel="Background Color"
                                                name="./bgColor"
                                                showSwatches="{Boolean}true"
                                                showDefaultColors="{Boolean}false">
                                                <items jcr:primaryType="nt:unstructured">
                                                    <white jcr:primaryType="nt:unstructured" value="#FFFFFF"/>
                                                    <light jcr:primaryType="nt:unstructured" value="#F5F5F5"/>
                                                    <dark jcr:primaryType="nt:unstructured" value="#333333"/>
                                                </items>
                                            </bgColor>
                                        </items>
                                    </column>
                                </items>
                            </columns>
                        </items>
                    </content>
                    <links
                        jcr:primaryType="nt:unstructured"
                        sling:resourceType="granite/ui/components/coral/foundation/container"
                        jcr:title="Links"
                        granite:icon="link"
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
                                            <actions
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/multifield"
                                                composite="{Boolean}true"
                                                fieldLabel="Call-to-Action Links">
                                                <field
                                                    jcr:primaryType="nt:unstructured"
                                                    sling:resourceType="granite/ui/components/coral/foundation/container"
                                                    name="./actions">
                                                    <items jcr:primaryType="nt:unstructured">
                                                        <link
                                                            jcr:primaryType="nt:unstructured"
                                                            sling:resourceType="cq/gui/components/coral/common/form/pagefield"
                                                            fieldLabel="Link"
                                                            name="./link"
                                                            rootPath="/content"
                                                            required="{Boolean}true"/>
                                                        <text
                                                            jcr:primaryType="nt:unstructured"
                                                            sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                                                            fieldLabel="Label"
                                                            name="./text"
                                                            required="{Boolean}true"/>
                                                    </items>
                                                </field>
                                            </actions>
                                        </items>
                                    </column>
                                </items>
                            </columns>
                        </items>
                    </links>
                </items>
            </tabs>
        </items>
    </content>
</jcr:root>
```
