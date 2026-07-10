# ADR-014: Mode Transitions May Require Representation-Aware Design

### Status
Accepted

### Context
MoonBlokz blockchain behavior spans two operating phases — collecting and ready — joined by a validating processing transition (per [PRD](../moonblokz-blockchain-prd.md) FR1). A key architecture question is whether these lifecycle points can share one dominant internal representation or whether some of them require distinct helper structures or transitional representations.

The blockchain lifecycle is not only a naming distinction. Different modes may place different demands on chain knowledge, validation maturity, branch bookkeeping, and derived-state support.

### Decision
Mode transitions are treated as **representation-aware design points**: it is not safe to assume that the collecting and ready phases and the processing transition between them share one unchanged internal structure with only a flag difference. The architecture must explicitly evaluate which data structures are shared across these lifecycle points, which change meaning across them, and which require phase- or transition-specific support.

The three-type Owned + View + Builder model for `Block` / `Transaction` (per FR61) that realizes this representation-awareness is normative in [`moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) §3.2; the `EmitScratch` outcome view source is normative in §6.6.

### Consequences
#### Positive
- Prevents underestimating lifecycle complexity.
- Helps avoid accidental misuse of structures across modes.
- Keeps lifecycle-driven design explicit.
- Encourages deliberate treatment of collecting-phase, processing-transition, and ready-phase behavior.

#### Trade-offs
- May increase implementation complexity.
- May require transitional representations or conversion steps.
- Delays premature simplification of lifecycle handling.
- Requires careful documentation of mode-specific invariants.

### Follow-up implications
This decision implies that later design work must define:
- which structures are shared across the collecting and ready phases and the processing transition,
- which structures change interpretation across modes,
- and whether any mode-specific helper structures are required.
