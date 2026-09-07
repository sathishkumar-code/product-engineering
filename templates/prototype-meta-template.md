# Prototype Metadata Template

Sidecar file for a promoted prototype export, tracked independently of the PRD's
own `repo_status`/`last_promoted_revision` — a prototype can be re-exported and
re-promoted without the PRD changing, and vice versa, so one shared field would
conflate two different things.

```markdown
# Prototype: <slug>

| Field | Value |
|---|---|
| claude_design_link | <project-level link — provenance only> |
| repo_status | not-promoted / promoted |
| last_promoted_revision | <timestamp/version last pushed to the GitLab prototypes/ folder> |
| retention | Permanent — no deletion step in this process (as of 2026-09-04; see `shashi-care-gitlab-binding.md` and `PROCESS-WALKTHROUGH.md`'s cheat-sheet "Deletion" entry). |
```

Deletion is never automatic, and per this project's current config, doesn't
happen as part of the process at all — see `skill-pjm-discipline.md`'s
"Prototype deletion" (a project-config decision, generically) and
`shashi-care-pjm-config.md`'s "Prototype deletion" (this project: removed
entirely).
