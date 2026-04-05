# MoonBlokz Simulator Concept Model

## Purpose of This Document

This document explains the conceptual operating model of the MoonBlokz radio simulator **as it currently exists in the `moonblokz-radio-simulator` codebase**.

Its purpose is to explain:

- what the simulator is trying to achieve for MoonBlokz,
- how the current application combines simulation, analysis, visualization, and remote control,
- why the simulator remains centered on reuse of the real radio library,
- what parts of reality it tries to preserve,
- what simplifications and scope limits remain explicit in the current implementation,
- and how the simulator now fits into the broader MoonBlokz workflow beyond the original Part VII/4 article framing.

This file is intentionally conceptual. It focuses on purpose, scope, design philosophy, and system boundaries rather than exact queue types, parsing details, or line-by-line code behavior.

- Use this document to understand **what the simulator/analyzer tool is for**, **why it exists as a multi-mode desktop surface**, and **where its boundary lies relative to the radio and telemetry subsystems**.
- Use [moonblokz-simulator-algorythm.md](./moonblokz-simulator-algorythm.md) for the formal runtime flow, scene semantics, signal model, analyzer logic, and UI-facing behaviors.
- Use [moonblokz-simulator-implementation.md](./moonblokz-simulator-implementation.md) for implementation-facing notes grounded in the current crate structure, Embassy integration, UI modules, analyzer pipeline, and control path.

## Source Basis

This document is grounded primarily in the current `moonblokz-radio-simulator` codebase, especially:

- `moonblokz-radio-simulator/Cargo.toml`
- `moonblokz-radio-simulator/src/main.rs`
- `moonblokz-radio-simulator/src/common/*`
- `moonblokz-radio-simulator/src/simulation/*`
- `moonblokz-radio-simulator/src/analyzer/*`
- `moonblokz-radio-simulator/src/control/*`
- `moonblokz-radio-simulator/src/ui/*`
- `moonblokz-radio-simulator/src/time_driver.rs`
- `moonblokz-radio-simulator/scenes/*`
- `moonblokz-radio-simulator/AGENTS.md`

The earlier Part VII/4 article remains useful background, but where the article and the current code differ in emphasis or scope, this document treats the **current codebase as authoritative**.

## Relationship to the Existing Radio Knowledge Base

The simulator documents extend the existing radio knowledge-base set in a specific way.

The radio documents explain:

- the communication protocol,
- the bounded async runtime,
- packetization and relay behavior,
- and the current `moonblokz-radio-lib` implementation model.

The simulator documents explain:

- how that radio behavior is exercised at network scale,
- how it is visualized inside the desktop application,
- how synthetic and log-driven node state are presented to the operator,
- and how one common UI surface is reused across simulation, replay, and live observation.

The broader field telemetry architecture behind those log-driven workflows now belongs primarily to the telemetry document set. Conceptually, the simulator is therefore best understood as the **desktop observability and experimentation environment that consumes telemetry context**, not as the primary home of Probe/HUB/Collector/CLI architecture.

## Business Analyst View: What Problem the Simulator Solves Today

The current codebase shows that the simulator now solves more than the original “network simulation before hardware deployment” problem.

It supports three distinct operator workflows:

1. **Simulation mode** — explore large synthetic scenes and observe how the radio algorithm behaves under configurable propagation assumptions.
2. **Real-time Tracking mode** (`RealtimeTracking`) — visualize and inspect live telemetry logs from deployed systems while also issuing remote control commands through Telemetry Hub.
3. **Log Visualization mode** (`LogVisualization`) — replay historical logs in time-synchronized form for post-mortem analysis.

This is an important conceptual expansion. The current application is not only a simulator anymore. It is a combined **simulator, analyzer, and operator console** for MoonBlokz radio behavior.

## Why Reusing the Real Radio Library Still Matters

The current code keeps the same strategic commitment emphasized in the article: each simulated node still runs the real `moonblokz-radio-lib` logic rather than a simplified reimplementation.

Conceptually, that matters because the simulator’s value comes from observing the real protocol behavior under many-node conditions, including:

- the same async task structure,
- the same queue semantics,
- the same relaying logic,
- the same message fragmentation and reassembly behavior,
- and the same CAD-driven transmit coordination model.

The simulator therefore remains a **validation environment around the real radio runtime**, not an abstract educational model.

## The Simulator as a Multi-Mode Desktop Tool

