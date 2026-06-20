# ADR-013: Bounded UTXO Retention Requires an Explicit Preservation Strategy

### Status
Accepted

### Context
MoonBlokz combines bounded blockchain history with UTXO-based address privacy. This creates direct pressure between preserving live address-state and keeping the chain usable for normal operation. Without an explicit policy, UTXO carry-forward could become an uncontrolled implementation detail that silently consumes useful chain capacity.

The brainstorming session identified bounded UTXO handling as one of the most dangerous architecture points in the blockchain design.

### Decision
MoonBlokz requires an explicit **bounded UTXO preservation strategy** rather than treating UTXO carry-forward as an incidental implementation detail: live UTXOs in a dropping block are identified from that block's spent-bit vector, the still-unspent outputs are replayed forward through a zero-input carry-forward transaction, each preservation step deducts a fixed chain-config custodian fee, and any output whose remaining amount would fall below the fee is discarded. Repeated carry-forward therefore erodes retained carry-forward-only residue until it clears naturally.

The carry-forward construction rules, the below-fee-discard rule, and the explicit no-saturation-handling stance are normative in [PRD](../moonblokz-blockchain-prd.md) FR51 (carry-forward) and FR52 (no UTXO-saturation handling); the spent-bit-vector representation that underpins identification is normative in [`moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) §6.2 and `spent_bits.rs` (§4.2), and uses the input model of [ADR-016](./ADR-016-sequence-indexed-utxo-input-model.md).

#### Fixed custodian fee

The custodian fee is a **fixed chain-config-derived value** with no dynamic inputs. The configuration module's custodian-fee accessor returns the fixed value derived from chain-config content alone; the blockchain module may invoke it whenever a custodian-fee value is needed (per carry-forward construction) and is not required to cache the result. The long-run pressure-relief mechanism therefore relies on the steady deduction of this fixed amount from every surviving carried-forward UTXO at each FR53 carry-forward step until the UTXO's pre-fee amount falls below the custodian fee and the UTXO is discarded under FR53's below-fee rule.

#### No explicit UTXO saturation handling

When the active-window's unspent UTXOs combine with `snake_chain` replay obligations to consume most or all of `MAX_BLOCK_SIZE` over many consecutive head blocks, the blockchain module performs no special saturation detection, status reporting, structured log emission, mempool admission backpressure, or replay-deferral mechanism. The chain continues ordinary block creation per the standard content-assembly priority, producing a sequence of replay-bearing blocks that may admit no ordinary mempool transactions. The mempool continues to accept new transactions throughout — including complex transactions that would produce additional UTXO outputs — and any such admitted transactions remain in the mempool until block capacity becomes available again. The condition self-clears across many tail-advance cycles because the fixed custodian fee reduces every surviving carried-forward UTXO at each carry-forward step, eventually discarding UTXOs whose pre-fee amount falls below the custodian fee, until block capacity for ordinary mempool transactions is restored.

Source: this ADR is currently accepted with a fixed-value custodian fee per [PRD](../moonblokz-blockchain-prd.md) FR51 / FR52. An earlier dynamic-input formulation (with the active-window UTXO count and active-window node-transfer count as accessor inputs) and a corresponding explicit UTXO saturation stall handling were briefly considered but have been reverted; the current PRD pins the fixed-fee model and removes the saturation-detection FR.

### Consequences
#### Positive
- Treats one of the core bounded-history tensions explicitly.
- Makes the long-run pressure-relief mechanism explicit: custodian-fee erosion gradually removes old preserved UTXOs.
- Keeps UTXO handling aligned with `snake_chain` realities.
- Makes the privacy-versus-throughput trade-off visible rather than hidden.

#### Trade-offs
- Long-lived small UTXOs are gradually consumed by preservation fees.
- The cleanup rate depends on both the custodian-fee setting and the size distribution of preserved UTXOs.
- Transient packing pressure can still occur even though persistent carry-forward residue is self-clearing.

### Follow-up implications
This decision implies that implementation work must define:
- the exact deterministic carry-forward construction,
- the per-step custodian-fee application logic,
- the chain-config-derived fixed custodian-fee value,
- and any modeling needed to choose practical custodian-fee parameters and the carry-forward lookahead `n` for a deployment.

The UTXO identity representation underpinning this strategy is defined separately in [ADR-016](./ADR-016-sequence-indexed-utxo-input-model.md): UTXO inputs use `(block_sequence, output_index)` references and spent state is tracked through per-block spent-bit vectors. Carry-forward identification at tail-drop time therefore reduces to reading the dropping block's spent-bit vector; the self-clearing custodian-fee rule of this ADR operates on top of that representation.
