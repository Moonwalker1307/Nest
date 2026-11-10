# AGENTS.md

Shared instructions for every coding agent working in this repo (Claude Code,
Codex, Cursor, Copilot). `CLAUDE.md` and `.github/copilot-instructions.md` only
point here, so edit this file, not those.

## What this repo is

Nest is a Claude Code plugin marketplace named `nest`. There is no app or build:
the repo is Markdown and JSON.

```
.claude-plugin/marketplace.json   lists every plugin (name, source, description, version)
plugins/<plugin>/
  .claude-plugin/plugin.json      name, version, description, author
  skills/<skill>/SKILL.md         skills, plus optional scripts/ and references/
  commands/*.md                   slash commands
  agents/*.md                     subagents
  hooks/hooks.json                hooks
  .mcp.json                       MCP servers the plugin brings
prompts/                          prompts that are not skills yet
mcp/                              shared or example MCP configs
templates/skill/SKILL.md          starter template for a new skill
```

## Adding a skill

1. Copy `templates/skill/` to `plugins/<plugin>/skills/<skill-name>/`.
2. In `SKILL.md`, set `name` to the folder name (kebab-case) and write a
   `description` that names when to use the skill. The description is what
   decides when the skill is picked.
3. Keep `SKILL.md` short and move long material into `references/`.
4. Bump the plugin's version (see below) and add the skill to the plugin table
   in `README.md`.

## Adding a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` with `name`, `version`
   (start at `0.1.0`), `description` and `author`.
2. Add an entry to `plugins` in `.claude-plugin/marketplace.json` with the same
   `name` and `version`, `"source": "./plugins/<name>"`, and a description.
3. Add the plugin to the plugin table and the install commands in `README.md`.

## Versions

Every change to a plugin bumps its version, in both `plugin.json` and its
`marketplace.json` entry, and the two must match. Installs only pick up updates
when the version changes. Use a minor bump for a new skill, command or agent,
and a patch bump for fixes to existing ones.

## Validate before committing

```
claude plugin validate .                  # marketplace manifest
claude plugin validate plugins/<plugin>   # each plugin you changed
```

Both must pass. Also check that any JSON you edited parses.

## Git rules

The full rules are in the `github-workflow` skill
(`plugins/nest-core/skills/github-workflow/SKILL.md`). The ones that matter most:

- Never push to `main`. Every change goes through a pull request.
- Never squash merge. Prefer rebase and merge, else a merge commit.
- Rebase onto `origin/main` to update a branch, then `git push --force-with-lease`.
  Never merge `main` into a branch.
- Branch names describe what is being committed. This repo uses
  `claude/<short-description>` for agent branches (for example
  `claude/add-dotnet-10-skill`); otherwise use `<type>/<short-description>`.
- Never reuse a branch name, even after its PR is merged and the branch deleted.
- One purpose per PR, and each commit stands on its own.
- "Automatically delete head branches" is on, so GitHub deletes merged branches.

## Writing style for skills and docs

Plain, direct sentences. Say what to do, not what the reader probably already
knows. No secrets anywhere; use `${ENV_VAR}` placeholders in MCP configs.
