# MoonBlokz Blockchain Algorithm Summary

A compact, implementation-free restatement of the blockchain consensus and ledger algorithm, written as self-contained input for adversarial analysis (attack surface, weaknesses). It is **not authoritative**: on any divergence the [Blockchain PRD](./moonblokz-blockchain-prd.md) wins, then the [ADRs](./blockchain-adrs/ADR-INDEX.md), the [Configuration Module Specification](./moonblokz-configuration-specification.md) and the [Blockchain Algorithm Model](./moonblokz-blockchain-algorythm.md). Source tags: `[FRn]` = Blockchain PRD, `[ADR-n]` = blockchain ADR, `[DEC-n]` = [Decision Register](./moonblokz-decision-register.md), `[OG-n]` = [Open Gaps Register](./moonblokz-open-gaps-register.md), `[config §n]` = Configuration Module Specification, `[crypto]` = [Crypto Algorithm Model](./moonblokz-crypto-algorythm.md), `[I-n]` = [Operating Model](./moonblokz-blockchain-operating-model.md) invariant. §15 lists the open specification gaps; §16 lists security-relevant facts, stated without evaluation.

## 0. System model

- Nodes: constrained devices on lossy, low-bandwidth broadcast radio (LoRa-class); partial, delayed, conflicting knowledge; no global clock — each node has only a local monotonic clock.
- Identity by sponsorship: a node exists only after an existing node registers its key and pays `registration_price`. Node ids are contiguous, stride-1, never reused.
- Consensus class: leader-based, vote-weighted, best-effort / eventually consistent; fork choice is a heaviest-branch rule over vote-derived block values. No PoW/PoS, no BFT finality.
- Bounded history: each node retains a sliding window of `W` active-chain blocks plus a verification horizon of `Hv` blocks below it; essential state is re-emitted at the head before it drops (`snake_chain`).
- Explicit non-goals: radio flooding/jamming by a stronger adversary; physical isolation with controlled misinformation ([Concept](./moonblokz-blockchain-concept.md) §Security Framing); chain-internal recovery from long disconnection [FR60, OG-006].

## 1. Notation and primitives

- `H` = SHA-256. Hash order in tie-breaks: digest as big-endian unsigned integer.
- `Sig` = one build-time backend: Schnorr over secp256k1 (pk = 32 B x-only, even-y; sig = `(r, s)`, 64 B) or BLS12-381 (pk 96 B, sig 48 B). `MAX_AGG = 50` aggregated signatures [crypto].
  - Schnorr sign: `k = H_tag("nonce", sk ‖ m ‖ ctr)` (deterministic), `R = kG` normalized to even y, `e = H_tag("challenge", r ‖ pk ‖ m) mod n`, `s = k + e·sk mod n`. Verify: `r < p`, `s < n`, `x(sG − eP) = r`.
  - Schnorr aggregate (one message `m` for all signers): `r_sum = Σ r_i mod p`; `seed = H_tag("aggregate", r_sum ‖ r_sum ‖ m)`; `w_i = H_tag("rand", seed ‖ i) ∈ [1, n−1]`; `s* = Σ w_i s_i mod n`; artifact `(count, s*, r_1..r_k)`, `34 + 32k` B. Verify `x(s*G − Σ w_i e_i P_i) = x(Σ w_i R_i)`; signer order is part of the artifact.
  - BLS aggregate: library multisignature aggregation, constant 50 B.
- `S_head`, `S_tail` (oldest active block; `0` until the chain reaches length `W`); `W` = window length (chain-config); `Hv ∈ [1, W]` = verification horizon and maximum switch depth, chain-config `verification_horizon` (default 50), identical on every node (named `H` in the PRD and [Glossary](./moonblokz-glossary.md)) [FR58, DEC-160].
- `M` = `max_known_node_id` watermark; node `n` exists ⇔ `n ∈ [0, M]`.
- `av(n)` = accumulated vote (u32, checked arithmetic); `vs` = `vote_scale`, `vi` = `vote_interest`.
- `pk0` = node #0 public key, compiled into every node (immutable trust anchor) [FR69].

## 2. Data model (binary, little-endian)

