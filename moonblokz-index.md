# MoonBlokz Knowledge Base Index

## Purpose

This document serves as the table of contents for the `moonblokz-info` knowledge base. It helps readers quickly find the most relevant project-level documents and understand what each file covers before opening it.

## Contents

### 1. [MoonBlokz Overview](./moonblokz-overview.md)

A strategic introduction to the MoonBlokz project based on Part I of the Medium series.

This document explains:

- why MoonBlokz exists,
- what problem it is trying to solve,
- which environments and use cases it targets,
- what its core goals and assumptions are,
- and what conceptual boundaries define the MVP.

Use this file when you want to understand the project from a product, vision, and requirements perspective before diving into implementation details.

### 2. [MoonBlokz Technology and Architecture](./moonblokz-technology.md)

A project-level technology and architecture summary that keeps the Part II foundations but is updated using the broader MoonBlokz knowledge base.

This document explains:

- why Rust and embedded-first design still shape the project,
- which architectural invariants connect blockchain, crypto, radio, storage, telemetry, and simulator work,
- how the major subsystems fit together as one bounded portable system,
- which implementation-grounded updates sharpen the original Part II framing,
- and which technology boundaries and open questions must remain explicit.

Use this file when you want a single architectural summary of how MoonBlokz is built before reading the subsystem-specific documents.

### 3. [MoonBlokz Blockchain Concept Model](./moonblokz-blockchain-concept.md)

A conceptual blockchain behavior document based on Parts III, IV, and V of the MoonBlokz series.

This document explains:

- how MoonBlokz distinguishes nodes from addresses,
- why unreliable radio communication turns the chain into a block-tree,
- why bounded storage leads to the `snake_chain` model,
- how collection, processing, and ready phases shape blockchain behavior,
- why parseability, connectivity, and full validity may emerge in stages,
- why the blockchain module is now best understood as a semantic event state machine,
- which blockchain-facing states are primary versus derived operational views,
- how balances, UTXOs, configuration, registration, and no-global-time assumptions fit together,
- and what conceptual limitations follow from long disconnections and bounded retention.

Use this file when you want to understand the operating idea of MoonBlokz blockchain behavior before reading more formal algorithm or implementation notes.

### 4. [MoonBlokz Blockchain Product Requirements Document](./moonblokz-blockchain-prd.md) — Authoritative

The authoritative source for the `moonblokz-blockchain` module's functional requirements (FR1–FR69) and Non-Functional Requirements.

This document explains:

- the executive summary, success criteria, and product scope of the `moonblokz-blockchain` crate,
- the user journeys driving radio intake, local interface interaction, branch divergence, creator role, and simulator-driven validation,
- the domain-specific requirements and main conceptual limitations under bounded retention and constrained connectivity,
- the full FR1–FR69 functional requirement catalog covering chain lifecycle, block and transaction intake, branching, mempool, queries, block creation, `snake_chain` and tail preservation, genesis and bootstrap, external input contracts, storage and retention policy, registration and balance scheduling, lifecycle and recovery, module boundary, simulation and observability, and initialization parameters,
- and the Non-Functional Requirements covering performance, reliability, security and integrity, capacity and resource bounds, integration, and observability and testability.

Use this file as the canonical reference whenever any other knowledge-base document — concept, algorithm, implementation, or ADR — cites an FR by number. Other knowledge-base files reference this document instead of restating its content, so divergence from the PRD must be treated as exact evidence of a knowledge-base inconsistency.

### 5. [MoonBlokz Blockchain Architecture Decision Document](./moonblokz-blockchain-architecture.md) — Authoritative

The authoritative source for the architecture of the `moonblokz-blockchain` crate and its sibling sub-crates (`moonblokz-mempool`, `moonblokz-vote`, `moonblokz-node-runtime`, future `moonblokz-configuration`).

This document explains:

