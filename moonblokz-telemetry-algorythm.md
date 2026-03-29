# MoonBlokz Telemetry Algorithm Model

## Purpose of This Document

This document provides a formal, algorithm-oriented description of the MoonBlokz field-testing telemetry system, with the Probe side grounded in the current `moonblokz-probe` implementation, the HUB side grounded in the current `moonblokz-telemetry-hub` implementation, the Collector side grounded in the current `moonblokz-log-collector` implementation, the CLI side grounded in the current `moonblokz-telemetry-cli` implementation, and the update-artifact-hosting side grounded in the current `moonblokz-telemetry-update-server` repository.

Its purpose is to capture:

- the current Probe runtime structure,
- the current HUB request/response and storage behavior,
- the current Collector polling, parsing, cursoring, and local append behavior,
- the current CLI parsing, validation, request-shaping, and submission behavior,
- the upload, buffering, filtering, retry, version-reporting, cleanup, and delayed-download rules visible in code,
- the currently implemented command-routing behavior,
- the currently implemented OTA flows for Probe binaries and Node firmware,
- and the boundaries of what the reviewed code defines versus what still belongs to broader article-level architecture.

This file is the primary knowledge-base document for the **formal operating flow** of the telemetry architecture, with special emphasis on what is currently implemented in `moonblokz-probe`, `moonblokz-telemetry-hub`, `moonblokz-log-collector`, `moonblokz-telemetry-cli`, and `moonblokz-telemetry-update-server`.

- Use [moonblokz-telemetry-concept.md](./moonblokz-telemetry-concept.md) for goals, system meaning, and architectural boundaries.
- Use [moonblokz-telemetry-implementation.md](./moonblokz-telemetry-implementation.md) for implementation-facing repository roles, current module structure, and engineering cautions.

## Source Basis

This document is grounded primarily in the current codebases, especially:

### Probe

- `moonblokz-probe/src/main.rs`
- `moonblokz-probe/src/config.rs`
- `moonblokz-probe/src/log_entry.rs`
- `moonblokz-probe/src/usb_manager.rs`
- `moonblokz-probe/src/usb_collector.rs`
- `moonblokz-probe/src/telemetry_sync.rs`
- `moonblokz-probe/src/command_executor.rs`
- `moonblokz-probe/src/update_manager.rs`

### HUB

- `moonblokz-telemetry-hub/src/lib.rs`
- `moonblokz-telemetry-hub/spin.toml`
- `moonblokz-telemetry-hub/API.md`
- `moonblokz-telemetry-hub/README.md`
- `moonblokz-telemetry-hub/IMPLEMENTATION.md`
- `moonblokz-telemetry-hub/QUICKSTART.md`

### Collector

- `moonblokz-log-collector/src/main.rs`
- `moonblokz-log-collector/src/config.rs`
- `moonblokz-log-collector/README.md`
- `moonblokz-log-collector/IMPLEMENTATION.md`
- `moonblokz-log-collector/config.toml.example`

### CLI

- `moonblokz-telemetry-cli/src/main.rs`
- `moonblokz-telemetry-cli/src/parser.rs`
- `moonblokz-telemetry-cli/src/client.rs`
- `moonblokz-telemetry-cli/src/config.rs`
- `moonblokz-telemetry-cli/README.md`
- `moonblokz-telemetry-cli/DEVELOPER.md`
- `moonblokz-telemetry-cli/EXAMPLES.md`
- `moonblokz-telemetry-cli/CHANGELOG.md`

### Update Server

- `moonblokz-telemetry-update-server/spin.toml`
- `moonblokz-telemetry-update-server/update-probe.sh`
- `moonblokz-telemetry-update-server/update-node.sh`
- `moonblokz-telemetry-update-server/assets/setup.sh`
- `moonblokz-telemetry-update-server/assets/probe/version.json`
- `moonblokz-telemetry-update-server/assets/node/version.json`

The article **MoonBlokz series part VII/5 — Field Testing Infrastructure** remains useful broader context, but where article-era expectations and current code differ, this document treats the code as authoritative for the reviewed components.

## Scope and Limits

This file captures behavior that is explicit in the current Probe, HUB, Collector, CLI, and Update Server implementations, including:

- Probe main task decomposition,
- bounded buffering and local log persistence,
- USB reconnect and recovery behavior,
- Probe upload request/response structure,
- HUB persistence and cleanup behavior,
- HUB delayed-download behavior,
- Collector request/response behavior,
- Collector timestamp-cursor advancement,
- CLI config loading and execution modes,
- CLI command parsing and JSON conversion rules,
- current command storage and routing behavior,
- current upload-interval handling,
- current Node and Probe update flows,
- and the current version-reporting rule.

It does **not** attempt to define:

- unrelated simulator UI details that do not affect telemetry consumption semantics,
- or any command semantics not currently visible in the reviewed repositories.

## Core Terminology Used in This Document

To stay aligned with the companion telemetry files, this document uses the following terms consistently:

- **USB manager** — the single owner of the serial connection to the node.
- **USB collector** — the task that timestamps received lines, writes the local node log, applies upload filtering, and fills the in-memory telemetry buffer.
- **Telemetry sync task** — the task that sleeps for the current interval, uploads buffered logs, updates the interval, and executes returned commands.
- **Node update manager** — the task that checks and applies RP2040 UF2 updates.
- **Probe update manager** — the task that checks and applies Probe binary self-updates.
- **Delivery buffer** — the bounded in-memory `Vec<LogEntry>` used for upload to the HUB.
- **Local node log** — the append-only file at `/home/moonblokz/node.log` containing all USB lines with Probe-generated UTC timestamps.
- **Synthetic telemetry entry** — a Probe-generated log entry, currently the `TM8` version line inserted before upload.
- **Update interval config** — the HUB-side stored `{ start_time, end_time, active_period, inactive_period }` structure.
- **Current update interval** — the single scalar interval value derived by the HUB from the stored interval config and returned to clients.
- **Cutoff time** — the HUB-side `now - current_update_interval * 1.1` threshold used for delayed log download.
- **Collector cursor** — the Collector’s current `last_log_timestamp` value held only in memory.
- **CLI parser** — the local command-language parser in `moonblokz-telemetry-cli/src/parser.rs`.
- **CLI command envelope** — the JSON object the CLI sends to the HUB `/command` endpoint.
- **Update Server** — the current Spin static-files deployment that publishes Probe and Node artifacts under fixed static paths.
- **Version manifest** — the current `version.json` file published beside each artifact family and containing `version` plus `crc32`.

