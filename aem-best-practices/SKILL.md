---
name: aem-best-practices
description: Full-stack Adobe Experience Manager. Use when writing or reviewing AEM code — Sling Models, HTL, Core Components, Touch UI dialogs, OSGi, DAM, headless/GraphQL, Dispatcher/CDN, Cloud Manager — on AEM as a Cloud Service or AEM 6.5/Classic. Not for Edge Delivery Services blocks; use aem-eds-frontend-best-practices for those.
---

# AEM Full-Stack Best Practices

Production-grade patterns for Adobe Experience Manager, written for a senior full-stack engineer who knows OSGi, Sling, and the JCR. Each reference file carries the correct pattern, the incorrect one beside it, and the anti-patterns that bite in production.

Default target is **AEMaaCS** (Cloud Service). Where behaviour differs on **AEM 6.5 / Classic** (on-prem/AMS), the file says so in an `AEM 6.5 / Classic differences` section — read it before shipping to those versions.

## How to use this skill

This is reference, not a sequence. Open the one or two files that bear on the task **before** writing code, apply the pattern, then move on. Match the surrounding code's idiom over the examples here when they conflict. Reach only for what the task touches — a dialog change needs `touch-ui/`, not the whole skill.

Coral 3 only: never emit Coral 2 resource types. Core Component reuse before custom. Cloud Service compatibility (no ECMA workflow scripts, microservice asset processing) unless the task is explicitly 6.5.

## Reference map

**Build, deploy, and dev workflow** — reach when setting up the pipeline, ClientLibs, or CI/CD:
- `rules/build-pipeline/ui-frontend-structure.md` — Webpack pipeline, module layout, SCSS/TS, aem-clientlib-generator, frontend-only vs full-stack pipeline
- `rules/build-pipeline/clientlibs-configuration.md` — ClientLibrary folders, `allowProxy`, manifests, dependencies vs embedding
- `rules/build-pipeline/frontend-dev-workflow.md` — local SDK, proxy dev server, HMR, ClientLib debugging, Jest, Git flow
- `rules/build-pipeline/osgi-frontend-config.md` — Externalizer, Sling Mapping, CORS, Referrer Filter, Rewriter, env vars, Repo Init
- `rules/build-pipeline/testing-quality.md` — AEM Mocks unit tests, Jest/JSDOM, Cypress/Playwright e2e, Cloud Manager quality gates
- `rules/build-pipeline/cloud-manager-deployment.md` — pipeline types, quality gates, env vars/secrets, blue-green, rollback, AIO CLI
- `rules/build-pipeline/rapid-dev-environments.md` — RDE setup, AIO CLI, bundle/package/config deploy, log tail, RDE vs local SDK vs pipeline
- `rules/build-pipeline/content-transfer-tool.md` — CTT migration, extraction/ingestion, top-up, checksums
- `rules/build-pipeline/migration-patterns.md` — JSP→HTL, Classic→Touch UI, 6.5→Cloud, Coral 2→3, Foundation→Core Components, BPA findings

**Component development** — reach when building or editing a component:
- `rules/component-development/component-architecture.md` — proxy components, Core Component delegation, resource-type inheritance, model structure
- `rules/component-development/core-components-frontend.md` — Core Component BEM, decoration, data attributes, ClientLib categories
- `rules/component-development/htl-templating.md` — HTL expressions, context, `data-sly-*`, resolving models, template libraries
- `rules/component-development/component-dialogs.md` — dialog structure, tabs, field basics (deep Touch UI fields live in `touch-ui/`)
- `rules/component-development/style-system.md` — policy-driven style classes, `cq:styleGroups`, author-selectable variants
- `rules/component-development/universal-editor-aemaacs.md` — Universal Editor instrumentation, `data-aue-*`, remote/local rendering
- `rules/component-development/spa-editor.md` — SPA Editor, JSON model API, editable React/Angular containers
- `rules/component-development/dam-asset-patterns.md` — DAM microservices, processing profiles, Asset Compute, metadata, renditions
- `rules/component-development/accessibility-seo.md` — semantic markup, ARIA, structured data, heading order
- `rules/component-development/forms-adaptive.md` — Adaptive Forms, rules, submit actions, Forms as a Cloud Service

**Touch UI / Granite authoring** — reach when building dialogs, widgets, RTE, overlays, or ACLs:
- `rules/touch-ui/coral-ui-granite-frontend.md` — Coral 3 resource types, Granite UI foundations, dialog chrome
- `rules/touch-ui/touch-ui-page-editor.md` — page editor, editables, drop targets, listeners
- `rules/touch-ui/dialog-advanced-fields.md` — multifields, nested fields, pathfields, tag pickers, validation
- `rules/touch-ui/dialog-custom-widgets.md` — custom Granite widgets, clientlib-backed fields, render conditions
- `rules/touch-ui/dialog-datasources.md` — dynamic dropdowns, servlet DataSources, ACS Commons generic lists
- `rules/touch-ui/dialog-clientlibs-patterns.md` — `cq.authoring.dialog` clientlibs, dialog-ready JS, field dependencies
- `rules/touch-ui/rte-configuration.md` — RTE plugins, paste rules, inline styles, custom toolbar
- `rules/touch-ui/edit-config-behavior.md` — `cq:editConfig`, drop targets, in-place editing, listeners
- `rules/touch-ui/sling-resource-merger-overlays.md` — overlays via `/apps`, Sling Resource Merger, `sling:hideResource`
- `rules/touch-ui/templates-page-properties-policies.md` — editable templates, structure/initial, policies, page properties
- `rules/touch-ui/security-patterns.md` — XSS in HTL/JS, CSRF, secure dialog data, clickjacking
- `rules/touch-ui/user-management-acls.md` — users/groups, ACLs, Repo Init, service users, closed user groups

