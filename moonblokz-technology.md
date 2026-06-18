# MoonBlokz Technology and Architecture

## Purpose of This Document

This document provides a project-level summary of MoonBlokz technology and architecture across the current knowledge base.

Its role is to explain how MoonBlokz is built as a whole:

- which technology choices define the project,
- which architectural boundaries are stable across subsystems,
- how the major repositories and runtime components fit together,
- and which constraints must remain visible when planning or implementing changes.

This document is intentionally broader than the original Part II technology article. It keeps the Part II architectural foundations, but it is updated to reflect the current blockchain, crypto, radio, simulator, telemetry, and storage documents.

## Source Basis

This document is based on the following knowledge-base sources:

- [`moonblokz-overview.md`](./moonblokz-overview.md)
- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md)
- [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md)
- [`moonblokz-blockchain-implementation.md`](./moonblokz-blockchain-implementation.md)
- [`moonblokz-crypto-concept.md`](./moonblokz-crypto-concept.md)
- [`moonblokz-crypto-algorythm.md`](./moonblokz-crypto-algorythm.md)
- [`moonblokz-crypto-implementation.md`](./moonblokz-crypto-implementation.md)
- [`moonblokz-radio-concept.md`](./moonblokz-radio-concept.md)
- [`moonblokz-radio-algorythm.md`](./moonblokz-radio-algorythm.md)
- [`moonblokz-radio-implementation.md`](./moonblokz-radio-implementation.md)
- [`moonblokz-simulator-concept.md`](./moonblokz-simulator-concept.md)
- [`moonblokz-simulator-algorythm.md`](./moonblokz-simulator-algorythm.md)
- [`moonblokz-simulator-implementation.md`](./moonblokz-simulator-implementation.md)
- [`moonblokz-telemetry-concept.md`](./moonblokz-telemetry-concept.md)
- [`moonblokz-telemetry-algorythm.md`](./moonblokz-telemetry-algorythm.md)
- [`moonblokz-telemetry-implementation.md`](./moonblokz-telemetry-implementation.md)
- [`moonblokz-storage-concept.md`](./moonblokz-storage-concept.md)
- [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md)
- [`moonblokz-storage-implementation.md`](./moonblokz-storage-implementation.md)

## Project-Level Technology Framing

MoonBlokz is a hyper-local DePIN blockchain for constrained devices that communicate over unreliable low-bandwidth radio instead of depending on globally available internet infrastructure.

At a technology level, MoonBlokz is not one single executable or one isolated repository. It is a coordinated system made of several focused Rust codebases and companion tools that together support:

- blockchain state propagation over radio,
- bounded on-device persistence,
- cryptographic signing and aggregation,
- simulation and replay-based analysis,
- and out-of-band telemetry, command, and update workflows.

The overall architecture is shaped by a small set of non-negotiable environmental assumptions:

- nodes may have very limited RAM and flash,
- radio links are weak, local, delayed, and lossy,
- global time cannot be assumed,
- some hardware lacks true random generation or crypto acceleration,
- and field testing needs observability that must not distort the LoRa mesh itself.

These assumptions are not local implementation details. They define the architecture of the whole project. This section, together with the architectural invariants below, is their canonical home: other documents apply these assumptions in their own context and reference these sections rather than restating them as independent facts.

## Core Technology Direction

## Rust as the Unifying Implementation Language

MoonBlokz is built around Rust because the project needs one technology stack that can span:

- embedded targets,
- desktop tools,
- command-line utilities,
- and server-side telemetry components.

Rust is therefore not only a language preference. It is the main portability mechanism that lets MoonBlokz keep shared engineering practices across very different runtime environments while still targeting constrained hardware.

Across the current knowledge base, this decision is reinforced by recurring implementation patterns:

- explicit type-driven APIs,
- compile-time feature selection,
- bounded memory strategies,
- and careful separation between portable logic and environment-specific adapters.

## `no_std` Orientation and Embedded Discipline

