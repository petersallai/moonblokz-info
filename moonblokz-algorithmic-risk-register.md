# MoonBlokz Algorithmic Risk Register

A living, ranked register of weaknesses in the blockchain consensus and ledger algorithm, each with candidate treatments, kept as a backlog source for sprint planning. It is **not authoritative**: the rules it evaluates are stated in the [Blockchain Algorithm Summary](./moonblokz-blockchain-algorithm-summary.md) (cited as `§n`) and decided in the [Blockchain PRD](./moonblokz-blockchain-prd.md), which wins on any divergence. Severities and treatments are analysis results, not requirements: a treatment binds only once a Project Lead decision ([Decision Register](./moonblokz-decision-register.md)) and an owner story exist.

## Threat Model Boundary

- **In scope:** any registered node or colluding set of nodes producing well-formed but adversarial blocks, transactions and support messages; unregistered radio senders injecting parseable messages; naturally lossy radio and partitions.
- **Out of scope** ([Blockchain Concept](./moonblokz-blockchain-concept.md) §Security Framing): radio flooding by a stronger adversary (the summary reads jamming into this, §0) and physical isolation with controlled misinformation. A risk that needs these is listed as *Outside threat model* only when the Project Lead keeps it for visibility, so a scope change can re-rank it.

## Rating, Status and Maintenance

- **Severity** (Critical · High · Medium · Low): impact × likelihood inside the threat model, with the §14 default parameters unless a row says otherwise.
- **Status:** *Open* · *Treatment proposed* (an `O-` row exists) · *Planned* (owner story named) · *Mitigated* (`DEC-` row and story) · *Accepted* (MVP limitation with its record) · *Outside threat model*.
- New risks take the next `AR-` number; numbers are never reused or renumbered, and a withdrawn risk's number stays retired (AR-015, withdrawn by the Project Lead on 2026-10-20). A mitigated or accepted risk keeps its row. A risk is re-rated when a rule or default it depends on changes, or when a simulation in the radio simulator replaces a model estimate below.

## Register

Ordered by severity at the time of entry.

