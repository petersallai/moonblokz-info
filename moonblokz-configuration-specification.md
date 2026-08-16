# MoonBlokz Configuration Module Specification — Authoritative

Ratified 2026-08-14.

This document specifies the `moonblokz-configuration` crate and its companion `moonblokz-vm` crate: the wire format of chain-configuration content, the shared parameter registry, the resolution model, the execution model of the mini virtual machine, the commitment lifecycle, and the integration surfaces towards the blockchain, the radio subsystem, and the host-side tooling.

It is the **authoritative source** for those two crates, in the same sense as the blockchain and storage Architecture Decision Documents are for theirs: any other knowledge-base document that describes the chain-config wire format, a parameter identifier, the resolution model, the VM execution model, or the configuration commitment surface defers to this one, and divergence from it is evidence of a knowledge-base inconsistency. Where its ratification changed decisions recorded elsewhere, those documents were amended in the same change rather than left to diverge; this specification therefore carries no divergence register of its own.

Neither crate is scaffolded yet: the specification is written ahead of the implementation, which is planned as Epic 5 Stories 5.6–5.9 and 5.11 of the blockchain BMAD flow.

---

## 1. Purpose and scope

A MoonBlokz chain must run with parameters that are identical on every node. Some of those parameters are plain constants; others are more usefully expressed as a function of chain state — a registration price that grows with the size of the network, for example. Both kinds must resolve to the same value on every node that holds the same chain, and neither may be a local tuning knob, because a divergence in these values is a divergence in validation outcome.

The configuration module owns that resolution. It parses the configuration content carried by chain-config blocks (`payload_type=3`), holds the tentative and durable commitment state, resolves every parameter through a fixed registry of code-baked defaults, and evaluates the small programs that compute the argument-dependent parameters. The blockchain consumes the result through per-parameter accessors and never interprets a configuration key itself ([BC-FR56](./moonblokz-blockchain-prd.md)).

In scope:

- the chain-config payload envelope and the encoding of configuration content,
- the shared parameter registry across all consuming subsystems,
- the three-tier resolution model and the accessor surface,
- content acceptance and the structural bound checks,
- the virtual machine: machine model, instruction set, resource limits, failure model,
- the tentative/durable commitment lifecycle, its persistence, and its error branches,
- the change-notification seam and the radio-parameter integration,
- the host-side tooling: the bytecode assembler and disassembler in `moonblokz-vm`, and the configuration encoder in `moonblokz-configuration`.

Out of scope: governance of runtime configuration changes (no runtime change path exists — configuration is locked for the lifetime of the chain), and the post-MVP smart-contract system. `moonblokz-vm` is the intended substrate for that system, and §13 records the thinking that should be the starting point when it is picked up — but nothing in §13 is built now, and no requirement follows from it beyond keeping the VM a self-contained crate.

---

## 2. Crate layout and dependency rules

Two new crates, each in its own repository, linked by path dependency per the workspace convention:

| Crate | Contents | Depends on |
|---|---|---|
| `moonblokz-vm` | Bytecode execution: machine model, instruction set, fuel accounting, host seam. No MoonBlokz domain concepts. | nothing beyond `core` |
| `moonblokz-configuration` | `ChainConfigTrait`, the parameter registry, code-baked defaults, content decoding and acceptance, the FR8 tentative/durable state, the accessor surface. | `moonblokz-vm`, `moonblokz-chain-types`, `moonblokz-crypto` |

Both crates are `no_std` and no-alloc. Neither may pull in `embassy` or `alloc`: the dependency-graph gate (`cargo tree -e normal | grep -iE 'embassy|alloc'` empty) applies to both, and transitively protects `moonblokz-blockchain`, which depends on `moonblokz-configuration`. This is what keeps the blockchain core host-testable without an async runtime, and it is the reason the transport-shaped integrations of §10 are injected rather than imported.

Each repository also carries its host-side tools as separate `std` packages (§11), kept out of the library packages so that feature unification cannot pull `std` into a `no_std` library.

`moonblokz-vm` knows nothing about chain configuration. It receives a program, its arguments, a fuel budget, and a host handle; every chain-specific concept — what a key means, what the fuel limit is, which parameter is being resolved — stays in `moonblokz-configuration`. The crate is therefore independently testable, and the later smart-contract runtime can build on it without inheriting the configuration registry.

---

## 3. Wire format

### 3.1 Chain-config payload envelope

A `payload_type=3` block payload is framed as:

```
config_value_count : u16                        (2 B, little-endian)
config_values      : config_value_count × {
                         key_byte     : u8      (1 B, see §3.2)
                         value_length : u8      (1 B)
                         value        : value_length bytes
                     }
content_signature  : [u8; SIGNATURE_SIZE]       (node #0, over the content region)
```

The **content region** is the count field plus every entry — `payload[..content_end]`. `content_end` is derived by walking the entries, never read from a fixed offset; that is what makes truncation and trailing padding detectable. The signature width is `moonblokz_crypto::SIGNATURE_SIZE`, never a literal.

Multi-byte fields are little-endian throughout, matching every other multi-byte field in `moonblokz-chain-types`.

The content region is the canonical byte sequence signed by node #0 per FR7. It is invariant for the lifetime of the chain and is reproduced byte-identically in every FR49 replay chain-config block, by every node that creates one. Every rule in this specification that could alter the byte sequence for a given logical configuration — key allocation, value framing, entry ordering — is therefore permanent from the moment a chain exists.

Entries are stored back-to-back with no padding. Entry order is not semantically significant, but it is byte-significant: the signature covers the bytes as written, so a replay reproduces the original order.

### 3.2 Key byte encoding

The key byte carries both the parameter identity and the form of the value that follows:

| Key byte | Meaning |
|---|---|
| `0x00` | Invalid. |
| `0x01`–`0x7E` | Parameter ID 1–126, value is a **literal**. |
| `0x7F` | Reserved (never allocated — see below). |
| `0x81`–`0xFE` | Parameter ID 1–126 (`key_byte & 0x7F`), value is **bytecode**. |
| `0x80` | Invalid (parameter ID 0 with the bytecode flag). |
| `0xFF` | Escape: a multi-byte key follows. Reserved, not implemented. |

Bit 7 of the key byte is the value-form flag: clear for a literal, set for a bytecode program. Bits 0–6 are the parameter ID.

Parameter ID `127` is permanently reserved and never allocated, because its bytecode form (`0x7F | 0x80`) would be `0xFF` and collide with the multi-byte-key escape. The usable ID range is therefore **1–126**.

The `0xFF` escape is a recorded forward-extension path, not an implemented feature: a decoder that encounters it treats the content as malformed, exactly as it treats an unknown key. Recording it keeps the one-byte key layout maximally compact while preserving a defined, non-breaking growth direction if more than 126 parameters are ever needed.

### 3.3 Value forms

**Literal.** `value_length` must equal the parameter's declared width **exactly** — a `u64` parameter requires `value_length == 8`. This exact-width rule is what keeps the decoder total and bounded: no variable-length integer parsing, no ambiguity about zero-padding, and a width mismatch is a clean rejection rather than a silent reinterpretation. Multi-byte scalars are little-endian; array-typed parameters carry their bytes verbatim.

**Bytecode.** `value_length` is the program length in bytes and is not constrained by the parameter's width; the program's result is narrowed to the declared width per §5.3. Not every parameter admits a bytecode value — see the value-form column of the registry and the rules in §6.

Because `value_length` is a single byte, a program is at most **255 bytes** (§7.2.1). Widening `value_length` to two bytes above a designated key range is a **recorded forward-extension path, not an implemented one**: a decoder today reads exactly one length byte, and a chain that needed longer programs would be adopting a format extension, not a larger value. It is recorded alongside the multi-byte key escape of §3.2 for the same reason — to keep the compact one-byte framing while leaving a defined, non-breaking growth direction.

### 3.4 Malformed content

Content is malformed — and its chain-config block is rejected as exact evidence of invalidity per FR16 — when any of the following holds:

