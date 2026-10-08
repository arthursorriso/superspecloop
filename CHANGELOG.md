# Changelog

## 0.1.0

First public release.

- `/superspecloop:build` runs a feature from a request to a pull request: Spec Kit's specify, a spec review loop with Gate 1, Spec Kit's plan, a plan review loop, tasks and analyze, the execution loop, and the whole-branch review with the integration menu.
- Stops only at Gate 1, at the integration decision and at a phase's own stop conditions; resumes from the ledger in a new session.
- Every decision taken without asking is a `Ruling:` line in the ledger, reaches every implementer and reviewer, and is listed in the pull-request description.
- Each run leaves `.pipeline/<feature>/metrics.md` (counts, results and times only), and [EVALUATION.md](EVALUATION.md) has a protocol and scorecard for judging the pipeline on a real feature.
