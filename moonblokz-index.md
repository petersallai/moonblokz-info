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

A technical foundation document based on Part II of the Medium series.

This document explains:

- why Rust was selected,
- how the project is structured as a portable core library,
- which modules and integration boundaries shape the architecture,
- how storage, crypto, randomness, clock, and radio are treated,
- and what technical decisions are already established versus intentionally deferred.

Use this file when you want to understand the technology stack, architectural direction, and platform abstraction model behind MoonBlokz.

### 3. [MoonBlokz Blockchain Concept Model](./moonblokz-blockchain-concept.md)

A conceptual blockchain behavior document based on Parts III, IV, and V of the MoonBlokz series.

This document explains:

- how MoonBlokz distinguishes nodes from addresses,
- why unreliable radio communication turns the chain into a block-tree,
- why bounded storage leads to the `snake_chain` model,
- how collection, processing, and ready phases shape blockchain behavior,
- why parseability, connectivity, and full validity may emerge in stages,
- why compact data representation is a protocol concern rather than just an implementation detail,
- which block types preserve active blockchain state,
- how balances, UTXOs, configuration, registration, and no-global-time assumptions fit together,
- and what conceptual limitations follow from long disconnections and bounded retention.

Use this file when you want to understand the operating idea of MoonBlokz blockchain behavior before reading more formal algorithm or implementation notes.

### 4. [MoonBlokz Blockchain Algorithm Model](./moonblokz-blockchain-algorythm.md)

A more formal, algorithm-oriented document based on Parts III, IV, and V of the MoonBlokz series.

This document explains:

- how the block-tree is reconstructed,
- how missing parents are detected and recovered,
- how startup collection and reconstruction phases lead into ready operation,
- how message-based voting selects block creators,
- how grace periods and approval fallback work,
- how `snake_chain` preserves balances, configuration, and live UTXOs,
- how branch pruning, block status progression, and active-chain switching affect the algorithm,
- what the main block, transaction, balance, and payload structures are,
- how uniqueness, serialization, and fragmentation boundaries affect algorithmic behavior,
- and what efficiency and failure conditions shape the algorithm.

Use this file when you want the full algorithmic flow described in Parts III, IV, and V, including the detailed main blockchain data structures, without dropping to low-level implementation detail.

### 5. [MoonBlokz Blockchain Implementation Notes](./moonblokz-blockchain-implementation.md)

An implementation-support document derived from Parts III, IV, and V of the MoonBlokz series.

This document explains:

- what state an implementation will need to track,
- how bounded storage changes blockchain engineering responsibilities,
- how canonical binary representation, serialization, and radio packetization affect design,
- how processing-state restart policy, pruning cost, and branch-switch recomputation affect engineering design,
- which values should remain configurable,
- what data-model constraints follow from tree-based consensus plus `snake_chain` retention,
- how bootstrap, replay scheduling, caching, flash-write constraints, and retained-history tradeoffs affect design,
- and which details must remain open until later series parts define them.

Use this file when you want implementation guidance that complements the conceptual and algorithm documents without guessing beyond the source material.

### 6. [MoonBlokz Crypto Concept Model](./moonblokz-crypto-concept.md)

A conceptual cryptography document grounded in the current `moonblokz-crypto-lib` codebase and aligned with the earlier Part VI direction.

This document explains:

- why MoonBlokz cryptography is driven by radio, storage, and microcontroller constraints,
- why the current library distinguishes direct signatures, aggregation-ready signatures, and aggregated evidence,
- why compile-time backend replaceability is a core design principle,
- why Schnorr remains the practical default while BLS remains an important alternative,
- how deterministic signing behavior and bounded aggregation fit MoonBlokz’s embedded context,
- and what strategic limitations remain, including the absence of a post-quantum path in the current library.

Use this file when you want to understand the purpose, trade-offs, and current conceptual direction of MoonBlokz cryptography before reading the formal or implementation-facing companion documents.

### 7. [MoonBlokz Crypto Algorithm Model](./moonblokz-crypto-algorythm.md)

A formal, algorithm-oriented cryptography document based on the current `moonblokz-crypto-lib` implementation.

This document explains:

- what public traits, constants, and error types define the current crypto subsystem,
- how compile-time feature selection chooses between the Schnorr and BLS families and among concrete backends,
- what the formal roles of `Crypto`, `PublicKey`, `Signature`, `MultiSignature`, and `AggregatedSignature` are,
- how deterministic Schnorr signing, deterministically weighted aggregation, and aggregate verification work in the current code,
- how the BLS wrapper differs structurally,
- and what explicit bounds such as `MAX_AGGREGATED_SIGNATURES = 50` mean for the formal model.

