# MoonBlokz Crypto Algorithm Model

## Purpose of This Document

This document provides a formal, algorithm-oriented description of the MoonBlokz cryptographic design in `moonblokz-crypto-lib`.

Its purpose is to capture:

- the public crypto API,
- the compile-time feature-selection model,
- the concrete backend set,
- family-specific constants,
- signature-type behavior,
- deterministic Schnorr signing,
- Schnorr aggregation behavior,
- the BLS proof-of-possession suites,
- and explicit structural limits present in the codebase.

This file is the **primary knowledge-base document for the formal crypto model**.

**Status (2026-11-16): built in Story 6.16.** The backends, features, encodings and proof-of-possession methods below are built (DEC-235, DEC-244..248); the four earlier backends (`schnorr-malachite`, `schnorr-num-bigint-dig`, `schnorr-crypto-bigint`, `bls-bls12_381-bls`) are removed. BLS on the chain is Story 6.17.

- Use [moonblokz-crypto-concept.md](./moonblokz-crypto-concept.md) for the higher-level explanation of design goals and trade-offs.
- Use [moonblokz-crypto-implementation.md](./moonblokz-crypto-implementation.md) for engineering implications, compatibility cautions, and maintenance guidance.

## Source Basis

This document is based on:

- `moonblokz-crypto-lib/Cargo.toml`, `src/lib.rs` and `README.md` (the trait model, constants and guards, which the target keeps except where stated);
- the ratified Story 6.16 decisions (`_bmad-output/implementation-artifacts/6-16-*.md`, D1–D8; DEC-235, DEC-244..250), which define the target modules `src/schnorr_k256_signer.rs` and `src/bls_zkcrypto_signer.rs`;
- BIP-340, RFC 9380 and the IETF BLS signature draft (ciphersuites, KeyValidate, FastAggregateVerify, proof of possession).

Where earlier article-era summaries and the codebase diverge, this file treats the code as authoritative; where a section is marked as ratified, the code is being brought to it.

## Scope and Limits

This document captures the crypto structure explicitly visible in the current library, including:

- the trait model,
- compile-time backend selection,
- active constants,
- serialized size rules,
- bounded aggregation,
- current signing and verification flows,
- backend-specific role mapping,
- and the tests that define expected behavior.

It does **not** attempt to provide:

- formal cryptographic proofs,
- product-level rationale,
- benchmark claims unless they are encoded or implied by the current codebase,
- or engineering migration strategy beyond what the current structure exposes.

## Core Terminology Used in This Document

To stay aligned with the companion conceptual and implementation files, this document uses the following terms consistently:

- **Signature** — the ordinary single-signer artifact represented by `Signature`.
- **Aggregation-ready signature** — the single-signer artifact represented by `MultiSignature`, intended for later combination.
- **Aggregated signature** — the combined artifact represented by `AggregatedSignature`.
- **Algorithm family** — the broader family, currently Schnorr or BLS.
- **Backend** — one concrete compile-time-selected implementation, named by its scheme feature (`schnorr`, `bls-min-sig`, `bls-min-pk`).

When this document refers to exact code constructs, it uses the public type names `Signature`, `MultiSignature`, and `AggregatedSignature` directly.

## Algorithmic Problem Statement

The current library is designed to provide a single crypto API that can:

1. create signatures from a private key,
2. verify those signatures against public keys,
3. create aggregation-ready signatures intended for later combination,
4. aggregate multiple aggregation-ready signatures,
5. verify aggregated signatures,
6. serialize and deserialize all public crypto artifacts,
7. work under `#![no_std]`,
8. and keep backend choice at compile time.

## Crate-Level Design Rules

The root library file enforces two critical compile-time rules.

### Rule 1: exactly one scheme feature must be enabled

Compilation fails if more than one of these features is enabled:

- `schnorr`
- `bls-min-sig`
- `bls-min-pk`

### Rule 2: at least one scheme feature must be enabled

