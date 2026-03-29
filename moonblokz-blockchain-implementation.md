# MoonBlokz Blockchain Implementation Notes

## Purpose of This Document

This document captures implementation-facing implications of the MoonBlokz blockchain model introduced in Part III and extended in Part IV and Part V of the MoonBlokz series. It is not a full implementation specification. Instead, it complements the conceptual and algorithm documents by identifying what an implementation will need to track, where configuration boundaries exist, how compact binary structures affect engineering choices, and which details must remain open until later articles or repository decisions define them.

- Use [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) for the conceptual description.
- Use [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) for the algorithm model and the detailed main data structures.

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

For implementation planning, this means:

- block-tree, branch selection, and `snake_chain` retention logic belong in the portable blockchain core,
- communication-specific delivery and missing-block requests must stay aligned with the communication boundary,
- storage behavior must reflect the persistence abstraction from Part II,
- serialization rules must remain deterministic across platforms,
- processing-phase crash behavior must be explicit rather than accidental,
- active-chain-switch recomputation must be treated as first-class implementation work,
- pruning cost and retained-history tradeoffs must influence storage design,
- formula-execution boundaries must remain tightly controlled if chain configuration later becomes more expressive,
- implementation choices must preserve best-effort, bounded, embedded-friendly behavior.

Part V makes this stronger still: compact binary structure is not merely a wire concern. It affects hashing, signing, fee calculation, fragmentation, cache design, and flash-write strategy.

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

The todo material highlights an implementation behavior that deserves explicit documentation: if the **processing** phase is interrupted, for example by restart, the current practical recovery policy may be to discard in-progress reconstructed state and start over. This uses the same lifecycle terminology as the conceptual and algorithm documents.

This matters because processing is not merely incremental block intake. It may be a reconstruction pass over a candidate chain. If that pass is interrupted, the implementation must decide whether partially rebuilt state is resumable or disposable.

For now, the conservative interpretation is:

- processing-state restart behavior should be treated as an explicit policy,
- a full restart of reconstruction is an acceptable current simplification,
- resumable processing should be treated as future enhancement rather than assumed capability.

## Implementation-Relevant State

Even without a final storage schema, Parts III, IV, and V imply that an implementation will need access to several categories of state.

### 1. Block storage state

The implementation must be able to retain:

- known blocks,
- parent relationships,
- child relationships or equivalent branch-navigation support,
- disconnected blocks whose parents are not yet available,
- block types,
- serialized block bytes or a lossless reconstruction path,
- and enough metadata to compare branches later.

The need for serialized-byte fidelity is stronger after Part V because signatures and hashes depend on exact byte layout.

### 2. Active-chain window state

Part IV adds a new first-class responsibility: the implementation must explicitly model the active-chain window.

It will likely need to track:

- configured active chain length,
- current head and tail sequence positions,
- which blocks are about to leave the living chain,
- whether required replay actions are pending before the tail advances,
- and whether replay data has already been materialized into new serialized blocks.

### 3. Current node-balance state

Because old transaction history may disappear, the implementation must be able to reconstruct and preserve the current node state independently of full history.

This implies explicit support for:

- current balance per node,
- current vote score per node,
- node identifier to public-key mapping,
- registration-derived node count progression,
- and determining which balances must be present in active balance blocks.

### 4. Live UTXO state

The address side of the model requires tracking:

- currently unspent UTXOs,
- which UTXOs originated in blocks that are nearing tail deletion,
- whether those UTXOs have already been scheduled for carry-forward,
- the custodian-fee-adjusted value that would be re-added,
- and enough reference information to rebuild zero-input replay transactions.

Part V makes one modeling decision explicit: UTXO references use transaction hashes, not sequence numbers. Any storage schema must support efficient lookup by transaction hash plus output index.

### 5. Recovery state

To support missing-parent recovery, the implementation will likely need to track:

- whether a block is fully connected to known ancestry,
- which parent references are unresolved,
- which fragment sets are still incomplete before a block can even be parsed,
- and whether recovery requests have already been made or still need scheduling.

### 6. Vote and participation state

The algorithm implies local tracking of:

- valid message counts per sender node,
- votes or vote scores per candidate creator,
- vote score resets after successful creation or penalty,
- vote-interest growth over time,
- spent-vote values recorded into block headers,
- and first-ranked creator information used by approval deviation logic.

### 7. Approval-support state

Parts III, IV, and V imply that the system must also track, at least conceptually:

- proposed fallback blocks,
- signed support messages for those blocks,
- support sufficiency against the configured majority requirement,
- evidence blocks created from that support,
- timing state related to grace periods,
- and whether `snake_chain` replay blocks have to be inserted before approval evidence is written.

The exact encoding and storage shape remain deferred.

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

The combined articles explicitly or implicitly suggest that several values should remain configurable at chain level or later design level.

### Formula execution boundary

