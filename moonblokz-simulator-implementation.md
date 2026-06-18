# MoonBlokz Simulator Implementation Notes

## Purpose of This Document

This document captures implementation-facing implications of the MoonBlokz radio simulator **as currently implemented in `moonblokz-radio-simulator`**.

It complements the conceptual and algorithm simulator documents by identifying:

- what the current crate is responsible for today,
- how the codebase is structured,
- how the simulator reuses embedded radio-library behavior,
- how the application now combines simulation, analyzer, UI, and control modules,
- what bounded-memory and bounded-queue assumptions still shape implementation behavior,
- and which implementation areas remain open, approximate, or evolution-sensitive.

- Use [moonblokz-simulator-concept.md](./moonblokz-simulator-concept.md) for the strategic role, scope, and conceptual meaning of the simulator.
- Use [moonblokz-simulator-algorythm.md](./moonblokz-simulator-algorythm.md) for the formal runtime flow and current behavioral rules.

## Source Basis

This document is grounded primarily in the current `moonblokz-radio-simulator` codebase, especially:

- `Cargo.toml`
- `src/main.rs`
- `src/common/*`
- `src/simulation/*`
- `src/analyzer/*`
- `src/control/*`
- `src/ui/*`
- `src/time_driver.rs`

Where article-era descriptions diverge from the current code, this document treats the code as authoritative.

## Scope and Intent

This is not a line-by-line source tour. Instead, it records the engineering consequences of the current implementation so future work can preserve important invariants and avoid accidental architectural drift.

## Current Crate Structure

The current crate structure is functionally divided into five major areas.

### `src/main.rs`

Owns:

- application startup,
- logger setup,
- UI/simulation channels,
- Embassy executor thread startup,
- mode dispatch,
- and the `eframe` main window.

### `src/common/`

Owns shared non-mode-specific helpers such as:

- scene loading and validation,
- connection-matrix decoding from logs.

### `src/simulation/`

Owns synthetic multi-node simulation, including:

- geometry,
- signal calculations,
- node tasks,
- the central network task,
- runtime packet/event data types,
- log capture of in-process radio-library output.

### `src/analyzer/`

Owns log-driven observation modes, including:

- mode-specific log loading,
- parsing telemetry-tagged events,
- analyzer state,
- replay timing,
- and UI synchronization from log streams.

### `src/control/`

Owns Telemetry Hub integration, including:

- command typing,
- config loading,
- and HTTP transport.

### `src/ui/`

Owns the full egui front end, including:

- mode selection,
- app state,
- top metrics panel,
- right inspector panel,
- map visualization.

### `src/time_driver.rs`

Owns the custom scaled Embassy time driver used throughout the application.

## Dependency and Feature Model

The current `Cargo.toml` shows several important implementation commitments.

### UI stack

- `eframe`
- `egui`
- `egui_extras`
- `image`

### Async/runtime stack

- `embassy-sync`
- `embassy-executor`
- `embassy-time`
- `embassy-futures`
- `embassy-time-driver`

### Radio library dependency

The simulator depends on `moonblokz-radio-lib` with features:

- `radio-device-simulator`
- `memory-config-large`
- `connection-matrix-logging`

This is an important implementation fact because it means the current host-side simulator deliberately uses the simulator backend and the large memory profile of the radio library.

### Auxiliary dependencies

The current code also depends on:

- `rand` and `rand_distr` for simulation randomness,
- `serde`, `serde_json`, and `toml` for scene/config loading,
- `rfd` for native file dialogs,
- `chrono` for timestamp handling,
- `reqwest` for Telemetry Hub HTTP commands.

## Main Process Architecture

The current application runs with a strict thread split.

### Main thread

The UI runs on the main thread because `eframe`/AppKit requirements make this the natural path on macOS.

### Embassy executor thread

All simulation or analyzer async tasks run on a dedicated background thread with a very large stack (`192 MB`).

The current code comments explicitly justify this large stack by the state requirements of many concurrent async tasks and queues.

## Intentional `'static` Lifetime Strategy

The current code repeatedly uses `Box::leak()` to satisfy Embassy’s `'static` lifetime expectations.

This applies to:

- UI channels,
- node input queues,
- node radio-device queues,
- the nodes output queue,
- and the executor itself.

### Engineering meaning

This is not accidental technical debt. It is the explicit bridge that allows the host application to run embedded-style async code without redesigning the library around shorter lifetimes.

### Operational assumption

The implementation assumes the whole process lifetime matches the useful lifetime of these allocations. This is acceptable for a desktop tool but should remain clearly separated from embedded-firmware memory practice.

## Logging and In-Process Telemetry Capture

The current startup path uses a custom `TeeLogger` around `env_logger`.

This allows the application to:

- emit normal logs,
- capture `moonblokz_radio_lib` logs in memory,
- drain them inside the simulation loop,
- and attach them to the corresponding node history.

This is important because the simulator does not rely only on explicit queue events for observability. It also treats log capture as part of runtime introspection.

