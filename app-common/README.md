# app-common

What both app cores and every front-end share. One rule: **toolkit-free**. Nothing here depends on Qt, on Flutter, or
on a file of the Status app. Plain C++17, Rust without a UI crate, JSON, Markdown.

| Directory | Holds | Arrives |
|---|---|---|
| `spec/` | [app-core-contract.md](spec/app-core-contract.md): what `app-core-qt` and `app-core-rust` both implement | written (task A0); each later task updates its section before code changes |
| `vectors/` | Known answers read by both cores and every backend: addresses, signed bytes, vault files | tasks B5, B6, R8 |
| `viewmodel/` | The view-model contract: per screen, its models, actions, events and flow states, in toolkit-neutral terms | task B10 |
| `txcodec/` | The C++ transaction codec and its C++ crypto, one copy for both cores (`app-core-rust` through `cxx`) | task B1b, after it is free of Qt |

Task ids are those of the plan in the Qt repository,
`scaas-qt-wallet-client/docs/superpowers/plans/2026-11-17-scaas-app-next.md`.

What goes where:

- A byte format, a path, a table, an error name: `spec/`, once, then the cores follow. A difference between a core and
  the contract is a bug in the core.
- A value a test compares against: `vectors/`, written by a generator or taken from a named outside tool, never typed
  into one core's test alone.
- Anything a screen shows or does: `viewmodel/`.
- Code that encodes or signs a native transaction and is shared: `txcodec/`. Code for one toolkit stays with that
  toolkit.
