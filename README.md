# OpenClawCash Codex plugin

Connects OpenAI Codex to the [OpenClawCash agent API](https://openclawcash.com/mcp) via the same `@openclawcash/mcp-server` used by Claude Code and every other MCP client — no separate Codex-specific backend.

## Install

```bash
codex plugin marketplace add openclawcash/agentwalletapi-codex
```

This repo carries its own `.agents/plugins/marketplace.json`, self-referencing only — kept because, unlike Claude Code, Codex's docs never confirm a bare single-plugin repo can skip a marketplace file. It's not shared with the sibling `../claude-code/` or `../hermes/` repos, which live in the same local folder only for development convenience.

Then set your API key before starting Codex:

```bash
export OPENCLAWCASH_AGENT_KEY=occ_your_api_key
```

## Structure

Uses Codex's "legacy compatibility" plugin layout (`.codex-plugin/plugin.json` + `.mcp.json` + `skills/` at the plugin root — this repo's own root), which the official docs confirm is still supported alongside the newer portable `agent-plugins.org` layout. `skills/agentwalletapi/` is hard-linked to `../claude-code/skills/agentwalletapi/` — same bytes, one copy on disk, but this repo still stands alone if published or cloned separately (verified: zipping this folder by itself, with no sibling present, still produces complete, correct file content). Neither is the actual source of truth; the `agentwalletapiSkill` repo is — see `../scripts/sync-skill.sh`.

## Maintainers

`.codex-plugin/plugin.json`'s `interface.defaultPrompt` must stay an array of at most 3 strings, each ≤128 characters — a reference implementation I checked while building this had it as a single string, which doesn't match the current spec. Re-check both constraints after editing it.
