# MoonBlokz Storage Algorithm Model

## Purpose of This Document

This document provides a formal, algorithm-oriented description of the current MoonBlokz storage behavior as implemented in `moonblokz-storage`, with the block and hash contract grounded explicitly in `moonblokz-chain-types`.

Its purpose is to capture:

- the current public storage API contract,
- the formal control-plane record model,
- the canonical block and hash constraints that storage relies on,
- the indexed block-storage rules,
- the backend-specific geometry and integrity behavior,
- the error outcomes that chain logic can rely on,
- and the exact limits of what the current crate still does not define.

This file is the **primary knowledge-base document for the current formal storage model**.

- Use [moonblokz-storage-prd.md](./moonblokz-storage-prd.md) for the authoritative FR1–FR53 / NFR1–NFR20 catalog that defines the required storage capabilities.
- Use [moonblokz-storage-architecture.md](./moonblokz-storage-architecture.md) for the authoritative crate-split, backend feature model, naming and structure patterns, and data-structure contracts that this formal model realizes.
- Use [moonblokz-storage-concept.md](./moonblokz-storage-concept.md) for the higher-level explanation of why the storage crate is shaped this way and how it differs from the earlier article framing.
- Use [moonblokz-storage-implementation.md](./moonblokz-storage-implementation.md) for engineering consequences of the current crate structure, feature model, mock-flash strategy, and backend implementation details.

## Source Basis

This document is grounded primarily in the current repositories, especially:

- `moonblokz-storage/src/lib.rs`
- `moonblokz-storage/src/error.rs`
- `moonblokz-storage/src/types.rs`
- `moonblokz-storage/src/backend_memory.rs`
- `moonblokz-storage/src/backend_rp2040.rs`
- `moonblokz-storage/src/conformance.rs`
- `moonblokz-storage/README.md`
- `moonblokz-storage/Cargo.toml`
- `moonblokz-chain-types/src/lib.rs`
- `moonblokz-chain-types/src/block.rs`
- `moonblokz-chain-types/src/hash.rs`
- `moonblokz-chain-types/src/error.rs`
- `moonblokz-chain-types/docs/block-data-structure.md`
- `moonblokz-chain-types/README.md`

Where the earlier Part VIII article and the current code differ, this file treats the current code as authoritative.

## Scope and Limits

This file captures the algorithmic structure explicitly visible in the current crate, including:

- compile-time backend selection rules,
- control-plane serialization and validation behavior,
- canonical block-size and hash constraints from `moonblokz-chain-types`,
- deterministic capacity and index-boundary behavior,
- the current memory-backend slot model,
- the current RP2040 page/slot/hash model,
- startup-time control-plane repair behavior,
- and backend conformance expectations visible in tests.

It does **not** attempt to define:

- chain-policy index assignment,
- `snake_chain` retention policy,
- flash wear-lifetime calculations beyond what is implicit in the current page-erase design,
- a complete runtime scan algorithm for node boot reconstruction,
- or any future backend behavior not implemented in the current repositories.

## Core Terminology Used in This Document

To keep the three storage knowledge-base files aligned, this document uses the following terms consistently:

- **StorageTrait** — the synchronous public storage contract.
- **storage_index** — the canonical external index used for block placement and retrieval.
- **Control plane** — the replicated node-local metadata record.
- **Control-plane replica** — one persisted copy of the control-plane record.
- **Block plane** — the indexed area used for storing blockchain blocks.
- **Slot** — one indexed block-storage location.
- **Canonical block bytes** — the `serialized_bytes()` view exposed by `Block`.
- **Canonical block hash contract** — the `calculate_hash(&[u8]) -> [u8; HASH_SIZE]` contract from `moonblokz-chain-types`.
- **Page** — one RP2040 erase/write unit of 4096 bytes.
- **Slot mapping** — the deterministic mapping from `storage_index` to page-local coordinates on RP2040.

## Algorithmic Problem Statement

The current storage crate must solve the following combined problem:

1. initialize a deterministic persistent storage region,
2. preserve a replicated control-plane record with corruption and compatibility checks,
3. persist blocks at externally chosen integer indexes,
4. retrieve blocks from those indexes with explicit typed outcomes,
5. preserve one stable public contract across multiple backend implementations,
6. remain compatible with embedded `no_std` synchronous integration,
7. and do all block persistence against the canonical block and hash rules defined in `moonblokz-chain-types`.

## Section A — Canonical Chain-Types Constraints Used by Storage

## A1. Canonical block constants

The current storage implementation relies on these chain-types constants:

- `MAX_BLOCK_SIZE = 2016`
- `HEADER_SIZE = 122`
- `MAX_PAYLOAD_SIZE = MAX_BLOCK_SIZE - HEADER_SIZE = 1894`
- `HASH_SIZE = 32`

