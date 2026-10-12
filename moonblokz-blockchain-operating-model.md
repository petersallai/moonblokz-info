# MoonBlokz Blockchain Operating Model

One-read picture of how the `moonblokz-blockchain` module behaves at runtime: which lifecycle phase does what with each event, and how a block moves through its statuses. It is a **consolidation, not a source**: every row cites the requirement that defines it, and the [Blockchain PRD](./moonblokz-blockchain-prd.md) wins on any divergence. Read the cited FR before relying on a detail.

The **Built** columns record implementation state as of 2026-10-12 (`moonblokz-blockchain` at `1496b76`): **built**, **partial**, or **not built**, with the owning story from the BMAD plan (`_bmad-output/planning-artifacts/epics.md`). Update them when a story lands.

Current scope: lifecycle and block statuses. Invariants, derived projections, and the main flows (forward extension, chain switch, tail replay, block creation) are still to be added.

## 1. Execution Model

- The module is a single-threaded synchronous core driven by calls: intake (`receive_block`, `receive_transaction`, `receive_support`, `submit_local_transaction`), `on_tick`, read-only queries, and the boot calls `init` / `process_genesis` / `initialize_from_storage` ([architecture](./moonblokz-blockchain-architecture.md) §1.3, §3; [ADR-006](./blockchain-adrs/ADR-006-moonblokz-blockchain-uses-a-single-threaded-synchronous-core.md)).
- **Processing is never observable between calls.** The FR3 pass runs to completion inside the `receive_block` or `initialize_from_storage` call that triggered it. During an FR59 restart rebuild the module accepts no intake at all.
- Chain configuration has three commitment states — absent, tentative (RAM only, re-derived on restart), durable (set once) — owned by the configuration module ([Configuration Module Specification](./moonblokz-configuration-specification.md) §8).

## 2. Lifecycle Phases and Transitions

Two resting phases, **Collecting** and **Ready**, joined by the transient **Processing** pass (FR1–FR5). Node #0 skips Collecting entirely (FR54).

| # | From → To | Trigger | What runs | On failure | FRs | Built |
|---|---|---|---|---|---|---|
| T0 | boot → Collecting | `init`, then `initialize_from_storage`: empty storage = join, populated = restart | Restart: rebuild the block-tree and `chain_heads` from durable blocks, load a durable configuration if one exists, evaluate FR2 | Restart reports `Rejected` (control plane unreadable / configuration unusable) | FR1, FR59 | built (5.1, 5.10) |
| T1 | boot → Ready | `process_genesis` on node #0 | Create blocks #0 and #1, durably lock the configuration, take #0/#1 as the starting active chain; no FR2 / FR3 | Refused off node #0 or on non-empty storage | FR54 | partial — #0/#1 stay Stored with no projections (Story 6.0) |
| T2 | Collecting → Processing | A segment satisfies FR2: genesis-anchored or at least `W` long, and containing a chain-config block; highest tip sequence wins, then lowest tip hash | FR3 pass: backward marking, candidate chain-config preload, forward traversal with per-node seeding | — | FR2, FR3 | built (5.2, 5.3, 5.13) |
| T3 | Processing → Ready | The FR6 re-execution passes and the FR8 final check matches | Durable set-once configuration lock; mismatching chain-config blocks deleted with descendants; the candidate becomes Active | — | FR4, FR6, FR8 | built (5.4, 5.9) |
| T4 | Processing → Collecting | Any FR6 invariant violation | FR5 atomic recovery: discard the working set, delete the earliest offending block (or the candidate head if unidentifiable) and its descendants, re-evaluate FR2 | — | FR5 | built (5.5) |
| T5 | Processing → Processing | Candidate configuration differs from the tentative one | Replace the tentative configuration with the candidate's (re-checked per FR7) and retry, or roll back per FR5 | — | FR8 | built (5.9) |
| T6 | within Collecting | First chain-config block passing FR7 and accepted by the configuration module | Tentative load; later chain-config blocks are stored but do not override it | Module rejection is exact evidence; no load | FR8 | built (5.9) |
| T7 | Ready → Processing → Ready | Deep-zone chain switch: a candidate tip at least `S_tail + W − 1` with higher branch value | FR3 pass on the candidate; on failure a fallback pass on the original chain | Collecting only if the fallback also fails | FR58 | not built (9.7) |
| T8 | Processing interrupted | Restart mid-pass | Partial state discarded; T0 restart path | — | FR3, FR59 | built (5.10) |

