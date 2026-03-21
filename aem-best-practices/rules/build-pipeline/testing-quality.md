---
title: Testing & Quality Assurance for AEM Cloud Service
impact: CRITICAL
impactDescription: Untested AEM code causes production failures, blocks Cloud Manager pipelines, and results in costly rollbacks across environments
tags: testing, quality, unit-tests, integration-tests, e2e, cypress, playwright, aem-mocks, cloud-manager, quality-gates, sonarqube, accessibility, visual-regression, ci-cd
---

## Testing & Quality Assurance for AEM Cloud Service

A comprehensive testing strategy for AEM as a Cloud Service follows the test pyramid: many fast unit tests, fewer integration tests, and targeted end-to-end tests. Cloud Manager enforces quality gates at every stage of the pipeline.

---

### 1. Unit Testing Sling Models with AEM Mocks

AEM Mocks (io.wcm) provide a mock AEM context for unit testing Sling Models, OSGi services, and Sling servlets without a running AEM instance. Tests execute at build time with Maven and JUnit 5.

**Maven dependencies (test scope):**

```xml
<dependencies>
  <!-- JUnit 5 -->
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
  </dependency>

  <!-- Mockito -->
  <dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
  </dependency>

  <!-- AEM Mocks (io.wcm) -->
  <dependency>
    <groupId>io.wcm</groupId>
    <artifactId>io.wcm.testing.aem-mock.junit5</artifactId>
    <version>5.6.2</version>
    <scope>test</scope>
  </dependency>

  <!-- Sling Mocks (included transitively, but useful for direct use) -->
  <dependency>
    <groupId>org.apache.sling</groupId>
    <artifactId>org.apache.sling.testing.sling-mock.junit5</artifactId>
    <version>3.5.0</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

**Correct -- AemContext setup with JUnit 5 extension:**

```java
import io.wcm.testing.mock.aem.junit5.AemContext;
import io.wcm.testing.mock.aem.junit5.AemContextExtension;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class HeroModelTest {

    // AemContext provides mock JCR, Sling ResourceResolver, and OSGi services
    private final AemContext ctx = new AemContext();

    @BeforeEach
    void setUp() {
        // Register the Sling Model class under test
        ctx.addModelsForClasses(HeroModel.class);

        // Load mock content from JSON (mirrors test class package under src/test/resources)
        ctx.load().json("/com/mysite/models/HeroModelTest.json", "/content/mysite");
    }

    @Test
    void testGetTitle() {
        // Set the current resource to the mock content node
        ctx.currentResource("/content/mysite/hero");

        // Adapt the request to the Sling Model
        HeroModel model = ctx.request().adaptTo(HeroModel.class);

        assertNotNull(model);
        assertEquals("Welcome to My Site", model.getTitle());
    }

    @Test
    void testEmptyTitle_returnsNull() {
        ctx.currentResource("/content/mysite/hero-empty");
        HeroModel model = ctx.request().adaptTo(HeroModel.class);

        assertNotNull(model);
        assertNull(model.getTitle());
    }
}
```

**Mock content JSON (`src/test/resources/com/mysite/models/HeroModelTest.json`):**

```json
{
  "hero": {
    "jcr:primaryType": "nt:unstructured",
    "sling:resourceType": "mysite/components/hero",
    "jcr:title": "Welcome to My Site",
    "description": "A hero banner component",
    "fileReference": "/content/dam/mysite/hero-image.jpg"
  },
  "hero-empty": {
    "jcr:primaryType": "nt:unstructured",
    "sling:resourceType": "mysite/components/hero"
  }
}
```

**Incorrect -- not registering the model class before adapting:**

```java
// WRONG: forgetting ctx.addModelsForClasses() causes adaptTo() to return null
@BeforeEach
void setUp() {
    ctx.load().json("/test-content.json", "/content");
}

@Test
void testModel() {
    ctx.currentResource("/content/hero");
    // This returns null because the model class was never registered
    HeroModel model = ctx.request().adaptTo(HeroModel.class);
    // NullPointerException!
    assertEquals("Title", model.getTitle());
}
```

---

### 2. Mocking OSGi Services and ModelFactory

When Sling Models depend on OSGi services or other models, register mock services in the AemContext with appropriate service ranking.

**Correct -- registering mock services with highest priority:**

```java
import org.apache.sling.models.factory.ModelFactory;
import org.mockito.Mock;
import static org.mockito.Mockito.*;
import static org.osgi.framework.Constants.SERVICE_RANKING;

