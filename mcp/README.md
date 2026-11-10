# MCP configs

Shared or example MCP server configs. A plugin that needs an MCP server
should ship its own `.mcp.json` inside its plugin folder; this folder is for
configs you copy into other projects.

Never commit secrets. Use environment variable placeholders instead:

```json
{
  "mcpServers": {
    "example": {
      "command": "npx",
      "args": ["-y", "some-mcp-server"],
      "env": { "API_KEY": "${EXAMPLE_API_KEY}" }
    }
  }
}
```
