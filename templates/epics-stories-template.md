# Epics / Stories / Tasks Template

Standard agile shape: an Epic groups Stories, each Story is written in
user-story form with Given/When/Then acceptance criteria. Tasks are non-user-facing
technical work added by System Architect, not Product Manager (see
`skill-sa-discipline.md`) — attached at whichever level they actually belong:
under a specific Story if the work is part of delivering that story, or at the
**Epic level** if it's foundational work the epic depends on but no single story
owns. INVEST is the standard checklist for whether a story is actually ready to
build.

```markdown
# Epic: <name>

| Field | Value |
|---|---|
| Source document | <path to the PRD / Enhancement Request / Bug Report> |
| Product | Core / SAL / SNF |
| Status | Draft / Ready for review / Approved |

## Epic summary
One or two sentences — the outcome this epic delivers.

## Epic-level functional spikes (added by Product Manager)
Some open questions can't be resolved by a decision in conversation — they need
actual investigative work first (a short user-validation session, a compliance
check, a data audit to size a problem). When an open question blocks story
readiness this way, raise it here as a bounded task rather than leaving it stuck as
a passive row in Open Questions with no path to resolution.
- [PM] <spike> — <what it needs to produce, e.g. "confirm with 3 SNF nurses which
  sequence they expect" — and which open question / story it unblocks>

## Epic-level technical tasks (added by System Architect)
Foundational work the epic's stories depend on, that isn't owned by any single
story — e.g. standing up a new service before any story can start, a schema
migration all stories rely on, a spike/research task to settle an open technical
question before stories can be estimated.
- [SA] <task> — <why it's epic-level rather than belonging to one story>

---

### Story <ID>: <short title>

**As a** <persona>, **I want** <capability>, **so that** <benefit>.

**Acceptance criteria** (Given/When/Then — one row per scenario, include edge
cases, not just the happy path):
| Given | When | Then |
|---|---|---|

| Field | Value |
|---|---|
| Prototype page | <prototype_page tag, if source is prototype-first> |
| Estimate | <story points or T-shirt size> |
| Priority | |
| tracker_id | <set by Project Manager persona after tracker creation — see the ID link-back rule> |

**Technical tasks** (added by System Architect, scoped to this specific story —
kept visually distinct from the Product Manager's story content above):
- [SA] <technical task, e.g. migration script, index addition, non-functional work
  this specific story implies but doesn't state>

---

### Story <ID+1>: <short title>
(repeat the block above per story)

---

## INVEST check
Before marking an epic's stories `status: ready`, check each one:
- **I**ndependent — doesn't require another unfinished story to be testable
- **N**egotiable — describes outcome, not a rigid implementation
- **V**aluable — delivers something a user or the business actually cares about
- **E**stimable — enough is known to size it
- **S**mall — fits within a sprint
- **T**estable — the acceptance criteria are concrete enough to verify

A story failing one of these should be split, clarified, or flagged — not passed
along oversized or ambiguous and left for System Architect or Project Manager to
untangle later.

**Spikes (functional or technical) are exempt from INVEST** — their output is an
answer or a decision, not user-facing behavior, so "Valuable" and "Testable" in the
story sense don't apply the same way. A spike's own bar is simpler: does it have a
bounded scope and a stated thing it needs to produce.
```

