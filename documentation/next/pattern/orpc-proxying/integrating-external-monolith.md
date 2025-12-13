---
status: draft
priority: high
type: reference
tags: [orpc, nextjs, monolith, integration, patterns]
---

# Integrating Next.js with External ORPC Monolith

## Overview

When you have an external ORPC monolith backend, you can integrate it with your Next.js app to get direct access while maintaining control over what's exposed. You can also use Server Actions alongside ORPC for orchestration and "stitching" multiple calls together.

## Architecture Pattern

```
┌─────────────────────────────────────┐
│   External ORPC Monolith            │
│   - Full API with all procedures    │
│   - Business logic                  │
│   - Database access                 │
└──────────────┬──────────────────────┘
               │
       ┌───────┴────────┐
       │                │
   Next.js          Next.js
   Server            Client
   (Direct)          (Proxy)
       │                │
   Full Access      Filtered Access
   (All procs)      (Public only)
```

## Setup: Connecting to External Monolith

### 1. Create ORPC Client for External Monolith

```typescript
// src/lib/orpc/external-client.ts
import { createClient } from '@orpc/client';
import { RPCLink } from '@orpc/client/fetch';
import type { ExternalMonolithRouter } from '@your-monolith/types'; // Import types from monolith

// Server-side client (full access to monolith)
export const externalOrpc = createClient<ExternalMonolithRouter>(
  new RPCLink({
    url: process.env.EXTERNAL_ORPC_URL || 'http://localhost:4000/rpc',
    headers: async () => {
      // Add auth headers, service tokens, etc.
      return {
        'Authorization': `Bearer ${process.env.SERVICE_TOKEN}`,
        'X-Service-Name': 'nextjs-app',
      };
    },
  })
);

// Client-side client (filtered access - see filtering section)
export const externalOrpcClient = createClient<FilteredRouter>(
  new RPCLink({
    url: '/api/rpc', // Proxy through Next.js
  })
);
```

### 2. Create Filtered Router for Client Access

```typescript
// src/lib/orpc/filtered-router.ts
import { router } from '@orpc/server';
import type { ExternalMonolithRouter } from '@your-monolith/types';

// Define which procedures from the monolith are safe for client access
export const filteredRouter = router({
  // Public procedures only
  notes: {
    getAll: procedure().query(async () => {
      // Proxy to external monolith
      return await externalOrpc.notes.getAll();
    }),
    getById: procedure().input(z.number()).query(async ({ input }) => {
      return await externalOrpc.notes.getById(input);
    }),
    create: procedure().input(createSchema).mutation(async ({ input }) => {
      return await externalOrpc.notes.create(input);
    }),
  },
  
  // Don't expose admin procedures to client!
  // admin.deleteAllNotes - NOT EXPOSED
});

export type FilteredRouter = typeof filteredRouter;
```

### 3. Create Next.js Proxy Route

```typescript
// src/app/api/rpc/[[...rest]]/route.ts
import { RPCHandler } from '@orpc/server/fetch';
import { filteredRouter } from '@/lib/orpc/filtered-router';

const handler = new RPCHandler(filteredRouter, {
  interceptors: [
    onError((error) => {
      console.error('ORPC Proxy Error:', error);
    }),
  ],
});

async function handleRequest(request: Request) {
  const { response } = await handler.handle(request, {
    prefix: '/api/rpc',
    context: {
      // Add Next.js context (user, session, etc.)
      user: await getCurrentUser(),
    },
  });

  return response ?? new Response('Not found', { status: 404 });
}

export const POST = handleRequest;
export const GET = handleRequest;
```

## Using Server Actions for Orchestration/Stitching

Server Actions are perfect for composing multiple ORPC calls and adding Next.js-specific logic:

### Pattern: Server Actions as Orchestration Layer

```typescript
// src/lib/actions/note-actions.ts
'use server';

import { externalOrpc } from '@/lib/orpc/external-client';
import { revalidatePath } from 'next/cache';

// Simple pass-through (if you want Server Action API)
export async function getNotes() {
  const notes = await externalOrpc.notes.getAll();
  return { success: true, data: notes };
}

// Orchestration: Stitch multiple ORPC calls together
export async function getNoteWithRelated(id: number) {
  // Call multiple ORPC procedures
  const [note, relatedNotes, tags] = await Promise.all([
    externalOrpc.notes.getById(id),
    externalOrpc.notes.getRelated(id),
    externalOrpc.tags.getByNoteId(id),
  ]);
  
  // Add Next.js-specific logic
  return {
    success: true,
    data: {
      note,
      relatedNotes,
      tags,
      // Computed fields
      hasRelated: relatedNotes.length > 0,
    },
  };
}

// Orchestration: Add Next.js-specific behavior
export async function createNoteWithDefaults(formData: FormData) {
  const title = formData.get('title') as string;
  
  // Call monolith
  const note = await externalOrpc.notes.create({
    title,
    // Add Next.js-specific defaults
    createdAt: new Date(),
    userId: await getCurrentUserId(), // Next.js context
  });
  
  // Next.js-specific: Revalidate paths
  revalidatePath('/notes');
  revalidatePath(`/notes/${note.id}`);
  
  return { success: true, data: note };
}

// Orchestration: Transform data
export async function getNotesForSidebar() {
  // Get from monolith
  const notes = await externalOrpc.notes.getAll();
  
  // Transform for Next.js UI needs
  return {
    success: true,
    data: notes.map(note => ({
      id: note.id,
      title: note.title,
      preview: note.content?.substring(0, 100),
      // Add computed fields
      isRecent: isRecent(note.createdAt),
    })),
  };
}
```

### Pattern: Hybrid Approach

