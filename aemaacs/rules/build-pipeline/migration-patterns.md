---
title: Frontend Migration Patterns (6.5 to Cloud Service)
impact: CRITICAL
impactDescription: Migration errors cause build failures, broken components, and runtime exceptions that block Cloud Service deployment
tags: migration, htl, jsp, classic-ui, touch-ui, coral3, core-components, dialog-conversion, repository-modernizer, bpa, cloud-service
---

## Frontend Migration Patterns

Migrating frontend code to AEM Cloud Service requires addressing JSP-to-HTL conversion, Classic UI dialog modernization, Foundation-to-Core-Component replacement, and Cloud Service architectural constraints. Use AEM Modernization Tools and Best Practices Analyzer to automate where possible.

### JSP to HTL Migration Guide

HTL (HTML Template Language) is the required scripting language for AEM Cloud Service. JSP files still compile but are strongly discouraged and will not be supported long-term.

#### Common JSP Patterns and HTL Equivalents

##### Scriptlets and Expressions

```jsp
<%-- JSP: scriptlet with expression --%>
<%
  String title = properties.get("jcr:title", "Default Title");
  String description = properties.get("jcr:description", "");
%>
<h1><%= title %></h1>
<% if (description != null && !description.isEmpty()) { %>
  <p><%= description %></p>
<% } %>
```

```html
<!-- HTL: expression with default and conditional -->
<h1>${properties.jcr:title @ default='Default Title'}</h1>
<p data-sly-test="${properties.jcr:description}">${properties.jcr:description}</p>
```

##### JSTL forEach / Iteration

```jsp
<%-- JSP: JSTL iteration --%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<ul>
  <c:forEach var="item" items="${itemList}">
    <li>${item.title}</li>
  </c:forEach>
</ul>
```

```html
<!-- HTL: data-sly-list -->
<ul data-sly-list.item="${model.items}">
  <li>${item.title}</li>
</ul>

<!-- HTL: data-sly-repeat (keeps the host element) -->
<li data-sly-repeat.item="${model.items}">${item.title}</li>
```

##### JSP Include / Component Rendering

```jsp
<%-- JSP: include another component --%>
<cq:include path="header" resourceType="myproject/components/header" />

<%-- JSP: include a script --%>
<jsp:include page="/apps/myproject/components/utils/helper.jsp" />
```

```html
<!-- HTL: data-sly-resource (include component) -->
<div data-sly-resource="${'header' @ resourceType='myproject/components/header'}"></div>

<!-- HTL: data-sly-include (include script) -->
<div data-sly-include="helper.html"></div>
```

##### Use-API (Replacing Scriptlet Logic)

```jsp
<%-- JSP: complex logic in scriptlet --%>
<%
  PageManager pageManager = resourceResolver.adaptTo(PageManager.class);
  Page currentPage = pageManager.getContainingPage(resource);
  List<Page> children = new ArrayList<>();
  Iterator<Page> it = currentPage.listChildren();
  while (it.hasNext()) {
    children.add(it.next());
  }
%>
```

```html
<!-- HTL: Use-API with Sling Model -->
<sly data-sly-use.model="com.myproject.models.NavigationModel">
  <nav data-sly-list.page="${model.childPages}">
    <a href="${page.path}.html">${page.title}</a>
  </nav>
</sly>
```

```java
// Sling Model backing the HTL template
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class NavigationModel {
    @ScriptVariable
    private Page currentPage;

    public List<Page> getChildPages() {
        List<Page> children = new ArrayList<>();
        Iterator<Page> it = currentPage.listChildren();
        it.forEachRemaining(children::add);
        return children;
    }
}
```

##### Taglibs and Formatting

```jsp
<%-- JSP: format date with JSTL --%>
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
<fmt:formatDate value="${event.date}" pattern="MMMM d, yyyy" />
```

```html
<!-- HTL: format via Use-API (no built-in date formatting in HTL) -->
<sly data-sly-use.model="com.myproject.models.EventModel">
  <time datetime="${model.isoDate}">${model.formattedDate}</time>
</sly>
```

##### Template and Call (Replacing JSP Tag Files)

```html
<!-- HTL: Define a reusable template -->
<template data-sly-template.card="${@ title, description, link}">
  <div class="cmp-card">
    <h3 class="cmp-card__title">${title}</h3>
    <p class="cmp-card__description">${description}</p>
    <a class="cmp-card__link" href="${link}">Read more</a>
  </div>
</template>

<!-- HTL: Call the template -->
<sly data-sly-use.tmpl="card-template.html">
  <sly data-sly-call="${tmpl.card @ title='My Title', description='My Description', link='/content/page.html'}"></sly>
</sly>
```