## Scene Model Implementation Notes

The current codebase has two scene representations:

- `common::scene::Scene` for shared loading/validation across simulation and analyzer modes,
- `simulation::types::Scene` for the simulation runtime.

This is a deliberate separation between:

- deserialized cross-mode configuration,
- and simulation-specific runtime data structures.

### Background image resolution

If a scene references `background_image`, the loader resolves it relative to the scene file directory.

### Scale factor caching

The loader precomputes `scale_x` and `scale_y` immediately after parsing.

This matters because the geometry code expects those cached factors rather than recomputing them repeatedly.

## Simulation Runtime Structure

The simulation runtime centers on two main components:

- the per-node `node_task`,
- the central `network_task`.

### Node-side responsibility

Each node task owns:

- one `RadioCommunicationManager`,
- one simulated radio-device input queue,
- one simulated radio-device output queue,
- one local arrived-message map for AddBlock sequence deduplication and recovery.

### Network-side responsibility

The network task owns:

- scene state,
- all node runtime structs,
- packet distribution,
- CAD scheduling,
- airtime-window evaluation,
- collision and SINR checks,
- measurement progress accounting,
- and UI refresh emission.

This separation is important and should remain intact.

## Queue Topology and Boundedness

The current simulation types define several bounded channels.

### Node input queue

- capacity: `10`

### Global nodes output queue

- capacity: `10`

### UI refresh queue

- capacity: `500`

### UI command queue

- capacity: `100`

The design implication is clear: the simulator still prefers bounded backpressure and dropped/limited work over unbounded growth.

## Per-Node History Retention

The UI-facing histories are explicitly bounded.

### Current limits

- `NODE_MESSAGES_CAPACITY = 1000`
- `NODE_LOG_LINES_CAPACITY = 1000`
- `NODE_FULL_MESSAGES_CAPACITY = 1000`

### Airtime waiting queue

- `MAX_AIRTIME_WAITING_PACKETS = 500`

The airtime queue also includes a warning threshold at 80% capacity and emergency cleanup behavior.

This is an implementation-level continuation of the broader MoonBlokz bounded-runtime philosophy.

## Signal Model Implementation Notes

The current signal model is implemented in `signal_calculations.rs` and is more concrete than the earlier article summary.

### Path loss

Uses log-distance path loss with a shadowing sample drawn from `Normal(0, sigma)`.

### Effective distance

The code computes effective distance by solving for the distance at which received power reaches a simplified receiving limit:

- `receiving_limit = noise_floor + snr_limit`

This makes effective distance a deterministic link-budget estimate rather than a sampled runtime value.

### Airtime

The code implements LoRa-style airtime math including:

- symbol time,
- preamble time,
- payload symbol count,
- CRC and low-data-rate optimization terms.

### CAD time

The current CAD estimate is exactly two symbols.

### Capture threshold

The collision model uses:

- `CAPTURE_THRESHOLD = 6.0 dB`

This should be documented as a simulator heuristic, not as a guaranteed property of real hardware.

## Geometry Implementation Notes

The geometry module intentionally uses conservative and robust primitive checks.

### Distance

The hot path uses squared distance in scaled physical meters.

### Obstacle support

The current obstacle set is:

- axis-aligned rectangles,
- circles.

### Segment logic

The implementation includes:

- point-in-shape checks,
- segment–rectangle checks,
- segment–circle checks,
- orientation-based segment intersection with collinear overlap handling,
- degenerate-segment handling.

This is more careful than a naive “rough overlap” approximation and should remain so.

## Analyzer Implementation Notes

The analyzer subsystem is now a major part of the crate and deserves explicit implementation treatment.

### Desktop-facing responsibility

The simulator owns the analyzer as a desktop consumer of telemetry logs. Its primary responsibility is to turn log input into UI-visible node histories, measurements, and map overlays.

### LogLoader

Implements two file-reading behaviors:

- tail-follow for Real-time Tracking mode,
- start-to-end replay for Log Visualization mode.

### Parser layering

The analyzer intentionally has two parsing layers:

1. raw log capture for any line containing `[node_id]`,
2. structured TM-event parsing for telemetry-tagged semantic events.

This split is important because the UI exposes both:

- structured packet/message history,
- and raw log stream history.

### Analyzer state

The analyzer uses per-node bounded `VecDeque` histories and keeps:

- packet histories,
- raw log histories,
- full-message histories,
- version info from TM8,
- active measurement ID,
- last processed timestamp.

### Delay recovery

Replay timing uses a sliding delay average and a simple catch-up heuristic rather than an exact PLL-style scheduler. This is a practical choice, not a mathematically perfect one.

For the broader telemetry-side meaning of TM-tagged logs, delayed HUB downloads, Collector extraction, and hub-mediated command routing, use the telemetry document set rather than extending this simulator file into a second telemetry architecture document.

## Connection Matrix Support

The current code makes connection-matrix visibility an implementation feature rather than a loose idea.