**Block** `b = (header, payload)`, `|b| ≤ L = block_size_limit ≤ 2016 B`.
Header: `version u8 (=1) | sequence u32 | creator u32 | mined_amount u32 | payload_type u8 | consumed_votes u32 | first_voted_node u32 | consumed_votes_from_first_voted_node u32 | previous_hash [32] | signature`.
- `hash(b) = H(bytes(b))`, signature bytes included.
- `signature = Sig_{sk(creator)}(header with the signature field zeroed ‖ H(payload))` [FR70].
- `sequence(b) = sequence(parent) + 1`; `sequence = u32::MAX` is always invalid (no wrap in MVP) [FR53].
- `payload_type ∈ {1 transaction, 2 balance, 3 chain-config, 4 approval evidence, 5 twin evidence}`.

**Transaction block** (type 1): `count u16`, then transactions, each starting `type u8 | vote u32`. A transaction block never holds both a registration (T2) and a complex transaction (T3) [FR6, FR9 Tier 1].

- **T1 node transfer**: `anchor u32, initializer, receiver, amount u64, fee u32, comment u64, sig`; `sig = Sig_{sk(initializer)}(transaction bytes, sig zeroed)`, covering `type` and `vote`.
- **T2 registration**: `initializer, new_node_id, registration_price u64, fee u64, new_pk, new_key_sig = Sig_{sk_new}(new_pk), sig = Sig_{sk(initializer)}(transaction bytes, sig zeroed, new_key_sig present)`. No anchor.
- **T3 complex**: `in_count u8, out_count u8`, inputs, outputs (≥ 1 output; UTXO outputs per block ≤ `max_block_utxo_output`).
  - UTXO input: `(q = block_sequence u32, i = output_index u8, a = amount u64, sig)`. `sig` is by the key equal to the referenced output's address, "over this input and all outputs" (preimage open, [OG-013]). `i` = position in block `q`'s UTXO-output stream (UTXO outputs only, transaction order, output order) [ADR-016, DEC-131].
  - Balance input: `(anchor, initializer, amount, comment, sig)`, sig "over this input and all outputs".
  - UTXO output: `(address = owner pk [32], amount)`. Balance output: `(receiver u32, amount)`.
  - `fee = Σ in − Σ out ≥ 0`. Zero inputs ⇒ valid only as a `snake_chain` carry-forward (§10.3).

**Balance block** (type 2): `count u16, max_node_id u32` (= the chain-global `M` at this sequence), entries `NodeInfo = (owner u32, balance u64, vote_count u32, public_key)` holding the owner's state at `sequence − 1`.

**Chain-config block** (type 3): `count u16, {key u8, len u8, value}*, content_sig = Sig_{sk0}(content region)`. Key bit 7 = literal/bytecode, bits 0–6 = parameter id ∈ [1, 126]; content is an override set over code defaults; an unknown id ⇒ reject. Bytecode values run on a deterministic, fuel-bounded VM; failure falls back to default, then fallback literal. Configuration is immutable for the chain's lifetime and every chain-config block is byte-identical [FR7, FR8, FR17, config §3–§5].

**Approval-evidence block** (type 4): `supporter_vote_sum u64 | aggregated signature | supporter id list`; byte encoding and binding to the deviation block are open [OG-008].

**Twin evidence block** (type 5): `c`, `s`, `hash(P)`; the other twin's header and payload hash (the proof); `c`'s accumulated-vote snapshot; with a quorum, a verdict = signer bitmap over the subgroup positions + aggregated signature [FR71].

**Support message** (off-chain): `(supporter_id, deviation_sequence, deviation_block_hash, sig)`; signed content per FR12 = `(proposer, deviation_sequence, deviation_block_hash)` (FR26 wording differs, [OG-014]).

**Twin vote** (off-chain): signed content `("MBTWN", c, s, hash(P), hash(chosen twin))`, one per member and `(c, s, P)` [FR71].

## 3. Node state

- Durable (authoritative): `node_id`, private key (outside the module), locked chain-config, retained blocks [FR59].
- Derived (recomputable from the window, per-block invertible) [FR34]: `balance(n)`, `av(n)`, `id → pk`, `M`, `seed_source_sequence(n)`, and per-block available-UTXO bitfield `U_q[i]` (1 ⇔ output exists and is unspent on the active chain) [ADR-016, DEC-132]. Creator order = sort by `(av desc, id asc)` [FR38].
- Block tree with statuses Stored / Connected / Active; bounded `chain_heads` table of all tips, with `branch_value` for Connected/Active heads [FR18, FR19].
- Mempool: non-durable, per node, randomized eviction; not consensus-relevant [FR30–FR33].

