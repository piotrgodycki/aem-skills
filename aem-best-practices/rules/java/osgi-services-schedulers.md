---
title: OSGi Services, Schedulers, and Event Handlers
impact: HIGH
impactDescription: OSGi services form the backbone of AEM backend logic — incorrect patterns cause memory leaks, clustered job failures, and resource resolver leaks that crash instances
tags: osgi, services, schedulers, sling-jobs, event-handlers, resource-change-listener, service-user, health-checks, configuration
---

## OSGi Services, Schedulers, and Event Handlers

OSGi Declarative Services (DS) annotations are the standard for AEM backend development. They manage service lifecycle, dependency injection, configuration, and scheduling. AEM as a Cloud Service requires DS annotations (R7+), not legacy Felix SCR annotations.

---

### 1. OSGi Service Fundamentals

#### DS Annotations (@Component)

```java
@Component(
    service = ProductService.class,          // Interface(s) this component registers as
    immediate = true,                         // Activate immediately (not lazily)
    configurationPolicy = ConfigurationPolicy.REQUIRE  // Require OSGi config to activate
)
@Designate(ocd = ProductServiceImpl.Config.class)
public class ProductServiceImpl implements ProductService {

    private static final Logger log = LoggerFactory.getLogger(ProductServiceImpl.class);

    @ObjectClassDefinition(
        name = "Product Service Configuration",
        description = "Configuration for the product catalog service"
    )
    @interface Config {
        @AttributeDefinition(name = "API Endpoint", description = "Product catalog API URL")
        String api_endpoint() default "https://api.example.com/products";

        @AttributeDefinition(name = "Cache TTL (seconds)")
        int cache_ttl() default 300;

        @AttributeDefinition(name = "Enabled")
        boolean enabled() default true;
    }

    private Config config;

    @Activate
    @Modified
    protected void activate(Config config) {
        this.config = config;
        log.info("Product service configured: endpoint={}, ttl={}",
            config.api_endpoint(), config.cache_ttl());
    }

    @Deactivate
    protected void deactivate() {
        log.info("Product service deactivated");
    }
}
```

**Incorrect — legacy Felix annotations (do not use):**

```java
// WRONG — deprecated Felix SCR annotations
@org.apache.felix.scr.annotations.Component
@org.apache.felix.scr.annotations.Service
@org.apache.felix.scr.annotations.Properties({
    @org.apache.felix.scr.annotations.Property(name = "api.endpoint", value = "...")
})
public class ProductServiceImpl implements ProductService { }
```

---

### 2. @Reference Injection

#### Static Reference (Default, Preferred)

```java
@Component(service = ContentService.class)
public class ContentServiceImpl implements ContentService {

    @Reference
    private ResourceResolverFactory resolverFactory;

    @Reference
    private QueryBuilder queryBuilder;

    @Reference(target = "(component.name=com.myproject.core.impl.PrimarySearchProvider)")
    private SearchProvider searchProvider; // target specific implementation
}
```

#### Reference Cardinality and Policy

```java
// Optional reference — service activates even if reference unavailable
@Reference(cardinality = ReferenceCardinality.OPTIONAL)
private volatile AnalyticsService analyticsService;

// Multiple references — collect all implementations
@Reference(cardinality = ReferenceCardinality.MULTIPLE, policy = ReferencePolicy.DYNAMIC)
private volatile List<ContentValidator> validators;

// Use volatile for DYNAMIC policy references (may change at runtime)
```

| Cardinality | Meaning |
|------------|---------|
| `MANDATORY` (default) | Must be available; service won't activate without it |
| `OPTIONAL` | May be null; service activates regardless |
| `MULTIPLE` | Collects all matching services |
| `AT_LEAST_ONE` | At least one must be available |

#### Service Ranking

```java
// Higher ranking = preferred when multiple implementations exist
@Component(
    service = SearchProvider.class,
    property = { "service.ranking:Integer=100" }
)
public class PrimarySearchProvider implements SearchProvider { }

@Component(
    service = SearchProvider.class,
    property = { "service.ranking:Integer=50" }
)
public class FallbackSearchProvider implements SearchProvider { }

// Consumer gets PrimarySearchProvider (ranking 100 > 50)
@Reference
private SearchProvider searchProvider;
```

