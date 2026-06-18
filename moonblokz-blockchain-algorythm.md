# MoonBlokz Blockchain Algorithm Model

## Purpose of This Document

This document provides a more formal, algorithm-oriented description of the MoonBlokz blockchain behavior developed across Part III, Part IV, and Part V of the MoonBlokz series. Its purpose is to capture the algorithmic flow described by the articles without turning deferred details into invented implementation rules.

This file is also the **primary knowledge-base document for the main blockchain data structures**. Detailed structure descriptions belong here because in MoonBlokz the shape of blocks, transactions, balances, and approval data directly affects algorithmic behavior.

- Use [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) as the **authoritative source** for every FR-numbered functional requirement and Non-Functional Requirement cited in this document; FR references in the algorithm prose resolve to the canonical wording in the PRD.
- Use [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) for the higher-level conceptual description.
- Use [`moonblokz-blockchain-implementation.md`](./moonblokz-blockchain-implementation.md) for implementation-facing implications and cautions.

## Source Basis

This document is based on:

- **MoonBlokz series part III. — Basic Algorithms** by Peter Sallai, published on Medium on March 4, 2025.
- **MoonBlokz series part IV. — snake_chain** by Peter Sallai, published on Medium on March 13, 2025.
- **MoonBlokz series part V. — Data Structures** by Peter Sallai, published on Medium on March 26, 2025.

## Scope and Limits

This file captures the algorithmic structure explicitly described or directly implied by the articles, together with current MoonBlokz planning decisions that refine the blockchain module boundary, including:

- block-tree formation,
- missing-parent recovery,
- creator selection by message-based voting,
- approval fallback,
- branch-value comparison,
- bounded active-chain retention through `snake_chain`,
- essential-state replay at the chain tail,
- UTXO carry-forward with custodian-fee reduction,
- node registration and bootstrap rules,
- lifecycle transitions between collection, reconstruction, and ready operation,
- startup dominant-chain acquisition,
- storage-pressure branch pruning,
- block status progression,
- active-chain switching and recomputation obligations,
- block, transaction, balance, and payload structures,
- binary serialization assumptions,
- and the main failure conditions described in Parts III, IV, and V.

It does **not** define implementation details that the articles defer, such as exact radio-layer packet formats, exact multi-signature construction, exact subgroup sampling mechanics, exact flash layout, or detailed mempool scheduling policy.

## Algorithmic Problem Statement

MoonBlokz must operate in a network where:

- nodes communicate over unreliable radio,
- different nodes may receive different subsets of blocks,
- bandwidth and processing power are constrained,
- nodes cannot store unbounded blockchain history,
- radio packets are much smaller than typical blocks,
- and the system cannot rely on heavyweight consensus or global clock synchronization.

The algorithm therefore has to solve five related problems:

1. nodes must recover enough of the same block-tree to reason about shared state,
2. nodes must choose one effective blockchain branch from that tree,
3. the active chain must always retain enough state to continue validating and operating,
4. tail deletion must not destroy balances, configuration, or live UTXOs,
5. and all core blockchain data must be representable compactly enough for radio transport and constrained storage.

## Runtime Lifecycle States

The combined blockchain behavior is best understood as a staged runtime lifecycle rather than one uniform mode. The canonical conceptual definition of these phases belongs in [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md); this algorithm document uses the same terminology and focuses on the operational consequences of those phases.

Across the blockchain knowledge-base set, **collection**, **processing**, and **ready** are the preferred phase names.

## Blockchain Boundary at Algorithm Level

At current MoonBlokz planning level, the blockchain module is treated as a **semantic event state machine**.

That means the algorithm should be read as operating on semantic units such as:

- candidate block received,
- support-related artifact received,
- transaction received,
- local command to create genesis or a transaction,
- local query for active-chain-facing state,
- and locally timed retry or block-creation triggers.

Communication transport, fragment handling, serialization/deserialization mechanics, and cryptographic backend implementation remain outside the algorithmic blockchain boundary, even when the blockchain rules still depend on their outputs.

## Authoritative and Derived Algorithmic State

The canonical statement of the authoritative-versus-derived distinction belongs in [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) and is formalized normatively in [ADR-003](./blockchain-adrs/ADR-003-authoritative-vs-derived-state-in-moonblokz-blockchain.md).

Within this algorithm document, the important consequence is that algorithmic workflows operate over a deliberately small authoritative base while treating active-chain selection, balance / UTXO truth, and vote / next-creator state as maintained operational views that may need recomputation or reconciliation.

### Collection state

In this state the node does not yet have an active valid chain. Its main objective is to discover and extend a dominant candidate chain quickly enough to reach an operational state.

Typical responsibilities include:

- receiving parseable blocks and storing them,
- tracking unresolved ancestry,
- requesting missing parents,
- maintaining branch-end or chain-part bookkeeping,
- identifying when a candidate chain is long enough to justify transition into reconstruction.

### Processing state

In this state the node has identified a candidate chain with sufficient length, but it still needs to reconstruct usable state from it.

Typical responsibilities include:

- marking which stored blocks belong to the selected chain,
- traversing that chain in both directions as needed for reconstruction,
- rebuilding node, balance, and vote state,
- rejecting the candidate chain if contradictions appear during reconstruction.

### Ready state

In this state the node has an active chain and maintains it continuously.

Typical responsibilities include:

- validating new candidate blocks,
- tracking side branches,
- maintaining block status information,
- recalculating active state on active-chain switches,
- pruning excess retained history within configured limits.

## Block Acceptance Pipeline

Across all lifecycle states, the todo material implies a practical block-acceptance pipeline.

1. A block arrives from the network.
2. If the block cannot be parsed according to its structural rules, discard it.
3. If the block is parseable, store it as known block data.
4. Determine whether its parent is already known locally.
5. If the parent is missing, schedule or trigger parent recovery.
6. If the parent is known, connect the block into the known block-tree or chain-part structure.
7. Apply state-specific follow-up logic:
   - in collection state, use it to extend candidate-chain knowledge,
   - in processing state, preserve it but avoid claiming final validity too early,
   - in ready state, treat it as an active-chain extension candidate or a side-branch candidate.

This pipeline is the operational counterpart of the conceptual document’s staged-validation idea: parseability, storage worthiness, ancestry connectivity, and full semantic validity do not have to become known at the same moment.

## Core Data-Structure Design Principles

Part V makes several design principles explicit.

### Compactness is part of correctness

Block and transaction structures must be small enough to:

- fit within bounded block sizes,
- be fragmented for radio transport,
- remain cacheable in small RAM,
- and avoid unnecessary flash writes.

### Fixed-size where possible, variable-size where necessary

MoonBlokz prefers fixed-size fields and compact integer types, but uses variable-length lists when the protocol needs structural flexibility, especially in transaction blocks.

### Structure is tied to economics

Transaction fees are size-sensitive. This means data-structure choices are not only technical choices. They also shape the economic incentives of the chain.

### No timestamp dependency

The data model avoids global timestamps. Sequence-based anchoring and structural uniqueness replace conventional time-based ordering and nonce strategies.

## Main Blockchain Data Structures

## 1. Block Structure

The fundamental data structure is the block. Every block has two parts:

- a **fixed-size header**,
- and a **variable-size payload**.

### Block header fields

The header has ten fields.

1. **`version: u8`**  
   Protocol version number. Part V states the current fixed value is `1`, but the field exists for forward compatibility.

2. **`sequence: u32`**  
   Sequence number of the block, starting from zero. The original genesis blocks `0` and `1` are identified by their fixed FR54 content, not by sequence value alone. In MVP, the active chain operates strictly within the unsigned 32-bit sequence space: the highest sequence value that may legitimately appear on the active chain is `u32::MAX − 1`, and every block whose declared `sequence == u32::MAX` is rejected at intake as exact evidence of invalidity. Sequence comparisons (`snake_chain`-window membership, `anchor_sequence` bounds, active-chain head/tail relationships, out-of-window detection) therefore use simple unsigned-integer `u32` ordering. Sequence wrap-around handling is a **post-MVP feature** — see "Sequence Wrap-Around" under Failure and Limit Cases below for the post-MVP framing. Source: the MVP no-wrap rule and the post-MVP deferral are normatively pinned by `_bmad-output/planning-artifacts/prd.md` FR53; the original Part III–V articles only note that "sequence numbering can restart if needed after very long time horizons" without specifying the mechanism.

3. **`creator: u32`**  
   Node identifier of the block creator.

