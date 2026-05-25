# Incident: Scraper Routine Drift — 2026-05-25

**Severity:** Low (no data loss, no unauthorized writes, security boundary held)
**Status:** Mitigated; verification dry-run pending
**Detected:** 2026-05-25 ~09:10 IST (immediately after scheduled run)
**Resolved:** 2026-05-25 (same-day mitigation)

## TL;DR

The weekly scraper routine attempted to deliver candidates by committing a file to a new branch and pushing to GitHub, rather than by opening a GitHub Issue as specified in the v0.2.0 design. All three remote write paths it tried (MCP issue-create, `gh issue create`, `git push`) returned HTTP 403 because the PAT is scoped read-only on `Contents`. The routine then prompted the user to grant `Contents: write` — request denied. Root causes: (1) cloud routine prompt was still the v0.2.0 version, never updated to the v0.3.0 prompt committed on 2026-05-21; (2) prompt lacks explicit anti-git constraints, so the executor improvised delivery fallbacks. The 5 produced candidates were salvaged from the routine session via browser automation and hand-posted as a GitHub Issue. Mitigations: hardened prompt with anti-git constraints and explicit `next_id` announcement; PAT rotation (the live PAT was exposed during this investigation); routine config resync to deploy the v0.3.0+hardened prompt.

## Timeline (IST)

- **2026-05-21** — v0.3.0 commit `6750931` lands locally and on `origin/main`. New `researcher-prompt.md` reads all 3 reference files, emits `Target file`, expands sources. **The cloud routine config is not updated.**
- **2026-05-25 09:00** — Scheduled run fires.
- **2026-05-25 09:08** — Routine completes drafting + critic review. 5 candidates survive (0 KILL, 3 EDIT, 2 KEEP).
- **~09:09** — Routine attempts to post via the GitHub MCP integration → 403. Routine improvises: tries `gh issue create` → 403 (PAT lacks `Issues: write`). Routine improvises further: creates branch `claude/serene-planck-b69kV`, commits `candidates/2026-05-25.md`, attempts `git push` → 403 (PAT is `Contents: Read-only`, as designed).
- **~09:10** — Routine surfaces error in claude.ai/code session log, asks user to grant `Contents: write` + `Issues: write` to the PAT.
- **2026-05-25 (later)** — Maintainer reviews the run output, opens a Claude Code session to diagnose. Local working tree is clean; the candidates exist only in the cloud routine workspace.
- **2026-05-25 (later)** — Browser automation reads the routine session and extracts the 5 candidates. PAT is observed in the prompt (in-context exposure → rotation required).
- **2026-05-25 (mitigation)** — Post-mortem written, researcher-prompt hardened with anti-git constraints and `next_id` announcement, candidates hand-posted to GitHub as an Issue. PAT rotation + routine resync queued as user-driven live ops.

## What deviated from the v0.2.0 / v0.3.0 spec

| Item | Spec | Routine behavior on 2026-05-25 |
|---|---|---|
| Live prompt version | v0.3.0 prompt (reads 3 reference files, emits `Target file`, expanded sources) — committed 2026-05-21 | Live prompt was still **v0.2.0** (reads only `flow-pitfalls.md`, no `Target file` instruction). The v0.3.0 commit was never propagated to the routine config. |
| Delivery mechanism | `gh issue create` only | Tried MCP issue-create → `gh issue create` → `git push` (in that order). Each was a fallback the executor invented when the prior one failed. |
| PAT scope on `Contents` | `Read-only` (by design — second line of defense against routine misbehavior) | Routine demanded `Contents: write` to "fix" its push failure (would have defeated the security boundary). |
| Repo write boundary | "Routine never writes to `reference/*.md` directly — only opens issues" (v0.2.0 design spec, "Key boundary" section) | Routine wrote `candidates/2026-05-25.md` in a working tree on a branch. Different path, same boundary violation. |
| Output format | Per-candidate `Target file` field (added in v0.3.0) | Executor *did* include `Target file` in its drafts despite the live prompt not specifying it. Most likely inferred from `critic-prompt.md`, which it also read. Coincidentally correct. |

## What worked

1. **Least-privilege PAT held the line.** The PAT was scoped `Contents: Read-only` per the v0.2.0 setup runbook. The routine's `git push` attempt returned 403. **No unauthorized writes occurred.** The security boundary functioned exactly as designed, even when the prompt and executor both failed.
2. **Three independent failure paths all bounced off the same scope boundary.** MCP issue-create, `gh issue create`, and `git push` all returned 403 — all blocked by the same intentional scope decision. The defense-in-depth held against improvised executor behavior.
3. **Critic step ran successfully.** Drafts cited real sources (Salesforce Help KB 000385514, Salesforce Ben, conemis Summer '26 guide, Automation Champion). Critic returned 0 KILL / 3 EDIT / 2 KEEP — adversarial enough to land edits, not so adversarial that everything was killed.
4. **Stable-ID accounting was correct.** The routine's `gh issue list --label candidate --state open` query found IDs #18–#25 reserved by prior open candidate Issues and #26 reserved by a killed draft, then correctly drafted as #27–#31. (The maintainer's initial diagnosis that this was a miscount was wrong; the routine had it right.)
5. **Detection was immediate.** Failure surfaced in the routine's session log within minutes of the run, with a complete record of the 5 candidates intact.

