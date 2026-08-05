---
author: Ankur Bhatnagar
---

# Accessibility — Full Reference

WCAG 2.1 AA patterns: keyboard navigation, ARIA roles, focus management, live regions, skip links, and screen reader patterns for design systems.

---

## Table of Contents
1. [WCAG 2.1 AA Requirements](#wcag-21-aa-requirements)
2. [Semantic HTML First](#semantic-html-first)
3. [Keyboard Navigation](#keyboard-navigation)
4. [Focus Management](#focus-management)
5. [ARIA Roles, States, and Properties](#aria-roles-states-and-properties)
6. [Live Regions](#live-regions)
7. [Images and Icons](#images-and-icons)
8. [Forms](#forms)
9. [Colour and Contrast](#colour-and-contrast)
10. [Motion and Animation](#motion-and-animation)
11. [Touch and Pointer Targets](#touch-and-pointer-targets)
12. [Skip Navigation](#skip-navigation)
13. [Screen Reader Patterns](#screen-reader-patterns)
14. [Accessibility Testing Checklist](#accessibility-testing-checklist)

---

## WCAG 2.1 AA Requirements

The design system targets **WCAG 2.1 Level AA**. The four principles are POUR:

| Principle | Meaning | Key requirements |
|---|---|---|
| **Perceivable** | Info and UI must be presentable to all senses | Text alternatives, captions, adaptable layout, minimum contrast |
| **Operable** | UI and navigation must be operable | Keyboard access, no seizure-inducing content, enough time |
| **Understandable** | Info and UI operation must be understandable | Readable text, predictable behavior, input assistance |
| **Robust** | Content must be robust enough for AT | Valid HTML, name/role/value |

Key contrast ratios (1.4.3 Contrast Minimum — AA):

| Text type | Required ratio |
|---|---|
| Normal text (< 18 pt / 14 pt bold) | 4.5 : 1 |
| Large text (>= 18 pt / 14 pt bold) | 3 : 1 |
| UI components and graphical objects | 3 : 1 |
| Decorative / logotype | No requirement |

---

## Semantic HTML First

Use the correct HTML element before reaching for ARIA. Native semantics provide built-in keyboard behaviour, state management, and AT support for free.

```html
<!-- Buttons and links — most common mistake -->
<!-- Wrong: div/span with click handler -->
<div onclick="submitForm()">Submit</div>

<!-- Right: native button -->
<button type="submit">Submit</button>

<!-- Wrong: button styled as link -->
<button onclick="navigate('/about')">About</button>

<!-- Right: anchor for navigation -->
<a href="/about">About</a>


<!-- Headings — define document outline -->
<!-- Wrong: styled div -->
<div class="heading-1">Page Title</div>

<!-- Right: semantic heading -->
<h1>Page Title</h1>


<!-- Lists — communicate structure to AT -->
<!-- Wrong: div-based list -->
<div class="nav">
  <div>Home</div>
  <div>About</div>
</div>

<!-- Right: semantic list -->
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>


<!-- Tables — must use proper table markup for AT -->
<table>
  <caption>Monthly sales figures</caption>
  <thead>
    <tr>
      <th scope="col">Month</th>
      <th scope="col">Revenue</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>January</td>
      <td>$42,000</td>
    </tr>
  </tbody>
</table>


<!-- Landmark elements — define page regions -->
<header role="banner">...</header>
<nav aria-label="Breadcrumb">...</nav>
<main>...</main>
<aside aria-label="Related articles">...</aside>
<footer role="contentinfo">...</footer>
```

HTML elements with built-in roles (do not re-declare with ARIA unless overriding):

| Element | Implicit role |
|---|---|
| `<button>` | `button` |
| `<a href="...">` | `link` |
| `<input type="checkbox">` | `checkbox` |
| `<input type="radio">` | `radio` |
| `<select>` | `listbox` |
| `<nav>` | `navigation` |
| `<main>` | `main` |
| `<header>` | `banner` (when top-level) |
| `<footer>` | `contentinfo` (when top-level) |
| `<aside>` | `complementary` |
| `<section>` | `region` (when has `aria-label` or `aria-labelledby`) |
| `<article>` | `article` |
| `<ul>` / `<ol>` | `list` |
| `<li>` | `listitem` |
| `<table>` | `table` |
| `<dialog>` | `dialog` |

---

## Keyboard Navigation

All interactive elements must be reachable and operable by keyboard alone.

### Standard key bindings

| Key | Expected behaviour |
|---|---|
| `Tab` | Move focus to the next focusable element |
| `Shift + Tab` | Move focus to the previous focusable element |
| `Enter` | Activate a link or button |
| `Space` | Activate a button; toggle a checkbox |
| `Arrow keys` | Navigate within a widget (menu, tabs, listbox, radio group) |
| `Escape` | Close a modal, dropdown, or tooltip; cancel an operation |
| `Home` / `End` | Jump to first / last item in a widget |
| `Page Up` / `Page Down` | Scroll or move within a large widget |

### Managing `tabindex`

```html
<!-- Naturally focusable — no tabindex needed -->
<button>Click me</button>
<a href="/page">Link</a>
<input type="text" />

<!-- Make a non-interactive element focusable (programmatic focus only) -->
<!-- e.g. dialog panel that receives focus on open, but is not in tab sequence -->
<div role="dialog" tabindex="-1" aria-labelledby="dialog-title">
  ...
</div>

<!-- tabindex="0" — adds to natural tab sequence at document position -->
<div role="button" tabindex="0" onclick="..." onkeydown="handleKey(event)">
  Custom widget
</div>

<!-- Never use tabindex > 0 — breaks natural tab order -->
<!-- Wrong: -->
<button tabindex="2">Second</button>
<button tabindex="1">First</button>
```

### Custom keyboard widget pattern (roving tabindex)

For widgets where arrow keys move within a group (menu, tabs, listbox, radio group):

```javascript
// Roving tabindex pattern
// Only ONE item in the group has tabindex="0" at any time.
// All others have tabindex="-1".
// Arrow keys move focus AND update which item has tabindex="0".

class RovingTabindex {
  constructor(container) {
    this.items = [...container.querySelectorAll('[role="tab"]')];
    this.current = 0;

    this.items.forEach((item, i) => {
      item.tabIndex = i === 0 ? 0 : -1;
      item.addEventListener("keydown", (e) => this.onKeyDown(e, i));
    });
  }

  onKeyDown(event, index) {
    let next = index;

    switch (event.key) {
      case "ArrowRight":
      case "ArrowDown":
        event.preventDefault();
        next = (index + 1) % this.items.length;
        break;
      case "ArrowLeft":
      case "ArrowUp":
        event.preventDefault();
        next = (index - 1 + this.items.length) % this.items.length;
        break;
      case "Home":
        event.preventDefault();
        next = 0;
        break;
      case "End":
        event.preventDefault();
        next = this.items.length - 1;
        break;
      default:
        return;
    }

    this.items[this.current].tabIndex = -1;
    this.items[next].tabIndex = 0;
    this.items[next].focus();
    this.current = next;
  }
}
```

---

## Focus Management

### Focus trap (modal, drawer, menu)

```javascript
// Focus must be trapped inside a modal while it is open.
// Only the elements inside the modal should be reachable by Tab/Shift+Tab.

function trapFocus(container) {
  const FOCUSABLE =
    'a[href], button:not([disabled]), input:not([disabled]), ' +
    'select:not([disabled]), textarea:not([disabled]), ' +
    '[tabindex]:not([tabindex="-1"])';

  function getFocusable() {
    return [...container.querySelectorAll(FOCUSABLE)].filter(
      (el) => !el.closest('[hidden]') && !el.closest('[aria-hidden="true"]')
    );
  }

  function handleKeyDown(event) {
    if (event.key !== 'Tab') return;

    const focusable = getFocusable();
    const first = focusable[0];
    const last  = focusable[focusable.length - 1];

    if (event.shiftKey) {
      if (document.activeElement === first) {
        event.preventDefault();
        last.focus();
      }
    } else {
      if (document.activeElement === last) {
        event.preventDefault();
        first.focus();
      }
    }
  }

  container.addEventListener('keydown', handleKeyDown);

  // Move focus into the container on open
  const firstFocusable = getFocusable()[0];
  if (firstFocusable) firstFocusable.focus();

  // Return cleanup function
  return () => container.removeEventListener('keydown', handleKeyDown);
}
```

### Focus restoration

```javascript
// Always restore focus to the trigger element when a modal or menu closes.

class ModalManager {
  open(modal, trigger) {
    this._previousFocus = trigger ?? document.activeElement;
    this._cleanup = trapFocus(modal);
    modal.removeAttribute('hidden');
    modal.setAttribute('aria-hidden', 'false');
  }

  close(modal) {
    this._cleanup?.();
    modal.setAttribute('hidden', '');
    modal.setAttribute('aria-hidden', 'true');
    this._previousFocus?.focus();
    this._previousFocus = null;
  }
}
```

### Focus ring CSS

```css
/* Never remove focus styles globally.
   Use :focus-visible to show them only for keyboard users. */

/* Wrong — removes focus from keyboard users too: */
/* *:focus { outline: none; } */

/* Right — scopes removal to mouse/pointer interactions: */
:focus:not(:focus-visible) {
  outline: none;
}

:focus-visible {
  outline:        3px solid var(--color-border-focus);
  outline-offset: 2px;
}

/* Component-level focus override */
.btn:focus-visible {
  outline:        3px solid var(--color-border-focus);
  outline-offset: 3px;
}

.form-field__input:focus-visible {
  outline:    none;                    /* replaced by box-shadow for inset look */
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.30);
}
```

---

## ARIA Roles, States, and Properties

Use ARIA only when native HTML semantics are insufficient. The first rule of ARIA: **do not use ARIA if you can use native HTML**.

### Common roles

```html
<!-- Alert — immediate important message -->
<div role="alert" aria-live="assertive">
  Your session will expire in 2 minutes.
</div>

<!-- Status — polite update -->
<div role="status" aria-live="polite">
  3 results found.
</div>

<!-- Dialog -->
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-desc"
  tabindex="-1"
>
  <h2 id="dialog-title">Confirm deletion</h2>
  <p id="dialog-desc">This action cannot be undone.</p>
</div>

<!-- Tooltip -->
<button aria-describedby="copy-tooltip" id="copy-btn">
  <svg aria-hidden="true">...</svg>
  <span class="sr-only">Copy to clipboard</span>
</button>
<div role="tooltip" id="copy-tooltip">
  Copy to clipboard
</div>

<!-- Tab widget -->
<div role="tablist" aria-label="Account sections">
  <button role="tab" aria-selected="true"  aria-controls="panel-profile" id="tab-profile">Profile</button>
  <button role="tab" aria-selected="false" aria-controls="panel-billing" id="tab-billing" tabindex="-1">Billing</button>
</div>
<div role="tabpanel" id="panel-profile" aria-labelledby="tab-profile">...</div>
<div role="tabpanel" id="panel-billing" aria-labelledby="tab-billing" hidden>...</div>

<!-- Menu (application menu, not navigation) -->
<button aria-haspopup="menu" aria-expanded="false" aria-controls="action-menu">
  Actions
</button>
<ul role="menu" id="action-menu" hidden>
  <li role="menuitem">Edit</li>
  <li role="menuitem">Duplicate</li>
  <li role="menuitem" aria-disabled="true">Delete</li>
</ul>

<!-- Combobox (autocomplete) -->
<input
  role="combobox"
  aria-expanded="false"
  aria-autocomplete="list"
  aria-controls="results-list"
  aria-activedescendant=""
/>
<ul role="listbox" id="results-list">
  <li role="option" aria-selected="false" id="option-1">Option one</li>
  <li role="option" aria-selected="false" id="option-2">Option two</li>
</ul>
```

### Key ARIA attributes

| Attribute | Type | Purpose |
|---|---|---|
| `aria-label` | Property | Accessible name when no visible label exists |
| `aria-labelledby` | Property | Points to element(s) providing the name |
| `aria-describedby` | Property | Points to element(s) providing description / hint |
| `aria-hidden="true"` | State | Removes element from AT; use for decorative elements |
| `aria-expanded` | State | Open/closed state of a collapsible widget |
| `aria-selected` | State | Selection state in listbox, tree, tab |
| `aria-checked` | State | Checked state for checkbox/radio/switch |
| `aria-pressed` | State | Toggle button on/off state |
| `aria-disabled="true"` | State | Disabled without removing from tab order |
| `aria-invalid="true"` | State | Field has validation error |
| `aria-required="true"` | Property | Field is required |
| `aria-busy="true"` | State | Region is loading |
| `aria-live` | Property | `"polite"` or `"assertive"` for live regions |
| `aria-atomic` | Property | Whether entire region is announced on change |
| `aria-relevant` | Property | What changes trigger announcements |
| `aria-current` | State | Current item in set: `"page"`, `"step"`, `"date"` |
| `aria-haspopup` | Property | Has associated popup: `"menu"`, `"listbox"`, `"dialog"` |
| `aria-controls` | Property | ID of element controlled by this one |
| `aria-owns` | Property | ID of element owned by this one (when not a DOM child) |
| `aria-modal="true"` | Property | Dialog is modal; AT restricts browsing to it |
| `aria-sort` | Property | Column sort direction: `"ascending"`, `"descending"` |
| `aria-colcount` / `aria-rowcount` | Property | Total count when not all rows are in DOM |

---

## Live Regions

Live regions announce content changes to screen readers without moving focus.

```html
<!-- Status (polite — waits for user to finish current action) -->
<div role="status" aria-live="polite" aria-atomic="true" id="status-region">
  <!-- Updated via JavaScript: -->
  <!-- "File saved successfully." -->
  <!-- "3 of 10 items selected." -->
</div>

<!-- Alert (assertive — interrupts immediately) -->
<div role="alert" aria-live="assertive" aria-atomic="true" id="alert-region">
  <!-- Reserved for critical errors and urgent notices -->
</div>

<!-- Log (history of messages, not atomic) -->
<div role="log" aria-live="polite" aria-label="Chat messages" aria-relevant="additions">
  <!-- New messages appended here -->
</div>
```

```javascript
// Announce a message without focusing an element
function announce(message, priority = 'polite') {
  const region = document.getElementById(
    priority === 'assertive' ? 'alert-region' : 'status-region'
  );

  if (!region) return;

  // Clear and re-set to guarantee re-announcement
  region.textContent = '';
  requestAnimationFrame(() => {
    region.textContent = message;
  });
}

// Usage
announce('File saved.');
announce('Error: connection lost.', 'assertive');
```

Live region rules:
- Inject the region into the DOM on page load — do not create it dynamically when needed.
- Set `aria-atomic="true"` when the entire region text should be read on update (e.g. a counter).
- Set `aria-atomic="false"` with `aria-relevant="additions"` for chat / log scrollback.
- Do not overuse assertive — it interrupts whatever the user was doing.

---

## Images and Icons

```html
<!-- Informative image: alt text describes the content -->
<img src="chart.png" alt="Bar chart showing sales growth of 23% in Q3 2024." />

<!-- Decorative image: empty alt so AT skips it -->
<img src="decorative-divider.svg" alt="" />

<!-- Icon with adjacent label: hide icon from AT -->
<button>
  <svg aria-hidden="true" focusable="false">...</svg>
  Save
</button>

<!-- Icon-only button: label on the button itself -->
<button aria-label="Close dialog">
  <svg aria-hidden="true" focusable="false">...</svg>
</button>

<!-- CSS background image with meaning: -->
<!-- Use role="img" and aria-label on the element -->
<div
  class="map"
  role="img"
  aria-label="Map showing our office at 123 Main Street, Chicago"
></div>

<!-- Inline SVG with title and desc for complex graphics -->
<svg role="img" aria-labelledby="chart-title chart-desc">
  <title id="chart-title">Monthly Revenue 2024</title>
  <desc id="chart-desc">Line chart showing revenue growth from $1.2M in January to $2.1M in December.</desc>
  <!-- chart paths -->
</svg>
```

Screen reader utility class (hide visually, keep in AT):

```css
/* Visually hidden but accessible */
.sr-only {
  position:   absolute;
  width:      1px;
  height:     1px;
  padding:    0;
  margin:     -1px;
  overflow:   hidden;
  clip:       rect(0, 0, 0, 0);
  white-space:nowrap;
  border:     0;
}

/* Visually hidden until focused (use for skip links) */
.sr-only-focusable:focus-visible {
  position:  static;
  width:     auto;
  height:    auto;
  overflow:  visible;
  clip:      auto;
  white-space: normal;
}
```

---

## Forms

```html
<!-- Every input must have an associated label -->

<!-- Method 1: label[for] (preferred) -->
<label for="username">Username</label>
<input id="username" type="text" name="username" />

<!-- Method 2: label wrapping (no for/id needed) -->
<label>
  Email
  <input type="email" name="email" />
</label>

<!-- Method 3: aria-labelledby (custom components) -->
<span id="search-label">Search orders</span>
<input type="search" aria-labelledby="search-label" />

<!-- Method 4: aria-label (icon-only inputs) -->
<input type="search" aria-label="Search" />


<!-- Error handling -->
<label for="amount">Amount (USD)</label>
<input
  id="amount"
  type="number"
  name="amount"
  aria-invalid="true"
  aria-describedby="amount-error amount-hint"
/>
<p id="amount-hint">Enter a value between $1 and $10,000.</p>
<p id="amount-error" role="alert">
  Amount must be a positive number.
</p>


<!-- Required fields -->
<label for="name">
  Full name
  <span aria-hidden="true"> *</span>     <!-- Visual asterisk, hidden from AT -->
</label>
<input
  id="name"
  type="text"
  required
  aria-required="true"                    <!-- Explicit for AT compatibility -->
/>


<!-- Fieldset and legend for radio/checkbox groups -->
<fieldset>
  <legend>Preferred contact method</legend>
  <label><input type="radio" name="contact" value="email" /> Email</label>
  <label><input type="radio" name="contact" value="phone" /> Phone</label>
  <label><input type="radio" name="contact" value="post" /> Post</label>
</fieldset>


<!-- Autocomplete attributes — help AT and browsers fill values -->
<input type="text"     autocomplete="name" />
<input type="email"    autocomplete="email" />
<input type="tel"      autocomplete="tel" />
<input type="text"     autocomplete="street-address" />
<input type="password" autocomplete="current-password" />
<input type="password" autocomplete="new-password" />
```

---

## Colour and Contrast

Contrast must be checked at design time, not as an afterthought.

| Token | Value | On background | Contrast ratio | Pass? |
|---|---|---|---|---|
| `--color-text-primary` | `#111827` | `#ffffff` (white) | 18.1 : 1 | AA + AAA |
| `--color-text-secondary` | `#4b5563` | `#ffffff` | 7.5 : 1 | AA + AAA |
| `--color-text-tertiary` | `#9ca3af` | `#ffffff` | 2.9 : 1 | Fail — decorative only |
| `--color-brand-primary` | `#2563eb` | `#ffffff` | 4.6 : 1 | AA |
| `--color-feedback-error` | `#dc2626` | `#ffffff` | 4.7 : 1 | AA |
| `--color-feedback-success` | `#16a34a` | `#ffffff` | 4.6 : 1 | AA |

Rules:
- `--color-text-tertiary` passes only as decorative text (placeholders, timestamps on large surfaces). Do NOT use it for essential information.
- Check dark mode tokens independently — surface colours change, so contrast ratios must be re-validated.
- Communicate errors and states with more than colour alone (icon + text message, not just red text).
- Disable states may use reduced contrast (below 4.5:1) when `aria-disabled="true"` is set — WCAG exempts them.

---

## Motion and Animation

```css
/* Always wrap all animation/transition in a prefers-reduced-motion check */

/* Base: animations are ON by default */
.slide-in {
  animation: slide-in var(--duration-slow) var(--easing-ease-out) both;
}

/* Override: remove animations for users who prefer reduced motion */
@media (prefers-reduced-motion: reduce) {
  .slide-in {
    animation: none;
    /* Provide an instant alternative if the animation conveys meaning */
    opacity: 1;
    transform: none;
  }
}

/* System-wide nuclear option (in tokens/global/_motion.css) */
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

Rules:
- No animation should be the ONLY way to convey information.
- Looping animations must stop or offer a pause mechanism (WCAG 2.2.2).
- Flashing content must not exceed 3 Hz (WCAG 2.3.1).
- Parallax, scroll-triggered animations, and auto-playing video should be disabled under `prefers-reduced-motion`.

---

## Touch and Pointer Targets

Minimum touch target: **44 × 44 px** (WCAG 2.5.5 — AAA target; WCAG 2.5.8 — AA minimum 24 × 24 px).

Design system default: 44 × 44 px for all interactive elements.

```css
/* Small visual elements can meet the size requirement
   via padding or an invisible hit area */

.icon-btn {
  display:         flex;
  align-items:     center;
  justify-content: center;
  width:           var(--space-11);    /* 44 px */
  height:          var(--space-11);   /* 44 px */
  border-radius:   var(--radius-md);
  cursor:          pointer;
}

/* Expand hit area without changing visual size */
.small-link {
  position:  relative;
  padding:   var(--space-2);          /* Extra hit area */
}
```

Mobile-specific: add `touch-action: manipulation` to eliminate the 300ms tap delay on buttons and links:

```css
a,
button,
[role="button"] {
  touch-action: manipulation;
}
```

---

## Skip Navigation

Skip links allow keyboard users to bypass repetitive navigation blocks.

```html
<!-- Place skip links as the very first child of <body> -->
<body>
  <a href="#main-content" class="skip-link">Skip to main content</a>
  <a href="#main-nav"     class="skip-link">Skip to navigation</a>

  <header>...</header>
  <nav id="main-nav">...</nav>

  <main id="main-content" tabindex="-1">
    <!-- tabindex="-1" allows the element to receive programmatic focus -->
    ...
  </main>
</body>
```

```css
/* Skip link — visually hidden until focused */
.skip-link {
  position:   absolute;
  top:        var(--space-2);
  left:       var(--space-2);
  z-index:    var(--z-index-toast);     /* Above everything */

  /* Hidden */
  width:      1px;
  height:     1px;
  overflow:   hidden;
  clip:       rect(0, 0, 0, 0);
  white-space:nowrap;
}

.skip-link:focus-visible {
  /* Visible */
  width:        auto;
  height:       auto;
  overflow:     visible;
  clip:         auto;
  white-space:  normal;

  background:   var(--color-surface-default);
  color:        var(--color-text-primary);
  padding:      var(--space-3) var(--space-4);
  border-radius:var(--radius-md);
  border:       2px solid var(--color-border-focus);
  font-weight:  var(--font-weight-medium);
  text-decoration: none;
  box-shadow:   var(--shadow-lg);
}
```

---

## Screen Reader Patterns

### Loading state

```html
<!-- Show spinner with SR announcement -->
<div aria-live="polite" aria-busy="true" id="order-section">
  <span class="sr-only">Loading orders, please wait.</span>
  <div class="spinner" aria-hidden="true"></div>
</div>

<!-- When loaded -->
<div aria-live="polite" aria-busy="false" id="order-section">
  <!-- Order content -->
</div>
```

### Pagination

```html
<nav aria-label="Order list pagination">
  <button aria-label="Previous page" aria-disabled="true">Previous</button>
  <button aria-label="Page 1" aria-current="page">1</button>
  <button aria-label="Page 2">2</button>
  <button aria-label="Page 3">3</button>
  <button aria-label="Next page">Next</button>
</nav>
```

### Breadcrumb

```html
<nav aria-label="Breadcrumb">
  <ol>
    <li><a href="/">Home</a></li>
    <li><a href="/orders">Orders</a></li>
    <li><span aria-current="page">Order #ORD-1042</span></li>
  </ol>
</nav>
```

### Expandable section (accordion)

```html
<div class="accordion">
  <h3>
    <button
      aria-expanded="false"
      aria-controls="faq-answer-1"
      id="faq-question-1"
    >
      What is the return policy?
    </button>
  </h3>
  <div
    id="faq-answer-1"
    role="region"
    aria-labelledby="faq-question-1"
    hidden
  >
    <p>You can return items within 30 days...</p>
  </div>
</div>
```

### Progress indicator (multi-step form)

```html
<nav aria-label="Order steps">
  <ol>
    <li aria-current="step">
      <span aria-label="Step 1 of 3, current step">
        <span aria-hidden="true">1.</span> Cart
      </span>
    </li>
    <li>
      <span aria-label="Step 2 of 3">
        <span aria-hidden="true">2.</span> Shipping
      </span>
    </li>
    <li aria-disabled="true">
      <span aria-label="Step 3 of 3, not yet available">
        <span aria-hidden="true">3.</span> Payment
      </span>
    </li>
  </ol>
</nav>
```

### Sort-able table column headers

```html
<table>
  <thead>
    <tr>
      <th scope="col">
        <button aria-sort="ascending">
          Date
          <span aria-hidden="true">↑</span>
        </button>
      </th>
      <th scope="col">
        <button aria-sort="none">
          Amount
          <span aria-hidden="true">↕</span>
        </button>
      </th>
    </tr>
  </thead>
  ...
</table>
```

---

## Accessibility Testing Checklist

Run this checklist before any component ships.

**Automated testing (fastest — catches ~30% of issues)**
- [ ] Run axe-core (e.g. `axe` browser extension, `@axe-core/playwright`) — zero violations
- [ ] Run WAVE tool — zero errors
- [ ] HTML validates with no errors (`validator.w3.org`)
- [ ] Colour contrast passes for all text and UI components (use Colour Contrast Analyser or Figma plugins)

**Keyboard testing**
- [ ] Tab through all interactive elements in DOM order
- [ ] All interactive elements receive visible focus
- [ ] Escape closes modals, dropdowns, and tooltips
- [ ] Arrow keys navigate within widgets (tabs, menus, listboxes)
- [ ] No keyboard trap (can always Tab away, except intentional modal trap)
- [ ] Skip link appears on first Tab press and works

**Screen reader testing**
- [ ] VoiceOver (macOS + Safari) — component name, role, and state announced correctly
- [ ] NVDA (Windows + Firefox or Chrome) — same
- [ ] Mobile: VoiceOver (iOS) + TalkBack (Android) for mobile patterns
- [ ] Errors and status changes are announced without focus move
- [ ] Images have meaningful alt text (or empty alt for decorative)
- [ ] Form fields: label, hint, and error all associated correctly

**Visual testing**
- [ ] 200% browser zoom — layout does not break, no content obscured
- [ ] 400% zoom (WCAG 1.4.10 Reflow) — content reflows to single column, no horizontal scroll
- [ ] Windows High Contrast Mode — all UI remains usable
- [ ] `prefers-color-scheme: dark` — sufficient contrast in dark mode
- [ ] `prefers-reduced-motion` — animations disabled or substituted

**Cognitive and usability**
- [ ] Error messages explain what is wrong and how to fix it
- [ ] Required fields clearly labelled
- [ ] Timeout warnings give adequate notice (WCAG 2.2.1)
- [ ] No unexpected context changes (no auto-navigation, no auto-focus moves)
