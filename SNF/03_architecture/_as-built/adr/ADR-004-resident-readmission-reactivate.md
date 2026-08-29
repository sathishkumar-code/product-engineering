# ADR-004 — Resident re-admission reactivates the existing record

**Status:** Accepted (decision — **build pending**; supersedes the 2026-06-20 "defer re-admission" disposition)
**Date:** 2026-06-21
**Area:** `senior_living_backend` — `src/controllers/resident.controller.ts` (create + delete paths), `src/integrations/pms/pcc/webhooks/handlers/patient/readmit.ts`, `src/models/resident.model.ts` (indexes)
**Related:** PRD `platform-foundation.md` PLAT-FR-58/59, `clinical-records.md` CLIN-FR-20a/CLIN-GAP-18; ADR-001 (convergent state); debt `SL-TD-07`, `SL-TD-09`; UAT RES-OQ-08 / RES-AC-12

## Context

A discharged/deleted resident may return and must be re-admitted. Two facts about the as-built code shape the decision:

1. **Identity = phone.** A resident's `cName` is the AWS Cognito username, which is the **phone number in E.164** (data-schema §2.1). The Mongo index `{facilityId, countryCode, phone}` is **not unique** ("per-facility uniqueness enforced in app"). The only de-facto uniqueness is Cognito rejecting a duplicate username at create time (`409 COGNITO_PHONE_ALREADY_EXISTS`, `resident.controller.ts:690-697`).
2. **Delete is a soft-delete.** Delete sets `deletedAt = new Date()` (`:1822`) and best-effort deletes the Cognito user (`:1834`); `status` is left untouched.

As-built, re-adding a same-phone resident **creates a brand-new document** (fresh `_id`, same `cName`), after proactively deleting the stale Cognito user (`:408-426`). The old soft-deleted row remains. This produces a **live + dead pair sharing one `cName`**. Because ~15 identity lookups do `Resident.findOne({ cName })` **without** `deletedAt: null` (e.g. `tvAuth.controller.ts:215`, `menu.controller.ts:177`, `chatAccessPolicy.ts:304/336/460`, `medication.controller.ts:221/323`, `labReports.controller.ts:93`, `pcc.service.ts:35`), such a lookup returns one match arbitrarily — typically the older, first-inserted row, i.e. the **soft-deleted ghost**. Admin list endpoints *do* filter `deletedAt: null`, so the lists look clean and mask the hazard (`SL-TD-09`). Separately, the PCC `patient.readmit` handler sets `status: Active` but never clears `deletedAt`, leaving a contradictory hidden record (`CLIN-GAP-18`).

Two implementation models were available:
- **New-record (as-built):** insert a fresh document on re-add; leave the soft-deleted row. Requires adding `deletedAt: null` to ~15 lookup sites to be safe, and discards prior history/care-team/family/PCC linkage.
- **Reactivate-existing:** on re-add, restore the prior soft-deleted record in place.

## Decision

Use **reactivate-existing**. On re-admission, match the prior soft-deleted record by `{facilityId, countryCode, phone}` (equivalently `cName`) and:
- **clear `deletedAt`** (un-hide it),
- set `status` back to `Active` (or the value supplied on re-add),
- re-create / re-enable the phone-based Cognito user,

keeping **exactly one** row per phone. Prior `status` history, care-team mappings, family members, and PCC linkage are retained. Apply the same `deletedAt`-clearing rule to the **PCC `patient.readmit`** path so the convergent-state model (ADR-001) is complete: whenever the derived status is not `Discharged`, clear `deletedAt` in the same `$set`.

## Consequences

**Positive**
- **Single row per `cName`** — the live+dead pair can no longer arise via re-add, so the ~15 unfiltered identity lookups (`SL-TD-09`) stay correct without touching each call site.
- **Smaller blast radius** — one focused change to the create/delete/readmit paths vs. editing ~15 lookups.
- **History preserved** — care-team, family, PCC linkage, and status timeline survive the discharge→readmit round-trip (a benefit for a returning resident).
- **Manual and PCC paths converge** on the same record-level rule.

**Negative / follow-ups**
- **`SL-TD-09` is mitigated, not closed.** No partial-unique index still means a second writer could create a duplicate. Recommended hardening: add a **partial unique index** on `{facilityId, countryCode, phone}` where `deletedAt: null` (precedent: `Conversation.directPairKey` partial unique index; `RehabTherapy` code partial index excluding soft-deleted rows — data-schema §2.1, REHAB-FR-12), and/or add `deletedAt: null` to identity lookups as defense in depth.
- **Cognito re-enable semantics** must be decided in build: re-create the user vs. re-enable a disabled user, and how the temporary-password / credential flow behaves on reactivation.
- **Delete-dialog copy** ("cannot be undone") becomes accurate to fix — re-admission is now supported.
- **Edge case:** a re-add for a phone whose prior record is **not** soft-deleted (i.e. an active duplicate) must still be rejected (the existing Cognito `409` path) — reactivation applies only when a soft-deleted match exists.

## UAT impact

- **RES-AC-12** (manual re-admit → reactivate) is added to Feature 03; **PENDING BUILD**, design now / run after the build lands. Scenarios RES-TS-50/51/52 replace the previously-excluded `RES-TS-EX-01`.
- The PCC `readmit` `deletedAt` fix is testable only once PCC connectivity is configured (**DC-08**).
