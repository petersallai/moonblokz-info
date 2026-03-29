# MoonBlokz Crypto Implementation Notes

## Purpose of This Document

This document captures implementation-facing implications of the current MoonBlokz cryptographic design as implemented in `moonblokz-crypto-lib`.

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

This document is based on the current contents of:

- `moonblokz-crypto-lib/Cargo.toml`
- `moonblokz-crypto-lib/src/lib.rs`
- `moonblokz-crypto-lib/src/schnorr_malachite_signer.rs`
- `moonblokz-crypto-lib/src/schnorr_num_bigint_dig_signer.rs`
- `moonblokz-crypto-lib/src/schnorr_crypto_bigint_signer.rs`
- `moonblokz-crypto-lib/src/bls_bls12_381_bls_signer.rs`
- `moonblokz-crypto-lib/README.md`
- `moonblokz-crypto-lib/run_tests.sh` as referenced by the repository guidance

## Scope and Intent

This is not a line-by-line code commentary. Instead, it records the engineering consequences of the current library design so that future MoonBlokz work can reuse, extend, or replace the crypto layer without breaking the broader system.

## Core Terminology Used in This Document

To stay aligned with the companion crypto files, this document uses the following terms consistently:

- **Signature** — the ordinary single-signer artifact represented by `Signature`.
- **Aggregation-ready signature** — the single-signer artifact represented by `MultiSignature`, intended to be combined later.
- **Aggregated signature** — the combined evidence artifact represented by `AggregatedSignature`.
- **Algorithm family** — Schnorr or BLS.
- **Backend** — one concrete compile-time-selected implementation.

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
- multiple backend options,
- and `no_std` compatibility.

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

The current crate contains four concrete backends.

### `schnorr-malachite`

- default backend
- big-integer implementation using `malachite-base` and `malachite-nz`
- closely mirrors the general Schnorr implementation model used by the older backends

### `schnorr-num-bigint-dig`

- alternate big-integer Schnorr implementation
- depends on a git source rather than only registry crates
- preserves the same public behavior with different arithmetic machinery

### `schnorr-crypto-bigint`

- fixed-width arithmetic Schnorr implementation using `crypto-bigint`
- now present in the codebase and must be treated as a real supported backend
- includes backend-specific regression vectors in the root test module

### `bls-bls12_381-bls`

- wrapper-style BLS backend
- delegates the core signature math to the external BLS ecosystem
- uses much thinner backend code than the Schnorr implementations

## Architect View: Boundary Discipline in the Current Code

Architecturally, the code separates concerns cleanly into these boundaries:

1. **crate root** — traits, constants, compile-time guards, re-exports, and root tests
2. **Schnorr backends** — curve arithmetic, deterministic signing, aggregation logic, serialization
3. **BLS backend** — wrapper logic, public-key aggregation, serialization
4. **Cargo feature layer** — exactly-one-backend selection

This boundary discipline is one of the most important implementation properties to preserve.

## Detailed Implications of the Compile-Time Feature Model

The `Cargo.toml` feature design has real engineering consequences.

### Internal family features

- `schnorr`
- `bls`

These let the crate root define family-level constants and conditional exports.

### Concrete backend features

- `schnorr-malachite`
- `schnorr-num-bigint-dig`
- `schnorr-crypto-bigint`
- `bls-bls12_381-bls`

### Practical consequences

- CI must test more than the default feature
- downstream users must disable default features when selecting a non-default backend
- docs and examples should keep reminding readers that exactly one backend feature is required

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

That is consistent with microcontroller-friendly design.

## Schnorr Backend Engineering Implications

The three Schnorr backends are structurally similar but not identical.

### Shared engineering behavior

All current Schnorr backends implement:

- secp256k1-style point arithmetic,
- `x`-only public-key serialization,
- deterministic nonce derivation via tagged SHA-256,
- identical public trait behavior,
- and the same deterministically weighted aggregation scheme.

### Important code-maintenance consequence

If one Schnorr backend changes behavior and the others do not, the stable public API alone will not protect semantic consistency. Future changes should therefore treat the three Schnorr backends as one behavioral family that needs parallel maintenance.

## Deterministic Nonce Implementation Cautions

The Schnorr backends all derive nonces from:

- tag,
- private key bytes,
- message,
- counter.

### Why this matters in implementation work

Any change to:

- hash input order,
- tag bytes,
- byte trimming behavior,
- endianness,
- or zero-retry logic

would change signature outputs and potentially break compatibility or regression expectations.

### Additional caution

The `schnorr-crypto-bigint` backend trims some little-endian byte slices before challenge and aggregation hashing, while the older big-integer backends naturally derive variable-length byte arrays from their integer types.

This means byte-level compatibility between Schnorr backends should never be assumed without explicit tests.

## Aggregation Engineering Implications

The current aggregation model is more constrained and stateful than a simplistic “combine signatures” API might suggest.

### Important properties

- aggregate input order matters
- aggregate verification expects matching public-key order
- aggregation depends on the message
- Schnorr aggregation stores all `r` components explicitly
- aggregation is rejected when empty
- aggregation is rejected above 50 inputs

### Engineering consequence

