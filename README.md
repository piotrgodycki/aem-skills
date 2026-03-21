# AEM Best Practices — Claude Code Skills

![AEM as a Cloud Service](https://img.shields.io/badge/AEM_as_a_Cloud_Service-FF0000?style=for-the-badge&logo=adobe&logoColor=white)
![Edge Delivery Services](https://img.shields.io/badge/Edge_Delivery_Services-FF0000?style=for-the-badge&logo=adobe&logoColor=white)
![skills.sh](https://img.shields.io/badge/skills.sh-Compatible-000000?style=for-the-badge&logo=vercel&logoColor=white)
![Claude Code](https://img.shields.io/badge/Claude_Code-Compatible-D97757?style=for-the-badge&logo=anthropic&logoColor=white)

> Written from a **senior expert full-stack AEM developer** perspective with deep knowledge of Sling, OSGi, JCR, and the Adobe ecosystem. Every rule delivers production-grade, enterprise-ready patterns — not beginner tutorials. Each file includes correct and incorrect code examples side by side, an anti-patterns section, and impact ratings (Critical / High / Medium).

Claude Code skills with **69 rule files** — battle-tested solutions from large-scale AEM as a Cloud Service implementations covering backend (Sling Models, servlets, OSGi, workflows), frontend (Webpack, ClientLibs, Core Components, Style System), authoring (Touch UI, Universal Editor, dialogs, policies), headless (GraphQL, Content Fragments, SPA/headless SDKs), and infrastructure (Cloud Manager, Dispatcher, CDN, Dynamic Media, performance optimization).

## Skills

### 1. `aem-best-practices` — Full-Stack AEM (52 rules)

```
aem-best-practices/
├── SKILL.md
└── rules/
    ├── build-pipeline/                          # Build, Workflow, Testing, Config (9)
    │   ├── ui-frontend-structure.md             # Webpack (common/dev/prod), SCSS/TS, clientlib-generator
    │   ├── clientlibs-configuration.md          # ClientLibs, proxy, HTL
    │   ├── frontend-dev-workflow.md             # SDK, HMR, debugging, Jest, Git
    │   ├── osgi-frontend-config.md              # Externalizer, CORS, env vars
    │   ├── testing-quality.md                   # AEM Mocks, Cypress, Playwright
    │   ├── migration-patterns.md                # JSP→HTL, 6.5→Cloud, Coral 2→3
    │   ├── cloud-manager-deployment.md          # Pipelines, quality gates, env vars, rollback
    │   ├── rapid-dev-environments.md            # RDE setup, AIO CLI, iteration workflow
    │   └── content-transfer-tool.md             # CTT extraction/ingestion, BPA, user mapping
    ├── component-development/                   # Components & Templates (10)
    │   ├── core-components-frontend.md          # BEM naming, proxy, delegation, JS hooks
    │   ├── component-dialogs.md                 # Full Granite UI field reference, multifield, validation
    │   ├── style-system.md                      # Layout/display styles, CSS patterns, policies
    │   ├── htl-templating.md                    # Block statements, XSS contexts
    │   ├── component-architecture.md            # Hierarchy, containers, versioning
    │   ├── spa-editor.md                        # React/Angular SPA SDK (deprecated)
    │   ├── accessibility-seo.md                 # WCAG 2.1 AA, SEO, JSON-LD
    │   ├── forms-adaptive.md                    # Adaptive Forms v3, theming
    │   ├── dam-asset-patterns.md                # Asset microservices, processing profiles, metadata
    │   └── universal-editor-aemaacs.md          # UE instrumentation, data-aue-*, models, CORS
    ├── java/                                    # Java & Backend (5)
    │   ├── sling-models-frontend.md             # Annotations, injection, JSON export
    │   ├── lombok-best-practices.md             # @Getter/@Slf4j with Sling Models
    │   ├── aem-workflows.md                     # WorkflowProcess, launchers, transient, payloads
    │   ├── sling-servlets.md                    # Resource types, URL decomposition, CSRF
    │   └── osgi-services-schedulers.md          # DS annotations, Sling Jobs, event handlers
    ├── analytics-tracking/                      # Analytics & Personalization (2)
    │   ├── adobe-data-layer.md                  # ACDL push from Java/HTL/JS, GTM bridge
    │   └── personalization-targeting.md          # ContextHub, Adobe Target, flicker
    ├── touch-ui/                                # Touch UI Authoring (12)
    │   ├── coral-ui-granite-frontend.md         # Coral UI 3 API
    │   ├── touch-ui-page-editor.md              # Editor architecture, layers
    │   ├── edit-config-behavior.md              # cq:editConfig, drop targets
    │   ├── rte-configuration.md                 # RTE plugins, toolbar
    │   ├── dialog-clientlibs-patterns.md        # Show/hide, validation
    │   ├── dialog-advanced-fields.md            # Pickers, upload, rendercondition
    │   ├── dialog-custom-widgets.md             # Custom fields, transforms
    │   ├── dialog-datasources.md                # DataSource servlets
    │   ├── sling-resource-merger-overlays.md    # Overlays, Resource Merger
    │   ├── templates-page-properties-policies.md # Templates, policies
    │   ├── security-patterns.md                 # XSS, CSRF, CSP, CORS
    │   └── user-management-acls.md              # IMS, Repo Init, ACLs, CUG, service users
    ├── layout/                                  # Layout (1)
    │   └── responsive-grid.md                   # Breakpoints, grid classes, nesting, policies
    ├── headless/                                # Headless & Content (6)
    │   ├── graphql-headless.md                  # Persisted queries, filtering, pagination, SDK
    │   ├── content-fragments.md                 # CF Models, variations, APIs
    │   ├── experience-fragments.md              # XF, Target export, SDI
    │   ├── headless-sdk-frameworks.md           # React/Next/Vue/Svelte SDKs
    │   ├── openapi-events.md                    # CF OpenAPI, AEM Eventing
    │   └── commerce-cif.md                      # CIF Core Components, GraphQL, catalog caching
    ├── multi-tenant/                            # Multi-Tenant (1)
    │   └── aemaacs-multi-tenant.md              # MSM, CAConfig, i18n
    └── performance/                             # Performance & CDN (6)
        ├── cloud-performance.md                 # CDN tiers, Cache-Control, stale-while-revalidate
        ├── cdn-configuration.md                 # Fastly config, BYOCDN, WAF, ESI
        ├── frontend-optimization.md             # CWV, capo.js, RUM, TTFB, HTTP/3
        ├── dynamic-media-assets.md              # Smart crops, WOID, responsive
        ├── query-optimization.md                # QueryBuilder, Oak indexes
        └── dispatcher-caching.md                # Cache rules, SDI, statfileslevel
```

### 2. `aem-eds-frontend-best-practices` — Edge Delivery Services (17 rules)

```
aem-eds-frontend-best-practices/
├── SKILL.md
└── rules/
    ├── block-development/                       # Blocks & Code (11)
    │   ├── eds-block-development.md             # decorate(), CSS scoping
    │   ├── eds-content-modeling.md              # JSON definitions, models
    │   ├── vanilla-js-patterns.md               # DOM, events, async, components
    │   ├── css-best-practices.md                # Custom properties, nesting, modern CSS
    │   ├── eds-experimentation.md               # A/B testing, audiences, RUM
    │   ├── eds-forms-sheets.md                  # Forms, spreadsheet data
    │   ├── eds-testing.md                       # Jest, Lighthouse CI, Playwright
    │   ├── service-workers.md                   # Offline fallback, caching, background sync
    │   ├── web-components.md                    # Custom elements, Shadow DOM, slots
    │   ├── edge-compute.md                      # Cloudflare Workers, auth, A/B at edge
    │   └── third-party-integrations.md          # Analytics, chat, payment, consent, facades
    ├── authoring/                               # Authoring (3)
    │   ├── universal-editor.md                  # data-aue-* attributes
    │   ├── eds-standard-blocks.md               # Adobe blocks + UE JSON models
    │   └── eds-sidekick-seo.md                  # Sidekick, SEO, redirects
    ├── multi-tenant/                            # Multi-Site (1)
    │   └── eds-multi-site.md                    # Repoless, theming, shared blocks
    └── performance/                             # Performance & CDN (2)
        ├── eds-performance.md                   # RUM, TTFB, HTTP/3, bfcache
        └── eds-cdn-configuration.md             # CDN, domains, headers, redirects
```

## Installation

```bash
# Via skills.sh
npx skills add <your-github-username>/aem-skills

# Or manually via symlink
ln -s /path/to/aem-skills/aem-best-practices ~/.claude/skills/aem-best-practices
ln -s /path/to/aem-skills/aem-eds-frontend-best-practices ~/.claude/skills/aem-eds-frontend-best-practices
```

## Sources

Official Adobe documentation (experienceleague.adobe.com, aem.live), Adobe GitHub repos.
