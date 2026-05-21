# ADR-002: Responsibility Boundary of `moonblokz-blockchain`

### Status
Accepted

### Context
The blockchain module sits next to several adjacent subsystem concerns: communication transport, fragmentation, storage, cryptography, the scoring module, local query serving, and chain-state management. Without a clear responsibility boundary, implementation would tend to pull transport and infrastructure concerns into the blockchain core.

MoonBlokz uses explicit subsystem boundaries across radio, storage, crypto, telemetry, and simulation. The blockchain module must follow the same discipline if it is to remain testable, portable, and conceptually clean.

### Decision
`moonblokz-blockchain` owns:
- blockchain decisions,
- staged validation logic,
- known-block and branch knowledge,
- active-chain selection,
- runtime mempool handling,
- vote / next-creator state,
- and local blockchain-facing query answers.

`moonblokz-blockchain` does **not** own:
- communication transport,
- fragment handling,
- serialization / deserialization adapters,
- storage implementation mechanics,
- crypto backend implementation,
- or the scoring module used as creator-selection input.

The blockchain module therefore remains a domain core with explicit surrounding adapters and dependencies, not an all-in-one blockchain engine.

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