## Algorithmic Problem Statement

The current telemetry implementation must solve the following combined problem:

1. maintain a durable-looking operational bridge to a serially connected node despite disconnections,
2. capture all node USB output to a local timestamped log file,
3. keep only a bounded filtered subset in memory for upload,
4. periodically contact the HUB even when there are no buffered logs,
5. store uploaded logs and pending commands centrally,
6. derive and return the currently effective upload interval centrally,
7. serve only sufficiently old logs to collectors so cross-Probe ordering is more stable,
8. append those delayed logs to a local file with a simple in-memory timestamp cursor,
9. let operators submit validated command requests to the HUB through a thin local CLI,
10. publish current Probe and Node update artifacts plus version manifests through a thin static update host,
11. manage independent update loops for Node and Probe,
12. and attempt recovery when the node appears stuck in bootloader mode.

## Section A — Probe Runtime Structure

## A1. Main task set

`moonblokz-probe/src/main.rs` initializes configuration, logging, channels, shared state, and spawns exactly five long-running async tasks:

1. `UsbManager::run()`
2. `usb_collector::run(...)`
3. `telemetry_sync::run(...)`
4. `update_manager::run_node_update(...)`
5. `update_manager::run_probe_update(...)`

## A2. Shared state set

The main Probe runtime shares these mutable values across tasks:

- `buffer: Arc<RwLock<Vec<LogEntry>>>`
- `filter_string: Arc<RwLock<String>>`
- `upload_interval: Arc<RwLock<Duration>>`
- cloned `Arc<Config>`
- cloned `UsbHandle`

## A3. Termination rule

The main task waits with `tokio::select!` for any spawned task to exit.

If any task ends, the main process logs an error and then returns.

This means the Probe assumes its worker tasks are meant to run indefinitely.

## Section B — Probe Configuration and Initialization Rules

## B1. Current Probe configuration fields

The Probe currently loads these config fields from TOML:

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

## B2. Current Probe default values

If not specified in config, current defaults are:

- `upload_interval_seconds = 300`
- `buffer_size = 10000`
- `filter_string = ""`
- `log_level = "info"`

## B3. Probe logger initialization rule

The Probe maps the configured textual log level to `simple_logger` level filters and initializes one global logger with UTC timestamps.

This logger controls the Probe’s own operational logs, not the node log stream.

## Section C — Probe USB Connection and Log Capture Algorithm

## C1. Probe channel topology

The current Probe code creates:

- `usb_cmd` channel with capacity `32`
- `usb_msg` channel with capacity `100`

The USB manager receives commands on the first and emits status/lines on the second.

## C2. USB manager main loop

`UsbManager::run()` performs the following loop:

1. try `connect_and_handle()`,
2. if it returns `Ok`, mark the connection as closed normally and restart connection tracking,
3. if it returns `Err`, emit `UsbMessage::Disconnected`, log the error, sleep with exponential backoff, and retry.

## C3. USB reconnect backoff rule

The current USB reconnect policy uses:

- `INITIAL_BACKOFF_MS = 1000`
- `MAX_BACKOFF_MS = 60000`

The backoff doubles after each failed attempt, capped at the maximum.

## C4. USB line handling rule

When connected, the USB manager:

1. opens the configured serial port at `115200`,
2. sends `UsbMessage::Connected`,
3. splits the port into read and write halves,
4. reads lines using `BufReader::read_line`,
5. trims the trailing newline,
6. emits non-empty lines as `UsbMessage::LineReceived(line)`.

## C5. USB command handling rule

When the USB manager receives `UsbCommand::SendCommand(command)`:

1. it writes `format!("{}\r\n", command)` to the serial writer,
2. flushes the writer,
3. returns an error if either write or flush fails.

## C6. Current Probe command-newline discrepancy

The current Probe codebase contains a small but important implementation inconsistency:

- `UsbManager` itself already appends `\r\n` to every command,
- but `update_manager::perform_node_firmware_update()` currently sends `"/BS\r\n"` as the command payload.

That means the wire output for this command may become `"/BS\r\n\r\n"`.

## C7. Bootloader-recovery trigger rule

The USB manager tracks how long the USB connection has been absent.

If the connection has been disconnected for at least:

- `BOOTLOADER_RECOVERY_TIMEOUT_SECS = 300`

and recovery has not yet been attempted during that disconnection episode, the manager calls:

- `update_manager::recover_node_from_bootloader()`.

## Section D — Probe USB Collector Algorithm

## D1. Local node log initialization rule

On startup, the collector opens:

- `/home/moonblokz/node.log`

in create+append mode.

If this file cannot be opened, the collector task fails.

## D2. Received-line processing rule

For each `UsbMessage::LineReceived(line)` the collector:

1. generates a UTC timestamp in `YYYY-MM-DDTHH:MM:SSZ` form,
2. appends `"{timestamp} {line}\n"` to `/home/moonblokz/node.log`,
3. reads the current `filter_string`,
4. if the filter is non-empty and `line` does not contain it, stops processing that line for upload purposes,
5. otherwise creates `LogEntry { timestamp, message: line }`,
6. pushes it into the delivery buffer.

