# Design token standards vs. our bespoke `cssVariables` mapping

Research date: 2026-07-31. Question: does an existing design-token standard already
model what we were about to invent as a bespoke `cssVariables` key in DESIGN.md —
including light/dark theme variants and token → CSS-custom-property generation?

## BOTTOM LINE

`REINVENTION: partial`

Adopt DTCG as the *container* and drop the bespoke Markdown key. Everything structural
in our plan already exists as a stable, dated standard: token paths (`colors.primary`)
are literally DTCG group nesting, our `{colors.primary}` assertion syntax is verbatim
DTCG alias syntax, the multi-level `--aura-accent` → `--aura-blue-500` → raw-value chain
is DTCG "Chained References" (explicitly allowed, MUST be followed to an explicit value),
and the `scope: [data-theme="dark"]` dimension of our export is the whole subject of a
second stable report — the Design Tokens **Resolver** Module 2025.10 (`modifiers` /
`contexts`), which is implemented today by Terrazzo's CSS plugin as configurable
per-context selectors. We would gain: a versioned file format with official JSON
Schemas, a sanctioned extension slot (`$extensions` with reverse-DNS keys, which tools
MUST preserve), free `.tokens.json` validation, and existing generators — so we stop
maintaining alias resolution, cycle detection and theme merging ourselves. What is
genuinely *not* covered and stays ours: (a) the direction we actually need — binding an
abstract token to a **pre-existing, externally-owned** custom property such as
`--lumo-primary-color`, since Style Dictionary, Terrazzo and design.md all *mint* names
from token paths rather than adopt yours (they make the mapping *configurable*, but the
mapping table itself is your data); (b) the CSS → tokens bootstrap direction, for which
we found no maintained tool; (c) our assertion vocabulary and the "assert the var() chain,
never the raw value" verification semantics, which no token spec addresses. Net advice:
keep the mapping, move it out of bespoke Markdown into `$extensions` under a key like
`com.vaadin.dramafinder` (or, if the mapping must stay in DESIGN.md for the human gate,
define it as the same shape so it round-trips), and treat 26-of-63 value collisions as
evidence *for* the standard, since that is exactly the aliasing the spec models.

---

## 1. Status of the DTCG format spec

**It is a W3C Community Group report, explicitly NOT a W3C Standard.** The published
snapshot states verbatim:

> "This specification was published by the Design Tokens Community Group. It is not a
> W3C Standard nor is it on the W3C Standards Track. […] This document was published by
> the DTCG as a Candidate Recommendation following the definitions provided by the W3C
> process. […] While not a W3C recommendation, this classification is intended to clarify
> that, after extensive consensus-building, this specification is intended for
> implementation. **This specification is considered stable.** Further updates will be
> provided in superseding specifications."
> — https://raw.githubusercontent.com/design-tokens/community-group/main/www/src/pages/TR/2025.10/format/index.html

Canonical identifiers and dates (evidenced):

| Item | Value |
| --- | --- |
| Canonical name | **Design Tokens Format Module 2025.10** |
| Header classification | **Final Community Group Report**, **28 October 2025** |
| Canonical URL | `https://www.designtokens.org/TR/2025.10/format/` |
| Licence regime | W3C Community Final Specification Agreement (FSA) |
| Companion report | **Design Tokens Resolver Module 2025.10**, Final Community Group Report, 28 October 2025, `https://www.designtokens.org/TR/2025.10/resolver/` |

Sources: header/status blocks of
https://raw.githubusercontent.com/design-tokens/community-group/main/www/src/pages/TR/2025.10/format/index.html
and
https://raw.githubusercontent.com/design-tokens/community-group/main/www/src/pages/TR/2025.10/resolver/index.html

