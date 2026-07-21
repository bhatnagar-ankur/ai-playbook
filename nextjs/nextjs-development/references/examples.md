# Examples — Full Reference

Complete annotated examples showing the skill's conventions working together.

---

## Table of Contents
1. [SSR Orders Page with streaming](#example-1-ssr-orders-page-with-streaming)
2. [Server Action form with useActionState + Zod](#example-2-server-action-form)
3. [ISR product catalogue with on-demand revalidation](#example-3-isr-product-catalogue)
4. [Auth.js login with credentials](#example-4-authjs-login)

---

## Example 1: SSR Orders Page with Streaming

Demonstrates: App Router SSR page, async Server Component, Suspense streaming,
loading.tsx skeleton, passing data to Client Component.

**Input (what the user asks):**
> "Create an orders page that fetches orders server-side, streams a skeleton
> while loading, and passes data to a client component for interactivity."

**Expected output:**

```typescript
// app/(dashboard)/orders/loading.tsx

/**
 * Displayed automatically by Next.js as the Suspense fallback while
 * the OrdersPage Server Component is fetching data.
 */
export default function OrdersLoading(): React.ReactElement {
  return (
    <section aria-label="Loading orders" aria-busy={true}>
      <div className="orders-skeleton">
        {Array.from({ length: 5 }).map((_, skeletonIndex) => (
          <div key={skeletonIndex} className="order-card-skeleton" aria-hidden="true" />
        ))}
      </div>
    </section>
  );
}
```

```typescript
// app/(dashboard)/orders/page.tsx

import { type Metadata }        from 'next';
import { ordersRepository }     from '../../../lib/db/orders.repository';
import { requireAuth }          from '../../../lib/auth/require-auth';
import { OrdersClient }         from '../../../features/orders/components/orders-client/orders-client.component';

export const metadata: Metadata = { title: 'Orders' };

/**
 * Orders list page — SSR (fresh per request).
 * Fetches on the server and passes data to OrdersClient for interactivity.
 * Auth check is belt-and-suspenders (middleware already guards this route).
 */
export default async function OrdersPage(): Promise<React.ReactElement> {
  const session       = await requireAuth();
  const initialOrders = await ordersRepository.findAllForUser(session.user.id);

  return (
    <main>
      <h1>Orders</h1>
      <p>{initialOrders.length} orders</p>
      <OrdersClient initialOrders={initialOrders} />
    </main>
  );
}
```

```typescript
// features/orders/components/orders-client/orders-client.component.tsx
'use client';

import { useState, useCallback }  from 'react';
import { useOrders }              from '../../hooks/use-orders.hook';
import { OrderCard }              from '../order-card/order-card.component';
import type { IOrder }            from '../../../../models/interfaces/i-order.interface';

interface OrdersClientProps {
  readonly initialOrders: IOrder[];
}

/**
 * Interactive client wrapper for the orders list.
 * Receives SSR data as initialData — no loading flash on first render.
 * TanStack Query handles background refresh and mutation re-fetching.
 */
export function OrdersClient({ initialOrders }: OrdersClientProps): React.ReactElement {
  const [selectedOrderId, setSelectedOrderId] = useState<string | null>(null);

  const { data: orders = initialOrders } = useOrders({
    initialData: initialOrders,
  });

  const handleSelectOrder = useCallback(
    (orderId: string) => setSelectedOrderId(orderId),
    [],
  );

  return (
    <section>
      <ul aria-label="Orders list" className="orders-grid">
        {orders.map((order) => (
          <li key={order.orderId}>
            <OrderCard
              order={order}
              isSelected={order.orderId === selectedOrderId}
              onSelect={handleSelectOrder}
            />
          </li>
        ))}
      </ul>

      {orders.length === 0 && (
        <p className="empty-state">No orders found.</p>
      )}
    </section>
  );
}
```

---

## Example 2: Server Action Form

Demonstrates: Server Action with Zod, `useActionState`, field-level errors,
root-level error, accessible form, `isSubmitting` state.

**Input (what the user asks):**
> "Create a form that uses a Server Action to update an order's status.
> Show field errors inline, handle server errors, disable the button while submitting."

**Expected output:**

```typescript
// app/(dashboard)/orders/actions.ts
'use server';

import { revalidateTag }         from 'next/cache';
import { z }                     from 'zod';
import { ordersRepository }      from '../../../lib/db/orders.repository';
import { requireAuth }           from '../../../lib/auth/require-auth';
import { OrderStatus }           from '../../../models/enums/order-status.enum';

export interface ActionResult {
  isSuccess:    boolean;
  errorMessage?: string;
  fieldErrors?:  Record<string, string[]>;
}

const updateStatusSchema = z.object({
  orderId: z.string().min(1, 'Order ID is required.'),
  status:  z.nativeEnum(OrderStatus, {
    errorMap: () => ({ message: 'Please select a valid status.' }),
  }),
});

/**
 * Updates an order's status.
 * Validates input, checks auth, mutates, and revalidates the cache.
 *
 * @param prevState - Required by useActionState; ignored here but must be declared
 * @param formData  - FormData from the form submission
 */
export async function updateOrderStatusAction(
  prevState: ActionResult | null,
  formData: FormData,
): Promise<ActionResult> {
  await requireAuth();

  const parseResult = updateStatusSchema.safeParse({
    orderId: formData.get('orderId'),
    status:  formData.get('status'),
  });

  if (!parseResult.success) {
    return {
      isSuccess:   false,
      fieldErrors: parseResult.error.flatten().fieldErrors as Record<string, string[]>,
    };
  }

  try {
    await ordersRepository.updateStatus(parseResult.data.orderId, parseResult.data.status);
    revalidateTag('orders');
    revalidateTag(`order-${parseResult.data.orderId}`);
    return { isSuccess: true };
  } catch (actionError) {
    return {
      isSuccess:    false,
      errorMessage: actionError instanceof Error ? actionError.message : 'Update failed.',
    };
  }
}
```

```typescript
// features/orders/components/update-status-form/update-status-form.component.tsx
'use client';

import { useActionState }      from 'react';
import { updateOrderStatusAction, type ActionResult }
  from '../../../../app/(dashboard)/orders/actions';
import { OrderStatus }         from '../../../../models/enums/order-status.enum';

interface UpdateStatusFormProps {
  readonly orderId: string;
  readonly currentStatus: OrderStatus;
}

const ALLOWED_TRANSITIONS: Record<OrderStatus, OrderStatus[]> = {
  [OrderStatus.Pending]:    [OrderStatus.Processing, OrderStatus.Cancelled],
  [OrderStatus.Processing]: [OrderStatus.Shipped,    OrderStatus.Cancelled],
  [OrderStatus.Shipped]:    [OrderStatus.Delivered],
  [OrderStatus.Delivered]:  [],
  [OrderStatus.Cancelled]:  [],
};

/**
 * Form that submits an order status update via Server Action.
 * Renders only transitions valid for the current status.
 */
export function UpdateStatusForm(
  { orderId, currentStatus }: UpdateStatusFormProps,
): React.ReactElement {
  const [actionResult, formAction, isPending] = useActionState<ActionResult | null, FormData>(
    updateOrderStatusAction,
    null,
  );

  const availableStatuses = ALLOWED_TRANSITIONS[currentStatus];

  if (availableStatuses.length === 0) {
    return <p className="status-note">No further status transitions available.</p>;
  }

  return (
    <form action={formAction} aria-label="Update order status">
      <input type="hidden" name="orderId" value={orderId} />

      {/* Server-level error */}
      {actionResult && !actionResult.isSuccess && actionResult.errorMessage && (
        <p role="alert" aria-live="assertive" className="form-error">
          {actionResult.errorMessage}
        </p>
      )}

      {/* Status select */}
      <div className="form-field">
        <label htmlFor="status">New status</label>
        <select
          id="status"
          name="status"
          defaultValue=""
          aria-invalid={!!actionResult?.fieldErrors?.status}
          aria-describedby="status-error"
        >
          <option value="" disabled>Select new status</option>
          {availableStatuses.map((availableStatus) => (
            <option key={availableStatus} value={availableStatus}>
              {availableStatus.charAt(0) + availableStatus.slice(1).toLowerCase()}
            </option>
          ))}
        </select>
        <span id="status-error" role="alert" aria-live="polite" className="field-error">
          {actionResult?.fieldErrors?.status?.[0]}
        </span>
      </div>

      <button type="submit" disabled={isPending} aria-busy={isPending} className="btn btn--primary">
        {isPending ? 'Updating…' : 'Update status'}
      </button>

      {/* Success confirmation */}
      {actionResult?.isSuccess && (
        <p role="status" aria-live="polite" className="form-success">
          Status updated successfully.
        </p>
      )}
    </form>
  );
}
```

---

## Example 3: ISR Product Catalogue

Demonstrates: ISR with tag-based revalidation, SSG for top products,
`generateStaticParams`, on-demand revalidation endpoint.

**Input (what the user asks):**
> "Create a product catalogue page with ISR. Pre-generate top-100 products at build time,
> serve remaining pages on first request, and add a revalidation webhook endpoint."

**Expected output:**

```typescript
// app/products/page.tsx

import { type Metadata }       from 'next';
import { productsRepository }  from '../../lib/db/products.repository';
import { ProductGrid }         from '../../features/products/components/product-grid/product-grid.component';

export const metadata: Metadata = { title: 'Products' };

/**
 * Product catalogue — ISR with 1-hour revalidate.
 * CDN caches the page; background regeneration happens after staleTime expires.
 */
export default async function ProductsPage(): Promise<React.ReactElement> {
  const products = await productsRepository.findAllPublished();

  return (
    <main>
      <h1>Products</h1>
      <ProductGrid products={products} />
    </main>
  );
}

export const revalidate = 3600;  // Revalidate at most every hour
```

```typescript
// app/products/[productId]/page.tsx

import { notFound }            from 'next/navigation';
import { type Metadata }       from 'next';
import { productsRepository }  from '../../../lib/db/products.repository';
import { ProductDetail }       from '../../../features/products/components/product-detail/product-detail.component';

interface ProductDetailPageProps {
  params: Promise<{ productId: string }>;
}

/**
 * Generates static HTML for the top 100 products at build time.
 * Other products are rendered on first request (ISR fallback).
 */
export async function generateStaticParams(): Promise<Array<{ productId: string }>> {
  const topProductIds = await productsRepository.findTopIds(100);
  return topProductIds.map((productId) => ({ productId }));
}

export async function generateMetadata({ params }: ProductDetailPageProps): Promise<Metadata> {
  const { productId }  = await params;
  const product        = await productsRepository.findById(productId);
  return {
    title:       product?.name ?? 'Product not found',
    description: product?.description,
  };
}

export default async function ProductDetailPage(
  { params }: ProductDetailPageProps,
): Promise<React.ReactElement> {
  const { productId } = await params;

  const product = await productsRepository.findById(productId, {
    next: { revalidate: 3600, tags: ['products', `product-${productId}`] },
  });

  if (!product) notFound();

  return <ProductDetail product={product} />;
}
```

```typescript
// app/api/revalidate/route.ts

import { NextResponse, type NextRequest } from 'next/server';
import { revalidateTag }                  from 'next/cache';
import { z }                              from 'zod';

const revalidateSchema = z.object({
  secret: z.string(),
  tag:    z.string().min(1),
});

/**
 * POST /api/revalidate
 * Called by CMS webhooks to purge specific product cache tags.
 * Protected by a shared secret.
 *
 * Example: POST /api/revalidate { "secret": "...", "tag": "product-prod-001" }
 */
export async function POST(request: NextRequest): Promise<NextResponse> {
  const parseResult = revalidateSchema.safeParse(await request.json());

  if (!parseResult.success) {
    return NextResponse.json({ error: 'Invalid request body' }, { status: 400 });
  }

  const { secret, tag } = parseResult.data;

  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 });
  }

  revalidateTag(tag);
  return NextResponse.json({ revalidated: true, tag });
}
```

---

## Example 4: Auth.js Login

Demonstrates: Auth.js v5 configuration, credentials provider, login Server Action,
login form with `useActionState`, session access in a layout.

**Input (what the user asks):**
> "Set up Auth.js with email/password login. Create a login form that submits via Server Action,
> maps auth errors to friendly messages, and shows the logged-in user's name in the layout."

**Expected output (key files):**

```typescript
// auth.ts
import NextAuth    from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import { z }       from 'zod';
import { usersRepository } from './src/lib/db/users.repository';
import { verifyPassword }  from './src/lib/auth/password';

const credentialsSchema = z.object({
  email:    z.string().email(),
  password: z.string().min(8),
});

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Credentials({
      authorize: async (rawCredentials) => {
        const parseResult = credentialsSchema.safeParse(rawCredentials);
        if (!parseResult.success) return null;

        const user = await usersRepository.findByEmail(parseResult.data.email);
        if (!user) return null;

        const isPasswordValid = await verifyPassword(
          parseResult.data.password,
          user.hashedPassword,
        );
        if (!isPasswordValid) return null;

        return { id: user.id, email: user.email, name: user.fullName, role: user.role };
      },
    }),
  ],
  callbacks: {
    jwt:     ({ token, user }) => ({ ...token, ...(user && { id: user.id, role: (user as { role?: string }).role }) }),
    session: ({ session, token }) => ({ ...session, user: { ...session.user, id: token.id as string, role: token.role as string } }),
  },
  pages: { signIn: '/login' },
});

export const { GET, POST } = handlers;
```

```typescript
// app/(auth)/login/actions.ts
'use server';

import { signIn }   from '../../../../auth';
import { AuthError } from 'next-auth';
import { redirect }  from 'next/navigation';

export interface LoginResult {
  errorMessage?: string;
}

export async function loginAction(
  prevState: LoginResult | null,
  formData: FormData,
): Promise<LoginResult> {
  try {
    await signIn('credentials', {
      email:    formData.get('email'),
      password: formData.get('password'),
      redirect: false,
    });
  } catch (authError) {
    if (authError instanceof AuthError && authError.type === 'CredentialsSignin') {
      return { errorMessage: 'Invalid email or password.' };
    }
    return { errorMessage: 'An error occurred. Please try again.' };
  }

  redirect('/dashboard');
}
```

```typescript
// app/(auth)/login/page.tsx
'use client';

import { useActionState }   from 'react';
import { loginAction, type LoginResult } from './actions';

export default function LoginPage(): React.ReactElement {
  const [loginResult, formAction, isPending] = useActionState<LoginResult | null, FormData>(
    loginAction,
    null,
  );

  return (
    <main className="auth-page">
      <h1>Sign in</h1>

      <form action={formAction} aria-label="Sign in form" noValidate>
        {loginResult?.errorMessage && (
          <p role="alert" aria-live="assertive" className="form-error">
            {loginResult.errorMessage}
          </p>
        )}

        <div className="form-field">
          <label htmlFor="email">Email address</label>
          <input
            id="email"
            name="email"
            type="email"
            autoComplete="email"
            required
          />
        </div>

        <div className="form-field">
          <label htmlFor="password">Password</label>
          <input
            id="password"
            name="password"
            type="password"
            autoComplete="current-password"
            required
          />
        </div>

        <button type="submit" disabled={isPending} aria-busy={isPending} className="btn btn--primary">
          {isPending ? 'Signing in…' : 'Sign in'}
        </button>
      </form>
    </main>
  );
}
```

```typescript
// app/(dashboard)/layout.tsx

import { auth }   from '../../../auth';
import { redirect } from 'next/navigation';
import Link         from 'next/link';
import { signOut }  from '../../../auth';

/**
 * Dashboard layout — renders the navigation bar with the current user's name.
 * Protects all (dashboard) routes: redirects to /login if unauthenticated.
 */
export default async function DashboardLayout(
  { children }: { children: React.ReactNode },
): Promise<React.ReactElement> {
  const session = await auth();
  if (!session) redirect('/login');

  return (
    <div className="dashboard-shell">
      <header className="dashboard-header">
        <nav aria-label="Main navigation">
          <Link href="/dashboard">Dashboard</Link>
          <Link href="/orders">Orders</Link>
        </nav>

        <div className="dashboard-header__user">
          <span>Welcome, {session.user.name}</span>
          <form action={async () => { 'use server'; await signOut({ redirectTo: '/login' }); }}>
            <button type="submit" className="btn btn--ghost">Sign out</button>
          </form>
        </div>
      </header>

      <main className="dashboard-content">
        {children}
      </main>
    </div>
  );
}
```