Compilation also fails if none of them is enabled.

### Consequence

The public crypto API is always bound to exactly one backend at build time.

## Feature Model

Built in Story 6.16 (DEC-246). The feature names the scheme, not the implementation:

| Feature | Backend | Scheme |
|---|---|---|
| `schnorr` (default) | `k256` 0.14 (`arithmetic`, `schnorr`; no `precomputed-tables`) | BIP-340 as implemented by `k256` 0.14, with the DEC-194 aggregate |
| `bls-min-sig` | zkcrypto `bls12_381` 0.9.2 (`groups`, `pairings`, `experimental`; no `sqrt-table`) | IETF `BLS_SIG_BLS12381G1_XMD:SHA-256_SSWU_RO_POP_` |
| `bls-min-pk` | zkcrypto `bls12_381` 0.9.2 | IETF `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_` |

A private `bls` helper feature, enabled by both BLS variants, may gate their shared code; the guards also refuse it without a variant. Every crate that forwards a backend forwards these three names, `default` containing `schnorr`; chain-types refuses both BLS features until Story 6.17.

## Public API Re-Export Model

The library re-exports same-named public types from the selected backend module:

- `Crypto`
- `PublicKey`
- `Signature`
- `MultiSignature`
- `AggregatedSignature`

This means the embedding project sees one stable API shape regardless of which backend is active.

## Public Traits

The current formal API is defined through five traits and one error enum.

### `CryptoError`

The public error enum currently contains:

- `InvalidPrivateKey`
- `InvalidPublicKey`
- `InvalidSignature`

### `SignatureTrait`

This trait defines:

- `new(bytes: &[u8]) -> Result<Self, CryptoError>`
- `serialize(&self) -> &[u8; SIGNATURE_SIZE]`

### `MultiSignatureTrait`

This trait defines:

- `new(bytes: &[u8]) -> Result<Self, CryptoError>`
- `serialize(&self) -> &[u8; MULTI_SIGNATURE_SIZE]`

In MoonBlokz terminology, `MultiSignature` is the **aggregation-ready signature** type.

### `AggregatedSignatureTrait`

This trait defines:

- `new(bytes: &[u8]) -> Result<Self, CryptoError>`
- `serialize(&self, out: &mut [u8]) -> Result<usize, CryptoError>`
- `get_count(&self) -> usize`
- `serialized_len(&self) -> usize`

### `PublicKeyTrait`

This trait defines:

- `new(bytes: &[u8]) -> Result<Self, CryptoError>`
- `serialize(&self) -> &[u8; PUBLIC_KEY_SIZE]`

### `CryptoTrait`

This trait defines the main cryptographic behavior:

- `new(private_key_bytes: [u8; PRIVATE_KEY_SIZE]) -> Result<Self, CryptoError>`
- `public_key(&self) -> &PublicKey`
- `sign(&self, message: &[u8]) -> Signature`
- `multi_sign(&self, message: &[u8]) -> MultiSignature`
- `verify_multi_signature(&self, message: &[u8], multi_signature: &MultiSignature, public_key: &PublicKey) -> bool`
- `verify_signature(&self, message: &[u8], signature: &Signature, public_key: &PublicKey) -> bool`
- `aggregate_signatures(&self, signatures: &[&MultiSignature], public_keys: &[&[u8; PUBLIC_KEY_SIZE]], message: &[u8]) -> Result<AggregatedSignature, CryptoError>` — built in Story 6.16 (DEC-194); a length mismatch, empty input and more than `MAX_AGGREGATED_SIGNATURES` inputs are errors; the BLS backends take and ignore the keys
- `verify_aggregated_signature(&self, message: &[u8], aggregated_signature: &AggregatedSignature, public_keys: &[&PublicKey]) -> bool`
- `prove_possession(&self) -> Signature` and `verify_possession(&self, public_key: &PublicKey, proof: &Signature) -> bool` — built in Story 6.16 (DEC-248); see §Proof of Possession

