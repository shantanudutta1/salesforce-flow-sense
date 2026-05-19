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
