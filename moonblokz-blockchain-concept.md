# MoonBlokz Blockchain Concept Model

## Purpose of This Document

This document explains the conceptual operating model of the MoonBlokz blockchain as developed across Part III, Part IV, and Part V of the MoonBlokz series. Its purpose is to describe what kinds of participants exist in the network, how blockchain state behaves under unreliable radio communication, why bounded storage changes the blockchain model, and why compact data structures are a first-class protocol concern rather than a low-level optimization detail.

This file is intentionally focused on conceptual understanding rather than full algorithm specification or implementation guidance.

- Use this document to understand **what the system is**, **why it behaves this way**, and **why bounded storage and compact encoding shape the blockchain design**.
- Use [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) for the formal algorithm description and the detailed data-structure definitions.
- Use [`moonblokz-blockchain-implementation.md`](./moonblokz-blockchain-implementation.md) for implementation-facing consequences and cautions.

## Source Basis

This document is based on:

- **MoonBlokz series part III. — Basic Algorithms** by Peter Sallai, published on Medium on March 4, 2025.
- **MoonBlokz series part IV. — snake_chain** by Peter Sallai, published on Medium on March 13, 2025.
- **MoonBlokz series part V. — Data Structures** by Peter Sallai, published on Medium on March 26, 2025.

## Relationship to Earlier MoonBlokz Documents

This file should be read after:

- [`moonblokz-overview.md`](./moonblokz-overview.md), which introduces the project vision and operating environment,
- [`moonblokz-technology.md`](./moonblokz-technology.md), which introduces the technology platform and architectural boundaries,
- and before the more detailed algorithm and implementation notes.

The authoritative source for the FR-numbered functional requirements (FR1–FR69) and the Non-Functional Requirements referenced from the algorithm, implementation, and ADR documents is [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md). This concept document explains the operating model behind those requirements; the canonical wording of any FR lives in the PRD.

This document answers the conceptual question: **what kind of blockchain does MoonBlokz become when it must operate over unreliable radio, cannot store unbounded history, and must keep every critical representation compact enough for constrained devices?**

## Business Analyst View: What Problem This Model Solves

Parts III and IV established that MoonBlokz cannot assume global ordering, instant agreement, or permanent retention of all historical blocks. Part V adds a third operational reality: even the representation of blockchain state must be designed for extremely limited radio packets, limited RAM, and flash storage with write-cycle constraints.

Together, these parts define the real problem MoonBlokz is solving:

- the network must tolerate temporary divergence,
- nodes must still recover enough shared state to continue operating,
- the system must preserve essential financial information even while deleting old blocks,
- and the structures used for that state must stay compact enough for microcontroller-class hardware.

This makes MoonBlokz conceptually different from a classical ever-growing blockchain. It is a **best-effort, eventually consistent, storage-bounded, size-conscious blockchain** designed for constrained local infrastructure.

## Network Entities

Part III separates the network into two entity types:

- **nodes**, which are active participants in the blockchain,
- and **addresses**, which are passive endpoints for holding and transferring value.

This distinction remains essential in Parts IV and V because storage policy, privacy, fees, consensus efficiency, and data-structure complexity all depend on it.

### Nodes

Nodes are active infrastructure participants, usually physical devices communicating through radio.

The article series assigns the following properties to nodes:

- only nodes can create blocks,
- only nodes can collect transaction fees and mining rewards,
- transactions between nodes are cheaper,
- node-to-node transfers are transparent,
- nodes do not pay custodian fees,
- and every node must be known to the network for the consensus model to work efficiently.

Conceptually, nodes are both the infrastructure layer and the governance population of the chain. Part V reinforces this by giving node-oriented transfers the smallest and cheapest transaction shape.

### Addresses

Addresses are passive entities used for holding and transferring value with better privacy properties.

The article series assigns the following trade-offs to addresses:

- users can create many addresses,
- separate addresses improve privacy,
- address-related transactions are more expensive,
- and stored funds are subject to custodian fees.

This means MoonBlokz treats privacy as a resource-consuming feature that must be priced because it increases blockchain storage, processing pressure, and data-structure size.

## Hybrid Value Model: Balances for Nodes, UTXOs for Addresses

Part IV introduced a hybrid storage and value model, and Part V makes the structural consequence explicit.

- When the receiver is a **node**, MoonBlokz uses a **balance-based** representation.
- When the receiver is an **address**, MoonBlokz uses a **UTXO-based** representation.

Conceptually, this split serves two different goals:

- balances keep the known-node infrastructure efficient enough for lightweight consensus,
- UTXOs preserve privacy and ownership isolation for addresses that are not part of the known consensus population.

Part V shows that this is not only a policy split but also a structural split. Node transfers are optimized for minimal size, while complex transactions carry richer variable structures because privacy and multi-party flexibility cost more.

## Why the Chain Becomes a Block-Tree

