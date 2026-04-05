# ADR-003: Authoritative vs Derived State in `moonblokz-blockchain`

### Status
Accepted

### Context
The blockchain module maintains many kinds of useful state: known blocks, mempool contents, active chain, balances, UTXOs, vote state, and operating mode. Under bounded resources, staged validation, and rare but legitimate chain switches, not all of these should be treated as equally authoritative durable truth.

MoonBlokz needs a deliberately small and defensible truth model so that restart behavior, reconciliation, persistence, and runtime caches remain understandable.

### Decision
`moonblokz-blockchain` distinguishes between **authoritative state** and **derived operational state**.

Authoritative state:
- known blocks,
- mempool,
- operation mode.

Derived operational state:
- active chain,
- balance / UTXO truth,
- vote / next-creator state.

Derived state may be incrementally maintained for performance, but it is not the primary durable truth source.

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
