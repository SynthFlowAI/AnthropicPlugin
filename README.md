# Synthflow AI - Claude Code/Cowork plugin

Official Synthflow plugin for Claude Code and Claude Cowork. Manage your AI voice agents directly from Claude.

## Installation

### claude.ai, Claude Desktop, Claude Cowork

Enable the official **Synthflow connector** from Claude's connector directory and sign in — it handles region selection for you, so there's nothing to configure. Install this plugin alongside it for the bundled [skills](#bundled-skills).

### Claude Code

If you already use the official connector, install the plugin for its skills and skip step 2 — the connector provides the tools.

1. Add the marketplace and install the plugin:
    ```
    /plugin marketplace add SynthFlowAI/AnthropicPlugin
    /plugin install synthflow@synthflow
    ```
    The plugin's bundled server connects to the Global endpoint. If your workspace runs in the US or EU region, see [Regions](#regions).

2. Authenticate via OAuth:
    ```
    /mcp
    ```
    Select the `synthflow` server and press Enter, then follow the browser prompts to log into Synthflow, pick your workspace, and click **Allow access**.

## Regions

Synthflow runs three regional MCP endpoints, and the endpoint must match your workspace's region — a mismatch stops the connection from working. Find your region under **Admin → Workspace Settings → Preferences → Customer Region**.

**The official connector handles this during sign-in.** The table below applies to the plugin's bundled server and to servers you add by hand:

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

## Headless / CI

The official connector needs an interactive sign-in, so it isn't available in headless, CI, or scheduled runs — use the plugin's bundled server with a Synthflow API key instead (generate one in **Admin → Workspace Settings → API Keys**):

```bash
claude mcp add --transport http synthflow https://mcp.synthflow.ai/mcp \
  --header "Authorization: Bearer $SYNTHFLOW_API_KEY"
```

Substitute your region's URL from the table above.

## Features

This plugin gives Claude access to Synthflow tools across these categories:

- **Agents** — create, update, version, publish, and roll back voice agents
- **Calls** — place test calls, review transcripts, and pull call analytics
- **Knowledge bases** — create knowledge bases from files or URLs and attach them to agents
- **Actions** — build and test API integrations agents can call mid-conversation
- **Phone numbers** — manage numbers, phone books, and contacts
- **Simulations** — run scripted test scenarios against agents before publishing
- **Webhook logs** — inspect integration request/response payloads when debugging
- **Documentation** — search the Synthflow docs, no sign-in required

### Bundled skills

The plugin also ships task-specific skills that Claude loads on demand to follow Synthflow best practices:

- **call-review** — audits an agent's recent calls against a problem checklist (unresolved outcomes, repetition, dropout, compliance risks) and reports flagged calls with transcript evidence.
- **prompt-review** — reviews a voice-agent prompt before deployment: contradictions, missing context, tool/action alignment, escalation gaps, compliance risk, and regression risk.

Skills activate automatically when their description matches your request (e.g. "review my calls", "audit this prompt before I publish"), so you generally don't need to invoke them by name.

## Example usage

```
> List my agents and summarize what each one does
> Create a new inbound sales qualification agent and publish it

> Show me the last 20 calls and flag any with negative sentiment
> Review the last 100 calls for my support agent and surface problems

> Review this prompt before I publish it
> Compare the draft against the live version — any regression risk?

> Create a knowledge base from these sources, then attach it to my support agent
> Run a simulation where a prospect asks about pricing and implementation time
```

## Documentation

- [Synthflow connector for Claude & Claude Code](https://docs.synthflow.ai/anthropic)
- [Synthflow MCP server](https://docs.synthflow.ai/mcp-server)
- [Plugin configuration and troubleshooting](./plugins/synthflow)

## License

MIT