1. the payload is shorter than `2 + SIGNATURE_SIZE`;
2. a `value_length` runs past the end of the payload;
3. `content_end + SIGNATURE_SIZE != payload.len()` (trailing or missing bytes);
4. a parameter ID appears more than once, in either value form;
5. the key byte is `0x00`, `0x80`, or `0xFF`, or its parameter ID is `127`;
6. the parameter ID is not allocated in the registry of §4;
7. a literal's `value_length` does not equal the parameter's declared width;
8. a bytecode value appears under a parameter marked literal-only;
9. an acceptance check of §6 fails.

**Unknown keys are rejected, not skipped.** The consequence is deliberate and must be stated where operators can find it: the content of a chain's configuration defines the minimum firmware capability required to participate in that chain. A node whose firmware predates a key used by the chain will never converge — it rejects every chain-config block and stays in collecting state. This is preferable to the alternative, in which the older node silently substitutes its own default for the unknown parameter and validates against different values than the rest of the network — a consensus split that produces no error anywhere. To keep the failure diagnosable rather than mute, rejection on an unallocated key is logged distinguishably from other rejections (`chain-config-unknown-key`, carrying the offending key byte), so an operator sees *the node is out of date*, not *a bad block arrived*.

---

## 4. Parameter registry

The registry is a single flat key space shared by every consuming subsystem. It is **permanent wire format**: an ID's meaning cannot change once any chain exists, IDs are allocated densely and are never reused or renumbered, and a new parameter takes the next free ID.

**Next free ID: 30.**

Value-form column: `L` = literal only; `L/B` = literal or bytecode. Args column: the arity of the accessor, hence of any bytecode program for that key, and the count a `GETPARAM` naming that key must declare (§7.2.3).

### 4.1 Blockchain parameters

| ID | Parameter | Type | Width | Default | Form | Args | Notes |
|---:|---|---|---:|---|:--:|:--:|---|
| 1 | `inter_block_interval_ms` | u64 | 8 | 60_000 | L/B | 0 | FR45 (b) |
| 2 | `grace_period_window_ms` | u64 | 8 | 30_000 | L/B | 0 | FR47 |
| 3 | `block_size_limit` | u16 | 2 | 2016 | L/B | 0 | Bounded by `MAX_BLOCK_SIZE` |
| 4 | `max_block_utxo_output` | u8 | 1 | 255 | L | 0 | Bound-checked, §6 |
| 5 | `max_aggregated_signatures` | u8 | 1 | 50 | L | 0 | Bound-checked, §6 |
| 6 | `vote_scale` | u16 (non-zero) | 2 | 1000 | L | 0 | FR37; zero is invalid |
| 7 | `vote_interest` | u8 | 1 | 5 | L/B | 0 | FR37 |
| 8 | `parent_recovery_per_head_retry_interval_ms` | u64 | 8 | 120_000 | L/B | 0 | FR19 / FR46 |
| 9 | `parent_recovery_min_emit_interval_ms` | u64 | 8 | 10_000 | L/B | 0 | FR46 |
| 10 | `required_support` | u8 | 1 | 3 | L | 0 | Bound-checked, §6 |
| 20 | `block_fill_threshold_percent` | u8 | 1 | 80 | L/B | 0 | FR45 (a) |
| 21 | `active_chain_length` | u16 | 2 | 500 | L | 0 | `W`; bound-checked against the compile-time capacity, §6 |
| 22 | `mempool_replenishment_interval_ms` | u64 | 8 | 500_000 | L/B | 0 | FR56 |
| 23 | `custodian_fee` | u64 | 8 | 1 | L/B | 0 | FR51 carry-forward |
| 24 | `registration_price` | u64 | 8 | 100 | L/B | 1 | Arg: registered-node count |
| 25 | `tx_fee_per_byte_min` | u64 | 8 | 0 | L/B | 0 | FR56 fee policy |
| 26 | `tx_fee_per_byte_max` | u64 | 8 | 1000 | L/B | 0 | FR56 fee policy |
| 27 | `deviation_replay_insertion_delay_ms` | u64 | 8 | 300_000 | L/B | 0 | FR29 pacing |
| 28 | `replay_block_reward` | u64 | 8 | 100 | L/B | 0 | FR36 (c) |

### 4.2 Radio parameters

Every radio parameter is argument-less by rule (§10.2).

| ID | Parameter | Type | Width | Default | Unit | Form |
|---:|---|---|---:|---|---|:--:|
| 11 | `echo_request_minimal_interval` | u16 | 2 | 1440 | minutes | L/B |
| 12 | `echo_messages_target_interval` | u8 | 1 | 100 | seconds | L/B |
| 13 | `echo_gathering_timeout` | u8 | 1 | 10 | minutes | L/B |
| 14 | `delay_between_tx_packets` | u16 | 2 | 200 | milliseconds | L/B |
| 15 | `delay_between_tx_messages` | u8 | 1 | 20 | seconds | L/B |
| 16 | `relay_position_delay` | u8 | 1 | 10 | seconds | L/B |
| 17 | `scoring_matrix` | [u8; 5] | 5 | `[255, 243, 65, 82, 143]` | encoded | L |
| 18 | `retry_interval_for_missing_packets` | u8 | 1 | 60 | seconds | L/B |
| 19 | `tx_maximum_random_delay` | u16 | 2 | 200 | milliseconds | L/B |

Values are stored and returned in their **native unit** — the unit the parameter is already expressed in throughout the radio code. No normalisation to milliseconds: it would move every default and the radio API for no gain, since the VM works in `u64` internally regardless.

`registration_price` keeps arity 1 while its default is a plain literal: the accessor still takes the registered-node count, and the default simply does not vary with it. A chain that wants a size-dependent price overrides the key with a program of the same arity.

`scoring_matrix` is literal-only because it is not a scalar: the VM returns a `u64` and has no array-valued result form. Should a computed matrix ever be needed, it requires a distinct result form and a distinct opcode group, and is a deliberate future extension rather than an oversight.

### 4.3 VM parameters

| ID | Parameter | Type | Width | Default | Form | Args |
|---:|---|---|---:|---|:--:|:--:|
| 29 | `vm_fuel_limit` | u32 | 4 | 20_000 | L | 0 |

`vm_fuel_limit` is **literal-only** for a bootstrapping reason: it bounds every program evaluation, so resolving it must not itself require running a program.

It is chain configuration rather than a code constant because it is consensus-relevant. The fuel limit directly determines returned values — a node that exhausts its budget receives the fallback, a node that does not receives the computed value — so a node-local limit would let two nodes validate the same chain with different `required_support` or `W`. Putting it in the chain configuration guarantees every node on a chain uses the same budget, and keeps it tunable per chain without coupling it to a firmware version.

### 4.4 Excluded from the registry

Not chain configuration, and never allocated a key:

- **`initial_total_network_currency`** — fixed by block #0 at genesis from the `initiateGenesis(...)` argument (FR54, FR56).
- **node #0's public key** — the `node_zero_public_key` code-level trust anchor (FR69).
- **Compile-time / node-level capacities** — mempool capacity, block-tree capacity, storage capacity, and the verification-horizon size `H`. These follow from the local node's memory and durable storage, not from the chain (FR56).

---

## 5. Resolution model

### 5.1 Three tiers

Every parameter resolves through the same chain:

1. **Chain-config override** — the entry present in the configuration content, literal or bytecode.
2. **Code-baked default** — used when the parameter is absent from the content. May itself be a literal or a bytecode program with the same argument semantics.
3. **Code-baked fallback literal** — a plain constant, used when the tier above fails at evaluation time.

Each tier is attempted in order and each evaluation starts with a **fresh fuel budget**. This matters: if the default inherited an exhausted budget, then whenever fuel exhaustion was the failure cause, the default tier could never run and every such failure would drop straight to the fallback, making the middle tier dead code.

When a parameter's code-baked default is a literal, that literal is also its fallback; a distinct fallback value is only meaningful for a bytecode default. No default in §4 is bytecode today, so the fallback equals the default for every current parameter.

The three-tier chain is what makes the accessor surface total: the last tier is a constant, so **every accessor always returns a value**.

### 5.2 The active-configuration handle

Availability is decided once, at handle acquisition, not per accessor:

```rust
pub trait ChainConfigTrait {
    fn active_configuration(&self) -> Option<ActiveConfig<'_>>;
    // FR8 state operations — see §8
}

pub struct ActiveConfig<'a> { /* borrows the module */ }

impl ActiveConfig<'_> {
    pub fn commitment(&self) -> Commitment;          // Tentative | Durable
    pub fn inter_block_interval_ms(&self) -> u64;
    pub fn registration_price(&self, registered_nodes: u32) -> u64;
    // ... one accessor per registry entry, arity per the registry
}
```

