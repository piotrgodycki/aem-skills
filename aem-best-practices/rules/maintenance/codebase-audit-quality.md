---
title: Codebase Audit & Quality Baseline
impact: HIGH
impactDescription: Changing an unfamiliar AEM repo without first mapping its conventions and quality baseline guarantees inconsistent code and silent regressions.
tags: audit, quality, onboarding, tech-debt, sonarqube, conventions, maintenance
---

## Codebase Audit & Quality Baseline

The first phase of every maintenance task (`spec-driven-development.md`) is understanding the repo you are about to change. This file is the checklist for mapping an existing AEM project and establishing a quality baseline before and after a change.

### 1. Map the Module Layout

A standard AEMaaCS Maven reactor; confirm which modules actually exist and what each owns.

| Module | Owns | Audit question |
|---|---|---|
| `all` | Container package (deploy artifact) | What embeds here? run modes? |
| `core` | Java — Sling Models, servlets, OSGi | Java version? bundle exports? |
| `ui.apps` | `/apps` — components, dialogs, clientlibs, configs | Component hierarchy? overlays? |
| `ui.content` | Sample/initial content, templates, policies | Editable templates? policies? |
| `ui.frontend` | Webpack build → clientlibs | Framework? build tooling? |
| `ui.config` | OSGi configs per run mode | Which run modes are configured? |
| `dispatcher` | Apache + Dispatcher config | Cloud (`src/`) or 6.5 (`dispatcher.any`)? |
| `it.tests` / `ui.tests` | Integration / UI tests | Do they run in the pipeline? |

**Correct — orient before editing:** read `pom.xml` reactor + each module `pom.xml`, `filevault` filters (`META-INF/vault/filter.xml`), the `.content.xml` of the component you'll touch, and the SDK/archetype version.

### 2. Detect the Deployment Target

Decide AEMaaCS vs 6.5 **first** — it gates which patterns are legal (see the `### AEM 6.5 / Classic differences` sections across this skill).

Signals:
- `dispatcher/src/conf.d` + `cdn.yaml` + `aem-sdk-api` dependency → **AEMaaCS**.
- `dispatcher.any` + `uber-jar` dependency + `cq-quickstart` → **6.5**.
- `<target>` version in Cloud Manager config, or `adobe/aem-sdk` vs `6.5.x` in dependencies.

### 3. Learn the Conventions (match them, don't impose yours)

- Naming: component groups, BEM prefix (`cmp-`, project prefix), clientlib categories.
- Package structure in `core` (by feature? by layer?).
- Sling Model style: `@Getter`/`@Slf4j` + injection (never `@Data` — see `java/lombok-best-practices.md`).
- HTL conventions, resource type inheritance (`sling:resourceSuperType`), proxy pattern usage.
- Lint/format config present: `.eslintrc`, `.prettierrc`, `checkstyle`, `pmd`, `spotless`.

New code must read like the surrounding code. A stylistically "correct" change that ignores repo conventions is a defect.

### 4. Establish the Quality Baseline

Capture the *before* state so you can prove the change did no harm.

**Correct — baseline then compare:**
```bash
# Static analysis baseline (AEMaaCS archetype ships these)
mvn -B clean verify                       # unit tests + checkstyle/pmd
mvn com.adobe.aem:aemanalyser-maven-plugin:analyse   # AEMaaCS readiness
# or SonarQube on 6.5
mvn sonar:sonar -Dsonar.projectKey=myco
```

- AEMaaCS: the **Cloud Manager quality gate** (code quality, security, OakPAL/BPA) is authoritative — a change must introduce **0 new** critical/blocker issues.
- 6.5: run SonarQube + the AEM Rules for SonarQube ruleset yourself; there is no enforced gate.
- Record test coverage before; a maintenance change should not drop it.

### 5. Build a Tech-Debt Map (don't fix it all now)

Note debt you encounter but keep it out of the current spec's scope unless the spec is explicitly a refactor:
- Foundation components / Coral 2 dialogs still in use → migration candidates.
- Deprecated APIs (`ResourceResolverFactory.getAdministrativeResourceResolver`, JCR admin sessions).
- Traversal queries, missing Oak indexes.
- Pinned/vulnerable dependencies (hand off to `third-party-maintenance.md`).

Surface the map; let the spec owner decide what becomes work.

### Anti-Patterns
- Editing a component before reading its `.content.xml`, dialog, and Sling Model.
- Imposing your personal style over the repo's existing conventions.
- Running no baseline, so you cannot prove the change introduced no new issues.
- Treating every piece of discovered tech debt as in-scope for the current change.
- Assuming AEMaaCS patterns on a 6.5 repo (or vice-versa) without checking the signals.
- Ignoring `filter.xml` — deploying a package that overwrites or deletes unrelated content.
