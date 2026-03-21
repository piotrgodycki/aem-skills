---
title: AEMaaCS Multi-Tenant / Multi-Site Architecture
impact: HIGH
impactDescription: Multi-tenant patterns define content isolation, code reuse, and governance across brands/regions in AEM Cloud Service
tags: aem, aemaacs, multi-tenant, multi-site, msm, live-copy, blueprint, clientlibs, i18n, dispatcher, osgi, templates
---

## AEMaaCS Multi-Tenant / Multi-Site Architecture

AEM Cloud Service supports multi-site deployments but does not offer true multi-tenancy. Environment configurations and system resources are always shared across all sites deployed on an environment. Careful architectural planning is required for content isolation, code organization, and governance.

### 1. Multi-Site Manager (MSM) / Live Copy

#### Core Concepts

| Term | Definition |
|------|-----------|
| **Blueprint (Source)** | Original pages serving as the basis for Live Copies |
| **Live Copy** | A synchronized copy maintaining an active link to its source |
| **Live Relationship** | The connection ensuring changes propagate from source to copy |
| **Rollout** | Process pushing modifications from source to Live Copies |
| **Synchronize** | Manual request pulling changes into Live Copy from source |

#### Blueprint Configuration

Blueprint configurations are **immutable data in AEM Cloud Service** -- not editable at runtime. Changes require Git deployment via CI/CD pipeline.

Creating a blueprint:
1. Navigate to **Tools > Sites > Blueprints**
2. Select **Create**, choose template
3. Define **Name**, **Source Path** (root page of source content), **Description**

Benefits of blueprint configurations:
- Enable the **Rollout** button for explicit content push
- Allow **Create Site** for language selection and structure configuration
- Define default rollout configurations for Live Copies

#### Out-of-the-Box Rollout Configurations

| Configuration | Trigger | Actions |
|--------------|---------|---------|
| **Standard Rollout Config** | On Rollout | `contentUpdate`, `contentCopy`, `contentDelete`, `referencesUpdate`, `productUpdate`, `orderChildren` |
| **Activate on Blueprint Activation** | On Activation | `targetActivate` |
| **Deactivate on Blueprint Deactivation** | On Deactivation | `targetDeactivate` |
| **Push on Modify** | On Modification | Content sync actions (use sparingly -- performance impact) |
| **Push on Modify (Shallow)** | On Modification | Content sync excluding reference updates |
| **Promote Launch** | On Rollout | `markLiveRelationship` for launch page promotion |

#### Rollout Triggers

- **On Rollout**: Manual via Rollout/Synchronize commands
- **On Modification**: Automatic when source page changes (performance warning)
- **On Activation**: When source page is published
- **On Deactivation**: When source page is unpublished

#### Synchronization Actions

| Action | Purpose |
|--------|---------|
| `contentCopy` | Copies missing source nodes to Live Copy |
| `contentDelete` | Removes Live Copy nodes absent on source |
| `contentUpdate` | Updates Live Copy with source changes |
| `referencesUpdate` | Updates internal references within Live Copy |
| `orderChildren` | Reorders child nodes matching blueprint |
| `editProperties` | Modifies properties using regex pattern mapping |
| `targetActivate` | Publishes Live Copy page |
| `targetDeactivate` | Unpublishes Live Copy page |
| `targetVersion` | Creates Live Copy version |
| `workflow` | Initiates configured workflow |
| `mandatory` | Sets ACL read-only permissions |
| `mandatoryContent` | Restricts property/ACL modifications |
| `mandatoryStructure` | Prevents node removal |
| `PageMoveAction` | Handles blueprint page relocation |
| `VersionCopyAction` | Creates Live Copy from published source version |

#### Inheritance and Cancellation

**Page-level inheritance**:
- **Suspend**: Temporarily stops synchronization; allows local modifications; relationship stays dormant
- **Resume**: Reinstates the live relationship with optional synchronization
- **Detach**: Permanently removes all live relationships (non-reversible)

**Component-level inheritance**:
- Cancel inheritance on individual components via toolbar icon
- Re-enable via **Re-enable Inheritance** icon
- Component order in paragraph system can be modified even with inheritance active