| ID | Risk | Severity | Treatment options | Status · Owner | Sources |
|---|---|---|---|---|---|
| AR-001 | A creator can re-include a transfer or balance input whose original block has left the window. | High | Block-level `anchor ≥ S_tail` for node transfers and balance inputs. | Open · — | §6, §16.10; [OG-017](./moonblokz-open-gaps-register.md) |
| AR-002 | Creator equivocation is neither detected nor penalized; the creator picks the winning twin. | High | Adopted: same-parent twins are held at the first-adopted twin, resolved by a subgroup verdict recorded in a twin evidence block, and the equivocator is excluded from block creation for `W` blocks (DEC-162..165). | Mitigated · DEC-162..165, 6.12, 6.13 | §11, §16.15; BC-FR21, BC-FR70–FR72 |
| AR-003 | A colluding pair concentrates accumulated vote and dominates block creation on low-traffic chains. | High (Medium at high traffic) | Credit at most one `vs` per vote target per block; alternatives: vote weight tied to fee, no credit for votes the creator includes for its partner. | Open · — | §7, §12, §16.3, §16.4; BC-FR37, BC-FR55 |
| AR-004 | Support signatures may not bind to proposer and deviating block. | High | Sign `(proposer, deviation sequence, deviating block hash)`. | Treatment proposed · O-002, story 6.4 | §2, §9; [OG-014](./moonblokz-open-gaps-register.md) |
| AR-005 | An approval-less deviation block can outweigh the original creator's block through its own vote credits. | Medium | Adopted: `consumed_votes` is the post-interest, pre-credit snapshot, so `d` is valued before its own credits and is never higher than `u`'s normal block (equal votes tie); verified at Tier 3 (DEC-156). | Mitigated · DEC-156, story 6.2 | §7, §9, §16.2; BC-FR28 |
| AR-006 | An approved deviation overturns about `r` normal blocks. | Medium | Evidence value independent of `m`; support only for direct head extensions; published confirmation depth `⌊m/2⌋ + 2`. | Open (O-012 partial) · — | §9, §11, §16.7, §16.11; BC-FR12 (a) |
| AR-007 | The `required_support` default (3, so `m = 5`) is not calibrated. | Medium | Per-backend defaults from evidence capacity, security and radio cost. | Open · O-023 | §1, §9, §14; [crypto concept](./moonblokz-crypto-concept.md) §Bounded aggregation |
| AR-008 | Approval liveness depends on the reachable share of `A`, which includes intermittent transaction senders. | Medium | Define `A` by liveness evidence (block or support creators); document minority-side halting as intended. | Open · — | §9, §16.6; ADR-015 |
| AR-009 | Unauthenticated Stored heads evict legitimate side branches from `chain_heads`. | Medium | Evict Stored before Connected heads; bound Stored head sequence against `S_head`; verify the creator signature whenever the key is derivable. | Open (DEC-157 partial: the creator signature at ready-state intake for branch-independent keys) · 6.2 | §4, §5, §11, §16.12; BC-FR19 |
| AR-010 | UTXO saturation griefing also inflates supply. | Medium | Post-MVP dynamic custodian fee; consider funding `replay_block_reward` from custodian fees. | Accepted · OG-005 | §10.3, §12; [OG-005](./moonblokz-open-gaps-register.md) |
| AR-011 | Mempool-replenishment requests are amplified, and a CRC32 collision suppresses a target transaction. | Low | Rate-limit responses per requester; randomized response suppression. | Open · — | BC-FR43, BC-FR46 |
| AR-012 | Node #0 may vote for itself for the chain's lifetime. | Low | Limit the exemption to the period before the first non-genesis registration. | Open · — | §6, §13; BC-FR6, BC-FR37, BC-FR54 |
| AR-013 | The tail-block creator can grind the subgroup seed of its own later proposals. | Low | Accept, or widen the seed with further past block hashes. | Open · — | §9, §16.6; ADR-015 |
| AR-014 | A collecting or restarting node can be captured by a forged high-sequence segment. | Medium if re-scoped | Reject seedless creator signatures, or add a checkpoint; sequence-plausibility bound in collecting. | Outside threat model (Project Lead, 2026-10-20) | §4, §16.13; BC-FR2, BC-FR3, BC-FR59 |
| AR-016 | A partition longer than `W`, or one whose two sides both advance more than `H` blocks past their divergence point, becomes a permanent fork; transactions stay valid on both sides. | Medium | Post-MVP long-disconnect recovery; a larger `H` widens the reconcilable range at storage cost. | Accepted · OG-006, DEC-159 | §16.14; [OG-006](./moonblokz-open-gaps-register.md); BC-FR58 |
| AR-017 | A ready node's deep-zone chain switch ran the window-anchored FR3 pass, whose pre-seed auto-acceptance let a branch define its own starting balances, votes, keys and carry-forwards. | High | Adopted: no deep zone; every switch is an in-place walk within `H` blocks of the head (DEC-159). | Mitigated · DEC-159, 9.7 | §4, §11; BC-FR3, BC-FR58 |
| AR-018 | Until Story 9.7 retains the verification horizon, a chain switch inside the bound (and an FR3 adoption) trusts the FR48 binding and the below-tail UTXO references of replayed blocks at positions up to `S_head`. | Low (Medium if a colluding set can outweigh the honest branch at low traffic) | Horizon retention `R = H` (9.7). | Planned · DEC-166, 9.7 | §10, §11; BC-FR23, BC-FR39, BC-FR58 |
| AR-019 | Registered nodes (valid signature, forged `consumed_votes`) and creators registered above the common ancestor can force failed chain-switch attempts. | Low | DEC-170's pre-pass covers fork-independent creators; rate limiting is not specified. | Open · — | §5, §11; BC-FR9, BC-FR23 |

## Detail Notes

### AR-001 — Out-of-window re-inclusion

Block-level rules for a node transfer or balance input are `s > anchor` plus no duplicate within the retained candidate chain; `anchor ≥ S_tail` holds only at mempool intake and reconciliation (§6, §16.10). Once the original block drops below `S_tail`, a creator may include the same signed transaction again: the initializer pays twice, the receiver and the creator gain. Any node reaches the creator role through rotation. The current code's Tier 1 check enforces only `block.sequence > anchor_sequence`.

### AR-002 — Creator equivocation

