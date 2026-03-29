# MoonBlokz Telemetry Implementation Notes

## Purpose of This Document

This document captures implementation-facing implications of the MoonBlokz field-testing telemetry architecture, with special focus on the current `moonblokz-probe`, `moonblokz-telemetry-hub`, `moonblokz-log-collector`, `moonblokz-telemetry-cli`, and `moonblokz-telemetry-update-server` repositories as the most concrete deployed-side, cloud-side, local-extraction, operator-command, and update-artifact-hosting sources of truth.

It complements the conceptual and algorithm telemetry documents by identifying:

- what engineering responsibilities are currently implemented in the Probe, HUB, Collector, CLI, and Update Server,
- what repository/module structure and runtime boundaries are explicit in code,
- what deployment and update assumptions are visible in the repos,
- which mechanisms are safety-critical or operationally sensitive,
- and which mismatches or evolution-sensitive areas should remain explicit.

- Use [moonblokz-telemetry-concept.md](./moonblokz-telemetry-concept.md) for the strategic and architectural explanation.
- Use [moonblokz-telemetry-algorythm.md](./moonblokz-telemetry-algorythm.md) for the formal end-to-end telemetry flow.

## Source Basis

This document is grounded primarily in the current repositories, especially:

### Probe

- `moonblokz-probe/Cargo.toml`
- `moonblokz-probe/src/main.rs`
- `moonblokz-probe/src/config.rs`
- `moonblokz-probe/src/error.rs`
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

### HUB

- `moonblokz-telemetry-hub/Cargo.toml`
- `moonblokz-telemetry-hub/src/lib.rs`
- `moonblokz-telemetry-hub/spin.toml`
- `moonblokz-telemetry-hub/API.md`
- `moonblokz-telemetry-hub/README.md`
- `moonblokz-telemetry-hub/IMPLEMENTATION.md`
- `moonblokz-telemetry-hub/QUICKSTART.md`
- `moonblokz-telemetry-hub/test_hub.sh`

### Collector

- `moonblokz-log-collector/Cargo.toml`
- `moonblokz-log-collector/src/main.rs`
- `moonblokz-log-collector/src/config.rs`
- `moonblokz-log-collector/README.md`
- `moonblokz-log-collector/IMPLEMENTATION.md`
- `moonblokz-log-collector/config.toml.example`

### CLI

- `moonblokz-telemetry-cli/Cargo.toml`
- `moonblokz-telemetry-cli/src/main.rs`
- `moonblokz-telemetry-cli/src/parser.rs`
- `moonblokz-telemetry-cli/src/client.rs`
- `moonblokz-telemetry-cli/src/config.rs`
- `moonblokz-telemetry-cli/README.md`
- `moonblokz-telemetry-cli/DEVELOPER.md`
- `moonblokz-telemetry-cli/EXAMPLES.md`
- `moonblokz-telemetry-cli/CHANGELOG.md`
- `moonblokz-telemetry-cli/PROJECT_SUMMARY.md`

### Update Server

- `moonblokz-telemetry-update-server/spin.toml`
- `moonblokz-telemetry-update-server/update-probe.sh`
- `moonblokz-telemetry-update-server/update-node.sh`
- `moonblokz-telemetry-update-server/assets/setup.sh`
- `moonblokz-telemetry-update-server/assets/probe/version.json`
- `moonblokz-telemetry-update-server/assets/node/version.json`

The Medium article remains useful broader context, but where the article or repo docs and the current implementation differ, this document treats the code as authoritative for the reviewed components.

## Scope and Intent

This is not a line-by-line commentary of the whole telemetry ecosystem. Instead, it records the major engineering consequences of the current Probe, HUB, Collector, CLI, and Update Server implementations so future work can preserve important constraints and make current mismatches explicit.

## Current Repository-Level Decomposition

The five reviewed repositories divide responsibilities between station-side runtime, cloud-side coordination, local extraction, operator command submission, and update-artifact hosting.

## Probe repository decomposition

