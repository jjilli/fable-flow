# fable-flow — Development Guide

How to install, run, tune, and extend the fable-flow pipeline — and why its prompts are written the way they are.

---

## 1. What it is

fable-flow implements this pipeline as a Claude Code plugin:

| Stage | Who | Model · effort | Parallelism | Output |
|---|---|---|---|---|
| Explore | `scout` ×3 | Sonnet 5 (`model: sonnet`) | 3 concurrent, one lens each | `.fable-flow/explore-*.md` |
| Plan | `architect` | Fable 5 (`model: fable`) · `effort: xhigh` | 1 | `.fable-flow/plan.md` |
| Implement | `implementer` ×N | Fable 5 · `effort: medium` · `isolation: worktree` | N concurrent, own git worktree each | track branches + reports |
| Merge | orchestrator (your session) | session model | — | `fable-flow/<slug>` integration branch |
| Review | `reviewer` ×N | Opus 4.8 (`model: opus`) · `effort: high` | N concurrent, one lens each | `.fable-flow/review-*.md`, fix loop |
| Ship | orchestrator | session model | — | local branch, or PR with `--pr` |
| Remember | orchestrator | session model | — | `.fable-flow/memory/lessons/` updated |
| Iterate (on demand) | orchestrator + `implementer`/`reviewer` | session + Fable/Opus | per bug | fixes, regression tests, `.fable-flow/iterations/*.md` |

The dashed feedback arrow in the original diagram — review findings flowing back to implementation — is the bounded fix loop: actionable findings (blocker/major at medium+ confidence) get fixed on the integration branch and only the lenses that blocked re-run, at most `--rounds` times (default 2).

Two stages go beyond the original diagram. **Remember** closes each run into per-repo memory (`.fable-flow/memory/lessons/`), which Phase 0 of the next run reads and injects into the architect, implementers, and reviewers — the Fable 5 guide's "construct a memory system" recommendation, wired into the cycle. **Iterate** (`/fable-flow:iterate`) is the loop for bugs you find by hand after implementation: reproduce first, root-cause, fix with a regression test, one correctness review pass, then bank the lesson.

Model aliases (`sonnet`, `opus`, `fable`) always resolve to the current generation, so today this is exactly Sonnet 5 / Fable 5 / Opus 4.8 and it tracks future releases automatically. Pin full IDs if you need reproducibility (§6).

## 2. Install

**As a plugin (normal use):**

```bash
claude plugin marketplace add jjilli/fable-flow
claude plugin install fable-flow@fable-flow
```

The repo is both the plugin and its own single-plugin marketplace (`.claude-plugin/marketplace.json` points at `./`). A local clone or plain directory works the same way: `claude plugin marketplace add /path/to/fable-flow`.

**For plugin development (no install):**

```bash
claude --plugin-dir /path/to/fable-flow
```

**Releases:** published at https://github.com/jjilli/fable-flow. Each release is tagged `fable-flow--v<version>` via `claude plugin tag`, which checks that `plugin.json` and the marketplace entry agree before tagging. Installed copies pick up new versions with `claude plugin update fable-flow`.

## 3. Running the pipeline

```
/fable-flow:ship <task description> [flags]
```

| Flag | Default | Meaning |
|---|---|---|
| `--tracks N` | 3 | Max parallel implementation tracks (the architect may use fewer) |
| `--reviewers N` | 3 if multi-track, else 2 | Review lenses: correctness → fidelity → integration |
| `--rounds N` | 2 | Max review→fix cycles before surviving findings are just reported |
| `--base REF` | current branch | Base for tracks and for the review diff |
| `--pr` | off | Push and open a GitHub PR at the end. Without it the pipeline stops at the local integration branch — publishing is deliberately opt-in. |

Stage commands run phases individually and share state through `.fable-flow/`:

- `/fable-flow:explore <task>` — scouts only
- `/fable-flow:plan [task] [--tracks N]` — architect (runs explore first if digests are missing)
- `/fable-flow:implement [--base REF]` — implementers + merge, from the saved plan
- `/fable-flow:review [--base REF] [--fix]` — reviewers on the current branch; **report-only unless `--fix`**
- `/fable-flow:iterate <bug description(s)> [--no-review]` — the post-implementation loop: reproduce the bug (it refuses to fix what it can't reproduce), root-cause it against the plan, fix on the current branch (an implementer subagent for bigger fixes), add a regression test, run one correctness-lens review, and update memory

Use stages while learning the pipeline, to resume after a failure (ship names the stage to resume from), or to review a branch fable-flow didn't build.

**What to expect at runtime:**

- **Long turns are normal.** Fable 5 requests on hard tasks can run many minutes each; a full ship on a real feature is comfortably tens of minutes. That's the design — check in on it rather than watching it.
- **Preconditions:** a git repo and a clean working tree. The pipeline refuses to start otherwise (it will never stash your uncommitted work).
- **Permissions:** implementers edit files and run tests, the orchestrator creates branches and merges. Run in a permission mode you're comfortable with (`acceptEdits` works well); `--pr` additionally needs `gh` authenticated.
- **Cost:** this is a premium pipeline by construction — several Fable 5 agents (higher per-token price than Opus) plus Opus reviewers per run. For routine tasks where you don't need it, don't use it; that's also why every command has `disable-model-invocation: true` — only you can trigger the pipeline, never the model on its own.

## 4. State directory

Everything the pipeline learns and decides is written to `.fable-flow/` in the target repo (added to `.git/info/exclude`, so it never shows up in your diffs):

```
.fable-flow/
├── task.md                      task, flags, slug, base SHA
├── explore-structure.md         scout digests
├── explore-conventions.md
├── explore-blast-radius.md
├── plan.md                      the architect's plan (contracts, tracks, merge order)
├── track-<n>-report.md          implementer reports (branch, commit, test evidence)
├── review-round-<r>-<lens>.md   reviewer reports
├── iterations/<n>-<slug>.md     iterate records (bug, repro evidence, root cause, fix, tests)
└── memory/lessons/<slug>.md     cross-run lessons — read at Phase 0, written at Phase 7 and by iterate
```

This is the audit trail (every claim in the final summary traces to a file here) and the resume mechanism (stage commands read whatever exists).

## 5. Why the prompts look like this — Fable 5 prompting

Fable prompting is different from past models, and this plugin is built around the official guidance ([Prompting Claude Fable 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5) — distilled into the bundled `/fable-flow:fable-prompting` skill). If you edit the prompts, keep these principles or quality drops:

1. **De-prescribe.** Prompts written for older models are too prescriptive for Fable 5 and *degrade* output. The agent files state goals, constraints, and output contracts — not numbered procedures. When you extend them, resist adding step lists and "CRITICAL: YOU MUST" language.
2. **Full spec up front.** Fable does its best long-horizon work when the first turn carries the complete task. That's why the orchestrator inlines the whole track manifest, contracts, and conventions digest into each implementer prompt rather than letting them re-discover context.
3. **Effort is the control surface.** Planning runs `xhigh` (one agent whose output gates everything downstream), review runs `high`, and implementation of a pre-planned track runs `medium` (Fable's `medium` is strong — the guide notes even `low` often beats prior models' `xhigh`). Per-agent `effort:` frontmatter is how these labels are realized.
4. **Fresh-context verifiers beat self-critique.** Reviewers are separate agents with clean context and an adversarial stance, per the guide's scaffolding advice — never the implementer grading its own work. The reviewer prompt also uses the coverage-first reporting language ("report every issue… do not filter") because Opus 4.8 follows conservative-reporting instructions so literally that recall drops.
5. **Grounded progress.** The implementer, reviewer, and orchestrator all carry the guide's audit-your-claims snippet ("only report work you can point to evidence for") — this is what makes the final summary trustworthy after an hour of autonomous work.
6. **Delegate freely, in parallel.** Fable dispatches and manages parallel subagents dependably, so the commands explicitly spawn scouts/implementers/reviewers in a single message to run concurrently instead of one-at-a-time.
7. **Boundaries, stated.** Dirty trees are refused rather than stashed; `/fable-flow:review` without `--fix` reports and stops; PRs require `--pr`. Fable takes initiative — the prompts define where it must not.
8. **Never ask for reasoning verbatim.** No prompt asks an agent to echo its thinking (that can trigger Fable's `reasoning_extraction` refusal). Agents report conclusions and evidence.
9. **Memory in the loop.** The guide's "construct a memory system" recommendation is implemented directly: one lesson per file under `.fable-flow/memory/lessons/` with a one-line summary on top, injected into architect/implementer/reviewer prompts at the start of every run (Phase 0), and updated at the end of every run (Phase 7) and every iterate. The guide's memory-discipline snippet governs what's worth saving — lessons, not instances.

The verbatim steering snippets live in [skills/fable-prompting/SKILL.md](skills/fable-prompting/SKILL.md) — that skill is also model-invocable, so Claude will pull it in automatically when you ask it to write or fix Fable-targeted prompts.

A companion skill, [skills/build-patterns/SKILL.md](skills/build-patterns/SKILL.md), carries the *engineering* lessons that fable-prompting's *prompting* lessons don't: the recurring integration failure where every track's suite is green but the round is broken at the seam (a background thread, a non-HTTP request scope, a real timing path), the reusable patterns that make later rounds cheaper (provider-adapter, enqueue-only worker, guarded migration, invariant validators that don't break existing input), and the verification discipline a green suite alone doesn't give — one real end-to-end roundtrip per round, a screenshot for UI, deterministic tests for time/IO seams. The architect, implementer, and reviewer prompts each point at it; it also loads automatically when planning/implementing/reviewing hits one of those seams. These aren't Fable-specific — they're what a long multi-round build taught the pipeline about its own blind spots.

A third skill, [skills/frontend-aesthetics/SKILL.md](skills/frontend-aesthetics/SKILL.md), gives the pipeline real UI-design capability so a user-facing track produces something that looks designed rather than generic "AI slop": committing to a design direction, the four dimensions to steer explicitly (distinctive typography, a committed palette, one orchestrated motion moment, atmospheric backgrounds), information architecture for "anyone can use", and the tactics that make a whole-app redesign land as one coherent diff (design-system first, preserve the test hooks, self-host fonts). Crucially it also carries the visual-verification step — serve the built app over seeded data and screenshot it with a headless browser — because a green test suite structurally cannot see a clipped label or an overflowing bar. The architect commits to a direction for UI tracks, and the reviewer looks at the rendered pages, not just the build log.

## 6. Customization

**Models.** Each agent's `model:` accepts aliases (`sonnet`, `opus`, `haiku`, `fable`), full IDs (`claude-sonnet-5`, `claude-opus-4-8`, `claude-fable-5`), or `inherit` (use the session model). Two common edits:

- *Cheaper pipeline:* set `agents/architect.md` and `agents/implementer.md` to `model: opus` (or `inherit` and run the session on the model you're paying for).
- *Reproducible pipeline:* replace aliases with full IDs so a future model generation can't change behavior under you.

Overrides that beat frontmatter, in order: the `CLAUDE_CODE_SUBAGENT_MODEL` environment variable, then a per-invocation model in the Agent tool call, then frontmatter, then the session model.

**Effort.** Per-agent `effort:` takes `low|medium|high|xhigh|max`. Raise the implementer to `high` for gnarly codebases; drop scouts' thoroughness by adding `effort: low` to `agents/scout.md` for very large repos where recon cost matters.

**Defaults for tracks / reviewers / rounds** live in prose in `commands/ship.md` — edit the flag-defaults line.

**Add a review lens** (e.g. `security`): add the lens description to `agents/reviewer.md`'s lens list and to the lens order in `commands/ship.md` and `commands/review.md`. Same pattern for scout lenses.

**Memory.** Lessons are per-repo and local — the whole `.fable-flow/` directory sits in `.git/info/exclude`, so memory never leaves your machine. To share hard-won lessons with a team, promote the stable ones into the repo's `CLAUDE.md` (all agents read that natively) and delete the local copies. To reset a repo's pipeline memory, delete `.fable-flow/memory/`. To inspect what the pipeline has learned, just read the files — they're plain Markdown with the summary on line one.

**Team conventions:** put repo-specific rules (test commands, deploy constraints) in the target repo's `CLAUDE.md` — agents inherit it; don't fork the plugin for per-repo knowledge.

## 7. Developing the plugin itself

Layout:

```
fable-flow/
├── .claude-plugin/
│   ├── plugin.json          manifest (name is the /fable-flow: namespace)
│   └── marketplace.json     makes this repo installable as its own marketplace
├── commands/                user-invoked pipeline verbs (flat .md, YAML frontmatter)
├── agents/                  subagent definitions (frontmatter + system prompt)
├── skills/fable-prompting/  auto-invocable knowledge skill: how to prompt Fable 5
├── skills/build-patterns/   auto-invocable knowledge skill: what breaks and what works when the plan turns into code
├── skills/frontend-aesthetics/  auto-invocable knowledge skill: designed-not-generic UI + visual verification
├── README.md
└── GUIDE.md
```

Workflow:

1. Edit files.
2. `claude plugin validate /Users/kennae/Claude/fable-flow --strict` — catches frontmatter YAML errors (note: an unquoted `argument-hint:` starting with `[` parses as a YAML array and silently drops your metadata — keep those values quoted).
3. In a running session, `/reload-plugins` picks up changes; or test un-installed with `claude --plugin-dir …`.
4. `claude plugin details fable-flow` shows the component inventory and projected token cost of what you're shipping into every session.

Facts worth knowing when editing:

- Commands and skills are namespaced by plugin name: `/fable-flow:ship`. Agents likewise: the orchestrator addresses `fable-flow:scout` etc.
- `disable-model-invocation: true` on every command means only a human can start the (expensive) pipeline. The `fable-prompting` skill deliberately omits it so Claude can auto-load the reference.
- `isolation: worktree` on the implementer is what gives each track its own checkout — stock Claude Code creates and cleans up the worktree; the implementer's first act is `git reset --hard <BASE_SHA>` so all tracks share an exact base regardless of what the worktree branched from.
- Plugin agents ignore `hooks`, `mcpServers`, and `permissionMode` frontmatter by design; don't add them here.
- Version bumps: update `version` in `plugin.json`; installed copies pick it up via `claude plugin update fable-flow`.

## 8. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `/fable-flow:ship` refuses to start | Dirty working tree — commit or stash yourself, then rerun. This is intentional. |
| A Fable agent declines a benign task (security-adjacent code, bio/life-sciences code) | Fable 5's safety classifiers can false-positive near those domains. Rerun that stage with the agent's `model:` set to `opus` — Opus 4.8 is the official fallback model. |
| Merge conflicts between tracks | The plan's ownership check should prevent this; if it happens, the Contracts section is the arbiter. Recurring conflicts mean the architect is splitting too aggressively — lower `--tracks`. |
| Leftover worktrees or `fable-flow/*-t*` branches after a failed run | `git worktree list` → `git worktree remove <path>` (add `--force` if the tree is dirty), `git worktree prune`, `git branch -D fable-flow/<slug>-t<n>`. |
| Review loop never converges | Findings surviving `--rounds` cycles are reported, not retried — read them; they're usually a real design problem or a plan/requirement mismatch. |
| Found a bug after the pipeline finished | `/fable-flow:iterate <describe it>` — reproduces first, fixes with a regression test, updates memory. It refuses to fix what it can't reproduce; give it the repro detail it asks for. |
| Memory contains a wrong or stale lesson | Delete or edit the file under `.fable-flow/memory/lessons/` — it's plain Markdown, and wrong lessons actively mislead future runs. |
| Commands missing after install | Restart the session or `/reload-plugins`; confirm with `claude plugin list`. |
| Pipeline feels slow | It's front-loaded deliberation: Fable turns run minutes. Use stage commands for tighter iteration, or lower effort/models per §6. |

## 9. Design decisions (so you don't re-litigate them)

- **PR is opt-in (`--pr`)** even though the diagram ends at PR: publishing to a remote is an outward-facing action, so the default stops at a local branch you can inspect.
- **The orchestrator refuses dirty trees instead of stashing** — the pipeline must never be the thing that loses your uncommitted work.
- **Aliases over pinned model IDs by default** — the diagram named the *current* generation of each tier; aliases keep that meaning as generations advance. Pin IDs when you need frozen behavior.
- **Review fixes happen on the integration branch, not by re-running tracks** — cheaper, and it matches the diagram's feedback arrow into a bounded loop instead of an unbounded rebuild.
- **State in `.fable-flow/` via `.git/info/exclude`** — auditable and resumable without ever touching tracked files.
- **Architect runs `xhigh`, above the diagram's "high" label** — planning is a single agent whose output gates every downstream track; the marginal cost of `xhigh` there is small against the cost of a bad plan. (The diagram is a starting point, not a contract.)
- **One shared memory, orchestrator-managed** — not per-agent `memory:` frontmatter silos. A reviewer's lesson must reach the next run's architect, and worktree implementers can't read `.fable-flow/` anyway, so the orchestrator injects lessons into prompts and owns all writes. Iterate records keep instances; memory keeps lessons.
- **Iterate refuses to fix unreproduced bugs** — a fix without a failing reproduction is a guess, and the pipeline's currency is evidence.
