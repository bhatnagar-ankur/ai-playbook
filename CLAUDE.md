# AI Playbook — Claude Code Context

This repo is a **skill library**, not a runnable application. It contains no source code, build system, or tests.

## What lives here

- `SKILL.md` files — structured coding standards for Claude Code to load into a target project
- `references/` subfolders — deep reference files loaded on demand by Claude during a session
- `_templates/SKILL-TEMPLATE.md` — canonical template for authoring new skills
- `_review/` — staging area for skills pending peer review

## How skills are used

Skills are **not** loaded here. They are copied into a target project's `skills/` directory, where Claude Code picks them up automatically. When working in this repo, your job is to **author, review, or improve** skill files — not to run them.

## Authoring a new skill

1. Copy `_templates/SKILL-TEMPLATE.md` to `<technology>/<technology>-<purpose>/SKILL.md`.
2. Fill in all frontmatter fields — especially `description` (Claude uses this to decide when to activate the skill).
3. Add deep reference files in `references/`.
4. Open a PR with the skill folder placed under `_review/`.
5. Update the catalog table in `README.md` after merge.

## Reviewing an existing skill

When asked to review or improve a skill:
- Read the full `SKILL.md` first, then the relevant `references/` files.
- Check that the `description` frontmatter accurately describes **when** the skill should activate.
- Verify the stack versions in frontmatter match the code examples.
- Ensure every rule has a clear rationale (implicit or explicit).
- Flag outdated version references (e.g. React 18 examples in a React 19 skill).

## Shared cross-skill rules

- TypeScript: strict mode, no `any`, `unknown` + type guard instead
- CSS: no `!important` except `prefers-reduced-motion` reset
- Python: `from __future__ import annotations`, `X | Y` unions, no `Optional[X]`
- Accessibility: WCAG 2.1 AA minimum on all UI-facing skills
- Testing: document the testing stack in `references/testing.md`
