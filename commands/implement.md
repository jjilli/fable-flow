---
description: Run the fable-flow Implement + Merge phases — one Fable 5 implementer per plan track, each in its own git worktree in parallel, then merge onto an integration branch and run integration verification.
argument-hint: "[--base REF]"
disable-model-invocation: true
---

## Context

- Branch: !`git branch --show-current`
- Working tree: !`git status --porcelain | head -20`

## Your job

Run the Implement and Merge phases of the fable-flow pipeline from the saved plan.

**Input:** $ARGUMENTS  (`--base REF` overrides the plan's base)

Preconditions: `.fable-flow/plan.md` exists (otherwise stop and point the user at `/fable-flow:plan`), and the working tree is clean (otherwise stop and tell the user what's uncommitted — never stash or discard their work yourself).

1. Read the plan. `BASE_SHA` = the plan's recorded base (or `--base`). `SLUG` = from `.fable-flow/task.md`, else derive from the plan title.
2. Spawn one `fable-flow:implementer` per track, **all in a single message** so they run concurrently — each already gets its own isolated worktree. Each prompt carries, inline: its full track manifest, the entire Contracts section, the conventions digest (`.fable-flow/explore-conventions.md`), any relevant lessons from `.fable-flow/memory/lessons/` (worktree subagents can't read `.fable-flow/` themselves), `BASE_SHA`, and branch slug `fable-flow/<SLUG>-t<n>`. Remind it: reset to `BASE_SHA` first, touch only owned files, commit before reporting. Save reports to `.fable-flow/track-<n>-report.md`.
3. Merge: `git checkout -b fable-flow/<SLUG> <BASE_SHA>`, then `git merge --no-ff` each track branch in the plan's merge order. Conflicts resolve in favor of the plan's Contracts. Run the plan's Integration verification commands; fix small breakage directly, or spawn an implementer with a fix manifest based on the integration branch's current SHA for anything larger.
4. Clean up merged track worktrees (`git worktree remove`, paths from the reports) and delete merged `fable-flow/<SLUG>-t*` branches.

Report with evidence: branch, tracks merged, the actual verification output, any deviations the implementers recorded, and anything blocked. If tests fail, say so with the output — don't proceed silently. Point the user at `/fable-flow:review` as the next step.
