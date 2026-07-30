# Does Claude Design's `design-sync` tooling already do design-to-structured-artifact extraction?

Research date: 2026-07-30. Question: before building a `design-to-spec` step that reads a Claude
Design project export (HTML + `_ds/` with `_ds_manifest.json`, `tokens/*.css`, `styles.css`,
`_ds_bundle.js`) and emits a structured JSON "view spec" (components by role + accessible name,
plus design tokens), does Anthropic's `design-sync` tooling already do this?

## Source tiers used in this document

Claims below are tagged so you can weigh them:

- **[OFFICIAL]** — Anthropic-published documentation or an Anthropic-owned GitHub repo.
- **[REGISTRY]** — npm registry API responses (authoritative for "does this package exist").
- **[OBSERVED]** — real artifacts committed to public third-party GitHub repos. Authoritative that
  the artifact *exists and looks like this*, not that the format is *supported*.
- **[UNOFFICIAL-VERBATIM]** — public "system prompt leak" repositories that contain what appear to
  be verbatim copies of Anthropic's bundled skill source and tool descriptions. **Not authoritative
  and not a stable contract.** Used only where it corroborates the ground-truth export in hand, and
  always flagged. Do not cite these to justify a product decision on its own.

---

## 1. Is there a publicly documented `design-sync` CLI or `/design-sync` skill?

**Yes — a `/design-sync` skill, documented in exactly one sentence-length table row. No public CLI,
no installable package.**

**[OFFICIAL]** `/design-sync` appears in the Claude Code commands reference, verbatim:

> `/design-sync [hint]` — **Skill.** Convert your repo's React design system and upload it to
> [Claude Design](https://claude.ai/design), so designs it produces use your real components.
> Optionally name the design system, for example `/design-sync Acme DS`. A first-time sync verifies
> every component and can take a few hours on a large repo. Available on the Anthropic API; on
> Amazon Bedrock, Google Cloud's Agent Platform, Microsoft Foundry, and Claude Platform on AWS the
> underlying tool can't reach claude.ai, so the command is unavailable

Source: <https://code.claude.com/docs/en/commands> (also served as
<https://code.claude.com/docs/en/commands.md>).

**[OFFICIAL]** A companion command, verbatim from the same table:

> `/design-login` — Authorize design-system access for `/design-sync` with your claude.ai account

Source: <https://code.claude.com/docs/en/commands>

**Installability / packaging:**

- It is a **bundled skill** shipped inside Claude Code, not an add-on. The commands reference links
  the word "Skill" to the bundled-skills section of the skills page.
  Source: <https://code.claude.com/docs/en/commands>, <https://code.claude.com/docs/en/skills>
- **[REGISTRY]** No Anthropic-published npm package exists. `@anthropic-ai/design-sync`,
  `design-sync-cli`, `@anthropic-ai/claude-design` and `claude-design-sync` all return
  `{"error":"Not found"}` from the npm registry.
  Source: `https://registry.npmjs.org/@anthropic-ai%2Fdesign-sync` (and the three other names).
- **[REGISTRY]** The npm names `design-sync` and `@design-sync/cli` **are taken by unrelated
  third-party Figma tooling** — `design-sync` is published by `neil-armstrong-instil` ("Syncing a
  figma design system into code via a 'pull' model") and the `@design-sync/*` scope by
  `salamaashoush`. Neither is Anthropic's. Do not confuse them.
  Source: <https://registry.npmjs.org/-/v1/search?text=design-sync&size=15>
- **[OFFICIAL]** GitHub code search for `org:anthropics design-sync` returns **0 results**. The
  skill's source is not in any public Anthropic repository.
  Source: GitHub code search API, query `org:anthropics design-sync`.
- **[OFFICIAL]** The public Claude Code CHANGELOG never mentions design-sync (only 4 unrelated
  matches for the substring "design": a frontend-design plugin tip, `/dataviz`, `/claude-api` agent
  design patterns, and a Grep redesign).
  Source: <https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md>
- **[OFFICIAL]** None of the 17 weekly "What's new" digests (`2026-w13` … `2026-w29`) mention
  `/design-sync`.
  Source: <https://code.claude.com/docs/en/whats-new/index> and the individual `.md` pages.
- The skill is **not present in this environment's Claude Code install** (filesystem-wide search for
  `*design*sync*` found nothing), consistent with the platform gating quoted above.

**Documentation depth: NOT ESTABLISHED beyond that one table row.** There is no `/design-sync`
reference page, no schema page, no tutorial in the official docs. `support.claude.com` and
`claude.com` help/product pages could **not** be verified from this environment — the outbound proxy
rejected CONNECT to both hosts (403), so the Claude Design help-centre articles
("Get started with Claude Design", "Set up your design system in Claude Design") were **not read**.
Anything they may contain is **NOT ESTABLISHED** here.

### Distinguish from Anthropic's *other*, unrelated design tooling

**[OFFICIAL]** `anthropics/knowledge-work-plugins` publishes a `design` plugin with `/design-system`,
`/handoff` (skill `design-handoff`), `/critique`, `/accessibility`, etc. These are **prompt-only
skills**: `design-handoff` generates prose handoff documentation from a Figma URL or a screenshot.
They have **no connection to Claude Design projects, `_ds_manifest.json`, or `design-sync`** — they
never read a Claude Design export and emit no machine-readable artifact.
Sources:
<https://raw.githubusercontent.com/anthropics/knowledge-work-plugins/main/design/README.md>,
<https://raw.githubusercontent.com/anthropics/knowledge-work-plugins/main/design/skills/design-handoff/SKILL.md>,
<https://raw.githubusercontent.com/anthropics/knowledge-work-plugins/main/design/skills/design-system/SKILL.md>

---

## 2. Which direction does it sync? **(decisive — it pushes upward only)**

**Local component library → Claude Design project. It is a one-way push. It does not produce a
structured local artifact from a project.**

**[OFFICIAL]** The command reference says it plainly: "Convert your repo's React design system and
**upload it to** Claude Design, so designs it produces use your real components."
Source: <https://code.claude.com/docs/en/commands>

**[UNOFFICIAL-VERBATIM]** The bundled skill's own frontmatter, from a leak repo, agrees:

> `description: Push a React design system to claude.ai/design. This runs a converter that bundles
> the real component code (from Storybook or a bare package) and uploads it. Use when the user runs
> /design-sync or says "sync my design system to Claude Design".`

Source: `https://raw.githubusercontent.com/asgeirtj/system_prompts_leaks/main/Anthropic/Claude%20Code/bundled-skills/design-sync/SKILL.md`

**[UNOFFICIAL-VERBATIM]** The underlying `DesignSync` MCP tool exposes read methods, but they exist
only to compute an upload diff — not to export a design. Its full method list is
`list_projects`, `get_project`, `list_files`, `get_file`, `create_project`, `finalize_plan`,
`write_files`, `delete_files`, `register_assets`, `unregister_assets`. Constraints that matter to us,
quoted:

> `list_files` — list paths in a project. Use this to build the structural diff.
>
> `get_file` — read one remote file's content. Capped at 256 KiB. Only call this when you need to
> compare content for a specific component the user named.
>
> `list_projects` — … **Filtered to writable projects only.**
>
> `get_project` — … Use to verify a `--project <uuid>` target is actually
> `type: PROJECT_TYPE_DESIGN_SYSTEM` before pushing — **that type is immutable at creation**, so
> pushing to a regular project never makes it a design system.

Source: `https://raw.githubusercontent.com/Piebald-AI/claude-code-system-prompts/main/system-prompts/tool-description-designsync.md`
(captured at Claude Code v2.1.178)

Three consequences for us:

1. There is **no "pull project" / "export view" method** at all. Per-file `get_file` at 256 KiB is a
   diff primitive, not an export path.
2. The tool is scoped to **design-system projects** (`PROJECT_TYPE_DESIGN_SYSTEM`) the user can
   **write** to. A regular Claude Design project containing a page/view is out of scope, and the type
   cannot be changed after creation.
3. Public commentary claiming `/design-sync` is a "two-way bridge" is **wrong or at best loose**. That
   framing appears widely on SEO aggregator sites (e.g. `skills-hub.ai`, `aiforanything.io`,
   `pasqualepillitteri.it`, `arte.itlibra.com`) and is contradicted by the official command text and
   the skill's own description. Treat those pages as unreliable.

**Verdict on question 2: `/design-sync` does not help us. It only pushes local code upward.**

---

## 3. Is `_ds_manifest.json` documented and stable? Are `x-import` / `x-dc` / `helmet` /
`component-from-global-scope` / `hint-size` documented?

### `_ds_manifest.json`

**NOT ESTABLISHED as documented — it is a server-generated internal artifact. There is no published
schema.**

- **[OFFICIAL]** The string `_ds_manifest` appears nowhere in the Claude Code docs (checked
  `commands.md`, `skills.md`, `llms.txt`, all `whats-new` pages) nor in the CHANGELOG.
- **[UNOFFICIAL-VERBATIM]** It is **generated by the Claude Design app server-side**, not by the CLI:

  > `register_assets` — legacy: register preview cards explicitly. The Design System pane now builds
  > its card index from each preview HTML's first-line `<!-- @dsCard group="…" -->` comment
  > (**compiled into `_ds_manifest.json` by the app's self-check**), so explicit registration is no
  > longer required for /design-sync uploads.

  Source: `.../tool-description-designsync.md` (as above)

- **[UNOFFICIAL-VERBATIM]** The design agent inside Claude Design is explicitly forbidden from
  writing it: "Do NOT write `_ds_bundle.js`, `_ds_manifest.json`, `_adherence.oxlintrc.json`, or a
  barrel `index.js` — **those are generated automatically**."
  Source: `https://raw.githubusercontent.com/asgeirtj/system_prompts_leaks/main/Anthropic/claude-design.md`

- **[UNOFFICIAL-VERBATIM]** This also explains our ground-truth `"source": "design-sync-cli"`. The
  CLI never writes the manifest; it drops a fence file that the server reads to stamp `source`.
  From `lib/emit.mjs`, with its own comment:

  ```js
  // Fence so consumers don't read a half-uploaded tree (see the Upload section of the skill).
  // The app's self-check reads `by` to set the manifest's `source`.
  writeFileSync(join(OUT, '_ds_needs_recompile'), JSON.stringify({ by: 'design-sync-cli' }));
  ```

  Source: `https://raw.githubusercontent.com/asgeirtj/system_prompts_leaks/main/Anthropic/Claude%20Code/bundled-skills/design-sync/lib/emit.mjs`

  So `source` is **provenance of the upload path**, not a format version: `"design-sync-cli"` means
  "uploaded by the CLI, then compiled by the app"; `"spa"` means "authored in the canvas".

- **[OBSERVED]** Corroborated independently by a third party who hit this in production, in the header
  comment of a hand-rolled workaround script:

  > Generate `_ds_manifest.json` (the claude.ai/design card index) from a built bundle. **The
  > package-build CLI does NOT emit this — normally the claude.ai/design app's server-side self-check
  > regenerates it.** That self-check does NOT fire on raw DesignSync file uploads, so after a CLI
  > sync the project keeps its previous manifest and the pane shows stale / "file not found" cards.

  Source: <https://raw.githubusercontent.com/RoboFinSystems/roboledger-app/main/.design-sync/gen-manifest.mjs>

  This is a useful stability signal in both directions: the format is consistent enough that people
  reimplement it, and unsupported enough that they *have to*.

#### Observed schema (de facto, from a real export in the wild)

**[OBSERVED]** GitHub code search finds **890 `_ds_manifest.json` files** in public repos, so this is
a widely emitted artifact. One inspected in full —
<https://raw.githubusercontent.com/nvidia-isaac/video_to_data/main/docs/chord/_ds/basic-nvidia-design-system-f939d4e7-d0bb-4994-9e26-a471f04510c4/_ds_manifest.json>
— note the `_ds/<slug>-<uuid>/` layout matches ours:

| Key | Type | Element shape |
|---|---|---|
| `namespace` | string | `"DesignSystem_6d5263"` — the `window.<ns>` global |
| `components` | list[18] | `{name, sourcePath}` |
| `startingPoints` | list[7] | `{name, path, previewPath, kind, section, subtitle, viewport}` |
| `cards` | list[35] | `{path, group, viewport, subtitle, name}` |
| `templates` | list[0] | (empty here) |
| `globalCssPaths` | list[7] | `"tokens/fonts.css"` — plain strings |
| `tokens` | list[165] | `{name, value, kind, definedIn}` + optional `scope`, `annotation` |
| `themes` | list[1] | `{selector, label}` e.g. `{"selector":"[data-mode=\"kaizen\"]","label":"Kaizen"}` |
| `fonts` | list[0] | |
| `brandFonts` | list[4] | `{family, status, tokens[], path}` |
| `source` | string | `"spa"` here; `"design-sync-cli"` in ours |

Observed `tokens[].kind` values: `color`, `font`, `other`, `radius`, `shadow`, `spacing`.
44 of 165 tokens carried a `scope` (theme-scoped), matching our `[data-theme="dark"]` observation.

**[OBSERVED]** Cross-checking four more real manifests (`jk-dot-com/jk.com-design-system`,
`utensils/nxv`, `elvonlabs/website`, plus the nvidia one) shows the key set is **stable but not
fixed**: all four carry the same 11–12 keys, but `hasThumbnailHtml` is present in three and absent in
the nvidia one. `themes` is `[]` in three and populated in one. A hand-authored test fixture
(`oalanicolas/claude-design-premium/fixtures/builder-ds/_ds_manifest.json`) has only 5 keys and no
`source`. **Conclusion: code defensively — treat every key as optional and every list as possibly
absent or empty.**

### `x-dc`, `helmet`, `x-import`, `component-from-global-scope`, `hint-size`

**NOT ESTABLISHED as documented anywhere public.** Zero hits in Anthropic's docs or public repos.

**[UNOFFICIAL-VERBATIM]** They are described in detail only inside the Claude Design agent's own
system prompt. Quoting the parts that pin down our export's semantics:

> Build every design as a **Design Component ("DC")**: a single `Name.dc.html` file that opens
> directly in a browser and can be imported by other DCs.

> **Props metadata** (`d_props_json`, optional) — the `data-props` JSON on the
> `<script data-dc-script>` tag (never on `<x-dc>`).

> For a script with no exports that registers itself globally, use `component-from-global-scope`
> instead of `component`: pass the **tag name** for a `customElements.define('my-tag', …)` web
> component, or the **global name** for a `window.Foo = …` React component… The name may be a dotted
> path (`NS.Button` → `window.NS.Button`).

> always set `hint-size` (placeholder + min-size while streaming)

> **Design-system components**: Load the design-system bundle in each DC's `<helmet>` (de-duped by
> URL), then mount its components with
> `<x-import component-from-global-scope="Namespace.Component" hint-size="…">children</x-import>`

> **Styling — inline styles only.** No stylesheets, no CSS classes, no "base styles" or design-token
> setup… `style="…"` compiles to a React style object; pseudo-states use `style-hover` /
> `style-active` / `style-focus` / `style-before` / `style-after`.

> The runtime file `support.js` is written for you; never write it.

Source: `https://raw.githubusercontent.com/asgeirtj/system_prompts_leaks/main/Anthropic/claude-design.md`

That "inline styles only" rule matters for us: in a `.dc.html` view, the layout values are inline
`style` attributes, **not** token references — so `tokens[]` from the manifest and the actual style
values in the view are two separate extraction problems.

---

## 4. Does anything extract a **page or view** into a structured machine-readable description?

**No. Everything published is oriented around syncing a design *system*, not describing a view.**

- **[OFFICIAL]** `/design-sync` converts a component library upward. Its unit of work is a component
  (`components/<group>/<Name>/{.jsx,.d.ts,.prompt.md,.html}`), never a rendered view.
  Source: <https://code.claude.com/docs/en/commands>
- **[UNOFFICIAL-VERBATIM]** The skill enumerates every artifact it produces — `_ds_bundle.js`,
  `_vendor/`, `styles.css`, `fonts/`, `tokens/`, `_ds_bundle.css`, `<Name>.d.ts`, `<Name>.prompt.md`,
  `<Name>.html` preview card, `_ds_sync.json`. **None is a view/page description**, and there is no
  role/accessible-name extraction anywhere in it.
  Source: `.../bundled-skills/design-sync/SKILL.md`
- **[OFFICIAL]** `anthropics/knowledge-work-plugins`' `design-handoff` is the closest thing in spirit
  — its "What to Include" list literally names "ARIA labels and roles", "Design token references",
  "Component variants and states". But it is a **prose prompt** aimed at Figma/screenshots that emits
  Markdown for humans, with no parser, no schema, and no awareness of Claude Design exports.
  Source: <https://raw.githubusercontent.com/anthropics/knowledge-work-plugins/main/design/skills/design-handoff/SKILL.md>
- **[OBSERVED]** GitHub code search for tooling that *reads* `_ds_manifest.json` returns ~50 hits,
  and on inspection they all **write or validate manifests** for hand-authored design systems
  (e.g. `RoboFinSystems/roboledger-app/.design-sync/gen-manifest.mjs`,
  `whimzyLive/nightshift-ai/.claude/skills/nightshift-design/scripts/check-tokens.mjs`). **None
  parses a `.dc.html` view into a component/role/label tree.** Code search for `x-import` +
  `component-from-global-scope` + parse/extract returns only copies of Anthropic's own starter
  components, no parser.

**This is the gap. The view-spec extraction we planned does not exist publicly, in any form.**

---

## 5. Documented way to render `.dc.html` headlessly, or export it to plain static HTML?

**NOT ESTABLISHED — no documented path for either.** But two concrete, useful facts:

**(a) The DC runtime does render standalone, and it needs the network.**

**[OBSERVED]** A real `support.js` from a public export
(<https://raw.githubusercontent.com/nvidia-isaac/video_to_data/main/docs/chord/support.js>, 58 KB)
opens with:

```
// GENERATED from dc-runtime/src/*.ts — do not edit. Rebuild with `cd dc-runtime && bun run build`.
```

The `dc-runtime` source is **not published**. The compiled runtime fetches its dependencies from
unpkg at runtime, with pinned SRI:

```js
var BABEL_URL     = "https://unpkg.com/@babel/standalone@7.26.4/babel.min.js";
var REACT_URL     = "https://unpkg.com/react@18.3.1/umd/react.production.min.js";
var REACT_SRI     = "sha384-DGyLxAyjq0f9SPpVevD6IgztCFlnMF6oW/XQGmfe+IsZ8TqEiDrcHkMLKI6fiB/Z";
var REACT_DOM_URL = "https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js";
```

**Practical implication for our headless plan:** a headless render needs reachable
`unpkg.com` (or a local mirror serving those exact URLs with matching SRI hashes — note this
research environment's proxy blocks `unpkg.com`). Budget for that.

**(b) The runtime mounts into ordinary DOM, so post-render extraction is viable.**

**[OBSERVED]** From the same `support.js`, its `parseDcDocument` replaces the custom element with a
plain host div before mounting React:

```js
const dc = doc.querySelector("x-dc");
const hostEl = doc.createElement("div");
hostEl.id = "dc-root";
dc.replaceWith(hostEl);
```

So after render the `x-` elements are gone and the tree is real DOM — computed styles, ARIA roles and
accessible names are readable. There is **no documented "export to static HTML"** step; you would be
snapshotting the post-mount DOM yourself.

**(c) There is prior art for the harness, but for cards, not views.**

**[UNOFFICIAL-VERBATIM]** `/design-sync`'s own screenshot step uses Playwright + Chromium against a
local static server:

```js
try { ({ chromium } = await import('playwright')); }
catch { console.error('playwright not installed — npm i playwright (in .ds-sync/) first'); process.exit(2); }
const browser = await chromium.launch(process.env.DS_CHROMIUM_PATH ? { executablePath: process.env.DS_CHROMIUM_PATH } : {});
const page = await browser.newPage({ viewport: { width: 900, height: 700 } });
try { await page.clock.setFixedTime(new Date('2024-05-15T12:00:00Z')); } catch {}
```

Source: `https://raw.githubusercontent.com/asgeirtj/system_prompts_leaks/main/Anthropic/Claude%20Code/bundled-skills/design-sync/package-capture.mjs`

It renders **component preview cards** and produces **PNGs for a model to eyeball**, not a structured
description of a view. The `page.clock.setFixedTime` trick is worth copying for determinism.

**[UNOFFICIAL-VERBATIM]** Claude Design's *user-facing* exports are documented (in the agent prompt)
as PDF, PPTX (editable and screenshot), video, and OBJ/GLB — none of which is "static HTML without
`x-` elements".
Source: `https://raw.githubusercontent.com/asgeirtj/system_prompts_leaks/main/Anthropic/claude-design.md`

---

## 6. Anything that maps design tokens to CSS custom properties, or validates an implementation
against a Claude Design project?

**Token mapping: yes, but in the wrong direction.** **Validation: exists but is internal and
unpublished.**

**Token mapping (wrong direction).** **[UNOFFICIAL-VERBATIM]** `/design-sync` *produces*
`tokens/*.css` as CSS custom properties from the repo and uploads them; the manifest's `tokens[]` is
then the server's parse of those files. The consuming direction — manifest → CSS custom properties in
*our* project — is not implemented anywhere published. Note also the invariant that constrains any
consumer:

> rendered designs receive only `styles.css`'s transitive `@import` closure. Any real component CSS
> (`_ds_bundle.css`) must be `@import`ed from `styles.css`

Source: `.../bundled-skills/design-sync/SKILL.md`. This confirms `globalCssPaths` + `styles.css` is
the intended entry point for token resolution, which is convenient for us.

**Validation / adherence.** **[UNOFFICIAL-VERBATIM]** An adherence mechanism exists inside Claude
Design but is unpublished and not addressable from outside:

> `<Name>.d.ts` … the sibling `.d.ts` is what gives a component its props contract, **adherence
> rules**, and starting-point eligibility

> Do NOT write `_ds_bundle.js`, `_ds_manifest.json`, **`_adherence.oxlintrc.json`** … those are
> generated automatically.

> it opens `path` in the user's tab bar, waits for it to load, then forks a background verifier
> subagent that reviews the output (console errors, screenshot, layout, JS probing, **design-system
> adherence**, recreation fidelity)

Source: `https://raw.githubusercontent.com/asgeirtj/system_prompts_leaks/main/Anthropic/claude-design.md`

So there is a generated oxlint config for adherence and an in-canvas verifier subagent — but
**[OFFICIAL]** neither `_adherence.oxlintrc.json` nor any adherence API is documented, and there is
no published way to run either against our own implementation.

**[OBSERVED]** Community substitutes exist and are worth reading as prior art, though they are
third-party and validate *authoring*, not *our implementation*: e.g. a token-drift gate that
cross-checks `_ds_manifest.json` `tokens[]` against `tokens/colors.css` —
<https://raw.githubusercontent.com/whimzyLive/nightshift-ai/main/.claude/skills/nightshift-design/scripts/check-tokens.mjs>

---

## BOTTOM LINE

**REUSE: no**

Nothing in our extraction step is already built: `/design-sync` is a one-way **push** of a local
React component library up into a Claude Design design-system project, and its `DesignSync` tool
exposes only `list_files`/`get_file` diff primitives scoped to *writable design-system* projects —
there is no project-to-local export, no view/page description, and no role-plus-accessible-name
extraction anywhere in Anthropic's published surface ([code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands)).
The only partially transferable pieces are prior art rather than reusable code: the Playwright +
Chromium + fixed-clock harness pattern that `/design-sync` uses to screenshot component cards, the
`globalCssPaths` → `styles.css` `@import` closure as the correct token-resolution entry point, and
the fact that the DC runtime replaces `<x-dc>` with a plain `<div id="dc-root">` so post-mount DOM
gives us real computed styles and ARIA — at the cost of needing reachable `unpkg.com` for pinned
React 18.3.1 / Babel 7.26.4.

**The manifest format is stable enough to code against defensively, but not to depend on.** It is a
server-generated internal artifact with **no published schema** — `_ds_manifest.json` is compiled by
the Claude Design app's self-check, which is why our export carries `"source": "design-sync-cli"`
(the CLI only drops a `_ds_needs_recompile` fence saying `{"by":"design-sync-cli"}`), and the design
agent is explicitly forbidden from writing it. Five real manifests inspected across public repos show
a consistent key set and element shapes matching our ground truth exactly, yet with real variance
(`hasThumbnailHtml` present in some and not others, `themes` often `[]`, `source` sometimes absent) —
so treat every key as optional, pin nothing to `source`, validate on read, and expect to re-verify
after any Claude Design release. `x-dc`, `helmet`, `x-import`, `component-from-global-scope` and
`hint-size` are **NOT ESTABLISHED — appears internal/undocumented**: they are described only in
leaked agent prompts, so any parser we write against them is reverse-engineering an unversioned
format and should be isolated behind a single adapter module with golden-file tests over our real
export.
