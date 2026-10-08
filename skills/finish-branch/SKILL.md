---
name: finish-branch
description: Use when every unit of a Spec Kit feature has been executed by the execution-loop skill — runs the whole-branch review against the spec, one wave of fixes, builds the pull-request description, and asks the human how to integrate the branch.
---

<!-- Derived from Superpowers' finishing-a-development-branch/SKILL.md and the Final Review section of subagent-driven-development/SKILL.md (MIT, (c) 2025 Jesse Vincent). See UPSTREAM.md. -->

# Finish Branch

Take a feature whose units are all complete to a branch a human can merge
with little or nothing to add: one whole-branch review against the spec, one
wave of fixes, a pull-request description built from the run's own records,
and the integration decision handed to the human.

**You are still the controller.** You coordinate; subagents review and fix.
The rules of the execution loop still hold: artifacts move as files, every
decision is a `Ruling:` line in the ledger, and you never fix code yourself.

**Core principle:** verify → review the whole branch → fix once → describe →
ask → execute → clean up.

## Paths

`FEATURE` is the feature directory (e.g. `specs/001-login`), `WS` the
workspace `pipeline-workspace FEATURE` prints. Loop scripts are in the
sibling skill directory `../execution-loop/scripts/`; this skill's own
script is `scripts/pr-body`. Invoke all of them with `bash`.

Subagent types: `superspecloop:branch-reviewer`, `superspecloop:implementer`,
`superspecloop:re-reviewer`.

## Step 1: Verify

All four must hold. If one fails, say which and stop — do not review a
branch that is not finished.

1. `WS/progress.md` exists, its first line names FEATURE, and every unit in
   `WS/units.md` has a `Unit <N>: complete` line.
2. `FEATURE/tasks.md` has no unchecked task (`- [ ]`).
3. The working tree is clean.
4. **Gates on this exact tree:**
   `bash ../execution-loop/scripts/run-gates FEATURE final "<full suite>" "<typecheck>" "<lint>"`
   with the Verification Commands in `WS/global.md`. A green run earlier in the
   session proves nothing about the tree you are about to integrate.

If the ledger already has a `Final review:` line, the review was done:
resume at the first step below that has no ledger line.

## Step 2: Find the Range and the Base

Two different things, and on a stacked branch they differ:

- **`FEATURE_BASE` — where this feature's work starts.** The sha on the
  ledger's `Start:` line. The whole-branch review and the pull-request
  description cover `FEATURE_BASE..HEAD` and nothing before it. A ledger
  without a `Start:` line predates it: use the merge-base below instead.
- **The base branch — where the work will be integrated.** Take it from the
  conversation if stated. Otherwise the default branch (`origin`'s HEAD, or
  whichever of `main` and `master` exists).

Compare them: `git merge-base <base branch> HEAD`.

- **Equal to `FEATURE_BASE`:** the ordinary case. Proceed.
- **Different:** this branch sits on top of commits that are not on the base
  branch and are not part of this feature — another feature not merged yet.
  Find where they come from (`git branch --points-at FEATURE_BASE`,
  `git branch --contains FEATURE_BASE`). The review range stays
  `FEATURE_BASE..HEAD`. The integration target is the human's call: in Step
  6, say that the branch is stacked on `<other branch>` and offer that
  branch as the pull-request and merge target, with the default branch as
  the alternative that would carry the other feature's commits too.

Append `Finish: range <FEATURE_BASE 7>..<HEAD 7>, base <base branch>` (and
`, stacked on <other branch>` when it is) to the ledger.

## Step 3: Whole-Branch Review

`bash ../execution-loop/scripts/review-package FEATURE FEATURE_BASE HEAD`
prints the package path.

Dispatch `superspecloop:branch-reviewer` on the most capable model available
(`opus`) with:

- the working directory
- the paths of `FEATURE/spec.md`, `FEATURE/plan.md` and any of
  `data-model.md`, `contracts/`, `research.md`, the constitution
- the diff path with FEATURE_BASE and HEAD
- `WS/final-gates.txt`
- `WS/progress.md`
- the review file `WS/final-review.md`

Do not summarize the feature for it, and never pre-judge: no "do not flag",
no "the units already covered X". It reads the spec and decides.

Append to the ledger:
`Final review: <Yes | With fixes | No> (<C> critical, <I> important, <F> to fix before merge, <W> can wait)`

