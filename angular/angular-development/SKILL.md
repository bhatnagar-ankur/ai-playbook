---
name: angular-development
description: >
  Enforces Angular 17+ best practices for components, services, forms, HTTP, state
  management, routing, accessibility, and testing. Use when the user asks to create,
  scaffold, refactor, or review Angular code — components, directives, pipes, services,
  guards, interceptors, resolvers, routes, or forms. Also use when the user mentions
  standalone component, signals, NgRx, BehaviorSubject, Reactive Forms, Signal Forms,
  lazy loading, OnPush, @if/@for/@switch/@defer, input/output signals, inject(),
  interface, enum, mapper, WCAG, accessibility, NgOptimizedImage, httpResource,
  zoneless, or any Angular CLI command. Apply to any file importing from `@angular/core`
  or containing `@Component`, `@Injectable`, `@Directive`, or `@Pipe`. Apply when
  migrating NgModule code to standalone.
version: 1.0.0
technology: angular
author: Ankur Bhatnagar
last_updated: 2026-07-21
compatibility: Angular >= 17, Angular CLI >= 17, Node >= 20, TypeScript >= 5.2
---

# Angular Development

## Overview

Codifies standards for Angular 17+ projects covering architecture, type contracts,
components, forms, state management, HTTP, routing, accessibility, performance, and
testing. UI-library-agnostic; applies across projects using Angular Material, PrimeNG,
DevExtreme, KendoUI, or custom libraries.

For detailed implementations see the `references/` folder — files there are loaded
only when relevant, at no extra token cost during tasks that do not need them.

## When to Use

Any Angular artifact (component, service, directive, pipe, guard, interceptor, resolver,
routes file, form), any Angular template, migration from NgModule to standalone, or any
`.component.ts`, `.service.ts`, `.directive.ts`, `.pipe.ts`, `.guard.ts`, `.spec.ts`,
or matching `.html` file.

Example trigger phrases: `"create a component"`, `"add a route guard"`,
`"scaffold a service"`, `"write a form validator"`, `"migrate this module to standalone"`.

## When NOT to Use

React, Next.js, or Vue projects; generic TypeScript or Node scripts not tied to Angular;
pure CSS/SCSS with no Angular-specific context.

---

## 1. Project Architecture & Folder Structure

All type contracts live under `models/` — never inside a feature or component folder.

```
src/app/
├── core/                          ← Singleton services, interceptors, guards, tokens
│   ├── guards/
│   ├── http/                      ← BaseHttpService + IHttpOptions
│   ├── interceptors/              ← auth, loading, error, logging
│   ├── services/                  ← LoadingService, NotificationService, LoggerService
│   └── tokens/                    ← InjectionTokens, HttpContextTokens
│
├── shared/                        ← Reusable standalone components, directives, pipes
│   ├── components/
│   ├── directives/
│   └── pipes/
│
├── models/                        ← ALL type contracts — never duplicate across features
│   ├── interfaces/                ← IUser, IOrder (I-prefix mandatory)
│   ├── enums/                     ← OrderStatus, UserRole
│   ├── constants/                 ← APP_ROUTES, PAGINATION_DEFAULTS
│   ├── validators/                ← Custom ValidatorFn factories
│   └── mappers/                   ← API DTO → domain model transforms
│
└── features/                      ← One folder per domain
    └── orders/
        ├── components/
        ├── services/
        ├── store/                 ← NgRx: actions, reducer, effects, selectors, facade
        └── orders.routes.ts
```

**Rules:**
- `models/` is global — never define the same type in two places.
- `core/` services use `providedIn: 'root'`; never import them into feature modules.
- `shared/` artifacts are standalone and imported individually where needed.
- No component, facade, or store effect may inject `HttpClient` directly.
  All HTTP calls go through `BaseHttpService`. → See `references/http-layer.md`.

---

## 2. Type System — Interfaces, Models, Enums, Constants & Mappers

> Full annotated examples: `references/type-system.md`

### Naming rules

