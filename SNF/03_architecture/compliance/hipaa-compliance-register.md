# HIPAA Compliance — Gap Analysis & Prioritized Worklog (Skilled Nursing Facility App)

Converted from `HIPAA_Gap_Worklog.xlsx` (uploaded 2026-08-28), preserving all rows, IDs, and text exactly — this is compliance-sensitive content and nothing here is paraphrased or summarized from the source.

Companion to `HIPAA_Compliance_Reference.docx` (requirements & citations) and `Staff_Auth_Workflow.docx` (staff auth flow diagram + Android/iOS/Web implementation pointers), both in the Compliance folder. This document tracks current-state gaps and fixes for product/dev to prioritize and action. Update Status and Owner as work proceeds.

**Only Sathish works on this document** — see `shashi-care-sa-config.md` for the access note.

## Legend

| Field | Value | Meaning |
|---|---|---|
| Priority | High | Address before broader rollout / next release. |
| Priority | Medium | Address in near-term backlog. |
| Priority | Low | Good practice, lower urgency. |
| Status | Not Started | No work begun. |
| Status | In Progress | Actively being built (per team's stated roadmap). |
| Status | Decision Needed | Requires a product/security policy decision, not just engineering. |
| Status | Needs Analysis | Problem is confirmed but no defined fix yet — requires design/research before it becomes an engineering task. |
| Status | Done | Verified complete — update Notes with verification method/date. |

## Summary (quick-scan status tracking)

| ID | Category | Priority | Status | Owner |
|---|---|---|---|---|
| 1 | BAA / Vendor | High | Not Started | — |
| 2 | BAA / Vendor | High | Not Started | — |
| 3 | Encryption & Storage | High | Not Started | — |
| 4 | Encryption & Storage | Medium | Not Started | — |
| 5 | Audit & Logging | High | Not Started | — |
| 6 | Data Retention | Medium | Not Started | — |
| 7 | Data Integrity | Medium | Not Started | — |
| 8 | Privacy Notice | Low | Not Started | — |
| 9 | Device Management | Medium | Not Started | — |
| 10 | Identity & Access | High | Not Started | — |
| 11 | Identity & Access | Medium | Not Started | — |
| 12 | Identity & Access | Low | Not Started | — |
| 13 | Identity & Access | High | Not Started | — |
| 14 | Authentication | Medium | Not Started | — |
| 15 | Identity & Access | High | Not Started | — |
| 16 | Audit & Logging | Medium | Not Started | — |
| 17 | Authentication | High | In Progress | — |
| 18 | Consent & Family Access | Medium | Not Started | — |
| 19 | Consent & Family Access | Medium | Not Started | — |
| 20 | Session Management | High | In Progress | — |
| 21 | Session Management | Medium | Decision Needed | — |
| 22 | Encryption & Storage | High | Not Started | — |
| 23 | Breach Notification | High | Not Started | — |
| 24 | Data Integrity | High | Needs Analysis | — |
| 25 | Identity & Access | High | Not Started | — |
| 26 | Identity & Access | Medium | Not Started | — |
| 27 | Authentication | Medium | Not Started | — |
| 28 | Authentication | Low | Not Started | — |
| 29 | Authentication | Low | Not Started | — |
| 30 | Session Management | Medium | Not Started | — |
| 31 | Device Management | Medium | Not Started | — |
| 32 | Session Management | Medium | Decision Needed | — |
| 34 | Session Management | Medium | Not Started | — |
| 35 | Session Management | High | Not Started | — |
| 36 | Device Management | High | Needs Analysis | — |
| 37 | Identity & Access | High | Not Started | — |
| 38 | Identity & Access | High | Not Started | — |
| 39 | Session Management | Medium | Not Started | — |
| 40 | Consent & Family Access | Medium | Not Started | — |

## Full detail

### 1. BAA / Vendor — Infrastructure (AWS)

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.308(b)(1); §164.314(a) |
| Created | 2026-07-16 |

**Current state**: Confirmed pending — AWS BAA has not yet been executed.

**Gap / risk**: No signed BAA with AWS covers PHI processed via Cognito/S3.

**Recommended fix**: Accept the AWS Business Associate Addendum in AWS Artifact for this workload's AWS account.

**Notes**: Precondition for everything else in this list. Confirmed pending as of 2026-07-20.

---

### 2. BAA / Vendor — Infrastructure (AWS)

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.308(b)(1) |
| Created | 2026-07-16 |

**Current state**: Cognito and S3 assumed HIPAA-eligible.

**Gap / risk**: HIPAA-eligible services list changes over time and hasn't been re-checked against current architecture.

**Recommended fix**: Cross-check every AWS service touching PHI (Cognito, S3, compute, logging) against AWS's current HIPAA-eligible services list.

---

### 3. Encryption & Storage — PHI documents (S3)

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(2)(iv) |
| Created | 2026-07-16 |

**Current state**: Confirmed pending — SSE-KMS is not yet enabled on PHI document buckets.

**Gap / risk**: Data may be sitting under default SSE-S3 (or unconfirmed encryption) rather than customer-managed KMS keys, weakening key-access auditability and Breach Notification safe-harbor protection.

**Recommended fix**: Enable SSE-KMS with customer-managed keys on all PHI document buckets; deny non-HTTPS requests via bucket policy.

**Notes**: Confirmed pending as of 2026-07-20. Call recordings tracked separately — see row 22. Also affects Breach Notification safe harbor (row 23).

---

### 4. Encryption & Storage — All PHI traffic

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(e)(1)–(2) |
| Created | 2026-07-16 |

**Current state**: TLS assumed in use for app traffic.

**Gap / risk**: Not confirmed TLS is enforced (not just available) on every PHI-carrying endpoint, incl. PCC sync and internal service calls.

**Recommended fix**: Audit all PHI-carrying endpoints (app↔backend, backend↔PCC, backend↔S3, internal service-to-service) to confirm TLS 1.2+ is mandatory with no plaintext fallback.

---

### 5. Audit & Logging — All PHI access

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(b) |
| Created | 2026-07-16 |

**Current state**: Confirmed: the app has no structured audit controls today.

**Gap / risk**: PHI record-level access (views, downloads, call playback, consent grants/revocations) is not logged at all — not just an infra-vs-app-level gap, there is currently no structured mechanism.

**Recommended fix**: Design and add application-level audit logging for every PHI read/write, tamper-resistant, retained on a defined schedule.

**Notes**: Confirmed as of 2026-07-20 — no structured controls currently exist.

---

### 6. Data Retention — Call recordings

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | Minimum Necessary — 45 CFR §164.502(b) |
| Created | 2026-07-16 |

**Current state**: No confirmed retention/deletion schedule for call recordings.

**Gap / risk**: Indefinite retention of PHI-containing audio increases breach exposure without a documented purpose.

**Recommended fix**: Define and enforce a retention/deletion schedule for call recordings; document rationale.

**Notes**: See row 22 for the consolidated call-recording encryption + retention item.

---

### 7. Data Integrity — PHI documents (S3)

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(c)(1) |
| Created | 2026-07-16 |

**Current state**: Versioning status on PHI document buckets not confirmed.

**Gap / risk**: Without versioning, accidental/malicious overwrites or deletions of PHI documents are not detectable or recoverable.

**Recommended fix**: Enable S3 object versioning on all PHI/document buckets.

---

### 8. Privacy Notice — Resident & family onboarding

| Field | Value |
|---|---|
| Priority | Low |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.520 |
| Created | 2026-07-16 |

**Current state**: No Notice of Privacy Practices step in onboarding.

**Gap / risk**: Residents/family are not shown or asked to acknowledge the facility's NPP within the app.

**Recommended fix**: Add an NPP link/acknowledgment step to resident and family onboarding flows; confirm correct NPP version per facility.

**Notes**: Primarily the SNF's (covered entity's) obligation; app is a natural surface for it.

