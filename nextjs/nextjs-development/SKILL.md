---
name: nextjs-development
version: 1.0.0
author: Ankur Bhatnagar
description: >
  Enforces Next.js 15+ App Router best practices covering Server and Client Components,
  mixed rendering strategies (SSR / SSG+ISR / CSR), Server Actions, authentication
  (Auth.js, external providers, custom JWT), API Routes, metadata/SEO, performance,
  and testing. Inherits the React type system, HTTP layer, and state management patterns.
technology: nextjs
compatibility: Next.js >= 15, React >= 19, TypeScript >= 5.0
last_updated: 2026-09-07
tags: [nextjs, react, app-router, server-components, server-actions, authjs, tanstack-query, typescript, testing-library, playwright]
triggers:
  - Next.js App Router file (page.tsx, layout.tsx, loading.tsx, error.tsx, not-found.tsx)
  - Server Component or Client Component ('use client', 'use server')
  - Server Action (action.ts, actions/ folder, action= prop)
  - generateStaticParams, generateMetadata, revalidatePath, revalidateTag
  - next/image, next/font, next/link, next/navigation
  - useRouter, usePathname, useSearchParams, useParams
  - NextResponse, NextRequest, middleware.ts
  - Route Handler (route.ts)
  - Auth.js / NextAuth, getServerSession, auth()
  - Clerk, Keycloak, JWT middleware
  - ISR, SSG, SSR, CSR, Suspense boundary
  - cookies(), headers() from next/headers
  - cache(), unstable_cache(), fetch with revalidate
---

# Next.js Development Skill

This skill guides Claude to produce idiomatic, type-safe Next.js 15+ code using the App Router.
It covers the full spectrum: Server Components as the default, Client Components where needed,
mixed per-route rendering strategies, Server Actions for mutations, and three authentication
approaches. Follow every rule in this document unless the **Customizing** section overrides it.

The default Server Actions form pattern in this skill (`useActionState`, `useOptimistic`) requires
React 19, which ships by default from Next.js 15 onward. Next.js 14.x projects do not get React 19
by default — either upgrade to Next.js 15+, or fall back to `useState` + `startTransition` for
pending/result state instead of `useActionState`/`useOptimistic`.

For shared type system, HTTP layer, and state management patterns — see the React skill at
`reactjs/react-development/SKILL.md`. This skill extends those patterns for the Next.js context.

For complete, annotated end-to-end examples that combine multiple sections of this skill (SSR
pages with streaming, Server Action forms, ISR catalogues, Auth.js login), see `references/examples.md`.

---

## 1. When to Use This Skill

Apply when:
- Creating or editing any file in the `app/` directory (pages, layouts, route handlers, actions)
- Choosing a rendering strategy for a page (SSR, SSG, ISR, CSR)
- Writing Server Actions for form submissions or data mutations
- Setting up or modifying authentication middleware and session handling
- Configuring `next/image`, `next/font`, metadata, or SEO tags
- Writing tests for Next.js pages, Server Actions, or Route Handlers

**Do NOT use when:**
- The project uses the Pages Router (`pages/` directory, no `app/` directory present) — App
  Router conventions here (Server Components, Server Actions, route.ts handlers) do not apply
- The project is a plain React SPA with no Next.js framework — defer to `react-development`
- The project is pinned to Next.js 14.x and cannot upgrade — treat the `useActionState`/
  `useOptimistic` examples in this skill as needing the `useState` + `startTransition` fallback
  (see the compatibility note above)

---

## 2. Project Structure (App Router)

