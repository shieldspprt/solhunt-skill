# Trust Model — Why SolHunt Cannot Drain Your Wallet

## The Problem

Every day, Solana users lose funds to malicious dApps that request token approvals, and to fake "recovery" services that steal private keys. SolHunt's architecture is designed to make it mathematically impossible for SolHunt (or any malicious actor with access to the MCP server) to steal funds.

## Architectural Guarantees

### 1. Unsigned Transactions Only

SolHunt's MCP server returns raw transaction bytes (base64-encoded). These bytes describe instructions that CLOSE token accounts and transfer a 15% fee. **SolHunt never receives, sees, or can produce a signed transaction.**

The agent or user must:
1. Receive the unsigned transaction bytes
2. Sign them locally with their own private key
3. Submit to any Solana RPC

If SolHunt's servers were fully compromised, an attacker would only be able to produce unsigned transactions — they cannot forge a signature.

### 2. Instruction-Level Isolation

Every recovery transaction built by SolHunt contains **only** these instruction types:

```
CloseAccount (token account) × N    ← recovers rent
Transfer (SOL) × 1                  ← 15% fee to SolHunt
```

There is no `Transfer` instruction from the user's main wallet, no `Delegate` instruction, and no arbitrary program invocation. The instruction set is closed.

### 3. Atomic Bundling

The 15% fee and the `CloseAccount` instructions are in the **same transaction**. This means:
- The fee is only paid if recovery succeeds
- There is no "recovery now, pay later" scenario
- Front-running the recovery is not possible

### 4. No Persistent Authority

SolHunt never requests:
- `Approve` or `Delegate` calls
- Token approval extensions
- Multisig setup
- Any persistent on-chain permission

The only on-chain action SolHunt ever triggers is closing already-empty token accounts.

## How to Verify

Audit the MCP server source at:
`https://github.com/shieldspprt/solhunt-recovery/blob/master/netlify/functions/mcp.ts`

Every transaction construction follows the pattern:
1. Build `CloseAccount` instructions for accounts with balance == 0
2. Add a `Transfer` instruction for exactly 15% of recovered SOL to the fee address
3. Serialize and return — never sign

## cURL Test (No Keys Required)

```bash
# Get a wallet report — no auth needed
curl -X POST https://solhunt.dev/.netlify/functions/mcp \
  -H "Content-Type: application/json" \
  -d '{"tool": "get_wallet_report", "arguments": {"wallet_address": "7nxJ4Nt...'}}'
```

## Official References

- [SolHunt GitHub](https://github.com/shieldspprt/solhunt-recovery)
- [SPL Token CloseAccount](https://github.com/solana-labs/solana-program-library/tree/master/token/program#closeaccount)
- [Solana Transaction Structure](https://docs.solana.com/developing/programming-model/transactions)
