# MoonBlokz Telemetry Concept Model

## Purpose of This Document

This document explains the conceptual operating model of the MoonBlokz field-testing telemetry system, with special attention to the currently implemented Probe, Telemetry HUB, Log Collector, Telemetry CLI, and Update Server behavior visible in the `moonblokz-probe`, `moonblokz-telemetry-hub`, `moonblokz-log-collector`, `moonblokz-telemetry-cli`, and `moonblokz-telemetry-update-server` repositories.

Its purpose is to explain:

- why MoonBlokz needs a telemetry subsystem that is separate from the LoRa mesh itself,
- what operational problems the field-testing infrastructure is meant to solve,
- how the Test Station, Probe, Telemetry HUB, Update Server, Log Collector, CLI, and Analyzer fit together,
- why logging, command execution, and OTA updates are treated as one integrated operational concern,
- and how the currently implemented Probe, HUB, Collector, CLI, and Update Server sharpen the practical meaning of those architectural boundaries.

This file is intentionally conceptual. It focuses on goals, roles, boundaries, and architectural meaning rather than exact endpoint schemas, command payloads, or line-by-line implementation detail.

- Use [moonblokz-telemetry-algorythm.md](./moonblokz-telemetry-algorythm.md) for the formal end-to-end flow of logs, commands, polling, cleanup, updates, and analysis.
- Use [moonblokz-telemetry-implementation.md](./moonblokz-telemetry-implementation.md) for implementation-facing notes, repository roles, current module structure, and engineering cautions.

## Source Basis

This document is grounded in five complementary source groups.

### Primary implementation source for Probe behavior

- `moonblokz-probe/src/main.rs`
- `moonblokz-probe/src/config.rs`
- `moonblokz-probe/src/log_entry.rs`
- `moonblokz-probe/src/usb_manager.rs`
- `moonblokz-probe/src/usb_collector.rs`
- `moonblokz-probe/src/telemetry_sync.rs`
- `moonblokz-probe/src/command_executor.rs`
- `moonblokz-probe/src/update_manager.rs`
- `moonblokz-probe/README.md`
- `moonblokz-probe/config.toml.example`
- `moonblokz-probe/moonblokz-probe.service`
- `moonblokz-probe/quick_start.sh`

### Primary implementation source for HUB behavior

- `moonblokz-telemetry-hub/src/lib.rs`
- `moonblokz-telemetry-hub/spin.toml`
- `moonblokz-telemetry-hub/API.md`
- `moonblokz-telemetry-hub/README.md`
- `moonblokz-telemetry-hub/IMPLEMENTATION.md`
- `moonblokz-telemetry-hub/QUICKSTART.md`
- `moonblokz-telemetry-hub/test_hub.sh`

### Primary implementation source for Log Collector behavior

- `moonblokz-log-collector/src/main.rs`
- `moonblokz-log-collector/src/config.rs`
- `moonblokz-log-collector/README.md`
- `moonblokz-log-collector/IMPLEMENTATION.md`
- `moonblokz-log-collector/config.toml.example`

### Primary implementation source for Telemetry CLI behavior

- `moonblokz-telemetry-cli/src/main.rs`
- `moonblokz-telemetry-cli/src/parser.rs`
- `moonblokz-telemetry-cli/src/client.rs`
- `moonblokz-telemetry-cli/src/config.rs`
- `moonblokz-telemetry-cli/Cargo.toml`
- `moonblokz-telemetry-cli/README.md`
- `moonblokz-telemetry-cli/DEVELOPER.md`
- `moonblokz-telemetry-cli/EXAMPLES.md`
- `moonblokz-telemetry-cli/CHANGELOG.md`
- `moonblokz-telemetry-cli/PROJECT_SUMMARY.md`

### Primary implementation source for Update Server behavior

- `moonblokz-telemetry-update-server/spin.toml`
- `moonblokz-telemetry-update-server/update-probe.sh`
- `moonblokz-telemetry-update-server/update-node.sh`
- `moonblokz-telemetry-update-server/assets/setup.sh`
- `moonblokz-telemetry-update-server/assets/probe/version.json`
- `moonblokz-telemetry-update-server/assets/node/version.json`

