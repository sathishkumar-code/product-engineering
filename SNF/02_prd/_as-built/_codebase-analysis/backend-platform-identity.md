# Senior Living Backend — Platform, Identity & Infrastructure Functional Spec (Reverse-Engineered)

> Source of truth: `senior_living_backend` codebase as of staging `e6469276` (2026-08-27).
> Scope: identity & access, multi-tenancy & product gating, notifications platform, TV platform,
> app-version management, cross-cutting infrastructure, resident lifecycle (platform view).
> Domain modules (wellness/dining/clinical) are covered by sibling analyses.

---

## 1. System overview

- Node/Express 5 (TypeScript), MongoDB/Mongoose, Socket.io, node-cron. Port 7000 (`src/server.ts:63-66`).
- Boot sequence (`src/server.ts`): load `.env` (optional `ENV_FILE` override) → `initConfig()` pulls **all secrets from AWS Secrets Manager** (`AWS_SECRET_NAME`, hardcoded region `us-west-1`, `src/config/index.ts:8`) and copies them into `process.env` → validate required env (**now only `AWS_S3_BUCKET_NAME` + `TV_JWT_SECRET`** — `ALLOWED_ORIGINS` is **no longer boot-required** and `PRE_ALLOWED_ORIGINS` is commented out, `src/server.ts:29-31`) → lazily `require()` app/db/socket/cron modules (so they see the injected env) → connect Mongo → start cron jobs **only if `ENABLE_NOTIFICATION_CRON === 'true'`** (announcement cron has a separate `ENABLE_ANNOUNCEMENT_NOTIFICATION_CRON` gate, §5.3) → listen.
- `src/app.ts` mounts ~60 routers under `/api/*`. CORS origins come from `ALLOWED_ORIGINS`; allowed headers include `x-facility-id`, `facilityid`, `is-tv`, `istv`. JSON body limit 12 MB.
- Layering: `routes/` → `controllers/` → `services/` → `models/`, with Zod validation middleware (`validate`, `validateQuery`, `validateParams` in `src/middleware/middleware.ts`) writing parsed payloads to `res.locals.body` / `res.locals.query`.

### Environment model
- `NODE_ENV` drives an env-var prefix: `DEVELOPMENT`/`DEV`/`LOCAL` → `STG_`, `PRE-PRODUCTION` → `PRE_`, else no prefix (`src/lib/common.ts:19-39`). e.g. `getUserPoolId()` reads `STG_USER_POOL_ID` in staging. So one secrets bundle can hold per-env values.
- `isDevelopmentEnv()` intentionally accepts the misspelling `developement` and — notably — also treats `pre-production` as development (`src/lib/common.ts:9-13`), which makes pre-production use the fixed temp password (see §3.2). On staging this is now the *only* trigger for the fixed temp password (the `Config.sendDefaultPassword` flag was removed).

---

## 2. Multi-tenancy and the facility model — KEY for product split

### 2.1 Tenant key: `x-facility-id` header
- Every `/api/*` route (mounted after `/api/config`, `/api/app-version`, `/api/auth` reset routes, and `/api/fax`) passes through `facilityMiddleware` (`src/app.ts:104`), which reads `x-facility-id` or `facilityid` headers and stores `req.facilityId` (`src/middleware/facilityMiddleware.ts`). Note `/api/fax` mounts at `app.ts:103`, **before** this global gate, with its own bypassable guards (§3 / §8.5).
- **Bug**: the missing-header check is `if (facilityId === null || facilityId === '')` — after the loop `facilityId` is `undefined` when absent, so the 400 guard never fires (`facilityMiddleware.ts:41`). Effectively the header is *optional* in practice; queries then run with an empty facility filter (`getFacilityFilter` returns `{}` — `src/lib/facility.ts:51-54`), i.e. **cross-tenant reads are possible when the header is omitted**.
- Helper `getFacilityId(req)` (`src/lib/facility.ts`) reads `facilityid` first, then `x-facility-id`. Almost every controller scopes Mongo queries via `{ ...getFacilityFilter(req), ... }`.
- Socket connections carry the facility in handshake headers or `auth.facilityId` (`src/socket/socketAuth.ts:47-59`).

### 2.2 Facility = one `Config` document (`src/models/config.model.ts`)
There is **no Facility collection**; the `Config` document keyed by `facilityId` *is* the facility record. It bundles:

| Area | Fields | Notes |
|---|---|---|
| Identity/branding | `facilityId`, `conciergeNo`, `lat`, `lng`, `timeZone` (default `America/Los_Angeles`), `theme.{primary,buttonColor}`, `logo`, `acknowledgementPdf` | `facilityName`, `tvSetupLocations`, `familyRelations` are **selected by controllers (`config.controller.ts:285,298,340`) but absent from the schema** — they exist only as loose Mongo fields |
| Feature/page visibility | `accessPages[]` (name, isHidden, rank, children[]) | The per-facility navigation/feature switchboard; defaults in `src/constants/accessDefaults.ts` |
| Staff role catalogue | `designations: string[]` | Declared in the TS interface (line 179) but **not registered in the schema** — mutated via `Config.updateOne(..., { strict:false })` (`config.controller.ts:674-725`) |
| Permission templates | `defaultPermissions.{global, designationGroup, designations}` | 3-layer template applied at staff creation (§3.4) |
| Booking policy | `bookingPermission` per module key (`SalonAppointment`, `MassageAppointment`, `PrivateTrainingAppointment`, `TransportationRequest`, `FamilyMealRequest`, `Care`, `RehabAppointment`, `Housekeeping`) | Gates on-behalf bookings by staff designation / family member (§3.5) |
| Dining | `meals.{breakfast,lunch,dinner}` time ranges, `mealRates`, `maxGuest`, `blackoutDates` | |
| Transport | `transportation.{pricePerMiles, pickupBufferMin/Max/Multiplier, appointmentDurationOptions, MaxMilesForTransport}` | |
| Rehab (AL flavor) | `rehab.{physicalThearapy, cognitiveEvaluation, rehabAvaulation, outsideAgency}` (sic — misspellings preserved in DB) | per-type duration/location/image |
| Chat | `chat.*` limits + access policy (`isResidentAllowed`, `staffDesignationAllowed[]` with `[]`/`["*"]` = all, `isAdminAllowed`) + attachment caps | |
| PMS | `pms[]` ({provider, baseUrl, isActive, config}) with providers `OPERA | POINTCLICKCARE | YARDI | TELS | CUSTOM`; `integratedModules` Map of module→provider (`SALON, MASSAGE, PT, CARE, TRANSPORTATION, ACTIVITY, DINING, HOUSEKEEPING, MAINTENANCE`) | Validator ensures `integratedModules` only references configured providers (`config.model.ts:731-737`) |
| Security/UX | `inactivityTimeout.{web,mobile}` minutes, `maxFutureBookingDays`, `maxFamilyMembersCount` | The per-facility `sendDefaultPassword` flag (which previously forced the fixed temp password `TempP@ssword123`) has been **removed** on staging — temp-password behaviour is now decided solely by `isDevelopmentEnv()` (see §3.2). |

