# Synthflow plugin

Connects Claude to the [Synthflow](https://synthflow.ai) AI voice-agent platform, and ships the review skills Claude uses on top of it.

Once connected, Claude can call Synthflow tools directly — managing assistants, phone numbers, calls, knowledge bases, actions, and simulations from your editor.

## Which connection to use

There are two ways to get the Synthflow tools, and you want **one** of them:

| Your setup | Use |
|------------|-----|
| claude.ai, Claude Desktop, Claude Cowork | The official **Synthflow connector** — it handles region selection during sign-in |
| Claude Code, interactive | Either. The connector if you already have it; otherwise this plugin's bundled server |
| Claude Code headless, CI, cron | An API-key connection set up outside the plugin (see [Headless / CI](#headless--ci)) |

If you use the connector, keep this plugin installed anyway — the skills below work against either connection.

## Skills

| Skill | What it does |
|-------|--------------|
| `call-review` | Audits the last 100 calls for an agent against a problem checklist (unresolved outcomes, repetition, dropout, compliance risks) and reports flagged calls with evidence. |
| `prompt-review` | Reviews a voice-agent prompt before deployment: contradictions, missing context, tool/action alignment, escalation gaps, compliance risk, and regression risk. |
| `safe-agent-change` | Changes a live agent through baseline → draft → diff → approval → publish → verify, preserving attached actions, with a guided rollback. |
| `regression-test` | Builds and runs simulated-caller test suites against an agent and reports pass/fail per case with transcript evidence, separating agent failures from bad test cases. |
| `call-report` | Builds call-volume and outcome reports for an agent, workspace, or agency: statuses, durations, call reasons from extractors, transfer and booking results, and trends, with every percentage shown against its count. |
| `build-action` | Creates, tests, and attaches actions (custom HTTP, extractors, transfers, SMS, booking, email) using the exact MCP field shapes, with secrets kept out of chat. |
| `diagnose-call` | Investigates why a specific call went wrong (transfer, cut-off, silence, failed action or webhook) and labels each cause as confirmed, likely, or not determinable from the available data. |

Claude invokes these automatically when the task fits; you can also run them explicitly, e.g. `/synthflow:call-review`.

## Regions

Synthflow runs three regional MCP endpoints, and **the endpoint must match your workspace's region** — a mismatch (e.g. the EU URL with a US workspace) stops the connection from working. Find your region under **Admin → Workspace Settings → Preferences → Customer Region**.

**With the official connector, this is handled for you** — region selection happens during sign-in, so there's nothing to configure.

The rest of this section applies only to the plugin's bundled server and to servers you add by hand:

| Region | MCP URL |
|--------|---------|
| Global | `https://mcp.synthflow.ai/mcp` |
| United States | `https://mcp.us.synthflow.ai/mcp` |
| European Union | `https://mcp.eu.synthflow.ai/mcp` |

The bundled server uses the Global endpoint. For a US or EU workspace, either use the official connector, or add the server with your region's URL:

```bash
claude mcp add --transport http synthflow https://mcp.eu.synthflow.ai/mcp
```

Then authenticate via `/mcp` as usual. You can keep the plugin enabled for its skills — its Global server will simply show as unauthenticated, which is safe to ignore.

## Setup

Either path authenticates with **OAuth** — you sign in through Synthflow and select a workspace, so there is no API key to copy or manage by hand.

### Official connector

Enable the Synthflow connector from Claude's connector directory and sign in, picking your region and workspace when prompted. Nothing else to configure — this is the recommended path on claude.ai, Desktop, and Cowork, and fine in interactive Claude Code.

### Plugin's bundled server

1. Install the plugin (Global-region workspaces work out of the box; US/EU, see [Regions](#regions)):

   ```
   /plugin install synthflow --marketplace SynthFlowAI/AnthropicPlugin
   ```

2. Run `/mcp`, select the `synthflow` server, and choose **Authenticate**. A browser window opens to Synthflow's sign-in, where you pick the workspace and click **Allow access**.
3. `/mcp` should now show the server as connected, with its tools listed.

### Headless / CI

Both paths above sign in through a browser. For headless, CI, or scheduled runs, connect to the Synthflow MCP server with an API key as described in [Synthflow's MCP server docs](https://docs.synthflow.ai/mcp-server). The plugin doesn't collect or read API keys; its skills work with whichever connection is set up.

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

- **`synthflow`** — the workspace server (Global endpoint — see [Regions](#regions) for US/EU workspaces). Claude Code detects that the server requires OAuth and handles the sign-in flow via `/mcp`. Tools added to the hosted server appear automatically — no plugin update required. The URL is deliberately static: config placeholders like `${user_config.*}` only resolve inside Claude Code and break the connector on other Claude surfaces. Skip this one if you're using the official connector.
- **`synthflow-docs`** — searches the official docs at [docs.synthflow.ai](https://docs.synthflow.ai) via a `searchDocs` tool. The workspace server and the official connector both expose an equivalent `search_docs`, so this exists for one reason: it needs no sign-in, letting Claude look up Synthflow features and setup steps before you've authenticated anything.

## Troubleshooting

- **Server not listed in `/mcp`** — make sure the plugin is enabled (`/plugin`), then run `/reload-plugins`.
- **Connection or sign-in fails outright** — check for a region mismatch: the configured MCP URL must match your workspace's Customer Region (see [Regions](#regions)).
- **`401 Unauthorized` after previously working** — the OAuth token expired and refresh failed. Open `/mcp`, select the server, and choose **Re-authenticate**.
- **Wrong workspace data** — you selected a different workspace during sign-in, or you're on a subaccount. Re-authenticate and pick the right workspace.

## Reference

- Synthflow connector for Claude & Claude Code: https://docs.synthflow.ai/anthropic
- Synthflow MCP server (API-key clients): https://docs.synthflow.ai/mcp-server
