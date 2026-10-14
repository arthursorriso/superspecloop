# SuperSpecLoop

An agentic pipeline for Claude Code built around [GitHub Spec Kit](https://github.com/github/spec-kit): the spec is reviewed against your request, the plan against the spec and the existing code, and the implementation unit by unit and then as a whole branch, all the way to the pull request.

> **Status: 0.2.0, early.** The whole pipeline runs end to end, from a request to a pull request. It has been run on four features of a sandbox API and one feature of a production codebase; 0.2 is the redesign that run called for (see [CHANGELOG.md](CHANGELOG.md)). [EVALUATION.md](EVALUATION.md) is how to judge it on yours. Not built: a separate test-writing agent (implementers write their own tests, test-first).

## Install

```
/plugin marketplace add arthursorriso/superspecloop
/plugin install superspecloop@superspecloop
```

Or from a shell: `claude plugin marketplace add arthursorriso/superspecloop` and `claude plugin install superspecloop@superspecloop`.

**Requirements**

- [GitHub Spec Kit](https://github.com/github/spec-kit) initialized in the project (`specify init`), with its Claude Code commands installed.
- A recent Claude Code: units run in subagents that dispatch subagents of their own.
- A git repository. Features are built on their own branch.
- A test command, and ideally a typecheck and a lint command: every unit is gated on them. The plan names them; the pipeline runs whatever it names.
- For the "open a pull request" option: a git remote and the GitHub CLI (`gh`).

**Cost.** The pipeline runs many subagents. Implementers and unit runners use the mid-tier model; the spec, plan and whole-branch reviews use the most capable one, because they find most of the real problems. Expect noticeably more of your plan or API budget than implementing directly. Two settings keep it down:

- Run the session itself on the mid-tier model (`/model sonnet`): the session only conducts.
- Compact earlier (`/autocompact 200k`): the pipeline keeps everything it needs on disk and resumes after a compaction.

## Usage

In a project with Spec Kit initialized:

```
/superspecloop:build Let users archive tasks and restore them later
```

That one command runs every phase and stops twice:

1. **Gate 1.** After the spec is written and reviewed against your request, you get the questions only you can answer and a table tracing each thing you asked for to a requirement and a scenario. You answer, then approve.
2. **Integration.** After the branch is implemented and reviewed, you get the pull-request description and choose: open a PR, merge locally, or keep the branch.

It also stops whenever only you can decide: when the plan review finds that the spec leaves something open, asks for something that cannot be built as written, or would have to be departed from. The spec is yours; no agent rules on it.

By default units are not reviewed one by one — the whole-branch review covers them. Add "with unit reviews" to the request for a review after every unit.

To continue a feature in a new session: `/superspecloop:build specs/003-archive`.

## What runs

| Phase | What happens |
|---|---|
| specify | Spec Kit writes `spec.md`; your request is frozen in `request.md` |
| spec review | A reviewer with clean context checks the spec against the request: missing items, unrequested scope, untestable criteria, ambiguity. An editor fixes what the request settles; everything else becomes a question. Stops at **Gate 1**. |
| plan | Spec Kit writes the plan |
| plan review | A reviewer checks the plan against the spec and against the existing code: uncovered requirements, reinvented helpers, broken conventions, over-engineering |
| tasks, analyze | Spec Kit generates and cross-checks the tasks; CRITICAL and HIGH findings the plan or tasks can absorb are fixed in one round, and one that needs the spec changed stops for you |
| execute | Tasks are grouped into units, one per user story. Each unit runs in its own context: a fresh implementer builds it test-first, scripts check that the suite, typecheck and lint pass and the spec is untouched, an optional unit review sends findings back for up to 5 fix rounds, and a script closes the unit — marking only the tasks the implementer actually performed. |
| finish | A whole-branch review checks every requirement, scenario and edge case against code and tests. Every spec item not met is fixed or put to you; one wave of fixes. Then the PR description and the **integration** choice. |

Everything the agents decided without asking is recorded as a ruling and listed in the final report and in the pull-request description.

Each phase is also a skill you can run on its own after the matching Spec Kit command: `spec-review-loop`, `plan-review-loop`, `execution-loop`, `finish-branch`. To run the execution in a git worktree and keep your checkout free, say so in the request.

## Structure

```
skills/     one per phase, plus build (the conductor); scripts live inside the skill that uses them
agents/     isolated roles: spec-reviewer, plan-reviewer, doc-editor, unit-runner, implementer, task-reviewer, re-reviewer, branch-reviewer
```

Run artifacts live in `.pipeline/<feature>/` (ignored by Git): the ledger, reviews, briefs, reports. Spec, plan and tasks stay in `specs/<feature>/`, managed by Spec Kit, together with `request.md` and `gate-1.md`.

## Developing the plugin

Clone the repository and add the clone as a marketplace: Claude Code then reads the plugin straight from your working copy, and edits apply at the next session or after `/reload-plugins`.

```
/plugin marketplace add /path/to/your/clone
/plugin install superspecloop@superspecloop
```

Check the manifests with `claude plugin validate .` before committing.

## Evaluating it

Every run leaves `.pipeline/<feature>/metrics.md`: what each phase found and
how long it took. [EVALUATION.md](EVALUATION.md) has the protocol and the
scorecard for judging the pipeline on a real feature, stage by stage.

## Credits

The execution engine is based on [Superpowers](https://github.com/obra/superpowers), by Jesse Vincent / Prime Radiant (MIT). SuperSpecLoop is not affiliated with or endorsed by the original project. See [UPSTREAM.md](UPSTREAM.md) for where each derived file came from.

## License

MIT. See [LICENSE](LICENSE).
