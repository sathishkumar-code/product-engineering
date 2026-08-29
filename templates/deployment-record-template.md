---
title: Deployment Record — <release-slug>
type: deployment-record
release_slug: <release-slug>
status: Draft
---

# Deployment Record — <release-slug>

> Instantiated from `templates/deployment-record-template.md`. Authored by the DevOps Engineer persona to record a production (or other environment) promotion event. A deployment can span multiple stories/slugs, so this record lives at `<folder>/01_releases/deployment-record-<release-slug>.md` — alongside the release plan workbook, NOT under `07_build/`, which is organized per-slug.

| Field | Value |
|---|---|
| Repo(s) deployed | |
| Environment | |
| Release plan reference | (path to the release plan / workbook this deployment executes) |
| Deployed by | |
| Date / time | |
| Result | Success / Rolled back / Partial |

## Slugs / stories included

(List every feature/enhancement/bug slug included in this deployment, with links to their `implementation-note-<slug>.md` and `qa-execution-report-<slug>.md`.)

## Pre-deploy blocker check

(Confirmation that the Release-blocking column was checked in `templates/compliance-register-template.md` and `templates/technical-debt-register-template.md` for every included slug, per `skill-devops-discipline.md`'s production promotion hard-stop check. State what was checked and the outcome — e.g. "No open Release-blocking: Yes items found" or list any that required Sathish's explicit override before promotion.)

## What happened

(Plain narrative of the deployment itself — steps taken, timing, anything notable.)

## Incidents / rollback

(Any incident during or after deployment. If a rollback occurred, record it here — note that a PHI-affecting rollback requires Sathish's approval per `skill-devops-discipline.md`. If none, state "None.")

## Post-deploy monitoring findings

(What monitoring showed after deployment — errors, performance, anything routed back to PM/SA as a new intent.md or Bug Report. If none yet, state "Monitoring ongoing" or "None observed.")