```
middleware.ts                     # Edge middleware for auth route protection (true project root — sibling of src/)
src/
├── app/                          # App Router root
│   ├── layout.tsx                # Root layout — fonts, providers, global shell
│   ├── page.tsx                  # Home page (Server Component by default)
│   ├── loading.tsx               # Root Suspense fallback
│   ├── error.tsx                 # Root error boundary ('use client')
│   ├── not-found.tsx             # 404 page
│   ├── globals.css
│   ├── (auth)/                   # Route group — shares no layout segment
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── register/
│   │       └── page.tsx
│   ├── (dashboard)/              # Route group — shares dashboard layout
│   │   ├── layout.tsx
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   └── orders/
│   │       ├── page.tsx          # Orders list (SSR)
│   │       ├── [orderId]/
│   │       │   └── page.tsx      # Order detail (SSR with generateStaticParams for known IDs)
│   │       └── loading.tsx       # Streaming skeleton for orders
│   └── api/                      # Route Handlers (REST-style endpoints)
│       └── webhooks/
│           └── route.ts
├── actions/                      # Server Actions (global; feature actions co-locate in features/)
├── components/                   # Shared components (Server Components unless marked 'use client')
├── features/                     # Feature-scoped components, hooks, stores
├── lib/                          # Server-side utilities
│   ├── auth/                     # Auth helpers (getSession, withAuth, middleware guards)
│   ├── db/                       # Database client / ORM helpers
│   └── cache/                    # Cache tag helpers, revalidation utilities
├── models/                       # Same type system as React skill
│   ├── interfaces/
│   ├── enums/
│   ├── constants/
│   └── mappers/
└── ...
```

---

## 3. Server Components vs Client Components

This is the most important architectural decision in App Router.

**Default: Server Component.** Every file in `app/` is a Server Component unless the file
starts with `'use client'`. Start as a Server Component and add `'use client'` only when needed.

| Need | Use |
|---|---|
| Data fetching from DB or API | Server Component |
| Static or dynamic HTML generation | Server Component |
| SEO metadata | Server Component |
| Access to `cookies()`, `headers()` | Server Component |
| `useState`, `useEffect`, `useReducer` | Client Component |
| Browser APIs (`window`, `document`) | Client Component |
| Event listeners (`onClick`, `onChange`) | Client Component |
| TanStack Query, Zustand, RTK | Client Component |
| `useRouter`, `usePathname`, `useSearchParams` | Client Component |
| Third-party components that use React hooks | Client Component |

**Rules:**
- Push `'use client'` as far down the tree as possible — keep parents as Server Components
- Never `'use client'` a layout unless it truly needs browser interactivity
- A Server Component can import a Client Component — but a Client Component cannot import a Server Component (the boundary is one-way)
- Pass server-fetched data as props to Client Components rather than re-fetching on the client

```typescript
// app/(dashboard)/orders/page.tsx — Server Component (no directive needed)

import { ordersRepository } from '../../../lib/db/orders.repository';
import { OrdersClient }     from '../../../features/orders/components/orders-client/orders-client.component';

/**
 * Orders page — fetches data on the server, passes it to the interactive client component.
 * Data fetch happens at request time (SSR). No client-side loading state needed for the initial load.
 */
export default async function OrdersPage(): Promise<React.ReactElement> {
  const initialOrders = await ordersRepository.findAll();

  return (
    <main>
      <h1>Orders</h1>
      {/* Pass pre-fetched data to client component for interactivity */}
      <OrdersClient initialOrders={initialOrders} />
    </main>
  );
}
```

```typescript
// features/orders/components/orders-client/orders-client.component.tsx
'use client';

import { useOrders }  from '../../hooks/use-orders.hook';
import type { IOrder } from '../../../../models/interfaces/i-order.interface';

interface OrdersClientProps {
  readonly initialOrders: IOrder[];
}

/**
 * Client Component wrapper for the orders list.
 * Receives SSR data as initialData to avoid a client-side loading flash,
 * then uses TanStack Query for background refresh and mutations.
 */
export function OrdersClient({ initialOrders }: OrdersClientProps): React.ReactElement {
  const { data: orders } = useOrders({ initialData: initialOrders });

  return (
    <ul aria-label="Orders list">
      {orders?.map((order) => (
        <li key={order.orderId}>
          <OrderCard order={order} />
        </li>
      ))}
    </ul>
  );
}
```

---

## 4. Rendering Strategies

Full examples in `references/rendering.md`. Choose per page — mixing is correct.

