# SA Review: feature-idt-reports-v2

| Field | Value |
|---|---|
| Source PRD | SNF/02_prd/features/prd-idt-reports-v2.md (v1.0, 2026-08-12; updated through the round-3 handoff, `idt-reports-handoff-round3.md`) |
| Product | SNF |
| review_round | 4 (Team review of the PRD and TD, `idt-reports-handoff-round3.md` — 9 PRD changes reviewed against the TD, plus one inconsistency discovered independently) |
| Verdict | Approved-with-changes — eight of nine round-3 items are fully resolved (Refresh already matched; Diagnosis rename; signer-designation simplified to `staff.mobile_access` with a facility setting/rollout step/risk removed; Rehab clear-on-deactivate fully specified — clears all fields including Goal, and re-enabling starts fresh rather than restoring cleared data; Rehab mark-complete-disabled fully specified). Field-level encryption (TD §3.4a) is **[Confirmed by Sathish]**, including the carry-forward decrypt/re-encrypt mechanism — no better alternative exists without new cryptographic risk or breaking an existing design invariant (§11 item 24 closed). The independently-discovered weight-trend calendar-week inconsistency is also resolved: `weightTrendSnapshot` is simplified to a plain mirror of `residentObservationTrends.readings` with no week logic at all (§11 item 25 closed). Only two items remain genuinely open — both correctly left that way rather than resolved: the Rehab Team authorization mechanism (Engineering-only, TD §11 item 21) and the Director bypass question (TD §11 item 20) — plus one non-blocking, low-priority item (date format, TD §11 item 22). Ready for Epics/Stories on everything except the Rehab Team mechanism, which the Permissions epic depends on. |

## PRD Review (Round 0 → closed out via this round's consistency pass)
A dedicated PRD-only review pass was never run separately — instead, every PRD requirement has been
checked against `feature-idt-reports-v2_TD.md` cumulatively across each round of Technical Design
Review below, culminating in a full section-by-section final pass (this entry). That final pass reads
every section of the current PRD (Sections 1–13) against the current TD end to end. Result: **every
in-scope PRD requirement has a corresponding, consistent TD design element** — status model, all writes
in Section 7, the permission matrix, all screen-spec behavior in Section 5 (list, form, discipline
completion, review/signature, PDF/fax), the two facility-configurable Non-functional settings, and the
newly-added Section 4.1 Refresh control, Section 4.2 rehab-checkbox-reset bullet, Section 5.1
bulk-action partial-failure copy, and Section 12 signer-designation-taxonomy bullet all match what
`feature-idt-reports-v2_TD.md` specifies, word-for-word where the PRD text was drawn directly from the
TD/brief. No unresolved contradiction between the two documents was found.

Two non-blocking documentation notes surfaced during this pass, neither of which blocks Epics/Stories:

1. **Open-question numbering collision.** PRD §11 numbers its one open item as "1" (archival
   semantics). TD §11 independently numbers its own open items starting at "1" too (Chief Complaint PCC
   provenance) — a different topic entirely. Anyone cross-referencing "item 1" between the two documents
   without also naming the document could point at the wrong question. Recommend disambiguating before
   Epics/Stories are drafted — e.g. prefixing references as "PRD §11 item 1" / "TD §11 item 1"
   consistently (the TD already does this in a few places, PRD does not need to), or renumbering one of
   the two lists.
2. **Two TD-only open items aren't mirrored in PRD §12 (Engineering follow-up)**, unlike the
   signer-designation-taxonomy item that now is. Rather than add them to the PRD piecemeal, Sathish
   directed folding them — plus the other small external-confirmation items already noted below — into
   one consolidated **Spike** task in the epics/stories list, so they're not lost heading into
   Epics/Stories even though none of them blocks story creation for the report itself. See that Spike
   task in the list below.

### PRD Review — Round 3 (`idt-reports-handoff-round3.md`)

The team's own review of the PRD and TD surfaced nine further PRD changes since round 2, all already
applied directly to `prd-idt-reports-v2.md` by the PM/team (not by this review — the PRD is edited only
by the PM/team, per standing practice). This pass checks each of the nine against the TD and updates it
accordingly. Outcome, grouped by how each landed:

- **Already matched, no TD change needed (1 item).** Item 5 (Refresh consolidated to one control) —
  the TD's Refresh design (§3.3) was already built as a single action across all three auto-populated
  sections; the PRD change confirms this, doesn't require it.
- **Resolved cleanly, TD updated (4 items).** Item 6 (Chief Complaint → Diagnosis, confirmed
  PCC-sourced) — field renamed in TD §3.4, closes the Diagnosis half of former TD §11 item 1. Item 7
  (signer designation taxonomy) — resolved *smaller* than previously designed: no facility setting
  needed at all, `staff.mobile_access == 'Doctor'` replaces the proposed `eligibleSignerDesignations`
  list, removing a facility setting, a rollout step, and a risk row (TD §3.3, §3.7, §9, §10). Items 2–3
  (Rehab discipline clear-on-deactivate; Rehab mark-complete checkbox disabled once Complete) — two new,
  well-scoped frontend features, fully specified in TD §3.4 and §6; **Sathish subsequently confirmed**
  both of item 2's open sub-questions: it clears all of the discipline's fields on deactivation,
  including the free-text Goal field, not only the chip ratings, and re-enabling the toggle afterward
  starts fresh rather than restoring the cleared data — fully resolved, closing TD §11 item 23.
- **Correctly left open, not resolved (2 items).** Item 8 (Rehab Team authorization mechanism) — the PM
  was explicit this is Engineering's call alone, not the PM's, the AI's, or this review's, even though a
  similar approach to item 7 *may* end up applying; the TD proposes no mechanism (TD §11 item 21). Item
  1 (Rehab Team no longer a single "Director of Rehab" designation) is confirmed as a persona change,
  and the resident-assignment requirement is confirmed to extend to the Rehab Team the same way it
  applies to Case Manager/Social Worker — but *how* to identify Rehab Team membership is exactly item
  8's open question, so this isn't fully actionable yet either. The pre-existing Director-bypass
  question (previously carried in the TD as an unstated assumption) is corrected to match its real,
  still-open status (PRD §11 item 2, TD §11 item 20) — this review does not silently confirm it either
  way. Item 9 (date format, fixed vs. facility-configurable) is low-priority and explicitly
  not-yet-decided (TD §11 item 22) — flagged, not resolved.
