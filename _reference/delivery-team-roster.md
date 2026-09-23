---
title: Delivery Team Roster — Shashi Care
type: reference
status: Active — roster populated and authoritative
last_updated: 2026-09-20
---

# Delivery Team Roster — Shashi Care

> Instantiated from `framework/templates/team-structure-template.md` (§1
> Roster), adapted for the **human delivery team** — the engineers, QA staff,
> and team leads who build and test Core / SAL / SNF — as distinct from
> `_reference/team-structure.md`, which covers the **AI-persona** roster and
> process RACI (Product Manager, System Architect, Project Manager, Process
> Architect, Developer, QA Engineer, DevOps Engineer personas and their
> hosting). See "Relationship to `team-structure.md`" below for the boundary
> between the two documents.
>
> Cross-product (Core / SAL / SNF), not duplicated per product-docs repo —
> same non-duplication principle `team-structure.md` already follows, and
> confirmed by Sathish for this artifact: the roster is cross-product because
> the Web/Admin Dev Team already spans SAL + SNF today (shared
> `senior_living_backend` / `senior_living_admin` codebase — see
> `_agent-instructions/shashi-care-developer-config.md`'s "Repos and product
> mapping" table).
>
> **Status: populated and authoritative.** The Roster and Team Leads tables
> below reflect the actual delivery-team information supplied by Sathish and
> are the authoritative source PjM consumes for the Sprint Sheet's Admin Dev
> Team / Mobile Dev Team fields — see "Governance" below for the change
> process going forward.

## Teams

Team boundaries mirror the repo/product boundaries already established in
`_agent-instructions/shashi-care-developer-config.md`'s "Repos and product
mapping" table — this file does not invent new team boundaries independent
of that source.

| Team | Repos | Products |
|---|---|---|
| Web/Admin Dev Team | `senior_living_backend`, `senior_living_admin` | SAL + SNF (combined, shared codebase) |
| Mobile Dev Team | `senior_living_reactnative`, `senior_living_skillednursing_resident`, `senior_living_staffapp`, `senior_living_tvapp` | SAL (`senior_living_reactnative`, `senior_living_tvapp`), SNF (`senior_living_skillednursing_resident`, `senior_living_staffapp`) |
| QA | (one QA Engineer instance per code repo above — see `shashi-care-qa-config.md`) | Core / SAL / SNF, per repo assigned |

No other team is added here — the Developer/QA/DevOps configs define no
further team boundary today. If a Core-specific team, a DevOps team, or any
other delivery team is stood up later, add it here only once an authoritative
source (config file or Sathish's direct instruction) defines it — do not
infer one.

## Roster

| Name    | Team        | Role             | Team Lead? | Products (Core / SAL / SNF) | Contact |
| ------- | ----------- | ---------------- | ---------- | --------------------------- | ------- |
| Rajan   | Shashi Care | Team Lead        | Yes        | Core / SAL / SNF            |         |
| Anish   | Shashi Care | Web Developer    | No         | Core / SAL / SNF            |         |
| Ronak   | Shashi Care | Web Developer    | No         | Core / SAL / SNF            |         |
| Vikas   | Shashi Care | Web Developer    | No         | Core / SAL / SNF            |         |
| Kapil   | Shashi Care | Mobile Developer | No         | Core / SAL / SNF            |         |
| Kalpesh | Shashi Care | Mobile Developer | No         | Core / SAL / SNF            |         |
| Payal   | Shashi Care | QA Lead          | No         | Core / SAL / SNF            |         |
| Gautam  | Shashi Care | UX Designer      | No         | Core / SAL / SNF            |         |

No names, roles, or contacts are invented. Sathish is the sole source for
this table's content — see "Governance" below.

## Team Leads

Explicit designation only — never inferred from seniority, title, or tenure.
PjM reads this table (not the Roster table's free-text Role column) to
determine the **default** Admin Dev Team / Mobile Dev Team assignee for the
Sprint Sheet's corresponding fields, overridable per sprint by PjM as normal
sprint-planning judgment.

| Team | Team Lead | Since | Notes |
|---|---|---|---|
| Web/Admin Dev Team | Rajan | | |
| Mobile Dev Team | Rajan | | |
| QA | Payal | | |

## Governance

- **Content authority**: Sathish. Team membership, role, and Team Lead
  designation are Sathish's call — not delegated to PM, SA, PjM, or any
  individual Team Lead.
- **Maintenance**: Process Architect authors and edits this file on the
  team's behalf, same as `_reference/team-structure.md` — no other persona
  edits it directly.
- **Change process**: every roster change (membership, role, or Team Lead
  designation) requires Sathish's confirmation before Process Architect
  applies it. There is currently **no self-reporting exception** — a Team
  Lead reporting a change to their own team still routes through Sathish
  first, the same as any other change. (This may be revisited later, but only
  on Sathish's explicit instruction — not assumed by default.)
- **Consumption**: Project Manager (PjM) reads this file directly — via the
  same "construct the path, don't search" pattern PjM already uses for
  doc-tree artifacts — to populate the Sprint Sheet's `Admin Dev Team` /
  `Mobile Dev Team` selectable values and default Team Lead pre-selection.
  PjM does not maintain its own copy of the roster, and does not invent
  selectable values independently of this file. If PjM needs a name, role,
  or Team Lead designation this file doesn't yet have, that's a handback to
  Sathish (via Process Architect for the actual edit) — the same shape as any
  other upstream-gap handback in `framework/_agent-instructions/skill-pjm-discipline.md`.
- Product Manager, System Architect, and other personas needing to know who's
  on a delivery team read this file directly rather than maintaining their
  own copy, same principle.

## Relationship to `team-structure.md`

| | `_reference/team-structure.md` | `_reference/delivery-team-roster.md` (this file) |
|---|---|---|
| Covers | The 7 AI personas (PM, SA, PjM, Process Architect, Developer, QA Engineer, DevOps Engineer), their hosting, and process RACI | The human delivery team building/testing the product (Web/Admin Dev, Mobile Dev, QA staff, Team Leads) |
| Changes when | The pipeline/process itself changes (rare, governed as a structural ADLC change) | Staffing changes (membership, role, Team Lead) — expected to be more frequent |
| Content authority | Sathish | Sathish |
| Maintained by | Process Architect | Process Architect |

Kept as two separate files rather than merged sections of one document
because they have different subjects and different change cadences — bundling
frequent operational roster edits with rare structural process edits would
put both under one edit history for no governance benefit. Each file
cross-references the other; neither duplicates the other's content.

## Coverage / on-call

Not yet applicable — see `_reference/team-structure.md`'s "Coverage /
on-call" section for the equivalent note on the AI-persona side. Revisit
if/when an on-call rotation is established for the delivery team above.
