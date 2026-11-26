# x402 Payment

Pay for an x402 HTTP resource by previewing payment options, then signing
one. x402 v2 only; supported networks are BSC, Base, and Solana (Ethereum is
not supported by the x402 v2 flow used here).

There is no hosted x402 signer — the signature is produced locally by the
muse wallet (EIP-712), and the payment is settled by the merchant from that
signature.

## `x402-payment preview`

Preview payment options from a merchant's `PaymentRequired` response
(the base64 `PAYMENT-REQUIRED` header value or its raw JSON).

Parse the payload yourself and build the options table:

| Field | Meaning |
|---|---|
| `paymentId` | UUID identifying this payment |
| `options[].index` | 1-based selector to pass to sign (first option is `1`, not `0`) |
| `options[].status` | `READY_TO_SIGN` / `ACTION_REQUIRED` / `NOT_SIGNABLE` — drives the next step |
| `options[].reasons` | Why the option isn't ready (`INSUFFICIENT_BALANCE`, `INVALID_ACCEPT_STRUCTURE`, `UNSUPPORTED_NETWORK`, `UNSUPPORTED_SCHEME`, `UNSUPPORTED_METHOD`, `NO_WALLET_ON_CHAIN`, `BLOCKED_BY_SECURITY_CHECK`, `BLOCKED_DAILY_LIMIT_REACHED`) |
| `options[].scheme` | x402 scheme, e.g. `exact` |
| `options[].assetTransferMethod` | e.g. `eip3009`, `permit2`, `spl-transfer` |
| `options[].binanceChainId` / chain | `56` (BSC), `8453` (Base), `CT_501` (Solana) |
| `options[].tokenAddress` / `tokenSymbol` / `amount` / `amountUsd` | What is paid |
| `options[].payTo` | Merchant address |
| `options[].currentBalance` / `currentBalanceUsd` | Your balance of that token on that chain (read via `chain.token_balance()`) |
| `options[].needApproveFirst` | Whether a Permit2 approve tx is required before this payment. If true, the approve is a separate on-chain transaction the user must confirm first — and it gets revoked after, per the standing rule. |

- The options are pre-sorted; earlier options are recommended over later ones.
- Only `READY_TO_SIGN` options can be signed directly. `ACTION_REQUIRED`
  options can be signed after the user takes a remediation action;
  `NOT_SIGNABLE` options cannot be signed.

## `x402-payment sign`

Sign the selected payment option and return the replay header.

The signature is an **EIP-712 typed-data signature** (EIP-3009
`transferWithAuthorization` or Permit2, per `assetTransferMethod`):

```python
# Build the typed data from the selected accept entry:
# domain: name, version, chainId, verifyingContract (the token contract)
# message: from, to, value, validAfter, validBefore, nonce  (EIP-3009)
domain_sep = core.keccak256(abi_encode(domain_type_hash, name_hash, version_hash, chain_id, token))
struct_hash = core.keccak256(abi_encode(type_hash, *fields))
digest = core.keccak256(b"\x19\x01" + domain_sep + struct_hash)
r, s, parity = core.sign_digest(priv, digest)
```

- Show the user the full payment (token, amount, payTo, expiry) before
  signing. A signature is a payment authorization — it moves funds once the
  merchant settles it.
- The result is the replay header: `PAYMENT-SIGNATURE: <base64 signature
  payload>`, valid until `signatureExpiresAt`.

### Replaying the Request

- Replay the original HTTP request with the `PAYMENT-SIGNATURE` header attached.
- If an approve tx was dispatched (Permit2), wait for it to confirm (receipt
  `status == 0x1`) before replaying.
- A signature can only be used once and is valid until `signatureExpiresAt`.
  If exceeded, restart from preview.
- The `PAYMENT-RESPONSE` response header (base64 JSON) carries settlement
  metadata, including the `txHash`.

## Guardrails

1. **Confirm before signing.** Confirm with the user before signing and only
   proceed once they've consented, unless the user explicitly asks to skip
   confirmation.
2. **Confirm before switching option.** Confirm before switching to a
   different payment option, network, or token.
3. **Confirm before changing resource.** Only pay for a different resource or
   service when the user has named it — never silently substitute; if the
   current one fails, report the error and let the user decide.
4. **Confirm before retrying more than once.** If replaying the request fails
   due to a network or unknown error, retry at most once automatically. Before
   any further retries, confirm with the user.

## Limitations

- Only x402 v2 is supported.
- Only BSC, Base, and Solana are supported. Ethereum is not supported.
- Solana signing is not covered by `muse/tx.py` (EVM-only) — Solana x402
  needs a separate signing path.
