# MoonBlokz Storage Product Requirements Document

## Authoritative Source Notice

This document is the **authoritative source** for the `moonblokz-storage` library's functional requirements (FR1–FR53) and Non-Functional Requirements (NFR1–NFR20) as referenced throughout the rest of the MoonBlokz knowledge base.

When other storage knowledge-base files (concept, algorithm, implementation) cite an FR or NFR by number, the canonical wording of that requirement lives in this file. If any other knowledge-base text appears to diverge from the content here, the divergence must be flagged explicitly per the rule in [`AGENTS.md`](./AGENTS.md), and resolution must be left to the user.

## Provenance

- Originating workflow: BMAD planning, eleven steps completed (`step-01-init` through `step-11-polish`) plus an edit pass (`step-e-01-discovery` through `step-e-03-edit`).
- Source artifact: `moonblokz-bmad-history/storage/planning-artifacts/prd.md`.
- Imported into the `moonblokz-info` knowledge base on 2026-06-18.
- Frontmatter from the originating workflow output was stripped during import; the body below is otherwise byte-identical to the source artifact at import time.
- The 2026-02-28 edit pass added control-plane management requirements (FR6–FR20), storage layout and capacity updates, explicit control-plane lifecycle mapping in the User Journeys, and a normalized capability-oriented phrasing style across the FR catalog.

## Related Knowledge-Base Documents

- [`moonblokz-storage-architecture.md`](./moonblokz-storage-architecture.md) — authoritative architecture decisions, crate-split layout, backend feature model, structure/pattern rules, and data structure contracts that realize these requirements.
- [`moonblokz-storage-concept.md`](./moonblokz-storage-concept.md) — conceptual storage operating model grounded in the current code.
- [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) — formal algorithm description grounded in the current code.
- [`moonblokz-storage-implementation.md`](./moonblokz-storage-implementation.md) — engineering consequences derived from the current code.

---

# Product Requirements Document - moonblokz

**Author:** Sasa
**Date:** 2026-02-23 23:00:44 CET

## Executive Summary

MoonBlokz requires a dedicated onboard blockchain storage layer to operate on constrained microcontrollers. The target deliverable is a Rust `no_std` developer library that provides deterministic, index-based persistence and retrieval of blockchain blocks, with RP2040 as the MVP backend and support for additional backends as a first-class design requirement.

The core problem is operational viability under strict memory constraints. RP2040-class hardware cannot retain enough blockchain data in RAM for sustained network participation, so durable flash-backed storage is a blocking dependency for boot reconstruction, block ingestion, and query serving.

The product is designed for MoonBlokz chain/runtime integration logic rather than end-user interaction. It must provide synchronous, bounded-memory behavior, explicit storage index addressing (`storage_index -> flash address`), integrity-safe retrieval through hash revalidation before returning block data, and control-plane lifecycle management for node boot metadata.

### What Makes This Special

This is blockchain-specific embedded storage, not generic embedded persistence. It is intentionally aligned to MoonBlokz runtime semantics: deterministic indexed access, verified reads, control-plane metadata persistence, and strict separation of concerns where storage owns persistence/integrity mechanics and chain logic owns chain-policy decisions (for example pruning and retention strategy).

The core differentiator is that no existing generic embedded storage implementation directly satisfies these blockchain data semantics under MoonBlokz constraints. The library therefore provides a domain-fit foundation that enables reliable operation now (RP2040) while preserving backend portability for future hardware targets.

Value proposition: MoonBlokz Storage gives constrained microcontrollers deterministic, integrity-checked blockchain persistence with a portable backend abstraction.

## Project Classification

- **Project Type:** developer_tool
- **Domain:** general
- **Complexity:** medium
- **Project Context:** greenfield

## Success Criteria

### User Success

- Chain/runtime maintainers can run a complete boot-to-participation cycle on RP2040 using persisted blockchain data.
- Startup successfully reconstructs required in-memory chain structures from flash-backed blocks under device RAM constraints.
- Runtime block retrieval for radio/query flows is reliable and integrity-safe.

### Business Success

- N/A for revenue/growth metrics (open-source engineering project).
- Project-level success is measured by technical adoption readiness:
  - Storage library is integration-ready for MoonBlokz chain/runtime.
  - RP2040 target support is complete for MVP.
  - API contracts are stable enough to support further backend implementations.

### Technical Success