These are not storage-local conventions. They are canonical upstream contracts.

## A2. Canonical block validity constraints

`Block::from_bytes(bytes)` currently accepts input only if:

1. `bytes.len() >= HEADER_SIZE`,
2. `bytes.len() <= MAX_BLOCK_SIZE`,
3. the parsed block structure is valid,
4. and specifically the version byte is non-zero.

The current `validate_structure()` implementation enforces:

- block length must be at least `HEADER_SIZE`,
- block version must be non-zero.

## A3. Empty-slot invariant inherited from chain-types

The chain-types crate explicitly reserves `version == 0` for storage empty-slot markers.

Therefore storage may safely rely on the invariant:

- any valid MoonBlokz block has first byte `!= 0`.

This invariant directly underpins empty-slot detection in the memory backend.

## A4. Canonical hash contract

The chain-types crate defines:

- `calculate_hash(input: &[u8]) -> [u8; HASH_SIZE]`
- using SHA-256,
- with fixed `HASH_SIZE = 32`.

Storage must treat this as the canonical hashing contract rather than defining a storage-local hash algorithm.

## Section B — Compile-Time Structural Rules

## B1. Backend exclusivity rule

Exactly one backend feature must be enabled at compile time:

- `backend-memory`
- `backend-rp2040`

Compilation fails if:

- no backend feature is enabled,
- or more than one backend feature is enabled.

## B2. Canonical backend alias rule

`MoonblokzStorage<const STORAGE_SIZE: usize>` is a type alias to exactly one backend, chosen by the active backend feature:

- memory backend when `backend-memory` is selected,
- RP2040 backend when `backend-rp2040` is selected.

## B3. Public constant set

The current public constants include:

- `INIT_PARAMS_SIZE = 100`
- `CONTROL_PLANE_COUNT = 3`
- `CONTROL_PLANE_VERSION = 1`

These constants are part of the storage contract because the control-plane record and API behavior depend on them.

## Section C — Public API Contract

## C1. `StorageTrait` operations

The current public contract exposes exactly these methods:

1. `init(private_key, own_node_id, init_params)`
2. `save_block(storage_index, block)`
3. `read_block(storage_index)`
4. `capacity()`
5. `set_chain_configuration(block)`
6. `load_control_data()`

## C2. Public return model

`load_control_data()` returns `ControlPlaneData` with:

- `private_key: [u8; PRIVATE_KEY_SIZE]`
- `own_node_id: u32`
- `init_params: [u8; INIT_PARAMS_SIZE]`
- `chain_configuration: Option<Block>`

## C3. Public error categories

The current public error model is:

- `InvalidIndex`
- `BlockAbsent`
- `IntegrityFailure`
- `ControlPlaneUninitialized`
- `ChainConfigurationAlreadySet`
- `ControlPlaneCorrupted`
- `ControlPlaneIncompatible`
- `InvalidConfiguration`
- `BackendIo { code }`

These are the main typed outcomes chain logic must reason about.

## Section D — Control-Plane Formal Model

## D1. Control-plane record fields

Both current backends serialize a logically equivalent control-plane record containing:

1. control-plane version (`u8`)
2. persisted private-key-size field (`u8`)
3. private key bytes (`[u8; PRIVATE_KEY_SIZE]`)
4. own node ID (`u32`, little-endian)
5. persisted init-params-size field (`u8`)
6. init parameter bytes (`[u8; INIT_PARAMS_SIZE]`)
7. persisted `MAX_BLOCK_SIZE` field (`u16`, little-endian)
8. reserved chain-configuration block space (`[u8; MAX_BLOCK_SIZE]`)
9. CRC32 checksum (`u32`, little-endian)

## D2. Control-plane validity conditions

A control-plane replica is accepted only if all of the following hold:

1. the replica is not interpreted as uninitialized,
2. the stored CRC32 matches the recomputed CRC32,
3. the stored control-plane version equals `CONTROL_PLANE_VERSION`,
4. the stored private-key-size field matches the current `PRIVATE_KEY_SIZE`,
5. the stored init-params-size field matches the current `INIT_PARAMS_SIZE`,
6. the stored `MAX_BLOCK_SIZE` matches the current runtime `MAX_BLOCK_SIZE`,
7. if a chain-configuration block is present, it parses successfully as `Block`.

## D3. Control-plane uninitialized conditions

### Memory backend

A replica is treated as uninitialized if all bytes in the serialized record are zero.

### RP2040 backend

A replica is treated as uninitialized if the serialized record bytes are either:

- all zero,
- or all `0xFF`.

