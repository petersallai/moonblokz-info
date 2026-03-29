# MoonBlokz Radio Concept Model

## Purpose of This Document

This document explains the conceptual operating model of the MoonBlokz radio subsystem as it is currently implemented in `moonblokz-radio-lib`.

Its purpose is to describe:

- what the radio layer is trying to achieve in MoonBlokz,
- what reliability model it uses,
- why the runtime architecture is built around bounded async tasks and static memory,
- how local topology knowledge, echo mapping, and adaptive relaying fit together,
- how the radio layer cooperates with the blockchain and application layers,
- and which important boundaries or limitations are explicit in the current codebase.

This file is intentionally focused on conceptual understanding rather than exact wire layout, queue constants, or line-by-line code structure.

- Use this document to understand **what the radio subsystem is**, **why it behaves this way**, and **which core design principles the current code preserves**.
- Use [moonblokz-radio-algorythm.md](./moonblokz-radio-algorythm.md) for the formal runtime flow, message handling rules, relay scoring, and recovery behavior.
- Use [moonblokz-radio-implementation.md](./moonblokz-radio-implementation.md) for the code-grounded API, feature flags, backend details, memory profiles, and engineering constraints.

## Source Basis

This document is grounded primarily in the current `moonblokz-radio-lib` codebase, especially:

- `moonblokz-radio-lib/Cargo.toml`
- `moonblokz-radio-lib/src/lib.rs`
- `moonblokz-radio-lib/src/messages/radio_message.rs`
- `moonblokz-radio-lib/src/messages/radio_packet.rs`
- `moonblokz-radio-lib/src/tx_scheduler.rs`
- `moonblokz-radio-lib/src/rx_handler.rs`
- `moonblokz-radio-lib/src/relay_manager.rs`
- `moonblokz-radio-lib/src/wait_pool.rs`
- `moonblokz-radio-lib/src/radio_devices/*`
- `moonblokz-radio-lib/README.md`

The earlier MoonBlokz article series remains useful background, but where article-era summaries and the current code differ, this document treats the code as authoritative.

## Relationship to Earlier MoonBlokz Documents

This file should be read after:

- [moonblokz-overview.md](./moonblokz-overview.md), which explains the MoonBlokz mission and operating environment,
- [moonblokz-technology.md](./moonblokz-technology.md), which explains the architecture and embedded-platform constraints,
- and ideally after the blockchain documents, because the radio layer exists mainly to keep blockchain state synchronized across a lossy local network.

This document answers the conceptual question:

**What kind of radio subsystem does MoonBlokz need when it must synchronize blockchain state over constrained broadcast hardware while keeping memory, timing, and overload behavior deterministic?**

## Core Terminology Used Across the Radio Documents

To keep the three radio knowledge-base files aligned, this terminology is used consistently throughout:

- **RadioMessage** — the logical application-level message handled by the MoonBlokz radio stack.
- **RadioPacket** — the fixed-size radio transmission unit sent over the physical link.
- **Echo mapping** — the active topology-discovery process using `request_echo`, `echo`, and `echo_result`.
- **Connection matrix** — a bounded local matrix of directional link-quality values between known nodes.
- **Wait pool** — the bounded delayed-relay structure that decides when or whether this node should retransmit a message.
- **Best-effort synchronization** — the design principle that the radio layer tries to spread current blockchain state broadly and quickly without guaranteeing delivery of every individual message.
- **Deterministic degradation** — the design principle that overload should result in bounded message loss rather than unbounded resource growth.

## Business Analyst View: What Problem the Radio Layer Solves

MoonBlokz is intended to run in environments where conventional internet infrastructure cannot be assumed. The radio subsystem therefore acts as the distributed communication layer that keeps the blockchain moving across a local mesh of constrained nodes.

The current codebase makes the business objective very clear:

- the radio layer is **not** a general-purpose chat or transport protocol,
- it is **not** optimized for guaranteed delivery of every packet,
- and it is **not** trying to simulate a centralized LoRaWAN-style network.

