# scaas-app

The end-user app of SCaaS: accounts and assets on native chains, Matrix chat, and, in the `scaas` build, SCaaS accounts
and tokens and PqChat, in one app.
Its name is **scaas-app**. It is not called a "wallet": it has chat.

Status of this repository: layout and the shared contract only. No app code is here yet. Everything cryptographic that
this app uses or will use is `UNAUDITED`.

## Layout

| Path | What it is | State |
|---|---|---|
| `app-common/` | Shared, toolkit-free material: the contract both app cores implement, the vectors, the view-model contract, the C++ transaction codec | `spec/` written; the rest arrives by task |
| `app-core-rust/` | The Rust app core and the app's one Rust library: a crate exporting a C ABI and a flutter_rust_bridge surface. Features: `native-core` (default, the app core proper), `scaas` (`scaas-sdk`, `ua-rs`), `pqchat`, `pqvault` (the app cores, which depend on the SDK). The Flutter and Rust front-ends link it whole; Qt links it whole (`SCAAS_QT_CORE=rust`) or without `native-core` beside `app-core-qt` (`cpp`, the default) (contract section 12) | not started |
| `app-rust/` | The Rust front-end, built with Dioxus. It links `app-core-rust` as a crate (no C ABI hop; the `scaas` feature flows through). Desktop first; it renders through the system WebView, so text input uses the OS input method | not started |
| `app-matrix/` | The Matrix crate, toolkit-free: one `matrix-rust-sdk` client and one login per app, Matrix's own end-to-end encryption, device verification. Every front-end uses it; in the `scaas` build `pqchat-core` sends its envelopes through the same client. Directory name proposed | not started |
| `app-flutter/` | The Flutter front-end, the mobile front-end; it reaches `app-core-rust` through flutter_rust_bridge | not started |
| `scaas-qt-wallet-client/` | **A separate repository**, ignored by this one. The Qt front-end and `app-core-qt`, the C++ app core. A fork of the Status desktop app (MPL-2.0), branch `qt-wallet` | Tasks 1–16 of its core plan built |
| `scaas-app-status/` | **A separate repository**, ignored by this one. An unmodified checkout of the upstream Status desktop app, kept as the design reference: screens are learned from its QML source, never from screenshots | reference only |

The two ignored repositories are cloned into this directory by hand; this repository does not track them.

## Plans

- [docs/plans/2026-11-17-scaas-app-rust.md](docs/plans/2026-11-17-scaas-app-rust.md): `app-core-rust`, `app-matrix`, `app-rust`
  (Dioxus) and `app-flutter` (groups R, M, X, F). PLANNED.
- The Qt front-end and `app-core-qt` (groups A, B, C): `docs/superpowers/plans/2026-11-17-scaas-app-next.md` in the
  `scaas-qt-wallet-client` repository.

## Two app cores, one contract

`app-core-qt` (C++, in `scaas-qt-wallet-client/`) and `app-core-rust` (here) implement the same contract,
[app-common/spec/app-core-contract.md](app-common/spec/app-core-contract.md): one vault format, one derivation-path
table, one chain-to-backend table, one `derive` / `sign` interface, one endpoint-list rule, one set of error names. Both
read the same rows of `app-common/vectors/`. CI cross-checks the two cores: the same mnemonic gives the same addresses,
the same unsigned transaction gives the same signed bytes, and a vault written by one core opens in the other. The Qt front-end runs on
either core, one per binary (`SCAAS_QT_CORE`, contract section 12); Flutter and Dioxus run on `app-core-rust`.

## Who signs what

- **Every SCaaS kind** (a Universal Account, an `mtx*` record, PQZ, Zether, PqChat) is signed by `ua-rs`, reached
  through `scaas-sdk`, and by nothing else. This is the SCaaS rule `CLI-UAL-01` (the SCaaS repository,
  `docs/decisions.md`, anchor `CLI-UAL-01`); the owner set its scope on 2026-11-17: it holds from `scaas-sdk` down.
- **Native chains** are signed by the app's own backends, exactly one backend per chain in a build: the C++ codec
  (`app-common/txcodec/`), Trust Wallet Core, or `ua-rs` (in the `scaas` build). A native backend never receives a
  SCaaS kind.

## Build flags

Variants are chosen by compile-time options only. Nothing is generated to drop functions.

| Flag | Where | Default | Effect |
|---|---|---|---|
| `SCAAS_BACKEND_TWC` | CMake (`app-core-qt`) | `ON` | compiles the Trust Wallet Core backend, the default for native chains |
| `SCAAS_BACKEND_TXCODEC_SHIP` | CMake (`app-core-qt`) | `OFF` | lets the chain table map a chain to the C++ codec in a release binary; test targets always compile the codec |
| `SCAAS` | CMake (`app-core-qt`) | `OFF` | brings in `scaas-sdk` and `ua-rs` together, and with them every SCaaS feature including PqChat |
| `native-core` | Cargo feature (`app-core-rust`) | on | the Rust app core proper; off in the Rust library a Qt binary links with `SCAAS_QT_CORE=cpp` |
| `SCAAS_QT_CORE` | CMake (Qt front-end; name proposed) | `cpp` | the Qt binary's one app core: `cpp` = `app-core-qt`, `rust` = `app-core-rust` through its C ABI |
| `scaas` | Cargo feature (`app-core-rust`) | off | the same as `SCAAS` |
| `SCAAS_PQCHAT`, `SCAAS_PQVAULT` | CMake (`app-core-qt`) | `ON` when `SCAAS` is `ON` | the app cores `pqchat-core` and `pqvault-core` (Cargo features `pqchat`, `pqvault`, each implying `scaas`); a `scaas` build may leave one out |

The default build links neither `ua-rs` nor `scaas-sdk`. It has native accounts and ordinary Matrix chat with Matrix's
own end-to-end encryption (Olm / Megolm: classical, not post-quantum). The `scaas` build adds PqChat beside it. There
is one Matrix stack for every front-end, `matrix-rust-sdk`; no libQuotient, no Waku. Switching a flag or a backend
never changes an account: the vault belongs to the app, and the same mnemonic gives the same addresses in every build.

## Licence and brand

New source files here carry `SPDX-License-Identifier: LicenseRef-Vegan-Community-1.0`. No code, asset or text of the
Status app is in this repository, and no Status name, logo or look is used: Status-derived files live only in
`scaas-qt-wallet-client/`, under MPL-2.0. `app-common/` holds nothing that depends on Qt, Flutter or a Status file.
Only source and build scripts are published; no built binaries.
