# Compliance Register Template

Same shape as the Technical Debt Register: a running register for most entries,
with a detailed write-up only for significant gaps. A compliance gap and a
technical debt item are structurally the same kind of thing — something known to
be short of a standard, tracked with a status until it's closed — just against a
regulatory control instead of a code quality bar.

## Register (one running file per product/entity in scope)

```markdown
# <Framework> Compliance Register — <Product/Entity>

| ID | Requirement/Control | Description | Gap (if any) | Status | Owner | Priority | Related PRD/Epic/TD | Logged | Target date | Resolved |
|---|---|---|---|---|---|---|---|---|---|---|
| C-01 | | | | Not yet assessed / Compliant / Gap identified / Remediation planned / Remediation in progress / Closed | | | | <date> | | |
```

**Requirement/Control** — cite the specific clause or control ID from the actual
framework document, not a paraphrase — a paraphrase drifts from the source over
time and becomes hard to re-verify against an audit or updated framework version.

**Gap** — be specific about what's actually missing or non-conforming, not just
that a gap exists. "Not yet assessed" is a legitimate status for entries logged
before review has happened — don't guess a gap to fill the field.

## Detailed write-up (for significant gaps only)

```markdown
# Compliance Gap: <C-ID> — <short name>

| Field | Value |
|---|---|
| Register entry | C-<ID> |
| Framework | |
| Requirement/Control | |

## What the requirement actually requires
Cite the control, don't paraphrase from memory of what it "probably means."

## What's missing today
Concrete, verified against current as-built behavior — not assumed from the
requirement's absence in a PRD.

## Risk if unaddressed
Concrete consequence, not just a severity label.

## What closing it would take
Rough shape of the work — promote to a Technical Design Document if this gets
prioritized and actually scheduled.
```
