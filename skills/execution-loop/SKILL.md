---
name: execution-loop
description: Use to execute a Spec Kit feature (specs/<feature>/ with spec.md, plan.md, tasks.md) unit by unit — a fresh implementer subagent per unit, a spec-and-quality review after each, and a bounded fix loop — until every task is done and reviewed. Use instead of /speckit.implement.
---

<!-- Derived from Superpowers' subagent-driven-development/SKILL.md (MIT, (c) 2025 Jesse Vincent). See UPSTREAM.md. -->

# Execution Loop

Execute a Spec Kit feature by converting its tasks into units, then
dispatching a fresh implementer subagent per unit, a review (spec compliance
+ code quality) after each, and a bounded fix loop.

**You are the controller.** You coordinate; you do not write code. Subagents
never inherit your session's context — you construct exactly what each one
needs, and hand it over as files. This keeps them focused and keeps your own
context free for coordination.

**Core principle:** fresh subagent per unit + review per unit (spec +
quality) + a ledger that survives compaction = high quality without a human
in the loop.

**Narration:** between tool calls, at most one short line. The ledger and
the tool results carry the record.

**Continuous execution:** do not pause to check in between units. "Should I
continue?" prompts and progress summaries waste the human's time — they
asked you to execute the feature, so execute it.

**Rulings, not stalls.** Conflicts, ambiguities, defects in the units, a cap
you would have asked to exceed — decide them. **spec.md is the binding
authority**, plan.md is its argument, units.md is your restatement of both,
and your judgment settles what none of them answers. Record every decision in
the ledger as `Ruling: <what you decided> — <why> — <what it costs if wrong>`,
and keep going. The literal text `Ruling:` matters — the final report and the
pull-request description are built by searching for it. To change an earlier
ruling, add a new line: `Ruling: amends <which> — ...`. A wrong ruling costs
rework the human can see and undo; a session parked on a question costs
their whole day.

**A ruling binds every unit after it, so every subagent gets the list.**
`WS/rulings.md` is every ruling line in the ledger — the plan review's
included — rewritten each time `task-brief` or `review-package` runs
(`bash scripts/rulings FEATURE` does it on demand). Its path goes in every
implementer and reviewer dispatch. Without it, a decision taken in the plan
review or in Unit 2 exists only in your context: the implementer of Unit 5
breaks it and the reviewer of Unit 5 has no way to notice.

**Four things stop you, and only these:**

1. An irreversible or destructive operation
2. A security-sensitive action
3. A side effect outside this repository's working tree that norms say you
   ask about first (a merge, a push to a shared branch, a publish, a
   production migration)
4. A spec or plan so broken that every path forward is a guess

For those, stop and ask.

## Paths

Scripts live in this skill's directory, `scripts/`. Invoke them with `bash`
(marketplace extraction can strip exec bits). Below, `FEATURE` is the
feature directory, e.g. `specs/001-login`, and `WS` is the workspace
directory `scripts/pipeline-workspace FEATURE` prints.

`WD` is the working directory: the repository root, or the worktree's root
when the feature runs in one. Run every script from `WD`, and name `WD` in
every dispatch — a subagent starts in the session's directory, which is the
wrong checkout when a worktree is in use.

Subagent types (plugin agents): `superspecloop:implementer`,
`superspecloop:task-reviewer`, `superspecloop:re-reviewer`.

## Setup

1. **Branch.** Run `bash scripts/ensure-branch FEATURE` and relay its line.
   On the repository's default branch it creates and switches to a branch
   named after the feature — nothing is ever implemented on `main`. On any
   other branch it stays put. The working tree must be clean except for
   untracked files under `specs/`. If the human asked for an isolated
   workspace, run the `superspecloop:worktree` skill first and use the path
   it reports as `WD`; otherwise work in place.
2. **Workspace.** Run `bash scripts/pipeline-workspace FEATURE`. Everything
   for this feature lives there: units, briefs, reports, review packages,
   ledger. Another feature's directory is never yours to read or write.
3. **Ledger.** Conversation memory does not survive compaction; controllers
   that lost their place have re-dispatched entire completed sequences — the
   most expensive failure observed. Track progress in `WS/progress.md`, not
   only in todos.
   - If it exists and its first line names FEATURE, units with a
     `Unit <N>: complete` line are DONE — do not re-dispatch them; resume at
     the first unit without one. A unit whose last line is a fix round is
     mid-loop: resume at the next round.
   - Otherwise create it with the first line
     `# SuperSpecLoop ledger — feature: FEATURE` and the second line
     `Start: <git rev-parse HEAD> on <branch>`. If the ledger exists (the
     spec or plan review created it), it already has its `Start:` line;
     leave it.
   - After compaction, trust the ledger and `git log` over your recollection.
