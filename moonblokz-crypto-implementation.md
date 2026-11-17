# MoonBlokz Crypto Implementation Notes

## Purpose of This Document

This document captures implementation-facing implications of the MoonBlokz cryptographic design in `moonblokz-crypto-lib`.

**Status (2026-11-16): built in Story 6.16.** Sections so marked describe the built crate (DEC-235, DEC-244..250); the four earlier backends are removed.

It complements the conceptual and algorithm documents by identifying:

- what the implementation is actually responsible for today,
- how the compile-time backend boundary works,
- what engineering constraints follow from the current API,
- how the individual backends differ,
- what limits are explicitly encoded in the library,
- and which practical cautions should guide future changes.

- Use [moonblokz-crypto-concept.md](./moonblokz-crypto-concept.md) for the conceptual explanation.
- Use [moonblokz-crypto-algorythm.md](./moonblokz-crypto-algorythm.md) for the formal API, constants, serialization rules, and algorithm flows.

## Source Basis

This document is based on:

- `moonblokz-crypto-lib/Cargo.toml`, `src/lib.rs`, `README.md` and `run_tests.sh`
- the ratified Story 6.16 decisions (`_bmad-output/implementation-artifacts/6-16-*.md`, D1–D8) and their evaluation measurements (QEMU `mps2-an385 -icount`, thumbv6m firmware linked without an allocator, `6-16-crypto-eval/`)
- the Project Lead's reference-hardware measurements in the series article *MoonBlokz series part VI — Crypto Algorithms* (2025-05-26), named the reference on 2026-10-28 (DEC-180), superseded on 2026-11-17 by the measured built backends (DEC-249)

## Scope and Intent

This is not a line-by-line code commentary. Instead, it records the engineering consequences of the current library design so that future MoonBlokz work can reuse, extend, or replace the crypto layer without breaking the broader system.

## Core Terminology Used in This Document

To stay aligned with the companion crypto files, this document uses the following terms consistently:

- **Signature** — the ordinary single-signer artifact represented by `Signature`.
- **Aggregation-ready signature** — the single-signer artifact represented by `MultiSignature`, intended to be combined later.
- **Aggregated signature** — the combined evidence artifact represented by `AggregatedSignature`.
- **Algorithm family** — Schnorr or BLS.
- **Backend** — one concrete compile-time-selected implementation, named by its scheme feature.

When discussing exact public code constructs, this document still uses the type names `Signature`, `MultiSignature`, and `AggregatedSignature` directly.

## Relationship to MoonBlokz Architecture

The current crypto crate fits the broader MoonBlokz pattern of keeping critical subsystems modular and replaceable.

For implementation planning, that means:

- the rest of MoonBlokz should depend on the public crypto surface, not backend internals,
- backend choice should remain a build-time concern,
- serialized artifacts should remain compact and predictable,
- and memory behavior should stay bounded and embedded-friendly.

## Business Analyst View: What the Current Implementation Enables

From a delivery perspective, the current implementation provides a usable cryptographic subsystem that already supports:

- ordinary signing and verification,
- aggregation-ready signer contributions,
- aggregated multi-party evidence,
- bounded aggregated-signature payloads,
- a build-time Schnorr / BLS choice,
- and `no_std` compatibility without an allocator.

It also makes one important product-level promise: the surrounding MoonBlokz code can switch crypto backends without changing the high-level application API.

## Current Implementation Responsibilities

The existing code implies these concrete responsibilities.

### 1. Preserve one stable public API across all backends

Every backend must provide compatible implementations for:

- `Crypto`
- `PublicKey`
- `Signature`
- `MultiSignature`
- `AggregatedSignature`
- and their corresponding traits

### 2. Enforce build-time backend exclusivity

The crate must never silently compile multiple crypto implementations into the same public surface.

### 3. Preserve family-specific serialization contracts

Code that changes sizes, field ordering, or aggregate encoding would change protocol-visible behavior.

### 4. Keep aggregation bounded

The library currently enforces:

- maximum aggregate count: `50`

Any change to this affects:

- memory assumptions,
- buffer sizing,
- serialization expectations,
- tests,
- and likely upstream protocol and documentation assumptions.

### 5. Avoid unnecessary allocation-centric APIs

