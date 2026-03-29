# MoonBlokz Radio Implementation Notes

## Purpose of This Document

This document captures implementation-facing implications of the current MoonBlokz radio design as implemented in `moonblokz-radio-lib`.

It complements the conceptual and algorithm radio documents by identifying:

- what the codebase is actually responsible for today,
- how the crate is structured,
- what public API surface exists,
- what compile-time features and memory profiles shape behavior,
- how the different radio backends behave,
- which engineering constraints are explicit in the implementation,
- and which details still remain open or code-evolution-sensitive.

- Use [moonblokz-radio-concept.md](./moonblokz-radio-concept.md) for the strategic and architectural explanation.
- Use [moonblokz-radio-algorythm.md](./moonblokz-radio-algorythm.md) for the formal runtime and relay behavior.

## Source Basis

This document is grounded primarily in the current `moonblokz-radio-lib` codebase, especially:

- `moonblokz-radio-lib/Cargo.toml`
- `moonblokz-radio-lib/src/lib.rs`
- `moonblokz-radio-lib/src/messages/*`
- `moonblokz-radio-lib/src/tx_scheduler.rs`
- `moonblokz-radio-lib/src/rx_handler.rs`
- `moonblokz-radio-lib/src/relay_manager.rs`
- `moonblokz-radio-lib/src/wait_pool.rs`
- `moonblokz-radio-lib/src/radio_devices/*`
- `moonblokz-radio-lib/README.md`
- `moonblokz-radio-lib/AGENTS.md`

Where earlier article-era descriptions diverge from the code, this document treats the code as authoritative.

## Scope and Intent

This is not a line-by-line commentary of the library. Instead, it records the engineering consequences of the current code so future work can preserve the important guarantees already embedded in the implementation.

## Current Crate Structure

The main current structure is:

- `src/lib.rs` — public API, compile-time guards, queue definitions, configuration types, manager initialization
- `src/messages/` — `RadioMessage`, `RadioPacket`, iterators, checksum logic, fragmentation helpers
- `src/tx_scheduler.rs` — outbound scheduling and packet emission
- `src/rx_handler.rs` — inbound packet handling, reassembly, duplicate suppression, relay/application routing
- `src/relay_manager.rs` — connection matrix, echo logic, timed tasks, application-result integration
- `src/wait_pool.rs` — delayed relay items, scoring, activation, eviction
- `src/radio_devices/` — active backend implementations and link-quality calculation helpers

This structure matches the current architecture invariants documented in the repository guidance.

## Compile-Time Feature Model

The crate currently uses compile-time feature selection in several dimensions.

### Radio backend selection

Exactly one of these must be enabled:

- `radio-device-echo`
- `radio-device-rp-lora-sx1262`
- `radio-device-simulator`

### Memory profile selection

Exactly one of these must be enabled:

- `memory-config-small`
- `memory-config-medium`
- `memory-config-large`

### Environment selection

- `std`
- `embedded`

### Optional behavior features

- `soft-packet-crc`
- `connection-matrix-logging`

### Default features

The current default is:

- `radio-device-echo`
- `memory-config-large`

## Engineering Meaning of the Feature Model

The compile-time feature structure matters because it encodes several non-negotiable design decisions:

- one active backend only,
- one memory profile only,
- explicit environment distinction,
- and protocol/runtime behavior partly selected at build time.

This means future evolution should preserve compile-time exclusivity unless the entire crate architecture is intentionally redesigned.

## Public API Surface

The current documented public interface is centered on `RadioCommunicationManager`.

### Public manager methods

- `new()`
- `initialize(...)`
- `send_message(...)`
- `receive_message().await`
- `report_message_processing_status(...)`

### Re-exported public types

The crate currently re-exports:

- `RadioMessage`
- `RadioPacket`
- `MessageType`
- `MessageError`
- `EchoResultItem`
- `EchoResultIterator`
- `calculate_link_quality`
- `normalize`

### Public config and result types

- `RadioConfiguration`
- `ScoringMatrix`
- `SendMessageError`
- `ReceiveMessageError`
- `RadioInitError`
- `IncomingMessageItem`
- `MessageProcessingResult`
- `ReceivedPacket`

## Initialization Strategy

The code currently supports two initialization modes.

### Embedded mode

- all channels are statically allocated globals,
- `initialize()` reuses those static queues,
- no heap is involved.

### `std` mode

- fresh channels are created at initialization,
- each channel is `Box::leak`-ed to obtain a `'static` reference for Embassy tasks,
- this intentionally uses heap allocation only on the host/simulation side.

### Engineering implication

This split is important and should remain explicit. The embedded path is heapless by design; the host path trades that for convenience and testability.

## Queue and Buffer Design

The code uses many fixed-size channels and buffers.

### Queue families currently present

- outgoing message queue
- incoming message queue
- TX packet queue
- RX packet queue
- RX state queue
- process result queue

### Buffer/state structures currently present

- incoming packet buffer
- duplicate cache (`LastReceivedMessages`)
- connection matrix
- wait pool
- echo response wait pool

### Engineering implication

