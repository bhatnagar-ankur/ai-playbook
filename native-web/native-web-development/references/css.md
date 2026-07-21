# CSS — Full Reference

Complete CSS patterns: reset, design tokens, BEM, Grid, Flexbox, animations, responsive design,
CSS Modules, and SCSS.
For the overview and quick rules see the **CSS Architecture** section in `SKILL.md`.

---

## Table of Contents
1. [CSS Reset](#css-reset)
2. [Design Tokens (Custom Properties)](#design-tokens-custom-properties)
3. [Base Styles](#base-styles)
4. [BEM — Full Patterns](#bem--full-patterns)
5. [Grid Layouts](#grid-layouts)
6. [Flexbox Patterns](#flexbox-patterns)
7. [Animations and Transitions](#animations-and-transitions)
8. [Responsive Design](#responsive-design)
9. [Container Queries](#container-queries)
10. [CSS Modules (Vite)](#css-modules-vite)
11. [SCSS with @use](#scss-with-use)
12. [Component CSS Examples](#component-css-examples)

---

## CSS Reset

```css
/* styles/reset.css */

*,
*::before,
*::after {
  box-sizing:    border-box;
  margin:        0;
  padding:       0;
}

html {
  font-size:              100%;   /* 16px base */
  -webkit-text-size-adjust: 100%;
  scroll-behavior:        smooth;
}

body {
  min-height:   100dvh;
  line-height:  1.5;
  -webkit-font-smoothing:  antialiased;
  -moz-osx-font-smoothing: grayscale;
}

img,
picture,
video,
canvas,
svg {
  display:   block;
  max-width: 100%;
}

input,
button,
textarea,
select {
  font:         inherit;
  color:        inherit;
  background:   transparent;
  border:       none;
  outline:      none;
}

button {
  cursor: pointer;
}

p,
h1, h2, h3, h4, h5, h6 {
  overflow-wrap: break-word;
}

/* Remove list styling for lists with role="list" (used in nav) */
ul[role='list'],
ol[role='list'] {
  list-style: none;
}

/* Remove animations for users who prefer reduced motion                          */
/* !important is required here to override inline or component-level animation    */
/* values set at higher specificity — this is the ONLY legitimate use of          */
/* !important in this codebase. Never use it elsewhere.                           */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration:        0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration:       0.01ms !important;
    scroll-behavior:           auto !important;
  }
}
```

---

## Design Tokens (Custom Properties)

```css
/* styles/tokens.css */

:root {
  /* ── Colours ───────────────────────────────────── */
  --color-primary:       #1a56db;
  --color-primary-dark:  #1e40af;
  --color-primary-light: #e8f0fe;

  --color-danger:        #e02424;
  --color-danger-light:  #fee2e2;
  --color-success:       #057a55;
  --color-success-light: #d1fae5;
  --color-warning:       #d97706;
  --color-warning-light: #fef3c7;
  --color-info:          #0284c7;
  --color-info-light:    #e0f2fe;

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

  /* ── Spacing ───────────────────────────────────── */
  --space-px: 1px;
  --space-0:  0;
  --space-1:  0.25rem;   /*  4px */
  --space-2:  0.5rem;    /*  8px */
  --space-3:  0.75rem;   /* 12px */
  --space-4:  1rem;      /* 16px */
  --space-5:  1.25rem;   /* 20px */
  --space-6:  1.5rem;    /* 24px */
  --space-8:  2rem;      /* 32px */
  --space-10: 2.5rem;    /* 40px */
  --space-12: 3rem;      /* 48px */
  --space-16: 4rem;      /* 64px */
  --space-20: 5rem;      /* 80px */
  --space-24: 6rem;      /* 96px */

  /* ── Typography ────────────────────────────────── */
  --font-family-base:  'Inter', system-ui, -apple-system, sans-serif;
  --font-family-mono:  'JetBrains Mono', 'Fira Code', monospace;

  --font-size-xs:   0.75rem;    /* 12px */
  --font-size-sm:   0.875rem;   /* 14px */
  --font-size-base: 1rem;       /* 16px */
  --font-size-lg:   1.125rem;   /* 18px */
  --font-size-xl:   1.25rem;    /* 20px */
  --font-size-2xl:  1.5rem;     /* 24px */
  --font-size-3xl:  1.875rem;   /* 30px */
  --font-size-4xl:  2.25rem;    /* 36px */

  --font-weight-normal:   400;
  --font-weight-medium:   500;
  --font-weight-semibold: 600;
  --font-weight-bold:     700;

  --line-height-tight:  1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;

  /* ── Border radius ─────────────────────────────── */
  --radius-none: 0;
  --radius-sm:   0.125rem;
  --radius-md:   0.375rem;
  --radius-lg:   0.5rem;
  --radius-xl:   0.75rem;
  --radius-2xl:  1rem;
  --radius-full: 9999px;

  /* ── Shadows ───────────────────────────────────── */
  --shadow-xs: 0 1px 2px rgb(0 0 0 / 0.05);
  --shadow-sm: 0 1px 3px rgb(0 0 0 / 0.1), 0 1px 2px rgb(0 0 0 / 0.06);
  --shadow-md: 0 4px 6px rgb(0 0 0 / 0.07), 0 2px 4px rgb(0 0 0 / 0.06);
  --shadow-lg: 0 10px 15px rgb(0 0 0 / 0.1), 0 4px 6px rgb(0 0 0 / 0.05);
  --shadow-xl: 0 20px 25px rgb(0 0 0 / 0.1), 0 10px 10px rgb(0 0 0 / 0.04);

  /* ── Z-index scale ─────────────────────────────── */
  --z-below:    -1;
  --z-base:      0;
  --z-raised:   10;
  --z-dropdown: 100;
  --z-sticky:   200;
  --z-overlay:  300;
  --z-modal:    400;
  --z-toast:    500;

  /* ── Transitions ───────────────────────────────── */
  --transition-fast:   150ms cubic-bezier(0.4, 0, 0.2, 1);
  --transition-normal: 250ms cubic-bezier(0.4, 0, 0.2, 1);
  --transition-slow:   350ms cubic-bezier(0.4, 0, 0.2, 1);

  /* ── Breakpoints (reference only — use in media queries) ── */
  /* sm: 640px | md: 768px | lg: 1024px | xl: 1280px | 2xl: 1536px */
}
```

---

## Base Styles

```css
/* styles/base.css */

@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

body {
  font-family: var(--font-family-base);
  font-size:   var(--font-size-base);
  font-weight: var(--font-weight-normal);
  line-height: var(--line-height-normal);
  color:       var(--color-neutral-900);
  background:  var(--color-neutral-50);
}

/* Heading scale */
h1 { font-size: var(--font-size-4xl); font-weight: var(--font-weight-bold);     line-height: var(--line-height-tight); }
h2 { font-size: var(--font-size-3xl); font-weight: var(--font-weight-bold);     line-height: var(--line-height-tight); }
h3 { font-size: var(--font-size-2xl); font-weight: var(--font-weight-semibold); line-height: var(--line-height-tight); }
h4 { font-size: var(--font-size-xl);  font-weight: var(--font-weight-semibold); }
h5 { font-size: var(--font-size-lg);  font-weight: var(--font-weight-medium);   }
h6 { font-size: var(--font-size-base);font-weight: var(--font-weight-medium);   }

/* Links */
a {
  color:            var(--color-primary);
  text-decoration:  underline;
  text-underline-offset: 3px;
}
a:hover  { color: var(--color-primary-dark); }
a:focus-visible {
  outline:        2px solid var(--color-primary);
  outline-offset: 3px;
  border-radius:  var(--radius-sm);
}

/* Focus visible — applies to all interactive elements */
:focus-visible {
  outline:        2px solid var(--color-primary);
  outline-offset: 3px;
  border-radius:  var(--radius-sm);
}

/* Utility classes */
.sr-only {
  position:    absolute;
  width:       1px;
  height:      1px;
  padding:     0;
  margin:      -1px;
  overflow:    hidden;
  clip:        rect(0, 0, 0, 0);
  white-space: nowrap;
  border:      0;
}

.skip-link {
  position:        absolute;
  top:             -100%;
  left:            var(--space-4);
  padding:         var(--space-2) var(--space-4);
  background:      var(--color-primary);
  color:           #fff;
  border-radius:   var(--radius-md);
  text-decoration: none;
  font-weight:     var(--font-weight-semibold);
  z-index:         var(--z-toast);
}
.skip-link:focus { top: var(--space-4); }
```

---

## BEM — Full Patterns

**Rules:** class names are lowercase-hyphen (`order-card`), one block per file (filename = block name), never use IDs as CSS selectors, no descendant selectors crossing block boundaries (`.order-card .btn` — wrong), never use `!important` — fix specificity with a more targeted selector instead.

### Button block

```css
/* styles/components/btn.css */

.btn {
  display:         inline-flex;
  align-items:     center;
  justify-content: center;
  gap:             var(--space-2);
  padding:         var(--space-2) var(--space-4);
  border:          2px solid transparent;
  border-radius:   var(--radius-md);
  font-size:       var(--font-size-sm);
  font-weight:     var(--font-weight-semibold);
  line-height:     1;
  text-decoration: none;
  cursor:          pointer;
  transition:      background var(--transition-fast),
                   border-color var(--transition-fast),
                   color var(--transition-fast),
                   box-shadow var(--transition-fast);
  white-space:     nowrap;
  user-select:     none;
}

.btn:focus-visible {
  outline:        2px solid var(--color-primary);
  outline-offset: 3px;
}

/* Modifiers */
.btn--primary {
  background:   var(--color-primary);
  border-color: var(--color-primary);
  color:        #fff;
}
.btn--primary:hover:not(:disabled) {
  background:   var(--color-primary-dark);
  border-color: var(--color-primary-dark);
}

.btn--secondary {
  background:   transparent;
  border-color: var(--color-neutral-300);
  color:        var(--color-neutral-700);
}
.btn--secondary:hover:not(:disabled) {
  background:   var(--color-neutral-100);
  border-color: var(--color-neutral-400);
}

.btn--danger {
  background:   var(--color-danger);
  border-color: var(--color-danger);
  color:        #fff;
}
.btn--danger:hover:not(:disabled) {
  background:   #c81e1e;
  border-color: #c81e1e;
}

.btn--ghost {
  background:   transparent;
  border-color: transparent;
  color:        var(--color-primary);
}
.btn--ghost:hover:not(:disabled) {
  background: var(--color-primary-light);
}

/* Size modifiers */
.btn--sm { padding: var(--space-1) var(--space-3); font-size: var(--font-size-xs); }
.btn--lg { padding: var(--space-3) var(--space-6); font-size: var(--font-size-base); }

/* Icon-only button */
.btn--icon {
  padding:       var(--space-2);
  border-radius: var(--radius-full);
  color:         var(--color-neutral-600);
}
.btn--icon:hover:not(:disabled) {
  background: var(--color-neutral-100);
  color:      var(--color-neutral-900);
}

/* State */
.btn:disabled,
.btn[aria-busy='true'] {
  opacity: 0.5;
  cursor:  not-allowed;
  pointer-events: none;
}
```

### Form field block

```css
/* styles/components/form-field.css */

.form-field {
  display:        flex;
  flex-direction: column;
  gap:            var(--space-1);
}

.form-field + .form-field {
  margin-top: var(--space-4);
}

.form-field__label {
  font-size:   var(--font-size-sm);
  font-weight: var(--font-weight-medium);
  color:       var(--color-neutral-700);
}

.form-field__required-marker {
  color:       var(--color-danger);
  margin-left: var(--space-1);
}

.form-field__input,
.form-field__select,
.form-field__textarea {
  width:         100%;
  padding:       var(--space-2) var(--space-3);
  border:        1px solid var(--color-neutral-300);
  border-radius: var(--radius-md);
  font-size:     var(--font-size-base);
  color:         var(--color-neutral-900);
  background:    #fff;
  transition:    border-color var(--transition-fast), box-shadow var(--transition-fast);
}

.form-field__input:focus,
.form-field__select:focus,
.form-field__textarea:focus {
  border-color: var(--color-primary);
  box-shadow:   0 0 0 3px rgb(26 86 219 / 0.15);
  outline:      none;
}

/* Error state — applied by JS via showFieldError() */
.form-field__input[aria-invalid='true'],
.form-field__select[aria-invalid='true'],
.form-field__textarea[aria-invalid='true'] {
  border-color: var(--color-danger);
}

.form-field__input[aria-invalid='true']:focus,
.form-field__select[aria-invalid='true']:focus {
  box-shadow: 0 0 0 3px rgb(224 36 36 / 0.15);
}

.form-field__hint {
  font-size: var(--font-size-xs);
  color:     var(--color-neutral-500);
}

.form-field__error {
  font-size: var(--font-size-xs);
  color:     var(--color-danger);
  font-weight: var(--font-weight-medium);
}

.form-field__checkbox-label {
  display:     flex;
  align-items: center;
  gap:         var(--space-2);
  font-size:   var(--font-size-sm);
  cursor:      pointer;
}

.form-actions {
  display:     flex;
  gap:         var(--space-3);
  margin-top:  var(--space-6);
  padding-top: var(--space-4);
  border-top:  1px solid var(--color-neutral-200);
}
```

---

## Grid Layouts

```css
/* Auto-fill responsive grid — cards grow to fill available space */
.cards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(18rem, 1fr));
  gap:     var(--space-4);
}

/* Fixed column grid */
.three-column-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap:     var(--space-6);
}

@media (max-width: 768px) {
  .three-column-grid {
    grid-template-columns: 1fr;
  }
}

/* Named area layout — dashboard with sidebar */
.dashboard-layout {
  display: grid;
  grid-template-areas:
    'header  header'
    'sidebar content'
    'footer  footer';
  grid-template-columns: 16rem 1fr;
  grid-template-rows:    auto 1fr auto;
  min-height: 100dvh;
}

.dashboard-layout__header  { grid-area: header; }
.dashboard-layout__sidebar { grid-area: sidebar; }
.dashboard-layout__content { grid-area: content; }
.dashboard-layout__footer  { grid-area: footer; }

@media (max-width: 1024px) {
  .dashboard-layout {
    grid-template-areas:
      'header'
      'content'
      'footer';
    grid-template-columns: 1fr;
  }
  .dashboard-layout__sidebar { display: none; }
}

/* Subgrid — align children across parent tracks */
.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap:     var(--space-4);
}

.form-grid__full-width {
  grid-column: 1 / -1;   /* Span all columns */
}
```

---

## Flexbox Patterns

```css
/* Navigation bar */
.nav-bar {
  display:         flex;
  align-items:     center;
  justify-content: space-between;
  gap:             var(--space-4);
  padding:         var(--space-3) var(--space-6);
  background:      #fff;
  border-bottom:   1px solid var(--color-neutral-200);
}

.nav-bar__links {
  display:     flex;
  align-items: center;
  gap:         var(--space-1);
  list-style:  none;
}

/* Card internal layout */
.order-card {
  display:        flex;
  flex-direction: column;
  gap:            var(--space-3);
}

.order-card__header {
  display:         flex;
  align-items:     flex-start;
  justify-content: space-between;
  gap:             var(--space-2);
}

.order-card__footer {
  display:     flex;
  align-items: center;
  gap:         var(--space-2);
  margin-top:  auto;   /* Push footer to bottom of card */
}

/* Centered content */
.page-center {
  display:         flex;
  align-items:     center;
  justify-content: center;
  min-height:      100dvh;
}

/* Equal-height columns */
.feature-row {
  display: flex;
  gap:     var(--space-6);
}

.feature-row__item {
  flex:    1;
  display: flex;
  flex-direction: column;
}

/* Stretch last child to fill */
.feature-row__item p:last-child { margin-top: auto; }
```

---

## Animations and Transitions

```css
/* ── Transition utilities ────────────────────────── */

.transition-colors {
  transition: color var(--transition-fast),
              background-color var(--transition-fast),
              border-color var(--transition-fast);
}

.transition-transform {
  transition: transform var(--transition-normal);
}

.transition-opacity {
  transition: opacity var(--transition-normal);
}

.transition-shadow {
  transition: box-shadow var(--transition-fast);
}

/* ── Keyframe animations ─────────────────────────── */

@keyframes fade-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}

@keyframes slide-in-up {
  from { opacity: 0; transform: translateY(var(--space-4)); }
  to   { opacity: 1; transform: translateY(0); }
}

@keyframes slide-in-right {
  from { opacity: 0; transform: translateX(var(--space-8)); }
  to   { opacity: 1; transform: translateX(0); }
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.5; }
}

/* ── Animation classes ───────────────────────────── */

.animate-fade-in {
  animation: fade-in var(--transition-normal) ease both;
}

.animate-slide-up {
  animation: slide-in-up var(--transition-normal) ease both;
}

.animate-spin {
  animation: spin 1s linear infinite;
}

.animate-pulse {
  animation: pulse 2s ease-in-out infinite;
}

/* ── Loading spinner ─────────────────────────────── */

.spinner {
  width:          1.5rem;
  height:         1.5rem;
  border:         2px solid var(--color-neutral-200);
  border-top-color: var(--color-primary);
  border-radius:  var(--radius-full);
  animation:      spin 0.6s linear infinite;
}

/* ── Skeleton loader ─────────────────────────────── */

@keyframes shimmer {
  from { background-position: -200% center; }
  to   { background-position:  200% center; }
}

.skeleton {
  background:           linear-gradient(90deg, var(--color-neutral-200) 25%, var(--color-neutral-100) 50%, var(--color-neutral-200) 75%);
  background-size:      200% auto;
  animation:            shimmer 1.5s ease-in-out infinite;
  border-radius:        var(--radius-md);
}

.order-card-skeleton {
  height:        7rem;
  border-radius: var(--radius-md);
}
```

---

## Responsive Design

```css
/* styles/breakpoints.css — reference (not imported; use values inline) */

/*
  Breakpoints:
  sm:  640px
  md:  768px
  lg:  1024px
  xl:  1280px
  2xl: 1536px

  Approach: Mobile-first — write base styles for mobile, add breakpoints upward.
*/

/* Mobile-first example */
.orders-grid {
  display: grid;
  grid-template-columns: 1fr;           /* Mobile: single column */
  gap:     var(--space-4);
}

@media (min-width: 640px) {
  .orders-grid {
    grid-template-columns: repeat(2, 1fr);   /* Tablet: 2 columns */
  }
}

@media (min-width: 1024px) {
  .orders-grid {
    grid-template-columns: repeat(3, 1fr);   /* Desktop: 3 columns */
  }
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  :root {
    --color-neutral-50:  #111827;
    --color-neutral-100: #1f2937;
    --color-neutral-900: #f9fafb;
    /* Override other tokens as needed */
  }
}

/* Print */
@media print {
  .site-header,
  .site-footer,
  .btn,
  .skip-link { display: none; }

  body { font-size: 12pt; color: #000; background: #fff; }
}
```

---

## Container Queries

Use container queries to style components based on their parent's size, not the viewport.

```css
/* Define a containment context on the parent */
.orders-section {
  container-type: inline-size;
  container-name: orders-section;
}

/* Style the child based on the container's width */
@container orders-section (min-width: 600px) {
  .order-card {
    flex-direction: row;
    align-items:    center;
  }

  .order-card__actions {
    margin-left: auto;
    flex-shrink: 0;
  }
}

/* Named containers allow targeting specific ancestors */
@container orders-section (min-width: 900px) {
  .order-card {
    grid-template-columns: 2fr 1fr 1fr auto;
  }
}
```

---

## CSS Modules (Vite)

CSS Modules scope class names locally. The build tool transforms `.class` to `._class_hash`.

```css
/* src/components/order-card/order-card.module.css */

/* Local class names — safe to use generic names without BEM */

.card {
  display:       flex;
  flex-direction: column;
  gap:           var(--space-3);
  padding:       var(--space-4);
  border:        1px solid var(--color-neutral-200);
  border-radius: var(--radius-lg);
  background:    #fff;
  box-shadow:    var(--shadow-xs);
  transition:    box-shadow var(--transition-fast),
                 transform  var(--transition-fast);
}

.card:hover {
  box-shadow:  var(--shadow-md);
  transform:   translateY(-2px);
}

/* Status variants — applied dynamically in JS */
.pending   { border-left: 4px solid var(--color-warning); }
.shipped   { border-left: 4px solid var(--color-success); }
.delivered { border-left: 4px solid var(--color-info); }
.cancelled { border-left: 4px solid var(--color-danger); }

.header {
  display:         flex;
  align-items:     flex-start;
  justify-content: space-between;
}

.orderId {
  font-size:   var(--font-size-base);
  font-weight: var(--font-weight-semibold);
  color:       var(--color-neutral-900);
  margin:      0;
}

.status {
  font-size:  var(--font-size-xs);
  color:      var(--color-neutral-500);
  margin-top: var(--space-1);
}

.amount {
  font-size:   var(--font-size-xl);
  font-weight: var(--font-weight-bold);
  color:       var(--color-neutral-900);
}

.actions {
  display:    flex;
  gap:        var(--space-2);
  margin-top: auto;
}
```

```javascript
// src/components/order-card/order-card.component.js

import styles from './order-card.module.css';

/**
 * @typedef {{ orderId: string; status: string; totalAmount: number }} Order
 */

/**
 * Creates an order card using CSS Module scoped classes.
 * @param {Order} order
 * @returns {HTMLElement}
 */
export function createOrderCard(order) {
  const statusKey   = order.status.toLowerCase();
  const statusClass = styles[statusKey] ?? '';

  const card       = document.createElement('article');
  card.className   = `${styles.card} ${statusClass}`.trim();
  card.setAttribute('aria-label', `Order ${order.orderId}, status: ${order.status}`);

  const header   = document.createElement('header');
  header.className = styles.header;

  const title    = document.createElement('h3');
  title.className  = styles.orderId;
  title.textContent = order.orderId;

  const status   = document.createElement('p');
  status.className  = styles.status;
  status.textContent = order.status;

  const amount   = document.createElement('p');
  amount.className  = styles.amount;
  amount.textContent = `$${order.totalAmount.toFixed(2)}`;

  header.append(title, status);
  card.append(header, amount);

  return card;
}
```

---

## SCSS with @use

```bash
# Install Dart Sass as dev dependency
npm install -D sass
```

```scss
/* src/styles/_tokens.scss */

// Colour palette
$color-primary:       #1a56db;
$color-primary-dark:  #1e40af;
$color-primary-light: #e8f0fe;
$color-danger:        #e02424;
$color-success:       #057a55;
$color-warning:       #d97706;
$color-neutral-200:   #e5e7eb;
$color-neutral-300:   #d1d5db;
$color-neutral-500:   #6b7280;
$color-neutral-600:   #4b5563;
$color-neutral-700:   #374151;
$color-neutral-900:   #111827;

// Spacing
$space-1:  0.25rem;
$space-2:  0.5rem;
$space-3:  0.75rem;
$space-4:  1rem;
$space-6:  1.5rem;
$space-8:  2rem;

// Typography
$font-size-xs:   0.75rem;
$font-size-sm:   0.875rem;
$font-size-base: 1rem;
$font-size-lg:   1.125rem;
$font-size-xl:   1.25rem;
$font-size-2xl:  1.5rem;
$font-weight-medium:   500;
$font-weight-semibold: 600;

// Radii and shadows
$radius-md:   0.375rem;
$radius-lg:   0.5rem;
$shadow-sm:   0 1px 3px rgb(0 0 0 / 0.1);
$shadow-md:   0 4px 6px rgb(0 0 0 / 0.07);

// Transitions
$transition-fast:   150ms cubic-bezier(0.4, 0, 0.2, 1);
$transition-normal: 250ms cubic-bezier(0.4, 0, 0.2, 1);
```

```scss
/* src/styles/_mixins.scss */

@use 'tokens' as t;

/// Responsive breakpoint mixin.
/// @param {string} $breakpoint - sm | md | lg | xl
@mixin respond-to($breakpoint) {
  $breakpoints: (
    'sm':  640px,
    'md':  768px,
    'lg': 1024px,
    'xl': 1280px,
  );

  $value: map-get($breakpoints, $breakpoint);

  @if $value {
    @media (min-width: $value) { @content; }
  } @else {
    @warn 'Unknown breakpoint: #{$breakpoint}';
  }
}

/// Visually hides an element while keeping it accessible to screen readers.
@mixin sr-only {
  position:    absolute;
  width:       1px;
  height:      1px;
  padding:     0;
  margin:      -1px;
  overflow:    hidden;
  clip:        rect(0, 0, 0, 0);
  white-space: nowrap;
  border:      0;
}

/// Truncate text with an ellipsis.
@mixin text-truncate {
  overflow:      hidden;
  text-overflow: ellipsis;
  white-space:   nowrap;
}

/// Focus ring style for interactive elements.
@mixin focus-ring($color: t.$color-primary) {
  outline:        2px solid $color;
  outline-offset: 3px;
  border-radius:  t.$radius-md;
}
```

```scss
/* src/styles/components/_order-card.scss */

@use '../tokens' as t;
@use '../mixins' as m;

.order-card {
  display:        flex;
  flex-direction: column;
  gap:            t.$space-3;
  padding:        t.$space-4;
  border:         1px solid t.$color-neutral-200;
  border-radius:  t.$radius-lg;
  background:     #fff;
  box-shadow:     t.$shadow-sm;
  transition:     box-shadow t.$transition-fast,
                  transform  t.$transition-fast;

  &:hover {
    box-shadow: t.$shadow-md;
    transform:  translateY(-2px);
  }

  &:focus-visible { @include m.focus-ring; }

  // ── Elements ──────────────────────────────────

  &__header {
    display:         flex;
    align-items:     flex-start;
    justify-content: space-between;
    gap:             t.$space-2;
  }

  &__id {
    font-size:   t.$font-size-base;
    font-weight: t.$font-weight-semibold;
    color:       t.$color-neutral-900;
    margin:      0;
    @include m.text-truncate;
  }

  &__status {
    font-size:  t.$font-size-xs;
    color:      t.$color-neutral-500;
    margin-top: t.$space-1;
  }

  &__amount {
    font-size:   t.$font-size-xl;
    font-weight: t.$font-weight-semibold;
    color:       t.$color-neutral-900;
  }

  &__actions {
    display:    flex;
    gap:        t.$space-2;
    margin-top: auto;
  }

  // ── Modifiers ─────────────────────────────────

  &--pending   { border-left: 4px solid t.$color-warning; }
  &--shipped   { border-left: 4px solid t.$color-success; }
  &--cancelled { border-left: 4px solid t.$color-danger;  }

  // ── Responsive ────────────────────────────────

  @include m.respond-to('md') {
    flex-direction: row;
    align-items:    center;

    .order-card__actions {
      margin-top:  0;
      margin-left: auto;
      flex-shrink: 0;
    }
  }
}
```

```scss
/* src/styles/main.scss — imports all partials */

@use 'tokens';
@use 'mixins';
@use 'components/order-card';
@use 'components/btn';
@use 'components/form-field';
```

---

## Component CSS Examples

### Status badge

```css
/* styles/components/status-badge.css */

.status-badge {
  display:         inline-flex;
  align-items:     center;
  gap:             var(--space-1);
  padding:         var(--space-1) var(--space-3);
  border-radius:   var(--radius-full);
  font-size:       var(--font-size-xs);
  font-weight:     var(--font-weight-semibold);
  text-transform:  uppercase;
  letter-spacing:  0.05em;
  white-space:     nowrap;
}

.status-badge--pending   { background: var(--color-warning-light); color: #92400e; }
.status-badge--shipped   { background: var(--color-success-light); color: #065f46; }
.status-badge--delivered { background: var(--color-info-light);    color: #075985; }
.status-badge--cancelled { background: var(--color-danger-light);  color: #991b1b; }
```

### Modal

```css
/* styles/components/modal.css */

dialog.modal {
  position:      fixed;
  inset:         0;
  margin:        auto;
  padding:       0;
  border:        none;
  border-radius: var(--radius-xl);
  box-shadow:    var(--shadow-xl);
  max-width:     min(90vw, 32rem);
  width:         100%;
  max-height:    90dvh;
  overflow:      hidden;
  animation:     slide-in-up var(--transition-normal) ease both;
}

dialog.modal::backdrop {
  background:  rgb(0 0 0 / 0.5);
  backdrop-filter: blur(2px);
  animation:   fade-in var(--transition-normal) ease both;
}

.modal__content {
  display:        flex;
  flex-direction: column;
  max-height:     90dvh;
  overflow:       hidden;
}

.modal__header {
  display:         flex;
  align-items:     center;
  justify-content: space-between;
  padding:         var(--space-4) var(--space-6);
  border-bottom:   1px solid var(--color-neutral-200);
}

.modal__title {
  font-size:   var(--font-size-xl);
  font-weight: var(--font-weight-semibold);
  margin:      0;
}

.modal__body {
  padding:    var(--space-6);
  overflow-y: auto;
  flex:       1;
}

.modal__footer {
  display:      flex;
  gap:          var(--space-3);
  padding:      var(--space-4) var(--space-6);
  border-top:   1px solid var(--color-neutral-200);
  justify-content: flex-end;
}
```