### Broader system-context source

- **MoonBlokz series part VII/5 — Field Testing Infrastructure** (Medium, Jan 25, 2026)

This means the telemetry document set is now **Probe-, HUB-, Collector-, CLI-, and Update-Server-code-grounded** where those five components are concerned, while remaining article-grounded for the wider multi-repository telemetry architecture that has not yet been revalidated end-to-end here.

## Relationship to the Existing MoonBlokz Knowledge Base

This document should be read after the radio and simulator documents.

Those documents explain:

- how MoonBlokz communicates over constrained LoRa links,
- how the radio behavior is simulated and observed,
- and how analyzer workflows fit into the current simulator application.

This telemetry document explains a different question:

**How does MoonBlokz observe, control, update, and operate a deployed field-test network without consuming the same constrained LoRa bandwidth that the experiment is trying to measure?**

That distinction remains central in the current codebase. The Probe does not try to extend LoRa into a management channel. The HUB does not try to become a LoRa protocol participant. The Collector does not try to become a second analytics backend. The CLI does not try to talk directly to devices. Instead, the system keeps operational traffic in a separate USB + WiFi/cloud path and treats the CLI and Collector as thin operator-side tools around the HUB.

## Core Terminology Used Across the Telemetry Documents

To keep the telemetry knowledge-base files aligned, this terminology is used consistently throughout:

- **Test Station** — the deployed field-testing unit combining a LoRa node and a WiFi-capable Probe host.
- **Node** — the RP2040-based MoonBlokz LoRa participant running the radio/blockchain firmware.
- **Probe** — the Raspberry Pi-side operational bridge between the node and the cloud telemetry infrastructure.
- **Telemetry HUB** — the cloud coordination component that aggregates logs, stores commands, calculates the effective upload interval, and serves ordered downloads.
- **Update Server** — the static artifact host for Probe and Node update binaries plus version metadata.
- **Log Collector** — the local component that incrementally downloads delayed HUB logs into a local append-only file using an in-memory timestamp cursor.
- **CLI** — the operator-facing command tool that parses a small command language locally and submits validated control requests to the HUB `/command` endpoint.
- **Analyzer** — the visualization and live/offline analysis environment, currently integrated into `moonblokz-radio-simulator`.
- **Out-of-band telemetry network** — the WiFi/cloud control path that is deliberately kept separate from the LoRa mesh.
- **TM marker** — the lightweight telemetry tag embedded in log lines, such as `TM1`, `TM2`, and `TM8`.
- **Update interval config** — the HUB-side globally stored active/inactive upload schedule, from which a single current `update_interval` value is derived.
- **Collector cursor** — the Collector’s current `last_log_timestamp` value held only in memory during one process lifetime.
- **CLI command envelope** — the JSON object sent by the CLI to the HUB containing `command` plus `parameters`.

## Business Analyst View: What Problem the Telemetry System Solves

Part VII/5 makes one practical point unavoidable: simulation is necessary, but it is not sufficient.

A MoonBlokz radio network can only be trusted after it is exercised in real physical environments with real distance, rooftops, interference, weather exposure, and deployment inconvenience. Once nodes are placed on roofs or other difficult-to-access locations, four operational needs become inseparable:

- **observability**, so the operator can understand what the network is doing,
- **controllability**, so targeted experiments can be initiated remotely,
- **updatability**, so the deployed software can evolve without manual retrieval of hardware,
- and **operability**, so humans have workable local tools for issuing commands and extracting logs.

The telemetry system exists to satisfy these needs simultaneously.

The current Probe, HUB, Collector, CLI, and Update Server code make this more concrete than the article alone did:

- the Probe continuously ingests logs from USB,
- the Probe keeps a bounded in-memory delivery buffer and a separate persistent local node log file,
- the Probe uploads telemetry batches even when there are no buffered logs so it can still receive commands and report version state,
- the HUB persists logs and commands explicitly,
- the HUB computes the currently effective upload interval from a stored active/inactive schedule,
- the HUB serves delayed downloads so collectors see a more stable global ordering,
- the Collector keeps a simple local append-only extraction workflow rather than trying to mirror the HUB database or maintain durable sync state,
- and the CLI gives the operator a small command language that validates some inputs locally before handing control requests to the HUB.

Conceptually, the telemetry system is therefore not just a monitoring sidecar. It is the operational runtime and coordination layer that makes a deployed MoonBlokz node fleet usable at all.

## Why Telemetry Must Be Out-of-Band

The central architectural constraint remains unchanged: the telemetry and control path must not interfere with the LoRa mesh that is being measured.

If the system used LoRa itself for:

- verbose logging,
- operator commands,
- or firmware distribution,

then the measurement environment would be contaminated by the management traffic.

MoonBlokz therefore treats telemetry as a **parallel operational network** rather than as another message family inside the mesh protocol. The LoRa network is reserved for normal MoonBlokz node behavior and deliberate test traffic. Logging, command dispatch, and OTA updates travel through USB, WiFi, and cloud services instead.

The current implementation reinforces this boundary strongly:

- LoRa behavior stays inside the node firmware,
- the Probe handles USB collection, HTTPS upload, update checks, reboot logic, and local persistence,
- the HUB handles log persistence, command queueing, interval policy, and delayed log serving,
- the Collector simply pulls already-prepared delayed logs into a local file for later use,
- and the CLI simply submits command requests to the HUB instead of reaching devices directly.

## The Test Station as the Foundational Telemetry Concept

The most important physical concept remains the **Test Station**.

A Test Station is a hybrid device with two communication domains:

- a **LoRa side**, where the embedded node participates in the MoonBlokz network,
- and a **WiFi/cloud side**, where the Probe communicates with the telemetry infrastructure.

This dual-role design preserves experimental integrity while still making the deployed node operationally manageable.

The current Probe code sharpens the meaning of this split:

- the node is treated as a serially connected peer reached through one owned USB manager,
- the Probe is treated as the Linux-side orchestrator that can survive network outages, retry cloud sync, and perform privileged OS operations.

The current HUB code sharpens the cloud side of the same split:

- the cloud does not push commands directly to devices,
- it queues commands for later pickup,
- it stores global interval policy centrally,
- and it serves log data conservatively enough to reduce cross-Probe ordering distortion.

The current CLI and Collector code sharpen the operator side of the same split:

- the CLI is where human-friendly command syntax is turned into a hub-compatible JSON request,
- the Collector is where delayed HUB-served logs become a durable local text log,
- and neither tool tries to bypass the HUB’s control and timing rules.

## Logging, Commanding, and Updating Are One Operational Problem

The article rejected the idea that telemetry is only about observability, and the Probe + HUB + CLI implementation confirms that judgment.

In the current codebase, the system as a whole is responsible for one integrated operating loop that includes:

- collecting logs from the node,
- filtering which logs are uploaded,
- persisting complete local Probe-side logs,
- polling the HUB,
- queueing and executing incoming commands,
- shaping operator commands into the HUB request format,
- storing pending commands centrally,
- calculating the current upload interval centrally,
- checking for Node updates,
- and checking for Probe updates.

The Collector adds one more pragmatic piece to this operating loop:

- it continuously turns the HUB’s delayed log stream into a local append-only file without introducing extra command or storage semantics.

This means MoonBlokz still does not separate “monitoring” from “fleet operations.” In the telemetry runtime, they are one combined responsibility, with the Collector intentionally staying at the simple extraction edge and the CLI intentionally staying at the simple command-submission edge.

## Heterogeneity as a First-Class Design Constraint

The telemetry architecture spans multiple execution environments with radically different constraints:

- RP2040 microcontroller firmware,
- Raspberry Pi / Pi Zero Linux systems,
- cloud-hosted WASI services,
- local CLI tools,
- and local desktop analysis tools.

