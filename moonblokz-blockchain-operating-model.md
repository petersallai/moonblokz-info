# MoonBlokz Blockchain Operating Model

One-read picture of how the `moonblokz-blockchain` module behaves at runtime: which lifecycle phase does what with each event, and how a block moves through its statuses. It is a **consolidation, not a source**: every row cites the requirement that defines it, and the [Blockchain PRD](./moonblokz-blockchain-prd.md) wins on any divergence. Read the cited FR before relying on a detail.

The **Built** columns record implementation state — **built**, **partial**, or **not built** — with story numbers from the BMAD plan (`_bmad-output/planning-artifacts/epics.md`): a number inside `built (…)` or after `built in` names the story that delivered that part; any other number names the story that owns the remaining work. They are kept current under [`AGENTS.md`](./AGENTS.md) (Operating Model Maintenance); last full review 2026-10-12 against `moonblokz-blockchain` `1496b76`.

Sections: 1 execution model · 2 lifecycle transitions · 3 event × phase matrix · 4 block statuses · 5 invariant catalog · 6 derived projections · 7 main flows.

## 1. Execution Model

- The module is a single-threaded synchronous core driven by calls: intake (`receive_block`, `receive_transaction`, `receive_support`, `submit_local_transaction`), `on_tick`, read-only queries, and the boot calls `init` / `process_genesis` / `initialize_from_storage` ([architecture](./moonblokz-blockchain-architecture.md) §1.3, §3; [ADR-006](./blockchain-adrs/ADR-006-moonblokz-blockchain-uses-a-single-threaded-synchronous-core.md)).
- **Processing is never observable between calls.** The FR3 pass runs to completion inside the `receive_block` or `initialize_from_storage` call that triggered it. During an FR59 restart rebuild the module accepts no intake at all.
- Chain configuration has three commitment states — absent, tentative (RAM only, re-derived on restart), durable (set once) — owned by the configuration module ([Configuration Module Specification](./moonblokz-configuration-specification.md) §8).

## 2. Lifecycle Phases and Transitions

Two resting phases, **Collecting** and **Ready**, joined by the transient **Processing** pass (FR1–FR5). Node #0 skips Collecting entirely (FR54).

| # | From → To | Trigger | What runs | On failure | FRs | Built |
|---|---|---|---|---|---|---|
| T0 | boot → Collecting | `init`, then `initialize_from_storage`: empty storage = join, populated = restart | Restart: rebuild the block-tree and `chain_heads` from durable blocks, load a durable configuration if one exists, evaluate FR2 | Restart reports `Rejected` (control plane unreadable / configuration unusable) | FR1, FR59 | built (5.1, 5.10) |
| T1 | boot → Ready | `process_genesis` on node #0 | Create blocks #0 and #1, durably lock the configuration, take #0/#1 as the starting active chain; no FR2 / FR3 | Refused off node #0 or on non-empty storage; a refusal of #0 / #1 by the per-block step is `InitialChainConfigRejected` | FR54 | built (1.4, 5.1, 6.0) — #0 / #1 run the shared per-block step in the complete context and are Active |
| T2 | Collecting → Processing | A segment satisfies FR2: genesis-anchored or at least `W + H` long (DEC-177), and containing a chain-config block; highest tip sequence wins, then lowest tip hash | FR3 pass: backward marking, candidate chain-config preload, forward traversal with per-node seeding | — | FR2, FR3 | built (5.2, 5.3, 5.13) |
| T3 | Processing → Ready | The FR6 re-execution passes and the FR8 final check matches | Durable set-once configuration lock; mismatching chain-config blocks deleted with descendants; the candidate becomes Active | — | FR4, FR6, FR8 | partial — lock, cleanup and promotion built (5.4, 5.9); several FR6 checks missing (§5: I-08–I-10, I-15, I-16, I-33) |
| T4 | Processing → Collecting | Any FR6 invariant violation | FR5 atomic recovery: discard the working set, delete the earliest offending block (or the candidate head if unidentifiable) and its descendants, re-evaluate FR2 | — | FR5 | built (5.5) |
| T5 | Processing → Processing | Candidate configuration differs from the tentative one | Replace the tentative configuration with the candidate's (re-checked per FR7) and retry, or roll back per FR5 | — | FR8 | built (5.9) |
| T6 | within Collecting | First chain-config block passing FR7 and accepted by the configuration module | Tentative load; later chain-config blocks are stored but do not override it | Module rejection is exact evidence; no load | FR8 | built (5.9) |
| T7 | Ready → Processing → Ready | Withdrawn (DEC-159): there is no deep-zone switch, so Ready never re-enters Processing; a branch beyond FR58's switch bound is reclaimed (§7.3, §7.4) | — | — | FR58 | — |
| T8 | Processing interrupted | Restart mid-pass | Partial state discarded; T0 restart path | — | FR3, FR59 | built (5.10) |

## 3. Event × Phase Matrix

Node #0 after genesis behaves as Ready. "Not-ready" is the FR1 indication returned to every caller.

