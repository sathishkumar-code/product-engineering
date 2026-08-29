# PRD — Senior Living

> **Status:** As-built v1.4 (reverse-engineered from code; baseline 2026-06-12, refreshed 2026-06-17 against production HEAD `981704e9`, refreshed 2026-06-21 against staging backend `62de4747`; updated 2026-07-03 against **production** backend `465e88fb` / admin `59d22ea`; refreshed 2026-07-12 against **production** backend `e075f578` / admin `324840a` / staffapp `master@4aa3849` / SL resident app `master@3af3c3e` — see per-repo architecture-doc delta reviews under `docs/reviews/2026-07-12/`). Changelog in [README](./README.md#changelog).
> **Scope basis:** The codebase is the only source of truth. This PRD documents current behavior as the requirement baseline; intended-but-incomplete functionality is flagged in per-module "Observations & candidate gaps" sections, not stated as requirements.
> **Companion PRD:** [Skilled Nursing](./prd-skilled-nursing.md) — shares this platform and most modules.
> **Personas:** [personas-and-roles.md](./personas-and-roles.md)
> **Module sub-docs:** [`modules/`](./modules/) — referenced throughout; this parent doc does not duplicate their detail.

---

## 1. Product overview

**Senior Living** is a resident-experience and facility-operations platform for assisted-living, independent-living, and memory-care communities. It connects four user groups around one facility-scoped backend:

- **Residents** book personal services (salon, massage, private training), request rides and housekeeping, browse dining menus, join activities, and receive announcements — from a mobile app and an in-room Android TV.
- **Family members** act on behalf of their linked resident (where the facility's booking policy permits) and request family meals.
- **Staff** receive and execute the resulting work — rides, appointments, housekeeping — through a designation-aware mobile app with real-time updates.
- **Facility admins** configure everything and run daily operations through a web dashboard: residents, staff and permissions, service catalogs and hours, menus and specials, transport rules and pricing, activities and attendance, announcements, and assisted-living therapy scheduling.

The platform is **multi-tenant by facility** (`x-facility-id` on every request), with per-facility configuration controlling which modules are visible, which personas may book each module, and notification behavior.

### Differentiation from Skilled Nursing

Senior Living and Skilled Nursing share one backend, one admin web app, one staff app, and one TV app. The differentiators are:

| Lever | Senior Living value |
|---|---|
| Facility type (`facilityType`) | `ASSISTED_LIVING` (admin UI switches table columns, labels, care-type filters) |
| Resident `careType` | `assisted_living`, `independent_living`, `memory_care` |
| Therapy model | **AL path**: four fixed therapy types (Physical Therapy, Cognitive Evaluation, Rehab Evaluation, Outside Agency) scheduled by admins via the `/care` engine — see [therapy-rehab.md §3A](./modules/therapy-rehab.md) |
| Resident mobile app | Dedicated binary: `senior_living_reactnative` — a **separate app** from the Skilled Nursing resident app (`senior_living_skillednursing_resident`); see the companion PRD's §1 for the client-distinctness note |
| Clinical modules | Not part of this product (medications/labs/IDT/care conferences/referrals-with-e-sign are Skilled Nursing scope; internal chat is a platform capability not surfaced in this product's resident app — see §7 item 2) |

There is **no single product flag** in the backend; the product boundary is emergent from facility type, resident care types, page-visibility config, and which client binary is shipped. This is a central finding for the refactoring phase.

---

## 2. Application surfaces

| Surface | Repo | Role in this product |
|---|---|---|
| Backend API | `senior_living_backend` | All business logic; facility-scoped; Express/TS + MongoDB; Cognito auth; Socket.io; FCM; crons |
| Admin web | `senior_living_admin` | Facility operations console (assisted-living mode) |
| Resident app | `senior_living_reactnative` | Resident self-service (see maturity caveat, §6) |
| Staff app | `senior_living_staffapp` | Designation-based work execution (driver, stylist, massage, trainer, housekeeping, maintenance) — and, since staging, real-time chat for every designation (see [messaging-chat.md](./modules/messaging-chat.md)) |
| TV app | `senior_living_tvapp` | In-room resident experience: live TV, dining, services, schedule, alerts, photos, music |

---

## 3. Personas

See [personas-and-roles.md](./personas-and-roles.md). For Senior Living specifically:

- Residents carry AL/IL/MC care types; the admin residents table shows Name / Unit / Care Type / Status / Family count.
- Staff designations with live mobile experiences: Transport Driver, Salon Stylist, Massage Therapist, Private Trainer, Housekeeping Staff, Maintenance Staff.
- Family members participate via the facility booking policy (per-module opt-in) and family meal requests.

---

## 4. Module map

Each module has a functional-spec sub-document. "SL scope" notes what applies to this product.

| # | Module | Sub-doc | SL scope |
|---|---|---|---|
| 1 | Platform foundation | [modules/platform-foundation.md](./modules/platform-foundation.md) | Full — multi-tenancy, identity/auth (Cognito+MFA), resident & family lifecycle, designations, access permissions, app versions, welcome-SMS credential delivery |
| 2 | Dining | [modules/dining.md](./modules/dining.md) | Full — menus, specials, menu library, diet plans, family meal requests |
| 3 | Wellness services | [modules/wellness-services.md](./modules/wellness-services.md) | Full — salon, massage, private training; shared slot/booking engine; waitlists |
| 4 | Transportation | [modules/transportation.md](./modules/transportation.md) | Full — rules, complimentary distance, request/approval/driver lifecycle, prebooking conflict detection, admin schedule/calendar view, staff-app edit + detail views |
| 5 | Housekeeping & maintenance | [modules/housekeeping-maintenance.md](./modules/housekeeping-maintenance.md) | Full — 4 request types, staff queues, TELS sync |
| 6 | Activities & attendance | [modules/activities-attendance.md](./modules/activities-attendance.md) | Full — activity CRUD/recurrence, RSVP, attendance, analytics, calendar PDF |
| 7 | Announcements & notifications | [modules/announcements-notifications.md](./modules/announcements-notifications.md) | Full — announcements, gallery, notification platform (FCM/socket/feed, reminder crons) |
| 8 | Dashboard & reporting | [modules/dashboard-reporting.md](./modules/dashboard-reporting.md) | Full — admin dashboard, payment history, settings, OAuth links |
| 9 | TV experience | [modules/tv-experience.md](./modules/tv-experience.md) | Full — provisioning, QR pairing, all TV sections |
| 10 | Therapy & rehab | [modules/therapy-rehab.md](./modules/therapy-rehab.md) | **§3A AL path only** — fixed therapy types via `/care`; outside-agency appointments; Therapy Evaluations view |
| 11 | Messaging / chat | [modules/messaging-chat.md](./modules/messaging-chat.md) | Platform capability exists (backend, admin web, and — since staging — the staff app), **not surfaced** in the SL resident app today. Chat underwent significant production hardening (delivery/read receipt reliability, group quorum, message delete, an installable admin PWA) in the 2026-07 window; the 2026-08 window added edit/forward/pin and, backend-side, a **fully non-destructive retention policy** (HIPAA/CA 7-year — corrects prior "eager S3 deletion" documentation) plus a new automated-messaging identity (`ChatSystemUser`) whose first consumer is Transportation, not an SL-facing module yet — see the module doc §1/§9 for detail |
| 12 | Clinical records | [modules/clinical-records.md](./modules/clinical-records.md) | **Out of product** — SL resident app contains mock health screens only (flagged as gap/intent signal) |
| 13 | Care coordination | [modules/care-coordination.md](./modules/care-coordination.md) | Out of product (Skilled Nursing) |

---

## 5. Product-level requirements (cross-cutting)

These hold across all modules; module docs carry the detail.

- **SL-PR-01 — Facility scoping.** Every read and write is scoped to the facility identified by `x-facility-id`. Per-facility `Config` controls page visibility (`accessPages`), per-module booking permissions (which personas may book; family opt-in; staff designation allowlists), meal windows, transport pricing parameters, notification configuration, and inactivity timeouts.
- **SL-PR-02 — Authentication.** All human personas authenticate via AWS Cognito (`USER_PASSWORD_AUTH`) with MFA (TOTP and/or SMS), forced password change on first login, and self-service password reset — both Cognito-native and a custom phone/email **OTP reset** flow with rate limiting and attempt tracking (PLAT-FR-13/13a/13b). Initial-credential and resend SMS now suppress Cognito's default invite copy in favor of a custom welcome message carrying a role-specific app-download link (PLAT-FR-14c). TV devices use the custom pairing-token model.
- **SL-PR-03 — Booking integrity.** All bookable modules (salon, massage, private training, AL therapy) resolve through a shared slot pipeline: venue hours → meal blackouts → existing same-service bookings → staff Google Calendar busy → resident cross-venue conflicts (UnifiedSchedule) → past-time cutoff. Full venues offer waitlisting.
- **SL-PR-04 — Unified resident agenda.** Resident bookings, activities, and appointments aggregate into a UnifiedSchedule that powers the resident "my schedule" views, cross-venue conflict checks, the admin recent-activity feed, and monthly payment history.
- **SL-PR-05 — Notification coverage.** State changes in resident-facing modules fan out via FCM push, socket events to the staff app, and a persistent notification feed, governed by per-facility `NotificationConfig` (~40 event types, default reminder offsets 20m/1h/1d) with send-dedup.
- **SL-PR-06 — Permission-gated administration.** Admin-web pages are gated by facility page visibility and per-staff read/write grants; every write path in the UI enforces read-only mode with explicit feedback.
- **SL-PR-07 — Auditability of money-bearing events.** Priced events (transport rides, service appointments, family meals) appear in the resident's monthly payment history with amount, status, and source module.

---

## 6. Surface maturity (load-bearing finding)

The as-built maturity of this product's surfaces is **uneven**, and any roadmap or refactor must account for it:

| Surface | Maturity | Evidence highlights |
|---|---|---|
| Backend | Live, broad | All SL modules implemented server-side; 2026-07 window added chat production hardening, an automated PCC resident-onboarding rewrite (primarily Skilled Nursing-facing), welcome-SMS credential delivery, and a new (unauthenticated — see companion PRD) manual PCC-sync tool. **2026-08 window (Skilled-Nursing-facing, noted here for platform-wide backend awareness):** chat v2 (edit/forward/pin, `ChatSystemUser` automation, non-destructive retention), resident/family phone-uniqueness enforcement removed platform-wide (not SN-specific — applies to any resident record regardless of product), a legacy-Cognito-pool migration path added to password reset, and a new **Critical** PCC-sync auth gap (G-30, two more endpoints gated only by a hardcoded shared secret) — see companion PRD §7 item 11 |
| Admin web | Live, widest client | All modules operable; some dead pages (Reports) and decorative settings; chat became an installable PWA ("Shashi Messaging") in the 2026-07 window |
| Staff app | Live, **three-flow model** | Operational designations get LEGACY task queues; clinical CM/Doctor/DON/SW now get the MIGRATED Skilled Nursing experience — as of 2026-08-21 grown from a 3-tab to an **up-to-5-tab** bar with new Transport and Documents/Scan tabs (transportation.md TRN-FR-25; personas-and-roles.md §2.2); `Message`-group roles get chat-only home. Chat is live for **every** flow with delivery/read-receipt reliability, message delete, group-membership events, a custom notification sound, list pagination, and (2026-08-21) forward/drafts/pin/mark-read/leave-group/message-info (messaging-chat.md MSG-FR-40/41/42) — offline sync was built but never merged, chat remains online-only. Twilio calling now feeds a Secure Call review/approval workflow (secure-call.md); inactivity logout is now layered under a new device-level **biometric App Lock** (platform-foundation.md PLAT-FR-75); sign-in gained phone-or-email + a Cognito user-pool migrator + 3-channel MFA (PLAT-FR-72b). A facility-configured (IANA) timezone layer and a Terms & Conditions acceptance step remain from auth. Massage/PT views still read-only; a `DietitianView` is built but unreachable; JS-bundle OTA updates (Expo/EAS) can now reach production without app-store review, with no rollback runbook yet. |
| TV app | Live | Calls tab mock; no resident sign-out; pull-only refresh |
| **Resident app (SL)** | **Substantially live (staging)** | Live: dining, announcements, **salon / massage / private-training booking (now fully API-backed)**, the three Health care flows (Physical Therapy, Cognitive Evaluation, Outside Agency) via `/care`, **MySchedule + a new Activities screen** against `/api/unified-schedule` + `/api/schedules`, Notifications, resident directory, upcoming appointments, a rebuilt Redux-backed Profile subsystem (edit profile, family-member CRUD, manage account, care team, change password, theme, TV pairing via QR, TV picture upload), housekeeping, transportation, profile. **Multi-tenancy resolved:** the app now injects `x-facility-id` (from the `FACILITY_ID` AsyncStorage key) on every request. **Foreground push wired** via `@notifee/react-native` (FCM `onMessage` rendering); background/terminated tap routing still missing. **Still open:** Medication + Advance Care Directive remain mock; the **PRODUCTION env URL still points at the Shashi Hotels backend** (`api.hospitality.andmv.com`) — a release-blocking misconfiguration. See [client-resident-app-sl analysis](./_codebase-analysis/client-resident-app-sl.md). |

The Skilled Nursing resident app (`senior_living_skillednursing_resident`) remains the **most complete resident client** and the de-facto reference implementation for resident UX, but on `staging` the SL app has closed most of its prior gap: the bulk of its booking/schedule/health/profile surfaces are now API-backed rather than mock. This convergence is the single most consequential input to the shared-component refactoring decision.

---

## 7. Top observations & candidate gaps (rollup)

Detailed, evidence-cited lists live in each module doc's §9. The platform-level headlines:

1. **Product boundary is emergent, not modeled** — no facility-level product flag; behavior derives from careType + facilityType + page config + client binary. A refactor should introduce an explicit product/feature-flag model. *(platform-foundation §9)*
2. **SL resident app has largely converged (staging)** — most prior mock/unwired surfaces are now API-backed (salon/massage/PT booking, `/care` health flows, MySchedule/Activities, profile subsystem); `x-facility-id` and foreground push are wired; ~60 test files now exist. Residual: Medication + Advance Care Directive still mock, background-push tap routing absent, and the **PRODUCTION env still points at the Shashi Hotels backend** (release-blocking). *(client analysis; wellness-services §9; announcements-notifications §9)*
3. **Authorization gaps server-side** — a cluster of admin CRUD routes (venues, schedules, announcements, menu, transport rules, housekeeping admin, resident/staff mutations, config writes) lack auth middleware; the facility-header guard is dead code (cross-tenant risk). A new instance of this pattern shipped 2026-07-10: an unauthenticated `POST /pcc-sync/*` tool that writes PHI for an arbitrary `facilityId` supplied in the request body (backend-only surface; primarily a Skilled Nursing PHI concern — see the companion PRD §7 for detail, since it touches PCC-linked facilities specifically, but it is the same platform-wide auth-gap class documented here). *(platform-foundation §9 and per-module §9s)*
4. **Client-trusted commercial inputs** — meal pricing, transport distance/complimentary determination computed or supplied client-side. *(dining §9, transportation §9)*
5. **Vertical triplication** — salon/massage/private-training are three drifted copies of one "bookable venue" pattern (naming drift `isDelete` vs `isDeleted`, divergent reschedule rules, inconsistent pagination meta). Prime shared-component refactoring target. *(wellness-services §9)*
6. **Timezone architecture** — a single process-wide facility timezone despite multi-tenant design; hardcoded LA timezone in places. The staff app has since added a client-side, facility-configured IANA timezone layer (`platform-foundation.md` PLAT-FR-15a) — a client-only mitigation; backend cron/slot math is unchanged. *(platform-foundation §9)*
7. **Staff app designation coverage** — the app now resolves a **three-way flow** (MIGRATED / LEGACY / CHAT_HOME) off backend `staffDirectoryRoles`; clinical CM/Doctor/DON/SW gained a full Skilled Nursing experience, but several roles still have no task experience, a built `DietitianView` is unreachable, massage/PT views remain read-only, and the flow gating is client-side off a hardcoded map. *(wellness-services §9, personas doc §2.2)*
8. **Staff-app biometric App Lock is device-level, not account-level (2026-08-21)** — a facility-configurable Face ID/Touch ID/passcode gate now protects every PHI screen, but enrollment is per-device: once one staff member enrolls a shared facility device, any subsequent user of that device unlocks the same way. Flagged for product/security sign-off before treating it as a hard per-user PHI control on shared devices. *(platform-foundation.md PLAT-FR-75)*

---

## 8. Out of scope for this PRD

- Technical architecture evaluation, API/schema reference, and code-quality findings beyond product-relevant observations (next phase: technical evaluation).
- The Skilled Nursing clinical/coordination modules (companion PRD).
- The Shashi Hotels platform (separate product family, despite shared heritage visible in the code).

## 9. Next steps

1. Review the candidate-gap sections (§9 of each module doc) and ratify which observations are *bugs to fix*, *intended scope to build*, or *de-scoped*.
2. Decide the explicit product/feature-flag model (replaces the emergent split) before refactoring.
3. Use the surface-maturity matrix (§6) to choose the reference client per module for the shared-component program.
4. Resolve the open product-decision questions in §10.

---

## 10. Open questions / product decisions

- **`HotelDemoSlideshowModal` (admin web) and the backend "hotel demo slideshows" feature — domain fit is questionable.** As of the 2026-07-12 architecture-doc delta review, `senior_living_admin` (`src/components/HotelDemoSlideshowModal.tsx`, 501 lines, merged via a branch named `hotel-slider-changes`) and `senior_living_backend` (`hotelDemoSlideshow.model.ts`/`.controller.ts`/`.routes.ts`, plus a new `Config.showSlideShowModal` flag and an unbounded-size S3 uploader) ship a config-gated, hidden-gesture-activated CRUD tool for a sales/demo slideshow (image/video/PDF slides with optional audio and per-slide duration). Neither the branch name, the commit trail, nor any existing `req.md`/PRD in this repository indicates this is in-scope resident-care functionality. **This is not documented above as normal product scope** — it is flagged here for a human product decision: (a) is this an intentional internal sales-demo tool that belongs in the Senior Living codebase, (b) is it an accidental cross-platform cherry-pick from the Shashi Hotels codebase that should be reverted, or (c) does it need its own scoped PRD if it is intentional. Until decided, treat it as unscoped, undocumented functionality — not a Senior Living resident-care feature. *(Evidence: `docs/reviews/2026-07-12/review-senior_living_admin.md` §"What changed" item 7 and Cross-cutting notes; `docs/reviews/2026-07-12/review-senior_living_backend.md` §"What changed" item 4.)*
