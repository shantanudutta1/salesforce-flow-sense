# Salesforce Flow Sense

A Claude Code plugin that gives Claude a curated catalog of common **Salesforce Flow, Apex, and deployment gotchas** to cross-reference before proposing fixes.

If you've ever asked Claude to help debug a Salesforce Flow and watched it confidently propose widening a field to silence a `STRING_TOO_LONG` error — when the real cause was a spam bot filling every field with garbage — this plugin is for you.

## What it does

When Claude is working on a Salesforce task (debugging an error, building a Flow, planning a deployment), the plugin's `gotcha-lookup` skill activates automatically and loads a reference catalog of pitfalls. Claude cross-references your situation against the catalog *before* proposing a fix, citing matching entries by number so you have a stable reference to look up.

Crucially, the skill enforces **diagnose root cause, not literal error** — Claude won't recommend a field-level fix until it's inspected the actual data for upstream patterns (spam, integration bugs, wrong source field).

## Install

```
/plugin marketplace add shantanudutta1/salesforce-flow-sense
/plugin install salesforce-flow-sense
```

Or for local testing during development:

```
claude --plugin-dir /path/to/salesforce-flow-sense
```

The skill auto-triggers; no command needed. To invoke explicitly:

```
/salesforce-flow-sense:gotcha-lookup
```

## What's inside

```
salesforce-flow-sense/
├── .claude-plugin/plugin.json        manifest
├── skills/gotcha-lookup/
│   ├── SKILL.md                      thin router skill
│   └── reference/
│       ├── flow-pitfalls.md          17 entries covering Flow + Apex
│       └── archive/                  Salesforce-fixed gotchas (dated)
├── docs/superpowers/specs/           design docs for upcoming work
├── tools/scraper/                    maintainer-side: routine notes
└── CHANGELOG.md
```

Coming in future versions:
- `deployment-pitfalls.md` — metadata deploy, record type activation, permission set companions
- `tooling-api-pitfalls.md` — Tooling API quirks, CustomObject table omissions

## How the catalog grows

Updates ship via plugin version bumps. A **scheduled Claude routine** runs weekly on the maintainer's Claude Pro allowance, sourcing candidate gotchas from:

1. Salesforce Known Issues + Release Notes (ground truth)
2. MVP/architect blogs — Automation Champion, UnofficialSF, Salesforce Ben, Jen Lee
3. Salesforce Stack Exchange — edge cases on the `[flow]` and `[apex]` tags
4. r/salesforce + Trailblazer Community — fresh "this bit us" stories

Candidates pass a two-stage filter before any catalog merge:

1. **Quality gate** — every candidate must have a concrete trigger, a non-obvious root cause, and an actionable prevention. Vague "be careful" advice is rejected.
2. **Adversarial critic** — a second Claude agent reviews drafts looking for hallucinations, redundancy, and entries that don't survive scrutiny. Default bias is to KILL.

Survivors are posted as GitHub Issues labeled `candidate` for the maintainer's weekly review. Catalog merges remain human-authored — the routine never writes to `reference/*.md` directly.

See [docs/superpowers/specs/2026-05-20-scraper-design.md](docs/superpowers/specs/2026-05-20-scraper-design.md) for the full v0.2.0 design.

## Contributing

Hit a gotcha that isn't catalogued?

1. Open an [issue](https://github.com/shantanudutta1/salesforce-flow-sense/issues) using the **What happened / Root cause / Prevention** structure
2. Or submit a PR adding the entry to the appropriate `reference/*.md` file with the next available stable ID

Once merged, the next version bump ships the entry to all installed users.

## Roadmap

| Version | Status | Highlights |
|---|---|---|
| **v0.1.0** | ✅ Released | Initial 17 seed entries; `gotcha-lookup` skill; static catalog |
| **v0.2.0** | 🚧 In design | Scheduled weekly Claude routine that drafts new candidates and opens them as GitHub Issues for review. Design: [docs/superpowers/specs/2026-05-20-scraper-design.md](docs/superpowers/specs/2026-05-20-scraper-design.md) |
| **v0.3.0** | 📋 Planned | `deployment-pitfalls.md` and `tooling-api-pitfalls.md` reference files; topic index for scalable dedup |

## Stable ID promise

Entry numbers (#1, #2, ...) are permanent. If an entry is archived because Salesforce fixed the underlying bug, the number is *not* reused — the archived entry is moved to `reference/archive/` with a date stamp and a note linking to the Salesforce Release that fixed it. This makes it safe to reference a gotcha by number across Claude sessions, blog posts, or runbooks.

## License

MIT. Use it, fork it, redistribute it — just preserve the copyright notice.
