---
title: Edge Delivery Services Testing & Quality Assurance
impact: HIGH
impactDescription: EDS sites must maintain a perfect Lighthouse score of 100 — testing and linting failures block pull request merges and degrade site performance
tags: edge-delivery, eds, testing, lighthouse, eslint, stylelint, playwright, accessibility, visual-regression, rum, aem-cli, blocks, quality
---

## Edge Delivery Services Testing & Quality Assurance

EDS enforces quality through automated linting, PageSpeed Insights checks on every pull request, and Real User Monitoring in production. Every EDS site can and should achieve a perfect Lighthouse score of 100.

---

### 1. Testing Block `decorate()` Functions with JSDOM

Block JavaScript exports a single `decorate()` function that receives a DOM element. Test it by creating a DOM fixture matching the EDS block table structure, calling `decorate()`, and asserting the resulting DOM.

**Correct -- unit testing a block with JSDOM (Jest):**

```javascript
// test/blocks/cards/cards.test.js
import { JSDOM } from 'jsdom';
import fs from 'fs';
import path from 'path';

// Helper: create a minimal EDS-like DOM environment
function createBlockDOM(html) {
  const dom = new JSDOM(`<!DOCTYPE html><html><body>${html}</body></html>`, {
    url: 'http://localhost:3000',
  });
  return dom.window.document;
}

describe('Cards Block', () => {
  let decorate;

  beforeAll(async () => {
    // Dynamically import the block module
    const module = await import('../../../blocks/cards/cards.js');
    decorate = module.default;
  });

  test('decorates card items with proper structure', () => {
    const doc = createBlockDOM(`
      <div class="cards">
        <div>
          <div><picture><img src="/media_abc123.jpeg" alt="Card image"></picture></div>
          <div>
            <h3>Card Title</h3>
            <p>Card description text</p>
          </div>
        </div>
        <div>
          <div><picture><img src="/media_def456.jpeg" alt="Second card"></picture></div>
          <div>
            <h3>Second Card</h3>
            <p>Another description</p>
          </div>
        </div>
      </div>
    `);

    const block = doc.querySelector('.cards');
    decorate(block);

    // Assert the block was decorated correctly
    const cards = block.querySelectorAll('.cards-card');
    expect(cards.length).toBe(2);

    const firstCard = cards[0];
    expect(firstCard.querySelector('.cards-card-image')).not.toBeNull();
    expect(firstCard.querySelector('.cards-card-body')).not.toBeNull();
    expect(firstCard.querySelector('h3').textContent).toBe('Card Title');
  });

  test('handles empty block gracefully', () => {
    const doc = createBlockDOM('<div class="cards"></div>');
    const block = doc.querySelector('.cards');

    // Should not throw
    expect(() => decorate(block)).not.toThrow();
  });
});
```

**Jest configuration for EDS projects (`jest.config.js`):**

```javascript
export default {
  testEnvironment: 'jsdom',
  roots: ['<rootDir>/test'],
  testMatch: ['**/*.test.js'],
  transform: {},
  // EDS uses native ES modules
  extensionsToTreatAsEsm: [],
  moduleNameMapper: {
    // Mock aem.js utilities if needed
    '^../../scripts/aem.js$': '<rootDir>/test/mocks/aem.js',
  },
};
```

**Mock for EDS runtime utilities (`test/mocks/aem.js`):**

```javascript
// Mock the core EDS utility functions used by blocks
export function createOptimizedPicture(src, alt = '', eager = false, breakpoints = [{ media: '(min-width: 600px)', width: '2000' }, { width: '750' }]) {
  const picture = document.createElement('picture');
  breakpoints.forEach((br) => {
    const source = document.createElement('source');
    if (br.media) source.setAttribute('media', br.media);
    source.setAttribute('srcset', `${src}?width=${br.width}&format=webply&optimize=medium`);
    picture.appendChild(source);
  });
  const img = document.createElement('img');
  img.src = `${src}?width=${breakpoints[breakpoints.length - 1].width}&format=webply&optimize=medium`;
  img.alt = alt;
  img.loading = eager ? 'eager' : 'lazy';
  picture.appendChild(img);
  return picture;
}

export function decorateIcons(element) {
  // no-op for tests
}

export function decorateBlock(block) {
  // no-op for tests
}

export function loadBlock(block) {
  return Promise.resolve();
}

export function loadCSS(href) {
  return Promise.resolve();
}
```