| Artifact | Convention | Example |
|---|---|---|
| Interface | `I` prefix + `PascalCase` | `IUser`, `IPagedResponse<T>` |
| Class/Model | `PascalCase` | `UserModel`, `OrderModel` |
| Enum name | `PascalCase` | `OrderStatus`, `UserRole` |
| Enum values | `PascalCase` (string) | `OrderStatus.Pending = 'PENDING'` |
| Constants object | `SCREAMING_SNAKE_CASE` | `PAGINATION_DEFAULTS` |
| Mapper class | `PascalCase` + `Mapper` suffix | `OrderMapper` |

### Key rules

- All interfaces must be prefixed with `I` — no exceptions. This makes contracts
  distinguishable from concrete classes/models at a glance, without checking the definition.
- Use **string enums** by default (survive serialisation, readable in logs).
- Every property must have an explicit type — never use `any`; use `unknown` and narrow.
- Use `readonly` on properties that must not be mutated after creation.
- Mapper static methods: `fromApi()`, `fromApiList()`, `toApiDto()`.
- Custom `ValidatorFn` factories live in `models/validators/`, not in components.

### Minimal quick-reference

```typescript
// Interface
export interface IOrder { readonly orderId: string; status: OrderStatus; }

// Enum
export enum OrderStatus { Pending = 'PENDING', Shipped = 'SHIPPED' }

// Constants
export const PAGINATION_DEFAULTS = { PAGE_SIZE: 25, DEFAULT_PAGE: 1 } as const;

// Mapper
export class OrderMapper {
  static fromApi(raw: Record<string, unknown>): IOrder { ... }
}
```

---

## 3. Component Architecture

```typescript
/**
 * [What this component does and where it is used — mandatory summary.]
 */
@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './user-card.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,  // mandatory on every component
})
export class UserCardComponent {
  private userService = inject(UserService);         // inject() over constructor injection

  user    = input.required<IUser>();                 // signal input — required
  theme   = input<'light' | 'dark'>('light');        // signal input — with default
  selected = output<IUser>();                        // typed signal output

  fullName = computed(() => `${this.user().firstName} ${this.user().lastName}`);
}
```

**Rules:**
- `ChangeDetectionStrategy.OnPush` on every component — no exceptions.
- `inject()` for DI in new code; constructor injection only when maintaining existing code.
  `inject()` works in functional guards, interceptors, and route resolvers where there is
  no constructor, needs less boilerplate, and is easier to call in isolation in tests.
- Every class must have a JSDoc summary directly above its decorator.
- Keep components under 200 lines; extract logic into services.

### When NgModule is still needed

In a standalone-first codebase, create an `NgModule` only when:
- Wrapping a third-party library that exports an `NgModule` (some older DevExtreme / KendoUI modules).
- Publishing a set of components as an npm library for external consumers.
- Incrementally migrating an existing feature — never rewrite a working module all at once.

For all new features: **standalone components + `*.routes.ts`** only.

---

## 4. Naming & Variable Conventions

| Construct | Convention | Example |
|---|---|---|
| Classes, enums, interfaces | `PascalCase` | `UserModel`, `IUser` |
| Variables, methods, properties | `camelCase` | `fetchOrders()`, `userList` |
| Module-level constants | `SCREAMING_SNAKE_CASE` | `MAX_PAGE_SIZE` |
| Private signals / subjects | `_` prefix + `camelCase` | `_items`, `_isLoading` |
| Component selectors | `app-` prefix + kebab-case | `app-user-card` |
| Files | kebab-case | `user-card.component.ts` |
| Observables | `$` suffix | `users$`, `selectedOrder$` |

**Variable naming rules — enforced, no exceptions:**
- No single-letter variable names anywhere.
- No vague abbreviations (`errMsg` → `errorMessage`, `ui` → `userIndex`).
- Booleans must be prefixed with `is`, `has`, `can`, or `should`:
  `isLoading`, `hasPermission`, `canDelete`, `shouldRefresh`.
- Collections must be plural: `users`, `orderItems`.

---

## 5. JSDoc Comments

- Every `@Component`, `@Injectable`, `@Directive`, `@Pipe`: class-level JSDoc required.
- Every public method: JSDoc with `@param` and `@returns` for non-trivial signatures.
- Private methods: JSDoc when purpose is not immediately obvious from the name.
- Inline comments for business rules, workarounds, or API quirks — not for obvious code.

