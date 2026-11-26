# Agent ID Claim Status

**No equivalent in the muse wallet — Binance-only feature. Do not invent one.**

The source skill's `wallet agent-id` checks whether a Binance Agentic Wallet
account has claimed an Agent ID — a `.ai` space id domain (e.g. `alice.ai`)
issued by Binance. The muse wallet has no Agent ID, no `.ai` domain registry,
and no claim flow. There is nothing to check, and a "not claimed" answer
would be meaningless.

## What to do instead

- If the user asks "did I claim my .ai name / Agent ID?": answer plainly —
  Agent IDs belong to Binance's Agentic Wallet; the muse wallet does not have
  one. If they hold a `.ai` domain, it lives in their Binance wallet, not here.
- If the user wants a human-readable name for their muse wallet: the
  equivalents are **ENS (`.eth`)** and **SpaceID (`.bnb`)** names, which they
  can own and point at their muse address. Resolving such a name to this
  wallet's address is covered in [domain-resolve.md](domain-resolve.md).
- If a token sale or feature requires an Agent ID as an eligibility check:
  say so explicitly — the muse wallet cannot pass that check, and do not
  claim otherwise. Offer the Binance path only if the user asks for it.

## Do not

- Do not run a fake "agent-id" check that always returns "not claimed".
- Do not tell the user to claim an Agent ID in an app as if it applied to
  this wallet.
- Do not use Agent ID status to infer eligibility for anything on the muse
  wallet.