## D4. Control-plane load and repair algorithm

The current load algorithm for both backends is conceptually:

1. iterate replicas in deterministic replica-index order,
2. attempt to deserialize each replica,
3. remember the first valid record,
4. track invalid replica indexes,
5. if no valid record exists:
   - return `ControlPlaneUninitialized` if all replicas are uninitialized,
   - otherwise prefer `ControlPlaneIncompatible` if any incompatible replica was seen,
   - otherwise return `ControlPlaneCorrupted`,
6. if a valid record exists:
   - return that record,
   - and rewrite invalid replicas from the first valid record as best-effort repair.

## D5. Control-plane write behavior

### `init(...)`

Both backends write a control-plane record with:

- private key set,
- own node ID set,
- init params set,
- `chain_configuration = None`.

### `set_chain_configuration(block)`

1. load current valid control-plane record,
2. if `chain_configuration` is already `Some`, return `ChainConfigurationAlreadySet`,
3. otherwise reconstruct the input through canonical block bytes,
4. store it as `Some(block)`,
5. write the updated record to all replicas.

## Section E — Memory Backend Formal Model

## E1. Capacity rule

For `MemoryBackend<const STORAGE_SIZE: usize>`:

- control-plane reserved bytes = `CONTROL_PLANE_COUNT * CONTROL_PLANE_ENTRY_SIZE`
- effective block-slot count =
  `floor((STORAGE_SIZE - control_plane_reserved_bytes) / MAX_BLOCK_SIZE)`
  when positive, otherwise `0`
- remainder bytes are unused.

With the current chain-types constant `MAX_BLOCK_SIZE = 2016`, this means slot geometry is defined directly by the canonical block-size contract.

## E2. Empty-slot rule

A memory-backend slot is treated as empty if `slot[0] == 0`.

This is valid only because the chain-types contract guarantees that a valid block must have non-zero version.

## E3. Save algorithm

For `save_block(storage_index, block)`:

1. validate `storage_index < capacity`, else return `InvalidIndex`,
2. compute slot byte range,
3. obtain canonical serialized block bytes through `block.serialized_bytes()`,
4. if serialized length exceeds `MAX_BLOCK_SIZE`, return `BackendIo { code: 1 }`,
5. zero the whole slot,
6. copy block bytes to the beginning of the slot.

## E4. Read algorithm

For `read_block(storage_index)`:

1. validate `storage_index < capacity`, else return `InvalidIndex`,
2. compute slot byte range,
3. if `slot[0] == 0`, return `BlockAbsent`,
4. attempt `Block::from_bytes(slot)`,
5. if parsing fails, return `BackendIo { code: 2 }`,
6. otherwise return the block.

## E5. Practical meaning of memory-backend parseability

Because the memory backend passes the full fixed slot buffer into `Block::from_bytes`, successful parseability depends on current chain-types rules:

- full slot length is acceptable because `MAX_BLOCK_SIZE = slot size`,
- version byte must be non-zero,
- the slot content must still be structurally acceptable to the canonical block parser.

This means the memory backend currently treats a stored block as valid if it remains parseable as a canonical block in the fixed slot representation.

## E6. Integrity behavior boundary

The memory backend does **not** maintain a stored block hash alongside each slot.

Its practical block-validity model is therefore:

- empty-slot detection,
- plus parseability of stored slot bytes,
- not RP2040-style explicit hash revalidation.

## Section F — RP2040 Backend Formal Model

## F1. Core geometry constants

The RP2040 backend defines:

- `FLASH_PAGE_SIZE = 4096`
- `RP2040_DEFAULT_FLASH_SIZE = 2 * 1024 * 1024`
- `SLOT_HASH_OFFSET = MAX_BLOCK_SIZE = 2016`
- `SLOT_SIZE_BYTES = MAX_BLOCK_SIZE + HASH_SIZE = 2048`
- `CONTROL_PLANE_RESERVED_BYTES = CONTROL_PLANE_COUNT * FLASH_PAGE_SIZE = 12288`
- `BLOCKS_PER_PAGE = floor(FLASH_PAGE_SIZE / SLOT_SIZE_BYTES) = 2`

So with the current chain-types contracts, each RP2040 flash page stores exactly two block slots.

## F2. Compile-time geometry guards

Compilation fails if either of these would be false:

- `BLOCKS_PER_PAGE >= 1`
- `CONTROL_PLANE_ENTRY_SIZE <= FLASH_PAGE_SIZE`

## F3. Valid backend-construction condition

The RP2040 backend requires `data_storage_start_address` to be page-aligned.

If `data_storage_start_address % FLASH_PAGE_SIZE != 0`, construction or initialization returns `InvalidConfiguration`.

