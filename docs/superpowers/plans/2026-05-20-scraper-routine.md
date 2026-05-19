# Scraper Routine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement and deploy the v0.2.0 scheduled Claude routine that drafts weekly Salesforce gotcha candidates from four community sources and opens them as GitHub Issues for maintainer review.

**Architecture:** Two-agent design (researcher + adversarial critic) running as a weekly `/schedule` routine in Anthropic's cloud. Prompts are version-controlled as markdown files in `tools/scraper/`. Outputs delivered as labeled GitHub Issues on `shantanudutta1/salesforce-flow-sense`. The routine never writes to `reference/*.md` directly — catalog merges remain human-authored PRs.

**Tech Stack:** Claude Code `/schedule` skill (creates the cloud routine), `gh` CLI (within the routine, for issue creation), `git` (within the routine, for catalog read), GitHub fine-grained PAT (auth).

**Spec reference:** [`docs/superpowers/specs/2026-05-20-scraper-design.md`](../specs/2026-05-20-scraper-design.md)

---

## Task 1: Version-control the researcher prompt

The researcher prompt is the "code" of the routine. Keeping it in git means we can iterate, diff, and revert across deployments.

**Files:**
- Create: `tools/scraper/researcher-prompt.md`

- [ ] **Step 1: Write the researcher prompt file**

Create `tools/scraper/researcher-prompt.md` with this exact content:

````markdown
# Researcher Prompt — v0.2.0

> Pasted into the `/schedule` routine config. Version-controlled here so changes are diffable across iterations.

---

You are a Salesforce automation gotcha curator for the salesforce-flow-sense
plugin. Your job: find non-obvious Flow/Apex/deployment pitfalls from
community sources and propose them as candidates.

## Sources (priority order)

**Tier 1 — Ground truth (highest priority):**
- Salesforce Known Issues board (search via WebSearch with queries like
  `site:salesforce.com "known issue" flow`)
- Latest Salesforce Release Notes (current release docs)

**Tier 2 — MVP / architect blogs:**
- automationchampion.com (Rakesh Gupta)
- unofficialsf.com
- salesforceben.com (flow/apex tag)
- jenwlee.com

**Tier 3 — Q&A signal:**
- salesforce.stackexchange.com — newest questions on `[flow]` and `[apex]` tags

**Tier 4 — Trench reports:**
- reddit.com/r/salesforce — posts mentioning "gotcha", "got bit",
  "wasted a day", "this bit us"

## Process

1. Clone the repo (read-only):
   ```
   git clone https://github.com/shantanudutta1/salesforce-flow-sense /tmp/sfs
   ```

2. Read the existing catalog to know what's already covered:
   ```
   cat /tmp/sfs/skills/gotcha-lookup/reference/flow-pitfalls.md
   ```
   Build an internal index of root causes (not just titles — the same
   gotcha can be described with different titles).

3. List open candidate issues to know what's already pending:
   ```
   gh issue list --repo shantanudutta1/salesforce-flow-sense \
                 --label candidate --state open --limit 50
   ```

4. **Source pass.** Spend ~75% of your tool budget on tiers 1–2, ~25%
   on tiers 3–4. Focus on content from the last 14 days.
   For each potential candidate, find:
   - The **concrete trigger** (specific scenario)
   - The **non-obvious root cause** (the WHY a competent dev would miss)
   - The **actionable prevention** (a checkable step, not "be careful")
   - A **source URL** you can cite

5. **Quality gate.** Reject any candidate that fails ANY of:
   - Trigger is concrete (not "sometimes flows fail")
   - Root cause is non-obvious (if a dev would immediately know WHY, not a gotcha)
   - Prevention is checkable (not "be careful")
   - Not already in the catalog (Step 2)
   - Not in an open candidate issue (Step 3)
   - Has a citable source URL

6. **Cap at 5 candidates per run.** If you find more, pick the strongest 5.

7. **Hand off to critic subagent.** Dispatch a fresh subagent with the
   critic prompt (see `tools/scraper/critic-prompt.md`) and the drafts
   as its input. Await verdicts.

8. **Apply verdicts:**
   - KILL: remove the entry
   - EDIT: incorporate the suggested edit
   - KEEP: include as-is

9. **Post output.**
   - If **0 candidates survive**: open issue titled
     `No new gotchas — YYYY-MM-DD` with body listing the sources scanned.
   - If **1+ candidates survive**: open issue titled
     `Gotcha candidates — YYYY-MM-DD` with label `candidate`. Body lists
     the surviving drafts plus the critic's verdicts as a transparency log.

