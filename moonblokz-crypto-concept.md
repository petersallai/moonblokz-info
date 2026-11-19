# MoonBlokz Crypto Concept Model

## Purpose of This Document

This document explains the conceptual operating model of MoonBlokz cryptography based on the current `moonblokz-crypto-lib` codebase and the earlier Part VI article context.

Its purpose is to describe:

- why cryptography is needed in MoonBlokz,
- which practical constraints shape the design,
- how MoonBlokz separates ordinary signatures, aggregation-ready signatures, and aggregated evidence,
- why MoonBlokz keeps multiple backend options behind one API,
- why Schnorr remains the primary practical direction,
- and which architectural limits are explicit in the current library.

This file is intentionally focused on conceptual understanding rather than exact API shape, byte-level serialization, or backend-internal implementation detail.

**Status (2026-11-16):** the backend set described here is built in Story 6.16 (DEC-235, DEC-245..247); the four earlier backends are removed.

- Use this document to understand **what cryptography must achieve in MoonBlokz**, **why the current trade-offs exist**, and **how the crypto subsystem fits the broader product and protocol context**.
- Use [moonblokz-crypto-algorythm.md](./moonblokz-crypto-algorythm.md) for the formal API, constants, serialization rules, and signing or verification flows.
- Use [moonblokz-crypto-implementation.md](./moonblokz-crypto-implementation.md) for engineering constraints, maintenance cautions, dependency concerns, and test implications.

## Source Basis

This document is based on:

- `moonblokz-crypto-lib/Cargo.toml`, `src/lib.rs` and `README.md`
- the ratified Story 6.16 decisions (`_bmad-output/implementation-artifacts/6-16-*.md`) and the epics entry for Story 6.16

It also remains consistent with the broader architectural direction described in the MoonBlokz article series, but this file treats the codebase as the authoritative source where code and earlier summaries differ.

## Relationship to Earlier MoonBlokz Documents

This file should be read after:

- [moonblokz-overview.md](./moonblokz-overview.md), which explains the project mission and operating environment,
- [moonblokz-technology.md](./moonblokz-technology.md), which explains the architecture and embedded-platform constraints,
- and preferably after the blockchain documents, because MoonBlokz cryptography exists to support transaction integrity, block legitimacy, and approval evidence.

This document answers the conceptual question:

**What kind of cryptographic design does MoonBlokz need when it runs on constrained devices, communicates over low-bandwidth links, and still needs both direct authorization and compact multi-party approval evidence?**

## Core Terminology Used Across the Crypto Documents

To keep the three crypto knowledge-base files aligned, this terminology is used consistently throughout:

- **Signature** — an ordinary single-signer authorization artifact.
- **Aggregation-ready signature** — the single-signer artifact represented in code as `MultiSignature`, intended to be combined later with other signer contributions.
- **Aggregated signature** — the combined evidence artifact represented in code as `AggregatedSignature`.
- **Backend** — one concrete implementation selected at compile time.
- **Algorithm family** — the broader crypto family, currently Schnorr or BLS.

The code uses the exact public type names `Signature`, `MultiSignature`, and `AggregatedSignature`. This document keeps those names when referring to code, but uses the phrase **aggregation-ready signature** to make the role of `MultiSignature` clearer for readers.

## Business Analyst View: What Problem the Crypto Layer Solves

MoonBlokz uses cryptography as an operational requirement, not as an isolated security add-on.

The current library clearly models three practical needs:

- ordinary signatures for transactions and blocks,
- aggregation-ready signatures for approval workflows,
- and aggregated signatures for compact multi-party evidence.

This means the crypto layer directly supports:

- authentication of blockchain actions,
- verification of author identity,
- evidence that multiple participants approved something,
- and proof formats compact enough for constrained transport and storage.

Conceptually, MoonBlokz does not only need “a secure signature algorithm.” It needs a **deployable signature subsystem** that can run on embedded targets, keep serialized data small, and still support evidence compression when many signers are involved.

## Why MoonBlokz Crypto Is Constraint-Driven

MoonBlokz cryptography is shaped by:

- radio and storage byte limits,
- repeated verification by many independent nodes,
- `no_std` deployment expectations,
- the need to keep runtime behavior bounded,
- and the need to evolve the backend later without changing the rest of the system.

This is why the library is not written as a generic “best available crypto” package. It is written as a **replaceable, compile-time-selected crypto layer** optimized for MoonBlokz’s operating environment.

## Required Conceptual Properties

The current library design implies the following required properties.

### Small serialized artifacts

MoonBlokz treats signatures, public keys, and aggregated signatures as protocol objects, not merely in-memory structures.

This matters because their sizes directly affect:

- radio transfer cost,
- blockchain storage cost,
- and evidence payload growth.

### Efficient verification

The public model makes verification a first-class concern because signatures are produced once but verified many times.

### Explicit support for aggregation

Aggregation is not an afterthought. The public model distinguishes direct authorization from approval evidence that can later be combined.

### Compile-time replaceability

