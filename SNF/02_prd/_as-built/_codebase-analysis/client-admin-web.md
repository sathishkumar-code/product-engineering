# Codebase Analysis — `senior_living_admin` (Admin Web App)

> Reverse-engineered functional specification of the admin web dashboard, produced for PRD work.
> Source of truth: the codebase at `senior_living_admin/` (React 18 + Vite + TypeScript, Redux Toolkit + TanStack React Query, AWS Cognito auth, Tailwind + Radix UI, Socket.IO, Recharts, Lexical).
> Note: package.json pins `react: ^18.2.0`; the older architecture doc says React 19 — treat React 18 as installed truth.
> **Checkpoint: full pass against `pre-production` HEAD `f5b461c6`, 2026-08-28.** This pass backfills the entire `d2b8d05` (2026-06-21, previous full-pass baseline) → `f5b461c6` (2026-08-27) interval across every section (§1–§4), superseding the 2026-08-25 partial refresh that had re-verified only §2.14's message-pinning addition. Primary source for the interval: the code-verified deltas v2.2–v2.8 in `docs/architecture/architecture-senior_living_admin.md` (production HEAD `59d22ea` through `pre-production` HEAD `f5b461c6`), re-framed here in user-facing/business-rule terms; spot-checked directly against source for the Care Conference Calendar, KPI Dashboard, Consent Forms, and the Residents list rework, which were under-specified for PRD purposes in the architecture pass.

---

## 1. App overview & information architecture

### 1.1 What this app is

Admin/staff web dashboard for a senior-living facility platform. One facility per login (multi-tenancy via `x-facility-id` request header). It manages residents, staff, activities, dining, transportation, housekeeping, salon/wellness services, rehab and clinical reporting, referrals, announcements, internal chat, and (new this interval) resident documents/consent and a live KPI analytics surface. Residents and families interact through separate mobile/TV apps; this app is the operational back-office that configures services and processes inbound requests.

### 1.2 Shell & navigation model

- **No React Router.** `MainApp.tsx` → `AppContent.tsx` renders views from a state variable (`activeView`); URLs never reflect the current page (no deep links, no back-button view history).
- `BASE_NAVIGATION` in `AppContent.tsx:109-222` is the full nav tree — every page that can exist, with **no facility-type branching in code**. View→component map at `AppContent.tsx:243-282`.
- Sidebar shows facility logo (from `facilityStorage`, fallback `/logo.png`), time-of-day greeting, notification bell with unread badge, chat unread-conversation badge, a **Help** button (Freshworks widget), an **admin-only "KPI Dashboard" quick-access button** (`isAdmin` gate, `AppContent.tsx:1001-1009` — opens a full-screen dialog rendering `KpiDashboardTab`, §2.18), and logout.

Full navigation tree (ids double as permission/page keys) — **19 top-level items, 29 sub-items**, current as of `f5b461c6`:

| Top-level item | Sub-items (view ids) |
|---|---|
| Dashboard ("Home") | — |
| Residents | — |
| Dining | `all-day-menu`, `specials`, `family-meal-requests`, `diet` |
| **Transport** (label sweep: was "Transportation") | `complimentary` ("Transport Rules", was "Transportation Rules"), `transport-requests` ("Schedule Transport"), `transport-calendar` (real day/week/month calendar — `TransportCalendar.tsx`) |
| Salon ("Salon & Services") | `salon-settings`, `salon-appointments` |
| Housekeeping | `extra-room-cleaning`, `extra-laundry`, `miscellaneous-service`, `maintenance-requests` |
| Massage-Therapy | — |
| Activities | `activities-schedule`, `activity-attendance`, `activity-attendance-report` |
| Rehab | AL set: `therapy-evaluations`, `physical-therapy`, `cognitive-sessions`, `private-training`, `outside-agency-services` · SNF set: `rehab-calendar`, `rehab-appointments`, `rehab-service`, `rehab-message`, `rehab-team`, `rehab-my-availability`, `idt-report` |
| Staff | — (renders `CareTeam`) |
| Access-Management | — |
| Announcements | — |
| **Referrals** (label: **"Home Health Referrals"**, was "Referrals" — nav renamed) | — |
| Messages | — |
| **Care Conference** (`schedule-care-conference`, **restructured this interval** — was a standalone top-level `care-conference-reports` item) | `care-conference-reports` ("Schedule Care Conference" → `CareConferenceReports`), `care-conference-calendar` ("Care Conference Calendar" → **`CareConferenceCalendar`, NEW**) |
| Reports | **empty `subItems: []` — effectively dead**; the KPI Dashboard (§2.18) is the real analytics surface that has since shipped alongside it, not a replacement for it |
| Notifications | — (top-level item → standalone `NotificationSettings.tsx`) |
| Settings | — (gained a KPI Dashboard tab, §2.18) |

The Rehab parent deliberately lists *both* the Assisted Living and Skilled Nursing sub-page sets; which set appears for a given facility is decided entirely by the facility-pages API (Filter 1 below), not by frontend branching. Source comment at `AppContent.tsx:104-107`: *"All pages — no facility-type branching. Filter 1 (facility API) and Filter 2 (staff permissions) handle what shows."*

Sub-item totals: Dining 4 + Transport 3 + Salon 2 + Housekeeping 4 + Activities 3 + Rehab 11 + Care Conference 2 = **29**.

### 1.3 The two-filter visibility model

Navigation visibility = Filter 1 ∩ Filter 2, applied in sequence in `AppContent.tsx:350-468`:

**Filter 1 — facility-enabled pages (applies to everyone, including ADMIN).**
- `GET config/access-pages/all` (`use-fetch-facility-pages.ts`, gated on facilityId, 5-min stale time) returns the facility's page tree: `{ name, isHidden, rank, children: [{ name, isHidden, rank }] }`.
- A page absent from (or hidden in) this response is hidden for *all* users regardless of role. **There is no admin bypass on Filter 1** — this is the mechanism that switches the SKILLED_NURSING vs ASSISTED_LIVING product variants (e.g., which Rehab sub-pages a facility sees).
- `rank` fields drive nav ordering: both parents and children are sorted by rank (missing rank → Infinity → last).
- Page-name matching uses `normalizePageName()`: lowercase + strip all non-alphanumerics — so `"Massage-Therapy"`, `"massage therapy"`, and `"MassageTherapy"` are equivalent. The same normalization is used everywhere permissions are compared (see §1.4).

**Filter 2 — staff access permissions (ADMIN bypasses).**
- ADMIN sees every facility-enabled page (`canAccessPage = isAdmin || canRead(...)`).
- STAFF visibility comes from `staff.accessPermissions` on their profile (`GET staff/profile`, normalized by `normalizeStaffAccessPermissions`).
- Sub-item inheritance rule (`canAccessSubItem`, `AppContent.tsx:395-411`, mirrored by `submenuCanRead`/`submenuCanWrite` in `utils/accessLevel.ts`):
  1. Child has its own permission entry → use it directly.
  2. Child has no entry AND no facility-enabled sibling has an explicit grant → inherit parent access.
  3. Child has no entry BUT some sibling is explicitly granted → **deny** ("unlisted = not granted").
  - This handles sub-pages not tracked in the access-pages permission API (SNF rehab views, Activities sub-pages).
- If a parent is accessible but the staff member has no sub-item-level grants yet, all facility sub-items are shown as fallback.
- **"KPI Dashboard" is a real permission key** (`usePageAccess("KPI Dashboard")`, Settings tab, §2.18/§2.17) — it behaves like any other Filter 1/Filter 2 gated page for the Settings-tab entry point; the separate **header quick-access button is `isAdmin`-only and bypasses this permission entirely** — a staff member could in principle be granted the "KPI Dashboard" page permission without ever seeing the header shortcut, and vice versa an admin sees the header button regardless of any per-page grant.

### 1.4 Role & permission model

- **Roles** come from Cognito groups in the ID token (`cognito:groups`): `ADMIN` and `STAFF`. `hasPortalAccess = isAdmin || isStaff`. `user.role` = first group; `user.cName` = Cognito username from the access token (used as the cross-system person key — residents and staff are addressed by `cName` throughout the APIs).
- **ADMIN**: full read+write on every facility-enabled page; `accessPermissions` ignored; profile via `GET admin/profile`.
- **STAFF**: profile via `GET staff/profile`; `accessPermissions: StaffAccessPermission[]` is a strictly 2-level tree of `{ name, allowed, isRead?, isWrite?, children[] }`.
  - Access levels: **NONE** (`false,false`), **READ** (`true,false`), **WRITE** (`true,true`). `isWrite` implies `isRead`.
  - Legacy compatibility (`accessUtils.ts:28-32`): records with only `allowed: true` and no read/write flags default to READ; `allowed: false` is NONE.
- **Initialization** (`useInitializePermissions.ts`): after login, the profile is fetched by role; `getAccessiblePageNames()` / `getWritablePageNames()` flatten the permission tree (parents + readable/writable children) into Redux `permissions.accessiblePages` / `writablePages` string arrays; `facilityId` from the profile is persisted (`facilityStorage` + Redux `facilitySlice`). Profile-fetch failure → tokens cleared, forced logout, error stashed in sessionStorage `loginError` and surfaced on the login page.
- **Per-page gating**: every module calls `usePageAccess(pageKey)` → `{ canView, canEdit, isReadOnly }`, backed by `useRoleBasedAccess.checkAccess / checkWriteAccess` (ADMIN always true; otherwise normalized-name membership test against the Redux page lists). The universal in-component pattern is a `requireEdit()` helper: if `!canEdit`, toast "You have read-only access" and abort; mutating buttons rendered `disabled={!canEdit}`. Read-only users still see the pages (read-only mode rather than hiding).
- **Designations** are staff job roles ("Case Manager", "Doctor"/"Physician", "Salon Stylist", "Housekeeping Staff", "Maintenance Staff", "Rehabilitation Specialist", "Transport Driver", …). They serve three functions:
  1. **Permission templates** — `GET/POST /config/designations`, `PUT /config/designations/{name}/permissions`, `DELETE /config/designations/{name}` manage named permission sets; assigning a designation seeds a staff member's `accessPermissions`. Individual staff can then be overridden per-member in Access Management (override wins). `DesignationManagement` now shows an **indeterminate checkbox state** for partial-grant designations and **hides the delete action while any staff member holds the designation** (in-use gating).
  2. **Functional staff lookups** — modules fetch staff filtered by designation or designation-group (Residents form: care-team assignment via `assignedStaff[]`; transport approval: drivers; housekeeping assignment: Housekeeping/Maintenance Staff; rehab: `designationGroup: "rehab"`; salon: "Salon Stylist"; referrals: "Physician").
  3. **Default staff assignment (new this interval).** When facility config `allowDefaultStaff` is true, `CareTeam.tsx`'s add/edit staff forms expose an **`isDefaultStaff`** checkbox per staff member (not offered for the "Physician" designation) — a staff member so marked is auto-populated onto every new resident's `assignedStaff[]` on admission, reducing manual care-team assignment for common roles.
- A hardcoded `ALL_ACCESS_ITEMS` list in `utils/roleBasedAccess.ts:9-22` (12 pages: dashboard, residents, staff, services, salon, transport, housekeeping, maintenance, dining, rehab, access-management, settings) acts as fallback defaults; the live permission tree is otherwise fully dynamic from `GET /config/access-pages`.
- **Chat is now designation-gated.** `DesignationManagement` additionally configures **which staff designations may use the chat UI at all** (`GET/PUT config/chat/staff-designation-allowed/{designation}` — §2.2), and a companion **staff-directory-role mapping** (`GET/PUT config/staff-directory-roles`) controls which designations a given viewer-designation may see in staff-lookup pickers.

