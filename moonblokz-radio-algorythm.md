# MoonBlokz Radio Algorithm Model

## Purpose of This Document

This document provides a formal, algorithm-oriented description of the current MoonBlokz radio behavior as implemented in `moonblokz-radio-lib`.

Its purpose is to capture:

- the formal runtime pipeline used by the library,
- the current message and packet model,
- the actual implemented message set,
- the echo-mapping and connection-matrix update behavior,
- the TX scheduling and radio-device control rules,
- the RX reassembly, duplicate suppression, and recovery flows,
- the wait-pool scoring and relay timing rules,
- and the exact limits of what the current codebase still does not define.

This file is the **primary knowledge-base document for the current formal radio behavior**.

- Use [moonblokz-radio-concept.md](./moonblokz-radio-concept.md) for the higher-level explanation of goals, trade-offs, and architectural meaning.
- Use [moonblokz-radio-implementation.md](./moonblokz-radio-implementation.md) for public API details, feature flags, backend behavior, memory profiles, and engineering implications.

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

Where article-era descriptions and the current code differ, this file treats the code as authoritative.

## Scope and Limits

This file captures the algorithmic structure explicitly visible in the current library, including:

- compile-time feature gates that shape behavior,
- the queue-backed runtime pipeline,
- current message-type definitions,
- packet fragmentation and selective retransmission behavior,
- connection-matrix maintenance,
- adaptive relay scoring,
- query/reply handling,
- duplicate suppression,
- packet- and message-level CRC validation,
- and reactive fragment recovery.

It does **not** attempt to define:

- the full higher-level blockchain consensus algorithm,
- the simulator’s full network model,
- undocumented future message types,
- or any final protocol evolution not currently visible in the code.

## Core Terminology Used in This Document

To stay aligned with the companion radio files, this document uses the following terms consistently:

- **RadioMessage** — the higher-level logical message representation.
- **RadioPacket** — the fixed-size wire-level transmission unit.
- **Connection matrix** — the local directional link-quality matrix.
- **Wait pool** — the bounded delayed-relay store.
- **Message connections** — the best-known propagated coverage vector attached to a wait-pool message.
- **Relay score** — the score indicating how much this node can improve message reach.
- **Activation time** — the future instant when a queued relay becomes eligible for transmission.

## Algorithmic Problem Statement

The current radio library must solve the following combined problem:

1. accept logical application messages,
2. convert them into radio packets when necessary,
3. coordinate physical send/receive/CAD behavior through a single radio device,
4. reconstruct multi-packet messages on the receive side,
5. detect duplicates and already-known content,
6. consult application-layer processing outcomes,
7. relay or reply only when the node adds useful value,
8. recover missing fragments reactively,
9. and do all of this with bounded queues and deterministic resource use.

## Section A — Compile-Time and Runtime Structural Rules

## A1. Feature exclusivity rules

Exactly one radio-device feature must be active:

- `radio-device-echo`
- `radio-device-rp-lora-sx1262`
- `radio-device-simulator`

Exactly one memory configuration feature must be active:

- `memory-config-small`
- `memory-config-medium`
- `memory-config-large`

Compilation fails if these exclusivity rules are violated.

## A2. Core compatibility constants

The library defines these protocol-visible size constants:

- `RADIO_PACKET_SIZE = 215`
- `RADIO_MAX_MESSAGE_SIZE = 2013`
- `RADIO_MULTI_PACKET_MESSAGE_HEADER_SIZE = 13`
- `RADIO_MULTI_PACKET_PACKET_HEADER_SIZE = 15`
- `RADIO_MAX_PACKET_COUNT = ceil((RADIO_MAX_MESSAGE_SIZE - 13) / (RADIO_PACKET_SIZE - 15))`

These values directly shape fragmentation and interoperability.

## A3. Memory-profile-dependent structural limits

The library currently uses compile-time memory profiles, and those profiles change the capacities of the connection matrix, packet buffers, wait pools, duplicate cache, and runtime queues.

