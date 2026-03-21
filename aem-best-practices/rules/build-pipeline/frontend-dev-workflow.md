---
title: Frontend Developer Workflow & Tooling for AEM Cloud Service
impact: HIGH
impactDescription: Inefficient frontend workflows slow development velocity and introduce deployment errors that are costly to fix in Cloud Manager pipelines
tags: frontend, workflow, sdk, proxy, webpack, debugging, clientlibs, cloud-manager, testing, git, aio-cli, feature-flags
---

## Frontend Developer Workflow & Tooling

Practical tips, tricks, and patterns for frontend development on AEM as a Cloud Service. These are the things experienced AEM developers know that save hours of debugging and rework.

---

### 1. AEM SDK Local Setup for Frontend Development

The AEM as a Cloud Service SDK Quickstart Jar provides a local author instance at `http://localhost:4502`. Frontend developers need a running local AEM instance to test component rendering, dialog interactions, and ClientLib delivery.

**Quick setup checklist:**

```bash
# Download the latest AEM SDK from Software Distribution
# https://experience.adobe.com/#/downloads/content/software-distribution/en/aemcloud.html

# Start local author instance
java -jar aem-sdk-quickstart-*.jar -p 4502

# First-time setup: deploy your project
cd /path/to/aem-project
mvn clean install -PautoInstallSinglePackage

# Frontend-only rebuild and deploy
cd ui.frontend
npm run build
cd ..
mvn clean install -PautoInstallPackage -pl ui.apps
```

**Correct -- keep SDK version in sync with Cloud Service:**
```bash
# Check your Cloud Service version in Cloud Manager > Environments
# Download the matching SDK version from Software Distribution
# Mismatch causes subtle rendering differences
```

**Incorrect -- using an outdated SDK:**
```bash
# Running a 6-month-old SDK while Cloud Service has auto-updated
# Core Components versions will differ, HTL behavior may change
# ALWAYS update your local SDK regularly
```

---

### 2. Proxy Patterns for Local Development

#### webpack-dev-server Proxy to AEM

The `ui.frontend` module supports a webpack-dev-server that proxies requests to a local AEM instance. This enables hot reloading of CSS/JS while AEM serves the HTML structure.

**webpack.dev.js configuration:**

```javascript
// ui.frontend/webpack.dev.js
module.exports = merge(common, {
    mode: 'development',
    devServer: {
        proxy: [
            {
                context: ['/content', '/etc.clientlibs', '/libs'],
                target: 'http://localhost:4502',
                auth: 'admin:admin',
                changeOrigin: true
            }
        ],
        port: 3000,
        hot: true,
        liveReload: true,
        // Serve static HTML from AEM for component context
        static: {
            directory: path.join(__dirname, 'src/main/webpack/static')
        }
    }
});
```

#### aem-site-theme-builder Proxy

For front-end pipeline projects, use the theme builder's built-in proxy:

```bash
# In your site theme project
npm install @adobe/aem-site-theme-builder

# Configure .env
AEM_URL=http://localhost:4502
AEM_SITE=my-site
AEM_PROXY_PORT=7001

# Start proxy -- serves your local CSS/JS against AEM-rendered pages
npx aem-site-theme-builder proxy
```

**Pro tip:** Set `AEM_URL` to a dev Cloud Service environment to test against real cloud content without running a local SDK:

```bash
# Point to cloud dev environment (requires auth token)
AEM_URL=https://author-p12345-e67890.adobeaemcloud.com
```

---

### 3. Hot Module Replacement with AEM

True HMR requires the webpack-dev-server proxy pattern. Without it, you must rebuild and redeploy after every CSS/JS change.

**Fast feedback loop pattern:**

```bash
# Terminal 1: Watch mode -- rebuilds ui.frontend on file changes
cd ui.frontend && npm run watch

# Terminal 2: Auto-sync built clientlibs to AEM
# Using aemsync for real-time JCR sync
npx aemsync -t http://admin:admin@localhost:4502 -w ../ui.apps/src/main/content/jcr_root
```

**Correct -- use aemsync for rapid iteration:**
```bash
# aemsync watches for file changes and pushes only diffs to AEM
npm install -g aemsync
aemsync -t http://admin:admin@localhost:4502 -w ui.apps/src/main/content/jcr_root
# Changes appear in AEM within 1-2 seconds
```

**Incorrect -- full Maven build on every change:**
```bash
# This takes 30-60 seconds per change -- kills productivity
mvn clean install -PautoInstallPackage  # DON'T do this for every CSS tweak
```

---

### 4. Content Sync/Export for Local Development

When you need realistic content locally without manually authoring pages:

```bash
# Export content package from a cloud dev environment using Package Manager
# http://localhost:4502/crx/packmgr/index.jsp

# Create a package filter for specific content paths:
# /content/my-site/en (include subtree)
# /content/dam/my-site (include subtree)
# /content/experience-fragments/my-site (include subtree)

# Download and install on local SDK
curl -u admin:admin -F file=@content-package.zip \
  -F install=true \
  http://localhost:4502/crx/packmgr/service.jsp
```

**Pro tip for teams -- create a shared content package:**