**Property-level inheritance**:
- Cancel inheritance for specific page properties via the link icon in page properties
- Broken-link icon indicates cancelled inheritance state

#### Rollout Configuration Resolution Order

1. **Live Copy page properties** (direct override)
2. **Blueprint page properties** (if Live Copy unconfigured)
3. **Live Copy parent page** (inherited)
4. **System default** (OSGi: `com.day.cq.wcm.msm.impl.LiveRelationshipManagerImpl`, property `liverelationshipmgr.relationsconfig.default`, default `/libs/msm/wcm/rolloutconfigs/default`)

#### Excluding Properties via OSGi

Five actions support exclusion configuration:

```
com.day.cq.wcm.msm.impl.actions.ContentCopyActionFactory
com.day.cq.wcm.msm.impl.actions.ContentDeleteActionFactory
com.day.cq.wcm.msm.impl.actions.ContentUpdateActionFactory
com.day.cq.wcm.msm.impl.actions.ReferencesUpdateActionFactory
com.day.cq.wcm.msm.impl.actions.PageMoveActionFactory
```

Configurable properties (regex patterns):
- `cq.wcm.msm.action.excludednodetypes` -- Excluded node types
- `cq.wcm.msm.action.excludedparagraphitems` -- Excluded paragraph items
- `cq.wcm.msm.action.excludedprops` -- Excluded page properties
- `cq.wcm.msm.action.ignoredMixin` -- Ignored mixin node types (contentUpdate only)

#### MSM Best Practices

- **Plan carefully** before starting -- treat MSM as a major website undertaking
- **Customize as little as possible** -- extensive customization compromises performance and upgradeability
- **Avoid onModify triggers** unless benefits outweigh performance costs
- **Do not create ungoverned chained inheritances** -- increases complexity
- Declare components as containers (`cq:isContainer` property) to preserve locally-added nested components
- **CUGs in the Permissions tab cannot be rolled out** from Blueprints
- Creating pages in blueprints automatically creates corresponding Live Copy pages
- Deleting blueprint pages removes corresponding Live Copy pages
- **Moving pages does not automatically move in Live Copies**

#### MSM Anti-Patterns

- Automating rollouts with `onModify` without performance testing
- Complex chained inheritance structures
- Manual language/content additions below first blueprint levels
- Skipping prototype and testing phases
- Granting excessive local content producer authority

---

### 2. Content Architecture for Multi-Tenant

#### JCR Content Tree Structure

```
/content/
├── brand-a/                    # Brand A site root
│   ├── language-masters/       # Blueprint content
│   │   ├── en/
│   │   ├── de/
│   │   └── fr/
│   ├── us/                     # Live Copy for US
│   │   └── en/
│   ├── de/                     # Live Copy for Germany
│   │   └── de/
│   └── ch/                     # Live Copy for Switzerland
│       ├── de/
│       ├── fr/
│       └── it/
├── brand-b/                    # Brand B site root
│   ├── language-masters/
│   │   └── en/
│   └── us/
│       └── en/
└── dam/
    ├── brand-a/                # Brand A assets
    │   ├── shared/
    │   ├── us/
    │   └── de/
    └── brand-b/                # Brand B assets
        └── shared/
```

#### Configuration Hierarchy (`/conf/`)

```
/conf/
├── global/                     # Global defaults (fallback)
│   └── settings/
│       └── wcm/
│           ├── templates/
│           └── policies/
├── brand-a/                    # Brand A configuration
│   └── settings/
│       ├── wcm/
│       │   ├── templates/      # Brand A templates
│       │   └── policies/       # Brand A policies
│       ├── dam/
│       │   └── imageserver/    # Brand A DAM config
│       └── cloudconfigs/       # Brand A cloud configs
└── brand-b/                    # Brand B configuration
    └── settings/
        └── wcm/
            ├── templates/
            └── policies/
```

#### Content-Configuration Linkage

Content references its configuration via `cq:conf` property on `jcr:content`:

```xml
<!-- /content/brand-a/jcr:content -->
<jcr:content
    cq:conf="/conf/brand-a"
    jcr:primaryType="cq:PageContent"/>
```

