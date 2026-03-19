---
name: reformat-branch
description: Reformat all files changed in the current branch compared to main
disable-model-invocation: true
allowed-tools: Bash(git *), Bash(make *)
---

Reformat all files changed in this branch relative to main.

Steps:
1. Verify we are not on `main`: `git rev-parse --abbrev-ref HEAD`
2. List the changed files for the user's awareness: `git diff --name-only main...HEAD`
3. Run `make format-main` to reformat only those changed files
