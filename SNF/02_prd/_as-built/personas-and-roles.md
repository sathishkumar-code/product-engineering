# Personas & Roles — Senior Living / Skilled Nursing Platform

> Shared reference for both product PRDs ([Senior Living](./prd-senior-living.md), [Skilled Nursing](./prd-skilled-nursing.md)).
> Reverse-engineered from the codebase (Cognito groups, designation constants, role checks). Code is the source of truth; detailed evidence lives in [`modules/platform-foundation.md`](./modules/platform-foundation.md).
> **Refreshed 2026-06-21** against `staging` (backend `62de4747`, admin `d2b8d05`, staff app `d4f8169`, SN resident `026ea88`, SL resident `97f75c4`) — the staff-app three-way app-flow model, the care-team `assignedStaff[]` consolidation, and the new staff-app Skilled Nursing clinical experience are reflected below.

---

## 1. Persona taxonomy

The platform recognizes five human persona types (Cognito groups) plus one machine identity:

| Persona | Identity model | Primary surfaces | Summary |
|---|---|---|---|
| **Super Admin** | Cognito group `SUPER_ADMIN` | Admin web | Platform-level administrator. Operationally similar to Admin in the current UI; distinction is enforced at the role level rather than through a distinct surface. |
| **Facility Admin** | Cognito group `ADMIN` | Admin web | Runs the facility: residents, staff, services, dining, transport, housekeeping, activities, clinical/coordination modules (SNF). Always passes the staff read/write permission filter (no per-page granularity for ADMIN); still subject to the facility's page-visibility config. |
| **Staff** | Cognito group `STAFF` + free-form `designation` | Staff mobile app, admin web (permission-scoped) | A facility employee. Capabilities are driven by **designation** (job role) and an explicit **per-page read/write permission tree**. Some designations get dedicated mobile experiences (driver, stylist, therapist, trainer, housekeeping/maintenance); clinical designations work mainly in the admin web today. |
| **Resident** | Cognito group `RESIDENT`, keyed by `cName` (phone-first username) | Resident mobile app (per-product binary), TV | The senior-living resident or skilled-nursing patient. Carries a `careType` (`assisted_living`, `memory_care`, `independent_living`, `skilled_nursing`) and a facility/room binding. Status lifecycle: Active → Away → Discharged. |
| **Family Member** | Cognito identity linked to a resident (max 4 per resident) | Resident mobile app (acting as resident), tap-to-call contacts | Authenticates with their own credentials, but the backend **normalizes family auth to the linked resident's identity** — a family member acts *as* the resident in bookings, meal requests, etc. Per-module family participation is a facility booking-policy opt-in. `hasPortalAccess` flags portal-enabled members. |
| **TV Device** (machine) | Custom TV JWT (30-min access / 30-day refresh), `istv` header | Android TV app | Device-scoped identity bound to facility + unit. Browsing is device-scoped; transactions require a resident to authorize via QR pairing (unit-number match enforced), after which the device temporarily acts for that resident. |

---

## 2. Staff designations

Designations are free-form strings configured per facility, with hardcoded groupings in the backend that drive behavior:

### 2.1 Designation groups (behavior-bearing)

| Group | Designations | Behavior driven |
|---|---|---|
| **Care team** | Nurse, Case Manager, Social Worker, Doctor, Dietitian (canonical strings in `src/constants/designations.ts`) | Chat initiation gating (residents may only chat with their assigned care team), SNF resident care-team assignment, IDT report section attribution. **Data-model note (staging):** the resident's five legacy per-role fields (`nurse`/`caseManager`/`socialWorker`/`doctor`/`dietitian`) were consolidated into a single flat, indexed `assignedStaff: string[]` (a `cName` array) plus an `assignedStaffDocs` populate virtual. Care-team membership is now a set of assigned staff rather than one-per-role; a one-time `migrateAssignedStaff.ts` script backfills the new shape (see candidate gaps). |
| **Rehab** | Director of Rehab, Rehab Therapists / Rehabilitation Specialist | Rehab team management, rehab appointment staffing, weekly availability self-service, rehab message handling. |
| **Supervision** | 13 SNF leadership roles (Executive Director … Front Desk) | Supervisory visibility groupings. |
| **Directory-hidden** | Housekeeping Staff, Maintenance Staff, Transport Driver, Maintenance, Salon Stylist | Excluded from resident-facing staff directories; receive operational request fan-out instead. |

### 2.2 Staff-app app-flow model & dedicated experiences (as-built, staging)

The staff app no longer routes every staff member into one shared designation
queue. After authentication, `appRoute.helper.ts` resolves the staff member into
one of **three app flows** (`src/utils/featureAccess.ts`), driven by the backend's
per-facility `staffDirectoryRoles` map (`Config.staffDirectoryRoles`, seeded from
`src/constants/staffDirectoryRoles.ts`) rather than a hardcoded designation list.
The resolved flow is persisted (`APP_FLOW`); a designation that resolves to no role
group is denied access (forced logout → `AccessDeniedScreen`).

