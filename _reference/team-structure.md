---
title: Team Structure — Product Engineering Team
type: reference
status: Active
last_updated: 2026-08-29
---

# Team Structure — Product Engineering Team

> Instantiated from `templates/team-structure-template.md`. This is the real, filled roster and RACI for the Shashi Care product engineering team, covering all 7 personas across both hosting systems: the Cowork persona-chat pipeline (Process Architect only) and the Hermes WSL orchestrator (Product Manager, System Architect, Project Manager — one shared instance each; Developer, QA Engineer, DevOps Engineer — one instance per code repo). **Moved 2026-08-29**: Product Manager, System Architect, and Project Manager ran in Cowork until this date — their Cowork projects are kept as a dormant fallback (not deleted) but are no longer part of the active pipeline. Developer/QA/DevOps read `shashi-care-docs` directly; Product Manager/System Architect/Project Manager instead read a manually-synced mirror of it in `product-engineering/` — see `_agent-instructions/shashi-care-process-architect-config.md`'s "Hermes copy sync convention" for why and how that mirror is kept current. See `_reference/PROCESS-WALKTHROUGH.md` for the full pipeline and that same config file for the governance boundary between the two systems.

## Roster

| Role | Hosted in | Notes |
|---|---|---|
| Product Owner | — | Sathish. Final authority — see RACI below. |
| Product Manager (PM) | Hermes | `skill-pm-discipline.md` + `shashi-care-pm-config.md`. One shared instance (not per-repo). Moved from Cowork 2026-08-29 — reads `shashi-care-docs` via the manually-synced `product-engineering/` mirror, not a live read. |
| System Architect (SA) | Hermes | `skill-sa-discipline.md` + `shashi-care-sa-config.md`. One shared instance (not per-repo). Moved from Cowork 2026-08-29 — reads `shashi-care-docs` via the manually-synced `product-engineering/` mirror, not a live read. |
| Project Manager (PjM) | Hermes | `skill-pjm-discipline.md` + `shashi-care-pjm-config.md`. One shared instance (not per-repo). Moved from Cowork 2026-08-29 — reads `shashi-care-docs` via the manually-synced `product-engineering/` mirror, not a live read. |
| Process Architect (PA) | Cowork | `skill-process-architect-discipline.md` + `shashi-care-process-architect-config.md`. Sole author of `_agent-instructions/`, `templates/`, `_reference/` for both systems — see Section 10 / "Known incidents" in its config. **Only persona still hosted in Cowork as of 2026-08-29.** |
| Developer | Hermes (WSL) | `skill-developer-discipline.md` + `shashi-care-developer-config.md`. One instance per code repo. |
| QA Engineer | Hermes (WSL) | `skill-qa-discipline.md` + `shashi-care-qa-config.md`. One instance per code repo. |
| DevOps Engineer | Hermes (WSL) | `skill-devops-discipline.md` + `shashi-care-devops-config.md`. One instance per code repo. |

## RACI

R = Responsible, A = Accountable, C = Consulted, I = Informed. Exactly one unconditional A per row.

| # | Decision / process | PM | SA | PjM | Process Architect | Developer | QA Engineer | DevOps Engineer | Sathish |
|---|---|---|---|---|---|---|---|---|---|
| 1 | PRD/ER/BR approval | R | C | I | I | I | I | I | A |
| 2 | SA technical review verdict | I | R/A | I | I | C | I | I | I |
| 3 | SA verdict escalation (3-round unsettled) | C | C | I | I | I | I | I | R/A |
| 4 | Technical Design approval | C | R | I | I | C | I | I | A |
| 5 | Epic/Story creation in tracker | C | C | R/A | I | I | I | I | I |
| 6 | Sprint scope commitment | I | I | R | I | C | C | I | A |
| 7 | Code implementation | I | C | I | I | R/A | I | I | I |
| 8 | Code merge to main | I | C | I | I | R | I | I | A |
| 9 | Test scenario/case authorship | R/A | C (technical) | I | I | I | C | I | I |
| 10 | Test execution & QA sign-off | I | I | I | I | C | R/A | I | I |
| 11 | Bug Report filed from QA/prod | C | I | I | I | C | R/A | C | I |
| 12 | Release plan drafting | R | C | I | I | I | I | C | A |
| 13 | Release go/no-go | C | C | R | I | C | C | C | A |
| 14 | Production promotion (incl. blocked-gap check) | I | C | I | I | I | I | R | A |
| 15 | Production incident response | I | R (design fix) | I | I | R (fix) | C | R (mitigate) | A |
| 16 | Pipeline/process definition change | C | C | C | R (proposes only) | I | I | I | A (Process Architect executes the actual edit) |

## Escalation path

| From | To | When |
|---|---|---|
| PM ↔ SA disagreement, unresolved after 3 review rounds | Sathish | Row 3 above |
| Developer/QA/DevOps finds a design, scope, or behavior gap | PM or SA (per `skill-*-discipline.md` return-path rules) | See each persona's "Deviation/return path" section |
| Any persona proposing a change to the pipeline itself | Process Architect | Process Architect is the sole executor of the edit; the proposing persona never edits `_agent-instructions/`, `templates/`, or `_reference/` directly |
| Release-blocking gap found at promotion time | Sathish (override required to proceed) | Row 14 above; see `skill-devops-discipline.md` |
| PHI-affecting rollback | Sathish (approval required before executing) | See `skill-devops-discipline.md` |

## Coverage / on-call

Not yet applicable — single-operator team at this stage. Revisit when the team grows beyond Sathish plus AI personas.
