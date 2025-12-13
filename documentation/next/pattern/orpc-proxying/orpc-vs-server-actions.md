---
status: draft
priority: high
type: decision
tags: [orpc, nextjs, server-actions, decision, comparison]
---

# ORPC vs Server Actions

## Comparison

### Server Actions (Current Approach)

**What it is:**
- Next.js-native feature
- Functions marked with `'use server'`
- Automatically creates HTTP endpoints
- Direct function calls from client components

**Example:**
```typescript
// Server Action
'use server';
export async function getNotes() {
  const notes = await noteModel.findAll();
  return { success: true, data: notes };
}

// Client Component
'use client';
const result = await getNotes(); // Direct call
```

### ORPC

**What it is:**
- RPC framework that can be proxied through Next.js
- Define procedures in a router
- Same SDK works server and client
- Can bypass HTTP server-to-server

**Example:**
```typescript
// ORPC Router
export const notesRouter = router({
  getAll: procedure().query(async () => {
    const notes = await noteModel.findAll();
    return notes;
  }),
});

// Client Component
'use client';
const notes = await orpc.notes.getAll(); // HTTP call
```

## Key Differences

| Feature | Server Actions | ORPC |
|---------|---------------|------|
| **Setup Complexity** | Simple (built-in) | More setup required |
| **Type Safety** | Good | Excellent (end-to-end) |
| **External API** | Need to re-expose | Same router, different adapter |
| **Server-to-Server** | HTTP call | Can bypass HTTP |
| **Mobile Apps** | Need new API | Same router |
| **Re-Exposing** | Yes (for different access patterns) | No (define once) |
| **Next.js Native** | ✅ Yes | ⚠️ Requires setup |

## When to Use Server Actions

**Use Server Actions if:**
- ✅ Building an internal Next.js app
- ✅ Want the simplest Next.js-native solution
- ✅ Don't need external API access
- ✅ Prefer less boilerplate
- ✅ Single Next.js application

**Example Use Case:**
- Internal admin dashboard
- Personal productivity app (like your todo app)
- Simple CRUD applications

## When to Use ORPC

**Use ORPC if:**
- ✅ Need a public API for external clients
- ✅ Want the same SDK for mobile apps
- ✅ Prefer RPC patterns
- ✅ Need streaming/subscriptions
- ✅ Building microservices architecture
- ✅ Want to avoid re-exposing logic
- ✅ **Have an external ORPC monolith** - Need direct (but filtered) access from Next.js

**Example Use Case:**
- SaaS platform with mobile apps
- Public API for third-party integrations
- Multi-service architecture
- Real-time features (streaming, subscriptions)
- **Next.js frontend connecting to ORPC monolith backend** - Direct access server-side, filtered proxy for clients

### Special Case: External ORPC Monolith

If you have an external ORPC monolith backend, ORPC is ideal because:

1. **Direct Access** - Server Components can call monolith procedures directly (no HTTP overhead)
2. **Filtered Access** - Create a filtered router for client-side access (security)
3. **Type Safety** - Import types from monolith package for end-to-end type safety
4. **Orchestration** - Use Server Actions alongside ORPC to stitch multiple monolith calls together

**Architecture:**
```
External ORPC Monolith
    ↓ (direct access)
Next.js Server Components (full access)
    ↓ (filtered proxy)
Next.js Client Components (public procedures only)
```

See `integrating-external-monolith.md` for detailed setup instructions.

## The Re-Exposing Problem

### With Server Actions

If you want to:
1. Use in Server Components ✅ (direct)
2. Use in Client Components ✅ (HTTP via Server Action endpoint)
3. Expose as REST API ❌ (must create new route handlers)
4. Use in another service ❌ (must create new API)

You end up re-exposing the same logic:

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

### With ORPC

Define once, use everywhere:

```typescript
// Define once
export const notesRouter = router({
  getAll: procedure().query(async () => { ... }),
});

// Use everywhere:
// ✅ Server Components (direct call)
// ✅ Client Components (HTTP proxy)
// ✅ REST API (same router, different adapter)
// ✅ External services (same router)
// ✅ Mobile apps (same router)
```

## State Management Note

**Important:** Both approaches require the same state management layer:

```typescript
// With Server Actions
export function useNotesPage() {
  const { data } = useQuery({
    queryFn: async () => {
      const result = await getNotes(); // API call
      return result.data;
    }
  });
  // ... all your state/behavior logic
}

// With ORPC - SAME!
export function useNotesPage() {
  const { data } = useQuery({
    queryFn: () => orpc.notes.getAll(), // Different API call, same pattern
  });
  // ... all your state/behavior logic - UNCHANGED
}
```

ORPC doesn't reduce state management work - it just changes the API layer.

## Recommendation for Your Todo App

**Current State:** Using Server Actions

**Recommendation:** Stick with Server Actions unless:
- You're adding a mobile app
- You need a public API
- You want to share code with other services
- You're building a microservices architecture
- **You have an external ORPC monolith** - ORPC provides direct access with filtering

**Why:** Server Actions are simpler, more Next.js-native, and sufficient for an internal app. The benefits of ORPC (unified API, no re-exposing) only matter if you need multiple access patterns or are integrating with an existing ORPC backend.

## Migration Path

If you decide to migrate to ORPC later:

1. **Phase 1:** Keep Server Actions, add ORPC router alongside
2. **Phase 2:** Migrate procedures one domain at a time
3. **Phase 3:** Update hooks to use ORPC client
4. **Phase 4:** Remove Server Actions

This allows gradual migration without breaking changes.