## 4. Lifecycle

`collecting → processing pass → ready`; a failed pass returns to collecting; a restart always re-enters collecting [FR1–FR5, FR59].

- **Collecting**: store every parseable block passing Tier 1; recover missing parents (≤ 1 request per tick, to the most-overdue Stored head, under per-head retry and global minimum-emit throttles) [FR19, FR46]. Stop when a continuous segment is (genesis-anchored ∨ length ≥ `W`) ∧ contains a chain-config block; choose `argmax tip.sequence`, tie `argmin tip.hash` — not branch value [FR2].
- **Config commitment**: the first chain-config block whose `content_sig` verifies against `pk0` and which the configuration module accepts is loaded tentatively; before ready the candidate's configuration must byte-match it, then it is durably locked (set-once). Chain-config-derived failures invalidate blocks only after the lock, except the UTXO-reference `W` bound and the per-block UTXO-output cap [FR8, FR9, DEC-129, DEC-133].
- **Processing pass** (one forward pass, non-resumable) [FR3, FR6]: backward-mark the candidate; preload its configuration (all its chain-config blocks byte-identical); forward-rebuild all derived state; validate every block and transaction.
  - Seeding: a node is seeded from its earliest in-segment balance entry, else its in-segment registration. **Pre-seed rule**: checks depending on a node's state before its seed are auto-accepted; structural, config-derived and UTXO-reference checks still run.
  - Window-anchored candidate: replay obligations whose dropping block precedes the segment are trusted; a UTXO input referencing a block below the anchor is trusted (declared amount included) if `≤ W` back, invalid if further [FR3, FR6, DEC-129].
  - `M` starts from the earliest balance block's `max_node_id` (0 at genesis).
  - Failure ⇒ atomic discard of the working state; delete the earliest offending block and its descendants (if unidentifiable: the candidate head only); return to collecting [FR5].
- **Ready**: intake, extension, fork choice, block production, approval.

## 5. Block intake and validity

Outcome ∈ {Accepted-new, Already-known, Invalid-evidence, Out-of-window, Deferred} [FR10]; duplicate key `(sequence, hash)` [FR11].

- **Out-of-window** (ready only): `S_new ≥ S_head + W` or `S_new < S_tail` ⇒ discarded, not stored; the forward case is logged as long-disconnect [FR60].
- **Tier 1** (block bytes + locked configuration; failure ⇒ not stored; the list is exhaustive) [FR9, FR16]: version; payload schema; `|b| ≤ L`; chain-config content equals the lock; ready-window membership; `s > anchor` for T1 and every balance input; no-self-vote; T3 with inputs: `Σ declared in ≥ Σ out` without u64 overflow; T2/T3 exclusion; `new_key_sig` valid; anchor checks (`new_node_id = 0 ⇒ new_pk = pk0`; balance entry `owner = 0 ⇒ pk = pk0`; `content_sig` valid against `pk0`); `sequence ≠ u32::MAX`; UTXO-output cap; UTXO `q < s` (code-level only, [OG-011]).
  - The creator signature is exact evidence in ready state for a direct extension of the active head with a derivable key, and for any block whose creator's key is branch-independent (node 0, or seeded at or below `S_tail`). Otherwise signature checks are opportunistic (a failure does not invalidate) and the decision waits for Tier 3. A verified block is marked and not verified again [FR9, DEC-157].
  - Header vote form: `first_voted_node = c` ⇒ `consumed_votes_from_first_voted_node = 0`; otherwise `consumed_votes ≤ consumed_votes_from_first_voted_node` [FR9].
  - For a ready direct head extension, also exact evidence: a non-proposer block above a deviation block; a carry-forward failing §10.3 (i)–(v).