### `src/main.rs`

Owns:

- CLI argument parsing,
- config loading,
- logger initialization,
- channel creation,
- shared-state creation,
- task spawning,
- and top-level task supervision.

### `src/config.rs`

Owns:

- the Probe configuration schema,
- default values for optional settings,
- TOML loading from disk.

### `src/log_entry.rs`

Owns:

- the serialized telemetry log entry structure exchanged with the HUB.

### `src/usb_manager.rs`

Owns:

- the single serial-port connection to the node,
- read/write multiplexing over that port,
- reconnect backoff,
- and long-disconnect bootloader recovery triggering.

### `src/usb_collector.rs`

Owns:

- local node-log file writing,
- Probe-side UTC timestamp generation,
- substring-based upload filtering,
- bounded delivery-buffer maintenance.

### `src/telemetry_sync.rs`

Owns:

- periodic upload scheduling,
- HTTPS upload to the HUB,
- upload retry backoff,
- parsing of returned commands and update interval,
- injection of the synthetic `TM8` version line.

### `src/command_executor.rs`

Owns:

- decoding returned command payloads,
- mapping recognized commands to USB actions, filter changes, reboot, or update-manager calls.

### `src/update_manager.rs`

Owns:

- hourly startup+periodic update checks,
- Node UF2 download and install flow,
- Probe binary self-update flow,
- privileged mount/copy/sync/reboot operations,
- recovery from a node apparently stuck in bootloader mode.

## HUB repository decomposition

### `src/lib.rs`

Owns the entire HUB component in one module, including:

- request routing,
- logger initialization,
- SQLite initialization and queries,
- key-value state access,
- `/update`, `/download`, and `/command` handlers,
- interval-config storage and lookup,
- and cleanup triggering.

### `spin.toml`

Owns:

- Spin application wiring,
- route registration,
- required and optional variables,
- key-value and SQLite capability declaration,
- build command.

### Supporting HUB docs and ops artifacts

- `API.md` — documented request/response contract, partially drifting from code in places
- `README.md` — overview and setup guidance, also partially drifting from code in places
- `IMPLEMENTATION.md` — summary of intended behavior, useful but not always current
- `QUICKSTART.md` — local spin-up and curl examples
- `test_hub.sh` — endpoint smoke-test helper

## Collector repository decomposition

### `src/main.rs`

Owns:

- CLI argument parsing,
- HTTP client creation,
- main polling loop,
- `/download` request construction,
- response parsing,
- timestamp-cursor advancement,
- append-only file writing,
- stderr logging through local macros.

### `src/config.rs`

Owns:

- TOML config parsing,
- serde field renaming for hyphenated keys,
- validation of required fields.

### Supporting Collector docs and artifacts

- `README.md` — overview and usage docs, partially drifting from code in several places
- `IMPLEMENTATION.md` — implementation summary, also drifting from code in several places
- `config.toml.example` — minimal currently supported config shape
- local output files such as `moonblokz-logs.log` are operational artifacts, not part of code logic

## CLI repository decomposition

### `src/main.rs`

Owns:

- top-level argument parsing,
- config loading,
- creation of the HTTP client wrapper,
- interactive REPL loop,
- single-command execution path,
- process exit behavior for parse and auth failures.

### `src/parser.rs`

Owns:

- the command enum,
- local command grammar,
- parameter tokenization,
- timestamp parsing,
- log-level validation,
- JSON payload conversion.

### `src/client.rs`

Owns:

- the reusable `reqwest::Client`,
- `/command` request submission,
- HTTP status classification,
- mapping hub responses into CLI-facing success or error strings.

### `src/config.rs`

Owns:

- TOML config loading,
- deserialization of `api-key` and `hub-url`.

### Supporting CLI docs and artifacts

- `README.md` — user-facing command docs, partially drifting from code
- `DEVELOPER.md` — architecture and extension notes, partially drifting from code
- `EXAMPLES.md` — example commands, several of which no longer match the parser exactly
- `CHANGELOG.md` — recent command-surface history
- `PROJECT_SUMMARY.md` — implementation summary against spec expectations
- `config.toml` — local example config file
- `examples.sh` — shell examples

