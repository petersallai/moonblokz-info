# ADR-003: Authoritative vs Derived State in `moonblokz-blockchain`

### Status
Accepted

### Context
The blockchain module maintains many kinds of useful state: known blocks, mempool contents, active chain, balances, UTXOs, vote state, and operating mode. Under bounded resources, staged validation, and rare but legitimate chain switches, not all of these should be treated as equally authoritative durable truth.

MoonBlokz needs a deliberately small and defensible truth model so that restart behavior, reconciliation, persistence, and runtime caches remain understandable.

### Decision
`moonblokz-blockchain` distinguishes **durable authoritative truth** — the node's own identity (`node_id` + private key), the saved `chain_config` block, and the retained blocks of the block-tree — from **derived operational state** that is recomputed from the active chain plus the `snake_chain` window and is never primary durable truth. Two runtime states sit outside the durable set and are neither: the **mempool** is network-held and non-durable (empty on restart, repopulated by ordinary network activity), authoritative only for its own contents per [ADR-009](./ADR-009-mempool-is-independent-but-chain-governed.md) and never blockchain truth; and the **operation mode** is not persisted, since a restart enters collecting state unconditionally. Derived state may be incrementally maintained for performance, but it is not primary durable truth.

The durable persistence boundary is normative in [PRD](../moonblokz-blockchain-prd.md) FR59; the six derived projections (per-node balance, accumulated vote, public-key mapping, block-navigation/UTXO cache, max-node-id watermark, seed-source-sequence) and their incremental-update trigger events are normative in FR34 and FR35; the mempool's non-durable independence is normative in FR30. The architectural realization (`NodeInfo` SoA, `ChainHeadsTable`, `VoteEngine`, etc.) is in [`moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) §1.2 and §4–§6.

### Consequences
#### Positive
- Keeps the durable truth model compact.
- Makes restart and recovery behavior clearer.
- Fits staged validation naturally.
- Supports explicit recomputation or reconciliation after chain switch.
- Helps prevent accidental creation of multiple competing truth sources.

#### Trade-offs
- Requires clear rules for when and how derived views are corrected.
- Increases the need for strong invariants around caches and reconciled state.
- May complicate reconciliation logic during active-chain changes.
- Requires careful communication so derived state is not mistaken for durable truth.

### Follow-up implications
This decision implies that later ADRs or design notes must define:
- the persistence threshold for durable blockchain storage,
- the Chain Knowledge Core as the primary orchestration source,
- and the reconcile workflow for derived state during active-chain changes.