`active_configuration()` returns `None` while no configuration is loaded, and a handle once one is — tentatively per FR8 during collecting state, or durably per FR54 at genesis and per FR8 at the ready transition. A caller that gets `None` cannot proceed with any configuration-derived work; the blockchain treats that as the signal that the corresponding staged-validation step (FR9) cannot yet be performed, and does not advance the affected block beyond the latest stage that needs no configuration-derived parameter.

The handle **borrows** the module. That makes FR56's no-caching rule structural rather than a documented request: the handle cannot outlive the invocation that acquired it, and it cannot be held across any mutation of the configuration state. Accessors are called at the moment a value is needed and their results are not retained.

The commitment state lives on the handle so that a value and the commitment state that produced it cannot be read from two different points in time.

Whether a value came from an override or from a default is **not** exposed. The accessor hides the distinction, per FR56. The only place the distinction surfaces is diagnostics — the change notification of §10.1 and the associated log record may report how many overrides the content carried — never the value-returning API.

### 5.3 Narrowing

The VM computes in `u64`. An accessor whose declared type is narrower returns the result narrowed by **saturation** to the declared type's maximum, consistent with the saturating arithmetic of the VM itself. Parameters carrying a structural invariant that must hold for correctness are not left to saturation: they are constrained at acceptance time instead (§6).

---

## 6. Content acceptance

Acceptance runs inside the configuration module, on the **raw declared values**, before any configuration is loaded. It is the only place where an out-of-range declared value is still visible: once a value has passed through a narrowing accessor it can no longer be distinguished from a legal one. Rejection is exact evidence of invalidity per FR16 and leaves no configuration loaded.

**Structural bound checks:**

1. `max_block_utxo_output ≤ UTXO_UNSPENT_BITS` — the compile-time per-block UTXO spent-bit width (FR8). A larger value cannot be represented by the local node's cache. `UTXO_UNSPENT_BITS` is a `pub const` of the configuration crate, pinned to the blockchain's real spent-bit width by a monomorphization-time assertion in the blockchain so the two cannot drift silently.
2. `required_support ≥ 1` (FR8) — otherwise ADR-015's `m = min(2·required_support − 1, |A|)` yields `m = -1`.
3. `required_support ≤ MAX_AGGREGATED_SIGNATURES` for the active crypto backend ([ADR-015](./blockchain-adrs/ADR-015-approval-subgroup-selection.md)).
4. `vote_scale ≠ 0` — it is the denominator of the FR37 anti-capture rule.
5. `block_size_limit ≤ MAX_BLOCK_SIZE` — the compile-time block buffer width.
6. `active_chain_length ≤ SNAKE_CHAIN_LENGTH_MAX` — the compile-time active-chain capacity.

The last of these is worth spelling out, because it settles what `W` is. The active-chain window length is **chain configuration**, not a build-time property of a node: every node on a chain must retain the same window, or they do not agree on what has dropped out of it. But a node cannot resize compile-time arrays from chain content, so the const generic becomes a **capacity** — `SNAKE_CHAIN_LENGTH_MAX` — and the chain-configured `W` must fit inside it. A node whose capacity is below the chain's `W` cannot participate and rejects the configuration; a node whose capacity exceeds it simply uses part of what it allocated. This is exactly the pattern `max_block_utxo_output ≤ UTXO_UNSPENT_BITS` already establishes: the chain states the requirement, the compile-time constant states what this build can honour, and the check at acceptance is where the two meet.

**Bound checks and value forms.** A structural bound can only be enforced on a value that is knowable at acceptance time. That gives a rule with three cases:

- Parameters whose bound must hold unconditionally and whose value could otherwise depend on runtime arguments are **literal-only** (IDs 4, 5, 6, 10, 21) — the declared value is checked directly.
- For any **argument-less** parameter carrying a bound, an override in bytecode form is permitted and is **evaluated once at acceptance**, under the ordinary fuel limit, with the bound applied to the result. An argument-less program's result cannot vary, so checking it once is sound. Any non-`Completed` outcome during acceptance evaluation — fuel exhaustion or any trap of §7.3 — **rejects the content** rather than falling back: a configuration that cannot be evaluated at commit time is a defective configuration, not a runtime condition. This evaluation is also what stands in for a bytecode verifier (§7.3), because it exercises the real decoder on the real program.
- Parameters that take arguments may not carry a structural bound, since no acceptance-time check can cover every argument value. The registry records the arity, and this rule constrains which parameters may ever be given one.

The acceptance-time evaluation is a validation step only. Its result is not retained: every accessor call re-resolves, per FR56.

---

## 7. The virtual machine

### 7.1 Machine model

A stack machine over `u64`:

- an **operand stack** of fixed maximum depth,
- a fixed array of **local slots**,
- a read-only **argument vector** supplied by the caller,
- a **program counter** over the bytecode,
- a **fuel counter**, decremented per executed instruction by that instruction's cost (§7.3),
- a **host handle** for host calls (§7.4).

There is no heap, no linear memory, and no host state beyond the handle. Arithmetic is unsigned and **saturating** on overflow and underflow; there are no signed operations. Division and modulo by zero yield `0` rather than trapping — a deliberate choice, because the divisor may be computed from a runtime argument and the accessor surface must remain total.

**Initial state is fully defined.** Local slots are zero-initialized before execution and no instruction may read machine state that was never written. An implementation-defined initial state would make results implementation-dependent, which on a consensus-bearing value is a correctness fault rather than a quality-of-implementation issue.

Instruction and immediate encoding is little-endian, matching the wire convention.

### 7.2 Bytecode format

#### 7.2.1 Program encoding

A program is a bare byte string — there is no header, no magic number, and no length prefix inside the program itself. Its length comes from the framing that carries it (`value_length` in §3.1), and its interpretation comes from the chain, which pins it (§7.6).

Instructions are variable-length: one **opcode byte**, followed by that opcode's immediate operand, if it has one. Immediates are little-endian, matching the wire convention everywhere else in the project. Instructions are decoded strictly sequentially from offset 0; there is no alignment requirement and no padding.

**Maximum program length is 255 bytes**, imposed by the one-byte `value_length` of the TLV framing rather than by the machine. That is a real design constraint on what a parameter program can express, and it is deliberate: a configuration payload competes for the same 2016-byte block as everything else, and programs at this scale stay auditable by eye. Widening `value_length` to two bytes above a designated key range is a recorded forward-extension path (§3.3), not an implemented one.

**Execution ends at `RET`.** Nothing requires `RET` to be the last byte of the program, and nothing scans for it ahead of time: a program whose control flow leaves the byte range simply traps (§7.3) and resolution falls to the next tier.

**Jump displacements are relative to the address of the following instruction** — that is, to the program counter *after* the jump opcode and its two immediate bytes have been consumed. A displacement of `0` therefore falls through, a negative displacement loops backwards. Displacements are two's-complement `i16`, little-endian. A destination outside the program traps. A destination *inside* the program is always meaningful, including one that lands in the middle of what the author intended as an instruction: decoding simply resumes at that offset, and the bytes there are decoded as instructions. There is no notion of an instruction boundary at runtime, so there is nothing to check — and because every node decodes the same bytes the same way, the result stays deterministic, which is the only property that matters here.

#### 7.2.2 Opcode space

Bytecode is permanent: once a chain exists, a program's bytes cannot be reinterpreted. The opcode space is therefore partitioned up front in aligned 16-byte blocks, with the room that current instructions do not need left explicitly reserved, so that later additions extend the encoding rather than filling in whatever bytes happened to be free.