---

### 9. Device Management — Staff mobile devices

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.310(d) |
| Created | 2026-07-16 |

**Current state**: No confirmed MDM policy referenced for staff devices.

**Gap / risk**: Staff devices accessing PHI may lack remote wipe, lock-screen enforcement, or restrictions on local PHI storage.

**Recommended fix**: Document and enforce an MDM policy for staff devices: remote wipe, lock-screen enforcement, no offline PHI cache.

**Notes**: Also a compensating control for the lost-device scenario (rows 10–11).

---

### 10. Identity & Access — Staff & physicians

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(1)–(2) |
| Created | 2026-07-16 |

**Current state**: Lost/stolen device handled by staff calling facility admin to deactivate account in Cognito (global sign-out).

**Gap / risk**: Cognito access tokens are stateless JWTs — AdminUserGlobalSignOut revokes refresh tokens but an already-issued access token keeps working (via local signature/expiry checks) until it naturally expires, unless the backend explicitly checks revocation status.

**Recommended fix**: Enable Cognito token revocation (jti/origin_jti claims). Keep access token TTL short (5–15 min). Have the backend check a fast revocation lookup (keyed by jti, populated the moment global sign-out fires) on every PHI-serving request — don't rely on local JWT signature/expiry checks alone.

**Notes**: This is the core fix; rows 11–12 are complementary, not substitutes.

