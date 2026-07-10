# MoonBlokz Storage Architecture Decision Document

## Authoritative Source Notice

This document is the **authoritative source** for the architecture of the `moonblokz-storage` library and its companion `moonblokz-chain-types` crate.

When other storage knowledge-base files (concept, algorithm, implementation) describe crate boundaries, public API surfaces, backend isolation rules, feature gating, naming conventions, project structure, or block/storage data-structure contracts, the canonical wording lives in this file. If any other knowledge-base text appears to diverge from the content here, the divergence must be flagged explicitly per the rule in [`AGENTS.md`](./AGENTS.md), and resolution must be left to the user.

The behavioral requirements (FR1–FR53, NFR1–NFR20) remain anchored in [`moonblokz-storage-prd.md`](./moonblokz-storage-prd.md); this architecture document specifies how those requirements are realized in code shape.

## Provenance

- Originating workflow: BMAD `bmad-create-architecture`, eight steps completed (Steps 1–8) on 2026-02-25.
- Source artifact: `moonblokz-bmad-history/storage/planning-artifacts/architecture.md`.
- Imported into the `moonblokz-info` knowledge base on 2026-06-18.
- Frontmatter from the originating workflow output was stripped during import.
- On 2026-06-18, the body was reconciled with the current `moonblokz-storage` and `moonblokz-chain-types` codebases (verified directly against `src/backend_rp2040.rs` and the repository directory tree). The reconciliation updated the FR-count framing (53 FRs after the 2026-02-28 PRD edit pass), the RP2040 slot geometry (per-slot hash metadata), and the Project Structure section (conformance as a single source file, plus the actual repository file set). Other content remains as originally authored.

## Related Knowledge-Base Documents

- [`moonblokz-storage-prd.md`](./moonblokz-storage-prd.md) — authoritative FR1–FR53 functional requirements and NFR1–NFR20 Non-Functional Requirements that this architecture realizes.
- [`moonblokz-storage-concept.md`](./moonblokz-storage-concept.md) — conceptual operating model grounded in the current code.
- [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) — formal algorithm description grounded in the current code.
- [`moonblokz-storage-implementation.md`](./moonblokz-storage-implementation.md) — engineering consequences grounded in the current code.

---

## Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**
The PRD defines a broad but coherent capability contract (FR1–FR53) centered on deterministic embedded storage for blockchain data. Architecturally, these requirements imply clear subsystem boundaries: storage lifecycle management, control-plane management, indexed persistence/retrieval, integrity/error semantics, runtime integration, backend portability via feature selection, blockchain-types contract separation, and documentation/distribution discipline. The feature set is implementation-agnostic at PRD level but strongly constrains architecture toward explicit interfaces and predictable state handling.

**Non-Functional Requirements:**
The strongest architectural drivers are deterministic behavior, bounded operation in constrained hardware contexts, integrity-safe retrieval semantics, and explicit error signaling under failure conditions. Integration constraints are especially important: Embassy-compatible runtime behavior, Rust `no_std` synchronous interfaces, and consistent semantics across RP2040 and in-memory test backend implementations. Encryption is explicitly out of scope at this layer.

**Scale & Complexity:**
Project scale is small-to-medium in scope with high correctness sensitivity. The requirements indicate a focused architecture with limited component breadth and strict contract discipline.

- Primary domain: embedded Rust storage library for blockchain runtime integration
- Complexity level: medium
- Estimated architectural components: 6-7 focused components

### Technical Constraints & Dependencies

- RP2040 hardware constraints (flash/RAM) and XIP behavior materially affect architectural decisions.
- APIs must remain Rust `no_std` and synchronous.
- Chain runtime drives startup as indexed read cycles; storage does not own chain-policy behavior.
- Separate blockchain-types crate is required; storage depends on canonical block definitions and hashing utility through explicit boundary contracts.
- Distribution model starts with Git dependency and later crates.io packaging; architecture should support versioned API stability.
- Simplicity-first implementation policy is mandatory: choose the most direct design that meets the contract and avoid optional abstraction layers that increase flash/RAM/CPU cost.
- Backend selection model should follow the moonblokz-crypto-lib pattern:
  - Cargo feature-based backend selection
  - compile-time enforcement that exactly one backend implementation is active
  - no common backend implementation code shared between backend modules