| Strategy | How to trigger | When to use |
|---|---|---|
| **SSR** (dynamic) | `fetch(url, { cache: 'no-store' })`, a dynamic function (`cookies()`, `headers()`), or `export const dynamic = 'force-dynamic'` | User-specific data, real-time inventory, session-dependent content |
| **SSG** | `generateStaticParams()` + no dynamic data | Marketing pages, blog posts, product catalogue with infrequent updates |
| **ISR** | `fetch(url, { next: { revalidate: N } })` or `revalidateTag()` | Content that changes occasionally (e.g. every hour) |
| **CSR** | `'use client'` + TanStack Query (no server fetch) | Highly interactive dashboards, user-specific widgets |
| **Streaming** | `Suspense` + async Server Components or `loading.tsx` | Long data fetches — show shell immediately, stream content |

```typescript
// SSR — fetch at every request (no caching)
async function getOrderById(orderId: string): Promise<IOrder> {
  const response = await fetch(`${process.env.API_BASE_URL}/orders/${orderId}`, {
    cache: 'no-store',   // Always fetch fresh — no CDN/server cache
    headers: { Authorization: `Bearer ${await getServerToken()}` },
  });
  if (!response.ok) throw new Error(`Failed to fetch order ${orderId}`);
  return OrderMapper.fromApi(await response.json());
}

// ISR — revalidate every 60 seconds via cache tag
async function getProductCatalogue(): Promise<IProduct[]> {
  const response = await fetch(`${process.env.API_BASE_URL}/products`, {
    next: { revalidate: 60, tags: ['products'] },
  });
  return ProductMapper.fromApiList(await response.json());
}

// SSG — generate static paths at build time
export async function generateStaticParams() {
  const orders = await ordersRepository.findAllIds();
  return orders.map((orderId) => ({ orderId }));
}
```

---

## 5. Server Actions

Full implementation in `references/server-actions.md`.

Server Actions are async functions marked `'use server'` that run on the server and can be
called from Client Components, `<form action={...}>` props, or event handlers.

**Use Server Actions when:** simple CRUD mutations, form submissions, no separate API needed.
**Use external API when:** complex business logic lives in .NET / Java backend.

```typescript
// app/(dashboard)/orders/actions.ts
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';
import { redirect }                      from 'next/navigation';
import { z }                             from 'zod';
import { ordersRepository }              from '../../../lib/db/orders.repository';
import { requireAuth }                   from '../../../lib/auth/require-auth';

const updateOrderStatusSchema = z.object({
  orderId: z.string().min(1),
  status:  z.enum(['PENDING', 'PROCESSING', 'SHIPPED', 'DELIVERED', 'CANCELLED']),
});

/**
 * Server Action: updates an order's status.
 * Validates input with Zod, checks auth, mutates, then revalidates relevant caches.
 *
 * @returns Object with success flag and optional error message.
 */
export async function updateOrderStatus(
  formData: FormData,
): Promise<{ isSuccess: boolean; errorMessage?: string }> {
  await requireAuth();  // Throws redirect to /login if unauthenticated

  const parseResult = updateOrderStatusSchema.safeParse({
    orderId: formData.get('orderId'),
    status:  formData.get('status'),
  });

  if (!parseResult.success) {
    return { isSuccess: false, errorMessage: parseResult.error.errors[0].message };
  }

  try {
    await ordersRepository.updateStatus(parseResult.data.orderId, parseResult.data.status);
    revalidateTag('orders');
    revalidatePath('/orders');
    return { isSuccess: true };
  } catch (actionError) {
    return {
      isSuccess:    false,
      errorMessage: actionError instanceof Error ? actionError.message : 'Update failed.',
    };
  }
}
```

**Calling a Server Action from a Client Component:**

