# MoonBlokz Open Gaps Register

## Role of This Document

This file is a cross-cutting register for unresolved gaps, verified documentation drift, and explicitly post-MVP design questions that affect MoonBlokz planning, implementation, deployment, or operations.

It is a consolidation and navigation layer, not an authoritative source. Each entry links to the knowledge-base document where the underlying fact, limitation, discrepancy, or open question is defined. If this register diverges from a linked source, the linked source wins and the divergence must be resolved under the source-fidelity rules in [`AGENTS.md`](./AGENTS.md).

## Source Basis

This register currently consolidates gaps already documented in:

- [`moonblokz-system-constraints.md`](./moonblokz-system-constraints.md) — numeric, capacity, airtime, flash, and storage-geometry gaps.
- [`moonblokz-telemetry-implementation.md`](./moonblokz-telemetry-implementation.md) — telemetry deployment-model drift, docs-vs-code drift, and implementation sharp edges.
- [`blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md`](./blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md) — accepted bounded-UTXO preservation strategy and explicit no-saturation-handling stance.
- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) — post-MVP dynamic custodian-fee and long-disconnect recovery concepts.

## Register

| ID | Area | Type | Current state | Impact | Primary source |
|---|---|---|---|---|---|
| OG-001 | Radio / deployment capacity | Gap | No regional LoRa duty-cycle limit is documented in the knowledge base. | Real-deployment airtime and capacity planning cannot be grounded in KB values. | [`moonblokz-system-constraints.md` §5/§7](./moonblokz-system-constraints.md#7-known-discrepancies-gaps) |
| OG-002 | Storage / flash durability | Gap | Flash wear-lifetime is not quantified. | Long-term durability budgeting remains undefined. | [`moonblokz-system-constraints.md` §4/§7](./moonblokz-system-constraints.md#7-known-discrepancies-gaps) |
| OG-003 | Storage geometry | Verify | A general slot-count formula divides by `MAX_BLOCK_SIZE` (2016), while the RP2040 placement contract uses `SLOT_SIZE_BYTES` (2048, including the per-slot hash). | General capacity estimates may slightly over-count slots unless the intended divisor is confirmed. | [`moonblokz-system-constraints.md` §7 G3](./moonblokz-system-constraints.md#7-known-discrepancies-gaps) |
| OG-004 | Telemetry / operations | Drift / sharp edges | Probe, HUB, Collector, CLI, and Update Server docs and deployment assumptions partially diverge from reviewed code. | Interoperability, operator expectations, update activation, command behavior, and log-collection reliability can be misunderstood if docs are treated as current without checking code-grounded notes. | [`moonblokz-telemetry-implementation.md` §Current Deployment-Model and Documentation Drift](./moonblokz-telemetry-implementation.md#current-deployment-model-and-documentation-drift), [`§Current Code-Grounded Inconsistencies and Sharp Edges`](./moonblokz-telemetry-implementation.md#current-code-grounded-inconsistencies-and-sharp-edges) |
| OG-005 | Blockchain / bounded UTXO retention | Accepted MVP limitation + post-MVP concept | The MVP uses a fixed chain-config-derived custodian fee and explicitly performs no special UTXO-saturation detection, status reporting, structured logging, mempool backpressure, or replay deferral. A dynamic saturation-aware custodian fee is only a post-MVP concept. | Under high live-UTXO accumulation, replay obligations can consume block capacity and ordinary mempool transactions may wait until fixed-fee erosion self-clears capacity. | [`ADR-013`](./blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md#no-explicit-utxo-saturation-handling), [`moonblokz-blockchain-concept.md` §Dynamic Custodian Fee](./moonblokz-blockchain-concept.md#dynamic-custodian-fee-post-mvp-concept) |
| OG-006 | Blockchain / long-disconnect recovery | Post-MVP concept | Long-disconnect resynchronization is out of MVP scope: the current model logs the condition and relies on external operator intervention. A chain-internal watcher/subgroup/anchor recovery path is conceptual only. | Deployments that expect long but recoverable disconnections need future requirement and architecture work before relying on chain-internal recovery. | [`moonblokz-blockchain-concept.md` §Long-Disconnect Recovery](./moonblokz-blockchain-concept.md#long-disconnect-recovery-post-mvp-concept) |

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
5. how backward compatibility with the current input-less custodian-fee accessor works,
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

## Related Documents

- [`moonblokz-system-constraints.md`](./moonblokz-system-constraints.md) — first stop for numeric constraints, capacity caps, memory budgets, flash geometry, and airtime-related gaps.
- [`moonblokz-telemetry-implementation.md`](./moonblokz-telemetry-implementation.md) — detailed source for telemetry drift, repository assumptions, and implementation sharp edges.
- [`blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md`](./blockchain-adrs/ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md) — accepted bounded-UTXO preservation strategy and no-saturation-handling stance.
- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) — source for post-MVP dynamic custodian-fee and long-disconnect recovery concepts.
- [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) — formal storage behavior that underlies flash and slot-geometry questions.
