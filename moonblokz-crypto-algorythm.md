# MoonBlokz Crypto Algorithm Model

## Purpose of This Document

This document provides a formal, algorithm-oriented description of the current MoonBlokz cryptographic design as implemented in `moonblokz-crypto-lib`.

Its purpose is to capture:

- the public crypto API,
- the compile-time feature-selection model,
- the concrete backend set,
- family-specific constants,
- signature-type behavior,
- deterministic Schnorr nonce derivation,
- current Schnorr aggregation behavior,
- BLS wrapper behavior,
- and explicit structural limits present in the codebase.

This file is the **primary knowledge-base document for the formal crypto model currently implemented in code**.

- Use [moonblokz-crypto-concept.md](./moonblokz-crypto-concept.md) for the higher-level explanation of design goals and trade-offs.
- Use [moonblokz-crypto-implementation.md](./moonblokz-crypto-implementation.md) for engineering implications, compatibility cautions, and maintenance guidance.

## Source Basis

This document is based on the current contents of:

- `moonblokz-crypto-lib/Cargo.toml`
- `moonblokz-crypto-lib/src/lib.rs`
- `moonblokz-crypto-lib/src/schnorr_malachite_signer.rs`
- `moonblokz-crypto-lib/src/schnorr_num_bigint_dig_signer.rs`
- `moonblokz-crypto-lib/src/schnorr_crypto_bigint_signer.rs`
- `moonblokz-crypto-lib/src/bls_bls12_381_bls_signer.rs`
- `moonblokz-crypto-lib/README.md`

Where earlier article-era summaries and the codebase diverge, this file treats the code as authoritative.

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
- **Backend** — one concrete compile-time-selected implementation of a family.

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

### Rule 1: exactly one implementation feature must be enabled

Compilation fails if more than one of these features is enabled:

- `schnorr-malachite`
- `schnorr-num-bigint-dig`
- `schnorr-crypto-bigint`
- `bls-bls12_381-bls`

### Rule 2: at least one implementation feature must be enabled

Compilation also fails if none of those concrete implementation features is enabled.

### Consequence

The public crypto API is always bound to exactly one backend at build time.

## Feature Model

The library uses a two-layer feature structure.

### Layer 1: algorithm family

- `schnorr`
- `bls`

### Layer 2: concrete implementation

- `schnorr-malachite`
- `schnorr-num-bigint-dig`
- `schnorr-crypto-bigint`
- `bls-bls12_381-bls`

### Default feature

The current default feature is:

- `schnorr-malachite`

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
- `aggregate_signatures(&self, signatures: &[&MultiSignature], message: &[u8]) -> Result<AggregatedSignature, CryptoError>`
- `verify_aggregated_signature(&self, message: &[u8], aggregated_signature: &AggregatedSignature, public_keys: &[&PublicKey]) -> bool`

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

When the `bls` family is active:

- `SIGNATURE_SIZE = 48`
- `MULTI_SIGNATURE_SIZE = 48`
- `PUBLIC_KEY_SIZE = 96`
- `PRIVATE_KEY_SIZE = 32`
- `AGGREGATED_SIGNATURE_CONSTANT_SIZE = 50`
- `AGGREGATED_SIGNATURE_VARIABLE_SIZE = 0`

### Global bounded-aggregation constants

Across all families:

- `MAX_AGGREGATED_SIGNATURES = 50`
- `MAX_AGGREGATED_SIGNATURE_BYTES = AGGREGATED_SIGNATURE_CONSTANT_SIZE + AGGREGATED_SIGNATURE_VARIABLE_SIZE * MAX_AGGREGATED_SIGNATURES`

This gives the library a compile-time upper bound for serialized aggregated-signature storage.

## Current Schnorr Algorithm Family Model

All three Schnorr backends implement the same effective public behavior with different arithmetic backends:

- Malachite-based big integers
- num-bigint-dig-based big integers
- crypto-bigint fixed-width arithmetic

### Shared curve model

All current Schnorr backends use the same effective secp256k1-style curve parameters:

- field modulus `p = FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F`
- order `n = FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141`
- generator point `G` with the standard secp256k1 base point coordinates

### Shared public-key representation

The public key is serialized as a `32`-byte `x` coordinate.

When reconstructing the curve point from bytes, the implementation:

1. interprets the bytes as `x`,
2. recomputes `y^2 = x^3 + 7 mod p`,
3. derives a square-root candidate,
4. rejects invalid `x` values,
5. chooses the even-`y` representative.

### Private-key validation rule

The private key must satisfy:

- `1 <= private_key < n`

Otherwise `Crypto::new` returns `InvalidPrivateKey`.

### Canonicalization during signer construction

After computing `public_key = private_key * G`, the current Schnorr implementations normalize the result so that the stored public key uses an even `y` coordinate.

If the initial public point has odd `y`, the implementation:

- reflects the public point to the even-`y` version,
- and replaces the stored private key with `n - private_key`.

This means signer initialization performs canonicalization, not mere storage.

## Schnorr Signing Model

The three Schnorr backends share the same effective signing flow.

### Deterministic nonce derivation

The signer derives a nonce candidate using a tagged SHA-256 hash over:

- tag `"nonce"`
- private key bytes
- message bytes
- little-endian counter bytes

The process repeats with incremented counter until the derived scalar is non-zero modulo `n`.

### Public nonce parity normalization

The implementation computes:

- `R = k0 * G`

If `R.y` is odd, the effective scalar becomes:

- `k = n - k0`

Otherwise:

- `k = k0`

### Challenge derivation

