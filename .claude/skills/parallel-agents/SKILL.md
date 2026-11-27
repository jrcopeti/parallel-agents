---
name: parallel-agents
description: Builds several independent features at once — one Git worktree and one subagent per feature — then reports a consolidated summary with the branch and worktree for each. Use when asked to implement multiple unrelated features in parallel.
argument-hint: <feature-1> <feature-2> ...
disable-model-invocation: true
allowed-tools:
  - Bash(git rev-parse:*)
  - Bash(git worktree:*)
  - Bash(git branch:*)
  - Bash(git status:*)
---

# Parallel feature development

Build each requested feature in its own Git worktree, with one subagent per worktree, then
consolidate the results.

Requested features: $ARGUMENTS

## Repository context

- Repo root: !`git rev-parse --show-toplevel 2>&1`
- Current branch: !`git rev-parse --abbrev-ref HEAD 2>&1`
- Existing worktrees: !`git worktree list 2>&1`

## Definitions

- `REPO_ROOT` — the repo root path above. `REPO` — its basename.
- `FEATURE` — one whitespace-separated token from the requested features. A quoted
  multi-word token is a feature brief: pass the full text to the subagent as its
  description, and take the slug from its leading words only — at most three, dropping
  filler — so `"about page: shows the version"` becomes `about-page`, not a slugified
  sentence. Branch and directory names come from the slug.
- `PARENT` — the directory containing `REPO_ROOT`, i.e. the repo's sibling level. Resolve
  it from `REPO_ROOT`, never from the current working directory, which may be a
  subdirectory of the repo.
- Each feature gets worktree `PARENT/REPO-FEATURE` on branch `feature/FEATURE`.
- `SHARED FILES` — the files every feature must edit to announce itself to the app: nav
  and menu definitions, route tables and registries, barrel or index exports, DI and
  plugin registration, i18n catalogs, migration ordering. Worktrees isolate agents from
  each other's edits, not from these. Left alone, each agent edits them as if it were the
  only feature, and the branches collide at merge.

## Step 1 — Parse and confirm

State the parse back in one line before touching anything:

```
REPO = expense-tracker; FEATURES = dark-mode, csv-export, budget-alerts
```

Stop and ask instead of guessing when:

- No features were given.
- The repo context above shows an error rather than a path — the working directory is not
  a Git repository.
- A target worktree directory or `feature/FEATURE` branch already exists. Ask whether to
  reuse, rename, or skip it.

Warn, but continue, if the repo has uncommitted changes: worktrees branch from `HEAD`, so
uncommitted work does not carry into them.

Then identify the `SHARED FILES` for this repo, before spawning anything. Look at how an
existing comparable feature is wired up — for a new page, find the nav or route definition
that lists the current pages; for a new module, find the barrel export or registry that
lists its siblings. List the paths you found. Every agent will be told to leave these
files untouched, and you apply their edits yourself in Step 6.

## Step 2 — Create worktrees

For each feature, run from `REPO_ROOT` with the absolute target path:

```bash
git worktree add -b feature/FEATURE PARENT/REPO-FEATURE
```

`PARENT/REPO-FEATURE` is a full path, not a `../` relative one, so the worktrees land
beside the repo whatever the working directory is. Then `git worktree list` to confirm all
of them landed as siblings of the repo, not inside it.

## Step 3 — Mirror the dev environment

Each worktree is a fresh checkout, so gitignored files do not come along. Mirror whatever
the main repo actually uses — check `REPO_ROOT` before copying:

- Install dependencies with the repo's package manager (respect its lockfile).
- Copy gitignored config the app needs to build or test, such as `.env`, `.env.local`, or
  local settings files.

Skip anything the repo does not have. Do not start services or servers.

## Step 4 — Spawn one subagent per feature

Launch all subagents in parallel, one per worktree, each with a prompt built from this
template:

```
Work in the worktree at <absolute path to REPO-FEATURE>. It is an isolated checkout on
branch feature/FEATURE; every file you touch must be inside it.

Implement: <FEATURE, or the full feature brief if one was given>

Requirements:
- Build the smallest complete implementation that satisfies the brief. Complete means it
  works end to end, not that it is elaborate. Do not add infrastructure, abstraction
  layers, configuration, or adjacent features the brief did not ask for — a brief naming
  one page means one page. When you catch yourself designing a subsystem, stop and
  implement the brief instead.
- Match the depth of the surrounding codebase. Reuse what is already there rather than
  building a more general version of it.
- Tests for the behavior you add, plus error handling on the paths that need it. Scale
  both to the feature: a static page needs a render test, not a suite.
- Build the project and run the tests. Do NOT start the application or any long-running
  process (no `npm run dev`, no servers, no watch mode) — parallel agents would collide
  on ports.
- Do NOT edit these shared files: <SHARED FILES list, or "none identified">. Other agents
  are adding their own features to those same files right now, and every edit to them
  becomes a merge conflict. Build your feature complete in every other respect — it just
  will not be reachable from the app's navigation yet. That is expected.
- When done, write a summary to <REPO_ROOT>/FEATURE.work.txt covering: what was
  implemented, files created and modified, dependencies added, testing approach and
  results, and integration notes for merging this branch. End it with an INTEGRATION
  POINTS section giving the exact edit each shared file needs — file path, and the literal
  lines to add — so it can be applied verbatim without reopening your work. Write "none"
  if the feature needs no such edit.
- Commit your work on the feature/FEATURE branch.
```

## Step 5 — Collect results

Wait for every subagent. For each one, confirm `REPO_ROOT/FEATURE.work.txt` exists and
read it. If a subagent finished without writing its summary, write the file from what it
reported before moving on.

A subagent can also die mid-run, which looks the same as one still working. If a worktree
has shown no new file activity for several minutes while others progress, inspect it:
`git status` and the file listing tell you whether anything was salvaged. If the tree is
untouched, relaunch that feature once with the same brief, plus whatever steered it wrong
— a stalled agent is usually stuck loading something it did not need. Report any stall and
relaunch in the final summary rather than hiding it.

## Step 6 — Report

Give one consolidated report:

- A per-feature status line: implemented, partial, or failed.
- What landed for each feature — files touched, dependencies added, tests run.
- A table of worktree path and branch name per feature.
- Integration next steps: suggested merge order, any features that touched the same files
  and will conflict, and the cleanup command:

  ```bash
  git worktree remove ../REPO-FEATURE
  ```

Then consolidate the INTEGRATION POINTS sections from every summary into one wiring plan,
grouped by shared file rather than by feature, so each file's full set of additions is
visible at once. Decide the details that need a view across all features — nav ordering,
route precedence, import grouping — rather than accepting each feature's local guess.

Do not apply the wiring yourself: the branches are not merged yet, and editing the shared
files now would conflict with the merges. Present the plan and offer to apply it as a
single commit once the user has merged the feature branches.

## Notes

- Pick features that are genuinely independent. Isolated worktrees prevent agents from
  overwriting each other live, but two agents editing the same file still produce
  conflicting branches at merge time. Say so up front when the requested features overlap.
- A conflict in a shared file is worse than it looks: resolving it by taking one side
  compiles, passes tests, and silently drops the other feature's registration, leaving
  working code unreachable. Keeping agents out of those files is what prevents it.
- `*.work.txt` is scratch output. Suggest adding it to `.gitignore` if it is not already.
