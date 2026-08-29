> Shashi.ai · Senior Living Product Requirements · Draft v0.1 · For PM review

# Safe Discharge Plan for SNF Residents

Product requirements for the five-step discharge planning workflow used by SNF discharge planners, and the physician signature loop that runs alongside it.

| Item | Value |
|---|---|
| Status | Draft for Product Management review |
| Audience | Product Manager (review & sign-off) → Development Lead and Architect (Epic & Story creation) |
| Sources | SNF Discharge Demo prototype (behaviour of record) and the Safe Discharge Plan Flow Write-up. Nothing in this document was inferred beyond those two sources; every gap is raised as an open question. |
| Design standards | Interface Standards v2.1 (typography, colour, spacing, interaction states, accessibility) |
| Open questions | 65 items, marked inline as **OQ-nn** and consolidated in §12 |

> How to read this document Every field table states the behaviour the prototype actually implements. Where the prototype is silent, the cell reads **Not specified** and carries an OQ reference — those are decisions for you, not defaults for engineering to choose. Amber callouts are questions that block estimation.

## 1. Scope

### 1.1 In scope

- Discharge Planning list (resident grid) and the new-plan resident picker.
- The five-step Safe Discharge Plan: Discharge Order, Home Health Referral, Discharge Planning Meeting, Discharge Medication List, Discharge Checklist.
- LIC 602 A (Form 602) generation, editing, print and fax.
- Referral document management: upload, view, include/exclude, delete, fax delivery tracking and retry.
- Pharmacy search and medication-list fax.
- Physician mobile _Pending Sign_ queue and signature submission.
- Read-only PointClickCare (PCC) resident data integration.

### 1.2 Out of scope for this release

Not stated by the business. **OQ-01**

## 2. Personas

| Persona | Surface | Responsibilities in this workflow |
|---|---|---|
| SNF Admin / Discharge Planner | Desktop web | Starts and owns the plan; completes every step except signature; sends documents to the physician, agencies and pharmacy; runs the final checklist and completes the discharge. |
| Attending Physician | Mobile app | Receives documents in a Pending Sign queue, reviews the rendered document, adds a signature and submits. Signed status returns to the SNF workflow and unblocks the dependent step. |

The Flow Write-up also names a _care team_ as an actor on Step 3, but the prototype models care-team members as meeting attendees selected by the Admin, not as system users.

