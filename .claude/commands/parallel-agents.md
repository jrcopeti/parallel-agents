---
description: Develop multiple features in parallel using Git worktrees and subagents
argument-hint: <repo-folder> <feature-1> <feature-2> ...
---

<!--
USAGE (reference — not an instruction):

  /parallel-agents <repo-folder> <feature-1> <feature-2> ...

Example:

  /parallel-agents expense-tracker dark-mode csv-export budget-alerts

  Creates ../expense-tracker-dark-mode, ../expense-tracker-csv-export and
  ../expense-tracker-budget-alerts, each on branch feature/<name>, and runs one
  subagent per worktree. Summaries land as <feature>.work.txt in expense-tracker/.

Notes:
  - Launch Claude Code from the PARENT folder of the repo (e.g. ~/code) — that is
    what the workflow below assumes. Where this command file itself lives does not
    matter; install it user-level (~/.claude/commands/) so it is available there.
  - The first argument is the repo folder, relative to that parent folder.
  - Feature names should be single-token kebab-case: they become directory
    suffixes and branch names.
  - Pick features that are genuinely independent. Agents editing the same files
    in separate worktrees produce conflicting branches at merge time.
  - The command stops and asks if the first argument isn't a Git repo, or if no
    features are given.
-->

I want to develop features in parallel using Git worktrees and subagents.

Arguments: $ARGUMENTS

Split that argument string on whitespace. The FIRST token is the main repo folder — call
its basename REPO (e.g. `expense-tracker`). EVERY token after it is one feature to build
in parallel — call each one a FEATURE (kebab-case it if it isn't already). Parse the
tokens yourself from the line above; do not rely on positional placeholders.

State your parse back before doing anything else ("REPO = x; FEATURES = a, b"), so a
mis-parse is caught before any worktree is created.

You are in the parent folder of the main repo. You will need to change to the main repo
folder to create the worktrees. If the first argument is missing, is not a directory, or
is not a Git repository, stop and ask for the correct repo folder instead of guessing.
If no features are given, stop and ask which features to build.

Please execute this complete workflow:

PHASE 1 - SETUP WORKTREES:
For each FEATURE:
1. Create a worktree at ../REPO-FEATURE with branch feature/FEATURE
2. Set up the development environment in each worktree (if needed) — mirror whatever the
   main repo uses (install dependencies, copy any gitignored env/config files it needs)
3. List all worktrees created

PHASE 2 - SPAWN SUBAGENTS:
For each FEATURE, run a subagent in parallel with these instructions:
- You are working in the REPO-FEATURE worktree directory
- This is a completely isolated development environment
- Implement the FEATURE feature with full functionality
- Include proper testing and error handling
- Compile and run tests, but don't attempt to run the application (e.g., don't do "npm run" or "npm run dev &", etc.)
- When complete, write a detailed summary in FEATURE.work.txt in the main repo directory
- The summary should include: what was implemented, files created/modified, dependencies added, testing approach, and integration notes

PHASE 3 - COORDINATION:
- Monitor all subagents working in parallel
- Ensure each subagent completes their feature implementation
- Verify each subagent creates their work summary file

PHASE 4 - FINAL SUMMARY:
After all subagents complete:
1. Read all the .work.txt files created by subagents in the main repo directory
2. Provide a comprehensive summary of what was accomplished
3. List all features implemented and their status
4. Provide next steps for integration, including the worktree path and branch name for each feature

Execute this complete parallel development workflow.
