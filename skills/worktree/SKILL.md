---
name: worktree
description: Use before the execution-loop skill when the human asks to run a Spec Kit feature in an isolated workspace (a git worktree), so their main checkout stays free while the loop runs. Optional - the loop works in place without it.
---

<!-- Derived from Superpowers' using-git-worktrees/SKILL.md (MIT, (c) 2025 Jesse Vincent). See UPSTREAM.md. -->

# Worktree

Give a Spec Kit feature its own checkout, so the execution loop can run for
as long as it needs while the human keeps working in theirs.

**This is opt-in.** Spec Kit already put the feature on its own branch, and
the loop runs fine in place. Use this only when the human asks for
isolation. Never create a worktree on your own initiative.

**Core principle:** detect existing isolation first, prefer the platform's
native tool, fall back to git, and never fight the harness.

## Step 0: Detect Existing Isolation

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" && pwd -P)
git rev-parse --show-superproject-working-tree   # prints a path inside a submodule
```

If `GIT_DIR != GIT_COMMON` and you are not in a submodule, you are already in
a linked worktree: do not create another. Skip to Step 3.

## Step 1: Preconditions

Stop and say which one failed:

- **The feature's documents are committed** on the feature branch:
  `git status --porcelain -- FEATURE` prints nothing and
  `git ls-files --error-unmatch FEATURE/tasks.md` succeeds. A worktree only
  contains committed files; uncommitted spec documents would not be there.
- **The rest of the working tree is clean.** Uncommitted changes elsewhere
  are the human's work in progress; ask what to do with them.

## Step 2: Create the Worktree

**Free the branch.** Git refuses to check out one branch in two places. If
the main checkout is on the feature branch, tell the human in one line that
their checkout is moving to `<base>`, then `git checkout <base>`. This is the
one visible change to their checkout, and it is what isolation means.

**Native tool first.** If your platform has a worktree tool (`EnterWorktree`,
a `/worktree` command, a `--worktree` flag) that can check out an existing
branch, use it and skip to Step 3. Creating one with git behind the
harness's back leaves state it cannot see or clean up.

**Git fallback.**

1. Location: a directory the human named, else an existing `.worktrees/` or
   `worktrees/` (`.worktrees/` wins), else `.worktrees/` at the repository
   root.
2. Make sure it is ignored, without touching any tracked file:
   ```bash
   mkdir -p .worktrees
   git check-ignore -q .worktrees/probe || printf '*\n' > .worktrees/.gitignore
   git status --porcelain -- .worktrees   # must print nothing
   ```
   An unignored worktree directory would commit a whole copy of the tree.
3. `git worktree add .worktrees/<branch> <branch>`

If `git worktree add` fails with a permission error, the sandbox blocked it:
say so, `git checkout <branch>` again, and run the loop in place.

## Step 3: Project Setup

From the worktree root, install dependencies with the project's own tool and
lockfile (`npm ci`, `pnpm install --frozen-lockfile`, `cargo build`,
`uv sync`, `go mod download`, ...). Skip it when the project has nothing to
install.

Spec Kit's per-checkout pointer (`.specify/feature.json`) does not exist in a
new worktree. The loop does not need it: it takes the feature directory as
its argument.

## Step 4: Hand Off

Report one line: `Worktree ready at <absolute path> on <branch>`.

That path is the **working directory** for everything that follows. The
execution loop runs its scripts from there and names it in every dispatch;
its workspace (`.pipeline/`) and its baseline test run live there too. The
baseline is the loop's Setup step — do not run the suite twice.

Removal is the finish-branch skill's job, and only for worktrees under
`.worktrees/` that this skill created.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "A long run deserves isolation — I'll set one up" | Opt-in. Ask nothing, create nothing, unless the human asked for it. |
| "I'm obviously not in a worktree" | Run Step 0. Harness-created isolation and submodules both fool eyeballing. |
| "`git worktree add` is quicker than finding a native tool" | A native tool owns placement and cleanup. Bypassing it creates state the harness cannot manage. |
| "The spec files are untracked, but they're right there" | They are in the main checkout, not in the worktree. Stop and have them committed. |
| "`.worktrees/` is surely ignored" | Check. One unignored worktree commits the whole tree into the repository. |
| "I'll add `.worktrees/` to the project's .gitignore" | That edits a tracked file on the human's base branch. The self-ignoring file needs no commit. |
