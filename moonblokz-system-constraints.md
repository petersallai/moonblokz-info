# MoonBlokz System Constraints & Limits Reference

## Role of This Document

This file is a **consolidation/navigation layer** for the numeric constraints, capacity caps, memory budgets, and physical limits that are spread across the MoonBlokz knowledge base. It exists so that cross-cutting questions — hardware sizing, RAM/flash budgeting, capacity planning, deployment tuning — can be answered from one place instead of stitching values together from concept, architecture, algorithm, and implementation documents.

**This document is NOT authoritative for the values it lists.** Each value's authoritative home is the linked source document. If a value here diverges from its cited source, the **source wins**, and the divergence must be flagged and resolved per [`AGENTS.md`](./AGENTS.md). Every row therefore carries an explicit source link; this file restates the value and a one-line implication only, never the full derivation.

When a new numeric cap, budget, or limit is introduced anywhere in the knowledge base, it must also be registered here (per the Cross-Cutting Fact Placement rule in [`AGENTS.md`](./AGENTS.md)).

## How to Use

- **Hardware sizing question** → start with §3 (RAM budget) + §1 (capacity caps) + §6 (radio profiles), then read the cited architecture sections for rationale.
- **Capacity planning** → §1 (const-generics) + §2 (crypto bounds).
- **CPU time and latency** → §2.1 (measured crypto timings and what they cost per block).
- **Flash / persistence sizing** → §4.
- **Radio / airtime** → §5 + §6.
- **Before trusting a margin figure** → read §7 (Known Discrepancies & Gaps) first.

---

## 0. Embedded Implementation Discipline (Project-Wide)

MoonBlokz targets constrained embedded environments first, and it does so as a complex system that deliberately pushes the hardware limits of the target MCU. Minimalism is therefore a primary design goal, not a cleanup item: both the size of in-RAM data structures and the size of the compiled binary are load-bearing constraints, alongside API surface and transient stack use. Only code that is genuinely required by the work at hand may be added. This restraint is not only a resource concern — less code means less noise, which keeps an already complex system easier to understand and review.

Project-wide rules:

- **Keep code and memory at the absolute minimum needed for the current story and explicitly planned follow-up stories.** Do not add fields, counters, helper functions, accessors, buffers, or public APIs just because they might be useful someday. If the current task does not require a piece of code, it does not belong in the change.
- **Minimize compiled code size, not only RAM.** Binary/flash size is as constrained as RAM. Prefer a single code path over parallel variants; avoid redundant helpers, gratuitous generics, and dead branches. Every function and match arm that survives compilation costs flash and adds reading noise.
- **Do not introduce unnecessary divergence for host/simulator execution.** Share one implementation between embedded and simulator/test builds; add a platform-specific path only when the target genuinely requires it, and keep that divergence as small as possible. A simulator-only convenience variant that the embedded target does not need is exactly the redundancy to avoid.
- **Every new field/function must have a visible consumer.** The consumer can be current code or a concrete upcoming story/FR/ADR. If no consumer is identifiable, leave it out and reintroduce it later with a precise name and rationale.
- **Avoid convenience getters for internal structs.** If a struct is private module storage metadata, prefer direct field access inside the owning module and tests. Add accessors only when they are part of an intentional public API boundary.
- **Avoid convenience derives.** Do not add `Debug`, broad `Clone`, or similar derives to production/public types unless a real consumer requires them. If a derive is needed for fixed-array initialization or another concrete embedded reason, keep it narrow and document why.
- **Prefer private implementation details over public surface.** Public APIs are long-lived contracts; keep them minimal and story-driven.
- **Document memory impact for new persistent fields/buffers.** Any new bounded array, per-entry field, cache, or scratch buffer should identify what it costs and which requirement/story needs it.
- **Future-possible is not enough.** A vague future use does not justify embedded RAM/code cost. Reintroduce the element when the future story makes the use concrete.

This rule is intentionally stricter than desktop/server Rust style. It applies across blockchain, mempool, vote, storage, radio, telemetry, and future runtime work unless a subsystem-specific PRD/architecture explicitly overrides it.