### 1.5 Auth flow (AWS Cognito, phone-**or-email** login — reworked this interval)

**Login overhaul, `LoginPage.tsx` (new).** Staff/admin sign-in was reworked from phone-only to a **phone-or-email segmented toggle** (`loginMethod: "phone" | "email"`, default `"phone"`); the phone tab keeps the international phone input (react-phone-input-2), the email tab is a plain email field validated with `EMAIL_REGEX`. Cognito `InitiateAuthCommand` with `USER_PASSWORD_AUTH` and computed `SECRET_HASH` = Base64(HMAC-SHA256(username + clientId, clientSecret)) via CryptoJS, same as before, just against whichever identifier the user chose.

- **Remember-me (new):** a checkbox at login persists the chosen identifier (`localStorage.rememberedPhone` / `rememberedEmail` + `rememberedLoginMethod`) and switches token storage from sessionStorage to localStorage so the session survives a browser restart; it does **not** shorten or otherwise change session length, only where the identifier/tokens live. `MainApp` reads the `rememberMe` flag to bypass the inactivity-logout timer entirely when set.
- **MFA-channel pinning (new):** immediately after the identifier + password are submitted (and again before an MFA challenge response), the client calls `POST /auth/login-mfa-channel { identifier, channel: "sms" | "email" }` (`pinOtpChannel()`, `LoginPage.tsx:41-51`, best-effort/silent-error) so an account with **both** a phone and an email gets its OTP on the channel matching the tab it signed in from, rather than Cognito silently defaulting to SMS for both. A failure here doesn't block sign-in — Cognito just picks its own default channel.
- **New Cognito challenges handled:** `EMAIL_OTP` (a 6-digit code delivered by email, parallel to the existing `SMS_MFA`) and `SELECT_MFA_TYPE` (Cognito asks which MFA type to use; the client answers automatically based on `loginMethod` — email tab → `EMAIL_OTP`, phone tab → `SMS_MFA` — so the user is never shown a manual MFA-type picker). Existing challenges (`NEW_PASSWORD_REQUIRED`, `MFA_SETUP`, `SOFTWARE_TOKEN_MFA`) are unchanged.
- **Legacy Cognito user-pool migration (new, login-time):** a `migrateUserPool()` service call transparently migrates an old-pool account to the current pool on first successful sign-in — invisible to the user beyond a normal login. This is one of **three independently-built Cognito-pool migration paths** across the platform (this one at admin/staff login; a separate one on the staff mobile app's own login; and a fourth on Admin/Staff password reset, below) — whether all three target the same legacy pool has not been confirmed cross-team (see `docs/prd/modules/platform-foundation.md` PLAT-FR-72/72a/72b/76).
- **Orphaned passwordless screen:** a `ResidentLogin.tsx` component (Cognito `USER_AUTH` choice flow, `EMAIL_OTP`/`SMS_OTP`) was built but is **not wired into any route or nav path** — dead code, not a shipped resident-login option on this app.
- **Challenge routing** (`AuthFlow.tsx` orchestrator, unchanged in shape, now also routes `EMAIL_OTP`):
  - `NEW_PASSWORD_REQUIRED` → `PasswordChange` (policy: ≥8 chars, uppercase, lowercase, number, special char `[!@#$%^&*(),.?":{}|<>]`); may chain into MFA challenges. First-login `PasswordChange` also requires **terms acceptance** (`use-accept-terms.ts` → `POST /auth/accept-terms`) plus a privacy-policy link.
  - `MFA_SETUP` → `MFASetup`: TOTP secret via `AssociateSoftwareTokenCommand`, QR (`otpauth://totp/{issuer}:{email}?secret=…`, qrcode.react) + copyable manual secret, then `VerifySoftwareTokenCommand` + challenge response → tokens.
  - `SOFTWARE_TOKEN_MFA` / `SMS_MFA` / **`EMAIL_OTP`** → `MFAVerification`: 6-digit code → `RespondToAuthChallengeCommand`.
  - **MFA reset** ("Lost Device?" in MFAVerification): `POST {VITE_PROD_URL}residents/resetMFA { email }` (no UI-level permission check — backend must validate/rate-limit), then Cognito `ForgotPasswordCommand` / `ConfirmForgotPasswordCommand`.
- **Forgot password** (`ForgotPassword.tsx` — **dead code, never imported**; superseded entirely by `StaffForgotPassword.tsx`, the active 3-step flow: phone/email → OTP (verified via `use-verify-otp-staff.ts` → `POST /auth/verify-otp`) → reset via backend API). **Legacy-pool fallback on reset (new, distinct from the login-time migrator above):** Admin/Staff password reset now also falls back to a separate legacy Cognito user pool (`OLD_USER_POOL_ID`) for lazy migration if the primary pool doesn't recognize the account — a fourth independently-built migration path (see PLAT-FR-76).
- **Token lifecycle** (`tokenService.ts`): access/id/refresh tokens + computed expiry, now in **localStorage or sessionStorage depending on remember-me**; silent refresh via `GetTokensFromRefreshTokenCommand` (with SECRET_HASH) and a queue so concurrent refreshes collapse into one; logout = `RevokeTokenCommand` + clear all auth/facility storage + Redux `resetStore()`, hardened this interval with a bounded race guard (`appReload.ts`) so a slow/failed `RevokeToken` call can't hang the logout UI. `MainApp` hydrates auth on load, verifies the access token with `GetUserCommand`, refreshes when within the expiry buffer (5-min buffer in MainApp; tokenService's own buffer is `1 * 60 * 1000` despite a 5-minute comment — see §4.5).
- **Session inactivity** (`use-inactivity-logout.ts`): timeout from `config.inactivityTimeout.web` (minutes); tracked events mousemove/mousedown/keydown/touchstart/scroll; at timeout−60s a `SessionTimeoutModal` counts down 60s ("Stay logged in" / "Log out"), then auto-logout. **Disabled entirely when remember-me is set**, and also disabled while the chat popup window is open/active.
- **HTTP clients** — two Axios instances:
  - `api` (base `VITE_PROD_URL`, localhost:3000 fallback; on this branch some deploy artifacts instead read `VITE_PREPROD_URL`/`VITE_SOCKET_PREPROD_URL` — see §4.5) — main backend. Request interceptor injects `Authorization: Bearer` + `x-facility-id`; response interceptor does one silent refresh-and-retry on 401, then routes everything through `errorMiddleware.handleApiError` (parse + toast, with duplicate-toast suppression by a `status:message` key).
  - `authApi` (base `VITE_PROD_AUTH_URL`, localhost:7000 fallback) — used by agencies, referrals, OAuth-link endpoints; **defaults `x-facility-id` to `"R101"` if unset** (questionable fallback); redirects to `/login` on failed refresh.
- **Sockets**: `socket.ts` (announcements/notifications) + `chatSocket.ts` (`/chat` namespace) on `VITE_SOCKET_PROD_URL`, both initialized after login for ADMIN/STAFF and torn down on logout.
- **Support widget**: `FreshWorksWidget` (Freshworks script, env-configured) loads only when an access token exists; destroyed on logout.

> Product framing of the login rework: four human-facing clients across the platform now implement **three architecturally distinct** sign-in mechanisms (passwordless-OTP on the SN resident app; password + backend-mediated MFA-channel-pinning + pool migrator, built twice independently, on admin-web and the staff app; TOTP-heavy legacy on the SL resident app). See `docs/prd/modules/platform-foundation.md` §9 item 20 for the cross-client consolidation question this raises.

### 1.6 Facility configuration

- `GET config` (`use-config.ts`, fired once facilityId is known via `useInitializeFacilityConfig`) returns `AppConfig`: `facilityId`, `facilityType`, `logo`, `theme.primary` (applied live as CSS theme color, with cached-theme flash prevention), `lat`/`lng` (facility coordinates for transport distance), `transportation` config object, `timeZone`, `inactivityTimeout`, `maxFamilyMembersCount`, chat attachment/group/pin config, meal config. **New config fields this interval:** `allowDefaultStaff` (bool, §1.4), `dischargeDateMaxPastDays` (referral discharge-date backdating window, default 28, §2.12), `chat.maxGroupMembers`, `chat.pinMessageDurationOptions` / `maxPinnedMessagesPerUser` / `maxPinnedMessagesPerConversation` (§2.14), `showSlideShowModal` (gates the unrelated `HotelDemoSlideshowModal` sales-demo tool, §3.16). Cached in localStorage (`appConfig`) with `initialDataUpdatedAt: 0` so a background refetch always runs on mount.
- `facilityStorage` (`utils/facilityStorage.ts`) is a localStorage wrapper + pub/sub (custom window event) for `facilityId`, `facilityType`, `facilityLogo`. **`FACILITY_TYPE` enum: `SKILLED_NURSING` | `ASSISTED_LIVING`** — the only two values the app recognizes; anything else reads back as null.
- `useFacility()` exposes reactive `{ facilityId, facilityType }` to components.

### 1.7 Redux store & client-state layout

Slices: `auth` (tokens, expiry, user identity incl. groups/role/designation/cName), `permissions` (accessiblePages/writablePages/isLoaded), `facility` (id, type, logo, coordinates, transportation config), `settings` (fontSize, highContrast, readAloud, notification toggles, integration flags), `notifications` (in-app feed + unreadCount), `chat` (pendingConversationId for cross-view navigation). `persistenceMiddleware` syncs settings/auth/permissions actions to localStorage and clears auth keys on logout; a root-level `resetStore` action wipes all slices. Server state lives exclusively in React Query.

---

## 2. Modules

### 2.1 Residents (`ResidentsManagement.tsx` + `AddResidentModal` + `EditResidentModal` + `ResidentDetails.tsx` (new) + `ResidentDocuments.tsx` (new, §2.19) + `SignConsentFormModal`/`ConsentFormPreview` (new, §2.20))

**Purpose:** master resident directory — CRUD, family members, staff-role assignments, login credentials, documents, and consent. Permission key: `usePageAccess("Residents")`.

**List view — reworked this interval.** The prior facility-type-conditional column set (SNF: Assigned Staff column; AL: Care Type + Unit No) has been **replaced by one unified column set for every facility type**: Resident/Room (name + photo + room subtitle — always labeled **"Room No"** now, the AL-specific "Unit No" label is gone), **Type of Admission**, **Length of Stay**, **Physician**, Actions. Both "Type of Admission" and "Length of Stay" are **sortable** (clickable column headers with a sort-direction icon). There is **no Care Type filter or column anymore** — `careType`/`facilityType` branching was removed from `ResidentsManagement.tsx` entirely (contrast the SNF/AL split still present elsewhere in the app — see §3, this is the one place it was retired).
- **Payer Source — added, then removed same interval.** A payer-source column and filter dropdown shipped, then were pulled 9 days later (`ResidentsManagement.tsx`, `new-resident-UI-payersource-column-hide`): the field is still read from the resident API and still accepted as a filter query param, but the column no longer renders and the filter dropdown is fixed to "all" — a UI-only reversal, not a data-model rollback. `ResidentDetails.tsx` hardcodes the displayed payer source value to `"—"`.
- **Status filter now offers Active / Discharged** (default view = Active), resolving the earlier gap where discharged residents had no way to be located by status — see §4.3 update. ("Away" remains a settable status in the Edit form and has its own badge color, but is not a filter option.)
- Search debounced; pagination now defaults to **50/page** (raised from 10) via the shared `RowsPerPageSelect`.
- `GET /residents?page&limit&search&status&payerSource&admissionType&sortBy&sortOrder`.

**Create (AddResidentModal, two-step form):**

