---
description: Run only the fable-flow Review phase — parallel high-effort Opus 4.8 reviewers (correctness, fidelity, integration) verify a branch's diff against the plan; findings are reported, and fixed only with --fix.
argument-hint: "[--base REF] [--reviewers N] [--rounds N] [--fix]"
disable-model-invocation: true
---

## Context

- Branch: !`git branch --show-current`
- HEAD: !`git rev-parse --short HEAD`

## Your job

Run the Review phase of the fable-flow pipeline against the current branch.

**Input:** $ARGUMENTS  (`--base REF` diff base, default: the base recorded in `.fable-flow/plan.md`, else the default branch · `--reviewers N` lens count, default 3 with a plan that has multiple tracks, else 2 · `--rounds N` max review→fix cycles when `--fix`, default 2 · `--fix` apply fixes for actionable findings; without it this command only reports)

1. Resolve the base ref. Load `.fable-flow/plan.md` and `.fable-flow/task.md` if present — reviewers verify against them; if absent, reviewers verify against the diff's own apparent intent and say so.
2. Spawn `fable-flow:reviewer` subagents **in a single message**, lenses in priority order `correctness`, `fidelity`, `integration`. Each prompt: the requirements, the plan (if any), the base ref, its lens. Save reports to `.fable-flow/review-round-<r>-<lens>.md`.
3. Triage: **actionable** = severity blocker/major with confidence medium+. Everything else is reported but never fix-looped.

Without `--fix`, the deliverable is the assessment: report the verdicts and findings ranked most-severe first, with each reviewer's execution evidence, and stop — don't change any files.

With `--fix`: fix actionable findings on the current branch (directly, or via a `fable-flow:implementer` based on the current SHA for larger fixes), re-run the plan's integration verification, then re-spawn only the lenses that blocked. At most `--rounds` cycles; whatever survives is reported, not silently retried.

Lead the report with the overall verdict. Quote real command output for any claim that tests pass. If reviewers disagree, present both positions rather than averaging them.
