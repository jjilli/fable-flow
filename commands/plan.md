---
description: Run only the fable-flow Plan phase — a high-effort Fable 5 architect turns the scout digests into a file-disjoint parallel track plan saved to .fable-flow/plan.md.
argument-hint: "[task description] [--tracks N]"
disable-model-invocation: true
---

Run the Plan phase of the fable-flow pipeline.

**Input:** $ARGUMENTS  (may refine or replace the task in `.fable-flow/task.md`; `--tracks N` caps parallel tracks, default 3)

1. Load the task (arguments, else `.fable-flow/task.md`) and the three scout digests `.fable-flow/explore-*.md`. If digests are missing, run the Explore phase first exactly as `/fable-flow:explore` describes, then continue.
2. Spawn one `fable-flow:architect` with: the task, all digests inline, the max track count, any relevant lessons from `.fable-flow/memory/lessons/`, and `Base: <current branch> @ <HEAD sha>`.
3. Save the plan verbatim to `.fable-flow/plan.md`.
4. Verify track file-ownership is pairwise disjoint. If two tracks own the same file: one revision request to the architect naming the overlaps; if it still overlaps, collapse the overlapping tracks into one and note that you did.

Then report: the plan's shape (tracks, what each owns, merge order), the contracts in one or two sentences, and the risks the architect called out. The deliverable of this command is the plan itself — don't start implementing. Point the user at `/fable-flow:implement` to execute it.