Instead, the radio layer is designed to:

- propagate new blocks, transactions, and support information widely enough for the blockchain to progress,
- recover missing pieces when they matter,
- minimize unnecessary airtime use,
- and do all of this within strict memory and timing limits suitable for embedded hardware.

Conceptually, MoonBlokz’s radio layer solves a **state-synchronization problem**, not a perfect-delivery problem.

## Why MoonBlokz Uses Direct LoRa Rather Than LoRaWAN

The current library still reflects the earlier strategic decision to use direct LoRa-style packet transport instead of LoRaWAN.

Conceptually, that means MoonBlokz wants:

- ad hoc peer-to-peer communication,
- no gateway-centered topology,
- no central network server,
- and no dependency on infrastructure that would break the project’s hyper-local operating goal.

The current code reinforces this choice by exposing a pluggable radio-device layer while keeping the higher-level mesh behavior fully MoonBlokz-specific.

## The Core Mission of the Radio Layer

The current codebase preserves one central mission:

- spread blockchain-relevant information broadly enough and quickly enough that the network can continue converging,
- while accepting that some individual packets or even some first-pass messages will be lost.

This leads to three conceptual consequences.

### 1. Fire-and-forget is normal

The library does not implement an acknowledgment-heavy transport protocol. Most successful deliveries are not explicitly confirmed.

### 2. Recovery is reactive

Instead of insisting on perfect first delivery, the system detects missing data later and requests what it needs.

### 3. Resource bounds are more important than perfect reliability

The implementation repeatedly chooses bounded queues, bounded buffers, and message dropping over potentially unbounded retries or memory growth.

## Radio Design Goals Preserved in Code

The current implementation implies the following durable design goals.

### Ad hoc decentralized topology

Nodes communicate through broadcast radio without central control.

### Local-only knowledge

Each node maintains only a local view of nearby network quality rather than a global topology graph.

### Loss tolerance

Some message loss is acceptable as long as the network can recover important missing information and the blockchain can keep converging.

### Airtime awareness

The runtime architecture, echo scheduling, and relay suppression are all designed to conserve airtime.

### Deterministic memory behavior

Static allocation, fixed-size buffers, and fixed-size queues are central properties of the design.

### Backend modularity

The higher-level mesh logic is designed to sit above a replaceable radio-device backend rather than being hardwired to one single hardware implementation.

## Architect View: The Radio Subsystem as a Three-Task Async Runtime Pipeline

The current code organizes the radio subsystem into three major async runtime tasks:

- **TX Scheduler**
- **Radio Device**
- **RX Handler**

These are connected through bounded queues.

Conceptually, this matters because MoonBlokz’s radio layer is not just a set of message rules. It is a **pipeline** with explicit points where work is accepted, delayed, transformed, dropped, or handed upward.

This layered structure separates concerns cleanly:

### TX Scheduler

Responsible for outbound pacing, packetization, and respecting receive-side coordination signals.

### Radio Device

Responsible for low-level physical send/receive/CAD behavior.

### RX Handler

Responsible for inbound packet collection, reassembly, duplicate suppression, relay coordination, and delivery to the application.

This separation is one of the most important architectural facts in the current codebase.

## Deterministic Degradation as a First-Class Design Principle

A major conceptual property of the current implementation is that failure under pressure is meant to be **predictable**.

If the system is overloaded:

- new outgoing messages may be dropped,
- queued packets may be dropped,
- newly received messages may be dropped,
- stale partial messages may be discarded.

This is not accidental behavior. It is the intended embedded-systems trade-off.

MoonBlokz explicitly prefers:

- bounded queues,
- bounded buffers,
- and message loss under pressure,

instead of:

- unbounded queue growth,
- heap exhaustion,
- or unpredictable blocking cascades.

## Static Memory as a Conceptual Commitment

The codebase makes static memory behavior part of the subsystem identity.

Conceptually, this means:

- all important working structures have fixed size,
- embedded builds avoid heap dependence,
- memory configuration is chosen at build time,
- and the runtime is designed around what is already known at compile time.