That fact is algorithmically important because it means some behavior is capacity-shaped at build time rather than fully dynamic at runtime.

However, the exact per-profile capacities are implementation-facing configuration data rather than core algorithm flow. For the current concrete capacities in the small, medium, and large profiles, use [moonblokz-radio-implementation.md](./moonblokz-radio-implementation.md).

The RX state queue size is currently fixed at 20.

## Section B — Public Runtime Pipeline

## B1. Task model

The library spawns three async tasks:

1. `radio_device_task`
2. `tx_scheduler_task`
3. `rx_handler_task`

These tasks are connected by channels created during `RadioCommunicationManager::initialize`.

## B2. Initialization algorithm

`RadioCommunicationManager::initialize_common` performs the following steps:

1. receive a `RadioConfiguration`, `Spawner`, `RadioDevice`, node ID, and RNG seed,
2. destructure configuration values,
3. spawn the radio device task,
4. spawn the TX scheduler task,
5. spawn the RX handler task,
6. store channel handles in manager state,
7. transition the manager from `Uninitialized` to `Initialized`.

If any task spawn fails, initialization fails with a specific `RadioInitError`.

## B3. Application-facing runtime flow

### Sending

1. application calls `send_message(message)`,
2. the message is inserted into `outgoing_message_queue` via `try_send`,
3. if the queue is full, the call fails with `SendMessageError::ChannelFull`.

### Receiving

1. application awaits `receive_message()`,
2. receives either:
   - `IncomingMessageItem::NewMessage(RadioMessage)`
   - `IncomingMessageItem::CheckIfAlreadyHaveMessage(message_type, sequence, payload_checksum)`

### Reporting result

1. application calls `report_message_processing_status(result)`,
2. result is enqueued into `process_result_queue`,
3. RX handler and relay manager consume it to decide relay/reply behavior.

## Section C — Current Message Model

## C1. Implemented message types

The current `MessageType` enum contains exactly these values:

1. `RequestEcho`
2. `Echo`
3. `EchoResult`
4. `RequestFullBlock`
5. `RequestBlockPart`
6. `AddBlock`
7. `AddTransaction`
8. `RequestNewMempoolItem`
9. `Support`

## C2. Message header rules

All messages begin with:

- byte 0: message type
- bytes 1..5: sender node ID

Some message types also carry:

- bytes 5..9: sequence or anchor sequence
- bytes 9..13: payload checksum or sender-specific extra field depending on type

## C3. Packet header rules

All packets begin with the same logical message header.

For multi-packet messages (`AddBlock`, `AddTransaction`), packet-local metadata follows:

- byte 13: total packet count
- byte 14: packet index

## C4. Single-packet vs multi-packet model

### Multi-packet messages in current code

Only these message types use the multi-packet path:

- `AddBlock`
- `AddTransaction`

### Single-packet messages in current code

All other message types are treated as single-packet messages.

Notably, the current codebase does **not** define a separate `RequestTransactionPart` message type yet, even though some earlier summaries described one. This should be read as a **not-yet-implemented** part of the protocol rather than as proof that the concept was abandoned.

## C5. Current constructor-level message semantics

### `RequestEcho`

Constructed with sender node ID only.

### `Echo`

Carries:

- sender node ID
- target node ID
- link quality

### `EchoResult`

Carries a list of items of the form:

- neighbor node ID
- send link quality
- receive link quality

Each item is 6 bytes and must fit in a single packet.

### `RequestFullBlock`

Carries:

- sender node ID
- requested block sequence

### `RequestBlockPart`

Carries:

- sender node ID
- block sequence
- payload checksum
- count of requested packet indices
- list of requested packet indices

### `AddBlock`

Carries:

- sender node ID
- sequence
- CRC32C payload checksum
- block payload bytes

### `AddTransaction`

Carries:

- sender node ID
- anchor sequence
- payload checksum
- transaction payload bytes

### `RequestNewMempoolItem`

Carries:

- sender node ID
- zero or more `(anchor_sequence, transaction_payload_checksum)` items