Because nodes work independently and may miss messages, they can create different successor blocks from different local states.

As a result, the practical data shape becomes a **block-tree** rather than a single line.

This means:

- nodes may share a common past,
- diverge into parallel present states,
- and temporarily disagree on what the latest valid chain actually is.

This is a normal property of MoonBlokz’s operating environment, not an exceptional failure.

## Why Data Size Is a Protocol Concern

Part V adds an important conceptual clarification: in MoonBlokz, data compactness is not merely an implementation preference. It is part of protocol viability.

Three constraints drive this:

- **radio packets are small**, so blockchain objects often cannot be transmitted whole,
- **device RAM is small**, so only limited active state and caches can remain in memory,
- **flash storage is slow and wear-limited**, so persistent writes must be minimized.

As a result, MoonBlokz does not treat “how data is shaped” as separable from “how the chain works.” The structure of blocks, transactions, balances, and replay information directly affects whether the protocol is operable on target devices.

## Blockchain Module as a Semantic State Machine

The current MoonBlokz design direction sharpens the module boundary around blockchain behavior.

Conceptually, `moonblokz-blockchain` is best understood as a **stateful semantic event state machine** rather than as a transport-aware byte-processing layer.

It:

- receives blockchain-relevant semantic events from network, local, and timer directions,
- maintains authoritative blockchain knowledge about known blocks, mempool contents, and current operating mode,
- derives the active chain and current economic state from that authoritative base,
- and answers local blockchain-facing queries needed by higher-level payment or application interfaces.

This framing keeps communication transport, serialization details, storage mechanics, radio-observation logic, and cryptographic implementation outside the conceptual blockchain responsibility boundary.

## Conceptual Boundary Between Core Truth and Derived Views

The current design direction also clarifies that not every useful blockchain-facing state should be treated as primary truth.

Conceptually, MoonBlokz distinguishes between:

- **authoritative blockchain knowledge** such as known blocks, runtime mempool contents, and the current operating mode,
- and **derived operational views** such as the currently selected active chain and the currently usable balance and UTXO state.

This matters because MoonBlokz must operate under bounded storage and staged validation. A node may safely retain some blockchain knowledge before it is able to derive a fully usable active state from it.

## Why the Tree Cannot Grow Forever

Part IV added a second defining constraint: nodes have limited storage capacity. The chain cannot grow without bound.

The chosen conceptual response is `snake_chain`:

- the chain has a bounded active length,
- when a new block arrives, the oldest block is eventually dropped,
- and any critical information that would otherwise be lost must be reintroduced at the tail of the chain.

This is why the model is called `snake_chain`: the head moves forward while the tail also moves forward.

Conceptually, MoonBlokz is therefore not just a fork-tolerant chain. It is a **moving window of chain state** that must continuously preserve the information required for continued operation.

## Operational Lifecycle Phases

The combined model implies a practical lifecycle rather than one uniform runtime mode.

Conceptually, MoonBlokz moves through three phases:

- **collection phase** — the node gathers blocks and looks for a dominant chain candidate,
- **processing phase** — the node reconstructs and validates enough state from that candidate chain to become operational,
- **ready phase** — the node tracks an active chain, validates new arrivals, and maintains bounded retention.

This lifecycle matters because MoonBlokz does not start from immediate full confidence. It must first discover, then reconstruct, and only after that operate continuously. The companion algorithm document describes the state transitions and operational consequences in more detail.

## Dominant Chain Acquisition as the First Objective

Before a node becomes operational, its first goal is not full historical recovery. Its first goal is to find a sufficiently complete and sufficiently dominant candidate chain as quickly as possible.

This means startup behavior is biased toward:

- finding the branch most likely to represent the active network,
- recovering enough missing ancestry to make that branch usable,
- and reaching an operational state sooner rather than maximizing historical completeness.

This emphasis fits the intended environment. On unreliable radio, recovering enough shared active state quickly can matter more than reconstructing the entire historical tree perfectly.

## Staged Validation

The todo material reinforces an important conceptual reading of the earlier articles: block handling does not have to collapse into a single yes-or-no acceptance moment.

A block may be:

- structurally parseable,
- worth storing,
- connected to known ancestry,
- and still not yet fully validated against all state-dependent rules.

MoonBlokz should therefore be understood as allowing **staged validation**:

- parsing and storage can happen first,
- branch connectivity can be established next,
- full semantic validation may require more chain context,
- and active-chain selection can remain provisional while that context is still being reconstructed.

This follows naturally from unreliable communication, bounded retention, and the late availability of some required state. The algorithm document translates this idea into a concrete block-acceptance pipeline and block-status progression model.

## What Must Survive Tail Deletion

The central conceptual question in Part IV is:

> What information must remain available even after the beginning of the chain has been deleted?

The answer is not “all history.” The answer is “all information necessary to preserve the current valid state.”