4. **`mined_amount: u32`**  
   Reward given to miners, excluding transaction fees. Part V explicitly stores this even though it is derivable from full history, because `snake_chain` may remove the historical blocks that would otherwise be needed for recalculation. The MoonBlokz blockchain module additionally credits the block creator with a chain-config-derived `replay_block_reward` whenever the accepted block carries a `snake_chain` essential-state replay obligation (chain-config replay, balance replay, or zero-input UTXO carry-forward); this reward is paid in addition to `mined_amount` and is not stored in the block header — it is computed from chain-config inputs at acceptance time. The `replay_block_reward` compensates the creator for fee-neutral maintenance content. Source: this addition is normatively defined by `_bmad-output/planning-artifacts/prd.md` FR36 (c) / FR59 and is not in the original Part III–V articles.

5. **`payload_type: u8`**  
   Current values are:
   - `1` = transaction block
   - `2` = balance block
   - `3` = chain configuration block
   - `4` = evidence block

6. **`consumed_votes: u32`**  
   Number of votes consumed by the creator to create the block.

7. **`first_voted_node: u32`**  
   Node with the most votes when the block was created. In the normal case this equals the creator. If not, approval logic is involved.

8. **`consumed_votes_from_first_voted_node: u32`**  
   Usually `0`. It is non-zero when the creator is not the first-voted node and votes of the first-ranked node are part of deviation accounting.

9. **`previous_hash: [u8;32]`**  
   Hash of the previous block, covering both header and payload.

10. **`signature: [u8;64]`**  
    Block creator signature for the entire block.

### Algorithmic meaning of the block header

The Part V header is not merely descriptive metadata. Several fields directly support earlier algorithms:

- `sequence` supports ordering, active-window tracking, and anchor checks.
- `mined_amount` and vote-consumption fields preserve value information that could otherwise be lost when historical blocks are pruned.
- `first_voted_node` and `consumed_votes_from_first_voted_node` make approval deviations auditable.
- `previous_hash` and `signature` preserve immutable branch linkage.

### Block size bound

The total block size is bounded by chain configuration (`MAX_BLOCK_SIZE`). This cap is algorithmically important because it constrains:

- how many transactions fit in a block,
- how many balances can be replayed at once,
- how many UTXOs can be carried forward in a maintenance transaction,
- and how efficiently `snake_chain` preservation can be scheduled.

## 2. Payload Type Model

The payload structure depends on `payload_type`.

### Payload type `1`: transaction block

Contains a transaction list.

### Payload type `2`: balance block

Contains compact node-state entries.

### Payload type `3`: chain configuration block

Contains configuration parameters for the chain.

### Payload type `4`: approval evidence block

Contains proof that a non-default block was supported by the required majority.

## 3. Transaction Block Structure

A transaction block begins with:

- **`transaction_count: u16`**
- followed by a list of transactions.

The theoretical limit is `65535` transactions, but in practice `MAX_BLOCK_SIZE` is the real limiting factor.

Each transaction starts with a fixed mini-header:

- **`type: u8`**
- **`vote: u32`**

### Transaction types

Part V currently defines three transaction types:

- `1` = node transfer
- `2` = registration
- `3` = complex transaction

### Per-block registration / complex-transaction mutual-exclusivity

A transaction block (`payload_type=1`) shall not simultaneously contain at least one registration transaction (`type=2`) and at least one complex transaction (`type=3`, including the zero-input UTXO carry-forward complex transaction used for `snake_chain` maintenance per Algorithm 12). A block whose payload mixes the two kinds is structurally invalid.

The reason is per-payload-type single-replay scheduling. Registrations contained in a transaction block can become the only in-window seed source for the registered nodes' derived state, so when such a block drops out of the active window it triggers balance replay (Algorithm 11). Complex-transaction UTXO outputs contained in a transaction block can remain unspent at drop time, so when such a block drops it triggers UTXO carry-forward (Algorithm 12). Allowing both kinds in the same dropping block would force a single tail advance to trigger balance replay and UTXO carry-forward simultaneously, breaking the per-payload-type single-replay invariant the head-emission rules of Algorithm 4, Algorithm 11, and Algorithm 12 rely on. Confining each transaction block to one of the two kinds keeps the multi-payload-type-per-tail-drop case structurally impossible in steady-state operation, so a single dropping tail block always triggers at most one payload-type replay obligation.

Node-transfer transactions (`type=1`) are unrestricted by this rule: a transaction block may contain node transfers alongside either registrations or complex transactions, but not registrations and complex transactions together.

### Vote field semantics

Each transaction can vote for a node and thereby increase its chance of future block creation.

At current module-boundary level, the choice of **which node gets the vote when a new transaction is accepted** is not made by the blockchain module from raw radio observations. Instead, an external **scoring module** may compute per-node scores from the senders of observed radio messages and provide the selected vote target to the caller that assembles the transaction. The blockchain module's local transaction-creation surface then receives a fully assembled signed transaction with that chosen vote target already recorded in its `vote` field, validates it, and admits it to the mempool when accepted. The blockchain **vote module** later tracks each node's accumulated vote count — both votes already present in transactions and blocks and the anti-capture interest growth applied on each block acceptance — and determines the next block creator from that accumulated state.

Restriction explicitly stated by the article — a single rule (the no-self-vote rule) applied at two structural sites:

- the rule: a transaction's `vote` field shall not equal any initializer present in the transaction's structure;
- application site 1 — node transfer and registration transactions: the transaction-level `initializer` field cannot equal `vote`;
- application site 2 — complex transactions that include one or more balance inputs: every balance input's `initializer` field cannot equal `vote` (a violation at any one balance input is sufficient to invalidate the transaction);
- a complex transaction with only UTXO inputs and no balance input has no transaction-level or balance-input-level initializer; the rule has no structural application to such a transaction.

Genesis exception: the no-self-vote rule and the related vote-target validity rule (every `vote` value must reference an existing node on the active chain) do not apply to transactions in genesis block #0. At genesis time, node #0 is the only node that exists, so any vote target other than node #0 would itself violate vote-target validity, and a vote for node #0 is the only consistent choice. Accordingly, the transactions inside block #0 carry `vote == 0` (node #0 voting for itself), and this is treated as a valid genesis vote rather than as a self-vote violation.

Permanent node-#0 chain-wide exceptions: from block #2 onward both rules resume, but two narrow exceptions remain in force for the lifetime of the chain — `vote == 0` is always a valid vote target, and `initializer == 0` with `vote == 0` (a node-#0 self-vote) never violates the no-self-vote rule. These chain-wide exceptions exist because between block #2 and the registration of the first non-genesis node, node #0 may still be the only existing node; without these exceptions, a node-#0-issued registration transaction would have no valid `vote` target and would deadlock further chain progress. From the point at which any non-genesis node has been registered, every other initializer continues to be subject to the unchanged no-self-vote and vote-target validity rules. Source: this clarification is normatively pinned by `_bmad-output/planning-artifacts/prd.md` FR37 / FR57 (i) / FR6; the original Part V genesis exception was confined to block #0, but algorithmic completeness requires the two narrow chain-wide exceptions above so the early-chain bootstrap deadlock is structurally avoided.

This means the transaction format itself participates in consensus state updates.

## 4. Node Transfer Transaction Structure

A node transfer is the minimal balance-to-balance transaction between nodes. Part V states that it has **fixed size: 96 bytes**.

### Fields

1. **`anchor_sequence: u32`**  
   Sequence number of the blockchain when the transaction was created. It acts as a local ordering anchor and replaces traditional account nonces. The transaction cannot be included in any block with sequence less than or equal to this value.

2. **`initializer: u32`**  
   Node that starts the transaction and pays the fee.

3. **`receiver: u32`**  
   Receiving node identifier.

4. **`amount: u64`**  
   Amount transferred.

5. **`fee: u32`**  
   Fee paid by the initializer. Part V states fee policy is configurable and may use per-byte pricing with minimum and maximum limits.

6. **`comment: u64`**  
   Free-form caller-controlled field used for reference numbers, counters, or other distinguishing data.

7. **`signature: [u8;64]`**  
   Initializer signature over the whole transaction content.

### Algorithmic role of `anchor_sequence`

`anchor_sequence` does several jobs at once:

- it provides a sequence-based time surrogate without requiring clocks,
- it prevents inclusion in blocks that are too old,
- it narrows the duplicate-check search window,
- and together with `comment` it supports uniqueness for repeated balance-style transfers.

### Transaction uniqueness rule for node transfers