A second tenant-scoped integration registry exists: **`IntegrationAvailable`** (`src/models/IntegrationAvailable.model.ts`) — `{facilityId, externalId (PCC facId), name: "pcc", type: "pms", baseUrl, orgUuid, clientId, clientSecret, token, isActive, timzone (sic)}`. PCC webhook handlers map inbound PCC `facId`/`orgUuid` back to a facility via this collection. Client secrets are stored **in plaintext in Mongo**.

### 2.3 How a facility's product/feature set is determined ("Senior Living" vs "Skilled Nursing")
There is **no single facility-level `productType` flag**. The split is assembled from four independent mechanisms:

1. **Resident-level `careType`** — enum `assisted_living | memory_care | independent_living | skilled_nursing` (`src/contants/index.ts:1-6`, `src/models/resident.model.ts:9,113-117`). Required on every resident. List endpoints filter by `?careType=` (`resident.controller.ts:701`, `pagination.schema.ts:21`). A facility "is" SNF only insofar as its residents are.
2. **Rehab therapy-type fork** — the clearest in-code product fork (`src/constants/rehab.ts:38-101`):
   - Assisted Living uses fixed therapy types `PHYSICAL_THERAPY | COGNITIVE_EVALUATION | REHAB_EVALUATION | OUTSIDE_AGENCY`, with durations/locations from `Config.rehab.*`.
   - Skilled Nursing uses `therapyType: 'OTHER'` + a dynamic `RehabTherapy` catalogue row referenced by `therapyId` (`isRehabTherapyIdRequired()`); comments in `RehabAppointment.model.ts:23,62` codify "OTHER ⇒ SKILLED_NURSING".
3. **`Config.accessPages` visibility** — modules are hidden per facility by setting `isHidden: true` (e.g. an SNF facility can hide `Salon`/`Private Training`; an AL facility can hide `Rehab` children like `therapy-evaluations`). Hidden pages are stripped for everyone, then staff see only pages matching their read permissions (`config.controller.ts:405-466`). Defaults list 15 top-level pages (`accessDefaults.ts`).
4. **Integration wiring** — presence of a PCC row in `IntegrationAvailable` + `Config.pms[]`/`integratedModules` turns on PCC patient/medication sync (SNF-typical); TELS for maintenance work orders; Lemedix (CrelioHealth) for lab reports.

> PRD recommendation: the product split is currently *emergent* (careType + accessPages + therapy fork + integrations); a PRD should specify an explicit facility-level product flag if the two SKUs diverge further.

### 2.4 Facility discovery endpoints
- `GET /api/config/getAllfacility` (optional auth): STAFF callers get only **their own** facility config (looked up via their Staff record); everyone else (including unauthenticated) gets **all facilities** (id, conciergeNo, lat/lng, designations, maxFutureBookingDays, facilityName, tvSetupLocations, familyRelations) (`config.controller.ts:275-317`). Used by app login screens to pick a facility.
- `GET /api/config/get-facility-data` — same projection for one facility; also upserts the caller's FCM token from `x-fcm-token` header.
- `GET /api/config/` — returns the **entire raw config** for the facility, **no auth** (`config.routes.ts:43`).

---

## 3. Identity & access

### 3.1 Personas and auth methods

| Persona | Cognito group | Mongo model | Login identity | Notes |
|---|---|---|---|---|
| Super admin | `SUPER_ADMIN` | (none — treated as ADMIN) | — | Only referenced in role checks |
| Facility admin | `ADMIN` | `Admin` (`Admin.model.ts`) | phone (E.164 = Cognito username) | Per-facility; global unique phone |
| Staff | `STAFF` | `Staff` (`Staff.model.ts`) | phone | Carries `designation` + `accessPermissions` |
| Resident | `RESIDENT` | `Resident` (`resident.model.ts`) | phone | `cName` = Cognito username |
| Family member | `FAMILY_MEMBER` | `FamilyMember` (`familyMember.model.ts`) | phone | Linked to exactly one resident |
| TV device | (groups copied from authorizing user) | `TvDevice` + `TvAuthToken` | custom HS256 JWT | §6 |

Roles come from the `cognito:groups` claim of an AWS **Cognito access token** (`USER_PASSWORD_AUTH` on the client side per repo docs). Cognito User Pool ID is resolved per environment via `getUserPoolId()` → `USER_POOL_ID`/`STG_USER_POOL_ID`/`PRE_USER_POOL_ID`.

### 3.2 Token verification (`src/middleware/authMiddleware.ts`)
1. If header `istv`/`is-tv` is truthy → **TV path** (§6.4); else expect `Authorization: Bearer <Cognito access token>`.
2. JWT verified via JWKS (`src/lib/jwksClient.ts`, issuer = `COGNITO_ISSUER`); must be `token_use === 'access'` with `sub` and `username`.
3. `req.user = { sub, username, groups, scope, clientId }`.
4. **Family-member normalization**: if groups include `FAMILY_MEMBER`, look up `FamilyMember` by `cName = payload.username`, then the linked `Resident`; rewrite `req.user.username` to the **resident's** cName and set `isFamilyMember`, `residentId`, `familyMemberId`, `familyMemberCName` (`authMiddleware.ts:99-145`). Downstream code can therefore always treat `req.user.username` as "the resident in scope". The same normalization happens for socket auth in chat (`chat.handler.ts:124-153`).
5. Helpers `isStaffOnlyRequest`, `isResidentOnlyRequest`, `isFamilyMemberOnlyRequest` implement exclusive-role checks.
6. `optionalAuthMiddleware` — same logic but always continues on failure.

Role gates: `requireAnyRole`, `requireAllRoles`, `requireRoleWithAdmin` (`src/middleware/roleMiddleware.ts`) against `COGNITO_USER_ROLES` (`src/contants/cognnito.types.ts`).

**Temporary password policy** (`src/lib/common.ts:43-63`): random 8-char Cognito-compliant password, except when `isDevelopmentEnv()` returns true, in which case it is the constant `TempP@ssword123`. **Staging change:** the `Config.sendDefaultPassword` branch has been removed — production no longer returns the literal at all. However `isDevelopmentEnv()` still treats `pre-production` as dev-like (`common.ts:9-13`), so **pre-prod continues to issue the static `TempP@ssword123`** (Design-gap G-11, reclassified High → Medium, partially mitigated; T-15 notes this widens dev-only branches into pre-prod). SMS delivery is via Cognito (`DesiredDeliveryMediums: ['SMS']`).

**MFA**: `forceCognitoSmsMfaPreferred` enables SMS MFA as preferred for every provisioned user unless env `MFA_ENFORCE=false` (`src/lib/cognitoUser.ts:176-201`). TOTP (software token) is explicitly disabled. MFA reset endpoint: `POST /api/residents/resetMFA` clears both MFA settings and best-effort `AdminUserGlobalSignOut` (`resident.controller.ts:2382-2455`); `GET /api/residents/mfa/configuration` exposes pool MFA mode.

### 3.3 Provisioning flows (all phone-first; Cognito username = E.164 phone)