- **Tier 2** (Connected: ancestry reaches the active chain; lookups only): locked configuration at the block's sequence (fee policy, price, custodian fee, approval threshold); `mined_amount = mining_reward`; UTXO input `q` retained, `i` in range, declared `a` equals the output amount; `new_pk` unique; structural `snake_chain` checks (the `D − W` reference is retained for every switchable branch, FR58).
- **Tier 3** (Active: replay of derived state, via the processing pass, chain-switch replay or a ready head extension): `consumed_votes`, `first_voted_node`, `consumed_votes_from_first_voted_node` match the projected vote state; creator credit; NodeInfo equals the state at `s − 1` and `max_node_id = M`; every rule of §6; `U_q[i] = 1`; carry-forward state; replay presence and order; evidence (subgroup membership, count ≥ `r`, aggregate verifies, ≤ `MAX_AGG`, recomputed `supporter_vote_sum`); twin evidence (proof header, `s ≥ S_tail`, one per `(c, s)` in the window, verdict bitmap and aggregate) [FR71]; creator entitled — top, admitted fallback or proposer, never an FR72-excluded node [I-10].
- **Pacing soft-fail** (ready intake, ordinary transaction blocks only): `fill(b) < θ·L` ∧ `arrival(b) − arrival(parent) < inter_block_interval` (local arrival times) ⇒ Deferred and re-evaluated after the remainder; not invalid.
- Ready direct head extension: validated before storing; valid ⇒ Stored → Active and becomes the head; invalid ⇒ not stored. Stored children of the new head are then applied, lowest hash first ([Algorithm Model](./moonblokz-blockchain-algorythm.md) §Ready phase, FR16).
- Invalid ⇒ the block and its descendants are deleted, only through a closed list of deletion paths [FR5, FR9, FR19].

## 6. Transaction validity (inclusion in a block at sequence `s`)

Common: every referenced id (initializer, receiver, balance-input initializer, balance-output receiver, `vote`) lies in `[0, M]`. No-self-vote: `vote ≠ initializer` (T1, T2) and `vote ≠` any balance-input initializer (T3); permanent exemption `initializer = 0 ∧ vote = 0`; `vote = 0` is always a valid target; block #0 is exempt [FR6, FR37, FR54].

- T1: `s > anchor`; `balance(init) ≥ amount + fee`; sig valid; key `(init, recv, amount, anchor, comment)` not already confirmed on the candidate chain; fee within the per-byte `[min, max]` policy (predicate open, [OG-016]).
- T2: `balance(init) ≥ price + fee`; `new_node_id = M + 1`, strictly sequential within a block; `new_pk` unique on the chain; `new_key_sig` and sig valid. Effect: `M += 1`, bind `id → pk`, burn `price`.
- T3 with ≥ 1 input: each UTXO input `(q, i, a)` has block `q` on the active chain, `q ≥ S_tail`, `U_q[i] = 1`, `a = amount(q, i)`; each balance input has `s > anchor` and sufficient balance; every sig valid; `Σ in ≥ Σ out`; no negative balance; the input set not already confirmed. Effect: clear `U_q[i]`; set the block's own output bits.
- Mempool intake (ready only) [FR14]: `anchor ≥ S_tail` with no upper bound; UTXO `q ≥ S_tail`; unknown `vote` target ⇒ Deferred, never Rejected; competing spends admitted. Uniqueness keys: T1 tuple above; T2 `new_pk`; T3 full ordered input set. The `anchor ≥ S_tail` bound has no block-level counterpart [OG-017].

## 7. Vote dynamics and creator selection

For every accepted block `b` with creator `c`, in order [FR37]:
1. `∀ n: av(n) += min(⌊av(n)·vi / vs⌋, vs)` — anti-capture interest, `c` included.
2. `∀ tx ∈ b` except carry-forward: `av(tx.vote) += vs`.
3. `av(c) := 0`. A deviation block additionally sets `av(first_voted_node) := 0`.
4. Overflow of any `av` ⇒ the block is invalid.

Header snapshots: `consumed_votes(b)` is `av(c)` after step 1 and before step 2. A deviation block's `consumed_votes_from_first_voted_node` is that node's value at the same point; it is verified at Tier 3 and used to undo the penalty, but it adds no branch value. So a block's own credits never enter its value, while step 3 still wipes the credits `b` gave `c` [FR37, DEC-156].

The expected creator after block `s` is the top of the creator order; zero-vote nodes are included, so node 0 leads in the all-zero state, and nodes under an FR72 exclusion are left out [FR38, FR44]. A twin evidence block also sets `av(c) := 0` for the equivocating creator [FR72]. The vote target is chosen outside the chain by the transaction creator (radio scoring module) and is validated only for existence and no-self-vote [FR55].

