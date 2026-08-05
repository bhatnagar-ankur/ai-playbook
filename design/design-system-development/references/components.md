---
author: Ankur Bhatnagar
---

# Component API Patterns — Full Reference

Deep-dive component contracts: Button, Form Field, Badge, Modal, Toast. Each entry covers variants, sizes, states, full API, accessibility, and CSS implementation.

---

## Table of Contents
1. [Button](#button)
2. [Form Field](#form-field)
3. [Badge](#badge)
4. [Modal / Dialog](#modal--dialog)
5. [Toast / Notification](#toast--notification)
6. [Component Pattern Rules](#component-pattern-rules)

---

## Button

### When to use
Trigger an action. Use `<button>` — never `<a>` — when no URL is involved.
Use `<a>` styled as a button only when navigating to a URL.

### Variants

| Variant | Use |
|---|---|
| `primary` | One per page section; highest visual priority |
| `secondary` | Secondary actions adjacent to a primary button |
| `ghost` | Tertiary actions; sits in toolbars and card headers |
| `destructive` | Irreversible actions (delete, remove, cancel subscription) |
| `link` | Inline text-level action; renders as underlined text |

### Sizes

| Size | Height | Font | Use |
|---|---|---|---|
| `sm` | 32 px | 14 px | Dense tables, toolbars |
| `md` (default) | 40 px | 16 px | Most UI |
| `lg` | 48 px | 18 px | Landing pages, hero CTAs |

### States

`default` → `hover` → `active` → `focus-visible` → `disabled` → `loading`

### API

| Prop | Type | Default | Description |
|---|---|---|---|
| `variant` | `"primary" \| "secondary" \| "ghost" \| "destructive" \| "link"` | `"primary"` | Visual style |
| `size` | `"sm" \| "md" \| "lg"` | `"md"` | Height and padding scale |
| `disabled` | `boolean` | `false` | Prevents interaction |
| `loading` | `boolean` | `false` | Shows spinner; prevents interaction |
| `loadingText` | `string` | `undefined` | SR-only label while loading |
| `fullWidth` | `boolean` | `false` | Stretches to fill container |
| `leftIcon` | `ReactNode \| string` | `undefined` | Icon before label |
| `rightIcon` | `ReactNode \| string` | `undefined` | Icon after label |
| `type` | `"button" \| "submit" \| "reset"` | `"button"` | Maps to HTML `type` attribute |
| `form` | `string` | `undefined` | Associates with a `<form>` by ID |
| `onClick` | `(event: MouseEvent) => void` | `undefined` | Click handler |
| `href` | `string` | `undefined` | Renders as `<a>` when set |
| `target` | `string` | `undefined` | `<a>` target; requires `href` |
| `rel` | `string` | `undefined` | `<a>` rel; requires `href` |

### Accessibility
- Never disable focus styles — use `:focus-visible` to hide on mouse, keep for keyboard.
- `disabled` renders `aria-disabled="true"` and `tabindex="-1"`.
- `loading` adds `aria-busy="true"` and `aria-label` from `loadingText`.
- Icon-only buttons must have `aria-label`.
- Destructive buttons: confirm via dialog before executing, never on single click.

### CSS Implementation

```css
/* components/button.css */

.btn {
  display:         inline-flex;
  align-items:     center;
  justify-content: center;
  gap:             var(--space-2);
  padding:         0 var(--btn-padding-x-md);
  height:          var(--btn-height-md);
  border-radius:   var(--btn-border-radius);
  font-size:       var(--btn-font-size-md);
  font-weight:     var(--btn-font-weight);
  line-height:     1;
  white-space:     nowrap;
  cursor:          pointer;
  border:          1px solid transparent;
  transition:      var(--btn-transition);
  text-decoration: none;
  user-select:     none;
}

/* Sizes */
.btn--sm {
  height:       var(--btn-height-sm);
  padding:      0 var(--btn-padding-x-sm);
  font-size:    var(--btn-font-size-sm);
}

.btn--lg {
  height:       var(--btn-height-lg);
  padding:      0 var(--btn-padding-x-lg);
  font-size:    var(--btn-font-size-lg);
}

/* Primary */
.btn--primary {
  background-color: var(--btn-primary-bg);
  color:            var(--btn-primary-text);
  border-color:     var(--btn-primary-border);
}

.btn--primary:hover:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-primary-bg-hover);
}

.btn--primary:active:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-primary-bg-active);
}

/* Secondary */
.btn--secondary {
  background-color: var(--btn-secondary-bg);
  color:            var(--btn-secondary-text);
  border-color:     var(--btn-secondary-border);
}

.btn--secondary:hover:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-secondary-bg-hover);
}

/* Ghost */
.btn--ghost {
  background-color: var(--btn-ghost-bg);
  color:            var(--btn-ghost-text);
  border-color:     var(--btn-ghost-border);
}

.btn--ghost:hover:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-ghost-bg-hover);
}

/* Destructive */
.btn--destructive {
  background-color: var(--btn-destructive-bg);
  color:            var(--btn-destructive-text);
  border-color:     var(--btn-destructive-border);
}

.btn--destructive:hover:not(:disabled):not([aria-disabled="true"]) {
  background-color: var(--btn-destructive-bg-hover);
}

/* Focus */
.btn:focus-visible {
  outline:    3px solid var(--color-border-focus);
  outline-offset: 2px;
}

/* Disabled */
.btn:disabled,
.btn[aria-disabled="true"] {
  opacity: var(--btn-disabled-opacity);
  cursor:  not-allowed;
  pointer-events: none;
}

/* Loading */
.btn--loading {
  position: relative;
  cursor:   wait;
}

.btn--loading .btn__label {
  opacity: 0;
}

.btn--loading::after {
  content:  "";
  position: absolute;
  width:    1em;
  height:   1em;
  border:   2px solid currentColor;
  border-top-color: transparent;
  border-radius:    50%;
  animation: btn-spin 0.6s linear infinite;
}

@keyframes btn-spin {
  to { transform: rotate(360deg); }
}

/* Full-width */
.btn--full-width {
  width: 100%;
}
```

---

## Form Field

### Anatomy
`label` + optional `hint` + `control` (`input` / `textarea` / `select`) + optional `error-message`

### States
`default` → `hover` → `focus` → `filled` → `error` → `disabled` → `read-only`

### API

| Prop | Type | Default | Description |
|---|---|---|---|
| `id` | `string` | required | Associates `<label>` with control |
| `label` | `string` | required | Visible label text |
| `hint` | `string` | `undefined` | Secondary helper below label |
| `error` | `string` | `undefined` | Inline error; sets `aria-invalid` |
| `disabled` | `boolean` | `false` | Disables control |
| `readOnly` | `boolean` | `false` | Prevents edits |
| `required` | `boolean` | `false` | Shows asterisk; sets `aria-required` |
| `size` | `"sm" \| "md" \| "lg"` | `"md"` | |
| `value` | `string` | `undefined` | Controlled value |
| `defaultValue` | `string` | `undefined` | Uncontrolled default |
| `onChange` | `(value: string) => void` | `undefined` | Change handler |
| `onBlur` | `(event: FocusEvent) => void` | `undefined` | Blur handler |
| `placeholder` | `string` | `undefined` | |
| `maxLength` | `number` | `undefined` | Shows character counter when set |
| `prefix` | `ReactNode` | `undefined` | Icon or text before input |
| `suffix` | `ReactNode` | `undefined` | Icon, button, or text after input |
| `autoComplete` | `string` | `undefined` | Maps to HTML `autocomplete` |

### CSS Implementation

```css
/* components/form-field.css */

.form-field {
  display:        flex;
  flex-direction: column;
  gap:            var(--space-1-5);
}

/* Label */
.form-field__label {
  font-size:   var(--font-size-sm);
  font-weight: var(--font-weight-medium);
  color:       var(--color-text-primary);
  line-height: var(--line-height-tight);
}

.form-field__label--required::after {
  content: " *";
  color:   var(--color-feedback-error);
  aria-hidden: true;   /* CSS pseudo-element hidden from AT; aria-required on input */
}

/* Hint */
.form-field__hint {
  font-size:  var(--font-size-sm);
  color:      var(--color-text-secondary);
  line-height: var(--line-height-normal);
}

/* Input wrapper (prefix/suffix support) */
.form-field__control-wrapper {
  position:    relative;
  display:     flex;
  align-items: center;
}

/* Input */
.form-field__input {
  width:            100%;
  height:           var(--input-height-md);
  padding:          0 var(--input-padding-x);
  border:           var(--border-width-thin) solid var(--input-border);
  border-radius:    var(--input-border-radius);
  background-color: var(--input-bg);
  color:            var(--input-text);
  font-size:        var(--input-font-size);
  line-height:      var(--line-height-normal);
  transition:       border-color var(--duration-normal) var(--easing-ease-in-out),
                    box-shadow   var(--duration-normal) var(--easing-ease-in-out);
  appearance: none;
}

.form-field__input::placeholder {
  color: var(--input-placeholder);
}

.form-field__input:hover:not(:disabled):not([readonly]) {
  border-color: var(--input-border-hover);
}

.form-field__input:focus-visible {
  outline:      none;
  border-color: var(--input-border-focus);
  box-shadow:   0 0 0 3px rgba(59, 130, 246, 0.25);
}

/* Error state */
.form-field--error .form-field__input {
  border-color: var(--input-border-error);
}

.form-field--error .form-field__input:focus-visible {
  box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.25);
}

/* Disabled */
.form-field__input:disabled {
  background-color: var(--input-disabled-bg);
  color:            var(--input-disabled-text);
  border-color:     var(--input-disabled-border);
  cursor:           not-allowed;
}

/* Read-only */
.form-field__input[readonly] {
  background-color: var(--color-surface-sunken);
  cursor:           default;
}

/* Error message */
.form-field__error {
  display:     flex;
  align-items: flex-start;
  gap:         var(--space-1);
  font-size:   var(--font-size-sm);
  color:       var(--color-feedback-error-text);
}

.form-field__error-icon {
  flex-shrink: 0;
  width:  1em;
  height: 1em;
  margin-top: 0.1em;
}

/* Prefix / suffix */
.form-field__prefix,
.form-field__suffix {
  display:     flex;
  align-items: center;
  position:    absolute;
  top:         50%;
  transform:   translateY(-50%);
  color:       var(--color-text-secondary);
  pointer-events: none;
}

.form-field__prefix { left:  var(--input-padding-x); }
.form-field__suffix { right: var(--input-padding-x); }

.form-field--has-prefix .form-field__input { padding-left:  calc(var(--input-padding-x) * 2 + 1em); }
.form-field--has-suffix .form-field__input { padding-right: calc(var(--input-padding-x) * 2 + 1em); }

/* Character counter */
.form-field__counter {
  align-self:  flex-end;
  font-size:   var(--font-size-xs);
  color:       var(--color-text-tertiary);
}

.form-field__counter--near-limit { color: var(--color-feedback-warning-text); }
.form-field__counter--at-limit   { color: var(--color-feedback-error-text); }
```

### Accessible HTML Template

```html
<div class="form-field form-field--error">
  <label class="form-field__label form-field__label--required" for="email">
    Email address
  </label>
  <p class="form-field__hint" id="email-hint">
    We'll send a confirmation link here.
  </p>

  <div class="form-field__control-wrapper form-field--has-suffix">
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
    />
    <span class="form-field__suffix" aria-hidden="true">
      <!-- error icon -->
    </span>
  </div>

  <p class="form-field__error" id="email-error" role="alert">
    <!-- error icon -->
    Please enter a valid email address.
  </p>
</div>
```

---

## Badge

### When to use
Inline label that conveys a category, status, or count. Non-interactive — use a `<span>`.

### Variants

| Variant | Semantic meaning |
|---|---|
| `neutral` | General-purpose; no particular status |
| `brand` | Product feature, subscription tier |
| `success` | Active, published, completed, online |
| `warning` | Expiring, pending, degraded |
| `error` | Failed, offline, blocked |
| `info` | Informational, in-review |

### Shapes
`rounded` (default) / `pill` / `square`

### API

| Prop | Type | Default | Description |
|---|---|---|---|
| `variant` | `"neutral" \| "brand" \| "success" \| "warning" \| "error" \| "info"` | `"neutral"` | Colour |
| `shape` | `"rounded" \| "pill" \| "square"` | `"rounded"` | Border radius |
| `size` | `"sm" \| "md"` | `"md"` | |
| `dot` | `boolean` | `false` | Shows a coloured dot before the label |
| `icon` | `ReactNode` | `undefined` | Icon before label |

### CSS Implementation

```css
/* components/badge.css */

.badge {
  display:         inline-flex;
  align-items:     center;
  gap:             var(--space-1);
  padding:         var(--badge-padding-y) var(--badge-padding-x);
  border-radius:   var(--radius-md);
  font-size:       var(--badge-font-size);
  font-weight:     var(--badge-font-weight);
  line-height:     var(--line-height-tight);
  white-space:     nowrap;
}

/* Shapes */
.badge--pill   { border-radius: var(--radius-full); }
.badge--square { border-radius: var(--radius-base); }

/* Size */
.badge--sm {
  font-size: calc(var(--font-size-xs) * 0.9);
  padding:   1px var(--space-1-5);
}

/* Variants */
.badge--neutral { background-color: var(--badge-neutral-bg); color: var(--badge-neutral-text); }
.badge--brand   { background-color: var(--badge-brand-bg);   color: var(--badge-brand-text); }
.badge--success { background-color: var(--badge-success-bg); color: var(--badge-success-text); }
.badge--warning { background-color: var(--badge-warning-bg); color: var(--badge-warning-text); }
.badge--error   { background-color: var(--badge-error-bg);   color: var(--badge-error-text); }

/* Dot */
.badge__dot {
  width:         0.5em;
  height:        0.5em;
  border-radius: 50%;
  background-color: currentColor;
  flex-shrink:   0;
}
```

---

## Modal / Dialog

### When to use
Interrupt the user to complete a focused task or confirm a destructive action. Use sparingly — prefer inline contextual UI where possible.

### When NOT to use
- Simple confirmation of non-destructive actions — use inline feedback instead.
- Displaying read-only information — use a drawer or detail panel.
- Deep nested workflows — push to a new page.

### Sizes

| Size | Width | Use |
|---|---|---|
| `sm` | 400 px | Simple confirmations |
| `md` (default) | 560 px | Forms, detail views |
| `lg` | 720 px | Complex forms, comparisons |
| `xl` | 900 px | Data tables, media |
| `fullscreen` | 100vw / 100vh | Mobile, immersive editors |

### API

| Prop | Type | Default | Description |
|---|---|---|---|
| `open` | `boolean` | required | Controlled open state |
| `onClose` | `() => void` | required | Called when user dismisses |
| `title` | `string` | required | Sets dialog `aria-labelledby` |
| `description` | `string` | `undefined` | Subtitle; sets `aria-describedby` |
| `size` | `"sm" \| "md" \| "lg" \| "xl" \| "fullscreen"` | `"md"` | |
| `closeOnOverlay` | `boolean` | `true` | Close when clicking the scrim |
| `closeOnEscape` | `boolean` | `true` | Close on Escape key |
| `showClose` | `boolean` | `true` | Show X button in header |
| `preventClose` | `boolean` | `false` | Blocks close (e.g. unsaved state) |
| `initialFocus` | `RefObject` | `undefined` | Element to receive focus on open |
| `returnFocus` | `RefObject` | `undefined` | Element to restore focus on close |

### Events / Slots

| Name | Description |
|---|---|
| `onClose` | Dialog dismissed |
| `slot: header` | Custom header replacing the default title/close row |
| `slot: footer` | Action buttons (typically Cancel + Confirm) |
| `slot: default / children` | Body content |

### Accessibility
- Renders a `<dialog>` element with `role="dialog"` and `aria-modal="true"`.
- `aria-labelledby` points to the title element; `aria-describedby` to the description if provided.
- Focus must be trapped inside the modal while open.
- On open, focus moves to `initialFocus` or the first focusable element.
- On close, focus returns to `returnFocus` or the element that triggered the modal.
- Escape key closes unless `closeOnEscape={false}`.
- Scroll on the page body must be locked while the modal is open (`overflow: hidden` on `<body>`).

### CSS Implementation

```css
/* components/modal.css */

/* Scrim overlay */
.modal-overlay {
  position:         fixed;
  inset:            0;
  background-color: var(--color-overlay-scrim);
  display:          flex;
  align-items:      center;
  justify-content:  center;
  padding:          var(--space-4);
  z-index:          var(--z-index-overlay);

  /* Entry animation */
  animation: overlay-in var(--duration-slow) var(--easing-ease-out) both;
}

@keyframes overlay-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}

/* Dialog panel */
.modal {
  position:         relative;
  background-color: var(--color-surface-overlay);
  border-radius:    var(--radius-xl);
  box-shadow:       var(--shadow-xl);
  display:          flex;
  flex-direction:   column;
  max-height:       calc(100vh - var(--space-16));
  width:            100%;
  z-index:          var(--z-index-modal);
  outline:          none;

  animation: modal-in var(--duration-slow) var(--easing-spring) both;
}

@keyframes modal-in {
  from { opacity: 0; transform: scale(0.96) translateY(var(--space-3)); }
  to   { opacity: 1; transform: scale(1)    translateY(0); }
}

/* Sizes */
.modal--sm { max-width: 400px; }
.modal--md { max-width: 560px; }
.modal--lg { max-width: 720px; }
.modal--xl { max-width: 900px; }

.modal--fullscreen {
  max-width:  none;
  max-height: none;
  width:      100vw;
  height:     100dvh;
  border-radius: 0;
}

/* Regions */
.modal__header {
  display:         flex;
  align-items:     flex-start;
  justify-content: space-between;
  gap:             var(--space-4);
  padding:         var(--space-5) var(--space-6);
  border-bottom:   1px solid var(--color-border-default);
  flex-shrink:     0;
}

.modal__title {
  font-size:   var(--font-size-lg);
  font-weight: var(--font-weight-semibold);
  color:       var(--color-text-primary);
  line-height: var(--line-height-tight);
}

.modal__description {
  font-size:  var(--font-size-sm);
  color:      var(--color-text-secondary);
  margin-top: var(--space-1);
}

.modal__body {
  flex:       1;
  overflow-y: auto;
  padding:    var(--space-6);
  overscroll-behavior: contain;
}

.modal__footer {
  display:         flex;
  justify-content: flex-end;
  gap:             var(--space-3);
  padding:         var(--space-4) var(--space-6);
  border-top:      1px solid var(--color-border-default);
  flex-shrink:     0;
}

/* Close button */
.modal__close {
  display:          flex;
  align-items:      center;
  justify-content:  center;
  width:            var(--space-8);
  height:           var(--space-8);
  border-radius:    var(--radius-md);
  border:           none;
  background:       transparent;
  color:            var(--color-text-secondary);
  cursor:           pointer;
  flex-shrink:      0;
  transition:       background-color var(--duration-normal) var(--easing-ease-in-out),
                    color var(--duration-normal) var(--easing-ease-in-out);
}

.modal__close:hover {
  background-color: var(--color-surface-sunken);
  color:            var(--color-text-primary);
}

.modal__close:focus-visible {
  outline:        3px solid var(--color-border-focus);
  outline-offset: 2px;
}
```

---

## Toast / Notification

### When to use
Brief, non-blocking feedback for an action that just completed. Auto-dismiss after 5 seconds. Stack multiple toasts.

### When NOT to use
- Errors requiring user action — use inline form validation or an error banner instead.
- Information requiring the user to read carefully — show a modal.

### Variants

| Variant | Use |
|---|---|
| `success` | Action completed (saved, uploaded, sent) |
| `error` | System-level failure the user must know about |
| `warning` | Action completed but with caveats |
| `info` | Neutral status update |

### API

| Prop | Type | Default | Description |
|---|---|---|---|
| `variant` | `"success" \| "error" \| "warning" \| "info"` | `"info"` | Icon and colour |
| `title` | `string` | required | Short description |
| `description` | `string` | `undefined` | Optional detail text |
| `duration` | `number \| null` | `5000` | ms before auto-dismiss; `null` = permanent |
| `action` | `{ label: string; onClick: () => void }` | `undefined` | Inline action link |
| `closeable` | `boolean` | `true` | Show dismiss button |
| `onDismiss` | `() => void` | `undefined` | Called when dismissed manually or auto |

### Accessibility
- Renders inside a `role="status"` live region for successes and `role="alert"` for errors.
- `aria-live="polite"` for info/success; `aria-live="assertive"` for error.
- Pause auto-dismiss when the user hovers or focuses the toast.
- Toast container positioned with `role="region"` and `aria-label="Notifications"`.

### CSS Implementation

```css
/* components/toast.css */

/* Container — portal to <body>, positioned top-right */
.toast-container {
  position:   fixed;
  top:        var(--space-4);
  right:      var(--space-4);
  display:    flex;
  flex-direction: column;
  gap:        var(--space-2);
  z-index:    var(--z-index-toast);
  max-width:  360px;
  width:      calc(100% - var(--space-8));
  pointer-events: none;
}

/* Individual toast */
.toast {
  display:          flex;
  align-items:      flex-start;
  gap:              var(--space-3);
  padding:          var(--space-3) var(--space-4);
  border-radius:    var(--radius-lg);
  background-color: var(--color-surface-raised);
  box-shadow:       var(--shadow-lg);
  border-left:      4px solid transparent;
  pointer-events:   auto;

  animation: toast-in var(--duration-slow) var(--easing-spring) both;
}

.toast.is-dismissing {
  animation: toast-out var(--duration-moderate) var(--easing-ease-in) both;
}

@keyframes toast-in {
  from { opacity: 0; transform: translateX(calc(100% + var(--space-4))); }
  to   { opacity: 1; transform: translateX(0); }
}

@keyframes toast-out {
  from { opacity: 1; transform: translateX(0); max-height: 200px; }
  to   { opacity: 0; transform: translateX(calc(100% + var(--space-4))); max-height: 0; }
}

/* Variants */
.toast--success { border-left-color: var(--color-feedback-success); }
.toast--error   { border-left-color: var(--color-feedback-error); }
.toast--warning { border-left-color: var(--color-feedback-warning); }
.toast--info    { border-left-color: var(--color-feedback-info); }

/* Icon */
.toast__icon {
  flex-shrink: 0;
  width:       1.25rem;
  height:      1.25rem;
  margin-top:  0.1em;
}

.toast--success .toast__icon { color: var(--color-feedback-success); }
.toast--error   .toast__icon { color: var(--color-feedback-error); }
.toast--warning .toast__icon { color: var(--color-feedback-warning); }
.toast--info    .toast__icon { color: var(--color-feedback-info); }

/* Content */
.toast__content { flex: 1; }

.toast__title {
  font-size:   var(--font-size-sm);
  font-weight: var(--font-weight-medium);
  color:       var(--color-text-primary);
  line-height: var(--line-height-tight);
}

.toast__description {
  font-size:  var(--font-size-sm);
  color:      var(--color-text-secondary);
  margin-top: var(--space-0-5);
}

.toast__action {
  display:    inline-block;
  font-size:  var(--font-size-sm);
  font-weight:var(--font-weight-medium);
  color:      var(--color-brand-primary);
  margin-top: var(--space-1);
  cursor:     pointer;
  border:     none;
  background: none;
  padding:    0;
  text-decoration: underline;
}

/* Close */
.toast__close {
  flex-shrink:    0;
  width:          var(--space-6);
  height:         var(--space-6);
  display:        flex;
  align-items:    center;
  justify-content:center;
  border:         none;
  background:     transparent;
  border-radius:  var(--radius-base);
  color:          var(--color-text-tertiary);
  cursor:         pointer;
  transition:     color var(--duration-normal) var(--easing-ease-in-out);
}

.toast__close:hover { color: var(--color-text-primary); }
.toast__close:focus-visible {
  outline:        2px solid var(--color-border-focus);
  outline-offset: 2px;
}

/* Progress bar (auto-dismiss timer) */
.toast__progress {
  position:  absolute;
  bottom:    0;
  left:      0;
  height:    2px;
  background-color: currentColor;
  opacity:   0.3;
  animation: toast-progress linear both;
  animation-duration: inherit; /* Set from JS via CSS custom property */
  transform-origin:  left;
}
```

---

## Component Pattern Rules

These rules apply to all components in the design system.

**API design**
- Keep APIs small and composable. Prefer slots/children over large prop lists.
- Use `variant` for visual style, `size` for scale, `state` for interactive states.
- Props that are always required must be documented as such — no silent defaults that mask errors.
- Boolean props use positive framing: `disabled`, `loading`, `required` — not `notDisabled` or `isEnabled`.

**State management**
- Components are controlled by default (accept `value` + `onChange`). Provide `defaultValue` for uncontrolled use.
- Never hold own state when a controlled API is provided — let the parent own truth.
- Loading and disabled states must prevent interaction, not just hide the element.

**Accessibility non-negotiables**
- Every interactive element must be reachable by keyboard.
- Focus styles must never be removed — use `:focus-visible` to scope to keyboard users.
- Every form control must have a programmatically associated label.
- Status changes and errors must be announced to screen readers via live regions or `aria-describedby`.

**CSS**
- Use component tokens (`--btn-*`, `--input-*`) not semantic tokens directly, so consumers can customise by overriding a single namespace.
- Never use `!important`. Fix specificity with a more targeted selector.
- Prefer `gap` over margin for spacing between sibling elements.
- Use `transition` only on `color`, `background-color`, `border-color`, `opacity`, `box-shadow`, and `transform` — never `all`.
