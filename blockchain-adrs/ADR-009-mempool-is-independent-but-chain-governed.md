# ADR-009: Mempool Is Independent but Chain-Governed

### Status
Accepted

### Context
The mempool must accept pending transactions and supply candidates for block creation, but it cannot become an independent source of blockchain truth. Active-chain changes may invalidate, confirm, or reactivate transactions depending on which branch is selected.

The internal architecture already places the Chain Knowledge Core above other working subsystems as the primary source of blockchain truth.

### Decision
The mempool is modeled as an **independent registry** with its own runtime behavior — it accepts pending transactions, retains them while relevant, and proposes block candidates — but it remains **chain-governed**: when active-chain truth changes, the Chain Knowledge Core drives mempool correction (removing newly confirmed transactions, reintroducing transactions that fell off the previously active chain when still eligible). The mempool supports blockchain operation but does not define blockchain truth.

The mempool's separate-crate split (`moonblokz-mempool`), compact byte-buffer storage, and the chain-driven reconciliation contract are normative in [`moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) §3.3 and §4.2 (`reconciliation.rs`); the FR-level rules are normative in [PRD](../moonblokz-blockchain-prd.md) FR30 (separate-module structure), FR32 (forward-extension and chain-switch reconciliation), and FR33 (randomized eviction).

### Consequences
#### Positive
- Keeps mempool logic practical and modular.
- Preserves blockchain truth as the controlling source.
- Makes chain-switch consequences explicit.
- Aligns mempool behavior with the authoritative-versus-derived state model.

#### Trade-offs
- Requires clear eligibility rules for reintroduction.
- May create non-trivial reconcile behavior under branch changes.
- Requires the mempool to tolerate restart loss without breaking durable blockchain truth.
- Increases the importance of clear stale-transaction and replay policy.

### Follow-up implications
This decision implies that later design work must define:
- mempool reintroduction policy after active-chain change,
- stale-transaction policy,
- and the relationship between mempool contents and block-proposal selection.
