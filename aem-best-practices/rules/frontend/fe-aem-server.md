---
title: fe-aem-server — Local HTL Development Server
impact: HIGH
impactDescription: fe-aem-server eliminates the HTL conversion step — frontend developers write production-ready HTL locally without a running AEM instance
tags: fe-aem-server, htl, local-dev, webpack, vite, sightly, aem, node, development-workflow
---

## fe-aem-server — Local HTL Development Server

`@kele23/fe-aem-server` is a Node.js Express server that renders HTL (Sightly) templates locally using `@adobe/htlengine`. Frontend developers write HTL directly — structured exactly as AEM JCR expects — and preview in the browser without a running AEM SDK.

---

### 1. The Problem It Solves

Traditional AEM frontend workflow:

```
FE writes HTML/Handlebars → Manually converts to HTL → Deploys to AEM SDK → Tests
          ↑ Every CSS class change requires conversion ↑
```

With fe-aem-server:

```
FE writes HTL directly → fe-aem-server renders locally → Deploys to AEM (zero changes)
```

- No HTML-to-HTL conversion step
- No running AEM instance needed for frontend development
- Hot reloading via Webpack or Vite integration
- HTL files are production-ready from the start

---

### 2. Installation

```bash
npm install --save-dev @kele23/fe-aem-server
```

**package.json scripts:**

```json
{
  "scripts": {
    "dev": "fe-aem-server --webpack-config webpack.dev.js --server-config server.config.json",
    "dev:vite": "fe-aem-server --vite-config vite.config.js --server-config server.config.json"
  }
}
```

---

### 3. Server Configuration

```json
// server.config.json
{
  "port": 4200,
  "contentRepos": [
    {
      "type": "file",
      "path": "./content",
      "mountPoint": "/content/mysite"
    }
  ],
  "componentsPath": "./ui.apps/src/main/content/jcr_root/apps/mysite/components",
  "modelBinding": "model"
}
```

**Configuration options:**

| Option | Description | Default |
|--------|-------------|---------|
| `port` | Dev server port | 4200 |
| `contentRepos` | Content providers (file-based or remote) | Required |
| `contentRepos[].type` | Provider type (`file` for local) | `file` |
| `contentRepos[].path` | Local content directory path | Required |
| `contentRepos[].mountPoint` | JCR path prefix | Required |
| `componentsPath` | Path to component HTL files | Required |
| `modelBinding` | Name of the data binding in HTL (replaces `data-sly-use.model`) | `model` |

---

### 4. Project Structure

Mirror the AEM JCR structure so HTL files work in both environments:

```
project/
├── ui.apps/src/main/content/jcr_root/apps/mysite/
│   └── components/
│       ├── hero/
│       │   ├── hero.html              # HTL template
│       │   └── _cq_dialog/
│       │       └── .content.xml       # Dialog (AEM only)
│       ├── teaser/
│       │   └── teaser.html
│       ├── tabs/
│       │   └── tabs.html
│       └── page/
│           ├── page.html              # Page template
│           ├── header.html
│           └── footer.html
├── content/                            # Mock content for fe-aem-server
│   └── mysite/
│       └── en/
│           ├── .content.json          # Page properties
│           └── jcr:content/
│               └── root/
│                   ├── .content.json  # Component data
│                   ├── hero/
│                   │   └── .content.json
│                   └── teaser/
│                       └── .content.json
├── ui.frontend/                        # CSS/JS source
│   ├── src/
│   │   ├── main.scss
│   │   └── main.ts
│   ├── webpack.dev.js
│   └── webpack.common.js
├── server.config.json
└── package.json
```

---

### 5. Mock Content Files

Create JSON files that simulate JCR node data (substitutes for Sling Models):

```json
// content/mysite/en/.content.json — page properties
{
  "jcr:primaryType": "cq:Page",
  "jcr:title": "Home",
  "jcr:description": "Welcome to our site",
  "sling:resourceType": "mysite/components/page"
}

// content/mysite/en/jcr:content/root/hero/.content.json — hero component data
{
  "sling:resourceType": "mysite/components/hero",
  "title": "Welcome to Our Site",
  "subtitle": "Discover amazing content",
  "backgroundImage": "/content/dam/mysite/hero-bg.jpg",
  "ctaLabel": "Learn More",
  "ctaLink": "/en/about",
  "overlayOpacity": 0.4
}

// content/mysite/en/jcr:content/root/teaser/.content.json
{
  "sling:resourceType": "mysite/components/teaser",
  "title": "Featured Article",
  "description": "Lorem ipsum dolor sit amet, consectetur adipiscing elit.",
  "image": "/content/dam/mysite/featured.jpg",
  "link": "/en/articles/featured"
}
```

---