## 8. Block production

- Ordinary block when `mempool_fill ≥ θ·L`, or `inter_block_interval` elapsed ∧ ≥ 1 includable transaction. Empty blocks are never produced; no traffic means no blocks, and the stall is not an error [FR45].
- Replay obligations (§10) are emitted as soon as the node is eligible (top, admitted fallback or proposer), without fill or timer wait, and take precedence over ordinary content and over evidence emission [FR45, FR49].
- Content order: replay first, then fee per byte, own transactions first, `hash_crc32`, bytes. `mined_amount = mining_reward`.

## 9. Fallback and approval

**Grace expansion** [Algorithm 5, FR44, FR47]: with `g = grace_period_window_ms`, after `k` local expiries the admitted set is the top `k + 1` of the creator order. Expiry changes no vote and does not shift the order; the original top may still produce at any time, and its block is never lower in value than an approval-less deviation branch (equal votes tie). The window's time origin is open [OG-015].

**Deviation block** `d` at sequence `p` by an admitted `x ≠ u` (`u` = top at `p − 1`): `first_voted_node = u`, `consumed_votes` = `x`'s value after step 1 of §7; on acceptance `av(x) := 0` and `av(u) := 0` [FR28, FR37]. It is accepted immediately as an active extension. `u` is the top, so `av(x) ≤ av(u)`, and `v(d)` never exceeds the normal block `n` that `u` could create at `p`. An approval-less deviation branch therefore cannot overtake the normal branch, and equal votes tie [FR21, FR28, DEC-156].

**Subgroup** [ADR-015, FR12], evaluated on the evaluator's local active chain:
- `A` = nodes that created at least one block or transaction within the active window.
- `seed = H(hash(tail) ‖ x ‖ p)`; `node_value(n) = (H(seed ‖ n) mod 2³²)·2³² + n`.
- `m = min(2r − 1, |A|)` with `r = required_support ∈ [1, MAX_AGG]`; subgroup = the `m` members of `A` with the smallest `node_value`.

**Support** [FR12]: node `n` signs iff `x` is inside `n`'s local grace window ∧ `n ∈ A` ∧ `n ∈` subgroup ∧ `n` has not supported a different block for the same `(x, p)` ∧ no tail replay is pending locally. Relay [FR26]: Accepted iff the format is valid ∧ not an LRU duplicate, plus — when evaluable — signature valid, tuple coherent, and, if `d` is on the local active chain, supporter ∈ subgroup ∧ no evidence yet.

**Evidence** [FR27]: only the proposer `x` accumulates; at `|supports| ≥ r` it emits evidence `e` with
`supporter_vote_sum = (⌊m/2⌋ + 1)·av_{p−1}(x) + av_{p−1}(u)`,
independent of which `r` members signed. Only `x` may create blocks above `d` on that branch; any other creator is exact evidence. Intervening replay blocks and `e` carry `consumed_votes = 0` and `first_voted_node = x`; replays are spaced by `deviation_replay_insertion_delay_ms`, and `e` is emitted without delay [FR29]. Insufficient support does not widen the subgroup: the next grace expiry admits the next candidate, with a fresh subgroup [ADR-015]. Contribution at `p`: `v(d) + v(e) = (⌊m/2⌋ + 2)·av(x) + av(u) > av(u)` [FR12 (a)].

**Twin conflict** [FR70–FR72]: twins = same creator `c`, sequence `s` and parent `P`, different hash (different parents are legitimate). Honest nodes never sign a second child of a parent. A ready node holds its first-adopted twin. Subgroup from the state at `P`: `seed = H("twin" ‖ hash(tail_P) ‖ c ‖ s)`, `A` without `c` and FR72-excluded nodes, `m = min(2r − 1, |A|)`; members vote once for their first-adopted twin (restarted nodes abstain at or below their ready-entry head); one round. The entitled creator of a branch emits the twin evidence block after due replays, with a verdict when it holds `r` votes for its own twin; a verdict lifts the hold for that branch. Penalty: while the evidence block is in the window, `c` is out of the creator order and of `A`. The block carries ordinary `consumed_votes` and no extra value.

## 10. `snake_chain` (bounded window)

### 10.1 Window

