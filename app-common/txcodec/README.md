# txcodec

Empty until task B1b. It will hold the C++ transaction codec that today lives in the Qt repository
(`src/chains/TxCodec.cpp`, `src/core/`, the C++ key and address code, and the `sr25519_ffi` crate), once that code no
longer uses a Qt type: `std::vector<uint8_t>` and `std::string` only, its own CMake target, no `Qt6::Core`.

One copy serves both cores: `app-core-qt` compiles it directly, `app-core-rust` calls it through `cxx` (task R6). It is
native backend (a) of the contract. It never encodes or signs a SCaaS kind.

Before a file moves here its history is checked: only files written for this project move; a file derived from the
Status app stays in the Qt repository under MPL-2.0.