## D3. Probe buffer eviction rule

Before pushing a new filtered `LogEntry`, the collector checks whether:

- `buf.len() >= config.buffer_size`

If true, it removes the oldest entry with `buf.remove(0)` before pushing the new entry.

This means the current Probe buffer is bounded and evicts oldest-first.

## D4. Connection-status handling rule

The collector does not alter state significantly on `Connected` or `Disconnected` messages.

It only logs the notification.

## Section E — Probe Telemetry Sync Algorithm

## E1. Telemetry sync main loop

`telemetry_sync::run()` behaves as follows:

1. build one `reqwest::Client`,
2. initialize upload backoff to `1000 ms`,
3. read the current shared upload interval,
4. sleep for that duration,
5. call `upload_telemetry(...)`,
6. on success reset backoff,
7. on failure sleep for the current backoff and then double it up to `60000 ms`.

## E2. Upload request preparation rule

`upload_telemetry(...)` currently prepares the upload body by:

1. cloning the full current delivery buffer,
2. obtaining `probe_version` via `get_current_probe_version()`, defaulting to `0` on error,
3. obtaining `node_version` via `get_current_node_version()`, defaulting to `0` on error,
4. generating a current UTC timestamp,
5. creating a synthetic telemetry message of the form:
   - `[INFO] moonblokz_probe: [<node_id>] *TM8* probe_version: <p>, node_version: <n>`
6. appending that synthetic entry to the outgoing logs.

Therefore, every upload currently includes at least one synthetic version-reporting entry even if the buffered log set was empty.

## E3. Current Probe upload request schema

The Probe currently POSTs to:

- `{server_url}/update`

with headers:

- `Content-Type: application/json`
- `X-Node-ID: <node_id>`
- `X-Api-Key: <api_key>`

and body:

```json
{
  "logs": [ ... ]
}
```

where each log entry contains `timestamp` and `message`.

## E4. Current HUB update response schema expected by Probe

The Probe currently expects a JSON object with these fields:

- `commands: Vec<Command>`
- `update_interval: u64`

This is narrower and more specific than some earlier article-era descriptions that treated command arrays alone as sufficient.

## E5. Probe success handling rule

If the response status is successful and the body parses into the expected schema:

1. clear the delivery buffer,
2. update the shared upload interval if the returned value differs,
3. execute returned commands sequentially.

## E6. Probe non-success status rule

If the response status is not successful:

1. log a warning,
2. return an error,
3. keep the delivery buffer intact,
4. let the caller apply retry backoff.

## E7. Probe parse-failure handling rule

If the HTTP status is successful but JSON parsing of the response fails:

1. log a warning,
2. clear the delivery buffer anyway,
3. do not execute any commands,
4. return success.

This means the current Probe treats `2xx` as proof of log delivery even if the response body is malformed.

## E8. Current Probe interval-control rule

The current Probe implementation does **not** interpret active/inactive schedule windows itself.

Instead, it simply stores the last returned `update_interval` from the HUB as one shared `Duration`.

Therefore, the current Probe-side scheduling model is:

- one mutable upload interval value,
- controlled by successful server responses,
- not by a local schedule command interpreter.

## Section F — HUB Storage and Configuration Algorithm

## F1. Current Spin variable set

The HUB currently declares these variables in `spin.toml`:

- `probe_api_key` (required)
- `log_collector_api_key` (required)
- `cli_api_key` (required)
- `delete_timeout` (default `60`)
- `cleanup_interval` (default `1`)
- `default_upload_interval` (default `300`)
- `loglevel` (default `info`)

## F2. Current fallback constants in code

If variables are missing or unparsable, the HUB code also contains internal fallback constants:

- `DEFAULT_CLEANUP_INTERVAL_MINUTES = 5`
- `DEFAULT_DELETE_TIMEOUT_MINUTES = 30`
- `DEFAULT_UPLOAD_INTERVAL_SECONDS = 300`
- `MAX_LOG_ITEMS_PER_DOWNLOAD = 10000`

This means there is a subtle distinction between the defaults declared in `spin.toml` and the internal fallback constants used if variable lookup/parsing fails.

## F3. Current SQLite schema

The HUB initializes these tables on demand:

### `log_messages`

- `id INTEGER PRIMARY KEY AUTOINCREMENT`
- `timestamp TEXT NOT NULL`
- `node_id INTEGER NOT NULL`
- `message TEXT NOT NULL`

### `commands`

- `id INTEGER PRIMARY KEY AUTOINCREMENT`
- `timestamp TEXT NOT NULL`
- `node_id INTEGER NOT NULL`
- `command TEXT NOT NULL`

It also creates an index:

- `idx_log_messages_timestamp` on `log_messages(timestamp)`.

## F4. Current key-value state

The HUB currently uses key-value store entries for:

- `last_cleanup_time`
- `update_interval`

The current code does **not** use the previously described `max_upload_interval` key. Instead, it stores one global serialized `UpdateIntervalConfig` and derives the current scalar interval from that config.

## F5. Current interval-config structure

The HUB currently stores the following structure in KV:

- `start_time: u64`
- `end_time: u64`
- `active_period: i64`
- `inactive_period: i64`

These times are stored as Unix timestamps, not ISO timestamps.

## F6. Current effective-interval rule

When the HUB needs the current upload interval it:

1. loads the stored interval config if present,
2. reads `now = Utc::now().timestamp() as u64`,
3. if `start_time <= now <= end_time`, returns `active_period`,
4. otherwise returns `inactive_period`,
5. if no config exists, returns the default interval.

## Section G — HUB `/update` Algorithm

## G1. Authentication rule

`POST /update` requires header:

- `X-Api-Key` matching `probe_api_key`

Otherwise the HUB returns `401 Unauthorized`.

## G2. Node-identification rule

