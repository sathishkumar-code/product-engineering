# Spec Template

Developer-facing, condensed from an approved source document — a PRD (feature),
Enhancement Request, or Bug Report, whichever produced this slug. Strips
business/market context down to what a developer needs. **Product Manager writes
this from its own approved source document** — same authorship boundary as
everything else in this system.

```markdown
# <Feature/Enhancement/Bug Name> — Spec

**<Product> | Developer Spec | derived from <PRD/ER/BR> v<version>**

| Field | Value |
|---|---|
| Source document | `<path>` (prd-<slug>.md / enhancement-request-<slug>.md / bug-report-<slug>.md), v<version> as of <date> —
  **re-derive this spec if the source document's Revision History gets a new
  entry after this date** |
| Status | Draft / Ready for review / Approved |
| repo_status | not-promoted / promoted |
| last_promoted_revision | Timestamp/version last pushed to the code repo, if promoted |

## Feature overview
One or two sentences — the business goal condensed to *why*, not the source
document's full framing.

## Goals
Bulleted, condensed from the source document's Assumptions/Current-behavior and
Scope sections.

## Success criteria
Optional — only if a developer-visible acceptance signal exists beyond the
functional spec itself (a specific measurable behavior, not a business metric
like adoption rate, which stays in the source document only).

## Scope
### In scope
### Out of scope
Pull boundaries from wherever they live in the source document — not just its
formally-labeled scope section. A developer needs the complete boundary, not
just the part filed under a scope heading.

## Workflow diagram
Mermaid sequence diagram (actor-to-actor message passing) — reads better for a
developer than a PRD's swim-lane convention, which serves a different audience
(Product Manager, QA). Only needed where there's a real flow to show — many
enhancements and bugs won't need one.

## Data model diagram (ERD)
Optional — valuable when the source document's own data section is prose rather
than a formal model. Mermaid `erDiagram`.

## Functional specification
Condensed table, same shape as the source document's own functional
spec/proposed-change section, trimmed of cross-references to other modules a
developer doesn't need to trace.

## Business rules
Canonical location — copied here in full, not summarized. `tech-spec-<slug>.md`
references these by ID and never reproduces them.

## Open questions
Carry forward any open question from the source document still unresolved. If
none, write "None — see <PRD/ER/BR>" explicitly rather than omitting the
section.
```