The original MoonBlokz architecture treated the portable core as `no_std`-friendly, and the current subsystem documents still reflect that embedded-first discipline.

This does not mean every repository is `no_std`. Some tools are desktop or server applications and depend on `std`. Instead, it means the project consistently favors designs that are compatible with constrained environments whenever that is practical:

- narrow public contracts,
- static or bounded allocation strategies in hot paths,
- explicit queue sizing,
- binary-oriented serialization,
- and cooperative processing instead of hidden autonomous runtime behavior.

## Compile-Time Replaceability

A strong cross-cutting pattern in the current codebase is compile-time backend selection.

This appears in multiple subsystems:

- crypto chooses one algorithm family and one concrete backend at build time,
- radio chooses one device backend and one memory profile at build time,
- storage chooses exactly one backend at build time,
- and host versus embedded execution differences are made explicit rather than hidden.

Architecturally, this means MoonBlokz prefers stable public interfaces with replaceable implementations behind them, instead of one universal runtime abstraction layer with dynamic dispatch everywhere.

This build-time replaceability should not be confused with node-level operational complexity. In the current MoonBlokz direction, backend and memory-profile selection are primarily product-build and deployment-image concerns. Once those choices are fixed for a given build, the intended node-level configuration surface can remain very small.

## The Main Architectural Invariants

The current knowledge base establishes several project-wide invariants that should guide future work. Together with the non-negotiable environmental assumptions above, they are stated canonically here; subsystem documents show how each invariant applies in their area and reference this section rather than restating it as an independent rule.

### 1. Best-effort operation is normal

MoonBlokz is designed for unreliable communication. The radio layer does not provide guaranteed delivery, and the blockchain layer does not assume strict global ordering.

Instead, the system uses:

- best-effort propagation,
- reactive recovery of missing information,
- staged validation,
- and eventual convergence where conditions permit.

This principle affects the entire architecture, not just the radio code.

### 2. Bounded resources are part of correctness

MoonBlokz treats limits on airtime, RAM, flash, queue depth, and payload size as architectural constraints.

The project therefore consistently favors:

- bounded queues,
- bounded signature aggregation,
- bounded storage windows,
- bounded telemetry buffers,
- and deterministic overload behavior.

In MoonBlokz, “resource bounds” are not merely optimization concerns. They are part of the design contract.

### 3. Data size is a protocol concern

The knowledge base repeatedly shows that serialization size is tied to radio fragmentation, flash layout, replay cost, and fee pressure.

As a result:

- binary representation matters,
- compact encodings matter,
- large objects must be fragmented or compressed carefully,
- and richer payloads are never free from a protocol perspective.

This is one of the clearest project-wide connections between blockchain, crypto, radio, and storage.

### 4. Global time is intentionally avoided

MoonBlokz assumes local elapsed-time measurement but not reliable shared wall-clock time.

That affects multiple layers:

- blockchain uniqueness and sequencing cannot depend on synchronized timestamps,
- radio behavior is driven by local timers and pacing,
- simulator time is synthetic and explicitly controlled,
- and telemetry coordination uses polling intervals and server policy rather than a global time model shared with the mesh.

### 5. Control and observability stay explicit

MoonBlokz prefers explicit control flow over hidden background behavior.

This appears in:

- host-owned event loops and processing steps,
- explicit radio pipeline tasks,
- explicit application feedback to the radio layer,
- explicit analyzer replay behavior,
- and explicit command / update paths in the telemetry stack.

This design style improves predictability on constrained systems and makes field behavior easier to inspect.

### 6. Telemetry and control are out-of-band

The telemetry documents make clear that logging, commands, and OTA updates are operationally important but must not distort the LoRa mesh.

For that reason, MoonBlokz keeps these concerns outside the radio blockchain network through the Probe, HUB, Collector, CLI, and analyzer integrations.

This is now a core architectural boundary, not an implementation accident.

## System Architecture Overview