## Output format (per entry)

```
### [next stable ID — fetch from catalog] — Short title
- **What happened**: [concrete symptom]
- **Root cause**: [non-obvious mechanism]
- **Prevention**: [checkable step]
- **Source(s)**: [URL(s)]
```

## Constraints

- Never write to `reference/*.md` directly — only open GitHub issues.
- Never propose more than 5 candidates per run.
- Always cite source URLs for each candidate.
- If a source is unreachable (rate-limited, 503), try the other three and
  note the unreachable one in the issue body.
- If all four sources are unreachable, open an issue labeled
  `routine-failure` describing what failed. The routine will self-heal
  next week.
````

- [ ] **Step 2: Verify the file matches the spec**

Run: `diff <(grep -A 100 "## 1. Researcher agent" docs/superpowers/specs/2026-05-20-scraper-design.md | head -80) <(cat tools/scraper/researcher-prompt.md | head -80)`

Expected: minor diff is OK (the plan file has more structure), but the source list and process steps must match.

- [ ] **Step 3: Commit**

```bash
git add tools/scraper/researcher-prompt.md
git commit -m "feat(scraper): version-control researcher prompt"
```

---

## Task 2: Version-control the critic prompt

The critic prompt is dispatched by the researcher as a subagent. Adversarial by design.

**Files:**
- Create: `tools/scraper/critic-prompt.md`

- [ ] **Step 1: Write the critic prompt file**

Create `tools/scraper/critic-prompt.md` with this exact content:

````markdown
# Critic Prompt — v0.2.0

> Dispatched as a fresh subagent by the researcher after drafts are complete.
> Sees only the drafts (no research context), so it judges them on the
> same evidence the maintainer would.

---

You are an adversarial reviewer of proposed Salesforce gotcha catalog entries.
The drafts below were generated by another agent from community sources.
Your job is to find reasons to KILL each one.

For each entry, return a verdict:

- **KILL**: the entry doesn't belong in the catalog. Give the reason.
- **EDIT**: the entry is on the right track but needs revision. Give the
  specific suggested edit.
- **KEEP**: the entry passes the quality gate as-is.

## Adversarial criteria

1. **Is the trigger concrete?**
   Can you imagine running the failing scenario yourself?
   Reject "sometimes X happens" or "users report Y."

2. **Is the root cause non-obvious?**
   Would a competent Salesforce dev guess wrong about WHY?
   If the root cause is "you forgot to deploy the field," that's not a
   gotcha — that's the obvious answer.

3. **Is the prevention checkable?**
   Can a dev tick a box and know they did it? Reject "be careful" or
   "always think about X."

4. **Is the entry hallucinated?**
   Does the cited source actually describe this gotcha, or is the agent
   inferring? **Spot-check at least one source URL per entry.**

5. **Is it redundant within this batch?**
   Even though dedup against the catalog was done, look for entries
   that overlap significantly with each other in THIS batch.

## Default bias

**KILL or EDIT.** KEEP only when an entry is genuinely strong.

> The cost of a bad KEEP (junk in the catalog) is higher than the cost
> of a false KILL (re-discovered next week).

## Output format

```
### Entry 1: "Short title from the draft"
**Verdict:** KILL | EDIT | KEEP
**Reason / suggested edit:** ...

### Entry 2: ...
```

After all entries, add a summary line:

```
**Summary:** X KILL, Y EDIT, Z KEEP (out of N total)
```
````

- [ ] **Step 2: Verify it matches the spec section**

Run: `wc -l tools/scraper/critic-prompt.md`

Expected: between 40 and 60 lines (compact prompt).

- [ ] **Step 3: Commit**

```bash
git add tools/scraper/critic-prompt.md
git commit -m "feat(scraper): version-control critic prompt"
```

---

## Task 3: Write the setup runbook

The runbook documents the human steps to deploy the routine. Future-you (or a contributor) needs this to re-deploy from scratch.

**Files:**
- Create: `tools/scraper/setup.md`

- [ ] **Step 1: Write the setup runbook**

Create `tools/scraper/setup.md` with this exact content:

````markdown
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
````

- [ ] **Step 2: Verify the runbook is complete**

Run: `grep -E "^## Step" tools/scraper/setup.md`

Expected output: 6 lines, "Step 1" through "Step 6".