| Event | Collecting | Processing | Ready | Built (Ready path) |
|---|---|---|---|---|
| Inbound block | Outcomes reduce to Accepted-new / Already-known / Invalid-evidence (no window check); Tier 1 only, so every kept block stays Stored; the first valid chain-config loads tentatively; FR2 re-evaluated (FR1, FR8, FR9, FR60) | Not separately specified for external intake; the pass itself runs Tier 1 + 2 + 3 atomically on the candidate (FR9) | Exactly one FR10 outcome: Accepted-new, Already-known, Invalid-evidence, Out-of-Snake-Chain-Window, Deferred; may carry a support message (FR12), a re-disseminated transaction (FR13), or a new block (FR45) — one per call, the rest through `NextCall`; no cause is returned — it goes to the FR64 log | partial — Tier 1 and storage built (4.2, 4.3); window check built (9.1); Tier 2 + 3 and forward extension of the active head built (6.0); side-block Tier 2 built (6.1); chain-switch Tier 3 6.3; response components 6.4, 7.5, 8.3 |
| Inbound transaction | Not-ready; no classification, no mempool admission (FR14) | Not-ready (FR10) | Exactly one FR14 outcome: Accepted-to-mempool, Already-confirmed, Duplicate-in-mempool, Deferred, Rejected | not built (7.3) |
| Local transaction submit | `Rejected(not-ready)` (FR55) | `Rejected(not-ready)` | Created, Held(reason), or Rejected(reason) (FR55) | not built (10.5) |
| Inbound support message | Accepted on format validity and LRU de-duplication alone; state-dependent checks deferred (FR26) | Same as Collecting (FR26) | Full FR26 checks; an Accepted message is accumulated by the proposer (FR27) | not built (6.5, 6.6) |
| Inbound twin vote | Relayed under FR26; conflict noted | Same as Collecting | Relay; note the conflict; a subgroup member votes once for its first-adopted twin, and the entitled creator collects votes for the evidence block (FR71) | not built (6.13) |
| Tick — parent recovery | At most one request per tick, toward the Stored head whose retry interval elapsed longest ago (FR1, FR19) | Not specified | Same as Collecting (FR19 has no phase limit) | built (4.4) |
| Tick — mempool replenishment | Not scheduled (FR46) | Not scheduled (FR46) | Runs on every node whose mempool is below the fill threshold, creator or not (FR46) | not built (8.4; responder 10.4) |
| Tick — block creation | Not-ready (FR1 gates FR44) | Not specified | If the local node is the current creator: create when the fill threshold is reached, or on the timer with at least one includable transaction; replay obligations go out as soon as the node is eligible (FR45, FR46) | not built (8.3, 8.4; replay 9.2–9.5) |
| Tick — grace period | Not specified | Not specified | Widen the admitted creator set by one; no accumulated-vote reset (FR47) | not built (8.5) |
| Queries | Not-ready (FR1, FR43) | Not-ready (FR42, FR43); FR40 / FR41 answers are not observable before Ready (FR8) | Transaction state Unknown / In-mempool / Confirmed with depth and the disputed flag (FR40, FR70); balance and accumulated vote, address UTXOs (FR41); active-chain blocks only, else not-available (FR42); at most one transaction (FR43) | not built (10.1–10.4; not-ready gates built in 5.1) |
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
| 1 | Stored (durable storage) | Intake, every phase; items marked *ready* only in Ready | version and payload schema; header vote form (DEC-157); *locked (any phase):* size ≤ `block_size_limit`, chain-config content equals the lock (FR17); UTXO-output count ≤ `max_block_utxo_output` (bitfield width before the lock); *ready:* window membership (FR60); `block.sequence > anchor_sequence`; *locked (any phase):* `anchor_sequence ≥ max(0, block.sequence − W)` (the FR3 pass checks it before the lock; DEC-174); no self-vote; registration / complex-transaction exclusivity; `new_key_signature`; node-#0 trust-anchor checks (FR69); *ready:* creator signature when the block directly extends the active head or its creator's key is branch-independent (node #0, or seeded at or below `S_tail`; DEC-157) |
| 2 | Connected | Ready (Processing: inside the FR3 pass) | ancestry reaches the active chain; locked chain-config rules at the block's sequence (fee policy, registration price, custodian fee, approval threshold); `mined_amount` equals `mining_reward`; UTXO-input structural completion (a reference below `S_tail` within `W` is trusted, DEC-129); structural `snake_chain` replay checks where the reference is retained (else deferred to Tier 3). Registration-key uniqueness is Tier 3 (a side branch may legitimately register a key the active chain registered after the fork). A block whose parent is Connected or Active (not the head) is judged before storing |
| 3 | Active | The FR3 pass, FR23 chain-switch forward replay, and in Ready a direct extension of the active head or an already-stored child of it (FR16, FR9) | vote fields against candidate vote state; creator balance; balance-block contents and `max_node_id`; transaction signatures, balances, duplicates, node-id stride, available-UTXO bits, no negative balance; carry-forward validity; approval-evidence cluster; pacing (soft-fail to Deferred) |

Checks derived from chain configuration can invalidate a block only after the configuration is durably locked (FR8, FR54). Under a tentative configuration a failure does not delete the block; it is re-evaluated when the tentative configuration changes or the lock happens (FR9). Two exceptions are enforced in the FR3 pass against the candidate's own preloaded configuration: the UTXO `W` distance bound (DEC-129) and the per-block UTXO-output cap (DEC-133).

### 4.3 Transitions

| From → To | Trigger | FRs | Built |
|---|---|---|---|
| intake → Stored | Tier 1 passes | FR9, FR16 | built (4.2) |
| intake → not stored | Tier 1 exact evidence (Invalid-evidence); a Ready direct extension of the active head failing Tier 2 / 3 (Invalid-evidence); outside the window in Ready; chain-config content ≠ lock (silent); duplicate (no-op); a valid block storage fails to persist (reported as Already-known, cause logged) | FR10, FR11, FR16, FR17, FR60 | built (4.2, 4.3); window check built (9.1); head-extension rejection built (6.0) |
| Stored → Connected | Ancestry reaches the active chain and Tier 2 passes; a new block whose parent is Connected or Active (not the head) is judged before storing and a failure is not stored | FR9, FR16 | built (6.1); further Tier 2 checks 6.6, 6.7, 7.2, 7.3, 9.2, 9.3, 9.5 |
| Stored → Active | A direct extension of the active head (checked before storing; a failure is not stored), or an already-stored child of the active head, passes Tier 2 and Tier 3 in one step | FR9, FR16 | built (6.0) |
| Connected → Active | FR23 reconciliation succeeds | FR9 | not built (6.3) |
| Stored → Active | FR3 pass succeeds (collecting → ready only) | FR3, FR9 | built (5.4) |
| Active → Connected | FR23 demotes the block's branch | FR9, FR23 | not built (6.3) |
| Stored / Connected stays | Tier 3 pacing soft-fail → Deferred, no deletion | FR9, FR10 | not built (8.3) |
| → deleted (Invalid) | Tier 1 failure or FR8 lock mismatch (from Stored); Tier 2 failure (from Stored); Tier 3 failure — via FR5 in the FR3 pass, via FR23, or for a stored child of the active head, always with descendants; FR5 rollback and phase reversion apply to the FR3 pass only | FR5, FR8, FR9 | partial — FR3-pass and FR8 paths built (5.5, 5.9); stored child of the active head built (6.0); side-block Tier 2 built (6.1); FR23 6.3 |
| non-active → deleted | `chain_heads` overflow: evict the lowest non-active head, delete blocks whose reference count reaches 0 | FR19 | built (4.4) |
| side branch → deleted | Storage pressure: exactly one lowest-value side branch; never active-chain or verification-horizon blocks | FR57, FR9 | not built (9.6) |
| side branch → deleted | Its divergence point left FR58's switch bound (`D ≤ S_head − H`), or it is a Stored fragment whose missing parent lies at `q ≤ S_head − H − 1` (incl. a stale genesis anchor the adopted chain does not contain): deleted whole at Ready entry or at the head extension or switch that moves it out | FR58, FR19 (iv) | partial — built (9.1) with the earlier `S_tail` criterion: forks at the dropped block at once, Stored fragments whose missing parent is at or below `S_tail` at Ready entry and the next tail advance; the `S_head − H` criterion for both 9.7 (DEC-159, DEC-161) |
| Active tail → dropped | Head extension past `W` (essential state replayed in that head, FR48): the block enters the verification horizon; the block leaving the horizon has its forks deleted (FR58), is released, and the new lowest retained block is unlinked | FR19 (v), FR48, FR58 | partial — release, branch reclamation and unlink built (9.1) with no horizon; triggered by the head extension built (6.0); horizon retention 9.7; replay content 9.2, 9.3, 9.5 |