@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class BylineModelTest {

    private final AemContext ctx = new AemContext();

    @Mock
    private ModelFactory modelFactory;

    @Mock
    private Image image;

    @BeforeEach
    void setUp() {
        ctx.addModelsForClasses(BylineModel.class);
        ctx.load().json("/com/mysite/models/BylineModelTest.json", "/content/mysite");

        // Register mock ModelFactory with highest priority to override AemContext default
        lenient().when(modelFactory.getModelFromWrappedRequest(
            eq(ctx.request()), any(Resource.class), eq(Image.class)))
            .thenReturn(image);

        ctx.registerService(ModelFactory.class, modelFactory,
            SERVICE_RANKING, Integer.MAX_VALUE);
    }

    @Test
    void testHasImage_whenImageExists() {
        lenient().when(image.getSrc()).thenReturn("/content/dam/mysite/author.jpg");
        ctx.currentResource("/content/mysite/byline");

        BylineModel model = ctx.request().adaptTo(BylineModel.class);
        assertTrue(model.hasImage());
    }
}
```

---

### 3. Frontend Unit Testing with Jest + JSDOM

ClientLib JavaScript can be unit tested with Jest using JSDOM to simulate the DOM environment. This is critical for testing component behavior, event handlers, and DOM manipulation.

**Jest configuration (`ui.frontend/jest.config.js`):**

```javascript
module.exports = {
  testEnvironment: 'jsdom',
  roots: ['<rootDir>/src'],
  testMatch: ['**/__tests__/**/*.test.js', '**/*.spec.js'],
  moduleNameMapper: {
    // Mock CSS imports (ClientLib CSS is handled by AEM, not webpack in tests)
    '\\.(css|less|scss)$': 'identity-obj-proxy',
  },
  setupFiles: ['<rootDir>/test/setup.js'],
  collectCoverage: true,
  coverageThreshold: {
    global: {
      branches: 50,
      functions: 50,
      lines: 50,
      statements: 50,
    },
  },
};
```

**Mocking AEM ClientLib dependencies (`ui.frontend/test/setup.js`):**

```javascript
// Mock global objects provided by AEM at runtime
global.Granite = {
  author: {
    DialogFrame: { currentFloatingDialog: null },
    edit: { EditableActions: {} },
  },
  I18n: { get: (key) => key },
  HTTP: {
    externalize: (path) => path,
    getContextPath: () => '',
  },
};

// Mock jQuery if components rely on it (common in legacy AEM code)
global.$ = global.jQuery = require('jquery');

// Mock Coral UI components
global.Coral = {
  commons: { ready: (el, cb) => cb() },
};
```

**Correct -- testing a component's DOM manipulation:**

```javascript
// src/components/accordion/__tests__/accordion.test.js
import { Accordion } from '../accordion';

describe('Accordion Component', () => {
  let container;

  beforeEach(() => {
    // Create DOM fixture matching AEM component markup
    document.body.innerHTML = `
      <div class="cmp-accordion" data-cmp-is="accordion">
        <div class="cmp-accordion__item" data-cmp-hook-accordion="item">
          <h3 class="cmp-accordion__header">
            <button class="cmp-accordion__button" data-cmp-hook-accordion="button">
              Section 1
            </button>
          </h3>
          <div class="cmp-accordion__panel" data-cmp-hook-accordion="panel"
               role="region" hidden>
            <p>Panel content</p>
          </div>
        </div>
      </div>
    `;
    container = document.querySelector('[data-cmp-is="accordion"]');
    new Accordion(container);
  });

  afterEach(() => {
    document.body.innerHTML = '';
  });

  test('clicking button expands the panel', () => {
    const button = container.querySelector('[data-cmp-hook-accordion="button"]');
    const panel = container.querySelector('[data-cmp-hook-accordion="panel"]');

    button.click();

    expect(panel.hidden).toBe(false);
    expect(button.getAttribute('aria-expanded')).toBe('true');
  });

  test('panel is hidden by default', () => {
    const panel = container.querySelector('[data-cmp-hook-accordion="panel"]');
    expect(panel.hidden).toBe(true);
  });
});
```

---

### 4. Integration Testing with AEM SDK (IT Module)

The `it.tests` module in the AEM project archetype contains Java-based integration tests that run against a live AEM instance using the `aem-cloud-testing-clients` library. These tests verify HTTP-level behavior, content creation, and component rendering.

**Maven dependency for integration tests:**

```xml
<dependency>
  <groupId>com.adobe.cq</groupId>
  <artifactId>aem-cloud-testing-clients</artifactId>
  <version>1.2.1</version>