| Range | Group | Status |
|---|---|---|
| `0x00` | — | **Permanently unallocated** |
| `0x01`–`0x0F` | Control and return | 4 allocated, 11 reserved |
| `0x10`–`0x1F` | Constants | 4 allocated, 12 reserved |
| `0x20`–`0x2F` | Stack | 3 allocated, 13 reserved |
| `0x30`–`0x3F` | Locals and arguments | 3 allocated, 13 reserved |
| `0x40`–`0x4F` | Arithmetic | 7 allocated, 9 reserved |
| `0x50`–`0x5F` | Bitwise | 6 allocated, 10 reserved |
| `0x60`–`0x6F` | Comparison | 6 allocated, 10 reserved |
| `0x70`–`0x7F` | Host calls | 1 allocated, 15 reserved |
| `0x80`–`0x8F` | Linear memory | Reserved, unallocated |
| `0x90`–`0x9F` | Persistent state | Reserved, unallocated |
| `0xA0`–`0xAF` | Byte-valued operands and results | Reserved, unallocated |
| `0xB0`–`0xBF` | Call frames | Reserved, unallocated |
| `0xC0`–`0xFF` | — | Reserved, unassigned |

`0x00` is permanently unallocated on purpose: zero is what a truncated, padded, or zero-initialized buffer is made of, and the most likely accidental "program" is a run of zero bytes. Keeping `0x00` undefined means such a buffer traps on its first instruction instead of executing something.

Every reserved opcode is undefined and traps at runtime (§7.3). The reservations record intent only — nothing behind them is designed or implemented, and §13 explains why they are worth writing down at this stage.

#### 7.2.3 Instruction reference

Stack effects are written left-to-right with the **top of stack on the right**: `[a b] → [c]` consumes `b` from the top and `a` beneath it, and leaves `c`. All arithmetic is unsigned `u64`; every instruction costs one fuel unit (§7.3).

**Control and return**

| Op | Mnemonic | Immediate | Stack | Semantics |
|---|---|---|---|---|
| `0x01` | `RET` | — | `[a] → []` | Ends execution; `a` is the program's result. |
| `0x02` | `JMP` | `rel: i16` | `[] → []` | Unconditional branch to `next_pc + rel`. |
| `0x03` | `JMPZ` | `rel: i16` | `[a] → []` | Branch to `next_pc + rel` if `a == 0`. |
| `0x04` | `JMPNZ` | `rel: i16` | `[a] → []` | Branch to `next_pc + rel` if `a != 0`. |

**Constants**

| Op | Mnemonic | Immediate | Stack | Semantics |
|---|---|---|---|---|
| `0x10` | `PUSH_U8` | `imm: u8` | `[] → [a]` | Pushes `imm` zero-extended to `u64`. |
| `0x11` | `PUSH_U16` | `imm: u16` | `[] → [a]` | Pushes `imm` zero-extended to `u64`. |
| `0x12` | `PUSH_U32` | `imm: u32` | `[] → [a]` | Pushes `imm` zero-extended to `u64`. |
| `0x13` | `PUSH_U64` | `imm: u64` | `[] → [a]` | Pushes `imm`. |

**Stack**

| Op | Mnemonic | Immediate | Stack | Semantics |
|---|---|---|---|---|
| `0x20` | `POP` | — | `[a] → []` | Discards the top. |
| `0x21` | `DUP` | — | `[a] → [a a]` | Duplicates the top. |
| `0x22` | `SWAP` | — | `[a b] → [b a]` | Exchanges the top two. |

**Locals and arguments**

| Op | Mnemonic | Immediate | Stack | Semantics |
|---|---|---|---|---|
| `0x30` | `LOAD` | `slot: u8` | `[] → [a]` | Pushes local slot `slot` (zero-initialized before execution). |
| `0x31` | `STORE` | `slot: u8` | `[a] → []` | Writes `a` to local slot `slot`. |
| `0x32` | `ARG` | `index: u8` | `[] → [a]` | Pushes argument `index` of the invocation. |

**Arithmetic** — all binary, `[a b] → [c]`

| Op | Mnemonic | Semantics |
|---|---|---|
| `0x40` | `ADD` | `a + b`, saturating at `u64::MAX`. |
| `0x41` | `SUB` | `a − b`, saturating at `0`. |
| `0x42` | `MUL` | `a × b`, saturating at `u64::MAX`. |
| `0x43` | `DIV` | `a / b`; **`0` when `b == 0`**. |
| `0x44` | `MOD` | `a mod b`; **`0` when `b == 0`**. |
| `0x45` | `MIN` | The smaller of `a` and `b`. |
| `0x46` | `MAX` | The larger of `a` and `b`. |

**Bitwise**

| Op | Mnemonic | Stack | Semantics |
|---|---|---|---|
| `0x50` | `AND` | `[a b] → [c]` | Bitwise `a & b`. |
| `0x51` | `OR` | `[a b] → [c]` | Bitwise `a \| b`. |
| `0x52` | `XOR` | `[a b] → [c]` | Bitwise `a ^ b`. |
| `0x53` | `NOT` | `[a] → [c]` | Bitwise complement of `a`. |
| `0x54` | `SHL` | `[a b] → [c]` | `a << b`; **`0` when `b ≥ 64`**. |
| `0x55` | `SHR` | `[a b] → [c]` | `a >> b`; **`0` when `b ≥ 64`**. |

**Comparison** — all binary, `[a b] → [c]`, pushing `1` for true and `0` for false

| Op | Mnemonic | Semantics |
|---|---|---|
| `0x60` | `EQ` | `a == b` |
| `0x61` | `NE` | `a != b` |
| `0x62` | `LT` | `a < b` |
| `0x63` | `LTE` | `a ≤ b` |
| `0x64` | `GT` | `a > b` |
| `0x65` | `GTE` | `a ≥ b` |

**Host calls**

| Op | Mnemonic | Immediate | Stack | Semantics |
|---|---|---|---|---|
| `0x70` | `GETPARAM` | `key_id: u8`, `argc: u8` | `[a₀ … a_{argc−1}] → [v]` | Resolves parameter `key_id` through the host (§7.4) and pushes its value. Consumes exactly `argc` operands from the stack, with argument `0` deepest and argument `argc−1` on top — the order in which they were pushed. |

`GETPARAM` is how one parameter is defined in terms of another. It is the only host-backed instruction defined today, and it is dispatched through the general host entry point of §7.4 rather than as a special case, so later host functions need no new machinery. Its nested evaluation draws from the same fuel budget as its caller (§7.3).

**The instruction is self-describing**: it carries the argument count rather than the VM obtaining it from the registry. The alternative would require the machine to ask the host how many operands to take before it could assemble the call, since the arity of a key is registry knowledge and the registry belongs to the configuration module — which would make the VM's coupling to MoonBlokz a two-way conversation rather than the single callback of §7.4. Encoding the count keeps the machine ignorant of what a parameter is, and it is the form that generalizes: a future call instruction (opcode group `0xB0`–`0xBF`) addresses a callee with no registry behind it at all.

The cost is that `argc` and the registry's `Args` column (§4) can disagree, so **the host validates the count**. It is the only party that can: it owns the registry. A mismatch is declined like any unresolved parameter, which is a runtime fallback rather than a live hazard — the `config-encoder` resolves each key through the registry when it assembles the content (§11), so the authoring path cannot emit a mismatch in the first place, and an argument-less program is additionally evaluated once at acceptance (§6).

Three definitional choices above are worth naming, because each removes a failure mode rather than merely picking a behaviour: division and modulo by zero yield `0`, shifts of 64 or more yield `0`, and subtraction saturates at `0`. Together with saturating addition and multiplication, this makes **every arithmetic instruction total** — no operand combination can trap. What remains able to fail is structural (fuel, stack depth, nesting), never arithmetic.

#### 7.2.4 Textual form

Bytecode has a readable form, and it is specified here rather than left to the assembler. Two reasons. The byte encoding is permanent, so its human-facing rendering should be too — a program written down in a review, a bug report, or a test fixture must mean the same thing in five years. And the round-trip through `vm-asm` and `vm-dis` is the conformance test for the instruction set (§11); a round-trip is only meaningful if both ends target a defined text, otherwise the two tools merely agree with each other.

**Lexical rules.** One instruction per line. Everything from a `;` to the end of the line is a comment. Blank lines are allowed. Mnemonics are the ones in §7.2.3 and are accepted case-insensitively. Leading and trailing whitespace is insignificant.

**Operands.** A numeric operand is decimal, or hexadecimal with a `0x` prefix. Jump displacements may be written as a signed decimal, but normally a label is used instead.

