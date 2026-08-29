# Module: Secure Call

> Applies to: Both (platform capability) — SN-flavored; the operating surface is the staff app and the resident/family apps
> FR prefix: SCALL
> Sources: code is source of truth — `senior_living_backend@pre-production` (`models/SecureCall.model.ts`, `constants/secureCall.ts`, `controllers/secureCall.controller.ts`, `services/secureCall.service.ts`, `controllers/twilio.webhook.controller.ts`, `middleware/twilioSignature.middleware.ts`, `utils/secureCallRecordingCrypto.ts`, `routes/{secureCall,twilio.webhook}.routes.ts`).
> **2026-08-08 (v1.5) — new module.** Secure Call was introduced in the 2026-07-31→08-07 window (`dot-secure-call`, `twilio-one-time-edit`, `feature/chat-v2-pre-production` batch) on **`pre-production`** (nothing in `production`). It formalizes and extends the pre-existing "Twilio Programmable Voice (staff→resident calls)" capability into a recorded, transcribed, summarized, staff-reviewed call record. There is **no admin-web surface** — the feature is backend + mobile (the staff app places calls; the resident/family apps read shared summaries).
> **2026-08-21 (v1.6) — first admin-web surface.** A new admin **Resident Documents** tab (`senior_living_admin`, `show-document-list-in-resident-details-pre-prod`, merged 2026-08-19) lists a resident's secure calls alongside their advance care directives and lets staff **approve** the AI-generated call summary from a new **Call Summary** modal. This is Secure Call's first reachable admin-web entry point — see SCALL-FR-08. The "no admin-web surface" framing above now describes the pre-2026-08-19 state only.

> **2026-08-21 (v1.7) — confirmed absent from the SN resident app.** `senior_living_skillednursing_resident` (`origin/master` HEAD `4734daf`) has **no consumer** of SCALL-FR-06 (`GET /api/secure-calls/my-calls`) anywhere in the repo — zero references to `secure-calls` or `SecureCall`. See SCALL-O-7.
---

## 1. Purpose & scope

Secure Call lets a **staff member place a masked voice call to a resident or a linked family member** through Twilio without exposing either party's real phone number, records the call, and runs it through an AI **transcribe → summarize** pipeline. The staff member then **reviews and approves** the generated summary and may optionally **share it with the resident/family member**, who can read it in their app. The record (and its encrypted transcript/summary) is the durable artifact — the point of the feature is an auditable, summarized, PHI-safe log of staff↔resident/family phone contact, not just call connectivity.

**In scope (as-built):** the `SecureCall` lifecycle — log-before-dial, Twilio dialing via a per-staff client identity, call/recording status callbacks, recording storage, the Lambda-driven transcript/summary write-back, staff summary review + approval, resident/family read of shared summaries. Envelope encryption of transcript/summary/summary-review text.

**Out of scope (other modules):** text chat (messaging-chat.md — a separate real-time channel); care-conference recordings/transcripts/summaries (care-coordination.md — a different entity with the same KMS-recording pattern); the generic transactional/OTP SMS and Twilio-voice plumbing shared with other flows (platform-foundation.md). The actual dial-plan TwiML (`voice.js`) and the transcribe/summary Lambda live outside this repo and are referenced, not owned, here.

---

## 2. Personas & surfaces