## Update Server repository decomposition

### `spin.toml`

Owns:

- the Spin application manifest,
- one `/static/...` HTTP route,
- use of the upstream `spin_static_fs.wasm` component,
- publication of the local `assets/` directory.

### `update-probe.sh`

Owns:

- refresh of `assets/probe/` from Probe build outputs,
- single-artifact validation for Probe binaries,
- deployment trigger via `spin cloud deploy`.

### `update-node.sh`

Owns:

- refresh of `assets/node/` from embedded-test build outputs,
- single-artifact validation for Node UF2 files,
- deployment trigger via `spin cloud deploy`.

### `assets/setup.sh`

Owns:

- a Probe-host installation flow,
- download of the current Probe binary and manifest,
- generation of `config.toml` and `start.sh`,
- creation of sudoers rules and systemd service,
- activation of the installed Probe host.

### `assets/probe/` and `assets/node/`

Own:

- the currently published update artifacts,
- the `version.json` manifests consumed by Probe update logic.

## Dependency and Runtime Model

## Probe runtime model

The current Probe `Cargo.toml` makes several implementation commitments explicit.

### Async/runtime stack

- `tokio` with full features

### Serialization/config stack

- `serde`
- `serde_json`
- `toml`

### Network stack

- `reqwest` with `rustls-tls`

### Serial/USB stack

- `tokio-serial`

### Operational utilities

- `chrono`
- `crc32fast`
- `clap`
- `log`
- `simple_logger`
- `anyhow`
- `thiserror`

### Engineering meaning

This is a Linux-hosted async daemon built around one evented runtime rather than a mix of shell glue and helper binaries. The Probe concentrates device I/O, HTTPS communication, and update logic in one Rust process.

## HUB runtime model

The current HUB `Cargo.toml` and `spin.toml` make a very different set of commitments explicit.

### Platform/runtime stack

- Spin SDK 3.x
- WASI/WASM target (`wasm32-wasip2`)

### Core dependencies

- `spin-sdk`
- `serde`
- `serde_json`
- `chrono`
- `anyhow`
- `log`
- `simple_logger`

### Capabilities

- one default SQLite database
- one default key-value store
- no outbound hosts

### Engineering meaning

The HUB is intentionally a capability-limited request/response component. It cannot hide state in process memory across requests and therefore encodes correctness through explicit SQLite + KV usage.

## Collector runtime model

The current Collector `Cargo.toml` makes yet another distinct set of commitments explicit.

### Async/runtime stack

- `tokio` with full features

### HTTP/serialization stack

- `reqwest`
- `serde`
- `serde_json`
- `toml`

### CLI / utility stack

- `clap`
- `chrono`
- `anyhow`
- `thiserror`

### Engineering meaning

The Collector is a small local Tokio program, but unlike the Probe it has no long-lived shared mutable state graph beyond one in-memory cursor and one `reqwest::Client`. It is intentionally much simpler than either the Probe or the HUB.

## CLI runtime model

The current CLI `Cargo.toml` makes a fourth set of commitments explicit.

### Async/runtime stack

- `tokio` with full features

### HTTP/serialization/config stack

- `reqwest`
- `serde`
- `serde_json`
- `toml`

### CLI / parsing utilities

- `clap`
- `chrono`
- `anyhow`
- `thiserror`

### Engineering meaning

The CLI is a small local Tokio program focused on command parsing and request submission. Unlike the Probe, it owns no long-lived device state. Unlike the HUB, it owns no shared coordination state. Unlike the Collector, it is control-plane rather than data-plane oriented.

## Update Server runtime model

The current Update Server repository makes a different set of commitments explicit from the other telemetry components.

### Platform/runtime stack

- Spin manifest v2
- HTTP static-files trigger at `/static/...`
- upstream `spin_static_fs.wasm` component

