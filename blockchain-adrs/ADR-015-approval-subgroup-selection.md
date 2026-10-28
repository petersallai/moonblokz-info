# ADR-015: Approval Subgroup Selection

### Status
Accepted

### Context
The MoonBlokz approval process legitimizes deviations from the default creator-selection result (Algorithm 6 in `moonblokz-blockchain-algorythm.md`). It requires a configurable minimum number of support signatures, carried as aggregated evidence inside an approval evidence block.

Two related details had remained deferred in the article series and were called out as open gaps in `moonblokz-blockchain-implementation.md`:

- how the subgroup of nodes whose support signatures count toward an approval is chosen,
- and how the set of "active" nodes referenced by that selection is defined.

When `required_support` is at least majority of the whole active network, a subgroup concept is not needed: the approval naturally involves the whole active set. When `required_support` is below that majority, the subgroup must be chosen in a way that is deterministic across all honest nodes, independent from any single node's control, resistant to grinding attacks, and implementable under MoonBlokz's bounded embedded constraints.

### Decision

Approval subgroup selection uses a deterministic, seed-derived rendezvous-hash ordering over a well-defined active-node set, with a subgroup size chosen so that `required_support` is exactly a majority inside the subgroup.

#### Active-node set

A node is **active** at an evaluation point if, within that point's active window, it created at least one block (of any payload type) or is a signer listed in an approval evidence block or a twin verdict bitmap. Transaction initiation alone does not count (Project Lead, 2026-10-28; DEC-185): sleeping senders cannot sign and so cost approval liveness (AR-008), and an identity could join `A` for one transaction fee. The evaluation point is the deviation's parent `p − 1` for an approval and the twins' shared parent `P` for a verdict, so every node derives the same set (O-030 closed). With `vote_interest = 50` creation rotates, so for `N ≤ W` every voted node creates within the window and `A` covers the network; for `N > W` it is a sample of about `W` creators in which a coalition keeps about its population share.

The active set `A` is therefore derivable from active-chain state per ADR-003 and ADR-008, bounded by the `snake_chain` length, and does not require separately persisted membership state.

A node under a BC-FR72 equivocation exclusion is not in `A` while the exclusion lasts (Project Lead, 2026-10-22; DEC-165), nor is a node under a late-node exclusion (BC-FR37, DEC-184, DEC-185).

The subgroup is drawn from the **voting set** `A'` = `A` without the deviation's proposer and its originally-top node, the two parties to the decision; when `A'` is empty no approval can form (Project Lead, 2026-10-27; DEC-172).

#### Seed

```
seed = H( "MBSEED"  ‖  proposed_sequence  ‖  circulating_currency(p − 1)  ‖  proposer_node_id )
```

- `circulating_currency(p − 1)` is the money on the chain after block `p − 1` — every balance plus every unspent UTXO amount, a `u64` — the Story 7.6 projection, maintained per block and identical on every node holding `p − 1` (Project Lead, 2026-10-28; DEC-192).
- It replaces the tail block's hash, which its creator could vary freely (transaction order, comments) and so grind for a subgroup it controls (AR-013). Within a block the total moves only through deterministic minting and burning and through registrations, whose price is burned; a creator can vary the seed only by including or leaving out pending registrations, usually one or two variants, and each sybil registration it adds for more variants costs the full registration price.
- The seed is known only once `p − 1` is fixed, where the tail hash was known `W` blocks ahead.
- `proposed_sequence` keeps proposals at different positions apart; `proposer_node_id` binds the seed to the proposer; the domain tag separates it from the twin verdict seed.
- On a window-anchored FR3 candidate the total is not served before the pass has covered the window, so subgroup checks of evidence blocks whose evaluation point lies in the segment's first `W` positions are trusted there, like other pre-window state (DEC-107).

#### Subgroup selection

For every active node `n`, compute a deterministic 64-bit ordering key:

```
node_value(n) = ( H(seed ‖ node_id(n))  mod 2³² ) · 2³²  +  node_id(n)
```

- `node_id` is a fixed 32-bit value, so the full `node_value` fits in 64 bits with a deterministic tiebreaker.
- Upper-32-bit collisions are statistically negligible at expected active-set sizes and are broken deterministically by the 32-bit `node_id`.

The subgroup is the `m` nodes of `A'` with the smallest `node_value`. Growing `m` only adds members (monotonicity), which preserves earlier selections if a wider subgroup is ever needed.

#### Subgroup size

```
r_eff = min( required_support ,  ⌊|A'| / 2⌋ + 1 )
m     = min( 2 · r_eff − 1 ,  |A'| )
```