| Persona | Surface | Role in this module |
|---|---|---|
| Staff / clinical user | Staff app | Places the call (`logSecureCall` → `POST /api/secure-calls`), reviews & approves the AI summary (`PUT /:id/update-summary`), toggles share-with-resident, sees the "awaiting my approval" and unviewed-call lists. |
| Admin | Admin web (**new, v1.6**) + API | The **Resident Documents** tab (`ResidentDetails` → `ResidentDocuments.tsx`) lists a resident's secure calls (merged with their advance care directives, `GET /advance-care-directives/admin/resident/:residentId`) and opens a **Call Summary** modal (AI Summary / Reviewed Summary / Transcript, `GET /secure-calls/:id`) with a single **Approve** action (`PUT /:id/update-summary { approvalStatus: "APPROVED" }`). This is per-resident, reached from Resident Details — there is still no facility-wide call log/queue view (SCALL-O-4, partially resolved). `GET /api/secure-calls` remains `STAFF|ADMIN`-gated for the underlying facility list. |
| Resident | Resident app | Reads the **shared** summary of a call they were on (`GET /api/secure-calls/my-calls`, `RESIDENT`-gated) — only when staff set `shareWithResident`. |
| Family member | Family app | Same as resident for calls placed to them (`FAMILY_MEMBER`-gated on `my-calls`). |
| External | Twilio webhook | Call-status and recording-status callbacks (`/webhooks/twillio`), signature-verified. |
| External | Transcribe/summary Lambda | Matches the S3 recording filename token `SAL_securecall_<id>_<part>` and writes KMS-encrypted `transcript`/`summary` per part back onto the record. |

Primary operating surface: **staff app** (call placement + review). The resident/family surface is read-only and gated on the staff share toggle.

---

## 3. Functional requirements (as-built)

- **SCALL-FR-01 — Log a call before dialing.** `POST /api/secure-calls` (`STAFF|ADMIN`) creates a **PENDING** `SecureCall` with the caller's `staffCName`/`staffId`/`cognitoSub`, the `calleeType` (`resident | family_member`), the callee cName + `calleePhone` (copied from the resident/family record for display), and `status: PENDING` / `approvalStatus: PENDING`. `callSid` is **not** known yet — it is correlated later from the recording callback by staff identity. The app calls this immediately before dialing so a record exists even if the call never connects. (`secureCall.controller.ts:createSecureCall`; `SecureCall.model.ts`)

- **SCALL-FR-02 — Masked dialing via a per-staff Twilio client identity.** The call is placed with Twilio using `From = client:<staffId>` so the staff member's real number is never exposed, and the callee sees a facility-masked number. The dial-plan TwiML (`voice.js`) sets the `statusCallback` and `recordingStatusCallback` to the webhook below. (`secureCall.service.ts`; dial-plan external)

