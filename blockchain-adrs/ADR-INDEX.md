# Blockchain ADR Index and Roadmap

This index summarizes the accepted ADR set created from the MoonBlokz blockchain brainstorming and follow-up refinement work.

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
4. [ADR-004: Durable Blockchain Storage Persists Only Signature-Valid Blocks](./ADR-004-durable-blockchain-storage-persists-only-signature-valid-blocks.md)
5. [ADR-005: Chain Knowledge Core Is the Primary Orchestration Source](./ADR-005-chain-knowledge-core-is-the-primary-orchestration-source.md)
6. [ADR-006: `moonblokz-blockchain` Uses a Single-Threaded Synchronous Core](./ADR-006-moonblokz-blockchain-uses-a-single-threaded-synchronous-core.md)
7. [ADR-007: Vote Engine Consumes Radio-Derived Score Input](./ADR-007-vote-engine-consumes-radio-derived-score-input.md)

### Local and Mempool Behavior
8. [ADR-008: Local Query Surface Is Active-Chain-Centered](./ADR-008-local-query-surface-is-active-chain-centered.md)
9. [ADR-009: Mempool Is Independent but Chain-Governed](./ADR-009-mempool-is-independent-but-chain-governed.md)
10. [ADR-010: Randomized Mempool Eviction Under Capacity Pressure](./ADR-010-randomized-mempool-eviction-under-capacity-pressure.md)

### Rare but Critical Runtime Behavior
11. [ADR-011: Chain-Switch Reconciliation Is a Structured Workflow](./ADR-011-chain-switch-reconciliation-is-a-structured-workflow.md)
12. [ADR-012: Verification Horizon Must Be Explicit Under Bounded Retention](./ADR-012-verification-horizon-must-be-explicit-under-bounded-retention.md)
13. [ADR-013: Bounded UTXO Retention Requires an Explicit Preservation Strategy](./ADR-013-bounded-utxo-retention-requires-an-explicit-preservation-strategy.md)
14. [ADR-014: Mode Transitions May Require Representation-Aware Design](./ADR-014-mode-transitions-may-require-representation-aware-design.md)

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
- and how vote-related responsibilities split across radio-side scoring and blockchain-side accumulation.

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

These define:
- how chain switches are handled,
- how much history or equivalent support must remain verifiable,
- how bounded UTXO preservation must be treated,
- and how lifecycle mode transitions affect internal representation.

---

## One-Line Summary of Each ADR

- **ADR-001** — The blockchain core operates on semantic events, not transport-level byte handling.
- **ADR-002** — The blockchain module has a narrow domain boundary and excludes transport, storage mechanics, crypto backend details, and radio-side score calculation.
- **ADR-003** — Known blocks, mempool, and operation mode are authoritative; active chain, balances/UTXOs, and vote state are derived.
- **ADR-004** — Durable blockchain storage persists only signature-valid blocks; helper indexes are not primary durable truth.
- **ADR-005** — The Chain Knowledge Core is the main orchestration source for chain truth and reconciliation.
- **ADR-006** — The blockchain core remains single-threaded, synchronous, and deterministic.
- **ADR-007** — External score calculation chooses the vote target for newly accepted transactions; blockchain owns vote accumulation and creator-selection consequences.
- **ADR-008** — The local query surface is active-chain-centered by default.
- **ADR-009** — The mempool is independent in operation but remains governed by chain truth.
- **ADR-010** — Mempool eviction under pressure is randomized to improve whole-network transaction survival.
- **ADR-011** — Chain switch is handled as a structured reconciliation workflow.
- **ADR-012** — Verification horizon must be explicit under bounded retention.
- **ADR-013** — Bounded UTXO handling requires an explicit preservation strategy.
- **ADR-014** — Lifecycle mode transitions may require representation-aware design.

---

## What These ADRs Still Imply

These ADRs settle the main architectural direction, but they still require deeper design work in several areas:

1. Chain Knowledge Core internal representation
2. Detailed chain-switch reconciliation invariants
3. Verification-horizon sizing and retained-information policy
4. Bounded UTXO carry-forward policy and saturation handling
5. Query payload definitions and depth semantics
6. Vote-target input contract from the external score-calculation module
7. Mode-specific shared versus distinct data structures

---

## Suggested Next Design Artifacts

Based on the accepted ADR set, the most useful next artifacts would be:

1. **Chain Knowledge Core architecture note**
2. **Chain-switch reconciliation design note**
3. **Bounded UTXO preservation design note**
4. **Local query contract note**
5. **External vote-target selection input contract**

---

## Status Summary

All ADRs listed in this index are currently treated as **Accepted** within the working design set created during this session.
