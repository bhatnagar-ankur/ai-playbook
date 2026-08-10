---
name: design-system-development
version: 1.0.0
technology: design
author: Ankur Bhatnagar
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

Do **not** apply to: framework-specific implementation (Angular, React, Next.js, Native Web
have their own skills that cover component implementation). This skill covers the
**design contract** — what components are, how they behave, and how they are named.
Implementation skills consume these contracts.

---

## 2. Design Token Architecture

Design tokens are the **single source of truth** for all visual decisions.
They live in one canonical file and are referenced everywhere else.

### Token hierarchy

```
Global tokens  →  Semantic tokens  →  Component tokens
(raw values)      (named intent)       (scoped to one component)

--color-blue-600   --color-primary      --btn-background
--space-4          --space-interactive  --btn-padding-x
```

### Global tokens (never reference in components)

```css
/* tokens/global.css */

:root {
  /* ── Colour palette ────────────────────── */
  --color-blue-50:  #eff6ff;
  --color-blue-100: #dbeafe;
  --color-blue-200: #bfdbfe;
  --color-blue-500: #3b82f6;
  --color-blue-600: #2563eb;
  --color-blue-700: #1d4ed8;
  --color-blue-900: #1e3a8a;

  --color-red-500:  #ef4444;
  --color-red-600:  #dc2626;

  --color-green-500: #22c55e;
  --color-green-600: #16a34a;

  --color-yellow-400: #facc15;
  --color-yellow-500: #eab308;

  --color-neutral-0:   #ffffff;
  --color-neutral-50:  #f9fafb;
  --color-neutral-100: #f3f4f6;
  --color-neutral-200: #e5e7eb;
  --color-neutral-300: #d1d5db;
  --color-neutral-400: #9ca3af;
  --color-neutral-500: #6b7280;
  --color-neutral-600: #4b5563;
  --color-neutral-700: #374151;
  --color-neutral-800: #1f2937;
  --color-neutral-900: #111827;
  --color-neutral-950: #030712;

  /* ── Spacing scale (4px base) ──────────── */
  --space-0:   0;
  --space-1:   0.25rem;   /*  4px */
  --space-2:   0.5rem;    /*  8px */
  --space-3:   0.75rem;   /* 12px */
  --space-4:   1rem;      /* 16px */
  --space-5:   1.25rem;   /* 20px */
  --space-6:   1.5rem;    /* 24px */
  --space-8:   2rem;      /* 32px */
  --space-10:  2.5rem;    /* 40px */
  --space-12:  3rem;      /* 48px */
  --space-16:  4rem;      /* 64px */
  --space-20:  5rem;      /* 80px */
  --space-24:  6rem;      /* 96px */

  /* ── Typography ────────────────────────── */
  --font-family-sans:  'Inter', ui-sans-serif, system-ui, sans-serif;
  --font-family-mono:  'JetBrains Mono', ui-monospace, monospace;

  --font-size-xs:   0.75rem;    /* 12px */
  --font-size-sm:   0.875rem;   /* 14px */
  --font-size-base: 1rem;       /* 16px */
  --font-size-lg:   1.125rem;   /* 18px */
  --font-size-xl:   1.25rem;    /* 20px */
  --font-size-2xl:  1.5rem;     /* 24px */
  --font-size-3xl:  1.875rem;   /* 30px */
  --font-size-4xl:  2.25rem;    /* 36px */

  --font-weight-regular:  400;
  --font-weight-medium:   500;
  --font-weight-semibold: 600;
  --font-weight-bold:     700;

  --line-height-tight:  1.25;
  --line-height-snug:   1.375;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.625;

  /* ── Border radius ─────────────────────── */
  --radius-none: 0;
  --radius-sm:   0.125rem;  /*  2px */
  --radius-md:   0.375rem;  /*  6px */
  --radius-lg:   0.5rem;    /*  8px */
  --radius-xl:   0.75rem;   /* 12px */
  --radius-2xl:  1rem;      /* 16px */
  --radius-full: 9999px;

  /* ── Shadows ───────────────────────────── */
  --shadow-sm:  0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md:  0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
  --shadow-lg:  0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
  --shadow-xl:  0 20px 25px -5px rgb(0 0 0 / 0.1), 0 8px 10px -6px rgb(0 0 0 / 0.1);

  /* ── Z-index scale ─────────────────────── */
  --z-base:    0;
  --z-raised:  10;
  --z-dropdown: 100;
  --z-sticky:  200;
  --z-overlay: 300;
  --z-modal:   400;
  --z-toast:   500;

  /* ── Transition ────────────────────────── */
  --duration-fast:   100ms;
  --duration-normal: 200ms;
  --duration-slow:   300ms;
  --ease-default:    cubic-bezier(0.4, 0, 0.2, 1);
  --ease-in:         cubic-bezier(0.4, 0, 1, 1);
  --ease-out:        cubic-bezier(0, 0, 0.2, 1);
  --ease-spring:     cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
```

