# Module: Wellness & Personal Services

> Applies to: Both
> FR prefix: WELL
> Sources: `_codebase-analysis/backend-wellness-dining-ops.md` (§0, §1, §7, §8), `client-admin-web.md` (§2.5, §2.6, §2.15, §4.4, §4.6), `client-resident-app-sl.md` (§3.5, §3.8, §3.9, §6), `client-resident-app-sn.md` (§3.4, §3.6, §3.7), `client-staff-app.md` (§1, §4.2–4.4, §5, §6), `client-tv-app.md` (§3.5, §3.6). Code is source of truth.

---

## 1. Purpose & scope

Wellness & Personal Services covers the three parallel **bookable-venue service verticals** — **Salon**, **Massage Therapy**, and **Private Training** — through which residents (or family/staff acting on their behalf) book paid, staff-delivered personal services at the facility.

Each vertical comprises the same building blocks (implemented three times in the backend and three times in the admin web app — see §9):

1. A **venue** record (store details: name, address, contact).
2. A **service catalog** (priced, duration-bound services, each owned by a specific staff member).
3. **Per-day operating hours** for the venue/staff member.
4. A **shared slot/booking engine** that derives bookable time slots from hours, meal windows, existing bookings, the owning staff member's Google Calendar, and the resident's cross-module calendar.
5. An **appointment lifecycle** (book → confirm/waitlist → complete/cancel) with waitlist promotion, reschedule, an auto-completion cron, notifications, and Google Calendar sync.
6. A **per-facility booking policy** controlling which personas may book on a resident's behalf.

In scope: everything above, across all five client surfaces. Out of scope (other modules): the `/care` clinical appointment engine (Physical Therapy evaluations, Cognitive, Outside Agency), SNF Rehab appointments, Transportation, Dining, Activities — these share the UnifiedSchedule spine and are referenced here only where they interact (conflict blocking, reminders).

Terminology note: "Private Training" (PT) is the assisted-living wellness flavor of physical training. Despite its wellness nature, its **staff page-permission key is `Rehab`** (backend route gate `privateTrainingAppointment.routes.ts:63`) — a naming/permission quirk the PRD must either preserve or explicitly migrate.

---

## 2. Personas & surfaces

| Persona | Surface | Capability today (as-built) |
|---|---|---|
| Resident | SN resident app (`senior_living_skillednursing_resident`) | Full salon booking/reschedule/cancel + history. Massage & PT service layers fully wired but **de-merchandised** (Services-hub cards commented out); reachable only via reschedule of an existing appointment. Cancel from unified My Schedule per type. |
| Resident | SL resident app (`senior_living_reactnative`) | **As of staging: fully API-backed for all three verticals.** Salon, Massage Therapy, and Private Training each merchandised in the Services hub with live book / reschedule / cancel / my-appointments / history against the same backend endpoints the SN app uses (`/api/{salon\|massage\|private-training}/{services, :serviceId/available-slots, book-appointment, reschedule-appointment, cancel-appointment, my-appointments[/history]}`). The prior "booking never persists / massage-PT mock" gap is closed; the SL and SN resident apps are now at near-parity for wellness. |
| Resident (TV-paired) | TV app (`senior_living_tvapp`) | Browse all three catalogs unauthenticated; book any of the three after QR-pair auth (7-day slot picker). View bookings in 7-day unified schedule. No cancel/waitlist on TV. |
| Family member | Resident apps (no separate family app) | Backend supports family-member login: auth middleware rewrites the family identity to the linked resident's `cName`; bookings allowed only when facility policy `bookingPermission[module].isFamilyMemberAllowed` is true. No family-specific UI exists in either resident app. |
| Staff — service owner (Salon Stylist / Massage Therapist / Private Trainer designations) | Staff app (`senior_living_staffapp`) | Salon Stylist: today's confirmed list + waitlist tab + "Move to Slot" promotion. Massage Therapist & Private Trainer: **read-only** "Today's Sessions". No availability/hours management anywhere in the staff app. |
| Staff — on-behalf booker (designation-allowlisted) | Backend booking routes | Any staff whose `Staff.designation` is in the module's `staffDesignationAllowed` config may book/view for a resident. Parallel "case manager" routes (salon only, of the three) let any STAFF act by resident `_id` without a designation check. |
| Admin / Super-admin | Admin web (`senior_living_admin`) | Full configuration: venue details, service CRUD with staff assignment, operating hours, appointment queues (Confirmed/Waitlist tabs), waitlist promotion, massage/PT status changes, PT admin-side booking, service-deletion impact review. Bypasses designation checks (module must still be configured). |
| TV device (unauthenticated) | TV app | Read-only catalog browsing via `/tv/services` routes (no auth on these endpoints). |

### 2.1 Capability matrix per surface and vertical

Legend: ● = live & wired · ◐ = partially wired / hidden · ○ = mock or unwired · — = absent by design.

| Capability | SN resident app | SL resident app | TV app | Staff app | Admin web |
|---|---|---|---|---|---|
| Browse salon catalog | ● | ● | ● (pre-auth) | — | ● |
| Book salon | ● | ● (`book-appointment`) | ● | — | — (queue only) |
| Join salon waitlist | ● (via book API) | ● (checkbox sends `joinWaitlist`) | ○ (flag parsed, no UI) | — | — |
| Cancel salon | ● (`cancel-appointment`) | ● (`cancel-appointment`) | — | — | — |
| Reschedule salon | ● | ● (`reschedule-appointment`) | — | ◐ (waitlist move-to-slot only) | ● (waitlist promote, can change date) |
| Browse/book massage | ◐ (live, de-merchandised) | ● (`book-appointment`) | ● | — | — (queue + status PATCH) |
| Browse/book private training | ◐ (live, de-merchandised) | ● (`book-appointment`) | ● | — | ● (admin book-appointment) |
| Today's session sheet | — | — | — | ● (3 designations) | ● (queues, all dates) |
| Service catalog CRUD | — | — | — | — | ● |
| Operating hours edit | — | — | — | — | ● |
| Venue/store profile edit | — | — | — | — | ● |
| Appointment history | ● | ● (per-vertical `my-appointments/history`) | ◐ (7-day schedule view) | — (today only) | ● |

This matrix is the single most important as-built fact for the PRD: **the SN resident app and admin web are the reference implementations**; as of staging the **SL resident app has reached resident-side parity for wellness booking** (all three verticals merchandised and API-backed). Remaining catch-up work is the TV waitlist (flag parsed, no UI) and the staff app, which is intentionally thin.

