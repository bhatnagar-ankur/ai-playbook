# Examples — Full Reference

Complete annotated examples showing the skill's conventions working together.

---

## Table of Contents
1. [Standalone component with signals, loading/error, @for + @empty](#example-1-standalone-productlistcomponent)
2. [Interface + enum + mapper working together](#example-2-order-type-system)
3. [NgRx Facade hiding the store from components](#example-3-ngrx-facade-in-a-component)
4. [Accessible login form with Reactive Forms and aria-live errors](#example-4-accessible-login-form)

---

## Example 1: Standalone ProductListComponent

Demonstrates: standalone component, signals, OnPush, `@for` + `@empty`, `@defer`,
`inject()`, `takeUntilDestroyed()`, loading/error state.

**Input (what the user asks):**
> "Create a ProductListComponent that fetches products from ProductService, renders them
> in a grid, shows a loading spinner while fetching, an error message on failure, and
> defers the heavy analytics panel below the fold."

**Expected output:**

```typescript
// features/products/components/product-list/product-list.component.ts
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  OnInit,
  inject,
  signal,
}                                   from '@angular/core';
import { takeUntilDestroyed }       from '@angular/core/rxjs-interop';
import { NgOptimizedImage }         from '@angular/common';
import { finalize }                 from 'rxjs';
import { ProductService }           from '../../services/product.service';
import { ProductCardComponent }     from '../product-card/product-card.component';
import { LoadingSpinnerComponent }  from '../../../../shared/components/loading-spinner/loading-spinner.component';
import { AnalyticsPanelComponent }  from '../analytics-panel/analytics-panel.component';
import { IProduct }                 from '../../../../models/interfaces/i-product.interface';

/**
 * Displays the product catalogue as a responsive grid.
 *
 * Responsibilities:
 * - Fetches products on init via ProductService
 * - Manages loading and error states inline via signals
 * - Defers the analytics panel until it enters the viewport
 *
 * Used on: /products (main catalogue), /admin/products (management view)
 */
@Component({
  selector:          'app-product-list',
  standalone:        true,
  imports:           [NgOptimizedImage, ProductCardComponent, LoadingSpinnerComponent, AnalyticsPanelComponent],
  templateUrl:       './product-list.component.html',
  changeDetection:   ChangeDetectionStrategy.OnPush,
})
export class ProductListComponent implements OnInit {
  private productService = inject(ProductService);
  private destroyRef     = inject(DestroyRef);

  /** Products returned by the API. Empty until the initial fetch completes. */
  products = signal<IProduct[]>([]);

  /** True while the products API request is in flight. */
  isLoadingProducts = signal(false);

  /**
   * User-facing error message displayed when the product fetch fails.
   * Null when no error is present.
   */
  productLoadError = signal<string | null>(null);

  ngOnInit(): void {
    this.loadProducts();
  }

  /**
   * Fetches all products from the API and populates the products signal.
   * Resets state before each fetch so stale errors are cleared on retry.
   */
  private loadProducts(): void {
    this.isLoadingProducts.set(true);
    this.productLoadError.set(null);

    this.productService.getProducts()
      .pipe(
        finalize(() => this.isLoadingProducts.set(false)),
        takeUntilDestroyed(this.destroyRef),
      )
      .subscribe({
        next:  (productList)    => this.products.set(productList),
        error: (fetchError: Error) => this.productLoadError.set(fetchError.message),
      });
  }
}
```

```html
<!-- product-list.component.html -->
@if (isLoadingProducts()) {
  <app-loading-spinner aria-label="Loading products" />

} @else if (productLoadError()) {
  <p class="error-message" role="alert" aria-live="assertive">
    {{ productLoadError() }}
  </p>

} @else {
  <section aria-label="Product catalogue">
    <div class="product-grid">
      @for (product of products(); track product.id) {
        <app-product-card [product]="product" />
      } @empty {
        <p class="empty-state">No products are available at this time.</p>
      }
    </div>
  </section>

  <!-- Analytics panel is deferred until it scrolls into view -->
  @defer (on viewport; prefetch on idle) {
    <app-analytics-panel />
  } @placeholder {
    <div class="analytics-placeholder" aria-hidden="true" style="height: 200px;"></div>
  } @loading (minimum 300ms) {
    <app-loading-spinner />
  }
}
```

---

## Example 2: Order Type System

Demonstrates: interface, string enum, constants, mapper — all in their dedicated folders.

**Input (what the user asks):**
> "Define the complete Order type system: interface, status enum, and a mapper that
> translates snake_case API fields to our camelCase domain model."

**Expected output:**

```typescript
// models/enums/order-status.enum.ts

/** Lifecycle states of a customer order as used in API payloads and the UI. */
export enum OrderStatus {
  Pending    = 'PENDING',
  Processing = 'PROCESSING',
  Shipped    = 'SHIPPED',
  Delivered  = 'DELIVERED',
  Cancelled  = 'CANCELLED',
}
```

```typescript
// models/interfaces/i-order.interface.ts
import { OrderStatus } from '../enums/order-status.enum';

/** Shape of an order record as returned by the Orders REST API. */
export interface IOrder {
  readonly orderId:    string;
  readonly customerId: string;
  status:              OrderStatus;
  totalAmount:         number;
  currencyCode:        string;
  placedAt:            string;   // ISO 8601 — convert to Date in view layer if needed
  items:               IOrderItem[];
}

/** A single product line within an order. */
export interface IOrderItem {
  readonly productId: string;
  productName:        string;
  quantity:           number;
  unitPrice:          number;
}

/** DTO used when creating a new order via POST /orders. */
export interface ICreateOrderDto {
  customerId: string;
  items:      Array<{ productId: string; quantity: number }>;
}

/** DTO for status-only partial updates via PATCH /orders/:id. */
export interface IUpdateOrderStatusDto {
  status: OrderStatus;
}
```

```typescript
// models/constants/api.constants.ts

/** URL paths and builder functions for the Orders REST API (v1). */
export const ORDERS_API = {
  BASE:   '/api/v1/orders',
  BY_ID:  (orderId: string) => `/api/v1/orders/${orderId}`,
  STATUS: (orderId: string) => `/api/v1/orders/${orderId}/status`,
  EXPORT: '/api/v1/orders/export',
} as const;
```

```typescript
// models/mappers/order.mapper.ts
import { IOrder, IOrderItem } from '../interfaces/i-order.interface';
import { OrderStatus }        from '../enums/order-status.enum';

/**
 * Transforms raw Orders API data into typed IOrder objects.
 * Centralises all field aliasing between the API's snake_case keys and the
 * application's camelCase domain properties.
 *
 * All methods are static — never instantiate this class.
 */
export class OrderMapper {
  /**
   * Converts a single raw API order record to a typed IOrder.
   * @param rawOrder - Untyped record from the API response body
   * @returns A fully typed IOrder conforming to the interface contract
   */
  static fromApi(rawOrder: Record<string, unknown>): IOrder {
    return {
      orderId:     rawOrder['order_id']     as string,
      customerId:  rawOrder['customer_id']  as string,
      status:      rawOrder['status']       as OrderStatus,
      totalAmount: rawOrder['total_amount'] as number,
      currencyCode:(rawOrder['currency_code'] as string) ?? 'USD',
      placedAt:    rawOrder['placed_at']    as string,
      items:       (rawOrder['items'] as Record<string, unknown>[]).map(
                     OrderMapper.mapLineItem,
                   ),
    };
  }

  /**
   * Converts an array of raw API order records to typed IOrder objects.
   * @param rawOrders - Array of untyped API response objects
   * @returns An array of typed IOrder objects
   */
  static fromApiList(rawOrders: Record<string, unknown>[]): IOrder[] {
    return rawOrders.map(OrderMapper.fromApi);
  }

  /**
   * Maps a single raw line-item record to a typed IOrderItem.
   * Private — only called by fromApi; not exposed in the public mapper API.
   */
  private static mapLineItem(rawItem: Record<string, unknown>): IOrderItem {
    return {
      productId:   rawItem['product_id']   as string,
      productName: rawItem['product_name'] as string,
      quantity:    rawItem['qty']          as number,
      unitPrice:   rawItem['unit_price']   as number,
    };
  }
}
```

---

## Example 3: NgRx Facade in a Component

Demonstrates: consuming a Facade, `async` pipe, `takeUntilDestroyed`, no direct store access.

**Input (what the user asks):**
> "Create an OrdersDashboardComponent that uses the OrdersFacade to show the order list,
> a loading indicator, and an error state."

**Expected output:**

```typescript
// features/orders/components/orders-dashboard/orders-dashboard.component.ts

/**
 * Displays the orders overview dashboard: a list of all orders with loading
 * and error feedback. Interacts exclusively with OrdersFacade — never with the
 * NgRx store directly.
 */
@Component({
  selector:        'app-orders-dashboard',
  standalone:      true,
  imports:         [AsyncPipe, OrderCardComponent, LoadingSpinnerComponent],
  templateUrl:     './orders-dashboard.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class OrdersDashboardComponent implements OnInit {
  private ordersFacade = inject(OrdersFacade);

  /** Stream of all orders from the store — consumed by the async pipe in the template. */
  orders$       = this.ordersFacade.orders$;

  /** Stream of the loading flag — drives the global spinner visibility. */
  isLoading$    = this.ordersFacade.isLoading$;

  /** Stream of the user-facing error message, null when no error is present. */
  errorMessage$ = this.ordersFacade.errorMessage$;

  ngOnInit(): void {
    this.ordersFacade.loadOrders();
  }
}
```

```html
<!-- orders-dashboard.component.html -->
@if (isLoading$ | async) {
  <app-loading-spinner aria-label="Loading orders" />

} @else if (errorMessage$ | async; as errorMessage) {
  <p class="error-message" role="alert">{{ errorMessage }}</p>

} @else {
  <section aria-label="Orders list">
    @for (order of (orders$ | async) ?? []; track order.orderId) {
      <app-order-card [order]="order" />
    } @empty {
      <p class="empty-state">No orders found.</p>
    }
  </section>
}
```

---

## Example 4: Accessible Login Form

Demonstrates: Reactive Forms strict typing, custom validator, `aria-live` error messages,
keyboard accessibility, `isSubmitting` signal.

**Input (what the user asks):**
> "Create a LoginFormComponent with email and password fields, a custom password-strength
> validator, inline error messages that are announced to screen readers, and a submit
> button disabled while the request is in flight."

**Expected output:**

```typescript
// models/validators/password.validator.ts

/**
 * Validates that a password contains at least one uppercase letter,
 * one lowercase letter, and one numeric digit.
 *
 * @returns ValidatorFn returning a `passwordStrength` error key on failure, null on pass.
 */
export function passwordStrengthValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const passwordValue = control.value as string;

    if (!passwordValue) {
      return null;  // Validators.required handles the empty case separately
    }

    const hasUpperCase   = /[A-Z]/.test(passwordValue);
    const hasLowerCase   = /[a-z]/.test(passwordValue);
    const hasNumericChar = /[0-9]/.test(passwordValue);

    return hasUpperCase && hasLowerCase && hasNumericChar
      ? null
      : {
          passwordStrength:
            'Password must contain at least one uppercase letter, one lowercase letter, and one number.',
        };
  };
}
```

```typescript
// models/interfaces/i-login-form.interface.ts

/** Typed shape of the login reactive form controls. */
export interface ILoginForm {
  email:      FormControl<string>;
  password:   FormControl<string>;
  rememberMe: FormControl<boolean>;
}
```

```typescript
// features/auth/components/login-form/login-form.component.ts

/**
 * Handles user authentication via a strictly-typed reactive form.
 * Validates email format and password strength before submission.
 * All form errors are surfaced inline and announced to screen readers via aria-live.
 */
@Component({
  selector:        'app-login-form',
  standalone:      true,
  imports:         [ReactiveFormsModule],
  templateUrl:     './login-form.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LoginFormComponent {
  private authService = inject(AuthService);
  private router      = inject(Router);

  /** Strictly-typed login form — nonNullable ensures getRawValue() never returns null. */
  loginForm = new FormGroup<ILoginForm>({
    email: new FormControl('', {
      nonNullable: true,
      validators:  [Validators.required, Validators.email],
    }),
    password: new FormControl('', {
      nonNullable: true,
      validators:  [
        Validators.required,
        Validators.minLength(8),
        passwordStrengthValidator(),
      ],
    }),
    rememberMe: new FormControl(false, { nonNullable: true }),
  });

  /** True while the login API call is in flight — disables the submit button. */
  isSubmitting = signal(false);

  /** Convenience accessors for cleaner template bindings. */
  get emailControl()    { return this.loginForm.controls.email; }
  get passwordControl() { return this.loginForm.controls.password; }

  /**
   * Submits the form if valid; marks all controls as touched on invalid
   * submission so field-level errors are surfaced to the user.
   */
  onSubmit(): void {
    if (this.loginForm.invalid) {
      this.loginForm.markAllAsTouched();
      return;
    }

    this.isSubmitting.set(true);

    this.authService
      .login(this.loginForm.getRawValue())
      .pipe(finalize(() => this.isSubmitting.set(false)))
      .subscribe({
        next: () => this.router.navigate(['/dashboard']),
      });
  }
}
```

```html
<!-- login-form.component.html -->
<form [formGroup]="loginForm" (ngSubmit)="onSubmit()" novalidate>

  <!-- Email field -->
  <div class="form-field">
    <label for="email">Email address</label>
    <input
      id="email"
      type="email"
      formControlName="email"
      autocomplete="email"
      [attr.aria-invalid]="emailControl.invalid && emailControl.touched"
      aria-describedby="email-error"
    />
    <!-- aria-live ensures the error is read aloud when it appears -->
    <span id="email-error" role="alert" aria-live="polite" class="field-error">
      @if (emailControl.invalid && emailControl.touched) {
        @if (emailControl.hasError('required')) { Email address is required. }
        @if (emailControl.hasError('email'))    { Enter a valid email address. }
      }
    </span>
  </div>

  <!-- Password field -->
  <div class="form-field">
    <label for="password">Password</label>
    <input
      id="password"
      type="password"
      formControlName="password"
      autocomplete="current-password"
      [attr.aria-invalid]="passwordControl.invalid && passwordControl.touched"
      aria-describedby="password-error"
    />
    <span id="password-error" role="alert" aria-live="polite" class="field-error">
      @if (passwordControl.invalid && passwordControl.touched) {
        @if (passwordControl.hasError('required'))        { Password is required. }
        @if (passwordControl.hasError('minlength'))       { Password must be at least 8 characters. }
        @if (passwordControl.hasError('passwordStrength')){ {{ passwordControl.errors?.['passwordStrength'] }} }
      }
    </span>
  </div>

  <!-- Remember me -->
  <div class="form-field form-field--inline">
    <input id="rememberMe" type="checkbox" formControlName="rememberMe" />
    <label for="rememberMe">Keep me signed in</label>
  </div>

  <!-- Submit — disabled while the API call is in flight -->
  <button
    type="submit"
    class="btn btn--primary"
    [disabled]="isSubmitting()"
    [attr.aria-busy]="isSubmitting()"
  >
    {{ isSubmitting() ? 'Signing in…' : 'Sign in' }}
  </button>

</form>
```
