---
name: spec-review-loop
description: Use right after Spec Kit's specify command has written spec.md and before planning — reviews the spec against the human's original request in a bounded loop (missing items, unrequested scope, untestable criteria, ambiguity), turns every open decision into a question, and stops at Gate 1 for the human's answers and approval.
---

# Spec Review Loop

Make a Spec Kit spec say exactly what the human asked for — nothing missing,
nothing added, every criterion checkable — and put in front of them the few
decisions only they can make.

**Why here:** an error in the spec is implemented faithfully by every stage
after it. No code review can catch "built the wrong thing correctly". This
is the cheapest review in the whole pipeline, and Gate 1 is the human's
highest-value five minutes.

**You are the controller.** A reviewer with clean context judges the spec; an
editor applies its findings; you move files between them and keep the
ledger. You do not edit the spec, and you do not review it.

**The one rule that has no exception: you never decide what the human
meant.** In the execution loop a controller rules and keeps going, because a
wrong ruling there is rework someone can see. Here a wrong guess is
invisible — it becomes the spec. So in this skill there are no rulings on
intent: an open decision is a question at Gate 1, whether or not anyone is
watching, however obvious the answer looks, and even when the reviewer
recommends an option.

## Paths

`FEATURE` is the feature directory (e.g. `specs/001-login`); `WD` the
working directory; `WS` the workspace
`bash ../execution-loop/scripts/pipeline-workspace FEATURE` prints. This
skill's script is `scripts/gate-report`. Invoke scripts with `bash`.

Subagent types: `superspecloop:spec-reviewer` (`opus`),
`superspecloop:doc-editor` (`sonnet`). Pass the model explicitly.

## Setup

1. `FEATURE/spec.md` must exist. If the ledger `WS/progress.md` has a
   `Gate 1: approved` line, there is nothing to do — say so and stop.
2. **Ledger.** Create `WS/progress.md` if it does not exist, with the first
   line `# SuperSpecLoop ledger — feature: FEATURE`. The execution loop
   continues this same file later. If it exists, resume: the last
   `Spec review:` or `Gate 1:` line says where you are.
3. **Branch and start point.** Run
   `bash ../execution-loop/scripts/ensure-branch FEATURE` and relay its line
   to the human. On the repository's default branch it creates and switches
   to a branch named after the feature, so nothing this pipeline commits
   lands on `main`; on any other branch it stays put and says where commits
   will land. When you create the ledger, make its second line
   `Start: <git rev-parse HEAD> on <branch>` — the finish step reviews the
   range from there, which is what keeps a feature stacked on another
   branch from being reviewed together with it.
4. **Freeze the request** in `FEATURE/request.md`. It is the yardstick for
   every round, so it must be the human's words, not a paraphrase and not
   the spec's own summary:
   - If the file exists, use it.
   - Else, if the human's request is in this conversation, write it there
     verbatim.
   - Else, copy the text of the `**Input**: User description:` line of
     spec.md, and append `Spec review: request reconstructed from spec.md`
     to the ledger so Gate 1 says so.
   - If none of these yields a request, ask the human for it.

   Never edit request.md afterwards. Later answers and changes of mind go
   into the spec's Clarifications, not into the request.

## The Loop

Review files are numbered in order: `WS/spec-review-1.md`, `-2.md`, ...
Edit reports: `WS/spec-edit-1.md`, ...

### 1. Full review

Dispatch `superspecloop:spec-reviewer` with: `WD`, the request file, the spec
file, mode `full`, and the review file `WS/spec-review-1.md`.

Give it nothing else: no summary of the feature, no account of how the spec
was written, no hint about what to look for or to overlook. Its value is
that it reads the request and the spec cold.

### 2. Edit rounds — at most 3

While the latest review has findings under `## Blocking`, and fewer than 3
edit rounds have run:

1. Dispatch `superspecloop:doc-editor` with: `WD`; the document to edit
   (`FEATURE/spec.md`, and `FEATURE/checklists/` if the spec's own checklist
   must follow); the authority (`FEATURE/request.md`); the items to apply
   ("every finding under `## Blocking` in <review file>"); and the edit
   report `WS/spec-edit-R.md`.
2. Dispatch `superspecloop:spec-reviewer` in mode `re-review` with: `WD`,
   the request file, the spec file, the previous review file, the edit
   report, and the next review file.
3. Append to the ledger:
   `Spec review: round <R>/3 (<B> blocking → <A> applied, <N> not applied; <Q> question(s) open)`

Only Blocking findings reopen the loop. Suggestions never do. Questions
never do — they wait for the human.

**At the cap**, if Blocking findings remain, stop looping. Do not adjudicate
them and do not ask for a fourth round: a spec that will not converge
usually means the request is unclear, and that is the human's to see. They
appear at Gate 1 as unresolved findings.

### 3. Gate 1

