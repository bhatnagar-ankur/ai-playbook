# Rendering Strategies — Full Reference

Complete examples for SSR, SSG, ISR, CSR, and streaming with Suspense.
For strategy selection rules see the **Rendering Strategies** section in `SKILL.md`.

---

## Table of Contents
1. [Server-Side Rendering (SSR)](#server-side-rendering-ssr)
2. [Static Site Generation (SSG)](#static-site-generation-ssg)
3. [Incremental Static Regeneration (ISR)](#incremental-static-regeneration-isr)
4. [Client-Side Rendering (CSR)](#client-side-rendering-csr)
5. [Streaming with Suspense](#streaming-with-suspense)
6. [Caching and Revalidation](#caching-and-revalidation)
7. [Mixed Strategy — Real-world Example](#mixed-strategy--real-world-example)

---

## Server-Side Rendering (SSR)

Renders fresh HTML on every request. Use when content is user-specific or changes per-request.

```typescript
// app/(dashboard)/orders/page.tsx

import { cookies }          from 'next/headers';
import { ordersRepository } from '../../../lib/db/orders.repository';
import { requireAuth }      from '../../../lib/auth/require-auth';
import { OrdersList }       from '../../../features/orders/components/orders-list/orders-list.component';

// Force dynamic rendering — no caching at route level
export const dynamic = 'force-dynamic';

/**
 * Orders list page — rendered fresh on every request.
 * Auth check prevents unauthenticated access server-side (belt-and-suspenders with middleware).
 */
export default async function OrdersPage(): Promise<React.ReactElement> {
  const session = await requireAuth();   // Redirects to /login if unauthenticated

  // fetch() with cache: 'no-store' also forces SSR when using external APIs
  const orders = await ordersRepository.findAllForUser(session.userId);

  return (
    <main>
      <h1>Your Orders</h1>
      <OrdersList initialOrders={orders} />
    </main>
  );
}
```

**External API SSR (using fetch):**

```typescript
// lib/api/orders-server.ts

/**
 * Fetches orders from the external API on the server.
 * cache: 'no-store' ensures no CDN or server-side caching — fresh per request.
 */
export async function fetchOrdersForUser(userId: string): Promise<IOrder[]> {
  const response = await fetch(
    `${process.env.API_BASE_URL}/users/${userId}/orders`,
    {
      cache:   'no-store',
      headers: {
        Authorization: `Bearer ${process.env.API_SERVICE_TOKEN}`,
        'X-User-Id':   userId,
      },
    },
  );

  if (!response.ok) {
    throw new Error(`Failed to fetch orders: ${response.statusText}`);
  }

  const rawOrders = await response.json() as Record<string, unknown>[];
  return OrderMapper.fromApiList(rawOrders);
}
```

---

## Static Site Generation (SSG)

Generates HTML at build time. Use for content that doesn't change between deployments.

```typescript
// app/products/[productId]/page.tsx

import { type Metadata }       from 'next';
import { productsRepository }  from '../../../lib/db/products.repository';
import { ProductDetail }       from '../../../features/products/components/product-detail/product-detail.component';
import { notFound }            from 'next/navigation';

interface ProductPageProps {
  params: Promise<{ productId: string }>;
}

/**
 * Generates all static product pages at build time.
 * Pages are served as static HTML — zero server compute per request.
 */
export async function generateStaticParams(): Promise<Array<{ productId: string }>> {
  const productIds = await productsRepository.findAllIds();
  return productIds.map((productId) => ({ productId }));
}

export async function generateMetadata({ params }: ProductPageProps): Promise<Metadata> {
  const { productId } = await params;
  const product = await productsRepository.findById(productId);

  return {
    title:       product?.name ?? 'Product not found',
    description: product?.description,
  };
}

export default async function ProductPage({ params }: ProductPageProps): Promise<React.ReactElement> {
  const { productId } = await params;
  const product = await productsRepository.findById(productId);

  if (!product) notFound();

  return <ProductDetail product={product} />;
}
```

---

## Incremental Static Regeneration (ISR)

Combines static delivery with automatic background regeneration. Use when content changes
infrequently (e.g. every few minutes to every hour).

### Time-based ISR

```typescript
// app/blog/[slug]/page.tsx

/**
 * Blog post page with ISR — revalidates every 3600 seconds (1 hour).
 * Serves cached HTML instantly; regenerates in the background when stale.
 */
export default async function BlogPostPage(
  { params }: { params: Promise<{ slug: string }> },
): Promise<React.ReactElement> {
  const { slug } = await params;

  // fetch() revalidate option sets the ISR interval for this request
  const response = await fetch(`${process.env.CMS_API_URL}/posts/${slug}`, {
    next: { revalidate: 3600, tags: ['blog-posts', `post-${slug}`] },
  });

  if (!response.ok) notFound();

  const post = await response.json() as IBlogPost;
  return <BlogPostView post={post} />;
}
```

### On-demand revalidation (tag-based)

```typescript
// app/api/revalidate/route.ts

import { NextResponse, type NextRequest } from 'next/server';
import { revalidateTag }                  from 'next/cache';

/**
 * POST /api/revalidate
 * Called by CMS webhooks to purge specific cache tags on content updates.
 * Protected by a secret token.
 */
export async function POST(request: NextRequest): Promise<NextResponse> {
  const { tag, secret } = await request.json() as { tag: string; secret: string };

  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 });
  }

  revalidateTag(tag);
  return NextResponse.json({ revalidated: true, tag });
}
```

### revalidatePath — after Server Action mutations

```typescript
// app/(dashboard)/orders/actions.ts
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

export async function createOrder(formData: FormData) {
  await ordersRepository.create(/* ... */);

  // Purge the orders list cache — next visit regenerates it
  revalidateTag('orders');
  revalidatePath('/orders');
}
```

---

## Client-Side Rendering (CSR)

Render in the browser using `'use client'` + TanStack Query. Use for dashboards or widgets
where data changes frequently and SSR provides no SEO benefit.

```typescript
// features/analytics/components/realtime-dashboard/realtime-dashboard.component.tsx
'use client';

import { useQuery }              from '@tanstack/react-query';
import { analyticsApiService }   from '../../services/analytics-api.service';
import { ANALYTICS_QUERY_KEYS }  from '../../constants/analytics-query-keys.constants';

/**
 * Real-time analytics dashboard — entirely client-side rendered.
 * Polls the API every 30 seconds via TanStack Query refetch interval.
 * SSR would be wasteful here since data is user-specific and changes constantly.
 */
export function RealtimeDashboard(): React.ReactElement {
  const { data: analyticsData, isLoading, isError } = useQuery({
    queryKey:       ANALYTICS_QUERY_KEYS.realtime,
    queryFn:        () => analyticsApiService.getRealtimeMetrics(),
    refetchInterval: 30_000,  // Poll every 30 seconds
    staleTime:       0,        // Always consider data stale
  });

  if (isLoading) return <DashboardSkeleton />;
  if (isError)   return <DashboardError />;

  return (
    <section aria-label="Real-time analytics">
      <MetricCard label="Active users"  value={analyticsData?.activeUsers ?? 0} />
      <MetricCard label="Orders today"  value={analyticsData?.ordersToday ?? 0} />
      <MetricCard label="Revenue today" value={analyticsData?.revenueToday ?? 0} />
    </section>
  );
}
```

**Wrapping CSR components in a page:**

```typescript
// app/(dashboard)/analytics/page.tsx  — Server Component page

import { Suspense }            from 'react';
import { RealtimeDashboard }  from '../../../features/analytics/components/realtime-dashboard/realtime-dashboard.component';
import { DashboardSkeleton }  from '../../../shared/components/dashboard-skeleton/dashboard-skeleton.component';

export const metadata = { title: 'Analytics' };

export default function AnalyticsPage(): React.ReactElement {
  return (
    <main>
      <h1>Analytics</h1>
      {/* Wrap Client Component in Suspense for the initial HTML frame */}
      <Suspense fallback={<DashboardSkeleton />}>
        <RealtimeDashboard />
      </Suspense>
    </main>
  );
}
```

---

## Streaming with Suspense

Show the page shell immediately while slow data streams in.

```typescript
// app/(dashboard)/orders/page.tsx — Streaming layout

import { Suspense }            from 'react';
import { OrdersListSkeleton }  from '../../../features/orders/components/orders-list-skeleton/orders-list-skeleton.component';
import { OrdersListServer }    from '../../../features/orders/components/orders-list-server/orders-list-server.component';
import { OrdersSummaryServer } from '../../../features/orders/components/orders-summary-server/orders-summary-server.component';

/**
 * Orders page using streaming — shell renders immediately while content fetches in parallel.
 * The <OrdersSummaryServer> and <OrdersListServer> start fetching simultaneously.
 */
export default function OrdersPage(): React.ReactElement {
  return (
    <main>
      <h1>Orders</h1>

      {/* Fast summary — resolves quickly, streams first */}
      <Suspense fallback={<SummarySkeleton />}>
        <OrdersSummaryServer />
      </Suspense>

      {/* Slow list — streams independently, does not block the summary */}
      <Suspense fallback={<OrdersListSkeleton />}>
        <OrdersListServer />
      </Suspense>
    </main>
  );
}
```

```typescript
// features/orders/components/orders-list-server/orders-list-server.component.tsx
// This is an async Server Component — it suspends while fetching

import { ordersRepository } from '../../../../lib/db/orders.repository';

/**
 * Async Server Component that fetches and renders orders.
 * Suspends with the nearest Suspense boundary while the fetch is in flight.
 */
export async function OrdersListServer(): Promise<React.ReactElement> {
  // This await suspends the component — the Suspense fallback shows in the meantime
  const orders = await ordersRepository.findAll();

  return (
    <ul aria-label="Orders list">
      {orders.map((order) => (
        <li key={order.orderId}>
          <OrderCard order={order} />
        </li>
      ))}
    </ul>
  );
}
```

**loading.tsx — automatic Suspense for route segments:**

```typescript
// app/(dashboard)/orders/loading.tsx

/**
 * Displayed automatically while the orders page Server Component fetches data.
 * Next.js wraps the page in a Suspense boundary and shows this as the fallback.
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

---

## Caching and Revalidation

### Cache function results with `unstable_cache`

```typescript
// lib/cache/orders-cache.ts

import { unstable_cache } from 'next/cache';
import { ordersRepository } from '../db/orders.repository';

/**
 * Cached version of findAll — results are memoised per-request and revalidated
 * by the 'orders' cache tag whenever an order mutation calls revalidateTag('orders').
 */
export const getCachedOrders = unstable_cache(
  async () => ordersRepository.findAll(),
  ['orders-list'],         // Cache key
  { tags: ['orders'] },   // Revalidation tags
);
```

### Per-fetch caching

```typescript
// Three explicit caching modes for fetch()

// 1. No cache — fresh on every request (SSR)
const data = await fetch(url, { cache: 'no-store' });

// 2. Time-based ISR — revalidate every N seconds
const data = await fetch(url, { next: { revalidate: 60 } });

// 3. On-demand (tag-based ISR) — purged by revalidateTag()
const data = await fetch(url, { next: { tags: ['orders'] } });

// 4. Default (force-cache) — cached indefinitely until redeploy or manual revalidation
const data = await fetch(url);
```

---

## Mixed Strategy — Real-world Example

A typical order management application combines all strategies:

| Route | Strategy | Why |
|---|---|---|
| `/` (marketing home) | SSG | Never changes per user; rebuild on deploy |
| `/products` | ISR (1h revalidate) | Changes infrequently; CDN delivery is fast |
| `/products/[id]` | SSG + ISR fallback | Pre-generate top 1000; others ISR on first hit |
| `/orders` | SSR | User-specific; must be fresh |
| `/orders/[id]` | SSR | User-specific; must be fresh |
| `/dashboard/analytics` | CSR | Real-time data; polling; no SEO needed |
| `/admin/settings` | SSR | Auth-dependent; changes per user role |
| `/login`, `/register` | SSG | Static form pages |
