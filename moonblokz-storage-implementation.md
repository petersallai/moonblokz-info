# MoonBlokz Storage Implementation Notes

## Purpose of This Document

This document captures implementation-facing implications of MoonBlokz storage **as currently implemented in `moonblokz-storage`**, with special attention to the storage-facing contracts defined by `moonblokz-chain-types`.

It complements the conceptual and algorithm storage documents by identifying:

- what the current crate is responsible for today,
- how the repository is structured,
- how the feature model shapes the build and runtime behavior,
- what engineering consequences follow from the two current backend implementations,
- how the RP2040 backend encodes flash geometry and host-side testing,
- how the canonical block and hash crate constrains storage engineering,
- and which implementation boundaries remain explicitly open.

- Use [moonblokz-storage-concept.md](./moonblokz-storage-concept.md) for the strategic role and current conceptual trade-offs of MoonBlokz storage.
- Use [moonblokz-storage-algorythm.md](./moonblokz-storage-algorythm.md) for the formal public API, control-plane record model, slot/page mapping rules, chain-types-derived geometry, and error semantics.

## Source Basis

This document is grounded primarily in the current repositories, especially:

- `moonblokz-storage/Cargo.toml`
- `moonblokz-storage/README.md`
- `moonblokz-storage/src/lib.rs`
- `moonblokz-storage/src/error.rs`
- `moonblokz-storage/src/backend_memory.rs`
- `moonblokz-storage/src/backend_rp2040.rs`
- `moonblokz-storage/src/conformance.rs`
- `moonblokz-storage/docs/prd.md`
- `moonblokz-storage/docs/architecture.md`
- `moonblokz-chain-types/README.md`
- `moonblokz-chain-types/src/lib.rs`
- `moonblokz-chain-types/src/block.rs`
- `moonblokz-chain-types/src/hash.rs`
- `moonblokz-chain-types/src/error.rs`
- `moonblokz-chain-types/docs/block-data-structure.md`

Where the earlier Part VIII article diverges from the current repositories, this document treats the codebases as authoritative.

## Scope and Intent

This is not a line-by-line source tour. Instead, it records the engineering consequences of the current implementation so future work can preserve current invariants and avoid silently re-expanding the crate into responsibilities it does not currently own.

## Current Repository Shape

The current repository is compact and intentionally narrow.

### Public contract modules

- `src/lib.rs` — feature gating, public constants, `ControlPlaneData`, `StorageTrait`, backend exports and selected-backend alias
- `src/types.rs` — canonical public type aliases such as `StorageIndex`
- `src/error.rs` — public error model

### Backend modules

- `src/backend_memory.rs` — in-memory backend for off-target integration and contract testing
- `src/backend_rp2040.rs` — RP2040 backend with actual flash-geometry model and host/mock support

### Validation module

- `src/conformance.rs` — backend-agnostic conformance tests for shared public semantics

### Supporting repository material

- `README.md` — integration guidance, feature rules, error-code map, example usage
- `docs/prd.md` and `docs/architecture.md` — BMAD-generated product and architecture intent for the crate
- `examples/` — host and embedded example projects
- `scripts/` and GitHub workflow — backend matrix support and CI automation

## Feature and Dependency Model

## Exactly one backend is selected at build time

The current crate uses feature-based backend selection with compile-time exclusivity:

- `backend-memory`
- `backend-rp2040`

This has several engineering consequences:

- builds remain small and target-specific,
- backend code is not selected dynamically at runtime,
- and adding a new backend must preserve this exclusivity model.

## Default feature set

The crate currently defaults to:

- `backend-memory`
- `schnorr`

The `schnorr` feature enables `moonblokz-crypto/schnorr-crypto-bigint`.

This means the storage crate’s default development posture is a host-friendly memory backend rather than embedded flash.

## Dependency boundaries

The current crate depends on:

- `moonblokz-chain-types` for `Block`, `MAX_BLOCK_SIZE`, `HEADER_SIZE`, `MAX_PAYLOAD_SIZE`, `HASH_SIZE`, and `calculate_hash`
- `moonblokz-crypto` for `PRIVATE_KEY_SIZE`
- `embassy-rp` only on ARM targets for the RP2040 backend

Engineering consequence:

- canonical block serialization and hashing stay outside storage,
- storage consumes those contracts rather than redefining them,
- and embedded flash integration is isolated behind target-specific dependencies.

## Chain-Types Boundary Engineering Consequences

## Storage is tied to one canonical immutable block representation

`moonblokz-chain-types` defines `Block` as an immutable wrapper over a fixed `[u8; MAX_BLOCK_SIZE]` internal buffer plus a logical length.

Engineering consequence:

- storage backends do not need their own canonical block serializer,
- `serialized_bytes()` is the authoritative byte sequence for storage-facing round-tripping,
- and any future change to `Block` binary layout is a storage compatibility event.

## Current block-size constants directly shape storage geometry

