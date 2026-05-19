# Design: Scheduled Gotcha Scraper Routine (v0.2.0)

> **Status:** Approved (brainstorming phase complete)
> **Owner:** @shantanudutta1
> **Date:** 2026-05-20
> **Target release:** salesforce-flow-sense v0.2.0

---

## Problem

The salesforce-flow-sense plugin shipped in v0.1.0 with a static seed catalog of 17 gotchas, ported from a private Jack Links knowledge base. For the plugin to stay valuable over time, the catalog must grow — but manually researching the Salesforce community every week is the kind of recurring chore that gets dropped within a month.

We need an **automated, sustainable mechanism** that surfaces candidate gotcha entries weekly with high enough quality that the maintainer can curate in ~10 minutes per week.

## Goals

- **Surface 0–5 candidate gotcha entries per week** from the Salesforce community.
- **Pre-filter for quality**: every candidate must pass the "concrete trigger / non-obvious root cause / actionable prevention" gate before reaching the maintainer.
- **Pre-dedup against existing catalog and pending candidates**: don't waste maintainer time triaging duplicates.
- **Run sustainably**: zero ongoing infrastructure to maintain; uses the maintainer's Claude Pro allowance.
- **Self-monitor**: even on weeks with no new gotchas, post an "all clear" issue so the maintainer knows the routine ran.

## Non-goals

- **Not building a code-based scraper.** Per-source HTML parsers are a maintenance burden disproportionate to their value when Claude's WebSearch/WebFetch already work.
- **Not auto-merging into the catalog.** All catalog mutations remain human-authored. The routine only opens issues.
- **Not a Salesforce intelligence digest.** Out of scope: release notes summaries, blog highlights, market commentary. Single-purpose: find gotchas.
- **Not auto-archiving stale entries.** Staleness audits remain a manual monthly task by the maintainer.
- **Not multi-source delivery.** We pick GitHub Issue. No Slack/email/RSS for v0.2.0.

## Decisions (locked during brainstorming, 2026-05-20)

| # | Decision | Rationale |
|---|---|---|
| 1 | **Scope: narrow gotcha hunter.** Single-purpose, drafts in standard catalog format. | Decoupling from broader "SF intelligence" vision keeps v0.2.0 shippable. |
| 2 | **Worker: Claude-as-researcher via `/schedule` routine.** Not a code-based scraper, not a hybrid. | Right tool for Claude Code's native scheduling capability. No per-source HTML parsing. No new infra. |
| 3 | **Sources: all 4 tiers**, with weighted priority (75% budget on tiers 1–2, 25% on tiers 3–4). | Maximum coverage. Tier ordering acts as a quality filter in the prompt. |
| 4 | **Delivery: GitHub Issue per week.** Not a PR, not a local file. | Issues match the semantics of "candidates up for discussion." No branch overhead on bad weeks. Phone-friendly triage. |
| 5 | **Cadence: weekly, Monday 9am local time.** | Smaller batches are easier to triage. Aligns with MVP blog publication rhythm. Well within Pro allowance. |
| 6 | **Quality control: two-agent design with adversarial critic subagent.** Not in-prompt self-review, not ralph-loop, not /review skill dependency. | Independent of plugin skill availability in cloud routines (which is uncertain). Critic sees only drafts, so it judges them on the same evidence the maintainer would. |

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Anthropic Cloud — scheduled routine                         │
│                                                              │
│  Mon 9am local: trigger fires                                │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────────────────────────────────┐                │
│  │  RESEARCHER agent                        │                │
│  │   1. git clone (read-only)               │                │
│  │   2. List open candidate issues          │                │
│  │   3. Research 4 tiers in priority order  │                │
│  │   4. Draft N candidates (0-5)            │                │
│  └──────────────────────────────────────────┘                │
│         │ drafts                                             │
│         ▼                                                    │
│  ┌──────────────────────────────────────────┐                │
│  │  CRITIC subagent (dispatched fresh)      │                │
│  │   - sees only the drafts                 │                │
│  │   - adversarial: try to kill each entry  │                │
│  │   - returns KILL / EDIT / KEEP per entry │                │
│  └──────────────────────────────────────────┘                │
│         │ verdicts                                           │
│         ▼                                                    │
│  ┌──────────────────────────────────────────┐                │
│  │  RESEARCHER agent (resumes)              │                │
│  │   - apply verdicts: cut KILLs, fix EDITs │                │
│  │   - if 0 survivors: post "no new gotchas"│                │
│  │   - else: open GitHub Issue with kept    │                │
│  └──────────────────────────────────────────┘                │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
              GitHub Issue on shantanudutta1/salesforce-flow-sense
                  (label: candidate)
                            │
                            ▼
              Maintainer reviews (Mon morning, ~10 min)
                            │
                            ▼
              Manually open a PR for keepers — merge
