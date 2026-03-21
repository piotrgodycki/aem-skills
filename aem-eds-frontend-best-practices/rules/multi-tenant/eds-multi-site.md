---
title: Edge Delivery Services Multi-Site Architecture
impact: HIGH
impactDescription: Multi-site patterns define code reuse, theming, and content organization across brands/regions in AEM Edge Delivery Services
tags: edge-delivery, eds, multi-site, repoless, helix-5, theming, blocks, fstab, paths-json
---

## Edge Delivery Services Multi-Site Architecture

Edge Delivery Services supports multi-site deployments through the repoless architecture (Helix 5), enabling multiple sites to share a single codebase while maintaining independent content sources, configurations, and branding.

### 1. Multi-Site Approaches in EDS

#### Traditional (Helix 4) vs Repoless (Helix 5)

| Aspect | Helix 4 (Traditional) | Helix 5 (Repoless) |
|--------|----------------------|---------------------|
| Repository | One GitHub repo per site | One shared repo for all sites |
| Configuration | `fstab.yaml` in each repo | Configuration Service API |
| Content source | Defined in `fstab.yaml` | Defined via API per site |
| Path mapping | `paths.json` in repo | API or `paths.json` (API takes precedence) |
| Code sync | Per-repo AEM Code Sync | Single repo, all sites auto-update |
| Site identity | `main--<repo>--<org>.aem.page` | `main--<site>--<org>.aem.page` |

#### Repoless Core Concepts

| Concept | Definition |
|---------|-----------|
| **Organization** | Named after GitHub org; contains sites, profiles, users |
| **Profile** | Groups reusable configurations (headers, indexes, metadata) shared across sites |
| **GitHub Repository** | Single codebase; first site syncs via AEM Code Sync |
| **Site** | Combines content + code + configuration; references one profile and one content source |
| **Content Source** | SharePoint, Google Drive, or AEM Author instance |

**Key principle**: Site-level settings override profile-level settings in conflicts.

#### Setting Up a Repoless Site

**Prerequisites**: AEM Cloud Service 2025.4+, first site fully configured with Universal Editor.

**Step 1 -- Create site configuration**:
```bash
curl -X PUT https://admin.hlx.page/config/<org>/sites/<site>.json \
    -H 'x-auth-token: <token>' \
    -H 'content-type: application/json' \
    --data '{
        "code": {
            "owner": "<github-org>",
            "repo": "<shared-repo>",
            "source": {
                "type": "github",
                "url": "https://github.com/<org>/<shared-repo>"
            }
        },
        "content": {
            "source": {
                "url": "https://author-p<programID>-e<envID>.adobeaemcloud.com/bin/franklin.delivery/<org>/<site>/...",
                "type": "markup",
                "suffix": ".html"
            }
        }
    }'
```

**Step 2 -- Configure path mappings**:
```bash
curl -X POST https://admin.hlx.page/config/<org>/sites/<site>/public.json \
    --data '{
        "paths": {
            "mappings": [
                "/content/<brand>/<region>/:/",
                "/content/<brand>/<region>/configuration:/.helix/config.json"
            ],
            "includes": ["/content/<brand>/<region>/"]
        }
    }'
```

**Step 3 -- Set access control**:
```bash
curl -X POST https://admin.hlx.page/config/<org>/sites/<site>/access.json \
    --data '{
        "admin": {
            "role": {
                "admin": ["admin@example.com"],
                "config_admin": ["<tech-account-id>@techacct.adobe.com"]
            },
            "requireAuth": "auto"
        }
    }'
```

New sites become immediately available at `https://main--<site>--<org>.aem.page`.

#### Traditional fstab.yaml (Helix 4)

```yaml
mountpoints:
  /: https://author-p<programID>-e<envID>.adobeaemcloud.com/bin/franklin.delivery/<org>/<repo>/main--<repo>--<org>
```

In Helix 5, `fstab.yaml` is no longer required. Content sources are configured via the Configuration Service API.

#### Configuration Migration (Helix 4 to Helix 5)

