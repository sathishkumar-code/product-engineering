# Intent: Care Conference Calendar — Click-to-Create

| Field | Value |
|---|---|
| Author | Sathish |
| Status | Superseded by PRD |
| Category | Enhancement |

## Problem
The Care Conference Calendar (`CareConferenceCalendar.tsx`, Day/Week/Month
views, admin web) is documented in the as-built (`client-admin-web.md` §2.11,
checkpoint to `pre-production` HEAD `f5b461c6`) as having **no create action at
all** — it's a browse/edit/delete surface only; new conferences still require
switching back to "Schedule Care Conference." Month view's day-click currently
navigates to Day view for every day (empty or not); Day/Week's hour-by-hour
grid has no documented empty-slot behavior.

## Proposed outcome
Per Sathish's specification: clicking an empty time slot (Day/Week) or an
empty date (Month) opens the new-conference dialog, pre-filled with the
clicked date (and time, in Day/Week) plus defaults of **15-minute duration,
Virtual, Care Conference meeting type** — all still editable. Clicking an
*existing* conference, in any view, continues to open that conference's own
popup (whose content/actions are the companion enhancement's concern,
`care-conference-edit-delete-group-notifications`). Month view's day-click was assumed (from the as-built) to double as
Day-view navigation — Sathish confirmed that's inaccurate; view switching is
handled by a separate control, so there's no navigation trade-off to worry
about.

## Affected users and systems
- **Personas:** Case Manager / Social Worker / Admin scheduling care
  conferences via admin web.
- **Systems:** `senior_living_admin` (`CareConferenceCalendar.tsx`; reuses
  form components already shared with `CareConferenceReports.tsx`, and the
  underlying `POST /care-conference` flow, including existing validation and
  the schedule-conflict check, CARE-FR-25a).
- **Product:** SNF (same scoping rationale as the companion enhancement).

## Constraints
- Target release: by **2026-09-02**.
- Must be backward-compatible; must not change or bypass existing create-time
  validation (residents/care-team required, duration→Zoom-limitation warning,
  schedule-conflict detection, etc.) beyond the new pre-filled defaults.
- Existing-conference click behavior (opens that conference's popup) is
  unchanged in all three views.

## Open questions
None remaining — the Month-view navigation concern raised in this ER's first
draft was based on an inaccurate as-built claim (day-click switching to Day
view) that Sathish has corrected; view switching is handled independently by
a separate control.