The queue/buffer topology is part of the three-task async runtime pipeline, not only an implementation detail. Each queue marks a handoff boundary between independent responsibilities.

## Deterministic Memory Profiles

The crate currently documents three memory profiles.

### Small

Approximate RAM target: **25 KB**

Current structural capacities:

- connection matrix: 10
- incoming packet buffer: 20
- wait pool: 5
- echo response wait pool: 3
- duplicate cache: 10
- outgoing message queue: 2
- incoming message queue: 2
- TX packet queue: 16
- RX packet queue: 2
- process result queue: 2

### Medium

Approximate RAM target: **60 KB**

Current structural capacities:

- connection matrix: 30
- incoming packet buffer: 30
- wait pool: 10
- echo response wait pool: 5
- duplicate cache: 20
- outgoing message queue: 3
- incoming message queue: 3
- TX packet queue: 32
- RX packet queue: 3
- process result queue: 5

### Large

Approximate RAM target: **120 KB**

Current structural capacities:

- connection matrix: 100
- incoming packet buffer: 50
- wait pool: 20
- echo response wait pool: 10
- duplicate cache: 30
- outgoing message queue: 8
- incoming message queue: 10
- TX packet queue: 64
- RX packet queue: 5
- process result queue: 10

Across all profiles, the RX state queue size is currently fixed at 20.

The code ties these profiles directly to queue sizes, matrix size, and buffer capacities.

### Engineering implication

Performance tuning and memory tuning are intentionally linked. Any future change to queue depths or matrix sizes should be considered part of the memory-profile contract.

## Message Model Engineering Implications

The current message layer is built around a byte-oriented `RadioMessage` type and a public-field `RadioPacket` type.

### Why this matters

The code is optimizing for:

- predictable serialized layout,
- low overhead,
- explicit control over fragmentation,
- and compatibility with fixed-size radio buffers.

### Practical implication

Future changes should preserve the message/packet split and avoid pushing too much enum-heavy abstraction into hot paths unless there is a strong reason.

## Protocol Surface and Document Boundary

The current code exposes a concrete radio protocol surface through `RadioMessage`, `RadioPacket`, `MessageType`, typed constructors, and message-specific accessors.

The exact currently implemented message set, field semantics, and reply relationships are documented in [moonblokz-radio-algorythm.md](./moonblokz-radio-algorythm.md), because that is the radio document dedicated to formal protocol behavior.

### Important discrepancy to keep explicit

The current codebase does **not** implement a separate `RequestTransactionPart` message type yet, even though some earlier article-derived summaries described one. The knowledge base should therefore reflect the code as it exists today while also treating `RequestTransactionPart` as a likely future protocol extension rather than a rejected idea.

## `RadioConfiguration` as the Runtime Tuning Surface

The current code defines `RadioConfiguration` with these fields:

- `delay_between_tx_packets`
- `delay_between_tx_messages`
- `echo_request_minimal_interval`
- `echo_messages_target_interval`
- `echo_gathering_timeout`
- `relay_position_delay`
- `scoring_matrix`
- `retry_interval_for_missing_packets`
- `tx_maximum_random_delay`

### Engineering implication

This struct is the current main behavioral tuning surface for the runtime. It is narrower and more explicit than article-only descriptions.

## `ScoringMatrix` as Compact Relay Policy

The current code defines a concrete `ScoringMatrix` type containing:

- a 4×4 score matrix,
- `poor_limit`,
- `excellent_limit`,
- `relay_score_limit`.

It also supports reconstruction from a compact 5-byte encoded form.

### Engineering implication

The compact encoding is no longer only a conceptual idea. It is part of the actual code interface and should be treated as current implementation behavior.

## Current Application/Radio Contract

The application and radio layers currently interact in a specific loop.

### Application receives

- newly reconstructed messages,
- duplicate-check queries.

### Application reports back

- new block added,
- new transaction added,
- support added,
- requested block found/not found,
- requested block parts found,
- reply transaction to send,
- already-have-message.

### Engineering implication

The embedding application is part of the effective protocol runtime. The radio layer relies on it for validation, duplicate knowledge, and some request resolution.

## TX Scheduler Engineering Notes

The code in `tx_scheduler.rs` currently enforces:

- inter-message delay,
- inter-packet delay,
- random jitter before sending,
- receive-state-induced pauses during foreign multi-packet reception,
- packet-by-packet forwarding into the radio device queue.

### Important detail

If `tx_packet_queue` is full, individual packets are dropped. This means packet drop can happen after logical message acceptance and before physical transmission.

## RX Handler Engineering Notes

The code in `rx_handler.rs` currently handles:

- single-packet fast-path processing,
- multi-packet buffering and assembly,
- duplicate suppression through recent-message cache,
- application-assisted duplicate suppression for multi-packet messages,
- packet-buffer eviction by oldest-message age,
- CRC validation for full `AddBlock` and `AddTransaction` payloads,
- periodic missing-packet scan.

### Important implementation nuance

The current explicit missing-fragment retry path is implemented for `AddBlock` fragment recovery. The code does not presently expose a symmetric dedicated transaction-part request path yet, so transaction-part recovery should be understood as a likely future extension rather than current behavior.