- Integrity/hash verification and error behavior are implemented inside each backend module, while conforming to shared public API contracts.

### Cross-Cutting Concerns Identified

- Determinism across startup/read/write flows
- Data integrity verification before data return
- Explicit, actionable error semantics for chain-level recovery
- Strict separation of responsibilities (storage mechanics vs chain policy)
- Feature-selected backend isolation with behavioral conformance
- Off-target testability via in-memory backend
- Documentation quality as an architectural deliverable (API usability and maintainability)

## Starter Template Evaluation

### Primary Technology Domain

Embedded Rust `no_std` library architecture with Cargo feature-selected backend implementations.

### Starter Options Considered

1. `cargo new --lib` (official Cargo baseline)
- Best fit for library-first architecture.
- Minimal scaffold, no firmware assumptions.
- Supports clean crate boundaries and feature gating.

2. `cargo-generate` template workflow
- Useful for repeatable scaffolding if a MoonBlokz template is later introduced.

3. `rp-rs/rp2040-project-template`
- Strong for firmware binaries, not ideal as primary starter for reusable library crates.

### Selected Starter: Dual Library Initialization with Cargo

**Rationale for Selection:**
The project explicitly requires two reusable crates with strict boundary separation: `moonblokz-storage` and `moonblokz-chain-types`. Official Cargo library initialization provides the cleanest baseline for this split.

**Initialization Commands:**

```bash
cargo new --lib moonblokz-storage
cargo new --lib moonblokz-chain-types
```

**Architectural Decisions Provided by Starter:**

**Language & Runtime:**
- Rust library baselines ready for `#![no_std]`.

**Build Tooling:**
- Standard Cargo/Rust toolchain and package conventions.

**Code Organization:**
- `moonblokz-chain-types` is the canonical block/hasher boundary crate.
- `moonblokz-storage` depends on `moonblokz-chain-types`.
- Backend selection and compile-time exclusivity rules live in `moonblokz-storage`.
- No shared backend implementation code across backend modules.

**Development Experience:**
- Low-friction initialization with no unnecessary firmware application boilerplate.

**Note:** Initial implementation stories should include creating both crates and establishing contract boundaries before backend-specific implementation.

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- Two separate repositories:
  - `moonblokz-storage`
  - `moonblokz-chain-types`
- `moonblokz-storage` depends on `moonblokz-chain-types` (path dependency allowed during development/testing).
- Backend selection in `moonblokz-storage` uses Cargo features with strict compile-time exclusivity:
  - exactly one backend feature must be enabled
  - compile error if zero or multiple backend features are enabled
- Public storage API remains Rust `no_std` and synchronous.
- Backend implementations remain isolated:
  - no shared backend implementation code
  - integrity/hash/error behavior implemented within each backend module.

**Important Decisions (Shape Architecture):**
- Rust toolchain policy: stable channel.
- CI/test strategy follows `moonblokz-crypto-lib/run_tests.sh` pattern:
  - backend-by-backend test execution with `--no-default-features` + one backend feature at a time
  - matrix-style validation of feature-isolated behavior
  - combined coverage/reporting across backend runs.
- Conformance expectation: all backend implementations must preserve identical contract semantics for core storage behavior.

**Deferred Decisions (Post-MVP / Later Phase):**
- Publication of both crates to `crates.io` is planned, but deferred to a later phase after initial Git dependency adoption and stabilization.

### Data Architecture

- Primary persisted unit is blockchain block data addressed by `storage_index`.
- Canonical block types and hash utility contract are owned by `moonblokz-chain-types`.
- `moonblokz-storage` consumes canonical types via crate boundary contracts.
- Storage addressing and integrity semantics are backend-specific implementations under shared public API expectations.

### Authentication & Security

- No authentication subsystem in this library scope.
- Security focus at this layer is integrity enforcement and explicit failure signaling.
- Encryption is intentionally out of scope for this storage-layer architecture.

### API & Communication Patterns

