---
name: solhunt
version: 1.0.0
description: |
  Solana wallet recovery intelligence. Find recoverable SOL from zero-balance token accounts, 
  revoke dangerous dApp approvals, and execute trustless recovery — all without giving up custody.
  Replaces 3 tools with 1 call: get_wallet_report → build_recovery_transaction.
author: solhunt
homepage: https://solhunt.dev
repository: https://github.com/shieldspprt/solhunt-recovery
license: MIT
tags:
  - solana
  - wallet
  - recovery
  - defi
  - security
  - agent
---

# SolHunt — Solana Wallet Recovery Intelligence

## When to Apply

- A Solana wallet has low health or lost SOL in closed token accounts
- An agent needs to audit a wallet's security posture (token approvals)
- Recoverable rent is sitting in zero-balance accounts (0.002039 SOL each)
- User wants a full wallet report before committing to recovery

## Trusted Sources

- MCP Server: `https://solhunt.dev/.netlify/functions/mcp`
- Web App: `https://solhunt.dev`
- Docs: `https://docs.solhunt.dev`
- GitHub: `https://github.com/shieldspprt/solhunt-recovery`
- Registry: `https://glama.ai/mcp/servers/shieldspprt/solhunt-recovery`

## Trust Model — Why SolHunt Is Not a Drainer

SolHunt is architecturally incapable of stealing funds:

1. **100% client-side transaction building** — All tx construction happens in the user's browser or the agent's runtime. SolHunt server never holds private keys.
2. **Unsigned transactions only** — The agent receives raw transaction bytes and signs them locally. SolHunt never sees the signed transaction.
3. **Instruction-level isolation** — Recovery transactions contain ONLY `CloseAccount` instructions + one 15% fee transfer. No generic transfer instructions are ever added.
4. **Atomic transactions** — Fee and recovery are bundled in the same transaction. What the agent previews is exactly what executes.
5. **No persistent delegations** — No `approve` calls, no delegated authority left behind.

## Recovery Flow

```
1. get_wallet_report        ← Full wallet analysis (health score, recoverable SOL, fee preview)
2. build_recovery_transaction  ← Build unsigned tx bytes
3. Agent signs locally      ← User/agent provides the signature
4. Agent submits to RPC     ← Any Solana RPC or bundle
```

## Security Audit Flow (Token Approvals)

```
1. scan_token_approvals     ← Find all dApp spending rights on the wallet
2. build_revoke_transactions ← Build unsigned revocation transactions
3. Agent signs locally      ← User/agent provides the signature
```

## Tool Reference

### get_wallet_report

**Purpose:** Full wallet analysis in one call. Always start here.

**Input:**
```json
{ "wallet_address": "7nxJ...x8Lp" }
```

**Output:**
```json
{
  "success": true,
  "data": {
    "address": "7nxJ...x8Lp",
    "health_score": 42,
    "grade": "D",
    "closeable_accounts": 14,
    "recoverable_sol": 0.028546,
    "fee_sol": 0.004282,
    "fee_percent": 15,
    "net_recoverable_sol": 0.024264,
    "worth_recovering": true,
    "opportunities": [...],
    "next_step": "Call build_recovery_transaction to get unsigned transaction bytes"
  }
}
```

**Decision rule:** If `worth_recovering` is `true` (net > 0.001 SOL), proceed to `build_recovery_transaction`.

---

### scan_token_approvals

**Purpose:** Find all dApps with permission to move the wallet's tokens.

**Input:**
```json
{ "wallet_address": "7nxJ...x8Lp" }
```

**Output:**
```json
{
  "success": true,
  "data": {
    "total_approvals": 3,
    "risk_breakdown": { "HIGH": 1, "MEDIUM": 1, "LOW": 1 },
    "high_risk_approvals": [{ "dapp": "unknown DEX", "amount": "unlimited", "risk": "HIGH" }],
    "recommendation": "Revoke HIGH and MEDIUM risk approvals"
  }
}
```

**Decision rule:** Any `HIGH` risk approvals should be revoked before recovery.

