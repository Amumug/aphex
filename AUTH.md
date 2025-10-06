# Authentication Architecture

This document explains how authentication works in Aphex CMS and how to swap auth providers without refactoring the entire system.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Current Auth Usage Analysis](#current-auth-usage-analysis)
3. [Centralization Recommendations](#centralization-recommendations)
4. [Current Implementation (Better Auth)](#current-implementation-better-auth)
5. [The Swap Point: AuthProvider Interface](#the-swap-point-authprovider-interface)
6. [What Changes vs What Stays](#what-changes-vs-what-stays)
7. [Migration Guide](#migration-guide)
8. [Best Practices](#best-practices)

---

## Architecture Overview

Aphex CMS separates authentication into three layers:

```
┌─────────────────────────────────────────────────────────────┐
│  CMS Core Package (@aphex/cms-core)                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ AuthProvider Interface (types.ts)                     │  │
│  │ - getSession(request): SessionAuth | null            │  │
│  │ - requireSession(request): SessionAuth               │  │
│  │ - validateApiKey(request): ApiKeyAuth | null         │  │
│  │ - requireApiKey(request): ApiKeyAuth                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ↑                                 │
└────────────────────────────┼─────────────────────────────────┘
                             │ implements
┌────────────────────────────┼─────────────────────────────────┐
│  App Layer (apps/studio)   │                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Auth Adapter (src/lib/server/auth/index.ts)          │  │
│  │ - Maps auth library to AuthProvider interface        │  │
│  │ - Handles user profile sync                          │  │
│  │ - Manages permissions/metadata                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ↑                                 │
│                            │ uses                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Auth Library (Better Auth / Clerk / Auth.js / etc.)  │  │
│  │ - Database tables (user, session, apikey)            │  │
│  │ - Login/signup flows                                 │  │
│  │ - OAuth providers                                    │  │
│  │ - Rate limiting, MFA, etc.                           │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Key Principle**: The CMS package only knows about the `AuthProvider` interface. It doesn't care about Better Auth, Clerk, Auth.js, or any specific implementation.

---

## Current Auth Usage Analysis

After scanning the codebase, here's where auth is currently used:

### ✅ Already Portable (Uses Generic Interfaces)

| File | Usage | Coupling |
|------|-------|----------|
| `packages/cms-core/src/hooks.ts` | Uses `AuthProvider` interface | ✅ None - already generic |
| `packages/cms-core/src/routes/*` | No direct auth usage | ✅ None - uses `locals.aphexCMS` |
| `src/routes/(protected)/admin/+layout.server.ts` | Uses `locals.auth` | ✅ None - generic SessionAuth type |
| `src/routes/(protected)/admin/settings/+page.server.ts` | Uses `locals.auth` | ✅ None - generic SessionAuth type |

### ⚠️ Better Auth Specific (Needs Abstraction)

| File | Usage | Coupling Level | Problem |
|------|-------|----------------|---------|
| `src/lib/server/auth/index.ts` | Implements `AuthProvider` | 🟡 Medium | **Expected** - this is the swap point |
| `src/lib/auth-client.ts` | Exports Better Auth client | 🔴 High | Client-side code imports this everywhere |
| `src/routes/login/+page.svelte` | Uses `authClient.signIn.email()` | 🔴 High | Login UI tightly coupled |
| `src/routes/(protected)/admin/+layout.svelte` | Uses `authClient.signOut()` | 🔴 High | Sign out UI tightly coupled |
| `src/routes/api/settings/api-keys/+server.ts` | Uses `auth.api.createApiKey()` directly | 🔴 High | Bypasses `AuthProvider` interface |
| `src/routes/(protected)/admin/settings/+page.server.ts` | Queries `apikey` table directly | 🟡 Medium | Couples to Better Auth schema |
| `src/hooks.server.ts` | Uses `svelteKitHandler` from Better Auth | 🟡 Medium | SvelteKit integration |
| `src/lib/server/db/auth-schema.ts` | Better Auth tables (user, session, apikey) | 🟡 Medium | **Expected** - auth owns its schema |

### 🔍 Key Findings

1. **API Key Management is NOT Abstracted**:
   - `src/routes/api/settings/api-keys/+server.ts` calls `auth.api.createApiKey()` directly
   - Queries `apikey` table using Better Auth's schema
   - **Problem**: Can't swap auth without rewriting API key routes

2. **Client-Side Auth Scattered**:
   - `authClient` imported in 3 different files
   - Each file uses Better Auth-specific methods (`signIn.email()`, `signOut()`, etc.)
   - **Problem**: UI components break when swapping auth

3. **No Auth Service Layer**:
   - Some code uses `locals.auth` (generic)
   - Some code uses `auth.api.*` (Better Auth specific)
   - Some code queries auth tables directly
   - **Problem**: Inconsistent access patterns

---

## Centralization Recommendations

To make auth truly swappable, implement these improvements:

### 1. Create Server-Side Auth Service

Wrap all auth operations behind a service layer:

```typescript
// src/lib/server/auth/service.ts
import type { SessionAuth } from '@aphex/cms-core/server';

export interface AuthService {
  // Session management
  getSession(request: Request): Promise<SessionAuth | null>;
  requireSession(request: Request): Promise<SessionAuth>;

  // API Key management
  listApiKeys(userId: string): Promise<ApiKey[]>;
  createApiKey(userId: string, data: CreateApiKeyData): Promise<ApiKeyWithSecret>;
  deleteApiKey(userId: string, keyId: string): Promise<boolean>;

  // User management (if needed)
  getCurrentUser(request: Request): Promise<User | null>;
}
```

**Current Implementation (Better Auth)**:

```typescript
// src/lib/server/auth/service.ts
import { auth } from './index';
import { db } from '$lib/server/db';
import { apikey } from '$lib/server/db/auth-schema';
import { eq } from 'drizzle-orm';

export const authService: AuthService = {
  async getSession(request) {
    const session = await auth.api.getSession({ headers: request.headers });
    // ... map to SessionAuth
  },

  async listApiKeys(userId) {
    return db.query.apikey.findMany({
      where: eq(apikey.userId, userId),
      columns: { id: true, name: true, /* ... */ }
    });
  },

  async createApiKey(userId, data) {
    return auth.api.createApiKey({
      body: { userId, name: data.name, /* ... */ }
    });
  },

  async deleteApiKey(userId, keyId) {
    const result = await db.delete(apikey)
      .where(and(eq(apikey.id, keyId), eq(apikey.userId, userId)));
    return result.rowCount > 0;
  }
};
```

**Benefits**:
- API routes import `authService` instead of `auth` directly
- All Better Auth-specific code in one file (`service.ts`)
- Swap auth = rewrite one file

**Migration**:
```typescript
// Before (coupled):
const result = await auth.api.createApiKey({ body: { ... } });

// After (abstracted):
const result = await authService.createApiKey(userId, { name, permissions });
```

### 2. Create Client-Side Auth Wrapper

Abstract client-side auth behind a generic interface:

```typescript
// src/lib/auth/client.ts
export interface AuthClient {
  signIn(email: string, password: string): Promise<{ error?: string }>;
  signUp(email: string, password: string): Promise<{ error?: string }>;
  signOut(): Promise<void>;
  useSession(): { data: SessionAuth | null; isLoading: boolean };
}
```

**Current Implementation (Better Auth)**:

```typescript
// src/lib/auth/client.ts
import { createAuthClient } from 'better-auth/svelte';

const betterAuthClient = createAuthClient({ /* ... */ });

export const authClient: AuthClient = {
  async signIn(email, password) {
    const result = await betterAuthClient.signIn.email({ email, password });
    return { error: result.error?.message };
  },

  async signUp(email, password) {
    const result = await betterAuthClient.signUp.email({ email, password });
    return { error: result.error?.message };
  },

  async signOut() {
    await betterAuthClient.signOut();
  },

  useSession() {
    return betterAuthClient.useSession();
  }
};
```

**Benefits**:
- UI components import generic `authClient`, not Better Auth client
- Swap auth = update `client.ts` implementation
- UI components unchanged

**Migration**:
```typescript
// Before (coupled):
import { authClient } from '$lib/auth-client';
await authClient.signIn.email({ email, password, callbackURL: '/admin' });

// After (abstracted):
import { authClient } from '$lib/auth/client';
await authClient.signIn(email, password);
```

### 3. Recommended File Structure

Organize auth code for easy swapping:

```
src/lib/
├── auth/
│   ├── index.ts              # Re-exports public API
│   ├── client.ts             # Client-side auth wrapper (AuthClient)
│   └── server/
│       ├── index.ts          # Server auth exports
│       ├── adapter.ts        # AuthProvider implementation
│       ├── service.ts        # AuthService implementation
│       └── better-auth/      # Better Auth specific code
│           ├── instance.ts   # Better Auth instance config
│           ├── schema.ts     # Better Auth database schema
│           └── hooks.ts      # User sync hooks
```

**Key Benefits**:
1. **All Better Auth code in one directory**: `src/lib/auth/server/better-auth/`
2. **Swap auth = replace one directory**: Delete `better-auth/`, add `clerk/` or `authjs/`
3. **Public API stays stable**: Components import from `src/lib/auth` (not `auth-client.ts`)

### 4. Implementation Priority

**Phase 1 - Server Side** (Highest Impact):
- [x] Create `AuthService` interface
- [ ] Implement `authService` using Better Auth
- [ ] Update API key routes to use `authService`
- [ ] Update settings page to use `authService`

**Phase 2 - Client Side** (UI Cleanup):
- [ ] Create `AuthClient` interface
- [ ] Implement `authClient` wrapper
- [ ] Update login page to use wrapper
- [ ] Update admin layout to use wrapper

**Phase 3 - Reorganization** (Optional):
- [ ] Move auth code to `src/lib/auth/` structure
- [ ] Isolate Better Auth code to subdirectory
- [ ] Update imports across codebase

### 5. Migration Checklist

When swapping from Better Auth to another provider:

**Server Side**:
- [ ] Implement new `AuthProvider` in `adapter.ts`
- [ ] Implement new `AuthService` in `service.ts`
- [ ] Update database schema (new auth tables)
- [ ] Update SvelteKit hook in `hooks.server.ts`
- [ ] Run database migrations

**Client Side**:
- [ ] Implement new `AuthClient` in `client.ts`
- [ ] Update login/signup UI (if using library components)
- [ ] Test session management across pages

**Testing**:
- [ ] Login/logout works
- [ ] API key CRUD operations work
- [ ] Protected routes enforce auth
- [ ] API endpoints accept both session + API key auth

---

## Current Implementation (Better Auth)

### 1. Database Schema (`src/lib/server/db/auth-schema.ts`)

Better Auth manages these tables:

```typescript
export const user = pgTable('user', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  emailVerified: boolean('email_verified'),
  image: text('image'),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at')
});

export const session = pgTable('session', {
  id: text('id').primaryKey(),
  expiresAt: timestamp('expires_at').notNull(),
  token: text('token').notNull().unique(),
  userId: text('user_id').references(() => user.id, { onDelete: 'cascade' })
});

export const apikey = pgTable('apikey', {
  id: text('id').primaryKey(),
  name: text('name'),
  key: text('key').notNull(),
  userId: text('user_id').references(() => user.id),
  rateLimitEnabled: boolean('rate_limit_enabled'),
  rateLimitMax: integer('rate_limit_max'),
  permissions: text('permissions'),
  metadata: text('metadata')
});
```

**CMS User Profile** (separate from auth):

```typescript
export const userProfiles = pgTable('cms_user_profiles', {
  userId: text('user_id').primaryKey(),
  role: text('role', { enum: ['admin', 'editor', 'viewer'] }),
  preferences: jsonb('preferences')
});
```

### 2. Better Auth Configuration (`src/lib/server/auth/index.ts`)

```typescript
import { betterAuth } from 'better-auth';
import { apiKey } from 'better-auth/plugins';

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: 'pg' }),
  emailAndPassword: { enabled: true },
  plugins: [
    apiKey({
      apiKeyHeaders: ['x-api-key'],
      rateLimit: {
        enabled: true,
        timeWindow: 1000 * 60 * 60 * 24, // 1 day
        maxRequests: 10000
      }
    })
  ],
  hooks: {
    after: createAuthMiddleware(async (ctx) => {
      // Sync user profile on signup
      if (ctx.path === '/sign-up/email' && ctx.context.user) {
        await db.insert(userProfiles).values({
          userId: ctx.context.user.id,
          role: 'editor'
        });
      }
    })
  }
});
```

### 3. AuthProvider Adapter

The adapter maps Better Auth to the CMS interface:

```typescript
export const authProvider: AuthProvider = {
  async getSession(request) {
    const session = await auth.api.getSession({ headers: request.headers });
    if (!session) return null;

    return {
      type: 'session',
      user: {
        id: session.user.id,
        email: session.user.email,
        name: session.user.name ?? undefined,
        image: session.user.image ?? undefined
      },
      session: {
        id: session.session.id,
        expiresAt: session.session.expiresAt
      }
    };
  },

  async validateApiKey(request) {
    const apiKeyHeader = request.headers.get('x-api-key');
    if (!apiKeyHeader) return null;

    const result = await auth.api.verifyApiKey({ body: { key: apiKeyHeader } });
    if (!result.valid) return null;

    return {
      type: 'api_key',
      keyId: result.key.id,
      name: result.key.name || 'Unnamed Key',
      permissions: ['read', 'write']
    };
  },

  // ... requireSession, requireApiKey
};
```

### 4. SvelteKit Hooks (`src/hooks.server.ts`)

```typescript
import { svelteKitHandler } from 'better-auth/svelte-kit';
import { createCMSHook } from '@aphex/cms-core/server';

