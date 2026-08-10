# AI Playbook — Claude Code Skill Repository

> A curated library of **SKILL.md** files that give [Claude Code](https://claude.ai/code) deep, project-specific knowledge of your tech stack — so every developer gets consistent, standards-aligned AI assistance without repeating instructions on every prompt.

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Skills](https://img.shields.io/badge/Skills-7-brightgreen)](#skill-catalog)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Compatible-orange)](https://claude.ai/code)

---

## Table of Contents

- [What is this?](#what-is-this)
- [Quick Start](#quick-start)
- [Repository Structure](#repository-structure)
- [Skill Catalog](#skill-catalog)
- [Skill Layout](#skill-layout)
- [Contributing a Skill](#contributing-a-skill)
- [Shared Conventions](#shared-conventions)
- [Audience](#audience)

---

## What is this?

**AI Playbook** is an organizational coding standards library built specifically for [Claude Code](https://claude.ai/code). It solves a common problem: as teams adopt AI coding assistants, each developer re-explains the same architectural rules, naming conventions, and stack preferences on every session.

This repo fixes that by shipping those rules as **SKILL.md** files — structured Markdown documents that Claude Code reads automatically when placed in a project's `skills/` directory. Drop in the Angular skill, and Claude instantly knows your component patterns, state management tiers, RxJS conventions, and accessibility requirements. No prompt engineering required.

**Skills currently available:** Angular 17+, React 19, Next.js 14+ (App Router), Native Web (Vanilla JS / Web Components), ASP.NET Core (.NET 8), Python 3.11+, and Design Systems.

---

## Quick Start

1. Browse the [Skill Catalog](#skill-catalog) and find the skill for your stack.
2. Copy the skill folder (containing `SKILL.md` and the `references/` subfolder) into your project's `skills/` directory.
3. Claude Code will automatically load and apply the skill when working in that project.
4. To contribute a new skill, copy `_templates/SKILL-TEMPLATE.md` and open a PR targeting `_review/`.

---

## Repository Structure

```
ai-playbook/
├── README.md                  ← You are here
├── CLAUDE.md                  ← Claude Code project context for this repo
├── _templates/
│   └── SKILL-TEMPLATE.md      ← Start here when authoring a new skill
├── _review/                   ← Skills pending peer review before publishing
│
├── angular/                   ← Angular 17+ skills
├── reactjs/                   ← React 19 skills
├── nextjs/                    ← Next.js 14+ (App Router) skills
├── native-web/                ← Vanilla HTML/CSS/JS / Web Components skills
├── dotnet/                    ← ASP.NET Core / .NET 8 skills
├── python/                    ← Python 3.11+ skills
└── design/                    ← Design system & token skills
```

Technology folders use lowercase kebab-case. Each skill lives in its own subfolder named `<technology>-<purpose>` (e.g. `angular/angular-development/`). New top-level folders are created when the first skill for a new technology is contributed.

---

## Skill Catalog

### Front-end

| Skill | Folder | Stack |
|---|---|---|
| Angular | `angular/angular-development/` | Angular 17+, TypeScript 5, RxJS 7, NgRx, Angular Material |
| React | `reactjs/react-development/` | React 19, TypeScript 5, TanStack Query, Zustand, Vitest |
| Next.js | `nextjs/nextjs-development/` | Next.js 14+ (App Router), TypeScript 5, Server Actions, Prisma, Auth.js |
| Native Web | `native-web/native-web-development/` | Vanilla HTML5, CSS3 Custom Properties, ES2022+, Web Components, Vite 5 |

### Back-end

| Skill | Folder | Stack |
|---|---|---|
| .NET Web API | `dotnet/dotnet-webapi-development/` | ASP.NET Core, .NET 8 LTS, EF Core 8, xUnit, FluentValidation, Moq |
| Python | `python/python-development/` | Python 3.11+, Typer, Pydantic v2, asyncio, Ruff, mypy |

### Design

| Skill | Folder | Covers |
|---|---|---|
| Design System | `design/design-system-development/` | Design tokens (three-tier), component API contracts, WCAG 2.1 AA, Figma handoff |

---

## Skill Layout

Every skill follows the same two-layer structure:

```
<technology>/<skill-name>/
├── SKILL.md          ← Core skill: conventions, rules, quick examples, and
│                        pointers to reference files. Load this into Claude Code.
└── references/
    ├── <topic-1>.md  ← Deep reference loaded on demand during a session
    ├── <topic-2>.md
    └── ...
```

### What each layer contains

**`SKILL.md`** is the entry point. It covers when to use the skill, project structure, the most important conventions, short code snippets, naming rules, and a "Customizing" section that lists every reference file and when to read it.

**`references/`** contains full, production-ready code for topics that need more depth — complete patterns, worked examples, testing strategies, and integration guides. Claude Code loads these on demand rather than all at once, keeping context lean.

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
2. **Fill it out** — follow the structure defined in the template. The `description` frontmatter field is critical: it tells Claude Code when to activate the skill.
3. **Add references** — place supporting files (full code samples, patterns, test strategies) in a `references/` subfolder alongside `SKILL.md`.
4. **Open a PR** — place your skill folder under `_review/` and open a pull request. A peer reviewer will approve and move it to the correct technology folder.
5. **Update the catalog** — add a row to the table above once the skill is merged.

### Naming Conventions

- Technology folders: `angular`, `reactjs`, `nextjs`, `native-web`, `dotnet`, `python`, `design`
- Skill folders: `<technology>-<purpose>` in kebab-case (e.g. `dotnet-webapi-development`)
- Version: start at `1.0.0`; bump minor for non-breaking additions, major for rewrites

---

## Shared Conventions

These coding standards apply across every skill in this repo:

- **CSS** — Never use `!important`. Fix specificity with a more targeted selector. The only accepted exception is the `prefers-reduced-motion` browser reset.
- **TypeScript** — Strict mode always on. No `any`, no untyped function parameters. Use `unknown` + type guard instead.
- **Python** — `from __future__ import annotations` in every file. Use `X | Y` union syntax. No `Optional[X]` — use `X | None`.
- **Accessibility** — All components target WCAG 2.1 AA minimum. Keyboard navigation, focus management, and screen reader patterns are non-negotiable.
- **Testing** — Each skill documents its testing stack in `references/testing.md`. Write tests alongside the code, not after.

---

## Audience

| Role | How they use this repo |
|---|---|
| **Developers** | Drop a skill into a project so Claude Code knows the team's framework conventions (Angular, React, Next.js, .NET, Python) |
| **Designers** | Use the Design System skill so Claude Code understands token hierarchy, component contracts, accessibility rules, and Figma handoff |
| **Architects** | Author or review skills to encode architectural decisions and enforce standards consistently across all AI-assisted development |

---

*Built with [Claude Code](https://claude.ai/code) · Licensed under [Apache 2.0](LICENSE)*