`POST /update` requires header:

- `X-Node-ID`

The value is parsed as `u32`.

## G3. Request-body rule

The HUB expects request JSON:

```json
{ "logs": [ { "timestamp": "...", "message": "..." }, ... ] }
```

## G4. Log-insertion rule

For each received log entry, the HUB inserts one row into `log_messages` with:

- uploaded `timestamp`
- request `node_id`
- uploaded `message`

The current implementation inserts entries row-by-row rather than within one explicit transaction.

## G5. Cleanup trigger rule in `/update`

After inserting logs, the HUB checks whether cleanup should run.

Cleanup runs when:

- there is no stored `last_cleanup_time`, or
- the elapsed time since `last_cleanup_time` is at least `cleanup_interval` minutes.

## G6. Cleanup behavior rule

When cleanup runs, the HUB deletes old rows from both tables using timestamp cutoff:

- `Utc::now() - delete_timeout minutes`

Each delete statement is limited to `10000` rows per call via a subquery.

Then it updates `last_cleanup_time` in KV.

## G7. Command-delivery rule in `/update`

After optional cleanup, the HUB:

1. loads all commands for the given `node_id` ordered by `id`,
2. deserializes each stored JSON string into `Command`,
3. deletes all commands for that node,
4. returns them in the response.

This means the current command-delivery model is:

- queue by node,
- FIFO by insertion ID,
- delete-on-delivery.

## G8. `/update` response rule

The HUB currently returns JSON object:

```json
{
  "commands": [ ... ],
  "update_interval": <current_interval>
}
```

This is the response shape the current Probe implementation depends on.

## Section H — HUB `/download` Algorithm

## H1. Authentication rule

`GET /download` requires header:

- `X-Api-Key` matching `log_collector_api_key`

Otherwise the HUB returns `401 Unauthorized`.

## H2. Current download cursor rule

The current HUB implementation uses query parameter:

- `last_log_timestamp`

not `last_log_message_id`.

That timestamp must parse as `DateTime<Utc>`.

## H3. Cutoff-time rule

Before serving logs, the HUB computes:

- `current_upload_interval = get_current_update_interval(...)`
- `cutoff_time = now - current_upload_interval * 1.1`

The 1.1 safety factor is implemented as float multiplication and cast back to integer seconds.

## H4. Download query rule

The HUB selects rows where:

- `timestamp > last_log_timestamp`
- `timestamp < cutoff_time`

with ordering:

- `ORDER BY timestamp ASC, id ASC`

and limit:

- `10000`

## H5. `/download` response rule

The HUB currently returns JSON object:

```json
{
  "logs": [ ... ],
  "update_interval": <current_interval>
}
```

where each log entry contains:

- `item_id`
- `timestamp`
- `node_id`
- `message`

This means the current `/download` response is richer than some earlier docs that described only a `logs` array.

## H6. Cleanup trigger rule in `/download`

The current HUB also runs the same cleanup check during `/download`, not only during `/update`.

This is an important behavior difference versus earlier documentation that described cleanup as upload-triggered only.

## Section I — Collector Algorithm

## I1. Current Collector configuration fields

The current Collector code loads exactly these TOML fields:

- `api-key`
- `hub-url`
- `log-file`

The reviewed code does **not** currently load or use `interval` or `max-items` from configuration.

## I2. Collector initial cursor rule

At startup the Collector initializes:

- `last_timestamp = 1970-01-01T00:00:00Z`

as a `DateTime<Utc>` value.

This cursor exists only in memory.

## I3. Collector initial poll interval rule

Before the first successful response, the Collector uses:

- `poll_interval = 60` seconds.

## I4. Collector main loop

`LogCollector::run()` behaves as follows:

1. ensure the local log file exists or can be created,
2. log startup information to stderr,
3. initialize `poll_interval = 60`,
4. call `fetch_and_save_logs()` repeatedly forever,
5. if the call succeeds, replace `poll_interval` with the returned `update_interval`,
6. sleep for `poll_interval` seconds.

## I5. Collector request rule

For each fetch cycle, the Collector:

1. constructs `GET {hub_url}/download?last_log_timestamp=<cursor>`,
2. sends header `X-Api-Key: <api-key>`,
3. waits up to `30 seconds` using the configured `reqwest::Client` timeout.

## I6. Collector response schema rule

The Collector currently expects JSON object:

- `logs: Vec<LogEntry>`
- `update_interval: u64`

and each log entry contains:

- `item_id: u64`
- `timestamp: String`
- `message: String`

The current Collector implementation does **not** deserialize `node_id`, even though the HUB currently returns it.

## I7. Collector success handling rule

On successful response parse, the Collector:

1. records `log_count = logs.len()`,
2. records `update_interval = response.update_interval`,
3. if `log_count > 0`, appends all logs to the output file,
4. scans all returned timestamps,
5. updates `last_timestamp` to the newest successfully parsed timestamp.

## I8. Collector timestamp-advancement rule

The Collector does **not** advance its cursor using `item_id`.

Instead it advances using the newest successfully parsed RFC3339 timestamp in the response.

If a returned log has an invalid timestamp string:

- the Collector logs an error,
- but still continues processing the rest of the batch.

## I9. Collector local file append rule

For each returned log entry, the Collector currently writes:

- `"{timestamp}:{message}\n"`

to the configured local file.

This is append-only behavior with explicit file flush after each batch.

## I10. Collector non-success status rule

If the response status is not successful, the Collector handles it as follows:

- `401` → returns a terminating error (`Invalid API key. Terminating.`)
- `400` → logs error and returns `(0, 60)`
- `5xx` → logs error and returns `(0, 60)`
- other non-success → logs error and returns `(0, 60)`

Therefore, current retry timing on server-side errors falls back to 60 seconds rather than preserving the previous server-provided interval.

## I11. Collector parse-failure rule

