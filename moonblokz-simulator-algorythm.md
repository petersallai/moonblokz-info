# MoonBlokz Simulator Algorithm Model

## Purpose of This Document

This document provides a formal, algorithm-oriented description of the MoonBlokz radio simulator **as currently implemented in `moonblokz-radio-simulator`**.

Its purpose is to capture:

- the actual top-level runtime structure,
- the mode-selection flow,
- the current scene-file model and validation rules,
- the simulation-side node/network event algorithm,
- the current signal-propagation and reception rules,
- the analyzer-side log replay and real-time tracking behavior,
- the UI-facing update contract,
- the current time-scaling algorithm,
- and the exact boundaries of what the codebase still does not model.

This file is the primary knowledge-base document for the current **formal simulator and analyzer behavior**.

- Use [moonblokz-simulator-concept.md](./moonblokz-simulator-concept.md) for the strategic role, system boundaries, and product-level meaning.
- Use [moonblokz-simulator-implementation.md](./moonblokz-simulator-implementation.md) for crate structure, implementation decisions, and engineering cautions.

## Source Basis

This document is grounded primarily in the current `moonblokz-radio-simulator` codebase, especially:

- `src/main.rs`
- `src/common/scene.rs`
- `src/common/connection_matrix.rs`
- `src/simulation/*`
- `src/analyzer/*`
- `src/control/*`
- `src/ui/*`
- `src/time_driver.rs`

Where earlier article-era descriptions differ from the current code, this document treats the code as authoritative.

## Scope and Limits

This document captures the behavior explicitly visible in the current simulator application, including:

- mode dispatch,
- scene parsing and validation,
- node and network queues,
- CAD and airtime-window processing,
- SNR and collision evaluation,
- analyzer log parsing and replay timing,
- simulator-side control invocation,
- and UI update semantics.

It does **not** attempt to define:

- undocumented internal intentions,
- a full RF-physics model,
- blockchain-layer consensus behavior,
- malicious-node simulation,
- or any future mode that is not present in the code.

## Core Terminology Used in This Document

To stay aligned with the companion simulator files, this document uses the following terms consistently:

- **Simulation mode** — the synthetic scene-driven in-process multi-node simulation.
- **Real-time Tracking mode** (`RealtimeTracking`) — analyzer mode that tails a live log file and may send Telemetry Hub commands.
- **Log Visualization mode** (`LogVisualization`) — analyzer mode that replays a historical log file from the beginning.
- **Node Task** — the per-node async task wrapping one radio-library instance.
- **Network Task** — the central simulation task responsible for propagation, CAD windows, and packet-delivery decisions.
- **Analyzer Task** — the async task that parses logs, reconstructs state, and drives the UI in analyzer modes.
- **Effective distance** — the deterministic precomputed range estimate used as a receive prefilter and UI range value.
- **Airtime waiting packet** — a queued receive-side candidate packet waiting until its simulated airtime completes and reception is evaluated.
- **Connection matrix parser** — the TM9 log decoder that reconstructs directional link-quality matrices.

## Algorithmic Problem Statement

The current application must solve the following combined problem:

1. start in one operator-selected mode,
2. load scene context and initialize the correct runtime pipeline,
3. either simulate many live nodes or reconstruct their activity from logs,
4. expose a common UI contract for map, metrics, node streams, measurements, and connection-matrix inspection,
5. optionally send remote out-of-band control commands in real-time tracking mode,
6. and maintain deterministic, bounded behavior despite host-side scaling and many concurrent events.

## Section A — Top-Level Runtime Structure

## A1. Main-thread split

The current application runs two main execution domains:

1. the **UI thread** on the main thread using `eframe/egui`,
2. the **Embassy executor thread** for async simulation or analyzer tasks.

## A2. UI–executor channels

The application creates exactly two main bounded channels between UI and async runtime:

- `UIRefreshChannel` with capacity `500`, used runtime → UI,
- `UICommandChannel` with capacity `100`, used UI → runtime.

These channels are intentionally heap-allocated and leaked for `'static` lifetime.

## A3. Mode dispatcher rule

