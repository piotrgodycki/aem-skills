---
title: Rich Text Editor (RTE) Configuration
impact: HIGH
impactDescription: RTE configuration controls what formatting options authors have — misconfiguration causes broken content, XSS risks, or missing editing features
tags: rte, richtext, plugins, toolbar, paraformat, styles, paste-rules, htmlRules, inplace-editing, dialog
---

## Rich Text Editor (RTE) Configuration

The AEM Rich Text Editor (RTE) is configured through plugins and UI settings. Each plugin controls a feature set, and the toolbar configuration determines which buttons appear in different editing modes.

### RTE Plugins and Features

Activate plugins under the `rtePlugins` node. Each plugin is a child node with a `features` property:

- `features="*"` — Enable all features of the plugin
- `features="bold,italic"` — Enable specific features only
- Omit the property or leave empty — Plugin disabled

#### Complete Plugin Reference

| Plugin ID | Features | Purpose |
|-----------|----------|---------|
| `edit` | `cut`, `copy`, `paste-default`, `paste-plaintext`, `paste-wordhtml` | Clipboard operations |
| `format` | `bold`, `italic`, `underline` | Inline text formatting |
| `justify` | `justifyleft`, `justifycenter`, `justifyright` | Paragraph alignment |
| `links` | `modifylink`, `unlink`, `anchor` | Hyperlinks and anchors |
| `lists` | `ordered`, `unordered`, `indent`, `outdent` | Lists and indentation |
| `table` | `table`, `removetable`, `insertrow`, `removerow`, `insertcolumn`, `removecolumn`, `cellprops`, `mergecells`, `splitcell`, `selectrow`, `selectcolumns` | Table editing |
| `image` | `image` | Inline image insertion |
| `paraformat` | `paraformat` | Paragraph formats (P, H1-H6) |
| `styles` | `styles` | CSS class dropdown |
| `misctools` | `specialchars`, `sourceedit` | Special characters, source HTML editing |
| `spellcheck` | `checktext` | Spell checking |
| `undo` | `undo`, `redo` | Undo/redo history |
| `subsuperscript` | `subscript`, `superscript` | Subscript/superscript |
| `findreplace` | `find`, `replace` | Find and replace |
| `keys` | — | Tab size configuration |

**Default enabled**: `format`, `link`, `list`, `justify`, and `control` plugins with all features.

### RTE in Dialog vs Inplace Editing

#### Dialog Mode (cq:dialog)

```xml
<text
    jcr:primaryType="nt:unstructured"
    sling:resourceType="cq/gui/components/authoring/dialog/richtext"
    name="./text"
    fieldLabel="Text"
    useFixedInlineToolbar="{Boolean}true">
    <rtePlugins jcr:primaryType="nt:unstructured">
        <format jcr:primaryType="nt:unstructured" features="bold,italic,underline"/>
        <justify jcr:primaryType="nt:unstructured" features="*"/>
        <lists jcr:primaryType="nt:unstructured" features="ordered,unordered"/>
        <links jcr:primaryType="nt:unstructured" features="modifylink,unlink"/>
        <paraformat jcr:primaryType="nt:unstructured" features="*">
            <formats jcr:primaryType="nt:unstructured">
                <p jcr:primaryType="nt:unstructured" tag="p" description="Paragraph"/>
                <h2 jcr:primaryType="nt:unstructured" tag="h2" description="Heading 2"/>
                <h3 jcr:primaryType="nt:unstructured" tag="h3" description="Heading 3"/>
                <h4 jcr:primaryType="nt:unstructured" tag="h4" description="Heading 4"/>
            </formats>
        </paraformat>
    </rtePlugins>
    <uiSettings jcr:primaryType="nt:unstructured">
        <cui jcr:primaryType="nt:unstructured">
            <inline
                jcr:primaryType="nt:unstructured"
                toolbar="[format#bold,format#italic,format#underline,#justify,#lists,links#modifylink,links#unlink,#paraformat]">
                <popovers jcr:primaryType="nt:unstructured">
                    <justify
                        jcr:primaryType="nt:unstructured"
                        items="[justify#justifyleft,justify#justifycenter,justify#justifyright]"
                        ref="justify"/>
                    <lists
                        jcr:primaryType="nt:unstructured"
                        items="[lists#unordered,lists#ordered,lists#outdent,lists#indent]"
                        ref="lists"/>
                    <paraformat
                        jcr:primaryType="nt:unstructured"
                        items="paraformat:getFormats:paraformat-pulldown"
                        ref="paraformat"/>
                </popovers>
            </inline>
            <dialogFullScreen
                jcr:primaryType="nt:unstructured"
                toolbar="[format#bold,format#italic,format#underline,justify#justifyleft,justify#justifycenter,justify#justifyright,lists#unordered,lists#ordered,lists#outdent,lists#indent,links#modifylink,links#unlink,#paraformat]">
                <popovers jcr:primaryType="nt:unstructured">
                    <paraformat
                        jcr:primaryType="nt:unstructured"
                        items="paraformat:getFormats:paraformat-pulldown"
                        ref="paraformat"/>
                </popovers>
            </dialogFullScreen>
        </cui>
    </uiSettings>
</text>
```

