---
title: CDN Configuration for AEM as a Cloud Service
impact: HIGH
impactDescription: Correct CDN configuration reduces latency by serving content from edge nodes, offloads origin traffic, and provides security filtering — misconfigurations cause cache misses, security gaps, or downtime
tags: cdn, fastly, caching, waf, traffic-filter, redirects, esi, custom-domain, byocdn, edge, purge, headers, cloud-manager, performance
---

## CDN Configuration for AEM as a Cloud Service

AEM as a Cloud Service includes an Adobe-managed CDN (powered by Fastly) that is automatically provisioned with every program. All traffic to AEM Publish flows through this CDN. Customers may optionally place their own CDN in front of Adobe's CDN (BYOCDN).

---

### 1. Adobe-Managed CDN (Fastly)

The Adobe-managed CDN is the default delivery layer included with every Cloud Service program. It is fully managed, optimized for AEM, and requires no separate contract.

**Architecture:**

```
Browser --> Adobe CDN (Fastly) --> Dispatcher (Apache httpd) --> AEM Publish
```

**Key capabilities:**

- Traffic filter rules (including WAF with Extended Security license)
- Request and response transformations (header manipulation, URL rewrites)
- Server-side redirects at CDN edge (301/302/307/308)
- Origin selectors (route to non-AEM backends per path)
- Edge Side Includes (ESI) for partial page caching
- Geolocation headers (`x-aem-client-country`, `x-aem-client-continent`)
- Custom CDN error pages
- Basic authentication for content protection
- Automatic marketing parameter stripping

**Configuration is code-based** -- all features are declared in YAML files under a `config/` folder in your Git repository and deployed via Cloud Manager config pipelines.

---

### 2. Customer-Managed CDN (BYOCDN)

A customer CDN (Akamai, Cloudflare, AWS CloudFront, etc.) can be placed in front of the Adobe-managed CDN. This is supported for the **publish** and **preview** tiers only -- never for author.

**When to use BYOCDN:**

- Existing CDN investment that would be costly to replace
- Regulatory or contractual requirement for a specific CDN vendor
- Advanced edge logic not available on Adobe CDN

**Configuration requirements:**

| Requirement | Details |
|-------------|---------|
| Origin hostname | `publish-p<PROGRAM_ID>-e<ENV_ID>.adobeaemcloud.com` |
| Host header | Must match the Adobe origin hostname |
| X-Forwarded-Host | Customer's production domain (e.g., `www.example.com`) |
| X-AEM-Edge-Key | Shared secret validating the customer CDN (minimum 32 bytes) |
| SNI | Must match the Adobe ingress domain |
| TLS | TLS 1.2 or higher required |

**YAML configuration for edge authentication:**

```yaml
# config/cdn.yaml
kind: "CDN"
version: "1"
data:
  authentication:
    authenticators:
      - name: edge-auth
        type: edge
        edgeKey1: ${{CDN_EDGEKEY_052824}}
        edgeKey2: ${{CDN_EDGEKEY_041425}}
    rules:
      - name: edge-auth-rule
        when: { reqProperty: tier, equals: "publish" }
        action:
          type: authenticate
          authenticator: edge-auth
```

The dual-key design (`edgeKey1` / `edgeKey2`) allows zero-downtime key rotation. Introduce the new key in the secondary slot, verify, then remove the old key.

**Debugging BYOCDN:**

```bash
# Verify edge key and host values reach Adobe CDN
curl -svo /dev/null https://publish-p12345-e67890.adobeaemcloud.com/page.html \
  -H "X-AEM-Edge-Key: <your-key>" \
  -H "X-Forwarded-Host: www.example.com" \
  -H "x-aem-debug: edge=true"
```

The `x-aem-debug: edge=true` header returns diagnostic information about authentication status and configuration.

---

### 3. CDN Configuration as Code

All CDN features are declared in YAML files placed under a top-level `config/` folder and deployed through Cloud Manager's **config pipeline**.

**Base file structure:**

```yaml
# config/cdn.yaml
kind: "CDN"
version: "1"
data:
  # One or more of the following sections:
  trafficFilters:
    rules: [...]
  requestTransformations:
    rules: [...]
  responseTransformations:
    rules: [...]
  redirects:
    rules: [...]
  originSelectors:
    rules: [...]
    origins: [...]
  authentication:
    authenticators: [...]
    rules: [...]
```

**Constraint:** The total configuration file size (including traffic filter rules) must not exceed **100 KB**.

**Evaluation order:** Traffic Filters -> Request Transformations -> Origin Selectors -> Response Transformations -> Redirects.

