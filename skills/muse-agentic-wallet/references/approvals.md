# Token Approvals

Manage token approvals: view, filter, and revoke EVM token authorizations.
There is no hosted approvals list — allowances are read directly from the
token contract via `eth_call`.

## `approvals list` — `allowance(owner, spender)`

```python
# allowance(address owner, address spender) -> uint256 ; selector 0xdd62ed3e
data = "0xdd62ed3e" + owner[2:].lower().rjust(64, "0") + spender[2:].lower().rjust(64, "0")
raw = chain.eth_call(token_contract, data)
allowance = int.from_bytes(raw, "big")   # base units; 2**256-1 == unlimited
```

- To list approvals you must know the (token, spender) pairs to check —
  there is no "show me all my approvals" query on-chain. Start from the
  pairs you have actually used (DEX routers, staking contracts,
  Permit2 `0x000000000022D473030F116dDEE9F6B43aC78BA3`), plus the tx log.
- Report each pair as: token symbol + full contract address, spender + full
  address, amount (human units, or "unlimited" for `2**256-1`), and the chain.
- A Permit2 approval is an allowance granted to the Permit2 contract, not to
  the end spender — say so when you report it, and check the Permit2
  allowance map for the actual spender when it matters.

## `approvals detail`

For one (token, spender) pair, report: token symbol/decimals/balance/price
(if known), spender, allowance amount, and whether it is unlimited. If the
pair was ever used, the tx log / explorer shows the approval transaction(s);
quote the txHash from the log, never from memory.

## `approvals revoke`

Revoke = set the allowance to 0 with an `approve(spender, 0)` contract call:

```python
# approve(address spender, uint256 amount) ; selector 0x095ea7b3, amount = 0
data = "0x095ea7b3" + spender[2:].lower().rjust(64, "0") + "0" * 64
ok, reason = tx.simulate(chain, address, token_contract, 0, data)
report = tx.prepare_and_send(chain, address, priv, token_contract, 0, data=data)
receipt = tx.wait(chain, report["tx_hash"])
# verify: allowance(...) == 0 afterwards
```

- Revoke always runs at `normal` speed unless the user says otherwise.
- **A txHash does not mean the revoke is confirmed.** The approval remains
  effective until the transaction is confirmed on-chain (receipt
  `status == 0x1`). Tell the user: submitted / pending, track it with the
  explorer link, and wait before considering the revocation complete. Then
  verify with a fresh `allowance()` read.

## Standing rule

**Revoke promptly after every transaction that granted an approval.** This is
a standing rule, not a suggestion. After a swap/stake/deposit that
needed an approval, the revoke is part of the same task — do it in the same
run, a few moments after the main transaction confirms, and report both tx
hashes. A transaction that granted no approval (e.g. a pure `claimFree()`
mint) needs no revoke — say so explicitly.
