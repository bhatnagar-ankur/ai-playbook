# HTML — Full Reference

Complete patterns for semantic markup, forms, accessibility, the `<template>` element, and `<dialog>`.
For the overview and naming rules see the **HTML Conventions** section in `SKILL.md`.

---

## Table of Contents
1. [Semantic Structure](#semantic-structure)
2. [Forms](#forms)
3. [Accessibility Patterns](#accessibility-patterns)
4. [Images and Media](#images-and-media)
5. [The template Element](#the-template-element)
6. [Dialog and Modals](#dialog-and-modals)
7. [Meta and SEO](#meta-and-seo)

---

## Semantic Structure

### Full page layout

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Orders | App Name</title>
  <link rel="stylesheet" href="/styles/reset.css" />
  <link rel="stylesheet" href="/styles/tokens.css" />
  <link rel="stylesheet" href="/styles/base.css" />
</head>
<body>
  <!-- First focusable element — keyboard users skip repetitive navigation -->
  <a href="#main-content" class="skip-link">Skip to main content</a>

  <header class="site-header" role="banner">
    <a href="/" class="site-logo" aria-label="Go to homepage">
      <img src="/assets/logo.svg" alt="App Name" width="120" height="40" />
    </a>
    <nav aria-label="Primary navigation">
      <ul role="list">
        <li><a href="/">Home</a></li>
        <li><a href="/orders" aria-current="page">Orders</a></li>
        <li><a href="/products">Products</a></li>
        <li><a href="/settings">Settings</a></li>
      </ul>
    </nav>
    <div class="site-header__actions">
      <button type="button" class="btn btn--icon" aria-label="Open account menu" aria-haspopup="menu">
        <svg aria-hidden="true" focusable="false" width="20" height="20"><!-- icon --></svg>
      </button>
    </div>
  </header>

  <main id="main-content" tabindex="-1">
    <!-- tabindex="-1" allows programmatic focus after skip-link navigation -->
    <article class="orders-page">
      <header class="orders-page__header">
        <h1 class="orders-page__title">Your Orders</h1>
        <!-- Live region announces count changes to screen readers -->
        <p class="orders-page__count" aria-live="polite" aria-atomic="true">12 orders</p>
      </header>

      <section class="orders-page__filters" aria-label="Filter orders">
        <!-- filter controls -->
      </section>

      <section class="orders-page__list" aria-label="Orders list" aria-busy="false">
        <!-- order cards rendered here -->
      </section>
    </article>
  </main>

  <aside class="help-sidebar" aria-label="Help and support">
    <h2>Need help?</h2>
    <p>Contact <a href="mailto:support@example.com">support@example.com</a></p>
  </aside>

  <footer class="site-footer" role="contentinfo">
    <nav aria-label="Footer navigation">
      <ul role="list">
        <li><a href="/privacy">Privacy policy</a></li>
        <li><a href="/terms">Terms of service</a></li>
      </ul>
    </nav>
    <p class="site-footer__copy">&copy; 2025 App Name. All rights reserved.</p>
  </footer>

  <script type="module" src="/scripts/main.js"></script>
</body>
</html>
```

### Article vs Section vs Div

| Element | Use when |
|---|---|
| `<article>` | Self-contained, redistributable content (order card, blog post, product). Can be syndicated independently. |
| `<section>` | Thematic grouping within a page — always includes a heading (visible or `.sr-only`). |
| `<div>` | No semantic meaning. Use only as a styling hook or layout container. |

```html
<!-- Correct: section wraps the group; article wraps each self-contained item -->
<section aria-label="Recent orders">
  <h2 class="sr-only">Recent orders</h2>

  <article class="order-card" aria-label="Order ORD-001, status: Shipped">
    <h3 class="order-card__id">Order ORD-001</h3>
    <p  class="order-card__status">Shipped</p>
    <p  class="order-card__amount">$128.00</p>
  </article>

  <article class="order-card" aria-label="Order ORD-002, status: Pending">
    <h3 class="order-card__id">Order ORD-002</h3>
    <p  class="order-card__status">Pending</p>
    <p  class="order-card__amount">$64.50</p>
  </article>
</section>
```

---

## Forms

### Complete form example

```html
<form class="order-form" id="create-order-form" novalidate>
  <!--
    novalidate disables browser native validation UI.
    Custom JS validation runs instead, giving full control over error messaging.
  -->

  <fieldset class="order-form__section">
    <legend class="order-form__section-title">Customer details</legend>

    <!-- Text input with hint and error -->
    <div class="form-field">
      <label class="form-field__label" for="customer-id">
        Customer ID
        <span aria-hidden="true" class="form-field__required-marker">*</span>
        <span class="sr-only">(required)</span>
      </label>
      <input
        class="form-field__input"
        id="customer-id"
        name="customerId"
        type="text"
        autocomplete="off"
        required
        aria-required="true"
        aria-describedby="customer-id-hint customer-id-error"
      />
      <p class="form-field__hint" id="customer-id-hint">
        Enter the customer's account number (e.g. CUST-0001).
      </p>
      <!-- hidden until validation fires -->
      <p class="form-field__error" id="customer-id-error" role="alert" aria-live="polite" hidden>
        Customer ID is required.
      </p>
    </div>

    <!-- Email input -->
    <div class="form-field">
      <label class="form-field__label" for="customer-email">Email address</label>
      <input
        class="form-field__input"
        id="customer-email"
        name="email"
        type="email"
        autocomplete="email"
        aria-describedby="customer-email-error"
      />
      <p class="form-field__error" id="customer-email-error" role="alert" aria-live="polite" hidden>
        Please enter a valid email address.
      </p>
    </div>
  </fieldset>

  <fieldset class="order-form__section">
    <legend class="order-form__section-title">Order details</legend>

    <!-- Select -->
    <div class="form-field">
      <label class="form-field__label" for="order-status">
        Status
        <span aria-hidden="true" class="form-field__required-marker">*</span>
        <span class="sr-only">(required)</span>
      </label>
      <select
        class="form-field__select"
        id="order-status"
        name="status"
        required
        aria-required="true"
        aria-describedby="order-status-error"
      >
        <option value="">Select a status</option>
        <option value="PENDING">Pending</option>
        <option value="PROCESSING">Processing</option>
        <option value="SHIPPED">Shipped</option>
        <option value="DELIVERED">Delivered</option>
        <option value="CANCELLED">Cancelled</option>
      </select>
      <p class="form-field__error" id="order-status-error" role="alert" aria-live="polite" hidden>
        Please select a status.
      </p>
    </div>

    <!-- Textarea -->
    <div class="form-field">
      <label class="form-field__label" for="order-notes">Notes</label>
      <textarea
        class="form-field__textarea"
        id="order-notes"
        name="notes"
        rows="4"
        maxlength="500"
        aria-describedby="order-notes-hint"
      ></textarea>
      <p class="form-field__hint" id="order-notes-hint">
        Optional. Maximum 500 characters.
      </p>
    </div>

    <!-- Checkbox group -->
    <fieldset class="form-field">
      <legend class="form-field__label">Notifications</legend>
      <label class="form-field__checkbox-label">
        <input type="checkbox" name="notifyEmail" value="1" />
        Notify customer by email on status change
      </label>
      <label class="form-field__checkbox-label">
        <input type="checkbox" name="notifySms" value="1" />
        Notify customer by SMS
      </label>
    </fieldset>
  </fieldset>

  <!-- Form-level error — shown when multiple fields fail or a server error occurs -->
  <div class="form-error" role="alert" id="form-level-error" hidden>
    <p>Please correct the errors above before submitting.</p>
  </div>

  <div class="form-actions">
    <button type="submit" class="btn btn--primary" aria-describedby="form-level-error">
      Create order
    </button>
    <button type="button" class="btn btn--secondary">
      Cancel
    </button>
  </div>
</form>
```

### Form validation helpers

```javascript
// scripts/utils/form.utils.js

/**
 * Marks a form field as invalid and shows its associated error message.
 * @param {HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement} field
 * @param {string} message
 */
export function showFieldError(field, message) {
  const errorEl = document.getElementById(`${field.id}-error`);

  field.setAttribute('aria-invalid', 'true');
  field.classList.add('form-field__input--error');

  if (errorEl) {
    errorEl.textContent = message;
    errorEl.removeAttribute('hidden');
  }
}

/**
 * Clears the invalid state from a form field.
 * @param {HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement} field
 */
export function clearFieldError(field) {
  const errorEl = document.getElementById(`${field.id}-error`);

  field.removeAttribute('aria-invalid');
  field.classList.remove('form-field__input--error');

  if (errorEl) {
    errorEl.setAttribute('hidden', '');
    errorEl.textContent = '';
  }
}

/**
 * Validates a required field and updates its error state.
 * @param {HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement} field
 * @param {string} [label] - Override label text for the error message
 * @returns {boolean}
 */
export function validateRequired(field, label) {
  const fieldLabel = label ?? field.labels?.[0]?.textContent?.trim() ?? 'This field';

  if (!field.value.trim()) {
    showFieldError(field, `${fieldLabel} is required.`);
    return false;
  }

  clearFieldError(field);
  return true;
}

/**
 * Validates an email field.
 * @param {HTMLInputElement} field
 * @returns {boolean}
 */
export function validateEmail(field) {
  const emailPattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

  if (field.value && !emailPattern.test(field.value)) {
    showFieldError(field, 'Please enter a valid email address.');
    return false;
  }

  clearFieldError(field);
  return true;
}

/**
 * Shows a form-level error message.
 * @param {string} containerId
 * @param {string} message
 */
export function showFormError(containerId, message) {
  const container = document.getElementById(containerId);
  if (!container) return;
  container.textContent = message;
  container.removeAttribute('hidden');
}

/**
 * Hides a form-level error message.
 * @param {string} containerId
 */
export function clearFormError(containerId) {
  const container = document.getElementById(containerId);
  if (!container) return;
  container.setAttribute('hidden', '');
  container.textContent = '';
}
```

### Wiring form validation

```javascript
// scripts/main.js  (excerpt)

import { validateRequired, validateEmail, showFormError, clearFormError } from './utils/form.utils.js';
import { createOrder } from './services/orders.service.js';

const form         = /** @type {HTMLFormElement} */ (document.getElementById('create-order-form'));
const customerIdEl = /** @type {HTMLInputElement} */ (document.getElementById('customer-id'));
const emailEl      = /** @type {HTMLInputElement} */ (document.getElementById('customer-email'));
const statusEl     = /** @type {HTMLSelectElement} */ (document.getElementById('order-status'));

form.addEventListener('submit', async (event) => {
  event.preventDefault();
  clearFormError('form-level-error');

  const isCustomerIdValid = validateRequired(customerIdEl);
  const isEmailValid      = validateEmail(emailEl);
  const isStatusValid     = validateRequired(statusEl);

  if (!isCustomerIdValid || !isEmailValid || !isStatusValid) {
    showFormError('form-level-error', 'Please correct the errors above before submitting.');
    // Move focus to first invalid field
    form.querySelector('[aria-invalid="true"]')?.focus();
    return;
  }

  try {
    await createOrder({
      customerId: customerIdEl.value,
      email:      emailEl.value,
      status:     statusEl.value,
    });
    // handle success — redirect or show confirmation
  } catch (error) {
    showFormError('form-level-error', error instanceof Error ? error.message : 'An error occurred.');
  }
});

// Clear error on input
[customerIdEl, emailEl, statusEl].forEach((field) => {
  field.addEventListener('input', () => {
    field.removeAttribute('aria-invalid');
  });
});
```

---

## Accessibility Patterns

### Focus management

```javascript
// scripts/utils/dom.utils.js

/**
 * Moves keyboard focus to a target element.
 * Adds tabindex="-1" if the element is not normally focusable.
 * @param {string | HTMLElement} target - CSS selector or element reference
 */
export function moveFocus(target) {
  const el = typeof target === 'string'
    ? /** @type {HTMLElement | null} */ (document.querySelector(target))
    : target;

  if (!el) return;

  const wasFocusable = el.hasAttribute('tabindex');
  if (!wasFocusable) el.setAttribute('tabindex', '-1');

  el.focus({ preventScroll: false });

  if (!wasFocusable) {
    el.addEventListener('blur', () => el.removeAttribute('tabindex'), { once: true });
  }
}

// Move focus to main content after client-side navigation
moveFocus('#main-content');

// Move focus to first error after failed form submission
moveFocus(document.querySelector('[aria-invalid="true"]'));
```

### Live regions

```html
<!-- Polite: announced when user is idle (non-urgent updates) -->
<div aria-live="polite" aria-atomic="true" id="status-announcer" class="sr-only"></div>

<!-- Assertive: interrupts immediately (errors, urgent warnings) -->
<div role="alert" aria-live="assertive" id="error-announcer" class="sr-only"></div>
```

```javascript
// scripts/utils/announce.utils.js

/**
 * Announces a message to screen readers via a live region.
 * @param {string} message
 * @param {'polite' | 'assertive'} [priority]
 */
export function announce(message, priority = 'polite') {
  const id = priority === 'assertive' ? 'error-announcer' : 'status-announcer';
  const el = document.getElementById(id);
  if (!el) return;

  // Clear then re-set to force re-announcement even if the message is identical
  el.textContent = '';
  requestAnimationFrame(() => {
    el.textContent = message;
  });
}

// Usage
announce('12 orders loaded.');
announce('Failed to load orders. Please try again.', 'assertive');
```

### Loading state pattern

```html
<section
  class="orders-list"
  aria-label="Orders list"
  aria-busy="true"
  id="orders-section"
>
  <!-- Skeleton shown while loading -->
  <div class="orders-list__skeleton" aria-hidden="true">
    <div class="order-card-skeleton"></div>
    <div class="order-card-skeleton"></div>
    <div class="order-card-skeleton"></div>
  </div>
</section>
```

```javascript
/**
 * Updates the loading state of a section element.
 * @param {HTMLElement} section
 * @param {boolean} isLoading
 */
export function setLoadingState(section, isLoading) {
  section.setAttribute('aria-busy', isLoading ? 'true' : 'false');
}
```

### Focus trap (for modals not using `<dialog>`)

Only needed when using a custom overlay instead of the native `<dialog>` element.
Prefer `<dialog>` — it traps focus automatically.

```javascript
// scripts/utils/focus-trap.utils.js

const FOCUSABLE_SELECTORS = [
  'a[href]', 'button:not([disabled])', 'input:not([disabled])',
  'select:not([disabled])', 'textarea:not([disabled])',
  '[tabindex]:not([tabindex="-1"])',
].join(', ');

/**
 * Traps focus within a container element.
 * @param {HTMLElement} container
 * @returns {{ release: () => void }} Call release() to remove the trap.
 */
export function trapFocus(container) {
  const focusableEls = /** @type {HTMLElement[]} */ ([...container.querySelectorAll(FOCUSABLE_SELECTORS)]);
  const firstEl = focusableEls[0];
  const lastEl  = focusableEls[focusableEls.length - 1];

  function handleKeyDown(event) {
    if (event.key !== 'Tab') return;

    if (event.shiftKey) {
      if (document.activeElement === firstEl) {
        event.preventDefault();
        lastEl.focus();
      }
    } else {
      if (document.activeElement === lastEl) {
        event.preventDefault();
        firstEl.focus();
      }
    }
  }

  container.addEventListener('keydown', handleKeyDown);
  firstEl?.focus();

  return {
    release: () => container.removeEventListener('keydown', handleKeyDown),
  };
}
```

---

## Images and Media

```html
<!-- Informative image — alt describes what it shows -->
<img
  src="/assets/products/widget-pro.jpg"
  alt="Widget Pro — stainless steel housing, 15 cm diameter"
  width="400"
  height="300"
  loading="lazy"
/>

<!-- Decorative image — empty alt, not announced by screen readers -->
<img
  src="/assets/backgrounds/hero-gradient.png"
  alt=""
  width="1440"
  height="600"
  aria-hidden="true"
/>

<!-- Icon alongside text — icon is decorative; text provides the label -->
<button class="btn btn--danger" type="button">
  <svg aria-hidden="true" focusable="false" width="16" height="16">
    <use href="/assets/icons.svg#trash"></use>
  </svg>
  Delete order
</button>

<!-- Icon-only button — aria-label provides the accessible name -->
<button class="btn btn--icon" type="button" aria-label="Delete order ORD-001">
  <svg aria-hidden="true" focusable="false" width="16" height="16">
    <use href="/assets/icons.svg#trash"></use>
  </svg>
</button>

<!-- Responsive image with multiple sizes -->
<picture>
  <source
    media="(min-width: 1024px)"
    srcset="/assets/hero-large.webp 1x, /assets/hero-large@2x.webp 2x"
    type="image/webp"
  />
  <source
    media="(min-width: 1024px)"
    srcset="/assets/hero-large.jpg 1x, /assets/hero-large@2x.jpg 2x"
  />
  <img
    src="/assets/hero-small.jpg"
    alt="Warehouse with orders being processed"
    width="800"
    height="450"
    loading="eager"
  />
</picture>
```

---

## The template Element

`<template>` defines reusable HTML fragments that are inert until cloned. Content inside
is not rendered or executed until it is imported via `content.cloneNode(true)`.

```html
<!-- Define the template in HTML — typically at the end of <body> or in a separate include -->
<template id="order-card-template">
  <article class="order-card">
    <header class="order-card__header">
      <h3 class="order-card__id"></h3>
      <span class="order-card__status-badge"></span>
    </header>
    <dl class="order-card__details">
      <dt class="sr-only">Customer</dt>
      <dd class="order-card__customer"></dd>
      <dt class="sr-only">Total</dt>
      <dd class="order-card__amount"></dd>
    </dl>
    <footer class="order-card__footer">
      <button type="button" class="btn btn--secondary order-card__view-btn">View details</button>
      <button type="button" class="btn btn--danger order-card__delete-btn">Delete</button>
    </footer>
  </article>
</template>
```

```javascript
// scripts/utils/template.utils.js

/**
 * Clones a <template> element and returns its first child element.
 * @template {HTMLElement} T
 * @param {string} templateId
 * @returns {T}
 */
export function cloneTemplate(templateId) {
  const template = /** @type {HTMLTemplateElement | null} */ (document.getElementById(templateId));
  if (!template) throw new Error(`Template #${templateId} not found in DOM.`);
  return /** @type {T} */ (template.content.cloneNode(true).firstElementChild);
}
```

```javascript
// scripts/components/order-card.component.js

import { cloneTemplate } from '../utils/template.utils.js';

/**
 * @typedef {{ orderId: string; customerId: string; status: string; totalAmount: number }} Order
 */

/**
 * Creates and returns an order card element from the #order-card-template.
 * @param {Order} order
 * @param {{ onView: (orderId: string) => void; onDelete: (orderId: string) => void }} handlers
 * @returns {HTMLElement}
 */
export function createOrderCard(order, handlers) {
  const card = cloneTemplate('order-card-template');

  card.setAttribute('aria-label', `Order ${order.orderId}, status: ${order.status}`);
  card.dataset.orderId = order.orderId;

  card.querySelector('.order-card__id').textContent           = order.orderId;
  card.querySelector('.order-card__customer').textContent     = order.customerId;
  card.querySelector('.order-card__amount').textContent       = `$${order.totalAmount.toFixed(2)}`;
  card.querySelector('.order-card__status-badge').textContent = order.status;
  card.querySelector('.order-card__status-badge').className  += ` order-card__status-badge--${order.status.toLowerCase()}`;

  card.querySelector('.order-card__view-btn').addEventListener('click', () => {
    handlers.onView(order.orderId);
  });

  card.querySelector('.order-card__delete-btn').addEventListener('click', () => {
    handlers.onDelete(order.orderId);
  });

  return card;
}
```

```javascript
// Rendering a list of orders using the template
import { createOrderCard } from './components/order-card.component.js';

function renderOrdersList(orders) {
  const container = document.getElementById('orders-section');
  if (!container) return;

  container.innerHTML = '';   // Clear existing content

  if (orders.length === 0) {
    container.innerHTML = '<p class="empty-state">No orders found.</p>';
    return;
  }

  // Build a DocumentFragment for a single DOM insertion (avoids layout thrash)
  const fragment = document.createDocumentFragment();

  orders.forEach((order) => {
    const card = createOrderCard(order, {
      onView:   (id) => navigateTo(`/orders/${id}`),
      onDelete: (id) => openConfirmDeleteDialog(id),
    });
    fragment.appendChild(card);
  });

  container.appendChild(fragment);
}
```

---

## Dialog and Modals

Use the native `<dialog>` element. It provides focus trapping, backdrop, `Escape` to close,
and `aria-modal` semantics without custom JS.

```html
<dialog
  class="modal"
  id="confirm-delete-dialog"
  aria-labelledby="confirm-delete-title"
  aria-describedby="confirm-delete-body"
>
  <article class="modal__content">
    <header class="modal__header">
      <h2 class="modal__title" id="confirm-delete-title">Delete order</h2>
      <button type="button" class="btn btn--icon modal__close-btn" aria-label="Close dialog">
        <svg aria-hidden="true" focusable="false" width="20" height="20"><!-- × icon --></svg>
      </button>
    </header>

    <div class="modal__body" id="confirm-delete-body">
      <p>
        Are you sure you want to delete order
        <strong class="modal__order-id-display"></strong>?
      </p>
      <p>This action cannot be undone.</p>
    </div>

    <footer class="modal__footer">
      <button type="button" class="btn btn--danger" id="confirm-delete-confirm-btn">
        Delete order
      </button>
      <button type="button" class="btn btn--secondary" id="confirm-delete-cancel-btn">
        Cancel
      </button>
    </footer>
  </article>
</dialog>
```

```javascript
// scripts/utils/dialog.utils.js

const dialog     = /** @type {HTMLDialogElement} */ (document.getElementById('confirm-delete-dialog'));
const confirmBtn = document.getElementById('confirm-delete-confirm-btn');
const cancelBtn  = document.getElementById('confirm-delete-cancel-btn');
const closeBtn   = dialog.querySelector('.modal__close-btn');
const orderIdDisplay = dialog.querySelector('.modal__order-id-display');

/** @type {string | null} */
let _pendingOrderId = null;

/** @type {(() => void) | null} */
let _onConfirm = null;

/**
 * Opens the confirm-delete dialog for a given order.
 * @param {string} orderId
 * @param {() => void} onConfirm - Called when the user confirms deletion
 */
export function openConfirmDeleteDialog(orderId, onConfirm) {
  _pendingOrderId = orderId;
  _onConfirm      = onConfirm;

  if (orderIdDisplay) orderIdDisplay.textContent = orderId;
  dialog.showModal();   // Traps focus, sets aria-modal, adds ::backdrop
}

function closeDialog() {
  dialog.close();
  _pendingOrderId = null;
  _onConfirm      = null;
}

confirmBtn?.addEventListener('click', () => {
  _onConfirm?.();
  closeDialog();
});

cancelBtn?.addEventListener('click', closeDialog);
closeBtn?.addEventListener('click', closeDialog);

// Close on backdrop click
dialog.addEventListener('click', (event) => {
  if (event.target === dialog) closeDialog();
});
```

---

## Meta and SEO

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <!-- Primary SEO -->
  <meta name="description" content="Manage your orders, track shipments, and view invoices." />
  <meta name="robots"      content="index, follow" />
  <meta name="author"      content="App Name" />

  <!-- Open Graph — social sharing previews -->
  <meta property="og:type"        content="website" />
  <meta property="og:title"       content="Orders | App Name" />
  <meta property="og:description" content="Manage your orders and track shipments." />
  <meta property="og:url"         content="https://example.com/orders" />
  <meta property="og:image"       content="https://example.com/assets/og-image.png" />
  <meta property="og:image:width"  content="1200" />
  <meta property="og:image:height" content="630" />

  <!-- Twitter / X card -->
  <meta name="twitter:card"        content="summary_large_image" />
  <meta name="twitter:title"       content="Orders | App Name" />
  <meta name="twitter:description" content="Manage your orders and track shipments." />
  <meta name="twitter:image"       content="https://example.com/assets/og-image.png" />

  <!-- Theme colour (mobile browser toolbar) -->
  <meta name="theme-color" content="#1a56db" />

  <!-- Canonical URL — prevents duplicate content issues -->
  <link rel="canonical" href="https://example.com/orders" />

  <!-- Favicon set -->
  <link rel="icon"             type="image/svg+xml" href="/favicon.svg" />
  <link rel="icon"             type="image/png"     href="/favicon-32x32.png" sizes="32x32" />
  <link rel="apple-touch-icon"                      href="/apple-touch-icon.png" sizes="180x180" />
  <link rel="manifest"                              href="/site.webmanifest" />

  <!-- Font preconnect — reduce latency for external fonts -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

  <!-- Critical CSS inline; non-critical loaded async -->
  <style>/* minimal above-the-fold CSS */</style>
  <link rel="preload" as="style" href="/styles/main.css"
        onload="this.onload=null;this.rel='stylesheet'" />

  <title>Orders | App Name</title>
</head>
```
