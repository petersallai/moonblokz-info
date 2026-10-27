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
| AR-001 | A creator can re-include a transfer or balance input whose original block has left the window. | High | Adopted: block-level `anchor ≥ max(0, s − W)` (the parent's window tail) for node transfers and balance inputs, which makes the window duplicate check complete (DEC-174). | Mitigated · DEC-174, 7.3 | §6, §16.10; BC-FR6, BC-FR9, BC-FR14 |
| AR-002 | Creator equivocation is neither detected nor penalized; the creator picks the winning twin. | High | Adopted: same-parent twins are held at the first-adopted twin, resolved by a subgroup verdict recorded in a twin evidence block, and the equivocator is excluded from block creation for `W` blocks (DEC-162..165). | Mitigated · DEC-162..165, 6.12, 6.13 | §11, §16.15; BC-FR21, BC-FR70–FR72 |
| AR-003 | A colluding pair concentrates accumulated vote and dominates block creation on low-traffic chains. | High (Medium at high traffic) | Adopted: at most one `vs` credit per vote target per block, with the `vote_interest` default raised to 50 (DEC-176); the pair falls to about its fair share. | Mitigated · DEC-176, 7.2 | §7, §12, §16.3, §16.4; BC-FR37, BC-FR55 |
| AR-004 | Support signatures may not bind to proposer and deviating block. | High | Sign `(proposer, deviation sequence, deviating block hash)`. | Treatment proposed · O-002, story 6.4 | §2, §9; [OG-014](./moonblokz-open-gaps-register.md) |
| AR-005 | An approval-less deviation block can outweigh the original creator's block through its own vote credits. | Medium | Adopted: `consumed_votes` is the post-interest, pre-credit snapshot, so `d` is valued before its own credits and is never higher than `u`'s normal block (equal votes tie); verified at Tier 3 (DEC-156). | Mitigated · DEC-156, story 6.2 | §7, §9, §16.2; BC-FR28 |
| AR-006 | An approved deviation overturns about `r` normal blocks. | Medium | Evidence value independent of `m`; support only for direct head extensions; published confirmation depth `⌊m/2⌋ + 2`. | Open (O-012 partial) · — | §9, §11, §16.7, §16.11; BC-FR12 (a) |
| AR-007 | The `required_support` default (3, so `m = 5`) is not calibrated. | Medium | Adopted: default 40, below the global aggregation cap of 50 and the FR58 `H = 50`, with DEC-172's `r_eff` for small networks; the radio cost still to be confirmed in the radio simulator (DEC-173). | Mitigated · DEC-173 | §1, §9, §14; [crypto concept](./moonblokz-crypto-concept.md) §Bounded aggregation |
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
| AR-020 | A coalition that stays the entitled creator for many consecutive blocks on its own private branch — through self-credited votes or members arranged next to each other in the creator order — builds a fully valid heavier branch and reorganizes the chain up to the FR58 bound. | Medium since DEC-176 (High before, at `vi = 5` and low traffic) | Adopted: `vote_interest` default 50 with the `vs` cap kept, and the per-target credit cap (DEC-176), which close the natural-fork route for `N ≳ 100`. Open: price votes (`tx_fee_per_byte_min > 0`); a published confirmation depth above any feasible adjacent cluster. Network-size-adaptive `vote_interest` is post-MVP (O-031). The FR70 twin hold already defeats the route that starts with a same-parent twin. | Open · DEC-176 (partial) | §7, §9, §11, §16.3, §16.4; BC-FR21, BC-FR37, BC-FR38, BC-FR70 |
| AR-021 | Coalition members that hold consecutive positions in the creator order and stay silent force a grace expiry for every skipped rank and an approved deviation for every slot, stalling the chain at no cost to themselves; they climb back by voting for each other. | Medium (High at a large `required_support` without the treatments) | Reset every skipped top at a deviation; exclude a node named as `first_voted_node` in the window from creation for `W` blocks; a two-level grace period (short when the admitted proposer issued nothing, long while a deviation is pending). | Open · — | §9, §16.6; BC-FR12, BC-FR37, BC-FR44, BC-FR47 |

## Detail Notes

### AR-001 — Out-of-window re-inclusion

Block-level rules for a node transfer or balance input are `s > anchor` plus no duplicate within the retained candidate chain; `anchor ≥ S_tail` holds only at mempool intake and reconciliation (§6, §16.10). Once the original block drops below `S_tail`, a creator may include the same signed transaction again: the initializer pays twice, the receiver and the creator gain. Any node reaches the creator role through rotation. The current code's Tier 1 check enforces only `block.sequence > anchor_sequence`.

Adopted 2026-10-27 (DEC-174): a block of sequence `s` requires `anchor ≥ max(0, s − W)` for every node transfer and balance input, the window tail of its parent and the bound the mempool admitted the transaction against. An earlier inclusion at `s₀` satisfies `s₀ > anchor ≥ s − W`, so it lies inside the window of `s` and the existing duplicate check finds it. The rule needs only the block's sequence and `W`, so every branch evaluates it the same way; it is Tier 1 once the configuration is locked, and the FR3 pass checks it before the lock against the candidate's own `W`. Residuals: the window-anchored FR3 pass cannot see duplicates before its segment, which it trusts like other pre-window history (DEC-129); a transaction can still be valid on both sides of a partition (AR-016). Story 7.3 adds the check beside the existing one.

### AR-002 — Creator equivocation

A creator signs two blocks at the same sequence, one paying a merchant and one spending the same funds elsewhere. With equal value, the larger serialized size wins the FR21 tie-break, so every node that receives both converges on the creator's choice. Depth is one block with an honest next creator, more under AR-003.

**Treatment (DEC-162..165, BC-FR70–FR72).** Only same-parent twins count: the twin at the divergence point is unavoidable for a profitable fork, because the creator entitled there is unique. A ready node never switches to the other twin without a verdict, so a late reveal fails outright. A subgroup chosen at the shared parent votes for its first-adopted twin, and the entitled creator records the proof and, with a quorum, the bitmap verdict in a twin evidence block; the verdict brings stranded nodes back. The equivocator is excluded from block creation and from `A` for `W` blocks, with its accumulated vote held at 0 so it returns without banked credits (DEC-175). Residuals: without a quorum the hold stays and nodes that adopted the attacker's twin first stay stranded until a restart; only the equivocator is provable, never its accomplices, who sign on one branch only; and each attempt costs about 99 votes of radio time (AR-007, AR-011), which the exclusion limits per identity.

### AR-003 — Vote concentration by a colluding pair

The creator alone selects the transactions, and therefore the vote credits, of its block. Each transaction credits `vs` independently of amount and fee (`tx_fee_per_byte_min = 0`), and fees from the creator's own transactions return to it. A 2016 B block holds about 18 node transfers. Interest adds only `⌊av/200⌋` per block and nothing to a node at `av = 0` (every creator after its block), so it does not offset 18 concentrated credits.

| Active nodes | Honest tx/block | Pair's share of blocks (fair: `2/(N+2)`) |
|---|---|---|
| 30 / 100 / 300 | 18 (full blocks) | 60% / 8.7% / 2.3% |
| 30 / 100 / 300 | 3 (low traffic) | 96% / 87% / 60% |

The pair votes only inside its own blocks. If it also buys space in honest blocks, it takes over 90% even at 1,000 nodes, at a fee cost. With at most one credit per target per block, the pair falls to 9% / 2.9% / 0.8% at low traffic, close to the fair share.

**Treatment (DEC-176).** The per-target credit cap, with `vote_interest` 50. The cap separates collusion from popularity: the pair's 18 credits collapse to one, while an honestly popular node rarely receives two votes in one block. Vote steering, measured as the block share of the 20 % of nodes that receive 80 % of the votes (vote-proportional target about 80 %; the interest pulls it down by design), at low / high traffic:

| `N` | `vi = 5`, no cap (before) | `vi = 5`, cap | `vi = 50`, cap (adopted) |
|---|---|---|---|
| 30 | 71% / 72% | 70% / 57% | 61% / 43% |
| 100 | 71% / 68% | 70% / 64% | 55% / 35% |
| 300 | 63% / 55% | 63% / 53% | 52% / 28% |

The cap alone fixes the pair at any `vi`; `vi = 50` is the price of AR-020. Higher `vi` costs little more steering (255: 52% / 31% at `N = 100`).

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

**Bound by DEC-159.** A switch is admitted only within `H` blocks of the switching node's own head, so an approved deviation that overturns more than `H` normal blocks would leave the nodes on the normal branch unable to switch while the deviation's nodes stay, splitting the network. At the default `r = 40` (DEC-173) an approval outweighs about 40 normal blocks, about 10 below the default `H = 50`: a normal branch that advanced `H` blocks is heavier than the approved branch in the median case, so the nodes on the approved branch, which stay within their own bound, switch to it instead of splitting. The margin narrows the overlap rather than closing it, since the overturn depth varies around its median; the full fixes stay open (the evidence value independent of `m`, or a larger `H`).

### AR-007 — `required_support` calibration

The Project Lead stated that the default must follow:

- **Schnorr:** the evidence-block capacity, about 36 B per supporter, at most about 50.
- **BLS:** security and practicality, because its evidence capacity (about 450) is not binding.

The binding cost is radio. All `m = 2r − 1` subgroup members send a support message of about 225 B. At a channel throughput of about 30–50 B/s, before relaying:

| `r` | 10 | 25 | 50 | 100 |
|---|---|---|---|---|
| Message volume | 4 KB | 11 KB | 22 KB | 44 KB |
| Whole-channel time | 1.5–2.5 min | 4–6 min | 7–12 min | 15–25 min |

A realistic range is therefore about 10–25 for Schnorr and 25–50 for BLS, to be confirmed in the radio simulator. The Project Lead set the default to 40 on 2026-10-27 (DEC-173), below the cap of 50 to keep the AR-006 depth under `H = 50`; with O-002's 113 B support message an approval round (79 messages) is about 9 KB. Since DEC-172 the value is a cap: a voting set smaller than `2r − 1` approves by simple majority, so a large `r` no longer halts a small network. Raising `r` lowers AR-006 likelihood but raises its depth, unless AR-006's decoupling is adopted.

### AR-008 — Approval liveness

Above a deviation only the proposer may build, and only replay or evidence blocks (§9), so a failed approval stalls ordinary progress. Each grace expiry admits the next candidate with a fresh subgroup. The tail hash and sequence are frozen during the stall, so each reachable proposer is one try.

Model: `|A| = 400`, success probability over all reachable proposers, with the expected wait to the first success:

| Reachable share of `A` | 0.3 | 0.4 | 0.45 | 0.5 |
|---|---|---|---|---|
| `r = 10` | 97%, ~51 min | 100%, ~7 min | 100%, ~3 min | 100%, ~2 min |
| `r = 25` | 9%, ~99 min | 100%, ~19 min | 100%, ~5 min | 100%, ~2 min |
| `r = 50` | ~0% | 80%, ~75 min | 100%, ~9 min | 100%, ~2 min |

A side holding less than about 40% of `A` halts until the partition heals. This protects safety, because no competing history is built. Because `A` counts every transaction sender in the window, intermittently awake devices can push the whole network below the threshold whenever the top creator is absent. Before DEC-172 no evidence could form at all with `|A| < r`; since then a small voting set approves by simple majority, so a side still halts only when less than a majority of `A'` is reachable.

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

### AR-016 — Permanent forks

Under DEC-159 a node switches only to a branch that diverges within `H` blocks of its own head. A partition side that cannot reach the FR12 approval quorum stalls at its first absent creator, so it can still adopt the other side's chain within the FR60 window. The fork becomes permanent before `W` only when both sides keep advancing — both must complete approvals, which needs a near-even split — and both pass `H` blocks beyond the divergence point.

### AR-017 — Deep-zone trust downgrade (mitigated)

Before DEC-159, a ready node switched to a branch diverging more than `H` blocks below its head through the window-anchored FR3 pass (the FR58 deep zone). The pass seeded every node without a seed in the shared prefix from the branch's own balance entries — balance, vote count and public key — and trusted replay obligations and UTXO references below the segment (§4). A branch could therefore define its own starting state: inflated vote counts give `consumed_votes` that win FR21, and zero-input carry-forwards for dropped blocks create money. The pass's justification, attestation by a continuous signed chain of length `≥ W`, is circular, because signatures and votes are checked against keys and seeds the branch itself supplies. **Treatment (DEC-159):** no deep zone; every switch is an in-place FR23 walk within `H` blocks of the head, so every check runs on the node's own verified state.

### AR-018 — Below-tail trust inside the bound (until 9.7)

A replayed block at position `s ≤ S_head` drops block `s − W`, which lies below `S_tail`; until the horizon is retained, neither a switch nor an FR3 adoption can check that binding, nor the spentness and amount of a UTXO reference in `[s − W, S_tail)`. A winning branch could therefore have dropped essential state, double-spend a below-tail output, or declare an amount for a non-existent output. Honest followers held those blocks in their window and would have deleted such blocks, so only a colluding set whose branch outweighs the honest one can build it (AR-003). DEC-159 bounds the depth by `H`.

### AR-019 — Forced switch attempts

A registered node can sign side blocks with forged `consumed_votes` that win FR22 and fail only at Tier 3 in the replay; so can a creator registered above the common ancestor, whose key the pre-pass cannot check. Each attempt costs the victim about `2·j_old + j_new` reads and undos plus the seed-source scans, and deletes the attacker's blocks, so the cost is linear in the attacker's signed blocks.

### AR-020 — Private-branch reorg by a vote-dominant coalition

Blocks cost nothing to make (no work, no timestamp, §16.1), so the length a private branch needs is no barrier; the barrier is that every block needs the creator entitled under the branch's own vote state. On a private branch the honest nodes never create, so their accumulated votes are never reset and grow only by interest, while the coalition credits its next creator with up to about 18 `vs` per block.

| Setting (vote model below, 3 honest tx/block) | 51-block private branch |
|---|---|
| `vi = 5` (default before DEC-176), pair, `N ≤ 300` | at every fork point, with a higher branch value |
| `vi ≈ 5·vs/N` (50 at `N = 100`, 17 at `N = 300`), coalition ≤ 10 % of `A`, no approvals | 0 % |
| same, coalition of 20 % of `A`, approvals with proposer grinding, `r = 3` | about 50 % per attempt at `N = 300` |
| same, `r = 50` | ≤ 7·10⁻⁹³ per attempt |
| coalition holding the top `c` creator positions | exactly `c` blocks, whatever `vi` and `r` |

**Treatment (DEC-176).** With the per-target credit cap the coalition spreads its block's credits over up to 18 members instead of one, so the cap alone does not stop it; `vi` does. Approvals are closed to a minority since DEC-172 (`r_eff` is a subgroup majority). Longest private run from natural fork points, 300 fork points, the coalition not colluding on the public chain (its better strategy):

| `vi` (with cap) | `N = 30` | `N = 100` | `N = 300` | `N = 1000` |
|---|---|---|---|---|
| 5 | 51 | 51 | 51 | 5 |
| 20 | 46 | 51 | 5 | 4 |
| 50 (adopted) | 14 | 6 | 5 | 4 |
| 255 | 6 | 4 | 5 | — |

Coalition of 20 % of the nodes; a pair stays at 2 blocks with the cap at every `vi`. The protection threshold is about `N·vi/vs ≳ 5`, and the vote steering at the threshold is the same at every `N` (about 55 % / 34 %, AR-003), so a network-size-adaptive `vi` would gain only a few points of steering at large `N` and the stronger bound below `N = 100`. It was not adopted: its only chain-info input, the registered-node count, can be inflated with dead sybil registrations to lower `vi` for everyone (at `N = 100`, 100 sybils halve it and reopen 51-block runs), and it needs a bound per-block call site and a window-anchored pass that can evaluate it (O-031). A network below about 100 nodes may declare a higher literal. The adjacent-cluster route is untouched.

Arranging members next to each other is feasible: members park at zero accumulated vote after creating (no interest on zero), then restart together on their own zero-fee votes. A heuristic strategy reached 50 adjacent members in 0.3–1.6 days at `N = 100` and 10–18 days with 55–60 members at `N = 1000` (none within 40 000 blocks at `N = 300`, which reflects the heuristic, not a bound). Paths to a reorg and what stops them today:

- **Twin at the divergence point** (the creator signs the public and the private block): held by FR70, resolved by FR71.
- **Withhold the public slots:** honest approved deviations fill them and their evidence outweighs the private branch under the current evidence value (AR-006); lowering that value without the rule that an approved deviation forfeits the absent creator's slot would reopen a late-release double spend.
- **No twin, no withholding:** the coalition must out-create the honest chain inside the FR58 bound, which the table above bounds.

Removing the `vs` cap from the interest step adds nothing against this and lets an accumulated vote overflow `u32` (a chain halt) once about 250–300 nodes climb at `vi = 50`.

### AR-021 — Silent coalition stall

A grace expiry admits one more rank (§9), and a deviation resets only the skipped top (`first_voted_node`). A cluster of `n` silent members ahead of the first honest node therefore costs about `n` expiries before the first deviation, then one approval per member. With a block propagation median `t₅₀` of 33 s on a 100-node measurement, an approval round is about 1–1.5 minutes and a safe grace period about 4–5·`t₅₀` (about 2.5 minutes, so that a pending evidence reaches a majority before the next rank is admitted). A 50-member cluster then stalls the chain for about 2 hours per cycle, about 40 minutes with a short grace (about 1.5·`t₅₀`) when the admitted proposer issued nothing; an approval also needs a majority of `A` reachable (AR-008). The members lose only their accumulated vote and climb back on each other's votes, so the stall repeats every cycle. Scattered silent members cost one expiry and one approval each.

## Analysis Models

The figures above come from simplified models, not from the radio simulator:

- **Vote model:** BC-FR37 steps 1–3 with `vs = 1000` and `vi = 5`. The creator is `argmax (av, −id)`. 18 or 3 honest transactions per block, with uniformly random vote targets (fixed three-neighbour targets give the same results). 20,000–30,000 blocks. No grace periods, radio delay or fee priority.
- **Reorg depth:** the largest `k` with `Σ v(next k normal blocks) < (⌊m/2⌋ + 2)·av(x)`.
- **Subgroup majority:** binomial over `m` seats.
- **Retries:** hypergeometric draws from `A`, one try per reachable proposer per grace step.
- **Private branch (AR-020):** the vote model with `N` up to 1000, a coalition of `c` members, a fork at every point where a member is top; on the private branch only coalition transactions (18 per block, all crediting the next member), honest nodes gain interest only; an honest top needs an approved deviation, counted with the probability that the coalition holds `r` of the `2r − 1` subgroup seats (hypergeometric, grinding over members' proposals where OG-015 allows it). 300–600 fork points per setting.
- **Clustering (AR-020):** park → synchronise → maintain, spending the coalition's own block credits and the free slots of honest blocks on zero-fee votes; success when the top 50 creator positions are all coalition; 40 000 blocks per setting.
- **Interest sweep (DEC-176):** the vote model with `vi` ∈ {5, 10, 20, 50, 100, 255}, with and without the per-target cap; 36 000–60 000 blocks per setting; steering with 20 % of nodes drawing 80 % of the vote targets; offline nodes (10 % and 30 %) for the deviation rate, which does not depend on `vi`; a greedy attacker spending every vote of every block to starve one node, for the accumulated-vote ceiling (about `N·vs`).
- **Stall (AR-021):** expiries and approval rounds counted from the block propagation measurement (`t₅₀` 33 s, `t₉₀` about 60 s, 100 random nodes), not simulated.

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