### Asset model

- local `assets/` directory copied into the deployed static filesystem
- `assets/probe/` for Probe binaries and manifest
- `assets/node/` for Node UF2 files and manifest
- `assets/setup.sh` for initial Probe host installation

### Engineering meaning

The Update Server is currently not a custom service with business logic. It is a static artifact publication surface whose behavior is shaped primarily by filesystem layout, manifest format, and deployment scripts.

## Current Shared-State and Persistence Strategy

## Probe side

The Probe currently uses a straightforward combination of:

- bounded Tokio `mpsc` channels for USB control and line/status delivery,
- `Arc<RwLock<...>>` for runtime-shared mutable state,
- cloned `Arc<Config>` for configuration access.

### Probe persistent state surfaces

1. config file on disk,
2. local node log at `/home/moonblokz/node.log`,
3. local deployed firmware/binary artifacts.

### Important Probe limitation

The in-memory delivery buffer is **not** persisted across process restarts.

## HUB side

The HUB persists all cross-request state explicitly.

### SQLite state

- `log_messages`
- `commands`

### Key-value state

- `last_cleanup_time`
- `update_interval`

### Important correction versus older docs

The current code stores the interval policy under `update_interval` as a serialized config object. It does **not** use the earlier `max_upload_interval` key described in older documentation/spec text.

## Collector side

The Collector persists only:

- the output log file specified by config.

The Collector does **not** persist:

- its last timestamp cursor,
- any item-id cursor,
- any local metadata database,
- any retry or pacing state across restart.

### Engineering implication

The Collector is intentionally operationally shallow. Restarting it does not resume a durable synchronization session; it starts again from epoch and depends on local file handling plus HUB retention to determine practical re-download behavior.

## CLI side

The CLI persists only:

- its local config file.

The CLI does **not** persist:

- command history,
- queued commands locally,
- command completion state,
- any local notion of fleet membership,
- any durable session state across interactive runs.

### Engineering implication

The CLI is intentionally execution-shallow. It shapes and submits commands, but durability and delivery are entirely delegated to the HUB and Probe side.

## Update Server side

The Update Server persists only what is present in its deployed static asset set.

That currently includes:

- one Probe version manifest,
- one current Probe binary,
- one Node version manifest,
- one current Node UF2,
- one setup script.

The Update Server does **not** persist:

- update request history,
- rollout state,
- per-node targeting state,
- download audit state,
- or any server-side decision-making state.

### Engineering implication

The Update Server is intentionally logic-light. Operational meaning comes from the current asset set and the external publish scripts, not from any internal server-side update workflow.

## Configuration Surface Notes

## Probe config surface

Current Probe config fields:

- `usb_port`
- `server_url`
- `api_key`
- `node_id`
- `node_firmware_url`
- `probe_firmware_url`
- `upload_interval_seconds`
- `buffer_size`
- `filter_string`
- `log_level`

### Important current Probe absences

There is currently no Probe-side config representation for:

- active/inactive upload schedule windows,
- multiple HUB endpoints,
- persistent filter storage beyond startup config,
- or per-command authorization separation.

## HUB config surface

Current Spin variables:

- `probe_api_key`
- `log_collector_api_key`
- `cli_api_key`
- `delete_timeout`
- `cleanup_interval`
- `default_upload_interval`
- `loglevel`

### Important current HUB characteristics

- interval policy is global, not per-node,
- cleanup cadence is configurable independently from retention period,
- declared defaults in `spin.toml` do not perfectly match the internal fallback constants in `src/lib.rs`.

## Collector config surface

Current Collector config fields actually loaded by code:

- `api-key`
- `hub-url`
- `log-file`

### Important current Collector absences

The reviewed Collector code does **not** currently load or use:

- `interval`
- `max-items`

even though some docs/spec text still mention them.

This is a real repo-level documentation drift and should remain explicit.

## CLI config surface

Current CLI config fields actually loaded by code:

- `api-key`
- `hub-url`

