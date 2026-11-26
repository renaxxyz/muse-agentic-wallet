# Preflight Checks

At the start of each wallet task, run these checks in order. Stop at the
first failure and tell the user what is missing — do not keep reading large
rule files or start building a transaction the wallet cannot execute.

---

## 1. Vault unlockable

The muse wallet needs the vault passphrase for this execution session. It is
supplied by the user in chat, used transiently via the `MUSE_PASSPHRASE` env
var, and never stored, printed, or put on a command line.

- **Passphrase available** → unlock: `Vault(path).unlock(pw)`, derive #0,
  and confirm the derived address matches the address you expect (the one
  you use as your main wallet). A mismatch means the wrong vault or the
  wrong passphrase — stop and say so.
- **Passphrase not available** → stop and ask the user for it. Read-only
  work (balances, quotes, simulations that don't need a signature) can still
  proceed without it — say so and do the read-only part.
- **Ordinary wallet (no vault)** → the private key comes the same way:
  transient env var for this run only (`WALLET_KEY` or equivalent), never
  written to disk. If neither is available, stop and ask.

## 2. Provider chain check

Before touching a nonce, verify the RPC provider is really on the target chain.

```python
chain = resolve(load_registry(), "<chain>")   # e.g. "bsc"
chain_id = chain.verify_chain_id()            # raises unless it matches the registry
```

- `verify_chain_id()` raises on mismatch → stop, report the provider's
  chain id vs the expected one. The provider is treated as hostile; a
  mismatch means the tx would go to the wrong chain.
- Also read `chain.block_number()` — a stale/failing RPC is a reason to stop,
  not to retry blindly.

## 3. Balance check

Read the native balance and, for token operations, the token balance:

```python
native = chain.balance(address)              # wei
token  = chain.token_balance(token, address)  # base units
```

- `value + gas_limit * max_fee` must be ≤ native balance, or the send will
  fail. `tx.prepare_and_send()` enforces this and raises
  `insufficient funds` — but check it yourself first so the user hears the
  number before any signing is attempted.
- If the balance is short, tell the user the exact shortfall and stop. Do not
  silently switch chains, tokens, or amounts.

## 4. Pending-nonce check

```python
pending = chain.call("eth_getTransactionCount", [address, "pending"])
latest  = chain.call("eth_getTransactionCount", [address, "latest"])
```

- `pending > latest` → a transaction is stuck in the mempool. The next tx
  cannot confirm until it resolves. Tell the user, and use
  [speedup-cancel.md](speedup-cancel.md) (speed up or cancel, same nonce)
  before starting a new transaction.
- `pending == latest` → free to send.

## 5. External-sign gate (only when the task needs it)

If the request involves a contract call or a message signature, the gate is
the **two-step preview flow** in [external-sign.md](external-sign.md) —
simulate, show the parsed transaction/message and the risks, get explicit
confirmation, then execute. There is no Developer Mode to enable and no App
to open; the confirmation happens in this conversation. Do the preflight
above first, so the user does not discover a missing passphrase or a wrong
chain only at the last step.