The published version table (https://raw.githubusercontent.com/design-tokens/community-group/main/README.md):

| name | url | date | status |
| --- | --- | --- | --- |
| 2025.10 | www.designtokens.org/TR/2025.10/ | 2025-10-28 | **Stable** |
| third-editors-draft | …/TR/third-editors-draft/ | 2025-07-21 | Draft |
| second-editors-draft | …/TR/second-editors-draft/ | 2022-06-14 | Draft |
| first-editors-draft | …/TR/first-editors-draft/ | 2021-09-23 | Draft |
| preview | …/TR/drafts/ | — | Experimental |

That README adds: *"tools can use the date as a version number to signify compliance.
For example: `2025.10`."*

**Churn warning (important for "do not build on something still churning").** The
markdown sources on the `main` branch are **not** 2025.10 — they are the *preview*
draft. `technical-reports/format/index.html` sets `specStatus: 'CG-DRAFT'`,
`isPreview: true`, and its Status section says: *"⚠️ This is a preview draft of in
progress changes. Do not refer to this document directly, and do not implement anything
in this document."*
(https://raw.githubusercontent.com/design-tokens/community-group/main/technical-reports/format/index.html).
So: cite/pin **2025.10**; the branch markdown is where the next version is being written.
The preview already contains post-2025.10 additions that we should *not* rely on (e.g.
JSON-Pointer `$ref` aliasing and `$root` group tokens appear in the preview markdown).
Official JSON Schemas for the stable version exist in-repo at
https://raw.githubusercontent.com/design-tokens/community-group/main/schemas/src/2025.10/format.json
and `.../schemas/src/2025.10/resolver.json`, published as
`https://www.designtokens.org/schemas/2025.10/format.json`
(https://raw.githubusercontent.com/design-tokens/community-group/main/schemas/README.md).

## 2. The token file format, and how `colors.primary` is expressed

Design token files are **JSON**, MIME type `application/design-tokens+json` (or
`application/json`), recommended extensions **`.tokens`** or **`.tokens.json`**
(https://raw.githubusercontent.com/design-tokens/community-group/main/technical-reports/format/file-format.md).

Rules, from §5–§6 of the 2025.10 report
(https://raw.githubusercontent.com/design-tokens/community-group/main/www/src/pages/TR/2025.10/format/index.html):

- "An object with a `$value` property is a token. […] The parent object's key is the
  token name." Name and value are both **required**.
- "A group is identified as a JSON object that does NOT contain a `$value` property."
  An object that has both `$value` and children is invalid — "Tools MUST report this as
  an error."
- `$type` may be set on a token **or inherited from the closest parent group** that
  declares it; if neither, and the value is not a reference, the token "MUST be
  considered invalid". "Tools MUST NOT attempt to guess the type of a token by
  inspecting the contents of its value."
- All spec-defined properties are `$`-prefixed, so token/group names MUST NOT begin with
  `$`, and **MUST NOT contain `{`, `}` or `.`** (because `.` is the alias path
  separator). ← directly constrains how we may name tokens.
- Groups are "arbitrary and tools SHOULD NOT use them to infer the type or purpose of
  design tokens" — i.e. `colors.primary` is a path, not a typed namespace.

`colors.primary` is a `primary` token inside a `colors` group, and is referenced as
`{colors.primary}` — the exact syntax our plan already uses in assertions. Concrete
example fetched from
https://raw.githubusercontent.com/design-tokens/community-group/main/technical-reports/format/aliases.md
(identical text in the 2025.10 report, §7.1.1):

```json
{
  "colors": {
    "blue": {
      "$value": {
        "colorSpace": "srgb",
        "components": [0, 0.4, 0.8],
        "hex": "#0066cc"
      },
      "$type": "color"
    }
  },
  "semantic": {
    "primary": {
      "$value": "{colors.blue}",
      "$type": "color"
    }
  }
}
```

Note the 2025.10 **color** value is a structured object (`colorSpace`, `components`,
optional `hex` fallback), not a hex string — a migration cost if we assume hex strings.
Dimensions are likewise objects: `{"value": 3, "unit": "rem"}` (§5.1, Example 2, same URL).

## 3. Aliasing / references — and multi-level chains

**Curly-brace syntax, and chains are explicitly legal.** From §7 of the 2025.10 report
(https://raw.githubusercontent.com/design-tokens/community-group/main/www/src/pages/TR/2025.10/format/index.html,
source markdown: https://raw.githubusercontent.com/design-tokens/community-group/main/technical-reports/format/aliases.md):

- Syntax: `"$value": "{group.token}"`. "The curly brace syntax […] always resolves to
  the `$value` property of the target token." It "can ONLY target complete tokens
  (objects with `$value` properties)".
- §7.2.2 **Chained References**: *"Aliases MAY reference other aliases. In this case,
  tools MUST follow each reference until they find a token with an explicit value."* The
  spec's own example resolves `semantic.link` → `semantic.brand` → `base.primary` — the
  same shape as our `--aura-accent` → `--aura-blue-500` → raw value.
- §7.2.3 **Circular References**: "References MUST NOT be circular"; tools "MUST detect
  and report this as an error affecting all tokens in the circular chain".
- Tools "SHOULD preserve references and therefore only resolve them whenever the actual
  value needs to be retrieved" — i.e. the standard *wants* the indirection preserved,
  which is precisely what our "assert the chain, not the raw value" rule needs.
- If a token's value is a reference and it has no `$type`, "its type is the resolved type
  of the token being referenced" (§5.2.2).

Also relevant to our 26-of-63 duplicate-value finding: the DTCG colour report states
"A design token's value MAY be a *reference* to another token. The same value MAY have
multiple names or aliases"
(https://raw.githubusercontent.com/design-tokens/community-group/main/technical-reports/color/token-naming.md).
Shared values are the expected outcome of an alias layer, not a data-quality problem.

The preview draft additionally introduces JSON-Pointer `$ref` aliasing (property-level
references such as `#/base/blue/$value/components/0`). **Preview only — do not build on
it** (see §1 churn warning).

## 4. Theme / mode variants — first-class, in a companion report

**Not in the format module; yes in the Resolver module, which is stable at the same
date.** The Resolver Module 2025.10 abstract: *"This specification extends the format and
describes a method to work with design tokens in multiple contexts (such as 'light mode'
and 'dark mode' color themes)."*
(https://raw.githubusercontent.com/design-tokens/community-group/main/www/src/pages/TR/2025.10/resolver/index.html)

Its stated motivation
(https://raw.githubusercontent.com/design-tokens/community-group/main/technical-reports/resolver/introduction.md)
names theming (light/dark/high-contrast), sizing and accessibility modes, and the
combinatorial-explosion problem: *"This format describes a mechanism for deduplicating
all repeat values of tokens across all contexts as well as enumerating all permutations
of contexts."*

Mechanism (https://raw.githubusercontent.com/design-tokens/community-group/main/technical-reports/resolver/syntax.md):
a separate **resolver document** with root properties `name`, `version` (MUST be
`2025.10`), `description`, `sets`, `modifiers`, `resolutionOrder` (required), `$schema`.

- A **set** is a collection of DTCG tokens via `sources` (inline tokens and/or `$ref` to
  files). Sources merge in array order; "if a token is declared multiple times, the last
  occurrence in the array will be the final value. Tools MUST respect array ordering."
- A **modifier** is a set with a `contexts` map for conditional values, plus an optional
  `default`. A modifier "SHOULD have two or more `contexts`"; 0 contexts MUST error. A
  modifier MAY reference a set but "MUST NOT reference any other modifier".

The spec's own light/dark example (same URL):

```json
{
  "$schema": "https://www.designtokens.org/schemas/2025.10/resolver.json",
  "modifiers": {
    "theme": {
      "description": "Color theme",
      "contexts": {
        "light": [{ "$ref": "theme/light.json" }],
        "lightHighContrast": [
          { "$ref": "theme/light.json" },
          { "$ref": "theme/dark-high-contrast.json" }
        ],
        "dark": [{ "$ref": "theme/dark.json" }],
        "darkHighContrast": [
          { "$ref": "theme/dark.json" },
          { "$ref": "theme/dark-high-contrast.json" }
        ]
      },
      "default": "light"
    }
  }
}
```

So the **established practice is separate files per theme, orchestrated by a resolver
document** — not `$extensions`, and not a per-token variant map. Terrazzo documents
exactly this file layout (`tokens/foundation/*` always applied, `tokens/themes/light|dark.tokens.json`
overriding) and notes the theme files are deliberately *incomplete*, containing only
aliases that get resolved against the foundation
(https://raw.githubusercontent.com/terrazzoapp/terrazzo/main/www/src/pages/docs/guides/resolver-contexts.md).
It also states resolvers are "the successor to legacy modes that are only understood by
Terrazzo/Cobalt"
(https://raw.githubusercontent.com/terrazzoapp/terrazzo/main/www/src/pages/docs/guides/resolvers.md).

Style Dictionary has **no** resolver/mode concept: a grep of its entire docs tree for
"resolver" returns only an unrelated `resolveReferences` utility mention, and its theming
story is separate builds per brand/theme (`examples/advanced/multi-brand-multi-platform`).
Verified in the cloned repo at commit `9a9cca0` (2026-06-21), package `style-dictionary@5.5.0`.

## 5. Official extensibility mechanism — yes, `$extensions`

§5.2.3 of the 2025.10 report
(https://raw.githubusercontent.com/design-tokens/community-group/main/www/src/pages/TR/2025.10/format/index.html;
source: https://raw.githubusercontent.com/design-tokens/community-group/main/technical-reports/format/design-token.md):

> "The optional `$extensions` property is an object where tools MAY add proprietary,
> user-, team- or vendor-specific data to a design token. […] The keys SHOULD be chosen
> such that they avoid the likelihood of a naming clash […] The **reverse domain name
> notation** is recommended for this purpose. Tools that process design token files
> **MUST preserve any extension data they do not themselves understand**."

Groups may also carry `$extensions` (§6.3 group properties table, same URL). An editor's
note explicitly widens it beyond vendors: *"The extensions section is not limited to
vendors. All token users can add additional data in this section for their own purposes."*

Two caveats to weigh before we move `cssVariables` there:

1. The spec says teams "SHOULD restrict their usage of extension data to optional
   meta-data that is not crucial to understanding that token's value." A property-name
   binding is not needed to understand the *value*, so it fits — but it *is* crucial to
   our verifier, so a token file stripped of extensions would silently disarm strict mode.
   Our loader must therefore treat a missing extension as an `error`, exactly as the plan
   already specifies for a missing `cssVariables` entry.
2. Interop is one-way: other tools must preserve it, but none will act on it.

Example shape (spec's canonical form, §5.2.3 Example 6), adapted:

```json
{
  "colors": {
    "primary": {
      "$type": "color",
      "$value": "{colors.blue.500}",
      "$extensions": {
        "com.vaadin.dramafinder": { "cssVariable": "--lumo-primary-color" }
      }
    }
  }
}
```

## 6. Token → CSS custom property generation

**Yes — three independent maintained implementations, all with configurable naming.**

### Style Dictionary (`style-dictionary@5.5.0`, commit `9a9cca0`, 2026-06-21)

- Format name: **`css/variables`** (enum `formats.cssVariables` = `'css/variables'`,
  https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/lib/enums/formats.js;
  implementation https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/lib/common/formats.js).
- The `--` prefix is not hardcoded per token; it comes from the CSS branch of
  `createPropertyFormatter`, which builds each line as
  `${indentation}${prefix}${token.name}${separator} ` with `formatDefaults.prefix = '--'`
  for `format: css`
  (https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/lib/common/formatHelpers/createPropertyFormatter.js).
  `prefix`, `indentation`, `separator`, `suffix` are all overridable via `formatting`.
- Path → name transform: **`name/kebab`**, literally
  `kebabCase([config.prefix].concat(token.path).join(' '))`
  (https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/lib/common/transforms.js).
  So `colors.primary` → `--colors-primary`, and a platform-level `prefix` prepends.
  Alternatives shipped: `name/camel`, `name/snake`, `name/constant`, `name/pascal`,
  `name/human`
  (https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/docs/src/content/docs/reference/Hooks/Transforms/predefined.mdx).
  Fully configurable — you may register your own `name`-type transform.
- The `css` transformGroup is `[attribute/cti, name/kebab, time/seconds, html/icon,
  size/rem, color/css, asset/url, fontFamily/css, cubicBezier/css, …/shorthand]`
  (https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/lib/common/transformGroups.js).
- Directly relevant options on `css/variables`
  (https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/docs/src/content/docs/reference/Hooks/Formats/predefined.md):
  - **`outputReferences`** — "Whether or not to keep references (a -> b -> c) in the
    output", also accepting a per-token function. This is what emits
    `--accent: var(--blue-500)` instead of flattening to a raw value — i.e. it reproduces
    our export's `value: "var(--aura-blue-500)"` shape.
  - **`outputReferenceFallbacks`** — emits `var(--x, <fallback>)`.
  - **`selector`** — "Override the root CSS selector. When a string array is provided,
    the styles will be nested within the specified selectors in order". Defaults to
    `:root` (verified in `formats.js`). This is how `[data-theme="dark"]` scoping is
    produced — but the *theme partitioning itself* is your build config, not a spec
    concept, in SD.
- DTCG support: "**As of version 4**, Style Dictionary has first-class support for the
  DTCG format", with a documented caveat: "*the latest format 2025.10 does not have full
  support yet in Style Dictionary — This is a work in progress in v5*" (issue #1590)
  (https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/docs/src/content/docs/info/DTCG.mdx).
  The 5.x changelog shows 2025.10 support landing piecemeal: 5.3.0 "Add support for DTCG
  v2025.10 structured color format in color transformers" (all 14 colour spaces), 5.3.1
  shadow/border shorthands, and a later entry adding the 2025.10 object-form `dimension`
  type "while remaining backwards compatible … for dimension tokens using string values"
  (https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/CHANGELOG.md).
  `usesDtcg` is normally auto-detected
  (https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/docs/src/content/docs/reference/config.md).
  Conversion helpers `convertToDTCG` / `convertJSONToDTCG` / `convertZIPToDTCG` exist
  (same CHANGELOG), but they convert *Style Dictionary v3 JSON*, not CSS.

### Terrazzo `@terrazzo/plugin-css@2.5.0` (repo commit `8753761`, 2026-07-27)

Package description: "Convert DTCG design tokens JSON into CSS variables […] Convert your
modes into CSS media queries for complete flexibility"
(https://raw.githubusercontent.com/terrazzoapp/terrazzo/main/packages/plugin-css/package.json).
This is the DTCG-native option and the closer fit for us:

- **`variableName: (token: TokenNormalized) => string`** — "Function that takes in a token
  ID and returns a CSS variable name. Use this if you want to prefix your CSS variables,
  or rename them in any way." Default idiom in the docs is
  `(token) => token.id.replace(/\./g, "-")`. Also `subValueVariableName`, `exclude`,
  `legacyHex`, `transform(token, { permutation })`
  (https://raw.githubusercontent.com/terrazzoapp/terrazzo/main/www/src/pages/docs/integrations/css.md).
  Because it is an arbitrary function, a lookup table from token path to an
  externally-owned property name (`colors.primary` → `--lumo-primary-color`) is a
  supported configuration, not a fork.
- Aliases are emitted as `var()` indirection, not flattened:
  `const transformAlias = (token) => \`var(${transformName(token)})\``
  (https://raw.githubusercontent.com/terrazzoapp/terrazzo/main/packages/plugin-css/src/transform.ts).
- **Resolver contexts → arbitrary selectors** via `permutations`, mapping an `input` such
  as `{ mode: "dark" }` to any wrapper CSS. The documented example produces both
  `@media (prefers-color-scheme: dark) { :root { … } }` and `[data-theme="dark"] { … }`
  from one token set (same `css.md`). It also documents the cascade subtlety we would
  otherwise have to discover the hard way: aliases "must be redeclared" inside a mode
  selector, "otherwise they are referencing the old value in the parent scope".
- Automatic gamut handling: out-of-sRGB colours are downconverted with extra
  `@media (color-gamut: p3 | rec2020)` blocks (same `css.md`) — meaning the *computed*
  value of one custom property can legitimately differ by display, which a raw-value
  assertion strategy would have failed on.

### `@google/design.md@0.4.0` — the tool our plan already adopts

This matters most for the reinvention question: the design.md CLI **already exports both
DTCG and CSS custom properties**.

- `export` command formats are a closed enum `['css-tailwind', 'json-tailwind',
  'tailwind', 'dtcg', 'css-vars']`, with `--prefix` for "Optional CSS custom property
  prefix for css-vars output"
  (https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/commands/export.ts).
  README: `npx @google/design.md export --format dtcg DESIGN.md > tokens.json`, described
  as "W3C Design Tokens Format Module"
  (https://raw.githubusercontent.com/google-labs-code/design.md/main/README.md).
- Its DTCG emitter is typed against 2025.10 ("DTCG Value Types (W3C Design Tokens Format
  Module 2025.10)", `DtcgColorValue { colorSpace: 'srgb'; components; hex? }`,
  `$type`/`$value`/`$description`, `$schema`)
  (https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/dtcg/spec.ts).
  Note it models neither `$extensions` nor aliases in its emitted types.
- Its css-vars emitter mints its **own** names: `color-${cssSafe(name)}`,
  `${group}-${cssSafe(name)}` for spacing/rounded, where `cssSafe` collapses dots to
  hyphens because "a literal dot makes a browser drop the declaration"
  (https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/css-vars/handler.ts).
  It emits resolved hex values, not `var()` chains.
- **No theme/variant concept**: no `dark`/theme handling in
  `packages/cli/src/linter/model/spec.ts` (grep for "dark" returns nothing), and the
  README's theme mentions are all Tailwind output. So DESIGN.md alone cannot express our
  light/dark axis — the Resolver can.

**Conclusion for Q6:** token-path → `--custom-property` generation, with documented and
configurable naming, is solved three times over. Our `cssVariables` key duplicates the
*generation* concern. What none of them do is *adopt* a property name that a third-party
theme already owns; they all assume they are the namer, and offer a hook
(`variableName`, a custom `name` transform, `--prefix`) as the escape valve.

## 7. Reverse direction: existing CSS custom properties → token file

**NOT ESTABLISHED.** No maintained, spec-aware CSS-custom-property → DTCG importer was
found. Specifically checked:

- **Style Dictionary**: no CSS parser. Its `parsers` hook is the documented extension
  point — "You can define custom parsers to parse design token files. This allows you to
  define your design token files in *any* language you like as long as you can write a
  parser for it", matching on a file-path regex and returning an object
  (https://raw.githubusercontent.com/style-dictionary/style-dictionary/main/docs/src/content/docs/reference/Hooks/parsers.md).
  A CSS→tokens bootstrap would be ours to write, but it has a sanctioned socket. No
  `postcss` / `css-tree` dependency exists in `package.json`.
- **Terrazzo**: has a real import command, but it is **Figma-only** —
  `packages/cli/src/import/` contains only `figma/`, and CLI help reads "import [path]
  Import from a Figma Design file" (verified in clone at commit `8753761`; docs page
  https://raw.githubusercontent.com/terrazzoapp/terrazzo/main/www/src/pages/docs/guides/import-from-figma.md).
- **design.md**: `export` only; the enum above has no import format.
- **Web search** (2026-07-31) for CSS→token importers surfaced only the opposite
  direction — palette/Tailwind-config *generators* that emit DTCG, and PostCSS plugins
  (`postcss-design-tokens`, `postcss-design-token-utils`) that *consume* a token file to
  produce CSS. `postcss-design-tokens` is documented as accepting "only Style Dictionary
  version 3 files". None reads an existing stylesheet to produce a token file.
  Searched result set included https://www.npmjs.com/package/postcss-design-tokens and
  https://github.com/saneef/postcss-design-token-utils (titles/URLs only — both hosts are
  blocked for fetching here, so the *capability* claims are second-hand and unverified).
- GitHub repository search for "css custom properties to design tokens DTCG import"
  returned **0 results**.

There is a DTCG ecosystem inventory —
`design-tokens/community-group` Discussion #312, "Tools that support the DTCG format" —
which would be the authoritative place to check for an importer, but GitHub HTML pages are
blocked in this environment, so it was **not checkable**.

Practical note for our bootstrap step: the raw material we hold
(`{name, value, kind, definedIn, scope}`) already contains everything a DTCG file needs
except `$type` mapping and colour-space conversion — `kind: "color"` → `$type: "color"`,
`value: "var(--aura-blue-500)"` → `$value: "{…}"` alias, `scope: [data-theme="dark"]` →
a resolver `modifiers.theme.contexts.dark` source file. That transformation is a
one-directional script we own; nothing standard performs it.

---

## Access limitations

Verified blocked in this environment (do not retry):

- `https://tr.designtokens.org/...` — connection fails (000). The rendered spec site is
  unreachable; findings above come from the authoring repository instead, which for a
  repo-authored spec is at least as primary.
- `https://www.designtokens.org/...` — not fetched (same host family as the above;
  content obtained from the repo's `www/` sources, which are what that site publishes).
- `https://styledictionary.com/...` — connection fails (000). Style Dictionary docs were
  read from `docs/src/content/docs/**` in its repo, which is that site's source.
- `https://github.com/...` HTML pages — 403. Consequence: Discussion #312 (ecosystem tool
  list) and issue #1590 (Style Dictionary 2025.10 support progress) could **not** be read;
  #1590's existence and framing come from Style Dictionary's own docs page.
- `https://www.npmjs.com/...` — 403.
- `WebFetch` on the above hosts — 403.
- GitHub MCP tools (`get_file_contents`, etc.) — "Access denied: repository … is not
  configured for this session. Allowed repositories: jcgueriaud1/dramafinder". Unusable
  for third-party repos.

What worked: `raw.githubusercontent.com` (200), **`git clone --depth 1` through the agent
proxy** (worked for all four repos below, and is the reason directory listings and version
metadata in this document are reliable rather than guessed), and `WebSearch`.

Path-probe results worth recording (404 = does not exist, so a later reader need not retry):

| Path | Result |
| --- | --- |
| `design-tokens/community-group/main/README.md` | 200 |
| `design-tokens/community-group/master/README.md` | 200 (same content; `main` is canonical) |
| `design-tokens/community-group/main/technical-reports/format/README.md` | **404** |
| `design-tokens/community-group/main/CHANGELOG.md` | **404** (a changelog exists only for the resolver: `technical-reports/resolver/CHANGELOG.md`) |
| `design-tokens/community-group/main/technical-reports/2025.10/format/index.html` | **404** |
| `design-tokens/community-group/2025.10/technical-reports/format/index.html` | **404** (no version branch) |
| `design-tokens/community-group/main/technical-reports/format/{terminology,file-format,design-token,groups,aliases,types,composite-types}.md` | 200 (all seven) |
| `design-tokens/community-group/main/www/src/pages/TR/2025.10/{format,resolver}/index.html` | 200 — **this is where the stable published spec lives** |
| `design-tokens/community-group/main/schemas/src/2025.10/{format,resolver}.json` | 200 |
| `amzn/style-dictionary/main/README.md`, `package.json` | 200 (redirects to `style-dictionary/style-dictionary`; the org moved) |

Repositories inspected, at these exact commits (all cloned 2026-07-31):

| Repo | Commit | Date | Version |
| --- | --- | --- | --- |
| `design-tokens/community-group` | `16c902d9327c18290e956a21130c445f1b88c40f` | 2026-07-30 | format & resolver 2025.10 published; `main` markdown = preview draft |
| `style-dictionary/style-dictionary` | `9a9cca0413c51be65030067c611124194babdf54` | 2026-06-21 | `style-dictionary@5.5.0` |
| `terrazzoapp/terrazzo` | `8753761bad19bbf0cabb9da33924211114715eae` | 2026-07-27 | `@terrazzo/plugin-css@2.5.0` |
| `google-labs-code/design.md` | `9bf8eae67128b6cc55ad9bf86665767deb4c11cd` | 2026-07-27 | `@google/design.md@0.4.0` |

Not verified / out of scope: whether Style Dictionary issue #1590 is now closed; whether
any tool listed in Discussion #312 implements a CSS importer; runtime behaviour of any of
the above (nothing was executed — all findings are from specification text, source code
and first-party documentation).