- **SCALL-FR-03 — Signature-verified Twilio webhook.** `POST /webhooks/twillio/status` and `/recording` (plus `GET /ping`) receive Twilio callbacks. **All callbacks pass `twilioSignature` middleware**, which verifies the `X-Twilio-Signature` HMAC — **best-effort unless `TWILIO_ENFORCE_WEBHOOK_SIGNATURE=true`, which fails closed.** Only a `completed` call is persisted; only a `completed` recording is downloaded and stored. The recording callback **correlates to the PENDING record by staff identity** (`staffId` → newest PENDING call), then sets `callSid`, `recordingSid`, and the S3 `recordingKeys`. *(Contrast: the WestFax delivery webhook (referrals.md O-8) has no signature verification — Secure Call's webhook does, and is the pattern to copy.)* (`twilio.webhook.controller.ts`; `twilioSignature.middleware.ts`)

- **SCALL-FR-04 — Recording storage + encrypted transcript/summary.** Completed recordings are uploaded to S3 (`recordingKeys[]`). An **external Lambda** transcribes and summarizes each recording part, matches the filename token `SAL_securecall_<id>_<part>`, and writes back a `recordings[]` entry per part — each with an **envelope-encrypted** `transcript` and `summary` (AES-256-GCM under a KMS-wrapped key; `secureCallRecordingCrypto.ts`), plus `partNumber`/`fileName`/`processedAt`. The Lambda owns `status` and rewrites it per recording part. This mirrors the care-conference recording-encryption scheme (SN-PR-02) but is a **separate** KMS envelope path. (`SecureCall.model.ts:ISecureCallRecording`; `secureCallRecordingCrypto.ts`)

- **SCALL-FR-05 — Staff summary review & approval.** `PUT /api/secure-calls/:id/update-summary` (`STAFF|ADMIN`) lets staff submit a **reviewed final `summary`** (envelope-encrypted under the AAD label `reviewedSummary`), recording `summaryUpdatedBy`/`summaryUpdatedAt`. Approval is tracked separately from `status` (which the Lambda owns): `approvalStatus` walks `PENDING → approved` with `approvedBy`/`approvedAt`. The staff "awaiting my approval" list is `GET /api/secure-calls?approvalStatus=PENDING`. A `viewedAt` timestamp drives an "unviewed" dot for staff. (`secureCall.controller.ts:updateSecureCallSummary`; indexes on `{facilityId, approvalStatus, createdAt}`)

  **Staff-app UI confirmed (2026-08-21, `senior_living_staffapp` `origin/master` HEAD `35b3cc82`).** The review/edit/approve flow above is implemented by two screens — `CallSummaryScreen` (review + approve) and `EditCallSummaryScreen` (edit the reviewed-summary text) — backed by a dedicated client-side `callSummary` state slice holding the call's PHI (recordings, callee name, initiating staff, the editable summary draft, `shareWithResident`, and the two status fields from this FR). No client-side transcription code exists anywhere in the app — consistent with SCALL-FR-04, it only displays and approves the Lambda's output. The upstream transcription/summarization vendor remains unidentified (§6 Integrations).

- **SCALL-FR-06 — Share with resident / family.** When staff set `shareWithResident: true` (the "tick mark"), the resident or family member who was on the call can read the reviewed `summary` via `GET /api/secure-calls/my-calls` (`RESIDENT | FAMILY_MEMBER`-gated, paginated). The `my-calls` filter has **no staff branch** — a staff caller gets a silently empty list and must use `GET /` instead. Default is **not shared** (`shareWithResident: false`). (`secureCall.controller.ts:listMySecureCalls`; `SecureCall.model.ts:69`)

- **SCALL-FR-07 — Recording URL access.** `GET /api/secure-calls/:id/recording-urls` (`STAFF|ADMIN`) returns signed URLs for the stored recording(s). (`secureCall.controller.ts:getSecureCallRecordingUrls`)

- **SCALL-FR-08 — Admin Resident Documents surface: review + approve (pre-production, NEW, v1.6).** A resident's Resident Details view in the admin web gains a **Resident Documents** tab (`ResidentDocuments.tsx`) that merges secure calls and advance care directives into one sortable/filterable feed via a new backend endpoint `GET /advance-care-directives/admin/resident/:residentId` (response items typed `type: 'directive' | 'secure_call'`). A secure-call row opens the **Call Summary** modal (`CallSummaryModal.tsx`), which fetches `GET /secure-calls/:id` and renders three collapsible sections — **AI Summary** (the Lambda-generated per-recording summaries, joined), **Reviewed Summary** (the staff-submitted `summary` from SCALL-FR-05, shown only if present), **Transcript** — with a single-button **Approve** action calling `PUT /secure-calls/:id/update-summary { approvalStatus: "APPROVED" }` (the same endpoint SCALL-FR-05 defines; the admin UI only ever sends the approval flag, never a revised `summary` — editing the reviewed text remains staff-app/API-only). Once approved, the button is replaced with a static "Approved" badge (re-derived from `approvalStatus === 'APPROVED' || Boolean(approvedAt)`). "Mark as viewed" for a secure-call row goes through a new shared endpoint `PATCH /advance-care-directives/:id/view { type: 'secure_call' }` (directives call the same route with an empty body). **⚠ New gap (SCALL-O-6):** both new routes — `GET /advance-care-directives/admin/resident/:residentId` and `PATCH /advance-care-directives/:id/view` — carry only `authMiddleware`, no `requireAnyRole(STAFF, ADMIN)` — despite living under an `/admin/...` path segment, any authenticated caller (including a resident or family member) can fetch or mark-viewed another resident's directive/secure-call feed by residentId. Cross-referenced in clinical-records.md CLIN-GAP-21 (same route file, same defect class as CLIN-GAP-03). (`senior_living_admin` `ResidentDocuments.tsx`, `CallSummaryModal.tsx`, `use-fetch-resident-directives.ts`, `use-fetch-secure-call.ts`, `use-mark-directive-viewed.ts`; `senior_living_backend` `advanceCareDirective.routes.ts:44-48`)

---

## 4. Business rules & policies

| # | Rule | Source |
|---|---|---|
| SCALL-BR-1 | A record is created **before** dialing (PENDING) so a call that never connects still leaves an audit row; `callSid` is backfilled from the recording callback by staff identity. | SCALL-FR-01/03 |
| SCALL-BR-2 | `status` (per-recording lifecycle) is owned by the summary Lambda; `approvalStatus` (staff sign-off) is owned by staff — the two are deliberately separate fields. | `SecureCall.model.ts:78-83` |
| SCALL-BR-3 | Transcript, per-part summary, and the staff-reviewed summary are all envelope-encrypted at rest; only the reviewed `summary` is ever exposed to a resident/family, and only when `shareWithResident` is true. | SCALL-FR-04/05/06 |
| SCALL-BR-4 | Multi-tenancy: every route is facility-scoped; `SecureCall.facilityId` indexed; the caller-scoped `my-calls` read additionally filters by the caller's resident/family cName. | `SecureCall.model.ts` indexes |
| SCALL-BR-5 | The Twilio webhook verifies the request signature (fail-closed only when `TWILIO_ENFORCE_WEBHOOK_SIGNATURE=true`). | SCALL-FR-03 |

---

## 5. Notifications & real-time behavior

No push/socket events are emitted by this module today. A shared summary surfaces passively on the resident/family app's next `my-calls` fetch; there is no notification when a call summary is shared or when a summary awaits staff approval (candidate — see §9). The `viewedAt`/unviewed-dot is a pull-time UI affordance, not a push.

---

## 6. Integrations

| Integration | Use |
|---|---|
| **Twilio Programmable Voice** | Masked staff→resident/family dialing via `client:<staffId>` identity; call- and recording-status callbacks to `/webhooks/twillio` (signature-verified). |
| **AWS S3** | Recording audio storage (`recordingKeys[]`); signed URLs on read. |
| **Transcribe/summary Lambda (external)** | Matches `SAL_securecall_<id>_<part>`, writes KMS-encrypted per-part transcript/summary. Same external provider family as the care-conference summary Lambda (`SAL_*_SUMMARY_PROVIDER`). |
| **AWS KMS** | Envelope encryption of transcript/summary/reviewed-summary (`secureCallRecordingCrypto.ts`) — a distinct envelope path from chat (per-conversation), PCC meds (per-resident), and care-conference recordings. |

---

## 7. Permissions & access control

- **Placement / review:** `POST /`, `GET /`, `GET /:id`, `GET /:id/recording-urls`, `PUT /:id/update-summary` require `authMiddleware` + `requireAnyRole(STAFF, ADMIN)`.
- **Caller read:** `GET /my-calls` requires `authMiddleware` + `requireAnyRole(RESIDENT, FAMILY_MEMBER)`; family members read as themselves (not normalized to the resident, unlike chat).
- **Webhook:** `/webhooks/twillio/*` is unauthenticated by design (Twilio-originated) but **signature-verified** via `twilioSignature`.
- **Tenant isolation:** all reads facility-scoped; `my-calls` additionally scoped to the caller's own cName.

---

## 8. Product-split notes

- Secure Call is a **platform capability with no facility-type branching** in code. It is SN-flavored in practice because the staff app's clinical designations (Case Manager, Doctor, Social Worker, etc.) are the natural callers, and those designations exist predominantly in SNF facilities. An AL rollout needs no backend change — only a staff-app surface for AL-side designations.
- **Admin-web surface added v1.6, but still narrower than a full oversight view.** The new Resident Documents tab (SCALL-FR-08) lets admin/staff review and approve a single resident's call summaries, reached only from that resident's Detail page — there is still no facility-wide call log or approval queue (unlike, say, the referral/ACD "Pending Sign" mobile queue). Call *placement* remains staff-app-only; the admin web is review/approve-only. A facility-wide oversight list is the remaining open product decision (§9, SCALL-O-4 partially resolved).

---

## 9. Observations & candidate gaps

- **SCALL-O-1 — No notification on share or on pending approval.** A shared summary and an awaiting-approval call are both discoverable only by polling (`my-calls` / `?approvalStatus=PENDING`). Residents/family are not told a summary is available; supervising staff are not told a summary awaits approval. Candidate: FCM push on share and on approval-pending.
- **SCALL-O-2 — Recording-to-call correlation is by "newest PENDING for this staff", not a hard key.** Because `callSid` is unknown at create time, the recording callback matches the newest PENDING call for the staff identity. Two calls placed by the same staff member in quick succession before either recording lands could in principle correlate to the wrong record. Candidate: pass a correlation token through the TwiML `From`/`statusCallback` and match on it.
- **SCALL-O-3 — Webhook signature enforcement is opt-in.** Verification is best-effort unless `TWILIO_ENFORCE_WEBHOOK_SIGNATURE=true`. It should be fail-closed in every deployed environment before production (the mechanism already exists — only the flag needs setting).
- **SCALL-O-4 — No admin oversight surface — partially resolved (v1.6).** The new Resident Documents tab (SCALL-FR-08) gives admin/staff a per-resident review + approve view (AI summary, reviewed summary, transcript, approve). **Residual:** there is still no facility-wide call log / audit queue — the underlying `GET /` (admin-reachable, facility-scoped) has no admin-web list view built over it; a supervisor must open each resident individually to find pending-approval calls.
- **SCALL-O-5 — `console.*` logging on a PHI-adjacent path** (consistent with the repo-wide logging-hygiene gap) — notable for a recorded-call feature. Confirm no call content / phone numbers are logged in plaintext.
- **SCALL-O-6 — New admin-surface routes lack a role gate (pre-production, NEW, v1.6).** The two backend routes added for the Resident Documents tab (SCALL-FR-08) — `GET /advance-care-directives/admin/resident/:residentId` and `PATCH /advance-care-directives/:id/view` — carry only `authMiddleware`, no `requireAnyRole(STAFF, ADMIN)`, despite the `/admin/...` path segment implying elevated access. Any authenticated caller (including a resident or family member normalized to a *different* resident) can supply an arbitrary `residentId` and read or mark-viewed that resident's directive + secure-call feed. Same defect class as clinical-records.md CLIN-GAP-03 (missing role gates on medication/lab staff reads); cross-referenced there as CLIN-GAP-21. Candidate fix: add `requireAnyRole(STAFF, ADMIN)` to both routes.
- **SCALL-O-7 — Resident-facing share surface is backend-ahead-of-UI (new, 2026-08-21).** SCALL-FR-06 (`GET /api/secure-calls/my-calls`, `shareWithResident`) has been reachable since Secure Call's v1.5 introduction, but `senior_living_skillednursing_resident` (`origin/master` HEAD `4734daf`) — the platform's reference resident client and the only resident app with a Health hub — has no screen, service call, or navigation entry that reads it (zero references to `secure-calls`/`SecureCall` in the repo). A staff member can toggle `shareWithResident: true` today and the resident/family member has no way to see the result in-app. Candidate: a Health-hub "Calls" surface, or fold shared summaries into an existing feed (e.g. alongside lab reports/directives).
