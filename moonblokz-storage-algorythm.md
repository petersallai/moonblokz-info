# MoonBlokz Storage Algorithm Model

## Purpose of This Document

This document provides a more formal, algorithm-oriented description of MoonBlokz onboard storage as described in **MoonBlokz series part VIII — Onboard Storage**.

Its purpose is to capture:

- the formal storage problem the article is solving,
- the persistent data categories that must be retained,
- the storage-unit model built on flash erase regions,
- the sizing rules for block storage units,
- the integrity-check and acceptance rules for stored data,
- the redundancy and fallback behavior for control data,
- the crash-recovery behavior described by the article,
- and the wear-lifetime estimation model used to justify the design.

This file is the **primary knowledge-base document for the formal storage model** described by the article.

- Use [moonblokz-storage-concept.md](./moonblokz-storage-concept.md) for the higher-level explanation of why the storage layer exists and how it fits the broader MoonBlokz design.
- Use [moonblokz-storage-implementation.md](./moonblokz-storage-implementation.md) for RP2040-specific engineering constraints, XIP implications, Embassy API consequences, and open implementation questions.

## Source Basis

This document is based on:

- **MoonBlokz series part VIII — Onboard Storage** by Peter Sallai, published on Medium.

## Scope and Limits

This file captures the algorithmic structure explicitly stated or directly implied by the article, including:

- flash page and sector granularity as inputs to the model,
- the required persisted node data,
- block-unit and control-unit separation,
- stored-block integrity-check structure,
- control-unit checksum and redundancy rules,
- startup acceptance and fallback behavior after interruption,
- and the lifetime-estimation logic used in the article.

It does **not** define details the article leaves open, such as:

- the exact internal indexing structures for finding blocks in flash,
- the full replacement algorithm for choosing which exact storage unit to rewrite next,
- a complete repository API,
- the exact scheduling policy for storage writes relative to radio work,
- or any storage compaction or garbage-collection algorithm beyond what the article states.

## Core Terminology Used in This Document

To keep the three storage knowledge-base files aligned, this document uses the following terms consistently:

- **Flash page** — the write/programming granularity described by the article.
- **Flash sector** — the erase granularity described by the article.
- **Storage unit** — one erase-sized persistent region, treated as the main MoonBlokz storage building block.
- **Block storage unit** — a storage unit dedicated to blockchain blocks plus their stored hashes.
- **Control storage unit** — a storage unit dedicated to node-local control data plus checksum.
- **Stored block hash** — the extra 32-byte hash stored next to each serialized block for integrity checking.
- **Primary control copy** — the control unit read first during normal operation.
- **Backup control copy** — additional redundant control units used if the primary copy fails integrity validation.

## Algorithmic Problem Statement

The storage model must satisfy all of the following constraints simultaneously:

1. retain enough blockchain data to reconstruct working chain state,
2. retain enough node-local control data for the node to recover its identity and startup context,
3. operate on flash where erase granularity is larger than ordinary logical updates,
4. tolerate interrupted writes and reboot without assuming full transactional storage machinery,
5. keep wear distributed widely enough that flash endurance remains practical,
6. and coexist with a bounded-chain blockchain model rather than unlimited historical retention.

## Section A — Flash Constraints Used by the Model

## A1. Flash asymmetry

The article’s storage model depends on three asymmetric flash properties:

- reads are comparatively easy,
- writes are slower,
- erases are the most expensive operation.

## A2. RP2040 granularity values used by the article

For the RP2040 target discussed by the article:

- programming is performed in **256-byte pages**,
- erasing is performed in **4 KB sectors**.

The storage model uses the erase unit, not the page size, as the top-level layout unit.

## A3. Shared-flash assumption

The article assumes the same external flash chip is used both for:

- firmware storage,
- and persistent MoonBlokz application data.

Therefore, total available storage for blockchain data is the flash capacity minus the space reserved for the application binary and minus the space reserved for control storage units.

## Section B — Required Persisted Data

## B1. Required blockchain data

The node must persist the valid blocks it has received.

The article treats these stored blocks as the basis for reconstructing the chain.

## B2. Required control data

The node must also persist a control record containing:

- the node’s private key,
- the node’s own ID,
- initialization parameters,
- a copy of the configuration block,
- and a `CRC32` checksum.

## B3. Mutability assumptions

The article states or implies the following update behavior:

- private key: written during initialization,
- node ID: written during initialization,
- initialization parameters: written during initialization,
- configuration-block copy: written when the configuration block is received during the gathering phase and not modified afterward except by reconstructing it again from chain knowledge if needed.

The article does not describe frequent rewriting of this control record during ordinary steady-state operation.

## Section C — Storage-Unit Layout Model

## C1. Main layout rule

Each **4 KB erase region** is treated as one independent **storage unit**.

This means the storage model is explicitly sector-aligned.

## C2. Storage-unit classes

There are two storage-unit classes.

### C2.1. Block storage units

A block storage unit stores multiple blockchain blocks.

Each stored block consists of:

- the serialized block bytes,
- plus one additional 32-byte stored hash.

### C2.2. Control storage units

A control storage unit stores:

- private key,
- node ID,
- initialization parameters,
- configuration-block copy,
- `CRC32` checksum.

## C3. Block storage-unit capacity formula

The article defines the number of blocks per storage unit as:

`N = FLOOR(STORAGE_UNIT_SIZE / (MAX_BLOCK_SIZE + BLOCK_HASH_SIZE))`

Where:

- `STORAGE_UNIT_SIZE` is the erase-unit size,
- `MAX_BLOCK_SIZE` is the configured maximum serialized block size,
- `BLOCK_HASH_SIZE` is the size of the extra stored integrity hash.

## C4. Default-capacity example given by the article

In the article’s default example:

- `MAX_BLOCK_SIZE = 2000` bytes,
- `BLOCK_HASH_SIZE = 32` bytes,
- `STORAGE_UNIT_SIZE = 4096` bytes,
- therefore one storage unit stores **2 blocks** together with **2 × 32-byte hashes**.

## Section D — Integrity Model

## D1. Stored-block integrity rule

For each block stored in a block storage unit:

1. the serialized block bytes are stored,
2. a 32-byte hash is stored alongside them,
3. when the block is read back, the node recomputes the hash from the stored bytes,
4. the recomputed hash is compared with the stored hash,
5. if they differ, the block is treated as corrupted.

## D2. Meaning of the stored-block hash

The article gives two explicit reasons for this extra hash:

- detect storage corruption or incomplete writes,
- allow a much cheaper consistency check than full digital-signature verification on every read.

## D3. Authenticity boundary

The article explicitly states that the stored hash does **not** replace the block signature.

Therefore:

- stored hash protects storage consistency,
- block signature protects authenticity.

## D4. Control-data integrity rule

For each control storage unit:

1. the control record is stored,
2. a `CRC32` checksum is stored with it,
3. on read, the checksum is verified,
4. if verification fails, that control copy is treated as invalid.

## Section E — Redundancy Model for Control Data

## E1. Redundancy precondition

The article states that the control data fits into a single storage unit, roughly **2200 bytes** in the default configuration with a 2 KB maximum block size.

## E2. Default redundancy factor

Because the control record is small, the article stores multiple copies.

Default: **3 copies**.

## E3. Normal read path

During normal operation:

1. read the primary control storage unit,
2. verify its CRC,
3. if valid, use it,
4. if invalid, attempt backup control copies.

## E4. Failure condition

The article’s practical failure condition is:

- the node is operationally lost only if **all** control copies are corrupted.

## E5. Why redundancy is asymmetric

The redundancy model is stronger for control data than for blocks because:

- block corruption can be compensated for by erasing and re-requesting from the network,
- but loss of control data may prevent correct node operation entirely.

## Section F — Crash-Recovery Rules

## F1. General acceptance rule after reboot

After reboot, a storage unit is accepted only if its integrity check succeeds.

Otherwise it is treated as invalid.

## F2. Control-unit crash behavior

The article describes control-unit updates as separate erase/write operations on separate copies.

Therefore, an interrupted update can corrupt at most one control copy at a time.

Recovery rule:

1. verify primary control copy,
2. if invalid, verify backup copies,
3. use the first valid copy found.

## F3. Versioning note for control data

The article explicitly downplays versioning concerns for control data because control fields change rarely. The main mutable control-related payload is the stored configuration block, and the article notes that this can be reconstructed from chain information if needed.

## F4. Block-unit crash behavior

If a block write is interrupted:

- the target block is lost,
- and, because erase is storage-unit-wide, the other block in the same unit may also be lost.

The article treats this as acceptable because missing blocks can be requested again from the network and invalid units can simply be repopulated later.

## Section G — Wear and Lifetime Estimation Model

## G1. Endurance assumption used in the article

The lifetime estimate uses a flash endurance of **10,000 erase cycles**.

## G2. Control-unit wear contribution

Control units contribute little to wear because they are written only:

- during initialization,
- and when the first configuration block is gathered.

## G3. Main source of wear

The main wear source is block storage.

The article’s reasoning is:

- `snake_chain` causes old blocks to expire,
- receiving a new block usually means overwriting expired storage,
- and this causes rewrites to be distributed across the available storage units.

## G4. Default-cycle interpretation in the article

In the default layout where each storage unit stores two blocks:

- one full `snake_chain` cycle rewrites the same storage unit **twice**.

## G5. 2 MB example used by the article

After reserving space for the binary and control storage units, the article estimates that approximately:

- **600 blocks** can be stored,
- around **500 blocks** form the chain itself,
- around **100 blocks** remain as fork headroom.

## G6. Time-based lifetime examples given by the article

The article gives these example interpretations:

- at **1 block per minute**, one full chain cycle takes about **500 minutes**, and expected lifetime is about **4.8 years**,
- at **1 block every 5 minutes**, expected lifetime rises to nearly **20 years**,
- with **16 MB flash**, lifetime grows much further because block-storage capacity is much larger.

## G7. Formal takeaway

The article’s algorithmic conclusion is not that one exact lifetime number is universally guaranteed. It is that, under the model’s assumptions, wear is spread broadly enough that flash endurance is not the primary limiting factor for practical deployment.

## Section H — Explicit Algorithmic Boundaries Left Open by the Article

The article leaves several important areas underspecified.

It does not formally define:

- how blocks are indexed across storage units,
- how the node maps a requested chain block to a physical storage unit quickly,
- whether there is a journal or metadata area beyond the units described,
- how block replacement order is encoded on flash,
- how rebuild or scan-on-boot works in detail,
- or how concurrent radio and storage demands are scheduled beyond acknowledging that they interact.

These omissions should remain explicit rather than being silently completed with invented rules.

## Review Notes

Post-change review against `moonblokz-info` documentation rules:

- **Consistency:** This file keeps the formal storage rules aligned with the earlier bounded-chain model and does not redefine blockchain semantics.
- **Logical soundness:** It separates persisted-data requirements, storage-unit math, integrity checks, fallback behavior, and wear estimation into distinct sections.
- **Feasibility:** The formal model remains implementable on the target flash hardware exactly because it is built around erase-unit alignment and bounded retention.
- **Redundancy:** Conceptual motivation and RP2040 engineering discussion are left primarily to the companion concept and implementation files.
- **Source fidelity:** All formulas, capacities, and recovery behaviors come from the cited article or are direct restatements of its explicit reasoning.