---

### 3. OSGi Configuration Patterns

#### Run-Mode Specific Configuration

```
config/
├── com.myproject.core.impl.ProductServiceImpl.cfg.json          # all environments
├── config.author/
│   └── com.myproject.core.impl.ProductServiceImpl.cfg.json      # author only
├── config.publish/
│   └── com.myproject.core.impl.ProductServiceImpl.cfg.json      # publish only
├── config.dev/
│   └── com.myproject.core.impl.ProductServiceImpl.cfg.json      # dev environment
├── config.stage/
│   └── com.myproject.core.impl.ProductServiceImpl.cfg.json      # stage environment
└── config.prod/
    └── com.myproject.core.impl.ProductServiceImpl.cfg.json      # production
```

#### Configuration File Format

```json
// com.myproject.core.impl.ProductServiceImpl.cfg.json
{
    "api.endpoint": "https://api.example.com/products",
    "cache.ttl:Integer": 300,
    "enabled:Boolean": true,
    "allowed.paths:String[]": [
        "/content/mysite",
        "/content/dam/mysite"
    ]
}
```

#### Factory Configurations

```java
@Component(
    service = EndpointConfig.class,
    configurationPolicy = ConfigurationPolicy.REQUIRE
)
@Designate(ocd = EndpointConfigImpl.Config.class, factory = true) // factory = true
public class EndpointConfigImpl implements EndpointConfig {

    @ObjectClassDefinition(name = "API Endpoint Configuration")
    @interface Config {
        @AttributeDefinition(name = "Endpoint Name")
        String name();

        @AttributeDefinition(name = "URL")
        String url();

        @AttributeDefinition(name = "Timeout (ms)")
        int timeout() default 5000;
    }
}
```

```json
// com.myproject.core.impl.EndpointConfigImpl~products.cfg.json
{ "name": "products", "url": "https://api.example.com/products", "timeout:Integer": 5000 }

// com.myproject.core.impl.EndpointConfigImpl~users.cfg.json
{ "name": "users", "url": "https://api.example.com/users", "timeout:Integer": 3000 }
```

---

### 4. Sling Schedulers

#### Simple Scheduled Task

```java
@Component(
    service = Runnable.class,
    property = {
        "scheduler.expression=0 0 2 * * ?",      // Cron: daily at 2 AM
        "scheduler.concurrent:Boolean=false",      // Prevent overlapping runs
        "scheduler.runOn=LEADER"                   // Only on cluster leader (Cloud Service)
    }
)
@Designate(ocd = CacheCleanupTask.Config.class)
public class CacheCleanupTask implements Runnable {

    private static final Logger log = LoggerFactory.getLogger(CacheCleanupTask.class);

    @ObjectClassDefinition(name = "Cache Cleanup Scheduler")
    @interface Config {
        @AttributeDefinition(name = "Cron Expression")
        String scheduler_expression() default "0 0 2 * * ?";

        @AttributeDefinition(name = "Enabled")
        boolean enabled() default true;
    }

    private boolean enabled;

    @Activate
    @Modified
    protected void activate(Config config) {
        this.enabled = config.enabled();
    }

    @Override
    public void run() {
        if (!enabled) return;
        log.info("Starting cache cleanup");
        // cleanup logic
    }
}
```

#### Scheduler API (Programmatic Scheduling)

```java
@Component(service = DynamicSchedulerService.class)
public class DynamicSchedulerService {

    @Reference
    private Scheduler scheduler;

    public void scheduleTask(String jobName, Runnable task, int intervalMinutes) {
        ScheduleOptions options = scheduler.EXPR("0 */" + intervalMinutes + " * * * ?");
        options.name(jobName);
        options.canRunConcurrently(false);
        scheduler.schedule(task, options);
    }

    public void unscheduleTask(String jobName) {
        scheduler.unschedule(jobName);
    }
}
```

---

### 5. Sling Jobs (Preferred for Clustered Environments)

Sling Jobs guarantee execution exactly once across a cluster, with persistence and retry. **Always prefer Sling Jobs over OSGi Scheduler for work that must complete reliably.**

