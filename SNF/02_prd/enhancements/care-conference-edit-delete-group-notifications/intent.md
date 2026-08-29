# Intent: Care Conference — Popup Edit/Delete Parity & Facility Group Notifications

| Field | Value |
|---|---|
| Author | Sathish |
| Status | Superseded by PRD |
| Category | Enhancement |

## Problem
The admin web Care Conferences list screen's popup already has Edit and
Delete; the Care Conference Calendar's popup has neither (confirmed by
Sathish — corrects the as-built, which had incorrectly stated the calendar
already offers both). The two popups' content also differs. Separately,
nothing notifies the wider facility team when a conference is scheduled,
edited, or cancelled, unlike Transportation (TRN-FR-26/MSG-FR-36).

## Proposed outcome
1. **Status decides the popup mode**, shared identically between the List
   screen and the Calendar: `SCHEDULED` conferences open in **edit mode**
   (Update + Delete buttons); every other status (`IN_PROGRESS`,
   `IN_REVIEW`, `COMPLETED`, `CANCELLED`) opens the existing Calendar
   read-only card instead, with no edit or delete action available.
2. Grid row Actions column reduced to the Delete icon only, under Delete's
   existing `SCHEDULED`-only gating (unchanged).
3. A message posts into a facility-configured "Care Conferences" chat group
   on schedule / edit-reschedule / cancel-delete, containing **only the
   resident's name, date, and time** — reusing `ChatSystemUser`/
   `moduleMessageBindings` with `'care-conference'` as a new `ModuleKey`.
4. `ChatSystemUser`/`moduleMessageBindings` provisioning stays **manual**
   for this ER; a self-service admin UI for it is flagged as a **separate
   candidate future enhancement**, not built here.

## Affected users and systems
- **Personas:** Case Manager / Social Worker / Admin (hosts and viewers of
  care conferences, admin web); indirectly, anyone in the facility's
  configured "Care Conferences" chat group.
- **Systems:** `senior_living_admin` (`CareConferenceReports.tsx`,
  `CareConferenceCalendar.tsx` — sharing a read-only card and a new edit-mode
  form between them), `senior_living_backend` (`careConference.service.ts` —
  possibly adding a `SCHEDULED`-only check to `PUT /care-conference/{id}` if
  it's currently unrestricted; `DELETE /:id`'s existing gating unchanged;
  `Config.chat.moduleMessageBindings`).
- **Product:** SNF (Care Conferences is documented only in the SNF as-built).

## Constraints
- Target release: by **2026-09-02**.
- Must be backward-compatible.
- No loosening of any existing status gating — if anything, `PUT` may need a
  new `SCHEDULED`-only restriction added (see Systems above).
- Chat message content limited to resident name, date, time — nothing else.

## Open questions
None remaining — all prior open questions have been resolved by Sathish (see
the Enhancement Request's §6 for the resolution table).
