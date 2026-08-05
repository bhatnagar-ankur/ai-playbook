# State Management — Full Reference

Complete code examples for all five tiers.
For the decision table and tier selection rules see the **State Management** section in `SKILL.md`.

---

## Table of Contents
1. [Tier 1 — useState / useReducer (Local)](#tier-1--usestate--usereducer)
2. [Tier 2 — Zustand (Feature-shared)](#tier-2--zustand)
3. [Tier 2 alt — Jotai (Atomic)](#tier-2-alt--jotai)
4. [Tier 3 — TanStack Query (Server state)](#tier-3--tanstack-query)
5. [Tier 4 — Redux Toolkit (App-wide client state)](#tier-4--redux-toolkit)
6. [Tier 5 — React Context + useReducer (Cross-cutting)](#tier-5--react-context--usereducer)
7. [Mixing Tiers](#mixing-tiers)

---

## Tier 1 — useState / useReducer

Best for: component-local UI state, no cross-component sharing needed.

```typescript
// features/cart/components/cart-item/cart-item.component.tsx

import { useState, useCallback } from 'react';
import type { ICartItem }        from '../../../../models/interfaces/i-cart-item.interface';

interface CartItemProps {
  readonly item:     ICartItem;
  onRemove:          (itemId: string) => void;
  onQuantityChange:  (itemId: string, newQuantity: number) => void;
}

/**
 * Renders a single cart line item with an inline quantity editor.
 * Local editing state is managed with useState — no store needed.
 */
export function CartItem({ item, onRemove, onQuantityChange }: CartItemProps): React.ReactElement {
  const [localQuantity, setLocalQuantity] = useState(item.quantity);

  const handleQuantityChange = useCallback(
    (event: React.ChangeEvent<HTMLInputElement>) => {
      const newQuantity = Number(event.target.value);
      setLocalQuantity(newQuantity);
      onQuantityChange(item.id, newQuantity);
    },
    [item.id, onQuantityChange],
  );

  return (
    <li className="cart-item">
      <span className="cart-item__name">{item.name}</span>
      <input
        type="number"
        min={1}
        value={localQuantity}
        onChange={handleQuantityChange}
        aria-label={`Quantity for ${item.name}`}
      />
      <button
        type="button"
        onClick={() => onRemove(item.id)}
        aria-label={`Remove ${item.name} from cart`}
      >
        Remove
      </button>
    </li>
  );
}
```

**useReducer** — when local state has multiple sub-values that transition together:

```typescript
// features/wizard/hooks/use-wizard-state.hook.ts

type WizardStep = 'details' | 'review' | 'confirm';

interface WizardState {
  currentStep:  WizardStep;
  isSubmitting: boolean;
  hasError:     boolean;
  errorMessage: string | null;
}

type WizardAction =
  | { type: 'NEXT_STEP'; step: WizardStep }
  | { type: 'SUBMIT_START' }
  | { type: 'SUBMIT_SUCCESS' }
  | { type: 'SUBMIT_FAILURE'; message: string };

const initialWizardState: WizardState = {
  currentStep:  'details',
  isSubmitting: false,
  hasError:     false,
  errorMessage: null,
};

function wizardReducer(state: WizardState, action: WizardAction): WizardState {
  switch (action.type) {
    case 'NEXT_STEP':
      return { ...state, currentStep: action.step };
    case 'SUBMIT_START':
      return { ...state, isSubmitting: true, hasError: false, errorMessage: null };
    case 'SUBMIT_SUCCESS':
      return { ...state, isSubmitting: false };
    case 'SUBMIT_FAILURE':
      return { ...state, isSubmitting: false, hasError: true, errorMessage: action.message };
    default:
      return state;
  }
}

/** Manages multi-step wizard state with typed action dispatch. */
export function useWizardState() {
  return useReducer(wizardReducer, initialWizardState);
}
```

---

## Tier 2 — Zustand

Best for: state shared across 2–3 components in the same feature, no server data involved.

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
 * Zustand store for shopping cart state.
 * Uses immer middleware so mutations inside `set` are applied immutably.
 * Export the hook — never export the raw store object.
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

// Derived selectors — co-locate with the store so consumers import them alongside the hook
export const selectCartItemCount  = (cartState: CartState): number =>
  cartState.items.reduce((sum, item) => sum + item.quantity, 0);

export const selectCartTotalPrice = (cartState: CartState): number =>
  cartState.items.reduce((sum, item) => sum + item.unitPrice * item.quantity, 0);

export const selectHasCartItems   = (cartState: CartState): boolean =>
  cartState.items.length > 0;
```

**Consuming the Zustand store in a component:**

```typescript
// features/cart/components/cart-summary/cart-summary.component.tsx

import { useCartStore, selectCartItemCount, selectCartTotalPrice } from '../../store/cart.store';

/**
 * Displays the cart item count and total price.
 * Subscribes to only the fields it needs via the selector pattern.
 */
export function CartSummary(): React.ReactElement {
  const itemCount  = useCartStore(selectCartItemCount);
  const totalPrice = useCartStore(selectCartTotalPrice);
  const clearCart  = useCartStore((cartState) => cartState.clearCart);

  return (
    <aside aria-label="Cart summary">
      <p>{itemCount} item{itemCount !== 1 ? 's' : ''}</p>
      <p>{totalPrice.toFixed(2)}</p>
      <button type="button" onClick={clearCart} disabled={itemCount === 0}>
        Clear cart
      </button>
    </aside>
  );
}
```

---

## Tier 2 alt — Jotai

Best for: fine-grained atomic state where only the components that read a specific atom re-render.

```typescript
// features/ui/atoms/sidebar.atoms.ts

import { atom } from 'jotai';

/** Controls whether the global sidebar navigation is open. */
export const isSidebarOpenAtom = atom<boolean>(false);

/** Derived atom — true only when sidebar is open on a mobile viewport. */
export const isMobileSidebarVisibleAtom = atom(
  (get) => get(isSidebarOpenAtom) && window.innerWidth < 768,
);
```

```typescript
// features/ui/components/sidebar-toggle/sidebar-toggle.component.tsx

import { useAtom } from 'jotai';
import { isSidebarOpenAtom } from '../../atoms/sidebar.atoms';

/** Button that toggles the global sidebar open/closed. */
export function SidebarToggle(): React.ReactElement {
  const [isSidebarOpen, setIsSidebarOpen] = useAtom(isSidebarOpenAtom);

  return (
    <button
      type="button"
      aria-expanded={isSidebarOpen}
      aria-controls="sidebar-nav"
      onClick={() => setIsSidebarOpen((prevState) => !prevState)}
    >
      {isSidebarOpen ? 'Close menu' : 'Open menu'}
    </button>
  );
}
```

---

## Tier 3 — TanStack Query

Best for: all server state — API data, loading/error states, background refetching, optimistic updates.

**Setup (app/providers.tsx):**

```typescript
// app/providers.tsx

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools }               from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime:          5 * 60 * 1000,  // 5 minutes
      retry:              2,
      refetchOnWindowFocus: false,
    },
  },
});

export function Providers({ children }: { children: React.ReactNode }): React.ReactElement {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && <ReactQueryDevtools />}
    </QueryClientProvider>
  );
}
```

**Query hook:**

```typescript
// features/orders/hooks/use-orders.hook.ts

