# Backend Codebase Analysis — Clinical / Care-Coordination Modules

> Reverse-engineered functional requirements from `senior_living_backend` source code.
> Scope: Medications, Labs, Referrals, Care coordination (Care, CareConference, IDT, ACD), Rehab/therapy, Agencies & case managers, Messaging/chat, Fax.
> Source of truth: code under `senior_living_backend/src/` as of `pre-production` `5fbe9a3c` (2026-08-28). All paths relative to that repo unless absolute.
> **2026-08-28 delta (`e6469276..5fbe9a3c`, `assign-practitioner-directly-via-pcc-pre-prod` branch):** new §1a — PCC **practitioner** sync (new `Practitioner` model + data pipeline) now auto-populates `Resident.assignedStaff[]` by name-matching against Staff, with skip-and-email-alert on an ambiguous match and matching-based unassignment on practitioner removal. Also new: a second, independent PCC OAuth/mTLS module (`pcc.core.ts`) and a new resource-agnostic shared PCC envelope key (`Resident.pcc_wrappedKeyB64`), both introduced for this feature (Observations 23–24).

---

## 0. Cross-cutting context (read first)

### Personas / roles (Cognito groups — `src/contants/cognnito.types.ts`)
`RESIDENT`, `FAMILY_MEMBER`, `STAFF`, `ADMIN`, `SUPER_ADMIN`. TV devices authenticate with a custom TV token (`authMiddleware` → `authTv`, `src/middleware/authMiddleware.ts:154`).

- **Family members** are normalized at auth time: `req.user.username` is rewritten to the **linked resident's cName** (`authMiddleware.ts:94-145`); the original family identity is preserved in `familyMemberCName`. Downstream code can therefore treat family members as "acting as the resident" for reads.
- **Staff designations** drive clinical roles. Care-team designations (`src/constants/designations.ts`): `Nurse`, `Case Manager`, `Social Worker`, `Doctor`, `Dietitian`. **Care-team model refactor (staging):** the resident's five legacy per-role fields (`nurse`/`caseManager`/`socialWorker`/`doctor`/`dietitian`) and the `DESIGNATION_TO_CARE_TEAM_FIELD` / `RESIDENT_CARE_TEAM_FIELDS` maps have been **removed**, replaced by a single indexed `assignedStaff: string[]` (array of staff `cName`s) on the Resident document (`resident.model.ts:33,94-97`), plus an `assignedStaffDocs` populate virtual (`resident.model.ts:153-156`). Assignment/lookups now go through `src/lib/assignedStaff.ts` (`isStaffAssignedToResident`, `getResidentAssignedStaff`) and `src/utils/assignedStaff.ts` (`notifyAssignedStaff`). A one-time, manually-run migration script `src/scripts/migrateAssignedStaff.ts` consolidates the legacy fields into `assignedStaff[]` (not wired into CI/boot — Tech-debt T-14). **As of 2026-08-28, `assignedStaff[]` also gains a second, automated writer** — PCC practitioner sync (§1a) — alongside the pre-existing manual admin-form writer. Rehab-eligible designations (`src/constants/rehab.ts:13`): `Director of Rehab`, `Rehab Therapists`. A large "supervision" group also exists (`rehab.ts:21` — Executive Director, Administrator, Director of Nursing, Case Manager, Dietary Services, Front Desk, etc.).
- **Access permissions** (module-level, used in notification fan-out & staff app): `ACCESS_PERMISSIONS` includes `REHAB`, `SERVICES`, `DASHBOARD`, etc. (`cognnito.types.ts:9`).

### Multi-tenancy
Every `/api/*` route passes `facilityMiddleware` (mounted at `app.ts:104`); controllers scope queries with `getFacilityFilter(req)` / `getFacilityId(req)` from the `x-facility-id` header. **Exception:** `/api/fax/*` mounts *before* the global facility gate at `app.ts:103` and carries its own per-route guards, which are bypassable via `FAX_LOCAL_BYPASS=true` (see §11 Fax; Observation 19).

### Booking context middleware (`src/middleware/bookingContextMiddleware.ts`)
`resolveBookingContext(modelKey)` decides **who is booking for whom** and enforces the facility's per-module booking policy stored at `Config.bookingPermission[modelKey]`:
- RESIDENT: always allowed self-service; any `residentCName` in request is ignored (spoof guard).
- FAMILY_MEMBER: allowed iff `policy.isFamilyMemberAllowed === true`; books for linked resident.
- STAFF: must supply a resident identifier (`residentCName`, legacy `cName`, or legacy `residentId`); allowed iff staff `designation` ∈ `policy.staffDesignationAllowed` (or in an allowed designation group, e.g. `rehab`, `supervision` — `designationGroup.ts`).
- ADMIN/SUPER_ADMIN: bypasses designation check but the policy must exist (per-facility opt-in).
Result is `req.bookingContext = { createdByType, createdByCName, residentCName }`, persisted on Care and RehabAppointment documents.

### UnifiedSchedule sync
`Care`, `RehabAppointment`, and `CareConference` models all have post-save/post-update/post-delete hooks that mirror documents into a `UnifiedSchedule` collection (`scheduleSync.service`), which powers the resident master calendar and **cross-venue conflict blocking** (`BLOCKING_SCHEDULE_TYPES = ['SALON','MASSAGE','PT','CARE','REHAB']`, `rehabAvailability.service.ts:57`).

### Auto-completion cron
`src/jobs/appointmentCompletion.cron.ts` (default `*/15 * * * *`, gated by `ENABLE_APPOINTMENT_COMPLETION_CRON`) calls `appointmentCompletion.service.ts`, which auto-transitions overdue CONFIRMED/SCHEDULED appointments to COMPLETED across `Care`, `Salon`, `Massage`, `PrivateTraining`, `RehabAppointment`, `CareConference` — one-by-one updates to preserve Mongoose hooks (TODO comments at `appointmentCompletion.service.ts:126,209`).

---

## 1. Medications

**Files:** `src/models/Medication.model.ts`, `src/routes/medication.routes.ts` (mounted `/api/medications`), `src/controllers/medication.controller.ts`, `src/validation/medication.schema.ts`, `src/utils/medicationPdfTemplate.ts`, PCC integration under `src/integrations/pms/pcc/`.

### Purpose
Per-resident medication list with **two data sources**:
1. **Manual entry** ("legacy" fields: `medicationName`, `strength`, `route`, `frequency`, `prescribingDoctor`, `startDate`) — created via API by the authenticated user for themselves.
2. **PCC (PointClickCare) PMS sync** — full PCC medication order payloads stored **field-encrypted** in `medication_data` (every value AES-encrypted with a per-resident KMS envelope data key; keys visible, values ciphertext — `Medication.model.ts:17-18`).