## F4. Capacity rule

The current RP2040 capacity algorithm is:

1. `available_bytes = RP2040_FLASH_SIZE - data_storage_start_address` (saturating)
2. `block_storage_bytes = available_bytes - CONTROL_PLANE_RESERVED_BYTES` (saturating)
3. `usable_pages = floor(block_storage_bytes / FLASH_PAGE_SIZE)`
4. `capacity = usable_pages * BLOCKS_PER_PAGE`

## F5. Slot-mapping rule

For any valid `storage_index`:

- `page_index = floor(storage_index / BLOCKS_PER_PAGE)`
- `slot_index = storage_index % BLOCKS_PER_PAGE`
- `byte_offset_in_page = slot_index * SLOT_SIZE_BYTES`

The page flash address is:

`data_storage_start_address + CONTROL_PLANE_RESERVED_BYTES + page_index * FLASH_PAGE_SIZE`

## F6. RP2040 save algorithm

For `save_block(storage_index, block)`:

1. validate `storage_index < capacity`, else return `InvalidIndex`,
2. map index to page and slot coordinates,
3. read the whole containing page into the page buffer,
4. zero the target slot region,
5. copy canonical block bytes into the block portion of the slot,
6. compute `calculate_hash()` over the fixed `MAX_BLOCK_SIZE` block region,
7. store that hash in the slot hash area,
8. erase the whole flash page,
9. write back the full page buffer.

## F7. RP2040 read algorithm

For `read_block(storage_index)`:

1. validate `storage_index < capacity`, else return `InvalidIndex`,
2. map index to page and slot coordinates,
3. read the containing page into the page buffer,
4. extract the target slot bytes,
5. if all slot bytes are `0xFF`, return `BlockAbsent`,
6. read stored hash from the slot hash area,
7. recompute hash over the fixed `MAX_BLOCK_SIZE` block area,
8. if hashes differ, return `IntegrityFailure`,
9. parse the fixed block area as `Block`,
10. if parsing fails, return `IntegrityFailure`,
11. otherwise return the block.

## F8. Practical meaning of RP2040 hash verification

The RP2040 backend hashes the fixed 2016-byte block area, not merely `block.serialized_bytes()` length.

So the stored RP2040 hash is currently a hash of the persisted fixed-size canonical block region as stored in the slot.

It therefore verifies storage-slot integrity, not merely the logical serialized prefix in isolation.

## F9. Practical meaning of RP2040 overwrite behavior

Because one slot write rewrites the containing 4096-byte page, the current RP2040 behavior is page-granular in flash even though the public API is slot-granular.

## Section G — `init()` Semantics

## G1. Memory backend `init()`

The memory backend:

1. validates control-plane capacity,
2. fills the entire storage byte array with zero,
3. serializes a fresh control-plane record with no chain configuration,
4. writes that same record to all control-plane replicas.

## G2. RP2040 backend `init()`

The RP2040 backend:

1. validates page alignment of the configured start address,
2. computes the flash pages from `data_storage_start_address` to end of configured storage,
3. erases every one of those pages,
4. builds a fresh control-plane record with no chain configuration,
5. writes that record to all control-plane replicas.

## Section H — Backend I/O Code Map

The current public `BackendIo { code }` map includes:

### Memory backend runtime codes

- `1` — oversized block input during save path
- `2` — parse failure for stored slot bytes on read path

### RP2040 runtime codes

- `210` — flash page read failed
- `211` — flash page erase failed
- `212` — flash page write failed
- `213` — unexpected RP2040 save-path backend branch or save-path encoding failure branch
- `220` — flash page read failed during retrieve path

### RP2040 mock-flash test codes

- `230` — mock flash read out of bounds
- `231` — mock flash erase range invalid or out of bounds
- `232` — mock flash write out of bounds

## Section I — Conformance Behavior Visible in Tests

The shared conformance tests assert at least the following cross-backend public semantics:

- successful save/read round trip,
- `BlockAbsent` for empty valid slots,
- `InvalidIndex` for reads and writes at or above capacity,
- capacity boundary consistency,
- mixed startup-scan outcomes where some indexes are populated and others absent.

This means backend parity in the current repository should be understood primarily at the level of **public typed storage semantics**, not necessarily identical internal integrity mechanisms.

## Section J — Explicit Current Boundaries

The current crates do **not** formally define:

- how `storage_index` values are assigned by chain logic,
- how a full boot scan should stop or continue across large index ranges,
- how retained-chain policy relates to slot reuse,
- whether `IntegrityFailure` should trigger automatic recovery from network at higher layers,
- or how future backends must replicate the RP2040 slot-hash layout specifically.

These questions remain outside the current formal storage contract.