The network rejects fully identical transactions. If an initializer wants to send multiple otherwise-identical transactions while the anchor is unchanged, the `comment` field must differ.

This is an important algorithmic alternative to classical nonce-based account transactions.

## 5. Registration Transaction Structure

Registration adds a new node identifier and public key to the network.

### Fields

1. **`initializer: u32`**  
   Existing node that initiates the registration and pays the costs.

2. **`new_node_id: u32`**  
   Identifier of the newly registered node. Part V says it is generated when the transaction is added to a block as `max node_id + 1`. The article also notes that this value is derivable from the active chain and is stored mainly so that key-to-node binding appears explicitly in one block. Genesis exception: the registration in block `#0` carries `new_node_id == 0`, replacing the `max + 1` rule for that single registration; the post-genesis `max_known_node_id` watermark equals `0`, and the `max + 1` rule resumes from block `#2` onward (so the first post-genesis registration carries `new_node_id == 1`). See Algorithm 17 for the broader genesis bootstrap handling.

3. **`registration_price: u64`**  
   Configurable registration cost. Stored even though it is derivable from chain rules, because it simplifies later balance calculations when earlier history may have been pruned.

4. **`fee: u64`**  
   Regular transaction fee.

5. **`new_public_key: [u8;32]`**  
   Public key of the new node. It must be globally unique.

6. **`new_key_signature: [u8;64]`**  
   Proof that the registering party possesses the corresponding private key. Part V describes this as a Schnorr-compatible signature of the public key itself.

7. **`signature: [u8;64]`**  
   Signature over the full transaction content by the initializer.

### Algorithmic meaning

Registration is not only an identity action. It changes:

- node roster size,
- future balance-block requirements,
- consensus population,
- and economic state because registration price is absorbed by the network.

## 6. Complex Transaction Structure

Complex transactions support mixed balance and UTXO semantics. Unlike node transfers, they are variable-size structures.

### Top-level shape

A complex transaction starts with:

- **`input_count: u8`**
- **`output_count: u8`**

Then come the inputs and outputs. Each entry starts with its own **`type: u8`** tag.

The article states:

- there may be zero or more inputs,
- there must be one or more outputs,
- maximum counts are nominally `255` each,
- but `MAX_BLOCK_SIZE` is the practical limit.

A transaction with **zero inputs** is a special `snake_chain` maintenance case used to re-add surviving UTXO outputs.

### 6.1 UTXO input (`input type = 0`)

Fields:

1. **`block_sequence: u32`**  
   Sequence number of the block containing the complex transaction whose UTXO is being spent. The reference is resolved against the current active chain only; within the `snake_chain` window the sequence uniquely identifies a stored block.

2. **`output_index: u8`**  
   Index of the referenced UTXO output within the block at that sequence. The index addresses the UTXO output position in the block's flattened transaction-output stream, consistent with the block's per-block spent-bit vector layout.

3. **`signature: [u8;64]`**  
   Signature over this input and all outputs, using the private key corresponding to the referenced UTXO address.

#### Why sequence numbers instead of transaction hashes?

The earlier Part V rationale preferred transaction-hash references on the grounds that they survive branch changes more gracefully. Under `snake_chain` bounded retention this argument does not yield a net benefit: carry-forward across the window boundary invalidates references in both models, and the sequence-indexed block storage used on the embedded target already provides O(1) lookup. Sequence references are therefore shorter, avoid the need for a hash-keyed auxiliary index, and pair naturally with a per-block spent-bit vector for unspent-state tracking. The decision and its consequences are recorded in [ADR-016](./blockchain-adrs/ADR-016-sequence-indexed-utxo-input-model.md).

### 6.2 Balance input (`input type = 1`)

Part V says the fields are the same as in a node transfer, except that the signature covers the balance input and all outputs.

That implies:

- `anchor_sequence: u32`
- `initializer: u32`
- `amount: u64`
- `comment: u64`
- `signature: [u8;64]`

Notably, the article does **not** list a standalone `fee` field for the balance input, because fee emerges from input-output difference at the complex-transaction level.

### 6.3 UTXO output (`output type = 0`)

Fields:

1. **`address: [u8;32]`**  
   Public key of the output owner.

2. **`amount: u64`**  
   Amount assigned to the UTXO.

### 6.4 Balance output (`output type = 1`)

Fields:

1. **`receiver: u32`**  
   Receiving node identifier.

2. **`amount: u64`**  
   Amount sent.

### Complex transaction fee rule

Part V states that a complex transaction is valid only if:

- total input amount is greater than or equal to total output amount,
- and the difference is the transaction fee.

This implies no separate fee field is needed at top level for complex transactions.

