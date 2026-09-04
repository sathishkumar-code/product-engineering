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
**As of 2026-09-04, GitLab-direct.** `product-engineering/` is frozen
(historical only) — this persona's own artifacts (`mapping-log.md`, the
tracker-sync material) now live directly in each product's GitLab `-docs`
repo (Shashi-Care-Core-docs, SAL-docs, SNF-docs) — see
`_reference/shashi-care-gitlab-binding.md`. Draft content in the checkout's
working tree on `main`; `product-team` commits it once Sathish confirms,
same as every other persona now — see "Commit mechanics" there.

## Access (Hermes) — not yet configured
This persona's ClickUp access (its exclusive tracker-write ownership — see
`shashi-care-clickup-binding.md`) is **not yet confirmed reachable from the
Hermes/WSL Claude Code CLI environment as of the 2026-08-29 move**. Until
Sathish confirms ClickUp is reachable, treat any tracker-write task as blocked
and escalate rather than assuming access exists or silently deferring the
write.

## Storage paths (relative to each product's GitLab repo root — Shashi-Care-Core-docs / SAL-docs / SNF-docs)
- Release plans: `releases/SAL-release-plan.xlsx` in SAL-docs,
  `releases/SNF-release-plan.xlsx` in SNF-docs — one tab per release, stacked
  sections. Core has no release-plan workbook yet; add one if/when Core starts
  shipping independent releases.
- Mapping log: `tracker-sync/mapping-log.md` — one per repo, not a single
  shared log. When checking idempotency for a Core-scoped item, only that
  repo's log needs checking; when in doubt about whether something shared
  already exists under a specific product instead, check that product's log
  too.

## Tracker
ClickUp — see `_reference/shashi-care-clickup-binding.md`.

## Prototype deletion
**Removed as a process step, 2026-09-04.** This persona no longer deletes the
GitLab `prototypes/<category>-<slug>/` folder at any point — it's retained
permanently, same as every other committed artifact (Sathish's decision, to
cut cognitive load and process overhead). Don't delete it, don't ask about
deleting it, don't track a deletion trigger. If Sathish wants an old
prototype export cleaned up, that's his own call made outside this process,
not something this persona initiates or confirms.

## Handover destination
`<folder>/04_handovers/<date>_pjm-to-pm_<topic>.md` or `_pjm-to-sa_<topic>.md`.
