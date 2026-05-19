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
├── tools/scraper/                    maintainer-side: weekly research pipeline
└── CHANGELOG.md
```

Coming in future versions:
- `deployment-pitfalls.md` — metadata deploy, record type activation, permission set companions
- `tooling-api-pitfalls.md` — Tooling API quirks, CustomObject table omissions

## How the catalog grows

Updates ship via plugin version bumps. The maintainer-side scraper (in `tools/scraper/`) runs weekly against:

1. Salesforce Known Issues + Release Notes (ground truth)
2. MVP/architect blogs — Automation Champion, UnofficialSF, Salesforce Ben
3. Salesforce Stack Exchange — edge cases
4. r/salesforce + Trailblazer Community — fresh "this bit us" stories

Draft entries pass a **quality gate** before merge:
- Concrete trigger (what condition makes it fire?)
- Non-obvious root cause (would a competent dev guess wrong about *why*?)
- Actionable prevention (a checkable step, not "be careful")

Entries that fail any check become best-practice tips and live elsewhere.

## Contributing

Hit a gotcha that isn't catalogued?

1. Open an [issue](https://github.com/shantanudutta1/salesforce-flow-sense/issues) using the **What happened / Root cause / Prevention** structure
2. Or submit a PR adding the entry to the appropriate `reference/*.md` file with the next available stable ID

Once merged, the next version bump ships the entry to all installed users.

## Stable ID promise

Entry numbers (#1, #2, ...) are permanent. If an entry is archived because Salesforce fixed the underlying bug, the number is *not* reused — the archived entry is moved to `reference/archive/` with a date stamp and a note linking to the Salesforce Release that fixed it. This makes it safe to reference a gotcha by number across Claude sessions, blog posts, or runbooks.

## License

MIT. Use it, fork it, redistribute it — just preserve the copyright notice.