---

### 11. Identity & Access — Staff & physicians

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(1)–(2) |
| Created | 2026-07-16 |

**Current state**: Proposed: check account validity on app launch/foreground.

**Gap / risk**: Only catches revocation at discrete launch/foreground events — a continuously open, foregrounded session between checks isn't covered. Only closes the gap if the check calls something that honors Cognito's revocation state (e.g., GetUser), not a local JWT re-validation.

**Recommended fix**: Implement the launch/foreground check against a Cognito API call that enforces revocation (e.g., GetUser). Treat as a good addition on top of row 10's backend fix, not a replacement for it.

---

### 12. Identity & Access — Staff & physicians

| Field | Value |
|---|---|
| Priority | Low |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(1)–(2) |
| Created | 2026-07-16 |

**Current state**: Proposed: push notification to trigger local token invalidation.

**Gap / risk**: Push delivery isn't guaranteed or instant (network/OS constraints); doesn't help if the device is offline.

**Recommended fix**: Use as a UX accelerant to shorten average time-to-lockout, layered on top of the server-side revocation check in row 10 — not a standalone control.

---

### 13. Identity & Access — Staff & physicians

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(2)(ii) |
| Created | 2026-07-16 |

**Current state**: No documented emergency access path if a clinician's MFA device is unavailable during a care emergency.

**Gap / risk**: §164.312(a)(2)(ii) Emergency Access Procedure is a required (not addressable) specification and currently undefined for this app.

**Recommended fix**: Define a break-glass access path (e.g., time-boxed elevated access granted by a second authorized staff member/admin), with mandatory post-hoc audit review of every use.

---

### 14. Authentication — Staff & physicians

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | NIST SP 800-63B (industry benchmark, not HIPAA text) |
| Created | 2026-07-16 |

**Current state**: OTP delivery channel not confirmed (SMS vs. authenticator app/push).

**Gap / risk**: If SMS-delivered, NIST 800-63B flags SMS as a “restricted” authenticator due to SIM-swap risk.

**Recommended fix**: Confirm OTP delivery channel; offer TOTP app-based or push-based OTP as an alternative, at least for staff/physician accounts (broadest access).

---

### 15. Identity & Access — External physicians

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.308(a)(3) (workforce/access authorization, by analogy) |
| Created | 2026-07-16 |

**Current state**: Physicians provisioned as app users; onboarding/offboarding process not confirmed.

**Gap / risk**: Physicians aren't facility employees — no confirmed tie to medical-staff credentialing at enrollment, nor a defined deprovisioning trigger when their affiliation/privileges end.

**Recommended fix**: Tie physician account creation to facility credentialing/medical-staff roster verification; define an offboarding trigger independent of employee HR offboarding.

**Notes**: Non-employee accounts are typically the weakest offboarding link.

---

### 16. Audit & Logging — Staff & physicians

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(b) |
| Created | 2026-07-16 |

**Current state**: Lost-device deactivation handled manually by facility admin.

**Gap / risk**: Not confirmed whether the deactivation event itself (who reported, when revoked) is captured in an audit trail.

