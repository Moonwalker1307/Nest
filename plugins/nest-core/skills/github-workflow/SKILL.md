---
name: github-workflow
description: Rules for branches, rebasing and merging pull requests on GitHub. Use whenever creating a branch, pushing, opening or updating a pull request, bringing a branch up to date with its base, resolving merge conflicts, or merging a PR.
---

# GitHub workflow

How to name branches, keep them up to date, and merge pull requests.

## Branch names

Every branch name says what is being committed on it.

- Format: `<type>/<short-description>` in lowercase kebab-case, for example `feat/github-workflow-skill`, `fix/safari-export-button`, `docs/install-steps`.
- Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`.
- The description names the actual change, in 2 to 5 words. Never use generic names such as `update`, `changes`, `wip`, `patch-1`, `my-branch`, or names made only of ids, dates or random suffixes.
- If an issue exists, the number may be added to the end: `fix/login-timeout-42`.
- If the work on a branch changes direction, rename the branch before opening a PR (`git branch -m <new-name>`, then push the new name and delete the old remote branch).
- If a tool or environment assigns a branch name you must use, keep it, and make the PR title say what is being committed instead.

## Keeping a branch up to date

Rebase when possible.

1. `git fetch origin <base>`
2. `git rebase origin/<base>`
3. Resolve conflicts commit by commit, then `git rebase --continue`.
4. Push with `git push --force-with-lease`. Never use a plain `--force`.

Do not pull the base branch into a feature branch with a merge commit (`git pull` without `--rebase`, or "Update branch" with merge). Set `git config pull.rebase true` if it is not set.

Only rebase branches you own. If someone else is also pushing to the branch, or the branch is shared, ask before rewriting its history.

## Merging a pull request

Never squash merge.

1. Prefer **rebase and merge** (`gh pr merge --rebase`, or `merge_method: "rebase"`). This keeps each commit and a straight history.
2. If a rebase merge is not possible (conflicts, the repository disallows it, or the commits must stay as merged), rebase the branch onto the base first and try again.
3. If it is still not possible, use a **merge commit** (`gh pr merge --merge`, or `merge_method: "merge"`).
4. Never use `--squash` or `merge_method: "squash"`, even if it is the repository default or the only option offered. If squash is the only method the repository allows, stop and ask.

Because commits are kept as they are, clean them up before merging: each commit should build and have a clear message (see the `commit-messages` skill). Use `git rebase -i` style fixups locally if needed, then force-push with lease.

## Rules

- Never squash merge.
- Rebase rather than merge to update a branch.
- Branch names always describe the change.
- Never rewrite history on the default branch or on a branch someone else owns.