```typescript
/**
 * Manages the shopping cart state using Angular Signals.
 * Scoped to this feature; does not depend on the global NgRx store.
 */
@Injectable({ providedIn: 'root' })
export class CartService {
  /**
   * Adds an item or increments its quantity if it already exists.
   * @param newItem - Cart item to add; must have a stable unique id
   */
  addItem(newItem: ICartItem): void { ... }
}
```

---

## 6. State Management

> Full tier examples with complete code: `references/state-management.md`

Choose the lowest tier that solves the problem. Migrate up only when the simpler tier
creates real friction.

| Scenario | Tier | Approach |
|---|---|---|
| Component-local or single-feature state | **1 — Signals** | `signal()` + `computed()` in a service |
| Shared across 2–3 features, reactive streams needed | **2 — RxJS** | `BehaviorSubject` + `.asObservable()` |
| App-wide, complex async effects, audit trails | **3 — NgRx** | Store + Effects + **Facade** |

**Rules common to all tiers:**
- Never expose a `BehaviorSubject` directly; always expose `.asObservable()`.
- Use `takeUntilDestroyed(this.destroyRef)` — never manual `ngOnDestroy` subscriptions.
- Components interact with NgRx only through a **Facade service** — never dispatch or select directly.

---

## 7. Templates — Control Flow

Use Angular 17+ built-in control flow in all new code.
Never use `*ngIf`, `*ngFor`, `*ngSwitch` in new files.

```html
@if (isLoading()) {
  <app-loading-spinner />
} @else if (errorMessage()) {
  <p class="error-message" role="alert">{{ errorMessage() }}</p>
} @else {
  <!-- Always provide a stable track expression — never track by $index unless no stable ID -->
  @for (orderItem of orderItems(); track orderItem.id) {
    <app-order-item [item]="orderItem" />
  } @empty {
    <p class="empty-state">No orders found.</p>
  }
}

@switch (orderStatus()) {
  @case (OrderStatus.Pending)   { <span class="badge badge--pending">Pending</span> }
  @case (OrderStatus.Shipped)   { <span class="badge badge--shipped">Shipped</span> }
  @default                      { <span class="badge badge--unknown">Unknown</span> }
}
```

---

## 8. Forms

### 8a. Reactive Forms — standard for Angular 17–20

Use **strictly-typed** Reactive Forms (`FormGroup<T>`, `FormControl<T>`, `nonNullable: true`).
Define a form-shape interface in `models/interfaces/` to co-locate the type contract.

```typescript
// models/interfaces/i-login-form.interface.ts
export interface ILoginForm {
  email:      FormControl<string>;
  password:   FormControl<string>;
  rememberMe: FormControl<boolean>;
}
```

```typescript
/**
 * Handles user authentication input via a strictly-typed reactive form.
 * Delegates submission to AuthService; does not call the API directly.
 */
@Component({
  selector: 'app-login-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './login-form.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LoginFormComponent {
  private authService = inject(AuthService);

  /** Strictly-typed login form — all controls are non-nullable with explicit defaults. */
  loginForm = new FormGroup<ILoginForm>({
    email: new FormControl('', {
      nonNullable: true,
      validators: [Validators.required, Validators.email],
    }),
    password: new FormControl('', {
      nonNullable: true,
      validators: [Validators.required, Validators.minLength(8), passwordStrengthValidator()],
    }),
    rememberMe: new FormControl(false, { nonNullable: true }),
  });

  /** True while the login API call is in flight. */
  isSubmitting = signal(false);

  /**
   * Submits the login form if valid; sets submitting state around the API call.
   * Marks all controls as touched on invalid submission to surface validation errors.
   */
  onSubmit(): void {
    if (this.loginForm.invalid) {
      this.loginForm.markAllAsTouched();
      return;
    }
    this.isSubmitting.set(true);
    // getRawValue() returns the fully typed value including disabled controls
    this.authService.login(this.loginForm.getRawValue())
      .pipe(finalize(() => this.isSubmitting.set(false)))
      .subscribe();
  }
}
```

