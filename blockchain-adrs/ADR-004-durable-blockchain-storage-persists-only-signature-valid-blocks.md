# ADR-004: Durable Blockchain Storage Persists Only Signature-Valid Blocks

### Status
Accepted

### Context
MoonBlokz uses staged validation. A block may be worth retaining before all state-dependent validation becomes possible, especially when parents are missing or chain context is incomplete. At the same time, the durable storage boundary must remain strict enough to avoid persisting arbitrary runtime artifacts as blockchain truth.

The blockchain truth model already distinguishes between authoritative durable state and derived helper state. Durable storage therefore needs a clear persistence threshold.

### Decision
Persistent blockchain storage holds only received blocks that have crossed the persistence threshold, currently defined as at least **signature-valid blocks**.

This includes signature-valid blocks whose parents are still missing.

Parent/child indexes, branch-navigation aids, and similar helper structures may exist, but they are treated as derived implementation support rather than primary durable truth.

### Consequences
#### Positive
- Aligns with staged validation.
- Supports operation under missing-parent conditions.
- Keeps durable truth narrow and defensible.
- Avoids treating helper indexes as canonical blockchain state.
- Makes restart and repair behavior conceptually cleaner.

#### Trade-offs
- Requires clear handling for blocks that are durable but not yet fully semantically validated.
- Requires helper structures to be rebuildable or replaceable.
- May complicate restart and repair flows if runtime indexes become expensive to reconstruct.
- Raises the importance of defining minimum persisted metadata clearly.

### Follow-up implications
This decision implies that later design work must define:
- the minimum metadata stored alongside persisted signature-valid blocks,
- the rebuild expectations for derived helper indexes,
- and the state progression from persistable to fully semantically validated to active-chain selected.