Active length ≤ `W`; the tail does not advance before the chain reaches `W`. Extending past `W` drops tail block `t`; if `t` carries essential state, the new head must be or carry the matching replay, otherwise the head is invalid [FR48]. Each dropping block triggers at most one replay category (T2/T3 exclusion), so one head always suffices:

| Dropping `t` | Required head content |
|---|---|
| chain-config block | byte-identical chain-config block [FR49] |
| balance block, or transaction block holding some node's only in-window seed | balance block (§10.2) |
| transaction block with unspent UTXO outputs | zero-input carry-forward transaction (§10.3), optionally with ordinary transactions (no T2) |
| approval or twin evidence block | nothing |

The creator of a replay-bearing block receives `replay_block_reward` [FR36 (c)].

### 10.2 Balance coverage [FR50]

Invariant: every registered node has a seed source (balance entry or registration) inside the window.
- Trigger 1: nodes with `seed_source_sequence = S_tail` ⇒ mandatory entries.
- Trigger 2: length = `W` ∧ nothing else due ∧ blocks since the latest balance block `> ⌈W / ⌈N/K⌉⌉` (`N = M + 1`, `K` = entries per block).
- Fill: mandatory entries, then the canonical bonus pack ordered by `(seed_source_sequence asc, id asc)` up to `L`; any other entry set is invalid. Entries hold the state at `s − 1`.

### 10.3 UTXO carry-forward [FR51, ADR-016]

For a dropping transaction block `t` with unspent set `O = {o : U_t[o] = 1}`: one zero-input transaction at the head (1:1 with `t`, no coalescing) with outputs `{(addr_o, a_o − c) : o ∈ O, a_o ≥ c}`, where `c = custodian_fee`; outputs with `a_o < c` are discarded (wording drift, [OG-018]). Acceptance requires (i) the tail advances by exactly one; (ii) `t` is a transaction block with UTXO outputs; (iii) the `(address, amount)` multiset equals `O` after the fee; (iv) below-fee outputs are omitted; (v) no extra outputs. Carry-forward is vote-neutral and fee-free; carried outputs get new references `(head_seq, i′)` and old references stop resolving.

## 11. Fork choice

- Per-block value: `v(b) = consumed_votes(b)` for a normal block; `v(e) = supporter_vote_sum` for evidence; `0` for replay blocks between `d` and `e` [FR21].
- `BV(h) = Σ_{b ∈ (S_tail, h]} v(b)`, relative to the shared `S_tail` and maintained incrementally (a tail slide subtracts `v(tail)` from every head).
- Admissible head: Connected/Active with a continuous tip-ending segment of ≥ `W` blocks, i.e. `tip.sequence ≥ S_tail + W − 1` once matured (`S_head − S_tail + 1 ≥ W`); before maturity: the whole genesis-anchored chain [FR21, DEC-154].
- Active head = `argmax BV` over admissible heads; ties: larger tip `|b|`, then lower tip hash [DEC-155].
- Twin hold: a ready node never switches to a branch whose first divergent block is a same-parent twin of its own block unless that branch carries a verdict-bearing twin evidence block for its twin [FR70, FR71].
- Re-evaluation only when a non-active head is extended (including a new fork) or goes Stored → Connected. Only the affected heads are compared with the active head, once per admission: the highest-ranked admissible one [FR22, DEC-152].
- Switch at divergence point `D` [FR23, FR58, DEC-159]:
  - `D ≥ S_head − Hv + 1`, measured against the node's own head: atomic undo to `D` and forward redo with Tier 3; the mempool is reconciled per block; a failure applies FR5 to the candidate.
  - `D ≤ S_head − Hv`: the branch never becomes active and is reclaimed when it leaves the bound; ready state has no full-reconstruction path.
- Storage pressure evicts exactly one side branch, the lowest `BV` (ties: lowest tip sequence, creator id, hash), back to the shared ancestor; active-chain and horizon blocks are never evicted [FR57]. `chain_heads` overflow evicts the non-active head with the lowest `head_sequence` [FR19].

## 12. Economics (supply flows)

- Genesis: node #0 self-transfer of `initial_total_network_currency` [FR54].
- Creator credit per accepted block: `mining_reward` (minted) + fees (T1/T2 explicit `fee`, T3 `in − out`, carry-forward 0) + `replay_block_reward` per replay-bearing block (credited with no debit counterpart) [FR36].
- Burned: `registration_price` per registration; `custodian_fee` per carried UTXO per carry-forward; discarded below-fee UTXOs.
- Vote credit per transaction = `vs`, independent of amount and fee.