## Relay Manager Engineering Notes

The current relay manager owns:

- connection matrix storage and node indexing,
- periodic echo scheduling,
- echo-response buffering,
- timed task selection,
- connection updates from packets and echo data,
- application-result-driven relay insertion,
- and duplicate/wait-pool coordination.

### Important implementation nuance

The relay manager does not store a global topology. It stores only bounded local state and uses linear searches intentionally because the bounded sizes are small enough for that trade-off.

## Connection Matrix Concrete Storage Model

The current code stores the connection matrix as:

- `connection_matrix: [[u8; N]; N]`
- `connection_matrix_nodes: [u32; N]`

### Cell encoding

- lower 6 bits = link quality
- upper 2 bits = dirty counter

### Engineering implication

This is now a concrete implementation contract, not just a conceptual matrix.

## Wait Pool Engineering Notes

The wait pool currently:

- stores delayed relay items in a fixed-size array,
- computes relay score from the current connection matrix and message coverage,
- recalculates score and activation time when new relays are observed,
- drops low-score messages,
- evicts the currently weakest item when full if a better item arrives.

### Important implementation nuance

Targeted replies use `requestor_index` and are scored only against the requester connection rather than the whole visible network.

## Radio Device Backend Inventory

The current backend set is:

### `echo`

- immediate loopback device,
- quality always 63,
- no CAD, no timing realism,
- useful for smoke tests and basic unit tests.

### `simulator`

- queue-based bridge to a network simulator,
- simulates CAD responses and busy-channel backoff,
- useful for multi-node testing without hardware.

### `rp_lora_sx1262`

- real RP2040 + SX1262 LoRa backend,
- async Embassy-based hardware integration,
- CAD-before-transmit runtime,
- optional software packet CRC,
- actual RSSI/SNR-based link-quality calculation.

## RP2040 SX1262 Backend Details Grounded in Code

The current RP backend:

- uses SPI1,
- takes explicit pins for NSS, reset, DIO1, busy, and optional transmit enable,
- allows explicit `tcxo_ctrl` configuration,
- configures modulation and packet parameters during initialization,
- uses a receive buffer sized as `RADIO_PACKET_SIZE + 2`.

### Important hardware behavior encoded in code

- CAD timeout exists because CAD may stall,
- software packet CRC exists because hardware CRC validation was unreliable,
- CAD busy handling uses randomized backoff.

## Link-Quality Utility Implementation Notes

The current code exports:

- `normalize(value, min, max)`
- `calculate_link_quality(rssi, snr)`

The current normalization constants are:

- RSSI min/max: `-120`, `-30`
- SNR min/max: `-20`, `10`

This means any future change here could materially shift relay behavior and connection-matrix interpretation, even without changing packet formats.

## Testing and Build Implications

The repository guidance and current crate design imply several important engineering practices.

### Typical test feature sets

- `std + radio-device-echo + memory-config-large`
- `std + radio-device-simulator + memory-config-large`

### Embedded build target

- RP2040 example using `thumbv6m-none-eabi`

### Engineering implication

The crate is intended to be exercised across both host-mode and embedded-mode configurations. Documentation should therefore avoid assuming that only one environment matters.

## Code-Grounded Limitations and Cautions

The current implementation still leaves several things open or fragile.

### 1. Message-set mismatch with older summaries

Earlier summaries may imply `RequestTransactionPart`; current code does not implement it yet.

### 2. Some naming remains article-era or legacy-shaped

For example, the constructor for `RequestNewMempoolItem` is currently named `get_mempool_state_with`.

### 3. Queue drops are normal runtime behavior

Users of the library must assume bounded channels can reject work under pressure.

### 4. No built-in security layer

Encryption/authentication remain higher-layer concerns.

### 5. Host and embedded initialization differ materially

`std` uses leaked heap allocations for queue lifetime; embedded uses pure static allocation.

## Relationship to the Radio Simulator Documents

Part VII/4 of the MoonBlokz series adds several implementation-facing conclusions that matter for this radio library as well:

- the library architecture is reusable in a host-mode multi-node simulator,
- Embassy portability across embedded and desktop targets is strategically valuable,
- queue lifetime assumptions can be adapted in the simulator without rewriting the radio runtime itself,
- and end-to-end simulation is important for validating relay efficiency, collision behavior, and missing-fragment recovery beyond unit-level tests.

For those companion implementation notes, continue with:

- [moonblokz-simulator-concept.md](./moonblokz-simulator-concept.md)
- [moonblokz-simulator-algorythm.md](./moonblokz-simulator-algorythm.md)
- [moonblokz-simulator-implementation.md](./moonblokz-simulator-implementation.md)

## What Must Remain Open

Even with the codebase as source of truth, several implementation areas remain intentionally open or evolution-sensitive:

- the simulator’s broader network and physics model,
- future backend additions or replacements,
- future queue-size tuning per profile,
- future protocol extensions beyond the current message set,
- and any later refinement of CAD, CRC, or recovery policy.
