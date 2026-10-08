---
title: Core & Platform Maintenance (Versions, Core Components, OSGi)
impact: HIGH
impactDescription: Uncontrolled AEM SDK, Core Component, or OSGi dependency upgrades break builds, dialogs, and rendered pages across the whole site.
tags: core-components, upgrade, aem-sdk, osgi, uber-jar, maintenance, versioning
---

## Core & Platform Maintenance

"Core" maintenance is keeping the platform foundation current: the AEM SDK/API version, Core Components, archetype-level config, and the OSGi dependency graph. These changes have the widest blast radius in the repo, so they are always spec-driven (`spec-driven-development.md`).

### 1. AEM SDK / API Version Bumps

AEMaaCS has no fixed version, but the `aem-sdk-api` / dispatcher tools you build against do. On 6.5 it is the `uber-jar` + service pack.

**Correct — controlled SDK bump (AEMaaCS):**
```xml
<!-- Bump deliberately; rebuild + rerun aemanalyser -->
<dependency>
  <groupId>com.adobe.aem</groupId>
  <artifactId>aem-sdk-api</artifactId>
  <version>2024.11.18572.20241121T120000Z-241000</version>
  <scope>provided</scope>
</dependency>
```
Then: `mvn clean verify` + `aemanalyser-maven-plugin:analyse` to catch removed/changed APIs **before** deploy.

**6.5 equivalent:** bump `uber-jar` to match the exact Service Pack installed on the instances. Mismatched uber-jar vs runtime SP = `NoSuchMethodError` at runtime.

**Incorrect — chasing "latest" blindly:** upgrading the SDK the day it drops, in the same PR as a feature, with no analyser run. Keep version bumps in their own spec/PR.

### 2. Core Components Upgrades

Core Components ship versioned (`v1`, `v2`, `v3`...). Your proxies inherit via `sling:resourceSuperType`.

- Upgrade the **dependency** (`core.wcm.components.*`) and the proxy `sling:resourceSuperType` deliberately, one version step at a time.
- A new Core Component major version can change HTL markup, data-layer output, and dialog structure → re-verify BEM targeting (`component-development/core-components-frontend.md`) and policies.
- Never fork/modify Core Components to "upgrade" them — re-point the proxy and layer customisations via BEM + Style System.

**Correct — proxy pins the version it inherits:**
```xml
<!-- /apps/myco/components/teaser/.content.xml -->
<jcr:root sling:resourceSuperType="core/wcm/components/teaser/v2/teaser"
          jcr:primaryType="cq:Component"/>
```
Upgrade = change `v2` → `v3`, then run the acceptance criteria (markup, data layer, dialog, visual diff).

### 3. OSGi Dependency Hygiene

- Keep third-party libs **out of the global classpath**; embed in your bundle and import-package only what you use, or use `<Embed-Dependency>` with care.
- Watch for split packages and version-range conflicts after a bump (`mvn dependency:tree`, bnd baseline).
- Export only stable API packages from `core`; mark internal packages `*.impl` and do not export them.

### 4. Repo-Wide Config & Run Modes

- Config changes under `ui.config` are platform-level — a wrong run-mode folder (`config.author` vs `config.publish`) silently applies everywhere or nowhere.
- Repo Init (`repoinit`) scripts for ACLs/service users are core infrastructure; change them with the same rigor as code.

### 5. Verify Platform Changes Broadly

Core changes need wide verification, not spot checks:
- Full `mvn clean verify` + analyser/quality gate.
- Deploy to a lower env; smoke-test a representative page set (templates × components).
- Visual regression on key templates.
- Check the error log for new OSGi `Unsatisfied`/`ClassNotFound`/`NoSuchMethod` on startup.

### AEM 6.5 / Classic differences
- Version axis is **Service Pack + Cumulative Fix Pack**, not a rolling SDK. Align `uber-jar` exactly to the installed SP.
- No automated analyser/quality gate — run SonarQube + AEM Rules yourself.
- Core Components are available on 6.5 but the max supported version depends on the SP; check the compatibility matrix before upgrading.
- Felix console is available to inspect the live bundle graph (`/system/console/bundles`).

### Anti-Patterns
- Bumping the SDK/uber-jar in the same PR as a feature change.
- Forking or editing Core Components instead of re-pointing the proxy and layering CSS.
- Skipping `aemanalyser` / `dependency:tree` after a version bump.
- Exporting `*.impl` packages from the `core` bundle.
- Mismatched `uber-jar` and runtime Service Pack on 6.5.
- Spot-checking one page after a platform-wide change.