The current code shows how seriously this heterogeneity is taken.

The Probe lives in a middle layer that must deal with:

- serial-device behavior,
- Linux filesystems,
- privileged reboot and mount operations,
- and HTTPS communication.

The HUB lives in a very different execution model:

- one Spin/WASI component,
- explicit SQLite and key-value state,
- no outbound network dependencies,
- request-scoped processing instead of long-running device loops.

The Collector lives in yet another model:

- one local Tokio process,
- no persistent sync database,
- one append-only output file,
- one in-memory timestamp cursor,
- and a direct dependence on the HUB’s response contract.

The CLI lives in a fourth operational model:

- one local Tokio process,
- one `reqwest` client,
- one TOML config file,
- one local parser that validates some command syntax before network submission,
- and no direct knowledge of whether a target node actually exists or whether a queued command will later be meaningful to the Probe.

Conceptually, the telemetry architecture succeeds by giving each layer only the kind of state and responsibilities that fit its runtime environment.

## Architect View: The Telemetry System as Three Operational Environments

The telemetry system still spans three major environments.

### 1. Embedded Test Station environment

This is where the node and Probe live together physically.

The node handles LoRa participation. The Probe handles:

- USB communication with the node,
- log capture,
- local log persistence,
- command forwarding,
- cloud synchronization,
- and unattended software updates.

### 2. Cloud coordination environment

This is where the Telemetry HUB and Update Server live.

The HUB is the operational coordination layer. The Update Server is the artifact-distribution layer. They remain intentionally separated so that:

- the HUB can stay focused on control, state, and log aggregation,
- while the Update Server remains a minimal static host.

The current HUB code also makes an important refinement explicit:

- the HUB is not only a queue and storage layer,
- it is also the place where global upload schedule policy is interpreted into the single current `update_interval` returned to Probes and collectors.

The current Update Server repository sharpens the artifact-hosting side of the same environment in a different way:

- it is currently a Spin static-files deployment rather than a custom application service,
- it serves prebuilt Probe and Node artifacts from fixed static paths under `/static/probe` and `/static/node`,
- and it also publishes an installation/setup script for Probe hosts under `/static/setup.sh`.

### 3. Local analysis and operations environment

This is where the Log Collector, CLI, and Analyzer live.

Conceptually, this is the operator’s working environment. It allows:

- collecting logs locally,
- issuing commands,
- observing the network live,
- replaying historical sessions,
- and visualizing topology and propagation on a map.

The current Collector code makes this environment more specific:

- the local extraction tool is intentionally lightweight,
- it does not persist its cursor across restarts,
- and it relies on the HUB’s delayed-serving model plus returned `update_interval` for pacing.

The current CLI code makes the same environment more specific from the control side:

- the operator can work interactively or in single-command mode,
- command parsing is local but intentionally narrow,
- and successful command submission means the HUB accepted the request, not that a Probe has already executed it.

## The Probe as the Bridge Between Worlds

The Probe is the conceptual center of the deployed-side architecture.

It bridges:

- a constrained embedded node connected over USB,
- and a richer IP-based telemetry network connected over WiFi.

The current code makes it clear that the Probe is not just a passive forwarder.

It is a long-running operational daemon that owns five concurrent concerns:

- USB connection management,
- USB log collection,
- telemetry synchronization,
- node update management,
- probe self-update management.

Conceptually, this matters because the Probe is not only the “bridge” between worlds. It is the **runtime coordinator** that keeps those worlds aligned on the station side.

## The Probe’s Two Log Paths Are Conceptually Important

One particularly important clarification visible in the code is that the Probe maintains **two different log uses**.

### 1. Local full log persistence

Every received USB line is written to `/home/moonblokz/node.log` with a UTC timestamp.

### 2. Filtered upload buffer

Only lines that match the current runtime filter are added to the in-memory upload buffer.

This means the Probe conceptually distinguishes between:

- **complete local forensic retention**, and
- **selective central telemetry upload**.

