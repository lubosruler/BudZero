# BudZKVM

> The ZK execution layer for Budlum's Universal Settlement vision.
> A small deterministic ISA (31 opcodes), a trace-generating VM, and
> a Plonky3 0.5.2-based STARK prover — the missing piece between
> any consensus and any settlement.

[![CI](https://img.shields.io/badge/CI-success-brightgreen)](https://github.com/lubosruler/BudZero/actions)
[![Tests](https://img.shields.io/badge/tests-58-blue)](https://github.com/lubosruler/BudZero)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Rust Version](https://img.shields.io/badge/rust-stable-orange.svg)](https://www.rust-lang.org/)

---

## The Vision — Universal Settlement Layer

Every blockchain today is an island. Bitcoin is secure but slow;
Ethereum is fast but permissioned at the validator set; CBDCs are
isolated by design. There is no single layer that *settles* value
across them all without one side trusting the other.

**Budlum is that layer.** It does not replace any blockchain — it
*verifies* them. Each chain runs on its own consensus (PoW, PoS, PoA,
BFT, ZK, or anything custom); Budlum holds the cryptographic proof
that the work was done. Cross-domain transfers become mathematically
settled, not human-trusted.

BudZKVM is the execution half of that vision: a STARK-provable VM
that can run arbitrary code on top of any domain and present the
result as a settlement proof. Combined with the BLS + Dilithium5
hybrid finality in the L1, the result is a settlement layer that is:

* **Post-quantum secure by design** — Dilithium5 is woven into the
  finality core, not bolted on.
* **Multi-consensus** — a domain is just a registered adapter; the L1
  does not care which consensus produced a block.
* **Provable end-to-end** — every state transition that lands on Budlum
  carries a STARK proof of the computation that produced it.
* **Permissionless** — anyone can submit a proof; no validator approval
  is required because the proof is self-verifying.

See [`BUDLUM_PARADIGMA_ANALIZI`](/lubosruler/budlum/blob/main/docs/03_paradigma_analizi.md)
(in the L1 repo) for the full strategic analysis.

---

## Where BudZKVM Fits

```
                    Consensus domains (the "producers")
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │   PoW    │   │   PoS    │   │   PoA    │   │    ZK    │
   │ domain   │   │ domain   │   │ domain   │   │ domain   │
   └─────┬────┘   └─────┬────┘   └─────┬────┘   └─────┬────┘
         │              │              │              │
         │   DomainFinalityAdapter produces             │
         │   a FinalityProof (BLS aggregate +            │
         │   Dilithium5 QC blob)                        │
         ▼              ▼              ▼              ▼
   ┌──────────────────────────────────────────────────────────┐
   │            BUDLUM SETTLEMENT LAYER (the L1)              │
   │   GlobalBlockHeader commits every domain's proof into    │
   │   a single settlement record.                            │
   │                                                          │
   │   ┌────────────────────────────────────────────────┐    │
   │   │  BudZKVM (this repo)                            │    │
   │   │  The execution engine that produces the          │    │
   │   │  STARK proofs. Cross-domain contracts run       │    │
   │   │  here; their output is verified on settlement.  │    │
   │   └────────────────────────────────────────────────┘    │
   └──────────────────────────────────────────────────────────┘
                            │
                            ▼
              Verified, settled, audit-able value transfer
```

---

## What This Repository Is

BudZKVM is the ZK-native virtual machine, language toolchain, and
STARK proving stack that powers Budlum's execution layer:

| Crate | Role |
| --- | --- |
| `bud-isa` | Instruction encoding, 31 opcodes, deterministic bytecode |
| `bud-vm` | 32-register, 64-bit, gas-metered VM with execution-trace generation |
| `bud-compiler` | BudL compiler (lexer, parser, sema, codegen) — **Tur 8** adds `match` expressions |
| `bud-proof` | Plonky3 0.5.2 STARK prover, 354-column trace matrix, LogUp CTL |
| `bud-cli` | CLI: compile, deploy, run, prove, verify, batch |
| `bud-state` | File-backed state, 64-depth SMT, account model, atomic commit/rollback |
| `bud-node` | Node integration layer (placeholder for future P2P/RPC) |
| `docs` | Turkish book-style documentation (10 chapters + 3 guides) |

---

## Production Hardening Achievements

The stack has completed its core production hardening plan:

1. **All 31 Opcodes Production-Ready** — Comparison (64-bit decomposition
   + equality prefix flags), Bitwise (bit decomposition + algebraic
   equivalence), Poseidon4 hash (alpha=7, Goldilocks), Storage
   (STORAGE_BASE partitioning via memory LogUp), VerifyMerkle
   (poseidon4_hash-based 64-depth).
2. **Profile-Based ISA Security** — `IsaProfile` gates experimental
   opcodes at compile/decode time. All 31 opcodes are now production.
3. **Soundness via Inverse Witnesses** — Div, Inv, Eq/Neq, Jnz, and Not
   all use arithmetic inverse witness patterns for zero/non-zero
   detection.
4. **R0 Hardware Zero** — Hardwired R0 constraint enforced at trace
   generation and AIR level.
5. **Padding Isolation** — `COL_CPU_ACTIVE` excludes padding rows from
   LogUp CTL lookups.
6. **3-Table LogUp CTL** — Register, Memory+Storage, and Program tables
   cross-verified via LogUp fractional sums.
7. **DoS-Protected Serialization** — `postcard` with bounded
   deserialization (10 MB limit).
8. **Structured Tracing** — `RUST_LOG=info` pipeline-wide tracing
   (compiler, VM, prover, CLI).
9. **8 Negative Soundness Tests** — Tampered comparison, bitwise,
   poseidon S-box, storage, PC, public inputs, program, invalid proof
   bytes.
10. **64-Depth Sparse Merkle Tree** — File-backed state with SMT
    inclusion/non-membership proofs, domain-separated state root.
11. **CI Pipeline (mainnet-readiness)** — `cargo fmt --check` +
    `cargo clippy -D warnings` + `cargo test --workspace` on every
    push to `main`.

---

## BudL Language — Tur 8 Surface

| Feature | Status | Tur |
| --- | :---: | :---: |
| `let` bindings, integer/hex literals, arithmetic | ✓ | 1–2 |
| `if` / `else` (statement form) | ✓ | 3 |
| `while` loops | ✓ | 3 |
| `for i in start..end` loops | ✓ | 4 |
| User-defined function calls (`fn` + `Call`/`Ret` ABI) | ✓ | 5 |
| Comments (`//` and `/* */`) | ✓ | 5 |
| Structs, dynamic heap memory (r31 HEAP_PTR) | ✓ | 6 |
| Static type system (sema: `u64`/`bool`/field) | ✓ | 7 |
| **Pattern matching (`match` expressions)** | ✓ | **8** |
| Algebraic data types, range patterns, exhaustiveness | ⏳ | 9+ |
| Witness variables, private ZK inputs | ⏳ | 9+ |
| Error spans with line/column | ⏳ | 9+ |

### Example: `match` in BudL

```bud
contract MatchExample {
    pub fn main() {
        let x = 0;
        match (x) {
            0 => { emit Result(100); },
            1 => { emit Result(200); },
            _ => { emit Result(999); },
        };
    }
}
```

---

## Quick Start

```bash
cargo check                              # Build check
cargo test --workspace                   # Full test suite (58/58)
RUST_LOG=info cargo run -p bud-cli -- run --program example.bud --sender 1
```

---

## Prover Architecture

```
VM Trace (Vec<Step>)
  +--> trace_matrix() -> 354-column RowMajorMatrix<Goldilocks>
  |     +- Columns 0-64:    Core + Selectors + Register + Memory + Soundness
  |     +- Columns 65-257:  Comparison/Bitwise (64-bit decomposition + eq flags)
  |     +- Columns 258-353: Poseidon (4-round state + S-box intermediates)
  +--> aux_trace_generator() -> 3-column LogUp accumulators (Reg, Mem+Stor, Prog)
  +--> prove_with_preprocessed() -> Proof
  +--> postcard::serialize -> ProofEnvelope
```

| File | Responsibility |
| --- | --- |
| `bud-proof/src/plonky3_air.rs` | Main AIR constraints (eval function, 354 columns) |
| `bud-proof/src/plonky3_prover.rs` | Adapter: trace matrix, aux trace, prove/verify |
| `bud-proof/src/bud_stark/prover.rs` | Prover flow: commit, challenge, quotient, open |
| `bud-proof/src/bud_stark/verifier.rs` | Verification flow |
| `bud-proof/src/bud_stark/folder.rs` | Constraint folders (prover/verifier) |
| `bud-proof/src/bud_stark/config.rs` | StarkGenericConfig type aliases |

---

## Book-Style Documentation

Turkish book-style guide that teaches ZKVM architecture through BudZKVM:

1. [Giriş — ZKVM Nedir?](docs/01_giris.md)
2. [Komut Seti Mimarisi (ISA)](docs/02_isa_ve_bytecode.md)
3. [Sanal Makine İnşası (VM)](docs/03_virtual_machine.md)
4. [ZK Dostu Mimari](docs/04_zk_friendly_architecture.md)
5. [STARK, AIR ve Plonky3](docs/05_stark_ve_plonky3.md)
6. [Derleyici ve Ekosistem](docs/06_compiler_ve_ekosistem.md)
7. [Prover Stabilizasyonu](docs/07_prover_stabilizasyonu_ve_testler.md)
8. [Production Hardening & Soundness](docs/08_production_hardening_ve_soundness.md)
9. [Faz 0 — Stabilizasyon ve Üretime Geçiş](docs/09_faz0_stabilizasyon.md)

Start here: [`docs/README.md`](docs/README.md).

---

## Detailed Roadmap

### Phase 0: Workspace Baseline — complete
- [x] Rust workspace, CI (fmt + clippy `-D warnings` + test)
- [x] Example Bud programs, command matrix

### Phase 1: ISA & Bytecode — complete
- [x] 31 opcodes, deterministic encoding, production/experimental profiles
- [x] All opcodes now production (0 experimental)

### Phase 2: VM Execution Engine — complete
- [x] 64-bit register-based VM, 32 regs, gas metering
- [x] Execution trace with full Step struct
- [x] Poseidon4 hash, VerifyMerkle (64-depth), Storage (HashMap)

### Phase 3: BudL Compiler — complete
- [x] Lexer (logos), recursive-descent parser, sema, codegen
- [x] while/for loops, operator precedence, comments
- [x] User-defined function calls (Call/Ret, caller-saved registers)
- [x] Static Type System (u64/bool/field, return types)
- [x] Struct support and Dynamic Heap Memory (r31 HEAP_PTR)
- [x] **Pattern matching (`match` expressions) — Tur 8**
- [ ] Better error spans (line/column tracking) — Tur 9+

### Phase 4: Plonky3 0.5.2 Prover — complete
- [x] StarkGenericConfig, PCS, challenger wiring
- [x] postcard serialization (bounded, DoS-safe)
- [x] ProofEnvelope with versioning

### Phase 5: AIR Constraint Coverage — complete
- [x] All 31 opcodes constrained: arithmetic, comparison, bitwise,
      poseidon, storage, merkle
- [x] Selector exclusivity, booleanity, PC transitions
- [x] 8 negative soundness tests

### Phase 6: LogUp Cross-Table Lookup — complete
- [x] 3-table LogUp: Register, Memory+Storage, Program CTL
- [x] Storage via STORAGE_BASE address partitioning
- [x] LogUp boundary constraints (first_row=0, last_row=0)

### Phase 7: Proof API & Transport — complete
- [x] ProofEnvelope with version, backend, FRI params
- [x] Public input Keccak256 hash binding
- [x] Bounded deserialization

### Phase 8: CLI & Developer Experience — functional
- [x] Run, prove, batch, deploy, call, verify commands
- [x] Structured tracing (`RUST_LOG`)
- [ ] JSON output mode, trace dump, state inspect

### Phase 9: State, Accounts, L1 Integration — in progress
- [x] File-backed state, 64-depth SMT, account model
- [x] Transactional commit/rollback, atomic state save
- [ ] L1-facing JSON-RPC node APIs

### Phase 10: Performance & Benchmarking — planned
- [ ] Criterion benchmarks, proof size/time tracking
- [ ] Rayon parallelism tuning

### Phase 11: Security Audit — planned
- [ ] External audit, full constraint review, fuzzing

### Phase 12: Docs & Learning Material — active
- [x] 10-chapter Turkish book + 3 developer guides
- [ ] Diagrams, opcode-to-constraint walkthrough, debugging guides

---

## Near-Term Plan (Faz 2 — Compiler Olgunlaştırma)

1. **Error Spans** — line/column numbers in compile errors.
2. **Algebraic Data Types & Exhaustiveness** — structs/enums as pattern
   subjects, range patterns, exhaustiveness checking.
3. **Witness Variables** — private ZK inputs (`witness` keyword).
4. **Match-as-expression** — surface the matched arm's value to `let` /
   `return` bindings.
5. **Benchmark suite** (criterion) — proving/verification time, proof
   size.
6. **Prover parallelism optimization** (Rayon).

---

## Verification Status

```bash
cargo check --workspace --all-targets     # Clean
cargo clippy --workspace -- -D warnings   # Clean
cargo test --workspace                    # 58 passed, 0 failed
cargo test -p bud-compiler                # 9/9 (3 match + 6 baseline)
cargo test -p bud-proof                   # 36 unit + 1 soundness_negative = 37/37
cargo test -p bud-vm                      # 6 unit + 2 trace_fixtures = 8/8
cargo test -p bud-state                   # 4/4
cargo fmt --all -- --check                # Clean
```

CI runs the same three gates on every push to `main`
(`.github/workflows/ci.yml`); the commit status badge is green at
all times for the `main` branch.

---

## Relationship to Budlum

BudZKVM is the ZK execution layer for the sibling Budlum L1 (see
[`lubosruler/budlum`][budlum]). The two repositories are siblings;
`budlum-core/Cargo.toml` consumes the ZK crates via path dependencies:

```toml
# budlum-core/Cargo.toml
bud-isa    = { path = "../BudZKVM/bud-isa" }
bud-vm     = { path = "../BudZKVM/bud-vm" }
bud-proof  = { path = "../BudZKVM/bud-proof" }
```

Cross-domain interaction goes through Budlum's existing
`CrossDomainMessage` primitive — there is no bespoke L1↔ZKVM bridge
protocol. Proof submission is fully permissionless: anyone can submit
a BudZKVM proof and Budlum verifies it natively via `bud_proof` (see
`budlum-core/src/prover/mod.rs`).

For the L1 side, see the [budlum-core repository][budlum] and the
[Budlum documentation book][budlum-book].

[budlum]: https://github.com/lubosruler/budlum
[budlum-book]: https://github.com/lubosruler/budlum/tree/main/docs

---

## License

MIT
