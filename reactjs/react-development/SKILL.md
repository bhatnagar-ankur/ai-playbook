---
name: react-development
version: 1.0.0
author: Ankur Bhatnagar
last_updated: 2026-09-07
description: >
  Enforces React 19+ best practices for components, hooks, type system,
  state management, HTTP layer, forms, routing, accessibility, and testing.
  Covers four state tiers (useState/useReducer, Context+useReducer, Zustand/Jotai,
  Redux Toolkit + TanStack Query) and a typed HTTP client with axios interceptors.
technology: react
compatibility: React >= 19, TypeScript >= 5.0
tags: [react, typescript, rtk, tanstack-query, zustand, jotai, react-hook-form, zod, tailwind, testing-library]
triggers:
  - React component, hook, or service file
  - useState, useReducer, useEffect, useContext, useRef, useMemo, useCallback
  - use() hook (React 19)
  - Redux Toolkit, RTK Query, createSlice, createAsyncThunk
  - TanStack Query, useQuery, useMutation, useInfiniteQuery
  - Zustand store, Jotai atom
  - React Hook Form, useForm, zodResolver
  - React Router v6, useNavigate, Outlet, RouterProvider
  - WCAG, aria, role, axe, accessibility
  - React.lazy, Suspense, React.memo, useDeferredValue, useTransition
  - CSS Modules, Tailwind, Styled Components
  - Jest, Vitest, @testing-library/react
---

# React Development Skill

This skill guides Claude to produce idiomatic, type-safe React 19+ code for production
applications. It covers all tiers from a single-component toggle up to complex cross-feature
state orchestration. Follow every rule in this document unless the **Customizing** section
overrides it for the current project.

---

## 1. When to Use This Skill

Apply when:
- Creating or refactoring React components (`.tsx`), custom hooks (`use*.ts`), or services
- Wiring up state management (RTK, TanStack Query, Zustand, Jotai, Context)
- Building forms with validation
- Setting up the HTTP layer (axios client + interceptors)
- Writing unit or integration tests for React code
- Reviewing existing code for correctness, accessibility, or performance

**Do NOT use when:**
- The project is a Next.js App Router app — defer to the `nextjs-development` skill instead,
  which already inherits this skill's type system, HTTP layer, and state management patterns

---

## 2. Project Structure

```
src/
├── app/                        # App shell, providers, router config
│   ├── App.tsx                 # Root component with provider tree
│   ├── providers.tsx           # Aggregated context/store providers
│   └── router.tsx              # React Router RouterProvider config
├── features/                   # One folder per product feature
│   └── orders/
│       ├── components/         # Feature UI components
│       ├── hooks/              # Custom hooks: use-orders.hook.ts
│       ├── services/           # API call modules: orders-api.service.ts
│       ├── store/              # RTK slices or Zustand stores
│       └── types/              # Feature re-exports from models/
├── shared/
│   ├── components/             # App-wide reusable UI (Button, Modal, etc.)
│   ├── hooks/                  # App-wide hooks (useDebounce, useLocalStorage)
│   └── utils/                  # Pure utility functions
├── models/
│   ├── interfaces/             # i-<entity>.interface.ts
│   ├── classes/                # <entity>.model.ts — domain models with computed/behaviour methods
│   ├── enums/                  # <entity>-<concept>.enum.ts
│   ├── constants/              # <domain>.constants.ts
│   └── mappers/                # <entity>.mapper.ts
├── core/
│   ├── http/                   # ApiClient, interceptors, IHttpOptions
│   └── auth/                   # Auth utilities, token helpers
└── styles/
    ├── globals.css
    └── variables.css
```

**Rules:**
- Feature code never imports from another feature — shared code lives in `shared/` or `models/`
- One component per file; file name matches the component name in `kebab-case`
- No barrel `index.ts` files that re-export everything — import directly from the source file
  (barrel files hurt tree-shaking and can cause circular-import issues)

---

## 3. Component Authoring Rules

### 3a. Always use functional components with TypeScript

