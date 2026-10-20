# MoonBlokz Decision Register

One row per Project Lead decision that shapes behavior, requirements, public API, configuration, or the delivery plan of the blockchain stack, with its date, scope, where it is recorded, and its current status. The rule itself lives in the owning document — the [Blockchain PRD](./moonblokz-blockchain-prd.md), [Architecture](./moonblokz-blockchain-architecture.md), [Configuration Module Specification](./moonblokz-configuration-specification.md), [Storage PRD](./moonblokz-storage-prd.md), or a BMAD story — which wins on any divergence; this register is where to find it and whether it still holds.

Not repeated here: architecture decisions logged in [architecture §14](./moonblokz-blockchain-architecture.md#14-decisions-log-consolidated) (rows 1–24) and the accepted ADRs in the [ADR index](./blockchain-adrs/ADR-INDEX.md). Implementation-detail rulings that change no behavior stay in their story files.

**Status:** *in force* · *done* (one-off action completed) · *superseded by DEC-n* · *withdrawn*. **Owner** names the story that carries remaining code work. **Recorded in** uses: PRD / ARCH / CFG / ST-PRD (KB documents, `§` or FR), `story x.y` (`_bmad-output/implementation-artifacts/<x>-<y>-*.md`), `epics` (`_bmad-output/planning-artifacts/epics.md`), `DW` (`deferred-work.md`), `retro-n`.

## 1. Lifecycle, Intake, and Validation

| ID | Date | Decision | Affects | Recorded in | Status · Owner |
|---|---|---|---|---|---|
| DEC-001 | 2026-07-10 | No signature-verification cache: signatures are verified directly each time. | FR9, FR11 | story 4.2; DW | in force |
| DEC-002 | 2026-07-11 | A valid block that storage cannot save is refused with `Unstorable`, outside the FR10 validity outcomes. | FR10 | story 4.3 | superseded by DEC-034 (code until 11.2) |
| DEC-003 | 2026-07-11 | Only intake classification performs FR11 duplicate detection, hashing the block once. | FR11 | story 4.3 | in force |
| DEC-004 | 2026-07-21 | Ready-only entry points exist from the start and answer `Result<_, NotReady>`, never `Option`. | FR1, FR14, FR40–FR44, FR55 | story 5.1 | in force · bodies Epics 7, 8, 10 |
| DEC-005 | 2026-07-22 | FR2 picks the highest tip sequence, then the lowest tip hash; aligning it with the FR21 tie-break is a deferred optimization. | FR2, FR21 | PRD FR2; story 5.2; DW | in force |
| DEC-006 | 2026-07-22 | A bootstrap-anchored genesis counts as already acquired for FR2; the genesis pre-anchor stays. | FR2, FR3 | story 5.2; DW | in force |
| DEC-007 | 2026-07-23 | FR6 runs inside the single derive walk; on failure the per-block UTXO bits are re-zeroed before the phase reverts. | FR3, FR6 | story 5.4 | in force |
| DEC-008 | 2026-07-23 | Approval-evidence validation is not part of 5.4. | FR6, FR27 | story 5.4 | in force · 6.7 (DEC-077) |
| DEC-009 | 2026-07-23 | Duplicates of confirmed transactions: UTXO side via the per-block UTXO bits now; balance side at FR14 append. | FR6, FR14 | story 5.4 #4; DW | in force · 7.3 |
| DEC-010 | 2026-07-23 | UTXO value accounting and input signatures move to 7.1; the creator-entitlement check moves to Epic 8. | FR6, FR44 | story 5.4 review; DW | value accounting superseded by DEC-128; signatures in force · 7.1, 8.2 |
| DEC-011 | 2026-07-29 | Byte-decidable invariants are Tier 1 only and are not re-checked by the FR6 pass. | FR6, FR9 | PRD FR6; story 5.4 | in force |
| DEC-012 | 2026-07-30 | After an `Invalid` recovery FR2 is re-evaluated in the same call; at most 2 passes per call (raised to 3 by DEC-063). | FR2, FR5 | story 5.5; DW | in force |
| DEC-013 | 2026-07-30 | A storage-read or mark-overflow failure in the pass deletes one block, like an FR6 violation. | FR5 | DW (5.5 review) | superseded by DEC-037 (storage reads) and DEC-134 (mark overflow) |
| DEC-014 | 2026-07-30 | An FR5 delete set always includes every descendant. | FR5, FR57 | story 5.5; DW | in force |
| DEC-015 | 2026-09-20 | Chain-config-derived checks invalidate a block only after the durable lock, and re-run at the lock. | FR8, FR9 | PRD FR9; story 5.9 | in force (exceptions: the UTXO `W` distance bound, DEC-129; the per-block UTXO-output cap, DEC-133) |
| DEC-016 | 2026-09-20 | Every chain-config block's content is checked at intake, not only the one that loads. | FR8, FR16 | story 5.9; retro-5 | in force |
| DEC-017 | 2026-09-20 | FR54 genesis waivers stay keyed on sequence alone at Tier 1. | FR54 | DW (4.2) | superseded by DEC-040 |
| DEC-018 | 2026-10-08 | FR2 selects only segments that carry a chain-config block (no head loss). | FR2, FR5, FR8 | PRD FR2; story 5.13 | in force |
| DEC-019 | 2026-10-10 | Validate where the verdict is final: a direct head extension runs Tier 2 + 3 before storing and is not stored on failure; a failing stored child of the head is deleted with descendants; no FR5 in Ready. | FR5, FR9, FR16 | PRD; story 6.0 D2 | done (6.0) |
| DEC-020 | 2026-10-10 | A Ready projection is always complete (FR50 coverage): no trust zone, no persisted pass flags; an unseeded actor is Invalid in Ready. | FR3, FR50 | story 6.0 D3; DW | done (6.0) · coverage check 9.3 |
| DEC-021 | 2026-10-10 | Interim sibling order for stored children of a new head: lowest block hash. | FR22 | story 6.0 D5 | superseded by DEC-150 |
| DEC-022 | 2026-10-10 | Node #0 runs the ordinary per-block step over its own blocks #0 / #1; no node-#0 trust zone. | FR54 | story 6.0 D6 | done (6.0) |
| DEC-023 | 2026-10-10 | Blocks validated at the head go Stored → Active directly, with no Connected interim. | FR9 | PRD FR9; story 6.0 D7 | done (6.0) |
| DEC-024 | 2026-10-10 | Forward-extension outcomes: `InvalidEvidence`, `Unstorable`, or `AcceptedSilently`; no new variant. | FR10 | story 6.0 D8 | done (6.0; `Unstorable` until 11.2, DEC-034) |
| DEC-025 | 2026-10-10 | A balance block's payload must equal the derived pre-state, else it is Invalid; its seed-source pointer moves to that block. | FR9, FR35, FR50 | story 6.0 D9 | done (6.0) |
| DEC-026 | 2026-10-12 | FR57 eviction never removes active-chain blocks. | FR9, FR57 | PRD FR9 | in force · 9.6 |
| DEC-027 | 2026-10-12 | A Tier 2 failure deletes a block from Stored (Tier 2 is the gate into Connected). | FR9 | PRD FR9 | in force |
| DEC-028 | 2026-10-12 | The closed deletion lists include the FR48 tail-drop release and FR58 reclamation. | FR9, FR19, FR20 | PRD | in force |
| DEC-029 | 2026-10-12 | Node #0's own genesis blocks skip the Tier 1 re-check; received #0 / #1 do not. | FR54 | PRD FR54 | in force |
| DEC-030 | 2026-10-12 | The block-size and FR17 Tier 1 checks apply whenever the configuration is durably locked, in any phase. | FR9, FR16, FR17 | PRD | in force |
| DEC-031 | 2026-10-12 | The FR9 Tier 1 list is exhaustive for intake-time exact evidence (adds FR53, FR34, and for direct head extensions FR27, FR51). | FR9, FR16 | PRD FR9 | in force |
| DEC-032 | 2026-10-12 | The FR3 pass validates the `snake_chain` preservation invariants inside the candidate; only obligations whose dropping block precedes the candidate are trusted. | FR3, FR6, FR9 | PRD FR6; epics 6.6, 9.2, 9.3, 9.5 | in force · 9.2, 9.3, 9.5, 6.6 |
| DEC-033 | 2026-10-12 | Radio-originated intake outcomes carry no cause; causes go to the FR64 log; local-interface (FR55) outcomes keep reasons. | FR10, FR55, FR64 | PRD FR10 | in force · 11.2 |
| DEC-034 | 2026-10-12 | A valid block that storage fails to persist is reported to the radio as `Already-known`. | FR10 | PRD FR10 | in force · 11.2 (6.0 keeps `Unstorable` meanwhile) |
| DEC-035 | 2026-10-12 | Each call returns at most one FR10 response component; the rest come through `NextCall`. | FR10 | PRD FR10 | in force |
| DEC-036 | 2026-10-12 | The pacing soft-fail (`Deferred` with `remaining_wait_time`) keeps its wait time: it is data, not a cause. | FR9, FR10 | PRD FR10 | in force · 8.3 |
| DEC-037 | 2026-10-12 | A local failure in the FR3 pass (storage read error, absent vote parameter) deletes nothing; the node stays in Collecting and retries. | FR3, FR5 | PRD FR5; story 6.0 Task 10 | done (6.0; extended by DEC-134) |
| DEC-038 | 2026-10-12 | Ready-state Tier 2 evaluation of side blocks (Stored → Connected) belongs to Story 6.1. | FR9, FR19, FR21 | epics 6.1; DW | done (6.1) |
| DEC-039 | 2026-10-12 | The inbound Tier 3 pacing rule belongs to Story 8.3. | FR9, FR10 | epics 8.3 | in force · 8.3 |
| DEC-040 | 2026-10-12 | Received genesis blocks get bootstrap exceptions only on a sequence **and** content match (all-zero `previous_hash` for #0): new Story 6.10. | FR54, FR69 | epics 6.10 | in force · 6.10 |
| DEC-145 | 2026-10-20 | A ready-state block whose parent is Connected or Active (not the active head) is judged at Tier 2 at intake, before it is stored, and enters Connected directly; a failure is exact evidence and the block is not stored (check first, then write, as DEC-019). Only blocks stored before their Tier 2 could run — orphans connected later, blocks retained while Collecting, local-failure retries — are judged by the side-branch driver after the write. | FR9, FR16 | PRD FR9, FR16; story 6.1 D4 | done (6.1) |
| DEC-147 | 2026-10-20 | Side-block Tier 2 in 6.1 builds the UTXO-input structural completion only (no available-bit check — spentness on a side branch is Tier 3); a reference below `S_tail` within `W` is trusted pre-window history at Tier 2 and in the FR23 replay (DEC-129, DEC-131), which 6.3 applies; registration-key uniqueness is Tier 3 (an active-roster collision is no evidence on a side branch). `mined_amount`, fee and price rules (7.2 / 7.3), the approval threshold (6.7) and the structural `snake_chain` checks (9.2 / 9.3 / 9.5 / 6.6) join one `tier2_check` shared with the head step. | FR9, FR39, ADR-016 | PRD FR9, FR39; story 6.1 D6 | done (6.1) · replay trust 6.3 |

## 2. Block-Tree, Branches, and Retention

| ID | Date | Decision | Affects | Recorded in | Status · Owner |
|---|---|---|---|---|---|
| DEC-041 | 2026-07-04 | Arrival timestamps live on `chain_heads` entries, not on every block entry. | FR18, FR9 | ARCH §6.3; story 4.1 | in force |
| DEC-042 | 2026-07-04 | Block entries are immutable once inserted, except narrow setters (status, active flag, available-UTXO bits, head refs). | FR9, FR23 | story 4.1, 4.2, 5.4; DW | in force |
| DEC-043 | 2026-07-12 | `head_ref_count` counts branches (out-degree toward heads); a fork point gains +1. | FR18, FR19 | PRD FR19; story 4.4; DW | in force |
| DEC-044 | 2026-07-12 | `chain_heads` entries carry `arrival_timestamp` and `missing_parent_hash` (72 B padded). | FR18, FR19 | ARCH §6.3; story 4.4 | in force |
| DEC-045 | 2026-07-12 | Deletion is lazy: storage has no delete; a freed slot is overwritten on reuse. | FR19, storage | story 4.4; ARCH §3.6 | in force (reaffirmed by DEC-056) |
| DEC-046 | 2026-07-12 | Parent recovery is emitted from the tick on exact `NextCall::At` deadlines; the global cooldown stays; the fixed tick interval is removed. | FR19, FR46 | story 4.4; epics | in force |
| DEC-047 | 2026-07-30 | After an FR5 deletion the surviving parent of the deleted root becomes an indexed tip (retargeting a removed entry). | FR5, FR19 | PRD FR19; story 5.5 | in force |
| DEC-048 | 2026-07-30 | Connected → Stored head demotion cannot arise (FR5, FR23). | FR19, FR23 | story 5.5; DW | in force (the tail drop deletes rather than demotes, DEC-125) |
| DEC-049 | 2026-07-30 | A retargeted head keeps the deleted head's arrival timestamp. | FR18 | story 5.5; DW | in force |
| DEC-050 | 2026-08-14 | The const generic is a capacity, `SNAKE_CHAIN_LENGTH_MAX`; the window `W` is chain configuration with `W ≤` capacity at acceptance. | FR2, FR48, FR56 | CFG §6; ARCH §14 row 23 | done (5.11) |
| DEC-051 | 2026-10-08 | FR2 uses the held configuration's `W`; a larger stray `W` only delays acquisition. | FR2, FR8 | story 5.11 | in force |
| DEC-052 | 2026-10-08 | No `VERIFICATION_HORIZON` const generic: `H` is node-level, derived from `W`. | FR58 | ARCH §14 row 24; story 5.11 | in force · 9.7 |
| DEC-053 | 2026-10-09 | R1 — a Ready-state forward-extension driver opens Epic 6 as Story 6.0; order 9.1 → 6.8 → 6.0 → 6.1. | FR9, FR48 | epics (Epic 6 reconciliation); story 9.1 | done (6.0) |
| DEC-054 | 2026-10-09 | Story 9.1 runs before Epic 6. | plan | retro-5; epics | done |
| DEC-055 | 2026-10-09 | R2 — the tail drop releases the block (retention 0 until 9.7), unlinks a surviving child, and can demote a side head to Stored. | FR19, FR48, FR58 | PRD FR19 (v); epics; story 9.1 | in force; demotion part superseded by DEC-125 |
| DEC-056 | 2026-10-09 | R3 — blocks deleted before a restart may come back; no tombstones, no eager erase. | FR48, FR59 | ARCH §3.6; epics; story 9.1 | in force |
| DEC-057 | 2026-10-09 | Epic 6 builds the cheap-zone walk only; zone dispatch and the deep zone go to 9.7. | FR23, FR58 | epics (Epic 6) | in force · 9.7 |
| DEC-058 | 2026-10-09 | The chain-switch walk gets its mempool calls from Story 7.4, not a pre-built slot. | FR23, FR32 | epics (Epic 6) | in force · 7.4 |
| DEC-059 | 2026-10-09 | Accepted interim: between 9.1 and 6.0 the FR60 window is live while the head is frozen. | FR48, FR60 | story 9.1 | done (interim ended by 6.0) |
| DEC-060 | 2026-10-10 | At Ready entry, blocks below the new `S_tail` are released even if obligated; FR48's binding governs extensions, not adoption. | FR2, FR48 | story 9.1 | in force |
| DEC-122 | 2026-10-13 | Ready entry recomputes every `chain_heads` entry, so the adopted chain's head never stays Stored asking for a parent below `S_tail` (resolves O-024). | FR19, FR48, FR60 | story 9.1 (review) | in force |
| DEC-123 | 2026-10-13 | DEC-060 also covers a block whose Ready-entry classification read fails: at adoption it is released; the classification only feeds the AR16 log. | FR48 | story 9.1 (review) | in force |
| DEC-124 | 2026-10-13 | Stored heads whose missing parent lies below `S_tail` (demoted side roots, late tail siblings, restart-resurrected fragments) keep requesting it as specified; the airtime cost is reclaimed by FR57/FR58. | FR19, FR48, FR57, FR58, FR60 | story 9.1 (review); DW | superseded by DEC-125 / DEC-126 (requests remain only between admission and the next tail advance) |
| DEC-125 | 2026-10-17 | The tail drop deletes every branch forked at the dropped block, whole, before releasing it (FR58 reclamation pulled forward from 9.7); no head is demoted to Stored. | FR19, FR48, FR58 | PRD FR19 (v); story 9.1 | in force |
| DEC-126 | 2026-10-17 | A Stored fragment whose lowest block's missing parent is at or below `S_tail` can never connect and is deleted whole by the tail advance that makes it so, or, if admitted later, by the next one (FR58 pulled forward from 9.7). | FR19, FR58, FR60 | PRD FR58; story 9.1 | in force |
| DEC-127 | 2026-10-17 | Ready entry unmarks every block outside the adopted chain and reclaims tail-out-of-range Stored fragments at once (after the `chain_heads` recompute), so a stale Collecting genesis anchor is removed rather than left Connected. | FR4, FR19, FR58 | PRD FR58; story 9.1 (review r2) | in force |
| DEC-061 | 2026-10-10 | Per-block atomicity comes from invertible operations shared by 6.0 and 6.3 (checked arithmetic); no journal or copy. | FR23, FR35 | story 6.0 D1; epics E6-10 | done (6.0) · committed-block undo 6.3 |
| DEC-062 | 2026-10-10 | UTXO resolution is a bounded ancestor walk plus one storage read; the code moves to ADR-016 `(block_sequence, output_index)` inputs (Story 6.8). | FR6, FR14, FR39 | story 6.0 D4; epics 6.8 | done (6.8) |
| DEC-063 | 2026-07-30 | On an FR8 mismatch the candidate's configuration is adopted and the pass re-run in the same call; at most 3 passes. | FR5, FR8 | story 5.9 #1 | in force |
| DEC-064 | 2026-10-12 | A chain switch never disconnects a head (FR23 no longer demotes). | FR19, FR23 | PRD FR23 | in force |
| DEC-065 | 2026-10-12 | FR34 field forms are logical; the architecture's representations (`NONE_REF`, block-index seed source) are equivalent. | FR34 | PRD FR34 | in force |
| DEC-140 | 2026-10-20 | `chain_heads_max_capacity` is a per-node build-time capacity (`MAX_BRANCH_COUNT`, 40 on the reference build), not a chain-config parameter. | FR19 | PRD FR19; ARCH const-generic catalog | in force |
| DEC-142 | 2026-10-19 | Branch value is the sum over `(S_tail, head]` (`S_tail` exclusive); a tail advance subtracts the value of the block that becomes the new `S_tail` (the released block was never in any sum), read in the window gate before any mutation — a failed read is a local failure. | FR19, FR21, FR22 | PRD FR19 (i), FR21 (iii), FR22 (c); story 6.1 D1 | done (6.1) |
| DEC-144 | 2026-10-19 | The active branch's value is one `Blockchain` field, `active_branch_value` (+8 B), because the active head is not always a `chain_heads` tip; a side head's value lives in its entry and is valid iff its head block is Connected; no mirror into the active head's entry. | FR19, FR21 | PRD FR19 (g); ARCH §6.4; story 6.1 D3 | done (6.1) |
| DEC-146 | 2026-10-19 | `active_branch_value` is seeded at Ready entry by the FR3 forward walk, which already reads every block (one window-tail helper shared with the trim); `process_genesis` seeds `consumed_votes(#1)`. No extra read and no new failure mode after the promotion. | FR4, FR21 | story 6.1 D5 | done (6.1) |
| DEC-148 | 2026-10-19 | The side-branch driver runs after every Ready admission (after the continuation) and at every Ready entry, scanning all pending side heads; only a transient local failure leaves a head pending, so no marker state is kept. | FR9, FR19, FR21 | story 6.1 D7 | done (6.1) |
| DEC-149 | 2026-10-20 | FR21 (v): FR5 runs only before any branch value exists; in Ready, an entry that a Tier 2 deletion retargets onto a Connected block is re-valued by the (ii) walk. Side heads are valued by the (ii) walk except a Connected side head's extension, which is incremental (`bv(parent) + v`, no read; DEC-145). | FR19, FR21 | PRD FR21 (v); story 6.1 D4, D8 | done (6.1) |
| DEC-150 | 2026-10-20 | The continuation chooses among stored children of the active head by the FR21 rank at the child level (the child's value, larger block, lowest creator, lowest hash), replacing DEC-021; the comparator is built in 6.1 and reused by 6.2 for heads (creator read only on a value-and-size tie); the admissibility predicate is 6.2's. | FR21, FR22 | ALG Ready phase; story 6.1 D9 | done (6.1) |
| DEC-151 | 2026-10-20 | When any stored child of the active head cannot be read, the continuation stops as a local failure and retries at a later admission; an unreadable sibling is never skipped in favour of a readable one (a node-local fault must not choose the active branch). | FR16, FR21, NFR5 | story 6.1 code review | done (6.1) |

## 3. Creator, Votes, and Approval

| ID | Date | Decision | Affects | Recorded in | Status · Owner |
|---|---|---|---|---|---|
| DEC-066 | 2026-07-03 | Vote arithmetic is checked `u32`; an overflowing step is invalid. | FR37 | PRD FR37; story 3.2 | in force |
| DEC-067 | 2026-07-04 | The vote engine is fully deterministic and has no PRNG. | FR38 | story 3.3; retro-3 | in force |
| DEC-068 | 2026-07-04 | `top_creator()` takes no parameters and scans; deadlines belong to the Epic 8 scheduler; `is_creator_within_rank` added. | FR38, FR44 | ARCH §3.4; story 3.3 | in force |
| DEC-069 | 2026-07-04 | Zero-vote nodes rank at the tail by ascending id, so node 0 is the bootstrap creator. | FR38 | PRD FR38; story 3.3 | in force |
| DEC-070 | 2026-07-30 | `required_support` defaults to 3; a value of 1 is refused as self-approval. | FR56, ADR-015 | story 5.7 #5 | in force |
| DEC-071 | 2026-08-18 | `required_support` is clamped to a chain-declared ceiling `max_aggregated_signatures` (ID 5). | FR56 | CFG (history); story 5.7 | superseded by DEC-075 |
| DEC-072 | 2026-09-19 | The vote engine starts unparameterized and refuses effects until configured. | FR37 | story 5.8 | in force |
| DEC-073 | 2026-10-09 | The proposer keeps at most `required_support` signatures and aggregates once at the threshold, in ascending id order. | FR27 | epics 6.6 | in force · 6.6 |
| DEC-074 | 2026-10-12 | `consumed_votes` is the creator's vote after the block's own interest and credits, before its reset (FR37 wins over FR45). | FR37, FR45 | PRD FR45 | in force · 8.3 |
| DEC-075 | 2026-10-12 | `required_support` is literal-only and refused at acceptance above the backend `MAX_AGGREGATED_SIGNATURES`; `max_aggregated_signatures` is retired. | FR56, ADR-015 | CFG §6 check 3; PRD FR56 | in force · 6.4 |
| DEC-076 | 2026-10-12 | Grace-period firing widens the admitted prefix and never resets votes. | FR44, FR46, FR47 | PRD FR46 | in force · 8.4, 8.5 |
| DEC-077 | 2026-10-12 | Incoming approval-evidence validation gets its own Story 6.7. | FR6, FR9, FR12, FR27 | epics 6.7 | in force · 6.7 |
| DEC-141 | 2026-10-20 | The BLS aggregated support signature is 50 B (`AGGREGATED_SIGNATURE_CONSTANT_SIZE`, as in the crypto library), not 96 B; architecture §6.5 is corrected. | FR27 | ARCH §6.5; crypto algorithm model | in force |
| DEC-143 | 2026-10-19 | Until Story 6.7 validates it, every block's FR21 value is its header `consumed_votes` (an evidence block's is 0 by FR27); the `supporter_vote_sum` accessor and value arm land in 6.7 with its validation, the payload format in 6.6. Until E6-8 and 6.7 every value is an unvalidated header field (recorded risk). | FR21, FR27 | story 6.1 D2; epics 6.7 | in force (interim, built in 6.1) · 6.6, 6.7 |

## 4. Transactions, Value, and Mempool

| ID | Date | Decision | Affects | Recorded in | Status · Owner |
|---|---|---|---|---|---|
| DEC-078 | 2026-06-28 | A registration may carry a `new_public_key` different from the signer's key. | FR6 | story 1.4 | in force |
| DEC-079 | 2026-06-28 | `initial_total_network_currency` is a genesis input, not chain configuration. | FR54, FR56 | PRD FR54; story 1.4 | in force |
| DEC-080 | 2026-06-30 | The mempool is `Mempool<COMPACT_BYTES, MAX_ENTRIES>`, independent of `MAX_NODES`. | FR30 | story 2.1 | in force |
| DEC-081 | 2026-06-30 | A complex transaction is "own" if any balance input or output names the local node; UTXO-only and zero-input ones are not. | FR33 | PRD FR33; story 2.1 | in force |
| DEC-082 | 2026-07-01 | A non-own arriving transaction joins the eviction draw and is rejected if drawn. | FR33 | story 2.2 | in force |
| DEC-083 | 2026-08-18 | An inverted fee range is a diagnostic; the fee-policy consumer resolves it deterministically. | FR56; IDs 25–26 | CFG §4.5 rule 6 | in force · 7.3 |
| DEC-084 | 2026-10-12 | The mining reward is the `mining_reward` parameter (ID 5, `u32`, default 0); `mined_amount` must equal it. | FR6, FR9, FR36, FR45 | PRD; CFG §4.1 | in force · 7.2 |
| DEC-085 | 2026-10-12 | Transactions must comply with the locked fee policy and registration price; carry-forward deducts the locked custodian fee. | FR6, FR9, FR51 | epics 7.3, 9.5 | in force · 7.3, 9.5 |
| DEC-086 | 2026-10-12 | Self-registration exists only at genesis; only the registration price is configurable. | FR6, FR9, FR54 | PRD | in force |
| DEC-087 | 2026-10-12 | The mempool admits Accepted-to-mempool and Deferred transactions (FR31). | FR14, FR31 | PRD FR31 | in force · 7.4 |
| DEC-088 | 2026-10-12 | Mempool capacity is a compile-time parameter, not chain configuration. | FR30, FR31, FR33 | PRD | in force |
| DEC-089 | 2026-10-12 | Eviction frees as much space as FR33 requires (repeated draws for own transactions). | FR31, FR33 | PRD FR31 | in force |
| DEC-090 | 2026-10-12 | The seeded eviction PRNG makes a node replayable from its own seed; no protocol correctness depends on eviction outcomes. | FR30, FR33, FR63 | PRD FR33 | in force |
| DEC-138 | 2026-10-19 | A direct head extension's vote apply runs after the durable write (so a save failure never needs `undo_block`) but before `chain_heads` sees the block; a vote-arithmetic refusal (`u32` overflow, practically unreachable) then deletes the written block before anything links to it, leaving only its bytes in the freed slot — accepted as the one exception to "a failing direct extension is never stored" (D2); no vote-preflight API (moonblokz-vote unchanged). | FR9, FR16 | story 6.0 code review; operating model §7.2 | done (6.0) |
| DEC-137 | 2026-10-18 | The blockchain task's stack is the Core 1 stack and is sized to the measured peak: 24 KB (19.2 KB `process_genesis` + ~25%), superseding the 6 KB task-stack figure; the static RAM margin drops by 18 KB (Schnorr ~28 KB; BLS-medium ~21 KB). Re-measure on material call-graph changes (resolves O-025). | architecture §7.3, §8; system constraints §3 | architecture §8 | done (node `CORE1_STACK`) |
| DEC-136 | 2026-10-18 | Blockchain-task stack budget, first step: a backend read fills one caller-owned buffer (`StorageTrait::read_block_into`; `read_block` is a provided method), and the blockchain's block-sized buffers (signature preimage, UTXO resolver, stored-child step, durable write, FR3 pass, FR5 recovery, FR8 commit / adopt) get their own frames, then re-measure. | architecture §8; storage `StorageTrait` | architecture §8; storage algorithm C1, E4, F7 | done (storage PR #4, blockchain PR #26) |
| DEC-135 | 2026-10-18 | Two balance-block checks follow from FR50 in a fully re-derived projection: every entry's `owner` must be ≤ the block's `max_node_id` (else `NodeIdOutOfRange`), and in a genesis-anchored FR3 pass or in Ready the earliest balance block's `max_node_id` must already equal the tracked watermark (not only later ones). | FR3, FR50 | PRD FR50; story 6.0 D9 | done (6.0) |
| DEC-134 | 2026-10-18 | A mark-overflow or missing-block failure in the FR3 pass is a local failure too (it lies in the node's own index, deeper than the tip): nothing is deleted, the node stays in Collecting and retries; after a missing block, every `parent_ref` naming a freed slot is unlinked, so the block becomes a Stored tail-point that FR19 asks its parent for (resolves O-021). | FR5, FR19 | PRD FR5 | done (6.0) |
| DEC-133 | 2026-10-18 | The FR6 per-block step (FR3 pass; Ready and forward replay through the shared step) checks a block's UTXO-output count against the held configuration's `max_block_utxo_output` even before the durable lock (`TooManyUtxoOutputs` → FR5). Justified like DEC-129: the FR3 preload fails the pass unless the held content is the candidate's own, so the value enforced is the one the lock commits. No lock-time retained-tree sweep, unlike `block_size_limit`: no per-block count is cached, an off-candidate block cannot become Active without the per-block step, and FR57/FR58 reclaim its slot. | FR6, FR9 | PRD FR6, FR9; story 6.11 D3 | done (6.11) |
| DEC-132 | 2026-10-18 | The per-block UTXO bitfield marks available outputs (1 = an unspent output exists) instead of spent ones, so a missing or spent output is rejected from RAM and the tail-drop UTXO obligation needs no read; the block UTXO-output cap (`max_block_utxo_output`, I-07) moves from 7.1 to Tier 1 in the same story, bounded by the bitfield width before the durable lock. Runs after 6.8, before 6.0. | ADR-016; FR9, FR34, FR48, FR51 | PRD FR9, FR34, FR51; ADR-016 amendment; story 6.11 | done (6.11) |
| DEC-131 | 2026-10-18 | A UTXO input declares the referenced output's `amount` (u64), which must equal it. Tier 1 decides `Σ declared inputs ≥ Σ outputs` and the `u64` bound for every complex transaction; the FR6 pass checks each declared amount against the output (`UtxoAmountMismatch`). Chosen over an explicit complex-transaction fee field: it matches FR9's "declared input amounts", and the signature commits to the amount. Confirmed 2026-10-18: a trusted pre-window input's declared amount is trusted on the same grounds as its existence (DEC-129's `W`-long signed segment). | ADR-016; FR3, FR6, FR9, FR36 | PRD FR3, FR9; ADR-016 amendment; story 6.8 | done (6.8) |
| DEC-130 | 2026-10-18 | UTXO inputs reference `(block_sequence, output_index)` instead of a transaction hash for processing performance; the more fragile chain is accepted: after a chain switch a spend whose creating transaction is re-included on the new branch stays valid with a hash reference but not with a sequence reference. | ADR-016; FR6, FR23, FR39 | ADR-016 Decision driver; algorithm §6.1 | in force |
| DEC-129 | 2026-10-17 | On a window-anchored FR3 candidate a UTXO input referencing a block below the anchor is trusted pre-window history if it is at most `W` back (no UTXO bit checked or cleared; the declared amount is trusted, DEC-131); more than `W` back is invalid on every candidate. `W` is the candidate's own preloaded configuration, so the bound applies before the durable lock (an exception to DEC-015). | FR3, FR6, FR14 | PRD FR3; story 6.8 review | done (6.8) |
| DEC-128 | 2026-10-12 | UTXO value accounting moves from 7.1 into 6.8: a UTXO input's referenced amount counts as an input, `Σ inputs ≥ Σ outputs` holds for every complex transaction with an input, and the fee is the surplus over all inputs and outputs. UTXO-input signatures stay in 7.1. | FR6, FR36 | story 6.8 D1 | done (6.8) |

## 5. Configuration and the VM

| ID | Date | Decision | Affects | Recorded in | Status · Owner |
|---|---|---|---|---|---|
| DEC-091 | 2026-07-30 | Configuration is a separate crate; the fixed-value stub retires and its constants become code-baked defaults. | FR56 | story 5.7 #1–#2 | done (5.7, 5.8) |
| DEC-092 | 2026-07-30 | Chain-config envelope: `u16` entry count, entries `{key, length, value}`, then node #0's content signature; the boundary is found by walking. | FR7, FR56 | CFG §3; story 5.7 #4 | in force |
| DEC-093 | 2026-07-30 | The FR7 content signature is a Tier 1 crypto check; bound checks run inside the configuration module on raw declared values. | FR7, FR8, FR16 | story 5.9 #3–#4 | in force |
| DEC-094 | 2026-08-13 | One flat, permanent key registry for every subsystem; key bit 7 selects literal or bytecode; `0xFF` reserved as escape. | FR56 | CFG §3–§4; story 5.7 | in force |
| DEC-095 | 2026-08-13 | Parameters are read through an optional active-configuration handle that also reports tentative vs durable. | FR56 | CFG §5.2; story 5.7 | in force |
| DEC-096 | 2026-08-14 | A full VM with loops in its own crate; caller-supplied fuel; typed outcomes; no load-time verifier; no signed arithmetic. | FR56 | CFG §7; story 5.6 | in force |
| DEC-097 | 2026-08-14 | The configuration specification is ratified and authoritative for the configuration and VM crates. | FR7, FR8, FR17, FR49, FR56 | CFG header | in force |
| DEC-098 | 2026-08-14 | Resolution has three tiers ending in a code-baked literal; the module notifies configuration changes through a required sink; the blockchain, not the module, calls storage. | FR56 | CFG §5, §8, §10; story 5.7, 5.8 | in force |
| DEC-099 | 2026-08-18 | Acceptance runs no program; bounds are checked on declared literals and again at resolution; value forms follow the bound; durations are `u32` ms; `block_size_limit` and radio parameters are literal-only. | FR56 | CFG §4.5, §6 | in force |
| DEC-100 | 2026-08-19 | The chain-info space exists (`GETCHAININFO`); `GETPARAM` is renamed `GETCONFIG`; `max_block_utxo_output` is two bytes wide. | FR34, FR56 | CFG §4.6, §7; story 5.6, 5.7 | in force |
| DEC-101 | 2026-08-31 | Chain-info identifiers are dense and permanent; a chain-info value must be a function of the active chain alone; opcode `0x71` allocated. | FR56 | CFG §4.6 | in force |
| DEC-102 | 2026-08-31 | Chain-info is allocated only when a parameter's arity cannot carry the input. | FR56 | CFG §4.6 (history) | superseded by DEC-106 |
| DEC-103 | 2026-09-02 | The `scoring_matrix` bit layout is permanent; a new layout needs a new identifier. | FR56; ID 17 | CFG §4.2 | in force |
| DEC-104 | 2026-09-03 | Build capacities reach the configuration module as a caller-supplied `BuildLimits` value (replacing copy-plus-assertion). | FR8, FR56 | CFG §6 | in force |
| DEC-105 | 2026-09-19 | With no configuration held, the block-size check falls back to `MAX_BLOCK_SIZE`, timers have no fallback, and parent recovery stays idle. | FR19, FR56 | story 5.8 | in force |
| DEC-106 | 2026-10-09 | Parameters take no arguments; which chain-info a value reads is the bytecode's decision; chain-info allocation is a product decision (ID 1 registered-node count, ID 2 circulating currency in 7.6). | FR56 | CFG §4.5–§4.6; PRD FR56; story 5.12 | in force · 7.6 |
| DEC-107 | 2026-10-09 | Chain-info is served only where its projection is complete; unbound call sites skip the dependent check. | FR3, FR35, FR56 | CFG §4.6 rule 5; story 5.12; epics 7.2 | in force · 7.2 |
| DEC-108 | 2026-10-12 | Chain maximum block size is `block_size_limit` (ID 3); `MAX_BLOCK_SIZE` is only the node's physical buffer width. | FR27, FR33, FR56 | PRD; glossary | in force |
| DEC-109 | 2026-10-12 | ID 5 is reassigned before any chain exists, from `max_aggregated_signatures` to `mining_reward`. | CFG §4.1 | CFG §4.1 | in force · 6.4, 7.2 |

## 6. Storage, Restart, Keys, and Public API

| ID | Date | Decision | Affects | Recorded in | Status · Owner |
|---|---|---|---|---|---|
| DEC-110 | 2026-06-24 | Transactions use per-kind owned types, a `TransactionView`, and builder methods instead of one owned wrapper. | chain-types | story 1.1 | in force |
| DEC-111 | 2026-06-28 | Signing is mandatory (`build_signed` only); exactly one crypto backend must be selected. | FR68 | story 1.4 | in force |
| DEC-112 | 2026-07-21 | One constructor (`init`) for every node; genesis is `process_genesis` building #0 and #1 in one call; join and restart share `initialize_from_storage`. | FR54, FR59 | ARCH §3.6; story 1.4, 5.1, 5.10 | in force |
| DEC-113 | 2026-07-22 | Read-only lookups are named `query_*`; creator-role determination is not public. | FR40–FR44 | ARCH §3.1; DW | in force |
| DEC-114 | 2026-09-20 | Restart recomputes each block's exact length, re-admits readable deleted blocks, and reports `ResumedCollecting`. | FR59 | story 5.10 | in force |
| DEC-115 | 2026-09-21 | On restart a stored configuration passes the FR7 check before the lock is restored, and the lock-time cleanups re-run; after a refused restart intake answers `Unstorable` until re-initialization. | FR7, FR8, FR59 | story 5.10 | in force |
| DEC-116 | 2026-10-10 | The BMAD `prd.md` and `architecture.md` are symbolic links to the KB documents. | PRD, ARCH | PRD / ARCH Provenance | done |
| DEC-117 | 2026-10-12 | The blockchain module never receives the private key: `load_control_data()` drops it, `load_private_key()` serves the runtime. | FR59, FR68; ST-FR11, ST-FR54 | ST-PRD; PRD FR59 | in force · 6.9 |

## 7. Process and Knowledge Base

| ID | Date | Decision | Affects | Recorded in | Status · Owner |
|---|---|---|---|---|---|
| DEC-118 | 2026-07-21 | A standalone node build-fix story is to be drafted, parallel to Epic 5. | moonblokz-node | retro-4 | open — no story exists (O-020) |
| DEC-119 | 2026-10-09 | Every epic starts with a reconciliation pass; every open deferred item gets an owner, a chore, or an explicit won't-do. | process | retro-5; DW | in force |
| DEC-120 | 2026-10-12 | The operating model and FR digest are kept current with every behavior change and every completed story. | KB, BMAD workflows | [AGENTS.md](./AGENTS.md) | in force |
| DEC-121 | 2026-10-12 | Every Project Lead decision is recorded in this register in the same change that applies it. | KB, BMAD workflows | [AGENTS.md](./AGENTS.md) | in force |
| DEC-139 | 2026-10-19 | Design-decision discipline: a proposed implementation choice is checked against the ratified principles, keeps any specified incremental mechanism, and prefers event-time information to new stored state; the proposal names what it checked. | All repos, BMAD workflows | [System Constraints](./moonblokz-system-constraints.md) §0.1 | in force |

## 8. Open Decisions

| ID | Since | Question — recommendation | Affects | Recorded in | Decide at |
|---|---|---|---|---|---|
| O-001 | 2026-10-09 | Approval-evidence payload layout — rec.: LE `supporter_vote_sum u64 ‖ deviation_sequence u32 ‖ n u8 ‖ ascending ids ‖ aggregated signature`. | FR21, FR27 | epics E6-1; DEC-143 | 6.6 (format) / 6.7 (value accessor) |
| O-002 | 2026-10-09 | What a support message signs — rec.: `"MBSUP" ‖ proposer ‖ deviation_seq ‖ hash`; radio `Support` 113 B. | FR12, FR26 | epics E6-2 | 6.4 |
| O-003 | 2026-10-09 | Where `required_support` and the replay delay are bound — rec.: at `proposed_sequence − 1`. | FR12, FR27 | epics E6-3 | 6.4 / 6.6 |
| O-004 | 2026-10-09 | Accumulator memory — rec.: accept ≈3.4 KB for Schnorr and correct architecture §6.5. | FR27, NFR1 | epics E6-4 | 6.6 |
| O-005 | 2026-10-09 | Non-conflict record — rec.: fixed ring of 8 entries, not durable. | FR12 | epics E6-5 | 6.4 |
| O-006 | 2026-10-09 | Support relay de-duplication — rec.: LRU of 32 × 12 B. | FR26 | epics E6-6 | 6.5 |
| O-007 | 2026-10-09 | Enforce "deviation `consumed_votes` strictly lower" as a Tier 3 invariant — rec.: yes, with O-001. | FR28 | epics E6-8 | 6.2 |
| O-008 | 2026-10-09 | Branch tie-break source — rec.: keep FR21 (size → creator → hash), size from the block entry. | FR21 | epics E6-9 | decided → DEC-150 |
| O-009 | 2026-10-09 | Balance-block seeding inverse — rec.: identity, by validating snapshot = pre-state (see DEC-025). | FR23, FR50 | epics E6-11 | 6.3 |
| O-010 | 2026-10-09 | Forward-walk failure during a switch — rec.: reverse in place to the original chain, then a Ready-safe deletion. | FR23, FR24 | epics E6-12 | 6.3 |
| O-011 | 2026-10-09 | Deep-zone candidates before 9.7 — rec.: no switch (interim). | FR23, FR58 | epics E6-13 | 6.3 |
| O-012 | 2026-10-09 | FR12 support trigger — rec.: only for direct active-head extensions; grace rank via a seam until 8.5. | FR12 | epics E6-14 | 6.4 |
| O-013 | 2026-10-08 | A lower config-carrying candidate wins at once over a higher one without configuration — the chain switch must cover it. | FR2, FR21–FR23 | story 5.13 | Epic 6 |
| O-014 | 2026-10-09 | FR50 replay sub-seed in AR11 contradicts the canonical FR50 ordering. | FR50, FR63 | epics (Epic 9) | 9.3 |
| O-015 | 2026-06-20 | Scoring getter (architecture §13 task 7) vs FR55's pre-populated `vote` — deferred, unresolved. | FR55 | epics AR17 | — |
| O-016 | 2026-07-24 | Can a block ever exceed `u16::MAX` bytes — rec.: no, close as `u16`. | chain-types, storage | DW triage D37 | — |
| O-017 | 2026-08-18 | `Cargo.lock` policy — rec.: commit it in binary-shipping repositories. | workspace | DW triage D45 | — |
| O-018 | 2026-08-18 | `#![forbid(unsafe_code)]` policy — rec.: add to chain-types and configuration. | workspace | DW triage D46 | — |
| O-019 | 2026-10-09 | VM CI size gate (hidden mnemonic cost) — rec.: an acceptance criterion in 11.5. | moonblokz-vm | DW triage D1 | 11.5 |
| O-020 | 2026-10-09 | Firmware has no `init` call site — rec.: a node-integration story once the node repository is ready (see DEC-118). | moonblokz-node | DW triage D11 | — |
| O-021 | 2026-10-12 | Does DEC-037 (no deletion on local failure) also cover mark-overflow and missing-block failures in the pass? | FR5 | DEC-013, DEC-037 | decided → DEC-134 |
| O-022 | 2026-10-09 | BLS Miri script never run — rec.: won't-do while BLS is not an embedded target. | crypto | DW triage D12 | — |
| O-023 | 2026-10-20 | `required_support` default (3, so `m = 5`) is too low: set it per backend — Schnorr by evidence-block capacity, BLS by security and practicality — rec.: none yet; measure support-message radio cost in the radio simulator. | FR12, FR27; CFG §4.1 ID 10 | [Algorithmic Risk Register](./moonblokz-algorithmic-risk-register.md) AR-007 | — |
| O-023 | 2026-10-09 | Up to 4 boot sweeps, O(`MAX_BLOCKS`²) tip discovery — rec.: won't-do until boot latency is measured. | FR59 | DW triage D51 | — |
| O-025 | 2026-10-18 | The measured blockchain-task stack (11.5–19.2 KB, DEC-136) exceeds architecture §8's 6 KB task-stack decision: raise the task stack to the measured peak, or cut further (verify the creator signature in place in the owned read buffer instead of a 2 KB preimage; slimmer genesis assembly; the by-value `ControlPlaneData` at restart)? | architecture §8 | DEC-136 | decided → DEC-137 |
| O-026 | 2026-10-20 | A local copy that fails its integrity check (`StorageError::IntegrityFailure`: the slot does not match its own hash, so it is permanently unusable — unlike a possibly transient `BackendIo`) is today a local failure that deletes nothing (DEC-037, DEC-134), so its block can stall the FR3 pass or a side head forever. Rec.: release the slot (lazy delete) and let parent recovery and the mesh re-fetch the block — a new authorized deletion path (FR9, FR19), applied uniformly at every read site (FR3 pass, continuation, side walk, UTXO resolver, tail pre-read); `view_retained` has to tell the two errors apart. Would resolve the 6.0 and 6.1 code-review deferrals on permanently unreadable blocks. | FR5, FR9, FR16, FR19 | story 6.1 code review; DW | next epic-start reconciliation |

## Related Documents

- [Blockchain PRD](./moonblokz-blockchain-prd.md) — the Provenance section records each requirement change; the FRs hold the rules.
- [Blockchain Operating Model](./moonblokz-blockchain-operating-model.md) — the current behavior these decisions produce, with build state.
- [Configuration Module Specification](./moonblokz-configuration-specification.md) — configuration and VM decisions in context.
- [Blockchain Architecture](./moonblokz-blockchain-architecture.md) §14 and the [ADR index](./blockchain-adrs/ADR-INDEX.md) — the architecture decision logs this register does not repeat.
- [Open Gaps Register](./moonblokz-open-gaps-register.md) — specification gaps and post-MVP questions, as opposed to pending decisions.
