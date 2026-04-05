# ADR-001: `moonblokz-blockchain` Is a Semantic Event State Machine

### Status
Accepted

### Context
`moonblokz-blockchain` is the central stateful blockchain core of MoonBlokz. It must operate under unreliable radio communication, bounded storage, staged validation, and strict resource limits, while still serving as the source of blockchain-facing truth for higher-level local consumers.

A key architecture question was whether the module should be modeled around transport-facing message handling or around higher-level semantic blockchain events.

MoonBlokz already treats communication transport, fragmentation, storage mechanics, and cryptographic implementation as separate subsystem concerns. The blockchain core therefore needs an identity that keeps protocol meaning separate from those outer mechanics.

### Decision
`moonblokz-blockchain` is defined as a **stateful semantic event state machine**.

It:
- consumes blockchain-relevant semantic events,
- maintains blockchain-relevant internal state,
- produces semantic decisions or responses,
- and serves read-only local blockchain-facing queries.

Its inputs are semantic, not transport-level byte streams. Its outputs are blockchain decisions and query answers, not communication-side delivery actions.

The module is therefore **not** defined as a transport-aware byte-processing component.

### Consequences
#### Positive
- Keeps the blockchain core focused on blockchain domain logic.
- Cleanly separates protocol meaning from transport representation.
- Improves testability by allowing semantic inputs to drive the module directly.
- Fits staged validation and partial-knowledge operation naturally.
- Supports deterministic reasoning about state changes.

#### Trade-offs
- Requires explicit adapter layers around the blockchain core.
- Requires a well-defined semantic event vocabulary.
- Pushes some integration complexity to surrounding modules.
- Makes boundary contracts more important and more visible.

### Follow-up implications
This decision implies that later ADRs must define:
- the responsibility boundary of the blockchain module,
- the authoritative versus derived state model,
- and the semantic contracts used between blockchain and surrounding modules.