**Declined to Judge.** Rule on each line the reviewer set aside — in scope
(it joins the fix list) or out of scope (say why) — as a `Ruling:` line.
Nothing set aside is dropped silently. Then run
`bash ../execution-loop/scripts/rulings FEATURE`, so `WS/rulings.md` has
these too before anyone is dispatched.

## Step 4: One Fix Wave

The fix list is every Critical and Important issue, every Fix Before Merge
item, and anything you ruled in scope. If it is empty, skip to Step 5.

A finding that conflicts with the spec or the plan is yours to rule on
first, with the spec as authority; ledger the ruling and leave the finding
in or out accordingly.

Dispatch **one** `superspecloop:implementer` (`sonnet`, or `opus` when the
list includes a design-level issue) with the whole list — not one fixer per
finding: each would rebuild context and re-run the suite. Its dispatch:

- the working directory
- "Your brief is the fix list in `WS/final-review.md`: every Critical and
  Important issue and every Fix Before Merge item" plus your in-scope
  rulings, verbatim
- the spec path, as the authority behind the list
- `WS/global.md` (global constraints and verification commands)
- `WS/rulings.md` (decisions already made; a fix must not undo one)
- "Fix each finding test-first where a test can express it. Change nothing
  the list does not ask for."
- the report file `WS/final-fix-report.md`

Record `FIX_BASE=$(git rev-parse HEAD)` before dispatching. When it reports
DONE:

1. `git diff --stat FIX_BASE HEAD -- FEATURE .specify` is empty.
2. `bash ../execution-loop/scripts/run-gates FEATURE final "<suite>" "<typecheck>" "<lint>"`
   — red gates go back to the same implementer once.
3. `bash ../execution-loop/scripts/review-package FEATURE FIX_BASE HEAD`, then
   dispatch `superspecloop:re-reviewer` with: the spec path as the brief, the
   fix list verbatim, the report file, the diff path, `WS/final-gates.txt`,
   `WS/rulings.md`, and the review file `WS/final-review-round-1.md`.

**There is no second wave.** Adjudicate whatever the re-review leaves open,
exactly as the loop's breaker does:

- reviewer wrong or contestable → `Final: parked — <finding> — Ruling: <why the code stands>`
- real but harmless to ship → park it the same way, ruling that it is real
- real and it makes the branch wrong against the spec → do not park it.
  Report it to the human in Step 6 as an open defect; the branch is not
  ready, and saying so is the correct result.

Append: `Final fix: (<X> addressed, <Y> open; commits <a7>..<b7>)`

## Step 5: Build the Pull-Request Description

Write `WS/pr-summary.md` — the only part you compose:

```markdown
# <type>(<scope>): <what the feature does, imperative, under 70 characters>

<Two to five sentences: what a user can do now that they could not before,
and anything a reviewer should look at first. No process narration.>

Spec: `FEATURE/spec.md`
```

Then `bash scripts/pr-body FEATURE FEATURE_BASE`. It writes the title to
`WS/pr-title.txt` and assembles `WS/pr-body.md` from the rest of the summary, the reviewer's Spec Coverage table, the
final gate run, every `Ruling:` line in the ledger, the Can Wait and parked
findings, and the review trail. Do not retype any of those: the script is
what guarantees no ruling is left out.

It prints how many rulings it found. If the body is missing something, the
source is wrong, not the body: fix the ledger line or the review file and run
the script again. Never edit `pr-body.md` by hand — the next run overwrites
it, and a hand-edited body is one nobody can regenerate.

If the repository has a pull-request template
(`.github/pull_request_template.md` or similar), fit the same content into
its sections instead of replacing it.

Then `bash scripts/run-metrics FEATURE`. It writes `WS/metrics.md`: what
each phase did, from the ledger and the review files, and when each file
was written. It is how a run is judged afterwards, so it is written before
anything is integrated.

## Step 6: Report and Ask

Your message to the human, in this order:

1. One line: what was built, units, commit range, final review verdict.
2. **Open defects**, if any survived Step 4 — first, not buried.
3. **Rulings I made** — every `Ruling:` line in the ledger, each with what
   it costs if wrong. Exhaustive, but sorted for a reader with five
   minutes: first the ones that could cost something (behavior a user
   would see, a test changed or weakened, scope dropped, anything called
   out of scope), in full; then the ones whose cost is none, one short line
   each under "Routine". Sixteen rulings in one flat list hide the two that
   matter.