## 3. Event × Phase Matrix

Node #0 after genesis behaves as Ready. "Not-ready" is the FR1 indication returned to every caller.

| Event | Collecting | Processing | Ready | Built (Ready path) |
|---|---|---|---|---|
| Inbound block | Outcomes reduce to Accepted-new / Already-known / Invalid-evidence (no window check); Tier 1 only, so every kept block stays Stored; the first valid chain-config loads tentatively; FR2 re-evaluated (FR1, FR8, FR9, FR60) | Not separately specified for external intake; the pass itself runs Tier 1 + 2 + 3 atomically on the candidate (FR9) | Exactly one FR10 outcome: Accepted-new, Already-known, Invalid-evidence, Out-of-Snake-Chain-Window, Deferred; may carry one support message (FR12), one re-disseminated transaction (FR13), one new block (FR45); no cause is returned — it goes to the FR64 log | partial — Tier 1 and storage built (4.2, 4.3); window check 9.1; Tier 2 6.1; Tier 3 / forward extension 6.0, 6.3; response components 6.4, 7.5, 8.3 |
| Inbound transaction | Not-ready; no classification, no mempool admission (FR14) | Not-ready (FR10) | Exactly one FR14 outcome: Accepted-to-mempool, Already-confirmed, Duplicate-in-mempool, Deferred, Rejected | not built (7.3) |
| Local transaction submit | `Rejected(not-ready)` (FR55) | `Rejected(not-ready)` | Created, Held(reason), or Rejected(reason) (FR55) | not built (10.5) |
| Inbound support message | Accepted on format validity and LRU de-duplication alone; state-dependent checks deferred (FR26) | Same as Collecting (FR26) | Full FR26 checks; an Accepted message is accumulated by the proposer (FR27) | not built (6.5, 6.6) |
| Tick — parent recovery | At most one request per tick, toward the Stored head whose retry interval elapsed longest ago (FR1, FR19) | Not specified | Same as Collecting (FR19 has no phase limit) | built (4.4) |
| Tick — mempool replenishment | Not scheduled (FR46) | Not scheduled (FR46) | Runs on every node whose mempool is below the fill threshold, creator or not (FR46) | not built (8.4; responder 10.4) |
| Tick — block creation | Not-ready (FR1 gates FR44) | Not specified | If the local node is the current creator: create when the fill threshold is reached, or on the timer with at least one includable transaction; replay obligations go out as soon as the node is eligible (FR45, FR46) | not built (8.3, 8.4; replay 9.2–9.5) |
| Tick — grace period | Not specified | Not specified | Widen the admitted creator set by one; no accumulated-vote reset (FR47) | not built (8.5) |
| Queries | Not-ready (FR1, FR43) | Not-ready (FR42, FR43); FR40 / FR41 answers are not observable before Ready (FR8) | Transaction state Unknown / In-mempool / Confirmed with depth (FR40); balance and accumulated vote, address UTXOs (FR41); active-chain blocks only, else not-available (FR42); at most one transaction (FR43) | not built (10.1–10.4; not-ready gates built in 5.1) |
| Creator-role determination | Not-ready (FR1) | Not specified | Whether the local node tops the FR38 creator order (FR44) | not built (8.2) |

## 4. Block Status Machine

### 4.1 Statuses

| Status | Meaning | FR |
|---|---|---|
| **Stored** | Parses, has a known `payload_type`, passed every applicable Tier 1 check | FR9 |
| **Connected** | Stored, ancestry reaches the active chain, passed Tier 2 | FR9 |
| **Active** | On the selected active chain, verified at Tier 3 | FR9 |
| Invalid | A terminal classification, not a status: the block and its transitive descendants are deleted | FR5, FR9 |

In Collecting every kept block is Stored; Processing exposes no intermediate status. Only Active counts as confirmed (FR9).

### 4.2 Check Tiers

