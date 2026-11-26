# Wallet Settings — limits and your policy

There are **no hosted settings** to read or change: no daily limit, no
`abnormalTxnHandling`, no `tradeAllTokens`, no prediction/DeFi/x402 quotas,
no Developer Mode toggle, no session expiry enforced by a backend. The
wallet is self-custody; the limits are your own rules, enforced by
procedure in this skill and in your standing memory.

## Your standing rules (do not weaken them)

- **On-chain transactions need explicit user confirmation.** Every send,
  swap, claim, mint, stake, approval, and revoke is confirmed by the user in
  this conversation before signing — unless the user explicitly said "without
  asking" for that specific action. A standing "just do it" for safe
  non-on-chain work never covers on-chain work.
- **Revoke approvals promptly after every transaction.** Any approval granted
  for a transaction is set back to 0 as soon as the transaction is done. See
  [approvals.md](approvals.md). A transaction that grants no approval needs
  no revoke — say so explicitly.
- **Simulate before broadcast.** Every state-changing call is dry-run through
  `eth_call` first. A reverting simulation means: do not sign.
- **Verify the provider.** `eth_chainId` is checked against the registry
  before any nonce is touched.
- **Credentials are transient.** The vault passphrase is supplied per session
  and never stored; private keys are never printed, logged, or put on a
  command line.
- **Standing rules are per-user.** A rule like "free mints only — never buy
  paid units, never post to X" is an example of a standing rule: it binds
  that user's wallet until they change it explicitly. Record such rules in
  memory and enforce them in the confirmation step.

## What to report when the user asks "what are my settings?"

There is no settings endpoint to call. Answer from the rules above, plus the
live facts you can read:

- Wallet addresses (derived): `python -m muse address --chain <c> --count N`
- Native + token balances: `chain.balance()` / `chain.token_balance()`
- Pending-nonce state: `pending` vs `latest` counts (see
  [wallet-view.md](wallet-view.md))
- Open approvals: `allowance(owner, spender)` reads (see
  [approvals.md](approvals.md))

If the user wants a limit changed (e.g. "never spend more than $X without
asking"), that is a standing rule to record in memory — not a setting to
toggle on a backend. Record it, repeat it back, and enforce it in this skill's
confirmation step.