Backend choice is a Cargo feature decision with compile-time guards.

Conceptually, this means MoonBlokz treats cryptography as **swappable infrastructure with a stable public surface**, not as a hard-wired dependency.

### Deterministic behavior where possible

Schnorr signing is BIP-340 with zero auxiliary randomness, so the nonce is derived deterministically from the private key and the message.

Conceptually, this reflects a preference for reducing fragile runtime randomness dependencies in constrained environments.

### Bounded aggregation

The library defines a compile-time constant `MAX_AGGREGATED_SIGNATURES`, currently `50`, that sets a global upper bound on aggregated-signature storage.

Conceptually, that means aggregated approval evidence is intentionally bounded. MoonBlokz is not modeling an unbounded "aggregate as many as you want" scheme. It is modeling a resource-conscious, fixed-limit approach suitable for embedded software and predictable serialization.

The **practical ceiling** for how many supporters can be carried inside one approval evidence block is backend-dependent, because `AGGREGATED_SIGNATURE_VARIABLE_SIZE` differs between backends. With a roughly 2 KB block-size budget, minus headers and a 4-byte node-id per signer, the backend-specific capacity currently works out as:

- Schnorr: `VARIABLE_SIZE = 32` bytes per signer → approximately 50 supporters per evidence block.
- BLS: `VARIABLE_SIZE = 0` bytes per signer → approximately 450 supporters per evidence block.

The global `MAX_AGGREGATED_SIGNATURES = 50` therefore matches the Schnorr capacity today but is a conservative cap for BLS. Any chain-config validator that enforces `required_support ≤ MAX_AGGREGATED_SIGNATURES` must treat the effective ceiling as a backend-calibrated value rather than as a single fixed number across all deployments.

## Why MoonBlokz Distinguishes Three Signature Roles

The current codebase exposes three separate signature concepts:

- `Signature`
- `MultiSignature`
- `AggregatedSignature`

This separation is conceptually important even where implementations overlap internally.

### `Signature`

This is the ordinary single-signature concept used for direct authorization or validation.

### `MultiSignature` as an aggregation-ready signature

This is the single-signer contribution intended to participate in later aggregation.

### `AggregatedSignature`

This is the compact evidence artifact produced by combining multiple aggregation-ready signatures.

This design shows that MoonBlokz treats approval evidence as a distinct business and protocol concern, not just as “more signatures.”

## Why MoonBlokz Keeps Multiple Backends

The library selects one of three scheme features at build time (DEC-246): `schnorr` (default), `bls-min-sig` and `bls-min-pk`. The feature names the scheme, not the implementation, and every crate of the stack forwards it.

Conceptually, this means MoonBlokz has separated three concerns:

1. **what the rest of the system expects from cryptography**,
2. **which algorithm family is active**,
3. **which concrete backend implements that family**.

This is a very important design choice. It means MoonBlokz is not betting the entire project on one library’s long-term viability: the earlier Schnorr arithmetic backends were replaced without changing the public API.

## Conceptual Meaning of the Current Backend Mix

Only libraries that build and run on the devices as `no_std` **without an allocator** qualify (Project Lead, 2026-11-14). That rule removed the allocating malachite and num-bigint-dig backends; the crypto-bigint backend went because it took minutes per operation and was not constant-time, and the earlier BLS wrapper because it was forgeable (DEC-245).

### Schnorr family

One backend: standard BIP-340 through `k256` (DEC-235), with MoonBlokz's own half-aggregation (DEC-194) on top. Using an audited library's single-signature path rather than custom signature code also gives MoonBlokz the official test vectors.

### BLS family

One backend on zkcrypto `bls12_381`, the IETF proof-of-possession suite in two variants (DEC-247): `bls-min-sig` keeps signatures small (48 B) for the radio, `bls-min-pk` keeps keys small (48 B) for builds with many nodes. On the RP2040 the smaller keys do not win: min-pk's hash-to-G2 signing needs a 70 KiB Core 1 stack, so it fits fewer nodes (211) than min-sig (258) ([System Constraints](./moonblokz-system-constraints.md) §3.4). BLS stays for stronger hardware and faster networks, where its constant-size aggregation pays off; it runs on the chain since Story 6.17.

## The Core Conceptual Trade-off

The current library structure preserves the central trade-off already visible in the earlier article context.

- **BLS** offers the conceptually cleaner aggregation story.
- **Schnorr** offers a standard, library-backed path (BIP-340) with explicit control over its bounded aggregation behavior.

The codebase reinforces this by keeping both families available, while making Schnorr the default through the crate’s default feature:

- default feature: `schnorr`

So conceptually, MoonBlokz is still optimized for **practical deployability first, backend replaceability second, ideal aggregation compactness third**.

## Why Schnorr Aggregation Is a Deliberate Compromise

In the Schnorr backends, aggregated signatures are not modeled as perfectly constant-size proofs.

Instead, the current code keeps:

- a signer count,
- one combined scalar-like value,
- and one serialized `r` component per contributing signer.