</dependency>
```

**Correct -- custom functional test with CQ Testing Client:**

```java
import com.adobe.cq.testing.client.CQClient;
import com.adobe.cq.testing.junit.rules.CQAuthorClassRule;
import com.adobe.cq.testing.junit.rules.CQRule;
import org.apache.sling.testing.clients.ClientException;
import org.junit.ClassRule;
import org.junit.Rule;
import org.junit.Test;

import static org.junit.Assert.*;

/**
 * Integration test class. Name must end in "IT" (not "Test")
 * to be picked up by Cloud Manager.
 */
public class PageRenderIT {

    @ClassRule
    public static final CQAuthorClassRule cqBaseClassRule = new CQAuthorClassRule();

    @Rule
    public CQRule cqBaseRule = new CQRule(cqBaseClassRule.authorRule);

    @Test
    public void testHomePageReturns200() throws ClientException {
        CQClient author = cqBaseClassRule.authorRule.getAdminClient(CQClient.class);

        // Verify the home page renders successfully
        author.doGet("/content/mysite/en.html", 200);
    }

    @Test
    public void testComponentJsonExport() throws ClientException {
        CQClient author = cqBaseClassRule.authorRule.getAdminClient(CQClient.class);

        // Verify Sling Model JSON export
        String json = author.doGet(
            "/content/mysite/en/jcr:content/root/hero.model.json", 200
        ).getContent();

        assertTrue("JSON export should contain title",
            json.contains("\"title\""));
    }
}
```

**Critical: the test JAR must include the Cloud-Manager-TestType manifest header:**

```xml
<!-- it.tests/pom.xml -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-assembly-plugin</artifactId>
  <configuration>
    <archive>
      <manifestEntries>
        <Cloud-Manager-TestType>integration-test</Cloud-Manager-TestType>
      </manifestEntries>
    </archive>
    <descriptorRefs>
      <descriptorRef>jar-with-dependencies</descriptorRef>
    </descriptorRefs>
  </configuration>
</plugin>
```

**Important: name test classes with `IT` suffix (not `Test`):**
- `PageRenderIT.java` -- executed by Cloud Manager
- `PageRenderTest.java` -- ignored by Cloud Manager (treated as unit test)

---

### 5. End-to-End Testing with Cypress (Cloud Manager UI Tests)

Cloud Manager supports custom UI tests packaged as Docker images in the `ui.tests/` module. Adobe recommends Cypress for its automatic waiting and real-time reloading.

**Project structure for Cypress UI tests:**

```
ui.tests/
├── Dockerfile
├── assembly-ui-test-docker-context.xml
├── pom.xml
├── test-module/
│   ├── cypress.config.js
│   ├── package.json
│   ├── lib/
│   │   └── commands.js
│   └── cypress/
│       ├── e2e/
│       │   ├── author/
│       │   │   ├── page-authoring.cy.js
│       │   │   └── component-dialog.cy.js
│       │   └── publish/
│       │       ├── page-render.cy.js
│       │       └── navigation.cy.js
│       ├── fixtures/
│       │   └── test-content.json
│       └── support/
│           ├── commands.js
│           └── e2e.js
└── testing.properties          # REQUIRED: ui-tests.version=1
```

**`testing.properties` -- required opt-in file:**

```properties
ui-tests.version=1
```

**Cypress configuration using Cloud Manager environment variables:**

```javascript
// ui.tests/test-module/cypress.config.js
const { defineConfig } = require('cypress');

