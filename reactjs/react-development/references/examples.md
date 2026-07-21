# Examples — Full Reference

Complete annotated examples showing the skill's conventions working together.

---

## Table of Contents
1. [Orders List with TanStack Query, loading/error/empty states](#example-1-orders-list-with-tanstack-query)
2. [Zustand cart store + component consuming it](#example-2-zustand-cart-feature)
3. [Accessible login form with React Hook Form + Zod](#example-3-accessible-login-form)
4. [RTK slice + facade hook + consuming component](#example-4-rtk-facade-pattern)

---

## Example 1: Orders List with TanStack Query

Demonstrates: TanStack Query, custom hook, query key constants, mapper in service,
loading/error/empty state, accessible list rendering.

**Input (what the user asks):**
> "Create an OrdersList component that fetches orders from the API using TanStack Query,
> shows a loading spinner while fetching, an error alert on failure, and an empty state message."

**Expected output:**

```typescript
// features/orders/constants/orders-query-keys.constants.ts
export const ORDERS_QUERY_KEYS = {
  all:    ['orders']                                        as const,
  byId:   (orderId: string) => ['orders', orderId]         as const,
} as const;
```

```typescript
// features/orders/services/orders-api.service.ts
import { apiClient }   from '../../../core/http/api-client';
import { OrderMapper } from '../../../models/mappers/order.mapper';
import { ORDERS_API }  from '../../../models/constants/api.constants';
import type { IOrder } from '../../../models/interfaces/i-order.interface';

export const ordersApiService = {
  async getOrders(): Promise<IOrder[]> {
    const response = await apiClient.get<Record<string, unknown>[]>(ORDERS_API.BASE);
    return OrderMapper.fromApiList(response.data);
  },
} as const;
```

```typescript
// features/orders/hooks/use-orders.hook.ts
import { useQuery }         from '@tanstack/react-query';
import { ordersApiService } from '../services/orders-api.service';
import { ORDERS_QUERY_KEYS } from '../constants/orders-query-keys.constants';

/**
 * Fetches all orders with TanStack Query caching.
 * Exposes isLoading, isError, error, and data for components to consume.
 */
export function useOrders() {
  return useQuery({
    queryKey: ORDERS_QUERY_KEYS.all,
    queryFn:  () => ordersApiService.getOrders(),
    staleTime: 5 * 60 * 1000,
  });
}
```

```typescript
// features/orders/components/orders-list/orders-list.component.tsx
import { useOrders }       from '../../hooks/use-orders.hook';
import { OrderCard }       from '../order-card/order-card.component';
import { LoadingSpinner }  from '../../../../shared/components/loading-spinner/loading-spinner.component';

/**
 * Displays the orders list with loading, error, and empty feedback.
 * All server state is owned by TanStack Query via the useOrders hook.
 *
 * Used on: /orders (main list view)
 */
export function OrdersList(): React.ReactElement {
  const { data: orders, isLoading, isError, error } = useOrders();

  if (isLoading) {
    return <LoadingSpinner aria-label="Loading orders" />;
  }

  if (isError) {
    return (
      <p className="error-message" role="alert" aria-live="assertive">
        {error instanceof Error ? error.message : 'Failed to load orders. Please try again.'}
      </p>
    );
  }

  if (!orders || orders.length === 0) {
    return <p className="empty-state">No orders found.</p>;
  }

  return (
    <section aria-label="Orders list">
      <ul className="orders-grid">
        {orders.map((order) => (
          <li key={order.orderId}>
            <OrderCard order={order} />
          </li>
        ))}
      </ul>
    </section>
  );
}
```

---

## Example 2: Zustand Cart Feature

Demonstrates: Zustand store with immer, selector functions, consuming hook, component.

**Input (what the user asks):**
> "Create a cart feature using Zustand. The store should support adding, removing,
> and updating quantity of items, with derived totals. Show a CartSummary component
> consuming the store."

**Expected output:**

```typescript
// features/cart/store/cart.store.ts
import { create } from 'zustand';
import { immer }  from 'zustand/middleware/immer';
import type { ICartItem } from '../../../models/interfaces/i-cart-item.interface';

interface CartState {
  items: ICartItem[];
}
interface CartActions {
  addItem:        (item: ICartItem) => void;
  removeItem:     (itemId: string) => void;
  updateQuantity: (itemId: string, newQuantity: number) => void;
  clearCart:      () => void;
}

/**
 * Shopping cart store.
 * Immer middleware allows direct mutations inside set() — they are applied immutably.
 */
export const useCartStore = create<CartState & CartActions>()(
  immer((set) => ({
    items: [],

    addItem: (newItem) =>
      set((cartState) => {
        const existingItem = cartState.items.find((item) => item.id === newItem.id);
        if (existingItem) {
          existingItem.quantity += 1;
        } else {
          cartState.items.push({ ...newItem, quantity: 1 });
        }
      }),

    removeItem: (itemId) =>
      set((cartState) => {
        cartState.items = cartState.items.filter((item) => item.id !== itemId);
      }),

    updateQuantity: (itemId, newQuantity) =>
      set((cartState) => {
        if (newQuantity <= 0) {
          cartState.items = cartState.items.filter((item) => item.id !== itemId);
          return;
        }
        const targetItem = cartState.items.find((item) => item.id === itemId);
        if (targetItem) targetItem.quantity = newQuantity;
      }),

    clearCart: () => set({ items: [] }),
  })),
);

export const selectCartTotalPrice = (cartState: CartState) =>
  cartState.items.reduce((sum, item) => sum + item.unitPrice * item.quantity, 0);

export const selectCartItemCount = (cartState: CartState) =>
  cartState.items.reduce((sum, item) => sum + item.quantity, 0);

export const selectHasCartItems = (cartState: CartState) =>
  cartState.items.length > 0;
```

```typescript
// features/cart/components/cart-summary/cart-summary.component.tsx
import {
  useCartStore,
  selectCartItemCount,
  selectCartTotalPrice,
  selectHasCartItems,
} from '../../store/cart.store';

/**
 * Displays the cart count and total. Subscribes only to derived selectors
 * to minimise re-renders when unrelated parts of the cart change.
 */
export function CartSummary(): React.ReactElement {
  const itemCount     = useCartStore(selectCartItemCount);
  const totalPrice    = useCartStore(selectCartTotalPrice);
  const hasCartItems  = useCartStore(selectHasCartItems);
  const clearCart     = useCartStore((cartState) => cartState.clearCart);

  return (
    <aside className="cart-summary" aria-label="Cart summary">
      <p>{itemCount} item{itemCount !== 1 ? 's' : ''}</p>
      <p>{totalPrice.toFixed(2)}</p>
      <button
        type="button"
        onClick={clearCart}
        disabled={!hasCartItems}
        aria-disabled={!hasCartItems}
      >
        Clear cart
      </button>
    </aside>
  );
}
```

---

## Example 3: Accessible Login Form

Demonstrates: React Hook Form, Zod schema, zodResolver, aria-live errors, isSubmitting,
keyboard accessibility, error handling.

**Input (what the user asks):**
> "Create a LoginForm with email and password fields, Zod validation, inline accessible
> error messages, and a submit button disabled while the request is in flight."

**Expected output:**

```typescript
// models/validators/schemas/login.schema.ts
import { z } from 'zod';

/**
 * Zod schema for the login form.
 * TypeScript type is inferred — never written twice.
 */
export const loginSchema = z.object({
  email: z.string()
    .min(1, 'Email address is required.')
    .email('Enter a valid email address.'),
  password: z.string()
    .min(1, 'Password is required.')
    .min(8, 'Password must be at least 8 characters.')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter.')
    .regex(/[0-9]/, 'Password must contain at least one number.'),
  rememberMe: z.boolean().default(false),
});

export type ILoginFormValues = z.infer<typeof loginSchema>;
```

```typescript
// features/auth/components/login-form/login-form.component.tsx
import { useForm }        from 'react-hook-form';
import { zodResolver }    from '@hookform/resolvers/zod';
import { useNavigate }    from 'react-router-dom';
import { loginSchema, type ILoginFormValues } from '../../../../models/validators/schemas/login.schema';
import { authApiService } from '../../services/auth-api.service';
import { APP_ROUTE_PATHS } from '../../../../models/constants/app.constants';

/**
 * Handles user authentication via a strictly-typed reactive form.
 * Validates email format and password complexity before submission.
 * All errors are surfaced inline and announced to screen readers.
 */
export function LoginForm(): React.ReactElement {
  const navigate = useNavigate();

  const {
    register,
    handleSubmit,
    setError,
    formState: { errors, isSubmitting },
  } = useForm<ILoginFormValues>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = async (formData: ILoginFormValues): Promise<void> => {
    try {
      await authApiService.login(formData);
      await navigate(APP_ROUTE_PATHS.DASHBOARD);
    } catch (loginError) {
      setError('root', {
        message: loginError instanceof Error
          ? loginError.message
          : 'Login failed. Please check your credentials and try again.',
      });
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate aria-label="Sign in form">

      {/* Root-level error (e.g. wrong credentials) */}
      {errors.root && (
        <p className="form-error" role="alert" aria-live="assertive">
          {errors.root.message}
        </p>
      )}

      {/* Email field */}
      <div className="form-field">
        <label htmlFor="email">Email address</label>
        <input
          id="email"
          type="email"
          autoComplete="email"
          aria-invalid={!!errors.email}
          aria-describedby="email-error"
          {...register('email')}
        />
        <span id="email-error" role="alert" aria-live="polite" className="field-error">
          {errors.email?.message}
        </span>
      </div>

      {/* Password field */}
      <div className="form-field">
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          autoComplete="current-password"
          aria-invalid={!!errors.password}
          aria-describedby="password-error"
          {...register('password')}
        />
        <span id="password-error" role="alert" aria-live="polite" className="field-error">
          {errors.password?.message}
        </span>
      </div>

      {/* Remember me */}
      <div className="form-field form-field--inline">
        <input id="rememberMe" type="checkbox" {...register('rememberMe')} />
        <label htmlFor="rememberMe">Keep me signed in</label>
      </div>

      {/* Submit */}
      <button
        type="submit"
        className="btn btn--primary"
        disabled={isSubmitting}
        aria-busy={isSubmitting}
      >
        {isSubmitting ? 'Signing in…' : 'Sign in'}
      </button>

    </form>
  );
}
```

---

## Example 4: RTK Facade Pattern

Demonstrates: RTK slice, typed thunk, selectors, facade hook, consuming component.
Use this pattern when RTK is needed for app-wide client state (not server data).

**Input (what the user asks):**
> "Create a notifications feature with RTK: a slice that holds a list of notifications,
> a hook that wraps dispatch and selectors, and a component that renders them."

**Expected output:**

```typescript
// features/notifications/store/notifications.slice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

export type NotificationType = 'success' | 'error' | 'warning' | 'info';

export interface INotification {
  readonly id:      string;
  message:          string;
  type:             NotificationType;
  isDismissed:      boolean;
}

interface NotificationsState {
  notifications: INotification[];
}

const initialNotificationsState: NotificationsState = {
  notifications: [],
};

export const notificationsSlice = createSlice({
  name:         'notifications',
  initialState: initialNotificationsState,
  reducers: {
    /**
     * Adds a new notification to the list.
     * @param notification - The notification to display; id must be pre-generated.
     */
    addNotification: (notificationsState, action: PayloadAction<INotification>) => {
      notificationsState.notifications.push(action.payload);
    },
    /**
     * Marks a notification as dismissed so it can be animated out.
     * @param notificationId - ID of the notification to dismiss
     */
    dismissNotification: (notificationsState, action: PayloadAction<string>) => {
      const targetNotification = notificationsState.notifications.find(
        (notification) => notification.id === action.payload,
      );
      if (targetNotification) {
        targetNotification.isDismissed = true;
      }
    },
    /** Removes all dismissed notifications from the list. */
    clearDismissedNotifications: (notificationsState) => {
      notificationsState.notifications = notificationsState.notifications.filter(
        (notification) => !notification.isDismissed,
      );
    },
  },
});

export const {
  addNotification,
  dismissNotification,
  clearDismissedNotifications,
} = notificationsSlice.actions;
export default notificationsSlice.reducer;
```

```typescript
// features/notifications/hooks/use-notifications.hook.ts
import { useDispatch, useSelector } from 'react-redux';
import { nanoid }                   from '@reduxjs/toolkit';
import {
  addNotification,
  dismissNotification,
  clearDismissedNotifications,
  type NotificationType,
} from '../store/notifications.slice';
import type { RootState }  from '../../../app/store';
import type { AppDispatch } from '../../../app/store';

/**
 * Facade hook for the Notifications RTK slice.
 * Components never import useDispatch or slice actions directly.
 */
export function useNotifications() {
  const dispatch     = useDispatch<AppDispatch>();
  const notifications = useSelector(
    (state: RootState) => state.notifications.notifications,
  );

  return {
    notifications,
    activeNotifications: notifications.filter((notification) => !notification.isDismissed),

    showNotification: (message: string, type: NotificationType = 'info') =>
      dispatch(addNotification({ id: nanoid(), message, type, isDismissed: false })),

    dismiss: (notificationId: string) =>
      dispatch(dismissNotification(notificationId)),

    clearDismissed: () =>
      dispatch(clearDismissedNotifications()),
  };
}
```

```typescript
// features/notifications/components/notification-list/notification-list.component.tsx
import { useEffect }          from 'react';
import { useNotifications }   from '../../hooks/use-notifications.hook';

/**
 * Renders active notifications as a live region.
 * Dismissed notifications are cleared after a 500ms animation delay.
 */
export function NotificationList(): React.ReactElement {
  const { activeNotifications, dismiss, clearDismissed } = useNotifications();

  // Remove dismissed notifications after the CSS exit animation completes
  useEffect(() => {
    const dismissTimer = setTimeout(clearDismissed, 500);
    return () => clearTimeout(dismissTimer);
  }, [clearDismissed]);

  return (
    <ul
      className="notification-list"
      aria-live="polite"
      aria-label="Notifications"
      aria-atomic="false"
    >
      {activeNotifications.map((notification) => (
        <li
          key={notification.id}
          className={`notification notification--${notification.type}`}
          role="status"
        >
          <span className="notification__message">{notification.message}</span>
          <button
            type="button"
            className="notification__dismiss"
            onClick={() => dismiss(notification.id)}
            aria-label={`Dismiss notification: ${notification.message}`}
          >
            ×
          </button>
        </li>
      ))}
    </ul>
  );
}
```
