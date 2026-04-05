# ADR-013: Bounded UTXO Retention Requires an Explicit Preservation Strategy

### Status
Accepted

### Context
MoonBlokz combines bounded blockchain history with UTXO-based address privacy. This creates direct pressure between preserving live address-state and keeping the chain usable for normal operation. Without an explicit policy, UTXO carry-forward could become an uncontrolled implementation detail that silently consumes useful chain capacity.

The brainstorming session identified bounded UTXO handling as one of the most dangerous architecture points in the blockchain design.

### Decision
MoonBlokz requires an explicit **bounded UTXO preservation strategy** rather than treating UTXO carry-forward as an incidental implementation detail.

That strategy must define:
- how live UTXOs are identified for carry-forward,
- how they compete with normal transaction capacity,
- how custodian-fee reduction participates in state pressure relief,
- and what happens when UTXO preservation threatens normal chain progress.

### Consequences
#### Positive
- Treats one of the core bounded-history tensions explicitly.
- Supports clearer reasoning about stall conditions and capacity pressure.
- Keeps UTXO handling aligned with `snake_chain` realities.
- Makes the privacy-versus-throughput trade-off visible rather than hidden.

#### Trade-offs
- May reveal difficult trade-offs between privacy support and throughput.
- Requires careful block-capacity policy design.
- May require simulator or model-based validation before the policy is trustworthy.
- Increases the need for explicit saturation and fallback behavior.

### Follow-up implications
This decision implies that later design work must define:
- the carry-forward policy and saturation thresholds,
- the role of custodian-fee reduction in practical pressure relief,
- and the behavior of the system when UTXO preservation begins to dominate usable block capacity.