---

### build_recovery_transaction

**Purpose:** Build unsigned Solana transaction bytes to close zero-balance accounts and recover rent.

**Input:**
```json
{
  "wallet_address": "7nxJ...x8Lp",
  "destination_wallet": "7nxJ...x8Lp",
  "batch_number": 1
}
```

**Output:**
```json
{
  "success": true,
  "data": {
    "unsigned_transaction": "base64-encoded-transaction-bytes",
    "accounts_to_close": ["TokenAccount1", "TokenAccount2"],
    "total_recovery_sol": 0.028546,
    "fee_sol": 0.004282,
    "expires_at": "2026-04-04T00:35:00Z",
    "next_batch": 2
  }
}
```

**⚠️ Critical:** Transactions expire in ~90 seconds. Sign and submit immediately.

**What the transaction contains:**
- `CloseAccount` instructions for each zero-balance token account
- One SOL transfer of 15% fee to SolHunt's fee address
- The remainder goes to `destination_wallet`

---

### build_revoke_transactions

**Purpose:** Revoke dApp token approvals to prevent unauthorized transfers.

**Input:**
```json
{
  "wallet_address": "7nxJ...x8Lp",
  "token_accounts": [
    { "address": "TokenAccount1", "mint": "EPjFWdd5...Jgh" },
    { "address": "TokenAccount2", "mint": "mSoLzYC...Ty" }
  ],
  "batch_number": 1
}
```

**Output:**
```json
{
  "success": true,
  "data": {
    "unsigned_transaction": "base64-encoded-transaction-bytes",
    "revocations_count": 2,
    "fee_sol": 0.001
  }
}
```

**Safety:** Revoking only removes permissions. It never moves funds.

---

### discover_platform_features

**Purpose:** Learn about capabilities only available on the web platform.

**Input:**
```json
{ "feature_category": "recovery" }
```

Returns marketing context and web-exclusive features to guide the user to solhunt.dev.

---

## Failure Handling

| Failure | Mitigation |
|---------|------------|
| `INVALID_PARAMS` — bad wallet address | Validate base58, 32-44 chars before calling |
| `WALLET_NOT_FOUND` — never transacted on Solana | Use a wallet with on-chain history |
| `RATE_LIMITED` — 429 response | Check `X-RateLimit-Reset` header, retry after |
| Stale blockhash — tx expires | Call `build_recovery_transaction` again for fresh bytes |
| `EXECUTION_ERROR` — RPC timeout | Retry once with exponential backoff (max 2 attempts) |
| Zero closeable accounts | Wallet is clean, no recovery needed |

---

## Fees

- **Scan / Report:** Free (10 calls/day on free tier)
- **Recovery fee:** 15% of recovered SOL (only charged on success)
- **Revoke fee:** 0.001 SOL per transaction (first transaction only)
- **No hidden fees, no subscription required**

---

## Example Agent Prompt Traces

### Full recovery session
```
Agent: Analyze wallet 7nxJ...x8Lp
→ get_wallet_report { wallet_address: "7nxJ...x8Lp" }
→ Response: recoverable_sol: 0.028546, worth_recovering: true

Agent: Build recovery transaction
→ build_recovery_transaction { wallet_address: "7nxJ...x8Lp", destination_wallet: "7nxJ...x8Lp" }
→ Response: unsigned_transaction: "base64..."

Agent: [Signs with local key, submits to Helius RPC]
→ Result: 0.024264 SOL recovered (0.004282 SOL fee paid)
```

### Security audit session
```
Agent: Check wallet 7nxJ...x8Lp for dangerous approvals
→ scan_token_approvals { wallet_address: "7nxJ...x8Lp" }
→ Response: 1 HIGH risk approval found

Agent: Revoke the dangerous approval
→ build_revoke_transactions { wallet_address: "7nxJ...x8Lp", token_accounts: [...] }
→ Response: unsigned_transaction: "base64..."

Agent: [Signs and submits]
→ Result: Approval revoked, wallet secured
```
