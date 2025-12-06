# SSR Prefetching Patterns

## Overview

Server-Side Rendering (SSR) prefetching allows you to fetch data on the server during the initial render and hydrate it into your client-side state management. This eliminates loading spinners on initial page loads, improves perceived performance, and enhances SEO by including data in the initial HTML response.

## Key Benefits

- **Instant Content** - Data arrives with the HTML, no loading states on initial render
- **Better Core Web Vitals** - Improved FCP and LCP metrics
- **SEO-Friendly** - Content is indexable by search engines
- **Progressive Enhancement** - Works with client-side state management (TanStack Query)
- **Automatic Hydration** - Seamless handoff from server to client
- **Reactive After Hydration** - Mutations and refetches work normally post-hydration

## Documentation Index

### [`benefits-of-prefetching.md`](./benefits-of-prefetching.md)
**Strategy & Decision Guide**

Understand when and what to prefetch for optimal performance. Covers:
- Performance and SEO benefits
- What to prefetch vs. what to skip
- The 80/20 rule for prefetching
- Common patterns for list, detail, and dashboard pages
- Practical recommendations and tradeoffs

**Start here if:** You want to understand the "why" behind prefetching and make informed decisions about what data to prefetch.

---

### [`prefetching-with-tanstack-query.md`](./prefetching-with-tanstack-query.md)
**Implementation Guide**

Step-by-step guide for implementing SSR prefetching with TanStack Query in Next.js. Covers:
- Creating server-side query client utilities
- Setting up HydrationBoundary for data transfer
- Prefetching in layouts vs. pages
- Query key patterns for server/client consistency
- Single item prefetching examples

**Start here if:** You're ready to implement prefetching and want concrete code examples.

---

## Quick Start Guide

### 1. **Understand the Benefits & Tradeoffs**
   → Read [`benefits-of-prefetching.md`](./benefits-of-prefetching.md)

### 2. **Implement with TanStack Query**
   → Read [`prefetching-with-tanstack-query.md`](./prefetching-with-tanstack-query.md)

## The Prefetching Decision

```
Without Prefetching:
Navigate → Render (empty) → Fetch → Loading spinner → Render with data

With Prefetching:
Navigate → Server prefetch → Render with data (no spinner)
```

## Key Concepts

- **Dehydration**: Serializing the server-side query cache into a format that can be sent to the client in the HTML response.

- **Hydration**: Restoring the serialized cache on the client so components have immediate access to prefetched data.

- **HydrationBoundary**: TanStack Query's component for passing dehydrated state to client components.

- **The 80/20 Rule**: Prefetch the 20% of data that provides 80% of the value—focus on critical path data.

## What to Prefetch

| ✅ DO Prefetch | ❌ DON'T Prefetch |
|----------------|-------------------|
| Above-the-fold content | Below-the-fold content |
| Critical path data | Heavy/slow queries (>500ms) |
| Small, fast queries | User-specific personalization |
| High-probability next actions | Everything indiscriminately |

## Related Documentation

- [`../orpc-proxying/`](../orpc-proxying/) - ORPC proxying patterns for unified server/client APIs
- [TanStack Query SSR Docs](https://tanstack.com/query/latest/docs/framework/react/guides/ssr) - Official TanStack Query SSR documentation