- **Genuinely new requirement, now confirmed by Sathish as Architect (1 item).** Item 4 (field-level
  encryption of all `idtreports`/`idtReportVersions` data) — this design had not previously specified
  encryption for these two collections at all (only for the unrelated `observations`/
  `residentObservationTrends` weight-trend collections). PRD §12 explicitly delegated this decision to
  the Architect rather than this review resolving it unilaterally. The recommended scope and pattern —
  reusing the platform's existing per-field AES-256-GCM approach — proposed in TD §3.4a is now
  **confirmed**, along with its most significant implementation wrinkle: carry-forward must decrypt the
  prior report's content and re-encrypt it fresh (new IV) onto the new report, rather than copying
  ciphertext directly. Sathish asked whether a better alternative to that cycle exists; it doesn't —
  copying ciphertext as-is only avoids the cost until the content's first edit and introduces a fragile
  special-cased code path; deterministic encryption trades away semantic security for one collection;
  per-document data keys don't avoid the decrypt/re-encrypt step at all; and referencing the source
  document instead of copying would reopen the `idtReportVersions` gap already fixed elsewhere in this
  design. Full alternatives analysis in TD §3.4a. This closes TD §11 item 24.
- **One additional inconsistency discovered independently during this pass, not itself a round 3
  handoff item — now resolved.** PRD §4.1 describes the report-level weight trend as "a rolling set of
  3 [readings] ... with no calendar-week grouping or bucketing" — this didn't match
  `weightTrendSnapshot`'s calendar-week-slot design (TD §3.4), which predated this PRD wording.
  **Confirmed by Sathish:** the PRD wording is correct — `weightTrendSnapshot` is simplified to
  directly mirror `residentObservationTrends`'s 3-reading ring buffer, with no calendar-week logic
  anywhere. This closes TD §11 item 25.

### Recommended technical epics/stories/spikes
Not yet raised as a formal `epics-stories.md` — see the Epics/Stories Review section below. Candidate
groupings, drawn from the TD's own structure, for whoever drafts that document next:
- **Report data model & lifecycle** — `idtreports`/`idtReportVersions`/`idtReportDeletionLogs` schema,
  Draft→Complete status, independent signature-state cycling (TD §3.3, §3.4). **[Design-review update]**
  `createdByType`/`updatedByType` are dropped from `idtreports` — confirmed remnants of the previous
  report structure, not needed here; no companion field replaces them. **[Round 3, PRD item 4 —
  RESOLVED, confirmed by Sathish]** All IDT report data now needs field-level encryption at rest
  (`idtreports` and `idtReportVersions` alike), a genuinely new requirement this design hadn't
  previously addressed. The scope and pattern in TD §3.4a (reusing the existing per-field AES-256-GCM
  approach already used for `observations`) are now confirmed, including the carry-forward
  decrypt/re-encrypt mechanism it requires — Sathish confirmed no better alternative exists (TD §3.4a
  has the full analysis). This epic is now ready to scope on the encryption dimension; build the three
  flagged implementation wrinkles correctly (carry-forward's decrypt/re-encrypt, the rehab-trend
  read-path decrypt cost, the Refresh-diff comparing decrypted values) rather than treating them as
  afterthoughts.
- **Auto-populated data & Refresh** — Create-time snapshotting plus the new Refresh endpoint/control
  (TD §3.3, §6).
- **Create & carry-forward, including bulk-create** — single/multi-resident Create, Start blank, the
  Draft-only partial-unique index, partial-failure toast (TD §3.3, §3.6, §3.9). **[Design-review update,
  item 13]** add a "view details" affordance for the full skipped-resident list on bulk-create failures,
  beyond the toast's "+N more" truncation — frontend-only, the bulk-create endpoint already returns the
  full per-item data.
- **Bulk delete** — multi-select delete, per-report re-verification, partial-failure toast (TD §3.10).
  **[Design-review update, item 13]** same "view details" affordance recommended for bulk-delete's
  partial-failure toast — frontend-only, no new API needed.
- **Discipline completion & rehab progression trend** — derived completion states, report-over-report
  table (TD §3.3, §3.4). **[Round 3 item 3]** Once a report is Complete, the Rehab mark-complete
  checkbox becomes disabled/read-only (still shown ticked) — frontend-only, no schema impact.
- **Rehab discipline clear-on-deactivate (new, round 3 item 2) — fully resolved.** Turning off a PT/OT
  toggle after ratings have been entered prompts a confirmation and clears that discipline's fields on
  confirm (frontend logic + the existing Save flow, no new endpoint, TD §3.4). **[Confirmed by Sathish]**
  clearing extends to **all** of the discipline's fields, including the free-text Goal field, not just
  the chip ratings; and re-enabling the toggle afterward **starts fresh**, not restoring the cleared
  data. Ready for story-writing as-is (TD §11 item 23 closed).
- **Send-for-signature & Sign** — signer-resolution query, single-select picker (TD §3.3). **[Round 3
  item 7 — RESOLVED, now smaller than previously designed]** The facility-configurable
  `eligibleSignerDesignations` setting is no longer needed. Engineering confirmed the `staff` object
  already carries `mobile_access`, valued `'Doctor'` for every signer-eligible user (attending MD and
  PA/NP alike); the query reads that field directly. This removes a facility setting, a rollout step,
  and a risk row this design previously carried — smaller scope for this epic, not larger.
- **Staff mobile app notifications** — join the existing shared workflow as a third document type (TD
  §3.11) — needs the registration-mechanism spike noted above (TD item 13).
- **PDF, print, and fax** — re-template the existing pipeline to the new schema (TD §6). **[Round 2
  addition]** now also covers: dropping the fax popover's saved-contacts shortlist for a single
  free-text number with US phone format validation, and adding a `status` field to `faxLog` (TD §3.4,
  §6).
- **Fax History section (new, round 2)** — collapsed-by-default UI on the report summary, rendered
  directly from the existing `faxLog` array (number/sent date-time/actor/status) — **no new endpoint
  needed**, frontend-only work once `faxLog.status` exists (TD §3.4, §6). The exact `status` enum
  values depend on the fax-vendor discussion already tracked in the Spike below (item 2).
- **List status display rework (new, round 2)** — remove the per-row Status chip from both tabs; add a
  Signature Status column (Not sent / Awaiting signature / Signed) to the Completed tab only, backed by
  the already-existing `signature.state` field and index — no new index needed (TD §6).
