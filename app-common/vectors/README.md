# vectors

Empty until the tasks below land. Both app cores and every native backend read the same rows; no core keeps a copy.

| File | Rows | Task |
|---|---|---|
| `native.json` | per chain and environment: mnemonic, BIP-39 passphrase, index, public key, address; one unsigned transaction and its signed bytes (or, for a randomized scheme, a signature to verify); one negative row | B5 |
| `vault/` | vault files with their passphrases, valid and refused (wrong parameters, wrong version, altered bytes) | B6, R3 |
| cross-core rows | the same rows run through `app-core-qt` and `app-core-rust` | R8 |

Rules: a row is written by a generator or taken from a named outside tool (the tool, its version and the command are
recorded in the row); it is never edited by hand to make a test pass. Test mnemonics only. Rows that the SCaaS
repository's `smartcontracts/vectors/` also carries must equal them.