```xml
<!-- In a dedicated ui.content.sample module, define filter.xml -->
<filter root="/content/mysite/en/sample-pages" mode="replace"/>
<filter root="/content/dam/mysite/sample-assets" mode="replace"/>
<!-- Commit this package to Git so all developers have consistent test content -->
```

---

### 5. Debugging ClientLibs

These debugging tools save hours when styles or scripts are not loading as expected.

#### debugClientLibs Parameter

Append `?debugClientLibs=true` to any page URL to load all ClientLib files individually (unminified, unmerged). This reveals which specific file contains an issue.

```
http://localhost:4502/content/mysite/en.html?debugClientLibs=true
```

**What it does:** Instead of loading a single merged `clientlib-base.js`, the browser loads each source file separately with `@import` statements for CSS.

#### DumpLibs Tool

Navigate to the dumplibs console to inspect all registered ClientLibs, their categories, dependencies, and embeds:

```
http://localhost:4502/libs/granite/ui/content/dumplibs.html
```

**Use cases:**
- Find which ClientLib category a specific file belongs to
- Debug dependency ordering issues
- Identify duplicate category names causing conflicts

#### Rebuild ClientLibs Cache

When ClientLibs appear stale or corrupted:

```
http://localhost:4502/libs/granite/ui/content/dumplibs.rebuild.html
```

#### Dependency Validation

Check for missing dependencies and circular references:

```
http://localhost:4502/libs/granite/ui/content/dumplibs.validate.html
```

**Correct -- systematic debugging approach:**
```
1. Open page with ?debugClientLibs=true
2. Check browser console for 404s on individual JS/CSS files
3. Use dumplibs.html to verify category names match
4. Check css.txt/js.txt manifests for typos in file paths
5. Verify allowProxy=true is set on the ClientLib node
```

**Incorrect -- common ClientLib debugging mistakes:**
```
# Forgetting that publish instances serve via /etc.clientlibs/ (proxy)
# Missing allowProxy=true means 404 on publish but works on author

# Forgetting to rebuild after adding new files to css.txt/js.txt
# The manifest files are not auto-discovered -- files MUST be listed

# Category name typos -- no error is thrown, the library simply doesn't load
```

---

### 6. CRXDE Lite and Developer Console

#### CRXDE Lite (Local/Dev Only)

```
http://localhost:4502/crx/de/index.jsp
```

**Useful for frontend devs:**
- Inspect rendered ClientLib output: `/libs/clientlibs/` or `/etc.clientlibs/`
- Check component resource types and content structure
- Verify dialog configurations live
- Quick-test HTL script changes (local only -- never edit in production)

**Important:** CRXDE Lite is read-only on AEM Cloud Service publish tiers. On author tier, it is available only in dev environments.

#### AEM Developer Console (Cloud Service)

```
https://dev-console-ns-{pod}.adobeaemcloud.com/
```

Access via Cloud Manager > Environments > Developer Console. Useful for:
- Checking OSGi configurations affecting frontend (e.g., Sling rewriter rules)
- Viewing bundles and services status
- Querying content via repository browser

---

### 7. Using aio CLI and Cloud Manager for Frontend Pipeline

The dedicated frontend pipeline deploys only `ui.frontend` artifacts, enabling frontend teams to ship independently from backend changes.

#### Setup

```bash
# Install Adobe I/O CLI
npm install -g @adobe/aio-cli
aio plugins:install @adobe/aio-cli-plugin-cloudmanager

# Authenticate
aio auth:login

# List pipelines
aio cloudmanager:list-pipelines --programId=12345

# Trigger frontend pipeline
aio cloudmanager:start-pipeline-execution --programId=12345 --pipelineId=67890
```

#### Frontend Pipeline Build Contract

The pipeline runs `npm run build` and deploys the `dist/` folder contents. Your `package.json` must define:

```json
{
  "name": "my-site-theme",
  "scripts": {
    "build": "webpack --mode production",
    "start": "webpack-dev-server --mode development"
  }
}
```

**Correct -- configure Node.js version:**
```bash
# Set NODE_VERSION environment variable in Cloud Manager pipeline config
# Supported versions: 12, 14 (default), 16, 18, 20, 22, 23
NODE_VERSION=20
```

**Incorrect -- assuming latest Node.js is available:**
```bash
# Cloud Manager defaults to Node 14 if NODE_VERSION is not set
# Your modern npm packages may fail silently on old Node
```

#### Naming Convention Best Practices

```
Theme module name:    wknd-theme         (matches site: /content/wknd)
Source folder:        wknd-theme-sources
Pipeline name:        wknd-theme-prod    (includes environment identifier)
```

This prevents deploying themes to wrong sites or creating race conditions with multiple pipelines.

---

### 8. Feature Flags and Environment-Specific Frontend Code

#### HTL-Based Environment Detection

```html
<!-- customheaderlibs.html -->
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html">
    <sly data-sly-call="${clientlib.all @ categories='myproject.base'}" />
</sly>

<!-- Environment-specific CSS class on body for conditional styling -->
<sly data-sly-use.config="com.myproject.core.models.EnvironmentConfig">
    <meta name="environment" content="${config.runMode}" />
</sly>
```

