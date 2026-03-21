---
title: cq:editConfig and Edit Behavior Configuration
impact: HIGH
impactDescription: Edit behavior controls component authoring interactions — drop targets, inline editing, toolbar actions, and event listeners
tags: editConfig, cq:listeners, cq:dropTargets, cq:inplaceEditing, cq:actionConfigs, cq:htmlTag, toolbar, drag-drop
---

## cq:editConfig and Edit Behavior Configuration

The `cq:editConfig` node (type `cq:EditConfig`) controls how a component behaves in the page editor — what toolbar actions appear, how drag-and-drop works, what happens after editing, and whether inline editing is available.

### Node Structure

```
<mycomponent> (cq:Component)
├── _cq_editConfig/.content.xml     (cq:EditConfig)
│   ├── cq:dropTargets/             (nt:unstructured)
│   │   └── <target>/               (cq:DropTargetConfig)
│   ├── cq:actionConfigs/           (nt:unstructured)
│   │   └── <action>/               (nt:unstructured)
│   ├── cq:formParameters           (nt:unstructured)
│   ├── cq:inplaceEditing           (cq:InplaceEditingConfig)
│   └── cq:listeners                (cq:EditListenersConfig)
├── _cq_childEditConfig/.content.xml (cq:EditConfig — for child components)
├── _cq_htmlTag/.content.xml        (nt:unstructured)
```

### cq:editConfig Properties

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    cq:actions="[edit,-,delete,insert,copymove]"
    cq:emptyText="Click to configure this component"
    cq:inherit="{Boolean}false"
    jcr:primaryType="cq:EditConfig">
</jcr:root>
```

| Property | Type | Description |
|----------|------|-------------|
| `cq:actions` | String[] | Toolbar actions: `edit`, `editannotate`, `delete`, `insert`, `copymove` |
| `cq:emptyText` | String | Placeholder text when component has no visible content |
| `cq:inherit` | Boolean | Whether missing values inherit from parent component (default: `false`) |
| `dialogLayout` | String | `fullscreen` to open dialog in fullscreen mode by default |

#### cq:actions Values

| Action | Description |
|--------|-------------|
| `edit` | Edit button — opens the component dialog |
| `editannotate` | Edit with annotation capability |
| `delete` | Delete component from page |
| `insert` | Insert new component before current |
| `copymove` | Copy and cut operations |
| `-` | Spacer separator |

### cq:dropTargets — Drag-and-Drop from Asset Finder

Configure what assets authors can drag onto the component from the side panel asset browser.

```xml
<cq:dropTargets jcr:primaryType="nt:unstructured">
    <image
        jcr:primaryType="cq:DropTargetConfig"
        accept="[image/gif,image/jpeg,image/png,image/webp,image/tiff,image/svg\\+xml]"
        groups="[media]"
        propertyName="./fileReference"/>
</cq:dropTargets>
```

| Property | Type | Description |
|----------|------|-------------|
| `accept` | String[] | MIME type regex patterns for accepted assets |
| `groups` | String[] | Asset finder groups to match (`media`, `product`, etc.) |
| `propertyName` | String | Component property to store the asset reference |

**Touch UI limitation**: Only the first drop target is used. Multiple drop targets work only in Classic UI.

### cq:inplaceEditing — Inline Editing

Enables editing directly on the component without opening a dialog.

```xml
<cq:inplaceEditing
    jcr:primaryType="cq:InplaceEditingConfig"
    active="{Boolean}true"
    editorType="text">
    <config jcr:primaryType="nt:unstructured">
        <rtePlugins jcr:primaryType="nt:unstructured">
            <format jcr:primaryType="nt:unstructured"
                features="bold,italic"/>
        </rtePlugins>
    </config>
</cq:inplaceEditing>
```

| Property | Type | Description |
|----------|------|-------------|
| `active` | Boolean | Enable/disable inplace editing |
| `editorType` | String | `text` (RTE), `plaintext`, `title`, or `image` |
| `configPath` | String | Path to editor configuration (alternative to nested `config` node) |

**Important**: Do NOT name the configuration child node `config` for RTE inplace editing — name it something else. Using `config` as the node name causes the RTE configuration to work only for administrators, not for `content-author` group members.

### cq:listeners — Edit Event Handlers

Control what happens after authoring actions (edit, insert, delete, move).

```xml
<cq:listeners
    jcr:primaryType="cq:EditListenersConfig"
    afteredit="REFRESH_PAGE"
    afterinsert="REFRESH_PAGE"
    afterdelete="REFRESH_PAGE"
    aftermove="REFRESH_PAGE"
    afterchildinsert="REFRESH_PAGE"/>
