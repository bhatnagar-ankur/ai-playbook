# Server Actions — Full Reference

Complete patterns for Server Actions: validation, auth guards, optimistic updates, error handling.
For the overview and when-to-use rules see the **Server Actions** section in `SKILL.md`.

---

## Table of Contents
1. [File Organisation](#file-organisation)
2. [Basic Server Action with Zod Validation](#basic-server-action-with-zod-validation)
3. [Auth Guard Helper](#auth-guard-helper)
4. [useActionState — Form Submission Pattern](#useactionstate--form-submission-pattern)
5. [useOptimistic — Optimistic Updates](#useoptimistic--optimistic-updates)
6. [Programmatic Invocation (non-form)](#programmatic-invocation-non-form)
7. [File Upload Action](#file-upload-action)
8. [Error Handling Patterns](#error-handling-patterns)

---

## File Organisation

```
app/
└── (dashboard)/
    └── orders/
        ├── actions.ts           # Server Actions for the orders route
        └── page.tsx

features/
└── orders/
    └── actions/
        └── orders.actions.ts    # Feature-level actions (shared across routes)
```

**Rules:**
- Co-locate actions with the route that primarily uses them (`app/.../actions.ts`)
- Move an action to `features/<feature>/actions/` when it's called from multiple routes
- Always mark the file or each function with `'use server'`
- Every Server Action must validate input with Zod before touching the database
- Every Server Action must check authentication before any data access

---

## Basic Server Action with Zod Validation

```typescript
// app/(dashboard)/orders/actions.ts
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';
import { redirect }                      from 'next/navigation';
import { z }                             from 'zod';
import { ordersRepository }              from '../../../lib/db/orders.repository';
import { requireAuth }                   from '../../../lib/auth/require-auth';
import { APP_ROUTE_PATHS }              from '../../../models/constants/app.constants';
import { OrderStatus }                  from '../../../models/enums/order-status.enum';

// ── Zod schemas ──────────────────────────────────────────────────────────────

const createOrderSchema = z.object({
  customerId: z.string().min(1, 'Customer ID is required.'),
  items:      z.array(
    z.object({
      productId: z.string().min(1),
      quantity:  z.number().int().min(1, 'Quantity must be at least 1.'),
    }),
  ).min(1, 'At least one item is required.'),
});

const updateOrderStatusSchema = z.object({
  orderId: z.string().min(1, 'Order ID is required.'),
  status:  z.nativeEnum(OrderStatus, {
    errorMap: () => ({ message: 'Invalid order status.' }),
  }),
});

// ── Return type shared by all mutating actions ────────────────────────────────

export interface ActionResult {
  isSuccess:    boolean;
  errorMessage?: string;
  fieldErrors?:  Record<string, string[]>;
}

// ── Actions ───────────────────────────────────────────────────────────────────

/**
 * Creates a new order.
 * Validates the FormData payload with Zod before inserting into the database.
 * Revalidates the orders list cache on success.
 *
 * @param formData - Raw FormData from a <form> submission
 * @returns ActionResult indicating success or structured error details
 */
export async function createOrderAction(formData: FormData): Promise<ActionResult> {
  await requireAuth();

  const parseResult = createOrderSchema.safeParse({
    customerId: formData.get('customerId'),
    items:      JSON.parse((formData.get('items') as string) ?? '[]'),
  });

  if (!parseResult.success) {
    return {
      isSuccess:   false,
      fieldErrors: parseResult.error.flatten().fieldErrors as Record<string, string[]>,
    };
  }

  try {
    await ordersRepository.create(parseResult.data);
    revalidateTag('orders');
    revalidatePath(APP_ROUTE_PATHS.ORDERS);
    return { isSuccess: true };
  } catch (actionError) {
    return {
      isSuccess:    false,
      errorMessage: actionError instanceof Error ? actionError.message : 'Failed to create order.',
    };
  }
}

/**
 * Updates the status of an existing order.
 *
 * @param prevState - Previous action state (required by useActionState)
 * @param formData  - FormData containing orderId and new status
 */
export async function updateOrderStatusAction(
  prevState: ActionResult | null,
  formData: FormData,
): Promise<ActionResult> {
  await requireAuth();

  const parseResult = updateOrderStatusSchema.safeParse({
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
      errorMessage: actionError instanceof Error ? actionError.message : 'Status update failed.',
    };
  }
}

/**
 * Deletes an order by ID and redirects to the orders list.
 * Uses redirect() — the action does not return when successful.
 *
 * @param formData - FormData containing the orderId to delete
 */
export async function deleteOrderAction(formData: FormData): Promise<ActionResult> {
  await requireAuth();

  const orderId = formData.get('orderId') as string | null;
  if (!orderId) {
    return { isSuccess: false, errorMessage: 'Order ID is required.' };
  }

  try {
    await ordersRepository.delete(orderId);
    revalidateTag('orders');
  } catch (actionError) {
    return {
      isSuccess:    false,
      errorMessage: actionError instanceof Error ? actionError.message : 'Delete failed.',
    };
  }

  redirect(APP_ROUTE_PATHS.ORDERS);  // Never returns — must be outside try/catch
}
```

---

## Auth Guard Helper

```typescript
// lib/auth/require-auth.ts

import { auth }    from '../../../auth';
import { redirect } from 'next/navigation';
import { APP_ROUTE_PATHS } from '../../models/constants/app.constants';
import type { Session } from 'next-auth';

/**
 * Asserts that the current request is authenticated.
 * Redirects to /login if the session is absent or expired.
 * Use at the top of every Server Action and Server Component that requires auth.
 *
 * @returns The validated session object — safe to destructure after this call.
 */
export async function requireAuth(): Promise<Session> {
  const session = await auth();

  if (!session || !session.user) {
    redirect(APP_ROUTE_PATHS.LOGIN);
  }

  return session;
}

/**
 * Asserts that the current user has one of the specified roles.
 * Redirects to /forbidden if the role check fails.
 *
 * @param allowedRoles - Roles permitted to proceed
 */
export async function requireRole(allowedRoles: string[]): Promise<Session> {
  const session = await requireAuth();

  const userRole = (session.user as { role?: string }).role;
  if (!userRole || !allowedRoles.includes(userRole)) {
    redirect(APP_ROUTE_PATHS.FORBIDDEN);
  }

  return session;
}
```

---

## useActionState — Form Submission Pattern

`useActionState` (React 19) connects a Client Component form to a Server Action and manages
the pending state, eliminating the need for manual `useState` + loading flag.

```typescript
// features/orders/components/create-order-form/create-order-form.component.tsx
'use client';

import { useActionState }    from 'react';
import { useForm }           from 'react-hook-form';
import { zodResolver }       from '@hookform/resolvers/zod';
import { createOrderAction, type ActionResult } from '../../../../app/(dashboard)/orders/actions';
import { createOrderSchema, type ICreateOrderFormValues }
  from '../../../../models/validators/schemas/create-order.schema';

/**
 * Form for creating a new order via Server Action.
 * useActionState provides the server result and pending state automatically.
 * React Hook Form handles client-side validation for instant feedback.
 */
export function CreateOrderForm(): React.ReactElement {
  const [actionResult, formAction, isPending] = useActionState<ActionResult | null, FormData>(
    createOrderAction,
    null,
  );

  const {
    register,
    handleSubmit,
    formState: { errors: clientErrors },
  } = useForm<ICreateOrderFormValues>({ resolver: zodResolver(createOrderSchema) });

  /**
   * Client-side pre-validation before submitting to the Server Action.
   * Packages the validated data into FormData for the server function.
   */
  const onClientSubmit = (validatedData: ICreateOrderFormValues) => {
    const formData = new FormData();
    formData.set('customerId', validatedData.customerId);
    formData.set('items', JSON.stringify(validatedData.items));

    // Trigger the Server Action with the prepared FormData
    const form = document.querySelector('form') as HTMLFormElement;
    form.requestSubmit();
  };

  return (
    <form action={formAction} onSubmit={handleSubmit(onClientSubmit)} noValidate>

      {/* Server-level error */}
      {actionResult && !actionResult.isSuccess && actionResult.errorMessage && (
        <p role="alert" aria-live="assertive" className="form-error">
          {actionResult.errorMessage}
        </p>
      )}

      <div className="form-field">
        <label htmlFor="customerId">Customer ID</label>
        <input
          id="customerId"
          type="text"
          aria-invalid={!!clientErrors.customerId || !!actionResult?.fieldErrors?.customerId}
          aria-describedby="customerId-error"
          {...register('customerId')}
        />
        <span id="customerId-error" role="alert" aria-live="polite" className="field-error">
          {clientErrors.customerId?.message
            ?? actionResult?.fieldErrors?.customerId?.[0]}
        </span>
      </div>

      <button type="submit" disabled={isPending} aria-busy={isPending}>
        {isPending ? 'Creating…' : 'Create order'}
      </button>

    </form>
  );
}
```

---

## useOptimistic — Optimistic Updates

Apply `useOptimistic` to show the expected result immediately while the Server Action is in flight.

```typescript
// features/orders/components/order-status-toggle/order-status-toggle.component.tsx
'use client';

import { useOptimistic, useTransition }   from 'react';
import { updateOrderStatusAction }         from '../../../../app/(dashboard)/orders/actions';
import { OrderStatus }                    from '../../../../models/enums/order-status.enum';
import type { IOrder }                    from '../../../../models/interfaces/i-order.interface';

interface OrderStatusToggleProps {
  readonly order: IOrder;
}

/**
 * Toggle that optimistically updates the displayed order status before the
 * Server Action confirms the change. Reverts on failure.
 */
export function OrderStatusToggle({ order }: OrderStatusToggleProps): React.ReactElement {
  const [isPending, startTransition] = useTransition();

  const [optimisticStatus, setOptimisticStatus] = useOptimistic(
    order.status,
    (_currentStatus, newStatus: OrderStatus) => newStatus,
  );

  const handleStatusChange = (newStatus: OrderStatus) => {
    startTransition(async () => {
      // Update UI immediately — before the server responds
      setOptimisticStatus(newStatus);

      const formData = new FormData();
      formData.set('orderId', order.orderId);
      formData.set('status', newStatus);

      await updateOrderStatusAction(null, formData);
      // On failure: useOptimistic automatically reverts to order.status
    });
  };

  return (
    <div className="order-status-toggle" aria-label="Order status">
      <span className={`status-badge status-badge--${optimisticStatus.toLowerCase()}`}>
        {optimisticStatus}
        {isPending && <span className="status-badge__spinner" aria-label="Updating…" />}
      </span>

      {order.status !== OrderStatus.Delivered && order.status !== OrderStatus.Cancelled && (
        <button
          type="button"
          onClick={() => handleStatusChange(OrderStatus.Shipped)}
          disabled={isPending}
        >
          Mark as Shipped
        </button>
      )}
    </div>
  );
}
```

---

## Programmatic Invocation (non-form)

Server Actions can be called programmatically from event handlers, not just form submissions.

```typescript
// features/orders/components/order-card/order-card.component.tsx
'use client';

import { useTransition }    from 'react';
import { deleteOrderAction } from '../../../../app/(dashboard)/orders/actions';

interface OrderCardProps {
  readonly orderId: string;
}

/**
 * Delete button that invokes a Server Action programmatically on click.
 * useTransition marks the UI as pending during the server call.
 */
export function OrderCardDeleteButton({ orderId }: OrderCardProps): React.ReactElement {
  const [isPending, startTransition] = useTransition();

  const handleDelete = () => {
    startTransition(async () => {
      const formData = new FormData();
      formData.set('orderId', orderId);
      await deleteOrderAction(formData);
    });
  };

  return (
    <button
      type="button"
      onClick={handleDelete}
      disabled={isPending}
      aria-busy={isPending}
      className="btn btn--danger"
    >
      {isPending ? 'Deleting…' : 'Delete order'}
    </button>
  );
}
```

---

## File Upload Action

```typescript
// app/(dashboard)/orders/actions.ts  (addition to existing file)
'use server';

import { writeFile } from 'fs/promises';
import path          from 'path';

const uploadAttachmentSchema = z.object({
  orderId:    z.string().min(1),
  fileType:   z.string().refine(
    (fileType) => ['application/pdf', 'image/jpeg', 'image/png'].includes(fileType),
    { message: 'Only PDF, JPEG, and PNG files are allowed.' },
  ),
  fileSizeInBytes: z.number().max(5 * 1024 * 1024, 'File must be 5 MB or smaller.'),
});

/**
 * Uploads an attachment for an order.
 * Validates file type and size before writing to the filesystem.
 */
export async function uploadOrderAttachmentAction(formData: FormData): Promise<ActionResult> {
  await requireAuth();

  const uploadedFile = formData.get('file') as File | null;
  if (!uploadedFile) {
    return { isSuccess: false, errorMessage: 'No file was provided.' };
  }

  const parseResult = uploadAttachmentSchema.safeParse({
    orderId:         formData.get('orderId'),
    fileType:        uploadedFile.type,
    fileSizeInBytes: uploadedFile.size,
  });

  if (!parseResult.success) {
    return {
      isSuccess:   false,
      fieldErrors: parseResult.error.flatten().fieldErrors as Record<string, string[]>,
    };
  }

  try {
    const fileBuffer    = Buffer.from(await uploadedFile.arrayBuffer());
    const safeFilename  = `${parseResult.data.orderId}-${Date.now()}${path.extname(uploadedFile.name)}`;
    const uploadPath    = path.join(process.cwd(), 'uploads', safeFilename);

    await writeFile(uploadPath, fileBuffer);
    await ordersRepository.addAttachment(parseResult.data.orderId, safeFilename);

    revalidateTag(`order-${parseResult.data.orderId}`);
    return { isSuccess: true };
  } catch (uploadError) {
    return {
      isSuccess:    false,
      errorMessage: uploadError instanceof Error ? uploadError.message : 'Upload failed.',
    };
  }
}
```

---

## Error Handling Patterns

| Scenario | Pattern |
|---|---|
| Validation error (field-level) | Return `{ isSuccess: false, fieldErrors }` — display inline via `fieldErrors` |
| Auth failure | Call `requireAuth()` — it `redirect()`s automatically |
| Business logic error | Return `{ isSuccess: false, errorMessage }` — display as form-level alert |
| Unhandled exception | Wrap in try/catch; return `errorMessage` with a user-safe string |
| Post-mutation redirect | Call `redirect()` outside try/catch (it throws internally) |

**Never throw** from a Server Action — thrown errors surface as unhandled exceptions in the
client; instead return a typed `ActionResult` object the Client Component can inspect.

```typescript
// ❌ Wrong
export async function badAction(formData: FormData) {
  throw new Error('This surfaces as an unhandled error boundary');
}

// ✅ Correct
export async function goodAction(formData: FormData): Promise<ActionResult> {
  try {
    await doWork();
    return { isSuccess: true };
  } catch (error) {
    return { isSuccess: false, errorMessage: 'Something went wrong.' };
  }
}
```
