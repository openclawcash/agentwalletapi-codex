# OpenclawCash API Endpoint Details

## Requirements

- Required env var: `AGENTWALLETAPI_KEY`
- Optional env var: `AGENTWALLETAPI_URL` (default `https://openclawcash.com`)
- Required local binary for bundled CLI script: `curl`
- Optional local binary: `jq` (used for pretty JSON output when available)
- Network access: `https://openclawcash.com`

## Security Notes

- Start with read-only calls first (`wallets`, `wallet`, `policy`, `balance`, `supported-tokens`), preferably on testnets.
- Write actions (`create`, `import`, `transfer`, `swap`, `approve`, `polymarket-*`) are high-risk and should use explicit confirmation in the CLI (`--yes`).
- `POST /api/agent/wallets/import` sends a private key to OpenclawCash for encrypted storage and managed execution.
- Wallet import and wallet creation are disabled unless the API key has permission enabled in dashboard (`allowWalletImport`, `allowWalletCreation`).
- API keys may also be scoped by chain (`all`/`evm`/`solana`) and by wallet (`all` or a specific set of selected wallets).
- `AGENTWALLETAPI_URL` may only be `https://openclawcash.com` or an `https://<subdomain>.openclawcash.com` host. The bundled CLI script validates this before attaching `X-Agent-Key` to any request and refuses to run otherwise, so the key cannot be redirected off-domain by an env var override.

## API Surfaces

- **Agent API (`/api/agent/*`)**: authenticate with `X-Agent-Key`.
- **Public install metadata API (`/api/public/agentwalletapi/skill/latest`)**: no auth required.
- **Public token list API (`/api/public/tokenlist`)**: no auth required. Token Lists v1 document covering every supported chain.

## Skill Install Metadata (Public, No Auth)

```
GET /api/public/agentwalletapi/skill/latest
```

Response:
```json
{
  "skill": "agentwalletapi",
  "version": "latest",
  "publishedAssets": {
    "skillMdUrl": "https://openclawcash.com/agentwalletapi/SKILL.md",
    "githubRepoUrl": "https://github.com/openclawcash/agentwalletapi",
    "setupScriptUrl": "https://openclawcash.com/agentwalletapi/scripts/setup.sh",
    "cliScriptUrl": "https://openclawcash.com/agentwalletapi/scripts/agentwalletapi.sh"
  },
  "installInstructions": {
    "targetDirectory": "<your-workspace>/skills/agentwalletapi",
    "steps": [
      "Preferred: run `npx -y @openclawcash/mcp-server` if this client supports MCP servers (requires AGENTWALLETAPI_KEY in the environment); skip the remaining steps if so",
      "Fallback only, when MCP is unavailable: git clone https://github.com/openclawcash/agentwalletapi <your-workspace>/skills/agentwalletapi",
      "Read <your-workspace>/skills/agentwalletapi/SKILL.md before running any command from the cloned skill"
    ],
    "agentPrompt": "Prefer MCP: if this client supports MCP servers, run \"npx -y @openclawcash/mcp-server\" (requires AGENTWALLETAPI_KEY in the environment) and stop there. Only if MCP is unavailable, git clone https://github.com/openclawcash/agentwalletapi into <your-workspace>/skills/agentwalletapi, then read <your-workspace>/skills/agentwalletapi/SKILL.md before running anything inside it."
  }
}
```

## Public Token List (Public, No Auth)

```
GET /api/public/tokenlist
GET /api/public/tokenlist?chainId=8453
GET /api/public/tokenlist?extended=false
```

