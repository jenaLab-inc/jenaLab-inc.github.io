# Flutter Publishing Rules

> Reference file for the Code skill. See `code/SKILL.md` for the main skill definition.

## Rendering Principles

- `build()` MUST be cheap and pure.
- Push state down to leaf widgets.
- Prefer multiple small `const` widgets over one large widget.
- Use `.builder` constructors for long lists.
- DO NOT place unnecessary `Opacity`, clipping, or shadow widgets at the top of the tree.
- Minimize effects that trigger `saveLayer`.

## Widget Structure

- One widget = one responsibility.
- Prefer extracting independent widgets over helper functions.
- DO NOT perform sorting, grouping, heavy computation, JSON parsing, or image post-processing inside `build()`.
- DO NOT recreate the same child widget every frame.
- Always check whether `const` construction is possible first.

## Layout & Scroll

- Use plain `ListView` only for small, static lists.
- Use `ListView.builder` / `GridView.builder` for large or infinite lists.
- Prefer sliver-based structures for complex scroll scenarios.
- Reduce the widget count inside each cell as item count grows.
- Keep per-cell state local to the cell.

## Interaction

- Show touch feedback within a single frame.
- Prefer localized transitions over full-screen animations.
- Minimize `setState` scope during drag/scroll.
- Prevent gesture-triggered state changes from causing full parent rebuilds.

## Flutter State Management

- Separate local widget state from app state.
- For simple apps/features, start with Provider (per official docs).
- For medium/large apps, split ViewModel/Repository layers.
- Manage screen state as immutable snapshots.
- DO NOT dump all app UI state into a single notifier.
- Objects that require `dispose()` MUST have clear ownership.

## Forbidden Patterns

- Heavy computation inside `build()`
- `setState` on the entire top-level tree
- Rendering large lists with a plain `ListView`
- Unnecessary `Opacity` / `Clip` / `saveLayer`
- Missing `dispose()` calls
