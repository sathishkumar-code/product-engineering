---
feature: IDT Reports
product: SNF (Skilled Nursing) — Care Coordination module
slug: idt-reports-v2
version: 1.0
status: Ready for Development Lead & Architect review — Epic and Story creation
date: 2026-08-12
design_reference: https://claude.ai/design/p/980b2832-30a4-41af-bf6d-ab769392d42b (interactive prototype for the list, form, and summary screens — demonstrates required behavior for Section 5; known limitations are listed in Section 13)
---

# SNF IDT Report Automation

Product requirements for the Interdisciplinary Team report: a pre-populated form that replaces manual document compilation, collects each discipline's contribution, and carries the report through physician signature into the resident record.

| | |
|---|---|
| **Status** | Ready for Development Lead and Architect review — Epic and Story creation. |
| **Audience** | Development Lead and Architect (Epic & Story creation), Engineering team. |
| **Business goal** | Cut IDT report compilation from a multi-hour manual process to a few minutes of review and exception-editing, and remove the transcription errors that come with re-keying clinical data. |
| **Release shape** | One report per resident per IDT meeting, created as staff need it — there is no fixed cadence (Section 1, A1). Three surfaces: the report list, the report form, and the review/signature summary. |
| **Sources** | Interactive prototype (behavioral reference for Section 5), PCC as the upstream clinical system. |
| **Related** | SNF Director Operations Dashboard (shares the resident record and PCC sync). The IDT report neither reads nor writes the discharge runway. |
| **Replaces** | The existing IDT Reports capability. This is a full workflow and data-model replacement, not an incremental change. No data migration is needed — the existing collection holds only stale test records and is cleaned up on production rollout. See Section 6. |
| **Open questions** | 4 open — see Section 11. One is blocking: item 1 (archival semantics). Item 2 (resident-assignment gating for Directors, raised in design review) is medium-priority and not yet settled. Item 3 (Rehab Team authorization mechanism) is medium-priority and deliberately reserved for Engineering to decide — not the PM, the AI, or the SA agent. Item 4 (date format — fixed vs. facility-configurable) is low-priority and newly raised for consideration. |

---

## 1. Assumptions

- **A1.** The IDT report is produced whenever the team holds an IDT meeting for a resident — there is no fixed cadence (not necessarily weekly); real-world practice varies by facility and may reasonably differ between a long-term and a short-term-stay resident. Staff create a new report as they need one; the app does not assume or enforce a cadence. One report is the record of one meeting; there is no partial report split across two meetings.
- **A2.** The application reads all clinical data from the Shashi Care backend, which syncs from PCC. The report never calls PCC directly.
- **A3.** PCC remains the system of record for demographics, medications, diet, code status and weights. The report is a point-in-time snapshot of those values, not an editor for them.
- **A4.** Completing a report and signing it are two separate acts. The team marks a report **Complete** when the meeting's content is recorded; the physician's signature is a fact recorded on the report afterward. A report may be completed, signed, edited, and re-sent for signature multiple times over its life.
- **A5.** Disciplines contribute to one shared document rather than filing separate notes; the report shows per-discipline completion, not per-discipline documents.
- **A6.** All report data persists server-side. Nothing in this specification may be implemented against client-side or browser-local storage.
- **A7.** This release fully replaces the existing IDT Reports capability, including PDF export. It is not an incremental addition. No data migration is required — the existing feature has never been used in production, so its collection holds only stale test records, which are cleaned up on production rollout rather than carried forward — see Section 6.

---

## 2. Scope

### 2.1 In scope

- IDT Reports list: one row per report, with resident, report date, attending MD, per-discipline completion, document status, and a Delete action.
- Bulk delete of Draft reports from the list via multi-select (Section 5.1).
- Report form with auto-populated sections (patient information, medications, diet, code status, weight trend) and team-entered sections (condition change including skin & wound, rehab status and cognition, discharge planning, orders and discussion).
- Carry-forward of team-entered content from the previous report, with a one-click "Start blank."
- Review summary: marking a report Complete, sending it for signature, and editing a report after signature with the change tracked in the audit trail.
- Staff mobile app push notifications for the signature workflow: the physician is notified when a report is sent for signature, and all staff assigned to the resident are notified when the physician signs (Section 5.4).
- PDF export and print of the report.
- Faxing the report as a PDF from the summary.
- The status model (Draft/Complete) and its editing rules, applied identically on the list and inside an open report.

### 2.2 Out of scope

- Editing clinical source data — medications, diet, code status, weights and demographics are corrected in PCC, never here.
- Scheduling the IDT meeting, attendance capture, and meeting minutes.
- Billing, MDS coding, and PDPM classification.
- Care-plan authoring — the report may reference care-plan goals; it does not maintain them.
- Notifications beyond the two IDT-specific signature-workflow push notifications on the staff mobile app (Section 5.4) — email, SMS, in-app banners, or any notification not tied to sending for signature or signing are out of scope.
- Platform-level analytics and HIPAA compliance infrastructure (this feature must comply with existing platform standards; it does not define new ones).
- Bulk submit — reports are only ever marked Complete or sent for signature one at a time, since the chosen residents on a bulk selection may have different attending physicians.
- Dictation (see Section 10, Next phase).

