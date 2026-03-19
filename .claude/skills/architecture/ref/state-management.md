# State Management Standards

> Reference file for the Architecture skill. See `architecture/SKILL.md` for the main skill definition.

## 1. Universal Publishing Principles

Core principle: **"Build small, read late, render only what is needed, keep state as close as possible."**

- Split every screen into small rendering units.
- Keep state as close to the reader as possible.
- When multiple places read and write, lift state only to the lowest common ancestor.
- UI renders state results only — side effects belong in dedicated lifecycle/effect scopes.
- DO NOT render things that are not visible, or defer their rendering.
- For long lists, render only the visible window — never the entire dataset.
- Animate only properties that do not trigger layout recalculation.
- Cancel network requests, listeners, and streams when the screen is removed.
- Server process memory is NOT an authoritative state store.
- Classify state into: **source state**, **derived state**, and **events**.

## 2. State Classification

### A. Ephemeral UI State

Examples: hover, focus, tab selection, accordion open/close, scroll position, temporary input values.

Rules:
- Keep inside the component/view/widget.
- DO NOT lift to global scope.
- If persistence is not needed, keep volatile.

### B. Screen State

Examples: displayed list, loading/error/empty status, sort/filter results, pagination state.

Rules:
- Keep in a per-screen state holder / view model / controller.
- Deliver to UI as an immutable snapshot.
- UI sends events upward only.

### C. App State

Examples: logged-in user, permissions, feature flags, shared settings, app theme/language.

Rules:
- Place at app root or feature boundary.
- DO NOT lift to global shared state unless multiple screens genuinely depend on it.

### D. Server State

Examples: API query data, cached remote responses, real-time subscription data.

Rules:
- Treat as a **cached projection**, not the source of truth on the frontend.
- Specify stale policy, refresh conditions, and cancel conditions explicitly.

### E. Backend Authority State

Examples: order status, payment result, inventory, session, workflow state, cache.

Rules:
- Store in external backing services: DB, event store, distributed cache, durable queue.
- DO NOT hold authoritative state in app server process memory for extended periods.

## 3. Frontend Common State Management

### 3.1 States That MUST NOT Be Stored

These look like state but must not be stored:
- Values computable from other state
- Copies of values already in server cache
- Labels computable from current selection
- `filteredItems` derivable from filter + items
- Threshold flags computable from scroll offset

Principle:
- Prefer **source state + derivation function** structure.
- DO NOT store the same semantic state in two places.

### 3.2 Minimize Subscription Scope

- FORBIDDEN: entire screen subscribing to global state.
- FORBIDDEN: entire list re-subscribing when a single item changes.
- High-frequency state MUST be subscribed by the smallest possible unit.
- Screen containers handle orchestration; leaves handle rendering.

### 3.3 Async State

- Every request MUST be cancellable.
- Prevent stale responses from overwriting current state — use version/token/id.
- Separate loading / error / empty / success states explicitly.
- Reduce duplicate in-flight requests for the same resource.
- FORBIDDEN: continuing to collect a stream after the screen is gone.

### 3.4 List State

- DO NOT keep the entire dataset in memory at all times.
- Manage by page, cursor, or visible window.
- For append-heavy feeds, define a policy for discarding old pages.
- Keep per-cell transient state local to the cell.

## 4. Backend State Management

### 4.1 Server Process Rules

- App servers MUST be stateless / share-nothing by default.
- DO NOT store authoritative state in process memory.
- On horizontal scaling, DB / cache / queue are responsible for state consistency.
- If sessions are needed, store them in an external store.

### 4.2 Cache Rules

- Cache is NOT the authoritative source.
- Cache keys MUST reflect domain, version, tenant, locale, and params.
- Every cache entry MUST have a TTL or explicit invalidation strategy.
- With multiple app server instances, prefer distributed cache over local in-memory cache.
- Use memory-efficient data structures for small aggregates.

### 4.3 Backend State Classification

| Type | Lifetime |
|---|---|
| Request State | Exists only during the request lifecycle |
| Session State | Scoped to the user session |
| Workflow State | Business flow (order, payment, approval) |
| Cache State | Derived / replicated data |
| Durable State | Source data persisted in DB |

Principle:
- Document which state has which lifecycle.
- Unknown lifecycles lead to memory leaks and stale state.

### 4.4 Efficiency Rules

- DO NOT hold large objects for the entire session duration.
- Load only necessary fields into memory.
- Distinguish hot-path DTOs from batch payloads.
- Monitor: cache hit rate, eviction, memory usage, stale ratio.

## 5. State Review Checklist

### Screen / Interaction
```
[ ] Does this screen render only what is visible?
[ ] Are lists lazy / windowed?
[ ] Do animations avoid triggering layout?
[ ] Does a state change avoid redrawing the entire screen?
[ ] Are async tasks for disappeared screens cleaned up?
```

### State
```
[ ] Is there a single source of truth?
[ ] Is derived state stored redundantly?
[ ] Is the state owner clear?
[ ] Are events distinguished from state?
[ ] Is the subscription scope for high-frequency state small enough?
```

### Backend
```
[ ] Where is the authority for this state?
[ ] Is it stuck in server memory?
[ ] Does the cache have TTL / invalidation?
[ ] Is consistency maintained across multiple instances?
```