Durable storage keeps only the node id, the private key, the chain-config block with its lock flag, and the retained blocks — no phase marker, no projections (FR59).

## 5. Invariant Catalog

What a valid chain must satisfy, grouped by scope. **Checked** names where the rule is enforced: T1 / T2 / T3 are the §4.2 tiers, *pass* is the FR3 processing pass (FR6 re-execution), *accept* is configuration-module acceptance. Rules decidable from a block's own bytes are Tier 1 only; the pass does not re-check them (FR6). The cited FRs hold the full rule.

### 5.1 Block Structure

| # | Invariant | FRs | Checked | Built |
|---|---|---|---|---|
| I-01 | Supported header version, known `payload_type`, payload parses with declared counts matching the bytes | FR6, FR9, FR16 | T1 | built (4.2) |
| I-02 | Block size ≤ `block_size_limit` once the configuration is durably locked; before that only the transport / storage ceiling applies | FR6, FR9, FR54 | T1 (locked, any phase); pass | built (4.2, 5.4, 5.9) |
| I-03 | `sequence` is the parent's + 1 and `previous_hash` links to the parent | FR6, FR53 | admission; pass | built (4.4, 5.4) |
| I-04 | No block has `sequence == u32::MAX`; creation stops at `u32::MAX − 1` | FR53 | T1; creator side | partial — intake built; creation ceiling 8.1 |
| I-05 | Creator signature valid against the chain-derived key (node #0: the trust anchor); a block verified once is marked and not re-verified while the node runs (RAM-only mark; DEC-157); a chain switch first verifies every unmarked new-branch block whose creator is fork-independent at the common ancestor (DEC-170) | FR6, FR9, FR16, FR54 | T1 in Ready for a direct head extension and for a branch-independent key; switch pre-pass; T3; pass | partial — pass built (5.4); head extension built (6.0); branch-independent intake check and verified mark built (6.2); chain-switch replay 6.3; switch pre-pass 6.3 |
| I-06 | A transaction block never holds both a registration and a complex transaction (carry-forward included) | FR6, FR9, FR51 | T1 | built (4.2) |
| I-07 | A block's UTXO-output count ≤ `max_block_utxo_output` | FR6, FR9, FR34, FR56 | T1 (locked; bitfield width before the lock, DEC-132); pass (held configuration, also before the lock, DEC-133) | built (6.11) |
| I-42 | The creator signature signs the header with the signature zeroed followed by `H(payload)`, so a header plus payload hash verifies it | FR9, FR70 | T1; T3; pass | not built (6.12) |

### 5.2 Creator, Votes, and Approval

| # | Invariant | FRs | Checked | Built |
|---|---|---|---|---|
| I-08 | `mined_amount` equals `mining_reward` in scope (genesis exempt) | FR6, FR9, FR36, FR56 | T2; pass | not built (7.2) — the pass credits the header value unchecked |
| I-09 | `consumed_votes`, `first_voted_node`, `consumed_votes_from_first_voted_node` match the derived vote state: post-interest, pre-credit snapshots of the creator and, for a deviation, of the creator-order top (DEC-156), so a deviation block is never higher than the normal block it replaces | FR6, FR9, FR21, FR37 | T1 (header form); T3; pass | built (6.2) |
| I-10 | The creator is entitled: FR38 top, an admitted grace-period fallback, or the deviation proposer | FR6, FR38, FR44, FR27 | T3; pass | not built (Epic 8) |
| I-11 | Above a deviation block only its creator extends the branch, with `consumed_votes = 0` and `first_voted_node = self`; intervening replays follow the tail fall-out order | FR9, FR16, FR27, FR29 | T1 for a direct head extension; T2; pass | not built (6.6; incoming evidence 6.7) |
| I-12 | Every `vote` names an existing node; `vote == 0` is always valid; at transaction intake an unknown target is Deferred, not invalid | FR6, FR14, FR37, FR54 | T3; pass | partial — pass built (5.4); head extension built (6.0); chain-switch replay 6.3 |
| I-13 | No self-vote, except node #0 voting 0 | FR6, FR9, FR54 | T1 | built (4.2) |
| I-14 | Accumulated-vote arithmetic never overflows `u32`; an overflowing step is invalid | FR23, FR37 | vote step (pass, T3) | built (3.1) |
| I-15 | Approval evidence: subgroup members, valid aggregate signature, count ≥ `r_eff` (FR12), size ≤ `MAX_AGGREGATED_SIGNATURES`, `supporter_vote_sum` per formula | FR6, FR9, FR12, FR21, ADR-015 | T3; pass | not built — the pass skips evidence blocks; 6.7 |
| I-43 | Twin evidence: the proof header has the creator `c`, sequence `s`, parent `P` and a valid FR70 signature, differs from the branch's own block at `s` (which has creator `c` and parent `P`); `s ≥ S_tail`; one per `(c, s)` in the window; a verdict has no bit at a position `≥ m`, at least `r_eff` bits, and a valid aggregate | FR6, FR9, FR71 | T3; pass | not built (6.13) |
| I-44 | A node under an FR72 exclusion (a twin evidence block for it in the window) is not an entitled creator and is not in `A`; the evidence block resets its accumulated vote, and it stays 0 through the excluded blocks `e + 1 … e + W` (no credits; DEC-175) | FR38, FR72 | T3; pass | not built (6.13) |

### 5.3 Value and Balances

| # | Invariant | FRs | Checked | Built |
|---|---|---|---|---|
| I-16 | The creator is credited `mined_amount` + fees + `replay_block_reward` (if the block carries replay) once per block, and the resulting balance is consistent; #0 / #1 exempt | FR6, FR36, FR54 | T3; pass | partial — `mined_amount` + fees credit built (5.3); replay reward and the consistency check 7.2, Epic 9 |
| I-17 | No negative balance: initializers cover amount / price plus fee and every balance input | FR6, FR9 | T3; pass | partial — seeded nodes built (5.4); checked credits (`BalanceOverflow`) and the head extension built (6.0); chain-switch replay 6.3 |
| I-18 | `registration_price` is absorbed by the network; the fee goes to the creator | FR6, FR36 | T3; pass | partial — debit and credit built (5.4); price check against configuration 7.3 (I-36) |
| I-19 | A complex transaction with inputs has Σ inputs ≥ Σ outputs (zero-input carry-forward exempt) | FR6, FR9 | T1 (declared amounts, DEC-131); T2 / pass (declared UTXO amount = referenced output) | partial — balance inputs built (4.2, 5.4), UTXO values built (6.8), both in the FR3 pass; Ready head extension built (6.0); chain-switch replay 6.3 |

### 5.4 Transactions by Type

| # | Invariant | FRs | Checked | Built |
|---|---|---|---|---|
| I-20 | `block.sequence > anchor_sequence` for node transfers and complex balance inputs (block #0 waived) | FR6, FR9, FR54 | T1 | built (4.2) |
| I-45 | `anchor_sequence ≥ max(0, block.sequence − W)` for node transfers and complex balance inputs (the parent's window tail), so the duplicate check over the window is complete (DEC-174) | FR6, FR9, FR14 | T1; pass | not built (7.3) |
| I-46 | At block `s ≥ H`, after FR37 steps 1–3, the creator of block `s − H` gains `⌈vote_scale / vote_interest⌉` (not the creator of `s`, not FR72-excluded or named by a twin evidence block `s`, `vote_interest > 0`) | FR37, DEC-178 | T3; pass | not built (7.2) |
| I-47 | A window-anchored candidate is at least `W + H` long, and a node's accumulated vote in it is seeded from its first seed source at or after the earliest block + `H` | FR2, FR3, DEC-177 | pass | not built (9.7) |
| I-21 | Transaction and input signatures valid (UTXO inputs against the referenced output's address) | FR3, FR6, FR9 | T3 (T1 opportunistic); pass | partial — transfers and registrations built (5.4); balance and UTXO inputs 7.1 |
| I-22 | No duplicate of a confirmed transaction (transfer key, complex input set) | FR6, FR9, FR14 | T3; pass | not built for balance-side keys (ratified deferral to Epic 7); UTXO side covered by available-UTXO bits |
| I-23 | Every referenced node id lies within `[0, max_known_node_id]` | FR6 | T3; pass | built (5.4) |
| I-24 | A registration's `new_public_key` is globally unique, also within a block | FR6, FR9, FR14 | T3; pass | partial — pass built (5.4); head extension built (6.0); side branches at the chain-switch replay 6.3 (not Tier 2: an active-roster collision is no evidence on a side branch, DEC-147) |
| I-25 | `new_key_signature` proves possession of the new key | FR6, FR9 | T1 | built (4.2) |
| I-26 | `new_node_id = max_known_node_id + 1`, stride 1 within a block; block #0 registers id 0 without incrementing | FR6, FR9, FR54 | T3; pass | built (5.4) — explicit #0 check missing |

### 5.5 UTXO

| # | Invariant | FRs | Checked | Built |
|---|---|---|---|---|
| I-27 | A UTXO input references `(block_sequence, output_index)` of an ancestor at most `W` back, index within its output stream; below `S_tail` (or a window-anchored candidate's anchor) the reference is trusted pre-window history (DEC-129) | ADR-016, FR3, FR6, FR9, FR14 | T1 (causality); T2; pass | partial — Tier-1 causality, the `W` distance bound, the window-anchored pre-anchor trust (DEC-129) and the FR3-pass resolver (ancestor walk plus one read) built (6.8); available-UTXO bit checked before the read, so a spent or missing output costs no read, built (6.11); Ready head extension built (6.0); side-block Tier 2 with the below-`S_tail` trust built (6.1); the chain-switch replay's trust 6.3 |
| I-28 | The referenced output is unspent at inclusion and spent once | ADR-016, FR6, FR9 | T3; pass | partial — pass built (5.4); head extension and the exact spend inverse built (6.0); chain-switch replay 6.3 |

### 5.6 `snake_chain` Preservation

Checked at head extension in Ready, and inside the FR3 pass against the candidate's own projections wherever the context lies inside the candidate; only obligations whose dropping block precedes the candidate are trusted (FR6, FR9).

| # | Invariant | FRs | Checked | Built |
|---|---|---|---|---|
| I-29 | Active-chain length ≤ `W` (before maturity: block #0 to the head) | FR48 | head extension | built (9.1, 6.0) — at Ready entry (9.1) and on head extension (6.0) |
| I-30 | The tail never drops unpreserved state: a head extension at a drop is (or carries) the replay of the dropping block's one payload category | FR45, FR48 | T2; creator side | partial — classification and kind binding built (9.1); enforced on inbound head extensions built (6.0); creator side 8.3; content 9.2, 9.3, 9.5 |
| I-31 | A dropping chain-config block is replayed in the next head with identical content | FR49 | T2; pass | not built (9.2) |
| I-32 | Every registered node keeps a seed source (balance entry or registration) inside every `W` window | FR50 | T2; pass | not built (9.3) |
| I-33 | A balance block's entries equal the state at `seq − 1`, `max_node_id` equals the watermark, and the entry set is the canonical mandatory set plus bonus pack | FR9, FR50 | T3; pass | partial — `max_node_id` built (5.4; from the first balance block of a genesis-anchored pass, 6.0); entries equal to the pre-state and owners ≤ `max_node_id` built (6.0, D9); canonical set and bonus pack 9.3 |
| I-34 | Carry-forward: exactly one tail block drops, it holds UTXO outputs, outputs equal `(recipient, amount − custodian_fee)` for every unspent output, below-fee outputs are dropped, nothing extra, no registration alongside, vote-neutral | FR9, FR51 | T1 for a direct head extension; T2; T3; pass | not built (9.5) |

### 5.7 Configuration

| # | Invariant | FRs | Checked | Built |
|---|---|---|---|---|
| I-35 | Every chain-config block is byte-identical to the durable lock | FR3, FR6, FR8, FR17 | T1 (silent discard after the lock); pass; FR8 cleanup | built (4.3, 5.9) |
| I-36 | Blocks comply with the locked configuration in scope (fee policy, registration price, custodian fee, approval threshold, other limits) | FR6, FR9 | T2 | not built beyond the size limit — fee policy and registration price 7.3, custodian fee 9.5, approval threshold 6.7 |
| I-37 | Configuration content is accepted: well-formed, within the §6 bounds of the [specification](./moonblokz-configuration-specification.md) (including `1 ≤ required_support ≤ MAX_AGGREGATED_SIGNATURES`, `1 ≤ H ≤ W` and `W + 2·H ≤` block-storage capacity) | FR8, FR56 | accept (→ FR16 exact evidence) | partial — built (5.7, 5.9, 5.11) except the restored check 3, which the code still clamps (6.4), and the `H` checks 14–16 (9.7) |
| I-38 | A candidate segment contains at least one chain-config block | FR2, FR3, FR8 | FR2 selection; pass | built (5.9, 5.13) |

### 5.8 Genesis and Trust Anchor

| # | Invariant | FRs | Checked | Built |
|---|---|---|---|---|
| I-39 | No artifact may assert a node #0 key other than `node_zero_public_key` (registration of id 0, balance entry of node 0, content signature) | FR7, FR9, FR50, FR69 | T1 | built (4.2, 5.9) |
| I-40 | Chain-config content carries node #0's signature, valid against the anchor | FR7 | T1 | built (5.9) |
| I-41 | Blocks #0 / #1 have the fixed genesis shape (all-zero `previous_hash`, node #0 registration and self-transfer, then the configuration); bootstrap exceptions need sequence **and** content match; self-built #0 / #1 skip the Tier 1 re-check | FR54, FR69 | T1 / pass for received genesis blocks | partial — creation and anchor checks built (1.4, 5.1); zero hash and content match 6.10 |

## 6. Derived Projections

Durable truth is only the node id, the private key (never handed to this module), the chain-config block with its lock flag, and the retained blocks (FR59, [ADR-003](./blockchain-adrs/ADR-003-authoritative-vs-derived-state-in-moonblokz-blockchain.md)). Everything below is derived, rebuilt on restart by the FR3 pass, and updated only by FR35's two triggers — forward extension and the FR23 walk; patching a projection outside the walk is forbidden. The mempool is neither durable nor derived: it is authoritative only for its own contents (FR30).

| Projection | Derived from | Forward update / chain-switch revert | Lives in | FRs | Built |
|---|---|---|---|---|---|
| Per-node balance | Seeds, transaction effects, creator credit | Apply per block / inverse of the block's effects | `NodeInfo.balances` (8 KB) | FR34, FR36, FR23 | partial — pass built (5.3, 5.4); head extension and per-block inverses built (6.0); chain-switch walk 6.3 |
| Accumulated vote | Vote credits (none to an FR72-excluded node), anti-capture interest, creator reset, deviation penalty, twin-evidence reset (FR72), liveness credit (DEC-178), balance-block seeds | FR37 steps in order / `undo_block` in reverse | `VoteEngine.accumulated_vote` (4 KB) | FR34, FR37 | built (3.1, 3.2); walk 6.3 |
| Public-key map | Registrations; slot 0 = trust anchor | Set per registration / clear on rollback | `NodeInfo.public_keys` (32 KB Schnorr, 96 KB BLS) | FR34, FR35 | partial — pass built (5.3); head extension and registration inverse built (6.0); chain-switch walk 6.3 |
| `max_known_node_id` | Earliest balance block's `max_node_id` (0 at genesis), +1 per registration | Increment / decrement | `NodeInfo.max_known_node_id` | FR3, FR34, FR54 | built (5.3, 5.4, 6.0) — pass; head extension and registration inverse (6.0) |
| `seed_source_sequence` | Latest balance entry or registration per node | Set to the applied block / bounded backward scan (≤ `W`) | `NodeInfo.seed_source_idx` (block index today) | FR34, FR50, FR23 | partial — the pass and the head extension set it; the move to a balance block for every listed entry built (6.0); sequence form 9.3; restore by backward scan 6.3 |
| Block navigation and available-UTXO bits | Durable block-tree; derived UTXO outputs and consumed UTXO inputs | Set on derivation, cleared on consumption / inverse for leaving blocks; re-zeroed on FR5 | `BlockEntry` table (600 × 76 B) | FR34, FR35, ADR-016 | partial — built (4.1, 5.3–5.5, 6.8, 6.11); head extension and per-block inverse built (6.0); walk 6.3 |
| Creator order | Accumulated vote, descending, ties by lower id; FR72-excluded nodes left out | Recomputed on demand | not stored | FR38 | built (3.3) |
| Equivocation exclusions | Twin evidence blocks in the window | Set when an evidence block is applied, cleared when it leaves the window; per-block inverse | window-derived; sized by 6.13 | FR72 | not built (6.13) |
| Mempool contents and eligibility | FR14 classification, Deferred marker, expiry sequence | Remove confirmed and expired / re-add on the backward walk, remove on the forward walk; FR15 re-evaluation | `Mempool` crate (~23–25 KB) | FR14, FR15, FR30, FR32 | partial — crate primitives built (2.1, 2.2); wiring 7.3–7.5 |
| Head `branch_value` | Per-block values above `S_tail` (exclusive) | Add on extension, subtract the new `S_tail` block's value on tail advance / unchanged by a switch | `ChainHeadEntry.branch_value` (Connected side heads); `active_branch_value` (the active branch) | FR19, FR21 | built (6.1); value hand-over on a switch 6.3 |
| `chain_heads` and block flags | Block-tree graph and the active-chain selection | FR19 events (i)–(v) | `ChainHeadsTable`, `BlockEntry.flags` | FR9, FR19, FR59 | partial — built (4.1, 4.4, 5.10); event (v) built (9.1); head-entry refresh on extension built (6.0); event (iii) 6.3 |
| `snake_chain` head and tail | Selected active chain | Promotion and head extension / set by the walk | two `u32` indices | FR19, FR48 | partial — head built (5.4); tail and its advance step built (9.1); head extension and tail advance in Ready built (6.0); set by the walk 6.3 |
| Active-node set `A` | Creators inside the active window | Recomputed when evaluated | not stored | FR12, ADR-015 | not built (6.4) |
| Approval accumulator | Accepted support for the proposer's own deviation | Accumulate / discarded on demotion and restart | `ApprovalAccumulator` (~2 KB; ~3.4 KB Schnorr per E6-4) | FR27 | not built (6.6) |

## 7. Main Flows

Each flow lists its trigger, its steps, and what happens on failure. Step tables cite the FR that defines the step; lifecycle transitions are §2, statuses §4, invariants §5.

### 7.1 Processing Pass

**Trigger:** an FR2 candidate in Collecting (T2) or a restart (7.8); never a chain switch (DEC-159). **Needs:** a loaded configuration and a chain-config block in the candidate.

| # | Step | FRs | Built |
|---|---|---|---|
| 1 | Enter Processing, reset the working set; the pass is not resumable | FR3, FR5 | built (5.3) |
| 2 | Mark the candidate backward from the tip; note the earliest balance block and first chain-config block | FR3 | built (5.3) |
| 3 | Zero the candidate's available-UTXO bits; preload and check its configuration against the held one (mismatch → T5 adopt) | FR3, FR8 | built (5.3, 5.9) |
| 4 | Seed the node-id watermark (earliest `max_node_id`, or 0 at genesis) | FR3, FR50, FR54 | built (5.3) |
| 5 | Traverse forward: FR6 state checks interleaved with derivation (roster, keys, balances, votes, available-UTXO bits, seed sources), including the `snake_chain` preservation checks inside the candidate; pre-seed acceptance only for window-anchored candidates | FR3, FR6, FR9, FR37, FR38, FR50 | partial — built (5.3, 5.4); per-block step shared with Ready and genesis, balance-entry equality built (6.0); missing checks see §5; `snake_chain` checks 9.2, 9.3, 9.5, 6.6 |
| 6 | Durably lock the configuration (storage first), promote the candidate Stored → Active, clean up mismatching or oversized blocks, enter Ready | FR4, FR8, FR9 | built (5.4, 5.9); `S_tail` built (9.1); `branch_value` seeding built (6.1) |

**On failure:** an FR6 violation runs FR5 (T4); the code retries once, plus one adopt retry. A local failure (storage read, missing vote parameter, mark overflow, missing block) deletes nothing: the node stays in Collecting and retries later; after a missing block, dangling parent references are unlinked so FR19 recovers the parent (FR5, DEC-134; built in 6.0). A durable-commit failure resets the projections and leaves the node in Collecting with its blocks intact. **Writes:** only the lock and deletions; everything else is RAM.

### 7.2 Forward Extension of the Active Head (Ready)

**Trigger:** a Ready block whose parent is the active head, then — after every Ready admission — a loop over already-stored children of each new head (orphans whose parent was just promoted, local-failure retries), at most `MAX_BLOCKS` passes. **Needs:** the durable lock; intake gates passed; a complete projection (§5 I-32).

| # | Step | FRs | Built |
|---|---|---|---|
| 1 | Intake gates: duplicate, window, FR17, Tier 1 | FR9, FR11, FR16, FR17, FR60 | built (4.2, 4.3); window built (9.1) |
| 2 | FR48 window gate when the extension passes `W`: classify the dropping tail, require the matching replay | FR48 | built (9.1, 6.0) |
| 3 | Per-block step (shared with the pass): block checks, transaction checks and invertible effects, balance-block equality, vote step, creator credit | FR6, FR9, FR16, FR35, FR36, FR37, FR50 | partial — built (6.0); vote header snapshots built (6.2); `mined_amount` bound 7.2; creator eligibility Epic 8; approval evidence 6.7; carry-forward 9.5; pacing 8.3 |
| 4 | On success: store the block, mark it Active (no Connected step), move the head, refresh `chain_heads`; advance the tail if gated | FR9, FR19, FR48 | built (9.1, 6.0) |
| 5 | Side effects: `branch_value`, mempool removal and expiry, deferred re-evaluation, seed sources | FR15, FR19, FR21, FR32, FR50 | partial — seed-source moves built (6.0); `branch_value` built (6.1); mempool 7.4; deferred re-evaluation 7.5; seed-source sequence form 9.3 |
| 6 | Loop over stored children of the new head, choosing by the FR21 rank at the child level (value, size, hash); a failing child is deleted with descendants | FR9, FR16, FR21 | built (6.0, 6.1, 6.2) |
| 7 | Response: at most one component per call — support message, re-disseminated transaction, or chained new block; further ones through `NextCall` | FR10, FR12, FR13, FR45 | not built (6.4, 7.5, 8.3) |

**On failure:** a failing direct extension is not stored (`Invalid-evidence`) and leaves derived state unchanged — every effect of the step is invertible and the applied prefix is inverted; a failing stored child is deleted with its descendants; a local error admits the block as Stored for a later retry; a storage failure after a good step reverts the step and reads as Already-known to the radio (the code still returns `Unstorable` until Story 11.2). No FR5 rollback or phase reversion in Ready. An unseeded actor is invalid in Ready, which relies on FR50 coverage (I-32, validated from 9.3). Accepted residual (DEC-138): the vote apply runs after the durable write but before `chain_heads` sees the block, so a vote-arithmetic refusal deletes the just-written block before anything links to it; only its bytes stay in the freed slot until reuse.

### 7.3 Chain Switch

**Trigger:** only FR22's two events — a head extension of a non-active Connected / Active head, or a Stored → Connected transition (including those of a Ready entry). The comparison runs once, after the admission, for the highest-ranked admissible affected head (DEC-152). Until 6.3 a win is not acted on (DEC-153). **Needs:** an operationally admissible candidate (FR21; derived from the window, DEC-154).

| # | Step | FRs | Built |
|---|---|---|---|
| 1 | Stored → Connected for side blocks: Tier 2 once ancestry reaches the active chain; failure deletes with descendants | FR9, FR19, FR21 | built (6.1) |
| 2 | Compare the highest-ranked admissible affected head with the active head: `branch_value`, then larger tip size, then lowest tip hash, in RAM only | FR21, FR22 | built (6.2); the switch it calls 6.3 |
| 3 | Bound: switch only if the divergence point satisfies `D ≥ S_head − H + 1` against the node's own head; a branch below it is reclaimed (§7.4 step 8), never switched; never across a same-parent twin without a verdict for the candidate's twin (§7.9); the common ancestor at or above the completeness horizon `C` (DEC-167); then the signature pre-pass for fork-independent creators (DEC-170) | FR23, FR58 | not built (6.3: `C`, pre-pass, and the bound with `H` from a seam returning 50, DEC-166; 9.7: the configured `H` and reclamation) |
| 4 | Backward walk to the common ancestor: invert every projection per block, re-add its transactions to the mempool, Active → Connected | FR9, FR23, FR32, FR35, FR39 | not built (6.3; only `undo_block` exists) |
| 5 | Forward walk to the new head: Tier 3 plus effects per block (the 7.2 step), remove transactions from the mempool, → Active; tail advances run against a virtual tail, their releases wait for the commit (DEC-168); below-tail references inside the bound are trusted until 9.7 (DEC-166) | FR9, FR23, FR39 | not built (6.3) |
| 6 | Commit: the deferred releases and FR21 (iii) subtractions, window expiry, deferred re-evaluation, `chain_heads` event (iii); discard a demoted proposer's accumulator | FR15, FR19, FR27, FR32 | not built (6.3, 6.6, 7.4, 7.5) |
| 7 | Withdrawn (DEC-159): no deep-zone path | FR58 | — |

**On failure (DEC-168):** a Tier 3 failure on the forward walk restores the original chain in place, then deletes the failing block with its descendants (no FR5); a local fault restores without deleting; a fault during the restore stops at the highest re-applied block, a valid shorter active chain. The whole switch runs inside one call, so no query sees a half-switched chain (FR24). No block bytes are written.

### 7.4 Tail Advance and Replay

**Trigger:** a head extension (inbound or created) that would take the chain past `W`, or Ready entry on a segment longer than `W`. Before the chain first reaches `W` nothing advances.

| # | Step | FRs | Built |
|---|---|---|---|
| 1 | Set `S_tail` at genesis and at every Ready promotion; on Ready entry release active blocks below it | FR48 | built (9.1) |
| 2 | Classify the dropping tail block: nothing, chain-config, seed source, or unspent UTXOs (at most one) | FR48–FR51 | built (9.1); inputs kept current on head extension built (6.0); on chain switch 6.3 / 7.2 |
| 3 | The head extension must be (or carry) the matching replay; otherwise it is invalid | FR16, FR48 | partial — decision built (9.1); enforced on inbound head extensions built (6.0); creator side 8.3 |
| 4 | Replay content: configuration identical to the lock; balance entries at `seq − 1` plus bonus pack; carry-forward 1:1 with fee reduction | FR9, FR49, FR50, FR51 | not built (9.2, 9.3, 9.5) |
| 5 | Spacing trigger: at length `W` with nothing pending, emit a bonus-pack balance block when spacing exceeds `⌈W / ⌈N_active / K_max⌉⌉` | FR50 | not built (9.4) |
| 6 | On acceptance: `replay_block_reward` once per replay-bearing block; seed sources updated | FR36, FR50 | not built (7.2, 9.x) |
| 7 | Advance the tail in the same step: the dropping block enters the verification horizon; the block leaving the horizon has the branches forked at it deleted (FR58), is released, and the new lowest retained block is unlinked; delete Stored fragments whose missing parent is at or below `S_tail` (as built; DEC-161 replaces this criterion, step 8); subtract the new tail's value from every branch | FR19, FR21, FR48, FR58 | built (9.1, 6.1) with no horizon; horizon retention 9.7 |
| 8 | Reclaim side branches whose divergence point left the switch bound (`D ≤ S_head − H`), and Stored fragments whose missing parent lies at `q ≤ S_head − H − 1` (DEC-161), at the head extension or switch that moves them out; keep the verification horizon `H`, which FR57 never evicts | FR58 | partial — reclamation at the tail advance built (9.1, step 7: forks and Stored fragments); the `S_head − H` bound and horizon `H` 9.7 (0 until then) |
| 9 | On a deviation branch the proposer emits intervening replays in tail fall-out order, spaced by `deviation_replay_insertion_delay_ms` | FR27, FR29 | not built (6.6) |

**On failure:** a non-preserving inbound extension is invalid (not stored, or deleted if already stored); a read error blocks the advance without mutation. Released blocks stay in storage until overwritten (restart footprint, 7.8). No UTXO-saturation handling (FR52).

### 7.5 Block Creation

**Trigger:** an FR46 deadline, the mempool reaching the fill threshold, an accepted block that makes this node creator, or a pending replay. **Needs:** Ready, an eligible local node, head below `u32::MAX − 1`.

| # | Step | FRs | Built |
|---|---|---|---|
| 1 | Is the local node the FR38 top, an admitted fallback, or the proposer? | FR38, FR44 | not built (8.2; vote-crate primitives built in 3.3) |
| 2 | Trigger: fill threshold reached, or timer elapsed with at least one includable transaction; replay obligations go at once | FR45, FR46 | not built (8.3) |
| 3 | Assemble: replay first, then a due FR71 twin evidence block, then mempool by fee per byte (own first on ties), never mixing registrations and complex transactions, within `block_size_limit` | FR45 | not built (8.3) |
| 4 | Header: next sequence, creator, `mined_amount = mining_reward`, `consumed_votes`, deviation fields, parent hash, signature via the crypto module over the header and `H(payload)`; never a second child of a parent (FR70) | FR45, FR53, FR68 | not built (8.1, 8.3) |
| 5 | Accept the created block through the 7.2 path, emit it as the outcome, reschedule | FR10, FR45, FR46 | not built (8.3, 8.4); the §7.2 path it reuses is built in 6.0 |
| 6 | Grace expiry: admit one more creator, no vote reset; a resulting deviation block opens the approval accumulator | FR27, FR37, FR44, FR47 | not built (8.5) |

**On failure:** no source specifies a failure path for creation itself. With nothing to include, no block is created and that is not an error. The inbound mirror — the pacing soft-fail to `Deferred` — is §4.3 (Story 8.3).

### 7.6 Deviation and Approval

| # | Step | FRs | Built |
|---|---|---|---|
| 1 | A direct head extension by a creator other than the FR38 top, with a valid signature, starts support evaluation | FR12, FR16 | not built (6.4) |
| 2 | Eligibility: creator inside the grace window; local node in `A'` (`A` without proposer and originally-top node) and among the `m` lowest `node_value`, `r_eff = min(required_support, ⌊\|A'\|/2⌋ + 1)`; no conflict; no pending tail replay | FR12, ADR-015 | not built (6.4) |
| 3 | Emit a signed support message as the intake outcome | FR10, FR12 | not built (6.4; signed content open, E6-2) |
| 4 | Support intake in every phase: format and de-duplication always; signature, coherence and subgroup checks once the deviation is on the local active chain; Accepted → relay | FR26 | not built (6.5) |
| 5 | Proposer: collect subgroup supporters; at `r_eff` aggregate once and build the evidence block (`supporter_vote_sum`, signature, ids) | FR12, FR27 | not built (6.6) |
| 6 | The deviation block is Active at once and penalizes the original top creator; its branch is never higher in value than the normal branch (equal votes tie; FR21 tie-break) until the evidence block lands | FR21, FR28, FR37 | partial — admitted Active with its FR37 vote effects by the head-extension step built (6.0); branch value built (6.1); two-stage scoring built (6.2); the evidence block's value 6.7 |
| 7 | Validate incoming evidence blocks (I-15) | FR6, FR9 | not built (6.7) |

**On failure:** an ineligible node sends nothing and gives no reason; eligibility is re-evaluated as the chain advances. The accumulator is not durable.

### 7.7 Transaction Lifecycle

| # | Step | FRs | Built |
|---|---|---|---|
| 1 | Not-ready gate outside Ready | FR1, FR14, FR55 | built (5.1) |
| 2 | Classify (window, uniqueness, Deferred for not-yet-evaluable, Rejected for incurable) | FR14 | not built (7.3) |
| 3 | Add to the mempool; at capacity evict at random, own transactions last | FR30, FR31, FR33 | crate built (2.1, 2.2); wiring 7.4 |
| 4 | Local submit: same validation, outcome Created / Held / Rejected with reason | FR55 | not built (10.5) |
| 5 | Chain changes: remove confirmed and expired, re-add on rollback, re-evaluate Deferred (never reject them) | FR15, FR32, FR39 | not built (6.3, 7.4, 7.5) |
| 6 | Re-dissemination addendum after an accepted external block with ordinary transactions | FR13 | not built (7.5) |
| 7 | Replenishment request below threshold; responder returns at most one transaction | FR43, FR46 | crate iterator built; not built (8.4, 10.4) |

The mempool is empty after a restart (FR30, FR59). Radio outcomes carry no cause (FR10); local outcomes do (FR55).

### 7.8 Restart

| # | Step | FRs | Built |
|---|---|---|---|
| 1 | Read the control plane once; unreadable with blocks present → `Rejected` | FR59 | built (5.10) |
| 2 | Load a durable configuration after re-checking its FR7 signature; failure → `Rejected`, nothing erased | FR7, FR8, FR59 | built (5.9, 5.10) |
| 3 | Rebuild the block-tree from every readable slot (no Tier 1 re-run), resolve parents, rebuild `chain_heads` | FR11, FR19, FR59 | built (5.10) |
| 4 | Without a durable lock, re-establish the tentative configuration from a stored chain-config block | FR8 | built (5.10) |
| 5 | Run the FR2 → FR3 → FR4 spine (7.1), then re-run the lock-time cleanups | FR2–FR4, FR8, FR9 | built (5.10) |
| 6 | Return `ResumedReady` or `ResumedCollecting`; release blocks below `S_tail` | FR48, FR59 | built (5.10, 9.1) |

Blocks deleted before the restart can return, because storage has no delete; every deletion driver re-fires, so the node converges ([architecture](./moonblokz-blockchain-architecture.md) §3.6). The control-plane load still returns the private key until Story 6.9.

### 7.9 Twin Conflict

**Trigger:** the intake of a second same-parent block by one creator (FR70), or a twin vote naming a block other than the one held (FR71). **Needs:** both twins share the parent `P`, which every involved node holds.

| # | Step | FRs | Built |
|---|---|---|---|
| 1 | Detect the conflict `(c, s, P)` at intake against the stored children of `P` | FR70 | not built (6.12) |
| 2 | Hold: FR22 never switches across the twin to the other twin without a verdict; the chain keeps extending | FR22, FR70 | not built (6.12) |
| 3 | FR40 flags confirmations at or above the unresolved twin as disputed | FR40, FR70 | not built (6.12) |
| 4 | Subgroup from the state at `P` (seed with `"twin"`, `A` without `c`); a member votes once for its first-adopted twin; a restarted node abstains at or below its ready-entry head; votes are relayed | FR26, FR71 | not built (6.13) |
| 5 | The entitled creator of a branch emits the twin evidence block after due replays: proof, `c`'s vote snapshot, and with a quorum the bitmap verdict | FR45, FR71 | not built (6.13) |
| 6 | Validate it (I-43) and apply FR72: exclude `c` from the creator order and `A` for `W` blocks, reset its accumulated vote and credit it nothing while excluded | FR71, FR72 | not built (6.13) |
| 7 | A node holding the other twin admits the evidence block on its side branch; a verdict for that branch lifts the hold and FR22 switches | FR22, FR71 | not built (6.13) |

**On failure:** without a quorum the evidence carries the proof only (penalty, no decision) and the hold stays; a node that adopted the losing twin first stays on it until a restart, whose FR2 acquisition normally picks the majority chain. One voting round only.

## Related Documents

- [Blockchain PRD](./moonblokz-blockchain-prd.md) — authoritative wording of every requirement cited here; find one by `^- FR<n>:` via the [FR/NFR Navigation Index](./moonblokz-blockchain-prd-fr-index.md).
- [Blockchain Architecture](./moonblokz-blockchain-architecture.md) — the API surface, module split, and data structures behind these behaviors.
- [Configuration Module Specification](./moonblokz-configuration-specification.md) — configuration commitment states and the parameters the checks read.
- [Blockchain Algorithm](./moonblokz-blockchain-algorythm.md) — data structures and the article-era reasoning behind the algorithms.
- [Glossary](./moonblokz-glossary.md) — `block_size_limit` vs. `MAX_BLOCK_SIZE`, `snake_chain`, and other cross-subsystem terms.
- [Open Gaps Register](./moonblokz-open-gaps-register.md) — what is still undecided.
