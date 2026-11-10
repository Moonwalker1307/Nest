# Nest

Skills, plugins and other things for working with AI.

Nest is a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces).
Each plugin bundles skills, slash commands, agents, hooks and MCP servers
around one theme.

## Install

In Claude Code:

```
/plugin marketplace add Moonwalker1307/Nest
/plugin install nest-core@nest
```

Skill folders also work on their own: upload a skill folder to claude.ai
(Settings, Capabilities) or copy it into `~/.claude/skills/`.

## Layout

```
.claude-plugin/marketplace.json   lists every plugin in this repo
plugins/<plugin>/
  .claude-plugin/plugin.json      name, version, description
  skills/<skill>/SKILL.md         skills (plus optional scripts/, references/)
  commands/*.md                   slash commands
  agents/*.md                     subagents
  hooks/hooks.json                hooks
  .mcp.json                       MCP servers the plugin brings
prompts/                          prompts that are not skills yet
mcp/                              shared or example MCP configs
templates/skill/SKILL.md          starter template for a new skill
```

## Plugins

| Plugin | What it has |
| --- | --- |
| `nest-core` | `commit-messages` skill, `/explain` command |

## Adding things

- **New skill:** copy `templates/skill/` into `plugins/<plugin>/skills/<name>/`
  and fill in `SKILL.md`. The `description` decides when Claude uses it.
- **New plugin:** create `plugins/<name>/.claude-plugin/plugin.json`, then add
  an entry to `.claude-plugin/marketplace.json`.
- Bump the plugin's `version` when you change it, so installs pick up updates.
- Never commit secrets. Use `${ENV_VAR}` placeholders in MCP configs.