module.exports = defineConfig({
  e2e: {
    baseUrl: process.env.AEM_AUTHOR_URL || 'http://localhost:4502',
    env: {
      AEM_AUTHOR_URL: process.env.AEM_AUTHOR_URL || 'http://localhost:4502',
      AEM_AUTHOR_USERNAME: process.env.AEM_AUTHOR_USERNAME || 'admin',
      AEM_AUTHOR_PASSWORD: process.env.AEM_AUTHOR_PASSWORD || 'admin',
      AEM_PUBLISH_URL: process.env.AEM_PUBLISH_URL || 'http://localhost:4503',
    },
    reporter: 'junit',
    reporterOptions: {
      mochaFile: process.env.REPORTS_PATH
        ? `${process.env.REPORTS_PATH}/results-[hash].xml`
        : 'results/results-[hash].xml',
    },
    video: false,
    screenshotOnRunFailure: true,
    defaultCommandTimeout: 10000,
  },
});
```

**Environment variables available at runtime in Cloud Manager:**

| Variable | Description |
|----------|-------------|
| `AEM_AUTHOR_URL` | Author instance URL |
| `AEM_AUTHOR_USERNAME` | Admin username |
| `AEM_AUTHOR_PASSWORD` | Admin password |
| `AEM_PUBLISH_URL` | Publish instance URL |
| `REPORTS_PATH` | Directory for JUnit XML reports |
| `SELENIUM_BASE_URL` | Selenium Grid URL (Selenium only) |
| `PROXY_HOST` | HTTP proxy hostname |

---

### 6. End-to-End Testing with Playwright

Playwright is also supported in Cloud Manager's UI test pipeline and is ideal for cross-browser testing.

**Correct -- Playwright configuration for AEM:**

```javascript
// ui.tests/test-module/playwright.config.js
import { defineConfig, devices } from '@playwright/test';

const proxyServer = process.env.HTTP_PROXY || '';