The aggregate serialization model writes into caller-provided buffers rather than returning newly allocated collections. This is part of the current embedded-friendly design and should not be removed casually.

## Public-Surface Engineering Implications

The library uses trait definitions for API consistency, but the embedding project consumes same-named structs re-exported from the active backend.

### Why this matters

This approach avoids forcing the wider codebase to carry:

- trait objects everywhere,
- complicated generic plumbing,
- or backend-specific type names.

### Practical consequence

If a future backend is added, it should preserve this same pattern unless there is a strong architectural reason to redesign the full public surface.

## Backend Inventory and Engineering Meaning

Built in Story 6.16. Only libraries that build and run as `no_std` **without an allocator** qualify (Project Lead, 2026-11-14), so the crate keeps two backends behind three scheme features (DEC-245, DEC-246):

| Module | Features | Library | Notes |
|---|---|---|---|
| `src/schnorr_k256_signer.rs` | `schnorr` (default) | `k256` 0.14 | BIP-340 through `k256::schnorr` (DEC-235); constant-time, NCC-reviewed (2023), no C code; `precomputed-tables` off (31.7 KB of RAM). Our own code is the DEC-194 aggregate only. |
| `src/bls_zkcrypto_signer.rs` | `bls-min-sig`, `bls-min-pk` | zkcrypto `bls12_381` 0.9.2 | The IETF proof-of-possession suite, one generic core for both variants (DEC-247); `sqrt-table` off (it pulls `alloc`). |

Removed: `schnorr-malachite` and `schnorr-num-bigint-dig` (they allocate on every operation; malachite was the crate default), `schnorr-crypto-bigint` (minutes per operation on an RP2040 and not constant-time, DEC-245), and `bls-bls12_381-bls` (forgeable, see the [Crypto Algorithm Model](./moonblokz-crypto-algorythm.md) §BLS Algorithm Family Model), with their dependencies (`malachite-*`, the `num-bigint-dig` git fork, `num-traits`, `crypto-bigint`, `bls12_381-bls`, `dusk-bls12_381`, `dusk-bytes`).

## Architect View: Boundary Discipline in the Current Code

Architecturally, the code separates concerns cleanly into these boundaries:

1. **crate root** — traits, constants, compile-time guards, re-exports, and root tests
2. **Schnorr backend** — BIP-340 through `k256`, the DEC-194 aggregate, serialization
3. **BLS backend** — the generic suite core instantiated per variant, serialization
4. **Cargo feature layer** — exactly-one-scheme selection

This boundary discipline is one of the most important implementation properties to preserve.

## Detailed Implications of the Compile-Time Feature Model

The `Cargo.toml` feature design has real engineering consequences.

Built in Story 6.16 (DEC-246).

### Scheme features

- `schnorr` (default)
- `bls-min-sig`
- `bls-min-pk`

The constants are gated per selected feature. A private `bls` helper feature, enabled by both BLS variants, may gate their shared code; Cargo features are public, so the guards also refuse `bls` without a variant.

### Practical consequences

- tests must run under all three features, not only the default
- downstream users must disable default features when selecting a BLS variant
- every forwarding crate (chain-types, storage, configuration, vote, mempool, blockchain, the node) forwards the same three names, and each builds and passes its tests under all three since Story 6.17 (DEC-255), with the crypto packages at `opt-level = 3` in the dev profile so the BLS legs stay short

## `#![no_std]` and Allocation Discipline

The crate root is explicitly `#![no_std]`.

### Engineering consequence

Any future change that introduces `std` assumptions into the core crate would be a major architectural regression.

### Current allocation style

The library avoids returning heap-allocated objects from the public API for serialized forms.

It also uses:

- fixed arrays for signatures and public keys,
- fixed arrays for stored Schnorr aggregate `r` values,
- fixed upper bounds for aggregate counts,
- and output-buffer-based serialization for aggregated signatures.

That is consistent with microcontroller-friendly design. In the ratified backends no operation allocates, and BLS verification uses two `pairing()` calls because `multi_miller_loop` and `G2Prepared` are `alloc`-gated in `bls12_381`.

## Schnorr Backend Engineering Implications

Built in Story 6.16 (DEC-235, DEC-244).

### One implementation, the library's