#### 3.1 Traffic Filter Rules

Traffic filter rules control which requests are allowed, blocked, or logged at the CDN edge before reaching the origin.

**Available condition properties:**

- Request: `path`, `url`, `queryString`, `method`, `tier`, `domain`
- Client: `clientIp`, `clientCountry`, `clientContinent`, `clientRegion`, `clientAsNumber`, `clientAsName`
- Headers/params: `reqHeader`, `queryParam`, `reqCookie`, `postParam`

**Predicate operators:** `equals`, `doesNotEqual`, `like`, `notLike`, `matches`, `doesNotMatch`, `in`, `notIn`, `exists`

**Correct -- IP allowlist with rate limiting:**

```yaml
kind: "CDN"
version: "1"
data:
  trafficFilters:
    rules:
      # Block known bad countries (OFAC)
      - name: block-ofac-countries
        when:
          allOf:
            - { reqProperty: tier, in: ["author", "publish"] }
            - reqProperty: clientCountry
              in: [SY, BY, MM, KP, IQ, CD, SD, IR, LR, ZW, CU, CI]
        action: block

      # Rate limit by client IP -- 60 requests per 10 seconds
      - name: rate-limit-by-ip
        when:
          reqProperty: tier
          equals: publish
        rateLimit:
          limit: 60
          window: 10
          penalty: 300
          count: all
          groupBy:
            - reqProperty: clientIp
        action: block

      # Allow known partner IPs
      - name: allow-partner-ips
        when:
          allOf:
            - { reqProperty: path, like: "/api/*" }
            - reqProperty: clientIp
              in: ["203.0.113.0/24", "198.51.100.0/24"]
        action:
          type: allow
```

**Rate limiting parameters:**

| Parameter | Values | Description |
|-----------|--------|-------------|
| `limit` | 10--10000 | Max requests per second (averaged over window) |
| `window` | 1, 10, or 60 | Sampling window in seconds |
| `penalty` | 60--3600 | Block duration in seconds after limit exceeded |
| `count` | `all`, `fetches`, `errors` | What to count (all requests, origin fetches, or errors) |
| `groupBy` | `clientIp`, etc. | Aggregation key |

Rate limits are evaluated **per CDN Point of Presence (PoP)**, not globally.

#### 3.2 WAF Rules (Extended Security License)

WAF traffic filter rules add specialized flags for detecting attack patterns. They require the Extended Security license add-on.

```yaml
kind: "CDN"
version: "1"
data:
  trafficFilters:
    rules:
      # Block attacks from known bad IPs (lowest false-positive rate)
      - name: block-attacks-from-bad-ips
        when:
          reqProperty: tier
          in: ["author", "publish"]
        action:
          type: block
          wafFlags:
            - ATTACK-FROM-BAD-IP

      # Log all attack patterns for analysis
      - name: log-all-attacks
        when:
          reqProperty: tier
          in: ["author", "publish"]
        action:
          type: log
          wafFlags:
            - ATTACK

      # Block SQL injection and XSS with alerting
      - name: block-sqli-xss
        when:
          reqProperty: tier
          equals: publish
        action:
          type: block
          alert: true
          wafFlags:
            - SQLI
            - XSS
```

**Key WAF flags:**

| Category | Flags |
|----------|-------|
| Malicious traffic | `SQLI`, `XSS`, `CMDEXE`, `TRAVERSAL`, `BACKDOOR`, `LOG4J-JNDI`, `CODEINJECTION`, `ATTACK`, `ATTACK-FROM-BAD-IP` |
| Suspicious traffic | `BAD-IP`, `BHH`, `ABNORMALPATH`, `NOUA`, `SCANNER`, `PRIVATEFILE`, `NULLBYTE`, `OOB-DOMAIN` |
| Miscellaneous | `DATACENTER`, `TORNODE`, `SANS`, `CVE`, `DOUBLEENCODING`, `JSON-ERROR`, `XML-ERROR` |

Setting `alert: true` triggers Actions Center notifications when a rule fires 10+ times within a 5-minute window.

**Important:** `allow` rules take precedence over `block` rules when conflicts occur. Start with `action: log` in production, then switch to `block` after verifying no false positives.

#### 3.3 Request Transformations

Modify incoming requests before they reach the origin.

```yaml
kind: "CDN"
version: "1"
data:
  requestTransformations:
    removeMarketingParams: true
    rules:
      # Add custom header for origin routing
      - name: add-routing-header
        when:
          reqProperty: path
          like: "/api/*"
        actions:
          - type: set
            reqHeader: x-api-version
            value: "v2"

      # Store a variable for later use in response transformations
      - name: capture-region
        when:
          reqProperty: clientCountry
          in: [US, CA, MX]
        actions:
          - type: set
            var: region
            value: americas
```

