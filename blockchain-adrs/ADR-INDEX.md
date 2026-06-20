# Blockchain ADR Index and Roadmap

This index summarizes the accepted ADR set created from the MoonBlokz blockchain brainstorming and follow-up refinement work.

The FR-numbered references that appear in individual ADRs (for example, "per FR9 Tier 3", "per FR50") anchor to the authoritative source at [`../moonblokz-blockchain-prd.md`](../moonblokz-blockchain-prd.md); the canonical wording of every cited FR lives there.

## Purpose

Use this file as the entry point to the current blockchain decision set.

It helps answer:
- which decisions are already accepted,
- how they group together,
- in what order they should be read,
- and which later design work they still imply.

---

## Recommended Reading Order

### Foundation
1. [ADR-001: `moonblokz-blockchain` Is a Semantic Event State Machine](./ADR-001-moonblokz-blockchain-is-a-semantic-event-state-machine.md)
2. [ADR-002: Responsibility Boundary of `moonblokz-blockchain`](./ADR-002-responsibility-boundary-of-moonblokz-blockchain.md)
3. [ADR-003: Authoritative vs Derived State in `moonblokz-blockchain`](./ADR-003-authoritative-vs-derived-state-in-moonblokz-blockchain.md)

### Internal Core
4. [ADR-004: Durable Blockchain Storage Persistence Threshold](./ADR-004-durable-blockchain-storage-persistence-threshold.md)
5. [ADR-005: Chain Knowledge Core Is the Primary Orchestration Source](./ADR-005-chain-knowledge-core-is-the-primary-orchestration-source.md)
6. [ADR-006: `moonblokz-blockchain` Uses a Single-Threaded Synchronous Core](./ADR-006-moonblokz-blockchain-uses-a-single-threaded-synchronous-core.md)
7. [ADR-007: Vote Module Consumes Scoring Module Input](./ADR-007-vote-module-consumes-scoring-module-input.md)

### Local and Mempool Behavior
8. [ADR-008: Local Query Surface Is Active-Chain-Centered](./ADR-008-local-query-surface-is-active-chain-centered.md)
9. [ADR-009: Mempool Is Independent but Chain-Governed](./ADR-009-mempool-is-independent-but-chain-governed.md)
10. [ADR-010: Randomized Mempool Eviction Under Capacity Pressure](./ADR-010-randomized-mempool-eviction-under-capacity-pressure.md)

### Rare but Critical Runtime Behavior
11. [ADR-011: Chain-Switch Reconciliation Is a Structured Workflow](./ADR-011-chain-switch-reconciliation-is-a-structured-workflow.md)
12. [ADR-012: Verification Horizon Must Be Explicit Under Bounded Retention](./ADR-012-verification-horizon-must-be-explicit-under-bounded-retention.md)
13. [ADR-013: Bounded UTXO Retention Requires an Explicit Preservation Strategy](./ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md)
14. [ADR-014: Mode Transitions May Require Representation-Aware Design](./ADR-014-mode-transitions-may-require-representation-aware-design.md)
15. [ADR-015: Approval Subgroup Selection](./ADR-015-approval-subgroup-selection.md)
16. [ADR-016: Sequence-Indexed UTXO Input Model with Per-Block Spent-Bit Vector](./ADR-016-sequence-indexed-utxo-input-model.md)

---

## Grouped View

### 1. Module identity and boundary
- ADR-001
- ADR-002
- ADR-003

These define:
- what the blockchain module is,
- what it is responsible for,
- and what counts as primary truth versus derived operational state.

### 2. Core internal architecture
- ADR-004
- ADR-005
- ADR-006
- ADR-007

These define:
- what is durable truth,
- what the Chain Knowledge Core owns,
- how the core executes,
- and how vote-related responsibilities split across the scoring module (which supplies the vote target at transaction-creation time) and the blockchain vote module (which tracks accumulated votes and determines the next block creator).

### 3. Local and transaction-facing behavior
- ADR-008
- ADR-009
- ADR-010

These define:
- the local query philosophy,
- the mempool’s role,
- and congestion behavior in the mempool.

### 4. Rare but correctness-critical behavior
- ADR-011
- ADR-012
- ADR-013
- ADR-014
- ADR-015
- ADR-016

These define:
- how chain switches are handled,
- how much history or equivalent support must remain verifiable,
- how bounded UTXO preservation must be treated,
- how lifecycle mode transitions affect internal representation,
- how the approval-process subgroup is deterministically selected,
- and how UTXO inputs are identified and how spent state is represented under bounded retention.

---

## One-Line Summary of Each ADR