#### Inplace Editing Mode (cq:editConfig)

```xml
<cq:inplaceEditing
    jcr:primaryType="cq:InplaceEditingConfig"
    active="{Boolean}true"
    editorType="text">
    <rteConfig jcr:primaryType="nt:unstructured">
        <rtePlugins jcr:primaryType="nt:unstructured">
            <format jcr:primaryType="nt:unstructured" features="bold,italic"/>
            <lists jcr:primaryType="nt:unstructured" features="ordered,unordered"/>
        </rtePlugins>
        <uiSettings jcr:primaryType="nt:unstructured">
            <cui jcr:primaryType="nt:unstructured">
                <inline
                    jcr:primaryType="nt:unstructured"
                    toolbar="[format#bold,format#italic,#lists]">
                    <popovers jcr:primaryType="nt:unstructured">
                        <lists
                            jcr:primaryType="nt:unstructured"
                            items="[lists#unordered,lists#ordered]"
                            ref="lists"/>
                    </popovers>
                </inline>
            </cui>
        </uiSettings>
    </rteConfig>
</cq:inplaceEditing>
```

**Critical**: Do NOT name the config node `config` — use any other name (e.g., `rteConfig`). The name `config` restricts RTE configuration to admin users only.

### Toolbar Button Syntax

Toolbar items use the format `PluginName#FeatureName`:

- `format#bold` — Bold button
- `links#modifylink` — Create/edit link button
- `#paraformat` — Popover trigger for paraformat
- `-` — Separator

Popovers group related actions under a dropdown:

```xml
<popovers jcr:primaryType="nt:unstructured">
    <paraformat
        jcr:primaryType="nt:unstructured"
        items="paraformat:getFormats:paraformat-pulldown"
        ref="paraformat"/>
</popovers>
```

### Key Properties

| Property | Description |
|----------|-------------|
| `useFixedInlineToolbar` | `{Boolean}true` — Fixed toolbar instead of floating. **Required for Touch UI dialogs.** |
| `customStart` | `{Boolean}true` — Control RTE start programmatically |
| `height` | Pixel height of the editable area |

### Paraformat Configuration

Define available paragraph formats (heading levels):

```xml
<paraformat jcr:primaryType="nt:unstructured" features="*">
    <formats jcr:primaryType="nt:unstructured">
        <p jcr:primaryType="nt:unstructured" tag="p" description="Paragraph"/>
        <h1 jcr:primaryType="nt:unstructured" tag="h1" description="Heading 1"/>
        <h2 jcr:primaryType="nt:unstructured" tag="h2" description="Heading 2"/>
        <h3 jcr:primaryType="nt:unstructured" tag="h3" description="Heading 3"/>
        <h4 jcr:primaryType="nt:unstructured" tag="h4" description="Heading 4"/>
        <h5 jcr:primaryType="nt:unstructured" tag="h5" description="Heading 5"/>
        <h6 jcr:primaryType="nt:unstructured" tag="h6" description="Heading 6"/>
    </formats>
</paraformat>
```

**Important**: Always include `<p>` tag. Removing it prevents authors from selecting paragraph format.

### Styles Dropdown Configuration

Define CSS classes authors can apply to text:

```xml
<styles jcr:primaryType="nt:unstructured" features="*">
    <styles jcr:primaryType="nt:unstructured">
        <highlight jcr:primaryType="nt:unstructured"
            cssName="text-highlight"
            text="Highlight"/>
        <lead jcr:primaryType="nt:unstructured"
            cssName="text-lead"
            text="Lead Paragraph"/>
        <small jcr:primaryType="nt:unstructured"
            cssName="text-small"
            text="Small Text"/>
    </styles>
</styles>
```

Reference CSS via `externalStyleSheets` property (multi-string array) on the parent node.

### Paste Rules (htmlPasteRules)

Control what HTML is allowed when pasting from external sources:

