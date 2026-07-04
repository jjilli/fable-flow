---
description: Run only the fable-flow Explore phase — three parallel Sonnet 5 scouts (structure, conventions, blast-radius) producing digests under .fable-flow/ for a later plan.
argument-hint: "<task description>"
disable-model-invocation: true
---

Run the Explore phase of the fable-flow pipeline for this task:

**Task:** $ARGUMENTS

(If no task is given above, reuse the one in `.fable-flow/task.md`; if neither exists, ask the user for one — that's input only they can provide.)

1. Create `.fable-flow/` in the repo root if missing, add `.fable-flow/` to `.git/info/exclude` if absent, and record the task in `.fable-flow/task.md`.
2. Spawn three `fable-flow:scout` subagents **in a single message** so they run concurrently — lenses `structure`, `conventions`, and `blast-radius`. Each prompt: the task, its lens, the current branch.
3. Save each digest verbatim to `.fable-flow/explore-<lens>.md`.

Then report: one short paragraph per lens with what matters most for planning, plus the risks and unknowns the scouts flagged. Point the user at `/fable-flow:plan` as the next step. Report only what the digests actually say — don't pad or speculate.