- [ ] **Step 3: Commit**

```bash
git add tools/scraper/setup.md
git commit -m "docs(scraper): add setup runbook for v0.2.0 routine"
```

---

## Task 4: Write the dry-run validation checklist

The "tests" for the routine. After a manual dry run, the maintainer walks through this checklist before declaring the routine ready for cron.

**Files:**
- Create: `tools/scraper/dry-run-checklist.md`

- [ ] **Step 1: Write the checklist**

Create `tools/scraper/dry-run-checklist.md` with this exact content:

````markdown
# Dry-Run Validation Checklist

> Walk through this checklist after every manual dry run before enabling
> (or re-enabling) the weekly cron. Each item should be a clear pass/fail.

## A. Routine execution

- [ ] **A1.** The routine ran to completion without throwing an error.
- [ ] **A2.** The routine's log (visible via `/schedule` skill) shows it
      cloned the repo, listed open issues, and made WebSearch/WebFetch calls.

## B. Source pass

- [ ] **B1.** The log shows the routine accessed (or attempted to access)
      at least one URL from EACH of the four source tiers.
- [ ] **B2.** Roughly 75% of the source-pass time was on tiers 1–2; only
      ~25% on tiers 3–4.
- [ ] **B3.** Any unreachable sources were noted in the output issue body.

## C. Output format

- [ ] **C1.** A GitHub Issue was created on the repo.
- [ ] **C2.** The issue title matches `Gotcha candidates — YYYY-MM-DD` or
      `No new gotchas — YYYY-MM-DD` (depending on whether candidates survived).
- [ ] **C3.** If candidates exist, the issue is labeled `candidate`.
- [ ] **C4.** Each candidate in the issue body has all four fields:
      What happened / Root cause / Prevention / Source(s).
- [ ] **C5.** Each candidate has a clickable source URL that actually loads
      and describes the gotcha. (Hallucination check — sample at least one.)

## D. Quality gate

- [ ] **D1.** Every candidate has a concrete trigger (a specific scenario,
      not "sometimes").
- [ ] **D2.** Every candidate has a non-obvious root cause (a competent
      dev would guess wrong about WHY).
- [ ] **D3.** Every candidate has a checkable prevention (not "be careful").
- [ ] **D4.** No candidate duplicates an existing entry in
      `reference/flow-pitfalls.md` (semantic dedup, not just title match).

## E. Critic step

- [ ] **E1.** The issue body includes a "Critic verdicts" section.
- [ ] **E2.** At least one KILL or EDIT verdict was issued (if not, the
      critic may not be running adversarially enough).
- [ ] **E3.** The summary line (`Summary: X KILL, Y EDIT, Z KEEP`) is
      present and consistent with the per-entry verdicts.

## F. Count

- [ ] **F1.** Total candidates in the issue is ≤ 5.

## G. Zero-day case

- [ ] **G1.** If no candidates met the gate, the issue is titled
      `No new gotchas — YYYY-MM-DD` with a body listing the sources scanned.
      (Not just an empty issue.)

---

## What to do if items fail

| Failing items | Likely fix |
|---|---|
| A1, A2 | Check the PAT permissions; check the `/schedule` config |
| B1, B2, B3 | Adjust the source-pass instructions in `researcher-prompt.md` |
| C1–C4 | Adjust the "Post output" section of `researcher-prompt.md` |
| C5, D1–D3 | Tighten the quality-gate language in `researcher-prompt.md`; make critic more adversarial in `critic-prompt.md` |
| D4 | Check that the routine is reading the catalog file before drafting (Process Step 2 in researcher-prompt.md) |
| E1–E3 | Verify the researcher is actually dispatching the critic subagent (Process Step 7) |
| F1 | Add a hard cap reminder in the researcher prompt |
| G1 | Adjust the "If 0 survive" branch of the "Post output" section |

A failure on any item triggers a prompt iteration (Setup Runbook Step 4).
````

- [ ] **Step 2: Verify the checklist sections**

Run: `grep -cE "^- \[ \]" tools/scraper/dry-run-checklist.md`

Expected: ≥ 18 checklist items.

- [ ] **Step 3: Commit**

```bash
git add tools/scraper/dry-run-checklist.md
git commit -m "docs(scraper): add dry-run validation checklist"
```

---

## Task 5: Update `tools/scraper/README.md` to point at new files

The existing README in this directory was a v0.1.0 placeholder. Replace it with a real index.

