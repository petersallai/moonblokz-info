# MoonBlokz Storage Implementation Notes

## Purpose of This Document

This document captures implementation-facing implications of MoonBlokz onboard storage as described in **MoonBlokz series part VIII — Onboard Storage**.

It complements the conceptual and algorithm storage documents by identifying:

- what engineering constraints follow from the RP2040 flash model,
- why XIP changes how persistent writes must be approached,
- how Embassy’s flash API shape influences implementation structure,
- why flash-aware layout decisions must remain explicit,
- how redundancy and corruption detection affect practical implementation,
- and which important details still remain open in the source material.

- Use [moonblokz-storage-concept.md](./moonblokz-storage-concept.md) for the strategic role and conceptual trade-offs of MoonBlokz storage.
- Use [moonblokz-storage-algorythm.md](./moonblokz-storage-algorythm.md) for the formal storage-unit model, integrity-check rules, fallback behavior, and lifetime-estimation logic.

## Source Basis

This document is based on:

- **MoonBlokz series part VIII — Onboard Storage** by Peter Sallai, published on Medium.

## Scope and Intent

This is not a repository walkthrough and not a complete engineering specification. Instead, it records the practical implementation consequences of the article so future engineering work can preserve the stated constraints without filling the gaps with undocumented assumptions.

## Relationship to the Earlier MoonBlokz Architecture

The earlier MoonBlokz documents already establish several constraints that storage must respect:

- the chain is bounded through `snake_chain`,
- radio processing remains timing-sensitive,
- embedded memory is limited,
- and persistent storage cannot be treated as unlimited archival history.

Part VIII adds the implementation-facing persistence side of those same constraints. Storage logic must therefore be engineered as a bounded embedded subsystem, not as a transparent database layer hidden underneath the chain.

## RP2040-Specific Engineering Constraints

The article makes the RP2040’s flash behavior a direct implementation concern.

### External flash rather than large internal non-volatile storage

The RP2040 typically boots from external QSPI flash.

Implementation consequence:

- MoonBlokz persistent data shares the same chip with firmware,
- so flash partitioning or reserved-offset planning is mandatory,
- and storage capacity calculations must always account for firmware occupancy first.

### Granularity mismatch between writing and erasing

The article uses:

- **256-byte programming pages**,
- **4 KB erase sectors**.

Implementation consequence:

- data layout must be sector-aware,
- write paths must avoid pretending that arbitrary in-place logical updates are cheap,
- and any mutation that requires erase must be treated as a whole-storage-unit operation.

### Endurance is finite

Implementation consequence:

- rewrites must be distributed,
- hot-spot updates should be avoided,
- and wear behavior must be part of data-layout design rather than an afterthought.

## XIP Implications for Storage Engineering

The article’s XIP discussion creates several non-optional engineering consequences.

### Flash writes interfere with ordinary execution from flash

Because the RP2040 executes code directly from external flash in normal operation, erase or write activity occupies the flash interface.

Implementation consequence:

- persistent updates are not merely slow I/O,
- they are events that may interfere with normal instruction fetch,
- so write routines must be designed with explicit awareness of system execution state.

### Timing-sensitive work must be considered during writes

The article explicitly connects this concern to MoonBlokz radio activity.

Implementation consequence:

- storage code cannot assume it can mutate flash at arbitrary moments without broader runtime impact,
- and higher-level scheduling or pausing strategy will matter even though the article does not yet define the full policy.

### Running flash operations from RAM is part of the expected model

The article states that firmware must typically perform flash updates carefully, usually by running the flash operation from RAM.

Implementation consequence:

- any real implementation should verify that the chosen flash-access path respects the RP2040 execution constraints,
- and low-level storage integration cannot be treated as an ordinary library call with no platform-specific execution considerations.

## Embassy Flash API Consequences

The article uses Embassy as the software foundation and emphasizes the shape of the API rather than any one code sample.

### Read / erase / write are separate first-class operations

Implementation consequence:

- storage abstractions should preserve these distinctions internally,
- rather than flattening flash into a fake random-write byte store.

### Size constants are part of correctness

The article explicitly calls out `ERASE_SIZE`, `PAGE_SIZE`, and `WRITE_SIZE`.

Implementation consequence:

- these values should drive layout and validation logic,
- not be treated as informal comments or assumptions,
- and future target changes should flow through these constraints rather than silently breaking the storage model.

### Blocking access remains a valid practical choice

The article rejects DMA as a fundamental solution because DMA does not remove XIP blocking.

Implementation consequence:

- a simpler blocking implementation can still be the correct engineering choice,
- provided the system-level effects of the blocking operation are handled deliberately.

## Storage-Layout Engineering Consequences

The article’s storage-unit design implies several practical responsibilities.

### Sector-aligned partitioning should remain explicit

Each erase-sized region becomes one storage unit.