If JSON parsing fails, `fetch_and_save_logs()` returns an error.

The outer loop logs the error and still sleeps for the current `poll_interval` value already held by the caller.

## I12. Collector restart behavior rule

Because `last_timestamp` is in-memory only and not persisted, each Collector restart resumes from the epoch.

This means the current implementation may re-download all still-available HUB logs after restart.

## Section J — CLI Algorithm

## J1. Current CLI configuration fields

The current CLI code loads exactly these TOML fields:

- `api-key`
- `hub-url`

## J2. Current CLI command-line surface

The current CLI supports these top-level arguments:

- `--config <path>` with default `config.toml`
- `--command <string>` for one-shot execution

If `--command` is omitted, the CLI enters interactive mode.

## J3. CLI startup rule

`src/main.rs` currently performs this sequence:

1. parse process arguments with `clap`,
2. load config from TOML,
3. create one `Client` with one `reqwest::Client`,
4. if `--command` was supplied, call `execute_single_command(...)`,
5. otherwise enter `interactive_mode(...)`.

## J4. CLI HTTP client rule

`Client::new(...)` builds one `reqwest::Client` with:

- timeout `30 seconds`

and stores the loaded config alongside it.

## J5. CLI interactive banner rule

Interactive mode prints:

- `MoonBlokz Telemetry CLI - Interactive Mode`
- `Type 'quit', 'exit', or 'bye' to exit`

then repeatedly prompts with:

- `> `

## J6. CLI interactive read-eval loop

For each interactive iteration the CLI:

1. prints the prompt,
2. flushes stdout,
3. reads one line from stdin,
4. trims whitespace,
5. ignores empty lines,
6. parses the input with `parse_command(...)`,
7. exits cleanly on `Quit`,
8. otherwise submits the parsed command to the HUB,
9. prints `OK` on success,
10. prints the error on failure.

## J7. CLI special authentication-exit rule

In interactive mode, if the error string contains:

- `401 Unauthorized`

then the CLI also prints:

- `Authentication failed. Please check your API key in the config file.`

and terminates the process with exit code `1`.

## J8. CLI single-command mode rule

In one-shot mode the CLI:

1. parses the supplied command string,
2. rejects `Quit` as invalid outside interactive mode,
3. submits the command if parsing succeeded,
4. prints `OK` on success,
5. otherwise prints the error and exits with status `1`.

## J9. CLI quit-command rule

The parser treats these case-insensitive inputs as `Quit`:

- `quit`
- `exit`
- `bye`

## J10. Current CLI supported command names

The parser currently accepts these case-insensitive command names:

- `set_update_interval`
- `set_log_level`
- `set_log_filter`
- `run_command`
- `update_node`
- `update_probe`
- `reboot_probe`
- `start_measurement`

This is an important code-grounded correction: the parser does **not** currently accept the documented top-level command name `command(...)`; it accepts `run_command(...)`.

## J11. CLI parameter grammar rule

If the input contains `(`, the parser:

1. treats the text before `(` as the command name,
2. requires the whole input to end with `)`,
3. treats the content inside the parentheses as the raw parameter string.

If the closing parenthesis is missing, parsing fails with:

- `Missing closing parenthesis`

## J12. CLI parameter-tokenization rule

`parse_params(...)` currently tokenizes parameters by:

- splitting on `=` between key and value,
- splitting on `,` between parameters when not inside double quotes,
- trimming surrounding whitespace on keys and values.

Parameter keys are matched case-insensitively.

## J13. Current CLI quoted-string behavior

The parameter tokenizer toggles quote-state when it sees `"`, but it also preserves the quote characters in the captured value.

Therefore, quoted string values such as:

- `log_filter="[ERROR]"`

are currently passed through with the quotes still present in the parsed string value.

This is an implementation sharp edge because documentation examples often use quotes as if they were only syntactic delimiters.

## J14. CLI node-targeting rule

For most command types the parser accepts:

- `node_id=<u32>`

as an optional parameter.

If `node_id` is omitted, the command variant stores `None`, which the HUB later interprets as a broadcast-style request.

## J15. CLI JSON node-key rule

When the CLI converts a parsed command to JSON, it emits:

- `"node id"`

with a space in the `parameters` object, not `node_id`.

This matches current HUB behavior even though the CLI’s local input syntax uses `node_id`.

## J16. `set_update_interval` CLI rule

For `set_update_interval`, the parser currently:

1. requires `start_time`, `end_time`, `active_period`, and `inactive_period`,
2. rejects any supplied `node_id`,
3. parses start/end as ISO timestamps into UTC,
4. parses periods as `u64`,
5. emits JSON command:

```json
{
  "command": "set_update_interval",
  "parameters": {
    "start_time": "...RFC3339 UTC...",
    "end_time": "...RFC3339 UTC...",
    "active_period": <u64>,
    "inactive_period": <u64>
  }
}
```

This is an important code-grounded correction: the current CLI does **not** support node-specific `set_update_interval(...)` requests even though some docs/examples still show them.

## J17. `set_log_level` CLI rule

The parser:

1. optionally parses `node_id`,
2. requires `log_level`,
3. uppercases the supplied value,
4. accepts only `TRACE | DEBUG | INFO | WARN | ERROR`,
5. emits JSON command `set_log_level` with parameter key `log_level` plus optional `node id`.

## J18. `set_log_filter` CLI rule

The parser:

1. optionally parses `node_id`,
2. requires `log_filter`,
3. emits JSON command `set_log_filter` with parameter key `log_filter` plus optional `node id`.

## J19. `run_command` CLI rule

The parser:

1. optionally parses `node_id`,
2. requires parameter `command`,
3. emits JSON command:

```json
{
  "command": "run_command",
  "parameters": {
    "command": "...",
    "node id": <optional>
  }
}
```

This matches the current Probe command executor better than the older documented `command` name.