export default defineConfig({
  testDir: './tests',
  outputDir: process.env.REPORTS_PATH || './results',
  timeout: 30000,
  retries: 1,
  reporter: [
    ['junit', { outputFile: `${process.env.REPORTS_PATH || './results'}/results.xml` }],
  ],
  use: {
    baseURL: process.env.AEM_AUTHOR_URL || 'http://localhost:4502',
    httpCredentials: {
      username: process.env.AEM_AUTHOR_USERNAME || 'admin',
      password: process.env.AEM_AUTHOR_PASSWORD || 'admin',
    },
    ...(proxyServer !== '' && { proxy: { server: proxyServer } }),
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
});
```

**Page Object Pattern for AEM e2e tests:**

```javascript
// ui.tests/test-module/tests/pages/SitesConsolePage.js
export class SitesConsolePage {
  constructor(page) {
    this.page = page;
    this.createButton = page.locator('coral-actionbar-primary [icon="add"]');
    this.pageList = page.locator('.cq-siteadmin-admin-childpages');
  }

  async navigate(path = '/sites.html/content/mysite') {
    await this.page.goto(path);
    await this.page.waitForSelector('.cq-siteadmin-admin-childpages');
  }

  async createPage(title, template) {
    await this.createButton.click();
    await this.page.locator('coral-list-item[value="page"]').click();
    // Select template
    await this.page.locator(`[data-path="${template}"]`).click();
    await this.page.locator('button:has-text("Next")').click();
    // Fill in title
    await this.page.locator('[name="./jcr:title"]').fill(title);
    await this.page.locator('button:has-text("Create")').click();
  }
}

// ui.tests/test-module/tests/pages/PageEditorPage.js
export class PageEditorPage {
  constructor(page) {
    this.page = page;
    this.editLayer = page.locator('#EditableToolbar');
  }

  async open(pagePath) {
    await this.page.goto(`/editor.html${pagePath}.html`);
    await this.page.waitForLoadState('networkidle');
  }

  async switchToEditMode() {
    await this.page.locator('[data-layer="Edit"]').click();
  }

  async openComponentDialog(componentPath) {
    const editable = this.page.frameLocator('#ContentFrame')
      .locator(`[data-path="${componentPath}"]`);
    await editable.click();
    await this.page.locator('[data-action="CONFIGURE"]').click();
    await this.page.waitForSelector('coral-dialog[open]');
  }
}
```

**Correct -- testing component dialogs (author-side):**

```javascript
// ui.tests/test-module/tests/author/hero-dialog.spec.js
import { test, expect } from '@playwright/test';
import { PageEditorPage } from '../pages/PageEditorPage';

test.describe('Hero Component Dialog', () => {
  let editor;

  test.beforeEach(async ({ page }) => {
    editor = new PageEditorPage(page);
    await editor.open('/content/mysite/en/test-page');
  });

  test('should save title from dialog', async ({ page }) => {
    await editor.openComponentDialog('/content/mysite/en/test-page/jcr:content/root/hero');

    const dialog = page.locator('coral-dialog[open]');
    await dialog.locator('[name="./jcr:title"]').fill('New Hero Title');
    await dialog.locator('button:has-text("Done")').click();

    // Verify the title renders in the content frame
    const contentFrame = page.frameLocator('#ContentFrame');
    await expect(contentFrame.locator('.cmp-hero__title')).toHaveText('New Hero Title');
  });
});
```

---

### 7. Cloud Manager Quality Gates

Cloud Manager enforces a multi-layered quality gate system during pipeline execution. Understanding these gates is essential to avoid deployment failures.

**Pipeline quality gate stages:**

| Gate | Production Pipeline | Non-Production Pipeline | Timeout |
|------|---------------------|------------------------|---------|
| Code Quality (SonarQube) | Blocking | Blocking | -- |
| Unit Tests | Blocking | Blocking | -- |
| Custom Functional Tests | Blocking | Opt-in | 60 min |
| Custom UI Tests | Blocking | Opt-in | 30 min |
| Experience Audit | Non-blocking | Non-blocking | -- |

**Code quality thresholds enforced by SonarQube:**

| Metric | Failure Threshold | Severity |
|--------|-------------------|----------|
| Security Rating | Below B | Critical (blocks pipeline) |
| Reliability Rating | Below C | Important (pauses pipeline) |
| Maintainability Rating | Below A | Important (pauses pipeline) |
| Test Coverage | Below 50% | Important (pauses pipeline) |
| Skipped Unit Tests | More than 1 | Info |
| Open Issues | More than 0 | Info |

**Correct -- suppressing SonarQube false positives with specific annotations:**

```java
// Suppress specific rule on the smallest possible scope
@SuppressWarnings("squid:S1166") // Exception handling rule
public void processContent(Resource resource) {
    try {
        // ... processing logic
    } catch (RepositoryException e) {
        LOG.error("Failed to process resource: {}", resource.getPath(), e);
    }
}
```

**Incorrect -- suppressing all warnings on the entire class:**

```java
// WRONG: too broad, hides real issues
@SuppressWarnings("all")
public class ContentService {
    // SonarQube can't help you here
}
```

---

### 8. AEM Test Framework (Granite Testing)

The `com.adobe.cq.testing` framework provides HTTP-based testing clients specifically designed for AEM functional testing.

**Key classes in the testing framework:**

```java
// CQClient -- base client for HTTP requests to AEM
CQClient client = cqBaseClassRule.authorRule.getAdminClient(CQClient.class);

// FormEntityBuilder -- create content via POST requests
FormEntityBuilder formEntity = FormEntityBuilder.create()
    .addParameter("jcr:primaryType", "cq:Page")
    .addParameter("jcr:content/jcr:title", "Test Page")
    .addParameter("jcr:content/sling:resourceType", "mysite/components/page");

client.doPost("/content/mysite/en", formEntity.build(), 201);

// CQSecurityClient -- manage users and permissions
CQSecurityClient securityClient = cqBaseClassRule.authorRule
    .getAdminClient(CQSecurityClient.class);
```

**Resource constraints for Cloud Manager test execution:**

| Resource | Custom Functional Tests | Custom UI Tests |
|----------|------------------------|-----------------|
| CPU | 0.5 cores | 2.0 cores |
| Memory | 0.5 Gi | 1.0 Gi |
| Timeout | 30 min | 30 min |
| Recommended | ~15 min | ~15 min |

---

### 9. Visual Regression Testing

Visual regression testing catches unintended visual changes in AEM components. Tools like Percy, Chromatic, and BackstopJS compare screenshots across builds.

**Correct -- BackstopJS configuration for AEM components:**

```json
{
  "id": "aem-visual-regression",
  "viewports": [
    { "label": "phone", "width": 375, "height": 812 },
    { "label": "tablet", "width": 768, "height": 1024 },
    { "label": "desktop", "width": 1440, "height": 900 }
  ],
  "scenarios": [
    {
      "label": "Hero Component",
      "url": "http://localhost:4502/content/mysite/en/test-visual.html",
      "selectors": [".cmp-hero"],
      "delay": 2000,
      "misMatchThreshold": 0.1,
      "requireSameDimensions": true
    },
    {
      "label": "Navigation Component",
      "url": "http://localhost:4502/content/mysite/en/test-visual.html",
      "selectors": [".cmp-navigation"],
      "delay": 1000,
      "hoverSelector": ".cmp-navigation__item--level-0:first-child",
      "postInteractionWait": 500
    }
  ],
  "paths": {
    "bitmaps_reference": "backstop_data/bitmaps_reference",
    "bitmaps_test": "backstop_data/bitmaps_test"
  },
  "engine": "playwright"
}
```

**Percy integration in CI pipeline (GitHub Actions example):**

```yaml
- name: Percy Visual Tests
  env:
    PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}
  run: |
    npx percy snapshot --config .percy.yml
