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
├── angular/                   ← Angular skills
├── reactjs/                   ← React skills
├── nextjs/                    ← Next.js skills
├── dotnet/                    ← .NET / C# skills
├── java/                      ← Java / Spring skills
├── python/                    ← Python skills
└── design/                    ← Design tooling & systems skills
```

Technology folders use lowercase kebab-case. Each skill lives in its own subfolder named `<technology>-<purpose>` (e.g. `angular/angular-component-library/`). New top-level folders are created when the first skill for a new technology is contributed.

---

## Skill Catalog

> Skills are listed here once they pass review. Add a row when merging from `_review/`.

| Technology | Skill | Description |
|---|---|---|
| — | — | No skills published yet. Be the first to contribute! |

---

## Contributing a Skill

1. **Copy the template** — duplicate `_templates/SKILL-TEMPLATE.md` into the appropriate technology folder: `<technology>/<technology>-<purpose>/SKILL.md`.
2. **Fill it out** — follow the structure defined in the template. The `description` frontmatter field is the most important: it tells Claude when to activate the skill.
3. **Add references** — place any supporting files (code samples, config snippets, diagrams) in a `references/` subfolder alongside `SKILL.md`.
4. **Open a PR** — place your skill folder under `_review/` and open a pull request. A peer reviewer will approve and move it to the correct technology folder.
5. **Update the catalog** — add a row to the table above once the skill is merged.

### Naming Conventions

- Technology folders: `angular`, `reactjs`, `nextjs`, `dotnet`, `java`, `python`, `design`
- Skill folders: `<technology>-<purpose>` in kebab-case
- Version: start at `1.0.0`; bump minor for non-breaking additions, major for rewrites

---

## Audience

| Role | Typical usage |
|---|---|
| **Developers** | Framework and library skills (Angular, React, Next.js, .NET, Java, Python) |
| **Designers** | Design-system and tooling skills (Figma workflows, tokens, accessibility) |
| **Architects** | Architectural skills (system design, API patterns, IaC, performance) |