### 6. HTL Templates with fe-aem-server

The `model` binding (configurable name) provides access to mock content data:

```html
<!-- components/hero/hero.html -->
<section class="cmp-hero" style="background-image: url('${model.backgroundImage}')">
  <div class="cmp-hero__overlay" style="opacity: ${model.overlayOpacity || 0.4}"></div>
  <div class="cmp-hero__content">
    <h1 class="cmp-hero__title">${model.title}</h1>
    <p class="cmp-hero__subtitle" data-sly-test="${model.subtitle}">
      ${model.subtitle}
    </p>
    <a class="cmp-hero__cta" href="${model.ctaLink}" data-sly-test="${model.ctaLabel}">
      ${model.ctaLabel}
    </a>
  </div>
</section>
```

```html
<!-- components/page/page.html — page wrapper -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>${model.title || 'My Site'}</title>
  <link rel="stylesheet" href="/clientlibs/clientlib-site.css">
</head>
<body>
  <sly data-sly-resource="${'header' @ resourceType='mysite/components/page/header'}" />
  <main>
    <sly data-sly-resource="${'root' @ resourceType='mysite/components/container'}" />
  </main>
  <sly data-sly-resource="${'footer' @ resourceType='mysite/components/page/footer'}" />
  <script src="/clientlibs/clientlib-site.js"></script>
</body>
</html>
```

---

### 7. Webpack Integration

```javascript
// ui.frontend/webpack.dev.js
const { merge } = require('webpack-merge');
const common = require('./webpack.common.js');

module.exports = merge(common, {
  mode: 'development',
  devtool: 'eval-source-map',
  output: {
    // fe-aem-server serves built assets from this path
    publicPath: '/clientlibs/',
  },
});
```

**Run:**

```bash
# fe-aem-server handles both HTL rendering AND Webpack HMR
npm run dev

# Open http://localhost:4200/content/mysite/en.html
```

---

### 8. Vite Integration

```javascript
// vite.config.js
import { defineConfig } from 'vite';

export default defineConfig({
  root: './ui.frontend/src',
  build: {
    outDir: '../../dist',
  },
  server: {
    // fe-aem-server proxies Vite for HMR
    hmr: true,
  },
});
```

```bash
npm run dev:vite
```

---

### 9. Development Workflow

```
1. FE developer creates/edits HTL in components/
2. Creates mock content JSON in content/
3. Runs `npm run dev` — fe-aem-server starts
4. Opens browser → sees rendered HTL with live CSS/JS reload
5. Iterates on markup, styles, and behavior
6. When done, commits — HTL files deploy directly to AEM with zero modification
```

**Multi-page setup:**

```json
// content/mysite/en/about/.content.json
{
  "jcr:primaryType": "cq:Page",
  "jcr:title": "About Us",
  "sling:resourceType": "mysite/components/page"
}
```

Navigate to `http://localhost:4200/content/mysite/en/about.html` — each mock content directory is a page.

---

### 10. Comparison with Other Dev Approaches

| Approach | Requires AEM SDK | HTL Support | Hot Reload | Effort |
|----------|-----------------|-------------|------------|--------|
| **fe-aem-server** | No | Full (htlengine) | Yes (Webpack/Vite) | Low |
| AEM local SDK | Yes (8GB RAM) | Full | No (deploy each time) | High |
| Webpack proxy to AEM | Yes | Full (proxied) | CSS/JS only | Medium |
| Storybook | No | No (React/HTML only) | Yes | Medium |

**Best combo**: fe-aem-server for daily FE development → AEM SDK for final integration testing → Cloud Manager for deployment.

---

### 11. Limitations

- **No Sling Model logic** — the `model` binding is flat JSON, not computed Java; complex model logic (formatted dates, conditional computed fields) must be tested on the real AEM SDK
- **No Sling resource resolution** — `data-sly-resource` works for components in the file tree, but dynamic resource type resolution is simplified
- **No author mode simulation** — no page editor, no drag-drop; use AEM SDK for author testing
- **No WCM mode** — `wcmmode.edit` is always false in fe-aem-server
- **Not a replacement for AEM** — it's a frontend development accelerator, not a full AEM emulator

---

### Anti-Patterns

- **Skipping AEM SDK testing entirely** — fe-aem-server is for FE velocity; always verify on the SDK before deploying
- **Complex mock content** — keep mock JSONs simple; if you need complex computed data, that's a Sling Model concern
- **Different folder structure from AEM JCR** — the whole point is parity; diverging structures means manual conversion
- **Using fe-aem-server for content authoring testing** — use the AEM SDK for anything involving dialogs, policies, or templates
- **Committing mock content to the AEM deployment** — content/ is dev-only; exclude from the Maven build
