# Synthflow MCP Marketplace

A Claude Code [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) that distributes Synthflow integrations.

## Plugins

| Plugin | Description |
|--------|-------------|
| [`synthflow-mcp`](./plugins/synthflow-mcp) | Connects Claude Code to the Synthflow hosted MCP server (OAuth) for managing AI voice agents, phone numbers, calls, and workflows. Includes call-review and prompt-review skills and a docs-search connector. |

## Install

Add this marketplace, then install the plugin:

```
/plugin marketplace add SynthFlowAI/AnthropicPlugin
/plugin install synthflow-mcp@synthflow
```

For local testing, point at the directory instead:

```
/plugin marketplace add ./
/plugin install synthflow-mcp@synthflow
```

Each plugin has its own setup steps — see its README. The `synthflow-mcp` plugin prompts for your workspace region's MCP URL at enable time and authenticates with OAuth: run `/mcp` and sign in to Synthflow ([setup](./plugins/synthflow-mcp#setup)).

## Develop

Test a plugin without installing it:

```bash
claude --plugin-dir ./plugins/synthflow-mcp
```

Validate the marketplace and plugin manifests:

```bash
claude plugin validate .
```
