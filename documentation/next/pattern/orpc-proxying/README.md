---
status: draft
priority: medium
type: reference
tags: [orpc, nextjs, proxying, index]
---

# ORPC Proxying Patterns

## Overview

ORPC (Open Remote Procedure Call) can be proxied through Next.js, providing a unified API that works seamlessly on both server and client sides. This approach eliminates the need to re-expose procedures for different access patterns and enables direct access to external ORPC monoliths with filtered, secure client access.

## Key Benefits

- **Unified API** - Same SDK works on server and client
- **No Re-Exposing** - Define procedures once, use everywhere
- **Type Safety** - End-to-end TypeScript types
- **Direct Access** - Server-to-server calls can bypass HTTP overhead
- **Filtered Access** - Control what's exposed to clients vs. server
- **Monolith Integration** - Direct access to external ORPC backends with partial exposure

## Documentation Index

### [`orpc-with-nextjs.md`](./orpc-with-nextjs.md)
**Getting Started Guide**

Learn how to set up ORPC proxying through Next.js. Covers:
- Creating ORPC route handlers (proxy)
- Setting up universal client SDK
- Server-to-server optimization
- Architecture patterns

**Start here if:** You're new to ORPC proxying or want to understand the basic setup.

---

### [`orpc-vs-server-actions.md`](./orpc-vs-server-actions.md)
**Decision Guide**

Compare ORPC with Next.js Server Actions to determine which approach fits your use case. Covers:
- Feature comparison table
- When to use each approach
- The "re-exposing" problem
- Migration strategies

**Start here if:** You're deciding between ORPC and Server Actions, or want to understand the tradeoffs.

---

### [`filtering-procedures.md`](./filtering-procedures.md)
**Security & Access Control**

Learn how to filter which ORPC procedures are exposed to different clients. Covers:
- Separate routers pattern (recommended)
- Context-based filtering
- Environment-based filtering
- Procedure-level filtering with metadata

**Start here if:** You need to restrict access to certain procedures or want to understand security patterns.

---

### [`integrating-external-monolith.md`](./integrating-external-monolith.md)
**Monolith Integration Guide**

Complete guide for integrating Next.js with an external ORPC monolith backend. Covers:
- Connecting to external monolith
- Creating filtered routers for client access
- Using Server Actions for orchestration/stitching
- Hybrid approaches
- Error handling and resilience

**Start here if:** You have an external ORPC monolith and want to integrate it with Next.js.

---

## Quick Start Guide

### 1. **Understanding ORPC Proxying**
   → Read [`orpc-with-nextjs.md`](./orpc-with-nextjs.md)

### 2. **Deciding if ORPC is Right for You**
   → Read [`orpc-vs-server-actions.md`](./orpc-vs-server-actions.md)

### 3. **Setting Up Security & Filtering**
   → Read [`filtering-procedures.md`](./filtering-procedures.md)

### 4. **Integrating External Monolith** (if applicable)
   → Read [`integrating-external-monolith.md`](./integrating-external-monolith.md)

## Common Use Cases

### Use Case 1: Internal Next.js App
**Recommended:** Server Actions
- Simple, Next.js-native solution
- No external API needed
- See [`orpc-vs-server-actions.md`](./orpc-vs-server-actions.md)

### Use Case 2: Next.js + Mobile App
**Recommended:** ORPC
- Unified API for web and mobile
- Same SDK everywhere
- See [`orpc-with-nextjs.md`](./orpc-with-nextjs.md)

### Use Case 3: Next.js + External ORPC Monolith
**Recommended:** ORPC with Filtering
- Direct server access to monolith
- Filtered client access for security
- Server Actions for orchestration
- See [`integrating-external-monolith.md`](./integrating-external-monolith.md)

### Use Case 4: Public API + Next.js Frontend
**Recommended:** ORPC
- Same router for API and frontend
- No re-exposing logic
- See [`orpc-with-nextjs.md`](./orpc-with-nextjs.md) and [`filtering-procedures.md`](./filtering-procedures.md)

## Architecture Patterns

### Pattern 1: Self-Contained ORPC Router
```
ORPC Router (in Next.js)
    ↓
Server Components (direct access)
    ↓
Client Components (HTTP proxy)
```

### Pattern 2: External Monolith Integration
```
External ORPC Monolith
    ↓ (direct access)
Next.js Server Components (full access)
    ↓ (filtered proxy)
Next.js Client Components (public only)
    ↓
Server Actions (orchestration layer)
```

## Key Concepts

- **Re-Exposing Problem**: With Server Actions, you often need to create new endpoints for different access patterns. ORPC solves this by defining procedures once.

- **Filtering**: Control which procedures are exposed to clients vs. server. Essential for security when integrating with external monoliths.

- **Orchestration**: Use Server Actions alongside ORPC to compose multiple ORPC calls and add Next.js-specific logic.

- **Type Safety**: Import types from your ORPC router or monolith package for end-to-end type safety.

## Related Documentation

- [`../ssr-prefetching/`](../ssr-prefetching/) - Server-side prefetching patterns
- [ORPC Official Docs](https://orpc.dev) - Official ORPC documentation