- API style: synchronous Rust library API (`no_std` compatible).
- Integration pattern: direct in-process calls from chain runtime.
- Error communication pattern: explicit typed errors, no silent fallback behavior.
- Startup flow contract: chain-initiated indexed read cycle for reconstruction.

### Frontend Architecture

- Not applicable for this architecture scope (library-only backend component).

### Infrastructure & Deployment

- Development distribution: Git dependency usage.
- Release distribution (later phase): crates.io publication for both crates.
- CI pipeline pattern: feature-isolated backend test matrix modeled after `moonblokz-crypto-lib` workflow.

### Decision Impact Analysis

**Implementation Sequence:**
1. Initialize both repositories/crates and boundary contracts.
2. Implement compile-time backend feature policy in `moonblokz-storage`.
3. Integrate `moonblokz-chain-types` dependency and canonical block + hash-function interfaces.
4. Implement RP2040 backend and in-memory backend independently.
5. Add backend-matrix CI tests with feature-isolated runs.
6. Stabilize API/contracts before later crates.io release phase.

**Cross-Component Dependencies:**
- `moonblokz-storage` depends on stable type definitions from `moonblokz-chain-types`.
- Backend conformance depends on consistent contract tests across feature-selected implementations.
- Release readiness depends on stable API and matrix-tested backend behavior.

## Implementation Patterns & Consistency Rules

### Pattern Categories Defined

**Critical Conflict Points Identified:**
8 areas where AI agents could diverge and break compatibility:
- crate boundary usage
- feature gating behavior
- index/addressing semantics
- error model shape
- hash/integrity behavior
- naming and file organization
- test/conformance structure
- docs format and coverage

### Naming Patterns

**Crate & Module Naming Conventions:**
- Crates: `moonblokz-storage`, `moonblokz-chain-types`
- Feature names: backend-specific, lowercase kebab-case (for example `backend-rp2040`, `backend-memory`)
- Backend modules: one module per backend feature; names mirror feature intent

**Type & API Naming Conventions:**
- Public Rust types: `PascalCase`
- Public functions/methods: `snake_case`
- Constants: `SCREAMING_SNAKE_CASE`
- Storage index terminology must be consistent: `storage_index` (never alias with multiple terms)

**Error Naming Conventions:**
- Error enums/variants must be explicit and stable in naming
- Distinguish integrity failure vs invalid index vs unavailable data with separate variants

### Structure Patterns

**Repository Organization:**
- Two separate repositories:
  - `moonblokz-chain-types`
  - `moonblokz-storage`
- `moonblokz-storage` may use relative path dependency to `moonblokz-chain-types` for local development/testing.

**Backend Isolation Rules:**
- No shared backend implementation code between backend modules.
- Integrity/hash verification and backend-specific error behavior are implemented per backend.
- Shared code (if any) is limited to contract-level helpers/types that do not implement backend behavior.

**Feature Gating Rules:**
- Exactly one backend feature enabled at compile time.
- Compile-time guard must fail build when zero or multiple backend features are enabled.

### Format Patterns

**API Contract Format:**
- Public API remains synchronous and `no_std`.
- Startup reconstruction interaction pattern is chain-initiated indexed read cycle.
- Retrieval returns data only after backend-integrity verification succeeds.

**Error Contract Format:**
- No silent fallback behavior.
- Errors must be typed and actionable for chain-level recovery logic.
- Error surface must be semantically consistent across all backend features.

### Communication Patterns

**Cross-Crate Contract Pattern:**
- `moonblokz-chain-types` is the canonical source for block types and hash utility contract.
- `moonblokz-storage` consumes those types without redefining canonical semantics.
- Any contract changes in types crate require explicit compatibility review in storage crate.

**Versioning Communication Pattern:**
- During Git-dependency phase, pin commit/tag intentionally.
- Later crates.io phase must preserve API contract continuity across versions.

### Process Patterns

**Testing & Conformance Pattern:**
- Follow `moonblokz-crypto-lib/run_tests.sh` style:
  - run backend tests with `--no-default-features`
  - enable exactly one backend feature per run
  - aggregate coverage/reporting across backend runs
