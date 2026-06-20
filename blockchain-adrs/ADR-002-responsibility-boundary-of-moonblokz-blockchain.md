# ADR-002: Responsibility Boundary of `moonblokz-blockchain`

### Status
Accepted

### Context
The blockchain module sits next to several adjacent subsystem concerns: communication transport, fragmentation, storage, cryptography, the scoring module, local query serving, and chain-state management. Without a clear responsibility boundary, implementation would tend to pull transport and infrastructure concerns into the blockchain core.

MoonBlokz uses explicit subsystem boundaries across radio, storage, crypto, telemetry, and simulation. The blockchain module must follow the same discipline if it is to remain testable, portable, and conceptually clean.

### Decision
`moonblokz-blockchain` owns blockchain decisions, staged validation, known-block and branch knowledge, active-chain selection, runtime mempool handling, vote / next-creator state, and local blockchain-facing query answers. It does **not** own communication transport, fragment handling, serialization adapters, storage implementation, crypto backend, or the radio-side scoring module.

The exact crate boundaries, sub-crate split, and API surfaces are normative in [`moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) §1–§3, and the boundary contract / public-API-only consumption rules are normative in [PRD](../moonblokz-blockchain-prd.md) FR61 and FR66.

### Consequences
#### Positive
- Preserves a clean domain boundary.
- Prevents transport and persistence details from contaminating blockchain logic.
- Makes the module easier to reason about, test, and evolve.
- Aligns the blockchain subsystem with the broader MoonBlokz architectural style.
- Helps keep the blockchain core portable across different environments.

#### Trade-offs
- Pushes complexity into boundary adapters and contracts.
- Requires explicit interfaces for score input, block persistence, and local queries.
- May feel less convenient than a monolithic implementation in the short term.
- Makes integration discipline mandatory rather than optional.

### Follow-up implications
This decision implies that later ADRs or design notes must define:
- the shape of semantic inputs and outputs around the blockchain core,
- the authoritative-versus-derived state model,
- and the concrete contracts between blockchain and radio, storage, and local-facing layers.
