# SA Review Comments Template

Formalizes the two-pass review System Architect runs per slug: once against the
source document — a PRD (feature), Enhancement Request, or Bug Report, whichever
applies — before Epics/Stories exist, once against the actual Epics/Stories once
Product Manager generates them. One running file per slug — both passes append to
it rather than creating separate files, so the full review history for a slug
stays in one place.

**Save as**: `SA-comments-<slug>.md`, in that same per-slug
`03_architecture/{features,enhancements,bugs}/<slug>/` folder — covers the source
document review, the Epics/Stories review, and the Technical Design review below,
all in this one file.

```markdown
# SA Review: <category>-<slug>

| Field | Value |
|---|---|
| Source document | <path> (prd-<slug>.md / enhancement-request-<slug>.md / bug-report-<slug>.md) |
| Product | Core / SAL / SNF |
| review_round | <count, this document-stage round — see escalation rule below> |
| Verdict | In review / Approved-as-is / Approved-with-changes / Blocked-escalate |

## Source Document Review (Round <N>)
Technical feedback on the PRD/ER/BR itself, referencing the specific section by
name. Applies with the same rigor regardless of category — an enhancement or bug
gets the same review discipline as a feature-scale PRD, not a lighter pass just
because the document is shorter.

### Recommended technical epics/stories/spikes
Raised now, before Epics/Stories formally exist, so Product Manager can fold these
in directly when drafting — avoids a second round trip purely to add what SA
already knew was needed.
- <recommended technical story/task/spike, and why>

## Epics/Stories Review (Round <N>)
Runs only once Product Manager has actually generated `epics-stories.md`. Confirms
the recommendations above got incorporated, and reviews the functional stories
themselves.

## Technical Design Review
Separate from the two sections above — this reviews the Technical Design itself,
whichever pathway produced it (SA-authored directly, or team-submitted via a
GitLab `architecture-submissions/` folder). Not tied to the source-document/
Epics-Stories round count; a TD can be reviewed at any point once it exists.

| Field | Value |
|---|---|
| Source | SA-authored / Team-submitted |
| Submission path (if team-submitted) | `<repo>/architecture-submissions/<category>-<slug>/...` |
| Original format (if team-submitted) | e.g. Word, PDF, Confluence export |
| Convert to standard template? | Asked Sathish: yes / no / pending |
| Verdict | In review / Approved-as-is / Approved-with-changes / Blocked |

### Findings
Technical feedback, referencing the specific section of the submitted design by
name — same discipline as source-document/Epics-Stories review: comment here,
don't edit the submission itself, regardless of its format.

## Escalation
**3 review rounds is the threshold.** If a source document (PRD, ER, or BR) is
still bouncing between Product Manager and System Architect after 3 rounds
without landing on Approved-as-is or Approved-with-changes, stop iterating and
escalate to Sathish for a direct decision rather than continuing to bounce it.
This is a manual discipline right now, not an automated check — the
`review_round` field exists so it's easy to see the count at a glance. The
Technical Design Review section above isn't subject to this same round count — a
TD that needs more than one pass just gets more entries under Findings, dated,
until it reaches a verdict.
```