## J20. `update_node`, `update_probe`, and `reboot_probe` CLI rule

For each of these commands the parser:

1. optionally parses `node_id`,
2. accepts empty parameter lists such as `update_node()`,
3. emits the corresponding command name with an empty or node-scoped `parameters` object.

## J21. `start_measurement` CLI rule

The parser:

1. requires `node_id`,
2. requires `sequence`,
3. parses both as `u32`,
4. emits JSON command `start_measurement` with parameters:
   - `node id`
   - `sequence`

## J22. Current local validation boundary in the CLI

The CLI locally validates:

- command name,
- presence of required parameters,
- integer parsing for `node_id`, `sequence`, and intervals,
- log-level enumeration,
- basic timestamp parseability.

The CLI does **not** locally validate:

- whether a target node currently exists,
- whether a broadcast will reach any nodes,
- whether a queued command will later be supported by the Probe,
- or whether a command that the docs show is actually accepted by the parser unless the code path exists.

## J23. CLI request-submission rule

For every non-quit parsed command, `client.send_command(...)` currently:

1. converts the command to JSON,
2. POSTs to `{hub_url}/command`,
3. sends headers:
   - `Content-Type: application/json`
   - `X-Api-Key: <api-key>`
4. waits up to `30 seconds`.

## J24. CLI HTTP status handling rule

The CLI client handles responses as follows:

- `200 OK` → return `OK`
- `401 Unauthorized` → return `Command error: 401 Unauthorized - Invalid API key`
- other `4xx` → return `Command error: <status> - <body>`
- `5xx` → return `Server error: <status> - <body>`
- other status → return `Unexpected response: <status>`

## Section K — Update Server Algorithm

## K1. Current runtime model

The current `moonblokz-telemetry-update-server` repository does not implement a custom application-specific update API.

Instead it declares one Spin HTTP trigger:

- route `/static/...`

and serves the local `assets/` directory through the upstream `spin_static_fs.wasm` component.

## K2. Current published path structure

The current repository layout and setup script imply these static URL families:

- `/static/probe/...`
- `/static/node/...`
- `/static/setup.sh`

The Probe-side setup script currently defaults to concrete Fermyon-hosted URLs under those paths.

## K3. Probe artifact publication rule

`update-probe.sh` currently performs this sequence:

1. delete existing `assets/probe/moonblokz_probe_*`,
2. delete `assets/probe/version.json`,
3. copy one new `moonblokz_probe_*` binary from:
   - `../moonblokz-probe/versioninfo/probe/`
4. copy one new `version.json` from the same source,
5. verify that `assets/probe/version.json` exists,
6. verify that exactly one `moonblokz_probe_*` file exists,
7. run `spin cloud deploy`.

## K4. Node artifact publication rule

`update-node.sh` currently performs this sequence:

1. delete existing `assets/node/moonblokz_node_*`,
2. delete `assets/node/version.json`,
3. copy one new `moonblokz_node_*` UF2 file from:
   - `../moonblokz-radio-lib/examples/moonblokz-radio-embedded-test/versioninfo/node/`
4. copy one new `version.json` from the same source,
5. verify that `assets/node/version.json` exists,
6. verify that exactly one `moonblokz_node_*` file exists,
7. run `spin cloud deploy`.

## K5. Current version-manifest schema rule

The currently published `assets/probe/version.json` and `assets/node/version.json` both use schema:

```json
{
  "version": <integer>,
  "crc32": "<hex-string>"
}
```

This is an important code-grounded fact because the current Probe update logic expects `crc32`, not a differently named checksum field.

## K6. Current setup-script role

`assets/setup.sh` is currently published as a static installation helper for Probe hosts.

Its current algorithm is to:

1. validate the runtime environment,
2. prompt for or require `node_id` and `api_key`,
3. create `/home/moonblokz/moonblokz-probe`,
4. download `version.json` from `/static/probe`,
5. extract `version`,
6. download `moonblokz_probe_<version>`,
7. generate `config.toml`,
8. generate `start.sh`,
9. configure passwordless sudo entries,
10. add the user to `dialout`,
11. create a systemd service whose `ExecStart` points to `start.sh`,
12. enable and start the service.

## K7. Current setup-script default URL rule

The setup script currently embeds default URLs for:

- `PROBE_DOWNLOAD_URL`
- `TELEMETRY_HUB_URL`
- `NODE_FIRMWARE_URL`

pointing to concrete Fermyon-hosted deployments.

This means the current setup flow is not purely environment-discovered; it ships with deployment-specific defaults that may need manual override outside the current hosted environment.

## K8. Current asset-singularity rule

Both publish scripts require exactly one current artifact file per family:

- one `moonblokz_probe_*`
- one `moonblokz_node_*`

Therefore, the currently deployed update-server model is:

- one current Probe binary,
- one current Node UF2,
- one version manifest per family,
- no multi-version browsing or custom update selection API.

## Section L — HUB `/command` Algorithm

## L1. Authentication rule

`POST /command` requires header:

- `X-Api-Key` matching `cli_api_key`

Otherwise the HUB returns `401 Unauthorized`.

## L2. Current command request envelope

The HUB expects request JSON with:

- `command: String`
- `parameters: Option<serde_json::Value>`

## L3. Special handling of `set_update_interval`

If `command == "set_update_interval"`, the HUB does **not** insert a command into the command queue.

Instead it:

1. requires parameters `start_time`, `end_time`, `active_period`, `inactive_period`,
2. parses start/end as ISO 8601 timestamps,
3. converts them to Unix timestamps,
4. stores one global `UpdateIntervalConfig` in KV under `update_interval`,
5. returns `200 OK` with body `OK`.

## L4. Current scope of interval configuration

The current HUB stores one global interval configuration.

It is not stored per node.

