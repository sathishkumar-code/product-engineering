---
feature: SNF Director Operations Dashboard
product: SNF (Skilled Nursing) — Dashboard, Reporting & Census
slug: director-operations-dashboard
version: 1.11
status: Ready for Development Lead and Architect review — Epic and Story creation.
date: 2026-08-21
design_reference: https://claude.ai/design/p/0f413a3f-eb51-4964-a3cd-30dce8fdb8c6?via=share — "SNF Census Dashboard" is the interactive prototype and behavioral reference for Section 5. "HMO Auth Review" and "HMO Auth Workspace" are Phase 2 design concepts, not part of this release (Section 10).
---

# SNF Director Operations Dashboard

Product requirements for the operational director's daily dashboard: skilled coverage runway, upcoming discharges and post-discharge follow-up for a 100–120 bed skilled nursing facility.

| | |
|---|---|
| **Status** | Ready for Development Lead and Architect review — Epic and Story creation. |
| **Audience** | Product Manager (review & sign-off) → Development Lead and Architect (Epic & Story creation). |
| **Business goal** | Maximise billable skilled days without compromising care. The dashboard exists to make the day a resident stops generating revenue visible early enough to act on it. |
| **Release shape** | One role: **Director (Admin)**. Short-stay skilled residents, plus long-term residents added to tracking one at a time. |
| **Sources** | "SNF Census Dashboard" interactive prototype — behavioral reference for Section 5. Post-discharge survey question/high-risk-answer/feedback-task mapping (Section 5.4) verified against the prototype's exported source (`SNF Census Dashboard.dc.html`, `wk1Questions(week)`, lines 1108–1130; extracted in full in `post-discharge-survey-reference.md`). |
| **Related** | **Discharge Planning** — a separate component, developed in parallel with this dashboard, that owns the discharge workflow's stages and statuses. Every dashboard element that depends on it is hidden or limited in Phase 1; the full inventory is Section 2.3. **IDT Reports** — a separate feature that also writes the resident's discharge date (Section 4.1, Section 7). |
| **Open questions** | 2 open (Section 11). None block Epic/Story creation for this release. |

---

## How to read this document

Every table states required behaviour. Section 5 requirements are tagged with the prototype page they correspond to, in `[brackets]`. Section 2.3 lists every element that depends on the Discharge Planning component and how each is handled until that component ships. Section 11 lists genuine open questions. Section 12 lists prototype artifacts that must not be carried into stories. Section 13 distinguishes the prototype's specific example data from the rules that data demonstrates — swapping the example changes nothing about required behaviour; swapping the rule does.

---

## 1. Assumptions

*These hold for the whole document; a change here reopens Section 6 and Section 7.*

- **A1.** A resident staying at the facility beyond both their discharge date and their skilled days is a non-possibility. The dashboard therefore models no overstay state, no post-exhaustion conversion, and no warning for either.
- **A2.** Because A1 holds operationally, the dashboard performs no threshold validation on the runway itself. The Coverage Expiration Alerts card (Section 5.1) is a derived count of residents already visible on the runway and a shortcut into it — not a notification system: nothing is delivered, queued, dismissed, acknowledged or escalated from it, and the day windows behind its counts are fixed presentation groupings, not configurable rules. The Flagged Areas & Tasks capability in Post Discharge Follow-up (Section 5.4) records task metadata for the Admin to review manually at the daily huddle — also not a delivery mechanism. The post-discharge survey notifications and reminders (Section 5.4) are the one real, system-initiated delivery mechanism in this release, and are scoped narrowly to that survey cadence only (Section 2.2).
- **A3.** Available skilled days is 100 by default (the Medicare Part A benefit period), maintained by the Admin on the dashboard. No eligibility feed exists in this release.
- **A4.** Skilled Coverage shows short-stay residents by default. A long-term/custodial resident who begins a skilled episode can be added to the surface individually by the Director (Section 5.2) and then follows the short-stay workflow in full. Managing the long-term population as a census of its own is not part of this release.
- **A5.** The dashboard reads all data from the Shashi Care app backend. It never calls PCC directly; PCC content reaches it through the existing sync.
- **A6.** A single Director (Admin) uses the dashboard per facility, so concurrent editing of the same resident is not expected.
- **A7.** Both the resident and their responsible party (if one is on file) receive the post-discharge follow-up survey — the dashboard does not choose a single respondent between them. The survey reaches the resident by push notification to their mobile app; the channel for the responsible party is not yet decided (Section 11). If both answer the same week's survey, the most recently submitted response is the one that counts. The dashboard never sends the survey itself beyond this always-both rule, never edits an answer, and never chooses a respondent; it only reads the resulting answers, and shows grey where none have come back yet. The Admin's ability to create follow-up tasks and record an action-taken note (Section 5.4) operates on what a survey flags, not on the survey itself.

---

## 2. Scope

### 2.1 In scope

- Facility summary: occupancy, skilled census, payer mix, care type mix, and average length of stay. Average length of stay is computed over short-stay residents only. `[SNF Census Dashboard → Dashboard]`
- Skilled Coverage runway — one bar per short-stay resident showing days used, days remaining, the discharge date, and benefit exhaustion (Section 5.2). `[SNF Census Dashboard → Dashboard, Skilled Coverage grid]`
- Upcoming Discharges (next 7 days), read from the residents collection (Section 5.3). `[SNF Census Dashboard → Dashboard, Upcoming Discharges]`
- Post Discharge Follow-up over 28 days: a read-only weekly survey delivered by push notification with daily reminders, plus Admin-created task metadata drawn from flagged high-risk answers and an Admin-editable Action Taken note — both reviewed manually at the daily huddle (Section 5.4). `[SNF Census Dashboard → Dashboard, Post Discharge Follow-up]`
- Director-side edits: available skilled days, discharge date, plan authorisation date, and a manual readmission-risk value (Section 4.2, Section 6). `[SNF Census Dashboard → Dashboard, Skilled Coverage grid]`
- Coverage Expiration Alerts summary card — counts of Medicare coverage and managed-care authorisations approaching their end, each acting as a shortcut into the surface that resolves them (Section 5.1). `[SNF Census Dashboard → Dashboard]`
- Adding a long-term resident to Skilled Coverage tracking through a resident picker, and removing them again with confirmation (Section 5.2). `[SNF Census Dashboard → Dashboard, "+ Add resident" modal]`
- Deep link out to the resident profile page.