| Tier | Gate into | Where it runs | What it checks (summary — FR9 holds the full list) |
|---|---|---|---|
| 1 | Stored (durable storage) | Intake, every phase; items marked *ready* only in Ready | version and payload schema; *locked (any phase):* size ≤ `block_size_limit`, chain-config content equals the lock (FR17); *ready:* window membership (FR60); `block.sequence > anchor_sequence`; no self-vote; registration / complex-transaction exclusivity; `new_key_signature`; node-#0 trust-anchor checks (FR69); *ready:* creator signature when the block directly extends the active head |
| 2 | Connected | Ready (Processing: inside the FR3 pass) | ancestry reaches the active chain; locked chain-config rules at the block's sequence (fee policy, registration price, custodian fee, approval threshold); `mined_amount` equals `mining_reward`; UTXO-input structural completion; registration key unique; structural `snake_chain` replay checks where the reference is retained (else deferred to Tier 3) |
| 3 | Active | The FR3 pass, FR23 chain-switch forward replay, and in Ready a direct extension of the active head or an already-stored child of it (FR16, FR9) | vote fields against candidate vote state; creator balance; balance-block contents and `max_node_id`; transaction signatures, balances, duplicates, node-id stride, UTXO spent bits, no negative balance; carry-forward validity; approval-evidence cluster; pacing (soft-fail to Deferred) |

Checks derived from chain configuration can invalidate a block only after the configuration is durably locked (FR8, FR54). Under a tentative configuration a failure does not delete the block; it is re-evaluated when the tentative configuration changes or the lock happens (FR9).

### 4.3 Transitions

| From → To | Trigger | FRs | Built |
|---|---|---|---|
| intake → Stored | Tier 1 passes | FR9, FR16 | built (4.2) |
| intake → not stored | Tier 1 exact evidence (Invalid-evidence); outside the window in Ready; chain-config content ≠ lock (silent); duplicate (no-op); a valid block storage fails to persist (reported as Already-known, cause logged) | FR10, FR11, FR16, FR17, FR60 | built (4.2, 4.3) — window check inactive until 9.1 |
| Stored → Connected | Ancestry reaches the active chain and Tier 2 passes | FR9 | not built (6.1) |
| Stored → Active | A direct extension of the active head (checked before storing; a failure is not stored), or an already-stored child of the active head, passes Tier 2 and Tier 3 in one step | FR9, FR16 | not built (6.0) |
| Connected → Active | FR23 reconciliation succeeds | FR9 | not built (6.3) |
| Stored → Active | FR3 pass succeeds (including the FR58 deep zone) | FR3, FR9 | built (5.4) |
| Active → Connected | FR23 demotes the block's branch | FR9, FR23 | not built (6.3) |
| Stored / Connected stays | Tier 3 pacing soft-fail → Deferred, no deletion | FR9, FR10 | not built (8.3) |
| → deleted (Invalid) | Tier 1 failure or FR8 lock mismatch (from Stored); Tier 2 failure (from Stored); Tier 3 failure — via FR5 in the FR3 pass, via FR23, or for a stored child of the active head, always with descendants; FR5 rollback and phase reversion apply to the FR3 pass only | FR5, FR8, FR9 | partial — FR3-pass and FR8 paths built (5.5, 5.9); Ready paths 6.0, 6.1, 6.3 |
| non-active → deleted | `chain_heads` overflow: evict the lowest non-active head, delete blocks whose reference count reaches 0 | FR19 | built (4.4) |
| side branch → deleted | Storage pressure: exactly one lowest-value side branch; never active-chain blocks | FR57, FR9 | not built (9.6) |
| side branch → deleted | Its divergence point fell below `S_tail` | FR58 | not built (9.7) |
| Active tail → dropped | Head extension past `W` (essential state replayed in that head, FR48): the block is released, a surviving child is unlinked, and a side head whose fork point falls below `S_tail` demotes to Stored until FR58 reclaims it | FR19 (v), FR48 | not built (9.1) |

Durable storage keeps only the node id, the private key, the chain-config block with its lock flag, and the retained blocks — no phase marker, no projections (FR59).

## Related Documents

- [Blockchain PRD](./moonblokz-blockchain-prd.md) — authoritative wording of every requirement cited here; find one by `^- FR<n>:` via the [FR/NFR Navigation Index](./moonblokz-blockchain-prd-fr-index.md).
- [Blockchain Architecture](./moonblokz-blockchain-architecture.md) — the API surface, module split, and data structures behind these behaviors.
- [Configuration Module Specification](./moonblokz-configuration-specification.md) — configuration commitment states and the parameters the checks read.
- [Blockchain Algorithm](./moonblokz-blockchain-algorythm.md) — data structures and the article-era reasoning behind the algorithms.
- [Glossary](./moonblokz-glossary.md) — `block_size_limit` vs. `MAX_BLOCK_SIZE`, `snake_chain`, and other cross-subsystem terms.
- [Open Gaps Register](./moonblokz-open-gaps-register.md) — what is still undecided.