- Read path always enforces hash verification before returning block data.
- Storage addressing is deterministic through `storage_index -> flash address`.
- Partial/invalid write effects are detected on subsequent read/startup and returned as explicit errors to chain logic.
- Storage remains policy-agnostic (no pruning/retention ownership in storage layer).
- Implementation respects `no_std`, synchronous API behavior, and bounded-memory design expectations.

### Measurable Outcomes

- **Initialization correctness:** Startup load process consistently reconstructs chain-required structures from persisted blocks in validation runs.
- **Integrity enforcement:** Retrieval behavior rejects hash mismatches with explicit errors in all tested mismatch scenarios.
- **Capacity conformance:** Supported block-slot envelope follows:
  `floor((FLASH_SIZE - APP_BINARY_SIZE - (CONTROL_PLANE_PAGE_SIZE * CONTROL_PLANE_COUNT)) / BLOCK_SIZE)`.
- **Boundary conformance:** Public storage API remains limited to indexed persistence/retrieval and integrity/error signaling responsibilities.
- **Resource discipline:** Storage-owned RAM usage is minimal and limited to required functional state.

## Product Scope

### MVP - Minimum Viable Product

- Complete RP2040 implementation of MoonBlokz storage functionality.
- `no_std`, synchronous API for chain/runtime integration.
- Control-plane initialization API: `init(private_key, own_node_id, init_params)` with destructive reset semantics.
- Indexed block persistence/retrieval with deterministic address derivation.
- Read-time hash verification and explicit error signaling on invalid data.
- Control-plane load API: `load_control_data()` returning private key, own node id, init parameters, and optional chain configuration block.
- Chain configuration control-plane API: `set_chain_configuration(Block)` as set-once operation unless full re-init is executed.
- Replicated control-plane persistence using `CONTROL_PLANE_COUNT` instances, with CRC32 validation and recovery behavior.
- Storage layout contract where control-plane replicas are stored first at `data_storage_start_address`, and block storage capacity/addressing is computed from the remaining region.
- Startup-compatible block loading behavior to support chain initialization.

### Growth Features (Post-MVP)

- Additional backend implementations beyond RP2040.
- Broader platform abstraction and portability tooling.
- Expanded validation harnesses across multiple hardware targets.

### Future Direction (Reference)

- Multi-backend storage library with consistent deterministic semantics across targets.
- Stable long-term storage contract for MoonBlokz runtime evolution.
- Continued strict separation between storage mechanics and chain-policy decisions.

## User Journeys

### Journey 1: Primary User - Success Path (Chain Runtime)

At first boot, the Chain Runtime starts in a constrained RP2040 environment where RAM cannot hold full blockchain history. Before normal operation, runtime executes `init(private_key, own_node_id, init_params)` to initialize control-plane state, clear control-plane replicas and block pages, and establish deterministic storage layout. On subsequent boots, runtime calls `load_control_data()` to load control-plane metadata (private key, node id, init params, optional chain configuration) before block reconstruction begins.

After control-plane load succeeds, runtime iterates indexed blocks and rebuilds in-memory chain structures needed for participation. Each retrieved block is integrity-checked (hash revalidation) before it is accepted into runtime state.

As new blocks arrive through radio flow, runtime validates chain-level semantics, then persists accepted blocks via deterministic `storage_index` addressing. During initialization flow, runtime can set chain configuration once through `set_chain_configuration(Block)`; later attempts return an explicit already-set error unless full re-initialization is performed. During normal operation, query paths request blocks by index and receive verified data for response handling.

The value climax is the first full boot-to-participation cycle that succeeds from persisted flash data, proving the node can operate beyond RAM-only limits. The resolution is stable day-to-day runtime behavior with deterministic persistence/retrieval guarantees.

### Journey 2: Primary User - Edge Case (Chain Runtime Recovery)

The runtime starts after an interrupted write event or flash anomaly. During control-plane load, storage reads replicated control-plane entries in order, validates CRC32, and uses the first valid replica. If one or more replicas fail checksum validation, storage continues with valid data and attempts best-effort repair of failed replicas during the same lifecycle.

During block load or indexed retrieval, hash verification may fail for one or more entries. Storage does not silently return suspect data; it surfaces explicit errors.

Runtime uses those errors to avoid corrupt-state propagation and applies chain-level recovery decisions outside storage policy scope. Even with invalid sectors/entries, deterministic indexing and explicit failure signals allow controlled startup behavior rather than undefined runtime corruption.

The value moment is predictable failure handling: corrupted data is detected and surfaced consistently, enabling chain logic to recover safely.

### Journey 3: Secondary User - MoonBlokz Chain Developer