A creator signs two blocks at the same sequence, one paying a merchant and one spending the same funds elsewhere. With equal value, the larger serialized size wins the FR21 tie-break, so every node that receives both converges on the creator's choice. Depth is one block with an honest next creator, more under AR-003.

**Treatment (DEC-162..165, BC-FR70–FR72).** Only same-parent twins count: the twin at the divergence point is unavoidable for a profitable fork, because the creator entitled there is unique. A ready node never switches to the other twin without a verdict, so a late reveal fails outright. A subgroup chosen at the shared parent votes for its first-adopted twin, and the entitled creator records the proof and, with a quorum, the bitmap verdict in a twin evidence block; the verdict brings stranded nodes back. The equivocator is excluded from block creation and from `A` for `W` blocks. Residuals: without a quorum the hold stays and nodes that adopted the attacker's twin first stay stranded until a restart; only the equivocator is provable, never its accomplices, who sign on one branch only; and each attempt costs about 99 votes of radio time (AR-007, AR-011), which the exclusion limits per identity.

### AR-003 — Vote concentration by a colluding pair

The creator alone selects the transactions, and therefore the vote credits, of its block. Each transaction credits `vs` independently of amount and fee (`tx_fee_per_byte_min = 0`), and fees from the creator's own transactions return to it. A 2016 B block holds about 18 node transfers. Interest adds only `⌊av/200⌋` per block and nothing to a node at `av = 0` (every creator after its block), so it does not offset 18 concentrated credits.

| Active nodes | Honest tx/block | Pair's share of blocks (fair: `2/(N+2)`) |
|---|---|---|
| 30 / 100 / 300 | 18 (full blocks) | 60% / 8.7% / 2.3% |
| 30 / 100 / 300 | 3 (low traffic) | 96% / 87% / 60% |

The pair votes only inside its own blocks. If it also buys space in honest blocks, it takes over 90% even at 1,000 nodes, at a fee cost. With at most one credit per target per block, the pair falls to 9% / 2.9% / 0.8% at low traffic, close to the fair share.

### AR-005 — Own-credit deviation value

Before DEC-156, `consumed_votes(d)` was the creator's vote after step 2 of §7, so it included the credits of `d`'s own transactions. With partner transactions voting for the fallback creator `x`, `v(d)` can grow by about 18 `vs` and exceed `av(u)`. The BC-FR28 premise that an approval-less deviation branch is weaker then fails, and the original creator's late block no longer wins. The same credits also enlarge the margin AR-006 measures. **Treatment (DEC-156):** both header snapshots exclude the block's own credits, while the FR37 step order, and with it the reset of a creator's own-block credits, stays unchanged. So `v(d) = I(av(x)) ≤ I(av(u))` by FR38, and the own-credit part of AR-006's margin is gone. Applying the credits after the reset instead was rejected: it would let a creator keep the credits it selects into its own block, shrinking AR-003 to a single node and worsening AR-012.

### AR-006 — Approval reorg depth

The deviation branch gains `(⌊m/2⌋ + 2)·av(x)` over the normal branch at the deviation point (§9). Simulated with the AR-003 vote model, the number of later normal blocks it still overturns is stable across network size and traffic:

| `r` | 3 | 10 | 25 | 50 | 100 |
|---|---|---|---|---|---|
| Overturned normal blocks (median) | 3 | 10 | 25 | 50 | 99 |

Paths to a successful deviation:

- **Partition healing.** The side lacking the original creator gathers `r` supports; on heal it overturns up to about `r` blocks of the other side.
- **Subgroup majority.** An attacker holding share `q` of `A` controls at least `r` of the `m` seats with this probability:

| `q` | `r = 10` | `r = 25` | `r = 50` |
|---|---|---|---|
| 0.2 | 0.16% | ~0 | ~0 |
| 0.3 | 3.3% | 0.17% | ~0 |
| 0.4 | 19% | 7.8% | 2.2% |

Spends of outputs re-included at another position become invalid rather than re-includable (§16.11).

Time no longer limits the attack at realistic `r`. Support collection takes minutes, during which the normal branch gains fewer than `r` blocks. The `deviation_replay_insertion_delay_ms` pause (300 s) is short against `r` blocks.

