---
title: Edge Delivery Services Sidekick, Admin & SEO
impact: HIGH
impactDescription: Sidekick configuration and SEO best practices directly affect author productivity and search visibility
tags: edge-delivery, eds, sidekick, seo, metadata, sitemap, redirects, open-graph, structured-data, admin-api
---

## EDS Sidekick, Admin & SEO

The AEM Sidekick is the primary authoring toolbar for Edge Delivery Services, providing preview, publish, and content management capabilities. SEO in EDS is managed through metadata blocks, bulk metadata spreadsheets, sitemaps, redirects, and structured data.

### Sidekick Overview

The AEM Sidekick is a Chrome extension that provides content authors with a context-aware toolbar for editing, previewing, and publishing content directly from their website pages.

#### Core Components

| Component           | Function                                                    |
|---------------------|-------------------------------------------------------------|
| Drag handle         | Repositions the toolbar on the page                         |
| Environment switcher| Toggles between production, live, and preview modes          |
| Action buttons      | Context-dependent preview, publish, edit actions             |
| Menu                | Project management, settings, plugin access                  |
| Sign-in             | Authentication for protected operations                      |

#### Operating Environments

| Environment | Label  | Description                                        |
|-------------|--------|----------------------------------------------------|
| Source       | —      | The editing view in Google Docs or Microsoft Word   |
| Preview      | Blue   | Latest changes; stakeholder review URLs             |
| Live         | Green  | Interim production (only when no production exists) |
| Production   | Green  | Public-facing website                               |

#### Publishing Workflow

Content flows: **Source** (edit) -> **Preview** (Sidekick preview action) -> **Live/Production** (Sidekick publish action).

### Bulk Preview/Publish Operations

Select multiple files in Google Drive or SharePoint to batch process:

- Preview multiple files simultaneously
- Publish multiple files at once
- Copy preview, live, or production URLs in batch
- Process media files (MP4, PDF, SVG, JPG, PNG)

The Sidekick displays operation status and enables bulk URL copying.

### Unpublishing and Deletion

Both require sign-in and appropriate roles:
- **Unpublish** — Removes content from live/production but preserves preview
- **Delete** — Permanently removes content from all environments (irreversible)

### Sidekick Configuration

Add projects by navigating to a source document or project URL and selecting "Add this project." Custom domains (production, preview, live) require manual addition.

#### Host Configuration in Site Config

```json
{
  "host": "www.example.com",
  "previewHost": "preview.example.com",
  "liveHost": "live.example.com",
  "reviewHost": "review.example.com"
}
```

### Custom Sidekick Plugins

Plugins extend the Sidekick with custom functionality. Configure them in the site configuration's `sidekick` object.

#### URL-Based Plugin (Opens External Resource)

```json
{
  "plugins": [
    {
      "id": "analytics",
      "title": "Analytics",
      "url": "/tools/sidekick/analytics.html",
      "passConfig": true,
      "passReferrer": true,
      "environments": ["preview", "live", "prod"],
      "isPalette": true,
      "paletteRect": "top: 50px; left: 50px; width: 400px; height: 300px;"
    }
  ]
}
```

#### Event-Based Plugin (Custom JavaScript)

```json
{
  "plugins": [
    {
      "id": "copy-metadata",
      "title": "Copy Metadata",
      "event": "copy-metadata",
      "environments": ["preview"],
      "pinned": true
    }
  ]
}
```

Listen for the event in your page code:

```javascript
const sk = document.querySelector('aem-sidekick');
if (sk) {
  sk.addEventListener('custom:copy-metadata', handleCopyMetadata);
} else {
  document.addEventListener('sidekick-ready', () => {
    document.querySelector('aem-sidekick')
      .addEventListener('custom:copy-metadata', handleCopyMetadata);
  }, { once: true });
}

function handleCopyMetadata(event) {
  const metadata = document.querySelector('.metadata');
  if (metadata) {
    navigator.clipboard.writeText(metadata.textContent);
  }
}
```