| Step 1 field | Required | Validation |
|---|---|---|
| First / Last Name | yes | non-empty |
| Phone | yes | intl input; country code + digits |
| Email | no | regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` if provided |
| Room Number | yes | non-empty (placeholder "e.g., A-101") |
| Care Type | yes | auto-set from facilityType, **disabled** |
| Date of Birth | yes | max = today (blur-enforced) |
| Gender | yes | Male / Female / Other |
| **Admission Date** (new) | yes | required, validated — feeds the list's "Length of Stay" column |
| Profile Photo | no | JPG/JPEG/PNG |

- Emergency Contact field and its validation were **removed** from the Edit form this interval (commented out in `EditResidentModal.tsx`) — a reversal of the earlier "emergency contact remains editable" behavior.
- Step 2 — roles & family: **Assigned Staff** — a single **`StaffMultiSelect`** (`components/ui/StaffMultiSelect.tsx`) writes the resident's `assignedStaff[]` (staff cName array); per-designation minimum-count enforcement (`validateStaffDesignations` against `config.staffAssignmentRequirements`) blocks save if required roles aren't covered. The same multi-select drives chat-mention scoping and the referral form's care-team/physician source. Insurance Name (optional); **family members** — Name/Email/Relation/Type/Phone with all-or-nothing validation per member; Relation options Spouse/Son/Daughter/Brother/Sister; Type options Emergency/Family; max count from config `maxFamilyMembersCount` (toast when exceeded). Each family member now carries **`isAuthorizedAppAccess`** — a family-portal-access toggle distinct from `hasPortalAccess` (whether they've ever logged in).
- **Duplicate phone/email: hard block → soft warning (reversed this interval).** The resident's own phone/email is checked against `GET /residents/check-duplicate` (fail-open on request error) and, if a match exists, shows a confirmation dialog ("are you sure?") rather than blocking the save — duplicates are now allowed with admin confirmation. This **reverses** the family-member phone-uniqueness cross-validation that shipped a few weeks earlier in the same window and directly contradicted it; see PLAT-FR-73/73a for the full before/after and the consent-form interaction below.
- **PCC contacts are now lazy-loaded**: `use-fetch-pcc-contacts.ts` (`GET residents/pcc-contacts?pid=<base64 patientId>`) fires only once the user opens the family-member section, not on modal open — avoids an unnecessary fetch/state-reset for the common case of adding a resident with no PCC prefill.
- Submit: `POST /residents` multipart (resident JSON incl. `assignedStaff[]`/`admissionDate`/family JSON + profilePicture file).

**Full-page resident detail — `ResidentDetails.tsx` (new, replaces the modal-based detail flow as the primary view).** Two tabs render: **Details** and **Documents** (§2.19); a `healthRecords` tab id remains declared but is not rendered (`ResidentHealthRecords` — the component file behind it was deleted; the commented-out import should be removed rather than re-enabled, see §4.1). The page embeds:
- **Family-member panel** with a health-info-access **revoke confirmation** dialog before turning off a family member's `isAuthorizedAppAccess`.
- **`SignConsentFormModal` / `ConsentFormPreview`** for family-member consent signing — see §2.20.
- `EditFamilyMemberModal` for per-family-member edits.
- The older `ViewResidentModal`/`ViewResidentModalNew` view-only modals still exist in the codebase but `ResidentDetails` is now the primary detail surface reached from the list.

**Other actions:**
- Delete: confirmation modal → `DELETE /residents/{id}` → cache prune + toast.
- **Payment History — the button and its handler were removed this interval.** `PaymentHistoryModal` and its import remain in `ResidentsManagement.tsx` but are now unreachable (no trigger renders); a resident's activity payment history has no UI path in as of this pass — a regression pending product confirmation (was: dollar-icon → month picker → `GET /residents/{id}/payment-history?month=`).

**Statuses:** types enum = `Active | Away | Discharged`; the list filter is now Active/Discharged only (both filterable — the badge/enum mismatch noted in the prior pass persists for `Away`/`Inactive` display cases, see §4.4).

**Notable endpoints:**

| Operation | Method & path |
|---|---|
| List / detail | `GET /residents` (paged+filtered incl. `sortBy`/`sortOrder`) / `GET /residents/{id}` |
| Create / update | `POST /residents`, `PUT /residents/{id}` (multipart: resident JSON + familyMembers JSON + photo) |
| Delete | `DELETE /residents/{id}` |
| Duplicate check (new) | `GET /residents/check-duplicate` |
| Send credentials | `POST /residents/send-credentials { id, type: resident\|family }` |
| Staff role lookups | `GET /staff?designation=…` (feeds `StaffMultiSelect` for `assignedStaff[]`) |
| PCC family contacts (lazy) | `GET residents/pcc-contacts?pid=<base64 patientId>` |
| Documents / consent (new) | see §2.19/§2.20 |

### 2.2 Staff & Access Management (`CareTeam.tsx`, `AccessManagement.tsx`, `DesignationManagement.tsx`)

**Staff (nav "Staff" → CareTeam)** — permission key `"Staff"`:
- Two tabs: **Staff** (full CRUD, search debounced, paginated) and **Admins** (read-only list + create modal; `GET /admin` unpaginated, `POST /admin`).
- Staff fields & validation: name (≥2 chars), designation (≥2 chars, picked from `GET /staff/staff-designation-list`), phone (react-phone-input-2; `splitPhoneValue()` extracts `+dialCode` + national digits; ≥7 digits, now displayed with the country-code prefix in the list), email (optional, regex; blank sanitized to an em-dash placeholder), profile picture (FileReader preview; multipart key `staff`; explicit empty string clears on edit), `active` flag (soft-delete/visibility), notes.
- **`isDefaultStaff` checkbox (new, config-gated §1.4):** shown in add/edit when `config.allowDefaultStaff` is true; excluded for the "Physician" designation.
- Endpoints: `GET /staff?designation&designationGroup&search&page&limit`, `POST /staff`, `PUT /staff/{id}`, `DELETE /staff/{id}`.
- **"Send Credentials"** per staff member — **reuses `POST /residents/send-credentials`** (misnamed endpoint, §4.4).
- Staff record also carries: `accessPermissions` tree, weekly `availability` (used by Rehab), `isGoogleLinked`/`isZoomLinked`/`zoomUserId` integration flags, `cName`, `countryCode`.
- Designation Management opens from here as a modal.

**Access Management** — permission key `"Access Management"`:
- Matrix UI assigning per-page READ/WRITE to individual staff. Permission catalog from `GET /config/access-pages` (cached 5 min); structure is fully dynamic from the API and strictly 2-level (parent → children, no grandchildren).
- Stats cards: total staff, staff with ≥1 permission granted, total permissions available.
- Modal: staff selector (all staff, unpaginated) + expandable permission grid with View/Edit toggles per page and child.
- **Parent/child sync rules** (`setNewAccessWithSync`): isRead is forced on when isWrite set; checking a parent applies to all children; checking a child derives the parent (parent isRead = any child read, isWrite = any child write); unchecking parent unchecks children; unchecking the last child unchecks the parent. Parents derived from child-only grants are computed for UI consistency but not persisted until explicitly toggled.
- Save: `PATCH /staff/{id}/access-permissions { accessPermissions }`; invalidates `['get-staff']` + `['get-staff-profile']` so a logged-in staff member's nav updates on next profile fetch.
- Display-label overrides: `Salon → "Salon & Services"` (the previous `Transport → "Transportation"` override was **removed** now that the nav label itself is "Transport" — see §1.2); otherwise kebab/camel names are title-cased for display.

**Designation Management:**
- Modal modes `list | create | edit`. A designation = `{ name, permissions: StaffAccessPermission[] }` — a named permission template; list shows per-designation staff counts.
- Rules: name required + unique; **deletion blocked (in-use gating) while any staff member holds the designation** ("This designation is already in use"); a partially-granted designation now renders with an **indeterminate checkbox state** rather than appearing fully checked or unchecked; names URL-encoded in DELETE/PUT paths (e.g., `Manager & Lead` → `Manager%20%26%20Lead`).
- **New this interval:** a **staff-directory-role mapping** tab (`use-staff-directory-roles.ts`, `GET/PUT config/staff-directory-roles`) controlling which designations a given viewer-designation may see in staff-lookup pickers, and a **chat-designation-allowed** toggle per designation (`use-chat-staff-designation-allowed.ts`, `GET config/chat/staff-designation-allowed`, `PUT config/chat/staff-designation-allowed/{designation}`) controlling which staff designations may use the chat UI at all.
- Endpoints: `GET/POST /config/designations`, `PUT /config/designations/{name}/permissions`, `DELETE /config/designations/{name}`. Cache invalidation refreshes both the designation list and the staff-designation dropdown source.
- Templates seed defaults; per-staff overrides in Access Management take precedence.

**Notable endpoints (Staff & Access cluster):**

| Operation | Method & path |
|---|---|
| Staff CRUD | `GET/POST /staff`, `PUT/DELETE /staff/{id}` |
| Designation dropdown source | `GET /staff/staff-designation-list?designationGroup&filterDesignation` |
| Admins | `GET /admin`, `POST /admin` |
| Permission catalog | `GET /config/access-pages` (per-staff matrix) · `GET config/access-pages/all` (facility Filter 1) |
| Save staff grants | `PATCH /staff/{id}/access-permissions` |
| Designation templates | `GET/POST /config/designations`, `PUT /config/designations/{name}/permissions`, `DELETE /config/designations/{name}` |
| Staff directory role map (new) | `GET/PUT config/staff-directory-roles` |
| Chat-allowed designations (new) | `GET config/chat/staff-designation-allowed`, `PUT config/chat/staff-designation-allowed/{designation}` |

**Key data shape — `StaffAccessPermission`:**

```typescript
{ name: string; allowed: boolean; isRead?: boolean; isWrite?: boolean;
  children?: Array<{ name; allowed; isRead?; isWrite? }> }   // strictly 2 levels
