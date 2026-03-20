---
title: Content Overlays & Sling Resource Merger
impact: HIGH
impactDescription: Overlays are the only supported way to customize Touch UI consoles, dialogs, and the page editor in Cloud Service
tags: overlays, sling-resource-merger, sling:hideResource, sling:orderBefore, apps, libs, customization
---

## Content Overlays & Sling Resource Merger

AEM Cloud Service uses overlays as the primary mechanism to customize built-in UI — consoles, dialogs, page editor, and admin interfaces. The Sling Resource Merger enables minimal diffs rather than full copies of `/libs` structures.

### How Overlays Work

AEM uses a search path that checks `/apps` before `/libs`. To customize a resource:

1. Mirror the `/libs` path structure under `/apps`
2. Define only the changes needed
3. The Sling Resource Merger merges your overlay with the original

```
Original:  /libs/cq/gui/components/authoring/dialog/...
Overlay:   /apps/cq/gui/components/authoring/dialog/...
```

The `/apps` version takes priority. Unchanged elements inherit from `/libs` automatically.

### Sling Resource Merger Properties

| Property | Type | Purpose |
|----------|------|---------|
| `sling:hideResource` | Boolean | Completely hide a resource and all its children |
| `sling:hideChildren` | String or String[] | Hide specific child nodes. Supports wildcard `*` |
| `sling:hideProperties` | String or String[] | Hide specific properties. Supports wildcard `*` |
| `sling:orderBefore` | String | Position this node before the named sibling |

### Overlay vs Override

| Approach | Mechanism | Mount Point | Use Case |
|----------|-----------|-------------|----------|
| **Overlay** | Search path (`/apps` before `/libs`) | `/mnt/overlay` | Customizing built-in UI (consoles, editor, dialogs) |
| **Override** | `sling:resourceSuperType` inheritance | `/mnt/override` | Extending component dialogs via supertype chain |

#### Accessing Merged Resources Programmatically

```java
// Overlay: get merged resource via mount point
Resource merged = resolver.getResource("/mnt/overlay" + relativePath);

// Override: get merged resource via mount point
Resource merged = resolver.getResource("/mnt/override" + absolutePath);
```

### Common Overlay Patterns

#### Hide a Node

Remove a tab, field, or section from a dialog or console:

```xml
<!-- /apps/cq/gui/components/authoring/dialog/tabs/items/advanced -->
<advanced
    jcr:primaryType="nt:unstructured"
    sling:hideResource="{Boolean}true"/>
```

#### Hide Specific Children

```xml
<!-- Hide specific child nodes -->
<items
    jcr:primaryType="nt:unstructured"
    sling:hideChildren="[child1,child2]"/>

<!-- Hide ALL children -->
<items
    jcr:primaryType="nt:unstructured"
    sling:hideChildren="*"/>
```

#### Hide Properties

```xml
<!-- Hide specific properties -->
<myNode
    jcr:primaryType="nt:unstructured"
    sling:hideProperties="[jcr:title,jcr:description]"/>

<!-- Hide ALL properties -->
<myNode
    jcr:primaryType="nt:unstructured"
    sling:hideProperties="*"/>
```

#### Reorder Nodes

```xml
<!-- Position "myTab" before "advancedTab" -->
<myTab
    jcr:primaryType="nt:unstructured"
    jcr:title="My Tab"
    sling:orderBefore="advancedTab"
    sling:resourceType="granite/ui/components/coral/foundation/container"/>
```

#### Add a New Property/Node

Create the corresponding node in `/apps` and add the new property or child:

```xml
<!-- /apps/path/to/overlay/myNewField -->
<myNewField
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
    fieldLabel="My New Field"
    name="./myNewField"/>
```

#### Redefine a Property Value

Create the matching node and set the property with the new value:

```xml
<!-- /apps/path/to/overlay/existingField -->
<existingField
    jcr:primaryType="nt:unstructured"
    fieldLabel="Updated Label"/>
```