- Contract/conformance tests must verify behavioral parity across backends.

**Documentation Pattern:**
- File-level module comments required.
- Public function docs include parameter descriptions and at least one usage example.
- `README.md` and “add new device support” guide must remain aligned with current feature and contract rules.

**Logging Pattern (Embedded Runtime):**
- Use the `log` crate as the canonical logging interface (same style as `moonblokz-radio-lib`).
- Use level-appropriate macros consistently: `trace!`, `debug!`, `info!`, `warn!`, `error!` (or `log!` with explicit `Level` where needed).
- Keep logging deterministic and low-overhead for embedded builds; avoid allocation-heavy formatting in hot paths.

**Binary Size Discipline Pattern (Embedded):**
- Prefer minimal implementations for `no_std` targets; avoid convenience features that are not required by public contract behavior.
- Avoid nonessential trait derives (for example `Clone`, `Debug`, `PartialEq`, `Eq`, `Default`) on production types unless explicitly justified by runtime requirements.
- Avoid `Default` trait implementations for core data models/builders when explicit initialization is sufficient.
- Prefer fixed-size parsing/encoding helpers that avoid panic-prone conversions and extra abstraction overhead.
- Keep optional diagnostics/dev helpers behind test-only usage or explicit feature gates.
- Prefer single-purpose functions and data paths over reusable-but-heavy generic layers for MVP embedded targets.

### Enforcement Guidelines

**All AI Agents MUST:**
- Respect strict crate boundary: types in `moonblokz-chain-types`, storage behavior in `moonblokz-storage`.
- Implement backend behavior only inside backend-specific modules.
- Preserve synchronous `no_std` API and explicit error semantics.
- Use canonical naming (`storage_index`, consistent error variants, stable feature names).
- Follow backend-matrix test pattern before merging implementation changes.
- Use `log` crate macros for runtime logging (aligned with `moonblokz-radio-lib` patterns), not ad hoc logging mechanisms.
- Apply binary-size discipline rules for embedded targets and justify any added derive/trait/abstraction that increases image size.
- Treat unnecessary implementation complexity as a defect and simplify before adding features.

**Pattern Enforcement:**
- CI gates: feature exclusivity checks + per-backend test runs + conformance suite.
- PR review checklist includes boundary, feature, and contract consistency checks.
- Pattern violations are treated as architectural defects, not style issues.

### Pattern Examples

**Good Examples:**
- Compile error if both `backend-rp2040` and `backend-memory` are enabled.
- Backend modules each implement their own integrity verification logic while exposing identical public contract behavior.
- Storage crate uses canonical block definitions and hash utility imported from types crate rather than redefining equivalents.

**Anti-Patterns:**
- Shared “common backend implementation” module used by multiple backend features.
- Returning partially valid data with warnings instead of explicit typed errors.
- Mixing chain-policy behavior (pruning/retention decisions) into storage implementation.
- Divergent backend behavior for the same API contract call.

## Project Structure & Boundaries

### Complete Project Directory Structure

```text
moonblokz-chain-types/
├── Cargo.toml
├── Cargo.lock
├── README.md
├── LICENSE
├── CHANGELOG.md
├── docs/
│   ├── block-data-structure.md
│   └── release-process.md
└── src/
    ├── lib.rs
    ├── block.rs
    ├── hash.rs
    ├── error.rs
    ├── transaction.rs
    └── balance.rs

moonblokz-storage/
├── Cargo.toml
├── Cargo.lock
├── README.md
├── LICENSE
├── CHANGELOG.md
├── run_tests.sh
├── docs/
│   ├── prd.md
│   ├── architecture.md
│   ├── add-new-device-support.md
│   └── release-process.md
├── src/
│   ├── lib.rs
│   ├── types.rs
│   ├── error.rs
│   ├── backend_memory.rs
│   ├── backend_rp2040.rs
│   └── conformance.rs
├── examples/
│   ├── moonblokz-storage-std-example/
│   │   ├── Cargo.toml
│   │   └── src/main.rs
│   └── moonblokz-storage-embedded-example/
│       ├── Cargo.toml
│       ├── memory.x
│       ├── build.rs
│       └── src/main.rs
├── scripts/
│   ├── run_backend_matrix.sh
│   └── check_backend_features.sh
└── .github/
    └── workflows/
        └── rust.yml
```

