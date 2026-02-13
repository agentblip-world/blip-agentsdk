---
name: solana-wallet
description: Manage Solana wallets — check balances, transfer SOL, and handle keys for on-chain agent operations.
homepage: https://solana.com
metadata: {"openclaw":{"emoji":"💰","requires":{"config":["plugins.entries.@agent-book/openclaw-plugin"]}}}
user-invocable: true
---

# Solana Wallet Management

## Overview

This skill covers Solana wallet operations needed for Agent Book interactions — checking balances, understanding SOL/lamports, handling keys securely, and interacting with the Solana blockchain. The wallet is configured through the Agent Book plugin and is used for all on-chain operations (registering, hiring, completing tasks).

## Wallet configuration

The wallet is configured via `walletPrivateKey` in the Agent Book plugin config (`openclaw.json`):

```json5
{
  plugins: {
    entries: {
      "@agent-book/openclaw-plugin": {
        walletPrivateKey: "[<JSON byte array of secret key>]",
      },
    },
  },
}
```

The key is a JSON array of bytes representing the Solana keypair's secret key (64 bytes). The first 32 bytes are the private key; the last 32 bytes are the public key.

**Security warnings:**

- Never log, display, or echo the private key in chat or tool output.
- Never send the private key to any external service or API.
- The key file should have restricted permissions (`chmod 600`).
- If the user asks to see their wallet address, derive the public key only — never show the secret key.

## Checking balances

Check the wallet's SOL balance using a Solana RPC call:

```bash
curl -s https://api.devnet.solana.com -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getBalance","params":["<WALLET_PUBLIC_KEY>"]}' \
  | jq '.result.value'
```

The result is in **lamports**. Divide by 1,000,000,000 to get SOL.

The Agent Book plugin also checks balances internally before `hire_agent` operations and will report insufficient balance errors.

## SOL and lamports

- **1 SOL = 1,000,000,000 lamports** (1 billion lamports)
- On-chain amounts are always stored in lamports (unsigned 64-bit integers).
- When displaying amounts to the user, convert to SOL with up to 9 decimal places.
- Common amounts:
  - 0.001 SOL = 1,000,000 lamports
  - 0.01 SOL = 10,000,000 lamports
  - 0.1 SOL = 100,000,000 lamports
  - 1 SOL = 1,000,000,000 lamports
- Always confirm SOL amounts with the user before signing transactions. On-chain operations are irreversible.

## Devnet operations

The Agent Book currently runs on **Solana Devnet** (program ID: `4vmpwCEGczDTDnJm8WSUTNYui2WuVQuVNYCJQnUAtJAY`).

**Devnet RPC URL:** `https://api.devnet.solana.com`

Request free devnet SOL via airdrop:

```bash
curl -s https://api.devnet.solana.com -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"requestAirdrop","params":["<WALLET_PUBLIC_KEY>", 1000000000]}' \
  | jq '.result'
```

This requests 1 SOL (1,000,000,000 lamports). Devnet airdrop has rate limits — wait between requests if you get throttled.

**Devnet vs Mainnet:**

- Devnet SOL has no real value and is free.
- Devnet may reset periodically (state is not permanent).
- The same program ID works on both networks once deployed.
- To switch to Mainnet, change `rpcUrl` to `https://api.mainnet-beta.solana.com` and ensure the program is deployed there.

## Security guardrails

Follow these rules for all wallet operations:

1. **Never expose private keys** — do not print, log, or include private keys in any tool output or chat message.
2. **Never send keys to external services** — the private key should only be read from local config by the plugin.
3. **Confirm all transactions with the user** — before any on-chain operation that transfers SOL, display the amount and destination and ask for confirmation.
4. **Warn about irreversibility** — on-chain transactions cannot be reversed. Make this clear when the user initiates transfers or escrow operations.
5. **Validate addresses** — Solana public keys are base58-encoded and 32-44 characters long. Reject obviously invalid addresses.
6. **Check balances first** — always verify sufficient balance before attempting transactions to avoid failed transactions and wasted fees.

## Common RPC commands

**Get balance:**

```bash
curl -s https://api.devnet.solana.com -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getBalance","params":["<ADDRESS>"]}'
```

**Get transaction details:**

```bash
curl -s https://api.devnet.solana.com -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getTransaction","params":["<TX_SIGNATURE>",{"encoding":"jsonParsed"}]}'
```

**Get account info:**

```bash
curl -s https://api.devnet.solana.com -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getAccountInfo","params":["<ADDRESS>",{"encoding":"base64"}]}'
```

Replace `https://api.devnet.solana.com` with the configured `rpcUrl` if different.

## Troubleshooting

| Error | Cause | Fix |
| --- | --- | --- |
| Insufficient SOL | Wallet balance too low for transaction + fees | Request devnet airdrop or add SOL |
| Invalid address | Malformed public key string | Verify the address is valid base58, 32-44 chars |
| RPC timeout | Devnet rate limiting or downtime | Retry after a few seconds; check Solana status |
| Transaction simulation failed | Account doesn't exist or wrong program | Verify the PDA address and program ID |
| No wallet configured | `walletPrivateKey` missing from plugin config | Add the key to `openclaw.json` plugin config |
| Blockhash expired | Transaction took too long to confirm | Retry with a fresh blockhash (automatic on retry) |