```typescript
// features/orders/components/order-status-form/order-status-form.component.tsx
'use client';

import { useActionState }    from 'react';
import { updateOrderStatus } from '../../../../app/(dashboard)/orders/actions';

interface OrderStatusFormProps {
  readonly orderId: string;
}

/**
 * Form that submits an order status update via Server Action.
 * useActionState manages the pending/result state without manual useState.
 */
export function OrderStatusForm({ orderId }: OrderStatusFormProps): React.ReactElement {
  const [actionState, formAction, isPending] = useActionState(updateOrderStatus, null);

  return (
    <form action={formAction}>
      <input type="hidden" name="orderId" value={orderId} />

      <select name="status" aria-label="New order status" defaultValue="">
        <option value="" disabled>Select status</option>
        <option value="PROCESSING">Processing</option>
        <option value="SHIPPED">Shipped</option>
        <option value="CANCELLED">Cancelled</option>
      </select>

      {actionState && !actionState.isSuccess && (
        <p role="alert" aria-live="assertive" className="field-error">
          {actionState.errorMessage}
        </p>
      )}

      <button type="submit" disabled={isPending} aria-busy={isPending}>
        {isPending ? 'Updating…' : 'Update status'}
      </button>
    </form>
  );
}
```

---

## 6. Authentication

Full implementation in `references/auth.md`. Three approaches are supported — choose one per project.

| Approach | Best for |
|---|---|
| **Auth.js (NextAuth v5)** | Projects with social login, credentials, magic link; self-hosted |
| **External provider (Clerk, Auth0, Keycloak)** | Enterprise SSO, managed auth, multi-tenant |
| **Custom JWT** | Full control, specific token format, existing auth backend |

### middleware.ts — route protection (all three approaches share this pattern)

```typescript
// middleware.ts  (project root — runs on every matched request at the Edge)

import { NextResponse, type NextRequest } from 'next/server';
import { APP_ROUTE_PATHS }                from './src/models/constants/app.constants';

/** Routes accessible without authentication. */
const PUBLIC_ROUTES = [
  APP_ROUTE_PATHS.LOGIN,
  APP_ROUTE_PATHS.REGISTER,
  '/api/webhooks',
];

/**
 * Edge middleware: redirects unauthenticated users to /login.
 * Auth validation delegates to the active auth provider's token check.
 */
export async function middleware(request: NextRequest): Promise<NextResponse> {
  const isPublicRoute = PUBLIC_ROUTES.some((publicRoute) =>
    request.nextUrl.pathname.startsWith(publicRoute),
  );

  if (isPublicRoute) return NextResponse.next();

  // Replace with the session check for your auth approach (see references/auth.md)
  const isAuthenticated = await validateSession(request);

  if (!isAuthenticated) {
    const loginUrl = new URL(APP_ROUTE_PATHS.LOGIN, request.url);
    loginUrl.searchParams.set('callbackUrl', request.nextUrl.pathname);
    return NextResponse.redirect(loginUrl);
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.svg).*)'],
};
```

---

## 7. API Routes (Route Handlers)

Use Route Handlers for webhooks, or when exposing internal endpoints consumed by third parties.
For internal mutations, prefer Server Actions over Route Handlers.

```typescript
// app/api/webhooks/stripe/route.ts

import { type NextRequest, NextResponse } from 'next/server';

/**
 * POST /api/webhooks/stripe
 * Receives Stripe webhook events and processes them server-side.
 * Auth: verified via Stripe signature header — no session check needed.
 */
export async function POST(request: NextRequest): Promise<NextResponse> {
  const rawBody     = await request.text();
  const stripeSignature = request.headers.get('stripe-signature');

  if (!stripeSignature) {
    return NextResponse.json({ error: 'Missing signature' }, { status: 400 });
  }

  try {
    const stripeEvent = await verifyStripeWebhook(rawBody, stripeSignature);
    await processStripeEvent(stripeEvent);
    return NextResponse.json({ received: true }, { status: 200 });
  } catch (webhookError) {
    console.error('[Stripe webhook] Error:', webhookError);
    return NextResponse.json({ error: 'Webhook processing failed' }, { status: 500 });
  }
}
```