### Personas & permissions
All medication routes require only `authMiddleware` — **no role middleware**:
- `GET /api/medications/` — "my medication list" for the authenticated cName (resident self-view; family members see linked resident's list via username normalization). Returns only `status: 'ACTIVE'`.
- `GET /api/medications/resident?cName=...&status=...` — staff/admin view of any resident's medications (status filter defaults ACTIVE). **No role check** — any authenticated identity can query any resident by cName (PHI exposure risk; tenant isolation IS enforced via facilityId).
- `GET /api/medications/resident/pdf?cName=...` — generates a medication-list PDF (Puppeteer `htmlToPdfBuffer`) with patient name, room, DOB, doctor, generated date, and a table of name/strength/type/route/start date/directions (`medication.controller.ts:247-349`, template `medicationPdfTemplate.ts`).
- `POST /` create (manual; status enum limited to ACTIVE/INACTIVE/COMPLETED via Zod), `GET/PUT/DELETE /:id` — scoped to the **caller's own** cName (`filter { _id, cName }`), so manual CRUD is effectively self-service only.
- DELETE is a **soft delete**: sets `status: 'INACTIVE'` (`medication.controller.ts:518`).

### Status model
`ACTIVE | INACTIVE | COMPLETED | UPDATED | PENDING | DISCONTINUED | INITIAL | STRIKEOUT | CANCEL_DISCONTINUE` (`Medication.model.ts:13`). Manual API only uses the first three; the rest mirror PCC order lifecycle (`mapMedStatus`, `pcc.service.ts:342`).

### PCC sync flow (webhook-driven)
PCC webhooks (`/webhooks/pcc`, `src/integrations/pms/pcc/webhooks/handlers/medication/`) handle `medication.add / update / discontinue / strikeout / cancelDiscontinue`:
1. Resolve resident by `pcc_patientId` + `pcc_orgUuid` (case-insensitive regex) — skip if no match.
2. Fetch full medication list from PCC preview1 API (paged 200/page, OAuth client-credentials with mTLS cert agent + per-client rate limiting; token cached with 5-min expiry buffer — `pcc.service.ts`).
3. Encrypt every payload field with the resident's medication data key (`resolveResidentMedicationKey` — KMS-wrapped key stored on `Resident.pcc_medication_wrappedKeyB64`, generated on first write, cached in-process; read paths pass `readonly=true` and degrade to empty `medication_data` if no key — `pcc.service.ts:22-57`). **Unchanged in this pass** — still a separate key/module from the new practitioner sync's shared key (§1a).
4. Upsert by `{facilityId, cName, orderId}`; status mapped via `mapMedStatus`. Discontinue stores `discontinueDate`. If an orderId disappears from PCC, the local record is marked with the event's terminal status (`DISCONTINUED`/`STRIKEOUT`/`CANCEL_DISCONTINUE`) (`medicationSync.ts:71-140`). `medication.update` deletes-then-reinserts the order (`medicationSync.ts:142-208`).

### Read-path decryption & shaping
List endpoints decrypt `medication_data` per record and reduce it to a display object: name/generic, strength (`dose doseUOM • route • scheduleType`), directions ("Take {dose} {doseUOM} {route} {frequency}"), start/end dates (formatted in the PCC integration's timezone, default `America/Los_Angeles`; field is literally `timzone` [sic] on `IntegrationAvailable`), prescribing doctor, narcotic flag (`formatMedicationData`, `medication.controller.ts:35-69`). Decryption failure or missing key → `medication_data: []` (graceful PHI redaction rather than 500).

### Notifications / events
None for medications (no FCM/socket emission found).

---

## 1a. Practitioners (PCC sync + automatic care-team assignment) — NEW (2026-08-28)

**Files:** `src/models/Practitioner.model.ts` (new); `src/integrations/pms/pcc/pcc.practitioners.ts`, `pcc.residentKeys.ts`, `pcc.core.ts` (all new); `src/integrations/pms/pcc/types/pcc.practitioners.types.ts` (new); `src/integrations/pms/pcc/webhooks/shared/practitionerSync.ts`, `practitionerStaffAssignment.ts` (both new); `webhooks/handlers/practitioner/medicalProfessionalAdd.ts`, `medicalProfessionalRemove.ts` (new); `src/constants/pccAlerts.ts` (new); `resident.model.ts` (+`pcc_wrappedKeyB64`); `webhooks/shared/residentOnboarding.ts` (+`syncResidentPractitioners`); `webhooks/handlers/patient/readmit.ts` (+1 call).

### Purpose
A new PCC-sourced clinical entity — a resident's attending/consulting physicians ("practitioners") — is now synced from PointClickCare and used to **automatically populate `Resident.assignedStaff[]`** by name-matching each practitioner against the facility's `Staff` roster. This is the first PCC integration to write to the care-team array that §0 / `docs/prd/modules/care-coordination.md` CARE-FR-63/BR-7 describe as the shared care-team spine (chat eligibility, notification fan-out, "my residents" scoping).

### Data model
`Practitioner` (`Practitioner.model.ts`): `{facilityId, cName, residentId?, pcc_patientId?, facId?, practitionerId?, status: 'active'|'inactive' (default active), source, practitioner_data}` — one document per `{facilityId, cName, practitionerId}` (unique compound index; also indexed `{facilityId, pcc_patientId, status}`). `practitioner_data` is the **full PCC practitioner payload, field-encrypted** the same shape as `Medication.medication_data` (field keys visible, values AES ciphertext): practitionerId, title, first/last name, address fields, phones, email, NPI, providerType, licence number, createdBy/revisionBy, etc. (27 fields, `types/pcc.practitioners.types.ts`).

### New shared PCC envelope key (coexists with medication's own key)
`resolveResidentPccKey(cName, facilityId)` (`pcc.residentKeys.ts`) generates/reads a **new, resource-agnostic** KMS-wrapped data key stored on `Resident.pcc_wrappedKeyB64` — the module comment describes it as "used for ALL resources (current and future)". Practitioners are the first (and, as of this pass, only) resource on it; **medications keep their own, separate, pre-existing key** (`Resident.pcc_medication_wrappedKeyB64`, `pcc.service.ts`'s `resolveResidentMedicationKey`, untouched this pass) — two independent per-resident PCC keys now coexist on the same Resident document. `pcc.residentKeys.ts` also ships a **read-only legacy-key fallback** (`resolveLegacyResidentPccKey`, one field name per resource: medication/nutritionOrder/allergyIntolerance/observation/diagnosticReport) scaffolded for a **future** migration of those resources onto the shared key — not called by any resource yet, including medication (dead code today). Key generation is single-flight de-duped in-process, then claimed atomically in Mongo (`findOneAndUpdate({pcc_wrappedKeyB64: {$exists:false}})`) so concurrent first-syncs across server instances converge on one key instead of each instance writing its own now-orphaned KMS ciphertext.

### Duplicate OAuth/token-cache module (`pcc.core.ts`)
`pcc.core.ts` (new) re-implements `resolvePccConfig`, `getPccAccessToken` (its own module-level `tokenCache` Map, 5-min expiry buffer), and `getPccHttpsAgent` (its own mTLS `https.Agent` singleton) **independently of** — not by importing — the pre-existing, identical functions already in `pcc.service.ts`. Only the new practitioner-sync files (`pcc.practitioners.ts`, `practitionerSync.ts`) import from `pcc.core.ts`; every other PCC flow (medication sync, patient webhooks, the manual `/pcc-sync/*` tool) still imports `pcc.service.ts`'s copies. Net effect: two separate in-memory OAuth token caches and two separate mTLS agents now exist for the same PCC client per process — see Observation 23.

### Sync flow (`pcc.practitioners.ts`)
`fetchPccPractitionerList` pages `GET {baseUrl}/api/public/preview1/orgs/{orgUuid}/practitioners?patientId=` (loop on `paging.hasMore`) using `pcc.core.ts`'s token/agent. `syncAndStorePractitioners` fetches (optionally filtered to specific `practitionerId`s), encrypts every field with the shared key, and `bulkWrite`-upserts by `{facilityId, cName, practitionerId}`, always writing `status: 'active'` — an upsert never itself marks a practitioner inactive; that only happens via the `remove` event, below.

### Webhook wiring (`webhooks/registry.ts`) — registry now **16 handlers** (was 14)
Two new event types on the existing unauthenticated `POST /webhooks/pcc` endpoint (same exposure class as CLIN-GAP-15 in `clinical-records.md`):
- **`medicalProfessional.add`** → `syncPractitionersByResourceId` (`practitionerSync.ts:73-124`): resolves the resident by `pcc_patientId`+`pcc_orgUuid`; **if no local resident exists yet**, backfills via the same shared onboarding path as `patient.admit` (`onboardMissingResidentForPractitionerEvent`, :22-70, `sendWelcome: false` — a silent backfill, not a genuine new-admission notification). If the resident exists, runs a **full** (not `resourceId`-filtered) practitioner re-sync — the code comments this is deliberate: PCC's practitioners-list read API can lag the webhook that announced the new practitioner, so filtering to just that `resourceId` risks a zero-match race and silently dropping it (`practitionerSync.ts:100-104`). After syncing, calls `assignStaffFromPractitioners`.
- **`medicalProfessional.remove`** → `practitionerStatusOnly(..., 'inactive')` (`practitionerSync.ts:132-186`): for each `resourceId`, `$set`s the stored `Practitioner.status` to `inactive` (logs+audits a no-op if no local record matches that `practitionerId`), then decrypts the now-inactive record(s)' name fields via the shared key and calls `unassignStaffForRemovedPractitioners`. Decrypt/unassign failures are caught and logged without affecting the status write.

### Automatic staff (care-team) assignment — `practitionerStaffAssignment.ts`
- **Add path** (`assignStaffFromPractitioners`, :107-148): for each synced practitioner, `matchStaffCNameByName` (:55-77) does an exact, case/whitespace-insensitive match of `firstName+lastName` (title ignored) against **active** `Staff` in the facility.
  - **0 matches:** skip — no assignment, no error.
  - **1 match:** `$addToSet`s that Staff `cName` onto `Resident.assignedStaff` (batched — every matched cName across all synced practitioners in one `updateOne`).
  - **2+ matches (ambiguous):** does **not** guess — skips assigning that practitioner and calls `sendDuplicateStaffAlert` instead. Code comment: "a wrong automatic assignment is worse than one requiring manual follow-up" (:103-106).
- **Remove path** (`unassignStaffForRemovedPractitioners`, :149-168): matches **every** active Staff cName sharing the removed practitioner's name — not just one, via `matchAllStaffCNamesByName` (:87-97) — and `$pull`s all of them from `assignedStaff`. Deliberately over-inclusive versus the add path's exact-one-or-alert, because the code cannot know which duplicate-named Staff record was actually the one auto-assigned earlier; `$pull` only removes whichever is actually present, so this can't misfire.
- **Duplicate-staff alert email** (`sendDuplicateStaffAlert`, :12-40): on an ambiguous add-path match, emails `DUPLICATE_STAFF_ALERT_EMAILS` (`src/constants/pccAlerts.ts` — **three hardcoded individual addresses at `pardypanda.com`**, i.e. the implementing engineering team, not a facility contact or a `Config`-sourced address) with facility name, resident name/id, practitioner name/id, and the colliding Staff ids. **This is an internal ops/dev notification with no resident-, family-, or facility-staff-visible surface** — nothing in admin web or either mobile app indicates that a physician failed to auto-attach to a resident's care team. See Observation 24 (candidate gap; not treated as a product requirement in its own right — recipients are not facility-configurable and the alert is not resident/staff/family-facing).

### Folded into admission / re-admission
`syncResidentPractitioners` (`webhooks/shared/residentOnboarding.ts`, new; try/catch, never throws) is now called from:
1. `onboardNewResident` — used by both `patient.admit` and `patient.readmit`'s "resident not found" path — alongside the pre-existing medication + contact sync, all three in one `Promise.all`. `patient.admit`'s own handler file needed **no changes** to gain this; it inherits it via the shared onboarding function it already called.
2. `patient.readmit`'s "resident exists" path (`readmit.ts:48-57`, new call) — previously readmit only converged status/demographics; it now **also** re-syncs practitioners and re-runs staff assignment on every readmit of an already-known resident, not only on first admission.

### Notifications / events
None resident/family/staff-facing (no FCM/socket emission for practitioner sync itself). The only notification-shaped side effect is the internal ops duplicate-staff-alert email, above.

---

## 2. Labs — LabPatient, LabReport, TestResult

**Files:** `src/models/LabPatient.model.ts`, `LabReport.model.ts`, `TestResult.model.ts`; `src/routes/labreports.routes.ts` (`/api/lab-reports`), `lemedix.webhook.routes.ts` (`/webhooks/lemedix`), `testResult.routes.ts` (`/api/test-results`); `src/services/lemedix.service.ts`; `src/controllers/labReports.controller.ts`, `testResult.controller.ts`.

### 2a. External lab integration (Lemedix / CrelioHealth)
Two unauthenticated webhooks (no signature verification, immediate 200 ACK, async processing):
- `POST /webhooks/lemedix/patient-registered` → `handlePatientRegistration`: stores a `LabPatient` record (demographics, insurance details array, referralId, raw payload). Facility resolved via `LEMEDIX_FACILITY_ID` env var ("must be set for the Redwood Grove account" — single-tenant mapping hack; falls back to using CrelioHealth `org_id` as facilityId, `lemedix.service.ts:13-20`). **LabPatient is never linked to a Resident** — it is a raw registry.
- `POST /webhooks/lemedix/report-submit` → `handleReportSubmit`:
  1. Resolve facility via `IntegrationAvailable.externalId === labId` (multi-tenant capable, unlike patient registration).
  2. **Resident matching by contact data**: `$or` over phone fields (`Patient Contact`, `Patient Alternate Contact`, `Contact No`) and `alternateEmail` against `Resident.phone/email`. Facility mismatch or no match → webhook ignored (`lemedix.service.ts:153-193`).
  3. Decode `reportBase64` PDF → upload to S3 under `lab-reports/{centreReportId}-{ts}.pdf`; base64 blob excluded from the stored `rawPayload`.
  4. Create `LabReport` with structured `reportFormatAndValues` (per-analyte value + `highlight`/`critical` flags + gender-specific reference bounds + unit), report metadata (sampleId, isSigned, branch), and links `residentId`/`cName`/`integrationId`.
- A catch-all `POST /webhooks/lemedix/` also routes to `handleReportSubmit` (logs "unknown webhook event").

### 2b. Resident visibility (`/api/lab-reports`)
- `GET /` — authenticated resident (or family member acting as resident) sees **their own** reports: filter `{cName, facilityId}`, optional `search` (testName regex) and `reportDate` (UTC day window). Returns testName, patientName, S3 key + presigned `signedUrl`, formatted report date. *Bug/observation: pagination `skip` is computed but commented out of the query (`labReports.controller.ts:114-120`), and there is leftover commented code + a `console.log` of the query.*
- `GET /:residentId` — staff/any-authed view of a specific resident's reports (resident resolved by `_id` within facility). No role middleware.
- The structured analyte values (`reportFormatAndValues`) are stored but **not exposed** by these endpoints — only the PDF link is surfaced. (Half-built analytics surface.)

### 2c. TestResult — manual uploads (self-service document locker)
`TestResult` is a simple per-cName file record (name, fileName, fileType pdf/image, S3 fileUrl, isActive). Routes (`/api/test-results`, auth only):
- `POST /` multipart upload (`s3Upload('file')` → S3 `test-results/` folder); name defaults to filename sans extension.
- `GET /` own active results with search (name/filename regex); presigned URLs.
- `PUT /:id` replace file and/or rename; `DELETE /:id` soft delete (`isActive: false`). All scoped `{facilityId, cName: caller}` — strictly self-service; `createdBy/updatedBy` audit fields.

---

## 3. Referrals (discharge orders + physician e-signature)

**Files:** `src/models/Referral.model.ts`, `src/routes/referral.routes.ts` (`/api/referrals`), `src/controllers/referral.controller.ts`, `src/validation/referral.schema.ts`, `src/services/referral.pdf.service.ts`, `referral.sign.service.ts`, `referral.notification.service.ts`, `src/utils/referralPdfTemplate.ts`.

### Purpose
Home-health discharge referral workflow: a staff member drafts a discharge-orders referral for a resident, selects a Home Health Agency (HHA), the system generates an unsigned discharge-orders PDF, the resident's assigned **doctor** reviews and signs it in-app, producing a signed PDF with an immutable signature audit record.

### Data entity
`Referral`: facilityId, residentId (ref Resident), orderDate, `dischargeTo` (free-text destination, required), `dischargeDate` (required), `homeServices` (free-shape map; PDF reads skilledNursing, woundCareRequired/Notes, physicalTherapy, occupationalTherapy, medicalSocialWorker, speechTherapy, homeHealthAide booleans), `additionalOrders` (pcpFollowUp, dmeRequired/dmeType, labWork/labWorkDetails), `physicianCertification` {signature S3 key, dateSigned, physicianNamePrint, licenseNumber}, `assignedPhysician` (ref Staff, indexed — the specific physician routed for sign-off, `Referral.model.ts:37,73`), `selectedHHA` (ref Agency), `unsignedPdfUrl`, `signedPdfUrl`, `signatureAudit` {doctorCName, signedAt, ipAddress, userAgent} (immutable, written once at signing).

### Status machine
`ReferralStatus = Draft | In Doctor Review | Doctor Approved | Sent` — now a **typed enum on the model** (`Referral.model.ts:23,88-90`, default `In Doctor Review`). `Sent` was previously a query-schema-only value that the model rejected; it is now a real status (set when the referral PDF is emailed to agencies — see Flow 6).
- Create with `isDraft: true` → `Draft`, **no notification, no PDF**.
- Create without draft flag → `In Doctor Review` + fire-and-forget: (a) push notifications, (b) unsigned PDF generation.
- Sign → `Doctor Approved` (set inside `signReferralPdf`).
- `updateReferral` allows arbitrary `status` writes from the enum (no transition guard).

### Flows
1. **Create** (`POST /`, auth only — no role middleware): validates body (Zod), creates referral (incl. optional `assignedPhysician`). If non-draft: `notifyOnReferralCreate(facilityId, residentId, referralId, assignedPhysician)` pushes FCM to the named `assignedPhysician` (resolved Staff cName), falling back to the resident's `assignedStaff` when `assignedPhysician` is unset (`referral.notification.service.ts:44-58`) ("A referral for {resident} has been submitted and is awaiting your review."), the resident ("Your referral has been submitted…"), and linked family members. In parallel `generateAndStoreReferralPdf` renders the discharge-orders HTML (facility display name derived from facilityId, resident name/DOB, attending physician name, home services checkboxes, additional orders, agency name) → Puppeteer PDF → S3 `referrals/unsigned/{id}-{ts}.pdf` → persists CloudFront URL in `unsignedPdfUrl`.
2. **List** (`GET /`): facility-wide, paginated, filter by residentId/status, search across resident first/last name, **assigned-doctor name**, and agency name (multi-collection regex search then `$in`); response decorates resident with signed profile picture URL and resolved doctor name/cName.
3. **Doctor queue** (`GET /my`): "referrals routed to the authenticated doctor" — scope is now driven by `assignedPhysician` (the caller's cName resolved to a Staff `_id`, then `filter.assignedPhysician = staffDoc._id`, `referral.controller.ts:300-313`), not the old `Resident.doctor === cName` care-team mapping; same search/status filters. Empty list if the caller has no matching Staff doc. This is the doctor-app review inbox.
4. **Sign** (`POST /:id/sign`): the signing flow is **in-house e-signature**, not a third-party e-sign vendor:
   - Client uploads the physician's signature image to S3 beforehand (presigned-URL flow) and passes `signatureKey`, `physicianNamePrint`, `licenseNumber`, `dateSigned`.
   - Server captures `ipAddress` (X-Forwarded-For aware) and `userAgent` from the HTTP request.
   - `signReferralPdf` (`referral.sign.service.ts`): downloads signature image from S3 → base64 data URL (so Puppeteer renders offline) → re-renders the full discharge HTML **with embedded signature block** → new PDF at `referrals/signed/{id}-{ts}.pdf` → updates `signedPdfUrl`, `physicianCertification.*`, `status: 'Doctor Approved'`, and `signatureAudit` (doctorCName = authenticated caller). The unsigned PDF is never modified ("immutable blank form").
   - No role check that the signer is actually a Doctor / the resident's doctor — any authenticated user hitting the endpoint becomes the audited signer.
5. **Update / Delete**: partial `$set` update; hard delete (`findOneAndDelete`).
6. **Email to agencies** (`POST /send-referrals-emails`, auth only): body `{ referralId, agencyIds[] }`. Resolves the referral + the selected agencies (facility-scoped), fetches the signed (or unsigned) referral PDF, **password-protects it** via `@cantoo/pdf-lib` (`pdfDoc.encrypt`) with a derived password = first initial of resident first name (upper) + birth year `YYYY` (`referral.controller.ts:519-524`), uploads a protected verification copy to S3 (`protectedPdfKey`), and sends it as an SES email attachment to every agency email (`SendRawEmailCommand` via `src/services/email.service.ts` / `src/config/ses.ts`). On success sets `status: 'Sent'` (`referral.controller.ts:620`). Missing firstName/birthDate → warns and sends without password protection.

---

## 4. Care coordination

### 4a. Care (ad-hoc care/therapy requests) — `Care.model.ts`, `/api/care`
A lightweight bookable "care appointment" used by the **assisted-living** product surface:
- Types: `PHYSICAL_THERAPY | COGNITIVE_EVALUATION | REHAB_EVALUATION | OUTSIDE_AGENCY` (`Care.model.ts:12`). OUTSIDE_AGENCY requires `agencyName` and `serviceType`.
- Status machine: `REQUESTED → CONFIRMED → COMPLETED | CANCELLED` (default REQUESTED). Upcoming = REQUESTED/CONFIRMED; history = COMPLETED/CANCELLED. Auto-completion cron flips overdue CONFIRMED entries (index at `Care.model.ts:137` is built for this).
- Fields: cName (resident), date + startTime/endTime (24h strings), location (default "Therapy Room"), reason, post-completion `summary`, staff `notes`, `createdByType/createdByCName` (booking context), Google Calendar sync fields.
- **Booking**: `POST /` behind `resolveBookingContext('Care')` — residents self-book, family if policy allows, staff/admin for a resident. Slot validation (`assertCareSlotAvailable`, `careAvailability.service.ts`) for PT/cognitive/outside-agency types; availability endpoint supports single-day and multi-day (`noOfDays`) slot grids.
- **Google Calendar**: PT and COGNITIVE_EVALUATION care entries sync to the facility's **Physical Therapist's** Google Calendar (`CARE_TYPES_SYNCED_TO_CALENDAR`, `careCalendar.ts`); create/update/cancel/delete propagate; `calendarSyncStatus` PENDING/SYNCED/FAILED.
- Staff panel reads: `GET /staff/upcoming`, `/staff/history` (requireAnyRole ADMIN|STAFF), facility-wide.
- **Security observations:** `GET /:id` and `DELETE /:id` are intentionally commented "Open — no auth required" (`care.routes.ts:75,87`) — an unauthenticated caller with a facility header can read or hard-delete any care record. An exported `cancelCare` handler exists but is not routed (dead code, `care.controller.ts:622`).

### 4b. Care Conference (IDT meeting w/ Zoom + recordings + AI transcripts) — `CareConference.model.ts`, `/api/care-conference`
Care-team/family conference scheduling with three delivery modes and a review pipeline.

- **Participants**: host staff (`staffCName`, also the Zoom host via their linked OAuth Zoom account), `residentCNames[]`, `familyMemberCNames[]`, `careTeamCNames[]` (staff). Populated by cName (not _id) joins.
- **Status machine** (`constants/careConference.ts`): `SCHEDULED → IN_PROGRESS → IN_REVIEW → COMPLETED`, plus `CANCELLED`.
  - `SCHEDULED` on create.
  - Cron `careConferenceEnable.cron.ts` (every minute) sets `isEnabled: true` + `IN_PROGRESS` within 5 minutes of start time and pushes "starting" notifications to residents + host + care team.
  - Attaching audio recordings (in-person only) sets `IN_REVIEW` (`careConference.service.ts:775`).
  - `POST /:id/complete` or summary review (`PUT /:id/update-summary`, allowed from IN_REVIEW/COMPLETED) sets `COMPLETED`; summary update stamps `summaryUpdatedBy/At` and back-fills every processed recording with `updatedSummary/updatedBy/updatedAt`.
  - DELETE = soft cancel to `CANCELLED` (only from SCHEDULED), removes calendar events, sends cancellation notifications.
- **Virtual meetings**: `where === 'Virtual'` → Zoom meeting provisioned **before** DB write (Zoom failure aborts creation, 502); stores joinUrl/startUrl/zoomMeetingId; scheduling-field updates PATCH the Zoom meeting. Host gets `startUrl`; everyone else `joinUrl` (`shapeForCaller`).
- **In-person meetings**: mobile uploads audio via presigned S3 PUT (`POST /audio-presign`, multiple filenames; keys can be attached atomically), `PATCH /:id/recording` persists keys; a **transcribe-processor Lambda** (external) populates `recordings[]` with per-part transcript + AI summary; `transcriptText` may also be fetched from Zoom after `recording.completed`. `GET /:id/recording-urls` returns 1-hour presigned GETs.
- **Sharing with resident**: `shareWithResident` boolean controls whether the reviewed summary is exposed to the resident.
- **Calendar**: Google Calendar events created for host + every care-team member (per-staff event IDs in `careTeamGoogleEventIds`), with Zoom link in description; `calendarSyncStatus` aggregates.
- **Permissions**: all management routes require STAFF|ADMIN. Staff (non-admin) listing is scoped to conferences they host (`staffCName` filter in `getCareConferences`); admins see all. Residents/family use `GET /my-conferences` (+ `/history`) — any authenticated caller, scoped by participant `$or` filter; family member identity uses original `familyMemberCName`.
- **Notifications** (`careConference.notification.service.ts`, 505 lines): scheduled / starting / completed / cancelled pushes to residents, family, and the named staff, gated per-facility by `notificationConfig` immediate-enable flags.
- Mirrored into UnifiedSchedule per resident.

### 4c. IDT Report (Interdisciplinary Team report) — `IDTReport.model.ts`, `/api/reports` (reports.routes.ts)
A structured clinical snapshot for a resident compiled by IDT staff (case manager + rehab + social worker):
- Sections: `basicInformation` (resident ref, attending MD ref, DOB, room, admission date, contacts), `medicalOverview` (codeStatus, weight, allergies, diet), `clinicalDetails` (changeOfCondition + upcoming RehabAppointment refs), `therapyDetails` (PT bedMobility/transfers/gait/device, OT, speech), `additionalNotes` (skinIssues), `dischargePlanning` (notes, destination, DME, caregiver), `medications: string[]`, role fields `caseManager`/`rehabMembers`/`socialWorker` (staff cNames), `doctor` ref, `isAgreed`.
- **Status machine**: `DRAFT | PENDING | SUBMITTED` (default PENDING). On create/update: explicit DRAFT honored; otherwise **auto-promote to SUBMITTED when all three role fields are filled or `isAgreed === true`** (`IDTReport.controller.ts:144-151`). Full Zod validation only enforced at SUBMITTED; drafts skip validation (`stripEmpty` removes empty strings to avoid cast errors).
- **Role auto-fill**: the submitting staff's designation determines which role field they populate (Case Manager → caseManager; Physical/Speech Therapist, Director of Rehab, Rehabilitation Specialist → rehabMembers; Social Worker → socialWorker) — frontend-supplied values for these fields are stripped (`IDTReport.controller.ts:115,129-140`). So the report is collaboratively completed by three disciplines, and submission notification fires once the trio is complete.
- **`POST /addreport` doubles as upsert**: passing `_id` updates an existing report (used for progressive multi-discipline filling).
- **Listing**: `GET /getreport` with `status=PENDING` (includes DRAFT) vs `HISTORY` (SUBMITTED) + date range. `GET /getreport/resident` is the staff "my residents" view: scope = residents assigned to the caller via designation mapping (`lib/myResidents.ts` — case manager/social worker/doctor), with search, doctor filter, and a `type=reports|care-conference|both` switch that co-returns the residents' care conferences. Also returns a `doctorNamesFilter` list (all staff with designation Doctor).
- **PDF**: two endpoints now exist (`reports.routes.ts:29-30`). `POST /getreport/:id/pdf` (`generateIDTReportPdf`) renders the full report via `idtReport.pdf.service.ts` → Puppeteer → uploads to S3, persists the CloudFront URL on `IDTReport.pdfUrl`, and returns `{ pdfUrl }`. `GET /getreport/:id/pdf` (`downloadIDTReportPdf`) streams the PDF for direct download. New model field `IDTReport.pdfUrl` (`IDTReport.model.ts:71`).
- **Notifications** (`idtReport.notification.service.ts`): created → named role staff (per-role message wording) + all SERVICES-permission staff + all REHAB-permission staff; submitted → SERVICES, DASHBOARD, REHAB permission groups; reminder API targets staff with pending sections. All gated by per-facility notificationConfig (`HEALTH_CARE` category: `CREATE_IDT_REPORT`, `IDT_REPORT_SUBMISSION`, `IDT_REPORT_REMINDER`). FCM with notification history persistence (`scheduleType: 'HEALTH_CARE'`).
- **Permissions observation**: `/api/reports` is mounted with bare `authMiddleware` only — **no STAFF/ADMIN role gate** on IDT create/update/delete. The previously-duplicated mount has been **resolved on staging** — `/api/reports` is now mounted exactly once (`app.ts:209`; tech-debt T-3 closed).

### 4d. Advance Care Directive — `AdvanceCareDirective.model.ts`, `/api/advance-care-directives`
Document locker for directives (living will/POLST-type files): per-cName uploads (pdf/image) to S3 `advance-care-directives/`, optional title (defaults to filename), soft delete via `isActive`, createdBy/updatedBy audit. Staging adds a **doctor e-signature workflow** on top, with new signature fields on the model: `signedBy` (doctor cName), `signedAt`, `signedPdf` (S3 key of the signed copy) (`AdvanceCareDirective.model.ts:16-18,35-37`).
- **Self-service CRUD** (`POST /`, `GET /`, `GET /:id`, `PUT /:id`, `DELETE /:id`) scoped `{facilityId, cName: caller}` (same pattern as TestResult).
- **Staff-added directive** (`POST /staff`, new): a staff member uploads a directive on behalf of a resident. Guarded by `isStaffOnlyRequest(req)` → 403 for non-staff (`advanceCareDirective.controller.ts:99-120`).
- **Doctor pending-signatures queue** (`GET /doctor/pending-signatures`, new): lists directives awaiting the doctor's signature.
- **Doctor sign** (`PUT /:id/sign`, new): doctor uploads their signature image (`s3Upload('file')`); server renders/stores the signed PDF and stamps `signedBy`/`signedAt`/`signedPdf`.
- `GET /resident/:residentId` — staff view of a resident's directives with an explicit guard: **only staff-only identities or the resident themself** may view (`isStaffOnlyRequest(req) || req.user.username === cName`) — notably stricter than the labs/medications staff reads.

---

## 5. Rehab / Therapy

**Files:** `RehabTherapy.model.ts`, `RehabAppointment.model.ts`, `RehabMessage.model.ts`; routes `/api/rehab` (therapy, appointments, rehab-message); services `rehabTherapy.service.ts`, `rehabAppointment.service.ts` (975 ln), `rehabAvailability.service.ts`, `rehabMessage.service.ts`, `rehabAppointment.notification.service.ts`; constants `src/constants/rehab.ts`.

### 5a. Therapy catalog (RehabTherapy)
Facility-scoped master list of therapies: name, **uppercase-normalized code unique per facility via partial index (soft-deleted rows excluded so codes can be reused)**, duration (minutes — drives slot length), description, image (S3 key, signed on read), `staffCName` creator. CRUD restricted to STAFF|ADMIN; list/get open to any authed role. Soft delete (`isDeleted`/`deletedAt`).

### 5b. Rehab appointments
- **Therapy typing — the product-split pivot** (`rehab.ts:40-101`, `RehabAppointment.model.ts:19-26`):
  - `PHYSICAL_THERAPY | COGNITIVE_EVALUATION | REHAB_EVALUATION | OUTSIDE_AGENCY` = fixed types used **when the appointment belongs to an ASSISTED_LIVING facility**; slot durations come from `Config.rehab.*` (legacy key spellings preserved); OUTSIDE_AGENCY reuses PT duration and carries `agencyName` + `serviceType` (PT or cognitive).
  - `OTHER` = **SKILLED_NURSING**: concrete therapy resolved dynamically via `therapyId` → RehabTherapy (duration from the catalog row). `isRehabTherapyIdRequired` mandates `therapyId` when therapyType is `OTHER` or omitted.
- **Status machine**: `SCHEDULED → COMPLETED | CANCELLED` (default SCHEDULED; missing status on old records treated as SCHEDULED; `cancelledAt` timestamp). Upcoming endpoints = SCHEDULED; history = COMPLETED/CANCELLED. Auto-completed by cron.
- **Staff assignment rule**: `staffCName` must belong to a staff whose designation ∈ `REHAB_STAFF_DESIGNATIONS` (`assertRehabStaffAssignable`, `rehabAppointment.service.ts:256`).
- **Slot engine** (`rehabAvailability.service.ts`): available slots for a staff member on a day = base grid (from Config) minus (a) facility meal windows (`Config.meals`), (b) the staff's existing bookings, (c) the staff's **cached Google Calendar busy slots**, (d) the resident's blocking UnifiedSchedule entries (cross-venue: SALON/MASSAGE/PT/CARE/REHAB). Booking re-validates the exact requested slot server-side and throws CONFLICT otherwise.
- **Routes/permissions**: create + list + history + by-resident + available-slots require STAFF|ADMIN (create additionally passes `resolveBookingContext('RehabAppointment')`, recording createdBy*); `my-appointments` (+history) are resident/family-facing via booking context; get/update/delete by id require auth only. Search spans resident name/email/unit and location.
- **Notifications**: created/cancelled/completed FCM to resident + family + assigned staff, gated by notificationConfig `HEALTH_CARE`/`CREATE_REHAB_APPOINTMENTS` etc. (`rehabAppointment.notification.service.ts`).

### 5c. Rehab messages (resident → rehab dept threads)
A simple triaged inbox, separate from chat:
- Resident-only create (`POST /rehab-message`, requireAnyRole RESIDENT): `topic` + `message`, both **KMS envelope-encrypted at rest** (AES-256-GCM, per-field wrapped data key, encryption context binds facilityId+cName+model+field to prevent ciphertext swapping — `rehabMessage.service.ts:55-66`). Plaintext never persisted; decrypted server-side on every read.
- Resident reads own (`GET /rehab-message/my-message`); staff/admin read all for facility (`GET /rehab-message`).
- **Status machine**: `NEW → IN_PROGRESS → CLOSED` (`rehab.ts:134`). Staff/admin `PATCH /:id` updates status; `replyBy` (cName) + `replyByRole` (STAFF|ADMIN) track who actioned it. No threaded replies — it is a status-tracked request queue, with the responder identity surfaced (populated staff/admin profile).

---

## 6. Agencies & Case-Manager schedule

### 6a. Agency (`Agency.model.ts`, `/api/agencies`)
Home Health Agency directory per facility: name, email, countryCode+phone, address (street/city/state/zip), `specialties[]`, status `Active|Inactive` (default Active). Plain CRUD behind auth only (no role gate); list supports search + status filter; **hard delete**; no duplicate-name guard. Consumed by Referrals (`selectedHHA`) — referral PDFs print the agency name; doctor referral search matches agency names.

### 6b. Case-manager schedule (`/api/case-manager`, `caseManagerSchedule.controller.ts` — 1,217 ln)
A unified day-schedule aggregator with **two persona variants** decided by the caller's designation (`controller:602-607`):
- **Case manager (default)**: aggregates, for a given date, all `SalonAppointment`s, `TransportationRequest`s and `RehabAppointment`s **created by the caller** (`createdByCName: cName`) plus `CareConference`s where the caller is host or care team — i.e. "what I booked / must attend today". Excludes cancelled/rejected.
- **Doctor**: requires `residentCName` query param; aggregates that resident's salon/transport/rehab/care-conference items from the day forward — i.e. a per-patient itinerary view.
Entries are normalized into typed cards (SALON/TRANSPORTATION/CARE_CONFERENCE/REHAB) with resident profile, createdBy resolution (resident vs staff creator), and chronologically sorted with day-name header.
- `GET /schedule-pdf` renders the same data as printable appointment cards (`buildUpcomingAppointmentsPdf` → Puppeteer).

---

## 7. Messaging / Chat

**Files:** `Conversation.model.ts`, `Message.model.ts`; `/api/chat` (`chat.routes.ts`); socket `src/socket/chat.handler.ts` (namespace `/chat`); constants `src/constants/chat.ts`.

**Module decomposition (staging).** The old monolithic `controllers/chat.controller.ts` (~970 LOC) and flat `services/chat/*.service.ts` files have been **split into packages**: controllers now live under `src/controllers/chat/` (`conversation`, `conversationSearch`, `group`, `mention`, `message`, `reaction`, `search`, `careTeam` controllers + `_helpers.ts` + `index.ts`), and services under `src/services/chat/` with `conversation/` (conversation, inbox, info, attachments, unread, careTeam) and `message/` (send, list, status, delete, reaction, system) sub-packages, plus top-level `attachment`, `mention`, `group`, `search`, `conversationSearch`, `chatEncryption`, `chatNotification`, `chatKeyCache`, `chatAccessPolicy`, `participant.repository` services. The chat route file imports `COGNITO_USER_ROLES` from the typo dir `../contants/cognnito.types` (Observation 12 — still present and actively imported).

### Who chats with whom (`chatAccessPolicy.ts` — the single source of truth)
Gating applies to **conversation initiation only**; existing participants keep access even if policy later tightens (intentional — no retroactive revocation; enforced going forward via search visibility only).
- Chat roles: `RESIDENT | STAFF | ADMIN` (family members chat **as the resident** due to username normalization; there is no distinct FAMILY chat role on documents, though FAMILY_MEMBER may call the HTTP routes).
- Gate 1: self-chat denied. Gate 2: **RESIDENT ↔ RESIDENT always denied**. Gate 3: per-facility config flags `chat.isResidentAllowed`, `chat.isAdminAllowed` (defaults permissive; 30 s config cache).
- RESIDENT → STAFF: target staff must be in the resident's **care team** (target cName ∈ `Resident.assignedStaff[]` — the legacy per-role fields are gone; `chatAccessPolicy.ts:305-320`) AND target designation in `chat.staffDesignationAllowed` (empty list / `"*"` = unrestricted).
- STAFF → RESIDENT: initiator designation must pass the allowlist; if the designation maps to a care-team field, the resident must have **this** staff in their `assignedStaff[]`; unmapped designations may reach any active resident.
- STAFF → STAFF: both designations must pass the allowlist.
- ADMIN ↔ anyone: allowed when `isAdminAllowed`, both parties active.
- `GET /chat/care-team-contacts` (RESIDENT only) returns the resident's chatable care-team directory.

### Conversations & messages
- Types `DIRECT` / `GROUP`. DIRECT dedup via sparse-unique `directPairKey` (sorted cName hash per facility) to prevent duplicate threads on concurrent first sends.
- **Groups are staff/admin-only constructs** (create/update/delete/membership/admin routes requireAnyRole STAFF|ADMIN|SUPER_ADMIN); group meta = name, picture, createdBy, admins[]; invariants: at least one admin; sole admin can't leave without promoting another. Group membership changes emit **SYSTEM messages** (GROUP_CREATED, MEMBER_ADDED/REMOVED/LEFT, NAME/PICTURE_CHANGED, ADMIN_PROMOTED/DEMOTED) rendered as centered activity rows.
- **Encryption**: per-conversation KMS-wrapped data key (`dataKey.wrappedKeyB64`, keyVersion); message text and inbox preview stored AES-256-GCM encrypted; mentions, reactions, and system events deliberately stored **unencrypted** to allow notification fan-out and badge queries without KMS round-trips; 5-min key cache.
- **Attachments**: image/video/audio/document; per-message caps (5 images, 2 videos, 10 total) and per-type size caps (defaults image 5 MB, video 30 MB via config defaults; per-facility attachment caps), MIME allowlists; video poster thumbnails paired by index; counts denormalized on the conversation (`mediaAttachmentCount`/`fileAttachmentCount`) for O(1) info panels; S3 objects deleted eagerly on message soft-delete. **S3-reference attachments** (`Message.isReference`/`sourceModel`/`sourceModelAttachmentId`, `Message.model.ts:48-62`): an attachment that references a document owned by another model (e.g. a report) is **never S3-deleted by chat**, so chat deletion cannot orphan another module's file.
- **Soft membership (groups)**: leaving a group adds the member to `Conversation.exitedMembers[]` rather than removing them; ex-members keep read access to history and are served **frozen** attachment counts (`frozenMediaCount`/`frozenFileCount` snapshotted at exit, `Conversation.model.ts:135-138,263-264`).
- **Statuses**: `SENT → DELIVERED → READ`, aggregate on the message plus per-member `deliveredTo[]`/`readBy[]` (groups flip aggregate only when all non-senders ack); per-user `unreadCounts` map on conversation; bulk mark-read per conversation.
- **Mentions** (`@`): GROUP-only, sentinel-wrapped tokens (STX/ETX control chars) in text, server-validated against participants, max 20/message; denormalized onto `lastMessage.mentions` for inbox badges.
- **Reactions**: single emoji per user per message; ADD/REMOVE(toggle)/REPLACE semantics.
- **Real-time** (Socket.io `/chat`): events `chat:new`, `chat:delivered`, `chat:read`, `chat:status`, `chat:group`, `chat:deleted`, `chat:reaction`, `chat:error`, and `chat:unread` (total unread pushed on connect). Push notifications via `chatNotification.service.ts` (FCM).
- **Soft message delete**: sender-only delete sets a `Message.deletedAt` tombstone (`Message.model.ts`) rather than removing the row; deleted messages render as a placeholder and drop their (non-reference) S3 objects. Indexed via `{conversationId, deletedAt}`.
- **Search & info endpoints** (staging additions): `GET /chat/conversations/search` (search the caller's conversations), `GET /chat/conversations/:id/info` (conversation info panel — members, denormalized counts), `GET /chat/conversations/:id/attachments` (paged media/file gallery), `GET /chat/mention-residents` (mentionable resident directory, server-scoped: STAFF candidates gated by `assignedStaff[]`), `GET /chat/care-team-contacts` (RESIDENT-only, the resident's chatable care-team directory). The legacy `GET /chat/search` user directory remains, respecting the same access policy.
- **Routing/RBAC**: `chat.routes.ts` now applies `router.use(authMiddleware, facilityMiddleware)` up front (`chat.routes.ts:83`) plus per-route `requireAnyRole(...)` — group management is STAFF|ADMIN|SUPER_ADMIN, `care-team-contacts` is RESIDENT-only, the rest allow any authed role.

### Message pinning (staging, 2026-08-27) — new `ConversationPinState` model

A conversation-wide, shared pinned-messages tray — distinct from the pre-existing per-user *conversation* pin (`ConversationMemberState.pinnedAt`, socket event `chat:pin-changed`) — backed by a new collection `ConversationPinState.model.ts` (one document **per conversation**, not per pin/user; unique index on `conversationId`): `pinnedMessages: [{messageId, pinnedBy (cName), pinnedByRole, pinnedAt, expiresAt?}]` (`ConversationPinState.model.ts:24-59`). Absent `expiresAt` = pins forever.

- **Routes** (`chat.routes.ts:380-423`): `PUT`/`DELETE /chat/messages/:messageId/pin`, `GET /chat/conversations/:id/pinned-messages` — all sit behind the router's existing `authMiddleware, facilityMiddleware` + `requireAnyRole(anyAuthedRole)` gate (no new auth surface). Listing requires **ACTIVE** participation only — the one pin/message endpoint in this module that does **not** carry history's ex-member read exception (`pin.service.ts:listPinnedMessagesForConversation` throws `FORBIDDEN` for a non-ACTIVE `membership.status`).
- **Dual-cap enforcement, atomic, no transaction** (`pin.service.ts:pinMessageForConversation`): step 1 idempotently upserts the `ConversationPinState` row (`$setOnInsert`, E11000 from a losing concurrent first-pin is swallowed); step 2 is a single `updateOne` whose `$expr` checks **both** `Config.chat.maxPinnedMessagesPerConversation` (default 5) and `maxPinnedMessagesPerUser` (default 5) against that same document's array in one write — no read-then-write race between the two caps. `modifiedCount === 0` is disambiguated into already-pinned / per-user-cap-hit / conversation-cap-hit; the two cap failures throw `413 PAYLOAD_TOO_LARGE`, with copy the code comments say is kept byte-identical to the admin app's own pre-check string (`pinLimitMessage` in the admin app's `domain/pinDomain.ts`).
- **Expiry**: `durationMinutes` is validated against `Config.chat.pinMessageDurationOptions` (default `[60, 420, 1440, 10080, -1]` minutes; `-1` = `PIN_DURATION_FOREVER` sentinel, `constants/chat.ts`). An explicitly-saved empty `pinMessageDurationOptions` array is rejected at the config-schema validator (`config.model.ts`) — `[].every(...)` is vacuously true and would otherwise silently make every pin request fail. **No TTL index** (Mongo TTL only expires whole documents, never array elements) — cleanup is primarily demand-driven: unpin and the tray-list read both `$pull` expired siblings as a side effect of their own write/read, backstopped by a **new daily cron** `src/jobs/pinExpiry.cron.ts` (`0 3 * * *`, gated by new env `ENABLE_CHAT_PIN_EXPIRY_CRON`, registered independently of `ENABLE_NOTIFICATION_CRON`).
- **Unpin race safety**: `unpinMessageForConversation` uses `findOneAndUpdate({new: false})` to read the pre-image atomically with its own `$pull`, so two concurrent unpins of the same message can't both observe "was pinned" and both fire a system message.
- **System messages & new socket event**: manual pin/unpin transitions insert a SYSTEM message (`MESSAGE_PINNED`/`MESSAGE_UNPINNED`, `constants/chat.ts` `SYSTEM_EVENT_TYPE`) — never for expiry sweeps or delete-triggered auto-unpin. Every pin/unpin/expiry/delete-triggered-unpin fans out over sockets via **new event `chat:message-pin-changed`** (`CHAT_SOCKET_EVENT.MESSAGE_PIN_CHANGED`, emitted by `chatNotification.service.ts:notifyMessagePinChanged`) to every conversation participant, socket-only (no FCM/web push) — distinct from the existing per-viewer `chat:pin-changed`.
- **Auto-unpin on delete**: `delete.service.ts:softDeleteMessageRecord` now also `$pull`s the just-deleted message out of `ConversationPinState` and returns `unpinned`/`totalPinnedCount` to the caller, which emits the pin-changed event with `reason: 'MESSAGE_DELETED'` — consistent with the fully-non-destructive chat retention posture (content stays, but a deleted message can't remain in the shared tray).
- **Read-path shaping**: `listPinnedMessagesForConversation` resolves `pinnedBy`/sender display names and `@mentions` (mirrors `listMessages`) rather than leaking raw cNames, respects each viewer's own "clear chat" floor (`memberState.clearedThroughMessageId`) independently of the shared pin state, and sweeps + reports expired entries (`sweptMessageIds`) so the controller can emit one `EXPIRED`-reason event per swept pin.
- Full spec: `docs/chat/feature/pin-message.md` (new, 476 lines) — the architecture reference for this feature plus its admin-app frontend implementation.

---

## 7a. Fax (Documo) — NEW

**Files:** `src/integrations/documo/documo.service.ts`, `documoConfig.ts`; `src/controllers/fax.controller.ts`; `src/routes/fax.routes.ts` (`/api/fax`); `src/validation/fax.schema.ts`.

A new outbound fax capability backed by the **Documo** HTTP API (base `https://api.documo.com`, Basic auth, env-gated on `DOCUMO_API_KEY`). The service exposes `sendFax`, `getAccountDetails`, `getFaxHistory` (`documo.service.ts:198,251,332`); the controller wraps them as `sendFaxController` / `getAccountDetailsController` / `getFaxHistoryController`.

- `POST /api/fax/send` — submit a fax (recipient number + attachments).
- `GET /api/fax/account` — Documo account details.
- `POST /api/fax/history` — fax history query.

**Mounting / security (`app.ts:103`, `fax.routes.ts:14-29`).** `/api/fax` mounts **before** the global `facilityMiddleware` (`app.ts:104`). Its per-route guards are `[facilityMiddleware, authMiddleware]` — but `FAX_LOCAL_BYPASS=true` replaces them with an **empty guard array**, leaving `/api/fax/*` completely unauthenticated and un-tenant-scoped (logged with a warning at boot). There is **no `NODE_ENV` production fail-closed** on the bypass (Observation 19 / Design-gap G-23, High). No send retry / idempotency yet (G-24); per-facility credential isolation on account/history endpoints to be verified before GA.

---

## 8. Product-split signals (Assisted Living vs Skilled Nursing)

The codebase is one backend serving multiple facility care levels; the SNF flavor is explicit:

1. **Care-level enum**: `CARE_TYPE_VALUES = ['assisted_living','memory_care','independent_living','skilled_nursing']` (`src/contants/index.ts:1`) — stored per **Resident** (`resident.model.ts:9,63,113-115`, required in `resident.schema.ts:40`), filterable in resident lists. There is no facility-level type field — care level is resident-granular.
2. **Rehab therapy typing** is the clearest split: fixed therapy types "are used when the appointment belongs to an ASSISTED_LIVING facility"; `OTHER` + dynamic `RehabTherapy` catalog is the **SKILLED_NURSING** path (`RehabAppointment.model.ts:19-26`, `rehab.ts:40-67`). SN gets a configurable therapy catalog + therapist-resource scheduling; AL gets fixed service types with Config-driven durations.
3. **SNF-flavored modules**: IDT reports (interdisciplinary team is an SNF/CMS construct), discharge referrals to Home Health Agencies with physician certification and SES-emailed password-protected PDFs, Documo fax, PCC (PointClickCare — SNF/LTC EHR) medication + **practitioner** sync (§1a) and patient webhooks, lab-report ingestion, case-manager/doctor schedule views. The PCC webhook registry grew **12 → 14 handlers** on staging (adding `patient.admit`/`patient.updateContactInfo`), and **14 → 16 handlers** this pass (adding `medicalProfessional.add`/`medicalProfessional.remove`, §1a) — the first PCC event pair to write `Resident.assignedStaff` directly, rather than only demographic/medication fields (`src/integrations/pms/pcc/webhooks/registry.ts`). New `familyMember` fields `phone`, `pcc_contactId` (indexed), `pcc_is_guarantor`, with a unique-sparse `{residentId, countryCode, phone}` index. These are consumed by the staff/clinical app surface; resident-facing surfaces (mobile/TV) consume "my medications", "my lab reports", "my conferences", rehab self-views.
4. The `referralPdfTemplate`/`medicationPdfTemplate` "skilled" hits are the `skilledNursing` home-service checkbox and PCC schedule strings — consistent with discharge-to-home-health workflows.

---

## 9. Notifications & real-time summary (clinical scope)

| Trigger | Channel | Recipients | Gate |
|---|---|---|---|
| Referral created (non-draft) | FCM + history | `assignedPhysician` (fallback resident `assignedStaff`), resident, family | `notifyOnReferralCreate` |
| Referral emailed to agencies | AWS SES (password-protected PDF attachment) | Selected agencies | `POST /send-referrals-emails` |
| PCC `patient.admit` | FCM + Twilio SMS | Active managers | webhook handler |
| PCC `medicalProfessional.add` ambiguous name match (§1a) | AWS SES email | 3 hardcoded internal addresses (`pccAlerts.ts`) — **not resident/family/facility-staff-facing** | `sendDuplicateStaffAlert` |
| IDT created / submitted / reminder | FCM + history | Named role staff; SERVICES / DASHBOARD / REHAB permission groups | `notificationConfig` HEALTH_CARE flags |
| Care conference scheduled / starting (cron, T-5 min) / completed / cancelled | FCM + history | Residents, family, host + care team | notificationConfig |
| Rehab appointment created / cancelled / completed | FCM + history | Resident, family, assigned rehab staff | notificationConfig HEALTH_CARE |
| Chat message / reaction / group events / read receipts | Socket.io `/chat` + FCM | Conversation participants | access policy |
| Appointment auto-completion | (silent) cron */15 min | — | env flags |
| Care conference enable | cron * * * * * | — | `ENABLE_CARE_CONFERENCE_ENABLE_CRON` |

Integrations: Zoom (OAuth per staff; meetings + transcript fetch; CareConference now also carries `pstnPassword` + `dialInNumbers[]` for PAC phone-conference dial-in, `CareConference.model.ts:58-60,108-109`), Google Calendar (per-staff OAuth; Care→PT calendar, CareConference→host+care team, busy-slot cache feeding rehab availability), AWS KMS (**4 distinct envelope-encryption schemes**: PCC medications per-resident key, **PCC practitioners' new resource-agnostic shared key (§1a)**, RehabMessage per-field keys, Chat per-conversation key), S3 + CloudFront (presigned uploads/downloads; PDFs), Puppeteer HTML→PDF (medication list, referral, IDT report, case-manager schedule), **AWS SES** (`SendRawEmailCommand` with PDF attachments — referral agency email; ops duplicate-staff alert, §1a; `src/config/ses.ts`, `src/services/email.service.ts`), **Twilio** (transactional SMS `sendSms` — `src/services/sms.service.ts`, env-gated on `TWILIO_ACCOUNT_SID/AUTH_TOKEN/FROM_NUMBER`; AWS SNS retained for reset-OTP `sendOtpSms`), **Documo** (outbound fax, §7a), **`@cantoo/pdf-lib`** (password-protected referral PDFs), FCM, PCC (mTLS + OAuth + rate limiter; **16** webhook handlers across **two independent OAuth/token-cache modules**, `pcc.service.ts` and the new `pcc.core.ts`, §1a), CrelioHealth/Lemedix (inbound webhooks), transcribe-processor Lambda (out-of-repo, writes `recordings[]`).

---

## 10. Observations (gaps, dead code, risks)

**Security / PHI**
1. `GET /api/care/:id` and `DELETE /api/care/:id` are deliberately unauthenticated (`care.routes.ts:73-87`) — read + hard-delete of care records without a token.
2. Lemedix webhooks accept unauthenticated POSTs with no signature verification and log full payloads (PHI) via `console.log` (`lemedix.webhook.routes.ts:21,33,45`); PCC webhook auth not reviewed here but lemedix is open.
3. Medication staff endpoints (`GET /resident`, `/resident/pdf`) and lab `GET /:residentId` have **no role middleware**: any authenticated facility user (incl. a resident) can pull another resident's medication PDF / lab list by cName/id. Contrast with ACD which guards (`isStaffOnlyRequest`).
4. `/api/reports` (IDT) has no role gate (bare `authMiddleware`); `updateIDTReport` `$set`s the raw request body unvalidated except via `IDTReportUpdateSchema`. (The previous duplicate-mount is fixed — single mount at `app.ts:209`.)
5. Referral signing does not verify the signer is a Doctor or the resident's assigned doctor; audit trail mitigates but doesn't prevent.
6. Lab resident-matching by phone/email is fragile — shared family phone numbers could attach reports to the wrong resident; facility mismatch is checked, identity confidence isn't.

**Dead / half-built**
7. `cancelCare` exported but never routed (`care.controller.ts:622`).
8. `getLabReportsFromResident` has commented-out residentId logic, a stray `console.log('Resident data:', ...)`, and pagination skip commented out — page param is accepted but ignored (`labReports.controller.ts:89-120`).
9. `LabPatient` records are stored but never linked to residents or surfaced by any read API (write-only registry). `LabReport.reportFormatAndValues` (structured analytes with critical flags) stored but never exposed — an obvious future "lab trends/alerts" feature.
10. ~~Referral query schema allows status `'Sent'` which the model enum doesn't contain.~~ **Resolved** — `'Sent'` is now a first-class model enum value set by `POST /send-referrals-emails` (`Referral.model.ts:23,88-90`).
11. `Referral` create accepts `physicianCertification` from the client at draft time even though signing overwrites it (pre-fill of name/license).
12. Two constants directories: `src/contants/` (typo, holds cognito types + care types) and `src/constants/` — inconsistent imports across files.
13. `IntegrationAvailable.timzone` (typo) used for medication date formatting.
14. Hardcoded fallbacks: `America/Los_Angeles` timezone throughout (env `FACILITY_TIMEZONE` is process-wide, not per-facility); facility display names derived by title-casing facilityId; `LEMEDIX_FACILITY_ID` single-account mapping; PCC `facId` default `'12'` in patient search (`pcc.service.ts:207`).
15. `console.log/error` used everywhere (no structured logger).
16. TODOs at `appointmentCompletion.service.ts:126,209` (per-doc updates to preserve hooks — known throughput limitation).
17. IDT `getIDTReports` list endpoint does not restrict by assigned residents (facility-wide), while `getreport/resident` does scope by designation mapping — two different visibility models for the same data.
18. Medication PDF `facilityName` falls back to raw facilityId or literal `'Shashi.ai'` (`medication.controller.ts:335`).

**Staging additions**
19. `/api/fax/*` (Documo) mounts before the global facility gate (`app.ts:103`) and its facility+auth guards are bypassable via `FAX_LOCAL_BYPASS=true` with no production fail-closed (G-23, High). No fax send retry/idempotency (G-24); verify per-facility Documo credential isolation before GA.
20. `GET /api/residents/pcc-contacts` is **unauthenticated** — declared before `authMiddleware` in `residents.route.ts:64`, so any caller with a facility header can fetch PCC family-member contacts for a base64 patientId (widens the existing unauthenticated-read surface, G-3).
21. `migrateAssignedStaff.ts` (legacy care-team fields → `assignedStaff[]`) is a manual one-time script not wired into CI/CD or boot — a missed run silently empties care-team filters across chat, referrals, and notifications (T-14, Medium).
22. The chat module decomposition into `controllers/chat/` + `services/chat/{conversation,message}/` is a positive structural counter-example to the still-growing god-class controllers (resident/transport/housekeeping/staff).

**2026-08-28 additions (practitioner sync, §1a)**
23. **Duplicate, independent PCC OAuth/mTLS module (`pcc.core.ts`).** New practitioner-sync code re-implements `resolvePccConfig`/`getPccAccessToken`/`getPccHttpsAgent` from scratch instead of importing the identical, pre-existing functions in `pcc.service.ts` — two separate in-memory OAuth token caches and two separate mTLS `https.Agent`s for the same PCC client now coexist per process. Not currently harmful (each cache works independently and correctly), but a maintainability/consistency risk: a future auth-flow change (e.g. a 401-retry policy fix, a new PCC scope, credential rotation handling) applied to one file will silently not apply to the other unless someone remembers both exist. *Candidate fix:* have `pcc.core.ts` re-export from (or be merged into) `pcc.service.ts`, or vice versa. (`pcc.core.ts`; `pcc.service.ts`.)
24. **Duplicate-staff-name alert has no facility-visible surface and hardcoded personal-email recipients.** `DUPLICATE_STAFF_ALERT_EMAILS` (`src/constants/pccAlerts.ts`) is three named individuals' `pardypanda.com` addresses — not sourced from `Config`, not per-facility, not rotatable without a code deploy. When triggered, a resident can be left with **no physician auto-attached to their care team**, and nothing in the product (admin web, staff app, resident app) indicates this happened — only the three hardcoded inboxes are told. *Candidate fix:* source recipients from a per-facility (or at minimum env-configured) ops contact, and/or surface skipped assignments as an admin-visible worklist item rather than only an out-of-band email. (`pccAlerts.ts`; `practitionerStaffAssignment.ts:12-40`.)
