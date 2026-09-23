---
title: Binding — Google Sprint Sheet (Shashi Care)
type: reference
status: Active — current operational tracker
last_updated: 2026-09-20
---

# Binding: Google Sprint Sheet — Shashi Care

Current-state tracker binding, parallel in role to
`_reference/shashi-care-clickup-binding.md`. This file defines the
**current operational source of truth for sprint execution** — the Google
Sprint Sheet (the `SNF-Redwood-Grove-Worklog` workbook and its Phase 4
operational structure).

## Status

- **Current operational tracker: the Google Sprint Sheet.** All sprint
  planning, sprint creation, sprint tracking, and day-to-day sprint-execution
  status for Shashi Care are recorded here, not in ClickUp.
- **ClickUp is a future-state work-management direction, not the current
  operational tracker.** See `_reference/shashi-care-clickup-binding.md`'s
  status header. No ClickUp migration is implied or authorized by this file.
- **Trello is a downstream visualization/management layer only.** It
  consumes from the Sprint Sheet for display/tracking purposes and is not
  a sprint-planning source of truth. PjM does not treat Trello state as
  authoritative when it diverges from the Sprint Sheet.

## Current implementation context

The existing `SNF-Redwood-Grove-Worklog` workbook, and its already-approved
Phase 4 operational structure, is the current implementation of this
binding. This file documents the approved current-state contract for that
structure; it does not introduce new behavior beyond what's already
approved.

## Ownership

PjM owns:
- Sprint planning
- Sprint creation
- Sprint tracking
- Maintenance of the current Sprint Sheet (the Phase 4 operational
  structure described below)

This mirrors, for the Sprint Sheet, the same ownership PjM already holds
for tracker creation/management in `framework/_agent-instructions/skill-pjm-discipline.md`
and `_reference/shashi-care-clickup-binding.md`'s "Access" section — the
Sprint Sheet is currently where that ownership is exercised.

## Phase 4 structural contract (approved)

The Sprint Sheet's columns, per the approved Phase 4 structure:

| Column | Notes |
|---|---|
| Week | |
| Component | Values governed by the product's existing component taxonomy — not duplicated here; see the relevant product/repo reference (e.g. `_agent-instructions/shashi-care-developer-config.md`'s repo/product mapping, `_reference/delivery-team-roster.md`'s Teams table). |
| Feature | |
| Acceptance criteria | |
| Priority | |
| Technical comments | |
| Rollout comments | |
| ETA | |
| Effort (man days) | Phase-4-specific extra column. |
| UI Design | |
| Spec ready | |
| Design ready | |
| Dev Status | |
| Admin Dev Team | Selectable values sourced from `_reference/delivery-team-roster.md` — see that file's "Governance" section; PjM does not maintain its own copy or invent values independently of it. |
| Mobile Dev Team | Same sourcing as Admin Dev Team, above. |
| QA Status | |
| Pre-prod | |
| Demo | |
| Prod | |
| Item ID | See "Item ID / Status ownership," below. |
| Status | See "Item ID / Status ownership," below. |

This file does not duplicate the contents of the component taxonomy or the
delivery-team roster — both remain canonical external references, read
directly by PjM at time of use, per the existing "construct the path,
don't search" pattern already established for this persona.

## Item ID / Status ownership and lifecycle

Item ID assignment and Status field lifecycle management for Sprint Sheet
rows are owned by PjM, per the already-approved current contract — this
mirrors, for the Sprint Sheet, PjM's existing exclusive tracker-item
ownership (creation, status/lifecycle progression, tracking) described
generically in `framework/_agent-instructions/skill-pjm-discipline.md` and
concretely for ClickUp in `_reference/shashi-care-clickup-binding.md`. No
new lifecycle states or ownership rules beyond what's already approved are
introduced by this file.

## Relationship to other bindings

| | This file (Sprint Sheet) | `shashi-care-clickup-binding.md` |
|---|---|---|
| Status | Current operational tracker | Future-state direction |
| Scope | Sprint planning, creation, tracking, execution | Target-state Epic/Story/Task tracker object model |
| Trello | Downstream visualization consumer of this file's data | N/A |

No ClickUp migration work, ClickUp architecture change, or ClickUp
activation is implied, scheduled, or authorized by this binding.
