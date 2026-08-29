> Shashi.ai · Senior Living Product Requirements · Draft v2.7 · Cleared for Dev/SA review

# Safe Discharge Plan for SNF Residents

Product requirements for the five-step discharge planning workflow used by SNF discharge planners, and the physician signature loop that runs alongside it.

| Item | Value |
|---|---|
| Status | Passed a final internal consistency pass (v2.6) and is cleared for its first Development Lead / Architect (SA) read. Every section reflects PM decisions to date; 12 open items remain, consolidated in §12 — **5 are Blocker priority** and will prevent story estimation for their area per the rule below until resolved. |
| Audience | Product Manager (review & sign-off) → Development Lead and Architect (Epic & Story creation) |
| Sources | SNF Discharge Demo prototype (behaviour of record) and the Safe Discharge Plan Flow Write-up, refined by PM decisions. Every remaining gap is raised as an open question in §12. |
| Design standards | Interface Standards v2.1 (typography, colour, spacing, interaction states, accessibility) |
| Date format | mm/dd/yy across the app. |
| Open questions | 12 items open — consolidated in §12. |
| Cross-check | This PRD was cross-checked against the exported prototype source (`SNF Discharge Demo.dc.html`, 2,014 lines): every button, input and handler was inventoried and checked against this document. `support.js` is the shared dc-runtime rendering framework with no discharge-planning business logic; `doc-page.js`, present in the same folder, is unrelated and out of scope (§13). |

> Amber callouts mark open questions. Items marked **Blocker** in §12 prevent story estimation for that area.

## 1. Scope

### 1.1 In scope

- Discharge Planning list (resident grid) and the new-plan resident picker.
- The five-step Safe Discharge Plan: Discharge Order, Home Health Referral, Discharge Planning Meeting, Discharge Medication List, Discharge Checklist. Discharge Planning Meeting attaches to the existing Care Conference feature via its API; attendee selection, Duration and Notes/agenda stay as this step's own UI inputs and are passed through to Care Conference, while scheduling storage, dial-in, invitations, recording, transcript, AI summary and completion tracking belong to Care Conference (see §5.5).
- LIC 602 A (Form 602) generation, editing, print and fax.
- Referral document management: upload, view (via the existing document-viewer component, opened in a new tab), include/exclude, delete, fax delivery tracking and retry.
- Selecting a pharmacy fax destination from the managed contact list, and medication-list fax. (Building/managing that contact list is part of the IDT Reports feature, not this PRD — see §1.3 and OQ-67.)
- Physician mobile _Pending Sign_ queue and signature submission.
- Read-only PointClickCare (PCC) resident data integration.

### 1.2 Out of scope for this release

Overall release scope not stated by the business. **OQ-01** Specific items confirmed out of scope:

- **Live pharmacy-directory search** — replaced by the managed pharmacy-contacts capability; see §5.6 and OQ-67.
- **Building the document viewer/preview component** — it already exists as a platform capability; this PRD only specifies that a document name link opens it in a new tab (§5.4).
- **Building the Agencies Directory** — it already exists with the information this workflow needs, including fax numbers; see the assumption below.
- **Building meeting scheduling storage, dial-in, invitations, recording, transcription and AI-summary infrastructure for Step 3** — provided by the existing Care Conference feature; this PRD only specifies that Step 3 attaches a Care Conference instance of a new "Discharge Planning" meeting type and stores its ID. **Building the AI-summary review/edit/approve UI** — that happens in the Care Conference UI, available to any Case Manager or Social Worker who attended the meeting, not built here. See §5.5.
- **Building a new fax UI component for Step 4's pharmacy fax** — a Fax component is already built as part of the IDT Reports feature and is reused here instead. See §5.6.
- **Building pharmacy contact management** — this belongs to the IDT Reports feature rather than this PRD; that PRD is being updated separately. See §5.6 and OQ-67.
- **The Step 5 Delivery Status block and the Referral Summary block** — both described in the Flow Write-up, neither is being built; see §5.7.
- **Building e-signature capture** (method, re-authentication, legal standard, stored artefact) — the product's existing signature module is reused instead. See §5.9.
- **Authentication, session handling and multi-facility scoping** — handled at the platform level, not by this feature. See §7.

### 1.3 Assumptions

- **Agencies Directory.** The product already has an Agencies Directory holding every home health agency record with the information this workflow needs, including fax numbers. This PRD assumes that directory as a given data source (§5.4) rather than specifying how it's built or maintained.
- **Care Conference feature.** The product already has a Care Conference feature that schedules meetings with facility-aware time zone handling, sources dial-in details from the resident's profile, sends invitations/reminders, and records/transcribes/AI-summarizes sessions. This PRD assumes that feature as a given dependency (§5.5) — extended with a new "Discharge Planning" meeting type — rather than specifying how Care Conference itself is built.
- **Fax component.** The product already has a Fax component, built for the IDT Reports feature. This PRD assumes that component is reused for Step 4's pharmacy fax (§5.6) rather than specifying a new one.
- **Pharmacy contact management.** This capability (managing the list of pharmacy contacts a planner selects from on Step 4) belongs to the IDT Reports feature, which the PM is updating separately. This PRD assumes it as a given dependency (§5.6) rather than specifying it — see OQ-67.
- **Signature module.** The product already has a signature module handling e-signature capture, re-authentication and the stored signature artefact. This PRD assumes that module is reused for physician sign-off (§5.9) rather than specifying a new one.
- **Responsive breakpoints.** Exact pixel breakpoints for the Discharge Planning grid's fold behaviour (§5.1), the minimum supported width, and tablet behaviour will be supplied by the UX designer via Figma directly to the dev team. This PRD assumes that source rather than specifying pixel values itself.

### 1.4 Templates pending from PM

The following document/print formats are needed from PM before implementation can be fully specified. Each is tracked in §12 until supplied.

| Item | Used on | What's needed | Open item |
|---|---|---|---|
| Discharge Order print format | Step 1 — "Print Discharge Order" (§5.2) | Confirmation of whether printing produces the on-screen step as-is or a separate formatted discharge order document, plus an image of the intended print format if the latter. | OQ-12 |
| Discharge Medication List download format | Step 4 — "Download List" (§5.6) | Production PDF format for the downloaded medication list. | OQ-41 |
| Physician signing document format | Physician mobile — Pending Sign detail view (§5.9) | Which resident details must appear on the rendered signing document, and the intended PDF format. | OQ-55 |

## 2. Personas

| Persona | Surface | Responsibilities in this workflow |
|---|---|---|
| Case Manager / Social Worker / Admin | Desktop web | Three named roles, all carrying identical permissions in this workflow. Starts and owns the plan; completes every step except signature; sends documents to the physician, agencies and pharmacy; runs the final checklist and completes the discharge. |
| Attending Physician | Mobile app | Receives documents in a Pending Sign queue, reviews the rendered document, adds a signature and submits. Signed status returns to the SNF workflow and unblocks the dependent step. |

The Flow Write-up also names a _care team_ as an actor on Step 3, but the prototype models care-team members as meeting attendees selected by the planner, not as system users.

The Attending Physician's mobile app and the "Staff mobile app" used by Case Manager/Social Worker/Admin for Care Conference (§5.5) and for physician-signature notifications (§5.9) are **one application**: PM confirmed the menu differs based on the logged-in user's role, not two separate builds.

No explicit delegation feature is needed: any Case Manager, Social Worker, or Admin may open and work on any in-progress discharge plan, since the three roles already share identical permissions (see §7).

## 3. PointClickCare integration

Confirmed by the business: PCC integration is via **API, read-only**. Resident information is pushed to this system through a **webhook** and can additionally be pulled through an **API query for manual refresh**. This system never writes back to PCC.

### 3.1 PCC-sourced fields and editability

Confirmed as PCC-sourced. The _Editable in app_ column is for PM to complete — it is deliberately left blank rather than guessed.

