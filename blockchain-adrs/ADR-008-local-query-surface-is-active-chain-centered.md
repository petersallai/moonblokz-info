# ADR-008: Local Query Surface Is Active-Chain-Centered

### Status
Accepted

### Context
The blockchain module needs a local-facing query surface that higher-level payment or application logic can use. Internally, MoonBlokz may retain branches, staged validation state, and other non-final blockchain knowledge. Exposing all of that by default would make the local interface more complex and less useful as a product-facing truth surface.

The broader blockchain boundary already separates internal chain complexity from the simpler local-facing operational truth that consumers need.

### Decision
The local query surface of `moonblokz-blockchain` is **active-chain-centered** by default.

This means:
- block queries resolve only against the active chain,
- transaction status distinguishes at least between unknown, in mempool, and in active chain,
- transaction answers in active chain also expose sequence depth,
- balance queries support a simple current answer and may optionally expose active-chain depth or context.

The default local surface is not a full branch-observability API.

### Consequences
#### Positive
- Keeps the local interface simple and payment-facing.
- Provides useful operational truth without leaking all internal branch complexity.
- Aligns with the intended product boundary of the blockchain module.
- Makes local consumers less dependent on internal branch and staged-validation details.

#### Trade-offs
- May require separate diagnostics or debugging interfaces later.
- Hides some internal ambiguity by default.
- Requires careful definition of depth and confidence semantics.
- May create pressure to maintain two different local-facing views: product-facing and diagnostic.

### Follow-up implications
This decision implies that later design work must define:
- the exact query payloads and meanings of active-chain depth values,
- the distinction between product-facing queries and deeper diagnostic visibility,
- and the query contracts for transaction, balance, and block lookups.
