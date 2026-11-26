# muse-agentic-wallet

An agentic wallet skill for the **muse wallet** — and for any ordinary
self-custody EVM wallet. Same capability surface as Binance's
`binance-agentic-wallet`, but the engine is a local vault + a small Python
library instead of Binance's hosted `baw` CLI. No Binance account. No hosted
wallet. No App approvals.

Adapted from [`binance-agentic-wallet` v1.15.0](https://github.com/binance/binance-skills-hub/tree/main/skills/binance-web3/binance-agentic-wallet)
(MIT). See [Attribution](#attribution).

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add renaxxyz/muse-agentic-wallet --skill muse-agentic-wallet
```

Or clone it and point your agent at the folder:

```bash
git clone https://github.com/renaxxyz/muse-agentic-wallet.git
# skill lives at skills/muse-agentic-wallet/SKILL.md
```

## Requirements

- **A wallet.** Either the muse wallet — a BIP-39 vault
  (`state/vault.json`, scrypt-encrypted) plus the `muse/` Python library
  (`core`, `vault`, `chain`, `tx`) and the `python -m muse` CLI — or any
  ordinary EVM wallet: a private key held transiently in the process (env
  var, never stored, never printed) and the same build → simulate → confirm
  → sign → broadcast → verify-receipt path. Every operation in this skill is
  documented as standard JSON-RPC calls, so it adapts to `web3.py`, `ethers`,
  or any other EVM library.
- **Python 3.9+** for the `muse/` library. No other runtime dependencies.
- Nothing else. No API keys, no hosted services, no Binance account.

## What it covers

| Area | How it works here |
|---|---|
| Balances, addresses, chains, history | Direct RPC reads (`eth_getBalance`, `eth_call`, receipts) |
| Send / transfer (native + ERC-20) | `python -m muse send` or `tx.prepare_and_send()` |
| DEX swaps (market orders) | Router contract call: quote via `getAmountsOut`, swap via calldata, poll receipt |
| Limit orders | Watcher (cron/hook) that fires a market swap on trigger — never a silent downgrade to immediate |
| Contract calls / message signing | Two-step preview: simulate → show risks → confirm → execute → verify receipt |
| Approvals | `allowance()` reads; revoke = `approve(spender, 0)`; revoke-after-transaction is a standing rule |
| Speed up / cancel pending tx | Same nonce + higher gas / 0-value self-send |
| Gas | `gas_price` / `priority_fee` / `base_fee`; speeds `slow`/`normal`/`fast`/`ape` |
| ENS / SpaceID domains | Resolve via registry/resolver `eth_call`s; resolve and send are separate turns |
| DeFi | Direct protocol contract calls + position reads (no hosted catalog) |
| Prediction markets | Venue APIs (Polymarket/Kalshi) + contract calls (no hosted engine) |
| x402 payments | EIP-3009 / Permit2 EIP-712 signing, replay with the signature header |
| Agent ID (`.ai`) | **No equivalent** — Binance-only; the skill says so plainly |

The skill's `SKILL.md` carries the full `baw` → muse command map, the
build-the-command rules, display rules, security policy, error handling, and
the common token address tables (BSC, Solana, Ethereum, Base, Arbitrum,
Polygon, Robinhood Chain).

## Safety rules (the core of this skill)

- **Explicit confirmation** for every on-chain action.
- **Simulate before broadcast** — a reverting `eth_call` means do not sign.
- **Verify the provider** — `eth_chainId` is checked against the registry
  before any nonce is touched.
- **Revoke approvals promptly** after every transaction that granted one.
- **A tx hash is not a confirmation** — report "submitted / pending" until
  the receipt shows `status == 0x1` and the on-chain effect is verified.
- **Credentials are transient** — the vault passphrase is supplied per
  session and never stored; private keys are never printed, logged, or put
  on a command line.
- **Never silently downgrade a conditional instruction** ("sell when it hits
  $X") to an immediate market order.

## Layout

```
skills/muse-agentic-wallet/
├── SKILL.md                  # routing table, rules, token addresses
└── references/
    ├── preflight.md          # vault / chain / balance / nonce checks
    ├── authentication.md     # vault unlock, derivation, ordinary wallets
    ├── wallet-view.md        # status, chains, addresses, balances, history
    ├── wallet-setting.md     # limits = your own standing rules
    ├── send.md               # native + ERC-20 transfers
    ├── market-order.md       # DEX swaps via router contract calls
    ├── limit-order.md        # trigger watchers (no hosted engine)
    ├── external-sign.md      # contract-call / sign-message two-step flow
    ├── gas.md                # fee levels
    ├── approvals.md          # allowance reads + revoke
    ├── security.md           # swap pre-check (on-chain token verification)
    ├── speedup-cancel.md     # same-nonce replacement
    ├── domain-resolve.md     # ENS / SpaceID resolution + transfer gate
    ├── agent-id-claim-status.md  # Binance-only; no equivalent
    ├── defi.md               # direct protocol contract calls
    ├── prediction.md         # venue APIs + contract calls
    └── x402-payment.md       # EIP-3009 / Permit2 signing
```

## Attribution

This skill is an adaptation of `binance-agentic-wallet` v1.15.0 from
[binance/binance-skills-hub](https://github.com/binance/binance-skills-hub)
(MIT, © Binance). The command routing, safety rules, display rules, and
token address tables follow the original; the engine (`baw` CLI + Binance
account + hosted wallet) is replaced throughout by the self-custody muse
wallet, and Binance-only features (Agent ID `.ai`, hosted limit orders,
hosted DeFi/prediction catalogs) are marked as having no equivalent, with
the closest working path documented instead.

## License

MIT — see [LICENSE](LICENSE).