- the crate-split architecture (4 new crates, 2 extended crates, 2 existing crates, 1 future crate) and how they relate,
- the single-outcome scheduling-pull API pattern with `CallResult<OutcomeEnum> + NextCall` shape,
- the 20-method public API surface of the `Blockchain<C, S, X, ...>` struct (3 init methods, 4 intake, 1 tick, 12 read-only),
- the three-type model (Owned + View + Builder) for `Block` / `Transaction` and friends,
- the 15 + 1 internal modules of `moonblokz-blockchain` (`lifecycle`, `scheduler`, `blocks`, `chain_heads`, `snake_chain`, `branch_value`, `intake`, `staged_validation`, `reconciliation`, `node_info`, `spent_bits`, `creator`, `approval`, `queries`, `emit_scratch`, + the `api.rs` surface),
- the const-generic catalog (`MAX_NODES`, `SNAKE_CHAIN_LENGTH`, `MAX_BLOCKS`, `MAX_BRANCH_COUNT`, etc. with default values),
- the sized data-structure catalog with bit-level layouts (`BlockEntry` 76 B padded, `ChainHeadEntry` 32 B padded, `NodeInfo` SoA, `ApprovalAccumulator` ~2 KB crypto-agnostic, `EmitScratch`, `SchedulerState`),
- the 264 KB SRAM RAM budget verified for Schnorr (~25-29% margin) and BLS (~3-6% margin),
- the FR1–FR69 coverage matrix showing 63 hard-covered FRs, 5 delegated to the future `moonblokz-configuration` crate, and 1 MVP-skip (FR52),
- the `ChainConfigTrait` outline for the future chain-config crate,
- the BLS deployment-tuning recipe with `MAX_NODES` reduction levers,
- the consolidated 22-row decisions log, the risks list, and the implementation roadmap.

Use this file as the canonical reference whenever any other knowledge-base document — concept, algorithm, or implementation — describes a chain-lib data structure, module boundary, public API method, RAM/stack budget, const-generic value, crate dependency edge, or initialization parameter. Divergence from the Architecture Decision Document must be treated as exact evidence of a knowledge-base inconsistency.

### 6. [MoonBlokz Blockchain Algorithm Model](./moonblokz-blockchain-algorythm.md)

A more formal, algorithm-oriented document based on Parts III, IV, and V of the MoonBlokz series.

This document explains:

- how the block-tree is reconstructed,
- how missing parents are detected and recovered,
- how startup collection and reconstruction phases lead into ready operation,
- how creator selection now consumes radio-derived scoring input at the blockchain boundary,
- how grace periods and approval fallback work,
- how `snake_chain` preserves balances, configuration, and live UTXOs,
- how branch pruning, block status progression, active-chain switching, and active-chain-centered local queries affect the algorithm,
- what the main block, transaction, balance, and payload structures are,
- how uniqueness, serialization, and fragmentation boundaries affect algorithmic behavior,
- and what efficiency and failure conditions shape the algorithm.

Use this file when you want the full algorithmic flow described in Parts III, IV, and V, including the detailed main blockchain data structures, without dropping to low-level implementation detail.

### 7. [MoonBlokz Blockchain Implementation Notes](./moonblokz-blockchain-implementation.md)

An implementation-support document derived from Parts III, IV, and V of the MoonBlokz series.

This document explains:

- what state an implementation will need to track,
- how the blockchain boundary separates semantic-event handling from transport, storage mechanics, crypto backends, and radio-derived scoring,
- how bounded storage changes blockchain engineering responsibilities,
- how restart, persistence, and reconciliation should be anchored in a compact authoritative base,
- how canonical binary representation, serialization, and radio packetization affect design,
- how processing-state restart policy, pruning cost, mempool behavior, and branch-switch recomputation affect engineering design,
- which values should remain configurable,
- and which details must remain open until later series parts define them.

Use this file when you want implementation guidance that complements the conceptual and algorithm documents without guessing beyond the source material.

### 8. [MoonBlokz Blockchain ADR Index](./blockchain-adrs/ADR-INDEX.md)

A compact index of the accepted architecture decision records for the current `moonblokz-blockchain` design direction.

This document explains:

- which blockchain architecture decisions are currently accepted,
- how those decisions group into module identity, boundary, state model, mempool behavior, and rare-runtime handling,
- in what order the accepted ADRs should be read,
- and which deeper design artifacts still follow from them.

