# Enhancement: Care Conference — Popup Edit/Delete Parity & Facility Group Notifications

| Field         | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Product       | SNF                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Status        | Approved                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Base feature  | `SNF/02_prd/_as-built/modules/care-coordination.md` (§3A Care conferences, CARE-FR-20, CARE-FR-21, CARE-FR-24 through CARE-FR-26); `SNF/02_prd/_as-built/_codebase-analysis/client-admin-web.md` §2.11 (checkpoint pass to `pre-production` HEAD `f5b461c6`, 2026-08-27/28); `SNF/02_prd/_as-built/modules/messaging-chat.md` MSG-FR-36; `SNF/02_prd/_as-built/modules/transportation.md` TRN-FR-19c, TRN-FR-26 (the patterns this enhancement mirrors) |
| Requested by  | Sathish                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Business goal | Bring Care Conference up to the same standard Transportation already meets: one reusable edit/delete surface shared between list and calendar (`TRN-FR-19c`), and automated facility-group notifications on lifecycle events (`TRN-FR-26`) — consistency across modules, not a one-off request.                                                                                                                                                                  |

## 1. Current behavior

Per `care-coordination.md` CARE-FR-20/21/24/26, `client-admin-web.md` §2.11,
and Sathish's direct confirmation:

- **List screen ("Schedule Care Conference," `CareConferenceReports.tsx`):**
  grid rows show Edit and Delete action icons; the row's popup **already has
  Edit and Delete** (confirmed by Sathish). Delete shows a confirmation dialog
  before soft-cancelling.
- **Care Conference Calendar (`CareConferenceCalendar.tsx`):** clicking an
  event opens an "Event detail popup" — **confirmed by Sathish to have no
  buttons at all today** (no Edit, no Delete, no other actions); it's a
  read-only card (resident profile photo, status badge, date/time, location,
  care-team roster with the host tagged, agenda/notes with empty-state
  placeholders). **Correction to the as-built:** `client-admin-web.md` §2.11
  (checkpoint to `pre-production` HEAD `f5b461c6`) states the calendar
  "already offers Edit and Delete directly" — this is inaccurate per
  Sathish's confirmation and should be corrected by whoever maintains that
  document, independent of this ER. This read-only card is being **reused**,
  not discarded, by the design below.
- **CARE-FR-21 (as-built):** the `DELETE /:id` action underlying every
  "Delete" in this ER is a **soft cancel to `CANCELLED`, permitted only from
  `SCHEDULED`** — tears down calendar events, sends cancellation
  notifications. Same operation as BR-2's "cancellation"; there is no
  separate hard-delete or independent "Deleted" status (unlike Transport,
  TRN-FR-19c/24). `PUT /care-conference/{id}`'s own status gating, if any,
  isn't documented in the as-built (see Notes to SA).
- **Transport's precedent (TRN-FR-19c):** the Schedule/Edit Transport modal
  was extracted into a reusable hook (`useScheduleTransportModal`) opened
  from **both** the request list and the calendar — one shared edit surface,
  not two separately-built popups. This is the pattern this ER asks Care
  Conference to adopt for its editable case.
- No automated notification of any kind fires today when a conference is
  scheduled, edited, or cancelled. Existing notifications are limited to FCM
  push + SMS to conference **participants**, not the wider facility team. The
  platform's `ChatSystemUser`/`Config.chat.moduleMessageBindings`/
  `sendModuleMessage()` mechanism (MSG-FR-36) already solves this for
  Transportation (TRN-FR-26) and is the mechanism this ER reuses.

## 2. Proposed change