Token Lists v1 document (https://uniswap.org/tokenlist.schema.json). Covers every chain OpenClawCash supports across EVM and Solana.

- **Default `?extended=true`**: curated OpenClawCash tokens merged with Uniswap default list (EVM) and Jupiter verified list (Solana). Curated entries win on (chainId, address) dedupe so logos and names are consistent. ~5000+ tokens.
- **`?extended=false`**: curated only (~67 tokens). Zero external dependencies, smaller payload, longer cache.
- **`?chainId=<num>`**: scope to one chain. EVM uses EIP-155 (1=Mainnet, 137=Polygon, 8453=Base, 11155111=Sepolia). Solana uses Solana Labs convention (101=mainnet).

CORS-allowed for browser consumers. `Cache-Control: public, max-age=60` (extended) or `300` (curated).

Response shape:
```json
{
  "name": "OpenClawCash Tokens (extended via Uniswap + Jupiter)",
  "timestamp": "2026-05-03T12:00:00.000Z",
  "version": { "major": 1, "minor": 0, "patch": 0 },
  "keywords": ["openclawcash", "managed-wallets", "agent-wallet", "extended", "uniswap", "jupiter"],
  "tokens": [
    { "chainId": 1, "address": "0xA0b8...eB48", "name": "USD Coin", "symbol": "USDC", "decimals": 6, "logoURI": "https://..." },
    { "chainId": 8453, "address": "0x8335...2913", "name": "USD Coin", "symbol": "USDC", "decimals": 6, "logoURI": "https://..." },
    { "chainId": 101, "address": "EPjFWdd5...zybapC8G4wEGGkZwyTDt1v", "name": "USD Coin", "symbol": "USDC", "decimals": 6, "logoURI": "https://..." }
  ]
}
```

## Global User Tag (Checkout Identity)

Checkout uses one account-level user tag for seller/buyer identity.

Read current value:
```
GET /api/agent/user-tag
X-Agent-Key: occ_your_api_key
```

Set value once (immutable after set):
```
PUT /api/agent/user-tag
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "userTag": "studio"
}
```

Response:
```json
{
  "userTag": "studio"
}
```

Notes:
- Tag format: lowercase letters/numbers with `.`, `_`, `-`, length 3-8.
- `PUT` returns `409 user_tag_locked` if already set.

## List Wallets

```
GET /api/agent/wallets
X-Agent-Key: occ_your_api_key
```

Returns discovery data only (id/label/address/network/chain) by default. Use `GET /api/agent/wallet` for full balances, or add `?includeBalances=true` for native balance on each listed wallet.

Response:
```json
[
  { "id": 2, "label": "Trading Bot", "address": "0x14ae8d93...", "network": "sepolia", "chain": "evm" },
  { "id": 5, "label": "SOL TEST", "address": "GmjrX8...", "network": "solana-devnet", "chain": "solana" }
]
```

Optional native balances in list response:
```
GET /api/agent/wallets?includeBalances=true
X-Agent-Key: occ_your_api_key
```

Example response:
```json
[
  {
    "id": 5,
    "label": "SOL MAIN",
    "address": "3LuJ8...",
    "network": "solana-mainnet",
    "chain": "solana",
    "balance": "0.02134 SOL",
    "nativeSymbol": "SOL"
  }
]
```

## Get Wallet Detail + Balances

```
GET /api/agent/wallet?walletId=2
X-Agent-Key: occ_your_api_key
```

Alternative:
```
GET /api/agent/wallet?walletLabel=Trading%20Bot
X-Agent-Key: occ_your_api_key
```

Alternative (by managed wallet address):
```
GET /api/agent/wallet?walletAddress=0x14ae8d93...
X-Agent-Key: occ_your_api_key
```

Optional:
```
GET /api/agent/wallet?walletId=2&chain=evm
```

Response:
```json
{
  "id": "W123ABC",
  "label": "Trading Bot",
  "address": "0x14ae8d93...",
  "network": "sepolia",
  "chain": "evm",
  "nativeBalanceDisplay": "0.048",
  "nativeBalanceBaseUnits": "48000000000000000",
  "balance": "0.048 ETH",
  "nativeSymbol": "ETH",
  "otherTokenCount": 1,
  "tokenBalances": [
    { "token": "0x0000...0000", "symbol": "ETH", "balance": "0.048", "balanceBaseUnits": "48000000000000000", "decimals": 18 },
    { "token": "0xA0b86991...", "symbol": "USDC", "balance": "250.0", "balanceBaseUnits": "250000000", "decimals": 6 }
  ]
}
```

Note: Use `GET /api/agent/policies` or `GET /api/agent/policy` to retrieve wallet policies.

## Rename Wallet

```
PATCH /api/agent/wallet
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request (select the wallet with exactly one of `walletId`, `walletLabel` (its current label), or `walletAddress`):
```json
{ "walletId": "W123ABC", "label": "Trading Bot v2" }
```

Response:
```json
{
  "id": "W123ABC",
  "label": "Trading Bot v2",
  "address": "0x14ae8d93...",
  "network": "sepolia",
  "chain": "evm"
}
```

Notes:
- `label`: 1-32 characters using letters, numbers, spaces, and `. _ - ( ) #`, starting with a letter or number. No emoji or other non-ASCII, no digits-only labels, no embedded addresses. Repeated spaces are collapsed. Must not match (case-insensitive) another live wallet's label (`409 wallet_label_taken`) or any of your wallet IDs (`400 invalid_wallet_label`). The same rules apply to create, import, and venue provisioning.
- Metadata only: no funds move and no extra API key permission is required. The wallet must be within the key's wallet and chain scope.
- Rate limited per API key (10 renames per 10 minutes by default), separately from create/import. Exceeding it returns `429 wallet_write_rate_limited` with `Retry-After`.
- Errors: `400 validation_error`, `400 invalid_wallet_label`, `401` (missing/invalid key), `404 wallet_not_found`, `409 wallet_label_taken`, `409 wallet_label_ambiguous`, `429 wallet_write_rate_limited`.

### Duplicate labels on older accounts

Some accounts created before label rules existed have wallets that share a label. Selecting one of them with `walletLabel` on any endpoint returns:

```json
{
  "code": "wallet_label_ambiguous",
  "message": "More than one wallet on this account uses this label, so no wallet was selected.",
  "retryable": false,
  "details": {
    "matchingWalletIds": ["W123ABC", "W456DEF"],
    "renameEndpoint": "PATCH /api/agent/wallet",
    "renameMcpTool": "wallet_rename"
  }
}
```

Recover by retrying with `walletId`, then asking your human which wallet should get a new unique label and renaming it here. `details.matchingWalletIds` only lists wallets your API key is allowed to use.

## Create Wallet (Agent API)

```
POST /api/agent/wallets/create
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "label": "Agent Ops Wallet",
  "network": "sepolia",
  "exportPassphrase": "your-strong-passphrase",
  "exportPassphraseStorageType": "env",
  "exportPassphraseStorageRef": "WALLET_EXPORT_PASSPHRASE_AGENT_OPS",
  "confirmExportPassphraseSaved": true
}
```

Response:
```json
{
  "id": 12,
  "label": "Agent Ops Wallet",
  "address": "0x1234...",
  "network": "sepolia",
  "chain": "evm"
}
```

Notes:
- API key must have wallet creation enabled (`allowWalletCreation`).
- Endpoint is rate-limited per API key; on limit exceeded returns `429` + `Retry-After`.
- Create requires `exportPassphrase` (minimum 12 characters).
- Create also requires `exportPassphraseStorageType` and `exportPassphraseStorageRef`.
- Agent must persist passphrase first, then send the storage fields plus `confirmExportPassphraseSaved: true`.

## Import Wallet (Agent API)

```
POST /api/agent/wallets/import
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "label": "Treasury Imported",
  "network": "solana-mainnet",
  "privateKey": "..."
}
```

Response:
```json
{
  "id": 13,
  "label": "Treasury Imported",
  "address": "GmjrX8...",
  "network": "solana-mainnet",
  "chain": "solana"
}
```

Notes:
- API key must have wallet import enabled (`allowWalletImport`).
- Supported networks: `mainnet`, `polygon-mainnet`, `base-mainnet`, `solana-mainnet`.
- Endpoint is rate-limited per API key; on limit exceeded returns `429` + `Retry-After`.

## Get Policies

List policies across all wallets for the authenticated API key:

```
GET /api/agent/policies
X-Agent-Key: occ_your_api_key
```

Response:
```json
[
  {
    "wallet": {
      "id": "W123ABC",
      "label": "Trading Bot",
      "address": "0x14ae8d93...",
      "network": "sepolia",
      "chain": "evm"
    },
    "policies": [
      {
        "id": 31,
        "type": "daily_spending_limit",
        "config": { "amount": "100" },
        "createdAt": "2026-01-15T10:00:00.000Z",
        "usage": {
          "spent": "45.00",
          "limit": "100.00",
          "symbol": "USD",
          "decimals": 2,
          "window": "24h"
        }
      }
    ]
  }
]
```

Notes:
- Returns hydrated policy data including current usage for spending limit policies.
- `usage` is populated for `daily_spending_limit`, `weekly_spending_limit`, and `monthly_spending_limit` types; other policy types return without a `usage` field.
- Usage data shows `spent`, `limit`, `symbol`, `decimals`, and `window` (24h/week/month).
- Full policy `type` list: `whitelist`, `spending_limit`, `daily_spending_limit`, `weekly_spending_limit`, `monthly_spending_limit`, `disallow_live_transactions`, `wallet_purpose`, `checkout_access`, `venue_access`, `max_open_escrows`, `trusted_counterparty_tags`.
- A blocked write returns `403 policy_violation` with a `policyType` field naming which policy blocked the request.

## Get Policy

Get policies for a specific wallet by walletId, walletLabel, or walletAddress:

```
GET /api/agent/policy?walletId=W123ABC
X-Agent-Key: occ_your_api_key
```

Alternative selectors:
```
GET /api/agent/policy?walletLabel=Trading%20Bot
GET /api/agent/policy?walletAddress=0x14ae8d93...
```

Response:
```json
{
  "wallet": {
    "id": "W123ABC",
    "label": "Trading Bot",
    "address": "0x14ae8d93...",
    "network": "sepolia",
    "chain": "evm"
  },
  "policies": [
    {
      "id": 31,
      "type": "daily_spending_limit",
      "config": { "amount": "100" },
      "createdAt": "2026-01-15T10:00:00.000Z",
      "usage": {
        "spent": "45.00",
        "limit": "100.00",
        "symbol": "USD",
        "decimals": 2,
        "window": "24h"
      }
    }
  ]
}
```

Notes:
- Requires exactly one wallet selector: `walletId`, `walletLabel`, or `walletAddress`.

## Wallet Transaction History

```
GET /api/agent/transactions?walletId=2
X-Agent-Key: occ_your_api_key
```

Alternative:
```
GET /api/agent/transactions?walletLabel=Trading%20Bot
X-Agent-Key: occ_your_api_key
```

Alternative (by managed wallet address):
```
GET /api/agent/transactions?walletAddress=0x14ae8d93...
X-Agent-Key: occ_your_api_key
```
Optional:
```
GET /api/agent/transactions?walletId=2&chain=evm
```

EVM bucket model: scope to a single EVM chain or merge across every EVM chain in the wallet's bucket:
```
GET /api/agent/transactions?walletId=2&network=base-mainnet
GET /api/agent/transactions?walletId=2&network=all
```
- `network=<id>`: returns activity on that specific EVM chain (mainnet, polygon-mainnet, base-mainnet, sepolia).
- `network=all`: merges activity across every supported EVM chain into one history.
- Omitted: returns activity on the wallet's default chain.
- Each row's `data.network` indicates which chain it ran on.
- Solana wallets are pinned to their cluster; the field is rejected if it doesn't match.

Response:
```json
[
  {
    "id": 0,
    "walletId": 2,
    "hash": "5tS4...sig",
    "to": "GmjrX8...",
    "value": "1000000000",
    "fee": "5000",
    "type": "transfer",
    "status": "confirmed",
    "data": "{\"source\":\"on-chain\",\"direction\":\"incoming\",\"token\":\"SOL\"}",
    "createdAt": "2026-02-19T17:15:00.000Z"
  }
]
```

## Transfer Native or Tokens

```
POST /api/agent/transfer
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

### Fields

| Field | Type | Required | Description |
|---|---|---|---|
| walletId | number \| string | One of walletId/walletLabel/walletAddress | Wallet numeric ID or public wallet ID from list wallets |
| walletLabel | string | One of walletId/walletLabel/walletAddress | Wallet name from dashboard (or use walletAddress as alternative selector where supported) |
| chain | string | No | Optional guard: `"evm"` or `"solana"` |
| to | string | Yes | Recipient address (0x... for EVM, base58 for Solana) |
| token | string | No | Token symbol or token address/mint. Defaults to chain native token (ETH/SOL) |
| amountDisplay | string | One of amountDisplay/valueBaseUnits | Human-readable amount (e.g., "100" for 100 USDC) |
| valueBaseUnits | string | One of amountDisplay/valueBaseUnits | Amount in base units (e.g., "100000000" for 100 USDC with 6 decimals) |
| amount | string | Deprecated | Legacy alias for amountDisplay |
| value | string | Deprecated | Legacy alias for valueBaseUnits |
| memo | string | No | Solana-only transfer memo. Max 5 words, max 256 UTF-8 bytes, no control/invisible characters |

### Examples

Send 0.01 ETH:
```json
{ "walletId": 2, "to": "0xRecipient...", "amountDisplay": "0.01" }
```

Send 100 USDC by symbol:
```json
{ "walletLabel": "Trading Bot", "to": "0xRecipient...", "token": "USDC", "amountDisplay": "100" }
```

Send USDC by contract address + base units:
```json
{ "walletId": 2, "to": "0xRecipient...", "token": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", "valueBaseUnits": "100000000" }
```

Send arbitrary ERC-20 by address + human amount:
```json
{ "walletId": 2, "to": "0xRecipient...", "token": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", "amountDisplay": "100" }
```

Send 0.01 SOL:
```json
{ "walletId": "Q7X2K9P", "to": "SolanaRecipientWalletAddress...", "token": "SOL", "amountDisplay": "0.01" }
```
Send 0.01 SOL with memo:
```json
{ "walletId": "Q7X2K9P", "to": "SolanaRecipientWalletAddress...", "token": "SOL", "amountDisplay": "0.01", "memo": "payment verification note" }
```
Optional chain guard example:
```json
{ "chain": "solana", "walletId": "Q7X2K9P", "to": "SolanaRecipientWalletAddress...", "amountDisplay": "0.01" }
```

### Response

```json
{
  "txHash": "0xabc123...",
  "status": "confirmed",
  "token": "USDC",
  "tokenAddress": "0xA0b86991...",
  "requestedValueBaseUnits": "100000000",
  "adjustedValueBaseUnits": "100000000",
  "requestedAmountDisplay": "100",
  "adjustedAmountDisplay": "100",
  "valueBaseUnits": "100000000",
  "amountDisplay": "100",
  "fee": "1000000",
  "feePercent": "1%",
  "feeAmount": "1.0",
  "netValue": "99000000",
  "netAmount": "99.0",
  "feeWalletAddress": "0x...",
  "feeTxHash": "0xdef456...",
  "memo": "payment verification note"
}
```

Behavior notes:
- Checkout escrow destinations are enforced separately from generic transfer:
  - If `to` is an open escrow address and the transfer network or asset does not match checkout settlement rules, API returns `409` with code `unsupported_funding_network` or `unsupported_funding_asset`.
  - Error `details.acceptedFundingAssets` provides the allowed funding asset for that escrow.
  - For escrow funding, use checkout endpoints instead of generic transfer:
    - `POST /api/agent/checkout/escrows/:id/quick-pay`
    - `POST /api/agent/checkout/escrows/:id/swap-and-pay`
    - `POST /api/agent/checkout/escrows/:id/funding-confirm` (external/manual tx confirm)
- Native transfers (EVM + Solana) enforce a minimum transferable amount preflight that considers platform fee and network fee.
- For native SOL transfers, server estimates network fee and may reduce requested gross amount so transfer + platform fee + network fee fits wallet balance.
- For first-time funding of a brand-new Solana address, a larger minimum transfer may be required; too-small requests return `400` with code `amount_below_min_transfer`.
- For native SOL with configured Solana fee wallet, recipient transfer and platform fee transfer are sent in one transaction.
- Memo is accepted only for Solana wallets; providing memo on EVM returns `400 invalid_transfer_input`.
- Memo validation: max 5 words, max 256 UTF-8 bytes, rejects control/invisible characters.
- If requested transfer cannot fit after required fees, API returns `400` with code `insufficient_balance`.
- If requested native transfer is below the minimum transferable amount after fee/network preflight, API returns `400` with code `amount_below_min_transfer`.

## Check Balances

```
POST /api/agent/token-balance
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

All balances (native + discovered/default token set):
```json
{ "walletId": 2 }
```

Specific token by symbol:
```json
{ "walletId": 2, "token": "USDC" }
```

Specific token by address/mint:
```json
{ "walletId": 2, "tokenAddress": "0xA0b86991..." }
```

Response:
```json
{
  "balances": [
    { "token": "0x0000...0000", "symbol": "ETH", "balance": "0.048", "decimals": 18 },
    { "token": "0xA0b86991...", "symbol": "USDC", "balance": "250.0", "decimals": 6 },
    { "token": "0xdAC17F...", "symbol": "USDT", "balance": "0", "decimals": 6 }
  ]
}
```

## Supported Tokens

```
GET /api/agent/supported-tokens?network=mainnet
GET /api/agent/supported-tokens?network=sepolia
GET /api/agent/supported-tokens?network=solana-mainnet
GET /api/agent/supported-tokens?network=solana-devnet
GET /api/agent/supported-tokens?chain=solana
```

Requires `X-Agent-Key`. Returns **recommended common, well-known tokens** for the specified network (defaults to mainnet).
Agents can still use any valid ERC-20 token contract address on EVM and any valid SPL mint on Solana.

Response:
```json
{
  "recommendedTokens": [
    { "address": "0x0000...0000", "symbol": "ETH", "name": "Ether", "decimals": 18 },
    { "address": "0xA0b86991...", "symbol": "USDC", "name": "USD Coin", "decimals": 6 }
  ],
  "guidance": {
    "message": "These are recommended common, well-known tokens. You can still use any valid ERC-20 token on EVM or any valid SPL mint on Solana.",
    "evm": "Any valid ERC-20 token contract address is supported in agent wallet operations.",
    "solana": "Any valid SPL token mint address is supported in agent wallet operations."
  }
}
```

Notes:
- ETH is native and represented as zero-address in API payloads.
- ERC-20 addresses are network-specific (mainnet and sepolia differ).
- SOL is native on Solana and represented by `native:sol`.

## Get Swap Quote (DEX)

```
POST /api/agent/quote?network=mainnet
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "chain": "evm",
  "tokenIn": "WETH",
  "tokenOut": "USDC",
  "amountIn": "10000000000000000"
}
```

Response:
```json
{
  "amountOut": "31234567",
  "amountOutHuman": "31.234567",
  "amountIn": "10000000000000000",
  "amountInHuman": "0.01",
  "route": "0xC02a... -> 0xA0b8...",
  "feePercent": "0.30%",
  "dex": "uniswap-v2",
  "network": "mainnet"
}
```

Solana (Jupiter) request example:
```json
{
  "chain": "solana",
  "walletId": 5,
  "tokenIn": "SOL",
  "tokenOut": "USDC",
  "amountIn": "10000000"
}
```

Errors:
- `400 invalid_quote_request` (`retryable: false`) — unknown token, invalid address, `tokenIn` equal to `tokenOut`, non-positive or malformed `amountIn`, or an amount under the router minimum. The request must change before retrying.
- `503 no_route_or_liquidity` (`retryable: true`) — no DEX route or not enough pool liquidity for this pair/amount. Transient: retry the same request after a short delay, or adjust the amount. Do **not** treat this as a permanently unsupported pair.
- `503 upstream_rpc_unavailable` (`retryable: true`) — upstream RPC/DEX timed out, refused the connection, or rate-limited. Retry with backoff.
- `500 quote_failed` (`retryable: true`) — unclassified. Retry once, then vary pair/amount.

Error responses include `details.reason` with a fixed value (`invalid_input`, `route_or_liquidity`, `upstream_unavailable`, `unclassified_quote_error`). Raw upstream router/RPC text is never returned.

## Execute Swap (DEX)

```
POST /api/agent/swap
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "chain": "evm",
  "walletId": 2,
  "tokenIn": "ETH",
  "tokenOut": "USDC",
  "amountIn": "10000000000000000",
  "slippage": 0.5
}
```

Response:
```json
{
  "txHash": "0xabc123...",
  "status": "confirmed",
  "amountOut": "7791993",
  "amountOutMin": "7753033",
  "tokenIn": "ETH",
  "tokenOut": "USDC",
  "fee": "100000000000000",
  "feePercent": "1%",
  "approvalTxHash": null,
  "feeTxHash": "0xdef456..."
}
```

If balance is insufficient for tokenIn, API returns:
```json
{
  "message": "Insufficient WETH balance in wallet 4. Available: 0 WETH, required: 0.001 WETH.",
  "code": "insufficient_token_balance",
  "walletId": 4,
  "token": "WETH",
  "available": "0",
  "required": "0.001"
}
```

Solana (Jupiter) example:
```json
{
  "chain": "solana",
  "walletId": 5,
  "tokenIn": "SOL",
  "tokenOut": "USDC",
  "amountIn": "10000000",
  "slippage": 0.5
}
```

Errors:
- `400 insufficient_token_balance` — wallet lacks `tokenIn` (payload shown above).
- `400 invalid_swap_request` (`retryable: false`) — the swap parameters themselves are unusable (unknown token, invalid address, same token in and out, bad amount).
- `500 swap_failed` (`retryable: true`) — temporary DEX execution or routing issue, **including a pair that cannot currently be routed or lacks liquidity**. Request a fresh quote, then retry with a lower amount or higher slippage.

## Checkout & Escrow (Agent API)

All checkout endpoints require:
- `X-Agent-Key: occ_your_api_key`
- Write calls require `Idempotency-Key`

### Create Pay Request

```
POST /api/agent/checkout/payreq
```

Creates a signed pay request and escrow wallet.

Timing fields (plain meaning):
- `expiresInSeconds`: deadline for buyer funding before request expires.
- `autoReleaseSeconds`: point when funded escrow can auto-release if no dispute is opened.
- `disputeWindowSeconds`: dispute window length after the auto-release point.

Validation rules:
- Minimum `3600` (1 hour) for all three fields.
- `disputeWindowSeconds` must be less than or equal to `autoReleaseSeconds`.

### Metadata Field (checkoutClientMetadataSchema)

The `metadata` field on checkout requests supports structured client data with validation:

```
"metadata": {
  "orderId": "order-12345",
  "customerId": "cust-67890",
  "notes": "Please deliver by end of day"
}
```

Schema constraints:
- **Keys**: Max 64 characters, alphanumeric with `.`, `_`, `:`, `-`, `/`, spaces
- **Values**: Max 280 characters (512 bytes), max 3 levels nesting, max 20 keys per object
- Metadata is stored and returned as provided — no key is filtered or stripped. Do not put API keys, secrets, passwords, or other sensitive values in metadata.

Values are normalized (trimmed, Unicode NFKC normalized) before storage.

### Get Pay Request

```
GET /api/agent/checkout/payreq/:id
```

Returns pay request details and current escrow linkage.

### Confirm Funding

```
POST /api/agent/checkout/escrows/:id/funding-confirm
```

Validates on-chain funding using tx hash + confirmations.

### Get Escrow

```
GET /api/agent/checkout/escrows/:id
```

Returns escrow lifecycle state, tx hashes, proof/dispute fields, and settlement values.

### Accept / Proof / Dispute

```
POST /api/agent/checkout/escrows/:id/accept
POST /api/agent/checkout/escrows/:id/proof
POST /api/agent/checkout/escrows/:id/dispute
```

Use these endpoints to claim buyer role, submit proof, and open a dispute.

### Quick Pay (Direct)

```
POST /api/agent/checkout/escrows/:id/quick-pay
```

Direct funding path when buyer wallet already has enough settlement token.

### Swap And Pay

```
POST /api/agent/checkout/escrows/:id/swap-and-pay
```

Two-step flow:
- Quote with `confirm: false`
- Execute with `confirm: true`

### Release / Refund / Cancel

```
POST /api/agent/checkout/escrows/:id/release
POST /api/agent/checkout/escrows/:id/refund
POST /api/agent/checkout/escrows/:id/cancel
```

Terminal lifecycle actions for settlement or cancellation.

### Webhooks

```
GET /api/agent/checkout/webhooks
POST /api/agent/checkout/webhooks
PATCH /api/agent/checkout/webhooks/:id
DELETE /api/agent/checkout/webhooks/:id
```

Subscribe and manage escrow event deliveries (`escrow.funded`, `escrow.released`, etc.).

## Polymarket Venue Setup

- Agent endpoint setup is disabled.
- Ask your human to complete setup at: https://openclawcash.com/venues/polymarket
- After user setup is complete, use the agent venue order/read/redeem endpoints below.
- MCP convenience tool: `polymarket_market_resolve`
  - Purpose: resolve `marketUrl` or `slug` plus human-readable `outcome` to the exact `tokenId` required by order endpoints.
  - Typical MCP flow:
    1. Call `polymarket_market_resolve` with `{ marketUrl|slug, outcome }`
    2. Use returned `outcome.tokenId` in `POST /api/agent/venues/polymarket/orders/market` or `/limit`

## Polymarket Market Resolver (Agent API)

```
GET /api/agent/venues/polymarket/market/resolve?marketUrl=https://polymarket.com/market/<slug>&outcome=No
X-Agent-Key: occ_your_api_key
```

Alternative query form:

```
GET /api/agent/venues/polymarket/market/resolve?slug=<slug>&outcome=No
X-Agent-Key: occ_your_api_key
```

Response:
```json
{
  "venue": "polymarket",
  "market": {
    "slug": "market-slug",
    "question": "Will X happen?",
    "conditionId": "0x...",
    "url": "https://polymarket.com/market/market-slug",
    "active": true,
    "closed": false
  },
  "outcome": {
    "requested": "No",
    "normalized": "No",
    "index": 1,
    "tokenId": "123456789..."
  },
  "outcomes": [
    { "label": "Yes", "tokenId": "111...", "selected": false },
    { "label": "No", "tokenId": "123456789...", "selected": true }
  ]
}
```

## Polymarket Limit Order (Agent API)

```
POST /api/agent/venues/polymarket/orders/limit
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "walletId": "Q7X2K9P",
  "tokenId": "123456",
  "side": "BUY",
  "price": 0.54,
  "size": 25
}
```

Response:
```json
{
  "venue": "polymarket",
  "status": "filled",
  "orderId": "optional-order-id",
  "txHash": "optional-tx-hash"
}
```

## Polymarket Market Order (Agent API)

```
POST /api/agent/venues/polymarket/orders/market
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "walletId": "Q7X2K9P",
  "tokenId": "123456",
  "side": "BUY",
  "amount": 25,
  "orderType": "FAK",
  "worstPrice": 0.65
}
```

Response:
```json
{
  "venue": "polymarket",
  "status": "filled",
  "orderId": "optional-order-id",
  "txHash": "optional-tx-hash"
}
```

Notes:
- For close-position intent on open markets, prefer market `SELL` (`side: "SELL"`).
- Use limit `SELL` only when a specific target price is requested.
- `amount` semantics: `BUY` means notional/collateral amount; `SELL` means share amount.

## Polymarket Account Summary (Agent API)

```
GET /api/agent/venues/polymarket/account?walletId=Q7X2K9P
X-Agent-Key: occ_your_api_key
```

Response:
```json
{
  "venue": "polymarket",
  "walletId": "Q7X2K9P",
  "network": "polygon-mainnet",
  "account": {
    "balanceAllowance": {},
    "apiKeysCount": 1
  }
}
```

## Polymarket Open Orders (Agent API)

```
GET /api/agent/venues/polymarket/orders?walletId=Q7X2K9P&status=OPEN&limit=50
X-Agent-Key: occ_your_api_key
```

Response:
```json
{
  "venue": "polymarket",
  "walletId": "Q7X2K9P",
  "network": "polygon-mainnet",
  "items": [],
  "nextCursor": null
}
```

## Polymarket Cancel Order (Agent API)

```
POST /api/agent/venues/polymarket/orders/cancel
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "walletId": "Q7X2K9P",
  "orderId": "your-order-id"
}
```

Response:
```json
{
  "venue": "polymarket",
  "walletId": "Q7X2K9P",
  "network": "polygon-mainnet",
  "status": "cancel_requested",
  "orderId": "your-order-id"
}
```

## Polymarket Clear Integration (Agent API)

```
POST /api/agent/venues/polymarket/unlink
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "walletId": "Q7X2K9P"
}
```

Response:
```json
{
  "venue": "polymarket",
  "walletId": "Q7X2K9P",
  "walletAddress": "0x...",
  "network": "polygon-mainnet",
  "status": "cleared"
}
```

## Polymarket Activity (Agent API)

```
GET /api/agent/venues/polymarket/activity?walletId=Q7X2K9P&limit=50
X-Agent-Key: occ_your_api_key
```

Response:
```json
{
  "venue": "polymarket",
  "walletId": "Q7X2K9P",
  "network": "polygon-mainnet",
  "items": [],
  "nextCursor": null
}
```

## Polymarket Positions (Agent API)

```
GET /api/agent/venues/polymarket/positions?walletId=Q7X2K9P&limit=100
X-Agent-Key: occ_your_api_key
```

Response:
```json
{
  "venue": "polymarket",
  "walletId": "Q7X2K9P",
  "network": "polygon-mainnet",
  "items": [
    {
      "conditionId": "0x...",
      "question": "Will BTC be above 100k by month end?",
      "outcome": "Yes",
      "status": "OPEN",
      "size": 12.5,
      "avgPrice": 0.42,
      "curPrice": 0.47,
      "currentValue": 5.875,
      "cashPnl": 0.625,
      "percentPnl": 11.9
    }
  ]
}
```

Notes:
- Positions are sourced from Polymarket open positions (Data API-backed).
- Response is filtered to open markets only (`closed !== true`, `active !== false`, and not past `endDate`).
- Position items include `cashPnl`, `percentPnl`, and `currentValue` (with computed fallback values when upstream fields are missing).

## Polymarket Redeemable Positions (Agent API)

```
GET /api/agent/venues/polymarket/redeemable?walletId=Q7X2K9P&limit=100
X-Agent-Key: occ_your_api_key
```

Response:
```json
{
  "venue": "polymarket",
  "walletId": "Q7X2K9P",
  "network": "polygon-mainnet",
  "userAddress": "0x49273d9d882032e11a02c8A6d66239D22492A0a5",
  "items": [
    {
      "tokenId": "96960274267773372131583647731391642541817323912358042961702119095512805226258",
      "conditionId": "0x19a4b3b4e2d612f2d39d27ca1e00a00b077264951984b5d4957ad517e4986ed4",
      "outcomeIndex": 1,
      "size": "391.5212",
      "sizeBaseUnits": "391521200",
      "negativeRisk": false,
      "title": "Will the Kings win?",
      "outcome": "Kings"
    }
  ]
}
```

Notes:
- Backed by Polymarket Data API `GET /positions?redeemable=true`.
- `userAddress` is the exact Polymarket account address used for the redeemable lookup.
- Use `items[].tokenId` as input to single-position redeem.

## Polymarket Redeem (Agent API)

```
POST /api/agent/venues/polymarket/redeem
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request (single position):
```json
{
  "walletId": "Q7X2K9P",
  "tokenId": "1234567890",
  "limit": 100,
  "signatureType": 0
}
```

Request (redeem all redeemable positions):
```json
{
  "walletId": "Q7X2K9P"
}
```

Response:
```json
{
  "venue": "polymarket",
  "walletId": "Q7X2K9P",
  "network": "polygon-mainnet",
  "mode": "single",
  "requestedTokenId": "1234567890",
  "requested": 1,
  "attempted": 1,
  "remaining": 0,
  "hasMoreRedeemable": false,
  "maxPerRequest": 1,
  "signingPath": "direct",
  "signatureType": 0,
  "successful": 1,
  "failed": 0,
  "results": [
    {
      "tokenId": "1234567890",
      "conditionId": "0x...",
      "outcomeIndex": 1,
      "sizeBaseUnits": "1000000",
      "negativeRisk": false,
      "status": "success",
      "settlement": "submitted",
      "signingPath": "direct",
      "txHash": "0xabc..."
    }
  ]
}
```

Notes:
- Server picks the signing path automatically from the wallet's configured `signatureType`:
  - `0` (direct EOA) → on-chain redeem signed by the wallet itself; wallet must hold a small amount of POL on Polygon to pay gas (cents).
  - `1` (Polymarket proxy) or `2` (Gnosis Safe) → gasless redeem via Polymarket's relayer; requires API key/secret/passphrase configured for the wallet.
- Optional `signatureType` request field defensively asserts the wallet's configured signing type. Mismatch returns `400 venue_config_invalid`. Omit to dispatch by wallet config.
- Response includes top-level `signingPath` (`"direct"` | `"gasless"`) and `signatureType`, plus the same `signingPath` on each result item.
- Discover a wallet's `signatureType` via `GET /api/agent/wallets` (`polymarket.signatureType`) or `GET /api/agent/venues/polymarket/account`.
- Call `GET /api/agent/venues/polymarket/redeemable` first, then use one returned `tokenId` for targeted redeem.
- Omit `tokenId` to redeem all currently redeemable positions.
- `limit` controls how many redeemable positions are scanned when listing candidates (default `100`, max `200`).
- Redeem requests are processed in bounded chunks to avoid edge timeout failures on large `redeem all` calls.
- For `mode: "all"`, repeat redeem calls while `hasMoreRedeemable` is `true`.
- `settlement: "submitted"` means submission succeeded and tx hash is available; on-chain confirmation can be checked asynchronously via transaction history.
- For direct-path redeems against an empty wallet, the API returns `400 venue_insufficient_native_gas`; fund ~$0.01 of POL on Polygon and retry.

## Token Approval (ERC-20)

```
POST /api/agent/approve
Content-Type: application/json
X-Agent-Key: occ_your_api_key
```

Request:
```json
{
  "chain": "evm",
  "walletId": 2,
  "tokenAddress": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "spender": "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D",
  "amount": "1000000000"
}
```

Response:
```json
{
  "txHash": "0xabc123...",
  "status": "confirmed"
}
```

Notes:
- Use base units for `amount` (e.g., USDC 1000 with 6 decimals = `1000000000`).
- ETH (native token) does not require approval.
- Wallet must have ETH for gas.

## Networks

- **mainnet**: Ethereum Mainnet (real ETH, all tokens)
- **polygon-mainnet**: Polygon PoS Mainnet (real POL + ERC-20 on Polygon)
- **base-mainnet**: Base Mainnet, Coinbase L2 (real ETH + ERC-20 on Base; native USDC, WETH, DAI, cbETH, USDbC supported)
- **sepolia**: Sepolia Testnet (test ETH, limited token selection: ETH, USDC, WETH, LINK)
- **solana-mainnet**: Solana Mainnet (real SOL + SPL tokens)
- **solana-devnet**: Solana Devnet (dev SOL + test SPL tokens)
- **solana-testnet**: Solana Testnet (test SOL + test SPL tokens)

EVM wallets are buckets: a wallet's `network` is its **default/home chain**, not a binding. The same wallet address is valid on every EVM chain. Pass an optional `network` field on `/api/agent/transfer`, `/api/agent/swap`, and `/api/agent/approve` to operate the wallet on a non-default EVM chain. Omit `network` to use the wallet's default. Solana wallets remain pinned to their cluster.

## Important Notes

- EVM token transfers require native gas (ETH on mainnet/sepolia/base-mainnet; POL on polygon-mainnet) on the operating chain
- Solana token transfers require SOL in the wallet for transaction fees
- Native SOL transfers account for network fee and may return adjusted transfer values in response
- Swap supports EVM (Uniswap-v2-compatible router on mainnet/polygon/base/sepolia) and Solana mainnet (Jupiter); Quote supports EVM and Solana mainnet; Approve is EVM-only
- Polymarket on-chain execution targets `polygon-mainnet` under the hood; any EVM-home wallet (mainnet, polygon-mainnet, base-mainnet) can be linked to Polymarket
- All Polymarket order/read/redeem endpoints require exactly one wallet selector (`walletId` or `walletAddress`)
- Platform fee is deducted from the token amount (not native gas), consistent with native transfers
- For transfer, use `amountDisplay` for simplicity (human-readable), use `valueBaseUnits` when you need precise base-unit control (legacy `amount`/`value` aliases are still accepted)
- Optional `chain` guard is supported on agent endpoints; mismatches return `400` with `code: "chain_mismatch"`. Optional `network` override is supported on EVM write endpoints; unknown or non-EVM networks for an EVM wallet return `400` with `code: "network_mismatch"`.
