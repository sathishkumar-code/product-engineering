# Care Conference Calendar — Click-to-Create — Tech-Spec

**SNF | Developer Tech-Spec | derived from Technical Design v1**

| Field                  | Value                                                                                                                                                                                                                                        |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Source TD              | `SNF/03_architecture/enhancements/care-conference-calendar-click-to-create/TD-care-conference-calendar-click-to-create.md`, as of 2026-08-30 (Status: Approved) — **re-derive this tech-spec if the TD is revised after this date** |
| Status                 | Approved                                                                                                                                                                                                                                     |
| repo_status            | promoted                                                                                                                                                                                                                                     |
| last_promoted_revision | v1 (Approved) — promoted to `SNF-docs/specs/enhancement-care-conference-calendar-click-to-create-tech-spec.md`, commit `3dc790e`                                                                                                            |

## Overview

`CareConferenceCalendar.tsx` (client-admin-web) currently has no create
affordance — clicking anywhere on the calendar does nothing unless it lands on
an existing conference; new conferences are only created from "Schedule Care
Conference." This builds a click-to-create entry point on Day, Week, and Month
views: clicking an empty slot/cell opens the existing shared scheduling form
in a new **create mode**, pre-filled from the clicked slot/date, submitting
through the unchanged `POST /care-conference` call.

## Non-goals

Carried forward from the TD §2.2, unchanged:

- No change to the creation form's field set, validation rules, or the
  schedule-conflict detection logic (BR-12) — reused as-is.
- No new backend endpoint, no `CareConference` schema change.
- No change to the existing-conference edit/delete popup's content or
  actions — companion enhancement `care-conference-edit-delete-group-notifications`'s
  scope.
- No fix to the calendar's pre-existing `COMPLETED`-status filter omission
  (O-11, BR-12) — unrelated, pre-existing.
- No correction of the as-built's inaccurate Month day-click → Day view claim
  — a documentation fix for `client-admin-web.md`, not a code deliverable
  here.
- No facility-group chat notification wiring — the existing "Schedule"
  notification already fires off `POST /care-conference` regardless of
  invocation source.

## Data model

None. No schema change. Reads/writes the same `CareConference` entity via the
same `/api/care-conference` endpoints already described in
CARE-FR-20/CARE-FR-25/CARE-FR-26a.

## Endpoints

No backend/API surface change. The existing `POST /care-conference` create
endpoint is reused unchanged (BR-CCC-01/BR-CCC-07, spec.md).

| Endpoint                  | Change | Notes                                                                                                                              |
| ------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| `POST /care-conference` | None   | Same request shape, validation, and conflict check regardless of invocation source (calendar click vs. "Schedule Care Conference") |

**Frontend-internal interface change (not an API change):** the shared
scheduling form component (consumed by both `CareConferenceReports.tsx` and
`CareConferenceCalendar.tsx`) gains an explicit `mode: 'create' | 'edit'` (or
equivalent) prop, and accepts synthesized initial values in create mode
instead of requiring a fetched record. No new client- or server-side
authorization check is introduced — the calendar's create invocation runs
under the same authenticated admin-web session and the same backend-side
STAFF|ADMIN role gate the existing create action already enforces
(CARE-FR-26).

## Core logic

**1. Click-target isolation (spike required — see Open Questions, TD-OQ-01).**
Before writing the click handler, run the spike defined in TD §3.2 against
the actual `CareConferenceCalendar.tsx` component and the shared
overlap-safe layout algorithm (shared with `TransportCalendar`):

- (a) Day/Week: does an hour cell's empty space remain independently
  clickable once same-start-time events split into columns, or does a
  full-row/column click handler swallow clicks landing in what looks like
  empty space?
- (b) Month: is the empty area of a day cell independently clickable both
  before and after the "+N more" toggle is expanded?
- (c) Do existing event elements need `stopPropagation()` added so an event
  click doesn't bubble into the new slot/day click handler?

Branch on the outcome:

- If empty regions are already independently clickable and isolated from
  event elements → additive: attach a new handler to the existing
  empty-region element.
- If the current markup swallows clicks in empty space (e.g. full-cell click
  zone with events as overlaid/absolutely-positioned children) → a small
  refactor of the shared event-layout markup is required first, verified not
  to regress `TransportCalendar`'s existing edit/delete click behavior
  (TRN-FR-19c) — that markup is shared between the two calendars.

**2. Existing-event-vs-empty-space dispatch (pseudocode, once §1 above is
resolved):**

```
onCellOrSlotClick(clickTarget, gridPosition):
  if clickTarget is an existing conference element:
    openExistingEditPopup(conferenceId)   # unchanged
  else:
    draft = buildDraft(gridPosition)
    openSharedForm(mode: 'create', initialValues: draft)
```