### Important current CLI absences

The reviewed CLI code does **not** currently load or use:

- a default node selection,
- interactive command history settings,
- output formatting options,
- retry policy configuration,
- or any local schedule persistence.

This keeps the tool simple, but narrower than some future-looking docs suggest.

## Update Server config surface

The current Update Server repository does not expose an application-level config schema comparable to the Probe, HUB, Collector, or CLI.

Its effective configuration surface is instead split across:

- `spin.toml` route and file-mount settings,
- the contents of `assets/`,
- the publish scripts’ source paths,
- and the setup script’s embedded default URLs.

### Important current Update Server characteristics

- artifact family paths are fixed by static directory layout,
- publication scripts assume concrete sibling-repository paths,
- the setup script embeds concrete default hosted URLs,
- deployment is currently triggered by `spin cloud deploy` from shell scripts.

## USB Ownership and Boundary Notes

The Probe codebase preserves one very important implementation invariant:

- `usb_manager.rs` is the single owner of the USB serial port.

Everything else interacts with the node through message passing and `UsbHandle`.

### Why this matters

- it avoids competing serial-port access,
- it localizes reconnect behavior,
- it centralizes command framing,
- and it creates a clean boundary between serial I/O and higher-level Probe logic.

This invariant should remain intact.

## Current Logging Model Notes

The current Probe logging model is more nuanced than the article summary alone suggested.

### Local node log path

Every received USB line is timestamped and written locally.

### Filtered central telemetry path

Only lines matching the current substring filter enter the upload buffer.

### Probe operational log path

The Probe itself logs through `simple_logger` with configurable verbosity.

### Synthetic telemetry insertion

The telemetry sync layer injects one Probe-generated `TM8` line per upload containing current Probe and Node version values.

### Engineering implication

The telemetry stream seen by the HUB is a mixture of:

- node-originated filtered log lines,
- and Probe-originated synthetic telemetry.

That mixed-origin property is now part of the effective interface and should be documented explicitly.

## Buffering, Ordering, Delivery, and Command-Shaping Notes

## Probe-side buffering

The current buffer is a plain in-memory `Vec<LogEntry>` protected by `RwLock`.

### Current behavior

- bounded by `buffer_size`,
- oldest-first eviction when full,
- cloned wholesale when preparing an upload,
- cleared after successful upload or after successful HTTP status with unparseable response body.

### Engineering implications

- large buffers increase clone cost at upload time,
- persistent outage beyond buffer capacity causes oldest-entry loss,
- process restart loses unsent buffered entries,
- successful-but-malformed HUB responses currently still drop buffered telemetry.

## HUB-side ordering and download delay

The HUB stores raw uploaded timestamps and later serves logs ordered by:

- `timestamp ASC, id ASC`

It also delays downloadable visibility using:

- `cutoff_time = now - current_update_interval * 1.1`

### Engineering implications

- the HUB is actively shaping downstream visibility rather than merely exposing inserts,
- collectors must use timestamp cursoring rather than assuming immediate availability,
- timestamp correctness from Probes directly affects global ordering behavior.

## Collector-side delivery model

The Collector currently:

- requests by `last_log_timestamp`,
- parses `item_id` but does not use it as its durable cursor,
- advances to the newest valid timestamp in a batch,
- appends `timestamp:message` lines to its output file,
- flushes the file after each batch.

### Engineering implications

- batch-local timestamp parse failures can weaken perfect cursor advancement,
- equal timestamps across batches can interact subtly with strict `>` timestamp filtering,
- restart behavior may cause repeated downloads of still-retained HUB data,
- the local file is append-only and may therefore accumulate duplicates across restarts.

## CLI-side command-shaping model

The CLI currently:

- parses a local DSL into a `Command` enum,
- validates some parameter types before any network call,
- rewrites `node_id` input into JSON key `node id`,
- converts accepted commands into one HUB `/command` request envelope,
- exits immediately on 401 in one-shot mode and also exits on 401 in interactive mode after printing guidance.