const authHook: Handle = async ({ event, resolve }) => {
  return svelteKitHandler({ event, resolve, auth });
};

const aphexHook = createCMSHook(cmsConfig);

export const handle = sequence(authHook, aphexHook);
```

---

## The Swap Point: AuthProvider Interface

This is defined in `packages/cms-core/src/types.ts`:

```typescript
export interface AuthProvider {
  // Session auth (browser, admin UI)
  getSession(request: Request): Promise<SessionAuth | null>;
  requireSession(request: Request): Promise<SessionAuth>;

  // API key auth (programmatic access)
  validateApiKey(request: Request): Promise<ApiKeyAuth | null>;
  requireApiKey(request: Request, permission?: 'read' | 'write'): Promise<ApiKeyAuth>;
}

export interface SessionAuth {
  type: 'session';
  user: {
    id: string;
    email: string;
    name?: string;
    image?: string;
  };
  session: {
    id: string;
    expiresAt: Date;
  };
}

export interface ApiKeyAuth {
  type: 'api_key';
  keyId: string;
  name: string;
  permissions: ('read' | 'write')[];
  lastUsedAt?: Date;
}
```

**This is the contract.** As long as you implement these 4 methods, you can use any auth library.

---

## What Changes vs What Stays

### ✅ Stays the Same (Zero Changes)

- **CMS Core Package** - Doesn't know about your auth library
- **Admin UI Components** - DocumentEditor, field components, etc.
- **CMS Configuration** - `aphex.config.ts` structure
- **API Routes** - Document/asset endpoints
- **User Profile Schema** - `cms_user_profiles` table
- **Authorization Logic** - Role checks, permission enforcement

### 🔄 Changes When Swapping Auth

| What Changes | Why |
|--------------|-----|
| **Database Tables** | Each auth library has its own schema (user, session, etc.) |
| **Auth Library Code** | Import from `clerk`, `@auth/sveltekit`, etc. instead of `better-auth` |
| **AuthProvider Adapter** | Rewrite adapter to call new auth library's API |
| **SvelteKit Auth Hook** | Use new library's SvelteKit integration |
| **Login/Signup UI** | Use new library's components or API |
| **API Key Management** | Implement or use new library's API key feature |

---

## Migration Guide

### Example: Switching from Better Auth to Clerk

#### 1. Update Dependencies

```bash
pnpm remove better-auth
pnpm add @clerk/sveltekit
```

#### 2. Update Database Schema

Replace `auth-schema.ts` with Clerk's schema (or none if using Clerk's cloud DB).

#### 3. Create New Auth Adapter

```typescript
// src/lib/server/auth/index.ts
import { createClerkClient } from '@clerk/sveltekit/server';
import type { AuthProvider, SessionAuth, ApiKeyAuth } from '@aphex/cms-core/server';

