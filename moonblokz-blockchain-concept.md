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

These record proof that a branch deviation from the normal vote-based creator-selection rule received majority support.

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

## Node Registration Concept

Part IV introduced the conceptual model for adding new nodes, and Part V clarifies that registration is also a compact, explicit transaction type.

MoonBlokz uses a hybrid registration model:

- nodes cannot simply self-join for free by default,
- every node can register a new node,
- registration has a configurable price,
- the new key must prove possession of its private key,
- and the registration fee is absorbed by the network rather than paid to a specific node.

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

## Resynchronization Limitation

The main conceptual limitation of `snake_chain` is that resynchronization works only while separated nodes still share some common block history with the active chain.

If a node is disconnected so long that the entire active chain window moves past the last shared block, then:

- the node can no longer resynchronize,
- and the network suffers a permanent fork relative to that node’s history.

This is one of the most important conceptual boundaries in the current MoonBlokz model. The system is designed for long but still bounded disconnection tolerance, not arbitrary offline recovery after unlimited time.

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

If live UTXOs occupy all available chain space, the chain may temporarily stop accepting new transactions until repeated custodian-fee reductions shrink the UTXO set.

### Long disconnects can become permanent forks

If no common active-chain overlap remains, rejoin is impossible in the current model.

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

## Technical Writer View: How to Read This Document

For knowledge-base purposes, this file should be read as the answer to these questions:

- What kinds of actors exist in MoonBlokz?
- Why is a block-tree normal in this environment?
- Why can MoonBlokz not keep the whole chain forever?
- Why does compact data representation matter conceptually, not just technically?
- What information must survive tail deletion?
- Why are balances, configuration, approvals, and UTXOs treated differently?
- Why does MoonBlokz avoid global time?
- What is the practical conceptual limit of resynchronization?

Readers looking for detailed structure layouts, field definitions, or implementation-facing storage concerns should continue with the companion files.

## Related Documents

- [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md)
- [`moonblokz-blockchain-implementation.md`](./moonblokz-blockchain-implementation.md)

## Review Notes

Post-change review against `moonblokz-info` rules:

- **Consistency:** This document now integrates Part V’s compact-data perspective while staying aligned with the Part III block-tree and Part IV `snake_chain` model.
- **Logical soundness:** The file keeps structural and economic consequences at the conceptual level and leaves detailed field layouts to the algorithm document.
- **Feasibility:** The document reflects the real operating constraints of MoonBlokz by treating representation size, bounded storage, and no-global-time assumptions as core design properties.