### Engineering implications

- successful CLI submission does not imply Probe execution,
- CLI compatibility depends on both HUB and Probe semantics,
- parser behavior is part of the effective operator contract,
- command examples that do not match the parser are real interoperability/documentation issues, not merely wording issues.

## Telemetry Contract Boundary Notes

The exact request/response shapes and formal command/update flows belong primarily in [moonblokz-telemetry-algorythm.md](./moonblokz-telemetry-algorythm.md).

From an implementation perspective, the important boundary is this:

- the Probe depends on the HUB’s current `/update` contract,
- the Collector depends on the HUB’s current `/download` contract,
- the CLI depends on the HUB’s current `/command` contract,
- and the Probe’s update managers depend on the Update Server’s current static artifact layout and `version.json` schema.

### Important drift to keep explicit

Some repo docs/spec text still describe older or different shapes, such as:

- Collector config fields `interval` and `max-items` being live in code,
- download by message ID instead of timestamp,
- `/update` returning only a command array,
- `max_upload_interval` KV usage instead of stored interval config,
- cleanup running only in `/update`,
- CLI accepting `command(...)` as the raw-command syntax when the parser currently accepts `run_command(...)`,
- CLI examples showing node-scoped `set_update_interval(...)` even though current parser logic rejects `node_id` for that command.

The code should be treated as authoritative for the reviewed components.

## Command Routing Notes

The current system splits command handling between CLI, HUB, and Probe.

## CLI responsibilities

- parse operator input,
- validate basic syntax and selected parameter types,
- convert commands into the expected hub JSON envelope,
- submit them to `/command`,
- surface HTTP status failures to the operator.

## HUB responsibilities

- authenticate CLI requests,
- special-case `set_update_interval`,
- otherwise serialize and queue commands,
- broadcast by expanding to all known node IDs,
- support both `node_id` and `node id` parameter keys.

## Probe responsibilities

- interpret command names,
- map commands to USB/filter/reboot/update actions,
- ignore unsupported commands safely.

## Collector responsibilities

- consume delayed log downloads,
- maintain in-memory timestamp progress,
- append logs durably to a local file,
- honor HUB-provided pacing.

### Important implication

The command path is hub-centric end to end. The CLI does not talk to Probes directly, and the Collector is not part of command routing at all.

## Update Manager Notes

The Probe update manager remains the most operationally sensitive part of the reviewed telemetry stack.

### Node update path includes

- version discovery from `version.json`,
- CRC32 validation,
- temp-file staging in `/tmp`,
- bootloader detection through `/dev` + `blkid`,
- privileged `mount`,
- privileged copy to `firmware.uf2`,
- filesystem `sync`,
- local deployed-artifact retention and cleanup.

### Probe self-update path includes

- version discovery from `version.json`,
- CRC32 validation,
- writing versioned executable in current working directory,
- setting executable permissions,
- generating `start.sh`,
- deleting older versioned executables,
- rebooting the host.

### Recovery path includes

- a bootloader recovery flow triggered from the USB layer after prolonged disconnect.

## Privileged Operations and Deployment Assumptions

The current Probe implementation assumes passwordless privilege for at least:

- `reboot`
- `mount`
- file copy to mounted bootloader path via `sudo cp`

The README also documents `umount`, although the current helper function is stubbed and does not actually run an unmount command.

### HUB deployment assumptions

The HUB assumes:

- Spin-managed HTTP routing,
- available default SQLite and KV stores,
- correct variable provisioning,
- no outbound network dependency.

### Collector deployment assumptions

The Collector assumes:

- local filesystem write access for the configured log file,
- network reachability to the HUB,
- a valid API key,
- and no need for local persistent sync metadata.

### CLI deployment assumptions

The CLI assumes:

- local filesystem read access to config,
- network reachability to the HUB,
- a valid CLI API key,
- and an operator willing to interpret `OK` as request acceptance rather than execution completion.

### Update Server deployment assumptions

