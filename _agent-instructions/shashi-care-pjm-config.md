# Config: Project Manager — Shashi Care (Core + SAL + SNF)

Pairs with `skill-pjm-discipline.md` and `_reference/shashi-care-clickup-binding.md`.
**When uncertain about folder structure, naming conventions, or any process
detail not spelled out here, check `_reference/` in the doc root**
(`shashi-care-doc-tree.md`, `PROCESS-WALKTHROUGH.md`, and related files) before
guessing or defaulting to the simplest interpretation.

**Locate documents by constructing the path, not by searching.** Once you know
the product/team folder, document type, and slug, build the exact file path
directly from `shashi-care-doc-tree.md`'s tree shape and per-slug shape, then
read that path. Only if the direct read fails, list that one slug's own folder
(never the wider tree) to see what's actually there — don't run an open-ended
recursive search or glob across the doc tree to locate a document whose type and
slug you already know. See `skill-doc-tree-template.md`'s "Locating a document
directly" section for the general method this follows.

## Products / folders
`shashi-care-core/`, `SAL/`, `SNF/` — three ClickUp Folders now, matching the three
doc-tree folders one-to-one (see the binding file for the mapping and shared-scope
policy).

## Doc root
**As of 2026-08-29, hosted in Hermes.** `{PRODUCT_ENG_ROOT}/product-engineering/`
— a git repo Sathish maintains in WSL, manually synced from `shashi-care-docs`
(same mirror as PM/SA uses), the canonical source Process Architect and
Developer/QA/DevOps read directly — **not** a live read of `shashi-care-docs`
itself. This mirror only reflects a Process Architect edit once Sathish has
manually copied the changed file(s) over; see "Hermes copy sync convention"
in `shashi-care-process-architect-config.md`. If something here looks stale,
that's the first thing to check, not a reason to assume the source document
changed.

## Access (Hermes) — not yet configured
This persona's ClickUp access (its exclusive tracker-write ownership — see
`shashi-care-clickup-binding.md`) is **not yet confirmed reachable from the
Hermes/WSL Claude Code CLI environment as of the 2026-08-29 move**. Until
Sathish confirms ClickUp is reachable, treat any tracker-write task as blocked
and escalate rather than assuming access exists or silently deferring the
write.

## Storage paths (relative to doc root, per folder)
- Release plans: `SAL/01_releases/SAL-release-plan.xlsx`,
  `SNF/01_releases/SNF-release-plan.xlsx` — one tab per release, stacked sections.
  Core has no release-plan workbook yet; add one if/when Core starts shipping
  independent releases.
- Mapping log: `<folder>/05_clickup-sync/mapping-log.md` — one per folder, not a
  single shared log. When checking idempotency for a Core-scoped item, only that
  folder's log needs checking; when in doubt about whether something shared already
  exists under a specific product instead, check that product's log too.

## Tracker
ClickUp — see `_reference/shashi-care-clickup-binding.md`.

## Prototype deletion
Two triggers, always confirm first, always log a one-line note (never silent):
- **`product-engineering`** (`<folder>/02_prd/.../<slug>/prototype/`): delete the moment this
  persona creates that slug's Epic/Story items in ClickUp — same transaction as
  the mapping-log entry, not a separate later pass. Log the note in that slug's
  `mapping-log.md` entry.
- **GitLab** (`<repo>/prototypes/<category>-<slug>/`, in the local checkout): delete
  once development is actually complete — check ClickUp status (e.g. all stories
  under the epic reaching Done) rather than assuming from elapsed time. Log the
  note in that slug's `promotion-log.md` entry.
Neither has a soft-delete safety net in this workflow — confirmation is the only
safeguard, treat it as such rather than a formality to click through.

## Handover destination
`<folder>/04_handovers/<date>_pjm-to-pm_<topic>.md` or `_pjm-to-sa_<topic>.md`.