The current application architecture establishes a broader conceptual model than the article alone suggested.

### Simulation mode

This is the most article-like mode.

It loads a synthetic scene, spawns many in-process nodes, runs the radio stack, models path loss, obstacles, airtime overlap, CAD timing, reception, and collisions, and renders the evolving network state.

### Real-time Tracking mode

This mode turns the application into a field-observation console.

It follows a live log stream, reconstructs network activity from telemetry-tagged logs, displays node and message activity on the same UI, and exposes hub-mediated control affordances when telemetry control is configured.

### Log Visualization mode

This mode replays historical logs from the beginning, preserving inter-event timing so users can inspect how a real network session unfolded.

Conceptually, the simulator owns the desktop experience of these modes. The broader telemetry meaning of live uploads, hub coordination, delayed downloads, command queueing, and OTA operations belongs in the telemetry documents. This means the current product concept is no longer just “simulate first, test later.” It is now “simulate, observe, replay, and consume telemetry-driven operations through one tool surface.”

## What Reality the Current Simulator Tries to Preserve

The current codebase preserves several kinds of reality that matter for MoonBlokz.

### Protocol reality

Because the radio library is reused, the current simulator preserves the real MoonBlokz radio behavior far better than a simplified model would.

### Topology reality

Node placement, world dimensions, obstacles, and precomputed effective distances all shape what can happen.

### Timing reality

Embassy-based async execution, CAD windows, packet airtime, inter-packet delays, and scaled virtual time preserve meaningful timing relationships even though the environment is synthetic.

### Message-history reality

The UI does not only show anonymous pulses. It exposes packet streams, full-message streams, and log streams per node.

### Field-observation reality

Analyzer modes let the same interface be driven by real telemetry logs rather than only by synthetic scene generation.

## What the Current Simulator Deliberately Simplifies

The codebase still preserves the same deliberate approximations that were already present in the article, but now the code makes them sharper.

### 2D geometry remains intentional

All map rendering, world coordinates, scene geometry, and obstacle logic remain fundamentally 2D.

The project guidance explicitly states that this is intentional for speed and clarity, even though the underlying radio concepts may later need to serve 3D deployment realities.

### Obstacles remain binary blockers

The current geometry and network code treat circles and rectangles as line-of-sight blockers.

Obstacles do not weaken signals gradually. They either block the straight path or they do not.

### RF propagation remains approximate

The current physical model includes:

- log-distance path loss,
- optional random shadowing,
- SNR-based decode thresholds,
- summed interference in linear power units,
- and a simplified capture-effect threshold.

But it still omits:

- reflections,
- diffraction,
- fine-grained multipath behavior,
- detailed modem behavior beyond the simplified model,
- and hardware-specific analog edge cases.

### Host-time control remains synthetic

The time driver preserves meaningful virtual timing but it is still a software-controlled host-side clock, not a reproduction of independent oscillator drift across all nodes.

## Architect View: The Current Simulator as Four Coordinated Subsystems

The current codebase is best understood as four coordinated conceptual subsystems.

### 1. The simulation subsystem

This loads scenes, spawns nodes, models propagation, evaluates packet receptions, and feeds UI updates.

### 2. The analyzer subsystem

This parses telemetry-style logs, reconstructs per-node packet and full-message histories, and reuses the same UI model for replay and live tracking. It is the simulator-side consumer of the telemetry stream rather than the source of telemetry architecture.

### 3. The control subsystem

This provides a simulator-side bridge to hub-mediated out-of-band commands during real-time tracking mode. The simulator owns the desktop affordance; the broader command-routing model belongs in the telemetry documents.

### 4. The UI subsystem

This gives all three operating modes one common operator surface: map, metrics, node inspector, connection-matrix visualization, measurement status, and control dialogs.

Conceptually, this means the current application is a **single desktop front end for multiple MoonBlokz radio-observation workflows**.

## Scene Files as Shared Operational Context

The current code shows that scene files are no longer only synthetic simulation inputs.

In simulation mode, they define:

- path-loss parameters,
- LoRa parameters,
- radio-module configuration,
- nodes,
- obstacles,
- world bounds,
- and optional background imagery.

In analyzer modes, they also serve as the static reference context that explains:

- where nodes are located,
- what their effective distances are,
- what map background to use,
- and what link-quality thresholds the UI should use.

This makes the scene file a shared bridge between **simulation** and **field analysis**.

