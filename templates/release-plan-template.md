# Release Plan Template

```markdown
# Release: <release name/version>

| Field | Value |
|---|---|
| Target date | |
| Scope (SAL / SNF / Shared) | |
| Objective / theme | The one or two things this release is actually for — tie back to the roadmap theme it serves |
| Status | Draft / Ready for review / Approved / In progress / Shipped |
| repo_status | not-promoted / promoted — same convention as the PRD template |
| last_promoted_revision | Timestamp/version last pushed to the code repo, if promoted |

## 1. Included
Features, enhancements, and bugs going into this release — by slug/path, not
restated content. Group by type (features / enhancements / bugs).

## 2. Explicitly excluded
Anything that was considered for this release and deliberately pushed out. This
matters as much as what's included — it's what stops "why isn't X in this release"
from being an open question three weeks in.

## 3. Success criteria
How you'll know this release did what it was for. Should trace back to the
objective/theme above and, where applicable, to the success-metrics field on the
included PRDs.

## 4. Dependencies and risks
Anything outside this team's control this release depends on (upstream integration
availability, another team's work, an open PRD question that's currently marked
non-blocking but could become one). Call out open PRD questions specifically if
their "current position" assumption turns out wrong.

## 5. Rollout plan
Phased (e.g. by facility, by percentage), feature-flagged, or all-at-once — and why.
For healthcare/clinical features specifically, state whether any facility is a pilot
before general availability.

## 6. Rollback plan
What happens if something in this release needs to be pulled after ship. Not
optional for anything touching clinical data or signed/immutable records.

## 7. Sprints
Link to `sprints/<sprint-slug>/sprint-plan.md` for each sprint under this release —
the Project Manager persona owns sprint breakdown from this plan, not the reverse.
```
