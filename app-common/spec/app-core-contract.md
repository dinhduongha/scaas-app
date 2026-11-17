# App-core contract

What `app-core-qt` (C++) and `app-core-rust` (Rust) both implement. Written once, here. A difference between a core and
this text is a bug in the core. Nothing below names a Qt, Flutter or Rust type: values are bytes, strings, integers and
JSON with named fields.

Every section carries a status tag: **SPECIFIED** (written, not built to this text) or **BUILT** (code and tests in both
cores). Today every section is SPECIFIED. Everything cryptographic here is `UNAUDITED`.

Sources: the owner's rulings of 2026-11-17 (recorded in the Qt repository,
`scaas-qt-wallet-client/docs/superpowers/specs/2026-11-17-scaas-app-review.md` section 0, and
`2026-11-17-scaas-app-three-frontends.md`). SCaaS rule ids (`CLI-UAL-01`, `ARCH-POS-15`, …) are defined in the SCaaS
repository, `docs/decisions.md`.

**`[= ua-rs-native]`** marks a value that must equal the one in the SCaaS-side spec for `ua-rs` as a native backend
(`docs/superpowers/specs/2026-11-17-ua-rs-native.md` in the SCaaS repository, branch `ua-rs-native`, itself SPECIFIED).
This text was compared with that spec on 2026-11-17; what was aligned is listed in [section 9.1](#91-comparison-of-2026-11-17).

## 1. Scope — SPECIFIED

- The app core owns: the vault, the derivation of **native** chain keys, the signing of **native** transactions, the
  endpoint lists for native chain RPC.
- It never derives, stores, encodes or signs a SCaaS kind (a Universal Account, an `mtx*` record, PQZ, Zether, PqChat).
  Those exist only in the `scaas` build, through `scaas-sdk` and `ua-rs` (`CLI-UAL-01`, in force from `scaas-sdk`
  down). A request to a native backend that names a SCaaS kind is refused with `ScaasKindRefused`.
- It does not own chat. Chat is a separate crate under the same contract: [section 11](#11-chat--specified).
- No screen, script or web page reaches a raw "sign this message" call. The only signing entry is
  `sign(chain, unsigned transaction)` of [section 5](#5-backend-interface--specified).

Chain ids used throughout: `evm`, `hedera`, `bitcoin`, `cosmos`, `solana`, `stellar`, `near`, `sui`, `aptos`, `gram`,
`polkadot`. Environments: `mainnet`, `testnet`, one value for the whole app. The app core maps names at the `ua-rs`
boundary: `gram` is `ton`, `hedera` is `hedera_ecdsa`, and `bitcoin` is `bitcoin_p2wpkh` or `bitcoin_p2tr` by the
account's type.

## 2. Vault — SPECIFIED

One file, one format, in every build and in both cores. The vault belongs to the app, not to a backend. Switching a
build flag or a backend never changes it.

### 2.1 File

`vault.json` in the app's private data directory. UTF-8 JSON, no byte-order mark. Binary fields are base64url without
padding.

```json
{
  "format": "scaas-app.vault",
  "v": 1,
  "seeds": [
    {
      "id": "0b9c2f0e-5d0e-4a51-9d3b-6a1f6f6f2a10",
      "label": "main",
      "words": 24,
      "kdf": {"name": "scrypt", "logN": 17, "r": 8, "p": 1, "salt": "<16 bytes>"},
      "aead": {"name": "aes-256-gcm", "iv": "<12 bytes>", "ct": "<ciphertext ‖ 16-byte tag>"}
    }
  ]
}
```

| Field | Rule |
|---|---|
| `format`, `v` | exactly `"scaas-app.vault"` and `1`. Anything else: `VaultVersion`. A core never guesses at another format. The keystore of the Qt client from before this contract is not read |
| `seeds` | one envelope per seed; the order is the display order. An empty list is a valid vault |
| `id` | UUID, lower case, unique in the file. Never reused |
| `label` | 1 to 64 Unicode scalar values, NFC, no control characters; unique in the file |
| `words` | 12, 15, 18, 21 or 24: the word count of the mnemonic. It must match the plaintext length (16, 20, 24, 28, 32 bytes); otherwise `VaultFormat` |
| `kdf` | exactly scrypt with `logN = 17`, `r = 8`, `p = 1` (`N = 2^17`), 32-byte output, a fresh random 16-byte `salt` per envelope. Any other name or parameter is refused with `VaultParams` **before** the KDF runs |
| `aead` | AES-256-GCM, a fresh random 12-byte `iv` on every write of the envelope, tag appended to the ciphertext |

An unknown top-level or envelope field is `VaultFormat`: the file holds nothing a core does not understand.

### 2.2 Envelope

- **Plaintext:** the BIP-39 entropy, raw, 16 to 32 bytes. Nothing else. Not the words, not the 64-byte seed, not the
  BIP-39 passphrase, not a derived key.
- **Key:** `scrypt(NFKC(vault passphrase) as UTF-8, salt, N = 2^17, r = 8, p = 1, 32)`.
- **Associated data:** the UTF-8 bytes of
  `scaas-app.vault|1|<id>|<label>|<words>|scrypt|17|8|1`. Changing a label re-seals the envelope.
- Each seed has its own passphrase and its own envelope. "One passphrase for all" is a front-end convenience: it opens
  each envelope with the same string. A wrong passphrase, an altered envelope and an altered header all give the one
  error `VaultOpen`.

### 2.3 Mnemonics

- A **generated** seed is always 24 words (256 bits of entropy from the operating system's generator).
- An **imported** mnemonic of 12, 15, 18 or 21 words is accepted. The core returns the flag `short_seed` with it; the
  front-end shows the warning at import and wherever that seed is listed. Any other count, a word outside the English
  list or a bad checksum: `MnemonicInvalid`.
- Normalisation before the checksum: NFKD, lower case, single spaces.
- The BIP-39 passphrase is entered at unlock and never written anywhere. An empty one is the main account set; another
  one is a hidden set. A wrong one is not an error: it gives other addresses. The core keeps it **as entered** and hands
  it to the backends unchanged: the BIP-39 seed normalises it (NFKD), the Polkadot derivation of section 3.1 does not.
  The front-end warns on a non-ASCII passphrase: its NFC and NFD forms give one BIP-39 seed but two Polkadot accounts.

### 2.4 Unlock, lock, write

- **Unlock** of a seed: open its envelope, then hand the mnemonic and the BIP-39 passphrase to every backend compiled
  into the build (`unlock` of [section 5](#5-backend-interface--specified)). In the `scaas` build `ua-rs` receives it
  the same way. Nothing secret comes back out of a backend.
- **Lock:** every backend's `lock`, then the core overwrites the entropy, the 64-byte seed, the passphrases and every
  derived key it holds. After lock, `derive` and `sign` return `Locked`.
- **Write:** to a temporary file in the same directory, `fsync` the file, rename over `vault.json`, `fsync` the
  directory. File mode `0600` where the platform has modes; otherwise the platform's app-private storage. A failed
  write leaves the old file in place and returns `VaultIo`.
- An error message or a log line never contains a secret, a passphrase or a mnemonic word.

## 3. Derivation paths — SPECIFIED

One table. Every backend reads it; no backend uses a default path of its own. `i` is the account index, from 0.
`[= ua-rs-native]` applies to every row.

| Chain | Curve | Path (mainnet) | Testnet differs by | Address |
|---|---|---|---|---|
| `evm` | secp256k1, BIP-32 | `m/44'/60'/0'/0/i` | nothing | last 20 bytes of keccak256 of the public key, EIP-55 |
| `hedera` | secp256k1, BIP-32 | `m/44'/3030'/0'/0/i` | nothing | the EVM alias, as `evm`. No `0.0.N` id is derived |
| `bitcoin` | secp256k1, BIP-32 | BIP-84: `m/84'/0'/0'/0/i` | coin type `1'` | P2WPKH bech32, `bc1q…` / `tb1q…` |
| `bitcoin`, Taproot (optional account type) | secp256k1, BIP-32 | BIP-86: `m/86'/0'/0'/0/i` | coin type `1'` | P2TR bech32m, `bc1p…` / `tb1p…`, the BIP-341 key tweak with no script path |
| `cosmos` | secp256k1, BIP-32 | `m/44'/118'/0'/0/i` | nothing | bech32 `cosmos`, RIPEMD-160 of SHA-256 of the compressed key |
| `solana` | ed25519, SLIP-10 | `m/44'/501'/i'/0'` | nothing | base58 of the public key |
| `stellar` | ed25519, SLIP-10 | `m/44'/148'/i'` | nothing | `G…` strkey |
| `near` | ed25519, SLIP-10 | `m/44'/397'/i'` | nothing | implicit account: the public key as 64 hex digits |
| `sui` | ed25519, SLIP-10 | `m/44'/784'/i'/0'/0'` | nothing | `0x` + BLAKE2b-256 of `0x00 ‖ public key` |
| `aptos` | ed25519, SLIP-10 | `m/44'/637'/i'/0'/0'` | nothing | `0x` + SHA3-256 of `public key ‖ 0x00` |
| `gram` | ed25519, SLIP-10 | `m/44'/607'/i'` | the testnet flag of the address and the wallet's network id | the address of a W5R1 wallet contract for that public key, workchain 0; never the bare public key |
| `polkadot` | sr25519 | no path: see 3.1 | SS58 prefix | SS58, prefix 0 on mainnet, 42 on testnet |

The secp256k1 and ed25519 rows for the families SCaaS has are the forms of the SCaaS repository,
`docs/clients/signers-clientkeys.md` section 9.

### 3.1 Polkadot

Owner rulings 2026-11-17: sr25519 with the substrate-bip39 derivation, from the bare mnemonic, as the default account of
polkadot.js.

1. Mini-secret: the first 32 bytes of `PBKDF2-HMAC-SHA512(password = BIP-39 entropy bytes, salt = "mnemonic" ‖
   UTF-8(BIP-39 passphrase as entered), 2048 rounds, 64 bytes)`. The password is the **entropy**, not the mnemonic
   sentence: this is not the BIP-39 seed. The passphrase is **not** NFKD-normalised here, as `substrate-bip39` and
   polkadot.js take it.
2. Expand the mini-secret to an sr25519 key pair (the Ed25519 expansion mode).
3. **No junction** by default: the key of step 2 is the account. Mainnet and testnet use the same key and differ only
   in the SS58 prefix.
4. An account may carry an optional junction path (`//polkadot`, `//paseo`, …) as a per-account setting, applied to the
   key of step 2. Grammar, chain codes and bounds are exactly those of `ua-rs-native` section 7.3: `//` hard and `/`
   soft junctions of 1 to 64 printable ASCII bytes; a digit-only junction is its `u64` little-endian, any other the
   SCALE string; an encoding of at most 32 bytes zero-padded, a longer one replaced by its BLAKE2b-256; at most 16
   junctions and 512 bytes. Refused with `JunctionInvalid`: a `///` password part, an empty junction, one starting
   with `0x` or `+`, a digit-only one outside `u64`. A path is public account data, never a constant in a core, never
   a password. A soft-derived key's signatures are not reproducible across tools: verify only.
5. Index: `derive("polkadot", 0)` is the bare account. Any other index without a junction is `IndexUnsupported`;
   further accounts are made with a junction.
6. Signing context: `substrate`.

## 4. Chain-to-backend table — SPECIFIED

Three native backends:

| | Backend | Compiled when |
|---|---|---|
| (a) | The C++ codec, `app-common/txcodec/` | always in test targets; in a release binary only where the table needs it |
| (b) | Trust Wallet Core, pinned at 4.8.4 | `SCAAS_BACKEND_TWC` (default on) |
| (c) | `ua-rs` | the `scaas` build only (`SCAAS` / Cargo feature `scaas`) |

Rules:

1. In one build every chain maps to **exactly one** backend. The table is a constant in each core, selected by
   compile-time options; nothing is generated.
2. A chain mapped to a backend that is not compiled is a build error, not a run-time fallback.
3. No SCaaS kind appears in this table.
4. A chain moves from (a) to (b), or to (c), only after the [cross-check](#8-cross-check-gate--specified) passes for
   that chain with both backends. Until then it stays where it is.
5. With (c) the **encoding** stays with (b) or (a): that backend builds the signing preimage, `ua-rs` signs the digest
   or message, the backend assembles the signed transaction. The digest comes only from a transaction built in this
   process, the user has seen the decoded transaction first, and the raw-digest opt-in is used only on native keys.

| Chain | Default build | `SCAAS_BACKEND_TWC=OFF`, `SCAAS_BACKEND_TXCODEC_SHIP=ON` | `scaas` build |
|---|---|---|---|
| `evm`, `hedera` | (b) | (a) | (c) signs, (b) encodes |
| `bitcoin` | (b) | (a) | as default, until `ua-rs` has the network |
| `cosmos`, `solana`, `stellar`, `sui`, `aptos`, `gram` | (b) where 4.8.4 gives this table's addresses; else (a) | (a) | (c) once `ua-rs` signs that chain's message; before that as default |
| `near` | (b) | (a) | as default, until `ua-rs` has the network |
| `polkadot` | (a): Trust Wallet Core 4.8.4 has only ed25519 `m/44'/354'/0'/0'/0'` for Polkadot and no sr25519 signer | (a) | (c) once `ua-rs` has an sr25519 signer; before that (a) |

The starting state of both cores is every chain at (a); each "(b)" above is a target that rule 4 has to confirm.

## 5. Backend interface — SPECIFIED

One interface for the three backends, the same in both cores.

| Call | Input | Output |
|---|---|---|
| `unlock` | seed id, mnemonic, BIP-39 passphrase | nothing; the backend holds what it needs until `lock` |
| `lock` | — | nothing; the backend wipes what it holds |
| `derive` | chain, environment, seed id, index, optional junction (`polkadot` only) | public key, address |
| `sign` | chain, environment, seed id, index, optional junction, unsigned transaction | signed transaction bytes, transaction id |

- `derive` and `sign` do no network I/O. Everything a transaction needs from a node (nonce, fee, block hash, gas
  objects, expiry) is a field of the unsigned transaction, filled by the caller. A backend never invents such a value.
- Signature rules, `[= ua-rs-native]`: ECDSA is RFC 6979 with no extra entropy, low-S, **no low-R grinding**; Ed25519 is
  RFC 8032; sr25519 is randomized.
- `sign` checks every field before it signs; a missing, extra or malformed field is `TxField`, a bad address
  `AddressInvalid`, a bad amount `AmountInvalid`.

### 5.1 Unsigned transactions

JSON with named fields. Amounts are decimal strings in the chain's smallest unit; byte strings are `0x` hex; no
floating point anywhere. One object is one native transfer unless a row says more.

| Chain | Fields | Signed output |
|---|---|---|
| `evm`, `hedera` | `chainId`, `nonce`, `to`, `value`, `data`, `gasLimit`, `maxFeePerGas`, `maxPriorityFeePerGas` | an EIP-1559 (type 2) transaction, RLP. Type 4 (EIP-7702) has no shape here: `TxTypeRefused` |
| `bitcoin` | `inputs` `[{txid, vout, value, scriptPubKey}]`, `outputs` `[{address, value}]` (change is an explicit output), `locktime`, `sequence` | a segwit transaction, `SIGHASH_ALL` (BIP-143); Taproot inputs `SIGHASH_DEFAULT` (BIP-341) |
| `cosmos` | `chainId`, `accountNumber`, `sequence`, `fee` `{amount, denom, gas}`, `memo`, `from`, `to`, `amount`, `denom` | `TxRaw`, `SIGN_MODE_DIRECT`, one `MsgSend` |
| `solana` | `recentBlockhash`, `from`, `to`, `lamports` | a legacy transaction: one signature and the message, one System transfer |
| `stellar` | `networkPassphrase`, `source`, `sequence`, `fee`, `timeBounds` `{min, max}`, `destination`, `amount`, `createAccount` (bool) | a v1 `TransactionEnvelope`, XDR |
| `near` | `signerId`, `nonce`, `receiverId`, `blockHash`, `deposit` | a Borsh `SignedTransaction`, one `Transfer` action |
| `sui` | `sender`, `gasPayment` `[{objectId, version, digest}]`, `gasPrice`, `gasBudget`, `recipient`, `amount` | BCS `TransactionData` V1 with the signature over its intent hash |
| `aptos` | `sender`, `sequenceNumber`, `maxGasAmount`, `gasUnitPrice`, `expirationTimestampSecs`, `chainId`, `recipient`, `amount` | a BCS signed transaction; the signature over `sha3_256("APTOS::RawTransaction") ‖ BCS(raw)` |
| `gram` | `walletId`, `seqno`, `validUntil`, `to`, `amount`, `bounce`, `sendMode`, `stateInit` (present exactly when `seqno` is 0) | the external message to the sender's W5R1 wallet, as a bag of cells |
| `polkadot` | `genesisHash`, `blockHash`, `era`, `specVersion`, `transactionVersion`, `nonce`, `tip`, `palletIndex`, `callIndex`, `dest`, `value` | a signed extrinsic, `balances.transferKeepAlive`; the pallet and call indices come from the chain's metadata, supplied by the caller |

## 6. Endpoint lists — SPECIFIED

`ARCH-POS-15` names the app: everything that reads or writes a chain through an RPC takes a **list** of endpoints.

- Per (chain, environment): an ordered list of `{url, class, label}`. `class` is `default` (shipped in the release's
  reviewed list), `named` (added by the user) or `discovered` (found at run time). A URL whose scheme is not `http` or
  `https` is refused when it is added: `EndpointScheme`.
- **Reads, `failover` (the default policy):** ask the first endpoint; on a transport error, a timeout, an HTTP 5xx or a
  malformed answer, ask the next. A well-formed error from the chain is an answer, not a reason to move on. When the
  list is exhausted: `EndpointAllFailed`. No retry beyond one pass of the list.
- **Reads, `strict_majority` (optional, per chain):** ask every endpoint; accept a value only when more than half of
  the list returned it; otherwise `EndpointNoMajority`. It needs at least three endpoints.
- **Broadcast:** the same signed bytes go to the endpoints; a transaction is **never signed again** to try another
  endpoint. "Already known" from a later endpoint counts as accepted. A rejection by the chain is `BroadcastRejected`;
  the core reports "sent" only after an endpoint accepted the bytes.
- In the `scaas` build the SCaaS side uses `scaas-rpc`'s rule, including: a read that feeds a decryption is never
  served by a `discovered` endpoint alone (`ARCH-POS-13`).

## 7. Error names — SPECIFIED

Stable names with stable numbers; a C ABI returns the number, every binding maps it back to the name. 0 is success.
Numbers are appended, never reused.

| No | Name | When |
|---|---|---|
| 1 | `VaultOpen` | wrong passphrase, or the envelope or its header altered |
| 2 | `VaultParams` | a KDF or AEAD name or parameter outside section 2 |
| 3 | `VaultVersion` | `format` or `v` not this contract's |
| 4 | `VaultFormat` | not this JSON shape: unknown field, bad length, duplicate id or label |
| 5 | `VaultIo` | read or write failed; the old file is intact |
| 6 | `Locked` | `derive` or `sign` while locked |
| 7 | `MnemonicInvalid` | word count, word list or checksum |
| 8 | `ChainUnknown` | a chain id outside section 1 |
| 9 | `EnvUnknown` | not `mainnet` or `testnet` |
| 10 | `IndexUnsupported` | an index the chain's row does not allow |
| 11 | `JunctionInvalid` | a junction on another chain, a soft junction, a password suffix |
| 12 | `BackendUnavailable` | the table's backend for this chain cannot run here |
| 13 | `TxField` | a missing, extra or malformed field of an unsigned transaction |
| 14 | `TxTypeRefused` | a transaction type this contract has no shape for |
| 15 | `AddressInvalid` | a destination that is not an address of that chain and environment |
| 16 | `AmountInvalid` | not a positive decimal integer string, or out of range |
| 17 | `EndpointScheme` | a URL that is not `http` or `https` |
| 18 | `EndpointAllFailed` | every endpoint of the list failed |
| 19 | `EndpointNoMajority` | `strict_majority` found no value on more than half |
| 20 | `BroadcastRejected` | the chain refused the signed bytes |
| 21 | `AbiMismatch` | a binding and its core disagree on the ABI version |
| 22 | `ScaasKindRefused` | a SCaaS kind reached a native backend |
| 23 | `Internal` | a failure with no better name; carries no secret |
| 24 | `MatrixLogin` | the homeserver refused the login, or discovery failed |
| 25 | `MatrixSession` | no session, or the stored one is no longer valid |
| 26 | `MatrixStore` | the local store cannot be opened, or no secret storage holds its key |
| 27 | `MatrixVerification` | a verification was cancelled, timed out or did not match |
| 28 | `MatrixSend` | an event was not accepted by the homeserver |

A warning is not an error: `short_seed` (section 2.3) comes back beside a success.

## 8. Cross-check gate — SPECIFIED

A CI job, the same idea as an oracle implementation beside a production one. It runs over `app-common/vectors/` for
every pair that covers a chain: two backends in one core, and the two cores.

- **Addresses:** the same mnemonic, BIP-39 passphrase, environment and index give the same public key and address.
  Rows: passphrase empty and non-empty, index 0 and 1, mainnet and testnet.
- **Signed bytes:** the same unsigned transaction gives the same signed bytes, for the deterministic schemes (ECDSA
  under section 5's rule, Ed25519). `gram` compares the representation hash of the signed message's root cell, not the
  bytes of the bag of cells.
- **Randomized schemes** (sr25519 on `polkadot`; BIP-340 with auxiliary randomness on Taproot inputs): each side signs,
  the other verifies; bytes are not compared. `polkadot` therefore compares addresses and cross-verifies signatures.
- **Vault:** a vault written by one core opens in the other with the same passphrase and gives the same addresses;
  every refused file of `vectors/vault/` is refused by both with the same error name.
- **Negative rows:** per chain, one changed field of the unsigned transaction must change the bytes; a signature made
  by another key must be refused.
- The gate must be able to fail: it is run once with a deliberately wrong path in one backend, and that red run is
  recorded.
- A chain covered by a single backend is printed as `single backend, no cross-check`; it is never silently skipped.
  Its rows are compared with values taken from a named outside tool.

## 9. What must equal `ua-rs-native` — SPECIFIED

For `ua-rs` to be backend (c) of a chain, these must be byte-equal between this contract and the SCaaS-side spec:

1. every row of the path table (section 3) for a network `ua-rs` has or adds, including Bitcoin BIP-84 and BIP-86 and
   NEAR;
2. the Polkadot derivation of section 3.1: sr25519, substrate-bip39 mini-secret from the entropy, no junction by
   default, the `substrate` signing context;
3. the address encodings of section 3;
4. the signature rules of section 5 (RFC 6979 without extra entropy, low-S, no low-R grinding; RFC 8032);
5. the preimage each chain hands to `ua-rs`. A 32-byte digest (`sign_digest`): the secp256k1 chains (Bitcoin: the
   BIP-143 or BIP-341 sighash), NEAR (`sha256` of the Borsh transaction), `gram` (the cell hash), Sui (the BLAKE2b-256
   intent hash), Stellar (the SHA-256 of the signature payload). The full message (`sign_message`, at most 128 KiB):
   Solana (the transaction message), Aptos (`sha3_256("APTOS::RawTransaction") ‖ BCS(raw)`), Polkadot (the SCALE
   signing payload, or its BLAKE2b-256 when longer than 256 bytes; the app's encoder applies that rule);
6. the rows of `app-common/vectors/native.json` that the SCaaS repository's `smartcontracts/vectors/native/native.json`
   also carries. That file is written by clientkeys first; its mnemonics (`abandon` ×11 `about`, the same with
   `TREZOR`, the Substrate development phrase, a 24-word phrase, a non-ASCII passphrase in NFC and NFD) are reused
   here, so the same rows run through all three backends;
7. the caller's obligations for a raw digest or message (`ua-rs-native` section 9, `CLI-SGN-54`): the preimage built in
   this process from named fields, the decoded transaction shown first, never with a UA key or a SCaaS kind, one key
   per network and purpose.

### 9.1 Comparison of 2026-11-17

Compared with `ua-rs-native` as it stood that day (SPECIFIED on its side too; it may still move, and then this section
is redone).

| Point | This contract before | `ua-rs-native` | Now |
|---|---|---|---|
| Polkadot passphrase in the mini-secret salt | NFKD-normalised | as given, not normalised | aligned: as given (sections 2.3, 3.1) |
| Polkadot junctions | hard only; soft refused | hard and soft; a digit-only junction is a `u64`; bounds 16 junctions, 512 bytes | aligned (section 3.1) |
| Preimage form per Ed25519 chain | NEAR, Sui and Stellar listed as full messages | NEAR and `gram` sign 32-byte hashes; messages for Solana, Aptos, Hedera body bytes, Substrate payloads | aligned (item 5); Sui and Stellar are 32-byte hashes by their own rules, which `ua-rs-native` does not list one by one |
| Bitcoin network names | one chain `bitcoin` with two account types | two networks `bitcoin_p2wpkh`, `bitcoin_p2tr` | kept; mapped at the boundary (section 1) |
| Vector file | `app-common/vectors/native.json` | `smartcontracts/vectors/native/native.json`, clientkeys writes first | both exist; shared rows byte-equal (item 6) |

Equal without a change: Bitcoin BIP-84 and BIP-86 paths with coin `1'` on testnets and the bech32 / bech32m addresses;
NEAR `m/44'/397'/i'` with the implicit account id; Polkadot sr25519, `substrate-bip39` from the entropy, the bare
account, index 0 only, the `substrate` context, SS58 prefixes 0 and 42; ECDSA by RFC 6979 with no extra data, low-S,
no low-R grinding; Taproot and sr25519 cross-verified, not byte-compared.

Not covered by `ua-rs-native`, so nothing to compare: the vault, the endpoint lists, the error names, the unsigned
transaction shapes, and Bitcoin change addresses (it derives the external chain `…/0/i` only; so does this contract,
and a change output is an address of that chain).

## 10. Open points

Not decided by this text; each is settled by the task named, then written back here.

1. **`ua-rs-native` is itself SPECIFIED** and marks its names and its junction grammar as proposals. Section 9.1 is
   redone when it changes or is built.
2. **Bitcoin path changed from the Qt client's code.** The built code derives `m/44'/0'/0'/0/0` and encodes P2WPKH; this
   contract follows the owner's brief (BIP-84, BIP-86 optional). The Qt client's pinned `bc1q…` and `tb1q…` addresses
   change with task B2. Taproot is specified here and built by no backend yet.
3. **Sui and Aptos transfer shape:** which transaction kind encodes a native transfer (Sui: the programmable
   transaction that splits the gas coin; Aptos: which entry function) is fixed by the first vector row both backends
   reproduce (task B5), not here.
4. **`gram` on testnet:** the W5R1 wallet id for testnet is pinned by a vector row (task B5).
5. **Polkadot era:** mortal or immortal by default is the caller's choice in the `era` field; the front-end default is
   set with the view-model contract (task B10).
6. **Account list:** which accounts exist (seed id, chain, index, optional junction, name) is not secret and is not in
   the vault. Its file format is written with the view-model contract (task B10).
7. **Matrix crate:** its directory (`app-matrix/` proposed, as a crate of its own rather than a module of
   `app-core-rust`), the pinned `matrix-sdk` version, and what a locked app shows of chat are set by the Matrix task
   group (M1, M6) and written back to section 11.
8. **Flag names** `SCAAS_BACKEND_TWC`, `SCAAS_BACKEND_TXCODEC_SHIP`, `SCAAS` and the feature `scaas` are settled; the
   pinned Trust Wallet Core version (4.8.4) moves only with a full cross-check run.

## 11. Chat — SPECIFIED

Owner ruling 2026-11-17. The app core of sections 1 to 8 does not own chat; a separate toolkit-free crate does
(`app-matrix/`, name proposed). Both builds and every front-end use it.

### 11.1 What each build offers

| Build | Chat |
|---|---|
| default (no SCaaS) | ordinary Matrix chat, end-to-end encrypted by Matrix itself (Olm / Megolm) |
| `scaas` | the same, **plus** PqChat sessions through `pqchat-core` |

- Matrix's encryption is classical, not post-quantum. The front-end says so where it describes a conversation's
  protection; it never presents a Matrix conversation as post-quantum.
- The two kinds of conversation are never mixed. A PqChat session is not a Matrix room to the user, even when a Matrix
  room carries its envelopes; a Matrix room is never upgraded into a PqChat session in place.
- Olm and Megolm keys belong to `matrix-sdk`. They are not SCaaS kinds, are not derived from a seed, and are not in the
  vault. `CLI-UAL-01` does not concern them; every PqChat signature and key still comes from `ua-rs`.
- No other transport and no home-made envelope: no Waku, no app-defined encryption layer.

### 11.2 One stack, one client, one login

- One Matrix stack for every front-end: `matrix-rust-sdk` (`matrix-sdk` with the feature `e2e-encryption`), pinned to
  one exact stable version. No libQuotient.
- One client and one login per app. The crate owns the session; `app-core-qt` reaches it through its C ABI,
  `app-core-rust` and the Dioxus front-end as a crate, Flutter through the bridge.
- In the `scaas` build `pqchat-core` uses **that same client** as its Matrix store: it hands over an already
  PQ-encrypted, signed envelope, which is sent as a `net.scaas.pqchat.v1` event, and every such event received goes
  back to `pqchat-core`. There is no second login and no second sync loop. So the Matrix crate, `pqchat-core`,
  `scaas-sdk` and `ua-rs` are linked into one Rust library in that build.
- In the default build a `net.scaas.pqchat.v1` event is shown as "needs the SCaaS build", never as a message.

### 11.3 Login and session

- Homeserver: an `https` URL (`EndpointScheme` otherwise), with `.well-known` discovery. A homeserver is not a chain
  RPC: `ARCH-POS-15`'s endpoint lists do not apply, an account has one homeserver.
- Login by password or by single sign-on in the system browser. The Matrix password is never stored.
- The session (user id, device id, access and refresh tokens) is kept by the crate so the app restores it at start
  without asking again. Logging out invalidates the token on the server and deletes the local store.
- Locking the vault does not log out of Matrix: the Matrix account is not tied to a seed. What a locked app shows of
  chat is the front-end's rule, set in the view-model contract.
- No token, key or message text appears in an error or a log line.

### 11.4 Store

- The SDK's state and crypto stores are on disk in the app's private data directory, encrypted under a random 32-byte
  store key made at first login.
- The store key is kept in the operating system's secret storage (Keychain, Android Keystore, the Secret Service,
  Windows credentials). It is not in `vault.json`, whose format holds seeds only (section 2), and it is not derived
  from a seed or a vault passphrase.
- Without that secret storage the crate refuses to persist a session (`MatrixStore`) rather than write the key beside
  the store; the user logs in again at each start.

### 11.5 Encryption and device verification

- A direct conversation the app creates is encrypted from its first event. The app never turns encryption off in a
  room. A room that is not encrypted carries a flag, and the front-end labels it.
- Cross-signing is set up at first login on a new account, or the device is verified against an existing one.
- Verifying a device: by comparing emoji (SAS) or by QR code, with the other device or the other user. The outcome is
  one of verified, unverified, or changed-since-verified; the last is shown as a warning, never silently accepted.
- A message from an unverified device is shown with that state. A message that cannot be decrypted is shown as
  undecryptable, never as empty and never dropped.
- Key backup: a recovery key is shown once at setup; server-side key backup is optional and off until the user turns
  it on. Losing every device and the recovery key loses the history; the app says so at setup.

### 11.6 What the front-end gets

Models and events, in the view-model contract's terms: the room list (id, name, last event, unread, encrypted,
verification state), a paged timeline per room, incoming events, send results, the login state, the verification
flow's steps. Actions: log in, log out, create a direct conversation, send text, start and answer a verification, set
up and use the recovery key. Attachments and calls are not in this version.
