# MoonBlokz Blockchain Implementation Notes

## Purpose of This Document

This document captures implementation-facing implications of the MoonBlokz blockchain model introduced in Part III and extended in Part IV and Part V of the MoonBlokz series. It is not a full implementation specification. Instead, it complements the conceptual and algorithm documents by identifying what an implementation will need to track, where configuration boundaries exist, how compact binary structures affect engineering choices, and which details must remain open until later articles or repository decisions define them.

- Use [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) as the **authoritative source** for every FR-numbered functional requirement and Non-Functional Requirement cited in this document; FR references in the implementation guidance resolve to the canonical wording in the PRD.
- Use [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) as the **authoritative source** for the chain-lib crate-split, public API, internal modules, sized data-structure layouts, RAM budget, stack-frame analysis, FR-coverage matrix, and decisions log. This document defers to the Architecture Decision Document for all concrete implementation-shape specifics.
- Use [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) for the conceptual description.
- Use [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) for the algorithm model and the detailed main data structures.
- Use [`blockchain-adrs/ADR-INDEX.md`](./blockchain-adrs/ADR-INDEX.md) for the current accepted blockchain architecture decision set and its reading order.

## Source Basis

This document is based on:

- **MoonBlokz series part III. — Basic Algorithms** by Peter Sallai, published on Medium on March 4, 2025.
- **MoonBlokz series part IV. — snake_chain** by Peter Sallai, published on Medium on March 13, 2025.
- **MoonBlokz series part V. — Data Structures** by Peter Sallai, published on Medium on March 26, 2025.

## Scope and Intent

Parts III, IV, and V together provide a strong implementation direction, but they still do not define a complete engineering spec. Because of that, this document has a narrower purpose:

- identify implementation responsibilities,
- highlight data that must be persisted or derived,
- point out what should remain configurable,
- show where bounded storage changes the design,
- show where binary layout and packetization change the design,
- and warn against hard-coding assumptions that the articles have not finalized.

## Relationship to Part II Architecture

Part II established that MoonBlokz is built around a portable core library with clear integration boundaries. Parts III, IV, and V add blockchain behavior on top of that structure.

## Current Module Boundary Direction

`moonblokz-blockchain` is a **single-threaded, synchronous, stateful semantic event state machine** with the responsibility partition between owned concerns (blockchain decisions, staged validation, block-tree state, active-chain selection, mempool, vote, query answers) and explicitly non-owned concerns (radio transport, fragment handling, wire-format serialization adapters, scoring-module input, storage mechanics, cryptographic backend implementation).

