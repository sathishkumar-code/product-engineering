# Module: Admin Dashboard, Reporting & Settings

> Applies to: Both
> FR prefix: DSH
> Sources: `_codebase-analysis/client-admin-web.md` (§1.4–1.7, §2.1, §2.15–2.17, §2.14b, §4.2–4.5), `_codebase-analysis/backend-platform-identity.md` (§2.2, §4, §10), `_codebase-analysis/backend-wellness-dining-ops.md` (§0.4). Code is source of truth.

---

## 1. Purpose & scope

This sub-document specifies the shared "operational home" surfaces of the admin web app:

1. **Dashboard** — the landing view after login: facility-level stat cards, an upcoming-appointments feed, an admin-only recent-activity feed, and permission-filtered quick actions.
2. **Resident payment history** — the per-resident monthly charge ledger aggregated across billable activity modules (reached from the Residents list, but functionally a reporting surface).
3. **Reports** — currently a dead/mock page; documented here as-built plus the intended chart set, as a candidate gap.
4. **Settings** — on staging, two tabs: account info (now editable: name/email persistence + profile-photo upload + embedded change-password) and accessibility preferences. Notification-preference toggles moved to a standalone **Notifications** nav page; the integrations surface (Google Calendar + Zoom OAuth linking + static chips) was extracted into `IntegrationSettings.tsx`, which is currently **imported nowhere** (orphaned — no live entry point).
5. **Facility configuration surface** — the per-facility `Config` fields this module consumes (timezone, inactivity timeout, theme/logo).

Out of scope: the domain modules whose data feeds the dashboard (Residents CRUD, Transport, Salon, Housekeeping, Rehab, Dining — each has its own module PRD), the chat/notification-panel internals beyond what the dashboard shell exposes, and TV/mobile settings.

---

## 2. Personas & surfaces

| Persona | Surface | What they get |
|---|---|---|
| Facility **ADMIN** (Cognito group `ADMIN`) | Admin web — Dashboard | All stat cards, upcoming appointments, **Recent Activity feed (admin-only)**, all quick actions |
| **STAFF** (Cognito group `STAFF`) | Admin web — Dashboard | Stat cards and quick actions filtered to their per-page access permissions; no Recent Activity feed |
| ADMIN / STAFF | Admin web — Residents → Payment History | Per-resident monthly charge summary + line items (the dollar-icon action is currently ungated) |
| ADMIN / STAFF | Admin web — Settings (Account + Accessibility) and standalone Notifications page | Account info (now editable: name/email + profile photo + change-password), accessibility, notification toggles (moved to Notifications page, still decorative). Google/Zoom linking still acts on the staff member's own profile **but its UI is orphaned on staging** (DSH-FR-16/17). |
| Facility operator (out-of-band) | Backend `Config` document | Timezone, inactivity timeout, theme color, logo — no admin UI exists for these; managed via config API/DB |

Visibility of the Dashboard nav item itself follows the platform two-filter model (facility `accessPages` ∩ staff `accessPermissions`); `Dashboard` is one of the 12 canonical permission names (`backend-platform-identity.md` §3.4).

---

## 3. Functional requirements (as-built)

### Dashboard (`DashboardOverview.tsx`, ~600 ln)

