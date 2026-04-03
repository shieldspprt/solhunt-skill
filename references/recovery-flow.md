# Recovery Flow — Step by Step

## Overview

SolHunt's recovery flow is designed for autonomous AI agents. Each step produces an output that feeds into the next step. All transactions remain unsigned — the agent signs locally.

## Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│  1. get_wallet_report                                   │
│     Input: wallet_address                                │
│     Output: health_score, recoverable_sol, fee, grade   │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼ worth_recovering = true?
┌─────────────────────────────────────────────────────────┐
│  2. build_recovery_transaction                          │
│     Input: wallet_address, destination_wallet           │
│     Output: unsigned_transaction (base64), expires_at   │
└──────────────────────┬──────────────────────────────────┘
                       │
                       │ Agent signs locally
                       ▼
┌─────────────────────────────────────────────────────────┐
│  3. Submit to Solana RPC                                 │
│     Broadcast signed transaction                         │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  4. Confirmation                                        │
│     RPC returns tx signature → block confirmation       │
│     User receives: net_recoverable_sol                  │
└─────────────────────────────────────────────────────────┘
```

## Step 1: Get Wallet Report

```json
// Request
{
  "tool": "get_wallet_report",
  "arguments": { "wallet_address": "7nxJ4NtZPfaG7nF..." }
}

// Response
{
  "success": true,
  "data": {
    "health_score": 38,
    "grade": "D",
    "closeable_accounts": 47,
    "recoverable_sol": 0.095733,
    "fee_sol": 0.014360,
    "fee_percent": 15,
    "net_recoverable_sol": 0.081373,
    "worth_recovering": true,
    "opportunities": [
      { "type": "rent_recovery", "accounts": 47, "sol_each": 0.002039 }
    ],
    "next_step": "Call build_recovery_transaction to get unsigned transaction bytes"
  }
}
```

## Step 2: Build Recovery Transaction

```json
// Request
{
  "tool": "build_recovery_transaction",
  "arguments": {
    "wallet_address": "7nxJ4NtZPfaG7nF...",
    "destination_wallet": "7nxJ4NtZPfaG7nF...",
    "batch_number": 1
  }
}

// Response
{
  "success": true,
  "data": {
    "unsigned_transaction": "AQAAAA...",
    "accounts_to_close": ["acc1", "acc2", "acc3"],
    "total_recovery_sol": 0.006117,
    "fee_sol": 0.000918,
    "expires_at": "2026-04-04T00:35:00Z",
    "next_batch": 2,
    "remaining_batches": 45
  }
}
```

**⚠️ The `unsigned_transaction` field contains base64-encoded unsigned transaction bytes. The agent or user must:**
1. Decode base64 → transaction bytes
2. Sign with the wallet's private key
3. Submit to a Solana RPC (Helius, QuickNode, Triton, etc.)

## Step 3: Sign and Submit

Pseudocode:
```typescript
import { decode } from 'bs58';
import { connection, sendTransaction } from '@solana/web3.js';

const txBytes = Buffer.from(response.data.unsigned_transaction, 'base64');
const signedTx = await wallet.signTransaction(txBytes);
const signature = await sendTransaction(signedTx, connection);
console.log(`Recovery confirmed: ${signature}`);
```

## Batch Strategy

Wallets with many zero-balance accounts need multiple batches. Solana's block engine processes ~1500-2000 CUs per slot, and each `CloseAccount` instruction consumes ~1500 CUs. Maximum ~15 accounts per transaction.

From `get_wallet_report`:
- `closeable_accounts` = total accounts to close
- `estimated_batches` = ceil(closeable_accounts / 15)

Process batch 1, then repeat `build_recovery_transaction` with `batch_number: 2`, etc., until all accounts are closed.

## Expiry Handling

Recovery transactions contain a recent blockhash and expire after ~90 seconds. If the agent cannot sign and submit within that window, simply call `build_recovery_transaction` again to get fresh bytes.

## Security Pre-flight (Optional but Recommended)

Before recovery, run a token approval scan:

```
scan_token_approvals { wallet_address: "7nxJ4NtZPfaG7nF..." }
```

If any `HIGH` risk approvals exist, revoke them first — a malicious dApp could drain a wallet mid-recovery if it has active approval on a remaining token account.
