# MoonBlokz Open Gaps Register

## Role of This Document

This file is a cross-cutting register for unresolved gaps, verified documentation drift, and explicitly post-MVP design questions that affect MoonBlokz planning, implementation, deployment, or operations.

It is a consolidation and navigation layer, not an authoritative source. Each entry links to the knowledge-base document where the underlying fact, limitation, discrepancy, or open question is defined. If this register diverges from a linked source, the linked source wins and the divergence must be resolved under the source-fidelity rules in [`AGENTS.md`](./AGENTS.md).

## Source Basis

This register currently consolidates gaps already documented in:

- [`moonblokz-system-constraints.md`](./moonblokz-system-constraints.md) — numeric, capacity, airtime, flash, and storage-geometry gaps.
- [`moonblokz-telemetry-implementation.md`](./moonblokz-telemetry-implementation.md) — telemetry deployment-model drift, docs-vs-code drift, and implementation sharp edges.
- [`blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md`](./blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md) — accepted bounded-UTXO preservation strategy and explicit no-saturation-handling stance.
- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) — post-MVP dynamic custodian-fee, long-disconnect recovery, and concurrent-ingestion processing-state concepts.
- [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md), [`moonblokz-blockchain-operating-model.md`](./moonblokz-blockchain-operating-model.md), and [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) — blockchain specification gaps and wording drift (approval-evidence encoding, UTXO-input rules, support-message signature, grace-period timing, fee policy, carry-forward discard).

## Register

