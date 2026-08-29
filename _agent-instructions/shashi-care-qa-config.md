# Config: QA Engineer — Shashi Care (per code repo)

Pairs with `skill-qa-discipline.md`. Same one-instance-per-repo model and
repo/product mapping as `shashi-care-developer-config.md` — see that file's
table.

## Where approved test material comes from
`<folder>/02_prd/{features,enhancements,bugs}/<slug>/test-scenarios.md` and
`test-cases.xlsx` (PM-Test-Cases and SA-Technical-Test-Cases sheets),
`qa_status` already at whatever the review-layer requires before execution
starts.

## Execution report location
`<folder>/07_build/{features,enhancements,bugs}/<slug>/qa-execution-report-<slug>.md`.

## Test/CI tooling
No dedicated binding file — inherits whatever each code repo's own CLAUDE.md
already documents (e.g. GitLab CI, ESLint→eslint-report.json, SonarQube gate,
Fastlane for the mobile/TV apps). A binding file gets created only if a
genuinely shared, cross-repo test-runner or CI dashboard is adopted later —
analogous to why `shashi-care-clickup-binding.md`/`shashi-care-gitlab-binding.md`
exist for tools that actually are shared across every repo.

## Bug Report filing
Uses Product Manager's existing `templates/bug-intake-questions.md` →
`templates/bug-report-template.md`, filed at
`<folder>/02_prd/bugs/<new-slug>/` per `shashi-care-doc-tree.md`'s per-slug
shape — same as any other direct-intake bug.

## Escalation contacts
PHI/compliance-flagged defect → Sathish and System Architect, immediately.
Waiver requests → Sathish, via the release go/no-go gate, never decided here.
