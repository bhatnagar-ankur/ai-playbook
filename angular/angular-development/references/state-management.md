# State Management — Full Reference

Complete code examples for all three tiers.
For the decision table and tier selection rules see the **State Management** section in `SKILL.md`.

---

## Table of Contents
1. [Tier 1 — Angular Signals](#tier-1--angular-signals)
2. [Tier 2 — RxJS BehaviorSubject](#tier-2--rxjs-behaviorsubject)
3. [Tier 3 — NgRx with Facade](#tier-3--ngrx-with-facade)
4. [Migrating Between Tiers](#migrating-between-tiers)

---

## Tier 1 — Angular Signals

Best for: component-local or single-feature state, no cross-feature dependencies.

```typescript
// features/cart/services/cart.service.ts

import { computed, inject, Injectable, signal } from '@angular/core';
import { ICartItem } from '../../../models/interfaces/i-cart-item.interface';

/**
 * Manages shopping cart state using Angular Signals.
 * Scoped to the cart feature; no cross-feature state dependencies.
 * Use this as the reference pattern for Tier 1 signal-based services.
 */
@Injectable({ providedIn: 'root' })
export class CartService {
  /** Internal mutable cart items — never expose the writable signal directly. */
  private _items = signal<ICartItem[]>([]);

  /** Public read-only view of all items in the cart. */
  items = this._items.asReadonly();

  /** Number of distinct product lines in the cart (not total quantity). */
  itemCount = computed(() => this._items().length);

  /** Total quantity across all line items. */
  totalQuantity = computed(() =>
    this._items().reduce((sum, item) => sum + item.quantity, 0)
  );

  /** Total price of all items including quantities. */
  totalPrice = computed(() =>
    this._items().reduce((sum, item) => sum + item.unitPrice * item.quantity, 0)
  );

  /** True when the cart contains at least one item. */
  hasItems = computed(() => this._items().length > 0);

  /**
   * Adds an item to the cart. If an item with the same id already exists,
   * its quantity is incremented rather than creating a duplicate entry.
   * @param newItem - The cart item to add; must have a stable unique id
   */
  addItem(newItem: ICartItem): void {
    this._items.update(currentItems => {
      const existingItem = currentItems.find(item => item.id === newItem.id);
      if (existingItem) {
        return currentItems.map(item =>
          item.id === newItem.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        );
      }
      return [...currentItems, { ...newItem, quantity: 1 }];
    });
  }

  /**
   * Removes an item from the cart entirely, regardless of quantity.
   * @param itemId - Unique identifier of the item to remove
   */
  removeItem(itemId: string): void {
    this._items.update(currentItems =>
      currentItems.filter(item => item.id !== itemId)
    );
  }

  /**
   * Updates the quantity of an existing item.
   * Removes the item if the new quantity is zero or negative.
   * @param itemId      - Unique identifier of the item to update
   * @param newQuantity - Target quantity; values ≤ 0 trigger removal
   */
  updateQuantity(itemId: string, newQuantity: number): void {
    if (newQuantity <= 0) {
      this.removeItem(itemId);
      return;
    }
    this._items.update(currentItems =>
      currentItems.map(item =>
        item.id === itemId ? { ...item, quantity: newQuantity } : item
      )
    );
  }

  /** Removes all items from the cart. */
  clearCart(): void {
    this._items.set([]);
  }
}
```

**Template usage — always call signals as functions:**

```html
@if (cartService.hasItems()) {
  <p>{{ cartService.itemCount() }} items · {{ cartService.totalPrice() | currency }}</p>
} @else {
  <p class="empty-state">Your cart is empty.</p>
}
```

---

## Tier 2 — RxJS BehaviorSubject

Best for: state shared across 2–3 features where reactive streams are needed.

```typescript
// core/services/user-state.service.ts

import { inject, Injectable }     from '@angular/core';
import { BehaviorSubject, map }   from 'rxjs';
import { IUser }                  from '../../models/interfaces/i-user.interface';
import { UserRole }               from '../../models/enums/user-role.enum';

/**
 * Provides a reactive stream of the currently authenticated user's profile.
 * Subscribe in components, guards, or resolvers that react to auth-state changes.
 *
 * Rule: never expose a BehaviorSubject directly.
 * Always expose the public observable via .asObservable().
 */
@Injectable({ providedIn: 'root' })
export class UserStateService {
  /** Internal subject — only this service may call .next(). */
  private _currentUser$ = new BehaviorSubject<IUser | null>(null);

  /**
   * Emits the currently authenticated user, or null when no user is logged in.
   * Subscribe with takeUntilDestroyed() — never subscribe without cleanup.
   */
  currentUser$ = this._currentUser$.asObservable();

  /** Emits true while a user is authenticated. */
  isAuthenticated$ = this.currentUser$.pipe(map(user => user !== null));

  /** Emits the current user's role, or null when unauthenticated. */
  currentRole$ = this.currentUser$.pipe(map(user => user?.role ?? null));

  /** Emits true when the current user has the Admin role. */
  isAdmin$ = this.currentRole$.pipe(map(role => role === UserRole.Admin));

  /**
   * Updates the current user. Call after a successful login or profile update.
   * @param user - The authenticated user record, or null to clear the session
   */
  setCurrentUser(user: IUser | null): void {
    this._currentUser$.next(user);
  }

  /** Clears the current user — equivalent to a logout from the state perspective. */
  clearCurrentUser(): void {
    this._currentUser$.next(null);
  }
}
```

**Consuming in a component with takeUntilDestroyed:**

```typescript
/**
 * Displays a personalised greeting using the current user's display name.
 * Subscribes to UserStateService and cleans up automatically on destroy.
 */
@Component({
  selector: 'app-user-greeting',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    @if (currentUser$ | async; as currentUser) {
      <p>Welcome back, {{ currentUser.firstName }}!</p>
    }
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserGreetingComponent {
  // Expose the stream directly for the async pipe — avoids manual subscription
  currentUser$ = inject(UserStateService).currentUser$;
}
```

---

## Tier 3 — NgRx with Facade

Best for: app-wide state, complex async side effects, strict audit trails.

### Folder structure

```
features/orders/store/
├── orders.actions.ts
├── orders.reducer.ts
├── orders.effects.ts
├── orders.selectors.ts
└── orders.facade.ts        ← Components ONLY talk to the Facade
```

### Actions

```typescript
// features/orders/store/orders.actions.ts

import { createActionGroup, emptyProps, props } from '@ngrx/store';
import { IOrder, ICreateOrderDto }              from '../../../models/interfaces/i-order.interface';

/**
 * Action group for the Orders feature.
 * Grouping actions keeps related actions co-located and auto-generates
 * consistent type strings (e.g. "[Orders] Load Orders").
 */
export const OrdersActions = createActionGroup({
  source: 'Orders',
  events: {
    'Load Orders':          emptyProps(),
    'Load Orders Success':  props<{ orders: IOrder[] }>(),
    'Load Orders Failure':  props<{ errorMessage: string }>(),

    'Create Order':         props<{ orderData: ICreateOrderDto }>(),
    'Create Order Success': props<{ createdOrder: IOrder }>(),
    'Create Order Failure': props<{ errorMessage: string }>(),

    'Select Order':         props<{ orderId: string }>(),
    'Clear Selected Order': emptyProps(),
  },
});
```

### Reducer

```typescript
// features/orders/store/orders.reducer.ts

import { createFeature, createReducer, on } from '@ngrx/store';
import { IOrder }                           from '../../../models/interfaces/i-order.interface';
import { OrdersActions }                    from './orders.actions';

/** Shape of the Orders feature state slice. */
interface IOrdersState {
  orders:          IOrder[];
  selectedOrderId: string | null;
  isLoading:       boolean;
  errorMessage:    string | null;
}

const initialState: IOrdersState = {
  orders:          [],
  selectedOrderId: null,
  isLoading:       false,
  errorMessage:    null,
};

/**
 * Feature-level reducer for the Orders domain.
 * createFeature() auto-generates selectors for every top-level state property.
 */
export const ordersFeature = createFeature({
  name: 'orders',
  reducer: createReducer(
    initialState,
    on(OrdersActions.loadOrders, state => ({
      ...state,
      isLoading:    true,
      errorMessage: null,
    })),
    on(OrdersActions.loadOrdersSuccess, (state, { orders }) => ({
      ...state,
      orders,
      isLoading: false,
    })),
    on(OrdersActions.loadOrdersFailure, (state, { errorMessage }) => ({
      ...state,
      isLoading:    false,
      errorMessage,
    })),
    on(OrdersActions.createOrderSuccess, (state, { createdOrder }) => ({
      ...state,
      orders: [...state.orders, createdOrder],
    })),
    on(OrdersActions.selectOrder, (state, { orderId }) => ({
      ...state,
      selectedOrderId: orderId,
    })),
    on(OrdersActions.clearSelectedOrder, state => ({
      ...state,
      selectedOrderId: null,
    })),
  ),
});

// Auto-generated selectors from createFeature():
export const {
  selectOrdersState,
  selectOrders,
  selectSelectedOrderId,
  selectIsLoading,
  selectErrorMessage,
} = ordersFeature;
```

### Effects

```typescript
// features/orders/store/orders.effects.ts

import { inject, Injectable }            from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, of, switchMap } from 'rxjs';
import { OrdersApiService }              from '../services/orders-api.service';
import { OrdersActions }                 from './orders.actions';

/**
 * Side-effect handlers for the Orders feature.
 * Each effect listens for an action, performs async work,
 * then dispatches a success or failure action.
 */
@Injectable()
export class OrdersEffects {
  private actions$ = inject(Actions);
  private ordersApiService = inject(OrdersApiService);

  /** Fetches all orders when loadOrders is dispatched. */
  loadOrders$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrdersActions.loadOrders),
      switchMap(() =>
        this.ordersApiService.getOrders().pipe(
          map(orders     => OrdersActions.loadOrdersSuccess({ orders })),
          catchError(err => of(OrdersActions.loadOrdersFailure({
            errorMessage: err.message ?? 'Failed to load orders.',
          }))),
        )
      ),
    )
  );

  /** Creates a new order when createOrder is dispatched. */
  createOrder$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrdersActions.createOrder),
      switchMap(({ orderData }) =>
        this.ordersApiService.createOrder(orderData).pipe(
          map(createdOrder => OrdersActions.createOrderSuccess({ createdOrder })),
          catchError(err   => of(OrdersActions.createOrderFailure({
            errorMessage: err.message ?? 'Failed to create order.',
          }))),
        )
      ),
    )
  );
}
```

### Selectors

```typescript
// features/orders/store/orders.selectors.ts

import { createSelector }   from '@ngrx/store';
import { OrderStatus }      from '../../../models/enums/order-status.enum';
import { selectOrders, selectSelectedOrderId } from './orders.reducer';

/**
 * Derived selector: the currently selected IOrder object,
 * or null when no order is selected or the ID does not match.
 */
export const selectSelectedOrder = createSelector(
  selectOrders,
  selectSelectedOrderId,
  (orders, selectedId) => orders.find(order => order.orderId === selectedId) ?? null,
);

/** Derived selector: orders with Pending status only. */
export const selectPendingOrders = createSelector(
  selectOrders,
  orders => orders.filter(order => order.status === OrderStatus.Pending),
);

/** Derived selector: total number of orders currently in the store. */
export const selectOrderCount = createSelector(
  selectOrders,
  orders => orders.length,
);
```

### Facade

```typescript
// features/orders/store/orders.facade.ts

import { inject, Injectable }           from '@angular/core';
import { Store }                        from '@ngrx/store';
import { ICreateOrderDto }              from '../../../models/interfaces/i-order.interface';
import { OrdersActions }                from './orders.actions';
import {
  selectOrders,
  selectIsLoading,
  selectErrorMessage,
}                                       from './orders.reducer';
import {
  selectSelectedOrder,
  selectPendingOrders,
  selectOrderCount,
}                                       from './orders.selectors';

/**
 * Public API for the Orders NgRx feature slice.
 *
 * Components, resolvers, and guards interact exclusively with this Facade.
 * They never import Store, dispatch actions, or call selectors directly.
 * This boundary makes it trivial to swap the underlying state tier without
 * changing any consumer code.
 */
@Injectable({ providedIn: 'root' })
export class OrdersFacade {
  private store = inject(Store);

  // ── Observables (selectors) ──────────────────────────────────────────────

  /** Emits the full list of orders held in the store. */
  orders$         = this.store.select(selectOrders);

  /** Emits the currently selected order, or null when none is selected. */
  selectedOrder$  = this.store.select(selectSelectedOrder);

  /** Emits orders that are currently in Pending status. */
  pendingOrders$  = this.store.select(selectPendingOrders);

  /** Emits the total number of orders in the store. */
  orderCount$     = this.store.select(selectOrderCount);

  /** Emits true while an API request is in flight. */
  isLoading$      = this.store.select(selectIsLoading);

  /** Emits a user-facing error message, or null when no error is present. */
  errorMessage$   = this.store.select(selectErrorMessage);

  // ── Dispatchers ──────────────────────────────────────────────────────────

  /** Triggers loading of all orders from the API. */
  loadOrders(): void {
    this.store.dispatch(OrdersActions.loadOrders());
  }

  /**
   * Submits a new order to the API and adds it to the store on success.
   * @param orderData - Validated order payload
   */
  createOrder(orderData: ICreateOrderDto): void {
    this.store.dispatch(OrdersActions.createOrder({ orderData }));
  }

  /**
   * Sets the active order selection in the store.
   * @param orderId - ID of the order to select
   */
  selectOrder(orderId: string): void {
    this.store.dispatch(OrdersActions.selectOrder({ orderId }));
  }

  /** Clears the active order selection. */
  clearSelectedOrder(): void {
    this.store.dispatch(OrdersActions.clearSelectedOrder());
  }
}
```

---

## Migrating Between Tiers

Migration should be driven by real friction — don't move tiers pre-emptively.

| Signal (T1 → T2) | When the service is injected in more than 2–3 features and reactive stream operators (debounceTime, combineLatest, etc.) would simplify the logic |
|---|---|
| RxJS (T2 → T3) | When async side effects need retry/cancel logic, when you need time-travel debugging, or when the state must be auditable across sessions |
| NgRx (T3 → T2) | Rare — only when the app scope shrinks significantly and the NgRx overhead outweighs the benefit |

The **Facade pattern** in Tier 3 is what makes downward migration possible without
touching components — components already talk to the Facade interface, which can be
re-implemented in Tier 2 without any template changes.