**Resolution order** (checked top-down):
1. Specific configuration via `cq:conf` property on content
2. Parent configurations (traversed up hierarchy)
3. Site configuration (`/conf/<siteconfig>`)
4. Global configuration (`/conf/global`)
5. Application defaults (`/apps`)
6. Product defaults (`/libs`)

#### Shared vs Brand-Specific Component Proxies

Each brand creates proxy components pointing to Core Components:

```
/apps/
├── brand-a/
│   └── components/
│       ├── title/
│       │   └── .content.xml    # sling:resourceSuperType="core/wcm/components/title/v3/title"
│       └── teaser/
│           └── .content.xml    # componentGroup="Brand A"
├── brand-b/
│   └── components/
│       ├── title/
│       │   └── .content.xml    # componentGroup="Brand B"
│       └── teaser/
│           └── .content.xml
└── shared/
    └── components/             # Shared custom components (not Core Component proxies)
        └── custom-nav/
```

**Proxy component `.content.xml` example**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          jcr:primaryType="cq:Component"
          jcr:title="Title"
          jcr:description="Brand A Title"
          componentGroup="Brand A"
          sling:resourceSuperType="core/wcm/components/title/v3/title"/>
```

Core Components belong to hidden groups (`.core-wcm`), preventing direct author access. Proxy components expose them via brand-specific `componentGroup` values.

#### Tenant Isolation Best Practices

- Use `componentGroup` and `allowedPaths` to restrict author visibility per brand
- Brand A authors see only Brand A components; Brand B sees only Brand B
- Configure workflow launchers per tenant repository path
- Avoid overlays -- they affect the entire AEM instance; use alternative paths for tenant-specific experiences
- Avoid vanity URLs -- no uniqueness guarantees; use Apache mod_rewrite dispatcher rules

#### DAM Structure

```
/content/dam/
├── brand-a/
│   ├── images/
│   ├── videos/
│   └── documents/
├── brand-b/
│   ├── images/
│   └── documents/
└── shared/                     # Cross-brand shared assets
    ├── icons/
    └── logos/
```

Configure workflow launchers to execute on tenant-specific DAM paths.

---

### 3. Frontend Multi-Tenant Patterns

#### ClientLib Structure for Multi-Tenant

```
/apps/
├── shared/
│   └── clientlibs/
│       ├── base/                          # Shared base styles/scripts
│       │   ├── css.txt
│       │   ├── js.txt
│       │   └── css/
│       │       ├── variables.css          # Shared design tokens
│       │       ├── mixins.css
│       │       └── components-base.css
│       └── vendor/                        # Third-party libraries
│           └── ...
├── brand-a/
│   └── clientlibs/
│       ├── site/                          # Brand A site clientlib
│       │   ├── .content.xml               # categories=["brand-a.site"]
│       │   ├── css.txt
│       │   └── css/
│       │       ├── variables-override.css # Brand A color/font overrides
│       │       └── brand-a-custom.css
│       └── components/                    # Brand A component overrides
│           └── ...
└── brand-b/
    └── clientlibs/
        └── site/
            ├── .content.xml               # categories=["brand-b.site"]
            └── css/
                └── variables-override.css # Brand B overrides
```

**ClientLib node definition** (`.content.xml`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0"
          jcr:primaryType="cq:ClientLibraryFolder"
          allowProxy="{Boolean}true"
          categories="[brand-a.site]"
          embed="[shared.base]"/>
```

The `embed` property pulls shared base styles into the brand-specific clientlib, producing a single combined CSS/JS output.

#### ClientLib Categories Naming Convention

- `shared.base` -- Core shared styles and scripts
- `shared.vendor` -- Third-party libraries
- `brand-a.site` -- Brand A site-level styles (embeds shared.base)
- `brand-a.components` -- Brand A component overrides
- `brand-b.site` -- Brand B site-level styles
- `brand-a.dependencies` -- Brand A specific dependencies

#### HTL Loading

```html
<sly data-sly-use.clientLib="${'/libs/granite/sightly/templates/clientlib.html'}">
    <sly data-sly-call="${clientLib.all @ categories='brand-a.site'}"/>
</sly>
```

#### SCSS / ui.frontend Organization for Multi-Brand