Therefore, current interval scheduling is fleet-global in the reviewed HUB code.

## L5. Ordinary command storage rule

For any other command, the HUB:

1. wraps the incoming request into the stored `Command` shape,
2. serializes it as JSON string,
3. checks parameters for either `node_id` or `node id`,
4. if one is present, inserts one command row for that node,
5. otherwise loads all distinct node IDs from `log_messages` and inserts one row per node.

## L6. Current command-validation boundary

Except for `set_update_interval`, the HUB performs only minimal semantic validation.

That means unknown or unsupported commands may still be queued and later delivered to Probes.

Therefore, the current system relies on the Probe to ignore unsupported commands safely.

## Section M — Probe Command Execution Algorithm

## M1. Current Probe command envelope

The Probe currently expects each command to contain:

- `command: String`
- `parameters: serde_json::Value` or absent

The parameters are then decoded into a permissive `CommandParameters` struct with optional string and integer fields.

## M2. Current Probe implemented command set

`command_executor.rs` currently implements these command names:

- `set_log_level`
- `set_log_filter`
- `run_command`
- `update_node`
- `update_probe`
- `reboot_probe`
- `start_measurement`

Unknown commands are logged and ignored.

## M3. `set_log_level` rule

The Probe accepts either `log_level` or `level` from parameters.

It maps them to these USB commands:

- `TRACE` → `/LT`
- `DEBUG` → `/LD`
- `INFO` → `/LI`
- `WARN` → `/LW`
- `ERROR` → `/LE`

## M4. `set_log_filter` rule

The Probe accepts either `log_filter` or `value` as the new filter string.

It updates the shared in-memory `filter_string` immediately.

The current implementation does **not** flush or rewrite the existing delivery buffer when the filter changes.

## M5. `run_command` rule

If parameters contain a non-empty `command`, that string is sent over USB.

Otherwise, if `value` is non-empty, that value is sent over USB.

## M6. `update_node` rule

The Probe immediately calls `check_and_update_node_firmware(...)`.

## M7. `update_probe` rule

The Probe immediately calls `check_and_update_probe(...)`.

## M8. `reboot_probe` rule

The Probe waits `2 seconds` and then calls `reboot_system()`.

## M9. `start_measurement` rule

The Probe expects a non-zero integer `sequence` parameter.

If `sequence == 0`, it logs a warning and does nothing.

Otherwise it sends the USB command:

- `/M_<sequence>_`

## M10. Not-currently-implemented command rule on Probe side

The current Probe implementation does **not** implement a `set_update_interval` command in `command_executor.rs`.

Therefore, current upload-interval control is a HUB-side policy mechanism, not a Probe-interpreted command.

## Section N — Probe Node Update Algorithm

## N1. Update loop timing

`run_node_update(...)` performs:

1. one startup check,
2. then one check every `CHECK_INTERVAL_SECONDS = 3600`.

## N2. Current node-version source

The Probe determines current Node version by scanning:

- `node_firmware/`

for files matching:

- `moonblokz_node_<version>.uf2`

If none are found, version `0` is assumed.

## N3. Current node-update flow

When a newer Node version is found:

1. GET `{node_firmware_url}/version.json`,
2. parse `{ version, crc32 }`,
3. compare to current version,
4. GET `{node_firmware_url}/moonblokz_node_<version>.uf2`,

In the currently reviewed update-server repository, these assets are published as static files under `/static/node/` and the publication scripts enforce exactly one current UF2 artifact plus one manifest.
5. verify CRC32,
6. write temp file to `/tmp/moonblokz_node_<version>.uf2`,
7. attempt to enter bootloader mode,
8. detect bootloader block device,
9. mount it at `/tmp/rpi-rp2-bootloader`,
10. copy the UF2 file to `firmware.uf2`,
11. run `sync`,
12. wait `5 seconds`,
13. copy the temp UF2 into `node_firmware/`,
14. delete older Node firmware versions.

## N4. Bootloader detection rule

The Probe looks for `/dev/sd*` devices and runs:

- `blkid -s LABEL -o value <device>`

It considers the device a valid RP2040 bootloader when the label equals:

- `RPI-RP2`

## N5. Current mount / unmount behavior

Mount uses:

- `sudo mount -t vfat <device> <mount_point>`

However, `unmount_bootloader(...)` currently returns `Ok(())` without invoking `umount`.

## N6. Node update failure rule

If the Node update process fails, the current code logs the error and returns it.

The source contains commented-out reboot behavior, but reboot-on-failure is **not** currently active in the executed path.

## N7. Recovery algorithm for bootloader-stuck state

`recover_node_from_bootloader()` performs:

1. detect whether a bootloader device is currently present,
2. locate the first deployed firmware in `node_firmware/`,
3. mount the bootloader device,
4. copy that deployed firmware to `firmware.uf2`,
5. run `sync`,
6. wait `5 seconds`,
7. return success.

## Section O — Probe Self-Update Algorithm

## O1. Update loop timing

`run_probe_update(...)` performs:

1. one startup check,
2. then one check every `CHECK_INTERVAL_SECONDS = 3600`.

## O2. Current probe-version source

The Probe determines current Probe version by scanning the current working directory for files matching:

- `moonblokz_probe_<version>`

If none are found, version `0` is assumed.

## O3. Current probe-update flow

When a newer Probe version is found:

1. GET `{probe_firmware_url}/version.json`,
2. parse `{ version, crc32 }`,
3. compare to current version,
4. GET `{probe_firmware_url}/moonblokz_probe_<version>`,

In the currently reviewed update-server repository, these assets are published as static files under `/static/probe/` and the publication scripts enforce exactly one current Probe binary plus one manifest.
5. verify CRC32,
6. write `./moonblokz_probe_<version>`,
7. set executable permissions on Unix,
8. write `./start.sh` that execs the canonicalized binary with `--config config.toml`,
9. set executable permissions on `start.sh`,
10. delete older `moonblokz_probe_<version>` files,
11. wait `5 seconds`,
12. call `sudo reboot`.