A Chain Developer is implementing or evolving chain logic and needs a storage contract that is simple, deterministic, and backend-agnostic. They integrate synchronous `no_std` APIs, rely on `storage_index` mapping semantics, and keep chain policy decisions in chain code.

During development, they test startup reconstruction, ingest persistence, and query retrieval paths against RP2040 constraints. They use storage error semantics to define clear chain behavior on integrity mismatch or unavailable data.

The value climax is when chain logic becomes stable because storage responsibilities are sharply bounded: persistence/integrity in storage, policy and reconciliation in chain logic. The resolution is faster iteration and fewer cross-layer ambiguities.

### Journey 4: Support/Troubleshooting - MoonBlokz Node Operator

A Node Operator manages deployed nodes and monitors whether nodes can reboot and rejoin correctly. After a power interruption, a node reboots and runtime reports storage integrity errors for specific indexed blocks.

Operator-facing tooling/logs (outside this library) shows deterministic error signals from storage, allowing operator workflows to distinguish corruption from radio/network issues. The operator can trigger standard operational procedures with confidence that storage failures are explicit and non-silent.

The value moment is operational clarity: failures are diagnosable, and nodes do not appear healthy while serving corrupted data. The resolution is improved field reliability and safer node lifecycle management.

### Journey 5: API/Integration - Storage Backend Implementer

A Backend Implementer wants to bring MoonBlokz storage semantics to new hardware after RP2040 MVP. They implement the same storage API contract and preserve deterministic index mapping, read-time hash verification, and explicit error behavior.

They validate conformance against capacity and integrity expectations, ensuring behavior parity with RP2040 semantics rather than hardware-specific ad hoc behavior. Integration tests confirm that chain/runtime code can run unchanged while backend implementation differs.

The value climax is successful backend substitution without changing chain-level storage expectations. The resolution is a portable multi-backend storage ecosystem with consistent semantics.

### Journey Requirements Summary

The journeys reveal required capability areas:

- Deterministic indexed persistence and retrieval API (`storage_index` mapping).
- Explicit control-plane lifecycle: first-boot `init(...)`, subsequent-boot `load_control_data()`, and immutable-until-reinit control metadata semantics.
- Replicated control-plane durability: deterministic multi-copy writes, CRC32-based read fallback, and best-effort replica repair.
- Set-once chain configuration lifecycle via `set_chain_configuration(Block)` with explicit already-set error semantics.
- Startup block iteration/loading support sufficient for chain reconstruction.
- Mandatory read-time integrity verification before data return.
- Explicit, actionable error signaling on invalid/partial/corrupt data.
- Strict separation of concerns between storage mechanics and chain policy.
- Backend portability via stable storage contracts and conformance behavior.
- Operational observability hooks (through upstream components) that expose storage failure modes clearly.

## Domain Requirements

### Compliance & Regulatory

- No mandatory external regulatory regime is currently targeted for MVP (no HIPAA/PCI/FedRAMP scope).
- Open-source engineering traceability still applies:
  - deterministic behavior documentation
  - explicit error semantics
  - validation evidence for integrity guarantees

### Technical Constraints

- Embedded constraints: RP2040 flash/RAM limits and no unbounded memory structures.
- Execution model constraints: MoonBlokz uses the Embassy framework; storage integration must remain compatible with Embassy-based runtime architecture.
- Determinism constraints: synchronous `no_std` API, bounded operations, predictable failure behavior.
- Integrity constraints: hash-verified reads only; never return data on hash mismatch.
- Control-plane schema constraints: persisted control-plane layout is
  `version:u8, private_key_size:u8, private_key:[u8;PRIVATE_KEY_SIZE], own_node_id:u32, init_params_size:u8, init_params:[u8;INIT_PARAMS_SIZE], max_block_size:u16, chain_config_block:[u8;MAX_BLOCK_SIZE], crc32:u32`.
- Constant constraints: `INIT_PARAMS_SIZE = 100`; `PRIVATE_KEY_SIZE` is imported from `moonblokz-crypto-lib`.
- Compatibility constraints: runtime constants from the binary must be checked against persisted size fields and return explicit errors on mismatch.
- Layout constraints: control-plane replicas are stored at the beginning of storage space (`data_storage_start_address` onward), and block storage begins after reserved control-plane pages.
- RP2040 fit constraint: control-plane data (including reserved chain-config block space) must fit in one flash page per replica with compile-time enforcement.
- Responsibility constraints: storage must not perform chain-policy actions (no implicit pruning/retention behavior).
- Simplicity-first constraints: prefer the simplest implementation that satisfies the contract; avoid unnecessary abstractions, derives, defaults, helper layers, and optional features that increase CPU/RAM/flash usage.