### 0.1 Design-Decision Discipline

An implementation design choice — a story decision, a review option, a refactor — is checked against three rules before it is recommended, and the proposal names what it checked:

- **Ratified principles win over local simplicity.** Find the decision-register principles the option touches and follow them. Example: validate where the verdict is final, then write; a block that fails is never stored (DEC-019). "The outcome is the same" or "the code is simpler" does not override a ratified principle.
- **Keep a specified incremental mechanism.** When the PRD or the architecture specifies incremental maintenance (for example FR21 (i), `branch_value += value` in `chain_heads`), do not replace it with a recomputation to save code. The minimalism of §0 also counts runtime cost: a storage read per event is not free. An option argued as simpler needs a bounded or measured cost on the other side, and "rare" is no argument without a bounded worst case.
- **Event-time information before new state.** If an option seems to need a persisted flag or field to tell cases apart, first check whether the mutating function already knows the case when the event happens and can hand it to its caller in the same call. Prefer that to new stored state, and do not reduce the choice to "new state or drop the mechanism".

Recorded as DEC-139 (Story 6.1 create-story).

## 1. Blockchain Capacity Caps (Const-Generics)

Authoritative source: [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §5 (Const generic catalog). Tuning levers: §12.

| Const generic | Default | Meaning / deployment implication |
|---|---:|---|
| `MAX_NODES` | 1000 (Schnorr; RP2040 BLS profiles 258 `bls-min-sig`, 211 `bls-min-pk`, §3.4) | **The build's node capacity.** Sizes all node-id-indexed arrays (`NodeInfo`, `VoteEngine.accumulated_vote`); every node holds an entry for every registered node. The network cap is chain parameter ID 32 `max_node_count`, which acceptance bounds by `MAX_NODES` (DEC-228). Dominant RAM driver — see §3. |
| `SNAKE_CHAIN_LENGTH_MAX` | 500 | **Capacity** of the active-chain window, not its length. The length in force, `W`, is chain configuration ([BC-FR56](./moonblokz-blockchain-prd.md)) and acceptance requires `W ≤ SNAKE_CHAIN_LENGTH_MAX`; a build whose capacity is below the chain's `W` cannot join. See D2 in §7. |
| `MAX_BLOCK_SIZE` | 2016 | **Build tier feature**, not a const generic: the block capacity a `moonblokz-chain-types` size-tier feature selects (`block-size-2016` default, `block-size-65536`; DEC-220). Acceptance requires the chain's `block_size_limit` to fit it; a larger block is a local refusal, never evidence (DEC-225). See §4. |
| `MAX_BLOCKS` | 600 | In-RAM block-table slots; 1:1 with RP2040 durable flash capacity (see §4). Chain-config acceptance also requires `W + 2·H ≤ MAX_BLOCKS` ([BC-FR8](./moonblokz-blockchain-prd.md), DEC-160). With `MAX_NODES` it is bounded by the branch-value proof `MAX_BLOCKS · VALUE_CAP ≤ u64::MAX`: a build profile keeps `MAX_BLOCKS · MAX_NODES` below about 8.6 · 10⁷ (architecture §6.4, DEC-264). |
| `MAX_BRANCH_COUNT` | 40 | Chain-heads table capacity; collecting-state branch headroom. |
| `MEMPOOL_COMPACT_BYTES` | 20160 | Mempool compact buffer (~10 × `MAX_BLOCK_SIZE`; at least `MAX_BLOCK_SIZE`, [BC-FR56](./moonblokz-blockchain-prd.md), DEC-223). |
| `MEMPOOL_MAX_ENTRIES` | 128 | Mempool index capacity. |
| `MAX_TX_PER_BLOCK` | ~64 | Derived; [BC-FR45](./moonblokz-blockchain-prd.md) included-keys gathering. |
| `MAX_BLOCK_UTXO_OUTPUT` | 256 | Per-block available-UTXO bitfield width (ADR-016); also the Tier 1 UTXO-output cap before the durable lock ([BC-FR9](./moonblokz-blockchain-prd.md)). |
| `MAX_AGGREGATED_SIGNATURES` | 50 | Approval subgroup cap (ADR-015). Matches Schnorr per-block supporter capacity (§2). |
| `PUBLIC_KEY_SIZE` | 32 / 96 / 48 | Schnorr / `bls-min-sig` / `bls-min-pk`; crypto-feature-dependent, see §2. |

**Using one of these to bound a chain-config parameter has a rule.** A capacity above is a compile-time constant of *one build*, so it may bound a chain-config parameter **only at acceptance**, where the answer is refusing the chain — never as a resolution-time fallback, which would leave a node participating with a different value than a differently-built node resolves. That is why the parameters bounded this way are literal-only, and why a parameter that needs a per-build ceiling but must stay computable has the *chain* declare the ceiling instead. The rule and the full checklist for allocating a parameter live in [Configuration Module Specification](./moonblokz-configuration-specification.md) §4.5, which wins on any divergence.

---

## 2. Crypto Constants & Bounded Aggregation

Authoritative source: [`moonblokz-crypto-algorythm.md`](./moonblokz-crypto-algorythm.md) §Family-Specific Constants + §Global bounded-aggregation constants.

| Constant | Schnorr | `bls-min-sig` | `bls-min-pk` | Note |
|---|---:|---:|---:|---|
| `SIGNATURE_SIZE` | 64 | 48 | 96 | Standalone signature. |
| `MULTI_SIGNATURE_SIZE` | 64 | 48 | 96 | |
| `PUBLIC_KEY_SIZE` | 32 | 96 | 48 | Min-sig keys are 3× larger — main RAM divergence (§3). |
| `PRIVATE_KEY_SIZE` | 32 | 32 | 32 | |
| `AGGREGATED_SIGNATURE_CONSTANT_SIZE` | 34 | 50 | 98 | Fixed part of an aggregated signature. |
| `AGGREGATED_SIGNATURE_VARIABLE_SIZE` | 32 | 0 | 0 | **Per-supporter incremental cost.** BLS aggregates into the constant slot (0 growth). |

Every wire layout follows these sizes (DEC-251; built in Story 6.17); the derived block sizes are in §4. A chain names its scheme (configuration ID 33 `signature_scheme`), and a build of another scheme refuses it at acceptance (DEC-252).

Global, all families:
- `MAX_AGGREGATED_SIGNATURES = 50` (count cap).
- Practical supporter capacity of an approval evidence block (DEC-195 / DEC-241 / DEC-242: a `⌈m/8⌉ B` signer bitmap, no per-signer id) in a 2 016 B block: with Schnorr the aggregate grows by 32 B per signer, so the fixed part is 1 339 B at `required_support = 40` (69 late entries fit) and 1 662 B at 50 (29 fit), and about 57 signers would fit with no late entry; with BLS the aggregate is constant (50 / 98 B), and the `m u8` field bounds the subgroup instead (`m = 2r − 1 ≤ 255`, so `r ≤ 128`). The global cap of 50 is just below the Schnorr ceiling; a BLS-centered deployment could carry more if the cap were raised. Chain acceptance guarantees that the fixed part fits (configuration §6 check 20, DEC-257).
- An evidence block always fits its run: every block at `s ≡ H − 1 (mod H)` must be clean (no UTXO output, no registration; [BC-FR48](./moonblokz-blockchain-prd.md), DEC-265), which costs about 2 % of block positions at the defaults (`W = 500`, `H = 50`) and keeps a proposer's replays within `H` positions of its deviation, so the evidence lands by `p + H` ([BC-FR27](./moonblokz-blockchain-prd.md), DEC-258).

`MAX_AGGREGATED_SIGNATURES` is also the per-build bound `required_support` is checked against at chain acceptance: the parameter is literal-only and refused above it, never clamped — the chain-declared ceiling `max_aggregated_signatures` was retired on 2026-10-12 (DEC-075). See [Configuration Module Specification](./moonblokz-configuration-specification.md) §6 for why a per-build bound must be literal-only.

Cross-check: the `ApprovalAccumulator` keeps the ranked subgroup and every held support's serialized multi-signature with its rank until the evidence block is built, because the DEC-194 weights need the complete signer set: ~3 686 B with Schnorr, ~2 886 / 5 286 B with `bls-min-sig` / `bls-min-pk` (DEC-236); see [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §6.5.

### 2.1 Measured crypto timings (RP2040) and what they cost

Authoritative source: [`moonblokz-crypto-implementation.md`](./moonblokz-crypto-implementation.md) §Measured costs of the built backends (DEC-249). Measured on the reference board (Waveshare RP2040-LoRa, 125 MHz) with the built `schnorr` backend (`k256`, DEC-235): **163 ms to sign, 209 ms to verify one signature, 13.3 s for a 40-signer aggregate check** (332 ms per signer; on Schnorr aggregation saves space, not time). These replace DEC-180's `schnorr-malachite` figures (0.9 s / 1.9 s / 2.9 s per signer, measured with an allocator), from which the earlier version of this table was derived. The BLS schemes verify one signature in 11.7 s (`bls-min-sig`) and 12.7 s (`bls-min-pk`) on the same board; Story 6.17's RP2040 BLS profiles carry those numbers.

A node relays a block or support message only after accepting it, so each relay hop adds the verification time. The exception is a support whose deviation is not on the receiver's active chain and whose supporter key the receiver cannot derive: it is accepted and relayed unverified ([BC-FR26](./moonblokz-blockchain-prd.md), DEC-232, AR-023). The costs below are derived, not measured: they assume about 3 hops to the median node, on top of the measured unsigned-block `t₅₀` of 33 s at 100 nodes (re-derived 2026-11-17 with the measured Schnorr numbers; the model is unchanged).

| Work | Verification per node | Remark |
|---|---:|---|
| Full transaction block (18 transfers + block signature) | about 4.0 s | median propagation about 45 s |
| Low-traffic block (3 transfers + block signature) | about 0.84 s | median propagation about 36 s |
| Approval evidence block, `r = 40` | about 13.5 s | median propagation about 1.2 min |
| Support message | 0.21 s | the proposer needs 40 (about 8.4 s); a node that relays all 79 spends about 17 s |
| One approval round, `r = 40` | — | about 2.5–3.5 min from a node's receipt of the deviation to its receipt of the evidence (about 4–5.5 min at the 90th percentile, scaled as before): block to the subgroup 0.6–0.75 min, 40 supports to the proposer about 0.7–1.4 min, evidence about 1.2 min. The radio, not verification, now dominates. DEC-203's 15-minute later grace step was set on the earlier 9–11-minute figure; whether to shorten it is open (O-040) |
| FR3 pass over `W + H = 550` blocks | about 8 min (low traffic) to 36 min (full blocks) | plus about 14 s per evidence block |
| Chain switch over `H = 50` full blocks | about 3.3 min | replay only |

An incoming evidence block's aggregate is verified inline, once per evidence block per node, after the cheap checks, and blocks the call: about 13.3 s with Schnorr, 51.6 s with `bls-min-sig` and 42.6 s with `bls-min-pk` at 40 signers (DEC-263). The "signatures accepted" mark keeps a chain-switch restore or a later walk from repeating it, or any transaction signature (DEC-259); a junk aggregate behind correct cheap fields still costs each judging node one verification (AR-026). Its frame makes the FR3 pass the `bls-min-sig` stack peak, so that Core 1 is 57 KiB (§3.3).

At about 4 s per full block, a node verifies about 15 full blocks per minute (about 270 transfers per minute, before support traffic), so the radio's airtime limits (§5), not verification, are expected to bind first. Under BLS on the RP2040 a full block takes about 3–4.5 minutes to verify (13 transfers at 12.7 s under `bls-min-pk`, 22 at 11.7 s under `bls-min-sig`), which is a BLS-profile limit (Story 6.17).

---

## 3. RAM / SRAM Budget (RP2040, 264 KB)

Authoritative source: [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §7 (RAM budget verification) + §8 (stack frames). Total SRAM target: **264 KB (RP2040)**.

### 3.1 Chain-lib data-structure costs (at default const-generics)

| Structure | Schnorr | `bls-min-sig` | `bls-min-pk` | Driver |
|---|---:|---:|---:|---|
| `NodeInfo` (SoA) | ~44 KB | ~108 KB | ~60 KB | `MAX_NODES` × (`PUBLIC_KEY_SIZE` + 8 + 4) |
| `BlockTable` | 48 KB | 48 KB | 48 KB | `MAX_BLOCKS` × 80 B (`u32` block length, DEC-224) |
| `ChainHeadsTable` | 2.88 KB | 2.88 KB | 2.88 KB | `MAX_BRANCH_COUNT` × 72 B (architecture §7.1; was listed here as 32 B) |
| `ApprovalAccumulator` | ~3.7 KB | ~2.9 KB | ~5.3 KB | the ranked subgroup and every held support's serialized multi-signature until the evidence block is built (DEC-236) |
| Active-set counter, exclusion markers, circulating currency, requester LRU, non-conflict ring, support-relay FIFO | ~4.3 KB | ~4.3 KB | ~4.3 KB | DEC-185 / DEC-213 (`MAX_NODES` × 2 B + 4 B tail creator), FR72 / DEC-184 (1 bit per node), DEC-192 (u64), DEC-189 (~384 B, sized by 10.4), DEC-198 / DEC-219 (16 × 13 B + 4 B support floor), DEC-199 / DEC-231 (128 × 12 B + 1 B cursor) |
| `EmitScratch` + scheduler + lifecycle + PRNG | ~2.1 KB | ~2.1 KB | ~2.1 KB | `MAX_BLOCK_SIZE` buffer + small fields |
| **`moonblokz-blockchain` subtotal** | **~103 KB** | **~167 KB** | **~122 KB** | matches architecture §7.1 / §7.2 (refreshed 2026-11-17); min-pk derived from the rows (estimate) |
| `Mempool` (sibling `moonblokz-mempool`) | ~24.3 KB | ~24.3 KB | ~24.3 KB | `MEMPOOL_COMPACT_BYTES` + compact index (128 × 32 B, `u32` offsets, DEC-223) including stored `transaction_fee` + small fields; authoritative layout in blockchain architecture §6.8 |
| `VoteEngine.accumulated_vote` (sibling `moonblokz-vote`) | ~4 KB | ~4 KB | ~4 KB | `MAX_NODES` × 4 B |
| **blockchain + mempool + vote** | **~131 KB** | **~194 KB** | **~148 KB** | estimates at 1 000 nodes; §3.2 counts the measured profiles |

Measured (Story 6.17, footprint probe): the blockchain's static RAM, its `VoteEngine` included, is about 54 800 B (about 58 500 / 57 800 / 60 000 B with Story 6.6's accumulator) plus 50 B (Schnorr), 114 B (`bls-min-sig`) or 66 B (`bls-min-pk`) per node of `MAX_NODES` (`SNAKE_CHAIN_LENGTH_MAX` 500, `MAX_BLOCKS` 600, `MAX_BRANCH_COUNT` 40, `MAX_BLOCK_UTXO_OUTPUT` 256); the per-node cost is the node-id-indexed arrays (`NodeInfo`, the active-set counter and `accumulated_vote`). The default 1000-node cap already consumes most of the RP2040 budget, so caps an order of magnitude larger (for example a city-scale payment network) exceed RP2040 SRAM and require an MCU with substantially more RAM.

### 3.2 Full-system allocation

Measured in Story 6.17 (DEC-254) and re-measured after Story 6.6's accumulator and Story 6.7's evidence check, at each scheme's RP2040 profile (§3.4), in bytes; 264 KB = 270 336 B.

| Consumer | Schnorr (1 000 nodes) | `bls-min-sig` (258) | `bls-min-pk` (211) | Basis |
|---|---:|---:|---:|---|
| blockchain incl. vote (probe `SLOT`) | 108 528 | 87 200 | 73 968 | measured (footprint probe) |
| node statics without Core 1 (USB console, executor pool, channels) | 11 720 | 11 720 | 11 720 | measured (node build) |
| Core 1 stack (blockchain task) | 32 768 | 58 368 | 71 680 | measured peak × 1.25, §3.3 |
| mempool | 24 883 | 24 883 | 24 883 | ~24.3 KB, not linked yet |
| radio-lib (memory-config-medium) | 61 440 | 61 440 | 61 440 | ~60 KB, source-confirmed |
| crypto-lib scratch | 4 096 | 4 096 | 4 096 | estimate |
| storage page buffer | 4 096 | 4 096 | 4 096 | estimate |
| other task stacks | 18 432 | 18 432 | 18 432 | estimate |
| **Total** | **265 963** | **270 235** | **270 315** | |
| **Margin (270 336 − total)** | **4 373 (~4.3 KB)** ✅ | **101** ✅ | **21** ✅ | |

The radio figure is the **source-confirmed** `memory-config-medium = ~60 KB` (radio module `README.md` + `lib.rs` profile consts; see §6 and §7/D1), additive to the separately-counted Embassy task stacks. The node statics replace the earlier ~4 KB "Embassy executor + statics" estimate. At 1 000 nodes no BLS variant fits, so each fixes a smaller `MAX_NODES` (§3.4), the largest count that fits; with margins of tens of bytes, any growth in a counted consumer lowers it. Schnorr keeps ~4.3 KB (~1.6%) with its 32 KiB Core 1 (DEC-137; ~7.9 KB before Story 6.6's accumulator). Authoritative: architecture §7.3/§7.4/§12.

### 3.3 Stack frames

- Chain-lib hosting task: runs on Core 1, whose stack is sized per scheme at ≥ 1.25 × the deepest root's measured peak, rounded up to a KiB (architecture §8, DEC-137 amended 2026-11-17, DEC-254). The deepest root is `process_genesis` with Schnorr (25 692 B) and `bls-min-pk` (56 552 B), because it signs, and with `bls-min-sig` the FR3 pass (`initialize_from_storage` 46 584 B), which verifies an approval evidence block's aggregate inline (Story 6.7, DEC-263), so Core 1 is **32 KiB / 57 KiB / 70 KiB**, selected by `#[cfg]` in the node's `CORE1_STACK`.
- `bls-min-pk`'s hash-to-G2 signing is what makes its Core 1 the largest; `bls-min-sig` signs in G1.
- Other Embassy tasks (radio, USB console): **4 KB**.

### 3.4 Deployment tuning (BLS profiles)

Per-lever savings: [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §12.1. Each BLS variant's RP2040 profile (Story 6.17, D12, DEC-254; re-fitted after Story 6.6's accumulator, and `bls-min-sig` again after Story 6.7's evidence check) fixes `MAX_NODES` at the largest count whose static RAM, its own Core 1 and the rest of §3.2 fit 264 KB; `SNAKE_CHAIN_LENGTH_MAX` 500 and `MAX_BLOCKS` 600 stay. Measured with the footprint probe (`footprint.py --scheme S`); architecture §12.2 carries the same figures.

| Profile | `MAX_NODES` | Core 1 | Static RAM (`SLOT`) | Node flash (estimated) | Margin |
|---|---:|---:|---:|---:|---:|
| Schnorr (default) | 1 000 | 32 KiB | 108 528 B | 105 408 B | 4 373 B ✅ |
| `bls-min-sig` | 258 | 57 KiB | 87 200 B | 141 016 B | 101 B ✅ |
| `bls-min-pk` | 211 | 70 KiB | 73 968 B | 146 400 B | 21 B ✅ |

A BLS build joins only chains with `max_node_count` at or below its `MAX_NODES` (DEC-228). Node flash stays far inside the ~800 KB left for code beside the 600 block slots, so `MAX_BLOCKS = 600` keeps its headroom. The profiles live in the footprint probe's `Blockchain` instantiation until the node instantiates the blockchain itself (the bridge, AR12 / Epic 8).

**Min-pk fits fewer nodes than min-sig on the RP2040.** The earlier estimates (about 450–500 nodes min-sig, 700–800 min-pk, and min-pk at about 263 KB with 1 000 nodes) assumed a 24 KB Core 1 for both. With the measured stacks, min-pk's hash-to-G2 signing forces a 70 KiB Core 1 against min-sig's 57 KiB (49 KiB before Story 6.7's inline evidence check), which outweighs its halved key array (66 against 114 B per node): 211 nodes against 258. The earlier expectation that min-pk is the better RP2040 BLS variant is reversed. The LRU public-key cache of architecture §12.3 remains a key-array lever (not MVP).

---

## 4. Flash / Storage Geometry

Authoritative sources: [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) (Appendix A + §E) and [`moonblokz-storage-architecture.md`](./moonblokz-storage-architecture.md) (RP2040 placement contract). Canonical block constants originate in `moonblokz-chain-types`.

| Constant | Value | Note |
|---|---:|---|
| `MAX_BLOCK_SIZE` | 2016 B | The reference tier's block capacity (chain-types `block-size-2016`); the values below that derive from it are the reference tier's. |
| `HEADER_SIZE` | 122 / 106 / 154 B | `58 + SIGNATURE_SIZE`, Schnorr / `bls-min-sig` / `bls-min-pk` (DEC-251; built in Story 6.17). |
| `MAX_PAYLOAD_SIZE` | 1894 / 1910 / 1862 B | `MAX_BLOCK_SIZE − HEADER_SIZE`; per scheme, 18 / 22 / 13 node transfers or 39 / 16 / 28 balance entries per block. |
| `HASH_SIZE` | 32 B | SHA-256 per-slot integrity hash. |
| `FLASH_PAGE_SIZE` | 4096 B | RP2040 erase/write unit. |
| `SLOT_SIZE_BYTES` | 2048 B | `MAX_BLOCK_SIZE + HASH_SIZE`. |
| `BLOCKS_PER_PAGE` | 2 | `floor(FLASH_PAGE_SIZE / SLOT_SIZE_BYTES)`; must be ≥ 1. |
| RP2040 flash | 2 MB | `MAX_BLOCKS = 600` derived from `2 MB − code − control plane ÷ 4 KB page × 2 slots/page`. |
| Empty-slot marker | `version == 0` | Reserved by chain-types for storage empty-slot detection. |

A block plus its trailing hash never crosses a page boundary. Flash wear-lifetime is **not** quantified (gap — §7).

---

## 5. LoRa / Airtime Physical Constraints

Authoritative sources: [`moonblokz-simulator-algorythm.md`](./moonblokz-simulator-algorythm.md) §D4 (airtime) and [`moonblokz-radio-concept.md`](./moonblokz-radio-concept.md).

| Parameter | Value / rule | Note |
|---|---|---|
| Radio type | Direct LoRa (not LoRaWAN) | radio-concept. |
| LoRa packet payload | ~255 B | Radio fragmentation unit; blocks (up to `MAX_BLOCK_SIZE = 2016 B` on the reference tier) are fragmented across packets. |
| Airtime model | f(bandwidth, spreading factor, payload) | LoRa-style estimation. |
| Spreading factor | `5..=12` | Validated range. |
| Airtime payload clamp | payloads > 255 B clamped to 255 for estimation | Simulator airtime rule. |
| Regional duty-cycle limit | **NOT DOCUMENTED** | No regional duty-cycle number stated anywhere — see §7. |

---

## 6. Radio Memory Profiles

Authoritative source: [`moonblokz-radio-implementation.md`](./moonblokz-radio-implementation.md) §Deterministic Memory Profiles. Compile-time selected (`memory-config-small` / `-medium` / `-large`). Tuning is intentionally coupled across queue depths, matrix size, and buffer capacities.

| Capacity | Small | Medium | Large |
|---|---:|---:|---:|
| Approx. RAM target | 25 KB | 60 KB | 120 KB |
| Connection matrix (neighbors) | 10 | 30 | 100 |
| Incoming packet buffer | 20 | 30 | 50 |
| Wait pool | 5 | 10 | 20 |
| Echo response wait pool | 3 | 5 | 10 |
| Duplicate cache | 10 | 20 | 30 |
| Outgoing message queue | 2 | 3 | 8 |
| Incoming message queue | 2 | 3 | 10 |
| TX packet queue | 16 | 32 | 64 |
| RX packet queue | 2 | 3 | 5 |
| Process result queue | 2 | 5 | 10 |

RX state queue is fixed at **20** across all profiles. The connection matrix is a **bounded local neighbor view**, not a global network map — total network size is not capped by matrix capacity (multi-hop relaying extends reach).

---

## 7. Known Discrepancies & Gaps

Per [`AGENTS.md`](./AGENTS.md), divergences must be flagged, not silently resolved. Resolution is left to the maintainers.

| # | Type | Detail | Impact |
|---|---|---|---|
| D1 | **Resolved (source-confirmed)** | Radio `memory-config-medium` RAM is **~60 KB**, confirmed in the radio module source (`moonblokz-radio-lib/README.md` states `~60KB RAM`; `lib.rs` sets `CONNECTION_MATRIX_SIZE=30` and the medium-profile queue/buffer consts). [`moonblokz-radio-implementation.md`](./moonblokz-radio-implementation.md) §Medium was correct; [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §7.3's ~30-40 KB is an under-count. | **Material — propagated.** §3.2 and §3.4 here are corrected to 60 KB: BLS at the 1000-node default **exceeds 264 KB by ~17 KB** (tuning mandatory); Schnorr keeps ~47 KB (~18%). Architecture §7.3/§7.4/§12 and index line 101 have been updated to match (radio = 60 KB; BLS requires mandatory `MAX_NODES` tuning at the default profile). |
| D2 | **Resolved (Story 5.11, 2026-10-08)** | The active-chain window split ([`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §5, decision #23) is in source: the const generic is `SNAKE_CHAIN_LENGTH_MAX`, and the FR60 window check and the FR2 stopping condition read the chain-configured `W`. | A chain that leaves `W` at its default 500 is refused by a smaller build such as the BLS-small profile (capacity 300, §3) ([configuration specification](./moonblokz-configuration-specification.md) §6 check 6). |
| G1 | Gap | No regional LoRa **duty-cycle** limit documented anywhere. | Capacity/airtime planning for real deployments cannot be grounded in KB values. |
| G2 | Gap | Flash **wear-lifetime** not quantified ([`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) explicitly defers it). | Long-term durability budgeting is undefined. |
| G3 | Verify | Storage general slot-count formula divides by `MAX_BLOCK_SIZE` (2016), while the RP2040 placement contract uses `SLOT_SIZE_BYTES` (2048, includes the per-slot hash). | General capacity estimate may slightly over-count slots vs. the hardware geometry; confirm intended divisor. |
| G4 | **Resolved (DEC-183, DEC-203)** — first step 240 000 ms (4 min), later steps `grace_period_step_ms` 900 000 ms (15 min, ID 31). | `grace_period_window_ms` defaults to 30 s ([Configuration Module Specification](./moonblokz-configuration-specification.md) §4.1 ID 2). That is below the measured unsigned-block propagation median of 33 s, and well below the verification-inclusive estimate of §2.1 (about 1–2.3 min per block). | A grace this short expires before most nodes hold the top creator's block and starts needless deviations; it is to be sized with the deviation-approval reform (OG-015, AR-021). |

---

## Related Documents

- [`moonblokz-open-gaps-register.md`](./moonblokz-open-gaps-register.md) — consolidates unresolved gaps and verification items, including the duty-cycle, flash wear-lifetime, and storage divisor items listed here.
- [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) — authoritative for const-generics, RAM budget, stack frames, tuning.
- [`moonblokz-crypto-algorythm.md`](./moonblokz-crypto-algorythm.md) — authoritative for crypto constants and bounded aggregation.
- [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) / [`moonblokz-storage-architecture.md`](./moonblokz-storage-architecture.md) — authoritative for block size and flash geometry.
- [`moonblokz-radio-implementation.md`](./moonblokz-radio-implementation.md) — authoritative for radio memory profiles.
- [`moonblokz-simulator-algorythm.md`](./moonblokz-simulator-algorythm.md) — authoritative for the airtime model.
