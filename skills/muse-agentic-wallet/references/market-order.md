# Market Order (DEX Swap)

Swap tokens on-chain at the current market price, get quotes, and check
swap status. There is **no hosted swap engine** — a swap is a contract call
to a DEX router (e.g. PancakeSwap / Uniswap), built, simulated, confirmed,
signed, and broadcast by the muse wallet itself.

## Security Pre-Check

Before any swap, complete the swap security pre-check in
[security.md §1](security.md#1-swap-security-pre-check). This includes
verifying the target token contract on-chain and presenting the security
summary to the user.

## Fees

A swap pays: the DEX fee (inside the route), the gas fee, and any router
fee. There is no Binance Web3 Wallet trading fee — but the router you call
may charge its own fee. Read the router's fee from the quote before confirming.

## Quote (no trade)

Get a swap quote without executing. Use the router's `getAmountsOut` via
`eth_call` (read-only, no signature needed):

```python
# PancakeSwap V2 router (BSC): 0x10ED43C718714eb63d5aA57B78B54704E256024E
# getAmountsOut(uint256 amountIn, address[] path) -> uint256[] amounts
# Build the calldata: selector 0xd06ca61f + amountIn (32B) + offset + path
amounts = chain.eth_call(router, get_amounts_out_data)
out = int.from_bytes(amounts[64:96], "big")   # amounts[1], base units
```

Show the user the expected output, the price impact, and the slippage
default ("auto") before they commit. A quote is a snapshot, not a locked
price — the actual fill is settled when the transaction is built and
confirmed.

## Swap (market order)

Build the router calldata (`swapExactTokensForTokens` / `swapExactETHForTokens`
/ `swapExactTokensForETH`, with `amountOutMin` from the quote × (1 − slippage)
and a `deadline`), then run the external-sign two-step flow:

1. **Preview** — `tx.simulate(chain, address, router, value, calldata)`.
   Show the user: from-token, to-token, amount in, expected amount out,
   `amountOutMin`, slippage, gas estimate, and the router address.
2. **Confirm** — explicit user confirmation (see SKILL.md "Build the Command").
3. **Execute** — `tx.prepare_and_send(chain, address, priv, router, value,
   data=calldata, speed="normal")`, then `tx.wait(chain, tx_hash)`.

If the swap needs a token approval first (`allowance < amountIn`), that is a
**separate transaction** the user must confirm first — and the standing rule
applies: **revoke the approval promptly after the swap** (see
[approvals.md](approvals.md)). Prefer an exact-amount approval over an
unlimited one.

### ⚠️ Mandatory: a txHash is NOT a completed swap — verify to a terminal state before reporting success

A broadcast response with a `txHash` means the swap has only been
**submitted**. It does **NOT** mean the swap executed. The transaction can
still revert on-chain (slippage, insufficient liquidity, routing).

**You MUST do this after every swap — do not skip it:**
1. **Do NOT tell the user the swap succeeded yet.** At most say it has been
   *submitted* and you are confirming.
2. **Poll `tx.wait(chain, tx_hash)` until the receipt lands.** Check
   `receipt["status"]`: `0x1` = success, `0x0` = reverted.
3. **Report based on the receipt, not on the broadcast:**
   - `status 0x1` → report success, include `txHash`, block number, and the
     actual received amount (read `balanceOf` after the swap — do not trust
     the quote). If the received amount is materially below the quote, say so.
   - `status 0x0` → **tell the user it failed**; do not present it as success
     or leave it ambiguous. Suggest retrying with higher slippage or a
     different route if appropriate.
   - Still pending after a reasonable wait (~30 s on BSC, longer on Ethereum)
     → tell the user it is *still processing*, not that it succeeded; keep an
     eye on it or let them re-check with the explorer link.

### Swap status

| Status | Meaning |
|---|---|
| `PENDING` (no receipt) | Being processed on-chain |
| Receipt `status 0x1` | Executed successfully |
| Receipt `status 0x0` | Execution reverted |
| No receipt after timeout | Unknown — check the explorer; do not claim success or failure |