## Signature Role Model

The current role mapping is:

- `Signature` — ordinary single-signer artifact
- `MultiSignature` — aggregation-ready single-signer artifact
- `AggregatedSignature` — final combined artifact verified against multiple public keys

This role model is common across backends, even where internal structures differ.

## Family-Specific Constants

The library defines family-specific serialization constants in `src/lib.rs`.

### Schnorr constants

When the `schnorr` family is active:

- `SIGNATURE_SIZE = 64`
- `MULTI_SIGNATURE_SIZE = 64`
- `PUBLIC_KEY_SIZE = 32`
- `PRIVATE_KEY_SIZE = 32`
- `AGGREGATED_SIGNATURE_CONSTANT_SIZE = 34`
- `AGGREGATED_SIGNATURE_VARIABLE_SIZE = 32`

### BLS constants

Per variant (DEC-247; `bls-min-sig` keeps the current BLS sizes):

| Constant | `bls-min-sig` | `bls-min-pk` |
|---|---:|---:|
| `SIGNATURE_SIZE`, `MULTI_SIGNATURE_SIZE` | 48 | 96 |
| `PUBLIC_KEY_SIZE` | 96 | 48 |
| `PRIVATE_KEY_SIZE` | 32 | 32 |
| `AGGREGATED_SIGNATURE_CONSTANT_SIZE` | 50 | 98 |
| `AGGREGATED_SIGNATURE_VARIABLE_SIZE` | 0 | 0 |

### Global bounded-aggregation constants

Across all families, the library currently uses:

- `MAX_AGGREGATED_SIGNATURES = 50`
- `MAX_AGGREGATED_SIGNATURE_BYTES = AGGREGATED_SIGNATURE_CONSTANT_SIZE + AGGREGATED_SIGNATURE_VARIABLE_SIZE * MAX_AGGREGATED_SIGNATURES`

This gives the library a compile-time upper bound for serialized aggregated-signature storage.

The **practical per-backend ceiling** for how many supporters fit into one approval evidence block differs because `AGGREGATED_SIGNATURE_VARIABLE_SIZE` differs between backends. With a roughly 2 KB block-size budget, minus headers and a 4-byte node-id per signer, the resulting capacities are approximately 50 supporters for Schnorr (`VARIABLE_SIZE = 32`) and approximately 450 supporters for BLS (`VARIABLE_SIZE = 0`). The current global cap of 50 therefore matches the Schnorr capacity today; a BLS-centered deployment could carry more supporters per approval evidence block if the global cap were raised.

## Schnorr Algorithm Family Model

Built in Story 6.16 (DEC-235): one backend, BIP-340 as implemented by `k256` 0.14 (`k256::schnorr`). It replaces three backends with a custom scheme (little-endian integers, trimmed hash inputs, x-only verification that accepted the negated point) that did not behave alike: the malachite and num-bigint-dig backends failed to verify about 1 in 256 of their own signatures.

### Curve model

secp256k1:

- field modulus `p = FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F`
- group order `q = FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141`
- generator point `G` with the standard secp256k1 base point coordinates

Every integer and 32 B value below is big-endian, as in BIP-340. `H_tag(tag, x) = SHA256(SHA256(tag) ‖ SHA256(tag) ‖ x)` is the BIP-340 tagged hash.

### Keys

- Private key: 32 B, `1 ≤ d < q`; otherwise `Crypto::new` returns `InvalidPrivateKey`.
- Public key: the 32 B x-only BIP-340 key. `PublicKey::new` lifts it to the point with even `y` and rejects `x ≥ p` and an `x` off the curve (`InvalidPublicKey`). As in BIP-340, a private key whose point has odd `y` signs as `q − d`.

## Schnorr Signing Model