##### Context-Aware Escaping

```html
<!-- HTL automatic XSS protection via display contexts -->
${properties.title}                          <!-- Escapes for HTML text (default) -->
${properties.title @ context='html'}         <!-- Explicit HTML context -->
${properties.link @ context='uri'}           <!-- URI context (validates URL) -->
${properties.style @ context='styleString'}  <!-- CSS value context -->
${properties.data @ context='scriptString'}  <!-- JavaScript string context -->
${properties.markup @ context='unsafe'}      <!-- No escaping (use with extreme caution) -->
```

### Classic UI to Touch UI Migration

#### Dialog Conversion (cq:dialog vs dialog)

| Classic UI (ExtJS) | Touch UI (Granite/Coral) |
|--------------------|--------------------------|
| `dialog` node (cq:Dialog) | `cq_dialog` node (nt:unstructured) |
| ExtJS xtypes | Granite UI resource types |
| `xtype="textfield"` | `granite/ui/components/coral/foundation/form/textfield` |
| `xtype="pathfield"` | `granite/ui/components/coral/foundation/form/pathfield` |
| `xtype="richtext"` | `cq/gui/components/authoring/dialog/richtext` |
| `xtype="selection"` (combo) | `granite/ui/components/coral/foundation/form/select` |
| `xtype="checkbox"` | `granite/ui/components/coral/foundation/form/checkbox` |
| `xtype="dialogfieldset"` | `granite/ui/components/coral/foundation/form/fieldset` |
| `xtype="tabpanel"` | `granite/ui/components/coral/foundation/tabs` |

#### Dialog Conversion Tool

The Dialog Conversion Tool automates Classic UI dialog migration:

1. Open **Tools > AEM Modernization Tools > Dialog Conversion** in AEM Author
2. Search for components with Classic UI dialogs
3. Select components to convert
4. Run conversion -- tool creates `cq_dialog` nodes from `dialog` nodes
5. Review generated Touch UI dialogs and fix any conversion gaps
6. Test all dialog fields and validation

**Limitations:** Complex ExtJS customizations (custom xtypes, JavaScript listeners) require manual migration.

### Coral 2 to Coral 3 Dialog Migration

AEM Cloud Service requires Coral 3 (CoralUI3). Coral 2 dialogs will not render correctly.

| Coral 2 | Coral 3 |
|---------|---------|
| `coral-Select` | `coral-select` (lowercase) |
| `coral-Button--primary` | `coral-button--primary` |
| `coral-Textfield` | `coral-textfield` |
| Class-based API (`new CUI.Widget()`) | Web Components API (`document.createElement('coral-select')`) |
| `data-init="*"` initialization | Auto-initialized Web Components |
| jQuery required | No jQuery dependency |

**Key change:** Coral 3 uses Web Components. Custom dialog JavaScript that manipulates Coral 2 classes or uses `CUI.*` constructors must be rewritten to use the Coral 3 Web Component API.

```javascript
// Coral 2 (REMOVE)
var select = new CUI.Select({ element: '#mySelect' });

// Coral 3 (REPLACE WITH)
var select = document.querySelector('coral-select#mySelect');
select.items.add({ value: 'new', content: { textContent: 'New Option' } });
```

### AEM 6.5 to Cloud Service Frontend Differences

#### Immutable /libs vs Mutable /apps

```
Cloud Service repository:
├── /libs/          # IMMUTABLE - Adobe-managed, cannot overlay or modify
├── /apps/          # IMMUTABLE at runtime - deployed via Cloud Manager only
├── /content/       # MUTABLE - content, editable at runtime
├── /conf/          # MUTABLE - configurations
├── /var/           # MUTABLE - runtime data
└── /etc/           # MUTABLE (limited) - mappings, designs (legacy)
```

- `/apps` and `/libs` cannot be modified at runtime (no CRXDE Lite writes, no JCR API writes)
- All code changes must go through Git and Cloud Manager pipeline
- Overlays of `/libs` content must be placed in `/apps` and deployed via code package

#### No CRXDE Lite on Publish

- CRXDE Lite is available on author (dev environments) for debugging only
- Not available on publish or stage/production author
- Never use CRXDE for production changes -- all changes must be in Git

#### No Custom Run Modes

```
AEM 6.5 (allowed):               Cloud Service (NOT allowed):
config.mysite/                    config.mysite/          ← BLOCKED
config.brand-a/                   config.brand-a/         ← BLOCKED
config.feature-x/                 config.feature-x/       ← BLOCKED

Cloud Service run modes (allowed only):
config/                           # All tiers, all environments
config.author/                    # Author tier
config.publish/                   # Publish tier
config.author.dev/                # Author + dev
config.publish.prod/              # Publish + production
```

