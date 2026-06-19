# MoonBlokz Storage PRD — FR/NFR Navigation Index

> **Non-authoritative navigation aid.** The authoritative source for every requirement listed here is [`moonblokz-storage-prd.md`](./moonblokz-storage-prd.md); on any divergence the PRD wins. The entries below are short navigation labels, not requirement text. Use this index to find which requirement covers a topic, then open only that one requirement in the PRD.

## How to read a single requirement without loading the whole PRD

To read just one requirement without reading the file in full:

1. Search the PRD for the requirement's anchor — functional requirements begin a line with `- FR<n>:` and non-functional requirements with `- NFR<n>:`. For FR21 the anchor pattern is `^- FR21:`; for NFR11 it is `^- NFR11:` (the trailing colon keeps `FR2`/`FR21` and `NFR1`/`NFR11` distinct).
2. Read that single line at the reported line number.

Search by anchor rather than citing a line: anchors are stable across edits, line numbers are not.

## Functional Requirements (FR1–FR53)

### Storage Lifecycle Management
| FR | Subject |
|----|---------|
| FR1 | Initialize storage on supported backend devices |
| FR2 | Startup block loading by iterating storage indexes |
| FR3 | Detect whether storage state is usable for reconstruction |
| FR4 | Retrieve storage capacity boundaries from block sizing constraints |
| FR5 | Determine availability of indexed storage slots |

### Control Plane Management
| FR | Subject |
|----|---------|
| FR6 | Call init() before first use to initialize storage state |
| FR7 | Accept private key, node ID, and init params in init() |
| FR8 | Erase all pages during init() before writing control-plane data |
| FR9 | Enforce immutable control-plane data after init() |
| FR10 | Set-once chain-configuration block via set_chain_configuration() |
| FR11 | Load all control-plane data via load_control_data() |
| FR12 | Explicit error from load_control_data() when uninitialized |
| FR13 | Persist control-plane schema fields including version and CRC32 |
| FR14 | Validate persisted size fields against runtime constants |
| FR15 | Reserve chain-config block space during init() before it is set |
| FR16 | Write control-plane data to all CONTROL_PLANE_COUNT replicas |
| FR17 | Read replicas in order, validate CRC32, return first valid |
| FR18 | Best-effort repair of failed control-plane replicas |
| FR19 | Place control-plane replicas at start of storage region |
| FR20 | Calculate block-storage slot capacity from remaining storage |

### Indexed Persistence & Retrieval
| FR | Subject |
|----|---------|
| FR21 | Persist a blockchain block at a specific storage_index |
| FR22 | Retrieve a blockchain block from a specific storage_index |
| FR23 | Map storage_index deterministically to physical address space |
| FR24 | Reject persistence operations targeting invalid or out-of-range indices |
| FR25 | Report whether an indexed block is absent, valid, or invalid |

### Integrity & Error Semantics
| FR | Subject |
|----|---------|
| FR26 | Recompute and verify block hash on retrieval |
| FR27 | Return explicit integrity errors on hash mismatch |
| FR28 | Detect partial or invalid write artifacts on read or startup |
| FR29 | Surface explicit error categories for chain-level recovery |
| FR30 | Avoid returning data that fails integrity validation |

### Chain Integration Contract
| FR | Subject |
|----|---------|
| FR31 | Use storage APIs from no_std synchronous Rust contexts |
| FR32 | Startup reconstruction via chain-initiated indexed read cycle |
| FR33 | Persist chain-accepted blocks during normal ingest flow |
| FR34 | Execute query retrieval flow via storage APIs |
| FR35 | Chain-policy decisions remain external to storage |

### Backend Abstraction & Portability
| FR | Subject |
|----|---------|
| FR36 | Backend-agnostic interface supporting multiple device implementations |
| FR37 | New hardware backend without changing chain-level storage semantics |
| FR38 | RP2040 backend implementation for MVP usage |
| FR39 | In-memory backend implementation for off-target testing |
| FR40 | Conformance validation of backends against shared API semantics |

### Blockchain Types Boundary
| FR | Subject |
|----|---------|
| FR41 | Block data structures in a dedicated blockchain types crate |
| FR42 | Consume canonical block definitions and calculate_hash contract |
| FR43 | Operate without owning chain-policy metadata semantics |

### Developer Experience & Distribution
| FR | Subject |
|----|---------|
| FR44 | Integrate storage as a Git dependency |
| FR45 | Integrate storage as a crates.io published crate |
| FR46 | Stable versioned public APIs for downstream dependency management |
| FR47 | API-level usage documentation sufficient for integrator onboarding |

### Documentation & Maintainability
| FR | Subject |
|----|---------|
| FR48 | File-level module documentation at the start of each source file |
| FR49 | Function-level documentation with input-parameter descriptions |
| FR50 | Struct and field documentation for public data models |
| FR51 | At least one usage example per public function |
| FR52 | README.md with API overview and integration guidance |
| FR53 | Guide for adding support for a new device or backend |

## Non-Functional Requirements (NFR1–NFR20)

### Performance
| NFR | Subject |
|-----|---------|
| NFR1 | Effective algorithms appropriate for constrained embedded hardware |
| NFR2 | Deterministic and bounded behavior across startup/read/write paths |
| NFR3 | Performance relative to hardware capabilities, not fixed SLA numbers |

### Security
| NFR | Subject |
|-----|---------|
| NFR4 | Validate block-hash consistency on retrieval paths |
| NFR5 | Never return data that fails integrity validation |
| NFR6 | Encryption out of scope; focus on integration-safe integrity |

### Reliability
| NFR | Subject |
|-----|---------|
| NFR7 | Explicit actionable errors for integrity mismatches and invalid writes |
| NFR8 | Avoid silent fallback behavior on invalid data |
| NFR9 | Predictable startup and read behavior under failure conditions |
| NFR10 | Control-plane modifications write all replicas deterministically |
| NFR11 | CRC32 validation and bounded fallback scanning on control-plane reads |
| NFR12 | Best-effort repair of invalid replicas after finding one valid |

### Integration
| NFR | Subject |
|-----|---------|
| NFR13 | Compatible with MoonBlokz Embassy-based runtime architecture |
| NFR14 | Public interfaces remain Rust no_std and synchronous |
| NFR15 | Synchronous APIs because RP2040 XIP flash blocks both cores |
| NFR16 | Strict architectural boundaries with chain logic and types crate |
| NFR17 | Consistent API semantics across RP2040 and non-RP2040 backends |

### Implementation Simplicity
| NFR | Subject |
|-----|---------|
| NFR18 | Simplicity-first implementation with minimal logic and state |
| NFR19 | New abstractions require explicit justification for overhead |
| NFR20 | Unnecessary complexity treated as a defect for MVP scope |

## Related Documents

- [`moonblokz-storage-prd.md`](./moonblokz-storage-prd.md) — the authoritative PRD this index points into; full requirement wording lives here.
- [`moonblokz-storage-architecture.md`](./moonblokz-storage-architecture.md) — architecture decisions, crate-split layout, and data-structure contracts realizing these requirements.
- [`moonblokz-storage-concept.md`](./moonblokz-storage-concept.md) — conceptual storage operating model.
- [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) — formal algorithms that implement these requirements.
- [`moonblokz-storage-implementation.md`](./moonblokz-storage-implementation.md) — engineering consequences derived from these requirements.
- [`moonblokz-index.md`](./moonblokz-index.md) — knowledge-base table of contents and topic finder.