### Semantic tokens (reference global tokens — these are what components use)

```css
/* tokens/semantic.css */

:root {
  /* ── Brand ─────────────────────── */
  --color-brand:          var(--color-blue-600);
  --color-brand-hover:    var(--color-blue-700);
  --color-brand-subtle:   var(--color-blue-50);
  --color-brand-contrast: var(--color-neutral-0);

  /* ── Feedback ───────────────────── */
  --color-danger:         var(--color-red-600);
  --color-danger-subtle:  var(--color-red-50, #fef2f2);
  --color-success:        var(--color-green-600);
  --color-success-subtle: var(--color-green-50, #f0fdf4);
  --color-warning:        var(--color-yellow-500);
  --color-warning-subtle: var(--color-yellow-50, #fefce8);

  /* ── Neutral surface ────────────── */
  --color-surface:        var(--color-neutral-0);
  --color-surface-raised: var(--color-neutral-50);
  --color-surface-overlay: var(--color-neutral-100);
  --color-border:         var(--color-neutral-200);
  --color-border-strong:  var(--color-neutral-400);

  /* ── Text ───────────────────────── */
  --color-text-primary:   var(--color-neutral-900);
  --color-text-secondary: var(--color-neutral-600);
  --color-text-disabled:  var(--color-neutral-400);
  --color-text-inverse:   var(--color-neutral-0);
  --color-text-on-brand:  var(--color-neutral-0);

  /* ── Interactive ────────────────── */
  --space-interactive:    var(--space-2) var(--space-4);   /* Default button padding */
  --radius-interactive:   var(--radius-md);
  --focus-ring:           0 0 0 3px var(--color-blue-200);
  --focus-ring-danger:    0 0 0 3px var(--color-red-200, #fecaca);
}
```

---

## 3. Component API Patterns

Every component has a documented API with: variants, sizes, states, slots, and events.

### API contract template

```markdown
## Button

### Props
| Prop      | Type                                        | Default     | Description                       |
|-----------|---------------------------------------------|-------------|-----------------------------------|
| variant   | 'primary' \| 'secondary' \| 'ghost' \| 'danger' | 'primary' | Visual style                      |
| size      | 'sm' \| 'md' \| 'lg'                        | 'md'        | Height and padding scale          |
| disabled  | boolean                                     | false       | Prevents interaction              |
| loading   | boolean                                     | false       | Shows spinner; disables click     |
| type      | 'button' \| 'submit' \| 'reset'             | 'button'    | HTML button type                  |
| icon      | string (icon name)                          | —           | Leading icon                      |
| iconTrail | string (icon name)                          | —           | Trailing icon                     |

### Events
| Event   | Payload | Description           |
|---------|---------|-----------------------|
| click   | MouseEvent | Standard click     |

### Slots
| Slot    | Description           |
|---------|-----------------------|
| default | Button label text     |

### Accessibility
- Always renders a `<button>` element — never use `<div>` or `<span>` for buttons
- `disabled` attribute prevents focus and announces state to screen readers
- `loading` adds `aria-busy="true"` and `aria-label="Loading…"` to the spinner
- Minimum touch target: 44×44px
```

### Component naming rules

- Component name: `PascalCase` in code, `kebab-case` for HTML element (`<app-button>`, `<ui-button>`)
- Variant prop: named by visual intent, not implementation (`primary` not `blue`)
- Size prop: `sm` / `md` / `lg` — never pixel values as prop values
- Boolean props: positive framing (`disabled` not `enabled`, `loading` not `notReady`)
- Events: past-tense verbs (`change`, `submit`, `dismiss`) — never `on` prefix in the event name itself

---

## 4. Typography System

