# Changelog

All notable changes to `salesforce-flow-sense` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] — 2026-05-21

### Added

- **`reference/deployment-pitfalls.md`** — new reference module for metadata deploy gotchas, record type activation order, permission set companions, package install state, and managed-vs-unmanaged prefix issues. Ships empty; entries accrete via the weekly scraper + maintainer PRs.
- **`reference/tooling-api-pitfalls.md`** — new reference module for Tooling API quirks, SOQL on metadata tables, `CustomObject`/`CustomField`/`EntityDefinition` traps, and `--use-tooling-api` flag pitfalls. Ships empty.
- **SKILL.md routing extended** — the `gotcha-lookup` skill now auto-triggers on deployment and Tooling API queries in addition to Flow/Apex work, and selectively loads only the reference file(s) relevant to the situation.
- **Scraper expanded to all three topic areas** — `tools/scraper/researcher-prompt.md` now reads all three reference files for dedup, hunts Flow + deployment + Tooling API sources, and adds a `Target file` field to each candidate so the maintainer knows which reference file an entry belongs in.
- **Critic gained target-file plausibility check** — `tools/scraper/critic-prompt.md` adds a 6th adversarial criterion that rejects entries miscategorized across reference files.
- **Dry-run checklist updated** — `tools/scraper/dry-run-checklist.md` adds C6 (Target file validity), D5 (global stable-ID continuity); D4 now spans all three reference files.

### Changed

- **Stable IDs documented as a single global series** across all three reference files. README's stable-ID promise is unchanged; this just makes the cross-file invariant explicit. Next available ID is `#18`.

### Notes

- Both new reference modules ship **empty**. Real entries land via the existing scraper pipeline (Mon 9am IST) followed by human-authored PRs. The structural plumbing (skill routing, dedup expansion, target-file tagging) lands this release so entries can be merged without further infra changes.
- "Topic index for scalable dedup" (originally tagged for v0.3.0) is **deferred to v0.4.0+**. Adding two empty files doesn't change prompt-context pressure, so the index isn't blocking yet.

## [0.2.0] — 2026-05-20

### Added

- **Scheduled weekly Claude routine** (`salesforce-flow-sense-weekly-scraper`) that drafts candidate gotcha entries from four community sources and opens them as GitHub Issues for maintainer review. Runs every Monday 9:00 AM IST in Anthropic's cloud on the maintainer's Pro allowance.
- **Two-agent design** — researcher agent drafts candidates, then dispatches an adversarial critic subagent that returns KILL / EDIT / KEEP verdicts before posting.
- **Version-controlled prompts** in `tools/scraper/researcher-prompt.md` and `tools/scraper/critic-prompt.md` so changes are diffable across iterations.
- **Operator artifacts**: `tools/scraper/setup.md` (one-time deployment runbook) and `tools/scraper/dry-run-checklist.md` (19-item validation checklist).
- **Initial dry run** (2026-05-20) posted [issue #3](https://github.com/shantanudutta1/salesforce-flow-sense/issues/3) with 4 surviving candidates (1 killed, 3 edited, 1 kept by the critic) — pipeline validated end-to-end.

### Design + plan

- Design spec: [`docs/superpowers/specs/2026-05-20-scraper-design.md`](docs/superpowers/specs/2026-05-20-scraper-design.md)
- Implementation plan: [`docs/superpowers/plans/2026-05-20-scraper-routine.md`](docs/superpowers/plans/2026-05-20-scraper-routine.md)

### Known limitations

- **Auth is via PAT-in-prompt (P2 pattern).** Any view of the routine config (API response, screenshot, screen-share) exposes the live PAT. PAT is scoped tightly (Issues read+write on this repo only, 1-year expiry) which bounds blast radius, but every exposure event must be treated as "rotate immediately."
- **Migration to MCP-based auth** is the long-term path (deferred to v0.3.0+).

## [0.1.0] — 2026-05-20

### Added

- Initial plugin manifest (`.claude-plugin/plugin.json`)
- `gotcha-lookup` skill (`skills/gotcha-lookup/SKILL.md`) with auto-trigger discovery rules and the "diagnose root cause, not literal error" diagnostic principle
- Seed catalog `reference/flow-pitfalls.md` with 17 entries:
  - #1 Flow Fast Update FLS strictness
  - #2 VF-embedded flow missing recordId param
  - #3 Screen Flow "Unhandled Fault" debugging via TraceFlag
  - #4 Flow duplicate check on collections (IsNull pitfall)
  - #5 Quick Action recordId convention
  - #6 Auto-number / formula blank after Screen Flow Create
  - #7 RecordType DeveloperName mismatch across orgs
  - #8 Assumed field API name vs. reality
  - #9 Formula field returns null due to broken formula definition
  - #10 Robust test record construction
  - #11 Flow activation `null__NotFound` (fields / Record Type active / FLS)
  - #12 Get Records crashes on null filter variable
  - #13 SSN as Double, not String
  - #14 `<apexPluginCalls>` vs `@InvocableMethod`
  - #15 Managed `pkg__` vs unmanaged `pkg_` namespace prefix
  - #16 First-activation generic error from missing dependencies
  - #17 Field-length error hiding spam/integration bug (the "look past the symptom" entry)
- MIT License
- README with install/usage and contribution guide
- Maintainer-side scraper scaffold in `tools/scraper/` (empty for v0.1.0, pipeline lands in v0.2.0)

### Notes

Stable ID promise: once an entry is numbered, the number is permanent even on archive. New entries take the next available number (#18 next).
