# ADR-005 — Facility timezone authority for client-side date/time rendering

**Status:** Proposed
**Date:** 2026-07-12
**Area:** `senior_living_backend` — `Config.timeZone` (`src/models/config.model.ts`), `GET /api/config/residency-details`; `senior_living_staffapp` — `src/utils/timezone.ts`, `src/polyfills/*`; (potentially) `senior_living_admin`, `senior_living_reactnative`, `senior_living_skillednursing_resident`, `senior_living_tvapp`
**Related:** `architecture-senior-living-product.md` §2.7; [`review-senior_living_staffapp.md`](../../reviews/2026-07-12/review-senior_living_staffapp.md)

## Context

`Config.timeZone` (String, default `'America/Los_Angeles'`) has existed on the facility config model since before this review cycle, but until now no client treated it as load-bearing for date/time *computation* — at most it was display metadata. In this cycle, `senior_living_staffapp` added a client-side IANA timezone layer (`@date-fns/tz`, plus an iOS-only FormatJS polyfill for a documented Hermes `Intl.DateTimeFormat` bug) that reads `Config.timeZone` via `GET /api/config/residency-details` and uses it to compute appointment/schedule/reminder times in the facility's local time rather than the device's local time.

This is architecturally significant because the platform has **five other client surfaces** (`senior_living_admin`, `senior_living_reactnative`, `senior_living_skillednursing_resident`, `senior_living_tvapp`, and the backend's own cron-driven SMS/notification reminders) that render or schedule the same underlying appointment/schedule data. Whether they treat time as facility-local or device-local was **not verified this cycle** for any of them. A facility can legitimately span a different timezone from a staff member's device (e.g. a corporate admin managing multiple facilities, or a traveling case manager), so a silent mismatch between "facility time" (staff app) and "device time" (everything else) could put the wrong time on a resident's calendar invite, an appointment reminder SMS, or a Zoom care-conference — a correctness class of bug that is easy to miss in same-timezone testing/UAT and only surfaces at multi-region scale.

## Options Considered

### Option A — Facility time is authoritative everywhere (adopt the staff app's approach platform-wide)
- **Pros:** One mental model — "what time does the facility see this at" — matches how care conferences, appointments, and reminders are actually experienced by staff and residents co-located with the facility. Backend already has the source field (`Config.timeZone`); no schema change needed.
- **Cons:** Requires auditing and likely changing five other client surfaces plus the backend's own cron/SMS-reminder scheduling (`src/server.ts` cron jobs currently run in the server process's own timezone, not per-facility). Non-trivial, cross-repo migration.

### Option B — Device-local time is authoritative everywhere (do not adopt the staff app's layer elsewhere)
- **Pros:** No migration needed for the other five surfaces; today's default behavior (implicit device-local rendering) is left alone.
- **Cons:** The staff app's new facility-timezone layer becomes a one-off inconsistency — the exact mismatch risk described above. A staff member off-site or a family member checking a resident's schedule from another timezone would see a different (wrong) time than the facility itself uses.

### Option C — Explicit split by surface class (facility-local for anything facility-operational; device-local for personal/informational views), documented per surface
- **Pros:** Pragmatic middle ground — e.g. care-conference start times and appointment reminders (facility-operational, multi-party) are facility-local; a resident's personal "my next appointment" card could reasonably show device-local with an explicit "(Facility time: X)" annotation. Avoids a blanket migration while still closing the correctness gap for the highest-risk surfaces (backend cron reminders, multi-party scheduling).
- **Cons:** Requires a documented, enforced rule per surface rather than one simple rule — more prone to drift without an owner.

## Decision

**Not yet decided — proposed for human/architect review.** No option is selected in this ADR; it exists to make the now-real inconsistency (one client treats facility time as authoritative, five do not, backend cron scheduling status unverified) a first-class, trackable architectural question rather than an implicit assumption. Recommend Option C as the working default given the mixed operational/personal nature of the surfaces involved, but this requires confirming with product which views are genuinely multi-party/facility-operational vs. personal.

## Consequences

### Positive
- Makes an existing silent assumption (device time == facility time) explicit and falsifiable.
- Gives the next cross-repo doc pass a concrete question to close rather than re-discovering the gap ad hoc.

### Negative
- Until decided, the platform ships with **inconsistent timezone semantics between at least the staff app and everything else** — flagged as an open question, not asserted as a defect, since it may be a non-issue for single-timezone customers (e.g. the current Redwood Grove first-deployment scope).

### Follow-ups
- [ ] Confirm with product/PM whether any near-term customer spans multiple timezones (if not, this is lower urgency).
- [ ] Audit `senior_living_backend`'s cron-driven reminder scheduling (`src/server.ts` cron jobs — appointment-completion, care-conference reminder, announcement) for which timezone they compute against.
- [ ] Audit `senior_living_admin`, `senior_living_reactnative`, `senior_living_skillednursing_resident`, `senior_living_tvapp` for current date/time rendering behavior (facility vs. device local) — none confirmed this cycle.
- [ ] If Option A or C is chosen, scope a ticket per affected repo.

## References
- `architecture-senior-living-product.md` §2.7 (Facility timezone layer)
- [`review-senior_living_staffapp.md`](../../reviews/2026-07-12/review-senior_living_staffapp.md) — "Timezone" section
- `senior_living_backend/src/models/config.model.ts` (`Config.timeZone`)