### Architectural Boundaries

**API Boundaries:**
- `moonblokz-storage` exposes a trait-defined public contract via `StorageTrait` in `src/lib.rs`.
- `moonblokz-chain-types` exposes canonical block/type contracts and a canonical hashing function.

**Component Boundaries:**
- `moonblokz-chain-types` contains type definitions, invariants, SHA-256 hashing utility contract, and serialized-byte access through `Block`.
- `moonblokz-storage` contains:
  - `StorageTrait` contract in `lib.rs`
  - backend implementations in isolated backend modules.
- Each backend module implements `StorageTrait` on its own concrete struct.

**Service Boundaries:**
- No network/service boundary in this scope (library-level architecture).
- Integration boundary is crate-to-crate contract usage.

**Data Boundaries:**
- Canonical block representation and hashing utility contract are owned by `moonblokz-chain-types`.
- Physical storage layout and backend-specific behavior are owned by backend modules in `moonblokz-storage`.

### Requirements to Structure Mapping

**Feature/FR Mapping (PRD groups → realized location):**
- FR1–FR5 (Storage Lifecycle Management) -> `StorageTrait` surface in `moonblokz-storage/src/lib.rs` and the conformance scenarios in `src/conformance.rs`.
- FR6–FR20 (Control Plane Management) -> control-plane serialization, replication, validation, and best-effort repair logic implemented inside each backend module (`src/backend_memory.rs`, `src/backend_rp2040.rs`); `ControlPlaneData` and related public constants in `src/lib.rs`.
- FR21–FR25 (Indexed Persistence & Retrieval) -> `storage_index` mapping and save/read paths inside each backend module; out-of-range rejection through `StorageError::InvalidIndex` defined in `src/error.rs`.
- FR26–FR30 (Integrity & Error Semantics) -> backend-local integrity logic (RP2040 per-slot hash verification in `backend_rp2040.rs`; memory backend parseability and empty-slot detection in `backend_memory.rs`); typed error surface in `src/error.rs`; cross-backend behavior parity in `src/conformance.rs`.
- FR31–FR35 (Chain Integration Contract) -> synchronous `no_std` `StorageTrait` in `src/lib.rs` and the explicit boundary against chain-policy responsibilities.
- FR36–FR40 (Backend Abstraction & Portability) -> compile-time backend feature exclusivity guard and selected-backend alias in `src/lib.rs`; in-memory backend in `src/backend_memory.rs`; RP2040 backend in `src/backend_rp2040.rs`; conformance suite in `src/conformance.rs`.
- FR41–FR43 (Blockchain Types Boundary) -> `moonblokz-chain-types/src/*` (canonical `Block`, `calculate_hash`, `HASH_SIZE`, `MAX_BLOCK_SIZE`) consumed by `moonblokz-storage/src/types.rs` and the backend modules.
- FR44–FR47 (Developer Experience & Distribution) -> crate metadata in `Cargo.toml`, `README.md`, `CHANGELOG.md`, and `docs/release-process.md` for both `moonblokz-storage` and `moonblokz-chain-types`.
- FR48–FR53 (Documentation & Maintainability) -> source-level rustdoc throughout `src/*`, `README.md`, `docs/add-new-device-support.md`, and the two embedded/host example projects under `examples/`.

**Cross-Cutting Concerns:**
- Feature exclusivity enforcement -> compile-time checks in `moonblokz-storage/src/lib.rs` plus the `scripts/check_backend_features.sh` and `scripts/run_backend_matrix.sh` helpers.
- Contract parity across backends -> conformance scenarios in `src/conformance.rs` exercised by the backend matrix runner and the GitHub Actions workflow at `.github/workflows/rust.yml`.
- Documentation standards -> enforced in both repos (`README.md`, `docs/`, source rustdoc).

### Integration Points

**Internal Communication:**
- Chain runtime uses concrete backend structs directly (not trait objects) for simpler memory management/control.
- Backend structs conform to the same trait contract (`StorageTrait`), ensuring semantic consistency.

