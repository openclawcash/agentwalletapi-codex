OpenClawCash Agent Wallet — Codex plugin.

Managed EVM and Solana wallets for AI agents: balances, transfers, swaps, approvals, governance policy
checks, cross-chain bridges, Get Paid checkout escrow, Polymarket, and YieldWolf Casino — over the
`openclawcash` MCP server (`@openclawcash/mcp-server`), with the `agentwalletapi` skill bundled for the
same safety model and wallet-label rules.

Install:

```bash
codex plugin marketplace add openclawcash/agentwalletapi-codex
```

Set `OPENCLAWCASH_AGENT_KEY` (create one at https://openclawcash.com) before starting Codex. The key is
only ever sent to `https://openclawcash.com`. Licensed MIT.
