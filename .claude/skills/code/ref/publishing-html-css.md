# HTML & CSS Publishing Rules

> Reference file for the Code skill. See `code/SKILL.md` for the main skill definition.

## HTML Markup Structure

- Use semantic HTML as the default.
- DO NOT nest meaningless wrapper `<div>` elements for styling purposes.
- DO NOT let DOM depth or sibling count grow excessively.
- Repeat the same structural pattern for blocks serving similar roles.
- Prefer native elements (`<button>`, `<a>`, `<form>`) over custom elements.
- FORBIDDEN: inline event attributes (`onclick="..."`).
- Add DOM nodes only when justified by one of: **semantics**, **layout**, or **interaction target**.

## Script Loading

- Use `defer` by default for DOM-dependent scripts.
- Use `async` only for independent external scripts that have no ordering dependency.
- Lazy-load non-critical widgets, ads, and embeds.
- Minimize JS dependencies for hero/above-the-fold content rendering.

## Images & Iframes

- Use `loading="lazy"` for below-the-fold images and iframes.
- Always provide `width` / `height` or aspect-ratio information to prevent layout shift.
- Use `srcset` / `sizes` for responsive images.
- FORBIDDEN: serving oversized source images and shrinking them with CSS.

## Events & Interaction

- Use passive listeners by default for touch/wheel/scroll events.
- Process scroll-based visual updates inside `requestAnimationFrame`.
- Prefer `ResizeObserver` over polling for element size observation.
- Cancel listeners and fetch requests when the view is removed or navigated away.
- Maintain a common cancel handle for fetch, event listeners, and timers.

## HTML State Management

- DO NOT use the DOM as a state store.
- Avoid duplicating values that already exist in DOM (e.g., form input intermediate values).
- Expose filter/sort/page state as URL state when possible.
- Transform network responses into a screen model — DO NOT copy raw responses across multiple components.

## CSS Layout

- Prefer Flexbox and Grid for layout.
- FORBIDDEN: using JS to continuously calculate and position elements.
- Use container queries when viewport media queries alone are insufficient.
- Structure large independent sections so that CSS `contain` can be applied.
- For long pages/feeds/catalogs, prefer `content-visibility: auto` + `contain-intrinsic-size`.

## CSS Animation

- Animate only `transform` and `opacity` by default.
- FORBIDDEN: animating `width`, `height`, `top`, `left`, `margin`.
- Keep interaction animations short: 120–240 ms range.
- Use CSS `transition` for hover/focus/press feedback.
- When JS-driven animation is necessary, use `requestAnimationFrame` only.
- Apply `will-change` only after profiling and only on targeted elements — NEVER declare it globally.

## CSS Style Complexity

- Use large blur, backdrop-filter, wide box-shadow, and blend-mode effects sparingly.
- Minimize large fixed/sticky layers.
- DO NOT stack layout + filter + box-shadow + mask + opacity changes on a single card/item simultaneously.

## CSS State Representation

- Control visual states via class names or `data-*` attributes.
- Minimize inline-style-based state toggling.
- Limit the number of properties that change when state changes.
- Choose the right hiding method by purpose: `display`, `visibility`, `opacity`, `content-visibility`.
