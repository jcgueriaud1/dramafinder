# Can Playwright's `matchesAriaSnapshot` replace D3.1 (structure assertions) in the Java binding?

Research date: 2026-07-31. Question (GitHub issue #6): D3 category 1 specifies structure assertions —
component present by role + accessible name, containment, sibling order. Playwright ships
`matchesAriaSnapshot`, which asserts a YAML tree of exactly those things. Does it exist in the **Java**
binding? Does it pierce shadow DOM, where Vaadin components keep their semantics? Can it be scoped to a
subtree and matched partially rather than exactly? And is there any Playwright API for asserting computed
CSS values or custom properties? If yes, D3.1 comes off the build list entirely and the interpreter shrinks.

## Source tiers used in this document

- **[OFFICIAL]** — Playwright's own documentation source, read from the `v1.59.0` tag of the
  `microsoft/playwright` GitHub repository (`docs/src/**`). Note: `playwright.dev` itself is **unreachable
  from this environment** (the outbound proxy rejects CONNECT with 403), so every doc quote below is taken
  from the repository source that generates those pages, at the exact version this project pins.
- **[SOURCE]** — implementation code read directly in `microsoft/playwright` at tag `v1.59.0`, or in the
  `playwright-java` sources jar published to Maven Central. Authoritative for *behaviour*, but an internal
  detail rather than a contract.
- **[OBSERVED]** — actually executed in this environment against Chromium: playwright-java 1.59.0 + its
  1.59.0 driver bundle, driving **real Vaadin 25.1.3 web components** (`@vaadin/text-field`,
  `@vaadin/button`, `@vaadin/checkbox`, `@vaadin/vertical-layout` installed from npm and bundled with
  esbuild) plus hand-written open- and closed-shadow-root custom elements. Every YAML block and PASS/FAIL
  line tagged `[OBSERVED]` is verbatim program output, not reconstruction.

**Environment caveat for [OBSERVED] results:** the driver wanted Chromium build `1217`; this machine has
`1194` preinstalled at `/opt/pw-browsers`, and `playwright install` is forbidden here, so the probe launched
with an explicit `setExecutablePath(".../chromium-1194/chrome-linux/chrome")`. Two Chromium builds of drift.
Nothing observed depends on a Chromium feature newer than shadow DOM or CSS custom properties, so this is
noted for completeness rather than as a real risk.

---

## 1. Java binding: existence and exact API surface

**Yes. Everything exists in Java, and the version this repo pins (1.59.0) has the newest parts too.**

`pom.xml` pins `com.microsoft.playwright:playwright:1.59.0` (`/home/user/dramafinder/pom.xml`).
All surface below was read from `playwright-1.59.0-sources.jar` on Maven Central.

### 1.1 The assertion

**[SOURCE]** `com/microsoft/playwright/assertions/LocatorAssertions.java`, verbatim:

```java
  /**
   * Asserts that the target element matches the given <a
   * href="https://playwright.dev/java/docs/aria-snapshots">accessibility snapshot</a>.
   * ...
   * @since v1.49
   */
  default void matchesAriaSnapshot(String expected) {
    matchesAriaSnapshot(expected, null);
  }
  ...
  void matchesAriaSnapshot(String expected, MatchesAriaSnapshotOptions options);
```

Two overloads, nothing more. And the options class, in full — this is the whole thing:

```java
  class MatchesAriaSnapshotOptions {
    /**
     * Time to retry the assertion for in milliseconds. Defaults to {@code 5000}.
     */
    public Double timeout;

    public MatchesAriaSnapshotOptions setTimeout(double timeout) { ... }
  }
```

Source: `https://repo1.maven.org/maven2/com/microsoft/playwright/playwright/1.59.0/playwright-1.59.0-sources.jar`

**[OFFICIAL]** Confirmed by the API doc source, which also carries the Java alias:

> ```
> ## async method: LocatorAssertions.toMatchAriaSnapshot
> * since: v1.49
> * langs:
>   - alias-java: matchesAriaSnapshot
> ```

Source: <https://raw.githubusercontent.com/microsoft/playwright/v1.59.0/docs/src/api/class-locatorassertions.md>

**File-based `.aria.yml` snapshots are JavaScript-only.** The second overload that takes a snapshot *name*
is explicitly gated:

> ```
> ## async method: LocatorAssertions.toMatchAriaSnapshot#2
> * since: v1.50
> * langs: js
> ...
> Snapshot is stored in a separate `.aria.yml` file in a location configured by
> `expect.toMatchAriaSnapshot.pathTemplate` and/or `snapshotPathTemplate` properties in the
> configuration file.
> ```

Source: same file. **[SOURCE]** Corroborated by absence: `MatchesAriaSnapshotOptions` has only `timeout`
in the sources jars for **1.49.0, 1.55.0, 1.56.0, 1.57.0, 1.58.0 and 1.59.0** (all six checked). There is
no `name`, no `pathTemplate`, no update-snapshots mode in Java. The `--update-snapshots` /
`--update-source-method=patch|3way|overwrite` machinery in the docs is `@playwright/test` only.

**The global children-mode config is also JS-only.** The docs show it as a `playwright.config.ts`
`expect.toMatchAriaSnapshot.children` setting. There is no Java equivalent — in Java the only way to
change match strictness is the per-template `/children:` property (section 3).

### 1.2 The generator

**[SOURCE]** `com/microsoft/playwright/Locator.java`:

```java
  /** ... @since v1.49 */
  default String ariaSnapshot() { return ariaSnapshot(null); }
  String ariaSnapshot(AriaSnapshotOptions options);
```

with

```java
  class AriaSnapshotOptions {
    public Integer depth;          // "When specified, limits the depth of the snapshot."
    public AriaSnapshotMode mode;  // AI | DEFAULT
    public Double timeout;         // default 30000
  }
```

`com.microsoft.playwright.options.AriaSnapshotMode` is a two-value enum: `AI`, `DEFAULT`.

**[SOURCE]** `com/microsoft/playwright/Page.java` additionally has a page-level generator:

```java
  /**
   * Captures the aria snapshot of the page. ...
   * @since v1.59
   */
  default String ariaSnapshot() { return ariaSnapshot(null); }
  String ariaSnapshot(AriaSnapshotOptions options);
```

**Version introduction table** (from `@since` javadoc tags plus direct inspection of the 1.49–1.59 sources
jars; all six jars were downloaded and read):

| API | Introduced | In 1.59.0 (pinned)? |
| --- | --- | --- |
| `LocatorAssertions.matchesAriaSnapshot(String[, MatchesAriaSnapshotOptions])` | **v1.49** | yes |
| `MatchesAriaSnapshotOptions.timeout` (the only option) | v1.49 | yes |
| `Locator.ariaSnapshot([AriaSnapshotOptions])` | **v1.49** | yes |
| `AriaSnapshotOptions.timeout` | v1.49 | yes |
| `AriaSnapshotOptions.mode` (`AriaSnapshotMode.AI`/`DEFAULT`) | **v1.59** | yes |
| `AriaSnapshotOptions.depth` | **v1.59** | yes |
| `Page.ariaSnapshot([AriaSnapshotOptions])` | **v1.59** | yes |
| `LocatorAssertions.hasCSS(String, String\|Pattern[, HasCSSOptions])` | v1.20 | yes |
| `LocatorAssertions.hasRole(AriaRole[, ...])`, `hasAccessibleName(...)`, `hasAccessibleDescription(...)` | v1.44 area | yes |
| `toMatchAriaSnapshot({ name })` — file-based | v1.50, **JS only** | **no Java equivalent** |

**[OFFICIAL]** on the two new options:

> ```
> ### option: Locator.ariaSnapshot.mode
> * since: v1.59
> - `mode` <[AriaSnapshotMode]<"ai"|"default">>
> ```
> ```
> ### option: Locator.ariaSnapshot.depth
> * since: v1.59
> - `depth` <[int]>
> ```

Source: <https://raw.githubusercontent.com/microsoft/playwright/v1.59.0/docs/src/api/class-locator.md>

There is **no** `matchesAriaSnapshot` on `PageAssertions` — that class only has `hasTitle` and `hasURL`.
Page-level structure assertions go through `assertThat(page.locator("body"))`.

**[OBSERVED]** All of the above compiled and ran against playwright-java 1.59.0. `Page.ariaSnapshot(...)`,
`Locator.ariaSnapshot(new Locator.AriaSnapshotOptions().setMode(AriaSnapshotMode.AI))`, and
`matchesAriaSnapshot(yaml, new LocatorAssertions.MatchesAriaSnapshotOptions().setTimeout(2500))` all work.

---

## 2. Shadow DOM

**Open shadow roots: pierced, including generation and matching. Closed shadow roots: invisible. Slotted
light-DOM children appear exactly once, at their flattened-tree position.**

### 2.1 The tree walk

**[SOURCE]** The aria tree builder is `packages/injected/src/ariaSnapshot.ts` (this file moved out of
`packages/playwright-core/src/server/injected/` in recent versions — that path 404s at `v1.59.0`). Its
`processElement` is the whole answer, verbatim:

```ts
    const assignedNodes = element.nodeName === 'SLOT' ? (element as HTMLSlotElement).assignedNodes() : [];
    if (assignedNodes.length) {
      for (const child of assignedNodes)
        visit(ariaNode, child, parentElementVisible);
    } else {
      for (let child = element.firstChild; child; child = child.nextSibling) {
        if (!(child as Element | Text).assignedSlot)
          visit(ariaNode, child, parentElementVisible);
      }
      if (element.shadowRoot) {
        for (let child = element.shadowRoot.firstChild; child; child = child.nextSibling)
          visit(ariaNode, child, parentElementVisible);
      }
    }
```

Source: <https://raw.githubusercontent.com/microsoft/playwright/v1.59.0/packages/injected/src/ariaSnapshot.ts>

Read literally, this is a **flattened-tree walk**:

1. A `<slot>` is replaced by its `assignedNodes()` — the slotted light-DOM children are visited *at the slot's
   position inside the shadow tree*.
2. Light-DOM children that **are** assigned to a slot (`child.assignedSlot` truthy) are skipped when walking
   the host's own children — so a slotted child is emitted **exactly once**, not twice.
3. `element.shadowRoot` children are walked after the unslotted light-DOM children. `element.shadowRoot` is
   `null` for a **closed** root, so closed shadow trees are silently skipped. There is no
   `openOrClosedShadowRoot` escape hatch on this path.
4. `visited` is a `Set<Node>` guarded at the top of `visit`, so a node reached twice is emitted once.

**[SOURCE]** Accessible-name computation does the *same* walk, so names composed from slotted or shadow
content resolve correctly. `packages/injected/src/roleUtils.ts`, inside the accname accumulator:

```ts
    const assignedNodes = element.nodeName === 'SLOT' ? (element as HTMLSlotElement).assignedNodes() : [];
    if (assignedNodes.length) {
      for (const child of assignedNodes)
        visit(child, false);
    } else {
      for (let child = element.firstChild; child; child = child.nextSibling)
        visit(child, true);
      if (element.shadowRoot) {
        for (let child = element.shadowRoot.firstChild; child; child = child.nextSibling)
          visit(child, true);
      }
      ...
    }
```

Source: <https://raw.githubusercontent.com/microsoft/playwright/v1.59.0/packages/injected/src/roleUtils.ts>

**[SOURCE]** One Vaadin-relevant gotcha, from the same file — light-DOM children of a shadow host that are
**not** assigned to any slot are treated as hidden:

```ts
    // When parent has a shadow root, all light dom children must be assigned to a slot,
    // otherwise they are not rendered and considered hidden for aria.
    if (element.parentElement && element.parentElement.shadowRoot && !element.assignedSlot)
      hidden = true;
```

That matches browser rendering, and it matters for Vaadin: an element you drop inside a Vaadin component
without a matching `slot=` will simply not appear in the snapshot.

### 2.2 Empirical confirmation, on real Vaadin components

**[OBSERVED]** Probe page: real `@vaadin/text-field@25.1.3`, `@vaadin/button`, `@vaadin/checkbox`,
`@vaadin/vertical-layout` (npm + esbuild bundle, `file://` page), plus a hand-written `<my-open-card>`
with an **open** shadow root containing a `role="group"`, an `<input aria-label="Shadow input">`, a named
`<slot name="title">`, a default `<slot>` and an `<a href="/more">`, and a `<my-closed-card>` with a
**closed** shadow root containing `role="group"` and a button.

`page.locator("body").ariaSnapshot()` printed, verbatim:

```yaml
- heading "Order form" [level=1]
- text: Customer name
- textbox "Customer name": Ada
- text: Email
- textbox "Email"
- checkbox "Subscribe" [checked]
- text: Subscribe
- button "Save"
- button "Cancel" [disabled]
- group "Open card":
  - heading "Slotted title" [level=2]
  - textbox "Shadow input": in-shadow
  - button "Slotted button"
  - link "Shadow link":
    - /url: /more
```

Read that carefully — five separate findings live in it:

1. **Open shadow DOM is pierced.** `group "Open card"`, `heading "Slotted title"` and `textbox
   "Shadow input"` all live inside `<my-open-card>`'s shadow root and all appear.
2. **Slotted content appears once, at its flattened position.** `heading "Slotted title"` is the shadow
   `<h2><slot name="title"></slot></h2>` filled by the light-DOM `<span slot="title">`; `button "Slotted
   button"` is a light-DOM child surfacing through the default `<slot>`, nested under the shadow `group`,
   in slot order — not duplicated at the host level.
3. **Closed shadow DOM is invisible.** `group "Closed card"` and `button "Hidden button"` are absent, and
   `page.locator("my-closed-card").ariaSnapshot()` returned the **empty string**. Asserting
   `- button "Hidden button"` against `body` **FAILED**, as expected.
4. **Vaadin semantics resolve correctly.** `vaadin-text-field` yields `textbox "Customer name": Ada` (name
   from the light-DOM `<label slot="label">` via `aria-labelledby`, value from the slotted `<input>`);
   `vaadin-checkbox` yields `checkbox "Subscribe" [checked]`; the disabled `vaadin-button` yields
   `button "Cancel" [disabled]`.
5. **The host element itself contributes no node, and label text leaks as a sibling.** There is no
   `vaadin-text-field` node — only the inner `textbox`. And there is a stray `- text: Customer name`
   *sibling* of the textbox (the same `<label>` counted once as accessible name, once as text). Any
   `/children: equal` template must list those stray text nodes; see section 3.

`page.locator("my-open-card").ariaSnapshot()`:

```yaml
- group "Open card":
  - heading "Slotted title" [level=2]
  - textbox "Shadow input": in-shadow
  - button "Slotted button"
  - link "Shadow link":
    - /url: /more
```

**Not verified:** whether an `<iframe>` boundary is crossed in `default` mode. **[OFFICIAL]** says `ai` mode
"Includes snapshots of `<iframe>`s inside the target", which implies `default` mode does not — but that was
not tested here. Irrelevant for Vaadin views, which do not use iframes for component internals.

---

## 3. Partial vs exact matching, and subtree scoping

**Default is a containment match: an ordered subsequence at each level, extra siblings tolerated, unlisted
children ignored. Accessible names are matched *exactly* (case-sensitive) unless written as `/regex/`.
Scoping is "call it on any Locator" — but with an important twist: the template is searched over the
*whole subtree*, not anchored at the locator's root.**

### 3.1 The matcher, read from source

**[SOURCE]** `packages/injected/src/ariaSnapshot.ts`, the container-mode dispatch:

```ts
  // Proceed based on the container mode.
  if (template.containerMode === 'contain')
    return containsList(node.children || [], template.children || []);
  if (template.containerMode === 'equal')
    return listEqual(node.children || [], template.children || [], false);
  if (template.containerMode === 'deep-equal' || isDeepEqual)
    return listEqual(node.children || [], template.children || [], true);
  return containsList(node.children || [], template.children || []);
```

and the two list algorithms, verbatim:

```ts
function listEqual(children, template, isDeepEqual): boolean {
  if (template.length !== children.length)
    return false;
  for (let i = 0; i < template.length; ++i) {
    if (!matchesNode(children[i], template[i], isDeepEqual))
      return false;
  }
  return true;
}

function containsList(children, template): boolean {
  if (template.length > children.length)
    return false;
  const cc = children.slice();
  const tt = template.slice();
  for (const t of tt) {
    let c = cc.shift();
    while (c) {
      if (matchesNode(c, t, false))
        break;
      c = cc.shift();
    }
    if (!c)
      return false;
  }
  return true;
}
```

`containsList` is a **greedy ordered-subsequence** match: it consumes children left to right looking for
each template entry in turn. Extra children before, between and after listed ones are skipped. Reordering
the template breaks it. That is precisely "containment + sibling order".

**[SOURCE]** Name matching:

```ts
function matchesStringOrRegex(text: string, template: aria.AriaRegex | string | undefined): boolean {
  if (!template)
    return true;                      // omitted name -> matches anything
  if (!text)
    return false;
  if (typeof template === 'string')
    return text === template;         // EXACT, case-sensitive
  return !!text.match(new RegExp(template.pattern));
}
```

So `- textbox "Custom"` does **not** match a textbox named `Customer name`. Substring matching requires
`/regex/` syntax. Text-node values (`- textbox "X": value`, `- text: foo`) go through `matchesTextValue`,
which additionally accepts a `/…/`-delimited raw string as a regex.

**[SOURCE]** The recognised attribute forms, from the YAML parser
(`packages/playwright-core/src/utils/isomorphic/ariaSnapshot.ts`, `_applyAttribute`) — `checked`
(`true`/`false`/`mixed`), `disabled`, `expanded`, `active`, `level` (number), `pressed`
(`true`/`false`/`mixed`), `selected`, and then:

```ts
    this._assert(false, `Unsupported attribute [${key}]`, errorPos);
```

Anything else is a **template parse error**, not a silent pass. Separately, keys starting with `/` are
handled: `/children` (`contain` | `equal` | `deep-equal`, anything else is an error) and any other
`/name:` becomes a *property* — only `url` is actually produced by the generator (for links), and it is
matched with `matchesTextValue`, so `/url` supports regex too.

**[SOURCE]** The scoping subtlety. The YAML root becomes a `fragment` node:

```ts
  const fragment: AriaTemplateNode = { kind: 'role', role: 'fragment' };
  ...
  // `- button` should target the button, not its parent.
  if (fragment.children?.length === 1 && (!fragment.containerMode || fragment.containerMode === 'contain'))
    return { fragment: fragment.children[0], errors: [] };
```

and matching runs through `matchesNodeDeep`, which recurses:

```ts
function matchesNodeDeep(root, template, collectAll, isDeepEqual) {
  const results = [];
  const visit = (node, parent) => {
    if (matchesNode(node, template, isDeepEqual)) { ... return !collectAll; }
    if (typeof node === 'string') return false;
    for (const child of node.children || []) {
      if (visit(child, node)) return true;
    }
    return false;
  };
  visit(root, null);
  return results;
}
```

**Consequence:** `matchesAriaSnapshot` is *existential over the locator's subtree*, not positional. A
template is satisfied if it matches the locator's root node **or any descendant**. So the locator bounds
the search, and nothing more. A single-node template like `- button "Save"` is really "there exists a
button named Save somewhere under this locator". This is exactly the semantics you want for a spec
interpreter — but it is stronger flexibility than "the locator is the root", and it means `/children: equal`
also only pins the children of *whichever* node the template matched.

### 3.2 What the official docs say

**[OFFICIAL]**, verbatim:

> * If the tree structure matches the template, the test passes; otherwise, it fails, indicating a mismatch
>   between expected and actual accessibility states.
> * The comparison is case-sensitive and collapses whitespace, so indentation and line breaks are ignored.
> * The comparison is order-sensitive, meaning the order of elements in the snapshot template must match the
>   order in the page's accessibility tree.

> ### Partial matching
> You can perform partial matches on nodes by omitting attributes or accessible names … In this example,
> the button role is matched, but the accessible name ("Submit") is not specified, allowing the test to
> pass regardless of the button's label.

> ### Strict matching
> By default, a template containing the subset of children will be matched
>
> The `/children` property can be used to control how child elements are matched:
> - `contain` (default): Matches if all specified children are present in order
> - `equal`: Matches if the children exactly match the specified list in order
> - `deep-equal`: Matches if the children exactly match the specified list in order, including nested children

> ### Matching with regular expressions
> Regular expressions allow flexible matching for elements with dynamic or variable text. Accessible names
> and text can support regex patterns.

Source: <https://raw.githubusercontent.com/microsoft/playwright/v1.59.0/docs/src/aria-snapshots.md>

### 3.3 Empirical matrix

**[OBSERVED]** Verbatim probe output (`PASS`/`FAIL` is the outcome; the label states the intent):

```
PASS  partial: only some children of body listed
PASS  partial: single textbox by accessible name, nothing else
PASS  in-order subset of siblings (skip middle ones)
FAIL  OUT-OF-ORDER siblings (Save before Customer name) - expect FAIL
PASS  regex accessible name /Custom/
FAIL  substring WITHOUT regex slashes "Custom" - expect FAIL (exact match)
PASS  role only, no name
PASS  checkbox [checked]
PASS  button [disabled]
PASS  containment: group 'Open card' contains link (nested)
PASS  /url property on shadow link
FAIL  /children: equal on #card (only 2 of 5 listed) - expect FAIL
FAIL  SCOPING: template matched against a locator that does NOT contain it - expect FAIL
PASS  SCOPING probe: does the template have to match at the ROOT of the locator? textbox nested deep under body root
FAIL  closed shadow root content visible? 'Hidden button' - expect FAIL
PASS  slotted light-dom button appears once via flattened tree
```

and from the second probe:

```
--- CONTAINMENT THROUGH A ROLE-LESS CONTAINER (vaadin-vertical-layout) ---
vaadin-vertical-layout role = null
FAIL  nest button under a 'generic' node in YAML (default mode)
FAIL  nest button under 'group' (layout has no role)
--- CASE SENSITIVITY / VALUE FORM ---
FAIL  lower-case name "save" (expect FAIL: case-sensitive)
PASS  textbox value form: - textbox "Customer name": Ada
FAIL  textbox WRONG value: - textbox "Customer name": Bob (expect FAIL)
PASS  /children: equal that lists EVERYTHING incl. stray text nodes
--- EXISTENTIAL vs POSITIONAL ---
PASS  template matches a DESCENDANT, not just locator root (group nested inside body)
```

Every line matches the source reading. Two deserve emphasis.

### 3.4 What the aria snapshot does NOT capture — the decisive limitation for this project

**Role-less elements produce no node at all, so containment through a Vaadin layout is not expressible in
the YAML.**

**[SOURCE]** `toAriaNode` returns `null` — meaning "no node, promote the children to the parent" — for
anything without a role:

```ts
  const defaultRole = options.includeGenericRole ? 'generic' : null;
  const role = roleUtils.getAriaRole(element) ?? defaultRole;
  if (!role || role === 'presentation' || role === 'none')
    return null;
```

`includeGenericRole` is `true` only for `mode: 'ai'`. **[SOURCE]** `toInternalOptions` — the mode used by
`matchesAriaSnapshot` is `default`, i.e. `{ visibility: 'aria', refs: 'none' }`, with no
`includeGenericRole`. And `matchesExpectAriaTemplate` hard-codes it:

```ts
export function matchesExpectAriaTemplate(rootElement, template) {
  const snapshot = generateAriaTree(rootElement, { mode: 'default' });
  ...
```

So the `generic` nodes you can see in `ai` mode are **not available to the assertion**.

**[OBSERVED]** This is not theoretical. `<vaadin-vertical-layout>` has `role = null`, and

```yaml
# page.locator("vaadin-vertical-layout").ariaSnapshot()
- text: Customer name
- textbox "Customer name": Ada
- text: Email
- textbox "Email"
- checkbox "Subscribe" [checked]
- text: Subscribe
- button "Save"
- button "Cancel" [disabled]
```

— completely flat. The layout contributes nothing, its children are hoisted into the grandparent, and both
attempts to express the nesting in YAML failed:

```
FAIL  nest button under a 'generic' node in YAML (default mode)
FAIL  nest button under 'group' (layout has no role)
```

For contrast, the same page in `ai` mode *does* show the structure, as `generic` nodes with `[ref=…]`:

```yaml
- generic [ref=e2]:
  - generic:
    - generic [ref=e4]:
      - generic [ref=e5]:
        - generic: Customer name
        - text: "*"
      - textbox "Customer name" [ref=e7]: Ada
    ...
```

but that tree is only reachable through `Locator.ariaSnapshot(mode=AI)`, which is a **string generator, not
a matcher**. There is no `matchesAriaSnapshot` variant that matches against the AI tree.

Also absent from the snapshot, for completeness: anything `display:none` / `aria-hidden="true"` /
unslotted-under-a-shadow-host (`isElementHiddenForAria`), `role="presentation"`/`"none"`, all CSS and
geometry, all non-ARIA DOM attributes (`theme=`, `alignment=`, custom properties), and closed shadow trees.

**The workaround for containment is locator scoping, not YAML nesting.** Because `matchesAriaSnapshot`
takes any `Locator`, `assertThat(page.locator("vaadin-vertical-layout#summary")).matchesAriaSnapshot("- button \"Save\"")`
does express "Save is inside that layout". That works — but only when the container is *addressable*, which
is exactly what D5's unlabeled-group problem is about.

---

## 4. CSS assertions, including custom properties

**`hasCSS` exists in Java, reads computed style, and — decisively — DOES work for CSS custom properties,
because the implementation calls `getPropertyValue`, not bracket access.**

### 4.1 The API

**[SOURCE]** `com/microsoft/playwright/assertions/LocatorAssertions.java` — four overloads, all `@since v1.20`:

```java
  default void hasCSS(String name, String value) { hasCSS(name, value, null); }
  void hasCSS(String name, String value, HasCSSOptions options);
  default void hasCSS(String name, Pattern value) { hasCSS(name, value, null); }
  void hasCSS(String name, Pattern value, HasCSSOptions options);
```

**[OFFICIAL]**:

> ```
> ## async method: LocatorAssertions.toHaveCSS
> * since: v1.20
> * langs:
>   - alias-java: hasCSS
>
> Ensures the [Locator] resolves to an element with the given computed CSS style.
> ```
> ```java
> assertThat(page.getByRole(AriaRole.BUTTON)).hasCSS("display", "flex");
> ```

Source: <https://raw.githubusercontent.com/microsoft/playwright/v1.59.0/docs/src/api/class-locatorassertions.md>

### 4.2 The one line that decides the custom-property question

**[SOURCE]** `com/microsoft/playwright/impl/LocatorAssertionsImpl.java` passes the property name through
untouched as `expressionArg` and sends `to.have.css` to the driver:

```java
  private void hasCSS(String name, ExpectedTextValue expectedText, Object expectedValue, HasCSSOptions options) {
    ...
    commonOptions.expressionArg = name;
    ...
    expectImpl("to.have.css", expectedText, expectedValue, message, commonOptions, "Assert \"hasCSS\"");
  }
```

and the injected script resolves it — `packages/injected/src/injectedScript.ts`, line 1606, verbatim:

```ts
      } else if (expression === 'to.have.css') {
        received = this.window.getComputedStyle(element).getPropertyValue(options.expressionArg);
```

Source: <https://raw.githubusercontent.com/microsoft/playwright/v1.59.0/packages/injected/src/injectedScript.ts>

That is `getPropertyValue(name)`, **not** `getComputedStyle(element)[name]`. `getPropertyValue` is the API
that works for custom properties; the bracket form returns `undefined` for them. So `hasCSS("--lumo-primary-color", …)`
is fully supported — accidentally rather than by documentation, but by the only code path that matters.

### 4.3 Empirical confirmation

**[OBSERVED]** Probe page CSS:

```css
html  { --lumo-primary-color: hsl(214, 100%, 48%); --app-accent: var(--lumo-primary-color); }
#card { --lumo-primary-color: hsl(120, 50%, 40%); color: var(--app-accent); }
```

Results, verbatim:

```
      computed color = rgb(0, 106, 245)
PASS  hasCSS('color', ...) computed
PASS  hasCSS('--lumo-primary-color', 'hsl(120, 50%, 40%)') on #card
      evaluate getPropertyValue('--app-accent') = [hsl(214, 100%, 48%)]
      evaluate getComputedStyle(e)['--app-accent'] = [undefined]
PASS  hasCSS('--app-accent', ...) - var() chain resolved?
      vaadin-text-field --lumo-primary-color = [hsl(120, 50%, 40%)]
PASS  hasCSS custom property inherited into shadow DOM part
      shadow input count = 1
      shadow input color = rgb(31, 31, 31)
PASS  hasCSS on an element INSIDE shadow DOM via >> locator
undefined custom prop via evaluate = []
PASS  hasCSS undefined custom property equals empty string
PASS  hasCSS custom property with regex overload
vaadin-button host --lumo-primary-color = [hsl(120, 50%, 40%)]
```

Seven things established:

1. **`hasCSS` asserts custom properties.** `hasCSS("--lumo-primary-color", "hsl(120, 50%, 40%)")` passes.
2. **The `Pattern` overload works on custom properties too** (`^hsl\(120`).
3. **An undefined custom property reads as the empty string**, and `hasCSS("--not-defined", "")` passes —
   so "token not defined" is *not* distinguishable from "token defined as empty" without extra care.
4. **The value returned is the RESOLVED value, not the `var()` chain.** `--app-accent` on `#card` reads
   `hsl(214, 100%, 48%)` even though `#card` redefines `--lumo-primary-color` to green — because the
   substitution happened at `html`, where `--app-accent` was declared, and the *substituted* value is what
   inherits. There is no API anywhere in Playwright that returns the unresolved `var()` chain.
5. **The bracket-access path really would have failed**: `getComputedStyle(e)['--app-accent']` is
   `undefined`. Worth remembering if any DramaFinder helper ever hand-rolls this.
6. **Custom properties inherit into shadow DOM as expected** — the `vaadin-text-field` host and the
   `vaadin-button` host both read the `#card`-scoped green value, and `hasCSS` on a locator pointing
   *inside* a Vaadin shadow root works normally.
7. **Format normalisation is a real problem.** A custom property returns its author-written text
   (`hsl(120, 50%, 40%)`), while a real color property returns Chromium's serialisation
   (`rgb(0, 106, 245)`). Comparing "does `background-color` equal token `{colors.primary}`" means comparing
   an `rgb(...)` against an `hsl(...)` string. **Direct string comparison will not work.**

**Concrete fallback and its limits.** `hasCSS` covers everything the plan needs *value-wise*; the fallback
to `locator.evaluate("e => getComputedStyle(e).getPropertyValue('--x')")` is only needed when you must read
a value in order to *compute* the expectation (e.g. resolve `{colors.primary}` to a concrete string, then
normalise it, then `hasCSS("background-color", normalised)`). Either way you get the **resolved** value, so
"assert the token chain" can only ever be implemented as "assert the two resolved values agree", plus your
own normalisation. Playwright provides neither the chain nor a color-equality comparator.

---

## BOTTOM LINE

**D3.1 does not come off the build list. Its *engine* does.**

`matchesAriaSnapshot` exists in Java (`@since v1.49`, pinned 1.59.0 has it plus 1.59-only `mode`/`depth`/
`Page.ariaSnapshot`), it pierces open shadow roots and flattens slots correctly on real Vaadin 25.1.3
components, it is a containment match with tolerated extra siblings and enforced sibling order, it is
scoped by whatever `Locator` you call it on, and `hasCSS` does read CSS custom properties. Those are four
genuine wins and they should be taken. But the assertion is a single all-or-nothing matcher over a YAML
string, and three things the plan requires sit outside what it can express.

**What can be deleted from the D3.1 build (real savings):**

- The hand-rolled accessibility walk. Stop extending `ComponentSnapshot` for presence/role/name. Emit one
  micro-template per structural assertion (`- button "Save"`, `- textbox /Custom/`, `- checkbox "Subscribe" [checked]`)
  and call `matchesAriaSnapshot` on a scoped `Locator`. That buys shadow piercing, spec-correct accname
  computation, `checked`/`disabled`/`expanded`/`level`/`pressed`/`selected`, `/url`, regex names, and
  Playwright's auto-wait/retry (`setTimeout`) for free.
- Sibling-order logic: put two or more entries in one template; `containsList` is exactly an ordered
  subsequence over one parent's children.
- Containment **when the container is addressable and role-bearing**: scope the locator to the container
  and assert presence inside it.

**What still has to be built — and why aria snapshots cannot supply it:**

1. **Spec-YAML → aria-snapshot-YAML translation, with a per-component role/name model.** D5 defines an
   abstract vocabulary (`button`, `text-input`, `select`, `grid`, `container`, …) that the interpreter maps
   to acceptable realizations. Aria snapshot YAML speaks concrete ARIA roles, so the mapping still has to
   exist — now targeting role *strings* instead of DramaFinder element classes. It also needs component
   knowledge that only shows up on real components: `vaadin-text-field` contributes **no node for the host**
   (only the inner `textbox`), and its `<label>` also emits a **stray `- text:` sibling**. Any
   `/children: equal` template must enumerate those artifacts, which argues for staying on the default
   `contain` mode and never generating `equal` from a spec.
2. **Containment through role-less Vaadin layouts — still yours.** `<vaadin-vertical-layout>` has
   `role = null`, so in the `default` mode the matcher uses, it produces **no node** and its children are
   hoisted. Nesting under it is simply unwriteable in the template; both attempts failed empirically. The
   `generic` nodes that would express it exist only in `ai` mode, which has no matcher. Containment must
   therefore be implemented as **locator scoping**, which means D5's group problem — locate the named
   members, compute their LCA, walk the ancestor chain — is untouched and remains the risky part (it is
   already gated by the group-resolution spike, build order item 2). Nothing here retires that spike.
3. **Per-assertion results and the tri-state routing (D7).** `matchesAriaSnapshot` throws one
   `AssertionError` carrying the entire received tree; it cannot say *which* node failed, cannot distinguish
   "missing" (`fail`) from "ambiguous target" (`error`), and cannot fill `{id, target, property, expected,
   actual, hint}`. Score = `100 × passed/(passed+failed)` needs assertions to be countable. So the
   interpreter keeps its own assertion loop and result model, calling `matchesAriaSnapshot` once per
   structural claim rather than once per view. Ambiguity detection (`Locator.count() > 1` → `error`) stays
   hand-written, since the matcher is existential and would happily pass on the first match.
4. **D3.2 (layout intent at the attribute level) is entirely unaffected.** `alignment="end"` on a Vaadin
   layout is a DOM attribute with no ARIA projection. Aria snapshots see none of it.
5. **D3.3 (token-chain assertions) is helped but not solved.** `hasCSS` does assert custom properties —
   that removes the need for an `evaluate` fallback for *reading* a token. What remains: `getPropertyValue`
   returns the **resolved** value, never the `var()` chain, and custom properties serialise as authored
   (`hsl(120, 50%, 40%)`) while real color properties serialise as Chromium normalises them
   (`rgb(0, 106, 245)`). "Assert the chain" is therefore implementable only as "resolve both sides, normalise,
   compare" — and the **color normalisation comparator is new code the plan has not accounted for**. Also
   note an undefined custom property reads as `""`, which `hasCSS(name, "")` happily passes, so
   "token missing" needs an explicit check rather than an empty-string expectation.

**Net effect on the plan:** D3.1's *mechanism* shrinks to a YAML-emitter plus `matchesAriaSnapshot` calls —
a real reduction, and it removes an entire class of shadow-DOM and accessible-name bugs DramaFinder would
otherwise have had to get right itself. D3.1's *semantics* — abstract role mapping, group/LCA resolution,
scoping, ambiguity, per-assertion reporting — survive intact, and the group-resolution spike stays the
gating risk. Build order is unchanged; item 3 (`DesignSpecVerifier`) just gets cheaper. One thing to add to
the plan explicitly: a color/format normaliser for D3.3.

Two contract notes worth recording, since both are version-sensitive:
`Locator.AriaSnapshotOptions.mode`/`depth` and `Page.ariaSnapshot` are **new in 1.59.0** (absent in 1.55–1.58,
verified against six sources jars), so any use of them raises this project's Playwright floor to 1.59.0.
And file-based `.aria.yml` snapshots plus the global `expect.toMatchAriaSnapshot.children` config are
**JavaScript-only** — in Java the YAML is always an inline string and strictness is always per-template
`/children:`. Since the interpreter generates its templates from the spec anyway, neither absence costs
anything.