**Java backend** — reach when writing Sling Models, servlets, OSGi services, or integrations:
- `rules/java/sling-models-frontend.md` — Sling Model structure, injectors, `@PostConstruct`, exporter, null-safety
- `rules/java/lombok-best-practices.md` — Lombok with Sling Models, `@Getter`, `@Slf4j`, pitfalls
- `rules/java/sling-servlets.md` — servlet registration by path vs resource type, selectors, SlingSafeMethods
- `rules/java/osgi-services-schedulers.md` — OSGi R7 DS, config via OCD, schedulers, service users
- `rules/java/aem-workflows.md` — Granite Workflow, process steps, launchers, transient workflows, Cloud vs on-prem
- `rules/java/third-party-integrations.md` — HTTP clients, connection pools, circuit breakers, caching, resilience

**Headless & commerce** — reach when delivering content to external apps:
- `rules/headless/graphql-headless.md` — GraphQL API, persisted queries, caching, filtering
- `rules/headless/content-fragments.md` — CF models, variations, references, fragment composition
- `rules/headless/experience-fragments.md` — XF variations, templates, export to targets
- `rules/headless/headless-sdk-frameworks.md` — AEM Headless SDK with React/Next.js/Vue/Svelte
- `rules/headless/openapi-events.md` — AEM OpenAPIs, Adobe I/O Events, webhooks
- `rules/headless/commerce-cif.md` — Commerce Integration Framework, CIF components, GraphQL to commerce backend

**Analytics & personalization** — reach when wiring tracking or targeting:
- `rules/analytics-tracking/adobe-data-layer.md` — ACDL, event schema, computed state, component data push
- `rules/analytics-tracking/personalization-targeting.md` — Target, offers, audiences, context hub, segmentation

**Performance** — reach when the task is caching, CDN, queries, or page speed:
- `rules/performance/dispatcher-caching.md` — Dispatcher cache rules, invalidation, auto-flush, stat levels
- `rules/performance/cdn-configuration.md` — CDN at the edge, Fastly, BYOCDN, WAF, ESI, cache headers
- `rules/performance/cloud-performance.md` — Cloud Service perf model, request coalescing, resource budgets
- `rules/performance/query-optimization.md` — JCR/Oak queries, indexes (`oak:index`), traversal limits
- `rules/performance/dynamic-media-assets.md` — Dynamic Media, Smart Imaging, image/video delivery, hotlinking
- `rules/performance/frontend-optimization.md` — critical CSS, lazy load, ClientLib splitting, Core Web Vitals

**Frontend (AEM-native)** — reach when writing SCSS/JS for server-rendered AEM (not EDS):
- `rules/frontend/scss-domain-structure.md` — domain-driven SCSS structure, BEM, tokens
- `rules/frontend/tailwind-aem.md` — Tailwind in AEM, BEM coexistence, Style System integration, purge config
- `rules/frontend/alpine-js-aem.md` — Alpine.js in HTL, reactive directives, stores, hydration
- `rules/frontend/fe-aem-server.md` — fe-aem-server for local HTL rendering without a running AEM
- `rules/frontend/vanilla/web-components-aem.md` — custom elements hydrating Core Component markup
- `rules/frontend/vanilla/module-architecture.md` — ES module structure, ClientLib loading, tree-shaking boundaries
- `rules/frontend/vanilla/event-architecture.md` — event delegation, custom events, pub/sub, teardown

**Layout & multi-tenant:**
- `rules/layout/responsive-grid.md` — Layout Container, responsive grid, breakpoints, emulator
- `rules/multi-tenant/aemaacs-multi-tenant.md` — MSM, live copies, rollout config, blueprints, i18n

**Maintenance (existing repos)** — reach when auditing or evolving a codebase you didn't build:
- `rules/maintenance/spec-driven-development.md` — spec-first change workflow for brownfield AEM
- `rules/maintenance/codebase-audit-quality.md` — auditing an inherited repo: structure, debt, risk map
- `rules/maintenance/core-platform-maintenance.md` — platform/version upkeep, deprecations, SDK bumps
- `rules/maintenance/frontend-maintenance.md` — frontend debt, ClientLib sprawl, dependency hygiene
- `rules/maintenance/backend-maintenance.md` — Java/OSGi debt, deprecated APIs, bundle health
- `rules/maintenance/third-party-maintenance.md` — third-party dependency and integration upkeep