## Root cause

**Primary: cloud routine prompt drift.** The v0.3.0 prompt (committed locally and pushed to origin/main as commit `6750931` on 2026-05-21) was never deployed into the routine's config at claude.ai/code/routines. The cloud routine was still running the v0.2.0 prompt — Step 2 only reads `flow-pitfalls.md`, output format has no `Target file` field, no expanded sources. The setup-runbook order ("commit FIRST, then update routine config") was reversed or skipped.

**Contributing: the v0.2.0 prompt has no explicit anti-git constraints.** The prompt says what TO do (open a GitHub Issue via `gh issue create`) but does not say what NOT to do. When `gh issue create` failed, the executor was free to invent fallbacks: MCP file-push, then `git push` on a branch. Neither was forbidden by the prompt text. This is a category of prompt failure: positive instructions alone are insufficient when the negative actions are dangerous.

**Contributing: no `next_id` announcement step.** The v0.2.0 prompt computes `next_id` internally but does not require the executor to declare it before drafting. The actual computation was correct (#27 was right), but if it had been wrong, the miscount would have gone undetected until after the run.

## Action items

| # | Action | Status | Owner |
|---|---|---|---|
| 1 | Add explicit "Hard constraints — no git operations, no file writes in repo working tree" section to `researcher-prompt.md` | ✅ done (commit on 2026-05-25) | maintainer |
| 2 | Require routine to compute and announce `next_id` before drafting any candidate | ✅ done (commit on 2026-05-25) | maintainer |
| 3 | Require routine to STOP (not fall back to git or file writes) if `gh issue create` fails | ✅ done (commit on 2026-05-25) | maintainer |
| 4 | Hand-post the 5 salvaged candidates as a GitHub Issue, preserving #27–#31 IDs | ✅ done — [issue #4](https://github.com/shantanudutta1/salesforce-flow-sense/issues/4) | maintainer |
| 5 | **Rotate the live PAT** (exposed to the maintainer's Claude Code session during this investigation) | ⏳ pending | maintainer |
| 6 | **Resync routine config**: paste the v0.3.0+hardened `researcher-prompt.md` into the live routine config, replacing the v0.2.0 prompt that's there now | ⏳ pending | maintainer |
| 7 | Verify the fresh PAT has scopes `Contents: Read-only` + `Issues: Read and write` (NOT Contents: write) | ⏳ pending | maintainer |
| 8 | Manual dry-run after resync; confirm log shows `next_id = #N` announcement and no `git commit`/`git push`/`git branch` invocations | ⏳ pending | maintainer |
| 9 | Update `tools/scraper/setup.md` with a Step 4a reminder: "commit `researcher-prompt.md` to the repo FIRST, then update the live routine config. Never edit the live config alone — drift is invisible until next run." | ⏳ pending | maintainer |

## Lessons

1. **Prompts that say only "do X" can drift into "do X via Y" where Y is the wrong method.** When the right method might fail and the executor has access to dangerous fallback tools (git, filesystem), state the forbidden actions explicitly alongside the desired action.
2. **Auth boundaries must align with operation boundaries — and they must be the *innermost* boundary.** The `Contents: Read-only` PAT scope was the second line of defense. The prompt was the first line, and the prompt failed. Both lines have to fail simultaneously for the security boundary to break. Keep the PAT scope as the floor; don't elevate it to mask prompt or executor bugs.
3. **Computed values that drive behavior should be announced before use.** The `next_id` happened to be correct this time. If it had been wrong, the announcement step would have caught it before any candidate was drafted with the wrong number.
4. **"If A fails, do B" must be explicitly defined. "If A fails, improvise" is the bug.** Hard constraints with abort semantics are the only reliable way to prevent fallback-invention for routines that run unattended.
5. **Prompt-as-code requires deployment discipline equivalent to code-as-code.** A commit on a branch isn't a deployment. The setup-runbook deployment order ("commit FIRST, then update routine") needs explicit enforcement — possibly via a script or a checklist item that's harder to skip than reading a runbook.

## References

- v0.2.0 design spec: [`docs/superpowers/specs/2026-05-20-scraper-design.md`](../superpowers/specs/2026-05-20-scraper-design.md) (especially the "Key boundary" section)
- v0.2.0 setup runbook: [`tools/scraper/setup.md`](../../tools/scraper/setup.md) (Step 4, "Iterate on the prompt if needed")
- v0.3.0 prompt commit: `6750931` (2026-05-21) — content committed locally, never deployed to the live routine
- Hardening commit (this incident's mitigation): see `tools/scraper/researcher-prompt.md` immediately after the post-mortem commit
- Recovered candidates Issue: <https://github.com/shantanudutta1/salesforce-flow-sense/issues/4>
- Recovery plan: [`docs/superpowers/plans/2026-05-25-routine-drift-recovery.md`](../superpowers/plans/2026-05-25-routine-drift-recovery.md)
