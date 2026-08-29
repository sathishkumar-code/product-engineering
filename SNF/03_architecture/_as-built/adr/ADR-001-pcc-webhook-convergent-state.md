# ADR-001 — PCC webhooks use a convergent-state (re-pull) model

**Status:** Accepted (as-built, production HEAD `981704e9`)
**Date:** 2026-06-17 (documenting an existing decision)
**Area:** `senior_living_backend` — `src/integrations/pms/pcc/webhooks/`
**Related:** PRD `clinical-records.md` §3D (CLIN-FR-20/21), backend architecture "PCC patient-lifecycle integration"

## Context

PointClickCare (the SNF/LTC EHR) sends webhook events for patient lifecycle (`updateResidentInfo`, `discharge`, `readmit`, `cancelAdmit`, `cancelDischarge`, `transfer`, `cancelTransfer`) and medications. The event payload carries only identifiers (`patientId`, `orgUuid`, `eventType`) — not the new patient state. The platform must keep each resident's local `status`, `dischargeDate`, and `unitNo` in sync with PCC.

Two implementation models were available:
- **Trust-payload:** derive the new state from the event *type* (e.g. `discharge` → `status: Discharged`).
- **Re-pull / convergent-state:** on every event, query PCC for the authoritative current patient record and derive state locally.

## Decision

Use the **convergent-state model**. A flat registry (`PCC_EVENT_HANDLERS`) dispatches each event O(1); the controller ACKs `200 {received:true}` synchronously and runs the handler asynchronously. Every patient handler: resolves the resident (`pcc_patientId` + case-insensitive `pcc_orgUuid`), re-pulls the authoritative PCC record (`fetchPccPatientData`), derives `status`/`dischargeDate`/`unitNo` via `buildPatientStatus`/`buildUnitNo`, `$set`s them, and `$push`es the raw snapshot onto `Resident.pcc_patient_details`. `buildPatientStatus` is binary: `Discharged` iff PCC's `patientStatus === 'Discharged'`, else `Active`.

## Consequences

**Positive**
- **Self-healing / idempotent at the field level** — repeated or out-of-order events converge to whatever PCC currently reports; a missed event is corrected by the next one.
- **Event semantics need not be exhaustive** — `cancelDischarge`, `cancelTransfer`, etc. need no bespoke status logic; they all re-derive.
- Synchronous ACK prevents PCC retry storms.

**Negative / follow-ups**
- **Stale-read window:** between ACK and the PCC pull, PCC may advance; under rapid sequential events the later pull can overwrite intended state. No ordering/sequencing guard. *(CLIN-GAP-17)*
- **No idempotency key:** `messageId` is logged but never checked, so duplicate deliveries append duplicate `pcc_patient_details` snapshots (`$push`, no `$addToSet`/cap/TTL).
- **At-least-once with loss risk:** fire-and-forget handler after ACK means an event in flight is lost if the process dies; no durable queue or dead-letter.
- **`Transferred` is now dead-write** — the binary derivation never emits it though the enum retains it. *(CLIN-GAP-16)*
- **Convergent state omits `deletedAt`** — handlers `$set` `status`/`dischargeDate`/`unitNo` but never touch the soft-delete flag, so a `patient.readmit` on a soft-deleted resident sets `status: Active` yet leaves `deletedAt` set, producing a record that is "active" but hidden from every `deletedAt: null` listing. The convergent model is incomplete for the discharge↔readmit round-trip. **Resolution (2026-06-21, see ADR-004):** clear `deletedAt` whenever derived status ≠ `Discharged`. *(CLIN-GAP-18; SL-TD-07.)*
- **Security:** the endpoint is unauthenticated and logs full PHI; `pcc_patient_details` is unencrypted `Mixed` (medication data is KMS-encrypted, this snapshot is not). *(CLIN-GAP-15)*

**Recommended next steps:** add signature verification + a redacting logger; add `messageId` dedupe and a durable queue/dead-letter; decide whether to retain or drop `Transferred`; include `deletedAt` in the convergent `$set` so readmit un-hides a reactivated resident (ADR-004).