```
ui.frontend/
├── src/main/webpack/
│   ├── base/                     # Shared across all brands
│   │   ├── _variables.scss       # Design tokens
│   │   ├── _mixins.scss          # Reusable mixins
│   │   ├── _typography.scss
│   │   └── _grid.scss
│   ├── brands/
│   │   ├── brand-a/
│   │   │   ├── _variables.scss   # Override shared variables
│   │   │   ├── main.scss         # Brand A entry point (imports base + overrides)
│   │   │   └── components/
│   │   └── brand-b/
│   │       ├── _variables.scss
│   │       ├── main.scss
│   │       └── components/
│   └── components/               # Shared component styles
│       ├── _title.scss
│       └── _teaser.scss
```

The `aem-clientlib-generator` in `ui.frontend` compiles SCSS into clientlibs placed in `ui.apps`.

#### Style System for Brand Variations

Style System injects CSS classes on the outer div of components. Configuration:

1. Template author edits component policy in template editor
2. Adds CSS class names with author-friendly labels
3. Content authors select styles from dropdown

**CSS class naming (BEM)**:
```css
/* Layout styles -- fundamental component variations */
.cmp-teaser--hero { ... }
.cmp-teaser--card { ... }

/* Display styles -- minor variations */
.cmp-teaser--primary-color { background-color: var(--brand-primary); }
.cmp-teaser--secondary-color { background-color: var(--brand-secondary); }

/* Context-specific (same class, different effect per layout) */
.cmp-teaser--hero.cmp-teaser--primary-color { color: green; }
.cmp-teaser--card.cmp-teaser--primary-color { background-color: green; }
```

**Best practices**:
- Decouple author-facing style names from CSS class names (e.g., author sees "Green" but CSS class is `.cmp-teaser--primary-color`)
- Minimize style options to reduce complexity and QA requirements
- Expose only brand-compliant combinations
- Core Components v2+ fully support Style System

---

### 4. Editable Templates for Multi-Tenant

#### Template Folder Hierarchy

```xml
/conf/
├── brand-a/
│   └── settings/ [sling:Folder]
│       └── wcm/ [cq:Page]
│           ├── templates/ [cq:Page]     # Brand A templates
│           │   ├── article-page/
│           │   ├── landing-page/
│           │   └── product-page/
│           └── policies/ [cq:Page]      # Brand A policies
│               └── wcm/
│                   └── foundation/
│                       └── components/
├── brand-b/
│   └── settings/
│       └── wcm/
│           ├── templates/               # Brand B templates
│           └── policies/
└── global/
    └── settings/
        └── wcm/
            ├── templates/               # Shared fallback templates
            ├── policies/                # Shared fallback policies
            └── template-types/          # Shared template types
```

#### Template Inheritance

Precedence order: **current folder > parent folders > /conf/global > /apps > /libs**

Each brand folder maintains isolated templates and policies. Organization-wide defaults live in `/conf/global`.

#### Template Types

Locations:
- Out-of-box: `/libs/settings/wcm/template-types`
- Custom: `/apps/settings/wcm/template-types` or `/conf/<brand>/settings/wcm/template-types`

Template types define the page component `sling:resourceType` and root node policies governing permitted components.

#### allowedTemplates Configuration

Templates are restricted via `cq:allowedTemplates` on `jcr:content` nodes:

```xml
<!-- /content/brand-a/jcr:content -->
<jcr:content
    cq:allowedTemplates="[/conf/brand-a/settings/wcm/templates/.*]"
    jcr:primaryType="cq:PageContent"/>
```

Resolution: first non-empty `cq:allowedTemplates` found ascending the page hierarchy is matched against template path.

**Best practice**: Set `cq:allowedTemplates` at the site root with a regex pattern for simplicity.

Additional template restriction properties (regex-based):
- `allowedPaths` on template -- restricts to specific page paths
- `allowedParents` on template -- restricts parent page locations
- `allowedChildren` on parent template -- restricts child template usage

Adobe recommends using only `cq:allowedTemplates` for simplicity.

#### Template ACL Groups