- **Edited-after-signature indicator (round 2, updated in Sathish's design-review pass)** — surface an
  indicator as the last item in the report form's sub-title, backed by the already-existing
  `signature.editedAfterSignAt` field — no new field needed, just exposing it in the report-get response
  (TD §3.3, §3.4, §6). **[Design-review update]** the indicator now shows the current signature status
  alongside the date, not the date alone — `signature.state` must also be included in the report-get
  response. Suggested PRD wording for the PM: replace "Edited after signature [date]" with something
  like **"Signed — edited after signature on [date]"** (PRD §5.2/§6 rule 4/§7 rule 4).
- **Grid sort/display fix (new, Sathish's design-review pass, items 1, 2, 4)** — three small,
  independent changes: (1) switch the list's default sort back to `reportDate` (from `createdAt`) to
  match the "Report Date" column the PRD actually displays (TD §3.6) — a distinction that only started
  to matter once `reportDate` became mutable; (2) display Report Date on the Edit and Summary pages too,
  not just the list — PRD content addition only, no schema/API change; (3) add the unsaved-changes
  confirmation dialog on navigating away from an unsaved report — already committed in PRD §10 but never
  previously captured in the TD, now documented with an explicit test case (TD §6, §8).
- **`idtReportVersions` schema fixes (new, Sathish's design-review pass, items 11–12, a real bug caught)**
  — two independent fixes to the version-history collection: (1) every version row must now capture all
  three auto-populated snapshots (Patient Information, Weight trend, Active medications), not only the
  team-entered content — the prior design incorrectly treated those snapshots as invariant across
  versions, which broke once Refresh + Save could change them; without this fix, a post-signature
  refresh would leave no record of what the auto data looked like at signing time, breaking the PRD's
  "must render years later exactly as signed" requirement; (2) add a `residentId` field to
  `idtReportVersions`, denormalized the same way `facilityId` already is, for direct resident-scoped
  audit queries without a join back through `idtreports` (TD §3.4, §3.10 for related risk detail).
- **`reportDate`/`createdAt` sequencing fix (new, round 2, confirmed and ready)** — switch
  carry-forward, the rehab progression trend, the general lookup index, and the retention-cleanup
  anchor from `reportDate` to `createdAt`, plus the server-side Save-time diff that detects a preceding
  Refresh (TD §3.3's dedicated block, §3.4, §3.6, §3.8). **Confirmed by Sathish** — ready for story
  creation; use the TD as the source of truth for this item, since the PRD text alone only specifies
  `reportDate` moving and doesn't mention `createdAt`.
- **Facility-level settings & scheduled jobs** — `staleDraftCleanupDays` and `versionRetentionYears`
  (down from three settings — `eligibleSignerDesignations` is removed, round 3 item 7), plus the
  stale-draft and retention-cleanup scheduled jobs and their dry-run/safety rollout (TD §3.7, §3.8,
  §9). **[Round 3 item 9, low priority, not decided]** Whether `mm/dd/yy` (TD round 2) should become a
  third facility setting rather than a fixed value is raised for consideration in PRD §11 item 4 — not
  yet decided, non-blocking either way.
- **Data cleanup rollout** — verify-then-clear the pre-production collection before deploy (TD §9).
- **Permissions** — server-side role-matrix enforcement (TD §7). **[Design-review update]** now
  includes the resident-assignment check for Case Manager/Social Worker (TD §6, §7) — a resident must
  be in the caller's `assignedStaff` for Create/Edit/Complete, reversing an earlier confirmed decision.
  **[Round 3, confirmed]** This requirement now explicitly extends to the Rehab Team too — no longer an
  open assumption. **[Round 3 item 1]** Rehab Team itself is no longer scoped to a single "Director of
  Rehab" designation — a facility's rehab team may include several designations. **Not ready for
  story-writing yet, on two distinct points:** (1) **how the system determines who counts as "Rehab
  Team"** is explicitly reserved for Engineering to answer on its own (PRD §8, §11 item 3, TD §11 item
  21) — not a call the PM, the AI, or this SA review gets to make, even by analogy to the now-resolved
  signer-designation question; this design deliberately proposes no field or list for it. (2) **whether
  Directors keep their facility-wide bypass** from the assignment requirement is a separately tracked,
  still-open PRD question (§11 item 2, TD §11 item 20) — this design does not assume they do.
- **Case Manager / Social Worker list rendering** — multi-value column/filter UI update, flagged as a
  risk if missed before Epics/Stories are drafted (TD §10, §6).
- **Resident weight-trend data (new, cross-feature dependency, not part of the IDT Reports codebase)** —
  now fully designed and fully confirmed against the real `observations` collection: a new
  `residentObservationTrends` collection (one document per `residentId`+`type`, a fixed-size 3-reading
  ring buffer — the latest reading replaces the oldest on every update, no calendar-week bucketing),
  upserted by the same PCC webhook that already writes to `observations`, consumed by both the Resident
  Info UI (latest entry) and the IDT report (all 3, as its trend). No scheduled "window advance" job is
  needed — staleness is a read-time comparison, not stored state. **[Confirmed by Sathish]** Retention
  scope: only 3 readings live in this collection; `observations` is untouched and keeps its full
  per-event history as the audit-of-record. **[Confirmed by Sathish]** Encryption: vitals (including
  weight) are PHI, so this collection stays field-level encrypted exactly like `observations` — no
  plaintext relaxation, and `recordedAt` staying plaintext (needed to order/evict readings) is confirmed
  acceptable. The proposed object shape and approach are confirmed as-is, no further design changes
  needed. Likely owned by whoever owns the Resident Info/patient-record feature and the PCC integration
  layer, with IDT Reports as a downstream consumer — **ready to scope once this upstream capability is
  built; no design sign-offs remain outstanding.** **[Resolved, discovered during round 3 review — not
  a round 3 handoff item itself]** A separate inconsistency surfaced comparing the current PRD text
  against the TD: PRD §4.1 describes the report-level weight trend as "a rolling set of 3 [readings]
  ... with no calendar-week grouping or bucketing," which didn't match `weightTrendSnapshot`'s former
  calendar-week-anchored-slot design. **Confirmed by Sathish:** the PRD wording is correct —
  `weightTrendSnapshot` is simplified to directly mirror `residentObservationTrends.readings`, with no
  calendar-week logic anywhere (TD §3.4). This closes TD §11 item 25.
- **Stale resident-data detection and remediation (new, later phase, not part of this release)** — once
  the weight-trend capability above exists, a scheduled check for residents whose data hasn't refreshed
  in over a week, triggering an on-demand PCC API pull first, then a staff notification via the existing
  mobile app workflow (§3.11) if still stale after that. Tracked in TD §12 as a future-phase item, not
  something to scope now.
- **Spike — pre-implementation confirmations and coordination.** **[Confirmed by Sathish]** One
  consolidated research/coordination task, not a build epic, covering five small external-confirmation
  items — none large enough alone to justify its own ticket, but each needs an answer from outside the
  IDT reports codebase before or during the epics/stories it touches:
  1. ~~**Eligible signer designation taxonomy**~~ — **RESOLVED, round 3 item 7.** No facility setting
     needed; the signer query reads `staff.mobile_access == 'Doctor'` directly. Removed from this
     Spike's remaining scope.
  2. **Notification-type registration mechanism** (TD item 13) — how "IDT report" gets registered as a
     new document type in the existing shared staff-mobile-app notification workflow (config, template,
     or code enum); confirm with whoever owns that workflow.
  3. **Code Status PCC-sync confirmation** (TD item 2) — whether this Patient-Information field
     actually syncs from PCC into the backend today; if not, Section 4.1's auto-population for it
     implies unscoped PCC integration work. **[Round 3 item 6]** The Diagnosis half of this item
     (formerly Chief Complaint) is now resolved — renamed, and the PM confirms it's available from PCC.
  4. **Referral workflow's tolerance of an anticipated, non-final discharge date** (TD §3.4 residual
     note) — confirm with whoever owns the Referral workflow that an IDT-set anticipated discharge date
     on `residents.dischargeDate` isn't misread as a confirmed one before an actual discharge is
     processed.
  5. **Archival/PCC-push semantics** (PRD §11 item 1) — what "archived into the patient record" means
     and whether PCC's document APIs support updating an already-pushed document; the one item that
     blocks archival stories specifically, not the rest of the release.
  6. **[Round 2 addition]** **Fax vendor integration and send-status values** (PRD §12, handoff round 2
     item 6, TD §3.4/§6) — what service actually transmits the fax and what send-status values it
     reports back (a simple "Sent" vs. a real delivery/failure status); settles `faxLog.status`'s exact
     enum and gates the new Fax History epic above.
  7. **[Round 2 addition]** **Nurse-persona permission sanity check** (handoff round 2 item 1, TD §11
     item 17) — confirm with whoever owns the permissions model that no code path grants IDT-report
     creation rights to a "Nurse" designation specifically, now that PRD §3 narrows the creating persona
     to Case Manager only. Expected to be a no-op confirmation, not a code change.
  8. ~~**IDT report encryption scope confirmation**~~ — **RESOLVED, confirmed by Sathish.** The
     recommended field-level encryption scope in TD §3.4a, including the carry-forward decrypt/
     re-encrypt mechanism, is confirmed as the design to build (TD §11 item 24 closed). No better
     alternative to the decrypt/re-encrypt cycle exists — see TD §3.4a's alternatives analysis. Removed
     from this Spike's remaining scope.

  **Not part of this Spike, tracked separately as a blocker rather than a quick confirmation:** the
  Rehab Team authorization mechanism (PRD §8, §11 item 3, TD §11 item 21) is explicitly reserved for
  Engineering to answer on its own — not a coordination item this Spike can fold in, since the PM was
  explicit it isn't the PM's, the AI's, or the SA agent's call even by analogy to item 1's resolution.
  The Permissions epic isn't ready for story-writing until that answer comes back.

  Owner: Architect/Development Lead, coordinating with PCC integration, the notification-workflow
  owner, the Referral-workflow owner, and (new this round) the fax-vendor integration owner and the
  permissions-model owner respectively. Nothing in this Spike blocks story creation for the core report
  — items 1 and 8 are both resolved, item 2 gates the notification epic reaching production, item 3 is
  informational for the auto-population epic, item 4 is a quick cross-team check, item 5 gates only
  the archival stories already called out as blocked in PRD §11, item 6 gates only the Fax History
  epic's exact status display (Fax itself and its `faxLog` write are unblocked), and item 7 is a
  no-op-expected sanity check that gates nothing.

## Epics/Stories Review (Round 0)
Not applicable yet — `epics-stories.md` hasn't been generated for this feature. Checked the full SNF
doc tree (`01_releases`, `02_prd`, `03_architecture`, `04_handovers`, `05_tracker-sync`) as part of this
round: no epics/stories artifact exists anywhere yet. With this round's consistency pass now complete,
creating that document is the clear next step — see the candidate list above as a starting point.

## Technical Design Review

| Field | Value |
|---|---|
| Source | Team-submitted |
| Submission path (if team-submitted) | `architecture-submissions/feature-idt-reports-v2/td_idt_reports_v2.pdf` |
| Original format (if team-submitted) | PDF |
| Convert to standard template? | Yes — converted. See `feature-idt-reports-v2_TD.md` (filed alongside this comments file). |
| Verdict | Approved-with-changes |

### Findings

All items that were open after the last pass have now been decided by Sathish, except two that are
being deliberately left open and one that's scheduled as a one-time discussion with engineering.
`feature-idt-reports-v2_TD.md` has been updated to reflect every decision — see that document for
the actual schema/design detail. Summary of what was decided:

- **Case Manager / Social Worker are multi-valued by design** — a resident can have several of each.
  Corrected in this pass: no dedicated fields are added to `idtreports` for this at all — a
  denormalized array field was the first draft, but Sathish flagged it as unnecessary complexity.
  The existing `residents.assignedStaff` is the single source of truth; the list resolves Case
  Manager/Social Worker live via a join (`idtreports.residentId` → `residents.assignedStaff`), the
  same pattern already used for the signer pool. Also confirmed: any Case Manager or Social Worker
  can start or complete any report they can reach — this isn't gated by a specific assignment. The
  one place `assignedStaff` does matter: the resident picker in the Create-report flow defaults to
  residents the requesting staff member is assigned to (not the full facility roster) — that's a UX
  default, explicitly not a backend authorization rule. Note for whoever drafts Epics/Stories: the
  PRD's list UI currently describes Case Manager/Social Worker as single-value columns, which now
  needs to render multiple names.
- **Signature recipient is single-select**, confirmed — no schema change needed.
- **Discharge date accepts today or any later date** — only strictly-past dates are rejected, to
  accommodate late entries.
- **Fax ships in this release** — `faxLog` is now treated as active, not "reserved pending a separate
  module." The exact integration with whatever service actually transmits the fax still needs
  confirming with engineering, but the scope question is settled.
- **`residents.dischargeDate` (the existing field) is reused directly** rather than adding a new
  `anticipatedDischargeDate` field. One residual point worth a quick check: this field is also used by
  the Referral workflow, which wasn't visible in this review — worth confirming it tolerates an
  anticipated (non-final) date being present before an actual discharge is processed.
- **PA/NP designation values vary by facility** — there's no single fixed designation string that
  works everywhere, so the signer-resolution query now reads from a new facility-level setting
  (`facilities.idtReportSettings.eligibleSignerDesignations`) instead of a hard-coded list. This is
  flagged as a one-time discussion item with the engineering team to settle the actual designation
  taxonomy — not fully resolved yet, by design.
- **Chief Complaint / Code Status PCC provenance** — left open intentionally, not blocking.
- **Version/record retention** — the design previously had no lifecycle policy for `idtReportVersions`
  at all (unbounded growth). Now decided: a default 10-year, facility-configurable retention window
  (`facilities.idtReportSettings.versionRetentionYears`), anchored on the report's immutable
  `reportDate`, after which a report and its full version history are purged together by a scheduled
  job (same pattern as stale-draft cleanup). Matches ordinary SNF medical-record retention practice.
  This needs a corresponding PRD update (Non-functional requirements, and an engineering follow-up
  item) — see the summary handed off alongside this file.
- **Bulk report creation** — the Create action's resident picker now supports selecting multiple
  residents and generating a Draft report for each in one action, instead of one resident per Create
  click. Alphabetically sorted, with a live client-side search filter as the user types. The picker's
  eligible-residents query gained one new exclusion as a result — residents who already have a report
  for the current week are no longer offered — to avoid most avoidable collisions with the
  one-report-per-resident-per-week constraint once many residents can be selected at once. See
  `feature-idt-reports-v2_TD.md` §3.9 for the full design, including three items still open: the
  exact post-creation landing screen, the exact partial-failure copy when one or more selected
  residents are skipped, and whether a selection cap is needed.
- **Bulk delete of Draft reports** — now in scope per the latest PRD update (moved from an explicit
  non-goal to Section 2.1). Reviewed and reflected in the design: bulk delete reuses the exact
  single-delete flow per report (no schema change — `idtReportDeletionLogs` already writes one row
  per deleted report regardless of trigger), re-verifies each report is still a Draft at delete time
  rather than trusting the client, and skips (rather than aborting the whole batch) any report that's
  since been Submitted by someone else. Bulk submit and bulk send-for-signature remain explicitly out
  of scope and single-report-only, per the PRD's stated rationale (a bulk resident selection can span
  more than one attending physician) — nothing in the design changes that. One open item carried over
  from bulk-create: the exact partial-failure UX (what the user sees when one or more selected
  reports/residents can't be processed) should be decided once, consistently, for both bulk actions —
  see `feature-idt-reports-v2_TD.md` §3.10 and §11 item 8.
- **PRD-vs-TD consistency review this round.** A full pass comparing the current PRD against
  `feature-idt-reports-v2_TD.md` surfaced five items, now resolved as follows:
  - **Staff mobile app push notifications (PRD §2.1/§2.2/§5.4/§7)** had no technical design at all.
    Now designed: two notifications join the existing shared signature-notification workflow (already
    serving the Health Referral Order Summary and Medication List) as a third document type — one to
    the chosen signer on Send for signature, one to every staff member in the resident's
    `assignedStaff` on Sign. No new schema needed; the underlying facts are already captured by
    existing signature fields. See `feature-idt-reports-v2_TD.md` §3.11 for the full design, including
    one open item: the exact mechanism for registering "IDT report" as a new document type in that
    existing (externally-owned) service.
  - **The discharge-date valid range** — PRD §5.2 says "must be a future date," the technical design
    says "today or any later date" per Sathish's earlier confirmation. Sathish confirmed the technical
    design is correct; the PRD will be updated to match, no technical design change needed.
  - **The single-resident Create/carry-forward flow (PRD §4.2)** was referenced in several places in
    the technical design as if already specified, but had never actually been written up. Now
    documented explicitly, including two points flagged as assumptions worth a quick confirmation:
    whether the Rehab "mark complete" checkbox resets on a new report (assumed yes) and whether "Start
    blank" needs to overwrite an already-persisted draft immediately or a lighter client-side-only
    reset is sufficient (assumed the latter). See §3.3.
  - **The list's resident A–Z sort (PRD §5.1)** had no supporting index in the technical design, even
    though two other places referred to it as already accounted for. Added, mirroring the existing
    Attending MD A–Z index. See §3.6.
  - **The signer picker's "attending MD and their PA/NP" wording (PRD §5.4)** reads as a narrower,
    one-to-one relationship than the technical design's broader `assignedStaff`-plus-designation
    query. Sathish confirmed the technical design's query is correct as written; the PRD wording will
    be updated to match, no technical design change needed.
  - **Follow-up on that item:** the eligible-signer pool this query resolves can itself contain more
    than one person (the attending MD plus one or more PA/NPs), and neither document previously
    stated outright that the user picks exactly one of them — it was only inferable from `sentTo`
    being a single field. Since the Create action (Section 5.1) also "opens a picker," but is
    multi-select, this is worth making explicit rather than relying on inference, so whoever builds
    the Send-for-signature picker doesn't build it as multi-select by analogy. The technical design
    now states this outright in three places (§3.3, §3.4). Recommended PRD addition to the Send for
    signature row (Section 5.4): "**The picker is single-select** — the user chooses exactly one
    recipient, even when it lists more than one eligible signer."

- **"No fixed cadence" change (PM-driven PRD update) — full TD cascade.** The PRD's assumption that
  IDT reports are produced on a fixed weekly cycle has been removed at the PM's direction: a weekly
  rule isn't standard practice across facilities and doesn't fit long-term vs. short-term-stay
  residents equally, so reports are now created whenever staff need one — the app neither decides nor
  exposes a configurable cadence setting. Every "week"-based reference in the PRD (report frequency,
  carry-forward context, the rehab progression table's "2 Weeks Ago / Last Week / This Week" columns,
  etc.) was reworded to report-sequence language ("previous report," "2 Reports Ago," "report-over-
  report"). This has a real technical design impact, now applied in `feature-idt-reports-v2_TD.md`:
  - The `idtreports.weekOf` field is removed entirely (it has no meaning without a week concept),
    along with its old unique index `{facilityId, residentId, weekOf}` that enforced "one report per
    resident per week." `idtReportDeletionLogs`' equivalent field is likewise removed.
  - Replacement invariant: a new MongoDB **partial unique index** on `{facilityId, residentId}`,
    filtered to `status: 'DRAFT'` (`partialFilterExpression`), enforcing "at most one open Draft per
    resident at a time" — Complete reports are unconstrained, so a resident can accumulate any number
    of them over time. This was originally an SA recommendation pending confirmation (item 14 below);
    **now confirmed** — see the follow-up entry below.
  - A new general (non-unique) index `{facilityId, residentId, reportDate}` was added to preserve the
    by-resident lookup the old unique index incidentally provided for carry-forward, rehab-trend, and
    the CM/SW live-join queries.
  - The rehab progression trend section (TD §3.3, §3.4) is relabeled "report-over-report," ordered by
    `reportDate` descending instead of calendar week.
  - The `weightTrendSnapshot` feature's own calendar-week bucketing (PRD §4.1) was, at the time,
    deliberately left unchanged — it reflected a separate, real PCC/regulatory weighing cadence, not
    the report cadence, with only its anchor-point language updated to derive from `reportDate` instead
    of the removed `weekOf`. **[Superseded, round 3]** PRD §4.1's wording was later changed to a plain
    3-reading rolling set with no calendar-week grouping at all, and Sathish has since confirmed that's
    the correct behavior — see the round 3 entry below; `weightTrendSnapshot` no longer buckets by
    calendar week in any way.
  - Bulk report creation and bulk delete (TD §3.9) — exclusion and failure-handling language updated
    from "already has a report this week" to "already has an open Draft."
  - **Flagged at the time, now resolved (see follow-up below):** PRD §9's Non-functional Performance
    bullet still read "52 reports per resident per year," a figure derived from the old weekly cadence
    — the PM's edit hadn't touched it, so it looked internally inconsistent with the rest of the
    cadence-removal.
- **Follow-up: PM resolved both cadence-cascade open items (14 and 15) directly in the PRD.**
  - **Item 14 (Draft-only replacement invariant)** — PRD §6 gained a new rule 5: "a resident has at
    most one open Draft at a time," with the reasoning spelled out (staff-driven creation with no
    fixed cadence means nothing else would stop two simultaneous open Drafts existing for the same
    resident). PRD §5.1's Create picker was updated to match: "eligible residents" is now explicitly
    defined as everyone **except** those who already have an open Draft. This confirms the TD's partial
    unique index (§3.6) and the picker's exclusion filter (§3.9) exactly as designed — no design change
    needed, just the open item closing out as confirmed rather than recommended.
  - **Item 15 (52 reports/resident/year figure)** — PRD §9 keeps the figure, but now states outright
    that it's "a planning-ceiling placeholder, not derived from cadence, to be revisited with real
    deployment data." This resolves the inconsistency directly: the number stays, its framing changes.
    No TD design change needed; the TD's existing caveats (§2.1, §3.2, §7) already anticipated this and
    are updated to reflect it's now a confirmed PM position rather than an open SA flag.
  - Both items are now closed in `feature-idt-reports-v2_TD.md` §11 (moved from "still genuinely open"
    to the resolved summary) and in the corresponding schema/risk tables (§3.6, §4, §10).
- **Sathish resolved nine of ten remaining open items in one pass (former TD §11 items 3–9, 11–12).**
  Applied in `feature-idt-reports-v2_TD.md`, dropping the open-questions table from 15 tracked items
  down to 4 (items 1, 2, 10, and 13 — item 10 explicitly stays open; the other three were pre-existing
  and intentionally still open, not new).
  - **Item 3 (auto-populated data staleness)** — resolved as a user-triggered **Refresh** action, not a
    passive staleness indicator. One button on an open report re-runs the exact same
    PCC/backend-resolution logic used at Create time for all three auto-populated fields
    (`patientInformationSnapshot`, `weightTrendSnapshot`, `medicationsSnapshot`) and returns the current
    values to the form; nothing is written until the user's next ordinary Save, so no new persistence
    path or version-row shape is needed. Available at any point in a report's life (including after
    signature, where a refresh-then-save is just an ordinary silent correction) — see TD §3.3, §6, §8.
  - **Items 4 and 5 (facility-settings rollout)** — both reframed from open design questions into
    concrete rollout/deployment steps in TD §9, a section that already existed for the data-cleanup
    plan. `eligibleSignerDesignations` becomes a required per-facility go-live task (still starting with
    the one-time engineering discussion to settle the designation taxonomy, now flagged in TD §11 no
    more). `versionRetentionYears` needs no rollout work at all — the application defaults it to 10 when
    a facility's settings document doesn't have it set, confirmed as sufficient without a backfill.
  - **Item 6 (storage tiering)** — moved out of Open Questions entirely into a new TD §12, "Deferred
    technical items (candidates for future epics)." This is a new section for tracking
    engineering-only deferrals that don't belong in the PRD (no user-facing behavior changes) but
    shouldn't get lost before Epics/Stories are drafted — currently holds just this one item.
  - **Item 7 (bulk-create landing behavior)** — confirmed: return to the list, and the list must
    refresh (re-fetch, not just rely on stale client state) so the newly created Drafts are visible
    immediately. TD §3.9.
  - **Items 8 and 9 (partial-failure UX and no selection cap)** — resolved together as one consistent
    pattern applied to both bulk actions: a toast/alert naming the skipped residents/reports and the
    reason, truncating to "+N more" for a long list, with the user able to retry a skipped item
    individually from the list's existing single-item actions. Suggested copy is now written into TD
    §3.9 (bulk-create) and §3.10 (bulk-delete). No cap on bulk-create's selection size — the existing
    exclusion filter already prevents most collisions before they happen, and the toast handles the rest.
  - **Item 10 (Create picker's resident display-name field) — stays open, at Sathish's explicit
    request.** An initial answer to reuse `patientInformationSnapshot.residentName` turned out to be
    based on a mix-up: the Create picker queries the `residents` collection directly (it must list
    residents with zero reports too), so it can't literally read that snapshot field the way the main
    list does. Once that was clarified, Sathish asked to leave the exact field name open for
    engineering rather than deciding it here. TD §3.9 keeps the recommendation — use whichever
    `residents` field already feeds `patientInformationSnapshot.residentName` at Create time, for
    consistency — but marked explicitly as a recommendation only, not a confirmed decision; the field's
    actual name needs confirming against the real `residents` schema, which wasn't visible in this
    review.
  - **Items 11 and 12 (rehab.markedComplete reset; "Start blank" reset scope)** — both confirmed exactly
    as this design's original assumptions in TD §3.3: the checkbox always resets to `false` on a new
    report, and "Start blank" stays a lighter client-side-only reset rather than an immediate backend
    overwrite of the persisted Draft.

- **PRD round-2 handoff (`idt-reports-handoff-round2.md`) reviewed and resolved against the TD.**
  All eight changes plus the one reconfirmation item were checked against `feature-idt-reports-v2_TD.md`
  and the design updated accordingly. Seven of the eight were straightforward — no open questions,
  either no schema/design impact (Nurse persona removal, `mm/dd/yy` date format, status-filter removal)
  or a design that already had the needed field and just needed exposing (Fax free-text + US phone
  validation, Fax History via the existing `faxLog` with no new endpoint, the Signature Status column
  via the existing `signature.state`, and the Edited-after-signature indicator via the existing
  `signature.editedAfterSignAt`). The Create-picker open-Draft exclusion was reconfirmed as-is, no
  change needed.
  - **The one substantive item — "Refresh now moves the report date on Save" — surfaced a genuine
    design problem, not just a schema update.** `reportDate` was this design's sole ordering/anchor key
    for carry-forward's "most recent report," the rehab progression trend's "two most recent prior
    reports," the general by-resident lookup index, and the retention-cleanup anchor date. Making
    `reportDate` mutable (as the PRD now requires) means a report could jump ahead of a genuinely newer
    one in "most recent" ordering, or reset its own retention clock indefinitely, just by being
    refreshed and saved periodically — exactly the ripple effects the handoff document itself flagged
    for review.
  - **Resolution (SA recommendation, not yet confirmed):** decouple the two roles. `reportDate` keeps
    its new PRD-defined meaning for display/filing only; every actual sequencing/retention use switches
    to the already-existing, genuinely-immutable `createdAt` field. Detecting "this save follows a
    Refresh" is done server-side, by diffing the incoming snapshot fields
    (`patientInformationSnapshot`/`weightTrendSnapshot`/`medicationsSnapshot`) against what's currently
    persisted at Save time — no new client-supplied flag needed. Full detail in TD §3.3's dedicated
    "`reportDate` mutation" block, with the corresponding updates carried through §3.4 (field notes),
    §3.6 (two renamed indexes), §3.8 (retention anchor), §4 (rejected alternative recorded), and §10
    (two new risk rows).
  - **Confirmed by Sathish** (formerly TD §11 item 16, now closed): "Use `reportDate` field to capture
    the revisions of the report date" — i.e. `reportDate`'s job is specifically to capture the report's
    date as revised by a Refresh + Save, while `createdAt` is the sole key for ordering/retention. This
    is exactly the design as proposed; no PRD change is needed, since the PRD's silence on `createdAt`
    isn't a contradiction, just a level of detail it doesn't need to carry.
  - One new traceability-only open item was also added: **TD §11 item 17**, a one-line confirmation
    that no code path grants report-creation rights to a "Nurse" designation specifically, now that PRD
    §3 narrows the creating persona to Case Manager only. This design's Clinical Staff matrix never
    included a Nurse designation, so no change is expected — just a sanity check for whoever owns the
    permissions model.

- **Sathish's own design-review pass — 13 comments against the finished TD, plus the real `residents`
  schema.** Reviewed and resolved as follows:
  - **Grid sort/display (items 1, 2, 4) — resolved.** The list's default sort switches from `createdAt`
    back to `reportDate` (TD §3.6), to match the "Report Date" column the PRD actually displays (§5.1)
    — a distinction that only started to matter once `reportDate` became mutable. Report Date is now
    also recommended for display on the Edit and Summary pages (no schema/API change, PRD content only
    — see the suggested wording below). The unsaved-changes confirmation dialog (already committed in
    PRD §10, but never captured in the TD) is now documented with an explicit test case (TD §6, §8).
  - **Carry-forward/rehab-trend ordering (items 3, 5) — reviewed again, no change.** Sathish asked why
    not sort these by `reportDate` alone, since a Refresh naturally makes it the latest. Reviewed and
    reconfirmed: Refresh never touches `rehab.pt`/`rehab.ot` or other team-entered content, so an old
    report refreshed-and-saved isn't actually more clinically current — only its `reportDate` label
    moved. `createdAt` stays correct for these two uses (TD §3.3).
  - **Edited-after-signature indicator (item 6) — resolved, PRD update suggested.** The form sub-title
    indicator will now show the current signature status alongside the date, not just the date alone
    (TD §3.3, §3.4, §6). Suggested PRD wording for the PM: replace "Edited after signature [date]" with
    something like **"Signed — edited after signature on [date]"** in Section 5.2/6 rule 4/7 rule 4,
    so the indicator carries both facts without requiring a separate check of the Completed tab's
    Signature Status column.
  - **Weight trend (item 7) — designed against the real `observations` schema; fully resolved, former
    TD §11 item 19 closed and removed from the open-questions table.** Weight trend data doesn't exist
    as a resident-level capability yet. With the actual `observations` collection reviewed (a shared,
    append-only, field-level-encrypted event log, one document per PCC observation event), the design is
    now concrete: a new `residentObservationTrends` collection, one document per `residentId`+`type`,
    holding a fixed-size 3-reading ring buffer (the latest reading replaces the oldest on every update —
    no calendar-week bucketing), upserted by the same webhook that writes to `observations` (TD §3.4).
    The earlier window-advance question is resolved — no scheduled job is needed, since staleness is a
    read-time comparison against today's date, not stored state. All three points that previously needed
    sign-off are now confirmed by Sathish: (1) retention scope — `residentObservationTrends` keeps only
    the 3 most recent readings, while `observations` is untouched and keeps its full per-event history as
    the audit-of-record; (2) encryption — vitals (including weight) are PHI, so this collection stays
    field-level encrypted exactly like `observations`, no relaxation; (3) `recordedAt` staying plaintext
    in the new collection (needed to order/evict readings), the same way `observations` already carries
    plaintext `createdAt`/`updatedAt`, is confirmed acceptable. HIPAA's Security Rule currently treats
    ePHI encryption as "addressable" rather than flatly mandatory, but encrypted PHI gets Breach
    Notification Rule safe-harbor treatment, making encryption the de facto standard regardless — a
    January 2025 HHS NPRM would make it mandatory outright but isn't yet finalized (OMB targets
    ~mid-2027). Net effect: encryption stays as designed either way. The proposed object shape and
    approach are confirmed as-is — no design changes remain. Also added, per Sathish's explicit request:
    a later-phase deferred item (TD §12) for detecting resident data that's gone stale for over a week,
    attempting an on-demand PCC pull, and notifying assigned staff if it's still stale — not part of this
    release. This whole area is a cross-feature dependency, not part of the IDT Reports codebase itself —
    see the epics note below.
  - **Case Manager/Social Worker assignment gating (item 8) — resolved, reverses an earlier confirmed
    decision.** Sathish corrected an assumption confirmed in an earlier round ("any Case Manager or
    Social Worker can act on any report they can reach, regardless of assignment"): the staff member
    must actually be assigned to the resident. This is now a real server-side authorization check, not
    just a UX default on the Create picker (TD §6, §7) — the corresponding risk and test entries have
    been inverted to match (TD §8, §10). The real `residents.assignedStaff` field turned out to be a
    flat array of staff identifiers with no per-designation structure, confirming the cross-reference
    approach (`Staff.designation`) already assumed elsewhere in the design. Two assumptions carried into
    this reversal still need a quick confirmation rather than being treated as settled: whether the same
    requirement extends to the Rehab Team/Director of Rehab role, and that Directors keep their
    facility-wide bypass.
  - **`createdByType`/`updatedByType` (item 9) — resolved, removed.** Sathish confirmed these are
    remnants of the previous IDT report structure, not required in this design. Both fields are
    dropped from `idtreports` (TD §3.4); no companion actor-type field is needed anywhere else in the
    schema. Former TD §11 item 18 is closed.
  - **Resident display-name field (item 10, formerly TD §11 item 10) — resolved.** The real `residents`
    schema has a pre-composed `name` field (e.g. `"Shariin Abreuu"`), distinct from `firstName`/
    `lastName` — confirmed as the field for the Create picker's display name, sort, and search (TD §3.9).
    This closes an item that had been open since round 1.
  - **`pdfUrl` implementation reference (item 10 of the original 13) — resolved.** Cross-referenced to
    the existing render pipeline (`buildIdtReportHtml`, `idtReport.pdf.service.ts`) directly in the
    field's Notes (TD §3.4).
  - **`idtReportVersions.content` snapshot gap (item 11) — resolved, a real bug caught.** The field's
    notes previously said the three auto-populated snapshots were "invariant across every version,"
    which was true before `reportDate`/Refresh became mutable and is false now. Fixed: every version row
    now captures all three snapshots, not only the team-entered content — otherwise a post-signature
    refresh would leave no record of what the auto data looked like at signing time, breaking the PRD's
    "must render years later exactly as signed" requirement (TD §3.3, §3.4, §10).
  - **`idtReportVersions.residentId` (item 12) — resolved.** Added for the same reason `facilityId` is
    already denormalized onto that collection — direct resident-scoped audit queries without a join
    back through `idtreports` (TD §3.4).
  - **Bulk failure detail UI (item 13) — resolved.** Added a note for the frontend/engineering team
    recommending a "view details" affordance for the full skipped-item list, beyond the toast's "+N
    more" truncation — no new API needed, the bulk endpoints already return the full per-item data
    (TD §3.9, §3.10).

- **Team review of the PRD and TD, round 3 (`idt-reports-handoff-round3.md`) — 9 PRD changes reviewed
  against the TD.** See the full breakdown under "PRD Review — Round 3" above; summarized against the
  TD here:
  - **Refresh consolidation (item 5) — already matched, no change.** TD §3.3's Refresh design was
    already one control across all three auto-populated sections.
  - **Diagnosis rename (item 6) — resolved.** `patientInformationSnapshot.chiefComplaint` renamed to
    `.diagnosis` (TD §3.4); PM confirms PCC-sourced, closing the Diagnosis half of former TD §11 item 1.
  - **Signer designation taxonomy (item 7) — resolved, smaller than previously designed.** No facility
    setting needed; the query reads `staff.mobile_access == 'Doctor'` directly (TD §3.3), removing the
    `eligibleSignerDesignations` facility setting, its rollout step, and its risk row (TD §3.7, §9, §10).
  - **Rehab clear-on-deactivate and mark-complete-disabled (items 2, 3) — fully resolved, newly
    specified.** Two new frontend behaviors, fully designed in TD §3.4/§6. **[Confirmed by Sathish]**
    item 2 clears all of the discipline's fields on deactivation, including the free-text Goal field,
    and re-enabling the toggle afterward starts fresh rather than restoring the cleared data — both of
    the PRD-acknowledged open sub-questions are now answered, closing former TD §11 item 23.
  - **Rehab Team scope and authorization mechanism (items 1, 8) — partially resolved, correctly left
    open on the mechanism.** Rehab Team is confirmed no longer limited to "Director of Rehab," and the
    resident-assignment requirement is confirmed to extend to it — but *how* the system determines Rehab
    Team membership is explicitly reserved for Engineering (new TD §11 item 21), not decided here or by
    the PM.
  - **Director bypass — corrected, not newly created.** This design previously carried the Director's
    facility-wide bypass as an unstated assumption; corrected to reflect its real status as a formally
    open PRD question (PRD §11 item 2, new TD §11 item 20) that this design does not assume settled.
  - **Date format (item 9) — flagged, not decided.** Low priority; new TD §11 item 22.
  - **Encryption (item 4) — new requirement, now confirmed by Sathish as Architect.** The recommended
    field-level encryption scope for `idtreports`/`idtReportVersions` in TD §3.4a, reusing the existing
    `observations` per-field AES-256-GCM pattern, is confirmed as the design to build — including its
    most significant implementation wrinkle, that carry-forward must decrypt the prior report's content
    and re-encrypt it fresh (new IV) rather than copy ciphertext directly. Sathish asked whether a
    better alternative exists; none does — copying ciphertext as-is only defers the cost until the
    content's first edit and adds a fragile special case; deterministic encryption trades away semantic
    security; per-document data keys don't avoid the decrypt/re-encrypt step; referencing the source
    document instead of copying would reopen the `idtReportVersions` gap already fixed elsewhere in this
    design. Full analysis in TD §3.4a. This closes former TD §11 item 24 — the Report data model &
    lifecycle epic can now be scoped as ready on this dimension.
  - **Weight-trend calendar-week inconsistency — discovered independently, not a round 3 handoff item —
    now resolved.** PRD §4.1's "no calendar-week grouping or bucketing" wording for the weight trend
    didn't match `weightTrendSnapshot`'s former calendar-week-slot design (TD §3.4). **Confirmed by
    Sathish:** the PRD wording is correct — `weightTrendSnapshot` is simplified to directly mirror
    `residentObservationTrends`'s existing 3-reading ring buffer, with no calendar-week logic anywhere.
    Closes former TD §11 item 25.

None of the remaining open items block Epics/Stories for the core report. **The encryption scope (TD
§11 item 24) is resolved** — the Report data model & lifecycle epic can be scoped as ready on that
dimension. Former open items 18 and 19 are both fully resolved as well (not merely pending external
input) — see the TD's §11 for their closing notes. Of the items opened this round (TD §11 items
20–25), only 20 (Director bypass), 21 (Rehab Team mechanism), and 22 (date format, low priority) remain
open — items 23, 24, and 25 are all now fully resolved and closed. Two of the three remaining items are
reserved for Engineering or a separate PM/Sathish decision (20, 21); the third (22) is flagged
low-priority and non-blocking.

## Escalation
**3 review rounds is the threshold.** If a PRD is still bouncing between Product Manager and System
Architect after 3 rounds without landing on Approved-as-is or Approved-with-changes, stop iterating
and escalate to Sathish for a direct decision rather than continuing to bounce it. This is a manual
discipline right now, not an automated check — the `review_round` field exists so it's easy to see
the count at a glance. The Technical Design Review section above isn't subject to this same round
count — a TD that needs more than one pass just gets more entries under Findings, dated, until it
reaches a verdict.
