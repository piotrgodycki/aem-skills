---
title: Sling Servlet Development Patterns
impact: HIGH
impactDescription: Sling Servlets expose HTTP endpoints — incorrect patterns cause security vulnerabilities, resource resolution failures, and poor API design
tags: sling-servlets, resource-types, url-decomposition, json-api, csrf, filters, post, get
---

## Sling Servlet Development Patterns

Sling Servlets handle HTTP requests in AEM. They bind to resource types (preferred) or paths, process requests, and return responses. Understanding Sling's URL decomposition and resource resolution is essential for correct servlet binding.

---

### 1. Servlet Registration: Resource Types vs Paths

#### @SlingServletResourceTypes (Preferred)

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "myproject/components/api/search",
    methods = HttpConstants.METHOD_GET,
    selectors = "data",
    extensions = "json"
)
public class SearchServlet extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {
        // Bound to: /content/mysite/search.data.json
        // (where search page has resourceType myproject/components/api/search)
        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");
        // ...
    }
}
```

#### @SlingServletPaths (Use Sparingly)

```java
@Component(service = Servlet.class)
@SlingServletPaths("/bin/myproject/healthcheck")
public class HealthCheckServlet extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("application/json");
        response.getWriter().write("{\"status\":\"ok\"}");
    }
}
```

**When to use each:**

| Approach | Use When | ACL | Caching |
|----------|----------|-----|---------|
| `@SlingServletResourceTypes` | Component-bound endpoints | Inherits resource ACLs | Dispatcher-cacheable |
| `@SlingServletPaths` | Global utility endpoints (/bin/) | **Must configure ACL manually** | Less cacheable |

**Always allow path servlets through Dispatcher:**

```
# dispatcher/src/conf.dispatcher.d/filters/filters.any
/0100 { /type "allow" /method "GET" /url "/bin/myproject/*" }
```

---

### 2. Sling URL Decomposition

Understanding how Sling decomposes URLs is critical:

```
/content/mysite/en/search.data.results.json/path/suffix?q=hello
|___________________________|____|_______|____|___________|______|
       Resource Path         Sel1  Sel2   Ext    Suffix    Query

Resource: /content/mysite/en/search
Selectors: ["data", "results"]
Extension: json
Suffix: /path/suffix
Query: q=hello
```

#### Accessing URL Components

```java
@Override
protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
        throws ServletException, IOException {
    // Resource path
    String resourcePath = request.getResource().getPath();

    // Selectors
    RequestPathInfo pathInfo = request.getRequestPathInfo();
    String[] selectors = pathInfo.getSelectors();       // ["data", "results"]
    String selectorString = pathInfo.getSelectorString(); // "data.results"

    // Extension
    String extension = pathInfo.getExtension(); // "json"

    // Suffix
    String suffix = pathInfo.getSuffix(); // "/path/suffix"
    Resource suffixResource = request.getRequestPathInfo().getSuffixResource();

    // Query parameters
    String query = request.getParameter("q"); // "hello"
    String[] multiValue = request.getParameterValues("tags");
}
```

---

### 3. GET Servlet Patterns

#### JSON API Response

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "myproject/components/api/products",
    methods = HttpConstants.METHOD_GET,
    selectors = "list",
    extensions = "json"
)
public class ProductListServlet extends SlingSafeMethodsServlet {

    private static final Logger log = LoggerFactory.getLogger(ProductListServlet.class);

    @Reference
    private ProductService productService;

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("application/json;charset=UTF-8");

        int offset = getIntParam(request, "offset", 0);
        int limit = getIntParam(request, "limit", 10);
        limit = Math.min(limit, 100); // cap maximum

        try {
            List<Product> products = productService.getProducts(
                request.getResourceResolver(), offset, limit);

            JsonObject result = new JsonObject();
            result.addProperty("offset", offset);
            result.addProperty("limit", limit);
            result.addProperty("total", productService.getTotal(request.getResourceResolver()));

            JsonArray items = new JsonArray();
            for (Product p : products) {
                JsonObject item = new JsonObject();
                item.addProperty("title", p.getTitle());
                item.addProperty("path", p.getPath());
                item.addProperty("price", p.getPrice());
                items.add(item);
            }
            result.add("items", items);

            response.getWriter().write(result.toString());

        } catch (Exception e) {
            log.error("Failed to fetch products", e);
            response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            response.getWriter().write("{\"error\":\"Internal server error\"}");
        }
    }

    private int getIntParam(SlingHttpServletRequest request, String name, int defaultValue) {
        String val = request.getParameter(name);
        if (val == null) return defaultValue;
        try {
            return Integer.parseInt(val);
        } catch (NumberFormatException e) {
            return defaultValue;
        }
    }
}
```