Any protocol layer that uses aggregated signatures must preserve signer ordering deterministically. If upstream code reorders signer contributions or public keys, aggregate verification can fail even when the individual signatures were valid.

## Serialization Contract Cautions

The library makes serialization formats protocol-visible.

### Schnorr aggregated signatures

Current encoding is:

- 2-byte little-endian count
- 32-byte combined `s`
- `count * 32` bytes of `r`

### BLS aggregated signatures

Current encoding is:

- 2-byte little-endian count
- 48-byte aggregated signature

### Practical consequence

Changing these formats is not a local refactor. It is a compatibility change that would ripple into any stored or transmitted MoonBlokz artifacts that depend on these bytes.

## Public-Key Reconstruction Cautions in Schnorr

Schnorr public keys are reconstructed from `x` only.

### Engineering consequence

This design is compact, but it means public-key parsing includes curve reconstruction logic and validation.

Future changes must preserve:

- modulus checks,
- curve-membership checks,
- and even-`y` normalization behavior,

or else deserialization and verification semantics may drift.

## BLS Backend Engineering Implications

The BLS backend behaves differently enough from Schnorr that engineers should avoid over-generalizing.

### Thinner implementation

Most of the complex mathematics are delegated to external library types.

### Different role mapping in implementation terms

In BLS:

- `Signature` and `MultiSignature` are genuinely different wrapper types
- `AggregatedSignature` stores a BLS multisignature internally

### Verification model

Aggregated verification depends on public-key aggregation through `MultisigPublicKey::aggregate`.

### Practical consequence

If BLS remains supported, tests and documentation must continue to describe family-specific behavior rather than pretending every backend behaves identically internally.

## Unsafe and Low-Level Code Cautions

The BLS backend uses `MaybeUninit` plus raw-slice reconstruction to aggregate signatures and public keys without heap allocations.

### Engineering consequence

This is efficient and compatible with the crate’s bounded-memory goals, but it means future modifications in those areas require extra care.

### Specific caution

Any change to the initialization logic or slice length calculations in those `MaybeUninit` paths should be reviewed carefully because mistakes there would create memory-safety risks even inside an otherwise safe Rust crate.

## Dependency and Supply-Chain Implications

The current `Cargo.toml` reveals a few practical maintenance facts.

### Optional dependency model

All crypto backends are behind optional dependencies. This is good for binary minimization and backend isolation.

### Git dependency

`schnorr-num-bigint-dig` depends on:

- `num-bigint-dig = { git = "https://github.com/petersallai/num-bigint.git", ... }`

This introduces a stronger maintenance and reproducibility dependency than pure crates.io sourcing.

### Mixed ecosystem surface

The crate spans multiple arithmetic ecosystems:

- Malachite
- num-bigint-dig
- crypto-bigint
- bls12_381-bls and dusk libraries

That increases replaceability, but also increases maintenance surface area.

## Documentation and Metadata Caveat

The current codebase contains a notable inconsistency that should be treated explicitly.

### Identified inconsistency

`Cargo.toml` declares:

- crate license: `MIT`

but repository guidance and older article-oriented documentation discuss licensing concerns around dependent libraries, especially in the Malachite path.

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

Because only one backend is active at a time and the default is `schnorr-malachite`, a plain `cargo test` does not validate the non-default backends.

### `schnorr-crypto-bigint` has extra regression coverage

That backend currently has explicit expected-byte regression vectors for signatures and aggregates.

### Engineering consequence

Future CI or release validation should include all supported backend feature combinations, not just the default backend.

## Performance and Benchmark Interpretation Caution

The current knowledge-base update is based on the codebase rather than benchmark articles.

### What can be stated safely from the code

- the default backend is `schnorr-malachite`
- BLS remains supported but optional
- multiple Schnorr arithmetic backends are maintained

### What should not be overstated from code alone

The current codebase by itself does not prove fresh benchmark rankings for all backends. Any claim about one backend being faster today must be backed by current measurements, not only by earlier article context.

This matters because the code now includes a `schnorr-crypto-bigint` backend that earlier documentation did not cover.

## Feasibility Boundaries for Future Change

### Safe kinds of future change

- adding more tests for existing semantics
- improving internal arithmetic performance without changing external behavior
- adding a new backend while preserving the public surface
- tightening validation where it does not alter valid serialized artifacts

### High-risk kinds of future change

- changing serialization formats
- changing tagged-hash input conventions
- changing signer-order sensitivity in aggregation
- removing the aggregate bound without redesigning memory expectations
- introducing `std` assumptions into the core crate
- drifting the three Schnorr backends into semantically different behaviors

## Technical Writer View: How to Use This File

Read this file when you want to answer questions such as:

- What engineering constraints follow from the current crypto library?
- What should be preserved if a backend is optimized or replaced?
- Which details are protocol-visible versus purely internal?
- Where are the main maintenance and compatibility risks?
- What special care is required around aggregation, serialization, and low-level backend code?

## Related Documents

- [moonblokz-crypto-concept.md](./moonblokz-crypto-concept.md)
- [moonblokz-crypto-algorythm.md](./moonblokz-crypto-algorythm.md)