**Recommended fix**: Add an audit log entry type for account deactivation / global sign-out events, timestamped and tied to the reporting workflow.

**Notes**: Also feeds breach risk assessment if PHI exposure during the gap window is ever in question.

---

### 17. Authentication — Family members & residents

| Field | Value |
|---|---|
| Priority | High |
| Status | In Progress |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(d); NIST SP 800-63B AAL2 |
| Created | 2026-07-16 |

**Current state**: Resolved design: no password, by intent, for adoption ease. Backend mediates login via Cognito Admin API; OTP is the primary factor. Mandatory biometric/PIN registration (Face ID > Fingerprint > Pattern > PIN) now serves as the intended second MFA factor. V1 ships this as a local-only biometric gate; V2 will add server-side verification.

**Gap / risk**: OTP alone is single-factor; the biometric/PIN addition closes that as long as it's a real second factor. In V1 (local-only gate), the backend has no way to verify the biometric check actually happened — a modified client could skip it — so V1 does not yet provide server-verifiable two-factor assurance, even though the UI presents it as MFA.

**Recommended fix**: V1 (in progress): ship the local biometric/PIN gate as designed. V2 (tracked separately): implement a device-bound key pair (Secure Enclave / Keystore, biometric-gated) with challenge-response signing verified server-side — e.g., via a Cognito CUSTOM_AUTH flow (DefineAuthChallenge / CreateAuthChallenge / VerifyAuthChallengeResponse Lambda triggers) — so the second factor is cryptographically enforceable, not just a local UI gate.

**Notes**: V1/V2 split agreed 2026-07-29. Since the backend now calls Cognito Admin APIs directly, confirm that backend service runs under a tightly scoped, least-privilege IAM role.

---

### 18. Consent & Family Access — Family members & residents

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.502(g); §164.510(b) |
| Created | 2026-07-16 |

**Current state**: Resident signs a consent form on a staff iPad naming authorized family members; family member views the signed consent and acknowledges it via a checkbox on first login (no signature required from the family member).

**Gap / risk**: Consent capture doesn't appear to distinguish who signed — the resident (self) vs. a personal representative acting under state-law authority. These are two distinct legal bases under HIPAA.

**Recommended fix**: Add a field at consent capture: “Signed by: Resident (self) / Personal Representative — [authority type: POA / Guardian / Other]”.

---

### 19. Consent & Family Access — Family members & residents

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.510(b) |
| Created | 2026-07-16 |

**Current state**: Consent appears to be set up via a staff-mediated iPad flow.

**Gap / risk**: Not confirmed whether the resident/personal representative can revoke a family member's access directly and immediately from their own app, vs. only via another staff-mediated visit.

**Recommended fix**: Confirm (or add) a self-service revoke-access control in the resident's app, enforced immediately across all of that family member's active sessions.

**Notes**: Real-time revocation was already flagged as a requirement in the reference document, Section 4.

---

### 20. Session Management — All roles

| Field | Value |
|---|---|
| Priority | High |
| Status | In Progress |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(2)(iii) |
| Created | 2026-07-16 |

**Current state**: Sessions currently auto-log-off after 7 days with no shorter inactivity lock in front of it.

**Gap / risk**: No control today guards against an unattended, unlocked device between full logins — only the 7-day ceiling exists.

**Recommended fix**: Ship the planned 5-minute inactivity lock. It must require genuine local re-authentication (PIN/biometric) to resume — a cosmetic screen dim that resumes on tap satisfies nothing. Use the same PIN/biometric mechanism for both staff and family/resident (also fixes row 17).

**Notes**: Already on the team's roadmap (Next Steps #1).

---

### 21. Session Management — All roles

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Decision Needed |
| Owner | — |
| HIPAA / Standard reference | NIST SP 800-63B AAL2 (industry benchmark, not HIPAA text) |
| Created | 2026-07-16 |

**Current state**: Proposed change: 7-day auto-log-off only after 7 days of inactivity; mandatory full re-auth every 30 days even if continuously active.

