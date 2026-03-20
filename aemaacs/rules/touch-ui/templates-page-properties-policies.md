---
title: Editable Templates, Page Properties & Component Policies
impact: HIGH
impactDescription: Templates and policies control what authors can do per page type — misconfiguration breaks page creation and component editing
tags: templates, page-properties, policies, design-dialog, cq:design_dialog, template-editor, content-policies
---

## Editable Templates, Page Properties & Component Policies

AEM Cloud Service uses editable templates with content policies to control page structure, allowed components, and component pre-configuration. This replaces the Classic UI design mode.

### Template JCR Structure

```
/conf/<myproject>/settings/wcm/templates/
└── <template-name>/
    ├── jcr:content                    # Template metadata
    │   ├── jcr:title                  # Template title
    │   └── status                     # draft | enabled | disabled
    ├── structure/                     # Fixed page structure
    │   └── jcr:content/
    │       └── root/                  # Root layout container
    │           └── <components>/      # Locked structural components
    ├── initial/                       # Initial content for new pages
    │   └── jcr:content/
    │       └── root/
    │           └── <components>/      # Pre-populated components
    ├── policies/                      # Policy mappings
    │   └── jcr:content/
    │       └── root/
    │           └── cq:policy          # Reference to policy definition
    └── thumbnail.png                  # Template thumbnail
```

### Template States

| State | Behavior |
|-------|----------|
| `draft` | Not available to authors for page creation |
| `enabled` | Available in page creation wizard |
| `disabled` | Hidden from page creation |

### Structure vs Initial Content

- **Structure**: Defines the fixed page skeleton. Components marked as locked cannot be moved/deleted by authors. Components marked as unlocked (`editable=true`) can be edited.
- **Initial Content**: Pre-populated content that appears when a new page is created. Changes to initial content do NOT propagate to existing pages.

### Template Types

Template types are blueprints for creating new templates. Stored at:
- `/libs/settings/wcm/template-types` (out of the box)
- `/apps/settings/wcm/template-types` (custom)
- `/conf/<myproject>/settings/wcm/template-types` (project-specific)

Template types are copied (not referenced) when creating a template. The only remaining connection is a static reference for informational purposes.

### Allowing Templates on Pages

Control which templates are available under specific content paths:

```xml
<!-- On a page's jcr:content node -->
<jcr:content
    jcr:primaryType="cq:PageContent"
    cq:allowedTemplates="[/conf/myproject/settings/wcm/templates/.*]"/>
```

Template availability is evaluated in order:
1. `cq:allowedTemplates` on ancestor pages (ascending hierarchy)
2. `allowedPaths` on the template itself
3. Application path matching (second-level path)
4. `allowedParents` on the template
5. `allowedChildren` on the parent page's template

### Content Policies (Replacing Design Mode)

Content policies define component configuration at the template level. They replace Classic UI's "design mode" and `cq:designPath`.

#### Policy Storage

```
/conf/<myproject>/settings/wcm/policies/
└── wcm/
    └── foundation/
        └── components/
            └── responsivegrid/
                └── <policy-name>/
                    ├── jcr:title        # Policy name
                    ├── components       # Allowed component groups/paths
                    └── ...properties    # Component-specific policy values
```

#### Policy Mapping

Policies are mapped to components in the template's `policies` node:

```
/conf/<myproject>/settings/wcm/templates/<template>/
    policies/
        jcr:content/
            root/
                cq:policy = "wcm/foundation/components/responsivegrid/default"
                <component-name>/
                    cq:policy = "myproject/components/title/default"
```

#### Policy vs Design Dialog

| Aspect | Design Dialog (cq:design_dialog) | Content Policy |
|--------|-----------------------------------|----------------|
| **UI Location** | Template editor "policy" icon | Template editor "policy" icon |
| **Storage** | `/conf/.../settings/wcm/policies/` | Same |
| **Definition** | `_cq_design_dialog/.content.xml` | Same |
| **Purpose** | Pre-configure component for template | Same |
| **Replaces** | Classic UI design mode | Same |

The `cq:design_dialog` is the Touch UI mechanism for defining what appears in the policy dialog in the template editor. It uses the same Granite UI resource types as `cq:dialog`.

### Design Dialog Example

