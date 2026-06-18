# MoonBlokz Glossary (Cross-Subsystem & Ambiguous Terms)

This glossary disambiguates terms that span more than one subsystem or that carry more than one meaning across the knowledge base. It is a navigation aid and is **not authoritative**: each entry links to the authoritative definition site, which wins on any divergence. Terms that are unambiguous within a single subsystem are intentionally omitted; see the [index](./moonblokz-index.md) for per-document scope.

| Term | Meaning in context | Authoritative definition site |
|---|---|---|
| **node** | An active participant in the blockchain (the consensus population); distinct from an address. | [Blockchain Concept](./moonblokz-blockchain-concept.md) — Network Entities |
| **address** | A passive endpoint for holding and transferring value, represented as UTXOs; distinct from a node (users can create many addresses). | [Blockchain Concept](./moonblokz-blockchain-concept.md) — Network Entities |
| **score — relay** | Radio metric for how much a node can improve message reach; drives the relay decision. | [Radio Algorithm](./moonblokz-radio-algorythm.md) §H4 |
| **score — vote target** | Radio-side per-node score computed from the senders of observed messages; selects the vote target embedded in a transaction before it reaches the blockchain. | [ADR-007](./blockchain-adrs/ADR-007-vote-module-consumes-scoring-module-input.md) |
| **score — accumulated vote** | Blockchain vote-module per-node vote accumulation that names the next expected block creator (`score += floor(score × vote_interest / vote_scale)`). | [ADR-007](./blockchain-adrs/ADR-007-vote-module-consumes-scoring-module-input.md), [Blockchain Algorithm](./moonblokz-blockchain-algorythm.md) |
| **active-chain window** (`SNAKE_CHAIN_LENGTH`) | The bounded-retention window of the active chain. | [Blockchain Architecture](./moonblokz-blockchain-architecture.md) §5 |
| **verification horizon** (`VERIFICATION_HORIZON`) | Retained knowledge beyond the active-chain window so that rare active-chain switches stay verifiable; an explicit concern distinct from the active-chain window. | [ADR-012](./blockchain-adrs/ADR-012-verification-horizon-must-be-explicit-under-bounded-retention.md), [BC-FR58](./moonblokz-blockchain-prd.md) |
