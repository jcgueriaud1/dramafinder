# Axe engine via the Deque library, but element scoping via `axe.run(node)`

Accessibility checking uses `com.deque.html.axe-core:playwright` as its engine,
result model, and source of axe-core itself. Whole-page checks delegate to
`AxeBuilder.analyze()`. Element-scoped checks do **not** — they inject
`AxeBuilder.getAxeScript()` and call `axe.run(el, options)` through
`Locator.evaluate`, then deserialise the raw result into Deque's `AxeResults`.
Both paths therefore return the same type, so failure semantics are written once.

## Why not `AxeBuilder.analyze()` for both

`AxeBuilder` is page-scoped: `analyze()` takes no argument, and targeting goes
through `include(String)` or `include(FromShadowDom)` — CSS selector chains. A
Playwright `Locator` cannot be converted back into a selector string, so with
`AxeBuilder` alone, `TextFieldElement.getByLabel(page, "Name")` yields nothing
that can be handed to axe. Callers would hand-write `FromShadowDom` chains and
abandon the element API, which is the one thing this library exists to provide.

axe-core accepts native DOM element references as context, not only selectors,
so passing the node behind a `Locator` sidesteps the problem entirely — and
works for nodes inside a shadow root, where a plain CSS `include` would not
reach.

## Considered and rejected

- **Marker attribute** — stamp a temporary attribute on the target, then
  `include("[data-df-a11y]")`. Keeps a single Deque entry point, but mutates the
  DOM under assertion, and axe's plain CSS include does not pierce shadow DOM.
- **Deriving a selector from the `Locator`** — locators are built with runtime
  filters (`.filter(…).first()`) that have no selector equivalent.
- **Hand-rolling axe entirely** (bundling `axe.min.js` ourselves) — avoids the
  dependency, but means owning the result mapping and manually tracking axe
  releases, and ships an MPL-2.0 asset inside an Apache-2.0 jar.

## Consequences

- The dependency puts jackson, commons-io, commons-codec, commons-compress and
  `dequeutilites` on the graph of a library whose only compile dependency was
  previously Playwright. This was accepted: using the maintained engine is worth
  the footprint.
- Playwright moved to 1.62.0. The axe artifact compiles against 1.60.0, so our
  direct declaration wins under nearest-wins and exactly one Playwright is on the
  graph.
- The element path uses plain `axe.run`, not `runPartial`/`finishRun`, so it does
  not traverse iframes. Irrelevant within a single component; page scope keeps
  full iframe support because it goes through `analyze()`.
- The engine is absent from public names (`AccessibilityCheck`, not `AxeCheck`)
  so it can be replaced without a breaking rename.