```css
/* Typography scale — use semantic class names, not utility classes */

.heading-1 {
  font-size:   var(--font-size-4xl);
  font-weight: var(--font-weight-bold);
  line-height: var(--line-height-tight);
  letter-spacing: -0.025em;
}

.heading-2 {
  font-size:   var(--font-size-3xl);
  font-weight: var(--font-weight-semibold);
  line-height: var(--line-height-tight);
  letter-spacing: -0.02em;
}

.heading-3 {
  font-size:   var(--font-size-2xl);
  font-weight: var(--font-weight-semibold);
  line-height: var(--line-height-snug);
}

.heading-4 {
  font-size:   var(--font-size-xl);
  font-weight: var(--font-weight-semibold);
  line-height: var(--line-height-snug);
}

.body-lg  { font-size: var(--font-size-lg);   line-height: var(--line-height-relaxed); }
.body-md  { font-size: var(--font-size-base);  line-height: var(--line-height-normal); }
.body-sm  { font-size: var(--font-size-sm);    line-height: var(--line-height-normal); }
.caption  { font-size: var(--font-size-xs);    line-height: var(--line-height-normal); color: var(--color-text-secondary); }
.overline { font-size: var(--font-size-xs);    font-weight: var(--font-weight-semibold); letter-spacing: 0.1em; text-transform: uppercase; }
.code     { font-family: var(--font-family-mono); font-size: 0.9em; }
```

**Typography rules:**
- Minimum body font size: 16px (1rem) — never below 14px for readable prose
- Line length: 45–75 characters for body text (`max-width: 65ch`)
- Never justify text — use left-align (start) for body copy
- Heading hierarchy must be logical — no skipping levels in HTML
- Avoid more than 2 font families per system

---

## 5. Colour System & Theming

### Contrast requirements (WCAG 2.1 AA)

| Pair | Min ratio | Notes |
|---|---|---|
| Normal text on background | 4.5:1 | `font-size < 18px` (or `< 14px bold`) |
| Large text on background | 3:1 | `font-size >= 18px` (or `>= 14px bold`) |
| UI component / state indicator | 3:1 | Borders, icons, input outlines |
| Decorative / disabled | No requirement | Must not convey meaning alone |

```css
/* Verify with: https://webaim.org/resources/contrastchecker/ */

/* These pairs meet AA: */
/* --color-text-primary (#111827) on --color-surface (#fff)  → 16.7:1 */
/* --color-brand (#2563eb) on --color-surface (#fff)         →  5.9:1 */
/* --color-text-secondary (#4b5563) on --color-surface (#fff) → 7.0:1 */
```

### Rules
- Never convey meaning by colour alone — always pair with an icon or text label
- Disabled states use opacity or muted colour — never the primary danger/success colour
- Never use `!important` to override theme values — cascade through token specificity instead

---

## 6. Spacing & Layout System

```css
/* Layout tokens */
:root {
  --layout-max-width:     80rem;      /* 1280px */
  --layout-gutter:        var(--space-4);
  --layout-gutter-lg:     var(--space-8);
  --layout-column-gap:    var(--space-6);
  --layout-section-gap:   var(--space-16);
}

/* Density variants */
:root {
  --density-compact:  -0.25;   /* Multiplier offset for compact mode */
  --density-default:   0;
  --density-spacious:  0.25;
}
```

**Spacing rules:**
- Always use token values — never hard-code pixel values
- Consistent 4px base grid: all spacing is a multiple of 4px
- Inner component spacing (padding, gaps): use `--space-1` through `--space-6`
- Section-level vertical rhythm: use `--space-8` through `--space-24`
- Layout gutters: `--layout-gutter` on mobile, `--layout-gutter-lg` on desktop

---

## 7. Accessibility (WCAG 2.1 AA)

All components must meet WCAG 2.1 Level AA. Requirements are listed below.

**Non-negotiable requirements:**
- Keyboard navigable — all interactive elements reachable with Tab and activated with Enter/Space
- Visible focus indicator — `:focus-visible` outline, minimum 3px, contrasting colour
- Screen reader labels — every interactive element has an accessible name
- No keyboard traps — users can always exit any focused region with Escape or Tab
- Form errors — announced via `aria-live` or `aria-describedby`, not colour alone
- Images — informative images have descriptive `alt`; decorative images have `alt=""`
- Motion — respect `prefers-reduced-motion`; never use `!important` except in the motion reset
- Touch targets — minimum 44×44px tappable area for all interactive elements

---

## 8. Figma Handoff Conventions

### Component naming in Figma