**Admin** (`POST /api/admin/`, auth required; `admin.controller.ts:191-287`):
create-only Cognito user → verify phone attr → force SMS MFA → add to `ADMIN` group → insert `Admin` doc (`active: true`). 409 on duplicate phone. A bootstrap variant `POST /api/admin/bootstrap-admin` (no auth) exists only when `ALLOW_ADMIN_BOOTSTRAP=true`.

**Staff** (`POST /api/staff/`, auth required; `staff.controller.ts:324-503`):
1. Duplicate check against active staff by normalized email and `(countryCode, phone)` (409s).
2. Optional profile picture → S3 (`staff/` folder).
3. Load facility `Config` (needed for password policy + permission templates; 500 if missing).
4. Cognito: create-only by phone → verify phone → force SMS MFA → add to `STAFF` group.
5. `accessPermissions` computed by **3-layer template merge** `mergePermissionsByDesignation(config, designation)` (§3.4).
6. Insert `Staff` (`cName = cognitoUsername`, `active: true`, optional `speciality` → `RehabTherapy` ref).
7. Optional weekly availability is applied post-create; invalid availability returns 400 **but the staff remains created**.

**Resident (+family) — admission** (`POST /api/residents/`; **no authMiddleware on this route** — see §10; `resident.controller.ts:391-665`):
1. Mongo transaction started; Cognito usernames tracked for rollback.
2. Resident Cognito user (create-only, phone-first, SMS MFA, `RESIDENT` group).
3. `Resident` doc created: profile (name split, birthDate/dob fallback, gender), `unitNo` (room), `careType` (required), `status` default `Active`, emergency contact normalized to E.164, optional PCC linkage fields (`pcc_patientId`, `pcc_facId`, `pcc_orgUuid`, `pcc_patient_details[]`).
4. Each family member: `createOrResolveCognitoUserByPhone` (idempotent — reuses existing Cognito user), verify phone + force MFA, add to `FAMILY_MEMBER` group, then `FamilyMember.insertMany` linked by `residentId`.
5. Commit; **on any error** abort transaction and best-effort delete all newly created Cognito users.
6. Fire-and-forget: if PCC fields present, `syncPccMedications` pulls medications, **envelope-encrypts every field with a KMS data key** (`encryptWithDataKey`, `resolveResidentMedicationKey`) and bulk-upserts `Medication` docs (`resident.controller.ts:564-616`).

**Family member (post-admission)** (`POST /api/residents/add-family-member`, auth): enforces resident exists & not deleted, optional `Config.maxFamilyMembersCount`, creates/resolves Cognito user, **rejects a phone already linked to a family member of a different resident** (with Cognito rollback if it was newly created) (`resident.controller.ts:247-322`). `PUT /edit-family-member`, `POST /remove-family-member` (removal does Cognito global sign-out → disable → delete, `cleanupFamilyMemberCognitoAccess`, lines 331-377).

**Resend credentials** (`POST /api/residents/send-credentials`, auth): for `type: resident|family|staff`, re-verifies phone, re-forces MFA, and replays `AdminCreateUser` with `MessageAction: "RESEND"` to re-send the temporary-password SMS (`resident.controller.ts:1647-1746`, `cognitoUser.ts:203-220`).

**Deletion semantics**:
- Resident: soft delete (`deletedAt`) + best-effort Cognito delete (`deleteResident`, `deleteCurrentResident`). Family member Cognito users are *not* cascaded on resident delete.
- Staff: soft delete (`active: false`) in a transaction + best-effort Cognito delete (`deleteStaff`, `deleteCurrentStaff` — the latter also sets `deletedAt`, a field not in the Staff schema).
- Admin: **hard delete** of the Mongo doc + Cognito delete.
- Self-service deletion endpoints exist for resident (`DELETE /api/residents/self-delete`) and staff (`DELETE /api/staff/self-delete`) — app-store account-deletion compliance.

**Logout** = clearing the stored FCM `pushToken` (`/api/residents/logout`, `/api/staff/logout`). Push tokens are upserted opportunistically from the `x-fcm-token` header on profile/config fetches (`src/lib/pushToken.ts`).

**Cognito export**: `GET /api/cognito/export` (auth, any role) streams a CSV of **all user-pool users** and also writes it to `uploads/` on the server (`cognito.controller.ts`) — admin/migration tooling.

**PCC bulk Cognito backfill — welcome message (staging, 2026-08-27, additive to an existing gap):** `POST /pcc-sync/syncResidentCognitoBulk` (`pccSync.controller.ts:bulkResidentCognitoSync`) is an out-of-band, idempotent Cognito-account backfill — mounted at `/pcc-sync`, **outside** `/api/*` and therefore outside both `facilityMiddleware` and `authMiddleware`; its only gate is a hardcoded, non-env-sourced shared-secret literal checked against `x-sync-secret`/`secretKey` (the same literal reused across three `/pcc-sync/*` bulk endpoints — see `architecture-senior_living_backend.md` Design Gap **G-30** / `docs/prd/modules/clinical-records.md` **CLIN-GAP-22**; not re-derived here). It processes every Resident in a facility one at a time (phone preferred, email fallback; existing accounts never modified) and, as of this window, its `finalizeNewAccount` helper (`pccSync.controller.ts`, inside `bulkResidentCognitoSync`) now fires a fire-and-forget welcome message after successfully creating a new Cognito account: `sendPasswordlessWelcome({ facilityId, phoneE164 | email, identifier, firstName, logLabel })` (`src/services/passwordlessLogin.service.ts:112-152`). This is **not new code** — `sendPasswordlessWelcome` is the same pre-existing service already used by the PCC `patient.admit`/readmit onboarding path (`src/integrations/pms/pcc/webhooks/shared/residentOnboarding.ts`) and by `resident.controller.ts`, `authReset.controller.ts`, and `consentForm.controller.ts` — this is only a new call site. It resolves `facilityName` from `Config` when not supplied, builds subject/SMS/text/HTML via `buildResidentPasswordlessWelcome`, sends over `sendCredentialsSms`/`sendEmail`, and logs (never throws) per-channel failures independently — same never-throws/logged-and-swallowed contract as every other caller.
**No new endpoint, no new auth check, no schema change** — the call sits entirely inside the existing hardcoded-secret-gated function and does not touch the secret check itself. It does, however, widen **G-30/CLIN-GAP-22**'s practical blast radius: anyone holding the compromised secret can now also trigger unsolicited outbound SMS/email contact to a resident's phone or email, in addition to the pre-existing DB/Cognito read-write access — worth folding into whatever remediation eventually closes that gap rather than tracked as a separate finding.

