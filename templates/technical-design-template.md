# Technical Design Document Template

Standard engineering design-doc shape (the pattern used by Google-style design docs,
RFCs, and ADRs): state the problem and goals before the design, document
alternatives and why they were rejected (not just the chosen approach), and
separate "what we're building" from "how we'll know it worked" and "what could go
wrong."

**Save as**: `TD-<slug>.md`, in `03_architecture/{features,enhancements,bugs}/<slug>/`.

```markdown
# Technical Design: <feature/enhancement/bug slug>

| Field | Value |
|---|---|
| Source PRD | <path> |
| Author | System Architect |
| Status | Draft / Ready for review / Approved |
| Reviewers | |
| Product | Core / SAL / SNF |
| Apps/surfaces affected | Carried forward from the source PRD/ER's own field — don't re-derive from scratch; flag it to Product Manager if the design work reveals the PRD/ER's field was incomplete or wrong |

## Revision history
Populated once a TD is revised after `status: approved` — most often triggered by
a dev-team question raised outside this document (chat, email, a grooming
session), same as the PRD's own Revision History table. Capture *why* a change
happened, not just that it did — git/commit history already answers "what
changed and when"; this table exists specifically to answer "what question or
session prompted it," which commit history alone doesn't carry.
`tech-spec-<slug>.md` re-derives itself whenever a new row lands here.

| Date | Triggered by | What changed | Changed by |
|---|---|---|---|

## 1. Context and problem statement
What the PRD asks for, restated in technical terms. Why this needs a design doc at
all — what's non-trivial about it.

## 2. Goals and non-goals
### 2.1 Goals
### 2.2 Non-goals
Explicit non-goals prevent scope creep during implementation as much as the PRD's
own out-of-scope section does — restate anything technically relevant even if it
duplicates the PRD.

## 3. Proposed design
Architecture and components affected, data flow, sequence of operations. Use a
diagram (Mermaid, if the tooling supports it) wherever a sequence or component
relationship is easier to see than to read.

## 4. Alternatives considered
| Alternative | Why rejected |
|---|---|

Include at least one alternative even if the chosen approach seems obvious — the
rejected options are what let a future reader know the tradeoff was actually
considered, not assumed. A "Why rejected" cell stacking more than one distinct
reason is a bullet list, not a run-on paragraph — see `PROCESS-WALKTHROUGH.md`'s
Key conventions cheat-sheet → Table-cell formatting.

## 5. Data model changes
Schema changes, migrations required, backward-compatibility considerations for
existing data.

## 6. API / interface changes
New or modified endpoints, contracts, integration touchpoints with whichever
upstream/downstream systems this project actually integrates with — see the
project-specific config for the current list rather than assuming a generic set.
Where an endpoint introduces new authorization logic (a new self-vs-admin check, a
new role gate), sketch it against the codebase's actual existing middleware/guard
pattern rather than generic pseudocode — a reviewer should be able to tell whether
this reuses an existing pattern or introduces a new one.

## 7. Non-functional considerations
Performance, scalability, security, accessibility, compliance (HIPAA/SOC2/GDPR as
applicable), audit/data-integrity requirements. State "not applicable" explicitly
rather than omitting a row — an omitted row reads as "not considered."

## 8. Testing strategy
What level of testing this needs (unit/integration/end-to-end), and anything that
needs a specific test environment or data setup.

## 9. Rollout and migration plan
### 9.1 Phased rollout
Stages this ships in (e.g. internal → pilot facility/tenant → general
availability), each with a concrete exit criterion — what must be true, observed,
or confirmed before advancing — not just phase names. Name what gates each stage:
an existing mechanism (a feature-page gate, an access flag already in the
platform) or a new one this design has to introduce — don't assume a gate exists
without checking.

### 9.2 Data migration
Migration steps and their order, backward compatibility during the transition
window, and an explicit rollback plan — including the point past which rollback
stops being clean (e.g. once real user data has been written under the new shape).

### 9.3 Observability
What signals this feature introduces that don't already have monitoring, and what
threshold on each would justify an alert. Scope this to what the feature actually
adds — a cache hit rate, a scheduled job's tick duration approaching its own
interval, a migration's before/after record count — not a generic monitoring
boilerplate list. If a new notification path bypasses the platform's normal
history/audit trail (as an explicit PRD requirement sometimes asks for), note how
a "was X notified?" question would still be answerable without it.

## 10. Risks and mitigations
| Risk | Likelihood/Impact | Mitigation |
|---|---|---|

A "Mitigation" (or "Likelihood/Impact") cell covering more than one distinct
point is a bullet list, not a dense paragraph — same table-cell formatting rule
as §4, see `PROCESS-WALKTHROUGH.md`'s Key conventions cheat-sheet →
Table-cell formatting.

## 11. Open questions
| ID | Question | Current position | Priority |
|---|---|---|---|

Same table-cell formatting rule as §4 and §10 — a "Current position" covering
more than one fact is a bullet list, not a paragraph.
```