**Bound by DEC-159.** A switch is admitted only within `H` blocks of the switching node's own head, so an approved deviation that overturns more than `H` normal blocks would leave the nodes on the normal branch unable to switch while the deviation's nodes stay, splitting the network. At the default `r = 3` the depth (about 3) is far below the default `H = 50`; a larger `r` needs `H` above the depth this table shows, or the evidence-value treatment above.

### AR-007 — `required_support` calibration

The Project Lead stated that the default must follow:

- **Schnorr:** the evidence-block capacity, about 36 B per supporter, at most about 50.
- **BLS:** security and practicality, because its evidence capacity (about 450) is not binding.

The binding cost is radio. All `m = 2r − 1` subgroup members send a support message of about 225 B. At a channel throughput of about 30–50 B/s, before relaying:

| `r` | 10 | 25 | 50 | 100 |
|---|---|---|---|---|
| Message volume | 4 KB | 11 KB | 22 KB | 44 KB |
| Whole-channel time | 1.5–2.5 min | 4–6 min | 7–12 min | 15–25 min |

A realistic range is therefore about 10–25 for Schnorr and 25–50 for BLS, to be confirmed in the radio simulator. Raising `r` lowers AR-006 likelihood but raises its depth, unless AR-006's decoupling is adopted.

### AR-008 — Approval liveness

Above a deviation only the proposer may build, and only replay or evidence blocks (§9), so a failed approval stalls ordinary progress. Each grace expiry admits the next candidate with a fresh subgroup. The tail hash and sequence are frozen during the stall, so each reachable proposer is one try.

Model: `|A| = 400`, success probability over all reachable proposers, with the expected wait to the first success:

| Reachable share of `A` | 0.3 | 0.4 | 0.45 | 0.5 |
|---|---|---|---|---|
| `r = 10` | 97%, ~51 min | 100%, ~7 min | 100%, ~3 min | 100%, ~2 min |
| `r = 25` | 9%, ~99 min | 100%, ~19 min | 100%, ~5 min | 100%, ~2 min |
| `r = 50` | ~0% | 80%, ~75 min | 100%, ~9 min | 100%, ~2 min |

A side holding less than about 40% of `A` halts until the partition heals. This protects safety, because no competing history is built. Because `A` counts every transaction sender in the window, intermittently awake devices can push the whole network below the threshold whenever the top creator is absent. With `|A| < r`, no evidence can form at all (§16.6).

### AR-009 — `chain_heads` eviction

Outside a ready direct head extension, a block is stored before its signature or state is checked (§5, §16.12). The `chain_heads` table holds `MAX_BRANCH_COUNT = 40` heads, and overflow evicts the non-active head with the lowest `head_sequence` (§11). About 40 forged blocks therefore displace every legitimate side branch:

- **Ready:** sequences just below `S_head + W`.
- **Collecting:** any sequence (§5).

The forged heads also draw parent-recovery requests (BC-FR46).

**Partial treatment (DEC-157, story 6.2):** in ready state the creator signature is verified at intake, before the write, whenever the creator's key is branch-independent: node #0, or a node seeded at or below `S_tail`. Forged blocks claiming such a creator are no longer stored. Still open: forgeries naming a creator seeded inside the window or an unknown id, forgeries by registered nodes, and the collecting case.

**Further treatment (DEC-170, story 6.3):** before a chain switch mutates anything, it verifies every unmarked new-branch block whose creator is fork-independent at the common ancestor, so a forgery by an unregistered sender naming any creator registered on the shared prefix no longer costs a walk. The residual is AR-019.

### AR-011 — Replenishment abuse

Every receiver of a replenishment request answers with one transaction, and no response rate limit is specified (BC-FR43, BC-FR46). A single forged request therefore costs many transmissions. A transaction ground (about 2³² hashes) to the `hash_crc32` of a target, once held by nodes, makes them skip the target in every exchange.

### AR-014 — Acquisition capture (outside threat model)

The ingredients are:

- collecting selects the highest tip sequence among qualifying segments and has no window bound (§4, §16.13);
- the pass auto-accepts pre-seed state-dependent checks, and how a seedless creator's block signature is verified is not specified (§4);
- a restart re-enters collecting (§4).

The attack needs radio-local presence, because ready nodes neither store nor relay out-of-window blocks (BC-FR60), and the forged segment must qualify before the genuine one. The victim then discards the real chain as out-of-window.

