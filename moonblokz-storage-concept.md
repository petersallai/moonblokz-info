# MoonBlokz Storage Concept Model

## Purpose of This Document

This document explains the conceptual operating model of MoonBlokz onboard storage as described in **MoonBlokz series part VIII — Onboard Storage**.

Its purpose is to describe:

- why storage becomes a first-class subsystem in MoonBlokz,
- why microcontroller flash cannot be treated like a normal filesystem or database,
- how `snake_chain` makes bounded blockchain storage possible,
- what kinds of data must survive on the node,
- why MoonBlokz separates block data from control data,
- and why redundancy, integrity checks, and crash tolerance are central to the storage design.

This file is intentionally focused on conceptual understanding rather than exact API shape, byte-level layout tables, or repository-level implementation detail.

- Use this document to understand **what storage must achieve in MoonBlokz**, **why the design is organized around flash erase units**, and **why bounded blockchain retention is viable at all on microcontroller-class hardware**.
- Use [moonblokz-storage-algorythm.md](./moonblokz-storage-algorythm.md) for the more formal storage model, sizing formulas, integrity-check flow, and recovery behavior.
- Use [moonblokz-storage-implementation.md](./moonblokz-storage-implementation.md) for engineering constraints related to RP2040 flash, XIP, Embassy flash access, endurance planning, and implementation-facing cautions.

## Source Basis

This document is based on:

- **MoonBlokz series part VIII — Onboard Storage** by Peter Sallai, published on Medium.

## Relationship to Earlier MoonBlokz Documents

This file should be read after:

- [moonblokz-overview.md](./moonblokz-overview.md), which explains the project mission and operating environment,
- [moonblokz-technology.md](./moonblokz-technology.md), which introduces the embedded architectural constraints,
- and preferably after the blockchain documents, because the storage article relies directly on the bounded-chain `snake_chain` model.

This document answers the conceptual question:

**What kind of persistent-storage design does MoonBlokz need when blockchain data must survive resets on flash-based microcontrollers that have slow erases, asymmetric writes, bounded endurance, and shared firmware/data storage?**

## Business Analyst View: What Problem the Storage Layer Solves

With blockchain, radio, and cryptography already defined at a conceptual level, Part VIII addresses a practical question that determines whether MoonBlokz can run on real devices at all: how can a node keep enough blockchain state persistently without exhausting flash, blocking the device excessively, or wearing out the storage medium too quickly?

The article frames storage as an infrastructure constraint rather than a secondary implementation detail. MoonBlokz nodes must be able to:

- survive resets and power loss,
- recover enough chain state to continue operating,
- keep the minimum persistent identity needed to act as a node,
- and do so on hardware where erase and write behavior is coarse-grained, slow, and finite.

This means MoonBlokz storage is conceptually not an ordinary persistence layer. It is a **bounded, flash-aware blockchain retention model** designed specifically for constrained non-volatile memory.

## Why Flash Cannot Be Treated Like Ordinary Mutable Storage

The article begins from the physical realities of flash.

On typical microcontrollers:

- flash retains data without power,
- reads are relatively easy,
- writes are slower,
- erases are slower still,
- and small in-place overwrite behavior cannot be assumed.

This matters because a blockchain node might seem, at first glance, to need ordinary record updates or append-only persistence. The article argues that this intuition is misleading on microcontroller flash.

Conceptually, flash is asymmetric:

- **reading is cheap compared with mutation**,
- **programming and erasing have different granularities**,
- and **repeated rewrites of the same area create a wear hotspot**.

So the storage problem is not merely “where to put blocks.” It is “how to organize persistent blockchain state so that the design remains compatible with erase-unit granularity, bounded endurance, and timing-sensitive node behavior.”

## Why RP2040 Flash Characteristics Matter to the Design

The RP2040 makes the constraints more concrete because it usually uses an external QSPI flash chip for both firmware and persistent application data.

The article highlights three conceptual consequences:

- firmware storage and MoonBlokz data share the same physical flash device,
- writes happen in smaller units than erases,
- and storage cannot be designed independently of execution behavior because code normally executes directly from that flash.

