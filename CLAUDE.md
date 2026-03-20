# AEM Skills — Claude Code Project

## Overview

This repository contains two Claude Code skills for Adobe Experience Manager (AEM) development:

- **`aemaacs/`** — `aem-best-practices` — 52 rule files for full-stack AEM as a Cloud Service
- **`aem-eds/`** — `aem-eds-frontend-best-practices` — 17 rule files for AEM Edge Delivery Services

## Project Structure

```
aem-skills/
├── CLAUDE.md              # This file
├── README.md              # Project overview, file tree, installation
├── aemaacs/
│   ├── SKILL.md           # Skill manifest (name, description, tools, rules list)
│   └── rules/             # 52 rule files organized by category
│       ├── build-pipeline/        # 9 files — Webpack, Cloud Manager, RDE, CTT, testing
│       ├── component-development/ # 10 files — Core Components, dialogs, DAM, UE, Style System
│       ├── java/                  # 5 files — Sling Models, servlets, OSGi, workflows
│       ├── analytics-tracking/    # 2 files — ACDL, personalization
│       ├── touch-ui/              # 12 files — Coral UI, dialogs, RTE, security, ACLs
│       ├── layout/                # 1 file — Responsive grid
│       ├── headless/              # 6 files — GraphQL, CFs, XFs, SDKs, Commerce/CIF
│       ├── multi-tenant/          # 1 file — MSM, i18n
│       └── performance/           # 6 files — CDN, Dispatcher, Dynamic Media, queries
└── aem-eds/
    ├── SKILL.md           # Skill manifest
    └── rules/             # 17 rule files organized by category
        ├── block-development/     # 11 files — blocks, JS, CSS, testing, SW, WC, edge, 3rd-party
        ├── authoring/             # 3 files — Universal Editor, standard blocks, Sidekick/SEO
        ├── multi-tenant/          # 1 file — Repoless, theming
        └── performance/           # 2 files — RUM/TTFB, CDN config
```

## Rule File Format

Every rule file follows this structure:

```markdown
---
title: Rule Title
impact: HIGH | MEDIUM | CRITICAL
impactDescription: One sentence explaining why this matters
tags: comma, separated, tags
---

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
- **YAML frontmatter**: Every rule file has `title`, `impact`, `impactDescription`, `tags`

## When Editing Rules

- Keep the existing frontmatter format — do not change field names
- Maintain the `impact` rating scale: `CRITICAL` > `HIGH` > `MEDIUM` > `LOW`
- Include real, working code examples (Java, HTL, XML, JS, CSS as appropriate)
- Show both correct and incorrect patterns for every major concept
- End every file with an Anti-Patterns section
- Use tables for reference material (field types, operators, comparison matrices)

## When Adding New Rules

1. Create the `.md` file in the appropriate `rules/` subdirectory
2. Add the rule entry to the parent `SKILL.md` file
3. Update `README.md` to reflect the new file in the tree and counts
4. Follow the same frontmatter and section structure as existing files

## Installation (for users)

```bash
# Symlink into Claude Code skills directory
ln -s /path/to/aem-skills/aemaacs ~/.claude/skills/aemaacs-frontend-best-practices
ln -s /path/to/aem-skills/aem-eds ~/.claude/skills/aem-eds-frontend-best-practices
```

## Sources

All content is based on official Adobe documentation:
- experienceleague.adobe.com
- aem.live
- developer.adobe.com
- Adobe GitHub repositories
