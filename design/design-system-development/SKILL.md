---
name: design-system-development
version: 1.0.0
technology: design
author: Ankur Bhatnagar
last_updated: 2026-09-07
description: >
  UI/UX design system creation skill covering design tokens, component APIs,
  typography, colour, spacing, accessibility (WCAG 2.1 AA), Figma handoff
  conventions, motion, dark mode, and component documentation.
---

# Design System Development Skill

This skill guides Claude to produce consistent, accessible, and well-documented
design systems — from token architecture through component APIs to Figma handoff.
Follow every rule in this document unless the **Customizing** section overrides it.

---

## 1. When to Use This Skill

Use this skill when:
- Creating or extending a design token set (colours, spacing, typography, radii)
- Defining component API contracts (props, slots, variants, states)
- Writing component documentation (usage, do/don't, accessibility notes)
- Establishing Figma naming conventions, component structure, or handoff rules
- Adding dark mode support to an existing system
- Reviewing or auditing a design system for WCAG 2.1 AA compliance
- Generating a base CSS/token layer for use with any CSS framework
- Example trigger phrases: `"create a design token"`, `"define a component API"`,
  `"audit for WCAG"`, `"set up Figma naming conventions"`, `"add dark mode"`

**Do NOT use when:**
- The task is framework-specific implementation — Angular, React, Next.js, and Native
  Web have their own skills that cover component implementation. This skill covers the
  **design contract** — what components are, how they behave, and how they are named.
  Implementation skills consume these contracts.

---

## 2. Design Token Architecture

Design tokens are the **single source of truth** for all visual decisions. They flow
through three tiers — never skip a tier by referencing a global token from a component:

```
Global tokens  →  Semantic tokens  →  Component tokens
(raw values)      (named intent)       (scoped to one component)

--color-blue-600   --color-primary      --btn-background
--space-4          --space-interactive  --btn-padding-x
```

**Rules:**
- Global tokens (raw palette, spacing scale, type scale, radii, shadows, z-index,
  transition primitives) are never referenced directly by components
- Semantic tokens name *intent* (`--color-brand`, `--color-danger`) and are the only
  tier components should consume
- Component tokens are scoped with a component prefix (`--btn-*`) and derive from
  semantic tokens only

```css
/* tokens/semantic.css — components consume this tier, never global.css directly */
:root {
  --color-brand:        var(--color-blue-600);
  --color-brand-hover:  var(--color-blue-700);
  --color-text-primary: var(--color-neutral-900);
  --color-surface:      var(--color-neutral-0);
  --focus-ring:         0 0 0 3px var(--color-blue-200);
}
```

Full token set: see `references/tokens.md` (complete global + semantic layers, spacing
& layout system, motion/animation, and dark mode overrides).

---

## 3. Component API Patterns & Documentation

Every component has a documented API — variants, sizes, states, slots, and events —
plus a documentation page following a fixed template (When to use, Variants, States,
API, Accessibility, Do/Don't).

**Rules:**
- Component name: `PascalCase` in code, `kebab-case` for the HTML element (`<ui-button>`)
- Variant prop: named by visual intent, not implementation (`primary` not `blue`)
- Size prop: `sm` / `md` / `lg` — never pixel values as prop values
- Boolean props: positive framing (`disabled` not `enabled`, `loading` not `notReady`)
- Events: past-tense verbs (`change`, `submit`, `dismiss`) — never an `on` prefix on
  the event name itself
- Every component ships with an Accessibility section stating its role, keyboard
  behaviour, and screen-reader announcements — not just its visual API

```markdown
### Button — Props (excerpt)
| Prop     | Type                                  | Default   | Description          |
|----------|----------------------------------------|-----------|------------------------|
| variant  | 'primary' \| 'secondary' \| 'danger'    | 'primary' | Visual style           |
| size     | 'sm' \| 'md' \| 'lg'                    | 'md'      | Height/padding scale   |
| disabled | boolean                                | false     | Prevents interaction   |
```

Full API contract template, the Component Documentation Template, and a complete
worked Button example: see `references/components.md` and `references/examples.md`.

---

## 4. Typography System

```css
.heading-1 {
  font-size:   var(--font-size-4xl);
  font-weight: var(--font-weight-bold);
  line-height: var(--line-height-tight);
  letter-spacing: -0.025em;
}

.body-md  { font-size: var(--font-size-base); line-height: var(--line-height-normal); }
.caption  { font-size: var(--font-size-xs); color: var(--color-text-secondary); }
```

**Typography rules:**
- Minimum body font size: 16px (1rem) — never below 14px for readable prose
- Line length: 45–75 characters for body text (`max-width: 65ch`)
- Never justify text — use left-align (start) for body copy (why: justification on the
  web stretches word spacing unevenly per line, producing ragged internal spacing and
  "rivers" of whitespace that slow reading and hurt low-vision users more than a simple
  ragged right edge does)
- Heading hierarchy must be logical — no skipping levels in HTML
- Avoid more than 2 font families per system (why: every extra family adds a font-loading
  request — increasing page weight and layout shift risk — and adds visual noise that
  raises cognitive load without adding communicative value)

---

## 5. Colour System & Contrast

| Pair | Min ratio | Notes |
|---|---|---|
| Normal text on background | 4.5:1 | Below the large-text threshold |
| Large text on background | 3:1 | ≥ 18pt (~24px), or ≥ 14pt bold (~19px bold) |
| UI component / state indicator | 3:1 | Borders, icons, input outlines |

**Rules:**
- Never convey meaning by colour alone — always pair with an icon or text label
- Disabled states use opacity or muted colour — never the primary danger/success colour
- Never use `!important` to override theme values — cascade through token specificity
  instead. The single system-wide exception is the `prefers-reduced-motion` reset
  (Section 9) — see `references/accessibility.md` for why.

Full contrast ratio table (with corrected, recomputed values) and the complete WCAG
2.1 AA checklist: see `references/accessibility.md`.

---

## 6. Spacing & Layout System

**Rules:**
- Always use token values — never hard-code pixel values
- Consistent 4px base grid: all spacing is a multiple of 4px
- Inner component spacing (padding, gaps): use `--space-1` through `--space-6`
- Section-level vertical rhythm: use `--space-8` through `--space-24`
- Layout gutters: `--layout-gutter` on mobile, `--layout-gutter-lg` on desktop

```css
:root {
  --layout-max-width:   80rem;   /* 1280px */
  --layout-gutter:      var(--space-4);
  --layout-section-gap: var(--space-16);
}
```

Full layout token set and density variants: see `references/tokens.md`.

---

## 7. Accessibility (WCAG 2.1 AA)

All components must meet WCAG 2.1 Level AA.

**Non-negotiable requirements:**
- Keyboard navigable — all interactive elements reachable with Tab, activated with Enter/Space
- Visible focus indicator — `:focus-visible` outline, minimum 3px, contrasting colour
- Screen reader labels — every interactive element has an accessible name
- No keyboard traps — users can always exit any focused region with Escape or Tab
- Form errors — announced via `aria-live` or `aria-describedby`, not colour alone
- Images — informative images have descriptive `alt`; decorative images have `alt=""`
- Motion — respect `prefers-reduced-motion` (full rule and code: Section 9 / `references/tokens.md`)
- Touch targets — minimum **24×24px** (WCAG 2.2 AA, SC 2.5.8); 44×44px is a stretch
  target (WCAG 2.1 AAA, SC 2.5.5), not the AA requirement — see `references/accessibility.md`
  for the full explanation

Full WCAG 2.1 AA checklist and contrast rules: see `references/accessibility.md`.

---

## 8. Figma Handoff Conventions

```
Pattern: {ComponentName}/{Variant}/{State}

Button/Primary/Default
Button/Primary/Hover
Badge/Success
```

**Rules:**
- Use English, sentence case for layer names: `Button label` not `ButtonLabel`
- Hidden layers start with `_` (`_Mask`, `_Shadow layer`) — never leave Figma default
  names (`Frame 42`, `Rectangle 1`)
- Map every semantic token to a Figma Variable in the matching collection (Color,
  Spacing, Typography); bind components via Code Connect so Inspect shows real props

Full naming patterns, layer rules, and Code Connect/variable conventions: see
`references/figma.md`.

---

## 9. Motion & Animation

```css
.interactive {
  transition: background-color var(--duration-fast) var(--ease-default);
}

/* prefers-reduced-motion — the only place !important is allowed in this system */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

**Motion rules:**
- Duration: state changes ≤ 150ms; entrances/exits ≤ 300ms; never > 500ms without user
  action (why: duration is a perceived-responsiveness signal — anything slower than
  ~150ms for a small state change reads as laggy, while a large layout change that
  resolves too fast feels jarring; the ceiling scales with how much of the UI is changing)
- Always animate `transform` and `opacity` for GPU compositing — never `width`/`height`/`top`
- Avoid looping animations in the main content area — they distract from content

Full transition primitives, keyframes, and the complete reduced-motion code block: see
`references/tokens.md`.

---

## 10. Dark Mode

**Rules:**
- Only override semantic tokens — global tokens never change
- Re-verify colour contrast in dark mode — light-mode ratios do not transfer
- Avoid pure `#000000`/`#ffffff` — use `--color-neutral-950`/`--color-neutral-50` instead
- Test all interactive states (hover, focus, active, disabled) in dark mode

```css
@media (prefers-color-scheme: dark) {
  :root {
    --color-surface:      var(--color-neutral-900);
    --color-text-primary: var(--color-neutral-50);
    --color-brand:        var(--color-blue-500);
  }
}
```

Full dark-mode override set (including the manual `[data-theme="dark"]` toggle): see
`references/tokens.md`.

---

## 11. Naming Conventions

| Artefact | Convention | Example |
|---|---|---|
| Global token | `--color-{palette}-{step}` | `--color-blue-600` |
| Semantic token | `--color-{role}[-{modifier}]` | `--color-brand`, `--color-text-secondary` |
| Component token | `--{component}-{property}` | `--btn-background` |
| CSS class | `kebab-case` | `.btn`, `.form-field`, `.status-badge` |
| Modifier class | `{block}--{modifier}` (BEM) | `.btn--primary`, `.btn--lg` |
| Figma component | `PascalCase/Variant/State` | `Button/Primary/Hover` |
| Figma variable | `{group}/{name}` | `brand/primary`, `text/secondary` |
| Icon name | `{noun}-{modifier}` | `arrow-right`, `check-circle` |

---

## 12. Customizing This Skill

### Reference file lookup

Claude reads this SKILL.md first; it opens a reference file only when a task needs
deeper detail on that specific topic.

| Topic | File | When to read |
|---|---|---|
| Full token set (global + semantic), spacing/layout, motion, dark mode | `references/tokens.md` | When creating or extending the token layer, tuning layout/density, or working with animation or dark-mode overrides |
| Component API contracts & documentation template | `references/components.md` | When defining a new component's props/slots/events or writing its documentation page |
| Contrast rules & full WCAG 2.1 AA checklist | `references/accessibility.md` | When auditing accessibility or needing exact contrast/touch-target requirements |
| Figma naming & handoff conventions | `references/figma.md` | When establishing or reviewing Figma component/variable naming |
| Full worked example | `references/examples.md` | When building a complete component end-to-end (tokens → API → Figma → docs) |

### Project overrides

Record project-level overrides here when the team deviates from a default.

```markdown
## Project Overrides — [Project Name]

- Brand colour: #0F4C81 (Cerulean) — use as --color-brand base
- Font: 'Poppins' for headings, 'Lato' for body (loaded via Google Fonts)
- Border radius: flat design — all radii are 0 or --radius-sm only
- Spacing base: 8px grid (not 4px)
- Icon library: Heroicons (outline) — not Phosphor
- Dark mode: manual toggle only (data-theme="dark") — no prefers-color-scheme
- Motion: no entrance animations — transitions only for interactive states
```
