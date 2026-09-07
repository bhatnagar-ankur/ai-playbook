# Testing — Full Reference

Complete testing patterns for vanilla JS and Web Components: unit tests, Shadow DOM
assertions, custom event assertions, accessibility checks, and end-to-end tests.
For the overview and quick rules see the **Testing** section in `SKILL.md`.

---

## Table of Contents
1. [Setup and Tooling](#setup-and-tooling)
2. [Unit Tests — Plain Functions and Utilities](#unit-tests--plain-functions-and-utilities)
3. [Unit Tests — Web Components (Shadow DOM)](#unit-tests--web-components-shadow-dom)
4. [Testing Custom Events](#testing-custom-events)
5. [Accessibility Testing with axe-core](#accessibility-testing-with-axe-core)
6. [End-to-End Testing with Playwright](#end-to-end-testing-with-playwright)
7. [Naming and Coverage Guidelines](#naming-and-coverage-guidelines)

---

## Setup and Tooling

**Preferred stack:**
- **Web Test Runner** (`@web/test-runner`) — runs tests in real browsers via a Playwright/Puppeteer
  launcher, which is the closest match to how Custom Elements and Shadow DOM actually execute
- **Vitest** with `happy-dom` or `jsdom` — a faster alternative when full browser fidelity isn't
  required; both environments support `customElements` and Shadow DOM
- **axe-core** for automated accessibility assertions
- **Playwright** for full end-to-end browser tests

### Web Test Runner config

```javascript
// web-test-runner.config.js
export default {
  files: 'scripts/**/*.test.js',
  nodeResolve: true,
  concurrency: 4,
};
```

### Vitest config (alternative)

```javascript
// vitest.config.js
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'happy-dom',   // or 'jsdom'
    globals:     true,
  },
});
```

---

## Unit Tests — Plain Functions and Utilities

Pure functions (mappers, formatters, validators, state modules) need no DOM — test them directly.

```javascript
// scripts/models/order.model.test.js
import { describe, it, expect } from 'vitest';
import { orderFromApi } from './order.model.js';

describe('orderFromApi', () => {
  it('should map snake_case API fields to camelCase properties', () => {
    const order = orderFromApi({ order_id: 'ord-001', total_amount: 99.99 });
    expect(order.orderId).toBe('ord-001');
    expect(order.totalAmount).toBe(99.99);
  });

  it('should default status to PENDING when the field is absent', () => {
    const order = orderFromApi({ order_id: 'ord-001' });
    expect(order.status).toBe('PENDING');
  });

  it('should return an empty items array when items is not an array', () => {
    const order = orderFromApi({ order_id: 'ord-001', items: null });
    expect(order.items).toEqual([]);
  });
});
```

---

## Unit Tests — Web Components (Shadow DOM)

Import the component module to register the custom element, create an instance, attach it to
the document, then assert against `element.shadowRoot` — never `document.querySelector`, which
cannot cross the shadow boundary.

```javascript
// scripts/components/status-badge.component.test.js
import { describe, it, expect, afterEach } from 'vitest';
import './status-badge.component.js';   // registers <status-badge>

afterEach(() => {
  document.body.innerHTML = '';
});

describe('<status-badge>', () => {
  it('should render the status text inside its shadow root', async () => {
    const el = document.createElement('status-badge');
    el.setAttribute('status', 'shipped');
    document.body.appendChild(el);

    // Await a microtask in case the component defers its render
    await Promise.resolve();

    const badge = el.shadowRoot.querySelector('.badge');
    expect(badge).not.toBeNull();
    expect(badge.textContent.trim()).toContain('shipped');
    expect(badge.classList.contains('badge--shipped')).toBe(true);
  });

  it('should re-render when the status attribute changes', async () => {
    const el = document.createElement('status-badge');
    document.body.appendChild(el);

    el.setAttribute('status', 'cancelled');
    await Promise.resolve();

    const badge = el.shadowRoot.querySelector('.badge');
    expect(badge.classList.contains('badge--cancelled')).toBe(true);
  });
});
```

**Rules:**
- Query into Shadow DOM with `element.shadowRoot.querySelector(...)` / `.querySelectorAll(...)`
- Await a microtask (`await Promise.resolve()`) or `requestAnimationFrame` after an attribute
  change if the component's render is not synchronous
- Test the public contract — attributes, properties, and dispatched events — never assert on
  `#private` fields directly
- For components using `attachInternals()` (form-associated custom elements), assert against
  `element.value` and the submitted `FormData`, not internal state

---

## Testing Custom Events

Components communicate via `CustomEvent`. Assert dispatched events with a listener, not by
inspecting internals.

```javascript
// scripts/components/create-order-form.component.test.js
import { describe, it, expect } from 'vitest';
import './create-order-form.component.js';

describe('<create-order-form>', () => {
  it('should dispatch order:submit with the entered customerId', async () => {
    const el = document.createElement('create-order-form');
    document.body.appendChild(el);
    await Promise.resolve();

    const submitPromise = new Promise((resolve) => {
      el.addEventListener('order:submit', (event) => resolve(event.detail), { once: true });
    });

    const input = /** @type {HTMLInputElement} */ (el.shadowRoot.getElementById('customer-id'));
    input.value = 'CUST-001';

    el.shadowRoot.getElementById('order-form')
      .dispatchEvent(new Event('submit', { cancelable: true }));

    const detail = await submitPromise;
    expect(detail.customerId).toBe('CUST-001');
  });

  it('should dispatch order:cancel when the cancel button is clicked', async () => {
    const el = document.createElement('create-order-form');
    document.body.appendChild(el);
    await Promise.resolve();

    const cancelPromise = new Promise((resolve) => {
      el.addEventListener('order:cancel', resolve, { once: true });
    });

    el.shadowRoot.getElementById('cancel-btn').click();

    await expect(cancelPromise).resolves.toBeDefined();
  });
});
```

---

## Accessibility Testing with axe-core

Run an automated accessibility scan against rendered output, including Shadow DOM content —
axe-core traverses **open** shadow roots.

```javascript
// scripts/components/order-card.component.test.js
import { describe, it, expect } from 'vitest';
import axe from 'axe-core';
import './order-card.component.js';

describe('<order-card> — accessibility', () => {
  it('should have no detectable accessibility violations', async () => {
    const el = document.createElement('order-card');
    el.setAttribute('order-id', 'ord-001');
    el.setAttribute('status', 'shipped');
    document.body.appendChild(el);
    await Promise.resolve();

    const results = await axe.run(el);
    expect(results.violations).toEqual([]);
  });
});
```

**Rules:**
- Run an axe-core scan on every page-level template and every reusable Web Component before merging
- axe-core only traverses **open** shadow roots — a component using `mode: 'closed'` cannot be
  scanned this way, which is one more reason to default to `mode: 'open'`
- Fix violations at the source (missing labels, insufficient contrast, wrong roles) — never
  suppress a rule without a documented reason

---

## End-to-End Testing with Playwright

Use Playwright for full browser tests: navigation, real user interaction, and cross-component
flows against a running dev server.

```javascript
// e2e/orders.spec.js
import { test, expect } from '@playwright/test';

test('creates an order and shows a success toast', async ({ page }) => {
  await page.goto('/orders');

  await page.getByRole('button', { name: 'Create order' }).click();

  // Playwright locators pierce the shadow DOM automatically — no special selector syntax
  const customerIdInput = page.locator('create-order-form').locator('#customer-id');
  await customerIdInput.fill('CUST-001');

  await page.getByRole('button', { name: 'Submit' }).click();

  await expect(page.locator('notification-toast')).toContainText('Order created');
});

test('is keyboard navigable from page load', async ({ page }) => {
  await page.goto('/orders');
  await page.keyboard.press('Tab');   // First focusable element — the skip link
  await expect(page.locator('.skip-link')).toBeFocused();
});
```

```javascript
// playwright.config.js
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  use:     { baseURL: 'http://localhost:3000' },
  webServer: {
    command:              'npm run dev',
    port:                 3000,
    reuseExistingServer: !process.env.CI,
  },
});
```

**Rules:**
- Cover critical user flows end to end (create/edit/delete, navigation, form submission) — do
  not duplicate every unit test case at this layer
- Assert on visible, accessible output (`getByRole`, `getByLabelText`) — not on CSS classes or
  internal DOM structure
- Playwright locators pierce open shadow roots automatically

---

## Naming and Coverage Guidelines

**Test file location:** co-locate with the file under test (`*.test.js`).

**Naming — full sentences:**
```javascript
// ✅ Correct — reads like a specification
it('should render the status text inside its shadow root')
it('should dispatch order:submit with the entered customerId')
it('should default status to PENDING when the field is absent')

// ❌ Wrong — too vague or implementation-focused
it('works')
it('renders')
it('calls the handler')
```

**Coverage targets:**

| Layer | Minimum |
|---|---|
| Pure utilities / mappers | 90% |
| Web Components (Shadow DOM render + events) | 75% |
| State modules | 80% |
| End-to-end critical flows | All primary user journeys |
