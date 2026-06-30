# MoonBlokz System Constraints & Limits Reference

## Role of This Document

This file is a **consolidation/navigation layer** for the numeric constraints, capacity caps, memory budgets, and physical limits that are spread across the MoonBlokz knowledge base. It exists so that cross-cutting questions — hardware sizing, RAM/flash budgeting, capacity planning, deployment tuning — can be answered from one place instead of stitching values together from concept, architecture, algorithm, and implementation documents.

**This document is NOT authoritative for the values it lists.** Each value's authoritative home is the linked source document. If a value here diverges from its cited source, the **source wins**, and the divergence must be flagged and resolved per [`AGENTS.md`](./AGENTS.md). Every row therefore carries an explicit source link; this file restates the value and a one-line implication only, never the full derivation.

When a new numeric cap, budget, or limit is introduced anywhere in the knowledge base, it must also be registered here (per the Cross-Cutting Fact Placement rule in [`AGENTS.md`](./AGENTS.md)).

## How to Use

- **Hardware sizing question** → start with §3 (RAM budget) + §1 (capacity caps) + §6 (radio profiles), then read the cited architecture sections for rationale.
- **Capacity planning** → §1 (const-generics) + §2 (crypto bounds).
- **Flash / persistence sizing** → §4.
- **Radio / airtime** → §5 + §6.
- **Before trusting a margin figure** → read §7 (Known Discrepancies & Gaps) first.

---

## 0. Embedded Implementation Discipline (Project-Wide)

MoonBlokz targets constrained embedded environments first. Planning and implementation must therefore treat code size, RAM footprint, API surface, and transient stack use as load-bearing constraints, not as cleanup items.

Project-wide rules:

- **Keep code and memory at the absolute minimum needed for the current story and explicitly planned follow-up stories.** Do not add fields, counters, helper functions, accessors, buffers, or public APIs just because they might be useful someday.
- **Every new field/function must have a visible consumer.** The consumer can be current code or a concrete upcoming story/FR/ADR. If no consumer is identifiable, leave it out and reintroduce it later with a precise name and rationale.
- **Avoid convenience getters for internal structs.** If a struct is private module storage metadata, prefer direct field access inside the owning module and tests. Add accessors only when they are part of an intentional public API boundary.
- **Avoid convenience derives.** Do not add `Debug`, broad `Clone`, or similar derives to production/public types unless a real consumer requires them. If a derive is needed for fixed-array initialization or another concrete embedded reason, keep it narrow and document why.
- **Prefer private implementation details over public surface.** Public APIs are long-lived contracts; keep them minimal and story-driven.
- **Document memory impact for new persistent fields/buffers.** Any new bounded array, per-entry field, cache, or scratch buffer should identify what it costs and which requirement/story needs it.
- **Future-possible is not enough.** A vague future use does not justify embedded RAM/code cost. Reintroduce the element when the future story makes the use concrete.

This rule is intentionally stricter than desktop/server Rust style. It applies across blockchain, mempool, vote, storage, radio, telemetry, and future runtime work unless a subsystem-specific PRD/architecture explicitly overrides it.

## 1. Blockchain Capacity Caps (Const-Generics)

