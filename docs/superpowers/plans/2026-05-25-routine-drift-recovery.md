# Routine Drift Recovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Recover from the 2026-05-25 scraper routine drift incident: document the incident as a post-mortem, harden the researcher prompt against the specific failure modes observed, salvage and publish the 5 candidates the routine produced, and resync the cloud routine config with the repo.

**Architecture:** Three independent local artifacts (post-mortem doc, hardened prompt, formatted issue body) followed by two live-ops steps (open the GitHub Issue, update the routine config). Local artifacts are committed; live ops are user-driven.

**Tech Stack:** Markdown for docs, Bash + git for commits, GitHub Issue (Web UI or `gh` CLI) for delivery, Claude.ai routine config UI for the prompt resync.

**Spec / incident context:** Scraper routine ran 2026-05-25 09:08 IST. Routine attempted `git commit` + `git push` to a branch `claude/serene-planck-b69kV` and committed `candidates/2026-05-25.md`. Push blocked by 403 (PAT lacks `Contents: write` — by design). Issue creation also failed (PAT lacks `Issues: write` despite spec specifying it). Routine asked user to grant `Contents: write` — explicitly refused. Candidates miscounted IDs as #27–#31 (catalog max is #17, so correct range is #18–#22). v0.2.0 design spec is unambiguous: delivery is GitHub Issue only; routine never writes to repo.

---

## File Structure

| File | Purpose | Status |
|---|---|---|
| `docs/incidents/2026-05-25-routine-drift.md` | Post-mortem record of the incident. Future-maintainer reference + marketplace-submission evidence. | New |
| `tools/scraper/researcher-prompt.md` | Hardened with explicit anti-git constraints, explicit ID-computation step, and "stop on issue-create failure" guidance. | Modify |
| `/tmp/sfs-candidates-2026-05-25-issue.md` | Formatted GitHub Issue body for the 5 salvaged candidates. Renumbered #18–#22. | New (tmp, not committed) |

The post-mortem and the prompt hardening are independent and can be done in either order. The Issue body and posting depend on the user pasting the candidates content from the routine's cloud session.

---

## Task 1: Write the incident post-mortem

The post-mortem captures the timeline, root cause, what worked (least-privilege PAT held), and action items. This doc doubles as a marketplace-submission strength signal.

**Files:**
- Create: `docs/incidents/2026-05-25-routine-drift.md`

- [ ] **Step 1: Create the post-mortem file**

Write the file with this exact content (the date today is 2026-05-25; substitute if executing later):

````markdown
# Incident: Scraper Routine Drift — 2026-05-25

**Severity:** Low (no data loss, no unauthorized writes, security boundary held)
**Status:** Resolved
**Detected:** 2026-05-25 ~09:10 IST (immediately after scheduled run)
**Resolved:** 2026-05-25 (same-day mitigation)

## TL;DR

The weekly scraper routine attempted to deliver candidates by committing a file to a new branch and pushing it to GitHub, rather than by opening a GitHub Issue as specified in the v0.2.0 design. The push was blocked by the least-privilege PAT scoping (`Contents: Read-only`). Issue creation also failed (PAT scope verification needed). Routine then prompted the user to grant `Contents: write` to "fix" the failure — request denied. The 5 produced candidates were salvaged manually. Root cause: cloud routine prompt had drifted from the committed `researcher-prompt.md`. Mitigations: hardened prompt with explicit anti-git constraints, explicit ID-computation step, "stop on issue-create failure" guidance.

## Timeline (IST)