import { useQuery, keepPreviousData } from '@tanstack/react-query';
import { ordersApiService }           from '../services/orders-api.service';
import { ORDERS_QUERY_KEYS }          from '../constants/orders-query-keys.constants';
import type { IOrderFilters }         from '../../../models/interfaces/i-order.interface';

/**
 * Fetches all orders, optionally filtered. Keeps previous data visible while
 * a new filtered result loads (pagination-friendly).
 */
export function useOrders(filters?: IOrderFilters) {
  return useQuery({
    queryKey:    ORDERS_QUERY_KEYS.filtered(filters ?? {}),
    queryFn:     () => ordersApiService.getOrders(filters),
    placeholderData: keepPreviousData,
  });
}

/**
 * Fetches a single order by ID.
 * Only runs when orderId is truthy.
 */
export function useOrderById(orderId: string | null) {
  return useQuery({
    queryKey: ORDERS_QUERY_KEYS.byId(orderId ?? ''),
    queryFn:  () => ordersApiService.getOrderById(orderId!),
    enabled:  !!orderId,
  });
}
```

**Mutation hook with optimistic update:**

```typescript
// features/orders/hooks/use-update-order-status.hook.ts

import { useMutation, useQueryClient } from '@tanstack/react-query';
import { ordersApiService }            from '../services/orders-api.service';
import { ORDERS_QUERY_KEYS }           from '../constants/orders-query-keys.constants';
import type { IOrder, IUpdateOrderStatusDto } from '../../../models/interfaces/i-order.interface';

/**
 * Mutation hook for updating an order's status.
 * Applies an optimistic update immediately and reverts on failure.
 */
