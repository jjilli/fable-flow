---
description: Post-implementation iteration loop for bugs found by hand — reproduce the bug, root-cause it, fix it on the current branch with a regression test, run one correctness review pass, and bank the lesson to pipeline memory.
argument-hint: "<bug description(s)> [--no-review]"
disable-model-invocation: true
---

## Context

- Branch: !`git branch --show-current`
- HEAD: !`git rev-parse --short HEAD`
- Working tree: !`git status --porcelain | head -20`

## Your job

Run the fable-flow iteration loop for bugs the user found manually:

**Bug report:** $ARGUMENTS

Flag: `--no-review` skips the reviewer pass (for trivial fixes).

This works on the **current branch** — typically a `fable-flow/<slug>` integration branch, but any branch is fine. Precondition: clean working tree (otherwise stop and tell the user what's uncommitted — never stash or discard their work yourself).

Load context first: `.fable-flow/plan.md` (its Contracts section still governs), `.fable-flow/task.md`, previous `.fable-flow/iterations/*.md`, and `.fable-flow/memory/lessons/` — past lessons often point straight at the cause.

For each reported bug, in order:

1. **Reproduce before touching anything.** Find or write the command/test that demonstrates the bug and run it; quote the failing output. If you cannot reproduce it, stop on that bug and report exactly what you tried and what detail would let you (that's input only the user can provide) — never shotgun-fix an unreproduced bug.
2. **Root-cause it.** The reproduction plus the plan tells you where to look. Distinguish "the code violates the plan" from "the plan or a contract was itself wrong" — say which, because a contract-level fix must be recorded in the plan, not patched around.
3. **Fix it.** Small fixes directly; larger ones via a `fable-flow:implementer` given a narrowed fix manifest and the current branch's HEAD as its base, then merge its branch. If a contract had to change, update the Contracts section of `.fable-flow/plan.md` to match reality.
4. **Prove it.** Three things, with real output quoted: the step-1 reproduction now passes; a new or extended regression test fails on the pre-fix code and passes now; the plan's Integration verification still passes.

Bugs whose fixes are clearly independent and touch disjoint files may be fixed in parallel via implementers (same ownership rule as the main pipeline). When in doubt, sequential — an interleaved half-fix costs more than a slow one.

Unless `--no-review`: spawn one `fable-flow:reviewer` (lens `correctness`) on this iteration's diff — base is the HEAD recorded before you started — with the plan, the bug reports, and any memory lessons. Actionable findings (blocker/major at medium+ confidence) get one fix pass and re-verification; anything surviving is reported, not looped.

Close the loop:

- Commit on the current branch: `fable-flow(iterate): <short bug slug>` — one commit per bug, or one coherent commit for tightly related bugs.
- Write `.fable-flow/iterations/<next-number>-<slug>.md`: the bug as reported, repro evidence, root cause, the fix, tests added, review verdict.
- Update `.fable-flow/memory/lessons/` when a bug reveals something durable — a trap in this codebase, a class of mistake the pipeline made, a contract that had to change. One lesson per file, one-line summary at the top; update rather than duplicate. The *instance* lives in the iteration record; the *lesson* is what stops the next run from repeating it.

Report: lead with what was broken and what now proves it fixed. Then, per bug: root cause in plain sentences, the regression test that pins it, the review verdict, and anything you couldn't reproduce or fix. Audit every claim against a tool result from this session — if something is unverified, say so.
