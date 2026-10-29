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
| AR-002 | Creator equivocation is neither detected nor penalized; the creator picks the winning twin. | High | Adopted: same-parent twins are held at the first-adopted twin, resolved by a subgroup verdict recorded in a twin evidence block, and the equivocator is excluded from block creation for `W` blocks (DEC-162..165). | Mitigated · DEC-162..165, DEC-198, 6.12, 6.13 | §11, §16.15; BC-FR21, BC-FR70–FR72 |
| AR-003 | A colluding pair concentrates accumulated vote and dominates block creation on low-traffic chains. | High (Medium at high traffic) | Adopted: at most one `vs` credit per vote target per block, with the `vote_interest` default raised to 50 (DEC-176); the pair falls to about its fair share. | Mitigated · DEC-176, 7.2 | §7, §12, §16.3, §16.4; BC-FR37, BC-FR55 |
| AR-004 | Support signatures may not bind to proposer and deviating block. | High | Adopted: the support signature covers `"MBSUP" ‖ proposer ‖ deviation_sequence ‖ deviation_block_hash` (O-002, DEC-187). | Mitigated · DEC-187, 6.4, 6.5 | §2, §9; DEC-187 |
| AR-005 | An approval-less deviation block can outweigh the original creator's block through its own vote credits. | Medium | Adopted: `consumed_votes` is the post-interest, pre-credit snapshot, so `d` is valued before its own credits and is never higher than `u`'s normal block (equal votes tie); verified at Tier 3 (DEC-156). | Mitigated · DEC-156, story 6.2 | §7, §9, §16.2; BC-FR28 |
| AR-006 | An approved deviation overturns about `r` normal blocks. | Medium | Evidence value independent of `m`; support only for a deviation whose parent `p − 1` lies on the supporter's active chain while it holds no evidence and no original-top block for `p`, so at most an approval-less deviation is displaced (DEC-200); published confirmation depth `⌊m/2⌋ + 2`. | Mitigated · DEC-200, 6.4 | §9, §11, §16.7, §16.11; BC-FR12 (a) |
| AR-007 | The `required_support` default (3, so `m = 5`) is not calibrated. | Medium | Adopted: default 40, below the global aggregation cap of 50 and the FR58 `H = 50`, with DEC-172's `r_eff` for small networks; the radio cost still to be confirmed in the radio simulator (DEC-173). | Mitigated · DEC-173 | §1, §9, §14; [crypto concept](./moonblokz-crypto-concept.md) §Bounded aggregation |
| AR-008 | Approval liveness depends on the reachable share of `A`, which includes intermittent transaction senders. | Medium | Adopted: `A` counts block creators and evidence or verdict signers in the window, not transaction senders, minus FR72 and late-node exclusions (DEC-185); minority-side halting stays intended; a pending first deviation no longer blocks the next proposer's round (DEC-200). | Mitigated · DEC-185, DEC-200, 6.4 | §9, §16.6; ADR-015 |
| AR-009 | Unauthenticated Stored heads evict legitimate side branches from `chain_heads`. | Medium | Evict Stored before Connected heads; bound Stored head sequence against `S_head`; verify the creator signature whenever the key is derivable. | Mitigated in ready state · DEC-157, DEC-170, DEC-188; collecting residual (AR-014) | §4, §5, §11, §16.12; BC-FR19 |
| AR-010 | UTXO saturation griefing also inflates supply. | Medium | Post-MVP dynamic custodian fee; consider funding `replay_block_reward` from custodian fees. | Accepted · OG-005 | §10.3, §12; [OG-005](./moonblokz-open-gaps-register.md) |
| AR-011 | Mempool-replenishment requests are amplified, and a CRC32 collision suppresses a target transaction. | Low | Adopted: answer a given requester at most once per replenishment interval (bounded local LRU, DEC-189); the CRC32 suppression is accepted, since the target still spreads by relay and FR13. | Mitigated · DEC-189, 10.4 | BC-FR43, BC-FR46 |
| AR-012 | Node #0 may vote for itself for the chain's lifetime. | Low | Adopted: the exemption lasts only while node #0 is the only registered node (`max_known_node_id == 0`), checked at Tier 3 (DEC-190). | Mitigated · DEC-190, 7.3 | §6, §13; BC-FR6, BC-FR37, BC-FR54 |
| AR-013 | The tail-block creator can grind the subgroup seed of its own later proposals. | Low since DEC-192 (was Medium below FR40 `final`) | Adopted: the seed is `H("MBSEED" ‖ p ‖ circulating_currency(p − 1) ‖ x)`; a creator changes the total only through included registrations, each burning the registration price (DEC-192). | Mitigated · DEC-192, 6.4, 6.7, 6.13, 7.6 | §9, §16.6; ADR-015 |
| AR-014 | A collecting or restarting node can be captured by a forged high-sequence segment. | Medium if re-scoped | Reject seedless creator signatures, or add a checkpoint; sequence-plausibility bound in collecting. | Outside threat model (Project Lead, 2026-10-20) | §4, §16.13; BC-FR2, BC-FR3, BC-FR59 |
| AR-016 | A partition longer than `W`, or one whose two sides both advance more than `H` blocks past their divergence point, becomes a permanent fork; transactions stay valid on both sides. | Medium | Post-MVP long-disconnect recovery; a larger `H` widens the reconcilable range at storage cost. | Accepted · OG-006, DEC-159 | §16.14; [OG-006](./moonblokz-open-gaps-register.md); BC-FR58 |
| AR-017 | A ready node's deep-zone chain switch ran the window-anchored FR3 pass, whose pre-seed auto-acceptance let a branch define its own starting balances, votes, keys and carry-forwards. | High | Adopted: no deep zone; every switch is an in-place walk within `H` blocks of the head (DEC-159). | Mitigated · DEC-159, 9.7 | §4, §11; BC-FR3, BC-FR58 |
| AR-018 | Before DEC-204, a chain switch inside the bound (and an FR3 adoption) trusted the FR48 binding and the below-tail UTXO references of replayed blocks at positions up to `S_head`, until Story 9.7 retained the verification horizon. | Medium (a single creator could have a skipped replay adopted at the branch tip) | Adopted: no switch trusts a block it cannot see — a switch runs only when every block its replay or a restore drops or references is retained, and the replay evaluates every FR48 binding (DEC-204); horizon retention `R = H` restores switching on matured chains (6.14). | Mitigated · DEC-204, 6.3, 6.14 | §10, §11; BC-FR23, BC-FR39, BC-FR58 |
| AR-019 | Registered nodes (valid signature, forged `consumed_votes`) and creators registered above the common ancestor can force failed chain-switch attempts. | Low | Accepted (DEC-191): the cost is linear in the attacker's own signed blocks, which are deleted, and DEC-170's pre-pass covers fork-independent creators. A signed invalid block is provable misbehaviour that could later be penalized like FR72. | Accepted · DEC-191 | §5, §11; BC-FR9, BC-FR23 |
| AR-020 | A coalition that stays the entitled creator for many consecutive blocks on its own private branch — through self-credited votes or members arranged next to each other in the creator order — builds a fully valid heavier branch and reorganizes the chain up to the FR58 bound. | Medium since DEC-176 (High before, at `vi = 5` and low traffic) | Adopted: `vote_interest` default 50 with the `vs` cap kept, and the per-target credit cap (DEC-176), which close the natural-fork route for `N ≳ 100`; `config-encoder` warns a founder whose `vote_interest / vote_scale` puts that bound above 100 nodes or below a stated network size (DEC-179). Adopted too: `tx_fee_per_byte_min` default 1 (DEC-181), which prices the only clustering route measured under the current rules, votes bought in honest blocks. Adopted too: the published confirmation rule — FR40 `final`, a confirmation at least `H` deep and not disputed, beyond every switch any node admits (DEC-182). Open: the registration-price model (O-033). Network-size-adaptive `vote_interest` is post-MVP (O-031). The FR70 twin hold already defeats the route that starts with a same-parent twin. | Mitigated · DEC-176, DEC-181, DEC-182 (residual: confirmations short of `final`) | §7, §9, §11, §16.3, §16.4; BC-FR21, BC-FR37, BC-FR38, BC-FR70 |
| AR-021 | Coalition members that hold consecutive positions in the creator order and stay silent force a grace expiry for every skipped rank and an approved deviation for every slot, stalling the chain at no cost to themselves; they climb back by voting for each other. | Medium (High at a large `required_support` without the treatments) | Adopted: the evidence block resets and excludes for `W` blocks the late nodes ranked above the proposer, as many as fit (DEC-184); grace admits the first fallback after 4 minutes and each further rank 15 minutes later (DEC-203), with a defined origin, and support stops at an evidence (DEC-183). | Mitigated · DEC-183, DEC-184, DEC-203, 6.4, 6.6, 6.7, 7.2, 8.5 | §9, §16.6; BC-FR12, BC-FR27, BC-FR37, BC-FR44, BC-FR47 |
| AR-022 | A coalition that leaves every transaction voting for an honest node out of its blocks captures block creation permanently once every honest accumulated vote is 0: a zero vote earns no interest and only a creator can credit it. | High | Adopted: liveness credit — `H` blocks after a block it created, a node gains `⌈vs/vi⌉` (DEC-178); window-anchored readiness at `W + H` keeps the credit verifiable in the FR3 pass (DEC-177). | Mitigated · DEC-177, DEC-178, 7.2, 6.14 | §7, §16.3, §16.4; BC-FR2, BC-FR3, BC-FR37, BC-FR45 |

