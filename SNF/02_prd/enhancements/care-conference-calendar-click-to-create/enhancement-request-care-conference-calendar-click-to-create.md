# Enhancement: Care Conference Calendar — Click-to-Create

| Field | Value |
|---|---|
| Product | SNF |
| Status | Ready for review |
| Base feature | `SNF/02_prd/_as-built/_codebase-analysis/client-admin-web.md` §2.11 (checkpoint pass to `pre-production` HEAD `f5b461c6`, 2026-08-27/28 — documents `CareConferenceCalendar.tsx`); `SNF/02_prd/_as-built/modules/care-coordination.md` CARE-FR-20, CARE-FR-25a; `SNF/02_prd/_as-built/modules/transportation.md` TRN-FR-19c (the calendar pattern this enhancement partially mirrors) |
| Requested by | Sathish |
| Business goal | Bring Care Conference's calendar up to the same standard Transport's calendar already meets — a scheduling entry point, not just a browse/edit/delete view (`TransportCalendar.tsx`, TRN-FR-19c). |

## 1. Current behavior
Per `client-admin-web.md` §2.11 (as-built, checkpoint to `pre-production` HEAD
`f5b461c6`):

- **`CareConferenceCalendar.tsx`** renders Day, Week, and Month views (default:
  Month) over the same `care-conference` data as the list screen, sharing form
  components/validation/conflict-checking with it.
- **Month view:** 7-column grid; each day shows up to 3 conferences with a
  "+N more" toggle. **Correction to the as-built:** the as-built
  (`client-admin-web.md` §2.11) states clicking a day switches to Day view —
  **Sathish confirms this is not accurate**; view switching (Day/Week/Month)
  happens via a separate view-switcher control on the page, independent of
  clicking a day. This is flagged as an as-built inaccuracy to correct
  separately; it removes the trade-off originally flagged in this ER's prior
  draft — a day's click is free to be repurposed for create with no
  navigation side-effect. **Week/Day views:** an hour-by-hour grid (7 AM–10 PM)
  with overlap-safe event layout.
- **Explicitly confirmed as-built (the gap this ER targets):** *"There is no
  'create new conference' action on the calendar itself — new conferences are
  still scheduled from 'Schedule Care Conference.'"*
- New-conference creation goes through the existing scheduling form:
  residents* multi-select, care team* multi-select, family members
  auto-derived, meeting type (`Care Conference` / `Family Meeting` / `Family
  Care Conference`), date* + time*, duration (15/30/40/45/60/90 min, >40 min
  Zoom-limitation warning), where (`In Person` / `Virtual`), location, notes
  (CARE-FR-20) — subject to the existing schedule-conflict check
  (CARE-FR-25a).

## 2. Proposed change
Per Sathish's direct specification (resolves the prior EQ-01):

- **Day view:** clicking any **empty time slot** opens the new-conference
  dialog with **date and time pre-filled** from the clicked slot, and these
  defaults pre-set: **duration = 15 min, where = Virtual, meeting type = Care
  Conference**. All fields remain editable before saving.
- **Week view:** same as Day view.
- **Month view:** clicking any **date** (the empty area of that day's cell,
  not an existing conference entry) opens the new-conference dialog with the
  **date pre-filled** (no time pre-fill — Month's cells have no time
  granularity) and the same defaults: duration = 15 min, where = Virtual,
  meeting type = Care Conference.
- **All views:** clicking an **existing conference** entry continues to open
  that conference's own popup (unchanged — see the companion ER,
  `care-conference-edit-delete-group-notifications`, for what that popup's
  content and actions should be).

**Resolved — no trade-off:** Sathish confirmed Month view's day-click does
not perform any navigation today (view switching is handled entirely by a
separate view-switcher control), so repurposing the day-click for
click-to-create has no side effect on how users reach Day view.

## 3. Scope
### 3.1 In scope
- Click-to-open-create-dialog from an empty hour slot (Day/Week) or empty date
  cell (Month), per the exact pre-fill/default spec above.
- Defaults: 15-minute duration, Virtual, Care Conference meeting type — user
  can still change any of these before saving.
- Reuse of the existing creation form/dialog and its existing validation,
  Zoom-limitation warning, and schedule-conflict check (CARE-FR-25a) unchanged.

### 3.2 Out of scope
- Content/actions of the popup that opens when clicking an **existing**
  conference — covered by the companion enhancement
  `care-conference-edit-delete-group-notifications`.
- Any change to the creation form's fields, validation, or the
  schedule-conflict logic itself, beyond the new pre-filled defaults.
- Facility-group chat notifications — covered by the companion enhancement.
- Fixing the calendar's `COMPLETED`-status filter omission (a separate,
  pre-existing as-built inconsistency, unrelated to this ER).
- Correcting the as-built's inaccurate claim about Month view's day-click
  navigating to Day view — that's a documentation fix for whoever maintains
  `client-admin-web.md`, not a deliverable of this ER.

## 4. Impact
- **Existing stories/tests:** new test coverage needed for the click-to-create
  interaction per view; no existing navigation behavior is being removed
  (per the correction above), so no regression risk there.
- **Base as-built doc:** `client-admin-web.md` §2.11 has an inaccurate claim
  about Month view's day-click (says it navigates to Day view; Sathish
  confirms it doesn't) that should be corrected independent of this ER,
  alongside documenting the new click-to-create behavior once shipped.
- **Does not require a base-PRD update** — no existing PRD for Care
  Conference; stands alone against the as-built ground truth.
- Shares the same `POST /care-conference` create call as the list screen, so
  the companion ER's "Schedule" chat-notification event already covers a
  conference created this way — no additional notification wiring needed here.

## 5. Notes to System Architect
- Confirm Day/Week's hour-slot click target is granular enough (distinct
  clickable regions per hour) to reliably distinguish "empty slot clicked"
  from "existing event clicked," given the overlap-safe event layout already
  in place.
- Flag the as-built inaccuracy on Month view's day-click for correction
  (§1 above) — unrelated to this ER's build but worth fixing since other
  work may rely on that section.

## 6. Open questions
| ID | Area | Question and current position | Priority |
|---|---|---|---|
| EQ-01 | UX | ~~Month view day-click conflict~~ — **resolved**: click-to-create applies to empty dates/slots in all three views per §2; existing conferences still open their own popup. | Resolved |
| EQ-02 | UX | ~~Month-view navigation trade-off~~ — **resolved, no conflict**: view switching is handled by a separate control, not the day-click; nothing to preserve or replace. | Resolved |