### 2.2 Out of scope

- **The discharge workflow itself, and everything on this dashboard that would read from or link into it.** Discharge Planning is a separate component, developed in parallel with this dashboard. The Upcoming Discharges progress column, the discharge management dialog's stage list and "Open Discharge Plan" link, the "Start plan" action, the discharge-day diamond's completed state, and reading readmission risk live from that component are all out of scope for this release — hidden or replaced with a manual fallback until Discharge Planning ships. Full inventory in Section 2.3.
- Revenue in currency. The dashboard counts billable *days*, never dollars.
- Long-term census management as a surface of its own — Medicaid recertification and private-pay review cycles, and with them the Medicare benefit-period rules (60-day wellness break, 20/80-day coinsurance split). Adding an individual long-term resident to skilled tracking *is* in scope (Section 5.2); managing the long-term population is not.
- Alert *delivery* from the Coverage Expiration Alerts card or the Flagged Areas & Tasks metadata — no email, SMS, dismissal, acknowledgement or escalation, and no user-configurable thresholds for either (A2). This is separate from the post-discharge follow-up survey's notifications and reminders (Section 5.4), which are a real, in-scope delivery mechanism to the resident, and — channel pending (Section 11) — to the responsible party.
- A shared or resettable huddle checklist for the morning stand-up. May be revisited later.
- A printable or exportable stand-up sheet.
- Analytics instrumentation.
- HIPAA safeguard specification (encryption, session timeout, screen-sharing policy). Handled at platform level, not in this PRD.
- Persisting filters, sort and view choice per user across sessions.
- Admissions and referral intake, staffing and scheduling, clinical charting.
- **The HMO Authorization Review screen and the auth-extension request workflow.** Phase 2 (Section 10). This release keeps the Skilled Coverage grid's HMO Auth column and the R1–R5 date clamps (Section 5.2), and the Coverage Expiration Alerts "Managed care auth expiring" count (Section 5.1), exactly as specified below — only the richer review/extension-request destination screen is deferred.

### 2.3 Dependencies on the Discharge Planning component

Discharge Planning is being built in parallel with this dashboard. Every dashboard element below depends on it, is out of scope for this release, and ships once that component is ready — tracked as Phase 2 (Section 10). None of these block Epic or Story creation for the rest of the dashboard; they simply don't exist yet.

| Dependent element | Where it would appear | This release | Phase 2 |
|---|---|---|---|
| Discharge planning progress column | Upcoming Discharges (Section 5.3) | Hidden entirely. | Shown, sourced live from Discharge Planning's stage status. |
| Stage list | Discharge management dialog (Section 5.5) | Hidden. The dialog shows only its header and the resident-profile link. | Shown, sourced live from Discharge Planning. |
| "Open Discharge Plan" link | Discharge management dialog (Section 5.5) | Hidden — there is nowhere to hand off to yet. | Deep-links into Discharge Planning at the resident's current stage. |
| "Start plan" action | Skilled Coverage, for a resident with no discharge plan (Section 5.2) | Hidden. The Admin can still set a discharge date directly (Rule 1, Section 7) — this only removes the dedicated action to start a formal plan. | Shown; creates a plan in Discharge Planning and hands it to the discharge planning team. |
| Diamond marker colour (Red / Amber / Green) | Skilled Coverage bar (Section 5.2) | Not built. The diamond renders solid Grey regardless of discharge-date status — no plan-status signal exists until Discharge Planning does. | Adds the full Red/Amber/Green colour, keyed to the resident's discharge-plan status (not started / in progress / complete), sourced from Discharge Planning. |
| Readmission risk | Section 4.2, Section 5.4 | Admin-maintained select (High / Moderate / Low / TBD), set manually per resident. | Read-only, sourced live from Discharge Planning; the Admin select is removed. |
| Deep link to Discharge Planning at a resident's current step | Section 2.1 | Not built (depends on the stage list, above). | Built alongside the stage list. |
| Discharge destination | Subtitle of the Follow-up notes panel only (Section 5.4) — the Week check-in panel's subtitle shows that week's check-in due date in this slot instead, not destination | Hidden — the subtitle shows resident name and discharge date only, with no destination segment. | Added to the subtitle, sourced live from Discharge Planning's Step 1 "Discharge to" selection (Home / ALF / Board & Care / Other, plus address where applicable). Confirmed out of scope for IDT Reports' own destination field (`prd-idt-reports-v2.md`, Section 5.2) — that's a separate, Social-Worker-entered value on the IDT report; both this dashboard and IDT Reports depend on Discharge Planning as the eventual source, they don't source it from each other. |

---

## 3. Personas

| Persona | Release | Use of this dashboard |
|---|---|---|
| **Operations Director (Admin role)** | This phase | The only user. Runs the 9:00 stand-up from it, sees which residents are approaching benefit exhaustion, and maintains available skilled days, discharge dates, and (until Discharge Planning ships, Section 2.3) readmission risk directly on the dashboard. |
| **Discharge Planner / clinical staff** | Next phase | Staff do not receive this dashboard. A separate staff dashboard is planned; its requirements are not in this document. |

*The attending physician is not a user of this dashboard.*

---

## 4. Data sources

### 4.1 Fields and provenance

*The dashboard reads exclusively from the Shashi Care app backend (A5). "Synced from PCC" identifies the upstream origin of the data, not a call the dashboard makes.*