---

### 2. Creating DOM Fixtures That Match EDS Block Structure

EDS blocks are authored as tables in documents and delivered as a specific DOM structure. Test fixtures must match this structure exactly.

**How EDS transforms authored content to DOM:**

```
Authored table:        Delivered DOM:
┌───────────┐          <div class="columns">
│ Columns   │ ──►        <div>           ← row
├─────┬─────┤              <div>         ← cell (col 1)
│ A   │ B   │                <p>A</p>
├─────┼─────┤              </div>
│ C   │ D   │              <div>         ← cell (col 2)
└─────┴─────┘                <p>B</p>
                           </div>
                         </div>
                         <div>           ← row 2
                           <div><p>C</p></div>
                           <div><p>D</p></div>
                         </div>
                       </div>
```

**Correct -- DOM fixture factory for tests:**

```javascript
// test/helpers/block-fixture.js

/**
 * Creates a DOM element matching EDS block structure.
 * @param {string} blockName - The block name (e.g., 'cards', 'hero')
 * @param {string[][]} rows - 2D array of cell HTML content
 * @param {string[]} variants - Block variant names (e.g., ['highlight', 'wide'])
 * @returns {HTMLElement} The block element
 */
export function createBlockFixture(blockName, rows = [], variants = []) {
  const block = document.createElement('div');
  const classes = [blockName, ...variants, 'block'];
  block.className = classes.join(' ');
  block.setAttribute('data-block-name', blockName);
  block.setAttribute('data-block-status', 'loading');

  rows.forEach((cells) => {
    const row = document.createElement('div');
    cells.forEach((cellContent) => {
      const cell = document.createElement('div');
      cell.innerHTML = cellContent;
      row.appendChild(cell);
    });
    block.appendChild(row);
  });

  return block;
}

// Usage in test:
import { createBlockFixture } from '../helpers/block-fixture.js';

test('hero block with image and text', () => {
  const block = createBlockFixture('hero', [
    [
      '<picture><img src="/media_hero.jpeg" alt="Hero"></picture>',
      '<h1>Main Heading</h1><p>Subtitle text</p>',
    ],
  ]);

  document.body.appendChild(block);
  decorate(block);

  expect(block.querySelector('h1').textContent).toBe('Main Heading');
});
```

---

### 3. Testing Block Options and Variants

EDS blocks support variants specified in the block name (e.g., `Columns (wide)` becomes class `columns wide`). Tests should cover each variant.

**Correct -- testing block variants:**

```javascript
// test/blocks/columns/columns.test.js
import { createBlockFixture } from '../../helpers/block-fixture.js';

describe('Columns Block', () => {
  let decorate;

  beforeAll(async () => {
    const module = await import('../../../blocks/columns/columns.js');
    decorate = module.default;
  });

  test('default variant applies equal column widths', () => {
    const block = createBlockFixture('columns', [
      ['<p>Column A</p>', '<p>Column B</p>'],
    ]);

    document.body.appendChild(block);
    decorate(block);

    const cols = block.querySelectorAll(':scope > div > div');
    expect(cols.length).toBe(2);
    // No special width class for default variant
    expect(block.classList.contains('wide')).toBe(false);
  });

  test('wide variant applies correct class', () => {
    const block = createBlockFixture('columns', [
      ['<p>Main content</p>', '<p>Sidebar</p>'],
    ], ['wide']);

    document.body.appendChild(block);
    decorate(block);

    expect(block.classList.contains('wide')).toBe(true);
  });

  test('handles single column gracefully', () => {
    const block = createBlockFixture('columns', [
      ['<p>Only column</p>'],
    ]);

    document.body.appendChild(block);
    expect(() => decorate(block)).not.toThrow();
  });

  afterEach(() => {
    document.body.innerHTML = '';
  });
});
```

---