4. Known issues not fixed (Can Wait and parked), and every `flake` line in
   the ledger.
5. The pull-request title and the path of `WS/pr-body.md`.
6. The path of `WS/metrics.md`, with one line: the Session block of
   `/usage`, run now in this session, is the token count; paste it at the
   end of that file.

**Say only what the record says.** The human's part in this run is their
request, their answers at Gate 1 and their approval — nothing else. Never
write that they were told, warned of or accepted something unless the words
are in the Gate 1 message or in their reply: "you chose B" is a fact, "you
chose B after I flagged the cost" is one only if the flag is on the page.
A consequence of their answer that nobody showed them is reported as what
it is — something they have not seen yet.

Then present exactly these options and wait. Integration is the human's
decision; pushing, opening a pull request and merging are side effects you
never take on your own.

```
What would you like to do with <branch>?

1. Push and open a pull request against <base>
2. Merge into <base> locally
3. Keep the branch as it is
```

From a detached HEAD, offer only 1 (pushing as a new branch) and 3.

Discarding the work is not on the menu. It happens only when the human asks
for it in so many words, and then only after they type `discard` in reply to
a list of exactly what will be deleted (branch, commits, worktree).

## Step 7: Execute

**1. Push and open a pull request.**
`git push -u origin <branch>`. If the push is rejected, the remote moved:
report it and stop — never force-push on your own. Then create the pull
request against `<base>` with the forge's CLI, the title from
`WS/pr-title.txt` and the body from `WS/pr-body.md`, e.g.
`gh pr create --base <base> --title "$(cat WS/pr-title.txt)" --body-file WS/pr-body.md`.
With no CLI, give the human the creation URL the push printed and the two
file paths. Report the pull-request URL. Append `Finish: PR <url>` to the ledger.
Keep the worktree and the workspace: review feedback gets fixed here.

**2. Merge locally.** From the main checkout: `git checkout <base>`,
`git pull` if it has an upstream, `git merge <branch>`, then run the full
suite on the merged result. If it fails, stop and leave everything in
place — nothing was pushed. When green: clean up (Step 8), then
`git branch -d <branch>`. Append `Finish: merged into <base>`.

**3. Keep.** Report the branch name and where it lives. Append
`Finish: kept`.

## Step 8: Clean Up

Runs for a local merge and a confirmed discard only.

- **Worktree:** remove it only if this pipeline created it — its path is
  under `.worktrees/` of the main checkout. First copy the worktree's
  `WS/metrics.md` to the same path under the main checkout's `.pipeline/`,
  since the workspace goes with the worktree. Then, from the main checkout:
  `git worktree remove <path>` then `git worktree prune`. If removal is
  refused because of uncommitted or untracked files, never `--force`: show
  the human `git -C <path> status --porcelain -uall` and ask whether to
  commit them, move them, or delete them. Any other worktree belongs to the
  host; leave it.
- **Workspace:** `WS` is inside the checkout it was created in and goes
  with it. In a normal checkout, delete everything in `WS` after a merge
  except `metrics.md`; the git history is the record now. Never touch
  another feature's directory under `.pipeline/`.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Every unit was reviewed, the final review is a formality" | Unit reviews cannot see duplication across units, a requirement no unit owned, or twenty deferred minors adding up. This review is the only one that reads the spec end to end. |
| "The gates passed at the end of the loop" | Run them on the tree you are about to integrate. |
| "I'll dispatch a fixer per finding, in parallel" | One fixer, the whole list. Per-finding fixers each rebuild context and re-run the suite, and their commits conflict. |
| "The re-review left one thing open — one more wave" | There is no second wave. Park it with a ruling, or report an open defect. |
| "I'll write the coverage table into the PR myself" | `pr-body` copies the reviewer's table. A table you retype is a table you can get wrong. |
| "This ruling is too small to list" | The list is exhaustive. A ruling the human never sees was a decision made in secret. |
| "They picked that option, so they accepted what follows from it" | They accepted what they were shown. If the consequence was not in the Gate 1 message, tell them now, as news. |
| "They obviously want a pull request" | Present the three options and wait. |
| "The push was rejected — force-push will fix it" | The remote moved. Report and stop. |
| "Removal refused — `--force` finishes the cleanup" | The refusal means files exist only there. Ask. |
