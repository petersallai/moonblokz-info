# ADR-014: Mode Transitions May Require Representation-Aware Design

### Status
Accepted

### Context
MoonBlokz blockchain behavior includes collection, processing, and ready-like operational phases. A key architecture question is whether these modes can share one dominant internal representation or whether some of them require distinct helper structures or transitional representations.

The blockchain lifecycle is not only a naming distinction. Different modes may place different demands on chain knowledge, validation maturity, branch bookkeeping, and derived-state support.

### Decision
Mode transitions are treated as **representation-aware design points**: it is not safe to assume that all operational modes (collecting, processing, ready) share one unchanged internal structure with only a flag difference. The architecture must explicitly evaluate which data structures are shared across modes, which change meaning across modes, and which require mode-specific support.

The three-type Owned + View + Builder model for `Block` / `Transaction` (per FR61) that realizes this representation-awareness is normative in [`moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) §3.2; the `EmitScratch` outcome view source is normative in §6.6.

### Consequences
#### Positive
- Prevents underestimating lifecycle complexity.
- Helps avoid accidental misuse of structures across modes.
- Keeps lifecycle-driven design explicit.
- Encourages deliberate treatment of collection, processing, and ready behavior.

#### Trade-offs
- May increase implementation complexity.
- May require transitional representations or conversion steps.
- Delays premature simplification of lifecycle handling.
- Requires careful documentation of mode-specific invariants.

### Follow-up implications
This decision implies that later design work must define:
- which structures are shared across collection, processing, and ready,
- which structures change interpretation across modes,
- and whether any mode-specific helper structures are required.
