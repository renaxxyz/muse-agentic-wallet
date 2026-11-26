# Authentication

The muse wallet is self-custody. There is no hosted session, no QR code, no
App to confirm in. "Signing in" means unlocking the local vault for this
execution session; "signing out" means dropping the passphrase from the
process. Keys never leave the process.

## The vault

- Location: `~/muse-beta-1.0.0/state/vault.json` (mode 600, scrypt +
  HMAC-CTR + EtM encrypted). Never copy it, never move it, never print it.
- Created with `python -m muse new --show-words` or imported with
  `python -m muse import` from an existing mnemonic. The mnemonic and the
  passphrase are **never recorded** in chat, memory, or any file.
- Unlock (per session, transient):

```python
import os
from muse.vault import Vault
vault = Vault(os.path.expanduser("~/muse-beta-1.0.0/state/vault.json")).unlock(
    os.environ["MUSE_PASSPHRASE"])   # env only — never a literal, never argv
```

- The passphrase is re-supplied by the user in chat each execution session.
  If it is not available, read-only work can still proceed; signing cannot.
- **Sign-out**: when the run ends, the passphrase dies with the process.
  There is nothing to revoke server-side. If a long-running job needs the
  vault again later, it asks the user again — it never caches the passphrase.

## Deriving addresses

```python
address, priv = vault.derive("bsc", 0)   # wallet #0 on BSC
```

- Same mnemonic, every chain: `vault.derive(chain, index)` returns
  `(address, private_key_int)`. Index 0 is the main wallet; 1, 2… are
  further wallets from the same mnemonic.
- **Verify the derivation**: after unlocking, derive #0 and compare it to the
  address you expect as your main wallet. A mismatch = wrong vault or wrong
  passphrase. Stop.
- Solana: the same mnemonic derives a Solana address via SLIP-0010 ed25519
  (`m/44'/501'/0'/0'`). Note: `muse/tx.py` is EVM-only — Solana signing and
  sending need a separate path; do not pretend an EVM tx works there.

## Ordinary wallets (no vault)

Any self-custody EVM wallet works the same way:

- Private key via env var for this run only (`WALLET_KEY`), never written to
  disk, never printed, never passed on a command line. Derive the address
  with `core.private_to_public(priv)` → `core.public_to_address(pub)` and
  verify it matches the address the user expects before signing anything.
- A keystore/JSON file, if the user supplies one, is read once in-process and
  never copied elsewhere.

## CLI equivalents

```bash
python -m muse address --chain bsc --count 5     # derive addresses
python -m muse balances --chain bsc --count 5     # read balances
python -m muse keys --chain bsc --index 0         # print a private key (only when the user explicitly asks; never log it)
python -m muse health                             # readiness checks
```

## Troubleshooting

- **Wrong passphrase / vault won't unlock** → the vault raises; report it
  plainly and ask the user to re-supply the passphrase. Do not retry with a
  guessed passphrase.
- **Derived address doesn't match** → stop. Do not send from an unexpected
  address. Ask the user to confirm which wallet they mean.
- **Vault file missing** → the wallet was never created/imported on this
  machine. Offer `python -m muse new` (new mnemonic) or `python -m muse import`
  (existing mnemonic) — both need the user present.