#### Job Consumer

```java
@Component(
    service = JobConsumer.class,
    property = {
        JobConsumer.PROPERTY_TOPICS + "=myproject/email/send"
    }
)
public class SendEmailJobConsumer implements JobConsumer {

    private static final Logger log = LoggerFactory.getLogger(SendEmailJobConsumer.class);

    @Reference
    private EmailService emailService;

    @Override
    public JobResult process(Job job) {
        String recipient = job.getProperty("recipient", String.class);
        String templatePath = job.getProperty("templatePath", String.class);

        try {
            emailService.send(recipient, templatePath);
            log.info("Email sent to: {}", recipient);
            return JobResult.OK;
        } catch (TemporaryException e) {
            log.warn("Temporary failure, will retry: {}", e.getMessage());
            return JobResult.FAILED; // Sling will retry
        } catch (PermanentException e) {
            log.error("Permanent failure, canceling job", e);
            return JobResult.CANCEL; // No retry
        }
    }
}
```

#### Creating Jobs

```java
@Reference
private JobManager jobManager;

public void queueEmail(String recipient, String templatePath) {
    Map<String, Object> props = new HashMap<>();
    props.put("recipient", recipient);
    props.put("templatePath", templatePath);

    Job job = jobManager.addJob("myproject/email/send", props);
    if (job != null) {
        log.info("Email job queued: {}", job.getId());
    }
}
```

#### Sling Jobs vs OSGi Scheduler

| Feature | OSGi Scheduler | Sling Jobs |
|---------|---------------|------------|
| Cluster-safe | Only with `scheduler.runOn=LEADER` | **Yes, built-in** |
| Guaranteed execution | No (missed if instance down) | **Yes, persisted** |
| Retry on failure | No | **Yes, configurable** |
| Monitoring | Limited | **Job queue dashboard** |
| Use case | Periodic tasks | Reliable work processing |

---

### 6. Event Handling

#### ResourceChangeListener (Preferred)

```java
@Component(
    service = ResourceChangeListener.class,
    property = {
        ResourceChangeListener.PATHS + "=/content/mysite",
        ResourceChangeListener.CHANGES + "=ADDED",
        ResourceChangeListener.CHANGES + "=CHANGED",
        ResourceChangeListener.CHANGES + "=REMOVED"
    }
)
public class ContentChangeListener implements ResourceChangeListener {

    private static final Logger log = LoggerFactory.getLogger(ContentChangeListener.class);

    @Reference
    private JobManager jobManager;

    @Override
    public void onChange(List<ResourceChange> changes) {
        for (ResourceChange change : changes) {
            log.debug("Resource {}: {} (type: {})", change.getType(),
                change.getPath(), change.isExternal() ? "external" : "local");

            // Queue work — never do heavy processing in the listener
            if (change.getPath().endsWith("/jcr:content")) {
                Map<String, Object> props = Map.of("path", change.getPath());
                jobManager.addJob("myproject/content/index", props);
            }
        }
    }
}
```

#### OSGi EventAdmin

```java
// Listening for OSGi events
@Component(
    service = EventHandler.class,
    property = {
        EventConstants.EVENT_TOPIC + "=com/day/cq/replication/job/publish",
        EventConstants.EVENT_FILTER + "=(path=/content/mysite/*)"
    }
)
public class ReplicationEventHandler implements EventHandler {

    private static final Logger log = LoggerFactory.getLogger(ReplicationEventHandler.class);

    @Override
    public void handleEvent(Event event) {
        String path = (String) event.getProperty("path");
        log.info("Content published: {}", path);
        // Queue invalidation or notification — keep lightweight
    }
}
```

---

### 7. Service User Mapping

**Never use `getAdministrativeResourceResolver()` — it is removed in Cloud Service.**

#### Configuration

```json
// org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended-myproject.cfg.json
{
    "user.mapping": [
        "com.myproject.core:content-reader=myproject-content-reader",
        "com.myproject.core:content-writer=myproject-content-writer",
        "com.myproject.core:workflow-service=myproject-workflow-user"
    ]
}
```

#### Repo Init for Service User