## Detail Notes

### AR-001 — Out-of-window re-inclusion

Block-level rules for a node transfer or balance input are `s > anchor` plus no duplicate within the retained candidate chain; `anchor ≥ S_tail` holds only at mempool intake and reconciliation (§6, §16.10). Once the original block drops below `S_tail`, a creator may include the same signed transaction again: the initializer pays twice, the receiver and the creator gain. Any node reaches the creator role through rotation. The current code's Tier 1 check enforces only `block.sequence > anchor_sequence`.

Adopted 2026-10-27 (DEC-174): a block of sequence `s` requires `anchor ≥ max(0, s − W)` for every node transfer and balance input, the window tail of its parent and the bound the mempool admitted the transaction against. An earlier inclusion at `s₀` satisfies `s₀ > anchor ≥ s − W`, so it lies inside the window of `s` and the existing duplicate check finds it. The rule needs only the block's sequence and `W`, so every branch evaluates it the same way; it is Tier 1 once the configuration is locked, and the FR3 pass checks it before the lock against the candidate's own `W`. Residuals: the window-anchored FR3 pass cannot see duplicates before its segment, which it trusts like other pre-window history (DEC-129); a transaction can still be valid on both sides of a partition (AR-016). Story 7.3 adds the check beside the existing one.

### AR-002 — Creator equivocation

