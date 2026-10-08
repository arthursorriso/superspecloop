# Upstream

SuperSpecLoop is inspired by [Superpowers](https://github.com/obra/superpowers) (MIT, © Jesse Vincent).
This file records where each derived part came from, so improvements to the original can be ported with a targeted diff.

## Current reference

| Field | Value |
|---|---|
| Repository | https://github.com/obra/superpowers |
| Base commit | `8ca22dba9a94f28898bbce59f2537ff4d87c747d` |
| Commit date | 2026-09-25 |
| Version | 6.4.2 |

## Source map

| File here | Source in Superpowers | Status |
|---|---|---|
| `skills/execution-loop/SKILL.md` | `skills/subagent-driven-development/SKILL.md` | adapted (stage 2) |
| `agents/implementer.md` | `skills/subagent-driven-development/implementer-prompt.md` | adapted (stage 2) |
| `agents/task-reviewer.md` | `skills/subagent-driven-development/task-reviewer-prompt.md` | adapted (stage 2) |
| `agents/re-reviewer.md` | `skills/subagent-driven-development/re-review-prompt.md` | adapted (stage 2) |
| `skills/execution-loop/scripts/review-package` | `skills/subagent-driven-development/scripts/review-package` | adapted (stage 2) |
| `skills/execution-loop/scripts/task-brief` | `skills/subagent-driven-development/scripts/task-brief` | adapted (stage 2) |
| `skills/execution-loop/scripts/pipeline-workspace` | `skills/subagent-driven-development/scripts/sdd-workspace` | adapted (stage 2) |
| `skills/worktree/SKILL.md` | `skills/using-git-worktrees/SKILL.md` | adapted (stage 4) |
| `skills/finish-branch/SKILL.md` | `skills/finishing-a-development-branch/SKILL.md` + the Final Review section of `skills/subagent-driven-development/SKILL.md` | adapted (stage 4) |
| `agents/branch-reviewer.md` | `skills/requesting-code-review/code-reviewer.md` | adapted (stage 4) |

Original parts (no upstream source): `skills/build`, `skills/spec-review-loop` (with `scripts/gate-report`), `skills/plan-review-loop`, `skills/tasks-to-briefs`, `agents/spec-reviewer.md`, `agents/plan-reviewer.md`, `agents/doc-editor.md`, and the scripts `mark-tasks`, `run-gates`, `ensure-branch`, `pipeline-phase`, `md-section`, `rulings`, `round-rulings`, `pr-body` and `run-metrics`. Not built: a separate test-writing agent and a hook that blocks implementers from editing tests (see README).

## How to sync

1. Clone or update Superpowers locally.
2. Diff only the source files listed in the map above:
   ```bash
   git diff <base-commit> HEAD -- skills/subagent-driven-development skills/using-git-worktrees skills/finishing-a-development-branch skills/requesting-code-review
   ```
3. Read the `RELEASE-NOTES.md` entries for that range.
4. Port whatever makes sense by hand and log it below.
5. Update the base commit in the reference table.

## Sync history

| Date | Upstream commit | What was ported |
|---|---|---|
| 2026-09-29 | `8ca22db` | Initial base (nothing ported yet) |
| 2026-09-29 | `8ca22db` | Execution engine adapted to Spec Kit (stage 2) |
| 2026-10-05 | `8ca22db` | Worktree, whole-branch review and branch finishing (stage 4) |

## Intentional differences from the original

Record every behavioral divergence here, so it is not undone by accident when porting improvements.

- **Input is Spec Kit, not a Superpowers plan.** The plan becomes `units.md` through the `tasks-to-briefs` skill; `task-brief` extracts `### Unit N`, not `### Task N`.
- **Workspace in `.pipeline/<feature>/`** instead of `.superpowers/sdd/<plan>/`, with a `feature-path` marker.
- **spec.md is the authority** for rulings; the original uses the spec the plan points to.
- **Prompts are plugin agents** (`agents/`) instead of templates pasted by the controller. The implementer has no tool to dispatch agents, so the "no subagents" rule is enforced by its tool list, not only by its text.
- **Reviewer has extra checks:** requirement without a test (Untested), exact value that drifted from the brief (Value drift), and duplication of existing code as mandatory named risks.
- **Baseline of every gate** (suite, typecheck, lint) recorded in the ledger before the first unit, through `run-gates`. It used to be the test suite alone, run before the verification commands were known; a lint error or an intermittent test the code already had then looked like the first unit's doing.
- **Spec Kit tasks marked `[X]`** when each unit completes (`mark-tasks`).
- **Whole-branch final review lives in `finish-branch`**, not in the loop. The reviewer is a plugin agent (`branch-reviewer`) that also builds a spec-coverage table (every requirement, scenario and edge case → code and test) and triages the deferred minors into "fix before merge" and "can wait".
- **The pull-request description is assembled by a script** (`pr-body`) from the reviewer's coverage table, the final gate run and every `Ruling:` line in the ledger, instead of being composed by the controller.
- **Finish menu puts the pull request first**, since that is what the pipeline is for; upstream lists the local merge first.
- **Worktrees are opt-in.** Upstream offers one before every plan; Spec Kit already isolates the feature on a branch, and a worktree needs the spec documents committed and the main checkout moved off the feature branch. The worktree directory is ignored with a self-ignoring `.gitignore` rather than a commit to the base branch.
- **Dispatches name a working directory and a `global.md` file** instead of pasting the global constraints each time.
- **Review packages omit the feature's documents, `.specify/` and lockfiles** from the diff body.
- **Review package carries new-file line numbers.** With a plain diff, reviewers cited the package's own line numbers as `file:line` (observed in the first sandbox run).
- **Reviews are saved to `unit-N-review.md`**, and each re-review to `unit-N-review-round-R.md` (reviewers get `Write` for that one file), so a run can be audited afterwards. The final message is still the report. One shared file was tried first; a re-reviewer overwrote the original review.
- **Implementers never touch `specs/` or `.specify/`.** The controller checks the range before review; in the first sandbox run an implementer committed the `tasks.md` checkmarks itself.
- **Report evidence is checked on the first delivery**, not only in fix rounds: summary lines copied from command output.
- **A trivial Setup phase merges into the first unit that uses it** instead of costing its own implementer and reviewer.
- **Gates run by the controller (`run-gates`) before every review.** Upstream tells reviewers to trust the implementer's reported test run; in both sandbox runs every reviewer answered "cannot verify the gates" and the controller re-ran them ad hoc. Red gates go back to the implementer without spending a reviewer.
- **Implementers default to `sonnet`**; `haiku` only for units without logic. Upstream allows the cheapest tier whenever the plan contains the complete code, which Spec Kit plans never do.
- **Test scaffolding copied 3+ times in a diff is Important**, and deferred minors on files a unit touches are carried into its dispatch. In the second sandbox run the same try/catch block was flagged Minor in four units and ended up copied about twenty times.
- **One unit per phase by default.** Two runs of the same feature produced 6 and 9 units; the 9-unit run cost more and needed a reordering ruling.
- **Spec and plan are reviewed before execution** (`spec-review-loop`, `plan-review-loop`). Upstream reviews its own design and plan documents inside `brainstorming` and `writing-plans`, which this plugin does not use. The spec loop is the one place where the controller makes no rulings: every open decision is a question at Gate 1.
- **`ensure-branch` creates the feature branch when the pipeline starts on the default branch.** Recent Spec Kit versions stopped creating one, and in the first spec-loop run the approved spec was committed to whatever branch was current.
- **The final review covers `Start:`..HEAD, not merge-base..HEAD.** Every loop records the commit it started from in the ledger, so a feature stacked on an unmerged branch is not reviewed together with the feature under it.
- **A gate failure that does not reproduce is logged as a flake, not "fixed".** In the first run on existing code, a one-off test timeout sent the unit back to its implementer, who reshaped a passing test from an earlier unit without ever finding a cause.
- **Every implementer and reviewer gets the rulings file** (`rulings.md`, rebuilt from the ledger by `task-brief` and `review-package`). Upstream carries decisions in the dispatch text, chosen by the controller. In one run new code broke an earlier ruling and only the whole-branch review noticed: the unit reviewer had never seen it.
- **Plan rulings are ledger lines first; the editor's rulings file is generated from them** (`round-rulings`). Written by hand, the file reached the editor while eight of nine decisions never reached the ledger, and so never reached the final report or the pull request.
- **Gate 1 answers are stored as a transcript**, and the final report may only say the human was told something if the Gate 1 message says it. A controller annotated an answer with a consequence "the human accepted" that had never been shown to them.
- **One conductor skill (`build`) chains every phase**, and takes the current phase from a script that reads the files and the ledger rather than from the conversation. Upstream chains its skills by having each one name the next.
