# Config: Developer — Shashi Care (per code repo)

Pairs with `skill-developer-discipline.md`. One Developer instance per code
repo listed below — not a single instance spanning all repos (see
`_reference/team-structure.md` for the reasoning: the backend/admin are shared
across SAL/SNF while the mobile/TV/staff clients are separate binaries with
separate release cadences).

## Repos and product mapping
| Repo | Product | Notes |
|---|---|---|
| senior_living_backend | SAL + SNF (combined) | single codebase, no product flag in code — split is emergent from facility careType + config |
| senior_living_admin | SAL + SNF (combined) | shared admin dashboard |
| senior_living_reactnative | SAL | assisted-living resident app |
| senior_living_skillednursing_resident | SNF | SN resident/family app |
| senior_living_staffapp | SNF | resolved 2026-08-29 — previously ambiguous |
| senior_living_tvapp | SAL | resolved 2026-08-29 — previously open |

Each repo's own CLAUDE.md/AGENTS.md is the actual source of truth for that
repo's branching, write access, and merge-gate conventions — this config
doesn't duplicate or override any of it.

## Where the build brief comes from
- `<folder>/03_architecture/{features,enhancements,bugs}/<slug>/tech-spec-<slug>.md`
- `<folder>/02_prd/{features,enhancements,bugs}/<slug>/spec.md`
- `<folder>/02_prd/{features,enhancements,bugs}/<slug>/epics-stories.md` — the
  assigned Story's acceptance criteria specifically, not the whole document

`<folder>` is whichever of `shashi-care-core/`, `SAL/`, `SNF/` the slug's
product front-matter says — construct the path directly per
`shashi-care-doc-tree.md`, don't search for it.

## Implementation note location
`<folder>/07_build/{features,enhancements,bugs}/<slug>/implementation-note-<slug>.md`
— see `shashi-care-doc-tree.md`'s Build-stage addition.

## Known Critical/compliance gaps to check before touching related code
Check `<folder>/03_architecture/technical-debt-register.md` and
`03_architecture/compliance/hipaa-compliance-register.md` before implementing
anything that touches an area with a logged Blocker-priority or open
compliance gap — two are already named explicitly: the pcc-sync hardcoded
shared-secret issue (no facility scoping) and the unauthenticated WestFax
delivery webhook. Implementing around a known gap without referencing it in
the implementation note is exactly the "design around debt without logging
it" failure System Architect's own discipline already warns against — the
same rule applies here.

## Handover destination
Implementation note (above) plus the MR itself in the code repo — no separate
Drive-side handover file the way PM↔SA use `04_handovers/`, since the
artifacts (MR + implementation note) already are the handover.
