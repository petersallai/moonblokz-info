# ADR-006: `moonblokz-blockchain` Uses a Single-Threaded Synchronous Core

### Status
Accepted

### Context
The blockchain module maintains authoritative and derived state under staged validation, bounded resources, and occasional active-chain changes. Hidden concurrency inside the blockchain core would increase the risk of race conditions, inconsistent derived-state updates, and hard-to-reproduce correctness bugs.

The accepted semantic-event-state-machine model already assumes serialized state transitions. The blockchain core therefore needs an execution model that preserves determinism rather than weakening it with internal concurrency.

### Decision
`moonblokz-blockchain` is implemented as a **single-threaded, synchronous, deterministic core**. Asynchronous orchestration, parallelism, and multicore concerns may exist around the module but remain outside the blockchain core boundary.

The execution model — single-`embassy::select` loop, single-outcome scheduling-pull API (`CallResult<OutcomeEnum> + NextCall`), Core 0 / Core 1 split — is normative in [`moonblokz-blockchain-architecture.md`](../moonblokz-blockchain-architecture.md) §1.1 and §1.3; the no-heap, no-panic, `no_std` guarantee is normative in [PRD](../moonblokz-blockchain-prd.md) FR65, with the deterministic-reconstruction requirement in NFR5.

### Consequences
#### Positive
- Simplifies reasoning about correctness.
- Aligns well with the semantic event state machine model.
- Reduces race-condition risk.
- Makes staged validation and chain-switch reconciliation easier to reason about.
- Preserves a clean serialized state-transition model.

#### Trade-offs
- May reduce raw throughput headroom.
- Requires external orchestration layers to serialize work before invoking the module.
- May create pressure to keep the core efficient and bounded under worst-case event bursts.
- Places more performance pressure on data structures and per-event processing cost.

### Follow-up implications
This decision implies that later design work must define:
- acceptable per-event processing cost,
- what work may remain synchronous inside the core,
- and what work must stay outside as asynchronous orchestration or preprocessing.