### 8b. Custom Validators — live in `models/validators/`

```typescript
// models/validators/password.validator.ts

/**
 * Validates that a password contains at least one uppercase letter,
 * one lowercase letter, and one numeric digit.
 *
 * @returns A ValidatorFn that returns a `passwordStrength` error key on failure.
 */
export function passwordStrengthValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const passwordValue = control.value as string;

    if (!passwordValue) {
      return null;  // let Validators.required handle empty values
    }

    const hasUpperCase  = /[A-Z]/.test(passwordValue);
    const hasLowerCase  = /[a-z]/.test(passwordValue);
    const hasNumericChar = /[0-9]/.test(passwordValue);

    return hasUpperCase && hasLowerCase && hasNumericChar
      ? null
      : { passwordStrength: 'Password must contain uppercase, lowercase, and numeric characters.' };
  };
}
```

### 8c. Template — form error display pattern

```html
<form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
  <div class="form-field">
    <label for="email">Email</label>
    <input id="email" type="email" formControlName="email" autocomplete="email" />
    @if (loginForm.controls.email.invalid && loginForm.controls.email.touched) {
      <span class="field-error" role="alert" aria-live="polite">
        @if (loginForm.controls.email.hasError('required')) { Email is required. }
        @if (loginForm.controls.email.hasError('email'))    { Enter a valid email address. }
      </span>
    }
  </div>

  <button type="submit" [disabled]="isSubmitting()">
    {{ isSubmitting() ? 'Signing in…' : 'Sign in' }}
  </button>
</form>
```

### 8d. Signal Forms (Angular 21+)

Signal Forms are stable from Angular v21 onward. For projects on Angular 17–20, use the
Reactive Forms pattern in section 8a above. When the project upgrades to v21+, migrate forms to the
Signal Forms API (`signalForm()`, `signalInput()`) for native signal integration and
simpler async validation. The validator functions in `models/validators/` are compatible
with both approaches.

---

## 9. Routing & Lazy Loading

```typescript
// app.routes.ts
export const APP_ROUTES: Routes = [
  {
    path: 'orders',
    loadChildren: () =>
      import('./features/orders/orders.routes').then(routes => routes.ORDER_ROUTES),
  },
  {
    path: 'users',
    loadComponent: () =>
      import('./features/users/components/user-list/user-list.component')
        .then(component => component.UserListComponent),
    canActivate: [authGuard],
  },
  { path: '',   redirectTo: '/dashboard', pathMatch: 'full' },
  { path: '**', loadComponent: () => import('./shared/components/not-found/not-found.component')
                                       .then(component => component.NotFoundComponent) },
];
```

- `loadComponent` for single-component routes; `loadChildren` for feature route files.
- Functional guards only (`CanActivateFn`, `CanMatchFn`) — no class-based guards for new code.
  Functional guards are tree-shakeable, require no DI boilerplate (no `@Injectable`
  class just to hold one check), and compose naturally with `inject()`.

```typescript
/** Redirects unauthenticated users to /login, preserving the intended URL. */
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router      = inject(Router);
  return authService.isLoggedIn()
    ? true
    : router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};
```

---

## 10. HTTP Layer

> Full implementation: `references/http-layer.md`

All HTTP communication follows this chain — no component or facade may bypass it:

```
Interceptors (auth → loading → error → logging)
        ↓
BaseHttpService   ← only class that uses HttpClient; generic typed verbs
        ↓
Feature API Service (e.g. OrdersApiService)
        ↓
Component / Facade
```

**Quick rules:**
- `provideHttpClient(withInterceptors([...]))` in `app.config.ts` — not `HttpClientModule`.
- `BaseHttpService` exposes: `get`, `post`, `put`, `patch`, `delete`, `head`, `options`.
- Pass `{ skipLoading: true }` to suppress the global spinner for background calls.
- Pass `{ skipAuth: true }` for public endpoints (login, token refresh).
- Loading interceptor uses a **counter**, not a boolean — concurrent calls do not cancel each other.

---

## 11. Accessibility

WCAG 2.1 AA compliance is mandatory on all user-facing output.

