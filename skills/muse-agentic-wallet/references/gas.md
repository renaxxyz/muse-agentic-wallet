# Gas Price

## `wallet gas-price` — fee levels

```python
chain.gas_price()     # current network gas price, wei
chain.priority_fee()  # eth_maxPriorityFeePerGas, wei
chain.base_fee()      # latest block baseFeePerGas, wei (0 on BSC)
```

Speed tiers in `tx.SPEED` (multiplier on priority fee + base-fee headroom):

| Speed | Priority mult | Base-fee headroom | Use when |
|---|---|---|---|
| `slow` | 0.8× | 1.15× | No rush, save on fees |
| `normal` | 1.0× | 1.35× | Default |
| `fast` | 1.6× | 1.6× | Faster confirmation wanted |
| `ape` | 3.0× | 2.2× | Time-critical (FCFS mint, etc.) |

```bash
python -m muse send --chain bsc --to 0x... --amount 0.02 --speed fast --dry-run
```

## Units

| Field | Unit |
|---|---|
| `gasPrice`, `maxFeePerGas`, `maxPriorityFeePerGas`, `baseFeePerGas` | wei (divide by 1e9 for gwei) |

## Important Notes

- **Gas price moves with network conditions.** What you report is a
  snapshot, not a locked quote. Never present it as the final transaction
  cost — the actual fee is settled when the transaction is built and
  confirmed.
- **Do not multiply a quoted gas price by a guessed gas limit.** If the user
  wants an actual cost, run the real estimate: `chain.estimate_gas(tx)` (×1.2
  headroom in `prepare_and_send`) or a `--dry-run` send, which reports the
  fee.
- **BSC is special**: `baseFeePerGas` is 0. EIP-1559 type-2 transactions work,
  but a legacy type-0 transaction (`build_legacy` / `sign_legacy` with a plain
  `gasPrice`) is the natural fit and what the DOTGENT mint used
  (0.1 gwei, 79,992 gas, ~0.000008 BNB).
- **MEDIUM / `normal` is the default** for sends and swaps. Suggest `fast`
  only when the user wants faster confirmation (e.g. an FCFS mint), and `slow`
  when they want to save on fees. Revokes use `normal` unless the user says
  otherwise.

## When to Use

Run a gas query when the user **asks** about gas — "how much is gas on BSC
right now?", "what's the difference between the fee tiers?", "is now a good
time to trade, gas-wise?"

Do **not** run it before every transaction. Transaction commands already pick
the right level on their own, so a pre-flight gas query just adds a round
trip without changing the outcome.
