---
name: <skill-identifier>
description: >
  One-paragraph description of what this skill does and when Claude should trigger it.
  Be specific about trigger phrases, file types, and contexts. Err on the side of being
  "pushy" — list concrete scenarios so the skill activates when it should.
  Example triggers: "when the user asks to create an Angular component", "when a .spec.ts
  file is involved", "when the user mentions state management in React".
version: 1.0.0
technology: <angular | reactjs | nextjs | dotnet | java | python | design | other>
author: <name or team>
reviewed_by: <reviewer name — filled after approval>
last_updated: <YYYY-MM-DD>
compatibility: <any required tools, CLIs, or dependencies — e.g. "Angular CLI >= 17, Node >= 20">
---

# <Skill Name>

## Overview

Brief description of what this skill helps Claude do and why it exists. Two to four sentences.
Explain the problem it solves and the value it delivers to the developer.

## When to Use

Clearly describe the scenarios, prompts, and contexts where this skill should activate.
Include example trigger phrases so Claude knows when to apply it.

- User asks to generate, scaffold, or refactor `<X>` files
- User mentions `<keyword>` or `<framework concept>`
- Files matching `<glob pattern>` are open or referenced
- Example phrases: "create a service", "add a unit test", "scaffold a module"

## When NOT to Use

Scenarios where this skill should NOT trigger, to prevent false activations.

- This skill does not apply to `<related but different technology>`
- Do not use when the user is asking about `<out-of-scope concern>`
- Do not combine with `<conflicting skill>` at the same time

## Instructions

> Core instructions Claude should follow when this skill is active.
> Write in imperative form. Be precise, opinionated, and practical.
> This section is the heart of the skill.

### Code Style & Conventions

- Describe naming conventions, file structure, import ordering, etc.
- List any linting rules or formatter settings that must be respected

### Architecture & Patterns

- Describe the preferred patterns for this technology (e.g. smart/dumb components, repository pattern)
- Explain any required abstractions or layering

### Dos and Don'ts

**Do:**
- Always do `<X>`
- Prefer `<approach A>` over `<approach B>` because `<reason>`

**Don't:**
- Never do `<Y>` — it causes `<problem>`
- Avoid `<anti-pattern>` — use `<alternative>` instead

### Error Handling & Edge Cases

- Describe how errors should be handled in this context
- Note any known gotchas or platform-specific quirks

## Examples

Provide at least two concrete input/output examples showing the skill in action.

---

**Example 1: `<Short title>`**

_Input (what the user asks):_
```
<paste a realistic user prompt here>
```

_Expected output (what Claude should produce):_
```<language>
// Paste the ideal code or artifact Claude should generate
```

---

**Example 2: `<Short title>`**

_Input:_
```
<paste a realistic user prompt here>
```

_Expected output:_
```<language>
// Paste the ideal code or artifact Claude should generate
```

---

## References

List any external docs, internal wiki pages, or files in the `references/` subfolder that
support this skill.

- [Official docs](<URL>)
- `references/<filename>` — description of what this file contains