### Integration Requirements

- Integrate with MoonBlokz chain/runtime init, ingest, and query flows.
- Maintain compatibility with the blockchain types codebase for canonical block definitions, `Block` serialized-byte boundary, and SHA-256 hash utility contract.
- Preserve backend abstraction so chain logic remains stable across current and future hardware implementations.

### Risk Mitigations

- **Risk:** Partial writes produce invalid block state.
  - **Mitigation:** detect on read/startup, return explicit error, let chain logic decide recovery.
- **Risk:** Testing becomes blocked by RP2040 hardware dependence.
  - **Mitigation:** implement a dummy in-memory test storage backend with identical API semantics for off-target blockchain testing.
- **Risk:** Storage/chain responsibility drift creates ambiguous behavior.
  - **Mitigation:** enforce API boundary and responsibility matrix with contract tests.
- **Risk:** Backend-specific divergence breaks semantic portability.
  - **Mitigation:** conformance tests for deterministic index mapping and integrity behavior across backends.

## Project-Type Requirements

### Project-Type Overview

MoonBlokz storage is a Rust `no_std` developer library consumed by MoonBlokz chain/runtime code. The immediate delivery model is source-distributed integration via Git dependency, with planned later publication to `crates.io`. The library's primary role is deterministic indexed block persistence/retrieval with integrity-safe behavior under embedded constraints.

### Technical Architecture Considerations

#### Language Matrix

- Rust-only implementation and consumption model.
- `no_std` required for embedded runtime compatibility.
- Public API must be callable from Rust chain/runtime code without non-Rust bindings.

#### Installation Methods

- Initial distribution via GitHub Git dependency.
- Future distribution target: `crates.io` package publication.
- Versioning and release process should preserve compatibility for chain/runtime integrators across both distribution channels.

#### API Surface

- API must expose deterministic indexed persistence/retrieval semantics aligned with prior sections:
  - index-based addressing model
  - read-time hash verification behavior
  - explicit error signaling on invalid/partial/corrupt data
- API contract should remain backend-agnostic to support RP2040 now and additional devices later.

#### Code Examples

- Every public function must include documentation covering:
  - input parameter descriptions
  - at least one concrete usage example
- Examples should reflect real MoonBlokz integration flows where possible (startup load, persist accepted block, query retrieval, error handling).

#### Migration Guide

- No formal migration guide is required for MVP.
- Rationale: current state is no-storage baseline; integration proceeds as first-time adoption rather than framework migration.

### Implementation Considerations

- Documentation is mandatory as a product requirement, not optional polish:
  - block comment at the beginning of every source file
  - comments on functions, structs, and fields
  - inline explanatory comments where needed for non-obvious logic
- `README.md` is required and must include API description and usage guidance.
- A "how to add new device support" guide is required to enable backend expansion after RP2040 MVP.
- Keep project-type scope focused: no visual design/store-compliance concerns for this developer-tool PRD.

## Project Scoping & Phased Development

### MVP Strategy & Philosophy

**MVP Approach:** problem-solving MVP  
**Resource Requirements:** 1 Rust embedded engineer

The MVP is explicitly scoped to unblock MoonBlokz operational viability on constrained hardware as quickly as possible, with strict focus on deterministic storage behavior and integration readiness rather than platform breadth.

### MVP Feature Set (Phase 1)

**Core User Journeys Supported:**
- Chain Runtime success path (boot reconstruction + normal persist/retrieve flow)
- Chain Runtime edge-case recovery (partial/invalid data detection and explicit error handling)
- MoonBlokz Chain Developer integration journey
- MoonBlokz Node Operator troubleshooting visibility
- Storage Backend Implementer baseline contract journey (for future expansion readiness)

**Must-Have Capabilities:**
- Complete RP2040 storage backend for MoonBlokz MVP operation.
- Deterministic `storage_index -> flash address` mapping.
- Read-time hash verification before returning block data.
- Explicit error signaling for invalid/partial/corrupt reads.
- Startup-compatible block loading behavior for chain initialization.
- Dummy in-memory storage backend for off-target blockchain testing.
- Separate blockchain types crate with block data structure definition and clear boundary to storage crate.
- Mandatory developer documentation:
  - file-level block comments
  - function/struct/field documentation
  - function parameter descriptions and at least one usage example per function
  - `README.md` with API description
  - guide for adding new device support

