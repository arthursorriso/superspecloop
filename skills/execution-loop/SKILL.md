---
name: execution-loop
description: Use to execute a Spec Kit feature (specs/<feature>/ with spec.md, plan.md, tasks.md) unit by unit — each unit run end to end by a fresh unit-runner subagent (implementer, gate checks, optional unit review, fix rounds), with scripts keeping the record — until every task is done or reported not done. Use instead of /speckit.implement.
---

<!-- Derived from Superpowers' subagent-driven-development/SKILL.md (MIT, (c) 2025 Jesse Vincent). See UPSTREAM.md. -->

# Execution Loop

Execute a Spec Kit feature by converting its tasks into units, then handing
each unit to a fresh `superspecloop:unit-runner` subagent that implements,
checks, optionally reviews and closes it, and replies with one line.

**You are the controller, and a light one.** You prepare the feature, dispatch
one runner per unit, and act on its one-line reply. You never run a unit's
steps yourself: implementing, checking, reviewing and closing happen inside
the runner, in a context that is thrown away when the unit is done. That is
what keeps your context — and the cost of every step you take — the same at
the last unit as at the first. In one measured run, a controller that ran
nine units itself carried every unit's history into the next, and was the
most expensive part of the pipeline.

**The record is written by scripts, not by you.** A unit becomes complete only
when the `unit-close` script says so, and it refuses when the review, the
gates or the task evidence is missing. A controller that wrote its own
completion lines once marked three unreviewed units "review clean", and
ticked tasks nobody performed.

**Narration:** at most one short line between tool calls. **Continuous
execution:** no check-ins between units.

**Rulings, not stalls — except on the spec.** Technical conflicts and
ambiguities are decided, not escalated: spec.md is the binding authority,
plan.md its argument, units.md your restatement of both. Every decision is a
ledger line `Ruling: <what> — <why> — <what it costs if wrong>`, written with
`bash scripts/ledger FEATURE "<line>"`. To change one, add
`Ruling: amends <which> — ...`. A ruling never makes the code do something
other than what the spec says: that is the human's decision, always.

**These stop you, and only these:**

1. A runner replies `QUESTION` — a decision about the spec. Stop and ask the
   human; when a conductor dispatched you, return the question to it.
2. A runner replies `BLOCKED`, and nothing you can change (context, model,
   splitting the unit) unblocks it.
3. An irreversible, destructive or security-sensitive action, or a side
   effect outside this working tree that norms say you ask about first.

## Paths

Scripts live in this skill's `scripts/` directory; pass its **absolute path**
to every runner as `S`. Invoke scripts with `bash`. `FEATURE` is the feature
directory (`specs/001-login`), `WS` the workspace
`bash scripts/pipeline-workspace FEATURE` prints, `WD` the working directory
(the repository root, or the worktree's root).

## Setup

1. **Branch.** `bash scripts/ensure-branch FEATURE`; relay its line. The tree
   must be clean except untracked files under `specs/`. For an isolated
   workspace, run the `superspecloop:worktree` skill first and use its path
   as `WD`.
2. **Workspace and ledger.** `bash scripts/pipeline-workspace FEATURE`. If
   `WS/progress.md` names FEATURE, resume: units with a `Unit <N>: complete`
   line are done. Otherwise create it with
   `# SuperSpecLoop ledger — feature: FEATURE` and
   `Start: <git rev-parse HEAD> on <branch>`.
3. **Mode.** If the ledger has no `Mode:` line, add one:
   `Mode: lean` by default — no per-unit review; the whole-branch review
   covers the units — or `Mode: full` when the human asked for unit reviews.
   In measured runs the per-unit review found the fewest real problems per
   call; the whole-branch review found the most.
4. **Units.** If `WS/units.md` does not exist, dispatch one `general-purpose`
   subagent (`sonnet`): "Load the `superspecloop:tasks-to-briefs` skill and
   run it for FEATURE. Reply only with its report." Ledger its counts and
   gaps. A gap is a ruling now (extend the nearest unit, ledgered) or an
   uncovered item reported at the end — never dropped. Failed preconditions:
   stop and tell the human.
5. **Global file.** Copy units.md's Global Constraints and Verification
   Commands, verbatim, into `WS/global.md`.
Steps 6 and 7 run once per feature: when resuming, skip them if the ledger
already has a `Baseline:` line — a baseline taken after units changed the
code is not a baseline.

6. **Baseline.** `bash scripts/baseline FEATURE "<suite>" "<typecheck>" "<lint>"`
   with the Verification Commands (`"-"` for one the project lacks). It saves
   the commands every later check uses and ledgers the result. A failure that
   smells intermittent: run it once more; if it passes, ledger
   `Baseline: flake — <test> — <what failed>, not reproduced`. A real failure
   is recorded, not fixed: later gates compare against it.
7. **Pre-flight scan.** Read units.md once. For every pair of units sharing a
   file or interface, and every unit against itself, write a table row to the
   ledger: what one produces, what the other consumes, what you found. Rule on
   each finding before the first unit. "Clean" without rows is not a scan.

## The Loop

For each unit, in order, without a `Unit <N>: complete` line:

1. If you decided anything about this unit that is not yet a ruling, ledger
   it first.
2. Dispatch `superspecloop:unit-runner` (`sonnet`) with: `WD`, `FEATURE`, the
   unit number, `S` (this skill's scripts directory, absolute), and the
   implementer model — `sonnet`; `haiku` only when units.md marks the unit
   `**Kind:** docs-config` (documentation, configuration, renames).
3. Act on its one-line reply:
   - `DONE: Unit N: complete (...)` → append nothing (the script already
     did), next unit.
   - `QUESTION: ...` → stop (see above). When the human answers, ledger the
     answer as a ruling quoting them, and dispatch a fresh runner for the
     same unit: it resumes from its recorded start.
   - `BLOCKED: ...` → change something — more context in the dispatch, split
     the unit (ledger the split), or `opus` as the implementer model — and
     dispatch again. Never retry unchanged. Still blocked: stop.

Never dispatch two runners at once, and never do a unit's work yourself —
not a check, not a fix, not a ledger completion line.

## When Every Unit Is Complete

Load `superspecloop:finish-branch` and follow it: it runs the gates on the
final tree, the whole-branch review, one wave of fixes, the pull-request
description and the integration menu.

Only if it cannot be loaded, stop with: units and commit range; every
`Ruling:` line, each with its cost if wrong; every `not done` task; deferred
minors and parked findings; uncovered spec items.

Keep the workspace until the branch is merged; the ledger is the recovery
map.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "This unit is small; I'll run it myself instead of dispatching a runner" | Then its history rides in your context through every later unit. Dispatch. |
| "unit-close refused, but the review was basically clean" | It refuses when the record would be false. Fix what it names. |
| "The task is done in spirit — the code is there" | A task is done when someone did it. Not done is a fine answer; a false tick is not. |
| "The runner's question is minor; I'll answer it to keep moving" | Questions about the spec are the human's. Stop and ask. |
| "Full mode would be safer, I'll turn it on" | The human chose the mode. Lean is the default because the whole-branch review catches more per call. |
| "I'll ledger 'complete' myself, the script is overkill" | That is exactly how three unreviewed units became "review clean". |