**Route Handler naming:**
- Export named functions matching HTTP methods: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`
- Never use default export in `route.ts` — the App Router dispatches requests by looking for a
  named export matching the HTTP method; a default export is never invoked and the route silently 404s
- Always type the return as `NextResponse` — an explicit return type catches missing-return code
  paths (e.g. a forgotten `return` in an `if` branch) at compile time instead of at request time

---

## 8. Metadata and SEO

```typescript
// app/(dashboard)/orders/[orderId]/page.tsx

import { type Metadata } from 'next';
import { ordersRepository } from '../../../../lib/db/orders.repository';

interface OrderDetailPageParams {
  params: Promise<{ orderId: string }>;
}

/**
 * Generates dynamic metadata for the order detail page.
 * Runs on the server — safe to fetch data here.
 */
export async function generateMetadata(
  { params }: OrderDetailPageParams,
): Promise<Metadata> {
  const { orderId } = await params;
  const order = await ordersRepository.findById(orderId);

  return {
    title:       `Order ${orderId} — ${order?.status ?? 'Details'}`,
    description: `View details for order ${orderId}`,
    robots:      { index: false, follow: false }, // Don't index order details
  };
}

export default async function OrderDetailPage(
  { params }: OrderDetailPageParams,
): Promise<React.ReactElement> {
  const { orderId } = await params;
  const order = await ordersRepository.findById(orderId);
  if (!order) notFound();

  return <OrderDetailView order={order} />;
}
```

**Root layout metadata (static):**

```typescript
// app/layout.tsx

import { type Metadata } from 'next';

export const metadata: Metadata = {
  title: {
    template: '%s | My App',
    default:  'My App',
  },
  description: 'Application description',
  openGraph: {
    type:   'website',
    locale: 'en_US',
  },
};
```

---

## 9. Images, Fonts, and Scripts

### Images — always use `next/image`

```typescript
import Image from 'next/image';

// Hero image (above the fold) — must have priority
<Image
  src="/hero-banner.jpg"
  alt="Orders dashboard overview"
  width={1200}
  height={630}
  priority              // Prevents LCP delay
/>

// Below-the-fold image — lazy loaded by default
<Image
  src={product.imageUrl}
  alt={product.name}
  width={300}
  height={300}
  sizes="(max-width: 768px) 100vw, 300px"
/>
```

**Rules:**
- Always provide `width` and `height` to prevent CLS
- Use `priority` on the largest above-the-fold image (LCP candidate)
- Use `sizes` for responsive images served from a CDN
- Never use `<img>` directly in Next.js — always `next/image`

### Fonts — use `next/font`

```typescript
// app/layout.tsx

import { Inter, Roboto_Mono } from 'next/font/google';

const inter = Inter({
  subsets:  ['latin'],
  variable: '--font-inter',
  display:  'swap',
});