### 3.4 Staff permission model
- Shape: `accessPermissions: [{name, allowed, isRead, isWrite, children: [{name, allowed, isRead, isWrite}]}]` mirroring `Config.accessPages` (`Staff.model.ts:40-53`).
- Semantics (`src/utils/staffPermissions.ts`): readable = `isRead || isWrite` with **upward propagation** (a readable child makes the parent readable); writable similarly. `syncPermissions` rebuilds a staff's grants to match the current accessPages structure (drops unknown names, preserves matching grants, applies updates).
- Defaults at creation: 3 layers, ascending priority — `defaultPermissions.global` → `defaultPermissions.designationGroup[<group of designation>]` → `defaultPermissions.designations[<exact designation>]` (`mergePermissionsByDesignation`, lines 151-184). Groups defined in `src/constants/designationGroup.ts`: `rehab` = [Director of Rehab, Rehab Therapists]; `supervision` = the 13 SNF leadership roles (§3.6).
- Enforcement middleware (`src/middleware/accessPermissionsMiddlewares.ts`): `requireAll/AnyStaffPermission[s]` (read) and `requireAll/AnyStaffWritePermission[s]` — **apply only when caller is STAFF**; ADMIN/RESIDENT pass through (role middleware is expected to gate them separately).
- Canonical permission names (`ACCESS_PERMISSIONS`, `src/contants/cognnito.types.ts:9-22`): Dashboard, Salon, Housekeeping, Dining, Access Management, Residents, Services, Transport, Maintenance, Massage Therapy, Rehab, Settings.
- Permission editing: `PATCH /api/staff/:id/access-permissions` (note: **no authMiddleware on this route**, `staff.route.ts:133-137`). Designation CRUD + per-designation templates: `/api/config/designations*` (auth).
- Staff profile fetch filters out permissions whose page is hidden in `Config.accessPages` (`staff.controller.ts:788-828`).

### 3.5 On-behalf booking policy (`resolveBookingContext`, `src/middleware/bookingContextMiddleware.ts`)
Per-module policy from `Config.bookingPermission[modelKey]`:
- RESIDENT: always allowed self-service; any supplied `residentCName` ignored (spoof guard).
- FAMILY_MEMBER: allowed iff `isFamilyMemberAllowed`; books for the linked resident only.
- STAFF: must supply a resident identifier (`residentCName` | legacy `cName` | `residentId`); allowed iff their `Staff.designation` is in `staffDesignationAllowed` (plus expansion of `staffDesignationGroupAllowed` groups).
- ADMIN/SUPER_ADMIN: bypass designation check but the module policy must exist (per-facility opt-in).
Result is `req.bookingContext = {createdByType, createdByCName, residentCName}`. Missing policy = deny for everyone except residents.

### 3.6 Staff designation catalogue (as found in code)
Designations are **facility-configurable strings** (`Config.designations`, unregistered field) — code only hardcodes these sets:

- Care-team designations (`src/constants/designations.ts`): **Nurse, Case Manager, Social Worker, Doctor, Dietitian**. (Previously each mapped to a dedicated resident field; staging consolidated assignment into `Resident.assignedStaff[]` — §3.7.)
- Rehab group (`src/constants/rehab.ts:13-16`): **Director of Rehab, Rehab Therapists**.
- Supervision group (SNF leadership, `rehab.ts:21-35`): **Executive Director, Administrator, Admissions Coordinator, Community Liaison, Director of Nursing, Director of Rehabilitation, Activities Director, Director of Social Services, Case Manager, Dietary Services, Business Office, Reception, Front Desk**.
- Hidden from resident/staff directory listings (`staff.controller.ts:59-65`): **Housekeeping Staff, Maintenance Staff, Transport Driver, Maintenance, Salon Stylist** (unless explicitly requested by designation filter).
- `getMyResidents` is restricted to designations **Case Manager, Social Worker, Doctor** (case-insensitive match, `staff.controller.ts:1000-1041`).
- `Staff.speciality` (ObjectId → `RehabTherapy`) supports "Rehabilitation Specialist"-type staff.

### 3.7 Resident ↔ family ↔ care-team linkage
- `FamilyMember.residentId` (1 resident → N family members; per-resident phone uniqueness index `{residentId, countryCode, phone}` unique-sparse). `type: Emergency|Family`, `relation`, `hasPortalAccess` (drives Cognito provisioning on update flows). **New PCC fields (staging):** `phone`, `pcc_contactId` (indexed), `pcc_is_guarantor` — synced by the PCC `patient.updateContactInfo` / `patient.admit` webhooks (`familyMember.model.ts:9,20,22,50,52`).
- **Care-team model refactor (staging):** the resident's five legacy flat fields (`nurse, caseManager, socialWorker, doctor, dietitian`) and their `nurseStaff`-style virtuals, the `RESIDENT_CARE_TEAM_FIELDS` set, and the `DESIGNATION_TO_CARE_TEAM_FIELD` map have all been **removed**. Care team is now a single indexed `assignedStaff: string[]` (array of Staff cNames) on the resident (`resident.model.ts:33,94-97`), with an `assignedStaffDocs` populate virtual (`resident.model.ts:153-156`). Assignment and chat-scoping go through `src/lib/assignedStaff.ts` (`isStaffAssignedToResident`, `getResidentAssignedStaff`) and `src/utils/assignedStaff.ts` (`notifyAssignedStaff`). A manual one-time migration `src/scripts/migrateAssignedStaff.ts` consolidates legacy → `assignedStaff[]` (not wired into CI/boot — T-14).

---

## 4. Resident lifecycle (platform view)

- **Statuses**: schema enum `Active | Away | Discharged | Transferred` (TS type omits `Transferred` — drift, `resident.model.ts:11` vs `:120`). `dischargeDate` field; PCC webhooks (`integrations/pms/pcc/webhooks/handlers/patient/*`) drive discharge/transfer/cancel transitions and resident info updates for PCC-linked facilities.
- **Soft delete** via `deletedAt`; listing queries filter `deletedAt: null`.
- **Room assignment** = `unitNo` (required string). UnitNo is also the join key for TV pairing authorization (§6.3).
- **Admission** = `POST /api/residents` (§3.3). `source` field and `pccPatientId`/`pcc_*` support PCC-originated residents; `getResidents` merges/injects DB residents matched by `pcc_patientId` even when careType/status filters would exclude them (`resident.controller.ts:784-869`).
- **Profile**: `GET /api/residents/profile` (self, updates `profileFetchAt`, upserts FCM token); pictures gallery (`/pictures` upload/list/delete, S3); profile picture upload; favorites (`favoritedBy`) and contact sharing (`shareContact`) for the resident directory (`getContact`).
- **Acknowledgement workflow**: facility uploads `Config.acknowledgementPdf`; residents (`PATCH /api/residents/acknowledge`) and staff (`PATCH /api/staff/acknowledge`) set `acknowledgement: true` + `acknowledgedAt` — a compliance attestation.
- **Terms acceptance (staging)**: `Admin` and `Staff` gained `isTermsAccepted` (default false) + `isTermsAcceptedAt` (`Admin.model.ts:20-22,40-41`, `Staff.model.ts:114-116,217-218`), set via `POST /api/auth/accept-terms` (auth-gated, `authReset.routes.ts:30`) — the first-login T&C gate from the web/staff apps. A standalone `POST /api/auth/verify-otp` (`authReset.routes.ts:28`) is now a distinct step in the custom-OTP forgot-password flow alongside `forgot-password` / `reset-password` (mirrored for staff under `/api/auth/staff/*`). `Staff.email` is now optional/non-unique.
- **Notification preferences**: per-user toggles limited to keys `DINING, SALON, TRANSPORT, HOUSE_KEEPING, REHAB` (`src/constants/notification.types.ts`); missing key = ON. Residents and staff have GET/PUT endpoints; staff additionally have a global `notificationStatus` boolean.
- **Payment history**: `/api/residents/payment-history` (+ per-resident variant) — service in `paymentHistory.service.ts` (domain detail out of scope here).

