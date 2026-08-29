# Team Structure Template

A cross-cutting reference document, not owned by any single persona — Product
Manager, System Architect, and Project Manager personas all read it for context
(who to route a question to, who has decision authority) but none of them edits it
unilaterally; treat changes to this document as Sathish's call.

```markdown
# Team Structure — <Project/Org name>

Last updated: <date>

## 1. Roster
| Name | Role | Products (Core / SAL / SNF) | Contact |
|---|---|---|---|

## 2. RACI — key decisions and processes
R = Responsible (does the work), A = Accountable (owns the outcome, final say),
C = Consulted (input sought before deciding), I = Informed (told after the fact).
Exactly one **A** per row — more than one accountable owner is the most common way
a RACI matrix stops being useful.

| Decision / process | Product Manager | System Architect | Project Manager | Sathish | Dev team |
|---|---|---|---|---|---|
| PRD approval | R | C | I | A | I |
| Technical design approval | I | R | I | C | A/C |
| Epic/Story creation in tracker | C | C | R | I | I |
| Sprint scope commitment | I | I | R | A | R |
| Release go/no-go | C | C | R | A | C |
| Production incident response | I | R | I | A | R |

Adjust roles/columns to match reality — this is a starting shape, not a fixed
schema; add rows for any recurring decision that currently has no clear owner.

## 3. Escalation path
Who to go to, in order, when something is blocked and the obvious owner is
unavailable or the decision exceeds their authority.

| Situation | First contact | Escalate to |
|---|---|---|

## 4. Coverage / on-call (if applicable)
| Role | Primary | Backup | Rotation |
|---|---|---|---|
```
