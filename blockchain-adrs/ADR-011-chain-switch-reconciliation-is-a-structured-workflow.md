# ADR-011: Chain-Switch Reconciliation Is a Structured Workflow

### Status
Accepted

### Context
Active-chain switches are rare in MoonBlokz, but they are legitimate and must be handled correctly. A chain switch affects more than the selected-branch pointer: vote state, derived balances / UTXOs, and mempool contents may all require correction.

The internal architecture already places the Chain Knowledge Core above other derived and working subsystems. A branch change therefore needs an explicit correction model rather than ad hoc local fixes.

### Decision
Active-chain switch is handled as a **structured reconciliation workflow** rather than as a small side effect of changing the selected branch.

The Chain Knowledge Core:
- identifies the common ancestor,
- walks backward as needed,
- then walks forward along the new active branch,
- and drives incremental correction of derived subsystems such as vote state, economic cache, and mempool.

### Consequences
#### Positive
- Makes chain-switch consequences explicit.
- Aligns with the authoritative-versus-derived state model.
- Provides one coherent way to repair multiple dependent subsystems.
- Prevents branch changes from becoming scattered local side effects.

#### Trade-offs
- Requires careful definition of common-ancestor handling.
- May still be expensive in rare worst-case situations.
- Needs clear invariants to confirm post-switch correctness.
- Requires later decisions about when incremental reconcile is sufficient versus when broader rebuild is needed.

### Follow-up implications
This decision implies that later design work must define:
- the detailed reconcile workflow for each derived subsystem,
- the invariants checked after reconciliation,
- and the fallback behavior when incremental correction is not sufficient.