At startup, a mode dispatcher task waits for a `UICommand::StartMode` or legacy `LoadFile` command.

Depending on the selected mode it spawns exactly one of:

- `simulation::network_task` for **Simulation**,
- `analyzer::analyzer_task` for **RealtimeTracking**,
- `analyzer::analyzer_task` for **LogVisualization**.

Once one of these tasks is spawned, the dispatcher returns.

## Section B — Scene File Model

## B1. Shared scene semantics

The current codebase uses scene files in both simulation and analyzer workflows.

The scene file root currently supports these fields:

- `nodes`
- `obstacles`
- `world_top_left`
- `world_bottom_right`
- `width`
- `height`
- optional `background_image`
- optional `link_quality_weak_threshold`
- optional `link_quality_excellent_threshold`

## B2. Simulation-mode scene requirements

In `SceneMode::Simulation`, the scene must also contain:

- `path_loss_parameters`
- `lora_parameters`
- `radio_module_config`

Each node must provide at least:

- `node_id`
- `position`
- `radio_strength`

## B3. Analyzer-mode scene requirements

In `SceneMode::Analyzer`, the physics blocks are optional, but each node must provide:

- `effective_distance`

Analyzer mode uses these effective distances for map visualization rather than recomputing them.

## B4. Validation rules currently enforced

The shared scene loader currently validates at least the following:

- node count must be between 1 and 10000,
- node IDs must be unique,
- node positions must remain within configured world bounds,
- in simulation mode, radio strength must stay within realistic range `[-50, 50] dBm`,
- in analyzer mode, every node must define `effective_distance`,
- spreading factor must be in `5..=12`,
- coding rate must be in `1..=4`,
- bandwidth must be positive,
- path-loss exponent must be positive,
- shadowing sigma must be non-negative,
- optional weak/excellent link-quality thresholds must satisfy `weak < excellent <= 63`,
- rectangle geometry must be valid,
- circle radius must be non-zero and remain within world bounds.

## B5. Precomputed scale rule

When a scene is loaded, the loader computes:

- `scale_x = width / (world_bottom_right.x - world_top_left.x)`
- `scale_y = height / (world_bottom_right.y - world_top_left.y)`

These are later used for world-distance calculations and map rendering.

## Section C — Simulation-Side Runtime Algorithm

## C1. Node initialization algorithm

When the simulation scene is loaded, the network task:

1. publishes link-quality thresholds to the UI using the encoded scoring matrix,
2. computes each node’s effective distance with `calculate_effective_distance(...)`,
3. publishes initial node positions and obstacles to the UI,
4. creates one `NodeInputQueue` per node,
5. spawns one `node_task` per node,
6. stores node runtime state in a `HashMap<u32, Node>`.

## C2. Node task structure

Each node task runs a three-way async select among:

- `manager.receive_message()` from `moonblokz-radio-lib`,
- `NodeInputQueueReceiver.receive()` for commands from the network task,
- simulator radio-device output queue events.

## C3. Node input commands currently supported

The current `NodeInputMessage` set is:

- `RadioTransfer(ReceivedPacket)`
- `SendMessage(RadioMessage)`
- `CADResponse(bool)`
- `RequestConnectionMatrix`

## C4. Node output events currently supported

The current `NodeOutputPayload` set is:

- `RadioTransfer(RadioPacket)`
- `MessageReceived(RadioMessage)`
- `RequestCAD`
- `NodeReachedInMeasurement(u32)`
- `FullMessageReceived { ... }`
- `FullMessageSent { ... }`

## C5. Node-level AddBlock handling rule

The node task maintains an `arrived_messages` map keyed by block sequence.

For incoming `AddBlock` messages:

1. extract sequence from bytes `5..9`,
2. if already present, treat as duplicate and stop,
3. otherwise store the message,
4. emit `NodeReachedInMeasurement(sequence)`,
5. emit `FullMessageReceived { message_type: AddBlock, ... }`,
6. report `MessageProcessingResult::NewBlockAdded` back to the radio manager.

For outgoing `AddBlock` sends:

1. extract sequence,
2. store it in `arrived_messages`,
3. emit `FullMessageSent { ... }`,
4. forward the message into the radio manager.

