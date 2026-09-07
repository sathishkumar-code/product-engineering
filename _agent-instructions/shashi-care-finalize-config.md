# Config: Finalize Document — Shashi Care

Pairs with `skill-finalize-document-discipline.md`. Project-specific paths and
companion-file naming only — the finalize logic itself lives in the paired skill file
and shouldn't need to change here.

## Where this runs
Any `prd-<slug>.md`/`enhancement-request-<slug>.md`/`bug-report-<slug>.md`, or its
sibling `spec.md`, under
`prd/{features,enhancements,bugs}/<slug>/`, or any
`TD-<slug>.md` under
`architecture/{features,enhancements,bugs}/<slug>/` — in each product's GitLab
`-docs` repo, see `shashi-care-doc-tree.md` for the full shape. `tech-spec-<slug>.md`,
alongside the TD in that same architecture folder, is deliberately NOT yet in
scope — see the skill file's scope note. Finalize is on-demand: Product
Manager runs it against a PRD/ER/`spec.md` it owns, System Architect runs it
against a TD it owns, whenever asked.

## Companion files this pass reads (never writes to)
- `SA-comments-<slug>.md`, at
  `architecture/{features,enhancements,bugs}/<slug>/SA-comments-<slug>.md`
  (`templates/sa-review-comments-template.md`) — the
  running review file, now filed alongside the TD rather than the PRD/ER; note
  this sits in a different top-level folder than the PRD/ER being finalized when
  Product Manager is the one running this pass. Read it to tell
  settled-but-narrated content apart from still-open findings; never edit or
  archive it as part of a finalize pass.
- Any ad hoc changeset file a slug has accumulated (this project's convention so far:
  `<category>-<slug>_PRD-changes-for-SA.md`-style files, e.g.
  `feature-director-operations-dashboard_PRD-changes-for-SA.md`) — same rule,
  read-only input, never edited or archived by this pass.
- `spec.md` has no companion review file of its own. When a `spec.md` revision
  traces back to an SA finding rather than a source-document change or direct
  Sathish input, `SA-comments-<slug>.md` above is the relevant read-only record —
  there's no second, spec-specific file to check.

## Spinning out an Enhancement Request
When Sathish confirms a candidate should become its own Enhancement Request (see the
skill file's escalation rule), file it exactly like any other direct-intake
enhancement — per `shashi-care-doc-tree.md`'s per-slug shape and
`skill-pm-discipline.md`'s intake pathway B, in the relevant product's GitLab
`-docs` repo:
- `prd/enhancements/<new-slug>/intent.md`
  (`templates/intent-template.md`)
- `prd/enhancements/<new-slug>/enhancement-request-<new-slug>.md`
  (`templates/enhancement-request-template.md`), naming it to match the existing
  convention already in use (e.g.
  `enhancement-request-care-conference-calendar-click-to-create.md`).

Base feature field points back at the document being finalized when it's the same
underlying feature, per the template's own `Base feature` row.

## Rebuild note
Product Manager and System Architect are Hermes-hosted and read this file (and
`skill-finalize-document-discipline.md`) via the manually-synced
`product-engineering/` mirror, not through a paste-ready file — any edit here
needs both changed paths called out and copied into that mirror per the "Hermes
copy sync convention" in `shashi-care-process-architect-config.md`. This file is
also still one of the two finalize source files concatenated into the
`cowork-instructions-PM.md` / `cowork-instructions-SA.md` dormant-fallback
artifacts — those are not rebuilt on routine edits; see that same config's
"Rebuild convention".
