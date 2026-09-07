# Design Tokens — Full Reference

Full token architecture, spacing/layout system, motion/animation, and dark mode
for the design-system-development skill. See `SKILL.md` for the short summary
and representative snippets; this file holds the complete, verbatim code.

---

## Design Token Architecture

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

## Spacing & Layout System

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

## Motion & Animation

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

/* prefers-reduced-motion — the only legitimate use of !important anywhere
   in this design system. Every other rule in this skill (colour theming,
   component styling, dark mode) must win on selector specificity instead. */
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
  (why: duration is a perceived-responsiveness signal — a state change that takes longer than
  ~150ms reads as laggy, while a large UI change that resolves in under ~200ms feels jarring
  or gets missed entirely; the ceiling scales with how much the UI is visually changing)
- Always animate `transform` and `opacity` for GPU compositing — never `width`/`height`/`top`
- Provide instant alternatives for `prefers-reduced-motion: reduce` users
- Avoid looping animations in the main content area — distract from content
- `!important` is permitted **only** in the `prefers-reduced-motion` block above — see
  `references/accessibility.md` for the accessibility rationale behind this exception

---

## Dark Mode

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
