# Testing — Full Reference

Complete testing patterns, mocking strategies, and examples.
For coverage targets and high-level rules see the **Testing** section in `SKILL.md`.

---

## Table of Contents
1. [Setup and Tooling](#setup-and-tooling)
2. [Unit Tests — Custom Hooks](#unit-tests--custom-hooks)
3. [Unit Tests — Zustand Stores](#unit-tests--zustand-stores)
4. [Unit Tests — RTK Slices](#unit-tests--rtk-slices)
5. [Unit Tests — Mappers](#unit-tests--mappers)
6. [Component Tests — React Testing Library](#component-tests--react-testing-library)
7. [HTTP Mocking with MSW](#http-mocking-with-msw)
8. [TanStack Query in Tests](#tanstack-query-in-tests)
9. [Accessibility Testing with jest-axe](#accessibility-testing-with-jest-axe)
10. [Naming and Coverage Guidelines](#naming-and-coverage-guidelines)

---

## Setup and Tooling

**Preferred stack:**
- **Vitest** (preferred) or **Jest** as the test runner
- **@testing-library/react** for component tests — behaviour-focused, no internal coupling
- **@testing-library/user-event** for realistic user interactions (keyboard, click)
- **MSW (Mock Service Worker)** for HTTP mocking — intercepts at the network level
- **jest-axe** for automated accessibility assertions

**Vitest config:**
```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles:  ['./src/test-setup.ts'],
    globals:     true,
  },
});
```

```typescript
// src/test-setup.ts
import '@testing-library/jest-dom';
import { cleanup } from '@testing-library/react';
import { afterEach } from 'vitest';
import { server } from './mocks/server';

beforeAll(()  => server.listen({ onUnhandledRequest: 'error' }));
afterEach(()  => { server.resetHandlers(); cleanup(); });
afterAll(()   => server.close());
```

---

## Unit Tests — Custom Hooks

Use `renderHook` from `@testing-library/react` to test hooks in isolation.

```typescript
// features/cart/hooks/use-cart.hook.test.ts

import { renderHook, act } from '@testing-library/react';
import { useCartStore }    from '../store/cart.store';
import type { ICartItem }  from '../../../models/interfaces/i-cart-item.interface';

const buildCartItem = (overrides?: Partial<ICartItem>): ICartItem => ({
  id:        'item-001',
  name:      'Widget Pro',
  unitPrice: 49.99,
  quantity:  1,
  ...overrides,
});

describe('useCartStore', () => {
  // Reset Zustand store state between tests
  beforeEach(() => {
    useCartStore.setState({ items: [] });
  });

  describe('addItem()', () => {
    it('should add a new item to an empty cart', () => {
      const { result } = renderHook(() => useCartStore());

      act(() => result.current.addItem(buildCartItem()));

      expect(result.current.items).toHaveLength(1);
      expect(result.current.items[0].name).toBe('Widget Pro');
    });

    it('should increment quantity when the same item is added twice', () => {
      const { result } = renderHook(() => useCartStore());
      const cartItem = buildCartItem();

      act(() => {
        result.current.addItem(cartItem);
        result.current.addItem(cartItem);
      });

      expect(result.current.items).toHaveLength(1);
      expect(result.current.items[0].quantity).toBe(2);
    });
  });

  describe('removeItem()', () => {
    it('should remove the item with the matching id', () => {
      const { result } = renderHook(() => useCartStore());

      act(() => {
        result.current.addItem(buildCartItem({ id: 'item-001' }));
        result.current.addItem(buildCartItem({ id: 'item-002', name: 'Gadget Basic' }));
        result.current.removeItem('item-001');
      });

      expect(result.current.items).toHaveLength(1);
      expect(result.current.items[0].id).toBe('item-002');
    });
  });
});
```

---

## Unit Tests — Zustand Stores

Test the store's pure logic without rendering any component.

```typescript
// features/cart/store/cart.store.test.ts

import { useCartStore, selectCartTotalPrice, selectCartItemCount } from './cart.store';

describe('cart store selectors', () => {
  beforeEach(() => useCartStore.setState({ items: [] }));

  it('should calculate total price correctly across multiple items', () => {
    useCartStore.setState({
      items: [
        { id: 'item-001', name: 'Widget', unitPrice: 10, quantity: 2 },
        { id: 'item-002', name: 'Gadget', unitPrice: 20, quantity: 1 },
      ],
    });

    const totalPrice = selectCartTotalPrice(useCartStore.getState());

    // 10*2 + 20*1 = 40
    expect(totalPrice).toBeCloseTo(40);
  });

  it('should report zero item count for an empty cart', () => {
    const itemCount = selectCartItemCount(useCartStore.getState());
    expect(itemCount).toBe(0);
  });
});
```

---

## Unit Tests — RTK Slices

Test reducers and thunks in isolation — no store setup required for reducers.

```typescript
// features/orders/store/orders.slice.test.ts

import ordersReducer, {
  selectOrder,
  clearSelectedOrder,
} from './orders.slice';
import type { IOrder } from '../../../models/interfaces/i-order.interface';
import { OrderStatus }  from '../../../models/enums/order-status.enum';

const buildOrder = (overrides?: Partial<IOrder>): IOrder => ({
  orderId:     'ord-001',
  customerId:  'cust-001',
  status:      OrderStatus.Pending,
  totalAmount: 100,
  currencyCode:'USD',
  placedAt:    '2024-01-01T00:00:00Z',
  items:       [],
  ...overrides,
});

const initialState = {
  orders:          [],
  selectedOrderId: null,
  isLoading:       false,
  errorMessage:    null,
};

describe('ordersSlice reducer', () => {
  it('should return the initial state when no action is dispatched', () => {
    expect(ordersReducer(undefined, { type: '@@INIT' })).toEqual(initialState);
  });

  it('should set selectedOrderId when selectOrder is dispatched', () => {
    const nextState = ordersReducer(initialState, selectOrder('ord-001'));
    expect(nextState.selectedOrderId).toBe('ord-001');
  });

  it('should clear selectedOrderId when clearSelectedOrder is dispatched', () => {
    const stateWithSelection = { ...initialState, selectedOrderId: 'ord-001' };
    const nextState = ordersReducer(stateWithSelection, clearSelectedOrder());
    expect(nextState.selectedOrderId).toBeNull();
  });
});
```

---

## Unit Tests — Mappers

Mappers are pure functions — test exhaustively with 90%+ coverage.

```typescript
// models/mappers/order.mapper.test.ts

import { OrderMapper } from './order.mapper';
import { OrderStatus } from '../enums/order-status.enum';

const buildRawOrder = (overrides?: Record<string, unknown>): Record<string, unknown> => ({
  order_id:      'ord-001',
  customer_id:   'cust-001',
  status:        'PENDING',
  total_amount:  99.99,
  currency_code: 'USD',
  placed_at:     '2024-01-15T10:00:00Z',
  items:         [
    {
      product_id:   'prod-001',
      product_name: 'Widget Pro',
      qty:          2,
      unit_price:   49.99,
    },
  ],
  ...overrides,
});

describe('OrderMapper', () => {
  describe('fromApi()', () => {
    it('should map snake_case API fields to camelCase domain properties', () => {
      const mappedOrder = OrderMapper.fromApi(buildRawOrder());

      expect(mappedOrder.orderId).toBe('ord-001');
      expect(mappedOrder.customerId).toBe('cust-001');
      expect(mappedOrder.status).toBe(OrderStatus.Pending);
      expect(mappedOrder.totalAmount).toBe(99.99);
      expect(mappedOrder.currencyCode).toBe('USD');
    });

    it('should default currencyCode to USD when the field is absent', () => {
      const mappedOrder = OrderMapper.fromApi(buildRawOrder({ currency_code: undefined }));
      expect(mappedOrder.currencyCode).toBe('USD');
    });

    it('should map nested line items correctly', () => {
      const mappedOrder = OrderMapper.fromApi(buildRawOrder());

      expect(mappedOrder.items).toHaveLength(1);
      expect(mappedOrder.items[0].productId).toBe('prod-001');
      expect(mappedOrder.items[0].quantity).toBe(2);
      expect(mappedOrder.items[0].unitPrice).toBe(49.99);
    });
  });

  describe('fromApiList()', () => {
    it('should map an array of raw records and preserve order', () => {
      const rawOrders = [buildRawOrder({ order_id: 'ord-001' }), buildRawOrder({ order_id: 'ord-002' })];
      const mappedOrders = OrderMapper.fromApiList(rawOrders);

      expect(mappedOrders).toHaveLength(2);
      expect(mappedOrders[0].orderId).toBe('ord-001');
      expect(mappedOrders[1].orderId).toBe('ord-002');
    });

    it('should return an empty array when given an empty list', () => {
      expect(OrderMapper.fromApiList([])).toEqual([]);
    });
  });
});
```

---

## Component Tests — React Testing Library

Test behaviour through the DOM — never test implementation details.

```typescript
// features/orders/components/orders-list/orders-list.component.test.tsx

import { render, screen, waitFor } from '@testing-library/react';
import userEvent                   from '@testing-library/user-event';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { OrdersList }              from './orders-list.component';
import type { IOrder }             from '../../../../models/interfaces/i-order.interface';
import { OrderStatus }             from '../../../../models/enums/order-status.enum';

const buildOrder = (overrides?: Partial<IOrder>): IOrder => ({
  orderId:     'ord-001',
  customerId:  'cust-001',
  status:      OrderStatus.Pending,
  totalAmount: 100,
  currencyCode:'USD',
  placedAt:    '2024-01-01T00:00:00Z',
  items:       [],
  ...overrides,
});

/** Wrapper that provides a fresh QueryClient for each test. */
function renderWithProviders(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return render(
    <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>,
  );
}

describe('OrdersList', () => {
  it('should display a card for each order returned by the API', async () => {
    // MSW handler for GET /api/v1/orders is set up in src/mocks/handlers.ts
    renderWithProviders(<OrdersList />);

    await waitFor(() => {
      expect(screen.getByText('ord-001')).toBeInTheDocument();
    });
  });

  it('should display the empty-state message when no orders are returned', async () => {
    // Override MSW handler to return empty list for this test
    server.use(
      http.get('/api/v1/orders', () => HttpResponse.json([])),
    );

    renderWithProviders(<OrdersList />);

    await waitFor(() => {
      expect(screen.getByText(/no orders found/i)).toBeInTheDocument();
    });
  });

  it('should display an error message when the API request fails', async () => {
    server.use(
      http.get('/api/v1/orders', () => new HttpResponse(null, { status: 500 })),
    );

    renderWithProviders(<OrdersList />);

    await waitFor(() => {
      expect(screen.getByRole('alert')).toBeInTheDocument();
    });
  });
});
```

---

## HTTP Mocking with MSW

```typescript
// src/mocks/handlers.ts

import { http, HttpResponse } from 'msw';
import { ORDERS_API }         from '../models/constants/api.constants';
import { OrderStatus }        from '../models/enums/order-status.enum';

/** Default handlers — used in all tests unless overridden per-test. */
export const handlers = [
  // GET /api/v1/orders
  http.get(ORDERS_API.BASE, () =>
    HttpResponse.json([
      {
        order_id:      'ord-001',
        customer_id:   'cust-001',
        status:        'PENDING',
        total_amount:  100,
        currency_code: 'USD',
        placed_at:     '2024-01-01T00:00:00Z',
        items:         [],
      },
    ]),
  ),

  // GET /api/v1/orders/:orderId
  http.get(`${ORDERS_API.BASE}/:orderId`, ({ params }) =>
    HttpResponse.json({
      order_id:      params.orderId,
      customer_id:   'cust-001',
      status:        'PENDING',
      total_amount:  100,
      currency_code: 'USD',
      placed_at:     '2024-01-01T00:00:00Z',
      items:         [],
    }),
  ),
];
```

```typescript
// src/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers }    from './handlers';

export const server = setupServer(...handlers);
```

---

## TanStack Query in Tests

```typescript
// Utility wrapper — re-use across test files
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { render }                           from '@testing-library/react';

export function renderWithQuery(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry:              false,  // Prevent retries masking test failures
        gcTime:             0,      // Prevent stale data leaking between tests
        staleTime:          0,
      },
    },
  });
  return render(<QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>);
}
```

---

## Accessibility Testing with jest-axe

```typescript
// features/auth/components/login-form/login-form.component.test.tsx

import { render }                    from '@testing-library/react';
import { axe, toHaveNoViolations }   from 'jest-axe';
import { LoginForm }                 from './login-form.component';

expect.extend(toHaveNoViolations);

describe('LoginForm — accessibility', () => {
  it('should have no detectable accessibility violations on initial render', async () => {
    const { container } = render(<LoginForm />);
    const axeResults = await axe(container);
    expect(axeResults).toHaveNoViolations();
  });
});
```

---

## Naming and Coverage Guidelines

**Test file location:** co-locate with the file under test (`*.test.tsx` / `*.test.ts`).

**Naming — full English sentences:**
```typescript
// ✅ Correct — reads like a specification
it('should redirect to /login when the auth token has expired')
it('should disable the submit button while the form is submitting')
it('should display the empty-state when the API returns an empty list')

// ❌ Wrong — too vague or implementation-focused
it('works')
it('calls the service')
it('renders correctly')
```

**Coverage targets:**

| Layer | Minimum |
|---|---|
| Custom hooks (business logic) | 80% |
| Components | 70% |
| API services | 80% |
| Zustand stores / RTK slices | 90% |
| Mappers (pure functions) | 90% |
| Zod schemas / validators | 90% |