| Field | Source | Notes |
|---|---|---|
| Resident name, room, unit | Shashi Care backend, synced from PCC | Read-only here. PCC remains system of record. |
| Payer | Shashi Care backend, synced from PCC | Medicare · Medicaid · Managed Care · Self-Pay. |
| Admission date / length of stay | Shashi Care backend, synced from PCC | Drives the day counter on every coverage bar. |
| Available skilled days | Fixed default of 100, maintained by Admin | Hard-coded to 100 for every resident (Medicare Part A benefit period). Admin updates it per resident when the real figure differs. No eligibility API, clearing house or PCC read in this release. |
| Discharge date | App backend — three entry points | A single field on the resident record; no separate planned/confirmed date. Settable from this dashboard (Admin); from IDT Reports' Orders/Discussion section via its "Discharge date set" decision chip (see `prd-idt-reports-v2.md`, Section 5.2); and, once built, from Discharge Planning's first step (Section 2.3). Last write wins; always editable. |
| Discharge planning progress | Discharge Planning component | Hidden this release — see Section 2.3. |
| Readmission risk | This release: Admin-maintained select on the dashboard. Phase 2: Discharge Planning, read live (Section 2.3). | See Section 4.2 for display and rollup rules. |
| Plan authorisation end date | Admin entry on the dashboard | Managed Care only. No payer-portal integration; Admin keys the date the payer has granted. |
| Follow-up survey answers (weeks 1–4) | App backend | Read-only for the Admin. Delivered to the resident by push notification with daily reminders (Section 5.4); responsible-party channel open (Section 11). Task metadata, a per-week Action Taken note, and a resident-level Additional notes field, all Admin-writable, are recorded separately (Section 5.4, Section 7). |
| Discharge destination | Discharge Planning component | Hidden this release — see Section 2.3. Shown in the Follow-up notes panel's subtitle only (Section 5.4), once available. |
| Occupancy and bed inventory | PCC Facility Beds API | Bed inventory from Facility Beds API; occupancy/payer/care mix derived from it and the resident list, never entered. |

### 4.2 Readmission risk

This release, before Discharge Planning exists, the Admin directly maintains readmission risk per resident via a select box: **High**, **Moderate**, **Low**, or **TBD** (the default, unset state). Once Discharge Planning ships, this becomes read-only and is sourced live from that component, and the Admin select is removed (Section 2.3).

Risk values are rendered as **plain black text** — High, Moderate, Low, TBD — with no colour coding on the text itself. (Colour is used elsewhere in this document for coverage bars, the discharge-day diamond, and Post Discharge Follow-up week segments — Section 5.2, Section 5.4 — never for the risk word itself.)

**Combined risk rule:** displayed risk is the highest of the current readmission-risk value (Section 4.1) and any week's survey-derived risk (Section 5.4). Order, highest to lowest: High > Moderate > Low > TBD. TBD carries no signal and is excluded from the comparison; if every input is TBD or unanswered, the displayed value is TBD. A survey response never lowers the displayed value — it is a **running maximum**. This single rule governs every risk value on the dashboard (Skilled Coverage, Upcoming Discharges, Post Discharge Follow-up, discharge management dialog) — no two surfaces can disagree.

Survey-derived risk for a week is binary: any High-risk question answered Yes → High for that week; otherwise → Low. There is no Moderate outcome from a survey (Section 5.4).

### 4.3 Refresh and concurrency

Pulls data on load and on navigation. Per A6, one Admin per facility, so concurrent-edit resolution, locking, and merge behaviour do not apply and must not be built.

### 4.4 Dates, time zone, "today"

- Facility time zone from facility settings; every date query (runway counts, "next 7 days", follow-up windows) must be evaluated in facility-local time, not browser/server time.
- All dates stored in backend must be UTC; converted at query/render time.
- "Today" cuts over at **end of facility's business hours**, not midnight (a 4pm discharge shouldn't roll off the board while evening shift is on it). **Business hours must be added to facility settings** — a new settings requirement created by this PRD.
- Date format fixed product-wide to **mm/dd/yy** for every discharge/auth/follow-up date; not a facility setting.

---

## 5. Screen specifications

### 5.1 Header and Coverage Expiration Alerts

`[SNF Census Dashboard → Dashboard]`

Greeting + facility-local date, facility summary cards beneath. No alert centre, no alert list, no notification tray.

| Row | Count & click behaviour |
|---|---|
| Medicare coverage expiring | Medicare residents with ≤5 skilled days remaining. Click maximises Skilled Coverage, sets payer filter=Medicare, applies *Expiring ≤5 days* scope. |
| Managed care auth expiring | Managed Care residents whose HMO auth ends in ≤5 days (not already past). Click behaves exactly like the Medicare row: maximises Skilled Coverage, sets payer filter=Managed Care, applies *Expiring ≤5 days* scope. It does not open a separate review screen — the full-screen HMO Authorization Review surface and its auth-extension request flow are Phase 2 (Section 10). |
| Consistency | Both land on a two-chip scope control *Expiring ≤N days \| All \<payer\>* at the right of the toolbar — persists until changed; clearing/choosing a payer chip returns to normal filter. |
| Zero state | Card renders neutral with zero counts; never hidden — a zero is information. |

*Specific counts shown on summary cards are illustrative example data, not requirements — see Section 13.*

### 5.2 Skilled Coverage (primary surface)

`[SNF Census Dashboard → Dashboard, Skilled Coverage grid, collapsed and maximised]`

One row per tracked resident (every short-stay + any long-term added via A4). Horizontal 0–100 day scale (not %). "Covered coverage end" below means the resident's benefit-exhaustion date (admission + available skilled days).

