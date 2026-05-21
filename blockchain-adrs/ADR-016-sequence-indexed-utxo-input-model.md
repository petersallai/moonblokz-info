# ADR-016: Sequence-Indexed UTXO Input Model with Per-Block Spent-Bit Vector

### Status
Accepted

### Context
Part V originally described UTXO inputs as hash references (`tr_hash`, `output_index`) on the grounds that hash references survive branch changes more gracefully. That choice required either an in-memory live UTXO set keyed by transaction hash or a hash-indexed storage lookup structure. Both options conflict with the constrained embedded target (RP2040-class memory and flash): a live UTXO set would grow with usage and must be maintained continuously, and a hash-indexed auxiliary structure duplicates information already available through the sequence-indexed block storage.

The `snake_chain` window bounds the region where UTXOs can be referenced in the first place. Within that window, block sequence numbers are unique and already serve as the primary storage index. Using the sequence as the UTXO reference collapses the input lookup to a direct sequence-indexed block read plus one output-position access.

Spent-state tracking, which is the other half of the UTXO model, can then be represented as a per-block bit vector — one bit per output in that block — co-located with the block in storage. This removes the need for any live UTXO set as a continuously maintained projection.

### Decision
MoonBlokz shall reference UTXOs by `(block_sequence, output_index)` rather than by `(transaction_hash, output_index)`. The complex-transaction UTXO input field shall carry `block_sequence: u32` and `output_index: u8` (replacing the previous `tr_hash: [u8;32]`).

Spent-state for UTXO outputs shall be represented as a **per-block spent-bit vector** co-located with the block in storage. Each block carries one bit per UTXO output produced by its transactions; the bit is `0` while the output is unspent on the current active chain and `1` once spent.

The spent-bit vector is a **derived projection** of the current active chain, not durable blockchain truth. The authoritative record of spend events remains the transactions contained in accepted blocks. On chain switch, spent-bit vectors covering blocks inside the rollback scope shall be recomputed by forward-replaying the new active chain from the common ancestor, consistent with the chain-switch reconciliation workflow (ADR-011).

This decision supersedes the Part V rationale that hash-based references are preferred because they "survive branch changes". Under `snake_chain` bounds, the sequence reference is equally stable within its valid scope (the active chain inside the retained window), and carry-forward already invalidates key references in both models, so the hash-based stability argument does not provide a net advantage.

### Consequences
#### Positive
- Removes the ongoing live UTXO set from the derived-state projection; frees working-memory budget on the embedded target.
- UTXO input lookup becomes O(1) against the sequence-indexed block storage, without a hash-keyed auxiliary index.
- Carry-forward at tail-drop time is simplified: the unspent output set of the dropping block is read directly from its spent-bit vector, with no separate "find unspent" scan or auxiliary data structure.
- Shorter wire representation for UTXO inputs (roughly 32 bytes saved per input, since `u32` sequence replaces a 32-byte hash).
- Spent-bit vectors are compact (1 bit per output) and co-located with the block, matching flash-storage-friendly access patterns.

#### Trade-offs
- UTXO references are meaningful only on a specific active chain: the same `(sequence, output_index)` pair may refer to different outputs on different branches. The blockchain module must resolve the reference against the current active chain only.
- Chain-switch reconciliation must explicitly recompute the spent-bit vectors of blocks inside the rollback scope; this is a bounded, deterministic operation but must be an explicit step of the workflow.
- Signatures over UTXO input references now commit to a sequence rather than to a content-addressable hash. Transactions that reference a sequence whose content changes across a chain switch become invalid — this is the correct behavior (the referenced UTXO is no longer available), but implementers must understand the implication.
- Long-disconnect forks (per blockchain module PRD FR64) cause the sequence namespace to diverge between the two evolving chains; cross-fork UTXO reference is undefined. MVP does not attempt reconciliation, so this is acceptable.
- Carry-forward across the `snake_chain` window boundary produces a new `(sequence, output_index)` pair for the carried-forward UTXO; pre-carry-forward references do not continue to resolve. This matches the previous hash-based behavior, where carry-forward produced a new transaction hash.

### Follow-up implications
This decision implies that later design work and dependent artifacts must define or update:
- the binary serialization of the UTXO input field (Algorithm 25 and Section 6.1 of the algorithm document),
- the complex-transaction validity rules so that input existence and unspent status are resolved against the sequence-indexed block storage and the per-block spent-bit vector (Algorithm 13),
- the carry-forward algorithm so that the unspent output set is read from the spent-bit vector of the dropping block (Algorithm 12),
- the chain-switch reconciliation workflow so that spent-bit vectors within the rollback scope are recomputed as part of the forward-replay step (ADR-011 and the derived-state projection definitions),
- the implementation-level representation of derived state: the "Live UTXO state" section of the implementation document shall be reframed as a per-block spent-bit vector projection.

This ADR does not change the carry-forward policy itself (ADR-013): custodian-fee reduction, compression into consecutive blocks, and below-fee discard all continue to apply. It changes only the representation of UTXO identity and spent state.