**External Integrations:**
- No mandatory third-party services in core library scope.
- Distribution path:
  - Git dependency phase first
  - crates.io publication phase later

**Data Flow:**
- Chain runtime -> concrete backend struct API -> backend-specific storage behavior
- Retrieval path -> backend integrity/hash verification -> typed result/error to caller

### File Organization Patterns

**Configuration Files:**
- `Cargo.toml` per repo with explicit feature declarations.
- CI under `.github/workflows/ci.yml`.
- `run_tests.sh` in storage repo follows moonblokz-crypto-lib backend matrix style.

**Source Organization:**
- `lib.rs` contains public trait contract, feature guards, and exports.
- Backend files are isolated per feature with no shared backend implementation code.
- Contract/conformance checks live in dedicated conformance modules.

**Test Organization:**
- Contract tests per backend under `tests/`.
- Feature exclusivity tests validate compile-time behavior.
- Conformance tests validate semantic parity between backends.

**Documentation Organization:**
- `README.md` in both repos for quick integration guidance.
- `docs/` in both repos for architecture rules, contracts, and extension guides.

### Development Workflow Integration

**Development Structure:**
- Library-focused workflow (`cargo check`, `cargo test`) per repo.
- Local path dependency allowed between repos for development.

**Build/Test Structure:**
- Backend matrix testing with `--no-default-features` and one backend feature per run.
- Coverage/report aggregation across backend runs.

**Deployment Structure:**
- Initial Git dependency integration.
- Later structured crates.io releases for both crates after contract stabilization.

## Architecture Validation Results

### Coherence Validation ✅

**Decision Compatibility:**
Core decisions are compatible and internally consistent:
- Two-repo model aligns with strict boundary goals.
- Feature-selected backend model aligns with compile-time exclusivity and isolated backend implementation.
- Synchronous `no_std` API aligns with RP2040/XIP constraints and explicit design intent.
- Chain runtime using concrete backend structs aligns with memory/control simplification goals while preserving trait-based contract discipline.

**Pattern Consistency:**
Patterns reinforce decisions:
- Naming, structure, and feature rules match backend isolation strategy.
- Error/integrity patterns are explicit and backend-local, with conformance parity expectations.
- Testing pattern mirrors established `moonblokz-crypto-lib` multi-feature backend verification approach.

**Structure Alignment:**
Project structure supports architecture:
- Repositories and directories cleanly represent boundaries.
- Trait contract location (`lib.rs`) and backend module layout are unambiguous.
- Conformance/test/documentation layout supports consistent AI-agent implementation behavior.

### Requirements Coverage Validation ✅

**Epic/Feature Coverage:**
No epics loaded in this workflow context; architecture was validated against PRD-defined capability contract and accepted project constraints.

**Functional Requirements Coverage:**
All key FR groups are architecturally supported:
- lifecycle/init/read cycle
- indexed persistence/retrieval
- integrity/error semantics
- backend abstraction and portability
- blockchain-types boundary
- developer distribution/documentation obligations

**Non-Functional Requirements Coverage:**
NFRs are supported by architectural choices:
- deterministic/bounded operation
- integrity-first behavior
- explicit failure signaling
- Embassy-compatible integration posture
- synchronous `no_std` API commitment
- backend semantic conformance expectations

### Implementation Readiness Validation ✅

**Decision Completeness:**
Critical implementation-blocking decisions are explicit:
- repo topology
- trait + concrete backend usage model
- feature exclusivity policy
- CI/test matrix pattern
- release-phase direction (Git first, crates.io later)

**Structure Completeness:**
Directory structures are concrete and implementation-oriented for both repos, including source, tests, CI, docs, and examples.

**Pattern Completeness:**
Conflict-prone areas are covered:
- feature gating
- naming consistency
- boundary enforcement
- backend parity validation
- documentation requirements

### Gap Analysis Results

**Critical Gaps:** none identified for current architecture-definition phase.

**Important Gaps:**
- Add explicit conformance test-case catalog definition before implementation begins.

**Nice-to-Have Gaps:**
- Add concise architecture decision index (ADR-style) in docs for long-term maintenance.