| App flow | Who | Experience |
|---|---|---|
| **MIGRATED** (Skilled Nursing) | Clinical/leadership groups mapped to it — today **Case Manager, Doctor, Director of Nursing, Social Worker** | Skilled Nursing module, bottom-tab bar **grown to up to 5 tabs as of 2026-08-21** (was documented as 4; corrected this pass): Home, Messages, a new **Transport** tab (real-time, role-gated to Case Manager/Social Worker/Director of Nursing), a new **Documents/Scan** tab (same role gate), and either **My Schedule** or **Pending Sign** (doctor-group e-signature queue) in the fifth slot. **Reports has never been reachable** — registered in code but commented out in every pass reviewed to date, including this one; the "Reports & MySchedule hidden for Doctor" framing in prior versions of this doc was stale/inaccurate and is corrected here. Plus ~45 screens: care conferences (schedule / detail / history / reports + audio recording), interdisciplinary & rehab reports, medication lists, test results, advanced care directives, resident directory/details, family party, the full service-request set, upcoming appointments, Secure Call review/approval, and real-time chat. |
| **LEGACY** | Operational designations — **Salon Stylist, Transport Driver, Housekeeping Staff, Activities Director, Maintenance Staff** (plus any unmapped designation, which defaults to LEGACY) | The prior designation task-queue views (below), now hosted in a 2-tab Home+Messages navigator. |
| **CHAT_HOME** | The `Message` group (e.g. **Dietitian, Receptionist**) | Chat-only home — Residents + Messages, no task queue. |

The `FacilityDesignation` enum in the staff app now carries **20 values** (was 17;
added Case Manager, Doctor, Dietitian). The mapping of a designation to an app flow
is configuration (`staffDirectoryRoles`), so the same designation string can route
differently per facility.

**LEGACY designation task views (as-built):**

| Designation | Staff-app capability |
|---|---|
| Transport Driver | Ride queue; Accept → Start → End lifecycle |
| Salon Stylist | Today's confirmed appointments + waitlist promotion (move-to-slot) |
| Massage Therapist | Today's sessions (read-only) |
| Private Trainer | Today's sessions (read-only) |
| Housekeeping Staff | Request queue: PENDING → IN_PROGRESS → COMPLETED |
| Maintenance Staff | Same queue view with maintenance flavor |
| Activities Director | Designation view now wired (`ActivitiesDirectorView`) |