| PCC field | Used on | Editable in app? | If editable, override rule |
|---|---|---|---|
| Resident name | Grid, picker, plan header, Step 1, Form 602 §II.1, physician queue | No |  |
| Date of birth | Step 1, Form 602 §II.2, physician document header | No |  |
| Attending physician | Grid, Step 1, signature routing, Form 602 §20 | No |  |
| Room number | Grid sub-line, picker, physician document header | No |  |
| Payer | Grid, picker | No |  |
| Admission date / length of stay | Grid (Length of Stay column) | Yes — elsewhere | Editable only if the admission date is missing, but **not within this feature** — the edit happens on the **Resident Profile page**, a separate feature out of scope for this PRD. Within Discharge Planning (grid, plan header, Step 1) the value is always read-only text. Does not write back to PCC. |
| Diagnoses | Form 602 §7, §8 | Yes | Updates the resident data in this app only — does not write back to PCC. |
| Medications | Step 4 medication list, Form 602 §7a/§8a, physician medication summary | No |  |

> **Resolved.** PM confirmed both facets flagged here: Admission date / length of stay does **not** write back to PCC, and it's edited on the **Resident Profile page** — a separate feature, out of scope for this PRD — not anywhere within the Discharge Planning plan itself. The row above is updated accordingly; **OQ-71 is closed.**