---

## 3. Functional requirements (as-built)

### 3.1 Venue / store details

- **WELL-FR-01** — Each vertical has a facility-scoped venue entity: Salon (name, address, location, contactNumber), Massage (same + `isActive`), PrivateTraining (same + `isActive`). [backend §1.1]
- **WELL-FR-02** — Admin can edit the salon store profile: name*, contact number* (formatted `(###) ###-####`), address* (≤200 chars), description* (≤500). The read-only "Store Details" display card is currently commented out in the admin UI; only the edit modal survives. [admin §2.5, §4.3]
- **WELL-FR-03** — Venue resolution at booking time: an appointment's `venueId` resolves from the service's venue or the facility default (`resolveSalonId`). [backend §1.2]
- **WELL-FR-04 (gap, as-built)** — Salon venue CRUD routes carry **no auth middleware at all**; massage venue delete requires auth, other venue routes none. The PRD target state must put venue CRUD behind admin write permission. [backend §1.3, §8.1]

### 3.2 Service catalog

- **WELL-FR-05** — A service record carries: name, description, location, image (S3 `imageKey` written / signed `imageUrl` read — platform media contract), **duration in minutes**, price, `isActive` toggle, soft-delete flag, and **`cName` = owning staff member**. The owning staff member is load-bearing: it drives slot availability (their calendar), appointment visibility, Google Calendar sync target, and booking notifications. [backend §1.1; admin §2.5, §2.6]
- **WELL-FR-06** — Admin service-create/edit validation (salon): name*, description* with **minimum 10 words**, location* ≤200, duration* (minutes), price* > 0, staff assignment **required for ADMIN, locked during edit for non-admin staff** (staff can only own their own services). Massage/PT validation is similar (name ≤100, description ≤500) but without the 10-word minimum. [admin §2.5, §2.6]
- **WELL-FR-07** — Per-service active/inactive toggle (`PATCH …/toggle`) hides the service from booking without deleting it; slot generation requires `isActive`. [backend §1.2 step 1; admin §2.5]
- **WELL-FR-08** — Soft-delete flag naming drifts: salon/massage use `isDeleted`, Private Training uses **`isDelete`**. Functional consequence: the PT slot-engine service lookup uses `findById` with **no facility filter and no soft-delete filter** — a deleted or cross-facility PT service can still produce slots. [backend §1.1, §1.2 step 1, §8.6]
- **WELL-FR-09** — **Service deletion impact check (admin two-phase delete):** before deleting a service, the admin UI fetches that service's CONFIRMED appointments, displays the count and list, and requires explicit confirmation. Non-admin users get a simple confirm. The identical pattern exists in all three verticals. Note: this is a client-side courtesy check only — the backend does not itself cascade, block, or re-status confirmed appointments on service deletion. [admin §2.5, §2.6]
- **WELL-FR-10** — TV catalog endpoints (`GET {salon|massage|private-training}/tv/services`) serve each vertical's active catalog **without authentication** for in-room browsing. [backend §1.3; tv §3.5]
- **WELL-FR-11 (gap, as-built)** — Service `PUT` and `PATCH …/toggle` routes have **no auth middleware** in all three verticals; `DELETE` has authMiddleware only (no role/permission gate). Only service `POST` is properly gated (STAFF/ADMIN/SUPER_ADMIN + write permission). [backend §1.3, §8.1]
- **WELL-FR-11a** — Service images follow the **platform media contract**: the client sends an S3 object `imageKey` (from the shared upload/gallery picker, with an optional save-to-gallery flag) and renders only the signed `imageUrl` returned by reads — signed URLs are never written back. Massage (and PT) use a discrete upload step (`POST …/services/upload` → `{imageKey, imageUrl}`) before create/update; salon uses the shared `ImageSelectionModal` directly. [admin §2.4 (gallery contract), §2.5, §2.6]

#### Service data dictionary (all three verticals)

| Field | Type / unit | Notes |
|---|---|---|
| `name` | string | Required everywhere; massage/PT ≤100 chars |
| `description` | string | Salon: min 10 words (admin validation); massage/PT ≤500 |
| `location` | string ≤200 | Where the service is delivered |
| `image` / `imageKey` | S3 key | Signed `imageUrl` on read |
| `duration` | integer minutes | Drives slot granularity (BR-1) |
| `price` | float, assumed USD | Display + payment-history aggregation only; >0 validated client-side |
| `cName` | staff username | **Owning staff member** — availability, visibility, calendar sync, notifications all key off this |
| `isActive` | boolean | Booking visibility toggle |
| `isDeleted` / `isDelete` (PT) | boolean | Soft delete; naming drift (WELL-FR-08) |

### 3.3 Per-day operating hours

