---
name: aem-best-practices
description: AEM as a Cloud Service (AEMaaCS) full-stack development guidelines — 69 rule files from a senior expert full-stack AEM developer perspective. Covers ui.frontend Webpack, Core Components BEM, ClientLibs, HTL, Sling Models, Lombok, Style System, Touch UI (Coral UI 3, page editor, RTE, editConfig, dialogs, overlays, policies, custom widgets, DataSources), Universal Editor, component architecture, accessibility/SEO, DAM/Assets (microservices, processing profiles, Asset Compute, metadata), Adobe Client Data Layer (ACDL), personalization/targeting, security, user management/ACLs, responsive grid, GraphQL, Content/Experience Fragments, headless SDK (React/Next.js/Vue/Svelte), OpenAPI/Events, Commerce/CIF, multi-tenant/MSM, Cloud Manager CI/CD, RDE, Content Transfer Tool, CDN (Fastly, BYOCDN, WAF, ESI), Dispatcher caching, Dynamic Media, query optimization, frontend performance, testing, Adaptive Forms, AEM Workflows, Sling Servlets, OSGi services/schedulers, migration patterns, React (feature-driven architecture, Context, hooks, storage, GraphQL, performance optimization, testing), Preact (Signals, islands, lightweight components), and Vanilla JS (Web Components, ES modules, DOM patterns, events, storage).
license: MIT
metadata:
  author: community
  version: "5.0.0"
  argument-hint: <file-or-pattern>
allowed-tools: Read Glob Grep Bash Edit Write
---

# AEMaaCS — Full-Stack Development Best Practices

> **You are a senior expert full-stack AEM developer.** Apply these 67 rules with deep understanding of AEM internals, OSGi, Sling, JCR, and the full Adobe stack. When writing code, follow enterprise-grade patterns with proper error handling, logging, and performance awareness. Prioritize maintainability, Core Component reuse, and Cloud Service compatibility.

69 rule files covering every aspect of full-stack development on Adobe Experience Manager as a Cloud Service.

> **Not for Edge Delivery Services** — if you are working with EDS blocks (`/blocks/`), vanilla JS/CSS, or the XWalk boilerplate, use the `aem-eds-frontend-best-practices` skill instead.

## Rules

### Build Pipeline & Dev Workflow (`rules/build-pipeline/`)
- `ui-frontend-structure.md` — Webpack pipeline (common/dev/prod configs), module organization, SCSS/TypeScript, aem-clientlib-generator, frontend pipeline vs full-stack, local proxy dev server
- `clientlibs-configuration.md` — ClientLibrary folders, `allowProxy`, manifests, HTL, dependencies vs embedding
- `frontend-dev-workflow.md` — Local SDK setup, proxy dev server, HMR, ClientLib debugging, Cloud Manager pipeline, Jest testing, Git workflow
- `osgi-frontend-config.md` — Externalizer, Sling Mapping, CORS, Referrer Filter, HTML Library Manager, Sling Rewriter, environment variables, Repo Init
- `testing-quality.md` — Sling Model unit tests (AEM Mocks), Jest/JSDOM, Cypress/Playwright e2e, Cloud Manager quality gates, visual regression, accessibility CI
- `migration-patterns.md` — JSP→HTL, Classic→Touch UI, 6.5→Cloud Service, Coral 2→3, Foundation→Core Components, BPA findings
- `cloud-manager-deployment.md` — Pipeline types (full-stack, frontend-only, config, web-tier), quality gates, environment variables/secrets, blue-green deployment, rollback, AIO CLI, Dispatcher validation
- `rapid-dev-environments.md` — RDE setup, AIO CLI commands, bundle/package/config deployment, log tailing, RDE vs local SDK vs pipeline, iteration workflow
- `content-transfer-tool.md` — CTT architecture (extraction/ingestion), BPA integration, migration sets, user mapping, incremental extraction, Cloud Acceleration Manager, post-migration validation