MRN and resident home address are PCC-sourced, synced to the Shashi Care backend the same way as the fields in the table above. Allergies and vitals are also PCC-sourced and synced the same way. Primary caregiver and transportation are captured and stored in the Shashi Care app itself, not sourced from PCC. (Readmission risk is separately established elsewhere in this PRD as the attending physician's own assessment, captured while signing the Discharge Order — §5.9 — not PCC-sourced either.) **Still open:** source for family members and relationships, and for TB test history, has not been confirmed — "vitals" was confirmed as PCC-sourced but wasn't stated to include TB test history, so that isn't assumed. **OQ-03** (narrowed to family members/relationships and TB test history)

PCC pushes resident data to this system via webhook: **onAdmission, onReAdmission, onDischarge, onMedicationChange**. Each webhook event's payload carries only the resident's PCC patient ID; on receipt, the app makes an API call back to PCC to query for the specific information it needs for that event. The full PCC API contracts (both FHIR and REST) are documented separately — see `SNF/03_architecture/integrations/pcc/api-contracts` — so this PRD doesn't reproduce payload schemas. There is no manual refresh directly against PCC — a Refresh control in the UI instead pulls the latest data from the **Shashi Care app backend**, for use when the discharge plan's data may have gone stale; this action is expected to be effectively instant, so no loading state is required for it. When PCC-sourced data has changed since the plan was created, the app shows a banner; clicking it triggers the same Refresh action. The plan never silently re-reads or silently keeps a stale snapshot. On completion, the Refresh action shows an alert confirming success or describing the failure, so the user always gets explicit feedback.

## 4. User journey

A discharge planner works one resident at a time. The journey below is the happy path; each step's gate is the condition that marks it Completed and lights the next segment of the progress bar.

| # | Step | Primary action | Completion gate |
|---|---|---|---|
| 0 | Discharge Planning list | Click a resident row, or _Start a new Safe Discharge Plan_ → pick a resident | Opens the plan at Step 1 |
| 1 | Discharge Order | Send to Physician for Signature | Order sent. Advances to Step 2 immediately; does _not_ wait for the signature. |
| 2 | Home Health Referral | Send Documents to N Agencies | Packet sent to every selected agency; per-agency fax delivery status does not affect it. While any selected agency is unsent the button stays on Step 2 and re-labels to _Send to N New Agencies_; once none are pending it becomes _Continue to Planning Meeting_. Delivery status stays visible even once the stage is complete — a checklist message on Step 5 (e.g. "Fax status pending for N agencies") links to the expanded Referral Packet Delivery Status panel (see §5.7). |
| 3 | Discharge Planning Meeting | Schedule Meeting, then Continue once the meeting is marked completed | Meeting completed. Scheduling alone advances the flow but does not complete the stage. |
| 4 | Discharge Medication List | Send to Physician for Signature; fax to pharmacy (optional); Continue | Medication list submitted to the physician for signature. Continue does not advance past Step 4 until that submission has happened. Faxing to the pharmacy is not part of this gate at all — it's fully optional and the user may choose not to send it. |
| 5 | Discharge Checklist | Complete Discharge | A successful Complete Discharge action. Of the Step 5 fields, only **DME Status** is mandatory; every other checklist item is optional. Complete Discharge is additionally disabled until Steps 1–4 are all complete, and a confirmation dialog precedes completion, telling the user editing remains open for 1 month after the Step 1 Discharge Date (see §5.7). Success banner shown on completion. |

Running alongside: whenever Step 1 or Step 4 sends a document, an item appears in the physician's Pending Sign queue, and the physician is notified via the shared mobile app described in §5.9. Either document may be sent to the physician any number of times — for example after a correction — with no cap on resubmission; if a physician needs changes, staff coordinate with them outside the app and resend rather than the physician declining in-app (see §5.9). The physician signs and submits; the signed status returns, staff are notified again, and Step 5's Discharge Checklist updates — for the medication list, Step 4's status pill also updates.

## 5. Screen specifications

### 5.1 Discharge Planning list

Landing screen. Header greeting shows the operator name and today's date in mm/dd/yy.

| Element | Source | Behaviour & validation |
|---|---|---|
| Columns | Shashi Care backend, synced from PCC + physician assessment (risk) | Nine columns, in order: **Resident, Room, Readmission Risk, Discharge Date, Length of Stay, Payer Source, Physician, Progress, Actions.** Resident, Room and Physician are read from the Shashi Care backend, not PCC directly: PCC remains the system of record and resident records are synced into the backend whenever the entry is added or edited there. |
| Column widths | — | Every column except Progress has a **fixed width**. Progress takes whatever width remains. |
| Readmission Risk | Plan (physician assessment) | Rendered in the **same plain format as Room — no chip, no status colour.** Value is High / Moderate / Low, or blank/"Not available" when no risk has been recorded. The value is **not** facility-entered: it is the attending physician's assessment, captured while signing the Discharge Order on the mobile app (§5.9). Until the physician records one, the cell reads Not available. Updated UI for this capture is being provided separately by PM as a dev task. |
| Discharge Date column | Plan | mm/dd/yy. Default sort column, ascending. |
| Sorting | — | Six columns are sortable: Resident, Discharge Date, Length of Stay, Payer Source, Physician and Progress. One column is active at a time; clicking an inactive header sorts it ascending, clicking the active header toggles direction. The active header is tinted and shows an up/down arrow; inactive sortable headers show a muted two-way chevron. Sort keys: Resident by surname then full name; Discharge Date chronologically; Length of Stay by numeric days; Payer Source and Physician alphabetically (Physician by surname); Progress by number of completed steps. Default is Discharge Date ascending. Sorting is applied after the physician filter and does not persist across sessions. |
| Length of Stay | PCC | Days, as text. |
| Payer Source | PCC | Value list is **Medicare, Medi-Cal, Managed Care, Private Pay**. |
| Physician | PCC | Attending physician name. |
| Progress | Plan | Value is genuinely **derived** from actual step completion. Because the five steps have no fixed required sequence, the tooltip does **not** name a "next" step — it simply reads "_n_ of 5 complete". |
| Row actions | — | A single **Delete** icon (trash), right-aligned in the Actions column. There is no Edit action. The icon stops row-click propagation and opens a confirmation dialog: title "Delete this discharge plan?", body naming the resident and room, actions Cancel and Delete plan (destructive). On confirm, the discharge plan for that resident — including all stage progress — is deleted and the row leaves the grid; the resident record itself is untouched and becomes eligible for a new plan. **The action cannot be undone** and there is no restore path. Delete is available to the same roles that have create access — Case Manager, Social Worker, Admin. |
| Row click | — | Opens the Safe Discharge Plan for that resident at Step 1 and seeds resident name, physician and discharge date. |
| Physician filter | Derived | Single-select "All Physicians" + distinct physicians. When set, the control is highlighted and a _Clear_ link appears. Filtering is client-side and exact-match on name. |
| Start a new Safe Discharge Plan | — | Opens the resident picker modal: **name only, no avatar**, "Room No _x_ · payer". The picker lists **only residents who do not already have a discharge plan** — residents already on the Discharge Planning list are excluded, so the same plan cannot be started twice. Selecting a resident creates the plan and opens it at Step 1; readmission risk starts unset, so the cell in the grid and the plan header reads **Not available** until the physician records one. The new plan joins the Discharge Planning grid **when it is first saved** (Save, or any action that commits the step); until then the resident stays out of the grid and remains available in the picker, and exiting without saving abandons the plan. Search filters the picker's resident list by name or room number, live as the user types. When every resident already has a plan (nothing left to pick), the picker shows the empty-state message **"No residents pending discharge."** |

The Discharge Planning list shows residents who **have** a discharge plan in flight. Residents without a plan do not appear here; they are reachable only through the new-plan picker.

The Progress column has a minimum width; below it the **Room** value folds under the resident's name, then the **Readmission Risk** value also folds under the resident's name (after Room, separated by a dot), followed by **Length of Stay** with "days" appended. Beyond that point, horizontal scroll appears rather than collapsing further. Exact pixel breakpoints for each fold-point, the minimum supported width, and tablet behaviour are not specified in this PRD — see the Responsive breakpoints assumption in §1.3.

### 5.2 Plan shell (all steps)

| Element | Behaviour |
|---|---|
| Context bar | Back arrow (exits the plan — same discard-confirmation behaviour as Cancel, see below), title, status badge, overall progress, and a sub-line of "resident · Discharge mm/dd/yy · destination · readmission-risk chip". The risk chip repeats the physician's assessment (§5.9) and reads "Risk not available" in grey until one is recorded. Opening a resident from the list loads that resident's recorded risk into the plan header, so the chip always reflects the resident being viewed. Risk is a per-resident value stored on the plan, not a global setting. |
| Status badge | "In progress" until the discharge is completed, then "Complete". |
| Overall Progress | Percentage = completed steps ÷ 5, rounded. Step completion uses the gates in §4. |
| Stepper | Five nodes labelled Discharge Order, Home Health Referral, Discharge Planning Meeting, Discharge Medication List, Discharge Checklist. Each node's sub-label shows that stage's own status (see the "Stage status" block in each step spec); stages without a defined status set fall back to Completed (green, tick) / In Progress (pink, ring) / Pending (grey). Every node is clickable and jumps to that step with no completion requirement — steps are freely navigable. This rule is about moving between steps, not about what enables the Complete Discharge action itself, which has its own cross-step gate — see §5.7. Connector bar turns green when the previous step is complete. |
| Print Discharge Order | Sits in the sticky action bar and is rendered **only on Step 1**; printing applies to the discharge order alone, not to the plan as a whole. What exactly it produces — the on-screen step, or a formatted discharge order document — is pending from PM; see §1.4. **OQ-12** |
| Close / Cancel | Back arrow, Close and Cancel are the **same exit action**. With no unsaved changes in the session, they return to the Discharge Planning list immediately. With unsaved changes, they open a confirmation dialog — "Discard unsaved changes?", body naming the resident, actions _Keep editing_ and _Discard changes_ (destructive). Discarding rolls the plan back to its state at the start of the session (for a plan created in this session, the new plan is abandoned) and returns to the list; Keep editing dismisses the dialog and leaves the user in place. This same alert-before-losing-changes behaviour applies to navigating away from **any section** with unsaved changes, not just to closing the whole plan — see Save below. |
| Save | There is **no auto-save anywhere** in the plan. **Any edit to any field, in any section, counts as unsaved** until Save is pressed. Save is present in the sticky action bar on every step; it saves the plan's current values without advancing the step and shows "Saved h:mm AM/PM" at the left of the action bar, clearing the unsaved-changes state. The user must be alerted (per the Close/Cancel dialog above) before navigating out of a section, or off the page, while unsaved changes exist. Saving does not change any stage status. The plan's overall status is "In progress" from the moment it is created through to final completion — there is no separate Draft state. |
| Action bar | Cancel · Save · primary action, right-aligned and sticky to the bottom of the plan. "Print Discharge Order" is inserted after Save on Step 1 only. |
| Primary action | Label per step: Send to Physician for Signature · Send All Documents to Agency (dynamic, see §5.4) · Continue · Continue · Complete Discharge. |

### 5.3 Step 1 — Discharge Order

| Field | Control | Source | Req. | Validation & behaviour |
|---|---|---|---|---|
| Resident name | Read-only text | PCC | Yes * | Carried from the selected resident. Not editable. |
| Date of birth | Read-only text | PCC | Yes * | mm/dd/yy. Labelled "· from PCC". No placeholder value renders in this field. PCC always has a DOB on file for an admitted resident, so this empty state cannot occur in practice; the field stays read-only regardless. |
| Attending physician | Read-only text | PCC | Yes * | Determines who receives the signature request by default. The planner can route the signature request to a different physician: clicking "Send to Physician for Signature" opens a physician-picker pop-up, and the physician chosen there receives that submission instead of the attending physician shown here. |
| Order date | Read-only text | System (date sent for signing) | No | Labelled "· date sent". Renders empty until "Send to Physician for Signature" (§5.2) is first pressed; populates with that send's timestamp, and **updates again on every resend** — it always reflects the most recent send, not the first (resubmission is allowed any number of times; see §4). **Correction:** two earlier drafts of this row are both superseded — the original candidate answer was the plan's creation date, and a subsequent PM answer was read as the physician's *signing* date; the confirmed answer is the *sending* date, explicitly not tied to signature at all. Since it's populated by the send action itself, it structurally cannot be a precondition of that same send — stays not required, and stays excluded from whatever field set OQ-23 confirms as required before send. |
| Discharge to | Segmented buttons | User | Yes * | Options: Home, ALF, Board & Care, Other. Single-select. Changing the value **clears the destination address** and switches the address autocomplete pool (facility names for ALF / Board & Care, street addresses otherwise). Selecting ALF reveals the Form 602 card. Form 602 applies to ALF only — Board & Care is confirmed not to require it. |
| Discharge date | Text (mm/dd/yy) + calendar picker | Plan / user | Not marked | Typed entry is accepted as-is; the calendar writes back mm/dd/yy. The discharge date must not be earlier than the meeting date — cross-validated against the Step 3 meeting date (§5.5), which must equally not fall later than the discharge date. Allowed range: up to 1 month in the past, with no upper (future) constraint. |
| Destination address | Search + autocomplete | Resident record | Not marked | Prefilled from the resident record; free text is allowed. Suggestions appear on focus once ≥1 character is typed, max 4, substring match. Production uses real Google Places integration; the resolved address is **encrypted at rest in the database** and only decrypted for display in this form. A "View on Google Maps" link opens a search URL for the current value. Whether a structured place ID is also stored (vs. just the display string) is not a mandatory requirement — left to the System Architect's technical judgment during implementation. |
| Form 602 card | Card link | — | Conditional | Rendered only when Discharge to = **ALF**. Opens the LIC 602 A modal (§5.8). Board & Care does not surface it. |
| Required home services | Checkbox list | User | Not marked | Skilled Nursing (RN/LVN), Physical Therapy, Occupational Therapy, Medical Social Worker, Speech Therapy, Home Health Aide — each with a descriptive sub-line. Multi-select; whole row is the hit target. Checking **Skilled Nursing** reveals an optional free-text "location & instructions" area beneath it; unchecking hides it (entered text is retained in state). |
| Additional orders | Checkbox list | User | Not marked | "DME ordered" and "Lab orders for Home Health nurse" — **boolean only**, no DME item list or lab detail is captured. |
| Additional notes | Textarea | User | No | Free text, resizable. **5,000-character limit**, applied globally to every free-text field in the plan (notes, instructions, comments, meeting summary) — a recommended default. |

#### Stage status — Discharge Order

Allowed values, in order. There is no other state for this stage.

| Status | Set when | Stage complete? |
|---|---|---|
| In Progress | Default on open — the plan lands on this stage when the discharge is created. | No |
| Submitted | The order is submitted to the attending physician for signing ("Send to Physician for Signature"). | **Yes** — the physician signature is not mandatory, so submission alone completes the stage and unblocks Step 2. |
| Signed | The attending physician signs the order in the mobile Pending Sign queue. | Yes (already complete at Submitted) |

Rules:

- Forward-only: In Progress → Submitted → Signed. Signed is terminal for this stage.
- Stage completion is driven by **Submitted**, not by Signed. The stepper node turns green and the overall progress percentage increments on submission.
- Signature status is still tracked and must be surfaced: the **Physician Orders Signed** checkbox in Step 5 (Discharge Checklist) stays unticked while the stage is Submitted and ticks only when the stage reaches Signed. A discharge can therefore be completed with that checkbox unticked.

> OQ-23 · Step 1 submission rules "Send to Physician for Signature" currently performs no validation: it fires with no destination, no date, no services selected, and advances the step. Which fields must be present before the order can be sent? (Re-send after signature is resolved — see §4: resubmission is allowed any number of times. Order date is now confirmed **not** part of whatever set this resolves to — see the flag on that row above.)

### 5.4 Step 2 — Home Health Referral

| Field | Control | Req. | Validation & behaviour |
|---|---|---|---|
| Select agencies | Multi-select chips | Yes * | Agencies are sourced from the product's existing **Agencies Directory** (see §1.3), which already holds the information this workflow needs, including fax numbers; building that directory is out of scope here. The send action must not fire when zero agencies are selected. Whether at least one agency being selected should also block *reaching* Step 2, versus just block the send action, is not specified. |
| Referral Documents table | Table | — | Columns: Document Name (with include checkbox), File, Size, Actions. Default set, in display order: Discharge Order, Form 602 (ALF only — see Include checkbox below), Medication list, POLST Form, Face sheet, Rehab notes, Wound care report. Header count reads "_n_ uploaded" and is simply the row count. Document actions in this table (upload, include/exclude, delete) auto-save and persist immediately, unlike every other field on this and other steps, which follow the plan-wide no-auto-save rule (§5.2) — uploaded documents need a parent (the plan/referral packet) to associate with. |
| Include checkbox | Checkbox | — | Checked by default. Unchecking greys the row. The only mandatory documents are Discharge Order (always) and Form 602 (when it applies) — no other document in the packet is mandatory. **Form 602 is listed in this table at all only when the destination is ALF.** While **any selected agency has not yet received the packet** the mandatory row(s) render checked, disabled and badged "Required," so no agency can be sent a packet without them; once every selected agency has received the packet the lock lifts, and a resend or retry to an already-served agency may carry a subset (for example a corrected wound care report). |
| Document name link | Link | — | Opens the document in a new tab, using the standard document-viewer component that already exists elsewhere in the product. Building that viewer is out of scope here (§1.2); this PRD only specifies that the link opens it. |
| Row Upload | Button | — | Available on every document row except Discharge Order, Form 602, and Medication list — an exclusion rule, so it also covers any additional document type added to the packet later. For today's default document set this yields four rows (POLST Form, Face sheet, Rehab notes, Wound care report). Opens the same Upload modal as the general uploader. The uploaded file **replaces** the existing document for that row, matched by Document Name. Document names must be unique across the packet (see Upload modal — Document name below). |
| Row Edit | Icon (pencil) | — | Opens an Edit Document modal (document name + file name), surfaced as a pencil icon in the Actions column, positioned **before** the Delete icon. Disabled, together with Delete, on auto-generated system-document rows (Discharge Order, Form 602) — those documents cannot be edited. |
| Row Delete | Icon | — | Opens a destructive confirmation naming the document and file: "…will be removed from the referral packet. This can't be undone." Confirm removes the row. The Delete icon is disabled on auto-generated system-document rows (Discharge Order, Form 602) — those documents cannot be deleted. Positioned after the Edit icon in the Actions column (see Row Edit above). |
| Upload modal — File | File input | Yes * | Accepts .pdf, .jpg, .jpeg, .png. Size shown in KB. Maximum file size 5 MB; no limit on document count; malware scanning is deferred to a later phase; retention is 10 years, or the facility-level retention setting from the IDT Reports PRD if one is configured. |
| Upload modal — Document name | Text + type chips | Yes * | Auto-derived from the chosen file: extension stripped, underscores and hyphens replaced by spaces, first letter capitalised. Five type chips overwrite the name when clicked, in the same order as the Referral Documents table: Medication list, POLST Form, Face sheet, Rehab notes, Wound care report. System-generated documents (Discharge Order, Form 602) are deliberately excluded — they are produced by the workflow, not uploaded — as are Rehab status, IDT Report and Transport Report, which are out of scope. _Add Document_ is disabled until a file is chosen **and** the name is non-empty after trimming; disabled state is grey with not-allowed cursor. Document names must be unique within the packet — if the chosen/typed name matches an existing document, the new upload replaces it rather than creating a second row. Exact UX for surfacing this to the user (confirmation prompt vs. silent replace) is not specified. |
| Primary action | Button | — | Label is dynamic: "Send Documents to _n_ Agenc(y/ies)" on first send; "Send to _n_ New Agenc(y/ies)" if some were already sent; "Continue to Planning Meeting" once no selected agency is pending. Sending creates a delivery log entry per document per agency. |

#### Stage status — Home Health Referral

Allowed values, in order. There is no other state for this stage.

| Status | Set when | Stage complete? |
|---|---|---|
| Pending | Stage 1 (Discharge Order) is not yet complete. The stage is reachable but not yet actionable in sequence. | No |
| In Progress | Stage 1 reaches Submitted (or Signed). | No |
| Sent | The referral packet fax is dispatched to the selected agencies. | **Yes** — irrespective of per-document fax delivery status. |

Rules:

- Forward-only: Pending → In Progress → Sent. Sent is terminal for this stage.
- Delivery outcomes (Delivered / Failed / Sending / Queued) are tracked and surfaced in the Referral Packet Delivery Status panel, but they do **not** change the stage status and do not un-complete the stage.
- Sending to additional agencies later does not move the stage backwards; it stays Sent.

#### Referral Packet Delivery Status

Appears only after the first send. Collapsed by default; the header shows a "Faxed to _n_ agencies" pill and a summary that reads either "All documents delivered" (green) or "_n_ document(s) failed to deliver" (red).

| Element | Behaviour |
|---|---|
| Agency row | Agency name, Last Attempt timestamp (mm/dd/yy, h:mm AM/PM), rolled-up status, and an action. Roll-up: any failure → "_n_ of _m_ failed"; else any Sending/Queued → Sending; else Delivered. |
| Row action | "Retry" when the agency has failures, otherwise "Resend". Both re-send every document for that agency: status → Sending, then Delivered after the transport confirms. |
| Retry All Agencies | Visible when the panel is expanded; re-sends every document to every agency in the log. |
| Per-document statuses | Delivered (green), Failed (red), Sending (blue), Queued (grey). Queued rows have no timestamp and render "—". |

The fax provider is **Westfax**; its API reports transmission status **per document**. There is no automatic retry policy for phase 1 — deferred to a later phase; retry today is manual only, via the Retry / Retry All Agencies actions. There are no failure notifications in phase 1 — the user checks delivery status in this UI; proactive notifications are deferred to a later phase.

### 5.5 Step 3 — Discharge Planning Meeting

The meeting in this step is not a bespoke Discharge Planning feature — it is an instance of the product's existing **Care Conference** feature, invoked with a new meeting type, **"Discharge Planning."** The Discharge Planning object stores **only a reference to that Care Conference meeting's ID**; none of the meeting's own data (schedule, location/dial-in, recording, transcript, summary) is duplicated or separately persisted here, and Meeting status in this workflow is **derived from** the attached Care Conference instance rather than tracked independently.

Care Conference is responsible for: facility time zone handling, the dial-in number (sourced from the resident's profile), invitations/calendar/reminders, and recording/transcription/AI-generated summary (retained 10 years, or per the facility-level retention setting — see the IDT Reports PRD). **Duration, the Notes/agenda field, and Family members/Care team attendee selection all remain inputs in this step's own UI** — they are captured here and passed to Care Conference via its API, not moved into a separate Care Conference screen.

The AI-generated summary **requires review** before becoming part of the record — that review, editing and approval happens in the **Care Conference UI**, performable by any Case Manager or Social Worker who attended the meeting, not only the plan's owner. The primary completion trigger is the Care Conference UI on the Staff mobile app — the user marks the conference complete there, which drives this step's derived Meeting status. As a fallback for when staff didn't use that app during the conference, this step and Step 5's Discharge Checklist (§5.7) each expose a manual completion checkbox — see "Mark meeting completed" below and the mirrored item in §5.7.

| Field | Control | Req. | Validation & behaviour |
|---|---|---|---|
| Family members | Multi-select dropdown with chips | No | Options show name and relationship. Placeholder "Select family members". Selecting an option closes the dropdown. List source is not defined; see OQ-03. This selection stays in this step's own UI, not the Care Conference attendee picker — it is passed to Care Conference via its API. |
| Care team | Multi-select dropdown with chips | Yes * | Options show name and role (Social Worker, Attending Physician, DON, Physical Therapy, Case Manager). The dropdown stays open for multiple picks. The schedule grid is four columns: row 1 is Family members + Care team (spanning the remaining three columns); row 2 is Date, Time, Duration and Where. Selected members render as chips; **at most two are shown and the remainder roll into a "+n" chip** whose tooltip lists the hidden names. Chips never wrap — the row stays one line high however many are selected. Required and validated: Schedule Meeting with none selected sets a red border on the field and the error "Select at least one care team member." No specific role is mandatory within the selection — the attending physician, for example, does not have to be one of the selected members. This selection stays in this step's own UI, not the Care Conference attendee picker — it is passed to Care Conference via its API. |
| Date | Text (mm/dd/yy) + calendar picker | Yes * | Picker minimum is tomorrow. Errors: empty → "Select a meeting date."; today or earlier → "Meeting date must be in the future." Error text renders below the field in red. Cross-validated against the discharge date — the meeting date must not fall later than the discharge date (and, from the Step 1 side, the discharge date must not be earlier than the meeting date; §5.3). Scheduling itself happens through Care Conference (see integration note above); this cross-validation rule applies to whatever date the attached Care Conference meeting carries. |
| Time | Select | Yes * | 15-minute increments from 9:00 AM to 9:00 PM inclusive. Errors: empty → "Select a meeting time."; outside range → "Time must be between 09:00 and 21:00." Time zone is handled by Care Conference, using the facility's configured time zone. |
| Duration | Select | Yes * | 15 minutes / 30 minutes / 45 minutes / 1 hour. Error when empty → "Select an estimated duration." Remains a Discharge-Planning-specific input in this step's own UI — the Care Conference object holds a duration field, but the value is captured here and passed through via the Care Conference API. |
| Where | Select | No | Facility conference room / Phone call / Patient's room. When the meeting is a phone call, the dial-in number is pulled from the resident's profile, and Care Conference handles the dial-in mechanics. |
| Notes | Textarea | No | Free text (labelled "agenda" in the Flow Write-up). Remains a Discharge-Planning-specific input in this step's own UI — the Care Conference object holds a notes/agenda field, but the value is captured here and passed through via the Care Conference API. |
| Meeting status | Badge (read-only) | Yes * | "Not scheduled" (amber) → "Scheduled" (green). Derived entirely from the attached Care Conference instance's own status. The Discharge Planning object stores only the Care Conference meeting ID — no schedule, location, recording, transcript or summary data is duplicated here. |
| Schedule Meeting | Button | — | Runs date/time/duration validation; on success sets status to Scheduled and re-labels to "Reschedule". Disabled (grey, not-allowed) once the meeting is completed. Invitations, calendar entries and reminders are Care Conference's responsibility. |
| Meeting Summary | Textarea | — | Before completion: placeholder card "The meeting summary will be filled in once the discharge planning meeting is completed." After completion: auto-generated, editable, tagged "Auto-generated · editable". This content is produced by Care Conference — its recording feature (Staff mobile app) captures the meeting, and an AI layer processes the transcript into the summary; it is not built or stored as Discharge Planning data. The AI-generated summary requires review — reviewing, editing and approving it happens in the **Care Conference UI**, not here, and any Case Manager or Social Worker who attended the meeting (not only the plan's owner) may do it. |
| Transcript | Expander + download | — | Available only after completion; expands to timestamped lines and downloads as a .txt file. Before completion a placeholder states it will be available after the meeting. The transcript itself is a Care Conference artifact, not Discharge Planning data — same integration as Meeting Summary above. |
| Mark meeting completed | Checkbox | — | Manual fallback completion path, only shown/enabled once the current date is past the scheduled meeting's date and time. The primary completion path remains the Care Conference UI on the Staff mobile app (see above); this checkbox exists for when staff didn't complete the meeting there. Checking it marks this stage Completed directly and ticks the mirrored **Discharge Planning Meeting Completed** checkbox in Step 5's Discharge Checklist (§5.7), and vice versa — checking either one completes the stage. A confirmation dialog precedes it, stating that the meeting cannot be rescheduled once marked complete this way (Completed is terminal for this stage — §6.2). |

Content (recording, transcript, summary) produced by Care Conference is retained for **10 years, or the facility-level retention setting if one is configured** — see the IDT Reports PRD for that setting.

#### Stage status — Discharge Planning Meeting

Allowed values, in order. There is no other state for this stage.

| Status | Set when | Stage complete? |
|---|---|---|
| Pending | Stage 2 (Home Health Referral) is not yet complete. | No |
| In Progress | Stage 2 reaches Sent, **or** the planner makes the first edit to any meeting field (attendees, date, time, duration, where, notes) — whichever happens first. Meeting planning may run in parallel with the referral. | No |
| Scheduled | The discharge planning meeting is scheduled (date, time and duration validated). | No |
| Completed | The discharge planning meeting is marked completed — via the Care Conference completion trigger (Staff mobile app), or via the manual fallback checkbox on this step or its mirrored checkbox in Step 5 (§5.7), available once the scheduled meeting date/time has passed. | **Yes** |

Rules:

- Forward-only: Pending → In Progress → Scheduled → Completed. Completed is terminal for this stage.
- The stages are not strictly sequential: this stage can enter In Progress on its own first edit while the Home Health Referral is still open.
- Rescheduling while the status is Scheduled keeps the status at Scheduled; it does not return to In Progress.
- Scheduling alone does **not** complete the stage — the stepper node turns green and overall progress increments only at Completed.
- Meeting summary and transcript become available at Completed (see the fields above).

### 5.6 Step 4 — Discharge Medication List

| Field | Control | Req. | Validation & behaviour |
|---|---|---|---|
| Medication list document | Document row | — | Named _Med_List_<ResidentLastName>.pdf_. Shows an "Approved by Physician" pill once signed. Sourced from PCC, synchronized whenever any medication changes (consistent with the `onMedicationChange` webhook, §3.1). A fresh PDF is generated every time this section is opened. Two icons: **View** (opens the document in the standard document-viewer component, same pattern as §5.4) and **Refresh** (regenerates the PDF from the latest data in the **app backend** — this page never calls PCC APIs directly). |
| Download List | Button | — | Downloads the list. Production PDF format is pending from PM; see §1.4. **OQ-41** |
| Send to Physician for Signature | Button | — | Three states: "Send to Physician for Signature" (pink) → "Sent — awaiting signature" (amber, clock icon) → "Approved by Physician" (green, non-interactive). Adds an item to the physician queue. |
| Select pharmacy contact | Select from managed contact list | Yes * | Live pharmacy-directory search is out of scope for phase 1. In its place, the user selects from a **managed list of pharmacy contacts** (name, address, fax number). This contact-management capability is part of the IDT Reports feature, not built here — that PRD is being updated separately by the PM. **OQ-67** Picking a contact shows the selected-pharmacy card (name, address, fax) with a clear (×) control. |
| Fax to Pharmacy | Button | — | Sets the header pill to "Faxed to pharmacy" and re-labels to "Faxed to Pharmacy". **Fax to Pharmacy is not a gate of any kind** — it fires with no pharmacy contact selected and no signed list required, and the user may choose not to send it at all. Uses the existing Fax component already built for the IDT Reports feature. |

#### Stage status — Discharge Medication List

Allowed values, in order. There is no other state for this stage.

| Status | Set when | Stage complete? |
|---|---|---|
| Pending | Stage 3 (Discharge Planning Meeting) is not yet complete and no edit has been made here. | No |
| In Progress | Stage 3 reaches Completed, **or** the first edit is made on this screen (medication review, pharmacy selection) — whichever happens first. | No |
| Submitted | The discharge medication list is sent to the attending physician for signature. | **Yes** — the physician signature is not mandatory, so submission alone completes the stage. |
| Signed | The attending physician signs the medication list in the mobile Pending Sign queue. | Yes (already complete at Submitted) |

Rules:

- Forward-only: Pending → In Progress → Submitted → Signed. Signed is terminal for this stage.
- Stage completion is driven by **Submitted**, not by Signed. The stepper node turns green and overall progress increments on submission.
- Faxing to the pharmacy is not a gate of any kind. It does not affect stage completion, and the user may choose not to send it at all.
- Signature status is still tracked and surfaced: the **Discharge Medication List Signed** checkbox in Step 5 ticks only at Signed.
- This stage may enter In Progress in parallel with earlier stages, on its own first edit.

### 5.7 Step 5 — Discharge Checklist

| Field | Control | Validation & behaviour |
|---|---|---|
| Physician Orders Signed | Derived checkbox | Read-only; ticks when the Discharge Order is signed by the physician. |
| Form 602 Signed | Manual checkbox | The only manually settable item in this list. Hidden entirely when the destination is not ALF. |
| Referral Sent to Agency | Derived checkbox | Read-only; ticks once Step 2 has sent the packet. Does not account for delivery failures. When one or more agencies still show a pending/failed fax, this checklist item shows a clickable message (e.g. "Fax status pending for N agencies") that jumps to and expands the Referral Packet Delivery Status panel on Step 2 (§5.4). |
| Discharge Planning Meeting Completed | Manual checkbox (fallback) | Unlike the derived checkboxes above and below, this one is directly editable here — a fallback for when staff didn't complete the meeting through the Care Conference UI (Staff mobile app). Checking it here also marks Step 3's meeting Completed (§5.5), and vice versa; checking either one completes the stage. Same gating and confirmation as Step 3: only available once the scheduled meeting date/time has passed, and a confirmation dialog warns the meeting cannot be rescheduled once marked complete this way. |
| Discharge Medication List Signed | Derived checkbox | Read-only; ticks when the physician signs the medication list. |
| DME Status | Segmented, **mandatory** | Delivered / Pending / Loaned / Not required. Single-select, defaults to Delivered. This is the one Step 5 field that must be set before Complete Discharge succeeds — every other checklist item on this step is optional. The definitions of these values don't matter — no logic is tied to the selection, and users already understand what each option means. |
| Home Health Agency Accepted | Segmented, required | Rendered **only when more than one agency was selected** in Step 2; lists those agencies. The Case Manager or Social Worker manually marks acceptance — it does not arrive from the agency side. When only one agency was selected, it is pre-selected by default rather than left for the planner to pick. |
| Transportation Confirmed | Checkbox | Manual. No structured transport-detail fields are needed — the checkbox alone is sufficient; if detail is required, the planner records it in the free-text Additional Comments field below. |
| Caregiver Training Completed | Checkbox | Manual. |
| Follow-up Appointment Scheduled | Checkbox | Manual. The appointment itself is not captured as data. |
| Additional Comments | Textarea (optional) | Free text. Starts empty in production. |
| Complete Discharge | Primary button | A successful Complete Discharge action is itself the gate for the plan. Except for DME Status (mandatory, see above), none of the other Step 5 fields block completion. The button is additionally disabled until Steps 1–4 are all complete — a cross-step gate on top of the DME Status requirement. This is the one action in this workflow gated on every prior stage's completion (see the "freely navigable" stepper note in §5.2, which is about moving between steps, not about this gate). A confirmation dialog precedes completion — worded clearly and concisely, stating that the plan will be saved and marked complete and that the user can continue to edit it for **up to 1 month after the Step 1 Discharge Date** (§5.3) — not from the Complete Discharge action's own timestamp. On confirm, sets the plan to Complete and shows the green confirmation banner naming the resident and destination. **Editing is blocked outright once the window elapses** — no grace period or notification. Whether post-completion edits (during the 1-month window) are audit-logged the same way as pre-completion ones (§8) is still open — see §6.1, **OQ-70** (narrowed). |

The Flow Write-up also describes a _Delivery Status_ block (status, discharge date, discharge time) and a _Referral Summary_ (resident + MRN, agency + phone, discharge date/time, destination, transportation, primary caregiver) on this step. Neither is being built — both are out of scope. Confirmed by reading the code directly: the Referral Summary is **not merely unbuilt** — it is fully computed in state with fixture values matching that exact field list (resident name, a hard-coded MRN, a hard-coded agency and phone, a discharge date from a prior year, destination, a hard-coded transportation provider, and a hard-coded primary caregiver) but the computed value is never referenced anywhere in the template, so it renders nowhere; the dead `referralSummary` computation can be disregarded (see §13).

#### Stage status — Discharge Checklist

Allowed values, in order. There is no other state for this stage.

| Status | Set when | Stage complete? |
|---|---|---|
| Pending | Stages 1–4 are not all complete and no edit has been made here. | No |
| In Progress | All four previous stages are complete, **or** the first edit is made on this screen (Form 602 tick, DME status, accepting agency, a discharge-preparation checkbox) — whichever happens first. | No |
| Completed | "Complete Discharge" is confirmed. | **Yes** — this also completes the discharge plan as a whole. |

Rules:

- Forward-only: Pending → In Progress → Completed. Completed is terminal for the stage and for the plan.
- This stage may enter In Progress in parallel with earlier stages, on its own first edit.
- The derived checkboxes (Physician Orders Signed, Referral Sent, Medication List Signed) are read-only reflections of the other stages and do not by themselves move this stage's status. The **Discharge Planning Meeting Completed** checkbox is the one exception on this screen — it's manually editable and, when checked, completes Step 3's stage (§5.5) rather than merely reflecting it.

### 5.8 LIC 602 A — Physician's Report for Community Care Facilities

**See Annex A.** The full field-by-field inventory — every section, control type, prefill source and value list — lives in *Annex A · LIC 602 A Field Specification* (`prd-discharge-planning-annex-a-form602.md`, separate document, so this PRD stays readable and the annex can be reviewed by clinical staff on its own). This section covers behaviour only.

Full-screen modal opened from the Step 1 Form 602 card (ALF destinations only). Reproduces the State of California CDSS form. Header pills read "Auto-populated · editable" and, when applicable, "_n_ field(s) need input".

| Area | Behaviour |
|---|---|
| Sections | I Facility Information · II Resident/Patient Information · III Authorization for Release · IV Diagnosis/Examination · 6 TB Test · 7 Primary Diagnosis · 8 Secondary Diagnoses · 9 Cognitive checks · 10 Contagious disease · 11 Allergies · 12 Other conditions · 13 Physical Health Status · 14 Mental Condition · 15 Capacity for Self-Care · 16 Medication Management · 17 Ambulatory Status · 18 Physical Health Status · 19 Comments · 20–24 Physician. |
| Field states | Every text field is editable inline. Empty fields render on an amber background with the placeholder "Not available — enter" and are counted in the header's missing-field pill. |
| Prefill | Facility name, street, city and ZIP are parsed from the Step 1 destination address string. Resident name and physician name come from the plan. Diagnoses, medications, allergies and TB fields carry sample values in the prototype. **OQ-51** |
| Yes/No blocks | Sections 13–16 are Yes/No grids; §13 additionally captures an assistive device per row, and every row has an explanation field. No row is required. |
| Print | Invokes the browser print dialog on the whole page. |
| Fax | Reveals an inline bar with Fax number and Recipient/facility (defaulted from the destination address), Send Fax and Cancel. On send a green confirmation strip reads "Faxed to <number>". No number validation, no delivery tracking, no retry. **OQ-52** |
| Save as Draft | Shows "Draft saved h:mm AM/PM". Persistence, versioning and who may edit after signature are not specified. **OQ-53** |
| Signature | §23 Physician's Signature is a plain text field. Form 602 is **not** routed through the physician Pending Sign queue, yet Step 5 has a "Form 602 Signed" checkbox. Confirmed by inspecting the physician queue's construction directly: it is built only from the Discharge Order and Discharge Medication List sends, with no Form 602 entry point at all. Confirm the intended signature path. **OQ-54** |

### 5.9 Physician mobile — Pending Sign

| Element | Behaviour |
|---|---|
| Queue | One card per document awaiting signature: resident name, document type (Discharge Order / Discharge Medication List), sent date and time, and "Sent by". Signed items remain in the list at reduced opacity with a "Signed" pill. |
| Empty state | "Nothing waiting for your signature." |
| Detail | Renders the document: type, generation date, facility, then Patient Name, Room No, Date of Birth, Doctor, and a Current Medication Summary table (medicine, dosage, frequency). Room ("A-101") and DOB ("03/24/1978") are hard-coded literals in the template, independent of the resident being viewed. Which resident details must appear in production, and the document's intended PDF format, are pending from PM; see §1.4. **OQ-55** |
| Add Sign | Applies a signature preview to the document and re-labels to "Signed". Capture method, re-authentication and the stored signature artefact are handled by the product's existing signature module — this feature depends on it rather than building its own (see §1.3). |
| Readmission risk | A "Readmission risk" radio group appears on the **Discharge Order** signing screen only (not on the Medication List). Options: Low / Moderate / High. **None is selected by default** and the field is optional — the physician may submit without choosing, in which case the risk stays empty and the resident list shows a grey "Not available" chip. A "Clear selection" link removes a chosen value before signing. Once the document is signed the control becomes read-only. The saved value is stored against that resident's plan and drives the risk chip in both the Discharge Planning list (§5.1) and the plan header (§5.2). PM is providing updated UI for this capture as a separate dev task, hosted in this same shared mobile app (§2) under the physician's role menu. |
| Submit | Disabled (grey, not-allowed) until a signature has been added. On submit the item is marked signed, the view returns to the queue, and the SNF workflow updates. |
| Decline / query | Declining to sign is not part of this workflow in phase 1. If changes are needed, staff coordinate with the physician outside the app and resubmit — a document may be sent to the physician any number of times (see §4). |
| Tab bar | Home, Messages, Pending Sign (active, badged), Profile. Only Pending Sign is in scope. |
| Notification | A bell with an unread dot is shown. Per PM: the physician is notified when a document is sent for signature, and again once they sign it. This is the same mobile app referred to elsewhere as the "Staff mobile app" (§5.5) — one application whose menu differs by the logged-in user's role, not two separate apps (see §2). Escalation when a document goes unsigned is deferred to a later phase, not phase 1. Authentication for this mobile app is already built at the platform level and is out of scope here. |

## 6. State machines

### 6.1 Plan

| State | Entered by | Notes |
|---|---|---|
| In progress | Plan opened for a resident | Badge in the plan header; progress % updates as steps complete. |
| Complete | Complete Discharge on Step 5 | **Not fully terminal for 1 month** — the plan remains editable for **1 month after the Step 1 Discharge Date** (§5.3) — not from the Complete Discharge action's own timestamp — communicated to the user via a confirmation dialog at the Complete Discharge action (§5.7). **Becomes fully terminal once that window elapses — editing is blocked outright, with no grace period or notification.** Whether edits made during the 1-month window are audit-logged the same way as pre-completion ones (§8) is still open, **OQ-70** (narrowed). |

No Draft, On hold or Cancelled state exists — confirmed not needed.

### 6.2 Step

Each stage has its **own** allowed status set — there is no single shared triplet. The authoritative definitions are the "Stage status" tables in §5.3–§5.7; summarised:

| Stage | Allowed statuses | Status that marks the stage complete |
|---|---|---|
| 1 · Discharge Order | In Progress → Submitted → Signed | **Submitted** (signature not mandatory) |
| 2 · Home Health Referral | Pending → In Progress → Sent | **Sent** (regardless of fax delivery outcome) |
| 3 · Discharge Planning Meeting | Pending → In Progress → Scheduled → Completed | **Completed** |
| 4 · Discharge Medication List | Pending → In Progress → Submitted → Signed | **Submitted** (signature not mandatory) |
| 5 · Discharge Checklist | Pending → In Progress → Completed | **Completed** |

**Transitions.** Every transition below is triggered by an explicit action or by a preceding stage completing; there are no timed or background transitions.

| Stage | From → To | Trigger | Effect |
|---|---|---|---|
| 1 | — → In Progress | Discharge plan is created (landing stage). | Stage is editable; overall progress 0%. |
| 1 | In Progress → Submitted | "Send to Physician for Signature". | **Stage complete.** Item added to the physician's Pending Sign queue; stage 2 → In Progress. |
| 1 | Submitted → Signed | Physician signs and submits from the mobile queue. | Ticks "Physician Orders Signed" in Step 5. No gate changes. |
| 2 | Pending → In Progress | Stage 1 reaches Submitted. | Agency selection and document list become the active work. |
| 2 | In Progress → Sent | Referral packet dispatched to **all** selected agencies. | **Stage complete.** Delivery Status panel appears; ticks "Referral Sent to Agency" in Step 5. |
| 2 | Sent → Sent (no change) | Sending to newly added agencies, Retry, Resend, or any fax failure. | Delivery states update only; the stage does not move or un-complete. |
| 3 | Pending → In Progress | Stage 2 reaches Sent, **or** first edit on this screen — whichever is first. | — |
| 3 | In Progress → Scheduled | "Schedule Meeting" passes validation (care team, date, time, duration). | Button re-labels to "Reschedule"; badge turns green. Stage still **not** complete. |
| 3 | Scheduled → Scheduled (no change) | Reschedule. | New date/time saved; status held at Scheduled. |
| 3 | Scheduled → Completed | Meeting completion trigger: the Care Conference completion action (Staff mobile app), or the manual fallback checkbox on Step 3 or Step 5, available once the scheduled meeting date/time has passed (§5.5). | **Stage complete.** Summary and transcript unlock; rescheduling disabled. |
| 4 | Pending → In Progress | Stage 3 reaches Completed, **or** first edit on this screen — whichever is first. | — |
| 4 | In Progress → Submitted | Medication list sent to the physician for signature. | **Stage complete.** Item added to the Pending Sign queue. |
| 4 | Submitted → Signed | Physician signs and submits. | Ticks "Discharge Medication List Signed" in Step 5. Faxing to the pharmacy is not gated by this or any other event — it's optional and independent of stage completion. |
| 5 | Pending → In Progress | Stages 1–4 all complete, **or** first edit here (Form 602 tick, DME status, accepting agency, a preparation checkbox) — whichever is first. | — |
| 5 | In Progress → Completed | "Complete Discharge". | **Stage complete** and the plan as a whole moves to Complete; confirmation banner shown. |

No transition in the table above is reversible, and no stage has a Blocked, On hold or Cancelled state (confirmed not needed).

Cross-cutting rules:

- Stage 1 has no Pending state — it is In Progress from the moment the plan is created, because it is the landing stage.
- **In Progress is not "the currently viewed step".** Stages 3, 4 and 5 enter In Progress either when the preceding stage completes **or** on their own first edit, whichever happens first — so several stages can be In Progress at once and work can proceed in parallel. Stage 2 enters In Progress only on stage 1 completion.
- Every stage's status is forward-only, and completion is sticky: nothing reverts a completed stage.
- Signature is tracked beyond completion. Stages 1 and 4 complete at Submitted but continue to Signed; that later transition changes no gate — it only ticks the corresponding derived checkbox in Step 5.
- Navigation remains unrestricted: any stepper node may be clicked at any time, independent of status.

### 6.3 Signature (per document)

| State | Transition | Surfaces |
|---|---|---|
| Not sent | — | Step 1 / Step 4 primary button in its default state. |
| Sent — awaiting signature | Planner sends | Appears in the physician queue; Step 4 shows an amber "Sent — awaiting signature". |
| Signature applied (unsubmitted) | Physician taps Add Sign | Local to the physician's detail view; enables Submit. |
| Signed / Approved | Physician submits | Green pill on Step 4; ticks the matching Step 5 checklist item; "Signed" pill in the queue. |

### 6.4 Document delivery (per document per agency)

| State | Transitions out |
|---|---|
| Queued | → Sending (retry or transport pickup). No timestamp while queued. |
| Sending | → Delivered, or → Failed. |
| Delivered | → Sending (manual Resend). |
| Failed | → Sending (manual Retry, per document, per agency, or all). |

### 6.5 Meeting

Not scheduled → Scheduled (validated schedule action; reschedule keeps this state) → Completed (via the Care Conference completion trigger, or the manual fallback checkbox on Step 3 / Step 5 once the scheduled date/time has passed — see §5.5; unlocks summary and transcript and disables rescheduling). Cancelled and No-show states do not exist — confirmed not needed.

## 7. Permissions

Two functional roles are modelled: the planner side is three named roles (Case Manager, Social Worker, Admin) with identical permissions in this workflow (see §2), and the Attending Physician. Any of the three planner roles may open and work on any in-progress plan — no delegation mechanism is needed. The matrix below records what the prototype allows; blank cells are for PM to decide.

| Action | Case Manager / Social Worker / Admin | Physician | Other roles |
|---|---|---|---|
| Create / open a discharge plan | Yes | No |  |
| Edit plan fields (Steps 1–5) | Yes | No |  |
| Send documents for signature | Yes | No |  |
| Sign a document | No | Yes |  |
| Edit Form 602 | Yes |  |  |
| Delete a referral document | Yes | No |  |
| Delete a discharge plan | Yes (§5.1) | No |  |
| Fax to agency / pharmacy | Yes | No |  |
| Complete the discharge | Yes | No |  |
| Read meeting transcript / summary | Yes |  |  |

Authentication, session handling and multi-facility scoping are handled at the platform level and are out of scope for this feature.

## 8. Audit trail & logging

No audit behaviour is implemented in the prototype. The events below are the ones the workflow produces and are the candidate set for auditing; the retention, format and access rules must come from you.

- Plan created, opened, step advanced, plan completed.
- Field changes on Steps 1–5 and on Form 602 (old value → new value, actor, timestamp).
- Document uploaded, renamed, included/excluded, deleted, viewed, downloaded.
- Document sent for signature; signature applied; signature submitted.
- Fax sent, delivered, failed, retried — per document, per recipient.
- Meeting scheduled, rescheduled, completed; summary edited; transcript downloaded.
- PCC webhook received and manual refresh invoked.

Full audit requirements — retention period, immutability, who can read the log, whether it must be exportable for survey/audit, and whether document views (not just edits) must be recorded — are out of scope for phase 1, deferred to a later phase.

## 9. Performance & offline

No targets have been set for phase 1 — performance and offline behaviour are deferred to a later phase. Candidate areas to revisit then:

- Page-load and interaction budgets for the grid (expected resident volume) and the plan.
- Whether the physician mobile app must work offline, and what happens if a signature is submitted without connectivity.
- Behaviour when a fax or PCC call times out mid-action.
- Concurrency: two planners open on the same resident.

## 10. Analytics

Nothing is instrumented today. Candidate events follow the same list as §8, plus time-in-step and drop-off between steps. Tooling, event naming and PHI policy for event properties are deferred to a later phase — not phase 1.

## 11. HIPAA

Every screen in this workflow displays PHI: resident identity, date of birth, MRN, room, payer, diagnoses, medications, allergies, mental and physical status (Form 602), meeting transcripts and home address. PHI also leaves the system by fax to agencies and pharmacies, and by file download (medication list, transcript).

The prototype implements no safeguards on its own. Per PM, the full HIPAA safeguard set — encryption in transit and at rest, session timeout and re-authentication, minimum-necessary access by role, BAAs with the fax, transcription and mapping vendors, whether resident address data may be sent to a third-party mapping API, PHI handling in logs and analytics, breach/incident handling, and secure disposal of downloaded files — is covered at the platform layer and is out of scope for this feature.

## 12. Consolidated open questions

Owner column left blank for you to assign. Items marked **Blocker** prevent story estimation for that area.

| ID | Area | Question | Priority | Owner |
|---|---|---|---|---|
| OQ-01 | Scope | What is explicitly out of scope for this release? | Blocker |  |
| OQ-03 | Data | Source for family members and relationships, and for TB test history — the rest of this row (MRN, home address, allergies, vitals, caregiver, transportation) is resolved; see §3.1. | Blocker |  |
| OQ-12 | Step 1 | What does "Print Discharge Order" produce — the on-screen step or a formatted order document? Consolidated with the other pending-template items in §1.4. **Pending:** PM to upload an image of the intended print format. | Medium |  |
| OQ-23 | Step 1 | Required fields before the order can be sent (Order date is confirmed excluded from this set — see §5.3). | Blocker |  |
| OQ-41 | Step 4 | Production PDF format for Download List. Consolidated with the other pending-template items in §1.4. | Medium |  |
| OQ-51 | Form 602 | Which 602 fields are auto-populated from PCC vs entered manually? | Blocker |  |
| OQ-52 | Form 602 | Default fax destination, number validation, and delivery tracking for the 602. | High |  |
| OQ-53 | Form 602 | Draft persistence, versioning, and lock-after-signature. | High |  |
| OQ-54 | Form 602 | Does the 602 route through the physician signature queue, or is it wet-signed? | Blocker |  |
| OQ-55 | Physician | Which resident details must appear on the signing document? Consolidated with the other pending-template items in §1.4. **Pending:** PM to share the intended PDF document format. | Medium |  |
| OQ-67 | Step 4 / Contacts | Pharmacy contact management belongs to the IDT Reports feature, which PM is updating separately — closes once that PRD lands. | Medium |  |
| OQ-70 | Step 5 / Plan | The 1-month post-completion editability window is confirmed measured from the Step 1 Discharge Date, and editing is confirmed blocked outright once it elapses, with no grace period (see §6.1). Still open: whether edits made during that 1-month window are audit-logged the same way as pre-completion edits (§8). | Medium |  |

## 13. Known prototype data defects

These are artefacts of the demo fixtures, not requirements. Listed so they are not carried into stories as intended behaviour.

- Step 1 Date of birth shows a future date; Form 602 §II.2 and the physician document header show two different birth dates.
- The physician document header hard-codes room A-101 regardless of resident.
- The Flow Write-up's referral summary uses a discharge date from a prior year.
- Facility identity is inconsistent: the sidebar reads "Skilled Nursing · Shashi.ai", document previews read "Redwood Grove" — a fixture artifact; facility identity/scoping itself is a platform-level concern, out of scope for this feature (see §7).
- Address and pharmacy fixtures span Illinois and California in the same plan.

### 13.1 Suspected dead code (computed, never rendered)

Added by the v0.2 code cross-check. These values are built in the prototype's state/render logic but are not referenced by any element in the template — confirmed by searching the full source for each variable name. They are not currently visible in the demo at all, so nothing here should be assumed to be "working but hidden"; each is flagged so it is designed on purpose next time, not silently dropped or silently kept.

- **Referral Summary (Step 5).** A complete `referralSummary` value is computed with the exact field set the Flow Write-up describes — resident name, a fixture MRN, a fixture agency and phone number, a discharge date (from a prior year — see the defect above), destination, a fixture transportation provider, and a fixture primary caregiver — but this value is never read by the template. The Referral Summary block is out of scope, so this computation is confirmed dead code with no production use — it does not need to be built out or repurposed.
- **`pharmacyPending`.** Computed (true whenever the pharmacy fax hasn't been sent and the active role is staff) but never referenced anywhere in the template. Likely intended as a pending-state indicator for Step 4 or Step 5 that was never wired up.
- **Fax status "tabs" remnants.** `faxTabs`, `activeFaxRows`, `activeFaxName`, and a `retryAgency` action are all computed but unused. They appear to be left over from an earlier per-agency-tab design for the Referral Packet Delivery Status panel (§5.4), since replaced by the current per-agency-row table with per-row Retry/Resend and a Retry All Agencies action — which is the version actually wired to the screen.
- **Rename entry point (`onRename`).** A handler exists in code (`onRename` on each referral document row); it is now surfaced as the Row Edit pencil icon in §5.4.

### 13.2 Scope note on the source files reviewed

`doc-page.js`, present in the same prototype export folder, is a generic reusable "paged printable document" web-component scaffold. It is not `<script src>`-loaded by `SNF Discharge Demo.dc.html` and none of its exports are referenced anywhere in that file — confirmed directly. It appears to be unrelated boilerplate carried over from the design tool's component starter kit, not part of this feature, and was excluded from the cross-check on that basis. `support.js` **is** loaded (it is the runtime that executes the whole prototype), but it is the shared dc-runtime rendering/templating framework used by every Claude Design prototype — confirmed by its own header comment ("GENERATED from dc-runtime/src/*.ts — do not edit") and by inspection of its contents, which are template compilation and React-wiring code with no discharge-planning-specific logic anywhere in it.

> Next step after PM sign-off: hand to Development Lead and Architect for Epic and Story creation. Epics were deliberately not proposed here — they should follow the answers to §12.
