# AI Playbook — Organizational Skill Repository

A centralized library of reusable **SKILL.md** files that teams across the organization can download and integrate into their Claude-powered projects. Each skill encapsulates best practices, architectural patterns, coding standards, and workflows for a specific technology or domain.

---

## Quick Start

1. Browse the catalog below to find a skill relevant to your work.
2. Copy the skill's folder (containing `SKILL.md` and any `references/`) into your project's `skills/` directory.
3. Claude will automatically pick up and apply the skill when working in that project.
4. To contribute a new skill, start from `_templates/SKILL-TEMPLATE.md` and open a PR targeting `_review/`.

---

## Repository Structure

```
ai-playbook/
├── README.md                  ← You are here
├── _templates/
│   └── SKILL-TEMPLATE.md      ← Start here when authoring a new skill
├── _review/                   ← Skills pending review before publishing
│
├── angular/                   ← Angular skills
├── reactjs/                   ← React skills
├── nextjs/                    ← Next.js skills
├── native-web/                ← Vanilla HTML/CSS/JS skills
├── dotnet/                    ← .NET / C# skills
├── python/                    ← Python skills
└── design/                    ← Design system & tooling skills
```

Technology folders use lowercase kebab-case. Each skill lives in its own subfolder named `<technology>-<purpose>` (e.g. `angular/angular-development/`). New top-level folders are created when the first skill for a new technology is contributed.

---

## Skill Catalog

### Front-end

| Skill | Path | Stack |
|---|---|---|
| Angular | `angular/angular-development/` | Angular 17+, TypeScript 5, RxJS 7, NgRx, Angular Material |
| React | `reactjs/react-development/` | React 18, TypeScript 5, TanStack Query, Zustand, Vitest |
| Next.js | `nextjs/nextjs-development/` | Next.js 14 (App Router), TypeScript 5, Server Actions, Prisma |
| Native Web | `native-web/native-web-development/` | Vanilla HTML5, CSS3 Custom Properties, ES2022+, Web Components |

### Back-end

| Skill | Path | Stack |
|---|---|---|
| .NET Web API | `dotnet/dotnet-webapi-development/` | ASP.NET Core, .NET 8 LTS, EF Core 8, xUnit, Moq |
| Python | `python/python-development/` | Python 3.11+, Typer, Pydantic v2, asyncio, Ruff, mypy |

### Design

| Skill | Path | Covers |
|---|---|---|
| Design System | `design/design-system-development/` | Design tokens (three-tier), component API contracts, WCAG 2.1 AA, Figma |

---

## Skill Layout

Every skill follows the same two-layer structure:

```
<technology>/<skill-name>/
├── SKILL.md          ← Core skill: conventions, rules, quick examples, and
│                        pointers to reference files. Load this into Claude.
└── references/
    ├── <topic-1>.md  ← Deep reference loaded on demand during a session
    ├── <topic-2>.md
    └── ...
```

### What each layer contains

**`SKILL.md`** is the entry point. It covers when to use the skill, project structure, the most important conventions, short code snippets, naming rules, and a "Customizing" section that lists every reference file and when to read it.

**`references/`** contains full, production-ready code for topics that need more depth than `SKILL.md` can hold — complete patterns, worked examples, testing strategies, and integration guides.

### Reference files by skill

| Skill | Reference files |
|---|---|
| Angular | `type-system`, `http-layer`, `state-management`, `testing`, `examples` |
| React | `type-system`, `http-layer`, `state-management`, `testing`, `examples` |
| Next.js | `rendering`, `server-actions`, `auth`, `testing`, `examples` |
| Native Web | `html`, `css`, `javascript`, `web-components`, `examples` |
| .NET Web API | `controllers`, `data-layer`, `auth`, `testing`, `examples` |
| Python | `type-system`, `concurrency`, `file-operations`, `patterns`, `examples` |
| Design System | `tokens`, `components`, `accessibility`, `figma`, `examples` |

---

## Contributing a Skill

1. **Copy the template** — duplicate `_templates/SKILL-TEMPLATE.md` into the appropriate technology folder: `<technology>/<technology>-<purpose>/SKILL.md`.
2. **Fill it out** — follow the structure defined in the template. The `description` frontmatter field is the most important: it tells Claude when to activate the skill.
3. **Add references** — place supporting files (full code samples, patterns, test strategies) in a `references/` subfolder alongside `SKILL.md`.
4. **Open a PR** — place your skill folder under `_review/` and open a pull request. A peer reviewer will approve and move it to the correct technology folder.
5. **Update the catalog** — add a row to the table above once the skill is merged.

### Naming Conventions

- Technology folders: `angular`, `reactjs`, `nextjs`, `native-web`, `dotnet`, `python`, `design`
- Skill folders: `<technology>-<purpose>` in kebab-case (e.g. `dotnet-webapi-development`)
- Version: start at `1.0.0`; bump minor for non-breaking additions, major for rewrites

---

## Shared Conventions

These rules apply across every skill in this repo:

- **CSS** — Never use `!important`. Fix specificity with a more targeted selector. The only accepted exception is the `prefers-reduced-motion` browser reset.
- **TypeScript** — Strict mode always on. No `any`, no untyped function parameters. Use `unknown` + type guard instead.
- **Python** — `from __future__ import annotations` in every file. Use `X | Y` union syntax. No `Optional[X]` — use `X | None`.
- **Accessibility** — All components target WCAG 2.1 AA minimum. Keyboard navigation, focus management, and screen reader patterns are non-negotiable.
- **Testing** — Each back-end and front-end skill documents its testing stack in `references/testing.md`. Write tests alongside the code, not after.

---

## Audience

| Role | Typical usage |
|---|---|
| **Developers** | Framework and library skills (Angular, React, Next.js, .NET, Python) |
| **Designers** | Design-system and tooling skills (tokens, component contracts, accessibility, Figma) |
| **Architects** | Any skill for architectural patterns, API design, and standards enforcement |
