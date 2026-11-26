# Prediction Markets

Browse prediction markets, query the user's positions and orders, and trade
outcome tokens.

There is **no hosted prediction engine** in the muse wallet — no
`prediction category list`, no backend order book, no hosted redeem. A
prediction market here means **Polymarket / Kalshi (or similar) via their
public APIs, plus contract calls** from the muse wallet when the market lives
on-chain. For strategy, sizing, and validation, use the
`prediction-market-strategy` skill; this file covers only the wallet
mechanics.

A prediction **market** asks a yes/no (or multi-outcome) question that will
resolve in the future. Each outcome is an ERC-1155 **outcome token**. Buying
an outcome token is betting on that outcome; if the market resolves in your
favor, the token can be redeemed for payout.

## Market / position queries

- **Categories / market list / detail / search / order book / last trade
  price**: read from the venue's public API (Polymarket Gamma / CLOB API,
  Kalshi API). A last-trade price is a **historical fill price**, not a live
  quote — it can be stale and does not reflect the current best bid/ask.
- **Positions / PnL / portfolio**: read from the venue's API for the user's
  wallet address, and cross-check on-chain (`balanceOf` of the outcome-token
  contract) when the market is on-chain. A `PENDING_CLAIM` position is a
  winning position not yet redeemed — redeem it (below).

## Trading Flow (two-step, always)

1. **Quote** — get a quote (price, amount, expected payout, fee, expiry)
   from the venue's API. The quote carries an id that the placement must
   reference.
2. **Confirm with the user** — show the quote and require an explicit "yes"
   before proceeding. Remind them to DYOR.
3. **Place** — submit the order with the quote id. Quotes expire; if
   placement fails because the quote is stale, fetch a new quote and try again.

## Placing / canceling / redeeming

- **Place**: submit the order to the venue (signed order / API call, or an
  on-chain contract call from the muse wallet). A successful submit means
  the order was **submitted**, not filled. For market orders fills are
  usually quick; limit orders rest until the trigger price is reached. Tell
  the user to verify execution before assuming the trade is complete.
- **Cancel**: cancel by order id through the venue's API. An order already
  filled cannot be canceled.
- **Redeem**: claim the payout from winning positions once their markets
  resolved — on-chain, this is a contract call (`redeem`/`claim` on the
  outcome-token / conditional-token contract) via the
  [external-sign.md](external-sign.md) two-step flow; off-chain, it is a
  venue API call. Find redeemable positions first (the `PENDING_CLAIM` set),
  then redeem by token id.

## Order status

| Status | Meaning |
|---|---|
| `PENDING` | Created but not yet submitted to the venue |
| `SUBMITTED` | Submitted, awaiting execution |
| `FILLED` | Fully filled |
| `PARTIALLY_FILLED` | Partially filled |
| `CANCELLED` | Canceled by user or system |
| `FAILED` | Failed to execute |
| `EXPIRED` | Expired (market ended) |
