---
name: sync-fork
description: Sync the fork's main branch from the upstream remote and push to origin
argument-hint: "[upstream-remote-name, defaults to 'upstream']"
disable-model-invocation: true
allowed-tools: Bash(git *)
---

Sync the fork by pulling latest `main` from the upstream remote and pushing to `origin`.

The upstream remote name defaults to `upstream`. If `$ARGUMENTS` is provided, use that instead.

Steps:
1. Determine the upstream remote: use `$ARGUMENTS` if provided, otherwise `upstream`
2. Verify the remote exists: `git remote get-url <remote>` — stop and inform the user if not found
3. Save the current branch name using `git rev-parse --abbrev-ref HEAD`
4. Run `git checkout main`
5. Run `git pull <remote> main`
6. Run `git push origin main`
7. Checkout the branch saved in step 3