| Element | Behaviour & rules |
|---|---|
| Column order | Resident · Skilled days · HMO auth · LOS · Discharge date · Risk\* · Skilled days (0–100) bar. |
| Resident column | Name/line 1, room/line 2. Editable skilled-days is its own column (2nd). "Resident" used throughout, never "patient". The name is a clickable link to the resident's profile page (Rule 10). |
| Coverage bar — Medicare | **Green:** admission → covered coverage end if no discharge date is set, or admission → discharge date if one is set. **Amber:** discharge date → covered coverage end, only once a discharge date is set. **Grey:** covered coverage end → end of bar. |
| Coverage bar — Managed Care | **Green:** admission → HMO auth date if no discharge date is set, or admission → discharge date if one is set. **Amber:** discharge date → HMO auth date once a discharge date is set, or HMO auth date → covered coverage end while no discharge date is set. **Grey:** covered coverage end → end of bar. |
| Markers | Black vertical line = today's LOS. Diamond = discharge date, draggable: solid **Grey** this release, regardless of whether a discharge date is set — the diamond's colour is keyed to the resident's discharge-plan status (not started / in progress / complete), which cannot be read until Discharge Planning ships. Once it does, the diamond takes Red/Amber/Green from that status (Phase 2 — Section 2.3). Clicking a diamond opens a read-only hover tooltip (Payer/LOS/Days remaining/Anticipated D/C). HMO auth end = flush amber tick (Managed Care only). Grey tick = benefit exhaustion. |
| HMO auth column | Managed Care concept only; other payers render **NA** in grey, no add/edit affordance. |
| Filters and sort | Maximised view only: payer chips + status view (active/discharged) + result count; sort by column header. Session-only, not persisted per user. |
| Exiting maximised view | Maximised view only: a **Back** button, not an icon-only collapse/minimise control, returns to the collapsed dashboard view (Rule 9). |
| Add resident (long-term) | Maximised view only, toolbar button. Opens resident picker: long-term residents not tracked, name+room, search, payer chips, room sort, checkbox multi-select, footer count. Idempotent — a tracked resident never appears in the picker. Added residents appear as a highlighted blank row with an **LT** tag, 0 skilled days, NA HMO auth, and "+ Set D/C". Names in this picker are plain text, not profile links (Rule 10) — rows here are selection targets for adding to tracking, not navigation. |
| Remove from tracking | Control on tracked long-term row only. Confirmation dialog names the resident/room and states that skilled days, auth, and discharge entries are discarded. |
| No discharge plan | No "Start plan" action this release (Section 2.3). The Admin sets a discharge date directly via the marker or date picker, the same as any other resident. |
| Row click | Opens discharge management dialog (Section 5.5) — except clicking the resident's name itself, which links to the profile page instead (Rule 10). |

\* Risk — Readmission Risk (Section 4.2).

**Date ceilings**