export function useUpdateOrderStatus() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ orderId, dto }: { orderId: string; dto: IUpdateOrderStatusDto }) =>
      ordersApiService.updateOrderStatus(orderId, dto),

    onMutate: async ({ orderId, dto }) => {
      // Cancel outgoing refetches so they don't overwrite our optimistic update
      await queryClient.cancelQueries({ queryKey: ORDERS_QUERY_KEYS.byId(orderId) });

      const previousOrder = queryClient.getQueryData<IOrder>(ORDERS_QUERY_KEYS.byId(orderId));

      // Optimistically update the cache
      queryClient.setQueryData<IOrder>(ORDERS_QUERY_KEYS.byId(orderId), (cachedOrder) =>
        cachedOrder ? { ...cachedOrder, status: dto.status } : cachedOrder,
      );

      return { previousOrder };
    },

    onError: (_error, { orderId }, context) => {
      // Revert the optimistic update if the mutation fails
      if (context?.previousOrder) {
        queryClient.setQueryData(ORDERS_QUERY_KEYS.byId(orderId), context.previousOrder);
      }
    },

    onSettled: (_data, _error, { orderId }) => {
      // Always refetch after success or error to sync with the server
      queryClient.invalidateQueries({ queryKey: ORDERS_QUERY_KEYS.byId(orderId) });
      queryClient.invalidateQueries({ queryKey: ORDERS_QUERY_KEYS.all });
    },
  });
}
```

**Consuming in a component:**

```typescript
// features/orders/components/orders-list/orders-list.component.tsx

import { useOrders } from '../../hooks/use-orders.hook';

/**
 * Renders the orders list with loading, error, and empty states.
 */
export function OrdersList(): React.ReactElement {
  const { data: orders, isLoading, isError, error } = useOrders();

  if (isLoading) {
    return <LoadingSpinner aria-label="Loading orders" />;
  }

  if (isError) {
    return (
      <p className="error-message" role="alert" aria-live="assertive">
        {error instanceof Error ? error.message : 'Failed to load orders.'}
      </p>
    );
  }

  if (!orders || orders.length === 0) {
    return <p className="empty-state">No orders found.</p>;
  }

  return (
    <ul aria-label="Orders list">
      {orders.map((order) => (
        <OrderCard key={order.orderId} order={order} onSelectOrder={handleSelectOrder} />
      ))}
    </ul>
  );
}
```

---

## Tier 4 — Redux Toolkit

Best for: complex app-wide client state, audit-trail requirements, or when RTK Query is already
used and server + client state should live in the same store.

```typescript
// features/orders/store/orders.slice.ts

import { createSlice, createAsyncThunk, type PayloadAction } from '@reduxjs/toolkit';
import { ordersApiService }   from '../services/orders-api.service';
import { OrderMapper }        from '../../../models/mappers/order.mapper';
import type { IOrder }        from '../../../models/interfaces/i-order.interface';

interface OrdersState {
  orders:          IOrder[];
  selectedOrderId: string | null;
  isLoading:       boolean;
  errorMessage:    string | null;
}

const initialOrdersState: OrdersState = {
  orders:          [],
  selectedOrderId: null,
  isLoading:       false,
  errorMessage:    null,
};

/** Async thunk — fetches all orders and maps raw API response to domain types. */
export const fetchOrders = createAsyncThunk<IOrder[], void, { rejectValue: string }>(
  'orders/fetchOrders',
  async (_arg, { rejectWithValue }) => {
    try {
      const rawResponse = await ordersApiService.getRawOrders();
      return OrderMapper.fromApiList(rawResponse);
    } catch (error) {
      return rejectWithValue(
        error instanceof Error ? error.message : 'Failed to fetch orders.',
      );
    }
  },
);

export const ordersSlice = createSlice({
  name:          'orders',
  initialState:  initialOrdersState,
  reducers: {
    /**
     * Sets the currently selected order ID.
     * @param orderId - ID of the order to select
     */
    selectOrder: (state, action: PayloadAction<string>) => {
      state.selectedOrderId = action.payload;
    },
    /** Clears the active order selection. */
    clearSelectedOrder: (state) => {
      state.selectedOrderId = null;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchOrders.pending, (ordersState) => {
        ordersState.isLoading    = true;
        ordersState.errorMessage = null;
      })
      .addCase(fetchOrders.fulfilled, (ordersState, { payload: fetchedOrders }) => {
        ordersState.isLoading = false;
        ordersState.orders    = fetchedOrders;
      })
      .addCase(fetchOrders.rejected, (ordersState, { payload: errorMessage }) => {
        ordersState.isLoading    = false;
        ordersState.errorMessage = errorMessage ?? 'Unknown error';
      });
  },
});

