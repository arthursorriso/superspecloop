# Evaluating SuperSpecLoop on real work

This file is the context for a test run on a real codebase, in a session that
knows nothing about how the plugin was built. Read it before the run, and
again when filling in the scorecard at the end.

## What the plugin is for

The goal: hand a feature to an agent and get back a pull request that needs
**no review changes, or very few** — code that matches the spec, covers every
flow, has no errors and duplicates nothing.

The bet: reviews are cheaper the earlier they happen. So the plugin reviews
the spec against the request (Gate 1), the plan against the spec and the
existing code, each unit of work against its brief, and the whole branch
against the spec — with fresh reviewers each time and bounded fix loops.

The cost: many more agent calls and more wall-clock time than implementing
directly. The question this evaluation answers is whether the reviews earn
that cost, **stage by stage**.

## What the sandbox runs showed

Three small features on a toy Express API, no comparison against the plain
Spec Kit flow. Treat these as hypotheses to test, not results.

| Stage | Evidence so far | Status |
|---|---|---|
| Spec review + Gate 1 | Found problems in 2 of 2 features: 12 blocking issues, 6 decisions for the human | Looks useful |
| Implementer tests (test-first) | 12 of 12 deliberate bugs caught by the suites | Looks useful; may be the model, not the pipeline |
| Unit reviews | 16 reviews, 2 rejected a unit (3 findings: docs, an untested scenario, an empty test); 14 clean; missed one issue the final review caught | Catches things; cost-benefit unknown |
| Whole-branch review | 3 reviews, 2 asked for fixes (5 fixes, none a behavior bug) | Catches things; cost-benefit unknown |
| Plan review | 2 features, 11 decisions, up to 7 agent calls; no record of preventing a duplicate | Unproven |
| Analyze | 1 run, 0 critical findings | Unproven |
| Overall goal | No PR was ever reviewed by a person | Unproven |

In the sandbox **no review caught a behavior bug** — and no behavior bug
escaped either. A toy project did not make the implementer fail. A real
codebase, with conventions, existing helpers and real business rules, is the
test that matters.

## Protocol

### Before the run

1. **Pick the feature.** Real, medium-sized (two to five user stories), with
   business rules, headed for a normal PR that colleagues will review. Not a
   bug fix, not a one-file change, not a migration.
2. **Write down the baseline** in the scorecard: from your last five or so
   PRs of similar size, the average number of review comments that asked for
   a change, and how many review rounds they took. If you can, note how long
   a feature like this usually takes you.
3. **Set the decision rules** in the scorecard now, before seeing results.

### During the run

4. Run everything in **one Claude Code session**:
   `/superspecloop:build <the request, as you would describe it to a colleague>`.
5. Intervene only at the two stops (Gate 1 and the integration menu). Note
   the minutes you spend at each, including reading.
6. If anything goes wrong in the pipeline itself (a phase skipped, a stop
   that should not have happened, a decision you disagree with), note it.

### At the end of the run

7. Before closing or clearing the session, run `/usage` and paste its
   Session block at the end of `.pipeline/<feature>/metrics.md` (the finish
   step writes that file). On a Team or Enterprise plan, also note the share
   `/usage` attributes to subagents and to this plugin. `metrics.md` holds
   only counts, PASS/FAIL results and times — no names and no text from the
   run — so it is safe to take out as it is.
8. Choose **option 1** (open a pull request). Opening a PR keeps the
   workspace with every review file.
9. **Review the PR yourself first**, as if a colleague wrote it. List every
   change you would ask for. Then let your colleagues review it normally.

### After the PR is merged or closed

10. Fill in the scorecard below, in `.pipeline/<feature>/scorecard.md` of
    the work repository. That directory is ignored by git, so the file can
    never land in a commit. **Never save it in the plugin's folder**: that
    folder is a clone of a personal repository, and a commit there would
    push it out.