### In simulation mode

The UI can request a matrix dump directly from a node.

### In analyzer mode

The shared `ConnectionMatrixParser` reconstructs matrices from TM9 log sequences.

### Encoding

Matrix rows are encoded using a compact 64-symbol alphabet:

- `A-Z` → `0..25`
- `a-z` → `26..51`
- `0-9` → `52..61`
- `-` → `62`
- `_` → `63`

This is important implementation knowledge because it is now part of the effective diagnostic interface.

## Control Subsystem Notes

The current control subsystem is intentionally minimal and explicit.

### Simulator-side configuration source

A `config.toml` file is searched for next to the selected scene file.

### Simulator-side config fields

- `api-key`
- `hub-url`

### Transport

The implementation uses synchronous `reqwest::blocking::Client` with a 30-second timeout.

This is notable because control commands are not routed through the async Embassy runtime itself; they are executed as blocking HTTP operations inside the analyzer command handler path.

### Boundary meaning

This module exists to let the desktop tool invoke hub-mediated control. The simulator owns the local config discovery and blocking HTTP call pattern. The telemetry documents own the end-to-end semantics of `/command`, queue expansion, Probe delivery, and command compatibility across repositories.

## UI Implementation Notes

The current UI remains immediate-mode egui and provides the operator surface used by all current application modes.

### Mode selector

The app starts in a mode selector rather than directly in simulation.

### AppState

`AppState` is the central UI-side store and now holds:

- mode-selection state,
- scene/map state,
- node inspector state,
- measurement state,
- simulation/analyzer timing state,
- connection-matrix cache,
- background image state,
- control modal state,
- persisted last-open-directory paths for each picker.

### Persistence

The app persists recent directories and right-panel width through `eframe` storage.

### Inspector tabs

The right panel currently supports:

- Radio Stream,
- Message Stream,
- Log Stream,
- Connection Matrix.

### Table virtualization

The UI consistently uses `egui_extras::TableBuilder` to keep long histories responsive.

### Background imagery

The map can load an optional background image as an egui texture and render it under nodes and obstacles.

## Time Driver Engineering Notes

The current `time_driver.rs` is more sophisticated than the article-level description.

### Fixed-point scaling

Speed is represented with Q32.32 fixed-point math rather than floating point.

### Scheduler thread

A dedicated scheduler thread handles Embassy wakeups.

### Lock-ordering discipline

The file explicitly documents a critical lock-ordering rule:

- always acquire `CLOCK` before `SCHED`

This is a real implementation invariant, not only a comment-level preference.

### Test coverage

The time driver has unit tests for:

- continuity across speed changes,
- inverse mapping correctness,
- safe handling of past targets.

## Testing Notes from the Current Codebase

The codebase includes unit tests at least for:

- signal calculations,
- geometry,
- log parsing,
- time-driver correctness,
- connection-matrix decoding.

That means the simulator crate already treats math, parsing, and timing as test-worthy subsystems rather than relying only on manual visual inspection.

## Important Current Limitations and Cautions

The current implementation still leaves several things open or intentionally approximate.

### 1. Simulation and analyzer share one UI but not one runtime model

The product surface is unified, but the underlying execution models remain distinct.

### 2. Collision/capture logic is simplified

The code uses a practical heuristic, not a detailed modem-accurate receive model.

### 3. Control transport is blocking HTTP

This is operationally acceptable for the current desktop tool, but it is a design choice that should stay explicit.

### 4. Analyzer parsing is log-format dependent

Changes to telemetry-tagged log formats can break reconstruction behavior.

### 5. Heavy use of leaked allocations is intentional but environment-specific

This is appropriate for the desktop simulator, not as a general pattern for firmware.

For the broader implementation split around Probe, Telemetry HUB, Update Server, Log Collector, and CLI — which surrounds the analyzer functions documented here — use [moonblokz-telemetry-implementation.md](./moonblokz-telemetry-implementation.md) rather than duplicating those operational details in this simulator file.

## What Must Remain Open

Even with the codebase as source of truth, several implementation areas remain intentionally open or evolution-sensitive:

- richer RF or obstacle models,
- future analyzer event types beyond the current TM set,
- future control commands and config schema evolution,
- possible non-blocking Telemetry Hub transport in the future,
- any future unification of simulation and analyzer runtimes,
- and any later move toward 3D or richer map semantics.

## Related Documents

- [`moonblokz-simulator-concept.md`](./moonblokz-simulator-concept.md) — conceptual role and scope of the simulator.
- [`moonblokz-simulator-algorythm.md`](./moonblokz-simulator-algorythm.md) — formal simulation, analyzer, and timing behavior implemented here.
- [`moonblokz-radio-implementation.md`](./moonblokz-radio-implementation.md) — the real radio library reused inside the simulator.
- [`moonblokz-telemetry-implementation.md`](./moonblokz-telemetry-implementation.md) — the telemetry stack the analyzer mode consumes.