#### Oak Indexes Must Be in Code

Custom Oak index definitions must be packaged in the `ui.apps` module and deployed via Cloud Manager:

```xml
<!-- /oak:index/myproject-lucene in ui.apps -->
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
           xmlns:oak="http://jackrabbit.apache.org/oak/ns/1.0"
    jcr:primaryType="oak:QueryIndexDefinition"
    type="lucene"
    compatVersion="{Long}2"
    async="async"
    evaluatePathRestrictions="{Boolean}true"
    includedPaths="[/content/mysite]"
    queryPaths="[/content/mysite]">
    <indexRules jcr:primaryType="nt:unstructured">
        <nt:unstructured jcr:primaryType="nt:unstructured">
            <properties jcr:primaryType="nt:unstructured">
                <title jcr:primaryType="nt:unstructured"
                    name="jcr:title"
                    propertyIndex="{Boolean}true"
                    analyzed="{Boolean}true"/>
            </properties>
        </nt:unstructured>
    </indexRules>
</jcr:root>
```

- Indexes are processed before traffic shifts during rolling deployments
- Index changes can significantly extend deployment time
- Test index definitions locally with the AEM SDK before pipeline deployment

#### ClientLib Changes (allowProxy Mandatory)

```xml
<!-- AEM 6.5: allowProxy was optional -->
<jcr:root jcr:primaryType="cq:ClientLibraryFolder"
    categories="[myproject.base]" />

<!-- Cloud Service: allowProxy is required for publish access -->
<jcr:root jcr:primaryType="cq:ClientLibraryFolder"
    categories="[myproject.base]"
    allowProxy="{Boolean}true" />
```

Without `allowProxy`, ClientLibs under `/apps` are not accessible on publish. This is the most common cause of missing CSS/JS after migration.

#### Dispatcher Immutable Configuration

- Dispatcher config is deployed as immutable code via Cloud Manager
- No runtime changes to Dispatcher rules
- Configuration validated during pipeline build
- Standard structure: `conf.d/` (vhosts), `conf.dispatcher.d/` (farm, filter, cache rules)
- Rewrite rules replace `/etc/map` for URL shortening on publish

#### Frontend Pipeline (New in Cloud Service)

A dedicated pipeline for frontend assets, separate from the full-stack pipeline:

```
Full-Stack Pipeline:                Frontend Pipeline (new):
├── Build all modules               ├── Build ui.frontend only
├── Deploy ui.apps                  ├── Deploy to /conf/<site>/settings/
├── Deploy ui.config                │   wcm/clientlibs/<hash>/
├── Deploy ui.content               ├── No full AEM restart
├── Run tests                       ├── Deploy in ~5 minutes
└── Restart AEM (rolling)           └── Independent release cycle
```

- Decouples frontend releases from backend
- Generates a unique hash-based ClientLib folder
- Requires Site Theme configured on the AEM Site root page
- Use `aem-site-theme-builder` for local development

### Foundation Components to Core Components Migration

| Foundation Component | Core Component Replacement |
|---------------------|---------------------------|
| `foundation/components/text` | `core/wcm/components/text/v2/text` |
| `foundation/components/image` | `core/wcm/components/image/v3/image` |
| `foundation/components/title` | `core/wcm/components/title/v3/title` |
| `foundation/components/list` | `core/wcm/components/list/v4/list` |
| `foundation/components/breadcrumb` | `core/wcm/components/breadcrumb/v3/breadcrumb` |
| `foundation/components/parsys` | Layout Container (editable templates) |
| `foundation/components/form/*` | `core/fd/components/form/*` (Adaptive Forms Core Components) |

**Migration approach:**
1. Create proxy components under `/apps/myproject/components/` that point to Core Components
2. Map old resource types to new ones using Sling Resource Merger or content transformation
3. Update HTL templates to use Core Component delegation patterns
4. Migrate dialog properties to match Core Component dialog structure

#### Proxy Component Pattern

```xml
<!-- /apps/myproject/components/title/.content.xml -->
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
           xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    jcr:primaryType="cq:Component"
    jcr:title="Title"
    sling:resourceSuperType="core/wcm/components/title/v3/title"
    componentGroup="My Project - Content" />
```

The proxy inherits all Core Component functionality. Customize only by:
- Adding extra dialog tabs in `_cq_dialog/.content.xml`
- Overriding specific HTL templates (e.g., `title.html`)
- Extending the Sling Model via delegation

### Repository Modernizer Tool

Restructures a project to match AEM Cloud Service requirements:

```
Before (AEM 6.x structure):        After (Cloud Service structure):
├── /apps/mysite/install/          ├── ui.apps/
│   └── mybundle.jar               │   └── /apps/mysite/
├── /apps/mysite/config/           ├── ui.config/
│   └── *.cfg.json                 │   └── /apps/mysite/osgiconfig/
├── /etc/designs/mysite/           ├── ui.content/
├── /etc/clientlibs/mysite/        │   └── /content/, /conf/
└── /content/dam/mysite/           └── ui.frontend/
                                       └── webpack/src/
```

**What it does:**
- Separates code (`ui.apps`) from content (`ui.content`) and config (`ui.config`)
- Moves OSGi configurations to proper `osgiconfig` folder structure
- Relocates `/etc` content to Cloud Service locations (`/conf`, `/apps`)
- Updates filter.xml files for each module
- Creates proper `all/` container package

### Best Practices Analyzer (BPA) -- Frontend Findings

Run BPA on your AEM 6.5 instance before migration. Key frontend-related findings:

| BPA Code | Description | Fix |
|----------|-------------|-----|
| **INST** | Custom artifact installed in /libs | Move to /apps overlay |
| **ECU** | Existing Classic UI usage | Convert dialogs to Touch UI |
| **DOPI** | Deprecated or legacy APIs | Replace with current API equivalents |
| **UMI** | Mutable content in immutable area | Move to /content or /conf |
| **CCOM** | Custom component using Foundation | Migrate to Core Component proxy |
| **OAUI** | Classic UI dialog found | Run Dialog Conversion Tool |
| **NCC** | Non-compliant code patterns | Refactor per Cloud Service requirements |
| **LOCP** | /libs overlay detected | Verify overlay still needed, move to /apps |

**Running BPA:**
1. Install BPA package on AEM 6.5 author
2. Navigate to **Tools > Operations > Best Practices Analyzer**
3. Click **Generate Report**
4. Export report for Cloud Acceleration Manager import

### Content Migration Considerations

#### Content Transformer

Automates content restructuring during migration:
- Updates resource types from Foundation to Core Components
- Transforms `/etc` content paths to new locations
- Rewrites hardcoded references (e.g., `/content/dam` absolute paths)
- Applies bulk property changes (e.g., adding `allowProxy` to clientlibs)

#### Content Transfer Tool (CTT)

Moves content from AEM 6.5 to Cloud Service:
- Extracts content from source AEM instance
- Ingests into Cloud Service via Cloud Acceleration Manager
- Supports incremental (top-up) transfers
- Does NOT transform content -- run Content Transformer first

### Feature Parity Gaps (6.5 vs Cloud Service)

| Feature | AEM 6.5 | Cloud Service | Migration Action |
|---------|---------|---------------|-----------------|
| CRXDE on publish | Available | Not available | Use Git + pipeline |
| Custom run modes | Unlimited | Fixed set only | Use env variables |
| `/libs` overlays | Allowed (risky) | Blocked | Move to `/apps` |
| Replication agents | Custom agents | Content Distribution | Remove custom agents |
| Workflow launchers | Full access | Limited | Use Assets processing profiles |
| Design mode/page | `/etc/designs` | Editable Templates + Style System | Migrate designs |
| Static templates | Supported | Deprecated | Convert to editable templates |
| Tag namespaces in /etc | `/etc/tags` | `/content/cq:tags` | Relocate tags |
| MSM Blueprint | `/etc/blueprints` | `/libs/msm` or `/apps` | Update references |
| Client install hooks | Supported | Not supported | Use Repo Init |
| Package install hooks | Supported | Not supported | Use Repo Init |

### Migration Checklist

```
Pre-migration:
[ ] Run Best Practices Analyzer on AEM 6.5
[ ] Run AEM Modernization Tools (dialog conversion, component conversion)
[ ] Run Repository Modernizer on project code
[ ] Convert all JSP components to HTL
[ ] Replace Foundation Components with Core Component proxies
[ ] Verify all ClientLibs have allowProxy=true
[ ] Move OSGi configs to .cfg.json format in osgiconfig/ folders
[ ] Replace hardcoded URLs with Externalizer or environment variables
[ ] Remove custom run mode folders
[ ] Package Oak index definitions in ui.apps

Frontend-specific:
[ ] Verify Coral 3 compatibility in all custom dialogs
[ ] Test all component dialogs in Touch UI
[ ] Validate Style System configurations work with Core Components
[ ] Set up ui.frontend module with proper webpack/build config
[ ] Configure frontend pipeline in Cloud Manager (if using)
[ ] Test ClientLib loading on publish (via allowProxy)
[ ] Verify Dispatcher rewrite rules match /etc/map expectations
[ ] Test all forms with Adaptive Forms Core Components
```