> OQ-02 · Roles Are Nurse / IDT / Social Worker separate authenticated roles in this release, or does everything non-physician run under one Admin role? Is delegation (one planner picking up another's plan) required?

## 3. PointClickCare integration

Confirmed by the business: PCC integration is via **API, read-only**. Resident information is pushed to this system through a **webhook** and can additionally be pulled through an **API query for manual refresh**. This system never writes back to PCC.

### 3.1 PCC-sourced fields and editability

Confirmed as PCC-sourced. The _Editable in app_ column is for PM to complete — it is deliberately left blank rather than guessed.

| PCC field | Used on | Editable in app? | If editable, override rule |
|---|---|---|---|
| Resident name | Grid, picker, plan header, Step 1, Form 602 §II.1, physician queue |  |  |
| Date of birth | Step 1, Form 602 §II.2, physician document header |  |  |
| Attending physician | Grid, Step 1, signature routing, Form 602 §20 |  |  |
| Room number | Grid sub-line, picker, physician document header |  |  |
| Payer | Grid, picker |  |  |
| Admission date / length of stay | Grid (Length of Stay column) |  |  |
| Diagnoses | Form 602 §7, §8 |  |  |
| Medications | Step 4 medication list, Form 602 §7a/§8a, physician medication summary |  |  |

Fields shown in the UI that are _not_ on the confirmed PCC list and therefore need a source decision: MRN, resident home address, family members and relationships, readmission risk, allergies, TB test history, height/weight/blood pressure, primary caregiver, transportation. **OQ-03**

> OQ-04 to OQ-06 · Integration mechanics

- **OQ-04** Which webhook events fire (admission, update, discharge, medication change) and what does each payload contain?
- **OQ-05** Where does the user trigger a manual refresh, and what does the UI show while it runs and when it fails?
- **OQ-06** If a PCC value changes after a plan is started, does the plan re-read it, keep the snapshot, or flag a conflict?

## 4. User journey

A discharge planner works one resident at a time. The journey below is the happy path implemented in the prototype; each step's gate is the condition that marks it Completed and lights the next segment of the progress bar.

| # | Step | Primary action | Completion gate (implemented) |
|---|---|---|---|
| 0 | Discharge Planning list | Click a resident row, or _Start a new Safe Discharge Plan_ → pick a resident | Opens the plan at Step 1 |
| 1 | Discharge Order | Send to Physician for Signature | Order sent. Advances to Step 2 immediately; does _not_ wait for the signature. |
| 2 | Home Health Referral | Send Documents to N Agencies | Packet sent to every selected agency. While any selected agency is unsent the button stays on Step 2 and re-labels to _Send to N New Agencies_; once none are pending it becomes _Continue to Planning Meeting_. |
| 3 | Discharge Planning Meeting | Schedule Meeting, mark the meeting completed, then Continue | **Meeting completed.** Scheduling alone advances the flow but does not complete the stage. Continue is blocked until the meeting is scheduled and re-runs validation on the schedule form. |
| 4 | Discharge Medication List | Send to Physician for Signature; fax to pharmacy; Continue | **Medication list submitted to the physician** (signature not required). Note: Continue is not gated on either. **OQ-26** |
| 5 | Discharge Checklist | Complete Discharge | Discharge marked complete; success banner shown. No checklist gating. **OQ-27** |

Running alongside: whenever Step 1 or Step 4 sends a document, an item appears in the physician's Pending Sign queue. The physician signs and submits; the signed status returns and updates Step 5's Documentation Checklist and, for the medication list, Step 4's status pill.

## 5. Screen specifications

### 5.1 Discharge Planning list

Landing screen. Header greeting shows the operator name and today's date in mm/dd/yyyy.

| Element | Source | Behaviour & validation |
|---|---|---|
| Resident / Risk column | Shashi Care backend, synced from PCC (name, room) + physician assessment (risk) | Name on line 1 (regular weight); room number and a readmission-risk chip on line 2. Chip values High (red), Moderate (amber), Low (green), or a grey "Not available" chip when no risk has been recorded. The value is **not** facility-entered: it is the attending physician's assessment, captured while signing the Discharge Order on the mobile app (§5.9). Until the physician records one, the chip reads Not available. Name and room are read from the Shashi Care backend, not PCC directly: PCC remains the system of record and resident records are synced into the backend whenever the entry is added or edited there. |
| Discharge Date column | Plan | mm/dd/yyyy. Default sort column, ascending. |
| Sorting | — | Six columns are sortable: Resident / Risk, Discharge Date, Length of Stay, Payer Source, Physician and Progress. One column is active at a time; clicking an inactive header sorts it ascending, clicking the active header toggles direction. The active header is tinted and shows an up/down arrow; inactive sortable headers show a muted two-way chevron. Sort keys: Resident by surname then full name; Discharge Date chronologically; Length of Stay by numeric days; Payer Source and Physician alphabetically (Physician by surname); Progress by number of completed steps. Default is Discharge Date ascending. Sorting is applied after the physician filter and does not persist across sessions. |
| Length of Stay | PCC | Days, as text. Hidden below 1280px and folded into the resident sub-line. |
| Payer | PCC | Medicare / Medicaid / Private Insurance / Self-Pay. Hidden below 1280px and folded into the sub-line. |
| Physician | PCC | Attending physician name. |
| Progress | Plan | Five segments plus "_n_ of 5". Tooltip reads "Not started", "_n_ of 5 complete · next: <step name>", or "All 5 steps complete". In the prototype the value is fixture data, not derived. **OQ-08** |
| Row actions | — | A single **Delete** icon (trash), right-aligned. There is no Edit action. The icon stops row-click propagation and opens a confirmation dialog: title "Delete this discharge plan?", body naming the resident and room, actions Cancel and Delete plan (destructive). On confirm, the discharge plan for that resident — including all stage progress — is deleted and the row leaves the grid; the resident record itself is untouched and becomes eligible for a new plan. **The action cannot be undone** and there is no restore path. Who may delete (role/permission) is not specified. **OQ-09** |
| Row click | — | Opens the Safe Discharge Plan for that resident at Step 1 and seeds resident name, physician and discharge date. |
| Physician filter | Derived | Single-select "All Physicians" + distinct physicians. When set, the control is highlighted and a _Clear_ link appears. Filtering is client-side and exact-match on name. |
| Start a new Safe Discharge Plan | — | Opens the resident picker modal: avatar with initials, name, "Room No _x_ · payer". The picker lists **only residents who do not already have a discharge plan** — residents already on the Discharge Planning list are excluded, so the same plan cannot be started twice. Selecting a resident creates the plan and opens it at Step 1; readmission risk starts unset, so the chip in the grid and the plan header reads **Not available** until the physician records one. The new plan joins the Discharge Planning grid **when it is first saved** (Save, or any action that commits the step); until then the resident stays out of the grid and remains available in the picker, and exiting without saving abandons the plan. A search field is shown but is not wired; no empty state is defined for when every resident already has a plan (**OQ-10**). |

The Discharge Planning list shows residents who **have** a discharge plan in flight. Residents without a plan do not appear here; they are reachable only through the new-plan picker.

**Responsive:** at ≥1280px the grid shows 7 columns; below 1280px Length of Stay and Payer collapse into the resident sub-line and the grid renders 5 columns. Behaviour below 1024px and on tablet is not specified. **OQ-11**

### 5.2 Plan shell (all steps)

| Element | Behaviour |
|---|---|
| Context bar | Back arrow (exits the plan — same discard-confirmation behaviour as Cancel, see below), title, status badge, overall progress, and a sub-line of "resident · Discharge mm/dd/yyyy · destination · readmission-risk chip". The risk chip repeats the physician's assessment (§5.9) and reads "Risk not available" in grey until one is recorded. Opening a resident from the list loads that resident's recorded risk into the plan header, so the chip always reflects the resident being viewed. Risk is a per-resident value stored on the plan, not a global setting. |
| Status badge | "In progress" until the discharge is completed, then "Complete". |
| Overall Progress | Percentage = completed steps ÷ 5, rounded. Step completion uses the gates in §4. |
| Stepper | Five nodes labelled Discharge Order, Home Health Referral, Discharge Planning Meeting, Discharge Medication List, Discharge Checklist. Each node's sub-label shows that stage's own status (see the "Stage status" block in each step spec); stages without a defined status set fall back to Completed (green, tick) / In Progress (pink, ring) / Pending (grey). Every node is clickable and jumps to that step with no completion requirement — steps are freely navigable. Connector bar turns green when the previous step is complete. |
| Print Discharge Order | Sits in the sticky action bar and is rendered **only on Step 1**; printing applies to the discharge order alone, not to the plan as a whole. What exactly it produces — the on-screen step, or a formatted discharge order document — is not specified. **OQ-12** |
| Close / Cancel | Back arrow, Close and Cancel are the **same exit action**. With no unsaved changes in the session, they return to the Discharge Planning list immediately. With unsaved changes, they open a confirmation dialog — "Discard unsaved changes?", body naming the resident, actions _Keep editing_ and _Discard changes_ (destructive). Discarding rolls the plan back to its state at the start of the session (for a plan created in this session, the new plan is abandoned) and returns to the list; Keep editing dismisses the dialog and leaves the user in place. What counts as "unsaved" depends on the save model still to be settled. **OQ-14** |
| Save | Present in the sticky action bar on every step. Saves the plan's current values without advancing the step and shows "Saved h:mm AM/PM" at the left of the action bar; the save also clears the unsaved-changes state, so exiting afterwards returns to the list with no prompt. Saving does not change any stage status. Step 2 separately claims "All changes auto-saved". Save vs auto-save vs draft state must be reconciled. **OQ-14** |
| Action bar | Cancel · Save · primary action, right-aligned and sticky to the bottom of the plan. "Print Discharge Order" is inserted after Save on Step 1 only. |
| Primary action | Label per step: Send to Physician for Signature · Send All Documents to Agency (dynamic, see §5.4) · Continue · Continue · Complete Discharge. |

### 5.3 Step 1 — Discharge Order

| Field | Control | Source | Req. | Validation & behaviour |
|---|---|---|---|---|
| Resident name | Read-only text | PCC | Yes * | Carried from the selected resident. Not editable in the prototype. |
| Date of birth | Read-only text | PCC | Yes * | mm/dd/yyyy. Labelled "· from PCC". The prototype renders a fixed placeholder value that is not a plausible DOB. **OQ-15** |
| Attending physician | Read-only text | PCC | Yes * | Determines who receives the signature request. Whether the planner may route to a different physician is not specified. **OQ-16** |
| Order date | Read-only text | System (date created) | Yes * | Labelled "· date created". Whether this is plan creation date or the physician's order date is ambiguous. **OQ-17** |
| Discharge to | Segmented buttons | User | Yes * | Options: Home, ALF, Board & Care, Other. Single-select. Changing the value **clears the destination address** and switches the address autocomplete pool (facility names for ALF / Board & Care, street addresses otherwise). Selecting ALF reveals the Form 602 card. |
| Discharge date | Text (mm/dd/yyyy) + calendar picker | Plan / user | Not marked | Typed entry is accepted as-is; the calendar writes back mm/dd/yyyy. No format, range or past-date validation is implemented, and it is not cross-checked against the meeting date. **OQ-18** |
| Destination address | Search + autocomplete | Resident record | Not marked | Prefilled from the resident record; free text is allowed. Suggestions appear on focus once ≥1 character is typed, max 4, substring match, and are labelled "powered by Google Maps". A "View on Google Maps" link opens a search URL for the current value. In the prototype the place data is mocked. **OQ-19** |
| Form 602 card | Card link | — | Conditional | Rendered only when Discharge to = **ALF**. Opens the LIC 602 A modal (§5.8). Board & Care does not surface it. **OQ-20** |
| Required home services | Checkbox list | User | Not marked | Skilled Nursing (RN/LVN), Physical Therapy, Occupational Therapy, Medical Social Worker, Speech Therapy, Home Health Aide — each with a descriptive sub-line. Multi-select; whole row is the hit target. Checking **Skilled Nursing** reveals an optional free-text "location & instructions" area beneath it; unchecking hides it (entered text is retained in state). |
| Additional orders | Checkbox list | User | Not marked | "DME ordered" and "Lab orders for Home Health nurse". Boolean only — no DME item list or lab detail is captured. The Flow Write-up flagged this as undecided. **OQ-21** |
| Additional notes | Textarea | User | No | Free text, resizable, no length limit. **OQ-22** |

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

> OQ-23 · Step 1 submission rules "Send to Physician for Signature" currently performs no validation: it fires with no destination, no date, no services selected, and advances the step. Which fields must be present before the order can be sent, and what happens if the resident's plan is re-sent after a signature already exists?

### 5.4 Step 2 — Home Health Referral

| Field | Control | Req. | Validation & behaviour |
|---|---|---|---|
| Select agencies | Multi-select chips | Yes * | Four fixed agencies in the prototype. Multi-select with tick indicator. Not enforced — the send action fires with zero agencies selected. Agency directory source, coverage rules and fax numbers not specified. **OQ-24** |
| Referral Documents table | Table | — | Columns: Document Name (with include checkbox), File, Size, Actions. Default set: Discharge Order, Form 602, POLST Form, Face sheet, Rehab notes, Medication list, Wound care report. Header count reads "_n_ uploaded" and is simply the row count. |
| Include checkbox | Checkbox | — | Checked by default. Unchecking greys the row. **Mandatory documents are locked per recipient:** the **Discharge Order** always, and **Form 602** whenever it applies (Discharge to = ALF). While **any selected agency has not yet received the packet** — on the first send, and again whenever an agency is added later — both render checked, disabled and badged "Required", so no agency can be sent a packet without them. Once every selected agency has received the packet the lock lifts, and a resend or retry to an already-served agency may carry a subset (for example a corrected wound care report). The include state is not otherwise wired to the fax payload, and there is no other "required document missing" rule (**OQ-25**). |
| Document name link | Link | — | Opens a read-only document viewer modal showing name, file, size and a placeholder page render. |
| Row Upload | Button | — | Shown only on Wound care report, Rehab notes, POLST Form and Face sheet. Opens the same Upload modal as the general uploader; it does not replace the row's file. **OQ-26** |
| Row Delete | Icon | — | Opens a destructive confirmation naming the document and file: "…will be removed from the referral packet. This can't be undone." Confirm removes the row. Auto-generated documents can be deleted with no restriction. **OQ-27** |
| Rename document | Modal | — | An Edit Document modal (document name + file name) is implemented but currently has **no entry point** in the UI. Confirm whether renaming is required and where it lives. **OQ-28** |
| Upload modal — File | File input | Yes * | Accepts .pdf, .jpg, .jpeg, .png. Size shown in KB. No max size, no page/count limit, no malware scan specified. **OQ-29** |
| Upload modal — Document name | Text + type chips | Yes * | Auto-derived from the chosen file: extension stripped, underscores and hyphens replaced by spaces, first letter capitalised. Five type chips overwrite the name when clicked: POLST Form, Face sheet, Rehab notes, Medication list, Wound care report. System-generated documents (Discharge Order, Form 602) are deliberately excluded — they are produced by the workflow, not uploaded — as are Rehab status, IDT Report and Transport Report, which are out of scope. _Add Document_ is disabled until a file is chosen **and** the name is non-empty after trimming; disabled state is grey with not-allowed cursor. Duplicate names are permitted. |
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
| Agency row | Agency name, Last Attempt timestamp (mm/dd/yyyy, h:mm AM/PM), rolled-up status, and an action. Roll-up: any failure → "_n_ of _m_ failed"; else any Sending/Queued → Sending; else Delivered. |
| Row action | "Retry" when the agency has failures, otherwise "Resend". Both re-send every document for that agency: status → Sending, then Delivered after the transport confirms. |
| Retry All Agencies | Visible when the panel is expanded; re-sends every document to every agency in the log. |
| Per-document statuses | Delivered (green), Failed (red), Sending (blue), Queued (grey). Queued rows have no timestamp and render "—". |

> OQ-30 to OQ-32 · Fax transport

- **OQ-30** Which fax provider, and does it report per-document or per-transmission status? The prototype's failures are simulated.
- **OQ-31** Automatic retry policy: attempts, back-off, and when a failure escalates to a person.
- **OQ-32** Who is notified on failure, through which channel, and is there an SLA?

### 5.5 Step 3 — Discharge Planning Meeting

| Field | Control | Req. | Validation & behaviour |
|---|---|---|---|
| Family members | Multi-select dropdown with chips | No | Options show name and relationship. Placeholder "Select family members". Selecting an option closes the dropdown. List source is not defined; see OQ-03. |
| Care team | Multi-select dropdown with chips | Yes * | Options show name and role (Social Worker, Attending Physician, DON, Physical Therapy, Case Manager). The dropdown stays open for multiple picks. The schedule grid is four columns: row 1 is Family members + Care team (spanning the remaining three columns); row 2 is Date, Time, Duration and Where. Selected members render as chips; **at most two are shown and the remainder roll into a "+n" chip** whose tooltip lists the hidden names. Chips never wrap — the row stays one line high however many are selected. Required and validated: Schedule Meeting with none selected sets a red border on the field and the error "Select at least one care team member." Whether specific roles are mandatory (for example the attending physician) is still open. **OQ-33** |
| Date | Text (mm/dd/yyyy) + calendar picker | Yes * | Picker minimum is tomorrow. Errors: empty → "Select a meeting date."; today or earlier → "Meeting date must be in the future." Error text renders below the field in red. Not cross-validated against the discharge date. **OQ-34** |
| Time | Select | Yes * | 15-minute increments from 9:00 AM to 9:00 PM inclusive. Errors: empty → "Select a meeting time."; outside range → "Time must be between 09:00 and 21:00." No time zone is captured or displayed. **OQ-35** |
| Duration | Select | Yes * | 15 minutes / 30 minutes / 45 minutes / 1 hour. Error when empty → "Select an estimated duration." |
| Where | Select | No | Facility conference room / Phone call / Patient's room. If "Phone call" is chosen, no dial-in details are captured. **OQ-36** |
| Notes | Textarea | No | Free text (labelled "agenda" in the Flow Write-up). |
| Meeting status | Badge (read-only) | Yes * | "Not scheduled" (amber) → "Scheduled" (green). Derived from the schedule action, not directly settable. |
| Schedule Meeting | Button | — | Runs date/time/duration validation; on success sets status to Scheduled and re-labels to "Reschedule". Disabled (grey, not-allowed) once the meeting is completed. No invitations, calendar entries or reminders are produced. **OQ-37** |
| Meeting Summary | Textarea | — | Before completion: placeholder card "The meeting summary will be filled in once the discharge planning meeting is completed." After completion: auto-generated, editable, tagged "Auto-generated · editable". The prototype fills it with fixed sample content. |
| Transcript | Expander + download | — | Available only after completion; expands to timestamped lines and downloads as a .txt file. Before completion a placeholder states it will be available after the meeting. |
| Mark meeting completed | Button | — | Labelled "Demo:" in the prototype — a stand-in for the real completion trigger. **OQ-38** |

> OQ-39 · Recording, transcription and AI summary The Flow Write-up raised this and it is still open. Is meeting audio actually captured? If so: capture device, consent capture from family, transcription vendor, where audio and transcript are stored and for how long, who may read them, and whether the AI summary must be reviewed before it becomes part of the record.

#### Stage status — Discharge Planning Meeting

Allowed values, in order. There is no other state for this stage.

| Status | Set when | Stage complete? |
|---|---|---|
| Pending | Stage 2 (Home Health Referral) is not yet complete. | No |
| In Progress | Stage 2 reaches Sent, **or** the planner makes the first edit to any meeting field (attendees, date, time, duration, where, notes) — whichever happens first. Meeting planning may run in parallel with the referral. | No |
| Scheduled | The discharge planning meeting is scheduled (date, time and duration validated). | No |
| Completed | The discharge planning meeting is marked completed. | **Yes** |

Rules:

- Forward-only: Pending → In Progress → Scheduled → Completed. Completed is terminal for this stage.
- The stages are not strictly sequential: this stage can enter In Progress on its own first edit while the Home Health Referral is still open.
- Rescheduling while the status is Scheduled keeps the status at Scheduled; it does not return to In Progress.
- Scheduling alone does **not** complete the stage — the stepper node turns green and overall progress increments only at Completed.
- Meeting summary and transcript become available at Completed (see the fields above).

### 5.6 Step 4 — Discharge Medication List

| Field | Control | Req. | Validation & behaviour |
|---|---|---|---|
| Medication list document | Document row | — | Named _Med_List_<ResidentLastName>.pdf_. View and Edit icons are present but inert. Shows an "Approved by Physician" pill once signed. Generation source and edit rights not specified. **OQ-40** |
| Download List | Button | — | Downloads the list. The prototype emits a plain-text file with a "Generated mm/dd/yyyy" header; the production format (PDF) needs confirming. **OQ-41** |
| Send to Physician for Signature | Button | — | Three states: "Send to Physician for Signature" (pink) → "Sent — awaiting signature" (amber, clock icon) → "Approved by Physician" (green, non-interactive). Adds an item to the physician queue. |
| Search pharmacy | Search + autocomplete | Yes * | Suggestions appear on focus with ≥1 character; substring match over name, address and fax; max 4 results. Picking one shows a selected-pharmacy card (name, address, fax) with a clear (×) control. No "no results" empty state is rendered in this control. **Source:** the pharmacy list is not held in the product. It must be retrieved from a public pharmacy directory API, queried live as the user types in the field (type-ahead against the API rather than a local list), and the chosen record's name, address and fax number are stored on the plan. The prototype uses a hard-coded pool as a stand-in. Provider, query parameters, rate limits, minimum characters before the first call, debounce interval, caching and the no-results / API-unavailable behaviour are still to be decided (**OQ-42**). |
| Fax to Pharmacy | Button | — | Sets the header pill to "Faxed to pharmacy" and re-labels to "Faxed to Pharmacy". **The rendered button is not gated** — it fires with no pharmacy selected and no signed list, although a gating rule (pharmacy selected _and_ medication list signed, with the labels "Select a pharmacy" / "Send Fax (needs signed list)") exists unused in the code. Confirm the intended rule. **OQ-43** |
| Not required — orders handled through another system | Checkbox | No | Opt-out of the pharmacy fax. Checking it **clears any selected pharmacy and the search text, and disables the pharmacy controls**: the search field greys out with the placeholder "Not required — handled through another system", suggestions no longer open, and the fax button greys to "Pharmacy fax not required". Unchecking re-enables the controls, but the previous selection is not restored. Its effect on step completion is unchanged — completion is driven by submission to the physician. **OQ-44** |
| One-month medication ordered | Yes/No toggle | — | **Specified in the Flow Write-up and used by the Step 4 completion gate, but not rendered on the screen** — it defaults to Yes. The write-up also calls for a "date ordered". Confirm whether this field and its date belong on Step 4. **OQ-45** |

#### Stage status — Discharge Medication List

Allowed values, in order. There is no other state for this stage.

| Status | Set when | Stage complete? |
|---|---|---|
| Pending | Stage 3 (Discharge Planning Meeting) is not yet complete and no edit has been made here. | No |
| In Progress | Stage 3 reaches Completed, **or** the first edit is made on this screen (medication review, pharmacy selection, "pharmacy not required") — whichever happens first. | No |
| Submitted | The discharge medication list is sent to the attending physician for signature. | **Yes** — the physician signature is not mandatory, so submission alone completes the stage. |
| Signed | The attending physician signs the medication list in the mobile Pending Sign queue. | Yes (already complete at Submitted) |

Rules:

- Forward-only: Pending → In Progress → Submitted → Signed. Signed is terminal for this stage.
- Stage completion is driven by **Submitted**, not by Signed. The stepper node turns green and overall progress increments on submission.
- Signature status is still tracked and surfaced: the **Discharge Medication List Signed** checkbox in Step 5 ticks only at Signed. Faxing to the pharmacy also remains gated on a signed list.
- This stage may enter In Progress in parallel with earlier stages, on its own first edit.

### 5.7 Step 5 — Discharge Checklist

| Field | Control | Validation & behaviour |
|---|---|---|
| Physician Orders Signed | Derived checkbox | Read-only; ticks when the Discharge Order is signed by the physician. |
| Form 602 Signed | Manual checkbox | The only manually settable item in this list. It is shown regardless of destination, even when Form 602 does not apply. **OQ-20** |
| Referral Sent to Agency | Derived checkbox | Read-only; ticks once Step 2 has sent the packet. Does not account for delivery failures. **OQ-31** |
| Discharge Medication List Signed | Derived checkbox | Read-only; ticks when the physician signs the medication list. |
| DME Status | Segmented, required | Delivered / Pending / Loaned / Not required. Single-select, defaults to Delivered. Not validated on completion. Definition of "Loaned" and the authority for these values not specified. **OQ-46** |
| Home Health Agency Accepted | Segmented, required | Rendered **only when more than one agency was selected** in Step 2; lists those agencies. Captures which agency accepted the referral. Who records acceptance, and whether it can arrive from the agency side, is not specified. **OQ-47** |
| Transportation Confirmed | Checkbox | Manual. No transport details are captured anywhere in the flow, although the Flow Write-up's referral summary shows a transport provider. **OQ-48** |
| Caregiver Training Completed | Checkbox | Manual. |
| Follow-up Appointment Scheduled | Checkbox | Manual. The appointment itself is not captured as data. |
| Additional Comments | Textarea (optional) | Free text. The prototype ships with pre-filled sample copy that must not become a production default. **OQ-49** |
| Complete Discharge | Primary button | Sets the plan to Complete and shows a green confirmation banner naming the resident and destination. No confirmation dialog, no checklist gating, and no defined post-completion editability. **OQ-27** |

The Flow Write-up also describes a _Delivery Status_ block (status, discharge date, discharge time) and a _Referral Summary_ (resident + MRN, agency + phone, discharge date/time, destination, transportation, primary caregiver) on this step. Neither is rendered in the current prototype, and the underlying values are fixture data. Confirm whether both are in scope. **OQ-50**

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
- The derived checkboxes (Physician Orders Signed, Referral Sent, Medication List Signed) are read-only reflections of the other stages and do not by themselves move this stage's status.

### 5.8 LIC 602 A — Physician's Report for Community Care Facilities

**See Annex A.** The full field-by-field inventory — every section, control type, prefill source and value list — lives in *Annex A · LIC 602 A Field Specification* (separate document, so this PRD stays readable and the annex can be reviewed by clinical staff on its own). This section covers behaviour only.

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
| Signature | §23 Physician's Signature is a plain text field. Form 602 is **not** routed through the physician Pending Sign queue, yet Step 5 has a "Form 602 Signed" checkbox. Confirm the intended signature path. **OQ-54** |

### 5.9 Physician mobile — Pending Sign

| Element | Behaviour |
|---|---|
| Queue | One card per document awaiting signature: resident name, document type (Discharge Order / Discharge Medication List), sent date and time, and "Sent by". Signed items remain in the list at reduced opacity with a "Signed" pill. |
| Empty state | "Nothing waiting for your signature." |
| Detail | Renders the document: type, generation date, facility, then Patient Name, Room No, Date of Birth, Doctor, and a Current Medication Summary table (medicine, dosage, frequency). Room and DOB are hard-coded in the prototype. **OQ-55** |
| Add Sign | Applies a signature preview to the document and re-labels to "Signed". No drawn-signature capture, PIN or re-authentication. **OQ-56** |
| Readmission risk | A "Readmission risk" radio group appears on the **Discharge Order** signing screen only (not on the Medication List). Options: Low / Moderate / High. **None is selected by default** and the field is optional — the physician may submit without choosing, in which case the risk stays empty and the resident list shows a grey "Not available" chip. A "Clear selection" link removes a chosen value before signing. Once the document is signed the control becomes read-only. The saved value is stored against that resident's plan and drives the risk chip in both the Discharge Planning list (§5.1) and the plan header (§5.2). |
| Submit | Disabled (grey, not-allowed) until a signature has been added. On submit the item is marked signed, the view returns to the queue, and the SNF workflow updates. |
| Decline / query | No path exists for a physician to reject a document or ask a question. **OQ-57** |
| Tab bar | Home, Messages, Pending Sign (active, badged), Profile. Only Pending Sign is in scope. |
| Notification | A bell with an unread dot is shown; push/SMS/email notification behaviour is not specified. **OQ-58** |

## 6. State machines

### 6.1 Plan

| State | Entered by | Notes |
|---|---|---|
| In progress | Plan opened for a resident | Badge in the plan header; progress % updates as steps complete. |
| Complete | Complete Discharge on Step 5 | Terminal in the prototype. Re-opening and editing after completion is undefined. **OQ-27** |

No Draft, On hold or Cancelled state exists. **OQ-59**

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
| 3 | Scheduled → Completed | Meeting completion trigger (**OQ-38**). | **Stage complete.** Summary and transcript unlock; rescheduling disabled. |
| 4 | Pending → In Progress | Stage 3 reaches Completed, **or** first edit on this screen — whichever is first. | — |
| 4 | In Progress → Submitted | Medication list sent to the physician for signature. | **Stage complete.** Item added to the Pending Sign queue. |
| 4 | Submitted → Signed | Physician signs and submits. | Ticks "Discharge Medication List Signed" in Step 5; unblocks faxing the list to the pharmacy. |
| 5 | Pending → In Progress | Stages 1–4 all complete, **or** first edit here (Form 602 tick, DME status, accepting agency, a preparation checkbox) — whichever is first. | — |
| 5 | In Progress → Completed | "Complete Discharge". | **Stage complete** and the plan as a whole moves to Complete; confirmation banner shown. |

No transition in the table above is reversible, and no stage has a Blocked, On hold or Cancelled state (**OQ-59**).

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

Not scheduled → Scheduled (validated schedule action; reschedule keeps this state) → Completed (completion trigger is OQ-38; unlocks summary and transcript and disables rescheduling). Cancelled and No-show states do not exist. **OQ-60**

## 7. Permissions

Only two roles are modelled. The matrix records what the prototype allows; blank cells are for PM to decide.

| Action | Admin / Planner | Physician | Other roles |
|---|---|---|---|
| Create / open a discharge plan | Yes | No |  |
| Edit plan fields (Steps 1–5) | Yes | No |  |
| Send documents for signature | Yes | No |  |
| Sign a document | No | Yes |  |
| Edit Form 602 | Yes |  |  |
| Delete a referral document | Yes | No |  |
| Fax to agency / pharmacy | Yes | No |  |
| Complete the discharge | Yes | No |  |
| Read meeting transcript / summary | Yes |  |  |

Authentication, session handling and multi-facility scoping are not modelled in the prototype. **OQ-61**

## 8. Audit trail & logging

No audit behaviour is implemented in the prototype. The events below are the ones the workflow produces and are the candidate set for auditing; the retention, format and access rules must come from you.

- Plan created, opened, step advanced, plan completed.
- Field changes on Steps 1–5 and on Form 602 (old value → new value, actor, timestamp).
- Document uploaded, renamed, included/excluded, deleted, viewed, downloaded.
- Document sent for signature; signature applied; signature submitted.
- Fax sent, delivered, failed, retried — per document, per recipient.
- Meeting scheduled, rescheduled, completed; summary edited; transcript downloaded.
- PCC webhook received and manual refresh invoked.

> OQ-62 · Audit requirements Retention period, immutability, who can read the log, whether it must be exportable for survey/audit, and whether document views (not just edits) must be recorded.

## 9. Performance & offline

No targets have been set. Decisions required: **OQ-63**

- Page-load and interaction budgets for the grid (expected resident volume) and the plan.
- Whether the physician mobile app must work offline, and what happens if a signature is submitted without connectivity.
- Behaviour when a fax or PCC call times out mid-action.
- Concurrency: two planners open on the same resident.

## 10. Analytics

Nothing is instrumented today. Candidate events follow the same list as §8, plus time-in-step and drop-off between steps. Tooling, event naming and whether PHI may appear in event properties are undecided. **OQ-64**

## 11. HIPAA

Every screen in this workflow displays PHI: resident identity, date of birth, MRN, room, payer, diagnoses, medications, allergies, mental and physical status (Form 602), meeting transcripts and home address. PHI also leaves the system by fax to agencies and pharmacies, and by file download (medication list, transcript).

The prototype implements no safeguards, so all controls are decisions for you rather than statements of fact. **OQ-65** covers at least: encryption in transit and at rest, session timeout and re-authentication, minimum-necessary access by role, BAAs with the fax, transcription and mapping vendors, whether resident address data may be sent to a third-party mapping API, PHI handling in logs and analytics, breach/incident handling, and secure disposal of downloaded files.

## 12. Consolidated open questions

Owner column left blank for you to assign. Items marked **Blocker** prevent story estimation for that area.

| ID | Area | Question | Priority | Owner |
|---|---|---|---|---|
| OQ-01 | Scope | What is explicitly out of scope for this release? | Blocker |  |
| OQ-02 | Roles | Are nurse / IDT / social worker separate roles? Is plan delegation required? | Blocker |  |
| OQ-03 | Data | Source for MRN, home address, family members, readmission risk, allergies, vitals, caregiver, transportation. | Blocker |  |
| OQ-04 | PCC | Which webhook events fire and what does each payload contain? | Blocker |  |
| OQ-05 | PCC | Where does manual refresh live, and what are its loading and failure states? | High |  |
| OQ-06 | PCC | Snapshot vs live re-read when PCC data changes mid-plan. | Blocker |  |
| OQ-07 | Grid | Readmission risk: source, scale and refresh cadence. | Medium |  |
| OQ-08 | Grid | Confirm grid progress is derived from the same step gates as the plan. | Medium |  |
| OQ-09 | Grid | Row Edit and Delete: what do they act on, who may use them, what confirmation? | High |  |
| OQ-10 | Grid | Resident picker: search, pagination, and the empty state when every resident already has a plan. (Exclusion of residents with an existing plan is now specified.) | Medium |  |
| OQ-11 | Responsive | Minimum supported width and tablet behaviour for the planner app. | Medium |  |
| OQ-12 | Step 1 | What does "Print Discharge Order" produce — the on-screen step or a formatted order document? | Medium |  |
| OQ-13 | Shell | ~~Unsaved-changes handling on Cancel/Close.~~ Resolved: back/Close/Cancel share one exit action with a discard-confirmation dialog (§3.3). | Resolved |  |
| OQ-14 | Shell | Save vs auto-save vs draft: one model for the whole plan. | Blocker |  |
| OQ-15 | Step 1 | Date of birth renders a placeholder value; confirm the PCC field and format. | High |  |
| OQ-16 | Step 1 | May the planner route the signature to a physician other than the attending? | High |  |
| OQ-17 | Step 1 | Order date: plan creation date or physician order date? | High |  |
| OQ-18 | Step 1 | Discharge date validation: allowed range, past dates, relation to meeting date. | Blocker |  |
| OQ-19 | Step 1 | Real Google Places integration? Store place ID / structured address? PHI implications. | Blocker |  |
| OQ-20 | Form 602 | Which destinations require Form 602? Should Board & Care surface it, and should the Step 5 item hide when not applicable? | Blocker |  |
| OQ-21 | Step 1 | DME and lab orders: boolean flag or itemised list? | High |  |
| OQ-22 | Global | Character limits for free-text fields (notes, instructions, comments, summary). | Medium |  |
| OQ-23 | Step 1 | Required fields before the order can be sent; behaviour on re-send after signature. | Blocker |  |
| OQ-24 | Step 2 | Agency directory source, fax numbers, and whether at least one agency is mandatory. | Blocker |  |
| OQ-25 | Step 2 | Include checkbox semantics, and which documents are mandatory in the packet. | Blocker |  |
| OQ-26 | Step 2 | Should row Upload replace that document rather than append a new row? | High |  |
| OQ-27 | Step 5 / Plan | Deleting system-generated documents; gating of Complete Discharge; editability after completion. | Blocker |  |
| OQ-28 | Step 2 | Is document rename required, and where is its entry point? | Low |  |
| OQ-29 | Step 2 | Upload limits: file size, count, malware scanning, retention. | High |  |
| OQ-30 | Fax | Fax provider and granularity of delivery status. | Blocker |  |
| OQ-31 | Fax | Automatic retry policy, and whether Step 5's "Referral Sent" should require delivery. | High |  |
| OQ-32 | Fax | Failure notification: who, how, and within what SLA. | High |  |
| OQ-33 | Step 3 | Is care team genuinely required, and is there a minimum composition? | High |  |
| OQ-34 | Step 3 | Must the meeting date fall before the discharge date? Any lead-time rule? | Blocker |  |
| OQ-35 | Step 3 | Time zone handling, and confirmation of the 09:00–21:00 window. | High |  |
| OQ-36 | Step 3 | Dial-in / video details for remote meetings. | Medium |  |
| OQ-37 | Step 3 | Are invitations, calendar entries or reminders in scope? | High |  |
| OQ-38 | Step 3 | What marks a meeting completed in production? | Blocker |  |
| OQ-39 | Step 3 | Recording, consent, transcription vendor, storage, retention, AI summary review. | Blocker |  |
| OQ-40 | Step 4 | Medication list generation source and who may edit it before signature. | Blocker |  |
| OQ-41 | Step 4 | Download format and whether downloads must be audited. | Medium |  |
| OQ-42 | Step 4 | Public pharmacy API: provider, query parameters, debounce/minimum characters, caching, and no-results / unavailable behaviour. | High |  |
| OQ-43 | Step 4 | Confirm the pharmacy fax gating rule (pharmacy selected + list signed). | Blocker |  |
| OQ-44 | Step 4 | Should unchecking "Not required" restore the previously selected pharmacy, and is an audit note needed when it is used? | Medium |  |
| OQ-45 | Step 4 | Is "one month of medication ordered" (plus date ordered) required on this step? | Blocker |  |
| OQ-46 | Step 5 | DME status value set and the definition of "Loaned". | Medium |  |
| OQ-47 | Step 5 | How is agency acceptance recorded, and can it arrive from the agency? | High |  |
| OQ-48 | Step 5 | Should transportation details be captured as data rather than a checkbox? | Medium |  |
| OQ-49 | Step 5 | Confirm Additional Comments starts empty in production. | Low |  |
| OQ-50 | Step 5 | Are Delivery Status and Referral Summary in scope, and what feeds them? | Blocker |  |
| OQ-51 | Form 602 | Which 602 fields are auto-populated from PCC vs entered manually? | Blocker |  |
| OQ-52 | Form 602 | Default fax destination, number validation, and delivery tracking for the 602. | High |  |
| OQ-53 | Form 602 | Draft persistence, versioning, and lock-after-signature. | High |  |
| OQ-54 | Form 602 | Does the 602 route through the physician signature queue, or is it wet-signed? | Blocker |  |
| OQ-55 | Physician | Which resident details must appear on the signing document? | Medium |  |
| OQ-56 | Physician | E-signature requirements: capture method, re-authentication, legal standard, stored artefact. | Blocker |  |
| OQ-57 | Physician | Decline / request-changes path and how it surfaces to the planner. | Blocker |  |
| OQ-58 | Physician | Notification channel, escalation on no signature, and authentication for the mobile app. | High |  |
| OQ-59 | Plan state | Are Draft, On hold or Cancelled plan states needed? | High |  |
| OQ-60 | Meeting state | Are Cancelled / No-show meeting states needed? | Medium |  |
| OQ-61 | Access | Authentication method, session policy, and multi-facility scoping. | Blocker |  |
| OQ-62 | Audit | Audit scope, retention, immutability, export and read access. | Blocker |  |
| OQ-63 | Non-functional | Performance targets, offline expectations, timeout handling, concurrent editing. | High |  |
| OQ-64 | Analytics | Tooling, event set, and PHI policy for event properties. | Medium |  |
| OQ-65 | HIPAA | Full safeguard set: encryption, session timeout, minimum-necessary access, vendor BAAs, third-party address lookup, PHI in logs, downloads. | Blocker |  |

## 13. Known prototype data defects

These are artefacts of the demo fixtures, not requirements. Listed so they are not carried into stories as intended behaviour.

- Step 1 Date of birth shows a future date; Form 602 §II.2 and the physician document header show two different birth dates.
- The physician document header hard-codes room A-101 regardless of resident.
- The Flow Write-up's referral summary uses a discharge date from a prior year.
- Facility identity is inconsistent: the sidebar reads "Skilled Nursing · Shashi.ai", document previews read "Redwood Grove". Confirm the facility model. **OQ-61**
- Address and pharmacy fixtures span Illinois and California in the same plan.

> Next step after PM sign-off: hand to Development Lead and Architect for Epic and Story creation. Epics were deliberately not proposed here — they should follow the answers to §12.