The constructor is currently named `get_mempool_state_with`, even though the message type is `RequestNewMempoolItem`.

### `Support`

Carries:

- sender node ID
- sequence
- supporter node ID
- signature bytes

## C6. Message reply relationships currently implemented

`RadioMessage::is_reply_to` currently defines these reply relationships:

- `Echo` replies to `RequestEcho` when sender node IDs match,
- `AddBlock` replies to `RequestFullBlock` when sequence matches,
- `AddTransaction` replies to `RequestNewMempoolItem` when the requested mempool list does **not** already contain that transaction reference.

No other message types currently participate in `is_reply_to` matching.

## Section D — TX Scheduler Algorithm

## D1. Main loop

The TX scheduler asynchronously selects between:

- next outgoing message from `outgoing_message_queue`,
- RX state updates from `rx_state_queue`.

## D2. RX-state-aware delay rule

The scheduler tracks two time constraints:

- time since last full message transmission,
- any receive-side delay timeout caused by in-progress foreign multi-packet reception.

Before sending a new message:

1. compute remaining inter-message delay,
2. compute remaining RX-state delay,
3. take the maximum of the two,
4. add random jitter up to `tx_maximum_random_delay`,
5. wait unless interrupted by another RX state update.

## D3. Packet send rule

Once a message is eligible:

1. compute its packet count,
2. iterate over each packet number,
3. obtain the packet via `message.get_packet(i)`,
4. try to send it into `tx_packet_queue`,
5. wait `delay_between_tx_packets` between packets.

If `tx_packet_queue` is full, that packet is dropped.

## D4. RX-state update interpretation

### `PacketedRxInProgress(packet_index, total_packet_count)`

The scheduler extends its TX delay by approximately the remaining packet count times the inter-packet delay.

### `PacketedRxEnded`

The scheduler clears the RX-induced delay immediately.

## Section E — Radio Device Layer Rules

## E1. Single-operation device invariant

The radio device layer assumes the backend can do only one of the following at a time:

- receive
- transmit
- CAD

## E2. Backend set currently implemented

The current codebase contains three backend implementations:

- `echo`
- `simulator`
- `rp_lora_sx1262`

## E3. Echo backend algorithm

1. wait for packet on TX queue,
2. immediately enqueue the same packet into RX queue with link quality 63,
3. if RX queue is full, drop the packet.

## E4. Simulator backend algorithm

1. concurrently listen for:
   - messages from the simulator input queue
   - packets from TX queue
2. if packet arrives from simulator, forward it to RX queue,
3. if packet arrives from TX queue:
   - request CAD from simulator,
   - if busy, wait `300 ms + random(0..200 ms)` and retry,
   - if clear, emit `SendPacket` to simulator.

## E5. RP2040 SX1262 backend algorithm

1. initialize SPI, pins, PHY state, and LoRa parameters,
2. enter main loop,
3. race between receive and outgoing packet availability,
4. on receive:
   - decode packet,
   - compute link quality from RSSI/SNR,
   - send `ReceivedPacket` to RX queue,
5. on transmit:
   - run CAD,
   - if busy, wait and retry,
   - if clear, transmit packet.

## E6. CAD timeout workaround

On the RP2040 SX1262 backend, CAD may occasionally never complete.

The current code applies a `CAD_TIMEOUT_MS = 1000` timeout and treats timeout as failure requiring retry.

## E7. Packet CRC rules

### Optional packet-level software CRC

When `soft-packet-crc` is enabled, the RP backend adds/checks a CRC-16-CCITT packet CRC in software.

### Full-message CRC

The message layer uses CRC32C for `AddBlock` and `AddTransaction` payload integrity.

## E8. Link quality calculation algorithm

The current code implements:

1. normalize RSSI from `[-120, -30]` to `[0, 63]`,
2. normalize SNR from `[-20, 10]` to `[0, 63]`,
3. compute `quality = (3 * norm_rssi + 7 * norm_snr) / 10`.