Use this file when you want the exact current crypto API and behavior without dropping into maintenance or evolution guidance.

### 8. [MoonBlokz Crypto Implementation Notes](./moonblokz-crypto-implementation.md)

An implementation-support cryptography document derived from the current `moonblokz-crypto-lib` codebase.

This document explains:

- what implementation responsibilities follow from the current crypto design,
- why compile-time backend selection and stable re-exports are important,
- how the chosen library structure avoids excessive generics, trait-object complexity, and unnecessary heap dependence,
- how backend choice affects maintenance, compatibility, low-level safety, and dependency risk,
- why bounded aggregation, buffer-based serialization, and multi-feature testing must be preserved,
- and which kinds of crypto evolution remain feasible versus compatibility-sensitive.

Use this file when you want engineering guidance that complements the conceptual and algorithm crypto documents while staying grounded in the actual codebase.

### 9. [MoonBlokz Radio Concept Model](./moonblokz-radio-concept.md)

A conceptual radio document grounded in the current `moonblokz-radio-lib` codebase.

This document explains:

- why MoonBlokz treats radio as a blockchain-state synchronization layer rather than a guaranteed-delivery transport,
- how local-only topology awareness, echo mapping, the connection matrix, and adaptive relaying fit together,
- why bounded queues, static memory, and deterministic degradation are core design principles,
- how the runtime pipeline supports the higher-level mesh behavior,
- and where the current implementation is narrower or more specific than earlier article-era summaries.

Use this file when you want to understand the role, design philosophy, and current conceptual boundaries of the MoonBlokz radio subsystem before reading the more formal or implementation-facing radio notes.

### 10. [MoonBlokz Radio Algorithm Model](./moonblokz-radio-algorythm.md)

A formal, algorithm-oriented radio document grounded in the current `moonblokz-radio-lib` implementation.

This document explains:

- the actual implemented message and packet model,
- the queue-backed runtime pipeline,
- the echo-mapping and connection-matrix update behavior,
- the current TX scheduling, RX handling, relay scoring, and reactive recovery rules,
- the precise wait-pool and timed-task behavior visible in code,
- and the exact boundaries of what the current codebase still does not define.

Use this file when you want the main formal description of currently implemented MoonBlokz radio behavior.

### 11. [MoonBlokz Radio Implementation Notes](./moonblokz-radio-implementation.md)

An implementation-support radio document grounded in the current `moonblokz-radio-lib` codebase.

This document explains:

- what engineering responsibilities follow from the current crate structure and public API,
- how feature flags, memory profiles, queues, and backends shape implementation behavior,
- which board-specific and backend-specific constraints are explicit today,
- why Embassy-based async execution, static allocation, and bounded queues are central to the design,
- how the current implementation differs from or sharpens earlier article-era summaries,
- and which details still remain open for later code evolution.

Use this file when you want practical engineering guidance for implementing or extending the documented MoonBlokz radio behavior without inventing details beyond the current sources.

### 12. [MoonBlokz Simulator Concept Model](./moonblokz-simulator-concept.md)

A conceptual simulator document grounded in the current `moonblokz-radio-simulator` codebase.

This document explains:

- how the application now combines simulation, live tracking, and historical replay on one desktop surface,
- why reuse of the real `moonblokz-radio-lib` remains central,
- what realities the current simulator tries to preserve,
- what deliberate simplifications still define its scope,
- and how scene files, measurements, visualization, and connection-matrix inspection fit together conceptually.

Use this file when you want to understand what the simulator currently is and what role it plays in MoonBlokz before reading the more formal or implementation-facing companion documents.

### 13. [MoonBlokz Simulator Algorithm Model](./moonblokz-simulator-algorythm.md)

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

### 14. [MoonBlokz Simulator Implementation Notes](./moonblokz-simulator-implementation.md)

An implementation-support simulator document grounded in the current `moonblokz-radio-simulator` crate.

This document explains:

- the current crate structure and module responsibilities,
- how embedded radio-library code is reused through Embassy and `'static` queue strategies,
- how simulation, analyzer, UI, and control modules are separated,
- what bounded queues, bounded histories, and time-driver invariants shape implementation behavior,
- how `egui/eframe`, `egui_extras`, and Telemetry Hub integration fit into the application,
- and which implementation limitations should remain explicit.

Use this file when you want practical engineering guidance that complements the conceptual and algorithm simulator documents.

### 15. [MoonBlokz Telemetry Concept Model](./moonblokz-telemetry-concept.md)

A conceptual telemetry document grounded in Part VII/5 and reconciled with the current telemetry repositories.

This document explains:

- why MoonBlokz field testing needs an out-of-band telemetry architecture,
- how the Test Station, Probe, Telemetry HUB, Update Server, Log Collector, CLI, and Analyzer fit together,
- why logging, command execution, and OTA updates are treated as one operational concern,
- what architectural boundaries keep telemetry from distorting LoRa measurements,
- how analyzer/live-tracking workflows fit into the telemetry architecture,
- and how the telemetry story relates to the simulator and field-analysis workflows.

Use this file when you want to understand why the MoonBlokz telemetry system exists and how its main parts fit together before reading the more formal or implementation-facing telemetry notes.

### 16. [MoonBlokz Telemetry Algorithm Model](./moonblokz-telemetry-algorythm.md)

A formal, algorithm-oriented telemetry document grounded primarily in the reviewed telemetry repositories and aligned with Part VII/5 where still relevant.

This document explains:

- the end-to-end flow of logs, commands, polling intervals, and ordered downloads,
- the Telemetry HUB’s persistence and serving rules,
- how CLI commands are addressed and routed,
- how Probe and Node OTA flows differ formally,
- and how analyzer-side live tracking, playback, and connection-matrix reconstruction depend on the telemetry pipeline.

Use this file when you want the main formal description of the telemetry architecture described in the field-testing article.

### 17. [MoonBlokz Telemetry Implementation Notes](./moonblokz-telemetry-implementation.md)

An implementation-support telemetry document grounded in the current telemetry repositories and used to capture deployment, configuration, repository structure, and engineering cautions.

This document explains:

- what engineering responsibilities are assigned to the Probe, HUB, Update Server, Collector, CLI, and analyzer-integration layers,
- why the node–Probe USB boundary is operationally important,
- what safety and failure-mode concerns matter for OTA updates,
- how TM-tagged logs shape compatibility between deployed systems and analysis tools,
- and which implementation boundaries should remain explicit as the telemetry stack evolves.

Use this file when you want implementation guidance that complements the conceptual and algorithm telemetry documents without scattering those details across the simulator files.

### 18. [MoonBlokz Storage Concept Model](./moonblokz-storage-concept.md)

A conceptual storage document based on Part VIII of the MoonBlokz series.

This document explains:

- why onboard storage becomes a first-class subsystem in MoonBlokz,
- why microcontroller flash cannot be treated like an ordinary filesystem,
- how `snake_chain` makes bounded blockchain storage possible,
- what data must survive persistently on the node,
- why block data and control data are separated,
- and why redundancy, integrity checking, and crash tolerance shape the storage design.

Use this file when you want to understand the role, design philosophy, and conceptual trade-offs of MoonBlokz onboard storage before reading the more formal or implementation-facing storage notes.

### 19. [MoonBlokz Storage Algorithm Model](./moonblokz-storage-algorythm.md)

A formal, algorithm-oriented storage document based on Part VIII of the MoonBlokz series.

This document explains:

- the formal persisted-data categories,
- the storage-unit model built around flash erase regions,
- the block-capacity sizing formula,
- the stored-hash and CRC-based integrity rules,
- the redundancy and fallback behavior for control data,
- the crash-recovery rules for invalid or interrupted writes,
- and the wear-lifetime estimation logic used to justify the design.

Use this file when you want the main formal description of the MoonBlokz storage model described in the onboard-storage article.

### 20. [MoonBlokz Storage Implementation Notes](./moonblokz-storage-implementation.md)

An implementation-support storage document based on Part VIII of the MoonBlokz series.

This document explains:

- what engineering constraints follow from RP2040 flash behavior,
- how XIP affects persistent-write design,
- why Embassy’s flash API shape matters,
- what practical consequences follow from sector-aligned storage units,
- how integrity checking, redundancy, and wear distribution affect engineering choices,
- and which implementation details remain explicitly open in the source material.

Use this file when you want implementation guidance that complements the conceptual and algorithm storage documents without inventing details beyond the article.

### 21. [MoonBlokz Radio Documentation Discrepancies](./moonblokz-radio-doc-discrepancies.md)

A discrepancy summary document comparing the earlier VII/1–VII/3 radio article framing with the current `moonblokz-radio-lib` implementation.

This document explains:

- where the current code is narrower than the earlier conceptual radio descriptions,
- which radio ideas were confirmed and made more concrete by implementation,
- how the current runtime, relaying, recovery, and backend model sharpen earlier summaries,
- which article-era features should now be read as future extensions rather than present behavior,
- and how future documentation should distinguish current implementation from long-term intent.

Use this file when you want a quick reconciliation between the historical radio design narrative and the current codebase-grounded documentation.

## Suggested Reading Order