#### Plugin Configuration Properties

| Property         | Type     | Description                                          |
|------------------|----------|------------------------------------------------------|
| `id`             | string   | Unique identifier (required)                         |
| `title`          | string   | Button label (required)                              |
| `titleI18n`      | object   | Localized titles `{ "de": "Analytik" }`              |
| `url`            | string   | URL to open (for URL-based plugins)                  |
| `event`          | string   | Custom event name (for event-based plugins)          |
| `pinned`         | boolean  | Show in toolbar vs. overflow menu                    |
| `environments`   | array    | Target environments: dev, edit, admin, preview, live, prod |
| `excludePaths`   | array    | Glob patterns to hide plugin on certain paths        |
| `includePaths`   | array    | Glob patterns to show plugin only on certain paths   |
| `isPalette`      | boolean  | Float as a palette over content                      |
| `isPopover`      | boolean  | Center above the plugin button                       |
| `isContainer`    | boolean  | Group related actions                                |
| `isBadge`        | boolean  | Display as a decorative label                        |
| `passConfig`     | boolean  | Pass site config to plugin URL                       |
| `passReferrer`   | boolean  | Pass current page URL to plugin URL                  |
| `confirm`        | boolean  | Require confirmation before action                   |

#### Customizing Built-In Plugins

Override default plugin behavior by referencing their IDs:

```json
{
  "plugins": [
    {
      "id": "publish",
      "excludePaths": ["**/drafts/**"],
      "environments": ["preview"],
      "confirm": true
    }
  ]
}
```

Built-in plugin IDs: `preview`, `update`, `publish`.

#### Special Views for File Types

Register custom viewers for specific file types:

```json
{
  "specialViews": [
    {
      "title": "JSON Viewer",
      "path": "**.json",
      "viewer": "/tools/sidekick/json-viewer/index.html"
    }
  ]
}
```

### Sidekick Library (Block Library for Authors)

The Sidekick Library provides a visual block catalog for content authors, removing the need to memorize block variations.

#### Setup Steps

1. Create `/tools/sidekick/` directory in your content source
2. Create `library` workbook (Excel) with sheets prefixed `helix-` (e.g., `helix-blocks`)
3. Add columns: `name` and `path`
4. Create block variation documents in subdirectories
5. Reference them in the `helix-blocks` sheet

#### Library HTML Configuration

Create `tools/sidekick/library.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Block Library</title>
</head>
<body>
  <script type="module">
    import { createLibrary } from 'https://www.aem.live/tools/sidekick/library/index.js';

    const library = document.createElement('sidekick-library');
    library.config = {
      base: '/tools/sidekick/library.json',
      plugins: {
        blocks: {
          encodeImages: true,
          viewPorts: [600, 900],
        },
      },
    };
    document.body.appendChild(library);
  </script>
</body>
</html>
```

#### Block Variation Metadata

In block variation documents, add metadata tables to customize presentation:

| Property                        | Description                                |
|---------------------------------|--------------------------------------------|
| `name`                          | Custom display name                        |
| `description`                   | Block explanation for authors               |
| `type`                          | `template` or `section` grouping           |
| `searchTags`                    | Comma-separated search terms               |
| `tableHeaderBackgroundColor`    | Hex color for table headers                |
| `contentEditable`               | Boolean to disable editing                 |
| `disableCopy`                   | Boolean to hide copy button                |

#### Custom Library Plugins

Export a `decorate()` function and a default configuration:

```javascript
// tools/sidekick/library/plugins/tags.js
export async function decorate(container, data, query, context) {
  const list = document.createElement('ul');
  data.forEach((item) => {
    const li = document.createElement('li');
    li.textContent = item.tag;
    li.addEventListener('click', () => {
      navigator.clipboard.writeText(item.tag);
      context.dispatchEvent(new CustomEvent('Toast', {
        detail: { message: `Copied: ${item.tag}` },
      }));
    });
    list.appendChild(li);
  });
  container.appendChild(list);
}

export default {
  title: 'Tags',
  searchEnabled: true,
};
```