---

## 5. Notifications platform

### 5.1 Channels
1. **FCM push** via firebase-admin; initialized from `FIREBASE_PROJECT_ID/CLIENT_EMAIL/PRIVATE_KEY` env (Secrets Manager); silently disabled if missing (`src/config/firebase.ts`).
2. **In-app socket** — namespace `/notifications`, Cognito-authenticated, room `user:<cName>`, event `notification:new` (`src/socket/notifications.handler.ts`, `emitInAppNotification` in `config/socket.ts:30-33`).
3. **History** — `NotificationHistory` (recipientType `RESIDENT|STAFF|FAMILY`, recipientCName, scheduleType/scheduleId, title, body, isRead/isDeleted). Read API: `GET /api/notifications` (auth, paginated). Web-panel-only staff (no push token) still get history rows + socket emits (`notification.service.ts:309-347`).

### 5.2 Per-facility notification configuration
`NotificationConfig` (one doc per facility) — modules → events → `{immediate: bool, scheduled: {enabled, offsets[]}}` (`notificationConfig.model.ts`). Lazily seeded from `DEFAULT_NOTIFICATION_MODULES` on first read (`notificationConfig.service.ts:6-13`). Default catalogue (`src/constants/notificationConfig.defaults.ts`), default reminder offsets **20 min / 1 h / 1 day**:

| Module | Events |
|---|---|
| TRANSPORT | RIDE_REQUESTED, RIDE_ASSIGNED, RIDE_REMINDER*, DRIVER_ARRIVED, TRIP_STARTED, TRIP_COMPLETED, RIDE_CANCELLED |
| MAINTENANCE | REQUEST_CREATED, REQUEST_ASSIGNED, WORK_IN_PROGRESS, REQUEST_COMPLETED |
| HOUSEKEEPING | CLEANING_REQUEST_RAISED, REQUEST_ASSIGNED, CLEANING_IN_PROGRESS, COMPLETED |
| ACTIVITIES | ACTIVITY_SCHEDULED, ACTIVITY_REMINDER*, ACTIVITY_CHANGED, ACTIVITY_CANCELLED, ACTIVITY_COMPLETED |
| SALON | APPOINTMENT_BOOKED, APPOINTMENT_CONFIRMED, WAITLIST_TO_CONFIRMED, APPOINTMENT_REMINDER*, APPOINTMENT_COMPLETED, APPOINTMENT_CANCELLED |
| HEALTH_CARE | CREATE_REHAB_APPOINTMENTS, REHAB_APPOINTMENT_REMINDER*, REHAB_APPOINTMENT_COMPLETED/CANCELLED, CARE_CONFERENCE_SCHEDULE/REMINDER*/COMPLETED/CANCELLED, CREATE_IDT_REPORT, IDT_REPORT_REMINDER (1d only), IDT_REPORT_SUBMISSION |
| DINING | MEAL_ORDER_PLACED, MEAL_ORDER_CONFIRMED, MEAL_READY, MEAL_DELIVERED, ORDER_CANCELLED |

(* = scheduled reminders enabled by default.) Admin CRUD via `/api/notification-config`. ~13 dedicated `*.notification.service.ts` files implement the immediate event sends per domain (salon, massage, PT, rehab, transport, care conference, IDT report, referral, service requests, chat, announcements, activity).

### 5.3 Cron jobs (`src/jobs/`, all gated behind `ENABLE_NOTIFICATION_CRON=true` at bootstrap; each also has its own `ENABLE_*` env kill-switch and `*_CRON_SCHEDULE` override; timezone `FACILITY_TIMEZONE` default America/Los_Angeles). **Staging:** 5 job files now produce **6 cron starts** (`server.ts:77-80` + care-conference enable/review).

| Job | Default schedule | Function |
|---|---|---|
| `notification.cron` | `* * * * *` (every minute) | Collects all distinct scheduled offsets across every facility's NotificationConfig (+ env default `NOTIFICATION_LEAD_MINUTES`, 15) and runs `sendScheduledNotificationsForOffset(offset)` for each |
| `appointmentCompletion.cron` | `*/15 * * * *` | Auto-completes/cancels overdue appointments across collections (`appointmentCompletion.service.ts`) |
| `announcement.cron` → `startAnnouncementNotificationCron` | **`*/10 * * * *`** (was `0 8 * * *`; override `ANNOUNCEMENT_NOTIFICATION_CRON_SCHEDULE`) | Finds announcements active today, atomically claims via `notificationProcessingAt`, sends FCM multicast — now also fires **1-hour-before** and **1-day-before** reminders. Gated by its own `ENABLE_ANNOUNCEMENT_NOTIFICATION_CRON` (default on unless `=false`). |
| `announcement.cron` → `startAnnouncementReminderCron` (NEW) | `* * * * *` (every minute) | Sends the per-announcement "1 hour before" reminder for announcements carrying a `startTime`; dedup key `${date}-reminder`. Same `ENABLE_ANNOUNCEMENT_NOTIFICATION_CRON` gate. |
| `careConferenceEnable.cron` | `* * * * *` | Enables care conferences starting within 5 min (`isEnabled=true`, status → IN_PROGRESS) and notifies residents + care team |
| `careConferenceReview.cron` | (own schedule) | Care-conference review-state transitions |

### 5.4 Scheduled reminder pipeline (`src/services/notification.service.ts`)
- Source of truth: `UnifiedSchedule` (mirror of all bookable items: SALON, MASSAGE, PT, CARE, CARE_CONFERENCE, REHAB, TRANSPORTATION, ACTIVITY).
- For each offset: window query (`scheduleDate`+`startTime` between now+offset and now+offset+`NOTIFICATION_WINDOW_MINUTES` (1)), excluding CANCELLED, max `NOTIFICATION_MAX_PER_RUN` (200) per tick.
- Per-facility check: offset must be in the facility's configured offsets for the mapped module/event.
- **Dedup/idempotency**: `NotificationSentLog` with unique `{scheduleId, offsetMinutes}` index — insert acts as an atomic claim across overlapping workers (`notification.service.ts:413-426`).
- Recipients: resident (FCM + history) → that resident's family members (multicast with family-specific body) → staff filtered by access-permission per service (`SERVICE_PERMISSION_MAP`: SALON→Salon, MASSAGE/CARE/CARE_CONFERENCE→Services, PT→Rehab, TRANSPORTATION→Transport) → secondary staff groups with bespoke copy (e.g. Dashboard+Services for ACTIVITY, Rehab for CARE; `SECONDARY_STAFF_GROUPS`). Staff matching is via `$elemMatch` on `accessPermissions` (`utils/staffPermissionQuery.ts`).

