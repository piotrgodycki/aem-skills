## Third-Party Dependency Maintenance

Long-lived AEM projects accumulate third-party dependencies (Maven for `core`, npm for `ui.frontend`) and external integrations (CRM, search, payment, analytics). This rule covers keeping them current and secure without destabilising the build. Spec-driven (`spec-driven-development.md`); treat security upgrades as their own PRs.

### 1. Inventory and Classify

- Maven: `mvn dependency:tree` — separate **provided** (AEM-supplied, do NOT bundle) from **compile** (yours to manage/embed).
- npm: `npm ls --all` / lockfile — distinguish build-time (dev) from shipped runtime code.
- Flag libraries that duplicate what AEM already provides (don't bundle a second JSON/HTTP/logging stack).

**Incorrect — bundling an AEM-provided API:** embedding `org.apache.sling.api` or a second SLF4J into your bundle causes classloader conflicts. Keep AEM-provided deps `<scope>provided</scope>`.

### 2. CVE Triage

**Correct — scan and prioritise:**
```bash
# npm
npm audit --omit=dev            # runtime vulns only
# Maven (OWASP dependency-check or similar)
mvn org.owasp:dependency-check-maven:check
```
- Prioritise by reachability + severity: a CVE in a transitively-included but unused code path is lower risk than one on your request hot path.
- Patch/minor bumps for security fixes go in a dedicated PR with the CVE id in the description.
- For `provided` AEM dependencies, the fix is an AEM Service Pack (6.5) or the rolling platform (AEMaaCS) — you cannot patch them from your build.

### 3. Upgrade Discipline

- One dependency (or one security batch) per PR; never mix with features.
- Respect the OSGi version ranges in `Import-Package`; a major bump can break the bundle wiring even if Maven resolves it (`core-platform-maintenance.md`).
- After an npm bump, dedupe and re-check bundle size (`frontend-maintenance.md`).
- Read the changelog for breaking changes; pin versions (no floating ranges in production).

### 4. External Integration Drift

Integrations break from the *other* side changing (API deprecations, cert rotation, rate-limit changes), not your code.

- Centralise each integration behind a service with timeouts, retries, and a circuit breaker (`java/third-party-integrations.md`); drift then degrades gracefully instead of taking down request threads.
- Keep endpoint URLs, versions, and credentials in OSGi config per run mode — never hard-coded (warn and extract if you find hard-coded secrets).
- Monitor the integration's deprecation notices; schedule migrations before the provider's sunset date, not after.
- Verify against a sandbox/stub (`MockWebServer`) in tests so an upgrade can't silently change the contract.

### 5. Verify Upgrades

- Full `mvn clean verify` + quality gate; front-end build + bundle-size compare.
- Smoke-test every feature touching the upgraded lib / integration.
- Re-run the vulnerability scan to confirm the CVE is actually resolved (not just moved).

### AEM 6.5 / Classic differences
- `provided` platform libraries are fixed by the **Service Pack / CFP**; schedule SP upgrades to pick up security fixes.
- No Adobe-managed platform patching — on-prem security posture is entirely your responsibility.
- Secrets for integrations use Crypto Support, not a Cloud secrets service.

### Anti-Patterns
- Bundling AEM-provided dependencies (second logging/HTTP/JSON stack) → classloader conflicts.
- Mixing dependency upgrades with feature changes in one PR.
- Floating version ranges in production builds.
- Hard-coded integration URLs/credentials instead of run-mode OSGi config.
- Calling external APIs with no timeout/circuit breaker, so provider drift cascades into request-thread exhaustion.
- Closing a CVE ticket without re-scanning to confirm the fix.
