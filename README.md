# BudZero — BudZKVM

STARK-provable execution for **[Budlum](https://github.com/lubosruler/budlum)**’s Universal Settlement Layer.

A compact deterministic ISA, a gas-metered VM that emits execution traces, and a [Plonky3](https://github.com/Plonky3/Plonky3) 0.5.x STARK prover/verifier. Domains produce state; BudZKVM proves the computation that produced it.

[![CI](https://github.com/lubosruler/BudZero/actions/workflows/ci.yml/badge.svg)](https://github.com/lubosruler/BudZero/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-stable-orange)](https://www.rust-lang.org/)

---

## Role in the stack

```
  Consensus domains (PoW / PoS / PoA / BFT / ZK)
                    │
                    ▼
         Budlum L1 settlement (proofs + bridge)
                    │
                    ▼
         ┌─────────────────────┐
         │  BudZero (this repo) │
         │  ISA · VM · STARK    │
         └─────────────────────┘
```

Budlum-core depends on `bud-isa`, `bud-vm`, and `bud-proof` via path crates (`../BudZero/...`).

---

## Workspace crates

| Crate | Purpose |
| --- | --- |
| `bud-isa` | Opcode set, encode/decode, **Production vs Testing profiles** |
| `bud-vm` | Interpreter, gas, storage ops, trace emission |
| `bud-proof` | Plonky3 AIR, prover, verifier, public inputs |
| `bud-compiler` | BudL → bytecode |
| `bud-state` | Account state + nested transaction backup stack |
| `bud-cli` / `bud-node` | Tooling |

---

## Quick start

```bash
git clone https://github.com/lubosruler/BudZero.git
cd BudZero

cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

**Feature flags**

| Feature | Effect |
| --- | --- |
| default | **Production** ISA — experimental opcodes (e.g. `VerifyMerkle`) rejected at decode |
| `experimental` | Enables experimental opcodes for ZK harness / research (`bud-proof` enables this for itself) |

---

## Soundness work (honest status)

AIR/public-input binding and Merkle verification are under active hardening (Tur 10–12):

| Item | Status |
| --- | --- |
| Public inputs (Z-A phases) | Partial binding on recent commits |
| `VerifyMerkle` path AIR (Z-B) | Expansion rows + constraints; **valid 64-depth positive test still `#[ignore]`** |
| Production gate | `VerifyMerkle` treated as experimental — **off in Production profile** |
| Termination / halt (Z-C/D) | Constraints + VM behaviour iterated in Tur 10.zk |
| Storage gas (SRead/SWrite) | Higher than Load/Store; AIR gas table aligned |

Until Z-B Commit 3.5 lands, do **not** treat Merkle membership inside STARK proofs as production-safe.

---

## Gas (selected)

| Opcode | Gas |
| --- | --- |
| Load / Store | 3 |
| SRead | 8 |
| SWrite | 12 |
| Poseidon / VerifyMerkle | 10 |

---

## State (`bud-state`)

- Nested transactions use a **LIFO `backup_stack`** (not a single-slot backup).
- `State::save()` returns `Result` (no process-killing `expect` on I/O failure).

---

## Development gates

CI enforces:

1. `cargo fmt --all -- --check`
2. `cargo clippy --workspace --all-targets -- -D warnings`
3. `cargo test --workspace`

No `#[allow(clippy::…)]` as a substitute for fixing lints on new work.

---

## Relationship to Budlum CI pin

Budlum’s GitHub Actions may pin a specific BudZero commit for prove/verify compatibility while mainline BudZero advances STARK work. When rebinding the pin, re-run Budlum’s full lib suite against the new HEAD.

---

## License

MIT — see [LICENSE](LICENSE).

## See also

- [Budlum L1](https://github.com/lubosruler/budlum) — settlement, bridge, multi-consensus
- Paradigm analysis in the L1 `docs/` tree
