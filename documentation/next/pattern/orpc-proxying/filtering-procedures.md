# Filtering ORPC Procedures

## Overview

You can filter which ORPC procedures are exposed to different clients. This allows you to:
- Keep sensitive operations server-only
- Create public APIs with limited access
- Maintain type safety while restricting access

## Approach 1: Separate Routers (Recommended)

Create different routers for different clients:

```typescript
// src/lib/orpc/procedures/internal.ts
export const internalRouter = router({
  // Admin-only, server-only procedures
  deleteAllNotes: procedure()
    .handler(async () => {
      // Dangerous operation - server only!
    }),
  
  runMigrations: procedure()
    .handler(async () => {
      // Server-only operation
    }),
});

// src/lib/orpc/procedures/public.ts
export const publicRouter = router({
  // Safe procedures for clients
  notes: {
    getAll: procedure().query(async () => { ... }),
    getById: procedure().input(z.number()).query(async ({ input }) => { ... }),
    create: procedure().input(createSchema).mutation(async ({ input }) => { ... }),
  },
});

// src/lib/orpc/router.ts
import { router } from '@orpc/server';
import { publicRouter } from './procedures/public';
import { internalRouter } from './procedures/internal';

// Full router (server-side only)
export const fullRouter = router({
  ...publicRouter,
  ...internalRouter,
});

// Client router (exposed via HTTP)
export const clientRouter = publicRouter; // Only public procedures!
```

Then in your Next.js route handler:

```typescript
// src/app/rpc/[[...rest]]/route.ts
import { RPCHandler } from '@orpc/server/fetch';
import { clientRouter } from '@/lib/orpc/router'; // Only expose client router!

const handler = new RPCHandler(clientRouter, {
  // ... config
});

export const POST = async (req: Request) => {
  const { response } = await handler.handle(req, {
    prefix: '/rpc',
  });
  return response ?? new Response('Not found', { status: 404 });
};
```

## Approach 2: Context-Based Filtering

Filter based on authentication/context:

```typescript
// src/lib/orpc/router.ts
import { router } from '@orpc/server';
import { procedure } from '@orpc/server';

// Base procedure with auth check
const protectedProcedure = procedure.use(async ({ ctx, next }) => {
  // Check if this is a server-side call (no HTTP)
  if (!ctx.request) {
    // Server-side direct call - allow everything
    return next({ ctx });
  }
  
  // Client-side HTTP call - check auth
  const token = ctx.request.headers.get('authorization');
  if (!token || !isValidToken(token)) {
    throw new Error('Unauthorized');
  }
  
  return next({ ctx: { ...ctx, user: getUserFromToken(token) } });
});

const adminProcedure = protectedProcedure.use(async ({ ctx, next }) => {
  if (ctx.user?.role !== 'admin') {
    throw new Error('Admin only');
  }
  return next({ ctx });
});

export const appRouter = router({
  // Public procedures
  notes: {
    getAll: procedure().query(async () => { ... }),
  },
  
  // Protected procedures (require auth)
  admin: {
    deleteAllNotes: protectedProcedure.mutation(async () => { ... }),
    
    // Admin-only procedures
    runMigrations: adminProcedure.mutation(async () => { ... }),
  },
});
```

## Approach 3: Environment-Based Filtering

Filter based on where it's being called:

```typescript
// src/lib/orpc/client.ts
import { createClient } from '@orpc/client';
import { RPCLink } from '@orpc/client/fetch';
import { fullRouter } from './router'; // Full router type
import { clientRouter } from './router'; // Client router type

// Server-side client (can access everything)
export const serverOrpc = createClient<typeof fullRouter>(
  new RPCLink({
    url: process.env.INTERNAL_RPC_URL || 'direct',
    // Can bypass HTTP and call directly
  })
);

// Client-side client (limited access)
export const clientOrpc = createClient<typeof clientRouter>(
  new RPCLink({
    url: '/rpc',
  })
);

// Universal client that switches based on environment
export const orpc = typeof window !== 'undefined' 
  ? clientOrpc  // Browser - limited router
  : serverOrpc; // Server - full router
```

## Approach 4: Procedure-Level Filtering

Add metadata to procedures and filter at runtime:

```typescript
// src/lib/orpc/procedures/index.ts
import { procedure } from '@orpc/server';

// Helper to mark procedures
const serverOnly = <T extends Procedure>(proc: T) => {
  return proc.meta({ serverOnly: true });
};

export const notesRouter = router({
  // Public procedures
  getAll: procedure().query(async () => { ... }),
  
  // Server-only (filtered out for client)
  deleteAllNotes: serverOnly(
    procedure().mutation(async () => { ... })
  ),
});

// Filter router for client
export function createClientRouter(baseRouter: typeof notesRouter) {
  // Filter out serverOnly procedures
  const filtered = Object.entries(baseRouter).reduce((acc, [key, proc]) => {
    if (proc.meta?.serverOnly) {
      return acc; // Skip server-only
    }
    acc[key] = proc;
    return acc;
  }, {} as typeof baseRouter);
  
  return router(filtered);
}
```

## Recommended Pattern

For most apps, use **Approach 1** (separate routers):

```typescript
// src/lib/orpc/procedures/public.ts
export const publicRouter = router({
  notes: {
    getAll: procedure().query(async () => {
      const result = await getNotes();
      return result.data;
    }),
    getById: procedure().input(z.number()).query(async ({ input }) => {
      const result = await getNote(input);
      return result.data;
    }),
    create: procedure().input(createSchema).mutation(async ({ input }) => {
      // ... safe operations
    }),
  },
});

// src/lib/orpc/procedures/internal.ts
export const internalRouter = router({
  // Server-only operations
  admin: {
    deleteAllNotes: procedure().mutation(async () => {
      // Dangerous - server only!
    }),
  },
});

// src/lib/orpc/router.ts
export const clientRouter = publicRouter; // Only public procedures
export const fullRouter = router({
  ...publicRouter,
  ...internalRouter,
});

// src/app/rpc/[[...rest]]/route.ts
import { clientRouter } from '@/lib/orpc/router'; // Only expose client router
const handler = new RPCHandler(clientRouter);
```

## Benefits

1. **Type Safety** - TypeScript knows what's available where
2. **Clear Separation** - Easy to see what's public vs internal
3. **Security** - Internal procedures never exposed to HTTP
4. **Flexibility** - Can still use full router server-side

## Usage in Components

```typescript
// Server Component - can use full router
export default async function AdminPage() {
  const result = await serverOrpc.admin.deleteAllNotes(); // ✅ Works
}

// Client Component - only public router
'use client';
export default function NotesPage() {
  const { data } = useQuery({
    queryFn: () => clientOrpc.notes.getAll(), // ✅ Works
  });
  
  // This would be a TypeScript error:
  // clientOrpc.admin.deleteAllNotes(); // ❌ Not in client router type!
}
```

