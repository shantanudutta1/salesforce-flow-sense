# Scraper Routine (maintainer-only)

> The weekly Claude routine that drafts candidate gotcha entries.
> **Not shipped to plugin users.** The routine itself runs in Anthropic's
> cloud — the files here are version-controlled artifacts (prompts,
> runbook, checklist) used to configure and validate it.

## Files

- **[`researcher-prompt.md`](./researcher-prompt.md)** — the main prompt
  fed to the weekly `/schedule` routine. Version-controlled so changes
  are diffable across iterations.
- **[`critic-prompt.md`](./critic-prompt.md)** — the adversarial-critic
  subagent prompt, dispatched by the researcher.
- **[`setup.md`](./setup.md)** — one-time setup runbook (PAT creation,
  `/schedule` invocation, dry run, cron enablement).
- **[`dry-run-checklist.md`](./dry-run-checklist.md)** — validation
  checklist to run after every manual dry run, before enabling cron.

## Design reference

The full v0.2.0 design lives at
[`docs/superpowers/specs/2026-05-20-scraper-design.md`](../../docs/superpowers/specs/2026-05-20-scraper-design.md).

## Iterating on prompts

When a dry run fails the checklist, the loop is:

1. Edit the relevant prompt file (`researcher-prompt.md` or `critic-prompt.md`)
2. Commit the change with a message describing what failed and what you changed
3. Update the routine's prompt via the `/schedule` skill
4. Re-trigger the routine manually
5. Re-validate against the checklist

Expect 1–2 iterations on the initial deployment. If you're at 3+ iterations
on the same issue, flag it as a design problem and revisit the spec.