| ID | Area | Type | Current state | Impact | Primary source |
|---|---|---|---|---|---|
| OG-001 | Radio / deployment capacity | Gap | No regional LoRa duty-cycle limit is documented in the knowledge base. | Real-deployment airtime and capacity planning cannot be grounded in KB values. | [`moonblokz-system-constraints.md` §5/§7](./moonblokz-system-constraints.md#7-known-discrepancies--gaps) |
| OG-002 | Storage / flash durability | Gap | Flash wear-lifetime is not quantified. | Long-term durability budgeting remains undefined. | [`moonblokz-system-constraints.md` §4/§7](./moonblokz-system-constraints.md#7-known-discrepancies--gaps) |
| OG-003 | Storage geometry | Verify | A general slot-count formula divides by `MAX_BLOCK_SIZE` (2016), while the RP2040 placement contract uses `SLOT_SIZE_BYTES` (2048, including the per-slot hash). | General capacity estimates may slightly over-count slots unless the intended divisor is confirmed. | [`moonblokz-system-constraints.md` §7 G3](./moonblokz-system-constraints.md#7-known-discrepancies--gaps) |
| OG-004 | Telemetry / operations | Drift / sharp edges | Probe, HUB, Collector, CLI, and Update Server docs and deployment assumptions partially diverge from reviewed code. | Interoperability, operator expectations, update activation, command behavior, and log-collection reliability can be misunderstood if docs are treated as current without checking code-grounded notes. | [`moonblokz-telemetry-implementation.md` §Current Deployment-Model and Documentation Drift](./moonblokz-telemetry-implementation.md#current-deployment-model-and-documentation-drift), [`§Current Code-Grounded Inconsistencies and Sharp Edges`](./moonblokz-telemetry-implementation.md#current-code-grounded-inconsistencies-and-sharp-edges) |
| OG-005 | Blockchain / bounded UTXO retention | Accepted MVP limitation + post-MVP concept | The MVP uses a fixed chain-config-derived custodian fee and explicitly performs no special UTXO-saturation detection, status reporting, structured logging, mempool backpressure, or replay deferral. A dynamic saturation-aware custodian fee is only a post-MVP concept. | Under high live-UTXO accumulation, replay obligations can consume block capacity and ordinary mempool transactions may wait until fixed-fee erosion self-clears capacity. | [`ADR-013`](./blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md#no-explicit-utxo-saturation-handling), [`moonblokz-blockchain-concept.md` §Dynamic Custodian Fee](./moonblokz-blockchain-concept.md#dynamic-custodian-fee-post-mvp-concept) |
| OG-006 | Blockchain / long-disconnect recovery | Post-MVP concept | Long-disconnect resynchronization is out of MVP scope: the current model logs the condition and relies on external operator intervention. A chain-internal watcher/subgroup/anchor recovery path is conceptual only. | Deployments that expect long but recoverable disconnections need future requirement and architecture work before relying on chain-internal recovery. | [`moonblokz-blockchain-concept.md` §Long-Disconnect Recovery](./moonblokz-blockchain-concept.md#long-disconnect-recovery-post-mvp-concept) |
| OG-007 | Blockchain / processing lifecycle state | Post-MVP concept | In the MVP, processing is a transient, non-resumable reconstruct-and-validate transition (BC-FR3 / BC-FR59), not a resting state that ingests blocks. A concept promotes it to a first-class operating state that keeps accepting blocks while reconstruction runs, making the lifecycle a three-state model (collecting, processing, ready). | On constrained hardware a slow processing pass extends the window during which the node cannot advance; without concurrent ingestion the node struggles to keep up with arriving blocks during that window. Realizing the concept requires reconciling the non-resumability contract, FR5 atomic-recovery scope, and the two-phase lifecycle framing. | [`moonblokz-blockchain-concept.md` §Processing as a Concurrent-Ingestion State](./moonblokz-blockchain-concept.md#processing-as-a-concurrent-ingestion-state-post-mvp-concept) |
| OG-008 | Blockchain / approval evidence | Gap | The approval-evidence payload content is fixed (`supporter_vote_sum`, aggregated signature, supporter-identity list per BC-FR27), but no source fixes its byte encoding: the identity-list count field, the per-backend aggregated-signature length, or how the evidence binds to the approved deviation block. | Evidence blocks cannot be encoded or validated interoperably until the wire layout is specified. | [`moonblokz-blockchain-algorythm.md` §9](./moonblokz-blockchain-algorythm.md#9-approval-payload), [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) BC-FR27 |
| OG-010 | Blockchain / sequence space | Post-MVP concept | In the MVP the `u32` block sequence never wraps: `u32::MAX` is always rejected and block creation stops at `u32::MAX − 1` (BC-FR53). Wrap-around handling (modular sequence space, content-matched genesis, ancestry-based comparisons) is a deferred planning direction only. | No practical MVP impact (about 4.29 billion blocks); a chain expected to outlive the sequence space needs requirement and architecture work first. | [`moonblokz-blockchain-algorythm.md` §Sequence Wrap-Around](./moonblokz-blockchain-algorythm.md#sequence-wrap-around), [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) BC-FR53 |
| OG-011 | Blockchain / UTXO causality | Gap | No PRD or algorithm rule states that a UTXO input's referenced block must be a strict ancestor (`block_sequence` below the consuming block's) or forbids spending an output created in the same block; Operating Model I-27's "retained ancestor" only implies it. The code enforces both (Tier 1 rejects `block_sequence ≥ sequence`; same-block chaining stays forbidden until Story 7.1). | The rule exists only in code and story records until it is written into BC-FR6 / BC-FR9. | [`moonblokz-blockchain-operating-model.md` I-27](./moonblokz-blockchain-operating-model.md), [`moonblokz-blockchain-prd.md` FR6, FR9](./moonblokz-blockchain-prd.md) |
| OG-012 | Blockchain / UTXO window bound | Drift | The bound on a UTXO reference is worded three ways: "retained block-tree" (BC-FR9 Tier 2), "retained window" (BC-FR6), and `block_sequence ≥ S_tail` (BC-FR14). They agree in effect once the tail drop unlinks the released block. | Readers can take the wordings for three different rules. | [`moonblokz-blockchain-prd.md` FR6, FR9, FR14](./moonblokz-blockchain-prd.md) |
| OG-013 | Blockchain / UTXO-input signature | Gap | "Signature over this input and all outputs" does not say which transaction header, tag, or count bytes the preimage covers, or how the signature field itself is treated. No code produces or verifies it yet (Story 7.1). | UTXO-input signatures cannot be produced or verified interoperably until the preimage is specified. | [`moonblokz-blockchain-algorythm.md` §6.1](./moonblokz-blockchain-algorythm.md#61-utxo-input-input-type--0) |
| OG-014 | Blockchain / support-message signature | Drift | BC-FR12 defines the support signature over `(proposer, deviation sequence, deviating block hash)`; BC-FR26 names the field `signature_over_deviation_block_sequence_by_supporter` and its check (ii) verifies "the supporter's signature over the `deviation_block_sequence` value". | Under the BC-FR26 reading a support signature would not bind to a specific proposer or deviating block; the signed content must be fixed before support messages and evidence can be produced and verified interoperably. | [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) BC-FR12, BC-FR26 |
| OG-015 | Blockchain / grace-period timing | Gap | No source states which event starts the grace-period window (the Operating Model marks the grace tick's trigger "Not specified"), and invariant I-10 lists "an admitted grace-period fallback" as a Tier 3 / processing-pass check without saying how a receiver decides grace admission for a block when no time is chain-verifiable. | Fallback admission (BC-FR44/BC-FR47), BC-FR12 support eligibility and I-10 validation cannot be implemented interoperably. | [`moonblokz-blockchain-operating-model.md`](./moonblokz-blockchain-operating-model.md) Tick — grace period, I-10; [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) BC-FR44, BC-FR47 |
| OG-016 | Blockchain / fee policy | Gap | BC-FR6 and BC-FR9 Tier 2 require compliance with the per-byte fee policy (`tx_fee_per_byte_min` / `tx_fee_per_byte_max`, configuration IDs 25 / 26), but no source states the validity predicate: which serialized size is priced, whether the bounds are inclusive, and which fee value applies to registrations and complex transactions. Owned by Story 7.3. | Fee validity is not decidable interoperably until the predicate is written. | [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) BC-FR6, BC-FR9; [`moonblokz-configuration-specification.md`](./moonblokz-configuration-specification.md) §4.1 |
| OG-018 | Blockchain / carry-forward discard | Drift | The below-fee discard is worded three ways: Algorithm 12 discards a UTXO "whose amount becomes smaller than the custodian fee" (after reduction), the Algorithm Model's UTXO-saturation paragraph speaks of a "pre-fee amount" below the fee, and BC-FR51 (iv) omits an output "whose amount falls below the custodian fee". None says whether an output equal to the fee is carried as a zero-amount output. | Creator and validator could derive different carry-forward output sets. | [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) Algorithm 12, §UTXO saturation stall; [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) BC-FR51 |
| OG-019 | Blockchain / grace-period default | Verify | `grace_period_window_ms` defaults to 30 s, below the measured unsigned-block propagation median (33 s) and well below the verification-inclusive estimate of about 1–2.3 min per block (RP2040 timings, DEC-180). | A grace this short expires before most nodes hold the top creator's block and starts needless deviations; to be sized with the deviation-approval reform (OG-015, AR-021). | [`moonblokz-system-constraints.md` §2.1 / §7 G4](./moonblokz-system-constraints.md#7-known-discrepancies--gaps) |

## Detail Notes

### OG-001 — Regional LoRa duty-cycle limit

The constraints reference records LoRa packet and airtime assumptions but states that no regional duty-cycle number is documented anywhere. The open item is not a disagreement between documents; it is missing deployment-specific constraint data.

### OG-002 — Flash wear-lifetime

The constraints reference records the RP2040 flash geometry and states that flash wear-lifetime is not quantified. The storage algorithm is the linked source for the storage behavior; no durability budget is currently established in the knowledge base.

### OG-003 — Storage slot-count divisor

The constraints reference marks this as `Verify`: the general estimate using `MAX_BLOCK_SIZE` and the RP2040 placement using `SLOT_SIZE_BYTES` may not be the same calculation. Until the intended divisor is confirmed, capacity estimates derived from the general formula should be treated carefully.

### OG-004 — Telemetry drift and sharp edges

The telemetry implementation notes list several current drift points and sharp edges that must remain explicit:

- Probe self-update writes versioned binaries plus `start.sh`, while an older service file starts a fixed binary path under `target/release`.
- HUB docs drift around `/download`, `/update`, filter command naming, KV key naming, interval-policy description, and cleanup trigger location.
- Collector docs drift around config fields, cursoring model, response-shape expectations, and `last_id` vs. timestamp state.
- CLI docs drift around `command(...)` vs. `run_command(...)`, node-scoped `set_update_interval(...)`, quoted string parsing, and broad syntax-support claims.
- Update Server and Probe assumptions differ around setup/startup models.
- Code-grounded sharp edges remain across Probe, HUB, Collector, CLI, and Update Server behavior.

The telemetry implementation document remains the detailed source; this register only points to the operational risk cluster.

### OG-005 — UTXO saturation and dynamic custodian-fee questions

The accepted MVP stance is fixed-fee carry-forward with no explicit saturation handling. The post-MVP dynamic custodian-fee concept leaves these questions open:

1. when the active-chain unspent-UTXO count is sampled during carry-forward,
2. whether fees are recomputed or captured canonically across chain switches,
3. how the fee curve is encoded for compact deterministic RP2040-class evaluation,
4. where fee bounds are enforced,
5. where the carry-forward flow binds the chain-info source, and what a window-anchored reconstruction does with fees it cannot evaluate,
6. and what structured observability should expose for fee resolution.

The concept does not change MVP requirements or commit to fee curves, encodings, or thresholds.

### OG-006 — Post-MVP long-disconnect recovery questions

The long-disconnect recovery concept is intentionally pre-design. It leaves these questions open for a future requirements and ADR pass:

1. watcher-mode exception to the current too-new-block discard rule,
2. same-chain-only recovery boundary and no cross-chain merging,
3. stale-subgroup coverage and retry limits,
4. private vs. public subgroup seed trade-off,
5. exact semantics of `n` consecutive too-new blocks,
6. confirmation timeout/count/deduplication/carry-over mechanics,
7. confirm-before-delete ordering,
8. retry behavior and anchor rollover,
9. replay determinism when private PRNG state affects subgroup selection,
10. recovery-anchor pinning,
11. subgroup-identifier disclosure,
12. new blockchain message types and size constraints,
13. and Sybil exposure against a long-disconnected trusted set.

The concept does not change MVP requirements and does not commit to parameter ranges, message layouts, or aggregation strategies.

### OG-007 — Processing as a concurrent-ingestion lifecycle state

The MVP treats processing as a transient reconstruct-and-validate pass: a single forward pass, not resumable across restarts (BC-FR3 / BC-FR59), that falls back to collecting under FR5 atomic recovery on any contradiction. The post-MVP concept promotes processing to a first-class operating state that continues to ingest blocks while reconstruction runs. It leaves these questions open:

1. whether the reconstruction pass stays atomic-and-non-resumable while only intake runs alongside it, or processing itself becomes resumable (reconciling BC-FR3 / BC-FR59),
2. which state FR5 atomic recovery rolls back on failure, and which concurrently-ingested blocks survive into the fallback collecting round,
3. whether an in-flight pass completes against its pinned candidate or is re-targeted when a stronger candidate arrives mid-pass, and how either choice preserves replay determinism,
4. how bounded-storage capacity pressure is shared between concurrent intake and the reconstruction working set,
5. how the authoritative FR1–FR5 two-phase-plus-transition lifecycle contract and its restatements are updated to a three-state model,
6. and what structured observability should expose to distinguish a slow-but-advancing pass from a stalled one.

The concept does not change MVP requirements and does not commit to a concurrency model, a resume mechanism, or a revised lifecycle-state contract.

## Related Documents

- [`moonblokz-system-constraints.md`](./moonblokz-system-constraints.md) — first stop for numeric constraints, capacity caps, memory budgets, flash geometry, and airtime-related gaps.
- [`moonblokz-telemetry-implementation.md`](./moonblokz-telemetry-implementation.md) — detailed source for telemetry drift, repository assumptions, and implementation sharp edges.
- [`blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md`](./blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md) — accepted bounded-UTXO preservation strategy and no-saturation-handling stance.
- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) — source for post-MVP dynamic custodian-fee, long-disconnect recovery, and concurrent-ingestion processing-state concepts.
- [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) — points here for the approval-evidence encoding gap.
- [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) — formal storage behavior that underlies flash and slot-geometry questions.
