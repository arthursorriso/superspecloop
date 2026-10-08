---
name: plan-review-loop
description: Use right after Spec Kit's plan command has written plan.md and before generating tasks — reviews the plan against the approved spec and against the existing codebase in a bounded loop (uncovered requirements, reinvented code, broken conventions, over-engineering), fixes what the spec and the code already settle, and rules on or escalates the rest.
---

# Plan Review Loop

Make a Spec Kit plan deliver every item of the spec, reuse what the codebase
already has, follow how this repository does things, and add nothing the
spec does not need — before it is broken into tasks.

**Why here:** this is where duplication is prevented instead of found. A
helper the plan reinvents becomes a task, then code, then a review finding,
then a fix. One search of the codebase now is cheaper than all of that.

**You are the controller.** A reviewer with clean context judges the plan
against the spec and the code; an editor applies its findings; you move
files and keep the ledger. You do not edit the plan and you do not review it.

**Decisions here are technical, so you may rule on them** — unlike in the
spec review, where every open decision belongs to the human. The spec is the
binding authority, the constitution binds next, the codebase's existing
practice after that; your judgment settles what none of them answers. Every
decision is a ledger line

`Plan round <R>: Ruling: <what you decided> — <why> — <what it costs if wrong>`

where `<R>` is the edit round that applies it. **The ledger line is the
ruling.** The editor's rulings file is copied from the ledger by a script,
and so are the final report and the pull-request description: a decision
with no ledger line reaches nobody, the editor included.

**Three things stop you:**

1. **The spec has to change.** A spec gap or a spec error is never yours to
   fix, and never the plan's to paper over. The human approved that
   document at Gate 1; only they change it.
2. **A decision that is expensive to reverse and could reasonably go either
   way** — a new external service, a storage engine, a data migration, a
   public API shape, a security model.
3. **Every path forward is a guess.**

For those, stop and ask. Everything else, rule and continue.

## Paths

`FEATURE`, `WD` and `WS` as in the spec review loop
(`bash ../execution-loop/scripts/pipeline-workspace FEATURE`). This skill's
script is `scripts/round-rulings`. Invoke scripts with `bash`.

Subagent types: `superspecloop:plan-reviewer` (`opus`),
`superspecloop:doc-editor` (`sonnet`). Pass the model explicitly.

## Setup

1. `FEATURE/spec.md` and `FEATURE/plan.md` must exist.
2. **Ledger** `WS/progress.md`: create it if missing (first line
   `# SuperSpecLoop ledger — feature: FEATURE`), otherwise resume from its
   last `Plan review:` line. If it has `Plan review: complete`, there is
   nothing to do.
3. **Branch and start point.** Run
   `bash ../execution-loop/scripts/ensure-branch FEATURE` and relay its line
   to the human. On the repository's default branch it creates and switches
   to a branch named after the feature, so nothing this pipeline commits
   lands on `main`; on any other branch it stays put and says where commits
   will land. When you create the ledger, make its second line
   `Start: <git rev-parse HEAD> on <branch>` — the finish step reviews the
   range from there, which is what keeps a feature stacked on another
   branch from being reviewed together with it.
4. If the ledger has no `Gate 1: approved` line, the spec was not approved
   through the spec review loop. Say so in one line and continue: a plan can
   be reviewed against any spec, but its findings are only as good as the
   spec is.
5. If spec.md still contains `[NEEDS CLARIFICATION`, stop: that is a
   question for the human, and planning on top of it builds on a guess.

## The Loop

Review files: `WS/plan-review-1.md`, `-2.md`, ... Edit reports:
`WS/plan-edit-1.md`, ...

### 1. Full review

Dispatch `superspecloop:plan-reviewer` with: `WD`; the spec file; the plan
documents that exist (`plan.md`, `research.md`, `data-model.md`,
`contracts/`, `quickstart.md`); the constitution path if there is one
(`.specify/memory/constitution.md`); mode `full`; and the review file.

Nothing else. Do not describe the codebase to it or point it at "the
relevant" directories: finding what already exists is its job, and a search
you narrowed is a search that misses.

### 2. Rule on what is yours

Before any edit, handle the kinds of finding the editor cannot. Append each
ruling to the ledger as a `Plan round <R>: Ruling: ...` line the moment you
make it, one line per decision. Say what is decided in words — "adopt
Suggestion 3: list `tasks.test.ts` among the files the plan changes", not
"adopt suggestions 1 to 5" — because the human reads that line in the pull
request, without the review file beside it. The same goes for rulings a
later review round calls for.

