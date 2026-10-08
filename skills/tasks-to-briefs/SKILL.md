---
name: tasks-to-briefs
description: Use to convert a Spec Kit feature (specs/<feature>/ with spec.md, plan.md and tasks.md) into execution units for the SuperSpecLoop execution loop. Normally dispatched by execution-loop in a fresh subagent; run it by hand only to preview how a feature will be split.
---

# Tasks to Briefs

Turn a Spec Kit feature into `units.md`: an ordered list of execution units,
each written so an implementer who has never seen the spec can build it
correctly, and a reviewer can judge it from one diff.

**Why this exists:** Spec Kit tasks are one-line pointers ("T014 Implement
UserService in src/services/user.py"). An implementer working from a pointer
guesses the requirements, and a reviewer comparing a diff against a pointer
cannot tell whether the spec was met. A unit carries the requirements
verbatim, so both can work from one file.

**You are not the planner.** The spec decides *what*, the plan decides *how*,
tasks.md decides the order of work. You regroup and restate — you never add
scope, drop scope, or resolve an ambiguity by choosing. What you cannot
place, you report as a gap.

## Inputs

The feature directory, e.g. `specs/001-login/`. Resolve the workspace first:

```bash
bash "<this skill's base directory>/../execution-loop/scripts/pipeline-workspace" specs/001-login
```

It prints the directory where `units.md` goes. Then read, once each:

- `spec.md` — required. The binding authority.
- `plan.md` — required. Technical Context, structure, constraints.
- `tasks.md` — required. The work items (`- [ ] T012 [P] [US1] ...`).
- `data-model.md`, `contracts/`, `research.md`, `quickstart.md` — if present.
- `.specify/memory/constitution.md` — if present; its MUST principles bind every unit.

## Preconditions — stop and report if any fails

- tasks.md has at least one unchecked task (`- [ ]`). Checked tasks
  (`- [X]`) are done: exclude them from units, list them as "already done".
- spec.md contains no `[NEEDS CLARIFICATION` marker. An open clarification is
  a question for the human (it belongs to the spec gate), not something a
  unit can carry. Report each marker with its line.

## Grouping rules

A unit is **one review surface**: a coherent change a reviewer can judge in
one diff. Group Spec Kit tasks into units with these rules, in order:

1. **Never cross a phase.** Setup, Foundational, each User Story phase and
   Polish are separate. A unit belongs to exactly one phase. One exception:
   a Setup phase whose tasks produce nothing a reviewer can judge on its own
   (creating directories, empty placeholder files) merges into the first
   unit that uses what it creates. A unit costs an implementer and a
   reviewer; two empty folders do not earn one.
2. **Tests travel with the code they verify.** A story's test tasks go into
   the same unit as the implementation tasks those tests cover. The
   implementer writes the failing tests first (TDD), then the code.
3. **One unit per phase by default.** A user-story phase — its tests and its
   implementation — is one unit. Split a phase only when it would exceed
   roughly 6 tasks or 400 changed lines, and then at the natural seams
   (model → service → endpoint). Every unit costs an implementer and a
   reviewer; splitting a small story in two doubles that and creates
   ordering problems between the halves.
4. **Batch same-shape trivia.** Several small config/setup tasks of the same
   kind become one unit.
5. **Respect declared dependencies** ("depends on T012") and the phase order.
   Order units so every dependency comes first. `[P]` markers do not matter:
   execution is sequential.
6. **Vague tasks get anchored, not expanded.** "Add validation and error
   handling" attaches to the unit whose code it modifies, and its
   requirements are the spec's FRs and edge cases that concern that code —
   named explicitly. If you cannot tell which requirements it means, report
   it as a gap instead of guessing.
7. **Every open task ID lands in exactly one unit.**

## Output: units.md

Write `<workspace>/units.md` with this structure. Copy exact values —
names, numbers, messages, formats, signatures, status codes — **verbatim**
from the source documents. Paraphrasing an exact value is a defect.

````markdown
# Units — feature: specs/001-login

**Spec:** specs/001-login/spec.md
**Plan:** specs/001-login/plan.md
**Already done:** T001, T002 (or: none)

## Global Constraints

Binding requirements that apply to every unit, copied verbatim:
- From plan.md Technical Context: language/version, framework, storage,
  testing tool, performance goals, constraints.
- From the constitution: each MUST principle, one line each.
- From spec.md: cross-cutting requirements (FRs that every unit must respect).

## Verification Commands

The exact commands for the focused test, full suite, typecheck and lint, as
the plan, quickstart or the repository's own config define them (read
package.json / pyproject.toml / Makefile if the plan does not say). Write
"not defined" rather than inventing one.

### Unit 1: <short imperative title>

**Covers tasks:** T010, T012, T013
**Phase:** Phase 3 — User Story 1 (P1)
**Depends on units:** none | 1, 2

#### Goal
One or two sentences: what exists after this unit that did not before.

#### Requirements
Verbatim from spec.md, each with its identifier:
- FR-003: ...
- US1 scenario 2: Given ..., When ..., Then ...
- Edge case: ...

#### Files
- Create: `src/services/login.py` — one-line responsibility
- Modify: `src/api/routes.py` — what changes
- Test: `tests/integration/test_login.py`

#### Interfaces
Exact signatures, entity fields, endpoint shapes and error formats from
plan.md, data-model.md and contracts/ that this unit creates or consumes.
Include what earlier units produce that this one calls.

#### Tests required
Each requirement above maps to at least one behavior to test:
- FR-003 → rejects a login with wrong password, returning <exact error>
- US1 scenario 2 → ...

#### Out of scope
What belongs to other units (by unit number), so the implementer does not
build it early.

## Coverage

| Spec item | Unit(s) |
|---|---|
| FR-001 | 1 |
| US1 scenario 1 | 2 |

| Task | Unit |
|---|---|
| T010 | 1 |

## Gaps

Spec items no task covers, tasks you could not anchor to requirements,
contradictions between spec, plan and tasks. One line each, citing the
source lines. "None" if none.
````

Unit headings must be exactly `### Unit N: <title>` with N counting from 1 —
the `task-brief` script extracts a unit by that heading. Use `####` for the
subsections inside a unit, and never put a `#` or `##` heading inside one.

## Self-check before reporting

Write down each check as you do it:

- Every unchecked task ID appears in exactly one unit's "Covers tasks", and
  in the task coverage table.
- Every FR, user-story acceptance scenario and edge case in spec.md appears
  in some unit's Requirements, or in Gaps.
- Every requirement in a unit maps to at least one "Tests required" line.
- No unit crosses a phase, except a trivial Setup merged into its first
  user. Dependencies point only to earlier units.
- No unit needs something only a later unit builds — including in its
  tests: a test that needs data in a certain state needs a way to reach that
  state by this unit. If the task order makes that impossible, report it
  under Gaps.
- Exact values match their source character for character.

## Report

Do not modify spec.md, plan.md or tasks.md. Reply with only:

- The path of `units.md`
- Number of units, and task IDs already done
- Gaps, one line each (or "none")
- Anything in the preconditions that failed
