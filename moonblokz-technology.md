# MoonBlokz Technology and Architecture

## Purpose of This Document

This document summarizes the technology selection and the high-level architecture introduced in Part II of the MoonBlokz series. Its goal is to capture the project's technical foundation in a form that is useful for planning, implementation, documentation, and future architectural review.

The document focuses on what the article explicitly establishes and extends it with carefully bounded architectural interpretation where that helps clarify implementation consequences.

## Source Basis

This document is based on:

- **MoonBlokz series part II. — Technologies & Architecture** by Peter Sallai, published on Medium on February 25, 2025.

## Project Goal and Technical Framing

MoonBlokz is a hyper-local DePIN blockchain designed for constrained devices that communicate over radio instead of depending on continuous, high-bandwidth internet connectivity. Part II does not redefine the strategic vision from Part I; instead, it turns that vision into an initial technology and architecture direction.

At this stage, the article answers a practical question:

> Given the constraints of microcontrollers, weak radio communication, and the need to run both on embedded devices and regular computers, what technology stack and architectural shape make MoonBlokz implementable?

The answer in Part II is built around three framing decisions:

- the implementation language should work well on both embedded targets and computers,
- the reusable blockchain logic should live in a portable library,
- and hardware- or platform-specific behavior should be abstracted behind integration modules provided by the embedding application.

This makes Part II the first concrete architectural foundation of the project.

## Business Analyst View: What Problem the Technology Must Solve

From a business and requirements perspective, the technology choices in Part II are not independent engineering preferences. They are direct responses to the operating conditions defined for MoonBlokz.

The project needs a technical foundation that can support:

- deployment on small, resource-constrained devices,
- operation in environments with unreliable or low-bandwidth communication,
- portability across multiple hardware setups,
- extension toward a broader local economic platform,
- and a development model that reduces the probability of subtle implementation bugs in a safety-critical distributed system.

Seen from that perspective, the article establishes that MoonBlokz is not simply a blockchain implementation project. It is a portability and systems-integration project that must preserve the same conceptual behavior across very different runtime environments.

That requirement strongly influences the architecture:

- core blockchain logic must be reusable,
- hardware dependencies must be isolated,
- and APIs must be designed around capabilities rather than specific devices.

## Why Rust Was Chosen

Part II evaluates the implementation language primarily through the lens of embedded viability and software quality.

The article reviews several options:

- **C/C++** offer mature embedded support, broad library availability, and strong performance, but provide less protection against common bugs and weak design patterns.
- **MicroPython** works on many microcontrollers, but is not positioned as the best choice when the system needs to extract maximum hardware performance.
- **Other languages** are described as either unsuitable for effective embedded development or unsuitable as a single language spanning both microcontrollers and computers.
- **Rust** is selected because it combines near-C/C++ performance with stronger support for safer and more idiomatic software design.

The article frames Rust as the best balance between low-level control and high-level language support. This is important for MoonBlokz because the project combines several difficult concerns at once:

- embedded execution,
- protocol correctness,
- cryptographic operations,
- radio communication,
- and cross-platform portability.

The explicit reason for choosing Rust is therefore not novelty. It is risk reduction without giving up performance.

## Consequences of the Rust Decision

The language choice immediately constrains the project in ways the article makes explicit.

### Supported prototype hardware is limited to Rust-capable embedded targets

Because MoonBlokz is implemented in Rust, the first prototype must use a microcontroller platform with sufficiently mature Rust support. The article selects RP2040-based devices as the initial direction and names concrete examples with integrated or attachable LoRa hardware.

This is a practical scoping decision:

- it narrows the hardware search space,
- it allows the project to move forward with real devices,
- and it keeps the prototype grounded in available toolchains.

### Initial distribution model is source-code based

The article states that MoonBlokz can only be distributed as source code at this stage. This reflects both the realities of Rust-based embedded development and common embedded distribution patterns.

This has several downstream implications:

- adopters need a build environment,
- hardware-specific builds remain the responsibility of the integrator or deployer,
- and release management is likely to be more artifact-fragmented than for a single packaged desktop or server application.

The article also notes that source distribution does **not** automatically imply open source. That decision remains open.

## High-Level Architectural Model

Part II introduces MoonBlokz as a reusable Rust library that can be embedded into different applications. The planned library name is `moon_blokz_lib`.

This library-centric approach is a major architectural decision. Instead of treating MoonBlokz as a single monolithic executable, the article defines it as a portable core that can be integrated into multiple runtime environments.

The library is planned as a `no_std` library.

This matters for two reasons:

- a `no_std` library can run in embedded environments without relying on the standard library,
- and the same library can still be used from standard applications on computers.

This design directly supports one of the project's stated needs: portability across microcontrollers and computer-based tooling or companion applications.

## Core Library Modules

The article identifies four main modules inside the library.

### BlockChain

The `BlockChain` module is responsible for blockchain management and related algorithms.