The `total input ≥ total output` rule above describes the case Part V addresses directly — complex transactions with at least one input. The zero-input `snake_chain` maintenance case noted in this section follows a separate validity regime defined by [Algorithm 12](#algorithm-12-transaction-and-utxo-preservation-on-tail-drop) and detailed in Algorithm 13's Complex transaction validity subsection: a zero-input complex transaction is permitted only as a carry-forward, the input-vs-output ordering rule does not apply to it, and its inherent fee is zero (the difference cannot be negative because Part V's fee derivation is replaced by the carry-forward's custodian-fee reduction model).

### Algorithmic meaning

Complex transactions are the bridge between the balance world and the UTXO world. They also support the `snake_chain` maintenance operation of re-materializing live UTXOs without normal spend inputs.

## 7. Balance Payload Structure

Balance blocks are much simpler than transaction blocks.

### Header fields

1. **`nodeinfo_count: u16`**  
   Number of balance entries in the block.

2. **`max_node_id: u32`**  
   The chain-global node-id watermark at this balance block's sequence — i.e., the highest node identifier registered anywhere on the active chain up to and including this block, not a per-block coverage subset count. The blockchain module's `max_known_node_id` derived projection is initialized from the earliest balance block in a candidate segment and is incremented by exactly one on every subsequent registration; any later balance block whose `max_node_id` field disagrees with the forward-traversal-tracked watermark at that block's sequence is exact evidence of invalidity. Source: the chain-global watermark interpretation is normatively pinned by `_bmad-output/planning-artifacts/prd.md` FR52 / FR57 (h); the original Part V wording ("highest node identifier known in the block's balance coverage") was ambiguous about whether the field was coverage-scoped or chain-global, and the PRD resolves it as chain-global.

### NodeInfo entry structure

Each node entry contains:

1. **`owner: u32`**  
   Node identifier.

2. **`balance: u64`**  
   Current money balance for that node.

3. **`vote_count: u32`**  
   Current total votes for the node.

4. **`public_key: [u8;32]`**  
   Public key used to validate future signatures.

### Algorithmic meaning of `max_node_id`

Part V highlights that `max_node_id` helps nodes verify completeness, especially when a newly joined node is collecting enough balance blocks to reconstruct an active chain. Because node identifiers are sequential, the chain-global watermark interpretation also represents node count coverage. The watermark is incremented exactly once per accepted registration transaction (transaction-by-transaction, including multiple registrations within a single block, which must form a strictly increasing `new_node_id` sequence with stride 1 starting from the pre-block watermark).

### Relationship to `snake_chain`

Balance payloads are the compact state checkpoints that let node-balance history survive pruning. Their structure is intentionally small so many node states can be packed efficiently into a block.

## 8. Chain Configuration Payload

Part V keeps the inner parameter catalog abstract, but the current MoonBlokz design now fixes the outer payload contract more precisely.

Algorithmically, a chain-configuration payload is treated as:

- first-class blockchain state,
- replayable under `snake_chain`,
- version-sensitive because configuration fields may evolve later,
- and bound to a stable content identity by a dedicated signature from node `#0`'s registering key.

The payload therefore contains:

1. the configuration-content bytes,
2. and a content-signature by node `#0` over the canonical bytes of that configuration content.

This content-signature is distinct from the ordinary block-creator signature in the block header:

- the block header proves who created the specific chain-config block instance,
- the content-signature proves that the configuration content itself is the chain's durable configuration content,
- and every replayed chain-config block reproduces the same configuration-content bytes and the same node-`#0` content-signature byte-for-byte.

The exact inner configuration parameter catalog and any future formula language remain separate design concerns, but the envelope rule above is now part of the algorithmic model.

## 9. Approval Payload

Part V also keeps approval payload details intentionally deferred. It states that approval is represented by a multi-signature plus a list of supporting nodes.

Algorithmically, this means the payload must be capable of proving:

- which block was approved,
- which nodes supported it,
- and that the evidence satisfies the configured majority requirement.

Subgroup selection and the supporting-node set are now defined in [ADR-015](./blockchain-adrs/ADR-015-approval-subgroup-selection.md). The evidence therefore only needs to carry actually signing supporters, so the relevant crypto bound is `required_support ≤ MAX_AGGREGATED_SIGNATURES(backend)` rather than a bound on the full subgroup size.

The approval evidence block payload begins with a `supporter_vote_sum: u64` field, computed by the proposer at evidence-block creation time per the closed-form formula `supporter_vote_sum = (m / 2 + 1) · deviance_creator_vote + original_creator_vote` (integer division for `m / 2`), where `m` is the subgroup size from ADR-015 (`m = min(2 · required_support − 1, |A|)`), `deviance_creator_vote` is the deviation block creator's `vote_count` at sequence `proposed_sequence − 1` (projected against the candidate chain that the deviation block extends), and `original_creator_vote` is the `vote_count` of the node that would have been the creator at `proposed_sequence` in the absence of the deviation — the top entry of the creator-order projection at sequence `proposed_sequence − 1` whose deviation triggered this approval — projected against the same candidate chain at the same sequence. The field is followed by the aggregated support signature and the list of supporting node identifiers. Other nodes verify the proposer's claim authoritatively at chain-switch reconciliation or processing-pass time by recomputing the formula against the candidate chain's per-node vote-score projection at sequence `proposed_sequence − 1` and comparing to the recorded `supporter_vote_sum` value; a discrepancy invalidates the evidence block. Because the formula depends only on `m`, `deviance_creator_vote`, and `original_creator_vote` — all fixed by the candidate chain at `proposed_sequence − 1` — the proposer cannot vary the resulting value by choosing which subgroup-eligible signers it admits into the aggregated evidence beyond satisfying `required_support`; this also guarantees that the deviation-bearing branch's combined contribution at the deviation point strictly exceeds the corresponding normal-branch per-block contribution at the same sequence, so a later normal-branch supplement cannot retroactively overtake the deviation-bearing branch under the branch-value rule. The payload field order — `supporter_vote_sum` first, then aggregated support signature, then supporter identity list — is established here; the exact binary encoding of each component (aggregated-signature byte length per backend, identity-list count format) follows the broader serialization rules of Section 10 and any backend-specific specifications.

## 10. Serialization Rules

Part V explicitly states the blockchain uses **binary serialization**.

### Serialization assumptions

- fields are stored sequentially,
- multi-byte fields use **little-endian** encoding,
- and blocks are both stored and transferred in this binary form.

### Algorithmic significance

This matters because binary layout is not just an implementation detail in a constrained protocol:

- signature scopes depend on deterministic byte layout,
- block hashes depend on exact serialized form,
- fee policy may depend on transaction size in bytes,
- and radio fragmentation works on serialized blocks.

## 11. Radio Packetization Consequences

Part V explains that many blocks or transactions are larger than a physical radio packet, especially on LoRa-like networks where packet size is around 256 bytes.

Algorithmically, this implies:

- a block is the logical blockchain unit,
- but not the physical radio unit,
- so the radio layer must fragment and reassemble serialized block content,
- and any large transaction type must remain valid even when transmitted as multiple radio messages.

This packetization logic is outside the blockchain data model itself, but it is a direct consequence of the data-structure sizes defined here.

## Algorithm 1: Block-Tree Construction

### Objective

Maintain a tree of candidate blockchain histories instead of assuming that one chain is always globally visible.

### Core rule

When a node receives a block:

1. validate its binary structure,
2. inspect `payload_type` and parse the correct payload schema,
3. if the block is a ready-state direct extension of the current active head and the creator key is already derivable from that same active chain, treat creator-signature invalidity as immediate exact evidence; otherwise defer authoritative signature invalidity decisions to the later candidate-side validation paths,
4. insert it into local storage if not already known,
5. link it to its parent if the parent is present,
6. otherwise keep it as a disconnected or partially connected block,
7. preserve all branches because the future winning branch is not yet known.

## Algorithm 2: Missing-Parent Detection and Recovery

### Trigger condition

A node receives a block whose parent block is not present locally.

### Recovery rule

1. infer that earlier history is missing,
2. use the child block’s `previous_hash` and `sequence`,
3. ask the network for the missing ancestor using the known sequence and hash,
4. repeat this recovery process as additional missing ancestors are discovered.

### Best-effort limit

If a leaf disappears without descendants and no later block reveals its absence, that leaf may remain permanently unknown. Recovery therefore aims for convergence on active and relevant branches, not guaranteed reconstruction of every historical leaf.

## Algorithm 2A: Initial Dominant-Chain Acquisition

### Objective

Before a node can operate normally, it must identify a sufficiently complete candidate chain that most likely represents the active network.

### Acquisition process

1. Track all known branch ends or equivalent chain-part endings.
2. Prefer extending the longest or otherwise most promising candidate chain.
3. If a newly arrived block has an unknown parent, request that specific parent block.
4. If the parent is already known, request earlier chain history of that candidate branch only when request-throttling rules permit it.
5. Continue until one of the practical stopping conditions is reached:
   - ancestry reaches block `#0`,
   - or the chain becomes long enough to satisfy the configured active-chain-length target.

### Interpretation

This is a startup convergence algorithm, not yet final semantic validation. Its purpose is to reach a reconstructable candidate chain quickly.

## Algorithm 2B: Full-Chain Reconstruction After Acquisition

### Objective

Once a sufficiently long candidate chain has been found, the node must reconstruct a usable active state from it before entering normal operation.

### Reconstruction process

1. Traverse backward from the candidate chain tip to mark which stored blocks belong to the chosen chain.
2. Before forward replay begins, preload the candidate chain-configuration content from the marked chain and reject the candidate if the in-scope chain-config blocks disagree or if no required chain-config block is present.
3. Traverse forward through the chosen chain to rebuild operational state.
4. Collect node information needed for later signature and balance validation, including per-node seed sources inside the retained segment. Each node is seeded from its earliest in-segment balance entry, or — if no such balance entry exists — from its in-segment registration.
5. Rebuild balances, votes, spent-bit vectors, and other active-chain state progressively.
6. When a transaction involves a node before that node's earliest in-segment seed source, apply the pre-seed rule: node-state-dependent checks for that node are temporarily auto-accepted inside the reconstruction pass, while structural, payload-level, chain-config-derived, and UTXO-reference checks that do not depend on that node's pre-state still execute normally.
7. UTXO input existence and per-block spent-bit tracking continue across the whole pass, including pre-seed zones, because they are resolved from the referenced outputs and the retained segment itself.
8. Validate every other invariant as soon as its prerequisite state is available.
9. If a contradiction, invalid signature, or irrecoverable missing prerequisite appears, apply the FR5 atomic recovery defined in the implementation notes: discard the forward-traversal working set in full and atomically, and either (a) when the offending block can be precisely identified, delete the offending block from durable storage together with every block whose only retained ancestry path runs through it — this explicitly includes the candidate-chain blocks between the offender and the pre-failure tip — or (b) when the offending block cannot be precisely identified, delete exactly one block from durable storage, the candidate chain's head. Then return the chain to collecting state. There is no configurable fallback depth; the single-block drop is the fixed fallback.

### Important consequence

Startup validation may therefore be a dedicated reconstruction pass rather than merely the same logic used for ordinary steady-state block extension.

## Algorithm 3: Creator Selection by Voting Over Scoring Module Input

### Objective

Select the next block creator using lightweight locally available information while keeping the scoring module outside the blockchain module boundary.

### Selection process

1. Receive vote-target selection input from the scoring module, derived from radio-layer observation of valid messages.
2. When a node accepts a new transaction, the scoring module determines which other node currently has the strongest local message-based score and therefore receives the vote recorded into the transaction.
3. Aggregate vote scores per candidate node inside blockchain state from the votes already present in accepted transactions and blocks, with each accepted transaction contributing one configured `vote_scale` credit to its target node.
4. When a block should be created, select the node with the highest accumulated blockchain vote score.
5. Break ties deterministically using node identifiers.
6. After successful block creation by the selected node, reset that node’s vote count to zero.

### Boundary note

The original article-era framing described vote preference in terms of directly observed valid messages. The current MoonBlokz module boundary refines that design so the blockchain vote module consumes scoring module input at transaction-creation time rather than owning the score computation itself. The scoring module produces the vote target; the vote module tracks the accumulated votes and determines the next block creator.

## Algorithm 4: Block Creation Readiness

A new transactional block is created when either:

- the currently eligible ordinary mempool transactions can fill at least a configured target percentage of `MAX_BLOCK_SIZE`,
- or the configured inter-block time has elapsed since the previous block and at least one ordinary transaction can be included.

Tail-drop preservation work is stricter: if a chain-config replay, balance replay, or UTXO carry-forward block is required to prevent loss of state, that replay block shall be emitted immediately after the previous block, without waiting for the transactional fill threshold or the ordinary inter-block timer; UTXO carry-forward in particular always proceeds without waiting whenever a transaction block is about to drop.

Replay-block content rule: each replay block emitted at the head reproduces every essential-state structure contained in the corresponding dropping tail block, subject to `MAX_BLOCK_SIZE`. The replay block's payload type matches the dropping block's content category — a dropping balance block is reproduced by a balance replay block carrying NodeInfo entries, a dropping chain-config block by a chain-config replay block carrying the durable-locked configuration, and a dropping transaction block by a transaction block carrying zero-input complex transactions for unspent UTXO carry-forward (which may also carry ordinary mempool transactions up to `MAX_BLOCK_SIZE`).

Single-payload-type-per-dropping-block invariant: under the registration / complex-transaction mutual-exclusivity rule of Section 3, a dropping transaction block (`payload_type=1`) contains either registrations or complex transactions, but never both, and therefore triggers at most one of the two transaction-driven replay obligations — balance replay (Algorithm 11, when a registration in the dropping block is the only in-window seed source for the newly registered node) or UTXO carry-forward (Algorithm 12, when the dropping block contains complex-transaction UTXO outputs still unspent at drop time). A dropping balance block (`payload_type=2`) triggers balance replay only; a dropping chain-config block (`payload_type=3`) triggers chain-config replay only; a dropping approval-evidence block (`payload_type=4`) triggers no replay obligation. A single tail advance therefore requires at most one replay block at the head, so the active chain length never exceeds `W` in steady-state operation. The replay-bearing creator receives one `replay_block_reward` for the replay block they produce, and creator selection for that replay block follows the ordinary creator-order projection.

Approval-workflow exception: under the approval workflow (Algorithm 6), the deviation block's creator (the proposer) is the only authorized creator of every block on the deviation-bearing branch above the deviation block. Consequently every intervening `snake_chain` essential-state replay block required between the deviation block and the associated approval evidence block is also created by the proposer. The replay-block content rule above still applies — each replay block reproduces the essential state of the dropping block matching its payload type — and the proposer accumulates one `replay_block_reward` for each such intervening replay block they create, in addition to ordinary `mined_amount` per accepted block.

Parts IV and V add two further constraints:

- maintenance work may have to consume part of the available block capacity before ordinary traffic,
- and `MAX_BLOCK_SIZE` limits what can be packed into a single block.

Source: the per-replay-type content rules originate in Parts III–V; the always-without-wait carry-forward rule, the approval-workflow attribution, and the single-payload-type-per-dropping-block invariant follow from the registration / complex-transaction mutual-exclusivity rule of Section 3 and from the per-payload-type single-replay scheduling that Algorithm 11 and Algorithm 12 rely on.

### Mempool replenishment

To stay prepared for becoming the block creator at any later moment, the blockchain module shall periodically emit an outbound mempool-replenishment request to the network whenever its mempool does not hold enough transactions to reach the transactional fill threshold. This behavior is not gated on the local node being the currently expected creator: any node whose mempool is below that threshold solicits additional content continuously, so that if the creator role shifts to it (through normal vote progression or through grace-period expansion), it already has the transactions needed to produce a sufficiently filled block.

The request carries the CRC32 values of the transactions the module already holds in the mempool — the same CRC32 values used as the canonical mempool identifier in the compacted mempool index. Responding nodes treat any of their own mempool transactions whose CRC32 matches an entry in the request as already-known to the requester and skip them; the responder selects at most one transaction the requester is still missing, prioritizing higher fee-per-byte and own-transaction precedence. CRC32 collisions across different transaction byte sequences are accepted as benign over-exclusion at this surface (the over-excluded transaction is simply skipped on this round and may be supplied on a later replenishment exchange); no full byte comparison happens during replenishment evaluation, because CRC32 is the canonical mempool identifier in MoonBlokz.

- The request period is a configurable timing parameter owned by the blockchain module.
- Requests are emitted only while the mempool content is below the transactional fill threshold; once the threshold is reached, no replenishment traffic is generated until the mempool drops below the threshold again (for example, after a block is accepted and its transactions are removed).
- Replenishment is an ordinary first-class behavior of the blockchain module, not a radio-layer or operator-driven concern: the decision of when to ask, how often, and when to stop belongs to the blockchain module.
- The radio layer is responsible only for the transport of the request and for the delivery of responding transactions back into the module through the normal transaction-intake surface.

This closes the gap between passive mempool acceptance (radio-delivered transactions arriving on their own) and the readiness obligation that applies to every node: when the network's natural traffic is insufficient, the module actively solicits the missing content rather than waiting indefinitely and then being caught unprepared when the creator role arrives.

## Algorithm 5: Grace-Period Controlled Deviation

### Objective

Allow the network to recover when the selected creator does not produce a block.

### Grace-period expansion

1. During the first grace period, only the highest-voted node’s block is accepted.
2. If that period expires without a block, permit the first-ranked and second-ranked nodes.
3. After double the grace period, permit the top three.
4. Continue widening the acceptable creator set as waiting continues.

### Vote-score reset is applied only to the first node

As a data-size optimization, **only the originally-top node's (the first node's) accumulated vote score is reset to zero** at the start of the fallback cycle. All other nodes that are admitted in later expansion steps but still miss the chance to produce the block — the second-ranked, third-ranked, and so on — **retain their vote scores unchanged**. There is exactly one reset per fallback cycle, regardless of how many grace-period expansions occur before some admitted node finally produces a block.