This yields a final `0..63` quality score.

## Section F — RX Handler Algorithm

## F1. Main event loop

The RX handler asynchronously selects among:

- `rx_packet_queue_receiver.receive()`
- `process_result_queue_receiver.receive()`
- timer for `relay_manager.calculate_next_timeout()`
- timer for `next_missing_packet_check`

## F2. Single-packet receive path

When a packet has `total_packet_count() == 1`:

1. build a `RadioMessage` using `RadioMessage::from_single_packet`,
2. call `process_message(...)` immediately.

## F3. Multi-packet receive path

When `total_packet_count() > 1`:

1. validate packet count against `INCOMING_PACKET_BUFFER_SIZE`,
2. send RX state updates to `rx_state_queue`,
3. notify relay manager through `process_received_packet`,
4. reject duplicate packet index if already buffered,
5. if this is the first packet of the message:
   - check the recent-message cache,
   - if unknown, send `CheckIfAlreadyHaveMessage` to the application queue,
6. if no empty packet-buffer slot exists, evict the oldest message group,
7. store packet with arrival timestamp,
8. check whether all packets for that message are now present,
9. if complete:
   - build a new `RadioMessage`,
   - add each packet via `add_packet`,
   - clear the fragments from the packet buffer,
   - set sender node ID,
   - call `process_message(...)`.

## F4. Packet-buffer eviction rule

If no empty slot is available:

1. find the buffered packet with the oldest arrival time,
2. identify its message by the first multi-packet header bytes,
3. remove all buffered packets belonging to that message,
4. reuse that slot for the new packet.

## F5. Recent-message duplicate cache rule

The RX handler tracks recently received messages in a circular buffer of:

- message type
- sequence
- payload checksum

This cache is checked for the first packet of new multi-packet messages.

## F6. Application duplicate-check rule

If the recent cache does not already know a multi-packet message:

1. extract `(message_type, sequence, payload_checksum)` from the packet,
2. send `IncomingMessageItem::CheckIfAlreadyHaveMessage(...)` to the application.

The application may later answer with `MessageProcessingResult::AlreadyHaveMessage(...)`, which causes all buffered fragments of that message to be dropped and the duplicate cache to be updated.

## F7. Message-level CRC validation rule

In `process_message(...)`, the RX side validates full payload CRC for:

- `AddBlock`
- `AddTransaction`

If `check_payload_checksum()` fails, the message is dropped.

## F8. Relay-manager interaction rule

After CRC validation:

1. call `relay_manager.process_received_message(message, link_quality)`,
2. interpret result:
   - `None` → no immediate relay action,
   - `SendMessage(reply_or_generated_message)` → send to outgoing queue,
   - `AlreadyHaveMessage` → do not forward to application.

## F9. Application delivery rule

If the message type is one of:

- `AddBlock`
- `RequestFullBlock`
- `RequestBlockPart`
- `AddTransaction`
- `RequestNewMempoolItem`
- `Support`

and relay manager did not declare it already known, enqueue `IncomingMessageItem::NewMessage(message)` for the application.

## F10. Processing-result feedback rule

When a `MessageProcessingResult` arrives:

- `NewBlockAdded` and `NewTransactionAdded` are stored in the recent-message cache,
- `AlreadyHaveMessage` clears buffered fragments and updates recent-message cache,
- all other results are forwarded into `relay_manager.process_processing_result(...)`.

## F11. Missing-fragment scan rule

At each retry interval:

1. inspect buffered incomplete messages,
2. select the incomplete message whose newest fragment is older than the retry interval and is the most recent among eligible candidates,
3. if that message is an `AddBlock`, construct `RequestBlockPart`,
4. enumerate all missing packet indices using the packet-check buffer,
5. enqueue the request into `outgoing_message_queue`.

The current code performs this explicit missing-fragment scan only for `AddBlock` recovery.

## Section G — Relay Manager Algorithm

## G1. Core state

`RelayManager` maintains:

- `connection_matrix`
- `connection_matrix_nodes`
- `connected_nodes_count`
- `wait_pool`
- `echo_responses_wait_pool`
- `next_echo_request_time`
- optional `echo_gathering_end_time`
- echo timing parameters
- own node ID
- RNG state

## G2. Connection matrix encoding

Each matrix cell is a `u8`:

- lower 6 bits = link quality
- upper 2 bits = dirty counter

`MAX_DIRTY_COUNT = 3`.

## G3. Node tracking rule

`connection_matrix_nodes` maps matrix indices to node IDs.

If a new node is discovered when the matrix is full, the library may replace the currently weakest sender-visible node if the new node’s most recent quality is stronger.

## G4. Timed task rule

`RelayManager::process_timed_tasks()` performs four possible actions, in order:

1. periodic echo request generation,
2. echo gathering completion and `EchoResult` generation,
3. activated wait-pool relay emission,
4. due echo-response emission.

## G5. Periodic echo request algorithm

When `Instant::now() >= next_echo_request_time`:

1. compute adaptive echo interval based on visible node count,
2. open a new echo gathering window ending at `now + echo_gathering_timeout`,
3. increment dirty counters for all non-diagonal non-zero matrix entries,
4. zero entries whose dirty counter exceeds `MAX_DIRTY_COUNT`,
5. schedule the next echo request time,
6. emit `RadioMessage::request_echo_with(own_node_id)`.

## G6. Echo gathering completion algorithm

When echo gathering timeout expires:

1. close the gathering window,
2. create empty `EchoResult`,
3. for every known neighbor other than self:
   - include it only if either directional dirty counter is zero,
   - include it only if at least one directional quality is non-zero,
4. stop if the single-packet `EchoResult` becomes full,
5. emit the `EchoResult`.

## G7. Echo response buffering algorithm

When handling `RequestEcho`:

1. create delayed `EchoResponseWaitPoolItem` with target node and measured quality,
2. if a free slot exists, store it,
3. otherwise replace the oldest pending echo response and immediately emit the displaced one.

## G8. Processing ordinary received packets

`process_received_packet(packet, link_quality)`:

1. find sender index if known,
2. update sender→self quality in the connection matrix,
3. pass sender connection row to wait pool for possible score updates on matching messages.

## G9. Processing received messages

`process_received_message(message, last_link_quality)`:

1. ensure sender exists in connection matrix, adding or replacing if necessary,
2. if `RequestEcho`:
   - queue an echo response and possibly emit a displaced response,
3. if `Echo`:
   - update sender→self and target→sender qualities,
4. if `EchoResult`:
   - integrate listed bidirectional qualities and add new nodes if there is room,
5. if the message is one of the relay-relevant types and already exists in wait pool or as a reply relation:
   - update its wait-pool scoring,
   - return `AlreadyHaveMessage`,
6. otherwise return `None`.

## G10. Processing application results

`process_processing_result(result)` currently handles:

- `RequestedBlockNotFound(sequence)` → enqueue a new `RequestFullBlock`
- `RequestedBlockFound(message)` → enqueue found block for relay
- `RequestedBlockPartsFound(message, requestor_node)` → enqueue a targeted reply if requestor is known
- `NewBlockAdded(message)` → enqueue block for relay using sender connections if known
- `NewTransactionAdded(message)` → enqueue transaction for relay
- `SendReplyTransaction(message)` → enqueue transaction reply
- `NewSupportAdded(message)` → enqueue support message for relay
- `AlreadyHaveMessage(...)` → handled by RX handler, not relay manager

## Section H — Wait Pool Algorithm

## H1. Wait-pool item state

Each `WaitPoolItem` stores:

- `message`
- `activation_time`
- `message_connections`
- `requestor_index`

## H2. Link-quality class rule

The scoring logic classifies connection qualities into four categories:

- `0` = Zero
- `1` = Poor (`0 < quality < poor_limit`)
- `2` = Fair (`poor_limit <= quality < excellent_limit`)
- `3` = Excellent (`quality >= excellent_limit`)

