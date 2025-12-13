---
status: draft
priority: medium
type: reference
tags: [nextjs, patterns, index]
---

# Next.js Pattern Library

## Overview

Collection of reusable patterns, strategies, and architectural approaches for Next.js applications.

## Documentation Index

### [`orpc-proxying/`](./orpc-proxying/)
**ORPC Proxying Patterns**

Patterns for proxying ORPC (Open Remote Procedure Call) through Next.js. Covers:
- Unified API for server and client
- Filtering procedures for security
- Integrating external ORPC monoliths
- Comparison with Server Actions

**Start here if:** You're using or considering ORPC with Next.js.

---

### [`ssr-prefetching/`](./ssr-prefetching/)
**SSR Prefetching Strategies**

Server-side rendering prefetching patterns with TanStack Query. Covers:
- Benefits and tradeoffs of prefetching
- Implementation guide
- What to prefetch vs. what to skip
- Performance optimization strategies

**Start here if:** You want to optimize initial page loads with SSR prefetching.

---

### [`notes/`](./notes/)
**Architecture Notes**

Architectural notes and decisions related to Next.js patterns. Includes:
- ORPC architecture patterns
- Design decisions and rationale

**Start here if:** You want to understand architectural decisions and patterns.

---

## Quick Start Guide

1. **Understanding ORPC**
   → Read [`orpc-proxying/README.md`](./orpc-proxying/README.md)

2. **Optimizing SSR Performance**
   → Read [`ssr-prefetching/README.md`](./ssr-prefetching/README.md)

3. **Architecture Decisions**
   → Browse [`notes/`](./notes/)

## Key Concepts

- **ORPC Proxying**: Unified API that works on both server and client
- **SSR Prefetching**: Server-side data fetching for instant client hydration
- **Pattern Library**: Reusable approaches for common Next.js challenges