4. **Units.** If `WS/units.md` does not exist, dispatch one
   `general-purpose` subagent (standard model) with: "Load the
   `superspecloop:tasks-to-briefs` skill and run it for FEATURE. Reply only
   with its report." Record the reply (unit count, already-done tasks, gaps)
   in the ledger.
   - If the reply is empty, read the Gaps and Coverage sections of
     units.md yourself.
   - If it reports failed preconditions (open `[NEEDS CLARIFICATION]`,
     nothing left to do), stop and tell the human.
   - Gaps are rulings you make now: a spec item with no task either gets a
     ruling to extend the nearest unit (ledger it; carry it into that unit's
     dispatch) or is reported to the human in the final message as uncovered.
     Never drop one silently.
5. **Read units.md once.** Create a todo per unit. Copy its Global
   Constraints and Verification Commands sections, verbatim, into
   `WS/global.md`: every dispatch points at that file instead of pasting the
   text again.
6. **Baseline.** Before any unit changes the code, run every gate, not only
   the tests:
   `bash scripts/run-gates FEATURE baseline "<full suite>" "<typecheck>" "<lint>"`
   with the Verification Commands in `WS/global.md`. It writes
   `WS/baseline-gates.txt`. Append one ledger line with the three results:
   `Baseline: suite PASS (<N> tests), typecheck PASS, lint FAIL (<what>)`.
   - A gate already failing is not the implementers' to explain, and nobody
     "fixes" it silently: the line says what fails, and unit gates compare
     against it.
   - A failure that smells intermittent (a timeout, a port, an ordering):
     run it once more. If it passes, append
     `Baseline: flake — <test name> — <what failed>, not reproduced`. A
     flake the code had before the feature started is worth knowing before
     Unit 1, not after the final gates.

### Pre-flight scan

Before dispatching Unit 1, scan units.md for conflicts, writing down what you
checked. The output is a table, not a verdict:

- One row for every pair of units that share a file or an interface: the two
  units, what one produces against what the other consumes, what you found.
- One row for every unit: does its own text agree with itself — tests
  required vs requirements, files created vs files later touched?
- Anything a unit mandates that the review rubric calls a defect (a test
  that asserts nothing, verbatim duplication).

Write the table to the ledger. Rule on every finding before execution, with
the spec as authority, and ledger each ruling. "The scan is clean" without
the rows is not a scan you ran.

## Model Selection

Use the least powerful model that can handle each role. **Always pass the
model explicitly** when dispatching — an omitted model falls back to the
agent's default.

- **Implementers:** `sonnet` by default. `haiku` only for units with no
  logic to get wrong: documentation, configuration, renames, moving files.
  `opus` for units that need design judgment or broad codebase
  understanding.
- **Reviewers:** `sonnet` as the floor; `opus` for a subtle diff
  (concurrency, security, money, data migrations). Scoped re-reviews of
  small fix diffs: `sonnet` or `haiku`.
- **Fix rounds 4–5:** at least one tier above the implementer that got stuck.

A cheaper implementer is not cheaper: in a measured run, `haiku` on logic
units added out-of-scope code, ignored a plan constraint, wrote the same test
scaffolding a dozen times, and the run took three times as long. Record the
model on each dispatch line in the ledger.

## The Unit Loop

Everything you paste into a dispatch — and everything a subagent prints back
— stays in your context for the rest of the session. Hand artifacts over as
files. Never dispatch implementers in parallel.

**Waiting:** while you have local work (ledger, next review package), keep
working. When idle, wait in bounded stretches; between them, post one line
of status and check for subagents that finished without reporting.

### 1. Dispatch the implementer

Record `BASE=$(git rev-parse HEAD)` before dispatching.

Run `bash scripts/task-brief FEATURE N`. It writes `WS/unit-N-brief.md`.
The report file is `WS/unit-N-report.md`.

Dispatch `superspecloop:implementer` with a message containing only:

1. One line on where this unit fits in the feature, and `WD`.
2. The brief path: "Read this first — it is your requirements, with the
   exact values to use verbatim."
3. `WS/global.md` — the Global Constraints and Verification Commands.
4. Interfaces and decisions from earlier units that the brief cannot know.
5. `WS/rulings.md` — "decisions already made for this feature; they bind
   you like the brief" — and, in the message, your resolution of any
   ambiguity you noticed in this unit. A resolution is a ruling: ledger it
   before you run `task-brief`, so it is in the file too.