### Validation Issues Addressed

- Complexity level set to medium (small-to-medium functional scope with high correctness sensitivity).
- Structure simplified to use `lib.rs` as canonical public contract location.
- Backend behavior model clarified:
  - trait-defined API
  - concrete backend struct usage by chain logic
  - backend-local integrity/hash/error implementation
  - no shared backend implementation code.

### Architecture Completeness Checklist

**✅ Requirements Analysis**
- [x] Project context thoroughly analyzed
- [x] Scale and complexity assessed
- [x] Technical constraints identified
- [x] Cross-cutting concerns mapped

**✅ Architectural Decisions**
- [x] Critical decisions documented
- [x] Technology stack fully specified
- [x] Integration patterns defined
- [x] Constraint-driven architecture choices documented

**✅ Implementation Patterns**
- [x] Naming conventions established
- [x] Structure patterns defined
- [x] Communication/contract patterns specified
- [x] Process/testing patterns documented

**✅ Project Structure**
- [x] Complete two-repo structure defined
- [x] Component boundaries established
- [x] Integration points mapped
- [x] Requirements-to-structure mapping completed

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** high

**Key Strengths:**
- Clear contract boundaries between crates
- Strong backend-isolation model
- Deterministic `no_std` sync API alignment with hardware/runtime constraints
- Feature-matrix testing model grounded in existing MoonBlokz practice
- Explicit documentation and extension expectations

**Areas for Future Enhancement:**
- Formalize conformance test catalog with required behavior matrix
- Add decision-log index for architecture evolution tracking
- Define crates.io release readiness criteria in detail for later phase

### Implementation Handoff

**AI Agent Guidelines:**
- Follow architectural decisions exactly.
- Keep backend implementation code isolated by feature.
- Preserve synchronous `no_std` public API contract in `lib.rs`.
- Use concrete backend structs in chain-runtime integrations.
- Enforce compile-time single-backend feature selection.
- Validate parity via backend matrix tests.

**First Implementation Priority:**
Initialize both crates (`moonblokz-storage`, `moonblokz-chain-types`), establish trait and boundary contracts, and implement compile-time backend feature exclusivity in `moonblokz-storage`.

## Data Structure Contract Appendix

### Scope

This appendix defines architecture-level data structure contracts and ownership boundaries for:
- `moonblokz-chain-types`
- `moonblokz-storage`

### Ownership Boundaries

- `moonblokz-chain-types` owns canonical block-related types, `calculate_hash` + `HASH_SIZE` contract, and the serialized-byte contract exposed by `Block`.
- `moonblokz-storage` owns storage-facing types required for indexed persistence/retrieval and backend-local integrity/error behavior.
- `moonblokz-storage` must not redefine canonical block layout or hash-function semantics that belong to `moonblokz-chain-types`.

### Block Data Structure Contract (Focused)

**Core Representation:**
- `Block` is read-only after creation.
- `Block` internally stores raw serialized bytes in:
  - `data: [u8; MAX_BLOCK_SIZE]`
- `Block` stores `len` (effective used bytes) to distinguish valid payload length from unused tail.
- All `Block` accessors read/parse from `data` only.

**Construction Pattern:**
- `BlockBuilder` is a primary constructor path for creating new blocks from structured fields.
- `Block::from_bytes(bytes: &[u8]) -> Result<Block, BlockError>` is also supported for validated binary-form construction.
- Construction paths validate structure and size bounds before returning immutable `Block`.
- `Block` itself does not expose mutating setters.

**Binary Format Source of Truth:**
- Layout follows MoonBlokz Part V:
  - fixed-size header + variable-size payload
  - little-endian for multi-byte fields
  - payload structure depends on payload type
- `MAX_BLOCK_SIZE` is a compile-time constant from `moonblokz-chain-types`.

**Header Fields (Part V):**
- `version (u8)`
- `sequence (u32)`
- `creator (u32)`
- `mined_amount (u32)`
- `payload_type (u8)`
- `consumed_votes (u32)`
- `first_voted_node (u32)`
- `consumed_votes_from_first_voted_node (u32)`
- `previous_hash ([u8;32])`
- `signature ([u8;64])`