### 4. Lighthouse CI for EDS Performance Regression

EDS enforces a Lighthouse score of 100 on every pull request via the AEM Code Sync GitHub App, which runs Google PageSpeed Insights automatically. You can also run Lighthouse CI locally and in your own CI pipelines.

**Correct -- Lighthouse CI configuration for EDS (`lighthouserc.js`):**

```javascript
module.exports = {
  ci: {
    collect: {
      url: [
        'https://main--mysite--myorg.aem.live/',
        'https://main--mysite--myorg.aem.live/about',
        'https://main--mysite--myorg.aem.live/blog/sample-post',
      ],
      numberOfRuns: 3,
      settings: {
        // EDS sites should hit 100 on all categories
        preset: 'desktop',
      },
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 1.0 }],
        'categories:accessibility': ['error', { minScore: 1.0 }],
        'categories:best-practices': ['error', { minScore: 1.0 }],
        'categories:seo': ['error', { minScore: 1.0 }],
        // Core Web Vitals
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 200 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

**GitHub Actions workflow for Lighthouse CI:**

```yaml
name: Lighthouse CI
on: [pull_request]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v12
        with:
          urls: |
            https://${{ github.head_ref }}--mysite--myorg.aem.page/
          configPath: ./lighthouserc.js
          uploadArtifacts: true
```

**EDS three-phase loading model (E-L-D) -- critical for maintaining score 100:**

| Phase | What loads | Impact |
|-------|-----------|--------|
| **Eager (E)** | LCP content, critical CSS, above-fold images | Must stay under 100kb aggregate |
| **Lazy (L)** | Below-fold blocks, remaining images | No TBT impact |
| **Delayed (D)** | Third-party scripts, analytics, consent | Loads 3+ seconds after LCP |

---

### 5. AEM CLI for Local Development and Testing

The `aem` CLI provides a local development server with hot reload and a reverse proxy to EDS preview content. It is the primary tool for testing blocks during development.

**Correct -- local development workflow:**

```bash
# Install the AEM CLI globally
npm install -g @adobe/aem-cli

# Start local dev server (from project root)
aem up

# Server starts at http://localhost:3000
# - Serves local JS/CSS from your working directory
# - Proxies content from https://main--mysite--myorg.aem.page
# - Hot reloads on file changes in blocks/ and styles/
```

**Testing against specific branches and environments:**

```bash
# Test against a feature branch preview
aem up --url https://feature-branch--mysite--myorg.aem.page

# Test against the live (production CDN) environment
aem up --url https://main--mysite--myorg.aem.live
```

**Using the `drafts/` folder for isolated testing:**

Create test documents in a `drafts/` folder to test blocks without affecting published content. Content in `drafts/` is excluded from sitemaps and search indexing.

```
SharePoint or Google Drive:
└── mysite/
    └── drafts/
        └── block-testing/
            ├── cards-test        ← Test all Cards variants here
            ├── hero-test         ← Test Hero component
            └── columns-test      ← Test Columns layout
```

Access at: `http://localhost:3000/drafts/block-testing/cards-test`

---

### 6. Testing Against Preview and Live URLs

EDS provides three environments with distinct URLs for testing at different stages.

**Environment URLs and their purpose:**

| Environment | URL Pattern | Purpose |
|-------------|-------------|---------|
| Local | `http://localhost:3000/` | Development with hot reload |
| Preview (`.aem.page`) | `https://<branch>--<site>--<org>.aem.page/` | Test content + code before publish |
| Live (`.aem.live`) | `https://<branch>--<site>--<org>.aem.live/` | CDN-cached production content |

**Correct -- testing the full publish cycle:**

```bash
# 1. Test locally first
aem up
# Browse http://localhost:3000/path/to/page

# 2. Push code to feature branch, preview content
# Preview URL updates automatically on push
# https://feature-xyz--mysite--myorg.aem.page/path/to/page

# 3. After merge to main, verify on live
# https://main--mysite--myorg.aem.live/path/to/page

# 4. Verify with PageSpeed Insights
npx psi https://main--mysite--myorg.aem.live/ --strategy=mobile
```

---

