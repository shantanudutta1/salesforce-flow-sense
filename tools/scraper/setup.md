# Setup Runbook — Scheduled Gotcha Scraper Routine

> This runbook documents the one-time setup steps for the v0.2.0 weekly
> routine. Follow once at initial deployment. Re-follow at token rotation
> (annual) or when migrating to a new maintainer account.

## Prerequisites

- Access to the maintainer's GitHub account (`shantanudutta1`)
- An active Claude Pro subscription (for the routine's allowance)
- Claude Code installed locally with the `/schedule` skill available

## Step 1 — Create the GitHub fine-grained PAT

1. Visit https://github.com/settings/personal-access-tokens
2. Click **"Generate new token"** (fine-grained, not classic)
3. Configure the token:
   | Field | Value |
   |---|---|
   | Token name | `salesforce-flow-sense-routine` |
   | Resource owner | `shantanudutta1` |
   | Expiration | **1 year** (set a calendar reminder for month 11 to rotate) |
   | Repository access | **Only select repositories** → pick `salesforce-flow-sense` |
4. Under **Repository permissions**, set:
   - `Contents`: **Read-only** (for `git clone`)
   - `Issues`: **Read and write** (for creating issues + listing pending ones)
   - Leave everything else at **No access**.
5. Click **Generate token**, then **immediately copy the value**. GitHub
   will not show it again.

## Step 2 — Invoke the /schedule skill to create the routine

From a Claude Code session, invoke:

```
/schedule
```

When prompted, provide:

- **Routine name:** `salesforce-flow-sense-weekly-scraper`
- **Cron schedule:** `0 9 * * 1` (Monday 9am)
- **Timezone:** Maintainer's local timezone (e.g., `America/Chicago`)
- **Prompt:** Paste the entire contents of `tools/scraper/researcher-prompt.md`
- **Environment variables:**
  - `GITHUB_TOKEN` = (the PAT from Step 1)
  - `REPO` = `shantanudutta1/salesforce-flow-sense`

Confirm the routine is created with `claude routine list` or via the
`/schedule` skill's list-routines command.

## Step 3 — Do a manual dry run (DO NOT enable cron yet)

The `/schedule` skill supports manual triggering. Trigger the routine
once and wait for it to finish.

Validate the output against `tools/scraper/dry-run-checklist.md`.

## Step 4 — Iterate on the prompt if needed

If the dry-run output fails one or more checklist items:

1. Edit `tools/scraper/researcher-prompt.md` (or `critic-prompt.md`)
2. Commit the change with a message describing what failed and what you changed
3. Update the routine's prompt via the `/schedule` skill
4. Re-trigger manually
5. Re-validate

Expect 1–2 iterations. If you're at 3+ iterations on the same issue,
flag it as a design problem and revisit the spec.

## Step 5 — Enable the weekly cron

Once the dry run passes the checklist, enable the routine via:

```
/schedule enable salesforce-flow-sense-weekly-scraper
```

(Exact command may differ — check the `/schedule` skill help.)

## Step 6 — Monitor the first 3 weeks

For each weekly run, check:

1. The GitHub Issue was created on time (Monday morning)
2. The candidates (if any) are plausible and source-cited
3. The dedup worked — no entries that overlap with the existing catalog

Small prompt tweaks are normal during this period. After 3 weeks of stable
output, the routine is considered production.

## Rollback / disable

To pause the routine (e.g., during a maintainer vacation):

```
/schedule disable salesforce-flow-sense-weekly-scraper
```

To delete the routine entirely:

```
/schedule delete salesforce-flow-sense-weekly-scraper
```

The PAT can be revoked at https://github.com/settings/personal-access-tokens
if you suspect it was leaked.