This is the domain core of the system. It is where chain handling logic belongs, including the rules and structures needed to maintain blockchain state.

### MoonBlokzNetwork

The `MoonBlokzNetwork` module is described as the central facade of the library.

Its role is to:

- expose a local API to the embedding program,
- connect the other internal modules,
- and present MoonBlokz functionality as a coherent subsystem rather than a loose set of primitives.

Architecturally, this suggests that the library should be consumed through a clear orchestration boundary instead of requiring external code to coordinate low-level internals manually.

### Communication

The `Communication` module contains the implementation-independent parts of radio communication, such as relaying algorithms.

This separation is important. It means the project distinguishes between:

- communication logic that belongs to MoonBlokz as a protocol system,
- and hardware-dependent radio mechanics that belong to the embedding environment.

That boundary is central to long-term portability.

### Utils

The `Utils` module contains utility functions shared by other parts of the library.

The article does not expand on its contents, so this should be treated as a support module rather than a place to infer major domain behavior.

## Required External Integration Modules

The library is not meant to operate in isolation. The embedding program must provide several external modules or services so the library can function on a concrete platform.

This is one of the most important architectural ideas in Part II: MoonBlokz keeps its portable logic in the library and pushes environment-specific concerns to integration boundaries.

### Storage

Storage is treated as a special case because the article argues that a very low-level universal storage API would not fit the real diversity of embedded persistence models.

Two example storage models are described:

- **memory-mapped plus page-write flash access**, such as on RP2040-style platforms,
- **file-system-based storage**, where data is managed as files and directories.

The article explains that these two models differ too much to share a single efficient low-level abstraction. As a result, the storage API should be **higher level**, exposing operations such as saving and querying blockchain blocks rather than raw storage primitives.

The article's example is especially important:

- blocks are binary payloads of roughly 2500 bytes,
- they are queried by a sequence identifier and a 32-byte hash,
- and the internal storage strategy may be radically different depending on whether a file system exists.

Architecturally, this means:

- storage abstraction is driven by **data access intent**, not hardware mechanics,
- storage backends may legitimately use very different internal indexing strategies,
- and the library should encapsulate the query/store semantics instead of leaking backend assumptions upward.

The article states that two distinct storage implementations will be provided within the library, both built on lower-level APIs supplied by the embedding program.

### Random

Random number generation is needed for several domains:

- cryptography,
- relay decisions,
- and selecting transactions to drop when the mempool is full.

The article distinguishes between two deployment cases:

- if the hardware provides a true random number generator, the project can use it broadly,
- if not, pseudo-random generation is acceptable for most algorithms except initial key generation.

In the second case, the key pair must be generated externally and copied to the node.

This is a meaningful system constraint because it avoids pretending all embedded hardware has the same cryptographic readiness. It also separates:

- what the node must be able to do locally at runtime,
- from what can be provisioned offline during deployment.

The described API is intentionally simple and provides random values of different types.

### Crypto

The `Crypto` integration boundary exists because cryptographic functions can be implemented in software but may also be accelerated by hardware on some chipsets.

The article therefore sets a flexible policy:

- the library includes a standard implementation,
- but the architecture should also allow hardware-accelerated crypto when available.

This is a good example of MoonBlokz's portability model. Functional behavior should remain stable while implementation details can vary based on platform capability.

### Clock

The `Clock` module is responsible for elapsed-time measurement rather than for globally synchronized wall-clock time.

This is fully aligned with the project's earlier assumption that nodes do not rely on shared global time. The article says the API should simply provide time in milliseconds.

However, it also highlights one non-trivial requirement: time must continue to increase monotonically even across reboot.

This is architecturally significant because it means the clock abstraction is not just a thin wrapper over a hardware timer. It may require additional persistence or reconstruction logic to preserve monotonic behavior after restart.

The article does not define that solution yet, so this remains an identified requirement rather than a resolved design.

### Radio

The `Radio` module is responsible for hardware-dependent communication behavior.

Its API is intentionally minimal:

- `send_message` adds a message to the output buffer,
- `get_received_message` retrieves a message from the input buffer in a non-blocking way,
- `process` lets the module maintain its input and output buffers as part of the main loop.

This design has several implications:

- radio I/O is expected to be buffer-oriented,
- library integration should avoid blocking behavior,
- and the runtime model is based on explicit periodic processing rather than hidden background execution.

The article also says that this module is responsible for formatting messages to fit radio frames, which means the interface boundary includes not only transport I/O but also frame-shaping concerns tied to the specific radio medium.

## Main Loop and Control Model

Part II explicitly addresses event-loop ownership.

The article says MoonBlokz state changes are driven by two event classes:

- arrival of incoming radio messages,
- and direct calls from the embedding program, such as creating a new transaction.

Two architectural options are considered:

1. the library owns the event loop and calls user-registered callbacks,
2. the embedding program owns the event loop and calls into the library during each iteration.

The article chooses the second option.