**Gap / risk**: Loosens today's absolute ceiling (currently 7 days regardless of activity) to 30 days for continuously active sessions — moves further from the NIST AAL2 benchmark (12-hour absolute reauthentication), which governs long-lived session/credential-compromise risk, a different risk than the unattended-device risk row 20 addresses.

**Recommended fix**: Acceptable only as a documented risk trade-off, and only once row 20 (genuine 5-min local lock) and row 10 (server-side token revocation) are both in place. Otherwise, consider keeping the active-use ceiling closer to 7–14 days. Either way, write the rationale into the Security Risk Assessment explicitly.

**Notes**: Decision for product/security owner, not purely an engineering task.

---

### 22. Encryption & Storage — Call recordings (PHI)

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(2)(iv); Minimum Necessary — 45 CFR §164.502(b) |
| Created | 2026-07-20 |

**Current state**: Call recordings are encrypted at rest, but the specific encryption method (SSE-S3 vs. customer-managed SSE-KMS) and a retention/deletion policy are both unconfirmed.

**Gap / risk**: Without confirming encryption type, Breach Notification safe-harbor status (row 3's KMS rationale) can't be claimed for recordings specifically. Without a retention policy, PHI-containing audio may be retained indefinitely, increasing breach exposure with no documented purpose.

**Recommended fix**: Confirm/upgrade call recordings to SSE-KMS with customer-managed keys (align with row 3). Separately, define and enforce a retention/deletion schedule specific to recordings, and document the rationale (e.g., tied to care-quality review windows, not indefinite).

**Notes**: Consolidates the call-recording-specific parts of rows 3 and 6. Confirmed pending as of 2026-07-20.

---

### 23. Breach Notification — Organization-wide

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §§164.400–414 |
| Created | 2026-07-20 |

**Current state**: No incident response / breach notification runbook exists today.

**Gap / risk**: §§164.400–414 require a documented risk assessment (the §164.402 four-factor test) to determine if an incident is a reportable breach, plus statutory notification deadlines (individuals: ≤60 days; HHS: ≤60 days for 500+ affected, annual log for under 500; media: required for 500+ in one state/jurisdiction). Without a runbook, there's no assigned owner for the risk assessment and a real risk of missing a deadline during an actual incident.

**Recommended fix**: Draft an incident response runbook: assign who performs the breach risk assessment, define notification timelines and templates, and reference this app's specific systems by name (PCC sync, S3 buckets, Cognito, call-recording pipeline) rather than writing a generic policy.

**Notes**: Confirmed not ready as of 2026-07-20. Should reference rows 3, 5, and 22 once those controls exist, since they affect safe-harbor and evidence available during an investigation.

---

### 24. Data Integrity — Staff edits, family-submitted docs, physician-signed orders

| Field | Value |
|---|---|
| Priority | High |
| Status | Needs Analysis |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(c)(1) |
| Created | 2026-07-20 |

**Current state**: Family members can edit their own information; the app accepts uploads of consent forms, mobile-scanned test reports, call recordings posted to PCC Communications, and physician-signed referral order summaries. Nothing is currently written back to PCC, but FHIR write-back APIs are available and planned soon. No marker currently distinguishes app-originated data/edits from PCC-synced data.

**Gap / risk**: Once FHIR write-back is enabled, there will be no way to tell whether a given field or document is the authoritative PCC-sourced record or an app-originated edit/upload — risking silent overwrites, conflicting versions between systems, and attribution disputes. Family-editable fields also have no confirmed change-tracking (who changed what, when) today.

**Recommended fix**: Needs dedicated analysis and design before FHIR write-back ships, covering at minimum: (1) a source/provenance marker (app-authored vs. PCC-synced) on every editable field and uploaded document; (2) change-tracking (versioning + audit trail) on all family-editable fields; (3) a conflict-resolution policy for the PCC write-back path (e.g., PCC remains system of record, app edits are proposed changes requiring PCC-side confirmation) defined before enabling FHIR writes.

**Notes**: Flagged as requiring analysis/solutioning, not yet a defined engineering fix. Blocks the planned FHIR write-back launch. Related to row 7 (S3 versioning), which is a narrower, already-actionable subset.

---

### 25. Identity & Access — Staff & physicians

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(1)–(2); §164.312(d) |
| Created | 2026-07-29 |

**Current state**: Biometric-gated app unlock relies on OS-level key invalidation (iOS biometryCurrentSet / Android's default biometric-enrollment invalidation) when device biometrics change, but app behavior on invalidation isn't yet defined.

**Gap / risk**: Invalidating the local key alone doesn't stop misuse — if the app allows silently re-registering a new biometric right after invalidation, an imposter who enrolls their own biometric on a stolen/unlocked device can regain access without ever supplying the original password/OTP.

**Recommended fix**: On catching the OS's specific 'key permanently invalidated' error (not a generic biometric-mismatch failure), wipe local session state, revoke the current Cognito session server-side (don't just redirect locally), and require a full password+OTP login before any new biometric can be enrolled. Log the invalidation event and, if feasible, alert the user via another channel (e.g., email) that a new biometric was registered on their device.

**Notes**: Confirm iOS uses biometryCurrentSet (not the similar-looking biometryAny, which does not invalidate) and Android does not disable setInvalidatedByBiometricEnrollment. Narrows one attack path; see row 26 for its limits.

---

### 26. Identity & Access — Staff & physicians

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(1)–(2) |
| Created | 2026-07-29 |

**Current state**: Staff devices allow saved/autofilled passwords; OTP is delivered to the same device via SMS or an already-logged-in email app.

**Gap / risk**: On a stolen but unlocked device, both authentication factors (autofilled password + device-delivered OTP) may be trivially available to whoever holds it — meaning the forced full re-login in row 25 does not fully protect against this specific scenario. The real backstop is prompt reporting and admin-side revocation.

**Recommended fix**: No code fix alone closes this. Ensure the lost-device reporting process (facility admin deactivation) is fast and clearly communicated to staff, and prioritize row 10 (server-side token revocation), since that is the control that actually limits exposure once a device is reported lost.

**Notes**: Documents a known limitation rather than a new fix — ties rows 10 and 25 together.

---

### 27. Authentication — Staff & physicians

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(d) |
| Created | 2026-07-29 |

**Current state**: Temp password sent via both SMS and email at account creation.

**Gap / risk**: Sending the identical secret over two channels doubles exposure (compromise of either channel exposes the credential) without a security benefit.

**Recommended fix**: Send the temp password to only the channel the user will authenticate with; use the other channel solely for an out-of-band 'account created' notice. Confirm the temp password is single-use and expires quickly (e.g., 24–48 hours), not Cognito's longer default.

---

### 28. Authentication — Staff & physicians

| Field | Value |
|---|---|
| Priority | Low |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(d) |
| Created | 2026-07-29 |

**Current state**: Flow assumes a forced permanent-password reset happens after first login with the temp password (Cognito NEW_PASSWORD_REQUIRED), but this isn't explicitly documented or confirmed.

**Gap / risk**: If unconfirmed, staff could continue using a system-generated temporary password indefinitely.

**Recommended fix**: Explicitly document and confirm the forced password-reset step and the password complexity policy as part of first login.

---

### 29. Authentication — Staff & physicians

| Field | Value |
|---|---|
| Priority | Low |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(d) |
| Created | 2026-07-29 |

**Current state**: The 5-minute inactivity lock unlocks via a tap plus biometric; Face ID could technically trigger passively without the tap.

**Gap / risk**: A purely passive Face ID scan (no explicit user gesture) could be triggered by someone holding the device up to a distracted or non-consenting user's face.

**Recommended fix**: Keep an explicit tap/gesture before triggering Face ID rather than a passive auto-trigger, and confirm the OS's attention/liveness detection is left enabled, not disabled for speed.

---

### 30. Session Management — Staff & physicians

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(2)(iii) |
| Created | 2026-07-29 |

**Current state**: App-open/relaunch (after kill or reboot) shows a biometric lock screen.

**Gap / risk**: If the underlying session/token has already expired or been revoked, showing a biometric lock screen first is a wasted, potentially misleading step — the app should route straight to full login in that case.

**Recommended fix**: On every app launch, check session/token validity first (ideally a live check, not just a locally cached expiry) and branch: valid session → biometric lock screen; expired/revoked/fresh install → full password+OTP login screen directly.

---

### 31. Device Management — Staff & physicians (Web/Desktop)

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(d) |
| Created | 2026-07-29 |

**Current state**: Plan assumes at least one OS biometric/PIN is registered on every device across Android, iOS, Mac, and Windows, or app usage is blocked entirely.

**Gap / risk**: Reasonably safe for mobile (a lock-screen credential is nearly universal), but not safe for desktop web: many corporate/older Windows laptops lack Windows Hello hardware, and most Mac desktops (iMac, Mac mini, external-keyboard setups) have no Touch ID at all — macOS also lacks a broad OS-level PIN API equivalent to Windows Hello PIN. External physicians on personal, non-managed devices add a further access-denial risk with no obvious support path.

**Recommended fix**: For web/desktop, don't rely on a biometric-hardware assumption (see row 32's web decision). Document a support path for external physicians who hit a hard block. Add a 'use password instead' escape hatch for temporary biometric failures (bandaged finger, PPE affecting Face ID, a faulty camera) rather than a hard lockout, given this is a clinical app where availability matters.