Single signatures use `k256`'s BIP-340 path (`sign_prehash` with `aux_rand = 0`, `verify_raw`), not a re-implementation, and the official BIP-340 vectors check it. The earlier three Schnorr backends claimed equivalent behavior but differed in trimming, index encoding and verification; with one backend there is nothing to keep in parallel.

### Byte representations

`PublicKey`, `Signature` and `MultiSignature` keep only their bytes (32 / 64 B), validated in `new` and re-parsed where used (about 1 M instructions, about 7 % of a verify), so a 40-signer `&[&PublicKey]` stays about 1.3 KB.

### The aggregate is our own code

`k256` keeps its `r` / `s` accessors and its tagged hash private, so the module has its own tagged hash and reads `sᵢ` from the signature bytes. The weight reduction `mod (q − 1)`, then `+ 1`, is one conditional subtraction on the 256-bit integer (`2²⁵⁶ < 2(q − 1)`), isolated in a helper with boundary tests at `q − 2`, `q − 1`, `q` and `2²⁵⁶ − 1`, since hashing practically never reaches the subtraction branch.

## Deterministic Nonce Implementation Cautions

BIP-340 with `aux_rand = 0³²` derives the nonce from the private key and the message only (DEC-235, FR63).

### Why this matters in implementation work

Any change to the auxiliary randomness, the tagged-hash conventions or the endianness changes signature outputs and breaks the BIP-340 vectors and the pinned aggregate vectors. Nothing of the earlier little-endian convention survives in the new module: keys, `r`, `s` and the aggregate `count` are big-endian.

`k256`'s `sign_raw` re-verifies under `debug_assertions`, so timing is measured in release builds only.

## Aggregation Engineering Implications

The current aggregation model is more constrained and stateful than a simplistic “combine signatures” API might suggest.

### Important properties

- aggregate input order matters
- aggregate verification expects matching public-key order
- aggregation depends on the message
- Schnorr aggregation stores all `r` components explicitly
- aggregation is rejected when empty, above 50 inputs, or when the signature and key counts differ
- a `count = 0` aggregate is refused at parse (the earlier backends verified an empty aggregate as true)

### Engineering consequence

Any protocol layer that uses aggregated signatures must preserve signer ordering deterministically. If upstream code reorders signer contributions or public keys, aggregate verification can fail even when the individual signatures were valid.

## Serialization Contract Cautions

The library makes serialization formats protocol-visible.

### Schnorr aggregated signatures

Ratified encoding (DEC-235):

- 2-byte big-endian count
- 32-byte big-endian combined `s`
- `count * 32` bytes of `r`

### BLS aggregated signatures

Ratified encoding (DEC-247):

- 2-byte count, big-endian (as in the Schnorr aggregate)
- the aggregated signature: 48 B (`bls-min-sig`) or 96 B (`bls-min-pk`)

### Practical consequence

Changing these formats is not a local refactor. It is a compatibility change that would ripple into any stored or transmitted MoonBlokz artifacts that depend on these bytes.

## Public-Key Reconstruction Cautions in Schnorr

Schnorr public keys are reconstructed from `x` only.

### Engineering consequence

This design is compact, but public-key parsing includes curve reconstruction and validation. In the ratified backend it is `k256`'s BIP-340 `lift_x` (`VerifyingKey::from_bytes`: modulus check, curve membership, even `y`), so the semantics are BIP-340's. An all-zero key, which the blockchain uses to mean "absent", stays safe: `x = 0` is not on secp256k1, and an all-zero BLS key lacks the compression flag.

## BLS Backend Engineering Implications

Built in Story 6.16 (DEC-247, DEC-248). The BLS backend behaves differently enough from Schnorr that engineers should avoid over-generalizing.

### One generic core

A private trait (or a small set of type parameters) names the signature group, the key group, the DST and the pairing argument order; hashing, KeyValidate, signing, aggregation and the pairing check are written once. Only the groups, the DSTs and the sizes differ between `bls-min-sig` and `bls-min-pk`.

### Encoding details

