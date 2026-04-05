# ADR-009: Mempool Is Independent but Chain-Governed

### Status
Accepted

### Context
The mempool must accept pending transactions and supply candidates for block creation, but it cannot become an independent source of blockchain truth. Active-chain changes may invalidate, confirm, or reactivate transactions depending on which branch is selected.

The internal architecture already places the Chain Knowledge Core above other working subsystems as the primary source of blockchain truth.

### Decision
The mempool is modeled as an **independent registry** with its own runtime behavior, but it remains **chain-governed**.

Normal responsibilities:
- accept pending transactions,
- retain them while relevant,
- propose block candidates.

When active-chain truth changes, the Chain Knowledge Core drives mempool correction by:
- removing transactions now confirmed in the new active chain,
- and reintroducing transactions that fell out of the previously active chain when they remain eligible.

The mempool therefore supports blockchain operation, but it does not define blockchain truth.

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
