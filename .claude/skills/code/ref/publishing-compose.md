# Android Jetpack Compose Publishing Rules

> Reference file for the Code skill. See `code/SKILL.md` for the main skill definition.

## Rendering Principles

Compose frames follow **Composition → Layout → Drawing**. Rules:

- Read state as late as possible.
- Move state reads that don't need composition to layout/draw phases.
- Use `derivedStateOf` to reduce recompositions from high-frequency state.
- Cache expensive computations with `remember`.
- Use lazy layouts by default for scrollable lists.
- Provide stable `key` values for lazy list items.

## Composable Structure

- Composables receive state and expose events.
- Screen-level composables subscribe only to ViewModel state.
- Item composables receive only the item model.
- DO NOT place business logic inside composables.
- DO NOT scatter flow collection, coroutine launch, or repository calls inside composable bodies.

## State Management

- Use state hoisting as the default pattern.
- Hoist state only up to the lowest common ancestor of readers/writers.
- Expose UI state as immutable objects.
- Deliver events via function callbacks or intent/event objects.
- Manage screen state in ViewModel via `StateFlow`.
- Use `collectAsStateWithLifecycle()` as the default in Compose UI.

## Interaction & Animation

- DO NOT let the entire screen directly read high-frequency values like scroll position.
- Ensure animation triggers change only local state.
- Minimize pointer input, nested scroll, and custom layout until profiled.
- Use small state holders for press/expand/collapse interactions.

## Lists & Grids

- Use `Column`/`Row` only for small, non-scrolling data.
- Use `LazyColumn`, `LazyRow`, `LazyVerticalGrid` for large data sets.
- Always provide `key` when items can be inserted, removed, or reordered.
- DO NOT perform heavy derived computations inside list cells.

## Forbidden Patterns

- Heavy work inside composable body
- Hoisting state higher than necessary
- Lazy lists without stable keys
- Collecting Flow without lifecycle awareness
- Subscribing to rapidly-changing state from a high-level composable