```

---

### 10. Accessibility Automated Testing

Automated accessibility testing should be part of every AEM project to catch WCAG violations early.

**Correct -- axe-core integration with Cypress:**

```javascript
// ui.tests/test-module/cypress/e2e/publish/accessibility.cy.js
import 'cypress-axe';

describe('Accessibility Tests', () => {
  beforeEach(() => {
    cy.visit('/content/mysite/en.html');
    cy.injectAxe();
  });

  it('home page has no critical a11y violations', () => {
    cy.checkA11y(null, {
      runOnly: {
        type: 'tag',
        values: ['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'],
      },
    }, (violations) => {
      violations.forEach((violation) => {
        cy.log(`${violation.id}: ${violation.description}`);
        violation.nodes.forEach((node) => {
          cy.log(`  - ${node.target}`);
        });
      });
    });
  });

  it('hero component meets a11y standards', () => {
    cy.checkA11y('.cmp-hero', {
      runOnly: {
        type: 'tag',
        values: ['wcag2a', 'wcag2aa'],
      },
    });
  });
});
```

**Correct -- pa11y CI configuration:**

```json
{
  "defaults": {
    "timeout": 30000,
    "wait": 2000,
    "standard": "WCAG2AA",
    "chromeLaunchConfig": {
      "args": ["--no-sandbox"]
    }
  },
  "urls": [
    {
      "url": "http://localhost:4503/content/mysite/en.html",
      "actions": ["wait for element .cmp-page__content to be visible"]
    },
    "http://localhost:4503/content/mysite/en/about.html"
  ]
}
```

**Correct -- Lighthouse CI configuration for AEM:**

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: [
        'http://localhost:4503/content/mysite/en.html',
        'http://localhost:4503/content/mysite/en/about.html',
      ],
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'categories:best-practices': ['warn', { minScore: 0.9 }],
        'categories:seo': ['warn', { minScore: 0.9 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

---

### 11. Test Data Management with Content Packages

Deterministic tests require consistent test content. Package test fixtures as AEM content packages.

**Correct -- test content package filter (`ui.tests.content/src/main/content/META-INF/vault/filter.xml`):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<workspaceFilter version="1.0">
  <filter root="/content/mysite/test-fixtures" mode="replace"/>
  <filter root="/content/dam/mysite/test-assets" mode="replace"/>
</workspaceFilter>
```

**Content structure for test fixtures:**

```
ui.tests.content/
└── src/main/content/jcr_root/
    └── content/
        └── mysite/
            └── test-fixtures/
                ├── .content.xml
                ├── visual-regression/
                │   └── .content.xml       # Pages with known component configs
                └── integration/
                    └── .content.xml       # Pages for IT module tests
```

**Install test content before tests, remove after:**

```bash
# Deploy test fixtures before running tests
mvn clean install -pl ui.tests.content -PautoInstallPackage

# Run integration tests
mvn verify -pl it.tests -Plocal

# Clean up test content
curl -u admin:admin -X DELETE http://localhost:4502/content/mysite/test-fixtures
```

---

### 12. Mocking GraphQL Responses for Headless Tests

