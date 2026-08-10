# Skills Audit Report
**Date:** 2026-08-10  
**Total Skills Audited:** 15  
**Auditor:** Claude (Skill Developer Mode)

---

## Summary

| Rating | Skills |
|--------|--------|
| ✅ Strong | skill-creator, react-development, design-system-development*, pptx, setup-writing-style, learn |
| 🟡 Good | docx, xlsx, pdf, morning, schedule, setup-cowork |
| 🔴 Needs Work | explain-usage, theme-factory, consolidate-memory |

*Has a critical broken-references issue (see below).

---

## Skill-by-Skill Overview

| Skill | Lines | Trigger Guidance | Examples | Guardrails | Status |
|-------|------:|:---:|:---:|:---:|--------|
| skill-creator | 485 | ✅ | ✅ | ✅ | Strong |
| react-development | 578 | ✅ | ✅ | ✅ | Strong |
| design-system-development | 568 | ✅ | ✅ | ✅ | Strong (broken refs) |
| pptx | 238 | ✅ | ✅ | ✅ | Strong |
| setup-writing-style | 227 | ✅ | ✅ | ✅ | Strong |
| learn | 78 | ✅ | ✅ | ✅ | Strong |
| morning | 136 | ❌ | ✅ | ✅ | Good |
| pdf | 314 | ❌ | ✅ | ✅ | Good |
| setup-cowork | 97 | ❌ | ✅ | ✅ | Good |
| docx | 91 | ✅ | ❌ | ✅ | Good |
| xlsx | 99 | ✅ | ✅ | ✅ | Good |
| schedule | 40 | ✅ | ✅ | ✅ | Good |
| consolidate-memory | 34 | ❌ | ✅ | ❌ | Needs Work |
| theme-factory | 59 | ❌ | ❌ | ❌ | Needs Work |
| explain-usage | 11 | ✅ | ❌ | ❌ | Needs Work |

---

## Critical Issue

### 🔴 `design-system-development` — 5 Missing Reference Files

The SKILL.md references these files which **do not exist** on disk:

```
references/tokens.md
references/components.md
references/accessibility.md
references/figma.md
references/examples.md
```

Claude will silently skip them at runtime, meaning the skill runs without its full context. This degrades output quality without any visible error.

**Fix:** Either create the missing files or remove the `references:` block from the frontmatter.

---

## Improvements Needed

### 1. `explain-usage` — Critically Thin (11 lines)

Only 11 lines with no body structure, no examples, and no guardrails. Claude has almost no guidance on *how* to explain usage — just that it should.

**Recommended additions:**
- What format to use (chart type, chart library, structure)
- A "do not" clause (e.g., don't expose raw token counts without context)
- At least one example output pattern

---

### 2. `theme-factory` — Missing Triggers and Examples

No trigger keywords and no concrete examples. Claude may not know when to invoke it proactively, and users get no reference for what a good themed output looks like.

**Recommended additions:**
- A `## When to Use` section with trigger phrases (e.g., "apply a theme", "make this look nicer", "style this")
- 1–2 before/after examples showing a themed artifact

---

### 3. `consolidate-memory` — No Trigger Guidance or Guardrails

The skill has a clear 3-phase workflow but nothing tells Claude *when* to invoke it or what to avoid (e.g., don't delete memories that are still active, don't consolidate if the memory directory is empty).

**Recommended additions:**
- Trigger conditions (e.g., "memory index > 25KB", "user says consolidate", "after a long session")
- A guardrail: don't run if fewer than 5 memory files exist

---

### 4. `morning`, `pdf`, `setup-cowork` — Missing Explicit Trigger Sections

These skills rely on the description field alone for triggering. Adding an in-body `## When to Use` section improves reliability, especially for edge-case phrasing.

**Affected skills:** `morning`, `pdf`, `setup-cowork`

---

### 5. `docx` — No Concrete Examples

The description is thorough but the body has no example showing what a generated document call looks like. A short snippet reduces Claude hallucinating incorrect `docx-js` API calls.

---

## What's Working Well

- **skill-creator** is the gold standard — rich trigger guidance, eval framework, blind comparison mode, and Cowork-specific instructions.
- **react-development** and **design-system-development** are comprehensive reference skills with good "When to Use / When NOT to Use" sections.
- **pptx** and **xlsx** have strong guardrails and QA steps built in.
- **setup-writing-style** has a well-sequenced multi-step workflow with clear consent and calibration phases.
- **learn** has the best "Don't trigger for" section — very precise anti-pattern list.

---

## Priority Fix List

| Priority | Skill | Action |
|----------|-------|--------|
| 🔴 High | design-system-development | Create or remove the 5 missing reference files |
| 🔴 High | explain-usage | Add structure, examples, guardrails |
| 🟡 Medium | theme-factory | Add trigger section + examples |
| 🟡 Medium | consolidate-memory | Add trigger guidance + guardrails |
| 🟢 Low | morning, pdf, setup-cowork | Add explicit "When to Use" section |
| 🟢 Low | docx | Add 1–2 code examples |