## C6. RequestBlockPart response rule

When a node receives `RequestBlockPart`:

1. extract requested sequence,
2. look up the stored AddBlock message,
3. decode requested packet indices from the iterator,
4. build a packet mask,
5. clone the original block message and add the requested packet list,
6. report `RequestedBlockPartsFound(response_message, requestor_node)` to the radio manager.

## C7. Network task main loop

The simulation `network_task` repeatedly:

1. computes the next interesting event time from all airtime and CAD windows,
2. computes a 10 ms UI-responsiveness tick deadline,
3. waits on a three-way select among:
   - next node output event,
   - next UI command,
   - timer at the earlier deadline,
4. processes exactly one branch per iteration.

## Section D — Propagation and Reception Algorithm

## D1. RadioTransfer handling rule

When a node emits `NodeOutputPayload::RadioTransfer(packet)`, the network task:

1. logs the packet into the sender node’s radio history,
2. creates a self-airtime entry marked `processed = true`,
3. increments total sent packet count,
4. notifies the UI about the transmission,
5. finds all target nodes within effective distance and with no blocking obstacle,
6. queues one `AirtimeWaitingPacket` per target node.

## D2. Target-node prefilter rule

A receiver candidate is considered only if:

- it is not the sender itself,
- squared scaled distance is below the sender’s cached effective-distance square,
- and `is_intersect(...)` reports no obstacle intersection.

## D3. RSSI calculation rule

For each target receiver, the network task computes:

- scaled physical distance from world coordinates,
- path loss via `calculate_path_loss(distance, params)`,
- RSSI via `tx_power - path_loss`.

Because shadowing is sampled inside `calculate_path_loss`, repeated packets over the same path may receive different sampled RSSI values.

## D4. Airtime calculation rule

The simulator uses a LoRa-style airtime calculation based on:

- bandwidth,
- spreading factor,
- coding rate,
- preamble symbols,
- CRC flag,
- low-data-rate optimization flag,
- payload size.

Payloads above 255 bytes are clamped to 255 for airtime estimation.

## D5. CAD request rule

When a node emits `RequestCAD`, the network task records a `CadItem` with:

- `start_time = now`,
- `end_time = now + get_cad_time(lora_parameters)`.

## D6. CAD completion rule

When CAD items expire, the network task declares channel activity if any airtime packet overlaps the CAD window and sends `NodeInputMessage::CADResponse(activity)` to the node.

## D7. Packet reception timing rule

A queued `AirtimeWaitingPacket` is evaluated only when its airtime window has ended.

The network task processes at most one unprocessed packet per node per event cycle to preserve reasonable ordering and bounded work.

## D8. Noise accumulation rule

For one candidate packet, the network task starts with baseline noise:

- `sum_noise = dbm_to_mw(noise_floor)`

Then for each overlapping packet it adds:

- `dbm_to_mw(other_packet.rssi)`

Finally:

- `total_noise = mw_to_dbm(sum_noise)`
- `sinr = packet_rssi - total_noise`

## D9. SNR limit rule

The decode threshold currently depends only on spreading factor:

- SF5 → `-2.5 dB`
- SF6 → `-5.0 dB`
- SF7 → `-7.5 dB`
- SF8 → `-10.0 dB`
- SF9 → `-12.5 dB`
- SF10 → `-15.0 dB`
- SF11 → `-17.5 dB`
- SF12 → `-20.0 dB`

## D10. Collision and capture rule

The simulator currently uses two simplified destructive-collision rules.

### Earlier-packet dominance rule

If an overlapping earlier packet exists and its RSSI is above the SNR limit, the later packet is considered destructively collided.

### Capture-threshold rule

If the currently evaluated packet is stronger than a later overlapping packet by more than `CAPTURE_THRESHOLD = 6 dB`, the code marks a destructive collision condition.

The current code should therefore be read as implementing a simplified capture/collision heuristic rather than a fully realistic modem model.

## D11. Successful reception rule

A packet is delivered to the receiver node only if:

- `sinr >= snr_limit`,
- and no destructive collision was detected.