### Component Development (`rules/component-development/`)
- `core-components-frontend.md` — BEM naming (`cmp-` prefix), proxy pattern (complete structure), CSS customization strategies (BEM targeting, Style System, context scoping), Sling Model delegation, JS hooks, component versioning/upgrades
- `component-dialogs.md` — Complete Granite UI field type reference (text, number, select, radio, checkbox, path, date, color, tag pickers), dialog XML template, simple/composite multifield, field validation, conditional visibility (show/hide), layout types, naming conventions
- `style-system.md` — Layout vs display styles (detailed), CSS organization pattern (base/layout/display), policy XML configuration, multi-style combinations, common style patterns (container, title, teaser), Design Dialog migration
- `dam-asset-patterns.md` — Asset microservices architecture, processing profiles (image/video), custom Asset Compute workers, metadata schemas, Smart Tags, web-optimized image delivery (AssetDelivery API), Asset Selector UI, Connected Assets
- `universal-editor-aemaacs.md` — UE architecture, data-aue-* instrumentation (resource, type, prop, label, model, filter), content types, container setup, component model JSON definitions, Remote SPA editing, CORS configuration, UE vs Page Editor migration, extension points
- `htl-templating.md` — All block statements, expression language, XSS display contexts, Use-API patterns, global objects
- `component-architecture.md` — Hierarchy (foundation→core→proxy→custom), containers, versioning, decoration tags, WCM modes, error handling
- `spa-editor.md` — React/Angular SPA SDK, MapTo, ModelManager, Remote SPA, routing (deprecated — migration guidance)
- `accessibility-seo.md` — WCAG 2.1 AA, ARIA in HTL, heading hierarchy, SEO meta/sitemap/JSON-LD, hreflang
- `forms-adaptive.md` — Adaptive Forms Core Components v3, theming, rule editor, submission actions, prefill, reCAPTCHA, FDM

### Java & Backend (`rules/java/`)
- `sling-models-frontend.md` — Annotations, injection, JSON export, delegation, multifield, @PostConstruct, testing with AEM Mocks
- `lombok-best-practices.md` — @Getter/@Slf4j/@Builder with Sling Models, why @Data breaks @Model, Lombok + OSGi services, Jackson integration, lombok.config, safety matrix
- `aem-workflows.md` — WorkflowProcess interface, custom process steps, launchers (event types, exclude lists), transient workflows, payload handling (JCR/DAM), ParticipantStepChooser, workflow metadata, programmatic management, Cloud Service differences
- `sling-servlets.md` — @SlingServletResourceTypes vs @SlingServletPaths, GET/POST/PUT handling, URL decomposition (selectors, extensions, suffix), JSON responses, CSRF protection, servlet filters, resource resolution order, service user mapping
- `osgi-services-schedulers.md` — DS annotations (@Component, @Reference, @Designate), OSGi configuration (run-mode specific, factory configs), Sling Schedulers, Sling Jobs (cluster-safe), ResourceChangeListener, EventHandler, service ranking, service users, health checks
- `third-party-integrations.md` — Pooled HTTP clients (Apache HttpClient 5), integration service pattern, circuit breaker, response caching with TTL, stale-while-revalidate, environment-specific OSGi configs, Cloud Manager secrets, Sling Model integration, async/parallel API calls, webhook receivers, error handling strategy, MockWebServer testing

### Analytics & Tracking (`rules/analytics-tracking/`)
- `adobe-data-layer.md` — ACDL architecture, full JSON state schema, **pushing data from Java/Sling Models** (ComponentData, DataLayerBuilder), **pushing from HTL** (data-cmp-data-layer), **pushing from JavaScript** (adobeDataLayer.push()), 9 custom event implementations (clicks, forms, video, carousel, tabs, accordion, search, errors, scroll depth), Adobe Launch/Tags integration, GTM bridge, debug mode
- `personalization-targeting.md` — ContextHub, Adobe Target (at.js / Web SDK), experience targeting, flicker prevention, caching conflicts, consent/privacy