**Supported actions:** `set` (header, cookie, query param, variable), `unset` (remove header/param), `transform` (regex replace, lowercase).

#### 3.4 Response Transformations

Modify CDN responses before they reach the client.

```yaml
kind: "CDN"
version: "1"
data:
  responseTransformations:
    rules:
      # Set CORS headers at CDN level
      - name: cors-headers
        when:
          reqProperty: path
          like: "/api/*"
        actions:
          - type: set
            respHeader: Access-Control-Allow-Origin
            value: "https://www.example.com"
          - type: set
            respHeader: Access-Control-Allow-Methods
            value: "GET, POST, OPTIONS"

      # Add security headers globally
      - name: security-headers
        when:
          reqProperty: tier
          equals: publish
        actions:
          - type: set
            respHeader: X-Content-Type-Options
            value: "nosniff"
          - type: set
            respHeader: X-Frame-Options
            value: "SAMEORIGIN"
```

#### 3.5 CDN-Level Redirects

Server-side redirects processed at the CDN edge, faster than Dispatcher or AEM-level redirects.

```yaml
kind: "CDN"
version: "1"
data:
  redirects:
    rules:
      # Simple path redirect
      - name: old-page-redirect
        when:
          reqProperty: path
          equals: "/old-page.html"
        action:
          type: redirect
          status: 301
          location: /new-page

      # Pattern-based redirect with regex
      - name: blog-redirect
        when:
          reqProperty: path
          matches: "^/blog/2023/(.*)"
        action:
          type: redirect
          status: 301
          location: "/articles/2023/$1"

      # Domain redirect
      - name: legacy-domain-redirect
        when:
          reqProperty: domain
          equals: "old.example.com"
        action:
          type: redirect
          status: 301
          location: "https://www.example.com"
```

Supported status codes: 301, 302, 303, 307, 308.

#### 3.6 Origin Selectors

Route specific URL patterns to non-AEM backends (third-party APIs, SPA hosting, Edge Delivery Services).

```yaml
kind: "CDN"
version: "1"
data:
  originSelectors:
    origins:
      - name: my-api
        domain: api.example.com
        # domain must NOT contain .adobeaemcloud.com (prevents loops)
      - name: eds-origin
        domain: main--mysite--myorg.aem.live
    rules:
      # Route API paths to external backend
      - name: api-routing
        when:
          reqProperty: path
          like: "/api/*"
        action:
          type: selectOrigin
          originName: my-api

      # Route EDS paths (scripts, styles, blocks)
      - name: eds-routing
        when:
          reqProperty: path
          like: "/blog/*"
        action:
          type: selectOrigin
          originName: eds-origin
```

**Incorrect -- origin domain pointing to adobeaemcloud.com:**

```yaml
# WRONG: This creates a request loop
origins:
  - name: bad-origin
    domain: publish-p12345-e67890.adobeaemcloud.com
```

---

### 4. CDN Caching Configuration

The Adobe CDN respects standard HTTP caching headers from the origin (AEM Publish via Dispatcher).

#### 4.1 Cache Control Headers

| Header | Purpose |
|--------|---------|
| `Cache-Control` | Primary cache directive. `private`, `no-cache`, `no-store` bypass CDN cache |
| `Surrogate-Control` | CDN-specific directive (respected by Adobe CDN, stripped before browser delivery) |
| `Expires` | Legacy expiration header |

Responses containing `Set-Cookie` headers bypass CDN caching entirely.

#### 4.2 Default TTLs

| Content Type | Default CDN TTL |
|-------------|-----------------|
| HTML | 5 minutes (configurable via `EXPIRATION_TIME` or `mod_headers`) |
| Client libraries (JS/CSS) | 30 days / immutable (content-hash in URL) |
| Images/blobs | Respect `Cache-Control` from origin |

**Correct -- set stale-while-revalidate for graceful cache refresh:**

```apache
# dispatcher/src/conf.d/available_vhosts/default.vhost
<LocationMatch "^/content/.*\.html$">
    Header set Cache-Control "max-age=300, stale-while-revalidate=1800, stale-if-error=3600"
    Header set Surrogate-Control "max-age=600"
</LocationMatch>
```

**Best practice:** Add `stale-while-revalidate` (30 minutes recommended) to all cache-control headers. This enables the CDN to serve stale content while revalidating in the background, preventing cache stampede.

#### 4.3 Marketing Parameter Stripping