### 7. Visual Regression Testing for EDS Blocks

Visual regression testing catches unintended style changes in blocks across breakpoints.

**Correct -- BackstopJS configuration for EDS blocks:**

```json
{
  "id": "eds-visual-regression",
  "viewports": [
    { "label": "mobile", "width": 375, "height": 812 },
    { "label": "tablet", "width": 768, "height": 1024 },
    { "label": "desktop", "width": 1440, "height": 900 }
  ],
  "scenarios": [
    {
      "label": "Hero Block",
      "url": "https://main--mysite--myorg.aem.page/drafts/visual-tests/hero",
      "selectors": [".hero"],
      "delay": 3000,
      "misMatchThreshold": 0.1
    },
    {
      "label": "Cards Block",
      "url": "https://main--mysite--myorg.aem.page/drafts/visual-tests/cards",
      "selectors": [".cards"],
      "delay": 3000,
      "misMatchThreshold": 0.1
    },
    {
      "label": "Full Page - Home",
      "url": "https://main--mysite--myorg.aem.page/",
      "selectors": ["document"],
      "delay": 5000,
      "misMatchThreshold": 0.2
    }
  ],
  "paths": {
    "bitmaps_reference": "test/visual/reference",
    "bitmaps_test": "test/visual/results"
  },
  "engine": "playwright"
}
```

**Correct -- Percy integration with GitHub Actions for EDS:**

```yaml
name: Visual Regression
on: [pull_request]

jobs:
  percy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Percy Snapshots
        env:
          PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}
        run: |
          npx @percy/cli snapshot \
            --base-url "https://${{ github.head_ref }}--mysite--myorg.aem.page" \
            percy-snapshots.yml
```

---

### 8. Linting: ESLint and Stylelint Configuration

EDS projects use **ESLint with airbnb-base** and **Stylelint** as standard linting. The AEM Code Sync GitHub App runs these checks automatically on every pull request. Do not submit PRs with linting errors.

**Correct -- standard EDS ESLint configuration (`.eslintrc.json`):**

```json
{
  "root": true,
  "extends": "airbnb-base",
  "env": {
    "browser": true,
    "es2022": true
  },
  "parserOptions": {
    "ecmaVersion": "latest",
    "sourceType": "module"
  },
  "rules": {
    "import/extensions": ["error", "always", { "ignorePackages": true }],
    "no-param-reassign": ["error", { "props": false }],
    "linebreak-style": ["error", "unix"]
  }
}
```

**Correct -- standard EDS Stylelint configuration (`.stylelintrc.json`):**

```json
{
  "extends": "stylelint-config-standard",
  "rules": {
    "selector-class-pattern": null,
    "no-descending-specificity": null
  }
}
```

**Running linters locally before pushing:**

```bash
# Run ESLint on all JS files
npx eslint blocks/ scripts/ --ext .js

# Run Stylelint on all CSS files
npx stylelint "blocks/**/*.css" "styles/**/*.css"

# Fix auto-fixable issues
npx eslint blocks/ scripts/ --ext .js --fix
npx stylelint "blocks/**/*.css" "styles/**/*.css" --fix
```

**Incorrect -- changing linting configuration for personal preference:**

```json
// WRONG: Do NOT modify linting rules for personal preference.
// Adobe's docs state: "Personal preference is not a good reason"
// Changing linting makes it hard to move developers between projects
// and breaks compatibility with AEM boilerplate and block collections.
{
  "extends": "airbnb-base",
  "rules": {
    "semi": ["error", "never"],
    "indent": ["error", 4],
    "quotes": ["error", "double"]
  }
}
```

---

### 9. Accessibility Testing for EDS Blocks

EDS sites must score 100 on Lighthouse accessibility. The AEM Code Sync bot checks PageSpeed Insights (which includes accessibility) on every PR.

**Correct -- axe-core testing for individual blocks:**

