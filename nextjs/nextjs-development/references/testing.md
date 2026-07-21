# Testing — Full Reference

Complete testing patterns for Next.js App Router: Server Actions, Server Components,
Route Handlers, middleware, and Playwright E2E.
For coverage targets see the **Testing** section in `SKILL.md`.

---

## Table of Contents
1. [Setup and Tooling](#setup-and-tooling)
2. [Testing Server Actions](#testing-server-actions)
3. [Testing Route Handlers](#testing-route-handlers)
4. [Testing Middleware](#testing-middleware)
5. [Testing Client Components in Next.js](#testing-client-components-in-nextjs)
6. [Testing Auth Guards](#testing-auth-guards)
7. [Playwright End-to-End Tests](#playwright-end-to-end-tests)
8. [MSW for API Mocking in Next.js](#msw-for-api-mocking-in-nextjs)
9. [Naming and Coverage Guidelines](#naming-and-coverage-guidelines)

---

## Setup and Tooling

```bash
npm install -D vitest @vitejs/plugin-react @testing-library/react @testing-library/jest-dom
npm install -D msw @playwright/test
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'node',      // Server-side tests (actions, handlers) run in node
    setupFiles:  ['./src/test-setup.ts'],
    globals:     true,
  },
});
```

```typescript
// vitest.config.client.ts  — separate config for Client Component tests
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    name:        'client',
    environment: 'jsdom',
    setupFiles:  ['./src/test-setup.client.ts'],
    globals:     true,
    include:     ['**/*.client.test.{ts,tsx}'],
  },
});
```

---

## Testing Server Actions

Server Actions are async functions — test them directly as regular async functions.
Mock any repositories, external APIs, or auth helpers they call.

```typescript
// app/(dashboard)/orders/actions.test.ts

import { describe, it, expect, vi, beforeEach } from 'vitest';
import { createOrderAction, updateOrderStatusAction } from './actions';
import { ordersRepository }  from '../../../lib/db/orders.repository';
import { requireAuth }       from '../../../lib/auth/require-auth';
import { OrderStatus }       from '../../../models/enums/order-status.enum';

// Mock dependencies
vi.mock('../../../lib/db/orders.repository');
vi.mock('../../../lib/auth/require-auth');
vi.mock('next/cache', () => ({ revalidateTag: vi.fn(), revalidatePath: vi.fn() }));

const buildFormData = (fields: Record<string, string>): FormData => {
  const formData = new FormData();
  Object.entries(fields).forEach(([key, value]) => formData.set(key, value));
  return formData;
};

describe('createOrderAction()', () => {
  beforeEach(() => {
    vi.mocked(requireAuth).mockResolvedValue({ user: { id: 'usr-001', role: 'ADMIN' } } as never);
    vi.mocked(ordersRepository.create).mockResolvedValue({ orderId: 'ord-new' } as never);
  });

  it('should return isSuccess true when the order is created successfully', async () => {
    const formData = buildFormData({
      customerId: 'cust-001',
      items:      JSON.stringify([{ productId: 'prod-001', quantity: 2 }]),
    });

    const result = await createOrderAction(formData);

    expect(result.isSuccess).toBe(true);
    expect(ordersRepository.create).toHaveBeenCalledOnce();
  });

  it('should return fieldErrors when customerId is missing', async () => {
    const formData = buildFormData({
      items: JSON.stringify([{ productId: 'prod-001', quantity: 1 }]),
    });

    const result = await createOrderAction(formData);

    expect(result.isSuccess).toBe(false);
    expect(result.fieldErrors?.customerId).toBeDefined();
  });

  it('should return fieldErrors when items array is empty', async () => {
    const formData = buildFormData({
      customerId: 'cust-001',
      items:      JSON.stringify([]),
    });

    const result = await createOrderAction(formData);

    expect(result.isSuccess).toBe(false);
    expect(result.fieldErrors?.items).toBeDefined();
  });

  it('should return an errorMessage when the repository throws', async () => {
    vi.mocked(ordersRepository.create).mockRejectedValue(new Error('DB connection failed'));

    const formData = buildFormData({
      customerId: 'cust-001',
      items:      JSON.stringify([{ productId: 'prod-001', quantity: 1 }]),
    });

    const result = await createOrderAction(formData);

    expect(result.isSuccess).toBe(false);
    expect(result.errorMessage).toBe('DB connection failed');
  });
});

describe('updateOrderStatusAction()', () => {
  beforeEach(() => {
    vi.mocked(requireAuth).mockResolvedValue({ user: { id: 'usr-001' } } as never);
    vi.mocked(ordersRepository.updateStatus).mockResolvedValue(undefined);
  });

  it('should update the status and return isSuccess true', async () => {
    const formData = buildFormData({
      orderId: 'ord-001',
      status:  OrderStatus.Shipped,
    });

    const result = await updateOrderStatusAction(null, formData);

    expect(result.isSuccess).toBe(true);
    expect(ordersRepository.updateStatus).toHaveBeenCalledWith('ord-001', OrderStatus.Shipped);
  });

  it('should return fieldErrors when an invalid status value is provided', async () => {
    const formData = buildFormData({
      orderId: 'ord-001',
      status:  'INVALID_STATUS',
    });

    const result = await updateOrderStatusAction(null, formData);

    expect(result.isSuccess).toBe(false);
    expect(result.fieldErrors?.status).toBeDefined();
  });
});
```

---

## Testing Route Handlers

Test Route Handlers by constructing `NextRequest` objects and asserting on the `NextResponse`.

```typescript
// app/api/webhooks/stripe/route.test.ts

import { describe, it, expect, vi } from 'vitest';
import { NextRequest }              from 'next/server';
import { POST }                     from './route';

vi.mock('../../../lib/payments/stripe', () => ({
  verifyStripeWebhook: vi.fn(),
  processStripeEvent:  vi.fn(),
}));

import { verifyStripeWebhook, processStripeEvent } from '../../../lib/payments/stripe';

describe('POST /api/webhooks/stripe', () => {
  it('should return 400 when the stripe-signature header is missing', async () => {
    const request = new NextRequest('http://localhost/api/webhooks/stripe', {
      method: 'POST',
      body:   JSON.stringify({ type: 'payment_intent.succeeded' }),
    });

    const response = await POST(request);

    expect(response.status).toBe(400);
  });

  it('should return 200 when the webhook is verified and processed successfully', async () => {
    vi.mocked(verifyStripeWebhook).mockResolvedValue({ type: 'payment_intent.succeeded' } as never);
    vi.mocked(processStripeEvent).mockResolvedValue(undefined);

    const request = new NextRequest('http://localhost/api/webhooks/stripe', {
      method:  'POST',
      headers: { 'stripe-signature': 'valid-sig' },
      body:    JSON.stringify({ type: 'payment_intent.succeeded' }),
    });

    const response = await POST(request);
    const body     = await response.json();

    expect(response.status).toBe(200);
    expect(body.received).toBe(true);
  });

  it('should return 500 when webhook processing throws an error', async () => {
    vi.mocked(verifyStripeWebhook).mockResolvedValue({ type: 'charge.failed' } as never);
    vi.mocked(processStripeEvent).mockRejectedValue(new Error('Processing failed'));

    const request = new NextRequest('http://localhost/api/webhooks/stripe', {
      method:  'POST',
      headers: { 'stripe-signature': 'valid-sig' },
      body:    '{}',
    });

    const response = await POST(request);

    expect(response.status).toBe(500);
  });
});
```

---

## Testing Middleware

```typescript
// middleware.test.ts

import { describe, it, expect, vi } from 'vitest';
import { NextRequest }              from 'next/server';
import { middleware }               from './middleware';
import { verifyJwt }               from './src/lib/auth/jwt';

vi.mock('./src/lib/auth/jwt');

const buildRequest = (pathname: string, sessionToken?: string): NextRequest => {
  const request = new NextRequest(`http://localhost${pathname}`);
  if (sessionToken) {
    request.cookies.set('auth_session', sessionToken);
  }
  return request;
};

describe('middleware()', () => {
  it('should allow public routes without authentication', async () => {
    const response = await middleware(buildRequest('/login'));
    expect(response.status).toBe(200);  // NextResponse.next()
  });

  it('should redirect to /login when no session cookie is present', async () => {
    const response = await middleware(buildRequest('/dashboard'));

    expect(response.status).toBe(307);
    expect(response.headers.get('Location')).toContain('/login');
  });

  it('should allow authenticated requests to protected routes', async () => {
    vi.mocked(verifyJwt).mockResolvedValue({
      userId: 'usr-001',
      role:   'ADMIN',
      email:  'admin@example.com',
    } as never);

    const response = await middleware(buildRequest('/dashboard', 'valid-token'));

    expect(response.status).toBe(200);
  });

  it('should redirect to /login when the session token is expired', async () => {
    vi.mocked(verifyJwt).mockRejectedValue(new Error('Token expired'));

    const response = await middleware(buildRequest('/dashboard', 'expired-token'));

    expect(response.status).toBe(307);
    expect(response.headers.get('Location')).toContain('/login');
  });
});
```

---

## Testing Client Components in Next.js

Client Component tests use the same `@testing-library/react` approach as the React skill.
Run these in the `jsdom` environment via `vitest.config.client.ts`.

```typescript
// features/orders/components/order-status-toggle/order-status-toggle.component.client.test.tsx

import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { OrderStatusToggle }                  from './order-status-toggle.component';
import { updateOrderStatusAction }            from '../../../../app/(dashboard)/orders/actions';
import { OrderStatus }                        from '../../../../models/enums/order-status.enum';

vi.mock('../../../../app/(dashboard)/orders/actions');

const buildOrder = () => ({
  orderId:     'ord-001',
  customerId:  'cust-001',
  status:      OrderStatus.Pending,
  totalAmount: 100,
  currencyCode:'USD',
  placedAt:    '2024-01-01T00:00:00Z',
  items:       [],
});

describe('OrderStatusToggle', () => {
  it('should display the current order status', () => {
    render(<OrderStatusToggle order={buildOrder()} />);
    expect(screen.getByText('PENDING')).toBeInTheDocument();
  });

  it('should call the Server Action when "Mark as Shipped" is clicked', async () => {
    vi.mocked(updateOrderStatusAction).mockResolvedValue({ isSuccess: true });

    render(<OrderStatusToggle order={buildOrder()} />);
    fireEvent.click(screen.getByRole('button', { name: /mark as shipped/i }));

    await waitFor(() => {
      expect(updateOrderStatusAction).toHaveBeenCalledOnce();
    });
  });
});
```

---

## Testing Auth Guards

```typescript
// lib/auth/require-auth.test.ts

import { describe, it, expect, vi } from 'vitest';
import { requireAuth, requireRole }  from './require-auth';
import { auth }                      from '../../../auth';
import { redirect }                  from 'next/navigation';

vi.mock('../../../auth');
vi.mock('next/navigation', () => ({ redirect: vi.fn() }));

describe('requireAuth()', () => {
  it('should return the session when the user is authenticated', async () => {
    const mockSession = { user: { id: 'usr-001', role: 'ADMIN' } };
    vi.mocked(auth).mockResolvedValue(mockSession as never);

    const session = await requireAuth();

    expect(session).toEqual(mockSession);
    expect(redirect).not.toHaveBeenCalled();
  });

  it('should call redirect to /login when no session exists', async () => {
    vi.mocked(auth).mockResolvedValue(null as never);

    await requireAuth().catch(() => {});  // redirect() throws internally in Next.js

    expect(redirect).toHaveBeenCalledWith('/login');
  });
});

describe('requireRole()', () => {
  it('should allow access when the user has an allowed role', async () => {
    vi.mocked(auth).mockResolvedValue({ user: { id: 'usr-001', role: 'ADMIN' } } as never);

    await expect(requireRole(['ADMIN', 'EDITOR'])).resolves.not.toThrow();
  });

  it('should redirect to /forbidden when the user role is not allowed', async () => {
    vi.mocked(auth).mockResolvedValue({ user: { id: 'usr-001', role: 'VIEWER' } } as never);

    await requireRole(['ADMIN']).catch(() => {});

    expect(redirect).toHaveBeenCalledWith('/forbidden');
  });
});
```

---

## Playwright End-to-End Tests

```typescript
// e2e/auth/login.spec.ts

import { test, expect } from '@playwright/test';

test.describe('Login flow', () => {
  test('should redirect to dashboard after successful login', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email address').fill('admin@example.com');
    await page.getByLabel('Password').fill('SecurePass123');
    await page.getByRole('button', { name: 'Sign in' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('heading', { name: /dashboard/i })).toBeVisible();
  });

  test('should display an error when credentials are invalid', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email address').fill('wrong@example.com');
    await page.getByLabel('Password').fill('wrongpassword');
    await page.getByRole('button', { name: 'Sign in' }).click();

    await expect(page.getByRole('alert')).toContainText(/invalid email or password/i);
  });

  test('should redirect unauthenticated users to /login', async ({ page }) => {
    await page.goto('/dashboard');
    await expect(page).toHaveURL(/\/login/);
  });
});
```

```typescript
// playwright.config.ts

import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir:  './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries:  process.env.CI ? 2 : 0,
  reporter: 'html',
  use: {
    baseURL:       'http://localhost:3000',
    trace:         'on-first-retry',
    screenshot:    'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
    { name: 'Mobile Safari', use: { ...devices['iPhone 14'] } },
  ],
  webServer: {
    command: 'npm run build && npm run start',
    port:    3000,
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## MSW for API Mocking in Next.js

For integration tests that call external APIs, use MSW in Node mode (not browser):

```typescript
// src/mocks/server.ts  (reused from React skill)

import { setupServer } from 'msw/node';
import { handlers }    from './handlers';

export const server = setupServer(...handlers);
```

```typescript
// src/test-setup.ts

import { server } from './mocks/server';

beforeAll(()  => server.listen({ onUnhandledRequest: 'warn' }));
afterEach(()  => server.resetHandlers());
afterAll(()   => server.close());
```

---

## Naming and Coverage Guidelines

**Test naming — full English sentences:**
```typescript
// ✅ Correct
it('should return isSuccess false when customerId is missing from FormData')
it('should redirect to /login when no valid session cookie is present')
it('should display the optimistic status immediately before the Server Action completes')

// ❌ Wrong
it('works')
it('calls the action')
it('handles error')
```

**Coverage targets:**

| Layer | Minimum | Tool |
|---|---|---|
| Server Actions | 80% | Vitest (node) |
| Route Handlers | 80% | Vitest (node) |
| Middleware | 80% | Vitest (node) |
| Auth guards (lib/auth) | 80% | Vitest (node) |
| Client Components | 70% | Vitest (jsdom) + RTL |
| Mappers | 90% | Vitest (node) |
| Critical E2E flows | 100% key paths | Playwright |
