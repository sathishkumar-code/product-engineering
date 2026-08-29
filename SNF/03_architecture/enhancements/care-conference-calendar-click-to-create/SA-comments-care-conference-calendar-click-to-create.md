# SA Review: enhancement-care-conference-calendar-click-to-create

| Field | Value |
|---|---|
| Source document | `SNF/02_prd/enhancements/care-conference-calendar-click-to-create/enhancement-request-care-conference-calendar-click-to-create.md` |
| Product | SNF |
| review_round | 1 |
| Verdict | Approved-as-is |

## Source Document Review (Round 1)

Grounded against `SNF/02_prd/_as-built/_codebase-analysis/client-admin-web.md` §2.11,
`SNF/02_prd/_as-built/modules/care-coordination.md` §3A (CARE-FR-20/21/22/24/25/26/26a,
BR-12, O-6, O-11), and `SNF/02_prd/_as-built/modules/transportation.md` TRN-FR-19c —
not just the ER's own citations.

**Feasibility: sound, no backend surface needed.** The ER's core claim — that this
reuses the existing creation form, existing validation, existing Zoom-limitation
warning, and the existing schedule-conflict check, and calls the same
`POST /care-conference` the list screen already uses — holds up against the as-built.
`CareConferenceCalendar.tsx` already imports its form fields, `ConflictPanel`, and
`JoinByPhoneSection` straight from `CareConferenceReports.tsx` (client-admin-web.md
§2.11, "the same 'shared list/detail/edit component' pattern" note), and BR-12 confirms
the calendar is a view over `CareConference`, not a second source of truth or a second
endpoint set. Nothing in this ER requires a new backend route, schema field, or change
to the conflict-detection logic itself. The companion ER
(`care-conference-edit-delete-group-notifications`) also confirms its own "Schedule"
facility-group notification already covers a conference created this way, since both
flows share the one `POST` call — no additional notification wiring falls out of this
ER.

**One distinction worth naming precisely, not a defect in the ER:** "reuse the
existing creation form/dialog... unchanged" is accurate for the *form component and
the create endpoint*, but not for the *invocation path*. As-built is explicit that
"There is no 'create new conference' action on the calendar itself" today —
`CareConferenceCalendar.tsx` currently only opens the shared form in **edit** mode,
seeded from an existing record. Opening that same form in **create** mode, seeded with
synthesized (not fetched) defaults computed from the clicked slot, is new integration
work on the calendar page, even though it reuses existing components end-to-end. Sized
as its own technical story below so it isn't assumed to be "free" because the form
itself is unchanged.

**The ER's own open technical question (Notes to SA §1) is real and worth a spike,
not a guess.** As-built confirms Day/Week render "an hour-by-hour grid (7 AM–10 PM)
with overlap-safe event layout (same-start-time events split into columns...)" —
consistent with a grid that *could* expose clean empty-cell click targets, but the
as-built is a behavioral description, not a DOM/component-structure guarantee. Given
the layout algorithm is shared with `TransportCalendar` (which has no click-to-create
requirement to have solved this same problem for), this needs to be verified against
the actual component before Day/Week/Month stories are estimated, not assumed from the
as-built prose. See spike below.

**Timezone convention must be carried through explicitly.** `client-admin-web.md`
confirms the Care Conference Calendar follows the facility-`timeZone` pattern (like
Transportation/Housekeeping), explicitly contrasted with the Rehab Calendar's UTC-date-
key policy, and separately notes "date/timezone handling is inconsistent by design
pocket" as a known cross-calendar risk. The new pre-fill logic is a new place this can
regress silently (e.g., if it derives date/time from the DOM cell using browser-local
time instead of the facility timezone already used for "today" highlighting). Flagged
as a technical task below rather than assumed to be automatic.