### Touch UI Authoring (`rules/touch-ui/`)
- `coral-ui-granite-frontend.md` — Coral UI 3 Web Components API, `Coral.commons.ready()`, Granite utilities, foundation events
- `touch-ui-page-editor.md` — Editor architecture, `Granite.author` namespace, events, layers, toolbar extension
- `edit-config-behavior.md` — `cq:editConfig`, toolbar actions, `cq:dropTargets`, `cq:inplaceEditing`, `cq:listeners`, `cq:htmlTag`
- `rte-configuration.md` — 16 RTE plugins, toolbar config, dialog vs inplace, paraformat, styles, paste rules
- `dialog-clientlibs-patterns.md` — Authoring clientlib categories, `dialog-ready` events, show/hide, validation API
- `dialog-advanced-fields.md` — Pathfield, file upload, tag picker, XF/CF pickers, color picker, nested multifield, `granite:rendercondition`
- `dialog-custom-widgets.md` — Custom Granite UI fields (JSP/HTL), field validation, event lifecycle, pre-save transforms
- `dialog-datasources.md` — DataSource servlets, dynamic select options, parameter passing, caching
- `sling-resource-merger-overlays.md` — `/apps` overlays, Merger properties, overlay vs override
- `templates-page-properties-policies.md` — Editable templates, content policies, design dialog, page properties extension
- `security-patterns.md` — XSS prevention (HTL contexts), CSRF tokens, CSP headers, CUG, Dispatcher filters, CORS
- `user-management-acls.md` — IMS integration (Adobe Admin Console), product profiles, Repo Init for permissions (ACLs, service users, groups), principal-based access control (Oak restrictions), CUG for gated content, token-based authentication, permission debugging

### Layout (`rules/layout/`)
- `responsive-grid.md` — Breakpoints (max 3, configurable), grid CSS classes (span, offset, hide, newline), Layout Container configuration, container nesting rules, template-level layout policy, custom breakpoint CSS generation, responsive images

### Headless & Content Delivery (`rules/headless/`)
- `graphql-headless.md` — Persisted queries (creation, execution, caching), complete filter operators (string/number/date/null), pagination (offset and cursor-based), parameterized queries, nested fragment references, web-optimized image delivery (_dynamicUrl), AEM Headless SDK (JS/React/Next.js ISR), CORS config, performance optimization
- `content-fragments.md` — CF Models (all field types), variations, versioning, HTL rendering, delivery APIs, AEM Eventing
- `experience-fragments.md` — XF vs CF, building blocks, variations, Target export, caching (SDI)
- `headless-sdk-frameworks.md` — JS/React/Next.js/Vue/Svelte SDKs, auth, CORS, image delivery, multi-environment
- `openapi-events.md` — CF OpenAPI (CRUD + delivery), AEM Eventing (webhooks, I/O Runtime), event-driven ISR
- `commerce-cif.md` — CIF architecture (Core Components + GraphQL connector), Adobe Commerce/Magento integration, product/category pickers, CIF URL provider, product data enrichment, cart/checkout, catalog caching, multi-store/multi-currency

### Multi-Tenant (`rules/multi-tenant/`)
- `aemaacs-multi-tenant.md` — MSM/Live Copy, content architecture, frontend theming, templates, Sling CAConfig, i18n, URL mapping

### Performance (`rules/performance/`)
- `cloud-performance.md` — Multi-tier CDN architecture (browser → Fastly → Origin Shield → Dispatcher → AEM), Cache-Control/Surrogate-Control strategies per content type, stale-while-revalidate patterns, immutable ClientLib URLs, cache invalidation (automatic + programmatic + CDN purge), auto-scaling, CDN hit ratio monitoring, Vary header optimization
- `cdn-configuration.md` — Adobe CDN (Fastly) config as code (YAML), BYOCDN setup, traffic filter/WAF rules, CDN redirects, ESI, custom domains, log forwarding
- `frontend-optimization.md` — Critical CSS, CWV, capo.js head order, RUM vs Lighthouse, TTFB (<800ms), HTTP/3, cookie consent via GTM, Speculation Rules, bfcache
- `dynamic-media-assets.md` — Smart crops, image presets, URL modifiers, Smart Imaging, responsive delivery, WOID, video, viewers
- `query-optimization.md` — QueryBuilder, JCR-SQL2, Oak indexes, slow query detection, N+1 prevention
- `dispatcher-caching.md` — Cache rules, statfileslevel, filters, TTL, flush agents, SDI, personalization, debugging

