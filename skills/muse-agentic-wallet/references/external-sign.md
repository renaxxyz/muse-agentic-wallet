# External Sign

Preview and execute user-built contract calls and message signatures. These
operations are intentionally two-step: first preview (simulate + inspect the
parsed operation), then execute only after explicit user confirmation.

There is no Developer Mode to enable and no App to open. The gate is the
preview itself plus the user's explicit confirmation in this conversation.

---

## Safety Flow

Always follow this flow:

1. Run the preview: build the exact transaction/message, simulate it
   (`eth_call`), decode the calldata, and compute the cost.
2. Show the user the parsed transaction or parsed message, the risks, and the
   authority changes (approvals granted, ownership changed, funds moved).
3. Ask for explicit confirmation before executing.
4. Execute only with the exact parameters from the successful preview. If
   preview/simulation fails, do not attempt execute.
5. After broadcast, poll the receipt to a terminal state (`tx.wait()`) and
   verify the on-chain effect (e.g. `balanceOf` changed) before reporting
   success.

## `contract-call preview`

Preview a contract call: simulate it and get the cost and effects.

### Build

```python
chain = resolve(load_registry(), "<chain>")
chain.verify_chain_id()
ok, reason = tx.simulate(chain, address, to_contract, value_wei, calldata)
# ok == False -> STOP. Report the revert reason; do not sign.
gas = int(chain.estimate_gas({"from": address, "to": to_contract,
                              "value": hex(value_wei), "data": calldata}) * 1.2)
cost = gas * chain.gas_price()
```

### Show the user (before asking to confirm)

- **To**: full contract address — and what the contract is (verified
  name/symbol/decimals read back via `eth_call`, or "unverified — bytecode
  present / no bytecode" if it cannot be verified).
- **Function**: decoded from the calldata selector (e.g. `claimFree()`,
  `approve(address,uint256)`, `swapExactTokensForTokens(...)`). If the
  selector is unknown, say so — never guess the function.
- **Value**: in wei **and** human units.
- **Gas**: estimate + gas price + total cost in native currency.
- **Effects**: what balances/allowances/ownership will change (from the
  simulation + the function's known semantics).
- **Risks**: high-risk target address, unlimited approval, funds leaving the
  wallet, irreversible state change.

### Worked example — free mint (DOTGENT, BSC, verified live)

Contract `0xA85C24B8f5938aCebE88f8353E071f432EbA0620` (BSC), function
`claimFree()` (selector `0xf366afc9`), value 0. Pre-checks read on-chain
before signing: `freeSold()` = 601/1000, `freeClaimed(claimingWallet)` =
false, `name()` = "Dotgent", `symbol()` = "DGTG". Legacy tx (BSC baseFee is
0), gas 80,892 est. → 97,070 with headroom, 0.1 gwei. Result (example
transaction): `0x664ad69a6bc644b73beb556948c0841f1d659ddf13f9ebace376cf20d6133746`,
block 135772781, status `0x1`, verified `balanceOf(claimingWallet)` =
1,000,000 DGTG. No approval was granted, so no revoke was needed. Example
driver pattern: a small script that unlocks the vault, checks the claim
state, simulates, builds a legacy tx, signs, broadcasts, waits for the
receipt, and verifies the balance.

## `contract-call execute`

Execute the previewed call — only after explicit confirmation, with the exact
previewed parameters:

```python
report = tx.prepare_and_send(chain, address, priv, to_contract, value_wei,
                             data=calldata, speed="normal")
receipt = tx.wait(chain, report["tx_hash"])
# receipt["status"] == 0x1 -> confirmed; 0x0 -> reverted, report as failed
```

- `BROADCASTED` (a txHash returned) means signed and submitted — **not**
  confirmed. Use the receipt to decide.
- After any approval granted by the call, revoke it promptly (see
  [approvals.md](approvals.md)).

## `sign-message preview` / `execute` / `result` / `history`

### Personal sign (EIP-191)

```python
digest = core.keccak256(b"\x19Ethereum Signed Message:\n" + str(len(msg)).encode() + msg.encode())
r, s, parity = core.sign_digest(priv, digest)
```

### EIP-712 typed data

```python
domain_sep = core.keccak256(abi_encode(domain_type_hash, name_hash, version_hash, chain_id, verifying_contract))
struct_hash = core.keccak256(abi_encode(type_hash, *fields))
digest = core.keccak256(b"\x19\x01" + domain_sep + struct_hash)
r, s, parity = core.sign_digest(priv, digest)
```

Working pattern: `~/workspace/zayro/zayro_driver.py` (session-key
registration, SIWS-style messages).

### Rules

- Show the user the **full message** before signing — for Permit/Permit2,
  show the authority being granted: `spender`, `sigDeadline`, token, amount,
  expiration. Never sign a message the user has not seen.
- A signature is not a transaction: it grants authority off-chain and can be
  replayed until its deadline. Say so when the message is a permit.
- **Result**: a completed signature returns `(r, s, v)` / the 65-byte
  signature. Keep a local signature log (time, domain, verifying contract,
  spender, deadline) — there is no hosted history.
- **History**: read the local log. There is no backend query.