Use this file when you want the shortest path into the current accepted blockchain design decisions before reading the full implementation notes or individual ADR files.

### 9. [MoonBlokz Crypto Concept Model](./moonblokz-crypto-concept.md)

A conceptual cryptography document grounded in the current `moonblokz-crypto-lib` codebase and aligned with the earlier Part VI direction.

This document explains:

- why MoonBlokz cryptography is driven by radio, storage, and microcontroller constraints,
- why the current library distinguishes direct signatures, aggregation-ready signatures, and aggregated evidence,
- why compile-time backend replaceability is a core design principle,
- why Schnorr remains the practical default while BLS remains an important alternative,
- how deterministic signing behavior and bounded aggregation fit MoonBlokz’s embedded context,
- and what strategic limitations remain, including the absence of a post-quantum path in the current library.

Use this file when you want to understand the purpose, trade-offs, and current conceptual direction of MoonBlokz cryptography before reading the formal or implementation-facing companion documents.

### 10. [MoonBlokz Crypto Algorithm Model](./moonblokz-crypto-algorythm.md)

A formal, algorithm-oriented cryptography document based on the current `moonblokz-crypto-lib` implementation.

This document explains:

- what public traits, constants, and error types define the current crypto subsystem,
- how compile-time feature selection chooses between the Schnorr and BLS families and among concrete backends,
- what the formal roles of `Crypto`, `PublicKey`, `Signature`, `MultiSignature`, and `AggregatedSignature` are,
- how deterministic Schnorr signing, deterministically weighted aggregation, and aggregate verification work in the current code,
- how the BLS wrapper differs structurally,
- and what explicit bounds such as `MAX_AGGREGATED_SIGNATURES = 50` mean for the formal model.

Use this file when you want the exact current crypto API and behavior without dropping into maintenance or evolution guidance.

### 11. [MoonBlokz Crypto Implementation Notes](./moonblokz-crypto-implementation.md)

An implementation-support cryptography document derived from the current `moonblokz-crypto-lib` codebase.

This document explains:

- what implementation responsibilities follow from the current crypto design,
- why compile-time backend selection and stable re-exports are important,
- how the chosen library structure avoids excessive generics, trait-object complexity, and unnecessary heap dependence,
- how backend choice affects maintenance, compatibility, low-level safety, and dependency risk,
- why bounded aggregation, buffer-based serialization, and multi-feature testing must be preserved,
- and which kinds of crypto evolution remain feasible versus compatibility-sensitive.

Use this file when you want engineering guidance that complements the conceptual and algorithm crypto documents while staying grounded in the actual codebase.

### 12. [MoonBlokz Radio Concept Model](./moonblokz-radio-concept.md)

A conceptual radio document grounded in the current `moonblokz-radio-lib` codebase.

This document explains:

- why MoonBlokz treats radio as a blockchain-state synchronization layer rather than a guaranteed-delivery transport,
- how local-only topology awareness, echo mapping, the connection matrix, and adaptive relaying fit together,
- why bounded queues, static memory, and deterministic degradation are core design principles,
- how the runtime pipeline supports the higher-level mesh behavior,
- and where the current implementation is narrower or more specific than earlier article-era summaries.

Use this file when you want to understand the role, design philosophy, and current conceptual boundaries of the MoonBlokz radio subsystem before reading the more formal or implementation-facing radio notes.

### 13. [MoonBlokz Radio Algorithm Model](./moonblokz-radio-algorythm.md)

A formal, algorithm-oriented radio document grounded in the current `moonblokz-radio-lib` implementation.

This document explains:

- the actual implemented message and packet model,
- the queue-backed runtime pipeline,
- the echo-mapping and connection-matrix update behavior,
- the current TX scheduling, RX handling, relay scoring, and reactive recovery rules,
- the precise wait-pool and timed-task behavior visible in code,
- and the exact boundaries of what the current codebase still does not define.

Use this file when you want the main formal description of currently implemented MoonBlokz radio behavior.