```
// org.apache.sling.jcr.repoinit.RepositoryInitializer-myproject.cfg.json
{
    "scripts": [
        "create service user myproject-content-reader\nset ACL for myproject-content-reader\n    allow jcr:read on /content/mysite\n    allow jcr:read on /content/dam/mysite\nend",
        "create service user myproject-content-writer\nset ACL for myproject-content-writer\n    allow jcr:read,jcr:modifyProperties,jcr:addChildNodes on /content/mysite\nend"
    ]
}
```

#### Usage Pattern

```java
private static final Map<String, Object> READER_AUTH = Map.of(
    ResourceResolverFactory.SUBSERVICE, "content-reader"
);

public List<String> getPublishedPages() {
    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(READER_AUTH)) {
        // Read-only operations with minimal permissions
        return findPages(resolver);
    } catch (LoginException e) {
        log.error("Service user login failed — check serviceusermapping config", e);
        return Collections.emptyList();
    }
    // ResourceResolver is auto-closed by try-with-resources
}
```

---

### 8. Health Checks

```java
@Component(
    service = HealthCheck.class,
    property = {
        HealthCheck.NAME + "=Product API Health",
        HealthCheck.MBEAN_NAME + "=productApiHealth",
        HealthCheck.TAGS + "=integrations",
        HealthCheck.TAGS + "=deep"
    }
)
public class ProductApiHealthCheck implements HealthCheck {

    @Reference
    private ProductService productService;

    @Override
    public Result execute() {
        FormattingResultLog log = new FormattingResultLog();

        try {
            boolean reachable = productService.ping();
            if (reachable) {
                log.info("Product API is reachable");
            } else {
                log.warn("Product API returned unhealthy status");
            }
        } catch (Exception e) {
            log.critical("Product API is unreachable: {}", e.getMessage());
        }

        return new Result(log);
    }
}
```

---

### 9. Anti-Patterns

#### Deprecated Admin Session

```java
// WRONG — removed in Cloud Service
ResourceResolver resolver = resolverFactory.getAdministrativeResourceResolver(null);
Session session = slingRepository.loginAdministrative(null);

// CORRECT — always use service users
try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(AUTH)) {
    // ...
}
```

#### Not Closing ResourceResolvers

```java
// WRONG — resolver leak crashes the instance
public void doWork() {
    ResourceResolver resolver = resolverFactory.getServiceResourceResolver(AUTH);
    // ... use resolver
    // forgot to close — JCR session leak!
}

// CORRECT — try-with-resources
public void doWork() {
    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(AUTH)) {
        // ... use resolver
    } // auto-closed
}
```

#### Heavy Processing in Event Listeners

```java
// WRONG — blocks the event thread, causes backlog
@Override
public void onChange(List<ResourceChange> changes) {
    for (ResourceChange change : changes) {
        callExternalAPI(change.getPath());  // Slow HTTP call
        reindexContent(change.getPath());   // Heavy operation
        sendNotification(change.getPath()); // More I/O
    }
}

// CORRECT — queue a Sling Job for heavy work
@Override
public void onChange(List<ResourceChange> changes) {
    for (ResourceChange change : changes) {
        jobManager.addJob("myproject/content/process", Map.of("path", change.getPath()));
    }
}
```

#### Scheduler Without Cluster Safety

```java
// WRONG — runs on every instance in the cluster
@Component(
    service = Runnable.class,
    property = { "scheduler.expression=0 */5 * * * ?" }
)
public class ImportTask implements Runnable {
    // Runs 3x if 3 instances — duplicate imports!
}

// CORRECT — run only on leader
@Component(
    service = Runnable.class,
    property = {
        "scheduler.expression=0 */5 * * * ?",
        "scheduler.runOn=LEADER"
    }
)
public class ImportTask implements Runnable { }

// BEST — use Sling Jobs for guaranteed single execution
```

#### Mutable Static State in OSGi

```java
// WRONG — shared mutable state across component lifecycle
public class MyService {
    private static final Map<String, Object> cache = new HashMap<>(); // Shared across restarts!
}

// CORRECT — use instance fields, cleared on deactivate
@Deactivate
protected void deactivate() {
    this.cache.clear();
}
```
