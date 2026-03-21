---
title: Third-Party Backend Integrations in AEM
impact: CRITICAL
impactDescription: Poorly integrated external APIs cause cascading failures, slow page rendering, security vulnerabilities, and author frustration
tags: integration, rest-api, http-client, circuit-breaker, caching, osgi, service-user, retry, timeout, error-handling
---

## Third-Party Backend Integrations in AEM

Production patterns for integrating AEM with external REST APIs, SOAP services, and backend systems — proper HTTP clients, timeouts, circuit breakers, caching, error handling, and OSGi configuration.

---

### 1. HTTP Client Setup (Apache HttpClient 5)

AEM as a Cloud Service bundles Apache HttpClient. Never instantiate clients per request — use a pooled, OSGi-managed singleton.

**Correct — OSGi-managed pooled client:**

```java
@Component(service = HttpClientFactory.class, immediate = true)
@Designate(ocd = HttpClientFactory.Config.class)
public class HttpClientFactory {

    @ObjectClassDefinition(name = "HTTP Client Factory")
    public @interface Config {
        @AttributeDefinition(name = "Connection timeout (ms)")
        int connectionTimeout() default 3000;

        @AttributeDefinition(name = "Socket timeout (ms)")
        int socketTimeout() default 5000;

        @AttributeDefinition(name = "Max connections per route")
        int maxPerRoute() default 20;

        @AttributeDefinition(name = "Max total connections")
        int maxTotal() default 100;
    }

    private CloseableHttpClient httpClient;

    @Activate
    protected void activate(Config config) {
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectTimeout(Timeout.ofMilliseconds(config.connectionTimeout()))
            .setResponseTimeout(Timeout.ofMilliseconds(config.socketTimeout()))
            .build();

        PoolingHttpClientConnectionManager connManager =
            new PoolingHttpClientConnectionManager();
        connManager.setDefaultMaxPerRoute(config.maxPerRoute());
        connManager.setMaxTotal(config.maxTotal());

        httpClient = HttpClients.custom()
            .setConnectionManager(connManager)
            .setDefaultRequestConfig(requestConfig)
            .setRetryStrategy(new DefaultHttpRequestRetryStrategy(
                2,                                    // max retries
                TimeValue.ofMilliseconds(500)         // retry interval
            ))
            .build();
    }

    @Deactivate
    protected void deactivate() {
        IOUtils.closeQuietly(httpClient);
    }

    public CloseableHttpClient getClient() {
        return httpClient;
    }
}
```

**Incorrect — client per request (leaks connections):**

```java
// DON'T DO THIS — creates and leaks a client every call
public String fetchData(String url) {
    CloseableHttpClient client = HttpClients.createDefault(); // leaked!
    HttpGet request = new HttpGet(url);
    return client.execute(request, response -> EntityUtils.toString(response.getEntity()));
}
```

---

### 2. Integration Service Pattern

Encapsulate each external system behind an OSGi service with configurable endpoints:

```java
@Component(service = ProductApiService.class)
@Designate(ocd = ProductApiService.Config.class)
@Slf4j
public class ProductApiService {

    @ObjectClassDefinition(name = "Product API Configuration")
    public @interface Config {
        @AttributeDefinition(name = "API Base URL")
        String apiBaseUrl() default "https://api.example.com/v2";

        @AttributeDefinition(name = "API Key", type = AttributeType.PASSWORD)
        String apiKey();

        @AttributeDefinition(name = "Cache TTL (seconds)")
        int cacheTtl() default 300;

        @AttributeDefinition(name = "Enable circuit breaker")
        boolean circuitBreakerEnabled() default true;
    }

    @Reference
    private HttpClientFactory httpClientFactory;

    private Config config;

    @Activate
    protected void activate(Config config) {
        this.config = config;
    }

    public Optional<Product> getProduct(String sku) {
        String url = config.apiBaseUrl() + "/products/" + sku;

        HttpGet request = new HttpGet(url);
        request.setHeader("Authorization", "Bearer " + config.apiKey());
        request.setHeader("Accept", "application/json");

        try {
            return httpClientFactory.getClient().execute(request, response -> {
                int status = response.getCode();
                if (status == 404) {
                    return Optional.empty();
                }
                if (status >= 400) {
                    log.warn("Product API error: {} for SKU {}", status, sku);
                    return Optional.empty();
                }
                String body = EntityUtils.toString(response.getEntity());
                return Optional.of(objectMapper.readValue(body, Product.class));
            });
        } catch (IOException e) {
            log.error("Product API connection failed for SKU {}", sku, e);
            return Optional.empty();
        }
    }

    public List<Product> searchProducts(String query, int limit) {
        URIBuilder builder;
        try {
            builder = new URIBuilder(config.apiBaseUrl() + "/products/search")
                .addParameter("q", query)
                .addParameter("limit", String.valueOf(limit));
        } catch (URISyntaxException e) {
            log.error("Invalid URI", e);
            return Collections.emptyList();
        }

        HttpGet request = new HttpGet(builder.toString());
        request.setHeader("Authorization", "Bearer " + config.apiKey());
        request.setHeader("Accept", "application/json");

        try {
            return httpClientFactory.getClient().execute(request, response -> {
                if (response.getCode() >= 400) {
                    log.warn("Product search failed: {}", response.getCode());
                    return Collections.emptyList();
                }
                String body = EntityUtils.toString(response.getEntity());
                return objectMapper.readValue(body,
                    objectMapper.getTypeFactory()
                        .constructCollectionType(List.class, Product.class));
            });
        } catch (IOException e) {
            log.error("Product search connection failed", e);
            return Collections.emptyList();
        }
    }
}
```

---

### 3. Circuit Breaker Pattern

Prevent cascading failures when an external API is down:

```java
@Component(service = CircuitBreaker.class)
public class CircuitBreaker {

    private enum State { CLOSED, OPEN, HALF_OPEN }

    private volatile State state = State.CLOSED;
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private volatile long lastFailureTime = 0;

    private static final int FAILURE_THRESHOLD = 5;
    private static final long RESET_TIMEOUT_MS = 30_000; // 30 seconds

    /**
     * Execute a supplier with circuit breaker protection.
     * Returns fallback value when circuit is open.
     */
    public <T> T execute(Supplier<T> action, T fallback) {
        if (state == State.OPEN) {
            if (System.currentTimeMillis() - lastFailureTime > RESET_TIMEOUT_MS) {
                state = State.HALF_OPEN;
            } else {
                log.warn("Circuit breaker OPEN — returning fallback");
                return fallback;
            }
        }

        try {
            T result = action.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            log.warn("Circuit breaker caught failure ({}/{})",
                failureCount.get(), FAILURE_THRESHOLD, e);
            return fallback;
        }
    }

    private void onSuccess() {
        failureCount.set(0);
        state = State.CLOSED;
    }

    private void onFailure() {
        lastFailureTime = System.currentTimeMillis();
        if (failureCount.incrementAndGet() >= FAILURE_THRESHOLD) {
            state = State.OPEN;
            log.error("Circuit breaker OPENED after {} failures", FAILURE_THRESHOLD);
        }
    }
}

// Usage in service:
@Reference
private CircuitBreaker circuitBreaker;

public Optional<Product> getProduct(String sku) {
    return circuitBreaker.execute(
        () -> fetchProductFromApi(sku),    // primary action
        Optional.empty()                    // fallback when circuit is open
    );
}
```

---

### 4. Response Caching

Cache external API responses to reduce latency and protect against outages:

```java
@Component(service = ApiCacheService.class)
@Designate(ocd = ApiCacheService.Config.class)
public class ApiCacheService {

    @ObjectClassDefinition(name = "API Response Cache")
    public @interface Config {
        @AttributeDefinition(name = "Max cache entries")
        int maxSize() default 1000;

        @AttributeDefinition(name = "Default TTL (seconds)")
        int defaultTtl() default 300;
    }

    private final ConcurrentHashMap<String, CacheEntry<?>> cache = new ConcurrentHashMap<>();
    private Config config;

    @Activate
    protected void activate(Config config) {
        this.config = config;
    }

    @SuppressWarnings("unchecked")
    public <T> T getOrFetch(String key, Supplier<T> fetcher) {
        return getOrFetch(key, fetcher, config.defaultTtl());
    }

    @SuppressWarnings("unchecked")
    public <T> T getOrFetch(String key, Supplier<T> fetcher, int ttlSeconds) {
        CacheEntry<?> entry = cache.get(key);
        if (entry != null && !entry.isExpired()) {
            return (T) entry.value;
        }

        T value = fetcher.get();
        if (value != null) {
            // Evict if over capacity
            if (cache.size() >= config.maxSize()) {
                evictExpired();
            }
            cache.put(key, new CacheEntry<>(value, ttlSeconds));
        }
        return value;
    }

    public void invalidate(String key) {
        cache.remove(key);
    }

    public void invalidateAll() {
        cache.clear();
    }

    private void evictExpired() {
        cache.entrySet().removeIf(e -> e.getValue().isExpired());
    }

    private static class CacheEntry<T> {
        final T value;
        final long expiresAt;

        CacheEntry(T value, int ttlSeconds) {
            this.value = value;
            this.expiresAt = System.currentTimeMillis() + (ttlSeconds * 1000L);
        }

        boolean isExpired() {
            return System.currentTimeMillis() > expiresAt;
        }
    }
}

// Usage:
@Reference
private ApiCacheService apiCache;

public Optional<Product> getProduct(String sku) {
    return apiCache.getOrFetch(
        "product:" + sku,
        () -> fetchProductFromApi(sku),
        300  // 5 minutes
    );
}
```

---

### 5. Environment-Specific Configuration

Use OSGi run-mode configs for different API endpoints per environment:

```
config/
├── com.mysite.services.ProductApiService.cfg.json              # default (local SDK)
├── config.dev/
│   └── com.mysite.services.ProductApiService.cfg.json          # dev
├── config.stage/
│   └── com.mysite.services.ProductApiService.cfg.json          # stage
└── config.prod/
    └── com.mysite.services.ProductApiService.cfg.json          # production
```

```json
// config.prod/com.mysite.services.ProductApiService.cfg.json
{
  "apiBaseUrl": "https://api.production.example.com/v2",
  "apiKey": "$[secret:PRODUCT_API_KEY]",
  "cacheTtl:Integer": 600,
  "circuitBreakerEnabled:Boolean": true
}
```

**Use Cloud Manager secrets** for API keys — never commit credentials:

```json
// Reference a Cloud Manager secret variable
{
  "apiKey": "$[secret:PRODUCT_API_KEY]"
}
```

---

### 6. Sling Model Integration

Expose external API data to HTL through Sling Models:

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = ProductModel.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
@Getter
public class ProductModel {

    @ValueMapValue
    private String productSku;

    @OSGiService
    private ProductApiService productApi;

    @OSGiService
    private ApiCacheService apiCache;

    private Product product;
    private boolean apiError;

    @PostConstruct
    protected void init() {
        if (StringUtils.isBlank(productSku)) return;

        Optional<Product> result = apiCache.getOrFetch(
            "product:" + productSku,
            () -> productApi.getProduct(productSku),
            300
        );

        if (result.isPresent()) {
            product = result.get();
        } else {
            apiError = true;
        }
    }

    public String getTitle() {
        return product != null ? product.getTitle() : "";
    }

    public String getPrice() {
        return product != null
            ? NumberFormat.getCurrencyInstance().format(product.getPrice())
            : "";
    }

    public boolean isEmpty() {
        return product == null;
    }

    public boolean isApiError() {
        return apiError;
    }
}
```

**HTL:**

```html
<sly data-sly-use.model="com.mysite.models.ProductModel" />
<div class="cmp-product" data-sly-test="${!model.empty}">
  <h2 class="cmp-product__title">${model.title}</h2>
  <p class="cmp-product__price">${model.price}</p>