```xml
<edit jcr:primaryType="nt:unstructured"
    features="*"
    defaultPasteMode="wordhtml">
    <htmlPasteRules jcr:primaryType="nt:unstructured">
        <allowBasics jcr:primaryType="nt:unstructured"
            bold="{Boolean}true"
            italic="{Boolean}true"
            underline="{Boolean}true"
            anchor="{Boolean}true"
            image="{Boolean}false"/>
        <allowBlockTags>[p,h1,h2,h3,h4,h5,h6]</allowBlockTags>
        <fallbackBlockTag>p</fallbackBlockTag>
        <table jcr:primaryType="nt:unstructured"
            allow="{Boolean}true"
            ignoreMode="remove"/>
        <list jcr:primaryType="nt:unstructured"
            allow="{Boolean}true"
            ignoreMode="remove"/>
    </htmlPasteRules>
</edit>
```

| Property | Description |
|----------|-------------|
| `defaultPasteMode` | `plaintext`, `wordhtml`, or `browser` |
| `allowBasics` | Which inline formats to preserve |
| `allowBlockTags` | Which block elements to keep |
| `fallbackBlockTag` | Default block tag for unrecognized elements |
| `table.ignoreMode` | `remove` (strip tables) or `paragraph` (convert to paragraphs) |
| `list.ignoreMode` | `remove` (strip lists) or `paragraph` (convert to paragraphs) |

### htmlRules — Link Configuration

Configure link behavior (sibling to `rtePlugins` node, not inside it):

```xml
<htmlRules jcr:primaryType="nt:unstructured">
    <links jcr:primaryType="nt:unstructured"
        cssInternal="internal-link"
        cssExternal="external-link"
        protocols="[https://,http://,mailto:,tel:]"
        defaultProtocol="https://">
        <targetConfig jcr:primaryType="nt:unstructured"
            mode="auto"
            targetInternal="_self"
            targetExternal="_blank"/>
    </links>
</htmlRules>
```

### Special Characters Configuration

```xml
<misctools jcr:primaryType="nt:unstructured" features="specialchars">
    <specialCharsConfig jcr:primaryType="nt:unstructured">
        <chars jcr:primaryType="nt:unstructured">
            <copyright jcr:primaryType="nt:unstructured" entity="&amp;copy;"/>
            <registered jcr:primaryType="nt:unstructured" entity="&amp;reg;"/>
            <trademark jcr:primaryType="nt:unstructured" entity="&amp;trade;"/>
            <range1 jcr:primaryType="nt:unstructured"
                rangeStart="{Long}9998"
                rangeEnd="{Long}10000"/>
        </chars>
    </specialCharsConfig>
</misctools>
```

### Table Styles

```xml
<table jcr:primaryType="nt:unstructured" features="*">
    <tableStyles jcr:primaryType="nt:unstructured">
        <striped jcr:primaryType="nt:unstructured"
            cssName="table-striped"
            text="Striped Rows"/>
    </tableStyles>
    <cellStyles jcr:primaryType="nt:unstructured">
        <highlight jcr:primaryType="nt:unstructured"
            cssName="cell-highlight"
            text="Highlight Cell"/>
    </cellStyles>
</table>
```

### Additional Settings

| Setting | Location | Property | Description |
|---------|----------|----------|-------------|
| Undo history | `undo/` | `maxUndoSteps` (Long) | Max undo steps (default: 50) |
| Tab size | `keys/` | `tabSize` (String) | Number of spaces per tab |
| Indent margin | `lists/` | `identSize` (Long) | Indent margin in pixels |

### Mode Differences

| Mode | Toolbar | Source Edit | Drag Images | Best For |
|------|---------|-------------|-------------|----------|
| Inline | Compact, floating/fixed | No | No | Quick edits |
| Fullscreen | Full toolbar | Yes | No | Detailed editing |
| Dialog | Configurable | Optional | Yes | Standard editing |
| Dialog fullscreen | Full toolbar | Yes | Yes | Complex content |

### Content Policies Integration

Template-level RTE configuration uses content policies. The UI settings (`uiSettings`) define what **can** be available; content policies then control what **is** available for each template:

- UI config must enable a feature first
- Content policy can then restrict that feature per template
- Core Components text component supports policy-based RTE configuration without JCR editing

### Pitfalls

- Do NOT name RTE config nodes `config` — non-admin users lose access
- Do NOT omit `useFixedInlineToolbar="{Boolean}true"` in dialog RTE — toolbar floats and misbehaves
- Do NOT remove `<p>` from paraformat — authors lose paragraph format selection
- `source-edit` is NOT available in inline editing mode
- Full-screen mode does NOT support drag images
- Paste rules only apply to `paste-wordhtml` mode
- `htmlRules` is a sibling of `rtePlugins`, not a child
- For custom RTE plugins in Touch UI, use the `rte.coralui3` library
- Spellcheck dictionaries must be MySpell format (.aff/.dic) placed at `/apps/cq/spellchecker/dictionaries/`
