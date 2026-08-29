# Sprint Plan & Status Template

Standard Scrum sprint-planning fields (goal, capacity, committed scope, burndown/
status, risks) combined with a running status section so one document serves both
planning and in-flight tracking — matching the PjM skill's rule to keep the local
sprint doc and the tracker in sync rather than maintaining two separate documents.

```markdown
# Sprint <N> — <Product: Core / SAL / SNF>

| Field | Value |
|---|---|
| Sprint dates | <start> – <end> |
| Sprint goal | One or two sentences — what this sprint exists to accomplish |
| Team capacity | <story points or person-days available this sprint> |
| Release | Link to the release-plan tab this sprint serves |

## 1. Committed scope
| Story/Task | Epic | Assignee | Estimate | Status | tracker_id |
|---|---|---|---|---|---|
| | | | | Backlog | |

Status values match the tracker binding: **Backlog → Development → Review → QA →
UAT → Done**, plus **Blocked** used alongside a stage rather than instead of it.
Don't invent a parallel status vocabulary here.

## 2. Capacity & carry-over
| Team member | Availability this sprint | Notes (PTO, partial allocation) |
|---|---|---|

Carried-over items from the previous sprint, listed separately with the reason they
didn't finish:
| Item | Reason not completed | Action for this sprint |
|---|---|---|

## 3. Status summary (update as the sprint progresses)
| Date | Points/items remaining | On track / At risk / Blocked | Notes |
|---|---|---|---|

Pull actual numbers from the tracker when updating this — don't estimate from
memory of what should be done by now.

## 4. Risks & blockers
| Risk/blocker | Impact | Owner | Mitigation |
|---|---|---|---|

## 5. Definition of Done (reference)
Link to or restate the team's Definition of Done so "complete" is judged
consistently across stories rather than per-story interpretation.
```