### Frontend — SCSS (`rules/frontend/`)
- `scss-domain-structure.md` — Domain-driven SCSS folder structure, design tokens, domain-scoped variables, barrel imports, Core Component overrides, responsive mixins, multi-tenant theming, Webpack integration

### Frontend — React (`rules/frontend/react/`)
- `feature-driven-architecture.md` — Domain/feature folder structure, barrel exports, co-located tests/styles, lazy-loaded feature modules, shared vs feature-specific code separation
- `context-state-management.md` — React Context for AEM page data, useReducer patterns, compound providers, persisted state, global vs local state boundaries, avoiding prop drilling
- `custom-hooks-aem.md` — Reusable hooks for AEM: useContentFragment, useGraphQL, useAuthorMode, useBreakpoint, useIntersection, useAemPage, hook composition patterns
- `storage-persistence.md` — localStorage/sessionStorage abstractions, IndexedDB for offline, cookie helpers, URL state sync, cross-tab communication, storage events, hydration-safe patterns
- `data-fetching-graphql.md` — AEM Headless SDK with React, persisted queries, SWR/TanStack Query integration, prefetching, pagination, error boundaries, ISR with Next.js, multi-environment config
- `performance-optimization.md` — React.memo, useMemo, useCallback, useTransition, useDeferredValue, code splitting with lazy/Suspense, virtualized lists, image optimization, bundle analysis, Core Web Vitals alignment
- `testing-react-aem.md` — Testing Library patterns for AEM components, mocking ModelManager, testing editable components, hook testing, integration tests with MSW, Storybook for AEM

### Frontend — Preact (`rules/frontend/preact/`)
- `preact-aem-setup.md` — Preact in ui.frontend (Webpack aliases), compat layer, bundle size comparison, when to choose Preact over React, HTL integration, islands architecture
- `signals-state.md` — Preact Signals for reactive state, computed values, effects, signal stores, Signals vs Context performance, shared state patterns, DevTools
- `lightweight-components.md` — Small-footprint Preact components for AEM, progressive enhancement, islands architecture, partial hydration, Preact + HTL hybrid rendering

### Frontend — Vanilla JS (`rules/frontend/vanilla/`)
- `web-components-aem.md` — Custom Elements for AEM, Shadow DOM with AEM styles, slots for HTL content projection, Coral UI interop, form-associated custom elements, AEM dialog integration
- `module-architecture.md` — ES module organization, dynamic import for AEM ClientLibs, feature modules, pub/sub event bus, dependency injection lite, barrel exports for vanilla JS
- `dom-patterns.md` — Efficient DOM manipulation, DocumentFragment batching, MutationObserver, template element cloning, event delegation, requestAnimationFrame scheduling, memory leak prevention
- `storage-utilities.md` — Storage abstraction layer, typed wrappers, TTL cache, namespace isolation per AEM site, quota management, fallback chain (memory → session → local → IndexedDB)
- `event-architecture.md` — Custom events with typed detail, event delegation patterns, AbortController cleanup, cross-component communication, integration with ACDL, decoupled pub/sub bus

## Key Principles

1. **You are a senior AEM expert** — write production-grade code, not tutorials
2. **RUM over Lighthouse** — field data (CrUX/RUM) is what Google ranks
3. **TTFB < 800ms** — maximize CDN hits, `stale-while-revalidate`, target <200ms AEM response
4. **Never modify Core Components** — proxy + BEM targeting
5. **Coral 3 only** — never Coral 2 resource types
6. **@Getter + @Slf4j on Sling Models** — never @Data (breaks injection)
7. **HTL display contexts** — never `@context='unsafe'`
8. **Push to ACDL from Sling Models** — implement ComponentData, use DataLayerBuilder
9. **Persisted GraphQL queries** in production — POST bypasses CDN
10. **Cookie consent via GTM** — load async, not in `<head>`
11. **Feature-driven architecture** — group by domain feature, not by file type
12. **Framework-appropriate patterns** — React for headless, Preact for islands, vanilla for traditional AEM
13. **Performance is measurable** — use React DevTools Profiler, webpack-bundle-analyzer, and RUM data