**Labels.** A label is `name:` — either on a line of its own or preceding an instruction on the same line. A jump takes either a label name or an explicit displacement; the assembler resolves a label to the displacement from the following instruction (§7.2.1). Label names are `[A-Za-z_][A-Za-z0-9_]*`.

**`PUSH` is an alias.** Written without a width, `PUSH n` assembles to the **narrowest** encoding that holds `n` — `PUSH_U8`, then `PUSH_U16`, `PUSH_U32`, `PUSH_U64`. The rule is deterministic, and it matters: programs live in a 255-byte budget, and an author who reaches for `PUSH_U64` out of habit spends five bytes where one would do. The explicit forms remain available and are what the disassembler emits.

**Canonical form.** `vm-dis` emits exactly one text for a given program: uppercase mnemonics, one instruction per line, a single space before an operand and a comma-and-space between the two operands of `GETPARAM`, decimal immediates, explicit `PUSH_*` widths, no comments, labels named `L0`, `L1`, … numbered by ascending target offset and placed on their own lines, LF line endings.

A jump whose destination does not begin a linearly decoded instruction is rendered as an explicit signed decimal displacement rather than a label: there is no line to attach a label to, and the displacement is what keeps the round trip exact.

Therefore `assemble(disassemble(bytes)) == bytes` for every program `vm-asm` accepts, and `disassemble(assemble(text))` is the canonical rendering of whatever the author wrote. The qualification matters in one case: a program whose jump destination leaves the byte range decodes and disassembles, but `vm-asm` refuses to reassemble it, because diagnosing exactly that is what §11 asks of the assembler. Its bytes remain inspectable through `vm-dis`.

**`GETPARAM` takes a number, not a name.** The assembler belongs to `moonblokz-vm`, which knows nothing about the parameter registry (§2), so the first operand is the numeric identifier and the second is the argument count. Both operands are written on one line, separated by a comma. Parameter names are a `config-encoder` convenience: in its input, `@inter_block_interval_ms` resolves through the registry to the identifier before the source reaches the assembler, and the encoder is also where an argument count that contradicts the registry is caught. The layering is visible in the syntax on purpose.

**Example — a derived parameter.** The grace-period window as half the inter-block interval, reading parameter 1 rather than restating its value:

```
GETPARAM 1, 0     ; inter_block_interval_ms, no arguments
PUSH 2
DIV
RET
```

Seven bytes: `70 01 00 10 02 43 01`. Written in the encoder's input the first line would be `GETPARAM @inter_block_interval_ms, 0`.

**Example — an argument-taking parameter.** The registration price growing with network size, clamped:

```
; registration_price(registered_nodes) = min(1000 + 5 * n, 50000)
ARG 0
PUSH 5
MUL
PUSH 1000
ADD
PUSH 50000
MIN
RET
```

Fourteen bytes, and worth reading against the encoding rules once:

```
32 00        ARG 0
10 05        PUSH_U8 5
42           MUL
11 E8 03     PUSH_U16 1000      ; little-endian
40           ADD
11 50 C3     PUSH_U16 50000
45           MIN
01           RET
```

**Example — a loop.** Compounding growth, one step per hundred registered nodes, which is the shape that needs backward jumps and therefore fuel rather than static analysis:

```
        PUSH 1000          ; price
        ARG 0
        PUSH 100
        DIV                ; [price tiers]
loop:   DUP
        JMPZ done
        PUSH 1
        SUB                ; tiers -= 1
        SWAP               ; [tiers price]
        DUP
        PUSH 10
        DIV
        ADD                ; price += price / 10
        SWAP               ; [price tiers]
        JMP loop
done:   POP
        RET
```

Twenty-seven bytes. At a thousand registered nodes the loop runs ten times and the whole evaluation costs 118 fuel units — four in the prologue, eleven per iteration, two to fall out, two to finish — a useful sense of scale for the budget in §12, and a reminder that the interesting limit here is the 255-byte program, not the arithmetic.

A closing observation on the literal-versus-bytecode choice: for a `u64` parameter holding a small number, a program is often *smaller* than the literal — `PUSH 60000 / RET` is four bytes against the literal's eight. Prefer the literal anyway. It is checkable by eye, needs no evaluation at acceptance, and cannot be misread; four bytes is not worth turning a constant into a program.

### 7.3 Limits and failure

Every way a program can fail is a runtime condition. There is **no load-time verification pass**; the end of this section explains why.

A program terminates without a result when:

- **fuel is exhausted** — the fuel counter reaches zero,
- **the operand stack overflows or underflows** — it exceeds its maximum depth, or an instruction consumes an absent operand,
- **the nesting depth is exceeded** — `GETPARAM` recursion passes the fixed maximum,
- **the opcode is undefined** — a reserved or unassigned byte is decoded (§7.2.2),
- **an instruction is truncated** — an immediate, or the opcode itself, extends past the end of the program; this is what a program running past its last instruction reaches,
- **control flow leaves the program** — a jump computes a destination outside the byte range,
- **an operand index is out of range** — an `ARG` index at or above the invocation's arity, or a `LOAD` / `STORE` slot outside the local-slot array,
- **a host call does not resolve** — `GETPARAM` names a parameter the host declines, either because it is unallocated or because the declared `argc` disagrees with the registry's arity for that key.

The program counter moves only by sequential advance or by a jump, so the two conditions above that concern leaving the program partition every way of doing so.

**The VM reports; it does not decide.** Execution returns a typed outcome and applies no policy of its own:

```rust
pub enum VmOutcome {
    Completed(u64),
    Trapped(TrapReason),   // every condition listed above except fuel exhaustion
    OutOfFuel,
}
```

The configuration module maps both non-`Completed` outcomes onto the next resolution tier (§5.1), so within this specification every condition above has the same effect: the tier fails, resolution moves on, and nothing is observable to the accessor's caller, whose surface remains total. The separation matters beyond this specification — a different caller may need a different policy for the same outcomes (§13.5) — and a VM that folded the policy into itself could not serve one.

**Fuel is charged per instruction from a cost table**, not one unit per instruction. Every instruction defined in §7.2 costs one unit today, which makes the table trivially uniform at present; `GETPARAM` additionally consumes whatever the nested evaluation spends from the shared budget. The table exists in this form from the start because it is consensus-critical and effectively permanent: a cost change changes which programs exhaust their budget, and therefore changes returned values. Introducing a table later — when an instruction whose real cost is not one unit first appears — would be a retroactive change to every chain's results.

**Fuel is one budget per accessor invocation, shared across nesting.** A `GETPARAM` sub-evaluation draws from the same budget as its caller, and exhaustion aborts the whole invocation rather than just the sub-evaluation. Per-sub-evaluation budgets would let a program compose arbitrarily many sub-evaluations, each individually under the limit, and evade the bound entirely. The budget is fresh per tier, not per nesting level (§5.1).

Cyclic references between parameters are not detected statically either. A cycle is caught at runtime by the nesting-depth limit and, failing that, by fuel — with the ordinary fallback outcome.

**Why there is no verifier.** A load-time pass could catch some of the conditions above — undefined opcodes, truncated immediates, out-of-range jump destinations and operand indices — but it could not catch the rest. Fuel exhaustion, stack depth and nesting depth are not decidable ahead of a run once the instruction set has backward jumps, and a jump into the middle of an intended instruction is not an error at all (§7.2.1). The runtime therefore has to be total regardless, and every condition needs defined behaviour there anyway. A verifier would not remove a single runtime check: it would duplicate a subset of them in a second code path, on a device where code size is a budgeted resource, in order to turn a fallback into a rejection for some inputs and not others. The runtime handling is the whole mechanism, and it is one mechanism.

What is *not* given up: everything such a pass would have caught is still caught before a chain commits to a configuration, by two cheaper means. Framing-level checks run at decode time — unallocated parameter identifier, duplicate key, literal width mismatch, bytecode under a literal-only key (§3.4) — because resolution cannot proceed without them. And every argument-less bytecode override is **evaluated once at acceptance** (§6), which exercises the real decoder on the real program: a malformed argument-less program is rejected there by being run, not by being inspected. Only an argument-taking program can carry breakage that no acceptance step reaches, and that one falls back at runtime, deterministically and identically on every node.

### 7.4 Host seam

The VM does not know what a parameter is. Host-backed instructions are dispatched through a single general entry point implemented by the configuration module:

```rust
pub trait VmHost {
    fn call(&self, func_id: u16, args: &[u64], fuel: &mut Fuel) -> Option<u64>;
}

pub const HOST_RESOLVE_PARAMETER: u16 = 0;   // args[0] = parameter id, args[1..] = its arguments
```

One host function is defined today. The entry point is general rather than parameter-specific so that later host capabilities are new `func_id` values rather than new trait methods — an added method is a breaking change for every implementor, an added identifier is not.

The remaining fuel is threaded through the call so that nested evaluation draws from the caller's budget. `None` propagates as a failed evaluation of the calling program.

`args.len() − 1` is the count the *program* declared, not the arity the registry records, so validating that the two agree is the host's responsibility and no one else's — it holds the registry, and the machine deliberately does not (§7.2.3). Declining is the whole of the remedy: it reaches the program as a failed host call and resolution falls to the next tier.

This is the whole of the VM's coupling to MoonBlokz: an identifier it does not interpret, and a callback it does not implement.

### 7.5 Deliberate non-features

Two capabilities are absent by decision, not by omission. Both are cheap to add later and impossible to remove later, which is why they are settled now.

**A program cannot observe its remaining fuel.** There is no instruction that reads the fuel counter. Exposing it would let a program branch on how much budget is left, which makes its result a function of the cost table — and the cost table is then frozen forever, because any repricing would silently change the behaviour of every program that inspects it. The EVM's `GAS` opcode is the cautionary case. Without introspection, the cost table remains something that could in principle be revised for a future chain; with it, it never can.

**A program cannot observe anything node-local or non-deterministic.** No wall-clock time, no randomness, no node-specific quantity — the local node's neighbour count, link qualities, queue depths, or storage occupancy. Every input is either an argument supplied by the caller or another configuration parameter. This is the same rule that makes radio parameters argument-less (§10.2), and for the same reason: a value that differs between nodes cannot be a value the network agrees on, and configuration exists precisely to be agreed on.

### 7.6 Instruction-set versioning

Configuration bytecode carries **no instruction-set version marker**, and this is a decision rather than an oversight. The configuration content is signed by node #0, is byte-invariant for the lifetime of the chain, and is reproduced byte-identically in every replay; the chain's configuration therefore pins its own interpretation, and an in-band version field would be a permanently stored constant that can never vary.

That reasoning is specific to configuration content and does not generalize. Any future artifact whose bytecode is authored independently of the chain's genesis — a deployed program, in the sense of §13 — must carry a version marker in its own deployment record, or the engine that executes it can never be revised. The absence of a version field here should not be read as a precedent for that case.

---

## 8. Lifecycle and persistence

### 8.1 Commitment states

The module holds at most one configuration content at a time, in one of three states: **absent** (no configuration; `active_configuration()` returns `None`), **tentative**, or **durable**.

- **Tentative load** — during collecting state, the first chain-config block whose FR7 content-signature verifies and whose content passes acceptance is loaded tentatively, so configuration-derived parameters become available for staged validation (FR8).
- **Durable promotion** — at the processing→ready transition, once the candidate active-chain segment's chain-config content has been verified byte-identical to the tentative content, the configuration is committed once and locked for the lifetime of the chain (FR8). At genesis the durable lock is established directly (FR54).
- **Discard** — the tentative content is dropped on the FR8 mismatch path, and the module returns to absent before adopting a new tentative.

A second durable promotion is refused, not silently re-applied.

**Single-buffer requirement.** Tentative and durable content are never two different values at once: promotion flips a flag over the same retained bytes. The module retains **one** `MAX_PAYLOAD_SIZE` buffer plus its state flag, which keeps its RAM footprint equal to the retention the blockchain performs today.

### 8.2 Who calls storage

The **blockchain** owns the storage handle and performs every storage call; the configuration module is a pure state machine over content bytes and has no storage dependency. This keeps the timing of the durable commit where the decision that triggers it lives — the FR8 ready transition — and avoids two owners of one `&mut` storage handle in a no-alloc, single-threaded design.

Concretely: the blockchain calls `StorageTrait::set_chain_configuration` at the durable commit, and `StorageTrait::load_control_data` at startup, passing the content to the configuration module in both directions.

### 8.3 Persistence of the tentative state

The tentative configuration is **RAM-only**. It is not written to a dedicated durable slot, for two reasons.

It does not need one: the tentative chain-config block is already retained in durable block storage as an ordinary block (FR8), and the FR59 restart rebuild walks stored blocks anyway, so the tentative state is re-derived on restart without a separate record.

And it would not fit comfortably: a control-plane replica must fit one 4096-byte flash page, and the record already occupies roughly 2161 bytes (version, key-size field, private key, node id, init-params-size field, init params, `MAX_BLOCK_SIZE` field, a reserved `MAX_BLOCK_SIZE` chain-config block area, CRC32). A second reserved block-sized area does not fit at all; even a payload-sized area (1894 B) would leave about 41 bytes of headroom while requiring a control-plane version bump and a migration of the safety-critical RP2040 flash format.

### 8.4 Startup

At startup the blockchain reads the control plane and, if a durable chain-configuration block is present, hands its content to the configuration module, which verifies and loads it as **durable**. During an FR59 rebuild that finds no durable configuration, a chain-config block encountered in stored blocks is offered as **tentative** through the ordinary path.

---

## 9. Error branches

Three failure paths need defined behaviour. The common principle across all three: **the module never destroys a durable configuration, and never enters an apparently working state on an incomplete or unverified one.**

**Durable commit returns a storage error.** The ready transition does not complete; the configuration stays tentative and the next processing pass retries. A partially completed commit is worse than a delayed ready state. If the flash fault is permanent the node retries and logs indefinitely — an operator-visible condition, not something the module should paper over.

Within this branch, an `AlreadySet` result is distinguished: storage reports a configuration while local state does not. If the stored content is **byte-identical** to ours, this is an idempotent success — a previous commit landed and the process was interrupted before the local lock was recorded. If it **differs**, it is a hard inconsistency and is handled like the third branch below.

**A stored durable configuration cannot be decoded by the current registry** — an unallocated key or a width mismatch, in practice a firmware downgrade or a chain using a newer key. The node does not enter ready state and returns a distinguishable initialization error. The stored data is **not** erased: the durable configuration is set-once and cannot be rewritten in any case, a firmware update can resolve the situation, and an erase could not be undone. This is the boot-time counterpart of the unknown-key rule of §3.4.

**A stored durable configuration fails its FR7 content-signature check** — typically because `node_zero_public_key` was initialized for a different chain. Same handling: refuse, report distinguishably, erase nothing. FR69's misconfiguration rule already states that recovery requires external operator action.

---

## 10. Consumers outside the blockchain

### 10.1 Change notification

The configuration module drives change notification itself, at the moment the change happens — not by a counter that a runtime polls, and not through the blockchain's outcome channel. Routing it through that channel would consume an emission slot, need a scheduler priority rule, and force the module to remember whether the notification had already been delivered; none of that is necessary.

The crate defines the seam and calls it on every state transition (tentative load, replacement on the mismatch path, durable promotion):

```rust
pub trait ConfigChangeSink {
    fn on_configuration_changed(&self, config: &ActiveConfig<'_>);
}
```

The sink is a **mandatory generic parameter** of the configuration module, so the no-op implementation used by tests, by the blockchain-only configuration, and by any consumer that does not care optimizes away entirely, and no runtime branch is paid per change.

The transport stays on the firmware side, where its dependencies already are: the node runtime implements the sink, builds the radio snapshot from the handle's accessors, and publishes it. Keeping `embassy-sync` out of the configuration crate is not cosmetic — it is what preserves the dependency gate of §2 for the blockchain.

### 10.2 Radio parameters

The radio subsystem consumes a `RadioConfiguration` snapshot rather than calling accessors. It runs on the other core, its real-time paths must stay non-blocking, and — decisively — it cannot supply arguments: the only chain-derived quantities live on the blockchain core, and the quantity the radio *does* know locally (its neighbour count) is node-specific and would produce a different value on every node, which is precisely what chain configuration exists to prevent.

**Radio parameters are argument-less by rule.** The registry records arity 0 for IDs 11–19 and this is a structural constraint on future allocations in that category, not an accident of the current set.