```xml
<!-- _cq_design_dialog/.content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="List Policy"
    sling:resourceType="cq/gui/components/authoring/dialog">
    <content
        jcr:primaryType="nt:unstructured"
        sling:resourceType="granite/ui/components/coral/foundation/container">
        <items jcr:primaryType="nt:unstructured">
            <tabs
                jcr:primaryType="nt:unstructured"
                sling:resourceType="granite/ui/components/coral/foundation/tabs"
                maximized="{Boolean}true">
                <items jcr:primaryType="nt:unstructured">
                    <properties
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
                                            <disableChildren
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
                                                fieldDescription="Disable child pages option for authors"
                                                name="./disableChildren"
                                                text="Disable Child Pages"
                                                value="{Boolean}true"/>
                                            <maxItems
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/numberfield"
                                                fieldLabel="Maximum Items"
                                                fieldDescription="Maximum number of items to display"
                                                name="./maxItems"
                                                min="{Long}1"
                                                max="{Long}100"/>
                                        </items>
                                    </column>
                                </items>
                            </columns>
                        </items>
                    </properties>
                    <!-- Include Style System tab -->
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

### Reading Policy Values in Sling Models

```java
@Model(adaptables = SlingHttpServletRequest.class)
public class MyComponentModel {

    @ScriptVariable
    private Style currentStyle;

    public int getMaxItems() {
        return currentStyle.get("maxItems", 10); // 10 = default
    }

    public boolean isChildPagesDisabled() {
        return currentStyle.get("disableChildren", false);
    }
}
```

### Style System Tab

Include the Style System tab in design dialogs for CSS class configuration:

```xml
<styletab
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/include"
    path="/mnt/overlay/cq/gui/components/authoring/dialog/style/tab_design/styletab"/>
```

This allows template authors to define CSS classes that content authors can apply to component instances.

### Page Properties Dialog

Page properties are defined in the page component's `cq:dialog`. The dialog inherits through `sling:resourceSuperType`:

```
Core Page Component: /libs/core/wcm/components/page/v3/page
  └── _cq_dialog/.content.xml  (base tabs: Basic, Social Media, Cloud Services, etc.)

Your Page Component: /apps/myproject/components/page
  └── sling:resourceSuperType = "core/wcm/components/page/v3/page"
  └── _cq_dialog/.content.xml  (extend/modify inherited tabs)
```

#### Adding a Custom Tab to Page Properties

```xml
<!-- /apps/myproject/components/page/_cq_dialog/.content.xml -->
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="Page Properties"
    sling:resourceType="cq/gui/components/authoring/dialog">
    <content jcr:primaryType="nt:unstructured">
        <items jcr:primaryType="nt:unstructured">
            <tabs jcr:primaryType="nt:unstructured">
                <items jcr:primaryType="nt:unstructured">
                    <analytics
                        jcr:primaryType="nt:unstructured"
                        jcr:title="Analytics"
                        sling:resourceType="granite/ui/components/coral/foundation/container"
                        sling:orderBefore="cloudservices"
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
                                            <trackingId
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                                                fieldLabel="Tracking ID"
                                                fieldDescription="Analytics tracking identifier for this page"
                                                name="./analyticsTrackingId"/>
                                        </items>
                                    </column>
                                </items>
                            </columns>
                        </items>
                    </analytics>
                </items>
            </tabs>
        </items>
    </content>
</jcr:root>
```

The three structures (core page, supertype chain, custom page) are merged into a single virtual resource at render time.

### Performance Considerations

- Keep fewer than 100 templates per site — Adobe recommends never exceeding 1000 for performance
- All content pages must include the `cq.shared` clientlib namespace — missing it causes JavaScript errors
- Avoid internationalizable content in templates — use Core Components localization features
- Template changes (structure/policies) propagate to existing pages; initial content changes do not

### Pitfalls

- Do NOT modify structures below tab-item level when customizing Core Component dialogs via Sling Resource Merger — hide parent tabs with `sling:hideResource` and add new tabs instead
- Do NOT expect initial content changes to update existing pages — only new pages get initial content
- Do NOT confuse `cq:dialog` (edit dialog for authors) with `cq:design_dialog` (policy dialog for template authors)
- Policy values are read via `currentStyle` (Style API) in Sling Models, NOT from the component resource
- All pages must include `cq.shared` — failing to do so produces JavaScript errors in the editor
- Template types are copied, not referenced — changing a template type does NOT update templates already created from it
- `cq:allowedTemplates` uses regex patterns, not exact paths — a missing `.*` suffix limits template selection