```javascript
// test/a11y/blocks.a11y.test.js
import { JSDOM } from 'jsdom';
import { configureAxe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

const axe = configureAxe({
  rules: {
    // EDS blocks are fragments, not full pages
    'html-has-lang': { enabled: false },
    'landmark-one-main': { enabled: false },
    'page-has-heading-one': { enabled: false },
  },
});

describe('Block Accessibility', () => {
  test('cards block has no a11y violations', async () => {
    const dom = new JSDOM(`
      <div class="cards">
        <ul>
          <li class="cards-card">
            <div class="cards-card-image">
              <picture><img src="/image.jpg" alt="Card image description"></picture>
            </div>
            <div class="cards-card-body">
              <h3>Card Title</h3>
              <p>Card description</p>
            </div>
          </li>
        </ul>
      </div>
    `);

    const results = await axe(dom.window.document.body);
    expect(results).toHaveNoViolations();
  });

  test('navigation block has proper ARIA attributes', async () => {
    const dom = new JSDOM(`
      <nav class="nav" aria-label="Main navigation">
        <ul>
          <li><a href="/">Home</a></li>
          <li>
            <a href="/products" aria-expanded="false" aria-haspopup="true">Products</a>
            <ul aria-hidden="true">
              <li><a href="/products/a">Product A</a></li>
            </ul>
          </li>
        </ul>
      </nav>
    `);

    const results = await axe(dom.window.document.body);
    expect(results).toHaveNoViolations();
  });
});
```

**Correct -- Playwright accessibility test for full EDS pages:**

```javascript
// test/a11y/page.a11y.spec.js
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Page Accessibility', () => {
  test('home page passes WCAG 2.1 AA', async ({ page }) => {
    await page.goto('https://main--mysite--myorg.aem.page/');
    // Wait for delayed phase to complete (3+ seconds)
    await page.waitForTimeout(4000);

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();

    expect(results.violations).toEqual([]);
  });

  test('each block type passes accessibility scan', async ({ page }) => {
    await page.goto('https://main--mysite--myorg.aem.page/drafts/a11y-test');
    await page.waitForTimeout(4000);

    const blocks = ['hero', 'cards', 'columns', 'tabs', 'accordion'];

    for (const blockName of blocks) {
      const results = await new AxeBuilder({ page })
        .include(`.${blockName}`)
        .withTags(['wcag2a', 'wcag2aa'])
        .analyze();

      expect(results.violations,
        `Block "${blockName}" has accessibility violations`
      ).toEqual([]);
    }
  });
});
```

---

### 10. Testing JSON Block Definitions

EDS XWalk projects (Universal Editor) define blocks via JSON files (`_blockname.json`). Validate these definitions to prevent authoring errors.

**Correct -- JSON schema validation for block definitions:**

```javascript
// test/models/block-definitions.test.js
import fs from 'fs';
import path from 'path';
import { glob } from 'glob';

describe('Block Definitions', () => {
  const blockJsonFiles = glob.sync('blocks/**/_*.json');

  test.each(blockJsonFiles)('%s is valid JSON', (filePath) => {
    const content = fs.readFileSync(filePath, 'utf-8');
    expect(() => JSON.parse(content)).not.toThrow();
  });

  test.each(blockJsonFiles)('%s has required definition fields', (filePath) => {
    const def = JSON.parse(fs.readFileSync(filePath, 'utf-8'));

    // Block definition must have these top-level keys
    expect(def).toHaveProperty('definitions');

    def.definitions.forEach((definition) => {
      expect(definition).toHaveProperty('title');
      expect(definition).toHaveProperty('id');
      expect(definition).toHaveProperty('plugins');

      // Each definition must reference a model
      if (definition.model) {
        expect(typeof definition.model).toBe('string');
      }
    });
  });

  test.each(blockJsonFiles)('%s models have valid field types', (filePath) => {
    const def = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
    const validFieldTypes = [
      'text', 'richtext', 'image', 'reference', 'aem-content',
      'aem-tag', 'boolean', 'number', 'date-time', 'select',
      'multiselect', 'tab',
    ];

    def.definitions.forEach((definition) => {
      if (definition.model && definition.model.fields) {
        definition.model.fields.forEach((field) => {
          if (field.component) {
            expect(validFieldTypes).toContain(field.component);
          }
        });
      }
    });
  });
});
```

---