- `r_eff` is a strict majority of the subgroup at every size, so two quorums for conflicting outcomes always share a member.
- In a large network `r_eff = required_support`, which is therefore a cap; in a shrunken network the subgroup becomes the whole voting set and the approval needs its simple majority. Before 2026-10-27 the threshold stayed `required_support` while `m` was capped at `|A|`, so no approval could form when `|A| < required_support`, and between `required_support` and `2·required_support − 1` it was a supermajority (DEC-172).

#### Chain-config invariant

Only the supporters that actually sign are written into the approval evidence, so the crypto aggregation ceiling constrains `required_support`, not `m`:

```
required_support ≤ MAX_AGGREGATED_SIGNATURES(backend)
```

`m` therefore has no direct crypto cap; it is bounded only through `required_support` and through `|A|`. `MAX_AGGREGATED_SIGNATURES` is backend-dependent (see the crypto documents).

#### Fallback when an approval cannot collect enough support

If an approval cannot collect `r_eff` signatures in time, the subgroup is **not** widened. Instead, the existing grace-period expansion (Algorithm 5) admits the next-ranked creator candidate, which starts a new approval with its own freshly derived subgroup. Approval fallback therefore stays aligned with the existing grace-period mechanics and does not duplicate them.

### Consequences

#### Positive
- Deterministic and verifiable from active-chain state alone, without persisted membership or external randomness.
- No grindable inputs: neither the proposer nor any single past creator can bias the subgroup.
- Self-capping for small networks: `m = |A'|` and a simple-majority threshold when the voting set is below the normal subgroup size.
- Monotonic in `m`, which keeps this ADR compatible with any future mechanism that needs to widen a deterministic ordering.
- Aligns with ADR-003 (authoritative vs. derived state), ADR-005 (Chain Knowledge Core orchestration), PRD FR55 (vote module scope vs. external `scoring_module`), and ADR-008 (active-chain-centered local queries).
- Aligns with the bounded-aggregation principle from `moonblokz-crypto-concept.md` by treating the crypto ceiling as a hard chain-config bound.

#### Trade-offs
- The "active node" definition is per-branch, so in rare multi-branch situations each branch computes its own subgroup. This is consistent with the active-chain-centered model but must be remembered when reasoning about reorgs.
- Chain-config validation must know the active crypto backend in order to reject `required_support` values that exceed that backend's `MAX_AGGREGATED_SIGNATURES`.
- A newly registered node becomes "active" as soon as its first transaction enters the active chain; this is intentional and interacts with the existing registration economics (Algorithm 14) rather than with the subgroup rule itself.

### Alternatives considered

- **Modulo-class partitioning on a seed**: cheap, but does not give a precise `m`, so the "50%+1 inside the subgroup" invariant is hard to preserve.
- **Threshold-based sortition (hash < target)**: elegant and Algorand-style, but produces a probabilistic subgroup size that conflicts with the deterministic `m`-from-`required_support` derivation.
- **Seed = H(past\_anchor ‖ proposer ‖ proposed\_block\_hash)**: rejected because the proposer can grind the proposed block's content (transaction ordering, optional fields) to bias the resulting subgroup. Including the proposed block hash in the seed opens a grind surface rather than strengthening the commitment.
- **Late-binding signature-modulo membership**: deciding subgroup membership from `support_message_signature mod N` would conceal subgroup identity until support messages arrive, raising the cost of selective targeting (jamming, isolation, DoS) against members whose identities are otherwise predictable as soon as the proposer is known. Rejected because the criterion is verifiable from each individual support message but is not reconstructible from the BLS-aggregated evidence block, since per-supporter signatures are absorbed by aggregation; only Schnorr `MultiSignature` evidence (which retains per-supporter signatures) preserves the audit path, at the cost of forfeiting the BLS profile's compact-evidence advantage. The scheme also yields a probabilistic `m`, sharing the precision concern of the modulo-class partitioning and threshold-based sortition variants above.
- **In-subgroup expansion fallback (widen `m` when signatures are insufficient)**: rejected because MoonBlokz already has a grace-period expansion mechanism. Duplicating it inside the subgroup concept would blur two responsibilities.

### Follow-up implications

This decision implies that later design work must define:

- the exact binary serialization of the approval evidence payload (this ADR fixes the selection rule but not the byte layout),
- the chain-config validator rule that enforces `required_support ≤ MAX_AGGREGATED_SIGNATURES(backend)`,
- and the interaction between the subgroup rule and the detailed grace-period progression, particularly for back-to-back approvals under `snake_chain` precedence (Algorithm 18).

The BC-FR71 twin verdict reuses this selection with a domain-separated seed, `H("twin" ‖ s ‖ circulating_currency(P) ‖ c)` (DEC-192), over the active set at the twins' shared parent `P`, without the equivocating creator `c` (DEC-164). Because its signers are a bitmap over the subgroup positions, the active set must be bit-identical on every node; the evaluation point and membership rule above make it so (DEC-185).
