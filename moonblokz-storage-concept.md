# MoonBlokz Storage Concept Model

## Purpose of This Document

This document explains the conceptual operating model of MoonBlokz storage as reflected by the current `moonblokz-storage` and `moonblokz-chain-types` codebases.

Its purpose is to describe:

- why storage is modeled as a narrow embedded contract rather than as a general database layer,
- how the current crate separates control-plane persistence from indexed block persistence,
- why backend portability is implemented through compile-time feature selection,
- how the canonical block and hash contracts from `moonblokz-chain-types` constrain storage behavior,
- how the current integrity and recovery model differs between the in-memory and RP2040 backends,
- and where the current implementation narrows, sharpens, or differs from the earlier Part VIII article framing.

This file is intentionally focused on conceptual understanding rather than exact byte offsets or repository-internal implementation walkthroughs.

- Use this document to understand **what the current storage crate is for**, **why its public contract is deliberately small**, and **which storage trade-offs and boundaries MoonBlokz currently treats as architectural rather than merely mechanical details**.
- Use [moonblokz-storage-algorythm.md](./moonblokz-storage-algorythm.md) for the formal API contract, slot/page mapping rules, control-plane record structure, error outcomes, backend behavior, and the exact chain-types-derived storage constraints.
- Use [moonblokz-storage-implementation.md](./moonblokz-storage-implementation.md) for engineering consequences of the current crate structure, feature model, RP2040 flash implementation, mock-flash testing strategy, and remaining open limits.

## Source Basis

This document is grounded primarily in the current repositories:

- `moonblokz-storage/Cargo.toml`
- `moonblokz-storage/README.md`
- `moonblokz-storage/src/lib.rs`
- `moonblokz-storage/src/error.rs`
- `moonblokz-storage/src/types.rs`
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

Where the earlier Part VIII article and the current code differ, this document treats the current codebases as authoritative and calls the difference out explicitly.

## Relationship to Earlier MoonBlokz Documents

This file should be read after:

- [moonblokz-overview.md](./moonblokz-overview.md), which explains the project mission and operating environment,
- [moonblokz-technology.md](./moonblokz-technology.md), which introduces the embedded architectural constraints,
- and preferably after the blockchain documents, because MoonBlokz storage still exists to support bounded blockchain recovery on constrained nodes.

This document answers the conceptual question:

**What kind of storage subsystem does MoonBlokz currently implement when storage must remain deterministic, `no_std`-compatible, backend-portable, and narrow enough to keep chain policy outside the storage crate while still depending on one canonical block and hash contract?**

## Business Analyst View: What Problem the Current Crate Solves

The current `moonblokz-storage` crate is not a full blockchain database and not a general flash-management framework. It solves a narrower product problem:

- let MoonBlokz chain logic initialize node-local control data,
- persist accepted blocks at deterministic indexes,
- retrieve them later through a synchronous API,
- detect important corruption cases explicitly,
- and keep the public contract stable across multiple backend implementations.

This is important because the current product boundary is much sharper than the article-era conceptual discussion. The storage crate is designed to support chain/runtime integration, not to own higher-level blockchain retention policy.

In practice, the crate now focuses on five user-visible capabilities:

- control-plane initialization,
- control-plane recovery and validation,
- indexed block persistence,
- indexed block retrieval,
- and deterministic capacity reporting.

## Why the Current Design Is a Contract Crate First

The README and source code both frame `moonblokz-storage` as a **storage contract crate**.

That conceptual choice matters. The primary artifact is not only the RP2040 storage behavior. It is the shared storage contract exposed through `StorageTrait`, together with conformance expectations across backends.

So the crate should be understood as having two layers of purpose:

1. define a minimal storage contract for chain/runtime code,
2. provide backend implementations that honor that contract.

This is a narrower and more implementation-driven framing than the earlier article’s broader storage-system narrative.

## Why the Storage Crate Depends on a Canonical Chain-Types Boundary

The current storage design depends on a separate canonical crate, `moonblokz-chain-types`, for the most important storage-facing facts about a block.

That crate defines:

- the immutable `Block` representation,
- the canonical binary layout,
- the exact maximum block size,
- the exact header size,
- the rule that valid block version must be non-zero,
- and the canonical SHA-256 hashing contract.

Conceptually, this means storage does **not** own the meaning of a block. It owns persistence mechanics around a block type whose byte layout and validation rules are authoritative elsewhere.

This is one of the key reasons the current storage scope remains narrow and stable.

## Why Canonical Serialized Bytes Matter Conceptually

In the current design, `Block` is already an immutable wrapper around canonical serialized bytes.

That means storage is not fundamentally persisting an abstract object graph and then serializing it later. Instead, storage is persisting a representation whose canonical byte form already exists inside the `Block` value.

Conceptually, this has three important consequences:

- storage can persist canonical bytes without inventing a second serialization format,
- control-plane chain-configuration storage can round-trip through canonical bytes explicitly,
- and integrity checking in the RP2040 backend is meaningful only because there is one canonical byte-level representation to hash and later re-parse.

## Why the Public API Is Intentionally Small

The current `StorageTrait` exposes only six synchronous operations:

- `init(...)`
- `save_block(...)`
- `read_block(...)`
- `capacity()`
- `set_chain_configuration(...)`
- `load_control_data()`

Conceptually, this means MoonBlokz storage currently treats the following as **in scope**:

- initialization of persistent node-local state,
- deterministic placement of blocks by external index,
- deterministic retrieval of blocks by external index,
- one-time storage of chain configuration inside control data,
- and explicit error signaling.

It treats the following as **out of scope** for the storage crate itself:

- chain pruning policy,
- retention strategy,
- block selection policy,
- chain reconstruction policy,
- and any general query/index subsystem beyond direct indexed slot access.

## Why Control Plane and Block Plane Remain Separate

The current implementation keeps the conceptual split between two persistent data classes.

### Control plane

The control plane stores the node-local recovery-critical state:

- private key,
- own node ID,
- initialization parameters,
- optional chain-configuration block,
- plus compatibility and checksum metadata.

### Block plane

The block plane stores blockchain blocks at deterministic external indexes.

This separation is not merely layout convenience. It reflects a business distinction:

- control-plane data defines whether the node can recover its own identity and startup context,
- while block-plane data is the indexed blockchain content that chain logic can scan, reconstruct from, or treat as absent/invalid.

## What the Current Control Plane Represents

The current code makes the control plane more explicit than the article did.

It is now a versioned, compatibility-checked record with:

- `CONTROL_PLANE_VERSION`,
- persisted `PRIVATE_KEY_SIZE`,
- persisted `INIT_PARAMS_SIZE`,
- persisted `MAX_BLOCK_SIZE`,
- the actual private key and node ID,
- fixed-size init parameters,
- optional chain configuration block payload space,
- and a trailing `CRC32` checksum.

Conceptually, the control plane therefore does three jobs at once:

1. stores node-local identity and startup parameters,
2. stores one optional chain configuration block,
3. acts as a compatibility boundary between persisted data and the current binary constants.

This compatibility role is an important sharpening of the article-era design.

## Why the Current Crate Uses Replica-Based Control Recovery

Both backends use `CONTROL_PLANE_COUNT = 3` replicated control-plane records.

Conceptually, this means the current crate treats control-plane durability as a first-class concern. But the current code goes further than simple fallback:

- it scans replicas in deterministic order,
- accepts the first valid record,
- and performs best-effort repair of invalid replicas by rewriting them from the first valid record.

So the control-plane model is not only “keep backups.” It is **read-valid-first, then repair**.

## Why Compatibility Checking Is Part of Recovery

The current code explicitly distinguishes:

- `ControlPlaneUninitialized`
- `ControlPlaneCorrupted`
- `ControlPlaneIncompatible`

This is conceptually important.

The control plane is not considered valid merely because its checksum matches. It must also agree with the current binary’s expected constants and schema version.

So recovery is not just corruption detection. It is also **binary-schema compatibility validation**.

## Why the Current Block Model Is Index-Oriented Rather Than Chain-Oriented

The current public contract stores and reads blocks by `storage_index`.

That means the storage layer currently models block persistence as:

- external chain logic chooses an index,
- storage deterministically maps that index to backend-specific placement,
- storage persists or retrieves the block from that slot.

Conceptually, this is narrower than the article’s discussion of bounded-chain maintenance over time. The crate does **not** currently implement `snake_chain` turnover policy, tail movement, or fork-retention rules. It assumes those choices happen outside the storage layer.

This is one of the most important current divergences from the article framing and must remain explicit.

## Why the `version == 0` Invariant Matters to Storage

`moonblokz-chain-types` explicitly reserves `version == 0` for storage empty-slot markers and requires valid MoonBlokz blocks to use a non-zero version.

This is not just a block-parser detail. It is a storage contract dependency.

Conceptually, it means:

- the memory backend can safely use `slot[0] == 0` as an empty-slot marker,
- the chain-types parser can reject zero-version content as malformed rather than ambiguous valid content,
- and storage emptiness detection and canonical block validity remain aligned.

This is one of the clearest examples of the storage crate depending directly on a chain-types invariant.

## Why the Two Backends Represent Two Different Conceptual Roles

The current crate ships two backends, but they do not aim for identical internal behavior.

### `backend-memory`

The in-memory backend is mainly a contract-testing and off-target integration backend.

Conceptually, it provides:

- a deterministic byte-array-backed storage model,
- control-plane replication and CRC validation,
- indexed block slots with simple overwrite semantics,
- empty-slot detection that depends on the canonical non-zero block-version invariant,
- but **not** the RP2040 backend’s full block-integrity model.

