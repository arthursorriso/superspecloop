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
- **Mode.** If the human asked for per-unit reviews ("with unit reviews",
  "full mode"), the ledger gets `Mode: full` when it is created; otherwise
  the execution loop writes `Mode: lean`. Say which in your first line.
- **Model.** You coordinate; the reviews that need the most capable model
  run on it by themselves. If this session is on the most capable model,
  say once that a mid-tier session model (`/model sonnet`) costs much less
  for the conductor's work, and carry on.

## Phases

Load a phase's skill when its phase starts, not before.

**Spec Kit's commands run in subagents, not here.** Each one reads the
feature's documents and writes new ones — the plan, the tasks, the analysis.
Run in this session, every page of that would ride in your context to the
end of the build, and every later step would pay for it again: in one
measured run, two fifths of the tokens were spent with more than 150k of
context. A dispatched command costs the same once and leaves one line here.
Each dispatch is a `general-purpose` subagent (`sonnet`) told: the working
directory; the feature directory; the command to run — "load the
`speckit-plan` skill and follow it", or, on an older install, "read
`.claude/commands/speckit.plan.md` and follow it"; the absolute path of the
execution-loop scripts directory (`../execution-loop/scripts/` from this
skill); "the human cannot be reached from where you run: never ask them
anything, and never answer a question the command would put to them"; and
the one-line reply to give.

### specify

Dispatch Spec Kit's specify command with the human's request, verbatim, and
these instructions: "Do not run its clarification round: leave every
`[NEEDS CLARIFICATION]` marker in the spec. Reply with one line:
`FEATURE: <the feature directory it created>`." The spec review turns every
marker into a Gate 1 question, so the human is asked everything once, in one
place.

When it replies, **freeze the request**: write the human's request, verbatim
— from their message, not from the subagent — to `FEATURE/request.md`.

From here on `FEATURE` is that directory.

### spec-review

Load `superspecloop:spec-review-loop` and follow it here: Gate 1 is a
conversation with the human. It ends at Gate 1 and you stop with it. When
the human approves, it commits the spec; then go on to the next phase
instead of only naming it.

### plan

Dispatch Spec Kit's plan command for FEATURE. Reply:
`PLANNED: <the documents it wrote>`, or `STOP: <what it needs decided>` if
it cannot plan without a decision. Append
`Plan: generated (<the documents>)` to the ledger with
`bash ../execution-loop/scripts/ledger FEATURE "<line>"`.

### plan-review

Run it in its own context, so its reviews and rulings never enter yours:
dispatch one `general-purpose` subagent (`opus` — it rules on technical
trade-offs) with: "Load the `superspecloop:plan-review-loop` skill and run it
for FEATURE in WD. The `superspecloop:build` skill dispatched you: reply
with one line, as the skill says." Add the answers file path when you are
dispatching it again after the human answered.

If `pipeline-phase` says plan-review is **waiting for the human's answers**
(a resumed session), do not dispatch: read the newest
`WS/spec-questions-*.md` that has no `-answers.md` beside it, put its
questions to the human, and continue from their answers as below.

Its reply:

- `COMPLETE: ...` → next phase.
- `STOP: <N> spec question(s) — <file>` → the plan needs the spec to change,
  or would depart from it. Read the file and put each question to the human
  in one or two lines with its options, then wait. Write their answers as a
  transcript to `<file without .md>-answers.md` — under each `## S<k>`, their
  words and nothing else; for a bare label, the option's text copied from the
  questions file, marked as copied — and dispatch the subagent again with
  that path. It applies them to the spec and continues the review.
- `STOP: <anything else>` → relay it and wait, like any phase stop.

### tasks

Dispatch Spec Kit's tasks command for FEATURE, "with test tasks before the
implementation tasks of each user story", and: "Then commit the feature
directory: `docs(tasks): <feature name> tasks generated`. Reply:
`TASKS: <count> task(s), <commit sha>`."

### analyze

Dispatch one subagent for the whole phase, with these instructions, and act
on its one-line reply:

> Run Spec Kit's analyze command for FEATURE. It reads spec, plan and tasks
> and reports inconsistencies; it changes nothing. It ends by offering to
> suggest remediation edits — that offer is for a human running it by hand.
> Do not answer it; decide by these rules:
>
> - **No CRITICAL or HIGH findings:** reply
>   `ANALYZED: <C> critical, <H> high, <M> medium, <L> low`.
> - **CRITICAL or HIGH findings that the plan or the tasks can absorb** (a
>   requirement with no task, a task that contradicts the plan, a term used
>   two ways): write them, with their locations and the fix the report
>   gives, to `WS/analyze-findings.md` (`WS` is what
>   `bash <scripts directory>/pipeline-workspace FEATURE` prints).
>   Dispatch `superspecloop:doc-editor` (`sonnet`) with the plan documents
>   and `tasks.md` as the documents to edit, `FEATURE/spec.md` as the
>   authority, that file as the items to apply, and `WS/analyze-edit-1.md`
>   as its report. Run analyze once more. Commit the feature directory:
>   `docs(tasks): apply analyze findings to <feature name> tasks`. Reply
>   `ANALYZED: <counts from the second run>, 1 remediation round`.
> - **A CRITICAL or HIGH finding that needs the spec to change, or that is
>   still there after the remediation round:** change nothing more and reply
>   `STOP: <the finding, its location, and what the spec would have to say>`.
>   Never edit spec.md.

On `ANALYZED: ...`, append `Analyze: <the rest of its line>` to the ledger
and go on; MEDIUM and LOW findings stay in the analysis, and the execution
loop's unit brief step reports coverage gaps again. On `STOP: ...`, tell the
human and wait: the spec is theirs.

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
| "Running the plan command here is simpler than dispatching it" | Then the plan, the tasks and the analysis ride in your context to the end of the build, and every later step pays for them again. Dispatch. |
| "Analyze asked whether to suggest remediation — I'll ask the human" | That offer is for a person running it by hand. The phase's rules decide: remediate what plan and tasks can absorb, stop only for the spec. |