```typescript
// features/orders/components/order-card/order-card.component.tsx

import React from 'react';
import type { IOrder } from '../../../../models/interfaces/i-order.interface';

interface OrderCardProps {
  readonly order: IOrder;
  onSelectOrder: (orderId: string) => void;
}

/**
 * Renders a summary card for a single order.
 * Calls onSelectOrder when the card is clicked or activated via keyboard.
 */
export function OrderCard({ order, onSelectOrder }: OrderCardProps): React.ReactElement {
  return (
    <article
      className="order-card"
      onClick={() => onSelectOrder(order.orderId)}
      onKeyDown={(e) => e.key === 'Enter' && onSelectOrder(order.orderId)}
      tabIndex={0}
      role="button"
      aria-label={`Order ${order.orderId}, status: ${order.status}`}
    >
      <h3 className="order-card__id">{order.orderId}</h3>
      <p className="order-card__status">{order.status}</p>
      <p className="order-card__total">{order.totalAmount} {order.currencyCode}</p>
    </article>
  );
}
```

**Component rules:**
- Props interface is always a named interface, never an inline type literal
- Props that should not be mutated by the child use `readonly`
- Always provide explicit return type (`React.ReactElement`)
- Use named exports — never default exports (makes refactoring safer)
- Wrap lists in semantic HTML (`<ul>/<li>`, `<section>`, `<article>`) not bare `<div>`
  (screen readers announce list semantics and item counts to assistive-technology users)

### 3b. Custom hooks

Custom hooks encapsulate logic that spans state + effects. They must start with `use`.

```typescript
// features/orders/hooks/use-orders.hook.ts

import { useQuery } from '@tanstack/react-query';
import { ordersApiService } from '../services/orders-api.service';
import { ORDERS_QUERY_KEYS } from '../constants/orders-query-keys.constants';

/**
 * Fetches all orders from the API with caching via TanStack Query.
 * Exposes loading, error, and data state for consumption in components.
 */
export function useOrders() {
  return useQuery({
    queryKey:  ORDERS_QUERY_KEYS.all,
    queryFn:   () => ordersApiService.getOrders(),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}
```

- Hook files: `use-<concept>.hook.ts`
- Hooks never render JSX — they return data, state, or callbacks
- Never call hooks conditionally

### 3c. React 19 features

| Feature | Rule |
|---|---|
| `use(promise)` | Use inside components/hooks to read async values with Suspense |
| `use(context)` | Prefer over `useContext()` in React 19 projects — same semantics, more flexible |
| `ref` as prop | No more `forwardRef` — pass `ref` directly to function components |
| `useActionState` | Use only for a form wired directly to a Server Action / async transition with **no form library** involved. For standard client-side forms with field-level validation UX, use React Hook Form + Zod instead (see Section 7) — `useActionState` is not a replacement for that pattern |
| `useOptimistic` | Use for optimistic UI updates before server confirmation |
| `<form action>` | Use Server Actions as the `action` prop (in Next.js) or `useActionState` action |

---

## 4. Type System

Full examples in `references/type-system.md`.

### 4a. Interfaces — mandatory `I` prefix

```typescript
// models/interfaces/i-user.interface.ts

/** User account as returned by the Users REST API. */
export interface IUser {
  readonly id:    string;
  readonly email: string;
  firstName:      string;
  lastName:       string;
  role:           UserRole;
  isActive:       boolean;
  createdAt:      string;  // ISO 8601 — convert to Date in the mapper
}
```

- Every API response shape gets an interface with the `I` prefix
- Every props interface is named (e.g., `OrderCardProps` — no `I` prefix for component props)
- Never use `any`; prefer `unknown` and narrow it in mappers

### 4b. Enums — always string values

```typescript
// models/enums/order-status.enum.ts
export enum OrderStatus {
  Pending    = 'PENDING',
  Processing = 'PROCESSING',
  Shipped    = 'SHIPPED',
  Delivered  = 'DELIVERED',
  Cancelled  = 'CANCELLED',
}
```

### 4c. Constants — `as const` objects in SCREAMING_SNAKE_CASE

```typescript
// models/constants/api.constants.ts
export const ORDERS_API = {
  BASE:  '/api/v1/orders',
  BY_ID: (orderId: string) => `/api/v1/orders/${orderId}`,
} as const;
```

### 4d. Mappers — static methods, never instantiate

```typescript
// models/mappers/order.mapper.ts
export class OrderMapper {
  static fromApi(raw: Record<string, unknown>): IOrder { ... }
  static fromApiList(rawList: Record<string, unknown>[]): IOrder[] { ... }
  static toApiDto(order: IOrder): ICreateOrderDto { ... }
}
```

---

## 5. State Management

Choose the tier that matches the scope of the state. Do not add complexity ahead of need.