### Post-MVP Features

**Phase 2 (Post-MVP):**
- Publish storage crate to `crates.io` (after initial Git dependency phase).
- Add additional hardware backend implementations beyond RP2040.
- Expand backend conformance tests/tooling to enforce semantic consistency across targets.

### Risk Mitigation Strategy

**Technical Risks:**  
Constrain MVP to deterministic core semantics; enforce strict API boundaries; validate integrity and failure behavior with RP2040 + in-memory backend contract tests.

**Market Risks:**  
Project is open-source engineering infrastructure; primary risk is adoption readiness rather than commercial demand. Mitigate by delivering integration-ready MVP and clear developer-facing docs/examples.

**Resource Risks:**  
With a 1-engineer MVP team, scope discipline is critical. Mitigate by excluding non-essential phases/features (no Phase 3 in current plan), and prioritizing contract correctness over breadth.

## Functional Requirements

### Storage Lifecycle Management

- FR1: MoonBlokz Chain Runtime can initialize storage on supported backend devices.
- FR2: MoonBlokz Chain Runtime can perform startup loading by iterating through storage indexes and reading blocks for reconstruction.
- FR3: MoonBlokz Chain Runtime can detect whether storage state is usable for chain reconstruction.
- FR4: MoonBlokz Chain Runtime can retrieve storage capacity boundaries derived from configured block sizing constraints.
- FR5: MoonBlokz Chain Runtime can determine availability of indexed storage slots.

### Control Plane Management

- FR6: MoonBlokz Chain Runtime can call `init(private_key, own_node_id, init_params)` before first use to initialize storage state.
- FR7: Storage can accept `private_key: [u8; PRIVATE_KEY_SIZE]`, `own_node_id: u32`, and `init_params: [u8; INIT_PARAMS_SIZE]` in `init()`, where `INIT_PARAMS_SIZE = 100`.
- FR8: Storage can erase (set to zero) all control-plane pages and all block-storage pages during `init()` before writing initial control-plane data.
- FR9: Storage can enforce immutable control-plane initialization data after `init()`, except when `init()` is called again to restart from empty state.
- FR10: Storage can persist a set-once chain configuration block via `set_chain_configuration(Block)` and return an explicit error if it was already set.
- FR11: Storage can load all control-plane data through `load_control_data()` and return private key, own node id, init parameters, and optional chain configuration block.
- FR12: Storage can return an explicit error from `load_control_data()` when storage has not been initialized.
- FR13: Storage can persist control-plane schema fields including version constant, persisted size fields, `max_block_size`, reserved `chain_config_block` space, and `crc32`.
- FR14: Storage can validate persisted size fields against runtime constants from the binary and return an explicit error on mismatch.
- FR15: Storage can reserve and persist chain-configuration block space during `init()` even when no chain configuration is set yet.
- FR16: Storage can write control-plane data to `CONTROL_PLANE_COUNT` replicated instances on every modifying operation (`init`, `set_chain_configuration`).
- FR17: Storage can read control-plane replicas in deterministic order, validate CRC32, return the first valid instance, and continue scanning on checksum failure.
- FR18: Storage can attempt best-effort repair of failed control-plane replicas after reading one valid replica.
- FR19: Storage can place control-plane replicas at the start of storage region (`data_storage_start_address`) and map block-storage addresses after the reserved control-plane area.
- FR20: Storage can calculate block-storage slot capacity from remaining storage after subtracting reserved replicated control-plane pages.

### Indexed Persistence & Retrieval

- FR21: MoonBlokz Chain Runtime can persist a blockchain block at a specific `storage_index`.
- FR22: MoonBlokz Chain Runtime can retrieve a blockchain block from a specific `storage_index`.
- FR23: Storage can map `storage_index` deterministically to physical storage address space.
- FR24: Storage can reject persistence operations targeting invalid or out-of-range indices.
- FR25: Storage can report whether a requested indexed block is absent, valid, or invalid.

### Integrity & Error Semantics

- FR26: Storage can recompute and verify block hash integrity on retrieval before returning block data.
- FR27: Storage can return explicit integrity errors when computed and stored block hashes differ.
- FR28: Storage can detect partial or invalid write artifacts during subsequent read/startup operations.
- FR29: Storage can surface explicit error categories suitable for chain-level recovery decisions.
- FR30: Storage can avoid returning data that fails integrity validation.

### Chain Integration Contract