The Update Server assumes:

- Spin-compatible deployment of a static-files component,
- correct inclusion of the `assets/` directory at deploy time,
- correct sibling-repository build outputs when publish scripts are run,
- operator-driven deployment via `spin cloud deploy`.

### Engineering implication

The telemetry stack depends on very different deployment invariants on the station side, cloud side, local extraction side, and local command side. Those invariants should remain explicit in documentation and operations guides.

## Current Deployment-Model and Documentation Drift

The reviewed repositories expose several meaningful mismatches.

### Probe self-update vs service startup

The service file starts a fixed binary path under `target/release`, while self-update writes versioned binaries plus `start.sh` in the working directory.

### HUB docs vs implementation

Current docs contain drift around at least these areas:

- `/download` cursor parameter,
- `/update` response shape,
- `/download` response shape,
- supported filter command naming,
- KV key naming and interval-policy description,
- cleanup trigger location.

### Collector docs vs implementation

Current Collector docs contain drift around at least these areas:

- config fields `interval` and `max-items`,
- message-ID cursoring instead of timestamp cursoring,
- response-shape expectations that omit `update_interval`,
- state model based on `last_id` rather than `last_timestamp`.

### CLI docs vs implementation

Current CLI docs contain drift around at least these areas:

- `command(...)` examples versus parser support for `run_command(...)`,
- node-scoped `set_update_interval(...)` examples versus parser rejection of `node_id`,
- quoted string examples that imply quotes are only delimiters even though current parser logic preserves them in the value,
- broad “full syntax support” claims that overstate current parser compatibility with the examples.

### Update Server repo vs Probe assumptions

The current Update Server repo aligns with the Probe on the expected `version.json` shape (`version` + `crc32`), but it also exposes a meaningful deployment-model distinction:

- the setup script installs and starts the Probe through `start.sh`,
- while the legacy Probe repository service file still references a fixed binary path under `target/release`.

This means the update server’s setup path and the Probe repo’s older service artifact encode different activation models.

### Why this matters

These are not cosmetic documentation issues. They affect interoperability between Probe, HUB, Collector, CLI, and operator expectations.

## Current Code-Grounded Inconsistencies and Sharp Edges

The reviewed repos currently contain several issues that documentation should not hide.

### Probe-side sharp edges

1. `/BS` newline duplication risk
2. `unmount_bootloader()` stubbed out
3. node update failure no longer reboots automatically
4. successful-but-malformed HUB response still clears Probe buffer
5. self-update activation path may not match the deployed service path

### HUB-side sharp edges

1. logger initialization is attempted inside request handling
2. most invalid client inputs bubble through generic error handling rather than being mapped explicitly to `400`
3. cleanup deletes old rows in batches of 10000 only per trigger
4. ordinary commands are minimally validated before queueing
5. older documentation describes behavior that no longer matches the code

### Collector-side sharp edges

1. config surface in code is narrower than the documented config surface
2. `item_id` is parsed but not used for cursor advancement
3. response parse failures are treated as outer-loop errors rather than in-function recoverable events
4. restart causes cursor reset to epoch and may duplicate locally collected logs
5. only `timestamp` and `message` are stored locally, so `node_id` is discarded even though the HUB returns it

### CLI-side sharp edges

1. parser accepts `run_command(...)`, not the documented `command(...)` alias
2. parser rejects `node_id` on `set_update_interval(...)` even though docs/examples still show node-specific forms
3. quoted string parameters keep their quote characters in the parsed value
4. command success reporting reflects HUB acceptance only, not Probe execution
5. interactive-mode auth failure detection depends on substring matching of the error text

### Update-Server-side sharp edges

1. deployment depends on external shell scripts and sibling-repository paths rather than one integrated build pipeline
2. the server publishes only one current artifact per family, so there is no built-in multi-version rollback browsing surface
3. the setup script embeds concrete hosted default URLs, which can drift from future deployment locations
4. publication safety checks verify singularity of artifacts, but there is no richer server-side validation beyond static file presence
5. the setup script and the older Probe service artifact encode different startup models (`start.sh` vs fixed binary path)

