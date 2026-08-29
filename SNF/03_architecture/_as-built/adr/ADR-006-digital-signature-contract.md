# ADR-006 — Digital-signature ("Pending Sign") document contract and audit posture

**Status:** Proposed
**Date:** 2026-07-12
**Area:** `senior_living_staffapp` — `PendingSignScreen`, `PendingSignDetailScreen`, `SignedPdfPreviewScreen` (client-side signed-PDF generation, `jspdf`/`pdf-lib`); `senior_living_backend` — inferred backend contract (`POST /api/auth/send-credentials`, unnamed digital-signature list/detail endpoints) — **not independently verified this cycle**
**Related:** `architecture-senior-living-product.md` §2.8; [`review-senior_living_staffapp.md`](../../reviews/2026-07-12/review-senior_living_staffapp.md)

## Context

`senior_living_staffapp` shipped a doctor-gated ("`isDoctorDesignation()`") digital-signature module this cycle. For doctor-group staff, it **replaces** the My Schedule tab in the MIGRATED tab bar with a "Pending Sign" queue. The client generates a multi-signature-per-page signed PDF locally (`jspdf`/`pdf-lib`) and appears to submit it against a backend contract that includes at least `POST /api/auth/send-credentials` (a lighter-weight credential-resend path) and unnamed digital-signature list/detail endpoints, inferred from client code only.

This matters architecturally for two reasons, neither of which was resolved this cycle because the backend repo's own 2026-07-11 review window did not cover the branch that presumably implements this contract:

1. **Data model.** No `data-schema.md` model exists yet for a "pending sign document." The client's implicit contract (list of pending documents per doctor, a detail/preview view, a signed-PDF submission) implies some backend collection or field set that has not been confirmed, named, or reviewed for facility-scoping (`facilityId`) or PHI-handling.
2. **Compliance/audit.** Digitally signed documents produced by a physician are, on their face, **legally relevant records** — the kind of artifact regulators and litigation discovery could ask about (who signed, when, what version, whether it was later modified). The existing platform debt registry (`technical-debt.md` `SL-TD-05`/`SL-TD-06`/`SL-DEF-04`) already documents that profile changes, resident changes, and care-conference actions have **no dedicated audit trail** — only field-level timestamps. If the digital-signature backend follows the same pattern, a legally-relevant signed document could exist with no actor/timestamp/version audit trail beyond whatever the client happened to submit.

## Options Considered

### Option A — Treat as an extension of the existing appointment/document pattern (no dedicated audit trail, consistent with current platform posture)
- **Pros:** Consistent with `SL-TD-05`/`SL-TD-06`/`SL-DEF-04` — the platform has already made this trade-off (field-timestamp attribution, no dedicated audit log) for other PHI-adjacent flows, accepted for the Redwood Grove first deployment.
- **Cons:** Signed documents carry materially higher compliance weight than a profile edit or a care-conference summary — a missing audit trail here is a different risk class, not just "more of the same."

### Option B — Require a dedicated audit trail (actor, timestamp, signature hash/version, prior-version retention) before this module ships to any facility that treats these documents as records of care
- **Pros:** Matches the legal/compliance weight of a physician's signature. Precedent exists in the platform for treating some PHI classes more strictly (e.g. KMS envelope encryption tiers in `data-schema.md` §3).
- **Cons:** Net-new backend build; blocks the feature from shipping to a facility that needs it until the audit layer lands. Scope/cost unknown until the actual backend contract is read.

### Option C — Ship as-is for facilities that treat these as internal convenience documents (not the legal record of care), explicitly out of scope for any facility requiring signed-document compliance controls
- **Pros:** Unblocks the current deployment; defers the compliance question to whichever facility actually needs it.
- **Cons:** Requires a product-level guarantee that no in-scope facility treats these as the authoritative signed record — risky to assert without a compliance/legal review.

## Decision

**Not yet decided — proposed for human/architect review**, and **cannot be fully decided until the actual backend contract is read** (out of scope for this documentation-only pass; the backend repo's 2026-07-11 review window did not cover this branch). This ADR exists to (a) flag that a legally-relevant document type shipped without a confirmed data model or audit posture, and (b) force an explicit choice among A/B/C rather than letting the existing "no dedicated audit trail" default apply by omission.

## Consequences

### Positive
- Prevents an implicit, undocumented compliance posture for a document class (physician signatures) that is materially different from the platform's other PHI-adjacent flows.
- Gives the next backend architecture-doc pass a concrete, named question to resolve (confirm the endpoint contract, the underlying model, and the audit posture together).

### Negative
- Until the backend contract is confirmed, this ADR cannot be promoted past `proposed` — it is a placeholder for a decision, not the decision itself.

### Follow-ups
- [ ] Backend-side review: confirm the actual endpoint set, request/response shapes, and underlying Mongoose model(s) for the digital-signature / "pending sign" feature (`senior_living_backend`, branch not identified this cycle).
- [ ] Once confirmed, add the model to `data-schema.md` §2 with full field/index/encryption detail (replacing the "likely model" hedge in `architecture-senior-living-product.md` §2.6).
- [ ] Product/compliance decision: which option (A/B/C) applies, and whether it varies per facility.
- [ ] If Option B, scope the audit-trail build alongside the existing `SL-TD-05`/`SL-TD-06`/`SL-DEF-04` audit-trail work rather than as a fourth parallel effort.

## References
- `architecture-senior-living-product.md` §2.8 (Digital signature / "Pending Sign" module)
- `technical-debt.md` `SL-TD-05`, `SL-TD-06`, `SL-DEF-04` (existing no-audit-trail debt, same family)
- [`review-senior_living_staffapp.md`](../../reviews/2026-07-12/review-senior_living_staffapp.md) — "Digital Signature / 'Pending Sign' module" section