```

### 2.3 Activities & Attendance (`MySchedule.tsx`, `ActivityAttendance.tsx`, `ActivityAttendanceReport.tsx`)

No functional changes this interval beyond the theme-aware PDF accent colour already noted (`getPdfPrimaryColor`, applies to every PDF template in the app, incl. this module's calendar export and the attendance report export).

- **Activities Schedule (`activities-schedule`, permission key `"Activities"`):**
- Dual views: list (paginated, searchable, sortable on 7 fields, day-of-week + active/inactive filters layered client-side) and month calendar (up to 4 events/day with "+N more", today highlighted).
- Activity model: name*, description, location, capacity* (≥1), image (S3 imageKey + signed imageUrl), `isActive` (soft visibility), startTime/endTime* (24h, end > start), and a **5-mode recurrence model** (`everyday`, `one-time`, `weekly`, `multiple-dates`, `date-range`) — the same vocabulary reused by Dining menu items, Specials, and Announcements (platform pattern, §4.6).
- Validation: 9-field error object; endDate ≥ startDate; per-pattern date requirements; errors inline, cleared on input; validation submit-time only.
- CRUD: `GET /schedules?page&limit&isAdminPanel=true&search&sortBy&sortOrder`, `POST /schedules`, `PUT/DELETE /schedules/{id}` (multipart, image under key `activity`).
- **PDF export:** HTML calendar built by `utils/pdfTemplate.ts` (`buildActivityCalendarHtml`), previewed in an A4 iframe and printed via the browser dialog — no PDF library anywhere in the app.

**Activity Attendance (`activity-attendance`):** unchanged — status set `NOT_MARKED | PRESENT | ABSENT`; bulk mark-all actions; `GET /schedule-attendance/schedules?date=` / `GET /schedule-attendance/{scheduleId}?date=` / `POST /schedule-attendance/{scheduleId}/mark?date=`.

**Activity Attendance Report (`activity-attendance-report`, read-only):** unchanged — per-resident/per-activity rollups, resident drill-down with streak + daily bar chart + distribution pie; PDF export via the same theme-aware `buildActivityCalendarHtml`.

**Calendar/Zoom OAuth linking** (consumed by Care Conferences & Settings): `GET /auth/google/url?cognitoUser={cName}` and `GET /auth/zoom/url?cognitoUser={cName}` via `authApi` → full-page OAuth redirect out; linked-state flags `isGoogleLinked`/`isZoomLinked` live on the staff profile.

### 2.4 Dining (`AllDayMenu.tsx`, `Specials.tsx`, `FamilyMealRequests.tsx`, `DietManagement.tsx`, `ImageSelectionModal.tsx`)

No functional changes this interval (a "Nutrition design" UI-work pass landed on `staging` mid-window but is not reflected in the current `pre-production` state observed here — treat as not-yet-shipped).

**All Day Menu (`all-day-menu`):** category → item hierarchy, pagination, search, "View Menu For [date]" filter matching item availability patterns (`EVERY_DAY`/`ONE_TIME`/`WEEKLY`/`MULTIPLE_DATES`/`DATE_RANGE`). Items support image or PDF attachments; per-item active/inactive toggle. CRUD: `POST/PUT/DELETE /items`; list via `GET /menu/getMenuForAdmin?page&limit&date&search`.

**Menu Library + shared image picker (platform pattern):** facility-wide reusable file gallery (`GET/POST /menu-library`, `DELETE /menu-library/{fileId}`); `ImageSelectionModal` is the shared picker used by Dining, Salon/wellness, Rehab Service, Announcements. **Platform-wide image contract:** the backend stores an S3 object key (`imageKey` — sent on create/update) and returns a signed display URL (`imageUrl` — used only for `<img src>`). Never send the signed URL back.

**Specials (`specials`):** promotional menu files scheduled with the shared repeat-pattern vocabulary; "This Week" 7-card view; upload/create, "Schedule from Library", update, delete, week read.

**Family Meal Requests (`family-meal-requests`):** requests grouped by date; **still no approve/reject UI** — admin levers remain configuration-only (per-meal-type pricing, blackout dates). Meal time windows exist in config but are not editable here.

**Diet (`diet`, DietManagement):** per-resident diet plans, resident immutable after creation, table shows only the first plan item (§4.4 unchanged).

**Notable endpoints (Dining cluster):** unchanged from the prior pass — see Appendix A for the shared conventions; per-endpoint detail: `GET /menu/getMenuForAdmin`, `POST/PUT/DELETE /items`, `PATCH /items/{id}/toggle`, `GET/POST /menu-library`, `DELETE /menu-library/{fileId}`, `GET /gallery?folder=`, `POST /daily-specials/upload`, `POST /daily-specials`, `POST /daily-specials/from-library`, `GET /daily-specials/week`, `PUT/DELETE /daily-specials/{id}`, `GET /family-meal-requests?date=`, `PUT /config/mealconfig`, `GET/POST /diet-plans`, `PUT/DELETE /diet-plans/{id}`.

### 2.5 Salon (`Salon.tsx` — `salon-settings`; `Appointments.tsx` — `salon-appointments`)

No functional changes this interval. Store profile CRUD, service catalog CRUD (min-10-word description, staff assignment, active toggle), two-phase delete-with-impact-check, operating hours, admin staff filter; Confirmed/Waitlist appointment tabs with waitlist→confirmed slot-reassignment flow. See Appendix A for endpoint conventions; unchanged endpoint set: `GET/PUT /salon/{id}`, `GET/POST /salon/services`, `PUT/DELETE /salon/services/{id}`, `PATCH .../toggle`, `GET /salon/schedule`, `POST /salon/schedule/update`, `GET /salon/appointments`, `PATCH /salon/appointments/{id}`, `POST /salon/{serviceId}/available-slots`.

### 2.6 Massage / PT / Cognitive / Private Training / Outside Agency (wellness cluster)

No functional changes this interval. Shared conventions across the cluster unchanged (status normalization UI↔API, minute durations, HH:mm↔12h times, uppercase↔title-case days, resident name-fallback chain). Massage Therapy structurally mirrors Salon under `/massage/*`; Physical Therapy/Cognitive Sessions ride the generic `/care` engine (view + create only, still no edit/delete/reschedule); Private Training is the hybrid catalog+schedule+booking model with a client-side pagination fallback; Therapy Evaluations is the AL-side cross-type aggregator; Outside Agency Services still records `agencyName` as a free-typed string, not a foreign key into the managed agency directory (referential-integrity gap, §4.4, unchanged).

### 2.7 Transportation (`ComplimentaryTransport.tsx` — `complimentary`; `AppointmentsTransport.tsx` — `transport-requests`; `TransportCalendar.tsx` — `transport-calendar`)

**Transportation Rules (nav label "Transport Rules"):** unchanged — per-destination-category complimentary/paid rules; CRUD + toggle; `GET /transportation-rules?isActive&isComplimentary`, `POST`, `PUT /{id}`, `PATCH /{id}/toggle`, `DELETE /{id}`.

**Transport Requests ("Schedule Transport") — several new admin workflows this interval:**
- **New-admission ("prospect") booking mode.** `useScheduleTransportModal.tsx` adds a `scheduleMode: "existing" | "newAdmission"` toggle to the admin booking flow: instead of picking a registered resident, the admin can book a ride for an unregistered new admission by entering a name/phone/notes directly (`newAdmissionName`/`newAdmissionPhone`/`newAdmissionNotes`). New-admission bookings **skip the max-distance check** (`skipMaxDistance: true`) and don't auto-derive pickup time from a resident's existing appointment data.
- **Busy-driver confirmation.** If the selected driver already has an overlapping ride, the backend returns a driver-conflict response and the UI opens a "book anyway?" confirmation dialog (`pendingDriverConflict` state) rather than silently blocking the booking; confirming resends the request with a `forceDriverAssign` flag. This exists alongside the pre-existing **resident-schedule conflict** dialog (overlapping appointment for the same resident, `forceBook` flag) — both can fire on the same booking attempt and are resolved independently.
- **Update-confirmation and discard-changes dialogs** on edit, and **edit permissions now gated by the caller's role and the request's current status** (an admin-panel config flag) rather than being uniformly editable.
- Uses `DatePickerInput` (`react-datepicker`-backed) in place of native `<input type="date">` across the scheduling forms.
- Driver selection continues to support a "Transport Driver"-designation staff list or an **outside-agency free-text driver name**.
- Expandable notes for special requests.
- **Status machine, Google Maps distance/pickup calculation, and price entry are otherwise unchanged** from the prior pass — see the endpoint table below; the "driver" query-param filter remains stubbed pending backend support (§4.3, unchanged), and price auto-calculation from `pricePerMiles` remains unimplemented (manual entry only).

**Transport Calendar** (real day/week/month calendar, not a placeholder — this superseded the former `ComingSoon.tsx` stub before this pass's window and remains the live implementation): colour-coded by status (`Completed`/`Approved`/`Pending`/`Requested`/`Unassigned`/`Cancelled`); event times formatted **timezone-aware** via the facility's configured `timeZone`; appointment details now include **`residentFreeAt`** (the resident's availability window after the appointment ends). Edit/delete (soft-delete) actions were added to the calendar view this interval, alongside the admin-web-wide "Transportation" → "Transport" label sweep.

**Notable endpoints (Transportation):**

| Operation | Method & path |
|---|---|
| Rules CRUD | `GET/POST /transportation-rules`, `PUT /{id}`, `PATCH /{id}/toggle`, `DELETE /{id}` |
| Requests list | `GET /resident-transportation?search&status&destinationType&page&limit` |
| Distance/pickup calc | `GET /resident-transportation/calculate-distance?lat&lng&appointmentStartTime` |
| Approve/reject/price | `PUT /resident-transportation/{id} { status, price, priceRemarks, driver? }` |
| Assign driver | `POST /resident-transportation/{id}/assign { cName }` |
| Admin booking (incl. new-admission mode) | `POST /resident-transportation/book-transportation` |
| Destination types | `GET /resident-transportation/destination-types` |

### 2.8 Housekeeping & Maintenance (`ExtraRoomCleaning.tsx`, `ExtraLaundry.tsx`, `MiscellaneousService.tsx`, `MaintenanceRequests.tsx`)

No functional changes this interval. Four near-identical request-queue screens (`EXTRA_ROOM_CLEANING`/`EXTRA_LAUNDRY`/`MISCELLANEOUS_SERVICE`/`MAINTENANCE`) over one backend; status machine `PENDING → IN_PROGRESS → COMPLETED` or `REJECTED`; role-dependent read endpoint (`housekeeping/housekeeping-admin` for ADMIN vs `housekeeping` for STAFF); single update endpoint for all transitions; timezone-aware date display via `appConfig.timeZone`.

### 2.9 Rehab suite — Skilled Nursing (`rehab-calendar`, `rehab-appointments`, `rehab-service`, `rehab-message`, `rehab-team`, `rehab-my-availability`)

No functional changes this interval. SNF-only exposure enforced by Filter 1, not component code. Rehab Calendar (monthly/daily views, hash-based therapy color theming, UTC date-key policy, print/PDF); Rehab Appointments (Upcoming/History tabs, slot-driven start/end time selection); Rehab Service (therapy catalog CRUD); Rehab Message (inbound resident messages, `NEW → IN_PROGRESS → CLOSED`); Rehab Team (staff CRUD scoped to "Rehabilitation Specialist", real-time "Available Now" badge); My Availability (weekly editor, auto-save debounce, `PUT staff/availabilty/{staffCName}` — endpoint typo is real and unchanged).

### 2.10 IDT Reports (`IDTReport.tsx`, view id `idt-report`)

Mostly unchanged this interval; two additions:
- **`IDTReportRecord`** now carries explicit `birthDate` and `admissionDate` fields (auto-filled from the resident, matching the Residents-module `admissionDate` addition, §2.1).
- **`attendingMD`** now handles both a legacy plain string and a new object shape (`{ name, cName }`), reflecting the backend's care-team refactor.
- **"Share to chat"** — IDT reports can now be forwarded into a chat conversation via `ShareWithModal` (picks/creates a conversation, generates a PDF URL if one doesn't already exist, sends it as a reference attachment with a prefilled @mention of the resident) — see §2.14.

Everything else — form sections, DRAFT→SUBMITTED lifecycle (commented-out auto-save still dead code, `IDTReport.tsx:307-322`), linked-appointment checkboxes, History tab, PDF export, absence of an explicit `usePageAccess` gate — is unchanged from the prior pass.

### 2.11 Care Conferences (`CareConferenceReports.tsx` — "Schedule Care Conference"; **`CareConferenceCalendar.tsx` (NEW) — "Care Conference Calendar"**; both now sub-items of a restructured top-level **"Care Conference"** nav group, §1.2)

**Purpose:** schedule and run IDT/family care conferences with Google Calendar + Zoom integration, recordings, and post-meeting summaries — now with a dedicated calendar view alongside the existing list/editor screen.

**Schedule Care Conference (`CareConferenceReports.tsx`, `care-conference-reports`) — additions this interval:**
- **Schedule-conflict detection on save.** Creating or updating a conference now checks the selected resident(s)' unified schedule for overlapping appointments (salon/massage/PT/care/rehab/transportation); a conflict returns a list rendered in a `ConflictPanel` and blocks the save until resolved or the conflict list is cleared. The same conflict machinery is shared with the new Calendar view's edit modal (below).
- **Delete-confirmation dialog** before cancelling a conference (previously an immediate action).
- **12-hour time formatting** (`formatTime12h`) for display; the summary editor is now a plain `Textarea` (the prior Markdown editor dependency was removed).
- **Virtual-meeting affordances:** a video-icon join button, and an exported **`JoinByPhoneSection`** component surfacing the Zoom PSTN dial-in number(s) and passcode for attendees without a Zoom client.
- **`meetingType`** field (Care Conference / Family Meeting / Family Care Conference) now explicit on the form.

**Care Conference Calendar (`CareConferenceCalendar.tsx`, nav id `care-conference-calendar`) — new.**
- A full **day / week / month calendar** of scheduled conferences (default view: month), built on the same `care-conference` API and sharing form components, validation, and conflict-checking with `CareConferenceReports` (imports `ModalField`, `ResidentSingleSelect`, `FamilyMultiSelect`, `ConflictPanel`, `JoinByPhoneSection`, and the meeting-type/duration/location constants directly from it — this is a second view over the same data, not a separate scheduling model).
- **Month view:** a 7-column grid; each day shows up to 3 conferences (grouped by identical start time, side-by-side when tied) with a "+N more" expand/collapse toggle; clicking a day switches to day view. **Week/day views:** an hour-by-hour grid (7 AM–10 PM) with overlap-safe event layout (same-start-time events split into columns; different-start-time overlaps stack vertically without visual collision — the same layout algorithm used by the Transportation calendar).
- **Status colour legend** matches the conference status machine: Scheduled (blue), In Progress (orange), Review (purple), Cancelled (red) — `COMPLETED` conferences are fetched (the calendar requests `SCHEDULED,IN_PROGRESS,IN_REVIEW,CANCELLED` explicitly, notably **omitting `COMPLETED`** from its own status filter) but have no assigned colour in the legend, an inconsistency to confirm with product before relying on the calendar to show completed conferences.
- **Event detail popup** (click any event): resident profile photo/name/room, status badge, date, time range, location (with `JoinByPhoneSection` if the location is "Phone"), a **care-team roster** — care team + auto-derived family members + the conference host, the host's row tagged "(Host)" and folded into the same list rather than shown separately — with an expand/collapse "view all" past 8 members, plus the agenda and post-meeting notes/summary (each showing an explicit "No agenda/notes added" placeholder when empty, not a blank field).
- **Edit and delete directly from the calendar**, using the same validated form fields as the list screen (resident, care team, family members, date/time, duration, location, agenda) and the same conflict-detection/conflict-panel behavior; delete requires a confirmation dialog. **There is no "create new conference" action on the calendar itself** — new conferences are still scheduled from "Schedule Care Conference"; the calendar is a browse/edit/delete surface layered on the same records, timezone-aware to the facility's configured time zone for "today" highlighting and the default landing date.
- Data source: `GET /care-conference?limit=1000&status=SCHEDULED,IN_PROGRESS,IN_REVIEW,CANCELLED&startDate&endDate`, refetched on navigation between days/weeks/months; update via `PUT /care-conference/{id}`, delete via `DELETE /care-conference/{id}` (same endpoints as the list screen — no new backend surface for the calendar).

**Notable endpoints (Care Conference group, both views):** `GET /care-conference?limit&status[&startDate&endDate]`, `POST /care-conference`, `PUT/DELETE /care-conference/{id}`, `PUT /care-conference/{id}/update-summary { summary, shareWithResident }`. No PDF export on either view. **Neither view has an explicit frontend permission gate** (`usePageAccess` absent) — same as the prior pass; access relies on nav-level filtering plus backend enforcement (see `docs/prd/modules/care-coordination.md` §7/§9 O-7).

### 2.12 Referrals & Agencies — now **"Home Health Referrals"** (`Referrals.tsx`, `GenerateReferralForm.tsx`, `ReferralPrintView.tsx` (new), `DynamicReferralFields.tsx`, `config/referralFormConfig.ts`)

**Purpose:** discharge-planning referrals from the attending physician to external home-health agencies, plus the agency directory. **Re-platformed this interval from email delivery to fax delivery via WestFax**, with substantial hardening around delivery visibility and document management. Nav label changed **"Referrals" → "Home Health Referrals"**.

- **Referral content (IReferral):** unchanged core shape — resident (+DoB, doctor/physician cName), selectedHHA/selectedAgency, dischargeTo + dischargeDate, home-services checklist, additional orders, physician certification (signature/date/printed name/license number). **Terminology:** "Doctor" replaced by "Physician" throughout; the doctor/physician staff picker is now **filtered to physician-designated staff only**.
- **Discharge Date backdating window (new).** The Discharge Date field now enforces a facility-configured maximum backdate (`appConfig.dischargeDateMaxPastDays`, client default 28 days if unset — the backend is the actual source of truth at submit time) instead of accepting an arbitrary past date; a previously-reported off-by-one on the discharge date (local-date parsing in the print view) is fixed.
- **Statuses renamed:** `Incomplete | Pending Signature | Ready to send | Sent` (was `isDraft`/`Pending`/`In Doctor Review`/`Doctor Approved`/`Sent`).
- **Fax delivery — WestFax re-platform (replaces the SES-email "Send to Agencies" path):**
  - The send-to-agencies modal now shows a **per-agency preview of "what's new"** (`POST /referrals/:id/sent-history/preview`) — only documents the agency hasn't already received are faxed, and agencies with no fax number on file are blocked from selection.
  - **Sending** creates a sent-history record and triggers the fax (`POST /fax/westfax/send`), fire-and-forget — delivery status lands later via the WestFax webhook updating a `FaxLog` entry per agency/document, not synchronously in the response.
  - A new **Fax Delivery History** modal (`ReferralHistoryModal`) shows per-document DELIVERED/FAILED/PENDING status with **per-agency retry** (`POST /fax/westfax/retry`) and **7-second polling** while any item is still pending, plus optimistic UI feedback on retry.
  - **Document CRUD** in the send-referral modal: attach, rename, delete, and download additional supporting documents (beyond the referral PDF itself) before sending; **Select-All/Clear-All** for document selection.
  - The referral's **medication list is now a separately-signable document** (its own `SignatureProgressChips` + `use-download-referral-medication-list.ts`), not bundled implicitly into the main physician certification signature.
  - The prior SES-email send path (`use-send-referral-emails.ts`) was **deleted outright** — there is no email fallback for agency delivery any more.
- **List & search:** the referral list gained **server-side pagination** and a **status filter** (`use-fetch-referrals.ts` sends `page`/`limit`/`search`/`status` — was client-side over a bulk fetch). Table columns: Resident, Physician Name, Status, Actions — the **Agencies column was removed**. `signedPdfUrl` is now surfaced: when present, the already-signed PDF is shown instead of regenerating one.
- **Print view:** a new standalone **`ReferralPrintView.tsx`** renders the discharge-orders + physician-certification print output (accepts `{ facility, form, dynamicFields, config }`), used from `Referrals.tsx`'s print action; **Physician Certification now prints unconditionally** (previously conditional); `pcpFollowUp` was removed from the form while `additionalNotes`/`includeMedicationList` were added; DOB is now backfilled from three separate sources rather than one.
- **Agencies CRUD** still lives here: name, email, phone, faxNumber, specialties[], address, Active/Inactive — via **`authApi`** (the second backend). A `MOCK_AGENCIES` array remains in `Referrals.tsx` as scaffolding, and the older mock-driven `GenerateReferralModal` (which read from it) is now **dead/unreachable** (its open-state is never set) — a delete candidate, not a live fallback.
- Deletion: `DELETE /referrals/{id}` (added this interval, via `use-delete-referral.ts`).

**Key data shape — `IReferral` (abridged, updated fields only):**

```typescript
{ ...prior shape,
  signedPdfUrl?, assignedPhysician /* Staff _id, replaces free-text doctor pick */,
  status: 'Incomplete'|'Pending Signature'|'Ready to send'|'Sent' }
