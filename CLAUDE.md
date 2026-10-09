# AEM Skills — Claude Code Project

## Overview

This repository contains two Claude Code skills for Adobe Experience Manager (AEM) development:

- **`aem-best-practices/`** — 66 rule files for full-stack AEM. AEMaaCS-first, with **AEM 6.5 / Classic** differences annotated. Includes Vanilla JS / Alpine / Tailwind frontend and a `maintenance/` workflow (spec-driven development) for existing repos.
- **`aem-eds-frontend-best-practices/`** — 18 rule files for AEM Edge Delivery Services

## Project Structure

```
aem-skills/
├── CLAUDE.md              # This file
├── README.md              # Project overview, file tree, installation
├── aem-best-practices/
│   ├── SKILL.md           # Lean router: name + description + reference map
│   └── rules/             # 66 rule files organized by category
│       ├── build-pipeline/        # 9 files — Webpack, Cloud Manager, RDE, CTT, testing
│       ├── component-development/ # 10 files — Core Components, dialogs, DAM, UE, Style System
│       ├── java/                  # 6 files — Sling Models, Lombok, servlets, OSGi, workflows, 3rd-party
│       ├── analytics-tracking/    # 2 files — ACDL, personalization
│       ├── touch-ui/              # 12 files — Coral UI, dialogs, RTE, security, ACLs
│       ├── layout/                # 1 file — Responsive grid
│       ├── headless/              # 6 files — GraphQL, CFs, XFs, SDKs, Commerce/CIF
│       ├── multi-tenant/          # 1 file — MSM, i18n
│       ├── performance/           # 6 files — CDN, Dispatcher, Dynamic Media, queries
│       ├── frontend/              # 7 files — AEM-native FE (SCSS, Tailwind, Alpine, fe-aem-server, Vanilla JS/Web Components)
│       │   └── vanilla/           # 3 files — Web Components, modules, events
│       └── maintenance/           # 6 files — spec-driven dev, audit/quality, core/fe/backend/3rd-party upkeep
└── aem-eds-frontend-best-practices/
    ├── SKILL.md           # Skill manifest
    └── rules/             # 18 rule files organized by category
        ├── block-development/     # 11 files — blocks, JS, CSS, testing, SW, WC, edge, 3rd-party
        ├── authoring/             # 4 files — Content-Driven Dev, Universal Editor, standard blocks, Sidekick/SEO
        ├── multi-tenant/          # 1 file — Repoless, theming
        └── performance/           # 2 files — RUM/TTFB, CDN config
```

## Skill design

Both skills use a lean-router structure:

- **`SKILL.md` is a lean router.** Frontmatter is only `name` + `description`. The description is a context pointer — front-loaded, "Use when…", with distinct trigger branches and a one-line disambiguation against the sibling skill. The body is a short overview plus a **reference map**: each rule file listed with a one-line "reach when…" pointer, grouped by task.
- **Rule files are disclosed reference** — no frontmatter. Progressive disclosure: the agent opens only the one or two files the task touches, never all 84 at once. The `##` heading is the title.
- Write positively (state the target, don't ban the anti-pattern), keep one source of truth per fact, and cut no-ops the model already obeys by default.

## Rule File Format

Every rule file follows this structure (no YAML frontmatter):

```markdown
## Rule Title

Introductory paragraph.

### Numbered Sections with Code Examples

**Correct — description:**
(code block)

**Incorrect — description:**
(code block)

### Anti-Patterns
- Bulleted list of what NOT to do
```

## Key Conventions

- **Tone**: Senior expert AEM developer — production-grade patterns, not tutorials
- **Code examples**: Always show both correct and incorrect patterns
- **Anti-patterns section**: Every rule file ends with common mistakes
- **No frameworks in EDS**: Vanilla JS + modern CSS only for Edge Delivery Services
- **Coral 3 only**: Never reference Coral 2 resource types in AEMaaCS rules
- **No frontmatter on rule files**: the `##` heading is the title; metadata lives in the SKILL.md reference map

## When Editing Rules

- Include real, working code examples (Java, HTL, XML, JS, CSS as appropriate)
- Show both correct and incorrect patterns for every major concept
- End every file with an Anti-Patterns section
- Use tables for reference material (field types, operators, comparison matrices)

## When Adding New Rules

1. Create the `.md` file in the appropriate `rules/` subdirectory (no frontmatter, start with `## Title`)
2. Add a one-line "reach when…" pointer under the right group in the parent `SKILL.md` reference map
3. Update `README.md` to reflect the new file in the tree and counts

## Installation (for users)

```bash
# Via skills.sh
npx skills add <your-github-username>/aem-skills
```

## Sources

All content is based on official Adobe documentation:
- experienceleague.adobe.com
- aem.live
- developer.adobe.com
- Adobe GitHub repositories