This is more than an optimization. It is part of how MoonBlokz maintains embedded safety and predictability.

## RadioMessage and RadioPacket as Distinct Conceptual Layers

The code draws a clear boundary between:

- **RadioMessage** — what the application and relay logic think in,
- **RadioPacket** — what the physical radio actually sends.

This distinction is conceptually important because:

- only some messages are fragmented,
- duplicate suppression and checksum logic operate partly at message level and partly at packet level,
- and relay logic reasons about messages, not just raw packets.

MoonBlokz therefore treats packetization as a transport concern layered beneath the logical message model.

## Byte-Oriented Message Model as a Deliberate Conceptual Trade-off

The codebase clearly chooses a byte-oriented message representation instead of a deeply nested enum hierarchy.

Conceptually, this means the project values:

- compactness,
- simpler wire-format evolution,
- direct control over serialized data,
- and predictable memory layout.

To compensate for the lower type richness, the code uses:

- typed constructors,
- accessor methods,
- iterators for structured payload subformats,
- and explicit error-returning mutation methods.

So the design is not “untyped”; it is **byte-oriented with controlled typed access**.

## Echo Mapping as the Basis of Local Network Awareness

The current code preserves echo mapping as the main source of local topology knowledge.

Conceptually:

- nodes periodically ask who can hear them,
- responders report measured quality,
- summaries are rebroadcast,
- and the results refresh the local connection matrix.

The code also adds an important conceptual refinement:

- echo responses are handled through a dedicated response pool and delayed slightly to reduce collision risk.

So echo traffic remains part of normal radio behavior, but it is treated as a special control flow rather than as ordinary blockchain-content relaying.

## The Connection Matrix as the Core Conceptual State Structure

The **connection matrix** remains the central local radio-state model.

Each node stores directional quality information between tracked nodes in a bounded matrix.

The current code makes several conceptual facts explicit:

- the matrix is **directional**, not symmetric,
- it is **bounded** by compile-time capacity,
- it is **aging**, using dirty counters to phase out stale entries,
- it is paired with a separate node-ID address book,
- and it is updated both by echo flow and by ordinary packet reception.

This means the matrix is not just a passive topology cache. It is the working state behind relay decisions, request targeting, and quality-based prioritization.

## Wait Pool as the Conceptual Heart of Stateful Relaying

The current code makes the **wait pool** one of the most important conceptual relay structures.

Messages that may be relayed are not usually transmitted immediately. Instead, they are inserted into a bounded delayed queue where their usefulness is re-evaluated over time.

Conceptually, this enables:

- better-positioned relays to transmit first,
- weaker relays to drop out later if they add little value,
- request-targeted messages to use different scoring than broad broadcasts,
- and dynamic updates as new relays are heard.

This means MoonBlokz relaying is not a one-shot yes/no decision. It is a **stateful timed competition** among possible relays.

## Application Validation as a Conceptual Relay Gate

One of the strongest conceptual facts in the current codebase is that the radio layer does not fully decide relaying alone.

Instead:

- the RX handler reconstructs and preliminarily processes a message,
- the application layer determines whether it is valid, new, already known, or answerable,
- then the radio layer uses that result to decide relay or reply behavior.

This means the current implementation conceptually treats propagation as **application-informed**.

MoonBlokz does not simply rebroadcast whatever it hears. It rebroadcasts what the higher layer has meaningfully accepted or turned into a valid response.

## Query/Reply Behavior as Part of the Same Mesh Model

The current code shows that request/response flows are not a separate protocol.

Instead, they are integrated into the same delayed relay framework:

- missing-block requests,
- block-part requests,
- mempool discovery,
- and the corresponding replies

all feed through the same bounded scheduling and relay machinery.

This is conceptually important because MoonBlokz keeps the network model unified. Requests, replies, and ordinary propagation are all handled as variations of the same mesh behavior rather than as disconnected subsystems.

## Reactive Recovery as a Layered Self-Healing Mechanism