---

### 4. POST Servlet Patterns

#### SlingAllMethodsServlet with CSRF Protection

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "myproject/components/api/contact",
    methods = HttpConstants.METHOD_POST,
    extensions = "json"
)
public class ContactFormServlet extends SlingAllMethodsServlet {

    private static final Logger log = LoggerFactory.getLogger(ContactFormServlet.class);

    @Reference
    private EmailService emailService;

    @Override
    protected void doPost(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("application/json;charset=UTF-8");

        // Validate CSRF token (AEM provides this automatically for authenticated requests)
        // For anonymous POST, use custom validation

        // Input validation
        String name = sanitize(request.getParameter("name"));
        String email = sanitize(request.getParameter("email"));
        String message = sanitize(request.getParameter("message"));

        List<String> errors = new ArrayList<>();
        if (StringUtils.isBlank(name)) errors.add("Name is required");
        if (!isValidEmail(email)) errors.add("Valid email is required");
        if (StringUtils.isBlank(message)) errors.add("Message is required");

        if (!errors.isEmpty()) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            JsonObject errorResponse = new JsonObject();
            errorResponse.addProperty("success", false);
            JsonArray errorArray = new JsonArray();
            errors.forEach(errorArray::add);
            errorResponse.add("errors", errorArray);
            response.getWriter().write(errorResponse.toString());
            return;
        }

        try {
            emailService.sendContactForm(name, email, message);
            response.getWriter().write("{\"success\":true}");
        } catch (Exception e) {
            log.error("Failed to send contact form", e);
            response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            response.getWriter().write("{\"success\":false,\"error\":\"Failed to send\"}");
        }
    }

    private String sanitize(String input) {
        if (input == null) return "";
        return ESAPI.encoder().encodeForHTML(input.trim());
    }

    private boolean isValidEmail(String email) {
        return StringUtils.isNotBlank(email) && email.matches("^[\\w.%+-]+@[\\w.-]+\\.[a-zA-Z]{2,}$");
    }
}
```

#### CSRF Token in Frontend

```javascript
// Fetch CSRF token before POST (authenticated users)
async function submitForm(formData) {
  const tokenResponse = await fetch('/libs/granite/csrf/token.json');
  const { token } = await tokenResponse.json();

  const response = await fetch('/content/mysite/en/contact.json', {
    method: 'POST',
    headers: {
      'CSRF-Token': token,
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams(formData)
  });
  return response.json();
}
```

---

### 5. Servlet Filters

```java
@Component(
    service = Filter.class,
    property = {
        "sling.filter.scope=REQUEST",
        "sling.filter.pattern=/content/mysite/.*",
        "service.ranking:Integer=100"
    }
)
public class ApiRateLimitFilter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(ApiRateLimitFilter.class);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        SlingHttpServletRequest slingRequest = (SlingHttpServletRequest) request;

        // Only filter API requests
        if (!"json".equals(slingRequest.getRequestPathInfo().getExtension())) {
            chain.doFilter(request, response);
            return;
        }

        // Rate limit logic
        if (isRateLimited(slingRequest)) {
            SlingHttpServletResponse slingResponse = (SlingHttpServletResponse) response;
            slingResponse.setStatus(429);
            slingResponse.setContentType("application/json");
            slingResponse.getWriter().write("{\"error\":\"Rate limit exceeded\"}");
            return;
        }

        chain.doFilter(request, response);
    }

    @Override public void init(FilterConfig filterConfig) { }
    @Override public void destroy() { }
}
```

**Filter scopes:**

| Scope | When |
|-------|------|
| `REQUEST` | Every request |
| `COMPONENT` | Every included component (sling:include) |
| `ERROR` | Error handling |
| `INCLUDE` | RequestDispatcher.include() |
| `FORWARD` | RequestDispatcher.forward() |

---

### 6. Streaming Binary Responses

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "myproject/components/api/export",
    methods = HttpConstants.METHOD_GET,
    selectors = "csv",
    extensions = "csv"
)
public class CsvExportServlet extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/csv;charset=UTF-8");
        response.setHeader("Content-Disposition", "attachment; filename=\"export.csv\"");

        try (PrintWriter writer = response.getWriter()) {
            writer.println("Title,Path,Status");

            Iterator<Resource> children = request.getResource().listChildren();
            while (children.hasNext()) {
                Resource child = children.next();
                ValueMap props = child.getValueMap();
                writer.printf("%s,%s,%s%n",
                    escapeCsv(props.get("jcr:title", "")),
                    child.getPath(),
                    props.get("status", "draft"));
            }
        }
    }
}
```

