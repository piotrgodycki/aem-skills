---
title: OSGi & Configuration Affecting Frontend
impact: HIGH
impactDescription: Misconfigured OSGi services break external links, CORS, authentication, and environment-specific rendering on publish
tags: osgi, cors, externalizer, referrer-filter, sling-mapping, clientlibs, html-library-manager, cloud-manager, environment-variables, repo-init
---

## OSGi & Configuration Affecting Frontend

AEM Cloud Service uses `.cfg.json` OSGi configuration files deployed through Cloud Manager. Many OSGi services directly affect frontend behavior: link generation, cross-origin requests, authentication, asset processing, and HTML delivery. All configurations live under `/apps/<project>/osgiconfig/` with run-mode folder targeting.

### Run-Mode Folder Structure

```
/apps/myproject/osgiconfig/
├── config/                          # All environments, all tiers
├── config.author/                   # Author tier only
├── config.publish/                  # Publish tier only
├── config.author.dev/               # Author tier, dev environment
├── config.author.stage/             # Author tier, stage environment
├── config.author.prod/              # Author tier, production
├── config.publish.dev/              # Publish tier, dev environment
├── config.publish.stage/            # Publish tier, stage environment
└── config.publish.prod/             # Publish tier, production
```

**Resolution rule:** The configuration with the highest number of matching run modes wins entirely. You cannot define partial properties in one folder and override others in a more specific folder -- the most specific match replaces the entire PID configuration.

### Externalizer Configuration

The Day CQ Link Externalizer (`com.day.cq.commons.impl.ExternalizerImpl`) transforms internal paths into absolute external URLs.

**File:** `com.day.cq.commons.impl.ExternalizerImpl.cfg.json`

```json
{
  "externalizer.domains": [
    "local http://localhost:4502",
    "author https://author-p12345-e67890.adobeaemcloud.com",
    "publish https://publish-p12345-e67890.adobeaemcloud.com"
  ]
}
```

**Cloud Service restrictions:**
- The four default domains (`local`, `author`, `publish`, `preview`) must always remain configured
- Do not override `EXTERNALIZER` environment variables directly; use `AEM_CDN_DOMAIN_PUBLISH` and `AEM_CDN_DOMAIN_PREVIEW` instead
- Cloud Service instances read domains from Cloud Manager environment variables at runtime
- Always use HTTPS for external links

**Usage in Sling Models / Java:**

```java
@Reference
private Externalizer externalizer;

// Publish-facing URL
String url = externalizer.publishLink(resourceResolver, "/content/mysite/en.html");

// Author-facing URL
String authorUrl = externalizer.authorLink(resourceResolver, "/content/mysite/en.html");
```

### Sling Mapping (/etc/map)

Resource mapping defines URL shortening, vanity URLs, and domain mapping. Mappings live under `/etc/map` (author) and `/etc/map.publish` (publish).

```
/etc/map/
├── http/
│   └── localhost.4502/
│       ├── sling:match = "localhost.4502/"
│       └── sling:internalRedirect = "/content/mysite/"
└── https/
    └── mysite.com/
        ├── sling:match = "mysite.com/"
        └── sling:internalRedirect = "/content/mysite/"
```

**Key details:**
- Vanity URLs do not support regex patterns
- The JCR Resource Resolver processes two lists top-to-bottom: Resolver Map Entries (URL to resource) and Mapping Map Entries (resource path to URL)
- View active mappings at `/system/console/jcrresolver`
- On Cloud Service, `/etc/map` must be deployed via content packages or Repo Init

### Referrer Filter OSGi Configuration

Controls which origins can make POST/PUT/DELETE requests (critical for headless SPA and external form submissions).

**File:** `org.apache.sling.security.impl.ReferrerFilter.cfg.json`

```json
{
  "allow.empty": false,
  "allow.hosts": [
    "my-spa.example.com",
    "my-mobile-app.example.com"
  ],
  "allow.hosts.regexp": [
    "https://.*\\.example\\.com:443"
  ],
  "filter.methods": [
    "POST",
    "PUT",
    "DELETE",
    "COPY",
    "MOVE"
  ],
  "exclude.agents.regexp": [
    ""
  ]
}
```

**Security warning:** Never use wildcard `*` in `allow.hosts` -- it disables authenticated access to GraphQL endpoints and exposes them publicly. Grant access exclusively to trusted domains.

