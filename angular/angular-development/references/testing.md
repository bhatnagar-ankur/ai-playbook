# Testing — Full Reference

Complete testing patterns, mocking strategies, and examples.
For coverage targets and high-level rules see the **Testing** section in `SKILL.md`.

---

## Table of Contents
1. [Setup & Tooling](#setup--tooling)
2. [Unit Tests — Services with Signals](#unit-tests--services-with-signals)
3. [Unit Tests — Services with RxJS](#unit-tests--services-with-rxjs)
4. [Component Tests — Angular Testing Library](#component-tests--angular-testing-library)
5. [HTTP Mocking](#http-mocking)
6. [Testing NgRx Facades](#testing-ngrx-facades)
7. [Testing Guards](#testing-guards)
8. [Async Testing Patterns](#async-testing-patterns)
9. [Naming & Coverage Guidelines](#naming--coverage-guidelines)

---

## Setup & Tooling

**Preferred stack:**
- `@testing-library/angular` — component tests (behaviour-focused, no implementation coupling)
- Jest (preferred) or Karma/Jasmine (acceptable for existing projects)
- `@ngrx/store/testing` — MockStore for NgRx facade tests

**Installation (Jest):**
```bash
npm install -D jest @types/jest jest-preset-angular @testing-library/angular @testing-library/jest-dom
```

---

## Unit Tests — Services with Signals

```typescript
// features/cart/services/cart.service.spec.ts
import { TestBed }    from '@angular/core/testing';
import { CartService } from './cart.service';
import { ICartItem }  from '../../../models/interfaces/i-cart-item.interface';

describe('CartService', () => {
  let service: CartService;

  const buildCartItem = (overrides?: Partial<ICartItem>): ICartItem => ({
    id:        'item-001',
    name:      'Widget Pro',
    unitPrice: 49.99,
    quantity:  1,
    ...overrides,
  });

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(CartService);
  });

  describe('addItem()', () => {
    it('should add a new item to an empty cart', () => {
      service.addItem(buildCartItem());

      expect(service.items()).toHaveLength(1);
      expect(service.items()[0].name).toBe('Widget Pro');
    });

    it('should increment quantity when the same item is added twice', () => {
      const cartItem = buildCartItem();
      service.addItem(cartItem);
      service.addItem(cartItem);

      expect(service.items()).toHaveLength(1);
      expect(service.items()[0].quantity).toBe(2);
    });

    it('should add a second distinct item without affecting the first', () => {
      service.addItem(buildCartItem({ id: 'item-001' }));
      service.addItem(buildCartItem({ id: 'item-002', name: 'Gadget Basic' }));

      expect(service.items()).toHaveLength(2);
    });
  });

  describe('removeItem()', () => {
    it('should remove the item with the matching id', () => {
      service.addItem(buildCartItem({ id: 'item-001' }));
      service.addItem(buildCartItem({ id: 'item-002', name: 'Gadget Basic' }));

      service.removeItem('item-001');

      expect(service.items()).toHaveLength(1);
      expect(service.items()[0].id).toBe('item-002');
    });

    it('should have no effect when the id does not exist in the cart', () => {
      service.addItem(buildCartItem());

      service.removeItem('non-existent-id');

      expect(service.items()).toHaveLength(1);
    });
  });

  describe('computed values', () => {
    it('should calculate the total price correctly across multiple items', () => {
      service.addItem(buildCartItem({ unitPrice: 10, quantity: 1 }));
      service.addItem(buildCartItem({ id: 'item-002', unitPrice: 20, quantity: 2 }));

      // 10*1 + 20*2 = 50
      expect(service.totalPrice()).toBeCloseTo(50);
    });

    it('should reflect hasItems as false when the cart is empty', () => {
      expect(service.hasItems()).toBe(false);
    });

    it('should reflect hasItems as true after an item is added', () => {
      service.addItem(buildCartItem());

      expect(service.hasItems()).toBe(true);
    });
  });
});
```

---

## Unit Tests — Services with RxJS

```typescript
// core/services/user-state.service.spec.ts
import { TestBed }          from '@angular/core/testing';
import { firstValueFrom }   from 'rxjs';
import { UserStateService } from './user-state.service';
import { IUser }            from '../../models/interfaces/i-user.interface';
import { UserRole }         from '../../models/enums/user-role.enum';

describe('UserStateService', () => {
  let service: UserStateService;

  const buildUser = (overrides?: Partial<IUser>): IUser => ({
    id:        'usr-001',
    email:     'ada@example.com',
    firstName: 'Ada',
    lastName:  'Lovelace',
    role:      UserRole.Admin,
    isActive:  true,
    createdAt: '2024-01-01T00:00:00Z',
    ...overrides,
  });

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(UserStateService);
  });

  it('should emit null before any user is set', async () => {
    const currentUser = await firstValueFrom(service.currentUser$);

    expect(currentUser).toBeNull();
  });

  it('should emit the user after setCurrentUser is called', async () => {
    const testUser = buildUser();
    service.setCurrentUser(testUser);

    const currentUser = await firstValueFrom(service.currentUser$);

    expect(currentUser).toEqual(testUser);
  });

  it('should emit true for isAuthenticated$ when a user is set', async () => {
    service.setCurrentUser(buildUser());

    const isAuthenticated = await firstValueFrom(service.isAuthenticated$);

    expect(isAuthenticated).toBe(true);
  });

  it('should emit false for isAuthenticated$ after clearCurrentUser is called', async () => {
    service.setCurrentUser(buildUser());
    service.clearCurrentUser();

    const isAuthenticated = await firstValueFrom(service.isAuthenticated$);

    expect(isAuthenticated).toBe(false);
  });

  it('should emit the correct role via currentRole$', async () => {
    service.setCurrentUser(buildUser({ role: UserRole.Editor }));

    const currentRole = await firstValueFrom(service.currentRole$);

    expect(currentRole).toBe(UserRole.Editor);
  });
});
```

---

## Component Tests — Angular Testing Library

```typescript
// features/products/components/product-list/product-list.component.spec.ts
import { render, screen, waitFor } from '@testing-library/angular';
import { of, throwError }          from 'rxjs';
import { ProductListComponent }    from './product-list.component';
import { ProductService }          from '../../services/product.service';
import { IProduct }                from '../../../../models/interfaces/i-product.interface';

const buildProduct = (overrides?: Partial<IProduct>): IProduct => ({
  id:    'prod-001',
  name:  'Widget Pro',
  price: 49.99,
  ...overrides,
});

describe('ProductListComponent', () => {
  const renderComponent = (productServiceOverrides: Partial<ProductService> = {}) =>
    render(ProductListComponent, {
      providers: [
        {
          provide: ProductService,
          useValue: {
            getProducts: () => of([buildProduct()]),
            ...productServiceOverrides,
          },
        },
      ],
    });

  it('should display a product card for each product returned by the service', async () => {
    const products = [
      buildProduct({ id: 'prod-001', name: 'Widget Pro' }),
      buildProduct({ id: 'prod-002', name: 'Gadget Basic' }),
    ];

    await renderComponent({ getProducts: () => of(products) });

    await waitFor(() => {
      expect(screen.getByText('Widget Pro')).toBeInTheDocument();
      expect(screen.getByText('Gadget Basic')).toBeInTheDocument();
    });
  });

  it('should show the empty-state message when no products are returned', async () => {
    await renderComponent({ getProducts: () => of([]) });

    await waitFor(() => {
      expect(screen.getByText(/no products are available/i)).toBeInTheDocument();
    });
  });

  it('should display an error message when the service call fails', async () => {
    await renderComponent({
      getProducts: () => throwError(() => new Error('Network error')),
    });

    await waitFor(() => {
      expect(screen.getByRole('alert')).toBeInTheDocument();
    });
  });
});
```

---

## HTTP Mocking

```typescript
// features/orders/services/orders-api.service.spec.ts
import { TestBed }                           from '@angular/core/testing';
import {
  HttpClientTestingModule,
  HttpTestingController,
}                                            from '@angular/common/http/testing';
import { provideHttpClientTesting }          from '@angular/common/http/testing';
import { provideHttpClient }                 from '@angular/common/http';
import { OrdersApiService }                  from './orders-api.service';
import { API_BASE_URL }                      from '../../../core/tokens/api-base-url.token';
import { ORDERS_API }                        from '../../../models/constants/api.constants';

describe('OrdersApiService', () => {
  let service:            OrdersApiService;
  let httpTestingController: HttpTestingController;
  const testApiBaseUrl =  'https://api.test.example.com';

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
        { provide: API_BASE_URL, useValue: testApiBaseUrl },
      ],
    });

    service               = TestBed.inject(OrdersApiService);
    httpTestingController = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // Verify no unexpected HTTP requests were made during the test
    httpTestingController.verify();
  });

  describe('getOrders()', () => {
    it('should send a GET request to the orders endpoint with pagination params', () => {
      service.getOrders(undefined, 2, 10).subscribe();

      const request = httpTestingController.expectOne(
        req => req.url === `${testApiBaseUrl}${ORDERS_API.BASE}`
              && req.params.get('page') === '1'    // page 2 → 0-based index 1
              && req.params.get('size') === '10',
      );

      expect(request.request.method).toBe('GET');
      request.flush({ data: [], totalCount: 0, page: 1, pageSize: 10 });
    });
  });

  describe('deleteOrder()', () => {
    it('should send a DELETE request to the correct order URL', () => {
      service.deleteOrder('ord-999').subscribe();

      const request = httpTestingController.expectOne(
        `${testApiBaseUrl}${ORDERS_API.BY_ID('ord-999')}`,
      );

      expect(request.request.method).toBe('DELETE');
      request.flush(null, { status: 204, statusText: 'No Content' });
    });
  });
});
```

---

## Testing NgRx Facades

```typescript
// features/orders/store/orders.facade.spec.ts
import { TestBed }          from '@angular/core/testing';
import { provideMockStore, MockStore } from '@ngrx/store/testing';
import { OrdersFacade }     from './orders.facade';
import { OrdersActions }    from './orders.actions';
import { selectOrders }     from './orders.reducer';
import { OrderStatus }      from '../../../models/enums/order-status.enum';
import { IOrder }           from '../../../models/interfaces/i-order.interface';
import { firstValueFrom }   from 'rxjs';

describe('OrdersFacade', () => {
  let facade:     OrdersFacade;
  let mockStore:  MockStore;

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

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideMockStore({
          initialState: { orders: { orders: [], isLoading: false, errorMessage: null, selectedOrderId: null } },
        }),
      ],
    });

    facade    = TestBed.inject(OrdersFacade);
    mockStore = TestBed.inject(MockStore);
  });

  it('should dispatch loadOrders action when loadOrders() is called', () => {
    const dispatchSpy = jest.spyOn(mockStore, 'dispatch');

    facade.loadOrders();

    expect(dispatchSpy).toHaveBeenCalledWith(OrdersActions.loadOrders());
  });

  it('should emit orders from the store via orders$', async () => {
    const testOrders = [buildOrder({ orderId: 'ord-001' })];
    mockStore.overrideSelector(selectOrders, testOrders);
    mockStore.refreshState();

    const emittedOrders = await firstValueFrom(facade.orders$);

    expect(emittedOrders).toEqual(testOrders);
  });
});
```

---

## Testing Guards

```typescript
// core/guards/auth.guard.spec.ts
import { TestBed }        from '@angular/core/testing';
import { Router }         from '@angular/router';
import { RouterTestingModule } from '@angular/router/testing';
import { authGuard }      from './auth.guard';
import { AuthService }    from '../services/auth.service';

describe('authGuard', () => {
  let authService: AuthService;
  let router:      Router;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [RouterTestingModule],
      providers: [
        {
          provide: AuthService,
          useValue: { isLoggedIn: jest.fn() },
        },
      ],
    });

    authService = TestBed.inject(AuthService);
    router      = TestBed.inject(Router);
  });

  it('should allow navigation when the user is logged in', () => {
    jest.spyOn(authService, 'isLoggedIn').mockReturnValue(true);

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as never, { url: '/dashboard' } as never)
    );

    expect(result).toBe(true);
  });

  it('should redirect to /login with returnUrl when the user is not logged in', () => {
    jest.spyOn(authService, 'isLoggedIn').mockReturnValue(false);

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as never, { url: '/dashboard' } as never)
    );

    expect(result).not.toBe(true);
    expect(result.toString()).toContain('/login');
  });
});
```

---

## Async Testing Patterns

```typescript
// Use fakeAsync + tick() for timer-based code in Karma/Jasmine
it('should auto-hide a notification after 3000ms', fakeAsync(() => {
  service.showNotification('Saved successfully');
  expect(service.isVisible()).toBe(true);

  tick(3000);

  expect(service.isVisible()).toBe(false);
}));

// Use firstValueFrom() for one-shot observable assertions (cleaner than done callback)
it('should emit the updated order after patchOrder resolves', async () => {
  const updatedOrder = await firstValueFrom(service.patchOrder('ord-001', { status: OrderStatus.Shipped }));
  expect(updatedOrder.status).toBe(OrderStatus.Shipped);
});
```

---

## Naming & Coverage Guidelines

**Test file location:** co-locate with the file under test (`*.spec.ts` alongside `*.ts`).

**Test naming convention — use full English sentences:**
```typescript
// ✅ Correct — reads like a specification
it('should redirect to /login when the auth token has expired')
it('should disable the submit button while the form is submitting')
it('should emit an empty array when no items match the filter criteria')

// ❌ Wrong — too vague or too implementation-focused
it('works')
it('calls the service')
it('ngOnInit test')
```

**Coverage targets:**

| Layer | Minimum |
|---|---|
| Services (business logic) | 80% |
| Components | 70% |
| Guards & interceptors | 80% |
| Mappers | 90% (pure functions — easy to test exhaustively) |
| Reducers | 90% (pure functions) |
| Effects | 70% |