11. Before taking anything out of work, go through the checklist in
    [Taking results out](#taking-results-out). Only `metrics.md` and the
    scorecard leave; nothing else from `.pipeline/` does.

## Metrics

| Metric | Source | Why it matters |
|---|---|---|
| Change requests on the PR (yours + colleagues') | The PR | The goal itself |
| Review rounds until approval | The PR | Same, measured in round trips |
| Behavior bugs found after merge (QA, production) within two weeks | Your notes | What every review missed |
| Findings per stage, and how many were real | Review files in `.pipeline/<feature>/` | Which stages earn their cost |
| PR comments a stage should have caught | You, per comment | Which stage failed |
| Your minutes at Gate 1 and at integration | Your notes | Human cost |
| Wall-clock time, time per phase | `metrics.md` (Timeline) | Waiting cost |
| Tokens and estimated cost | `/usage` Session block | Machine cost |
| Agent calls per stage | `metrics.md` (Counts) | Where the cost goes |
| Pipeline defects | Your notes | Whether the tool is ready |

A finding is **real** if you would have asked for that change in a PR
review. It is **noise** if it is true but you would not have cared. It is
**wrong** if the reviewer misread the code or the spec.

## Taking results out

The scorecard is built from categories and numbers so that it says how the
pipeline did without saying anything about the work. Before it leaves:

- [ ] Every cell is a number, a category from the list below, or a stage name.
- [ ] No code, file or directory names, endpoints, table or field names.
- [ ] No business rules, product names, customer names, people or teams.
- [ ] No links (PR, ticket, repository) and no branch or commit ids.
- [ ] Anything that does not fit a category is `other`, with no explanation.
- [ ] Your company's rules allow taking these numbers out.

`metrics.md` passes this checklist by construction. The review files, the
ledger, the briefs, `rulings.md` and the PR description do not: they never
leave.

**Finding categories:** missing requirement · untested case · wrong
behavior · wrong value · duplication · convention · error handling ·
performance · security · docs · readability · other.

**Feature kinds:** new endpoint · change to an endpoint · UI flow ·
background job · integration with another system · data change · other.

## Scorecard

Fill it in at `.pipeline/<feature>/scorecard.md`. A Claude session at work
can do most of it with this prompt:

> Read `EVALUATION.md` in the superspecloop plugin folder,
> `.pipeline/<feature>/metrics.md`, the review files in `.pipeline/<feature>/`
> and the comments on the pull request. Fill in the scorecard of
> EVALUATION.md in `.pipeline/<feature>/scorecard.md`: classify each finding
> of the reviews as real, noise or wrong, and each PR comment by the stage
> that should have caught it, using only the categories EVALUATION.md
> lists. Then go through the "Taking results out" checklist against the
> file and report any cell that fails it.

```markdown
# SuperSpecLoop scorecard — run <n>, <date>

## Decision rules (written before the run)
- The full pipeline is worth it if: <e.g. change requests ≤ half the baseline, and my time ≤ 30 min>
- A stage is dropped if: <e.g. zero real findings in two runs>
- The run failed if: <e.g. the PR needed more changes than the baseline>

## Feature
- Size: <user stories> user stories, <files> files, <lines> lines changed
- Kind: <feature kind from the list>
- Codebase: <language> / <framework>, ~<n>k lines, test suite: <yes/no>

## Baseline (last ~5 comparable PRs)
- Change requests per PR: <avg>
- Review rounds: <avg>
- Typical time to build a feature like this: <hours/days>

## This run
- Change requests on the PR: <n> (mine: <n>, colleagues: <n>)
- Review rounds: <n>
- Behavior bugs found after merge: <n>
- My minutes at Gate 1: <n> · at integration: <n> · reviewing the PR: <n>
- Wall-clock time of the build: <h:mm> (longest gap: <min>, between <pipeline file> and <pipeline file>)
- Tokens / estimated cost: <from /usage> · subagent share: <%>

## Findings per stage
| Stage | Agent calls | Findings | Real | Noise | Wrong | Real ones, by category |
|---|---|---|---|---|---|---|
| Spec review + Gate 1 | | | | | | |
| Plan review | | | | | | |
| Analyze | | | | | | |
| Unit reviews | | | | | | |
| Whole-branch review | | | | | | |

## PR comments, by the stage that should have caught them
| Category | Severity | Should have been caught by | Why it was not |
|---|---|---|---|
| | critical / important / minor | spec / plan / unit / final / none (taste) | not in the spec / unwritten team convention / reviewer missed it / reviewer rated it minor / outside the diff / other |

## Pipeline defects
- <phase skipped / wrong stop / bad ruling / crash / other>: <count>

## Verdict
- Against the decision rules: <met / not met, per rule>
- Keep: <stages> · Drop or slim: <stages> · Change: <what>
```

## How to read the results

- **Few PR changes and the reviews found real things** → the pipeline is
  doing its job; look at cost per stage to slim it.
- **Few PR changes but the reviews found almost nothing** → the model does
  the work; the reviews are insurance. Keep Gate 1 and the final review,
  test a lighter flow.
- **Many PR changes** → look at which stage should have caught them. If it
  is always "none (taste)", the spec and conventions need to carry more of
  your team's preferences; if it is a stage, that stage needs fixing.
- **One run is one data point.** Two or three features before dropping or
  keeping a stage for good.
