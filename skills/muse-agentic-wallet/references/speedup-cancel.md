# Speed Up & Cancel Pending Transactions

Manage pending (unconfirmed) transactions by speeding them up or cancelling
them. Both work by replacing the original transaction with a new one that
uses the **same nonce** and higher gas.

## Critical Constraint: Only the First Pending Transaction Can Be Acted On

> **Only the transaction with nonce = confirmedNonce + 1 (i.e., the first
> transaction in the pending list) can be sped up or cancelled.** You cannot
> skip ahead and act on a later pending transaction — EVM nonces must be
> processed sequentially.

If a user wants to cancel or speed up a transaction that is NOT the first in
the pending queue, inform them:
- They must first resolve (speed up or cancel) all preceding pending
  transactions in nonce order.
- Example: if pending list shows nonce 5, 6, 7 and the user wants to cancel
  nonce 7, they must first handle nonce 5, then 6, then 7.

## `wallet cancel` — replace with a 0-value self-send

Cancel a pending transaction by submitting a replacement transaction (same
nonce, higher gas) that sends 0 value to yourself.

```python
nonce = <nonce of the pending tx>
gas_price = chain.gas_price() * 2        # higher than the original
fields = tx.build_legacy(chain, nonce, address, 0, "0x", 21000, gas_price)  # or build_1559
raw = tx.sign_legacy(fields, priv, chain.chain_id)
new_hash = chain.send_raw(raw)
```

- The cancel transaction replaces the original by using the same nonce with
  higher gas. A broadcast means the cancel tx was submitted — it does **not**
  guarantee the original will be replaced. The replacement is confirmed only
  when the cancel tx is mined (receipt `status == 0x1`).
- After cancelling, verify with a fresh pending-nonce check that the original
  transaction is gone (or shows as replaced on the explorer).
- **Only the first pending transaction (confirmed nonce + 1) can be
  cancelled.** Attempting to cancel a later transaction will fail.
- Cancel executes immediately once broadcast — confirm with the user first.

## `wallet speed-up` — same payload, higher gas

Speed up a pending transaction by resubmitting it with the same payload
(to, value, data) and the same nonce, but higher gas fees.

```python
# Rebuild the ORIGINAL tx fields with the same nonce and payload,
# but gas_price (legacy) or maxFeePerGas/maxPriorityFeePerGas (1559) bumped:
# LOW bump = x1.1, HIGH bump = x1.25 (take the higher of the bump and the
# current network gas price, so the replacement is actually competitive).
fields = tx.build_legacy(chain, nonce, orig_to, orig_value, orig_data, orig_gas, bumped_gas_price)
raw = tx.sign_legacy(fields, priv, chain.chain_id)
new_hash = chain.send_raw(raw)
```

- **Only the first pending transaction can be sped up.**
- A broadcast means the speed-up tx was submitted — confirmation depends on
  network conditions. Track both hashes with `chain.receipt()` until one of
  them confirms.
- Confirm with the user first.

## Querying Pending Transactions

```python
pending = chain.call("eth_getTransactionCount", [address, "pending"])
latest  = chain.call("eth_getTransactionCount", [address, "latest"])
# pending > latest -> (pending - latest) transaction(s) stuck in the mempool
# Identify them on the explorer (pending tab for the address).
```

## Error Handling

| Situation | Meaning | Action |
|---|---|---|
| No pending transaction found | Nothing stuck | No action needed. |
| Transaction already confirmed | Already finalized | No action needed. |
| Replacement nonce not first | Not the first in the pending queue | Handle the earliest pending transaction first, in nonce order. |
| Replacement underpriced | Gas bump too low | Raise the bump (HIGH = ×1.25) and retry. |
| Chain mismatch | Wrong chain | Verify `eth_chainId` before rebuilding. |

## Workflow: Helping Users with Pending Transactions

1. **List pending transactions** (pending vs latest counts + explorer).
2. **If none:** tell the user they have no pending transactions.
3. **If some exist:** show hash, nonce, chain, and time submitted.
4. **Identify the actionable one:** only the first pending transaction
   (lowest nonce = confirmed nonce + 1) can be sped up or cancelled.
   Highlight it. If the user asks to act on a different one, explain the
   nonce-order rule.
5. **Confirm the action** with the user.
6. **Execute** (speed up or cancel as above).
7. **Report the result:** the new txHash and its status. Remind the user to
   check the explorer for confirmation.
8. **If more pending transactions remain:** repeat from step 1 for the next
   one in sequence.
