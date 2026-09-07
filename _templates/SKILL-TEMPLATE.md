---
name: <skill-identifier>                   # e.g. angular-development
description: >
  One-paragraph description of what this skill does and when Claude should activate it.
  Be specific about trigger phrases, file types, and contexts.
  Example: "Use when building Angular applications. Triggers on: creating components,
  services, pipes, directives; writing RxJS streams; configuring NgRx state; writing
  tests for Angular code."
version: 1.0.0
technology: <angular | reactjs | nextjs | native-web | dotnet | python | design | other>
author: <name>
last_updated: <YYYY-MM-DD>
---

# <Skill Name>

> One-sentence summary of what this skill enforces and for whom.
> Example: "Opinionated conventions for building production Angular applications with
> TypeScript 5, RxJS 7, and NgRx."

---

## 1. When to Use

Apply this skill when:

- User asks to create, scaffold, or refactor `<X>` files or features
- User mentions `<keyword>`, `<framework concept>`, or `<file pattern>`
- The project contains `<config file>` or `<dependency>`
- Example trigger phrases: `"create a service"`, `"add a unit test"`, `"scaffold a module"`

**Do NOT use when:**

- The project uses `<related but different technology>` instead
- The user is asking about `<out-of-scope concern>`
- A more specific skill (e.g. `<skill-name>`) is already active

---

## 2. Project Structure

```
<project-root>/
├── <config-file>              ← <purpose>
├── <config-file>              ← <purpose>
└── src/
    ├── <folder>/              ← <purpose>
    │   ├── <subfolder>/       ← <purpose>
    │   └── <subfolder>/       ← <purpose>
    └── <folder>/              ← <purpose>
```

Rules:
- `<folder naming rule>`
- `<file placement rule>`
- `<module / barrel file rule>`

---

## 3. Toolchain & Configuration

> List the tools, lock files, and key config snippets that must be present.
> Keep config examples minimal — just the fields that matter for this skill.

**Required tools:** `<tool-1>`, `<tool-2>`, `<linter>`, `<formatter>`

```<config-format>
# <config-file-name>
# Key fields only — omit boilerplate

<key>: <value>    # <reason>
<key>: <value>    # <reason>
```

---

## 4. <Core Topic A — e.g. Components, Controllers, Models>

> Replace this section header with the first major technical topic for the stack.
> Add as many core topic sections as needed (typically 4–8).
> Each section: rules as a bullet list + one short representative code snippet.
> Full patterns and complete examples belong in the references/ files.

Rules:
- `<rule>`
- `<rule>`
- `<rule>`

```<language>
// Short representative snippet — 10–25 lines maximum.
```

---

## 5. <Core Topic B — e.g. Services, Data Layer, State>

Rules:
- `<rule>`
- `<rule>`

```<language>
// Short representative snippet
```

---

## 6. <Core Topic C — e.g. Error Handling, Validation>

Rules:
- `<rule>`
- `<rule>`

```<language>
// Short representative snippet
```

---

## 7. <Core Topic D — e.g. Testing>

> If this skill has a dedicated references/testing.md, keep this section to
> 3–5 bullet rules and point there for the full strategy.

Rules:
- `<rule>`
- `<rule>`

---

## 8. <Core Topic E — e.g. Performance>

Rules:
- `<rule>`
- `<rule>`

---

## 9. Naming Conventions

| Construct | Rule | Example |
|---|---|---|
| `<construct>` | `<rule>` | `<example>` |
| `<construct>` | `<rule>` | `<example>` |
| `<construct>` | `<rule>` | `<example>` |
| `<construct>` | `<rule>` | `<example>` |
| `<construct>` | `<rule>` | `<example>` |

---

## 10. Code Quality

- **Linter:** `<tool>` — run with `<command>`. Zero warnings policy.
- **Formatter:** `<tool>` — run with `<command>`. Enforced in CI.
- **Type checking:** `<tool>` — run with `<command>`. Strict mode on.
- Never disable rules inline without a comment explaining why.
- `<any other stack-specific quality rule>`

---

## Customizing

> This section tells Claude which reference file to read for each topic.
> Claude reads SKILL.md first; it opens a reference file only when it needs
> deeper detail on that specific topic. Update the table as files are added.

| Topic | File | When to read |
|---|---|---|
| `<topic>` | `references/<filename>.md` | When the user asks about `<trigger>` |
| `<topic>` | `references/<filename>.md` | When the user asks about `<trigger>` |
| `<topic>` | `references/<filename>.md` | When the user asks about `<trigger>` |
| `<topic>` | `references/<filename>.md` | When the user asks about `<trigger>` |
| Full examples | `references/examples.md` | When producing a complete feature or needing a production-ready pattern |

### Project Overrides

> This is a different mechanism from the table above: the table tells Claude
> which of *this skill's own* files to open. This section instead gives a
> consuming project a place to record its own deviations from the skill's
> defaults, without forking or editing the skill itself.
>
> When a project adopts this skill, its team can paste a block like the one
> below into their own project docs (or a project-local `references/project-overrides.md`
> they create themselves — this skill does not ship one) to declare where they
> intentionally diverge. Claude should treat overrides recorded this way as
> taking precedence over the corresponding rule in this SKILL.md.

```markdown
## Project Overrides — [Project Name]

- `<rule this project changes>`: `<the project's actual choice>` — not the skill's default
- `<rule this project changes>`: `<the project's actual choice>` — not the skill's default
- `<rule this project changes>`: `<the project's actual choice>` — not the skill's default
```
