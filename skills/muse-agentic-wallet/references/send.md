# Send Tokens

Transfer native currency or ERC-20 tokens to a recipient address.

## `wallet send`

### CLI

```bash
# Send 0.02 BNB
python -m muse send --chain bsc --to 0x1234...5678 --amount 0.02 --dry-run
python -m muse send --chain bsc --to 0x1234...5678 --amount 0.02 --wait

# Send 10 USDT (ERC-20)
python -m muse send --chain bsc --to 0x1234...5678 --amount 10 \
  --token 0x55d398326f99059fF775485246999027B3197955 --wait

# Send everything (sweep)
python -m muse sweep --chain bsc --to 0x1234...5678 --count 10 --dry-run
```

| Flag | Meaning |
|---|---|
| `--chain` | Registry key (`bsc`, `ethereum`, `base`, …) — required |
| `--to` | Recipient address (full, checksummed) — required |
| `--amount` | Human-readable units (e.g. `0.02`, `10`) — required for `send` |
| `--token` | ERC-20 contract address; omit for native |
| `--index` | Which derived wallet to send from (default 0) |
| `--speed` | `slow` / `normal` / `fast` / `ape` (default `normal`) |
| `--gas` | Override gas limit (omit to estimate ×1.2) |
| `--dry-run` | Build + simulate, report fee/nonce/gas, do **not** broadcast |
| `--wait` | Poll the receipt after broadcast |

### Library

```python
from muse import tx
report = tx.prepare_and_send(chain, address, priv, to, amount_wei,
                             speed="normal", data="0x", dry_run=False)
# report: chain, from, to, value, gas_limit, max_fee_gwei, priority_gwei,
#         nonce, balance, simulation, tx_hash, explorer, broadcast
receipt = tx.wait(chain, report["tx_hash"])   # poll to terminal state
```

For an ERC-20 transfer, `data` is `tx.erc20_transfer_data(to, amount)` and
`amount` (value) is 0. `prepare_and_send` does the full path: balance check,
gas estimate ×1.2, `cost = value + gas*max_fee` check, simulation, sign,
broadcast. A reverting simulation raises before anything is signed.

### Important: a tx hash does not mean on-chain confirmation

A successful response with a `txHash` means the transaction has been
**broadcast** — it does **not** mean it has been confirmed or finalized.
Transactions can still fail after broadcast.

After receiving a `txHash`, always tell the user:
1. The transaction has been submitted and is **pending** confirmation.
2. They can track it with the explorer link (`chain.explorer_tx(txHash)`).
3. They should wait for on-chain confirmation (receipt `status == 0x1`)
   before considering the transfer complete.

### Recipient verification

There is no hosted address book. Before sending:

- Show the user the **full** recipient address and the amount, and get
  explicit confirmation. Never send to a truncated or guessed address.
- If the recipient came from a domain name, it must have been resolved first
  (see [domain-resolve.md](domain-resolve.md)) and confirmed in a separate
  turn — never pass a domain string as `--to`.
- If the recipient is a contract address, say so: many contracts cannot
  receive funds or return them.