## O4. Current service-model discrepancy

The repository currently contains a systemd service file that starts:

- `/home/pi/moonblokz-probe/target/release/moonblokz-probe --config ...`

However, the self-update flow writes and refreshes:

- `./moonblokz_probe_<version>`
- `./start.sh`

This is a real deployment-model inconsistency unless the deployed service is manually adjusted to use `start.sh` or a versioned-binary path.

## Section P — Analyzer / Simulator Telemetry Consumption Algorithm

## P1. Analyzer startup dependency rule

When the simulator enters analyzer mode, it consumes telemetry outputs rather than defining telemetry production.

The current startup sequence is:

1. load the selected scene in analyzer mode,
2. build per-node static context such as effective distance,
3. in real-time tracking mode, look for `config.toml` next to the selected scene,
4. if that config loads successfully, enable hub-mediated control affordances in the UI,
5. initialize node/map state before processing any telemetry log lines.

## P2. Log-source rule

The analyzer currently supports two telemetry-consumption sources:

- tail-follow of a live local log file for real-time tracking,
- start-to-end replay of a historical local log file for visualization.

This means the simulator consumes telemetry after it has already been produced and made available locally; it is not itself the authority on Probe upload, HUB storage, or Collector download rules.

## P3. Structured telemetry decoding rule

The analyzer interprets telemetry-tagged log lines into UI state using the current TM-event family, including packet send/receive activity, measurement events, version information, and connection-matrix reconstruction from TM9 sequences.

Therefore, analyzer behavior depends directly on telemetry log compatibility even though it remains a desktop concern rather than a station-side or hub-side algorithm.

## P4. Replay-timing dependency rule

For historical playback, the analyzer preserves inter-event timing relationships from timestamps embedded in the local log stream and uses a bounded catch-up heuristic when host-side processing falls behind.

This is distinct from HUB delayed-serving behavior: the HUB shapes when logs become downloadable, while the analyzer shapes how already-local logs are replayed for human inspection.

## P5. Hub-mediated control invocation rule

When real-time tracking control is enabled, the simulator may translate UI actions into authenticated HTTP requests to Telemetry Hub.

At this boundary the simulator is only a telemetry consumer and control initiator. The formal semantics of command routing, queueing, broadcast expansion, and Probe execution remain defined by Sections L and M of this document.

## P6. Connection-matrix request rule in live tracking

In live tracking mode, the analyzer may request connection-matrix data through hub-mediated command forwarding rather than direct local node access.

This reinforces the architectural split:

- simulation mode can inspect synthetic in-process node state directly,
- live tracking mode must rely on the out-of-band telemetry/control path and compatible telemetry log output.

## Main Algorithmic Conclusions

From the perspective of the MoonBlokz knowledge base, the current `moonblokz-probe`, `moonblokz-telemetry-hub`, `moonblokz-log-collector`, `moonblokz-telemetry-cli`, and `moonblokz-telemetry-update-server` repositories establish these formal telemetry conclusions:

1. the Probe runtime is a five-task async system with explicit shared state and channel boundaries,
2. the HUB runtime is a single Spin component with explicit SQLite and KV persistence,
3. the Collector runtime is a simple local polling loop with one in-memory timestamp cursor and one append-only output file,
4. the CLI runtime is a simple local request-shaping loop with one config, one parser, and one `/command` HTTP client,
5. the current Update Server runtime is a Spin static-files deployment rooted at `/static/...`, not a custom update API service,
6. every USB line is persisted locally on the station side, while only filter-matching lines are buffered for upload,
7. telemetry uploads occur even when no filtered logs exist because the Probe still needs command retrieval and version reporting,
8. the current HUB response contract for `/update` is `{ commands, update_interval }`,
9. the current HUB response contract for `/download` is `{ logs, update_interval }`,
10. current interval handling is globally configured at the HUB and consumed as a single scalar by the Probe and Collector,
11. current CLI interval-setting input is global-only and intentionally narrower than some documentation examples,
12. current update-artifact hosting publishes exactly one current Probe binary and one current Node UF2 together with `version.json` manifests containing `version` and `crc32`,
13. delayed download serving is currently based on timestamp cursoring plus a 1.1× interval-derived cutoff,
14. the current Collector advances by newest valid timestamp, not by `item_id`,
15. the current CLI accepts `run_command(...)`, not the documented `command(...)` alias,
16. ordinary command routing is queue-based and minimally validated at the HUB, so Probe-side safe ignoring remains essential,
17. the current Probe/HUB/Collector/CLI/Update-Server surfaces contain important discrepancies that documentation must keep explicit,
18. Probe-generated `TM8` version lines are part of the current telemetry stream semantics,
19. the simulator analyzer is formally downstream of this telemetry pipeline and should be read as a telemetry consumer rather than a second source of telemetry architecture.

## Review Notes

Post-change review against `moonblokz-info` documentation rules:

- **Consistency:** This document now describes the reviewed telemetry flow from station side, cloud side, collector side, operator CLI side, and update-artifact-hosting side rather than leaving the update host at article-only level.
- **Logical soundness:** The file clearly separates what the reviewed code actually implements today from older spec/doc expectations, including the global HUB-owned interval policy, timestamp-based collector cursoring, the CLI’s actual parser and `/command` submission behavior, and the Update Server’s actual role as a static Spin fileserver.
- **Feasibility:** The described algorithms match the current code: bounded buffering, substring filtering, serial reconnect with recovery attempts, explicit SQLite/KV persistence, global interval policy, delayed log serving, timestamp-cursor-based collection, local CLI validation, hub-mediated command submission, static artifact publication, hourly update checks, and version-reporting telemetry injection.
