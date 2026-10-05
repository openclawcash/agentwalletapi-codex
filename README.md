# OpenClawCash Agent Wallet (Codex plugin)

A crypto wallet for AI agents on Ethereum, Polygon, Base and Solana. Your agent gets an **API key, never
a private key**, and every action is checked against the spending limits, allowlist and testnet-only rules
you set before anything is signed. It can send, swap, bridge, trade on Polymarket and get paid through
escrow. Backed by [OpenClawCash](https://openclawcash.com).

The plugin connects Codex to the `openclawcash` MCP server (`@openclawcash/mcp-server` on npm) and bundles
the `agentwalletapi` skill, so Codex follows the same safety model, approval flow, and wallet-label rules
as every other OpenClawCash integration. Wallet keys and policy enforcement stay server-side.

## Install

```bash
codex plugin marketplace add openclawcash/agentwalletapi-codex
```

Then set your API key before starting Codex:

```bash
export OPENCLAWCASH_AGENT_KEY=occ_your_api_key
```

Get a key at [openclawcash.com](https://openclawcash.com) (sign up, create a wallet, open API Keys).

## Requirements

| | |
|---|---|
| Required env var | `OPENCLAWCASH_AGENT_KEY` |
| Optional env var | `OPENCLAWCASH_BASE_URL` (default `https://openclawcash.com`) |
| Runtime | Node.js with `npx` available |

The key is only ever sent to `https://openclawcash.com` (or an `https://<subdomain>.openclawcash.com` host).

## What you get

- The `openclawcash` MCP server: wallet, transfer, swap, bridge, approval, escrow checkout and Polymarket
  tools, all running through your wallet policies.
- The `agentwalletapi` skill — endpoint reference, safety model, and wallet-label rules; see
  [`skills/agentwalletapi/SKILL.md`](skills/agentwalletapi/SKILL.md).

## Layout

```
.codex-plugin/plugin.json          plugin manifest
.agents/plugins/marketplace.json   marketplace index for this repo
.mcp.json                          MCP server declaration
skills/agentwalletapi/             the skill
```

## License

MIT — see [LICENSE](LICENSE). "OpenClawCash" is a trademark of OpenClawCash; forks must not imply
endorsement.
