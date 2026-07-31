# Research: is `google-labs-code/design.md` a usable design-token source of truth?

Date: 2026-07-31. Investigated version: format `alpha`, CLI `@google/design.md@0.4.0`.

## BOTTOM LINE

- **CLAIM A: holds** — the format is exactly as described: YAML front matter carries the normative tokens, the markdown body carries rationale, and the spec says so in those words ("The tokens are the normative values; the prose provides context for how to apply them", [docs/spec.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/docs/spec.md)).
- **CLAIM B: holds** — extensibility is explicit, not merely tolerated: the validator source states "The DESIGN.md schema is intentionally extensible (custom keys are allowed) … unrelated extension keys stay silent" ([unknown-key.ts](https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/linter/rules/unknown-key.ts)), and a real `lint` run with a `cssVariables` key returned 0 errors / 0 warnings (see [Empirical verification](#empirical-verification)).
- **CLAIM C: holds** — `lint` and `diff` exist under exactly those names in the shipped CLI, plus `export` and `spec` ([README.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/README.md), confirmed by running `node dist/index.js --help` from the published tarball).

**Recommendation.** Build on it, but do not expect it to carry the executable-token mechanism by itself. The format and its `lint` are a genuinely good fit for a project-level source of truth — Apache-2.0, real npm releases, machine-readable, and explicitly extensible so `cssVariables` is legal rather than merely unpunished. The two gaps that matter to us are that the format has **no concept of theme/mode variants** (light vs dark values for one token) and that `diff` is **blind to custom top-level keys** — so a rename inside our `cssVariables` block reports `regression: false`. Plan to own theming and the `cssVariables` diffing/verification ourselves, and treat the upstream CLI as a structural linter for the standard sections only.

---

## 1. Does the repo exist? What is it?

Yes. `https://github.com/google-labs-code/design.md` exists on the default branch `main`.

Self-description (verbatim, [README.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/README.md)):

> A format specification for describing a visual identity to coding agents. DESIGN.md gives agents a persistent, structured understanding of a design system.

Repository metadata (GitHub API via the `github` MCP server, `search_repositories` with `repo:google-labs-code/design.md`):

| Field | Value |
|:--|:--|
| Owner | `google-labs-code` (GitHub **Organization**) |
| Default branch | `main` |
| Language | TypeScript |
| License | Apache-2.0 (`spdx_id: Apache-2.0`) |
| Created | 2026-04-10 |
| Last push | 2026-07-27 |
| Last updated | 2026-07-31 |
| Stars / forks | 26,799 / 2,181 |
| Open issues | 30 |
| Archived | `false` |
| Homepage | `https://stitch.withgoogle.com/docs/design-md/specification` |

Corroborating file evidence:

- [LICENSE](https://raw.githubusercontent.com/google-labs-code/design.md/main/LICENSE) is the Apache License 2.0 text (201 lines).
- Source headers carry `Copyright 2026 Google LLC` under Apache-2.0 ([unknown-key.ts](https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/linter/rules/unknown-key.ts)).
- [package.json](https://raw.githubusercontent.com/google-labs-code/design.md/main/package.json) is a private monorepo root (`"name": "design-monorepo"`, `workspaces: ["packages/*"]`, `packageManager: bun@1.3.9`, turbo).

**Version / release status.** The format itself is at version `alpha`. From the README's own Status section:

> The DESIGN.md format is at version `alpha`. The spec, token schema, and CLI are under active development. Expect changes to the format as it matures.

The CLI has real, regular npm releases (`https://registry.npmjs.org/@google%2Fdesign.md`):

| Version | Published |
|:--|:--|
| 0.1.0 | 2026-04-21 |
| 0.1.1 | 2026-04-21 |
| 0.2.0 | 2026-05-26 |
| 0.3.0 | 2026-06-15 |
| **0.4.0** (`latest`) | 2026-07-27 |

**Maintenance signal: good.** Five releases across four months, most recent four days before this research, repo pushed the same day as the 0.4.0 publish, not archived, 30 open issues on a repo with 26.8k stars. Caveats: it is pre-1.0 and self-declared `alpha`, and the README disclaims security-program coverage — "This project is not eligible for the Google Open Source Software Vulnerability Rewards Program" ([README.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/README.md)). The npm metadata's `license` field is `null` even though the tarball ships an Apache-2.0 `LICENSE` file — cosmetic, but worth knowing if you run license scanners.

## 2. CLAIM A — the real format

**Holds.** [docs/spec.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/docs/spec.md) states it directly:

> A DESIGN.md file contains two parts: An optional YAML frontmatter, and a markdown body. The YAML front matter contains machine-readable design tokens. The markdown body sections provide human-readable design rationale and guidance. … **The tokens are the normative values; the prose provides context for how to apply them.**

Note the front matter is **optional** per the spec — a DESIGN.md may legitimately be prose-only.

### Concrete example (verbatim from the fetched README)

From [README.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/README.md):

```md
---
name: Heritage
colors:
  primary: "#1A1C1E"
  secondary: "#6C7278"
  tertiary: "#B8422E"
  neutral: "#F7F5F2"
typography:
  h1:
    fontFamily: Public Sans
    fontSize: 3rem
  body-md:
    fontFamily: Public Sans
    fontSize: 1rem
  label-caps:
    fontFamily: Space Grotesk
    fontSize: 0.75rem
rounded:
  sm: 4px
  md: 8px
spacing:
  sm: 8px
  md: 16px
---

## Overview

Architectural Minimalism meets Journalistic Gravitas. The UI evokes a
premium matte finish — a high-end broadsheet or contemporary gallery.

## Colors

The palette is rooted in high-contrast neutrals and a single accent color.

- **Primary (#1A1C1E):** Deep ink for headlines and core text.
- **Secondary (#6C7278):** Sophisticated slate for borders, captions, metadata.
- **Tertiary (#B8422E):** "Boston Clay" — the sole driver for interaction.
- **Neutral (#F7F5F2):** Warm limestone foundation, softer than pure white.
```

### Defined top-level keys — exactly nine

The canonical list is in the validator source as `SCHEMA_KEYS` ([packages/cli/src/linter/parser/spec.ts](https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/parser/spec.ts)):

```ts
/** Canonical top-level YAML keys per the DESIGN.md schema. */
export const SCHEMA_KEYS = [
  'version',
  'name',
  'description',
  'omitted',
  'colors',
  'typography',
  'rounded',
  'spacing',
  'components',
] as const;
```

This matches the schema block in [docs/spec.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/docs/spec.md). Note there is **no `elevation`/`shadow` token group** — Elevation & Depth is a prose-only section.

Token types (from [docs/spec.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/docs/spec.md)):

- **Color** — any valid CSS color string: hex, named, `rgb()`/`hsl()`/`hwb()`, `oklch()`/`oklab()`/`lch()`/`lab()`, `color-mix()`. Internally converted to sRGB for WCAG checks; original preserved for export. Hex `#RRGGBB` is the recommended default.
- **Dimension** — number plus unit; **valid units are only `px`, `em`, `rem`**.
- **Typography** — object with `fontFamily`, `fontSize`, `fontWeight`, `lineHeight`, `letterSpacing`, `fontFeature`, `fontVariation`.
- **Token reference** — `{path.to.token}`.
- **Component sub-tokens** — a closed set: `backgroundColor`, `textColor`, `typography`, `rounded`, `padding`, `size`, `height`, `width`.

Section order is normative-ish (present sections "should appear in the sequence listed"): Overview / Colors / Typography / Layout / Elevation & Depth / Shapes / Components / Do's and Don'ts. Out-of-order sections are a `section-order` **warning**; a duplicate section heading is an **error** that rejects the file.

### Formal schema or validator?

- **No published JSON Schema.** `schema.json` and `design.schema.json` both 404 at the repo root (see [Access limitations](#access-limitations) for the full probe list).
- **There is a real validator**: a Zod-based (`zod ^3.24.0`) parser + an 11-rule linter, shipped in `@google/design.md`. The spec document itself is generated, not hand-maintained: `docs/spec.md` opens with `<!-- Generated from spec.mdx + spec-config.ts | version: alpha --> <!-- Do not edit directly. Run bun run spec:gen to regenerate. -->`, and a machine-readable `spec-config.yaml` ships in the tarball. So the spec, the linter, and the CLI's `spec` command are driven from one source — a good sign for consistency.

### Is `colors.primary` a valid token path?

**Yes**, and it is the spec's own canonical example. `docs/spec.md`: "A token reference must be wrapped in curly braces, and contain an object path to another value in the YAML tree" — e.g. `{colors.primary-60}`. The spec also requires that "At least the `primary` color palette must be defined". A reference must resolve to a primitive (not a group), except inside `components`, where composite references like `{typography.label-md}` are permitted. Unresolvable references are the linter's only **error**-severity rule (`broken-ref`).

## 3. CLAIM B — are unknown top-level keys legal? **This is the important one**

**Holds — and it is explicit permission, not silence.** Three independent lines of evidence:

**(a) Explicit extensibility statement in validator source.** The doc comment on the `unknown-key` rule ([packages/cli/src/linter/linter/rules/unknown-key.ts](https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/linter/rules/unknown-key.ts)):

```ts
/**
 * Unknown key — warns when a top-level YAML key looks like a typo of a known
 * schema key. The DESIGN.md schema is intentionally extensible (custom keys
 * are allowed), so only close matches to known keys are reported; unrelated
 * extension keys stay silent.
 */
```

**(b) The README's rules table says the same.** [README.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/README.md): "`unknown-key` | warning | A top-level YAML key looks like a typo of a known schema key (e.g. `colours:` → `colors:`); **custom extension keys stay silent**".

**(c) No strict-mode rejection anywhere.** There is no `additionalProperties: false` (no JSON Schema at all), and the front-matter object is not Zod-`.strict()`. The only `unknownKeys: "strict"` occurrences in the bundle are inside vendored Zod internals, not applied to the DESIGN.md schema. The parser deliberately **retains** every unrecognised top-level key rather than stripping or rejecting it — `packages/cli/src/linter/parser/spec.ts` documents the field as "Raw YAML values for all top-level keys (known and unknown), used by lint rules", and `packages/cli/src/linter/parser/handler.ts` passes `rawValues: raw` straight through. The model layer then computes `unknownKeys` as a set difference against `SCHEMA_KEY_SET` and hands it to the rules.

### Important caveat: `cssVariables` is legal, but it is not *inert*

Two rules inspect unknown top-level keys, and one of them can bite:

1. **`unknown-key` (warning)** — fires only on a Levenshtein distance ≤ 2 against one of the nine schema keys. `cssVariables` is nowhere near any of them, so it stays silent. Verified empirically below.
2. **`token-like-ignored` (warning)** — fires when an unknown top-level key's value *looks like a design-token map*. From [token-like-ignored.ts](https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/linter/rules/token-like-ignored.ts) (as bundled), it recurses through the value and returns true if any key is one of `fontFamily`/`fontSize`/`fontWeight`/`lineHeight`/`letterSpacing`, or any string value (≤ 64 chars) matches `^#([0-9a-fA-F]{3,4}|[0-9a-fA-F]{6}|[0-9a-fA-F]{8})$` or `^-?\d*\.?\d+[a-zA-Z%]+$`.

Our proposed shape — `cssVariables: { colors.primary: "--my-brand-blue" }` — has values like `--my-brand-blue`, which match neither the hex nor the dimension regex, so the rule stays quiet. **But the shape matters**: if anyone ever writes literal hex values or dimensions under `cssVariables` (or any other custom key), lint starts emitting a warning telling them to move the values into a recognised section. Keep custom-key values to CSS custom-property *names*, never raw token values.

### The real problem with CLAIM B is downstream, not legality

Both the linter and the exporters treat custom keys as **inert data**. Concretely, verified by running the CLI:

- `export` **ignores** `cssVariables` entirely — the `token-like-ignored` message says so in its own words: "It will be silently ignored by export commands."
- `diff` **cannot see changes inside custom keys**. Renaming `--my-brand-blue` to `--renamed-blue` and diffing the two files produced all-empty `added`/`removed`/`modified` arrays for every group and `"regression": false`. The `diff` command's token comparison is hard-scoped to `colors`, `typography`, `rounded`, `spacing`, `components`.

So the mechanism is **legal and durable in the file**, but the upstream CLI provides **zero** verification coverage for it. Any guarantee that `cssVariables` stays in sync with the CSS is ours to build.

### Empirical verification

Run against `@google/design.md@0.4.0`, unpacked from `https://registry.npmjs.org/@google/design.md/-/design.md-0.4.0.tgz`, on Node v22.22.2.

Input (abridged) — a valid DESIGN.md with a custom `cssVariables` top-level key using dotted token paths as keys:

```yaml
---
version: alpha
name: Dramafinder
colors:
  primary: "#1A1C1E"
  secondary: "#6C7278"
typography:
  body-md: { fontFamily: Public Sans, fontSize: 16px, lineHeight: 1.6 }
rounded: { sm: 4px }
spacing: { md: 16px }
cssVariables:
  colors.primary: "--my-brand-blue"
  colors.secondary: "--my-brand-slate"
components:
  button-primary:
    backgroundColor: "{colors.primary}"
    textColor: "#ffffff"
---
```

`node dist/index.js lint DESIGN.md` output:

```json
{
  "findings": [
    { "severity": "info",
      "message": "Design system defines 2 colors, 1 typography scale, 1 rounding level, 1 spacing token, 1 component.",
      "rule": "token-summary" }
  ],
  "summary": { "errors": 0, "warnings": 0, "infos": 1 }
}
```

Exit code `0`. **Zero warnings** — `cssVariables` with dotted token-path keys is accepted silently.

Contrast case, to pin down the boundary: a custom key `brandColors: { extra: "#ABCDEF" }` produced

```json
{ "severity": "warning", "path": "brandColors",
  "message": "\"brandColors\" looks like a design-token map but is not a recognized schema key (colors, typography, spacing, rounded, components). It will be silently ignored by export commands. Rename it to a supported key or move its values under a recognized section.",
  "rule": "token-like-ignored" }
```

Also verified: nested colour tokens (`colors.background.light`) lint clean and are exported as flattened names.

## 4. CLAIM C — does the CLI exist, with `lint` and `diff`?

**Holds.** Installation ([README.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/README.md)):

```bash
npm install @google/design.md
# or
npx @google/design.md lint DESIGN.md
```

Two bin names ship, per the [npm metadata](https://registry.npmjs.org/@google%2Fdesign.md): `design.md` and `designmd`, both pointing at `dist/index.js`. The README documents a Windows gotcha — the `.md` suffix in the `design.md` bin name collides with the Markdown file association during command resolution, so on Windows use `npx -p @google/design.md designmd lint DESIGN.md`. Requires Node `>=18.0.0`. Dependencies: `citty`, `remark-*`/`unified`, `yaml`, `zod`.

Actual subcommands, from running the published binary (`node dist/index.js --help`):

```
USAGE design.md lint|diff|export|spec
```

| Command | Signature | What it does |
|:--|:--|:--|
| `lint` | `lint [--format json\|text] <FILE>` | Validates one DESIGN.md. Exit `1` if errors, else `0`. |
| `diff` | `diff [--format json\|text] <BEFORE> <AFTER>` | Compares **two DESIGN.md files**. Exit `1` on regression. |
| `export` | `export --format <fmt> [--prefix <p>] <FILE>` | Emits tokens in another format. |
| `spec` | `spec [--rules] [--rules-only] [--format markdown\|json]` | Prints the spec (for injecting into agent prompts). |

All commands accept `-` for stdin. There is also a programmatic API: `import { lint } from '@google/design.md/linter'`, returning `{ findings, summary, designSystem }`.

### What `lint` actually validates

Eleven rules, confirmed by running `spec --rules-only --format json`:

| Rule | Severity | Checks |
|:--|:--|:--|
| `broken-ref` | **error** | `{...}` references that don't resolve; circular refs; unknown component sub-tokens |
| `missing-primary` | warning | colors defined but no `primary` |
| `contrast-ratio` | warning | component `backgroundColor`/`textColor` pairs below WCAG AA 4.5:1 |
| `orphaned-tokens` | warning | color tokens never referenced by any component |
| `missing-typography` | warning | colors defined but no typography |
| `section-order` | warning | prose sections out of canonical order |
| `unknown-key` | warning | top-level key within edit distance 2 of a schema key (typo catcher) |
| `token-like-ignored` | warning | unknown top-level key holding token-like values |
| `token-summary` | info | token counts per section |
| `missing-sections` | info | optional sections (spacing, rounded) absent |
| `omitted-rules` | info | validates the `omitted:` declaration |

Note what `lint` does **not** do: it never looks at your source code. It is a self-consistency and accessibility checker for the DESIGN.md file alone. `broken-ref` is the only thing that can fail the build by default.

### What `diff` actually compares

**Two DESIGN.md files** — never a DESIGN.md against code. Verified output shape (real run, modifying `colors.secondary` and adding `rounded.lg`):

```json
{
  "tokens": {
    "colors":     { "added": [], "removed": [], "modified": ["secondary"] },
    "typography": { "added": [], "removed": [], "modified": [] },
    "rounded":    { "added": ["lg"], "removed": [], "modified": [] },
    "spacing":    { "added": [], "removed": [], "modified": [] },
    "components": { "added": [], "removed": [], "modified": [] }
  },
  "findings": { "before": {...}, "after": {...}, "delta": { "errors": 0, "warnings": 0 } },
  "regression": false
}
```

"Regression" means strictly *more lint errors or warnings in the after file* — it is not a semantic design judgement. And as noted under CLAIM B, the five token groups above are the **only** things compared: custom top-level keys are invisible to `diff`.

Minor implementation gap observed: `--format text` is advertised on both `lint` and `diff`, but `diff --format text` still emitted JSON in 0.4.0. Don't depend on the text renderer.

## 5. Theme / mode variants (light vs dark)

**No. The format has no concept of theme or mode variants.** This is the most consequential gap for us.

- The nine top-level keys contain nothing for modes, themes, schemes, or conditional values ([parser/spec.ts](https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/parser/spec.ts)).
- Grepping the full generated spec (`dist/spec.md` from the 0.4.0 tarball, identical in substance to [docs/spec.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/docs/spec.md)) for `theme`, `dark`, `light mode`, `color-scheme`, `mode` returns **no** hits describing variant semantics. The only "theme" hits are about Tailwind theme configs; the only "mode" hits are about layout *models*.
- Variance in the format is modelled **only for component states**, and even then by naming convention rather than structure: "A component may have a variant for different UI states such as active, hover, pressed … those variant components may be defined under a different but related key, for example, `button-primary`, `button-primary-hover`". There is no analogous facility for light/dark.

You *can* smuggle modes in by nesting or naming — and it lints clean — but the tooling flattens them into a single scope, which is the wrong output for dark mode. Verified: input

```yaml
colors:
  primary: "#1A1C1E"
  background:
    light: "#FFFFFF"
    dark: "#101214"
```

lints with 0 errors / 0 warnings, and `export --format css-vars` produces:

```css
:root {
  --color-primary: #1a1c1e;
  --color-background-light: #ffffff;
  --color-background-dark: #101214;
}
```

Both values land in the **same `:root` block** as differently-named properties. There is no `@media (prefers-color-scheme: dark)` or `[data-theme]` scoping anywhere in the emitter. The DTCG export does the same, emitting literal dotted names `"background.light"` and `"background.dark"` as sibling tokens. The css-vars handler even documents the flattening explicitly: "Nested tokens flatten to dotted keys (e.g. `background.light`); a literal dot makes a browser drop the declaration, so collapse dots to hyphens."

Conclusion: light/dark is entirely **outside the format's model**. If we need one token with two values, we must either (a) encode both as separately-named tokens and own the mode-scoping ourselves, or (b) maintain two DESIGN.md files. Neither is supported by upstream tooling.

## 6. Does it model token → CSS custom property?

**Partly — and not in the form our plan needs.** There are two exporters that emit CSS custom properties, both using a **fixed, tool-chosen naming scheme**. Neither accepts a user-supplied mapping.

1. **`export --format css-vars`** (present in 0.4.0 but **not documented in the README** — I found it in `--help`): emits a `:root { … }` block with names derived mechanically from the token group and name — `--color-primary`, `--spacing-md`, `--rounded-sm`. A `--prefix` option adds a global prefix. Verified by running it.
2. **`export --format css-tailwind`**: emits a Tailwind v4 `@theme { … }` block using Tailwind's own CSS-variable namespaces (`--color-*`, `--font-*`, `--text-*`, `--leading-*`, `--tracking-*`, `--font-weight-*`, `--radius-*`, `--spacing-*`) ([README.md](https://raw.githubusercontent.com/google-labs-code/design.md/main/README.md)).

Plus `json-tailwind` (Tailwind v3 `theme.extend` JSON), `tailwind` (alias), and `dtcg` (W3C Design Tokens Format Module).

So: **generating** CSS custom properties is in scope; **declaring which custom property a token maps to** is not. The name is a function of the token path, not a value you control. There is no `cssVariables`-like facility in the spec, and — as established in section 3 — a custom key holding such a mapping is legal but completely ignored by every exporter. Mapping `colors.primary` → `--my-brand-blue` is **outside the format's scope** and must be implemented by us.

If our CSS custom property names happened to follow the `--color-<token>` / `--spacing-<token>` / `--rounded-<token>` convention, we could use `export --format css-vars` directly and drop the `cssVariables` key entirely. That is worth considering before building a bespoke mechanism.

## Implications for our plan

1. `cssVariables` as a custom top-level key is **safe and explicitly sanctioned**. Keep its values to CSS property *names* only (never hex/dimension literals) so `token-like-ignored` stays quiet.
2. Upstream `lint` gives us free structural + WCAG-contrast + broken-reference checking on the standard sections. Gate CI on its exit code; only `broken-ref` is error-severity, so consider promoting selected warnings ourselves.
3. Upstream `diff` gives us **nothing** for `cssVariables`. If drift in that mapping is a risk we care about, we need our own comparator. The "for free" framing of CLAIM C is true for `lint`/`diff` as such, but not for our extension.
4. **Theming is our problem.** The format cannot express light/dark for one token, and the exporters actively flatten any workaround into one `:root` scope. Design our own theming layer; do not expect it to survive a round-trip through `export`.
5. The format is `alpha` with an explicit "expect changes" notice, at 0.4.0 after four months. Pin the CLI version and re-verify on upgrade. `SCHEMA_KEYS` growing a real `cssVariables` (or a theming key) in a future release would silently change the meaning of our file.
6. Consider whether `export --format css-vars --prefix …` or `dtcg` output could replace the bespoke mechanism. Aligning our CSS property names to the built-in convention would remove custom-key drift risk entirely.

## Access limitations

Most of the public web is blocked in this environment. What worked and what did not:

**Blocked / unreachable:**

| URL | Result |
|:--|:--|
| `https://github.com/google-labs-code/design.md` (HTML) | **403** (proxy policy denial) |
| `https://www.npmjs.com/package/@google/design.md` (HTML) | **403** |
| `https://stitch.withgoogle.com/docs/design-md/specification` (the repo's own homepage / hosted spec) | **000** — connection failed, never reached |
| `https://tr.designtokens.org/format/` | **403** (per proxy failure log) |
| `mcp__github__list_commits` on `google-labs-code/design.md` | Denied — "repository is not configured for this session. Allowed repositories: jcgueriaud1/dramafinder" |

**Worked:**

- `https://raw.githubusercontent.com/google-labs-code/design.md/main/<path>` — 200. Primary source for all repo-file citations.
- `https://registry.npmjs.org/@google%2Fdesign.md` and the `.tgz` tarball — 200 (`registry.npmjs.org` is in the proxy's no-proxy list). This let me unpack and **execute** the real CLI, which is the strongest evidence in this document.
- `github` MCP `search_repositories` — returned full repo metadata (stars, license, dates, default branch).

**Consequences — what is NOT ESTABLISHED:**

- **Commit history, contributor count, and release cadence beyond npm publish dates**: NOT ESTABLISHED (`list_commits` denied, GitHub HTML blocked). Maintenance judgement rests on npm publish timestamps plus the API's `pushed_at`/`archived` fields.
- **Issue and PR contents** (e.g. whether custom top-level keys or theming are under active discussion upstream): NOT ESTABLISHED. 30 open issues exist; their subjects are unknown.
- **The hosted specification at `stitch.withgoogle.com`**: NOT ESTABLISHED. It may contain normative statements — including about extensibility or theming — that differ from the in-repo `docs/spec.md`. All spec claims here derive from `docs/spec.md` on `main` and the generated `dist/spec.md` in the 0.4.0 tarball, which agree with each other.
- **Whether a JSON Schema is published anywhere outside the repo**: NOT ESTABLISHED. It is absent from the repo (probed 404: `schema.json`, `design.schema.json`, `SPEC.md`, `spec.md`, `CHANGELOG.md`, `DESIGN.md`, `examples/DESIGN.md`, `src/*.ts`, `tsconfig.json`, `AGENTS.md`, `CLAUDE.md`, `docs/spec.mdx`). Note the DTCG export references `https://www.designtokens.org/schemas/2025.10/format.json`, which is the W3C schema, not a DESIGN.md schema.

**Oddity worth flagging:** `raw.githubusercontent.com` returned byte-identical content for both `main` and `master` (13,148-byte README). The GitHub API reports `default_branch: main`, so `master` is presumably an alias or the proxy normalises it. All citations above use `/main/`.

## Sources

- `https://raw.githubusercontent.com/google-labs-code/design.md/main/README.md`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/docs/spec.md`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/package.json`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/LICENSE`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/parser/spec.ts`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/parser/handler.ts`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/linter/rules/unknown-key.ts`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/linter/rules/token-like-ignored.ts`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/src/linter/spec-config.yaml`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/packages/cli/package.json`
- `https://raw.githubusercontent.com/google-labs-code/design.md/main/CONTRIBUTING.md`
- `https://registry.npmjs.org/@google%2Fdesign.md`
- `https://registry.npmjs.org/@google/design.md/-/design.md-0.4.0.tgz` (unpacked; `dist/index.js`, `dist/linter/index.js`, `dist/spec.md`, `dist/spec-config.yaml`, `dist/linter/css-vars/*.d.ts`, `LICENSE`, `package.json`) — and executed on Node v22.22.2
- GitHub REST API via `github` MCP `search_repositories`, query `repo:google-labs-code/design.md`