A creator signs two blocks at the same sequence, one paying a merchant and one spending the same funds elsewhere. With equal value, the larger serialized size wins the FR21 tie-break, so every node that receives both converges on the creator's choice. Depth is one block with an honest next creator, more under AR-003.

**Treatment (DEC-162..165, BC-FR70–FR72).** Only same-parent twins count: the twin at the divergence point is unavoidable for a profitable fork, because the creator entitled there is unique. A ready node never switches to the other twin without a verdict, so a late reveal fails outright. A subgroup chosen at the shared parent votes for its first-adopted twin, and the entitled creator records the proof and, with a quorum, the bitmap verdict in a twin evidence block; the verdict brings stranded nodes back. The equivocator is excluded from block creation and from `A` for `W` blocks, with its accumulated vote held at 0 so it returns without banked credits (DEC-175). Residuals: without a quorum the hold stays and nodes that adopted the attacker's twin first stay stranded until a restart; only the equivocator is provable, never its accomplices, who sign on one branch only; and each attempt costs about 99 votes of radio time (AR-007, AR-011), which the exclusion limits per identity. The records that stop a node from supporting both twins of a twinning proposer, or voting twice, are not durable (DEC-198): after a restart the node supports and votes nothing at or below its ready-entry head, and a twin above it is itself FR70/FR72 evidence, so the residual is low.

### AR-003 — Vote concentration by a colluding pair

The creator alone selects the transactions, and therefore the vote credits, of its block. Each transaction credits `vs` independently of amount and fee (`tx_fee_per_byte_min` was 0 before DEC-181), and fees from the creator's own transactions return to it. A 2016 B block holds about 18 node transfers. Interest adds only `⌊av/200⌋` per block and nothing to a node at `av = 0` (every creator after its block), so it does not offset 18 concentrated credits.

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

**Larger coalitions (accepted residual, DEC-186).** A coalition's blocks credit its own members (up to 18) while an honest block credits its `T` senders' targets, so at low traffic a coalition creates more than its share (vote model, cap, `vi = 50`, liveness credit; coalition block share at 1 / 3 / 18 honest transactions per block):

| `N`, coalition | 10 % | 20 % | 30 % |
|---|---|---|---|
| 30 | 68 / 20 / 9 % | 81 / 69 / 27 % | 85 / 79 / 50 % |
| 100 | 41 / 26 / 11 % | 56 / 47 / 27 % | 62 / 55 / 38 % |
| 300 | 15 / 13 / 11 % | 25 / 23 / 21 % | 35 / 33 / 31 % |

With full blocks the share stays near the population share. A per-block vote budget (`K` credits split evenly) cut the 20 % coalition at `N = 100` from 47 % to 31 % at 3 transactions per block but changed little with full blocks, so it was rejected: the target regime is full blocks, which the 15-minute `inter_block_interval_ms` now produces from about one transaction per minute. The remaining dominance of quiet, under-filled periods is accepted; it weighs on rewards (the mining reward will be non-zero, O-034) and on periodic censorship (AR-022), not on confirmations (`final`, DEC-182) or `A` (DEC-185). Below about 100 nodes a higher `vote_interest` helps (`N = 30`, 20 %, 3 per block: 69 % → 42 % at 255; DEC-179 warns).

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

Time no longer limits the attack at realistic `r`. Support collection takes minutes, during which the normal branch gains fewer than `r` blocks: with the measured RP2040 verification times an approval round is about 8–10 minutes, about 8–10 blocks at one block per minute (system constraints §2.1, DEC-180). The `deviation_replay_insertion_delay_ms` pause (300 s) is short against `r` blocks.

