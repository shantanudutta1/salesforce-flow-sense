# Salesforce Tooling API Pitfalls

> Curated catalog of common Salesforce Tooling API gotchas. Scoped to: SOQL-on-metadata via the Tooling API, `CustomObject` / `CustomField` / `EntityDefinition` table quirks, `--use-tooling-api` flag traps in `sf` / `sfdx` CLI, and shape differences between Tooling API and the REST/Metadata APIs.
>
> Each entry follows the format: **What happened → Root cause → Prevention**.
>
> **Stable IDs are one global series shared across `flow-pitfalls.md`, `deployment-pitfalls.md`, and `tooling-api-pitfalls.md`.** Once an entry is numbered, the number is permanent even if the entry is archived. New entries take the next available number across all reference files combined.

---

_No entries yet. Candidates flow in via the weekly scraper routine (see `tools/scraper/`) and land here as maintainer-authored PRs._

---

## Contributing a new gotcha

If you've hit a Salesforce Tooling API gotcha not catalogued here, please open an issue at https://github.com/shantanudutta1/salesforce-flow-sense/issues with:

1. **What happened** — the concrete symptom (query result, error text, missing row)
2. **Root cause** — the non-obvious mechanism, ideally with a reference to Salesforce docs or a Stack Exchange thread
3. **Prevention** — a checkable step, not generic advice

Entries that don't separate symptom from cause from prevention will be asked to be reformatted before merge — the structure is what makes the catalog useful.
