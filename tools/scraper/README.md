# Scraper Pipeline (maintainer-only)

This directory contains the weekly research pipeline used to source new gotcha candidates. **It is not shipped to plugin users** — they receive the curated `reference/*.md` files only.

## Status

v0.1.0: scaffolding only. The actual pipeline lands in v0.2.0.

## Planned design

Weekly cron (Mondays, 9am local):

1. **Source pass** (in priority order):
   - Salesforce Known Issues + Release Notes (ground truth)
   - MVP/architect blogs — Automation Champion (Rakesh Gupta), UnofficialSF, Salesforce Ben, Jen Lee
   - Salesforce Stack Exchange — newest + highest-voted on `flow` / `apex` tags
   - r/salesforce + Trailblazer Community

2. **Draft output** → `reference/flow-pitfalls-pending.md` (NOT merged into the live file)

3. **Maintainer review** — apply quality gate per entry:
   - Concrete trigger?
   - Non-obvious root cause?
   - Actionable prevention?

4. **Merge keepers** into `reference/flow-pitfalls.md` with the next stable ID

5. **First Monday of each month**: dedupe + staleness audit. Move Salesforce-fixed gotchas to `reference/archive/` with a `Fixed in: <release>` note.

## Tooling decision (pending)

Built-in WebSearch + WebFetch should cover most sources. Escalate to Firecrawl/Tavily only if JS-rendered content blocks us.
