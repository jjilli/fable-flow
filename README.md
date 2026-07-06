# fable-flow

A Claude Code plugin that turns one task description into a shipped branch using a multi-agent pipeline tuned for **Claude Fable 5**:

```
            ┌─ scout (Sonnet 5) ── structure ──┐
   task ──> ├─ scout (Sonnet 5) ── conventions ├──> architect (Fable 5, xhigh effort)
            └─ scout (Sonnet 5) ── blast-radius┘         │  file-disjoint track plan
                                                          ▼
            ┌─ implementer (Fable 5, medium) ── own worktree ─┐
            ├─ implementer (Fable 5, medium) ── own worktree ─├──> merge ──> N × reviewer
            └─ implementer (Fable 5, medium) ── own worktree ─┘    │        (Opus 4.8, high)
                                                                   │              │
                                                                   ▲──── fix loop ┘
                                                                   ▼
                                                             branch / PR
```

Every prompt in this plugin follows the official [Prompting Claude Fable 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5) guide — goals and constraints instead of step lists, verbatim steering snippets, fresh-context verifiers, grounded progress claims.

## Install

```bash
claude plugin marketplace add jjilli/fable-flow
claude plugin install fable-flow@fable-flow
```

From a local clone the same works with a path: `claude plugin marketplace add /path/to/fable-flow`. For plugin development, load it without installing: `claude --plugin-dir /path/to/fable-flow`

## Use

```
/fable-flow:ship add rate limiting to the public API --pr
```

Or run phases individually: `/fable-flow:explore`, `/fable-flow:plan`, `/fable-flow:implement`, `/fable-flow:review` — and when you find a bug by hand afterwards, `/fable-flow:iterate <bug>` reproduces it, fixes it with a regression test, and banks the lesson.

The pipeline remembers: lessons from every run and iteration are stored in `.fable-flow/memory/lessons/` and injected into the architect, implementers, and reviewers of the next run. Pipeline state, artifacts, and memory live in `.fable-flow/` in your repo (git-ignored locally).

## Contents

| Component | Files |
|---|---|
| Commands | `commands/ship.md`, `explore.md`, `plan.md`, `implement.md`, `review.md`, `iterate.md` |
| Agents | `agents/scout.md` (Sonnet 5), `architect.md` (Fable 5 · xhigh), `implementer.md` (Fable 5 · medium · worktree), `reviewer.md` (Opus 4.8 · high) |
| Skills | `skills/fable-prompting/` — the distilled Fable 5 prompting reference · `skills/build-patterns/` — battle-tested engineering patterns and seam traps from long multi-round builds (the recurring "green per track, broken at the seam" failure, reusable patterns, verification discipline beyond a green suite) |

**Read [GUIDE.md](GUIDE.md)** for the full development guide: how the pipeline works, customization, cost/runtime expectations, and how to extend it.
