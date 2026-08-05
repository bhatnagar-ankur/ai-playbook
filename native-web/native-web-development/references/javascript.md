# JavaScript — Full Reference

Complete ES2020+ patterns: modules, DOM, events, Fetch, error handling, and JSDoc types.
For the overview and quick rules see the **JavaScript Conventions** section in `SKILL.md`.

---

## Table of Contents
1. [ES Modules](#es-modules)
2. [JSDoc Type Annotations](#jsdoc-type-annotations)
3. [DOM Selection and Manipulation](#dom-selection-and-manipulation)
4. [Event Handling](#event-handling)
5. [Custom Events](#custom-events)
6. [Fetch API and HTTP Utility](#fetch-api-and-http-utility)
7. [Error Handling](#error-handling)
8. [Async Patterns](#async-patterns)
9. [Module Singleton Services](#module-singleton-services)
10. [Local Storage Helper](#local-storage-helper)
11. [ES2020+ Feature Reference](#es2020-feature-reference)

---

## ES Modules

Every file is an ES module. Use named imports and exports.

```javascript
// ── Named exports (preferred) ──────────────────────

// scripts/utils/dom.utils.js
export function querySelector(selector, context = document) {
  return context.querySelector(selector);
}

export function querySelectorAll(selector, context = document) {
  return [...context.querySelectorAll(selector)];
}

// ── Named imports ──────────────────────────────────

import { querySelector, querySelectorAll } from './utils/dom.utils.js';

// ── Re-exports ─────────────────────────────────────

// scripts/utils/index.js  (if you want a single import point for utils)
export { querySelector, querySelectorAll } from './dom.utils.js';
export { http }                            from './http.utils.js';
export { announce }                        from './announce.utils.js';

// ── Dynamic import (code splitting in Vite) ────────

async function loadDashboard() {
  const { renderDashboard } = await import('./components/dashboard.component.js');
  renderDashboard(document.getElementById('app'));
}
```

**Rules:**
- Always use the `.js` file extension in import paths — browsers require it
- Never use default exports — named exports make search and refactoring predictable
- Never use `import *` — import only what is needed
- Module paths are always relative to the current file: `'./utils/http.utils.js'`

---

## JSDoc Type Annotations

No TypeScript — use JSDoc for type hints. Editors (VS Code) read these for autocomplete and errors.

```javascript
// scripts/models/order.model.js

/**
 * Represents an order from the API.
 * @typedef {Object} Order
 * @property {string}      orderId      - Unique order identifier
 * @property {string}      customerId   - Customer account ID
 * @property {OrderStatus} status       - Current order status
 * @property {number}      totalAmount  - Order total in the base currency unit
 * @property {string}      currencyCode - ISO 4217 currency code (e.g. 'USD')
 * @property {string}      placedAt     - ISO 8601 timestamp
 * @property {OrderItem[]} items        - Line items in this order
 */

/**
 * @typedef {Object} OrderItem
 * @property {string} productId - Product identifier
 * @property {string} name      - Product display name
 * @property {number} quantity  - Number of units
 * @property {number} unitPrice - Price per unit
 */

/**
 * @typedef {'PENDING' | 'PROCESSING' | 'SHIPPED' | 'DELIVERED' | 'CANCELLED'} OrderStatus
 */

/**
 * Creates an Order object from a raw API response.
 * @param {Record<string, unknown>} raw - Raw JSON from the API
 * @returns {Order}
 */
export function orderFromApi(raw) {
  return {
    orderId:      String(raw['order_id'] ?? ''),
    customerId:   String(raw['customer_id'] ?? ''),
    status:       /** @type {OrderStatus} */ (raw['status'] ?? 'PENDING'),
    totalAmount:  Number(raw['total_amount'] ?? 0),
    currencyCode: String(raw['currency_code'] ?? 'USD'),
    placedAt:     String(raw['placed_at'] ?? ''),
    items:        Array.isArray(raw['items'])
      ? raw['items'].map(orderItemFromApi)
      : [],
  };
}

/**
 * @param {Record<string, unknown>} raw
 * @returns {OrderItem}
 */
function orderItemFromApi(raw) {
  return {
    productId: String(raw['product_id'] ?? ''),
    name:      String(raw['name'] ?? ''),
    quantity:  Number(raw['quantity'] ?? 0),
    unitPrice: Number(raw['unit_price'] ?? 0),
  };
}
```

```javascript
// Function signatures with JSDoc

/**
 * Renders a list of orders into a container element.
 *
 * @param {Order[]} orders          - Orders to render
 * @param {HTMLElement} container   - Target container element
 * @param {{ onSelect: (orderId: string) => void }} callbacks
 * @returns {void}
 */
export function renderOrdersList(orders, container, callbacks) {
  // ...
}

/**
 * Fetches and caches orders for a given customer.
 *
 * @param {string} customerId
 * @param {{ signal?: AbortSignal }} [options]
 * @returns {Promise<Order[]>}
 * @throws {Error} If the request fails or the response is not ok
 */
export async function fetchOrdersByCustomer(customerId, options = {}) {
  // ...
}
```

---

## DOM Selection and Manipulation

### Safe selection helpers

```javascript
// scripts/utils/dom.utils.js

/**
 * Returns the first element matching a CSS selector within an optional context.
 * Throws if the element is not found (use for required elements).
 * @template {HTMLElement} T
 * @param {string} selector
 * @param {Document | HTMLElement} [context]
 * @returns {T}
 */
export function getElement(selector, context = document) {
  const el = /** @type {T | null} */ (context.querySelector(selector));
  if (!el) throw new Error(`Element not found: ${selector}`);
  return el;
}

/**
 * Returns all elements matching a CSS selector as an array.
 * Returns an empty array when no matches are found.
 * @template {HTMLElement} T
 * @param {string} selector
 * @param {Document | HTMLElement} [context]
 * @returns {T[]}
 */
export function getElements(selector, context = document) {
  return /** @type {T[]} */ ([...context.querySelectorAll(selector)]);
}

/**
 * Safely toggles a class on an element.
 * @param {HTMLElement | null | undefined} el
 * @param {string} className
 * @param {boolean} [force]
 */
export function toggleClass(el, className, force) {
  el?.classList.toggle(className, force);
}

/**
 * Sets an element's text content safely.
 * @param {HTMLElement | null | undefined} el
 * @param {string} text
 */
export function setText(el, text) {
  if (el) el.textContent = text;
}

/**
 * Shows a hidden element by removing the `hidden` attribute.
 * @param {HTMLElement | null | undefined} el
 */
export function show(el) {
  el?.removeAttribute('hidden');
}

/**
 * Hides an element by setting the `hidden` attribute.
 * @param {HTMLElement | null | undefined} el
 */
export function hide(el) {
  el?.setAttribute('hidden', '');
}
```

### Batching DOM updates (avoid layout thrash)

```javascript
// ❌ Wrong — reads and writes are interleaved, causing forced reflows
elements.forEach((el) => {
  const height = el.getBoundingClientRect().height;   // READ — triggers reflow
  el.style.marginTop = `${height * 0.5}px`;           // WRITE
});

// ✅ Correct — batch reads first, then batch writes
const heights = elements.map((el) => el.getBoundingClientRect().height);   // Batch READ
elements.forEach((el, i) => {
  el.style.marginTop = `${heights[i] * 0.5}px`;                            // Batch WRITE
});

// ✅ For animations — use requestAnimationFrame
function animateIn(el) {
  el.style.opacity   = '0';
  el.style.transform = 'translateY(1rem)';

  // Schedule state change in next frame so the browser registers the initial state
  requestAnimationFrame(() => {
    el.style.transition = 'opacity 250ms ease, transform 250ms ease';
    el.style.opacity    = '1';
    el.style.transform  = 'translateY(0)';
  });
}
```

### DocumentFragment for bulk inserts

```javascript
/**
 * Renders a list of orders into a container using a DocumentFragment.
 * A single DOM insertion avoids repeated reflows.
 * @param {Order[]} orders
 * @param {HTMLElement} container
 */
export function renderOrdersList(orders, container) {
  const fragment = document.createDocumentFragment();

  for (const order of orders) {
    const card = createOrderCard(order);
    fragment.appendChild(card);
  }

  container.innerHTML = '';
  container.appendChild(fragment);
}
```

---

## Event Handling

### Direct event listeners

```javascript
// scripts/main.js

const createBtn = document.getElementById('create-order-btn');

// Use named functions so they can be removed later
function handleCreateBtnClick() {
  openCreateOrderModal();
}

createBtn?.addEventListener('click', handleCreateBtnClick);

// Cleanup when no longer needed
createBtn?.removeEventListener('click', handleCreateBtnClick);
```

### Event delegation (efficient for dynamic lists)

```javascript
/**
 * Attaches a delegated event listener to a parent container.
 * Handles clicks on matching descendant elements, including dynamically added ones.
 *
 * @param {HTMLElement} container
 * @param {string} selector    - CSS selector of the target descendants
 * @param {string} eventType   - DOM event type (e.g. 'click')
 * @param {(event: Event, target: HTMLElement) => void} handler
 * @returns {() => void} Cleanup function — call to remove the listener
 */
export function delegate(container, selector, eventType, handler) {
  function listener(event) {
    const target = /** @type {HTMLElement} */ (event.target);
    const match  = target.closest(selector);
    if (match && container.contains(match)) {
      handler(event, /** @type {HTMLElement} */ (match));
    }
  }

  container.addEventListener(eventType, listener);
  return () => container.removeEventListener(eventType, listener);
}

// Usage — handles clicks on all .order-card__delete-btn elements, even dynamically added ones
const cleanupDelegate = delegate(
  document.getElementById('orders-section'),
  '.order-card__delete-btn',
  'click',
  (event, btn) => {
    const orderId = btn.closest('[data-order-id]')?.dataset.orderId;
    if (orderId) openConfirmDeleteDialog(orderId);
  },
);

// Call cleanupDelegate() when the section is destroyed
```

### Keyboard event handling

```javascript
/**
 * Allows an element to be activated via keyboard (Enter and Space) like a native button.
 * Only needed for non-button interactive elements (e.g. custom card with click handler).
 * Prefer <button> when possible.
 *
 * @param {HTMLElement} el
 * @param {() => void} handler
 */
export function makeKeyboardActivatable(el, handler) {
  el.setAttribute('role', 'button');
  if (!el.hasAttribute('tabindex')) {
    el.setAttribute('tabindex', '0');
  }

  el.addEventListener('click', handler);

  el.addEventListener('keydown', (event) => {
    if (event.key === 'Enter' || event.key === ' ') {
      event.preventDefault();
      handler();
    }
  });
}
```

---

## Custom Events

Use `CustomEvent` to communicate between modules without creating direct dependencies.

```javascript
// ── Dispatching ────────────────────────────────────

/**
 * Dispatches an 'order:created' event on document with the new order as detail.
 * @param {Order} order
 */
export function emitOrderCreated(order) {
  document.dispatchEvent(
    new CustomEvent('order:created', {
      detail:  { order },
      bubbles: false,
    }),
  );
}

/**
 * Dispatches an 'order:deleted' event on document.
 * @param {string} orderId
 */
export function emitOrderDeleted(orderId) {
  document.dispatchEvent(
    new CustomEvent('order:deleted', {
      detail:  { orderId },
      bubbles: false,
    }),
  );
}

// ── Subscribing ────────────────────────────────────

/** @param {(order: Order) => void} handler */
export function onOrderCreated(handler) {
  document.addEventListener('order:created', (event) => {
    handler(/** @type {CustomEvent<{ order: Order }>} */ (event).detail.order);
  });
}

/** @param {(orderId: string) => void} handler */
export function onOrderDeleted(handler) {
  document.addEventListener('order:deleted', (event) => {
    handler(/** @type {CustomEvent<{ orderId: string }>} */ (event).detail.orderId);
  });
}

// ── Constants for event names ──────────────────────

export const ORDER_EVENTS = /** @type {const} */ ({
  CREATED:        'order:created',
  DELETED:        'order:deleted',
  STATUS_CHANGED: 'order:statuschanged',
  LIST_REFRESHED: 'order:listrefreshed',
});
```

---

## Fetch API and HTTP Utility

```javascript
// scripts/utils/http.utils.js

/** Default request timeout in milliseconds. */
const DEFAULT_TIMEOUT_MS = 10_000;

/**
 * Typed HTTP utility wrapping `fetch` with timeout, JSON handling, and error normalisation.
 *
 * @param {string} url
 * @param {RequestInit & { timeoutMs?: number }} [options]
 * @returns {Promise<unknown>}
 * @throws {Error} On non-ok responses, network errors, or timeout
 */
export async function http(url, options = {}) {
  const { timeoutMs = DEFAULT_TIMEOUT_MS, ...fetchOptions } = options;

  const controller = new AbortController();
  const timeoutId  = setTimeout(() => controller.abort('timeout'), timeoutMs);

  try {
    const response = await fetch(url, {
      ...fetchOptions,
      signal:  controller.signal,
      headers: {
        'Content-Type': 'application/json',
        ...fetchOptions.headers,
      },
    });

    if (!response.ok) {
      let errorMessage = `HTTP ${response.status}: ${response.statusText}`;

      // Try to extract an error message from the response body
      try {
        const errorBody = await response.json();
        if (errorBody?.message) errorMessage = String(errorBody.message);
      } catch {
        // Ignore JSON parse errors on error responses
      }

      throw new Error(errorMessage);
    }

    if (response.status === 204) return null;

    return response.json();
  } catch (error) {
    if (error instanceof Error) {
      if (error.name === 'AbortError') {
        throw new Error('The request timed out. Please try again.');
      }
    }
    throw error;
  } finally {
    clearTimeout(timeoutId);
  }
}

/**
 * Convenience wrapper for GET requests.
 * @param {string} url
 * @param {RequestInit} [options]
 * @returns {Promise<unknown>}
 */
export function get(url, options = {}) {
  return http(url, { ...options, method: 'GET' });
}

/**
 * Convenience wrapper for POST requests with a JSON body.
 * @param {string} url
 * @param {unknown} body
 * @param {RequestInit} [options]
 * @returns {Promise<unknown>}
 */
export function post(url, body, options = {}) {
  return http(url, { ...options, method: 'POST', body: JSON.stringify(body) });
}

/**
 * Convenience wrapper for PATCH requests with a JSON body.
 * @param {string} url
 * @param {unknown} body
 * @param {RequestInit} [options]
 * @returns {Promise<unknown>}
 */
export function patch(url, body, options = {}) {
  return http(url, { ...options, method: 'PATCH', body: JSON.stringify(body) });
}

/**
 * Convenience wrapper for DELETE requests.
 * @param {string} url
 * @param {RequestInit} [options]
 * @returns {Promise<unknown>}
 */
export function del(url, options = {}) {
  return http(url, { ...options, method: 'DELETE' });
}
```

### Orders service using the http utility

```javascript
// scripts/services/orders.service.js

import { get, post, patch, del } from '../utils/http.utils.js';
import { orderFromApi }           from '../models/order.model.js';

const BASE = '/api/orders';

/** @returns {Promise<import('../models/order.model.js').Order[]>} */
export async function fetchOrders() {
  const raw = /** @type {Record<string, unknown>[]} */ (await get(BASE));
  return raw.map(orderFromApi);
}

/**
 * @param {string} orderId
 * @returns {Promise<import('../models/order.model.js').Order>}
 */
export async function fetchOrderById(orderId) {
  const raw = /** @type {Record<string, unknown>} */ (await get(`${BASE}/${orderId}`));
  return orderFromApi(raw);
}

/**
 * @param {{ customerId: string; items: Array<{ productId: string; quantity: number }> }} payload
 * @returns {Promise<import('../models/order.model.js').Order>}
 */
export async function createOrder(payload) {
  const raw = /** @type {Record<string, unknown>} */ (await post(BASE, payload));
  return orderFromApi(raw);
}

/**
 * @param {string} orderId
 * @param {import('../models/order.model.js').OrderStatus} status
 * @returns {Promise<void>}
 */
export async function updateOrderStatus(orderId, status) {
  await patch(`${BASE}/${orderId}/status`, { status });
}

/**
 * @param {string} orderId
 * @returns {Promise<void>}
 */
export async function deleteOrder(orderId) {
  await del(`${BASE}/${orderId}`);
}
```

---

## Error Handling

```javascript
// ── Try/catch with typed errors ────────────────────

/**
 * Loads orders into state, handling loading and error states.
 */
async function loadOrders() {
  const section = document.getElementById('orders-section');

  try {
    setLoadingState(section, true);
    setLoading(true);

    const orders = await fetchOrders();
    setOrders(orders);
    announce(`${orders.length} orders loaded.`);
  } catch (error) {
    const message = error instanceof Error
      ? error.message
      : 'An unexpected error occurred.';

    setError(message);
    announce(message, 'assertive');
  } finally {
    setLoadingState(section, false);
  }
}

// ── Cancellable fetch with AbortController ─────────

let loadController = null;

async function loadOrdersWithCancel() {
  // Cancel any in-flight request before starting a new one
  loadController?.abort();
  loadController = new AbortController();

  try {
    const orders = await fetchOrders({ signal: loadController.signal });
    setOrders(orders);
  } catch (error) {
    if (error instanceof Error && error.name === 'AbortError') return;  // Expected cancellation
    setError(error instanceof Error ? error.message : 'Failed to load orders.');
  }
}

// ── Global error boundary ──────────────────────────

window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled promise rejection:', event.reason);
  // Report to error monitoring (e.g. Sentry)
  // reportError(event.reason);
  event.preventDefault();   // Suppress browser console error in production
});
```

---

## Async Patterns

```javascript
// ── Sequential async ───────────────────────────────

async function processOrder(orderId) {
  const order    = await fetchOrderById(orderId);
  const customer = await fetchCustomerById(order.customerId);
  return { order, customer };
}

// ── Parallel async — Promise.all ───────────────────

async function loadDashboard() {
  // Both requests start simultaneously; total time = slowest request, not sum
  const [orders, customers] = await Promise.all([
    fetchOrders(),
    fetchCustomers(),
  ]);
  return { orders, customers };
}

// ── Parallel with partial failures — Promise.allSettled ──

async function loadDashboardWithFallback() {
  const results = await Promise.allSettled([
    fetchOrders(),
    fetchCustomers(),
    fetchAnalytics(),   // Non-critical — ok if this fails
  ]);

  const [ordersResult, customersResult, analyticsResult] = results;

  return {
    orders:    ordersResult.status    === 'fulfilled' ? ordersResult.value    : [],
    customers: customersResult.status === 'fulfilled' ? customersResult.value : [],
    analytics: analyticsResult.status === 'fulfilled' ? analyticsResult.value : null,
  };
}

// ── Optional chaining and nullish coalescing ───────

/**
 * @param {Order | null | undefined} order
 * @returns {string}
 */
function getOrderDisplayName(order) {
  return order?.orderId ?? 'Unknown order';
}

function getFirstItemName(order) {
  return order?.items?.[0]?.name ?? 'No items';
}

// ── Logical assignment operators ───────────────────

function ensureDefaults(config) {
  config.timeout  ??= 5000;    // Only assign if null or undefined
  config.retries  ??= 3;
  config.baseUrl  ||= '/api';  // Assign if falsy (null, undefined, '', 0, false)
}
```

---

## Module Singleton Services

Use module-level state for shared services. ES modules are cached by the runtime — importing
the same module from multiple files always returns the same instance.

```javascript
// scripts/services/auth.service.js

/** @typedef {{ userId: string; email: string; role: string }} AuthUser */

/** @type {AuthUser | null} */
let _currentUser = null;

/**
 * Returns the currently authenticated user, or null if not logged in.
 * @returns {AuthUser | null}
 */
export function getCurrentUser() {
  return _currentUser ? structuredClone(_currentUser) : null;
}

/**
 * Returns true if there is an authenticated user.
 * @returns {boolean}
 */
export function isAuthenticated() {
  return _currentUser !== null;
}

/**
 * Sets the current user after a successful login.
 * @param {AuthUser} user
 */
export function setCurrentUser(user) {
  _currentUser = user;
  document.dispatchEvent(new CustomEvent('auth:userchanged', { detail: { user: structuredClone(user) } }));
}

/**
 * Clears the current user session.
 */
export function clearCurrentUser() {
  _currentUser = null;
  document.dispatchEvent(new CustomEvent('auth:userchanged', { detail: { user: null } }));
}

/**
 * Initialises the auth service by loading the stored session.
 * Call once during app startup.
 * @returns {Promise<void>}
 */
export async function initAuth() {
  try {
    const response = await fetch('/api/auth/me');
    if (response.ok) {
      const raw = await response.json();
      _currentUser = {
        userId: String(raw.id),
        email:  String(raw.email),
        role:   String(raw.role),
      };
    }
  } catch {
    _currentUser = null;
  }
}
```

```javascript
// scripts/main.js — application entry point

import { initAuth, isAuthenticated } from './services/auth.service.js';
import { loadOrders }                from './services/orders.service.js';
import { renderApp }                 from './app.js';

async function bootstrap() {
  await initAuth();

  if (!isAuthenticated()) {
    window.location.href = '/login.html';
    return;
  }

  renderApp();
  await loadOrders();
}

bootstrap().catch((error) => {
  console.error('Application failed to start:', error);
  document.body.innerHTML = '<p>Failed to load the application. Please refresh.</p>';
});
```

---

## Local Storage Helper

```javascript
// scripts/utils/storage.utils.js

/**
 * Reads a value from localStorage and parses it as JSON.
 * Returns `defaultValue` if the key is absent or the value is invalid JSON.
 *
 * @template T
 * @param {string} key
 * @param {T} defaultValue
 * @returns {T}
 */
export function getStorageItem(key, defaultValue) {
  try {
    const raw = localStorage.getItem(key);
    return raw !== null ? /** @type {T} */ (JSON.parse(raw)) : defaultValue;
  } catch {
    return defaultValue;
  }
}

/**
 * Serialises a value as JSON and writes it to localStorage.
 * Silently ignores storage quota errors.
 *
 * @param {string} key
 * @param {unknown} value
 * @returns {boolean} True if the write succeeded
 */
export function setStorageItem(key, value) {
  try {
    localStorage.setItem(key, JSON.stringify(value));
    return true;
  } catch {
    return false;   // Storage full or access denied (e.g. private browsing)
  }
}

/**
 * Removes an item from localStorage.
 * @param {string} key
 */
export function removeStorageItem(key) {
  try {
    localStorage.removeItem(key);
  } catch {
    // Ignore — access denied
  }
}

// Usage
import { getStorageItem, setStorageItem } from './utils/storage.utils.js';

const FILTERS_KEY = 'orders:filters';

/** @typedef {{ status?: string; from?: string; to?: string }} OrderFilters */

/** @returns {OrderFilters} */
export function getSavedFilters() {
  return getStorageItem(FILTERS_KEY, {});
}

/** @param {OrderFilters} filters */
export function saveFilters(filters) {
  setStorageItem(FILTERS_KEY, filters);
}
```

---

## ES2020+ Feature Reference

```javascript
// Optional chaining ?. — short-circuits to undefined on null/undefined
const name  = user?.profile?.displayName;
const first = items?.[0]?.name;
const price = getPrice?.();

// Nullish coalescing ?? — falls back only on null or undefined (not '' or 0)
const label  = user.name ?? 'Anonymous';
const count  = data.count ?? 0;

// Nullish assignment ??= — assign only if null or undefined
config.timeout ??= 5000;

// Optional catch binding — no variable needed when the error isn't used
try {
  value = JSON.parse(raw);
} catch {
  value = null;
}

// Promise.allSettled — all promises run even if some reject
const results = await Promise.allSettled([p1, p2, p3]);

// Dynamic import() — lazy load a module
const { renderChart } = await import('./charts.js');

// globalThis — safe global reference in any environment
const isProduction = globalThis.location?.hostname !== 'localhost';

// Numeric separators — improve readability of large numbers
const maxFileSize = 5_242_880;   //  5 MB
const apiTimeout  = 10_000;      // 10 s

// String.prototype.matchAll — returns all regex match objects
const matches = [...text.matchAll(/order-(\w+)/g)];
const ids     = matches.map((m) => m[1]);

// Object.fromEntries — convert entries or Map to object
const filters = Object.fromEntries(
  new URL(window.location.href).searchParams.entries(),
);

// Array.flat and Array.flatMap
const allItems = orders.flatMap((order) => order.items);
const nested   = [[1, 2], [3, 4]].flat();

// structuredClone — deep clone without JSON round-trip
const copy = structuredClone(originalObject);

// at() — negative indexing for arrays and strings
const last        = items.at(-1);
const secondToLast = items.at(-2);
```
