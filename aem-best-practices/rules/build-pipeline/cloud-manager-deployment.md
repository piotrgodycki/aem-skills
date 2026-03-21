---
title: Cloud Manager CI/CD and Deployment
impact: HIGH
impactDescription: Cloud Manager is the sole deployment path to AEM as a Cloud Service — misconfigured pipelines cause failed deployments, skipped quality gates, and environment drift
tags: cloud-manager, ci-cd, pipelines, deployment, quality-gates, environments, frontend-pipeline, aio-cli
---

## Cloud Manager CI/CD and Deployment

Cloud Manager is the only way to deploy code to AEM as a Cloud Service. There is no direct package installation on author/publish. Understanding pipeline types, quality gates, and environment management is essential for reliable delivery.

---

### 1. Pipeline Types

| Pipeline Type | Deploys | Build | Use Case |
|--------------|---------|-------|----------|
| **Full-Stack** | All modules (ui.apps, core, ui.content, etc.) | Maven | Standard deployment |
| **Frontend-Only** | `ui.frontend` module only | npm | Rapid frontend iteration |
| **Config** | CDN/Dispatcher config only | YAML validation | Infrastructure changes |
| **Web-Tier** | Dispatcher config only | Apache/Dispatcher validation | Dispatcher-only changes |

#### Full-Stack Pipeline

Builds the entire project and deploys all packages:

```
Source (Git) → Build (Maven) → Unit Tests → Code Quality → Security Scan
  → Deploy to Dev → Integration Tests → Deploy to Stage → Performance Tests
  → Manual Approval → Deploy to Production
```

#### Frontend-Only Pipeline

Deploys only the `ui.frontend` module — no Java build, no content packages:

```
Source (Git) → Build (npm) → Deploy ui.frontend artifact to CDN
```

**Requirement:** Project must use the AEM frontend module structure:

```
ui.frontend/
├── package.json
├── webpack.common.js
├── webpack.prod.js
└── src/
    ├── main/
    │   └── webpack/
    │       ├── static/     → copied to clientlib
    │       └── site/       → compiled CSS/JS
    └── test/
```

```json
// ui.frontend/package.json — required scripts
{
  "scripts": {
    "build": "webpack --config webpack.prod.js",
    "test": "jest"
  }
}
```

---

### 2. Pipeline Stages and Quality Gates

#### Code Quality Gate

Cloud Manager runs SonarQube-based analysis. Critical/Blocker issues fail the pipeline.

**Default rules include:**
- Security vulnerabilities (SQL injection, XSS, etc.)
- Bug patterns (null dereference, resource leaks)
- Code smells (complexity, duplication)
- OakPAL package checks (content structure validation)
- Dispatcher configuration validation

**Custom quality rules** in `ui.apps/src/main/content/META-INF/vault/filter.xml`:

```xml
<!-- Ensure filter.xml doesn't overwrite system paths -->
<workspaceFilter version="1.0">
    <filter root="/apps/myproject" mode="merge"/>
    <filter root="/content/myproject" mode="merge"/>
    <!-- NEVER: <filter root="/libs" /> -->
</workspaceFilter>
```

#### Functional Testing

Cloud Manager runs custom functional tests from the `it.tests` module:

```java
// it.tests/src/main/java/com/myproject/it/tests/PublishPageIT.java
@ExtendWith(AemIntegrationTestExtension.class)
public class PublishPageIT {

    @Test
    @DisplayName("Homepage returns 200")
    void homePageReturns200() {
        given()
            .baseUri(System.getProperty("aem.publish.url"))
        .when()
            .get("/content/mysite/en.html")
        .then()
            .statusCode(200);
    }
}
```

#### UI Testing

Cypress/Playwright tests from the `ui.tests` module:

```javascript
// ui.tests/test-module/cypress/e2e/basic.cy.js
describe('Homepage', () => {
  it('loads and displays hero', () => {
    cy.visit('/content/mysite/en.html');
    cy.get('.cmp-teaser').should('be.visible');
    cy.get('.cmp-teaser__title').should('not.be.empty');
  });
});
```

---

### 3. Environment Management

#### Environment Types

| Environment | Purpose | Auto-Deploy | Lifecycle |
|------------|---------|-------------|-----------|
| **Dev** | Development/testing | Pipeline target | Persistent |
| **Stage** | Pre-production validation | Pipeline target | Persistent |
| **Production** | Live site | Pipeline target (with approval) | Persistent |
| **RDE** | Rapid iteration | Direct via AIO CLI | Resets periodically |

#### Environment Variables

Configure via Cloud Manager UI or API:

```
Type: Environment Variable
Name: PRODUCT_API_URL
Value: https://api.example.com/products
Environment: Production
Service: Author | Publish | All
```

Access in OSGi configuration:

```json
// com.myproject.core.impl.ProductServiceImpl.cfg.json
{
    "api.endpoint": "$[env:PRODUCT_API_URL;default=https://api.staging.example.com/products]"
}
```

Access in code:

```java
// Via OSGi config (preferred — see above)
// Or directly (less preferred):
String value = System.getenv("PRODUCT_API_URL");
```

**Secret variables** — values hidden in UI, masked in logs:

```
Type: Secret
Name: API_SECRET_KEY
Value: sk-xxxxxxxxxxxx
```

```json
// Reference in OSGi config
{
    "api.key": "$[secret:API_SECRET_KEY]"
}
```

---

### 4. Deployment Behavior

#### Blue-Green Deployment

AEM as a Cloud Service uses rolling deployments with no downtime:

```
1. New code deployed to a subset of pods
2. Health checks validate new pods
3. Traffic gradually shifted to new pods
4. Old pods drained and terminated
5. If health checks fail → automatic rollback
```

**Important implications:**
- Two versions run simultaneously during deployment
- Content changes during deployment are preserved
- OSGi configs take effect as pods restart
- Sling Jobs in-flight may run on old or new code

#### Content Freeze During Deployment

```
DO NOT:
- Run bulk content operations during deployments
- Trigger workflow bulk processing during deployments
- Run reindex operations during deployments

DO:
- Coordinate deployment windows with content teams
- Use maintenance windows for large content operations
```

---

### 5. AIO CLI Integration

```bash
# Install AIO CLI
npm install -g @adobe/aio-cli
aio plugins:install @adobe/aio-cli-plugin-cloudmanager

# Authenticate
aio auth:login

# List environments
aio cloudmanager:list-environments --programId=12345

# Start pipeline execution
aio cloudmanager:start-execution --programId=12345 --pipelineId=67890

# Check pipeline status
aio cloudmanager:get-current-execution --programId=12345 --pipelineId=67890

# Download logs
aio cloudmanager:download-logs --programId=12345 --environmentId=111 --service=author --name=aemerror --days=1

# Set environment variable
aio cloudmanager:set-environment-variables --programId=12345 --environmentId=111 \
  --variable PRODUCT_API_URL https://api.example.com

# Content copy (stage → dev)
aio cloudmanager:content-copy --programId=12345 \
  --sourceEnvironmentId=222 --destEnvironmentId=111 \
  --includePaths="/content/mysite/en"
```

---

### 6. Dispatcher Configuration Validation

Cloud Manager validates Dispatcher config in the build stage:

```
dispatcher/
├── src/
│   ├── conf.d/
│   │   ├── available_vhosts/
│   │   │   └── default.vhost
│   │   ├── enabled_vhosts/
│   │   │   └── default.vhost → ../available_vhosts/default.vhost
│   │   ├── rewrites/
│   │   │   └── rewrite.rules
│   │   └── variables/
│   │       └── custom.vars
│   └── conf.dispatcher.d/
│       ├── available_farms/
│       │   └── default.farm
│       ├── enabled_farms/
│       │   └── default.farm → ../available_farms/default.farm
│       ├── cache/
│       │   └── rules.any
│       ├── filters/
│       │   └── filters.any
│       └── clientheaders/
│           └── clientheaders.any
└── validator            # Adobe-provided validation tool
```

**Local validation before commit:**

```bash
# Download and run the Dispatcher validator
docker run --rm -v $(pwd)/dispatcher/src:/mnt/dev/src \
  adobe/aem-cs-dispatcher-tools:latest validate /mnt/dev/src
```

---

### 7. Rollback Procedures

Cloud Manager does not have a one-click rollback. Instead:

```bash
# Option 1: Revert the git commit and trigger a new pipeline
git revert HEAD
git push origin main
# Pipeline deploys the reverted code

# Option 2: Deploy a previous tag
git checkout v1.2.3
git push origin HEAD:main -f   # Force push to main (coordinate with team)
```

**Content rollback** requires content copy from another environment or restoring from backup (contact Adobe Support for production).

---

### 8. Anti-Patterns

#### Skipping Quality Gates

```
// WRONG — suppressing SonarQube rules in Cloud Manager
// Cloud Manager quality gates exist for a reason
// Fix the issues, don't suppress them

// CORRECT — fix the flagged issues or configure custom rules
// Use @SuppressWarnings only for verified false positives with a comment
@SuppressWarnings("squid:S1166") // Exception intentionally not rethrown — logged and handled
```

#### Missing Environment-Specific Configs

```
// WRONG — hardcoded URLs that differ per environment
private static final String API_URL = "https://api.prod.example.com";

// CORRECT — use environment variables via OSGi config
// "$[env:PRODUCT_API_URL;default=https://localhost:4502]"
```

#### Deploying Content as Code

```xml
<!-- WRONG — deploying mutable content in code package -->
<filter root="/content/mysite/en">
    <exclude pattern="/content/mysite/en/.*"/>
</filter>

<!-- CORRECT — only deploy structure/templates, not authored content -->
<filter root="/content/mysite" mode="merge">
    <include pattern="/content/mysite"/>
    <exclude pattern="/content/mysite/.*"/>
</filter>
```

#### Not Testing Dispatcher Changes

```bash
# WRONG — pushing Dispatcher changes without local validation
git add dispatcher/
git commit -m "Updated Dispatcher config"
git push  # May fail at Cloud Manager validation stage

# CORRECT — validate locally first
docker run --rm -v $(pwd)/dispatcher/src:/mnt/dev/src \
  adobe/aem-cs-dispatcher-tools:latest validate /mnt/dev/src
# Then commit and push
```