`bash scripts/gate-report FEATURE` writes `FEATURE/gate-1.md` from the latest
review, the edit reports and the ledger, and prints how many questions are
open. Never write or edit that file yourself.

Append `Gate 1: waiting — <Q> question(s)` to the ledger. Then stop, and tell
the human, in this order:

1. One line: the spec was reviewed against their request in `<R>` round(s).
2. The open questions, each in one or two lines with its options — enough to
   answer from the chat without opening a file. When the reviewer says what
   an option rules out or costs, that goes in the line: the human can only
   accept a consequence they were shown, and nobody may later say they did.
3. Anything removed as not requested, in one line, so they can ask for it
   back.
4. The branch the approved spec will be committed to, if `ensure-branch`
   said it is not named after the feature — they may want a new one.
5. The path of `FEATURE/gate-1.md`, which also has the traceability table.
6. What you need: answers to the questions, or — with no open questions —
   their approval of the spec.

Gate 1 always stops, even with zero questions and a clean review. The spec
is the contract for everything after it; a human approves it.

### 4. After the human replies

**They answered questions** (in the chat, or by filling the Answer lines in
gate-1.md):

1. Write the answers verbatim, numbered as the questions were, to
   `WS/spec-answers-<n>.md`. The file is a transcript: under each number,
   the human's words and nothing of yours. When the answer is only a label
   ("B", "the recommended one"), add the option's text copied from
   gate-1.md, marked as copied:

   ```markdown
   ## Q2
   Human: B
   Option B (copied from gate-1.md): <the option's text, unchanged>
   ```

   No consequences, no "which means", no "accepted": the editor and the
   reviewer read this file as the human's intent, so a sentence you add is
   a decision you made in their name. If an answer has a consequence they
   may not have seen, leave it out — the re-review raises it as a new
   question if it matters, and Gate 1 comes round again.
2. Dispatch `superspecloop:doc-editor` with the same documents; the
   authority (the request file and the answers file); the items to apply
   (the answers); and the next edit report.
3. Dispatch `superspecloop:spec-reviewer` in `re-review` mode as before,
   adding every answers file written so far.
4. Append `Gate 1: <K> answer(s) applied`.
5. An answer often has consequences the spec does not cover yet (a rule
   with no scenario that tests it, a case the answer leaves open), so the
   re-review may find new Blocking findings. Fix them in edit rounds
   exactly as in "Edit rounds" above — editor, re-review — with a fresh cap of 3 for
   this pass, and append after each round:
   `Spec review: round <R>/3 after answers (<B> blocking → <A> applied, <N> not applied; <Q> question(s) open)`
   Every round gets its line: the review files and the ledger must tell
   the same story. At the cap, stop as in the first pass.
6. Run `gate-report` again and return to Gate 1.

**They asked for a change** ("also add X", "drop the filter"): that is new
intent. Write it to an answers file and apply it the same way. Do not touch
request.md.

**They approved with questions still open:** approval does not answer them.
Ask once, naming the questions: "Use the recommended option for Q2 and Q3?"
A yes is an answer — apply it as above.

**They approved and nothing is open:**

1. Append `Gate 1: approved` to the ledger.
2. Commit the feature's documents, and only those:
   `git add FEATURE && git commit -m "docs(spec): <feature name> approved at gate 1"`.
3. If the `superspecloop:build` skill is conducting this run, return to
   it: it runs the plan next. Otherwise tell them the next step: Spec Kit's
   plan command, then the `superspecloop:plan-review-loop` skill.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The recommended option is obviously right — I'll apply it and save them a question" | Then it costs them one word to confirm. Applying it yourself is deciding what they meant. |
| "Nobody is watching; a question would stall the run" | This gate is meant to stall. Stop with the question on the table. |
| "The reviewer missed something I can see — I'll tell the editor" | You are not the reviewer. If it matters, it will surface at Gate 1 or in the plan review. |
| "I'll tell the reviewer what the feature is about, to save it time" | Its value is reading the request cold. Your summary is the author's reasoning, leaking in. |
| "The Extra is a nice feature; removing it is a loss" | The request is the yardstick. It is listed at Gate 1; the human can ask for it back in one line. |
| "Three rounds and one Blocking left — one more will do it" | The cap is the signal. Show the human the unresolved finding. |
| "The rounds after the answers are follow-up, not worth a ledger line" | In one run three of them left no line; the metrics showed one round where five reviews ran. Every round is a line. |
| "No questions and a clean review — I'll go straight to the plan" | Gate 1 always stops. |
| "I'll tidy gate-1.md a little" | The script builds it so nothing can be summarized away. Fix the source and run it again. |
| "The request was vague, so I'll write a better one into request.md" | The request file is the human's words, frozen. Vagueness becomes questions. |
| "I'll note in the answers file what their choice implies, to help the editor" | Then the editor applies your inference as their intent, and the final report says they accepted it. Their words only. |
