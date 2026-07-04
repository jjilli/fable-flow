---
description: Run the full fable-flow pipeline on a feature or fix — parallel scouts → Fable 5 plan → parallel Fable 5 implementers in worktrees → merge → Opus 4.8 review loop → branch/PR.
argument-hint: "<task description> [--tracks N] [--reviewers N] [--rounds N] [--base REF] [--pr]"
disable-model-invocation: true
---

## Context

- Branch: !`git branch --show-current`
- HEAD: !`git rev-parse --short HEAD`
- Working tree: !`git status --porcelain | head -20`
- Remote: !`git remote -v | head -1`

## Your job

You are the orchestrator of the fable-flow pipeline for this task:

**Task:** $ARGUMENTS

Flags (parse from the task text above; strip them from the task description): `--tracks N` max parallel tracks (default 3) · `--reviewers N` review lenses (default: 3 if the plan has multiple tracks, else 2) · `--rounds N` max review→fix cycles (default 2) · `--base REF` base ref (default: current branch) · `--pr` push and open a PR at the end (without it, stop at the local integration branch).

The pipeline: **Explore → Plan → Implement → Merge → Review → Ship.** Run it end to end without pausing between phases. Pause for the user only when the work genuinely requires them: a destructive or irreversible action, a real scope change, or input that only they can provide.

### Phase 0 — Preflight

Requirements to proceed: inside a git repository, and a clean working tree. If the tree is dirty, stop and tell the user what's uncommitted — never stash or discard their work yourself.

Then set up state:
- `BASE_SHA` = resolved `--base` (or current HEAD). `SLUG` = short kebab-case name for the task (≤24 chars).
- Create `.fable-flow/` in the repo root and add `.fable-flow/` to `.git/info/exclude` if not already there (local ignore — don't edit the tracked `.gitignore`).
- Write `.fable-flow/task.md`: the task, flags, SLUG, BASE_SHA, timestamp. Every later phase writes its artifacts here too — this is the audit trail and the resume point for the stage commands (`/fable-flow:explore|plan|implement|review`).

### Phase 1 — Explore (3 × scout, parallel)

Spawn three `fable-flow:scout` subagents **in a single message** so they run concurrently — one per lens: `structure`, `conventions`, `blast-radius`. Each prompt: the task, its lens, and the base branch. Save each digest verbatim to `.fable-flow/explore-<lens>.md`.

### Phase 2 — Plan (architect)

Spawn one `fable-flow:architect` with: the task, all three digests inline, the max track count, and `Base: <branch> @ <BASE_SHA>`. Save its plan verbatim to `.fable-flow/plan.md`.

Before accepting the plan, check the one property the pipeline cannot survive without: **track file-ownership must be pairwise disjoint.** If two tracks own the same file, send the architect one revision request naming the overlaps; if the revision still overlaps, collapse the overlapping tracks into one. A single-track plan is fine — the pipeline shape doesn't change.

### Phase 3 — Implement (N × implementer, parallel worktrees)

Spawn one `fable-flow:implementer` per track, **all in a single message**. Each implementer already runs in its own isolated git worktree. Its prompt must contain, inline: its full track manifest from the plan, the entire Contracts section, the conventions digest, `BASE_SHA`, and its branch slug `fable-flow/<SLUG>-t<n>`. Remind it: reset to `BASE_SHA` first, work only on owned files, commit before reporting.

Save each track report to `.fable-flow/track-<n>-report.md`. If a track reports `blocked`, don't halt the world: continue with the completed tracks and handle the gap at merge (fix it directly, or spawn a fresh implementer for it with a narrowed manifest).

### Phase 4 — Merge

In the main checkout: `git checkout -b fable-flow/<SLUG> <BASE_SHA>`, then merge each reported track branch in the plan's merge order with `git merge --no-ff`. On conflicts, the plan's Contracts section is the arbiter — resolve to match it. Then run the plan's Integration verification commands. Small breakage: fix directly on the integration branch. Large breakage: spawn one implementer with a fix manifest and the **integration branch's current SHA** as its base, then merge its branch.

Do not proceed to review with a failing build/test run unless the failure predates the pipeline (prove that against `BASE_SHA` and say so).

### Phase 5 — Review (N × reviewer, parallel, with feedback loop)

Spawn `fable-flow:reviewer` subagents **in a single message**, one per lens in this priority order: `correctness`, `fidelity`, `integration` (integration only when more than one track merged, unless `--reviewers` says otherwise). Each prompt: the task requirements, the full plan, `BASE_SHA` as the base ref, and its lens. Save reports to `.fable-flow/review-round-<r>-<lens>.md`.

Triage: a finding is **actionable** if severity is blocker or major with confidence medium+. If any reviewer blocked or actionable findings exist: fix them on the integration branch (directly, or via an implementer based on the integration SHA for larger fixes), re-run the integration verification, and re-spawn **only the lenses that blocked**. At most `--rounds` cycles — whatever survives after that is reported, not silently retried. Minor/low-confidence findings are never fix-looped; they go in the final summary.

### Phase 6 — Ship

- Clean up: `git worktree remove` each track worktree (paths are in the track reports; use `--force` only for `blocked` tracks whose leftover state you've already captured) and delete merged `fable-flow/<SLUG>-t*` branches. Keep the integration branch.
- With `--pr`: push the integration branch and open a PR with `gh pr create` — body: what/why, tracks table, test evidence, review verdicts, surviving minor findings. If `gh` or a remote is missing, fall back to leaving the branch and say so.
- Without `--pr`: stop at the local integration branch and tell the user the exact command to publish when ready.

### Final summary

Before reporting, audit each claim against a tool result from this session — only report work you can point to evidence for; if something is not yet verified, say so explicitly. If tests fail, say so with the output. Lead with the outcome: branch name, what was built, test results, review verdicts, surviving findings, and the one next step. Complete sentences, no working shorthand.

If any phase fails in a way you can't recover, stop there, report faithfully what happened and what state `.fable-flow/` and the branches are in, and name the stage command that resumes from that point.
