# ADR-010: Randomized Mempool Eviction Under Capacity Pressure

### Status
Accepted

### Context
The mempool is capacity-bounded and may need to evict transactions under congestion. Deterministic eviction policies would tend to make many nodes discard similar transactions, reducing the chance that the network as a whole preserves a diverse transaction set.

MoonBlokz operates in a best-effort distributed environment where whole-network transaction survival matters more than making every node keep the same subset of pending transactions.

### Decision
When mempool capacity is exceeded, eviction should be **randomized** rather than purely deterministic.

The purpose is to maximize the likelihood that different nodes retain different subsets of transactions, improving whole-network transaction survival under congestion.

### Consequences
#### Positive
- Improves diversity of retained transactions across the network.
- Aligns with MoonBlokz’s best-effort distributed operating model.
- Avoids synchronized local eviction patterns.
- Treats congestion behavior as a network-level resilience tool, not only a local cache policy.

#### Trade-offs
- May feel less intuitive than fee- or age-based deterministic policies.
- May complicate local explainability.
- Requires care if some future transaction-priority model is introduced.
- Requires later definition of the precise randomization policy.

### Follow-up implications
This decision implies that later design work must define:
- whether eviction is fully uniform random or constrained by eligibility classes,
- how randomization interacts with any future priority concepts,
- and how mempool observability should explain eviction outcomes if needed.