**Distribution.** The node runtime publishes the snapshot through an `embassy_sync::watch::Watch`, whose latest-value, multi-consumer semantics fit the case: several radio tasks hold their own copies of the values, a superseded configuration must never be delivered after a newer one, and no queue may grow. A `Channel` would allow config messages to accumulate behind traffic and deliver a stale snapshot after a fresh one. `embassy-sync` 0.7 — already the pinned version — provides `Watch`, so no version change is required.

**Update semantics.** New values apply to decisions taken **after** the update; deadlines already scheduled run to completion and are not recomputed. Pending echo-request and echo-gathering deadlines, wait-pool entries already queued with the previous relay-position delay, and the TX scheduler's next scheduled instant all stand. The alternative — recomputing pending deadlines — needs a defined answer at every site for a recomputed deadline that has already passed, and buys nothing at the frequency at which configuration actually changes.

That frequency is low by construction. Tentative→durable promotion is byte-identical by FR8 and therefore never changes a value; the runtime compares the newly built snapshot against the last published one and publishes only on difference. In practice a node publishes once, on the `default → tentative` transition, plus the rare FR8 mismatch path.

**Forward path.** Should a network-size-dependent pacing parameter ever be wanted, the transport already exists and only a recomputation trigger on the blockchain core would be added — recompute after each active-chain head change, compare, publish on difference. It is a defined extension, not a redesign. It is not built now: no current radio parameter has a meaningful chain-state input.

---

## 11. Tooling

Three tools, split by the same boundary as the crates: whatever operates on **bytecode** belongs to `moonblokz-vm`, and only what operates on **configuration content** belongs to `moonblokz-configuration`.

| Tool | Repository | Operates on |
|---|---|---|
| `vm-asm` — assembler | `moonblokz-vm` | Assembly source (§7.2.4) → bytecode |
| `vm-dis` — disassembler | `moonblokz-vm` | Bytecode → canonical source (§7.2.4) |
| `config-encoder` | `moonblokz-configuration` | Parameter overrides → signed configuration content |

Each is a `std` binary in a **separate tool package** within its repository, not a feature of the library. Separation at the package level rather than by feature flag is deliberate: Cargo unifies features per package, so a `std` tool target sharing a package with the `no_std` library could pull `std` into the library's own build and quietly break the dependency gate of §2.

**`vm-asm` — assembler.** Translates the textual form of §7.2.4 to bytecode and is the only place where structural mistakes in a program are diagnosed. Because the runtime carries no verifier (§7.3), an undefined opcode, a truncated immediate, a jump out of range, or an operand index above the declared arity merely trap and fall back on-device — correct behaviour, but a poor diagnostic. Static checking belongs here, where it costs no device code size and can produce a message that names the offending instruction. The assembler also enforces the 255-byte program limit of §7.2.1, which is otherwise only discovered when the framing refuses the value.

**`vm-dis` — disassembler.** Renders bytecode back to readable form: for tests, for reviewing a proposed genesis configuration before it is signed, and for diagnosing a chain whose configuration is known only as bytes. It is the counterpart the assembler is tested against — a round-trip through both is the conformance test for the ISA, and it is meaningful because §7.2.4 pins the canonical text both ends target.

Both belong to `moonblokz-vm` because they encode the instruction set, and the instruction set is what that crate owns. A disassembler in the configuration repository would mean two places that must agree on opcode meanings.

**`config-encoder`.** Produces the configuration content handed to `initiateGenesis(...)`: it takes a description of parameter overrides — literal values, and assembly source for computed parameters, where a parameter may be referenced by name as `@name` (§7.2.4) — resolves each name to its registry identifier, frames the override set per §3, and appends the node #0 content-signature. It applies the framing checks of §3.4 and the acceptance checks of §6, including the acceptance-time evaluation of argument-less bytecode, so that a configuration the tool accepts is one the network accepts.

For the bytecode entries it **calls `vm-asm` as a library**: `moonblokz-configuration` already depends on `moonblokz-vm`, so the `config-encoder` package depends on the `moonblokz-vm-asm` package directly, which exposes the assembler as a library beside its binary, rather than reimplementing an assembler or forcing the author to run a two-step pipeline by hand. The dependency direction matches the crate layering — configuration knows about bytecode, the VM knows nothing about configuration.

The specification fixes each tool's input and output formats and the checks it must apply; the implementations are separate work.

---

## 12. Constants and budgets

Named here, valued after measurement — this specification does not invent numbers it cannot ground:

| Constant | Meaning | Value |
|---|---|---|
| `MAX_PAYLOAD_SIZE` | Retained content buffer | 1894 B (existing) |
| `UTXO_UNSPENT_BITS` | Per-block spent-bit width, pinned to the blockchain | 256 (existing) |
| `SNAKE_CHAIN_LENGTH_MAX` | Compile-time active-chain capacity; bounds the chain-configured `W` | 500 (existing default) |
| `SIGNATURE_SIZE` | Content-signature width | from `moonblokz-crypto` |
| Maximum program length | Bytecode value size, bounded by the framing | 255 B (`value_length: u8`) |
| VM operand stack depth | Fixed maximum | 16 |
| VM local slot count | Fixed array size | 8 |
| `GETPARAM` nesting depth | Fixed maximum | 3 |
| `vm_fuel_limit` default | Code-baked default of ID 29 | 20_000 |

The module's own RAM footprint is one `MAX_PAYLOAD_SIZE` buffer plus its state flag — unchanged from the retention the blockchain performs today — plus the VM's fixed stack and slot array, which are the only new allocations and are sized by the four values above.

The VM's operand stack and slot array are stack-resident rather than owned, so they cost a frame rather than a static footprint. Measured on `thumbv6m-none-eabi` at the firmware's release profile, one interpreter frame is `8 × (stack depth + slot count) + 112` bytes — 304 B at the values above. The nesting depth costs nothing itself, but it multiplies the frame count, because a nested evaluation re-enters the interpreter through the host: the worst case is four frames, 1216 B, plus the host's own frame at each level. This has to fit beside the FR45 block-creation peak within the blockchain task's stack, since parameter resolution runs at that peak.

`vm_fuel_limit` bounds evaluation *time*, not program size — a program with a backward jump has no size-derived bound — so it is set against how long a runaway evaluation may hold the core. A static estimate of 55–70 cycles per instruction puts 20_000 units near 9 ms at 133 MHz, against the 118 units the most expensive worked example of §7.2.4 spends. The timing half of that estimate has not been confirmed on hardware.

---

## 13. Future direction: smart contracts

**Nothing in this section is in scope.** No part of it is designed, specified, or built by this document, and no implementation obligation follows from it. It is recorded for one reason: `moonblokz-vm` is the intended substrate of a later smart-contract capability, and a handful of decisions taken now either keep that path open or quietly close it. When the capability is picked up, this is where the thinking starts — not from a blank page, and not from a general-purpose contract platform's assumptions, which do not survive contact with this environment.

### 13.1 The constraint envelope

Four numbers decide the shape of anything that could run here. Contract code must fit **one block** (`MAX_BLOCK_SIZE`, 2016 B, less header and signature). The chain's whole throughput is on the order of **tens of bytes per second**. There is **no global time** — only block sequence. And persistent state is subject to `snake_chain` bounded retention, so it is either **small and replayed, or gone**.

The consequence is that a contract interaction is a **rare, high-value event** — "the work was accepted", "the shipment arrived", "the lease expired" — never a per-second micro-interaction. A design that assumes otherwise is designing for a different network.

One environmental property works in the opposite direction, and it is decisive: participants are **registered, identified nodes**, not anonymous keys. That is what dissolves the oracle problem, which is normally the hardest part of making contracts useful. A fact about the physical world enters the chain as a **signature by a known node** — the site supervisor, the inspector, the weighbridge, the gate controller. Essentially every viable contract family below runs on that one pattern.

### 13.2 Contract families that fit

Grounded in the use cases the project targets (industrial worksites, agriculture, logistics and ports, community backup payments, disaster response, off-grid communities, machine-to-machine coordination, research expeditions, humanitarian relief):