## 13. Genesis and trust anchor [FR54, FR69]

Block #0 (by node 0, `previous_hash = 0³²`): registration of node 0 (`new_node_id = 0`, `new_pk = pk0`) and a self-transfer of the initial supply, `vote = 0`; it bypasses balance and fee checks and sets `M := 0`. Block #1: the chain-config block with `content_sig` by `sk0`. Bootstrap exceptions apply only when sequence ∈ {0, 1} ∧ the content matches. Only node 0 can initiate genesis; it locks the configuration and enters ready directly. Any artifact asserting a node-0 key other than `pk0` is exact evidence.

## 14. Parameters (chain-config id, default) [config §4.1, §6]

| id | parameter | default | | id | parameter | default |
|---:|---|---|---|---:|---|---|
| 1 | `inter_block_interval_ms` | 60 000 | | 20 | `block_fill_threshold_percent` (θ) | 80 |
| 2 | `grace_period_window_ms` (g) | 30 000 | | 21 | `active_chain_length` (W) | 500 |
| 3 | `block_size_limit` (L) | 2016 | | 22 | `mempool_replenishment_interval_ms` | 500 000 |
| 4 | `max_block_utxo_output` | 255 | | 23 | `custodian_fee` (c) | 1 |
| 5 | `mining_reward` | 0 | | 24 | `registration_price` | 100 |
| 6 | `vote_scale` (vs) | 1000 | | 25 | `tx_fee_per_byte_min` | 0 |
| 7 | `vote_interest` (vi) | 5 | | 26 | `tx_fee_per_byte_max` | 1000 |
| 8 | `parent_recovery_per_head_retry_interval_ms` | 120 000 | | 27 | `deviation_replay_insertion_delay_ms` | 300 000 |
| 9 | `parent_recovery_min_emit_interval_ms` | 10 000 | | 28 | `replay_block_reward` | 100 |
| 10 | `required_support` (r) | 3 (⇒ m = 5) | | 29 | VM fuel limit | its ceiling |
| | | | | 30 | `verification_horizon` (Hv) | 50 |

Literal-only and bound-checked at acceptance: 3, 4, 10, 21, 30. The others may be bytecode programs reading other parameters and chain-info (today: the registered-node count). Acceptance also checks `1 ≤ r ≤ MAX_AGG`, `vs ≠ 0`, `W ≤` build capacity, `1 ≤ Hv ≤ W`, `W + 2·Hv ≤` block-storage capacity, `θ ≤ 100` and `fee_min ≤ fee_max` (literals only).

## 15. Open specification gaps

Each entry is defined in the [Open Gaps Register](./moonblokz-open-gaps-register.md):

- [OG-008] approval-evidence byte encoding and its binding to the deviation block.
- [OG-011] UTXO causality (`q < s`, no same-block spend) exists only in code.
- [OG-012] the UTXO reference bound is worded three ways.
- [OG-013] input-signature preimage of complex transactions.
- [OG-014] support-message signed content (FR12 vs FR26).
- [OG-015] grace-period time origin, and how creator entitlement `I-10` is checked.
- [OG-016] fee-policy validity predicate.
- [OG-017] no block-level `anchor ≥ S_tail` bound.
- [OG-018] carry-forward below-fee discard wording.
- Post-MVP: [OG-005] UTXO saturation / dynamic custodian fee; [OG-006] long-disconnect recovery; [OG-007] concurrent ingestion while processing; [OG-010] sequence wrap.

## 16. Security-relevant facts (stated, not evaluated)

