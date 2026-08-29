# IDT Reports — Change Summary for SA / Technical Design Update (Round 3)

**Purpose:** everything below changed in the PRD (`prd-idt-reports-v2.md`) since the last technical design sync (round 2, `idt-reports-handoff-round2.md`, already reflected in `feature-idt-reports-v2_TD.md` per the round-3 SA comments). For each item: what changed, where it lives in the PRD, and what it likely means for the TD. Technical-design impact is my read, not confirmed engineering — the SA agent should verify against the actual schema/design before treating any of it as settled.

---

## 1. Rehab Team no longer scoped to a single "Director of Rehab" designation (PRD §3, §8)

**Change:** The Rehab Team persona is not limited to "Director of Rehab" — a facility's rehab team may include several designations (e.g. PT, OT, SLP, and/or Director of Rehab). PRD §8's Clinical Staff definition no longer names a single designation.

**TD impact:** The role-matrix check for Rehab Team (Create/Edit/Mark Complete, and the resident-assignment gating already in the TD) can no longer assume a single hard-coded designation. **See item 8 below — the exact mechanism is explicitly not decided and reserved for Engineering.** Do not build against an assumed field/list yet.

---

## 2. Rehab discipline toggle-off now clears ratings, with confirmation (PRD §5.2)

**Change:** Turning off a Physical Therapy or Occupational Therapy toggle (the two disciplines with chip-rated fields) after any ratings have been entered now prompts a confirmation that those ratings will be cleared. On confirm, the discipline's chip-rated assist-scale values (and, for PT, the numeric Gait distance) are cleared and the toggle turns off. No confirmation is needed if nothing was entered. Does not apply to Speech Therapy (no rated fields).

**TD impact:** New clear-on-deactivate logic (client confirmation dialog + a clear operation on the discipline's rated fields). **Left explicitly open in the PRD, not assumed:** (a) whether the free-text Goal field is cleared along with the ratings, and (b) whether re-enabling a toggle restores the previously-cleared data or starts empty. Needs a decision before this is built.

---

## 3. Rehab mark-complete checkbox disabled once report is Complete (PRD §5.3, §6 rule 2)

**Change:** Once the report's document status is Complete, the Rehab section's mark-complete checkbox becomes disabled/read-only (still shown ticked) — whether Complete was reached via an explicit tick + Submit, or implicitly via Submit alone. Rationale: document status only moves forward and can't revert to Draft, so the checkbox has nothing further to control at that point.

**TD impact:** Small, frontend-facing state change on an existing control. No schema impact expected.

---

## 4. New Encryption NFR — all IDT report data (PRD §9, §12)

**Change:** All IDT report data — both PCC-sourced auto-populated snapshots (Patient Information, Weight trend, Active medications) and staff-entered content (Clinical Information, Rehab Status & Cognition, Discharge Planning, Orders/Discussion) — must be field-level encrypted at rest, consistent with how other PHI on the platform is handled. Applies to `idtreports` and `idtReportVersions` alike, not only specific fields.

**TD impact:** The SA's prior design-review pass discussed encryption for the weight-trend/`observations` collection specifically, not the report collections themselves. **Needs Architect confirmation** of whether the current `idtreports`/`idtReportVersions` schema design already meets this requirement or needs a change to close a gap. New PRD §12 engineering follow-up item. Should be settled before the Report data model & lifecycle epic is built.

---

## 5. Refresh consolidated to a single control (PRD §4.1)

**Change:** The three per-section Refresh controls (Patient Information, Weight trend, Active medications) are replaced by **one Refresh control on the open report** that re-pulls current values for all three sections at once.

**TD impact:** If the TD's Refresh design assumed three independent triggers/endpoints, confirm whether they collapse to one client action calling the same underlying refresh logic for all three fields together, or whether the endpoint(s) themselves change. Everything else about Refresh (doesn't save by itself, available post-signature, moves report date on save) is unchanged.

---

## 6. "Chief Complaint" renamed to "Diagnosis"; confirmed sourced from PCC (PRD §5.2)

**Change:** The Patient Information field "Chief Complaint" is renamed to "Diagnosis." The PM confirms this field is available in PCC.

**TD impact:** Display-layer rename, no schema change expected. This resolves the Diagnosis half of the Spike's item 3 ("Chief Complaint / Code Status PCC-sync confirmation") — **Code Status's PCC-sync confirmation is still open.**

---

## 7. Eligible signer designation taxonomy — RESOLVED, no facility setting needed (PRD §12)

**Change:** After consulting Engineering, the previously-open "eligible signer designation taxonomy" item is resolved. **No facility-configurable list is needed.** The `staff` object already carries a `mobile_access` field, valued `Doctor` for every signer-eligible user (the attending MD and their PA/NP); the signer-resolution query for Send-for-signature should read this field directly.

**TD impact:** If the TD's signer-resolution design (§3.3/§3.7 per the round-3 SA comments) references a new `facilities.idtReportSettings.eligibleSignerDesignations` setting, **that setting is no longer needed** — replace with a query against `staff.mobile_access == 'Doctor'`. No new facility setting, no per-facility rollout step.

---

## 8. Rehab Team authorization mechanism — explicitly left open, Engineering-only (PRD §8, §11 item 3)

**Change:** Unlike item 7 above, the analogous question for Rehab Team designations was **not** resolved the same way. The PM was explicit: this should be answered by Engineering on its own, not decided by the PM, by AI, or by the SA agent — even though "a similar approach" to item 7 may end up applying. This is now tracked as a genuine open question, PRD §11 item 3 (medium priority), rather than a pre-decided engineering-follow-up item.

**TD impact:** **Do not design the Rehab Team authorization check against an assumed field (e.g. `mobile_access`), a hard-coded designation, or a new facility-configurable list.** This item, and the Permissions epic generally, should not be considered ready for story-writing until Engineering answers this directly. Please route this question back through the same engineering channel that resolved item 7, and report the answer back rather than deciding it within the TD.

---

## 9. Date format — raised as a possible facility-configurable setting, not yet decided (PRD §4.3, §11 item 4)

**Change:** The PRD's `mm/dd/yy` date format (previously stated as fixed product-wide) is now described as the *default*, with a new note: whether this should instead be a facility-configurable setting (the same pattern as `staleDraftCleanupDays`/`versionRetentionYears`) is raised for consideration but **not decided**. New PRD §11 item 4, low priority.

**TD impact:** If the TD's date-formatting work (grid sort/display fix, per the round-3 SA comments) assumes a single hard-coded `mm/dd/yy` format, flag that this may need to become a facility setting depending on how item 4 is resolved — don't build the hard-coded version as final yet. This is a low-priority, non-blocking consideration, not a confirmed requirement change.

---

# Net effect on scope

Two items (Rehab toggle-clear, Rehab checkbox disable) are new, well-scoped frontend/product changes ready to design against directly. One item (encryption) needs an Architect confirmation before its owning epic is built. One item (signer designation) is now *smaller* than previously designed — a facility setting comes out of scope entirely. One item (Rehab authorization mechanism) is *explicitly not yours or the PM's to decide* — please escalate to Engineering rather than resolving it in the TD. One item (date format) is a low-priority, not-yet-decided consideration that may or may not change scope later.

*Companion artifacts also updated today: `prd-idt-reports-v2.md` (all nine items above) and `IDT-Reports-Ticket-Review.xlsx` (`05_tracker-sync/`), which has all corresponding Tickets-tab rows marked "NEW," "UPDATED," "RESOLVED," or "OPEN QUESTION, ENGINEERING ONLY" for easy scanning.*