6. Deferred minors from the ledger on files this unit will modify, as "known
   issues in files you will touch — do not repeat these patterns in new
   code". Fixing the old occurrences stays out of scope.
7. The report file path.

Never paste prior units' history ("state after Units 1–3"). A fresh
subagent needs its unit, the interfaces it touches, and the constraints.

Record the implementer's agent ID — fix rounds 1–3 resume it.

### 2. Handle the report

- **DONE** → three checks, then the review:
  1. `git diff --stat BASE HEAD -- specs .specify` is empty. An implementer
     that touched the spec, the plan or the `tasks.md` checkboxes gets
     resumed to revert that commit; marking tasks is your step 5.
  2. The report has RED evidence copied from command output (the failing
     assertion, not a paraphrase). If missing, resume the implementer once
     to add it.
  3. **Gates.** Run
     `bash scripts/run-gates FEATURE N "<full suite>" "<typecheck>" "<lint>"`
     with the Verification Commands in `WS/global.md`. It saves the full output
     to `WS/unit-N-gates.txt` and prints one PASS/FAIL line per command. Any
     FAIL goes straight back to the implementer (resume it with the gates
     file path) — never spend a reviewer on red gates. Failures the baseline
     already had do not count; say so in the resume message.

     **A failure that does not reproduce is not fixed by editing a test.**
     When a gate fails on a timeout or anything else that smells
     intermittent, run `run-gates` once more before resuming anyone. If it
     passes, append
     `Unit <N>: flake — <test name> — <what failed>, not reproduced` to the
     ledger and go on to the review. Send it to the implementer only if it
     fails again, and then the brief is "find the cause", not "make it
     pass": a test reshaped to dodge a failure nobody understood hides the
     bug, if there is one. Flake lines reach the human in the final report.
- **DONE_WITH_CONCERNS** → read the concerns. Correctness or scope concerns
  get addressed before review; observations ("file is getting large") are
  noted and review proceeds.
- **NEEDS_CONTEXT** → answer completely and resume the same implementer.
- **BLOCKED** → change something: more context (same model), a more capable
  model, split the unit (ledger the split as a ruling), or rule on a defect
  in the brief and carry the ruling in the re-dispatch. Never retry
  unchanged.

### 3. Review the unit

Run `bash scripts/review-package FEATURE BASE HEAD` (the BASE you recorded —
never `HEAD~1`, which drops all but the last commit). The package path on
its first line goes to the reviewer; the package never enters your context.

Dispatch `superspecloop:task-reviewer` with: `WD`, the brief path, the
report path, the gates file path, the diff path with BASE and HEAD,
`WS/global.md`, `WS/rulings.md`, and the review file path
`WS/unit-N-review.md`. The reviewer saves its review there; it is the audit
trail of what was checked.

- Do not ask the reviewer to re-run tests the implementer already ran.
- Do not add open-ended directives ("check all uses") without a concrete,
  unit-specific reason.
- **Never pre-judge findings.** If your message contains "do not flag",
  "don't treat X as a defect", "at most Minor" or "the plan chose" — stop:
  you are sparing yourself a review loop. The rulings file is the only way
  a decision of yours reaches the reviewer: it is the list the human will
  read, and the reviewer may contest any line of it. Do not point at a
  ruling in the message, and never ledger one to head off a finding you
  expect.
- Never accept a review missing either verdict. Implementer self-review
  never replaces this review.

Resolve every ⚠️ "cannot verify from diff" item yourself — you hold the
cross-unit context. A confirmed gap is a failed spec review and joins the
fix loop.

### 4. The fix loop

Triggered by spec ❌, any Critical or Important finding, or a confirmed ⚠️.

Two routes leave before the loop starts:

- **Minor findings** go to the ledger as
  `Unit <N>: minor (deferred): <one-liner>`. They never enter the loop; the
  final review triages them.
- **Plan-mandated findings** (or any finding that conflicts with the brief's
  text) are yours to rule on, with the spec as authority. Ledger the ruling
  before acting. Never dismiss a finding because the brief mandates it, and
  never dispatch a fix that contradicts the brief without a recorded ruling.

Everything else enters the loop. **A round = one fix dispatch + one scoped
re-review. Five rounds maximum per unit.**

- **Rounds 1–3:** resume the original implementer (SendMessage to its agent
  ID) with the open findings verbatim. If it cannot be resumed, dispatch a
  fresh one with the brief, `WS/global.md`, `WS/rulings.md`, the report file
  and the findings — the report file is the persistent memory either way.