The todo material raises an important future-facing implementation question: if chain configuration eventually contains formulas, should those formulas be evaluated by a constrained expression evaluator or by a more general virtual machine?

At the current knowledge-base level, the safe guidance is not to choose prematurely, but to preserve the boundary conditions any such mechanism would have to satisfy:

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
- grace-period duration,
- approval timing windows,
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

### Anti-capture parameter

The vote-interest mechanism uses a small parameter `y`. This should remain configurable or centrally defined.

## Genesis and Bootstrap Implications

Part IV defines a special two-block bootstrap, and Part V reinforces that those blocks use the same byte-structured blockchain model with some special validation exceptions.

Implementation planning therefore needs a dedicated bootstrap path for:

- creating block `#0` with registration and initial self-transfer,
- allowing its special validation behavior,
- creating block `#1` with the chain configuration,
- serializing both in canonical form,
- and then switching into normal operational rules.

This should be treated as explicit startup logic rather than a quirky exception hidden in general validation code.

## Active-Chain Switch Consequences

The todo material makes clear that an active-chain switch is not just a matter of choosing a new tip hash.

A practical implementation may need to recompute or refresh at least:

- creator-score state,
- node-derived active balances or equivalent active-chain projections,
- branch bookkeeping structures,
- mempool assumptions that depended on the previous active chain.

Reorg handling should therefore be implemented as a structured recomputation workflow rather than a small side effect attached to tip replacement.

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

Because blocks are capped and complex transactions may be large, the scheduler must consider not only semantic priority but also packing feasibility. A replay obligation that does not fit may need to be split across successive blocks where the protocol allows it.

### Registration-aware balance scheduling matters

The implementation should not blindly emit a balance block after every registration. The article’s strategy is to delay until either:

- enough registrations exist for efficient packing,
- or the registration transaction is close enough to tail loss that the new node’s balance state must be preserved.

## Important Non-Implementation Gaps

The articles intentionally leave several things unspecified. These should be treated as open items, not filled with local assumptions.

### 1. Exact communication model

The articles defer exact request/response formats, retry behavior, support-message transport, anti-duplication rules, and fragment protocol details.

### 2. Multi-signature or evidence efficiency

The articles point toward special multi-signature constructions. Current planning should not assume naive signature bundling is final.

### 3. Random subgroup selection

The approval model references majority approval by a randomly selected subgroup but postpones the details.

### 4. Active-node-window semantics

The phrase “previous `n` blocks” is not yet fully formalized.

### 5. Mutable configuration support

Part IV mentions future support for configuration changes with approval and compatibility checks, but the current model does not define that mechanism.

### 6. Exact long-disconnect recovery strategy

Part IV clearly states that once active-chain overlap is gone, resynchronization fails. It does not yet define any alternate recovery path such as trusted snapshots, external rebootstrap, or assisted reseeding.

### 7. Exact chain-config payload schema

Part V confirms the existence of configuration payloads but defers the detailed parameter schema and dynamic-formula mechanism to a later article.

## Practical Engineering Cautions

### Do not hide pruning behind generic storage GC

`snake_chain` pruning changes blockchain semantics. It is not merely a space-saving garbage-collection concern.

### Keep current-state derivation explicit

Because history disappears, the code should make it clear where current balances, vote scores, chain config, and live UTXOs come from.

### Preserve canonical bytes or canonical reconstruction

If the implementation cannot guarantee exact binary fidelity, hash verification, signature verification, packet reassembly, and fee calculations may drift.

### Keep replay generation deterministic and bounded

The target environment is constrained. Replay logic should avoid unbounded scans or unbounded per-block work in hot paths.

### Expect complex approval interactions

Because essential replay blocks can be inserted before evidence blocks, approval workflows should be designed as state machines rather than linear happy-path flows.

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

## Technical Writer View: How to Use This File

Read this file when you want to answer questions such as:

- What must an implementation remember after Parts IV and V?
- How do binary layout and packetization change blockchain engineering responsibilities?
- Which values should remain configurable?
- Which areas are too underspecified to hard-code yet?
- Where are the hardest approval, retention, and transport interactions likely to appear?

## Related Documents

- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md)
- [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md)

## Review Notes

Post-change review against `moonblokz-info` rules:

- **Consistency:** This file now reflects Part V’s binary-structure, memory, flash, and packetization implications while staying aligned with the Part III consensus model and Part IV bounded-storage `snake_chain` rules, while also documenting processing restart policy, pruning cost, retained-history tradeoffs, active-chain-switch recomputation, and formula-execution boundaries suggested by the todo material.
- **Logical soundness:** The text distinguishes clearly between protocol obligations, engineering consequences, and still-deferred transport or payload specifics, while keeping evaluator-versus-VM choices and restart semantics explicitly open where the source material is not yet final.
- **Feasibility:** The document now gives implementation planning enough structure to proceed on serialization, storage, caching, fragmentation boundaries, recomputation responsibilities, and pruning tradeoffs without pretending that deferred details are already settled.