const clerkClient = createClerkClient({
  secretKey: env.CLERK_SECRET_KEY
});

export const authProvider: AuthProvider = {
  async getSession(request) {
    const sessionId = request.headers.get('x-clerk-session-id');
    if (!sessionId) return null;

    const session = await clerkClient.sessions.getSession(sessionId);
    if (!session) return null;

    const user = await clerkClient.users.getUser(session.userId);

    return {
      type: 'session',
      user: {
        id: user.id,
        email: user.emailAddresses[0]?.emailAddress ?? '',
        name: `${user.firstName} ${user.lastName}`,
        image: user.imageUrl
      },
      session: {
        id: session.id,
        expiresAt: new Date(session.expireAt)
      }
    } satisfies SessionAuth;
  },

  async requireSession(request) {
    const session = await this.getSession(request);
    if (!session) throw new Error('Unauthorized');
    return session;
  },

  async validateApiKey(request) {
    const apiKey = request.headers.get('x-api-key');
    if (!apiKey) return null;

    // Implement using Clerk's metadata or custom table
    const key = await db.query.apiKeys.findFirst({
      where: eq(apiKeys.key, apiKey)
    });

    if (!key) return null;

    return {
      type: 'api_key',
      keyId: key.id,
      name: key.name,
      permissions: key.permissions
    } satisfies ApiKeyAuth;
  },

  async requireApiKey(request, permission) {
    const apiKey = await this.validateApiKey(request);
    if (!apiKey) throw new Error('Unauthorized');
    if (permission && !apiKey.permissions.includes(permission)) {
      throw new Error('Forbidden');
    }
    return apiKey;
  }
};
```

#### 4. Update SvelteKit Hooks

```typescript
// src/hooks.server.ts
import { handleClerk } from '@clerk/sveltekit/server';
import { createCMSHook } from '@aphex/cms-core/server';

