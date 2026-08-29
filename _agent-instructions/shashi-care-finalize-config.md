# Config: Finalize Document — Shashi Care

Pairs with `skill-finalize-document-discipline.md`. Project-specific paths and
companion-file naming only — the finalize logic itself lives in the paired skill file
and shouldn't need to change here.

## Where this runs
Any `prd-<slug>.md`/`enhancement-request-<slug>.md`/`bug-report-<slug>.md` under
`{Core|SAL|SNF}/02_prd/{features,enhancements,bugs}/<slug>/`, or any
`TD-<slug>.md` under
`{Core|SAL|SNF}/03_architecture/{features,enhancements,bugs}/<slug>/` — see
`shashi-care-doc-tree.md` for the full shape. Finalize is on-demand: Product
Manager runs it against a PRD/ER it owns, System Architect runs it against a TD
it owns, whenever asked.

## Companion files this pass reads (never writes to)
- `SA-comments-<slug>.md`, at `{Core|SAL|SNF}/03_architecture/
  {features,enhancements,bugs}/<slug>/SA-comments-<slug>.md`
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

## Spinning out an Enhancement Request
When Sathish confirms a candidate should become its own Enhancement Request (see the
skill file's escalation rule), file it exactly like any other direct-intake
enhancement — per `shashi-care-doc-tree.md`'s per-slug shape and
`skill-pm-discipline.md`'s intake pathway B:
- `<Product>/02_prd/enhancements/<new-slug>/intent.md`
  (`templates/intent-template.md`)
- `<Product>/02_prd/enhancements/<new-slug>/enhancement-request-<new-slug>.md`
  (`templates/enhancement-request-template.md`), naming it to match the existing
  convention already in use (e.g.
  `enhancement-request-care-conference-calendar-click-to-create.md`).

Base feature field points back at the document being finalized when it's the same
underlying feature, per the template's own `Base feature` row.

## Rebuild note
This file is one of the two finalize source files concatenated into both
`cowork-instructions-PM.md` and `cowork-instructions-SA.md` — see the rebuild
convention in `shashi-care-process-architect-config.md`. Any edit here needs both
paste-ready files rebuilt and both re-pasted into their live Cowork projects.