At a high level, MoonBlokz is composed of six tightly related technical areas:

1. **Blockchain** — defines bounded-chain behavior, state reconstruction, approvals, balances, UTXOs, and active-state preservation.
2. **Crypto** — provides signatures, aggregation-ready signatures, and aggregated evidence through replaceable backends.
3. **Radio** — provides the bounded message-propagation substrate used to synchronize blockchain-related state over LoRa-class links.
4. **Storage** — provides compact indexed persistence contracts for control-plane data and canonical block bytes.
5. **Telemetry** — provides out-of-band logs, command routing, OTA flows, and analysis data collection.
6. **Simulator / Analyzer** — provides a desktop environment for simulation, live tracking, replay, visualization, and control integration.

These are separate subsystems, but they are intentionally interdependent.

## Subsystem Roles and Architectural Boundaries

### Blockchain

The blockchain subsystem is not modeled as an infinite globally ordered chain. It is modeled as a block-tree operating under weak connectivity and bounded retention.

At current design level, `moonblokz-blockchain` is also best understood as a stateful semantic event state machine with a deliberately narrow responsibility boundary.

Its architectural role is to:

- reconstruct and maintain useful blockchain state under unreliable propagation,
- tolerate missing parents and delayed validation,
- preserve the active economic state through the `snake_chain` retention model,
- and keep balances, configuration, approval evidence, and live UTXOs reconstructable near the active head.

The current boundary refinement also treats communication transport, storage mechanics, crypto backend details, and radio-derived creator-scoring logic as external dependencies rather than internal blockchain responsibilities.

Important project-level implications from the blockchain documents include:

- startup and recovery are first-class states,
- dominant-chain acquisition is a practical objective before full reconstruction,
- branch switching and replay are normal protocol work,
- and chain compactness is tied directly to radio and storage feasibility.

The storage subsystem supports blockchain persistence, but it does **not** own blockchain policy. Retention rules, active-state replay, and branch semantics belong above the storage contract.

### Crypto

The crypto subsystem is designed as a replaceable crate with a stable public role model rather than one hard-coded cryptographic implementation.

Its architectural role is to provide:

- direct signatures,
- aggregation-ready signatures,
- and aggregated evidence structures

within explicit embedded limits.

The current documents show that MoonBlokz uses:

- compile-time backend selection,
- Schnorr as the practical default family,
- BLS as an important alternative family,
- deterministic signing behavior where appropriate,
- and a hard bound on aggregated evidence size.

This means crypto is not a purely isolated utility layer. Its serialization size, verification behavior, and aggregation strategy affect blockchain payload design and radio/storage capacity planning.

### Radio

The radio subsystem is the synchronization fabric of MoonBlokz. It does not behave like a reliable transport stack.

Its architectural role is to provide:

- bounded message transmission and reception,
- local topology awareness through echo mapping,
- stateful relaying through the connection matrix and wait pool,
- packet fragmentation and reassembly for large messages,
- and reactive self-healing through missing-part and missing-block requests.

At project level, the connection matrix should be understood as bounded local neighbor knowledge rather than a global network map. This means MoonBlokz can still target much larger multi-hop networks than any one node can track directly, because end-to-end reachability depends on relaying across local neighborhoods rather than on globally complete topology state.

The current radio architecture is explicitly organized around a three-task runtime pipeline:

- TX Scheduler,
- Radio Device,
- RX Handler.

That pipeline, together with the Relay Manager and wait-pool logic, makes radio behavior both bounded and stateful.

The radio subsystem also depends on the application layer for some validation and duplicate handling decisions. This explicit application/radio contract is an important architectural boundary.

### Storage

The storage subsystem is intentionally narrow. It is not a full blockchain database.

Its architectural role is to provide:

- a compact persistence contract for canonical block bytes,
- replicated control-plane persistence for recovery-critical metadata,
- backend-specific integrity handling,
- and a backend split suitable for both host/testing and RP2040-class deployment.

