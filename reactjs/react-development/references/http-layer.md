# HTTP Layer — Full Reference

Complete implementation of the React HTTP infrastructure.
For the architectural overview and quick rules see the **HTTP Layer** section in `SKILL.md`.

---

## Table of Contents
1. [HttpContext Tokens (Skip Flags)](#httpcontext-tokens-skip-flags)
2. [IHttpOptions Interface](#ihttpoptions-interface)
3. [ApiClient — Configured Axios Instance](#apiclient--configured-axios-instance)
4. [LoadingService](#loadingservice)
5. [Interceptors](#interceptors)
6. [App Registration](#app-registration)
7. [Feature API Service Example](#feature-api-service-example)
8. [TanStack Query Integration](#tanstack-query-integration)

---

## HttpContext Tokens (Skip Flags)

Skip flags travel on the Axios request config via a custom `metadata` field —
the pattern mirrors Angular's `HttpContextToken` without requiring a framework primitive.

```typescript
// core/http/http-context.tokens.ts

/**
 * Metadata keys attached to individual Axios requests to control interceptor behaviour.
 *
 * Usage:
 *   apiClient.get('/public/data', { metadata: { skipAuth: true } })
 *   apiClient.post('/silent/update', { skipLoading: true })
 */
export interface IRequestMetadata {
  /**
   * When true, the auth interceptor does not inject the Authorization header.
   * Use for public endpoints that do not require authentication.
   */
  skipAuth?: boolean;

  /**
   * When true, the loading interceptor does not increment or decrement
   * the global loading counter for this request.
   * Use for silent background requests (e.g. polling, prefetch).
   */
  skipLoading?: boolean;
}
```

---

## IHttpOptions Interface

```typescript
// core/http/http-options.interface.ts

import type { AxiosRequestConfig } from 'axios';
import type { IRequestMetadata }   from './http-context.tokens';

/**
 * Extended Axios request config that carries per-request interceptor overrides.
 * Pass to any ApiClient method to control auth token injection and loading state.
 *
 * @example
 * apiClient.get('/public/health', { skipAuth: true, skipLoading: true });
 * apiClient.post('/orders', payload, { skipLoading: false });
 */
export interface IHttpOptions extends AxiosRequestConfig {
  metadata?: IRequestMetadata;
}

// Augment AxiosRequestConfig so TypeScript accepts `metadata` on the config object
declare module 'axios' {
  interface InternalAxiosRequestConfig {
    metadata?: IRequestMetadata;
  }
}
```

---

## ApiClient — Configured Axios Instance

```typescript
// core/http/api-client.ts

import axios, { type AxiosInstance } from 'axios';
import { authInterceptor }           from './interceptors/auth.interceptor';
import { loadingInterceptor }        from './interceptors/loading.interceptor';
import { errorInterceptor }          from './interceptors/error.interceptor';
import { loggingInterceptor }        from './interceptors/logging.interceptor';

/**
 * Singleton Axios instance shared across all feature API services.
 *
 * Rules:
 * - This is the ONLY place that creates an Axios instance.
 * - Feature API services import and call this instance — they never create their own.
 * - Interceptors are applied once, in a defined order: auth → loading → error → logging.
 */
function createApiClient(): AxiosInstance {
  const instance = axios.create({
    baseURL: import.meta.env.VITE_API_BASE_URL ?? '/api',
    timeout: 30_000,
    headers: {
      'Content-Type': 'application/json',
      'Accept':       'application/json',
    },
  });

  // Register interceptors in order — request interceptors run last-registered first
  authInterceptor(instance);
  loadingInterceptor(instance);
  errorInterceptor(instance);
  loggingInterceptor(instance);

  return instance;
}

/** The application's shared HTTP client. Import this in feature API services. */
export const apiClient = createApiClient();
```

---

## LoadingService

```typescript
// core/http/loading.service.ts

import { atom, getDefaultStore } from 'jotai';

/**
 * Tracks how many HTTP requests are currently in flight using a counter
 * rather than a boolean flag, so concurrent requests do not cancel each other's state.
 *
 * Uses a Jotai atom so any component can subscribe to the loading state without
 * prop-drilling or a separate context provider.
 *
 * Alternative: use a Zustand store if the project does not use Jotai.
 */

/** Internal counter — never export this; use the derived atom instead. */
const activeRequestCountAtom = atom<number>(0);

/** Derived read-only atom — true when at least one request is in flight. */
export const isGlobalLoadingAtom = atom<boolean>(
  (get) => get(activeRequestCountAtom) > 0,
);

const jotaiStore = getDefaultStore();

/**
 * Increments the active request counter.
 * Called by the loading interceptor on request start.
 */
export function incrementLoadingCount(): void {
  jotaiStore.set(activeRequestCountAtom, (prev) => prev + 1);
}

/**
 * Decrements the active request counter, floored at zero to prevent underflow.
 * Called by the loading interceptor on request completion (success or error).
 */
export function decrementLoadingCount(): void {
  jotaiStore.set(activeRequestCountAtom, (prev) => Math.max(0, prev - 1));
}

/**
 * Resets the counter to zero.
 * Use on logout or navigation to clear any stuck loading state.
 */
export function resetLoadingCount(): void {
  jotaiStore.set(activeRequestCountAtom, 0);
}
```

**Consuming the loading state in a component:**

```typescript
// shared/components/global-loading-bar/global-loading-bar.component.tsx

import { useAtomValue } from 'jotai';
import { isGlobalLoadingAtom } from '../../../core/http/loading.service';

/**
 * Thin progress bar displayed at the top of the viewport whenever any
 * HTTP request is in flight. Reads the shared loading atom.
 */
export function GlobalLoadingBar(): React.ReactElement | null {
  const isGlobalLoading = useAtomValue(isGlobalLoadingAtom);

  if (!isGlobalLoading) return null;

  return (
    <div
      className="global-loading-bar"
      role="progressbar"
      aria-label="Loading"
      aria-busy={true}
    />
  );
}
```

---

## Interceptors

### auth.interceptor.ts

```typescript
// core/http/interceptors/auth.interceptor.ts

import type { AxiosInstance } from 'axios';
import { STORAGE_KEYS }       from '../../models/constants/app.constants';

/**
 * Attaches the stored Bearer token to every outgoing request.
 * Skipped when the request config has metadata.skipAuth = true.
 */
export function authInterceptor(axiosInstance: AxiosInstance): void {
  axiosInstance.interceptors.request.use((requestConfig) => {
    const shouldSkipAuth = requestConfig.metadata?.skipAuth === true;

    if (!shouldSkipAuth) {
      const authToken = localStorage.getItem(STORAGE_KEYS.AUTH_TOKEN);
      if (authToken) {
        requestConfig.headers.Authorization = `Bearer ${authToken}`;
      }
    }

    return requestConfig;
  });
}
```

### loading.interceptor.ts

```typescript
// core/http/interceptors/loading.interceptor.ts

import type { AxiosInstance } from 'axios';
import {
  incrementLoadingCount,
  decrementLoadingCount,
} from '../loading.service';

/**
 * Increments the global loading counter on request start and decrements on completion.
 * Skipped when the request config has metadata.skipLoading = true.
 * Uses a counter (not a boolean) to correctly handle concurrent requests.
 */
export function loadingInterceptor(axiosInstance: AxiosInstance): void {
  axiosInstance.interceptors.request.use((requestConfig) => {
    if (!requestConfig.metadata?.skipLoading) {
      incrementLoadingCount();
    }
    return requestConfig;
  });

  axiosInstance.interceptors.response.use(
    (response) => {
      if (!response.config.metadata?.skipLoading) {
        decrementLoadingCount();
      }
      return response;
    },
    (responseError) => {
      if (!responseError.config?.metadata?.skipLoading) {
        decrementLoadingCount();
      }
      return Promise.reject(responseError);
    },
  );
}
```

### error.interceptor.ts

```typescript
// core/http/interceptors/error.interceptor.ts

import type { AxiosInstance, AxiosError } from 'axios';
import { HttpErrorCode }                  from '../../models/enums/http-error-code.enum';
import { APP_ROUTE_PATHS, STORAGE_KEYS }  from '../../models/constants/app.constants';

/**
 * Centralised HTTP error handling:
 * - 401 Unauthorized: clears auth token and redirects to /login
 * - 403 Forbidden: redirects to /forbidden
 * - 5xx Server errors: logs to console (extend to error monitoring as needed)
 */
export function errorInterceptor(axiosInstance: AxiosInstance): void {
  axiosInstance.interceptors.response.use(
    (successResponse) => successResponse,
    (responseError: AxiosError) => {
      const statusCode = responseError.response?.status;

      if (statusCode === HttpErrorCode.Unauthorized) {
        localStorage.removeItem(STORAGE_KEYS.AUTH_TOKEN);
        window.location.href = APP_ROUTE_PATHS.LOGIN;
      }

      if (statusCode === HttpErrorCode.Forbidden) {
        window.location.href = APP_ROUTE_PATHS.FORBIDDEN;
      }

      if (statusCode !== undefined && statusCode >= 500) {
        console.error('[HTTP] Server error:', responseError.response?.data);
        // Extend: send to error monitoring service (Sentry, Datadog)
      }

      return Promise.reject(responseError);
    },
  );
}
```

### logging.interceptor.ts

```typescript
// core/http/interceptors/logging.interceptor.ts

import type { AxiosInstance } from 'axios';

/**
 * Logs all outgoing requests and incoming responses/errors in development mode.
 * Disabled in production via the NODE_ENV check.
 */
export function loggingInterceptor(axiosInstance: AxiosInstance): void {
  if (process.env.NODE_ENV !== 'development') return;

  axiosInstance.interceptors.request.use((requestConfig) => {
    console.log(
      `[HTTP] ▶ ${requestConfig.method?.toUpperCase()} ${requestConfig.url}`,
      requestConfig.params ?? '',
    );
    return requestConfig;
  });

  axiosInstance.interceptors.response.use(
    (response) => {
      console.log(
        `[HTTP] ✔ ${response.status} ${response.config.method?.toUpperCase()} ${response.config.url}`,
      );
      return response;
    },
    (responseError) => {
      console.error(
        `[HTTP] ✖ ${responseError.response?.status ?? 'ERR'} ${responseError.config?.method?.toUpperCase()} ${responseError.config?.url}`,
        responseError.response?.data ?? responseError.message,
      );
      return Promise.reject(responseError);
    },
  );
}
```

---

## App Registration

```typescript
// main.tsx  (or app/providers.tsx — wherever the provider tree is assembled)

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Provider as JotaiProvider }        from 'jotai';
// apiClient is initialised on import — interceptors registered automatically
import './core/http/api-client';

const queryClient = new QueryClient();

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <JotaiProvider>
      <QueryClientProvider client={queryClient}>
        <AuthProvider>
          <App />
        </AuthProvider>
      </QueryClientProvider>
    </JotaiProvider>
  </StrictMode>,
);
```

---

## Feature API Service Example

```typescript
// features/orders/services/orders-api.service.ts

import { apiClient }   from '../../../core/http/api-client';
import { OrderMapper } from '../../../models/mappers/order.mapper';
import { ORDERS_API }  from '../../../models/constants/api.constants';
import type {
  IOrder,
  ICreateOrderDto,
  IUpdateOrderStatusDto,
  IOrderFilters,
} from '../../../models/interfaces/i-order.interface';
import type { IPagedResponse } from '../../../models/interfaces/i-paged-response.interface';

/**
 * All HTTP calls for the Orders feature.
 * Imports apiClient only — never axios directly.
 * Returns mapped domain types, not raw API shapes.
 */
export const ordersApiService = {
  /**
   * Fetches a paginated list of orders.
   * @param filters - Optional filter criteria
   * @param page    - 1-based page number (default: 1)
   * @param size    - Items per page (default: 25)
   */
  async getOrders(
    filters?: IOrderFilters,
    page = 1,
    size = 25,
  ): Promise<IOrder[]> {
    const response = await apiClient.get<Record<string, unknown>[]>(ORDERS_API.BASE, {
      params: { page, size, ...filters },
    });
    return OrderMapper.fromApiList(response.data);
  },

  /**
   * Fetches a single order by its unique identifier.
   * @param orderId - The unique order ID
   */
  async getOrderById(orderId: string): Promise<IOrder> {
    const response = await apiClient.get<Record<string, unknown>>(
      ORDERS_API.BY_ID(orderId),
    );
    return OrderMapper.fromApi(response.data);
  },

  /**
   * Creates a new order.
   * @param orderData - Validated order payload
   */
  async createOrder(orderData: ICreateOrderDto): Promise<IOrder> {
    const response = await apiClient.post<Record<string, unknown>>(
      ORDERS_API.BASE,
      orderData,
    );
    return OrderMapper.fromApi(response.data);
  },

  /**
   * Updates the status of an existing order.
   * @param orderId - ID of the order to update
   * @param dto     - Status update payload
   */
  async updateOrderStatus(orderId: string, dto: IUpdateOrderStatusDto): Promise<IOrder> {
    const response = await apiClient.patch<Record<string, unknown>>(
      ORDERS_API.STATUS(orderId),
      dto,
    );
    return OrderMapper.fromApi(response.data);
  },

  /**
   * Fully replaces an order record.
   * @param orderId   - ID of the order to replace
   * @param orderData - Complete order payload
   */
  async replaceOrder(orderId: string, orderData: ICreateOrderDto): Promise<IOrder> {
    const response = await apiClient.put<Record<string, unknown>>(
      ORDERS_API.BY_ID(orderId),
      orderData,
    );
    return OrderMapper.fromApi(response.data);
  },

  /**
   * Deletes an order. Returns void on 204 No Content.
   * @param orderId - ID of the order to delete
   */
  async deleteOrder(orderId: string): Promise<void> {
    await apiClient.delete(ORDERS_API.BY_ID(orderId));
  },

  /**
   * Checks whether an order exists without downloading the full body.
   * @param orderId - ID to verify
   */
  async orderExists(orderId: string): Promise<boolean> {
    try {
      await apiClient.head(ORDERS_API.BY_ID(orderId));
      return true;
    } catch {
      return false;
    }
  },

  /**
   * Fetches allowed HTTP methods for an order resource.
   * @param orderId - ID of the order to inspect
   */
  async getOrderOptions(orderId: string): Promise<string[]> {
    const response = await apiClient.options(ORDERS_API.BY_ID(orderId));
    const allowHeader = response.headers['allow'] as string | undefined;
    return allowHeader ? allowHeader.split(',').map((method) => method.trim()) : [];
  },
} as const;
```

---

## TanStack Query Integration

When TanStack Query is used (Tier 3), feature API service functions become the `queryFn`
and `mutationFn` values. The `apiClient` interceptors still run for every request.

```typescript
// features/orders/hooks/use-orders.hook.ts

import { useQuery }         from '@tanstack/react-query';
import { ordersApiService } from '../services/orders-api.service';
import { ORDERS_QUERY_KEYS } from '../constants/orders-query-keys.constants';

/** Wraps ordersApiService.getOrders() with TanStack Query caching. */
export function useOrders() {
  return useQuery({
    queryKey: ORDERS_QUERY_KEYS.all,
    queryFn:  () => ordersApiService.getOrders(),
  });
}
```

The loading state from TanStack Query (`isLoading`, `isFetching`) covers query-initiated
requests. The `isGlobalLoadingAtom` from `LoadingService` covers all requests including
those made outside of TanStack Query (e.g. direct `apiClient` calls in event handlers).