For environments created after October 2023, the CDN automatically strips common marketing query parameters before evaluating the cache key:

```
utm_source, utm_medium, utm_campaign, utm_term, utm_content,
gclid, gdftrk, _ga, mc_*, trk_*, dm_i, _ke, sc_*, fbclid, msclkid, ttclid
```

Regex pattern: `^(utm_.*|gclid|gdftrk|_ga|mc_.*|trk_.*|dm_i|_ke|sc_.*|fbclid|msclkid|ttclid)$`

To disable:

```yaml
# config/cdn.yaml
kind: "CDN"
version: "1"
data:
  requestTransformations:
    removeMarketingParams: false
```

#### 4.4 Cache Key

The CDN cache key includes:
- Full request URL (path + query parameters after marketing param stripping)
- `Vary` header values (typically `Accept-Encoding`)

**Incorrect -- using Vary: User-Agent (fragments cache):**

```apache
# WRONG: Creates separate cache entries per user agent string
Header set Vary "Accept-Encoding, User-Agent"
```

**Correct -- only use Vary for encoding:**

```apache
Header set Vary "Accept-Encoding"
```

#### 4.5 Cache Purge

**Via Cloud Manager UI:** Manual purge from environment details page.

**Via Purge API (programmatic):**

```yaml
# config/cdn.yaml -- configure purge API token
kind: "CDN"
version: "1"
data:
  authentication:
    authenticators:
      - name: purge-auth
        type: purge
        purgeKey1: ${{CDN_PURGEKEY_031224}}
        purgeKey2: ${{CDN_PURGEKEY_021225}}
    rules:
      - name: purge-auth-rule
        when: { reqProperty: tier, equals: publish }
        action:
          type: authenticate
          authenticator: purge-auth
```

```bash
# Purge a specific URL
curl -X PURGE "https://publish-p12345-e67890.adobeaemcloud.com/content/page.html" \
  -H "X-AEM-Purge-Key: <your-purge-key>"
```

**Important:** The CDN does not automatically flush when Dispatcher invalidation occurs. The CDN respects TTLs independently. Use Surrogate-Control to set CDN-specific TTLs shorter than browser TTLs.

---

### 5. Custom Domain Configuration

Custom domains are managed in Cloud Manager and mapped to the AEM-managed CDN.

#### 5.1 Setup Steps