Conceptually, this is a compromise between:

- storing every signature separately,
- and requiring ideal constant-size aggregation from every supported algorithm family.

MoonBlokz therefore accepts a **bounded, partially compressed aggregation model** rather than requiring mathematically identical aggregation properties from all backends.

## Why BLS Remains Conceptually Important

The BLS backend is still relevant conceptually because it demonstrates the other end of the design space:

- constant-size aggregated signatures,
- explicit aggregation of public keys for verification,
- and a rogue-key defence that rests on a verified proof of possession for every key rather than on per-signer coefficients (DEC-248).

Even if MoonBlokz does not currently center the system around BLS, keeping it in the codebase preserves an important architectural option.

## Deterministic Nonces as a Conceptual Safety Choice

Schnorr signing is BIP-340 with zero auxiliary randomness, so the nonce follows from the private key and the message alone.

Conceptually, this shows that MoonBlokz treats weak randomness on constrained devices as a real operational risk. The design therefore shifts from “generate a random nonce safely every time” to “derive a reproducible nonce safely from controlled inputs.”

## Fixed Limits as Part of the Conceptual Model

The library uses fixed limits and fixed-size serialization conventions.

Most importantly:

- `MAX_AGGREGATED_SIGNATURES = 50` as the current global compile-time cap,
- with a backend-dependent practical ceiling (see "Bounded aggregation" above): approximately 50 supporters per evidence block for Schnorr, approximately 450 for BLS, determined by each backend's `AGGREGATED_SIGNATURE_VARIABLE_SIZE` together with the per-signer node-id overhead and the available block payload budget.

Conceptually, this means the crypto subsystem is designed for predictable upper bounds in memory and payload handling, while still allowing a BLS-using deployment to carry substantially more supporters per approval evidence block if the global cap is later raised.

This fits the broader MoonBlokz philosophy of bounded behavior on constrained devices.

## Why Compact Public-Key Encoding Matters Conceptually

In the Schnorr backend, the public key is the BIP-340 x-only key, with the even-`y` convention used when reconstructing the point.

Conceptually, this means MoonBlokz prefers compact public-key representation and accepts reconstruction logic as part of verification behavior.

That is not merely a low-level encoding detail. It reflects the larger system priority of keeping durable and transmitted artifacts small.

## Replaceability as an Explicit Design Principle

The library’s compile-time guards reject both:

- enabling more than one concrete backend,
- and enabling none.

Conceptually, this means replaceability is tightly controlled rather than loosely optional.

MoonBlokz wants the rest of the system to experience a single crypto surface, while making the concrete backend an explicit build choice.

## Current Strategic Limitation: No Post-Quantum Path in the Library

The current codebase contains only classical elliptic-curve-style Schnorr and BLS options.

So the present conceptual state is clear:

- MoonBlokz has a replaceable classical crypto subsystem,
- but it does not yet have a practical post-quantum-ready implementation path in the library.

No standardized post-quantum signature fits the consensus path today (sizes, the `MAX_NODES` key array, no compact aggregation); a migration path — a key-type tag and a signed key rotation, LMS / HSS for low-volume signatures — is an open, undecided question (O-039).

## Main Conceptual Conclusions

### Cryptography is a protocol-capacity concern

In MoonBlokz, serialized crypto artifacts affect bandwidth, storage, and validation workload directly.

### Aggregation is essential, but bounded and backend-dependent

MoonBlokz values aggregation enough to model it explicitly, but the exact aggregation behavior depends on the active algorithm family.

### Replaceability is not accidental

The current library deliberately separates the public crypto contract from backend choice.

### Deterministic signing behavior reduces embedded risk

The Schnorr designs show a clear preference for deterministic nonce derivation over dependence on runtime entropy quality.

### Fixed limits are part of the design, not an implementation accident

The current global aggregation limit of 50 signatures is a conceptual commitment to bounded behavior, with a backend-dependent practical ceiling per approval evidence block (approximately 50 for Schnorr, approximately 450 for BLS).

### MoonBlokz still prioritizes practical deployability

The default backend choice continues to favor the practical Schnorr path over a more elegant but differently constrained BLS-centered design.

## Architect View: Structural Meaning of the Current Library

From an architectural point of view, the current crypto library establishes several durable ideas:

- cryptography is a separable crate with a stable public surface,
- backend choice is compile-time selected,
- aggregation is part of the main crypto contract,
- bounded serialization is a first-class concern,
- and algorithm-family and backend concerns are intentionally separated.

The replacement of the earlier Schnorr and BLS backends without an API change (Story 6.16) confirms that MoonBlokz expects crypto backend evolution over time.

## Related Documents

- [`moonblokz-crypto-algorythm.md`](./moonblokz-crypto-algorythm.md) — the exact current public crypto API, constants, backends, and aggregation behavior.
- [`moonblokz-crypto-implementation.md`](./moonblokz-crypto-implementation.md) — engineering constraints, maintenance risks, and what must be preserved across backends.
