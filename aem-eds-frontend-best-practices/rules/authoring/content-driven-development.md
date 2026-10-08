---
title: Content-Driven Development & Reuse-First Workflow
impact: HIGH
impactDescription: Code-first EDS development builds against imagined requirements, produces author-hostile blocks, and duplicates blocks that already exist in the Block Collection.
tags: eds, cdd, content-first, block-collection, block-party, workflow, authoring, reuse
---

## Content-Driven Development & Reuse-First Workflow

Adobe's official EDS guidance is **Content-Driven Development (CDD)**: content reality drives code, not the reverse. This rule captures the philosophy and the reuse-first loop that should precede writing any block.

### 1. The Loop

```
FIND CONTENT → CHECK FOR EXISTING BLOCK → MODEL → BUILD → TEST WITH REAL CONTENT → REVIEW
     │                  │                    │       │              │
     │                  │                    │       │              └─ PSI/Lighthouse on a real page URL
     │                  │                    │       └─ decorate() against the real DOM
     │                  │                    └─ pick a canonical model (eds-content-modeling.md)
     │                  └─ Block Collection first, then Block Party — don't rebuild what exists
     └─ find or author real test content BEFORE coding (drafts/ folder)
```

### 2. Author Needs Come First

Authors are the primary users of the structures you create. Content models and block structures must be:
- **Intuitive** — authors understand what goes where without training.
- **Forgiving** — common mistakes are easy to spot and fix.
- **Flexible** — room for creativity within structure.

This often means *more complex decoration code*. That trade-off is correct: developer convenience is secondary to author experience.

### 3. Test Content Before Code

Create or find real content *before* writing the block. It is a multiplier, not extra work:
- Test as you write — catch issues immediately, iterate faster.
- The same content becomes the PSI/Lighthouse validation URL required on every PR.
- Well-structured test content doubles as living author documentation.

Use a `drafts/` folder (excluded from sitemap/indexing) for isolated block testing.

**Incorrect — code-first:** write the block against assumptions → discover the assumptions were wrong → rewrite → scramble to create content at PR time. Feels fast, is slow.

### 4. Reuse First: Block Collection, then Block Party

Before building a custom block, check whether a vetted one exists.

| Source | What it is | When |
|---|---|---|
| **Block Collection** | Adobe-maintained, vetted for performance/a11y/content-modeling | **Prefer this.** Start here for common blocks (cards, hero, columns, tabs, carousel…) |
| **Block Party** | Community repository — broader variety, plugins, integrations | Specialised needs not covered by the Collection |

- Block Collection: <https://github.com/adobe/aem-block-collection> · <https://www.aem.live/developer/block-collection>
- Copy and adapt a reference block rather than starting from scratch; keep its content model and performance characteristics.
- For official how-to/documentation questions, consult the aem.live docs (the Adobe `docs-search` skill queries them directly) — don't guess EDS APIs from memory.

**Incorrect — rebuilding Cards from scratch** when the Block Collection Cards block already handles the content model, lazy images, and accessibility you need.

### 5. Content as a Contract

The initial content structure is a contract between authors and developers: authors promise to structure content a certain way, developers promise that structure renders correctly. Changing a shipped block's expected table structure breaks every existing page that uses it — treat the model as a stable interface and version variants instead of mutating the contract.

### Anti-Patterns
- Writing block JS/CSS before any real content exists.
- Building a custom block without checking the Block Collection / Block Party first.
- Optimising the model for developer convenience at the expense of author clarity.
- Changing a live block's expected content structure (breaks authored pages).
- Guessing EDS APIs instead of checking aem.live docs.
- Creating test content only at PR time, so no validation happened during development.