### Storage Type & Placement Contract

- Storage index type uses canonical naming: `storage_index`.
- RP2040 backend uses flash page-based placement with compile-time constants:
  - `FLASH_PAGE_SIZE = 4096`
  - `SLOT_HASH_OFFSET = MAX_BLOCK_SIZE`
  - `SLOT_SIZE_BYTES = MAX_BLOCK_SIZE + HASH_SIZE`
  - `BLOCKS_PER_PAGE = FLASH_PAGE_SIZE / SLOT_SIZE_BYTES` (floor)
- `BLOCKS_PER_PAGE` must be `>= 1`, otherwise the compile-time guard in `backend_rp2040.rs` rejects the configuration.
- Each slot stores the canonical block bytes followed by a per-slot integrity hash. The hash is computed over the fixed `MAX_BLOCK_SIZE` block region and stored at offset `SLOT_HASH_OFFSET` within the slot.
- Mapping rules:
  - `page_index = storage_index / BLOCKS_PER_PAGE`
  - `slot_index = storage_index % BLOCKS_PER_PAGE`
  - `byte_offset_in_page = slot_index * SLOT_SIZE_BYTES`
  - `flash_address = page_base(page_index) + byte_offset_in_page`
- A block (and its trailing hash) never crosses page boundaries.
- If `SLOT_SIZE_BYTES` does not divide page size exactly, tail bytes in each page remain reserved/unused.

### Invariants

- A block returned from storage is valid only if integrity/hash verification succeeds.
- Invalid index and unavailable block are distinct outcomes.
- Partial/invalid persisted data must produce explicit typed errors.
- Backend implementations must preserve identical public contract semantics for the same input conditions.

### Serialization Contract

- Canonical serialized bytes are represented directly by `Block` (`as_bytes` / `serialized_bytes`) in `moonblokz-chain-types`.
- Storage backends consume and persist this canonical `Block` byte form without backend-specific reinterpretation of canonical type meaning.
- Any `Block` binary layout change is a cross-crate compatibility event.

### Hashing Contract

- Canonical typed hash methods are exposed by `moonblokz-chain-types` on chain objects (`Block::hash()`, `BlockView::hash()`, `TransactionView::hash()`, and owned transaction builders' `hash()` methods).
- The raw-byte hashing utility remains exposed as:
  - `const HASH_SIZE: usize = 32`
  - `fn calculate_hash(input: &[u8]) -> [u8; HASH_SIZE]`
- Hash algorithm is SHA-256.
- Chain logic should prefer typed `hash()` methods for canonical block/transaction identity; storage backends may still use `calculate_hash` directly for fixed-size slot-integrity byte ranges.
- Hash algorithm, output-size, or typed hash-byte-range changes are cross-crate compatibility events.

### Backend Conformance Data Requirements

- Shared conformance scenarios must include:
  - valid read/write cycle
  - hash mismatch handling
  - partial/invalid data handling
  - index boundary handling
  - error category parity checks

### Evolution Rules

- Additive changes to data structures require backward-compatibility review.
- Breaking type/binary-layout changes require explicit versioning and migration strategy definition before release-phase publication.
- Architecture and docs must be updated before implementation diverges from this contract.

### AI Agent Implementation Guidance

- Use canonical types from `moonblokz-chain-types` directly.
- Do not introduce shadow/duplicate canonical block representations or hash-function definitions in storage.
- Keep backend-local data structures isolated to backend modules.
- Validate behavior against conformance scenarios before merge.

## Related Documents

- [`moonblokz-storage-prd.md`](./moonblokz-storage-prd.md) — the requirements (FR/NFR) this architecture realizes.
- [`moonblokz-storage-concept.md`](./moonblokz-storage-concept.md) — conceptual overview of the storage subsystem.
- [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) — algorithm-level storage behavior.
- [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) — canonical block layout and `moonblokz-chain-types` that storage depends on.
- [`moonblokz-system-constraints.md`](./moonblokz-system-constraints.md) — gathers block-size and flash-geometry limits among cross-cutting limits.