1. **Add SSL certificate** to Cloud Manager (DV via Let's Encrypt or upload custom EV/OV)
2. **Add custom domain name** (verified via DNS)
3. **Create domain mapping** (domain -> environment + CDN type + certificate)
4. **Configure DNS** (only after mapping is complete)
5. **Verify status** in Cloud Manager

**DNS configuration:**

| Domain Type | Record Type | Value |
|-------------|-------------|-------|
| Subdomain (www) | CNAME | `cdn.adobeaemcloud.com` |
| Apex domain | A record | `151.101.3.10`, `151.101.67.10`, `151.101.131.10`, `151.101.195.10` |

**Verify without DNS propagation:**

```bash
curl -svo /dev/null https://www.example.com \
  --resolve www.example.com:443:151.101.3.10
```

#### 5.2 DV Certificate Provisioning

Adobe automatically provisions DV certificates via Let's Encrypt for Adobe-managed CDN domains. ACME validation is required. Certificates auto-renew before expiration.

#### 5.3 Constraints

- Maximum 500 custom domains per environment
- Wildcard domains (`*.example.com`) not supported
- Author tier does not support custom domains
- Domains cannot be shared across multiple environments
- One domain per addition (no bulk add)

---

### 6. Edge Side Includes (ESI)

ESI enables partial page caching at the CDN level. The parent page carries a long TTL, while ESI-included fragments have their own, shorter TTLs.

**How it works:**

```
1. Browser requests /page.html
2. CDN serves cached HTML with <esi:include> tags
3. CDN fetches each ESI fragment from origin (or fragment cache)
4. CDN assembles final response and delivers to browser
```

**ESI is enabled by default** on all AEMaaCS environments (including production). No manual activation is needed.

**Apache configuration for ESI pages:**

```apache
# dispatcher/src/conf.d/available_vhosts/default.vhost
<LocationMatch "^/content/mysite/.*\.html$">
    # Activate ESI processing at CDN
    Header set x-aem-esi "on"
    # Send uncompressed so CDN can parse ESI tags
    SetEnv no-gzip 1
    # Re-compress after ESI assembly
    Header set x-aem-compress "on"
    # Long TTL for parent page
    Header set Cache-Control "max-age=600"
</LocationMatch>
```

**AEM component HTL with ESI include:**

```html
<!-- Parent page template -->
<header>
    <esi:include src="/content/mysite/fragments/nav.html" />
</header>
<main>
    ${page.content @ context='html'}
</main>
<footer>
    <esi:include src="/content/mysite/fragments/footer.html" />
</footer>
```

**Fragment Sling servlet (short TTL):**

```java
// Servlet producing the nav fragment
response.setHeader("Cache-Control", "max-age=60");
response.setHeader("Content-Type", "text/html");
```

**Limitations:**

| Constraint | Value |
|------------|-------|
| Max nesting depth | 5 levels |
| Max fragments per page | 256 |
| Processing | Sequential (not concurrent) |

**Incorrect -- too many low-TTL ESI fragments:**

```html
<!-- WRONG: 50+ ESI includes with no-cache adds significant latency -->
<esi:include src="/fragment1.html" />
<esi:include src="/fragment2.html" />
<!-- ... 48 more fragments with Cache-Control: no-cache -->
```

Each ESI fragment is fetched sequentially. Many low-TTL or no-cache fragments compound latency.

**ESI vs SDI (Sling Dynamic Include):**

| Feature | ESI | SDI |
|---------|-----|-----|
| Processing layer | CDN (Fastly) | Dispatcher (Apache SSI) or CDN |
| Cache scope | CDN edge cache | Dispatcher cache or CDN |
| Fragment resolution | CDN fetches from origin | Dispatcher fetches from AEM |
| Use case | Personalized headers, dynamic nav | Component-level cache variation |
| Configuration | Apache headers + HTML tags | OSGi config + Sling resource types |

---

### 7. Performance Monitoring

#### 7.1 CDN Log Forwarding

Forward CDN logs in real time to external monitoring platforms.

**Supported destinations:**

| Destination | CDN Log Support |
|-------------|----------------|
| Splunk | Available |
| Datadog | Available |
| Azure Blob Storage | Available |
| Elasticsearch / OpenSearch | Available |
| HTTPS (generic webhook) | Available |
| Amazon S3 | Planned |
| New Relic | Planned |

**Configuration:**

```yaml
# config/logForwarding.yaml
kind: "LogForwarding"
version: "1"
data:
  splunk:
    default:
      enabled: true
      host: "splunk-host.example.com"
      token: "${{SPLUNK_TOKEN}}"
      index: "AEMaaCS"
    cdn:
      enabled: true
      token: "${{SPLUNK_TOKEN_CDN}}"
      index: "AEMaaCS_CDN"
```

Store tokens as Cloud Manager Secret Environment Variables (set Service Applied to "All").

**CDN log fields:**

| Field | Description |
|-------|-------------|
| `aem_env_id` | Environment identifier |
| `aem_env_type` | Environment type (dev/stage/prod) |
| `aem_program_id` | Cloud Manager program ID |
| `aem_tier` | publish / author / preview |
| `sourcetype` | `aemcdn` |

#### 7.2 CDN Analytics in Cloud Manager

Cloud Manager provides built-in CDN analytics:

- **Cache hit ratio** -- target 90%+ for production sites
- **Origin response time** -- monitor for regressions
- **5xx error rate** -- alert on spikes
- **Traffic volume** -- bandwidth and request counts

#### 7.3 Cache Debugging Headers

Use these response headers to diagnose caching behavior:

| Header | Meaning |
|--------|---------|
| `X-Cache` | `HIT` or `MISS` indicating CDN cache status |
| `Age` | Seconds since the response was cached at CDN |
| `X-Cache-Info` | Additional cache decision details |

```bash
# Check cache status of a URL
curl -sI https://www.example.com/page.html | grep -i 'x-cache\|age\|cache-control'
```

#### 7.4 Actions Center Alerts

Traffic filter rules with `alert: true` trigger notifications in Actions Center when a rule fires 10+ times within a 5-minute window. Use this for:

- DDoS detection (rate limit rules firing)
- WAF attack pattern alerts
- Suspicious traffic from specific regions

---

### Quick Reference: CDN Configuration Checklist

1. Place `cdn.yaml` under `config/` in your Git repository
2. Deploy via Cloud Manager config pipeline (target: dev -> stage -> prod)
3. Start traffic filter rules in `log` mode, then switch to `block` after validation
4. Set `stale-while-revalidate` on all cacheable responses
5. Verify cache hit ratio > 90% in Cloud Manager analytics
6. Configure log forwarding to your monitoring platform
7. For BYOCDN: validate X-AEM-Edge-Key and X-Forwarded-Host with `x-aem-debug: edge=true`
8. For custom domains: configure DNS only after domain mapping is verified in Cloud Manager
