# Limit Order

Place, list, and cancel limit orders. A limit order executes automatically
once the token hits the specified trigger price.

There is **no hosted limit-order engine** in the muse wallet — no backend
holds your order and fires it for you. A limit order here is a **watcher**:
a scheduled check (cron/hook) that reads the live price and, when the
trigger is hit, executes a market swap under the rules below.

## Security Pre-Check

Before arming any limit order, complete the swap security pre-check in
[security.md §1](security.md#1-swap-security-pre-check).

## Arming a limit order (buy / sell)

A limit order is a standing instruction with four parts — record all four
before arming it:

1. **Trigger price** (USD) — the price that activates the order.
2. **Side + amounts** — buy: spend `fromTokenQty` of USDT/USDC/native to buy
   `toToken`; sell: sell `fromTokenQty` of the token for USDT/USDC/native.
3. **Slippage** — default "auto"; disclose it.
4. **Expiry** — when the order dies if the trigger is never hit (default: ask
   the user; never leave an order armed forever without saying so).

Arming means creating a watcher job (cron/hook) that:

- reads the live price on a fixed cadence,
- does nothing while the price is away from the trigger,
- when the trigger is hit, runs the market-swap flow in
  [market-order.md](market-order.md) (quote → confirm? — see below) and then
  **disarms itself** (one-shot),
- logs every check so the user can audit it.

**Confirmation at execution:** a watcher that fires a swap without the user
present must have been explicitly authorized when armed ("execute the swap
automatically when the trigger hits, up to $X, slippage Y"). If the user did
not authorize automatic execution, the watcher only **notifies** the user
that the trigger hit and waits for their go-ahead. Never auto-execute a trade
the user did not authorize to run unattended.

## Listing / checking a limit order

A limit order's state lives in the watcher job + its log, not on a backend:

| Status | Meaning |
|---|---|
| `WORKING` | Armed; waiting for the trigger price. The expected resting state — not "still processing". |
| `TRIGGERED` | Price condition met; the swap is being executed (or the user was notified). |
| `PENDING` | Swap submitted on-chain, awaiting receipt. |
| `FINISHED` | Swap receipt `status 0x1`; report the actual filled amount. |
| `FAILED` | Swap receipt `status 0x0` or the swap could not be built. Say so plainly. |
| `EXPIRED` | Trigger never hit before expiry. The order is dead — **tell the user**, do not leave it looking like it is still waiting. |
| `CANCELED` | Canceled by the user. State it explicitly. |

## Canceling a limit order

Cancel = disable/remove the watcher job (cron/hook) and record the
cancellation in the log. An order whose swap already executed cannot be
canceled — the swap is on-chain and irreversible.

## ⚠️ Mandatory: arming is NOT a fill

Arming a limit order means nothing was bought or sold. Report a placement as
a placement: state the trigger price, the side, and the expiry. Never say the
user bought or sold anything at this point. `WORKING` can last for days —
tell the user how to check it later (the watcher log / job name), do not poll
it to a terminal state, and do not imply it is about to execute.

## Conditional instructions — never silently downgrade

See SKILL.md step 8. If the trigger cannot be armed (no price feed, no
watcher available, the asset has no liquid route): **do NOT fall back to an
immediate market swap.** Stop, tell the user what failed, and offer the
explicit choices: (a) hold and act when the price actually reaches the target,
(b) execute now at the *current* market price — state it and how far it is
from their target, or (c) pick a different asset. Wait for the user to choose.