This leads to four conceptual block types.

### 1. Transaction blocks

These contain operational financial activity, including:

- node-to-node balance transfers,
- node registration,
- and complex transactions combining balance and UTXO inputs and outputs.

Part V makes the conceptual distinction sharper: transaction types are intentionally separated because MoonBlokz prices flexibility and privacy according to structural cost.

### 2. Balance blocks

These contain many node balances with the metadata required to keep node state reconstructable.

Conceptually, balance blocks are compact checkpoints for the known-node part of the network.

### 3. Chain-config blocks

These store the chain configuration. The article treats configuration as immutable after initialization in the current model.

This means the configuration is not just deployment metadata. It is part of durable chain state and must remain available inside the active chain.

### 4. Approval evidence blocks

These record proof that a branch deviation from the normal accumulated-vote-based creator-selection rule received majority support.

Conceptually, these blocks allow later nodes to understand why a non-default block became legitimate.

## Consistency Under snake_chain

Part III focused on recovering enough of the same block-tree. Part IV added the requirement that the **active chain must always contain enough state to continue operation**. Part V strengthens this by showing that state continuity depends on explicitly structured replayable data, not just informal bookkeeping.

Under `snake_chain`, consistency means more than sharing branch structure. It also means:

- the active chain still contains all node balances needed to reconstruct the current balance model,
- the active chain still contains the chain configuration,
- and any live UTXOs that would otherwise be lost have been reintroduced before their source block disappears.

So consistency becomes both:

- **structural consistency** about branch knowledge,
- and **state continuity** despite bounded retention.

## The “Last one, take the lead!” Preservation Idea

The core preservation rule of Part IV is simple conceptually:

- when an essential block is about to fall off the tail,
- the information that must survive is appended again near the head.

Part V makes clear that this preservation strategy also depends on compact representations:

- repeated chain-config content must remain compact enough to fit normal block production,
- replayed balances must preserve the minimum fields needed for validation,
- and carried-forward UTXOs must use a structure that remains portable across forks and re-additions.

This mechanism turns old historical state into compact current-state continuation.

## Transaction Uniqueness Without Global Time

Part V adds a major conceptual principle: MoonBlokz avoids timestamp-based identity and ordering.

Instead:

- block order is represented by chain sequence,
- balance-style transactions use `anchor_sequence` plus free-form comment data to avoid accidental duplication,
- and UTXO spending uses one-time consumption semantics rather than global-time ordering.

Conceptually, this reflects the same anti-centralization choice seen elsewhere in the series: MoonBlokz avoids relying on globally synchronized clocks because that would introduce external infrastructure dependency and security risk.

## Custodian Fee as a Storage-Pressure Tool

The custodian fee has a conceptual role beyond economics.

It also acts as a pressure mechanism against unbounded address-state accumulation:

- each time a surviving UTXO is re-added to the chain, its value is reduced,
- tiny UTXOs can disappear entirely if their value falls below the fee,
- and users are therefore incentivized to merge UTXOs and free chain space.

This means MoonBlokz uses pricing not only for incentives and privacy, but also for **bounded-state maintenance**.

## Dynamic Custodian Fee (Post-MVP Concept)

