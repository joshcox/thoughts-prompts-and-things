# ORPC Proxying Through Next.js

## Overview

ORPC (Open Remote Procedure Call) can be proxied through Next.js, allowing you to use the same SDK on both server and client sides. This provides a unified API without needing to re-expose procedures for different access patterns.

## The Core Benefit: No Re-Exposing

### The Problem with Server Actions

With Server Actions, you often need to re-expose the same logic multiple times:

```typescript
// Server Action
'use server';
export async function getNotes() { ... }

// REST API (if you want external access)
export async function GET() {
  const notes = await getNotes(); // Re-exposing!
  return Response.json(notes);
}

// GraphQL resolver (if you add GraphQL)
export const resolvers = {
  Query: {
    notes: () => getNotes(), // Re-exposing again!
  }
};
```

### The ORPC Solution

With ORPC, you define procedures once and use them everywhere:

```typescript
// Define once
export const notesRouter = router({
  getAll: procedure().query(async () => { ... }),
});

// Use everywhere:
// ✅ Server Components (direct call, no HTTP)
// ✅ Client Components (HTTP proxy)
// ✅ REST API (same router, different adapter)
// ✅ External services (same router)
// ✅ Mobile apps (same router)
```

## Setup: Proxying ORPC Through Next.js

### 1. Create the ORPC Route Handler (Proxy)

```typescript
// src/app/rpc/[[...rest]]/route.ts
import { RPCHandler } from '@orpc/server/fetch';
import { router } from '@/lib/orpc/router'; // Your ORPC router
import { onError } from '@orpc/server';

const handler = new RPCHandler(router, {
  interceptors: [
    onError((error) => {
      console.error('ORPC Error:', error);
    }),
  ],
});

async function handleRequest(request: Request) {
  const { response } = await handler.handle(request, {
    prefix: '/rpc',
    context: {}, // You can add auth, user context, etc.
  });

  return response ?? new Response('Not found', { status: 404 });
}

// Handle all HTTP methods
export const HEAD = handleRequest;
export const GET = handleRequest;
export const POST = handleRequest;
export const PUT = handleRequest;
export const PATCH = handleRequest;
export const DELETE = handleRequest;
```

### 2. Create a Universal Client SDK

```typescript
// src/lib/orpc/client.ts
import { RPCLink } from '@orpc/client/fetch';
import { createClient } from '@orpc/client';
import { router } from './router'; // Same router type

const link = new RPCLink({
  url: typeof window !== 'undefined' 
    ? `${window.location.origin}/rpc`  // Client-side
    : `${process.env.NEXT_PUBLIC_URL || 'http://localhost:3000'}/rpc`, // Server-side
  headers: async () => {
    if (typeof window !== 'undefined') {
      return {}; // Browser - no headers needed
    }
    
    // Server-side - forward Next.js headers
    const { headers } = await import('next/headers');
    return await headers();
  },
});

// Same SDK used everywhere!
export const orpc = createClient<typeof router>(link);
```

### 3. Use the Same SDK Everywhere

```typescript
// Server Component
export default async function ServerPage() {
  const notes = await orpc.notes.getAll(); // ✅ Works!
  return <NotesList notes={notes} />;
}

// Client Component
'use client';
export default function ClientPage() {
  const { data: notes } = useQuery({
    queryKey: ['notes'],
    queryFn: () => orpc.notes.getAll(), // ✅ Same SDK!
  });
  
  return <NotesList notes={notes} />;
}
```

## Server-to-Server Optimization

When ORPC detects it's being called from the same Next.js server, it can bypass HTTP and make direct function calls:

```typescript
// Server Component - Direct call (no HTTP!)
export default async function ServerPage() {
  const notes = await orpc.notes.getAll(); 
  // Next.js can optimize this - same process, no network overhead
}

// Client Component - HTTP call (through proxy)
'use client';
export default function ClientPage() {
  const { data } = useQuery({
    queryFn: () => orpc.notes.getAll(), // HTTP call to /rpc
  });
}
```

## Architecture Diagram

```
┌─────────────────────────────────────┐
│   Your Business Logic (ORPC Router) │
│   - notes.getAll()                 │
│   - notes.getById()                │
│   - notes.create()                 │
└──────────────┬──────────────────────┘
               │
       ┌───────┴────────┐
       │                │
   Server          Client
   (Direct)        (HTTP Proxy)
       │                │
   No HTTP         /rpc endpoint
   Overhead        (Next.js Route)
```

## Benefits

1. **Unified API** - Same SDK server/client/external
2. **Type Safety** - End-to-end TypeScript types
3. **No Re-Exposing** - Define once, use everywhere
4. **Future Flexibility** - Easy to add external clients
5. **Server Optimization** - Direct calls when server-to-server

## What ORPC Doesn't Handle

**Important:** ORPC handles the API layer, not state management or behavior:

- ✅ API definition (procedures/routes)
- ✅ Request/response serialization
- ✅ Type safety across the wire
- ✅ HTTP transport (or direct calls server-side)

- ❌ React Query state/caching
- ❌ Form state management
- ❌ UI state
- ❌ Routing logic
- ❌ Optimistic updates
- ❌ Loading/error states

You still need all your hooks, TanStack Query, form management, etc. ORPC just replaces the API call layer.

