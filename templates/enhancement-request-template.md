# Enhancement Request Template

For changes to an existing feature that don't warrant a full new PRD. If the change
is large enough to need its own personas/scope/NFRs section, it's a feature-scale
PRD, not an enhancement — use the PRD template instead.

**Save as**: `enhancement-request-<slug>.md`, in `02_prd/enhancements/<slug>/` —
never a bare `ER.md`.

```markdown
# Enhancement: <name>

| Field | Value |
|---|---|
| Product | SAL / SNF / SAL+SNF |
| Apps/surfaces affected | Every app/surface this touches (web, staff app, resident app, etc. — see the project config's canonical app list; name all, not just the primary one) |
| Status | Draft / Ready for review / Approved |
| Base feature | Path/link to the existing PRD or as-built doc this enhances |
| Requested by | |
| Business goal | Why this is worth doing now |
| repo_status | not-promoted / promoted — set by whoever runs the promotion step, not by this persona |
| last_promoted_revision | Timestamp/version last pushed to the code repo, if promoted. If this is older than the document's last-modified time, a re-promotion is due. |

## 1. Current behavior
What the system does today. Cite the as-built doc or existing PRD — don't describe
current behavior from memory or assumption.

## 2. Proposed change
What should change, specific enough to write stories from directly.

## 3. Scope
### 3.1 In scope
### 3.2 Out of scope

## 4. Impact
What existing stories, tests, or adjacent features this touches. Does the base PRD
need a corresponding update, or does this stand alone as a delta?

## 5. Notes to System Architect
The specific technical question(s), if any.

## 6. Open questions
Same table format as the PRD template (ID | Area | Question and current position |
Priority), including its table-cell formatting rule (a multi-fact "current
position" is a bullet list, not a paragraph — see `PROCESS-WALKTHROUGH.md`'s Key
conventions cheat-sheet → Table-cell formatting) if there are any; omit the
section entirely if there are none — don't leave an empty table.

## Revision history
Populated once this document is revised after `status: approved` — most often
triggered by a dev-team question raised outside this document (chat, email, a
grooming session). Capture *why* a change happened, not just that it did —
git/commit history already answers "what changed and when"; this table exists
specifically to answer "what question or session prompted it."

| Date | Triggered by | What changed | Changed by |
|---|---|---|---|
| | e.g. "dev-team question via Slack — clarify X" | | |
```