**3. Draft construction (`CareConferenceDraft`, TD §3.3):**

```
buildDraft(gridPosition):
  date = deriveDateFromGridPosition(gridPosition, facility.timeZone)
  time = (view is Day or Week) ? deriveTimeFromGridPosition(gridPosition, facility.timeZone) : undefined
  return {
    date,
    time,                       # omitted on Month
    duration: 15,
    where: 'Virtual',
    meetingType: 'Care Conference',
    # residents, care team, family members, location, notes: left unset,
    # exactly as "Schedule Care Conference" leaves them unset today
  }
```

**4. Facility-timezone-correct derivation (BR-CCC-05).** `deriveDateFromGridPosition`
/`deriveTimeFromGridPosition` must reuse or mirror exactly the same
grid-position → date/time derivation this calendar already performs
internally for "today" highlighting (CARE-FR-26a) — using facility
`timeZone`, never browser-local `Date` construction, and never the Rehab
Calendar's separate UTC-date-key convention (`client-admin-web.md`
§4.4/§4.6). This is a new code path, not a copy of an existing one, so it
must be written to explicitly reuse the existing convention rather than
reimplemented from scratch.

**5. Month-view empty-cell click boundary, by cell state (TD §3.5):**

| Cell state                          | Expected click target for create-mode                                                                                                |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| 0 conferences that day              | Entire cell body (minus any date-number/header chrome that already has its own click behavior, if any)                               |
| 1–3 conferences, unexpanded        | Any empty area of the cell not occupied by a rendered conference chip                                                                |
| 4+ conferences, "+N more" collapsed | Empty area not occupied by chips or the "+N more" toggle itself (the toggle's own click must continue to expand, not trigger create) |
| 4+ conferences, "+N more" expanded  | Empty area of the now-taller cell not occupied by any of the expanded chips                                                          |

Day/Week hour cells follow the same principle at hour granularity: any region
of the hour row not occupied by an event column.

## Notifications / integration behavior

No new notification wiring. The existing "Schedule" notification already
fires off the shared `POST /care-conference` call regardless of invocation
source (calendar click vs. "Schedule Care Conference" list screen) — no
analytics/invocation-source tag is added (TD-OQ-03, Resolved).

## UI components

- **`CareConferenceCalendar.tsx`** — gains the new empty-slot/cell click
  handler and draft-construction logic (§Core logic above). May also require
  a change to the shared overlap-safe event-layout markup it consumes,
  depending on the §Core logic step 1 spike outcome (that markup is shared
  with `TransportCalendar`).
- **Shared scheduling form component** (from `CareConferenceReports.tsx`) —
  gains an explicit `mode: 'create' | 'edit'` prop and accepts synthesized
  initial values (a `CareConferenceDraft`) in create mode instead of only a
  fetched record.
- **`TransportCalendar`'s shared overlap-safe layout markup** — touched only
  if the spike concludes a refactor is needed; must not regress
  `TransportCalendar`'s existing edit/delete click behavior (TRN-FR-19c) if
  touched.

No new components are introduced; no toolbar/dialog is added as a substitute
(rejected alternative, TD §4).

## Business rules

See `spec.md` BR-CCC-01 through BR-CCC-07 for full text. Referenced directly
by this tech-spec's logic:

- BR-CCC-01 — reuse, not replacement (shared form/validation/conflict-check).
- BR-CCC-02 — pre-fill scope by view (Day/Week: date+time; Month: date only).
- BR-CCC-03 — default field values (duration=15, where=Virtual,
  meetingType=Care Conference), all editable.
- BR-CCC-04 — existing-entry clicks unaffected.
- BR-CCC-05 — facility-timezone-correct pre-fill (see Core logic §4).
- BR-CCC-06 — Month-view day-click has no navigation side effect.
- BR-CCC-07 — no change to conflict/validation logic.

No genuinely technical-only rules beyond spec.md's set are added here.

## Open questions

Mandatory carry-forward from the TD (§11). TD-OQ-01 is dispositioned in this
document (Deferred, by Sathish, explicit and authoritative — see table
below); TD-OQ-02 and TD-OQ-03 are Resolved. Full disposition of any other
still-open item is required only at this tech-spec's own eventual approval
(development-readiness gate), not at this v1 draft.

| ID                                                   | Question                                                                                                                                                                                                                                         | Current status                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Priority |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| TD-OQ-01 / EQ-03                                     | Whether Day/Week's empty hour-slot space and Month's empty day-cell area remain independently clickable once the overlap-safe column layout / "+N more" expand are in play, and whether existing event elements need`stopPropagation()` added. | **Open Question status: Open** — the underlying technical question itself is unresolved; it is not Resolved and not Accepted-as-is. **Disposition: Deferred** (Sathish, explicit, authoritative, recorded at this tech-spec gate) — Sathish has decided what to do about the open question: investigation is deferred to the developer's click-target-isolation spike (Core logic §1 / TD §3.2), which must be run during implementation before Day/Week/Month story sizing. These are two separate facts: (a) the disposition — Deferred — is settled and authoritative now; (b) the underlying technical answer remains Open and will be produced by the spike. This is a blocking condition on Day/Week/Month story sizing, not on TD or tech-spec approval. Do not build the click handler for Day/Week/Month without first running this spike and recording which of the two design-implication branches (additive handler vs. shared-layout markup refactor) applies. | High     |
| TD-OQ-02                                             | Should the new create-on-click empty-region targets (hour-cell background, day-cell background) be keyboard-focusable/ARIA-labeled?                                                                                                              | **Resolved** (Sathish, direct instruction, carried into TD §7/§11) — not required; matches existing mouse-only parity across this calendar and `TransportCalendar`. No further action needed in this tech-spec.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Resolved |
| TD-OQ-03                                             | Should a conference created via the calendar be distinguishable from one created via "Schedule Care Conference" (invocation-source tag)?                                                                                                         | **Resolved** (Sathish, direct instruction, carried into TD §9.3/§11) — no differentiation needed; no analytics tag added. No further action needed in this tech-spec.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Resolved |
| (doc-maintenance, carried from SA Round 1 / TD §11) | `care-coordination.md`'s FR-ID for schedule-conflict detection is dangling (cited as "CARE-FR-25a" but no such FR exists in §3A's sequence).                                                                                                  | Not actioned by this tech-spec or the TD — a documentation-maintenance item for whoever maintains`care-coordination.md`. Does not block implementation since the underlying behavior cited is correctly described regardless of FR-ID.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Low      |