These discrepancies are valuable knowledge-base content because they affect maintenance, ops behavior, and interoperability confidence.

## Engineering Consequences for Future Work

The current Probe, HUB, Collector, and CLI code imply several constraints that future evolution should preserve or address deliberately.

### Preserve

- single-owner USB access,
- bounded in-memory delivery behavior,
- explicit separation of local full log retention vs filtered upload,
- explicit SQLite + KV persistence on the HUB,
- explicit delayed-download logic,
- explicit append-only local Collector output,
- explicit hub-centric command routing,
- visible error logging for operational diagnosis,
- local CLI validation before network submission.

### Address carefully

- deployment activation path for Probe self-update,
- volatility of unsent buffered logs across restart,
- malformed-success response handling that currently discards buffered telemetry,
- command and documentation drift,
- bootloader mount/unmount correctness,
- explicit `400` handling for malformed HUB requests,
- whether global interval policy should remain fleet-wide or evolve per-node,
- whether the Collector should gain durable cursor persistence or item-id-based replay control,
- whether the CLI should support the documented `command(...)` alias and dequote string parameters,
- whether the Update Server should remain a pure static host or evolve toward richer release-management and rollback capabilities,
- whether setup-script default URLs should be externalized instead of embedded.

## Analyzer / Simulator Integration Notes

The current analyzer lives in `moonblokz-radio-simulator`, but several implementation boundaries now belong more naturally in the telemetry knowledge base because they define how telemetry is consumed operationally.

### Scene-backed telemetry interpretation

The analyzer depends on scene files for static operational context such as node positions, effective distance values, map bounds, and optional background imagery.

This means telemetry interpretation is not based on logs alone; it is based on logs plus a colocated scene model that explains how those logs should be visualized.

### Parser layering as telemetry compatibility surface

Analyzer parsing currently combines:

- raw `[node_id]` log capture for per-node log streams,
- structured TM-event decoding for packet/message/version semantics,
- and TM9 connection-matrix reconstruction.

This makes TM-tagged log format compatibility an implementation concern shared between telemetry producers and the simulator-side analyzer.

### Local control bridge, not control authority

The simulator control module discovers `config.toml` next to the selected scene and uses a blocking `reqwest::blocking::Client` to invoke Telemetry Hub.

That is an implementation boundary worth keeping here because it explains where desktop-side configuration and HTTP invocation live. But the simulator is still not the authority on end-to-end command semantics; it is only the local bridge into the telemetry control path.

### Why this belongs here

Without these notes, the simulator implementation file has to repeat too much telemetry-side context just to explain why analyzer parsing, replay, connection-matrix viewing, and hub-mediated control exist. Keeping these boundaries here lets the simulator docs stay focused on the desktop crate while the telemetry docs own the cross-repository operational interface.

## Relationship to the Wider Telemetry Stack

This file is intentionally Probe-, HUB-, Collector-, CLI-, Update-Server-, and analyzer-integration-heavy because those are the telemetry-relevant implementation boundaries currently reviewed.

For simulator-local UI/runtime detail that does not materially change telemetry semantics, continue using the simulator document set.

## What Must Remain Open

Even with the current Probe, HUB, Collector, and CLI repos as source of truth, several implementation areas remain intentionally open or evolution-sensitive:

- the exact current analyzer implementation details,
- whether the self-update deployment model is standardized around `start.sh` or systemd direct binary execution,
- whether the static Update Server should stay deployment-script-driven or gain a richer release-management surface,
- rollback behavior after partially successful updates,
- whether the in-memory Probe buffer should eventually gain durable spillover,
- whether global upload scheduling should remain globally shared across all nodes,
- whether the Collector should remain intentionally stateless across restart or evolve a durable cursor model,
- and whether the CLI should remain intentionally minimal or grow richer operator affordances such as history, dry-run, structured output, and better semantic validation.