**Semantic HTML first — always prefer native elements over ARIA:**
- Use `<button>` (not `<div role="button">`) for clickable actions.
- Use `<nav>`, `<main>`, `<section>`, `<article>`, `<aside>` for page regions.
- Use `<label for="...">` or `aria-label` on every form control.

**Mandatory rules:**
- Every `<img>` must have an `alt` attribute (empty string `alt=""` for decorative images).
- Interactive elements must be keyboard-accessible — test Tab/Shift-Tab/Enter/Space/Escape.
- Focus must be managed when content changes dynamically (dialogs opening, route navigation,
  error messages appearing): use `cdkTrapFocus` (CDK) or `@angular/cdk/a11y` `FocusMonitor`.
- Dynamic messages (errors, status updates) need `aria-live="polite"` or `role="alert"`.
- Color contrast ratio ≥ 4.5:1 for normal text, ≥ 3:1 for large text (WCAG AA).
- Never convey information by color alone — add an icon, label, or pattern.

**Tooling:**
- Install `axe-core` (`npm i -D axe-core`) and run `axe(document)` in development to catch
  violations before code review.
- Use the `CDK A11y` module (`@angular/cdk/a11y`) for focus traps, live announcements,
  and high-contrast mode detection.

```html
<!-- Error announced to screen readers as soon as it appears -->
<span role="alert" aria-live="polite" class="field-error">
  {{ errorMessage() }}
</span>

<!-- Decorative icon — hidden from assistive technology -->
<mat-icon aria-hidden="true">chevron_right</mat-icon>

<!-- Meaningful icon used alone — must have a label -->
<button aria-label="Delete order {{ order.orderId }}">
  <mat-icon aria-hidden="true">delete</mat-icon>
</button>
```

---

## 12. Performance & Modern Angular APIs

### @defer — lazy rendering

Use `@defer` to push non-critical UI out of the initial render. Prefer `on viewport` for
below-the-fold content and `on idle` for analytics or low-priority widgets.

```html
<!-- Heavy chart deferred until it enters the viewport -->
@defer (on viewport; prefetch on idle) {
  <app-analytics-chart [data]="reportData()" />
} @placeholder {
  <!-- Shown before the deferred block is triggered -->
  <div class="chart-placeholder" aria-busy="true" style="height: 300px;"></div>
} @loading (minimum 300ms) {
  <!-- Shown while the lazy chunk is downloading -->
  <app-loading-spinner />
} @error {
  <p class="error-message" role="alert">Failed to load chart. Please refresh.</p>
}
```

**`@defer` trigger options:** `on idle`, `on viewport`, `on interaction`, `on hover`,
`on timer(Xms)`, `on immediate`, `when <expression>`. Add `prefetch on idle` to download
the chunk early without rendering it.

### NgOptimizedImage — optimised image loading

```typescript
// Import in the component's imports array
import { NgOptimizedImage } from '@angular/common';
```

```html
<!-- LCP image — add priority to disable lazy loading and preload the request -->
<img ngSrc="/assets/hero.webp" width="1200" height="600" priority
     alt="Dashboard hero banner" />

<!-- Non-LCP images are lazy-loaded by default -->
<img ngSrc="/assets/product-{{ product.id }}.jpg" width="300" height="300"
     alt="{{ product.name }} product thumbnail" />
```

- Always provide `width` and `height` to prevent layout shift (CLS).
- Never use `ngSrc` and `src` on the same element.

### Future APIs — version notes

| API | Stable from | Notes |
|---|---|---|
| **Signal Forms** (`signalForm()`) | Angular 21 | Validator functions in `models/validators/` are compatible — no rewrite needed |
| **`httpResource()`** | Angular 19 (preview) | Reactive GET binding to a signal; replaces service `get()` + `subscribe()` for read-only data. Do not use in production on Angular 17/18. |
| **Zoneless change detection** | Angular 18 (experimental), 19+ (stable) | Opt in with `provideExperimentalZonelessChangeDetection()`. Requires all state changes to flow through signals or `markForCheck()`. Remove `zone.js` from `polyfills` once fully migrated. |