### `backend-rp2040`

The RP2040 backend is the embedded-target backend that models actual page-based flash behavior.

Conceptually, it provides:

- page-aligned storage geometry,
- erase/write/read paths tied to 4 KB flash pages,
- per-slot hash metadata for block integrity,
- and host-side mock-flash testing that preserves RP2040 geometry semantics.

So backend portability in the current crate means **shared public semantics with backend-specific storage mechanics** rather than identical physical behavior.

## Integrity as a Backend-Specific but Contract-Relevant Concern

The current code preserves the article’s broad idea that stored data must not be trusted blindly, but it narrows how that is implemented.

### Control plane

Both backends use CRC32-based validation and compatibility checks.

### Block plane

Only the RP2040 backend currently implements explicit stored-hash verification for blocks.

The in-memory backend instead uses parseability and slot-state semantics, returning backend-I/O-style errors when stored bytes cannot be parsed back into a block.

This is a significant conceptual clarification relative to the article: the current crate does **not** present one identical block-integrity strategy across all backends.

## Why the RP2040 Hash Contract Is Narrower Than “Hash the Block”

`moonblokz-chain-types` documents that callers typically hash `block.serialized_bytes()`. The RP2040 storage backend does something more specific for storage integrity.

It hashes the fixed `MAX_BLOCK_SIZE` slot-sized block area, not only the logical serialized length.

Conceptually, that means the RP2040 stored hash is not a general blockchain identity hash. It is a **storage-slot integrity hash** over the persisted fixed-size block region.

That distinction matters because it explains why the RP2040 backend can detect corruption of the stored slot representation while still relying on canonical block parsing afterward.

## Why `init()` Is a Destructive Reset in the Current Model

The current code makes `init(...)` strongly destructive.

- Memory backend: fills the entire byte array with zeroes before writing the initial control-plane record.
- RP2040 backend: erases every page from the configured storage start to the end of the reserved flash region before writing control-plane replicas.

Conceptually, `init()` is therefore not an incremental setup call. It is a **reset-and-reseed operation** that establishes a clean storage state.

## Why Chain Configuration Is Treated as Set-Once Control Data

The current code persists the chain configuration through `set_chain_configuration(...)` and returns `ChainConfigurationAlreadySet` on later attempts.

Conceptually, this means chain configuration is currently treated as:

- control-plane state rather than ordinary block-plane state,
- write-once unless full reinitialization occurs,
- and part of the node’s startup context rather than a frequently changing persistent record.

This is consistent with the article’s direction, but the code makes the lifecycle rule much more explicit.

## Explicit Differences from the Earlier Part VIII Article

The current codebase sharpens or changes the earlier storage narrative in several important ways.

### 1. The crate is smaller in scope than the article-level storage story

The article discussed storage in terms of bounded blockchain persistence under `snake_chain`.

The current crate does **not** implement:

- automatic chain-window maintenance,
- replay scheduling,
- block expiration policy,
- or fork-headroom policy.

It implements a deterministic indexed storage contract that chain logic can use.

### 2. The current RP2040 layout is page-based per slot group

The article described a 4 KB storage unit that could hold multiple blocks plus hashes.

The current RP2040 backend implements:

- 4 KB flash pages,
- multiple slots per page,
- each slot sized as `MAX_BLOCK_SIZE + HASH_SIZE`,
- and page rewrites when any slot within the page is updated.

That is compatible with the article’s flash-awareness, but the concrete formalization is now code-defined rather than article-defined.

### 3. The memory backend intentionally does not mirror all RP2040 integrity behavior

The current docs and PRD explicitly allow the in-memory backend to serve happy-path and off-target testing goals without claiming full RP2040 integrity parity.

### 4. Control-plane compatibility checking is now part of the design

The article discussed corruption detection. The current code adds schema/version/size compatibility checks as a first-class part of control-plane validity.

### 5. Canonical block validity is now sharper than the article implied

The chain-types crate makes current block validity depend on a narrower and more explicit canonical contract than the article alone described.

So storage parseability and block acceptance now depend directly on the upstream chain-types validation rules rather than on a looser article-level description. The exact size and validity constraints belong primarily in [moonblokz-storage-algorythm.md](./moonblokz-storage-algorythm.md).

## Explicit Limits of the Current Source Material

Even with the current codebases, some broader storage questions remain intentionally outside the crate.

The current repositories do **not** define:

- how chain logic chooses `storage_index` values over time,
- how `snake_chain` retention and block eviction are coordinated with storage placement,
- how live-chain reconstruction walks these indexes in a full node,
- how storage write timing is coordinated with radio work at whole-system level,
- or how future non-RP2040 backends should handle block-integrity metadata beyond preserving public API semantics.

These limits should remain explicit rather than being silently filled in by documentation guesswork.
