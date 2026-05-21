---
name: gotcha-lookup
description: Use when debugging Salesforce errors, building or modifying Salesforce Flows (record-triggered, screen, autolaunched, scheduled), writing Apex triggers or invocable methods, planning Salesforce metadata deployments, or querying the Tooling API. Cross-references a curated catalog of common Salesforce gotchas — FLS payload issues, namespace migration, Quick Action conventions, auto-number timing, recursion, bulkification, data-quality root causes, deployment ordering, Tooling API quirks, and more — before proposing fixes.
---

# Salesforce Flow Sense — Gotcha Lookup

You are working on a Salesforce automation task. Before proposing a fix or design, **cross-reference the curated gotcha catalog** in `reference/` to see if a known pitfall matches the situation.

## When to use this skill

Trigger this skill when ANY of the following apply:

- A Salesforce Flow (record-triggered, screen, autolaunched, scheduled) is failing, misbehaving, or being built/modified
- An Apex trigger, batch, queueable, or invocable method is failing or being designed
- A Salesforce deployment is being planned or has failed (metadata, fields, record types, flows, permission sets, packages)
- An error email arrived from a Flow / Process Builder / Apex
- A Tooling API query is returning unexpected, missing, or malformed rows (e.g., `CustomObject`, `CustomField`, `EntityDefinition`, `FieldDefinition`)
- The user is using `--use-tooling-api` on the `sf` / `sfdx` CLI and seeing surprising results
- The user is debugging "why did this not fire" / "why was this null" / "why did this not deploy" / "why is this missing from the Tooling API"
- The user is asking about Flow best practices, governor limits, deployment ordering, permission set companions, or context (system vs user mode)

## Workflow

1. **Identify the failure mode or feature area** from the user's message and any error text they shared.
2. **Load only the reference module(s) relevant to the situation** via Read — do not load all three by default:
   - `reference/flow-pitfalls.md` — flow runtime errors, screen flow quirks, fast-update FLS, auto-number timing, Quick Actions, subflow versioning, Apex invocable / trigger / batch gotchas
   - `reference/deployment-pitfalls.md` — metadata deploy gotchas, record type activation order, permission set companions, package install state, managed-vs-unmanaged prefixes during deploy
   - `reference/tooling-api-pitfalls.md` — Tooling API quirks, SOQL on metadata tables, `CustomObject` / `CustomField` / `EntityDefinition` traps, `--use-tooling-api` flag pitfalls

   Routing: a Flow or Apex runtime symptom → `flow-pitfalls.md`. A deploy error or activation-order question → `deployment-pitfalls.md`. A `sf data query --use-tooling-api` surprise or missing row in a metadata table → `tooling-api-pitfalls.md`. Load multiple files only when the situation genuinely spans (e.g., a Flow that fails to activate after a deploy — load flow + deployment).
3. **Diagnose the root cause, not the literal error.** A field-level error (STRING_TOO_LONG, REQUIRED_FIELD_MISSING) often hides an upstream problem (spam, integration bug, wrong source field). Inspect ALL values in the failing record before proposing a field-level fix.
4. **Cite matching gotcha numbers** when recommending fixes — gives the user a stable reference they can look up. Stable IDs are one global series across all three reference files, so `#7` is unambiguous regardless of which file it lives in.
5. **If no entry matches**, proceed normally with your own analysis, AND flag the gap explicitly so the catalog can grow:
   > "I don't see a matching gotcha in the catalog for this. If the root cause turns out to be non-obvious, consider opening an issue at https://github.com/shantanudutta1/salesforce-flow-sense/issues so it can be added."

## Diagnostic principles (apply even before loading reference files)

- **Read every field value in error emails, not just the one that triggered the error** — multiple fields with identical garbage values usually means spam or a broken upstream integration, not a field-length bug.
- **Don't widen a field to silence a length error without investigating the source data** — you may be silently letting bad data through.
- **Don't assume a field's API name from its label** — verify with `sf sobject describe` or a `FieldDefinition` query.
- **Don't trust the Tooling API's `CustomObject` table alone** — it has known omissions; cross-check with `sobject describe`.
- **Use `--target-org` explicitly on every CLI command** — never rely on defaults.

## Catalog format

Every entry in `reference/*.md` follows this structure for fast scanning:

```
### #N — Short title
- **What happened**: concrete symptom
- **Root cause**: the non-obvious mechanism
- **Prevention**: a checkable step, not "be careful"
```

**Stable IDs are one global series** across `flow-pitfalls.md`, `deployment-pitfalls.md`, and `tooling-api-pitfalls.md`. A given `#N` lives in exactly one file. Once an entry is numbered, the number is permanent — even if the entry is later archived.

If the user reports a symptom that matches the "What happened" of any entry, surface that entry by number and reason from its Root cause and Prevention.