### CORS Configuration

Cross-Origin Resource Sharing is managed via factory OSGi configurations. Each policy is a separate instance.

**File:** `com.adobe.granite.cors.impl.CORSPolicyImpl~myproject-graphql.cfg.json`

Place in `config.author/` for author-tier (OSGi-based CORS):

```json
{
  "alloworigin": [
    "https://spa.example.com",
    "https://app.example.com"
  ],
  "alloworiginregexp": [
    "https://.*\\.example\\.com"
  ],
  "allowedpaths": [
    "/graphql/execute.json.*",
    "/content/experience-fragments/.*",
    "/api/assets/.*"
  ],
  "supportedheaders": [
    "Origin",
    "Accept",
    "X-Requested-With",
    "Content-Type",
    "Access-Control-Request-Method",
    "Access-Control-Request-Headers",
    "Authorization"
  ],
  "supportedmethods": [
    "GET",
    "HEAD",
    "POST"
  ],
  "maxage:Integer": 1800,
  "supportscredentials": true,
  "exposedheaders": [""]
}
```

**Tier differences:**
- **Author tier:** CORS is configured via OSGi in `config.author/`. Include `Authorization` in `supportedheaders` and set `supportscredentials: true`
- **Publish/Preview tier:** CORS is configured in Dispatcher vhost configuration, not via OSGi. Remove `Origin` from `clientheaders.any` so Dispatcher manages CORS headers

### Apache Sling Authentication Handler

**File:** `org.apache.sling.engine.impl.auth.SlingAuthenticator.cfg.json`

```json
{
  "auth.sudo.cookie": "sling.sudo",
  "auth.sudo.parameter": "sudo",
  "auth.annonymous": false,
  "sling.auth.requirements": [
    "+/content/mysite",
    "-/content/mysite/public"
  ]
}
```

- Use `+` prefix to require authentication, `-` to allow anonymous access
- Critical for gated content and preview environments
- On publish tier, authentication requirements affect CDN caching behavior

### AEM Link Checker Configuration

**File:** `com.day.cq.rewriter.linkchecker.impl.LinkCheckerImpl.cfg.json`

```json
{
  "service.check_external": false,
  "service.special_link_prefix": [
    "javascript:",
    "data:",
    "mailto:",
    "tel:",
    "#"
  ],
  "service.special_link_patterns": [
    "https://external-app\\.example\\.com/.*"
  ]
}
```

Disable external link checking in author to prevent performance issues with SPAs and headless apps that generate dynamic URLs.

### DAM Asset Processing Profiles

Processing profiles determine which renditions are generated for uploaded assets. These directly affect frontend image delivery.

- **Standard processing:** Uses Asset Compute microservices (Cloud Service only)
- **Custom profiles:** Define rendition widths, formats (JPEG, WebP, AVIF), and quality
- **Dynamic Media:** Enables on-the-fly image transformation via URL parameters
- **Web-optimized delivery:** Automatic WebP/AVIF serving based on browser support

Configure profiles in AEM Author at **Tools > Assets > Processing Profiles** and apply to DAM folders.

### HTML Library Manager Settings

The HTML Library Manager controls minification, debug mode, and gzip compression for ClientLibs.

**File:** `com.adobe.granite.ui.clientlibs.impl.HtmlLibraryManagerImpl.cfg.json`

```json
{
  "htmllibmanager.minify": true,
  "htmllibmanager.debug": false,
  "htmllibmanager.gzip": true,
  "htmllibmanager.timing": false,
  "htmllibmanager.debug.init.js": "granite.utils"
}
```

**Cloud Service behavior:**
- Minification uses Google Closure Compiler (GCC) for JS with ECMASCRIPT_2018 target
- Per-clientlib overrides via node properties: `cssProcessor` and `jsProcessor`
- Provide raw (unminified) source files; let the platform handle optimization
- Debug mode (`?debugClientLibs=true`) only works on author with appropriate permissions

### Sling Rewriter Pipeline

Configure link rewriting for CSS/JS references and content links.

**File:** `/apps/myproject/config/rewriter/myproject-transformer`