Authoritative source: [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §5 (Const generic catalog). Tuning levers: §12.

| Const generic | Default | Meaning / deployment implication |
|---|---:|---|
| `MAX_NODES` | 1000 | **Network-wide registered-node cap.** Sizes all node-id-indexed arrays (`NodeInfo`, `VoteEngine.accumulated_vote`); every node holds an entry for every registered node. Dominant RAM driver — see §3. |
| `SNAKE_CHAIN_LENGTH` (W) | 500 | Active-chain window length (bounded retention). |
| `VERIFICATION_HORIZON` (H) | 20 | [BC-FR58](./moonblokz-blockchain-prd.md) cheap-zone boundary; retained knowledge beyond the active window for rare chain switches. |
| `MAX_BLOCKS` | 600 | In-RAM block-table slots; 1:1 with RP2040 durable flash capacity (see §4). |
| `MAX_BRANCH_COUNT` | 40 | Chain-heads table capacity; collecting-state branch headroom. |
| `MEMPOOL_COMPACT_BYTES` | 20160 | Mempool compact buffer (~10 × `MAX_BLOCK_SIZE`). |
| `MEMPOOL_MAX_ENTRIES` | 128 | Mempool index capacity. |
| `MAX_TX_PER_BLOCK` | ~64 | Derived; [BC-FR45](./moonblokz-blockchain-prd.md) included-keys gathering. |
| `MAX_BLOCK_UTXO_OUTPUT` | 256 | Per-block spent-bit vector size (ADR-016). |
| `MAX_AGGREGATED_SIGNATURES` | 50 | Approval subgroup cap (ADR-015). Matches Schnorr per-block supporter capacity (§2). |
| `PUBLIC_KEY_SIZE` | 32 (Schnorr) / 96 (BLS) | Crypto-feature-dependent; see §2. |

---

## 2. Crypto Constants & Bounded Aggregation

Authoritative source: [`moonblokz-crypto-algorythm.md`](./moonblokz-crypto-algorythm.md) §Family-Specific Constants + §Global bounded-aggregation constants.

| Constant | Schnorr | BLS | Note |
|---|---:|---:|---|
| `SIGNATURE_SIZE` | 64 | 48 | Standalone signature. |
| `MULTI_SIGNATURE_SIZE` | 64 | 48 | |
| `PUBLIC_KEY_SIZE` | 32 | 96 | BLS keys are 3× larger — main RAM divergence (§3). |
| `PRIVATE_KEY_SIZE` | 32 | 32 | |
| `AGGREGATED_SIGNATURE_CONSTANT_SIZE` | 34 | 50 | Fixed part of an aggregated signature. |
| `AGGREGATED_SIGNATURE_VARIABLE_SIZE` | 32 | 0 | **Per-supporter incremental cost.** BLS aggregates into the constant slot (0 growth). |

Global, all families:
- `MAX_AGGREGATED_SIGNATURES = 50` (count cap).
- Practical per-approval-block supporter capacity within a ~2 KB block budget (minus header + 4 B node-id/signer): **~50 supporters Schnorr**, **~450 supporters BLS**. The global cap of 50 currently matches the Schnorr ceiling; a BLS-centered deployment could carry more if the cap were raised.

Cross-check: this is why the `ApprovalAccumulator` (§3) costs ~36 B/supporter on Schnorr (`4 + 32`) and ~4 B/supporter on BLS (`4 + 0`), at an identical ~2 KB allocated buffer — see [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §6.5.

---

## 3. RAM / SRAM Budget (RP2040, 264 KB)

Authoritative source: [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §7 (RAM budget verification) + §8 (stack frames). Total SRAM target: **264 KB (RP2040)**.

### 3.1 Chain-lib data-structure costs (at default const-generics)

| Structure | Schnorr | BLS | Driver |
|---|---:|---:|---|
| `NodeInfo` (SoA) | ~44 KB | ~108 KB | `MAX_NODES` × (`PUBLIC_KEY_SIZE` + 8 + 4) |
| `BlockTable` | 45.6 KB | 45.6 KB | `MAX_BLOCKS` × 76 B |
| `ChainHeadsTable` | 1.28 KB | 1.28 KB | `MAX_BRANCH_COUNT` × 32 B |
| `ApprovalAccumulator` | ~2 KB | ~2 KB | `MAX_BLOCK_SIZE` buffer (crypto-agnostic) |
| `EmitScratch` + scheduler + lifecycle + PRNG | ~2.1 KB | ~2.1 KB | `MAX_BLOCK_SIZE` buffer + small fields |
| **`moonblokz-blockchain` subtotal** | **~95 KB** | **~159 KB** | matches architecture §7.1 / §7.2 |
| `Mempool` (sibling `moonblokz-mempool`) | ~22–23 KB | ~22–23 KB | `MEMPOOL_COMPACT_BYTES` + compact index + small fields; authoritative layout in blockchain architecture §6.8 |
| `VoteEngine.accumulated_vote` (sibling `moonblokz-vote`) | ~4 KB | ~4 KB | `MAX_NODES` × 4 B |
| **blockchain + mempool + vote** | **~122 KB** | **~186 KB** | carried into §3.2 |

Raising `MAX_NODES` costs ~48 B/node (Schnorr) or ~112 B/node (BLS) across the node-id-indexed arrays (`NodeInfo` + `accumulated_vote`), derived from architecture §7.1 and corroborated by its §12.1 tuning levers. The default 1000-node cap already consumes most of the RP2040 budget, so caps an order of magnitude larger (for example a city-scale payment network) exceed RP2040 SRAM and require an MCU with substantially more RAM.

### 3.2 Full-system allocation

| Consumer | Schnorr | BLS |
|---|---:|---:|
| blockchain + mempool + vote | ~122 KB | ~186 KB |
| radio-lib (memory-config-medium) | ~60 KB | ~60 KB |
| crypto-lib scratch | ~4 KB | ~4 KB |
| storage page buffer | ~4 KB | ~4 KB |
| Embassy task stacks (5-6 × 4-6 KB) | ~24 KB | ~24 KB |
| Embassy executor + statics | ~4 KB | ~4 KB |
| **Subtotal allocated** | **~218 KB** | **~282 KB** |
| **Margin (264 KB − allocated)** | **~46 KB (~17%)** ✅ | **~−18 KB (exceeds 264 KB)** ❌ |

The radio figure is the **source-confirmed** `memory-config-medium = ~60 KB` (radio module `README.md` + `lib.rs` profile consts; see §6 and §7/D1), additive to the separately-counted Embassy task stacks. **Consequence:** at the default 1000-node profile the BLS backend **exceeds the 264 KB ceiling by ~18 KB** — const-generic tuning (e.g. `MAX_NODES 1000 → 500`, §3.4) is therefore **mandatory** for BLS, not optional. Schnorr keeps a comfortable ~46 KB (~17%) margin. The authoritative architecture §7.3/§7.4/§12 and the index have been updated to match this source-confirmed figure (radio = 60 KB).

### 3.3 Stack frames

- Chain-lib hosting Embassy task: **6 KB** stack (block-creation `BlockBuilder` is the hot spot).
- Other Embassy tasks (radio, USB console): **4 KB**.

### 3.4 Deployment tuning (BLS headroom)

Per-lever savings: [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) §12.1. The BLS margins below apply those levers on top of the source-confirmed ~60 KB radio figure (§3.2); architecture §12.2 now carries the same figures.

| Profile | `MAX_NODES` | `SNAKE_CHAIN_LENGTH` | BLS margin (radio = 60 KB) |
|---|---:|---:|---:|
| BLS-large (default) | 1000 | 500 | **~−18 KB (does not fit)** ❌ |
| BLS-medium | 500 | 500 | ~39 KB ✅ |
| BLS-small | 250 | 300 | ~73 KB ✅ |

Most impactful lever: `MAX_NODES 1000 → 500` saves ~56 KB (BLS) but halves the network-size cap — and is the minimum tuning needed to make BLS fit at all once the radio medium profile is counted at its real ~60 KB.

---

## 4. Flash / Storage Geometry

Authoritative sources: [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) (Appendix A + §E) and [`moonblokz-storage-architecture.md`](./moonblokz-storage-architecture.md) (RP2040 placement contract). Canonical block constants originate in `moonblokz-chain-types`.

| Constant | Value | Note |
|---|---:|---|
| `MAX_BLOCK_SIZE` | 2016 B | Canonical max serialized block (chain-types). |
| `HEADER_SIZE` | 122 B | Implied: `MAX_BLOCK_SIZE − MAX_PAYLOAD_SIZE`. |
| `MAX_PAYLOAD_SIZE` | 1894 B | `MAX_BLOCK_SIZE − HEADER_SIZE`. |
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
| LoRa packet payload | ~255 B | Radio fragmentation unit; blocks (up to `MAX_BLOCK_SIZE = 2016 B`) are fragmented across packets. |
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
| G1 | Gap | No regional LoRa **duty-cycle** limit documented anywhere. | Capacity/airtime planning for real deployments cannot be grounded in KB values. |
| G2 | Gap | Flash **wear-lifetime** not quantified ([`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) explicitly defers it). | Long-term durability budgeting is undefined. |
| G3 | Verify | Storage general slot-count formula divides by `MAX_BLOCK_SIZE` (2016), while the RP2040 placement contract uses `SLOT_SIZE_BYTES` (2048, includes the per-slot hash). | General capacity estimate may slightly over-count slots vs. the hardware geometry; confirm intended divisor. |

---

## Related Documents

- [`moonblokz-open-gaps-register.md`](./moonblokz-open-gaps-register.md) — consolidates unresolved gaps and verification items, including the duty-cycle, flash wear-lifetime, and storage divisor items listed here.
- [`moonblokz-blockchain-architecture.md`](./moonblokz-blockchain-architecture.md) — authoritative for const-generics, RAM budget, stack frames, tuning.
- [`moonblokz-crypto-algorythm.md`](./moonblokz-crypto-algorythm.md) — authoritative for crypto constants and bounded aggregation.
- [`moonblokz-storage-algorythm.md`](./moonblokz-storage-algorythm.md) / [`moonblokz-storage-architecture.md`](./moonblokz-storage-architecture.md) — authoritative for block size and flash geometry.
- [`moonblokz-radio-implementation.md`](./moonblokz-radio-implementation.md) — authoritative for radio memory profiles.
- [`moonblokz-simulator-algorythm.md`](./moonblokz-simulator-algorythm.md) — authoritative for the airtime model.