- **WELL-FR-12** — Weekly hours are stored one row per (venue, day-of-week): `openTime`, `closeTime`, `isClosed`, unique on `(venueId, day)`. A day with no row, or `isClosed`, or missing open/close times produces a "closed" day with zero slots. [backend §1.1, §1.2 step 2]
- **WELL-FR-13** — Admin hours editing diverges per vertical (UI drift over one data model): Salon = per-day single-row updates; Massage = single-day upsert **plus bulk all-7-days upsert**; Private Training = dedicated Operating Hours tab with **draft-then-bulk-save** of all 7 days. [admin §2.5, §2.6, §4.4]
- **WELL-FR-14** — Admin can scope services/hours/appointments to one staff member via an admin-only designation-filtered staff dropdown (salon: designation "Salon Stylist", with an "All Staff" sentinel). Hours are fetched per staff `cName`. [admin §2.5]
- **WELL-FR-15 (gap, as-built)** — Weekly-schedule upsert routes (single and bulk) have **no auth** in all three verticals. [backend §1.3, §8.1]
- **WELL-FR-16 (gap, as-built)** — Staff have **no surface to manage their own hours or availability**: the staff app contains no hours/availability/calendar-link UI (the profile's `isGoogleLinked` flag is fetched and ignored). Hours management is admin-web-only. [staff §6]

### 3.4 Slot generation — the shared booking engine

For a service + date (single-day, or multi-day via `noOfDays`), bookable slots are derived by a fixed subtractive pipeline (`availableSlots.service.ts`, triplicated per vertical):

- **WELL-FR-17** — **Pipeline order (normative, as-built):**
  1. Service must exist and be `isActive` (salon/massage also require not soft-deleted; PT skips both filters — WELL-FR-08).
  2. Venue weekly hours for that weekday must exist and not be closed; otherwise the day returns "closed".
  3. **Base slots** generated back-to-back from openTime → closeTime at the service's `duration` granularity.
  4. **Meal blackout:** slots overlapping the facility's breakfast/lunch/dinner windows (`Config.meals`) are removed.
  5. **Same-service bookings:** slots already booked for this service (statuses PENDING/CONFIRMED; WAITLIST rows excluded) are removed.
  6. **Staff busy:** the owning staff member's cached Google Calendar busy windows are subtracted (§6).
  7. **Resident cross-venue conflicts:** slots overlapping the requesting resident's UnifiedSchedule rows of types SALON / MASSAGE / PT / CARE / REHAB with "upcoming" statuses are removed.
  8. **Past-time cutoff:** on the current day, slots whose start time has already passed are removed.
  [backend §1.2]
- **WELL-FR-18** — **Cross-venue conflict rule:** the blocking type set is exactly {SALON, MASSAGE, PT, CARE, REHAB}; "upcoming" statuses counted are {PENDING, CONFIRMED, Pending, Approved, REQUESTED} (mixed-case union across modules). ACTIVITY joins and TRANSPORTATION rows are **not** in the blocking set — an activity RSVP or a ride does not block a wellness booking (asymmetric: activity joins DO check wellness conflicts on the activity side). [backend §0.4, §5.2, §8.15]
- **WELL-FR-19** — Booked-range computation drift: massage and PT compute an existing booking's occupied range as `startTime + service.duration` (ignoring the stored `endTime`); salon uses the stored `endTime`. If a service's duration is edited after bookings exist, massage/PT availability and salon availability disagree. [backend §1.2 step 5, §8.6]
- **WELL-FR-20** — **Waitlist eligibility flag:** `canJoinWaitlist = (no remaining slots) AND (base slots nonempty)` — i.e. the venue is open and the service would fit, but every time is taken. In multi-day responses, a fully-booked day returns its **baseSlots** (the pre-booking template) so clients can offer waitlist times. [backend §1.2 step 9]
- **WELL-FR-20a** — **Request shapes:** clients request availability either for a single date or as a multi-day window (`noOfDays` — the TV app always asks for 7 days). Multi-day responses return, per day: open/closed state, remaining slots, `canJoinWaitlist`, and (when full) the baseSlots template. Slot identity is the (startTime, endTime) pair — there are no slot IDs; booking submits the literal times, which is why the engine re-validates membership at write (WELL-FR-21). [backend §1.2; tv §3.5]
- **WELL-FR-20b** — Because the resident-conflict step (pipeline step 7) personalizes availability, **slot responses are per-resident**, not cacheable venue-wide: the same service/date returns different slots for different residents. Pre-auth surfaces (TV before pairing) can only show catalogs, not slots. [backend §1.2.7; tv §3.5]

### 3.5 Booking

- **WELL-FR-21** — Normal booking: the requested (start, end) must be one of the currently available slots; the backend then runs a **second explicit UnifiedSchedule conflict check** (defense in depth against races) and rejects with "You already have another appointment at this time". On success the appointment is written with `status: CONFIRMED`. [backend §1.2]
- **WELL-FR-22** — Appointment record (all three verticals), with field semantics:

  | Field | Notes |
  |---|---|
  | `facilityId` | Tenant scope (schema-optional — see §7 known holes) |
  | `cName` | Resident username; virtual `resident` populated by join |
  | `serviceId`, `venueId` | Service booked; venue via resolution (BR-23) |
  | `date` | UTC start-of-day; clock times live in `startTime`/`endTime` |
  | `startTime` / `endTime` | "HH:mm" 24h strings; UI converts to 12h |
  | `status` | WELL-FR-33 enum |
  | `specialRequest` | Free text; rendered on staff day sheets (massage/PT cards) |
  | `isJoinedWaitList` | True when created via waitlist join |
  | `googleEventId`, `calendarSyncStatus` | §6 |
  | `createdByType`, `createdByCName` | Booking provenance (§3.10) |
  | `notes` (PT only) | Extra free-text field — schema drift vs salon/massage |

  Every save/update/delete syncs a UnifiedSchedule row via model hooks (§3.12). [backend §1.1, §0.4]
- **WELL-FR-23** — Booking surfaces, as wired today:
  - **SN resident app:** full flows for all three verticals (book / reschedule / cancel / my-appointments / history); massage and PT flows hidden from the Services hub but live (reachable via reschedule). [sn §3.6, §3.7]
  - **TV app:** book flow for all three — catalog card → QR-pair auth gate → 7-day slot dialog (`noOfDays: 7`, closed days filtered) → book → success/failure dialog. Booking surfaces in the TV unified schedule on next visit. [tv §3.5, §3.6]
  - **SL resident app (updated on staging):** all three verticals are now merchandised in the Services hub and fully API-backed — salon, massage, and private training each post real bookings via `POST /api/{vertical}/book-appointment`, fetch slots via `GET /api/{vertical}/:serviceId/available-slots`, and support `reschedule-appointment` / `cancel-appointment` and `my-appointments[/history]`. The prior "no API call / mock / unregistered screen" behavior is gone — the SL flows mirror the SN canonical flow against the same endpoints. [sl services hub + `src/services/services/{salon,massageTherapy,PrivateTraining}`]
  - **Admin web:** PT has an explicit admin-side booking action (`book-appointment`); salon and massage admin surfaces are confirmation/queue tools, not booking creators — bookings originate from resident-facing apps. [admin §2.5, §2.6]
- **WELL-FR-24** — Maximum booking horizon: a `maxSalonBookDays` config value exists and is fetched by the SL resident app but is **never enforced** in any client calendar; the SN app books via the slot API without a horizon check; TV is fixed at 7 days. No server-side horizon enforcement is documented. Target state should define and enforce a per-facility booking horizon server-side. [sl §3.5; tv §3.5]

**Reference booking flows (as-built):**

*SN resident app — salon (the canonical resident flow)* [sn §3.7]:
1. Services hub → Salon → catalog from the services API, multi-service selection with a detail modal.
2. Slot picker for the chosen service + date (per-resident availability, WELL-FR-20b).
3. Book via the dedicated book endpoint → success/failure confirmation; appointment appears in My Schedule (unified agenda) and Upcoming Appointments.
4. Reschedule and cancel via dedicated endpoints; history via my-appointments/history.

*TV app — all three verticals* [tv §3.5]:
1. Services tab → sub-category (Salon / Massage_Therapy / Private_Training_Session) selects the catalog; unknown categories fall back to salon.
2. "Book" on a card → auth gate: if no TV access token, the QR pairing `LoginDialog` pops; the resident scans/authorizes from their phone.
3. 7-day slot dialog (`noOfDays: 7`; closed days filtered; `canJoinWaitlist` carried but unused).
4. Book with `{date, startTime, endTime, <vertical>ServiceId}` → success/failure dialog + toast; booking surfaces in the TV's 7-day unified schedule on next entry (view-only there — no cancel/modify from TV).

*Staff on-behalf (backend contract — no dedicated client UI)* [backend §0.3]:
1. Staff submits the booking with a resident identifier (`residentCName` / legacy `cName` / `residentId`).
2. `resolveBookingContext` validates the staff designation against the module's allowlist (or admin bypass).
3. The appointment persists with `createdByType: STAFF` and the staff member's `createdByCName` — distinguishable from self-service bookings in any future reporting.

### 3.6 Waitlist

- **WELL-FR-25** — Joining the waitlist (`joinWaitlist: true`) is allowed **only when `canJoinWaitlist` is true** (no free slots that day). The requested time must match a baseSlot; the resident-conflict check is **skipped**; the appointment is stored with `status: WAITLIST`, `isJoinedWaitList: true`. [backend §1.2]
- **WELL-FR-26** — Waitlist promotion is **salon-only and staff/admin-driven**: `PATCH /appointments/:id/move` with a new time (and optionally a new date) transitions WAITLIST → CONFIRMED, guarded by "only waitlist appointments can be moved". Massage and PT have **no promotion endpoint** — a waitlisted massage/PT appointment can only be promoted via the unguarded generic status PATCH (WELL-FR-31). [backend §1.1]
- **WELL-FR-27** — Promotion surfaces:
  - Staff app (Salon Stylist): Waitlist tab → "Move to Slot" → same-day-only slot grid (slots fetched for the appointment's own date; no date picker) → move. The client falls back from PATCH to POST on 404/405 (environment API drift workaround). [staff §4.2]
  - Admin web: Waitlist tab → select appointment → fetch open slots for that service/date → pick slot → confirm (admin variant can also change date). [admin §2.5]
- **WELL-FR-28** — Waitlist client coverage: the SN salon flow and, **as of staging, the SL salon flow** both let a resident join the waitlist — when the slots API returns `canJoinWaitlist`, the SL `SalonBookAppointmentScreen` shows a waitlist checkbox and sends `joinWaitlist: true` on the shared book endpoint (`SalonBookAppointmentScreen/index.tsx:301,348,507-511`). The prior "SL checkbox never sent" gap is closed for salon. Remaining gap: the TV app still parses and maps `canJoinWaitlist` per day but implements **no waitlist UI**. [tv §3.5; sl `SalonBookAppointmentScreen`]
- **WELL-FR-29** — Waitlist-confirmed notification: on promotion, the resident receives a dedicated push (`SALON_WAITLIST_CONFIRMED`) and admins are notified. [backend §1.4]

### 3.7 Cancel & reschedule

- **WELL-FR-30** — Cancel: allowed for the **owning resident context only** ("You can only cancel your own appointment") — resident self, family for their linked resident, or staff/admin on-behalf within booking policy. Sets `status: CANCELLED`; fires notifications (resident + service staff + admins) and deletes/updates the Google Calendar event. As of staging **both** resident apps cancel via the dedicated `POST /api/{vertical}/cancel-appointment` routes (the prior SL empty-body-update abuse is gone). [backend §1.1, §1.4; sn §3.4; sl `src/services/services/{salon,massageTherapy,PrivateTraining}`]
- **WELL-FR-31** — Reschedule re-validates the new (date, start, end) against fresh availability and conflicts, **excluding the appointment's own UnifiedSchedule row** from the conflict set. Allowed source statuses diverge: salon permits PENDING/CONFIRMED/**WAITLIST**; massage/PT permit only CONFIRMED/PENDING. [backend §1.1, §1.2]
- **WELL-FR-32** — Reschedule surfaces: SN app per vertical (in the SN app massage/PT remain de-merchandised, reachable via reschedule from Upcoming Appointments / My Schedule); admin waitlist-promotion doubles as a reschedule for salon; staff-app move-to-slot is the stylist's reschedule. **SL app (updated on staging): reschedule now persists** via `POST /api/{vertical}/reschedule-appointment` for all three verticals. No client offers a cancellation-reason capture or reschedule-reason flow. [sn §3.6; admin §4.3; staff §4.2; sl `src/services/services/{salon,massageTherapy,PrivateTraining}`]

### 3.8 Appointment statuses & transitions

- **WELL-FR-33** — Status enum (all three): `PENDING | CONFIRMED | COMPLETED | WAITLIST | CANCELLED`. Schema default is PENDING, but the booking flow writes CONFIRMED or WAITLIST directly — PENDING is effectively vestigial in the wellness verticals (it still participates in slot-blocking and reminder status sets). [backend §1.1]

  | Status | Meaning (as-built) | Blocks slots? | Counts as "upcoming" for cross-venue conflicts? | Reminder eligible? |
  |---|---|---|---|---|
  | `PENDING` | Schema default; not written by booking flow; legacy/edge | Yes (same-service) | Yes | Yes (UnifiedSchedule row exists) |
  | `CONFIRMED` | Successful booking; the working state | Yes | Yes | Yes |
  | `WAITLIST` | Joined when day full; awaiting staff/admin promotion | **No** | No | No (not in upcoming set) |
  | `COMPLETED` | Set by cron when endTime passes (never by a human) | No | No | No |
  | `CANCELLED` | Terminal; set by owning resident context | No | No | No |
- **WELL-FR-34** — Transitions observed in code (normative as-built matrix):

  | From → To | Mechanism | Guard |
  |---|---|---|
  | (new) → CONFIRMED | book | slot availability + conflict check |
  | (new) → WAITLIST | book with joinWaitlist | `canJoinWaitlist` true; baseSlot match |
  | WAITLIST → CONFIRMED | salon `move` (staff/admin) | "only waitlist appointments can be moved" |
  | any → any (massage/PT only) | generic status PATCH (staff/admin) | **no transition matrix enforced** |
  | PENDING/CONFIRMED(/WAITLIST salon) → same status, new time | reschedule | fresh availability + conflicts |
  | owning context → CANCELLED | cancel | self/linked-resident only |
  | CONFIRMED → COMPLETED | auto-completion cron | endTime passed in facility TZ |

  Salon has **no generic status endpoint**; massage/PT's status PATCH can set anything (including reviving CANCELLED or un-completing). [backend §1.1]
- **WELL-FR-35** — **Auto-complete cron:** every 15 minutes (default), overdue CONFIRMED appointments whose `endTime` has passed in the facility timezone transition to COMPLETED — no grace period. Processing is paged (500), idempotent, and uses per-document updates specifically so UnifiedSchedule sync hooks fire. Consequence for the PRD: "completed" is a time-derived state, not a staff attestation — no client offers a manual complete action (staff app has none; salon admin has none; massage/PT admin could only via the generic PATCH). [backend §0.5, §1.1; staff §8.14]

### 3.9 Staff-side views

- **WELL-FR-36** — **Salon Stylist (staff app):** two-segment day sheet — "Today's" (CONFIRMED) and "Waitlist" (WAITLIST), each paginated 10/page, hard-pinned to today computed on-device (no look-ahead/back). Card: service + price, customer name, unit no, date, time. Waitlist cards carry "Move to Slot" (§3.6). No complete/cancel actions exist for stylists. [staff §4.2]
- **WELL-FR-37** — **Massage Therapist (staff app):** read-only "Today's Sessions" — CONFIRMED only, today only; card adds a conditional Special Request block. No actions, no pull-to-refresh (socket events trigger a full skeleton reload). [staff §4.3]
- **WELL-FR-38** — **Private Trainer (staff app):** identical read-only pattern, but the fetch sends **no status filter** (all statuses for today appear, including CANCELLED/WAITLIST) and pull-to-refresh is wired. [staff §4.4]
- **WELL-FR-39** — **Staff queue visibility (backend):** the staff/admin appointments list applies a service-ownership filter for staff-only callers — staff see **only appointments for services they own** (`serviceVisibilityFilter.cName = caller`); admins can filter by any staff `cName`. The "my assigned requests" route used by the staff app is gated by login only (any authenticated user, scoped to services they own). [backend §1.3]
- **WELL-FR-40** — Known backend quirk to preserve-or-fix: the salon appointments date filter applies `startTime >= now` to **every** future date, not just today — hiding earlier-time appointments on later days in multi-day staff/admin queries. [backend §8.13]
- **WELL-FR-40a** — **Real-time refresh for staff:** the staff app holds one Socket.io connection scoped to the Home screen, subscribing per designation to the vertical's upsert/delete events. Event payloads are discarded; any event re-fetches page 1 of the active view (salon: silent per-tab; massage: full skeleton flash; PT: refresh-control mode). There are no client→server emits — the socket is consume-only. [staff §5.1]
- **WELL-FR-40b** — **Admin appointment queues:** Confirmed and Waitlist tabs (salon), independently paginated and staff-filterable; massage adds a status PATCH and a `confirmedCount` meta; PT adds admin booking. Admin queues are date-unrestricted (unlike the staff app's today-pin) and are the only surface where past appointments can be reviewed per vertical. [admin §2.5, §2.6]

### 3.10 Per-facility booking policy (`resolveBookingContext`)

- **WELL-FR-41** — All booking/viewing/cancel routes for the three verticals run through a central booking-context middleware keyed by module name (SalonAppointment, MassageAppointment, PrivateTrainingAppointment in facility config `bookingPermission`):
  - **RESIDENT** — always allowed, self-service only; any resident identifier in the request body is ignored (spoof guard).
  - **FAMILY_MEMBER** — allowed only if `bookingPermission[module].isFamilyMemberAllowed === true`; always acts for the linked resident.
  - **STAFF** — must supply a resident identifier; allowed only if their free-text `Staff.designation` appears in the module's `staffDesignationAllowed` list (expanded by designation groups, e.g. `rehab`, `supervision`).
  - **ADMIN/SUPER_ADMIN** — must supply the resident identifier and the module must be configured at all; designation check bypassed.
  - Output `{createdByType, createdByCName, residentCName}` is persisted on every created appointment — full provenance of on-behalf bookings. [backend §0.3]
- **WELL-FR-42** — A parallel **case-manager route family** (`/case-manager/resident/:residentId/...`, salon only among the three verticals) lets **any STAFF** book/view for a resident by `_id`, bypassing `resolveBookingContext` and its designation allowlist entirely — a deliberate but policy-inconsistent escape hatch. [backend §0.3, §1.3]
- **WELL-FR-43** — There is **no admin UI** for editing `bookingPermission` (family opt-in, designation allowlists) per module — the policy is config-data with no management surface. Target state needs a booking-policy settings page. [admin: absent; backend §0.3]

### 3.11 Resident-facing history & spend

- **WELL-FR-44** — Residents (SN app) get my-appointments + history per vertical, and a cross-module day agenda (UnifiedSchedule) where SALON/MASSAGE/PT items open a detail sheet with cancel. [sn §3.4, §3.7]
- **WELL-FR-45** — Admin resident **Payment History** aggregates wellness spend: per-month line items typed `SALON | MASSAGE | PT` (among TRANSPORT/MEAL/WELLNESS) with status, date/time, location, provider, amount. Note: no payment processing exists anywhere in the platform — these are derived charges, not transactions. [admin §2.1]
- **WELL-FR-46** — The admin dashboard counts pending wellness requests (permission-gated per viewer) and merges salon + PT + massage bookings into an "Upcoming Appointments" feed (max 8) with click-through to the owning module. [admin §2.15]

### 3.12 UnifiedSchedule integration (the cross-module calendar spine)

- **WELL-FR-47** — Every wellness appointment write (save / update / delete) synchronizes exactly one denormalized `UnifiedSchedule` row (`scheduleType: SALON | MASSAGE | PT`) via Mongoose post-hooks. The sync contract requires controllers to use `{ new: true }` on findOneAndUpdate or the hook writes stale data — a documented backend contract that any new endpoint must honor. [backend §0.4]
- **WELL-FR-48** — The UnifiedSchedule row is what powers: cross-venue conflict checking (BR-5), the reminder cron's scan window, the resident's unified day agenda (SN app My Schedule), the TV 7-day schedule, the admin "recent activity" feed, and the upcoming-appointments counts on resident dashboards. Wellness appointments therefore have **two representations** that must not drift: the vertical's own collection (source of truth for status/times) and the UnifiedSchedule projection (source of truth for cross-module queries). [backend §0.4; sn §3.4; tv §3.6; admin §2.15]
- **WELL-FR-49** — Reschedule conflict checks exclude the appointment's own UnifiedSchedule row; the auto-completion cron deliberately uses per-document updates so these hooks keep firing at scale (paged 500). [backend §1.2, §0.5]

---

## 4. Business rules & policies (consolidated)

| # | Rule | Value / behavior | Evidence |
|---|---|---|---|
| BR-1 | Slot granularity | Back-to-back slots at service `duration` minutes from venue open → close | backend §1.2.3 |
| BR-2 | Meal blackout | Slots overlapping facility breakfast/lunch/dinner windows removed | backend §1.2.4 |
| BR-3 | Same-service occupancy | PENDING + CONFIRMED block; WAITLIST does not block | backend §1.2.5 |
| BR-4 | Staff busy subtraction | Owning staff's cached Google busy windows subtract slots | backend §1.2.6, §0.6 |
| BR-5 | Cross-venue blocking set | SALON, MASSAGE, PT, CARE, REHAB block each other per resident; statuses {PENDING, CONFIRMED, Pending, Approved, REQUESTED} | backend §0.4 |
| BR-6 | One-way activity conflict | Activity joins check wellness conflicts; wellness bookings ignore ACTIVITY rows | backend §0.4, §8.15 |
| BR-7 | Past-time cutoff | Same-day slots with start < now removed | backend §1.2.8 |
| BR-8 | Waitlist eligibility | Only when venue open + zero free slots; time must match a baseSlot; conflict check skipped | backend §1.2.9, §1.2 |
| BR-9 | Booking double-check | Availability membership + second UnifiedSchedule conflict check at write | backend §1.2 |
| BR-10 | Cancel ownership | Only the owning resident context may cancel | backend §1.1 |
| BR-11 | Reschedule source statuses | Salon: PENDING/CONFIRMED/WAITLIST; Massage/PT: PENDING/CONFIRMED | backend §1.1 |
| BR-12 | Waitlist promotion | Salon only, staff/admin write permission, from WAITLIST only | backend §1.1 |
| BR-13 | Auto-completion | CONFIRMED → COMPLETED once endTime passes in facility TZ; cron every 15 min; no grace | backend §0.5 |
| BR-14 | Family booking opt-in | Per facility per module: `bookingPermission[module].isFamilyMemberAllowed` | backend §0.3 |
| BR-15 | Staff on-behalf allowlist | `Staff.designation` ∈ `staffDesignationAllowed` (+ designation groups); admins bypass | backend §0.3 |
| BR-16 | Booking provenance | `createdByType`/`createdByCName` persisted on every appointment | backend §0.3 |
| BR-17 | Staff queue visibility | Staff see only appointments for services they own | backend §1.3 |
| BR-18 | Admin staff-assignment rule | Service owner `cName` required for ADMIN; non-admins locked to self during edit | admin §2.5 |
| BR-19 | Deletion impact | Admin two-phase delete: show CONFIRMED appointments for the service before delete (client-side only) | admin §2.5 |
| BR-20 | Facility timezone caveat | Cron TZ and slot math use the process-wide `FACILITY_TIMEZONE` env, not per-facility `Config.timeZone` — multi-TZ deployments unsupported | backend §8.16 |
| BR-21 | Slot identity | Slots have no IDs; booking submits literal (startTime, endTime) which must match a currently-available slot | backend §1.2 |
| BR-22 | Per-resident availability | Slot responses are personalized by the resident's calendar — not cacheable venue-wide | backend §1.2.7 |
| BR-23 | Venue fallback | `venueId` resolves from the service's venue or the facility default at booking time | backend §1.2 |
| BR-24 | Shared reminder config | SALON, MASSAGE, PT all map to NotificationConfig module `SALON` / event `APPOINTMENT_REMINDER` — one toggle/offset set for all three | backend §0.5 |
| BR-25 | Reminder dedupe | Atomic unique claim on `(scheduleId, offsetMinutes)` prevents duplicate sends across cron ticks | backend §0.5 |
| BR-26 | Media contract | Write S3 `imageKey`, read signed `imageUrl`; never persist signed URLs | admin §2.4 |

---

## 5. Notifications & real-time behavior

**Push (FCM) lifecycle-event matrix** [backend §0.5, §1.4]:

| Event | Resident | Family (linked) | Service-owner staff | Permission-group staff | Admins |
|---|---|---|---|---|---|
| Booking created (CONFIRMED) | confirmation push | confirmation push | "new appointment request" push | — | broadcast notify |
| Waitlist joined | confirmation (waitlist title) | confirmation | "new waitlist request" push (distinct title) | — | broadcast notify |
| Waitlist → confirmed | `SALON_WAITLIST_CONFIRMED` push | — | — | — | notify |
| Cancel | push | — | push | — | notify |
| Reschedule | push | — | push | — | notify |
| Appointment reminder (cron) | push | push | — | by permission map: SALON→`Salon`, MASSAGE→`Services`, PT→`Rehab` | — |

Notes:
- Wellness uses **assigned-staff** fan-out (`notifyCreationForAssignedStaff` — the service owner), unlike Dining/Housekeeping which fan out to all staff holding a page permission. Salon has the richest notification set; massage/PT are lighter copies. [backend §1.4]
- Every send is logged to `NotificationHistory` (recipient type, schedule type/id, title, body) — an audit trail the PRD can build delivery reporting on. [backend §0.5]
- **Reach caveat (updated on staging):** the SL app now harvests an FCM token at splash and sends it on every request (`pushToken` / `x-fcm-token` headers) plus renders foreground notifications via `@notifee/react-native`, so SL residents are reachable (the prior "token never registered" gap is closed for foreground delivery; background/terminated tap routing is still missing — see §5 gaps). Family members receive pushes only if they have a device session with a registered token.

**Reminder cron** [backend §0.5]:

- Runs every minute; collects per-facility offsets from `NotificationConfig.modules[].events[].scheduled.offsets` (env default 15 min) and scans UnifiedSchedule rows whose start falls in each window.
- Module mapping quirk: SALON, MASSAGE, and PT all map to NotificationConfig module **`SALON`**, event `APPOINTMENT_REMINDER` — the three verticals share one reminder toggle/offset set.
- Staff reminder recipients are resolved by page permission: SALON → `Salon`, MASSAGE → **`Services`**, PT → **`Rehab`** — three different permission keys across one flow.
- Duplicate-send protection via atomic unique insert on `(scheduleId, offsetMinutes)`.

**Socket events** [backend §0.4; staff §5.1]:

- UnifiedSchedule sync emits upsert/delete socket events on every appointment write.
- The staff app subscribes per designation: `mobile-SalonAppointment-request-upserted/-deleted`, `mobile-MassageAppointment-…`, `mobile-PrivateTrainingAppointment-…`. Payloads are ignored — any event bumps a refresh counter and the active view re-fetches page 1 (salon silently; massage with a full skeleton flash — known UX drift).
- The admin web app does not consume wellness socket events (its sockets serve announcements/notifications/chat); queues refresh via React Query only.

**Client notification handling gaps** [staff §5.2; sl Splash + Api interceptor + `services/Notifications/foregroundNotifications.ts`; sn §4]: the staff app displays pushes but has no typed handling or deep links; the SL resident app now harvests an FCM token (Splash), sends it on every request, and renders foreground pushes via notifee — but background/terminated tap-through routing is still absent; the SN app registers tokens and displays foreground pushes generically without tap-through routing.

---

## 6. Integrations (Google Calendar)

**Linking flow:**
1. Staff member (or admin on their session) opens admin web → Settings → Integrations → "Link Google Calendar".
2. Full-page OAuth redirect (`/auth/google/url?cognitoUser={cName}`); return trip carries `?sync=success|retry` URL params which are toasted and cleared.
3. The Google refresh token is stored on the staff record; `isGoogleLinked` flips true on the profile. [admin §2.3, §2.17]
4. The staff app neither exposes nor consumes the link state (`isGoogleLinked` fetched, never read) — linking is web-only. [staff §1.4]

**Sync behavior matrix:**

| Trigger | Direction | Effect |
|---|---|---|
| Appointment booked | SAL → Google | CREATE event on service-owner's primary calendar; `googleEventId` + `calendarSyncStatus` recorded |
| Appointment rescheduled | SAL → Google | UPDATE the event |
| Appointment cancelled | SAL → Google | DELETE the event |
| Staff calendar changes (webhook) | Google → SAL | Refresh `cachedCalendarBusySlots` on the staff record |
| Slot generation | Google → SAL (read) | Cached busy windows subtract availability (BR-4) |
**Details & semantics:**

- Outbound sync is **fire-and-forget async**: booking success never waits on or fails with Google. Per-appointment outcome is recorded as `calendarSyncStatus: PENDING | SYNCED | FAILED` alongside `googleEventId`. Events carry extended properties (model type, appointment id, facility id) so webhook reconciliation can map calendar events back to appointments. [backend §0.6]
- Inbound availability means **an externally-created calendar event blocks resident bookings** for that staff member's services — the staff member's personal calendar is, in effect, their availability-blocking tool (and today their *only* self-service one, given WELL-FR-16). Webhook-driven refresh keeps the cache current (`GOOGLE_CALENDAR_WEBHOOK_BASE_URL`). [backend §0.6, §1.2.6; staff §6]
- **Unlinked staff degrade silently**: if the service owner has no Google link, the busy-subtraction step contributes nothing — slots reflect venue hours and bookings only. No client warns that availability is calendar-blind for unlinked staff.
- **Failure semantics gap**: FAILED sync status is recorded but no retry/repair surface exists in any client, and `calendarSyncStatus` is rendered nowhere. The PRD target state should define a re-sync mechanism and surface sync health to admins.

---

## 7. Permissions & access control

| Layer | Mechanism | Wellness specifics |
|---|---|---|
| Role | Cognito groups RESIDENT / FAMILY_MEMBER / STAFF / ADMIN / SUPER_ADMIN; TV JWT | Family auth rewrites identity to the linked resident; TV requests carry the paired resident's identity. [backend §0.1] |
| Page permission (staff) | `Staff.accessPermissions` read/write per page; middlewares **no-op for non-STAFF** (admins pass on role) | Salon routes → `Salon` permission; Massage routes → `Massage Therapy`; **PT routes → `Rehab`**; massage reminder staff → `Services`. Three different keys across one vertical family. [backend §1.3, §0.5] |
| Booking policy | `resolveBookingContext` per module (§3.10) | Family opt-in + staff designation allowlist + admin bypass; provenance persisted. |
| Admin web gating | `usePageAccess` per view (`salon-settings`, `salon-appointments`, `Massage-Therapy`, `private-training`); read-only users see pages with disabled writes + toast | Facility-pages API (Filter 1) decides which wellness pages exist per facility; staff grants (Filter 2) refine. [admin §1.3, App.B] |
| Staff app gating | **Designation-only** — `accessPermissions` fetched but never read | Salon Stylist / Massage Therapist / Private Trainer designations get views; 11 other designations get an empty screen. [staff §1.2, §1.4] |
| Known holes (as-built) | No-auth routes: venue CRUD, schedule upsert (incl. bulk), service update/toggle, TV catalog (intentional); service DELETE auth-only; case-manager routes role-only; `facilityMiddleware` missing-header check is dead code, so unscoped cross-tenant queries are possible | backend §1.3, §8.1, §8.2 |

Target-state recommendation for the PRD: collapse the three permission keys to one wellness scheme (or one key per vertical, consistently), put all admin-panel CRUD routes behind ADMIN/write-permission middleware, and fix tenant-header enforcement — these are the highest-leverage security items in the module.

---

## 8. Product-split notes

- **One backend serves both products.** All three verticals are facility-type-agnostic in backend and admin code; exposure is governed by the facility-pages config (admin Filter 1) and per-app merchandising, not by code branches. [admin §1.3, §3]
- **Private Training sits on the AL side of the split**: it appears under the admin **Rehab** parent in the AL sub-page set (`private-training`) and is permission-keyed `Rehab`; the SNF rehab suite (`rehab/appointments`, therapist availability) is a separate engine. A unified product must decide whether PT remains a wellness vertical or merges into rehab. [admin §1.2, §2.6; backend §7.2]
- **Resident-app split is per-repo, not per-flag**: as of staging both resident apps have a complete, live wellness implementation against the same endpoints — **the SL app now merchandises and books all three verticals** (salon, massage, private training), reaching the parity that was the SN → SL direction. (Note the SN app still keeps massage/PT *de-merchandised* — tiles commented out — reachable only via reschedule; the SL app merchandises all three.) The remaining divergence is cosmetic, not a fork to preserve. [sl Services hub + `src/services/services/*`; sn `ServicesScreen/index.tsx:72-81`]
- **TV app is shared** and books all three verticals via TV-token identity; its catalog routes are deliberately unauthenticated for pre-pairing browsing. [tv §3.5]
- **Per-resident `careType`** (assisted_living / memory_care / independent_living / skilled_nursing) exists on residents and surfaces in payloads, but no wellness behavior branches on it today. [backend §7.1]

---

## 9. Observations & candidate gaps

Ranked; each with evidence. "Triplication" items are candidates for the PRD's single parameterized service-vertical module (admin §4.6.3 names this explicitly).

1. **Triplication drift (backend slot engine).** The engine is copy-pasted three times with semantic differences: PT service lookup skips facility + soft-delete filters; massage/PT compute booked ranges from duration vs salon's stored endTime; PT's flag is `isDelete` vs `isDeleted`. One engine, parameterized, removes three classes of bug. [backend §8.6]
2. **Triplication drift (lifecycle).** Reschedule-allowed statuses differ (salon includes WAITLIST); only salon has a guarded waitlist-promotion endpoint; massage/PT expose a free-form status PATCH with **no transition matrix** (any state to any state); salon has no status endpoint at all. The PRD should define one status machine and one promotion flow for all three. [backend §1.1, §8.7]
3. **Triplication drift (admin UI).** Three near-identical admin implementations diverge in capability: only PT has draft-mode hours; only massage has bulk schedule upsert *and* a status PATCH; PT slot lookup **reuses the salon slots endpoint** (cross-module coupling that breaks if salon-specific logic is added). [admin §2.6, §4.4]
4. **Permission-key fragmentation.** PT gated by `Rehab`, massage queue by `Massage Therapy`, massage reminders by `Services`, salon by `Salon` — four keys across three verticals, with the staff app ignoring permissions entirely (designation-only). [backend §1.3 quirk note, §0.5; staff §1.4]
5. **Unauthenticated admin-surface routes.** Venue CRUD, weekly-schedule upsert, service update/toggle have no auth in all three verticals — combined with the dead facility-header check, these are writable cross-tenant. Blocker-grade target-state fix. [backend §8.1, §8.2]
6. **SL resident app booking — RESOLVED on staging.** Salon, massage, and private training are now merchandised in the Services hub and fully API-backed (real `book-appointment` / `available-slots` / `reschedule-appointment` / `cancel-appointment` / `my-appointments[/history]`); cancel uses the dedicated route, not the empty-body update; massage/PT are no longer mock/dead-end. **Residual gaps:** a per-facility booking horizon (`maxSalonBookDays`) is still not enforced anywhere; the TV surface still parses `canJoinWaitlist` without a waitlist UI. (The SL salon surface now *does* offer waitlist join — see WELL-FR-28.) The SL and SN resident apps are now at near-parity for wellness booking. [sl Services hub + `src/services/services/{salon,massageTherapy,PrivateTraining}`]
7. **Read-only staff views / missing completion attestation.** Stylists cannot complete or cancel; massage/PT staff have no actions at all; completion is purely time-derived (cron). If the business needs no-show tracking or service-rendered confirmation, it is net-new. [staff §4.2–4.4, §8.14; backend §0.5]
8. **Staff cannot manage their own availability.** No hours, time-blocking, or Google-link flow exists in the staff app; everything routes through admin web + external Google Calendar. [staff §6]
9. **Waitlist UX is incomplete on the TV surface only.** As of staging both resident apps (SN and SL salon) can join the waitlist; the TV still parses the flag without UI. Waitlist behavior beyond join (notify on opening? auto-promote order? expiry?) is still undefined — there is no queue ordering or position concept, and promotion remains salon-only / manual. [tv §3.5; sl `SalonBookAppointmentScreen`; backend §1.2]
10. **No booking-policy admin UI.** `bookingPermission` (family opt-in, designation allowlists) — the central authorization config — has no management surface; same for meal windows that drive slot blackouts. [backend §0.3; admin §2.4 note]
11. **Hidden-but-live booking paths (SN).** Massage/PT de-merchandised but reachable via reschedule — inconsistent UX if legacy appointments exist; decide ship-or-remove per vertical. [sn §7.3]
12. **Backend quirks to schedule fixes for:** future-date `startTime >= now` filter hides appointments (backend §8.13); the case-manager bypass routes (backend §0.3); process-wide `FACILITY_TIMEZONE` blocking multi-TZ tenancy (backend §8.16); salon "Store Details" display card commented out in admin (admin §4.3); staff-app move-to-slot PATCH→POST fallback indicating cross-environment API drift (staff §8.10).
13. **No pricing/transaction layer.** Prices are display + payment-history aggregation only (assumed USD floats); no charge capture, invoicing, or currency handling anywhere in the verticals. If billing is a product goal, it is net-new. [admin §2.1, §4.5]
14. **Status-vocabulary normalization debt.** Wellness uses UPPER_SNAKE statuses while Transport uses Title Case and the cross-venue "upcoming" set is a mixed-case union (BR-5); admin hooks normalize to lowercase UI values; the SL app even mixes `Canceled`/`CANCELLED` spellings. A unified API needs a single status-normalization story. [admin §4.4; backend §0.4; sl §6.5]
15. **No capacity dimension.** A venue/staff member can serve exactly one appointment per slot per service; there is no multi-chair/multi-station concept, no concurrent-capacity field, and no per-staff parallelism — capacity is implied by the single owning `cName`. If facilities need multi-station salons, the data model needs a capacity or resource layer. [backend §1.1, §1.2]
16. **Reminder/notification config is shared and partially editable.** All three verticals share one reminder event (BR-24) and `NotificationConfig` offsets have no admin UI in scope; admin Settings notification toggles are decorative (localStorage only). [backend §0.5; admin §2.17]

### Open questions for product

1. Should waitlist promotion become automatic (FIFO on cancellation) or stay staff-curated? Today there is no queue order, no position visibility, and no expiry.
2. Is COMPLETED meant to assert service delivery (needs staff attestation + no-show state) or just "time passed" (current cron semantics)?
3. Does Private Training stay a wellness vertical (own permission key) or merge into Rehab — and if it stays, does its permission key migrate off `Rehab`?
4. The SL resident app has now reached (and on merchandising, exceeded) SN parity for wellness booking on staging — is the consolidation direction still SN-as-single-app, or is the SL app the canonical resident surface going forward?
5. Are massage/PT meant to relaunch on resident surfaces (currently de-merchandised in SN, mock in SL), or is salon the only resident-bookable vertical near-term?
6. Per-facility booking horizon, cancellation cutoffs, and no-show fees: none exist — are they required for v1 of the unified product?
