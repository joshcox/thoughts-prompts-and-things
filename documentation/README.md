---
status: draft
priority: medium
type: reference
tags: [documentation, index]
---

# Documentation

## Overview

This directory contains documentation, patterns, strategies, and guides organized by topic area.

## Documentation Index

### [`frontend-portability-pattern/`](./frontend-portability-pattern/)
**Frontend Portability Pattern**

Architecture pattern for building scalable, portable UI in monorepo environments. Includes three-layer architecture, hoisting strategies, and refactoring guides.

**Key topics:** Architecture, hoisting, component portability, monorepo patterns

---

### [`next/`](./next/)
**Next.js Patterns**

Patterns, strategies, and architectural approaches for Next.js applications. Includes ORPC proxying, SSR prefetching, and architecture notes.

**Key topics:** ORPC, SSR, prefetching, Next.js architecture

---

### [`jobs/`](./jobs/)
**Jobs & Background Tasks**

Strategies for implementing and monitoring background jobs and scheduled tasks.

**Key topics:** Job execution, distributed locking, observability

---

### [`design/`](./design/)
**Design & Design Systems**

Documentation related to design systems, Figma plugins, and bridging the gap between design and engineering.

**Key topics:** Figma plugins, design systems, design-engineering collaboration

---

## Quick Navigation

- [Frontend Portability Pattern](./frontend-portability-pattern/) - Architecture for portable UI
- [Next.js Patterns](./next/) - Next.js-specific patterns and strategies
- [Jobs & Background Tasks](./jobs/) - Background job execution patterns
- [Design](./design/) - Design system documentation

## Note Classification

All documentation follows the [Obsidian notebook classification system](../../.cursor/rules/base.mdc#note-classification-system) with frontmatter fields for:
- `status` - Progress/completeness (seed, draft, developing, review, complete, archived)
- `priority` - Importance (low, medium, high, critical)
- `type` - Content category (note, pattern, strategy, reference, draft, decision)
- `tags` - Topic tags for cross-referencing