**Minor, non-blocking documentation gap surfaced during grounding (not this ER's to
fix):** both this ER and the companion ER cite "CARE-FR-25a" as the schedule-conflict-
detection requirement whose behavior stays unchanged. `care-coordination.md` §3A has no
FR heading numbered 25a — the sequence runs CARE-FR-20…26, 26a, and the conflict-
detection behavior is described only in `client-admin-web.md` §2.11's prose ("Schedule-
conflict detection on save..."), never given its own FR-ID. The underlying behavior
being cited is real and correctly described in both ERs; only the FR-ID itself is a
dangling reference in the as-built doc. Worth closing now that two ERs cite it by name,
but it's a `care-coordination.md` maintenance item, not a change to either ER.

**Scope boundaries are clean.** Exclusion of the existing-conference popup content
(deferred to the companion ER), the pre-existing `COMPLETED`-filter gap (O-11), and the
Month-view day-click-navigates-to-Day-view as-built inaccuracy are all correctly kept
out of this ER's build — none of them are touched by adding click-to-create on empty
slots/cells.

### Recommended technical epics/stories/spikes

- **Spike — Calendar click-target isolation (Day/Week hour grid + Month day cell).**
  Read `CareConferenceCalendar.tsx` and the shared overlap-safe layout code (the
  algorithm reused from `TransportCalendar`) to confirm, concretely: (a) in Day/Week,
  whether an hour cell's empty space remains an independent click target once same-
  start-time events split into columns within it, or whether a full-row/full-column
  click handler would swallow the click; (b) in Month, whether the empty area of a day
  cell is independently clickable both before and after the "+N more" toggle is
  expanded; (c) whether existing event elements need `stopPropagation()` added so an
  event click doesn't also bubble into a new slot/day click-to-create handler — today
  there's no create-on-click path at all, so this conflict has never had to be solved.
  This directly answers the ER's own Notes-to-SA §1 question. Do this before
  estimating the Day/Week/Month stories — the answer determines whether a small
  refactor of the event-layout markup is needed alongside the new click handlers.

- **Technical story — Create-mode invocation of the shared scheduling form from the
  Calendar.** `CareConferenceCalendar.tsx` only opens the shared form in edit mode
  today, seeded from an existing `CareConference` record. Add a create-mode
  invocation path — new to the calendar page — that opens the same form seeded with
  synthesized defaults per view: Day/Week get date+time from the clicked slot plus
  duration=15/where=Virtual/meetingType=Care Conference; Month gets date only (no
  time), same other defaults; all fields remain editable before save, per the ER. This
  is genuinely new wiring, not a byproduct of "the form is unchanged" — call it out
  explicitly in estimation.

- **Technical task (attach to the story above) — Facility-timezone-correct slot-to-
  datetime conversion.** Derive the pre-filled date/time from the clicked grid
  position using the same facility-`timeZone` convention this calendar already uses
  for "today" highlighting/default landing date — not browser-local time and not the
  Rehab Calendar's UTC-date-key convention. Call out explicitly given the codebase's
  own documented history of per-calendar timezone inconsistency.

- **Technical task — Month-view empty-cell click boundary, by cell state.** Define the
  exact clickable region for click-to-create across a day cell with 0 conferences, 1–3
  conferences (unexpanded), and 4+ conferences with "+N more" expanded. The ER's
  product intent (empty area → create, existing entry → its own popup) is already
  clear; this task turns it into concrete per-state acceptance criteria once the spike
  above establishes what the current markup actually allows.

## Epics/Stories Review (Round 1)

Not yet applicable — `epics-stories.md` for this slug does not exist yet. Per the
gating rule (`skill-pm-discipline.md`), Epics/Stories drafting also requires a ready
Technical Design in addition to this settled ER verdict; given this ER's small,
contained footprint (no new backend surface, no schema change), a full
`technical-design-template.md` document may be more process than the change warrants —
that's Sathish's call, not decided here. This review's recommended
spike/story/task list above is written so it can be folded directly into
Epics/Stories once that gate is satisfied, whatever form it takes.

## Escalation

Not applicable — settled at Approved-as-is in round 1.
