# Synthflow plugin

Connects Claude Code to the [Synthflow](https://synthflow.ai) AI voice-agent platform through its hosted MCP server.

Once connected, Claude can call Synthflow tools directly — managing assistants, phone numbers, calls, knowledge bases, actions, and simulations from your editor.

## Skills

| Skill | What it does |
|-------|--------------|
| `call-review` | Audits the last 100 calls for an agent against a problem checklist (unresolved outcomes, repetition, dropout, compliance risks) and reports flagged calls with evidence. |
| `prompt-review` | Reviews a voice-agent prompt before deployment: contradictions, missing context, tool/action alignment, escalation gaps, compliance risk, and regression risk. |

Claude invokes these automatically when the task fits; you can also run them explicitly, e.g. `/synthflow:call-review`.

## Regions

Synthflow runs three regional MCP endpoints. **The server URL and your workspace must be in the same region** — a mismatch (e.g. the EU URL with a US workspace) stops the connection from working. Find your region under **Admin → Workspace Settings → Preferences → Customer Region**.

| Region | MCP URL |
|--------|---------|
| Global | `https://mcp.synthflow.ai/mcp` |
| United States | `https://mcp.us.synthflow.ai/mcp` |
| European Union | `https://mcp.eu.synthflow.ai/mcp` |

The plugin's bundled connector uses the Global endpoint. For a US or EU workspace, add the server with your region's URL instead:

```bash
claude mcp add --transport http synthflow https://mcp.eu.synthflow.ai/mcp
```

Then authenticate via `/mcp` as usual. You can keep the plugin enabled for its skills — its Global connector will simply show as unauthenticated, which is safe to ignore.

## Setup

The server authenticates with **OAuth** — you sign in through Synthflow and select a workspace, so there is no API key to copy or manage by hand.

1. Install and enable the plugin (Global-region workspaces work out of the box; US/EU, see [Regions](#regions)).
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

The plugin ships two remote MCP server definitions (`.mcp.json`):

```json
{
  "mcpServers": {
    "synthflow": {
      "type": "http",
      "url": "https://mcp.synthflow.ai/mcp"
    },
    "synthflow-docs": {
      "type": "http",
      "url": "https://docs.synthflow.ai/_mcp/server"
    }
  }
}
```

- **`synthflow`** — the workspace server (Global endpoint — see [Regions](#regions) for US/EU workspaces). Claude Code detects that the server requires OAuth and handles the sign-in flow via `/mcp`. Tools added to the hosted server appear automatically — no plugin update required. The URL is deliberately static: config placeholders like `${user_config.*}` only resolve inside Claude Code and break the connector on other Claude surfaces.
- **`synthflow-docs`** — searches the official docs at [docs.synthflow.ai](https://docs.synthflow.ai) via a `searchDocs` tool. No sign-in needed, so Claude can look up Synthflow features and setup steps even before you authenticate.

## Troubleshooting

- **Server not listed in `/mcp`** — make sure the plugin is enabled (`/plugin`), then run `/reload-plugins`.
- **Connection or sign-in fails outright** — check for a region mismatch: the configured MCP URL must match your workspace's Customer Region (see [Regions](#regions)).
- **`401 Unauthorized` after previously working** — the OAuth token expired and refresh failed. Open `/mcp`, select the server, and choose **Re-authenticate**.
- **API-key connection returns `Unauthorized`** — the key is unset, wrong, or expired in the shell that launched Claude Code. Confirm with `echo $SYNTHFLOW_API_KEY`, then regenerate it in Workspace Settings if needed.
- **Wrong workspace data** — you selected a different workspace during sign-in, or you're on a subaccount. Re-authenticate and pick the right workspace.

## Reference

- Synthflow connector for Claude & Claude Code: https://docs.synthflow.ai/anthropic
- Synthflow MCP server (API-key clients): https://docs.synthflow.ai/mcp-server