This section captures a concept for making the custodian fee chain-saturation-aware, beyond what the MVP supports. In the MVP, the custodian fee is resolved by the `configuration_module` as an **input-less** chain-config-derived value: the same fee applies whether the active chain is nearly empty or nearly full. The chain-stall risk under high UTXO accumulation (described under [UTXO growth can stall the chain](#utxo-growth-can-stall-the-chain)) therefore depends on a fee value tuned in advance for the expected saturation range, with no chain-internal feedback when actual saturation diverges from expectations.

Making the fee responsive to current saturation gives the chain a self-regulating relief mechanism: at low saturation the fee stays small (light pressure on tiny UTXOs); at high saturation the fee scales up (faster reclamation of chain space).

The concept here is intentionally pre-design: it captures the operating idea and the questions a post-MVP requirement and architecture pass must answer. It does not change any MVP-scope requirement.

### Operating idea

The proposed mechanism extends the `configuration_module`’s custodian-fee accessor with one new input: the **count of currently-unspent UTXOs on the active chain**.

This input is cheap to derive from existing data. Under the per-block UTXO spent-bit-vector model (ADR-016), every active-chain block already carries a bit vector whose unset bits identify still-unspent outputs. The active-chain-wide unspent UTXO count is therefore the sum of unset bits across the active-window blocks — an `O(W)` popcount-based computation against state the module already maintains.

With this input available, the configuration content can specify a fee curve rather than a fixed value. The `configuration_module`’s existing mini-VM already supports both literal constants and bytecode-evaluated functions of accessor inputs, and the **registration price** accessor already takes a similar derived-state input (the current registered-node count). The custodian-fee accessor would follow the same pattern: same interface shape, same VM semantics, same fall-back-to-default behavior when a chain-config payload omits the accessor.

### Fit with existing structural decisions

The concept aligns with several decisions that are already part of the MVP model:

- The `configuration_module` boundary remains the canonical source for custodian-fee resolution; only the accessor’s input contract is extended.
- The pattern matches the existing `registration price` accessor, which already takes the registered-node count as input — there is no new architectural shape, only a new input applied to a parameter that today takes none.
- Per-block UTXO spent-bit vectors (ADR-016) already expose the information needed for cheap unspent counting; no new persistent state is introduced.
- Active-chain-centered query semantics (ADR-008) are preserved: the count is derived strictly from the current active chain, not from incidental block-tree contents.
- Replay determinism is preserved as long as the count is sampled at a deterministic moment in the carry-forward flow (open question 1 below).

### Open questions for the post-MVP design

The concept introduces tensions that a post-MVP requirement and ADR pass must resolve. They are deliberately left open here:

1. **Sampling moment.** “The count of unspent UTXOs on the active chain” must pin down exactly when the count is sampled inside the UTXO carry-forward flow, so that every node replaying the same active-chain state arrives at the same fee. Candidate moments include: at the start of carry-forward processing for the dropping tail block, per individual UTXO before it is reduced, or once per accepted block. Each choice has slightly different replay semantics and the post-MVP design must commit to one.
2. **Cross-block consistency under chain switch.** Chain-switch reconciliation can change which blocks are on the active chain, which changes the unspent count, which retroactively changes the fee that would have applied. The post-MVP design must decide whether previously-applied fees are recomputed against the new active chain or whether the fee captured at acceptance time is canonical and carried forward unchanged.
3. **Fee-curve encoding.** The configuration content must express the mapping from `unspent_utxo_count` to fee in a form the configuration-module mini-VM can evaluate cheaply and deterministically on RP2040-class hardware. The post-MVP design must choose the encoding (sampled table, piecewise-linear segments, polynomial coefficients, free bytecode) so that the curve stays compact and bounded in evaluation cost.
4. **Bounds enforcement.** A curve that grows without limit could effectively confiscate small UTXOs; one that stays flat defeats the purpose. The post-MVP design must decide where bounds enforcement lives: as a protocol-level hard ceiling and floor, or as a configuration-content-level convention that the configuration author is responsible for honoring.
5. **Backward compatibility with the input-less custodian-fee accessor.** A chain-config payload that omits the custodian-fee accessor (or supplies one with the current “no input” shape) must continue to resolve to a sensible value. The post-MVP design must decide whether the input-less form is still accepted (the unspent-count input is supplied but ignored by an input-less bytecode) or whether the dynamic form replaces it entirely (forcing chain-config evolution to roll out together).
6. **Observability.** The structured-log record for each fee resolution should expose both the unspent-count input and the resolved fee, so that simulator replay and operator inspection can verify deterministic behavior across nodes and diagnose curves that misbehave under unexpected saturation patterns.

This section captures only the conceptual idea and the open questions. It is not a requirement, does not change any MVP-scope FR (including the current “no input” specification for the custodian-fee accessor), and does not commit to specific fee curves, encodings, or thresholds. The corresponding requirement-level wording — including the extended `configuration_module` custodian-fee accessor input contract — and any architectural decision records will be created when the post-MVP work begins.

## Node Registration Concept

Part IV introduced the conceptual model for adding new nodes, and Part V clarifies that registration is also a compact, explicit transaction type.

MoonBlokz uses a hybrid registration model:

- nodes cannot simply self-join for free by default,
- every node can register a new node,
- registration has a configurable price,
- the new key must prove possession of its private key,
- and the registration price is absorbed by the network rather than paid to a specific node; the ordinary transaction fee is a distinct amount and is handled separately.

The conceptual reason is that adding nodes is not free for the system. More nodes mean:

- more balance data must be preserved,
- more balance blocks are needed,
- and overall efficiency declines.

## Genesis and Chain Initialization

Part IV defines a two-block genesis model:

- **Block #0** is a transaction block created by node `#0`, containing node `#0` registration and an initial self-transfer representing the initial currency.
- **Block #1** is the chain configuration block, also created by node `#0`.

Conceptually, this means MoonBlokz does not begin from a purely symbolic genesis. It begins from the smallest chain state that can bring the system into normal operation.

## Efficient Distribution of Balance Blocks

The chain works most efficiently when balance blocks are evenly distributed across the active chain.

Conceptually, this matters because:

- a new node needs all node balances from the living chain,
- balance blocks that are too clustered can increase transaction delay,
- and replaying dropped balance blocks gives the system a chance to gradually smooth their distribution.

This means `snake_chain` is not only about keeping data alive. It is also about arranging active-state data in a way that supports practical network performance.

## Consensus Objective Under snake_chain

The approval and branch-selection logic from Part III still applies, but Part IV adds a stronger preservation rule:

- if a chain-config block or balance block is dropped from the chain,
- repeating that block type at the tail takes precedence,
- even if an approval-related block also needs to be added.

Conceptually, preserving essential active-state data is the stronger rule because losing it would make the chain itself no longer reconstructable by active participants.

## Efficiency Framing

Parts IV and V explicitly accept that bounded storage and compact encoding introduce trade-offs.

If the chain did not need to delete history and optimize representation sizes, it would not need:

- repeated balance blocks,
- repeated chain configuration,
- UTXO carry-forward work caused by tail deletion,
- or carefully minimized field sets in transaction and block representations.

The conceptual claim of the series is that these are practical compromises required by the actual deployment environment, not design mistakes.

## Selective Forgetting of Non-Active Branches

Bounded storage limits not only the active chain window but also how many non-active branches can be preserved in parallel.

MoonBlokz may therefore forget some branches deliberately:

- branch tips still compete for scarce storage,
- older or weaker branch endings may become less worth retaining,
- and pruning a branch can be an operational choice rather than proof that the branch was impossible.

The MoonBlokz block-tree is therefore not a full immutable historical forest. It is a bounded working set of branch knowledge maintained for continued operation under constrained storage.

## Immutable but Late-Validated Chain Configuration

The todo material suggests a useful clarification of the current configuration model.

Chain configuration is still treated as **immutable durable state** in the current MoonBlokz design. However, complete validation of that configuration may depend on having enough blockchain context available first.

Both statements can therefore be true at the same time:

- the chain configuration does not change during normal operation in the current model,
- but a node may only confirm all configuration-dependent rules after enough chain state has been reconstructed.

This is another example of MoonBlokz favoring staged reconstruction over instant complete understanding.

## Local Query Surface as a Product Boundary

The current design direction adds one more useful conceptual boundary: the local side of the blockchain module is not only for debugging or maintenance. It is also the foundation of a future payment-facing interface.

That local-facing side should therefore expose active-chain-centered answers rather than full internal branch observability by default.

Examples include:

- transaction status queries that distinguish between unknown, present in mempool, and present in the active chain,
- active-chain block lookup rather than arbitrary block-tree exploration,
- and balance queries that can optionally report how deeply the visible answer is supported within the active chain.

This keeps the internal block-tree and staged-validation complexity inside the blockchain module while still exposing a useful operational truth surface to higher-level local consumers.

## Resynchronization Limitation

The main conceptual limitation of `snake_chain` is that resynchronization works only while separated nodes still share some common block history with the active chain.

If a node is disconnected so long that the entire active chain window moves past the last shared block, then:

- the node can no longer resynchronize,
- and the network suffers a permanent fork relative to that node’s history.

This is one of the most important conceptual boundaries in the current MoonBlokz model. The system is designed for long but still bounded disconnection tolerance, not arbitrary offline recovery after unlimited time.

## Long-Disconnect Recovery (Post-MVP Concept)

This section captures a concept for partial recovery from long-disconnect states, beyond what the MVP currently supports. The MVP keeps long-disconnect resynchronization out of scope: a node that detects the long-disconnect condition (an incoming block whose sequence is so far ahead that the active chain window cannot be bridged by ordinary parent recovery) today only emits a diagnostic log entry, and external operator intervention is the only recovery path. For deployments where occasional long disconnects are expected as normal operating conditions, a chain-internal recovery path would substantially reduce operator burden.

The concept here is intentionally pre-design: it captures the operating idea and the questions a post-MVP requirement and architecture pass must answer. It does not change any MVP-scope requirement.

### Trust source: the node’s own known active-chain participants

The recovery concept’s trust source is the node’s own (potentially stale) active-chain knowledge: the set of nodes the local active chain considers active, in the sense already used by the approval-subgroup concept (ADR-015). The local node trusts that those known nodes were honest when they entered its active chain, and uses a majority of a deterministically-selected subgroup of them to authenticate the recovery anchor.

This is a partial-coverage trust model by construction. The further the local active chain has fallen behind reality, the more likely it is that a non-trivial fraction of the trusted set is no longer reachable, no longer holds the requested block, or has rotated its keys. The mechanism is intentionally probabilistic: it works under wide but not universal boundary conditions and is not a substitute for staying connected.

### Outline of the proposed mechanism

The proposed mechanism works, at a conceptual level only, as follows:

1. **Watcher activation.** In ready state, when an incoming block’s sequence is too far ahead to be bridged by ordinary parent recovery (matching the existing long-disconnect detection condition), the node activates an internal long-disconnect watcher. Normal ready-state operation on the local active chain continues unchanged during watcher activity; the watcher is purely additive observation.
2. **Anchor capture.** The watcher counts consecutive incoming too-new blocks. After observing `n` of them (where `n` is a chain-config parameter), the watcher retains the `n`-th observed block as a **recovery anchor** in a dedicated single-slot recovery state outside the regular block-tree, so that the anchor is not subject to capacity-pressure eviction.
3. **Subgroup determination.** The node forms a **recovery subgroup** of `m` known active-chain nodes (where `m` is a chain-config parameter). Subgroup membership is derived from a deterministic ordering whose seed combines the local node’s private PRNG state (initialized from the node’s private key) with the anchor’s sequence. This pattern echoes the approval-subgroup approach of ADR-015 but uses a private rather than a public seed: the recovery subgroup is the requester’s own private querying decision rather than a globally verifiable consensus selection, so external verifiability of the choice is traded for external unpredictability of who will be queried.
4. **Authentication request.** The node emits a recovery-request message carrying the anchor’s sequence, its hash, and the subgroup member identifiers. The message propagates through the radio layer’s existing relay pipeline like any other blockchain message.
5. **Confirmation response.** A subgroup member that receives the request checks whether the requested `(sequence, hash)` matches a block in its own active chain. If it does, the member emits a confirmation message signed over the `(sequence, hash)` value.
6. **Majority-confirmation outcome.** When the requesting node has collected confirmation messages from at least `50 % + 1` of the subgroup, it deletes its stored blocks, saves the recovery anchor as the only retained block, and transitions back into collecting state. The ordinary collecting → processing → ready lifecycle then re-establishes operational state against the anchor.
7. **Retry on insufficient confirmations.** If insufficient confirmations are received, the watcher waits until another `n` consecutive too-new blocks have been observed and retries against the newly-arrived anchor sequence, which yields a fresh subgroup through the changed PRNG-plus-sequence seed.

### Fit with existing structural decisions

The concept aligns with several decisions that are already part of the MVP model:

- The `ready → collecting → processing → ready` re-entry path uses the lifecycle already specified by the chain lifecycle and recovery requirements; no new lifecycle state is introduced.
- The subgroup-based authentication pattern reuses the conceptual approach of the approval-subgroup mechanism (ADR-015).
- The trust-from-known-participants idea is consistent with how accumulated-vote-based creator selection and approval evidence already work.
- “The recovered chain must still be the same chain” follows naturally from the set-once chain-configuration lock: a recovery anchor whose chain belongs to a different chain-config will fail the durable-lock match check during the subsequent processing → ready transition, which is the correct behavior under the existing “no cross-chain merging” boundary.

### Open questions for the post-MVP design

The concept introduces tensions that a post-MVP requirement and ADR pass must resolve. They are deliberately left open here:

1. **Modification of the current long-disconnect intake behavior.** The current ready-state rule discards too-new blocks entirely. The watcher concept requires retaining at least the recovery anchor outside the regular block-tree, so the intake rule for too-new blocks needs an explicit watcher-mode exception.
2. **Same-chain-only recovery as an explicit boundary.** The concept recovers only from a disconnect on the same chain (same locked chain-configuration content). Cross-chain merging remains out of scope, as in the MVP; the post-MVP wording must state this explicitly so the durable-lock failure path is treated as expected behavior rather than as a recovery defect.
3. **Stale-subgroup probabilistic coverage.** The longer the disconnect, the more likely it is that subgroup members are no longer reachable. Deployment guidance must communicate that the recovery is “wide but not universal”, and the post-MVP design must decide how many retries are reasonable before the watcher gives up.
4. **Private vs. public subgroup seed.** Using a private PRNG seed (derived from the requester’s private key) is intentional: external attackers cannot pre-position colluding subgroup members because they cannot predict who will be queried. The trade-off is that the subgroup choice is not externally verifiable. This is acceptable because the subgroup is a private querying decision rather than a consensus input. The contrast with the public-seed approval-subgroup of ADR-015 must be made explicit in a future ADR so that the two patterns are not conflated.
5. **Counter semantics for “n consecutive too-new blocks”.** The precise definition of consecutive — arrival order vs. distinct sequences, reset behavior when a non-too-new arrival interleaves, treatment of duplicate retransmissions, treatment of fork siblings at the same sequence — is left for post-MVP refinement.
6. **Confirmation-collection mechanics.** Timeout vs. count-bounded waiting, deduplication of confirmations by signer node identifier, partial-confirmation carry-over across retries, and the choice between aggregated and individual signatures (the existing aggregated-evidence pattern is a natural candidate) are all post-MVP design choices.
7. **Order of operations: confirm before delete.** The subgroup-selection input depends on the node’s current active chain. Deletion of stored blocks must therefore happen only after the `50 % + 1` confirmations have been processed, so that the trust basis remains available until the recovery is committed.
8. **Retry mechanics.** The proposed retry on each subsequent batch of `n` too-new blocks yields a fresh subgroup through the changed seed. Whether partially-collected confirmations carry over between retries, and how the watcher chooses between waiting longer with the current anchor and rolling forward to a new anchor, is left for post-MVP design.
9. **Replay determinism for the private seed.** Because the subgroup selection consumes the node’s private PRNG state, deterministic replay (a core MVP property) requires that this state be captured as part of the replay input set, in the same spirit as the existing node-identity and trust-anchor initialization parameters.
10. **Anchor pinning.** Holding the recovery anchor in a dedicated single-slot recovery state outside the block-tree avoids any interaction with capacity-pressure eviction. If a post-MVP design instead places the anchor inside the block-tree, an explicit pin flag would be required to prevent the anchor from being evicted before the recovery completes.
11. **Subgroup-identifier disclosure.** The recovery-request message necessarily lists the subgroup member identifiers so that recipients can recognize whether the request concerns them. This makes the subgroup choice observable to attackers — a minor observability cost accepted as the price of relay-compatible addressing.
12. **New blockchain message types.** The recovery request and the confirmation response are new message types in the radio-layer’s blockchain-message set. The relay pipeline can forward them opaquely, but the message-type enumeration and any chain-config-bound size constraints (driven by the `m` subgroup-identifier list) must be defined explicitly.
13. **Sybil exposure against a long-disconnected trusted set.** If an adversary obtains control of a sufficient fraction of the local active chain’s known nodes over the course of a long disconnect (through compromise or key compromise), they can produce a successful but fraudulent confirmation set. This is the general weakness of any trust-from-known-set system and is not unique to this mechanism; the post-MVP design must acknowledge it explicitly rather than overstating the recovery’s security guarantee.

This section captures only the conceptual idea and the open questions. It is not a requirement, does not change any MVP-scope FR, and does not commit to specific parameter ranges, message layouts, or aggregation strategies. The corresponding requirement-level wording — including the watcher-mode exception to the current long-disconnect-intake rule — and any architectural decision records will be created when the post-MVP work begins.

## Processing as a Concurrent-Ingestion State (Post-MVP Concept)

This section captures a concept for promoting **processing** from a transient transition to a first-class operating state that keeps ingesting blocks while it runs, beyond what the MVP supports. In the MVP model, processing is a transient reconstruct-and-validate pass rather than a resting phase (see [Operational Lifecycle Phases](#operational-lifecycle-phases)): it runs as a single forward pass that is not resumable across restarts ([Blockchain PRD](./moonblokz-blockchain-prd.md) FR3 / FR59), and on any contradiction it falls back to the collecting phase under FR5 atomic recovery. Blocks that arrive while the node is in processing are preserved but are not claimed finally valid.

On RP2040-class hardware, reconstruct-and-validate of a long candidate chain can be slow. If that pass behaves as a blocking transition, a slow reconstruction extends the window during which the node cannot advance and complicates keeping up with concurrently arriving blocks. Making processing a real operating state — one that continues to accept and store incoming blocks while the reconstruction proceeds, on equal footing with the collecting and ready phases — lets a node stay abreast of network activity even when its own reconstruction takes a long time. Under this concept the lifecycle becomes a genuine three-state model (**collecting**, **processing**, **ready**) rather than two phases joined by a single processing transition.

The concept here is intentionally pre-design: it captures the operating idea and the questions a post-MVP requirement and architecture pass must answer. It does not change any MVP-scope requirement.

### Operating idea

The proposed mechanism keeps the reconstruct-and-validate work running against a pinned candidate chain while ordinary block intake continues alongside it:

- incoming blocks are parsed, stored, and connected to known ancestry during processing, exactly as the existing [staged validation](#staged-validation) model already allows, instead of being merely held pending completion of the pass;
- the reconstruction pass targets a specific candidate chain captured when processing begins, so that its forward traversal remains well-defined even as new blocks arrive;
- when the pass completes successfully the node promotes to ready; when it fails it falls back to collecting under the existing atomic-recovery rule, and the blocks ingested concurrently remain available to the next collecting round rather than being discarded with the failed working set.

### Fit with existing structural decisions

The concept aligns with several ideas already part of the MVP model:

- Staged validation ([Staged Validation](#staged-validation)) already separates parse/store/connect from full semantic validation, so concurrent intake during processing reuses an existing conceptual seam rather than introducing a new one.
- The two operational phases and the processing transition are already named states in the lifecycle; the concept promotes the transition to a peer state without inventing a new vocabulary.
- Active-chain-centered query semantics (ADR-008) still define what the node may answer; a processing node continues to have no promoted ready active chain until the pass completes.

### Open questions for the post-MVP design

The concept introduces tensions that a post-MVP requirement and ADR pass must resolve. They are deliberately left open here:

1. **Non-resumability contract.** The current pass is a single forward pass that is not resumable across restarts (FR3 / FR59). A concurrent-ingestion processing state must decide whether the reconstruction pass stays atomic-and-non-resumable while only intake runs alongside it, or whether processing itself becomes interruptible and must define resume semantics. This directly affects FR3 / FR59 and must be reconciled with them.
2. **Atomic-recovery scope on failure.** FR5 atomic recovery today discards the forward-traversal working set in full on any contradiction. The design must define precisely which state is rolled back and which concurrently-ingested blocks survive into the fallback collecting round, so that recovery stays atomic without throwing away independently-valid intake.
3. **Candidate stability vs. re-targeting.** Blocks arriving during processing may extend or reveal a stronger candidate branch than the one the pass is reconstructing. The design must decide whether the in-flight pass completes against its pinned candidate and re-evaluates afterward, or whether it is abandoned and re-targeted mid-pass, and how either choice preserves replay determinism.
4. **Bounded-storage interaction.** Concurrent intake competes with the reconstruction working set for bounded storage and RAM. The design must define how capacity-pressure eviction (and the reconstruction working set’s own footprint) behave while both proceed, so that intake during processing cannot starve or corrupt the pass.
5. **Lifecycle-model reconciliation.** Promoting processing to a first-class state changes the “two phases joined by a processing transition” model — stated in the [Blockchain PRD](./moonblokz-blockchain-prd.md) FR1–FR5 and echoed across the concept and algorithm documents — into a three-state model. The post-MVP wording must update the authoritative lifecycle contract and reconcile every restatement of the two-phase framing in the same change.
6. **Observability.** Structured logging should distinguish a processing node that is making reconstruction progress from one that is merely accumulating intake, so that operators and simulator replay can tell a slow-but-advancing pass from a stalled one.

This section captures only the conceptual idea and the open questions. It is not a requirement, does not change any MVP-scope FR (including the current FR3 / FR59 single-forward-pass, non-resumable processing specification), and does not commit to a concurrency model, a resume mechanism, or a revised lifecycle-state contract. The corresponding requirement-level wording and any architectural decision records will be created when the post-MVP work begins.

## Security Framing

The article series continues to separate algorithmic protection from physical-world limits.

MoonBlokz does not claim to solve attacks such as:

- radio flooding by a stronger adversary,
- physical isolation and controlled misinformation,

Part V reinforces the design choice to avoid global time as a trust dependency.

## Main Conceptual Limitations

### Bounded history is intentional

MoonBlokz deliberately gives up permanent full-history retention on every node.

### Richer transactions cost more

Complex and privacy-preserving transactions consume more bytes and therefore more scarce chain capacity.

### UTXO growth can stall the chain

If live UTXOs occupy all available chain space, the chain may temporarily stop accepting new transactions until repeated custodian-fee reductions shrink the UTXO set. A post-MVP concept for making the custodian fee respond to active-chain UTXO saturation is captured under [Dynamic Custodian Fee (Post-MVP Concept)](#dynamic-custodian-fee-post-mvp-concept); the limitation above describes the MVP behavior under a fixed-by-default custodian fee and is not changed by the existence of that concept section.

### Long disconnects can become permanent forks

If no common active-chain overlap remains, rejoin is impossible in the current MVP model. A partial recovery approach based on a known-active-node subgroup is captured as a post-MVP concept under [Long-Disconnect Recovery (Post-MVP Concept)](#long-disconnect-recovery-post-mvp-concept); the limitation above describes the MVP behavior and is not changed by the existence of that concept section.

### Slow processing can stall progress

On constrained hardware the processing reconstruct-and-validate pass can be slow, and in the MVP it behaves as a transient transition rather than a resting state that keeps ingesting blocks. A post-MVP concept for promoting processing to a first-class operating state that continues to accept blocks while it runs is captured under [Processing as a Concurrent-Ingestion State (Post-MVP Concept)](#processing-as-a-concurrent-ingestion-state-post-mvp-concept); the limitation above describes the MVP behavior and is not changed by the existence of that concept section.

### Configuration mutability is deferred

The article mentions future configuration changes as a possibility, but the current conceptual model treats chain configuration as fixed after initialization.

## Architect View: Structural Meaning of Parts III, IV, and V

From an architectural point of view, the combined model establishes several durable ideas:

- the effective blockchain state is chosen from a tree, not assumed from a single chain,
- local knowledge matters more than globally synchronized rounds,
- state continuity must survive tail deletion,
- chain configuration is part of durable blockchain state,
- balances and UTXOs have different preservation strategies,
- storage bounds are a first-class protocol design constraint,
- and compact binary representations are part of protocol design because radio, RAM, and flash limitations are inseparable from correctness.

## Related Documents

- [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) — the formal current flow of blocks, transactions, voting, and chain reconciliation.
- [`moonblokz-blockchain-implementation.md`](./moonblokz-blockchain-implementation.md) — code-level crate structure, APIs, and engineering cautions.