### 11. End-to-End Testing EDS Pages with Playwright

Playwright is ideal for E2E testing of EDS pages because it handles the async loading phases (Eager, Lazy, Delayed) and supports testing across browsers.

**Correct -- Playwright configuration for EDS:**

```javascript
// playwright.config.js
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './test/e2e',
  timeout: 30000,
  retries: 1,
  reporter: [['html', { open: 'never' }]],
  use: {
    baseURL: process.env.EDS_BASE_URL || 'https://main--mysite--myorg.aem.page',
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
  },
  projects: [
    { name: 'Desktop Chrome', use: { ...devices['Desktop Chrome'] } },
    { name: 'Mobile Safari', use: { ...devices['iPhone 13'] } },
  ],
});
```

**Correct -- E2E tests for EDS pages with page object pattern:**

```javascript
// test/e2e/pages/EDSPage.js
export class EDSPage {
  constructor(page) {
    this.page = page;
  }

  async goto(path = '/') {
    await this.page.goto(path);
    // Wait for EDS lazy phase to complete
    await this.page.waitForFunction(() => {
      return document.querySelector('main > div:last-child')
        ?.getAttribute('data-block-status') === 'loaded'
        || document.readyState === 'complete';
    }, { timeout: 10000 });
  }

  async waitForDelayedPhase() {
    // Delayed phase loads 3+ seconds after LCP
    await this.page.waitForTimeout(4000);
  }

  async getBlockByName(blockName) {
    return this.page.locator(`.${blockName}[data-block-status="loaded"]`);
  }
}

// test/e2e/pages/HomePage.js
import { EDSPage } from './EDSPage.js';

export class HomePage extends EDSPage {
  constructor(page) {
    super(page);
    this.hero = page.locator('.hero');
    this.cards = page.locator('.cards');
    this.nav = page.locator('nav');
  }

  async navigate() {
    await this.goto('/');
  }
}
```

**Correct -- E2E test specs:**

```javascript
// test/e2e/homepage.spec.js
import { test, expect } from '@playwright/test';
import { HomePage } from './pages/HomePage.js';

test.describe('Home Page', () => {
  let homePage;

  test.beforeEach(async ({ page }) => {
    homePage = new HomePage(page);
    await homePage.navigate();
  });

  test('hero block renders with heading and image', async () => {
    await expect(homePage.hero).toBeVisible();
    await expect(homePage.hero.locator('h1')).not.toBeEmpty();
    await expect(homePage.hero.locator('picture img')).toBeVisible();
  });

  test('navigation links are functional', async ({ page }) => {
    const navLinks = homePage.nav.locator('a');
    const count = await navLinks.count();
    expect(count).toBeGreaterThan(0);

    // Verify first nav link navigates successfully
    const href = await navLinks.first().getAttribute('href');
    await navLinks.first().click();
    await page.waitForLoadState('networkidle');
    expect(page.url()).toContain(href);
  });

  test('cards block renders all card items', async () => {
    await expect(homePage.cards).toBeVisible();
    const cardItems = homePage.cards.locator('li');
    const count = await cardItems.count();
    expect(count).toBeGreaterThan(0);

    // Each card has an image and body
    for (let i = 0; i < count; i++) {
      await expect(cardItems.nth(i).locator('img')).toBeVisible();
    }
  });

  test('page loads within performance budget', async ({ page }) => {
    const timing = await page.evaluate(() => {
      const nav = performance.getEntriesByType('navigation')[0];
      return {
        ttfb: nav.responseStart - nav.requestStart,
        domContentLoaded: nav.domContentLoadedEventEnd - nav.startTime,
        load: nav.loadEventEnd - nav.startTime,
      };
    });

    // EDS pages should load fast
    expect(timing.ttfb).toBeLessThan(800);
    expect(timing.domContentLoaded).toBeLessThan(2000);
    expect(timing.load).toBeLessThan(3000);
  });
});

// test/e2e/blocks/tabs.spec.js
import { test, expect } from '@playwright/test';
import { EDSPage } from '../pages/EDSPage.js';

test.describe('Tabs Block', () => {
  test('tab switching shows correct content', async ({ page }) => {
    const edsPage = new EDSPage(page);
    await edsPage.goto('/drafts/block-testing/tabs-test');

    const tabs = page.locator('.tabs .tabs-tab');
    const panels = page.locator('.tabs .tabs-panel');

    // Click second tab
    await tabs.nth(1).click();

    // First panel hidden, second visible
    await expect(panels.nth(0)).toBeHidden();
    await expect(panels.nth(1)).toBeVisible();
  });
});
```

