---
name: commit-messages
description: Write clear git commit messages from staged changes. Use when the user asks for a commit message, asks to commit, or wants help describing a change.
---

# Commit messages

Write a commit message that explains what changed and why, based on the staged diff.

## Steps

1. Run `git diff --staged` to see the change. If nothing is staged, ask whether to stage everything or only some files.
2. Work out the single purpose of the change. If the diff does several unrelated things, suggest splitting it into separate commits.
3. Write the message in this format:

```
<summary in imperative mood, at most 72 characters>

<body: why the change was needed and anything a reviewer should know.
Wrap lines at 72 characters. Leave the body out for trivial changes.>
```

## Rules

- Start the summary with a verb in imperative mood: "Add", "Fix", "Remove", not "Added" or "Fixes".
- Do not end the summary with a period.
- Explain the reason in the body. The diff already shows what changed.
- Follow the repository's existing style if it differs, for example Conventional Commits (`feat:`, `fix:`). Check with `git log --oneline -20`.
