# parallel-agents

A [Claude Code](https://claude.com/claude-code) skill that builds several features at once:
one Git worktree and one subagent per feature, then a consolidated summary of everything
that landed.

Each feature gets a fully isolated checkout, so the agents never trip over each other's
edits, and every feature arrives on its own branch ready to review and merge.

## Install

A skill is a directory holding a `SKILL.md`. This one is a single file:

```bash
# available everywhere, for your user
mkdir -p ~/.claude/skills/parallel-agents
curl -o ~/.claude/skills/parallel-agents/SKILL.md \
  https://raw.githubusercontent.com/jrcopeti/parallel-agents/main/.claude/skills/parallel-agents/SKILL.md
```

For a single project, put it at `your-project/.claude/skills/parallel-agents/SKILL.md`
instead — and commit it, so your team gets it too.

To track this repo and get updates with `git pull`, clone it and symlink the skill
directory; Claude Code follows symlinks:

```bash
git clone https://github.com/jrcopeti/parallel-agents.git
ln -s "$PWD/parallel-agents/.claude/skills/parallel-agents" ~/.claude/skills/parallel-agents
```

Claude Code picks up new and edited skills without a restart.

## Usage

Run Claude Code from **inside** the repo you want to build in:

```
/parallel-agents <feature-1> <feature-2> ...
```

Every argument is one feature to build in parallel. The repo root is resolved from Git, so
there is no repo argument to pass.

```
cd ~/code/expense-tracker
/parallel-agents dark-mode csv-export budget-alerts
```

That call creates:

| Worktree                           | Branch                  | Summary written to                       |
| ---------------------------------- | ----------------------- | ---------------------------------------- |
| `../expense-tracker-dark-mode`     | `feature/dark-mode`     | `expense-tracker/dark-mode.work.txt`     |
| `../expense-tracker-csv-export`    | `feature/csv-export`    | `expense-tracker/csv-export.work.txt`    |
| `../expense-tracker-budget-alerts` | `feature/budget-alerts` | `expense-tracker/budget-alerts.work.txt` |

Single kebab-case tokens become directory suffixes and branch names directly. Quote an
argument to give a feature more direction — `/parallel-agents "dark mode with a system
preference toggle" csv-export` — and the slug is derived from it, with the full text
handed to that feature's subagent as its brief.

## How it works

1. **Parse and confirm** — states the repo and feature list back before creating anything,
   and stops to ask if a target branch or worktree already exists.
2. **Setup worktrees** — creates `../<repo>-<feature>` on branch `feature/<feature>` for
   each feature, and mirrors the main repo's dev environment into it (dependencies, any
   gitignored env files it needs).
3. **Spawn subagents** — one subagent per worktree, each implementing its feature with
   tests and error handling, then committing on its branch. They build and run tests but
   never start the app, so nothing fights over a port.
4. **Collect results** — waits for every subagent and checks each wrote its
   `<feature>.work.txt` summary into the main repo.
5. **Report** — reads those files back and reports what was implemented, files touched,
   dependencies added, testing approach, and the worktree and branch for each feature,
   plus a suggested merge order.

## Notes

- Pick features that are genuinely independent. Agents editing the same files in separate
  worktrees still produce conflicting branches at merge time.
- Worktrees branch from `HEAD`, so uncommitted work in the main checkout does not carry
  into them.
- The skill is manual-only (`disable-model-invocation: true`) — it runs when you type
  `/parallel-agents`, never on Claude's initiative, since each run creates branches,
  worktrees, and one subagent per feature. Drop that line from the frontmatter if you want
  Claude to reach for it on its own.
- Add `*.work.txt` to your `.gitignore`.
- Clean up when you're done merging: `git worktree remove ../<repo>-<feature>`.

<details>
<summary>Upgrading from the slash command</summary>

This was previously a command file at `.claude/commands/parallel-agents.md`, which took the
repo folder as its first argument and ran from the repo's *parent* directory. Custom
commands have since been merged into skills. Delete the old file
(`rm ~/.claude/commands/parallel-agents.md`), install the skill, and run it from inside the
repo with feature names only.

</details>
