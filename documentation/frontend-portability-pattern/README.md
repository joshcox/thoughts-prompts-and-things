---
status: draft
priority: high
type: reference
tags: [frontend-portability, architecture, patterns, index]
---

# Frontend Portability Pattern

## Overview

The Frontend Portability Pattern enables building scalable, maintainable, and reusable UI in a monorepo environment through a three-layer architecture (Data, Domain, Presentation) and systematic hoisting of application concerns.

## Documentation Index

### [`architecture.md`](./architecture.md)
**Core Architecture Documentation**

Comprehensive guide to the Frontend Portability Pattern. Covers:
- Three-layer architecture (Data, Domain, Presentation)
- Monorepo structure and hybrid app/package pattern
- Compositional UI techniques
- Embedding scenarios (standalone, shallow, deep)
- Dependency management strategies
- Public API surface design
- Data layer substitution patterns

**Start here if:** You want to understand the complete architecture and design principles.

---

### [`guides/hoisting-refactoring-guide.md`](./guides/hoisting-refactoring-guide.md)
**Practical Refactoring Guide**

Step-by-step instructions for refactoring components using the hoisting algorithm. Covers:
- Three-step hoisting process
- Common refactoring patterns
- Complete examples with before/after code
- Testing strategies for hoisted components
- Troubleshooting common issues

**Start here if:** You're ready to refactor existing components to follow the pattern.

---

### [`technical-strategy/hoisting-algorithm.md`](./technical-strategy/hoisting-algorithm.md)
**Technical Strategy & Principles**

Deep dive into the hoisting algorithm and execution principles. Covers:
- When to use hoisting
- How hoisting works
- Design guidelines for props interfaces
- Common patterns and pitfalls
- State placement strategies

**Start here if:** You want to understand the technical strategy and principles behind hoisting.

---

## Quick Start Guide

1. **Understand the Architecture**
   → Read [`architecture.md`](./architecture.md)

2. **Learn the Hoisting Strategy**
   → Read [`technical-strategy/hoisting-algorithm.md`](./technical-strategy/hoisting-algorithm.md)

3. **Apply Hoisting to Your Components**
   → Read [`guides/hoisting-refactoring-guide.md`](./guides/hoisting-refactoring-guide.md)

## Key Concepts

- **Three-Layer Architecture**: Separation of Data (API access), Domain (state orchestration), and Presentation (context-free UI)
- **Hoisting**: Systematic process of extracting application concerns from components
- **Portability**: Components that work across multiple applications without modification
- **Composition**: Building complex UIs from simple, reusable pieces

## Related Documentation

- [Portable Frontend Pattern Rules](../../.cursor/rules/portable-frontend-pattern/) - Cursor rules for implementing the pattern

