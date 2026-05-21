# ADR-004: Durable Blockchain Storage Persistence Threshold

### Status
Accepted

### Context
MoonBlokz uses staged validation. A block may be worth retaining long before all state-dependent checks, and even before its signature can be verified, because the public-key material needed for signature verification may belong to a node that becomes known only once more chain context has been reconstructed. At the same time, durable storage must reject content that is already provably unusable so that persistent capacity is not wasted on garbage.

The blockchain truth model distinguishes between authoritative durable state and derived helper state. Durable storage therefore needs a persistence threshold that is **permissive during collection and processing** but **tightens once ready-state evidence becomes available**.

### Decision
Persistent blockchain storage holds every received block **unless the module has exact evidence that the block cannot be accepted**. At the current design level, the intake-time exact-evidence forms are:

1. **Parsing failure** — the received byte stream does not conform to the structural rules of any known block payload type.
2. **Ready-state direct-active-chain-extension creator-signature invalidity** — the received block directly extends the current active-chain head (`previous_hash = active_chain_head_hash`), the creator's public key is derivable from the current active chain, and the block signature verifies as invalid against that key.
3. **Chain-config content-signature invalidity** — a chain-config block fails verification of its node-`#0` content-signature against `node_zero_public_key`.
4. **Configuration-module rejection of chain-config content** — the external configuration module rejects the chain-config content under the rules exposed through the blockchain boundary.
5. **Ready-state chain-config content mismatch** — a chain-config block's configuration content differs byte-for-byte from the durable-locked chain configuration.
6. **Deviation-branch creator-exclusivity violation** — a block above a deviation block on the deviation-bearing branch is created by a node other than the deviation block's creator.
7. **FR74 node-zero trust-anchor mismatches** — any artifact that asserts a node-`#0` public key different from `node_zero_public_key`, including:
   - a registration transaction with `new_node_id == 0` and `new_public_key ≠ node_zero_public_key`,
   - a balance block NodeInfo entry with `owner == 0` and `public_key ≠ node_zero_public_key`,
   - or chain-config signature material that fails the same trust-anchor rule.

When this evidence is not present, the block is stored. In particular, blocks whose parents are missing, whose signature cannot yet be verified because the creator's key is not yet known, or whose semantic validity depends on still-unavailable chain context, are all retained.

Outside the narrow ready-state direct-active-extension case above, creator-signature invalidity against the active-chain projection remains non-authoritative at intake time. Under the chain-switch reconciliation model with side-branch-distinct public-key projections, the active chain's per-node public-key projection is not the canonical reference for a signer that may become valid only on a divergent candidate branch. Authoritative creator-signature invalidity in those broader cases is therefore deferred to the blockchain module PRD FR9 Tier 3 chain-switch reconciliation path or the blockchain module PRD FR3 processing pass, where the candidate-side public-key projection serves as the canonical reference. The Tier 1 opportunistic signature-verification side effect described in blockchain module PRD FR9 (signature-cache priming against the active-chain projection at intake time outside the direct-active-extension exception) is informational only and does not trigger durable-storage rejection.

Parent/child indexes, branch-navigation aids, and similar helper structures may exist, but they are treated as derived implementation support rather than primary durable truth.

### Consequences
#### Positive
- Aligns with staged validation: storage does not wait for information the module may not have yet.
- Supports missing-parent, missing-creator-key, and partial-ancestry conditions without discarding potentially useful blocks.
- Rejects chain-config, trust-anchor, and deviation-branch violations at the earliest point where the current design treats them as exact evidence.
- Keeps durable truth narrow by expelling only content that is already provably wrong under the accepted PRD semantics.
- Avoids treating helper indexes as canonical blockchain state.
- Gives the collection phase a larger working set so dominant-chain acquisition can converge faster.

#### Trade-offs
- Storage may briefly contain blocks whose creator signatures later turn out to be invalid against the candidate-side public-key projection, requiring deferred rejection at blockchain module PRD FR9 Tier 3 chain-switch reconciliation or blockchain module PRD FR3 processing-pass time. The only intake-time creator-signature exception remains the ready-state direct-active-extension case above.
- Requires explicit handling for the transition from "stored but unverified" to "stored and signature-valid" to "active-chain selected".
- Puts stronger weight on the block-status progression model (see Algorithm 21 in the algorithm document) because the persisted set is wider than the fully verified set.
- Raises the importance of defining minimum persisted metadata, including which verification stages the block has already passed.

### Follow-up implications
This decision implies that later design work must define:
- the minimum metadata stored alongside persisted blocks, including verification-stage markers,
- the rebuild expectations for derived helper indexes,
- the deferred-rejection path that removes stored blocks proven invalid by blockchain module PRD FR9 Tier 3 chain-switch reconciliation or blockchain module PRD FR3 processing pass against the candidate-side public-key projection,
- the processing-phase deletion path that removes stored blocks proven invalid by full active-chain re-execution during the collecting → processing transition (signature invalidity against a now-known creator key, duplicate-transaction detection, negative-balance outcome, vote- or creator-rule violation, chain-config non-compliance, or `snake_chain` preservation failure), as a deferred-rejection mechanism that complements the blockchain module PRD FR9 Tier 3 chain-switch path noted above,
- and the state progression from persistable to fully semantically validated to active-chain selected.

The intake-time persistence threshold (the exact-evidence forms above) and the post-storage deletion paths (blockchain module PRD FR9 Tier 3 chain-switch reconciliation evidence and blockchain module PRD FR3 processing-pass re-execution evidence per blockchain module PRD FR5) are distinct lifecycle events. The intake threshold remains permissive; the deletion paths apply only after the corresponding candidate-side or processing-phase evidence has become available, with the candidate-side public-key projection (not the active-chain projection) serving as the canonical reference for deferred creator-signature-invalidity decisions.

The processing-phase deletion path is required for forward progress, not optional cleanup. If a block proven invalid by full active-chain re-execution were kept in durable storage, the dominant-chain acquisition rule would re-select the same candidate tip on the next collecting iteration, the re-execution would fail the same invariant at the same position, and the module would loop indefinitely between collecting and processing. Removing the proven-invalid block — or, when the specific offender cannot be identified, lowering the candidate chain's head by one block at a time until the offender is expelled — is the mechanism that breaks this livelock. The cost of losing a block that might in principle be valid on another branch is bounded because radio-layer re-dissemination can restore it; an unbreakable livelock would not be bounded.