On success the network task:

1. computes link quality via `moonblokz_radio_lib::calculate_link_quality(packet_rssi, sinr)`,
2. sends `NodeInputMessage::RadioTransfer(ReceivedPacket { packet, link_quality })`,
3. records a non-collision `NodeMessage`,
4. increments total received packet count.

## D12. Collision-record rule

If the packet was involved in collision and failed reception, the network task:

1. records a collision `NodeMessage`,
2. increments total collision count,
3. does not deliver the packet to the node.

## Section E — Time Model

## E1. Global scaled-time driver

The application registers a custom Embassy time driver using `time_driver_impl!`.

This driver maps host time to virtual time with a Q32.32 fixed-point speed factor.

## E2. Public speed interface

The UI-facing speed functions are:

- `set_simulation_speed_percent(percent)`
- `get_simulation_speed_percent()`

Allowed values are clamped to `1..=1000`.

## E3. Continuity-preserving speed-change rule

When simulation speed changes:

1. the code computes current virtual time under the old mapping,
2. updates only the real-time origin while keeping virtual origin fixed,
3. changes the scale factor,
4. bumps a scheduler epoch counter,
5. wakes the scheduler thread.

This preserves virtual-time continuity and prevents already-scheduled deadlines from wrapping into the past.

## E4. Scheduler wake rule

The scheduler thread:

- tracks the earliest queued virtual wakeup,
- maps it back to real time,
- slices waits to at most 25 ms,
- and re-evaluates promptly when epoch changes occur.

## E5. Auto-speed rule in simulation mode

If auto-speed is enabled, the network task measures delay relative to the next scheduled event.

Current heuristic:

- if delay is under 8 ms repeatedly, increase speed slowly,
- if delay exceeds 8 ms, decrease speed,
- do not go below 20%,
- do not exceed 1000%.

## Section F — Analyzer Algorithm

## F1. Scene and control initialization rule

When analyzer mode starts, the analyzer task:

1. loads the scene in `SceneMode::Analyzer`,
2. builds a `node_id -> effective_distance` map,
3. if in Real-time Tracking mode (`RealtimeTracking`), looks for `config.toml` next to the selected scene,
4. if config exists and loads successfully, creates a `TelemetryClient`,
5. publishes control availability to the UI,
6. initializes UI nodes, obstacles, dimensions, and mode.

## F2. LogLoader behavior

The analyzer uses `LogLoader`.

### Real-time Tracking mode

- opens the log file,
- seeks to the end,
- polls every 50 ms for new lines.

### Log Visualization mode

- opens the log file,
- reads from the beginning,
- returns EOF when finished.

## F3. Main analyzer select rule

The analyzer main loop waits on a two-way select between:

- next log line,
- next UI command.

If a log line must be delayed to preserve playback timing, it enters a second select between:

- `Timer::after(remaining_wait)`,
- next UI command.

## F4. Raw-log capture rule

For every log line that contains a `[node_id]` pattern, the analyzer attempts to capture:

- timestamp,
- node ID,
- raw content after the bracket,
- log level.

These are stored in per-node raw log history for the Log Stream tab.

## F5. Structured TM-event parsing rule

The current analyzer parses these structured telemetry events:

- `TM1` — packet transmitted,
- `TM2` — packet received,
- `TM3` — start measurement,
- `TM4` — full message received,
- `TM5` — packet CRC mismatch,
- `TM6` — AddBlock fully received,
- `TM7` — AddBlock sent,
- `TM8` — version information.

It also parses `TM9` through the shared `ConnectionMatrixParser` from raw log content.

## F6. Delay-tracking rule

The analyzer maintains a sliding window of the last 100 delay samples.

For each parsed log event after the first one:

1. compute time difference between current and previous log timestamps,
2. compute real elapsed wall-clock time since the previous processed event,
3. derive current delay,
4. update average delay,
5. if average delay is lower than current delay, reduce the next wait to 90% to catch up gradually.

This heuristic applies only to playback timing decisions, not to simulation-mode virtual time.

## F7. Event-to-UI mapping rule

### `SendPacket`