| Tier | Tool | When to use |
|---|---|---|
| **T1 — Local** | `useState` / `useReducer` | UI toggles, form state, loading flags scoped to one component |
| **T2 — Feature-shared** | Zustand store or Jotai atom | State shared across 2–3 components in one feature, no server data |
| **T3 — Server state** | TanStack Query (`useQuery`, `useMutation`) | All API data — handles caching, background refetch, error/loading |
| **T4 — App-wide client** | Redux Toolkit (RTK) | Complex client-only state across many features; time-travel debug; audit trail |
| **T5 — Cross-cutting** | React Context + `useReducer` | Auth user, theme, i18n — values that change rarely |

Full patterns with examples in `references/state-management.md`.

**Rules:**
- Never reach for RTK when TanStack Query alone covers the need (server data is Query's job)
- Never put server data (API responses) into RTK or Zustand — let TanStack Query own it
- Never expose a Zustand store's setter directly from a component — wrap in a hook
  (encapsulation: lets you change the store's internal shape without touching every consumer,
  and gives you one place to add validation or logging later)
- Context is for values that change rarely; do not use it as a generic state bus

### 5a. TanStack Query — query key discipline

Query keys must be centralised constants, never inline strings:

```typescript
// features/orders/constants/orders-query-keys.constants.ts
export const ORDERS_QUERY_KEYS = {
  all:    ['orders']                       as const,
  byId:   (orderId: string) => ['orders', orderId] as const,
  filtered: (filters: IOrderFilters) => ['orders', 'filtered', filters] as const,
} as const;
```

### 5b. RTK — slice discipline

Each feature has one slice. Business logic belongs in `createAsyncThunk` or RTK Query endpoints, never inside components.

---

## 6. HTTP Layer

Full implementation in `references/http-layer.md`.

A single `ApiClient` module (wrapping axios) is the **only place** that constructs requests.
No component or hook may import `axios` directly.

```
core/http/
├── api-client.ts          # Configured axios instance
├── http-options.interface.ts  # IHttpOptions, skip flags
├── http-context.tokens.ts  # SKIP_LOADING, SKIP_AUTH metadata
└── interceptors/
    ├── auth.interceptor.ts
    ├── loading.interceptor.ts
    ├── error.interceptor.ts
    └── logging.interceptor.ts
```

**Rules:**
- Auth token injection: `auth.interceptor.ts` only
- 401 redirect: `error.interceptor.ts` only
- Loading counter: `LoadingService` (signal/atom-based counter, not boolean)
- Per-request opt-out: pass `{ skipAuth: true }` or `{ skipLoading: true }` via request config metadata
- Feature API services import `apiClient` from `core/http/api-client.ts` exclusively

---

## 7. Forms — React Hook Form + Zod

Full examples in `references/examples.md`.

**Decision rule:** Use React Hook Form + Zod (below) for standard client-side forms that need
field-level validation UX (inline errors, `aria-invalid`, disabling submit while pending).
Reserve `useActionState` (Section 3c) for a form wired directly to a Server Action or async
transition with no form library in the mix — the two patterns are not interchangeable and
should not be mixed within the same form.

```typescript
// models/validators/schemas/login.schema.ts
import { z } from 'zod';

/** Zod schema for the login form. Infer the TypeScript type from the schema. */
export const loginSchema = z.object({
  email:      z.string().email('Enter a valid email address.'),
  password:   z.string()
                .min(8, 'Password must be at least 8 characters.')
                .regex(/[A-Z]/, 'Password must contain at least one uppercase letter.')
                .regex(/[0-9]/, 'Password must contain at least one number.'),
  rememberMe: z.boolean().default(false),
});

export type ILoginFormValues = z.infer<typeof loginSchema>;
```

```typescript
// features/auth/components/login-form/login-form.component.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { loginSchema, type ILoginFormValues } from '../../../../models/validators/schemas/login.schema';

export function LoginForm(): React.ReactElement {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<ILoginFormValues>({ resolver: zodResolver(loginSchema) });

  const onSubmit = async (formData: ILoginFormValues): Promise<void> => {
    await authService.login(formData);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
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
      <button type="submit" disabled={isSubmitting} aria-busy={isSubmitting}>
        {isSubmitting ? 'Signing in…' : 'Sign in'}
      </button>
    </form>
  );
}
```

**Form rules:**
- Always use `zodResolver` — Zod is the single source of validation truth
- Derive the TypeScript type with `z.infer<typeof schema>` — never write the type twice
- Put schemas in `models/validators/schemas/<form-name>.schema.ts`
- Always mark form fields `aria-invalid` and connect error messages via `aria-describedby`
- Use `isSubmitting` from `formState` to disable the submit button — never manage it manually

---

## 8. Routing — React Router v6

```typescript
// app/router.tsx
import { createBrowserRouter, RouterProvider, Outlet } from 'react-router-dom';
import { lazy, Suspense } from 'react';
import { authGuard } from '../core/auth/auth.guard';

const OrdersPage = lazy(() => import('../features/orders/pages/orders-page/orders.page'));
const DashboardPage = lazy(() => import('../features/dashboard/pages/dashboard.page'));

const router = createBrowserRouter([
  {
    path:    '/',
    element: <RootLayout />,
    children: [
      {
        path:   'dashboard',
        element: (
          <Suspense fallback={<PageSpinner />}>
            <DashboardPage />
          </Suspense>
        ),
        loader: authGuard,
      },
      {
        path:   'orders',
        element: (
          <Suspense fallback={<PageSpinner />}>
            <OrdersPage />
          </Suspense>
        ),
        loader: authGuard,
      },
    ],
  },
]);

export function AppRouter(): React.ReactElement {
  return <RouterProvider router={router} />;
}
```

**Routing rules:**
- All route-level components are lazy-loaded with `React.lazy` + `Suspense`
- Auth protection via loader functions — never wrapping JSX in a ternary
  (a loader redirects before the protected component ever renders, avoiding a flash of
  protected content, and keeps the check colocated with routing instead of scattered through
  render logic)
- Use `createBrowserRouter` (v6 data router) — not `<BrowserRouter>` wrapping the tree

---

## 9. Accessibility (WCAG AA — Mandatory)

- Every interactive element is keyboard-reachable: `tabIndex`, `onKeyDown` for custom elements
- Every `<input>` has a visible `<label>` linked by `htmlFor` / `id`
- Error messages use `role="alert"` and `aria-live="polite"` (assertive only for critical errors)
- Images have descriptive `alt` text; decorative images use `alt=""`
- Colour contrast minimum 4.5:1 for text, 3:1 for UI components
- Modal dialogs trap focus and restore it on close — use `@radix-ui/react-dialog` or `@headlessui/react` rather than rolling custom focus traps
- Integrate `axe-core` in tests: `@axe-core/react` in development, `jest-axe` in unit tests

---

## 10. Performance

### 10a. Memoisation — use only where measured

```typescript
// Only wrap expensive computations — useCallback/useMemo add overhead themselves
const sortedOrders = useMemo(
  () => [...orders].sort((orderA, orderB) => orderB.totalAmount - orderA.totalAmount),
  [orders],
);

// Stable callback reference required when passed to memoised child
const handleSelectOrder = useCallback(
  (orderId: string) => dispatch(selectOrder(orderId)),
  [dispatch],
);
```

- `React.memo` on a component only when the parent re-renders frequently and props rarely change
- Profile with React DevTools Profiler before adding memoisation
- Prefer `useDeferredValue` for non-urgent state updates over manual debounce

### 10b. Code splitting

- Route-level lazy loading is mandatory (see Section 8)
- Heavy third-party components (charts, rich text editors) use `React.lazy` at the import site
- `Suspense` boundaries must have a meaningful `fallback` — never `fallback={null}`

### 10c. Images

- Use `<img>` with explicit `width` and `height` to prevent CLS
- For Next.js projects use `next/image` (see Next.js skill)
- Prefer `loading="lazy"` on below-the-fold images

---

## 11. Styling

This project uses multiple styling approaches — choose consistently within a feature.

| Approach | File convention | When to use |
|---|---|---|
| Tailwind CSS | Utility classes inline | Rapid layout, spacing, colour |
| CSS Modules | `component-name.module.css` | Complex component-specific styles |
| Styled Components / Emotion | `component-name.styles.ts` | Dynamic styles driven by component props |
| UI library (MUI, PrimeReact, DevExtreme, KendoUI) | Component APIs | Pre-built complex widgets |

**Rules:**
- Do not mix Tailwind and CSS Modules on the same element — pick one per component
- Never use inline `style={{}}` for anything other than truly dynamic values (e.g. chart widths)
- Never use `!important` — if you need it to win specificity, fix the selector or restructure the CSS Module scope instead
- UI library themes must be configured via the library's theme provider, not overridden with `!important`
- Do not mix UI libraries in the same feature (e.g. MUI + PrimeReact on the same page) —
  each library adds its own bundle weight, and mixing them produces inconsistent theming
  and accessibility behaviour across the page

---

## 12. Testing

Full patterns in `references/testing.md`.

**Stack:**
- **Vitest** (preferred) or **Jest** for the test runner
- **@testing-library/react** for component tests — behaviour-focused, no implementation details
- **MSW (Mock Service Worker)** for HTTP mocking in tests
- **jest-axe** for automated accessibility checks in unit tests

**Coverage targets:**

| Layer | Minimum |
|---|---|
| Custom hooks | 80% |
| Components | 70% |
| API services | 80% |
| Mappers | 90% |
| RTK reducers / Zustand stores | 90% |
| Zod schemas / validators | 90% |

**Test naming — full English sentences:**
```typescript
// ✅ Correct
it('should display an error message when the login request fails')
it('should disable the submit button while the form is submitting')

// ❌ Wrong
it('works')
it('calls authService')
```

---

## 13. Naming Conventions

| Artefact | Convention | Example |
|---|---|---|
| Component file | `kebab-case.component.tsx` | `order-card.component.tsx` |
| Hook file | `use-<concept>.hook.ts` | `use-orders.hook.ts` |
| Service file | `<entity>-api.service.ts` | `orders-api.service.ts` |
| Store (Zustand) | `<entity>.store.ts` | `orders.store.ts` |
| RTK slice | `<entity>.slice.ts` | `orders.slice.ts` |
| Schema file | `<form-name>.schema.ts` | `login.schema.ts` |
| Enum file | `<entity>-<concept>.enum.ts` | `order-status.enum.ts` |
| Interface file | `i-<entity>.interface.ts` | `i-order.interface.ts` |
| Constants file | `<domain>.constants.ts` | `api.constants.ts` |
| Mapper file | `<entity>.mapper.ts` | `order.mapper.ts` |
| CSS Module | `<component>.module.css` | `order-card.module.css` |
| Test file | `<component>.test.tsx` | `order-card.component.test.tsx` |

**Variable naming rules:**
- No single-letter variable names (except loop indices where semantically clear: `i`)
- Boolean variables: must start with `is`, `has`, `can`, or `should`
  - ✅ `isLoading`, `hasError`, `canSubmit`, `shouldRefetch`
  - ❌ `loading`, `error`, `submit`, `refetch`
- Event handlers: prefix with `handle` (e.g., `handleSubmit`, `handleSelectOrder`)
- Props that are callbacks: prefix with `on` (e.g., `onSelectOrder`, `onClose`)

---

## 14. React 19 API Reference

For `use()`, `ref` as prop, `useActionState`, and `useOptimistic`, see the table in Section 3c —
they are stable, mandatory-baseline APIs for this skill, not upcoming features.

The two items below are the only React 19 capabilities not already covered in Section 3c:

| Feature | Available from | Notes |
|---|---|---|
| React Compiler | React 19+ (opt-in) | Auto-memoises — run `react-compiler-healthcheck` first |
| Server Components | React 19 (via framework) | Use via Next.js App Router; not available in pure Vite/CRA projects |

---

## 15. Customizing This Skill

### Reference file lookup

Claude reads this SKILL.md first; open a reference file only when the task needs deeper
detail on that specific topic.

| Topic | File | When to read |
|---|---|---|
| Type system (interfaces, model classes, enums, constants, mappers) | `references/type-system.md` | When creating or reviewing API-response shapes, domain models, or mappers |
| HTTP layer | `references/http-layer.md` | When setting up or modifying the axios client, interceptors, or per-request options |
| State management | `references/state-management.md` | When choosing or implementing a state tier (Zustand, Jotai, RTK, TanStack Query, Context) |
| Testing | `references/testing.md` | When writing or reviewing unit/integration tests, mocks, or coverage for React code |
| Full examples | `references/examples.md` | When producing a complete feature or needing a production-ready end-to-end pattern |

### Project overrides

Record project-level overrides here when the team deviates from a default.

```markdown
## Project Overrides — [Project Name]

- State: Zustand only; RTK is not used in this project
- Styling: Tailwind CSS exclusively; no CSS Modules or Styled Components
- UI library: MUI v6; PrimeReact is not installed
- Testing: Jest instead of Vitest (legacy project configuration)
- Query staleTime default: 2 minutes (override from 5 minutes in section 5a)
- API base URL token: imported from environment variable `REACT_APP_API_URL`
```