| File-Based (Helix 4) | API-Based (Helix 5) |
|----------------------|---------------------|
| `fstab.yaml` | ContentConfig |
| `.helix/headers.xlsx` | HeadersConfig |
| `robots.txt` in repo | RobotsConfig |
| `.helix/config.xlsx` (CDN) | CDNConfig |
| `.helix/config.xlsx` (metadata) | MetadataConfig |
| `tools/sidekick/config.json` | SidekickConfig |
| `helix-query.yaml` | IndexConfig |
| `helix-sitemap.yaml` | SitemapConfig |

---

### 2. Theming in EDS

#### CSS Custom Properties Architecture

All theming is based on CSS custom properties (variables) defined at `:root` level in `styles/styles.css`:

```css
:root {
    /* Colors */
    --primary-color: rgb(255, 234, 3);
    --secondary-color: rgb(32, 32, 32);
    --background-color: #ffffff;
    --light-color: #f4f4f4;
    --text-color: #202020;
    --link-color: #035fe6;
    --link-hover-color: #136ff6;

    /* Typography */
    --heading-font-family: 'Asar', asar-normal-400-fallback, sans-serif;
    --body-font-family: 'Source Sans Pro', source-sans-pro-400-fallback, sans-serif;
    --base-font-size: 16px;
    --heading-font-size-xl: 48px;
    --heading-font-size-l: 36px;
    --heading-font-size-m: 24px;

    /* Spacing */
    --spacing-xs: 4px;
    --spacing-s: 8px;
    --spacing-m: 16px;
    --spacing-l: 32px;
    --spacing-xl: 64px;
}
```

#### Multi-Site Theming with Ensemble Pattern

**Directory structure**:
```
project-root/
├── styles/
│   ├── styles.css              # Base styles (shared across all sites)
│   ├── fonts.css               # Font definitions
│   └── themes/
│       ├── brand-a-styles.css  # Brand A variable overrides
│       └── brand-b-styles.css  # Brand B variable overrides
├── blocks/                     # Shared blocks
│   ├── header/
│   ├── footer/
│   └── hero/
└── templates/                  # Shared templates
    └── articles/
        ├── articles.css
        └── articles.js
```

**Brand A theme** (`styles/themes/brand-a-styles.css`):
```css
:root {
    --primary-color: #003049;
    --secondary-color: #d62828;
    --background-color: white;
    --text-color: #003049;
    --heading-font-family: 'Playfair Display', serif;
    --button-color: purple;
}
```

**Brand B theme** (`styles/themes/brand-b-styles.css`):
```css
:root {
    --primary-color: #2d6a4f;
    --secondary-color: #40916c;
    --background-color: #f4f4f4;
    --text-color: #262626;
    --heading-font-family: 'Inter', sans-serif;
    --button-color: blue;
}
```

#### Dynamic Theme Loading

Use metadata to trigger site-specific CSS:

```javascript
import { toClassName, getMetadata, loadCSS } from '../../scripts/aem.js';

async function loadSiteCss() {
    const theme = toClassName(getMetadata('theme'));
    switch (theme) {
        case 'brand-a':
            loadCSS(`${window.hlx.codeBasePath}/styles/themes/brand-a-styles.css`);
            break;
        case 'brand-b':
            loadCSS(`${window.hlx.codeBasePath}/styles/themes/brand-b-styles.css`);
            break;
        default:
            break;
    }
}
```

**Key principle**: Components reference CSS variables (not hardcoded values), so they automatically adapt to the active theme without individual modifications.

#### Conditional Theming by Hostname

```javascript
function getThemeByHostname() {
    const hostname = window.location.hostname;
    if (hostname.includes('brand-a')) return 'brand-a';
    if (hostname.includes('brand-b')) return 'brand-b';
    return 'default';
}
```

#### Font Configuration Pattern

`styles/fonts.css`:
```css
@font-face {
    font-family: 'Asar';
    font-weight: 400;
    font-display: swap;
    src: url('https://fonts.gstatic.com/...') format('woff2');
    unicode-range: U+0000-00FF;
}

/* Fallback font to prevent CLS */
@font-face {
    font-family: 'asar-normal-400-fallback';
    size-adjust: 95.7%;
    src: local('Arial');
}
```

Use `font-display: swap` for instant text rendering. Provide fallback fonts with `size-adjust` to match custom font metrics and prevent Cumulative Layout Shift.

---

### 3. Content Structure for Multi-Site EDS