| Path | Group | Permissions |
|------|-------|-------------|
| `/conf/<brand>/settings/wcm/templates` | template-authors | read, write, replicate |
| `/conf/<brand>/settings/wcm/policies` | template-authors | read, write, replicate |

#### Template Constraints

- Keep templates under 100 per instance; avoid exceeding 1000 (performance)
- Template changes automatically reflect on associated pages but not on existing content
- All content pages require `cq.shared` client library
- Avoid internationalizable content in templates; use Core Components localization

---

### 5. Sling Context-Aware Configuration (CAConfig)

#### Configuration Structure

```
/conf/<site>/sling:configs/<configuration-name>
```

Standard path for Core Component configurations:
```
/conf/brand-a/sling:configs/com.adobe.cq.wcm.core.components.models.Page
```

#### Content-to-Configuration Linkage

Content resources reference their configuration via `cq:conf` (AEM) or `sling:configRef` (Sling):

```xml
<!-- /content/brand-a/jcr:content -->
<jcr:content
    cq:conf="/conf/brand-a"
    jcr:primaryType="cq:PageContent"/>
```

#### Programmatic Access

```java
Conf conf = resource.adaptTo(Conf.class);
ValueMap imageServerSettings = conf.getItem("dam/imageserver");
String bgkcolor = imageServerSettings.get("bgkcolor", "FFFFFF");
```

The lookup is transparent -- developers do not hardcode configuration paths; the system resolves them contextually.

#### Configuration Browser

Access at **Tools > General > Configuration Browser**. Features supported per configuration:
- Context Hub Segments
- Content Fragment Models
- Editable Templates
- Cloud Configurations
- GraphQL Persistent Queries

Features cannot be unselected after configuration creation.

#### Debugging

- **ConfMgr Console** (`/system/console/conf`) -- Resolve configurations for specific content paths
- **Context-Aware Configuration Console** (`/system/console/slingcaconfig`) -- Query configurations and view properties

#### Cloud Configurations per Tenant

Each tenant can have separate cloud service configurations:
```
/conf/brand-a/settings/cloudconfigs/
├── analytics/
├── translation/
└── launch/
```

Referenced from content via `cq:conf` linkage, allowing different analytics accounts, translation providers, etc., per brand.

#### OSGi Configuration for Multi-Tenant

**Folder naming pattern**:
```
/apps/<app-name>/osgiconfig/config.<author|publish>.<dev|stage|prod>/
```

**Multi-tenant example**:
```
/apps/shared/osgiconfig/
├── config/                              # All targets (default)
├── config.author/                       # Author only
├── config.publish/                      # Publish only
├── config.author.dev/                   # Author + dev environment
└── config.publish.prod/                 # Publish + prod environment
```

**Format**: `.cfg.json` files (JSON-based, Apache Sling format).

**Run modes in AEM Cloud Service** (only these are supported):
- Service: `author`, `publish`
- Environment: `dev`, `stage`, `prod`
- No custom run modes allowed

**Environment-specific values** use Cloud Manager variables:
```json
{
    "connection.timeout": 1000,
    "api.url": "$[env:API_URL]",
    "api.key": "$[secret:API_KEY]"
}
```

Variable types:
- `$[env:VAR_NAME]` -- environment variables (max 2048 chars)
- `$[secret:SECRET_NAME]` -- secret variables (not stored in Git)
- Optional defaults: `$[env:VAR_NAME;default=fallback]`

Set variables via Cloud Manager API or CLI:
```bash
aio cloudmanager:set-environment-variables ENVIRONMENT_ID \
    --variable API_URL "https://api.example.com" \
    --secret API_KEY "my-secret-key"
```

Limit: 200 variables per environment.

---

### 6. Internationalization (i18n)

#### Language Copy Structure

```
/content/brand-a/
├── language-masters/           # Blueprint/source content
│   ├── en/                     # English master
│   ├── de/                     # German master
│   └── fr/                     # French master
├── us/                         # US Live Copy
│   └── en/
├── de/                         # Germany Live Copy
│   └── de/
└── ch/                         # Switzerland Live Copy
    ├── de/
    ├── fr/
    └── it/
```

**Language root naming**: Use ISO codes (`en`, `de`, `fr`, `en_US`, `en_GB`, `en-gb`).