The challenge is derived with tagged SHA-256 over:

- tag `"challenge"`
- serialized `r` bytes
- serialized public key bytes
- message bytes

Then reduced modulo `n`.

### Signature equation

The signature is stored as `(r, s)` where:

- `r = R.x`
- `s = k + e * private_key mod n`

### `sign` versus `multi_sign`

In all current Schnorr backends:

- `sign(message)` and `multi_sign(message)` use the same underlying signing logic
- they differ only in the wrapper type returned

So the role distinction is semantic and protocol-oriented, not mathematical, for Schnorr.

## Schnorr Single-Signature Verification Model

The Schnorr backends verify signatures by:

1. checking that `r < p` and `s < n`,
2. recomputing the challenge from `r`, public key, and message,
3. computing `sG + (n - e)P`,
4. accepting if the resulting point’s `x` coordinate equals `r`.

This same logic is used for both:

- `verify_signature`
- `verify_multi_signature`

## Schnorr Aggregation Model

The current Schnorr aggregation implementation is more specific than a simple “combine signatures” description.

### Input validation

Aggregation fails with `InvalidSignature` if:

- the input signature list is empty,
- or the input count exceeds `MAX_AGGREGATED_SIGNATURES`.

### Serialized aggregate structure

A Schnorr `AggregatedSignature` stores:

- `count: usize`
- `r_bytes: [[u8; 32]; MAX_AGGREGATED_SIGNATURES]`
- combined `s`

The serialized form is:

1. `count` as `u16` little-endian
2. `s` as `32` bytes
3. one `32`-byte `r` value per contributing aggregation-ready signature

So the serialized length is:

- `34 + 32 * count`

### Deterministically weighted aggregation

The current Schnorr aggregate implementation does not simply sum all component `s` values.

It performs these steps:

1. sum all `r` values modulo `p` to get `r_sum`
2. derive an `initial_random_seed` with tagged hash `"aggregate"` over `r_sum`, `r_sum`, and `message`
3. for each signer index `i`, derive a per-signer deterministic weight with tagged hash `"rand"` over the seed and the serialized index
4. constrain each weight to the interval `1..n-1`
5. compute the aggregated `s` as the weighted modular sum of all component `s` values

This means the current Schnorr aggregate is a **deterministically weighted aggregate**, not just a naive additive bundle.

### Aggregate verification

To verify an aggregated Schnorr signature, the implementation:

1. checks that `count == public_keys.len()`
2. recomputes the same `r_sum` and aggregate seed
3. reconstructs each `R` point from its serialized `r` value
4. recomputes the same per-index deterministic weight
5. multiplies each reconstructed `R` point by its weight and sums them
6. recomputes each signer challenge `e`
7. multiplies each public key by `(n - e) * weight` and sums them
8. computes `sG`
9. accepts if the final point sum has the same `x` coordinate as the weighted sum of reconstructed `R` points

### Important consequence

The aggregate verification depends on:

- the exact order of signatures and public keys,
- the message,
- the per-signer `r` values,
- and the deterministic weight derivation.

So the current Schnorr aggregate artifact is not order-free in the conceptual sense. The code treats signer position as part of the aggregate definition.

## BLS Algorithm Family Model

The BLS backend is implemented as a wrapper around the `bls12_381-bls` library ecosystem.

### Public-key model

- `PublicKey` stores a `BLS_PublicKey`
- serialized size is `96` bytes

### Single-signature model

- `sign(message)` calls `SecretKey::sign(message)`
- result size is `48` bytes

### Aggregation-ready signature model

- `multi_sign(message)` calls `SecretKey::sign_multisig(&public_key, message)`
- result size is `48` bytes

### Verification behavior

- ordinary signature verification uses `public_key.verify(signature, message)`
- aggregation-ready signature verification first aggregates the single public key into a `MultisigPublicKey`, then verifies the `MultiSignature` against that aggregate public key

### BLS aggregate structure

A BLS `AggregatedSignature` stores:

- `count`
- one aggregated multisignature object
- serialized bytes of length `50`

Its serialized form is:

1. `count` as `u16` little-endian
2. `48` bytes of aggregated signature

So BLS aggregation is constant-sized with respect to signature body length.

### BLS aggregate creation

The BLS backend:

1. rejects empty inputs and inputs larger than `MAX_AGGREGATED_SIGNATURES`
2. clones the first aggregation-ready signature
3. aggregates the remaining aggregation-ready signatures into it
4. stores the final result with the count prefix

### BLS aggregate verification

To verify an aggregated BLS signature, the implementation:

1. checks that `count == public_keys.len()`
2. rejects counts above `MAX_AGGREGATED_SIGNATURES`
3. aggregates the provided public keys into a `MultisigPublicKey`
4. verifies the aggregated signature against that aggregated public key and the message

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

### Backend-specific regression coverage

The `schnorr-crypto-bigint` backend includes regression-vector tests for:

- exact signature bytes
- exact aggregate serialization layout
- aggregate count and `r` component ordering

This gives that backend stronger byte-level behavioral specification than the others currently have in the root test suite.

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

## Technical Writer View: How to Read This File

This document should be read as the answer to these questions:

- What is the exact current public crypto API?
- Which backends exist today?
- What constants and limits define serialized behavior?
- How does current Schnorr signing and aggregation work?
- How does the BLS wrapper differ structurally?
- What behaviors are explicitly enforced by tests?

## Related Documents

- [moonblokz-crypto-concept.md](./moonblokz-crypto-concept.md)
- [moonblokz-crypto-implementation.md](./moonblokz-crypto-implementation.md)
