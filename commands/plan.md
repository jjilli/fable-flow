---
description: Run only the fable-flow Plan phase — a high-effort Fable 5 architect turns the scout digests into a file-disjoint parallel track plan saved to .fable-flow/plan.md.
argument-hint: "[task description] [--tracks N]"
disable-model-invocation: true
---

Run the Plan phase of the fable-flow pipeline.

**Input:** $ARGUMENTS  (may refine or replace the task in `.fable-flow/task.md`; `--tracks N` caps parallel tracks, default 3)

1. Load the task (arguments, else `.fable-flow/task.md`) and the three scout digests `.fable-flow/explore-*.md`. If digests are missing, run the Explore phase first exactly as `/fable-flow:explore` describes, then continue.
2. **New/greenfield project — brainstorm direction first.** When the task is a whole new project or a whole-product brief (little or no existing code to constrain the choices), don't jump to the plan. Surface **every** open design and direction decision you have and put each one to the user — do not truncate to a handful, and never silently assume a default on their behalf. Ask them as card-based multiple-choice questions (one decision per card, your recommended option first, and an option *preview* — a small ASCII mockup, code snippet, or layout sketch — wherever a visual side-by-side helps). The card UI takes only a few at a time, so ask across **as many rounds as it takes**, and keep going until nothing material is unresolved. Look up any *fact* yourself and spend the user's attention only on genuine decisions — but on **all** of them. Fold the answers into `.fable-flow/task.md` before planning; if planning later exposes a decision you never settled, put that to the user too rather than guessing. For a well-specified change to an existing codebase, skip this and plan directly. (This is the default clarification; the heavier opt-in `grilling` skill is only for when the user explicitly asks to be grilled.)
3. Spawn one `fable-flow:architect` with: the task, all digests inline, the max track count, any relevant lessons from `.fable-flow/memory/lessons/`, and `Base: <current branch> @ <HEAD sha>`.
4. Save the plan verbatim to `.fable-flow/plan.md`.
5. Verify track file-ownership is pairwise disjoint. If two tracks own the same file: one revision request to the architect naming the overlaps; if it still overlaps, collapse the overlapping tracks into one and note that you did.

Then report: the plan's shape (tracks, what each owns, merge order), the contracts in one or two sentences, and the risks the architect called out. The deliverable of this command is the plan itself — don't start implementing. Point the user at `/fable-flow:implement` to execute it.