---

## 13. Browser Compatibility

- Maintain a `.browserslistrc` at the project root; Angular CLI targets it automatically.
- Angular's new control flow, signals, `@defer`, and `NgOptimizedImage` are all compiled
  down and work in all CLI-targeted browsers — use them freely.
- Feature-detect optional browser APIs before use:

```typescript
// Safe: check before using APIs that may be absent in older targets
if (typeof IntersectionObserver !== 'undefined') {
  const observer = new IntersectionObserver(this.handleIntersection.bind(this));
  observer.observe(this.elementRef.nativeElement);
}
```

- Do not write vendor-prefixed CSS manually; PostCSS/Autoprefixer is included in the CLI.
- Verify support on [caniuse.com](https://caniuse.com) before using any CSS feature
  that is not in the project's `browserslist` baseline.
- Never use `!important` — fix specificity with a more targeted selector or restructure the View Encapsulation boundary instead.

---

## 14. Code Quality

- `strict: true` in `tsconfig.json` — no exceptions.
- Never use `any`; use `unknown` and narrow where necessary.
- `readonly` on arrays and objects that must not be mutated.
- One class per file.
- No business logic in templates — only method calls and signal/observable reads.
- Always handle loading and error states; never leave the UI in an ambiguous state.
- Components: max 200 lines. Extract into services or composable functions beyond that.
- UI libraries: do not mix libraries within the same feature
  (e.g. no PrimeNG table inside an Angular Material dialog) — mixing libraries bloats
  bundle size and produces inconsistent theming and accessibility behavior across the feature.

---

## 15. Testing

> Full patterns, examples, and mocking guides: `references/testing.md`

**Rules:**
- Use `@testing-library/angular` for component tests; Jest or Karma/Jasmine for unit tests.
- Mock all HTTP with `provideHttpClientTesting()`.
- Test **behaviour**, not implementation — never test private methods.
- Test case names as full sentences: `'should redirect to /login when token is expired'`.
- Coverage targets: ≥ 80% on services; ≥ 70% on components.
- Use `fakeAsync` + `tick()` for timer-based async code.

---

## Examples

> Full annotated examples: `references/examples.md`

- **Example 1** — standalone `ProductListComponent` with signals, loading/error states, `@for` + `@empty`
- **Example 2** — `IOrder` interface + `OrderStatus` enum + `OrderMapper` working together
- **Example 3** — NgRx `OrdersFacade` hiding store from components
- **Example 4** — accessible login form with Reactive Forms, custom validator, and `aria-live` error messages

---

## Customizing

| Topic | File | When to read |
|---|---|---|
| Type system — interfaces, models, enums, mappers | `references/type-system.md` | When creating or reviewing interfaces, enums, constants, or API-to-domain mappers |
| HTTP layer | `references/http-layer.md` | When adding an API service, interceptor, or any code that calls `BaseHttpService` |
| State management | `references/state-management.md` | When choosing or implementing a state tier (Signals, RxJS, or NgRx) |
| Testing | `references/testing.md` | When writing or reviewing `.spec.ts` files, mocks, or coverage for services/components |
| Full examples | `references/examples.md` | When producing a complete feature or needing a production-ready end-to-end pattern |

## Customizing This Skill for Your Project

Create `references/project-overrides.md` in your project when you need to record
project-specific deviations from this skill (this file does not ship with the skill —
you create it the first time you need it). Document:

| What | Example overrides |
|---|---|
| Selector prefix | `acme-` instead of `app-` |
| State management default | "All features use NgRx; skip Tier 1/2" |
| UI library restrictions | "DevExtreme grid: disable virtual scrolling + column reorder together" |
| API base URL token | Token name and where it is provided |
| Test runner | Jest config path, custom matchers |
| Browserslist target | Specific browser/version matrix |
| Banned patterns | Specific APIs or approaches the team has ruled out |
| Folder deviations | Where the project diverges from the canonical layout |

**Contributing improvements back:** If you discover a pattern worth sharing, open a PR
against this file in the shared skill repository. Add the rule with a code example,
bump `version` (minor for additions, major for breaking changes), update `last_updated`.