The authoritative definition of this boundary — including the trait-based handles for crypto / storage / chain-config, the bridge layer (`moonblokz-node-runtime`), the two-core split on RP2040, the crate-split into `moonblokz-blockchain` + `moonblokz-mempool` + `moonblokz-vote`, and the single-outcome scheduling-pull API pattern — lives in [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §1 (framework) and §2 (crate-split).

The implementation-facing consequences below remain relevant where they capture engineering cautions and source-article bridging that the Architecture Decision Document does not restate.

## Implementation-Relevant Constraints from Part V

Part V adds three explicit engineering constraints that should shape design decisions from the start.

### 1. Radio messages are smaller than many blockchain objects

Blocks and especially complex transactions may exceed radio packet size. The implementation therefore needs explicit fragmentation and reassembly support outside the logical blockchain data model.

### 2. RAM is fast but severely limited

The article assumes only a few hundred kilobytes of device memory, much of it needed for stack usage. This means:

- caches must be bounded,
- working sets must stay small,
- and parsing or validation should avoid large transient allocations.

### 3. Persistent storage is slower and wear-limited

Flash storage has finite write cycles. This means the implementation should minimize write frequency and avoid treating persistent state updates as cheap.

The article explicitly suggests that specialized storage structures such as B-trees and in-memory caches will be discussed separately. Even before those later details, the constraint itself is already binding.

## Processing-State Crash Semantics

The **processing** phase is not merely incremental block intake. It may be a reconstruction pass over a candidate chain. If that pass is interrupted, for example by restart, the implementation must decide whether partially rebuilt state is resumable or disposable.

For now, the conservative interpretation is:

- processing-state restart behavior should be treated as an explicit policy,
- a full restart of reconstruction is an acceptable current simplification,
- resumable processing should be treated as future enhancement rather than assumed capability.

## Failed Full-Chain Validation Recovery

Failure during full-chain validation in the processing phase is a distinct situation from restart. The chain has been traversed and the reconstruction failed against a rule that is already enforceable at the reached position.

Recovery combines two scoped actions that must be applied together, not substituted for each other:

1. **Reconstruction working-set rollback** — the derived state accumulated during the forward traversal is discarded in full and atomically, regardless of the durable-deletion path chosen below. This always applies and restores the module to a clean state before returning to collecting.
2. **Durable-storage deletion under the ADR-004 processing-phase deletion path** — the block proven invalid by the re-execution is removed from durable storage as well. If the specific offending block can be identified from the failure signal, only that block is deleted. If it cannot be precisely identified, the fallback is to delete **exactly one block — the candidate chain's head**; repeating this single-block drop across subsequent collecting → processing cycles eventually expels the offender. There is no configurable fallback depth. Every transitive descendant of a deleted block — every block whose `previous_hash` chain passes through it — is also deleted, since a block has exactly one parent link and that link cannot be re-routed (the `previous_hash` is immutable). This explicitly includes every block on the candidate chain between the offender and the pre-failure tip when the offender is precisely identified. Sibling subtrees on other branches of the block-tree (blocks whose `previous_hash` chain does not pass through any deleted block) are unaffected and remain in storage.

After these actions, the module returns to collecting state and resumes dominant-chain acquisition on the remaining retained history.

The durable deletion is required for forward progress, not optional cleanup. If the offending block remained in durable storage, the next collecting iteration would re-select the same candidate tip by the dominant-chain acquisition rule, re-enter processing, fail the same invariant at the same position, and loop indefinitely between collecting and processing. Removing the offender (or, in the fallback case, lowering the tip one block at a time) is the only mechanism that breaks this livelock. The loss of a block that might in principle be valid on another branch is bounded and acceptable because the radio layer can re-disseminate blocks on demand; an unbreakable livelock would not be acceptable.

Discarding the working-set in full is preferred for atomicity and simplicity: working-set state and durable storage are separate layers, so a full working-set reset does not remove durable blocks beyond those identified by the deletion path. The single-block durable fallback keeps recovery bounded and predictable without introducing a configuration surface whose correct value would be hard to reason about in practice.

This recovery path is distinct from the intake-time persistence threshold of ADR-004: intake remains permissive, and durable deletion here is authorized only by the processing-phase "proven invalid" evidence produced by the re-execution itself.

## Authoritative Versus Derived State

The current module design direction keeps the authoritative state deliberately small.

The canonical conceptual explanation of authoritative versus derived blockchain state belongs in [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) and is formalized in [ADR-003](./blockchain-adrs/ADR-003-authoritative-vs-derived-state-in-moonblokz-blockchain.md).

From an implementation perspective, the key consequence is that restart, persistence, and recovery behavior should be anchored in a compact authoritative base, while selected active-chain state, balance / UTXO truth, and vote / next-creator state should be engineered as maintained operational views that can be corrected or recomputed when necessary.

## In-Memory State Categories

The concrete in-memory layouts, field-by-field sizes, alignment, sentinels, and module-by-module owner of every authoritative and derived state category are defined in [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md):

- **Block-tree metadata** → architecture §4.2 (`blocks.rs`) and §6.2 (`BlockEntry` 76 B padded × `MAX_BLOCKS`, co-located ADR-016 spent-bit vector).
- **Active-chain window state** → architecture §4.2 (`snake_chain.rs`) and §6.4 (two `u32` fields directly on `Blockchain<...>`).
- **Branch / chain-heads tracking** → architecture §4.2 (`chain_heads.rs`) and §6.3 (`ChainHeadEntry` 32 B padded × `MAX_BRANCH_COUNT`).
- **Per-node SoA state (public keys, balances, FR50 seed-source projection, max_known_node_id)** → architecture §4.2 (`node_info.rs`) and §6.1.
- **UTXO spent-bit projection** → architecture §4.2 (`spent_bits.rs`) and §6.2 (co-located in `BlockEntry`).
- **Approval evidence accumulation** → architecture §4.2 (`approval.rs`) and §6.5 (`ApprovalAccumulator` ~2 KB crypto-agnostic MAX_BLOCK_SIZE buffer).
- **Mempool** → architecture §3.3 (sub-crate API) and §6.8 (internals).
- **Vote engine** → architecture §3.4 (sub-crate API) and §6.9 (internals).
- **Scheduler / `NextCall` deadlines** → architecture §4.2 (`scheduler.rs`) and §6.7 (`SchedulerState` ~72 B with `Option<u64>` deadlines).
- **Emit scratch / outcome view source** → architecture §4.2 (`emit_scratch.rs`) and §6.6.

The bridging principle that ADR-004 establishes — that durable storage is permissive during collection and processing and remains permissive in ready state except when intake-time exact evidence is already present — is preserved verbatim in the architecture's intake module (§4.2 `intake.rs`). The intake-time exact-evidence forms (parsing failure; direct-active-extension creator-signature invalidity when `previous_hash = active_chain_head_hash` and the creator key is derivable from the current active chain; chain-config content-signature invalidity against `node_zero_public_key`; configuration-module rejection of chain-config content; ready-state chain-config content mismatch against the durable-locked configuration; deviation-branch creator-exclusivity violation; FR69 node-zero trust-anchor mismatches on node-`#0` registration, balance-block NodeInfo, or chain-config signature material) map onto `Rejected(RejectReason)` variants of the `ReceiveBlockOutcome` enum per PRD FR10's Implementation annotation.

The scoring-module boundary remains an external concern: the blockchain module does not determine the transaction's vote target (vote-target selection) from raw radio observation; that is the scoring module's responsibility. The scoring module supplies the vote target written into the locally assembled signed transaction before the blockchain module sees it; the chain-lib vote sub-crate (`moonblokz-vote`, architecture §3.4) owns only the per-node **accumulated vote**, the per-acceptance `vote_scale` credit rule, the integer anti-capture growth `accumulated_vote += floor(accumulated_vote × vote_interest / vote_scale)`, the creator reset on successful block creation, the grace-period reset policy, and the next-creator determination.

The serialized-byte fidelity requirement remains stronger after Part V because signatures and hashes depend on exact byte layout — see the "Serialization and Encoding Implications" section below for the canonical-serialization engineering consequences that the architecture does not restate.

## Data Modeling Implications

Parts III, IV, and V do not define concrete structs or persistence schemas, but they imply several strong modeling constraints.

### Tree-first rather than chain-first storage

Any implementation that stores only the currently preferred chain would be incompatible with the source material. Forks, disconnected blocks, and later ancestor recovery are all first-class concerns.

### Windowed-chain rather than immutable-full-history assumptions

Any implementation that assumes all validation must always be based on full historical replay will conflict with `snake_chain`.

### Lossless binary representation matters

Because hashes, signatures, size-based fees, and radio fragmentation all depend on exact binary form, implementations should preserve either:

- canonical serialized bytes for each block and transaction,
- or a rigorously defined serializer that can reproduce exactly the same byte stream.

Ad hoc in-memory representations that cannot guarantee byte-for-byte stability would be risky.

### Variable-size transaction handling must stay bounded

Complex transactions have optional and repeated elements. This means parsing, validation, and caching logic must avoid unbounded recursion or allocation-heavy expansion.

### Approval evidence is part of durable chain state

Approval support cannot remain an ephemeral network-only concept if later nodes must understand why a branch won.

### Replay provenance should remain auditable

Part IV strongly suggests that the system will need to explain why replay content exists:

- which dropped block caused the replay,
- which balances were updated,
- which UTXOs were carried forward,
- and how custodian-fee reductions were applied.

The article does not specify an explicit provenance format, but an implementation that loses this traceability may become very difficult to debug.

## Serialization and Encoding Implications

Part V explicitly states that blocks are stored and transferred in binary form with little-endian encoding for all multi-byte fields.

### Engineering consequences

- serialization must be deterministic across all supported platforms,
- signature code must operate on canonical byte ranges,
- hash code must use the exact canonical serialization,
- parser code must be version-aware from the first byte-level boundary,
- and tests should include byte-level fixtures, not only semantic round-trips.

### Versioning implications

Because the first header field is `version`, parser code should be designed to:

- reject unsupported future versions cleanly,
- avoid ambiguous decoding paths,
- and preserve forwards-compatibility boundaries.

## Packetization Boundary

Part V explicitly says radio packetization belongs in the radio layer, not in the logical blockchain structure.

### Practical implication

A clean architecture should separate:

- **logical blockchain unit**: full serialized block,
- **transport unit**: radio packet fragment,
- **reassembly buffer state**: fragment-tracking state before a complete block is available,
- **validation boundary**: parse and validate only after full deterministic reassembly.

### Why this matters

If parsing or partial validation leaks too early into fragment handling, the implementation may become vulnerable to:

- fragment-order bugs,
- duplicate-fragment corner cases,
- malformed partial-state handling,
- and wasted CPU or RAM on invalid incomplete data.

The article does not define the exact fragmentation format yet, so this boundary should stay explicit and modular.

## Pruning Cost Is Not Negligible

The todo material suggests that branch deletion under storage pressure is not a trivial bookkeeping step.

Implementation-wise, pruning may require:

- repeated persistent-storage reads and deletes,
- branch-end or equivalent index maintenance,
- parent/child relationship updates,
- follow-up consistency checks after each removed block or removed segment.

Pruning policy therefore cannot be evaluated only by counting removed blocks. The implementation also needs to consider I/O cost and metadata churn.

## Caching and Flash Considerations

Part V directly motivates implementation techniques such as in-memory caching and flash-aware data structures.

## Mempool-Specific Engineering Notes

The current design direction treats the mempool as authoritative runtime state but not as durable blockchain truth. That means mempool contents may be lost across restart, active-chain changes must be allowed to remove now-confirmed transactions from the mempool, and transactions that fall out of the active chain during a chain switch may need to be reintroduced into the mempool. When mempool capacity is exhausted, randomized eviction (per [ADR-010](./blockchain-adrs/ADR-010-randomized-mempool-eviction-under-capacity-pressure.md)) is preferable to deterministic eviction because it increases the chance that the network as a whole retains a more diverse transaction set.

The compact-storage layout, the index entry shape, the eligibility iterator, the deferred-flag handling, the byte-buffer compaction strategy, the CRC32-based duplicate detection, the FR43 sub-seeded PRNG for random eviction with own-transaction prioritization, and the full sub-crate API surface are defined in [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §3.3 (`Mempool<...>` API, 10 method-groups) and §6.8 (internals: compact buffer ~20 KB + 128 index entries + sub-seed PRNG ≈ 21.5 KB total).

### RAM-side expectations

The implementation should expect to cache only a subset of:

- recent blocks,
- branch tips,
- balance snapshots,
- UTXO lookup hot sets,
- and partial fragment reassembly state.

### Flash-side expectations

Persistent storage likely needs:

- append-friendly or locality-aware write patterns,
- indexing support for block hash and transaction-hash lookup,
- bounded rewrite behavior,
- and careful separation between hot mutable indexes and colder immutable block data.

The article mentions B-trees as a likely implementation direction for flash storage, but it does not yet define the final storage design. That should remain an open architectural decision until the relevant module article or repository implementation makes it concrete.

## Retention Size vs. Revalidation Cost

The todo material sharpens an implementation tradeoff that is only implicit in the article series.

If the node retains only a narrow history window beyond the currently active chain, it saves storage but increases the chance that a later active-chain switch will require broader revalidation or near-full recomputation.

If the node retains a larger verification horizon, such as roughly an extra chain-length worth of history, future active-chain switches may become cheaper to validate but the storage footprint increases.

This tradeoff should be documented explicitly because it affects:

- storage sizing,
- pruning policy,
- validation latency after branch changes,
- whether active-chain switching feels like a lightweight update or a heavyweight recovery event.

## Configuration Boundaries Suggested by Parts III, IV, and V

The combined articles explicitly or implicitly suggest that several values should remain configurable at chain level or later design level. The chain-lib interface to those values is the `ChainConfigTrait` outlined in [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §11; the state and execution model behind the trait lives in the future `moonblokz-configuration` crate, which will receive its own BMAD covering FR7, FR8, FR17, FR49, and FR56. The categories below preserve the conceptual rationale for what must remain configurable; the concrete trait surface and its delegation contract belong to the Architecture Decision Document.

### Formula execution boundary

If chain configuration eventually contains formulas, the open question is whether they are evaluated by a constrained expression evaluator or by a more general virtual machine. The current architecture defers this decision to the future `moonblokz-configuration` crate (FR56 mini-VM capability is the placeholder term). Whichever mechanism is chosen must preserve the boundary conditions any such evaluator would have to satisfy:

- deterministic behavior across all nodes,
- bounded runtime,
- bounded memory usage,
- explicit and auditable available inputs,
- no hidden dependence on wall-clock time or external services,
- clear failure behavior for invalid expressions or unsupported operations.

A restricted expression evaluator is easier to reason about and validate. A virtual machine offers more flexibility but introduces a much larger correctness and resource-control surface.

### Economic configuration

The articles say monetary behavior may depend on chain-level configuration, including:

- transaction-fee policy,
- registration fee,
- custodian fee,
- additional currency creation or consumption,
- and economic functions based on network-wide parameters such as actual node count, total currency, and average transaction size.

### Timing configuration

The following are implied configuration candidates:

- enough time has passed to force block creation,
- transactional block-fill threshold percentage of `MAX_BLOCK_SIZE`,
- grace-period duration,
- approval timing windows,
- mempool-replenishment request interval (used continuously whenever the local mempool is below the transactional fill threshold, independently of whether the local node is the currently expected creator; see Algorithm 4 mempool replenishment in `moonblokz-blockchain-algorythm.md`),
- and retention-related scheduling margins.

The todo material also suggests that some timing behavior may later depend on runtime-derived metrics such as average block time or capped grace-time formulas.

If that direction is adopted, the implementation will need to define:

- which history window contributes to the average,
- whether the metric is computed from active-chain data only,
- how aggressively outliers are filtered,
- and how formula inputs remain deterministic across nodes with imperfect history overlap.

### Retention and sizing configuration

Parts IV and V introduce or imply:

- active chain length,
- maximum block size,
- radio packet size as a compile-time or module parameter,
- and the `n`-block lookahead used when compacting re-added UTXOs.

### Registration policy configuration

The implementation must be able to represent at least the options explicitly mentioned by the article:

- registration price,
- self-registration enablement,
- and permissioned registration by node `#0`.

### Majority and support thresholds

Approval depends on a predetermined number of support messages or majority conditions. These thresholds must be explicit rather than hidden inside ad hoc logic.

Per [ADR-015](./blockchain-adrs/ADR-015-approval-subgroup-selection.md), chain-config validation must reject `required_support` values that exceed the active crypto backend's `MAX_AGGREGATED_SIGNATURES`, because only the actually signing supporters enter the evidence block. The subgroup size `m` is then derived as `min(2 · required_support − 1, |A|)` and does not carry a direct crypto cap of its own.

### Anti-capture parameter

The vote-interest mechanism is parameterized by two configuration values: `vote_scale` (the numeric value of one received vote credit and the denominator of the anti-capture rule) and `vote_interest` (the per-block growth numerator). Anti-capture growth then uses integer arithmetic: `accumulated_vote += floor(accumulated_vote × vote_interest / vote_scale)`.

## Genesis and Bootstrap Implications

Part IV defines a special two-block bootstrap (block `#0` with node-`#0` registration and initial self-transfer; block `#1` with chain configuration), and Part V reinforces that those blocks use the same byte-structured blockchain model with some special validation exceptions. This must be treated as explicit startup logic rather than a quirky exception hidden in general validation code.

The bootstrap is realized in [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §3.6 as three separate `initialize_*` constructors (`initialize_genesis`, `initialize_join`, `initialize_from_storage`). The genesis path emits block `#0` as the outcome of `initialize_genesis` and block `#1` on the next `on_tick` via `TickOutcome::GenesisChainConfigCreated` — the single-outcome scheduling-pull pattern naturally splits the two-block sequence across two calls.

## Active-Chain Switch Consequences

Active-chain switch is not just a matter of choosing a new tip hash. It is a rare but first-class correction event: the Chain Knowledge Core remains the orchestrating truth source while derived subsystems (per-node accumulated vote and creator-order state, node-derived active balances, branch bookkeeping, mempool eligibility) are reconciled against the newly selected branch. A practical strategy is to walk backward to the common ancestor and then forward along the new active branch while updating derived state incrementally — chain switch is a structured recomputation workflow rather than a side effect attached to tip replacement.

The concrete reconciliation workflow (backward walk → forward walk; spent-bits clear/replay; balance rollback/replay; accumulated-vote rollback/apply; mempool re-eligibility recheck; FR58 deep-zone reconstruction) is defined in [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §4.2 (`reconciliation.rs` module) and the supporting [ADR-011](./blockchain-adrs/ADR-011-chain-switch-reconciliation-is-a-structured-workflow.md).

## Scheduling Implications of snake_chain

Part IV is especially important for scheduling behavior, and Part V adds a size-awareness dimension.

### Replay work is protocol work, not maintenance trivia

When balances, configuration, or live UTXOs are about to leave the active chain, replay must happen in time. This means replay generation belongs in the same seriousness category as block production and approval handling.

### Block priorities matter

The implementation needs a policy for competing block intents, at minimum among:

- ordinary transaction blocks,
- balance replay blocks,
- chain-config replay blocks,
- UTXO carry-forward transactions,
- and approval evidence blocks.

### Size-aware packing matters

Because blocks are capped and complex transactions may be large, the scheduler must consider not only semantic priority but also packing feasibility. Under the current PRD, balance replay at a single tail step is guaranteed to fit in one balance block because only one block's worth of seed sources becomes newly droppable at once; size pressure remains relevant for other replay content, especially UTXO carry-forward.

### Registration-aware balance scheduling matters

The implementation should not blindly emit a balance block after every registration. The article’s strategy is to delay until either:

- enough registrations exist for efficient packing,
- or the registration transaction is close enough to tail loss that the new node’s balance state must be preserved.

## Important Non-Implementation Gaps

The articles intentionally leave several things unspecified. These should be treated as open items, not filled with local assumptions.

### 1. Exact communication model

The articles defer exact request/response formats, retry behavior, support-message transport, anti-duplication rules, and fragment protocol details.

### 2. Multi-signature or evidence efficiency

Partly resolved. Per [ADR-015](./blockchain-adrs/ADR-015-approval-subgroup-selection.md), `MAX_AGGREGATED_SIGNATURES = 50` is the default. Per [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §6.5, the `ApprovalAccumulator` allocates a fixed `MAX_BLOCK_SIZE` buffer (~2 KB) that is crypto-agnostic; BLS aggregation packs roughly 5-10× more supporters into the same buffer than Schnorr (per-supporter cost ~4 B with BLS vs ~36 B with Schnorr). What remains open is whether future crypto backends introduce new aggregation primitives that change this tradeoff.

### 3. Random subgroup selection

Resolved by [ADR-015](./blockchain-adrs/ADR-015-approval-subgroup-selection.md). The subgroup is derived by a deterministic rendezvous-hash ordering seeded by the snake-chain tail hash, the proposer node identity, and the proposed sequence.

### 4. Active-node-window semantics

Resolved by [ADR-015](./blockchain-adrs/ADR-015-approval-subgroup-selection.md). A node is active if it appears in the current active `snake_chain` as the creator of at least one block or transaction.

### 5. Mutable configuration support

Partly addressed. [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §11 outlines a `ChainConfigTrait` (tentative/durable transitions per FR8, lock/discard per FR17, replay handling per FR49) that the chain-lib consumes via a trait handle, with state living in a future separate `moonblokz-configuration` crate. The mini-VM capability suggested by FR56 (programmable governance) is explicitly deferred to that future crate's own BMAD; the current architecture treats `ChainConfigTrait` as opaque to the chain-lib core. What remains open is the precise content shape, the approval mechanics for configuration changes mid-chain, and the mini-VM execution model.

### 6. Exact long-disconnect recovery strategy

Part IV clearly states that once active-chain overlap is gone, resynchronization fails. It does not yet define any alternate recovery path such as trusted snapshots, external rebootstrap, or assisted reseeding.

### 7. Chain-config payload envelope vs. open parameter catalog

The outer chain-config payload envelope is now fixed: it carries canonical configuration-content bytes plus a content-signature by node `#0`'s registering key, and replay chain-config blocks reproduce both byte-for-byte. What remains open is the inner parameter catalog and any future dynamic-formula mechanism discussed by later articles.

## Practical Engineering Cautions

### Do not hide pruning behind generic storage GC

`snake_chain` pruning changes blockchain semantics. It is not merely a space-saving garbage-collection concern.

### Keep current-state derivation explicit

Because history disappears, the code should make it clear where current balances, accumulated vote, chain config, and live UTXOs come from.

### Preserve canonical bytes or canonical reconstruction

If the implementation cannot guarantee exact binary fidelity, hash verification, signature verification, packet reassembly, and fee calculations may drift.

### Keep replay generation deterministic and bounded

The target environment is constrained. Replay logic should avoid unbounded scans or unbounded per-block work in hot paths.

### Expect complex approval interactions

Because essential replay blocks can be inserted before evidence blocks while the same proposer-controlled deviation-bearing branch remains in progress, approval workflows should be designed as state machines rather than linear happy-path flows. Those inserted replay blocks do not by themselves start a fresh approval cycle for the already-pending deviation; once the required support package is complete, the proposer can emit the evidence block immediately.

### Plan for observability

This part of the design will be hard to debug without good telemetry around:

- tail movement,
- replay triggers,
- carried-forward UTXOs,
- approval delays,
- fragment reassembly failures,
- and resynchronization failures.

## Business Analyst View: Why This Matters for Delivery

From a delivery perspective, this document helps prevent several common failures:

- implementing Part III consensus as if unlimited history were available,
- implementing pruning as if it had no consensus consequences,
- or treating binary layout and packetization as late-stage integration details.

The safe interpretation is:

- some behaviors are stable enough to structure code around,
- some parameters are intentionally configurable,
- and some critical mechanics are still pending future design work.

## Architect View: Suggested Boundary Discipline

Architecturally, the combined logic should eventually be implemented with a clean separation between:

- persistent block-tree state,
- active current-state projections,
- communication-triggered recovery,
- creator-selection and approval state,
- retention scheduling and replay generation,
- canonical serialization and hashing,
- packet fragmentation and reassembly,
- and policy/configuration values.

This separation should make later evolution easier when data structures, communication details, and cryptographic evidence become more concrete.

## Related Documents

- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) — the conceptual role and design philosophy of the blockchain subsystem.
- [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) — the formal algorithm-level behavior this code realizes.