- **DSH-FR-01 — Total Residents card.** Show the facility's resident count sourced from `GET /residents`. As-built the card counts the returned array with no pagination handling, so the number is wrong once residents exceed one page (evidence: client-admin-web.md §2.15 item 1). PRD intent: a true count (use list `meta.total` or a count endpoint).
- **DSH-FR-02 — Pending Requests card (permission-gated aggregate).** Sum of pending items across: transport requests + salon appointments + housekeeping requests + private training + massage. Each component is included **only if the viewer passes `checkAccess()` for that module**, so the displayed number is personalized per viewer's permissions. Housekeeping uses the role-dependent read endpoint (`housekeeping/housekeeping-admin` for ADMIN vs `housekeeping` for STAFF).
- **DSH-FR-03 — Today's Activities card.** Count of activity schedules occurring today: `GET /schedules?date=YYYY-MM-DD`, displaying `meta.total`.
- **DSH-FR-04 — Care Requests card.** Current-month rehab appointments via `GET /rehab-appointments?startDate&endDate&limit=500`, filtered client-side to PT/ST/OT therapy types. (Inherently SNF-flavored — see §8.)
- **DSH-FR-05 — Upcoming Appointments feed.** Max 8 entries: merged salon + private training + massage bookings with status pending/confirmed, sorted by date+time; 24h→12h time formatting with mixed-format date parsing. Clicking an entry navigates to the owning module view (state-based navigation — no URL change, per the app's router-less shell).
- **DSH-FR-06 — Recent Activity feed (ADMIN only).** `GET /unified-schedule/recent-activity?limit=5` — a normalized cross-module event stream covering Transport / Salon / Dining / Housekeeping / Massage Therapy / Rehab, with color-coded status badges (approved = green, pending = orange, completed = blue). Backend: the endpoint lives on the **UnifiedSchedule** spine and returns "humanized actions per type" (backend-wellness-dining-ops.md §0.4). Hidden for STAFF.
- **DSH-FR-07 — Quick Actions.** Permission-filtered shortcut tiles: Residents, Staff, Book Salon, Schedule Transport. Each tile renders only if the viewer can access the target module.
- **DSH-FR-08 — No charts on the dashboard.** The dashboard is cards + feeds only; all charting in the app lives in Activity Attendance Report and the (dead) Reports page.

### Resident payment history (`PaymentHistoryModal`, 433 ln)

- **DSH-FR-09 — Monthly charge ledger per resident.** From the Residents list (dollar icon, currently **ungated** by page permission), a month picker drives `GET /residents/{id}/payment-history?month=`. Response renders: a summary header (`totalAmount`, `totalActivitiesCount`) and activity line items.
- **DSH-FR-10 — Charge line-item taxonomy.** Each line item is typed `TRANSPORT | SALON | MASSAGE | PT | MEAL | WELLNESS` and carries status, date/time, location, provider, and amount. A "View More" dialog shows the detail; **location and provider are hidden for MEAL** items.
- **DSH-FR-11 — Backend payment-history service.** `/api/residents/payment-history` (self-scope, used by resident/family apps) plus the per-resident admin variant; both implemented in `paymentHistory.service.ts` (backend-platform-identity.md §4). Amounts are aggregated from the source modules (transport ride price, salon/massage/PT service price, family-meal `guests × mealRate`); no payment processing or collection exists — this is a **statement**, not billing (no gateway; see DSH-FR-19 "Payment Gateway: Not Connected" chip).

### Reports (`ReportsOverview.tsx`)

- **DSH-FR-12 — Reports page (as-built: dead, twice over).**
  1. The nav entry is `{ id: "Reports", subItems: [] }` — a parent with an empty sub-items array never resolves to a clickable view (and the view-registry key is lowercase `"reports"`, a casing mismatch besides).
  2. The page content is 100% hardcoded mock data: 4 fake stat cards (Occupancy 95.4%, Staff-to-Resident 1:4.2, Avg Stay 18 mo, Satisfaction 4.8/5) and Recharts tabs — Occupancy (line, Jan–Jun), Care Types (bar: Independent 35 / Assisted 42 / Memory 28 / Skilled 19), Services usage (bar), Wellness checkups-vs-incidents (line). **No API calls.**
  - The mock content is the de-facto design intent and should seed the real report set if/when built (see §9 G-1).

- **DSH-FR-23 — KPI Dashboard (pre-production, NEW — the first real API-backed analytics surface).** Distinct from the dead mock Reports page (DSH-FR-12), a new **admin-only** KPI dashboard renders live, facility-scoped operational metrics over a date range. Two entry points: a **header quick-access button** ("KPI Dashboard", `LayoutDashboard` icon, visible only when `isAdmin`) opening a full-screen dialog, and a **Settings "KPI Dashboard" tab** gated on `usePageAccess("KPI Dashboard").canView` (`AppContent.tsx`, `SettingsPage.tsx`, `KpiDashboardTab.tsx` ~486 ln).
  - **Data source:** `GET /reports/daily-summary?date=<from>&endDate=<to>` (`dailySummaryReport.controller.ts`) — **query-time MongoDB aggregations** (no cron/materialized store), facility-scoped, over `Resident` / `Message` / `TransportationRequest` / `CareConference`. Returns: `residentsOnboarded {PCC, MANUAL}`, `residentsDischarged`, `transportationRequests {created, updated, served}`, `careConferences {created, updated, served}`, `messagesSentByUser` (map), `conversationsByUser` (map), `uniqueConversations`.
  - **What the admin sees** (Recharts, default range = last 14 days): (1) **Resident & Messaging Overview** — stat tiles for Residents Onboarded (with PCC vs Manual breakdown), Residents Discharged, Unique Conversations, Active Messaging Groups; (2) **Care & Referral Activity** (pre-production, NEW, v1.6) — a third tile row: Secure Calls Made, Referrals Sent, Documents Signed, sourced from three new `DailySummaryReport` fields (`secureCallsMade`, `referralsSent`, `documentsSigned`); (3) **Workflow Snapshot** — grouped bar charts for Transportation and Care Conferences (created/updated/served), a PCC-vs-Manual onboarding donut, and a messaging summary card; (4) **Staff Messaging Activity** — "Messages Sent by Staff" and "Conversations by Staff" leaderboards with a Chart⇄Table toggle (collapsing beyond top-8 into an "Other (n)" bucket). Loading skeletons + a Retry error state.
  - **Facility Occupancy — added then reverted same day (v1.6, never shipped).** The same 2026-08-20 merge that added Care & Referral Activity also added a fourth tile row — Total Beds, Occupied Beds, Available Beds, Occupancy Rate — reading an `occupancy` object with no corresponding field ever added to `DailySummaryReport` or the backend report endpoint. A follow-up merge 14 minutes later ("Error resolved") cleanly reverted the block and its two new icon imports, leaving no dead code or partial wiring. **As-built: Facility Occupancy does not exist on the KPI Dashboard** — see §9 for the observation; do not treat the reverted commit as evidence of an in-progress feature without confirming a follow-up PR actually re-adds it.
  - This is the concrete answer to §9 G-1 ("build the real report set") for the operational/messaging/care-referral slice — occupancy/clinical KPIs from the DSH-FR-12 mock remain unbuilt (the one same-day attempt at occupancy was reverted, not merged). (`use-daily-summary.ts`, `utils/kpiDashboard.ts`, `KpiDashboardTab.tsx`)

### Settings (`SettingsPage.tsx` — restructured on staging to two tabs: Account + Accessibility)

> **Staging restructure:** The Settings page is now two tabs — **Account** and **Accessibility**. The former Notification-Preferences tab moved to a standalone top-level **Notifications** nav page (`NotificationSettings.tsx`, DSH-FR-15), and the **Integrations** tab (Google/Zoom OAuth, DSH-FR-16/17/19) was extracted into `IntegrationSettings.tsx` which is **imported nowhere** — the OAuth linking UI is currently **unreachable** in the admin app (the `useCalendarLink`/`useZoomLink` hooks survive but only `IntegrationSettings.tsx` consumes them, and nothing renders it). A second orphaned `AccessibilitySettings.tsx` was likewise created and de-integrated. The backend OAuth routes are unchanged; only the admin entry point is gone.

- **DSH-FR-13 — Account tab (now editable + persists on staging).** Shows name / email / role; on staging the Account tab **persists name and email** via `api.put('staff/{id}'|'admin/{id}', { name, email })` and supports a **profile-photo upload** (react-easy-crop, 512px / 5MB caps) via multipart `api.put`. It also **embeds `ChangePassword`** (authenticated Cognito `ChangePasswordCommand`). The prior "read-only, no API call" behavior is replaced; the "Save does not save" debt is resolved for the Account tab (`SettingsPage.tsx:125,140-162,352`).
- **DSH-FR-14 — Accessibility tab.** Three preferences, persisted Redux → localStorage:
  - Font size small/medium/large — a CSS-variable multiplier (0.875 / 1 / 1.25) applied to the document root by `FontSizeSync`. **Functional.**
  - High-contrast switch. **Functional** (CSS-level).
  - Read-aloud switch — **persists a flag but no audio implementation exists anywhere.**
  - The "Save Preferences" button is decorative (settings persist on toggle; the button only fires a toast).
- **DSH-FR-15 — Notification Preferences (now a standalone top-level "Notifications" page, still decorative).** As of staging this moved out of Settings into its own nav item (`{ id: "Notifications" }` → `NotificationSettings.tsx`, `AppContent.tsx:207,289`). The toggles remain **toast-only / no backend sync, no behavioral effect** (`NotificationSettings.tsx:35` fires a success toast). The backend's real per-user notification-preference system (keys `DINING, SALON, TRANSPORT, HOUSE_KEEPING, REHAB`, missing key = ON) is still not wired to this UI (backend-platform-identity.md §4).
- **DSH-FR-16 — Google Calendar linking (logic intact, UI orphaned on staging).** The `useCalendarLink` hook still calls `GET /auth/google/url?cognitoUser={cName}` via `authApi` → full-page OAuth redirect, reads linked state from `isGoogleLinked`, and toasts/clears `?sync=success|retry`. **But the only component that renders it, `IntegrationSettings.tsx`, is imported nowhere** — so there is no live entry point to link Google Calendar from the admin app on staging. (The hook and backend route are unchanged; restoring the UI is a re-integration, not new work.)
- **DSH-FR-17 — Zoom linking (logic intact, UI orphaned on staging).** Same situation: `useZoomLink` → `GET /auth/zoom/url?cognitoUser={cName}`, `isZoomLinked`/`zoomUserId`, `?zoom=success|denied|error` toasts — but it lives only in the orphaned `IntegrationSettings.tsx`. Zoom linkage still powers Care Conference meeting creation server-side; `zoomAuthRequired` flags re-link prompts. No reachable admin UI to perform the link today.
- **DSH-FR-18 — What linking enables.** Backend mirrors appointments (Salon, Massage, PT, Care, Care Conference) into the linked staff member's Google Calendar with extended properties for two-way reconciliation, maintains `cachedCalendarBusySlots[]` via Google push-notification webhooks, and creates Zoom meetings for care conferences (backend-platform-identity.md §8.3).
- **DSH-FR-19 — Static integration chips (decorative, now also orphaned).** "Email Service — Connected", "Payment Gateway — Not Connected", "SMS Notifications — Connected" are hardcoded display chips with no backing integration state; they live in the now-orphaned `IntegrationSettings.tsx` (DSH-FR-16/17) and are not rendered anywhere on staging.

### Dashboard shell adjuncts

- **DSH-FR-20 — Notification bell panel.** The shell's bell icon combines (1) Redux-stored in-app notifications (socket-pushed) with unread badge, and (2) today's announcements fetched live. Mark-read and "Mark all as read" are **local-only — no API**; the feed is **lost on page reload** (no server persistence/seeding, despite the backend exposing `GET /api/notifications` history).
- **DSH-FR-21 — Session inactivity logout.** Timeout minutes from `config.inactivityTimeout.web`; activity tracked via mousemove/mousedown/keydown/touchstart/scroll; at timeout−60 s a `SessionTimeoutModal` counts down 60 s ("Stay logged in" / "Log out"), then auto-logout.

### Facility configuration surface (consumed by this module)

- **DSH-FR-22 — Facility config fields.** From the per-facility `Config` document via `GET config` (cached in localStorage, background refetch on mount): `facilityId`, `facilityType`, `logo` (sidebar branding, fallback `/logo.png`), `theme.primary` (applied live as CSS theme color with cached-theme flash prevention), `timeZone` (default `America/Los_Angeles`), `inactivityTimeout.{web,mobile}` (minutes), `lat`/`lng`, plus module configs owned by other PRDs. **There is no admin UI to edit any of these** — `Config` writes happen via mostly-unauthenticated config endpoints / DB (see §9 G-6).

---

## 4. Business rules & policies

- **BR-1 — Personalization by permission, not by role alone.** The Pending Requests aggregate and Quick Actions are composed per-viewer: a STAFF user with Salon but not Transport access sees a pending count that excludes transport. Two staff members can legitimately see different numbers on the "same" card.
- **BR-2 — Recent Activity is an ADMIN privilege.** The cross-module feed exposes activity across modules the viewer might not individually hold permissions for; restricting it to ADMIN is the (implicit) mitigation.
- **BR-3 — Payment history is informational.** Charges are derived from module records (ride price, service price, guests × meal rate); there is no invoice, payment status, or collection flow. `WELLNESS` exists in the line-item type union but no admin module currently labels itself WELLNESS — treat as reserved/legacy.
- **BR-4 — Care Requests = clinical rehab only.** The card counts PT/ST/OT therapy-coded rehab appointments for the current month; AL-side `/care` engine sessions (PT/Cognitive/Outside Agency) are **not** counted.
- **BR-5 — Accessibility preferences are per-browser, not per-account.** They persist in localStorage, so they do not roam across devices or survive a cleared browser profile.
- **BR-6 — Month is the payment-history reporting grain.** The API takes a `?month=` parameter; no date-range or year view exists.

---

## 5. Notifications & real-time behavior

- The dashboard itself does **not** subscribe to real-time updates — stat cards and feeds are fetch-on-mount (React Query). Backend UnifiedSchedule sync emits `mobile-<Model>-request-upserted/-deleted` socket events on the default namespace, but the admin dashboard does not consume them (and those events are globally broadcast with no facility scoping — a backend gap noted in backend-platform-identity.md §10.16).
- The shell-level bell panel receives in-app notifications over the authenticated `/notifications` socket namespace (`notification:new`, room `user:<cName>`) and merges today's announcements; persistence is client-side only (DSH-FR-20).
- Settings notification toggles have **no effect** on any push/socket/email channel (DSH-FR-15).

---

## 6. Integrations (Google Calendar, Zoom OAuth)

> **Staging note:** the link/callback logic below is intact (hooks `useCalendarLink`/`useZoomLink`, `authApi`, backend routes), but the only UI that exposes "Connect" — `IntegrationSettings.tsx` — is imported nowhere, so there is **no reachable admin entry point** to initiate linking on staging (DSH-FR-16/17). Treat the linking UI as a re-integration task.

| Aspect | Google Calendar | Zoom |
|---|---|---|
| Link initiation | `GET /auth/google/url?cognitoUser={cName}` (authApi) → consent redirect (scope `calendar.events`, offline) | `GET /auth/zoom/url?cognitoUser={cName}` (authApi) → consent redirect |
| Callback | `GET /auth/callback` stores `googleRefreshToken` on the Staff row | `/auth/zoom/*` callback stores `zoomRefreshToken`, `zoomUserId` |
| Linked-state flags | `Staff.isGoogleLinked` | `Staff.isZoomLinked`, `zoomAuthRequired` (re-link prompt) |
| What it powers | Appointment mirroring (Salon/Massage/PT/Care/CareConference) into the staff calendar; busy-slot cache via Google push webhooks | Care-conference Zoom meeting creation; recordings/transcripts via Zoom webhooks |
| Return-trip UX | `?sync=success\|retry` toast | `?zoom=success\|denied\|error` toast |

Security note for follow-up (backend): **none of the `/auth/*` link/callback routes carry auth middleware**, and the Google callback **upserts** a Staff doc keyed by attacker-controlled `state` (cognitoSub) — a forged state can create junk Staff records (backend-platform-identity.md §8.3, §10.15). The PRD position: callback must validate state against an issued, signed nonce and must never upsert.

---

## 7. Permissions & access control

- **Nav visibility:** Dashboard, Reports, and Settings entries pass through the two-filter model — facility `accessPages` (Filter 1, no admin bypass) ∩ staff `accessPermissions` (Filter 2, ADMIN bypasses). `Dashboard` and `Settings` are canonical permission names; `Reports` is not in the canonical list (consistent with it being dead).
- **In-page gating:** stat-card components and quick actions individually call `checkAccess()` per source module (DSH-FR-02, DSH-FR-07). Recent Activity renders only for ADMIN (DSH-FR-06).
- **Gaps (as-built):**
  - The Payment History action on the Residents list is **ungated** — any user who can see the Residents list can open any resident's charge ledger.
  - `GET /api/unified-schedule/recent-activity` admin-only restriction is frontend-conditional; backend enforcement should be verified/required.
  - Settings is reachable by all portal users; Google/Zoom linking operates on the caller's own staff record (correct scope), but the underlying `/auth` routes are unauthenticated server-side (§6).

---

## 8. Product-split notes (Senior Living vs Skilled Nursing)

- The Dashboard, Settings, and payment-history surfaces are **facility-type agnostic in code** — identical for both products; differentiation comes entirely from which feeder modules the facility enables via `accessPages` (client-admin-web.md §3).
- **Care Requests card (DSH-FR-04) is SNF-leaning:** it queries the SNF-side `rehab/appointments` engine and filters PT/ST/OT therapy codes. An AL facility (whose rehab runs on the `/care` engine with fixed therapy types) gets a structurally empty/meaningless card. A unified product should either make the card source facility-aware or derive it from UnifiedSchedule.
- **Pending Requests composition self-adjusts** per facility: components for disabled modules are absent because the viewer fails `checkAccess()` for pages the facility hides — so the card works for both SKUs without code branching.
- Payment-history line-item types span both lifestyle (SALON, MASSAGE, MEAL) and clinical-adjacent (PT, TRANSPORT) modules; the taxonomy needs no split, only per-facility availability.
- If the two SKUs diverge further, the backend analysis recommends an explicit facility-level product flag (today the split is emergent: careType + accessPages + therapy fork + integrations — backend-platform-identity.md §2.3); dashboard card composition would be a natural consumer of such a flag.

---

## 9. Observations & candidate gaps

| # | Observation | Evidence | PRD disposition |
|---|---|---|---|
| G-1 | **Reports page is double-dead**: unreachable nav (empty `subItems`) + 100% mock charts; the only live reporting is Activity Attendance Report and clinical documents (IDT, Care Conference) | client-admin-web.md §2.16, §4.2 | Candidate feature: real reporting suite. Intended chart set per the mock: occupancy trend, care-type mix, services usage, wellness checkups-vs-incidents, plus occupancy %, staff-to-resident ratio, avg stay, satisfaction stat cards. UnifiedSchedule + resident/payment data can feed most of these without new write paths. |
| G-2 | **Backend mock dashboard endpoint**: `GET /api/admin/getAdminData` returns hardcoded fake appointments/activities and `residentCount: 100` | backend-platform-identity.md §10.6 (`admin.controller.ts:54-136`) | The frontend dashboard does **not** use it (it composes from live module endpoints). Remove or replace with a real aggregate endpoint — a single dashboard-summary API would also fix G-3. |
| G-3 | **Total Residents miscounts** beyond one page (counts array length, ignores pagination meta) | client-admin-web.md §2.15 | Bug; fix via `meta.total` or count endpoint. |
| G-4 | **Decorative Settings surfaces (partially changed on staging)**: the Account tab now genuinely persists name/email + profile photo (DSH-FR-13), so that "Save does not save" debt is resolved there. Still decorative: the Accessibility-tab "Save Preferences" button (toast-only), the Read-Aloud flag (no audio), and the Notification toggles (now on the standalone Notifications page, still localStorage/toast-only, key vocabulary disjoint from the backend's real preference system). The Email/Payment/SMS chips and the Google/Zoom linking UI are now **orphaned** (`IntegrationSettings.tsx` imported nowhere — DSH-FR-16/17/19). | `SettingsPage.tsx`, `NotificationSettings.tsx`, `IntegrationSettings.tsx`; backend-platform-identity.md §4 | Wire notification toggles to the backend preference API (reconciling key sets); re-integrate or delete `IntegrationSettings.tsx` / `AccessibilitySettings.tsx`; implement or drop read-aloud. |
| G-5 | **In-app notification feed is volatile** — Redux-local, lost on reload; backend `NotificationHistory` + `GET /api/notifications` exists but is unused by the admin shell | client-admin-web.md §2.14b, §4.5; backend-platform-identity.md §5.1 | Seed the panel from the history API on login; persist read state server-side. |
| G-6 | **No facility-settings admin UI**: timezone, inactivity timeout, theme color, logo, meal windows etc. live on the `Config` doc with no edit surface; several `/api/config` write routes are unauthenticated | backend-platform-identity.md §2.2, §10.11 | Candidate feature: a facility-settings admin page (super-admin/admin gated) replacing out-of-band config edits; prerequisite: auth on config write routes. |
| G-7 | **Payment History modal ungated** + month-only grain, no export | client-admin-web.md §2.1 | Gate behind Residents (or a dedicated Billing) permission; consider date-range + CSV/print export as fast-follow. |
| G-8 | **`use-recent-activity.ts` wiring uncertainty** — referenced by Dashboard but missing from one directory scan | client-admin-web.md §4.1 | Verify hook file presence/wiring before building on the feed. |
| G-9 | **Recent-activity socket potential**: UnifiedSchedule already emits upsert/delete events, but globally broadcast without facility scoping or auth on the default namespace | backend-platform-identity.md §8.4, §10.16 | If the feed goes live-updating, fix event scoping first (facility rooms + auth). |
| G-10 | **Token-refresh buffer mismatch** in the shell (tokenService 1 min vs MainApp 5 min vs 5-min comment) | client-admin-web.md §4.5 | Engineering hygiene note; align to one buffer. |
| G-11 | **Facility Occupancy tiles were attempted and reverted same day (2026-08-20, v1.6)** — a KPI Dashboard PR added a Total/Occupied/Available Beds + Occupancy Rate tile row referencing an `occupancy` data source that was never wired to the backend `daily-summary` report; reverted 14 minutes later, cleanly (no dead code). Signals real appetite for an occupancy KPI. | `KpiDashboardTab.tsx` commits `f30e16d2`/`6b65ff77` | Candidate feature: add an `occupancy` block to `GET /reports/daily-summary` (bed/room counts already exist on `TransportationRequest`-adjacent facility data) before re-attempting the frontend tile. |