---

### 12. Real User Monitoring (RUM) as Production Testing

EDS includes Operational Telemetry (formerly Real Use Monitoring) that collects Core Web Vitals from real visitors. This serves as continuous production testing, validating that lab-based Lighthouse scores hold up in the field.

**Metrics collected by Operational Telemetry:**

| Metric | Description | Target |
|--------|-------------|--------|
| LCP (Largest Contentful Paint) | Visual loading performance | < 2.5s |
| INP (Interaction to Next Paint) | Responsiveness | < 200ms |
| CLS (Cumulative Layout Shift) | Visual stability | < 0.1 |
| TTFB (Time to First Byte) | Server response time | < 800ms |

**How Operational Telemetry works:**

- Sampling ratio: 1:100 page views (privacy-preserving)
- No cookies, no sessions, no visitor tracking
- Data includes: hostname, URL, device type, OS, referrer
- The `/.rum` endpoint must be accessible (not blocked by CDN)
- Telemetry does not count as content requests (no billing impact)

**Using RUM data as a testing signal:**

```javascript
// Example: check if a recent deployment degraded Core Web Vitals
// Use Adobe's RUM Dashboard or build custom monitoring

// The aem.js library automatically injects the telemetry script.
// In delayed.js, you can add custom checkpoints:
export default function loadDelayed() {
  // Custom performance checkpoint (if using sampleRUM from aem.js)
  const { sampleRUM } = window.hlx || {};
  if (sampleRUM) {
    // Track custom interaction
    sampleRUM('custom-event', { source: 'cta-hero', target: '/products' });
  }
}
```

**Configuration options (via Cloud Manager environment variables):**

| Variable | Purpose |
|----------|---------|
| `AEM_OPTEL_DISABLED=true` | Disable telemetry entirely |
| `AEM_OPTEL_INCLUDED_PATHS` | Monitor specific pages only |
| `AEM_OPTEL_EXCLUDED_PATHS` | Exclude specific pages |

**Correct -- using RUM data to validate deployments:**

Monitor Core Web Vitals trends after each deployment. If LCP, INP, or CLS regresses beyond thresholds, investigate recent code changes. RUM provides field data that complements lab-based Lighthouse scores -- a page scoring 100 in Lighthouse may still have poor field performance due to real-world network conditions, device diversity, or third-party script behavior.

---

### 13. Automated Quality Checks on Pull Requests

The AEM Code Sync GitHub App runs two automated checks on every pull request:

1. **Linting** -- ESLint (airbnb-base) and Stylelint on all JS/CSS files
2. **PageSpeed Insights** -- Lighthouse scores for Performance, Accessibility, SEO, and Best Practices

**PR submission checklist:**

```markdown
Before requesting PR review:
- [ ] No ESLint errors (`npx eslint blocks/ scripts/`)
- [ ] No Stylelint errors (`npx stylelint "blocks/**/*.css" "styles/**/*.css"`)
- [ ] Lighthouse score is 100 on both mobile and desktop
- [ ] Include URLs to pages where reviewers can see the code in action
- [ ] PR title and description match the scope of changes
- [ ] Use Draft PR for work-in-progress
```

**Incorrect -- submitting a PR with linting errors or degraded Lighthouse:**

```
// WRONG: The AEM Code Sync bot will flag these issues.
// Do not submit PRs with:
// - ESLint violations
// - Stylelint violations
// - Lighthouse score below 100 on any category
// - Missing page URLs for reviewer testing
```

**Branch protection best practice:**

Configure `main` branch protection requiring at least one approval before merge. Use scaled trunk-based development: merge small PRs frequently to limit QA scope.
