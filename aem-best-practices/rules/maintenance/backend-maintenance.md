## Backend Maintenance

Maintaining the `core` Java layer of an existing AEM project: refactoring Sling Models and services, retiring deprecated APIs, and fixing performance regressions — without changing behaviour the author/front-end relies on. Always spec-driven (`spec-driven-development.md`) with unit tests as acceptance criteria.

### 1. Refactor Behind Stable Contracts

The Sling Model's exported JSON and getter surface is a contract for HTL and headless consumers. Refactor internals freely; change the contract only via the spec.

**Correct — internal refactor, contract unchanged:**
```java
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
@Getter @Slf4j
public class TeaserModel {
    @ValueMapValue private String title;   // same JSON key, same getter
    // internal: extracted a private helper; no public surface change
}
```

**Incorrect — silent contract break:** renaming a getter or `@JsonProperty` key during a "cleanup" breaks HTL bindings and headless clients with no compile error.

### 2. Retire Deprecated APIs Safely

Common deprecations to migrate during maintenance:

| Deprecated | Replacement |
|---|---|
| `ResourceResolverFactory.getAdministrativeResourceResolver` | Service user + `getServiceResourceResolver` (mapping in `ui.config`) |
| JCR admin session | `ResourceResolver` + service user |
| `@Reference` on fields (Felix SCR) | OSGi DS R7 annotations (`org.osgi.service.component.annotations`) |
| `SlingServlet` (Felix) | `@Component(service=Servlet.class)` + `@SlingServletResourceTypes` |
| Synchronous external calls in request thread | Pooled HttpClient + timeout + circuit breaker (`java/third-party-integrations.md`) |

Migrate one deprecation per PR; keep the behaviour identical and covered by a test.

### 3. Resource & Session Hygiene

- Always close `ResourceResolver`/session opened in code (`try-with-resources`); leaks surface as slow degradation, not an immediate crash.
- Never open admin resolvers; wire a least-privilege service user.
- Bound every external call with connect/read timeouts; a hanging integration takes down request threads.

### 4. Fix Performance Regressions by Measurement

- Reproduce with data: request timing, thread dumps for stuck requests, `QueryStat`/slow-query log (`performance/query-optimization.md`).
- Eliminate N+1 resource resolution and per-request repeated queries; cache with explicit TTL where safe.
- Confirm the fix against the metric that regressed, not a proxy.

### 5. Tests Are the Acceptance Criteria

- AEM Mocks (`io.wcm.testing` / `aem-mock`) for Sling Models and servlets; assert both the refactored path and edge cases.
- A maintenance change must not drop coverage; add a test that would have caught the original bug.
- Run the full module suite + quality gate before the PR.

### AEM 6.5 / Classic differences
- OSGi DS annotations, Sling Models, servlets and AEM Mocks are identical (`java/osgi-services-schedulers.md` 6.5 notes).
- Secrets via Crypto Support, not Cloud Manager secrets.
- `uber-jar` must match the installed Service Pack (`core-platform-maintenance.md`).
- Felix console (`/system/console`) is available to inspect services/components live.

### Anti-Patterns
- Renaming getters / JSON keys during "cleanup" without treating it as a contract change.
- Migrating multiple unrelated deprecations in one PR.
- Opening admin resolvers/sessions or leaving resolvers unclosed.
- External calls without timeouts on the request thread.
- Fixing a "slow" endpoint by guessing instead of measuring.
- Refactoring with no test that would have caught the original defect.