That distinction was only implicit in the article, but it is explicit in the implementation and materially improves the operational model.

## The Telemetry HUB as Coordination, Delay, and Policy

The article described the Telemetry HUB as the central coordination point, not as a complex analytics engine. The current code makes that role more specific.

Conceptually, the HUB is currently responsible for five kinds of coordination:

1. **Probe coordination** — accept uploads and return pending commands plus the current `update_interval`.
2. **Collector coordination** — serve logs older than a safety cutoff, plus the same current `update_interval`.
3. **CLI coordination** — accept operator command requests at `/command` and either queue them or special-case global schedule policy updates.
4. **Command coordination** — queue commands by node and broadcast them by expanding to all known node IDs.
5. **Policy coordination** — store one global active/inactive interval configuration in key-value storage and derive the current effective upload interval from it.

This makes the HUB more than a passive mailbox, but still much smaller than an analytics or orchestration platform.

## The CLI as a Deliberately Thin Command-Shaping Layer

The current CLI code clarifies another important architectural boundary.

The CLI is **not**:

- a direct device-control channel,
- a fleet-state database,
- a workflow engine,
- or an authority on which commands are ultimately meaningful on the Probe.

Instead, it is a thin operator-side command-shaping layer that:

- loads a local API key and HUB URL,
- parses a small command language,
- validates some parameter types locally,
- converts accepted commands into a HUB-compatible JSON envelope,
- and submits that envelope to `/command`.

Conceptually, this keeps the CLI simple, scriptable, and hub-centric.

It also means the CLI’s notion of a “successful” command is limited: `OK` means the HUB accepted the request, not that a Probe has already received or executed it.

## The Collector as a Deliberately Thin Local Projection

The current Collector code clarifies an important architectural boundary.

The Collector is **not**:

- a second telemetry database,
- a command participant,
- a replay engine,
- or a stateful synchronization service.

Instead, it is a thin local projection layer that:

- polls the HUB,
- follows the HUB’s current pacing via `update_interval`,
- appends downloaded logs to a local file,
- and advances only an in-memory timestamp cursor.

Conceptually, this keeps the Collector simple and robust, but also means restart behavior is intentionally lossy in terms of cursor state: a restart re-begins from the epoch unless the operator manages output files separately.

## Operational Liveness Is More Important Than Log Volume

The current Probe code reveals an important conceptual property:

telemetry uploads happen even when the filtered log buffer is empty.

That is because the upload is not only a log-transfer operation. It is also the mechanism for:

- receiving pending commands,
- reporting Probe and Node version information,
- and staying synchronized with the HUB-controlled upload timing.

This means the upload cycle is conceptually a **heartbeat-plus-delivery** mechanism, not merely a batch shipper for logs.

The HUB code complements this conceptual model by always returning the current effective interval together with command data.

The Collector code completes it by actually honoring the returned `update_interval` from `/download`, rather than relying only on a local static polling period.

The CLI completes the control side by ensuring operator requests are always hub-mediated rather than trying to short-circuit the heartbeat model.

## Version Reporting Is Part of Telemetry Semantics

The current telemetry sync code adds a synthetic telemetry line on every upload:

- a `TM8` log entry containing `probe_version` and `node_version`.

That makes version visibility part of the normal telemetry model rather than only part of update diagnostics.

Conceptually, the Probe is therefore not merely shipping node-originated lines. It is also publishing Probe-side operational state into the shared telemetry stream.

## The HUB’s Delayed Download Rule Is a Conceptual Invariant

The current HUB code makes one important conceptual point concrete:

logs are not served to collectors immediately after insertion.

Instead, the HUB computes a cutoff time based on the current effective upload interval with a 1.1 safety multiplier and only serves logs older than that cutoff.

Conceptually, this means the HUB prioritizes:

- a more stable cross-Probe time ordering,
- over immediate visibility of the newest lines.

This is one of the most important architectural properties of the current telemetry system because it directly shapes what downstream collectors and analyzers can assume.

## The Collector’s Cursor Model Is Also a Conceptual Invariant