- **Rounds 4–5:** dispatch a fresh implementer on a more capable model, with
  the brief, `WS/global.md`, `WS/rulings.md`, the report file, the open
  findings and: "A prior implementer
  attempted this unit [N] times; you own it now. Read the report file for
  what was tried."
- **Every round:** before re-review, confirm the appended fix report has the
  covering tests, the command and the output.
- **Re-review is scoped:** `bash scripts/review-package FEATURE FIX_BASE HEAD`
  (FIX_BASE = the head the previous review saw), then dispatch
  `superspecloop:re-reviewer` with `WD`, the brief, the findings verbatim, the
  report file, the diff path, `WS/rulings.md`, a fresh gates file (after the
  fix, run
  `run-gates FEATURE N-round-R ...`, which writes
  `WS/unit-N-round-R-gates.txt`; red gates go back to the implementer and
  still count as the round) and its own review file
  `WS/unit-N-review-round-R.md`. New Critical/Important breakage in the fix
  diff joins the open list. Out-of-scope observations go to the ledger as
  deferred minors.
- **After each round** append:
  `Unit <N>: fix round <R>/5 (<X> addressed, <Y> open — <one-liners>; commits <a7>..<b7>)`

Never fix findings yourself — controller fixes skip review.

**The breaker.** When round 5 still leaves findings open, stop dispatching
and adjudicate each one:

- **Reviewer wrong, or contestable:** `Unit <N>: parked — <finding> — Ruling: <why the code stands>`
- **Real, nothing downstream builds on it:** park it the same way, ruling
  that it is real and deferred.
- **Real and load-bearing** (a later unit builds on it, or it reveals a spec
  or plan defect): rule on the smallest change that unblocks dependent work,
  ledger it as `Unit <N>: Ruling: ...`, carry it into the next dispatch.
  Stop only when every path forward is a guess.

Adjudicate only at the cap. Adjudicating earlier to end a loop is
pre-judging under another name. A silent discard is forbidden.

### 5. Complete the unit

When the review is clean — or every open finding is parked with a ruling at
the cap:

1. Mark its Spec Kit tasks done:
   `bash scripts/mark-tasks FEATURE T010 T012 T013` (the unit's "Covers
   tasks"), then commit tasks.md: `chore(tasks): mark T010, T012, T013 done`.
2. Append to the ledger:
   `Unit <N>: complete (commits <base7>..<head7>, review clean)` or
   `Unit <N>: complete (commits <base7>..<head7>, <K> parked)`.
3. Mark the todo complete and dispatch the next unit.

Never move on while the review has open Critical/Important findings that are
neither fixed nor parked with a ruling at the cap.

## When Every Unit Is Complete

Load the `superspecloop:finish-branch` skill and follow it. It re-runs the
gates on the final tree, dispatches the whole-branch review against the
spec, runs one wave of fixes, builds the pull-request description, and asks
the human how to integrate. Do not write a final summary before it: its
report carries the rulings, and the deferred minors only get triaged there.

Only if that skill cannot be loaded, stop here with a message containing:

- Units completed and the commit range
- **Rulings I made** — every ledger line containing `Ruling:`, in order, each
  with what it costs if wrong. The list is exhaustive: it is the only place
  the decisions you took on the human's behalf reach them.
- Deferred minors and parked findings
- Spec items reported as uncovered, if any

Keep the workspace until the branch is merged; the ledger is the recovery map.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Close enough on spec compliance" | Spec gaps = not done. Fix, or hit the cap and adjudicate. |
| "I'll fix it myself, dispatching is overhead" | Controller fixes pollute your context and skip review. |
| "One more round will converge" | Past the cap the failure is structural. Adjudicate and route. |
| "The reviewer will just find something new" | Re-reviews are scoped; new findings on untouched code go to the ledger. |
| "This finding is obviously wrong, I'll drop it" | Adjudicate only at the cap, and ledger it. |
| "The fix was small, skip the re-review" | Unreviewed fixes are how regressions land. |
| "The spec gap is small, I'll just add it to a unit" | Fine — as a ledgered ruling carried in the dispatch, never silently. |
| "Ledger bookkeeping is overhead" | The ledger is what survives compaction. |
| "This reviewer doesn't need the rulings; none is about this unit" | You are guessing which ones apply. The file is twenty lines; the reviewer decides. |
| "Typecheck and lint were clean last time; the baseline is the test suite" | Then a lint error in Unit 1's gates is nobody's or everybody's. Run all three before the first unit. |