This is an important control-plane decision because it keeps orchestration responsibility in the host application. That approach:

- simplifies the library's responsibilities,
- gives the embedding application more control,
- fits embedded main-loop patterns well,
- and avoids forcing one runtime model on every target environment.

From an architecture standpoint, MoonBlokz is therefore designed as a cooperative subsystem, not as an autonomous runtime container.

## Architect View: Key Design Implications

Viewed as an architecture document, Part II establishes several principles that should shape implementation decisions in the rest of the project.

### 1. Core logic and platform adaptation are deliberately separated

The architecture is built around a portable core library plus environment-specific adapters. This separation is essential if the same blockchain logic must run on both microcontrollers and computers.

### 2. Abstractions are capability-driven, not hardware-driven

The storage example makes this explicit. MoonBlokz should ask for operations such as storing or querying a block, not for raw flash primitives. This allows radically different persistence strategies without changing the core logic.

### 3. The architecture favors explicit control over hidden runtime behavior

The chosen event-loop model, non-blocking receive path, and explicit `process` step all point toward a design style where scheduling and control remain visible. This is especially valuable on constrained devices.

### 4. Hardware acceleration is optional, not foundational

Crypto acceleration and true random generation can improve implementation quality on some targets, but the system is not defined in a way that depends on them universally. This preserves portability.

### 5. Some critical design questions are intentionally deferred

The article deliberately stops at the high-level architectural level. It does not yet define:

- blockchain algorithms,
- detailed data structures,
- exact storage schemas,
- monotonic-time recovery across reboot,
- or precise radio-frame handling rules.

That is a strength, not a gap, as long as later documents preserve the boundaries introduced here.

## Technical Writer View: Recommended Knowledge-Base Interpretation

For knowledge-base purposes, Part II should be treated as the **technology selection and integration-boundary document** of the MoonBlokz series.

Part I explains *why* MoonBlokz exists and what constraints define the problem space. Part II explains *what technical foundation* the project chooses in response.

The most important documentation takeaways are:

- MoonBlokz is centered on a portable Rust core.
- The core is intended to be `no_std`.
- The project cleanly separates domain logic from hardware integration.
- Storage is abstracted at a higher level because backend models vary too widely.
- The host application owns the main loop.
- Some requirements are identified early even when implementation details are deferred.

This framing is useful because it helps later documents stay consistent. Future protocol, networking, storage, and runtime details should build on these boundaries rather than accidentally collapsing them.

## Key Architectural Principles Established in Part II

The following principles are directly supported by the article and should be preserved across the codebase and documentation.

- **Rust-first implementation strategy:** prioritize performance with stronger safety guarantees than traditional embedded defaults.
- **Portable core library:** put MoonBlokz logic into an embeddable library rather than a single fixed application.
- **`no_std` compatibility:** keep the core usable in constrained embedded targets while remaining reusable from standard applications.
- **Host-provided integration boundaries:** treat storage, randomness, crypto, clock, and radio as platform-facing interfaces.
- **High-level storage abstraction:** model persistence around blockchain operations rather than raw storage mechanics.
- **Non-blocking radio interaction:** use buffered message passing and explicit processing steps.
- **Host-owned control flow:** keep event-loop control in the embedding application.
- **Deferred low-level design:** postpone algorithmic and data-structure commitments until later series parts define them.

## Open Questions and Deferred Decisions

Part II intentionally leaves several design areas open. They should be treated as pending architectural topics rather than assumed behavior.

### Open technical questions explicitly or implicitly left for later

- How blockchain algorithms and consensus behavior are defined in detail.
- How blocks, indexes, and on-device persistence structures are represented concretely.
- How monotonic elapsed time is preserved across reboot.
- How radio frames are encoded, segmented, retried, or validated in detail.
- How software and hardware crypto implementations are selected and validated.
- How source distribution, licensing, and possible openness of the project will be handled.

These are not contradictions. They are scoped deferrals.

## Relationship to the Rest of the Series

Within the broader MoonBlokz series, this document should be understood as the bridge between concept and implementation.

- **Part I** defines the motivation, use cases, and core constraints.
- **Part II** selects the technology platform and high-level architecture.
- **Later parts** are expected to define algorithms, data structures, and protocol behavior that fit inside the boundaries established here.

## Review Notes

Post-change review against `moonblokz-info` rules:

- **Consistency:** This document stays consistent with the cited Part II article and with the broader project framing already documented in `moonblokz-overview.md`.
- **Logical soundness:** The text distinguishes article-stated facts from architectural interpretation and avoids inventing unsupported low-level mechanisms.
- **Feasibility:** The described architecture is feasible for an embedded-first Rust project because it isolates hardware-specific concerns behind integration boundaries and keeps the portable core compact in scope.
- **Potential ambiguity noted:** The article names the planned library as `moon_blokz_lib`. Current repository naming conventions may differ. This document preserves the article's wording rather than assuming a final crate name.