### Admin API for Site Management

The Admin API (`admin.aem.page`) provides programmatic access to site operations:

```bash
# Preview a page
curl -X POST 'https://admin.aem.page/preview/{owner}/{repo}/{branch}/path/to/page'

# Publish a page
curl -X POST 'https://admin.aem.page/live/{owner}/{repo}/{branch}/path/to/page'

# Trigger cache invalidation
curl -X POST 'https://admin.aem.page/cache/{owner}/{repo}/{branch}/path/to/page'

# Configure bulk metadata
curl -X POST 'https://admin.aem.page/config/{owner}/{repo}.json' \
  -H 'Content-Type: application/json' \
  -d '{"metadata": {"sources": ["/metadata.json"]}}'
```

---

## SEO in Edge Delivery Services

### Metadata Block for Title, Description, and OG Tags

Every EDS page should include a metadata block at the bottom of the document:

| Metadata          | Value                                              |
|-------------------|----------------------------------------------------|
| Title             | My Page Title                                      |
| Description       | A concise description of the page content          |
| Image             | /media/hero-image.jpg                              |
| og:title          | My Page Title                                      |
| og:description    | A concise description for social sharing           |
| og:image          | /media/social-preview.jpg                          |
| twitter:card      | summary_large_image                                |
| twitter:title     | My Page Title                                      |
| twitter:description | A concise description for Twitter                |
| robots            | index,follow                                       |

These values are rendered as `<meta>` tags in the page `<head>` automatically by the EDS runtime.

### Bulk Metadata

Manage metadata at scale with a `metadata` spreadsheet (or `metadata.xlsx` for SharePoint) in the project root.

#### Spreadsheet Structure