1. Start with [MoonBlokz Overview](./moonblokz-overview.md) to understand the mission, use cases, and high-level constraints.
2. Continue with [MoonBlokz Technology and Architecture](./moonblokz-technology.md) to understand the implementation strategy and architectural foundation.
3. Then read [MoonBlokz Blockchain Concept Model](./moonblokz-blockchain-concept.md) to understand the operating idea of MoonBlokz blockchain behavior.
4. Continue with [MoonBlokz Blockchain Algorithm Model](./moonblokz-blockchain-algorythm.md) for the full Parts III, IV, and V algorithmic flow and the detailed main data structures.
5. Then read [MoonBlokz Blockchain Implementation Notes](./moonblokz-blockchain-implementation.md) for implementation-facing constraints, configuration boundaries, and cautions.
6. Continue with [MoonBlokz Crypto Concept Model](./moonblokz-crypto-concept.md) to understand the role, trade-offs, and constraints of MoonBlokz cryptography.
7. Then read [MoonBlokz Crypto Algorithm Model](./moonblokz-crypto-algorythm.md) for the formal crypto structure, signature roles, deterministic signing and aggregation behavior, and explicit size and count limits.
8. Continue with [MoonBlokz Crypto Implementation Notes](./moonblokz-crypto-implementation.md) for implementation-facing cryptography guidance, dependency cautions, and testing implications.
9. Then read [MoonBlokz Radio Concept Model](./moonblokz-radio-concept.md) to understand why MoonBlokz radio behaves as a synchronization-first, bounded embedded subsystem.
10. Continue with [MoonBlokz Radio Algorithm Model](./moonblokz-radio-algorythm.md) for the current code-grounded runtime flow of messages, packets, scheduling, relaying, and recovery.
11. Continue with [MoonBlokz Radio Implementation Notes](./moonblokz-radio-implementation.md) for the actual API, feature model, backend behavior, and implementation-facing cautions.
12. Then read [MoonBlokz Simulator Concept Model](./moonblokz-simulator-concept.md) to understand how MoonBlokz currently validates, observes, replays, and controls radio-network behavior through the desktop simulator application.
13. Continue with [MoonBlokz Simulator Algorithm Model](./moonblokz-simulator-algorythm.md) for the formal simulation, analyzer, timing, and control behavior.
14. Continue with [MoonBlokz Simulator Implementation Notes](./moonblokz-simulator-implementation.md) for the current crate structure, queue/lifetime strategy, UI modules, analyzer pipeline, and implementation cautions.
15. Then read [MoonBlokz Telemetry Concept Model](./moonblokz-telemetry-concept.md) to understand why field testing needs a separate telemetry architecture and how the operational components fit together.
16. Continue with [MoonBlokz Telemetry Algorithm Model](./moonblokz-telemetry-algorythm.md) for the formal flow of logs, commands, polling control, OTA, and analyzer interaction.
17. Continue with [MoonBlokz Telemetry Implementation Notes](./moonblokz-telemetry-implementation.md) for repository-level responsibilities, update-path cautions, and telemetry-specific engineering constraints.
18. Then read [MoonBlokz Storage Concept Model](./moonblokz-storage-concept.md) to understand why bounded blockchain persistence on flash is possible at all and how MoonBlokz separates recoverable block data from recovery-critical control data.
19. Continue with [MoonBlokz Storage Algorithm Model](./moonblokz-storage-algorythm.md) for the formal storage-unit layout, integrity-check rules, redundancy behavior, crash recovery, and wear-lifetime model.
20. Continue with [MoonBlokz Storage Implementation Notes](./moonblokz-storage-implementation.md) for RP2040/XIP/Embassy engineering consequences, capacity-planning cautions, and explicitly open storage design questions.
21. Use [MoonBlokz Radio Documentation Discrepancies](./moonblokz-radio-doc-discrepancies.md) when you need to compare the earlier VII/1–VII/3 article narrative with the current code-grounded radio documents.

## Review Notes

Post-change review against `moonblokz-info` documentation rules:

- **Consistency:** The index now includes a dedicated three-file storage document set in addition to the existing blockchain, crypto, radio, simulator, and telemetry groups.
- **Logical soundness:** The file presents a staged path from overall MoonBlokz context through blockchain, crypto, radio, simulator, telemetry, and finally onboard storage, which depends on the earlier `snake_chain` and embedded-platform constraints.
- **Feasibility:** The updated structure remains simple and maintainable while giving future readers a clear path from protocol behavior into persistent-storage design.
- **Redundancy:** Each large topic keeps the same concept / algorythm / implementation split, reducing overlap and making navigation more predictable.
- **Source fidelity:** The new storage entries are explicitly scoped to Part VIII and do not claim broader implementation detail than that source provides.
