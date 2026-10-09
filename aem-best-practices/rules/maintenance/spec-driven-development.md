## Spec-Driven Development for Existing AEM Repositories

Greenfield skills assume you are building. In practice most AEM work is **maintenance of an existing repo** — fixing a component, upgrading a dependency, extending a dialog. Spec-Driven Development (SDD) front-loads a short, verifiable specification before touching code, so the change is scoped, reviewable, and testable against explicit acceptance criteria.

This file is the **entry-point workflow**. The per-area rules (`codebase-audit-quality.md`, `core-platform-maintenance.md`, `frontend-maintenance.md`, `backend-maintenance.md`, `third-party-maintenance.md`) are the toolbox each phase draws on.

### 1. The Loop

```
AUDIT → SPEC → CONFIRM → IMPLEMENT → VERIFY → REVIEW
  │        │       │          │          │        │
  │        │       │          │          │        └─ diff vs spec + code-review skill
  │        │       │          │          └─ run acceptance criteria (tests, author UI, env)
  │        │       │          └─ smallest change that satisfies the spec, nothing more
  │        │       └─ user/PO signs off on scope + acceptance criteria BEFORE coding
  │        └─ write the spec (below) from what the audit found
  └─ map the affected modules, conventions, blast radius (codebase-audit-quality.md)
```

Never jump from a ticket straight to `IMPLEMENT`. The audit and spec are cheap; a wrong change to a shared Core Component proxy or a dispatcher filter is not.

### 2. The Spec Template

Keep it short — half a page. It is a contract, not a design doc.

**Correct — a spec for a maintenance change:**
```markdown
## Spec: Teaser CTA opens in new tab when linking externally

### Problem
Authors have no way to make the Teaser CTA open external links in a new tab.

### Scope (modules touched)
- ui.apps: /apps/myco/components/teaser/_cq_dialog (add checkbox)
- core: TeaserModel delegate (expose `ctaTarget`)
- ui.frontend: none (markup already renders `target`)

### Non-goals
- No change to Core Component version
- No change to existing authored content (default = current behaviour)

### Acceptance criteria
1. New "Open in new tab" checkbox appears in the Link tab, unchecked by default.
2. When checked, rendered CTA has `target="_blank" rel="noopener"`.
3. Existing teasers render unchanged (no `target`).
4. Sling Model unit test covers checked/unchecked.
5. No new quality-gate violations; dispatcher cache behaviour unchanged.

### Blast radius
Shared proxy used on 40+ pages — verify no layout/style regression.
```

**Incorrect — no spec, "just add a checkbox":** the dev modifies the Core Component directly instead of the proxy, defaults the value to `true` (changing 40 live pages), and ships with no test. Caught only in prod.

### 3. Acceptance Criteria Must Be Executable

Each criterion maps to a verification you can actually run:

| Criterion type | How it is verified |
|---|---|
| Rendering / markup | Sling Model unit test (AEM Mocks) or HTL output assertion |
| Author UX | Manual dialog check on author + screenshot; or UI test |
| No regression | Visual diff on representative pages; existing test suite green |
| Performance | Query/Dispatcher/RUM metric unchanged or improved |
| Quality | Cloud Manager quality gate / SonarQube delta = 0 new issues |

If a criterion cannot be verified, it is a wish, not a criterion — rewrite it.

### 4. Confirm Before Implementing

Surface the spec to the requester and get explicit sign-off on **scope + acceptance criteria + non-goals**. This is where scope creep dies. In AEM specifically, confirm the deployment target (AEMaaCS vs 6.5), because it changes what is even possible (see the `### AEM 6.5 / Classic differences` sections).

### 5. Implement the Smallest Change

- Touch only the modules listed in Scope. If you discover you need more, **stop and update the spec**, don't silently expand.
- Prefer proxy + delegation over editing shared/Core Components (`core-platform-maintenance.md`).
- Keep the change reviewable: one concern per PR.

### 6. Verify, Then Review

- Run every acceptance criterion. Report pass/fail honestly — a half-met criterion is a failed one.
- Run the repo's quality gates and the `code-review` skill (or Adobe `code-review`) before opening the PR.
- The PR description restates the spec so the reviewer checks code *against the agreed contract*, not against their own guess.

### Anti-Patterns
- Coding before the spec is written and confirmed.
- Acceptance criteria that cannot be executed ("make it better", "clean up the code").
- Expanding scope mid-implementation without updating the spec.
- Editing a shared Core Component / proxy without recording its blast radius in the spec.
- Treating the spec as a design essay instead of a short, testable contract.
- Writing the spec against AEMaaCS when the target is 6.5 (or vice-versa).
- Marking the task done when only some acceptance criteria pass.