---

### 32. Session Management — Staff & physicians (Web)

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Decision Needed |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(2)(iii) |
| Created | 2026-07-29 |

**Current state**: Decision: implement the biometric-gated lock on mobile only for v1; web uses a simple full logoff after 15 minutes of inactivity (password + OTP to resume), with the same 7-day/30-day ceiling layered on top.

**Gap / risk**: Reasonable v1 trade-off and compliant on its own terms, but web sessions often run on more physically shared/exposed workstations (nursing stations, shared office desktops) than a personal mobile device — 15 minutes may be looser than the actual physical risk warrants if any of these are shared/kiosk-style machines rather than personally-assigned laptops.

**Recommended fix**: Confirm whether facility web access is via personally-assigned or shared workstations; tighten the timeout (e.g., closer to 5–10 min) for shared machines. Implement the check using elapsed wall-clock time since last activity (checked on tab/window focus or OS wake), not a live JavaScript countdown alone, since browsers throttle background timers and a naive implementation may not fire reliably after the machine sleeps or the tab is backgrounded.

**Notes**: Revisit WebAuthn (Windows Hello / Touch ID via browser) in a later phase for parity with mobile and reduced login friction.

---

### 34. Session Management — Residents & family members

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(2)(iii) |
| Created | 2026-07-29 |

