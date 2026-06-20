# ADR-008: Local Query Surface Is Active-Chain-Centered

### Status
Accepted

### Context
The blockchain module needs a local-facing query surface that higher-level payment or application logic can use. Internally, MoonBlokz may retain branches, staged validation state, and other non-final blockchain knowledge. Exposing all of that by default would make the local interface more complex and less useful as a product-facing truth surface.

The broader blockchain boundary already separates internal chain complexity from the simpler local-facing operational truth that consumers need.

### Decision
The local query surface of `moonblokz-blockchain` is **active-chain-centered** by default: queries resolve only against the active chain, expose simple `Unknown / In-mempool / Confirmed` statuses, optional active-chain depth, and balance / UTXO views — they are not a full branch-observability API.

The 12 read-only public methods that realize this surface (and their depth semantics) are normative in [`moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) §3.1 and `queries.rs` (§4.2); the per-query outcome contracts are normative in [PRD](../moonblokz-blockchain-prd.md) FR40 (transaction state), FR41 (balance / UTXO), FR42 (block retrieval), FR43 (top-mempool exchange).

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