The current storage geometry is downstream of the canonical block and hash constants defined in `moonblokz-chain-types`.

Engineering consequence:

- memory-backend slot size is defined by the canonical maximum block size,
- RP2040 slot payload size and total slot size are defined by the canonical block-size and hash-size contracts,
- and RP2040 page packing therefore depends directly on upstream chain-types constants.

The exact constants and derived geometry belong primarily in [moonblokz-storage-algorythm.md](./moonblokz-storage-algorythm.md).

## The version invariant is operationally important

The chain-types crate explicitly reserves `version == 0` for storage empty-slot markers and rejects zero-version blocks in `Block::from_bytes()` and `BlockBuilder::build()`.

Engineering consequence:

- the memory backend’s `slot[0] == 0` empty-slot rule is safe only because of this upstream invariant,
- and changing that invariant would require coordinated storage redesign.

## `Block::from_bytes()` defines current storage parseability

The chain-types parser currently enforces the canonical structural entry conditions for a stored block.

Engineering consequence:

- storage read success today means “the canonical block parser accepted the current byte shape,” not “full blockchain semantic validity was proven,”
- and storage docs should not overstate what block parsing currently guarantees.

The exact parser constraints belong primarily in [moonblokz-storage-algorythm.md](./moonblokz-storage-algorythm.md).

## Hashing semantics have two engineering uses

`moonblokz-chain-types` documents `calculate_hash(block.serialized_bytes())` as the normal canonical hashing pattern.

The RP2040 backend uses the same hash function but applies it to the fixed 2016-byte slot block area.

Engineering consequence:

- there is one canonical hash algorithm,
- but storage uses it in a storage-integrity context rather than only in a logical block-identity context,
- so documentation must distinguish “canonical algorithm” from “exact byte range hashed in one backend.”

## Public API Engineering Consequences

## Synchronous `no_std` contract is a hard constraint

The trait remains synchronous and `no_std`.

Engineering consequence:

- callers must integrate storage into synchronous control flow,
- backends cannot assume async runtime ownership,
- and design choices that require asynchronous coordination belong outside the current crate boundary.

## `ControlPlaneData` is intentionally minimal

The public control-plane data struct omits convenience derives such as `Debug` and `Clone` in production builds.

Engineering consequence:

- embedded binary-size discipline is a first-class implementation concern,
- and future additions to public data models should be justified carefully.

## Error model is intentionally coarse but typed

The public error enum uses a small number of semantically meaningful categories plus backend-local numeric I/O codes.

Engineering consequence:

- contract users can write deterministic recovery logic against the named variants,
- while backend maintainers still retain a narrow escape hatch for backend-specific failures.

## Memory Backend Engineering Consequences

## Storage is byte-array-backed and compile-time sized

`MemoryBackend<const STORAGE_SIZE: usize>` stores one flat byte array.

Engineering consequence:

- all geometry is compile-time predictable,
- capacity is deterministic,
- and tests can use exact-size fixtures to validate slot boundaries.

## Control-plane region is packed, block plane is slot-linear

Unlike the RP2040 backend, the memory backend reserves only the exact control-plane entry bytes it needs, not full 4 KB pages.

Engineering consequence:

- the memory backend is contract-oriented rather than hardware-faithful,
- and it should not be mistaken for a flash-geometry simulator.

## Block storage is zero-cleared and parseability-based

The memory backend zeros the target slot before writing a block, and later treats `slot[0] == 0` as empty.

Engineering consequence:

- slot emptiness depends on the chain-types non-zero block-version invariant,
- and corruption detection is weaker than the RP2040 backend because the memory backend relies on parseability rather than stored-hash verification.

## Fixed-slot parsing is currently accepted by chain-types

The memory backend reads blocks by passing the entire fixed slot buffer into `Block::from_bytes()`.

Engineering consequence:

- fixed-slot padding is currently tolerated by the canonical parser in this backend path,
- and any future tightening of `Block::from_bytes()` could change memory-backend read behavior.

This is a real compatibility-sensitive boundary between the two crates.

## Replica repair happens in-memory too

The memory backend implements the same “first valid replica, then repair invalid replicas” logic as the RP2040 backend.

Engineering consequence:

- control-plane semantics are intentionally kept aligned across backends even when block-plane integrity behavior differs.

## RP2040 Backend Engineering Consequences

## Geometry is page-centric even though the API is slot-centric

The RP2040 backend exposes indexed slot operations publicly, but internally it rewrites full 4096-byte pages.

Engineering consequence:

- one logical slot write is implemented as read-page, mutate-slot, erase-page, write-page,
- page buffering is unavoidable in the current design,
- and write amplification exists at page granularity.

## `data_storage_start_address` is a critical integration parameter

The RP2040 backend does not own the full flash map. Instead, it receives a start address for the region reserved to MoonBlokz storage.

Engineering consequence:

- the embedding firmware must allocate storage space explicitly,
- must keep that start address page-aligned,
- and must account for three reserved control-plane pages at the start of the storage region.

## Hash metadata is stored per slot, not per page as one summary

The current backend stores one hash next to each fixed-size block area.

Engineering consequence:

- integrity verification is local to each slot,
- not tied to a page-level manifest,
- and partial-slot corruption can be detected independently of other slots in the same page.

## RP2040 hash range is fixed-size, not logical-length

The current backend computes the stored hash over the full fixed-size block region, not just the logical `serialized_bytes()` prefix.

Engineering consequence:

- the stored integrity hash covers the padded slot representation,
- and this should remain explicit in any future optimization or migration work.

## Empty-flash semantics are represented as `0xFF`

The RP2040 backend treats all-`0xFF` slots as absent, which matches erased-flash expectations.

Engineering consequence:

- startup and retrieval behavior distinguish erased flash from malformed persisted data,
- and test fixtures can model partial writes by manually constructing non-`0xFF` slots with broken hashes.

## Host-side mock flash is part of the implementation strategy

On non-ARM or test builds, the RP2040 backend uses an in-memory `MockFlash` with the same read/erase/write surface.

Engineering consequence:

- most geometry, integrity, and error-path logic can be tested off-device,
- while the public backend contract remains the same,
- and backend-local error codes 230–232 are reserved for mock-flash failures.

## Control-plane Implementation Consequences

## Control-plane layout is duplicated intentionally across backends

Both backends independently serialize and deserialize a logically equivalent control-plane record rather than sharing one backend implementation helper.

Engineering consequence:

- backend isolation is preserved,
- but compatibility between the two implementations must be maintained intentionally through tests and review.

## Control-plane compatibility is tied to runtime constants

Persisted `PRIVATE_KEY_SIZE`, `INIT_PARAMS_SIZE`, and `MAX_BLOCK_SIZE` are checked against current binary constants.

Engineering consequence:

- changing these constants is a persisted-data compatibility event,
- and storage docs, release notes, and migration expectations should treat such changes seriously.

## Chain configuration storage uses canonical block round-tripping

Both backends reconstruct the saved configuration as `Block::from_bytes(block.serialized_bytes())` before persisting.

Engineering consequence:

- this preserves canonical serialized-byte behavior,
- and it avoids silently storing a block value that cannot round-trip through the canonical block parser.

## Testing and Conformance Consequences

## Shared conformance focuses on public semantics, not full mechanical identity

The current conformance tests validate:

- round-trip save/read behavior,
- absent-slot behavior,
- invalid-index behavior,
- capacity-boundary behavior,
- mixed startup-scan style reads.

Engineering consequence:

- public behavior parity is the current required invariant,
- but backend internals are still allowed to differ where explicitly intended.

## RP2040 tests go beyond public conformance

The RP2040 backend includes tests for:

- page-boundary mapping,
- hash mismatch detection,
- malformed slot detection,
- partial-slot detection,
- control-plane repair,
- and start-address alignment validation.

Engineering consequence:

- the RP2040 backend currently carries the stronger integrity specification in practice,
- and future backends should document clearly whether they match RP2040-style integrity or only shared conformance semantics.

## Example and Distribution Consequences

## Examples encode the intended integration flow

The README and example projects reinforce a simple pattern:

1. call `load_control_data()`
2. if uninitialized, call `init(...)`
3. save blocks by index
4. read blocks back by index

Engineering consequence:

- this is the current intended use shape for integrators,
- and documentation should continue to present storage as a direct chain-runtime dependency rather than an autonomous storage service.

## Git dependency is still the primary integration model

The README states Git dependency as the recommended current integration path, with crates.io as a later phase.

Engineering consequence:

- API stability matters now, even before registry publication,
- and repository consumers may track commit-level changes closely.

## Important Current Boundaries and Risks

## The crate does not yet implement full article-era storage policy

The current implementation does not manage:

- bounded-chain movement,
- automatic replacement policy,
- replay scheduling,
- wear-leveling policy beyond current page/slot mapping,
- or network recovery of corrupted blocks.

These remain external responsibilities.

## Page rewrite cost is real on RP2040

The current RP2040 implementation’s correctness depends on whole-page buffering and rewrite.

Engineering consequence:

- future optimization work must preserve hash correctness, page mapping, and control-plane reservation semantics,
- and must not silently change slot layout or absent-slot detection.

## Persisted constant changes are compatibility-sensitive

Because control-plane compatibility checks include embedded size constants, future changes to core constants can strand old persisted control data unless a migration strategy is added.

## Chain-types changes are storage compatibility events

Because storage depends directly on canonical block layout, version invariants, and hash size:

- changes to `MAX_BLOCK_SIZE`,
- changes to block parsing strictness,
- changes to the `version == 0` reservation,
- or changes to the canonical hash contract

are all storage-relevant compatibility events and should be treated as such in both documentation and release planning.
