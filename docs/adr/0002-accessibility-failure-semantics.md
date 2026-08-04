# What makes an accessibility check fail

A check fails on **violations** only, counted as **distinct rule ids**, against
the **WCAG 2.0/2.1 A+AA** rule set. Incompletes are reported but never fatal.
Ignored rules are still evaluated and still reported — they are filtered out of
the count after the run, never disabled before it.

Each of those four choices departs from the obvious reading, so each is recorded.

## Violations fail; incompletes do not

axe-core reports `violations` (it is certain) and `incomplete` (it cannot decide,
a human must look). Vaadin themes generate incompletes constantly — colour
contrast over gradients and overlays cannot be computed — so failing on them
would make the zero-config method unusable on an untouched view. They are
surfaced in the failure message instead, where a human can act on them.

An incomplete is not a lesser violation, and impact does not promote one: a
`critical` incomplete is still an open question, not a failure.

## Thresholds count rules, not nodes

axe returns one entry per failing rule, each carrying every failing node. One
contrast bug in a Grid is 1 rule and 87 nodes. Counting nodes would make
thresholds drift with test data — adding a row silently breaks the build — so
the unit is the rule. Node counts appear in the message for blast radius but are
never counted.

This is the hardest of these to reverse: changing the unit silently changes the
meaning of every threshold in every consumer's test suite.

## The default profile excludes best-practice

Out of the box axe runs every rule it has, including `best-practice` and
`experimental`, which fire on stock Vaadin components. A default that fails on
code the consumer cannot fix would force everyone to build an ignore list before
the feature is usable, so the default is the conformance baseline teams are
actually held to. `best-practice` remains one call away.

## Ignores filter rather than disable

The obvious implementation is to pass ignored rule ids to axe's `disableRules()`,
which is cheaper — axe skips the work. We run them anyway and subtract
afterwards, buying three things that route cannot give:

- suppressions stay visible in the report, so an ignore list cannot quietly hide
  a growing problem;
- a **stale ignore** — an entry matching nothing, because someone fixed the
  underlying bug — becomes detectable and reportable;
- node-level ignores remain possible later without changing the mechanism.

The cost is that axe still evaluates rules whose results are discarded. Accepted:
observability of suppressed debt matters more than scan time.

## Consequences

- Ignores are rule ids only, and declared per call. A project-wide list is
  planned; per-call ignores are therefore documented as **additive** from the
  start so adding a global layer is a pure addition, and any override verb is
  reserved for then.
- A check waits once for Vaadin to go idle and then scans once. It does not poll
  like a Playwright assertion: an a11y defect is a property of settled markup and
  will not resolve on a second look, so retrying would only multiply a 1–3s scan.