### 5.5 Announcements (platform broadcast)
`Announcement` model: title/description/iconType, `type: single|multiple|range`, `audience: family|resident|both` (default both), date fields, **`startTime`/`endTime` time-of-day fields (staging)** driving the new 1-hour-before reminder, `notificationDates[]` (per-day + `${date}-reminder` send ledger), `notificationProcessingAt` (processing lock), soft delete. Creation emits socket `new-announcement` to all clients (`emitNewAnnouncement`); the daily cron sends FCM multicast to all residents and/or family members of the facility with push tokens (500-token chunks), payload `{type: 'ANNOUNCEMENT', announcementId}` (`announcement.notification.service.ts`).

---

## 6. TV platform

### 6.1 Entities
- `TvDevice` `{facilityId, deviceId (unique per facility), createdAt, lastSeenAt}`.
- `TvPairingSession` `{facilityId, sessionId (uuid), qrToken (6-char), deviceId, unitNo, status: PENDING|AUTHORIZED|EXPIRED|USED, cName, groups[], expiresAt (TTL index), authorizedAt}`.
- `TvAuthToken` `{facilityId, cName, deviceId, groups[], refreshTokenHash (sha256, unique), issuedAt, expiresAt (TTL index), lastUsedAt, revoked/revokedAt}` + virtual `resident` populated by cName.

TTLs (`src/utils/tvAuth.ts:27-39`, env-overridable): QR 120 s; access token 30 min; refresh token 30 days. Access token = HS256 JWT signed with `TV_JWT_SECRET`, payload `{cName, device:'tv', deviceId}`.

### 6.2 End-to-end pairing flow (QR)
1. **TV registers**: `POST /api/tv/register {deviceId}` (facility header validated by Zod `facilityHeaderSchema`; upserts TvDevice, updates lastSeenAt). No device authentication — registration is open.
2. **TV requests pairing** — two equivalent transports:
   - HTTP `POST /api/tv/pairing/create {deviceId, unitNo?}` — reuses an existing PENDING unexpired session, else creates one (sessionId uuid + 6-char uppercase `qrToken`), returns `{sessionId, qrToken, expiresIn}`.
   - Socket namespace **`/tv`** event `pairing:create` (headers `facilityid`, `deviceid`, `unitno` required) — same logic; **unitNo is mandatory on the socket path**; a stale session for another room triggers a fresh session; socket joins room `tv-pairing:<facilityId>:<sessionId>`; a one-shot in-memory timer marks expiry and emits `pairing:expired` (`tvPairing.handler.ts:68-108,156-245`).
   The TV renders the QR (sessionId/qrToken) + the 6-char code.
3. **Resident authorizes on mobile**: `POST /api/tv/pairing/authorize {sessionId | qrToken}` with **Cognito auth**. Business rules: session must be PENDING and unexpired; caller must have a Resident profile; **resident.unitNo must equal session.unitNo** (sessions without unitNo are rejected outright — prevents cross-room pairing) (`tvAuth.controller.ts:214-239`). On success: status→AUTHORIZED, stores `cName` + caller's Cognito `groups`, emits `pairing:authorized {sessionId}` to the TV's socket room.
4. **TV exchanges for tokens**: HTTP `POST /api/tv/auth/exchange {sessionId, deviceId}` or socket `auth:exchange`. Validates AUTHORIZED + deviceId match + not expired; **revokes all prior active TvAuthTokens for (cName, deviceId)** (single active session per user-device); issues access JWT + 64-hex refresh token (stored hashed); session → USED. Socket variant additionally persists `groups` on the token (HTTP variant does **not** — inconsistency; TV sessions minted over HTTP have `groups: undefined` → `req.user.groups = []`).
5. **Refresh**: `POST /api/tv/auth/refresh {refreshToken}` — hash lookup, facility match, revocation/expiry checks, returns new access token, bumps `lastUsedAt`. No refresh-token rotation.

### 6.3 TV request authentication (`authTv`, `authMiddleware.ts:154-204`)
Requests with `istv: true` header: verify HS256 JWT, require payload `{device:'tv', cName, deviceId}`, then require a **live TvAuthToken record** (facility + cName + deviceId, not revoked, not expired) — i.e. server-side session revocation works even within the JWT's 30-min window. Resulting `req.user = {username: cName, groups: tokenRecord.groups ?? [], isTv: true}` — the TV impersonates the authorizing resident.

### 6.4 TV content endpoints
- `GET /api/config/getHotelForAndroidTV` — full facility config for the TV launcher (name inherited from the Shashi Hotels codebase).
- `GET /api/android-tv/get-android-categories/:facilityId` (optional auth) — entertainment category tree from `AndroidCategories` (loose `categories/subCategories/noSubCategories/userpreferencesData` objects; `isShashiId`, `emailId` fields betray hotel-app lineage). `POST /android-categories` is a **stub** — its insert logic is commented out and it returns 201 unconditionally (`routes/androidCategories.ts:17-26`).
- Weather for the TV dashboard: `GET /api/config/currentTemperature` proxies OpenWeather with a **hardcoded API key in source** (`config.controller.ts:254`).

---

## 7. App version management
`AppVersion` `{bundleId (unique), androidForceUpdateVersion, androidVersion, iosForceUpdateVersion, iosVersion}`. Single public endpoint `GET /api/app-version/:bundleId` (mounted **before** facilityMiddleware, so no facility header needed). Supports min-version force-update prompts per mobile app bundle. No write API — managed directly in DB.

---

## 8. Cross-cutting infrastructure

### 8.1 File storage (S3 + CloudFront)
- Multer in-memory upload middleware (`s3Upload`, `s3UploadAny`, `s3UploadMultiple`, `chatUpload`) with 30 MB cap and an extension/MIME allowlist covering images/video/audio/docs (error message stale: says "JPG, JPEG, PNG, WEBP, PDF").
- `uploadToS3(req, folder)` streams to `AWS_S3_BUCKET_NAME` under folder keys: `staff/`, `admins/`, `profile-pictures/`, `items/`, `resident/`, chat attachments, gallery images, referral PDFs, lab reports, acknowledgement PDFs.
- Read URLs: `toSignedUrl()` now returns **public CloudFront URLs** (`https://$CLOUDFRONT_DOMAIN/<key>`) — naming is legacy; no signing actually occurs (`s3.service.ts`).
- Direct-upload path: `GET /uploads/presigned` (auth) returns a 1-hour presigned **PUT** URL for `<folder>/<ts>-<sanitized filename>`; an older multipart `/uploads` endpoint is commented out in `app.ts:166-175`.
- Chat attachments are KMS-envelope-encrypted (see `utils/kmsEnvelope.ts`, `chatEncryption.service.ts`); PCC medication data likewise (`pcc_medication_wrappedKeyB64` on Resident).

