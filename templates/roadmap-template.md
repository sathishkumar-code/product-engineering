# Roadmap Template

Standard SaaS practice favors a **theme-based, Now/Next/Later roadmap** over a
feature-list-with-dates roadmap — dates on individual features create false
precision and go stale immediately, whereas themes stay accurate longer and
communicate intent even when exact timing shifts. Keep a lightweight timeline view
alongside it for stakeholders who need a date-oriented picture, but treat the
theme view as primary.

```markdown
# <Product> Roadmap

Last updated: <date>

## Now
Themes actively being worked, this quarter/cycle.

| Theme | Why it matters | Scope (SAL / SNF / Shared) | Status | Related PRDs |
|---|---|---|---|---|
| | | | | |

## Next
Themes committed for the upcoming cycle but not yet started.

| Theme | Why it matters | Scope | Target timeframe | Related PRDs |
|---|---|---|---|---|
| | | | | |

## Later
Themes under consideration, not yet committed. No target date — a date here implies
a commitment that hasn't been made.

| Theme | Why it matters | Scope | Related PRDs |
|---|---|---|---|
| | | | |

## Shared SAL/SNF themes
Call out explicitly which themes span both products, since the table columns above
can make a shared theme look like it belongs to one product by where it's listed.

## Timeline view (secondary, optional)
A quarter-by-quarter table for stakeholders who need dates. Only populate for "Now"
and "Next" themes — don't put speculative dates on "Later" items.

| Theme | Q<n> YYYY | Q<n+1> YYYY | ... |
|---|---|---|---|
```

## Why theme-based, not feature-based
A feature-by-feature roadmap with dates answers "when does X ship" and is wrong the
moment scope shifts. A theme-based roadmap answers "what are we trying to accomplish
and in what order" and stays true even as the specific features under a theme
change. Individual features live in `01_releases/` and `02_prd/` — the roadmap
should link to them, not restate their detail.
