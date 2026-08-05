# HTTP Layer — Full Reference

Complete implementation of the HTTP infrastructure.
For the architectural overview and quick rules see the **HTTP Layer** section in `SKILL.md`.

---

## Table of Contents
1. [HttpContext Tokens](#httpcontext-tokens)
2. [IHttpOptions Interface](#ihttpoptions-interface)
3. [BaseHttpService](#basehttpservice)
4. [LoadingService](#loadingservice)
5. [Interceptors](#interceptors)
6. [app.config.ts Registration](#appconfigts-registration)
7. [Feature API Service Example](#feature-api-service-example)

---

## HttpContext Tokens

```typescript
// core/tokens/http-context.tokens.ts
import { HttpContextToken } from '@angular/common/http';

/**
 * Set to true on requests that should NOT trigger the global loading indicator.
 * Default: false — loading is shown for every request unless explicitly skipped.
 *
 * Usage: pass { skipLoading: true } to any BaseHttpService method.
 */
export const SKIP_LOADING = new HttpContextToken<boolean>(() => false);

/**
 * Set to true on requests that must NOT carry the Authorization header.
 * Use for public endpoints: login, token refresh, public asset downloads.
 * Default: false — auth header is attached to every request.
 *
 * Usage: pass { skipAuth: true } to any BaseHttpService method.
 */
export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);
```

---

## IHttpOptions Interface

```typescript
// core/http/interfaces/i-http-options.interface.ts
import { HttpContext, HttpHeaders, HttpParams } from '@angular/common/http';

/**
 * Extended HTTP options accepted by all BaseHttpService verb methods.
 * Merges standard Angular HttpClient options with project-specific control flags.
 */
export interface IHttpOptions {
  /** Additional headers merged with those provided by the interceptor chain. */
  headers?: HttpHeaders | Record<string, string | string[]>;

  /** Query parameters appended to the request URL. */
  params?: HttpParams | Record<string, string | number | boolean | ReadonlyArray<string | number | boolean>>;

  /**
   * When true, suppresses the global loading indicator for this request.
   * Use for background polling, silent token refresh, or low-priority reads.
   * Translates to the SKIP_LOADING HttpContextToken internally.
   */
  skipLoading?: boolean;

  /**
   * When true, the auth interceptor will not attach the Authorization header.
   * Use for unauthenticated endpoints such as login or public APIs.
   * Translates to the SKIP_AUTH HttpContextToken internally.
   */
  skipAuth?: boolean;

  /**
   * Provide a pre-built HttpContext to merge with. When provided, skipLoading
   * and skipAuth flags are set on this context rather than creating a new one.
   */
  context?: HttpContext;

  /** Expected response type. Defaults to 'json'. */
  responseType?: 'json' | 'text' | 'blob' | 'arraybuffer';
}
```

---

## BaseHttpService

```typescript
// core/http/base-http.service.ts
import { inject, Injectable }                from '@angular/core';
import { HttpClient, HttpContext, HttpResponse } from '@angular/common/http';
import { Observable }                        from 'rxjs';
import { SKIP_AUTH, SKIP_LOADING }           from '../tokens/http-context.tokens';
import { IHttpOptions }                      from './interfaces/i-http-options.interface';

/**
 * Central HTTP gateway for the application.
 *
 * Wraps Angular's HttpClient with fully-typed generic verb methods and translates
 * IHttpOptions flags (skipLoading, skipAuth) into HttpContext tokens that the
 * interceptor chain reads.
 *
 * RULES — enforced by code review:
 * 1. This is the ONLY class permitted to inject HttpClient.
 * 2. Feature API services must inject BaseHttpService, not HttpClient.
 * 3. Never call BaseHttpService from a component, facade, or NgRx effect directly
 *    — always go through a feature API service (e.g. OrdersApiService).
 */
@Injectable({ providedIn: 'root' })
export class BaseHttpService {
  private httpClient = inject(HttpClient);

  /**
   * Performs an HTTP GET request.
   * Use for read-only data retrieval; must not trigger server-side side effects.
   *
   * @param url     - Absolute URL or path (combined with API_BASE_URL in feature services)
   * @param options - Optional headers, params, and control flags
   * @returns Observable emitting the typed response body
   */
  get<TResponse>(url: string, options?: IHttpOptions): Observable<TResponse> {
    return this.httpClient.get<TResponse>(url, this.buildOptions(options));
  }

  /**
   * Performs an HTTP POST request.
   * Use to create a new resource or submit data that causes a server-side change.
   *
   * @param url     - Target endpoint URL
   * @param body    - Request payload; serialised to JSON automatically
   * @param options - Optional headers, params, and control flags
   * @returns Observable emitting the typed response body
   */
  post<TResponse>(url: string, body: unknown, options?: IHttpOptions): Observable<TResponse> {
    return this.httpClient.post<TResponse>(url, body, this.buildOptions(options));
  }

  /**
   * Performs an HTTP PUT request.
   * Use for a full replacement of an existing resource.
   * The body must represent the complete resource — omitted fields are cleared server-side.
   * Prefer patch() when you only have partial data.
   *
   * @param url     - Target resource URL
   * @param body    - Complete resource payload
   * @param options - Optional headers, params, and control flags
   * @returns Observable emitting the typed response body
   */
  put<TResponse>(url: string, body: unknown, options?: IHttpOptions): Observable<TResponse> {
    return this.httpClient.put<TResponse>(url, body, this.buildOptions(options));
  }

  /**
   * Performs an HTTP PATCH request.
   * Use for partial updates — send only the fields that changed.
   * Prefer this over put() when the client does not hold the full resource state.
   *
   * @param url     - Target resource URL
   * @param body    - Partial resource payload (only changed fields)
   * @param options - Optional headers, params, and control flags
   * @returns Observable emitting the typed response body
   */
  patch<TResponse>(url: string, body: Partial<unknown>, options?: IHttpOptions): Observable<TResponse> {
    return this.httpClient.patch<TResponse>(url, body, this.buildOptions(options));
  }

  /**
   * Performs an HTTP DELETE request.
   * Use to remove a resource. Expect HTTP 204 No Content on success.
   *
   * @param url     - URL of the resource to delete
   * @param options - Optional headers, params, and control flags
   * @returns Observable that completes when deletion is confirmed
   */
  delete<TResponse>(url: string, options?: IHttpOptions): Observable<TResponse> {
    return this.httpClient.delete<TResponse>(url, this.buildOptions(options));
  }

  /**
   * Performs an HTTP HEAD request.
   * Use to check resource existence or retrieve headers without downloading the body.
   * Useful for checking Last-Modified, ETag values, or confirming a URL before navigation.
   *
   * @param url     - Target resource URL
   * @param options - Optional headers, params, and control flags
   * @returns Observable emitting the full HTTP response (body is always empty for HEAD)
   */
  head(url: string, options?: IHttpOptions): Observable<HttpResponse<unknown>> {
    return this.httpClient.head(url, {
      ...this.buildOptions(options),
      observe: 'response',
    });
  }

  /**
   * Performs an HTTP OPTIONS request.
   * Use for explicit capability discovery (CORS preflight is handled automatically;
   * this method is for programmatic interrogation of supported methods).
   *
   * @param url     - Target resource URL
   * @param options - Optional headers, params, and control flags
   * @returns Observable emitting the server's capability response
   */
  options<TResponse>(url: string, options?: IHttpOptions): Observable<TResponse> {
    return this.httpClient.options<TResponse>(url, this.buildOptions(options));
  }

  /**
   * Translates IHttpOptions into the format Angular's HttpClient expects.
   * Sets SKIP_LOADING and SKIP_AUTH context tokens when the flags are provided.
   *
   * @param options - Caller-supplied options; undefined is safe
   * @returns Options object compatible with all HttpClient verb methods
   */
  private buildOptions(options?: IHttpOptions): Record<string, unknown> {
    const httpContext = options?.context ?? new HttpContext();

    if (options?.skipLoading) {
      httpContext.set(SKIP_LOADING, true);
    }
    if (options?.skipAuth) {
      httpContext.set(SKIP_AUTH, true);
    }

    return {
      headers:      options?.headers,
      params:       options?.params,
      responseType: options?.responseType ?? 'json',
      context:      httpContext,
    };
  }
}
```

---

## LoadingService

```typescript
// core/services/loading.service.ts
import { computed, Injectable, signal } from '@angular/core';

/**
 * Tracks the number of HTTP requests currently in flight using a counter.
 *
 * Uses a counter (not a boolean) so concurrent requests do not prematurely
 * hide the loading indicator — it stays visible as long as any request is pending.
 *
 * Consumers:
 * - LoadingInterceptor: calls increment() on request departure, decrement() on settle.
 * - Layout components: bind to isLoading to show/hide the global progress bar or spinner.
 */
@Injectable({ providedIn: 'root' })
export class LoadingService {
  /** Number of HTTP requests currently in flight. Never exposed directly. */
  private activeRequestCount = signal<number>(0);

  /**
   * True while one or more HTTP requests are in flight.
   * Bind this in the root layout to control the global loading indicator.
   *
   * @example
   * // In the root shell component template:
   * @if (loadingService.isLoading()) { <app-progress-bar /> }
   */
  isLoading = computed(() => this.activeRequestCount() > 0);

  /**
   * Increments the active request counter by one.
   * Called by LoadingInterceptor immediately before a request is sent.
   */
  increment(): void {
    this.activeRequestCount.update(count => count + 1);
  }

  /**
   * Decrements the active request counter by one, clamped to zero.
   * Called by LoadingInterceptor in finalize() after a request completes or errors.
   */
  decrement(): void {
    this.activeRequestCount.update(count => Math.max(0, count - 1));
  }

  /**
   * Resets the counter to zero immediately.
   * Use only in error-recovery or navigation scenarios where all in-flight
   * requests are known to be cancelled. Prefer increment/decrement in normal flow.
   */
  reset(): void {
    this.activeRequestCount.set(0);
  }
}
```

---

## Interceptors

### Auth Interceptor

```typescript
// core/interceptors/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject }            from '@angular/core';
import { AuthService }       from '../services/auth.service';
import { SKIP_AUTH }         from '../tokens/http-context.tokens';

/**
 * Attaches `Authorization: Bearer <token>` to every outgoing HTTP request.
 *
 * Skips attachment when:
 * - The SKIP_AUTH context token is true (public endpoints — login, refresh, etc.)
 * - The request already carries an Authorization header (caller has set it manually)
 * - No access token is available (unauthenticated state — let the request proceed and
 *   let the error interceptor handle the 401 response)
 */
export const authInterceptor: HttpInterceptorFn = (request, next) => {
  const authService = inject(AuthService);

  const shouldSkipAuth =
    request.context.get(SKIP_AUTH) ||
    request.headers.has('Authorization');

  if (shouldSkipAuth) {
    return next(request);
  }

  const accessToken = authService.getAccessToken();

  if (!accessToken) {
    return next(request);
  }

  const authenticatedRequest = request.clone({
    setHeaders: { Authorization: `Bearer ${accessToken}` },
  });

  return next(authenticatedRequest);
};
```

### Loading Interceptor

```typescript
// core/interceptors/loading.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject }            from '@angular/core';
import { finalize }          from 'rxjs';
import { LoadingService }    from '../services/loading.service';
import { SKIP_LOADING }      from '../tokens/http-context.tokens';

/**
 * Shows and hides the global loading indicator by maintaining a request counter
 * in LoadingService. The spinner remains visible while any request is still in flight.
 *
 * Per-request opt-out:
 * - Set { skipLoading: true } in IHttpOptions on the BaseHttpService method call, OR
 * - Set the SKIP_LOADING HttpContextToken to true on the request directly.
 *
 * The finalize() operator ensures decrement() is called whether the request
 * succeeds, errors, or is cancelled — preventing the counter from leaking.
 */
export const loadingInterceptor: HttpInterceptorFn = (request, next) => {
  const loadingService      = inject(LoadingService);
  const shouldSkipLoading   = request.context.get(SKIP_LOADING);

  if (shouldSkipLoading) {
    return next(request);
  }

  loadingService.increment();

  return next(request).pipe(
    finalize(() => loadingService.decrement()),
  );
};
```

### Error Interceptor

```typescript
// core/interceptors/error.interceptor.ts
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { inject }                               from '@angular/core';
import { Router }                               from '@angular/router';
import { catchError, throwError }               from 'rxjs';
import { AuthService }                          from '../services/auth.service';
import { NotificationService }                  from '../services/notification.service';
import { APP_ROUTE_PATHS }                      from '../../models/constants/app.constants';
import { HttpErrorCode }                        from '../../models/enums/http-error-code.enum';

/**
 * Intercepts HTTP error responses and applies a consistent global handling strategy.
 *
 * Strategy per status code:
 * - 401 Unauthorized  → clears the session and redirects to /login with returnUrl
 * - 403 Forbidden     → navigates to /forbidden
 * - 404 Not Found     → re-throws (caller decides — may be a valid empty state)
 * - 422 Unprocessable → re-throws (form layer handles field-level validation errors)
 * - 5xx Server Error  → shows a generic user-facing error toast; re-throws
 * - 0 (network error) → shows a connectivity error toast; re-throws
 *
 * All errors are re-thrown after global handling so feature services and components
 * can add context-specific responses (e.g. showing an inline error message).
 */
export const errorInterceptor: HttpInterceptorFn = (request, next) => {
  const authService          = inject(AuthService);
  const router               = inject(Router);
  const notificationService  = inject(NotificationService);

  return next(request).pipe(
    catchError((httpError: HttpErrorResponse) => {
      switch (httpError.status) {
        case HttpErrorCode.Unauthorized:
          authService.clearSession();
          router.navigate([APP_ROUTE_PATHS.LOGIN], {
            queryParams: { returnUrl: router.url },
          });
          break;

        case HttpErrorCode.Forbidden:
          router.navigate([APP_ROUTE_PATHS.FORBIDDEN]);
          break;

        case 0:
          // Status 0: request never reached the server (offline, DNS failure, CORS block)
          notificationService.showError(
            'Unable to reach the server. Please check your internet connection.',
          );
          break;

        default:
          if (httpError.status >= HttpErrorCode.InternalServerError) {
            notificationService.showError(
              'A server error occurred. Please try again or contact support if the issue persists.',
            );
          }
          break;
      }

      // Always re-throw so feature-level error handling can respond if needed
      return throwError(() => httpError);
    }),
  );
};
```

### Logging Interceptor

```typescript
// core/interceptors/logging.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { tap }                             from 'rxjs';

/**
 * Logs every outgoing HTTP request and its corresponding response or error,
 * including the elapsed time in milliseconds.
 *
 * Logging is the last interceptor in the chain so it captures the final state
 * of the request after auth, loading, and error interceptors have processed it.
 *
 * Production note: replace console.debug / console.error with an injected
 * LoggerService that routes to your observability platform (Application Insights,
 * Datadog, etc.) so logs are not visible in the browser console in production.
 */
export const loggingInterceptor: HttpInterceptorFn = (request, next) => {
  const requestStartTimeMs = Date.now();

  console.debug(`[HTTP] → ${request.method} ${request.url}`);

  return next(request).pipe(
    tap({
      next: (event) => {
        if (event instanceof HttpResponse) {
          const elapsedMs = Date.now() - requestStartTimeMs;
          console.debug(
            `[HTTP] ← ${request.method} ${request.url} | ${event.status} | ${elapsedMs}ms`,
          );
        }
      },
      error: (httpError: HttpErrorResponse) => {
        const elapsedMs = Date.now() - requestStartTimeMs;
        console.error(
          `[HTTP] ✕ ${request.method} ${request.url} | ${httpError.status} | ${elapsedMs}ms`,
          httpError.message,
        );
      },
    }),
  );
};
```

---

## app.config.ts Registration

Register all interceptors in this exact order. The chain runs top-to-bottom on
the way out and bottom-to-top on the way back in.

```typescript
// app.config.ts
import { ApplicationConfig }    from '@angular/core';
import { provideRouter }        from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { APP_ROUTES }           from './app.routes';
import { authInterceptor }      from './core/interceptors/auth.interceptor';
import { loadingInterceptor }   from './core/interceptors/loading.interceptor';
import { errorInterceptor }     from './core/interceptors/error.interceptor';
import { loggingInterceptor }   from './core/interceptors/logging.interceptor';
import { API_BASE_URL }         from './core/tokens/api-base-url.token';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(APP_ROUTES),
    provideHttpClient(
      withInterceptors([
        authInterceptor,     // 1st out — token attached before the request leaves
        loadingInterceptor,  // 2nd out — spinner starts
        errorInterceptor,    // 3rd out — wraps the response stream to catch errors
        loggingInterceptor,  // last out / last in — sees the final state of everything
      ]),
    ),
    {
      provide:  API_BASE_URL,
      useValue: environment.apiBaseUrl,
    },
  ],
};
```

---

## Feature API Service Example

```typescript
// features/orders/services/orders-api.service.ts
import { inject, Injectable }  from '@angular/core';
import { Observable }          from 'rxjs';
import { map }                 from 'rxjs/operators';
import { BaseHttpService }     from '../../../core/http/base-http.service';
import { API_BASE_URL }        from '../../../core/tokens/api-base-url.token';
import {
  IOrder,
  ICreateOrderDto,
  IUpdateOrderStatusDto,
  IPagedResponse,
  IOrderFilters,
}                              from '../../../models/interfaces/i-order.interface';
import { OrderMapper }         from '../../../models/mappers/order.mapper';
import { PAGINATION_DEFAULTS } from '../../../models/constants/app.constants';
import { ORDERS_API }          from '../../../models/constants/api.constants';

/**
 * Handles all HTTP communication with the Orders REST API.
 *
 * - Injects BaseHttpService, never HttpClient directly.
 * - Maps raw API responses to typed domain objects via OrderMapper.
 * - Error handling and loading state are delegated to the interceptor chain.
 */
@Injectable({ providedIn: 'root' })
export class OrdersApiService {
  private baseHttp   = inject(BaseHttpService);
  private apiBaseUrl = inject(API_BASE_URL);

  /**
   * Retrieves a paginated, filterable list of orders.
   *
   * @param filters    - Optional filter criteria (status, date range, customer ID)
   * @param pageNumber - 1-based page index; API internally converts to 0-based
   * @param pageSize   - Records per page
   * @returns Observable of a typed paged response with mapped OrderModel data
   */
  getOrders(
    filters?: IOrderFilters,
    pageNumber: number = PAGINATION_DEFAULTS.DEFAULT_PAGE,
    pageSize:   number = PAGINATION_DEFAULTS.PAGE_SIZE,
  ): Observable<IPagedResponse<IOrder>> {
    return this.baseHttp.get<IPagedResponse<Record<string, unknown>>>(
      `${this.apiBaseUrl}${ORDERS_API.BASE}`,
      {
        params: {
          page: pageNumber - 1,   // API uses 0-based paging; UI uses 1-based
          size: pageSize,
          ...filters,
        },
      },
    ).pipe(
      map(response => ({
        ...response,
        data: OrderMapper.fromApiList(response.data),
      })),
    );
  }

  /**
   * Retrieves a single order by its unique identifier.
   * @param orderId - UUID of the order to fetch
   * @returns Observable emitting the matching typed IOrder
   */
  getOrderById(orderId: string): Observable<IOrder> {
    return this.baseHttp
      .get<Record<string, unknown>>(`${this.apiBaseUrl}${ORDERS_API.BY_ID(orderId)}`)
      .pipe(map(OrderMapper.fromApi));
  }

  /**
   * Checks whether an order exists without downloading its full body.
   * Uses HTTP HEAD — suitable for lightweight pre-navigation existence checks.
   * @param orderId - UUID of the order to verify
   * @returns Observable that completes if found; errors with 404 if not
   */
  orderExists(orderId: string): Observable<unknown> {
    return this.baseHttp.head(`${this.apiBaseUrl}${ORDERS_API.BY_ID(orderId)}`);
  }

  /**
   * Creates a new order.
   * @param orderData - Validated order payload conforming to ICreateOrderDto
   * @returns Observable emitting the newly created, mapped IOrder
   */
  createOrder(orderData: ICreateOrderDto): Observable<IOrder> {
    return this.baseHttp
      .post<Record<string, unknown>>(`${this.apiBaseUrl}${ORDERS_API.BASE}`, orderData)
      .pipe(map(OrderMapper.fromApi));
  }

  /**
   * Partially updates an existing order — only supplied fields are changed.
   * Prefer this over putOrder when the client does not hold the full resource.
   * @param orderId       - UUID of the order to update
   * @param changedFields - Fields to update; omitted fields are unchanged server-side
   * @returns Observable emitting the updated, mapped IOrder
   */
  patchOrder(orderId: string, changedFields: Partial<IOrder>): Observable<IOrder> {
    return this.baseHttp
      .patch<Record<string, unknown>>(
        `${this.apiBaseUrl}${ORDERS_API.BY_ID(orderId)}`,
        changedFields,
      )
      .pipe(map(OrderMapper.fromApi));
  }

  /**
   * Fully replaces an existing order record.
   * All fields must be supplied — omitted fields are cleared on the server.
   * @param orderId   - UUID of the order to replace
   * @param orderData - Complete order payload
   * @returns Observable emitting the replaced, mapped IOrder
   */
  putOrder(orderId: string, orderData: ICreateOrderDto): Observable<IOrder> {
    return this.baseHttp
      .put<Record<string, unknown>>(
        `${this.apiBaseUrl}${ORDERS_API.BY_ID(orderId)}`,
        orderData,
      )
      .pipe(map(OrderMapper.fromApi));
  }

  /**
   * Deletes an order by ID. Expects HTTP 204 No Content on success.
   * @param orderId - UUID of the order to delete
   * @returns Observable that completes when deletion is confirmed
   */
  deleteOrder(orderId: string): Observable<void> {
    return this.baseHttp.delete<void>(
      `${this.apiBaseUrl}${ORDERS_API.BY_ID(orderId)}`,
    );
  }

  /**
   * Polls for the current status of an order without triggering the global spinner.
   * The skipLoading flag suppresses the loading indicator for this background call.
   * @param orderId - UUID of the order to poll
   * @returns Observable emitting only the status field of the order
   */
  pollOrderStatus(orderId: string): Observable<IUpdateOrderStatusDto> {
    return this.baseHttp.get<IUpdateOrderStatusDto>(
      `${this.apiBaseUrl}${ORDERS_API.STATUS(orderId)}`,
      { skipLoading: true },
    );
  }
}
```