---

## 3. Personas

| Persona | Release | Use of the report |
|---|---|---|
| **Case Manager** | This phase | Creates the report, reviews carried-forward content, records condition change and clinical information, and submits to the physician. |
| **Social Worker** | This phase | Completes discharge planning: destination, DME, caregiver, notes. |
| **Rehab Team (PT / OT / SLP)** | This phase | Completes rehab status per discipline (PT, OT, Speech Therapy) and a cognition narrative, plus weight-bearing status — no visit frequency (that belongs to the therapy plan of care, not this report) and no cognitive score. |
| **Attending Physician** | This phase | Reviews the summary and signs, or their **PA/NP — Physician Assistant / Nurse Practitioner** — signs on their behalf. Does not author sections. A report needing correction is edited by the team and re-sent for signature — there is no return-to-team or rejection flow. |
| **Director / Administrator** | This phase | Oversight: sees all reports and their document status and signature state, and can delete drafts. Does not sign. |

---

## 4. Data sources

### 4.1 Auto-populated fields

Every field below is drawn from the Shashi Care backend at the moment the report is created and is marked **Auto from PCC** in the UI. It is read-only in the report (A3). Values are **snapshotted onto the report**, not read live — a signed report must render years later exactly as it was signed.

**Refresh.** Because these values are snapshotted rather than read live, a report left open a long time as a Draft or while Awaiting signature can show stale auto-populated data. A **single Refresh control** on the open report re-pulls current values for all three auto-populated sections (Patient Information, Weight trend, Active medications) at once, replacing what's shown in each — this replaces an earlier per-section design with one control for the whole report, at the PM's direction. Refresh doesn't save by itself — the refreshed values persist the next time the user saves, the same as any other edit — and is available at any point in the report's life, including after signature (handled the same as any other post-signature correction, Section 6). **Saving after a Refresh also moves the report date (Section 4.3) to that save date** — since the auto-populated content now reflects that day, the report is re-filed under it.

| Field | Notes |
|---|---|
| Resident name, room, date of birth | Header identity block on the report and the list row. |
| Attending MD | Determines the default signer and the PA/NP offered alongside them (Section 5.4). |
| Active medications | Name, dose, route, frequency. The full active list as at the report date; appears in the summary in the same order. |
| Weight trend | The **last 3 weight readings on file**, each shown with its date, most recent last, with the direction of change. Readings are a rolling set of 3 — a new reading replaces the oldest once a fourth exists — with **no calendar-week grouping or bucketing**. If there are fewer than 3 readings on file, show **N/A** for each missing slot; if there are none at all, show **N/A** across the row. Compacted to a single row in the form. |

### 4.2 Carry-forward from the previous report

Creating a report for a resident who already has one pre-fills all team-entered fields from the most recent report and shows a banner naming its date.

- Auto-populated fields (Section 4.1) are always refreshed from current data and never carried forward — the banner states this explicitly.
- **Start blank** clears every carried-forward field in one action and dismisses the banner. It cannot be undone; the previous report is unaffected.
- Carry-forward reads the most recent report of any status, including a draft, so an abandoned draft doesn't lose recent context.
- Carried-forward text is normal editable content with no separate "unchanged" marking, in the form and in the summary. Free text is never diffed. **Rehab progress is the exception** — it is compared explicitly; see the rehab progression table in Section 5.4.
- The Rehab team's mark-complete checkbox (Section 5.3) does not carry forward — it always starts unticked on a new report, even though the rest of Rehab Status & Cognition's content does carry forward. It reflects a judgment about *this* assessment, not prior content.

### 4.3 Dates and time zone

- All stored timestamps are UTC; every date rendered is evaluated in the **facility time zone**.
- The report date **is the creation date** — set by the system, not directly user-editable, and the date the report is filed under. It is not the meeting date. **The one exception:** saving after a Refresh (Section 4.1) moves the report date to that save date, since the data now pertains to that day.
- The report date is shown in the header of the report form and the summary, in addition to the list column (Section 5.1) — it was previously list-only.
- Date format defaults to **`mm/dd/yy`** product-wide, everywhere a date is rendered (the report, list columns, summary, PDF). **Under consideration, not yet decided:** whether this should instead be a facility-configurable setting rather than a fixed value, the same pattern as the other facility-level settings in this document — see Section 11, item 4.

---

## 5. Screen specifications

### 5.1 IDT Reports list

