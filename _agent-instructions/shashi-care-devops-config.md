# Config: DevOps Engineer — Shashi Care (per code repo)

Pairs with `skill-devops-discipline.md`. Same one-instance-per-repo model and
repo/product mapping as `shashi-care-developer-config.md` — see that file's
table.

## Release/deployment tooling per repo
No dedicated binding file — inherits whatever each code repo's own CLAUDE.md
already documents (GitLab CI, Fastlane for the mobile/TV apps, direct
container deploy for the Node/React repos). Add a binding file only if a
genuinely shared, cross-repo deployment tool is adopted later.

## Where production-promotion blockers are sourced
Per-product, not one uniform path — verified against the actual filesystem
2026-08-29, since what's live doesn't yet match a single generic shape:

| Product | Technical debt source (today) | Compliance source (today) |
|---|---|---|
| SNF | `SNF/03_architecture/_as-built/technical-debt.md` — a pre-existing, populated registry with its own schema (`Severity`: Blocker/Critical/High/Medium/Low; `Decision/Status` as free text), **not** the generic template shape and no `Release-blocking` column. Treat `Severity: Blocker` or `Severity: Critical` whose `Decision/Status` isn't `Resolved` as the blocking signal. | `SNF/03_architecture/compliance/hipaa-compliance-register.md` — real, 39-entry, **Sathish-edit-only**, own schema (`Priority`: High/Medium/Low; `Status`: Not Started/In Progress/Decision Needed/Needs Analysis/Done), no `Release-blocking` column. Treat `Priority: High` whose `Status` isn't `Done` as the blocking signal. |
| SAL | none yet — `SAL/03_architecture/` has no content | none yet |
| shashi-care-core | none yet — `shashi-care-core/03_architecture/` has no content | none yet |

The generic, template-shaped `<folder>/03_architecture/technical-debt-register.md`
and a generic per-product compliance register (carrying the
**Release-blocking: Yes/No** column added 2026-08-29 to
`templates/technical-debt-register-template.md` and
`templates/compliance-register-template.md`) don't exist yet for any product.
Check those first once/if PM or SA stands them up for a given product — but an
absent register is not the same as a clean check; see
`skill-devops-discipline.md`'s hard-stop section for the escalation rule.
Two gaps are already named by hand in the top-level repo CLAUDE.md as of
2026-08-29 regardless of register state: the pcc-sync hardcoded shared-secret
issue (no facility scoping) and the unauthenticated WestFax delivery webhook.

**SAL/shashi-care-core "none yet" is a scheduled gap, not an oversight.**
Sathish confirmed 2026-08-29: System Architect completes an initial
`technical-debt-register.md` + `compliance-register.md` logging pass for each
product before its first real deployment — see
`PROCESS-WALKTHROUGH.md`'s Open worklog items. Until that happens, this
persona keeps escalating rather than promoting for either product; that's the
correct behavior, not a bug to route around.

**SNF's fallback proxy is time-bound, not permanent.** Sathish decided
2026-08-29: a `Release-blocking`-equivalent field gets added to
`_as-built/technical-debt.md` and `hipaa-compliance-register.md` when SNF's
first real release-plan drafting with PjM begins (`PROCESS-WALKTHROUGH.md`
Stage 10), not before. Until that cycle happens, keep using the
Severity/Priority proxy above for SNF — don't treat its absence as something
this persona should chase or flag repeatedly.

## Deployment record location
`<folder>/01_releases/deployment-record-<release-slug>.md` — see
`shashi-care-doc-tree.md`'s Build/Release-stage addition. One record per
actual deployment event, referencing every slug it includes — not duplicated
per slug the way implementation notes and QA reports are, since one
deployment commonly bundles several slugs.

## Return-path destination
New intent.md/Bug Report → Product Manager, filed per the standard
direct-intake path (`shashi-care-doc-tree.md`'s per-slug shape), same as any
other enhancement/bug.