const robotoMono = Roboto_Mono({
  subsets:  ['latin'],
  variable: '--font-roboto-mono',
  display:  'swap',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${robotoMono.variable}`}>
      <body>{children}</body>
    </html>
  );
}
```

---

## 10. Type System

Same conventions as the React skill — `I`-prefixed interfaces, string enums, `as const`
constants, static mapper classes. See `references/type-system.md` in the React skill.

**Next.js-specific typed page/layout props:**

```typescript
// app/(dashboard)/orders/[orderId]/page.tsx

/** Typed params for dynamic segments — always declare this interface in page files. */
interface OrderDetailPageProps {
  params:      Promise<{ orderId: string }>;       // async in Next.js 15+; sync in 14
  searchParams: Promise<Record<string, string | string[]>>;
}

export default async function OrderDetailPage(
  { params, searchParams }: OrderDetailPageProps,
): Promise<React.ReactElement> {
  const { orderId }    = await params;
  const { tab }        = await searchParams;
  ...
}
```

---

## 11. State Management in Next.js

**Server Components:** no state hooks — fetch data directly, pass as props.

**Client Components:** use the same tier system as the React skill.

**Key boundary rule:** Never pass non-serialisable values (class instances, functions, Dates)
across the Server → Client boundary as props. Pass plain objects only; instantiate models
on the client side.

```typescript
// ❌ Wrong — Date is not serialisable across the boundary
<OrderCard order={{ ...order, createdAt: new Date(order.createdAt) }} />

// ✅ Correct — pass the ISO string, convert to Date in the Client Component
<OrderCard order={order} />   // order.createdAt is string
```

---

## 12. Performance

Full patterns in `references/rendering.md`.

| Technique | When to use |
|---|---|
| `Suspense` + async Server Component | Stream slow data while shell renders immediately |
| `loading.tsx` | Automatic streaming skeleton for route segments |
| `generateStaticParams` + `revalidate` | ISR for content that doesn't need to be dynamic |
| `unstable_cache()` | Cache expensive server function results between requests |
| `next/image` with `priority` | Avoid LCP delay on hero images |
| Route Groups `(group)` | Share layouts without adding URL segments |
| Parallel routes `@slot` | Render multiple pages in the same layout simultaneously |

---

## 13. Testing

Full patterns in `references/testing.md`.

**Stack:**
- **Vitest** (unit/integration) for Server Actions, utilities, mappers, and hooks
- **@testing-library/react** for Client Component tests
- **Playwright** for end-to-end tests covering full user flows

**Coverage targets:**

| Layer | Minimum |
|---|---|
| Server Actions | 80% |
| Route Handlers | 80% |
| Server utility functions | 80% |
| Client Components | 70% |
| Middleware | 80% |
| Mappers | 90% |

---

## 14. Naming Conventions

| Artefact | Convention | Example |
|---|---|---|
| Page file | `page.tsx` | `app/orders/page.tsx` |
| Layout file | `layout.tsx` | `app/(dashboard)/layout.tsx` |
| Loading skeleton | `loading.tsx` | `app/orders/loading.tsx` |
| Error boundary | `error.tsx` | `app/orders/error.tsx` |
| Route handler | `route.ts` | `app/api/webhooks/route.ts` |
| Server Action file | `actions.ts` | `app/orders/actions.ts` |
| Shared Server Component | `kebab-case.component.tsx` | `order-detail-view.component.tsx` |
| Client Component | `kebab-case.component.tsx` + `'use client'` | `orders-client.component.tsx` |
| Server Action function | `verbNoun` camelCase | `updateOrderStatus`, `createOrder` |
| Lib utility | `kebab-case.ts` | `lib/auth/require-auth.ts` |

Boolean variable prefix rules (same as React skill): `is`, `has`, `can`, `should`.

**CSS conventions:** Never use `!important` — fix specificity with a more targeted selector or restructure the component boundary instead. For UI library theming, always use the library's theme provider (e.g. MUI `ThemeProvider`, shadcn/ui CSS variables) rather than overriding with `!important`.

---

## 15. Future APIs

| Feature | Available from | Notes |
|---|---|---|
| Async `params` / `searchParams` | Next.js 15 | Params are Promises — always `await` them |
| `'use cache'` directive | Next.js 15 (experimental) | Replaces `next: { revalidate }` fetch options |
| Partial Prerendering (PPR) | Next.js 14+ (experimental) | Static shell + dynamic Suspense holes; opt-in per route |
| `after()` API | Next.js 15 | Run work after a response is sent (analytics, cleanup) |
| React 19 Server Actions in form props | React 19 + Next.js 14+ | Stable — use `<form action={serverAction}>` |
| Turbopack (stable) | Next.js 15 | Replace webpack; run `next dev --turbopack` |

---

## 16. Customizing This Skill

```markdown
## Project Overrides — [Project Name]

- Auth: Auth.js with GitHub + Credentials providers (not external Keycloak)
- Database: Prisma with PostgreSQL (not generic repository pattern)
- Rendering: ISR with 60s revalidate as default; SSR only for session-dependent pages
- Server Actions: used for all mutations; no separate .NET backend calls from Next.js
- API base URL: NEXT_PUBLIC_API_URL environment variable
- Testing: Playwright for E2E (already configured); Vitest for unit tests
```
