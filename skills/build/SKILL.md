---
name: build
description: Use to take a feature from a request to a reviewed pull request in one run — chains Spec Kit (specify, plan, tasks, analyze) with the SuperSpecLoop review loops, execution loop and branch finishing, stopping only at Gate 1 and at the integration decision. Also resumes a feature already in progress.
argument-hint: "<what you want built> | <feature directory to resume>"
---

# Build

Run the whole pipeline for one feature: request → reviewed spec → reviewed
plan → tasks → implemented, reviewed branch → pull request.

```text
$ARGUMENTS
```

The text above is the human's input. It is one of:

- **A request** in their own words — start a new feature.
- **A feature directory** (`specs/003-export`) or the word `resume` — continue
  a feature already in progress.
- **Empty** — if `.specify/feature.json` names a feature directory, resume
  it; otherwise ask what they want built.

**You are the conductor.** Each phase has its own skill, which has its own
rules and its own subagents. Your job is the order, the hand-offs and the
stops. You do not restate a phase's rules, shortcut its steps, or carry its
output forward in your head: everything a later phase needs is on disk.

**Where you are is what the disk says.** After every phase, and after every
stop, run

`bash ../execution-loop/scripts/pipeline-phase FEATURE`

and do what it prints. Never infer the phase from the conversation — a
session that was compacted or restarted resumes correctly only because the
phase comes from files and the ledger.

## Stops

The pipeline runs unattended between exactly these points:

1. **Gate 1** — always. The human answers the spec's open questions and
   approves it. Nothing is planned on an unapproved spec.
2. **The integration decision** — always. Pull request, local merge or keep
   is the human's call.
3. **A phase's own stop conditions** — the plan review stops when the spec
   has to change or a decision is expensive to reverse; the execution loop
   stops on destructive, security-sensitive or out-of-tree actions. When a
   phase stops, you stop: relay its question and wait.

Between them, do not check in, summarize progress or ask whether to
continue. Announce each phase in one line and run it.

## Before the First Phase

- `.specify/` must exist. If it does not, Spec Kit is not initialized in
  this project: say so and stop.
- Spec Kit's commands are skills named `speckit-specify`, `speckit-plan`,
  `speckit-tasks` and `speckit-analyze` (older installs expose them as
  `/speckit.specify` and so on). Use whichever this project has.
- If the human asked for an isolated worktree, the `superspecloop:worktree`
  skill runs once, just before the `execute` phase.

## Phases

Load a phase's skill when its phase starts, not before.

### specify

Run Spec Kit's specify command with the human's request, verbatim.

Two things differ from running it by hand:

- **Do not run its clarification round.** If it wants to ask the human
  questions about `[NEEDS CLARIFICATION]` markers, do not ask them and do
  not answer them: leave the markers in the spec. The spec review turns
  every marker into a Gate 1 question, so the human is asked everything
  once, in one place.
- **Freeze the request.** As soon as the feature directory exists, write the
  human's request, verbatim, to `FEATURE/request.md`.

From here on `FEATURE` is the directory specify created.

### spec-review

Load `superspecloop:spec-review-loop` and follow it. It ends at Gate 1 and
you stop with it. When the human approves, it commits the spec; then go on
to the next phase instead of only naming it.

### plan

Run Spec Kit's plan command.

### plan-review

Load `superspecloop:plan-review-loop` and follow it. It normally completes
without stopping; its rulings go to the ledger. Its closing report to the
human is one line here — the rulings reach them at the end.

### tasks

Run Spec Kit's tasks command, asking it to include test tasks before the
implementation tasks of each user story. Then commit the feature directory:
`docs(tasks): <feature name> tasks generated`.

### analyze

Run Spec Kit's analyze command. It reads spec, plan and tasks and reports
inconsistencies; it changes nothing.

- **No CRITICAL findings:** append
  `Analyze: <C> critical, <H> high, <M> medium, <L> low` to the ledger and
  go on. The execution loop's unit brief step reports coverage gaps again.
- **CRITICAL findings that plan or tasks can absorb** (a requirement with no
  task, a task that contradicts the plan): write the findings to
  `WS/analyze-findings.md`, dispatch `superspecloop:doc-editor` (`sonnet`)
  with the plan documents and `tasks.md` as the documents to edit, the spec
  as authority and that file as the items to apply, then run analyze once
  more. Commit, and append the `Analyze:` line with `, 1 remediation round`.
- **A CRITICAL finding that needs the spec to change, or that survives the
  remediation round:** stop and tell the human. The spec is theirs.

### execute

Load `superspecloop:execution-loop` and follow it. When every unit is
complete it hands off to finish-branch on its own.

### finish

Load `superspecloop:finish-branch` and follow it. It ends with the
integration menu and you stop with it.

### done

Say what the ledger's last `Finish:` line says, and stop.

## Resuming

A build can be picked up in a new session at any time:
`/superspecloop:build specs/003-export`. Run `pipeline-phase` and continue
from what it prints. Each phase's skill resumes from the ledger on its own;
do not redo work a ledger line says is done.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "I know where we are — no need to run the phase script" | After a compaction you do not. The script is one call. |
| "The spec looks fine; I'll skip the review to save time" | The review loop is the reason this pipeline exists. Every phase runs. |
| "I'll answer specify's clarification questions myself to keep moving" | Those are the human's questions. Leave the markers for Gate 1. |
| "Gate 1 approved — I'll tell them to run the plan command" | You are the conductor. Run it. |
| "I'll summarize what the plan review decided before moving on" | It is in the ledger and reaches the human at the end. One line, next phase. |
| "Analyze found a gap in the spec; I'll have the editor patch spec.md" | The spec changed only at Gate 1, by the human. Stop and tell them. |
| "The phase stopped to ask something minor; I'll answer and continue" | A phase's stop is your stop. Relay the question. |