- `sign` and `multi_sign` produce identical bytes: BIP-340 signing with `aux_rand = 0³²` over the whole message. With zero auxiliary randomness the nonce depends on the key and the message only, so signing is deterministic (FR63).
- The signature is `r ‖ s`, 32 B each: `r` is the x coordinate of the even-`y` nonce point `R`, and `s = k + e·d mod q` with the BIP-340 challenge `e = int(H_tag("BIP0340/challenge", r ‖ P ‖ m)) mod q`.
- `sign` is infallible in the trait; it could fail only if the nonce or `s` were zero on all 256 retries (probability about 2⁻²⁰⁴⁸).
- `Signature::new` and `MultiSignature::new` parse exactly 64 B and refuse `r ≥ p`, `s ≥ q`, `r = 0` and `s = 0` (refusing `s = 0` deviates from BIP-340 negligibly).

So the role distinction between `sign` and `multi_sign` is semantic and protocol-oriented, not mathematical, for Schnorr.

## Schnorr Single-Signature Verification Model

`verify_signature` and `verify_multi_signature` are BIP-340 verification: compute `R = s·G − e·P` and accept only if `R` is not the identity, has even `y`, and its `x` coordinate equals `r`.

## Schnorr Aggregation Model

Built in Story 6.16 (DEC-194, encoding DEC-235). Half-aggregation of `k` BIP-340 signatures `(rᵢ, sᵢ)` by keys `Pᵢ` (32 B x-only), all on one message `m`, in the given order.

### Input validation

Aggregation returns an error if the input is empty, exceeds `MAX_AGGREGATED_SIGNATURES`, or has a different number of signatures and public keys.

### Serialized aggregate structure

1. `count` as `u16` big-endian (unlike the little-endian wire integers of DEC-195 and DEC-218, to stay consistent with BIP-340)
2. `s` as 32 bytes
3. one 32-byte `r` per contributing signature

The serialized length is `34 + 32 · count`. `AggregatedSignature::new` parses this prefix (the approval-evidence payload continues after it, DEC-195), reports `serialized_len`, and refuses `count = 0`.

### Deterministically weighted aggregation

1. `seed = H_tag("MoonBlokz/aggregate", r₁ ‖ … ‖ r_k ‖ P₁ ‖ … ‖ P_k ‖ m)`, every value untrimmed as on the wire.
2. `wᵢ = (int(H_tag("MoonBlokz/weight", seed ‖ i)) mod (q − 1)) + 1` for `i = 0 … k − 1`, `i` as `u32` big-endian, so every weight lies in `[1, q − 1]`.
3. `s = Σ wᵢ·sᵢ mod q`.

The seed binds every signer's nonce and key and the message, as published half-aggregation schemes with a security proof do (DEC-194); the earlier form derived it from `Σ rᵢ mod p` and the message only. An external cryptographic review is recommended, not required.

### Aggregate verification

Verification refuses `count = 0`, `count > MAX_AGGREGATED_SIGNATURES`, `count ≠ public_keys.len()`, `s ≥ q`, and any `rᵢ ≥ p` or not on the curve. Otherwise it:

1. recomputes `seed`, every `wᵢ`, and every BIP-340 challenge `eᵢ = int(H_tag("BIP0340/challenge", rᵢ ‖ Pᵢ ‖ m)) mod q`,
2. lifts each `rᵢ` to `Rᵢ` with even `y`,
3. accepts only if `s·G = Σ wᵢ·Rᵢ + Σ (wᵢ·eᵢ)·Pᵢ` as points, so neither the negated point nor an empty aggregate verifies.

The sums use no multiscalar call: two constant-time scalar multiplications per signer into an accumulator, about 6 KB of stack (DEC-244).

No precondition on the keys is needed for Schnorr, since each signature binds its own key through its challenge; the chain still verifies a proof of possession at registration. Signer distinctness (no key twice in one aggregate) is the caller's check.

### Important consequence

The aggregate verification depends on:

- the exact order of signatures and public keys,
- the message,
- the per-signer `r` values,
- and the deterministic weight derivation.

So the Schnorr aggregate artifact is not order-free: signer position is part of the aggregate definition.

## Proof of Possession

Built in Story 6.16 (DEC-248). `prove_possession` returns a `Signature` (`SIGNATURE_SIZE` bytes), so the registration's `new_key_signature` keeps its size per backend; `verify_possession` checks it against a parsed `PublicKey`.

- **Schnorr:** a BIP-340 signature by the key's owner over `"MoonBlokz/pop" ‖ public key (32 B)`, the ASCII tag unterminated; verification is BIP-340 verification of that message. The tag keeps proofs and ordinary signatures in separate domains: no ordinary signature over 32 B of data serves as a proof, and no proof as an ordinary signature. The separation is by message, not by challenge tag, so an ordinary signature over exactly `"MoonBlokz/pop" ‖ key` (45 B) is a proof too; no MoonBlokz message has that form (transactions start with their type byte, blocks with the version, supports with `MBSUP`, chain configurations with their entry count), and Schnorr aggregation does not rely on the proof (code review, 2026-11-16).
- **BLS (both variants):** the suite's `PopProve` / `PopVerify` with its proof-of-possession DST (`BLS_POP_BLS12381G1_XMD:SHA-256_SSWU_RO_POP_` for `bls-min-sig`, `BLS_POP_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_` for `bls-min-pk`): the proof is `sk · H_pop(serialized public key)` in the signature group; verification runs KeyValidate on the key and the pairing check.

The registration transaction carries this proof ([Blockchain Algorithm Model](./moonblokz-blockchain-algorythm.md) §5 Registration Transaction Structure).

## BLS Algorithm Family Model

Built in Story 6.16 (DEC-247): zkcrypto `bls12_381` 0.9.2, the IETF proof-of-possession suite in two variants through one generic core, whose parameters are the signature group, the key group, the DST and the pairing argument order; a build compiles one variant. It replaces the `bls12_381-bls` wrapper, whose `h0(m) = hash_to_scalar(m)·G1` made every message hash's discrete logarithm public, so one signature yielded a valid signature on any message.

### Variants

| | `bls-min-sig` | `bls-min-pk` |
|---|---|---|
| Ciphersuite | `BLS_SIG_BLS12381G1_XMD:SHA-256_SSWU_RO_POP_` | `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_` |
| Signatures (hash-to-curve, RFC 9380) | G1, 48 B | G2, 96 B |
| Public keys (key aggregation) | G2, 96 B | G1, 48 B |
| Aggregate | 50 B | 98 B |

Min-sig keeps wire artifacts small; min-pk halves the `MAX_NODES` key array.

### Keys

