# Security Reference

Security pre-check procedures for swap operations.

---

## §1 Swap Security Pre-Check

> **Applies to**: market swaps in [market-order.md](market-order.md) and
> limit buy/sell in [limit-order.md](limit-order.md)

Before constructing any swap command, perform the following steps:

### Step 1: Identify the target token

- For a market swap and a limit buy: the target is the token being bought.
- For a limit sell: the target is the token being received (already
  restricted to BNB/USDT/USDC, so skip to Step 4).
- If the target is a known trusted asset from the **Common Token Addresses**
  table in SKILL.md → skip to Step 4.

### Step 2: Verify the token contract on-chain

There is no hosted token-audit skill here. Verify directly, with reads:

1. **Bytecode exists**: `eth_getCode(token)` is non-empty. An EOA posing as
   a token is an immediate stop.
2. **Metadata reads back**: `name()`, `symbol()`, `decimals()` via `eth_call`
   — and the symbol matches what the user asked for. A mismatch (e.g. user
   said "USDT" but the contract says something else) is a stop.
3. **Total supply / holder sanity**: `totalSupply()` reads back; if a public
   explorer shows the deployer still holds nearly all supply or the contract
   is unverified with no renounced ownership, say so.
4. **Honeypot signals you can check by simulation**: simulate a small
   sell (`getAmountsOut` + a dry-run of the router call) — a simulation that
   reverts on the sell side while the buy side simulates fine is a strong
   honeypot signal. Report it, do not proceed.
5. **Tax check**: read the contract's buy/sell tax if it exposes one; a tax
   above 5% is a warning, above 10% is high risk. If the tax cannot be read,
   say so — do not assume zero.

### Step 3: Interpret results and present to user

Present a security summary: contract address (full), name/symbol/decimals as
read on-chain, whether the contract is verified on the explorer, any risk
items found (tax, honeypot signal, ownership concentration), and the gaps
(what could not be verified). **Let the user decide whether to proceed.**

### Step 4: Continue with swap flow

If the user decides to proceed, continue with the standard swap flow in the
relevant reference file (confirm slippage → build → simulate → execute).

### Unavailable Handling

| Scenario | Action |
|---|---|
| A verification read fails (RPC / timeout) | Warn: "Token verification is temporarily unavailable." Require explicit user acknowledgment before proceeding. |
| Contract has no bytecode / metadata mismatch | Stop. Do not proceed. Tell the user the address does not look like the token they named. |
| Simulation reverts on the sell side | Treat as a honeypot signal. Tell the user plainly and require explicit acknowledgment; recommend not proceeding. |
| Tax or ownership data cannot be read | Say so explicitly. Require explicit user acknowledgment before proceeding without it. |

Never silently skip. The user must always be informed before proceeding.