**Files:**
- Modify: `tools/scraper/README.md`

- [ ] **Step 1: Read the current file**

```bash
cat tools/scraper/README.md
```

(For context only — the next step overwrites it.)

- [ ] **Step 2: Replace the file content**

Overwrite `tools/scraper/README.md` with:

````markdown
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
````

- [ ] **Step 3: Verify the new content**

Run: `head -5 tools/scraper/README.md`

Expected: title line `# Scraper Routine (maintainer-only)`.

- [ ] **Step 4: Commit**

```bash
git add tools/scraper/README.md
git commit -m "docs(scraper): replace placeholder README with v0.2.0 file index"
```

---

## Task 6 (LIVE OPS): User creates the GitHub PAT

This is a manual step the **user must perform in their browser** — the executing agent cannot create PATs on the user's behalf. The agent provides instructions and waits.

**Files:** None (browser action)

- [ ] **Step 1: Send the user the instructions**

The executing agent posts:

> Please create the GitHub fine-grained PAT now, following these steps:
>
> 1. Visit https://github.com/settings/personal-access-tokens
> 2. Click **"Generate new token"** (fine-grained, not classic)
> 3. Configure:
>    - Token name: `salesforce-flow-sense-routine`
>    - Resource owner: `shantanudutta1`
>    - Expiration: **1 year**
>    - Repository access: **Only select repositories** → `salesforce-flow-sense`
> 4. Under **Repository permissions**:
>    - Contents: **Read-only**
>    - Issues: **Read and write**
>    - (Leave all others at No access)
> 5. Click **Generate token** and **immediately copy the value** —
>    GitHub will not show it again.
> 6. Reply with the token value (it will be passed straight to the
>    `/schedule` config; never written to a file or logged).

- [ ] **Step 2: Wait for the user to reply with the token**

Do NOT proceed to Task 7 until the user provides the token value.

- [ ] **Step 3: Acknowledge receipt without echoing the token**

Once received, reply only with:

> Token received. Proceeding to create the routine via `/schedule`.

Do NOT paste the token back into chat. Do NOT save it to a file.

---

## Task 7 (LIVE OPS): Create the routine via /schedule

**Files:** None (invokes the `/schedule` skill)

- [ ] **Step 1: Invoke /schedule**

From the executing agent's session, invoke the `/schedule` skill.

- [ ] **Step 2: Configure the routine**

Provide these parameters to the `/schedule` skill:

- **Routine name:** `salesforce-flow-sense-weekly-scraper`
- **Cron schedule:** `0 9 * * 1`
- **Timezone:** ask the user for their local TZ if not already known
  (e.g., `America/Chicago`)
- **Prompt:** the full contents of `tools/scraper/researcher-prompt.md`
- **Environment variables:**
  - `GITHUB_TOKEN` = the PAT from Task 6
  - `REPO` = `shantanudutta1/salesforce-flow-sense`
- **Initial state:** **disabled** (do not enable cron yet; we dry-run first)

- [ ] **Step 3: Verify the routine was created**

Run `/schedule list` (or equivalent) and confirm
`salesforce-flow-sense-weekly-scraper` appears with status "disabled".

- [ ] **Step 4: Capture the routine ID**

Note the routine's unique ID for Task 8.

---

## Task 8 (LIVE OPS): Manual dry run

**Files:** None (triggers the routine via `/schedule`)

- [ ] **Step 1: Manually trigger the routine**

Use `/schedule trigger salesforce-flow-sense-weekly-scraper` (or
equivalent) to fire the routine once, immediately.

- [ ] **Step 2: Wait for completion**

The routine should complete within ~10 minutes. Monitor its log via
`/schedule logs salesforce-flow-sense-weekly-scraper`.

- [ ] **Step 3: Locate the output issue**

Find the GitHub Issue it created on
https://github.com/shantanudutta1/salesforce-flow-sense/issues

---

## Task 9 (LIVE OPS): Validate against the dry-run checklist

**Files:** None (manual review)

- [ ] **Step 1: Walk through `tools/scraper/dry-run-checklist.md` item by item**

For each checkbox in the checklist file, mark pass/fail.

- [ ] **Step 2: Report the result**

The executing agent reports to the user:

> Dry-run validation: X/Y items passed. Failing items: [...].

- [ ] **Step 3: Decision point**

- If all items pass → proceed to Task 11 (enable cron).
- If 1+ items fail → proceed to Task 10 (iterate prompt).