- Private key: 32 B big-endian (the suite's I2OSP form), `1 ≤ sk < r`, `r` the group order.
- `PublicKey::new` is the suite's KeyValidate: compressed-point decoding with the subgroup check, and the identity refused. Signature parsing refuses the identity and non-subgroup points the same way.

### Signing and verification

- `sign` and `multi_sign` are the suite's `Sign`: `σ = sk · H(m)`, hashed into the signature group with the suite's DST.
- Verification is one pairing equation, computed with two `pairing()` calls and a comparison (the multi-Miller loop needs an allocator in `bls12_381`).

### BLS aggregate structure

`count` as `u16`, then the aggregated signature (48 or 96 B): 50 or 98 B, constant with respect to the signer count. The count is big-endian, as in the Schnorr aggregate.

### BLS aggregate creation

Same message for every signer: `σ = Σ σᵢ`, with no per-signer coefficients. The input checks are the trait's (empty input, more than `MAX_AGGREGATED_SIGNATURES`, length mismatch); the key bytes are taken and ignored.

### BLS aggregate verification

The suite's FastAggregateVerify: `count == public_keys.len()`, at most `MAX_AGGREGATED_SIGNATURES`, and `σ` verified against `Σ Pᵢ` and the message. **Precondition:** every key carries a verified proof of possession, since the suite has no rogue-key coefficients (DEC-248).

## Serialization and Deserialization Model

The codebase makes serialization a core part of the formal contract.

### `Signature` and `MultiSignature`

Both support:

- `new(bytes)` for parsing
- `serialize()` for fixed-size output

### `AggregatedSignature`

Aggregated signatures support:

- `new(bytes)` for parsing family-specific serialized forms
- `serialize(out)` for caller-provided buffers
- `serialized_len()` to query exact encoded length
- `get_count()` to expose signer count

### Buffer-oriented design

The aggregate serialization API writes into a caller-provided output buffer instead of allocating a new `Vec`.

That is part of the library’s bounded-memory formal contract.

## Error and Failure Model

The current code uses a small explicit error vocabulary.

### Construction failures

- invalid private key bytes → `InvalidPrivateKey`
- invalid public key bytes → `InvalidPublicKey`
- invalid signature or aggregate bytes → `InvalidSignature`

### Operational false results

Verification methods return `bool`, not `Result`.

So malformed or mismatched verification inputs usually produce:

- `false`

rather than a richer error.

## Test-Backed Expected Behavior

The library test suite in `src/lib.rs` establishes several expected behaviors.

### Positive behavior

The tests cover:

- normal sign and verify
- aggregate sign and verify
- single-item aggregate verification
- serialization and deserialization of all major artifact types
- aggregate count reporting
- multiple signer creation and verification

### Negative behavior

The tests also cover rejection or failure for:

- altered messages
- wrong public keys
- invalid serialized lengths
- empty aggregate input
- aggregate verification with empty public-key list

### Byte-level vectors

Built in Story 6.16; they replace the `schnorr-crypto-bigint` regression vector:

- `schnorr`: all 19 official BIP-340 vectors — signing through the trait for vectors 0 and 15–18 (`aux_rand = 0`), verification for 0–18 with a parse failure counting as invalid — plus pinned DEC-194 aggregate vectors;
- `bls-min-sig`: RFC 9380 Appendix J.9.1 (hash to G1) plus signature vectors pinned once after an out-of-band cross-check against an independent implementation;
- `bls-min-pk`: RFC 9380 Appendix J.10.1 (hash to G2) and the Ethereum consensus-spec BLS tests (min-pk, POP DST): all `sign`, `verify` and `fast_aggregate_verify` cases of `ethereum/bls12-381-tests` v0.1.2 (commit `89ff3a8`), vendored in `src/bls_eth_vectors.rs`; its `deserialization_G1` / `G2` cases run under both variants (the identity is refused by KeyValidate).

The story's negative tests add the aggregate refusals above (`count = 0`, the negated point, `s ≥ q`, an unliftable `r`, swapped key or signature order, a dropped signer) and the proof-of-possession refusals (another key's proof, an ordinary signature over the key bytes).

## Business Analyst View: What the Formal Model Enables

From a product-capability perspective, the current formal model gives MoonBlokz:

- single-signer proofs,
- aggregation-ready signer contributions,
- aggregated multi-party evidence,
- bounded-size serialized artifacts,
- and compile-time backend choice without API churn.

## Architect View: Structural Interpretation

Architecturally, the current formal crypto model has these layers:

1. **public trait and constant layer**
2. **compile-time family and backend selection layer**
3. **family-specific serialization model**
4. **family-specific signing and aggregation logic**
5. **bounded aggregate storage model**
6. **test-backed behavior layer**

## Related Documents

- [`moonblokz-crypto-concept.md`](./moonblokz-crypto-concept.md) — why MoonBlokz distinguishes the signature kinds and keeps crypto compile-time swappable.
- [`moonblokz-crypto-implementation.md`](./moonblokz-crypto-implementation.md) — engineering constraints and compatibility risks of the current crypto library.