The current documents establish two conceptual planes:

- a **control plane** for critical small metadata such as chain configuration,
- and a **block plane** for indexed block persistence.

The current storage design also shows that compatibility checks and replica repair are part of control-plane recovery. This is more precise than the earlier high-level “storage abstraction” framing from Part II.

### Telemetry

Telemetry is the operational support system for real deployments and test environments.

Its architectural role is to provide:

- out-of-band log capture,
- command routing,
- node and Probe update workflows,
- centralized policy such as polling intervals,
- delayed and ordered download for collectors,
- and data paths for analyzer and live-tracking tools.

The telemetry architecture is split across three environments:

- the embedded or test-station environment,
- the cloud coordination environment,
- and the local analysis / operations environment.

Within that model, the Probe is the bridge between node-local USB/log/update behavior and the HUB-based coordination layer.

This subsystem is operationally essential, but it is intentionally kept outside the LoRa blockchain/radio path.

### Simulator and Analyzer

The simulator is not only a development toy. It is a core architectural support tool that reuses the real radio library and connects simulation, observation, and analysis.

Its architectural role is to provide:

- simulation mode for radio-network experiments,
- real-time tracking of deployed systems through telemetry-fed logs,
- historical log replay and visualization,
- scene-based spatial context,
- connection-matrix inspection,
- and a desktop UI surface for understanding network behavior.

The simulator therefore acts as a bridge between code-grounded radio behavior and field-observation workflows.

It also makes clear that MoonBlokz values architecture-level testability and observability, not only deployable node runtime behavior.

## Cross-Subsystem Relationships

The reviewed documents show several important subsystem relationships.

### Blockchain, radio, and storage are tightly coupled by size and recovery cost

Blockchain objects are larger than many radio payloads, so fragmentation is unavoidable for some message types. That in turn makes missing-part recovery and replay scheduling part of the real protocol behavior.

At the same time, bounded retention means storage cannot simply keep unlimited history. The blockchain therefore depends on replayable active-state preservation near the head, while storage provides only the underlying persistence contract.

### Crypto choices affect protocol capacity

Signature and aggregated-evidence formats influence:

- blockchain payload size,
- radio fragmentation pressure,
- serialization contracts,
- and verification cost.

For MoonBlokz, cryptography is therefore partly a capacity-planning concern.

### Telemetry and simulator form the observability layer around the mesh

The simulator/analyzer depends on telemetry logs and HUB-mediated control paths for live tracking and replay.

This creates a deliberate architectural separation:

- the mesh carries blockchain-related messages,
- while telemetry carries operational visibility and control.

### Host tools and embedded nodes share concepts but not identical runtime models

MoonBlokz reuses logic and semantics across embedded devices, desktop tools, and backend services, but the documents make clear that these are not collapsed into one runtime model.

Examples include:

- radio host versus embedded initialization differences,
- memory versus RP2040 storage backends,
- simulator time-driver behavior versus embedded local timers,
- and telemetry server behavior versus field-node operation.

This is a portability architecture built on shared contracts, not on pretending all targets are the same.

## Current Implementation-Grounded Technology Picture

The present knowledge base sharpens the original Part II architectural framing in several important ways.

### The project is now a multi-repository Rust system

MoonBlokz currently spans several focused repositories and crates rather than one single portable library with a small set of helpers.

At the knowledge-base level, the main architectural areas include:

- blockchain logic and its data/state model,
- `moonblokz-crypto-lib`,
- `moonblokz-radio-lib`,
- `moonblokz-storage`,
- `moonblokz-radio-simulator`,
- and the telemetry-side repositories such as Probe, HUB, Collector, CLI, and Update Server.

This is consistent with the original portable-core idea, but more operationally mature.

### The radio layer is more concrete than the Part II abstraction

Part II described a minimal radio integration boundary. The current radio documents show a much more specific design with:

- message and packet layers,
- queue-backed async tasks,
- connection matrices,
- relay scoring,
- duplicate caches,
- CRC handling,
- and explicit fragmentation/recovery rules.

The architecture is still modular, but the current project has moved far beyond a purely abstract “send / receive / process” radio model.

### Storage is narrower than a generic persistence abstraction

Part II emphasized high-level storage operations instead of low-level flash APIs. The current storage crate confirms that direction, but in a more constrained form than a generic “save and query blockchain data” interpretation might suggest.

The present storage contract is:

- synchronous,
- small in public surface,
- block-index oriented,
- and explicit about backend-specific recovery/integrity differences.

This means the storage layer is deliberately minimal and should not silently accumulate blockchain-policy responsibilities.

### Telemetry is a full operational subsystem

Part II did not yet define the later operational stack. The current documents now establish that MoonBlokz requires a distinct telemetry architecture with:

- a Linux-based Probe daemon,
- a Spin/WASI HUB with SQLite and key-value state,
- thin CLI and Collector clients,
- static update artifact hosting,
- and analyzer-facing log consumption.

This is now a central part of how MoonBlokz is developed, tested, observed, and updated.

### The simulator is a first-class architecture tool

The current project includes a desktop simulator/analyzer that reuses real radio logic, visualizes connection state, and supports both synthetic scenarios and live telemetry-fed analysis.

That makes simulation and replay part of the project architecture, not merely auxiliary documentation support.

## What Must Remain Explicit

The current knowledge base also defines several boundaries that should remain visible.

### MoonBlokz does not promise perfect reliability

The project is built around loss tolerance, reactive recovery, and eventual convergence where possible. Documentation should not drift toward internet-style reliable-delivery assumptions.

### The radio layer does not provide a full security transport stack

The current radio documents explicitly note the absence of a built-in network-layer security system. Security-sensitive interpretations must therefore stay aligned with the actual current scope.

### Storage is not the owner of active-chain policy

The storage crate persists blocks and control-plane data, but it does not fully define branch retention, UTXO replay policy, or `snake_chain` scheduling decisions.

### Telemetry policy is intentionally simple today

The HUB currently acts as a central policy point for intervals and command delivery, while Collector and CLI remain thin and intentionally shallow. Future richer policy or persistence models would be architectural changes, not small implementation tweaks.

### The simulator remains approximate in physical modeling

The simulator preserves important timing and topology realities, but it is still an approximation. Path loss, collision, capture, and obstacle models should not be described as a perfect RF truth model.

### Some strategic questions remain open

The knowledge base still leaves some areas intentionally unresolved or only partially defined, including:

- some detailed blockchain-policy choices,
- some communication and approval-format details,
- long-term configuration mutability behavior,
- future telemetry policy richness,
- and future cryptographic evolution such as any post-quantum path.

These should remain explicit open boundaries rather than being filled with unsupported assumptions.

## Business Analyst View: Why This Technology Shape Exists

From a business and delivery perspective, MoonBlokz technology is designed to make a local economic system feasible under severe infrastructure constraints.

The architecture supports that goal by combining:

- a blockchain model that tolerates weak communication,
- a radio model that works under bounded embedded conditions,
- a storage model that fits limited flash,
- a crypto model that balances deployability with aggregation needs,
- and an operational telemetry layer that makes field use and testing practical.

In other words, the technology stack exists to make the project viable where mainstream internet-dependent designs would fail.

## Architect View: The Most Important Structural Reading

The most important architectural interpretation of MoonBlokz today is this:

MoonBlokz is a bounded, portable, multi-subsystem Rust platform for local blockchain coordination over weak radio, surrounded by explicit observability and operational tooling.

Its architecture is successful only if it preserves all of the following at once:

- portable contracts,
- bounded runtime behavior,
- explicit control flow,
- out-of-band operations support,
- and fidelity to the constraints of low-bandwidth local networks.

Any future change that weakens those properties should be treated as a significant architectural decision.


