# Bug Report Template

**Save as**: `bug-report-<slug>.md`, in `02_prd/bugs/<slug>/` — never a bare `BR.md`.

```markdown
# Bug: <short name>

| Field | Value |
|---|---|
| Product / environment | SAL / SNF, and prod / staging / other |
| Status | Draft / Ready for review / Approved |
| Reported by | |
| Severity | Blocker / High / Medium / Low |
| Frequency | Always / Intermittent / Once observed |
| repo_status | not-promoted / promoted — set by whoever runs the promotion step, not by this persona |
| last_promoted_revision | Timestamp/version last pushed to the code repo, if promoted. If this is older than the document's last-modified time, a re-promotion is due. |

## 1. Summary
One or two sentences: what's wrong.

## 2. Steps to reproduce
Numbered, specific enough that someone else could follow them exactly.

## 3. Expected behavior

## 4. Actual behavior

## 5. Impact
Who/how many are affected, and how (blocked entirely, degraded, cosmetic).

## 6. Suspected root cause
If any — otherwise state "Not yet determined."

## 7. Notes to System Architect
The specific technical question this persona can't answer alone — e.g. "is this a
data model issue or a UI issue?"

## 8. Supporting evidence
Error messages, screenshots, log excerpts, if available.

## Revision history
Populated once this document is revised after `status: approved`. Capture *why* a
change happened, not just that it did.

| Date | Triggered by | What changed | Changed by |
|---|---|---|---|
| | e.g. "dev-team question via Slack — clarify X" | | |
```
