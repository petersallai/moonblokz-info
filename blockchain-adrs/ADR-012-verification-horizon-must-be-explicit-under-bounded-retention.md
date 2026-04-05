# ADR-012: Verification Horizon Must Be Explicit Under Bounded Retention

### Status
Accepted

### Context
MoonBlokz uses bounded retention through `snake_chain`, but rare chain switches and staged validation still require enough retained knowledge to keep active-chain corrections verifiable. If retention is too narrow, later branch changes may become too expensive, too weakly checkable, or both.

The architecture therefore needs more than an active-chain window. It also needs an explicit understanding of how much retained knowledge is required to preserve practical verifiability.

### Decision
MoonBlokz defines verification horizon as an explicit architecture concern rather than treating the active-chain window as the only implicit validation boundary.

Verification horizon determines how much retained chain knowledge or equivalent validation support is needed so that rare active-chain changes remain verifiable and manageable under bounded resources.

### Consequences
#### Positive
- Makes the storage/verification trade-off explicit.
- Improves architecture clarity around rare chain changes.
- Supports more predictable revalidation behavior.
- Prevents bounded retention from silently eroding confidence in rare but important branch corrections.

#### Trade-offs
- May increase storage and helper-index pressure.
- Requires design work to define the minimal acceptable retained horizon.
- May expose tension between correctness margin and embedded limits.
- May require simulator or implementation-driven measurement before the horizon can be tuned confidently.

### Follow-up implications
This decision implies that later design work must define:
- the retained information needed beyond the immediate active-chain window,
- the relationship between verification horizon and chain-switch reconciliation,
- and the practical limits beyond which branch changes are no longer cheaply verifiable.