The codebase preserves and sharpens the earlier reactive-recovery idea.

Recovery now exists on at least two conceptual layers.

### Blockchain/message-level recovery

If a required block is missing, a node requests it.

### Packet-fragment-level recovery

If a multi-packet message is incomplete, the RX handler periodically scans for old incomplete assemblies and requests only the missing packet indices.

This creates a self-healing radio layer beneath the blockchain logic. It allows the network to accept imperfect first-pass delivery while still converging later.

## Current Protocol Scope

Conceptually, the current protocol surface is narrower than some earlier article-era summaries implied.

In particular:

- the current code implements a concrete, bounded set of radio message families,
- selective block-fragment recovery is implemented,
- and transaction-part recovery remains a plausible future extension rather than current behavior.

For the exact currently implemented message set, field semantics, and reply relationships, use [moonblokz-radio-algorythm.md](./moonblokz-radio-algorythm.md). The purpose of this conceptual file is to explain **why** those protocol pieces exist and **how they fit together**, rather than to restate the full message catalog.

## Important Code-Grounded Scope Boundaries

The current codebase also makes several limits explicit.

### No network-layer security features

The radio library itself does not implement encryption or authentication. The code and README both treat those as higher-layer blockchain concerns.

### No ACK-driven transport

The implementation does not revolve around acknowledgments or handshake-based delivery.

### No global topology knowledge

Nodes reason from local connection state only.

### No unlimited buffers

All major queues and buffers are capacity-limited by design.

## What the Current Codebase Establishes

From the perspective of the MoonBlokz knowledge base, the current `moonblokz-radio-lib` implementation establishes these conceptual conclusions:

1. MoonBlokz radio is a **blockchain synchronization layer**, not a guaranteed-delivery transport.
2. The system is built around **three async runtime tasks** linked by bounded queues.
3. The implementation prefers **deterministic degradation** over unbounded growth.
4. Echo mapping and the connection matrix provide **local-only topology awareness**.
5. Relaying is **stateful, delayed, adaptive, and score-driven**.
6. The application layer participates directly in deciding what is new, valid, duplicate, or answerable.
7. Recovery is **reactive** and happens at both message and packet-fragment levels.
8. The radio device layer is **modular**, allowing multiple backends while preserving one higher-level mesh model.

## Relationship to the Radio Simulator Documents

Part VII/4 of the MoonBlokz series adds an important complementary perspective: the radio subsystem is not only an embedded communication layer but also something MoonBlokz intentionally validates through a dedicated multi-node simulator.

That simulator perspective does not change the conceptual role of the radio subsystem described here, but it sharpens two important points:

- the radio design must be reusable in host-mode simulation as well as on embedded hardware,
- and the practical behavior of relaying, collisions, topology depth, and missing-fragment recovery must be observable at network scale, not only reasoned about locally.

For that perspective, continue with:

- [moonblokz-simulator-concept.md](./moonblokz-simulator-concept.md)
- [moonblokz-simulator-algorythm.md](./moonblokz-simulator-algorythm.md)
- [moonblokz-simulator-implementation.md](./moonblokz-simulator-implementation.md)

## What Still Remains Open

Even with the current code as source of truth, some areas remain intentionally open or only partially documented:

- the simulator’s broader network model beyond the backend interface,
- future backend evolution beyond the current echo/simulator/SX1262 set,
- final long-term queue sizes and tuning decisions for all deployment profiles,
- and any protocol evolution that later code or later series parts may introduce.

## Technical Writer View: How to Read the Radio Knowledge-Base Set

This conceptual file should be read first when the goal is to understand the role and design philosophy of the MoonBlokz radio subsystem.

Then continue with:

- [moonblokz-radio-algorythm.md](./moonblokz-radio-algorythm.md) to see the formal current flow of messages, packetization, scheduling, relaying, reassembly, and recovery,
- and [moonblokz-radio-implementation.md](./moonblokz-radio-implementation.md) to see the actual code-level API, feature model, memory configuration, backend responsibilities, and engineering cautions.