</div>
<div class="cmp-product__error" data-sly-test="${model.apiError}">
  <p>Product information is temporarily unavailable.</p>
</div>
```

---

### 7. Async Patterns for Non-Blocking Calls

For pages with multiple API calls, use async to avoid sequential blocking:

```java
@Model(adaptables = SlingHttpServletRequest.class)
@Getter
public class DashboardModel {

    @OSGiService
    private ProductApiService productApi;

    @OSGiService
    private ReviewApiService reviewApi;

    @OSGiService
    private InventoryApiService inventoryApi;

    private Product product;
    private List<Review> reviews;
    private InventoryStatus inventory;

    @PostConstruct
    protected void init() {
        String sku = resource.getValueMap().get("productSku", "");
        if (StringUtils.isBlank(sku)) return;

        ExecutorService executor = Executors.newFixedThreadPool(3);
        try {
            // Fire all API calls in parallel
            CompletableFuture<Optional<Product>> productFuture =
                CompletableFuture.supplyAsync(() -> productApi.getProduct(sku), executor);
            CompletableFuture<List<Review>> reviewsFuture =
                CompletableFuture.supplyAsync(() -> reviewApi.getReviews(sku), executor);
            CompletableFuture<InventoryStatus> inventoryFuture =
                CompletableFuture.supplyAsync(() -> inventoryApi.getStatus(sku), executor);

            // Wait for all with timeout
            CompletableFuture.allOf(productFuture, reviewsFuture, inventoryFuture)
                .get(5, TimeUnit.SECONDS);

            product = productFuture.get().orElse(null);
            reviews = reviewsFuture.get();
            inventory = inventoryFuture.get();
        } catch (TimeoutException e) {
            log.warn("Dashboard API calls timed out for SKU {}", sku);
        } catch (Exception e) {
            log.error("Dashboard API error for SKU {}", sku, e);
        } finally {
            executor.shutdown();
        }
    }
}
```

---

### 8. Webhook / Callback Handling

Receive webhooks from external systems via Sling Servlets:

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "mysite/components/webhook-receiver",
    methods = "POST",
    extensions = "json"
)
@Slf4j
public class WebhookReceiverServlet extends SlingAllMethodsServlet {

    @Reference
    private ProductApiService productApi;

    @Override
    protected void doPost(SlingHttpServletRequest request,
                          SlingHttpServletResponse response) throws IOException {

        // Verify webhook signature
        String signature = request.getHeader("X-Webhook-Signature");
        String body = IOUtils.toString(request.getReader());

        if (!verifySignature(body, signature)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return;
        }

        // Parse and process
        JsonObject payload = JsonParser.parseString(body).getAsJsonObject();
        String event = payload.get("event").getAsString();

        switch (event) {
            case "product.updated":
                String sku = payload.get("sku").getAsString();
                // Invalidate cache so next request fetches fresh data
                apiCache.invalidate("product:" + sku);
                log.info("Product cache invalidated for SKU {}", sku);
                break;
            case "inventory.changed":
                apiCache.invalidateAll();
                break;
            default:
                log.debug("Unhandled webhook event: {}", event);
        }

        response.setStatus(HttpServletResponse.SC_OK);
        response.getWriter().write("{\"status\":\"ok\"}");
    }

    private boolean verifySignature(String body, String signature) {
        // HMAC-SHA256 verification against shared secret
        // Implementation depends on the external system
        return StringUtils.isNotBlank(signature);
    }
}
```

---

### 9. Error Handling Strategy

| HTTP Status | Action | Cache Behavior |
|-------------|--------|---------------|
| 200 | Return data | Cache for TTL |
| 304 | Return cached | Extend cache TTL |
| 400 | Log warning, return empty | Don't cache |
| 401/403 | Log error, check API key config | Don't cache |
| 404 | Return empty/Optional.empty() | Cache "not found" for short TTL |
| 429 | Back off, retry after delay | Use stale cache |
| 500-503 | Log error, return cached/fallback | Serve stale cache |
| Timeout | Log warning, return cached/fallback | Serve stale cache |

