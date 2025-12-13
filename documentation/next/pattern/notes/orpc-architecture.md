---
status: draft
priority: high
type: pattern
tags: [architecture, orpc, nextjs, rpc, patterns]
---

Yes. **With orpc you can auto-generate typed SDKs** and thereby eliminate almost all "telephone translation" layers. The recommended revision is to treat the generated SDK as the *only* boundary artifact, and push all glue upward into React-only facades.

Below is an updated, stricter architecture consistent with current orpc practice.

---

## 1. Canonical Rule (Revised)

**orpc schema ⇒ generated SDK ⇒ adapters only at the edges**

No handwritten clients. No duplicated contracts. No domain re-expression.

```
orpc Procedures (authoritative)
        ↓
Generated SDK (typed, pure)
        ↓
Environment Adapter (transport + context)
        ↓
React Hook Facades
        ↓
Frontend Aggregates (page/application layer)
```

---

## 2. Generated SDK as the Domain API

orpc can generate:

* server callers
* HTTP clients
* type-safe SDKs usable in Node, RSC, browser, tests

This SDK becomes your **domain repository equivalent**.

Properties:

* deterministic
* referentially transparent (from caller POV)
* no React / Next.js / cache semantics

```ts
import { userSdk } from '@/rpc/generated';
```

No wrapper needed.

---

## 3. Environment Binding (Single Point)

Bind the SDK once per runtime.

### Server (RSC / route handlers)

```ts
export const serverSdk = userSdk.createCaller({
  ctx: serverContext,
});
```

### Client (browser)

```ts
export const clientSdk = userSdk.createClient({
  transport: httpTransport('/api/rpc'),
});
```

This is the *only* place environment branching exists.

---

## 4. React Glue = Hook Facades Only

Hooks **decorate** the SDK with:

* caching
* invalidation
* suspense semantics
* optimistic updates

They do not translate types or logic.

```ts
export function useUserRepo() {
  return {
    user: (id: UserId) =>
      useQuery({
        queryKey: ['user', id],
        queryFn: () => clientSdk.user.getById({ id }),
      }),

    updateProfile: () =>
      useMutation({
        mutationFn: clientSdk.user.updateProfile,
      }),
  };
}
```

This layer is intentionally boring and mechanical.

---

## 5. Frontend Aggregates (Explicitly Non-Domain)

Your observation is correct and important:

> frontend aggregates are *not* domain aggregates

They are:

* view-model builders
* orchestration graphs
* latency amortizers

```ts
export function useUserSettingsPage() {
  const user = useUserRepo();
  const billing = useBillingRepo();

  return {
    user: user.user('me'),
    plan: billing.currentPlan(),
  };
}
```

These are allowed to:

* stitch domains
* fan out RPCs
* reshape data

They are **explicitly unstable** over time.

---

## 6. Hoisting Rule (Performance-Driven Refactoring)

This is where expert teams converge.

### Hoist to Next.js API when:

* N+1 fanout emerges
* waterfall latency dominates
* data must be fetched atomically

Result:

```
Frontend Aggregate
    ↓ (hoisted)
Next.js API (still using SDK)
```

### Hoist to Backend Domain when:

* aggregate becomes canonical
* language appears in business discussions
* invariants or transactions are required

Result:

```
Frontend Aggregate
    ↓ (promoted)
Official Domain Procedure (orpc)
```

No rewrite—only relocation.

---

## 7. Why This Avoids Architectural Drift

* SDK generation prevents contract skew
* Frontend aggregates are explicitly disposable
* Promotion path is linear and reversible
* No early “backend-for-frontend” dogma

---

## 8. Emerging Pattern (Flagged Speculation)

⚠️ **Speculation**

Some teams now:

* generate *multiple SDKs* from the same orpc graph
* one “raw”
* one “denormalized / batched”

This treats SDK generation as a **compile-time query planner**, not just typing.

Early, but promising.

---

## 9. Compressed Summary

* Yes: auto-generated orpc SDKs should be the *only* client
* React hooks are pure glue
* Frontend aggregates are intentional, unstable, and hoistable
* Performance pressure dictates migration direction
* Domain language remains singular and authoritative

This is a clean, future-proof monolith posture.

---

### Sources

* orpc SDK generation patterns
* tRPC/orpc community design discussions
* TanStack Query architectural guidance
* Clean Architecture applied to RPC-first systems