```

**Notable endpoints (Referrals cluster, new/changed):**

| Operation | Method & path |
|---|---|
| List (server-paginated, status filter) | `GET /api/referrals?search&status&page&limit` |
| Delete | `DELETE /referrals/{id}` |
| Sent-history preview | `POST /referrals/:id/sent-history/preview` |
| Send fax | `POST /fax/westfax/send` |
| Retry fax | `POST /fax/westfax/retry` |
| Fax delivery history | (consumed via `ReferralHistoryModal`, polls the sent-history/fax-log endpoints) |
| Document CRUD | `use-upload/rename/delete/download-referral-document` hooks |
| Agencies | `GET/POST /api/agencies`, `PUT/DELETE /api/agencies/{id}` (authApi) |

No explicit frontend permission gate on Referrals (unchanged — see `docs/prd/modules/referrals.md` for the full referral-module PRD, including the backend-side gaps this UI surface sits on top of).

### 2.13 Announcements (`Announcements.tsx`)

No functional changes this interval. Time-windowed broadcasts; `single | multiple | range` display types; `startTime`/`endTime` fields feeding the backend's 1-hour-before reminder; `resident | family | both` audience; 16 preset icons or an uploaded/gallery image; no draft state; soft delete; feeds the NotificationPanel's "today's announcements" section.

### 2.14 Messages / Chat (`components/Message/`, `chatSocket.ts`, `chatApi.ts`, `hooks/chat/*`)

**Purpose:** full internal chat — DIRECT (1:1) and GROUP conversations among staff, residents, and admins. No frontend permission gate on the Messages nav item itself; **which staff designations may use chat at all is now facility-configurable** (`config/chat/staff-designation-allowed`, §2.2/§2.1). This is the module with the most churn in this interval — it shipped as a re-architected, domain-driven module partway through the window and then gained a large feature set on top of that foundation, and it is also now an **installable PWA** scoped to `/messages`.

**Foundation (carried over, unchanged in shape):** `Message/domain/*` (pure logic: conversation previews, reaction reducer, optimistic-send builder, system-event labels, attachment-type resolution, status ranking), `Message/message/*` + `Message/attachment/*` (presentational), `hooks/chat/*` (the React Query cache + Zod-validated socket-reconciler layer — every real-time event flows through one `applyServerEvent` reconciler rather than a hand-rolled buffer). Conversations are `DIRECT` (peer participant) or `GROUP` (name/picture/admins/participant previews); messages carry text and/or `image|video|audio|document` attachments (per-type size/count caps from facility config), @mentions (GROUP only, Lexical typeahead, now including **resident** candidates via `GET /chat/mention-residents`, server-scoped `STAFF_ASSIGNED | FACILITY_ALL | NONE`), reactions, replies, and `SENT < DELIVERED < READ` receipts.

**Installable PWA (new this interval).** The `/messages` popup window is now installable as its own app ("Shashi Messaging" / re-iconed "Shashi Care" mid-window) via a service worker scoped **only to `/messages`**, not the admin panel root — deliberately, so the install prompt never appears on the main admin surface. Cross-window sync (badge/read/focus/active-conversation signals) moved from `window.opener`-targeted messaging to a same-origin `BroadcastChannel`, reaching every open admin tab and the installed PWA regardless of how the window was opened. Push notifications (VAPID) are wired only through the popup, not the main admin shell — a user who never opens the chat popup gets no push subscription (see §4.5/Design Gaps).

**Editing, forwarding, drafts (new):**
- **Edit** a sent text message (sender's own messages only) — reopens the composer pre-filled with the original content; edit is hidden once the facility's configured edit window has elapsed; an "Edited" label appears on the bubble.
- **Forward** one or more messages to one or more targets (existing conversations, groups, or a brand-new DM) in a single action, fully optimistic with per-target rollback if an individual send fails.
- **Per-conversation drafts** — unsent composer text now survives closing the tab/window (persisted to localStorage), though drafts are not synced live across open windows (each re-merges from storage independently).
- A hover **Copy Text** option, sent-message notification sound disabled, and a corrected "Jump to Present" unread badge (now derived from the live thread rather than an accumulating counter).

**Two independent pin concepts (new — do not conflate):**
- **Conversation pin** (per-user, device-local-only preference) — pins a conversation to the top of *that user's own* sidebar; never visible to or shared with other participants.
- **Message pin** (new, shared) — any participant can pin or unpin a single message **for the whole conversation**, visible to everyone in it, with an optional duration (a facility-configured list of durations, or "Forever"). A pinned-message tray shows: an inline "Pinned" indicator on the bubble itself, a banner surfacing the single newest not-yet-expired pin (click to jump to it, click again to cycle to the next-older one), and a "show all pinned" side panel with per-row unpin and (once >3 pins exist) search. Real-time pin/unpin/expiry updates broadcast to every participant, distinct from the conversation pin's device-local-only sync channel. **Per-user and per-conversation pin caps are enforced by the backend**; the client only dims the pin action and shows a (deliberately number-free) "limit reached" tooltip as a UX shortcut — a stale client-side count can still be rejected server-side.
- A **jump-to-any-historical-message** capability was built alongside message pinning — first used by "go to message" from the pin banner/panel, but written as a general-purpose primitive for future reuse (e.g. a future message-search "jump to result").
- **Known-incomplete, flagged explicitly by its own commit message:** a same-window diagnostic pass at improving "does the conversation land exactly at the bottom on open / after sending / during rapid message arrival" scroll behavior shipped fixes for three separately-reported cases, but the author's own commit message states the underlying issue is **not fully resolved in all cases** — treat as an open investigation, not a shipped fix, in any release note or ticket referencing it.

**Per-conversation management (new/changed):**
- **Clear Chat (groups) / Delete Conversation (direct)** — a per-user action on the conversation list that hides the thread for the acting user only (`PUT /chat/conversations/:id/clear`); does not delete the conversation for other participants, and (per the backend's parallel documentation) does not physically purge message content — a HIPAA/CA 7-year retention rationale.
- **Message Info panel** — per-recipient Delivered/Read status with a tabbed view plus a flat alphabetical "All recipients" list; opened from a message's action menu.
- **Deleted-message tombstone** ("You deleted a message" / "Message was deleted") shown in the conversation-list preview; a hover-reveal delete action on conversation-list rows.
- **`maxGroupMembers`** enforced numerically from facility config when creating or adding to a group.
- Clicking a staff/admin @mention inside a message opens a **profile panel** (`UserInfoPanel`) for that person.
- **URL linkification** — plain URLs in message text now render as clickable links.
- **Share to chat** — the IDT Report screen (§2.10) can forward a report PDF into a chosen conversation via a dedicated conversation-picker modal, with a prefilled composer message and an @mention of the resident.

**Real-time (`chatSocket.ts`):** unchanged in shape — Socket.IO `/chat` namespace singleton, two-tier callback registry (global = AppContent badges/toasts; page = open Messages view), exponential-backoff reconnect, proactive token refresh. New/changed inbound events this interval: `chat:edited`, `chat:message-pin-changed` (broadcast to every participant — distinct from the pre-existing device-local `chat:pin-changed` for conversation pins), and `chat:cleared` (drives the per-user Clear/Delete-conversation UI).

**Loading & search:** unchanged — cursor-paginated infinite scroll, conversation-level-only search (no full-text message search); the conversation sidebar list is now virtualized for performance under large inboxes.

**Group management:** unchanged — create/update/delete (creator only)/add-remove members/promote-demote admins.

**Key endpoints (unchanged core + new):** `GET /chat/conversations?limit=50`, `GET /chat/conversations/{id}` (+`/messages`, `/info`, `/attachments`, **`/pinned-messages`** (new)), `POST /chat/messages`, `DELETE /chat/messages/{id}`, `PUT /chat/conversations/{id}/read`, **`PUT /chat/conversations/:id/clear`** (new), **`PUT`/`DELETE /chat/messages/:messageId/pin`** (new), `GET /chat/mention-residents` (new).

**Unread model:** unchanged — sidebar badge counts distinct conversations with `unreadCount > 0`; `chat:unread` socket snapshot is authoritative on connect/reconnect.

### 2.14b In-app notifications (`NotificationPanel.tsx`, `notificationSlice.ts`)

No functional changes this interval. Bell-icon panel combining Redux-stored notifications (socket/in-app pushed, local-only read state, no server persistence) with today's announcements fetched live.

### 2.15 Dashboard (`DashboardOverview.tsx`)

No functional changes this interval. 4 stat cards (Total Residents, permission-gated Pending Requests aggregate, Today's Activities, current-month Care Requests), Upcoming Appointments feed, ADMIN-only Recent Activity feed (`GET /unified-schedule/recent-activity`), permission-filtered Quick Actions. No charts here — see §2.18 KPI Dashboard for the app's real charting/analytics surface.

### 2.16 Reports (`ReportsOverview.tsx`)

**Still effectively dead** — empty `subItems: []` nav parent, and the page content remains 100% hardcoded mock data (Recharts tabs over fake occupancy/care-type/services/wellness numbers). **This interval's real answer to "give admins live operational metrics" is the new KPI Dashboard (§2.18), which is a separate surface (header button + Settings tab) and does not replace or wire into this page.** The Activity Attendance Report (§2.3) and clinical documents remain the only other live "report" surfaces.

### 2.17 Settings (`SettingsPage.tsx`)

No structural changes to the existing Account/Accessibility tabs this interval (still: Account persists profile photo + name/email + embedded `ChangePassword`; Accessibility persists font size/high contrast/read-aloud with "Read Aloud" still having no audio implementation). Two additions:
- `countryCode` (from the staff/admin profile) is now displayed in the Account tab; the email field is read-only until focused (avoids browser autofill conflicts) and format-validated before save.
- **A new "KPI Dashboard" tab** renders inside Settings, gated by `usePageAccess("KPI Dashboard").canView` — this is the second of the two entry points into §2.18 (the other being the admin-only header button).

The staff designation is now shown in the header profile dropdown (in place of the generic role label, when a designation is present). Standalone Notifications page (`NotificationSettings.tsx`) is unchanged — still toast-only, no backend persistence.

### 2.18 KPI Dashboard (`KpiDashboardTab.tsx`, new)

**Purpose:** the first real, API-backed analytics surface in the admin app (as distinct from the dead mock Reports page, §2.16). Reachable two ways: an **admin-only** header quick-access button (`isAdmin`-gated) opening a full-screen dialog, and a **Settings tab** gated by the ordinary `usePageAccess("KPI Dashboard")` permission (so a facility could in principle grant a non-admin staff member the Settings-tab view without the header shortcut, or vice versa for an admin — §1.3).

- **Date-range control:** a from/to date picker (default: trailing 14 days) with an explicit "Apply" button — changing the fields does not refetch until applied; a spinner shows while a background refetch is in flight.
- **Resident & Messaging Overview** tile row: Residents Onboarded (with a PCC-vs-Manual breakdown caption), Residents Discharged, Unique Conversations, Active Messaging Groups.
- **Care & Referral Activity** tile row: Secure Calls Made, Referrals Sent, Documents Signed.
- **Workflow Snapshot:** grouped bar charts for Transportation Requests and Care Conferences (created/updated/served counts), an Onboarding Source donut (PCC vs Manual), and a gradient "Total Messages Sent" hero card (message + conversation volume, normalized bars, staff-member count).
- **Staff Messaging Activity:** two ranked leaderboards (Messages Sent by Staff, Conversations by Staff) with a Chart/Table view toggle per card; the chart view truncates to the top 8 entries (grouping the remainder is supported by the underlying ranking utility but not currently exercised in these two cards).
- **Data source:** a single `GET /reports/daily-summary?date=<from>&endDate=<to>` call — query-time aggregation, not a cron/materialized store; facility-scoped.
- **Facility Occupancy tiles were attempted and reverted the same day** they were added (a fourth tile row — Total/Occupied/Available Beds + Occupancy Rate — referencing a data field the backend endpoint never actually returned) — **occupancy is not part of the KPI Dashboard as shipped**; do not describe it as available in any downstream release note.
- Full requirement-level detail (data-source aggregation logic, facility scoping, the reverted-occupancy history) is documented in `docs/prd/modules/dashboard-reporting.md` DSH-FR-23 — this entry describes the admin-web-visible behavior only.

### 2.19 Resident Documents & Advance Care Directives (`ResidentDocuments.tsx`, `CallSummaryModal.tsx`, new — Documents tab of `ResidentDetails.tsx`, §2.1)

**Purpose:** a new PHI-adjacent surface merging two record kinds into one per-resident feed: uploaded **advance care directive** documents and **AI-summarized secure-call transcripts** (recorded staff↔resident/family phone calls, reviewed here for the first time outside the staff mobile app).

- **Directives:** list/sort/filter uploaded directive files; upload supports **image-to-PDF conversion** at upload time (so a photographed paper directive becomes a viewable/printable PDF) and an optional **"Send to Physician for Verification & Signature"** routing at upload time (physician picked from staff filtered to the "Physician" designation) — this feeds the unified physician e-signature queue also used by referrals (`/api/signatures`).
- **Secure calls:** rows open `CallSummaryModal`, which shows the AI-generated call summary alongside the full transcript and an **Approve** action that formally marks the reviewed summary approved — the first place this is reviewable from the admin portal rather than only on a staff member's phone.
- A print/download document viewer is provided for both record kinds.
- **Endpoints:** `GET advance-care-directives/admin/resident/:id` (combined feed), `POST advance-care-directives/staff` (upload), `PATCH advance-care-directives/:id/view` (mark-viewed), `GET secure-calls/:id` (transcript/summary detail), `PUT secure-calls/:id/update-summary` (approve).
- **This is new PHI-adjacent surface whose facility-scoping/role enforcement on these five routes had not been independently re-verified against the backend at the time this admin-web pass was written** — see `docs/prd/modules/clinical-records.md` (CLIN-FR-26, CLIN-GAP-21) for the backend-side gap this UI sits on top of (the mark-viewed and admin-resident-feed routes currently carry only generic auth, no staff/admin role gate) before treating this surface as fully access-controlled.
- The Documents tab replaced a previously-tabbed `ResidentHealthRecords` component, whose import is now commented out in `ResidentDetails.tsx` pointing at a file that no longer exists in the repo (§4.1).

### 2.20 Consent Forms (`SignConsentFormModal.tsx`, `ConsentFormPreview.tsx`, `ui/SignatureCanvas.tsx`, new — reached from `ResidentDetails.tsx`, §2.1)

**Purpose:** in-portal HIPAA authorization workflow letting a resident (via the admin acting on their behalf, in person) authorize named family members to access the resident's health information through the resident/family-facing apps — a resident-side consent artifact, not a staff/admin-facing agreement.

- **Step 1 — select signers (`SignConsentFormModal`):** pick which of the resident's family members are being authorized on this form ("Select All" or individually); before proceeding, the client validates the selection against `POST /consent-forms/validate-members` — if a chosen family member is **already linked to another resident's signed consent form**, a blocking dialog explains a family member can only be tied to one resident's signed consent at a time (a family member cannot be double-authorized across residents). An unexpected validation error **fails open** (proceeds anyway) rather than blocking the admin on an infrastructure issue.
- **Step 2 — preview & sign (`ConsentFormPreview`):** renders a print-styled HIPAA authorization document (facility name, the selected family members in a numbered table with name + relation, the standard authorization language) with a **dependency-free canvas signature pad** (`SignatureCanvas`) capturing the resident's signature; today's date is fixed and non-editable. Submitting requires a captured signature; a confirmation dialog guards navigating away/cancelling once a signature has been drawn.
- **On successful sign:** the record is persisted, the signed PDF opens in a new tab, and — per the backend's documented behavior — **signing optimistically grants the selected co-signers portal/app access** rather than requiring a separate manual access-grant step.
- **Endpoints:** `GET/POST /consent-forms`, `POST /consent-forms/validate-members`.
- Full requirement-level detail (the PHI-auth gap on 3 of 5 consent-form routes, the co-sign access-grant mechanics, and per-signature PDF regeneration) is documented in `docs/prd/modules/clinical-records.md` CLIN-FR-25 / CLIN-GAP-20 — this entry describes the admin-web-visible flow only.

---

## 3. Product-split signals (every facilityType / SKILLED_NURSING conditional found)

The SNF vs AL product split is mostly **data-driven** — the facility-pages API (Filter 1, §1.3) decides which nav items exist per facility — with a small, and this interval **shrinking**, number of code-level conditionals.

**Change this interval: the Residents list's SNF/AL column split was retired.** The prior split (SNF: Resident/Room + a single Assigned Staff column; AL: Name + Unit No + Care Type) has been replaced by **one unified column set for every facility type** (Resident/Room "Room No" + Type of Admission + Length of Stay + Physician, §2.1) — `ResidentsManagement.tsx` no longer branches on `facilityType`/`careType` at all. This removes what was previously the single largest code-level product-split point in the admin app; the Care Type filter/enum distinction (`assisted_living`/`memory_care`/`independent_living`/`skilled_nursing`) still exists on the resident record and elsewhere in the platform, just not as a list-view branch here any more.

| # | Location | Conditional behavior |
|---|---|---|
| 1 | `AppContent.tsx:109-222` (BASE_NAVIGATION) | Rehab parent contains both AL sub-pages (therapy-evaluations, physical-therapy, cognitive-sessions, private-training, outside-agency-services) and SNF sub-pages (rehab-calendar, -appointments, -service, -message, -team, my-availability, idt-report). Per-facility selection happens via Filter 1, not code. |
| 2 | `AddResidentModal.tsx`, `EditResidentModal.tsx` | `defaultCareType = facilityType.toLocaleLowerCase()` — resident care type pre-set from facility type and disabled in forms. |
| 3 | `IDTReport.tsx:159, 331, 402, 530-532` | IDT report's facility-name field auto-filled from localStorage `facilityType`; display name derived by title-casing the enum ("Skilled Nursing"). |
| 4 | `RehabCalendar.tsx`, `MySchedule.tsx` | PDF/print headers derive the facility display name from `facilityStorage.getFacilityType()` (title-cased snake_case). |
| 5 | `constants/rehab.ts` | `ASSISTED_LIVING_THERAPY_TYPES = [PHYSICAL_THERAPY, COGNITIVE_EVALUATION, REHAB_EVALUATION, OUTSIDE_AGENCY]` — the AL-side therapy vocabulary, consumed by `TherapyEvaluations.tsx` and `OutsideAgencyServices.tsx`. Labels only defined for PT + Cognitive. |
| 6 | `hooks/use-config.ts` | `facilityType` from `GET config` validated against the two-value enum and persisted (localStorage + Redux facilitySlice). |
| 7 | `utils/facilityStorage.ts` | Canonical `FACILITY_TYPE = { SKILLED_NURSING, ASSISTED_LIVING }` — the only recognized values. |

**Retired this interval:** the former Residents-table SNF/AL column split and the Care Type filter dropdown (both removed with the unified column rework, §2.1); the AL-specific "Unit No" room label (now always "Room No").

**Implications for the product split:**
- Concrete code divergence points are now only six: the Rehab navigation set, the AL therapy-type vocabulary, the resident-form default care type, and three document/print headers. The Residents list itself — previously the most visible split — is no longer one of them.
- Two parallel scheduling backends still exist along the split: the AL-side generic **`/care`** engine (type-discriminated) vs the SNF-side **`rehab/appointments`** engine — a structural duplication any unified product would need to reconcile (unchanged).
- Resident-level care types (`assisted_living`, `memory_care`, `independent_living`, `skilled_nursing`) remain a **four-value** taxonomy, while facilityType is a **two-value** facility enum — Memory Care and Independent Living exist only as resident care types, not facility variants (unchanged).
- The Care Conference Calendar (§2.11), KPI Dashboard (§2.18), Resident Documents/ACD (§2.19), and Consent Forms (§2.20) are all **facility-type-agnostic** in code — same as Chat, Staff/Access, Dashboard, and Settings, they are differentiated purely by which pages the facility-config API enables (Care Conference is nominally SNF-flavored per its product framing in `docs/prd/modules/care-coordination.md`, but nothing in the admin-web code itself branches on facility type for it).

---

## 4. Observations (dead code, mock pages, half-built features, inconsistencies)

### 4.1 Dead components & files (not routed / unused)

Verified by import analysis (0 importers unless noted):

- **`Housekeeping.tsx`** — static mock weekly schedule, no API, never imported. The live housekeeping features are the four request-queue screens.
- **`TransportManagement.tsx`** — mock data only, no hooks, not routed.
- **`ActivitiesEvents.tsx`** — 0 importers.
- **`ViewResidentModal.tsx`** — superseded first by `ViewResidentModalNew.tsx`, and now by the full-page `ResidentDetails.tsx` as the primary detail surface (§2.1); both modal variants remain in the codebase but are no longer the primary flow.
- **`AddServiceModal.tsx`** — 0 importers, yet contains a live facilityType conditional (copy-paste survivor).
- **`CalendarPdfPreview.tsx`** — 0 importers (PDF preview logic exists inline in MySchedule).
- **`ForgotPassword.tsx`** — 0 importers; superseded by `StaffForgotPassword.tsx` (§1.5).
- **`ComingSoon.tsx`** — orphaned; `transport-calendar` renders the real `TransportCalendar.tsx`.
- **`AccessibilitySettings.tsx`, `IntegrationSettings.tsx`** — created during an earlier Settings split, then de-integrated; imported nowhere.
- **`SidebarUserProfile.tsx`** — imported but its render is commented out in `AppContent.tsx`; not currently shown in the sidebar footer.
- **New this interval — `ResidentHealthRecords` reference in `ResidentDetails.tsx`.** A `healthRecords` tab id remains declared and its import is commented out, pointing at a component file that **no longer exists in the repo** (deleted before this pass). Harmless while commented out, but the stale reference (import + unused `TABS` entry) should be deleted rather than left as a breadcrumb.
- **New this interval — `PaymentHistoryModal` import/render in `ResidentsManagement.tsx` is unreachable.** The trigger button and its handler (`handleViewPaymentHistory`) were removed while the modal component and its import remain wired in — dead on arrival unless the button is reinstated (§2.1).
- **New this interval — the mock-driven `GenerateReferralModal`** (built on `MOCK_AGENCIES`) is now dead/unreachable — its open-state is never set anywhere in `Referrals.tsx` (§2.12).
- **Duplicate hook pairs (`.ts` vs `.tsx`, same default export name, different signatures)** — unchanged from the prior pass: `use-fetch-residents.ts`/`.tsx`, `use-fetch-housekeeping.tsx`/`.ts`, `use-get-staff.ts`/`.tsx`.
- **`.codex`** — an empty 0-byte file committed at the repo root.
- **`HotelDemoSlideshowModal.tsx`** (new, not dead but flagged for scope) — a config-gated (`facilityConfig.showSlideShowModal`) sales-demo slideshow tool, unrelated to resident-care operations; opened automatically from `AppContent.tsx` when the flag is set. Worth confirming ownership/scope with product before extending it further — it sits oddly among resident/clinical/operational modules.

### 4.2 Mock-data & decorative surfaces

- **Reports page**: entirely hardcoded numbers/charts *and* unreachable from nav (empty subItems) — double-dead (§2.16). The new KPI Dashboard (§2.18) is a genuinely live analytics surface but is a separate entry point, not a fix to this page.
- **Settings**: Account tab genuinely persists (name/email + cropped profile photo); the Notifications page toggles and the Accessibility-tab "Save" remain decorative (toast-only, no backend sync); "Read Aloud" persists a flag with no audio behavior.
- **Referrals `MOCK_AGENCIES`** — mock array still in source next to the live agencies API; its one former consumer (`GenerateReferralModal`) is now dead code (§4.1).
- **SendEmailModal** (Residents): component complete but never opened — unchanged from the prior pass.

### 4.3 Half-built / stubbed features

- **Family meal request approval**: still no approve/reject UI — configuration-only (pricing + blackout dates). Meal-period time windows remain config data with no edit UI.
- **Transport driver filter**: dropdown UI exists; query param remains commented out pending backend support (unchanged, the repo's only TODO-comment site).
- **Transport price auto-calc**: `pricePerMiles` config exists but is never used; ride price is manual entry (unchanged).
- **`Started` transport status**: still declared in the status union, unreachable in UI.
- **IDT auto-save**: still fully written but commented out (unchanged).
- **AllDayMenu**: drag-drop reordering and item-description edit UI remain commented out (unchanged).
- **Salon "Store Details" card**: still commented out while the edit modal remains live (unchanged).
- **Therapy Evaluations / PT / Cognitive / Outside Agency**: still create + view only — no edit/delete/reschedule anywhere in the wellness `/care` cluster (unchanged).
- **Resolved this interval — discharged residents were previously unfilterable.** The Residents status filter now offers Active and Discharged explicitly (default Active); this closes the specific gap the prior pass flagged, though "Away" (still a settable status via Edit) is not offered as a filter option and there is no combined "all statuses" view besides the two explicit choices.
- **New this interval — Payment History has no UI trigger.** See §2.1/§4.1 — was previously a live, if simple, feature; now unreachable pending a product decision on whether to reinstate the button.
- **Still no reschedule/cancellation-reason flow** in any appointment module; no appointment history view in wellness modules (unchanged).

### 4.4 Cross-cutting inconsistencies

- **Two Axios bases**: unchanged — `api` (`VITE_PROD_URL`, though this branch's build config actually reads `VITE_PREPROD_URL`/`VITE_SOCKET_PREPROD_URL`, §4.5) vs `authApi` (`VITE_PROD_AUTH_URL`) with a hardcoded `x-facility-id` fallback of `"R101"`, used for agencies/referrals/OAuth-link calls.
- **Copy-paste query-key bugs**: unchanged (`use-delete-staff.ts` invalidates `['get-activities']` instead of `['get-staff']`; MySchedule's delete has the analogous bug).
- **`POST /residents/send-credentials` reused for staff** credential sends — unchanged, misleading endpoint naming.
- **Endpoint typo in a production path**: `PUT staff/availabilty/{cName}` (sic) — unchanged.
- **Pagination meta drift**: unchanged (`totalPages`/`pageCount`, `total`/`totalCount` inconsistency across domains); default page size is now 50 in most list views (was 10/30) via the shared `RowsPerPageSelect`.
- **Status-case conventions differ by domain**: unchanged.
- **`agencyName` as a free string** in outside-agency `/care` appointments — unchanged, still not an agency FK.
- **Cross-module endpoint reuse**: private-training slot lookup still calls the salon slots endpoint — unchanged.
- **Date/timezone handling inconsistent by design pocket**: unchanged (Rehab calendar UTC date-keys; wellness calendars UTC dates + local times; housekeeping via facility `timeZone`; meal times raw). The new Care Conference Calendar (§2.11) follows the facility-`timeZone` pattern (like Transportation/Housekeeping), not the Rehab calendar's UTC-date-key pattern.
- **Permission-key casing is rescued by normalization**: unchanged; **"KPI Dashboard"** joins the list of permission keys passed to `usePageAccess` (Appendix B).
- **Modules with no frontend permission gate**: IDT Report, both Care Conference views (Schedule + Calendar), Referrals, and Messages still rely on nav-level filtering + backend enforcement only.
- **Resident `Status` enum mismatch**: unchanged in the type declarations (`Active|Away|Discharged` enum vs `Active|Away|Inactive` badge cases) — now partially addressed at the *filter* level (Active/Discharged are both selectable), but the underlying type/badge inconsistency itself was not touched.
- **Edit-phone asymmetry**: unchanged (resident's own phone immutable post-creation; emergency contact was editable, but the emergency-contact field itself was removed from the Edit form this interval — see §2.1 — narrowing this asymmetry's practical surface rather than resolving it).
- **Service-vertical triplication**: Salon / Massage / Private Training remain three near-identical implementations — unchanged.
- **New this interval — two contradictory phone/email-uniqueness behaviors shipped and un-shipped in the same window.** A family-member phone-uniqueness cross-validation was added, then removed and replaced by a resident-level soft duplicate-warning (`GET /residents/check-duplicate`) a few weeks later (§2.1) — see PLAT-FR-73/73a for the full sequencing; treat "duplicate phone/email is blocked" as **false** for the current build.
- **New this interval — the Care Conference Calendar's status filter silently omits `COMPLETED`.** The calendar's own data fetch requests `SCHEDULED,IN_PROGRESS,IN_REVIEW,CANCELLED` and has no legend entry for a completed conference, while the sibling list screen's History tab does show completed conferences — the two views of the same underlying data therefore disagree on whether a completed conference is visible (§2.11).

### 4.5 Security & quality flags

- **Phone number logged to the console on every login-page render remnant** (`helpers.ts:74`) — unchanged; the broader `chatSocket.ts` unguarded-log issue from the prior pass remains resolved (DEV-gated).
- **MFA reset endpoint** (`POST /residents/resetMFA { email }`) still callable from the unauthenticated login screen with no client-side restriction.
- **Token refresh buffer mismatch**: unchanged (`tokenService.ts` 1-minute buffer vs. a 5-minute code comment; MainApp uses a real 5-minute buffer).
- **Cognito client secret ships in the frontend bundle** (`VITE_COGNITO_CLIENT_SECRET`) — unchanged, inherent to the chosen Cognito flow (ADMIN-1 in the platform risk register).
- **Default facility fallback `"R101"`** in `authApi` — unchanged.
- **New this interval — build config drift.** `api.ts` and the chat/notification sockets on this branch read `VITE_PREPROD_URL` / `VITE_SOCKET_PREPROD_URL` rather than the documented `VITE_PROD_URL` / `VITE_SOCKET_PROD_URL` — a pre-production build artifact that must be reconciled before this branch promotes to `production`.
- **New this interval — `GET /residents/check-duplicate` is unauthenticated and facility-optional** (per the backend-side review referenced in PLAT-FR-73a) — an omitted `x-facility-id` header checks across every facility rather than being rejected; the response is boolean-only (no PHI payload) but the tenancy-isolation pattern is the same class of gap as the platform's BE-1/BE-2 Blockers.
- **New this interval — Resident Documents/ACD and Consent Forms sit on backend routes not yet fully role-gated** — see §2.19/§2.20 and `docs/prd/modules/clinical-records.md` CLIN-GAP-20/21 for the specifics; flagged here because the admin-web surface itself has no additional gating beyond nav-level filtering.
- **RehabTeam 60s availability interval**: cleanup-on-unmount still not observed — unchanged.
- **No test files** anywhere in the analyzed module set; the new chat `domain/*` and `hooks/chat/*` modules added this interval (pin, forward, drafts, jump-to-message) are likewise untested. New deps this interval: `react-easy-crop`, `react-is`, `@lexical/link`, `@radix-ui/react-context-menu`.
- **Notifications are Redux-local only** — unchanged, lost on reload.
- **Prices carry no currency/precision handling; durations carry no unit validation** — unchanged.

### 4.6 Reusable platform patterns worth formalizing in the PRD

1. **Recurrence vocabulary** — one-time / weekly / multiple-dates / date-range / everyday — shared by Activities, Menu items, Specials, and Announcements. Unchanged; should be specified once as a platform scheduling primitive.
2. **imageKey/imageUrl contract + shared gallery** — unchanged, a platform media service in all but name.
3. **Service-vertical template** (Salon / Massage / Private Training) — unchanged, still three copy-paste instances.
4. **Request-queue template** (4 housekeeping types; transport approval a close sibling) — unchanged.
5. **`/care` generic appointment engine** vs the SNF `rehab/appointments` engine — unchanged, still two parallel scheduling systems.
6. **requireEdit()/usePageAccess read-only pattern** — unchanged, universal write-gating UX.
7. **cName as the person key** — unchanged, threads through residents, staff, bookings, chat, OAuth linking, and now consent forms / resident documents / KPI-dashboard staff leaderboards.
8. **New this interval — the "shared list/detail/edit component" pattern between a screen and its calendar sibling.** `CareConferenceCalendar.tsx` deliberately imports its form fields, validation, and conflict-detection UI straight from `CareConferenceReports.tsx` rather than re-implementing them, and the day/week overlap-safe event-layout algorithm is explicitly reused from the Transportation calendar. Worth calling out as the pattern to follow if/when a similar calendar view is added for another appointment type (e.g. Rehab or Salon), rather than each calendar re-deriving its own layout math.
9. **New this interval — a two-step "select participants → sign on a rendered document" flow** now exists in two places built independently: Consent Forms (§2.20, resident/family) and the physician e-signature surface behind Resident Documents/Referrals (`/api/signatures`, staff/physician-facing). Worth a platform-level "signable document" component/pattern rather than continuing to build each signing surface bespoke.

---

## Appendix A — Shared API & data conventions

**Response envelope (most `api` endpoints):**

```typescript
{ success: boolean, data: T[] | T,
  meta?: { page, limit, total | totalCount, totalPages | pageCount, confirmedCount? } }
```

- Pagination params are `page` (1-indexed) + `limit` everywhere; meta field names drift between domains (§4.4). Default page size across list views is now **50** (raised from 10/30) via `RowsPerPageSelect`.
- Mutations: create `POST` (FormData when files involved), update `PUT /{id}`, delete `DELETE /{id}`, state toggles `PATCH /{id}/toggle` or `/{id}/status`, bulk `POST /…/bulk`.

**Common value mappings:**

| Concern | API form | UI form |
|---|---|---|
| Days of week | `MONDAY`…`SUNDAY` | `Monday`… (DAY_MAP / REVERSE_DAY_MAP) |
| Times | `"HH:mm"` 24h | `"h:mm AM/PM"` |
| Durations | integer minutes | `"N Min"` |
| Wellness statuses | `CONFIRMED/PENDING/COMPLETED/CANCELLED` | lowercase |
| Referral statuses (new) | `Incomplete\|Pending Signature\|Ready to send\|Sent` | same (title-case, shown as-is) |
| Permission names | free-cased strings | normalized lowercase-alphanumeric on compare |

**Key Resident shape (types.ts, abridged, updated fields marked):**

```typescript
Resident {
  _id, name, firstName?, lastName?, unitNo?,
  phone?, countryCode?, emergencyContact?, emergencyContactCountryCode?, email?,
  careType: 'assisted_living'|'memory_care'|'independent_living'|'skilled_nursing',
  gender?, status: 'Active'|'Inactive'|'Away' /* enum also has Discharged */,
  admissionDate,                                // NEW — required on create, drives "Length of Stay"
  admissionType?, payerSource?,                  // NEW — payerSource read but not rendered/filtered (§2.1)
  isResponsibleParty?, hasLoginAccount?, hasConsentForm?, hasUnviewedSignedDocument?,  // NEW
  assignedStaff: string[],                       // unified care-team array (any designation)
  insuranceName?, birthDate?,
  profilePicture?, profileFetchAt?,              // null until first login → "Send Credentials" shown
  familyMembers: [{ name, email, phone, type: 'Emergency'|'Family',
                    relation: 'Spouse'|'Son'|'Daughter'|'Brother'|'Sister',
                    hasPortalAccess, isAuthorizedAppAccess,  // NEW — distinct family-portal-access toggle
                    profileFetchAt? }],
  // PCC variant rows: pcc_patientId/facId/orgUuid, roomDesc+bedDesc
}
```

**Shared UI components reused across modules:** `ImageSelectionModal` (gallery contract), `DeleteConfirmationModal`, `RowsPerPageSelect`, `StaffSelectDropdown`, `StaffMultiSelect` (new — chip/search/hover multi-select for `assignedStaff[]` and Care Conference team selection), `AccessDenied`, `CalendarPdfPreview`/`pdfTemplate.ts` (browser-print "PDF", theme-aware accent colour), `FontSizeSync`, `SessionTimeoutModal`, `FreshWorksWidget`, `DatePickerInput` (new — `react-datepicker` wrapper replacing native date inputs in Transport/Care Conference forms), `SignatureCanvas` (new — dependency-free canvas signature pad used by Consent Forms and referral physician certification).

**Environment variables consumed (config surface):** `VITE_PROD_URL` / **`VITE_PREPROD_URL`** (main API — this branch reads the latter, §4.5), `VITE_PROD_AUTH_URL` (authApi), `VITE_SOCKET_PROD_URL` / **`VITE_SOCKET_PREPROD_URL`** (sockets), `VITE_COGNITO_CLIENT_ID` / `VITE_COGNITO_CLIENT_SECRET` / `VITE_USER_POOL_ID` / `VITE_REGION` (Cognito), `VITE_GOOGLE_MAPS_API_KEY` (transport), `VITE_FRESH_WORK_WIDGET_ID` / `VITE_FRESH_WORK_WIDGET_URL` (support widget), `VITE_VAPID_PUBLIC_KEY` (new — browser web push for the chat PWA).

## Appendix B — Per-module permission keys (as passed to `usePageAccess`)

| Module | Key |
|---|---|
| Residents | `"Residents"` |
| Staff | `"Staff"` |
| Access Management | `"Access Management"` |
| Activities schedule | `"Activities"` |
| All Day Menu / Specials / Family Meals / Diet | `"all-day-menu"` / `"specials"` / `"family-meal-requests"` / `"diet-management"` |
| Transport rules / requests | `"complimentary"` / `"transport-requests"` |
| Salon settings | `"salon-settings"` |
| Housekeeping pages | `"extra-room-cleaning"`, `"extra-laundry"`, `"miscellaneous-service"`, `"maintenance-requests"` |
| Rehab service / appointments / message / team / availability | `"rehab-list"` / `"rehab-appointments"` / `"rehab-message"` / `"rehab-team"` / `"Rehab"` |
| Announcements | `"Announcements"` |
| **KPI Dashboard (new)** | **`"KPI Dashboard"`** — gates only the Settings-tab entry point; the header quick-access button is a separate `isAdmin` check, not this permission key (§1.3/§2.18) |
| IDT Report, Care Conferences (both views), Referrals, Messages | **no frontend gate** (nav filtering + backend only) |

Keys survive their casing inconsistencies only because both grant storage and checks normalize to lowercase-alphanumeric (§1.3, §4.4).