const clerkHook = handleClerk();
const aphexHook = createCMSHook(cmsConfig);

export const handle = sequence(clerkHook, aphexHook);
```

#### 5. Update Login UI

Replace Better Auth login components with Clerk's:

```svelte
<script>
  import { SignIn } from '@clerk/sveltekit';
</script>

<SignIn />
```

#### 6. Update User Profile Sync

Use Clerk's webhooks to sync user profiles:

```typescript
// src/routes/api/webhooks/clerk/+server.ts
export const POST: RequestHandler = async ({ request }) => {
  const event = await request.json();

  if (event.type === 'user.created') {
    await db.insert(userProfiles).values({
      userId: event.data.id,
      role: 'editor'
    });
  }

  return new Response('OK');
};
```

### Example: Switching to Auth.js (Next Auth)

Similar pattern, implement `AuthProvider` using Auth.js APIs:

```typescript
import { getServerSession } from '@auth/sveltekit';

export const authProvider: AuthProvider = {
  async getSession(request) {
    const session = await getServerSession(authOptions);
    if (!session?.user) return null;

    return {
      type: 'session',
      user: {
        id: session.user.id,
        email: session.user.email!,
        name: session.user.name ?? undefined,
        image: session.user.image ?? undefined
      },
      session: {
        id: session.sessionToken,
        expiresAt: session.expires
      }
    };
  },
  // ... implement other methods
};
```

---

## Best Practices

### 1. Keep Auth Logic in One File

**Good:**
```
src/lib/server/auth/
  └── index.ts  (all auth logic here)