```typescript
// Use ORPC directly for simple operations
export default async function NotesPage() {
  const notes = await externalOrpc.notes.getAll(); // Direct ORPC call
  return <NotesList notes={notes} />;
}

// Use Server Actions for complex orchestration
export default async function NoteDetailPage({ params }: { params: { id: string } }) {
  const result = await getNoteWithRelated(parseInt(params.id)); // Server Action stitches calls
  return <NoteDetail data={result.data} />;
}
```

## Benefits of This Architecture

### 1. **Direct Access When Needed**

```typescript
// Server Components - Direct access to monolith
export default async function AdminPage() {
  // Full access to all monolith procedures
  const stats = await externalOrpc.admin.getStats();
  const users = await externalOrpc.users.getAll();
  
  return <AdminDashboard stats={stats} users={users} />;
}
```

### 2. **Filtered Access for Clients**

```typescript
// Client Components - Only safe procedures
'use client';
export default function NotesPage() {
  const { data } = useQuery({
    queryFn: () => externalOrpcClient.notes.getAll(), // Filtered router
  });
  
  // TypeScript prevents accessing admin procedures:
  // externalOrpcClient.admin.deleteAllNotes(); // ❌ Type error!
}
```

### 3. **Orchestration Layer**

Server Actions provide a perfect place to:
- Compose multiple ORPC calls
- Add Next.js-specific logic
- Transform data for UI needs
- Handle Next.js features (revalidation, etc.)

### 4. **Gradual Migration**

You can migrate incrementally:

```typescript
// Phase 1: Use Server Actions (existing code)
export async function getNotes() {
  // Old: Direct database call
  // New: Call monolith
  return await externalOrpc.notes.getAll();
}

// Phase 2: Use ORPC directly where it makes sense
export default async function NotesPage() {
  const notes = await externalOrpc.notes.getAll(); // Direct call
}

// Phase 3: Use Server Actions for orchestration
export async function getNoteWithRelated(id: number) {
  // Stitch multiple calls
}
```

## Filtering Strategies

### Strategy 1: Explicit Filtering (Recommended)

```typescript
// Explicitly define what's exposed
export const filteredRouter = router({
  notes: {
    getAll: procedure().query(async () => {
      return await externalOrpc.notes.getAll();
    }),
    // Only expose safe operations
  },
  // Don't include admin, internal, etc.
});
```

### Strategy 2: Middleware-Based Filtering

```typescript
// Filter at runtime based on context
const publicProcedure = procedure.use(async ({ ctx, next }) => {
  // Check if this is a client call
  if (ctx.request) {
    // Client call - only allow public procedures
    const procedureName = ctx.path.join('.');
    if (isAdminProcedure(procedureName)) {
      throw new Error('Unauthorized');
    }
  }
  
  return next({ ctx });
});
```

### Strategy 3: Type-Level Filtering

```typescript
// Use TypeScript to ensure only safe procedures are exposed
type PublicProcedures = Pick<
  ExternalMonolithRouter,
  'notes' | 'todos' // Only these namespaces
>;

export const filteredRouter: PublicProcedures = {
  notes: externalOrpc.notes,
  todos: externalOrpc.todos,
  // TypeScript prevents adding admin, etc.
};
```

## Error Handling and Resilience

```typescript
// src/lib/orpc/external-client.ts
import { createClient } from '@orpc/client';
import { RPCLink } from '@orpc/client/fetch';

const link = new RPCLink({
  url: process.env.EXTERNAL_ORPC_URL,
  headers: async () => ({ ... }),
  // Add retry logic, timeout, etc.
  fetch: async (url, options) => {
    try {
      const response = await fetch(url, {
        ...options,
        signal: AbortSignal.timeout(5000), // 5s timeout
      });
      
      if (!response.ok) {
        throw new Error(`ORPC call failed: ${response.statusText}`);
      }
      
      return response;
    } catch (error) {
      // Handle network errors, timeouts, etc.
      console.error('ORPC call failed:', error);
      throw error;
    }
  },
});
```

## Caching Strategy

```typescript
// Use TanStack Query for client-side caching
'use client';
export function useNotes() {
  return useQuery({
    queryKey: ['notes'],
    queryFn: () => externalOrpcClient.notes.getAll(),
    staleTime: 60 * 1000, // Cache for 1 minute
  });
}

// Server-side: Consider adding caching layer
export async function getNotes() {
  // Could add Redis cache, etc.
  return await externalOrpc.notes.getAll();
}
```

## Best Practices

1. **Always filter client access** - Never expose full monolith API to clients
2. **Use Server Actions for orchestration** - Perfect for composing calls
3. **Handle errors gracefully** - Network calls can fail
4. **Cache appropriately** - Reduce load on monolith
5. **Type safety** - Import types from monolith package
6. **Gradual migration** - Don't rewrite everything at once

## Example: Complete Integration

```typescript
// 1. External monolith client
export const externalOrpc = createClient<MonolithRouter>(...);

// 2. Filtered router for clients
export const filteredRouter = router({
  notes: { getAll, getById, create },
  // No admin procedures!
});

// 3. Server Actions for orchestration
'use server';
export async function getNoteWithRelated(id: number) {
  const [note, related] = await Promise.all([
    externalOrpc.notes.getById(id),
    externalOrpc.notes.getRelated(id),
  ]);
  return { note, related };
}

// 4. Use in components
export default async function Page() {
  const data = await getNoteWithRelated(123); // Server Action
  return <NoteDetail data={data} />;
}
```

This architecture gives you the best of both worlds: direct access to your monolith when needed, filtered access for clients, and Server Actions for Next.js-specific orchestration.

