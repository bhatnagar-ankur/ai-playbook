# Examples — Full Reference

Four complete, end-to-end worked examples demonstrating native web development patterns.

---

## Table of Contents
1. [Vite Project Bootstrap](#vite-project-bootstrap)
2. [Pure-Browser Orders List](#pure-browser-orders-list)
3. [Design System Setup — Tokens and Base CSS](#design-system-setup--tokens-and-base-css)
4. [Full Feature — Orders Page with State, Fetch, and Web Components](#full-feature--orders-page-with-state-fetch-and-web-components)

---

## Vite Project Bootstrap

A minimal Vite project for a native-web application — no framework, no TypeScript.

### File structure

```
my-app/
├── index.html
├── vite.config.js
├── package.json
└── src/
    ├── main.js
    ├── app.js
    ├── styles/
    │   ├── reset.css
    │   ├── tokens.css
    │   └── base.css
    └── components/
        └── (empty — add components here)
```

### package.json

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev":     "vite",
    "build":   "vite build",
    "preview": "vite preview"
  },
  "devDependencies": {
    "vite": "^5.0.0",
    "sass": "^1.70.0"
  }
}
```

### vite.config.js

```javascript
// vite.config.js

import { defineConfig } from 'vite';

export default defineConfig({
  root: '.',
  build: {
    outDir:      'dist',
    emptyOutDir: true,
    target:      'es2020',
    rollupOptions: {
      input: { main: './index.html' },
    },
  },
  css: {
    modules: {
      localsConvention: 'camelCase',
    },
  },
  server: {
    port:   3000,
    open:   true,
    proxy: {
      // Proxy API calls to your backend during development
      '/api': {
        target:      'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
});
```

### index.html (Vite entry)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta name="description" content="Order management application" />
  <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
  <title>Orders App</title>
</head>
<body>
  <a href="#main-content" class="skip-link">Skip to main content</a>

  <header class="site-header" role="banner">
    <nav aria-label="Primary navigation">
      <ul role="list">
        <li><a href="/" aria-current="page">Orders</a></li>
      </ul>
    </nav>
  </header>

  <main id="main-content" tabindex="-1">
    <div id="app"></div>
  </main>

  <!-- Vite resolves this at build time -->
  <script type="module" src="/src/main.js"></script>
</body>
</html>
```

### src/main.js (entry point)

```javascript
// src/main.js

import './styles/reset.css';
import './styles/tokens.css';
import './styles/base.css';

import { registerComponents } from './components/index.js';
import { initApp }            from './app.js';

registerComponents();

document.addEventListener('DOMContentLoaded', () => {
  initApp(document.getElementById('app'));
});
```

### src/app.js

```javascript
// src/app.js

/**
 * Initialises the application within the given container.
 * @param {HTMLElement | null} container
 */
export function initApp(container) {
  if (!container) {
    console.error('App container not found.');
    return;
  }

  container.innerHTML = `
    <h1>Orders</h1>
    <p>Application is running.</p>
  `;
}
```

---

## Pure-Browser Orders List

A complete orders list page with no build step. Uses `<template>`, Fetch API, BEM CSS,
and a module-level state singleton. Serves directly from any static file server.

### File structure

```
orders-app/
├── index.html
├── styles/
│   ├── reset.css
│   ├── tokens.css
│   ├── base.css
│   └── components/
│       ├── btn.css
│       ├── order-card.css
│       └── status-badge.css
└── scripts/
    ├── main.js
    ├── state/
    │   └── orders.state.js
    ├── services/
    │   └── orders.service.js
    ├── utils/
    │   ├── http.utils.js
    │   └── dom.utils.js
    └── components/
        ├── order-card.component.js
        └── status-badge.component.js
```

### index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Orders | App</title>
  <link rel="stylesheet" href="/styles/reset.css" />
  <link rel="stylesheet" href="/styles/tokens.css" />
  <link rel="stylesheet" href="/styles/base.css" />
  <link rel="stylesheet" href="/styles/components/btn.css" />
  <link rel="stylesheet" href="/styles/components/order-card.css" />
  <link rel="stylesheet" href="/styles/components/status-badge.css" />
</head>
<body>
  <a href="#main-content" class="skip-link">Skip to main content</a>

  <header class="site-header" role="banner">
    <nav aria-label="Primary navigation">
      <ul role="list">
        <li><a href="/" aria-current="page">Orders</a></li>
      </ul>
    </nav>
  </header>

  <main id="main-content" tabindex="-1">
    <article class="orders-page">
      <header class="orders-page__header">
        <h1 class="orders-page__title">Orders</h1>
        <p class="orders-page__count" aria-live="polite" id="orders-count"></p>
      </header>

      <!-- Loading / error / empty states -->
      <div id="orders-loading" aria-live="polite" hidden>
        <p class="loading-text">Loading orders…</p>
      </div>

      <div id="orders-error" role="alert" aria-live="assertive" hidden>
        <p class="error-text" id="orders-error-message"></p>
        <button type="button" class="btn btn--secondary" id="orders-retry-btn">
          Try again
        </button>
      </div>

      <div id="orders-empty" hidden>
        <p class="empty-state">No orders found.</p>
      </div>

      <!-- Orders grid — rendered by JS -->
      <section
        class="orders-grid"
        id="orders-list"
        aria-label="Orders list"
        aria-busy="false"
      ></section>
    </article>
  </main>

  <!-- Card template -->
  <template id="order-card-template">
    <article class="order-card">
      <header class="order-card__header">
        <h3 class="order-card__id"></h3>
        <span class="order-card__badge status-badge"></span>
      </header>
      <dl class="order-card__details">
        <dt class="sr-only">Customer</dt>
        <dd class="order-card__customer"></dd>
        <dt class="sr-only">Amount</dt>
        <dd class="order-card__amount"></dd>
      </dl>
      <footer class="order-card__footer">
        <button type="button" class="btn btn--secondary order-card__view-btn">View</button>
      </footer>
    </article>
  </template>

  <!-- Screen reader announcer -->
  <div id="sr-announcer" class="sr-only" aria-live="polite" aria-atomic="true"></div>

  <script type="module" src="/scripts/main.js"></script>
</body>
</html>
```

### scripts/state/orders.state.js

```javascript
// scripts/state/orders.state.js

/**
 * @typedef {'idle' | 'loading' | 'success' | 'error'} LoadState
 *
 * @typedef {Object} OrdersState
 * @property {import('../models/order.model.js').Order[]} orders
 * @property {LoadState} loadState
 * @property {string | null} errorMessage
 */

/** @type {import('../models/order.model.js').Order[]} */
let _orders = [];
/** @type {LoadState} */
let _loadState = 'idle';
/** @type {string | null} */
let _errorMessage = null;

export function getOrders()       { return structuredClone(_orders); }
export function getLoadState()    { return _loadState; }
export function getErrorMessage() { return _errorMessage; }

/** @param {import('../models/order.model.js').Order[]} orders */
export function setOrders(orders) {
  _orders       = orders;
  _loadState    = 'success';
  _errorMessage = null;
  dispatch();
}

export function setLoading() {
  _loadState    = 'loading';
  _errorMessage = null;
  dispatch();
}

/** @param {string} message */
export function setError(message) {
  _loadState    = 'error';
  _errorMessage = message;
  dispatch();
}

function dispatch() {
  document.dispatchEvent(
    new CustomEvent('orders:statechanged', {
      detail: {
        orders:       getOrders(),
        loadState:    _loadState,
        errorMessage: _errorMessage,
      },
    }),
  );
}
```

### scripts/main.js

```javascript
// scripts/main.js

import { fetchOrders }                              from './services/orders.service.js';
import { setOrders, setLoading, setError, getOrders } from './state/orders.state.js';

// ── DOM references ──────────────────────────────────────────────────────────

const ordersCountEl   = document.getElementById('orders-count');
const ordersListEl    = document.getElementById('orders-list');
const ordersLoadingEl = document.getElementById('orders-loading');
const ordersErrorEl   = document.getElementById('orders-error');
const ordersErrorMsgEl= document.getElementById('orders-error-message');
const ordersEmptyEl   = document.getElementById('orders-empty');
const retryBtn        = document.getElementById('orders-retry-btn');
const announcer       = document.getElementById('sr-announcer');

/** @type {HTMLTemplateElement} */
const cardTemplate = /** @type {HTMLTemplateElement} */ (document.getElementById('order-card-template'));

// ── State subscription ──────────────────────────────────────────────────────

document.addEventListener('orders:statechanged', (event) => {
  const { orders, loadState, errorMessage } = /** @type {CustomEvent} */ (event).detail;
  render(orders, loadState, errorMessage);
});

// ── Render ──────────────────────────────────────────────────────────────────

/**
 * @param {import('./models/order.model.js').Order[]} orders
 * @param {string} loadState
 * @param {string | null} errorMessage
 */
function render(orders, loadState, errorMessage) {
  // Show/hide panels
  const isLoading = loadState === 'loading';
  const isError   = loadState === 'error';
  const isEmpty   = loadState === 'success' && orders.length === 0;
  const hasList   = loadState === 'success' && orders.length > 0;

  ordersLoadingEl?.toggleAttribute('hidden', !isLoading);
  ordersErrorEl?.toggleAttribute('hidden', !isError);
  ordersEmptyEl?.toggleAttribute('hidden', !isEmpty);
  ordersListEl?.setAttribute('aria-busy', isLoading ? 'true' : 'false');

  if (isError && ordersErrorMsgEl) {
    ordersErrorMsgEl.textContent = errorMessage ?? 'An error occurred.';
  }

  if (ordersCountEl) {
    ordersCountEl.textContent = hasList ? `${orders.length} orders` : '';
  }

  if (hasList && ordersListEl) {
    renderOrdersList(orders, ordersListEl);
    announce(`${orders.length} orders loaded.`);
  }
}

/**
 * @param {import('./models/order.model.js').Order[]} orders
 * @param {HTMLElement} container
 */
function renderOrdersList(orders, container) {
  const fragment = document.createDocumentFragment();

  for (const order of orders) {
    const card = createOrderCard(order);
    fragment.appendChild(card);
  }

  container.innerHTML = '';
  container.appendChild(fragment);
}

/**
 * @param {import('./models/order.model.js').Order} order
 * @returns {HTMLElement}
 */
function createOrderCard(order) {
  const card = /** @type {HTMLElement} */ (cardTemplate.content.cloneNode(true).firstElementChild);

  card.setAttribute('aria-label', `Order ${order.orderId}, status: ${order.status}`);
  card.classList.add(`order-card--${order.status.toLowerCase()}`);
  card.dataset.orderId = order.orderId;

  const idEl       = card.querySelector('.order-card__id');
  const badgeEl    = card.querySelector('.order-card__badge');
  const customerEl = card.querySelector('.order-card__customer');
  const amountEl   = card.querySelector('.order-card__amount');
  const viewBtn    = card.querySelector('.order-card__view-btn');

  if (idEl)       idEl.textContent       = order.orderId;
  if (badgeEl)    badgeEl.textContent    = order.status;
  if (badgeEl)    badgeEl.className     += ` status-badge--${order.status.toLowerCase()}`;
  if (customerEl) customerEl.textContent = order.customerId;
  if (amountEl)   amountEl.textContent   = `$${order.totalAmount.toFixed(2)}`;

  viewBtn?.addEventListener('click', () => {
    window.location.href = `/orders/${order.orderId}`;
  });

  return card;
}

/** @param {string} message */
function announce(message) {
  if (!announcer) return;
  announcer.textContent = '';
  requestAnimationFrame(() => { announcer.textContent = message; });
}

// ── Data loading ────────────────────────────────────────────────────────────

async function loadOrders() {
  setLoading();

  try {
    const orders = await fetchOrders();
    setOrders(orders);
  } catch (error) {
    setError(error instanceof Error ? error.message : 'Failed to load orders.');
  }
}

// ── Event wiring ────────────────────────────────────────────────────────────

retryBtn?.addEventListener('click', () => loadOrders());

// ── Bootstrap ───────────────────────────────────────────────────────────────

loadOrders();
```

---

## Design System Setup — Tokens and Base CSS

A complete design token setup that can be copied into any project as the starting point.

### styles/reset.css

(See the full reset in `references/css.md` — copy as-is.)

### styles/tokens.css

(See the full tokens in `references/css.md` — copy as-is.)

### styles/base.css

```css
/* styles/base.css */

body {
  font-family: var(--font-family-base);
  font-size:   var(--font-size-base);
  line-height: var(--line-height-normal);
  color:       var(--color-neutral-900);
  background:  var(--color-neutral-50);
}

h1 { font-size: var(--font-size-4xl); font-weight: var(--font-weight-bold);     line-height: var(--line-height-tight); margin-bottom: var(--space-4); }
h2 { font-size: var(--font-size-3xl); font-weight: var(--font-weight-bold);     line-height: var(--line-height-tight); margin-bottom: var(--space-3); }
h3 { font-size: var(--font-size-2xl); font-weight: var(--font-weight-semibold); line-height: var(--line-height-tight); margin-bottom: var(--space-2); }

a {
  color:                var(--color-primary);
  text-underline-offset: 3px;
}
a:hover { color: var(--color-primary-dark); }
a:focus-visible {
  outline:        2px solid var(--color-primary);
  outline-offset: 3px;
  border-radius:  var(--radius-sm);
}

:focus-visible {
  outline:        2px solid var(--color-primary);
  outline-offset: 3px;
  border-radius:  var(--radius-sm);
}

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

/* ── Layout containers ────────────────────────── */

.container {
  width:      100%;
  max-width:  72rem;   /* 1152px */
  margin-inline: auto;
  padding-inline: var(--space-4);
}

@media (min-width: 1024px) {
  .container { padding-inline: var(--space-6); }
}

/* ── Site header ──────────────────────────────── */

.site-header {
  position:     sticky;
  top:          0;
  z-index:      var(--z-sticky);
  background:   #fff;
  border-bottom: 1px solid var(--color-neutral-200);
  box-shadow:   var(--shadow-xs);
}

.site-header > nav {
  display:         flex;
  align-items:     center;
  gap:             var(--space-4);
  padding:         var(--space-3) var(--space-6);
  max-width:       72rem;
  margin-inline:   auto;
}

.site-header nav ul {
  display:     flex;
  gap:         var(--space-1);
  list-style:  none;
}

.site-header nav a {
  display:         block;
  padding:         var(--space-2) var(--space-3);
  border-radius:   var(--radius-md);
  text-decoration: none;
  font-size:       var(--font-size-sm);
  font-weight:     var(--font-weight-medium);
  color:           var(--color-neutral-600);
  transition:      background var(--transition-fast), color var(--transition-fast);
}
.site-header nav a:hover       { background: var(--color-neutral-100); color: var(--color-neutral-900); }
.site-header nav a[aria-current="page"] {
  background: var(--color-primary-light);
  color:      var(--color-primary);
  font-weight: var(--font-weight-semibold);
}

/* ── Site footer ──────────────────────────────── */

.site-footer {
  border-top:     1px solid var(--color-neutral-200);
  padding:        var(--space-6);
  text-align:     center;
  color:          var(--color-neutral-500);
  font-size:      var(--font-size-sm);
  margin-top:     auto;
}

/* ── Page layout patterns ─────────────────────── */

.page {
  padding:    var(--space-6) var(--space-4);
  max-width:  72rem;
  margin-inline: auto;
}

.page__header {
  display:         flex;
  align-items:     baseline;
  justify-content: space-between;
  gap:             var(--space-4);
  margin-bottom:   var(--space-6);
  padding-bottom:  var(--space-4);
  border-bottom:   1px solid var(--color-neutral-200);
}

/* ── Utility ──────────────────────────────────── */

.text-muted { color: var(--color-neutral-500); }
.text-sm    { font-size: var(--font-size-sm); }
.text-lg    { font-size: var(--font-size-lg); }
.font-bold  { font-weight: var(--font-weight-bold); }

.empty-state {
  text-align:  center;
  padding:     var(--space-12) var(--space-4);
  color:       var(--color-neutral-500);
  font-size:   var(--font-size-lg);
}

.loading-text {
  text-align:  center;
  padding:     var(--space-8);
  color:       var(--color-neutral-500);
}

.error-text {
  color:       var(--color-danger);
  font-weight: var(--font-weight-medium);
  margin-bottom: var(--space-3);
}
```

---

## Full Feature — Orders Page with State, Fetch, and Web Components

A complete production-quality orders page combining all patterns: module state, Fetch API,
Web Components, BEM CSS, and accessibility.

### Feature file structure

```
orders-feature/
├── index.html
├── styles/
│   ├── reset.css
│   ├── tokens.css
│   └── base.css
└── scripts/
    ├── main.js
    ├── models/
    │   └── order.model.js
    ├── state/
    │   └── orders.state.js
    ├── services/
    │   └── orders.service.js
    ├── utils/
    │   ├── http.utils.js
    │   └── dom.utils.js
    └── components/
        ├── index.js
        ├── order-card.component.js    (Web Component)
        ├── status-badge.component.js  (Web Component)
        └── notification-toast.component.js (Web Component)
```

### scripts/models/order.model.js

```javascript
// scripts/models/order.model.js

/**
 * @typedef {Object} Order
 * @property {string}      orderId
 * @property {string}      customerId
 * @property {OrderStatus} status
 * @property {number}      totalAmount
 * @property {string}      currencyCode
 * @property {string}      placedAt
 */

/** @typedef {'PENDING' | 'PROCESSING' | 'SHIPPED' | 'DELIVERED' | 'CANCELLED'} OrderStatus */

/**
 * Maps a raw API response object to a typed Order.
 * @param {Record<string, unknown>} raw
 * @returns {Order}
 */
export function orderFromApi(raw) {
  return {
    orderId:      String(raw['order_id']      ?? raw['orderId']      ?? ''),
    customerId:   String(raw['customer_id']   ?? raw['customerId']   ?? ''),
    status:       /** @type {OrderStatus} */ (String(raw['status'] ?? 'PENDING')),
    totalAmount:  Number(raw['total_amount']  ?? raw['totalAmount']  ?? 0),
    currencyCode: String(raw['currency_code'] ?? raw['currencyCode'] ?? 'USD'),
    placedAt:     String(raw['placed_at']     ?? raw['placedAt']     ?? ''),
  };
}
```

### scripts/components/order-card.component.js

```javascript
// scripts/components/order-card.component.js

/**
 * <order-card> Web Component.
 *
 * Attributes:
 *   order-id     {string}  — order identifier
 *   customer-id  {string}  — customer identifier
 *   status       {string}  — order status (PENDING, SHIPPED, etc.)
 *   amount       {string}  — formatted amount string, e.g. "$128.00"
 *   placed-at    {string}  — ISO 8601 date string
 *
 * Events dispatched (bubbles + composed):
 *   order-card:view   — user clicked "View"
 *   order-card:delete — user clicked "Delete"
 */
export class OrderCard extends HTMLElement {
  static observedAttributes = ['order-id', 'customer-id', 'status', 'amount', 'placed-at'];

  #shadow;
  #abortController;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
    this.#abortController = new AbortController();
  }

  connectedCallback() {
    this.#render();
    this.#attachListeners();
  }

  disconnectedCallback() {
    this.#abortController.abort();
  }

  attributeChangedCallback(_name, oldValue, newValue) {
    if (newValue !== oldValue) this.#render();
  }

  get orderId()     { return this.getAttribute('order-id')    ?? ''; }
  get customerId()  { return this.getAttribute('customer-id') ?? ''; }
  get status()      { return this.getAttribute('status')      ?? 'PENDING'; }
  get amount()      { return this.getAttribute('amount')      ?? '$0.00'; }
  get placedAt()    { return this.getAttribute('placed-at')   ?? ''; }

  #formattedDate() {
    if (!this.placedAt) return '';
    try {
      return new Date(this.placedAt).toLocaleDateString('en-US', {
        year: 'numeric', month: 'short', day: 'numeric',
      });
    } catch {
      return this.placedAt;
    }
  }

  #render() {
    const statusClass  = `badge--${this.status.toLowerCase()}`;
    const cardModifier = `card--${this.status.toLowerCase()}`;

    this.#shadow.innerHTML = `
      <style>
        :host { display: block; }

        .card {
          display:        flex;
          flex-direction: column;
          gap:            0.75rem;
          padding:        1rem;
          border:         1px solid #e5e7eb;
          border-radius:  0.5rem;
          background:     #fff;
          box-shadow:     0 1px 2px rgb(0 0 0 / 0.05);
          transition:     box-shadow 150ms ease, transform 150ms ease;
        }
        .card:hover { box-shadow: 0 4px 6px rgb(0 0 0 / 0.07); transform: translateY(-1px); }

        .card--pending   { border-left: 4px solid #d97706; }
        .card--processing{ border-left: 4px solid #1a56db; }
        .card--shipped   { border-left: 4px solid #057a55; }
        .card--delivered { border-left: 4px solid #0284c7; }
        .card--cancelled { border-left: 4px solid #e02424; }

        .card__header {
          display:         flex;
          align-items:     flex-start;
          justify-content: space-between;
          gap:             0.5rem;
        }

        .card__id {
          font-size:   0.875rem;
          font-weight: 600;
          color:       #111827;
          margin:      0;
        }

        .badge {
          display:        inline-flex;
          padding:        0.125rem 0.5rem;
          border-radius:  9999px;
          font-size:      0.6875rem;
          font-weight:    600;
          text-transform: uppercase;
          letter-spacing: 0.05em;
          flex-shrink:    0;
        }
        .badge--pending    { background: #fef3c7; color: #92400e; }
        .badge--processing { background: #dbeafe; color: #1e40af; }
        .badge--shipped    { background: #d1fae5; color: #065f46; }
        .badge--delivered  { background: #e0f2fe; color: #075985; }
        .badge--cancelled  { background: #fee2e2; color: #991b1b; }

        .card__details {
          display:               grid;
          grid-template-columns: auto 1fr;
          gap:                   0.125rem 0.5rem;
          font-size:             0.8125rem;
        }
        dt { color: #6b7280; }
        dd { color: #111827; font-weight: 500; }

        .card__amount {
          font-size:   1.125rem;
          font-weight: 700;
          color:       #111827;
        }

        .card__date {
          font-size: 0.75rem;
          color:     #9ca3af;
        }

        .card__actions {
          display:    flex;
          gap:        0.5rem;
          margin-top: auto;
        }

        button {
          display:         inline-flex;
          align-items:     center;
          justify-content: center;
          padding:         0.375rem 0.75rem;
          border:          1px solid #d1d5db;
          border-radius:   0.375rem;
          font-size:       0.8125rem;
          font-weight:     500;
          cursor:          pointer;
          transition:      background 150ms ease, border-color 150ms ease;
          background:      transparent;
          color:           #374151;
        }
        button:hover { background: #f3f4f6; border-color: #9ca3af; }
        button:focus-visible {
          outline:        2px solid #1a56db;
          outline-offset: 2px;
          border-radius:  3px;
        }
        button.danger        { color: #dc2626; border-color: #fca5a5; }
        button.danger:hover  { background: #fee2e2; border-color: #dc2626; }
      </style>

      <article class="card ${cardModifier}" aria-label="Order ${this.orderId}, status: ${this.status}">
        <header class="card__header">
          <h3 class="card__id">${this.orderId}</h3>
          <span class="badge ${statusClass}" role="status">${this.status}</span>
        </header>

        <dl class="card__details">
          <dt>Customer</dt>
          <dd>${this.customerId}</dd>
        </dl>

        <p class="card__amount">${this.amount}</p>
        ${this.#formattedDate() ? `<p class="card__date">Placed ${this.#formattedDate()}</p>` : ''}

        <div class="card__actions">
          <button type="button" id="view-btn">View details</button>
          <button type="button" class="danger" id="delete-btn">Delete</button>
        </div>
      </article>
    `;
  }

  #attachListeners() {
    const options = { signal: this.#abortController.signal };

    this.#shadow.getElementById('view-btn')?.addEventListener('click', () => {
      this.dispatchEvent(new CustomEvent('order-card:view', {
        detail: { orderId: this.orderId },
        bubbles: true, composed: true,
      }));
    }, options);

    this.#shadow.getElementById('delete-btn')?.addEventListener('click', () => {
      this.dispatchEvent(new CustomEvent('order-card:delete', {
        detail: { orderId: this.orderId },
        bubbles: true, composed: true,
      }));
    }, options);
  }
}

customElements.define('order-card', OrderCard);
```

### scripts/components/index.js

```javascript
// scripts/components/index.js

import { OrderCard }           from './order-card.component.js';
import { NotificationToast }   from './notification-toast.component.js';

export function registerComponents() {
  const defs = [
    ['order-card',         OrderCard],
    ['notification-toast', NotificationToast],
  ];

  for (const [name, ctor] of defs) {
    if (!customElements.get(name)) customElements.define(name, ctor);
  }
}
```

### scripts/main.js (full orders page)

```javascript
// scripts/main.js

import { registerComponents }                           from './components/index.js';
import { fetchOrders, deleteOrder }                     from './services/orders.service.js';
import { setOrders, setLoading, setError, getOrders }   from './state/orders.state.js';

registerComponents();

// ── DOM references ────────────────────────────────────────────────────────────

const ordersListEl  = /** @type {HTMLElement} */ (document.getElementById('orders-list'));
const loadingEl     = document.getElementById('orders-loading');
const errorEl       = document.getElementById('orders-error');
const errorMsgEl    = document.getElementById('orders-error-message');
const emptyEl       = document.getElementById('orders-empty');
const countEl       = document.getElementById('orders-count');
const retryBtn      = document.getElementById('orders-retry-btn');
const announcer     = document.getElementById('sr-announcer');

// ── State subscription ────────────────────────────────────────────────────────

document.addEventListener('orders:statechanged', (event) => {
  const { orders, loadState, errorMessage } = /** @type {CustomEvent} */ (event).detail;

  const isLoading = loadState === 'loading';
  const isError   = loadState === 'error';
  const hasList   = loadState === 'success' && orders.length > 0;
  const isEmpty   = loadState === 'success' && orders.length === 0;

  loadingEl?.toggleAttribute('hidden', !isLoading);
  errorEl?.toggleAttribute('hidden', !isError);
  emptyEl?.toggleAttribute('hidden', !isEmpty);
  ordersListEl.setAttribute('aria-busy', isLoading ? 'true' : 'false');

  if (isError && errorMsgEl) errorMsgEl.textContent = errorMessage ?? 'An error occurred.';
  if (countEl) countEl.textContent = hasList ? `${orders.length} orders` : '';

  if (hasList) {
    renderOrdersList(orders);
    announce(`${orders.length} orders loaded.`);
  }

  if (isEmpty) announce('No orders found.');
});

// ── Rendering ─────────────────────────────────────────────────────────────────

/** @param {import('./models/order.model.js').Order[]} orders */
function renderOrdersList(orders) {
  const fragment = document.createDocumentFragment();

  for (const order of orders) {
    const card = document.createElement('order-card');
    card.setAttribute('order-id',    order.orderId);
    card.setAttribute('customer-id', order.customerId);
    card.setAttribute('status',      order.status);
    card.setAttribute('amount',      `$${order.totalAmount.toFixed(2)}`);
    card.setAttribute('placed-at',   order.placedAt);
    fragment.appendChild(card);
  }

  ordersListEl.innerHTML = '';
  ordersListEl.appendChild(fragment);
}

// ── Event delegation — handle card events ─────────────────────────────────────

ordersListEl.addEventListener('order-card:view', (event) => {
  const { orderId } = /** @type {CustomEvent<{ orderId: string }>} */ (event).detail;
  window.location.href = `/orders/${orderId}`;
});

ordersListEl.addEventListener('order-card:delete', async (event) => {
  const { orderId } = /** @type {CustomEvent<{ orderId: string }>} */ (event).detail;

  if (!confirm(`Delete order ${orderId}? This cannot be undone.`)) return;

  try {
    await deleteOrder(orderId);

    const remaining = getOrders().filter((o) => o.orderId !== orderId);
    setOrders(remaining);

    showToast({ variant: 'success', message: `Order ${orderId} deleted.` });
  } catch (error) {
    showToast({
      variant: 'error',
      title:   'Delete failed',
      message: error instanceof Error ? error.message : 'Could not delete the order.',
    });
  }
});

// ── Toast helper ──────────────────────────────────────────────────────────────

/**
 * @param {{ variant: string; title?: string; message: string }} options
 */
function showToast({ variant, title, message }) {
  let container = document.getElementById('toast-container');

  if (!container) {
    container = document.createElement('div');
    container.id = 'toast-container';
    Object.assign(container.style, {
      position: 'fixed', bottom: '1.5rem', right: '1.5rem',
      display: 'flex', flexDirection: 'column', gap: '0.75rem',
      zIndex: '500', maxWidth: '22rem', width: '100%',
    });
    document.body.appendChild(container);
  }

  const toast = document.createElement('notification-toast');
  toast.setAttribute('variant',    variant);
  toast.setAttribute('duration',   '5000');
  toast.setAttribute('dismissible', '');

  if (title) {
    const titleEl = document.createElement('span');
    titleEl.slot        = 'title';
    titleEl.textContent = title;
    toast.appendChild(titleEl);
  }

  toast.appendChild(document.createTextNode(message));
  container.appendChild(toast);
}

// ── Announce ──────────────────────────────────────────────────────────────────

/** @param {string} message */
function announce(message) {
  if (!announcer) return;
  announcer.textContent = '';
  requestAnimationFrame(() => { announcer.textContent = message; });
}

// ── Data loading ──────────────────────────────────────────────────────────────

async function loadOrders() {
  setLoading();
  try {
    const orders = await fetchOrders();
    setOrders(orders);
  } catch (error) {
    setError(error instanceof Error ? error.message : 'Failed to load orders.');
  }
}

retryBtn?.addEventListener('click', () => loadOrders());

loadOrders();
```
