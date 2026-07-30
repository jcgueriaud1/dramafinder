# Design pipeline plan: design → spec → implementation → verification

This document is the durable reference for the design-verification pipeline
shipping in DramaFinder 1.2.0. Progress tracking lives in GitHub issues; this
file describes the design and the *why* behind each decision. All decisions
below were resolved in a structured design interview; deferred items are
listed explicitly at the end.

## The problem

Given a design (an image, or a Claude Design HTML export), an agent must be
able to (1) implement or update a Vaadin view from it, and (2) verify the
implementation — without pixel-perfect comparison, which is too flaky to
drive an agent loop.

## Approach in one paragraph

Never compare image-to-image. A structured **view spec** (JSON) is extracted
once from the design and becomes the pivot artifact: the implementation
target, and the input to a **deterministic verification rung** (a spec
interpreter running DramaFinder/Playwright assertions against the live app).
A separate **VLM judge rung** compares screenshots against the *original
design* — not the spec — so extraction errors are catchable. Project-wide
design vocabulary lives in a root **DESIGN.md** ([google-labs-code
format](https://github.com/google-labs-code/design.md)) extended with a
`cssVariables` mapping. DramaFinder ships **mechanism only**: a stateless
verifier with a compact JSON output contract, plus skills that run one pass
and read one result. The **caller owns all policy**: loops, thresholds, fix
strategy.

## Decisions and rationale

### D1 — Placement: everything in this repo

Spec schema, `design-to-spec` skill, verifier, and `visual-verdict` skill all
live in parttio/dramafinder. The spec's assertion vocabulary is coupled to
the DramaFinder element API, the judge extends the existing
`vaadin-playwright-screenshot` capture machinery, and the marketplace
distribution already exists. Extracting the framework-neutral parts later is
cheap; premature generality now is not.

### D2 — Deterministic rung: spec interpreter, not generated assertions

A new engine in `org.vaadin.addons.dramafinder.agent`:

```java
VerificationResult result = DesignSpecVerifier.verify(page, Path.of("design/specs/order-form.json"));
```

The interpreter loads the spec JSON and executes assertions generically. The
spec stays the single source of truth (no generated-code drift), and failures
come back as structured diffs, which is what makes agent fixes surgical
instead of guesswork.

**Hybrid boundary:** the interpreter cannot know how to *reach* a view.
Generated `AgentVerifyIT` classes (the existing batch pattern on
`VisualVerificationTest`) own navigation, login, and state seeding, then call
the verifier per view/state. Navigation is generated code; design assertions
are interpreted spec.

### D3 — Assertion vocabulary v1

Three categories, deliberately bounded:

1. **Structure** — component present (role + accessible name), containment,
   sibling order. Builds on what `ComponentSnapshot` already walks.
2. **Layout intent at the attribute level** — assert `alignment="end"` on a
   layout component, not measured geometry. Component attributes encode most
   layout intent structurally; geometric bounding-box relations are the long
   tail and are **deferred**.
3. **Visual properties, token-only** — assertions reference DESIGN.md token
   paths (`{colors.primary}`); the interpreter asserts the CSS custom
   property *chain*, never raw values. Theme-agnostic by construction: Lumo,
   Aura, and fully custom themes are all just data in the `cssVariables`
   mapping. No raw-value/Delta-E fallback in v1.

### D4 — DESIGN.md as project-level source of truth

- Root `DESIGN.md` in the consuming project, google-labs format: YAML front
  matter = normative tokens, prose = rationale. Their CLI (`lint`, `diff`)
  is usable in the pipeline for free.
- **`cssVariables` extension key** (legal — unknown top-level keys are
  tolerated by the spec) maps token paths to the project's CSS custom
  properties: `colors.primary: --my-brand-blue`. This is what makes token
  assertions executable per project.
- **Strict mode:** a token with no `cssVariables` mapping is an `error`, not
  a silent fallback. Every verification failure therefore has exactly two
  routable causes: a DESIGN.md change (new/updated token, missing mapping)
  or an implementation fix.
- **Human-gated writes:** skills may *propose* DESIGN.md content (bootstrap)
  or additions (diff for review), never silently edit. That gate is what
  makes it a source of *truth*.
- Markdown never carries executable assertions; view specs are JSON, and any
  human-readable summary is generated *from* JSON, never parsed from prose.

### D5 — Element identity

- **Labeled elements:** abstract **role + accessible name**
  (`{"role": "button", "label": "Save"}`). Roles come from a small abstract
  vocabulary (`button`, `text-input`, `select`, `grid`, `heading`, `link`,
  `container`, …); the interpreter owns the role → acceptable-realizations
  mapping (`button` → `vaadin-button` or native `<button>`). The design
  expresses intent; the implementation keeps freedom of realization
  (`vaadin-card` vs styled `div` vs `section`). The mapping is
  package-private — not an extension point in v1.
- **Disambiguation** via `within` scoping (element or group reference).
  Residual ambiguity = `error`, never first-match.
- **Unlabeled containers (groups):** a design never shows "a div"; it shows a
  visual grouping whose *contents* have names. A group is declared by its
  named members plus a spec-internal `id` (used only in messages and `within`
  references — never looked up in the DOM):

  ```json
  { "type": "group", "id": "order-summary",
    "contains": [ {"role":"heading","label":"Summary"},
                  {"role":"button","label":"Submit"} ],
    "assertions": [ {"property":"border-color","token":"{colors.outline}"} ] }
  ```

  The interpreter locates members, computes their lowest common ancestor, and
  evaluates visual assertions **existentially over the ancestor chain** from
  the LCA up to (excluding) the next enclosing group's boundary: "some
  element wrapping exactly this group carries this border."
- **Resolution order:** groups first, then scoped element assertions.
  Circular references (a group member scoped `within` that same group) are
  rejected at spec load.
- **Known limits, accepted for v1:** groups with zero named members are not
  expressible on this rung (VLM judge territory); the existential chain-walk
  can in principle false-pass on a coincidental wrapper — see the spike in
  the build order, which gates this mechanism on evidence.
- Side effect embraced: unlabeled elements are unaddressable, so the verifier
  pushes implementations toward proper accessible names — consistent with
  DramaFinder's label-first API philosophy.

### D6 — Viewports: one spec per ground-truth rendering

- `viewport: {width, height}` is **required** spec metadata; the interpreter
  sets the Playwright viewport itself rather than trusting the harness.
- **Image path:** the screenshot is the only ground truth at one implied
  width → one spec per screenshot. Multiple mockups of one view (mobile +
  desktop) = multiple specs, verified in the same run.
- **Claude Design HTML path:** the design is renderable, so real ground truth
  exists at any width. Breakpoints are an explicit **opt-in list** on the
  extraction invocation (default: desktop 1920×1080 only), each producing its
  own spec. Rationale for opt-in: Claude Design responsiveness is often
  incidental Tailwind reflow, not designed intent — extracting a mobile spec
  by default would elevate an accident to a requirement.
- No breakpoint semantics in the schema; it only ever knows "this spec, this
  viewport."

### D7 — Output contract and score; no loop in this repo

`DesignSpecVerifier` is a pure function from (page, spec) to result. It
writes `verification-result.json` into `target/agent-report/<view>/` (the
directory `AgentReporting` already owns):

```json
{
  "schemaVersion": 1,
  "view": "order-form",
  "viewport": {"width": 1920, "height": 1080},
  "score": 87,
  "passed": 26, "failed": 4, "errors": 2,
  "assertions": [
    { "id": "a-014", "status": "fail",
      "target": "button 'Save' (within group order-summary)",
      "property": "background-color",
      "expected": "{colors.primary} → --my-brand-blue",
      "actual": "--my-error-color",
      "hint": "…" }
  ]
}
```

- **Tri-state:** `fail` = implementation problem; `error` = spec/DESIGN.md
  problem (unmapped token, unresolvable or ambiguous target, invalid group).
  This is the two-way routing made machine-readable.
- **Score** = `100 × passed / (passed + failed)` — implementation quality
  only. The contract states the score is **only valid when `errors == 0`**;
  the documented (not enforced) caller policy is: resolve errors first, then
  threshold on score. Unweighted; no weight field exists in v1.
- **JUnit semantics:** `verify()` stays green on assertion failures. Red
  means broken machinery (app unreachable, spec unparseable, unknown
  `schemaVersion`). Anything else would impose threshold=100 and steal the
  caller's policy.
- **No loop, threshold, or retry logic anywhere in this repo** — not in
  Java, not in skill text. The caller runs verify → reads score → fixes →
  re-runs, with its own iteration cap and threshold.
- No per-assertion screenshots: the deterministic diff is self-sufficient;
  screenshots enter only at the judge rung (the existing failure-hook
  screenshot on *test* failure is unchanged).

### D8 — VLM judge: the `visual-verdict` skill

Framework-neutral skill (no Vaadin in its description): compares a generated
screenshot against reference image(s) and returns a strict JSON verdict.
Based on the agreed skeleton, with three amendments:

1. **Threshold is a parameter** (`pass_threshold`, documented default 90);
   the loop section is one line: the verdict returns to the caller, iteration
   policy is the caller's. Same mechanism/policy split as D7.
2. **Checklist-grounded score:** the judge first enumerates discrete yes/no
   items (derived from the view spec's groups/structure when available, plus
   standard intent items: hierarchy, grouping, emphasis, "reads as the same
   design"), answers each with rationale, then computes
   `score = 100 × yes/total`. Output shape unchanged (`score`, `verdict`,
   `category_match`, `differences[]`, `suggestions[]`, `reasoning`);
   `differences[]` = the failed items. This is what keeps a VLM score from
   coin-flipping around the threshold.
3. **Optional `view_spec` input:** when present, each difference carries
   `route: "implementation" | "spec"` — spec route meaning "the design shows
   X, the spec never asserted X," which sends the fix to the extraction
   skill, not the implementer. When absent, the skill is fully standalone.

Judging is performed by the **calling agent itself** (it is a capable VLM
with the images in context) — no API keys, no pinned model in the skill.
Accepted trade-off: verdicts vary with the host model; acceptable because
the judge is a workflow gate, not a benchmark. Reproducible judging belongs
to external eval harnesses that pin their own judge model.

Pixel-diff tooling (pixelmatch overlays) is a **debug aid only** — used to
localize hotspots and convert them into `differences[]`/`suggestions[]`,
never authoritative.

**Inputs:** implementation screenshot via the existing
`VisualVerificationTest.shot()` machinery (reuse
`vaadin-playwright-screenshot`, don't reinvent capture); reference = the
design image, or — Claude Design path — the design HTML rendered and
screenshotted at the same viewport as the spec, giving a true side-by-side.

The judge compares against the **design, not the spec**: the spec is a lossy
extraction, and judging against it would make extraction errors invisible
(implementation faithfully matches a misreading, everything passes, result
looks wrong). Judging against the design makes this rung a genuinely
independent check that can *catch spec errors* — the `spec-gap` route.

### D9 — The `design-to-spec` skill

Two extraction paths converging on one spec schema:

- **Claude Design HTML path (ground truth, built first):** parse the exported
  markup — DOM hierarchy and computed styles *are* the design. The LLM's job
  is translation (generic div/Tailwind structures → role vocabulary), not
  perception. Deterministic input, debuggable, low hallucination risk.
- **Image path (lossy, built second):** VLM extraction; the emitted spec is a
  reviewable draft. Nearest-token mapping happens here at extraction time —
  the raw hex/px seen in the image is matched to the closest DESIGN.md token,
  keeping the interpreter token-only.

**First-run bootstrap (no DESIGN.md present):** the skill scans the project's
theme CSS (and/or the Claude Design export), proposes `DESIGN.md` including
the `cssVariables` mapping, and **pauses for human review** before
extraction proceeds. On later runs it may propose token additions as a diff,
never a silent edit.

Outputs land in `design/specs/<view>[-<viewport>].json` — deliberately not
under `src/test/resources`: specs are design artifacts consumed by tests and
by non-Java tooling (the judge, external evals) alike.

### D10 — Skills inventory after this work

| Skill | Role | Notes |
| --- | --- | --- |
| `design-to-spec` | new | extraction + DESIGN.md bootstrap |
| `verify-ui-against-design` | new/rewritten | one verification pass: run `AgentVerifyIT`, read result JSON, route fail→code / error→spec-or-DESIGN.md; **no loop** |
| `visual-verdict` | new | generic screenshot-vs-reference verdict, D8 |
| `vaadin-playwright-screenshot` | existing | capture half of the judge rung |
| `vaadin-playwright-test` | existing | unchanged |

### D11 — Testing in this repo

- **Tier 1 — Interpreter ITs (JUnit, CI):** fixture Vaadin views under
  `src/test` paired with hand-written spec JSONs. One conforming view plus a
  family of **mutant views**, each broken in exactly one known way (wrong
  token on a button, missing component, swapped sibling order, border on the
  wrong wrapper, ambiguous duplicate labels, unmapped token, nested/one-member
  groups, coincidental-border wrappers). Each mutant asserts the verifier
  emits *precisely* the expected fail/error entry — the false-positive hunt
  made permanent. Reuse the existing IT harness that boots a test app.
- **Tier 2 — Schema contract tests (pure JUnit, CI):** spec and result JSONs
  validated against JSON Schema files that live in `docs/` and ship in
  `META-INF/dramafinder/` (agent-consumable, per the discoverability
  pattern). Malformed specs rejected at load with named reasons; the
  circular-group-reference rule is tested here.
- **No evals in dramafinder.** No model-calling harness, datasets, or eval
  code in this repo. External harnesses may consume the published fixtures
  for the closed-loop extraction eval: render fixture view → screenshot →
  `design-to-spec` → verify the extracted spec against the very view it came
  from → score should be ~100; any drop is extraction error by construction,
  no human labeling needed. Mutant screenshots likewise calibrate
  `visual-verdict` (should score below threshold). Note: PLAN.md's in-repo
  eval suite for `vaadin-playwright-test` predates this decision and may be
  relocated later for consistency (non-blocking).

### D12 — API surface, versioning, release

- **Java API, minimal and sealed:** public =
  `DesignSpecVerifier.verify(Page, Path)` +
  `verify(Page, String)` overload + `VerificationResult`. Role mapping,
  group resolver, token-chain evaluation all package-private — the group
  algorithm is exactly what the spike may force to change. Extension points
  deferred until demanded.
- **Schemas versioned from day one:** `"schemaVersion": 1` in spec and result
  JSONs; unknown versions rejected with a clear error.
- **Release: 1.2.0** — purely additive; projects not using specs are
  untouched. Skills and schemas ship in the same versioned artifact
  (marketplace plugin + `META-INF/dramafinder/`), so a project's DramaFinder
  version pins its entire pipeline contract — no skill/interpreter skew.

## Consuming-project conventions

```
DESIGN.md                      # root; google-labs format + cssVariables key; human-gated
design/specs/<view>[-<viewport>].json
target/agent-report/<view>/verification-result.json   # disposable, per run
src/test/java/.../AgentVerifyIT.java                  # generated navigation + verify calls
```

Three layers, three lifetimes: DESIGN.md (project-wide, stable) → view specs
(per design, regenerated when designs change) → results (per run,
disposable).

## Pipeline walkthrough

1. `design-to-spec` (bootstrap DESIGN.md if absent → human review) →
   `design/specs/*.json`
2. Agent implements/updates the view **from the spec**, not the image.
3. Caller loop: `verify-ui-against-design` runs `AgentVerifyIT` →
   `DesignSpecVerifier` writes result JSON → caller routes `error`→
   spec/DESIGN.md, `fail`→code, thresholds on score, decides iteration.
4. Once deterministic rung is green enough: `visual-verdict` compares the
   implementation screenshot against the original design (side-by-side on
   the Claude Design path), routes `spec-gap` findings back to extraction.

## Build order

| # | Deliverable | Gate / rationale |
| --- | --- | --- |
| 1 | Spec + result JSON Schemas, contract tests | everything else codes against them |
| 2 | **Group-resolution spike** on nasty fixtures | gate: no false positives; else tighten the existential walk to a sole-child-chain rule. Fixtures become permanent Tier-1 tests |
| 3 | `DesignSpecVerifier` + mutant fixture suite | depends on 1, 2 |
| 4 | `design-to-spec` — Claude Design path first, image path second | deterministic input first; needs 3 to self-check |
| 5 | `verify-ui-against-design` rewrite + `visual-verdict` | depends on 3 |
| 6 | Docs, `META-INF` packaging, marketplace entries, llms.txt update → **1.2.0** | |

## Deferred (explicitly out of v1)

- Geometric layout relations (bounding-box alignment/adjacency with
  tolerances)
- Raw-value comparison / Delta-E fallback for unmapped tokens
- Assertion weights (would arrive only as a schemaVersion bump, justified by
  session data on which assertion classes predict human rejection)
- Groups with zero named members (VLM judge covers them)
- Role-realization mapping as a public extension point
- Breakpoint semantics inside the schema (multi-viewport = multiple specs)
- Relocating the `vaadin-playwright-test` eval suite out of the repo