| URL            | title                | description              | image              | robots          |
|----------------|----------------------|--------------------------|--------------------|-----------------|
| **             | My Site              | Default site description | /media/default.jpg |                 |
| /blog/**       | Blog - My Site       | Latest blog articles     | /media/blog.jpg    |                 |
| /experiments/**|                      |                          |                    | noindex,nofollow|
| /drafts/**     |                      |                          |                    | noindex,nofollow|

#### Precedence Rules (Highest to Lowest)

1. Page-level metadata blocks (in the document)
2. Bulk metadata sheets (evaluated top to bottom, last match wins)
3. Default values

Use empty string values (`""`) to explicitly remove metadata for matching paths.

Multiple metadata sources can be configured via the Admin API. Later sources overwrite earlier values but cannot delete them.

### Open Graph and Twitter Card Meta Tags

Set OG and Twitter tags in either the page metadata block or bulk metadata:

```
# In page metadata block:
og:title       → <meta property="og:title" content="...">
og:description → <meta property="og:description" content="...">
og:image       → <meta property="og:image" content="...">
og:type        → <meta property="og:type" content="...">

twitter:card        → <meta name="twitter:card" content="...">
twitter:title       → <meta name="twitter:title" content="...">
twitter:description → <meta name="twitter:description" content="...">
twitter:image       → <meta name="twitter:image" content="...">
```

The `Image` metadata field automatically populates `og:image` if `og:image` is not explicitly set.

### Structured Data (JSON-LD)

EDS supports two approaches to adding Schema.org structured data:

#### Block-Based Schema (Client-Side)

Generate JSON-LD from visible page content with JavaScript. Ideal for FAQs, reviews, recipes:

```javascript
// blocks/faq/faq.js
export default function decorate(block) {
  const items = [];
  block.querySelectorAll('.faq-item').forEach((item) => {
    const question = item.querySelector('h3')?.textContent;
    const answer = item.querySelector('p')?.textContent;
    if (question && answer) {
      items.push({
        '@type': 'Question',
        name: question,
        acceptedAnswer: { '@type': 'Answer', text: answer },
      });
    }
  });

  if (items.length) {
    const script = document.createElement('script');
    script.type = 'application/ld+json';
    script.textContent = JSON.stringify({
      '@context': 'https://schema.org',
      '@type': 'FAQPage',
      mainEntity: items,
    });
    document.head.appendChild(script);
  }
}
```

#### Page-Based Schema (Metadata-Driven)

Embed structured data in page metadata for inclusion in the initial HTML. Best for product/offer data from PIM/ERP systems:

```javascript
// In scripts.js — generate JSON-LD from page metadata
function addStructuredData() {
  const title = getMetadata('og:title') || document.title;
  const description = getMetadata('og:description') || getMetadata('description');
  const image = getMetadata('og:image');

  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'WebPage',
    name: title,
    description,
    image: image ? new URL(image, window.location.origin).href : undefined,
  };

  const script = document.createElement('script');
  script.type = 'application/ld+json';
  script.textContent = JSON.stringify(jsonLd);
  document.head.appendChild(script);
}
```

Google recommends page-based schema for content that needs to be indexed quickly: "We strongly recommend using HTML for critical content that you want to be indexed quickly."

### Sitemap Generation and Configuration

EDS generates sitemaps automatically without configuration. A `sitemap.xml` is created from published documents.

#### Default Behavior

- Automatically generated `sitemap.xml` for all published pages
- Referenced from `robots.txt` for search engine discovery
- Limit: 50,000 URLs and 50MB per sitemap file

#### Custom Configuration with helix-sitemap.yaml

Place `helix-sitemap.yaml` in your project root (or manage via the configuration service as `sitemap.yaml`):

```yaml
# Simple sitemap
sitemaps:
  main:
    source: /query-index.json
    destination: /sitemap.xml

# With last modification dates
sitemaps:
  main:
    source: /query-index.json
    destination: /sitemap.xml
    lastmod: YYYY-MM-DD
```

#### Multi-Language Sitemaps with hreflang

```yaml
languages:
  en:
    source: /en/query-index.json
    destination: /sitemap-en.xml
    hreflang: en
    default: true
  fr:
    source: /fr/query-index.json
    destination: /sitemap-fr.xml
    hreflang: fr
    alternate: /fr/{path}
  de:
    source: /de/query-index.json
    destination: /sitemap-de.xml
    hreflang: de
    alternate: /de/{path}
```

The `default: true` property adds `x-default` hreflang entries for the default language.

#### Sitemap Index

For large sites, create a `sitemap-index.xml` referencing all sitemaps:

```yaml
sitemaps:
  index:
    destination: /sitemap-index.xml
```

#### Domain Customization

Set `cdn.prod.host` in project configuration to control the domain in sitemap URLs.

### Robots.txt Configuration

The `robots.txt` content is configured via the Robots Config API (part of the configuration service). Ensure your sitemap is discoverable from `robots.txt`.

Common patterns:

```
User-agent: *
Allow: /
Disallow: /drafts/
Disallow: /experiments/

Sitemap: https://www.example.com/sitemap.xml
```

### Redirect Handling

Manage redirects with a `redirects` spreadsheet (or `redirects.xlsx`) in the project root.

#### Spreadsheet Structure

| Source                  | Destination                        |
|-------------------------|------------------------------------|
| /old-page               | /new-page                          |
| /blog/2023/old-post     | /blog/archive/old-post             |
| /products/legacy        | https://newsite.com/products       |

Key behaviors:
- Only **301 permanent redirects** are supported via the spreadsheet
- Redirects take precedence over existing content at the same URL
- Only the `helix-default` sheet is processed if multiple sheets exist
- Source paths are relative to the domain; destinations can be absolute or relative
- Preview changes via Sidekick before publishing

#### Wildcard Redirects

```
Source: /old-section/*
Destination: /new-section/$1
```

The `*` captures the path suffix; `$1` inserts it in the destination.

For other redirect types (302, 307), configure at the CDN level.

### Canonical URLs

EDS automatically sets canonical URLs to the page's own URL. For custom canonicals, add metadata:

| Metadata   | Value                              |
|------------|------------------------------------|
| canonical  | https://www.example.com/main-page  |

Ensure canonical URLs return 2xx status codes (not 3xx or 4xx).

### hreflang for Multi-Language Sites

Configure hreflang in `helix-sitemap.yaml` (see Sitemap section above). For page-level hreflang, add to `head.html`:

```html
<link rel="alternate" hreflang="en" href="https://www.example.com/en/page">
<link rel="alternate" hreflang="fr" href="https://www.example.com/fr/page">
<link rel="alternate" hreflang="x-default" href="https://www.example.com/en/page">
```

Or generate dynamically in `scripts.js`:

```javascript
function addHreflang() {
  const path = window.location.pathname;
  const langs = {
    en: 'https://www.example.com/en',
    fr: 'https://www.example.com/fr',
    de: 'https://www.example.com/de',
  };

  Object.entries(langs).forEach(([lang, base]) => {
    const link = document.createElement('link');
    link.rel = 'alternate';
    link.hreflang = lang;
    link.href = `${base}${path}`;
    document.head.appendChild(link);
  });
}
```

### Favicon Configuration

Place a `favicon.ico` file in your code repository root. For multi-site setups (repoless), place individual `favicon.ico` files in each content source's root directory and publish via Sidekick or Admin API.

Use `.ico` format for maximum browser compatibility.

### 404 Page Customization

Create a `404.html` file in your GitHub repository root. This is served for any URL that does not match existing content or code resources.

```html
<!DOCTYPE html>
<html>
<head>
  <title>Page Not Found</title>
  <link rel="stylesheet" href="/styles/styles.css">
</head>
<body>
  <header></header>
  <main>
    <div class="section">
      <h1>Page Not Found</h1>
      <p>The page you are looking for does not exist.</p>
      <p><a href="/" class="button primary">Go to Homepage</a></p>
    </div>
  </main>
  <footer></footer>
  <script src="/scripts/scripts.js" type="module"></script>
</body>
</html>
```

### SEO-Friendly URL Structure

EDS URLs are derived from document names and folder structure:
- Document `My Great Blog Post` becomes `/my-great-blog-post`
- Folder `blog/2025/` + document `Post Title` becomes `/blog/2025/post-title`
- Keep URLs lowercase, hyphenated, and descriptive
- Avoid deep nesting (3 levels maximum recommended)
- Use the redirects spreadsheet when restructuring URLs

### Performance as SEO (Core Web Vitals)

EDS is designed for top Core Web Vitals scores, which directly impact search rankings:

| Metric | Target     | EDS Approach                                           |
|--------|------------|--------------------------------------------------------|
| LCP    | < 2.5s     | Server-side rendering, CDN delivery, optimized images  |
| INP    | < 200ms    | Vanilla JS, no framework overhead                      |
| CLS    | < 0.1      | Explicit image dimensions, no late-loading shifts       |

Key practices:
- Load critical CSS inline, defer non-critical styles
- Use `loading="lazy"` on below-fold images (EDS does this automatically)
- Defer third-party scripts to `delayed.js`
- Avoid adding Adobe Target or heavy martech to all pages — enable selectively via metadata
- Monitor performance via EDS Operational Telemetry (RUM) dashboard

### Sidekick Development Workflows

Two approaches for testing Sidekick customizations without affecting production:

1. **Configuration Service** — Create a temporary site copy (e.g., `mysite-dev`) for testing plugin changes
2. **Repository Branch** — Use a `dev` branch with `/tools/sidekick/config.json`; merge to main via PR when ready
