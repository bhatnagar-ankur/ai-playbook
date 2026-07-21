---
author: Ankur Bhatnagar
---

# Design Tokens — Full Reference

Complete token set: global primitive tokens, semantic tokens, component-level tokens, dark mode overrides, and density variants.

---

## Table of Contents
1. [Global Tokens — Colour Palette](#global-tokens--colour-palette)
2. [Global Tokens — Spacing](#global-tokens--spacing)
3. [Global Tokens — Typography](#global-tokens--typography)
4. [Global Tokens — Shape & Shadow](#global-tokens--shape--shadow)
5. [Global Tokens — Motion](#global-tokens--motion)
6. [Global Tokens — Z-Index](#global-tokens--z-index)
7. [Semantic Tokens — Light Mode](#semantic-tokens--light-mode)
8. [Semantic Tokens — Dark Mode Overrides](#semantic-tokens--dark-mode-overrides)
9. [Component Tokens — Examples](#component-tokens--examples)
10. [Density Variants](#density-variants)
11. [Token Naming Conventions](#token-naming-conventions)
12. [JavaScript / TypeScript Token Map](#javascript--typescript-token-map)

---

## Global Tokens — Colour Palette

> Global colour tokens are raw values. **Never reference them directly in components** — always go through a semantic token.

```css
/* tokens/global/_color.css */

:root {
  /* Blue */
  --color-blue-50:  #eff6ff;
  --color-blue-100: #dbeafe;
  --color-blue-200: #bfdbfe;
  --color-blue-300: #93c5fd;
  --color-blue-400: #60a5fa;
  --color-blue-500: #3b82f6;
  --color-blue-600: #2563eb;
  --color-blue-700: #1d4ed8;
  --color-blue-800: #1e40af;
  --color-blue-900: #1e3a8a;

  /* Red */
  --color-red-50:  #fef2f2;
  --color-red-100: #fee2e2;
  --color-red-200: #fecaca;
  --color-red-300: #fca5a5;
  --color-red-400: #f87171;
  --color-red-500: #ef4444;
  --color-red-600: #dc2626;
  --color-red-700: #b91c1c;
  --color-red-800: #991b1b;
  --color-red-900: #7f1d1d;

  /* Green */
  --color-green-50:  #f0fdf4;
  --color-green-100: #dcfce7;
  --color-green-200: #bbf7d0;
  --color-green-300: #86efac;
  --color-green-400: #4ade80;
  --color-green-500: #22c55e;
  --color-green-600: #16a34a;
  --color-green-700: #15803d;
  --color-green-800: #166534;
  --color-green-900: #14532d;

  /* Yellow / Amber */
  --color-yellow-50:  #fffbeb;
  --color-yellow-100: #fef3c7;
  --color-yellow-200: #fde68a;
  --color-yellow-300: #fcd34d;
  --color-yellow-400: #fbbf24;
  --color-yellow-500: #f59e0b;
  --color-yellow-600: #d97706;
  --color-yellow-700: #b45309;
  --color-yellow-800: #92400e;
  --color-yellow-900: #78350f;

  /* Neutral (Gray) */
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

  /* Purple */
  --color-purple-50:  #faf5ff;
  --color-purple-100: #f3e8ff;
  --color-purple-200: #e9d5ff;
  --color-purple-300: #d8b4fe;
  --color-purple-400: #c084fc;
  --color-purple-500: #a855f7;
  --color-purple-600: #9333ea;
  --color-purple-700: #7e22ce;
  --color-purple-800: #6b21a8;
  --color-purple-900: #581c87;

  /* Orange */
  --color-orange-50:  #fff7ed;
  --color-orange-100: #ffedd5;
  --color-orange-200: #fed7aa;
  --color-orange-300: #fdba74;
  --color-orange-400: #fb923c;
  --color-orange-500: #f97316;
  --color-orange-600: #ea580c;
  --color-orange-700: #c2410c;
  --color-orange-800: #9a3412;
  --color-orange-900: #7c2d12;

  /* Transparent */
  --color-transparent: transparent;
  --color-white:       #ffffff;
  --color-black:       #000000;

  /* Alpha variants — use sparingly */
  --color-black-a10: rgba(0, 0, 0, 0.10);
  --color-black-a20: rgba(0, 0, 0, 0.20);
  --color-black-a40: rgba(0, 0, 0, 0.40);
  --color-black-a60: rgba(0, 0, 0, 0.60);
  --color-white-a10: rgba(255, 255, 255, 0.10);
  --color-white-a20: rgba(255, 255, 255, 0.20);
  --color-white-a40: rgba(255, 255, 255, 0.40);
  --color-white-a80: rgba(255, 255, 255, 0.80);
}
```

---

## Global Tokens — Spacing

```css
/* tokens/global/_spacing.css */
/* Scale: 4 px base grid */

:root {
  --space-0:   0;          /*   0 px */
  --space-px:  1px;        /*   1 px */
  --space-0-5: 0.125rem;   /*   2 px */
  --space-1:   0.25rem;    /*   4 px */
  --space-1-5: 0.375rem;   /*   6 px */
  --space-2:   0.5rem;     /*   8 px */
  --space-2-5: 0.625rem;   /*  10 px */
  --space-3:   0.75rem;    /*  12 px */
  --space-3-5: 0.875rem;   /*  14 px */
  --space-4:   1rem;       /*  16 px */
  --space-5:   1.25rem;    /*  20 px */
  --space-6:   1.5rem;     /*  24 px */
  --space-7:   1.75rem;    /*  28 px */
  --space-8:   2rem;       /*  32 px */
  --space-9:   2.25rem;    /*  36 px */
  --space-10:  2.5rem;     /*  40 px */
  --space-11:  2.75rem;    /*  44 px */
  --space-12:  3rem;       /*  48 px */
  --space-14:  3.5rem;     /*  56 px */
  --space-16:  4rem;       /*  64 px */
  --space-20:  5rem;       /*  80 px */
  --space-24:  6rem;       /*  96 px */
  --space-32:  8rem;       /* 128 px */
  --space-40:  10rem;      /* 160 px */
  --space-48:  12rem;      /* 192 px */
  --space-64:  16rem;      /* 256 px */
}
```

---

## Global Tokens — Typography

```css
/* tokens/global/_typography.css */

:root {
  /* Font families */
  --font-family-sans:  "Inter", system-ui, -apple-system, BlinkMacSystemFont,
                       "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  --font-family-mono:  "JetBrains Mono", "Fira Code", "Cascadia Code",
                       ui-monospace, "Courier New", monospace;

  /* Font size scale */
  --font-size-xs:   0.75rem;   /*  12 px */
  --font-size-sm:   0.875rem;  /*  14 px */
  --font-size-base: 1rem;      /*  16 px */
  --font-size-lg:   1.125rem;  /*  18 px */
  --font-size-xl:   1.25rem;   /*  20 px */
  --font-size-2xl:  1.5rem;    /*  24 px */
  --font-size-3xl:  1.875rem;  /*  30 px */
  --font-size-4xl:  2.25rem;   /*  36 px */
  --font-size-5xl:  3rem;      /*  48 px */
  --font-size-6xl:  3.75rem;   /*  60 px */

  /* Font weights */
  --font-weight-regular:   400;
  --font-weight-medium:    500;
  --font-weight-semibold:  600;
  --font-weight-bold:      700;

  /* Line heights */
  --line-height-none:    1;
  --line-height-tight:   1.25;
  --line-height-snug:    1.375;
  --line-height-normal:  1.5;
  --line-height-relaxed: 1.625;
  --line-height-loose:   2;

  /* Letter spacing */
  --letter-spacing-tight:  -0.025em;
  --letter-spacing-normal:  0em;
  --letter-spacing-wide:    0.025em;
  --letter-spacing-wider:   0.05em;
  --letter-spacing-widest:  0.1em;
}
```

---

## Global Tokens — Shape & Shadow

```css
/* tokens/global/_shape.css */

:root {
  /* Border radius */
  --radius-none:   0;
  --radius-sm:     0.125rem;   /*  2 px */
  --radius-base:   0.25rem;    /*  4 px */
  --radius-md:     0.375rem;   /*  6 px */
  --radius-lg:     0.5rem;     /*  8 px */
  --radius-xl:     0.75rem;    /* 12 px */
  --radius-2xl:    1rem;       /* 16 px */
  --radius-3xl:    1.5rem;     /* 24 px */
  --radius-full:   9999px;

  /* Border widths */
  --border-width-none:   0;
  --border-width-thin:   1px;
  --border-width-medium: 2px;
  --border-width-thick:  4px;

  /* Shadows */
  --shadow-none: none;
  --shadow-xs:   0 1px 2px 0 rgba(0, 0, 0, 0.05);
  --shadow-sm:   0 1px 3px 0 rgba(0, 0, 0, 0.10),
                 0 1px 2px -1px rgba(0, 0, 0, 0.10);
  --shadow-md:   0 4px 6px -1px rgba(0, 0, 0, 0.10),
                 0 2px 4px -2px rgba(0, 0, 0, 0.10);
  --shadow-lg:   0 10px 15px -3px rgba(0, 0, 0, 0.10),
                 0 4px 6px -4px rgba(0, 0, 0, 0.10);
  --shadow-xl:   0 20px 25px -5px rgba(0, 0, 0, 0.10),
                 0 8px 10px -6px rgba(0, 0, 0, 0.10);
  --shadow-2xl:  0 25px 50px -12px rgba(0, 0, 0, 0.25);
  --shadow-inner: inset 0 2px 4px 0 rgba(0, 0, 0, 0.06);

  /* Focus ring — consistent across all interactive elements */
  --shadow-focus: 0 0 0 3px rgba(59, 130, 246, 0.50);   /* blue-500 @ 50% */
}
```

---

## Global Tokens — Motion

```css
/* tokens/global/_motion.css */

:root {
  /* Durations */
  --duration-instant:  0ms;
  --duration-fast:     100ms;
  --duration-normal:   150ms;   /* State changes: hover, active */
  --duration-moderate: 200ms;
  --duration-slow:     300ms;   /* Entrances / exits */
  --duration-slower:   400ms;
  --duration-slowest:  500ms;

  /* Easing functions */
  --easing-linear:          linear;
  --easing-ease:            ease;
  --easing-ease-in:         cubic-bezier(0.4, 0, 1, 1);
  --easing-ease-out:        cubic-bezier(0, 0, 0.2, 1);
  --easing-ease-in-out:     cubic-bezier(0.4, 0, 0.2, 1);
  --easing-spring:          cubic-bezier(0.34, 1.56, 0.64, 1);  /* Slight overshoot */
  --easing-anticipate:      cubic-bezier(0.36, 0, 0.66, -0.56); /* Wind-up before exit */

  /* Shorthand transition presets */
  --transition-colors:    color var(--duration-normal) var(--easing-ease-in-out),
                          background-color var(--duration-normal) var(--easing-ease-in-out),
                          border-color var(--duration-normal) var(--easing-ease-in-out),
                          fill var(--duration-normal) var(--easing-ease-in-out);
  --transition-opacity:   opacity var(--duration-normal) var(--easing-ease-in-out);
  --transition-shadow:    box-shadow var(--duration-normal) var(--easing-ease-in-out);
  --transition-transform: transform var(--duration-slow) var(--easing-ease-out);
  --transition-all:       all var(--duration-normal) var(--easing-ease-in-out);
}

/* Honour user motion preference */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration:        0.01ms !important; /* The only legitimate use of !important */
    animation-iteration-count: 1 !important;
    transition-duration:       0.01ms !important;
    scroll-behavior:           auto !important;
  }
}
```

---

## Global Tokens — Z-Index

```css
/* tokens/global/_z-index.css */

:root {
  --z-index-base:        0;
  --z-index-raised:     10;     /* Sticky table headers, sticky sidebars */
  --z-index-dropdown:  100;     /* Dropdowns, autocomplete panels */
  --z-index-sticky:    200;     /* Sticky navigation bars */
  --z-index-overlay:   300;     /* Scrim / dimmer behind modal */
  --z-index-modal:     400;     /* Modal dialogs */
  --z-index-toast:     500;     /* Toast notifications — always on top */
}
```

---

## Semantic Tokens — Light Mode

```css
/* tokens/semantic/_semantic.css */
/* Semantic tokens reference global tokens. Components reference semantic tokens. */

:root {
  /* --- Brand --- */
  --color-brand-primary:          var(--color-blue-600);
  --color-brand-primary-hover:    var(--color-blue-700);
  --color-brand-primary-active:   var(--color-blue-800);
  --color-brand-primary-subtle:   var(--color-blue-50);
  --color-brand-secondary:        var(--color-neutral-800);
  --color-brand-secondary-hover:  var(--color-neutral-900);

  /* --- Feedback: Error --- */
  --color-feedback-error:         var(--color-red-600);
  --color-feedback-error-hover:   var(--color-red-700);
  --color-feedback-error-subtle:  var(--color-red-50);
  --color-feedback-error-border:  var(--color-red-300);
  --color-feedback-error-text:    var(--color-red-700);

  /* --- Feedback: Warning --- */
  --color-feedback-warning:       var(--color-yellow-500);
  --color-feedback-warning-hover: var(--color-yellow-600);
  --color-feedback-warning-subtle:var(--color-yellow-50);
  --color-feedback-warning-border:var(--color-yellow-300);
  --color-feedback-warning-text:  var(--color-yellow-800);

  /* --- Feedback: Success --- */
  --color-feedback-success:       var(--color-green-600);
  --color-feedback-success-hover: var(--color-green-700);
  --color-feedback-success-subtle:var(--color-green-50);
  --color-feedback-success-border:var(--color-green-300);
  --color-feedback-success-text:  var(--color-green-700);

  /* --- Feedback: Info --- */
  --color-feedback-info:          var(--color-blue-500);
  --color-feedback-info-subtle:   var(--color-blue-50);
  --color-feedback-info-border:   var(--color-blue-200);
  --color-feedback-info-text:     var(--color-blue-700);

  /* --- Surface --- */
  --color-surface-page:           var(--color-neutral-50);
  --color-surface-default:        var(--color-neutral-0);
  --color-surface-raised:         var(--color-neutral-0);
  --color-surface-overlay:        var(--color-neutral-0);
  --color-surface-sunken:         var(--color-neutral-100);
  --color-surface-disabled:       var(--color-neutral-100);

  /* --- Text --- */
  --color-text-primary:           var(--color-neutral-900);
  --color-text-secondary:         var(--color-neutral-600);
  --color-text-tertiary:          var(--color-neutral-400);
  --color-text-disabled:          var(--color-neutral-300);
  --color-text-inverse:           var(--color-neutral-0);
  --color-text-link:              var(--color-blue-600);
  --color-text-link-hover:        var(--color-blue-700);
  --color-text-link-visited:      var(--color-purple-600);

  /* --- Border --- */
  --color-border-default:         var(--color-neutral-200);
  --color-border-strong:          var(--color-neutral-400);
  --color-border-focus:           var(--color-blue-500);
  --color-border-disabled:        var(--color-neutral-200);
  --color-border-error:           var(--color-feedback-error-border);

  /* --- Interactive --- */
  --color-interactive-primary:         var(--color-brand-primary);
  --color-interactive-primary-hover:   var(--color-brand-primary-hover);
  --color-interactive-primary-active:  var(--color-brand-primary-active);
  --color-interactive-primary-text:    var(--color-neutral-0);
  --color-interactive-secondary:       var(--color-neutral-0);
  --color-interactive-secondary-hover: var(--color-neutral-50);
  --color-interactive-secondary-border:var(--color-neutral-300);

  /* --- Overlay / Scrim --- */
  --color-overlay-scrim:          var(--color-black-a60);
}
```

---

## Semantic Tokens — Dark Mode Overrides

```css
/* tokens/semantic/_dark.css */
/* Override only the tokens that change in dark mode. */

@media (prefers-color-scheme: dark) {
  :root {
    /* --- Brand --- */
    --color-brand-primary:         var(--color-blue-400);
    --color-brand-primary-hover:   var(--color-blue-300);
    --color-brand-primary-active:  var(--color-blue-200);
    --color-brand-primary-subtle:  rgba(59, 130, 246, 0.15);   /* blue-500 @ 15% */

    /* --- Feedback: Error --- */
    --color-feedback-error:        var(--color-red-400);
    --color-feedback-error-hover:  var(--color-red-300);
    --color-feedback-error-subtle: rgba(239, 68, 68, 0.15);
    --color-feedback-error-border: var(--color-red-700);
    --color-feedback-error-text:   var(--color-red-300);

    /* --- Feedback: Warning --- */
    --color-feedback-warning:       var(--color-yellow-400);
    --color-feedback-warning-subtle:rgba(245, 158, 11, 0.15);
    --color-feedback-warning-border:var(--color-yellow-700);
    --color-feedback-warning-text:  var(--color-yellow-300);

    /* --- Feedback: Success --- */
    --color-feedback-success:       var(--color-green-400);
    --color-feedback-success-subtle:rgba(34, 197, 94, 0.15);
    --color-feedback-success-border:var(--color-green-700);
    --color-feedback-success-text:  var(--color-green-300);

    /* --- Feedback: Info --- */
    --color-feedback-info:          var(--color-blue-400);
    --color-feedback-info-subtle:   rgba(59, 130, 246, 0.15);
    --color-feedback-info-border:   var(--color-blue-700);
    --color-feedback-info-text:     var(--color-blue-300);

    /* --- Surface --- */
    --color-surface-page:          var(--color-neutral-950);
    --color-surface-default:       var(--color-neutral-900);
    --color-surface-raised:        var(--color-neutral-800);
    --color-surface-overlay:       var(--color-neutral-800);
    --color-surface-sunken:        var(--color-neutral-950);
    --color-surface-disabled:      var(--color-neutral-800);

    /* --- Text --- */
    --color-text-primary:          var(--color-neutral-50);
    --color-text-secondary:        var(--color-neutral-400);
    --color-text-tertiary:         var(--color-neutral-500);
    --color-text-disabled:         var(--color-neutral-600);
    --color-text-inverse:          var(--color-neutral-900);
    --color-text-link:             var(--color-blue-400);
    --color-text-link-hover:       var(--color-blue-300);
    --color-text-link-visited:     var(--color-purple-400);

    /* --- Border --- */
    --color-border-default:        var(--color-neutral-700);
    --color-border-strong:         var(--color-neutral-500);
    --color-border-disabled:       var(--color-neutral-700);

    /* --- Interactive --- */
    --color-interactive-primary-text:    var(--color-neutral-900);
    --color-interactive-secondary:       var(--color-neutral-800);
    --color-interactive-secondary-hover: var(--color-neutral-700);
    --color-interactive-secondary-border:var(--color-neutral-600);
  }
}

/* Explicit data-theme attribute — takes precedence over media query */
[data-theme="dark"] {
  /* Same overrides as above; allows JS-toggled dark mode to win */
  --color-brand-primary:         var(--color-blue-400);
  --color-brand-primary-hover:   var(--color-blue-300);
  --color-brand-primary-active:  var(--color-blue-200);
  --color-brand-primary-subtle:  rgba(59, 130, 246, 0.15);
  --color-surface-page:          var(--color-neutral-950);
  --color-surface-default:       var(--color-neutral-900);
  --color-surface-raised:        var(--color-neutral-800);
  --color-surface-overlay:       var(--color-neutral-800);
  --color-surface-sunken:        var(--color-neutral-950);
  --color-surface-disabled:      var(--color-neutral-800);
  --color-text-primary:          var(--color-neutral-50);
  --color-text-secondary:        var(--color-neutral-400);
  --color-text-tertiary:         var(--color-neutral-500);
  --color-text-disabled:         var(--color-neutral-600);
  --color-text-inverse:          var(--color-neutral-900);
  --color-text-link:             var(--color-blue-400);
  --color-border-default:        var(--color-neutral-700);
  --color-border-strong:         var(--color-neutral-500);
  --color-border-disabled:       var(--color-neutral-700);
  --color-feedback-error:        var(--color-red-400);
  --color-feedback-error-text:   var(--color-red-300);
  --color-feedback-success:      var(--color-green-400);
  --color-feedback-success-text: var(--color-green-300);
  --color-feedback-warning:      var(--color-yellow-400);
  --color-feedback-warning-text: var(--color-yellow-300);
  --color-feedback-info:         var(--color-blue-400);
  --color-feedback-info-text:    var(--color-blue-300);
  --color-interactive-secondary:       var(--color-neutral-800);
  --color-interactive-secondary-hover: var(--color-neutral-700);
  --color-interactive-secondary-border:var(--color-neutral-600);
}
```

---

## Component Tokens — Examples

Component tokens scope to a component namespace and reference semantic tokens. They are the knobs consumers use to customise a single component without changing anything else.

```css
/* tokens/components/_button.css */

.btn {
  /* Geometry */
  --btn-height-sm:          var(--space-8);        /* 32 px */
  --btn-height-md:          var(--space-10);       /* 40 px */
  --btn-height-lg:          var(--space-12);       /* 48 px */
  --btn-padding-x-sm:       var(--space-3);
  --btn-padding-x-md:       var(--space-4);
  --btn-padding-x-lg:       var(--space-5);
  --btn-border-radius:      var(--radius-md);

  /* Typography */
  --btn-font-size-sm:       var(--font-size-sm);
  --btn-font-size-md:       var(--font-size-base);
  --btn-font-size-lg:       var(--font-size-lg);
  --btn-font-weight:        var(--font-weight-medium);

  /* Primary variant */
  --btn-primary-bg:         var(--color-interactive-primary);
  --btn-primary-bg-hover:   var(--color-interactive-primary-hover);
  --btn-primary-bg-active:  var(--color-interactive-primary-active);
  --btn-primary-text:       var(--color-interactive-primary-text);
  --btn-primary-border:     transparent;

  /* Secondary (outlined) variant */
  --btn-secondary-bg:       var(--color-interactive-secondary);
  --btn-secondary-bg-hover: var(--color-interactive-secondary-hover);
  --btn-secondary-text:     var(--color-text-primary);
  --btn-secondary-border:   var(--color-interactive-secondary-border);

  /* Ghost variant */
  --btn-ghost-bg:           transparent;
  --btn-ghost-bg-hover:     var(--color-brand-primary-subtle);
  --btn-ghost-text:         var(--color-brand-primary);
  --btn-ghost-border:       transparent;

  /* Destructive variant */
  --btn-destructive-bg:     var(--color-feedback-error);
  --btn-destructive-bg-hover:var(--color-feedback-error-hover);
  --btn-destructive-text:   var(--color-neutral-0);
  --btn-destructive-border: transparent;

  /* States */
  --btn-disabled-opacity:   0.40;
  --btn-focus-shadow:       var(--shadow-focus);

  /* Transition */
  --btn-transition:         var(--transition-colors);
}
```

```css
/* tokens/components/_input.css */

.input {
  --input-height-sm:         var(--space-8);
  --input-height-md:         var(--space-10);
  --input-height-lg:         var(--space-12);
  --input-padding-x:         var(--space-3);
  --input-border-radius:     var(--radius-md);
  --input-border-width:      var(--border-width-thin);
  --input-font-size:         var(--font-size-base);

  --input-bg:                var(--color-surface-default);
  --input-border:            var(--color-border-default);
  --input-border-hover:      var(--color-border-strong);
  --input-border-focus:      var(--color-border-focus);
  --input-border-error:      var(--color-border-error);
  --input-text:              var(--color-text-primary);
  --input-placeholder:       var(--color-text-tertiary);
  --input-disabled-bg:       var(--color-surface-disabled);
  --input-disabled-text:     var(--color-text-disabled);
  --input-disabled-border:   var(--color-border-disabled);
}
```

```css
/* tokens/components/_badge.css */

.badge {
  --badge-padding-x:       var(--space-2);
  --badge-padding-y:       var(--space-0-5);
  --badge-border-radius:   var(--radius-full);
  --badge-font-size:       var(--font-size-xs);
  --badge-font-weight:     var(--font-weight-medium);

  /* Neutral */
  --badge-neutral-bg:    var(--color-neutral-100);
  --badge-neutral-text:  var(--color-neutral-700);

  /* Brand */
  --badge-brand-bg:      var(--color-brand-primary-subtle);
  --badge-brand-text:    var(--color-brand-primary);

  /* Success */
  --badge-success-bg:    var(--color-feedback-success-subtle);
  --badge-success-text:  var(--color-feedback-success-text);

  /* Warning */
  --badge-warning-bg:    var(--color-feedback-warning-subtle);
  --badge-warning-text:  var(--color-feedback-warning-text);

  /* Error */
  --badge-error-bg:      var(--color-feedback-error-subtle);
  --badge-error-text:    var(--color-feedback-error-text);
}
```

---

## Density Variants

Density affects spacing and font size only — never colour. Components scale by overriding their own component tokens inside a density context class.

```css
/* tokens/density/_compact.css */
/* Apply .density-compact to a container to make all children denser */

.density-compact {
  --space-scale: 0.75;

  /* Button compact overrides */
  --btn-height-sm:    var(--space-6);       /* 24 px */
  --btn-height-md:    var(--space-8);       /* 32 px */
  --btn-height-lg:    var(--space-10);      /* 40 px */
  --btn-padding-x-sm: var(--space-2);
  --btn-padding-x-md: var(--space-3);
  --btn-font-size-sm: var(--font-size-xs);
  --btn-font-size-md: var(--font-size-sm);

  /* Input compact overrides */
  --input-height-sm:  var(--space-6);
  --input-height-md:  var(--space-8);
  --input-height-lg:  var(--space-10);
}

/* tokens/density/_comfortable.css */
/* Default density — no overrides needed; tokens already set comfortable values */

/* tokens/density/_spacious.css */
.density-spacious {
  --btn-height-sm:    var(--space-10);
  --btn-height-md:    var(--space-12);
  --btn-height-lg:    var(--space-14);
  --btn-padding-x-sm: var(--space-4);
  --btn-padding-x-md: var(--space-6);
  --btn-font-size-lg: var(--font-size-xl);

  --input-height-sm:  var(--space-10);
  --input-height-md:  var(--space-12);
  --input-height-lg:  var(--space-14);
}
```

---

## Token Naming Conventions

| Tier | Pattern | Example |
|---|---|---|
| Global | `--{category}-{scale}` | `--color-blue-600`, `--space-4` |
| Semantic | `--color-{intent}-{variant}` | `--color-feedback-error`, `--color-text-secondary` |
| Component | `--{component}-{variant}-{property}` | `--btn-primary-bg-hover`, `--input-border-error` |

Rules:

- Hyphens only — no underscores, no camelCase in token names.
- Scale suffixes (50–900 for colour, 0–64 for spacing) are always numeric.
- Variant suffixes (`-hover`, `-active`, `-subtle`, `-text`, `-border`) describe the use context, not the raw value.
- Never include a hex colour or pixel value in a semantic or component token name. Name what it **means**, not what it looks like. `--color-text-danger` is wrong — use `--color-text-error` (intent) or `--color-feedback-error-text` (semantic pattern).

---

## JavaScript / TypeScript Token Map

Export tokens for use in JavaScript (for canvas drawing, charting libraries, animation engines, etc.).

```typescript
// src/design-system/tokens.ts
// Auto-generated from CSS tokens — do not edit manually.
// Regenerate with: npm run tokens:build

export const color = {
  brand: {
    primary:       "var(--color-brand-primary)",
    primaryHover:  "var(--color-brand-primary-hover)",
    primarySubtle: "var(--color-brand-primary-subtle)",
    secondary:     "var(--color-brand-secondary)",
  },
  feedback: {
    error:   "var(--color-feedback-error)",
    warning: "var(--color-feedback-warning)",
    success: "var(--color-feedback-success)",
    info:    "var(--color-feedback-info)",
  },
  surface: {
    page:    "var(--color-surface-page)",
    default: "var(--color-surface-default)",
    raised:  "var(--color-surface-raised)",
    overlay: "var(--color-surface-overlay)",
    sunken:  "var(--color-surface-sunken)",
  },
  text: {
    primary:   "var(--color-text-primary)",
    secondary: "var(--color-text-secondary)",
    tertiary:  "var(--color-text-tertiary)",
    disabled:  "var(--color-text-disabled)",
    inverse:   "var(--color-text-inverse)",
    link:      "var(--color-text-link)",
  },
  border: {
    default:  "var(--color-border-default)",
    strong:   "var(--color-border-strong)",
    focus:    "var(--color-border-focus)",
    disabled: "var(--color-border-disabled)",
    error:    "var(--color-border-error)",
  },
} as const;

export const space = {
  0:    "var(--space-0)",
  1:    "var(--space-1)",
  2:    "var(--space-2)",
  3:    "var(--space-3)",
  4:    "var(--space-4)",
  6:    "var(--space-6)",
  8:    "var(--space-8)",
  10:   "var(--space-10)",
  12:   "var(--space-12)",
  16:   "var(--space-16)",
  24:   "var(--space-24)",
} as const;

export const fontSize = {
  xs:   "var(--font-size-xs)",
  sm:   "var(--font-size-sm)",
  base: "var(--font-size-base)",
  lg:   "var(--font-size-lg)",
  xl:   "var(--font-size-xl)",
  "2xl":"var(--font-size-2xl)",
  "3xl":"var(--font-size-3xl)",
  "4xl":"var(--font-size-4xl)",
} as const;

export const radius = {
  none: "var(--radius-none)",
  sm:   "var(--radius-sm)",
  base: "var(--radius-base)",
  md:   "var(--radius-md)",
  lg:   "var(--radius-lg)",
  xl:   "var(--radius-xl)",
  full: "var(--radius-full)",
} as const;

export const shadow = {
  none: "var(--shadow-none)",
  xs:   "var(--shadow-xs)",
  sm:   "var(--shadow-sm)",
  md:   "var(--shadow-md)",
  lg:   "var(--shadow-lg)",
  xl:   "var(--shadow-xl)",
  focus:"var(--shadow-focus)",
} as const;

export const duration = {
  instant:  "var(--duration-instant)",
  fast:     "var(--duration-fast)",
  normal:   "var(--duration-normal)",
  moderate: "var(--duration-moderate)",
  slow:     "var(--duration-slow)",
} as const;

export const zIndex = {
  base:     "var(--z-index-base)",
  raised:   "var(--z-index-raised)",
  dropdown: "var(--z-index-dropdown)",
  sticky:   "var(--z-index-sticky)",
  overlay:  "var(--z-index-overlay)",
  modal:    "var(--z-index-modal)",
  toast:    "var(--z-index-toast)",
} as const;

// Resolve a CSS variable to its computed value at runtime (client only)
export function resolveToken(token: string): string {
  return getComputedStyle(document.documentElement)
    .getPropertyValue(token.replace(/^var\((.+)\)$/, "$1"))
    .trim();
}
```