| Element | Behavior & rules |
|---|---|
| Tabs | **In-progress** and **Completed**, split by document status (Section 6), not by date. In-progress holds every Draft; Completed holds every report marked Complete. In-progress is the default tab, and each tab explains itself on hover ("Reports still in progress (draft)" / "Completed reports"). Neither tab has a date horizon — a draft stays In-progress indefinitely (see the stale-draft cleanup rule in Section 9). |
| Resident, Report Date, Attending MD, Case Manager, Social Worker, Rehab Team | Standard list columns; report date and attending MD as defined in Section 4. |
| Signature Status (Completed tab only) | Replaces the old Status chip, which was removed from both tabs as redundant — the In-progress/Completed tabs already convey document status (Section 6). Only the **Completed** tab carries this column, showing the report's signature state (**Not sent** / **Awaiting signature** / **Signed**, Section 6); the In-progress tab has no equivalent column, since every row there is a Draft by definition and cannot yet be sent for signature. |
| Discipline columns | Each reads **Pending**, **In-progress**, or **Complete** for that discipline's section group (Section 5.3). A Complete report cannot show a Pending discipline — submission marks all three Complete. Clicking a cell opens the report scrolled to that discipline's section, which is briefly highlighted. |
| Search | A single full-width search field sits between the tabs and the filter row; matches resident name or room number within the selected tab, combining with the filters rather than replacing them. |
| Filters and sort | Attending MD, Case Manager, and Social Worker, with the live result count beside them; sort by newest, oldest, resident A–Z, or attending MD A–Z. Filter options are built from the reports actually present in the tab. There is no document-status filter — the In-progress/Completed tabs (above) already partition by status, so a redundant filter was removed. Session-only, not persisted per user. |
| Row click | A **Draft** row opens the editable form; a **Complete** row opens the read-only summary, from which the report can be reopened for editing (Section 5.4). There are no View or Edit icons — the row itself is the affordance and the status decides which mode it opens in. |
| Delete | The only action in the Actions column. Enabled on **Draft rows only**; on a Complete row it stays visible but disabled, greyed, with the reason on hover ("A completed report is part of the record and cannot be deleted") — every row in the Completed tab shows a disabled control rather than an empty Actions cell. Deleting opens a confirmation modal naming the report's date and resident and stating that only drafts can be deleted; its actions are *Keep report* and *Delete permanently*. Confirming deletes the draft permanently (hard delete) and shows a toast; the action cannot be undone. |
| Bulk delete | Each **Draft** row carries a select checkbox (Complete rows do not — they are not deletable, so they take no part in bulk selection). Selecting one or more rows surfaces a bulk action bar with the selected count and a single **Delete selected** action. Confirming opens a modal naming the count of reports and stating that only drafts can be deleted; its actions are *Keep reports* and *Delete permanently*. Confirming hard-deletes every selected report and shows a toast with the count; the action cannot be undone. There is no bulk submit or bulk send-for-signature — those stay single-report actions, since a bulk resident selection can span more than one attending physician (Section 2.2). If one or more selected reports can no longer be deleted — most likely because someone else just marked one Complete — those reports are skipped and the rest of the batch is still deleted. A toast names the skipped report(s) and why, for example: *"4 of 5 reports deleted. Skipped: Jane Doe's report — already marked Complete by someone else. Completed reports can't be deleted."* |
| Create | A single primary action opens a resident picker supporting multi-select. The picker lists **eligible residents** — every current resident **except those who already have an open Draft** (Section 6, rule 5) — alphabetically, and filters live as the user types. There is no limit on how many residents can be selected in one Create action. Confirming with one or more residents selected generates a new Draft report for each, pre-filled per Sections 4.1 and 4.2, and returns to the list, which refreshes so the new Drafts appear immediately. If one or more selected residents can no longer be created for — most likely because someone else created an open Draft for them in the moment before confirming — those residents are skipped and the rest of the batch is still created. A toast names the skipped resident(s) and why, for example: *"4 of 5 reports created. Skipped: Jane Doe — an open Draft already exists for that resident. You can create a report for her individually once her existing draft is completed."* |

### 5.2 Report form — structure

Sections appear in a fixed order and each is a card. Auto-populated cards carry an *Auto from PCC* badge; the rest are team-entered.

**Form sub-title.** Beneath the resident header, the form's sub-title line ends with a **"Signed — edited after signature on [date]"** indicator — for example, "Signed — edited after signature on 03/14/26" — as its last item, shown only once the report has been saved after being signed (Section 6, rule 4). Carrying the current signature status alongside the date means this one indicator is legible without also checking the Completed tab's Signature Status column (Section 5.1). This is the only surface where that fact is surfaced; the summary and list are unchanged.

