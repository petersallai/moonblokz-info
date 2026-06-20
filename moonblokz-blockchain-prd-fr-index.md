# MoonBlokz Blockchain PRD — FR/NFR Navigation Index

> **Non-authoritative navigation aid.** The authoritative source for every requirement listed here is [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md); on any divergence the PRD wins. The entries below are short navigation labels, not requirement text. Use this index to find which requirement covers a topic, then open only that one requirement in the PRD.

## How to read a single requirement without loading the whole PRD

The PRD is large and each functional requirement is a single very long line, so reading the file in full is costly. To read just one:

1. Search the PRD for the requirement's anchor — functional requirements begin a line with `- FR<n>:` and non-functional requirements with `- NFR<n>:`. For FR52 the anchor pattern is `^- FR52:`; for NFR12 it is `^- NFR12:` (the trailing colon keeps `FR5`/`FR52` and `NFR1`/`NFR12` distinct).
2. Read that single line at the reported line number.

Search by anchor rather than citing a line: anchors are stable across edits, line numbers are not.

## Functional Requirements (FR1–FR69)

### Chain Lifecycle & State Progression
| FR | Subject |
|----|---------|
| FR1 | Collecting-state initialization with not-ready gating for dependent behaviors |
| FR2 | Dominant-chain acquisition stopping conditions and deterministic tip selection |
| FR3 | Processing-state transition executing full-chain reconstruction forward pass |
| FR4 | Ready-state entry conditioned on successful chain validation |
| FR5 | Atomic working-set rollback and durable-deletion recovery on validation failure |
| FR6 | Full candidate-chain block and transaction invariant verification before ready |
| FR7 | Chain-config content-signature validation against node-zero trust anchor |
| FR8 | Chain-config tentative-load and durable-lock commitment phases |
| FR9 | Three-status block lifecycle and three-tier validation check system |

### Block & Transaction Intake
| FR | Subject |
|----|---------|
| FR10 | Block-intake deterministic five-outcome classification |
| FR11 | Already-known duplicate detection via in-memory block-tree index |
| FR12 | Deviating-block endorsement support-message eligibility and generation |
| FR13 | Under-disseminated mempool transaction re-dissemination on block intake |
| FR14 | Transaction-intake five-outcome classification in ready state |
| FR15 | Deferred mempool transaction re-evaluation on active-chain changes |
| FR16 | Block stored unless Tier 1 exact invalidity evidence exists |
| FR17 | Chain-config content-mismatch blocks silently discarded post-durable-lock |

### Chain Knowledge, Branching & Reconciliation
| FR | Subject |
|----|---------|
| FR18 | Per-block block-tree metadata retention requirements |
| FR19 | Missing-parent recovery scheduling via bounded chain_heads table |
| FR20 | Multiple candidate branches simultaneously retained in block-tree |
| FR21 | Branch-value computation and highest-value active-branch selection |
| FR22 | Active-branch re-evaluation on non-active head value increase |
| FR23 | Chain-switch reconciliation as atomic step-wise backward/forward walk |
| FR24 | Consistent single-branch derived state visible to all callers |
| FR25 | Chain-switch and approval-evidence paths treated as first-class behaviors |
| FR26 | Support-message intake, validation rules, and relay classification |
| FR27 | Approval-evidence block construction and accumulation by deviation proposer |
| FR28 | Deviation-block immediate acceptance and two-stage branch scoring |
| FR29 | Snake_chain replay precedence preserved during deviation-approval workflow |

### Mempool, Balances & Vote State
| FR | Subject |
|----|---------|
| FR30 | Mempool as separate module with compacted bounded storage |
| FR31 | Mempool add invoked only on Accepted-to-mempool outcome |
| FR32 | Mempool reconciliation on forward extension and chain-switch events |
| FR33 | Randomized mempool eviction with own-transaction priority preservation |
| FR34 | Six derived operational-state projections maintained from active chain |
| FR35 | Derived projections incrementally updated on two trigger event types |
| FR36 | Block-creator mining reward, transaction fee, and replay-reward credit |
| FR37 | Accumulated-vote accumulation, anti-capture interest growth, and creator reset |
| FR38 | Creator-order projection derived and refreshed from vote state |
| FR39 | Rollback-sensitive derived-state correctness through FR23 reconciliation |

### Queries & Retrieval
| FR | Subject |
|----|---------|
| FR40 | Transaction-state query returning Unknown, In-mempool, or Confirmed |
| FR41 | Node-balance and address-UTXO unspent-output query surfaces |
| FR42 | Active-chain block retrieval by sequence or hash, ready-state only |
| FR43 | Top-mempool-items replenishment exchange read-only surface |

### Block Creation & Timeout Recovery
| FR | Subject |
|----|---------|
| FR44 | Local creator determination and grace-period fallback set expansion |
| FR45 | Block-creation triggers, content-assembly priority, and header construction |
| FR46 | External time-driver next-wakeup query and tick callback contract |
| FR47 | Grace-period expiry advancing fallback cycle without accumulated-vote reset |

