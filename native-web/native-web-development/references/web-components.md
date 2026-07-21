# Web Components — Full Reference

Complete patterns for Custom Elements v1, Shadow DOM, HTML Templates, slots, and custom events.
For the overview and quick rules see the **Web Components** section in `SKILL.md`.

---

## Table of Contents
1. [Custom Elements Lifecycle](#custom-elements-lifecycle)
2. [Shadow DOM](#shadow-dom)
3. [Observed Attributes](#observed-attributes)
4. [HTML Templates with Slots](#html-templates-with-slots)
5. [Named Slots](#named-slots)
6. [Custom Events from Components](#custom-events-from-components)
7. [Cleanup in disconnectedCallback](#cleanup-in-disconnectedcallback)
8. [Registering Components](#registering-components)
9. [Complete Example — notification-toast](#complete-example--notification-toast)
10. [Complete Example — data-table](#complete-example--data-table)
11. [Patterns and Anti-patterns](#patterns-and-anti-patterns)

---

## Custom Elements Lifecycle

| Callback | When it fires | Use for |
|---|---|---|
| `constructor()` | Element is created or cloned | Attach Shadow DOM, initialise private fields |
| `connectedCallback()` | Element is inserted into the DOM | Fetch data, add event listeners, render |
| `disconnectedCallback()` | Element is removed from the DOM | Remove listeners, abort fetches, cancel timers |
| `attributeChangedCallback(name, oldValue, newValue)` | A watched attribute changes | Re-render or update a specific part of the UI |
| `adoptedCallback()` | Element is moved to a new document | Rarely needed |

```javascript
// scripts/components/base-element.js

/**
 * Base class demonstrating all lifecycle hooks.
 * Do NOT use this as a shared base class in production — extend HTMLElement directly.
 */
export class LifecycleDemo extends HTMLElement {
  /** Attributes to watch for changes. Must be static. */
  static observedAttributes = ['title', 'status', 'count'];

  /** @type {ShadowRoot} */
  #shadow;

  /** @type {AbortController} */
  #abortController;

  constructor() {
    super();   // Always call super() first

    // Attach Shadow DOM in the constructor — before connectedCallback
    this.#shadow = this.attachShadow({ mode: 'open' });
    this.#abortController = new AbortController();
  }

  connectedCallback() {
    // Element is now in the DOM
    this.#render();
    this.#attachListeners();
  }

  disconnectedCallback() {
    // Element is being removed — clean up to prevent memory leaks
    this.#abortController.abort();
    this.#abortController = new AbortController();   // Reset for potential re-connection
  }

  /**
   * Called when an attribute listed in observedAttributes changes.
   * @param {string} name
   * @param {string | null} oldValue
   * @param {string | null} newValue
   */
  attributeChangedCallback(name, oldValue, newValue) {
    if (newValue === oldValue) return;   // Guard against no-op updates
    this.#render();
  }

  #render() {
    const title  = this.getAttribute('title') ?? '';
    const status = this.getAttribute('status') ?? 'idle';

    this.#shadow.innerHTML = `
      <style>
        :host { display: block; }
      </style>
      <div class="component">
        <h3>${title}</h3>
        <span>${status}</span>
      </div>
    `;
  }

  #attachListeners() {
    // Use AbortController signal to auto-remove listeners on disconnectedCallback
    const options = { signal: this.#abortController.signal };
    this.#shadow.addEventListener('click', this.#handleClick.bind(this), options);
  }

  #handleClick(event) {
    // Handle clicks within the shadow root
  }
}
```

---

## Shadow DOM

Shadow DOM encapsulates styles and DOM structure. Styles inside do not leak out; styles outside do not leak in (except CSS custom properties, which cross the boundary).

```javascript
class StatusBadge extends HTMLElement {
  static observedAttributes = ['status', 'label'];

  #shadow;

  constructor() {
    super();
    // mode: 'open' — shadow root accessible via el.shadowRoot (recommended)
    // mode: 'closed' — shadow root inaccessible externally (use rarely)
    this.#shadow = this.attachShadow({ mode: 'open' });
  }

  connectedCallback() { this.#render(); }

  attributeChangedCallback(_name, oldValue, newValue) {
    if (newValue !== oldValue) this.#render();
  }

  #render() {
    const status = this.getAttribute('status') ?? 'pending';
    const label  = this.getAttribute('label') ?? status;

    // Inline styles in the shadow root — encapsulated, won't affect the page
    this.#shadow.innerHTML = `
      <style>
        /* :host refers to the custom element itself */
        :host {
          display:     inline-block;
          font-family: inherit;   /* Inherits from the light DOM */
        }

        /* :host() — apply styles when the host has a specific attribute or class */
        :host([hidden]) { display: none; }
        :host(:focus)   { outline: 2px solid blue; }

        .badge {
          display:        inline-flex;
          align-items:    center;
          padding:        0.25rem 0.625rem;
          border-radius:  9999px;
          font-size:      0.75rem;
          font-weight:    600;
          text-transform: uppercase;
          letter-spacing: 0.05em;
          /* Custom properties cross the shadow boundary */
          font-family:    var(--font-family-base, system-ui, sans-serif);
        }

        .badge--pending   { background: #fef3c7; color: #92400e; }
        .badge--shipped   { background: #d1fae5; color: #065f46; }
        .badge--delivered { background: #dbeafe; color: #1e40af; }
        .badge--cancelled { background: #fee2e2; color: #991b1b; }
      </style>
      <span class="badge badge--${status}" role="status" aria-label="${label}">
        <slot>${label}</slot>
      </span>
    `;
  }
}

customElements.define('status-badge', StatusBadge);
```

```html
<!-- CSS custom properties cross the shadow boundary — useful for theming -->
<style>
  :root { --font-family-base: 'Inter', sans-serif; }
</style>

<status-badge status="shipped">Shipped</status-badge>
```

---

## Observed Attributes

Attributes are always strings. Convert to the appropriate type inside the component.

```javascript
class ProgressRing extends HTMLElement {
  static observedAttributes = ['value', 'max', 'size', 'stroke-width'];

  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
  }

  connectedCallback() { this.#render(); }

  attributeChangedCallback(_name, oldValue, newValue) {
    if (newValue !== oldValue) this.#render();
  }

  // Typed property accessors backed by attributes

  /** @returns {number} */
  get value() { return Number(this.getAttribute('value') ?? 0); }
  /** @param {number} v */
  set value(v) { this.setAttribute('value', String(Math.min(Math.max(0, v), this.max))); }

  /** @returns {number} */
  get max() { return Number(this.getAttribute('max') ?? 100); }
  /** @param {number} v */
  set max(v) { this.setAttribute('max', String(v)); }

  /** @returns {number} */
  get size() { return Number(this.getAttribute('size') ?? 48); }

  /** @returns {number} */
  get strokeWidth() { return Number(this.getAttribute('stroke-width') ?? 4); }

  #render() {
    const radius      = (this.size - this.strokeWidth) / 2;
    const circumference = 2 * Math.PI * radius;
    const progress    = (this.value / this.max) * circumference;
    const offset      = circumference - progress;

    this.#shadow.innerHTML = `
      <style>
        :host { display: inline-block; }
        svg { transform: rotate(-90deg); }
        .track  { fill: none; stroke: #e5e7eb; }
        .fill   { fill: none; stroke: var(--color-primary, #1a56db); stroke-linecap: round; transition: stroke-dashoffset 0.3s ease; }
      </style>
      <svg
        width="${this.size}"
        height="${this.size}"
        viewBox="0 0 ${this.size} ${this.size}"
        aria-label="${this.value}% of ${this.max}"
        role="progressbar"
        aria-valuenow="${this.value}"
        aria-valuemin="0"
        aria-valuemax="${this.max}"
      >
        <circle class="track" cx="${this.size / 2}" cy="${this.size / 2}" r="${radius}" stroke-width="${this.strokeWidth}" />
        <circle
          class="fill"
          cx="${this.size / 2}"
          cy="${this.size / 2}"
          r="${radius}"
          stroke-width="${this.strokeWidth}"
          stroke-dasharray="${circumference}"
          stroke-dashoffset="${offset}"
        />
      </svg>
    `;
  }
}

customElements.define('progress-ring', ProgressRing);
```

```html
<!-- Using property setters via JS -->
<progress-ring id="order-progress" value="65" max="100" size="64"></progress-ring>
<script type="module">
  import './components/progress-ring.component.js';
  const ring = document.getElementById('order-progress');
  ring.value = 80;   // Triggers attributeChangedCallback via the setter
</script>
```

---

## HTML Templates with Slots

`<slot>` elements are placeholders where light DOM content is projected into the shadow root.

### Default slot (unnamed)

```javascript
class AppCard extends HTMLElement {
  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
    // Define the template structure once in the constructor
    this.#shadow.innerHTML = `
      <style>
        :host {
          display:       block;
          border:        1px solid #e5e7eb;
          border-radius: 0.5rem;
          overflow:      hidden;
          background:    #fff;
          box-shadow:    0 1px 3px rgb(0 0 0 / 0.1);
        }
        .card-body { padding: 1rem; }
      </style>
      <div class="card-body">
        <!-- Unnamed slot receives all light DOM children not assigned to a named slot -->
        <slot></slot>
      </div>
    `;
  }
}

customElements.define('app-card', AppCard);
```

```html
<!-- Light DOM children are projected into the unnamed slot -->
<app-card>
  <h3>Order ORD-001</h3>
  <p>Status: Shipped</p>
</app-card>
```

---

## Named Slots

Named slots project specific light DOM elements into designated positions within the shadow root.

```javascript
class OrderCard extends HTMLElement {
  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
    this.#shadow.innerHTML = `
      <style>
        :host {
          display:        flex;
          flex-direction: column;
          border:         1px solid #e5e7eb;
          border-radius:  0.5rem;
          overflow:       hidden;
        }
        .card__header {
          display:         flex;
          align-items:     center;
          justify-content: space-between;
          padding:         0.75rem 1rem;
          border-bottom:   1px solid #e5e7eb;
          background:      #f9fafb;
        }
        .card__body   { padding: 1rem; flex: 1; }
        .card__footer {
          display:      flex;
          gap:          0.5rem;
          padding:      0.75rem 1rem;
          border-top:   1px solid #e5e7eb;
          justify-content: flex-end;
        }
        /* Fallback content shown when a slot is empty */
        .fallback { color: #9ca3af; font-style: italic; }
      </style>
      <header class="card__header">
        <!-- named slot: <element slot="header"> projects here -->
        <slot name="header">
          <span class="fallback">No title</span>
        </slot>
        <slot name="badge"></slot>
      </header>
      <div class="card__body">
        <!-- unnamed slot: all children without a slot attribute project here -->
        <slot></slot>
      </div>
      <footer class="card__footer">
        <slot name="actions">
          <!-- Fallback if no actions slot content is provided -->
          <button type="button">Close</button>
        </slot>
      </footer>
    `;
  }
}

customElements.define('order-card', OrderCard);
```

```html
<!-- Using named slots -->
<order-card>
  <!-- slot="header" → projects into <slot name="header"> -->
  <h3 slot="header">Order ORD-001</h3>

  <!-- slot="badge" → projects into <slot name="badge"> -->
  <status-badge slot="badge" status="shipped">Shipped</status-badge>

  <!-- No slot attribute → projects into the unnamed <slot> -->
  <dl>
    <dt>Customer</dt>
    <dd>CUST-001</dd>
    <dt>Total</dt>
    <dd>$128.00</dd>
  </dl>

  <!-- slot="actions" → projects into <slot name="actions"> -->
  <div slot="actions">
    <button type="button" class="btn btn--secondary">Cancel</button>
    <button type="button" class="btn btn--primary">View details</button>
  </div>
</order-card>
```

---

## Custom Events from Components

Components communicate with their parent by dispatching `CustomEvent` instances.
Never call parent methods directly — that creates tight coupling.

```javascript
class CreateOrderForm extends HTMLElement {
  static observedAttributes = ['submitting'];

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

  #render() {
    this.#shadow.innerHTML = `
      <style>
        :host { display: block; }
        form  { display: flex; flex-direction: column; gap: 1rem; }
      </style>
      <form id="order-form" novalidate>
        <input id="customer-id" name="customerId" type="text" placeholder="Customer ID" required />
        <button type="submit">Create order</button>
        <button type="button" id="cancel-btn">Cancel</button>
      </form>
    `;
  }

  #attachListeners() {
    const options = { signal: this.#abortController.signal };
    const form      = this.#shadow.getElementById('order-form');
    const cancelBtn = this.#shadow.getElementById('cancel-btn');

    form?.addEventListener('submit', (event) => {
      event.preventDefault();
      this.#handleSubmit(/** @type {HTMLFormElement} */ (form));
    }, options);

    cancelBtn?.addEventListener('click', () => this.#emitCancelled(), options);
  }

  /** @param {HTMLFormElement} form */
  #handleSubmit(form) {
    const customerId = /** @type {HTMLInputElement} */ (
      this.#shadow.getElementById('customer-id')
    ).value.trim();

    if (!customerId) return;

    /**
     * Dispatch a 'order:submit' CustomEvent.
     * The parent listens with:
     * formEl.addEventListener('order:submit', (e) => handleOrderSubmit(e.detail))
     */
    this.dispatchEvent(
      new CustomEvent('order:submit', {
        detail:  { customerId },
        bubbles: true,    // Propagates up through the DOM
        composed: true,   // Crosses the shadow boundary — parent can listen on the host element
      }),
    );
  }

  #emitCancelled() {
    this.dispatchEvent(
      new CustomEvent('order:cancel', {
        bubbles:  true,
        composed: true,
      }),
    );
  }
}

customElements.define('create-order-form', CreateOrderForm);
```

```html
<!-- Parent listens to events on the host element -->
<create-order-form id="order-form"></create-order-form>

<script type="module">
  import './components/create-order-form.component.js';
  import { createOrder } from './services/orders.service.js';

  const form = document.getElementById('order-form');

  form.addEventListener('order:submit', async (event) => {
    const { customerId } = event.detail;
    await createOrder({ customerId });
  });

  form.addEventListener('order:cancel', () => {
    history.back();
  });
</script>
```

**`bubbles` and `composed` rules:**
- `bubbles: true` — event propagates up the regular DOM tree
- `composed: true` — event crosses the shadow boundary so parent elements can hear it
- For events that should only be heard by direct listeners on the host, use `bubbles: false, composed: false`

---

## Cleanup in disconnectedCallback

Every listener, timer, and fetch registered in `connectedCallback` must be cleaned up in `disconnectedCallback`. The `AbortController` pattern is the cleanest approach.

```javascript
class OrderPoller extends HTMLElement {
  #shadow;
  #abortController;
  #pollIntervalId;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.#abortController = new AbortController();
    this.#render();
    this.#startPolling();
    this.#attachListeners();
  }

  disconnectedCallback() {
    // 1. Abort any in-flight fetch (http utility respects the signal)
    this.#abortController.abort();

    // 2. Clear any timers
    clearInterval(this.#pollIntervalId);

    // 3. Event listeners attached with { signal } are removed automatically
    //    by the abort above — no need to call removeEventListener manually
  }

  #render() {
    this.#shadow.innerHTML = `
      <style>
        :host { display: block; padding: 1rem; }
        .status { font-size: 0.875rem; color: #6b7280; }
      </style>
      <p class="status" aria-live="polite">Checking for updates…</p>
      <slot></slot>
    `;
  }

  #attachListeners() {
    // Adding { signal } to addEventListener options means the listener
    // is automatically removed when the AbortController aborts
    this.#shadow.addEventListener(
      'click',
      this.#handleClick.bind(this),
      { signal: this.#abortController.signal },
    );
  }

  async #fetchLatest() {
    try {
      const response = await fetch('/api/orders/latest', {
        signal: this.#abortController.signal,
      });
      if (response.ok) {
        const data = await response.json();
        this.#updateDisplay(data);
      }
    } catch (error) {
      if (error instanceof Error && error.name === 'AbortError') return;   // Expected
      console.error('Polling error:', error);
    }
  }

  #startPolling() {
    this.#fetchLatest();  // Immediate first fetch
    this.#pollIntervalId = setInterval(() => this.#fetchLatest(), 30_000);
  }

  /** @param {unknown} data */
  #updateDisplay(data) {
    const statusEl = this.#shadow.querySelector('.status');
    if (statusEl) statusEl.textContent = `Last updated just now`;
  }

  #handleClick() { /* ... */ }
}

customElements.define('order-poller', OrderPoller);
```

---

## Registering Components

Register all custom elements in a single entry point. The browser throws if the same name is registered twice.

```javascript
// scripts/components/index.js — component registry

import { StatusBadgeElement }    from './status-badge.component.js';
import { OrderCardElement }      from './order-card.component.js';
import { CreateOrderFormElement} from './create-order-form.component.js';
import { ProgressRingElement }   from './progress-ring.component.js';
import { AppModalElement }       from './app-modal.component.js';
import { NotificationToast }     from './notification-toast.component.js';

/**
 * Registers all custom elements used by this application.
 * Safe to call multiple times — skips already-registered elements.
 */
export function registerComponents() {
  const definitions = [
    ['status-badge',       StatusBadgeElement],
    ['order-card',         OrderCardElement],
    ['create-order-form',  CreateOrderFormElement],
    ['progress-ring',      ProgressRingElement],
    ['app-modal',          AppModalElement],
    ['notification-toast', NotificationToast],
  ];

  for (const [name, constructor] of definitions) {
    if (!customElements.get(name)) {
      customElements.define(name, constructor);
    }
  }
}
```

```javascript
// scripts/main.js

import { registerComponents } from './components/index.js';

registerComponents();
```

---

## Complete Example — notification-toast

A fully-functional notification toast component with named slots, custom events, auto-dismiss, and cleanup.

```javascript
// scripts/components/notification-toast.component.js

/**
 * @typedef {'info' | 'success' | 'warning' | 'error'} ToastVariant
 */

export class NotificationToast extends HTMLElement {
  static observedAttributes = ['variant', 'duration', 'dismissible'];

  #shadow;
  #abortController;
  /** @type {ReturnType<typeof setTimeout> | null} */
  #dismissTimer = null;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
    this.#abortController = new AbortController();
  }

  connectedCallback() {
    this.#render();
    this.#attachListeners();
    this.#scheduleDismiss();
  }

  disconnectedCallback() {
    this.#abortController.abort();
    if (this.#dismissTimer) clearTimeout(this.#dismissTimer);
  }

  attributeChangedCallback(_name, oldValue, newValue) {
    if (newValue !== oldValue) this.#render();
  }

  /** @returns {ToastVariant} */
  get variant() {
    return /** @type {ToastVariant} */ (this.getAttribute('variant') ?? 'info');
  }

  /** @returns {number} Auto-dismiss delay in ms. 0 means no auto-dismiss. */
  get duration() {
    return Number(this.getAttribute('duration') ?? 5000);
  }

  /** @returns {boolean} */
  get isDismissible() {
    return this.hasAttribute('dismissible');
  }

  #render() {
    const icons = {
      info:    '&#x2139;',
      success: '&#x2713;',
      warning: '&#x26A0;',
      error:   '&#x2715;',
    };

    this.#shadow.innerHTML = `
      <style>
        :host {
          display:  block;
          max-width: 22rem;
          width:    100%;
        }

        .toast {
          display:       flex;
          align-items:   flex-start;
          gap:           0.75rem;
          padding:       0.875rem 1rem;
          border-radius: 0.5rem;
          border:        1px solid transparent;
          box-shadow:    0 4px 6px rgb(0 0 0 / 0.07);
          animation:     slide-in 0.25s cubic-bezier(0.4, 0, 0.2, 1) both;
        }

        @keyframes slide-in {
          from { opacity: 0; transform: translateX(1rem); }
          to   { opacity: 1; transform: translateX(0); }
        }

        .toast--info    { background: #eff6ff; border-color: #bfdbfe; }
        .toast--success { background: #f0fdf4; border-color: #bbf7d0; }
        .toast--warning { background: #fffbeb; border-color: #fde68a; }
        .toast--error   { background: #fef2f2; border-color: #fecaca; }

        .toast__icon {
          flex-shrink: 0;
          font-size:   1.25rem;
          line-height: 1;
          margin-top:  -0.125rem;
        }
        .toast--info .toast__icon    { color: #2563eb; }
        .toast--success .toast__icon { color: #16a34a; }
        .toast--warning .toast__icon { color: #d97706; }
        .toast--error .toast__icon   { color: #dc2626; }

        .toast__body { flex: 1; min-width: 0; }

        .toast__title {
          font-weight: 600;
          font-size:   0.875rem;
          margin: 0 0 0.25rem;
          color: #111827;
        }
        .toast__message {
          font-size: 0.875rem;
          color:     #374151;
          margin:    0;
        }

        .toast__dismiss {
          flex-shrink:  0;
          background:   none;
          border:       none;
          cursor:       pointer;
          padding:      0.125rem;
          border-radius: 0.25rem;
          color:        #9ca3af;
          line-height:  1;
          font-size:    1rem;
          align-self:   flex-start;
        }
        .toast__dismiss:hover { color: #374151; }
        .toast__dismiss:focus-visible {
          outline:        2px solid currentColor;
          outline-offset: 2px;
        }
        .toast__dismiss[hidden] { display: none; }
      </style>

      <div class="toast toast--${this.variant}" role="alert" aria-live="assertive" aria-atomic="true">
        <span class="toast__icon" aria-hidden="true">${icons[this.variant]}</span>
        <div class="toast__body">
          <!-- Named slot: <element slot="title"> -->
          <slot name="title">
            <p class="toast__title">${this.variant.charAt(0).toUpperCase() + this.variant.slice(1)}</p>
          </slot>
          <!-- Default slot: message content -->
          <slot></slot>
        </div>
        <button
          type="button"
          class="toast__dismiss"
          aria-label="Dismiss notification"
          ${this.isDismissible ? '' : 'hidden'}
        >
          &#x2715;
        </button>
      </div>
    `;
  }

  #attachListeners() {
    const options    = { signal: this.#abortController.signal };
    const dismissBtn = this.#shadow.querySelector('.toast__dismiss');

    dismissBtn?.addEventListener('click', () => this.#dismiss(), options);
  }

  #scheduleDismiss() {
    if (this.duration <= 0) return;
    this.#dismissTimer = setTimeout(() => this.#dismiss(), this.duration);
  }

  #dismiss() {
    this.dispatchEvent(
      new CustomEvent('toast:dismissed', {
        bubbles:  true,
        composed: true,
        detail:   { variant: this.variant },
      }),
    );
    this.remove();
  }
}

customElements.define('notification-toast', NotificationToast);
```

```html
<!-- Usage examples -->

<!-- Basic info toast (auto-dismisses after 5s) -->
<notification-toast variant="info" duration="5000" dismissible>
  <span slot="title">Order received</span>
  Your order ORD-001 has been placed successfully.
</notification-toast>

<!-- Success toast (no auto-dismiss, no dismiss button) -->
<notification-toast variant="success" duration="0">
  Payment confirmed for $128.00.
</notification-toast>

<!-- Error toast -->
<notification-toast variant="error" duration="0" dismissible>
  <span slot="title">Failed to update order</span>
  The server returned an error. Please try again.
</notification-toast>
```

```javascript
// Toast manager — creates and stacks toasts programmatically

/**
 * @param {{ variant: ToastVariant; title?: string; message: string; duration?: number }} options
 */
export function showToast({ variant, title, message, duration = 5000 }) {
  let container = document.getElementById('toast-container');

  if (!container) {
    container = document.createElement('div');
    container.id = 'toast-container';
    container.setAttribute('aria-label', 'Notifications');
    Object.assign(container.style, {
      position:   'fixed',
      bottom:     '1.5rem',
      right:      '1.5rem',
      display:    'flex',
      flexDirection: 'column',
      gap:        '0.75rem',
      zIndex:     '500',
      maxWidth:   '22rem',
      width:      '100%',
    });
    document.body.appendChild(container);
  }

  const toast = document.createElement('notification-toast');
  toast.setAttribute('variant', variant);
  toast.setAttribute('duration', String(duration));
  toast.setAttribute('dismissible', '');

  if (title) {
    const titleEl = document.createElement('span');
    titleEl.slot          = 'title';
    titleEl.textContent   = title;
    toast.appendChild(titleEl);
  }

  toast.appendChild(document.createTextNode(message));
  container.appendChild(toast);
}

// Usage
showToast({ variant: 'success', title: 'Saved', message: 'Order status updated.' });
showToast({ variant: 'error',   title: 'Error',  message: 'Failed to save.', duration: 0 });
```

---

## Complete Example — data-table

A sortable, accessible data table Web Component.

```javascript
// scripts/components/data-table.component.js

/**
 * @typedef {{ key: string; label: string; sortable?: boolean }} Column
 * @typedef {Record<string, string | number>} Row
 */

export class DataTable extends HTMLElement {
  static observedAttributes = ['sort-key', 'sort-dir'];

  #shadow;
  /** @type {Column[]} */
  #columns = [];
  /** @type {Row[]} */
  #rows = [];
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

  /**
   * Sets the column definitions.
   * @param {Column[]} columns
   */
  setColumns(columns) {
    this.#columns = columns;
    this.#render();
  }

  /**
   * Sets the row data and re-renders.
   * @param {Row[]} rows
   */
  setRows(rows) {
    this.#rows = rows;
    this.#render();
  }

  get sortKey()  { return this.getAttribute('sort-key') ?? ''; }
  get sortDir()  { return this.getAttribute('sort-dir') ?? 'asc'; }

  #getSortedRows() {
    if (!this.sortKey) return this.#rows;

    return [...this.#rows].sort((a, b) => {
      const aVal = a[this.sortKey] ?? '';
      const bVal = b[this.sortKey] ?? '';
      const cmp  = String(aVal).localeCompare(String(bVal), undefined, { numeric: true });
      return this.sortDir === 'asc' ? cmp : -cmp;
    });
  }

  #render() {
    const rows    = this.#getSortedRows();
    const columns = this.#columns;

    const headerCells = columns.map((col) => {
      const isSorted = col.key === this.sortKey;
      const nextDir  = isSorted && this.sortDir === 'asc' ? 'desc' : 'asc';
      const ariaSorted = isSorted ? ` aria-sort="${this.sortDir === 'asc' ? 'ascending' : 'descending'}"` : '';

      return col.sortable
        ? `<th scope="col"${ariaSorted}>
             <button type="button" class="th-sort" data-key="${col.key}" data-dir="${nextDir}" aria-label="Sort by ${col.label}">
               ${col.label}
               <span aria-hidden="true">${isSorted ? (this.sortDir === 'asc' ? ' ↑' : ' ↓') : ' ↕'}</span>
             </button>
           </th>`
        : `<th scope="col">${col.label}</th>`;
    }).join('');

    const bodyRows = rows.map((row) => {
      const cells = columns.map((col) => `<td>${String(row[col.key] ?? '')}</td>`).join('');
      return `<tr>${cells}</tr>`;
    }).join('');

    this.#shadow.innerHTML = `
      <style>
        :host { display: block; }
        .table-wrapper { overflow-x: auto; }
        table {
          width: 100%;
          border-collapse: collapse;
          font-size: 0.875rem;
        }
        th, td {
          padding: 0.625rem 0.75rem;
          text-align: left;
          border-bottom: 1px solid #e5e7eb;
        }
        th {
          font-weight: 600;
          background:  #f9fafb;
          color:       #374151;
        }
        tr:last-child td { border-bottom: none; }
        tr:hover td { background: #f9fafb; }
        .th-sort {
          background:   none;
          border:       none;
          cursor:       pointer;
          font:         inherit;
          font-weight:  600;
          color:        inherit;
          padding:      0;
          display:      flex;
          align-items:  center;
          gap:          0.25rem;
        }
        .th-sort:focus-visible {
          outline:        2px solid #1a56db;
          outline-offset: 2px;
          border-radius:  3px;
        }
      </style>
      <div class="table-wrapper" role="region" aria-label="Data table" tabindex="0">
        <table>
          <thead><tr>${headerCells}</tr></thead>
          <tbody>${bodyRows || '<tr><td colspan="${columns.length}">No data available.</td></tr>'}</tbody>
        </table>
      </div>
    `;
  }

  #attachListeners() {
    this.#shadow.addEventListener('click', (event) => {
      const btn = /** @type {HTMLElement} */ (event.target).closest('.th-sort');
      if (!btn) return;

      const key = btn.dataset.key ?? '';
      const dir = btn.dataset.dir ?? 'asc';

      this.setAttribute('sort-key', key);
      this.setAttribute('sort-dir', dir);

      this.dispatchEvent(new CustomEvent('table:sort', {
        detail: { key, dir },
        bubbles: true,
        composed: true,
      }));
    }, { signal: this.#abortController.signal });
  }
}

customElements.define('data-table', DataTable);
```

```javascript
// Usage
import './components/data-table.component.js';
import { fetchOrders } from './services/orders.service.js';

const table = document.getElementById('orders-table');

table.setColumns([
  { key: 'orderId',    label: 'Order ID',  sortable: true  },
  { key: 'status',     label: 'Status',    sortable: true  },
  { key: 'totalAmount',label: 'Total',     sortable: true  },
  { key: 'placedAt',   label: 'Placed at', sortable: true  },
  { key: 'actions',    label: 'Actions',   sortable: false },
]);

const orders = await fetchOrders();
table.setRows(orders.map((o) => ({
  orderId:     o.orderId,
  status:      o.status,
  totalAmount: `$${o.totalAmount.toFixed(2)}`,
  placedAt:    new Date(o.placedAt).toLocaleDateString(),
  actions:     'View | Delete',
})));
```

---

## Patterns and Anti-patterns

### Patterns

```javascript
// ✅ Use Shadow DOM for style encapsulation
this.#shadow = this.attachShadow({ mode: 'open' });

// ✅ Clean up in disconnectedCallback using AbortController
connectedCallback() {
  this.#abortController = new AbortController();
  el.addEventListener('click', handler, { signal: this.#abortController.signal });
}
disconnectedCallback() {
  this.#abortController.abort();
}

// ✅ Communicate outward via CustomEvent with composed: true
this.dispatchEvent(new CustomEvent('my:event', { detail: {}, bubbles: true, composed: true }));

// ✅ Guard against rapid re-renders in attributeChangedCallback
attributeChangedCallback(_name, oldValue, newValue) {
  if (newValue === oldValue) return;
  this.#render();
}

// ✅ Use private class fields for internal state
#abortController;
#cache = new Map();
```

### Anti-patterns

```javascript
// ❌ Extending built-in elements — Safari does not support this
class MyButton extends HTMLButtonElement { }
customElements.define('my-button', MyButton, { extends: 'button' });   // Breaks in Safari

// ❌ Direct parent mutation — tight coupling, fragile
document.getElementById('some-external-el').textContent = 'updated';   // Use CustomEvent instead

// ❌ querySelector across shadow boundaries
document.querySelector('my-component .internal-class');   // Returns null — shadow DOM blocks this

// ❌ innerHTML with user content
this.#shadow.innerHTML = `<p>${userContent}</p>`;   // XSS risk — sanitise first

// ✅ Sanitise user content before inserting
const p = document.createElement('p');
p.textContent = userContent;   // textContent escapes HTML — safe
this.#shadow.appendChild(p);

// ❌ Not cleaning up listeners
connectedCallback() {
  document.addEventListener('click', this.#handler);   // Leaks — never removed
}
// ✅ Always clean up
connectedCallback() {
  document.addEventListener('click', this.#handler, { signal: this.#abortController.signal });
}
```