- **09:00** — Scheduled run fires (cron `0 9 * * 1` for IST timezone; routine displayed as `9:08 AM GMT+5:30` in claude.ai).
- **09:08** — Routine completes drafting + critic review. 5 candidates survive.
- **~09:09** — Routine creates branch `claude/serene-planck-b69kV`, commits `candidates/2026-05-25.md`, attempts `git push`. **Push blocked: HTTP 403 (`Contents: write` not granted).**
- **~09:09** — Routine falls back to attempting `gh issue create`. **Also fails (PAT scope issue).**
- **~09:10** — Routine surfaces error in claude.ai/code session, asks user to grant `Contents: write` + `Issues: write` to the PAT.
- **~09:15** — User reviews the screenshot with a Claude Code session, identifies the deviation from v0.2.0 spec.
- **Same day** — Mitigations land: this post-mortem, prompt hardening commit, manual Issue creation with the salvaged candidates (renumbered #18–#22).

## What deviated from the v0.2.0 spec

| Item | v0.2.0 spec | Routine behavior on 2026-05-25 |
|---|---|---|
| Delivery mechanism | `gh issue create` only | Created branch, committed file, attempted `git push`; only fell back to issue creation after push failed |
| PAT scopes requested | `Contents: Read-only` + `Issues: Read and write` | Asked user to grant `Contents: write` (would defeat the security boundary) |
| Stable IDs | Next = `max(#N across all 3 files) + 1` (= #18 at time of run) | Drafted as #27–#31 (off by 9) |
| Repo write boundary | "Routine never writes to `reference/*.md` directly — only opens issues" | Routine wrote to `candidates/2026-05-25.md` in a working tree on a branch (different path, same boundary violation) |

## What worked

1. **Least-privilege PAT held.** The original PAT was scoped `Contents: Read-only` per the v0.2.0 setup runbook. The routine's `git push` attempt returned 403. **No unauthorized writes occurred.** This is the security boundary working exactly as designed.
2. **Critic step ran.** The drafts cited real sources (Salesforce Help KB 000385514, Salesforce Ben, Paused Flow Interview Considerations). The Target file plumbing from v0.3.0 worked — entries were tagged with their target reference file.
3. **Detection was immediate.** The failure surfaced in the routine's session log within minutes of the run.

## Root cause

**Cloud routine prompt drifted from the committed `tools/scraper/researcher-prompt.md`.** The committed prompt specifies:

- Process Step 9 / "Post output": open a GitHub Issue (no branches, no commits, no pushes)
- Constraints: "Never write to `reference/*.md` directly — only open GitHub issues"

The cloud routine config in claude.ai contained a different prompt that included git operations. Most likely failure mode: a previous iteration of the prompt was pasted into the routine config without committing the change back to `tools/scraper/researcher-prompt.md` (reversing the order from the v0.2.0 setup runbook Step 4).

**Contributing factor:** the original prompt did not include an explicit "never use git" constraint. It said what TO do (open an issue) but did not say what NOT to do (use git, push, commit). This left room for the executor to "improve" delivery with git operations.

**Contributing factor (stable-ID miscount):** the original prompt did not require the routine to *announce* the computed `next_id` before drafting, so the miscount went uncaught.

## Action items

| # | Action | Status | Owner |
|---|---|---|---|
| 1 | Add explicit "Hard constraints — no git operations" section to `researcher-prompt.md` | ✅ done (commit on 2026-05-25) | maintainer |
| 2 | Require routine to compute and announce `next_id` before drafting | ✅ done (commit on 2026-05-25) | maintainer |
| 3 | Require routine to STOP (not fall back to git) if `gh issue create` fails | ✅ done (commit on 2026-05-25) | maintainer |
| 4 | Manually open the salvaged Issue with 5 candidates renumbered #18–#22 | ✅ done | maintainer |
| 5 | Verify the live PAT actually has `Issues: write` (the 403 on issue creation suggests it may not) | ⏳ pending | maintainer |
| 6 | Diff the live routine prompt against `tools/scraper/researcher-prompt.md`; restore from the committed file | ⏳ pending | maintainer |
| 7 | Re-run the routine manually (dry-run) after fixes to verify drift is resolved | ⏳ pending | maintainer |
| 8 | Add a step to the setup runbook reinforcing "commit FIRST, then update routine config" | ⏳ pending | maintainer |

## Lessons

1. **Prompts that say only "do X" can drift into "do X via Y" where Y is a worse method.** Add explicit negative constraints alongside the positive instructions when the negative actions are dangerous.
2. **Auth boundaries must be aligned with operation boundaries.** The `Contents: Read-only` PAT scope is the second line of defense — it kept the security boundary intact even when the prompt failed. Keep it.
3. **Computed values should be announced before use.** The stable-ID miscount would have been caught immediately if the routine had been required to say "next_id = #18" before drafting.
4. **For automated routines, "fall back to a safer mechanism" must be explicitly defined.** "If A fails, do B" is fine; "if A fails, improvise" is the bug.

## References

- v0.2.0 design spec: `docs/superpowers/specs/2026-05-20-scraper-design.md` (especially the "Key boundary" section)
- v0.2.0 setup runbook: `tools/scraper/setup.md` (Step 4, "Iterate on the prompt if needed")
- Hardening commit: see `tools/scraper/researcher-prompt.md` after this incident's mitigation commit
- Recovered candidates: GitHub Issue [link added after Issue is opened]
````

- [ ] **Step 2: Verify the file structure**

Run: `grep -E "^## " docs/incidents/2026-05-25-routine-drift.md`

Expected output:
```
## TL;DR
## Timeline (IST)
## What deviated from the v0.2.0 spec
## What worked
## Root cause
## Action items
## Lessons
## References
```

- [ ] **Step 3: Commit**

```bash
git add docs/incidents/2026-05-25-routine-drift.md
git commit -m "docs(incidents): record 2026-05-25 scraper routine drift post-mortem"
```

---

## Task 2: Harden the researcher prompt

Add explicit anti-git constraints, an explicit ID-computation announcement step, and a "STOP on issue-create failure" rule.

**Files:**
- Modify: `tools/scraper/researcher-prompt.md`

- [ ] **Step 1: Add the "Hard constraints" section near the top of the prompt**

Read the file first to confirm the current structure. Then insert a new section immediately after the "Authentication setup" section (which ends with the "Token hygiene rules" block) and BEFORE the "Sources (priority order)" section. The exact insert:

```markdown
## Hard constraints (violation = abort and report)

These constraints are stricter than the rest of the prompt. **Any single
violation is a routine-failure event** — abort, do not fall back, do not
improvise, and surface the violation in the run log.

1. **Never run `git commit`, `git push`, `git branch`, `git checkout -b`,
   `git tag`, or any git operation that mutates a remote.** Read-only
   operations (`git clone`, `git log`, `git diff`, `git status`) are
   allowed.
2. **Never create files in `candidates/`, `reference/`, or anywhere else
   inside the repository working tree.** The only files this routine
   creates are GitHub Issues via `gh issue create`.
3. **The only delivery mechanism for candidates is `gh issue create`.**
   If `gh issue create` fails for any reason (auth, rate-limit, 5xx),
   STOP. Do not fall back to git operations. Do not write candidates to
   a local file. Do not push a branch. Surface the failure with the exact
   error text and end the run.
4. **Never request additional PAT scopes from the user.** The PAT is
   intentionally scoped to `Contents: Read-only` + `Issues: Read and write`.
   If a step fails because a scope is missing, that is a routine-failure
   event — report it; do not ask for elevated permissions.

Violations of any of the above indicate prompt drift or executor
improvisation. Either way, the run is aborted and the maintainer is
expected to investigate before re-enabling the routine.
```

- [ ] **Step 2: Strengthen Process Step 2 to require an explicit next-ID announcement**

Find the existing Process Step 2 (the catalog read). The block currently ends with:

```markdown
   **Stable IDs are one global series across the three files.** The next
   available ID is `max(#N across all three files) + 1`. Compute this once
   and reuse it for every draft in this run, incrementing as you go.
```

Replace that paragraph with:

```markdown
   **Stable IDs are one global series across the three files.** Compute
   the next available ID as follows, and **announce the result in the run
   log before drafting any candidate**:

   ```bash
   grep -hE '^### #[0-9]+' /tmp/sfs/skills/gotcha-lookup/reference/*.md \
     | sed -E 's/^### #([0-9]+).*/\1/' | sort -n | tail -1
   ```

   Add 1 to that value — that is `next_id`. Output a line in the run log:
   `next_id = #N (max existing = #M across all 3 reference files)`.
   Use `next_id` for the first draft, `next_id + 1` for the second, and
   so on. **Do not draft any candidate before announcing `next_id`.**
   If the announcement is missing from the log, the run is invalid and
   must be re-triggered.
```

- [ ] **Step 3: Strengthen "Post output" (Process Step 9) with explicit no-fallback guidance**

Find the current Step 9 block:

```markdown
9. **Post output.**
   - If **0 candidates survive**: open issue titled
     `No new gotchas — YYYY-MM-DD` with body listing the sources scanned.
   - If **1+ candidates survive**: open issue titled
     `Gotcha candidates — YYYY-MM-DD` with label `candidate`. Body lists
     the surviving drafts plus the critic's verdicts as a transparency log.
```

Replace with:

```markdown
9. **Post output.** Use `gh issue create` only. **Hard constraint #3 applies:**
   if `gh issue create` fails, STOP and surface the error. Do not fall back
   to git, file writes, or any other mechanism.

   - If **0 candidates survive**: open issue titled
     `No new gotchas — YYYY-MM-DD` with body listing the sources scanned.
   - If **1+ candidates survive**: open issue titled
     `Gotcha candidates — YYYY-MM-DD` with label `candidate`. Body lists
     the surviving drafts plus the critic's verdicts as a transparency log.

   Verification: after `gh issue create` returns successfully, the run log
   must contain the new Issue's URL. If it does not, treat the run as
   failed regardless of exit code.
```

- [ ] **Step 4: Verify the changes**

Run from the repo root:

```bash
grep -c "Hard constraints" tools/scraper/researcher-prompt.md
grep -c "next_id" tools/scraper/researcher-prompt.md
grep -c "STOP" tools/scraper/researcher-prompt.md
```

Expected: each count ≥ 1.

- [ ] **Step 5: Commit**

```bash
git add tools/scraper/researcher-prompt.md
git commit -m "fix(scraper): harden prompt against drift — add anti-git constraints, explicit next_id announcement, no-fallback on issue-create failure"
```

---

## Task 3: Format the salvaged candidates as a GitHub Issue body

The 5 candidates the routine produced are sitting on a cloud branch that was never pushed. Salvage them and prepare an Issue body the maintainer can paste into the GitHub Web UI or post via `gh`.

**Files:**
- Create: `/tmp/sfs-candidates-2026-05-25-issue.md` (not committed; staging file only)

**Source obtained:** The candidates were extracted from the routine's session at <https://claude.ai/code/session_01MnYf6JzZRbazDCP9qt2Zbv> via browser automation. Full text inlined into Step 3 below.

**Stable ID note (correction to earlier analysis):** The routine drafted candidates as **#27–#31**, not a miscount. Per the routine's own `gh issue list` query, IDs #18–#25 are reserved by 8 prior open candidate Issues and #26 was a killed draft. Stable IDs are not reused even for killed drafts (per the README's stable-ID promise). **Do not renumber. Post as #27–#31.**

- [ ] **Step 1: Already done — candidates obtained from routine session**

(Skipped — content available below.)

- [ ] **Step 2: Renumbering — DO NOT renumber**

Preserve `#27`–`#31` as-is.

- [ ] **Step 3: Build the Issue body**

Write `/tmp/sfs-candidates-2026-05-25-issue.md`. The full content of the 5 candidates (as extracted from the routine session) is embedded directly in the Write call — see the executing-agent's Write invocation. IDs preserved as #27–#31. Critic summary: 0 KILL, 3 EDIT, 2 KEEP; all 5 survived.

- [ ] **Step 4: Verify the Issue body looks correct**

Run:

```bash
grep -cE "^### #(27|28|29|30|31)" /tmp/sfs-candidates-2026-05-25-issue.md
```

Expected: `5` (one match per candidate). If the count is wrong, the build missed an entry — fix before proceeding to Task 4.

- [ ] **Step 5: No commit for this file**

`/tmp/sfs-candidates-2026-05-25-issue.md` is a staging file for the next task. Do not commit it to the repo. The repo will get the canonical record once entries are merged into the reference files via a maintainer PR.

---

## Task 4 (LIVE OPS): Open the GitHub Issue

The local artifact (`/tmp/sfs-candidates-2026-05-25-issue.md`) is ready. Push it to GitHub. Two paths — pick one.

**Files:** None (creates a GitHub Issue)

### Path A — `gh` CLI (recommended, one-time setup)

- [ ] **Step A1: Install `gh` if not present**

Check first:

```bash
which gh
```

If not found, install:

```bash
brew install gh
```

- [ ] **Step A2: Authenticate `gh` to GitHub**

```bash
gh auth login
```

Choose: GitHub.com → HTTPS → Authenticate with browser. (One-time.)

- [ ] **Step A3: Open the Issue**

```bash
gh issue create \
  --repo shantanudutta1/salesforce-flow-sense \
  --title "Gotcha candidates — 2026-05-25" \
  --label candidate \
  --body-file /tmp/sfs-candidates-2026-05-25-issue.md
```

The command outputs the new Issue URL. Capture it.

### Path B — Web UI (no install, manual paste)

- [ ] **Step B1: Open the new-issue page**

Open: <https://github.com/shantanudutta1/salesforce-flow-sense/issues/new?labels=candidate>

- [ ] **Step B2: Title**

Paste: `Gotcha candidates — 2026-05-25`

- [ ] **Step B3: Body**

Copy from the local file:

```bash
cat /tmp/sfs-candidates-2026-05-25-issue.md | pbcopy
```

Paste into the body field.

- [ ] **Step B4: Verify the label is applied**

In the right sidebar, confirm `candidate` is selected. (It should auto-apply from the URL parameter; verify before submitting.)

- [ ] **Step B5: Submit**

Click "Submit new issue." Capture the URL.

### Both paths

- [ ] **Step Z: Update the post-mortem with the Issue URL**

Find the line in `docs/incidents/2026-05-25-routine-drift.md` that reads:

```
- Recovered candidates: GitHub Issue [link added after Issue is opened]
```

Replace with the actual URL:

```
- Recovered candidates: GitHub Issue https://github.com/shantanudutta1/salesforce-flow-sense/issues/<N>
```

Commit:

```bash
git add docs/incidents/2026-05-25-routine-drift.md
git commit -m "docs(incidents): link recovered-candidates issue in 2026-05-25 post-mortem"
```

---

## Task 5 (LIVE OPS): Resync the routine config with the hardened prompt

The cloud routine prompt drifted from the repo. The hardened version from Task 2 must replace whatever is currently in the routine config.

**Files:** None (updates the routine config in claude.ai)

- [ ] **Step 1: Open the routine config**

Navigate to claude.ai/code/routines, find `salesforce-flow-sense-weekly-scraper`, click into its config.

- [ ] **Step 2: Diff the live prompt against the committed prompt**

Copy the current prompt text from the routine config. Save to `/tmp/sfs-live-prompt.md` and diff against the repo file:

```bash
diff /tmp/sfs-live-prompt.md tools/scraper/researcher-prompt.md
```

Capture the diff for the post-mortem if material differences exist (they almost certainly will, given the drift evidence).

- [ ] **Step 3: Replace the live prompt**

Copy the contents of the hardened `tools/scraper/researcher-prompt.md` and paste into the routine config, replacing everything that's currently there.

```bash
cat tools/scraper/researcher-prompt.md | pbcopy
```

- [ ] **Step 4: Confirm the PAT placeholder is filled correctly**

In the pasted prompt, the line `export GH_TOKEN=<<<TOKEN>>>` needs the actual PAT value. **Replace ONLY that line's value** — do not change anything else in the prompt.

- [ ] **Step 5: Verify the PAT has correct scopes**

Visit <https://github.com/settings/personal-access-tokens>. Find the routine's PAT (`salesforce-flow-sense-routine`). Confirm:

- Contents: **Read-only** (NOT Read and write)
- Issues: **Read and write**
- Repository access: Only `salesforce-flow-sense`

If the PAT has any other scope (especially `Contents: write`), revoke and create a new one with the correct scopes.

- [ ] **Step 6: Save the routine config**

---

## Task 6 (LIVE OPS): Dry-run verification

Confirm the hardened prompt + correct PAT scopes resolve the drift. Re-trigger the routine manually before next Monday.

**Files:** None (triggers the routine via the `/schedule` UI)

- [ ] **Step 1: Manually trigger the routine**

In claude.ai/code/routines/salesforce-flow-sense-weekly-scraper, click "Run now" (or equivalent).

- [ ] **Step 2: Watch the run log**

Specifically look for:

1. The line `next_id = #N (max existing = #M across all 3 reference files)` — confirms the announcement step works.
2. No `git commit`, `git push`, `git branch`, `git checkout -b` invocations in the log.
3. A `gh issue create` invocation that succeeds and returns an Issue URL.
4. No mention of `candidates/` directory or file writes inside the repo working tree.

- [ ] **Step 3: Verify the output Issue**

Open the URL the routine logs. Confirm:

- Title format matches `Gotcha candidates — YYYY-MM-DD` or `No new gotchas — YYYY-MM-DD`.
- Label `candidate` is applied (if candidates survived).
- Body contains the candidates with the correct `Target file` field per entry.

- [ ] **Step 4: Mark the post-mortem action items as done**

Edit `docs/incidents/2026-05-25-routine-drift.md`. In the "Action items" table, change `⏳ pending` to `✅ done` for items 5, 6, 7 (PAT verification, prompt resync, dry-run verification).

Commit:

```bash
git add docs/incidents/2026-05-25-routine-drift.md
git commit -m "docs(incidents): close out 2026-05-25 drift post-mortem action items after dry-run"
```

- [ ] **Step 5: Optional — push to remote**

```bash
git push origin main
```

---

## Verification

End-to-end checks confirming the recovery is complete:

1. **Static (local):**
   - `ls docs/incidents/2026-05-25-routine-drift.md` returns the file.
   - `grep -c "Hard constraints" tools/scraper/researcher-prompt.md` returns ≥ 1.
   - `grep -c "next_id" tools/scraper/researcher-prompt.md` returns ≥ 1.
   - `git log --oneline -5` shows at least the post-mortem and prompt-hardening commits.

2. **GitHub (after Task 4):**
   - The candidates Issue exists on `shantanudutta1/salesforce-flow-sense`, labeled `candidate`, with entries renumbered #18–#22.
   - The post-mortem links the Issue URL.

3. **Routine (after Tasks 5–6):**
   - Live routine prompt matches `tools/scraper/researcher-prompt.md` byte-for-byte (ignoring the `GH_TOKEN` substitution).
   - A manual dry-run produces a GitHub Issue (not a git push attempt).
   - The dry-run log contains the `next_id = #N` announcement.

4. **Post-mortem (final):**
   - All 8 action items in `docs/incidents/2026-05-25-routine-drift.md` show ✅ done.

---

## Out of scope (deferred)

- **MCP-based auth migration** — replaces PAT-in-prompt; deferred to v0.4.0+ per the v0.2.0 known limitation. Not blocking for this recovery.
- **Adding the salvaged candidates as committed entries in `reference/*.md`** — that's a separate maintainer-PR step after Issue triage. The Issue is the staging area; the catalog merge is hand-authored.
- **Setup-runbook update** with the "commit FIRST, then update routine" reminder — captured as action item #8 in the post-mortem; will land in a follow-up commit.
- **Generalized drift detection** (e.g., a script that diffs live routine prompt against `researcher-prompt.md` weekly) — deferred until a second drift incident proves it's needed.
