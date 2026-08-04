# Context

Ubiquitous language for Drama Finder — Playwright utility classes for Vaadin.

## Glossary

### Element

A typed Playwright wrapper around one Vaadin component (`ButtonElement`,
`GridElement`, …). An Element is identified by a **locator** and exposes the
component's behaviour and assertions. "Element" always means the Drama Finder
wrapper, never a raw DOM element — say **node** for that.

### Node

A raw DOM element in the browser. Distinct from an Element.

### Violation

An accessibility failure that axe-core is certain about. Reported by the user as
an **error**. The two words name the same thing; prefer *violation* in code and
Javadoc, since it is axe-core's own term.

### Incomplete

An accessibility finding axe-core could not decide, requiring human review.
Reported by the user as a **warning**. Common in Vaadin themes, where colour
contrast over gradients or images cannot be computed. Prefer *incomplete* in
code and Javadoc.

An Incomplete is *not* a lesser Violation — it is an unanswered question. A
finding is never both.

### Impact

axe-core's severity grading of a single finding: `minor`, `moderate`,
`serious`, `critical`. Orthogonal to the Violation/Incomplete split — an
Incomplete can be `critical`. Impact is not severity-of-error; it does not
promote an Incomplete into a Violation.

### Scope

The part of the page a given accessibility check covers: either the whole
**page**, or a single **Element**. Scope decides what is examined, never what
counts as a failure.

### Profile

The set of axe-core rules a check runs, expressed as axe tags. The default
profile is WCAG 2.0 and 2.1, levels A and AA. `best-practice` and
`experimental` rules are outside the default profile and must be opted into.

Profile decides which rules run. Scope decides where they run. Neither decides
what fails.

### Threshold

The maximum number of failing **rules** tolerated before a check fails —
counted as distinct rule ids, never as failing nodes. One rule failing on
eighty-seven nodes is one, so a threshold does not drift when test data grows.
Node counts are reported, never counted.

Violations and Incompletes have separate thresholds; the default tolerates zero
Violations and any number of Incompletes.

### Ignore

A rule id, or a single occurrence of one, deliberately excluded from a check's
counting. Ignored findings are still evaluated by axe-core and still reported —
they are subtracted after the run, never disabled before it. An Ignore
suppresses counting, not detection.

### Stale ignore

An Ignore that no longer matches any finding — the underlying problem was
fixed, and the entry now hides nothing. Detectable only because Ignores filter
rather than disable.