This is sufficient for anti-capture dynamics because the originally-top node carries the full penalty; it is also sufficient for audit purposes because the subsequently admitted nodes are the ones that are given the chance to create, not penalized for inactivity in that cycle.

## Algorithm 6: Approval Process

### Objective

Legitimize deviation from the default creator-selection result and make the deviation-bearing branch competitively defensible against later competing normal branches.

### Approval flow

1. An eligible node proposes and creates a deviation block after the applicable grace-period condition; this block may extend the creator's current active chain immediately.
2. Other nodes send digitally signed support messages for that proposed block.
3. As soon as the proposer collects the predetermined support package — the required 50%+1 majority represented by `required_support` for the deterministically selected subgroup — it creates another block containing approval evidence.
4. The branch containing that evidence becomes the legitimacy-bearing and branch-value-competitive branch for that deviation.

### Supporting subgroup

The set of nodes whose support signatures count toward an approval is selected deterministically by the rule defined in [ADR-015](./blockchain-adrs/ADR-015-approval-subgroup-selection.md), which also defines which nodes are considered "active" for this purpose. The approval flow above is independent from the selection mechanics, and the seed and ordering formulas are not duplicated here.

### Fallback when support is insufficient

When an approval cannot collect enough support in time, the subgroup is not widened. The grace-period expansion in Algorithm 5 admits the next-ranked creator candidate, which starts a new approval with its own freshly derived subgroup.

### Important notes

- approval is slower and heavier than normal block creation,
- approval is exceptional, not the preferred path,
- and the evidence mechanism is essential because deviation must be provable later from blockchain state.

## Algorithm 7: Penalty Handling

### Trigger condition

The highest-voted node fails to create the expected block within its grace period.

### Penalty rule

Reset the vote score of that first missed node to zero.

## Algorithm 8: Branch Value Calculation

### Objective

Choose the effective blockchain branch from the block-tree using local information.

### Value rules from Parts III and V

