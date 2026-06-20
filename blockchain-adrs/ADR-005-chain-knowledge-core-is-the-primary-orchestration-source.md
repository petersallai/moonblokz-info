# ADR-005: Chain Knowledge Core Is the Primary Orchestration Source

### Status
Accepted

### Context
The blockchain module contains multiple internal concerns: known blocks, branch handling, active-chain selection, recovery, approval, `snake_chain` preservation, mempool behavior, vote state, and derived economic state. Without a clear internal organizing principle, these concerns could become loosely coupled competing centers of truth.

The broader architecture already distinguishes authoritative truth from derived operational state. The module therefore needs one internal source that owns chain truth and drives reconciliation of the rest.

### Decision
The module is internally organized around a **Chain Knowledge Core** as the primary orchestration source: it owns known blocks, branch and connectivity knowledge, active-chain selection, operation mode, and the recovery / approval / `snake_chain` consequences. Derived subsystems (economic-state cache, mempool registry, vote module) maintain their own internal state but are corrected from the Chain Knowledge Core when active-chain truth changes.

The 15 + 1 internal modules that realize this organization (`lifecycle`, `chain_heads`, `snake_chain`, `branch_value`, `reconciliation`, `node_info`, `creator`, `approval`, `queries`, …) are normative in [`moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) §4.

### Consequences
#### Positive
- Gives the module a clear internal backbone.
- Avoids multiple competing truth sources.
- Makes chain-switch handling conceptually cleaner.
- Supports a consistent reconciliation model for derived subsystems.
- Aligns the internal structure with the authoritative-versus-derived state model.

#### Trade-offs
- Makes the Chain Knowledge Core architecturally critical and therefore design-sensitive.
- May centralize too much logic if internal boundaries are not kept disciplined.
- Requires carefully designed update and reconcile flows.
- Increases the importance of defining core invariants precisely.

### Follow-up implications
This decision implies that later design work must define:
- the internal representation of known blocks, branches, and connectivity,
- the invariants of the Chain Knowledge Core,
- and the correction workflow used when derived subsystems must be reconciled after active-chain change.
