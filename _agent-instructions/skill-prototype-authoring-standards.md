# Skill: Prototype Authoring Standards for Downstream Agent Consumption

For any design tool used to build interactive prototypes that a downstream agent
(e.g. a Product Manager persona) will later read to draft requirements. Two goals:
test data that actually demonstrates every business rule rather than just the happy
path, and inline comments that state business logic and prototype-only shortcuts
explicitly rather than requiring the downstream agent to infer them.

## Test data coverage
When seeding data for a prototype, cover:
- **Happy path** — at least one straightforward, unremarkable example per flow.
- **Boundary values** — the edges of any range or threshold (minimum, maximum, the
  value immediately on either side of a cutoff).
- **Every distinct state/status value** — if there's a state model, seed at least
  one record in each state, not just the common ones.
- **Empty/null states** — what the page looks like with zero records, a missing
  optional field, an unset value.
- **Error/invalid states** — what an invalid input, a failed validation, or a
  blocked action actually looks like, not just what success looks like.
- **Role/permission variations** — if different roles see or can do different
  things, seed at least one example demonstrating each role's view.
- **Repetition where a rule should hold** — a rule demonstrated once could be
  coincidence; the same rule holding consistently across several varied examples is
  what makes it recognizable as a real rule rather than an artifact of one example.

## Inline business-logic comments
Tag business logic explicitly, right above the code that implements it, with a
consistent marker:
```
// BUSINESS RULE: <plain-language statement, e.g. "Only a Nurse role can transition
// status from Pending Review to Approved">
```
This lets a downstream agent reliably locate and extract the actual rule rather
than inferring it purely from data patterns — inference is a reasonable fallback,
not a substitute for an explicit statement when the logic is meaningful enough to
warrant a comment in the first place.

Tag prototype-only shortcuts distinctly, so a downstream agent doesn't mistake an
implementation shortcut for a requirement:
```
// PROTOTYPE ONLY: <what this is and why it's not production-representative, e.g.
// "using localStorage instead of a real backend for demo purposes">
```

## Data locality
Keep all data synthetic and self-contained within the prototype — no live external
calls, no dependency on a real backend. Beyond the obvious demo convenience, this
also means a downstream agent reading the exported source has no invisible
external dependency to miss; everything the logic needs is present in the file
itself.

## Page/flow structure
One page = one coherent flow, with a stable, descriptive page name. This is what
lets a downstream document reference "which page" a requirement came from without
ambiguity.

## What this doesn't replace
None of the above removes a downstream agent's own cross-check against the actual
prototype before finalizing requirements — good authoring reduces how much there is
to catch, it doesn't guarantee nothing needs catching.