### 14. [MoonBlokz Radio Implementation Notes](./moonblokz-radio-implementation.md)

An implementation-support radio document grounded in the current `moonblokz-radio-lib` codebase.

This document explains:

- what engineering responsibilities follow from the current crate structure and public API,
- how feature flags, memory profiles, queues, and backends shape implementation behavior,
- which board-specific and backend-specific constraints are explicit today,
- why Embassy-based async execution, static allocation, and bounded queues are central to the design,
- how the current implementation differs from or sharpens earlier article-era summaries,
- and which details still remain open for later code evolution.

Use this file when you want practical engineering guidance for implementing or extending the documented MoonBlokz radio behavior without inventing details beyond the current sources.

### 15. [MoonBlokz Simulator Concept Model](./moonblokz-simulator-concept.md)

A conceptual simulator document grounded in the current `moonblokz-radio-simulator` codebase.

This document explains:

- how the application now combines simulation, live tracking, and historical replay on one desktop surface,
- why reuse of the real `moonblokz-radio-lib` remains central,
- what realities the current simulator tries to preserve,
- what deliberate simplifications still define its scope,
- and how scene files, measurements, visualization, and connection-matrix inspection fit together conceptually.

Use this file when you want to understand what the simulator currently is and what role it plays in MoonBlokz before reading the more formal or implementation-facing companion documents.

### 16. [MoonBlokz Simulator Algorithm Model](./moonblokz-simulator-algorythm.md)

A formal, algorithm-oriented simulator document grounded in the current `moonblokz-radio-simulator` implementation.

This document explains:

- the actual mode-dispatch and runtime structure,
- the validated scene-file model,
- the simulation-side node/network event flow,
- the current path-loss, airtime, CAD, SINR, collision, and obstacle rules,
- the analyzer-side log replay and real-time tracking behavior,
- the simulator-side control invocation boundary,
- and the exact bounded histories and queues visible in code.

Use this file when you want the main formal description of the current simulator and analyzer behavior without dropping all the way to code-level maintenance detail.

### 17. [MoonBlokz Simulator Implementation Notes](./moonblokz-simulator-implementation.md)

An implementation-support simulator document grounded in the current `moonblokz-radio-simulator` crate.

This document explains:

- the current crate structure and module responsibilities,
- how embedded radio-library code is reused through Embassy and `'static` queue strategies,
- how simulation, analyzer, UI, and control modules are separated,
- what bounded queues, bounded histories, and time-driver invariants shape implementation behavior,
- how `egui/eframe`, `egui_extras`, and Telemetry Hub integration fit into the application,
- and which implementation limitations should remain explicit.

Use this file when you want practical engineering guidance that complements the conceptual and algorithm simulator documents.

### 18. [MoonBlokz Telemetry Concept Model](./moonblokz-telemetry-concept.md)

A conceptual telemetry document grounded in Part VII/5 and reconciled with the current telemetry repositories.

This document explains:

- why MoonBlokz field testing needs an out-of-band telemetry architecture,
- how the Test Station, Probe, Telemetry HUB, Update Server, Log Collector, CLI, and Analyzer fit together,
- why logging, command execution, and OTA updates are treated as one operational concern,
- what architectural boundaries keep telemetry from distorting LoRa measurements,
- how analyzer/live-tracking workflows fit into the telemetry architecture,
- and how the telemetry story relates to the simulator and field-analysis workflows.

Use this file when you want to understand why the MoonBlokz telemetry system exists and how its main parts fit together before reading the more formal or implementation-facing telemetry notes.

### 19. [MoonBlokz Telemetry Algorithm Model](./moonblokz-telemetry-algorythm.md)

A formal, algorithm-oriented telemetry document grounded primarily in the reviewed telemetry repositories and aligned with Part VII/5 where still relevant.

This document explains:

- the end-to-end flow of logs, commands, polling intervals, and ordered downloads,
- the Telemetry HUB’s persistence and serving rules,
- how CLI commands are addressed and routed,
- how Probe and Node OTA flows differ formally,
- and how analyzer-side live tracking, playback, and connection-matrix reconstruction depend on the telemetry pipeline.