The current Collector implementation makes another important point explicit:

its progress cursor is based on the newest successfully parsed **timestamp**, not on item IDs, and that cursor lives only in process memory.

Conceptually, this means:

- the Collector is optimized for simplicity rather than resumable synchronization,
- restart behavior is intentionally coarse,
- and timestamp parseability is part of effective interoperability between HUB and Collector.

The Collector does parse `item_id`, but does not currently use it as the primary synchronization cursor.

## The CLI’s Local Validation Model Is Also a Conceptual Invariant

The current CLI implementation makes a parallel point on the control side:

its local parser validates only a bounded subset of the overall control problem.

Conceptually, this means:

- command syntax and some parameter typing are enforced locally,
- but semantic acceptance still depends on the HUB and later on the Probe,
- broadcast scope is decided by omission of `node_id` for most commands,
- `set_update_interval` is treated as globally targeted from the CLI side,
- and unsupported or mismatched command semantics can still exist beyond the CLI because the HUB performs only narrow validation for non-special commands.

So the CLI improves operator ergonomics, but it does not replace end-to-end command compatibility.

## Update Infrastructure as an Operational Safety Requirement

OTA updating remains essential because the deployed units may be physically difficult to reach.

The architecture therefore includes:

- a Node update path based on BOOTSEL and UF2 copy,
- a Probe self-update path based on executable replacement plus reboot,
- and a static update-artifact host that publishes version metadata plus exactly one current Node artifact and one current Probe artifact.

The current Probe code also shows that the update story includes a recovery concept:

- if USB stays disconnected for five minutes, the USB manager attempts a bootloader recovery flow using the currently deployed firmware.

That means the telemetry system is conceptually not just about applying updates, but also about recovering from update-adjacent failure states.

## Analyzer and Visualization as Part of Telemetry, Not Just Simulator UX

Live tracking and offline playback remain part of the telemetry architecture.

The analyzer is conceptually where telemetry data becomes operational understanding.

That includes:

- real-time network tracking,
- offline log playback,
- node-specific logs,
- map-based node placement,
- measurement tracking,
- and connection-matrix visualization.

The simulator documents explain the desktop application surface in detail. The telemetry documents explain **why those capabilities exist** for field operations and how the Probe-side telemetry stream, HUB-side delayed serving, Collector-side local append-only extraction, and CLI-mediated control support them.

## Scene Files and Analyzer Workflows as Operational Telemetry Context

One important boundary refinement is that the simulator should not be read as the primary home of telemetry architecture even though it hosts the current Analyzer UI.

Conceptually:

- the telemetry system explains why live tracking, replay, connection-matrix inspection, and hub-mediated control exist for field operations,
- while the simulator explains how one desktop application renders those workflows.

The current scene files therefore matter to telemetry in a narrow but important way: they provide the static operational context that lets telemetry logs become understandable on a map. They describe where nodes are expected to be, what background image should be used, and what effective-distance or link-quality context the analyzer should apply when interpreting telemetry-derived events.

That means analyzer workflows are now best understood as **telemetry-consumption workflows rendered by the simulator**, not as reasons to duplicate Probe/HUB/Collector/CLI architecture inside the simulator knowledge-base files.

## Important Code-Grounded Scope Boundaries

The `moonblokz-probe`, `moonblokz-telemetry-hub`, `moonblokz-log-collector`, and `moonblokz-telemetry-cli` codebases together make several current limits explicit.

### Upload filtering is substring-based on the Probe

The Probe currently applies a simple runtime substring filter, not a structured event schema filter.

### The in-memory telemetry buffer is bounded on the Probe

The Probe drops the oldest buffered telemetry entry when the buffer reaches configured capacity.

### The upload interval is globally configured at the HUB, but consumed as one scalar by clients

The HUB stores active/inactive schedule configuration and computes one current `update_interval` value.

The Probe and Collector each simply consume that returned scalar duration.

The CLI can submit schedule changes, but the effective policy remains globally interpreted at the HUB.

