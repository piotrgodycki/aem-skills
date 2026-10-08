# AGENTS.md

Entry point for coding agents. This repo ships two Claude Code skills of AEM best-practice rules. Full project conventions live in **[CLAUDE.md](./CLAUDE.md)** — read it before editing.

## Pick the right skill

| You are working on… | Use |
|---|---|
| Full-stack AEM — `ui.frontend`/Webpack, ClientLibs, HTL, Granite dialogs, Sling Models, OSGi, Dispatcher, headless | **`aem-best-practices/`** |
| Edge Delivery Services — `/blocks/`, vanilla JS/CSS, XWalk boilerplate, Universal Editor, Document Authoring | **`aem-eds-frontend-best-practices/`** |

Each skill's `SKILL.md` is its manifest and the index of every rule file — start there.

## Deployment target matters

`aem-best-practices` is **AEMaaCS-first**. Where behaviour differs on **AEM 6.5 / Classic (on-prem/AMS)**, the rule carries an `### AEM 6.5 / Classic differences` section — check it before applying Cloud-only patterns (RDE, Asset Microservices, Fastly/`cdn.yaml`, auto-scaling). Confirm the target first (signals in `maintenance/codebase-audit-quality.md`).

## Changing an existing repo? Go spec-driven

For maintenance/brownfield work, follow **`aem-best-practices/rules/maintenance/spec-driven-development.md`**: AUDIT → SPEC → CONFIRM → IMPLEMENT → VERIFY → REVIEW. Write a short, testable spec and confirm scope before coding.

## Working in a legacy / brownfield repo

Most real AEM work is on an existing, often aging codebase. Before changing anything:

1. **Audit first, don't assume.** Run `maintenance/codebase-audit-quality.md` — map the module layout, `filter.xml` scope, conventions, and the quality baseline. Never edit a component before reading its `.content.xml`, dialog, and Sling Model.
2. **Detect the era.** Signals of a legacy stack and what they imply:
   - `dispatcher.any` + `uber-jar` + `cq-quickstart` → **AEM 6.5 / on-prem** (apply the `### AEM 6.5 / Classic differences` guidance).
   - JSP components, `<cq:include>`, Foundation components (`foundation/components/...`), Coral 2 dialogs (`cq:dialog` with `xtype`) → legacy authoring; migration candidates, not patterns to copy.
   - Classic UI (`/libs/cq/ui`), ExtJS widgets, `design_dialog` → pre-Touch-UI; flag, don't extend.
3. **Match what's there, then improve at the edges.** New code reads like the surrounding code (naming, structure, style) even if you'd personally do it differently. A stylistically "correct" change that ignores repo conventions is a defect.
4. **Don't modernise uninvited.** JSP→HTL, Coral 2→3, Foundation→Core Components, 6.5→Cloud are *migrations* with their own blast radius — propose them as scoped specs (`migration-patterns.md`), never smuggle them into an unrelated change.
5. **Record debt, don't fix it all now.** Note deprecated APIs, traversal queries, vulnerable deps in a tech-debt map; let the spec owner decide what becomes work. Hand dependency/CVE work to `third-party-maintenance.md`.
6. **Smallest reversible change.** Prefer proxy + delegation + BEM/Style-System layering over editing shared or Core Components. Keep the change behind stable contracts (JSON keys, getters, markup hooks) — see `backend-maintenance.md` / `frontend-maintenance.md`.
7. **Verify broadly.** Legacy code has hidden coupling: run the quality gate, smoke-test a representative page set, visual-diff key templates, and watch the error log for new OSGi `Unsatisfied`/`ClassNotFound` after deploy.

> Rule of thumb: in a legacy repo, understanding and a confirmed spec are cheap; a wrong change to a shared component or a Dispatcher filter is not.

## Editing these skills

- Follow the rule-file contract and the add/edit checklists in **[CLAUDE.md](./CLAUDE.md)** (frontmatter, correct/incorrect examples, Anti-Patterns, impact ratings).
- Keep counts in sync across `SKILL.md`, `README.md`, and `CLAUDE.md` when adding/removing rules.
- Client-data & git rules: never commit secrets; feature branch + PR; don't push without explicit approval.