1. **Status decides which of two popup modes opens** on row-click (List) or
   event-click (Calendar), used identically from both surfaces:

   - **`SCHEDULED`:** opens **in edit mode by default**, pre-filled with the
     conference's current data (residents, care team, family members,
     date/time, duration, where, location, notes/agenda), with two buttons —
     **Update** (saves via the existing `PUT /care-conference/{id}`, existing
     validation and schedule-conflict check CARE-FR-25a unchanged) and
     **Delete** (triggers the existing `DELETE /:id` soft-cancel, existing
     confirmation dialog, existing `SCHEDULED`-only gating — CARE-FR-21/BR-2,
     unchanged).
   - **Every other status** (`IN_PROGRESS`, `IN_REVIEW`, `COMPLETED`,
     `CANCELLED`): opens the **existing Calendar "Event detail popup"** as a
     **read-only** view — no Update, no Delete. Per Sathish: editing is not
     allowed on an `IN_PROGRESS` conference (this supersedes an earlier
     answer in this ER's history that had `IN_PROGRESS` as editable — the
     final rule is `SCHEDULED`-only for both actions, matching today's
     existing `DELETE` gating exactly).
     Net effect: **no loosening of any existing status gating is needed** for
     Delete. For Update, see Notes to SA — if `PUT` is currently unrestricted
     by status, this ER requires *adding* a new restriction so Update only
     works from `SCHEDULED` (a tightening, not a loosening).
2. **One shared component pair** — a read-only detail view (the Calendar's
   existing card, reused as-is) and an edit-mode form (new, matching
   Transport's `useScheduleTransportModal` pattern) — used identically from
   the List screen and the Calendar, rather than each surface maintaining its
   own version.
3. **Grid row Actions column reduced to the Delete icon only**, shown/enabled
   under Delete's existing `SCHEDULED`-only rule (unchanged) — Edit is
   reached via row-click into the edit-mode popup.
4. **Facility group notification on the conference lifecycle:** add
   `'care-conference'` as a new `ModuleKey` consumer of the existing
   `ChatSystemUser`/`sendModuleMessage()` mechanism. A message posts into the
   facility's configured "Care Conferences" chat group (set via
   `Config.chat.moduleMessageBindings['care-conference']` — the
   per-facility-setting mechanism Transportation already uses) on Schedule
   (create), Edit/reschedule (via Update), and Cancel/Delete (the single
   `DELETE /:id` operation — one event, not two). **Message content, per
   Sathish: patient (resident) name, date, and time only** — no notes,
   agenda, location, or attendee list. This is a deliberately minimal
   payload; it still discloses that a named resident has a care-coordination
   event on the calendar to everyone in the configured chat group, which is
   new PHI exposure relative to today (see Impact).

   Mirrors Transport's card-per-event pattern (shared template renderer,
   per-event `messageNotifyPreference` opt-out, `sendModuleMessage` failure
   never blocking the calling API response). Independent of items 1–3.

## 3. Scope

### 3.1 In scope

- Status-gated popup mode: edit-mode form (Update + Delete) for `SCHEDULED`
  only; read-only detail (reusing the existing Calendar card) for every other
  status.
- Sharing both the read-only card and the new edit-mode form between the List
  screen and the Calendar, rather than each surface having its own.
- Verifying (and, only if `PUT` is currently unrestricted, adding) a
  `SCHEDULED`-only status check on `PUT /care-conference/{id}` so Update
  can't be invoked from any other status via a direct API call either, not
  just hidden in the UI.
- Grid row actions reduced to Delete only, under Delete's existing gating.
- New `care-conference` module-message binding + chat card template,
  restricted to patient name, date, and time, for the schedule /
  edit-reschedule / cancel-delete events.
- Facility-level configurability of the destination chat group (via the
  existing `moduleMessageBindings` config shape, manual provisioning — see
  below).

### 3.2 Out of scope

- **Building an admin-facing UI/API to create `ChatSystemUser` identities or
  manage `moduleMessageBindings`** — per Sathish, this ER uses **manual
  provisioning only** (the same manual DB-write mechanism Transportation
  uses today, MSG-FR-36 §9 gap). Sathish has explicitly marked **a
  self-service admin UI for this as a candidate future enhancement** in its
  own right — not part of this ER, and not yet drafted as its own
  intent.md/ER unless Sathish asks for that separately.