```

### Key boundary

The routine **never writes to `reference/*.md` directly** — only opens issues. Integration into the catalog is always a human-authored PR. This keeps catalog quality gated by the maintainer, not the LLM.

## Components

### 1. Researcher agent (prompt-as-code)

The "code" of the routine. Runs every Monday. Full prompt:

```
You are a Salesforce automation gotcha curator for the salesforce-flow-sense
plugin. Your job: find non-obvious Flow/Apex/deployment pitfalls from
community sources and propose them as candidates.

## Sources (priority order)

Tier 1 — Ground truth:
  - Salesforce Known Issues board (search via WebSearch)
  - Latest Salesforce Release Notes

Tier 2 — MVP / architect blogs:
  - automationchampion.com (Rakesh Gupta)
  - unofficialsf.com
  - salesforceben.com (flow/apex tag)
  - jenwlee.com

Tier 3 — Q&A signal:
  - salesforce.stackexchange.com [flow] [apex] tags, newest first

Tier 4 — Trench reports:
  - reddit.com/r/salesforce — posts mentioning "gotcha", "got bit",
    "wasted a day"

## Process

1. Clone repo (read-only):
   git clone https://github.com/shantanudutta1/salesforce-flow-sense
2. Read skills/gotcha-lookup/reference/flow-pitfalls.md → know what's
   already catalogued.
3. gh issue list --label candidate --state open → know what's pending.
4. Source pass: spend ~75% of tool budget on tiers 1–2, ~25% on tiers 3–4.
   Focus on content from the last 14 days.
5. For each candidate, find: trigger / root cause / prevention / source URL.
6. Quality gate (reject if any fail):
   - Concrete trigger (specific scenario, not "sometimes")
   - Non-obvious root cause (would a competent dev guess wrong about WHY?)
   - Actionable prevention (checkable step, not "be careful")
   - Not already in catalog, not in open issue
   - Has a citable source URL
7. Cap at 5 candidates per run.
8. Hand off to critic subagent with drafts. Await verdicts.
9. Apply verdicts: cut KILLs, fix EDITs, keep KEEPs.
10. Post:
    - If 0 survive: issue "No new gotchas — YYYY-MM-DD" with sources scanned.
    - Else: issue "Gotcha candidates — YYYY-MM-DD", label "candidate",
      with drafts + critic verdicts.

## Output format (per entry)

### [next stable ID] — Short title
- **What happened**: [symptom]
- **Root cause**: [mechanism]
- **Prevention**: [checkable step]
- **Source(s)**: [URL(s)]
```

### 2. Critic subagent (prompt)

Dispatched by the researcher after drafts are complete. Receives only the drafts (no research context). Adversarial.

```
You are an adversarial reviewer of proposed gotcha entries. Find reasons
to KILL each one. Default bias: KILL or EDIT. KEEP only when genuinely strong.

For each entry return: KILL | EDIT | KEEP + reason / suggested edit.

Check:
1. Is the trigger concrete? (not "sometimes flows fail")
2. Is the root cause non-obvious? (if a dev would immediately know why,
   not a gotcha)
3. Is the prevention checkable? (not "be careful")
4. Is the entry hallucinated? Spot-check at least one cited URL.
5. Is it redundant within this batch?

The cost of a bad KEEP (junk in the catalog) is higher than the cost
of a false KILL (re-discovered next week).
```

### 3. GitHub authentication

One credential: a fine-grained GitHub Personal Access Token, scoped **only** to `shantanudutta1/salesforce-flow-sense`, with permissions:

- `Contents: Read` (for `git clone`)
- `Issues: Write` (for creating and labelling issues)

**Token expiry: 1 year.** Re-auth annually as hygiene. Stored as an encrypted environment variable in the routine config (managed by the `/schedule` skill).

### 4. Dedup mechanism

Two checks, **before** the source pass (so we don't burn tool budget researching duplicates):

1. **Catalog check:** Researcher reads `skills/gotcha-lookup/reference/flow-pitfalls.md` and builds an internal index of catalogued root causes (not just titles — same gotcha can be described with different titles).
2. **Pending check:** Researcher lists open `candidate` issues from the last 8 weeks: `gh issue list --label candidate --state open --limit 50`.

During drafting, each candidate is checked: "is this root cause already in (1) or (2)?" If yes, reject.

### 5. Failure handling

| Scenario | Behavior |
|---|---|
| 0 candidates met quality gate | Post "No new gotchas — YYYY-MM-DD" issue. Confirms routine ran. |
| One source unreachable (rate-limited, 503) | Try remaining 3 sources, note the unreachable source in the issue body |
| All 4 sources unreachable | Open issue with label `routine-failure` describing what failed. Self-heals next week. |
| GitHub API call fails when posting issue | Routine logs failure to its run log (visible via `/schedule` skill). Maintainer sees on next manual check. |
| Critic subagent fails to return | Researcher posts drafts unreviewed with banner: "⚠️ Critic agent did not respond — review manually" |

## Implementation flow (high-level)

This section describes the **deployment path**, not the in-routine logic. The detailed implementation plan is written separately via the `writing-plans` skill.

1. **Maintainer creates GitHub fine-grained PAT** scoped to `salesforce-flow-sense` with `Contents: Read` + `Issues: Write`. Expiry 1 year.
2. **Invoke `/schedule` skill** to create the weekly routine:
   - Cron: `0 9 * * 1` (Monday 9am local time)
   - Prompt: researcher prompt (Section: Components, Component 1)
   - Env var: `GITHUB_TOKEN` = the PAT
3. **Manual dry run** via `/schedule` to validate output before enabling cron.
4. **Iterate on prompt** based on dry run (expect 1–2 rounds).
5. **Enable the weekly cron.**
6. **Monitor for 3 weeks**; small adjustments until quality stabilizes.

## Risks & open questions

| # | Risk / Question | Mitigation |
|---|---|---|
| 1 | **Scheduled routine tool availability.** Uncertain whether `/schedule` routines have access to `gh` CLI and full WebFetch in the cloud sandbox. | Verify during dry run (step 3). Fallback: use GitHub REST API via `curl` instead of `gh` CLI. |
| 2 | **Subagent dispatch from a scheduled routine.** Unverified that researcher can dispatch a critic subagent from inside a cloud routine. | Verify during dry run. Fallback: collapse to in-prompt self-review (option R1 from brainstorming). |
| 3 | **Source URL reliability.** Some sources (Reddit, Stack Exchange) rate-limit anonymous scraping. | Failure-handling table covers this; degrades gracefully. |
| 4 | **Pro allowance consumption growth.** If the prompt expands to richer research over time, weekly cost could grow. | Monitor via Pro usage dashboard. Cap candidates at 5 per run to bound tool calls. |
| 5 | **Token expiry mid-year without notice.** | 1-year expiry with a calendar reminder at month 11 to rotate. Out of scope for v0.2.0 to automate rotation. |
| 6 | **Catalog growth changing the dedup prompt context size.** When `flow-pitfalls.md` reaches ~100 entries, reading the full file in context becomes meaningful tokens. | Defer to v0.3.0: introduce a generated `topics.md` index that the researcher can read instead of the full file. Not blocking for v0.2.0. |

## Out of scope (deferred)

- **`deployment-pitfalls.md` and `tooling-api-pitfalls.md` reference files** — v0.1.0 ships flow-pitfalls only; deployment + Tooling API land in v0.3.0+ once the catalog growth model is validated.
- **PR-based delivery** — could be added in v0.3.0+ if Issue triage proves to be a bottleneck.
- **Slack / email notifications** — defer until there's evidence the maintainer doesn't see Mondays issues.
- **Automatic catalog rotation / archival** — staleness audits remain manual monthly.

## Success criteria for v0.2.0

The routine is considered successful if:

1. It runs for **4 consecutive weeks** without manual intervention.
2. At least **2 of the 4 weeks** produce 1+ candidate that survives both the critic and the maintainer review (i.e., gets merged into the catalog).
3. **Zero false additions** to the catalog from hallucinated entries (caught by maintainer review).
4. Maintainer review time stays under **15 minutes per week** on average.

If 2+ of these miss after 4 weeks, the design is reconsidered (probably tightening the source list, the quality gate, or both).

## References

- Brainstorming session: this conversation, 2026-05-20
- Plugin v0.1.0 release: https://github.com/shantanudutta1/salesforce-flow-sense/releases/tag/v0.1.0
- Existing catalog: `skills/gotcha-lookup/reference/flow-pitfalls.md`
- Claude Code `/schedule` skill (built-in, manages cron routines on Anthropic infrastructure)