**Cross-cutting staff-app capabilities (all flows):** real-time chat on the
Socket.io `/chat` namespace (a process-wide `chatSocketService` singleton driving an
`unread` store slice and FCM/Notifee chat notifications with deep-link tap routing) —
as of 2026-08-21 extended with message forward, PHI-aware Keychain drafts, conversation
pin/mark-read/leave-group, and a Message Info sheet (messaging-chat.md MSG-FR-40/41/42;
offline sync was built but never merged — chat is still online-only), Twilio Programmable
Voice outbound resident calling (`CallScreen`, CallerID = facility concierge number, now
feeding a Secure Call summary/approval workflow when recorded), a 24-hour inactivity
logout (`SessionGuard` + `useInactivityTimeout`) now layered **underneath** a new
**device-level biometric App Lock** (Face ID/Touch ID/passcode, facility-configurable,
gates every PHI screen before it mounts — platform-foundation.md PLAT-FR-75), sign-in
extended to phone-or-email with a Cognito user-pool-migrator fallback and 3-channel MFA
(PLAT-FR-72b), a force/optional app-update gate (now also OTA-updatable via Expo/EAS —
not a product requirement, see the repo's CLAUDE.md), and a resident acknowledgment gate.
Chat screens are physically shared between the LEGACY and Skilled Nursing stacks via
route-name aliasing.

> **Persona-gap update.** The prior "no clinical-role mobile experience" gap is
> now **partially closed**: Case Manager, Doctor, Director of Nursing,
> and Social Worker have a first-class Skilled Nursing staff-app experience, and
> chat reaches every staff member. **2026-08-21 update:** two of those four
> designations (Case Manager, Social Worker — plus Director of Nursing) also gained
> dedicated **Transport** and **Documents/Scan** tabs (transportation.md TRN-FR-25);
> the relationship between the new Transport tab and the pre-existing case-manager
> Transport screens is an open product question (transportation.md §9 O17), not yet
> resolved. Remaining gaps: a `DietitianView` exists but is
> unreachable (not wired into the LEGACY designation switch); Nurse, the rehab
> therapist roles, dining, and receptionist still have no task experience; and the
> flow gating is client-side off a hardcoded `ROLE_GROUP_FLOW_MAP`. (See module
> docs §9 and the staff-app analysis.)

### 2.3 Permission model (applies to staff in admin web)

Two stacked filters determine what a staff member sees and can do in the admin web:

1. **Facility page visibility** (`Config.accessPages`) — which modules exist for this facility at all. Applies to admins too.
2. **Per-staff read/write grants** (`accessPermissions` tree, with designation-level templates as defaults) — read unlocks the page; write unlocks mutations. ADMIN bypasses this filter only.

Designations act as **permission templates**: assigning a designation seeds the staff member's permission tree, which can then be customized per person.

### 2.3.1 SNF admin-web access matrix (desired-state, pilot configuration)

The matrix below is the **desired-state** page-access configuration for the
Skilled Nursing pilot — the designation→page permission template the platform is
expected to enforce in the admin web. It is the access contract the SN test
catalog asserts the code against (see
[`requirements/skilled-nursing-test-catalog/explore-notes.md`](../requirements/skilled-nursing-test-catalog/explore-notes.md) §4); any
divergence in the running code surfaces as an Axis-A allow/deny test failure, not
a silent edit here.

> ✓ = role may access the page. Empty = no access (page hidden / request refused).
> The first column **Manager** ≈ the facility manager/administrator role (full
> suite). Role-name reconciliation with §2.1: **CM** = Case Manager, **SW** =
> Social Worker, **Dietitian** = Dietician, **Driver** = Transport Driver,
> **Maint** = Maintenance Staff, **Msg-only** = chat-only staff (Residents +
> Messages only), **Rehab** = Rehab group, **Act. Dir** = Activity Director.

| Page \ Role | Manager | CM | SW | Nurse | Doctor | Dietitian | Driver | Maint | Msg-only | Rehab | Act. Dir |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Residents | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Messages | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Transport | ✓ | ✓ | ✓ | ✓ | | | | | | | |
| Conference | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | ✓ | |
| Activities | ✓ | ✓ | ✓ | | | | | | | | ✓ |
| Dining | ✓ | | | | | ✓ | | | | | |
| Rehab | ✓ | | | | | | | | | ✓ | |
| Maintenance | ✓ | | | | | | | ✓ | | | |
| IDT Report | ✓ | ✓ | ✓ | ✓ | | | | | | ✓ | ✓ |
| Staff | ✓ | | ✓ | | | | | | | | |
| Access | ✓ | | | | | | | | | | |
| Announcements | ✓ | ✓ | | | | | | | | | ✓ |
| Facility Settings | ✓ | | | | | | | | | | |

Notes:
- **Residents** and **Messages** are universal — every staff role reaches them
  (Messages access is the page filter; *who* a resident may chat with is gated
  separately by care-team assignment, see §2.1).
- **Driver, Maintenance, Msg-only** are effectively staff-app/operational roles in
  the admin web: they see only Residents + Messages (Maintenance additionally sees
  the Maintenance page).
- **Access** (access-management) and **Facility Settings** are Manager-only.
- This matrix omits salon/housekeeping admin pages, consistent with the pilot
  exclusions captured in the SN test catalog.

---

## 3. Persona × product matrix

| Persona | Senior Living | Skilled Nursing |
|---|---|---|
| Facility Admin | Residents (careType AL/IL/MC), services, dining, transport, housekeeping, activities, announcements, AL therapy scheduling | Same, plus: SNF resident roster (case manager/social worker/doctor columns), rehab suite, IDT reports, care conferences, referrals & agencies, messages |
| Staff (operational) | Driver / stylist / therapist / trainer / housekeeping flows (staff-app LEGACY flow) + cross-flow chat & Twilio calling | Identical (shared staff app) |
| Staff (clinical) | — (AL therapy is admin-scheduled) | **Staff-app MIGRATED flow** for Case Manager / Doctor / Director of Nursing / Social Worker (care conferences, IDT/rehab reports, medications, test results, ACDs, resident directory, care-team chat); plus rehab team & availability and referral participation in the admin web. Other clinical roles (Nurse, rehab therapists, Dietitian→chat-only) remain admin-web / chat-only. |
| Resident | `senior_living_reactnative` app + TV — now API-backed self-service (salon/massage/private-training booking, `/care` therapy flows, schedule/activities) with `x-facility-id` scoping and foreground push | `senior_living_skillednursing_resident` app + TV — adds medications, lab results, rehab, care-team chat, advance care directives, care conferences, brain games, TV pairing; HIPAA inactivity logoff, force-update, acknowledgment & discharge gates |
| Family Member | Acting-as-resident in bookings/meals (policy opt-in); contact directory | Same, plus care-conference participation as invitees |
| TV Device | Full TV experience | Full TV experience (pairing flow surfaced in SN resident app) |

---

## 4. Identity notes that shape requirements

- **`cName` (Cognito username) is the universal person key** across all modules — bookings, chat, audit fields, staff assignment all key on it.
- **Phone-first usernames**: logins normalize phone numbers to `+<digits>`. The SL resident app uses email+TOTP while the SN resident app and (now, on staging) the **staff app** use phone+password with an MFA challenge only when the pool issues one (the staff app previously authenticated by email and ran a self-enrolling TOTP setup flow, since removed) — a login inconsistency inherited by any unified-login ambition. Forgot/reset password has moved off Cognito onto a backend custom-OTP flow (`/api/auth/forgot-password`, `/api/auth/reset-password`) in the SN resident and staff apps.
- **Family-as-resident normalization** means per-family-member attribution is lost in most flows (a family booking looks like a resident booking). Flagged as a candidate gap in module docs.
- **Provisioning is admin-driven**: residents, family members, staff, and admins are created from the admin web; "Send credentials" issues Cognito invites (shown until first profile fetch).