---

### 7. Sling Resource Resolution Order

Understanding resolution priority prevents servlet conflicts:

```
1. Servlet by exact resource type + selector + extension + method
2. Servlet by resource type + selector + extension
3. Servlet by resource type + extension
4. Servlet by resource type supertype (walks up hierarchy)
5. Servlet by sling:resourceType default (sling/servlet/default)
6. Path-bound servlet (/bin/...)
```

```java
// More specific wins:
@SlingServletResourceTypes(
    resourceTypes = "myproject/components/page",
    selectors = "navigation",
    extensions = "json",
    methods = "GET"
)
// Wins over:
@SlingServletResourceTypes(
    resourceTypes = "myproject/components/page",
    extensions = "json",
    methods = "GET"
)
```

---

### 8. Service User Mapping for Servlets

```json
// org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended-myproject.cfg.json
{
    "user.mapping": [
        "com.myproject.core:servlet-service=myproject-servlet-user"
    ]
}
```

```java
// In servlet — use service user for operations beyond request user's permissions
@Reference
private ResourceResolverFactory resolverFactory;

private static final Map<String, Object> AUTH = Map.of(
    ResourceResolverFactory.SUBSERVICE, "servlet-service"
);

@Override
protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
        throws ServletException, IOException {
    // Request resolver — use for reading content the current user can see
    ResourceResolver userResolver = request.getResourceResolver();

    // Service resolver — use for system operations (audit logs, etc.)
    try (ResourceResolver serviceResolver = resolverFactory.getServiceResourceResolver(AUTH)) {
        // Write audit entry with system permissions
    } catch (LoginException e) {
        log.error("Service user login failed", e);
    }
}
```

---

### 9. Anti-Patterns

#### Path Servlets Without ACL Configuration

```java
// WRONG — accessible to anonymous users if Dispatcher allows /bin/
@SlingServletPaths("/bin/myproject/admin/deleteAll")
public class DangerousServlet extends SlingAllMethodsServlet { }

// CORRECT — always configure ACL for path servlets:
// 1. Restrict in Dispatcher filters
// 2. Add rep:policy ACL on the path
// 3. Check permissions programmatically
```

#### No Input Validation

```java
// WRONG — direct use of user input
String path = request.getParameter("path");
Resource r = resolver.getResource(path); // path traversal vulnerability

// CORRECT — validate and constrain
String path = request.getParameter("path");
if (path == null || !path.startsWith("/content/mysite/")) {
    response.setStatus(400);
    return;
}
// Also normalize to prevent ../  traversal
path = ResourceUtil.normalize(path);
if (path == null || !path.startsWith("/content/mysite/")) {
    response.setStatus(400);
    return;
}
```

#### Returning HTML from API Endpoints

```java
// WRONG — mixing concerns
response.getWriter().write("<html><body>Result: " + result + "</body></html>");

// CORRECT — return structured data, let frontend render
response.setContentType("application/json");
response.getWriter().write("{\"result\":\"" + result + "\"}");
```

#### Not Closing Resources

```java
// WRONG — resource resolver leak
ResourceResolver resolver = resolverFactory.getServiceResourceResolver(AUTH);
// forgot to close → memory leak, session leak

// CORRECT — always try-with-resources
try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(AUTH)) {
    // use resolver
}
```

#### Overly Broad Servlet Binding

```java
// WRONG — catches ALL GET requests for all pages
@SlingServletResourceTypes(
    resourceTypes = "cq/Page",
    methods = "GET",
    extensions = "json"
)

// CORRECT — use selectors to narrow scope
@SlingServletResourceTypes(
    resourceTypes = "myproject/components/page",
    selectors = "myapi",
    methods = "GET",
    extensions = "json"
)
```