#### AEM Author Content Tree per Site/Brand

When using AEM Author as content source with MSM integration:

```
/content/<brand>/
├── language-masters/           # Blueprint content
│   ├── en/
│   ├── de/
│   └── fr/
├── ch/                         # Switzerland (Live Copy)
│   ├── de/
│   ├── fr/
│   └── it/
├── de/                         # Germany (Live Copy)
│   └── de/
└── us/                         # US (Live Copy)
    └── en/
```

Each localized folder maps to its own EDS site:
- `/content/<brand>/ch/` -> `website-ch` (aem.live site)
- `/content/<brand>/de/` -> `website-de` (aem.live site)
- `/content/<brand>/us/` -> `website-us` (aem.live site)

**1:1 relationship**: Each AEM MSM site maps to one aem.live site with shared Git repository and codebase.

#### Path Mapping per Site

**Format**: `<internal_path>:<external_path>`

Mapping rules are applied in order; **the last matching rule wins**.

**Examples**:
```json
{
    "paths": {
        "mappings": [
            "/content/brand-a/ch/:/",
            "/content/brand-a/ch/configuration:/.helix/config.json"
        ],
        "includes": ["/content/brand-a/ch/"]
    }
}
```

Common mapping patterns:
- `../path/:/` -- Map folder to site root
- `../path:/anotherpath` -- Map to alternate path (vanity URL)
- `../path/en:/folder/` -- Map to subfolder

**Includes**: Controls which content paths replicate to Edge Delivery Services. Add DAM paths to expose assets: `/content/dam/brand-a/documents` -> `/assets/...`

#### EDS Configuration in AEM per Site

In **Tools > Cloud Services > Edge Delivery Services Configuration**:
1. Select localized folder (e.g., `/conf/brand-a/ch`)
2. Create configuration:
   - **Organization**: GitHub org name
   - **Site name**: Matches aem.live site (`website-ch`)
   - **Project type**: `aem.live with repoless config setup`

#### helix-query.yaml for Content Indexing

```yaml
indices:
  blog:
    include:
      - /blog/**
    target: /query-index.json
    properties:
      title:
        select: head > meta[property="og:title"]
        value: attribute(el, "content")
      image:
        select: head > meta[property="og:image"]
        value: attribute(el, "content")
      author:
        select: head > meta[name="author"]
        value: attribute(el, "content")
      date:
        select: head > meta[name="publication-date"]
        value: attribute(el, "content")
      description:
        select: head > meta[name="description"]
        value: attribute(el, "content")
```

**Important**: If `helix-query.yaml` exists in the repo, new indexes must be manually added to it. In Helix 5, index configuration migrates to the Configuration Service (IndexConfig).

Pages with `robots: noindex` meta tag are automatically excluded from indexes (unless a `robots` column exists in the index sheet). With custom definitions in `helix-query.yaml`, use a filter formula to enforce noindex.

**Debugging**: `aem up --print-index` displays index records locally.

---

### 4. Shared Block Libraries

#### Block Collection

