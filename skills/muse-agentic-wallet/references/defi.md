# DeFi Commands

Query DeFi protocols, investment opportunities, and the user's DeFi
positions. Execute DeFi transactions (deposit, redeem, LP, claim).

There is **no hosted DeFi catalog** in the muse wallet — no
`protocol-list`, no `investment-list`, no backend position feed. Every DeFi
action is a **direct contract call to the protocol** (the
[external-sign.md](external-sign.md) two-step flow) plus **position reads via
`eth_call`** against the protocol's own contracts.

> **Preview before executing**: before running any state-changing DeFi call,
> simulate it (`tx.simulate()`), show the user the estimated fee, balance
> changes, and any warnings. Only proceed after the user confirms, unless the
> user explicitly asks to skip confirmation.

## Post-trade verification

After a successful state-changing call, on-chain data is not immediately
reflected in every read. Follow these rules:

1. **Set expectations on submit.** A returned `txHash` only means the
   transaction was **submitted** — it may still be pending in the mempool,
   and it could still fail on-chain (revert, run out of gas, be dropped).
   Tell the user something like "submitted; not yet confirmed on-chain. Data
   may take a moment to update, ask any time to check." Do NOT claim the
   deposit / redeem / claim is complete, and do NOT call any verification
   query immediately after — it will just show stale data.
2. **First verification = normal read.** When the user asks to check, read
   the position from the protocol contract (`eth_call`) — no refresh flag
   exists here; a fresh `eth_call` is already a fresh read.
3. **Never claim "transaction failed" based on a stale read.** A `txHash` is
   only proof the tx was **submitted**. If position reads still show no
   change, the answer is "syncing — check the block explorer for the tx's
   on-chain status", not "it failed".

## Display Rules (apply to ALL DeFi reads)

When summarizing or rendering any field below to the user, format to
**exactly 2 decimal places** (round half-up). Do not drop trailing zeros, do
not switch to scientific notation, do not omit the unit.

| Field kind | Render as |
|---|---|
| APY / APR (pre-formatted) | Quote the protocol's display string verbatim (e.g. `"4.32%"`) |
| TVL (USD) | `$1,187,356,864.00` |
| USD values | `$57.40` |
| Health factor | `1.85` |
| Ratios / shares (0–1) | `50.00%` |
| Slippage (basis points) | `1.00%` |

- **APY is base APY only** — it does NOT include earn campaign / promotional
  APY. Numbers may differ from the protocol's website. Mention this if asked.
- **At the protocol level, APY is the MAX APY** across all the protocol's
  pools — not every pool yields that rate. Call it "max APY" or "up to X%".
- Token amounts keep their original precision; do not force them to 2 decimals.

## Operations (all via direct contract calls)

| Operation | Muse path |
|---|---|
| List protocols / investments | No catalog. The user names the protocol; you read its contracts (TVL, APY, pools) via `eth_call` against the protocol's own addresses. Never invent a protocol address. |
| Position query | Read the protocol's position/balance functions via `eth_call` (e.g. `balanceOf`, `earned`, `healthFactor`). Group by protocol; highlight a lending health factor < 1.5 as a warning — don't auto-act. |
| Deposit / stake / supply | Contract call to the protocol's deposit/stake function (preview → confirm → `prepare_and_send` → `wait` → verify position increased). If an approval is needed first, it is a separate confirmed transaction — and it gets revoked after, per the standing rule. |
| Redeem / unstake / withdraw | Contract call to the protocol's redeem function. Some protocols have a lock-up: if the redeem returns a delay (e.g. 7 days), tell the user the funds mature after that delay and a second `claim` call is needed later to actually withdraw. |
| LP add / remove | Contract call to the router/position manager. **You name one token; the wallet must already hold BOTH pool tokens in the correct ratio** — the call does not swap the input into the paired token. For V3/V4, the position is an NFT (`nftId`, `tickLower`, `tickUpper`); only convert ticks to prices if the user explicitly asks. |
| Claim (fees / rewards / matured redemptions) | Contract call to the protocol's claim function, with the claim type (reward / LP fee / redemption) and the position/redemption id from the position read. |

## Worked example — staking (Zayro, Giwa Sepolia)

Staking contract `0x0ebc5Da986cE4Af4Bbdc754DBcC990B8105601eE`, ZAYRO token
`0xb075A37c43FBfD45239166E3AFBf752A9A533355`. Flow used by the example driver:
`approve(staking, amount)` → wait → `stake(amount)` → wait → verify the
staked balance; later `unstake` + `claim` for rewards. After staking, verify
the allowance is back to 0 (nothing left to revoke).