```

**Bad:**
```
src/lib/server/auth/
  ├── session.ts
  ├── api-keys.ts
  ├── permissions.ts
  └── sync.ts
```

Why? When swapping auth, you replace **one file** instead of hunting across multiple files.

### 2. Separate CMS User Data from Auth User Data

```typescript
// Auth tables (managed by auth library)
- user (id, email, password hash, etc.)
- session (id, token, expiresAt)

// CMS tables (managed by you)
- cms_user_profiles (userId, role, preferences)
```

This separation means:
- Auth library can change without touching CMS data
- CMS roles/permissions stay consistent across auth providers

### 3. Don't Leak Auth Library Types

**Bad:**
```typescript
import type { Session } from 'better-auth';

export async function getCMSUser(): Promise<Session> { /* ... */ }
```

**Good:**
```typescript
import type { SessionAuth } from '@aphex/cms-core/server';

export async function getCMSUser(): Promise<SessionAuth> { /* ... */ }
```

Why? Code outside `auth/index.ts` should only use CMS types, not auth library types.

### 4. Handle API Keys Consistently

If your new auth library doesn't support API keys, implement them yourself:

```typescript
// src/lib/server/db/schema.ts
export const apiKeys = pgTable('api_keys', {
  id: text('id').primaryKey(),
  key: text('key').notNull(),
  userId: text('user_id').notNull(),
  permissions: jsonb('permissions').$type<('read' | 'write')[]>(),
  expiresAt: timestamp('expires_at')
});