**Current state**: Decision: 30-minute inactivity window for this role (vs. staff's 5 minutes), justified by a different risk profile — a personal device exposing at most one consented resident's data, not a shared workstation with broad multi-resident access.

**Gap / risk**: None — this is a documented, risk-based decision, not an open gap. Recorded here so the rationale is captured for the Security Risk Assessment.

**Recommended fix**: Implement the 30-minute window; document the staff-vs-resident risk-profile rationale explicitly alongside it.

**Notes**: Decision made 2026-07-29.

---

### 35. Session Management — Residents & family members

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | Minimum Necessary — 45 CFR §164.502(b) |
| Created | 2026-07-29 |

**Current state**: Plan: gate only sensitive content (health records, documents, consent management) behind the biometric/PIN check; allow low-sensitivity views (dashboard, messages, activity feed) to resume without re-prompting within the 30-minute window.

**Gap / risk**: Reduces how often an elderly or low-tech user must authenticate at all, without weakening protection on the content that matters — but requires product to explicitly classify which screens count as "sensitive" vs. not, or the gating will be inconsistently applied.

**Recommended fix**: Define and document the sensitivity classification per screen/feature; implement step-up authentication (biometric/PIN prompt) only when navigating into a classified-sensitive screen.

**Notes**: Same pattern as mobile banking apps: view balance freely, authenticate to transact.

---

### 36. Device Management — Residents (elderly-specific)

| Field | Value |
|---|---|
| Priority | High |
| Status | Needs Analysis |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(d); general reasonableness of Security Rule safeguards |
| Created | 2026-07-29 |

**Current state**: App mandates at least one biometric/PIN, else usage is prohibited entirely, with no accommodation defined for users who can't reliably manage any of them.

**Gap / risk**: Elderly residents are more likely than staff or family to have unreliable fingerprint scans (thinner/drier skin), and some will have cognitive or vision impairment making even PIN entry difficult — a hard mandate risks locking residents out of their own health information with no recourse.

**Recommended fix**: Confirm the PIN entry UI is accessibility-tuned (large touch targets, high contrast, forgiving of retries). For residents who genuinely cannot manage any device credential, define an explicit assisted-setup accommodation (a family member or facility staff helps set up/manage access) tied to the existing personal-representative concept, rather than a silent hard lockout.

**Notes**: Needs a defined accommodation process, not just a UI fix — flagged as requiring analysis, matching row 24's convention.

---

### 37. Identity & Access — Residents & family members

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(1)–(2) |
| Created | 2026-07-29 |

**Current state**: No lost-device reporting or revocation path is defined for this role. Unlike staff, there is no admin console or web access for them to use.

**Gap / risk**: A lost or stolen resident/family device has no defined way to get access revoked promptly, unlike staff's "call facility admin" path.

**Recommended fix**: Two complementary paths: (1) self-service — an in-app "manage my devices/sessions" screen, usable from any other device already logged into the same account, to revoke a specific session immediately; (2) staff-assisted — extend the existing staff app's resident/family management screen (already used for consent capture) with a "revoke this person's access" action, for cases with no second device to self-serve from (most relevant for residents specifically). Both should call the same backend session-revocation mechanism being built for staff (row 10), not a separate implementation.

**Notes**: Staff-assisted path reuses the consent-management relationship already being built — avoids standing up a separate support channel.

---

### 38. Identity & Access — Residents & family members

| Field | Value |
|---|---|
| Priority | High |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(1)–(2); §164.312(d) |
| Created | 2026-07-29 |

**Current state**: Same biometric-key-invalidation exposure as staff (rows 25–26) applies to this role once the mandatory biometric/PIN ships.

**Gap / risk**: If the OS reports the biometric key as permanently invalidated (a new biometric was enrolled on the device) and the app allows silent re-registration, an imposter with a lost/found device could regain access without ever supplying the original OTP.

**Recommended fix**: Apply the same fix as rows 25–26: on the OS's specific key-invalidated error, wipe local session state, revoke the session server-side, and require a full OTP login before allowing new biometric registration. Same iOS (biometryCurrentSet, not biometryAny) / Android (setInvalidatedByBiometricEnrollment) implementation guidance applies.

**Notes**: Mirrors rows 25–26 for this role.

---

### 39. Session Management — Residents & family members

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.312(a)(2)(iii) |
| Created | 2026-07-29 |

**Current state**: App relaunch (after kill or device reboot) shows the app lock screen.

**Gap / risk**: Same issue as staff (row 30): if the underlying session/token has already expired or been revoked, showing the biometric lock screen first is a wasted, potentially misleading step.

**Recommended fix**: Check session/token validity first (live check, not a cached expiry) and branch: valid session → lock screen; expired/revoked/fresh install → full OTP login directly.

**Notes**: Mirrors row 30 for this role.

---

### 40. Consent & Family Access — Residents & family members

| Field | Value |
|---|---|
| Priority | Medium |
| Status | Not Started |
| Owner | — |
| HIPAA / Standard reference | 45 CFR §164.502(g); §164.510(b) |
| Created | 2026-07-29 |

**Current state**: The resident-signs / family-acknowledges (checkbox) consent flow (rows 18–19) was defined separately from this authentication flow, and the new sign-in write-up doesn't explicitly reference it.

**Gap / risk**: Risk of the two designs drifting apart if not explicitly reconciled — e.g., unclear whether consent capture triggers the install-link SMS/email (sign-in step 1) and exactly when the checkbox acknowledgment happens relative to biometric registration (sign-in step 5).

**Recommended fix**: Confirm and document explicitly how the consent/acknowledgment flow (rows 18–19) integrates with the sign-in sequence described here, so the two aren't maintained as separate, potentially inconsistent designs.

**Notes**: Confirmation/documentation item, not a new technical gap.

---