export const { selectOrder, clearSelectedOrder } = ordersSlice.actions;
export default ordersSlice.reducer;
```

**Selectors** — co-locate with slice or in a dedicated `orders.selectors.ts`:

```typescript
// features/orders/store/orders.selectors.ts

import { createSelector }  from '@reduxjs/toolkit';
import type { RootState }  from '../../../app/store';
import { OrderStatus }     from '../../../models/enums/order-status.enum';

export const selectOrders          = (state: RootState) => state.orders.orders;
export const selectSelectedOrderId = (state: RootState) => state.orders.selectedOrderId;
export const selectIsLoadingOrders = (state: RootState) => state.orders.isLoading;
export const selectOrdersError     = (state: RootState) => state.orders.errorMessage;

/** Memoised: the full IOrder object for the selected ID. */
export const selectSelectedOrder = createSelector(
  selectOrders,
  selectSelectedOrderId,
  (orders, selectedId) => orders.find((order) => order.orderId === selectedId) ?? null,
);

/** Memoised: orders in Pending status only. */
export const selectPendingOrders = createSelector(
  selectOrders,
  (orders) => orders.filter((order) => order.status === OrderStatus.Pending),
);
```

**Custom hook wrapping dispatch + selector:**

```typescript
// features/orders/hooks/use-orders-store.hook.ts

import { useDispatch, useSelector } from 'react-redux';
import { fetchOrders, selectOrder, clearSelectedOrder } from '../store/orders.slice';
import {
  selectOrders,
  selectIsLoadingOrders,
  selectOrdersError,
  selectSelectedOrder,
} from '../store/orders.selectors';
import type { AppDispatch } from '../../../app/store';

/**
 * Facade hook for the Orders RTK slice.
 * Components never import useDispatch/useSelector directly — always go through this hook.
 */
export function useOrdersStore() {
  const dispatch = useDispatch<AppDispatch>();

  return {
    orders:         useSelector(selectOrders),
    selectedOrder:  useSelector(selectSelectedOrder),
    isLoading:      useSelector(selectIsLoadingOrders),
    errorMessage:   useSelector(selectOrdersError),

    loadOrders:          () => dispatch(fetchOrders()),
    selectOrderById:     (orderId: string) => dispatch(selectOrder(orderId)),
    clearOrderSelection: () => dispatch(clearSelectedOrder()),
  };
}
```

---

## Tier 5 — React Context + useReducer

Best for: values that change rarely — auth user, theme, locale. Not a general state bus.

```typescript
// core/auth/auth.context.tsx

import { createContext, useContext, useReducer, type ReactNode } from 'react';
import type { IUser } from '../../models/interfaces/i-user.interface';

interface AuthState {
  currentUser:     IUser | null;
  isAuthenticated: boolean;
}

type AuthAction =
  | { type: 'SET_USER'; user: IUser }
  | { type: 'CLEAR_USER' };

function authReducer(state: AuthState, action: AuthAction): AuthState {
  switch (action.type) {
    case 'SET_USER':
      return { currentUser: action.user, isAuthenticated: true };
    case 'CLEAR_USER':
      return { currentUser: null, isAuthenticated: false };
    default:
      return state;
  }
}

interface AuthContextValue extends AuthState {
  setCurrentUser:   (user: IUser) => void;
  clearCurrentUser: () => void;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

/**
 * Provides authenticated user state to the component tree.
 * Wrap the app root with this provider (see app/providers.tsx).
 */
export function AuthProvider({ children }: { children: ReactNode }): React.ReactElement {
  const [authState, dispatch] = useReducer(authReducer, {
    currentUser:     null,
    isAuthenticated: false,
  });

  const contextValue: AuthContextValue = {
    ...authState,
    setCurrentUser:   (user) => dispatch({ type: 'SET_USER', user }),
    clearCurrentUser: ()     => dispatch({ type: 'CLEAR_USER' }),
  };

  return (
    <AuthContext.Provider value={contextValue}>
      {children}
    </AuthContext.Provider>
  );
}

/**
 * Hook for consuming auth state. Throws if used outside AuthProvider.
 */
export function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

---

## Mixing Tiers

A real application mixes tiers by scope. The typical combination:

| Need | Solution |
|---|---|
| Server data (orders, users) | TanStack Query hooks |
| Cart / wizard local feature state | Zustand store per feature |
| Auth user context | Tier 5 Context + useReducer |
| Complex client-only app state | RTK slice |
| Single-component toggle / counter | useState |

**Never mix:** Do not put the same data in both TanStack Query and RTK/Zustand.
TanStack Query's cache is the single source of truth for server data.