### 8.2 Error handling conventions
- Two coexisting wrappers: `asyncHandler` (`errors/global-error-handler.ts`) forwarding to the single `errorHandler` (`middleware/error-handler.ts`) which maps ZodError→400 (field list), Mongoose ValidationError→400, MulterError→400, "Request aborted"→400, `AppError` (statusCode-carrying)→its status, else 500. Mounted last in `app.ts:222`.
- In practice **most controllers try/catch inline** and hand-roll `{success, message}` JSON with mixed shapes (`{message}` vs `{success,message,data}` vs `{error}`); `console.log/console.error` is the only logging (Morgan `dev` for HTTP). No request IDs, no structured logger.

### 8.3 Google Calendar & Zoom integrations (staff-linked OAuth)
- `Staff` carries `googleRefreshToken`, `isGoogleLinked`, calendar push-channel fields (`googleCalendarChannelId/ResourceId/ChannelExpiration`), `cachedCalendarBusySlots[]`, `lastCalendarSyncAt`; and `zoomRefreshToken`, `isZoomLinked`, `zoomUserId`, `zoomAuthRequired`.
- Flow (routes under `/auth`, `callback.routes.ts` — **no auth on any of them**): `GET /auth/google/url?cognitoSub=` → consent URL (scope `calendar.events`, offline, state=cognitoSub) → `GET /auth/callback` stores refresh token on the Staff row (**`upsert: true`** — a forged state can create a junk Staff doc, `callback.controller.ts:69-76`). Webhook `POST /auth/google/calendar/webhook` receives Google push notifications and refreshes `cachedCalendarBusySlots`; manual `register-watch`, `sync-now`, `busy-slots` endpoints support ngrok-based dev (`GOOGLE_CALENDAR_WEBHOOK_BASE_URL`).
- `googleCalendarSync.service.ts` mirrors appointments (Salon, Massage, PT, Care, CareConference) into the staff member's Google Calendar using extended properties (`modelType`, `appointmentId`, `facilityId`) for two-way reconciliation.
- Zoom: OAuth URL/callback (`/auth/zoom/*`), used by care-conference/long-meeting creation (`zoomMeeting.service.ts`); `zoomAuthRequired` flags re-link prompts. Zoom webhooks mounted at `/webhooks/zoom` (Zoom-signed), recordings callback at `/zoom`.

### 8.4 Socket.io event catalogue (`src/config/socket.ts` + handlers)

| Namespace | Auth | Direction | Event | Audience / room |
|---|---|---|---|---|
| `/` (default) | none | S→C | `new-announcement` | broadcast to all connected clients (optionally excluding sender socket) |
| `/` | none | S→C | `mobile-<Model>-request-upserted` / `-deleted` (UnifiedSchedule models: SalonAppointment, MassageAppointment, PrivateTrainingAppointment, RehabAppointment, Care, CareConference, TransportationRequest, Schedule…) | **global broadcast** (`io.emit`) — no facility scoping |
| `/` | none | S→C | `mobile-<module>-request-upserted/deleted` (generic `emitAppRequestEvent`) | global broadcast |
| `/tv` | facility header only | C→S | `pairing:create`, `auth:exchange` | TV devices |
| `/tv` | — | S→C | `pairing:created`, `pairing:authorized`, `pairing:expired`, `pairing:error`, `auth:tokens`, `auth:error` | room `tv-pairing:<facilityId>:<sessionId>` or requesting socket |
| `/notifications` | Cognito JWT + facility | S→C | `notification:new` | room `user:<cName>` |
| `/chat` | Cognito JWT + facility (family→resident normalization) | C→S | `chat:delivered` (`message:delivered`), `chat:read` (`message:read`) with ack callbacks | — |
| `/chat` | — | S→C | `chat:new`, `chat:status`, `chat:group`, `chat:deleted`, `chat:reaction`, `chat:error`, `chat:unread` (on connect) | rooms `user:<cName>`, `facility:<facilityId>` |

### 8.5 Inbound webhooks
- `POST /webhooks/pcc` — PointClickCare events routed via `webhooks/registry.ts`; **no signature verification** in the route. The registry grew **12 → 14 handlers** on staging, adding `patient.admit` (notifies active managers via FCM + Twilio SMS) and `patient.updateContactInfo` (syncs resident + `familyMember` contact data, removing deleted contacts) — `registry.ts:31-32`. Existing handlers cover patient discharge/transfer/update and medication add/update/discontinue/strikeout/cancelDiscontinue.
- `POST /webhooks/tels/*` — TELS maintenance/work-order callbacks; `telsWebhookAuth` validates `X-API-KEY` with timing-safe compare against `TELS_WEBHOOK_SECRET`, but **skips auth entirely (with a warning) if the secret is unset**.
- `POST /webhooks/lemedix/*` — CrelioHealth lab events; **no auth**, fire-and-forget processing.
- `POST /webhooks/zoom` — Zoom-signed payloads.

### 8.6 New outbound integrations (staging)
- **AWS SES** (`@aws-sdk/client-ses`) — transactional email with PDF attachments via `SendRawEmailCommand` (`src/config/ses.ts`, `src/services/email.service.ts`); used by the referral agency-email flow (`POST /api/referrals/send-referrals-emails`).
- **Twilio** (`twilio` 6.0.2) — transactional SMS `sendSms` (`src/services/sms.service.ts`), env-gated on `TWILIO_ACCOUNT_SID` / `TWILIO_AUTH_TOKEN` / `TWILIO_FROM_NUMBER`; used by the PCC `patient.admit` manager alerts. **AWS SNS is retained** for reset-OTP `sendOtpSms`.
- **Documo** (fax) — outbound HTTP fax integration (`src/integrations/documo/`), `POST /api/fax/send` etc.; env-gated on `DOCUMO_API_KEY`, Basic auth, base `https://api.documo.com`. `/api/fax` mounts before the global facility gate (`app.ts:103`) and its facility+auth guards are **bypassable via `FAX_LOCAL_BYPASS=true`** with no production fail-closed (G-23). See backend-clinical-care §7a.
- **`@cantoo/pdf-lib`** — password-protects emailed referral PDFs.

---

## 9. Product-split signals (complete inventory)