| Section | Owner | Contents |
|---|---|---|
| Patient Information | Auto | Name, room, DOB, attending MD, admission date, **Diagnosis** (renamed from "Chief Complaint" at the PM's direction; confirmed available from PCC), diet, code status. |
| Clinical Information | Case Manager | Change in condition since the previous report (free text), the auto weight trend row, and **Skin & Wound Status as a free-text field within this same card.** Skin & Wound is not a section of its own — it belongs to the same clinical picture and the same author as the rest of Clinical Information. Wound measurements are typed, not structured (structured wound documentation is next-phase — Section 10). |
| Rehab Status & Cognition | Rehab Team | Heads with two fields: **Prior Level of Function** (free text) and **Weight-Bearing Status** (typeahead: As tolerated · Non weight bearing · Partial weight bearing · TTWB). Below that, one block per discipline, each with its own active/not-active toggle: **Physical Therapy** — a chip-rated row (assist scale: **Mod Ind · SBA · CGA · Min Assist · Mod Assist · Max Assist · TD**) for Bed Mobility, Sup-Sit Transfers, Sit-Stand Transfers, Gait, and DME Used; Gait additionally has a numeric **Distance … feet** field; plus a free-text Goal. **Occupational Therapy** — the same chip-rated assist scale applied to Upper Body Dressing, Lower Body Dressing, and Toilet Transfers; plus a free-text Goal. **Speech Therapy** — a single free-text Notes field (swallowing, communication, diet tolerance) and an **Import Report** action for an outside SLP report — no rated rows; Import Report is deferred to a later phase (Section 10) regardless. **Cognition** — one free-text field; there is **no cognitive score and no BIMS entry** — cognition is described in words, not rated. **There are no visit-frequency fields anywhere in this section** — frequency belongs to the therapy plan of care, not the IDT report. The section closes with its own **mark-complete checkbox** (Section 5.3, including the rule for when it's disabled). **Turning a discipline's toggle off (PT or OT — the two disciplines with chip-rated fields) after any ratings have been entered prompts a confirmation that those ratings will be cleared; confirming clears that discipline's chip-rated values (and, for Physical Therapy, the numeric Gait distance) and turns the toggle off.** Turning the toggle off with nothing entered needs no confirmation. **Not specified — flagged for confirmation:** whether the free-text Goal field is cleared along with the ratings, and whether turning a discipline back on restores its previously cleared data or starts empty; this PRD does not assume an answer to either. This clearing behavior does not apply to Speech Therapy, which has no rated fields to clear. |
| Discharge Planning | Social Worker | Destination, DME needed / owned, caregiver, planning notes. |
| Orders / Discussion | Case Manager | Decision chips **Continue Skilled Care** and **Discharge Date Set** (with its date), plus free-text orders and discussion notes. The two chips are **not mutually exclusive** — both may be selected at the same time, and when they are, both are simply stored and displayed as-is; no special handling. The discharge date **must be today or a later date** — only strictly past dates are rejected, to accommodate late entries. It represents an **anticipated** discharge date, not an actual/final discharge (it is not equivalent to running PCC's discharge process). Once set, the discharge date becomes part of the resident's data and must be consumable by any other part of the product that displays it — this is not local to the IDT report. |
| Medications Details | Auto | Active medication list as at the report date. |

### 5.3 Discipline completion

The three discipline columns on the list read **Pending**, **In-progress**, or **Complete** — this is the final, required copy. Pending is drawn with a dashed border so an untouched discipline is legible at a glance. Every cell is clickable and jumps straight to that discipline's section in the form. Only Rehab is asserted by a person; the other two are derived:

- **Rehab Team** — an explicit **"mark this section complete" checkbox** in the Rehab Status & Cognition section. **While the report is a Draft**, the column reads **Complete** only when that box is ticked, whatever the fields contain, and **In-progress** otherwise; the rehab team owns the judgement of when their assessment is done up to that point. **Submitting the report marks Rehab Complete as well, even if the checkbox was never ticked** — completion is implicit on submission, the same as for Case Manager and Social Worker below. If the report is later reopened for editing, the **checkbox displays as ticked by default**, so the form stays consistent with the Complete state it's already in. **Once the report is Complete, the checkbox is disabled (read-only, shown ticked)** — document status only moves forward and cannot be taken back to Draft (Section 6, rule 2), so the checkbox has nothing further to control at that point; it remains visible only to show that the discipline was completed.
- **Case Manager** (Clinical Information) and **Social Worker** — no completion control of their own. Their fields are optional: a report with an empty Clinical Information or Discharge Planning section is a legitimate report. While the report is a Draft, the cell reflects content only — **Pending** when nothing has been entered, **In-progress** once anything has. **Submitting the report marks all three Complete** — submission is the completion signal for Case Manager and Social Worker, unconditionally.
- **Nothing blocks Submit.** An unticked rehab checkbox or an empty section does not prevent marking the report Complete, and produces no warning dialog. This is consistent with — not an exception to — the rule above: submission completes all three disciplines regardless of what's been ticked or filled in.

### 5.4 Review, completion and signature

Completion and signature are independent acts (A4). Marking a report Complete is the team's act and needs no physician; sending it for signature is a separate action available once it is Complete.

| Step | Behavior |
|---|---|
| Submit | Marks the report **Complete** and opens the summary. Confirmed in a dialog that states the consequence ("Submit this report? This marks the report Complete for [resident]. You can keep editing it afterwards — it will not go back to Draft."). Does not involve the physician and does not send anything. Submitting an already-Complete report keeps it Complete and records the edit. |
| Send for signature | Enabled only once the report is Complete; disabled with the tooltip "Submit the report first" beforehand. Opens a picker listing the **attending MD and their PA/NP**, and records signature state **Awaiting signature** with the recipient and sent date. The picker is single-select — the user chooses exactly one recipient, even when it lists more than one eligible signer. Re-sending replaces the pending entry and appends to the submission history. Dispatches a push notification to the recipient on the staff mobile app (see **Staff mobile app notifications** below). |
| Signature | The signer signs the summary; the report records **Signed** with signer and date. Signature does not change the document status — a report is already Complete when it is sent. The signature itself is a **digital signature**, captured at the point of signing as part of the Send-for-signature flow — not a credential re-entry step-up, and not a separate third-party e-signature vendor integration. The specific capture mechanism (for example, click-to-sign against the authenticated session versus an embedded signing widget) is an engineering decision. Dispatches a push notification to all staff assigned to the resident on the staff mobile app (see **Staff mobile app notifications** below). |
| Editing after signature | A signed report can be reopened and edited any number of times. There is no addendum flow — an edit is just an edit. The save is recorded with its date and the user who made it in the audit trail (Section 7). The report form's sub-title (Section 5.2) shows a **"Signed — edited after signature on [date]"** indicator once this happens — the one place this fact is surfaced; the summary and list carry no stamp, badge, or warning. Whether to re-send for a fresh signature is the team's judgement, taken with the ordinary Send for signature action; the existing signature stands until they do. |
| Fax | Available from the summary alongside Print. Opens a small popover — *Fax report*, "Sends the full IDT report as a PDF cover fax" — with a single **free-text fax number field**, validated to a **US phone number format** (10 digits; formatted/masked as the user types). There are no saved contacts to pick from — the product only stores fax numbers for Home Health Agencies, which are not a recipient of IDT reports, so the field is free text every time. *Send fax* validates the number is present and correctly formatted, closes the popover, and confirms with a toast naming the number; *Cancel* dismisses it. Faxing is available regardless of document or signature status and never changes either. Each send is audit-logged with number, actor, and timestamp (Section 7), and appears in **Fax history** (below). |
| Fax history | A **Fax History** section on the summary, **collapsed by default**, expands to list every fax sent for this report — number, sent date/time, actor, and send status — sourced from the same audit log as the Fax write (Section 7). The exact status values shown (for example, a simple "Sent" versus a delivery/failure status from the transmitting fax service) depend on that service's integration — see Section 12. |
| Archival | On signature, the report is written to the resident record as a permanent, immutable document. Whether it also pushes to PCC as a Progress Note or document, and in what format, is open — see Section 11, item 1. |

**Staff mobile app notifications:** the staff mobile app already runs a signature push-notification workflow for two other document types — the Health Referral Order Summary and the Medication List. The IDT report joins that same workflow, with two notifications:

- **On Send for signature:** the attending physician (or their PA/NP, whichever was chosen in the picker) receives a push notification on the staff mobile app. The notification must identify the document type as an **IDT report**, so it reads distinctly from a Health Referral Order Summary or Medication List notification.
- **On Signature:** every staff member assigned to the resident receives a push notification on the staff mobile app confirming the report has been signed — not just whoever submitted it, since any of them may work on the resident's record further.

Both notifications ride the existing staff mobile app notification workflow (delivery mechanism, tap-to-open behavior, and notification-center listing follow that workflow's existing pattern); this PRD defines only the trigger, recipient, and IDT-identifying content for the two new notification events.

**Rehab progression (report-over-report):** for each PT and OT chip-rated task (Bed Mobility, Sup-Sit Transfers, Sit-Stand Transfers, Gait, and DME Used for PT; Upper Body Dressing, Lower Body Dressing, and Toilet Transfers for OT), the summary renders a **2 REPORTS AGO / LAST REPORT / THIS REPORT** trend table. This report's value is the rating entered in Rehab Status & Cognition; the two earlier columns come from the two prior reports. Cognition and Speech Therapy are free text, not rated, and are not part of this trend table. This is the one place the report compares across reports — free text is never diffed (Section 4.2) — and it exists because trajectory, not this report's absolute level, is what the physician is deciding on. Read-only.

- **"2 Reports Ago" and "Last Report" mean the two most recent prior reports for the resident**, populated in report sequence regardless of how far apart in time they actually fall — cadence between IDT reports is staff-driven, not fixed (Section 1, A1).
- If the resident has **fewer than two prior reports**, the corresponding column or columns show **N/A**.

**Empty-field display in the summary:** any field left empty — free text (Change in Condition, Skin & Wound Status) or structured (Discharge Planning Notes, Orders/notes) — shows an em-dash placeholder ("—"). This is unified across all field types; there is no field left blank or omitted.

### 5.5 Dictation

A microphone control is present on every free-text field. It is intentionally inert in this release — reference only, not an active feature. Dictation itself ships in a later phase (Section 10); the technical groundwork is tracked as engineering follow-up (Section 12).

### 5.6 PDF export and print

PDF export is required for this release. A PDF/print pipeline already exists for this report type and should be **reused and re-templated**, not rebuilt: the existing pipeline (`buildIdtReportHtml`, `idtReport.pdf.service.ts`) and its two existing endpoints — a direct-download render, and a generate-and-store variant that persists a `pdfUrl` on the report record — both carry forward unchanged.

- The rendered PDF must reflect this report's section layout: Patient Information, Clinical Information (including Skin & Wound Status), Rehab Status & Cognition (including the PT/OT assist-scale detail and the report-over-report rehab progression table), Discharge Planning, Orders/Discussion, and Medications Details.
- Both endpoints carry forward: an on-demand download from an open report, and a generate-once-store-forever variant whose `pdfUrl` is exposed from the report record — this is what the summary's **Print** button should call, rather than the browser's native print.
- The PDF must render a **signed** report faithfully, including the recorded signer and signature date, with no visual distinction for a report edited after signing — the PDF should reflect current content, consistent with how the summary itself behaves.
- Whether the PDF is also the archival artifact referenced in Section 5.4, or a separate thing, is open — see Section 11, item 1.

---

## 6. Status model

The report has exactly **two** document-status values:

| Status | Definition | What is permitted |
|---|---|---|
| **Draft** | Created, not yet marked complete. | Opens editable. Save, Submit, Delete. Cannot be sent for signature. |
| **Complete** | The team has marked the content final. | Opens as the read-only summary and can be reopened for editing at any point in its life. Save keeps it Complete; it can be edited, sent, and re-sent as often as needed, each edit stamped with its date and author in the audit trail. Not deletable. A correction after signature is silent — no stamp, no notification (Section 5.4). |

**Signature state** is a separate fact recorded on the report, never a document status. It takes one of three values and is shown inside the report, in the summary header, and — for a Complete report only — in the Completed tab's Signature Status column (Section 5.1):

| Signature state | Meaning and what is shown |
|---|---|
| Not sent | Never sent to a signer. "Not yet sent to physician." |
| Awaiting signature | Sent and waiting. Shows recipient and sent date. |
| Signed | Shows signer and signature date. A later edit does not change this display and adds no message — the edit is recorded in the audit trail only (Section 7). |

**Rules:**

1. Document status governs the *actions* available (Submit, Send, Delete), not whether the content can be typed into. A report is editable at every point in its life, including after signature; what changes is the consequence of saving (Section 5.4).
2. Document status only moves forward: Draft → Complete. There is no un-complete. Signature state may cycle without limit (sent → signed → edited → re-sent → re-signed), and that cycling never alters the document status.
3. A disabled control states why on hover rather than disappearing. This applies to every action — Send for signature on a Draft, and Delete on a Complete report, are both rendered disabled with their reason, never removed.
4. An edit made after a signature is recorded, and surfaced in one place: the report form's sub-title (Section 5.2) shows a **"Signed — edited after signature on [date]"** indicator once it happens, in addition to the ordinary audit trail entry (date and user). The summary and list carry no stamp, banner, or badge — the report continues to read as signed there until it is re-sent and re-signed.
5. **A resident has at most one open Draft at a time.** Since report creation is staff-driven with no fixed cadence (Section 1, A1), nothing else would stop two open, unsubmitted reports from existing for the same resident at once. A resident with an open Draft is not offered again in the Create picker (Section 5.1) until that Draft is marked Complete; there is no limit on how many Complete reports a resident accumulates over time.

**Data cleanup, not migration.** This replaces the existing IDT Reports feature, which used a three-value status (`Draft` / `Pending` / `Submitted`). No migration of existing data is required: the existing feature has not been used in production, so every record in its collection is a stale test record. On production rollout, the existing collection is **cleaned up (removed)** rather than mapped into the new status model.

---

## 7. Writes performed by the report

| Write | Trigger | What is permitted |
|---|---|---|
| Section save | Save on an open report | Records author and timestamp with the save. |
| Submit | Marking a report Complete | Records the completion timestamp and user. Sets document status to Complete. |
| Signed-at + signer | Signature by the attending MD or their PA/NP | Sets signature state Signed. Does not change document status or freeze content. |
| Signature-request notification | Send for signature | Dispatches a push notification to the recipient (attending MD or PA/NP) on the staff mobile app, identifying the document as an IDT report (Section 5.4). Does not change document or signature status. |
| Signed notification | Signature by the attending MD or their PA/NP | Dispatches a push notification to all staff assigned to the resident on the staff mobile app confirming the report has been signed (Section 5.4). Does not change document or signature status. |
| Edit-after-signing audit entry | Save on a signed report | Records date and user in the audit trail. Unlimited and append-only. Surfaced in the UI only as the form sub-title's "Signed — edited after signature on [date]" indicator (Section 5.2); the summary and list remain unchanged. |
| Deletion | Delete in the Actions column, or Bulk delete from the list (Section 5.1) | Draft only. **Hard delete** — the record is removed permanently, with only the audit entry retained. A bulk delete writes one audit entry per report deleted, identical to a single delete. Available to Clinical Staff and the Director. |
| Fax send | Send fax | Records number, actor, timestamp, and send status. Surfaced to the user in the Fax History section (Section 5.4). |
| Discharge date set or changed | Ticking "Discharge Date Set" and choosing a date in Orders / Discussion | Writes the anticipated discharge date to the **resident record** (not just the report) so it can be consumed by any other part of the product that displays it. This is the one field in the report that writes somewhere other than the report itself. |

The report writes nothing back to **PCC** in this release. It never modifies medications, diet, code status, weights, or demographics — those remain PCC-sourced and read-only. The discharge date write above is internal to the Shashi Care resident record, not a write to PCC.

---

## 8. Permissions

Permissions must be enforced **server-side**. Hiding an action in the UI is not sufficient — every action below must be checked on the backend regardless of what the client sends.

| Action | Clinical Staff | Physician | Director |
|---|---|---|---|
| View the list and any report | Yes | Yes | Yes |
| Create a report | Yes | No | Yes |
| Edit a report (any status) | Yes — any section, including another discipline's | No | Yes |
| Mark Complete (Submit) | Yes | — | Yes |
| Send for signature | Yes | — | Yes |
| Sign | No | Yes — the attending MD or their PA/NP | No |
| Delete a draft | Yes | No | Yes |

**Concurrency:** last write wins when two people edit the same report at once — no lock, no merge, no warning. Every save is versioned with its author, so an overwrite is recoverable from the audit trail (Section 7).

**Clinical Staff** in this table refers to the roles who can edit the report: the **Case Manager**, the **Social Worker**, and the **Rehab Team**. The **Physician** is staff as well but is not included under "Clinical Staff" here — their permissions are the separate column in this table.

**Rehab Team designations.** The Rehab Team is not limited to a single "Director of Rehab" designation — a facility's rehab team may include several designations (for example PT, OT, SLP, and/or Director of Rehab). **How the system determines which staff count as a Rehab Team member for authorization purposes is an open question, deliberately left to Engineering to decide — not settled by this PRD, and explicitly not to be resolved by the PM, the AI, or the SA agent.** This follows the same pattern as the signer-resolution question below: after consulting Engineering on that question, a similar existing-staff-attribute approach may apply here too, but the exact field, value(s), and query are unsettled — see Section 11, item 3.

**Resident assignment.** A Case Manager, Social Worker, or Rehab Team member can only Create, Edit, or Mark Complete a report for a resident they are actually assigned to — this is enforced server-side, not just a default on the Create picker (Section 5.1). This reverses an earlier position (any Case Manager or Social Worker could act on any report they could reach, regardless of assignment); confirmed to extend to the Rehab Team the same way — evaluated against the facility-configured Rehab Team designation list above. **One point is not yet settled — see Section 11:** whether the Director's facility-wide access is exempt from this requirement.

---

## 9. Non-functional requirements

- **Performance.** Sized to 500 residents per facility and 52 reports per resident per year. **This 52/year figure is a planning ceiling only, not derived from any cadence** — report creation is staff-driven with no fixed cadence (Section 1, A1), so it is retained purely as a round, conservative number for the performance model, to be revisited once real deployment data is available. At that volume, search combined with the Attending MD / Case Manager / Social Worker / status filters and sort must reliably narrow the list to a specific resident's report. The underlying list-rendering strategy — unpaginated fetch, server-side pagination, infinite scroll, or virtualization — is an engineering decision; what's required is that the user never has to manually page through the full set to find one report.
- **Version integrity.** Content is not frozen by signature, and edits after signing are invisible in the UI, so versioning carries the whole burden: every save is a version with author and timestamp, and the exact content that was signed must be reproducible on later reads alongside the current content.
- **Audit.** Every write in Section 7 is logged.
- **Persistence.** Server-side. No requirement in this document may be satisfied by client-side or session-only state.
- **Accessibility.** Standard platform accessibility conformance applies.
- **HIPAA.** Standard platform compliance requirements apply; this feature introduces no new data-handling category.
- **Encryption.** All IDT report data — both auto-populated snapshots sourced from PCC (Patient Information, Weight trend, Active medications) and staff-entered content (Clinical Information, Rehab Status & Cognition, Discharge Planning, Orders/Discussion) — must be field-level encrypted at rest, consistent with how other PHI in the platform is handled. This applies to the report and its full version history alike, not only to specific fields. Confirming the current encryption posture of the report's schema against this requirement, and closing any gap, is tracked as engineering follow-up — see Section 12.
- **Stale-draft cleanup.** A scheduled job clears drafts untouched for 15 days (facility-configurable; 15 is the default), evaluated in the facility time zone, and audit-logged like any other deletion. Implementation mechanics are tracked as engineering follow-up — see Section 12.
- **Record retention.** A report and its complete version/audit history (Section 7) are kept for a facility-configurable retention period, defaulting to 10 years, before being purged — matching standard SNF medical-record retention practice out of the box, while letting a facility with a stricter state requirement override the default.

---

## 10. Next phase

Deferred deliberately; not required for this release.

- Push the signed report to PCC as a document or Progress Note.
- Structured wound documentation with measurements and staging (Section 5.2 keeps Skin & Wound as free text for now).
- **Import Report** for Speech Therapy (Section 5.2) — the button is not built in this release.
- Dictation into every free-text field — the control is already present but inert (Section 5.5).
- Autosave of in-progress drafts, replacing explicit-save-only. This release ships manual Save with an unsaved-changes guard.
- Per-discipline sign-off, replacing derived completion (Section 5.3) with an explicit action.
- A physician-side queue view across residents, rather than the shared list. This release records who a report was sent to but builds no queue surface.
- Cadence prompting — flagging a resident who appears overdue for a new IDT report, once real deployment data shows what typical intervals look like per facility and per resident type (long-term vs. short-term stay). Any such rule would need to be configurable rather than a fixed calendar rule, consistent with there being no fixed cadence in this release (Section 1, A1).

---

## 11. Open questions

| # | Area | Question | Priority |
|---|---|---|---|
| 1 | Archival | What does "archived into the patient record" mean — a record kept in the app only, a document pushed to PCC, or both? If both, which PCC object, and what happens when a report is edited after the original was already pushed — a replacement, a new version, or a second document? **Resolution depends on confirming whether PCC's document APIs support updating or versioning an already-pushed document, or only creating a new one each time** — this is a technical capability question that needs an answer before the product question can be decided. **Blocks archival stories.** | Blocker |
| 2 | Resident-assignment gating — Director bypass | Section 8's new resident-assignment requirement is confirmed for Case Manager, Social Worker, and the Rehab Team. Does the Director's existing facility-wide access remain an exception to it, or should Directors also be scoped to assigned residents? Raised in design review, not yet settled. | Medium |
| 3 | Rehab Team authorization mechanism | How the system determines which staff count as a Rehab Team member for authorization purposes (Section 8) is deliberately left open for Engineering to decide — this is not a product decision to be made in this PRD, and it is explicitly not to be resolved by the PM, the AI, or the SA agent either. Following engineering consultation on the analogous signer-resolution question (Section 12), a similar existing-staff-attribute approach may apply here too, but the exact field, value(s), and query are unsettled. | Medium |
| 4 | Date format — fixed vs. facility-configurable | Section 4.3 currently fixes the date format product-wide as `mm/dd/yy`. Raised for consideration, not yet decided: should this instead be a facility-configurable setting (like `staleDraftCleanupDays` or `versionRetentionYears` in Section 9) rather than a single hard-coded value, so a facility could choose its own preferred format? | Low |

---

## 12. Engineering follow-up

Decided at the product level; the remaining work is technical, not a product ambiguity.

- **Dictation transcription (Section 5.5).** Which transcription service, whether a medical vocabulary is required, audio retention policy and HIPAA posture, and what the user sees on a failed transcription are not answerable by product decision — they need a timeboxed research spike. Deliverable: a recommendation covering (a) two or three candidate services compared on medical-vocabulary accuracy, latency, and cost per hour; (b) BAA availability and HIPAA posture for each, including whether audio is retained by the vendor and for how long; (c) streaming versus record-then-upload for a bedside device; (d) failure and offline behavior the user sees; (e) a build estimate for the append-to-field integration. Owner: Development Lead, with Architect input. Blocks dictation stories only — nothing else in this release.
- **Stale-draft cleanup mechanics (Section 9).** The 15-day default is decided. Where the setting lives, who can edit it, and whether cleanup is a hard delete or an archive is an architecture decision for the Architect. Does not block story creation for the report itself.
- **Retention-cleanup mechanics.** The retention period is a per-facility setting, defaulting to 10 years (Section 9). Exact facility-configuration surface, whether cold-storage tiering is worth adding within the retention window purely as a cost optimization, and the safe rollout of the cleanup job (a dry-run period before enabling deletion) are architecture decisions for the Architect. Does not block story creation for the report itself.
- **Eligible signer designation taxonomy — RESOLVED, after engineering consultation.** No facility-configurable list is needed. The `staff` object already carries a `mobile_access` field, valued `Doctor` for every signer-eligible user (the attending MD and their PA/NP); the signer-resolution query for Send-for-signature reads this field directly. No new facility setting and no per-facility rollout step are needed for this item.
- **Fax send status granularity.** The Fax History section (Section 5.4) needs to show a send status per fax, but the exact values available (a simple confirmation that the send request was accepted, versus a true delivery/failure status from the transmitting fax service) depend on which service performs the actual transmission and what it reports back. Confirming that integration is engineering work, not a product decision. Does not block story creation for the report itself.
- ~~Rehab Team designation taxonomy~~ — **moved to Section 11, item 3.** After the eligible-signer question above was resolved by consulting Engineering, the PM directed that the analogous Rehab Team question be left open for Engineering to decide on its own, rather than have the PM, the AI, or the SA agent settle it here or assume the same mechanism applies. See Section 11.
- **IDT report encryption posture.** Section 9 now requires field-level encryption of all IDT report data, both auto-populated and staff-entered. Confirming whether the current `idtreports`/`idtReportVersions` schema design already meets this, or needs a design change to close a gap, is an architecture decision for the Architect. Does not block story creation for the report itself, but should be settled before the Report data model & lifecycle epic is built.

---

## 13. Known prototype limitations

The linked prototype is a design reference for Section 5's behavior. The following are artifacts of building a clickable demo, not requirements, and should not be carried into stories:

- Patient information, active medications, and weight trend are fixed demo fixtures, identical for every resident — not representative of real data variability.
- Report content and edit state are held in browser storage in the prototype; production must persist server-side per Section 9 (A6).
- The list search field in the prototype is not yet wired to the row filter; filters and sort are functional. Build the full search-plus-filter behavior specified in Section 5.1.
- Completion and signature are simulated within a single browser session (no physician queue, no second user). Build the real Send-for-signature and Sign flow specified in Section 5.4.
- "Today" is pinned to a fixed date in the prototype so seeded data stays coherent; production uses the real current date.
- The dictation control is present but intentionally inert, consistent with Section 5.5.
- Deleting a draft in the prototype only removes it for the browsing session; production deletion is a real hard delete per Section 7.
- Reopening a Complete report in the prototype mislabels its own header ("Create New IDT Report") and primary button ("Submit Changes"). Build the labels as specified in Section 5.4, not as currently shown.
- The discipline-column **Complete** state (Section 5.3) is not yet built in the prototype — every report there currently shows Pending or In-progress only. Build to Section 5.3 as written; this is a known prototype gap, not a change to the specification.
