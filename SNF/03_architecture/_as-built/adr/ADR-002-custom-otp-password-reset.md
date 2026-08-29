# ADR-002 — Custom SMS OTP password reset on top of Cognito

**Status:** Accepted (as-built, production HEAD `981704e9`)
**Date:** 2026-06-17 (documenting an existing decision)
**Area:** `senior_living_backend` — `routes/authReset.routes.ts`, `routes/authStaffReset.routes.ts`, `services/cognitoReset.service.ts`, `models/PasswordResetOtp.model.ts`
**Related:** PRD `platform-foundation.md` PLAT-FR-13a/13b/14a, backend architecture "Authentication & account recovery"

## Context

AWS Cognito is the system of record for credentials and MFA. Cognito's native forgot-password flow exists (and remains wired on one admin-web component). But the platform is **phone-first** (Cognito username = E.164 phone) and the product wanted a fully controlled, SMS-delivered OTP recovery — including a staff variant keyed by email — with explicit attempt/cooldown policy and a UI that can validate the OTP before asking for a new password.

## Decision

Implement a **custom OTP reset layer** mounted at `/api/auth/*` **before** `facilityMiddleware` (no tenant header, no auth token — the only unauthenticated write paths outside `/webhooks/*`):
- Resident/family by **phone** (3-step: `forgot-password` → `verify-otp` → `reset-password`); staff by **email** (2-step: `forgot-password` → `reset-password`, OTP checked inline).
- OTPs are SHA-256-hashed in `PasswordResetOtp` with a Mongo **TTL index**; 300 s validity, 3-attempt cap, 60 s resend cooldown, single-use (`deleteMany` on issue and on success).
- Reset applies Cognito `AdminSetUserPassword (Permanent:true)` then `AdminUserGlobalSignOut`.
- SMS via AWS SNS (`Transactional`) with sender-ID branching for non-US numbers.

## Consequences

**Positive**
- Phone-first SMS recovery aligned with the platform identity model; full control of OTP policy and copy.
- TTL + attempt cap + cooldown + single-use give a clear, auditable security boundary; `express-rate-limit` adds 5/15-min (forgot) and 10/15-min (verify+reset) guards.

**Negative / follow-ups**
- **The platform now owns the OTP security boundary** (previously Cognito's responsibility) — a deliberate trade-off requiring ongoing care.
- **Per-replica rate limiting:** the in-process memory store means limits are effectively `max × replicas` on the ECS Fargate target. *(needs a Redis-backed store)*
- **Email non-unique resolution (staff):** since email is now non-unique (PLAT-FR-10), `Staff.findOne({ email })` can resolve the wrong record. *(key staff reset on phone, or enforce uniqueness for reset-eligible staff)*
- **No verified client consumer for `/api/auth/staff/*`** yet; the admin-web "staff" forgot-password hooks actually call the phone-keyed resident endpoints — naming/wiring to reconcile.
- **No rollback** if `GlobalSignOut` fails after the password is set (old sessions live until natural expiry).