- Any change to who can *view* a conference (CARE-FR-26 visibility rules
  unchanged) — this ER only adds a notification side-channel.
- The status-machine itself, Zoom/Google Calendar integration, and the
  **COMPLETED-conference summary edit/share flow (CARE-FR-24)** — untouched;
  COMPLETED conferences now consistently show the read-only card under this
  ER's rule, same as `IN_REVIEW`/`CANCELLED`/`IN_PROGRESS`.
- The calendar's click-to-**create** behavior on empty slots — covered by the
  companion enhancement `care-conference-calendar-click-to-create`.

## 4. Impact

- **Existing stories/tests:** new coverage needed for the status-gated popup
  mode (edit-mode only for `SCHEDULED`; read-only card for everything else,
  including `IN_PROGRESS` — a change from this ER's own earlier draft, which
  had briefly allowed editing during `IN_PROGRESS`). Grid's Edit icon removal
  needs corresponding test updates. If `PUT` is currently callable from any
  status, a new backend test for the added `SCHEDULED`-only restriction.
- **Base as-built doc:** `client-admin-web.md` §2.11 states the Calendar
  already has Edit/Delete — inaccurate per Sathish's confirmation; should be
  corrected independent of whether this ER ships.
- **Compliance:** flag for the HIPAA compliance register
  (`SNF/03_architecture/compliance/hipaa-compliance-register.md`) — even
  restricted to patient name/date/time, posting this into a chat group is a
  new PHI exposure surface beyond today's host/admin-scoped visibility
  (CARE-FR-26): it discloses that a named resident has a care-coordination
  event to everyone in that facility's configured chat group. The minimal
  content set (name/date/time only, no clinical notes or agenda) is the
  minimum-necessary outcome of this review; Sathish would add the register
  entry per the hipaa-compliance-check skill's access restriction.
- **Does not require a base-PRD update** — no existing PRD for Care
  Conference; this ER stands alone against the as-built ground truth.

## 5. Notes to System Architect

- Confirm whether `CareConferenceReports.tsx` and `CareConferenceCalendar.tsx`
  can converge on shared read-only-card and edit-mode-form components, or
  whether current divergence makes this a non-trivial refactor.
- **Confirm `PUT /care-conference/{id}`'s current status gating.** If it's
  already `SCHEDULED`-only (matching Delete), no backend change is needed.
  If it's currently unrestricted (callable from any status), this ER requires
  *adding* a `SCHEDULED`-only check — a new restriction, not a relaxation —
  so the API can't be used to edit an `IN_PROGRESS`/`COMPLETED`/etc.
  conference even if the UI no longer offers that path.
- Confirm `moduleMessageBindings`/`ChatSystemUser` can support a second
  `ModuleKey` (`care-conference`) without changes beyond config + a new
  template/`resolveMentions` restricted to the three approved fields
  (resident name, date, time).
- No action needed on the "no provisioning UI for `ChatSystemUser`" gap
  (MSG-FR-36 §9) for this ER — Sathish has confirmed manual provisioning is
  acceptable here and flagged a self-service admin UI as a separate,
  not-yet-scheduled future enhancement.

## 6. Open questions

All open questions from this ER's review have been resolved by Sathish:

| ID    | Area              | Resolution                                                                                                                                                |
| ----- | ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| EQ-01 | As-built accuracy | Calendar popup has no buttons today; as-built's contrary claim is inaccurate, flagged for correction.                                                     |
| EQ-02 | UX                | Non-`SCHEDULED` statuses show read-only detail — the existing Calendar "Event detail popup" card, reused as-is.                                        |
| EQ-03 | UX                | Editing is not allowed on an`IN_PROGRESS` conference — Update, like Delete, is `SCHEDULED`-only.                                                     |
| EQ-04 | Compliance        | Chat message content is restricted to patient (resident) name, date, and time only.                                                                       |
| EQ-05 | Platform          | Manual provisioning only for`ChatSystemUser`/`moduleMessageBindings`; a self-service admin UI is a candidate future enhancement, not part of this ER. |