### Practical Examples

#### Overlay a Console Column

Add a custom column to the Sites console:

```
/apps/cq/gui/content/
    siteadmin/
        admin/
            listview/
                columns/
                    customColumn (nt:unstructured)
                        jcr:title = "Custom Column"
                        ...
```

#### Overlay Page Properties Dialog

Add a custom tab to page properties:

```
/apps/core/wcm/components/page/v3/page/
    _cq_dialog/
        content/
            items/
                tabs/
                    items/
                        customTab (nt:unstructured)
                            jcr:title = "Custom Properties"
                            sling:resourceType = "granite/ui/components/coral/foundation/container"
                            sling:orderBefore = "..."
                            items/
                                columns/
                                    ...fields...
```

#### Hide a Tab from Core Component Dialog

```xml
<!-- Hide the "Styles" tab from core teaser dialog -->
<!-- /apps/core/wcm/components/teaser/v2/teaser/_cq_dialog/content/items/tabs/items/styletab -->
<styletab
    jcr:primaryType="nt:unstructured"
    sling:hideResource="{Boolean}true"/>
```

### Extending Core Component Dialogs

When extending Core Components, use the `sling:resourceSuperType` approach (override) rather than overlaying `/libs`:

```xml
<!-- /apps/myproject/components/teaser/.content.xml -->
<jcr:root
    jcr:primaryType="cq:Component"
    jcr:title="Teaser"
    sling:resourceSuperType="core/wcm/components/teaser/v2/teaser"
    componentGroup="My Project"/>
```

Then in the dialog, use Sling Resource Merger properties to modify the inherited dialog:

```xml
<!-- /apps/myproject/components/teaser/_cq_dialog/.content.xml -->
<!-- Only define what you're changing; everything else inherits -->
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="Teaser"
    sling:resourceType="cq/gui/components/authoring/dialog">
    <content jcr:primaryType="nt:unstructured">
        <items jcr:primaryType="nt:unstructured">
            <tabs jcr:primaryType="nt:unstructured">
                <items jcr:primaryType="nt:unstructured">
                    <!-- Hide a parent tab -->
                    <actions
                        jcr:primaryType="nt:unstructured"
                        sling:hideResource="{Boolean}true"/>
                    <!-- Add a new tab -->
                    <customTab
                        jcr:primaryType="nt:unstructured"
                        jcr:title="Custom"
                        sling:resourceType="granite/ui/components/coral/foundation/container"
                        sling:orderBefore="linkAndActions">
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
                                            <myField
                                                jcr:primaryType="nt:unstructured"
                                                sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                                                fieldLabel="Custom Field"
                                                name="./customField"/>
                                        </items>
                                    </column>
                                </items>
                            </columns>
                        </items>
                    </customTab>
                </items>
            </tabs>
        </items>
    </content>
</jcr:root>
```

### Best Practices

- **Minimal structure**: Only recreate the skeleton path needed to reach your changes
- **Use `nt:unstructured`** for intermediary nodes — no need to match original node types
- **Never modify `/libs`** — changes are overwritten on upgrades
- **All customizations under `/apps`** — this is mandatory on Cloud Service
- **Prefer Sling Resource Merger properties** over copying entire node trees
- **Test overlays after SDK updates** — `/libs` changes may affect your overlays

### Pitfalls

- Do NOT copy entire `/libs` structures to `/apps` — only replicate the minimum path to your change
- Do NOT modify structures below tab-item level when customizing Core Components — hide parent tabs and add new ones instead
- Do NOT use overlays for component rendering (HTL/Sling Models) — use `sling:resourceSuperType` instead
- Sling Resource Merger only works with Touch UI (the sole UI in Cloud Service)
- `sling:hideResource` hides the resource AND all its children — there is no way to hide a resource but keep its children
- `sling:orderBefore` value must be the exact node name of an existing sibling
- Intermediary nodes in overlay paths must exist even if they have no changes — they form the skeleton path to your actual modification
