# Tech-Spec Template

Developer-facing, condensed from an approved Technical Design — the full TD
stays authoritative for System Architect and QA. **System Architect writes this
from its own approved TD** — same authorship boundary as everything else.

**Save as**: `tech-spec-<slug>.md`, alongside the TD in that same per-slug
`03_architecture/{features,enhancements,bugs}/<slug>/` folder. Informally called
the Impl Spec by the team — the document's own heading can say so, but this stays
the canonical filename.

```markdown
# <Feature Name> — Tech-Spec

**<Product> | Developer Tech-Spec | derived from Technical Design v<version>**

| Field | Value |
|---|---|
| Source TD | `<path>`, v<version> as of <date> — **re-derive this tech-spec if
  the TD is revised after this date** |
| Status | Draft / Ready for review / Approved |
| repo_status | not-promoted / promoted |
| last_promoted_revision | Timestamp/version last pushed to the code repo, if promoted |

## Overview
One or two sentences — what this builds, condensed from the TD's context/problem
statement.

## Non-goals
Carried forward from the TD's own non-goals, explicit not paraphrased — what
stops a developer from over-building into territory the TD deliberately scoped
out.

## Data model
Condensed from the TD's proposed design — trimmed of rationale, not of fields.
If the TD's schema has a field, it belongs here too, even if the explanation for
*why* stays in the TD only.

## Endpoints
Condensed API table — purpose per endpoint, including any deprecated/
transitional endpoint a developer needs to know is being phased out.

## Core logic
Condensed pseudocode for anything genuinely non-trivial (a precedence rule, a
caching strategy, a state resolution function) — trimmed of alternatives
discussion, accurate to what the TD specifies.

## Notifications / integration behavior
Condensed from the TD — delivery mechanism, triggers, and explicit non-triggers
carried forward as clearly as what does fire.

## UI components
Condensed list of components/hooks named in the TD, enough to know what to
build vs. reuse.

## Business rules
Not reproduced here — reference by ID (e.g. "See `spec.md` BR1–BR6"). Add only
rules that are genuinely technical-only and don't appear in `spec.md`.

## Open questions
Mandatory, not optional. Carry forward every unresolved TD open question in
full, especially anything marked High priority or tied to a high-impact risk —
those are exactly what a developer must not discover by building the wrong
thing first. If every question is resolved, write "None — see TD" explicitly.

## Testing checklist
Condensed by category (Unit/Integration/Performance/E2E) — but preserve any
test the TD explicitly ties to a named risk rather than generalizing it away.

## Rollout summary
Condensed from the TD's rollout/migration plan — phases and what gates moving
between them, enough that "what does done look like for phase 1" is answerable
from this document alone.
```
