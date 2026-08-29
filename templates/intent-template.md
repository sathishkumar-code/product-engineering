# Intent Template

Adapted from Anthropic's AI-Native SDLC playbook's `intent.md` concept — a fast,
human-readable capture of a raw idea, in the originator's own words, committed
*before* any PRD/Enhancement Request/Bug Report drafting begins. Not a
replacement for those documents — a cheap precursor that lets Sathish or Product
Manager decide something's worth the fuller work before committing to it, and
gives whichever pathway follows (Design-prototype-first or Direct intake) a
clear seed to build from.

```markdown
# Intent: <short name>

| Field | Value |
|---|---|
| Author | Whoever originated the idea |
| Status | Draft / Accepted / Superseded by PRD |
| Category | Feature / Enhancement / Bug |

## Problem
What can't be done today, who's affected — in the originator's own words, no
formal language required.

## Proposed outcome
What better looks like.

## Affected users and systems
Personas/roles involved, and which of Core/SAL/SNF this touches.

## Constraints
Anything ruled out up front.

## Open questions
Anything unresolved at this stage — expected to carry forward into the PRD, not
necessarily resolved here.
```

## How it's produced
Product Manager brainstorms with whoever has the idea — asking the scoping
questions an analyst would (scope, users, constraints, success) — drafts
`intent.md`, and the originator corrects anything misunderstood before it's
committed. For enhancements/bugs, this is the same conversation
`enhancement-intake-questions.md`/`bug-intake-questions.md` already have PM run —
those questions are the elicitation method; `intent.md` is now the artifact that
conversation produces, committed *before* drafting the fuller Enhancement
Request/Bug Report. For features, this precedes or runs alongside the Claude
Design prototype work — a fast way to scope what the prototype should actually
demonstrate before investing in building it.

## What it isn't
Not a second source of requirements once the PRD/ER/BR exists — once that
document is drafted, `intent.md`'s status moves to `Superseded by PRD` and the
PRD becomes the reference. It doesn't promote to GitLab and isn't a durable
long-term record; its whole value is being fast and disposable.