1. Every time-dependent rule (grace admission, pacing soft-fail, support eligibility) uses the node's local clock and local arrival times; no time is chain-verifiable [FR46, FR9, FR12].
2. `consumed_votes(b)` excludes the vote credits of transactions inside `b` itself: it is `av(c)` after interest and before credits (§7, DEC-156). The credits `b` gives its creator are wiped by the reset, and those it gives anyone else count from then on.
3. No-self-vote forbids only `vote = initializer`; a vote for any other existing id is unrestricted. Each transaction adds `vs` regardless of value; `tx_fee_per_byte_min` defaults to 0.
4. Interest adds at most `vs` per node per block; with defaults it is `⌊av/200⌋`, i.e. ~0.5 % per block until `av ≥ 200·vs` (derived).
5. Support non-conflict is scoped to `(proposer, sequence)`, so one node may support deviations by different proposers at the same sequence. `A` and the seed are evaluated per branch on the supporter's local chain.
6. If `|A| < r`, no evidence can form: the threshold stays `r` while `m` is capped at `|A|` (derived). The seed contains no proposer-controlled block content; the tail hash was fixed `W` blocks earlier.
7. `supporter_vote_sum` is a closed form of chain state, not of the signer set.
8. Schnorr aggregate weights depend on `(Σ r_i mod p, m, i)` only, not on the public keys. Registration requires proof of possession `new_key_sig = Sig_{sk_new}(new_pk)` (ordinary signature type).
9. `hash(b)` covers the signature; the block signature covers the header with the signature field zeroed and `H(payload)` [FR70].
10. Block-level rules for T1 and balance inputs are `s > anchor` plus no duplicate within the retained candidate chain; `anchor ≥ S_tail` is enforced only at mempool intake and mempool reconciliation [FR14, FR32, OG-017].
11. UTXO references `(q, i)` are branch-relative: after a reorg the same pair may name a different output, and spends of outputs re-included at another position become invalid [ADR-016].
12. Outside direct head extensions and blocks whose creator key is branch-independent (DEC-157), blocks are stored before signature and state validation (opportunistic signatures, Tier 3 deferred); bounded storage is reclaimed by lowest-`BV` side-branch eviction [FR16, FR57].
13. Bootstrap acquisition picks the highest tip sequence, not value, among qualifying segments; the processing pass auto-accepts pre-seed state-dependent checks and trusts obligations and UTXO references preceding a window-anchored segment [FR2, FR3, FR6].
14. Ready nodes discard blocks with `S_new ≥ S_head + W`, and forks with `D ≤ S_head − Hv` are deleted, so a partition lasting more than `W` blocks, or one whose two sides both advance more than `Hv` blocks past the divergence, becomes a permanent fork with operator-only recovery [FR58, FR60].
15. Same-parent equivocation is held at each node's first-adopted twin, resolved by a subgroup verdict in a twin evidence block, and penalized by a `W`-block exclusion of the creator from block creation and from `A`; only the equivocating creator is provable. Same-creator blocks at one sequence on different parents are legitimate and are resolved by FR21 [FR70–FR72].
16. Configuration is immutable and anchored to the compiled-in `pk0`, so any change requires a new chain; unknown configuration ids cause rejection (firmware-capability gating) [FR8, config §3.4].
17. Branch switching is local and pairwise (triggered head vs active head); `Hv` is chain configuration and bounds the switch depth identically on every node [FR22, FR58].

## Related Documents

- [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) — authoritative wording of every FR cited here; wins on any divergence.
- [`moonblokz-blockchain-algorythm.md`](./moonblokz-blockchain-algorythm.md) — the full algorithm and data-structure model this summary compresses.
- [`moonblokz-blockchain-operating-model.md`](./moonblokz-blockchain-operating-model.md) — runtime logic by phase and event, the invariant catalog (`I-n`) and build state.
- [`blockchain-adrs/ADR-015-approval-subgroup-selection.md`](./blockchain-adrs/ADR-015-approval-subgroup-selection.md) and [`ADR-016`](./blockchain-adrs/ADR-016-sequence-indexed-utxo-input-model.md) — rationale and rejected alternatives for subgroup selection and sequence-indexed UTXO references.
- [`moonblokz-crypto-algorythm.md`](./moonblokz-crypto-algorythm.md) — the signing, verification and aggregation model behind §1.
- [`moonblokz-configuration-specification.md`](./moonblokz-configuration-specification.md) — parameter registry, acceptance checks and VM behind §2 and §14.
- [`moonblokz-open-gaps-register.md`](./moonblokz-open-gaps-register.md) — definitions of the gaps listed in §15.
- [`moonblokz-algorithmic-risk-register.md`](./moonblokz-algorithmic-risk-register.md) — the evaluation of the §16 facts: ranked risks and candidate treatments.
- [`moonblokz-glossary.md`](./moonblokz-glossary.md) — disambiguates terms used here in more than one sense (`seed`, `replay`, `score`, `head`, `tail`).