Use this file when you want the main formal description of the telemetry architecture described in the field-testing article.

### 20. [MoonBlokz Telemetry Implementation Notes](./moonblokz-telemetry-implementation.md)

An implementation-support telemetry document grounded in the current telemetry repositories and used to capture deployment, configuration, repository structure, and engineering cautions.

This document explains:

- what engineering responsibilities are assigned to the Probe, HUB, Update Server, Collector, CLI, and analyzer-integration layers,
- why the node–Probe USB boundary is operationally important,
- what safety and failure-mode concerns matter for OTA updates,
- how TM-tagged logs shape compatibility between deployed systems and analysis tools,
- where current deployment and documentation drift must remain explicit,
- and which implementation boundaries should remain explicit as the telemetry stack evolves.

Use this file when you want implementation guidance that complements the conceptual and algorithm telemetry documents without scattering those details across the simulator files.

### 21. [MoonBlokz Storage Concept Model](./moonblokz-storage-concept.md)

A conceptual storage document grounded in the current `moonblokz-storage` and `moonblokz-chain-types` codebases.

This document explains:

- why the storage crate is modeled as a narrow embedded contract rather than a full blockchain database,
- how the current crate separates control-plane persistence from indexed block persistence,
- why backend portability is implemented through compile-time feature selection,
- how the canonical block and hash contracts from `moonblokz-chain-types` constrain storage behavior,
- how the current memory and RP2040 backends differ conceptually,
- and where the current implementation narrows or sharpens the earlier Part VIII article framing.

Use this file when you want to understand the role, current design philosophy, and conceptual trade-offs of MoonBlokz storage before reading the more formal or implementation-facing storage notes.

### 22. [MoonBlokz Storage Algorithm Model](./moonblokz-storage-algorythm.md)

A formal, algorithm-oriented storage document grounded in the current `moonblokz-storage` implementation and the canonical `moonblokz-chain-types` block and hash contracts.

This document explains:

- the current public storage API contract,
- the control-plane record and replica-repair model,
- the upstream block-size and hash constraints storage depends on,
- the memory-backend slot rules,
- the RP2040 page and slot mapping rules,
- the current integrity and error semantics,
- and the backend conformance expectations visible in tests.

Use this file when you want the main formal description of the currently implemented MoonBlokz storage behavior.

### 23. [MoonBlokz Storage Implementation Notes](./moonblokz-storage-implementation.md)

An implementation-support storage document grounded in the current `moonblokz-storage` repository and its dependency on `moonblokz-chain-types`.

This document explains:

- the current crate structure and feature model,
- what engineering consequences follow from the memory and RP2040 backend split,
- how the canonical block layout and hash contract shape storage geometry and parsing behavior,
- how the RP2040 backend encodes flash geometry and host-side testing,
- what compatibility and binary-size considerations are explicit today,
- and which implementation boundaries remain open.

Use this file when you want implementation guidance that complements the conceptual and algorithm storage documents while staying grounded in the current codebase.

### 24. [MoonBlokz Website Summary](./website.md)

A concise summary of the current public MoonBlokz website.

This document explains:

- where the public website is available,
- what major homepage sections it currently contains,
- that the site exposes dedicated public use-case pages while the detailed use-case list is maintained in the overview document,
- what kind of public-facing material the website currently links to,
- and how the current public site is packaged for static deployment.

Use this file when you need a quick orientation to the current public web presence and its deployment form without reading the full website source.


## Suggested Reading Order

