# Authentication — Full Reference

Complete implementation for all three auth approaches supported in this skill.
For the strategy decision table see the **Authentication** section in `SKILL.md`.

---

## Table of Contents
1. [Approach 1 — Auth.js (NextAuth v5)](#approach-1--authjs-nextauth-v5)
2. [Approach 2 — External Provider (Clerk, Auth0, Keycloak)](#approach-2--external-provider)
3. [Approach 3 — Custom JWT](#approach-3--custom-jwt)
4. [Shared Middleware Pattern](#shared-middleware-pattern)
5. [Session in Server Components and Actions](#session-in-server-components-and-actions)
6. [Role-based Access Control](#role-based-access-control)

---

## Approach 1 — Auth.js (NextAuth v5)

Best for: self-hosted applications, social providers (Google, GitHub), credentials login,
magic link, and any scenario where a third-party auth service is not required.

### Installation and config

```typescript
// auth.ts  (project root)

import NextAuth               from 'next-auth';
import Credentials            from 'next-auth/providers/credentials';
import Google                 from 'next-auth/providers/google';
import { z }                  from 'zod';
import { usersRepository }    from './src/lib/db/users.repository';
import { verifyPassword }     from './src/lib/auth/password';
import type { DefaultSession } from 'next-auth';

/** Extend the default Session type to include the user's role. */
declare module 'next-auth' {
  interface Session {
    user: {
      id:   string;
      role: string;
    } & DefaultSession['user'];
  }
}

const credentialsSchema = z.object({
  email:    z.string().email(),
  password: z.string().min(8),
});

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Google({
      clientId:     process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),

    Credentials({
      /**
       * Validates credentials and returns the user object on success.
       * Returning null causes a CredentialsSignin error.
       */
      authorize: async (rawCredentials) => {
        const parseResult = credentialsSchema.safeParse(rawCredentials);
        if (!parseResult.success) return null;

        const { email, password } = parseResult.data;
        const user = await usersRepository.findByEmail(email);
        if (!user) return null;

        const isPasswordValid = await verifyPassword(password, user.hashedPassword);
        if (!isPasswordValid) return null;

        return { id: user.id, email: user.email, name: user.fullName, role: user.role };
      },
    }),
  ],

  callbacks: {
    /**
     * Adds the user's id and role to the JWT token on sign-in.
     * These fields are then available in the session callback.
     */
    jwt: ({ token, user }) => {
      if (user) {
        token.id   = user.id;
        token.role = (user as { role?: string }).role ?? 'VIEWER';
      }
      return token;
    },

    /**
     * Copies id and role from the JWT into the session object
     * so Server Components can access them via auth().
     */
    session: ({ session, token }) => ({
      ...session,
      user: {
        ...session.user,
        id:   token.id as string,
        role: token.role as string,
      },
    }),
  },

  pages: {
    signIn: '/login',
    error:  '/login',
  },
});

/** Export route handler for /api/auth/[...nextauth] */
export const { GET, POST } = handlers;
```

```typescript
// app/api/auth/[...nextauth]/route.ts

export { GET, POST } from '../../../../auth';
```

### Using the session in Server Components

```typescript
// app/(dashboard)/profile/page.tsx

import { auth }   from '../../../../auth';
import { redirect } from 'next/navigation';

export default async function ProfilePage(): Promise<React.ReactElement> {
  const session = await auth();

  if (!session) {
    redirect('/login');
  }

  return (
    <main>
      <h1>Welcome, {session.user.name}</h1>
      <p>Role: {session.user.role}</p>
    </main>
  );
}
```

### Login form with signIn()

```typescript
// app/(auth)/login/actions.ts
'use server';

import { signIn }       from '../../../../auth';
import { AuthError }    from 'next-auth';
import { redirect }     from 'next/navigation';
import { APP_ROUTE_PATHS } from '../../../models/constants/app.constants';

/**
 * Server Action that signs the user in with credentials.
 * Maps NextAuth errors to user-friendly messages.
 */
export async function loginAction(
  prevState: { errorMessage?: string } | null,
  formData: FormData,
): Promise<{ errorMessage?: string }> {
  try {
    await signIn('credentials', {
      email:    formData.get('email'),
      password: formData.get('password'),
      redirect: false,
    });
  } catch (authError) {
    if (authError instanceof AuthError) {
      switch (authError.type) {
        case 'CredentialsSignin':
          return { errorMessage: 'Invalid email or password.' };
        default:
          return { errorMessage: 'An error occurred. Please try again.' };
      }
    }
    throw authError;
  }

  redirect(APP_ROUTE_PATHS.DASHBOARD);
}
```

---

## Approach 2 — External Provider

Best for: enterprise SSO (Keycloak, Okta), managed multi-tenant auth (Clerk, Auth0),
or when the auth service is pre-selected by the organization.

### Clerk (example)

```typescript
// middleware.ts

import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

const isPublicRoute = createRouteMatcher(['/login(.*)', '/register(.*)', '/api/webhooks(.*)']);

export default clerkMiddleware(async (clerkAuth, request) => {
  if (!isPublicRoute(request)) {
    await clerkAuth.protect();  // Redirects to Clerk's sign-in page if unauthenticated
  }
});

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

```typescript
// lib/auth/require-auth.ts  (Clerk version)

import { auth }     from '@clerk/nextjs/server';
import { redirect } from 'next/navigation';
import { APP_ROUTE_PATHS } from '../../models/constants/app.constants';

/**
 * Asserts authentication and returns the Clerk auth object.
 * Redirects to login if the user is unauthenticated.
 */
export async function requireAuth() {
  const clerkAuth = await auth();

  if (!clerkAuth.userId) {
    redirect(APP_ROUTE_PATHS.LOGIN);
  }

  return clerkAuth;
}
```

### Keycloak / Auth0 (OIDC via Auth.js)

Configure the OIDC provider in `auth.ts` — the session and middleware patterns are identical to Approach 1:

```typescript
// auth.ts  (Keycloak via Auth.js OIDC provider)

import Keycloak from 'next-auth/providers/keycloak';

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Keycloak({
      clientId:     process.env.KEYCLOAK_CLIENT_ID!,
      clientSecret: process.env.KEYCLOAK_CLIENT_SECRET!,
      issuer:       process.env.KEYCLOAK_ISSUER!,
    }),
  ],
  // ... callbacks same as Approach 1
});
```

---

## Approach 3 — Custom JWT

Best for: teams that manage their own auth backend, have a specific JWT format, or integrate
with an existing non-standard auth service.

```typescript
// lib/auth/jwt.ts

import { SignJWT, jwtVerify, type JWTPayload } from 'jose';

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

export interface AppJwtPayload extends JWTPayload {
  userId: string;
  role:   string;
  email:  string;
}

/**
 * Signs a new JWT with the user's identity claims.
 * Used after successful credential verification.
 */
export async function signJwt(payload: Omit<AppJwtPayload, 'iat' | 'exp'>): Promise<string> {
  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('8h')
    .sign(JWT_SECRET);
}

/**
 * Verifies a JWT and returns its payload.
 * Throws if the token is invalid or expired.
 */
export async function verifyJwt(token: string): Promise<AppJwtPayload> {
  const { payload } = await jwtVerify(token, JWT_SECRET);
  return payload as AppJwtPayload;
}
```

```typescript
// lib/auth/session.ts

import { cookies }    from 'next/headers';
import { verifyJwt, type AppJwtPayload } from './jwt';

const SESSION_COOKIE = 'auth_session';

/**
 * Reads and verifies the session JWT from the request cookies.
 * Returns null if no valid session exists.
 */
export async function getSession(): Promise<AppJwtPayload | null> {
  const cookieStore = await cookies();
  const sessionToken = cookieStore.get(SESSION_COOKIE)?.value;

  if (!sessionToken) return null;

  try {
    return await verifyJwt(sessionToken);
  } catch {
    return null;  // Token expired or invalid
  }
}
```

```typescript
// middleware.ts  (Custom JWT version)

import { NextResponse, type NextRequest } from 'next/server';
import { verifyJwt }                      from './src/lib/auth/jwt';
import { APP_ROUTE_PATHS }               from './src/models/constants/app.constants';

const PUBLIC_ROUTES = [APP_ROUTE_PATHS.LOGIN, '/api/auth/login'];

export async function middleware(request: NextRequest): Promise<NextResponse> {
  const isPublicRoute = PUBLIC_ROUTES.some((route) =>
    request.nextUrl.pathname.startsWith(route),
  );
  if (isPublicRoute) return NextResponse.next();

  const sessionToken = request.cookies.get('auth_session')?.value;

  if (!sessionToken) {
    const loginUrl = new URL(APP_ROUTE_PATHS.LOGIN, request.url);
    loginUrl.searchParams.set('callbackUrl', request.nextUrl.pathname);
    return NextResponse.redirect(loginUrl);
  }

  try {
    await verifyJwt(sessionToken);  // Throws on invalid/expired token
    return NextResponse.next();
  } catch {
    const loginUrl = new URL(APP_ROUTE_PATHS.LOGIN, request.url);
    return NextResponse.redirect(loginUrl);
  }
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

```typescript
// lib/auth/require-auth.ts  (Custom JWT version)

import { getSession }    from './session';
import { redirect }      from 'next/navigation';
import { APP_ROUTE_PATHS } from '../../models/constants/app.constants';
import type { AppJwtPayload } from './jwt';

/**
 * Returns the current session payload.
 * Redirects to /login if no valid session exists.
 */
export async function requireAuth(): Promise<AppJwtPayload> {
  const session = await getSession();
  if (!session) redirect(APP_ROUTE_PATHS.LOGIN);
  return session;
}
```

---

## Shared Middleware Pattern

All three approaches converge on the same middleware pattern:

```typescript
// middleware.ts — shared shape (fill in validateSession per approach)

import { NextResponse, type NextRequest } from 'next/server';

const PUBLIC_PATHS = ['/login', '/register', '/api/auth', '/api/webhooks'];

export async function middleware(request: NextRequest): Promise<NextResponse> {
  const isPublicPath = PUBLIC_PATHS.some((path) =>
    request.nextUrl.pathname.startsWith(path),
  );
  if (isPublicPath) return NextResponse.next();

  const isAuthenticated = await validateSession(request);   // Swap per approach

  if (!isAuthenticated) {
    const loginUrl = new URL('/login', request.url);
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

## Session in Server Components and Actions

```typescript
// In a Server Component
const session = await auth();           // Auth.js
const clerkAuth = await auth();         // Clerk
const session = await getSession();     // Custom JWT

// In a Server Action
const session = await requireAuth();    // All approaches — throws redirect if not logged in
```

---

## Role-based Access Control

```typescript
// lib/auth/require-auth.ts  (role check addition — works for all three approaches)

/**
 * Asserts that the current user has one of the allowed roles.
 * Redirects to /forbidden if the role check fails.
 *
 * @param allowedRoles - Array of roles permitted to proceed
 */
export async function requireRole(allowedRoles: string[]): Promise<void> {
  const session = await requireAuth();
  const userRole = (session as { role?: string }).role;

  if (!userRole || !allowedRoles.includes(userRole)) {
    redirect(APP_ROUTE_PATHS.FORBIDDEN);
  }
}
```

```typescript
// Usage in a Server Action or page
await requireRole(['ADMIN', 'EDITOR']);
```

```typescript
// Role-based UI in a Server Component
const session = await auth();
const userRole = session?.user.role ?? 'VIEWER';

return (
  <nav>
    {userRole === 'ADMIN' && (
      <Link href="/admin">Admin panel</Link>
    )}
  </nav>
);
```