// src/lib/server/auth/index.ts
export const authProvider: AuthProvider = {
  async validateApiKey(request) {
    const key = request.headers.get('x-api-key');
    const record = await db.query.apiKeys.findFirst({
      where: eq(apiKeys.key, key)
    });
    // ... validate and return ApiKeyAuth
  }
};
```

### 5. Use Feature Flags for Gradual Migration

If you have production users, migrate gradually:

```typescript
// aphex.config.ts
const USE_CLERK = env.FEATURE_CLERK === 'true';

export default createCMSConfig({
  auth: {
    provider: USE_CLERK ? clerkAuthProvider : betterAuthProvider
  }
});
```

Test in staging with Clerk, keep Better Auth in production until ready.

---

## Testing the Swap

After implementing a new auth provider, test these scenarios:

### Session Auth
- [ ] Login with email/password
- [ ] Access protected admin routes
- [ ] Session persists across page reloads
- [ ] Logout clears session
- [ ] Expired sessions redirect to login

### API Key Auth
- [ ] Create API key from admin UI
- [ ] Make API request with `x-api-key` header
- [ ] Rate limiting works
- [ ] Permission enforcement works (read vs write)
- [ ] Expired/invalid keys are rejected

### User Profile Sync
- [ ] New user signup creates CMS profile
- [ ] User deletion cleans up CMS profile
- [ ] Roles persist correctly

### CMS Functionality
- [ ] Document CRUD operations work
- [ ] Asset upload/management works
- [ ] Permission checks work (admin vs editor vs viewer)
- [ ] API routes respect authentication

---

## Summary

**The AuthProvider interface is your swap point.** As long as you:

1. Implement the 4 methods (`getSession`, `requireSession`, `validateApiKey`, `requireApiKey`)
2. Return the correct types (`SessionAuth`, `ApiKeyAuth`)
3. Handle user profile sync

...you can swap auth providers without touching:
- CMS core package
- Admin UI components
- API routes
- Document/asset logic
- Existing CMS data

The entire auth swap happens in **one file**: `src/lib/server/auth/index.ts`.
