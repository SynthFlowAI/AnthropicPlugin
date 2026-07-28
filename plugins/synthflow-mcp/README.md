# Synthflow MCP

Connects Claude Code to the [Synthflow](https://synthflow.ai) AI voice-agent platform through its hosted MCP server.

Once connected, Claude can call Synthflow tools directly — managing assistants, phone numbers, calls, knowledge bases, actions, and simulations from your editor.

## Regions

Synthflow runs three regional MCP endpoints. **The server URL and your workspace must be in the same region** — a mismatch (e.g. the EU URL with a US workspace) stops the connection from working. Find your region under **Admin → Workspace Settings → Preferences → Customer Region**.

| Region | MCP URL |
|--------|---------|
| Global | `https://mcp.synthflow.ai/mcp` |
| United States | `https://mcp.us.synthflow.ai/mcp` |
| European Union | `https://mcp.eu.synthflow.ai/mcp` |

When you enable the plugin, Claude Code prompts for the server URL (defaulting to Global) — paste your region's URL there. To change it later, re-run the prompt from `/plugin` or edit `pluginConfigs` in `~/.claude/settings.json`.

## Setup

The server authenticates with **OAuth** — you sign in through Synthflow and select a workspace, so there is no API key to copy or manage by hand.

1. Install and enable the plugin, entering your region's MCP URL when prompted.
2. Run `/mcp`, select the `synthflow` server, and choose **Authenticate**. A browser window opens to Synthflow's sign-in, where you pick the workspace and click **Allow access**.
3. `/mcp` should now show the server as connected, with its tools listed.

### Alternative: API key (headless / CI)

Environments without a browser can connect with a Synthflow API key instead (generate one in **Admin → Workspace Settings → API Keys**):

```bash
claude mcp add --transport http synthflow https://mcp.synthflow.ai/mcp \
  --header "Authorization: Bearer $SYNTHFLOW_API_KEY"
```

Substitute your region's URL from the table above. Claude Code deduplicates servers by URL, so this and the plugin's bundled server won't produce duplicate tools.

## How it works

The plugin ships a single remote MCP server definition (`.mcp.json`):

```json
{
  "mcpServers": {
    "synthflow": {
      "type": "http",
      "url": "${user_config.mcp_url}"
    }
  }
}
```

`${user_config.mcp_url}` is the region URL you chose at enable time (declared in `plugin.json` under `userConfig`, defaulting to the Global endpoint). Claude Code detects that the server requires OAuth and handles the sign-in flow via `/mcp`. Tools added to the hosted server appear automatically — no plugin update required.

## Troubleshooting

- **Server not listed in `/mcp`** — make sure the plugin is enabled (`/plugin`), then run `/reload-plugins`.
- **Connection or sign-in fails outright** — check for a region mismatch: the configured MCP URL must match your workspace's Customer Region (see [Regions](#regions)).
- **`401 Unauthorized` after previously working** — the OAuth token expired and refresh failed. Open `/mcp`, select the server, and choose **Re-authenticate**.
- **API-key connection returns `Unauthorized`** — the key is unset, wrong, or expired in the shell that launched Claude Code. Confirm with `echo $SYNTHFLOW_API_KEY`, then regenerate it in Workspace Settings if needed.
- **Wrong workspace data** — you selected a different workspace during sign-in, or you're on a subaccount. Re-authenticate and pick the right workspace.

## Reference

- Synthflow connector for Claude & Claude Code: https://docs.synthflow.ai/anthropic
- Synthflow MCP server (API-key clients): https://docs.synthflow.ai/mcp-server