- **`## Suggestions`** — normally left alone. Adopt one as a ruling only
  when leaving it out would make a task fail or be built wrong (the plan
  forgets a file that must change, a test that will break). Better is not
  enough.
- **`## Questions`** — a technical trade-off. If it meets a stop condition
  above, stop and ask the human. Otherwise rule: usually the reviewer's
  recommended option, unless the spec, the constitution or existing practice
  points elsewhere. Ledger the ruling.
- **`## Spec Gaps`** — stop condition 1, with one exception: if the plan can
  be made correct without the spec saying anything different (the gap is
  the plan assuming more than the spec states), rule that the plan drops the
  assumption. If the spec itself must change, stop and tell the human what
  is missing, where, and what the plan needs to know.
- **A Blocking finding that contradicts the spec** — the reviewer is wrong
  or the spec is; rule with the spec as authority, and ledger it.

### 3. Edit rounds — at most 3

While there are Blocking findings or rulings to apply, and fewer than 3 edit
rounds have run:

1. `bash scripts/round-rulings FEATURE R` writes `WS/plan-rulings-R.md`
   from this round's ledger lines and prints how many it found. Never write
   or edit that file yourself: if a ruling is missing from it, the ledger
   line is missing — append it and run the script again.
2. Dispatch `superspecloop:doc-editor` with: `WD`; the documents to edit
   (the plan documents — never spec.md); the authority (the spec file); the
   items to apply ("every finding under `## Blocking` in <review file>, and
   the rulings in <rulings file>"); and the edit report `WS/plan-edit-R.md`.
3. Dispatch `superspecloop:plan-reviewer` in mode `re-review` with the same
   inputs plus the previous review file, the rulings file and the edit
   report.
4. Append:
   `Plan review: round <R>/3 (<B> blocking → <A> applied, <N> not applied; <K> ruling(s))`
   with `<K>` as the script printed it.

A ruling made when no edit round will run (the plan stands as it is) still
gets its ledger line, numbered as the next round.

Only Blocking findings reopen the loop. Suggestions never do.

**At the cap**, adjudicate each Blocking finding still open, as the
execution loop's breaker does:

- reviewer wrong or contestable → `Plan: parked — <finding> — Ruling: <why the plan stands>`
- real, and the tasks can still be built correctly → park it the same way,
  ruling that it is real; it surfaces again in the code review if it matters
- real, and the tasks would be built wrong → stop and tell the human. A plan
  that will not converge on something load-bearing is not ready.

### 4. Complete

1. Append `Plan review: complete (<R> round(s), <K> ruling(s), <P> parked)`,
   with `<K>` the total the last `round-rulings` run printed.
2. Commit the plan documents:
   `git add FEATURE && git commit -m "docs(plan): <feature name> plan reviewed"`.
3. Report to the human in a few lines: rounds run; what the review changed
   that they would care about (existing code the plan now reuses, anything
   removed as unneeded); every ruling with what it costs if wrong; anything
   parked. Then the next step: Spec Kit's tasks command, its analyze
   command, and the `superspecloop:execution-loop` skill.

If the `superspecloop:build` skill is conducting this run, shorten that
report to one line and return to it: it runs the tasks next, and the rulings
reach the human at the end.

This skill does not stop for approval when it completes cleanly. The human
approved the spec; the plan is how, and its decisions are on the record.
If they asked to review the plan before tasks are generated, stop here and
wait for them.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The spec is clearly wrong here — I'll have the editor fix it while it's in there" | The human approved the spec. Stop and tell them. |
| "The plan can just assume an answer to the spec gap" | Then the gap is built in, invisibly. Drop the assumption or ask. |
| "I'll point the reviewer at the two directories that matter" | The reinvented helper is in the directory you did not think of. |
| "This trade-off is technical, so I'll rule on the database choice too" | Expensive to reverse and it could go either way: that one is the human's. |
| "The reviewer's Reinvention finding is a stretch; the new helper is cleaner" | Existing code wins unless you can ledger why not. "Cleaner" is how duplicates are born. |
| "Three rounds, one finding left, it's minor anyway" | Then park it with a ruling. Dropping it silently is not an option. |
| "I'll fix that one line of plan.md myself" | Controller edits skip review. Dispatch the editor. |
| "The rulings are in the rulings file; the ledger line is duplication" | The file is built from the ledger. Written by hand, it reached the editor and nobody else: eight decisions never made it to the pull request. |