```xml
<!-- .content.xml -->
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="nt:unstructured"
    contentTypes="[text/html]"
    enabled="{Boolean}true"
    generatorType="htmlparser"
    order="{Long}1"
    serializerType="htmlwriter"
    transformerTypes="[linkchecker,linkrewriter]"/>
```

Use rewriter pipelines to transform absolute URLs to relative, rewrite asset paths for CDN, or strip author-specific link prefixes.

### Environment-Specific Configuration with Variables

#### Environment Variables (non-secret)

```json
{
  "api.endpoint": "$[env:API_ENDPOINT;default=https://api.dev.example.com]",
  "analytics.id": "$[env:ANALYTICS_SUITE_ID;default=dev-suite]"
}
```

#### Secret Variables

```json
{
  "api.key": "$[secret:API_KEY]",
  "service.password": "$[secret:SERVICE_PASSWORD]"
}
```

**Rules for variable names:**
- Length: 2--100 characters
- Pattern: `[a-zA-Z_][a-zA-Z_0-9]*`
- Reserved prefixes (ignored if set by customers): `INTERNAL_`, `ADOBE_`, `CONST_`
- Adobe public API prefix: `AEM_`
- Maximum 200 variables per environment, values up to 2048 characters

**Set variables via Cloud Manager CLI:**

```bash
# Environment variable
aio cloudmanager:set-environment-variables <ENV_ID> --variable API_ENDPOINT "https://api.prod.example.com"

# Secret variable
aio cloudmanager:set-environment-variables <ENV_ID> --secret API_KEY "abc123"
```

**Or via API:**

```bash
PATCH /program/{programId}/environment/{environmentId}/variables
```

### Repo Init Scripts

Repo Init creates initial repository structures, service users, and ACLs at deployment time. Configured as OSGi properties on `org.apache.sling.jcr.repoinit.RepositoryInitializer`.

**File:** `org.apache.sling.jcr.repoinit.RepositoryInitializer~myproject.cfg.json`

```json
{
  "scripts": [
    "create service user myproject-service\nset ACL for myproject-service\n  allow jcr:read on /content/mysite\n  allow jcr:read on /content/dam/mysite\nend\n\ncreate path (sling:Folder) /content/mysite/config\nset ACL for everyone\n  allow jcr:read on /content/mysite/config\nend"
  ]
}
```

**Common Repo Init operations:**
- Create service users for backend integrations
- Set ACLs for content paths
- Create initial folder structures
- Register namespaces and node types
- Set properties on existing nodes

### Content Distribution (Publish/Invalidation)

AEM Cloud Service uses Sling Content Distribution instead of traditional replication agents:
- **Forward distribution:** Author to publish (automatic on activation)
- **Invalidation:** CDN cache purge on publish activation
- **Golden publish:** Content is distributed to all publish instances atomically
- No custom replication agents -- use Cloud Service APIs for programmatic activation

### Anti-Patterns

```
AVOID these common mistakes:

1. Hardcoded URLs in OSGi config
   BAD:  "api.url": "https://api.prod.example.com"
   GOOD: "api.url": "$[env:API_ENDPOINT;default=https://api.dev.example.com]"

2. Publish-tier config in author folder (or vice versa)
   BAD:  config.author/com.adobe.granite.cors.impl.CORSPolicyImpl~publish.cfg.json
   GOOD: config.publish/ for publish-specific, config.author/ for author-specific

3. Missing environment overrides
   BAD:  Only config/ with hardcoded prod values
   GOOD: config/ with defaults + config.publish.prod/ with production overrides

4. Wildcard referrer filter
   BAD:  "allow.hosts": ["*"]
   GOOD: "allow.hosts": ["my-spa.example.com"]

5. CORS on publish via OSGi instead of Dispatcher
   BAD:  config.publish/CORSPolicyImpl.cfg.json
   GOOD: Dispatcher vhost configuration for publish-tier CORS

6. Secrets in Git
   BAD:  "api.key": "abc123secret"
   GOOD: "api.key": "$[secret:API_KEY]"

7. Custom run modes (not supported in Cloud Service)
   BAD:  config.mysite/, config.brand-a/
   GOOD: config.author.dev/, config.publish.prod/

8. Deploying /etc/map without corresponding Dispatcher rules
   BAD:  Sling mapping configured but Dispatcher rewrites missing
   GOOD: /etc/map AND Dispatcher rewrite rules aligned
```
