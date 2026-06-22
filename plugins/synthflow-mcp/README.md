# Synthflow MCP

Connects Claude Code to the [Synthflow](https://synthflow.ai) AI voice-agent platform through its hosted MCP server at `https://mcp.synthflow.ai/mcp`.

Once enabled, Claude can call Synthflow tools directly — managing assistants, phone numbers, calls, and workflows from your editor.

## Prerequisites

A Synthflow API key. Generate one in **Admin → Workspace Settings → API Keys**.

## Setup

The plugin reads your API key from the `SYNTHFLOW_API_KEY` environment variable, so the key is never stored in the plugin or committed to git.

Export it in your shell profile (`~/.zshrc`, `~/.bashrc`, etc.):

```bash
export SYNTHFLOW_API_KEY="sk_your_key_here"
```

Reload your shell (or open a new terminal), then start Claude Code. Verify the connection:

```
/mcp
```

You should see the `synthflow` server listed as connected, along with its available tools.

## How it works

The plugin ships a single MCP server definition (`.mcp.json`):

```json
{
  "mcpServers": {
    "synthflow": {
      "type": "http",
      "url": "https://mcp.synthflow.ai/mcp",
      "headers": {
        "Authorization": "Bearer ${SYNTHFLOW_API_KEY}"
      }
    }
  }
}
```

`${SYNTHFLOW_API_KEY}` is expanded from your environment at load time.

## Troubleshooting

- **`Unauthorized: Invalid or expired token`** — `SYNTHFLOW_API_KEY` is unset, wrong, or expired. Confirm with `echo $SYNTHFLOW_API_KEY` in the same shell that launched Claude Code, then regenerate the key in Workspace Settings if needed.
- **Server not listed in `/mcp`** — make sure the plugin is enabled (`/plugin`), then run `/reload-plugins`.
- **Env var not picked up** — Claude Code reads the environment of the process that launched it. After editing your shell profile, open a new terminal or `source` it before relaunching.

## Reference

- Synthflow MCP docs: https://docs.synthflow.ai/mcp-server
