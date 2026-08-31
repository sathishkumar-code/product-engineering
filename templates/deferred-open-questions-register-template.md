# Deferred Open Questions Register Template

Fallback tracker only. Most **Deferred** Open Question dispositions route to
Technical Debt or a new Enhancement Request instead (see
`_reference/PROCESS-WALKTHROUGH.md`'s "Open Question lifecycle and the
development-readiness gate"). Use this register only when a deferred item is
genuinely neither: a decision or a piece of evidence that's temporarily
unavailable and needs revisiting later, not tracked work in its own right.

## Register (one running file per product folder)

```markdown
# Deferred Open Questions Register — <Product: Core / SAL / SNF>

| ID | Question | Why deferred | Revisit trigger | Owner | Status | Source document | Logged | Resolved |
|---|---|---|---|---|---|---|---|---|
| DOQ-01 | | | | | Open / Resolved | | <date> | |
```

**Question** — the original Open Question text, unabridged, not a
paraphrase. Someone revisiting this later shouldn't have to dig up the
source document to know what was actually being asked.

**Why deferred** — Sathish's stated reason for deferring rather than
resolving now (e.g. "waiting on the PCC integration contract renewal,"
"needs production usage data not yet available").

**Revisit trigger** — the concrete condition that means it's time to look
again — a date, an event, a dependency landing — not just "later."

**Owner** — who's responsible for noticing the trigger and bringing the
question back for a decision; defaults to Sathish if no one else is a
better fit.

**Status** — `Open` until the question is actually answered; then
`Resolved`. A Deferred Open Question row being marked `Dispositioned` in its
source PRD/ER/TD closes the *question*, not this entry — this register entry
stays `Open` until the deferred question itself is finally answered.

**Source document** — the PRD/ER/TD (with slug) and Open Question row this
entry traces back to, so the two stay linked in both directions.

**Resolved** — when the deferred question is finally answered, record the
date and how it was resolved (a one-line note is enough) rather than
deleting the row — the register is a running log, not a to-do list that
gets emptied.
