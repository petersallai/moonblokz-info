# ADR-014: Mode Transitions May Require Representation-Aware Design

### Status
Accepted

### Context
MoonBlokz blockchain behavior includes collection, processing, and ready-like operational phases. A key architecture question is whether these modes can share one dominant internal representation or whether some of them require distinct helper structures or transitional representations.

The blockchain lifecycle is not only a naming distinction. Different modes may place different demands on chain knowledge, validation maturity, branch bookkeeping, and derived-state support.

### Decision
Mode transitions are treated as **representation-aware design points**. It is not safe to assume up front that all operational modes can share one unchanged internal structure with only a flag difference.

The architecture must explicitly evaluate which data structures:
- are shared across modes,
- change meaning across modes,
- or require mode-specific support.

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
