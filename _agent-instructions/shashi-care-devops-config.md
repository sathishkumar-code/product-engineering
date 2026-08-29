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
`<folder>/03_architecture/technical-debt-register.md` and
`<folder>/03_architecture/compliance/hipaa-compliance-register.md`, live —
not a separate maintained list. Both carry a **Release-blocking: Yes/No**
column (added 2026-08-29) — check that first; fall back to Priority: Blocker
/ Status: Gap identified (material severity) for older entries not yet
tagged. Two gaps are already named by hand in the top-level repo CLAUDE.md as
of 2026-08-29: the pcc-sync hardcoded shared-secret issue (no facility
scoping) and the unauthenticated WestFax delivery webhook.

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