- **R1.** HMO auth date may not fall later than end of granted skilled days — clamped to the resident's actual granted skilled-day end.
- **R2.** Discharge date may not fall later than HMO auth date or skilled-day end — clamped to whichever comes first. Editable via marker drag, the date picker, the LOS input, or by clicking the Discharge date column value directly and editing it in place (parity with R4's HMO auth inline edit) — every entry point saves immediately and re-clamps under R1–R3.
- **R3.** Neither clamp may push a date earlier than today. Where LOS already exceeds granted skilled days, the ceiling is today's LOS — no active resident is ever shown a past discharge date.
- **R4.** HMO auth value renders red if auth ends in <5 days, green otherwise. There is no auth-extension affordance in this grid — extension is negotiated with the payer outside the dashboard (the dedicated review/extension-request workflow is Phase 2, Section 10). Once approved externally, the Admin clicks the HMO auth value and edits the date in place, or drags the auth tick; the write saves immediately and re-clamps under R1–R3.
- **R5.** Collapsed cards show the five residents with fewest skilled days remaining; maximised view shows all.

**Shared visual language:** one traffic-light palette (green `#00A86B`, amber `#FF8A00`, red `#E8342A`) for coverage bars, the discharge-day diamond, and progress signals; all dates in `mm/dd/yy`. Readmission risk is plain black text only, never colour-coded and never a chip (Section 4.2). This palette and date format are a fixed design-system contract — see Section 13.

### 5.3 Upcoming Discharges

`[SNF Census Dashboard → Dashboard, Upcoming Discharges]`

Residents with discharge date in next 7 days, ascending. Columns: Resident (name/room — name is a clickable link to the resident's profile page, Rule 10), Discharge date (`mm/dd/yy` + Today/Tomorrow/In n days), Risk\* (plain black text, Section 4.2). Collapsed=5; maximised=full list w/ filters+sortable headers. Clicking a row opens the discharge management dialog (Section 5.5), except clicking the resident's name itself, which links to the profile page instead (Rule 10). Exiting the maximised view uses a **Back** button, not an icon-only collapse control (Rule 9).

\* Risk — Readmission Risk (Section 4.2).

The discharge-planning-progress column is hidden this release — see Section 2.3.

### 5.4 Post Discharge Follow-up

`[SNF Census Dashboard → Dashboard, Post Discharge Follow-up, collapsed and maximised]`

Discharged residents, 0–28 day scale, four equal 7-day segments (one per week, counted from the discharge date — Section 5.4 delivery rules below), each fills left-to-right by elapsed portion. Sortable by risk or recency. Each row identifies the resident (name — clickable, links to the resident profile page, Rule 10) alongside their 4-week progress bar. Like Skilled Coverage and Upcoming Discharges, this card has a collapsed default view and a maximised full view; the maximised view exits via a **Back** button (Rule 9), not an icon-only collapse control. The survey itself is read-only for the Admin (A7): the Admin never answers or edits a survey question. What the Admin can do is act on what a survey flags — creating task metadata and recording a note — described below.

**Delivery and reminders**

- Each week is a 7-day period counted from the discharge date (discharge date = day 0) — not the calendar (Sunday–Saturday) week. On that basis, the survey is pushed to the resident's mobile app on day 5 post-discharge (day 5 of week 1), day 12 (day 5 of week 2), day 19 (day 5 of week 3), and day 26 (day 5 of week 4) — four notifications total. The responsible party, if one is on file, receives the same survey on the same schedule; the delivery channel for the responsible party is not yet decided (Section 11).
- A daily reminder applies only to the **current week's** outstanding survey — reminders are never sent for a week that isn't the current one. A week's reminders stop the moment that week's response is received, and the cycle starts again from day one when the next week's survey notification is sent. A week's reminders never carry over past that week's boundary, whether or not it was ever answered.
- If both the resident and the responsible party answer the same week's survey, the most recently submitted response is the one that counts; the earlier one is overridden.

*Week segment colour:* a week that has not started yet is **light grey**. The current week and any completed week are filled **dark grey** by default, and the current week's number is shown in **bold**. Once a survey response lands for a week, that week's entire segment switches from dark grey to the risk colour derived from the response — **Green** for Low, **Red** for High (Section 4.2) — replacing the default fill for the full width of that week's segment.

*Survey questions and risk rule:* every question is Yes/No. A week's survey-derived risk is binary: any High-risk question answered Yes → High for that week; otherwise → Low (Section 4.2).

Each question's high-risk answer surfaces one specific, fixed feedback task — a one-to-one mapping per question, not a shared pool of generic templates (see **Flagged Areas & Tasks** below). Question wording, high-risk answers, and feedback-task text below are taken verbatim from the prototype's source (`post-discharge-survey-reference.md`).

**Week 1 — home safety and continuity of care**

| Question | High-risk answer | Feedback task |
|---|---|---|
| Are you alone at home? | Yes | Arrange caregiver / social work support — resident alone at home |
| Were you able to pick up your medications from the pharmacy? | No | Resolve pharmacy pickup barrier and confirm meds in hand |
| Did the home health agency contact you? | No | Call home health agency — start of care not made |
| Did you get the durable medical equipment that was ordered? | No | Follow up with DME vendor on undelivered equipment |
| Have you had any falls since discharge? | Yes | Fall follow-up: notify PCP and request home safety evaluation |

**Week 2 — services started and basic needs met**

| Question | High-risk answer | Feedback task |
|---|---|---|
| Have you started receiving home health rehabilitation? | No | Escalate to home health agency — rehab visits not started |
| Are you taking your medications as prescribed? | No | Pharmacist medication review call |
| Do you have access to meals? | No | Refer to meal delivery / community nutrition program |
| Do you have a follow-up appointment with your PCP? | No | Book PCP follow-up appointment and confirm with resident |
| Have you had any falls since discharge? | Yes | Fall follow-up: notify PCP and request home safety evaluation |

**Week 3 — sustained recovery**

| Question | High-risk answer | Feedback task |
|---|---|---|
| Are you taking your medications as prescribed? | No | Pharmacist medication review call |
| Have you seen any provider since discharge? | No | Arrange provider visit — no post-discharge visit yet |
| Are you still receiving home health services? | No | Verify home health status — services appear stopped |
| Have you had any falls since discharge? | Yes | Fall follow-up: notify PCP and request home safety evaluation |
| Have you needed to go to the ER / urgent care since discharge? | Yes | Review ER visit — confirm diagnosis and readmission risk plan |

**Week 4** — identical question set, high-risk answers, and feedback tasks as Week 3 (confirmed in source: `if (week === 3 || week === 4)`, no week-specific variation between them).

**Risk rollup:** displayed risk = highest of the current readmission-risk value and any week with a response, and never comes back down once raised (Section 4.2).

**Week check-in panel:** clicking a week segment opens a "Week N check-in" panel:

- **Subtitle:** resident name (clickable — links to the resident profile page, Rule 10) · Discharged on `mm/dd/yy` · "Week N check-in due `mm/dd/yy`," dot-separated — N and the due date are this specific week's own (not the resident's destination, which appears only in the Follow-up notes panel's subtitle below). Due date = discharge date + 7×N, the end of that week's 7-day window (Section 5.4) — the same boundary after which that week's reminders stop being sent.
- **Survey questions (Yes/No/N/A):** read-only display of the resident's or responsible party's answers. The Admin cannot edit or save changes to these.
- **Flagged Areas & Tasks:** each question in the week's table above has exactly one fixed feedback-task text tied to it, surfaced only when that specific question's answer matches its listed high-risk value — a one-to-one mapping per question, not a shared pool of generic templates the Admin picks from. A week can surface anywhere from zero to five flagged tasks, depending on how many of its five questions came back high-risk; a Low-risk answer to a question surfaces nothing for that question. The Admin selects **"Create task"** for one flagged item or **"Create all tasks"** for every flagged item in that week, saving the selected task text(s) as metadata on the follow-up record — a label and timestamp, nothing more. This is a checklist for the daily huddle, not a task-management system: no owner, due date, status, or assignment, and nothing is delivered or surfaced to any other user or surface. The Admin reviews these manually with the team at the 9:00 huddle. Task text is fixed and pre-set per question (tables above) — no freeform task authoring this phase.
- **Action Taken:** a free-text field, editable and saved by the Admin, recording what was actually done about a flagged week. Persists with the follow-up record.
- **Save & close:** persists the Flagged Areas & Tasks selections and the Action Taken note. It does not alter the week segment's colour or badge — that is driven solely by the survey result, never by whether a task was created or a note was saved.

**Follow-up notes panel:** a separate note icon (independent of the four week segments) opens a resident-level panel that rolls up all four weeks together, rather than showing one week at a time:

- **Subtitle:** resident name (clickable — links to the resident profile page, Rule 10) · Discharged on `mm/dd/yy` · destination, dot-separated — the destination segment is specific to this panel (the Week check-in panel's subtitle shows that week's check-in due date instead, above). This release, destination is omitted — it depends on Discharge Planning (Section 2.3) and isn't available yet — so the subtitle reads resident name · Discharged on `mm/dd/yy` until Discharge Planning ships and supplies it.
- **Summary of responses:** one row per week (1–4), stating whether that week's survey has been answered and, if so, what it found —
  - Not yet responded → **"Not yet responded."**
  - Responded, no high-risk answer → **"All OK."**
  - Responded, one or more high-risk answers → a summary line built by joining together that week's flagged feedback-task text(s) (Week 1–4 tables above), shown for any High-risk week even if only one question triggered it. This is a deterministic template join, not natural-language generation: the 20 (week, question) high-risk triggers in this document collapse to only 12 distinct feedback-task strings (several repeat verbatim across weeks — "Fall follow-up..." appears in all four), so every possible week summary is a fixed, closed-form combination over those 12 strings, computed the same way every time. No AI/LLM call, no third-party vendor, and no synchronous-vs-asynchronous generation question — the join is cheap enough to compute on read.
- **Tasks:** a consolidated list of every flagged high-risk task across all four weeks, not just the currently-open week — deduplicated by task text, so a task flagged in more than one week (e.g. a fall flagged in both Week 1 and Week 3) shows as a single row, not once per week. If that task has already been created — whether via this list's own **Add** button or via **"Create task"/"Create all tasks"** in any week's check-in panel (Section 5.4) — both write the same follow-up task metadata record (Section 7), so the row shows its added status instead of a button; otherwise the row shows an **Add** button. A week with no high-risk answers contributes no rows. This list carries no owner, due date, or status beyond "added" — the same metadata-only model as the per-week Flagged Areas & Tasks list.
- **Additional notes:** a single free-text field for the resident's whole follow-up record — distinct from the per-week **Action Taken** note in the Week check-in panel above — up to 5,000 characters (the app-wide text-field limit used elsewhere in this product).

**Badge:** the follow-up notes icon shows a badge with the count of follow-up tasks created for that resident across all four weeks (via either the per-week check-in panel or this panel's consolidated list) — the same count reflected in the Tasks list above. The badge is a plain count, with no colour-coding or urgency signal.

Full task assignment, ownership, due dates, and staff follow-up are Phase 2 (Section 10) — this release is metadata only, reviewed manually.

### 5.5 Discharge management dialog

`[SNF Census Dashboard → Dashboard, row click on Upcoming Discharges or Skilled Coverage]`

| Element | Behaviour |
|---|---|
| Header | Resident name (clickable — links to the resident profile page, Rule 10), discharge date and day number, plus risk as plain black text ("High"/"Moderate"/"Low"/"TBD", Section 4.2). |

This release, the dialog contains only the header above, whose resident-name link is the dialog's one deep link out to the resident's profile page. The stage list and the hand-off into Discharge Planning are Phase 2 (Section 2.3).

### 5.6 Responsive behaviour

Two devices: laptop (daily use) and wall-display TV (morning stand-up). Design laptop-first, verify legibility at TV viewing distance. Below 860px sidebar collapses to bottom bar, tables reflow to single column; tablet/phone not target devices.

---

## 6. Business rules

> **RULE 1 · DISCHARGE CANNOT EXCEED AUTHORISED DAYS** — No resident may hold a discharge date beyond their available skilled days — a business impossibility (A1), not a warning. Enforced silently at every entry point. Because the situation cannot arise in practice, no notification, confirmation or log entry accompanies the clamp.

| Rule | Statement |
|---|---|
| 2 · Runway definition | Days of runway = days until earlier of discharge date and benefit exhaustion. No discharge date → runway = days of benefit remaining, resident flagged as unplanned. |
| 3 · A dated resident stays active | Future discharge date stays on active runway until date passes; date changeable if plan changes. |
| 4 · Progress granularity (Phase 2) | Once Discharge Planning ships, discharge planning progress will be a percentage derived from completed stages ÷ total stage count, per that component's own stage definition. Not built this release (Section 2.3). |
| 5 · Colour discipline | One traffic-light palette carries every coverage-bar, discharge-day-diamond, and progress signal: green `#00A86B`, amber `#FF8A00`, red `#E8342A`. Readmission risk is plain black text — never colour-coded and never a chip (Section 4.2). Payer chips are the one non-semantic use of colour. |
| 6 · Risk display | This release, readmission risk is an Admin-maintained value, shown as plain black text (Section 4.2). Once Discharge Planning ships, it becomes read-only from that component. |
| 7 · No configurable thresholds | No day threshold is user-configurable, no validation built on one. The 5-day window behind Coverage Expiration Alerts (Section 5.1) is a fixed presentation grouping — raises nothing, blocks nothing. |
| 8 · Tracking is not admission | Adding/removing a long-term resident from Skilled Coverage changes only what the dashboard tracks. Never an admission, level-of-care change, or discharge; writes nothing to the resident's clinical record. |
| 9 · Maximised-view exit | Every maximised card (Skilled Coverage — Section 5.2; Upcoming Discharges — Section 5.3; Post Discharge Follow-up — Section 5.4) exits via an explicit **Back** button, never an icon-only collapse/minimise control. Returns to the collapsed dashboard view. |
| 10 · Resident name links to profile | Everywhere a resident's name appears as a display element on this dashboard — the Skilled Coverage grid, Upcoming Discharges, Post Discharge Follow-up's row list and its popups' subtitles (Section 5.4), and the discharge management dialog header (Section 5.5) — the name itself is a clickable link to that resident's profile page. Where the name sits inside an otherwise-clickable row, clicking the name specifically opens the profile; every other click target in that row keeps its own existing behaviour unchanged — Skilled Coverage and Upcoming Discharges: opens the discharge management dialog (Section 5.5); Post Discharge Follow-up: each week segment still opens its own Week check-in panel, and the note icon still opens the Follow-up notes panel (Section 5.4). **Exception:** the long-term resident picker (Section 5.2) — its rows are selection targets for adding a resident to tracking, not navigation links, so names there are not profile links. |

---

## 7. Writes performed by the dashboard

| Write | Entry points | Constraints |
|---|---|---|
| Available skilled days | Inline editor in Skilled days column (Admin only) | 0–100, default 100. Clamps discharge date (Rule 1). |
| Discharge date | Marker drag, date picker, LOS input, or inline click-to-edit on the Discharge date column value, on this dashboard; the "Discharge date set" decision chip in IDT Reports' Orders/Discussion section (`prd-idt-reports-v2.md`, Section 5.2); and, once built, Discharge Planning's first step (Section 2.3) | Cannot exceed authorised days; cannot precede today. One shared field — last write wins regardless of which surface wrote it. |
| Readmission risk (this release only) | Select box (High / Moderate / Low / TBD) in Skilled Coverage and the discharge management dialog | Admin only. Manual placeholder until Discharge Planning ships, at which point this write is removed (Section 2.3). |
| Long-term resident tracking | Add via resident picker (maximised toolbar); remove via row control, behind confirmation | Per-resident tracking flag, Admin only. Neither touches admission, level of care, or clinical record. |
| Plan authorisation date | Click HMO auth value to edit in place; drag auth tick | Managed Care only. Clamped by R1–R3. |
| Follow-up task metadata | "Create task" / "Create all tasks" in the Week check-in panel, and the equivalent per-row **Add** button in the Follow-up notes panel's consolidated Tasks list (Section 5.4) — both entry points write the same record | Saves a template label + timestamp on the follow-up record. No owner, due date, status, assignment, or delivery. Metadata only, reviewed manually at the daily huddle. |
| Follow-up action-taken note | Free-text "Action Taken" field in the Week check-in panel (Section 5.4) | Plain text, saved with the follow-up record. Per-week. |
| Follow-up additional notes | Free-text "Additional notes" field in the Follow-up notes panel (Section 5.4) | Plain text, up to 5,000 characters. One field per resident's whole follow-up record — not per week, and separate from the per-week Action Taken note above. |

**Explicitly not dashboard writes:**

- Follow-up survey answers — completed by the resident or responsible party; the dashboard only reads them (Section 5.4). The push notifications and daily reminders that solicit those answers are system-triggered on the schedule in Section 5.4, not an Admin action.
- Task assignment, ownership, due dates, or completion tracking — out of scope this phase; task metadata (above) is the full extent of this capability until Phase 2 (Section 10).
- HMO auth-extension requests — negotiated with the payer outside the dashboard; the dedicated request workflow is Phase 2 (Section 10).
- Starting a discharge plan — no "Start plan" action exists this release (Section 2.3).

**Save model:** every write saves immediately via API call — no explicit save button, no draft state, no batch commit. UI reflects persisted value once call succeeds; failed call surfaces inline error, leaves previous value in place.

---

## 8. Permissions and access

One role: **Director (Admin)**. Every action available to that role, none to any other. Staff receive a different dashboard in a later phase.

| Action | Director (Admin) | Any other role |
|---|---|---|
| View census and coverage runway | Yes | No access to this dashboard |
| Edit available skilled days | Yes | — |
| Move the discharge date | Yes | — |
| Set the readmission-risk value (this release only, Section 4.2) | Yes | — |
| Read the weekly follow-up surveys | Read only — resident/responsible party answers | — |
| Add or remove a long-term resident from tracking | Yes | — |
| Open the resident profile | Yes | — |

Starting a discharge plan and deep-linking into Discharge Planning are Phase 2 actions (Section 2.3); they are not part of this release's permission set.

---

## 9. Non-functional requirements

- **Performance.** Size to 500 residents per facility. All performance budgets measured against a 500-resident facility; virtualisation required if runway can't meet them at that size.
- **Notifications.** Push delivery to the resident's mobile app and the day-5 / weekly reminder cadence (Section 5.4) rely on the platform's existing push infrastructure. The responsible-party channel is not yet decided (Section 11).
- **Audit.** Next phase. Logging of actor/old value/new value/timestamp for revenue-bearing writes (Section 7) deferred; nothing audited this release.
- **Accessibility.** Next phase. Includes keyboard equivalent for marker drag.
- **Analytics.** Out of scope — nothing instrumented.
- **HIPAA.** Out of scope for this PRD; platform-level safeguards apply. Follow-up answers and skilled-day overrides must be persisted in the app backend, not browser storage.

---

## 10. Next phase

*Agreed direction, deliberately not in this release.*

- **Full Discharge Planning integration** (Section 2.3): the Upcoming Discharges progress column, the discharge management dialog's stage list and "Open Discharge Plan" link, the "Start plan" action, the diamond marker's full Red/Amber/Green colour logic, and switching readmission risk from an Admin-maintained value to a live read from Discharge Planning.
- A separate **staff dashboard**, with its own requirements.
- **Full task management system**: assignment of follow-up tasks to specific staff, due dates, status/completion tracking, and follow-up — extending this release's metadata-only task list (Section 5.4). Paired with a **staff-facing dashboard** that lets staff see and manage their own assigned tasks from a single page.
- **HMO Authorization Review**: the full-screen managed-care authorization review surface and its auth-extension request workflow, potentially incorporating the richer utilization-review concept explored under "HMO Auth Workspace" (named payer plans, a continued-stay documentation checklist, a clinical-justification field, an audit log). This release's Coverage Expiration Alerts count and the Skilled Coverage grid's HMO Auth column/clamps (Section 5.1, Section 5.2) are unaffected.
- Push follow-up survey answers to **PCC Progress Notes**.
- Extend follow-up answer visibility to **staff associated with the resident**.
- **Audit trail** for revenue-bearing edits.
- **Accessibility** conformance, including keyboard operation of the discharge marker.
- Long-term skilled-days management, which carries the Medicare benefit-period rules and the custodial view with it.

---

## 11. Open questions

| # | Area | Question | Priority |
|---|---|---|---|
| 1 | Follow-up notification channel | The resident receives the follow-up survey by push notification to their mobile app; the resident app's own UI for displaying and answering it is a separate, undiscussed dependency on that app's team. The channel for the responsible party (who may not have the resident app) is not yet decided. | Medium |
| 2 | HMO Authorization Review scope | Phase 2 (Section 10) has two candidate design directions — a simpler embedded review flow and a richer "HMO Auth Workspace" case-management concept. Which becomes the real spec is undecided. | Low — Phase 2 only; does not affect this release. |

---

## 12. Known prototype limitations

*Artifacts of building a clickable demo — not requirements, and should not be carried into stories.*

- Occupancy, payer mix, and care-type mix are fixed numbers in the prototype, not derived from the resident list. Production must derive them from the Facility Beds API and the census.
- "Today" is pinned to a fixed date in the prototype so the seeded runway stays meaningful; production must use the facility clock and business-hours cut-over (Section 4.4).
- Post-discharge destinations and follow-up outcomes in the prototype are generated, and its 30-day scale assumes a check-in cadence now refined by this document's delivery/reminder rules to a 0–28 day scale, four real 7-day weeks counted from the discharge date (Section 5.4).
- The notification bell and its red-dot badge in the prototype are decorative and must not be built — there is no alert centre in this product (Section 5.1).
- The prototype holds all Admin edits in browser `localStorage`; production must save immediately through the API (Section 7, Section 9).
- Left-nav items other than Dashboard (Home, Residents, Messages, Care Conference, Rehab, Reports, Staff, Settings) are inert placeholders in the prototype, consistent with this being a single-page product this phase.
- The prototype's survey Yes/No/N/A buttons are editable; production must render them read-only (Section 5.4).
- The prototype renders risk values in colour (red/amber/green) and dates in an `MMM D`-style format; production renders risk as plain black text and dates as `mm/dd/yy` (Section 4.2, Section 4.4).
- The prototype's summary card is labeled "Coverage Alerts"; production labels it "Coverage Expiration Alerts" (Section 5.1).
- The prototype's nested in-dashboard sub-workflow on the "Discharge Order" stage row (with its own "Schedule meeting" action), and the stage list itself, are placeholders for the Discharge Planning component — the entire stage list and hand-off are out of scope for this release (Section 2.3).
- The prototype's "Send new HMO auth request" flow, and the separate "HMO Auth Workspace" exploration, belong to Phase 2 (Section 10) — do not build them this release.
- One resident's (Samuel Grant's) seed data in the prototype shows 96 skilled days, but the HMO-auth-end dialog clamps against a hardcoded 100 — a data artifact, not a rule; Rule R1 (clamp to the resident's actual granted skilled days) is correct as written.

---

## 13. Example values vs. rules

*Some values in the prototype are the specific rule being demonstrated; others are arbitrary seed data standing in for whatever a real facility's data will be. Swapping a rule changes required behaviour; swapping an example value does not.*

**Rules:**

- 100 available skilled days as the default (A3) — the Medicare Part A benefit-period length, not an arbitrary number.
- The 5-day Coverage Expiration Alert window (Section 5.1) — a fixed presentation grouping, applied identically to Medicare and Managed Care.
- The exact hex values `#00A86B` / `#FF8A00` / `#E8342A` for coverage bars, the discharge-day diamond, and Post Discharge Follow-up week segments (Section 5.2, Section 5.4) — a binding visual contract for those elements only. Readmission risk text is never colour-coded (Section 4.2).
- `mm/dd/yy` as the fixed, product-wide date format (Section 4.4) — a binding format choice, not incidental.
- The 0–100 day coverage-bar scale and the 0–28 day / 4-segment (one real 7-day week each) follow-up scale — structural to how the two surfaces are read.
- The risk running-maximum rule and the binary Yes/No high-risk mapping for each of the 20 survey questions across 4 weeks (Section 5.4) — clinical logic; survey-derived risk is High or Low only, never Moderate.
- The exact feedback-task text tied to each of the 20 (week, question) high-risk flags (Section 5.4) — verbatim strings from the prototype's `wk1Questions(week)` source, not placeholder examples. Each question maps to exactly one fixed task string; production must render this exact wording, not a paraphrase or a shared generic template pool.
- The R1–R5 date-clamping ceilings (Section 5.2).
- The day-5 / every-5th-day survey notification cadence, with each week a 7-day period counted from the discharge date rather than the calendar week (Section 5.4) — a fully specified rule.

**Example values:**

- All resident names and room numbers seen in the prototype (Harold Weiss, Grace Liu, Karen Mbeki, Gerald Pope, Rosa Iglesias, Samuel Grant, Alma Whitfield, Thomas Reyes, Nadia Farah).
- The specific counts shown on summary cards (occupancy percentage, bed counts, average LOS, Coverage Expiration Alert counts, payer-mix counts) — all demo fixtures, see Section 12.
- The specific payer-plan names explored for HMO Auth Workspace (e.g. Aetna MA, Humana Gold, UHC MA, BCBS Advantage) — illustrative; that whole screen is Phase 2 (Section 10).
- The specific pinned "today" date used to keep the prototype's seed data coherent (Section 12) — the rule is that today's date is always the live date (Section 4.4), not this particular date.

---

## 14. Engineering follow-up

Decided at the product level; the remaining work is technical, not a product ambiguity.

- **Test-data cleanup.** Samuel Grant's seed data (96 skilled days, but clamped against a hardcoded 100 in the HMO-auth-end dialog) should be corrected for a clean demo; no logic change is needed, since Rule R1 (clamp to the resident's actual granted skilled days) is already correct as written.
- **Push notification implementation** (Section 5.4, Section 9, Section 11 item 1). The day-5/12/19/26 cadence is settled (Section 5.4); confirm the responsible-party channel with the Architect before Story creation for Section 5.4. Reminder start/stop mechanics are also settled and need no further product decision.
- **HMO Authorization Review scoping** (Section 10, Section 11 item 2). When Phase 2 begins, reconcile the two candidate design directions into one spec before building.
