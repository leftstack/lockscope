# LockScope

`lockscope` is a forensic inspection tool for Bitcoin Core wallet files.

It reads locked wallets, extracts the wallet structure, and checks for signs of tampering, corruption, fabricated key material, or misleading metadata. It is aimed at situations where you want to answer questions like:

- Is this wallet structurally consistent?
- Does it look like a normal encrypted Bitcoin Core wallet?
- Are the encrypted keys, master key, and metadata internally plausible?
- Do the transactions and visible addresses line up with Bitcoin on-chain data?
- Does this file look like a Bitcoin wallet at all, or a different fork / mixed-signal wallet?

LockScope is not a password cracker and does not decrypt the wallet. It is a read-only analysis tool.

## What it supports

LockScope currently supports both major Bitcoin Core wallet storage families:

- Legacy Berkeley DB wallets
- Modern SQLite descriptor wallets

It can parse and report on:

- `mkey` master key records
- `ckey` encrypted private keys
- plain `key` records when present
- labeled addresses (`name` / `purpose`)
- stored transactions
- descriptor records and active descriptor script pubkeys in modern wallets

## What it checks

LockScope performs structural and forensic checks such as:

- presence and structure of the wallet master key
- expected encrypted key sizes
- duplicate encrypted key record detection
- ciphertext entropy checks to catch zeroed or fabricated key blobs
- public key format validation
- `defaultkey` consistency checks for legacy wallets
- descriptor cache / active script reference consistency for descriptor wallets
- labeled-address ownership checks against wallet keys or wallet descriptors
- transaction timestamp sanity checks

The output is designed to be readable for manual review, while `--json` makes it usable from scripts and pipelines.

## On-chain verification

LockScope can optionally verify wallet artifacts against Bitcoin chain data with `--verify-chain`.

Two backends are supported:

- `explorer`
  Uses an Esplora-compatible HTTP API such as [Blockstream](https://blockstream.info/api) or [mempool.space](https://mempool.space/api).
- `rpc`
  Uses your own Bitcoin Core JSON-RPC node for local verification.

Explorer mode is convenient and easy to try. RPC mode is better for privacy and for local, self-controlled verification.

Current chain verification covers:

- TXID existence / confirmation lookup
- address activity and balance via explorer APIs
- current UTXO/balance verification via Bitcoin Core RPC `scantxoutset`

When using the RPC backend, historical arbitrary TX lookups depend on Bitcoin Core `txindex=1`. If `txindex` is disabled or still syncing, LockScope warns and marks those TX lookups as unknown instead of falsely reporting them missing.

## Pre-flight network classification

Before Bitcoin-specific chain verification, LockScope performs a pre-flight classification pass.

This helps distinguish:

- normal Bitcoin wallets
- ambiguous wallets
- mixed-signal wallets
- wallets that appear to target a different known cryptocurrency or fork

If the file shows strong non-Bitcoin or mixed-network signals, LockScope will still do structural analysis, but it will skip Bitcoin-specific on-chain verification rather than query the wrong network and produce misleading results.

## What LockScope is useful for

- identifying possible fraud indicators in wallets
- triaging "loaded wallet" claims before deeper investigation
- checking whether wallet metadata & visible addresses are internally consistent
- confirming whether a wallet’s visible artifacts correspond to real Bitcoin chain data
- comparing legacy & descriptor wallet structures without opening Bitcoin Core

## Limitations

- It does not decrypt the wallet.
- It does not recover forgotten passwords.
- It does not prove ownership of coins by itself.
- It does not currently support every Bitcoin fork as a first-class chain backend.
- RPC address verification is UTXO-focused; full historical TX verification through RPC depends on `txindex`.


## Example usage

Analyze a wallet locally and print the human-readable report:

```powershell
lockscope wallet.dat
```

Analyze a modern descriptor wallet and emit JSON:

```powershell
lockscope wallet.dat --json
```

Verify on-chain data using the default Esplora-compatible explorer backend:

```powershell
lockscope wallet.dat --verify-chain
```

Verify on-chain data against a different Esplora-compatible site:

```powershell
lockscope wallet.dat --verify-chain --chain-backend explorer --explorer-url https://mempool.space/api
```

Verify on-chain data through a local Bitcoin Core RPC node using cookie auth:

```powershell
lockscope wallet.dat --verify-chain --chain-backend rpc --rpc-cookie X:\crypto\bitcoin\data\.cookie
```

Verify on-chain data through a local Bitcoin Core RPC node using username/password:

```powershell
lockscope wallet.dat --verify-chain --chain-backend rpc --rpc-url http://127.0.0.1:8332 --rpc-user bitcoinrpc --rpc-pass yourpassword
```

## Disclaimer

<p style="color: red;"><strong>Important:</strong> There is no known method that can guarantee whether a locked <code>wallet.dat</code> file is authentic or fraudulent from file inspection alone. A wallet can appear structurally valid while still being misleading, incomplete, transplanted from another context, or intentionally prepared to imitate a real Bitcoin Core wallet. Encrypted contents also limit what can be proven without the password, signing ability, or independent provenance.</p>

<p style="color: red;">LockScope is meant to provide better insight into locked <code>wallet.dat</code> files by surfacing structure, consistency signals, and potential red flags. It is not designed to "prove" fraudulence or authenticity.</p>