```

#### Available Events

| Event | When Fired | Default Handler |
|-------|------------|-----------------|
| `beforedelete` | Before component removal | — |
| `beforeedit` | Before editing starts | — |
| `beforecopy` | Before copying | — |
| `beforemove` | Before moving | — |
| `beforeinsert` | Before insertion (Touch UI only) | — |
| `beforechildinsert` | Before child insertion (containers only) | — |
| `afterdelete` | After component removal | `REFRESH_SELF` |
| `afteredit` | After editing completes | `REFRESH_SELF` |
| `aftercopy` | After copying | `REFRESH_SELF` |
| `afterinsert` | After insertion | `REFRESH_INSERTED` |
| `aftermove` | After moving | `REFRESH_SELFMOVED` |
| `afterchildinsert` | After child insertion (containers only) | — |

#### Handler Values

| Value | Behavior |
|-------|----------|
| `REFRESH_SELF` | Refresh only the current component |
| `REFRESH_PAGE` | Refresh the entire page |
| `REFRESH_INSERTED` | Refresh the inserted component |
| `REFRESH_SELFMOVED` | Refresh the moved component |
| Custom function | `function(path, definition) { ... }` |

**Critical rule for nested components**: `aftermove` and `aftercopy` **must** be set to `REFRESH_PAGE` for nested components. Using `REFRESH_SELF` causes rendering issues.

**Common pattern**: Use `REFRESH_PAGE` when the component affects other parts of the page (navigation, headers, sidebars) or when server-side logic generates content that depends on sibling/parent components.

### cq:actionConfigs — Custom Toolbar Buttons

Add custom actions to the component toolbar:

```xml
<cq:actionConfigs jcr:primaryType="nt:unstructured">
    <separator0
        jcr:primaryType="nt:unstructured"
        xtype="tbseparator"/>
    <manage
        jcr:primaryType="nt:unstructured"
        handler="function(){/* custom action */}"
        text="Custom Action"/>
</cq:actionConfigs>
```

**Note**: `cq:actionConfigs` is primarily for Classic UI. In Touch UI, component toolbar customizations are typically done through clientlib hooks.

### cq:formParameters — Hidden Default Values

Set default values as hidden form parameters when the dialog is submitted:

```xml
<cq:formParameters
    jcr:primaryType="nt:unstructured"
    name="photos/primary"
    sling:resourceType="myproject/components/image"/>
```

Each property on this node becomes a hidden form field submitted with the dialog.

### cq:childEditConfig

Controls editing behavior for child components that lack their own `cq:editConfig`. Same structure and properties as `cq:editConfig`. Useful for container components that need to enforce consistent editing behavior on children.

### cq:htmlTag — HTML Wrapper Decorators

Inject attributes onto the auto-generated wrapper `<div>` around the component:

```xml
<!-- _cq_htmlTag/.content.xml -->
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    jcr:primaryType="nt:unstructured"
    class="my-custom-class"
    data-component="hero"/>
```

To suppress the wrapper div entirely, set `cq:noDecoration="{Boolean}true"` on the component node.

### Complete Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    cq:actions="[edit,-,delete,insert,copymove]"
    cq:emptyText="Drag an image here or click to configure"
    jcr:primaryType="cq:EditConfig">
    <cq:dropTargets jcr:primaryType="nt:unstructured">
        <image
            jcr:primaryType="cq:DropTargetConfig"
            accept="[image/gif,image/jpeg,image/png,image/webp,image/tiff,image/svg\\+xml]"
            groups="[media]"
            propertyName="./fileReference"/>
    </cq:dropTargets>
    <cq:inplaceEditing
        jcr:primaryType="cq:InplaceEditingConfig"
        active="{Boolean}true"
        editorType="text">
        <rteConfig jcr:primaryType="nt:unstructured">
            <rtePlugins jcr:primaryType="nt:unstructured">
                <format jcr:primaryType="nt:unstructured"
                    features="bold,italic,underline"/>
                <lists jcr:primaryType="nt:unstructured"
                    features="ordered,unordered"/>
            </rtePlugins>
        </rteConfig>
    </cq:inplaceEditing>
    <cq:listeners
        jcr:primaryType="cq:EditListenersConfig"
        afteredit="REFRESH_PAGE"
        afterinsert="REFRESH_PAGE"
        afterdelete="REFRESH_PAGE"
        aftermove="REFRESH_PAGE"/>
</jcr:root>
```

### Pitfalls

- Do NOT name inplace editing config nodes `config` — use a different name to avoid admin-only access
- Do NOT use `REFRESH_SELF` for `aftermove`/`aftercopy` on nested components — use `REFRESH_PAGE`
- Do NOT define multiple drop targets expecting them to work in Touch UI — only the first is used
- `cq:actionConfigs` with inline handlers is primarily a Classic UI pattern
- `cq:emptyText` only shows when the component renders no visible HTML
- `cq:inherit` defaults to false — explicitly set it if you need parent component behavior inheritance