This makes flash usable for MoonBlokz, but only if the storage model is aligned with hardware reality rather than pretending flash behaves like a general-purpose disk or database.

## Why XIP Changes the Meaning of a Flash Write

The RP2040 uses **execute-in-place** (XIP), so code normally runs directly from flash instead of first being copied into RAM.

This is beneficial because it preserves RAM, but it also means flash mutation is operationally intrusive. While flash erase or program activity is in progress, the flash interface is busy and ordinary execution from flash cannot continue in the usual way.

Conceptually, this turns storage updates into a system-level event rather than a private persistence detail. In MoonBlokz, storage activity must coexist with radio handling and other timing-sensitive work, so persistent updates have to be performed carefully and intentionally.

## Why Embassy Flash Support Fits the MoonBlokz Model

The article uses Embassy’s RP2040 flash support as the software-side foundation.

Conceptually, this is a good fit because the interface exposes flash with explicit:

- read operations,
- erase operations,
- and write operations,
- along with explicit erase-size and write-size constraints.

That matters because MoonBlokz does not want an abstraction that hides flash semantics. The storage model depends directly on those semantics. The design intentionally accepts that higher-level logic must be built on top of erase-unit and write-unit rules.

The article also notes that DMA-based flash access does not remove the key XIP limitation. For MoonBlokz, that means a simpler blocking model is conceptually acceptable because asynchronous transfer alone would not make flash mutation truly non-disruptive.

## Why MoonBlokz Can Use a Bounded Chain at All

A conventional blockchain grows forever, which is a poor fit for finite flash.

The storage design is only viable because MoonBlokz already uses `snake_chain`.

The article’s conceptual argument is:

- old blocks do not need to be retained forever,
- essential active state can be preserved near the living end of the chain,
- and bounded retention is therefore compatible with correct future operation.

This is the key enabling idea behind MoonBlokz storage. If the chain required permanent full history, flash-based microcontroller storage would be fundamentally misaligned with the protocol. Instead, `snake_chain` shifts the burden away from unbounded history retention and toward explicit state preservation rules.

## What Must Be Stored Persistently

The article identifies two broad persistent data categories.

### 1. Valid blockchain blocks

Nodes must retain the valid blocks they have received because these blocks are the basis for reconstructing the chain.

The article treats these blocks as indispensable. If the chain could be reconstructed without them, they would not need to exist as blockchain content in the first place.

### 2. Control data required to restore node identity and operation

In addition to the chain itself, the node must preserve a minimal set of control data.

The article explicitly includes:

- the node’s private key,
- the node’s own identifier,
- initialization parameters,
- a stored copy of the configuration block,
- and integrity data for this control record.

Conceptually, this means MoonBlokz distinguishes between:

- **chain state that can be recovered or requested again**, and
- **node-local identity data whose loss can make the node unable to operate correctly**.

## Why the Layout Separates Block Data from Control Data

The storage layout is built directly around the 4 KB erase region.

Each erase-sized region becomes one independent **storage unit**.

The article then divides storage units into two conceptual classes:

- **block storage units** for blockchain blocks,
- **control storage units** for node-local recovery-critical data.

This separation matters because the consequences of corruption are very different.

- If a block is lost, the node may be able to request it again from the network.
- If the private key or node identifier is lost, the node may no longer be able to function as itself.

So MoonBlokz does not treat all persistent bytes as equally critical. The layout encodes different recovery expectations directly into the storage model.

## Why Stored Blocks Carry Additional Hashes

For each stored block, the article adds a 32-byte hash alongside the serialized block bytes.

Conceptually, this hash serves two roles:

- it detects storage corruption or incomplete writes by comparing stored bytes with a recomputed hash,
- and it avoids treating expensive signature verification as the first integrity test on every read.

The article is explicit that this hash does **not** replace the block signature. The signature still provides authenticity, while the stored hash provides storage-integrity checking.

## Why Control Data Uses CRC Instead of the Block Hash Scheme

Control storage units use a `CRC32` checksum rather than the per-block stored-hash method.

Conceptually, the two mechanisms reflect two different data classes:

- block data is stored as many individually recoverable blockchain objects,
- while control data is a small composite recovery record that is verified as one unit.

The article does not claim that CRC provides authenticity. It is a corruption-detection mechanism appropriate to this local control record.

## Redundancy as a Consequence of Recovery Asymmetry

The control data is small enough to fit comfortably within one storage unit, so the article introduces redundancy by storing multiple copies.

By default, it uses three copies.

Conceptually, redundancy is justified because:

- block loss is usually recoverable from the network,
- but control-data loss may be fatal for the node.

So control-data replication is not just a reliability improvement. It reflects the deeper asymmetry between **recoverable chain content** and **locally irreplaceable node identity state**.

## Crash Consistency as an Acceptance Rule

The article frames crash consistency in a deliberately pragmatic way.

After reboot, a storage unit is accepted only if its integrity check succeeds. If not, it is treated as invalid and the node recovers by using:

- a valid backup copy for control storage,
- or the network for missing block data.

This means MoonBlokz storage is conceptually designed around **integrity-checked acceptance after interruption**, not around transactional multi-step rollback machinery.

The article explicitly notes that separate updates of redundant control copies mean one interrupted update can corrupt at most one control copy at a time.

## Why Block Loss Is Acceptable Within Limits

Block storage is described as less forgiving than control storage.

If a block write is interrupted, the affected block is lost, and because erase happens at storage-unit granularity, the other block in the same unit may be lost as well.

Conceptually, MoonBlokz accepts this because the storage model is part of a networked blockchain system, not a stand-alone archival store. Missing blocks can be requested again, and corrupted units can be treated as invalid until repopulated.

This is a very important conceptual boundary: MoonBlokz storage is designed for **recoverable operational persistence**, not for absolute permanent local retention of every stored byte.

## Why Flash Lifetime Appears Practical

The article concludes with a durability argument based on erase-cycle estimates and expected block rates.

The conceptual takeaway is not the exact number of years in one hardware configuration, but the design principle:

- writes are distributed over many storage units,
- `snake_chain` turnover naturally rotates which units are rewritten,
- control units are rarely updated,
- and realistic network block rates make flash wear acceptable.

So the storage layout is not only logically compatible with MoonBlokz. The article argues that it is also operationally viable over practical deployment lifetimes.

## Strategic Trade-off Introduced by the Storage Design

The article makes one broader trade-off explicit.

MoonBlokz does not solve finite storage by compressing history arbitrarily or by introducing a general-purpose database layer. Instead, it combines:

- bounded blockchain semantics from `snake_chain`,
- flash-aligned storage units,
- separate treatment of recoverable and irreplaceable data,
- and simple integrity-based crash recovery.

This shifts complexity away from raw storage capacity and into protocol rules such as balance replay, configuration carry-forward, and UTXO preservation. The storage design is therefore best understood as one part of a broader system-level trade-off between **protocol complexity** and **embedded storage feasibility**.

## Explicit Limits of the Source Material

The article gives a clear storage direction, but it does not fully specify every implementation detail.

It does **not** fully define, for example:

- the exact in-flash indexing structures for locating blocks,
- the complete update schedule for replacing expired blocks,
- exact concurrency handling with radio activity during writes,
- the full API shape of the storage repository,
- or the detailed persistent metadata needed for all reconstruction workflows.

Those points should not be invented in the knowledge base as settled facts.

## Review Notes

Post-change review against `moonblokz-info` documentation rules:

- **Consistency:** This document stays aligned with the earlier MoonBlokz blockchain documents by treating `snake_chain` as the reason bounded persistent retention is possible.
- **Logical soundness:** It separates conceptual storage goals from more formal sizing, integrity-check, and implementation detail.
- **Feasibility:** The document preserves the article’s claim that flash-aware bounded storage is practical on target hardware under realistic write rates.
- **Redundancy:** Detailed formulas, recovery flows, and RP2040-specific engineering details are left to the companion storage files instead of being duplicated here.
- **Source fidelity:** All claims are drawn from or directly implied by the cited Part VIII article; no additional storage mechanisms are introduced as established behavior.