**Bound by DEC-159.** A switch is admitted only within `H` blocks of the switching node's own head, so an approved deviation that overturns more than `H` normal blocks would leave the nodes on the normal branch unable to switch while the deviation's nodes stay, splitting the network. At the default `r = 40` (DEC-173) an approval outweighs about 40 normal blocks, about 10 below the default `H = 50`: a normal branch that advanced `H` blocks is heavier than the approved branch in the median case, so the nodes on the approved branch, which stay within their own bound, switch to it instead of splitting. The margin narrows the overlap rather than closing it, since the overturn depth varies around its median; the full fixes stay open (the evidence value independent of `m`, or a larger `H`). A confirmation flagged FR40 `final` (at least `H` deep, DEC-182) is out of any deviation's reach on the node that reports it. Two fixes were considered and rejected on 2026-10-28 (DEC-183). An approved deviation taking precedence over the absent top's slot before branch value would make a minority partition that formed one approval refuse the longer majority chain at merge, a permanent split. A small evidence value without that precedence lets about 2–4 adjacent attackers overturn an approval by withholding, whereas today's value needs about 40.

### AR-007 — `required_support` calibration

The Project Lead stated that the default must follow:

- **Schnorr:** the evidence-block capacity, about 36 B per supporter, at most about 50.
- **BLS:** security and practicality, because its evidence capacity (about 450) is not binding.

The binding cost is radio. All `m = 2r − 1` subgroup members send a support message of 113 B (O-002, DEC-187). At a channel throughput of about 30–50 B/s, before relaying:

| `r` | 10 | 25 | 40 (default) | 50 | 100 |
|---|---|---|---|---|---|
| Message volume | 2.1 KB | 5.5 KB | 8.9 KB | 11.2 KB | 22.5 KB |
| Whole-channel time | 0.7–1.2 min | 1.8–3.1 min | 3–5 min | 3.7–6.2 min | 7.5–12.5 min |

An earlier version of this table assumed 225 B per message, twice the actual size; the range below was drawn from it. Verification CPU adds to the radio time (DEC-180).

