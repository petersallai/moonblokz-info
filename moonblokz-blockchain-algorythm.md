# MoonBlokz Blockchain Algorithm Model

## Purpose of This Document

This document provides a more formal, algorithm-oriented description of the MoonBlokz blockchain behavior developed across Part III, Part IV, and Part V of the MoonBlokz series. Its purpose is to capture the algorithmic flow described by the articles without turning deferred details into invented implementation rules.

This file is also the **primary knowledge-base document for the main blockchain data structures**. Detailed structure descriptions belong here because in MoonBlokz the shape of blocks, transactions, balances, and approval data directly affects algorithmic behavior.

- Use [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) for the higher-level conceptual description.
- Use [`moonblokz-blockchain-implementation.md`](./moonblokz-blockchain-implementation.md) for implementation-facing implications and cautions.

## Source Basis

This document is based on:

- **MoonBlokz series part III. — Basic Algorithms** by Peter Sallai, published on Medium on March 4, 2025.
- **MoonBlokz series part IV. — snake_chain** by Peter Sallai, published on Medium on March 13, 2025.
- **MoonBlokz series part V. — Data Structures** by Peter Sallai, published on Medium on March 26, 2025.

## Scope and Limits

This file captures the algorithmic structure explicitly described or directly implied by the articles, including:

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

The combined blockchain behavior is best understood as a staged runtime lifecycle rather than one uniform mode. The terminology below intentionally matches the conceptual document: **collection**, **processing**, and **ready** are the preferred phase names across the blockchain knowledge-base set.

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
   Sequence number of the block, starting from zero. Blocks `0` and `1` are genesis blocks. The article explicitly notes that overflow is not a practical concern because the active chain only needs a bounded sequence space and sequence numbering can restart if needed after very long time horizons.

3. **`creator: u32`**  
   Node identifier of the block creator.

4. **`mined_amount: u32`**  
   Reward given to miners, excluding transaction fees. Part V explicitly stores this even though it is derivable from full history, because `snake_chain` may remove the historical blocks that would otherwise be needed for recalculation.

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

### Vote field semantics

Each transaction can vote for a node and thereby increase its chance of future block creation.

Restrictions explicitly stated by the article:

- a node cannot vote for itself,
- for node transfer and registration transactions, the initializer cannot receive the vote,
- and for a complex transaction that includes a balance input, the initializer also cannot be the voted node.

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
   Identifier of the newly registered node. Part V says it is generated when the transaction is added to a block as `max node_id + 1`. The article also notes that this value is derivable from the active chain and is stored mainly so that key-to-node binding appears explicitly in one block.

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

1. **`tr_hash: [u8;32]`**  
   Hash of the complex transaction containing the UTXO being spent.

2. **`output_index: u8`**  
   Index of the referenced UTXO output within that transaction.

3. **`signature: [u8;64]`**  
   Signature over this input and all outputs, using the private key corresponding to the referenced UTXO address.

#### Why hashes instead of sequence numbers?

Part V explicitly states that hashes are used instead of sequence numbers, unlike Bitcoin-style references, because hash references are more stable when transactions must be re-added after branch changes. The cost is extra space.

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

### Algorithmic meaning

Complex transactions are the bridge between the balance world and the UTXO world. They also support the `snake_chain` maintenance operation of re-materializing live UTXOs without normal spend inputs.

## 7. Balance Payload Structure

Balance blocks are much simpler than transaction blocks.

### Header fields

1. **`nodeinfo_count: u16`**  
   Number of balance entries in the block.

2. **`max_node_id: u32`**  
   Highest node identifier known in the block’s balance coverage.

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

Part V highlights that `max_node_id` helps nodes verify completeness, especially when a newly joined node is collecting enough balance blocks to reconstruct an active chain. Because node identifiers are sequential, it effectively also represents node count coverage.

### Relationship to `snake_chain`

Balance payloads are the compact state checkpoints that let node-balance history survive pruning. Their structure is intentionally small so many node states can be packed efficiently into a block.

## 8. Chain Configuration Payload

Part V keeps this payload abstract in this article. It states only that these blocks contain configuration parameters and that a separate article will discuss available configuration points and dynamic formulas.

Algorithmically, we can safely conclude that the payload must be treated as:

- first-class blockchain state,
- replayable under `snake_chain`,
- and version-sensitive because configuration fields may evolve later.

## 9. Approval Payload

Part V also keeps approval payload details intentionally deferred. It states that approval is represented by a multi-signature plus a list of supporting nodes.

Algorithmically, this means the payload must be capable of proving:

- which block was approved,
- which nodes supported it,
- and that the evidence satisfies the configured majority requirement.

But the exact encoding remains unresolved in the article series at this stage.

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

1. validate its binary structure and signature,
2. inspect `payload_type` and parse the correct payload schema,
3. insert it into local storage if not already known,
4. link it to its parent if the parent is present,
5. otherwise keep it as a disconnected or partially connected block,
6. preserve all branches because the future winning branch is not yet known.

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
2. Traverse forward through the chosen chain to rebuild operational state.
3. Collect node information needed for later signature and balance validation.
4. Validate blocks only when the required prerequisite state is already available.
5. Rebuild balances, votes, and other active-chain state progressively.
6. If a contradiction, invalid signature, or irrecoverable missing prerequisite appears, reject the reconstructed result and restart collection or recovery as needed.

### Important consequence

Startup validation may therefore be a dedicated reconstruction pass rather than merely the same logic used for ordinary steady-state block extension.

## Algorithm 3: Creator Selection by Message-Based Voting

### Objective

Select the next block creator using only lightweight, locally available information.

### Selection process

1. Observe valid messages and count them per sender.
2. When a node makes a transaction, it votes for the node that has sent it the most valid messages.
3. Aggregate vote scores per candidate node.
4. When a block should be created, select the node with the highest vote score.
5. Break ties deterministically using node identifiers.
6. After successful block creation by the selected node, reset that node’s vote count to zero.

## Algorithm 4: Block Creation Readiness

A new block is created when either:

- enough transactions exist to fill it,
- or enough time has passed.

Parts IV and V add two further constraints:

- maintenance work may have to consume part of the available block capacity before ordinary traffic,
- and `MAX_BLOCK_SIZE` limits what can be packed into a single block.

## Algorithm 5: Grace-Period Controlled Deviation

### Objective

Allow the network to recover when the selected creator does not produce a block.

### Grace-period expansion

1. During the first grace period, only the highest-voted node’s block is accepted.
2. If that period expires without a block, permit the first-ranked and second-ranked nodes.
3. After double the grace period, permit the top three.
4. Continue widening the acceptable creator set as waiting continues.

## Algorithm 6: Approval Process

### Objective

Legitimize deviation from the default creator-selection result.

### Approval flow

1. An eligible node proposes a block after the applicable grace-period condition.
2. Other nodes send digitally signed support messages for that proposed block.
3. When the proposer collects the predetermined amount of support, it creates another block containing approval evidence.
4. The branch containing that evidence becomes the legitimacy-bearing branch for that deviation.

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

- a normal block’s value equals the spent vote score associated with that block,
- an approval block’s value equals the penalized top node’s vote score plus the vote scores of the supporters recorded in the evidence block,
- and the necessary vote-consumption information is preserved explicitly in the block header because full history may later disappear.

## Algorithm 9: Anti-Capture Vote Interest

### Threat addressed

A small group may try to dominate block creation by generating many transactions and repeatedly accumulating creator rights.

### Interest rule

If a node has vote score `x`, then each new block adds an extra `x * y` votes to it, where `y` is a small parameter.

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
3. create a balance block at the tail containing those current values,
4. ensure the living chain still contains balances for all nodes.

## Algorithm 12: Transaction and UTXO Preservation on Tail Drop

### Objective

Allow old transaction blocks to disappear without losing live address-state.

### Rule when a transaction block is dropped

1. inspect the dropped block’s transactions,
2. discard balance outputs that no longer need preservation,
3. discard UTXOs that have already been spent,
4. identify UTXOs that are still unspent,
5. create a special **complex transaction with zero inputs** and outputs corresponding to those surviving UTXOs,
6. reduce each re-added UTXO by the custodian fee,
7. discard any UTXO whose amount becomes smaller than the custodian fee.

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

A complex transaction is valid only if:

- every referenced UTXO input exists and is unspent,
- every balance input is valid against current balance state,
- every input signature is valid for its scope,
- total inputs are greater than or equal to total outputs,
- and no resulting node balance becomes negative.

## Algorithm 14: Node Registration

### Objective

Permit network growth without making Sybil-style expansion free.

### Default registration flow

1. an existing node creates a registration transaction,
2. the initiator pays the configured registration amount and transaction fee,
3. the transaction registers a new public key as a node,
4. the registration fee is absorbed by the network rather than paid to a specific node.

### Configurable variants stated by the article

- free joining if registration price is `0` and self-signed registration is enabled,
- permissioned joining if only node `#0` may register new nodes.

## Algorithm 15: Balance-Block Scheduling for New Nodes

A new balance block does **not** have to be created immediately after each registration.