- a normal block’s value equals the spent vote score associated with that block (recorded as `consumed_votes` in the block header);
- an approval block’s value equals the value carried in its `supporter_vote_sum: u64` payload field at the start of the evidence block’s payload (per Section 9 of this document); the proposer computes the field by the closed-form formula `supporter_vote_sum = (m / 2 + 1) · deviance_creator_vote + original_creator_vote` (integer division, with `m` the subgroup size, `deviance_creator_vote` the deviation block creator’s vote score, and `original_creator_vote` the would-have-been original creator’s vote score, all projected against the candidate chain at sequence `proposed_sequence − 1`), and the value is validated authoritatively at chain-switch / processing-pass time by recomputing the formula and comparing — a discrepancy invalidates the evidence block; the `consumed_votes_from_first_voted_node` field of the deviation block records the originally-top node’s pre-penalty vote score for audit purposes only and does **not** contribute to the evidence block’s value;
- a `snake_chain` essential-state replay block (chain-config replay per Algorithm 11, balance replay per Algorithm 11, or zero-input UTXO carry-forward per Algorithm 12) intervening between a deviation block and its associated evidence block contributes value zero, because such blocks carry `consumed_votes = 0` by deliberate header construction (per blockchain module PRD FR28);
- and the necessary vote-consumption information for normal blocks (`consumed_votes`) and the auditable deviation-record information for deviation blocks (`first_voted_node`, `consumed_votes_from_first_voted_node`) are preserved explicitly in the block header because full history may later disappear.

### Branch selection and tie-break

The active branch is the operationally admissible candidate branch with the highest cumulative branch value. Operationally admissible means the candidate retains a continuous tip-ending segment long enough to cover the full `snake_chain` window: once the chain has matured to length `W`, this means at least `W` blocks on that candidate branch; before the chain first reaches length `W`, the admissible bootstrap case is a genesis-anchored candidate spanning the entire currently existing chain from block `#0` to its tip. A deviation-bearing branch that does not yet contain its associated approval evidence block does not overtake the corresponding normal branch: because the deviation block is created only after the originally-top creator missed its opportunity, the deviation block's own `consumed_votes` is lower than the ordinary top-creator contribution that would have won at that same sequence, so approval is the mechanism that can later make the deviation branch competitive. When two or more candidate branches are tied on cumulative branch value, the tie shall be resolved deterministically in the following order: first by the **serialized size of the branch tip block in bytes** (larger size wins — under equal vote-derived branch values, the branch carrying more block content is preferred); then, if still tied, by the **node identifier of the branch tip's creator** (lower node identifier wins); and finally, if the tip creator is also tied under creator equivocation at the same sequence with equally-sized blocks, by the **lowest tip block hash** interpreted as a big-endian unsigned integer. This tie-break applies only in ready/processing state, where vote scores are trusted; collecting-state candidate selection follows its own dedicated rule and does not use branch value.

## Algorithm 9: Anti-Capture Vote Interest

### Threat addressed

A small group may try to dominate block creation by generating many transactions and repeatedly accumulating creator rights.

### Interest rule

If a node has vote score `x`, then each accepted block updates that node's score in integer arithmetic as `x := x + floor(x × vote_interest / vote_scale)`, where `vote_scale` is both the numeric value of one received vote credit and the denominator of the anti-capture rule, and `vote_interest` is the per-block growth numerator. `floor(a / b)` here is ordinary unsigned integer division.

## Algorithm 10: snake_chain Tail Advancement

### Objective

Maintain a bounded active chain length while preserving all information required for continued operation.

### Core rule

1. Maintain an active chain window of configured length.
2. When a new block arrives near the head, the oldest block at the tail eventually becomes droppable.
3. Before allowing essential state to disappear, schedule replay work so the required information remains present in the active chain.

## Algorithm 11: Essential Block-Type Preservation

### Chain-config rule

If a chain-config block would drop out of the active chain:

1. create a new chain-config block at the end of the chain,
2. repeat the same configuration content,
3. temporarily prioritize this maintenance action over optional work.

### Balance-block rule

If balances would be lost from the active chain:

1. identify the affected nodes,
2. compute their **current** balances and vote scores,
3. create a balance block at the head containing those current values,
4. ensure the living chain still contains balances for all affected nodes and therefore preserves full rolling-window coverage across the chain,
5. rely on the fact that this replay step fits into one balance block because a single tail advance makes only one block's worth of seed sources newly droppable and the replacement block only has to restate the nodes affected by that frontier.

## Algorithm 12: Transaction and UTXO Preservation on Tail Drop

### Objective

Allow old transaction blocks to disappear without losing live address-state.

### Rule when a transaction block is dropped

1. inspect the dropped block’s transactions,
2. discard balance outputs that no longer need preservation,
3. read the dropping block's per-block spent-bit vector (per [ADR-016](./blockchain-adrs/ADR-016-sequence-indexed-utxo-input-model.md)) to classify each UTXO output as spent or unspent on the current active chain,
4. create a special **complex transaction with zero inputs** whose outputs correspond to the still-unspent UTXOs identified in step 3,
5. reduce each re-added UTXO by the custodian fee,
6. discard any UTXO whose amount becomes smaller than the custodian fee.

Repeated carry-forward is intentionally self-clearing: each preservation step reduces every surviving UTXO by the custodian fee, and once the remaining amount falls below that fee the UTXO disappears instead of being replayed again. The custodian fee is therefore the long-run relief mechanism that eventually clears retained carry-forward-only residue.

### Compression rule

To compress data, re-added UTXOs may be collected from the following `n` blocks, where `n` is configurable, until block-size limit is reached.

If not enough such UTXOs exist, new mempool transactions may also be included in the same block.

## Algorithm 13: Transaction Validity Rules by Structure

Part V makes several structural validation rules explicit.

### Node transfer validity

A node transfer is valid only if:

- the initializer has sufficient balance for amount plus fee,
- the block sequence is strictly greater than `anchor_sequence`,
- the signature is valid,
- and the transaction is not a duplicate of an already accepted identical transaction.

### Registration validity

A registration transaction is valid only if:

- the initializer is an existing node,
- the initializer can pay registration price plus fee,
- the public key is unique,
- the `new_key_signature` proves possession of the corresponding private key,
- and the initializer signature is valid.

### Complex transaction validity

