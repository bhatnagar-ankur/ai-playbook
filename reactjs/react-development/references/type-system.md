# Type System — Full Reference

Detailed examples for interfaces, model classes, enums, constants, and mappers.
For naming rules and folder structure see the **Type System** section in `SKILL.md`.

---

## Table of Contents
1. [Interfaces](#interfaces)
2. [Model Classes](#model-classes)
3. [Enums](#enums)
4. [Constants](#constants)
5. [Mappers](#mappers)

---

## Interfaces

File location: `src/models/interfaces/`
File naming: `i-<entity>.interface.ts`

All API-response shapes are prefixed with `I`. Use `readonly` for fields that must not
be mutated after creation. Never use `any` — prefer `unknown` and narrow in mappers.

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

/** DTO sent when creating a new user account. */
export interface ICreateUserDto {
  email:     string;
  firstName: string;
  lastName:  string;
  role:      UserRole;
}

/** Generic paginated response envelope used across all list endpoints. */
export interface IPagedResponse<TItem> {
  readonly data:       TItem[];
  readonly totalCount: number;
  readonly page:       number;
  readonly pageSize:   number;
}

/** Filter parameters accepted by the Users list endpoint. */
export interface IUserFilters {
  role?:       UserRole;
  isActive?:   boolean;
  searchTerm?: string;
}
```

```typescript
// models/interfaces/i-order.interface.ts

import { OrderStatus } from '../enums/order-status.enum';

/** Customer order as returned by the Orders REST API. */
export interface IOrder {
  readonly orderId:    string;
  readonly customerId: string;
  status:              OrderStatus;
  totalAmount:         number;
  currencyCode:        string;
  placedAt:            string;  // ISO 8601
  items:               IOrderItem[];
}

/** A single line item within an order. */
export interface IOrderItem {
  readonly productId: string;
  productName:        string;
  quantity:           number;
  unitPrice:          number;
}

/** DTO for creating a new order via POST /orders. */
export interface ICreateOrderDto {
  customerId: string;
  items:      Array<{ productId: string; quantity: number }>;
}

/** DTO for partial status update via PATCH /orders/:id. */
export interface IUpdateOrderStatusDto {
  status: OrderStatus;
}
```

**Component props interfaces** — do NOT use the `I` prefix (they are not API contracts):

```typescript
// features/orders/components/order-card/order-card.component.tsx

/** Props accepted by the OrderCard component. */
interface OrderCardProps {
  readonly order:      IOrder;
  onSelectOrder:       (orderId: string) => void;
  isHighlighted?:      boolean;
}
```

---

## Model Classes

File location: `src/models/classes/`
File naming: `<entity>.model.ts`

Use plain **interfaces** for data shapes with no behaviour.
Use **classes** only when the model needs computed properties or shared utility methods.

```typescript
// models/classes/user.model.ts

import { IUser }    from '../interfaces/i-user.interface';
import { UserRole } from '../enums/user-role.enum';

/**
 * Domain model for a user account.
 * Extends the API interface with computed display helpers reused
 * across multiple components.
 */
export class UserModel implements IUser {
  readonly id:    string;
  readonly email: string;
  firstName:      string;
  lastName:       string;
  role:           UserRole;
  isActive:       boolean;
  createdAt:      string;

  /** Parsed Date derived from the ISO 8601 createdAt string. */
  createdAtDate: Date;

  constructor(data: IUser) {
    Object.assign(this, data);
    this.createdAtDate = new Date(data.createdAt);
  }

  /** Returns the user's full display name, e.g. "Ada Lovelace". */
  get fullName(): string {
    return `${this.firstName} ${this.lastName}`;
  }

  /** Returns true if this user has administrative privileges. */
  get isAdmin(): boolean {
    return this.role === UserRole.Admin;
  }

  /** Returns the user's initials for avatar components, e.g. "AL". */
  get initials(): string {
    return `${this.firstName.charAt(0)}${this.lastName.charAt(0)}`.toUpperCase();
  }
}
```

---

## Enums

File location: `src/models/enums/`
File naming: `<entity>-<concept>.enum.ts`

Always use **string enums** unless there is a specific reason for numeric values.
String values survive JSON serialisation and are readable in logs without a lookup table.

```typescript
// models/enums/user-role.enum.ts

/** Roles that control access permissions across the application. */
export enum UserRole {
  Admin  = 'ADMIN',
  Editor = 'EDITOR',
  Viewer = 'VIEWER',
}
```

```typescript
// models/enums/order-status.enum.ts

/** Lifecycle states of a customer order. */
export enum OrderStatus {
  Pending    = 'PENDING',
  Processing = 'PROCESSING',
  Shipped    = 'SHIPPED',
  Delivered  = 'DELIVERED',
  Cancelled  = 'CANCELLED',
}
```

```typescript
// models/enums/http-error-code.enum.ts

/**
 * HTTP status codes used in the error interceptor.
 * Numeric enums are acceptable when values map to a stable numeric protocol.
 */
export enum HttpErrorCode {
  BadRequest          = 400,
  Unauthorized        = 401,
  Forbidden           = 403,
  NotFound            = 404,
  UnprocessableEntity = 422,
  InternalServerError = 500,
  ServiceUnavailable  = 503,
}
```

---

## Constants

File location: `src/models/constants/`
File naming: `<domain>.constants.ts`

Group related constants in a `const` object marked `as const`. Name objects in
`SCREAMING_SNAKE_CASE`. This preserves literal types and prevents accidental mutation.

```typescript
// models/constants/app.constants.ts

/** Application-wide pagination defaults used by all list endpoints. */
export const PAGINATION_DEFAULTS = {
  PAGE_SIZE:     25,
  MAX_PAGE_SIZE: 100,
  DEFAULT_PAGE:  1,
} as const;

/** Local-storage and session-storage key names. */
export const STORAGE_KEYS = {
  AUTH_TOKEN:   'auth_token',
  USER_PREFS:   'user_preferences',
  REDIRECT_URL: 'redirect_url',
} as const;

/** Route path strings used in navigate() calls and route definitions. */
export const APP_ROUTE_PATHS = {
  LOGIN:     '/login',
  DASHBOARD: '/dashboard',
  FORBIDDEN: '/forbidden',
  ORDERS:    '/orders',
  USERS:     '/users',
} as const;
```

```typescript
// models/constants/api.constants.ts

/** Base paths and URL builder functions for the Orders REST API. */
export const ORDERS_API = {
  BASE:   '/api/v1/orders',
  BY_ID:  (orderId: string) => `/api/v1/orders/${orderId}`,
  STATUS: (orderId: string) => `/api/v1/orders/${orderId}/status`,
  EXPORT: '/api/v1/orders/export',
} as const;

/** Base paths for the Users REST API. */
export const USERS_API = {
  BASE:  '/api/v1/users',
  BY_ID: (userId: string) => `/api/v1/users/${userId}`,
} as const;
```

**TanStack Query key constants** — centralise all query keys to prevent string duplication:

```typescript
// features/orders/constants/orders-query-keys.constants.ts

import type { IOrderFilters } from '../../../../models/interfaces/i-order.interface';

/** Centralised TanStack Query keys for the Orders feature. */
export const ORDERS_QUERY_KEYS = {
  /** Base key — used to invalidate ALL order queries at once. */
  all:      ['orders']                                        as const,
  /** Specific order by ID. */
  byId:     (orderId: string) => ['orders', orderId]         as const,
  /** Filtered list — filters object is part of the key for automatic re-fetch. */
  filtered: (filters: IOrderFilters) => ['orders', 'list', filters] as const,
} as const;
```

---

## Mappers

File location: `src/models/mappers/`
File naming: `<entity>.mapper.ts`

Mappers decouple raw API responses from typed domain models. All field aliasing,
default-value assignment, and type coercion belongs here — never inline in a hook or component.
Static methods only; never instantiate a mapper.

Standard signatures:
- `fromApi(raw)` — single API record → domain object
- `fromApiList(rawList)` — array → array
- `toApiDto(model)` — domain model → API-ready DTO

**Class vs. interface as the mapper's return type:** target a model **class** (e.g.
`UserMapper.fromApi` returning `UserModel`) when the entity needs behaviour — computed
properties or shared utility methods beyond the raw data, per the Model Classes section above.
Target the plain **interface** directly (e.g. `OrderMapper.fromApi` returning `IOrder`) when the
shape is a pure data-transfer object with no behaviour. Do not create a class just to mirror an
interface with no added methods.

```typescript
// models/mappers/user.mapper.ts

import { IUser, ICreateUserDto } from '../interfaces/i-user.interface';
import { UserModel }             from '../classes/user.model';
import { UserRole }              from '../enums/user-role.enum';

/**
 * Transforms raw Users API data into typed UserModel instances and vice versa.
 * All field renaming and defaults are defined here — not in hooks or components.
 */
export class UserMapper {
  /**
   * Converts a single raw API user record to a UserModel.
   * @param rawUser - Untyped API response object for one user
   * @returns A fully typed UserModel instance with computed properties available
   */
  static fromApi(rawUser: Record<string, unknown>): UserModel {
    const userInterface: IUser = {
      id:        rawUser['user_id']       as string,
      email:     rawUser['email_address'] as string,
      firstName: rawUser['first_name']    as string,
      lastName:  rawUser['last_name']     as string,
      role:      rawUser['role']          as UserRole,
      isActive:  (rawUser['status'] as string) === 'ACTIVE',
      createdAt: rawUser['created_at']    as string,
    };
    return new UserModel(userInterface);
  }

  /**
   * Converts an array of raw API user records to UserModel instances.
   */
  static fromApiList(rawUsers: Record<string, unknown>[]): UserModel[] {
    return rawUsers.map(UserMapper.fromApi);
  }

  /**
   * Converts a UserModel to the DTO format required when creating a user via the API.
   */
  static toApiDto(model: UserModel): ICreateUserDto {
    return {
      email:     model.email,
      firstName: model.firstName,
      lastName:  model.lastName,
      role:      model.role,
    };
  }
}
```

```typescript
// models/mappers/order.mapper.ts

import { IOrder, IOrderItem, ICreateOrderDto } from '../interfaces/i-order.interface';
import { OrderStatus }                         from '../enums/order-status.enum';

/**
 * Transforms raw Orders API data into typed IOrder objects.
 * Handles field aliasing between snake_case API keys and camelCase domain properties.
 */
export class OrderMapper {
  /**
   * Maps a single raw API order record to a typed IOrder.
   */
  static fromApi(rawOrder: Record<string, unknown>): IOrder {
    return {
      orderId:      rawOrder['order_id']     as string,
      customerId:   rawOrder['customer_id']  as string,
      status:       rawOrder['status']       as OrderStatus,
      totalAmount:  rawOrder['total_amount'] as number,
      currencyCode: (rawOrder['currency_code'] as string) ?? 'USD',
      placedAt:     rawOrder['placed_at']    as string,
      items:        (rawOrder['items'] as Record<string, unknown>[]).map(OrderMapper.mapLineItem),
    };
  }

  /**
   * Converts an array of raw API order records to typed IOrder objects.
   */
  static fromApiList(rawOrders: Record<string, unknown>[]): IOrder[] {
    return rawOrders.map(OrderMapper.fromApi);
  }

  /**
   * Maps a raw line-item record to a typed IOrderItem.
   * Private — called internally by fromApi only.
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