Instead, create a balance block when either:

- enough new registrations exist to fill the block efficiently,
- or the tail is about to consume the block containing the registration transaction.

## Algorithm 16: Balance-Block Distribution Optimization

### Objective

Keep the active chain efficient by spreading balance blocks across it rather than clustering them.

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
- and it bypasses normal fee calculations.

## Algorithm 18: Approval and snake_chain Precedence

### Conflict addressed

Two rules may compete:

- dropped chain-config or balance data must be replayed at the end of the chain,
- and approval deviations require an evidence block to be added.

### Precedence rule

The stronger rule is the `snake_chain` repeat-block rule.

### Consequences

1. essential state-preservation blocks may be inserted before the approval evidence block,
2. multiple blocks may appear between the approved block and its evidence block,
3. those inserted blocks may themselves trigger additional approval requirements,
4. when the original approval completes, new approval procedures may need to start immediately for subsequent blocks.

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

### Important status of this algorithm

This remains partially unresolved. In particular, the todo material itself leaves open whether the model should track branch points explicitly or whether tip-oriented tracking is sufficient.

For knowledge-base purposes, the safe algorithmic statement is therefore limited to this: MoonBlokz likely needs a compact branch-segment bookkeeping structure during collection and pruning, but the final exact shape of that structure is not yet stable.

## Algorithm 20: Storage-Pressure Branch Eviction

### Objective

When bounded block storage or bounded chain-part storage is exhausted, the node must free space without losing the most operationally valuable branch information.

### Eviction rule

1. Examine the known retained branch ends or equivalent chain-part endings.
2. Select the branch whose terminal sequence is smallest or otherwise oldest according to the chosen bounded-storage rule.
3. Delete blocks backward from that branch tip until reaching a shared ancestor or divergence point that must remain.
4. Update branch-end and chain-part bookkeeping after each deletion or deletion batch.
5. If chain-part metadata itself reaches its bound, trigger the same eviction logic even if raw block storage is not yet full.

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

## Failure and Limit Cases

### Majority-loss stall condition

As already described in Part III, approval may fail if too many active nodes disappear too quickly.

### UTXO saturation stall

If re-added live UTXOs occupy all available block capacity, the chain may temporarily stall because no additional transactions can fit.

### Long-disconnection permanent fork

Resynchronization works only while disconnected nodes still share a common block with the active chain.

If the active chain window moves entirely beyond the last shared block, the disconnected node cannot rejoin and a permanent fork results.

### Verification-horizon tradeoff under pruning

The todo material highlights an important bounded-storage tradeoff.

If a branch remains only partially covered by still-retained history, the node may be able to classify it as stored or connected without being able to fully verify every rule immediately.

That means MoonBlokz may face a tradeoff between:

- retaining more history so later active-chain switches can be validated cheaply,
- or retaining less history and accepting that some active-chain switches may require broader recomputation or full revalidation.

This tradeoff is algorithmically important because it affects how aggressively the implementation can prune while still keeping future branch changes manageable.

### Deferred formal details

Parts III, IV, and V still leave several algorithmically relevant details for later work:

- exact communication protocol,
- exact approval messaging and evidence encoding,
- multi-signature efficiency,
- subgroup selection,
- active-node window formalization,
- exact chain-configuration schema,
- and exact handling of mutable future configuration.

These should be treated as unresolved dependencies, not silently filled in.

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

## Technical Writer View: How to Read This File

This document should be read as the answer to these questions:

- What is the full algorithmic flow described in Parts III, IV, and V?
- What are the main blockchain data structures?
- How do those structures support branch selection, replay, and uniqueness?
- How does `snake_chain` alter ordinary blockchain behavior?
- How are registration, genesis, serialization, and transport boundaries handled?
- What are the main efficiency and failure limits?

## Related Documents

- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md)
- [`moonblokz-blockchain-implementation.md`](./moonblokz-blockchain-implementation.md)

## Review Notes

Post-change review against `moonblokz-info` rules:

- **Consistency:** This file now reflects Part V data structures while preserving the Part III consensus model and Part IV `snake_chain` rules, while also documenting the implied lifecycle, startup reconstruction, pruning, status, and reorg behaviors highlighted by the todo material.
- **Logical soundness:** Detailed field descriptions are limited to what the article states explicitly; newer startup, pruning, and chain-part notes are framed as implied or still-partially-open behavior rather than falsely finalized protocol law.
- **Feasibility:** The document now gives a structurally grounded algorithm view that is suitable for planning real data models, validation logic, lifecycle handling, and transport boundaries without inventing missing protocol details.