## Testing checklist

- **Unit:**
  - Slot-to-datetime derivation (Core logic §4) — Day/Week hour-slot →
    date+time, Month cell → date-only; explicit assertion it uses facility
    `timeZone`, not browser-local `Date` construction.
  - Mode-prop branching on the shared form component (create vs. edit,
    synthesized vs. fetched initial values).
- **Integration:**
  - Click-target isolation per the Month cell-state table (Core logic §5) —
    empty-region click opens create mode with correct pre-fill; existing-event
    click opens the existing edit popup — across Day/Week's overlap-column
    layout and each Month cell state (0 / 1–3 / 4+ collapsed / 4+ expanded).
  - Create-mode and edit-mode submissions both flow through the same
    `POST`/`PATCH` behavior unchanged (no new validation bypass, no new
    conflict-check bypass).
- **E2E:** full flow per spec.md's functional spec rows #1–#6 — click empty
  Day slot → pre-filled create dialog → edit → save → new event appears on
  calendar; same for Week and Month; click existing conference in any view →
  confirms it opens the *existing* popup, not create mode (regression check).
- **Regression on `TransportCalendar`:** if the Core logic §1 spike concludes
  a shared-layout markup refactor is needed, run `TransportCalendar`'s
  existing edit/delete click-behavior tests (TRN-FR-19c) against the
  refactored markup before merge.
- **Test data/environment:** admin-web session with existing Care Conference
  records spanning same-start-time overlaps (column-split layout) and a Month
  day with 4+ conferences ("+N more").

## Rollout summary

No phased/staged rollout — frontend-only, no-schema-change enhancement,
shipped behind the existing facility-config gate that already controls
whether "Care Conference Calendar" is enabled for a facility (same gate
CARE-FR-26a's view already sits behind; no new gate introduced). No data
migration, no rollback-cleanliness boundary beyond a standard frontend
revert.

**Exit criterion (done for this enhancement):** the full Testing checklist
above passes, and — per TD-OQ-01's Deferred disposition (Sathish, explicit
and authoritative, recorded at this tech-spec's own development-readiness
gate) — the developer has run the TD §3.2 click-target-isolation spike
during implementation, before Day/Week/Month story sizing, and confirmed its
resulting technical answer against Core logic §5's cell-state table.

TD-OQ-01's disposition (Deferred) is settled now and is what satisfies this
tech-spec's development-readiness requirement for this question — it is not
an unmet gate condition. The underlying technical question itself (the
actual click-target-isolation behavior) remains Open: it has not been
resolved, run, or implemented yet. Running the TD §3.2 spike and confirming
its result against Core logic §5's cell-state table is a required follow-up
obligation created by the Deferred disposition, not evidence that this
document's own approval gate is still open.