A realistic range is therefore about 10–25 for Schnorr and 25–50 for BLS, to be confirmed in the radio simulator. The Project Lead set the default to 40 on 2026-10-27 (DEC-173), below the cap of 50 to keep the AR-006 depth under `H = 50`; with O-002's 113 B support message an approval round (79 messages) is about 9 KB. Since DEC-172 the value is a cap: a voting set smaller than `2r − 1` approves by simple majority, so a large `r` no longer halts a small network. Raising `r` lowers AR-006 likelihood but raises its depth, unless AR-006's decoupling is adopted. Since DEC-203 the first fallback is admitted after 4 minutes and each further rank 15 minutes later, about the time an approval takes to reach most nodes (9–11 minutes from the deviation's receipt, 14–17 at the 90th percentile, system constraints §2.1), so a second round starts mainly when the first is slow or fails (DEC-183 had admitted one every 4 minutes, starting 2–3 overlapping rounds); the stop rule (no support after an evidence) ends the rest, and branch value decides between approved branches. **CPU (DEC-180):** at the measured 2.9 s per signer (RP2040, current build), an `r = 40` evidence block costs about 2 minutes of verification on every node, paid again at every relay hop, and each support message 1.9 s. An approval round therefore takes about 8–10 minutes rather than the 1–1.5 minutes of radio time alone. The Project Lead kept `r = 40` on 2026-10-28 and deferred crypto acceleration to a later development phase (O-032).

### AR-008 — Approval liveness

Above a deviation only the proposer may build, and only replay or evidence blocks (§9), so a failed approval stalls ordinary progress. Each grace expiry admits the next candidate with a fresh subgroup, and its sibling deviation collects support while the earlier one is still pending (DEC-200). The tail hash and sequence are frozen during the stall, so each reachable proposer is one try. Since DEC-203 every try after the first costs 15 minutes, so the waits below, computed with the earlier shorter steps, grow to about `4 + 15(n − 1)` minutes for `n` tries.

Model: `|A| = 400`, success probability over all reachable proposers, with the expected wait to the first success:

| Reachable share of `A` | 0.3 | 0.4 | 0.45 | 0.5 |
|---|---|---|---|---|
| `r = 10` | 97%, ~51 min | 100%, ~7 min | 100%, ~3 min | 100%, ~2 min |
| `r = 25` | 9%, ~99 min | 100%, ~19 min | 100%, ~5 min | 100%, ~2 min |
| `r = 50` | ~0% | 80%, ~75 min | 100%, ~9 min | 100%, ~2 min |

A side holding less than about 40% of `A` halts until the partition heals. This protects safety, because no competing history is built. Because `A` counts every transaction sender in the window, intermittently awake devices can push the whole network below the threshold whenever the top creator is absent. Before DEC-172 no evidence could form at all with `|A| < r`; since then a small voting set approves by simple majority, so a side still halts only when less than a majority of `A'` is reachable. **Treatment (DEC-185).** `A` now counts block creators and evidence or verdict signers in the window, not transaction senders, so a subgroup is drawn from nodes that recently acted, and late-node-excluded nodes leave it. With `vi = 50`, the per-target cap and the liveness credit, creation rotates: in the vote model every voted node created within `W` blocks for `N ≤ 300`, so `A` covered the network, and at `N = 1000` `A` held about `W = 500` creators with a 10–30 % coalition at about its population share (a subgroup majority at 30 %: about 4·10⁻⁵ at `r = 40`). Counting senders gave a lower coalition share only because every sender was assumed to answer, and let an identity join `A` for one transaction fee. Residual: a node that created a block and then went silent stays in `A` for up to `W` blocks.

### AR-009 — `chain_heads` eviction

Outside a ready direct head extension, a block is stored before its signature or state is checked (§5, §16.12). The `chain_heads` table holds `MAX_BRANCH_COUNT = 40` heads, and overflow evicts the non-active head with the lowest `head_sequence` (§11). About 40 forged blocks therefore displace every legitimate side branch:

- **Ready:** sequences just below `S_head + W`.
- **Collecting:** any sequence (§5).

The forged heads also draw parent-recovery requests (BC-FR46).

**Partial treatment (DEC-157, story 6.2):** in ready state the creator signature is verified at intake, before the write, whenever the creator's key is branch-independent: node #0, or a node seeded at or below `S_tail`. Forged blocks claiming such a creator are no longer stored. Still open: forgeries naming a creator seeded inside the window or an unknown id, forgeries by registered nodes, and the collecting case.

**Eviction order (DEC-188).** In ready state a full table now evicts a Stored head before any Connected one, and among Stored heads the one farthest from `S_head`, so forged Stored heads near `S_head + W` go first and legitimate side branches near the head stay. Connected heads keep the lowest-`head_sequence` rule; collecting keeps it too, which leaves the collecting case to AR-014.

**Further treatment (DEC-170, story 6.3):** before a chain switch mutates anything, it verifies every unmarked new-branch block whose creator is fork-independent at the common ancestor, so a forgery by an unregistered sender naming any creator registered on the shared prefix no longer costs a walk. The residual is AR-019.

### AR-011 — Replenishment abuse

Every receiver of a replenishment request answers with one transaction, and no response rate limit is specified (BC-FR43, BC-FR46). A single forged request therefore costs many transmissions. A transaction ground (about 2³² hashes) to the `hash_crc32` of a target, once held by nodes, makes them skip the target in every exchange.

### AR-013 — Subgroup-seed grinding

The subgroup seed `H(hash(tail) ‖ x ‖ p)` takes the tail block of `p − 1`, which its creator built `W` blocks earlier and can vary freely (transaction order, comments). A coalition that creates that block and later has a member admitted as proposer at `p` can grind for a seed whose subgroup it controls. Probability that one random seed gives a coalition at least `r_eff = 40` of `m = 79` seats:

| `\|A'\|` | 20 % | 25 % | 30 % | 35 % |
|---|---|---|---|---|
| 298 | 4·10⁻¹⁴ | 3·10⁻⁹ | 4·10⁻⁶ | 6·10⁻⁴ |
| 498 | 1·10⁻¹¹ | 5·10⁻⁸ | 2·10⁻⁵ | 1·10⁻³ |

A 30 % coalition thus needs about 2.5·10⁵ tries of about `\|A'\|` hashes each, seconds on a PC; below `\|A'\| ≈ 100` the subgroup is almost all of `A'` and grinding gains nothing. With such a subgroup it can approve its deviation against an honest top's block and overturn up to about 40 blocks, all below FR40 `final` (`H = 50`). Accepted on 2026-10-28 (DEC-191): a confirmation below `final` was already the caller's risk (DEC-182), and a deterministic grind-free seed would need VRF or commit-reveal.

**Treatment (DEC-192).** The seed now takes the sequence and the money on the chain after `p − 1` instead of the tail hash. Fees, minting and the tail-drop burns leave the creator no choice; only the registrations a block includes move the total, so a creator has one or two seed variants per block, and each extra variant built from its own sybil registrations burns the full registration price (O-033). The seed is also known later — once `p − 1` is fixed instead of `W` blocks ahead. What remains is the luck a coalition already had: about `c · 4·10⁻⁶` per position at 30 % for a seed that happens to favour one of its members.

### AR-014 — Acquisition capture (outside threat model)

The ingredients are:

- collecting selects the highest tip sequence among qualifying segments and has no window bound (§4, §16.13);
- the pass auto-accepts pre-seed state-dependent checks, and how a seedless creator's block signature is verified is not specified (§4);
- a restart re-enters collecting (§4).

The attack needs radio-local presence, because ready nodes neither store nor relay out-of-window blocks (BC-FR60), and the forged segment must qualify before the genuine one. The victim then discards the real chain as out-of-window.

### AR-016 — Permanent forks

Under DEC-159 a node switches only to a branch that diverges within `H` blocks of its own head. A partition side that cannot reach the FR12 approval quorum stalls at its first absent creator, so it can still adopt the other side's chain within the FR60 window. The fork becomes permanent before `W` only when both sides keep advancing — both must complete approvals, which needs a near-even split — and both pass `H` blocks beyond the divergence point. FR40 `final` is node-local (DEC-182): on such a fork the two sides report different histories as final.

### AR-017 — Deep-zone trust downgrade (mitigated)

Before DEC-159, a ready node switched to a branch diverging more than `H` blocks below its head through the window-anchored FR3 pass (the FR58 deep zone). The pass seeded every node without a seed in the shared prefix from the branch's own balance entries — balance, vote count and public key — and trusted replay obligations and UTXO references below the segment (§4). A branch could therefore define its own starting state: inflated vote counts give `consumed_votes` that win FR21, and zero-input carry-forwards for dropped blocks create money. The pass's justification, attestation by a continuous signed chain of length `≥ W`, is circular, because signatures and votes are checked against keys and seeds the branch itself supplies. **Treatment (DEC-159):** no deep zone; every switch is an in-place FR23 walk within `H` blocks of the head, so every check runs on the node's own verified state.

### AR-018 — Below-tail trust inside the bound (withdrawn, DEC-204)

A replayed block at position `s ≤ S_head` drops block `s − W`, which lies below `S_tail`; until the horizon is retained, neither a switch nor an FR3 adoption can check that binding, nor the spentness and amount of a UTXO reference in `[s − W, S_tail)`. A winning branch could therefore have dropped essential state, double-spend a below-tail output, or declare an amount for a non-existent output. The UTXO half needs a colluding set: a reference below `S_tail` sits in a replayed block below the branch tip, which a follower would have refused, so only an accomplice builds on it (AR-003). The binding half does not: a single creator whose block at the tip skips the replay of `S_head − W`, and outranks the head it competes with (a late top-creator block, a grace fallback of larger size), is refused by the nodes that receive it first and adopted unchecked by those that had already released `S_head − W`; the network then converges on it. **Treatment (DEC-204):** the interim trust is withdrawn. A switch runs only when every block its replay or a restore drops or references is retained, and the replay evaluates the binding of every replayed block, so a skipped replay is exact evidence on every node. Until Story 6.14 retains the horizon, a chain at least `W` long does not switch (a liveness cost, not a safety one); the FR3 pass evaluates the binding wherever the dropped block lies in the segment, and a window-anchored candidate's first `W` positions remain pre-window trust.

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

Arranging members next to each other was feasible under the earlier rules: members parked at zero accumulated vote, then restarted together on their own zero-fee votes (50 adjacent members in 0.3–1.6 days at `N = 100` with `vi ≈ 5·vs/N`). Re-run on 2026-10-28 under the current rules (per-target cap and `vi = 50`, DEC-176; liveness credit, DEC-178) with the same heuristic, the free route — the coalition's own blocks only — reached its target in no setting within 40 000 blocks, except once without the liveness credit. The only route that still worked buys votes in the free slots of honest blocks at low traffic (3 honest transactions per block): 10 adjacent members in 0.1–0.6 days with about 0.4–2.6 thousand bought transactions (`N` = 100 / 300), and 50 in 1.1–1.4 days with 12–15 thousand at `N = 300` and in 3.2 days with 39 thousand at `N = 1000`. With full blocks there is no free slot, and no setting succeeded. Those votes pay real fees to honest creators, so `tx_fee_per_byte_min = 1` prices this route at about 101 units per bought vote: roughly 37–265 thousand units for 10 members and 1.2–4 million for 50, plus the sybil registrations (O-033) (DEC-181). A heuristic that found no route does not prove that none exists. Since DEC-186 (15-minute `inter_block_interval_ms`) honest blocks fill from about one transaction per minute, so the free slots this route buys exist only in quieter traffic. Paths to a reorg and what stops them today:

- **Twin at the divergence point** (the creator signs the public and the private block): held by FR70, resolved by FR71.
- **Withhold the public slots:** honest approved deviations fill them and their evidence outweighs the private branch under the current evidence value (AR-006); lowering that value without the rule that an approved deviation forfeits the absent creator's slot would reopen a late-release double spend.
- **No twin, no withholding:** the coalition must out-create the honest chain inside the FR58 bound, which the table above bounds.

Removing the `vs` cap from the interest step adds nothing against this and lets an accumulated vote overflow `u32` (a chain halt) once about 250–300 nodes climb at `vi = 50`.

### AR-022 — Censorship capture through zero votes

A zero accumulated vote earns no interest (BC-FR37), and a node leaves zero only through a credit in a block, whose content its creator chooses; the FR9 pacing rule bounds when a block arrives, never what it contains. A coalition whose blocks credit only its own members, and which leaves out every transaction that votes for an honest node, therefore keeps the creator role once every honest vote is 0: its members always outrank the honest nodes, no grace period expires, and the honest nodes cannot even register or stay in `A`. Every honest vote is 0 at start-up, and at low traffic an honest block adds about as many credits as its own creator reset removes, so the honest pool drains. A censored honest node cannot be told from an absent one on chain; the only liveness evidence a coalition cannot remove is a block the node created.

Model (vote model with the DEC-176 cap and `vi = 50`; `T` honest votes per honest block, which under the FR9 pacing rule at the default 60 s is about the honest transactions per minute; 3–4 runs per setting, at start-up and after 5 000 honest blocks):

| | No liveness credit | Liveness credit at `H = 50` (DEC-178) |
|---|---|---|
| Permanent capture, `T = 1` (pair, 10 % and 20 % coalitions, `N` = 30 / 100 / 300) | every run, within 1–500 blocks | none |
| Permanent capture, `T = 2` | at start-up with a 20 % coalition only (3 of 8 runs) | none |
| Permanent capture, `T ≥ 3` | none | none |
| Honest block share under attack, `T = 1`, 20 % coalition, `N` = 100 / 300 (honest node share 80 %) | 0 % / 0 % | 43 % / 75 % |
| Vote steering (popular 20 % of nodes, `T = 3`), `N` = 30 / 100 / 300 | 61 % / 55 % / 52 % | 60 % / 44 % / 27 % |
| Deviation share with 30 % of the nodes gone silent | 0.1–0.3 % | 0.6–1.0 % |

Delays of `W − H` and `W` kept more vote steering (44 % at `N = 300`) but rescued slower (18 % / 51 % honest share) and, referring to a block the window-anchored FR3 pass cannot see, needed either `2W` blocks at readiness or a new per-node `last_created` field in every balance entry. `H` lets the pass seed accumulated votes past the credit's reach with `W + H` blocks (DEC-177), which a node entering ready needs anyway for the FR58 reserve (AR-018). A lift only at zero (set the vote to `⌈vs/vi⌉` when it is 0) measured the same (within 0.4 points) and was set aside: the unconditional credit needs no condition and its inverse is a subtraction. A pseudo-random credit was rejected: under capture the coalition creates every seed and can grind it, and it would lift silent nodes as well. Residuals: censorship becomes periodic rather than permanent (a 20 % coalition still holds most blocks at `T = 1`), and vote steering weakens in large networks at low traffic. With the 15-minute `inter_block_interval_ms` (DEC-186) blocks fill from about one transaction per minute, so the `T = 1` regime means quieter traffic than before; the residual is accepted (AR-003).

### AR-021 — Silent coalition stall

A grace expiry admits one more rank (§9), and a deviation resets only the skipped top (`first_voted_node`). A cluster of `n` silent members ahead of the first honest node therefore costs about `n` expiries before the first deviation, then one approval per member. The measured block propagation median `t₅₀` of 33 s (100 nodes) used unsigned test blocks. Relaying follows verification, so a real low-traffic block reaches the median node in about 1 minute, and an `r = 40` approval round takes about 8–10 minutes (system constraints §2.1, DEC-180). A safe grace period must let a pending evidence reach a majority before the next rank is admitted, so it now covers about 10 minutes. A 50-member cluster then stalls the chain for about 8 hours per cycle, and still about 75 minutes with a short grace (about 1.5·`t₅₀`, 1.5 minutes) when the admitted proposer issued nothing. The earlier figures, 1–1.5-minute rounds and a stall of about 2 hours, assumed radio time only. **Treatment (DEC-183, DEC-184).** Without a wider reset each member would cost its own expiry sequence and approval, about `n(n−1)/2 · g + n` rounds per cycle, because only `u` is reset and the grace expansion restarts at every position. Resetting several nodes needs their prior votes for FR23, and the header holds one, so the evidence block now carries the late nodes with their votes, as many as fit, and resets and excludes them for `W` blocks: a 50-member cluster costs about 50 expiries (about 3.3 hours at 4 minutes; about 12.3 hours since DEC-203 made every step after the first 15 minutes long) and one approval, and cannot repeat within `W` blocks. An honest node lands on the list only if it produced nothing within its grace step (4 minutes for the first fallback, 15 minutes after it); an approval also needs a majority of `A` reachable (AR-008). The members lose only their accumulated vote and climb back on each other's votes, so the stall repeats every cycle. Scattered silent members cost one expiry and one approval each.

## Analysis Models

The figures above come from simplified models, not from the radio simulator:

- **Vote model:** BC-FR37 steps 1–3 with `vs = 1000` and `vi = 5`. The creator is `argmax (av, −id)`. 18 or 3 honest transactions per block, with uniformly random vote targets (fixed three-neighbour targets give the same results). 20,000–30,000 blocks. No grace periods, radio delay or fee priority.
- **Reorg depth:** the largest `k` with `Σ v(next k normal blocks) < (⌊m/2⌋ + 2)·av(x)`.
- **Subgroup majority:** binomial over `m` seats.
- **Retries:** hypergeometric draws from `A`, one try per reachable proposer per grace step.
- **Private branch (AR-020):** the vote model with `N` up to 1000, a coalition of `c` members, a fork at every point where a member is top; on the private branch only coalition transactions (18 per block, all crediting the next member), honest nodes gain interest only; an honest top needs an approved deviation, counted with the probability that the coalition holds `r` of the `2r − 1` subgroup seats (hypergeometric, grinding over members' proposals where OG-015 allows it). 300–600 fork points per setting.
- **Clustering (AR-020):** park → synchronise → maintain, spending the coalition's own block credits and, optionally, bought slots in honest blocks; success when the top 10 or 50 creator positions are all coalition; 40 000 blocks per setting; re-run on 2026-10-28 with `vi = 50`, the per-target cap and the liveness credit.
- **Interest sweep (DEC-176):** the vote model with `vi` ∈ {5, 10, 20, 50, 100, 255}, with and without the per-target cap; 36 000–60 000 blocks per setting; steering with 20 % of nodes drawing 80 % of the vote targets; offline nodes (10 % and 30 %) for the deviation rate, which does not depend on `vi`; a greedy attacker spending every vote of every block to starve one node, for the accumulated-vote ceiling (about `N·vs`).
- **Capture (AR-022):** the vote model with a coalition that credits only its own members and omits every honest vote from its blocks, honest blocks carrying `T` uniform votes; capture means no honest block in the last 5 000 of 30 000 blocks; the liveness credit (or a lift only at zero) is applied after the creator reset to the creator of the block `K` back; silent nodes receive no votes.
- **Stall (AR-021):** expiries and approval rounds counted from the block propagation measurement (`t₅₀` 33 s, `t₉₀` about 60 s, 100 random nodes, unsigned test blocks), plus the measured RP2040 verification time at about 3 relay hops (system constraints §2.1); not simulated.

## Assessed §16 Facts

The §16 facts this section listed as not yet assessed were assessed on 2026-10-28:

- **16.5, several proposers supported at one sequence.** Settled by DEC-183. Grace keeps admitting ranks while approvals are pending, so rounds can overlap; a node stops supporting once it holds an evidence block or the top's own block, and branch value decides between approved branches. Since DEC-203 later ranks follow 15 minutes apart, so overlap is mostly limited to slow or failed rounds, and DEC-200 lets a sibling deviation collect support while the first is pending.
- **16.8, Schnorr aggregate weights.** No attack was found. Key cancellation (rogue keys) is blocked by the registration proof of possession. The weights, however, are derived from the sum of the nonce `r` values and the message only ([Crypto Algorithm](./moonblokz-crypto-algorythm.md) §Deterministically weighted aggregation), whereas published half-aggregation schemes with a security proof bind each weight to every signer's nonce, key and message. Decided in DEC-194: the weight seed binds the full ordered list of `(R_i, P_i)` and the message, which costs little next to the 2.9 s per signer of verification; an external cryptographic review of the scheme and of the x-only comparisons is recommended, not required. The x-only check follows the BIP340 convention, without its even-`y` check on the reconstructed point.
- **16.17, pairwise re-evaluation.** Safe. Every head's branch value shares `S_tail`, so a tail slide shifts all heads equally, and the order of two heads changes only when one of them extends; FR22 re-evaluates exactly then. Admissibility only narrows as the tail advances, and a twin verdict or a Stored → Connected change arrives as an extension or admission event. Separate block arrivals are separate events, so a failed switch never blocks another branch's own trigger. The one gap was an admission that connects several heads at once (a shared missing ancestor arriving): if the best one's switch failed, the next waited for its own extension. Since DEC-193 the next-ranked affected head is tried within the same admission.
- **§5, pacing on local arrival times.** Accepted. A local clock can only defer a block (soft-fail) and never invalidates it, so differing arrival times delay acceptance on some nodes without splitting the chain.

## Related Documents

- [`moonblokz-blockchain-algorithm-summary.md`](./moonblokz-blockchain-algorithm-summary.md) — the rule statement every row cites as `§n`, including the §16 facts evaluated here.
- [`moonblokz-blockchain-prd.md`](./moonblokz-blockchain-prd.md) — authoritative wording of every cited FR; a treatment changes the PRD only through a decision.
- [`moonblokz-decision-register.md`](./moonblokz-decision-register.md) — the `O-` and `DEC-` rows that carry treatment decisions and owner stories.
- [`moonblokz-open-gaps-register.md`](./moonblokz-open-gaps-register.md) — specification gaps and accepted limitations that several risks depend on.
- [`blockchain-adrs/ADR-015-approval-subgroup-selection.md`](./blockchain-adrs/ADR-015-approval-subgroup-selection.md) — subgroup rule and rejected alternatives behind AR-006 to AR-008 and AR-013.
- [`moonblokz-blockchain-concept.md`](./moonblokz-blockchain-concept.md) — §Security Framing, the source of the threat-model boundary.
- [`moonblokz-simulator-concept.md`](./moonblokz-simulator-concept.md) — the tool for replacing model estimates with radio-level measurements.