| Signal | Location | Gating effect |
|---|---|---|
| `CARE_TYPE_VALUES = assisted_living, memory_care, independent_living, skilled_nursing` | `src/contants/index.ts:1-8` (note misspelled folder) | Resident classification; `?careType=` filters on resident/diet/housekeeping listings |
| `Resident.careType` enum (required) | `resident.model.ts:113-117` | Per-resident product flavor |
| `THERAPY_TYPES` with `OTHER` ⇒ SKILLED_NURSING; `ASSISTED_LIVING_THERAPY_TYPES` (fixed 4) | `src/constants/rehab.ts:38-101` | AL: fixed therapies configured in `Config.rehab.*`; SN: dynamic `RehabTherapy` catalogue via `therapyId` (`isRehabTherapyIdRequired`) |
| `RehabAppointment` comments: "`therapyType === 'OTHER'` → appointment belongs to SKILLED_NURSING" | `RehabAppointment.model.ts:23,62` | Same fork at the data layer |
| `SUPERVISION_STAFF_DESIGNATIONS` (Executive Director … Front Desk) | `rehab.ts:21-35` | SNF-style org chart baked into the `supervision` designation group |
| `Config.accessPages[].isHidden` | `config.model.ts:105-116`, `accessDefaults.ts` | Per-facility module on/off (the de-facto feature flag system) |
| `Config.designations` (free-form per facility) | `config.model.ts:179` + designation CRUD | Facility role catalogue differs between AL and SNF deployments |
| `Config.pms[]` + `integratedModules` (`POINTCLICKCARE`, `TELS`, …) | `config.model.ts:50-83` | PMS-of-record per module |
| `IntegrationAvailable` (name `pcc`) | `IntegrationAvailable.model.ts` | PCC org/fac credentials per facility; presence ⇒ SNF-style clinical sync (patients, medications, webhooks) |
| `referral` home-services checklist incl. `skilledNursing` flag; referral PDF: "ready for discharge from SNF" | `referral.sign.service.ts:81`, `referralPdfTemplate.ts:567` | SNF discharge/referral workflow |
| IDT reports, Care Conferences, Lab reports (Lemedix), Medications (PCC) | respective modules | Clinically-oriented (SNF) feature cluster |
| Salon / Massage / Private Training / Family meals / Brain games / Gallery | respective modules | Lifestyle (Senior Living) feature cluster — disabled per facility only via accessPages/bookingPermission |

**Conclusion:** feature availability per facility = `Config.accessPages` (visibility) ∩ staff `accessPermissions` (who) ∩ `bookingPermission` (who may book for whom) ∩ integration presence (PCC/TELS/Lemedix), with the AL↔SN rehab fork keyed off therapyType/careType — not a single product flag.

---

## 10. Observations (TODOs, dead code, inconsistencies, risks)

**Structural quirks**
1. **Duplicated constants folders**: `src/constants/` and misspelled `src/contants/` both exist and are both actively imported (`contants/` holds careType values, Cognito roles, transportation). Consolidation needed.
2. Misspellings frozen into contracts: route `PUT /api/staff/availabilty/:staffCName` (typo deliberately preserved, `staff.route.ts:59-62`), DB keys `Config.rehab.physicalThearapy` / `rehabAvaulation`, `IntegrationAvailable.timzone`, file `cognnito.types.ts`.
3. `Config` interface vs schema drift: `designations`, `facilityName`, `tvSetupLocations`, `familyRelations` are read/written but not in the Mongoose schema (writes use `strict:false`).
4. `Resident.status` TS type omits `Transferred` though schema allows it.
5. ~~`/api/reports` is mounted twice in `app.ts`.~~ **Resolved on staging** — single mount at `app.ts:209` (tech-debt T-3 closed).

**Dead / stub / mock code**
6. `GET /api/admin/getAdminData` returns **hardcoded mock dashboard data** (fake appointments/activities, residentCount 100) — `admin.controller.ts:54-136`.
7. `POST /api/android-tv/android-categories` is a no-op stub (insert commented out).
8. Commented-out multipart `/uploads` endpoint in `app.ts`; commented `oauth2Client` import in `auth.controller.ts`; `auth.controller.ts` duplicates `getGoogleAuthUrl` already in `googleAuth.controller.ts` (only the latter is routed).
9. Only 2 TODOs in the codebase (both in `appointmentCompletion.service.ts` about per-document updates).

**Security-relevant gaps (for PRD/eng follow-up)**
10. `facilityMiddleware` missing-header check never triggers (checks `null`/`''` but value is `undefined`) — facility scoping is effectively advisory; combined with `getFacilityFilter` returning `{}`, an omitted header widens queries across tenants.
11. **Unauthenticated mutating endpoints**: `POST /api/residents` (create resident + Cognito accounts), `GET/PUT/DELETE /api/residents/:id`, `GET /api/admin/`, `PATCH /api/staff/:id/toggle`, `PATCH /api/staff/:id/access-permissions`, `DELETE /api/staff/:id`, all of `/api/items` CRUD, most `/api/config` writes (`PUT /access-pages`, `/mealconfig`, `/acknowledgement-pdf`, `/default-permissions`) carry **no authMiddleware** — only the (broken) facility header requirement.
12. Hardcoded OpenWeather API key in `config.controller.ts:254`.
13. PCC and Lemedix webhooks unauthenticated; TELS webhook auth silently disabled when secret missing.
14. `GET /api/cognito/export` lets **any authenticated user** (any role) export the entire user pool as CSV, and writes the CSV to local disk.
15. Google OAuth callback upserts Staff by attacker-controlled `state` (cognitoSub) without validation.
16. Unified-schedule socket events broadcast globally (`io.emit`) — cross-facility data leakage to any connected client on the default namespace (which requires no auth).
17. TV HTTP `auth/exchange` doesn't persist Cognito `groups` on the token (socket path does) — TVs paired over HTTP get empty groups, so any TV-side role checks differ by transport.
18. CSV export, resident create, etc. write into the repo-local `uploads/` directory (also used by Multer historically) — ephemeral in containers.
19. `package.json` has no `seed:config` script although repo CLAUDE.md documents one (docs drift).

**Design observations**
20. Identity is phone-first everywhere; email is optional metadata. Phone numbers are globally unique per persona collection (`countryCode+phone` unique indexes on Admin/Staff; per-resident-facility index for residents; per-resident for family members). A person can hold multiple personas only via the same phone resolving (`createOrResolveCognitoUserByPhone`) — family members reuse Cognito users across facilities, but are blocked from linking to two residents.
21. Cognito↔Mongo rollback choreography is consistent (track created users, delete on transaction abort), but Cognito group membership/MFA changes are not transactional — partial failures can leave group-less users.
22. The "facility" concept is conflated with the Config doc; multi-facility orgs, facility lifecycle (create/archive), and cross-facility admin are absent from the API surface (config docs are created out-of-band).

**Staging additions**
23. `/api/fax/*` (Documo) mounts before the global facility gate (`app.ts:103`) and its facility+auth guards are bypassable via `FAX_LOCAL_BYPASS=true` with no production fail-closed (G-23, High). No fax send retry/idempotency (G-24).
24. `GET /api/residents/pcc-contacts` is **unauthenticated** (declared before `authMiddleware`, `residents.route.ts:64`) — widens the unauthenticated-read surface; PCC contact-sync handlers also expand the forge-able write surface onto `familyMember` (G-3/G-5 widened).
25. `migrateAssignedStaff.ts` (legacy care-team fields → `assignedStaff[]`) is a manual one-time script not wired into CI/CD or boot — a missed run silently empties care-team filters (T-14).
26. `isDevelopmentEnv()` treating pre-production as dev-like means pre-prod still issues the static `TempP@ssword123` and widens other dev-only branches into pre-prod (G-11 Medium / T-15 Low).
27. The chat module was decomposed into `controllers/chat/` + `services/chat/{conversation,message}/` packages — a positive structural counter-example to the still-growing god-class controllers (resident 2997, transport 2453, housekeeping 1876, staff 1348 lines on staging).