- **Conditional payment / escrow** — funds locked, released when M-of-N designated nodes attest completion, refunded past a deadline sequence. State is a few tens of bytes; the aggregated-signature primitive already exists for exactly this shape of evidence.
- **Custody handoff** — a shipment moves node to node, each transfer a signed step; payment releases only when the step sequence is complete and in order. State: one position index and the expected next party.
- **Service metering and tariffs (M2M)** — a pump, scale, charger, or gate registers an event; the contract prices it from a tariff function and settles. The tariff is precisely the kind of function the configuration VM already evaluates.
- **Access rights** — who may open which gate or start which machine, in which shift. A small ACL, granted and revoked by signed instruction, read by the device from the chain.
- **Shared-resource booking** — crane, irrigation quota, charging bay, weighing slot. Deterministic ordering by `(sequence, node_id)`, the tie-break discipline the chain already uses; no clock required.
- **Work attestation and payroll** — check-in/out attested by a supervisor node, accrual per worker, payout on a sequence interval. A few bytes per worker — the same shape the chain already replays for balances.
- **Bond and slashing** — a deposit forfeited on attested misconduct. Workable precisely because participants are identified.
- **Revenue splitting** — an incoming amount divided among parties in fixed shares. Nearly stateless, and therefore the cheapest family of all; it would still work even if contracts were restricted to pure functions.
- **Local issuance and rationing** — ration, fuel, or water credits issued by an authorized node and spent by holders. Per-holder balances, structurally identical to what the chain already carries.
- **Mutual risk pool** — small communal fund paying out on attested events; escrow inverted, with many contributors.

### 13.3 What does not fit

Anything needing **external data** (price feeds, weather, an internet API): there is no oracle and determinism forbids one. Anything keyed to **wall-clock time**: only sequence exists. Anything with **large or growing state** — AMMs, order books, metadata-bearing tokens, long-history logic — because bounded retention removes the history underneath it. Anything iterating a **large collection**: fuel and a few hundred bytes of memory do not permit it. And anything **interaction-frequent**: at a 60-second inter-block interval over LoRa, a contract call is an occasional event.

These are consequences of the hyper-local, infrastructure-independent positioning, not temporary gaps to be closed later.

### 13.4 State under `snake_chain`: the rent model

This is the hard problem, and MoonBlokz already contains its solution in miniature.

Persistent state on this chain is not a one-off cost but a **recurring** one: essential state must be replayed into fresh head blocks before it falls out of the window (FR49 for chain configuration, FR50 for balances, FR51 for unspent UTXOs). Contract state would inherit that obligation — permanently, for every node, at every node's expense. That does not scale on a 264 KB device.

**The precedent.** A UTXO already pays for its own persistence. FR51 reduces every carried-forward UTXO by the configured custodian fee and **discards any whose amount would fall below that fee**; the PRD calls this self-clearing — each preservation step strictly reduces the residue, so carry-forward-only value disappears instead of accumulating forever. A contract-rent rule is therefore not a new principle but the generalization of an existing one:

| | UTXO (FR51) | Contract (candidate) |
|---|---|---|
| What it pays for | Carrying across the window | Carrying across the window **and** continuous slot occupancy |
| Payment source | The carried value itself | A separately fundable rent balance |
| Termination condition | Amount falls below the fee | Rent balance is exhausted |
| How it terminates | **Omitted** from the carry-forward | **Omitted** from the replay |

**Death by omission** is the property that makes this cheap: no deletion transaction, no cleanup block, no consensus event. When state would be replayed, the creator checks whether it is still funded and simply does not carry it. Garbage collection costs zero chain traffic.

**Lazy accounting.** Rent must never be a per-block transaction — that alone would swamp the chain. Store only `(last_topup_sequence, balance_at_topup, rate)` and derive the rest:

```
remaining     = balance_at_topup − (current_sequence − last_topup_sequence) × rate_per_block
death_sequence = last_topup_sequence + balance_at_topup / rate_per_block
```

Deterministic on every node, and the termination point is known **in advance**, which is what lets a replaying creator decide without consulting anything beyond the contract record.

**Where the money goes — follow the existing idiom.** The custodian fee is credited to no one: FR36 (b) states that a zero-input carry-forward contributes zero fee "because its output value already reflects custodian-fee reduction". The fee is absorbed by the network, exactly as `registration_price` is, while the creator doing the replay work is compensated by the separately issued `replay_block_reward` (FR36 (c)). The chain's idiom is therefore **the state holder burns, the worker is minted**, and contract rent should follow it rather than introduce a second, contradictory flow.

**Charge basis.** FR51 charges per replay event — roughly once per window traversal. A contract is better charged **per block**, because a UTXO is only bytes inside a block, while a contract additionally occupies a slot in every node's contract table, continuously and independently of where the window happens to be. The two bases are convertible in expectation (`fee_per_replay ≈ rate_per_block × W`) but they price different resources, and the contract genuinely consumes the one the per-block basis prices.

**The asymmetry that forbids copying FR51 outright.** When a UTXO dies, only its own residual value is lost. When a contract dies it may take **someone else's money** with it — an escrow's locked deposit. Hence: the rent balance must be **separate** from value held in custody; a beneficiary designated at creation receives held value on termination; and a contract whose semantics depend on surviving to a deadline (a deadline-bearing escrow) should be required to pre-fund rent to that sequence, which is checkable at creation.

**Open questions this leaves.** Whether top-up is permissionless (recommended — it defuses the obvious griefing move, a counterparty letting an escrow lapse); a minimum creation deposit, to stop slot-squatting with a few blocks of rent; a hard `MAX_CONTRACTS` capacity, since rent bounds economic demand but not instantaneous count, plus a deterministic rule for a full table (rejection is safer than eviction, which is gameable); and the explicit rule that **there is no resurrection** — once state leaves the window, bounded retention makes it unrecoverable.

**A wider observation.** Contract state would be the first state class that pays for its own permanence in full. The open-gaps register notes that bounded UTXO retention has no saturation handling in the MVP; the same mechanism is a candidate answer there later. Not a reason to touch it now — a reason to know the pattern generalizes.

### 13.5 What this asks of the VM today

Most of what a contract system needs is **additive** and can be built later without disturbing anything specified here: linear memory, persistent state behind host functions, byte-valued results, call frames, execution context, cryptographic host functions, and an effect-list model in which the VM returns proposed effects for the blockchain to validate and apply rather than mutating anything itself.

A small number of decisions are **not** additive, because bytecode and its cost model are permanent once a chain exists: how fuel is priced per instruction, what the VM returns when execution does not complete, how the opcode space is partitioned, how the host interface is shaped, what the machine deliberately does not expose, and how the initial machine state is defined. Those are the only smart-contract-driven items worth settling while the VM is being specified at all, and they are settled — in §7.1 through §7.6, as part of the current scope. Each is a shape decision with no functional cost today: the cost table is uniform, one host function is defined, the reserved opcode ranges are empty, and the two non-features are prohibitions rather than mechanisms. Nothing else from this section is anticipated in the design.

---

## Related Documents

- [MoonBlokz Blockchain Product Requirements Document](./moonblokz-blockchain-prd.md) — FR56 assigns parameter ownership to this module, FR7 / FR8 / FR17 / FR49 define the commitment and replay obligations it serves, and FR54 / FR69 fix the genesis and trust-anchor boundaries it must not cross.
- [MoonBlokz Blockchain Architecture Decision Document](./moonblokz-blockchain-architecture.md) — §2 places the two crates in the crate-split, §5 carries the const-generic catalog including the `SNAKE_CHAIN_LENGTH_MAX` capacity, and §11 records the `ChainConfigTrait` surface the blockchain sees.
- [MoonBlokz Blockchain Algorithm Model](./moonblokz-blockchain-algorythm.md) — §8 states the chain-config payload envelope and key-byte encoding in the blockchain's own terms and points here for the parameter registry.
- [MoonBlokz Radio Implementation Notes](./moonblokz-radio-implementation.md) — the `RadioConfiguration` fields that become registry identifiers 11–19, and the snapshot distribution defined in §10.2 here.
- [MoonBlokz System Constraints & Limits Reference](./moonblokz-system-constraints.md) — the RAM, flash and capacity budgets this module's footprint and the VM's fixed structures must fit inside.
- [MoonBlokz Glossary](./moonblokz-glossary.md) — disambiguates *chain configuration* and *active configuration*, and the active-chain window `W` versus its compile-time capacity.