## Visualization Is Now a Core Product Surface

The current code confirms even more strongly than the article that visualization is not a side feature.

The UI is designed around:

- a world map,
- radio transmission animations,
- selected-node overlays,
- measurement progress,
- per-node radio streams,
- per-node full-message streams,
- per-node log streams,
- and connection-matrix inspection.

This means the conceptual purpose of the application is not simply to “run” the model. It is to make the network understandable to a human operator.

## Connection Matrix Inspection as a First-Class Concept

One important conceptual refinement visible in the current code is the presence of explicit connection-matrix inspection.

The application can now surface connection-matrix information in both synthetic and log-driven workflows, visualize directional links, and expose decoded matrix contents in the inspector.

This turns local topology knowledge into a visible operator concept rather than a hidden internal state.

## Measurement Support as a Conceptual Bridge Between Simulation and Deployment

The current code keeps measurement support as a major concept, but broadens how it is used.

In simulation mode, the application can inject a synthetic `AddBlock` message from a selected node.

In real-time tracking mode, it can send a `start_measurement` command to the Telemetry Hub for a selected node.

The UI then tracks:

- which nodes were reached,
- how many packets were sent during measurement,
- and milestone times such as 50%, 90%, and 100% coverage.

Conceptually, this makes measurement a shared evaluation language across both simulation and live systems.

## Remote Control as an Explicit Out-of-Band Concept

The simulator codebase also makes one architectural boundary very explicit: control traffic is outside the radio mesh.

At the simulator boundary, that means real-time tracking may consult `config.toml` next to the selected scene and issue hub-mediated control requests for the observed network. The exact hub contract, command queueing, and Probe-side execution semantics belong in the telemetry documents.

This is conceptually aligned with the broader MoonBlokz architecture, where telemetry and control are kept out-of-band relative to low-bandwidth mesh traffic.

## Current Scope Boundaries

The current codebase also makes several limits explicit.

### No adversarial simulation layer

There is still no malicious-node or protocol-attack simulation model here.

### No fine-grained physical RF simulator

The application remains a network-level and protocol-level simulator, not a laboratory RF emulator.

### No unified “all modes at once” execution

The main application selects one operating mode at startup and then spawns either simulation or analyzer logic.

### No guarantee that visualization equals perfect truth

Both simulation and analyzer flows simplify, filter, and summarize data for UI usefulness.

The UI is an observability surface, not a mathematically complete record of all internal state.

## What the Current Codebase Establishes

From the perspective of the MoonBlokz knowledge base, the current `moonblokz-radio-simulator` codebase establishes these conceptual conclusions:

1. the simulator is now a **multi-mode desktop application**, not only a synthetic simulation harness,
2. reuse of `moonblokz-radio-lib` remains a core architectural commitment,
3. the application now combines **simulation**, **live tracking**, **historical replay**, and **remote control**,
4. scene files act as a shared operational context for both synthetic and field-observation workflows,
5. visualization and inspection are first-class parts of the product concept,
6. measurement support spans both simulation and live network analysis,
7. connection-matrix visibility is now an explicit operator-facing concept,
8. out-of-band telemetry control is a core architectural boundary preserved by the current design.

## What Still Remains Open

Even with the current code as source of truth, some conceptual areas remain intentionally open or only partially modeled:

- richer RF effects beyond path loss, shadowing, simplified capture, and obstacle blocking,
- true 3D deployment visualization,
- adversarial or malicious-behavior analysis,
- longer-term evolution of analyzer telemetry formats,
- and any future unification or expansion of operating modes.

## Technical Writer View: How to Read the Radio Simulator Document Set

Read this conceptual document first when you want to understand:

- what the simulator currently is,
- how its role has expanded beyond the original article,
- what realities it preserves,
- and what boundaries still define the current product.

Then continue with:

- [moonblokz-simulator-algorythm.md](./moonblokz-simulator-algorythm.md) for the formal simulation, analyzer, and timing behavior,
- [moonblokz-simulator-implementation.md](./moonblokz-simulator-implementation.md) for the engineering implications of the current crate structure,
- [moonblokz-telemetry-concept.md](./moonblokz-telemetry-concept.md) and its companion telemetry files when you need the broader Probe/HUB/Collector/CLI field-testing architecture around the analyzer workflows,
- and the main radio documents if you need the underlying protocol behavior being exercised and visualized.