When testing headless frontends (React, Angular) that consume AEM Content Fragment GraphQL APIs, mock the responses to isolate frontend logic.

**Correct -- mocking AEM GraphQL with MSW (Mock Service Worker):**

```javascript
// src/mocks/handlers.js
import { graphql, HttpResponse } from 'msw';

export const handlers = [
  // Mock AEM persisted query endpoint
  graphql.query('ArticleList', () => {
    return HttpResponse.json({
      data: {
        articleList: {
          items: [
            {
              _path: '/content/dam/mysite/articles/article-1',
              title: 'Test Article',
              slug: 'test-article',
              author: { name: 'Jane Doe' },
              featuredImage: {
                _dynamicUrl: '/adobe/dynamicmedia/deliver/dm-aid--abc123/article-hero.jpg',
              },
            },
          ],
        },
      },
    });
  }),
];

// src/mocks/server.js (Node.js for Jest)
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
export const server = setupServer(...handlers);

// Jest setup
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

### 13. CI/CD Pipeline Testing Stages

Cloud Manager pipelines execute tests in a defined order. Understanding the stages helps structure test suites correctly.

**Pipeline execution order:**

```
1. Code Checkout
2. Maven Build (unit tests execute here)
3. Code Quality Scan (SonarQube + OakPAL + Dispatcher validation)
4. Deploy to Stage
5. Product Functional Tests (Adobe-maintained)
6. Custom Functional Tests (it.tests module, JAR)
7. Custom UI Tests (ui.tests module, Docker)
8. Experience Audit (Lighthouse, non-blocking)
9. Deploy to Production (after approval)
```

**Best practices for the test pyramid in AEM:**

- **Unit tests (70%):** Fast, isolated, cover Sling Models, servlets, OSGi services. Use AEM Mocks. Target 50%+ coverage to pass Cloud Manager gate.
- **Integration tests (20%):** HTTP-based tests against AEM SDK or Cloud environment. Focus on critical content paths, API endpoints, and cross-component interactions.
- **E2E tests (10%):** Cypress or Playwright against full AEM stack. Cover key authoring flows and critical publish-side user journeys. Keep total runtime under 15 minutes.

**Incorrect -- inverting the test pyramid:**

```
// WRONG: heavy on slow e2e tests, few unit tests
// Results in:
// - Long pipeline execution times (>30 min for UI tests alone)
// - Flaky tests due to infrastructure dependencies
// - Late feedback on regressions
// - Cloud Manager timeout failures
```

---

### 14. Testing ClientLib JavaScript

DOM fixtures and event simulation are essential for testing ClientLib JS outside a running AEM instance.

**Correct -- testing ClientLib JS with DOM fixtures and event simulation:**

```javascript
// test/clientlib/tabs.test.js
describe('Tabs ClientLib', () => {
  beforeEach(() => {
    // Create DOM fixture matching AEM component HTML output
    document.body.innerHTML = `
      <div class="cmp-tabs" data-cmp-is="tabs">
        <ol class="cmp-tabs__tablist" role="tablist">
          <li class="cmp-tabs__tab cmp-tabs__tab--active"
              role="tab" tabindex="0" aria-selected="true"
              data-cmp-hook-tabs="tab">Tab 1</li>
          <li class="cmp-tabs__tab"
              role="tab" tabindex="-1" aria-selected="false"
              data-cmp-hook-tabs="tab">Tab 2</li>
        </ol>
        <div class="cmp-tabs__tabpanel cmp-tabs__tabpanel--active"
             role="tabpanel" data-cmp-hook-tabs="tabpanel">Panel 1</div>
        <div class="cmp-tabs__tabpanel"
             role="tabpanel" data-cmp-hook-tabs="tabpanel" hidden>Panel 2</div>
      </div>
    `;
  });

  test('keyboard navigation with arrow keys', () => {
    const tabs = document.querySelectorAll('[data-cmp-hook-tabs="tab"]');
    const panels = document.querySelectorAll('[data-cmp-hook-tabs="tabpanel"]');

    // Simulate right arrow key on first tab
    const event = new KeyboardEvent('keydown', {
      key: 'ArrowRight',
      bubbles: true,
    });
    tabs[0].dispatchEvent(event);

    // After component handles event, second tab should be active
    // (assertion depends on component implementation)
  });
});
```
