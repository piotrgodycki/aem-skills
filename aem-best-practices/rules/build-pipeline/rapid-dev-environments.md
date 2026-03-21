---
title: Rapid Development Environments (RDE)
impact: MEDIUM
impactDescription: RDE enables rapid iteration without full pipeline deployments — misuse leads to untested code in production and reliance on non-persistent environments
tags: rde, rapid-development, aio-cli, local-development, iteration, cloud-service
---

## Rapid Development Environments (RDE)

Rapid Development Environments (RDE) allow developers to deploy and test changes on a real Cloud Service environment in seconds, without running a full Cloud Manager pipeline. They bridge the gap between the local SDK and Cloud Manager pipelines.

---

### 1. When to Use RDE

| Scenario | Local SDK | RDE | Cloud Manager Pipeline |
|----------|-----------|-----|----------------------|
| Initial development | **Best** | Good | Slow |
| Cloud-specific behavior testing | Can't test | **Best** | Slow |
| Integration testing with cloud services | Can't test | **Best** | Good |
| Quick bug reproduction on cloud | Can't test | **Best** | Slow |
| Final validation before release | No | No | **Required** |
| Production deployment | No | No | **Required** |

**Use RDE when:**
- Testing code that behaves differently on Cloud vs local SDK
- Debugging Cloud-specific issues (asset microservices, CDN, IMS)
- Rapid iteration on OSGi configurations
- Testing Dispatcher configurations in a real environment

**Do NOT use RDE as:**
- A permanent development environment (resets periodically)
- A replacement for the full pipeline (no quality gates)
- A shared team environment (single-developer workflow)

---

### 2. Setup

#### Prerequisites

```bash
# Install AIO CLI and plugins
npm install -g @adobe/aio-cli
aio plugins:install @adobe/aio-cli-plugin-aem-rde

# Login to Adobe IMS
aio login

# Select your organization, program, and RDE environment
aio aem:rde:setup
```

#### Verify Connection

```bash
# Check RDE status
aio aem:rde:status

# Output:
# Environment: dev-rde-1234
# Status: Ready
# Author: https://author-p12345-e67890.adobeaemcloud.com
# Publish: https://publish-p12345-e67890.adobeaemcloud.com
```

---

### 3. Deploying to RDE

#### Deploy OSGi Bundles

```bash
# Deploy a single bundle JAR
aio aem:rde:install core/target/com.myproject.core-1.0.0-SNAPSHOT.jar

# Deploy with specific target (author or publish)
aio aem:rde:install core/target/com.myproject.core-1.0.0-SNAPSHOT.jar --target=author
aio aem:rde:install core/target/com.myproject.core-1.0.0-SNAPSHOT.jar --target=publish
```

#### Deploy Content Packages

```bash
# Deploy a content package
aio aem:rde:install ui.apps/target/com.myproject.ui.apps-1.0.0-SNAPSHOT.zip

# Deploy with filters (only specific paths)
aio aem:rde:install ui.apps/target/com.myproject.ui.apps-1.0.0-SNAPSHOT.zip \
  --path=/apps/myproject/components
```

#### Deploy OSGi Configurations

```bash
# Deploy a single config file
aio aem:rde:install ui.config/src/main/content/jcr_root/apps/myproject/osgiconfig/config/\
com.myproject.core.impl.ProductServiceImpl.cfg.json

# Deploy all configs in a folder
aio aem:rde:install ui.config/target/com.myproject.ui.config-1.0.0-SNAPSHOT.zip
```

#### Deploy Dispatcher Configuration

```bash
# Deploy Dispatcher config
aio aem:rde:install dispatcher/src --type=dispatcher-config
```

#### Deploy Frontend Module

```bash
# Build and deploy ui.frontend
cd ui.frontend && npm run build && cd ..
aio aem:rde:install ui.frontend/dist
```

---

### 4. Monitoring and Debugging

#### Tail Logs

```bash
# Tail author logs
aio aem:rde:logs --target=author

# Tail publish logs
aio aem:rde:logs --target=publish

# Filter by logger
aio aem:rde:logs --target=author --include=com.myproject
```

#### Check Deployed Artifacts