**Constraint**: Only one level allowed between language master node and language roots. Deeper nesting prevents language copy recognition.

If the page name does not identify the language, AEM checks the `cq:language` property on the page.

#### i18n Dictionary Structure

Dictionaries created in the repository for component UI translations:

```
/apps/brand-a/i18n/
├── en.json                     # English dictionary
├── de.json                     # German dictionary
└── fr.json                     # French dictionary
```

**AEM Cloud Service behavior**: Runtime translation creates dictionaries in `/content/cq:i18n/<projectName>`, regardless of whether the source is in `/apps` or `/content`. Dictionaries in `/apps` or `/libs` are immutable -- cannot be translated at runtime.

**Translation projects**: i18n dictionaries can be added to Translation Projects. The translation process:
1. Creates dictionary language copies in `/content/cq:i18n`
2. Sends copies for translation (XLIFF export supported)
3. Imports translations back

#### Translation Integration

AEM supports two translation approaches:
- **Human Translation**: Content moves to translation providers, returns for import
- **Machine Translation**: Automated translation via integrated services

Users must be members of `project-administrators` group for Language Copy features.

---

### 7. URL Mapping and Domain Configuration

#### Sling Mapping (`/etc/map`)

**Structure**:
```
/etc/map/
├── http/
│   ├── branda.com/              # sling:Mapping node
│   │   └── sling:internalRedirect = "/content/brand-a"
│   └── brandb.com/
│       └── sling:internalRedirect = "/content/brand-b"
└── map.publish/                 # Publish-specific mappings
```

**Mapping node properties**:
- `sling:match` (String) -- Pattern matching request URLs
- `sling:internalRedirect` (String) -- Target path for internal redirects

**Domain-to-content mapping examples**:

| Node Path | sling:internalRedirect | Purpose |
|-----------|----------------------|---------|
| `/etc/map/http/branda.com` | `/content/brand-a` | Root content |
| `/etc/map/http/branda.com/libs` | `/libs` | Shared libraries |
| `/etc/map/http/branda.com/etc/designs` | `/etc/designs` | Designs |
| `/etc/map/http/branda.com/etc/clientlibs` | `/etc/clientlibs` | Client libraries |

Debug mappings at `/system/console/jcrresolver`.

#### Dispatcher Multi-Domain Configuration

**Virtual host per domain** (`httpd.conf`):
```apache
<VirtualHost *:80>
    ServerName branda.com
    DocumentRoot /usr/lib/apache/httpd-2.4.3/htdocs/content/brand-a
    <Directory /usr/lib/apache/httpd-2.4.3/htdocs/content/brand-a>
        <IfModule disp_apache2.c>
            SetHandler dispatcher-handler
            ModMimeUsePathInfo On
        </IfModule>
    </Directory>
</VirtualHost>
```

**Dispatcher farm per domain**:
```
/farm_branda {
    /virtualhosts { "branda.com" }
    /renders { /rend01 { /hostname "127.0.0.1" /port "4503" } }
    /filter {
        /0001 { /type "deny" /glob "*" }
        /0023 { /type "allow" /glob "*/en*" }
    }
    /cache {
        /docroot "/path/to/cache/content/brand-a"
    }
}
```

**Required server aliases**: `*.local`, `localhost`, `127.0.0.1`, `*.adobeaemcloud.net`, `*.adobeaemcloud.com`.

**Cache invalidation**: Use a dedicated flush farm with `statfileslevel` set to 2+ for domain-level `.stat` files.

#### Custom Domain Names (Cloud Manager)

Domains managed in your own CDN do not require Cloud Manager installation. They are available to AEM via `X-Forwarded-Host` header matching vhost definitions.

#### AEM Cloud Service Dispatcher Structure

