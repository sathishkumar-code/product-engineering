# Architecture Decision Records — Senior Living

Each ADR captures one material architectural decision in the senior-care platform, reverse-engineered from the as-built code (the only source of truth) and the PRD/architecture v1.1 refresh (2026-06-17, production HEAD `981704e9`).

| ADR | Title | Status |
|---|---|---|
| [ADR-001](./ADR-001-pcc-webhook-convergent-state.md) | PCC webhooks use a convergent-state (re-pull) model rather than trusting the event payload | Accepted (as-built) |
| [ADR-002](./ADR-002-custom-otp-password-reset.md) | Custom SMS OTP password reset on top of Cognito (vs Cognito-native forgot-password) | Accepted (as-built) |
| [ADR-003](./ADR-003-chat-dual-channel-push.md) | Chat push uses a dual-channel split (FCM offline-only + Web Push to all) with in-memory presence | Accepted (as-built), with scaling caveat — **amended 2026-07-12** (admin PWA `/messages`-scope + `BroadcastChannel` decisions) |
| [ADR-004](./ADR-004-resident-readmission-reactivate.md) | Resident re-admission reactivates the existing soft-deleted record (clear `deletedAt`) rather than inserting a new row | Accepted (decision — build pending) |
| [ADR-005](./ADR-005-facility-timezone-authority.md) | Facility timezone authority for client-side date/time rendering — should facility-local time (not device-local) be authoritative across all clients? | **Proposed** (2026-07-12) |
| [ADR-006](./ADR-006-digital-signature-contract.md) | Digital-signature ("Pending Sign") document contract and audit posture | **Proposed** (2026-07-12) |

> Most ADRs here describe decisions **already embodied in the code**. "Accepted (as-built)" means the decision is in production; the *Consequences* sections flag the trade-offs and the follow-ups a forward roadmap should address. ADR-004 is the first **forward** decision — "Accepted (decision — build pending)" means the direction is agreed but the code change has not yet landed (tracked as `SL-TD-07`). ADR-005 and ADR-006 are **recommend-only** — they surface a now-real inconsistency/gap for human review; no direction has been chosen yet.