1. Start with [MoonBlokz Overview](./moonblokz-overview.md) to understand the mission, use cases, and high-level constraints.
2. Continue with [MoonBlokz Technology and Architecture](./moonblokz-technology.md) to understand the implementation strategy and architectural foundation.
3. Then read [MoonBlokz Blockchain Concept Model](./moonblokz-blockchain-concept.md) to understand the operating idea of MoonBlokz blockchain behavior.
4. Then read [MoonBlokz Blockchain Product Requirements Document](./moonblokz-blockchain-prd.md) for the authoritative FR1–FR69 functional requirements and Non-Functional Requirements; this is the canonical anchor every other blockchain document references by FR number.
5. Then read [MoonBlokz Blockchain Architecture Decision Document](./moonblokz-blockchain-architecture.md) for the authoritative crate-split, public API, internal modules, data-structure layouts, RAM budget, and decisions log; this is the canonical anchor every other blockchain document references for chain-lib implementation shape.
6. Continue with [MoonBlokz Blockchain Algorithm Model](./moonblokz-blockchain-algorythm.md) for the full Parts III, IV, and V algorithmic flow and the detailed main data structures.
7. Then read [MoonBlokz Blockchain Implementation Notes](./moonblokz-blockchain-implementation.md) for implementation-facing constraints, configuration boundaries, and cautions.
8. Then read [MoonBlokz Blockchain ADR Index](./blockchain-adrs/ADR-INDEX.md) for the currently accepted architecture decisions and their recommended reading order before moving into adjacent subsystem documents.
9. Continue with [MoonBlokz Crypto Concept Model](./moonblokz-crypto-concept.md) to understand the role, trade-offs, and constraints of MoonBlokz cryptography.
10. Then read [MoonBlokz Crypto Algorithm Model](./moonblokz-crypto-algorythm.md) for the formal crypto structure, signature roles, deterministic signing and aggregation behavior, and explicit size and count limits.
11. Continue with [MoonBlokz Crypto Implementation Notes](./moonblokz-crypto-implementation.md) for implementation-facing cryptography guidance, dependency cautions, and testing implications.
12. Then read [MoonBlokz Radio Concept Model](./moonblokz-radio-concept.md) to understand why MoonBlokz radio behaves as a synchronization-first, bounded embedded subsystem.
13. Continue with [MoonBlokz Radio Algorithm Model](./moonblokz-radio-algorythm.md) for the current code-grounded runtime flow of messages, packets, scheduling, relaying, and recovery.
14. Continue with [MoonBlokz Radio Implementation Notes](./moonblokz-radio-implementation.md) for the actual API, feature model, backend behavior, and implementation-facing cautions.
15. Then read [MoonBlokz Simulator Concept Model](./moonblokz-simulator-concept.md) to understand how MoonBlokz currently validates, observes, replays, and controls radio-network behavior through the desktop simulator application.
16. Continue with [MoonBlokz Simulator Algorithm Model](./moonblokz-simulator-algorythm.md) for the formal simulation, analyzer, timing, and control behavior.
17. Continue with [MoonBlokz Simulator Implementation Notes](./moonblokz-simulator-implementation.md) for the current crate structure, queue/lifetime strategy, UI modules, analyzer pipeline, and implementation cautions.
18. Then read [MoonBlokz Telemetry Concept Model](./moonblokz-telemetry-concept.md) to understand why field testing needs a separate telemetry architecture and how the operational components fit together.
19. Continue with [MoonBlokz Telemetry Algorithm Model](./moonblokz-telemetry-algorythm.md) for the formal flow of logs, commands, polling control, OTA, and analyzer interaction.
20. Continue with [MoonBlokz Telemetry Implementation Notes](./moonblokz-telemetry-implementation.md) for repository-level responsibilities, update-path cautions, and telemetry-specific engineering constraints.
21. Then read [MoonBlokz Storage Concept Model](./moonblokz-storage-concept.md) to understand why bounded blockchain persistence on flash is possible at all and how MoonBlokz separates recoverable block data from recovery-critical control data.
22. Continue with [MoonBlokz Storage Algorithm Model](./moonblokz-storage-algorythm.md) for the formal storage-unit layout, integrity-check rules, redundancy behavior, crash recovery, and wear-lifetime model.
23. Continue with [MoonBlokz Storage Implementation Notes](./moonblokz-storage-implementation.md) for RP2040/XIP/Embassy engineering consequences, capacity-planning cautions, and explicitly open storage design questions.
24. Then read [MoonBlokz Website Summary](./website.md) when you need a compact summary of the public website, its location, current public-facing structure, and deployment form.