### Command validation is split and intentionally narrow

The CLI validates some syntax and parameter types locally.

The HUB treats `set_update_interval` specially, but otherwise mostly stores whatever command payload it is given and leaves interpretation to the Probe.

The Probe still needs to log and ignore unknown commands safely.

### The Collector is intentionally restart-shallow

The Collector does not persist its cursor, so restarting it conceptually means starting a fresh extraction pass from the epoch.

### The CLI is intentionally execution-shallow

The CLI does not track command delivery or command completion. It only submits accepted requests and reports the HTTP result.

### The Update Server is intentionally logic-light

The current `moonblokz-telemetry-update-server` repository does not implement custom update APIs.

Instead, it exposes static assets through a Spin fileserver and relies on the Probe to perform version checks, checksum validation, download, and installation logic.

### Several documentation files now lag behind current behavior

The current repositories still contain documentation drift relative to the running code.

At the conceptual level, the important point is simply that the telemetry architecture should now be read through the reviewed code when article-era or repo-era summaries differ.

Use [moonblokz-telemetry-implementation.md](./moonblokz-telemetry-implementation.md) for the repository-specific drift and sharp-edge details.

## What the Current Probe, HUB, Collector, and CLI Codebases Establish

From the perspective of the MoonBlokz knowledge base, the current codebases establish these conceptual conclusions:

1. the Probe is a long-running station-side operational daemon, not a thin forwarder,
2. the HUB is a compact Spin/WASI coordination service, not a generalized telemetry platform,
3. the Collector is an intentionally thin local extraction tool rather than a durable synchronization engine,
4. the CLI is an intentionally thin operator-side command-shaping tool rather than a direct device controller,
5. the Update Server is currently an intentionally thin static artifact host rather than a custom update-control service,
6. telemetry remains out-of-band from the LoRa mesh,
7. the Probe separates complete local log retention from filtered central upload,
8. the HUB adds global interval policy and delayed log serving on top of raw storage,
9. the Collector turns delayed cloud-served logs into a local append-only file while following HUB-directed pacing,
10. the CLI turns human command syntax into HUB requests while staying hub-centric,
11. the current update path depends on fixed static artifact paths and version metadata published by the Update Server,
12. telemetry upload is a heartbeat-style control loop even when there are no filtered logs,
13. Probe-originated telemetry state such as version info is part of the shared telemetry stream,
14. the current Probe, HUB, Collector, CLI, and Update Server are all more specific than the article-era summary in how they handle filtering, intervals, cleanup, downloads, cursoring, command syntax, and artifact hosting,
15. the broader multi-component telemetry architecture still stands, but the reviewed component behavior should now be read through the current code rather than article assumptions when they differ.

## What Still Remains Open

Even with the reviewed telemetry repositories as source of truth, some conceptual areas remain intentionally open or only partially specified here:

- the current analyzer-side code behavior beyond the simulator documents,
- stronger authentication and artifact-signing approaches,
- long-term scaling or retention policy beyond the current bounded cleanup model,
- and whether field-testing telemetry should later evolve into production operational infrastructure.

## Technical Writer View: How to Read the Telemetry Document Set

Read this conceptual document first when you want to understand:

- why the telemetry system exists,
- what role each major component plays,
- why telemetry is separated from the LoRa mesh,
- how the Probe currently sharpens the station-side operational model,
- how the HUB currently sharpens the cloud-side coordination model,
- how the Collector currently sharpens the local extraction model,
- how the CLI currently sharpens the operator control model,
- and how the field-testing architecture fits into the broader MoonBlokz project.

Then continue with:

- [moonblokz-telemetry-algorythm.md](./moonblokz-telemetry-algorythm.md) for the formal flow of logs, commands, polling, cleanup, updates, and analyzer interaction,
- [moonblokz-telemetry-implementation.md](./moonblokz-telemetry-implementation.md) for implementation-facing repository responsibilities, current module structure, and engineering cautions,
- and the simulator documents when you need the current desktop behavior of live tracking, playback, and operator interaction.
