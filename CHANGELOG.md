# Changelog

## 0.2.0

Redesign after the first run on a production codebase, where the reviews found most problems before the pull request but the controller was the most expensive part of the run and, three times, wrote down something that did not happen.

- **Each unit runs in its own context.** A new `unit-runner` agent takes one unit from implementation to close and replies with one line, so the session's context — and the cost of each of its steps — no longer grows with every unit.
- **Scripts keep the record.** `unit-start`, `unit-check` and `unit-close` replace about fifteen manual steps per unit. A unit closes only when its last check passed on HEAD and, in full mode, its review is clean; `baseline` saves the verification commands once for every later gate run.
- **Tasks are ticked only when performed.** The implementer reports every task as `done — <evidence>` or `not done — <reason>`; only the first kind is marked in `tasks.md`, and the rest reach the pull request as open items.
- **Departures from the spec always reach you.** Spec gaps found by the plan review become questions that hold the pipeline until you answer; spec items the whole-branch review finds unmet are fixed or put to you, and lead the pull-request description. No ruling can settle either.
- **Lean mode by default:** no per-unit review — in the measured runs it found the fewest real problems per call — unless the request asks for unit reviews.
- **Fewer units:** one per user story, with setup and polish merged in.
- **Spec Kit's commands and the plan review run in their own context**, dispatched by the conductor. The session no longer carries the plan, the tasks and the analysis to the end of the build.
- **Analyze no longer stops the build to ask.** CRITICAL and HIGH findings that the plan or the tasks can absorb are applied in one remediation round; only a finding that needs the spec to change stops for you.
- **The pull request's coverage table is current.** Rows the fix wave addressed show the re-review's status, not the status from before the fixes.
- **Wrapped task lines reach the record whole**, and the pull request lists the whole-branch review's minor issues under known issues, since the fix wave does not touch them.

## 0.1.0

First public release.

- `/superspecloop:build` runs a feature from a request to a pull request: Spec Kit's specify, a spec review loop with Gate 1, Spec Kit's plan, a plan review loop, tasks and analyze, the execution loop, and the whole-branch review with the integration menu.
- Stops only at Gate 1, at the integration decision and at a phase's own stop conditions; resumes from the ledger in a new session.
- Every decision taken without asking is a `Ruling:` line in the ledger, reaches every implementer and reviewer, and is listed in the pull-request description.
- Each run leaves `.pipeline/<feature>/metrics.md` (counts, results and times only), and [EVALUATION.md](EVALUATION.md) has a protocol and scorecard for judging the pipeline on a real feature.