- FR31: MoonBlokz Chain Runtime can use storage APIs from `no_std` synchronous Rust execution contexts.
- FR32: MoonBlokz Chain Runtime can execute startup reconstruction as a chain-initiated indexed read cycle using documented storage APIs.
- FR33: MoonBlokz Chain Runtime can persist chain-accepted blocks through storage APIs during normal ingest flow.
- FR34: MoonBlokz Chain Runtime can execute query retrieval flow via storage APIs.
- FR35: Storage can preserve strict separation of concerns where chain-policy decisions remain external to storage.

### Backend Abstraction & Portability

- FR36: Storage can expose a backend-agnostic interface supporting multiple device implementations.
- FR37: Storage Backend Implementer can implement a new hardware backend without changing chain-level storage semantics.
- FR38: Storage can provide an RP2040 backend implementation for MVP usage.
- FR39: Storage can provide an in-memory backend implementation for off-target blockchain testing.
- FR40: Storage can support conformance validation of backend implementations against shared API semantics.

### Blockchain Types Boundary

- FR41: MoonBlokz Chain Developer can use block data structures from a dedicated blockchain types crate separated from storage.
- FR42: Storage can consume canonical block definitions and the `calculate_hash(&[u8]) -> [u8; HASH_SIZE]` contract via the blockchain types boundary.
- FR43: Storage can operate without owning chain-policy metadata semantics beyond required persistence integrity metadata.

### Developer Experience & Distribution

- FR44: MoonBlokz Chain Developer can integrate storage as a Git dependency during initial adoption.
- FR45: MoonBlokz Chain Developer can integrate storage as a published crate after `crates.io` release.
- FR46: Storage can provide stable versioned public APIs suitable for downstream dependency management.
- FR47: Storage can provide API-level usage documentation sufficient for integrator onboarding.

### Documentation & Maintainability

- FR48: Developers can access file-level module documentation at the start of each source file.
- FR49: Developers can access function-level documentation including input parameter descriptions.
- FR50: Developers can access struct and field documentation for public data models.
- FR51: Developers can access at least one usage example per public function.
- FR52: Developers can access a `README.md` with API overview and integration guidance.
- FR53: Developers can access a guide describing how to add support for a new device/backend.

## Non-Functional Requirements

### Performance

- NFR1: Storage operations shall use effective algorithms appropriate for constrained embedded hardware.
- NFR2: Storage behavior shall remain deterministic and bounded across startup/read/write paths.
- NFR3: Performance expectations shall be interpreted relative to hardware capabilities (RP2040 flash/RAM and execution model), not fixed universal SLA numbers.

### Security

- NFR4: Storage shall enforce integrity-focused behavior by validating block hash consistency on retrieval paths.
- NFR5: Storage shall never return data that fails integrity validation.
- NFR6: Encryption requirements are out of scope for this storage layer; security focus is integration-safe integrity handling.

### Reliability

- NFR7: Storage shall return explicit, actionable errors for integrity mismatches and partial/invalid write artifacts.
- NFR8: Storage shall avoid silent fallback behavior when invalid data is encountered.
- NFR9: Startup/read behavior shall remain predictable and consistent under failure conditions to support deterministic chain-level recovery.
- NFR10: Control-plane modifications (`init`, `set_chain_configuration`) shall write all `CONTROL_PLANE_COUNT` replicas deterministically.
- NFR11: Control-plane reads shall validate CRC32 and perform bounded fallback scanning across replicas.
- NFR12: When at least one valid control-plane replica is found, storage shall attempt best-effort repair of invalid replicas without breaking deterministic read behavior.

### Integration

- NFR13: Storage integration shall remain compatible with MoonBlokz Embassy-based runtime architecture.
- NFR14: Public storage interfaces shall remain Rust `no_std` and synchronous.
- NFR15: Although async storage techniques (for example DMA-enabled approaches) may exist, this product shall intentionally use synchronous APIs because RP2040 XIP flash operations block both cores.
- NFR16: Storage shall preserve strict architectural boundaries with chain logic and blockchain types crate responsibilities.
- NFR17: RP2040 and non-RP2040 backend implementations (including in-memory test backend) shall preserve consistent API semantics and behavioral contracts.

### Implementation Simplicity

- NFR18: Implementation choices shall be simplicity-first for embedded limits, using minimal logic and minimal state needed to satisfy requirements.
- NFR19: New abstractions, traits, derives, or helper layers that increase binary size or runtime overhead shall require explicit justification in architecture/task artifacts.
- NFR20: Documentation and code reviews shall treat unnecessary complexity as a defect for MVP scope.
