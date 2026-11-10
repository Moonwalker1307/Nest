---
name: github-workflow
description: Rules for branches, commits, pull requests and merging on GitHub. Use whenever creating a branch, committing, pushing, opening or updating a pull request, bringing a branch up to date with its base, resolving merge conflicts, merging a PR, or cleaning up after a merge.
---

# GitHub workflow

How to name branches, keep history clean, open pull requests and merge them.

## Never push straight to the default branch

Every change, however small, goes on its own branch and lands through a pull request. Never commit or push directly to `main` (or whatever the default branch is). If you find yourself on the default branch with changes, create a branch first: `git switch -c <type>/<short-description>`.

## Branch names

Every branch name says what is being committed on it.

- Format: `<type>/<short-description>` in lowercase kebab-case, for example `feat/github-workflow-skill`, `fix/safari-export-button`, `docs/install-steps`.
- Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`.
- The description names the actual change, in 2 to 5 words. Never use generic names such as `update`, `changes`, `wip`, `patch-1`, `my-branch`, or names made only of ids, dates or random suffixes.
- If an issue exists, the number may be added to the end: `fix/login-timeout-42`.
- If the work on a branch changes direction, rename the branch before opening a PR (`git branch -m <new-name>`, then push the new name and delete the old remote branch).
- If a tool or environment assigns a branch name you must use, keep it, and make the PR title say what is being committed instead.

## One purpose per pull request

- A PR does one thing: one feature, one fix, one refactor. Keep it small enough to review in one sitting.
- If the work grows to cover unrelated changes, split it into separate branches and PRs.
- Don't mix drive-by formatting or renames into a functional change. Put them in their own PR.

## Clean commits

Commits are kept as they are when merged (never squashed), so each one must stand on its own.

- Each commit builds and passes tests, and has a clear message (see the `commit-messages` skill).
- Before merging, fold "fix typo", "wip", "address review" and similar commits into the commit they belong to. Use `git commit --fixup=<sha>` and then `git rebase --autosquash origin/<base>` (run with `GIT_SEQUENCE_EDITOR=:` and `-i` when no editor is available), and force-push with lease.
- Squashing commits together locally to tidy a branch is fine. Squash *merging* the PR is not.

## Never commit secrets

- Never commit API keys, tokens, passwords, private keys, or `.env` files. Check `git diff --staged` before committing and keep such files in `.gitignore`.
- If a secret has been pushed, even to a branch that was later deleted, treat it as leaked: tell the user to rotate or revoke it right away. Removing the commit does not make it safe.

## Keeping a branch up to date

Rebase when possible.

1. `git fetch origin <base>`
2. `git rebase origin/<base>`
3. Resolve conflicts commit by commit, then `git rebase --continue`.
4. Push with `git push --force-with-lease`. Never use a plain `--force`.

Do not pull the base branch into a feature branch with a merge commit (`git pull` without `--rebase`, or "Update branch" with merge). Set `git config pull.rebase true` if it is not set.

Only rebase branches you own. If someone else is also pushing to the branch, or the branch is shared, ask before rewriting its history.

## Pull request title and description

- The title says what the PR changes, in imperative mood: "Add GitHub workflow skill", not "Updates" or the branch name.
- The description says what changed and why, and anything a reviewer should check or know.
- Link the issue when there is one, with a closing keyword so it closes on merge: `Closes #12`, `Fixes #12`.
- Follow the repository's PR template if it has one.

## Merging a pull request

Only merge when it is ready:

- CI is green on the current head commit.
- No review is requesting changes, and open review threads are resolved.
- The branch has no merge conflicts.

Never squash merge.

1. Prefer **rebase and merge** (`gh pr merge --rebase`, or `merge_method: "rebase"`). This keeps each commit and a straight history.
2. If a rebase merge is not possible (conflicts, the repository disallows it, or the commits must stay as merged), rebase the branch onto the base first and try again.
3. If it is still not possible, use a **merge commit** (`gh pr merge --merge`, or `merge_method: "merge"`).
4. Never use `--squash` or `merge_method: "squash"`, even if it is the repository default or the only option offered. If squash is the only method the repository allows, stop and ask.

## After merging

Delete the branch, both on GitHub and locally:

- `gh pr merge` with `--delete-branch`, or `git push origin --delete <branch>`
- `git switch <base> && git pull && git branch -d <branch>`
- `git fetch --prune` to drop stale remote-tracking branches.

## Rules

- Never push straight to the default branch.
- Never squash merge.
- Rebase rather than merge to update a branch.
- Branch names always describe the change.
- One purpose per PR, and every commit stands on its own.
- Never commit secrets; rotate any that leak.
- Never merge with red CI or a changes-requested review.
- Delete the branch after merging.
- Never rewrite history on the default branch or on a branch someone else owns.