- **ADR-001** — The blockchain core operates on semantic events, not transport-level byte handling.
- **ADR-002** — The blockchain module has a narrow domain boundary and excludes transport, storage mechanics, crypto backend details, and the scoring module.
- **ADR-003** — Known blocks, mempool, and operation mode are authoritative; active chain, balances/UTXOs, and vote state are derived.
- **ADR-004** — Durable blockchain storage retains every received block except those already rejected by intake-time exact evidence (parse failure, direct-active-extension signature invalidity, chain-config / trust-anchor mismatch cases, configuration-module rejection, or deviation-branch creator-exclusivity violation); broader creator-signature-invalidity decisions remain deferred to blockchain module PRD FR9 Tier 3 chain-switch reconciliation or blockchain module PRD FR3 processing pass; helper indexes are not primary durable truth.
- **ADR-005** — The Chain Knowledge Core is the main orchestration source for chain truth and reconciliation.
- **ADR-006** — The blockchain core remains single-threaded, synchronous, and deterministic.
- **ADR-007** — The scoring module computes per-node scores from the senders of observed radio messages and supplies the vote target to the upstream local transaction-building path before the signed transaction reaches the blockchain module; the blockchain vote module tracks each node's accumulated vote and determines the next block creator from that state.
- **ADR-008** — The local query surface is active-chain-centered by default.
- **ADR-009** — The mempool is independent in operation but remains governed by chain truth.
- **ADR-010** — Mempool eviction under pressure is randomized to improve whole-network transaction survival.
- **ADR-011** — Chain switch is handled as a structured reconciliation workflow.
- **ADR-012** — Verification horizon must be explicit under bounded retention.
- **ADR-013** — Bounded UTXO handling requires an explicit preservation strategy.
- **ADR-014** — Lifecycle mode transitions may require representation-aware design.
- **ADR-015** — Approval subgroup is selected by rendezvous-hash ordering over active-chain-derived active nodes, seeded by the snake-chain tail hash, the proposer node identity, and the proposed sequence; subgroup size is tied to `required_support`, capped by the active set, with no in-subgroup expansion fallback.
- **ADR-016** — UTXO inputs reference `(block_sequence, output_index)` rather than transaction hashes; spent state is tracked as a per-block spent-bit vector co-located with each block, eliminating the need for an in-memory live UTXO set.

---

## What These ADRs Still Imply

These ADRs settle the main architectural direction. Their downstream design work has largely been completed during the 2026-05-19 → 2026-06-17 architecture workflow recorded in [`../moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md). Status:

1. **Chain Knowledge Core internal representation** — addressed in architecture §4 (15 + 1 internal modules) and §6 (sized data-structure catalog with `BlockEntry`, `ChainHeadsTable`, `NodeInfo` SoA, `SchedulerState`, etc.).
2. **Detailed chain-switch reconciliation invariants** — addressed in architecture §4.2 (`reconciliation.rs` module) covering the backward-walk + forward-walk workflow.
3. **Verification-horizon sizing and retained-information policy** — addressed in architecture §5 (`VERIFICATION_HORIZON = 20` const default) and the snake_chain bounded-retention model (W=500).
4. **Bounded UTXO carry-forward policy and saturation handling** — addressed in architecture §6.2 (co-located spent-bit vector per `BlockEntry`) and §4.2 (`spent_bits.rs`); ADR-013 + ADR-016 remain the conceptual anchors. FR52 ("no UTXO saturation detection") remains an explicit MVP-skip.
5. **Query payload definitions and depth semantics** — addressed in architecture §3.1 (12 read-only public methods on `Blockchain<...>`) and §4.2 (`queries.rs` module).
6. **Vote-target input contract from the scoring module** — partly open. The scoring-module → blockchain-vote-module contract (ADR-007) is conceptually settled, but the exact scoring-module API surface remains a separate design artifact.
7. **Mode-specific shared versus distinct data structures** — addressed in architecture §3.2 (three-type Owned + View + Builder model) and §6.6 (`EmitScratch` outcome view source).

---

## Suggested Next Design Artifacts

Based on the accepted ADR set:

1. **Chain Knowledge Core architecture note** — **DONE.** Superseded by [`../moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md).
2. **Chain-switch reconciliation design note** — **DONE.** Covered by architecture §4.2 (`reconciliation.rs` module).
3. **Bounded UTXO preservation design note** — **DONE.** Covered by architecture §6.2 (co-located spent-bit vector) and ADR-016.
4. **Local query contract note** — **DONE.** Covered by architecture §3.1 (read-only API surface) and §4.2 (`queries.rs`).
5. **Scoring module vote-target selection input contract** — still open. The scoring-module is explicitly outside the `moonblokz-blockchain` boundary (ADR-002 / ADR-007), and its concrete API contract belongs to a separate design artifact yet to be authored.

---

## Status Summary

All ADRs listed in this index are currently treated as **Accepted** within the working design set created during this session. The downstream architectural work derived from this ADR set was completed on 2026-06-17 — see [`../moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) for the authoritative consolidated reference.