The [AEM Block Collection](https://github.com/adobe/aem-block-collection) provides commonly-used blocks (Accordion, Carousel, Embed, Fragment, Modal, Quote, Search, Tabs, Table, Video). Blocks must be used on more than half of all AEM projects to qualify.

**Integration pattern**: Copy block content structure and adapt CSS/JS to project needs. Blocks are **not designed for backward compatibility** across versions.

#### Block Structure Principles

Each block is fully independent and self-contained:
```
blocks/<block-name>/
├── <block-name>.js             # Block logic
├── <block-name>.css            # Block styles
└── _<block-name>.json          # Block definition (UE only)
```

Seven requirements:
1. **Intuitive** -- Easy to author
2. **Usable** -- No external dependencies
3. **Responsive** -- Works across breakpoints
4. **Context Aware** -- Inherits CSS context (text/background colors via variables)
5. **Localizable** -- No hardcoded text
6. **Fast** -- Maintains performance standards
7. **SEO and A11y** -- Accessible and search-optimized

#### Git Subtree for Shared Plugins

Use `git subtree` (not submodules) for EDS plugins:

```bash
# Add plugin
git subtree add --squash --prefix plugins/aem-assets-plugin \
    git@github.com:adobe-rnd/aem-assets-plugin.git main

# Update plugin
git subtree pull --squash --prefix plugins/aem-assets-plugin \
    git@github.com:adobe-rnd/aem-assets-plugin.git main
```

#### Monorepo / Ensemble Approach

With repoless (Helix 5), the shared codebase IS the monorepo. All sites execute the same blocks from one GitHub repository:

```
project-root/
├── blocks/
│   ├── header/                 # Shared header block
│   │   ├── header.js
│   │   └── header.css
│   ├── footer/                 # Shared footer block
│   ├── hero/                   # Shared hero block
│   └── brand-specific-cta/     # Block used only by specific brand
├── styles/
│   ├── styles.css              # Base shared styles
│   └── themes/                 # Per-brand CSS variable overrides
├── templates/
│   └── articles/               # Shared templates
├── scripts/
│   ├── scripts.js              # Shared site scripts
│   └── delayed.js              # Shared deferred scripts
├── head.html                   # Shared <head> content
└── component-definition.json   # Compiled block definitions
```

When code updates push to the repository, all dependent repoless sites automatically use the new version.

#### Override Patterns for Site-Specific Block Behavior

**Approach 1 -- Metadata-driven behavior**:
```javascript
export default function decorate(block) {
    const theme = getMetadata('theme');
    if (theme === 'brand-a') {
        // Brand A specific rendering
        block.classList.add('brand-a-variant');
    }
    // Shared logic continues
}
```

**Approach 2 -- CSS variable-driven styling**:
```css
/* blocks/hero/hero.css */
.hero {
    background-color: var(--primary-color);
    color: var(--text-color);
    font-family: var(--heading-font-family);
    padding: var(--spacing-xl);
}
```
Each brand overrides variables in its theme CSS. No block CSS changes needed.

**Approach 3 -- Conditional CSS loading per brand**:
```css
/* styles/themes/brand-a-styles.css */
.hero .hero-cta {
    border-radius: 50px;
    text-transform: uppercase;
}
```

#### Community Block Reuse (Block Party)

The [Block Party](https://www.aem.live/developer/block-collection) is a community showcase of blocks, code snippets, and integrations that can be reused and customized.

---

### 5. Repoless Multi-Site with MSM Integration

#### Complete Setup Flow

1. **Create AEM content structure** with language-masters and Live Copies per region
2. **Set up first site** with full Universal Editor tutorial
3. **Create `/conf/<brand>/<region>` configurations** for each localized site
4. **Register repoless sites** via Configuration Service API (one per region)
5. **Configure path mappings** pointing each site to its AEM content path
6. **Set up EDS Cloud Configuration** in AEM per region
7. **Verify**: Preview and publish content, confirm rendering at regional site URLs

#### Example: Multi-Region Setup

```
AEM Content:
/content/website/
├── language-masters/en, /de, /fr, /it
├── ch/de, /fr, /it, /en        -> aem.live site: website-ch
├── de/de, /en                   -> aem.live site: website-de
└── us/en                        -> aem.live site: website-us

EDS Sites (shared code from one GitHub repo):
https://main--website-ch--org.aem.page  (Swiss site)
https://main--website-de--org.aem.page  (German site)
https://main--website-us--org.aem.page  (US site)
```

#### Verification

Check configuration at: `https://<branch>--<site>--<org>.aem.page/config.json`

Read configuration via API: `GET https://admin.hlx.page/config/<org>/sites/<site>.json`

---

### Multi-Site Anti-Patterns in EDS

| Anti-Pattern | Why to Avoid |
|-------------|-------------|
| Separate GitHub repos per regional variant | Code duplication; sync overhead; use repoless instead |
| Hardcoded brand colors/fonts in block CSS | Prevents theme switching; use CSS variables |
| Hardcoded text in blocks | Breaks localization; use content-driven text |
| Modifying `fstab.yaml` for multi-site in Helix 5 | Ignored when Configuration Service is active |
| Single `paths.json` for multiple sites | Each repoless site needs its own path mapping via API |
| Heavy JavaScript frameworks for theming | Violates EDS performance principles; use CSS variables |
| Copying blocks between repos | Use repoless shared codebase or git subtree |