## H3. Scoring matrix rule

`ScoringMatrix` contains:

- a 4×4 score matrix,
- `poor_limit`,
- `excellent_limit`,
- `relay_score_limit`.

The code also supports compact decoding from a 5-byte packed representation.

## H4. Relay score calculation rule

For each message in the wait pool:

1. compare this node’s connections against `message_connections`,
2. map both sides into quality classes,
3. sum the category-to-category scores from the scoring matrix.

If `requestor_index` is set, score only the requestor connection.

## H5. Position calculation rule

To compute own relay position:

1. calculate own score,
2. for every other known node that likely received the message (`item_connections[i] > poor_limit`):
   - calculate that node’s score using the full connection matrix,
   - if that score is greater than own score, increment own position.

## H6. Adding a new wait-pool message

To add a new message:

1. initialize `message_connections` from the sender’s connection row,
2. force `message_connections[0] = 63` to indicate self has the message,
3. calculate own score,
4. if score is below `relay_score_limit` and this is not a targeted requestor message, discard it,
5. if score is zero, discard it,
6. calculate own position,
7. set activation time to:

`now + position * relay_position_delay + random_jitter`

8. insert into empty slot if available,
9. otherwise replace the currently weakest-scoring item only if the new message scores higher.

## H7. Updating an existing wait-pool message

When a message already in the pool is seen again:

1. merge sender connection qualities into `message_connections` using per-node max,
2. recalculate own score,
3. if score falls below threshold (or zero for targeted messages), remove it,
4. otherwise recalculate position,
5. update activation time accordingly.

## H8. Activation rule

A message becomes relay-ready when `Instant::now() >= activation_time`.

`get_next_activated_message()` removes and returns the first ready item found.

## H9. Connection-matrix replacement rule

If a tracked node is removed from the connection matrix, the wait pool clears that index from all `message_connections`.

If the removed node was a targeted requestor for an item, that wait-pool item is removed entirely.

## Relationship to the Radio Simulator Documents

Part VII/4 of the MoonBlokz series adds a network-level validation perspective to the formal radio behavior documented here.

In particular, the simulator article reinforces that the algorithm described in this file is intended to be observed under:

- topology-dependent propagation,
- contention and collisions,
- selective relay dominance by better-positioned nodes,
- saturated-region relay suppression,
- and explicit missing-fragment recovery in larger network scenarios.

Those observations do not replace the code-grounded algorithm described here, but they do show how it behaves when many node instances run together.

For that companion view, continue with:

- [moonblokz-simulator-concept.md](./moonblokz-simulator-concept.md)
- [moonblokz-simulator-algorythm.md](./moonblokz-simulator-algorythm.md)
- [moonblokz-simulator-implementation.md](./moonblokz-simulator-implementation.md)

## Main Algorithmic Conclusions

From the perspective of the MoonBlokz knowledge base, the current codebase establishes these formal conclusions:

1. the radio library is built as three queue-connected async tasks,
2. exactly nine message types are currently implemented,
3. only `AddBlock` and `AddTransaction` are fragmented into multiple packets,
4. duplicate suppression is application-assisted and uses both a recent-message cache and higher-layer duplicate checks,
5. relay timing is score-based, delayed, and dynamically updated,
6. connection quality uses a 0–63 directional metric carried through the connection matrix,
7. missing-fragment recovery is explicit and currently implemented for block fragments,
8. application processing results directly influence relay and reply behavior,
9. bounded queues and bounded pools are part of the algorithm, not only implementation detail.

## Review Notes

Post-change review against `moonblokz-info` documentation rules:

- **Consistency:** This document now matches the current `moonblokz-radio-lib` codebase rather than relying only on article-era radio summaries.
- **Logical soundness:** The file is careful to distinguish what is actually implemented in code from what earlier descriptions may have suggested, especially around message types and fragment-recovery behavior.
- **Feasibility:** The described algorithms remain realistic for constrained hardware: fixed-size structures, explicit timing, bounded relay competition, and reactive gap repair.
