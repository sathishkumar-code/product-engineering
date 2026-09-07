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
| senior_living_staffapp | SNF | |
| senior_living_tvapp | SAL | |

Each repo's own CLAUDE.md/AGENTS.md is the actual source of truth for that
repo's branching, write access, and merge-gate conventions — this config
doesn't duplicate or override any of it.

## Where the build brief comes from
**GitLab-direct** — see `_reference/shashi-care-doc-tree.md` and
`shashi-care-gitlab-binding.md`:
- `architecture/{features,enhancements,bugs}/<slug>/tech-spec-<slug>.md`
- `prd/{features,enhancements,bugs}/<slug>/spec.md`
- The assigned Story's acceptance criteria:
  `readiness/{features,enhancements,bugs}/<slug>/epics-stories.md` in the
  product's GitLab `-docs` repo.

`<repo>` is whichever of Shashi-Care-Core-docs, SAL-docs, SNF-docs the
slug's product front-matter says — construct the path directly per
`shashi-care-doc-tree.md`, don't search for it. GitLab checkout access is
not yet confirmed specifically for this persona — same open-item status as
PM's (see `shashi-care-pm-config.md`'s "Access (Hermes)"); escalate to
Sathish rather than assuming it exists.

## Implementation note location
`build/{features,enhancements,bugs}/<slug>/implementation-note-<slug>.md` in
the product's GitLab `-docs` repo — see `_reference/shashi-care-doc-tree.md`.
Draft in the checkout's working tree; `product-team` is the sole actor that
commits it, on this persona reporting the note complete (no approval-gate
field) — see `shashi-care-gitlab-binding.md`'s "Commit mechanics." No
feature branch for this persona's own authoring. `product-engineering`
holds none of this — see `shashi-care-process-architect-config.md`'s
"Hermes as primary host."

## Known Critical/compliance gaps to check before touching related code
Check `_as-built/architecture/technical-debt.md` and
`compliance/<product>-compliance-register.md` (both in the product's GitLab
`-docs` repo) before implementing
anything that touches an area with a logged Blocker-priority or open
compliance gap — two are already named explicitly: the pcc-sync hardcoded
shared-secret issue (no facility scoping) and the unauthenticated WestFax
delivery webhook. Implementing around a known gap without referencing it in
the implementation note is exactly the "design around debt without logging
it" failure System Architect's own discipline already warns against — the
same rule applies here.

## Handover destination
Implementation note (above) plus the MR itself in the code repo — no separate
handover file, same principle PM/SA/PjM now use for their own committed
documents.
