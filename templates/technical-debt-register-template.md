# Technical Debt Register Template

Standard practice treats technical debt as a running, prioritized register (like a
backlog) rather than one-off documents — most items need only a row; a few
significant items warrant their own write-up, linked from the register rather than
replacing it.

## Register (one running file per product folder)

```markdown
# Technical Debt Register — <Product: Core / SAL / SNF>

| ID | Description | Location/component | Type | Impact if unaddressed | Effort | Priority | Status | Related PRD/Epic/TD | Logged | Resolved |
|---|---|---|---|---|---|---|---|---|---|---|
| TD-01 | | | Architecture / Code quality / Missing tests / Outdated dependency / Manual process | | S/M/L | Blocker/High/Medium/Low | Open / Planned / In progress / Resolved / Won't fix | | <date> | |
```

**Type** — pick the closest fit; add new categories sparingly and keep them
consistent across entries once introduced.

**Impact if unaddressed** — be specific about the actual consequence (e.g. "blocks
horizontal scaling past N facilities," "silent data loss risk on concurrent edits"),
not just a severity label — the severity label alone doesn't help someone
prioritize between two "High" items.

**Won't fix** is a legitimate status, not a failure to close the loop — record the
reason so the decision doesn't get silently revisited.

## Detailed write-up (for significant items only)

```markdown
# Technical Debt: <TD-ID> — <short name>

| Field | Value |
|---|---|
| Register entry | TD-<ID> |
| Product | Core / SAL / SNF |

## What it is
Concrete description of the debt — not just "the code is messy," but what
specifically is fragile, duplicated, or missing.

## How it got here
Brief context — a pivot, a deadline tradeoff, an integration bolted on. This isn't
about assigning blame; it's what tells a future reader whether the same shortcut is
about to happen again elsewhere.

## What it's costing
Concretely — slower feature delivery in a specific area, a recurring class of bugs,
onboarding friction for new engineers, a scaling ceiling.

## What fixing it would take
Rough shape of the work, not a full technical design — promote to a Technical
Design Document if this gets prioritized and actually scheduled.
```
