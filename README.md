# parallel-agents

A reusable [Claude Code](https://claude.com/claude-code) slash command that builds several
features at once: one Git worktree and one subagent per feature, then a consolidated
summary of everything that landed.

Each feature gets a fully isolated checkout, so the agents never trip over each other's
edits, and every feature arrives on its own branch ready to review and merge.

## Install

Drop the command file into a `.claude/commands/` directory:

```bash
# available in one project
mkdir -p your-project/.claude/commands
curl -o your-project/.claude/commands/parallel-agents.md \
  https://raw.githubusercontent.com/jrcopeti/parallel-agents/main/.claude/commands/parallel-agents.md

# or available everywhere, for your user
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/parallel-agents.md \
  https://raw.githubusercontent.com/jrcopeti/parallel-agents/main/.claude/commands/parallel-agents.md
```

The user-level install is the one to prefer: the workflow runs from the *parent* folder
that holds your project repos, and a user-level command is available there without that
parent having to be a Git repo itself.

## Usage

```
/parallel-agents <repo-folder> <feature-1> <feature-2> ...
```

The first argument is the repo folder; every argument after it is one feature to build in
parallel. Run Claude Code from the **parent folder** of that repo.

```
/parallel-agents expense-tracker dark-mode csv-export budget-alerts
```

That call creates:

| Worktree                        | Branch                  | Summary written to                   |
| ------------------------------- | ----------------------- | ------------------------------------ |
| `../expense-tracker-dark-mode`     | `feature/dark-mode`     | `expense-tracker/dark-mode.work.txt`     |
| `../expense-tracker-csv-export`    | `feature/csv-export`    | `expense-tracker/csv-export.work.txt`    |
| `../expense-tracker-budget-alerts` | `feature/budget-alerts` | `expense-tracker/budget-alerts.work.txt` |

## How it works

1. **Setup worktrees** — creates `../<repo>-<feature>` on branch `feature/<feature>` for
   each feature, and mirrors the main repo's dev environment into it (dependencies, any
   gitignored env files it needs).
2. **Spawn subagents** — one subagent per worktree, each implementing its feature with
   tests and error handling. They compile and run tests but never start the app, so
   nothing fights over a port.
3. **Coordination** — waits for every subagent and checks each wrote its
   `<feature>.work.txt` summary into the main repo.
4. **Final summary** — reads those files back and reports what was implemented, files
   touched, dependencies added, testing approach, and the worktree and branch for each
   feature.

## Notes

- Feature names should be single-token kebab-case — they become directory suffixes and
  branch names.
- Pick features that are genuinely independent. Agents editing the same files in separate
  worktrees still produce conflicting branches at merge time.
- The command stops and asks rather than guessing if the first argument isn't a Git repo,
  or if no features are given.
- Clean up when you're done merging: `git worktree remove ../<repo>-<feature>`.