#### WCM Mode Detection in Frontend JavaScript

```javascript
// Detect AEM authoring mode from the wcmmode cookie or URL parameter
function getWcmMode() {
    // Check URL parameter first (used with ?wcmmode=disabled)
    const urlParams = new URLSearchParams(window.location.search);
    const urlMode = urlParams.get('wcmmode');
    if (urlMode) return urlMode;

    // Check cookie (set by AEM editor: EDIT or PREVIEW)
    const cookie = document.cookie
        .split('; ')
        .find(row => row.startsWith('wcmmode='));
    return cookie ? cookie.split('=')[1] : 'disabled';
}

// Guard analytics and third-party scripts from firing in edit mode
const wcmMode = getWcmMode();
if (wcmMode === 'disabled') {
    // Only load analytics on publish (wcmmode=disabled means publish or preview)
    initAnalytics();
    initGTM();
}
```

**Correct -- conditional script loading by environment:**
```javascript
if (getWcmMode() === 'disabled') {
    // Production-only: analytics, A/B testing, chat widgets
    loadThirdPartyScripts();
}
```

**Incorrect -- loading analytics in edit mode:**
```javascript
// This fires on every author page load, polluting analytics data
// and slowing the AEM editor
initAnalytics();  // NO! Check wcmmode first
```

---

### 9. Automated Testing Patterns for AEM Frontend

#### Jest + JSDOM for ClientLib JavaScript

```javascript
// __tests__/accordion.test.js
import { Accordion } from '../src/main/webpack/components/accordion/accordion';

describe('Accordion Component', () => {
    let container;

    beforeEach(() => {
        // Simulate AEM component markup
        document.body.innerHTML = `
            <div class="cmp-accordion" data-cmp-is="accordion">
                <div class="cmp-accordion__item" data-cmp-hook-accordion="item">
                    <button class="cmp-accordion__button"
                            data-cmp-hook-accordion="button">
                        Section 1
                    </button>
                    <div class="cmp-accordion__panel"
                         data-cmp-hook-accordion="panel"
                         role="region">
                        Content 1
                    </div>
                </div>
            </div>
        `;
        container = document.querySelector('[data-cmp-is="accordion"]');
    });

    test('expands panel on button click', () => {
        const accordion = new Accordion({ element: container });
        const button = container.querySelector('.cmp-accordion__button');
        const panel = container.querySelector('.cmp-accordion__panel');

        button.click();
        expect(panel.classList.contains('cmp-accordion__panel--expanded')).toBe(true);
        expect(button.getAttribute('aria-expanded')).toBe('true');
    });

    test('collapses panel on second click', () => {
        const accordion = new Accordion({ element: container });
        const button = container.querySelector('.cmp-accordion__button');

        button.click(); // expand
        button.click(); // collapse
        expect(button.getAttribute('aria-expanded')).toBe('false');
    });
});
```

#### package.json Test Configuration

```json
{
  "scripts": {
    "test": "jest --coverage",
    "test:watch": "jest --watch"
  },
  "jest": {
    "testEnvironment": "jsdom",
    "moduleNameMapper": {
      "\\.(css|less|scss)$": "identity-obj-proxy"
    },
    "collectCoverageFrom": [
      "src/main/webpack/**/*.js",
      "!src/main/webpack/vendor/**"
    ]
  }
}
```

---

### 10. Git Workflow for AEM Projects

#### Branch Strategy with Cloud Manager

```
main (production)
  ├── develop (development environment -- auto-deploy via full-stack pipeline)
  ├── feature/JIRA-123-new-header (feature branches)
  └── release/2024.03 (optional release branches)
```

**Cloud Manager pipeline triggers:**
- **Full-stack pipeline:** Triggered on push to `main` or `develop` (configurable)
- **Frontend pipeline:** Can be triggered independently for `ui.frontend` changes
- **Config pipeline:** For dispatcher and CDN configuration changes

**Correct -- separate frontend and backend changes:**
```bash
# Frontend-only change: use frontend pipeline (deploys in ~5 minutes)
git checkout -b feature/JIRA-456-new-styles
# Make CSS/JS changes in ui.frontend/
git commit -m "feat(ui.frontend): update header styles for rebrand"
git push origin feature/JIRA-456-new-styles
# After merge: trigger frontend pipeline only
```

**Incorrect -- bundling frontend with backend changes:**
```bash
# Mixing ui.frontend + core Java changes in one PR
# Forces a full-stack pipeline (20-40 minutes) for a CSS fix
# Keep frontend and backend changes in separate PRs when possible
```

#### Frontend Pipeline Coordination

When backend changes alter component HTML output:

1. Backend team deploys new HTL to dev environment via full-stack pipeline
2. Frontend team tests against dev using `npx aem-site-theme-builder proxy` with `AEM_URL` pointing to dev
3. Frontend code must work with both old (production) and new (dev) HTML output
4. Backend deploys to production, then frontend deploys cleanup changes

This ensures zero-downtime transitions and prevents broken pages during deployment windows.