```java
// Stale-while-revalidate pattern
public <T> T getWithStaleFallback(String key, Supplier<T> fetcher, int ttl) {
    CacheEntry<?> entry = cache.get(key);

    // Fresh cache — return immediately
    if (entry != null && !entry.isExpired()) {
        return (T) entry.value;
    }

    // Try to fetch fresh data
    try {
        T fresh = fetcher.get();
        if (fresh != null) {
            cache.put(key, new CacheEntry<>(fresh, ttl));
            return fresh;
        }
    } catch (Exception e) {
        log.warn("Fetch failed for key {}, serving stale", key, e);
    }

    // Stale fallback — better than nothing
    if (entry != null) {
        log.info("Serving stale cache for key {}", key);
        return (T) entry.value;
    }

    return null;
}
```

---

### 10. Testing Integrations

```java
// Unit test with mocked HTTP responses
@ExtendWith(AemContextExtension.class)
class ProductApiServiceTest {

    private final AemContext context = new AemContext();
    private MockWebServer mockServer;
    private ProductApiService service;

    @BeforeEach
    void setUp() throws Exception {
        mockServer = new MockWebServer();
        mockServer.start();

        // Configure service with mock server URL
        Map<String, Object> config = Map.of(
            "apiBaseUrl", mockServer.url("/v2").toString(),
            "apiKey", "test-key",
            "cacheTtl", 60
        );
        service = context.registerInjectActivateService(ProductApiService.class, config);
    }

    @AfterEach
    void tearDown() throws Exception {
        mockServer.shutdown();
    }

    @Test
    void testGetProduct_success() {
        mockServer.enqueue(new MockResponse()
            .setBody("{\"sku\":\"ABC\",\"title\":\"Test Product\",\"price\":29.99}")
            .setHeader("Content-Type", "application/json"));

        Optional<Product> result = service.getProduct("ABC");

        assertTrue(result.isPresent());
        assertEquals("Test Product", result.get().getTitle());
    }

    @Test
    void testGetProduct_404_returnsEmpty() {
        mockServer.enqueue(new MockResponse().setResponseCode(404));

        Optional<Product> result = service.getProduct("NONEXISTENT");

        assertTrue(result.isEmpty());
    }

    @Test
    void testGetProduct_serverError_returnsEmpty() {
        mockServer.enqueue(new MockResponse().setResponseCode(500));

        Optional<Product> result = service.getProduct("ABC");

        assertTrue(result.isEmpty());
    }

    @Test
    void testGetProduct_timeout_returnsEmpty() {
        mockServer.enqueue(new MockResponse()
            .setBodyDelay(10, TimeUnit.SECONDS)); // exceeds socket timeout

        Optional<Product> result = service.getProduct("ABC");

        assertTrue(result.isEmpty());
    }
}
```

---

### Anti-Patterns

- **Creating HTTP clients per request** — leaks connections and sockets; use a pooled OSGi-managed client
- **No timeouts** — a slow API blocks AEM rendering threads; always set connection + socket timeouts
- **Hardcoded API URLs** — use OSGi configs with run-mode specific values
- **Committing API keys** — use Cloud Manager secret variables (`$[secret:KEY]`)
- **No circuit breaker** — when an API is down, every page request waits for timeout; circuit breaker fails fast
- **No caching** — every page render calls the external API; cache responses with TTL
- **Synchronous blocking in Sling Models** — for multiple API calls, use CompletableFuture with timeouts
- **Swallowing exceptions silently** — always log with enough context (URL, status code, SKU/ID) for debugging
- **No fallback UI** — when the API is down, show a degraded experience instead of a broken page
- **No retry strategy** — transient network errors should be retried (with exponential backoff)
- **Testing against real APIs** — use MockWebServer or WireMock for deterministic, fast tests