```
Pattern: {ComponentName}/{Variant}/{State}

Examples:
Button/Primary/Default
Button/Primary/Hover
Button/Primary/Disabled
Button/Secondary/Default
Form Field/Default/Empty
Form Field/Default/Error
Form Field/Default/Filled
Modal/Default
Badge/Success
Badge/Danger
```

### Layer naming rules
- Use English, sentence case: `Button label` not `ButtonLabel` or `button label`
- Auto Layout frames: name after the component (`Card body`, `Input wrapper`)
- Hidden layers start with `_` (`_Mask`, `_Shadow layer`)
- Never leave Figma default names (`Frame 42`, `Rectangle 1`)

### Variables & Code Connect
- Map every semantic token to a Figma Variable in the matching collection (Color, Spacing, Typography)
- Use Code Connect to bind Figma components to their code counterparts so Inspect shows real props
- Export tokens as JSON via the Tokens Studio plugin or Figma Variables REST API for CI sync

---

## 9. Motion & Animation

```css
/* Base transition for interactive state changes */
.interactive {
  transition:
    background-color var(--duration-fast)   var(--ease-default),
    border-color     var(--duration-fast)   var(--ease-default),
    color            var(--duration-fast)   var(--ease-default),
    box-shadow       var(--duration-fast)   var(--ease-default),
    opacity          var(--duration-normal) var(--ease-default);
}

/* Entrance animations */
@keyframes fade-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}

@keyframes slide-in-up {
  from { opacity: 0; transform: translateY(var(--space-4)); }
  to   { opacity: 1; transform: translateY(0); }
}

/* prefers-reduced-motion — only legitimate use of !important */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration:        0.01ms !important;
    animation-iteration-count: 1      !important;
    transition-duration:       0.01ms !important;
    scroll-behavior:           auto   !important;
  }
}
```

**Motion rules:**
- Duration: state changes ≤ 150ms; entrances/exits ≤ 300ms; never > 500ms without user action
- Always animate `transform` and `opacity` for GPU compositing — never `width`/`height`/`top`
- Provide instant alternatives for `prefers-reduced-motion: reduce` users
- Avoid looping animations in the main content area — distract from content

---

## 10. Dark Mode

```css
/* tokens/dark.css — override semantic tokens */

@media (prefers-color-scheme: dark) {
  :root {
    --color-surface:        var(--color-neutral-900);
    --color-surface-raised: var(--color-neutral-800);
    --color-surface-overlay: var(--color-neutral-700);
    --color-border:         var(--color-neutral-700);
    --color-border-strong:  var(--color-neutral-500);

    --color-text-primary:   var(--color-neutral-50);
    --color-text-secondary: var(--color-neutral-400);
    --color-text-disabled:  var(--color-neutral-600);

    --color-brand:          var(--color-blue-500);
    --color-brand-hover:    var(--color-blue-400);
    --color-brand-subtle:   var(--color-blue-900);

    --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.4);
    --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.4);
  }
}

/* Manual toggle support alongside media query */
[data-theme="dark"] {
  /* Same overrides as @media block above */
}
```

**Dark mode rules:**
- Only override semantic tokens — global tokens never change
- Re-verify colour contrast in dark mode — light-mode ratios do not transfer
- Avoid pure `#000000` backgrounds — use `--color-neutral-900` or `--color-neutral-950`
- Avoid pure `#ffffff` on dark — use `--color-neutral-50` or `--color-neutral-100`
- Test all interactive states (hover, focus, active, disabled) in dark mode

---

## 11. Component Documentation Template

Every component in the design system must include:

```markdown
# {ComponentName}

> One-sentence description of the component's purpose.

## When to use
- [Use case 1]
- [Use case 2]

## When NOT to use
- [Anti-pattern 1 — link to alternative]

## Variants
[Table or descriptions of each variant]

## Sizes
[Table or descriptions: sm / md / lg]

## States
Default, Hover, Focus, Active, Disabled, Loading, Error

## API
[Props / slots / events table]

## Accessibility
- Role: [e.g. `button`, `dialog`, `listitem`]
- Keyboard: [Tab, Enter, Space, Escape behaviours]
- Screen reader: [what is announced and when]
- Focus management: [where focus goes on open/close/submit]

## Do / Don't
| Do | Don't |
|----|-------|
| Use for primary actions | Stack multiple primary buttons |

## Related components
- [Link to related component]
```

---

## 12. Naming Conventions

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

## 13. Customizing This Skill

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