- The private key is big-endian (the suite's I2OSP form), while zkcrypto's `Scalar::from_bytes` is little-endian, so the bytes are reversed.
- `from_compressed` performs the subgroup check but accepts the infinity encoding, so KeyValidate adds an explicit identity rejection.
- Hash-to-curve uses `bls12_381`'s `experimental` RFC 9380 implementation; hash-to-G2 works without `alloc` (only `ExpandMsgXmd::into_vec` needs it).

### Key-validation cost

Decompressing and subgroup-checking a G2 key (min-sig) costs about 50–89 M instructions, paid on every verify that parses a key: a 40-key aggregate check is about 41 s with per-use validation against about 9 s with pre-validated keys (QEMU-based estimates). An LRU cache of validated keys ([Blockchain Architecture](./moonblokz-blockchain-architecture.md) §12.3) is the remedy if it binds; it matters now that the chain uses BLS (Story 6.17).

### Practical consequence

Tests and documentation describe family-specific behavior rather than pretending every backend behaves identically internally.

## Unsafe and Low-Level Code Cautions

Built in Story 6.16: neither backend has `unsafe` code (the earlier BLS backend's `MaybeUninit` aggregation goes), so the crate takes `#![forbid(unsafe_code)]` and its Miri script is removed.

## Dependency and Supply-Chain Implications

The current `Cargo.toml` reveals a few practical maintenance facts.

### Optional dependency model

All crypto backends are behind optional dependencies. This is good for binary minimization and backend isolation.

### Ratified dependency set (Story 6.16, DEC-250)

- `k256` 0.14 (`default-features = false`, `arithmetic`, `schnorr`); `schnorr` pulls `signature` 3 and, through `digest`, the optional `ecdsa` / `der` crates, which LTO strips from the firmware.
- zkcrypto `bls12_381` 0.9.2 (`default-features = false`, `groups`, `pairings`, `experimental`).
- SHA-256: the Schnorr backend hashes with `k256`'s re-exported `sha2` 0.11, and the workspace moves to `sha2` 0.11; `sha2` 0.10 is an optional dependency enabled only by the BLS variants, because `bls12_381`'s `ExpandMsgXmd` is bound to `digest` 0.10.

All sources are crates.io; the `num-bigint-dig` git fork goes. `Cargo.lock` stays untracked.

## Documentation and Metadata Caveat

The current codebase contains a notable inconsistency that should be treated explicitly.

### Identified inconsistency

`Cargo.toml` declares:

- crate license: `MIT`

but repository guidance and older article-oriented documentation discuss licensing concerns around dependent libraries, especially in the Malachite path, which Story 6.16 removes.

### Resolution guidance

This is not necessarily a contradiction, because crate metadata and dependency-license obligations are different things. However, future documentation should continue to distinguish clearly between:

- the license of `moonblokz-crypto` itself,
- and the licenses of enabled dependencies.

No change is made here beyond flagging that distinction explicitly.

## Testing Implications

The root test module in `src/lib.rs` gives several practical lessons.

### Root tests are backend-polymorphic

Most tests are written against the common public API and therefore apply to whichever backend is active.

### Default `cargo test` is not enough

Because only one backend is active at a time and the default is `schnorr`, a plain `cargo test` does not validate the BLS variants.

### Byte-level vectors

Built in Story 6.16: official BIP-340 vectors and pinned DEC-194 aggregate vectors for `schnorr`, RFC 9380 hash-to-curve vectors plus pinned or Ethereum consensus-spec signature vectors for the BLS variants ([Crypto Algorithm Model](./moonblokz-crypto-algorythm.md) §Byte-level vectors). They replace the `schnorr-crypto-bigint` regression vector.

### Engineering consequence

`run_tests.sh` runs the three scheme features, with clippy (`--all-targets -D warnings`) and `cargo fmt --check` per feature; crypto-lib has no CI of its own.

## Performance and Benchmark Interpretation Caution

The current knowledge-base update is based on the codebase rather than benchmark articles.

### What can be stated safely from the code

- the default scheme is `schnorr` (built in Story 6.16)
- BLS remains supported but optional, in two variants

### What should not be overstated from code alone

The codebase by itself does not prove benchmark rankings. Any claim about one backend being faster must be backed by measurements; repeat them when a backend, its arithmetic library or the build profile changes.

### Measured costs of the built backends (RP2040 and QEMU)

**Measured on the reference hardware (DEC-249, 2026-11-17):** Waveshare RP2040-LoRa at 125 MHz (the Embassy default clock), the `test_crypto` `crypto_timing` firmware per scheme (release, `opt-level = "z"`, linked without an allocator, timed with the 1 µs timer, nothing logged during the run; logs in `6-16-crypto-eval/final/hardware-rp2040-*.log`). Instruction counts and painted stacks come from QEMU (`mps2-an385 -icount shift=0`) on the same trait (Story 6.16 AC6). Flash is the `test_crypto` link proof exercising every trait item. Keygen and multi-sign are averages over 40.

| Scheme | Flash | Keygen | Sign (QEMU) | Verify (QEMU) | 40-signer aggregate (QEMU) | 40-signer aggregate verify (QEMU) |
|---|---:|---:|---|---|---|---|
| `schnorr` (`k256` 0.14, BIP-340) | 41 KB | 158 ms | **163 ms** (12.1 M, 5.9 KB stack) | **209 ms** (15.9 M, 10.1 KB) | **39 ms** (1.5 M, 1.1 KB) | **13.3 s**, 332 ms per signer (1.02 G, 6.1 KB; DEC-244) |
| `bls-min-sig` (zkcrypto `bls12_381` 0.9.2) | 98 KB | 2.05 s | **2.02 s** (79 M, 14.5 KB) | **11.7 s** (772 M, 22.5 KB) | **5.85 s** (276 M, 7.2 KB; 31.4 s before AC0) | **51.3 s** (3.56 G, 26.0 KB; keys decompressed per use) |
| `bls-min-pk` (zkcrypto `bls12_381` 0.9.2) | 100 KB | 1.32 s | **4.07 s** (299 M, 41.3 KB) | **12.7 s** (895 M, 33.6 KB) | **20.0 s** (1.46 G, 18.0 KB; 39.6 s before AC0) | **42.5 s** (1.93 G, 35.6 KB) |

Key decompression (QEMU only): 1.0 M / 1.6 KB (Schnorr), 71 M / 22.2 KB (min-sig, G2), 26 M / 9.8 KB (min-pk, G1). The proof of possession costs exactly a signature to make and a verification to check (measured: 163 / 209 ms, 2.02 / 11.7 s, 4.07 / 12.7 s).

What the measurement shows:

- **About 1.65–1.9 cycles per instruction** for most operations, so the QEMU counts predict RP2040 time well; operations dominated by square roots and inversions run at 3.2–3.7 (min-sig signing's hash to G1, min-sig aggregation, min-pk keygen and aggregate verify at 2.75–3.3).
- **Schnorr is 5.5–9× faster than DEC-180's malachite figures** (sign 0.9 s → 163 ms, verify 1.9 s → 209 ms, aggregate verify 2.9 s → 332 ms per signer) and links without an allocator.
- **Aggregation saves space, not time, on Schnorr** (332 ms per signer against 209 ms for a separate verification); on BLS it saves both (min-sig: 51.3 s for 40 against about 7.8 min verified one by one).
- **BLS on the RP2040 is slow:** a single verification takes 11.7–12.7 s, so a full block (13 transfers under `bls-min-pk`, 22 under `bls-min-sig`, plus its block signature) takes about 3–4.5 minutes to verify. Story 6.17's RP2040 BLS profiles carry these numbers.
- **BLS aggregation re-validated every contribution** (the subgroup check inside the aggregation) although `MultiSignature::new` had already validated it: most of the 31–40 s for 40 was that repeated check. Story 6.17 AC0 dropped it (`from_compressed_unchecked`, relying on the `MultiSignature` type invariant; the identity sum is still refused; O-040). QEMU, 40 contributions: `bls-min-sig` 1 063 640 960 → 275 710 920 instructions (stack 9 768 → 7 176 B), 3.86× fewer; `bls-min-pk` 2 858 314 640 → 1 460 960 360 (22 656 → 17 952 B), 1.96× fewer (`6-17-ac0/`). On the RP2040 (`6-17-ac0/hardware-rp2040-bls-*.log`), `bls-min-sig` dropped from 31.4 s to 5.85 s (5.37×) and `bls-min-pk` from 39.6 s to 20.0 s (1.98×); the other operations moved by at most +0.7 % (code layout, untouched paths).
- `bls-min-pk` moves hash-to-curve into G2, so signing and its stack grow about 2× and 3× against `bls-min-sig`. On the chain its deepest root, `process_genesis` (which signs), peaks at 56 544 B (55.2 KB) against 39 840 B (38.9 KB) for min-sig, so its RP2040 Core 1 is 70 KiB against 49 KiB, and min-pk fits fewer nodes than min-sig (211 against 330; [System Constraints](./moonblokz-system-constraints.md) §3.4).

### Earlier measurements (DEC-180, superseded)

These are DEC-180's figures: the earlier backends, measured with an allocator. No node build uses them (every firmware build links without an allocator, and the node never ran malachite). **Superseded on 2026-11-17 by DEC-249's hardware run** (the table above); kept as history.

The Project Lead measured the backends on the reference hardware and published the results in the series article *MoonBlokz series part VI — Crypto Algorithms* (2025-05-26). On 2026-10-28 the Project Lead named these the current reference (DEC-180). Hardware: Waveshare RP2040-LoRa (Raspberry Pi Pico 1 class, dual-core 133 MHz, 264 KB SRAM). Test project: Embassy firmware, 10 signers, SHA-256, message `"Hello, world!"`. "Aggregate" is the aggregation of 10 multi-signatures, and "Verify aggregated" is the verification of that aggregate.

| Backend (profile, binary) | Sign | Verify | Multi-sign | Aggregate (10) | Verify aggregated (10) |
|---|---:|---:|---:|---:|---:|
| `bls-bls12_381-bls` (`opt-level = "z"`, 220 KB) | 4.6 s | 13.1 s | 6.9 s | 3 s | 66 s |
| `bls-bls12_381-bls` (`"s"`, 428 KB) | 7 s | 17.1 s | 10.5 s | 3 s | 76 s |
| `bls-bls12_381-bls` (`3`, 718 KB) | 7.8 s | 19.6 s | 11.7 s | 3 s | 85 s |
| `schnorr-num-bigint-dig` (`"z"`, 123 KB) | 2.7 s | 5.4 s | 2.7 s | 0 s | 85 s |
| `schnorr-num-bigint-dig` (`"s"`, 181 KB) | 2 s | 4.1 s | 2 s | 0 s | 65 s |
| `schnorr-num-bigint-dig` (`3`, 242 KB) | 2 s | 4.1 s | 2 s | 0 s | 63 s |
| `schnorr-malachite` (`"z"`, 596 KB) | 0.9 s | 1.9 s | 0.9 s | 0 s | 29 s |
| `schnorr-malachite` (`"s"`, 576 KB) | 0.7 s | 1.4 s | 0.7 s | 0 s | 22 s |
| `schnorr-malachite` (`3`, 1,124 KB) | 1 s | 2.1 s | 1 s | 0 s | 33 s |

What followed from them:

- **The KB's derived limits used the `schnorr-malachite` `"z"` row** (DEC-180) until 2026-11-17: about 1.9 s per verification and 2.9 s per signer for an aggregate; system constraints §2.1 now uses the measured `k256` numbers. The `"s"` profile was about 25 % faster.
- **Aggregation saved space, not time** on that backend (2.9 s against 1.9 s per signer).
- **The `bls12_381-bls` rows are of the forgeable backend** that Story 6.16 removes.
- **Acceleration is deferred to a later development phase (O-032).**

## Feasibility Boundaries for Future Change

### Safe kinds of future change

- adding more tests for existing semantics
- improving internal arithmetic performance without changing external behavior
- adding a new backend while preserving the public surface and the no-allocator rule
- tightening validation where it does not alter valid serialized artifacts

### High-risk kinds of future change

- changing serialization formats
- changing tagged-hash input conventions
- changing signer-order sensitivity in aggregation
- removing the aggregate bound without redesigning memory expectations
- introducing `std` assumptions into the core crate
- a Schnorr implementation beside `k256`'s BIP-340 path (DEC-245)

## Related Documents

- [`moonblokz-crypto-concept.md`](./moonblokz-crypto-concept.md) — the conceptual role and design philosophy of the crypto subsystem.
- [`moonblokz-crypto-algorythm.md`](./moonblokz-crypto-algorythm.md) — the formal public API, constants, backends, and aggregation behavior this code realizes.
