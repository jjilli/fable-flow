---
name: grilling
description: Relentless, one-question-at-a-time interview that stress-tests a plan or design before any code is written. Invoke ONLY when the user explicitly asks for it — they say "grill me", "grill the plan", "grilling", or otherwise ask to be grilled / to pressure-test the design. Do NOT auto-invoke it for ordinary planning or clarification; the default new-project brainstorm (card-based multiple-choice questions) is a separate, lighter thing. This is the opt-in deep interrogation.
---

# Grilling

Invoke this only when the user explicitly asks to be grilled (or to grill the
plan/design). It is intentionally heavier than normal clarification — a
relentless interview, not a few questions. If the user hasn't asked for it, don't
start it.

When they have:

Interview the user relentlessly about every aspect of this plan until you reach a
shared understanding. Walk down each branch of the design tree, resolving
dependencies between decisions one by one. For each question, provide your
recommended answer.

Ask the questions **one at a time**, waiting for feedback on each before
continuing. Asking multiple questions at once is bewildering.

If a *fact* can be found by exploring the codebase, look it up rather than asking —
spend the user's attention only on decisions. The *decisions* are theirs: put each
one to them and wait for the answer.

Do not enact the plan until the user confirms you have reached a shared
understanding.

In this pipeline, grilling sits **before** `/fable-flow:plan` — it hardens the
task and its design decisions so the architect plans against a settled intent
rather than guesses. Once grilling ends in a shared understanding, feed the
resolved decisions into the task (`.fable-flow/task.md`) and proceed to planning.

---
*Adapted from Matt Pocock's `grilling` skill — https://github.com/mattpocock/skills/blob/main/skills/productivity/grilling/SKILL.md*