Complex transactions follow two distinct validity regimes depending on whether the transaction has any inputs. The split is dictated by the existence of the zero-input `snake_chain` maintenance case introduced in Section 6 and made operational by [Algorithm 12](#algorithm-12-transaction-and-utxo-preservation-on-tail-drop).

#### Complex transaction with at least one input

A complex transaction with at least one input is valid only if:

- every referenced UTXO input resolves against the current active chain: the block at `block_sequence` exists within the retained window, the `output_index` is within the bounds of that block's UTXO-output stream, and the corresponding bit in that block's spent-bit vector is `0` (unspent) per [ADR-016](./blockchain-adrs/ADR-016-sequence-indexed-utxo-input-model.md),
- every balance input is valid against current balance state,
- every input signature is valid for its scope,
- total inputs are greater than or equal to total outputs,
- and no resulting node balance becomes negative.

#### Zero-input complex transaction

A zero-input complex transaction is permitted only as a `snake_chain` UTXO carry-forward, and its validity is governed exclusively by the carry-forward rules of [Algorithm 12](#algorithm-12-transaction-and-utxo-preservation-on-tail-drop). The rules of the with-input case above — in particular the `total inputs ≥ total outputs` rule — do not apply to it: by construction a carry-forward has zero inputs and one or more positive outputs. A zero-input complex transaction that does not satisfy Algorithm 12's carry-forward correctness rules — including a zero-input transaction that does not correspond to any soon-to-drop transaction block within the configured carry-forward lookahead `n` — is invalid.

## Algorithm 14: Node Registration

### Objective

Permit network growth without making Sybil-style expansion free.

### Default registration flow

1. an existing node creates a registration transaction,
2. the initiator pays the configured registration amount and transaction fee,
3. the transaction registers a new public key as a node,
4. the registration price is absorbed by the network rather than paid to a specific node; the ordinary transaction fee is a distinct amount and is handled separately.

### Configurable variants stated by the article

- free joining if registration price is `0` and self-signed registration is enabled,
- permissioned joining if only node `#0` may register new nodes.

## Algorithm 15: Balance-Block Scheduling for New Nodes

A new balance block does **not** have to be created immediately after each registration.

Instead, create a balance block when either:

- enough new registrations exist to fill the block efficiently,
- or the tail is about to consume the block containing the registration transaction.

## Algorithm 16: Balance-Block Distribution Optimization (Post-MVP)

### Objective

Keep the active chain efficient by spreading balance blocks across it rather than clustering them. This is a post-MVP optimization rather than a correctness-critical MVP rule.

### Conceptual rule

1. estimate the optimal spacing between balance blocks from active-chain length and balance-block count,
2. when replaying a dropped balance block, compare current spacing with desired spacing,
3. if spacing is too large, move the replayed balance block forward by several sequences,
4. if spacing is too small, replay immediately because delaying would risk losing required state,
5. repeat over many cycles to smooth the distribution.

## Algorithm 17: Genesis Initialization

Part IV defines a two-block initialization sequence.

### Genesis rule

1. **Block `#0`** is a transaction block created by node `#0`.
2. It contains:
   - a registration transaction for node `#0`,
   - and a transfer from node `#0` to itself containing the initial total network currency.
3. **Block `#1`** is the chain-config block created by node `#0`.

### Special handling

The first block is processed differently from later blocks:

- it may be signed with the same key used for registration,
- it bypasses normal balance checks,
- it bypasses normal fee calculations,
- the no-self-vote rule and the vote-target validity rule (Section 3 above and Algorithm 13) do not apply to its transactions: at genesis time node `#0` is the only existing node, so its registration transaction and self-transfer carry `vote == 0` (node `#0` votes for itself), and this is the only consistent choice; from block `#2` onward both rules resume in the ordinary way for every other initializer, but two narrow chain-wide exceptions remain in force for the lifetime of the chain — `vote == 0` is always accepted as a valid vote target, and `initializer == 0` with `vote == 0` (a node-`#0` self-vote) never violates the no-self-vote rule, so node `#0` retains a valid vote choice even while it is the only registered node and can therefore issue further registration transactions before any non-genesis node exists,
- the `max_known_node_id` watermark is initialized to `0` directly at genesis (no balance block exists in the genesis segment to seed it from), the `new_node_id == max_known_node_id + 1` rule of registration validity is replaced for the block `#0` registration by the explicit invariant `new_node_id == 0`, and the watermark is not incremented by the block `#0` registration (post-genesis the watermark equals `0`, matching the highest-known-node-id semantics of `max_node_id` in balance-block payloads),
- and block `#1` carries a content-signature by node `#0`'s registering key over the canonical configuration-content bytes, distinct from the ordinary block signature in the header.

## Algorithm 18: Approval and snake_chain Precedence

### Conflict addressed

Two rules may compete:

- dropped chain-config or balance data must be replayed at the end of the chain,
- and approval deviations require an evidence block to be added.

### Precedence rule

The stronger rule is the `snake_chain` repeat-block rule.

### Consequences

1. essential state-preservation blocks may be inserted before the approval evidence block,
2. multiple blocks may appear between the deviation block and its evidence block,
3. those intervening replay blocks remain part of the same proposer-controlled deviation-bearing branch and do not by themselves start a fresh approval cycle for the already-pending deviation,
4. once the required support package is complete, the proposer may emit the approval evidence block immediately; this step is not gated by ordinary transactional block-fill thresholds, ordinary inter-block timer conditions, or the presence of ordinary mempool transactions.

## Algorithm 19: Chain-Part Maintenance

### Objective

The todo material suggests a practical intermediate structure for collection- and pruning-time bookkeeping: a chain-part model that tracks partial chains by their current start and end references together with a stable identifier.

### Intended responsibilities

A chain-part structure may be used to:

- record the beginning and end of a known partial chain,
- estimate candidate-chain length during startup,
- merge two partial chains when new connectivity is discovered,
- support branch-end maintenance under bounded memory,
- and help choose which branch segment to evict under storage pressure.

### Status of this algorithm

The blockchain module PRD now resolves this as follows: branch tracking is **tip-oriented** through the FR19 `chain_heads` table (every block-tree tip indexed with per-head metadata: head_block_id, head_sequence, connected flag, tail-point cache for Stored heads, connection-point cache for Connected/Active heads, and `last_request_timestamp` for Stored heads). Branch points are not separately enumerated; they are implicit in the block-tree graph. Block-level shared-ancestry membership is captured by the per-block `head_ref_count` field (FR18 / FR19), which counts how many chain_heads entries' ancestry passes through that block — this is sufficient for bounded eviction without explicit branch-point tracking. The chain_heads table is bounded by `chain_heads_max_capacity` and supports a deterministic eviction discipline (smallest non-active `head_sequence` wins; head_ref_count protects shared blocks). See FR19 and FR20 for the full normative rules.

## Algorithm 20: Storage-Pressure Branch Eviction

### Objective

When bounded block storage or bounded chain-part storage is exhausted, the node must free space without losing the most operationally valuable branch information.

### Eviction rule

1. Examine the known retained branch ends or equivalent chain-part endings.
2. Select the retained side branch whose tip has the lowest cumulative branch value.
3. Break ties by lowest tip sequence, then by lowest tip creator node identifier, then by the lowest tip block hash interpreted as a big-endian unsigned integer.
4. Delete blocks backward from that branch tip until reaching a shared ancestor or divergence point that must remain.
5. Update branch-end and chain-part bookkeeping after each deletion or deletion batch.
6. If chain-part metadata itself reaches its bound, trigger the same eviction logic even if raw block storage is not yet full.

### Interpretation

This algorithm makes bounded-storage behavior explicit at the branch level. Eviction is not only tail advancement of the active chain. It may also remove side branches that are no longer worth retaining under current limits.

## Algorithm 21: Block Status Progression

### Objective

Represent the fact that a known block can move through several operational states before it is trusted or used by the active chain.

### Practical status model

The todo material suggests at least the following conceptual statuses:

- **stored** — the block is parseable and retained locally,
- **connected** — the block has a known ancestry path into locally retained history,
- **verified** — the block has passed the currently possible semantic checks,
- **invalid** — the block failed required checks,
- **active** — the block is currently part of the selected active chain.

### Important note

The precise transition graph is still open, especially when earlier retained history is missing and full verification must be deferred.

## Algorithm 22: Active-Chain Switch and Recomputation

### Objective

When a side branch overtakes the current active branch according to the local chain-selection rules, the node must recompute all dependent active state.

### Recomputation rule

1. Determine that the competing branch outranks the current active chain according to the configured sequence, creator, score, or equivalent branch-selection rules.
2. Identify the common ancestor or equivalent branch-merge point.
3. Recalculate the active score table.
4. Recalculate node state derived from the active chain.
5. Recalculate branch bookkeeping derived from the active chain.
6. Recalculate mempool acceptance assumptions that depended on the previous chain.
7. Mark the new branch as active and demote the previous one.

### Interpretation

This algorithm clarifies that an active-chain switch is not only a pointer change. It is a state recomputation event.

## Algorithm 23: Effective-Chain Determination Under snake_chain

A node determines the effective blockchain by:

1. storing the known block-tree,
2. recovering missing ancestors when possible,
3. tracking creator-selection and approval outcomes through vote-derived value,
4. preserving chain configuration, balances, and live UTXOs within the active-chain window,
5. comparing candidate branches by accumulated block value plus valid preservation state,
6. following the branch whose structure, value, and retained active state make it the winning effective chain.

## Algorithm 24: Efficiency Estimation

Part IV provides explicit efficiency formulas.

### Balance-only efficiency

Given:

- `chain_length` = active chain length in blocks,
- `node_count` = total number of nodes,
- `balance_per_block` = number of balances per block,

then:

- `number_of_balance_blocks = node_count / balance_per_block`
- `number_of_non_transaction_blocks = node_count / balance_per_block + 1`
- `efficiency = (chain_length - number_of_non_transaction_blocks) / chain_length`
- equivalently `efficiency = 1 - node_count / (balance_per_block * chain_length) - 1 / chain_length`

The article’s examples indicate that balance-related `snake_chain` loss is typically below 5% in normal setups.

### UTXO-related efficiency

For re-added UTXOs, the same general pressure applies, but the cost can become much worse because the number of UTXOs is not naturally capped by node count.

## Algorithm 25: Serialization and Fragmentation Boundary

Part V adds an important boundary rule.

### Logical rule

The blockchain validates and stores **whole serialized blocks**.

### Physical transport rule

The radio layer may split those serialized blocks into smaller packets because physical packets are smaller than blocks.

### Consequence

Any transport design must preserve:

- deterministic reconstruction of the original byte stream,
- stable hash and signature verification after reassembly,
- and bounded handling of partial or missing fragments.

The exact packet format is outside the scope of the article, but the boundary itself is explicit.

## Algorithm 26: Local Active-Chain Query Surface

### Objective

Expose a practical local-facing blockchain truth surface without requiring full internal branch observability.

### Query model

1. Transaction queries should distinguish at least between unknown, present in mempool, and present in the active chain.
2. If a transaction is present in the active chain, the response should also expose its active-chain depth in sequence terms.
3. Block queries should resolve only against the current active chain rather than the full retained block-tree.
4. Balance queries may expose a simple current answer by default, with an optional richer answer that also reports how deeply the visible balance is supported inside the active chain.

### Interpretation

This query surface is intentionally product-facing and active-chain-centered. It does not deny the existence of side branches internally, but it avoids turning the default local interface into a full branch-inspection API.

## Failure and Limit Cases

### Majority-loss stall condition

As already described in Part III, approval may fail if too many active nodes disappear too quickly.

### UTXO saturation stall

If re-added live UTXOs occupy all available block capacity, the chain may temporarily stall because no additional transactions can fit. The blockchain module performs no special saturation detection, status reporting, log emission, mempool admission backpressure, or replay deferral: ordinary block creation continues against whatever pending replay obligations exist, producing a sequence of replay-bearing blocks that admit no ordinary mempool transactions. The condition self-clears across many tail-advance cycles because the fixed custodian fee reduces every surviving carried-forward UTXO at each carry-forward step until UTXOs whose pre-fee amount falls below the custodian fee are discarded under FR53's below-fee rule, eventually freeing block capacity for ordinary mempool transactions again. The mempool continues to accept new transactions normally throughout. Source: this no-special-handling rule is normatively pinned by `_bmad-output/planning-artifacts/prd.md` FR55 / FR59 / FR53.

### Sequence Wrap-Around

**MVP rule.** In MVP, the `sequence: u32` field is a strictly increasing unsigned 32-bit value with a hard ceiling at `u32::MAX − 1`. The blockchain module shall not create or accept any block whose declared `sequence == u32::MAX`: block creation suspends structurally once the active-chain head reaches `sequence == u32::MAX − 1`, and every inbound block with `sequence == u32::MAX` is exact evidence of invalidity at intake regardless of its `previous_hash` linkage. Within the MVP sequence space the chain never wraps, and every sequence comparison (`snake_chain`-window membership, `anchor_sequence` bounds, head/tail ordering, out-of-window detection per Algorithms 2A / 10 / 13, branch-value admissibility per Algorithm 8) is interpreted with simple unsigned-integer ordering of `u32` values. Reserving `u32::MAX` as an always-rejected sentinel avoids any boundary-condition ambiguity at the ceiling — no legitimate `u32::MAX` block exists to which a wrap-around continuation could attempt to link. This is a deliberate MVP scope decision: any realistic embedded-LoRa deployment is expected to remain far below `2^32 ≈ 4.29 billion blocks` over the project's intended lifetime.

**Post-MVP framing (deferred, not implemented in MVP).** A future revision is expected to introduce sequence wrap-around handling so that the chain can grow indefinitely across the `u32` boundary. The current planning direction (not yet binding) anticipates:

- a circular sequence space modulo `2^32`, in which a newly created head block whose predecessor (resolved by `previous_hash`) has `sequence == u32::MAX` carries `sequence == 0` and forward extension continues from there;
- a wrap-block-vs-genesis discrimination rule based on content match — the original genesis blocks remain pinned by their FR54 content (block `#0`: a node `#0` registration with `new_public_key == node_zero_public_key` plus a node `#0` self-transfer carrying the chain's `initial_total_network_currency`; block `#1`: the chain-config block whose content matches the durable-locked configuration), and the bootstrap-only validation exceptions of Algorithm 17 apply only to those original genesis blocks identified by content match, never to wrap-blocks;
- ancestry-based sequence-comparison rules so that `snake_chain`-window membership, `anchor_sequence` bounds, head/tail ordering, and out-of-window detection remain consistent across the wrap point; an implementation may still use the modular distance `(S_head − target) mod 2^32 < W` as an equivalent fast check while the chain is fully in window, falling back to ancestry traversal when an ambiguity arises;
- re-scoping of genesis-anchored bootstrap clauses (Algorithm 2A genesis-anchored stopping condition, Algorithm 8 genesis-anchored admissibility) so they refer exclusively to the original genesis identified by content match — once the chain has wrapped at least once, the original genesis has long since left the active window and only the active-length-satisfying clauses apply.

None of this post-MVP behavior is implemented in MVP. FR53's MVP rejection of `u32::MAX` keeps the wrap surface closed until that work is taken up.

Source: the MVP no-wrap rule and the post-MVP deferral are normatively pinned by `_bmad-output/planning-artifacts/prd.md` FR53 (and the Project Scoping § Post-MVP Extensions section that lists sequence wrap-around as a deferred feature). The original Part III–V articles only note that "sequence numbering can restart if needed after very long time horizons" without specifying the mechanism.

### Long-disconnection permanent fork

Resynchronization works only while disconnected nodes still share a common block with the active chain.

If the active chain window moves entirely beyond the last shared block, the disconnected node cannot rejoin and a permanent fork results.

Long-disconnect detection and permanent-fork entry are **not two separate states**; they are the same event viewed from two angles. Once a node observes that every active-chain block it knows has fallen outside the broader network's active-chain window (for example, incoming blocks consistently reference a `previous_hash` that cannot be resolved locally and carry sequences far beyond the local head plus the `snake_chain` window), the node:

- does not accept those out-of-window blocks into its authoritative chain,
- continues its ordinary operation against its local active chain — including own block creation, mempool handling, and creator-role behavior,
- and emits a structured log record marking the observation.

Because the local node keeps operating and producing blocks against its own active chain while the rest of the network does the same against theirs, the two chains evolve independently from that moment onward. The detection rule describes the observable trigger; the permanent fork describes the structural outcome. No further automatic reconciliation is possible from chain state alone, and the current model does not define a recovery path beyond external operator action.

### Verification-horizon tradeoff under pruning

The todo material highlights an important bounded-storage tradeoff.

If a branch remains only partially covered by still-retained history, the node may be able to classify it as stored or connected without being able to fully verify every rule immediately.

That means MoonBlokz may face a tradeoff between:

- retaining more history so later active-chain switches can be validated cheaply,
- or retaining less history and accepting that some active-chain switches may require broader recomputation or full revalidation.

This tradeoff is algorithmically important because it affects how aggressively the implementation can prune while still keeping future branch changes manageable.

The MoonBlokz blockchain module pins this tradeoff with an explicit verification-horizon parameter `H` of additional retained prior active-chain blocks beyond the active `snake_chain` window of length `W`, with the constraint `0 ≤ H ≤ W` and a default value of `H = ⌊W / 10⌋` when no implementation-specific tuning is applied. `H = W` is a generous choice consistent with the implementation-document recommendation of "roughly an extra chain-length worth of history" for nodes whose storage budget allows it. `H` is implementation-defined per node (not chain-config-derived); replay determinism is unaffected by per-node `H` differences because `H` only changes the local efficiency class of chain-switch reconciliation, not the eventual active-chain selection. Source: the explicit parameter and default are normatively defined by `_bmad-output/planning-artifacts/prd.md` FR61 / FR59 and ADR-012; the original Part III–V articles describe the tradeoff but do not pin a default.

### Deferred formal details

Parts III, IV, and V still leave several algorithmically relevant details for later work:

- exact communication protocol,
- exact approval messaging and evidence encoding,
- multi-signature efficiency,
- exact chain-configuration schema,
- and exact handling of mutable future configuration.

These should be treated as unresolved dependencies, not silently filled in.

The originally listed items "subgroup selection" and "active-node window formalization" are now resolved by [ADR-015](./blockchain-adrs/ADR-015-approval-subgroup-selection.md).

## Business Analyst View: Why This Algorithm Exists

From a business and product perspective, the combined algorithm tries to balance:

- low-resource operability,
- predictable progress,
- protection against double spending,
- bounded device storage,
- radio-compatible payload sizes,
- economic reward for useful participation,
- and survivability under imperfect communication.

The Part V data structures matter because the blockchain is only useful if its rules can still fit the physical and economic constraints of the target platform.

## Architect View: Structural Interpretation

Architecturally, Parts III, IV, and V imply that the full algorithm has at least these layers:

1. **observation layer** — receiving blocks and messages,
2. **reconstruction layer** — building the block-tree and recovering missing ancestors,
3. **selection layer** — choosing a preferred creator by vote scores,
4. **recovery layer** — approval-based fallback when normal selection fails,
5. **retention layer** — preserving balances, configuration, and live UTXOs as the tail advances,
6. **valuation layer** — assigning branch value so later reconciliation remains deterministic,
7. **structure layer** — keeping every logical unit compact, serializable, and fragmentation-tolerant.

## Related Documents

- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) — the conceptual role and design philosophy of the blockchain subsystem.
- [`moonblokz-blockchain-implementation.md`](./moonblokz-blockchain-implementation.md) — code-level crate structure, APIs, and engineering cautions.