### AR-018 — Below-tail trust inside the bound (until 9.7)

A replayed block at position `s ≤ S_head` drops block `s − W`, which lies below `S_tail`; until the horizon is retained, neither a switch nor an FR3 adoption can check that binding, nor the spentness and amount of a UTXO reference in `[s − W, S_tail)`. A winning branch could therefore have dropped essential state, double-spend a below-tail output, or declare an amount for a non-existent output. Honest followers held those blocks in their window and would have deleted such blocks, so only a colluding set whose branch outweighs the honest one can build it (AR-003). DEC-159 bounds the depth by `H`.

### AR-019 — Forced switch attempts

A registered node can sign side blocks with forged `consumed_votes` that win FR22 and fail only at Tier 3 in the replay; so can a creator registered above the common ancestor, whose key the pre-pass cannot check. Each attempt costs the victim about `2·j_old + j_new` reads and undos plus the seed-source scans, and deletes the attacker's blocks, so the cost is linear in the attacker's signed blocks.

### AR-016 — Permanent forks

Under DEC-159 a node switches only to a branch that diverges within `H` blocks of its own head. A partition side that cannot reach the FR12 approval quorum stalls at its first absent creator, so it can still adopt the other side's chain within the FR60 window. The fork becomes permanent before `W` only when both sides keep advancing — both must complete approvals, which needs a near-even split — and both pass `H` blocks beyond the divergence point.

### AR-017 — Deep-zone trust downgrade (mitigated)

Before DEC-159, a ready node switched to a branch diverging more than `H` blocks below its head through the window-anchored FR3 pass (the FR58 deep zone). The pass seeded every node without a seed in the shared prefix from the branch's own balance entries — balance, vote count and public key — and trusted replay obligations and UTXO references below the segment (§4). A branch could therefore define its own starting state: inflated vote counts give `consumed_votes` that win FR21, and zero-input carry-forwards for dropped blocks create money. The pass's justification, attestation by a continuous signed chain of length `≥ W`, is circular, because signatures and votes are checked against keys and seeds the branch itself supplies. **Treatment (DEC-159):** no deep zone; every switch is an in-place FR23 walk within `H` blocks of the head, so every check runs on the node's own verified state.

## Analysis Models

The figures above come from simplified models, not from the radio simulator:

- **Vote model:** BC-FR37 steps 1–3 with `vs = 1000` and `vi = 5`. The creator is `argmax (av, −id)`. 18 or 3 honest transactions per block, with uniformly random vote targets (fixed three-neighbour targets give the same results). 20,000–30,000 blocks. No grace periods, radio delay or fee priority.
- **Reorg depth:** the largest `k` with `Σ v(next k normal blocks) < (⌊m/2⌋ + 2)·av(x)`.
- **Subgroup majority:** binomial over `m` seats.
- **Retries:** hypergeometric draws from `A`, one try per reachable proposer per grace step.

## Not Yet Assessed

The following §16 facts are not yet analyzed:

- 16.5: several proposers can be supported at the same sequence (competing approved deviations).
- 16.8: Schnorr aggregate weights are independent of the public keys, and the aggregate check is x-only.
- 16.17: branch re-evaluation is pairwise.
- §5: the pacing soft-fail relies on local arrival times.

## Related Documents

- [`moonblokz-blockchain-algorithm-summary.md`](./moonblokz-blockchain-algorithm-summary.md) — the rule statement every row cites as `§n`, including the §16 facts evaluated here.
- [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) — authoritative wording of every cited FR; a treatment changes the PRD only through a decision.
- [`moonblokz-decision-register.md`](./moonblokz-decision-register.md) — the `O-` and `DEC-` rows that carry treatment decisions and owner stories.
- [`moonblokz-open-gaps-register.md`](./moonblokz-open-gaps-register.md) — specification gaps and accepted limitations that several risks depend on.
- [`blockchain-adrs/ADR-015-approval-subgroup-selection.md`](./blockchain-adrs/ADR-015-approval-subgroup-selection.md) — subgroup rule and rejected alternatives behind AR-006 to AR-008 and AR-013.
- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) — §Security Framing, the source of the threat-model boundary.
- [`moonblokz-simulator-concept.md`](./moonblokz-simulator-concept.md) — the tool for replacing model estimates with radio-level measurements.
