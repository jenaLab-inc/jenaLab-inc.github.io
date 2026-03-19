# iOS SwiftUI Publishing Rules

> Reference file for the Code skill. See `code/SKILL.md` for the main skill definition.

## Baseline

- Write all new iOS UI in SwiftUI first.
- Prefer Observation-based state models when possible.
- Avoid overusing shared observable objects that cause broad invalidation.
- Keep `body` focused on declarations, not computations.
- Use `List` or `LazyVStack` for large lists.

## State Tool Selection

| Property Wrapper | Use When |
|---|---|
| `@State` | Local value state owned by the view |
| `@Binding` | Child needs to mutate parent's state |
| `@StateObject` | View owns a reference-type model |
| `@ObservedObject` | Reference model injected from outside |
| `@EnvironmentObject` | Truly app-wide, low-frequency shared state only |

Additional rules:
- DO NOT spread high-churn state via `@EnvironmentObject`.
- On modern iOS, prefer Observation / `@Observable` first.

## View Structure

- Extract sub-views for parts with different state dependencies.
- Prevent small, non-layout-affecting state changes from invalidating the entire parent view.
- DO NOT repeat formatter creation, large array sorting, or filter chains inside `body`.
- Use stable identity values in `ForEach`.

## Async Work

- Prefer `.task(id:)` for view-lifecycle-bound work.
- Bind network/loading tasks that should cancel on disappearance to the view lifecycle.
- Handle background work in actor / task / model layer — views subscribe to results only.

## UIKit Interop Notes

When UIKit is used alongside SwiftUI, the same principles apply:
- Reuse cells properly.
- Minimize observation scope.
- Minimize update/layout invalidation.
- Prefer clear state owners over large shared objects.
- Leverage observation tracking on modern platforms.

## Forbidden Patterns

- Expensive computation inside `body`
- Broad `@EnvironmentObject` overuse
- Unstable identity in `ForEach`
- Orphaned async tasks that outlive the view lifecycle
- Cramming parts with different state dependencies into one `body`
