# Wallet View Commands

Read-only views of the wallet. None of these sign anything.

## `wallet status` — is the wallet ready?

There is no hosted session to query. Status is three local checks:

```python
chain = resolve(load_registry(), "<chain>")
chain.verify_chain_id()      # provider really on this chain?
chain.block_number()         # RPC live and current?
address, _ = vault.derive("<chain>", 0)   # vault unlocked, address matches?
```

| State | Meaning |
|---|---|
| All three pass | Ready. Proceed. |
| Vault locked / no passphrase | Not signed in — ask for the passphrase (signing only; reads still work). |
| `verify_chain_id` raises | Provider on the wrong chain — stop. |
| RPC fails / block stale | Provider down — stop, do not retry blindly. |

## `wallet chains` — supported chains

The registry in `muse/chain.py`:

| Key | Chain ID | Native | Explorer |
|---|---|---|---|
| `ethereum` | 1 | ETH | etherscan.io |
| `base` | 8453 | ETH | basescan.org |
| `arbitrum` | 42161 | ETH | arbiscan.io |
| `optimism` | 10 | ETH | optimistic.etherscan.io |
| `polygon` | 137 | POL | polygonscan.com |
| `bsc` | 56 | BNB | bscscan.com |
| `sepolia` | 11155111 | ETH | sepolia.etherscan.io |

Custom chains (e.g. Giwa Sepolia, chainId 91342) are **not** in the registry —
they live in drivers with their own RPC + chain-id check (see
[external-sign.md](external-sign.md)). Never add a chain to the registry
silently; if a task needs one, build it in the driver and verify `eth_chainId`
before use.

## `wallet address` — get addresses

```bash
python -m muse address --chain bsc --count 5
```

```python
for i in range(5):
    address, _ = vault.derive("bsc", i)
```

Show full addresses, never truncated — a truncated address cannot be verified.

## `wallet balance` — token balances

```bash
python -m muse balances --chain bsc --count 5
```

```python
chain.balance(address)                 # native, wei
chain.token_balance(token, address)    # ERC-20, base units
chain.token_meta(token)                # name, symbol, decimals
```

- Tokens with a zero balance are not interesting; say so rather than listing
  pages of zeros.
- A token with no price is **unknown price**, not worthless. Never present a
  missing price as `$0.00` value.
- Always show the token contract address next to the symbol.

## `wallet tx-history` — transaction history

There is **no hosted history API**. History is read per transaction:

```python
receipt = chain.receipt(tx_hash)        # None = not mined yet
confirmations = chain.confirmations(tx_hash)
chain.explorer_tx(tx_hash)              # explorer link
```

- Keep your own tx log: after every broadcast, record `txHash`, chain, to,
  value, data, and the receipt once it lands. The log is the history.
- To look up a past transaction, use the explorer (the link above) or the
  receipt. Never claim a tx is confirmed from the broadcast response alone.
- For a pending transaction, see [speedup-cancel.md](speedup-cancel.md).

## `wallet tx-lock` — is the wallet blocked from sending?

```python
pending = chain.call("eth_getTransactionCount", [address, "pending"])
latest  = chain.call("eth_getTransactionCount", [address, "latest"])
```

| State | Meaning |
|---|---|
| `pending == latest` | Free to send. |
| `pending > latest` | A transaction is pending on-chain (or a replacement is waiting). Nothing to do but wait, speed it up, or cancel it — see [speedup-cancel.md](speedup-cancel.md). Do not start a new transaction and assume it will queue cleanly; nonces are sequential. |

## `wallet gas-price` — fee levels

See [gas.md](gas.md). In short: `chain.gas_price()`,
`chain.priority_fee()`, `chain.base_fee()`; speeds `slow` / `normal` /
`fast` / `ape` in `tx.SPEED`.