---

## Task 10 (LIVE OPS, CONDITIONAL): Iterate the prompt

Only if Task 9 reports failing items. Expect 1–2 iterations max.

**Files:**
- Modify: `tools/scraper/researcher-prompt.md` (likely) or `tools/scraper/critic-prompt.md`

- [ ] **Step 1: Identify what failed and which prompt to edit**

Use the "What to do if items fail" table in `dry-run-checklist.md` to
map failing items to the correct prompt section.

- [ ] **Step 2: Edit the prompt file**

Edit the appropriate file. The change should be the **minimum** edit
that addresses the failing checklist item — avoid scope creep.

- [ ] **Step 3: Commit the change with a clear message**

```bash
git add tools/scraper/researcher-prompt.md     # or critic-prompt.md
git commit -m "fix(scraper): <what failed>; <what changed>"
```

Example: `fix(scraper): C5 hallucinated source URL; require critic to spot-check at least one URL`

- [ ] **Step 4: Update the routine's prompt via /schedule**

Use the `/schedule` skill's update-prompt command (or equivalent) to
push the new prompt to the routine config.

- [ ] **Step 5: Re-trigger and re-validate**

Loop back to Task 8 Step 1.

> **Stopping condition:** If you're on iteration 3+ on the same checklist
> item, STOP. The issue is a design problem, not a prompt problem.
> Revisit `docs/superpowers/specs/2026-05-20-scraper-design.md` and
> re-run the brainstorming skill for that section.

---

## Task 11 (LIVE OPS): Enable the weekly cron

**Files:** None (enables the routine via `/schedule`)

- [ ] **Step 1: Enable the routine**

Run: `/schedule enable salesforce-flow-sense-weekly-scraper`

- [ ] **Step 2: Verify the schedule shows as enabled**

Run: `/schedule list`

Expected: `salesforce-flow-sense-weekly-scraper` shows status "enabled"
with next-run time set to the upcoming Monday 9am.

---

## Task 12 (FINALIZE): Bump version, update CHANGELOG, tag v0.2.0, push

The routine is live. Mark v0.2.0 as shipped.

**Files:**
- Modify: `.claude-plugin/plugin.json`
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Bump version in `plugin.json`**

Edit `.claude-plugin/plugin.json`. Change `"version": "0.1.0"` to `"version": "0.2.0"`.

- [ ] **Step 2: Add v0.2.0 entry to `CHANGELOG.md`**

Insert this block in `CHANGELOG.md` immediately after the `# Changelog`
header line and before the `## [0.1.0]` block:

```markdown
## [0.2.0] — YYYY-MM-DD

### Added

- Scheduled weekly Claude routine (`/schedule salesforce-flow-sense-weekly-scraper`) that drafts candidate gotcha entries from four community sources and opens them as GitHub Issues for maintainer review
- Two-agent design: researcher + adversarial critic subagent
- Version-controlled prompts in `tools/scraper/researcher-prompt.md` and `tools/scraper/critic-prompt.md`
- Setup runbook (`tools/scraper/setup.md`) and dry-run validation checklist (`tools/scraper/dry-run-checklist.md`)

### Design

- Design spec committed at `docs/superpowers/specs/2026-05-20-scraper-design.md`
- Implementation plan at `docs/superpowers/plans/2026-05-20-scraper-routine.md`
```

Replace `YYYY-MM-DD` with today's actual date.

- [ ] **Step 3: Commit the version bump**

```bash
git add .claude-plugin/plugin.json CHANGELOG.md
git commit -m "chore: bump version to v0.2.0"
```

- [ ] **Step 4: Tag the release**

```bash
git tag -a v0.2.0 -m "v0.2.0 — scheduled scraper routine live"
```

- [ ] **Step 5: Push the commits and the tag**

```bash
git push origin main
git push origin v0.2.0
```

- [ ] **Step 6: Confirm on GitHub**

Open https://github.com/shantanudutta1/salesforce-flow-sense/releases
and confirm the v0.2.0 tag appears.

---

## Done

Routine is live and self-running. The maintainer's job from here is:

- Monday mornings: triage the GitHub Issue (~10 min)
- Cherry-pick keepers into the catalog via human-authored PRs
- First Monday of each month: dedupe pass + staleness audit (manual)
- Month 11: rotate the GitHub PAT (calendar reminder)
- After 4 weekly runs: evaluate against the v0.2.0 success criteria in the spec