### Snake_chain & Tail Preservation
| FR | Subject |
|----|---------|
| FR48 | Bounded active-chain length and pre-drop essential-state preservation check |
| FR49 | Chain-config replay block emitted before dropping chain-config block |
| FR50 | Balance-block replay triggers and per-node seed-source coverage invariant |
| FR51 | UTXO carry-forward with custodian-fee reduction for unspent outputs |
| FR52 | No UTXO-saturation detection, backpressure, or special-case handling |
| FR53 | Sequence u32 ceiling with explicit u32::MAX block refusal |

### Genesis & Bootstrap
| FR | Subject |
|----|---------|
| FR54 | Genesis initiation, bootstrap block creation, and validation exceptions |

### External Input Contracts
| FR | Subject |
|----|---------|
| FR55 | Local transaction-creation surface with Created, Held, Rejected outcomes |
| FR56 | Chain-configurable parameters accessed via configuration_module accessors |

### Storage & Retention Policy
| FR | Subject |
|----|---------|
| FR57 | Capacity-pressure eviction removes single lowest-value side branch |
| FR58 | Verification horizon and cheap-zone vs. deep-zone chain-switch discipline |

### Lifecycle & Recovery
| FR | Subject |
|----|---------|
| FR59 | Durable-storage scope, restart reconstruction, and intake suspension |
| FR60 | Out-of-snake-chain-window classification and long-disconnect diagnostic log |

### Module Boundary
| FR | Subject |
|----|---------|
| FR61 | Whole-block boundary contract using canonical moonblokz-chain-types |

### Simulation, Observability & Public API
| FR | Subject |
|----|---------|
| FR62 | Same moonblokz-blockchain crate build for simulator and embedded runtime |
| FR63 | Deterministic replay from sealed event sequence and PRNG seed |
| FR64 | Structured log records emitted for every internally significant event |
| FR65 | no_std synchronous API without heap allocation and no-panic guarantee |
| FR66 | Adjacent components consume blockchain state only via public API |

### Initialization Parameters
| FR | Subject |
|----|---------|
| FR67 | local_node_id immutable code-level node-identity parameter |
| FR68 | All signing delegated to crypto module; no private key held |
| FR69 | node_zero_public_key immutable code-level trust-anchor parameter |

## Non-Functional Requirements (NFR1–NFR26)

### Performance
| NFR | Subject |
|-----|---------|
| NFR1 | Bounded memory and no history-length-dependent scans on steady-state paths |
| NFR2 | Retained-state-only intake and query without ready-state full-chain reconstruction |
| NFR3 | Lightweight scheduling without wall-clock polling beyond the external tick model |
| NFR4 | Efficient branch, head, and reconciliation prechecks with deep-zone-only reconstruction |

### Reliability
| NFR | Subject |
|-----|---------|
| NFR5 | Deterministic authoritative-state reconstruction from identical retained and replay inputs |
| NFR6 | No partially applied derived state visible to callers on failure |
| NFR7 | Safe discard-and-recompute of partial reconstruction state on restart during processing |
| NFR8 | Convergence tolerance under partial, delayed, and conflicting network knowledge |

### Security and Integrity
| NFR | Subject |
|-----|---------|
| NFR9 | Cryptographic validation against canonical projections, not transport-time observations |
| NFR10 | ADR-004 permissive durable intake with deferred authoritative invalidation |
| NFR11 | Correctness protection preferring rejection/reversion over partial acceptance |
| NFR12 | Chain-configuration integrity preserved for the chain lifetime |

### Capacity and Resource Bounds
| NFR | Subject |
|-----|---------|
| NFR13 | Combined RP2040-LoRa runtime footprint fit, not the blockchain module alone |
| NFR14 | RP2040-LoRa fit without dropping correctness-critical behaviors |
| NFR15 | Portability to stronger no_std platforms without semantic changes |
| NFR16 | Operation within bounded retention, horizon, block-tree, and mempool budgets |
| NFR17 | Optional retained history as efficiency optimization, not correctness prerequisite |
| NFR18 | Capacity pressure handled by deterministic eviction and bounded retention |
| NFR19 | Large-network degradation through documented bounded mechanisms |

### Integration
| NFR | Subject |
|-----|---------|
| NFR20 | Clear typed boundaries to adjacent runtime components |
| NFR21 | Integration contracts stable enough to avoid competing authoritative interpretations |
| NFR22 | Semantically narrow dependencies with truth selection inside the module |
| NFR23 | Simulator and replay exercise the same semantic contracts as runtime |

### Observability and Testability
| NFR | Subject |
|-----|---------|
| NFR24 | Structured logs reconstruct correctness-critical decisions without hidden inspection APIs |
| NFR25 | Deterministic replay in test and simulation from retained inputs |
| NFR26 | Bounded, test-harness-controllable non-deterministic production behaviors |

## Related Documents

- [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) — the authoritative PRD this index points into; full requirement wording lives here.
- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) — operating-model background for these requirements.
- [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) — formal algorithms that implement these requirements.
- [`moonblokz-blockchain-implementation.md`](./moonblokz-blockchain-implementation.md) — engineering guidance derived from these requirements.
- [`blockchain-adrs/ADR-INDEX.md`](./blockchain-adrs/ADR-INDEX.md) — architecture decisions taken in service of these requirements.
- [`moonblokz-index.md`](./moonblokz-index.md) — knowledge-base table of contents and topic finder.
