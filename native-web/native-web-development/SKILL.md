---
author: Ankur Bhatnagar
version: 1.0.0
technology: native-web
compatibility: ES2020+ | Evergreen browsers | Vite 5+
---

# Native Web Development Skill

Opinionated standards for building web applications with plain HTML, CSS, and JavaScript (ES2020+).
Covers both **pure-browser** (no build step) and **Vite-based** projects, BEM CSS architecture,
CSS custom properties, Web Components, and the Fetch API.

No framework. No TypeScript. No external runtime dependencies unless explicitly added.

---

## Table of Contents
1. [Project Structure](#project-structure)
2. [HTML Conventions](#html-conventions)
3. [CSS Architecture](#css-architecture)
4. [JavaScript Conventions](#javascript-conventions)
5. [Web Components](#web-components)
6. [State Management](#state-management)
7. [HTTP and Fetch API](#http-and-fetch-api)
8. [Accessibility (WCAG AA)](#accessibility-wcag-aa)
9. [Performance](#performance)
10. [CSS Modules (Vite)](#css-modules-vite)
11. [SCSS (Vite)](#scss-vite)
12. [Naming Conventions](#naming-conventions)
13. [Code Quality Rules](#code-quality-rules)
14. [Quick Reference](#quick-reference)

---

## 1. Project Structure

Two project modes are supported. Choose based on build complexity.

### Pure Browser (no build step)

```
my-app/
├── index.html
├── styles/
│   ├── reset.css
│   ├── tokens.css                  # Design tokens — CSS custom properties
│   ├── base.css                    # Typography and body defaults
│   └── components/
│       └── order-card.css          # BEM component styles
├── scripts/
│   ├── main.js                     # Entry point — loaded as type="module"
│   ├── state/
│   │   └── orders.state.js         # Module-level state singleton
│   ├── services/
│   │   └── orders.service.js       # Fetch-based data service
│   ├── components/
│   │   └── order-card.component.js # Web Component definition
│   └── utils/
│       ├── http.utils.js
│       ├── dom.utils.js
│       └── form.utils.js
└── assets/
    └── images/
```

### Vite Project (with build step)

```
my-app/
├── index.html                      # Vite entry — script via <script type="module">
├── vite.config.js
├── package.json
├── public/                         # Copied as-is to dist/
│   └── favicon.svg
└── src/
    ├── main.js                     # Application entry point
    ├── styles/
    │   ├── reset.css
    │   ├── tokens.css
    │   └── base.css
    ├── components/
    │   └── order-card/
    │       ├── order-card.component.js
    │       └── order-card.module.css   # Scoped CSS Module
    ├── state/
    │   └── orders.state.js
    ├── services/
    │   └── orders.service.js
    ├── utils/
    │   ├── http.utils.js
    │   └── dom.utils.js
    └── models/
        └── order.model.js          # JSDoc type definitions
```

### Vite configuration

```javascript
// vite.config.js

import { defineConfig } from 'vite';

export default defineConfig({
  root: '.',
  build: {
    outDir:      'dist',
    emptyOutDir: true,
    target:      'es2020',
  },
  css: {
    modules: {
      localsConvention: 'camelCase',   // card__title → cardTitle in JS
    },
  },
});
```

---

## 2. HTML Conventions

- Use semantic elements: `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, `<footer>`
- Every `<form>` input must have an associated `<label>` — never use `placeholder` as a substitute
- Use `<button type="button">` for non-submit actions; always specify the `type` attribute
- Images must have `alt` text; decorative images use `alt=""`
- Headings follow strict hierarchy — one `<h1>` per page, no skipped levels
- Use `<dialog>` for modals; avoid custom ARIA-heavy implementations when native HTML suffices
- Never nest interactive elements (button inside anchor, anchor inside button)
- Load scripts with `<script type="module">` — deferred by default

```html
<!-- index.html — document shell -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta name="description" content="Manage your orders and track shipments." />
  <title>Orders | App Name</title>
  <link rel="stylesheet" href="/styles/reset.css" />
  <link rel="stylesheet" href="/styles/tokens.css" />
  <link rel="stylesheet" href="/styles/base.css" />
</head>
<body>
  <a href="#main-content" class="skip-link">Skip to main content</a>

  <header class="site-header" role="banner">
    <nav aria-label="Primary navigation">
      <ul role="list">
        <li><a href="/">Home</a></li>
        <li><a href="/orders" aria-current="page">Orders</a></li>
      </ul>
    </nav>
  </header>

  <main id="main-content" tabindex="-1">
    <!-- Page content -->
  </main>

  <footer class="site-footer" role="contentinfo">
    <p>&copy; 2025 App Name</p>
  </footer>

  <script type="module" src="/scripts/main.js"></script>
</body>
</html>
```

For full HTML patterns (forms, accessibility, `<template>`, `<dialog>`) see `references/html.md`.

---

## 3. CSS Architecture

### Design tokens (custom properties)

All design decisions live in `tokens.css`. Never hardcode colours, spacing, or font values anywhere else.

```css
/* styles/tokens.css */

:root {
  /* Colours */
  --color-primary:       #1a56db;
  --color-primary-hover: #1e40af;
  --color-danger:        #e02424;
  --color-success:       #057a55;
  --color-warning:       #d97706;
  --color-neutral-100:   #f9fafb;
  --color-neutral-300:   #d1d5db;
  --color-neutral-600:   #4b5563;
  --color-neutral-900:   #111827;

  /* Spacing scale */
  --space-1:  0.25rem;
  --space-2:  0.5rem;
  --space-3:  0.75rem;
  --space-4:  1rem;
  --space-6:  1.5rem;
  --space-8:  2rem;
  --space-12: 3rem;
  --space-16: 4rem;

  /* Typography */
  --font-family-base: 'Inter', system-ui, sans-serif;
  --font-size-xs:   0.75rem;
  --font-size-sm:   0.875rem;
  --font-size-base: 1rem;
  --font-size-lg:   1.125rem;
  --font-size-xl:   1.25rem;
  --font-size-2xl:  1.5rem;
  --font-weight-normal:   400;
  --font-weight-medium:   500;
  --font-weight-semibold: 600;
  --font-weight-bold:     700;

  /* Border radius */
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 1rem;
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px rgb(0 0 0 / 0.1);

  /* Transitions */
  --transition-fast:   150ms ease;
  --transition-normal: 250ms ease;
}
```

### BEM naming

Block\_\_Element--Modifier. No nesting beyond BEM depth.

```css
/* Block */
.order-card { }

/* Elements */
.order-card__header { }
.order-card__title  { }
.order-card__status { }
.order-card__amount { }
.order-card__actions { }

/* Modifiers — always applied alongside the base block class */
.order-card--pending   { }
.order-card--shipped   { }
.order-card--cancelled { }
```

**BEM rules:**
- All class names are lowercase with hyphens (`order-card`, not `orderCard`)
- One block per CSS file, filename matches the block name (`order-card.css`)
- Never use IDs as CSS selectors
- Avoid descendant selectors that cross block boundaries (`.order-card .btn` — wrong)
- Apply modifiers alongside the base class: `class="order-card order-card--pending"`
- Never use `!important` — if you need it to win specificity, the selector is wrong; use a more specific selector or restructure the rule instead. The only accepted exception is the `prefers-reduced-motion` browser reset.

### Layout: Grid and Flexbox

```css
/* Grid — two-dimensional page layouts */
.orders-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(18rem, 1fr));
  gap:     var(--space-4);
  padding: var(--space-4);
}

/* Flexbox — one-dimensional component internals */
.order-card__header {
  display:         flex;
  align-items:     center;
  justify-content: space-between;
  gap:             var(--space-2);
}
```

For full CSS patterns (animations, media queries, container queries, SCSS, CSS Modules) see `references/css.md`.

---

## 4. JavaScript Conventions

- **ES2020+ only** — optional chaining `?.`, nullish coalescing `??`, `Promise.allSettled`, dynamic `import()`, `BigInt`, `globalThis`
- Native ES modules: `import`/`export` in every file; no global scope communication
- `const` by default; `let` when reassignment is necessary; never `var`
- `async`/`await` for all asynchronous code — no raw `.then()` chains
- JSDoc for all exported function signatures and non-obvious types
- Named function declarations for top-level functions; arrow functions for callbacks

```javascript
// scripts/services/orders.service.js

import { http } from '../utils/http.utils.js';

const ORDERS_ENDPOINT = '/api/orders';

/**
 * @typedef {import('../models/order.model.js').Order} Order
 */

/**
 * Fetches all orders from the API.
 * @returns {Promise<Order[]>}
 */
export async function fetchOrders() {
  return http(ORDERS_ENDPOINT);
}

/**
 * Fetches a single order by ID.
 * @param {string} orderId
 * @returns {Promise<Order>}
 */
export async function fetchOrderById(orderId) {
  return http(`${ORDERS_ENDPOINT}/${orderId}`);
}

/**
 * Creates a new order.
 * @param {{ customerId: string; items: Array<{ productId: string; quantity: number }> }} payload
 * @returns {Promise<Order>}
 */
export async function createOrder(payload) {
  return http(ORDERS_ENDPOINT, {
    method: 'POST',
    body:   JSON.stringify(payload),
  });
}
```

For full JS patterns (DOM, events, event delegation, custom events, error handling) see `references/javascript.md`.

---

## 5. Web Components

Web Components are the component primitive for native web projects. Use whenever reusable,
encapsulated UI is needed.

- Extend `HTMLElement` — never extend built-in elements (Safari does not support customised built-ins)
- Always declare `static observedAttributes` for attributes that affect rendering
- Use Shadow DOM (`attachShadow({ mode: 'open' })`) for style encapsulation
- Communicate upward via `CustomEvent` — never call parent methods directly
- Clean up in `disconnectedCallback`: remove event listeners, cancel timers, abort fetches
- Use private class fields (`#field`) for internal state

```javascript
// scripts/components/status-badge.component.js

export class StatusBadgeElement extends HTMLElement {
  static observedAttributes = ['status'];

  /** @type {ShadowRoot} */
  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.#render();
  }

  /**
   * @param {string} _name
   * @param {string | null} oldValue
   * @param {string | null} newValue
   */
  attributeChangedCallback(_name, oldValue, newValue) {
    if (newValue !== oldValue) this.#render();
  }

  #render() {
    const status = this.getAttribute('status') ?? 'pending';

    this.#shadow.innerHTML = `
      <style>
        :host { display: inline-block; }
        .badge {
          display:       inline-flex;
          align-items:   center;
          padding:       0.25rem 0.625rem;
          border-radius: 9999px;
          font-size:     0.75rem;
          font-weight:   600;
          text-transform: uppercase;
          letter-spacing: 0.05em;
        }
        .badge--pending   { background: #fef3c7; color: #92400e; }
        .badge--shipped   { background: #d1fae5; color: #065f46; }
        .badge--delivered { background: #dbeafe; color: #1e40af; }
        .badge--cancelled { background: #fee2e2; color: #991b1b; }
      </style>
      <span class="badge badge--${status}" role="status">
        <slot>${status}</slot>
      </span>
    `;
  }
}

customElements.define('status-badge', StatusBadgeElement);
```

```html
<!-- Usage -->
<status-badge status="shipped">Shipped</status-badge>
<status-badge status="pending">Pending</status-badge>
```

For full Web Component patterns (named slots, templates, custom events, cleanup) see `references/web-components.md`.

---

## 6. State Management

No state library — use module-level singletons and the browser's `CustomEvent` API.

```javascript
// scripts/state/orders.state.js

/** @typedef {import('../models/order.model.js').Order} Order */

/** @type {Order[]} */
let _orders = [];

/** @type {boolean} */
let _isLoading = false;

/** @type {string | null} */
let _errorMessage = null;

/**
 * Returns a defensive copy of the current orders list.
 * @returns {Order[]}
 */
export function getOrders() {
  return structuredClone(_orders);
}

export function getIsLoading() { return _isLoading; }
export function getErrorMessage() { return _errorMessage; }

/**
 * Replaces the orders list and dispatches a state change event.
 * @param {Order[]} newOrders
 */
export function setOrders(newOrders) {
  _orders = newOrders;
  _isLoading = false;
  _errorMessage = null;
  dispatch();
}

/** @param {Order} order */
export function addOrder(order) {
  _orders = [..._orders, order];
  dispatch();
}

export function setLoading(isLoading) {
  _isLoading = isLoading;
  dispatch();
}

/** @param {string} message */
export function setError(message) {
  _errorMessage = message;
  _isLoading    = false;
  dispatch();
}

function dispatch() {
  document.dispatchEvent(
    new CustomEvent('orders:statechanged', {
      detail:  { orders: getOrders(), isLoading: _isLoading, errorMessage: _errorMessage },
      bubbles: false,
    }),
  );
}
```

```javascript
// Subscribing to state changes in a component
document.addEventListener('orders:statechanged', (event) => {
  const { orders, isLoading, errorMessage } = event.detail;
  renderOrdersList(orders, isLoading, errorMessage);
});
```

**Rules:**
- State modules live in `state/` — never in DOM attributes or global `window` properties
- State is mutated only through exported setter functions — never via direct external assignment
- `structuredClone()` for defensive copies — prevent accidental external mutation
- Components subscribe to `CustomEvent` on `document` — loose coupling, no direct references

---

## 7. HTTP and Fetch API

```javascript
// scripts/utils/http.utils.js

/**
 * Typed HTTP utility with timeout and JSON serialisation.
 *
 * @param {string} url
 * @param {RequestInit} [options]
 * @returns {Promise<unknown>}
 * @throws {Error} When the response status is not ok or the request times out
 */
export async function http(url, options = {}) {
  const controller = new AbortController();
  const timeoutId  = setTimeout(() => controller.abort(), 10_000);  // 10 s

  try {
    const response = await fetch(url, {
      ...options,
      signal:  controller.signal,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    // 204 No Content — return null instead of attempting to parse an empty body
    if (response.status === 204) return null;

    return response.json();
  } catch (fetchError) {
    if (fetchError instanceof Error && fetchError.name === 'AbortError') {
      throw new Error('Request timed out. Please try again.');
    }
    throw fetchError;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

---

## 8. Accessibility (WCAG AA)

Non-negotiable. Every page and component must pass WCAG 2.1 Level AA.

- **Keyboard navigability**: all interactive elements reachable and operable via keyboard
- **Focus visible**: never remove `:focus-visible` outline without providing an equivalent replacement
- **Colour contrast**: minimum 4.5:1 for normal text; 3:1 for large text and UI components
- **Screen reader text**: `.sr-only` utility for visually hidden but announced text
- **ARIA**: use only where native HTML semantics are insufficient
- **Error messages**: always linked to their input via `aria-describedby`
- **Loading states**: announced with `aria-live="polite"` or `aria-busy="true"`
- **Skip link**: first focusable element on every page, targets `#main-content`

```css
/* Utility: visually hidden but screen-reader accessible */
.sr-only {
  position:   absolute;
  width:      1px;
  height:     1px;
  padding:    0;
  margin:     -1px;
  overflow:   hidden;
  clip:       rect(0, 0, 0, 0);
  white-space: nowrap;
  border:     0;
}

/* Skip link — visible on focus only */
.skip-link {
  position:   absolute;
  top:        -100%;
  left:       var(--space-4);
  padding:    var(--space-2) var(--space-4);
  background: var(--color-primary);
  color:      #fff;
  border-radius: var(--radius-md);
  text-decoration: none;
  font-weight: var(--font-weight-semibold);
  z-index:    100;
}
.skip-link:focus {
  top: var(--space-4);
}
```

---

## 9. Performance

- `<script type="module">` is deferred by default — no `defer` attribute needed
- Use `loading="lazy"` on all below-the-fold images
- Prefer CSS animations over JS animations — use `transform` and `opacity` for GPU compositing
- Use `IntersectionObserver` for scroll-triggered behaviour and lazy loading
- Avoid layout thrash: batch DOM reads before writes; never interleave
- Use `requestAnimationFrame` for visual updates; never `setTimeout` for animations
- Dynamic `import()` for code splitting in Vite projects:

```javascript
// Load a heavy module only when needed
async function openAnalyticsDashboard() {
  const { renderDashboard } = await import('./components/analytics-dashboard.component.js');
  renderDashboard(document.getElementById('dashboard-container'));
}
```

---

## 10. CSS Modules (Vite)

Available only in Vite projects. Name files `.module.css`. Import as an object in JS.

```css
/* src/components/order-card/order-card.module.css */

.card {
  border:        1px solid var(--color-neutral-300);
  border-radius: var(--radius-md);
  padding:       var(--space-4);
  background:    #fff;
  box-shadow:    var(--shadow-sm);
  transition:    box-shadow var(--transition-fast);
}

.card:hover { box-shadow: var(--shadow-md); }

.cardShipped   { border-left: 4px solid var(--color-success); }
.cardCancelled { border-left: 4px solid var(--color-danger); }
.cardPending   { border-left: 4px solid var(--color-warning); }

.title  { font-size: var(--font-size-lg); font-weight: var(--font-weight-semibold); margin: 0; }
.status { font-size: var(--font-size-sm); color: var(--color-neutral-600); }
.amount { font-size: var(--font-size-base); font-weight: var(--font-weight-medium); }
```

```javascript
// src/components/order-card/order-card.component.js

import styles from './order-card.module.css';

/**
 * Creates an order card DOM element with scoped CSS Module classes.
 * @param {{ orderId: string; status: string; totalAmount: number }} order
 * @returns {HTMLElement}
 */
export function createOrderCard(order) {
  const statusClass = styles[`card${order.status.charAt(0).toUpperCase()}${order.status.slice(1).toLowerCase()}`] ?? '';

  const article       = document.createElement('article');
  article.className   = `${styles.card} ${statusClass}`.trim();
  article.setAttribute('aria-label', `Order ${order.orderId}, status: ${order.status}`);

  article.innerHTML = `
    <h3 class="${styles.title}">${order.orderId}</h3>
    <p class="${styles.status}">${order.status}</p>
    <p class="${styles.amount}">$${order.totalAmount.toFixed(2)}</p>
  `;

  return article;
}
```

---

## 11. SCSS (Vite)

Install `sass` as a dev dependency: `npm install -D sass`. Use `.scss` extension.

```scss
/* src/styles/_tokens.scss */

$color-primary:       #1a56db;
$color-primary-hover: #1e40af;
$color-danger:        #e02424;
$color-success:       #057a55;
$color-neutral-300:   #d1d5db;
$color-neutral-600:   #4b5563;
$color-neutral-900:   #111827;

$space-2:  0.5rem;
$space-4:  1rem;
$space-6:  1.5rem;
$font-size-sm:   0.875rem;
$font-size-lg:   1.125rem;
$font-weight-semibold: 600;
$radius-md:  0.5rem;
$shadow-sm:  0 1px 2px rgb(0 0 0 / 0.05);
$shadow-md:  0 4px 6px rgb(0 0 0 / 0.1);
$transition-fast: 150ms ease;
```

```scss
/* src/styles/components/_order-card.scss */

@use '../tokens' as t;

.order-card {
  border:        1px solid t.$color-neutral-300;
  border-radius: t.$radius-md;
  padding:       t.$space-4;
  background:    #fff;
  box-shadow:    t.$shadow-sm;
  transition:    box-shadow t.$transition-fast;

  &:hover { box-shadow: t.$shadow-md; }

  &__title {
    font-size:   t.$font-size-lg;
    font-weight: t.$font-weight-semibold;
    margin:      0 0 t.$space-2;
  }

  &__status {
    font-size: t.$font-size-sm;
    color:     t.$color-neutral-600;
  }

  &--shipped   { border-left: 4px solid t.$color-success; }
  &--cancelled { border-left: 4px solid t.$color-danger; }
}
```

**SCSS rules:**
- Use `@use` — never `@import` (deprecated in Dart Sass)
- Namespace all `@use` imports: `@use '../tokens' as t`
- Partials use underscore prefix (`_order-card.scss`), imported without it
- Nesting limited to BEM `&__element` and `&--modifier` — no arbitrary descendant nesting

---

## 12. Naming Conventions

| Item | Convention | Example |
|---|---|---|
| HTML files | kebab-case | `order-detail.html` |
| CSS files | kebab-case | `order-card.css` |
| JS files | kebab-case + role suffix | `order-card.component.js` |
| JS service files | `.service.js` | `orders.service.js` |
| JS state files | `.state.js` | `orders.state.js` |
| JS utility files | `.utils.js` | `dom.utils.js`, `http.utils.js` |
| JS model files | `.model.js` | `order.model.js` |
| CSS Module files | `.module.css` | `order-card.module.css` |
| SCSS partials | `_` prefix, no extension import | `_order-card.scss` |
| CSS classes (BEM) | lowercase kebab | `order-card__title--active` |
| JS functions | camelCase | `fetchOrders()`, `renderOrderCard()` |
| JS constants | SCREAMING_SNAKE_CASE | `API_BASE_URL`, `ORDER_STATUSES` |
| JS private fields | `#field` | `#shadow`, `#abortController` |
| Custom element names | namespaced kebab | `order-card`, `app-modal`, `ui-spinner` |

---

## 13. Code Quality Rules

**Always:**
- Use named exports — no anonymous default exports
- Declare variables at the top of their scope with `const`/`let`
- Guard against null before DOM access: `document.getElementById('x')?.addEventListener(...)`
- Call `event.preventDefault()` for all custom form handlers to block native browser submission
- Add `aria-label` or visible label text to every interactive element
- Use template literals for HTML strings — never string concatenation
- Use `structuredClone()` to return copies of mutable state
- Remove all `console.log` calls before committing

**Never:**
- Use `var`, `eval()`, `document.write()`, or `innerHTML` with unsanitised user input
- Rely on `window.*` global variables for cross-module communication
- Animate using `left`/`top`/`width` — use `transform` and `opacity`
- Access `.style` directly for layout-affecting properties — toggle CSS classes instead
- Extend built-in HTML elements via `customElements.define('my-button', ..., { extends: 'button' })` — Safari does not support this
- Put `<script>` tags in the `<body>` — all scripts go at the bottom as `type="module"`

---

## 14. Quick Reference

| Need | Approach |
|---|---|
| Reusable encapsulated UI | Web Component (`HTMLElement` + Shadow DOM) |
| Page-level layout | CSS Grid with `grid-template-columns` / `grid-template-areas` |
| Component internals layout | Flexbox |
| Shared design values | CSS custom properties in `tokens.css` / `_tokens.scss` |
| CSS class naming | BEM (`block__element--modifier`) |
| Scoped styles (Vite) | CSS Modules (`.module.css`) |
| Preprocessed CSS (Vite) | SCSS with `@use` |
| Cross-component state | Module singleton + `CustomEvent` on `document` |
| HTTP calls | `http()` utility wrapping `fetch` with `AbortController` |
| Type hints in JS | JSDoc `/** @param {string} id @returns {Promise<Order>} */` |
| Visually hidden text | `.sr-only` utility class |
| HTML fragments | `<template>` element + `content.cloneNode(true)` |
| Modals | Native `<dialog>` + `.showModal()` |
| Code splitting | Dynamic `import('./module.js')` (Vite) |