```bash
# List all deployed bundles
aio aem:rde:inspect --target=author --type=osgi-bundle

# List deployed configurations
aio aem:rde:inspect --target=author --type=osgi-config

# Check specific bundle status
aio aem:rde:inspect --target=author --type=osgi-bundle --name=com.myproject.core
```

---

### 5. RDE Reset and Cleanup

```bash
# Reset the RDE to a clean state (removes all custom deployments)
aio aem:rde:reset

# Delete specific deployed artifact
aio aem:rde:delete --target=author --type=osgi-bundle --name=com.myproject.core
```

---

### 6. Development Workflow

#### Recommended Iteration Cycle

```
1. Develop locally (AEM SDK + IDE)
   ↓
2. Unit test locally (mvn test)
   ↓
3. Deploy to RDE for cloud validation
   aio aem:rde:install core/target/myproject.core-1.0.0-SNAPSHOT.jar
   aio aem:rde:install ui.apps/target/myproject.ui.apps-1.0.0-SNAPSHOT.zip
   ↓
4. Validate on RDE (test cloud-specific behavior)
   ↓
5. Commit and push to Git
   ↓
6. Cloud Manager pipeline (full quality gates)
   ↓
7. Deploy to Dev → Stage → Production
```

#### Quick Iteration Script

```bash
#!/bin/bash
# deploy-rde.sh — build and deploy to RDE
set -e

echo "Building core bundle..."
mvn clean install -pl core -am -DskipTests

echo "Building ui.apps..."
mvn clean install -pl ui.apps -DskipTests

echo "Deploying to RDE..."
aio aem:rde:install core/target/com.myproject.core-1.0.0-SNAPSHOT.jar
aio aem:rde:install ui.apps/target/com.myproject.ui.apps-1.0.0-SNAPSHOT.zip

echo "Tailing logs..."
aio aem:rde:logs --target=author --include=com.myproject
```

---

### 7. Limitations

| Limitation | Impact |
|-----------|--------|
| No custom run modes | Cannot test run-mode-specific configs (e.g., `config.myrunmode`) |
| Periodic reset | Environment resets to baseline — don't store persistent data |
| No production deployment | RDE is for testing only — must use Cloud Manager for prod |
| Single developer workflow | Not designed for shared team development |
| No quality gates | Code deployed to RDE bypasses SonarQube, OakPAL, etc. |
| Limited content | Starts with minimal sample content |
| No pipeline triggers | Cannot trigger Cloud Manager pipelines from RDE |

---

### 8. RDE vs Local SDK vs Cloud Manager

| Feature | Local SDK | RDE | Cloud Manager Dev |
|---------|-----------|-----|-------------------|
| Speed | Instant | Seconds | 30-60 min pipeline |
| Cloud fidelity | Low | **High** | **High** |
| Asset microservices | No | **Yes** | **Yes** |
| CDN behavior | No | **Yes** | **Yes** |
| IMS authentication | Simulated | **Real** | **Real** |
| Dispatcher testing | Local only | **Real cloud** | **Real cloud** |
| Quality gates | No | No | **Yes** |
| Team collaboration | No | No | **Yes** |
| Persistence | Full | Temporary | Full |

---

### 9. Anti-Patterns

#### Using RDE as Permanent Dev

```bash
# WRONG — deploying to RDE and considering it "deployed"
# RDE resets periodically, your changes will be lost

# CORRECT — always commit to Git and run Cloud Manager pipeline for persistent deployment
```

#### Skipping Pipeline After RDE Validation

```bash
# WRONG workflow
# "It works on RDE, ship it to prod" → No quality gates, no stage testing

# CORRECT workflow
# RDE validate → commit → Cloud Manager pipeline → Dev → Stage → Prod
```

#### Deploying SNAPSHOT to RDE Without Local Testing

```bash
# WRONG — deploy untested code directly to RDE
mvn clean install -DskipTests
aio aem:rde:install core/target/myproject.core-1.0.0-SNAPSHOT.jar

# CORRECT — run unit tests first, then deploy to RDE for integration testing
mvn clean test
mvn clean install -DskipTests  # only skip tests for the install phase
aio aem:rde:install core/target/myproject.core-1.0.0-SNAPSHOT.jar
```
