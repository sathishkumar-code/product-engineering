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
| Apps/surfaces affected | Carried forward from the TD's own field — every app/surface this tech-spec's work touches, not just the repo the reader happens to be in |

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
transitional endpoint a developer needs to know is being phased out. When
this tech-spec covers more than one app/surface, add an **App(s)** column
naming which app(s) call or own each endpoint, so a developer working in a
single repo can filter to their own rows without reading the whole table.

## Core logic
Condensed pseudocode for anything genuinely non-trivial (a precedence rule, a
caching strategy, a state resolution function) — trimmed of alternatives
discussion, accurate to what the TD specifies. Prefix each block with which
app/surface it lives in (e.g. `[Staff app]`) whenever this tech-spec spans
more than one; omit the tag only for logic that's genuinely shared/backend
and not owned by one specific app.

## Notifications / integration behavior
Condensed from the TD — delivery mechanism, triggers, and explicit non-triggers
carried forward as clearly as what does fire. Name which app/surface triggers
each notification and which app/surface(s) receive it — the two are often
different apps (e.g. staff app action notifying the resident/family app).

## UI components
Condensed list of components/hooks named in the TD, enough to know what to
build vs. reuse. Prefix each with its owning app/surface (e.g. `[Resident
app]`) whenever this tech-spec spans more than one — this is usually the
clearest signal of which repo a given component actually belongs in.

## Business rules
Not reproduced here — reference by ID (e.g. "See `spec.md` BR1–BR6"). Add only
rules that are genuinely technical-only and don't appear in `spec.md`.

## Open questions
Mandatory, not optional. Carry forward every unresolved TD open question in
full, especially anything marked High priority or tied to a high-impact risk —
those are exactly what a developer must not discover by building the wrong
thing first. If every question is resolved, write "None — see TD" explicitly.
Carries the TD's own table-cell formatting forward too — a "Current position"
covering more than one fact stays a bullet list here, not a paragraph (see
`PROCESS-WALKTHROUGH.md`'s Key conventions cheat-sheet → Table-cell formatting).

## Testing checklist
Condensed by category (Unit/Integration/Performance/E2E) — but preserve any
test the TD explicitly ties to a named risk rather than generalizing it away.

## Rollout summary
Condensed from the TD's rollout/migration plan — phases and what gates moving
between them, enough that "what does done look like for phase 1" is answerable
from this document alone.
```