Implementation consequence:

- storage code should model units explicitly,
- not merely raw offsets,
- because integrity handling, replacement, and recovery all operate at unit granularity.

### Block bytes must remain reproducible and verifiable

Because each stored block carries a stored hash computed from the block bytes:

- block serialization must be deterministic,
- read-back verification must use the exact stored bytes,
- and incomplete writes must be detectable by recomputation.

### Control data should remain a single coherent record

The control unit is treated as a compact recovery bundle.

Implementation consequence:

- it is reasonable to implement control data as one serialized structure with one checksum,
- rather than as many separately mutable small records,
- because the article’s recovery logic expects whole-copy validation and fallback.

### Fixed-length initialization parameters imply reserved payload space

The article defines a fixed **100-byte** initialization-parameter field.

Implementation consequence:

- storage structures should preserve this fixed reservation exactly as part of the control record,
- even though MoonBlokz itself does not interpret the field.

## Integrity and Recovery Engineering Consequences

The article’s integrity rules are simple, but they have direct implementation implications.

### Stored-hash validation should happen before trusting block contents

Implementation consequence:

- block-loading paths should verify the stored hash before treating the block as usable persisted data,
- and corruption handling should treat a mismatch as invalid local storage rather than as proof of invalid blockchain authorship.

### CRC validation should gate use of control copies

Implementation consequence:

- startup logic should validate the primary control copy first,
- then fall back through backups,
- and should not silently merge partially valid control fragments across copies, because the article describes whole-copy fallback.

### Recovery strategy is class-dependent

Implementation consequence:

- block corruption should trigger re-fetch or repopulation logic,
- while control corruption should trigger backup-copy selection,
- because the article assigns different recovery mechanisms to the two data classes.

## Wear-Distribution Engineering Consequences

The article’s lifetime reasoning depends on an implementation property that should remain explicit: block rewrites should be spread over the available block-storage space rather than concentrated in a few locations.

Implementation consequence:

- replacement behavior must preserve the intended broad distribution of erases,
- because the durability argument depends on it.

The article does not define the exact bookkeeping mechanism for this distribution, so the knowledge base should not pretend that one specific allocator or ring-buffer design is already mandated.

## Capacity-Planning Consequences

The article’s example calculations create practical planning duties.

### Firmware reservation must be explicit

The storage calculation only works after reserving space for:

- the application binary,
- and the control storage units.

Implementation consequence:

- deployment-specific flash maps matter,
- and a storage implementation should make the reserved regions explicit rather than implicit.

### Fork headroom must not be consumed accidentally

The article’s 2 MB example distinguishes chain capacity from extra fork headroom.

Implementation consequence:

- implementations should preserve the idea that not all stored blocks belong to the ideal main chain length,
- because extra capacity is intentionally reserved for unresolved forks.

## Security Boundary Explicitly Left by the Article

The article acknowledges a security trade-off in storing the private key directly in flash.

Implementation consequence:

- this design assumes accidental corruption is a more immediate concern than physical key extraction,
- and deployments with stronger physical-security requirements may need secure-element or equivalent protection.

This should be documented as a threat-model boundary, not silently treated as universally sufficient key protection.

## Important Open Engineering Questions

The article leaves several implementation-relevant points open.

It does not fully specify:

- the concrete crate/module API of `moonblokz-storage`,
- the exact serialized control-record format,
- the exact on-flash directory or lookup scheme for blocks,
- how startup scanning discovers valid units efficiently,
- how writes are coordinated with radio activity in a full runtime,
- how corruption events are surfaced to higher layers,
- or whether future targets beyond RP2040 must preserve the same unit sizes.

These are real design questions, but they are not settled by the article and should remain explicitly open in the knowledge base.

## Engineering Guidance That Follows Directly from the Article

Without inventing missing detail, the article still supports several concrete implementation guidelines:

- preserve flash semantics in the storage abstraction instead of hiding them,
- keep block and control persistence paths distinct,
- validate stored bytes before trusting them,
- treat crash recovery as integrity-checked acceptance plus fallback,
- reserve storage space explicitly for firmware and redundant control units,
- and preserve wide wear distribution as a design invariant.

## Review Notes

Post-change review against `moonblokz-info` documentation rules:

- **Consistency:** This document remains aligned with the concept and algorithm storage files and keeps RP2040/XIP/Embassy concerns in the implementation layer.
- **Logical soundness:** It focuses on engineering consequences rather than re-explaining the full storage concept or repeating all sizing formulas.
- **Feasibility:** The notes describe a realistic embedded implementation direction without claiming missing repository details as settled.
- **Redundancy:** Formal rules and conceptual motivations are referenced back to the companion storage files rather than duplicated in full.
- **Source fidelity:** Every implementation recommendation is grounded in explicit statements or direct engineering consequences of the cited article.