- store packet record,
- increment total sent count,
- emit `NodeSentRadioMessage(node_id, message_type, effective_distance)`,
- emit `RadioMessagesCountUpdated(...)`.

### `ReceivePacket`

- store packet record,
- increment total received count,
- emit `RadioMessagesCountUpdated(...)`.

### `StartMeasurement`

- set active measurement ID.

### `AddBlockReceived`

- if sequence matches active measurement, emit `NodeReachedInMeasurement(node_id, sequence)`,
- store `FullMessage { is_outgoing: false }`.

### `AddBlockSent`

- store `FullMessage { is_outgoing: true }`.

### `VersionInfo`

- store `(probe_version, node_version)` per node.

### `PacketCrcError`

- store packet history entry marked later as collision-like in the UI.

## F8. EOF behavior in log visualization

When EOF is reached in Log Visualization mode (`LogVisualization`):

1. emit `UIRefreshState::VisualizationEnded`,
2. stop log replay,
3. continue responding to UI commands indefinitely.

## Section G — Control Algorithm

## G1. Config discovery rule

In real-time tracking mode, control configuration is derived from the selected scene path by replacing the filename with `config.toml` in the same directory.

## G2. Current control config fields

`config.toml` currently provides:

- `api-key`
- `hub-url`

## G3. Current control affordance set

The simulator UI currently exposes typed control actions for:

- update interval changes
- log-level changes
- log-filter changes
- arbitrary node command forwarding
- measurement start

The exact command envelope, HUB validation, queueing semantics, and Probe execution behavior belong in [moonblokz-telemetry-algorythm.md](./moonblokz-telemetry-algorythm.md).

## G4. Transport boundary rule

The simulator currently sends control actions as authenticated HTTP requests to Telemetry Hub using JSON payloads derived from `ControlCommand::to_payload()`.

From the simulator’s perspective, this is simply an explicit out-of-band control path. The broader telemetry contract and fleet-routing semantics are intentionally documented in the telemetry files rather than duplicated here.

## Section H — UI Update Contract

## H1. Node inspector streams

The UI maintains three node-local history streams and one matrix view:

- Radio Stream
- Message Stream
- Log Stream
- Connection Matrix

## H2. Right-panel interaction rule

When a node is selected on the map:

1. the UI sends `RequestNodeInfo(node_id)`,
2. the runtime responds with detailed node-local histories,
3. the right panel renders virtualized tables newest-first.

## H3. Measurement milestones

The UI records milestone times and packet counts automatically when current reached-node percentage crosses:

- 50%
- 90%
- 99.9% (treated as 100%)

## H4. Connection matrix query rule

In Simulation mode or Real-time Tracking mode (`RealtimeTracking`) with control available:

- opening the Connection Matrix tab for a node triggers a request if no matrix is already cached,
- simulation mode requests it directly from the node task,
- analyzer mode requests it by sending `/CM` through Telemetry Hub.

## Main Algorithmic Conclusions

From the perspective of the MoonBlokz knowledge base, the current codebase establishes these formal simulator conclusions:

1. the application is a mode-switched desktop runtime with one UI thread and one Embassy async executor thread,
2. scene files now serve both simulation and analyzer workflows,
3. simulation mode runs many real radio-library instances inside one process through node tasks and a central network task,
4. propagation uses effective-distance prefiltering, obstacle rejection, sampled path loss, LoRa-style airtime, SNR limits, and simplified collision heuristics,
5. replay and real-time tracking are first-class analyzer algorithms, not just future ideas,
6. time scaling is implemented as a continuity-preserving global virtual clock rather than a simple sleep multiplier,
7. connection-matrix reconstruction and visualization are now part of the formal runtime behavior,
8. remote control is explicitly out-of-band and Telemetry-Hub-based,
9. all important histories and queues remain bounded by design.

For the broader field-testing telemetry flow that feeds these analyzer behaviors — including Probe uploads, HUB ordering rules, Collector downloads, CLI command routing, OTA/update paths, and the formal meaning of these control requests — use [moonblokz-telemetry-algorythm.md](./moonblokz-telemetry-algorythm.md) rather than duplicating that architecture here.
