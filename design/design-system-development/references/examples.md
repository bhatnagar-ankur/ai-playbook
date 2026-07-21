---
author: Ankur Bhatnagar
---

# Examples — Full Reference

Complete worked examples for design system implementation: full token file, Button component (CSS + API doc), Form Field with error state, Modal pattern, and a compound form page.

---

## Table of Contents
1. [Full Token File](#full-token-file)
2. [Button Component — Complete Implementation](#button-component--complete-implementation)
3. [Form Field with Error State](#form-field-with-error-state)
4. [Modal Pattern — Full Implementation](#modal-pattern--full-implementation)
5. [Compound Form Page](#compound-form-page)
6. [Toast System](#toast-system)

---

## Full Token File

A single entry-point CSS file that imports all token partials in the correct order. Global tokens first, then semantic overrides, then component tokens.

```css
/* tokens/index.css */

/* 1. Global (primitive) tokens — raw values */
@import "./global/_color.css";
@import "./global/_spacing.css";
@import "./global/_typography.css";
@import "./global/_shape.css";
@import "./global/_motion.css";
@import "./global/_z-index.css";

/* 2. Semantic tokens — light mode default */
@import "./semantic/_semantic.css";

/* 3. Dark mode overrides */
@import "./semantic/_dark.css";

/* 4. Component tokens */
@import "./components/_button.css";
@import "./components/_input.css";
@import "./components/_badge.css";
@import "./components/_modal.css";
@import "./components/_toast.css";

/* 5. Density variants */
@import "./density/_compact.css";
@import "./density/_spacious.css";
```

Import order in the application entry point:

```css
/* src/styles/main.css */

/* Tokens must load first — all components depend on them */
@import "../design-system/tokens/index.css";

/* Base / reset */
@import "./base/reset.css";
@import "./base/typography.css";

/* Component styles */
@import "./components/button.css";
@import "./components/form-field.css";
@import "./components/badge.css";
@import "./components/modal.css";
@import "./components/toast.css";

/* Page / layout styles */
@import "./layouts/page.css";
@import "./layouts/grid.css";
```

---

## Button Component — Complete Implementation

### CSS

```css
/* components/button.css */

/* ── Base ─────────────────────────────────────────────── */
.btn {
  /* Layout */
  display:         inline-flex;
  align-items:     center;
  justify-content: center;
  gap:             var(--space-2);

  /* Geometry */
  padding:         0 var(--btn-padding-x-md);
  height:          var(--btn-height-md);
  border-radius:   var(--btn-border-radius);
  border:          1px solid transparent;

  /* Typography */
  font-family:     var(--font-family-sans);
  font-size:       var(--btn-font-size-md);
  font-weight:     var(--btn-font-weight);
  line-height:     1;
  text-decoration: none;
  white-space:     nowrap;

  /* Interaction */
  cursor:          pointer;
  user-select:     none;
  transition:      var(--btn-transition);
  position:        relative;
  overflow:        hidden;
}

/* ── Sizes ────────────────────────────────────────────── */
.btn--sm {
  padding:   0 var(--btn-padding-x-sm);
  height:    var(--btn-height-sm);
  font-size: var(--btn-font-size-sm);
}

.btn--lg {
  padding:   0 var(--btn-padding-x-lg);
  height:    var(--btn-height-lg);
  font-size: var(--btn-font-size-lg);
}

.btn--full-width {
  width: 100%;
}

/* ── Primary ──────────────────────────────────────────── */
.btn--primary {
  background-color: var(--btn-primary-bg);
  color:            var(--btn-primary-text);
  border-color:     transparent;
}

.btn--primary:hover:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-primary-bg-hover);
}

.btn--primary:active:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-primary-bg-active);
}

/* ── Secondary ────────────────────────────────────────── */
.btn--secondary {
  background-color: var(--btn-secondary-bg);
  color:            var(--btn-secondary-text);
  border-color:     var(--btn-secondary-border);
}

.btn--secondary:hover:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-secondary-bg-hover);
}

/* ── Ghost ────────────────────────────────────────────── */
.btn--ghost {
  background-color: var(--btn-ghost-bg);
  color:            var(--btn-ghost-text);
  border-color:     transparent;
}

.btn--ghost:hover:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-ghost-bg-hover);
}

/* ── Destructive ──────────────────────────────────────── */
.btn--destructive {
  background-color: var(--btn-destructive-bg);
  color:            var(--btn-destructive-text);
  border-color:     transparent;
}

.btn--destructive:hover:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-destructive-bg-hover);
}

/* ── Link ─────────────────────────────────────────────── */
.btn--link {
  background-color: transparent;
  color:            var(--color-text-link);
  border-color:     transparent;
  height:           auto;
  padding:          0;
  text-decoration:  underline;
  font-weight:      var(--font-weight-regular);
}

.btn--link:hover:not(:disabled):not([aria-disabled="true"]) {
  color: var(--color-text-link-hover);
}

/* ── Focus ────────────────────────────────────────────── */
.btn:focus-visible {
  outline:        3px solid var(--color-border-focus);
  outline-offset: 3px;
}

/* ── Disabled ─────────────────────────────────────────── */
.btn:disabled,
.btn[aria-disabled="true"] {
  opacity:        var(--btn-disabled-opacity);
  cursor:         not-allowed;
  pointer-events: none;
}

/* ── Loading ──────────────────────────────────────────── */
.btn[aria-busy="true"] {
  cursor: wait;
}

.btn[aria-busy="true"] .btn__label {
  opacity: 0;
}

.btn[aria-busy="true"] .btn__icon {
  display: none;
}

.btn__spinner {
  display:       none;
  position:      absolute;
  inset:         0;
  align-items:   center;
  justify-content:center;
}

.btn[aria-busy="true"] .btn__spinner {
  display: flex;
}

.btn__spinner-ring {
  width:          1.2em;
  height:         1.2em;
  border:         2px solid currentColor;
  border-top-color: transparent;
  border-radius:  50%;
  animation:      btn-spin 0.6s linear infinite;
}

@keyframes btn-spin {
  to { transform: rotate(360deg); }
}

/* ── Icon-only ────────────────────────────────────────── */
.btn--icon-only {
  padding:      0;
  width:        var(--btn-height-md);
  aspect-ratio: 1;
}

.btn--icon-only.btn--sm {
  width: var(--btn-height-sm);
}

.btn--icon-only.btn--lg {
  width: var(--btn-height-lg);
}
```

### HTML templates

```html
<!-- Primary button -->
<button class="btn btn--primary" type="button">
  <span class="btn__label">Save changes</span>
  <span class="btn__spinner" aria-hidden="true">
    <span class="btn__spinner-ring"></span>
  </span>
</button>

<!-- Primary with left icon -->
<button class="btn btn--primary" type="button">
  <svg class="btn__icon" aria-hidden="true" width="16" height="16">...</svg>
  <span class="btn__label">Add item</span>
  <span class="btn__spinner" aria-hidden="true">
    <span class="btn__spinner-ring"></span>
  </span>
</button>

<!-- Loading state -->
<button class="btn btn--primary" type="button" aria-busy="true" aria-label="Saving…">
  <span class="btn__label" aria-hidden="true">Save changes</span>
  <span class="btn__spinner" aria-hidden="true">
    <span class="btn__spinner-ring"></span>
  </span>
</button>

<!-- Secondary -->
<button class="btn btn--secondary btn--sm" type="button">
  <span class="btn__label">Cancel</span>
</button>

<!-- Ghost -->
<button class="btn btn--ghost" type="button">
  <span class="btn__label">Learn more</span>
</button>

<!-- Destructive -->
<button class="btn btn--destructive" type="button">
  <span class="btn__label">Delete account</span>
</button>

<!-- Icon-only with aria-label -->
<button class="btn btn--ghost btn--icon-only btn--sm" type="button" aria-label="Close">
  <svg class="btn__icon" aria-hidden="true" width="16" height="16">...</svg>
</button>

<!-- Disabled -->
<button class="btn btn--primary" type="button" disabled aria-disabled="true">
  <span class="btn__label">Submit</span>
</button>

<!-- Link styled as button -->
<a class="btn btn--primary" href="/dashboard">
  <span class="btn__label">Go to dashboard</span>
</a>
```

### Component API documentation (Markdown table format)

| Property | Type | Default | Required | Description |
|---|---|---|---|---|
| `variant` | `"primary" \| "secondary" \| "ghost" \| "destructive" \| "link"` | `"primary"` | No | Visual style |
| `size` | `"sm" \| "md" \| "lg"` | `"md"` | No | Height and padding scale |
| `type` | `"button" \| "submit" \| "reset"` | `"button"` | No | Native button type |
| `disabled` | `boolean` | `false` | No | Prevents all interaction |
| `loading` | `boolean` | `false` | No | Shows spinner; sets `aria-busy` |
| `loadingLabel` | `string` | `undefined` | No | SR label during loading (e.g. "Saving…") |
| `fullWidth` | `boolean` | `false` | No | Stretches to 100% of container |
| `leftIcon` | `ReactNode` | `undefined` | No | Icon rendered before the label |
| `rightIcon` | `ReactNode` | `undefined` | No | Icon rendered after the label |
| `href` | `string` | `undefined` | No | Renders as `<a>` when provided |
| `onClick` | `(e: MouseEvent) => void` | `undefined` | No | Click handler |
| `form` | `string` | `undefined` | No | Associates with a form by ID |
| `className` | `string` | `undefined` | No | Additional CSS classes |

---

## Form Field with Error State

Full end-to-end example showing a form field in error state with all accessibility hooks wired up.

### CSS (supplement to components/form-field.css)

```css
/* The textarea variant shares the input token set */
.form-field__textarea {
  width:            100%;
  min-height:       var(--space-24);    /* 96 px — 6 lines approx */
  padding:          var(--space-2) var(--input-padding-x);
  border:           var(--border-width-thin) solid var(--input-border);
  border-radius:    var(--input-border-radius);
  background-color: var(--input-bg);
  color:            var(--input-text);
  font-family:      var(--font-family-sans);
  font-size:        var(--input-font-size);
  line-height:      var(--line-height-normal);
  resize:           vertical;
  transition:       border-color var(--duration-normal) var(--easing-ease-in-out),
                    box-shadow   var(--duration-normal) var(--easing-ease-in-out);
}

.form-field__textarea:focus-visible {
  outline:      none;
  border-color: var(--input-border-focus);
  box-shadow:   0 0 0 3px rgba(59, 130, 246, 0.25);
}

.form-field--error .form-field__textarea {
  border-color: var(--input-border-error);
}
```

### HTML — Text input in error state

```html
<div class="form-field form-field--error">
  <!--
    form-field--error class:
    - Changes border to --input-border-error (red)
    - Styles the error message text in --color-feedback-error-text
  -->

  <label class="form-field__label form-field__label--required" for="email">
    Email address
  </label>

  <p class="form-field__hint" id="email-hint">
    We'll send order updates here.
  </p>

  <div class="form-field__control-wrapper">
    <input
      class="form-field__input"
      type="email"
      id="email"
      name="email"
      required
      aria-required="true"
      aria-invalid="true"
      aria-describedby="email-hint email-error"
      autocomplete="email"
      value="not-an-email"
    />
  </div>

  <p class="form-field__error" id="email-error" role="alert">
    <svg class="form-field__error-icon" aria-hidden="true" viewBox="0 0 16 16">
      <path d="M8 1a7 7 0 1 0 0 14A7 7 0 0 0 8 1zm0 4a.75.75 0 0 1 .75.75v3a.75.75 0 0 1-1.5 0v-3A.75.75 0 0 1 8 5zm0 6.5a.875.875 0 1 1 0-1.75.875.875 0 0 1 0 1.75z" />
    </svg>
    Please enter a valid email address.
  </p>
</div>
```

### HTML — Select with hint

```html
<div class="form-field">
  <label class="form-field__label form-field__label--required" for="country">
    Country
  </label>

  <p class="form-field__hint" id="country-hint">
    Select the country where you want delivery.
  </p>

  <div class="form-field__control-wrapper">
    <select
      class="form-field__input"
      id="country"
      name="country"
      required
      aria-required="true"
      aria-describedby="country-hint"
    >
      <option value="" disabled selected>Select a country…</option>
      <option value="us">United States</option>
      <option value="gb">United Kingdom</option>
      <option value="ca">Canada</option>
      <option value="au">Australia</option>
    </select>
  </div>
</div>
```

### HTML — Textarea with character counter

```html
<div class="form-field">
  <label class="form-field__label" for="notes">
    Order notes
    <span class="form-field__label-optional">(optional)</span>
  </label>

  <div class="form-field__control-wrapper">
    <textarea
      class="form-field__textarea"
      id="notes"
      name="notes"
      maxlength="500"
      aria-describedby="notes-counter"
      rows="4"
    ></textarea>
  </div>

  <span class="form-field__counter" id="notes-counter" aria-live="polite">
    <!-- Updated by JS as the user types: "320 / 500" -->
    0 / 500
  </span>
</div>
```

---

## Modal Pattern — Full Implementation

A complete, accessible modal pattern with focus trap, scroll lock, and keyboard handling.

```html
<!-- Trigger button -->
<button class="btn btn--primary" type="button" id="delete-trigger">
  <span class="btn__label">Delete account</span>
</button>

<!-- Modal — rendered in a portal to <body> -->
<div class="modal-overlay" id="delete-overlay" hidden aria-hidden="true">
  <div
    class="modal modal--sm"
    role="dialog"
    aria-modal="true"
    aria-labelledby="delete-dialog-title"
    aria-describedby="delete-dialog-desc"
    tabindex="-1"
    id="delete-dialog"
  >
    <!-- Header -->
    <div class="modal__header">
      <div>
        <h2 class="modal__title" id="delete-dialog-title">
          Delete account
        </h2>
        <p class="modal__description" id="delete-dialog-desc">
          This action is permanent and cannot be undone.
        </p>
      </div>
      <button
        class="modal__close btn btn--ghost btn--icon-only btn--sm"
        type="button"
        aria-label="Close dialog"
      >
        <svg aria-hidden="true" width="16" height="16" viewBox="0 0 16 16">
          <path d="M3.72 3.72a.75.75 0 0 1 1.06 0L8 6.94l3.22-3.22a.75.75 0 1 1 1.06 1.06L9.06 8l3.22 3.22a.75.75 0 1 1-1.06 1.06L8 9.06l-3.22 3.22a.75.75 0 0 1-1.06-1.06L6.94 8 3.72 4.78a.75.75 0 0 1 0-1.06z"/>
        </svg>
      </button>
    </div>

    <!-- Body -->
    <div class="modal__body">
      <p>
        Are you sure you want to delete your account?
        All of your data, including orders and saved addresses, will be permanently removed.
      </p>
      <p>
        Type <strong>delete</strong> below to confirm.
      </p>

      <div class="form-field" style="margin-top: var(--space-4);">
        <label class="form-field__label" for="confirm-delete">
          Confirm deletion
        </label>
        <div class="form-field__control-wrapper">
          <input
            class="form-field__input"
            type="text"
            id="confirm-delete"
            name="confirm"
            placeholder='Type "delete"'
          />
        </div>
      </div>
    </div>

    <!-- Footer -->
    <div class="modal__footer">
      <button class="btn btn--secondary" type="button" id="delete-cancel">
        <span class="btn__label">Cancel</span>
      </button>
      <button class="btn btn--destructive" type="submit" id="delete-confirm" disabled>
        <span class="btn__label">Delete account</span>
        <span class="btn__spinner" aria-hidden="true">
          <span class="btn__spinner-ring"></span>
        </span>
      </button>
    </div>
  </div>
</div>
```

```javascript
// modal.js — lightweight no-dependency modal manager

class Modal {
  constructor(triggerId, overlayId, dialogId) {
    this.trigger = document.getElementById(triggerId);
    this.overlay = document.getElementById(overlayId);
    this.dialog  = document.getElementById(dialogId);
    this._prevFocus = null;
    this._cleanup   = null;

    this.trigger?.addEventListener('click', () => this.open());
    this.overlay?.addEventListener('click', (e) => {
      if (e.target === this.overlay) this.close();
    });
    this.dialog?.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') this.close();
    });

    const closeBtn = this.dialog?.querySelector('[aria-label="Close dialog"]');
    closeBtn?.addEventListener('click', () => this.close());
  }

  open() {
    this._prevFocus = document.activeElement;

    // Unlock overlay
    this.overlay.removeAttribute('hidden');
    this.overlay.setAttribute('aria-hidden', 'false');

    // Lock body scroll
    document.body.style.overflow = 'hidden';

    // Trap focus
    this._cleanup = trapFocus(this.dialog);

    // Announce to screen readers
    this.dialog.setAttribute('aria-hidden', 'false');
  }

  close() {
    this.overlay.setAttribute('hidden', '');
    this.overlay.setAttribute('aria-hidden', 'true');

    // Restore scroll
    document.body.style.overflow = '';

    // Remove focus trap
    this._cleanup?.();
    this._cleanup = null;

    // Restore focus
    this._prevFocus?.focus();
  }
}

// Focus trap implementation (see accessibility.md for full version)
function trapFocus(container) {
  const FOCUSABLE = 'button:not([disabled]), input:not([disabled]), [tabindex]:not([tabindex="-1"])';
  const els = () => [...container.querySelectorAll(FOCUSABLE)];

  function onKeyDown(e) {
    if (e.key !== 'Tab') return;
    const focusable = els();
    const first = focusable[0];
    const last  = focusable[focusable.length - 1];

    if (e.shiftKey) {
      if (document.activeElement === first) { e.preventDefault(); last.focus(); }
    } else {
      if (document.activeElement === last)  { e.preventDefault(); first.focus(); }
    }
  }

  container.addEventListener('keydown', onKeyDown);
  els()[0]?.focus();
  return () => container.removeEventListener('keydown', onKeyDown);
}

// Wire up
const deleteModal = new Modal('delete-trigger', 'delete-overlay', 'delete-dialog');

// Enable confirm button only when input matches "delete"
const confirmInput  = document.getElementById('confirm-delete');
const confirmButton = document.getElementById('delete-confirm');

confirmInput?.addEventListener('input', () => {
  const ready = confirmInput.value.toLowerCase().trim() === 'delete';
  confirmButton.disabled = !ready;
  confirmButton.setAttribute('aria-disabled', String(!ready));
});
```

---

## Compound Form Page

A complete shipping address form showing how tokens, form fields, buttons, and validation work together.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Shipping address — Checkout</title>
  <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>

  <!-- Skip link -->
  <a href="#main-content" class="skip-link">Skip to main content</a>

  <!-- SR live regions (injected in <head> so AT registers them early) -->
  <div id="status-region" role="status"  aria-live="polite"    aria-atomic="true" class="sr-only"></div>
  <div id="alert-region"  role="alert"   aria-live="assertive" aria-atomic="true" class="sr-only"></div>

  <header>
    <nav aria-label="Checkout steps">
      <ol class="step-list">
        <li aria-current="step">
          <span aria-label="Step 1 of 3, current step: Shipping">1. Shipping</span>
        </li>
        <li aria-label="Step 2 of 3: Payment">2. Payment</li>
        <li aria-label="Step 3 of 3: Review" aria-disabled="true">3. Review</li>
      </ol>
    </nav>
  </header>

  <main id="main-content" tabindex="-1">
    <h1>Shipping address</h1>

    <form id="shipping-form" novalidate>

      <!-- First / Last name row -->
      <div class="form-row">
        <div class="form-field">
          <label class="form-field__label form-field__label--required" for="first-name">
            First name
          </label>
          <div class="form-field__control-wrapper">
            <input
              class="form-field__input"
              type="text"
              id="first-name"
              name="firstName"
              required
              aria-required="true"
              autocomplete="given-name"
            />
          </div>
        </div>

        <div class="form-field">
          <label class="form-field__label form-field__label--required" for="last-name">
            Last name
          </label>
          <div class="form-field__control-wrapper">
            <input
              class="form-field__input"
              type="text"
              id="last-name"
              name="lastName"
              required
              aria-required="true"
              autocomplete="family-name"
            />
          </div>
        </div>
      </div>

      <!-- Address line 1 -->
      <div class="form-field">
        <label class="form-field__label form-field__label--required" for="address1">
          Address line 1
        </label>
        <div class="form-field__control-wrapper">
          <input
            class="form-field__input"
            type="text"
            id="address1"
            name="address1"
            required
            aria-required="true"
            autocomplete="address-line1"
          />
        </div>
      </div>

      <!-- Address line 2 -->
      <div class="form-field">
        <label class="form-field__label" for="address2">
          Address line 2
          <span class="form-field__label-optional">(optional)</span>
        </label>
        <div class="form-field__control-wrapper">
          <input
            class="form-field__input"
            type="text"
            id="address2"
            name="address2"
            autocomplete="address-line2"
          />
        </div>
      </div>

      <!-- City / State / ZIP row -->
      <div class="form-row form-row--3-col">
        <div class="form-field">
          <label class="form-field__label form-field__label--required" for="city">
            City
          </label>
          <div class="form-field__control-wrapper">
            <input
              class="form-field__input"
              type="text"
              id="city"
              name="city"
              required
              aria-required="true"
              autocomplete="address-level2"
            />
          </div>
        </div>

        <div class="form-field">
          <label class="form-field__label form-field__label--required" for="state">
            State
          </label>
          <div class="form-field__control-wrapper">
            <select
              class="form-field__input"
              id="state"
              name="state"
              required
              aria-required="true"
              autocomplete="address-level1"
            >
              <option value="" disabled selected>Select…</option>
              <option value="CA">California</option>
              <option value="NY">New York</option>
              <option value="TX">Texas</option>
            </select>
          </div>
        </div>

        <div class="form-field">
          <label class="form-field__label form-field__label--required" for="zip">
            ZIP code
          </label>
          <div class="form-field__control-wrapper">
            <input
              class="form-field__input"
              type="text"
              id="zip"
              name="zip"
              required
              aria-required="true"
              inputmode="numeric"
              maxlength="10"
              autocomplete="postal-code"
            />
          </div>
        </div>
      </div>

      <!-- Phone -->
      <div class="form-field">
        <label class="form-field__label form-field__label--required" for="phone">
          Phone number
        </label>
        <p class="form-field__hint" id="phone-hint">
          For delivery updates only. Not shared with third parties.
        </p>
        <div class="form-field__control-wrapper">
          <input
            class="form-field__input"
            type="tel"
            id="phone"
            name="phone"
            required
            aria-required="true"
            aria-describedby="phone-hint"
            autocomplete="tel"
            inputmode="tel"
          />
        </div>
      </div>

      <!-- Actions -->
      <div class="form-actions">
        <a class="btn btn--secondary" href="/cart">
          <span class="btn__label">Back to cart</span>
        </a>
        <button class="btn btn--primary" type="submit" id="submit-btn">
          <span class="btn__label">Continue to payment</span>
          <span class="btn__spinner" aria-hidden="true">
            <span class="btn__spinner-ring"></span>
          </span>
        </button>
      </div>

    </form>
  </main>

</body>
</html>
```

```css
/* Page-level layout (not design system — application CSS) */

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: var(--space-4);
}

.form-row--3-col {
  grid-template-columns: 1fr 1fr auto;
}

@media (max-width: 640px) {
  .form-row,
  .form-row--3-col {
    grid-template-columns: 1fr;
  }
}

main {
  max-width:  640px;
  margin:     0 auto;
  padding:    var(--space-8) var(--space-4);
}

form {
  display:        flex;
  flex-direction: column;
  gap:            var(--space-5);
  margin-top:     var(--space-6);
}

.form-actions {
  display:         flex;
  justify-content: space-between;
  align-items:     center;
  gap:             var(--space-3);
  padding-top:     var(--space-2);
  border-top:      1px solid var(--color-border-default);
  margin-top:      var(--space-2);
}

h1 {
  font-size:   var(--font-size-2xl);
  font-weight: var(--font-weight-semibold);
  color:       var(--color-text-primary);
  line-height: var(--line-height-tight);
}
```

---

## Toast System

```javascript
// toast-system.js — framework-agnostic toast manager

class ToastSystem {
  constructor() {
    this._container = this._createContainer();
    this._queue     = [];
    this._idCounter = 0;
  }

  _createContainer() {
    const el = document.createElement('div');
    el.className   = 'toast-container';
    el.setAttribute('role', 'region');
    el.setAttribute('aria-label', 'Notifications');
    document.body.appendChild(el);
    return el;
  }

  _createToast({ variant = 'info', title, description, duration = 5000, action, closeable = true }) {
    const id   = `toast-${++this._idCounter}`;
    const el   = document.createElement('div');
    el.className = `toast toast--${variant}`;
    el.id        = id;
    el.setAttribute(
      variant === 'error' ? 'role' : 'role',
      variant === 'error' ? 'alert' : 'status'
    );

    el.innerHTML = `
      <span class="toast__icon" aria-hidden="true">${this._icon(variant)}</span>
      <div class="toast__content">
        <p class="toast__title">${title}</p>
        ${description ? `<p class="toast__description">${description}</p>` : ''}
        ${action ? `<button class="toast__action" type="button">${action.label}</button>` : ''}
      </div>
      ${closeable ? `
        <button class="toast__close" type="button" aria-label="Dismiss notification">
          <svg aria-hidden="true" viewBox="0 0 16 16" width="12" height="12">
            <path d="M3.72 3.72a.75.75 0 0 1 1.06 0L8 6.94l3.22-3.22a.75.75 0 1 1 1.06 1.06L9.06 8l3.22 3.22a.75.75 0 1 1-1.06 1.06L8 9.06l-3.22 3.22a.75.75 0 0 1-1.06-1.06L6.94 8 3.72 4.78a.75.75 0 0 1 0-1.06z"/>
          </svg>
        </button>
      ` : ''}
    `;

    // Close button
    el.querySelector('.toast__close')?.addEventListener('click', () => this.dismiss(id));

    // Action button
    if (action) {
      el.querySelector('.toast__action')?.addEventListener('click', () => {
        action.onClick();
        this.dismiss(id);
      });
    }

    // Pause auto-dismiss on hover / focus
    let timer;
    const startTimer = () => {
      if (duration === null) return;
      timer = setTimeout(() => this.dismiss(id), duration);
    };
    const clearTimer = () => clearTimeout(timer);

    el.addEventListener('mouseenter', clearTimer);
    el.addEventListener('mouseleave', startTimer);
    el.addEventListener('focusin',    clearTimer);
    el.addEventListener('focusout',   startTimer);

    startTimer();
    return { el, id };
  }

  _icon(variant) {
    const icons = {
      success: '<svg viewBox="0 0 16 16" width="20" height="20"><path fill="currentColor" d="M8 1a7 7 0 1 0 0 14A7 7 0 0 0 8 1zm3.78 4.22-4.5 4.5a.75.75 0 0 1-1.06 0l-2-2a.75.75 0 1 1 1.06-1.06L6.75 8.19l3.97-3.97a.75.75 0 1 1 1.06 1.06z"/></svg>',
      error:   '<svg viewBox="0 0 16 16" width="20" height="20"><path fill="currentColor" d="M8 1a7 7 0 1 0 0 14A7 7 0 0 0 8 1zm0 4a.75.75 0 0 1 .75.75v3a.75.75 0 0 1-1.5 0v-3A.75.75 0 0 1 8 5zm0 7a.875.875 0 1 1 0-1.75A.875.875 0 0 1 8 12z"/></svg>',
      warning: '<svg viewBox="0 0 16 16" width="20" height="20"><path fill="currentColor" d="M8.22.8a.25.25 0 0 0-.44 0L.55 13.7c-.1.17.02.38.22.38h14.46c.2 0 .32-.21.22-.38L8.22.8zM8 5.5a.75.75 0 0 1 .75.75v3a.75.75 0 0 1-1.5 0v-3A.75.75 0 0 1 8 5.5zm0 7a.875.875 0 1 1 0-1.75A.875.875 0 0 1 8 12.5z"/></svg>',
      info:    '<svg viewBox="0 0 16 16" width="20" height="20"><path fill="currentColor" d="M8 1a7 7 0 1 0 0 14A7 7 0 0 0 8 1zm0 3.5a.875.875 0 1 1 0 1.75A.875.875 0 0 1 8 4.5zm0 3a.75.75 0 0 1 .75.75v4a.75.75 0 0 1-1.5 0v-4A.75.75 0 0 1 8 7.5z"/></svg>',
    };
    return icons[variant] ?? icons.info;
  }

  show(options) {
    const { el, id } = this._createToast(options);
    this._container.appendChild(el);
    return id;
  }

  dismiss(id) {
    const el = document.getElementById(id);
    if (!el) return;

    el.classList.add('is-dismissing');
    el.addEventListener('animationend', () => el.remove(), { once: true });
  }

  success(title, options = {}) { return this.show({ ...options, variant: 'success', title }); }
  error(title, options = {})   { return this.show({ ...options, variant: 'error',   title }); }
  warning(title, options = {}) { return this.show({ ...options, variant: 'warning', title }); }
  info(title, options = {})    { return this.show({ ...options, variant: 'info',    title }); }
}

// Singleton
const toast = new ToastSystem();

// Usage examples
toast.success('Order placed!', { description: 'You'll receive a confirmation email shortly.' });
toast.error('Payment failed', { description: 'Please check your card details and try again.', duration: null });
toast.info('3 items updated', {
  action: { label: 'View changes', onClick: () => console.log('view') },
});
```