```
dispatcher/
├── src/
│   ├── conf.d/                  # Apache config
│   │   ├── available_vhosts/
│   │   │   ├── brand-a.vhost
│   │   │   └── brand-b.vhost
│   │   ├── enabled_vhosts/
│   │   │   ├── brand-a.vhost -> ../available_vhosts/brand-a.vhost
│   │   │   └── brand-b.vhost -> ../available_vhosts/brand-b.vhost
│   │   └── rewrites/
│   │       ├── brand-a_rewrite.rules
│   │       └── brand-b_rewrite.rules
│   └── conf.dispatcher.d/      # Dispatcher config
│       ├── available_farms/
│       │   ├── brand-a.farm
│       │   └── brand-b.farm
│       ├── enabled_farms/
│       ├── filters/
│       └── cache/
```

#### Vanity URL Constraints

- No regex pattern support
- No uniqueness guarantees in multi-tenant setups
- **Best practice**: Use Apache `mod_rewrite` dispatcher rules for centrally-managed URL routing instead of vanity URLs

---

### 8. Multi-Tenant Project Structure (Maven)

#### Multi-Tenant Package Organization

```
all (container package)
├── common.ui.apps              # Shared immutable code
├── common.ui.config            # Shared OSGi configs
├── site-a.core                 # Site A OSGi bundle
├── site-a.ui.apps              # Site A immutable code
├── site-a.ui.config            # Site A OSGi configs
├── site-a.ui.content           # Site A mutable content
├── site-b.core                 # Site B OSGi bundle
├── site-b.ui.apps              # Site B immutable code
├── site-b.ui.config            # Site B OSGi configs
└── site-b.ui.content           # Site B mutable content
```

#### Embedding in `all/pom.xml`

```xml
<embeddeds>
    <embedded>
        <groupId>${project.groupId}</groupId>
        <artifactId>common.ui.apps</artifactId>
        <type>zip</type>
        <target>/apps/shared-packages/application/install</target>
    </embedded>
    <embedded>
        <groupId>${project.groupId}</groupId>
        <artifactId>site-a.ui.apps</artifactId>
        <type>zip</type>
        <target>/apps/site-a-packages/application/install</target>
    </embedded>
    <embedded>
        <groupId>${project.groupId}</groupId>
        <artifactId>site-a.ui.content</artifactId>
        <type>zip</type>
        <target>/apps/site-a-packages/content/install</target>
    </embedded>
</embeddeds>
```

Use `-packages` suffix to prevent destructive installation behavior.

#### Package Dependencies

- `all`: No dependencies
- `common.ui.apps`: No dependencies
- `site-a.ui.apps`: Depends on `common.ui.apps`
- `site-a.ui.content`: Depends on `site-a.ui.apps`
- `site-b.ui.apps`: Depends on `common.ui.apps`

#### Container Filter (`all/filter.xml`)

```xml
<filter root="/apps/shared-packages"/>
<filter root="/apps/site-a-packages"/>
<filter root="/apps/site-b-packages"/>
```

#### Cloud Manager Deployment

Only `all` deploys; mark others with:
```xml
<properties>
    <cloudManagerTarget>none</cloudManagerTarget>
</properties>
```

#### Git Submodules Approach

For large-scale multi-tenant with separate teams:

```
client-main/                    # Shell repository
├── .gitmodules
├── pom.xml                     # Lists all submodules as Maven modules
├── client-commons/             # Shared code (submodule)
├── client-website-a/           # Site A (submodule)
└── client-website-b/           # Site B (submodule)
```

**Limitation**: All code deploys every time -- no selective submodule deployment.

#### Repo Init for Tenant Baseline

Define tenant-specific users, service users, groups, and ACLs as code:
```
/apps/site-a/osgiconfig/config.author/
    org.apache.sling.jcr.repoinit.RepositoryInitializer-site-a.cfg.json
```

---

### Multi-Tenant Anti-Patterns Summary

| Anti-Pattern | Why to Avoid |
|-------------|-------------|
| Overlapping `filter.xml` across teams | One team's deployment erases another's content |
| Overlays (`/apps` overlaying `/libs`) | Affect entire AEM instance across all tenants |
| Vanity URLs for multi-tenant | No uniqueness guarantees |
| Shared JVM resources without planning | Memory, CPU, disk I/O impact all tenants |
| Per-tenant backups | Backup/restore spans entire repository |
| Custom run modes | Not supported in AEM Cloud Service |
| Exceeding 1000 templates | Performance degradation |
| Direct Core Component references | Use proxy components for tenant isolation |
