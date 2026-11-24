---
title: "Lesson 56 — Substrate/Polkadot concepts and Rust clients"
---

# Lesson 56 — Substrate/Polkadot concepts and Rust clients

**Module 18 · Other blockchain ecosystems (first lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `blake2 0.11.0`, `ed25519-dalek 3.0.0`, `hex 0.4.3`, `serde_json 1.0.154`, `twox-hash 2.1.5`, the Lesson 53 crate `solanalab` as a path dependency (only for its base-58 code), and, for tests only, `parity-scale-codec 3.7.5` (59 packages in `Cargo.lock`, the crate itself included)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** Substrate is a framework for building blockchains in Rust; Polkadot is a network built with it, in which many chains ("parachains") share the security of one coordinating chain. This lesson teaches the parts that a Rust *client* programmer meets first, and builds each one small enough to read: the **SCALE** codec (how every value on such a chain is serialized), the **hashing and storage-key layout** (how a value is found), **SS58** (the text form of an account), the **extrinsic** (a signed transaction, and exactly what the signature binds), a tiny **runtime** with a block builder (nonces, fees, existential deposit, failed calls, events) and a **JSON-RPC front end** shaped like a node's. The SCALE codec is cross-checked against the real `parity-scale-codec` crate; the keys, addresses and signatures were also computed by separate Python programs. What I did **not** do: install the Substrate or Polkadot toolchain, run a node, connect to any network, or use `subxt`, `sp-core`, `polkadot-js` or any real client library. Everything about the real stack's consensus (BABE, GRANDPA), parachains, cross-chain messaging (XCM), FRAME pallets and metadata is **from memory** and is labeled so; the chain here is a teaching model, not the real runtime (its "state root" is a plain hash, not Substrate's Merkle trie). **Not audited; nothing here secures anything.** Not covered: governance, staking, runtime upgrades in WebAssembly, writing real pallets, light clients, and the sr25519 and ecdsa signature schemes (the model signs with ed25519, which the real format also supports).

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation sites were not. The twox-hash storage prefixes, the SS58 addresses, the extrinsic wire bytes and the ed25519 key and signature were computed by **separate Python programs** (✓: a pure-Python xxHash64, `hashlib.blake2b`, and the `cryptography` package), not by the code under test; two of the storage prefixes and Alice's address are also values I remembered from real chains, and the Python programs reproduced them.

Builds on [Lesson 50](50_networking_determinism.md) (determinism), [Lesson 51](51_rpc_nodes_indexers.md) (JSON-RPC, nodes, indexers), [Lesson 52](52_web3_reliability.md) (finality, retries, idempotency), [Lesson 49](49_merkle_transactions_consensus.md) (Merkle roots, transactions, consensus), [Lesson 48](48_crypto_primitives.md) (hashes and signatures), [Lesson 53](53_solana_model.md) (the Solana model, a useful contrast, and its base-58 code), [Lesson 39](39_testing.md), [Lesson 40](40_property_testing_fuzzing.md), [Lesson 41](41_benchmarking_profiling.md) and [Lesson 26](26_error_design.md). The next lesson, Lesson 57, appears here only as plain text.

---

## 1. Why this lesson exists

A Solana client (Lesson 53) talks to programs that read raw bytes. A Substrate client talks to a node that publishes a self-description (the *metadata*), and everything on the wire is SCALE. The skill that carries over from either is the same: **know exactly how bytes become values, and refuse everything else.** Each claim here has an experiment:

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| SCALE integers are little-endian and "compact" integers have four modes | known answers from the specification for 0, 1, 42, 63, 64, 69, 16383, 16384, 65535, 2³⁰−1, 2³⁰, 2³², `u128::MAX`; 3,000 random values agree with `parity-scale-codec` (§3) |
| A decoder must accept one encoding per value | six non-canonical compact forms (a value in a wider mode, a big integer with a trailing zero byte) are refused; so is a 17-byte big integer (§3.3) |
| A length prefix is attacker input | a vector that claims 2³⁰ elements and supplies none is refused before anything is allocated (§3.4) |
| Our codec and the real one agree on nested values | 500 random `Vec<(u32, Option<Vec<u8>>)>` encode to identical bytes and decode back in both directions; 4,000 random garbage inputs never panic (§3.5) |
| Storage keys are computed, not stored | `twox128("System") ++ twox128("Account")` and two other prefixes equal values computed in Python, and the map key for an account is `prefix ++ blake2_128(id) ++ id`, from which the id can be recovered (§4) |
| An address carries a checksum and a network | Alice's key gives `5Grwva…` at prefix 42; eight addresses match Python; **all 2,736 single-character edits of one address are rejected** (§5) |
| A signature binds more than the call | changing genesis hash, spec version, tx version, call, nonce, tip or signer each invalidates it; the wire bytes of a transfer equal a Python build byte for byte (§6) |
| Flipping any bit of an extrinsic either fails to parse or yields a different, invalid one | all 1,128 bits of a 141-byte extrinsic (§6.4) |
| A failed call is still included and still pays | an overdrawn transfer leaves the nonce up by one and the balance down by the fee; supply equals balances in every one of 40 random rounds (§7) |
| Reaping an account resets its nonce | a signed transfer becomes valid again after its sender is reaped and refunded (§7.4) |
| A JSON-RPC front end has a small, testable contract | six methods, five error codes pinned to their numbers, every rejection reason named (§8) |
| Tests can have a bug of their own | the first SS58 decoder passed small prefixes and failed every one from 64 up, found by a round-trip test (§5.3) |

Total in the practical challenge: one library crate, 6 modules, **80 tests**, a benchmark example and **132 mutants** run against the test suite.

---

## 2. The stack at a glance (from memory)

Nothing in this section was run; it is the map, and §3 onward is the territory.

- **Substrate** is a Rust framework for writing a blockchain's node and *runtime*. The runtime is the state-transition function (the equivalent of a Solana program set plus the bank, Lesson 53), compiled both natively and to WebAssembly; the chain stores the WebAssembly in its own state, which is how a chain upgrades its rules without a new node binary.
- **FRAME** is the set of libraries for building a runtime from **pallets** (modules such as balances, system, timestamp). Each pallet has storage items, calls (dispatchable functions), events and errors. A storage item's address is derived from the pallet and item names (§4); a call's address on the wire is a pair of indexes (pallet, call), which is what §6 encodes.
- **Polkadot** is a *relay chain* that provides shared security and coordination; **parachains** are chains that attach to it and have their blocks validated by the relay chain's validators. **XCM** is the message format parachains use to talk to each other. Consensus pieces I remember by name only: **BABE** (block production) and **GRANDPA** (finality, so that "best" and "finalized" heads differ, which a client must handle; Lesson 52).
- A node publishes **metadata**: a SCALE-encoded description of every pallet, call, storage item and type. A generic client reads it to know how to encode a call and decode a storage value. In Rust, I remember `subxt` as the client library that generates typed code from metadata, and `sp-core`, `sp-runtime` and `parity-scale-codec` as the building-block crates; in JavaScript, `polkadot-js`. I did not use any of them except `parity-scale-codec` as a test oracle.
- The legacy JSON-RPC methods (`state_getStorage`, `author_submitExtrinsic`, `chain_getHeader`, `system_accountNextIndex`) are the ones §8 imitates; I remember a newer family (`chainHead_v1_*`, `transaction_v1_*`) designed so that a client can follow the chain without the node holding unbounded state for it. Check current documentation for exact names and behavior.

---

## 3. SCALE ✓

SCALE ("Simple Concatenated Aggregate Little-Endian") is a compact, non-self-describing binary encoding: no field names, no type tags, so a decoder must already know the type. That is the opposite of JSON and the same trade as Borsh on Solana (Lesson 53).

### 3.1 The rules the crate implements

- Fixed-width integers are **little-endian** (`u16` `0x0102` is `02 01`).
- `bool` is one byte, `0` or `1`; anything else is an error.
- `Option<T>` is `00` for none, `01` followed by the value for some.
- `Vec<T>` and `String` are a **compact length** followed by the items or the UTF-8 bytes.
- Arrays have no length (the type says how many); tuples are their fields in order.
- A **compact integer** has four modes, chosen by the two low bits of the first byte.

| Mode (low bits) | Range | Bytes | Layout |
|---|---|---|---|
| `00` | 0 – 63 | 1 | value in the upper 6 bits |
| `01` | 64 – 16,383 | 2 | little-endian `(value << 2) \| 01` |
| `10` | 16,384 – 2³⁰−1 | 4 | little-endian `(value << 2) \| 10` |
| `11` | 2³⁰ – 2¹²⁸−1 | 5 – 17 | first byte `((n − 4) << 2) \| 11`, then `n` little-endian bytes, `n` the fewest that hold the value |

✓ Known answers (from the SCALE specification as I remember it; every one of them also matches `parity-scale-codec` and my reading of the layout above): 0 → `00`, 1 → `04`, 42 → `a8`, 63 → `fc`, 64 → `0101`, 69 → `1501`, 16383 → `fdff`, 16384 → `02000100`, 65535 → `feff0300`, 2³⁰−1 → `feffffff`, 2³⁰ → `0300000040`, 2³² → `070000000001`, `u128::MAX` → `33` followed by sixteen `ff`.

### 3.2 The compact integer in code

```rust
impl Encode for Compact {
    fn encode_to(&self, out: &mut Vec<u8>) {
        let v = self.0;
        if v < 1 << 6 {
            out.push((v as u8) << 2);
        } else if v < 1 << 14 {
            out.extend_from_slice(&(((v as u16) << 2) | 0b01).to_le_bytes());
        } else if v < 1 << 30 {
            out.extend_from_slice(&(((v as u32) << 2) | 0b10).to_le_bytes());
        } else {
            let bytes = v.to_le_bytes();
            let n = 16 - (v.leading_zeros() as usize / 8);
            out.push((((n - 4) as u8) << 2) | 0b11);
            out.extend_from_slice(&bytes[..n]);
        }
    }
}
```

```rust
impl Decode for Compact {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        let first = take(input, 1)?[0];
        match first & 0b11 {
            0b00 => Ok(Compact(u128::from(first >> 2))),
            0b01 => {
                let rest = take(input, 1)?[0];
                let v = u16::from_le_bytes([first, rest]) >> 2;
                if v < 1 << 6 {
                    return Err(DecodeError::NonCanonical);
                }
                Ok(Compact(u128::from(v)))
            }
            0b10 => {
                let rest = take(input, 3)?;
                let v = u32::from_le_bytes([first, rest[0], rest[1], rest[2]]) >> 2;
                if v < 1 << 14 {
                    return Err(DecodeError::NonCanonical);
                }
                Ok(Compact(u128::from(v)))
            }
            _ => {
                let n = usize::from(first >> 2) + 4;
                if n > 16 {
                    return Err(DecodeError::Overflow);
                }
                let b = take(input, n)?;
                if b[n - 1] == 0 {
                    return Err(DecodeError::NonCanonical);
                }
                let mut buf = [0u8; 16];
                buf[..n].copy_from_slice(b);
                let v = u128::from_le_bytes(buf);
                if v < 1 << 30 {
                    return Err(DecodeError::NonCanonical);
                }
                Ok(Compact(v))
            }
        }
    }
}
```

### 3.3 Canonical decoding ✓

Look at the decoder: it rejects a value that fits in a narrower mode than the one used (`NonCanonical`), a big integer whose last byte is zero (it could have used fewer bytes), a big integer below 2³⁰, and one longer than 16 bytes (`Overflow`). The reason is not pedantry. If two byte strings decode to the same value, then *hashing the bytes* (an extrinsic hash, §6) and *comparing values* disagree, and a signature over one encoding does not cover the other. A malleable encoding is how the same transaction acquires two ids. ✓ `compact_rejects_non_canonical_forms` feeds `0100`, `05000000`, `070100000000`, `0b010000000000`, `0f01000000000000` and `070000004000` (all non-canonical) and requires `NonCanonical`.

### 3.4 Hostile lengths ✓

```rust
impl<T: Decode> Decode for Vec<T> {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        let n = Compact::decode(input)?.0;
        // every item takes at least one byte, so a longer list cannot fit
        if n > input.len() as u128 {
            return Err(DecodeError::TooLong);
        }
        (0..n).map(|_| T::decode(input)).collect()
    }
}
```

A vector decoder that does `Vec::with_capacity(n)` with a length it just read has handed the sender a memory allocator. Here every element takes at least one byte, so a length larger than the remaining input cannot be satisfied and is refused first (`TooLong`). ✓ `vec_known_answer_and_hostile_length` sends `0300000040` (2³⁰ elements, no data) to `Vec<u8>` and `Vec<u64>`. The collect after it allocates as it goes, so memory use is proportional to the input actually supplied. (Zero-sized item types would break the "at least one byte" argument; the crate has none.)

### 3.5 Against the real crate ✓

`parity-scale-codec` is a dev-dependency only: the crate's own codec is what the tests of the other modules use, and the real one is the oracle. ✓ 3,000 random compact values (random bit widths, so every mode and boundary is hit) encode to identical bytes; 500 random nested values (`Vec<(u32, Option<Vec<u8>>)>`) encode identically and decode back through *both* decoders; and 4,000 random byte strings go through `decode_all::<Compact>` and two composite decoders, where the only requirements are "never panic" and "if we accept, the real decoder accepts the same value". An oracle is only as good as your belief that it is independent of what you are testing; here it is written by other people and shares no code with this crate.

---

## 4. Hashing and storage keys ✓

A chain's state is a key-value map. A pallet's storage item gets a 32-byte **prefix**, and a *map* item then appends the hashed key:

```rust
/// How a map key is hashed into a storage key.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Hasher {
    /// BLAKE2-128 of the key, then the key itself (safe for keys an attacker picks).
    Blake2_128Concat,
    /// 64-bit xxHash of the key, then the key itself (fast; only for keys that are not attacker-chosen).
    Twox64Concat,
    /// The key as it is.
    Identity,
}

impl Hasher {
    /// Appends the hashed form of `key`.
    pub fn hash_to(self, key: &[u8], out: &mut Vec<u8>) {
        match self {
            Hasher::Blake2_128Concat => {
                out.extend_from_slice(&blake2_128(key));
                out.extend_from_slice(key);
            }
            Hasher::Twox64Concat => {
                out.extend_from_slice(&twox_64(key));
                out.extend_from_slice(key);
            }
            Hasher::Identity => out.extend_from_slice(key),
        }
    }

    /// The key that was hashed, if the hasher keeps it (the "Concat" ones do).
    pub fn recover(self, hashed: &[u8]) -> Option<&[u8]> {
        match self {
            Hasher::Blake2_128Concat => hashed.get(16..),
            Hasher::Twox64Concat => hashed.get(8..),
            Hasher::Identity => Some(hashed),
        }
    }
}
```

```rust
/// The 32-byte prefix of a storage item: `twox128(pallet) ++ twox128(item)`.
pub fn storage_prefix(pallet: &str, item: &str) -> [u8; 32] {
    let mut out = [0u8; 32];
    out[..16].copy_from_slice(&twox_128(pallet.as_bytes()));
    out[16..].copy_from_slice(&twox_128(item.as_bytes()));
    out
}

/// The key of one entry of a map item.
pub fn map_key(pallet: &str, item: &str, hasher: Hasher, key: &[u8]) -> Vec<u8> {
    let mut out = storage_prefix(pallet, item).to_vec();
    hasher.hash_to(key, &mut out);
    out
}
```

- `twox_128` is two 64-bit xxHash values (seeds 0 and 1) written little-endian. xxHash is fast and **not** collision-resistant against an adversary, which is fine for names the runtime author chooses.
- `Blake2_128Concat` hashes the key with BLAKE2b at 128 bits and then appends the key itself. It is the safe choice for keys an attacker can pick (an account id, a user-supplied string): they cannot cheaply steer into a chosen part of the key space. Appending the key makes the hash *reversible by reading*: `recover` returns the original, which is how an indexer lists "all accounts" by scanning a prefix (exercise 2).
- `Twox64Concat` is the cheaper version for keys that are not attacker-chosen (a counter, a sequential id); `Identity` stores the key as it is and is only for keys that are already uniformly random (for example a hash).

✓ Known answers computed in Python: `twox128("System")` = `26aa394eea5630e07c48ae0c9558cef7`; `twox128("Account")` = `b99d880ec681799c0cf30e8886371da9`; `twox128("Balances")` = `c2261276cc9d1f8598ea4b6a74b15c2f`; `twox128("TotalIssuance")` = `57c875e4cff74148e4628f264b974c80`; `twox128("Timestamp")` = `f0c365c3cf59d671eb72da0e7a4113c4`; `twox128("Now")` = `9f1f0515f462cdcf84e0f1d6045dfcbb`; `blake2b-256("")` = `0e5751c026e543b2e8ab2eb06099daa1d1e5df47778f7787faab45cdf12fe3a8`. Two of the joined prefixes, `26aa…cef7` ‖ `b99d…1da9` for the account map and `f0c3…c4` ‖ `9f1f…bb` for the clock, are keys I remember seeing in real chain queries; the Python programs reproduced them, and the tests pin them.

A consequence for clients: **a storage key is a pure function of names**, so a client can compute it without a node, read it with `state_getStorage`, and decode the value with the type from metadata. A consequence for security: if a runtime used `Identity` or a weak hasher on attacker-chosen keys, an attacker could crowd a key range or make iteration expensive for others.

---

## 5. SS58 addresses ✓

An account is a 32-byte public key. The text form people paste is **SS58**: base-58 of `prefix ‖ key ‖ checksum`, where the prefix names the network (42 is the generic Substrate prefix, which makes addresses start with `5`) and the checksum is the first two bytes of BLAKE2b-512 over `"SS58PRE" ‖ prefix ‖ key`.

```rust
fn prefix_bytes(prefix: u16) -> Result<Vec<u8>, Ss58Error> {
    match prefix {
        0..=63 => Ok(vec![prefix as u8]),
        64..=MAX_PREFIX => {
            let first = (((prefix & 0b0000_0000_1111_1100) >> 2) as u8) | 0b0100_0000;
            let second = ((prefix >> 8) as u8) | (((prefix & 0b11) as u8) << 6);
            Ok(vec![first, second])
        }
        _ => Err(Ss58Error::BadPrefix),
    }
}

fn checksum(prefix: &[u8], id: &[u8; 32]) -> [u8; 2] {
    let mut data = b"SS58PRE".to_vec();
    data.extend_from_slice(prefix);
    data.extend_from_slice(id);
    let h = blake2_512(&data);
    [h[0], h[1]]
}

/// The address text of `id` on the network with `prefix`.
pub fn encode(prefix: u16, id: &[u8; 32]) -> Result<String, Ss58Error> {
    let p = prefix_bytes(prefix)?;
    let mut raw = p.clone();
    raw.extend_from_slice(id);
    raw.extend_from_slice(&checksum(&p, id));
    Ok(base58::encode(&raw))
}
```

```rust
/// Reads an address back into its prefix and account id.
pub fn decode(text: &str) -> Result<(u16, [u8; 32]), Ss58Error> {
    let raw = base58::decode(text).map_err(|_| Ss58Error::BadBase58)?;
    let (prefix_len, prefix) = match raw.first() {
        Some(&b) if b < 64 => (1, u16::from(b)),
        Some(&b) if b < 128 => {
            let second = *raw.get(1).ok_or(Ss58Error::BadLength)?;
            // the shift is on a u8 on purpose: the 0b01 marker bits fall off the top
            let lower = u16::from(b << 2) | u16::from(second >> 6);
            let upper = u16::from(second) & 0b0011_1111;
            (2, lower | (upper << 8))
        }
        Some(_) => return Err(Ss58Error::BadPrefix),
        None => return Err(Ss58Error::BadLength),
    };
    if raw.len() != prefix_len + 32 + 2 {
        return Err(Ss58Error::BadLength);
    }
    // a two-byte form is only for prefixes that need it, and the reserved ones are refused
    if prefix_len == 2 && prefix < 64 || prefix == 46 || prefix == 47 {
        return Err(Ss58Error::BadPrefix);
    }
    let id: [u8; 32] = raw[prefix_len..prefix_len + 32].try_into().expect("32");
    if raw[prefix_len + 32..] != checksum(&raw[..prefix_len], &id) {
        return Err(Ss58Error::BadChecksum);
    }
    Ok((prefix, id))
}
```

### 5.1 The prefix has two sizes

A prefix below 64 is one byte. From 64 to 16,383 it takes two bytes: the first holds bits 2–7 of the prefix with `01` in the top bits, and the second holds the high bits and bits 0–1 (in its top two bits). Prefixes 46 and 47 are reserved and refused on decode, and a two-byte form of a prefix that fits in one byte is refused (a second encoding of the same address, §3.3 again).

### 5.2 What the checksum is for ✓

It catches typing and copying mistakes: ✓ `every_single_character_change_is_caught` replaces each of the 48 characters of an address with each of the other 57 base-58 digits, 2,736 edits in all, and every one fails to decode to the original. (With a 2-byte checksum, an *arbitrary* corruption slips through about 1 time in 65,536; the test shows that every single-character edit of this one address is caught, and nothing more: do not extrapolate to other addresses or to multi-character damage.) It does **not** authenticate anything: anyone can compute a valid address for any key, and it does not say who controls the key. And the prefix is part of the text only: the same key at prefix 0 and prefix 42 is the same account with two spellings (`15oF4u…` and `5Grwva…`), so code that compares address *strings* instead of 32-byte ids compares spellings. ✓ Alice's well-known sr25519 public key `d43593c7…a27d` gives `5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY` at 42 (a value I remembered; Python agreed), `15oF4uVJwmo4TdGW7VfQxNLavjCXviqxT9S1MgbjMNHr6Sp5` at 0 and `HNZata7iMYWmk5RvZRTiAsSDhV8366zq2YGb3tLH5Upf74F` at 2 (Python), and 100, 64 and 16383 give the two-byte forms in the tests.

### 5.3 A bug in my first version ✓

My first `decode` computed the low bits of the prefix as `(u16::from(first) << 2) | (second >> 6)`. That widens the byte *before* shifting, so the two marker bits `01` that the encoder put on top of the first byte are shifted into bit 8 of the prefix instead of falling off the end. The round-trip test over prefixes failed at 64, which came back as 320. Small prefixes (one byte) never touch this path, which is why every "5…" address test passed. The fix is to shift the *byte* (`first << 2` on a `u8`, which discards the top bits) and widen afterwards. The reference decoder does the same, as I remember it, with an 8-bit shift. Exercise 3 and the debugging challenge reproduce it; ✓ the buggy reader is wrong for exactly 8,128 of the 16,320 two-byte prefixes, those with bit 8 clear (when bit 8 is set, the stray bit is absorbed by the `|`). A bug that is wrong for half the inputs and right for the other half is the kind that survives a quick test.

---

## 6. The extrinsic ✓

An *extrinsic* is anything that comes from outside the chain: in practice, a transaction. The wire form in this model (format version 4, from memory the current one at the time of writing):

```text
compact(length of the rest)
version byte        0x84 = signed, v4      0x04 = unsigned, v4
-- signed only --
address type        0x00 = a 32-byte account id follows
signer              32 bytes
signature type      0x00 = ed25519
signature           64 bytes
era                 0x00 = immortal
nonce               compact
tip                 compact
-- always --
call                pallet index, call index, arguments
```

### 6.1 What the signature binds ✓

```rust
/// The bytes that get signed. Payloads over 256 bytes are hashed first.
pub fn signing_payload(call: &Call, extra: &Extra, ctx: &ChainContext) -> Vec<u8> {
    let mut p = call.encode();
    encode_extra(extra, &mut p);
    ctx.spec_version.encode_to(&mut p);
    ctx.tx_version.encode_to(&mut p);
    p.extend_from_slice(&ctx.genesis);
    p.extend_from_slice(&ctx.genesis); // block hash: immortal era uses genesis
    if p.len() > 256 {
        blake2_256(&p).to_vec()
    } else {
        p
    }
}
```

The signed payload is the call, the extra fields (era, nonce, tip) and four values that are **not sent**: the runtime's spec version and transaction version, the genesis hash and the "birth block" hash. The receiver recomputes them from its own view of the chain, so a signature made for one chain, or one runtime version, does not verify on another. If the payload is longer than 256 bytes, its BLAKE2-256 hash is signed instead (✓ the boundary is tested at exactly 256 and 257 bytes: a payload of 256 is signed as is, 257 is hashed). Compare Solana's recent blockhash (Lesson 53), which does the "which chain and when" job by a different route.

✓ `signature_binds_the_chain` and `signature_binds_every_signed_field`: changing the genesis hash, the spec version, the tx version, the call (value 1000 → 1001), the nonce, the tip or the signer each makes `verify` fail. ✓ The wire bytes of a transfer of 1000 units from the key of seed `07…07` to `02…02`, nonce 5, tip 0, spec version 100, tx version 1, genesis `aa…aa`, are the 141 bytes starting `2d028400ea4a6c63…` and ending `…0202a10f`, and the extrinsic hash is `c679b38f…855b`. These were built independently in Python (a manual SCALE encoding and the `cryptography` package's ed25519), and the Rust code produces the same bytes (ed25519 is deterministic, so the signature matches too) and also parses what Python built.

### 6.2 Encoding and parsing

```rust
/// The wire form: compact length, then the body.
pub fn to_bytes(&self) -> Vec<u8> {
    let mut body = Vec::new();
    match &self.signed {
        Some(s) => {
            body.push(V4_SIGNED);
            body.push(0); // MultiAddress::Id
            body.extend_from_slice(&s.signer);
            body.push(0); // MultiSignature::Ed25519
            body.extend_from_slice(&s.signature);
            encode_extra(&s.extra, &mut body);
        }
        None => body.push(V4_UNSIGNED),
    }
    self.call.encode_to(&mut body);
    let mut out = Compact(body.len() as u128).encode();
    out.extend_from_slice(&body);
    out
}
```

```rust
/// Parses the wire form; the length prefix must match exactly.
pub fn from_bytes(bytes: &[u8]) -> Result<Extrinsic, DecodeError> {
    let mut input = bytes;
    let Compact(len) = Compact::decode(&mut input)?;
    if len != input.len() as u128 {
        return Err(if len > input.len() as u128 {
            DecodeError::Eof
        } else {
            DecodeError::Trailing
        });
    }
    let version = u8::decode(&mut input)?;
    let signed = match version {
        V4_UNSIGNED => None,
        V4_SIGNED => {
            let tag = u8::decode(&mut input)?;
            if tag != 0 {
                return Err(DecodeError::InvalidTag(tag));
            }
            let signer = <[u8; 32]>::decode(&mut input)?;
            let sig_tag = u8::decode(&mut input)?;
            if sig_tag != 0 {
                return Err(DecodeError::InvalidTag(sig_tag));
            }
            let signature = <[u8; 64]>::decode(&mut input)?;
            let era = u8::decode(&mut input)?;
            if era != 0 {
                return Err(DecodeError::InvalidTag(era));
            }
            let Compact(nonce) = Compact::decode(&mut input)?;
            let nonce = u64::try_from(nonce).map_err(|_| DecodeError::Overflow)?;
            let Compact(tip) = Compact::decode(&mut input)?;
            Some(Signed {
                signer,
                signature,
                extra: Extra { nonce, tip },
            })
        }
        other => return Err(DecodeError::InvalidTag(other)),
    };
    let call = Call::decode(&mut input)?;
    if !input.is_empty() {
        return Err(DecodeError::Trailing);
    }
    Ok(Extrinsic { signed, call })
}
```

Every tag is checked (address type, signature type, era), the nonce must fit in 64 bits (`Overflow` otherwise), the length prefix must equal the remaining bytes exactly (`Eof` when it promises more, `Trailing` when there is more), and nothing may follow the call.

### 6.3 Nonce, tip, era

The **nonce** orders one account's transactions and makes each signed payload single-use *while the account exists* (§7.4 shows when it does not). The **tip** is an extra fee to the block author. The **era** says how long the transaction stays valid: the model supports only *immortal* (`00`); I remember a *mortal* form with a period and a phase that makes a transaction expire after a number of blocks, and the "birth block hash" in the payload is what ties it to a block. An immortal transaction signed with the genesis hash is valid forever, which is the weakest setting; real wallets use mortal ones.

### 6.4 Every bit ✓

✓ `flipping_any_bit_of_the_wire_form_breaks_or_changes_it` flips each of the 1,128 bits of the Python-built extrinsic. Each result either fails to parse, or parses to a different extrinsic whose signature does not verify. That includes the length prefix: a parser that ignored it would accept the flipped form as the original (a mutant that ignores it is caught by this very test, §16).

---

## 7. A tiny runtime ✓

The model chain keeps its state as a sorted map from storage key to SCALE bytes, exactly as §4 lays them out: `System.Account` maps an account id (hashed with `Blake2_128Concat`) to `{ nonce: u32, free: u128 }`, and `Balances.TotalIssuance` is a `u128`.

### 7.1 Admission, then execution

```rust
/// Validates and queues an extrinsic. Returns its hash.
pub fn submit(&mut self, bytes: &[u8]) -> Result<[u8; 32], Invalid> {
    let xt = Extrinsic::from_bytes(bytes).map_err(|_| Invalid::BadEncoding)?;
    let s = xt.signed.as_ref().ok_or(Invalid::Unsigned)?;
    if !xt.verify(&self.ctx) {
        return Err(Invalid::BadSignature);
    }
    let expected = self.next_nonce(&s.signer);
    if s.extra.nonce < expected {
        return Err(Invalid::Stale);
    }
    if s.extra.nonce > expected {
        return Err(Invalid::Future);
    }
    if self.account(&s.signer).free < BASE_FEE + s.extra.tip {
        return Err(Invalid::CannotPayFee);
    }
    let h = xt.hash();
    self.pool.push(xt);
    Ok(h)
}
```

`submit` is the pool's entry check: it parses, rejects unsigned extrinsics, verifies the signature against the chain, requires the nonce to be exactly the next expected one counting what is already pending (`Stale` below it, `Future` above it) and requires the sender to afford the fee. Block building then re-checks at execution, because the world can change between admission and inclusion.

```rust
/// Applies one extrinsic. `Err` means invalid at execution time (not included).
fn apply(&mut self, xt: &Extrinsic, events: &mut Vec<Event>) -> Result<(), Invalid> {
    let s = xt.signed.as_ref().ok_or(Invalid::Unsigned)?;
    let who = s.signer;
    let mut acc = self.account(&who);
    if u64::from(acc.nonce) != s.extra.nonce {
        return Err(if s.extra.nonce < u64::from(acc.nonce) {
            Invalid::Stale
        } else {
            Invalid::Future
        });
    }
    let fee = BASE_FEE + s.extra.tip;
    if acc.free < fee {
        return Err(Invalid::CannotPayFee);
    }
    // fee and nonce are committed before the call runs
    acc.free -= fee;
    acc.nonce += 1;
    self.burn(fee);
    self.put_account(&who, acc);
    let result = self.dispatch(&who, &xt.call, events);
    if let Err(error) = result {
        events.push(Event::Failed { who, error });
    }
    self.reap(&who, events);
    Ok(())
}
```

```rust
fn dispatch(
    &mut self,
    who: &[u8; 32],
    call: &Call,
    events: &mut Vec<Event>,
) -> Result<(), DispatchError> {
    match call {
        Call::Remark(r) => {
            events.push(Event::Remark {
                who: *who,
                len: r.len() as u32,
            });
            Ok(())
        }
        Call::Transfer { dest, value } => {
            if dest == who {
                return Err(DispatchError::SelfTransfer);
            }
            let mut from = self.account(who);
            if from.free < *value {
                return Err(DispatchError::InsufficientBalance);
            }
            let mut to = self.account(dest);
            if to.free + value < EXISTENTIAL_DEPOSIT {
                return Err(DispatchError::BelowExistentialDeposit);
            }
            from.free -= value;
            to.free += value;
            self.put_account(who, from);
            self.put_account(dest, to);
            events.push(Event::Transfer {
                from: *who,
                to: *dest,
                value: *value,
            });
            Ok(())
        }
    }
}
```

### 7.2 A failed call is still included ✓

The order inside `apply` is the point. Fee and nonce are committed **before** the call runs; the call then either succeeds or reports a `DispatchError`, which becomes an event, and the extrinsic stays in the block. An overdrawn transfer therefore costs its sender the fee and a nonce, and changes nothing else. ✓ `failed_dispatch_is_included_and_still_pays`: balance 500, transfer 9,999 → nonce 1, balance 499, receiver untouched, event `Failed(InsufficientBalance)`, supply still equals the sum of balances. This is close to what Lesson 55 §4.3 showed for Solana, where a failed transaction leaves nothing but the fee; the difference here is that the nonce moves too, and the failed extrinsic is part of the block. Two other outcomes exist: an extrinsic that is **invalid at execution time** (it became unaffordable, or its sender was reaped) is dropped and not included (✓ `a_transaction_that_became_unaffordable_is_dropped_not_included`).

### 7.3 Existential deposit and reaping ✓

An account must keep at least `EXISTENTIAL_DEPOSIT` (10 in the model); below it, the account is **reaped**: its storage entry is deleted and the dust is burned (✓ `a_sender_left_under_the_deposit_is_reaped_and_dust_is_burned`: the total issuance drops by the fee and by the 4 units of dust, and still equals the sum of the balances). The reason is state hygiene: without it, anyone could create millions of one-unit accounts and the chain would store them forever. The receiving side is checked too: a transfer that would leave a *new* account under the deposit fails with `BelowExistentialDeposit` (✓), and an account left with exactly the deposit stays (✓ `an_account_left_with_exactly_the_deposit_stays`, the boundary a mutant attacked, §16).

### 7.4 Reaping resets the nonce ✓

The account entry holds the nonce, so deleting the entry deletes the nonce. ✓ `reaping_resets_the_nonce_which_is_why_replay_protection_needs_more` signs a transfer at nonce 0, lets it drain the sender below the deposit, funds the sender again, and then **resubmits the same bytes**: the pool accepts it, because the account is at nonce 0 once more. In the model this is a real replay, and it is possible because the era is *immortal*. The remedies, as I remember them, are mortal eras (a transaction signed for a window of blocks cannot be replayed after the window) and keeping the account alive while it has a nonce; the lesson is that **"the nonce prevents replay" is true only as long as the nonce is stored.** The matching test `a_stale_nonce_after_reap_and_refund_is_not_executed` pins the other half: a transaction signed for nonce 1 is not executed against a reaped-and-refunded account that is back at 0.

### 7.5 Blocks and roots

```rust
/// Builds the next block from the pool, in order.
pub fn produce_block(&mut self) -> &Block {
    let pool = std::mem::take(&mut self.pool);
    let mut included = Vec::new();
    let mut events = Vec::new();
    for xt in pool {
        if self.apply(&xt, &mut events).is_ok() {
            included.push(xt.to_bytes());
        }
    }
    let parent = self.best().header.clone();
    let header = Header {
        parent_hash: parent.hash(),
        number: parent.number + 1,
        state_root: self.state_root(),
        extrinsics_root: extrinsics_root(&included),
    };
    self.blocks.push(Block {
        header,
        extrinsics: included,
        events,
    });
    self.best()
}
```

A header holds the parent hash, the block number (compact), a state root and an extrinsics root; the block hash is BLAKE2-256 of the SCALE encoding of the header, so a block commits to its parent and the chain is a hash chain (✓ `blocks_chain_by_hash`; exercise 6 writes the check). Two things are simplified on purpose. The **state root** is a hash of all sorted (key, value) pairs; Substrate uses a base-16 Merkle-Patricia trie (from memory), which allows a *proof* that one key has one value without sending the whole state. This plain hash cannot do that. And the **extrinsics root** is the hash of the SCALE list of extrinsics, not a trie; ✓ it is tested to depend on the list (two blocks that differ in one transfer amount have different roots). The model has no digest items, no consensus and no author.

### 7.6 Conservation ✓

`sum_balances` reads every account entry back out of storage and adds them. ✓ `conservation_holds_under_random_traffic` runs 40 rounds of random transfers (three signers, five receivers, amounts 0–399, tips 0–2), ignoring the ones the pool refuses, and after each block checks that the sum of balances equals the total issuance, that the block number advanced by one and that the header's state root equals the root of the current state.

---

## 8. A JSON-RPC front end ✓

```rust
/// Handles one request text and returns the response text.
pub fn handle(chain: &mut Chain, request: &str) -> String {
    let v: Value = match serde_json::from_str(request) {
        Ok(v) => v,
        Err(_) => return err(Value::Null, PARSE_ERROR, "Parse error").to_string(),
    };
    let id = v.get("id").cloned().unwrap_or(Value::Null);
    let Some(method) = v.get("method").and_then(Value::as_str) else {
        return err(id, INVALID_REQUEST, "Invalid Request").to_string();
    };
    if v.get("jsonrpc") != Some(&json!("2.0")) {
        return err(id, INVALID_REQUEST, "Invalid Request").to_string();
    }
    let params = v.get("params").cloned().unwrap_or_else(|| json!([]));
    let Some(params) = params.as_array() else {
        return err(id, INVALID_PARAMS, "Invalid params").to_string();
    };
    call(chain, method, params, id).to_string()
}

fn call(chain: &mut Chain, method: &str, params: &[Value], id: Value) -> Value {
    let bad = |id: Value| err(id, INVALID_PARAMS, "Invalid params");
    match method {
        "state_getStorage" => {
            let Some(key) = params.first().and_then(Value::as_str).and_then(from_hex) else {
                return bad(id);
            };
            match chain.storage(&key) {
                Some(v) => ok(id, json!(to_hex(v))),
                None => ok(id, Value::Null),
            }
        }
        "author_submitExtrinsic" => {
            let Some(bytes) = params.first().and_then(Value::as_str).and_then(from_hex) else {
                return bad(id);
            };
            match chain.submit(&bytes) {
                Ok(h) => ok(id, json!(to_hex(&h))),
                Err(e) => err(
                    id,
                    INVALID_TX,
                    &format!("Invalid Transaction: {}", invalid_name(e)),
                ),
            }
        }
        "chain_getHeader" => {
            let h = match params.first() {
                None | Some(Value::Null) => chain.best().header.clone(),
                Some(Value::String(s)) => {
                    let Some(hash) = from_hex(s) else {
                        return bad(id);
                    };
                    match (0..)
                        .map_while(|n| chain.block(n))
                        .find(|b| b.header.hash().as_slice() == hash)
                    {
                        Some(b) => b.header.clone(),
                        None => return ok(id, Value::Null),
                    }
                }
                _ => return bad(id),
            };
            ok(id, header_json(&h))
        }
        "chain_getBlockHash" => {
            let n = match params.first() {
                None | Some(Value::Null) => chain.best().header.number,
                Some(Value::Number(n)) => match n.as_u64() {
                    Some(n) => n,
                    None => return bad(id),
                },
                _ => return bad(id),
            };
            match chain.block(n) {
                Some(b) => ok(id, json!(to_hex(&b.header.hash()))),
                None => ok(id, Value::Null),
            }
        }
        "system_accountNextIndex" => {
            let Some((_, who)) = params
                .first()
                .and_then(Value::as_str)
                .and_then(|s| ss58::decode(s).ok())
            else {
                return bad(id);
            };
            ok(id, json!(chain.next_nonce(&who)))
        }
        "system_chain" => ok(id, json!("Teaching Dev")),
        "dev_produceBlock" => {
            let b = chain.produce_block();
            ok(
                id,
                json!({"number": b.header.number, "extrinsics": b.extrinsics.len()}),
            )
        }
        _ => err(id, METHOD_NOT_FOUND, "Method not found"),
    }
}
```

Six methods: `state_getStorage` (hex key → hex value or `null`), `author_submitExtrinsic` (hex bytes → hash, or error `1010` with a reason), `chain_getHeader`, `chain_getBlockHash`, `system_accountNextIndex` (an SS58 address → next nonce counting the pool, so a wallet can queue several transactions) and, as a stand-in for a node's development mode, `dev_produceBlock`. The method names and result shapes are from memory of the legacy API; the error code 1010 for an invalid transaction is also from memory.

The contract is small enough to pin completely. ✓ Parse errors (`-32700`, `id` null), a request without a method or with the wrong `jsonrpc` value (`-32600`), unknown methods (`-32601`), bad parameters (`-32602`, including a non-array `params`, a malformed hex string and a negative block number) and an invalid transaction (`1010`) each have a test, and so does each *reason* (`BadEncoding`, `UnsignedNotAllowed`, `BadProof`, `Stale`, `Future`, `Payment`) in the error text. The numbers are asserted as literals in their own test: the other tests compare codes with the crate's own constants, so a change to a constant would pass them all (the mutants that change `PARSE_ERROR` and `INVALID_TX` are caught only by the literal test, §16).

### 8.1 What a real Rust client does differently (from memory)

A real client does not hard-code this layout. It fetches the **metadata** from the node, builds the encoder and decoder for each call and storage item from it, and (in `subxt`, as I remember it) can generate typed Rust code at compile time from a metadata snapshot so that a call that does not exist fails to compile. The cost is that a runtime upgrade can change the metadata under a running client: the **spec version** in §6.1 is how a node refuses a transaction signed for the old runtime, and a client should re-fetch metadata when it changes. For reading history, the questions of Lesson 51 and Lesson 52 apply unchanged: track the *finalized* head for anything that moves money, handle reorganizations of the best head, make writes idempotent (the extrinsic hash is the key), and treat the node as untrusted input.

---

## 9. Cost ✓

Release build, single machine, three runs of `examples/bench.rs` (lowest and highest):

- compact encode of a 4-byte-mode value: 19 ns; compact decode: 7 ns.
- storage map key (two `twox128`, one `blake2_128`): 203–269 ns.
- SS58 encode: 1.08–1.19 µs; SS58 decode: 635–652 ns (base-58 conversion and a BLAKE2b-512 each).
- extrinsic signing (ed25519): 26.1–26.8 µs; parse and verify: 35.5–41.2 µs.
- a block of 50 signed transfers, including signing, submitting and building from a fresh chain: 3.54–3.56 ms in total, about 71 µs per transfer, almost all of it ed25519.

The pattern is the usual one for chains (Lesson 49): the codec and the hashes are cheap, **signatures dominate**. These are micro-measurements of a model on a shared two-core virtual machine; they say nothing about the real runtime, whose cost is governed by weight accounting and storage access (from memory), not by these functions.

---

## 10. Rust notes

- **Traits as the codec boundary.** `Encode` appends to a `&mut Vec<u8>` (no intermediate allocation per field); `Decode` takes `&mut &[u8]` and advances the slice, which is how a decoder consumes exactly what it needs and leaves the rest. `decode_all` adds the "nothing left over" rule in one place.
- **Macros for the repetitive impls.** `fixed_int!` generates the five integer impls. Reading the generated code is not needed; reading the *tests* of it is.
- **`const` generics.** `[u8; N]` implements both traits for every `N`, so `[u8; 32]` ids and `[u8; 64]` signatures need no wrapper type.
- **Newtypes.** `Compact(pub u128)` is a newtype so that "a compact-encoded number" is a different type from `u128` (Lesson 15).
- **Errors as a closed enum.** `DecodeError` has one variant per reason, so a test can name the reason (`NonCanonical`, `TooLong`, `Trailing`), and a client can map them (Lesson 26).
- **`BTreeMap` for state.** Sorted iteration makes the state root deterministic and lets `sum_balances` scan by prefix (Lesson 50).
- **A dev-dependency as an oracle.** `parity-scale-codec` appears only in `[dev-dependencies]`, so the shipped library does not depend on it and the tests do.
- **Dependencies in the dev profile.** `[profile.dev.package."*"] opt-level = 3` makes ed25519 and the hashes fast in `cargo test` without slowing compilation of the crate itself; before adding it the extrinsic tests took about 7 seconds.

---

## 11. Common mistakes

1. Decoding with `Vec::with_capacity(length_from_the_wire)` (§3.4).
2. Accepting several encodings of one value, then hashing bytes while comparing values (§3.3).
3. Comparing address *strings* across networks instead of 32-byte ids (§5.2).
4. Treating the SS58 checksum as authentication (§5.2).
5. Using a cheap hasher for attacker-chosen map keys (§4).
6. Believing a signature binds only the call; it also binds the chain and the runtime version, so a signed payload built for a test network is not portable, and a client that caches spec versions signs for the wrong one after an upgrade (§6.1).
7. Treating "the nonce stops replay" as unconditional (§7.4).
8. Assuming a failed extrinsic is not in the block, and not charged (§7.2).
9. Following the best head for payments instead of the finalized head (Lesson 52).
10. Hard-coding pallet and call indexes across runtime upgrades instead of reading metadata (§8.1).
11. Parsing a block's extrinsics without checking each length prefix against the remaining bytes (exercise 5).
12. Tests that compare error codes against the crate's own constants, so changing the constant is invisible (§8, §16).
13. A round-trip test that only covers inputs where encode and decode share a bug, such as small prefixes (§5.3); an independent oracle is what breaks the tie.
14. Calling a model "the node". The model has no consensus, no trie, no weights, no WebAssembly and no networking.

---

## 12. Security notes and honest limits

- This is a **model written from memory**, with a codec that agrees with the real one on every tested value and with everything else unverified against the real runtime. I ran no node and no real client library. A clean test run says nothing about a deployed chain.
- The state root is not a Merkle-Patricia trie: no light-client proofs, no partial verification. The extrinsics root is a plain hash.
- The model signs with ed25519 only. The real chains use sr25519 (a Schnorr scheme over Ristretto, from memory) for most accounts; its signatures are randomized, so a known-answer test would need a fixed nonce source.
- Parsing and verifying a transaction you did not construct is a security boundary: refuse non-canonical encodings, trailing bytes and unknown tags, and verify the signature against *your* view of the chain before spending any other effort on it.
- A signature over an immortal era is a bearer instrument for as long as the account's nonce and balance allow (§7.4). Do not sign immortal transactions on real chains.
- Addresses are not authentication, and the same key has many spellings.
- Mutation testing here measures this crate's tests (§16), not the real stack.
- If you find a vulnerability in a real chain or client, report it privately to its maintainers and give them time to fix it. Never test anything against a network you do not own or have not been authorized to test.

---

## 13. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Write `compact_size(n)`: how many bytes the compact encoding of `n` takes, without encoding it.
2. **(Easy)** Write `account_of_key(key)`: return the account id from a `System.Account` storage key, or `None` if the key is anything else (including a key whose hash part does not match the id).
3. **(Easy)** Write `read_prefix(bytes)`: the network prefix and how many bytes it took, from the front of decoded SS58 bytes, for both the one-byte and two-byte forms.
4. **(Medium)** Write `next_nonce(account_nonce, pending)`: the next usable nonce when some transactions are already in the pool, given in any order.
5. **(Medium)** Write `split(body)`: cut a block body (extrinsics, each with its own compact length prefix) into the individual extrinsics, refusing truncated input.
6. **(Medium-hard)** Write `check_chain(headers)`: verify that parent hashes and block numbers form a chain, and report the first header that does not.
7. **(Hard)** Write `diff(a, b)` for two storage maps: which keys were created, deleted or changed, in key order.
8. **(Hard)** Write `classify(text)`: decide whether a request text deserves a JSON-RPC parse error, an invalid-request error, or is a request, and say which method.

<details markdown="1">
<summary>Answers</summary>

1. A possible answer (✓ compiled and run in a scratch crate with tests):

   ```rust
   //! Exercise 1: how many bytes does a compact integer take, without encoding it?

   /// The length of the compact encoding of `n`.
   pub fn compact_size(n: u128) -> usize {
       match n {
           0..=0x3f => 1,
           0x40..=0x3fff => 2,
           0x4000..=0x3fff_ffff => 4,
           _ => {
               let bytes = 16 - n.leading_zeros() as usize / 8;
               1 + bytes
           }
       }
   }
   ```

   Three fixed sizes and then `1 + n` bytes, `n` being the byte length of the value. The test compares the function with both encoders for every power of two from 2⁰ to 2¹²⁷ and its neighbours, plus the mode boundaries.
2. A possible answer (✓):

   ```rust
   //! Exercise 2: read the account id back out of a `System.Account` storage key.

   use substratelab::hashing::{Hasher, blake2_128, storage_prefix};

   /// `Some(id)` only if `key` is exactly `System.Account` + `blake2_128(id)` + `id`
   /// and the stored hash really is the hash of that id.
   pub fn account_of_key(key: &[u8]) -> Option<[u8; 32]> {
       let prefix = storage_prefix("System", "Account");
       let rest = key.strip_prefix(&prefix[..])?;
       let id: [u8; 32] = Hasher::Blake2_128Concat.recover(rest)?.try_into().ok()?;
       (rest[..16] == blake2_128(&id)).then_some(id)
   }
   ```

   It strips the 32-byte prefix, recovers the id using the hasher, and checks that the 16 hash bytes really are `blake2_128(id)`. Without that last check, a key with a corrupted hash part would be accepted; the test flips one hash bit and also refuses a key from another pallet (`Balances.Account` has the same shape), a key with a byte missing and a key with a byte added. This is the function an indexer uses to turn a scanned key back into an account.
3. A possible answer (✓):

   ```rust
   //! Exercise 3: the network prefix at the front of decoded SS58 bytes.

   /// The prefix and how many bytes it took, or `None` if the first byte is not a prefix start.
   pub fn read_prefix(bytes: &[u8]) -> Option<(u16, usize)> {
       let first = *bytes.first()?;
       match first {
           0..=63 => Some((u16::from(first), 1)),
           64..=127 => {
               let second = *bytes.get(1)?;
               // the shift is on a u8 on purpose: the two marker bits fall off the top
               let lower = u16::from(first << 2) | u16::from(second >> 6);
               let upper = u16::from(second & 0b0011_1111);
               Some((lower | (upper << 8), 2))
           }
           _ => None,
       }
   }
   ```

   Note the comment: `first << 2` is on the `u8`. The test encodes prefixes 0, 1, 42, 63, 64, 65, 255, 256, 1000 and 16383 with the crate and reads them back, and refuses an empty input, a lone first byte of a two-byte prefix, and a first byte over 127.
4. A possible answer (✓):

   ```rust
   //! Exercise 4: the next usable nonce when some transactions are already pending.

   /// `account_nonce` is what the chain has; `pending` are nonces of transactions in the pool,
   /// in any order. The next nonce is the end of the unbroken run that starts at `account_nonce`.
   pub fn next_nonce(account_nonce: u64, pending: &[u64]) -> u64 {
       let mut seen: Vec<u64> = pending
           .iter()
           .copied()
           .filter(|n| *n >= account_nonce)
           .collect();
       seen.sort_unstable();
       seen.dedup();
       let mut next = account_nonce;
       for n in seen {
           if n == next {
               next += 1;
           } else {
               break;
           }
       }
       next
   }
   ```

   The next nonce is the end of the unbroken run starting at the account's nonce; a gap stops the run (a pending transaction at 7 does not help while 6 is missing), duplicates count once and stale entries below the account nonce are ignored. This is what `system_accountNextIndex` has to compute, and the crate's `next_nonce` does the same walk over the pool.
5. A possible answer (✓):

   ```rust
   //! Exercise 5: cut a block body into extrinsics.

   use substratelab::scale::{Compact, Decode, DecodeError};

   /// A body is a sequence of extrinsics, each with its own compact length prefix.
   /// Returns each whole extrinsic, prefix included.
   pub fn split(mut body: &[u8]) -> Result<Vec<&[u8]>, DecodeError> {
       let mut out = Vec::new();
       while !body.is_empty() {
           let start = body;
           let Compact(len) = Compact::decode(&mut body)?;
           let len = usize::try_from(len).map_err(|_| DecodeError::TooLong)?;
           if body.len() < len {
               return Err(DecodeError::Eof);
           }
           let used = start.len() - body.len() + len;
           out.push(&start[..used]);
           body = &body[len..];
       }
       Ok(out)
   }
   ```

   Each part includes its own prefix, so the parts can be hashed or forwarded unchanged. It refuses a length larger than the rest (`Eof`) and a length that does not fit in a `usize` (the test passes a 16-byte compact integer of all `ff`). It never allocates from a length before checking it against the input.
6. A possible answer (✓):

   ```rust
   //! Exercise 6: check that a list of headers forms a chain.

   use substratelab::runtime::Header;

   /// `Ok(())` if every header's parent hash is the previous header's hash and its number is
   /// one higher; otherwise the index of the first header that breaks the chain.
   pub fn check_chain(headers: &[Header]) -> Result<(), usize> {
       for (i, w) in headers.windows(2).enumerate() {
           if w[1].parent_hash != w[0].hash() || w[1].number != w[0].number + 1 {
               return Err(i + 1);
           }
       }
       Ok(())
   }
   ```

   The test builds five real blocks, then corrupts one header's state root (so the *next* header's parent hash no longer matches: the break is reported at the next index, 3), another's number, and swaps two headers. A shorter or empty list is a valid chain.
7. A possible answer (✓):

   ```rust
   //! Exercise 7: what changed between two storage snapshots.

   use std::collections::BTreeMap;

   #[derive(Debug, Clone, PartialEq, Eq)]
   pub enum Change {
       Created(Vec<u8>),
       Deleted(Vec<u8>),
       Changed(Vec<u8>),
   }

   /// Changes in key order.
   pub fn diff(a: &BTreeMap<Vec<u8>, Vec<u8>>, b: &BTreeMap<Vec<u8>, Vec<u8>>) -> Vec<Change> {
       let mut keys: Vec<&Vec<u8>> = a.keys().chain(b.keys()).collect();
       keys.sort();
       keys.dedup();
       keys.into_iter()
           .filter_map(|k| match (a.get(k), b.get(k)) {
               (None, Some(_)) => Some(Change::Created(k.clone())),
               (Some(_), None) => Some(Change::Deleted(k.clone())),
               (Some(x), Some(y)) if x != y => Some(Change::Changed(k.clone())),
               _ => None,
           })
           .collect()
   }
   ```

   It merges the two key sets, sorts and removes duplicates, and classifies each key. The same comparison, applied to the chain's storage before and after a block, is the first thing to print when a state-root mismatch is reported.
8. A possible answer (✓):

   ```rust
   //! Exercise 8: which JSON-RPC error does a request text deserve?

   use serde_json::Value;

   #[derive(Debug, PartialEq, Eq)]
   pub enum Class {
       /// Not JSON at all (-32700).
       ParseError,
       /// JSON, but not a request object (-32600).
       InvalidRequest,
       /// A request, with its method name.
       Request(String),
   }

   pub fn classify(text: &str) -> Class {
       let Ok(v) = serde_json::from_str::<Value>(text) else {
           return Class::ParseError;
       };
       let ok = v.get("jsonrpc") == Some(&Value::String("2.0".into()));
       match (ok, v.get("method").and_then(Value::as_str)) {
           (true, Some(m)) => Class::Request(m.to_string()),
           _ => Class::InvalidRequest,
       }
   }
   ```

   Not JSON → parse error; JSON that is not an object with `"jsonrpc": "2.0"` and a string `"method"` → invalid request; otherwise a request. The test also runs the same three texts through the crate's real front end and checks that the codes agree.

</details>

---

## 14. Debugging challenge

A wallet developer writes a reader for the network prefix at the front of decoded SS58 bytes. Their tests use addresses starting with `5` (prefix 42) and a few others, and all pass. A user on a network with prefix 100 reports that their address "belongs to network 356". Find the bug and fix it.

```rust
/// The faulty program.
pub mod buggy {
    /// Reads the network prefix from the front of decoded SS58 bytes.
    pub fn read_prefix(bytes: &[u8]) -> Option<(u16, usize)> {
        let first = *bytes.first()?;
        match first {
            0..=63 => Some((u16::from(first), 1)),
            64..=127 => {
                let second = *bytes.get(1)?;
                let lower = (u16::from(first) << 2) | u16::from(second >> 6);
                let upper = u16::from(second & 0b0011_1111);
                Some((lower | (upper << 8), 2))
            }
            _ => None,
        }
    }
}
```

<details markdown="1">
<summary>Solution</summary>

The bug is in the line that builds `lower`: `(u16::from(first) << 2)` widens `first` to 16 bits **before** shifting, so the two marker bits `01` in the top of the first byte move into bit 8 of the result instead of being discarded. Prefix 100 is `0b0110_0100`: the encoder writes the first byte as `((100 & 0xfc) >> 2) | 0x40 = 0x59` and the second as `(100 >> 8) | ((100 & 3) << 6) = 0x00`; the buggy reader computes `0x59 << 2 = 0x164` (356) instead of `0x64` (100), because the marker bit survived. One-byte prefixes (below 64) never reach this code, so every common address passes.

Why it is not wrong for every two-byte prefix: the stray bit lands in bit 8, and the result is combined with `upper << 8` by `|`. When the prefix's own bit 8 is set, the stray bit is absorbed. ✓ The test measures it: the buggy reader is wrong for exactly 8,128 of the 16,320 prefixes from 64 to 16,383 (the ones with bit 8 clear), and right for the rest.

Fixed version (✓ compiled; the test compares it with the crate's encoder for every prefix from 64 to 16,383):

```rust
/// The corrected program.
pub mod fixed {
    /// Reads the network prefix from the front of decoded SS58 bytes.
    pub fn read_prefix(bytes: &[u8]) -> Option<(u16, usize)> {
        let first = *bytes.first()?;
        match first {
            0..=63 => Some((u16::from(first), 1)),
            64..=127 => {
                let second = *bytes.get(1)?;
                // `first << 2` must drop the two marker bits, so shift the byte, then widen
                let lower = u16::from(first << 2) | u16::from(second >> 6);
                let upper = u16::from(second & 0b0011_1111);
                Some((lower | (upper << 8), 2))
            }
            _ => None,
        }
    }
}
```

The general lesson: **bit-field code needs a test over the whole domain, not a few samples**, and a round-trip against an independent encoder is stronger than a handful of known addresses. When the domain is small enough (here 16,384 values) just loop over all of it.

</details>

---

## 15. Interview questions

1. *What is SCALE, and how does it differ from JSON or Protocol Buffers?*
2. *Describe the four modes of a compact integer. Why does a decoder have to reject non-canonical encodings?*
3. *A peer sends a vector with a length prefix of 2³⁰ and no data. What should your decoder do, and why?*
4. *How is a storage key for a map item built? Why `Blake2_128Concat` for accounts but a cheaper hasher for some other maps?*
5. *What is SS58, and what does its checksum protect against? What does it not protect against?*
6. *The same key has two SS58 addresses on two networks. How should a service store accounts?*
7. *What does a signature on a Substrate-style transaction cover? Why the genesis hash and the spec version?*
8. *How do nonce and era together prevent replay? Describe a case in which the nonce alone does not.*
9. *Why is a failed call still included in a block, and who pays?*
10. *What is the existential deposit for? What can go wrong with a small transfer to a new account?*
11. *Explain relay chain versus parachain, and what "shared security" means.*
12. *How does a generic client know how to encode a call it was not compiled against? What happens at a runtime upgrade?*
13. *Best head versus finalized head: which do you use for an exchange deposit, and what does an indexer do on a reorganization?*
14. *How would you test client code for a chain without running a node? What would the tests not tell you?*
15. *You are asked to review a decoder for a binary protocol. What do you check first?*

<details markdown="1">
<summary>Answers</summary>

1. SCALE is a compact binary encoding with no field names or type tags: the decoder must already know the type (from the metadata or the source). Fixed integers are little-endian, collections have a compact length prefix, enums a variant index. JSON is self-describing and text; Protocol Buffers has field numbers and wire types so that it can skip unknown fields. SCALE is smaller and simpler and has no forward compatibility built into the bytes, which is why versioning lives in the runtime (spec version, metadata).
2. Mode `00`: one byte, 0–63; `01`: two bytes, up to 2¹⁴−1; `10`: four bytes, up to 2³⁰−1; `11`: a first byte giving the number of following bytes minus four, then that many little-endian bytes. Each value has exactly one shortest encoding. A decoder that accepts longer forms makes two byte strings mean one value, so a hash or signature over the bytes covers one of them while a comparison of values treats both as equal, which opens malleability (one transaction, two ids).
3. Refuse it before allocating: each element takes at least one byte, so the length cannot exceed the remaining input. Otherwise `with_capacity` hands the sender a memory allocation of their choosing. Also bound total input size at the network layer.
4. `twox128(pallet) ‖ twox128(item) ‖ hash(key) ‖ key` for the `*Concat` hashers. `Blake2_128Concat` is cryptographic, so an attacker cannot choose keys to cluster; it is right for attacker-chosen keys such as accounts. `Twox64Concat` is faster and acceptable when keys come from the runtime (sequential ids). `Identity` for keys that are already uniform. The concatenated key lets you recover the original key by reading the storage key.
5. Base-58 of prefix, 32-byte key and a 2-byte BLAKE2b-512-based checksum. It catches typos and wrong-network pastes (the prefix is inside the checksum). It does not authenticate the owner, hide anything, or catch deliberate substitution of another valid address; two bytes also let about one in 65,536 random corruptions through.
6. Store the 32-byte account id (and, if useful, the network separately); render SS58 at the edge for the network in use. Comparing strings compares spellings.
7. The call, era, nonce, tip, and four values not on the wire: spec version, transaction version, genesis hash and the birth block hash. Genesis hash ties the signature to one chain, spec/tx version to one runtime generation; so a signature cannot be replayed on a fork, another chain, or after an incompatible upgrade.
8. The nonce makes each signed payload valid once per account state; a mortal era limits the validity to a window. If an account is reaped, its nonce is deleted and resets to zero, so an old immortal transaction can become valid again after the account is refunded (§7.4 reproduces it).
9. Fee and nonce are charged before the call runs, so a failing call cannot be used to get free block space; the failure is recorded as an event. The sender pays. Only an extrinsic that is invalid before execution (bad signature, bad nonce, cannot pay) is excluded.
10. It forces accounts to hold a minimum so the chain does not store unlimited dust accounts; below it the account is reaped and the dust is burned. A transfer smaller than the deposit to a new account fails; a sender who spends down to just under the deposit loses the remainder and the nonce.
11. The relay chain coordinates and finalizes; parachains are separate chains with their own state transition function, whose blocks the relay chain's validators check. Security is "shared" because a parachain does not need its own validator set to get finality (from memory; details depend on current protocol parameters).
12. It fetches metadata, which describes pallets, calls and types in SCALE, and builds encoders from it (or generates typed code from a snapshot). At a runtime upgrade the spec version changes; transactions signed for the old one are rejected, and a client must refresh metadata (and any cached types).
13. Finalized for anything irreversible; the best head can be reorganized. An indexer stores blocks by hash with their parent, treats the best chain as provisional, and on a reorganization rolls back to the common ancestor and replays; writes are keyed by extrinsic hash and block hash so they are idempotent (Lessons 51 and 52).
14. Test encoders and decoders against known vectors and an independent implementation, test the pool and block logic in a model, and test RPC handling with recorded or synthetic responses. This does not tell you about real consensus, real weights, runtime upgrades, networking, or whether your metadata matches the live chain.
15. Every length against the remaining input; every tag and enum index; canonical forms; trailing bytes; integer widths and overflow; recursion depth; what happens on truncation; then fuzz it and cross-check with a second implementation.

</details>

---

## 16. Practical challenge: `substratelab`

Build a library crate that:

1. Implements **SCALE** (`Encode`, `Decode`, `decode_all`): fixed-width little-endian integers, `bool`, compact integers in all four modes with canonical-only decoding, `Vec`, `String`, `Option`, arrays and pairs; hostile lengths refused before allocation; checked against the real `parity-scale-codec` as a dev-dependency.
2. Implements **hashing and storage keys**: BLAKE2b at 128, 256 and 512 bits, `twox_64`, `twox_128`, the three hashers with key recovery, `storage_prefix` and `map_key`.
3. Implements **SS58**: one- and two-byte prefixes, the checksum, reserved prefixes, strict decoding.
4. Implements the **extrinsic**: signed (ed25519, deterministic from a seed) and unsigned v4 format, the signing payload with chain context and the 256-byte hashing rule, strict parsing, and the extrinsic hash.
5. Implements a **runtime and block builder**: `System.Account` and `Balances.TotalIssuance` in SCALE under real storage keys, a pool with nonce rules, fees, transfer and remark calls, dispatch errors that still pay, an existential deposit with reaping, events, headers that chain by hash and a state root.
6. Implements a **JSON-RPC front end** with the six methods of §8, passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, has a benchmark example and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

The path assumes the Lesson 53 crate sits at `../../p53/solanalab` in my scratch directory (it is used only for its base-58 functions); point it at wherever you keep that crate, or copy the base-58 code.

```toml
[package]
name = "substratelab"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
blake2 = "0.11.0"
ed25519-dalek = "3.0.0"
hex = "0.4.3"
serde_json = "1.0.154"
solanalab = { version = "0.1.0", path = "../../p53/solanalab" }
twox-hash = "2.1.5"

[dev-dependencies]
parity-scale-codec = "3.7.5"

[profile.dev.package."*"]
opt-level = 3
```

### `src/lib.rs`

```rust
//! A teaching model of the Substrate/Polkadot stack: the SCALE codec, storage
//! keys, SS58 addresses, signed extrinsics, a small runtime with a block
//! builder, and a JSON-RPC front end. **A model written from memory of the
//! design; not the real stack; not audited; not for production.**

pub mod extrinsic;
pub mod hashing;
pub mod rpc;
pub mod runtime;
pub mod scale;
pub mod ss58;
```

### `src/scale.rs`

```rust
//! The SCALE codec: the compact, non-self-describing binary format Substrate
//! uses for storage, extrinsics and RPC payloads. Decoding is strict: a value
//! has exactly one valid encoding, and anything else is an error.

/// Why decoding failed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DecodeError {
    /// The input ended early.
    Eof,
    /// A compact integer was not in its shortest form.
    NonCanonical,
    /// A compact integer does not fit the target type.
    Overflow,
    /// A boolean byte other than 0 or 1.
    InvalidBool(u8),
    /// An enum or option tag with no meaning.
    InvalidTag(u8),
    /// A length that cannot be satisfied by the remaining input.
    TooLong,
    /// Text that is not UTF-8.
    Utf8,
    /// Bytes left over after a complete value.
    Trailing,
}

/// Values with a SCALE encoding.
pub trait Encode {
    /// Appends the encoding.
    fn encode_to(&self, out: &mut Vec<u8>);

    /// The encoding as a new vector.
    fn encode(&self) -> Vec<u8> {
        let mut v = Vec::new();
        self.encode_to(&mut v);
        v
    }
}

/// Values that can be read back.
pub trait Decode: Sized {
    /// Reads one value and advances the input past it.
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError>;
}

/// Decodes a value that must use every byte of `bytes`.
pub fn decode_all<T: Decode>(bytes: &[u8]) -> Result<T, DecodeError> {
    let mut input = bytes;
    let v = T::decode(&mut input)?;
    if input.is_empty() {
        Ok(v)
    } else {
        Err(DecodeError::Trailing)
    }
}

fn take<'a>(input: &mut &'a [u8], n: usize) -> Result<&'a [u8], DecodeError> {
    if input.len() < n {
        return Err(DecodeError::Eof);
    }
    let (head, rest) = input.split_at(n);
    *input = rest;
    Ok(head)
}

macro_rules! fixed_int {
    ($($t:ty),*) => {$(
        impl Encode for $t {
            fn encode_to(&self, out: &mut Vec<u8>) {
                out.extend_from_slice(&self.to_le_bytes());
            }
        }
        impl Decode for $t {
            fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
                let b = take(input, std::mem::size_of::<$t>())?;
                Ok(<$t>::from_le_bytes(b.try_into().expect("exact size")))
            }
        }
    )*};
}
fixed_int!(u8, u16, u32, u64, u128);

impl Encode for bool {
    fn encode_to(&self, out: &mut Vec<u8>) {
        out.push(u8::from(*self));
    }
}
impl Decode for bool {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        match take(input, 1)?[0] {
            0 => Ok(false),
            1 => Ok(true),
            b => Err(DecodeError::InvalidBool(b)),
        }
    }
}

/// A "compact" integer: small values take few bytes.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Compact(pub u128);

impl Encode for Compact {
    fn encode_to(&self, out: &mut Vec<u8>) {
        let v = self.0;
        if v < 1 << 6 {
            out.push((v as u8) << 2);
        } else if v < 1 << 14 {
            out.extend_from_slice(&(((v as u16) << 2) | 0b01).to_le_bytes());
        } else if v < 1 << 30 {
            out.extend_from_slice(&(((v as u32) << 2) | 0b10).to_le_bytes());
        } else {
            let bytes = v.to_le_bytes();
            let n = 16 - (v.leading_zeros() as usize / 8);
            out.push((((n - 4) as u8) << 2) | 0b11);
            out.extend_from_slice(&bytes[..n]);
        }
    }
}

impl Decode for Compact {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        let first = take(input, 1)?[0];
        match first & 0b11 {
            0b00 => Ok(Compact(u128::from(first >> 2))),
            0b01 => {
                let rest = take(input, 1)?[0];
                let v = u16::from_le_bytes([first, rest]) >> 2;
                if v < 1 << 6 {
                    return Err(DecodeError::NonCanonical);
                }
                Ok(Compact(u128::from(v)))
            }
            0b10 => {
                let rest = take(input, 3)?;
                let v = u32::from_le_bytes([first, rest[0], rest[1], rest[2]]) >> 2;
                if v < 1 << 14 {
                    return Err(DecodeError::NonCanonical);
                }
                Ok(Compact(u128::from(v)))
            }
            _ => {
                let n = usize::from(first >> 2) + 4;
                if n > 16 {
                    return Err(DecodeError::Overflow);
                }
                let b = take(input, n)?;
                if b[n - 1] == 0 {
                    return Err(DecodeError::NonCanonical);
                }
                let mut buf = [0u8; 16];
                buf[..n].copy_from_slice(b);
                let v = u128::from_le_bytes(buf);
                if v < 1 << 30 {
                    return Err(DecodeError::NonCanonical);
                }
                Ok(Compact(v))
            }
        }
    }
}

impl<T: Encode> Encode for Vec<T> {
    fn encode_to(&self, out: &mut Vec<u8>) {
        Compact(self.len() as u128).encode_to(out);
        for x in self {
            x.encode_to(out);
        }
    }
}
impl<T: Decode> Decode for Vec<T> {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        let n = Compact::decode(input)?.0;
        // every item takes at least one byte, so a longer list cannot fit
        if n > input.len() as u128 {
            return Err(DecodeError::TooLong);
        }
        (0..n).map(|_| T::decode(input)).collect()
    }
}

impl Encode for String {
    fn encode_to(&self, out: &mut Vec<u8>) {
        Compact(self.len() as u128).encode_to(out);
        out.extend_from_slice(self.as_bytes());
    }
}
impl Decode for String {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        let n = Compact::decode(input)?.0;
        if n > input.len() as u128 {
            return Err(DecodeError::TooLong);
        }
        let b = take(input, n as usize)?;
        String::from_utf8(b.to_vec()).map_err(|_| DecodeError::Utf8)
    }
}

impl<T: Encode> Encode for Option<T> {
    fn encode_to(&self, out: &mut Vec<u8>) {
        match self {
            None => out.push(0),
            Some(x) => {
                out.push(1);
                x.encode_to(out);
            }
        }
    }
}
impl<T: Decode> Decode for Option<T> {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        match take(input, 1)?[0] {
            0 => Ok(None),
            1 => Ok(Some(T::decode(input)?)),
            b => Err(DecodeError::InvalidTag(b)),
        }
    }
}

impl<const N: usize> Encode for [u8; N] {
    fn encode_to(&self, out: &mut Vec<u8>) {
        out.extend_from_slice(self);
    }
}
impl<const N: usize> Decode for [u8; N] {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        Ok(take(input, N)?.try_into().expect("exact size"))
    }
}

impl<A: Encode, B: Encode> Encode for (A, B) {
    fn encode_to(&self, out: &mut Vec<u8>) {
        self.0.encode_to(out);
        self.1.encode_to(out);
    }
}
impl<A: Decode, B: Decode> Decode for (A, B) {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        Ok((A::decode(input)?, B::decode(input)?))
    }
}
```

### `src/hashing.rs`

```rust
//! The hash functions Substrate uses, and the storage-key layout built from them.

use blake2::digest::consts::{U16, U32};
use blake2::{Blake2b, Blake2b512, Digest};
use twox_hash::XxHash64;

/// BLAKE2b with a 256-bit output.
pub fn blake2_256(data: &[u8]) -> [u8; 32] {
    Blake2b::<U32>::digest(data).into()
}

/// BLAKE2b with a 128-bit output.
pub fn blake2_128(data: &[u8]) -> [u8; 16] {
    Blake2b::<U16>::digest(data).into()
}

/// BLAKE2b with a 512-bit output.
pub fn blake2_512(data: &[u8]) -> [u8; 64] {
    Blake2b512::digest(data).into()
}

/// 64-bit xxHash with seed 0, little-endian.
pub fn twox_64(data: &[u8]) -> [u8; 8] {
    XxHash64::oneshot(0, data).to_le_bytes()
}

/// Two xxHash64 values (seeds 0 and 1), concatenated: 128 bits.
pub fn twox_128(data: &[u8]) -> [u8; 16] {
    let mut out = [0u8; 16];
    out[..8].copy_from_slice(&XxHash64::oneshot(0, data).to_le_bytes());
    out[8..].copy_from_slice(&XxHash64::oneshot(1, data).to_le_bytes());
    out
}

/// How a map key is hashed into a storage key.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Hasher {
    /// BLAKE2-128 of the key, then the key itself (safe for keys an attacker picks).
    Blake2_128Concat,
    /// 64-bit xxHash of the key, then the key itself (fast; only for keys that are not attacker-chosen).
    Twox64Concat,
    /// The key as it is.
    Identity,
}

impl Hasher {
    /// Appends the hashed form of `key`.
    pub fn hash_to(self, key: &[u8], out: &mut Vec<u8>) {
        match self {
            Hasher::Blake2_128Concat => {
                out.extend_from_slice(&blake2_128(key));
                out.extend_from_slice(key);
            }
            Hasher::Twox64Concat => {
                out.extend_from_slice(&twox_64(key));
                out.extend_from_slice(key);
            }
            Hasher::Identity => out.extend_from_slice(key),
        }
    }

    /// The key that was hashed, if the hasher keeps it (the "Concat" ones do).
    pub fn recover(self, hashed: &[u8]) -> Option<&[u8]> {
        match self {
            Hasher::Blake2_128Concat => hashed.get(16..),
            Hasher::Twox64Concat => hashed.get(8..),
            Hasher::Identity => Some(hashed),
        }
    }
}

/// The 32-byte prefix of a storage item: `twox128(pallet) ++ twox128(item)`.
pub fn storage_prefix(pallet: &str, item: &str) -> [u8; 32] {
    let mut out = [0u8; 32];
    out[..16].copy_from_slice(&twox_128(pallet.as_bytes()));
    out[16..].copy_from_slice(&twox_128(item.as_bytes()));
    out
}

/// The key of one entry of a map item.
pub fn map_key(pallet: &str, item: &str, hasher: Hasher, key: &[u8]) -> Vec<u8> {
    let mut out = storage_prefix(pallet, item).to_vec();
    hasher.hash_to(key, &mut out);
    out
}
```

### `src/ss58.rs`

```rust
//! SS58: the checksummed base-58 text form of a 32-byte account id, with a
//! network prefix (42 is the generic Substrate prefix, which gives addresses
//! starting with `5`).

use crate::hashing::blake2_512;
use solanalab::base58;

/// Why an address was rejected.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Ss58Error {
    /// Not valid base-58.
    BadBase58,
    /// Wrong length for a 32-byte account id with a 1- or 2-byte prefix.
    BadLength,
    /// The prefix is reserved or malformed.
    BadPrefix,
    /// The checksum does not match.
    BadChecksum,
}

/// Largest prefix that fits in the two-byte form.
pub const MAX_PREFIX: u16 = 16383;

fn prefix_bytes(prefix: u16) -> Result<Vec<u8>, Ss58Error> {
    match prefix {
        0..=63 => Ok(vec![prefix as u8]),
        64..=MAX_PREFIX => {
            let first = (((prefix & 0b0000_0000_1111_1100) >> 2) as u8) | 0b0100_0000;
            let second = ((prefix >> 8) as u8) | (((prefix & 0b11) as u8) << 6);
            Ok(vec![first, second])
        }
        _ => Err(Ss58Error::BadPrefix),
    }
}

fn checksum(prefix: &[u8], id: &[u8; 32]) -> [u8; 2] {
    let mut data = b"SS58PRE".to_vec();
    data.extend_from_slice(prefix);
    data.extend_from_slice(id);
    let h = blake2_512(&data);
    [h[0], h[1]]
}

/// The address text of `id` on the network with `prefix`.
pub fn encode(prefix: u16, id: &[u8; 32]) -> Result<String, Ss58Error> {
    let p = prefix_bytes(prefix)?;
    let mut raw = p.clone();
    raw.extend_from_slice(id);
    raw.extend_from_slice(&checksum(&p, id));
    Ok(base58::encode(&raw))
}

/// Reads an address back into its prefix and account id.
pub fn decode(text: &str) -> Result<(u16, [u8; 32]), Ss58Error> {
    let raw = base58::decode(text).map_err(|_| Ss58Error::BadBase58)?;
    let (prefix_len, prefix) = match raw.first() {
        Some(&b) if b < 64 => (1, u16::from(b)),
        Some(&b) if b < 128 => {
            let second = *raw.get(1).ok_or(Ss58Error::BadLength)?;
            // the shift is on a u8 on purpose: the 0b01 marker bits fall off the top
            let lower = u16::from(b << 2) | u16::from(second >> 6);
            let upper = u16::from(second) & 0b0011_1111;
            (2, lower | (upper << 8))
        }
        Some(_) => return Err(Ss58Error::BadPrefix),
        None => return Err(Ss58Error::BadLength),
    };
    if raw.len() != prefix_len + 32 + 2 {
        return Err(Ss58Error::BadLength);
    }
    // a two-byte form is only for prefixes that need it, and the reserved ones are refused
    if prefix_len == 2 && prefix < 64 || prefix == 46 || prefix == 47 {
        return Err(Ss58Error::BadPrefix);
    }
    let id: [u8; 32] = raw[prefix_len..prefix_len + 32].try_into().expect("32");
    if raw[prefix_len + 32..] != checksum(&raw[..prefix_len], &id) {
        return Err(Ss58Error::BadChecksum);
    }
    Ok((prefix, id))
}
```

### `src/extrinsic.rs`

```rust
//! The extrinsic: a SCALE-encoded, length-prefixed, versioned transaction.
//! Signed extrinsics carry an ed25519 signature over a payload that also
//! binds the chain (spec version, tx version, genesis hash).

use crate::hashing::blake2_256;
use crate::scale::{Compact, Decode, DecodeError, Encode};
use ed25519_dalek::{Signature, Signer, SigningKey, Verifier, VerifyingKey};

/// Version byte: top bit = signed, low bits = format version 4.
pub const V4_SIGNED: u8 = 0x84;
pub const V4_UNSIGNED: u8 = 0x04;

/// The calls this teaching runtime understands.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Call {
    /// pallet 0, call 0: store nothing, just record a remark.
    Remark(Vec<u8>),
    /// pallet 1, call 0: move `value` to `dest`.
    Transfer { dest: [u8; 32], value: u128 },
}

impl Encode for Call {
    fn encode_to(&self, out: &mut Vec<u8>) {
        match self {
            Call::Remark(r) => {
                out.extend_from_slice(&[0, 0]);
                r.encode_to(out);
            }
            Call::Transfer { dest, value } => {
                out.extend_from_slice(&[1, 0]);
                out.push(0); // MultiAddress::Id
                dest.encode_to(out);
                Compact(*value).encode_to(out);
            }
        }
    }
}

impl Decode for Call {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        let pallet = u8::decode(input)?;
        let call = u8::decode(input)?;
        match (pallet, call) {
            (0, 0) => Ok(Call::Remark(Vec::<u8>::decode(input)?)),
            (1, 0) => {
                let tag = u8::decode(input)?;
                if tag != 0 {
                    return Err(DecodeError::InvalidTag(tag));
                }
                let dest = <[u8; 32]>::decode(input)?;
                let Compact(value) = Compact::decode(input)?;
                Ok(Call::Transfer { dest, value })
            }
            (p, _) => Err(DecodeError::InvalidTag(p)),
        }
    }
}

/// The part of a signed extrinsic that is both sent and signed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Extra {
    pub nonce: u64,
    pub tip: u128,
}

/// What the signer binds that is not sent: the chain it is for.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct ChainContext {
    pub spec_version: u32,
    pub tx_version: u32,
    pub genesis: [u8; 32],
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Signed {
    pub signer: [u8; 32],
    pub signature: [u8; 64],
    pub extra: Extra,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Extrinsic {
    pub signed: Option<Signed>,
    pub call: Call,
}

fn encode_extra(e: &Extra, out: &mut Vec<u8>) {
    out.push(0); // era: immortal
    Compact(u128::from(e.nonce)).encode_to(out);
    Compact(e.tip).encode_to(out);
}

/// The bytes that get signed. Payloads over 256 bytes are hashed first.
pub fn signing_payload(call: &Call, extra: &Extra, ctx: &ChainContext) -> Vec<u8> {
    let mut p = call.encode();
    encode_extra(extra, &mut p);
    ctx.spec_version.encode_to(&mut p);
    ctx.tx_version.encode_to(&mut p);
    p.extend_from_slice(&ctx.genesis);
    p.extend_from_slice(&ctx.genesis); // block hash: immortal era uses genesis
    if p.len() > 256 {
        blake2_256(&p).to_vec()
    } else {
        p
    }
}

impl Extrinsic {
    /// Signs `call` with a deterministic key from `seed`.
    pub fn sign(seed: [u8; 32], call: Call, extra: Extra, ctx: &ChainContext) -> Extrinsic {
        let key = SigningKey::from_bytes(&seed);
        let payload = signing_payload(&call, &extra, ctx);
        let sig = key.sign(&payload);
        Extrinsic {
            signed: Some(Signed {
                signer: key.verifying_key().to_bytes(),
                signature: sig.to_bytes(),
                extra,
            }),
            call,
        }
    }

    pub fn unsigned(call: Call) -> Extrinsic {
        Extrinsic { signed: None, call }
    }

    /// Checks the signature against the chain context. Unsigned: `true`.
    pub fn verify(&self, ctx: &ChainContext) -> bool {
        let Some(s) = &self.signed else { return true };
        let Ok(vk) = VerifyingKey::from_bytes(&s.signer) else {
            return false;
        };
        let payload = signing_payload(&self.call, &s.extra, ctx);
        vk.verify(&payload, &Signature::from_bytes(&s.signature))
            .is_ok()
    }

    /// The wire form: compact length, then the body.
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut body = Vec::new();
        match &self.signed {
            Some(s) => {
                body.push(V4_SIGNED);
                body.push(0); // MultiAddress::Id
                body.extend_from_slice(&s.signer);
                body.push(0); // MultiSignature::Ed25519
                body.extend_from_slice(&s.signature);
                encode_extra(&s.extra, &mut body);
            }
            None => body.push(V4_UNSIGNED),
        }
        self.call.encode_to(&mut body);
        let mut out = Compact(body.len() as u128).encode();
        out.extend_from_slice(&body);
        out
    }

    /// Parses the wire form; the length prefix must match exactly.
    pub fn from_bytes(bytes: &[u8]) -> Result<Extrinsic, DecodeError> {
        let mut input = bytes;
        let Compact(len) = Compact::decode(&mut input)?;
        if len != input.len() as u128 {
            return Err(if len > input.len() as u128 {
                DecodeError::Eof
            } else {
                DecodeError::Trailing
            });
        }
        let version = u8::decode(&mut input)?;
        let signed = match version {
            V4_UNSIGNED => None,
            V4_SIGNED => {
                let tag = u8::decode(&mut input)?;
                if tag != 0 {
                    return Err(DecodeError::InvalidTag(tag));
                }
                let signer = <[u8; 32]>::decode(&mut input)?;
                let sig_tag = u8::decode(&mut input)?;
                if sig_tag != 0 {
                    return Err(DecodeError::InvalidTag(sig_tag));
                }
                let signature = <[u8; 64]>::decode(&mut input)?;
                let era = u8::decode(&mut input)?;
                if era != 0 {
                    return Err(DecodeError::InvalidTag(era));
                }
                let Compact(nonce) = Compact::decode(&mut input)?;
                let nonce = u64::try_from(nonce).map_err(|_| DecodeError::Overflow)?;
                let Compact(tip) = Compact::decode(&mut input)?;
                Some(Signed {
                    signer,
                    signature,
                    extra: Extra { nonce, tip },
                })
            }
            other => return Err(DecodeError::InvalidTag(other)),
        };
        let call = Call::decode(&mut input)?;
        if !input.is_empty() {
            return Err(DecodeError::Trailing);
        }
        Ok(Extrinsic { signed, call })
    }

    /// The extrinsic hash (what explorers show): blake2-256 of the wire bytes.
    pub fn hash(&self) -> [u8; 32] {
        blake2_256(&self.to_bytes())
    }
}
```

### `src/runtime.rs`

```rust
//! A tiny chain: SCALE-encoded storage under Substrate-style keys, a
//! transaction pool, and a block builder. The state root here is a plain hash
//! over sorted entries, NOT Substrate's Patricia-Merkle trie.

use crate::extrinsic::{Call, ChainContext, Extrinsic};
use crate::hashing::{Hasher, blake2_256, map_key, storage_prefix};
use crate::scale::{Compact, Decode, DecodeError, Encode, decode_all};
use std::collections::BTreeMap;

/// Existential deposit: an account below this is reaped.
pub const EXISTENTIAL_DEPOSIT: u128 = 10;
/// Flat fee per included signed extrinsic (burned).
pub const BASE_FEE: u128 = 1;

/// `System.Account` value.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub struct AccountInfo {
    pub nonce: u32,
    pub free: u128,
}

impl Encode for AccountInfo {
    fn encode_to(&self, out: &mut Vec<u8>) {
        self.nonce.encode_to(out);
        self.free.encode_to(out);
    }
}
impl Decode for AccountInfo {
    fn decode(input: &mut &[u8]) -> Result<Self, DecodeError> {
        Ok(AccountInfo {
            nonce: u32::decode(input)?,
            free: u128::decode(input)?,
        })
    }
}

/// Why an extrinsic cannot even be included (the pool rejects it).
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Invalid {
    BadEncoding,
    BadSignature,
    Unsigned,
    Stale,
    Future,
    CannotPayFee,
}

/// Why a dispatched call failed (it is still included; the fee is still paid).
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DispatchError {
    InsufficientBalance,
    BelowExistentialDeposit,
    SelfTransfer,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Event {
    Transfer {
        from: [u8; 32],
        to: [u8; 32],
        value: u128,
    },
    Remark {
        who: [u8; 32],
        len: u32,
    },
    Failed {
        who: [u8; 32],
        error: DispatchError,
    },
    Reaped {
        who: [u8; 32],
        dust: u128,
    },
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Header {
    pub parent_hash: [u8; 32],
    pub number: u64,
    pub state_root: [u8; 32],
    pub extrinsics_root: [u8; 32],
}

impl Header {
    pub fn encode(&self) -> Vec<u8> {
        let mut o = Vec::new();
        o.extend_from_slice(&self.parent_hash);
        Compact(u128::from(self.number)).encode_to(&mut o);
        o.extend_from_slice(&self.state_root);
        o.extend_from_slice(&self.extrinsics_root);
        o
    }
    pub fn hash(&self) -> [u8; 32] {
        blake2_256(&self.encode())
    }
}

#[derive(Debug, Clone)]
pub struct Block {
    pub header: Header,
    pub extrinsics: Vec<Vec<u8>>,
    pub events: Vec<Event>,
}

pub struct Chain {
    storage: BTreeMap<Vec<u8>, Vec<u8>>,
    pub ctx: ChainContext,
    pool: Vec<Extrinsic>,
    blocks: Vec<Block>,
}

pub fn account_key(who: &[u8; 32]) -> Vec<u8> {
    map_key("System", "Account", Hasher::Blake2_128Concat, who)
}
pub fn total_issuance_key() -> Vec<u8> {
    storage_prefix("Balances", "TotalIssuance").to_vec()
}

fn extrinsics_root(xts: &[Vec<u8>]) -> [u8; 32] {
    blake2_256(&xts.to_vec().encode())
}

impl Chain {
    /// A chain whose genesis endows the given accounts.
    pub fn new(endowed: &[([u8; 32], u128)]) -> Chain {
        let mut storage = BTreeMap::new();
        let mut total = 0u128;
        for (who, amount) in endowed {
            let info = AccountInfo {
                nonce: 0,
                free: *amount,
            };
            storage.insert(account_key(who), info.encode());
            total += amount;
        }
        storage.insert(total_issuance_key(), total.encode());
        let mut c = Chain {
            storage,
            ctx: ChainContext {
                spec_version: 100,
                tx_version: 1,
                genesis: [0; 32],
            },
            pool: Vec::new(),
            blocks: Vec::new(),
        };
        let header = Header {
            parent_hash: [0; 32],
            number: 0,
            state_root: c.state_root(),
            extrinsics_root: extrinsics_root(&[]),
        };
        c.ctx.genesis = header.hash();
        c.blocks.push(Block {
            header,
            extrinsics: vec![],
            events: vec![],
        });
        c
    }

    pub fn state_root(&self) -> [u8; 32] {
        let mut buf = Vec::new();
        for (k, v) in &self.storage {
            buf.extend_from_slice(&k.encode());
            buf.extend_from_slice(&v.encode());
        }
        blake2_256(&buf)
    }

    pub fn storage(&self, key: &[u8]) -> Option<&Vec<u8>> {
        self.storage.get(key)
    }

    pub fn account(&self, who: &[u8; 32]) -> AccountInfo {
        self.storage
            .get(&account_key(who))
            .and_then(|v| decode_all::<AccountInfo>(v).ok())
            .unwrap_or_default()
    }

    pub fn total_issuance(&self) -> u128 {
        self.storage
            .get(&total_issuance_key())
            .and_then(|v| decode_all::<u128>(v).ok())
            .unwrap_or(0)
    }

    /// Sum of all account balances, read from storage.
    pub fn sum_balances(&self) -> u128 {
        let prefix = storage_prefix("System", "Account");
        self.storage
            .iter()
            .filter(|(k, _)| k.starts_with(&prefix))
            .filter_map(|(_, v)| decode_all::<AccountInfo>(v).ok())
            .map(|a| a.free)
            .sum()
    }

    pub fn best(&self) -> &Block {
        self.blocks.last().expect("genesis")
    }
    pub fn block(&self, n: u64) -> Option<&Block> {
        self.blocks.get(usize::try_from(n).ok()?)
    }
    pub fn pool_len(&self) -> usize {
        self.pool.len()
    }

    /// Next nonce counting pending pool transactions (like `system_accountNextIndex`).
    pub fn next_nonce(&self, who: &[u8; 32]) -> u64 {
        let mut n = u64::from(self.account(who).nonce);
        for x in &self.pool {
            if let Some(s) = x
                .signed
                .as_ref()
                .filter(|s| &s.signer == who && s.extra.nonce == n)
            {
                n = s.extra.nonce + 1;
            }
        }
        n
    }

    /// Validates and queues an extrinsic. Returns its hash.
    pub fn submit(&mut self, bytes: &[u8]) -> Result<[u8; 32], Invalid> {
        let xt = Extrinsic::from_bytes(bytes).map_err(|_| Invalid::BadEncoding)?;
        let s = xt.signed.as_ref().ok_or(Invalid::Unsigned)?;
        if !xt.verify(&self.ctx) {
            return Err(Invalid::BadSignature);
        }
        let expected = self.next_nonce(&s.signer);
        if s.extra.nonce < expected {
            return Err(Invalid::Stale);
        }
        if s.extra.nonce > expected {
            return Err(Invalid::Future);
        }
        if self.account(&s.signer).free < BASE_FEE + s.extra.tip {
            return Err(Invalid::CannotPayFee);
        }
        let h = xt.hash();
        self.pool.push(xt);
        Ok(h)
    }

    fn put_account(&mut self, who: &[u8; 32], info: AccountInfo) {
        self.storage.insert(account_key(who), info.encode());
    }

    fn burn(&mut self, amount: u128) {
        let t = self.total_issuance() - amount;
        self.storage.insert(total_issuance_key(), t.encode());
    }

    /// Applies one extrinsic. `Err` means invalid at execution time (not included).
    fn apply(&mut self, xt: &Extrinsic, events: &mut Vec<Event>) -> Result<(), Invalid> {
        let s = xt.signed.as_ref().ok_or(Invalid::Unsigned)?;
        let who = s.signer;
        let mut acc = self.account(&who);
        if u64::from(acc.nonce) != s.extra.nonce {
            return Err(if s.extra.nonce < u64::from(acc.nonce) {
                Invalid::Stale
            } else {
                Invalid::Future
            });
        }
        let fee = BASE_FEE + s.extra.tip;
        if acc.free < fee {
            return Err(Invalid::CannotPayFee);
        }
        // fee and nonce are committed before the call runs
        acc.free -= fee;
        acc.nonce += 1;
        self.burn(fee);
        self.put_account(&who, acc);
        let result = self.dispatch(&who, &xt.call, events);
        if let Err(error) = result {
            events.push(Event::Failed { who, error });
        }
        self.reap(&who, events);
        Ok(())
    }

    fn dispatch(
        &mut self,
        who: &[u8; 32],
        call: &Call,
        events: &mut Vec<Event>,
    ) -> Result<(), DispatchError> {
        match call {
            Call::Remark(r) => {
                events.push(Event::Remark {
                    who: *who,
                    len: r.len() as u32,
                });
                Ok(())
            }
            Call::Transfer { dest, value } => {
                if dest == who {
                    return Err(DispatchError::SelfTransfer);
                }
                let mut from = self.account(who);
                if from.free < *value {
                    return Err(DispatchError::InsufficientBalance);
                }
                let mut to = self.account(dest);
                if to.free + value < EXISTENTIAL_DEPOSIT {
                    return Err(DispatchError::BelowExistentialDeposit);
                }
                from.free -= value;
                to.free += value;
                self.put_account(who, from);
                self.put_account(dest, to);
                events.push(Event::Transfer {
                    from: *who,
                    to: *dest,
                    value: *value,
                });
                Ok(())
            }
        }
    }

    /// Accounts under the existential deposit are removed; the dust is burned.
    fn reap(&mut self, who: &[u8; 32], events: &mut Vec<Event>) {
        let acc = self.account(who);
        if acc.free < EXISTENTIAL_DEPOSIT {
            self.storage.remove(&account_key(who));
            if acc.free > 0 {
                self.burn(acc.free);
            }
            events.push(Event::Reaped {
                who: *who,
                dust: acc.free,
            });
        }
    }

    /// Builds the next block from the pool, in order.
    pub fn produce_block(&mut self) -> &Block {
        let pool = std::mem::take(&mut self.pool);
        let mut included = Vec::new();
        let mut events = Vec::new();
        for xt in pool {
            if self.apply(&xt, &mut events).is_ok() {
                included.push(xt.to_bytes());
            }
        }
        let parent = self.best().header.clone();
        let header = Header {
            parent_hash: parent.hash(),
            number: parent.number + 1,
            state_root: self.state_root(),
            extrinsics_root: extrinsics_root(&included),
        };
        self.blocks.push(Block {
            header,
            extrinsics: included,
            events,
        });
        self.best()
    }
}
```

### `src/rpc.rs`

```rust
//! A JSON-RPC 2.0 front end over the in-process chain, shaped like the
//! legacy Substrate methods (`state_getStorage`, `author_submitExtrinsic`, ...).
//! Method names and result shapes are from memory of the real node.

use crate::runtime::{Chain, Invalid};
use crate::ss58;
use serde_json::{Value, json};

pub const PARSE_ERROR: i64 = -32700;
pub const INVALID_REQUEST: i64 = -32600;
pub const METHOD_NOT_FOUND: i64 = -32601;
pub const INVALID_PARAMS: i64 = -32602;
/// Substrate uses 1010 for "invalid transaction" in `author_submitExtrinsic`.
pub const INVALID_TX: i64 = 1010;

pub fn to_hex(b: &[u8]) -> String {
    format!("0x{}", hex::encode(b))
}

pub fn from_hex(s: &str) -> Option<Vec<u8>> {
    hex::decode(s.strip_prefix("0x")?).ok()
}

fn err(id: Value, code: i64, message: &str) -> Value {
    json!({"jsonrpc":"2.0","id":id,"error":{"code":code,"message":message}})
}

fn ok(id: Value, result: Value) -> Value {
    json!({"jsonrpc":"2.0","id":id,"result":result})
}

fn invalid_name(i: Invalid) -> &'static str {
    match i {
        Invalid::BadEncoding => "BadEncoding",
        Invalid::BadSignature => "BadProof",
        Invalid::Unsigned => "UnsignedNotAllowed",
        Invalid::Stale => "Stale",
        Invalid::Future => "Future",
        Invalid::CannotPayFee => "Payment",
    }
}

fn header_json(h: &crate::runtime::Header) -> Value {
    json!({
        "parentHash": to_hex(&h.parent_hash),
        "number": format!("0x{:x}", h.number),
        "stateRoot": to_hex(&h.state_root),
        "extrinsicsRoot": to_hex(&h.extrinsics_root),
    })
}

/// Handles one request text and returns the response text.
pub fn handle(chain: &mut Chain, request: &str) -> String {
    let v: Value = match serde_json::from_str(request) {
        Ok(v) => v,
        Err(_) => return err(Value::Null, PARSE_ERROR, "Parse error").to_string(),
    };
    let id = v.get("id").cloned().unwrap_or(Value::Null);
    let Some(method) = v.get("method").and_then(Value::as_str) else {
        return err(id, INVALID_REQUEST, "Invalid Request").to_string();
    };
    if v.get("jsonrpc") != Some(&json!("2.0")) {
        return err(id, INVALID_REQUEST, "Invalid Request").to_string();
    }
    let params = v.get("params").cloned().unwrap_or_else(|| json!([]));
    let Some(params) = params.as_array() else {
        return err(id, INVALID_PARAMS, "Invalid params").to_string();
    };
    call(chain, method, params, id).to_string()
}

fn call(chain: &mut Chain, method: &str, params: &[Value], id: Value) -> Value {
    let bad = |id: Value| err(id, INVALID_PARAMS, "Invalid params");
    match method {
        "state_getStorage" => {
            let Some(key) = params.first().and_then(Value::as_str).and_then(from_hex) else {
                return bad(id);
            };
            match chain.storage(&key) {
                Some(v) => ok(id, json!(to_hex(v))),
                None => ok(id, Value::Null),
            }
        }
        "author_submitExtrinsic" => {
            let Some(bytes) = params.first().and_then(Value::as_str).and_then(from_hex) else {
                return bad(id);
            };
            match chain.submit(&bytes) {
                Ok(h) => ok(id, json!(to_hex(&h))),
                Err(e) => err(
                    id,
                    INVALID_TX,
                    &format!("Invalid Transaction: {}", invalid_name(e)),
                ),
            }
        }
        "chain_getHeader" => {
            let h = match params.first() {
                None | Some(Value::Null) => chain.best().header.clone(),
                Some(Value::String(s)) => {
                    let Some(hash) = from_hex(s) else {
                        return bad(id);
                    };
                    match (0..)
                        .map_while(|n| chain.block(n))
                        .find(|b| b.header.hash().as_slice() == hash)
                    {
                        Some(b) => b.header.clone(),
                        None => return ok(id, Value::Null),
                    }
                }
                _ => return bad(id),
            };
            ok(id, header_json(&h))
        }
        "chain_getBlockHash" => {
            let n = match params.first() {
                None | Some(Value::Null) => chain.best().header.number,
                Some(Value::Number(n)) => match n.as_u64() {
                    Some(n) => n,
                    None => return bad(id),
                },
                _ => return bad(id),
            };
            match chain.block(n) {
                Some(b) => ok(id, json!(to_hex(&b.header.hash()))),
                None => ok(id, Value::Null),
            }
        }
        "system_accountNextIndex" => {
            let Some((_, who)) = params
                .first()
                .and_then(Value::as_str)
                .and_then(|s| ss58::decode(s).ok())
            else {
                return bad(id);
            };
            ok(id, json!(chain.next_nonce(&who)))
        }
        "system_chain" => ok(id, json!("Teaching Dev")),
        "dev_produceBlock" => {
            let b = chain.produce_block();
            ok(
                id,
                json!({"number": b.header.number, "extrinsics": b.extrinsics.len()}),
            )
        }
        _ => err(id, METHOD_NOT_FOUND, "Method not found"),
    }
}
```

### `tests/common.rs`

```rust
#![allow(dead_code)]
pub struct Rng(pub u64);
impl Rng {
    pub fn next_u64(&mut self) -> u64 {
        self.0 ^= self.0 << 13;
        self.0 ^= self.0 >> 7;
        self.0 ^= self.0 << 17;
        self.0
    }
    pub fn below(&mut self, n: u64) -> u64 {
        self.next_u64() % n
    }
    pub fn bytes(&mut self, n: usize) -> Vec<u8> {
        (0..n).map(|_| self.next_u64() as u8).collect()
    }
}
/// Compact values with interesting magnitudes.
pub fn interesting(r: &mut Rng) -> u128 {
    let bits = r.below(129) as u32;
    let v = (u128::from(r.next_u64()) << 64) | u128::from(r.next_u64());
    if bits == 0 {
        0
    } else if bits >= 128 {
        v
    } else {
        v >> (128 - bits)
    }
}
```

### `tests/scale_tests.rs`

```rust
mod common;
use common::*;
use parity_scale_codec::{Compact as PC, Decode as PD, Encode as PE};
use substratelab::scale::*;

fn enc<T: Encode>(t: &T) -> Vec<u8> {
    t.encode()
}

#[test]
fn compact_known_answers_from_the_spec() {
    let cases: &[(u128, &str)] = &[
        (0, "00"),
        (1, "04"),
        (42, "a8"),
        (63, "fc"),
        (64, "0101"),
        (69, "1501"),
        (16383, "fdff"),
        (16384, "02000100"),
        (65535, "feff0300"),
        ((1 << 30) - 1, "feffffff"),
        (1 << 30, "0300000040"),
        (u128::from(u32::MAX), "03ffffffff"),
        (1 << 32, "070000000001"),
        (u128::MAX, "33ffffffffffffffffffffffffffffffff"),
    ];
    for (v, h) in cases {
        assert_eq!(hex::encode(Compact(*v).encode()), *h, "value {v}");
        assert_eq!(
            decode_all::<Compact>(&hex::decode(h).unwrap()),
            Ok(Compact(*v))
        );
    }
}

#[test]
fn compact_matches_parity_codec_on_random_values() {
    let mut r = Rng(0x1234_5678_9abc_def1);
    for _ in 0..3000 {
        let v = interesting(&mut r);
        assert_eq!(Compact(v).encode(), PC(v).encode(), "value {v}");
        let bytes = PC(v).encode();
        assert_eq!(decode_all::<Compact>(&bytes), Ok(Compact(v)));
    }
}

#[test]
fn compact_rejects_non_canonical_forms() {
    // 1 in two-byte mode, four-byte mode, and big-integer mode
    for h in [
        "0100",
        "05000000",
        "0301000000",
        "070100000000",
        "0b010000000000",
        "0f01000000000000",
    ] {
        let b = hex::decode(h).unwrap();
        assert!(
            matches!(decode_all::<Compact>(&b), Err(DecodeError::NonCanonical)),
            "{h}"
        );
    }
    // big mode with a trailing zero byte (value fits in fewer bytes)
    assert_eq!(
        decode_all::<Compact>(&hex::decode("070000004000").unwrap()),
        Err(DecodeError::NonCanonical)
    );
}

#[test]
fn compact_big_mode_length_limits() {
    // n = 17 bytes -> first byte ((17-4)<<2)|3 = 0x37
    let mut b = vec![0x37];
    b.extend_from_slice(&[1; 17]);
    assert_eq!(decode_all::<Compact>(&b), Err(DecodeError::Overflow));
}

#[test]
fn compact_truncated_is_eof() {
    for h in ["01", "02", "0300", "07000000"] {
        assert_eq!(
            decode_all::<Compact>(&hex::decode(h).unwrap()),
            Err(DecodeError::Eof),
            "{h}"
        );
    }
    assert_eq!(decode_all::<Compact>(&[]), Err(DecodeError::Eof));
}

#[test]
fn integers_are_little_endian() {
    assert_eq!(enc(&0x0102u16), [2, 1]);
    assert_eq!(enc(&0x01020304u32), [4, 3, 2, 1]);
    assert_eq!(enc(&1u64), [1, 0, 0, 0, 0, 0, 0, 0]);
    assert_eq!(enc(&1u128).len(), 16);
    assert_eq!(enc(&7u8), [7]);
    assert_eq!(decode_all::<u32>(&[4, 3, 2, 1]), Ok(0x01020304));
}

#[test]
fn bool_option_and_strings() {
    assert_eq!(enc(&true), [1]);
    assert_eq!(enc(&false), [0]);
    assert_eq!(decode_all::<bool>(&[2]), Err(DecodeError::InvalidBool(2)));
    assert_eq!(enc(&None::<u8>), [0]);
    assert_eq!(enc(&Some(5u8)), [1, 5]);
    assert_eq!(
        decode_all::<Option<u8>>(&[2, 5]),
        Err(DecodeError::InvalidTag(2))
    );
    assert_eq!(enc(&"hi".to_string()), [8, b'h', b'i']);
    assert_eq!(decode_all::<String>(&[4, 0xff]), Err(DecodeError::Utf8));
}

#[test]
fn vec_known_answer_and_hostile_length() {
    assert_eq!(hex::encode(enc(&vec![1u16, 2, 3])), "0c010002000300");
    // claims 2^30 elements but supplies none: refused before allocating
    let b = hex::decode("0300000040").unwrap();
    assert_eq!(decode_all::<Vec<u8>>(&b), Err(DecodeError::TooLong));
    assert_eq!(decode_all::<Vec<u64>>(&b), Err(DecodeError::TooLong));
}

#[test]
fn trailing_bytes_are_refused() {
    assert_eq!(decode_all::<u8>(&[1, 2]), Err(DecodeError::Trailing));
    assert_eq!(decode_all::<u32>(&[1, 2, 3]), Err(DecodeError::Eof));
}

#[test]
fn nested_values_agree_with_parity_codec() {
    let mut r = Rng(99);
    for _ in 0..500 {
        let n = r.below(40) as usize;
        let v: Vec<(u32, Option<Vec<u8>>)> = (0..n)
            .map(|_| {
                let o = if r.below(2) == 0 {
                    None
                } else {
                    {
                        let l = r.below(100) as usize;
                        Some(r.bytes(l))
                    }
                };
                (r.next_u64() as u32, o)
            })
            .collect();
        let ours = enc(&v);
        assert_eq!(ours, PE::encode(&v));
        assert_eq!(
            decode_all::<Vec<(u32, Option<Vec<u8>>)>>(&ours),
            Ok(v.clone())
        );
        let theirs = <Vec<(u32, Option<Vec<u8>>)> as PD>::decode(&mut &ours[..]).unwrap();
        assert_eq!(theirs, v);
    }
}

#[test]
fn decoding_garbage_never_panics_and_agrees_on_validity_of_compact() {
    let mut r = Rng(7);
    for _ in 0..4000 {
        let len = r.below(20) as usize;
        let b = r.bytes(len);
        let ours = decode_all::<Compact>(&b);
        // parity accepts exactly what we accept, when the input is fully consumed
        let mut s = &b[..];
        let theirs = <PC<u128> as PD>::decode(&mut s);
        if let Ok(Compact(v)) = ours {
            assert_eq!(theirs.map(|c| c.0).ok(), Some(v));
            assert!(s.is_empty());
        }
        let _ = decode_all::<Vec<String>>(&b);
        let _ = decode_all::<(Option<u16>, Vec<u8>)>(&b);
    }
}

#[test]
fn array_and_tuple() {
    assert_eq!(enc(&[1u8, 2, 3]), [1, 2, 3]);
    assert_eq!(enc(&(1u8, 2u16)), [1, 2, 0]);
    assert_eq!(decode_all::<(u8, u16)>(&[1, 2, 0]), Ok((1, 2)));
}
```

### `tests/hashing_tests.rs`

```rust
use substratelab::hashing::*;

fn h(x: [u8; 32]) -> String {
    hex::encode(x)
}

#[test]
fn blake2_known_answers() {
    assert_eq!(
        h(blake2_256(b"")),
        "0e5751c026e543b2e8ab2eb06099daa1d1e5df47778f7787faab45cdf12fe3a8"
    );
    assert_eq!(blake2_128(b"").len(), 16);
    assert_eq!(blake2_512(b"").len(), 64);
    assert_ne!(blake2_256(b"a"), blake2_256(b"b"));
}

#[test]
fn well_known_storage_prefixes() {
    assert_eq!(
        hex::encode(storage_prefix("System", "Account")),
        "26aa394eea5630e07c48ae0c9558cef7b99d880ec681799c0cf30e8886371da9"
    );
    assert_eq!(
        hex::encode(storage_prefix("Balances", "TotalIssuance")),
        "c2261276cc9d1f8598ea4b6a74b15c2f57c875e4cff74148e4628f264b974c80"
    );
    assert_eq!(
        hex::encode(storage_prefix("Timestamp", "Now")),
        "f0c365c3cf59d671eb72da0e7a4113c49f1f0515f462cdcf84e0f1d6045dfcbb"
    );
}

#[test]
fn twox_hashes_are_little_endian_xxh64() {
    assert_eq!(
        hex::encode(twox_128(b"System")),
        "26aa394eea5630e07c48ae0c9558cef7"
    );
    assert_eq!(twox_64(b"System"), twox_128(b"System")[..8]);
    assert_ne!(twox_128(b"System")[..8], twox_128(b"System")[8..]);
}

#[test]
fn hasher_concat_can_recover_keys_identity_too() {
    let key = [9u8; 32];
    for hs in [
        Hasher::Blake2_128Concat,
        Hasher::Twox64Concat,
        Hasher::Identity,
    ] {
        let mut out = Vec::new();
        hs.hash_to(&key, &mut out);
        assert_eq!(hs.recover(&out), Some(&key[..]));
    }
    let mut out = Vec::new();
    Hasher::Blake2_128Concat.hash_to(&key, &mut out);
    assert_eq!(out.len(), 16 + 32);
    assert_eq!(&out[..16], &blake2_128(&key));
    let mut out = Vec::new();
    Hasher::Twox64Concat.hash_to(&key, &mut out);
    assert_eq!(out.len(), 8 + 32);
    assert_eq!(&out[..8], &twox_64(&key));
    let mut out = Vec::new();
    Hasher::Identity.hash_to(&key, &mut out);
    assert_eq!(out, key);
}

#[test]
fn recover_rejects_short_input() {
    assert_eq!(Hasher::Blake2_128Concat.recover(&[0; 15]), None);
    assert_eq!(Hasher::Twox64Concat.recover(&[0; 7]), None);
    assert_eq!(Hasher::Blake2_128Concat.recover(&[0; 16]), Some(&[][..]));
    assert_eq!(Hasher::Twox64Concat.recover(&[0; 8]), Some(&[][..]));
}

#[test]
fn map_key_layout() {
    let k = map_key("System", "Account", Hasher::Blake2_128Concat, &[1; 32]);
    assert_eq!(&k[..32], &storage_prefix("System", "Account"));
    assert_eq!(k.len(), 32 + 16 + 32);
    assert_eq!(&k[32..48], &blake2_128(&[1; 32]));
    assert_eq!(&k[48..], &[1; 32]);
}
```

### `tests/ss58_tests.rs`

```rust
use substratelab::ss58::*;

const ALICE: &str = "d43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d";

fn alice() -> [u8; 32] {
    hex::decode(ALICE).unwrap().try_into().unwrap()
}

#[test]
fn known_addresses_computed_independently() {
    assert_eq!(
        encode(42, &alice()).unwrap(),
        "5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY"
    );
    assert_eq!(
        encode(0, &alice()).unwrap(),
        "15oF4uVJwmo4TdGW7VfQxNLavjCXviqxT9S1MgbjMNHr6Sp5"
    );
    assert_eq!(
        encode(2, &alice()).unwrap(),
        "HNZata7iMYWmk5RvZRTiAsSDhV8366zq2YGb3tLH5Upf74F"
    );
    assert_eq!(
        encode(42, &[0; 32]).unwrap(),
        "5C4hrfjw9DjXZTzV3MwzrrAr9P1MJhSrvWGWqi1eSuyUpnhM"
    );
    assert_eq!(
        encode(42, &[1; 32]).unwrap(),
        "5C62Ck4UrFPiBtoCmeSrgF7x9yv9mn38446dhCpsi2mLHiFT"
    );
    assert_eq!(
        encode(100, &alice()).unwrap(),
        "gCUAENmXKSahAy2XpHuCaKsgLA5LAQM6tDTVUpbVgmVyaFrWC"
    );
    assert_eq!(
        encode(16383, &alice()).unwrap(),
        "yNa8JpqfFB3q8A29rCwSgxvdU94ufJw2yKKxDgznS5m1PoFvn"
    );
    assert_eq!(
        encode(64, &alice()).unwrap(),
        "cEaNSpz4PxFcZ7nT1VEKrKewH67rfx6MfcM6yKojyyPz7qaqp"
    );
}

#[test]
fn round_trip_over_prefixes() {
    for p in [
        0u16, 1, 2, 42, 45, 48, 62, 63, 64, 65, 255, 256, 1000, 4096, 16382, 16383,
    ] {
        let a = encode(p, &alice()).unwrap();
        if p == 46 || p == 47 {
            continue;
        }
        assert_eq!(decode(&a), Ok((p, alice())), "prefix {p}");
    }
}

#[test]
fn prefix_out_of_range_is_refused() {
    assert_eq!(encode(16384, &alice()), Err(Ss58Error::BadPrefix));
    assert_eq!(encode(u16::MAX, &alice()), Err(Ss58Error::BadPrefix));
}

#[test]
fn reserved_prefixes_are_refused_on_decode() {
    for p in [46u16, 47] {
        let a = encode(p, &alice()).unwrap();
        assert_eq!(decode(&a), Err(Ss58Error::BadPrefix));
    }
}

#[test]
fn every_single_character_change_is_caught() {
    let a = encode(42, &alice()).unwrap();
    let alphabet = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";
    let mut caught = 0;
    let mut total = 0;
    for i in 0..a.len() {
        for c in alphabet.chars() {
            let orig = a.as_bytes()[i] as char;
            if c == orig {
                continue;
            }
            let mut s: Vec<char> = a.chars().collect();
            s[i] = c;
            let s: String = s.into_iter().collect();
            total += 1;
            if decode(&s) != Ok((42, alice())) {
                caught += 1;
            }
        }
    }
    assert_eq!(caught, total);
}

#[test]
fn bad_inputs() {
    assert_eq!(decode(""), Err(Ss58Error::BadLength));
    assert_eq!(decode("0OIl"), Err(Ss58Error::BadBase58));
    assert_eq!(decode("5Grw"), Err(Ss58Error::BadLength));
    // a different network's prefix with a copied checksum is not valid
    let a = encode(42, &alice()).unwrap();
    let mut raw = solanalab::base58::decode(&a).unwrap();
    raw[0] = 0;
    let forged = solanalab::base58::encode(&raw);
    assert_eq!(decode(&forged), Err(Ss58Error::BadChecksum));
}

#[test]
fn two_byte_form_for_small_prefix_is_refused() {
    // build the 2-byte form of prefix 5 by hand: ident=5 -> first=((5&0xfc)>>2)|0x40, second=(5>>8)|((5&3)<<6)
    let ident = 5u16;
    let first = (((ident & 0xfc) >> 2) as u8) | 0x40;
    let second = ((ident >> 8) as u8) | (((ident & 3) as u8) << 6);
    let p = vec![first, second];
    let mut data = b"SS58PRE".to_vec();
    data.extend_from_slice(&p);
    data.extend_from_slice(&alice());
    let h = substratelab::hashing::blake2_512(&data);
    let mut raw = p;
    raw.extend_from_slice(&alice());
    raw.extend_from_slice(&h[..2]);
    let text = solanalab::base58::encode(&raw);
    assert_eq!(decode(&text), Err(Ss58Error::BadPrefix));
}

#[test]
fn leading_byte_over_127_is_a_bad_prefix() {
    let mut raw = vec![200u8];
    raw.extend_from_slice(&alice());
    raw.extend_from_slice(&[0, 0]);
    assert_eq!(
        decode(&solanalab::base58::encode(&raw)),
        Err(Ss58Error::BadPrefix)
    );
}
```

### `tests/extrinsic_tests.rs`

```rust
mod common;
use common::*;
use substratelab::extrinsic::*;
use substratelab::scale::DecodeError;

fn ctx() -> ChainContext {
    ChainContext {
        spec_version: 100,
        tx_version: 1,
        genesis: [0xaa; 32],
    }
}
fn transfer() -> Call {
    Call::Transfer {
        dest: [2; 32],
        value: 1000,
    }
}

const WIRE: &str = "2d028400ea4a6c63e29c520abef5507b132ec5f9954776aebebe7b92421eea691446d22c007004449d38ad5dfc9bfc2cb35b028c75dca5d55d87b9c700d2ebf44e1475b9630a9e2e0499ae06b2be2fe6d53219adde2b48aca56efe359f72d3b8866ce71d050014000100000202020202020202020202020202020202020202020202020202020202020202a10f";

#[test]
fn wire_bytes_match_an_independent_python_build() {
    let x = Extrinsic::sign([7; 32], transfer(), Extra { nonce: 5, tip: 0 }, &ctx());
    assert_eq!(hex::encode(x.to_bytes()), WIRE);
    assert_eq!(
        hex::encode(x.hash()),
        "c679b38f95c8778da99c4a848c2a21dd70edff12d102d4460158dfdb4e35855b"
    );
}

#[test]
fn parse_what_python_built() {
    let x = Extrinsic::from_bytes(&hex::decode(WIRE).unwrap()).unwrap();
    assert!(x.verify(&ctx()));
    assert_eq!(x.call, transfer());
    assert_eq!(x.signed.unwrap().extra, Extra { nonce: 5, tip: 0 });
}

#[test]
fn signature_binds_the_chain() {
    let x = Extrinsic::sign([7; 32], transfer(), Extra { nonce: 5, tip: 0 }, &ctx());
    assert!(x.verify(&ctx()));
    let mut other = ctx();
    other.genesis = [0xbb; 32];
    assert!(!x.verify(&other));
    let mut other = ctx();
    other.spec_version = 101;
    assert!(!x.verify(&other));
    let mut other = ctx();
    other.tx_version = 2;
    assert!(!x.verify(&other));
}

#[test]
fn signature_binds_every_signed_field() {
    let x = Extrinsic::sign([7; 32], transfer(), Extra { nonce: 5, tip: 0 }, &ctx());
    let mut y = x.clone();
    y.call = Call::Transfer {
        dest: [2; 32],
        value: 1001,
    };
    assert!(!y.verify(&ctx()));
    let mut y = x.clone();
    y.signed.as_mut().unwrap().extra.nonce = 6;
    assert!(!y.verify(&ctx()));
    let mut y = x.clone();
    y.signed.as_mut().unwrap().extra.tip = 1;
    assert!(!y.verify(&ctx()));
    let mut y = x.clone();
    y.signed.as_mut().unwrap().signer = [3; 32];
    assert!(!y.verify(&ctx()));
}

#[test]
fn flipping_any_bit_of_the_wire_form_breaks_or_changes_it() {
    let good = hex::decode(WIRE).unwrap();
    let orig = Extrinsic::from_bytes(&good).unwrap();
    for i in 0..good.len() {
        for bit in 0..8 {
            let mut b = good.clone();
            b[i] ^= 1 << bit;
            match Extrinsic::from_bytes(&b) {
                Err(_) => {}
                Ok(x) => assert!(x != orig && !x.verify(&ctx()), "byte {i} bit {bit}"),
            }
        }
    }
}

#[test]
fn length_prefix_must_match() {
    let good = hex::decode(WIRE).unwrap();
    let mut longer = good.clone();
    longer.push(0);
    assert_eq!(Extrinsic::from_bytes(&longer), Err(DecodeError::Trailing));
    assert_eq!(
        Extrinsic::from_bytes(&good[..good.len() - 1]),
        Err(DecodeError::Eof)
    );
    assert_eq!(Extrinsic::from_bytes(&[]), Err(DecodeError::Eof));
}

#[test]
fn unsigned_and_remark_round_trip() {
    let x = Extrinsic::unsigned(Call::Remark(b"hello".to_vec()));
    let b = x.to_bytes();
    assert_eq!(hex::encode(&b), "240400001468656c6c6f"); // body is 9 bytes -> prefix 0x24
    assert_eq!(Extrinsic::from_bytes(&b), Ok(x.clone()));
    assert!(x.verify(&ctx()));
}

#[test]
fn unknown_versions_calls_and_tags_are_refused() {
    // version byte
    assert!(matches!(
        Extrinsic::from_bytes(&[0x10, 0x05, 0, 0, 0]),
        Err(DecodeError::InvalidTag(5))
    ));
    // unknown pallet
    assert!(matches!(
        Extrinsic::from_bytes(&[0x10, 0x04, 9, 0, 0]),
        Err(DecodeError::InvalidTag(9))
    ));
    // bad address tag in a transfer
    let mut b = vec![0u8; 0];
    b.push(0x04);
    b.extend_from_slice(&[1, 0, 1]);
    b.extend_from_slice(&[0; 32]);
    b.push(0);
    let mut w = vec![((b.len() as u8) << 2)];
    w.extend_from_slice(&b);
    assert!(matches!(
        Extrinsic::from_bytes(&w),
        Err(DecodeError::InvalidTag(1))
    ));
}

#[test]
fn big_payloads_are_hashed_before_signing() {
    let small = signing_payload(
        &Call::Remark(vec![1; 10]),
        &Extra { nonce: 0, tip: 0 },
        &ctx(),
    );
    assert!(small.len() < 256);
    let big = signing_payload(
        &Call::Remark(vec![1; 300]),
        &Extra { nonce: 0, tip: 0 },
        &ctx(),
    );
    assert_eq!(big.len(), 32);
    // exactly at the boundary: 256 is not hashed, 257 is
    let base = signing_payload(&Call::Remark(vec![]), &Extra { nonce: 0, tip: 0 }, &ctx()).len();
    // remark of n bytes adds n + (len prefix); pick n so total is 256 / 257
    let pick = |target: usize| {
        (0..400usize)
            .find(|n| {
                let c = Call::Remark(vec![0; *n]);
                let mut raw = substratelab::scale::Encode::encode(&c);
                raw.extend_from_slice(&[0, 0, 0]);
                raw.len() + 8 + 64 == target
            })
            .unwrap()
    };
    let n256 = pick(256);
    let n257 = pick(257);
    assert!(base < 256);
    assert_eq!(
        signing_payload(
            &Call::Remark(vec![0; n256]),
            &Extra { nonce: 0, tip: 0 },
            &ctx()
        )
        .len(),
        256
    );
    assert_eq!(
        signing_payload(
            &Call::Remark(vec![0; n257]),
            &Extra { nonce: 0, tip: 0 },
            &ctx()
        )
        .len(),
        32
    );
}

#[test]
fn ed25519_known_answer_for_the_deterministic_key() {
    use ed25519_dalek::{Signer, SigningKey};
    let k = SigningKey::from_bytes(&[7; 32]);
    assert_eq!(
        hex::encode(k.verifying_key().to_bytes()),
        "ea4a6c63e29c520abef5507b132ec5f9954776aebebe7b92421eea691446d22c"
    );
    assert_eq!(
        hex::encode(k.sign(b"hello").to_bytes()),
        "359a315920d9541c3cc2a1dd1839f3e40bf23358a1d93a6ebd8303c0310ceb5025e679222ab016b4d822c5001e787e00c0ceaa6ac3c6e80248a944bd47104f0c"
    );
}

#[test]
fn random_calls_round_trip() {
    let mut r = Rng(5);
    for _ in 0..300 {
        let call = if r.below(2) == 0 {
            let l = r.below(300) as usize;
            Call::Remark(r.bytes(l))
        } else {
            let mut d = [0u8; 32];
            d.copy_from_slice(&r.bytes(32));
            Call::Transfer {
                dest: d,
                value: interesting(&mut r),
            }
        };
        let extra = Extra {
            nonce: r.next_u64() >> r.below(64),
            tip: interesting(&mut r),
        };
        let mut seed = [0u8; 32];
        seed.copy_from_slice(&r.bytes(32));
        let x = Extrinsic::sign(seed, call, extra, &ctx());
        let back = Extrinsic::from_bytes(&x.to_bytes()).unwrap();
        assert_eq!(back, x);
        assert!(back.verify(&ctx()));
    }
}
```

### `tests/runtime_tests.rs`

```rust
mod common;
use common::*;
use substratelab::extrinsic::*;
use substratelab::hashing::*;
use substratelab::runtime::*;
use substratelab::scale::{Encode, decode_all};

fn pk(seed: u8) -> [u8; 32] {
    ed25519_dalek::SigningKey::from_bytes(&[seed; 32])
        .verifying_key()
        .to_bytes()
}
fn chain() -> Chain {
    Chain::new(&[(pk(1), 1000), (pk(2), 500)])
}
fn xfer(c: &Chain, seed: u8, nonce: u64, to: [u8; 32], value: u128, tip: u128) -> Vec<u8> {
    Extrinsic::sign(
        [seed; 32],
        Call::Transfer { dest: to, value },
        Extra { nonce, tip },
        &c.ctx,
    )
    .to_bytes()
}

#[test]
fn genesis_state_and_keys() {
    let c = chain();
    assert_eq!(
        c.account(&pk(1)),
        AccountInfo {
            nonce: 0,
            free: 1000
        }
    );
    assert_eq!(c.total_issuance(), 1500);
    assert_eq!(c.sum_balances(), 1500);
    // the raw storage value is plain SCALE under the documented key
    let key = map_key("System", "Account", Hasher::Blake2_128Concat, &pk(1));
    assert_eq!(
        c.storage(&key).unwrap(),
        &AccountInfo {
            nonce: 0,
            free: 1000
        }
        .encode()
    );
    assert_eq!(
        c.storage(&storage_prefix("Balances", "TotalIssuance"))
            .unwrap(),
        &1500u128.encode()
    );
    assert_eq!(c.best().header.number, 0);
    assert_eq!(c.best().header.parent_hash, [0; 32]);
}

#[test]
fn transfer_moves_value_and_burns_the_fee() {
    let mut c = chain();
    let tx = xfer(&c, 1, 0, pk(3), 100, 4);
    c.submit(&tx).unwrap();
    c.produce_block();
    assert_eq!(
        c.account(&pk(1)),
        AccountInfo {
            nonce: 1,
            free: 1000 - 100 - BASE_FEE - 4
        }
    );
    assert_eq!(c.account(&pk(3)).free, 100);
    assert_eq!(c.total_issuance(), 1500 - BASE_FEE - 4);
    assert_eq!(c.sum_balances(), c.total_issuance());
    assert_eq!(c.best().header.number, 1);
    assert_eq!(c.best().extrinsics, vec![tx]);
    assert!(matches!(
        c.best().events[0],
        Event::Transfer { value: 100, .. }
    ));
}

#[test]
fn blocks_chain_by_hash() {
    let mut c = chain();
    let g = c.best().header.hash();
    c.produce_block();
    assert_eq!(c.best().header.parent_hash, g);
    let b1 = c.best().header.hash();
    c.produce_block();
    assert_eq!(c.best().header.parent_hash, b1);
    assert_eq!(c.best().header.number, 2);
    assert_ne!(g, b1);
    assert_eq!(c.ctx.genesis, g);
}

#[test]
fn state_root_changes_with_state_and_only_then() {
    let mut c = chain();
    let r0 = c.state_root();
    c.produce_block(); // empty block
    assert_eq!(c.best().header.state_root, r0);
    c.submit(&xfer(&c, 1, 0, pk(3), 100, 0)).unwrap();
    c.produce_block();
    assert_ne!(c.best().header.state_root, r0);
    assert_eq!(c.best().header.state_root, c.state_root());
}

#[test]
fn extrinsics_root_commits_to_the_included_list() {
    let mut a = chain();
    let mut b = chain();
    a.submit(&xfer(&a, 1, 0, pk(3), 100, 0)).unwrap();
    b.submit(&xfer(&b, 1, 0, pk(3), 101, 0)).unwrap();
    a.produce_block();
    b.produce_block();
    assert_ne!(
        a.best().header.extrinsics_root,
        b.best().header.extrinsics_root
    );
    assert_ne!(a.best().header.hash(), b.best().header.hash());
}

#[test]
fn same_inputs_same_chain() {
    let run = || {
        let mut c = chain();
        for i in 0..5u64 {
            c.submit(&xfer(&c, 1, i, pk(3), 10 + i as u128, 0)).unwrap();
        }
        c.produce_block();
        c.best().header.hash()
    };
    assert_eq!(run(), run());
}

#[test]
fn pool_rejections() {
    let mut c = chain();
    assert_eq!(c.submit(&[1, 2, 3]), Err(Invalid::BadEncoding));
    assert_eq!(
        c.submit(&Extrinsic::unsigned(Call::Remark(vec![])).to_bytes()),
        Err(Invalid::Unsigned)
    );
    // wrong chain
    let mut other = c.ctx;
    other.genesis = [1; 32];
    let bad = Extrinsic::sign(
        [1; 32],
        Call::Remark(vec![]),
        Extra { nonce: 0, tip: 0 },
        &other,
    )
    .to_bytes();
    assert_eq!(c.submit(&bad), Err(Invalid::BadSignature));
    // future nonce
    assert_eq!(c.submit(&xfer(&c, 1, 1, pk(3), 1, 0)), Err(Invalid::Future));
    // cannot pay the fee
    assert_eq!(
        c.submit(&xfer(&c, 9, 0, pk(3), 1, 0)),
        Err(Invalid::CannotPayFee)
    );
    assert_eq!(
        c.submit(&xfer(&c, 1, 0, pk(3), 1, 5000)),
        Err(Invalid::CannotPayFee)
    );
    assert_eq!(c.pool_len(), 0);
}

#[test]
fn nonces_replays_and_pool_ordering() {
    let mut c = chain();
    let t0 = xfer(&c, 1, 0, pk(3), 20, 0);
    c.submit(&t0).unwrap();
    assert_eq!(c.next_nonce(&pk(1)), 1);
    // same nonce again while pending is stale
    assert_eq!(c.submit(&t0), Err(Invalid::Stale));
    c.submit(&xfer(&c, 1, 1, pk(3), 20, 0)).unwrap();
    assert_eq!(c.next_nonce(&pk(1)), 2);
    c.produce_block();
    assert_eq!(c.account(&pk(1)).nonce, 2);
    // replay after inclusion
    assert_eq!(c.submit(&t0), Err(Invalid::Stale));
    assert_eq!(c.next_nonce(&pk(1)), 2);
}

#[test]
fn failed_dispatch_is_included_and_still_pays() {
    let mut c = chain();
    c.submit(&xfer(&c, 2, 0, pk(3), 9999, 0)).unwrap();
    c.produce_block();
    assert_eq!(c.best().extrinsics.len(), 1);
    assert_eq!(
        c.account(&pk(2)),
        AccountInfo {
            nonce: 1,
            free: 500 - BASE_FEE
        }
    );
    assert_eq!(c.account(&pk(3)).free, 0);
    assert!(c.best().events.iter().any(|e| matches!(
        e,
        Event::Failed {
            error: DispatchError::InsufficientBalance,
            ..
        }
    )));
    assert_eq!(c.sum_balances(), c.total_issuance());
}

#[test]
fn existential_deposit_on_the_receiver() {
    let mut c = chain();
    c.submit(&xfer(&c, 1, 0, pk(3), EXISTENTIAL_DEPOSIT - 1, 0))
        .unwrap();
    c.produce_block();
    assert!(c.best().events.iter().any(|e| matches!(
        e,
        Event::Failed {
            error: DispatchError::BelowExistentialDeposit,
            ..
        }
    )));
    assert_eq!(c.storage(&account_key(&pk(3))), None);
    c.submit(&xfer(&c, 1, 1, pk(3), EXISTENTIAL_DEPOSIT, 0))
        .unwrap();
    c.produce_block();
    assert_eq!(c.account(&pk(3)).free, EXISTENTIAL_DEPOSIT);
}

#[test]
fn self_transfer_is_a_dispatch_error() {
    let mut c = chain();
    c.submit(&xfer(&c, 1, 0, pk(1), 5, 0)).unwrap();
    c.produce_block();
    assert!(c.best().events.iter().any(|e| matches!(
        e,
        Event::Failed {
            error: DispatchError::SelfTransfer,
            ..
        }
    )));
    assert_eq!(c.account(&pk(1)).free, 1000 - BASE_FEE);
}

#[test]
fn a_sender_left_under_the_deposit_is_reaped_and_dust_is_burned() {
    let mut c = chain();
    // 500 - fee 1 - 495 = 4 left, under the deposit of 10
    c.submit(&xfer(&c, 2, 0, pk(3), 495, 0)).unwrap();
    c.produce_block();
    assert_eq!(c.storage(&account_key(&pk(2))), None);
    assert!(
        c.best()
            .events
            .iter()
            .any(|e| matches!(e, Event::Reaped { dust: 4, .. }))
    );
    assert_eq!(c.total_issuance(), 1500 - BASE_FEE - 4);
    assert_eq!(c.sum_balances(), c.total_issuance());
}

#[test]
fn reaping_resets_the_nonce_which_is_why_replay_protection_needs_more() {
    let mut c = chain();
    let t0 = xfer(&c, 2, 0, pk(3), 495, 0);
    c.submit(&t0).unwrap();
    c.produce_block();
    assert_eq!(c.account(&pk(2)).nonce, 0); // gone, so back to zero
    // fund it again; the old signed transfer would be valid again
    c.submit(&xfer(&c, 1, 0, pk(2), 600, 0)).unwrap();
    c.produce_block();
    assert_eq!(
        c.submit(&t0),
        Ok(Extrinsic::from_bytes(&t0).unwrap().hash())
    );
}

#[test]
fn remark_costs_only_the_fee() {
    let mut c = chain();
    let x = Extrinsic::sign(
        [1; 32],
        Call::Remark(b"hi".to_vec()),
        Extra { nonce: 0, tip: 0 },
        &c.ctx,
    )
    .to_bytes();
    c.submit(&x).unwrap();
    c.produce_block();
    assert_eq!(c.account(&pk(1)).free, 1000 - BASE_FEE);
    assert!(matches!(c.best().events[0], Event::Remark { len: 2, .. }));
}

#[test]
fn conservation_holds_under_random_traffic() {
    let mut r = Rng(2024);
    let mut c = chain();
    for round in 0..40 {
        for _ in 0..r.below(6) {
            let seed = (1 + r.below(3)) as u8;
            let who = pk(seed);
            let nonce = c.next_nonce(&who);
            let to = pk((1 + r.below(5)) as u8);
            let value = u128::from(r.below(400));
            let tip = u128::from(r.below(3));
            let _ = c.submit(&xfer(&c, seed, nonce, to, value, tip));
        }
        c.produce_block();
        assert_eq!(c.sum_balances(), c.total_issuance(), "round {round}");
        assert_eq!(c.best().header.number, round + 1);
        assert_eq!(c.best().header.state_root, c.state_root());
    }
}

#[test]
fn storage_values_decode_with_the_public_codec() {
    let c = chain();
    let key = account_key(&pk(1));
    let raw = c.storage(&key).unwrap();
    assert_eq!(
        decode_all::<AccountInfo>(raw),
        Ok(AccountInfo {
            nonce: 0,
            free: 1000
        })
    );
    assert_eq!(raw.len(), 4 + 16);
}

#[test]
fn header_encoding_layout() {
    let c = chain();
    let h = &c.best().header;
    let e = h.encode();
    assert_eq!(e.len(), 32 + 1 + 32 + 32); // number 0 is one compact byte
    assert_eq!(&e[..32], &h.parent_hash);
    assert_eq!(e[32], 0);
    assert_eq!(h.hash(), blake2_256(&e));
}

#[test]
fn block_lookup() {
    let mut c = chain();
    c.produce_block();
    assert!(c.block(0).is_some());
    assert!(c.block(1).is_some());
    assert!(c.block(2).is_none());
    assert!(c.block(u64::MAX).is_none());
}
```

### `tests/rpc_tests.rs`

```rust
use serde_json::{Value, json};
use substratelab::extrinsic::*;
use substratelab::hashing::*;
use substratelab::rpc::*;
use substratelab::runtime::*;
use substratelab::scale::Encode;
use substratelab::ss58;

fn pk(seed: u8) -> [u8; 32] {
    ed25519_dalek::SigningKey::from_bytes(&[seed; 32])
        .verifying_key()
        .to_bytes()
}
fn req(c: &mut Chain, method: &str, params: Value) -> Value {
    let r = json!({"jsonrpc":"2.0","id":1,"method":method,"params":params}).to_string();
    serde_json::from_str(&handle(c, &r)).unwrap()
}
fn chain() -> Chain {
    Chain::new(&[(pk(1), 1000)])
}

#[test]
fn get_storage_returns_hex_scale() {
    let mut c = chain();
    let key = to_hex(&map_key(
        "System",
        "Account",
        Hasher::Blake2_128Concat,
        &pk(1),
    ));
    let r = req(&mut c, "state_getStorage", json!([key]));
    assert_eq!(
        r["result"],
        json!(to_hex(
            &AccountInfo {
                nonce: 0,
                free: 1000
            }
            .encode()
        ))
    );
    assert_eq!(r["id"], json!(1));
    assert_eq!(r["jsonrpc"], json!("2.0"));
    let r = req(&mut c, "state_getStorage", json!(["0x00"]));
    assert_eq!(r["result"], Value::Null);
}

#[test]
fn submit_and_produce() {
    let mut c = chain();
    let x = Extrinsic::sign(
        [1; 32],
        Call::Transfer {
            dest: pk(2),
            value: 50,
        },
        Extra { nonce: 0, tip: 0 },
        &c.ctx,
    );
    let r = req(
        &mut c,
        "author_submitExtrinsic",
        json!([to_hex(&x.to_bytes())]),
    );
    assert_eq!(r["result"], json!(to_hex(&x.hash())));
    let addr = ss58::encode(42, &pk(1)).unwrap();
    assert_eq!(
        req(&mut c, "system_accountNextIndex", json!([addr.clone()]))["result"],
        json!(1)
    );
    let r = req(&mut c, "dev_produceBlock", json!([]));
    assert_eq!(r["result"], json!({"number": 1, "extrinsics": 1}));
    assert_eq!(
        req(&mut c, "system_accountNextIndex", json!([addr]))["result"],
        json!(1)
    );
    let h = req(&mut c, "chain_getHeader", json!([]));
    assert_eq!(h["result"]["number"], json!("0x1"));
}

#[test]
fn rejected_submit_has_code_1010_and_a_reason() {
    let mut c = chain();
    let x = Extrinsic::sign(
        [1; 32],
        Call::Remark(vec![]),
        Extra { nonce: 7, tip: 0 },
        &c.ctx,
    );
    let r = req(
        &mut c,
        "author_submitExtrinsic",
        json!([to_hex(&x.to_bytes())]),
    );
    assert_eq!(r["error"]["code"], json!(INVALID_TX));
    assert_eq!(r["error"]["message"], json!("Invalid Transaction: Future"));
    assert!(r.get("result").is_none());
}

#[test]
fn block_hash_and_header_by_hash() {
    let mut c = chain();
    c.produce_block();
    let h0 = req(&mut c, "chain_getBlockHash", json!([0]))["result"].clone();
    assert_eq!(h0, json!(to_hex(&c.ctx.genesis)));
    let h1 = req(&mut c, "chain_getBlockHash", json!([]))["result"].clone();
    assert_eq!(h1, json!(to_hex(&c.best().header.hash())));
    assert_eq!(
        req(&mut c, "chain_getBlockHash", json!([9]))["result"],
        Value::Null
    );
    let hdr = req(&mut c, "chain_getHeader", json!([h0]));
    assert_eq!(hdr["result"]["number"], json!("0x0"));
    assert_eq!(
        req(&mut c, "chain_getHeader", json!(["0x1234"]))["result"],
        Value::Null
    );
}

#[test]
fn protocol_errors() {
    let mut c = chain();
    let r: Value = serde_json::from_str(&handle(&mut c, "{not json")).unwrap();
    assert_eq!(r["error"]["code"], json!(PARSE_ERROR));
    assert_eq!(r["id"], Value::Null);
    let r: Value = serde_json::from_str(&handle(&mut c, r#"{"jsonrpc":"2.0","id":3}"#)).unwrap();
    assert_eq!(r["error"]["code"], json!(INVALID_REQUEST));
    let r: Value = serde_json::from_str(&handle(
        &mut c,
        r#"{"jsonrpc":"1.0","id":3,"method":"system_chain"}"#,
    ))
    .unwrap();
    assert_eq!(r["error"]["code"], json!(INVALID_REQUEST));
    let r = req(&mut c, "nope", json!([]));
    assert_eq!(r["error"]["code"], json!(METHOD_NOT_FOUND));
    let r = req(&mut c, "state_getStorage", json!([5]));
    assert_eq!(r["error"]["code"], json!(INVALID_PARAMS));
    let r = req(&mut c, "state_getStorage", json!(["zz"]));
    assert_eq!(r["error"]["code"], json!(INVALID_PARAMS));
    let r: Value = serde_json::from_str(&handle(
        &mut c,
        r#"{"jsonrpc":"2.0","id":"a","method":"system_chain","params":{}}"#,
    ))
    .unwrap();
    assert_eq!(r["error"]["code"], json!(INVALID_PARAMS));
    assert_eq!(r["id"], json!("a"));
    let r: Value = serde_json::from_str(&handle(
        &mut c,
        r#"{"jsonrpc":"2.0","id":"a","method":"system_chain"}"#,
    ))
    .unwrap();
    assert_eq!(r["result"], json!("Teaching Dev"));
}

#[test]
fn params_are_validated() {
    let mut c = chain();
    assert_eq!(
        req(&mut c, "system_accountNextIndex", json!(["nonsense"]))["error"]["code"],
        json!(INVALID_PARAMS)
    );
    assert_eq!(
        req(&mut c, "system_accountNextIndex", json!([]))["error"]["code"],
        json!(INVALID_PARAMS)
    );
    assert_eq!(
        req(&mut c, "author_submitExtrinsic", json!(["0xzz"]))["error"]["code"],
        json!(INVALID_PARAMS)
    );
    assert_eq!(
        req(&mut c, "author_submitExtrinsic", json!(["0x010203"]))["error"]["code"],
        json!(INVALID_TX)
    );
    assert_eq!(
        req(&mut c, "chain_getBlockHash", json!(["x"]))["error"]["code"],
        json!(INVALID_PARAMS)
    );
    assert_eq!(
        req(&mut c, "chain_getBlockHash", json!([-1]))["error"]["code"],
        json!(INVALID_PARAMS)
    );
    assert_eq!(
        req(&mut c, "chain_getHeader", json!([5]))["error"]["code"],
        json!(INVALID_PARAMS)
    );
}

#[test]
fn hex_helpers() {
    assert_eq!(to_hex(&[1, 0xab]), "0x01ab");
    assert_eq!(from_hex("0x01ab"), Some(vec![1, 0xab]));
    assert_eq!(from_hex("01ab"), None);
    assert_eq!(from_hex("0x1"), None);
    assert_eq!(from_hex("0x"), Some(vec![]));
}
```

### `tests/gap_tests.rs`

```rust
//! Tests written by reading the planned mutant list: each pins something the earlier tests left loose.
use serde_json::{Value, json};
use substratelab::extrinsic::*;
use substratelab::rpc::*;
use substratelab::runtime::*;
use substratelab::scale::{Compact, DecodeError, Encode};
use substratelab::ss58;

fn pk(seed: u8) -> [u8; 32] {
    ed25519_dalek::SigningKey::from_bytes(&[seed; 32])
        .verifying_key()
        .to_bytes()
}
fn tx(c: &Chain, seed: u8, nonce: u64, call: Call, tip: u128) -> Vec<u8> {
    Extrinsic::sign([seed; 32], call, Extra { nonce, tip }, &c.ctx).to_bytes()
}
fn send(c: &mut Chain, seed: u8, nonce: u64, to: [u8; 32], value: u128) {
    let b = tx(c, seed, nonce, Call::Transfer { dest: to, value }, 0);
    c.submit(&b).unwrap();
}
fn rpc(c: &mut Chain, method: &str, params: Value) -> Value {
    let r = json!({"jsonrpc":"2.0","id":1,"method":method,"params":params}).to_string();
    serde_json::from_str(&handle(c, &r)).unwrap()
}

const WIRE: &str = "2d028400ea4a6c63e29c520abef5507b132ec5f9954776aebebe7b92421eea691446d22c007004449d38ad5dfc9bfc2cb35b028c75dca5d55d87b9c700d2ebf44e1475b9630a9e2e0499ae06b2be2fe6d53219adde2b48aca56efe359f72d3b8866ce71d050014000100000202020202020202020202020202020202020202020202020202020202020202a10f";

#[test]
fn documented_constants_are_pinned() {
    assert_eq!(EXISTENTIAL_DEPOSIT, 10);
    assert_eq!(BASE_FEE, 1);
    assert_eq!(V4_SIGNED, 0x84);
    assert_eq!(V4_UNSIGNED, 0x04);
    assert_eq!(ss58::MAX_PREFIX, 16383);
}

#[test]
fn json_rpc_error_code_values() {
    assert_eq!(PARSE_ERROR, -32700);
    assert_eq!(INVALID_REQUEST, -32600);
    assert_eq!(METHOD_NOT_FOUND, -32601);
    assert_eq!(INVALID_PARAMS, -32602);
    assert_eq!(INVALID_TX, 1010);
}

#[test]
fn every_rejection_has_its_own_name_over_rpc() {
    let mut c = Chain::new(&[(pk(1), 1000)]);
    let msg = |r: &Value| r["error"]["message"].as_str().unwrap().to_string();
    let r = rpc(&mut c, "author_submitExtrinsic", json!(["0x010203"]));
    assert_eq!(msg(&r), "Invalid Transaction: BadEncoding");
    let u = Extrinsic::unsigned(Call::Remark(vec![])).to_bytes();
    let r = rpc(&mut c, "author_submitExtrinsic", json!([to_hex(&u)]));
    assert_eq!(msg(&r), "Invalid Transaction: UnsignedNotAllowed");
    let mut other = c.ctx;
    other.genesis = [1; 32];
    let bad = Extrinsic::sign(
        [1; 32],
        Call::Remark(vec![]),
        Extra { nonce: 0, tip: 0 },
        &other,
    );
    let r = rpc(
        &mut c,
        "author_submitExtrinsic",
        json!([to_hex(&bad.to_bytes())]),
    );
    assert_eq!(msg(&r), "Invalid Transaction: BadProof");
    let good = tx(&c, 1, 0, Call::Remark(vec![]), 0);
    rpc(&mut c, "author_submitExtrinsic", json!([to_hex(&good)]));
    let r = rpc(&mut c, "author_submitExtrinsic", json!([to_hex(&good)]));
    assert_eq!(msg(&r), "Invalid Transaction: Stale");
    let poor = tx(&c, 9, 0, Call::Remark(vec![]), 0);
    let r = rpc(&mut c, "author_submitExtrinsic", json!([to_hex(&poor)]));
    assert_eq!(msg(&r), "Invalid Transaction: Payment");
}

#[test]
fn prefix_byte_128_is_not_a_two_byte_prefix() {
    let mut raw = vec![128u8];
    raw.extend_from_slice(&pk(1));
    raw.extend_from_slice(&[0, 0]);
    let text = solanalab::base58::encode(&raw);
    assert_eq!(ss58::decode(&text), Err(ss58::Ss58Error::BadPrefix));
}

fn with_byte(offset: usize, value: u8) -> Vec<u8> {
    let mut w = hex::decode(WIRE).unwrap();
    w[offset] = value;
    w
}

#[test]
fn signed_extrinsic_tags_are_checked_one_by_one() {
    // wire offsets: 3 = address tag, 36 = signature kind, 101 = era
    for off in [3usize, 36, 101] {
        assert_eq!(
            Extrinsic::from_bytes(&with_byte(off, 1)),
            Err(DecodeError::InvalidTag(1)),
            "offset {off}"
        );
    }
}

#[test]
fn a_nonce_that_does_not_fit_in_u64_is_an_overflow() {
    let w = hex::decode(WIRE).unwrap();
    let body = &w[2..];
    // replace the one-byte nonce (body offset 100) with Compact(2^64)
    let big = Compact(1u128 << 64).encode();
    let mut nb = body[..100].to_vec();
    nb.extend_from_slice(&big);
    nb.extend_from_slice(&body[101..]);
    let mut out = Compact(nb.len() as u128).encode();
    out.extend_from_slice(&nb);
    assert_eq!(Extrinsic::from_bytes(&out), Err(DecodeError::Overflow));
    // and u64::MAX itself is fine
    let ok = Compact(u128::from(u64::MAX)).encode();
    let mut nb = body[..100].to_vec();
    nb.extend_from_slice(&ok);
    nb.extend_from_slice(&body[101..]);
    let mut out = Compact(nb.len() as u128).encode();
    out.extend_from_slice(&nb);
    let x = Extrinsic::from_bytes(&out).unwrap();
    assert_eq!(x.signed.unwrap().extra.nonce, u64::MAX);
}

#[test]
fn state_root_depends_on_keys_and_on_values() {
    let a = Chain::new(&[(pk(1), 1000)]);
    let b = Chain::new(&[(pk(2), 1000)]); // same values, different key
    let c = Chain::new(&[(pk(1), 999)]); // same key, different value
    assert_ne!(a.state_root(), b.state_root());
    assert_ne!(a.state_root(), c.state_root());
}

#[test]
fn next_nonce_is_per_account() {
    let mut c = Chain::new(&[(pk(1), 1000), (pk(2), 1000)]);
    send(&mut c, 1, 0, pk(3), 20);
    assert_eq!(c.next_nonce(&pk(1)), 1);
    assert_eq!(c.next_nonce(&pk(2)), 0);
    send(&mut c, 2, 0, pk(3), 20);
    send(&mut c, 2, 1, pk(3), 20);
    assert_eq!(c.next_nonce(&pk(1)), 1);
    assert_eq!(c.next_nonce(&pk(2)), 2);
}

#[test]
fn an_account_that_can_just_pay_the_fee_is_accepted_and_included() {
    let mut c = Chain::new(&[(pk(1), BASE_FEE)]);
    let b = tx(&c, 1, 0, Call::Remark(vec![]), 0);
    assert!(c.submit(&b).is_ok());
    c.produce_block();
    assert_eq!(c.best().extrinsics.len(), 1);
    assert_eq!(c.total_issuance(), 0);
}

#[test]
fn transferring_everything_after_the_fee_works() {
    let mut c = Chain::new(&[(pk(1), 100)]);
    send(&mut c, 1, 0, pk(3), 100 - BASE_FEE);
    c.produce_block();
    assert_eq!(c.account(&pk(3)).free, 99);
    assert!(
        c.best()
            .events
            .iter()
            .any(|e| matches!(e, Event::Transfer { value: 99, .. }))
    );
    assert_eq!(c.storage(&account_key(&pk(1))), None); // left with zero: reaped
}

#[test]
fn an_account_left_with_exactly_the_deposit_stays() {
    let mut c = Chain::new(&[(pk(1), 100)]);
    send(&mut c, 1, 0, pk(3), 100 - BASE_FEE - EXISTENTIAL_DEPOSIT);
    c.produce_block();
    assert_eq!(c.account(&pk(1)).free, EXISTENTIAL_DEPOSIT);
    assert_eq!(c.account(&pk(1)).nonce, 1);
    assert!(
        !c.best()
            .events
            .iter()
            .any(|e| matches!(e, Event::Reaped { .. }))
    );
}

#[test]
fn a_transaction_that_became_unaffordable_is_dropped_not_included() {
    let mut c = Chain::new(&[(pk(1), 30)]);
    send(&mut c, 1, 0, pk(3), 10); // leaves 19
    let hungry = tx(&c, 1, 1, Call::Remark(vec![]), 25); // looked affordable (30 >= 26) at submit
    c.submit(&hungry).unwrap();
    c.produce_block();
    assert_eq!(c.best().extrinsics.len(), 1);
    assert_eq!(c.account(&pk(1)).nonce, 1);
    assert_eq!(c.account(&pk(1)).free, 19);
    assert_eq!(c.pool_len(), 0);
    assert_eq!(c.sum_balances(), c.total_issuance());
}

#[test]
fn a_transaction_after_its_sender_was_reaped_is_dropped() {
    let mut c = Chain::new(&[(pk(1), 12)]);
    let t0 = tx(&c, 1, 0, Call::Remark(vec![]), 2); // fee 3 -> 9 left, reaped
    let t1 = tx(&c, 1, 1, Call::Remark(vec![]), 0);
    c.submit(&t0).unwrap();
    c.submit(&t1).unwrap();
    c.produce_block();
    assert_eq!(c.best().extrinsics.len(), 1);
    assert_eq!(c.total_issuance(), 12 - 3 - 9);
    assert_eq!(c.sum_balances(), 0);
}

#[test]
fn strings_longer_than_a_few_bytes_round_trip() {
    let s = "substrate and polkadot".to_string();
    let e = s.encode();
    assert_eq!(e.len(), 1 + s.len());
    assert_eq!(substratelab::scale::decode_all::<String>(&e), Ok(s));
}

fn rewrap(body: &[u8]) -> Vec<u8> {
    let mut out = Compact(body.len() as u128).encode();
    out.extend_from_slice(body);
    out
}

#[test]
fn one_byte_after_the_call_is_trailing_data() {
    let w = hex::decode(WIRE).unwrap();
    let mut body = w[2..].to_vec();
    body.push(0);
    assert_eq!(
        Extrinsic::from_bytes(&rewrap(&body)),
        Err(DecodeError::Trailing)
    );
}

#[test]
fn call_indexes_must_match_exactly() {
    // pallet 0 (System) has only call 0; pallet 1 (Balances) has only call 0
    let remark = [0x04u8, 0, 1, 0];
    assert_eq!(
        Extrinsic::from_bytes(&rewrap(&remark)),
        Err(DecodeError::InvalidTag(0))
    );
    let transfer = [0x04u8, 1, 1, 0];
    assert_eq!(
        Extrinsic::from_bytes(&rewrap(&transfer)),
        Err(DecodeError::InvalidTag(1))
    );
}

#[test]
fn a_stale_nonce_after_reap_and_refund_is_not_executed() {
    let mut c = Chain::new(&[(pk(1), 12), (pk(2), 1000)]);
    let t0 = tx(&c, 1, 0, Call::Remark(vec![]), 2); // fee 3, leaves 9: reaped
    let t1 = tx(&c, 1, 1, Call::Remark(vec![]), 0); // signed for nonce 1
    c.submit(&t0).unwrap();
    send(&mut c, 2, 0, pk(1), 100); // refunds the reaped account before t1 runs
    c.submit(&t1).unwrap();
    c.produce_block();
    assert_eq!(c.best().extrinsics.len(), 2);
    assert_eq!(
        c.account(&pk(1)),
        AccountInfo {
            nonce: 0,
            free: 100
        }
    );
    assert_eq!(c.sum_balances(), c.total_issuance());
}

#[test]
fn missing_params_means_an_empty_list() {
    let mut c = Chain::new(&[(pk(1), 12)]);
    c.produce_block();
    c.produce_block();
    let r = handle(
        &mut c,
        r#"{"jsonrpc":"2.0","id":1,"method":"chain_getBlockHash"}"#,
    );
    let v: Value = serde_json::from_str(&r).unwrap();
    assert_eq!(v["result"], json!(to_hex(&c.best().header.hash())));
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
use std::hint::black_box;
use std::time::Instant;
use substratelab::extrinsic::*;
use substratelab::hashing::{Hasher, map_key};
use substratelab::runtime::Chain;
use substratelab::scale::{Compact, Decode, Encode};
use substratelab::ss58;

fn time<F: FnMut()>(name: &str, iters: u32, mut f: F) {
    for _ in 0..(iters / 10).max(1) {
        f();
    }
    let t = Instant::now();
    for _ in 0..iters {
        f();
    }
    let ns = t.elapsed().as_nanos() as f64 / f64::from(iters);
    println!("{name:<34} {ns:>12.0} ns/op");
}

fn main() {
    let ctx = ChainContext {
        spec_version: 100,
        tx_version: 1,
        genesis: [0; 32],
    };
    let call = Call::Transfer {
        dest: [2; 32],
        value: 1000,
    };
    let x = Extrinsic::sign([7; 32], call.clone(), Extra { nonce: 5, tip: 0 }, &ctx);
    let bytes = x.to_bytes();
    let id = [9u8; 32];
    time("compact encode (u32-sized)", 2_000_000, || {
        black_box(Compact(black_box(123_456)).encode());
    });
    time("compact decode", 2_000_000, || {
        let b = [0x02, 0x89, 0x07, 0x00];
        black_box(Compact::decode(&mut &b[..]).unwrap());
    });
    time("storage map key (blake2_128_concat)", 500_000, || {
        black_box(map_key(
            "System",
            "Account",
            Hasher::Blake2_128Concat,
            black_box(&id),
        ));
    });
    time("ss58 encode", 100_000, || {
        black_box(ss58::encode(42, black_box(&id)).unwrap());
    });
    let addr = ss58::encode(42, &id).unwrap();
    time("ss58 decode", 100_000, || {
        black_box(ss58::decode(black_box(&addr)).unwrap());
    });
    time("extrinsic sign (ed25519)", 5_000, || {
        black_box(Extrinsic::sign(
            [7; 32],
            call.clone(),
            Extra { nonce: 5, tip: 0 },
            &ctx,
        ));
    });
    time("extrinsic parse + verify", 5_000, || {
        let p = Extrinsic::from_bytes(black_box(&bytes)).unwrap();
        black_box(p.verify(&ctx));
    });
    let endowed: Vec<([u8; 32], u128)> = (1..=50u8)
        .map(|s| {
            (
                ed25519_dalek::SigningKey::from_bytes(&[s; 32])
                    .verifying_key()
                    .to_bytes(),
                1_000_000,
            )
        })
        .collect();
    time("block of 50 transfers (submit+build)", 100, || {
        let mut c = Chain::new(&endowed);
        for s in 1..=50u8 {
            let xt = Extrinsic::sign(
                [s; 32],
                Call::Transfer {
                    dest: endowed[0].0,
                    value: 10,
                },
                Extra { nonce: 0, tip: 0 },
                &c.ctx,
            );
            c.submit(&xt.to_bytes()).unwrap();
        }
        black_box(c.produce_block().header.number);
    });
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **80 tests** (scale 12, hashing 6, SS58 8, extrinsic 11, runtime 18, RPC 7, gap tests 18) and 8 consecutive full runs without a failure (nothing uses sockets, real time or OS randomness; the random tests use a fixed-seed generator).

**Mutation check ✓.** I made 132 small source edits (a "mutant") across the six source modules, each in its own copy of the crate, and ran the whole test suite against each. **130 were caught by a failing test and 2 are equivalent (M95, M115); none fails to compile and none survives unexplained.** **What the first pass found.** I designed these mutants after reading the code and wrote 13 of the 18 extra tests (in `gap_tests.rs`) before the first run, to pin things I could already see were loose: the literal values of the constants and error codes, every named rejection over RPC, the tag bytes of an extrinsic, the 64-bit nonce limit, state-root sensitivity to keys and to values, nonces per account, and the boundaries of the fee, transfer and deposit rules. Against the 75 tests then in place, 125 of the 132 mutants were caught and 7 survived. Two are equivalent (below). Five were real gaps, each closed by one new test: **strings longer than three bytes** (a mutant that truncated them survived, because the only string tested was `"hi"`), **one trailing byte after the call** (the parser accepted a body one byte too long), **call indexes** (pallet 0 with call 1 was accepted as a remark), **a stale nonce after the sender was reaped and refunded in the same block** (the check at execution time could be removed without any test noticing, until a test arranged the order: spend, refund, then the stale transaction), and **missing `params`** (a default of `[1]` instead of `[]` went unnoticed on a chain with only one block, until the test used two). The last one is a reminder that a boundary test needs a case where the wrong answer differs from the right one. Two mutants remain, each with a stated reason for being equivalent; none fails to compile. The column "change" is generated from the edit itself (the part that differs, with a little context, whitespace collapsed); `…` marks omitted text. The tests ran in the dev profile with optimized dependencies.

| # | module | change | result |
|---|---|---|---|
| M1 | `scale` | `if v <` → `if v <=` | caught by `compact_known_answers_from_the_spec` |
| M2 | `scale` | `} else if v <` → `} else if v <=` | caught by `compact_known_answers_from_the_spec` |
| M3 | `scale` | `} else if v <` → `} else if v <=` | caught by `compact_known_answers_from_the_spec` |
| M4 | `scale` | `…v as u8) << 2` → `…v as u8) << 1` | caught by `parse_what_python_built` |
| M5 | `scale` | `<< 2) \| 0b01` → `<< 2) \| 0b10` | caught by `parse_what_python_built` |
| M6 | `scale` | `<< 2) \| 0b10` → `<< 2) \| 0b01` | caught by `random_calls_round_trip` |
| M7 | `scale` | `16` → `15` | caught by `random_calls_round_trip` |
| M8 | `scale` | `(((n - 4` → `(((n - 3` | caught by `random_calls_round_trip` |
| M9 | `scale` | `…u128::from(first >> 2` → `…u128::from(first >> 1` | caught by `parse_what_python_built` |
| M10 | `scale` | `if v <` → `if v <=` | caught by `compact_known_answers_from_the_spec` |
| M11 | `scale` | `if v <` → `if v <=` | caught by `compact_known_answers_from_the_spec` |
| M12 | `scale` | `if n > 16` → `if n > 15` | caught by `random_calls_round_trip` |
| M13 | `scale` | `if b[n - 1` → `if b[0` | caught by `random_calls_round_trip` |
| M14 | `scale` | `if v <` → `if v <=` | caught by `compact_known_answers_from_the_spec` |
| M15 | `scale` | `…first >> 2) + 4` → `…first >> 2) + 3` | caught by `random_calls_round_trip` |
| M16 | `scale` | `if n >` → `if n >=` | caught by `random_calls_round_trip` |
| M17 | `scale` | `if n >` → `if n >=` | caught by `bool_option_and_strings` |
| M18 | `scale` | `0 ` → `0 \| 2 ` | caught by `bool_option_and_strings` |
| M19 | `scale` | `out.push(1` → `out.push(2` | caught by `bool_option_and_strings` |
| M20 | `scale` | `…DecodeError::InvalidTag(b` → `…DecodeError::InvalidTag(b ^ 1` | caught by `bool_option_and_strings` |
| M21 | `scale` | `…self.len() as u128` → `…self.len() as u128 + 1` | caught by `random_calls_round_trip` |
| M22 | `scale` | `1 ` → `1 \| 2 ` | caught by `bool_option_and_strings` |
| M23 | `scale` | `b => Err(DecodeError::InvalidBool(b` → `_ => Err(DecodeError::InvalidBool(0` | caught by `bool_option_and_strings` |
| M24 | `scale` | `if input.is_empty()` → `if input.len() < 2` | caught by `decoding_garbage_never_panics_and_agrees_on_validity_of_compact` |
| M25 | `scale` | `…input.len() ` → `…input.len() + 1 ` | caught by `length_prefix_must_match` |
| M26 | `scale` | `…out.extend_from_slice(self.as_bytes()` → `…out.extend_from_slice(&self.as_bytes()[..self.len().min(3)]` | caught by `strings_longer_than_a_few_bytes_round_trip` |
| M27 | `hashing` | `…0, data).to_l` → `…0, data).to_b` | caught by `twox_hashes_are_little_endian_xxh64` |
| M28 | `hashing` | `XxHash64::oneshot(0` → `XxHash64::oneshot(1` | caught by `twox_hashes_are_little_endian_xxh64` |
| M29 | `hashing` | `…&XxHash64::oneshot(0` → `…&XxHash64::oneshot(1` | caught by `twox_hashes_are_little_endian_xxh64` |
| M30 | `hashing` | `…&XxHash64::oneshot(1` → `…&XxHash64::oneshot(2` | caught by `twox_hashes_are_little_endian_xxh64` |
| M31 | `hashing` | `…hashed.get(16` → `…hashed.get(15` | caught by `hasher_concat_can_recover_keys_identity_too` |
| M32 | `hashing` | `…=> hashed.get(8` → `…=> hashed.get(7` | caught by `hasher_concat_can_recover_keys_identity_too` |
| M33 | `hashing` | `…&blake2_128(key` → `…&blake2_128(b""` | caught by `hasher_concat_can_recover_keys_identity_too` |
| M34 | `hashing` | `out.extend_from_slice(&twox_64(key` → `out.extend_from_slice(&twox_64(b""` | caught by `hasher_concat_can_recover_keys_identity_too` |
| M35 | `hashing` | `…out.extend_from_slice(key` → `…out.extend_from_slice(&[]` | caught by `hasher_concat_can_recover_keys_identity_too` |
| M36 | `hashing` | `out[..16].copy_from_slice(&twox_128(pallet` → `out[..16].copy_from_slice(&twox_128(item` | caught by `well_known_storage_prefixes` |
| M37 | `hashing` | `out[16..].copy_from_slice(&twox_128(item` → `out[16..].copy_from_slice(&twox_128(pallet` | caught by `well_known_storage_prefixes` |
| M38 | `hashing` | `hasher.hash_to(key, &mut out)` → `let _ = hasher` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M39 | `hashing` | `Hasher::Identity => Some(hashed` → `Hasher::Identity => hashed.get(1..` | caught by `hasher_concat_can_recover_keys_identity_too` |
| M40 | `ss58` | `0..=63` → `0..=62` | caught by `round_trip_over_prefixes` |
| M41 | `ss58` | `\| 0b01` → `\| 0b10` | caught by `known_addresses_computed_independently` |
| M42 | `ss58` | `<< 6` → `<< 5` | caught by `known_addresses_computed_independently` |
| M43 | `ss58` | `b"SS58PRE` → `b"SS58PRF` | caught by `known_addresses_computed_independently` |
| M44 | `ss58` | `[h[0], h[1` → `[h[1], h[0` | caught by `known_addresses_computed_independently` |
| M45 | `ss58` | `64..=MAX_PREFIX` → `64..=16384` | caught by `prefix_out_of_range_is_refused` |
| M46 | `ss58` | `Some(&b) if b <` → `Some(&b) if b <=` | caught by `round_trip_over_prefixes` |
| M47 | `ss58` | `…&b) if b < 128` → `…&b) if b < 129` | caught by `prefix_byte_128_is_not_a_two_byte_prefix` |
| M48 | `ss58` | `u16::from(second >> 6` → `u16::from(second >> 5` | caught by `round_trip_over_prefixes` |
| M49 | `ss58` | `…second) & 0b00` → `…second) & 0b01` | caught by `round_trip_over_prefixes` |
| M50 | `ss58` | `…\| (upper << 8` → `…\| (upper << 7` | caught by `round_trip_over_prefixes` |
| M51 | `ss58` | `prefix_len + 32 + 2` → `prefix_len + 32 + 1` | caught by `submit_and_produce` |
| M52 | `ss58` | `prefix_len == 2 && prefix < 64 \|\| prefix` → `prefix` | caught by `two_byte_form_for_small_prefix_is_refused` |
| M53 | `ss58` | `\|\| prefix == 46` → `\|\| false` | caught by `reserved_prefixes_are_refused_on_decode` |
| M54 | `ss58` | `\|\| prefix == 47` → `\|\| false` | caught by `reserved_prefixes_are_refused_on_decode` |
| M55 | `ss58` | `checksum(&raw[..prefix_len` → `checksum(&[` | caught by `submit_and_produce` |
| M56 | `ss58` | `Ss58Error::BadBase58` → `Ss58Error::BadLength` | caught by `bad_inputs` |
| M57 | `ss58` | `…&data); [h[0], h[1]` → `…&data); [h[0], 0` | caught by `known_addresses_computed_independently` |
| M58 | `extrinsic` | `…V4_SIGNED: u8 = 0x84` → `…V4_SIGNED: u8 = 0x85` | caught by `flipping_any_bit_of_the_wire_form_breaks_or_changes_it` |
| M59 | `extrinsic` | `…V4_UNSIGNED: u8 = 0x04` → `…V4_UNSIGNED: u8 = 0x05` | caught by `unknown_versions_calls_and_tags_are_refused` |
| M60 | `extrinsic` | `out.extend_from_slice(&[1, 0` → `out.extend_from_slice(&[1, 1` | caught by `parse_what_python_built` |
| M61 | `extrinsic` | `out.extend_from_slice(&[0, 0` → `out.extend_from_slice(&[0, 1` | caught by `random_calls_round_trip` |
| M62 | `extrinsic` | `out.push(0` → `out.push(1` | caught by `parse_what_python_built` |
| M63 | `extrinsic` | `Compact(*value)` → `value` | caught by `parse_what_python_built` |
| M64 | `extrinsic` | `if tag != 0` → `if false` | caught by `flipping_any_bit_of_the_wire_form_breaks_or_changes_it` |
| M65 | `extrinsic` | `if tag != 0` → `if false` | caught by `flipping_any_bit_of_the_wire_form_breaks_or_changes_it` |
| M66 | `extrinsic` | `if sig_tag != 0` → `if false` | caught by `flipping_any_bit_of_the_wire_form_breaks_or_changes_it` |
| M67 | `extrinsic` | `if era != 0` → `if false` | caught by `flipping_any_bit_of_the_wire_form_breaks_or_changes_it` |
| M68 | `extrinsic` | `if p.len() >` → `if p.len() >=` | caught by `big_payloads_are_hashed_before_signing` |
| M69 | `extrinsic` | `…p.len() > 256` → `…p.len() > 257` | caught by `big_payloads_are_hashed_before_signing` |
| M70 | `extrinsic` | `p.extend_from_slice(&ctx.genesis); // block hash: immortal era uses…` → `∅` | caught by `big_payloads_are_hashed_before_signing` |
| M71 | `extrinsic` | `ctx.spec_version.encode_to(&mut p);` → `∅` | caught by `big_payloads_are_hashed_before_signing` |
| M72 | `extrinsic` | `ctx.tx_version.encode_to(&mut p);` → `∅` | caught by `big_payloads_are_hashed_before_signing` |
| M73 | `extrinsic` | `out.push(0); // era: immortal` → `∅` | caught by `big_payloads_are_hashed_before_signing` |
| M74 | `extrinsic` | `Compact(u128::from(e.nonce)).encode_to(out);` → `∅` | caught by `big_payloads_are_hashed_before_signing` |
| M75 | `extrinsic` | `Compact(e.tip).encode_to(out);` → `∅` | caught by `big_payloads_are_hashed_before_signing` |
| M76 | `extrinsic` | `…else { return tru` → `…else { return fals` | caught by `unsigned_and_remark_round_trip` |
| M77 | `extrinsic` | `if len != input.len() as u128` → `if false` | caught by `flipping_any_bit_of_the_wire_form_breaks_or_changes_it` |
| M78 | `extrinsic` | `DecodeError::Eof } else { DecodeError::Trailing` → `DecodeError::Trailing } else { DecodeError::Eof` | caught by `length_prefix_must_match` |
| M79 | `extrinsic` | `u64::try_from(nonce).map_err(\|_\| DecodeError::Overflow)?` → `nonce as u64` | caught by `a_nonce_that_does_not_fit_in_u64_is_an_overflow` |
| M80 | `extrinsic` | `body.push(0); // MultiSignature::Ed25519` → `∅` | caught by `random_calls_round_trip` |
| M81 | `extrinsic` | `if !input.is_empty()` → `if input.len() > 1` | caught by `one_byte_after_the_call_is_trailing_data` |
| M82 | `extrinsic` | `(0, 0` → `(0, _` | caught by `call_indexes_must_match_exactly` |
| M83 | `extrinsic` | `(1, 0` → `(1, _` | caught by `flipping_any_bit_of_the_wire_form_breaks_or_changes_it` |
| M84 | `extrinsic` | `blake2_256(&self.to_bytes` → `blake2_256(&self.call.encode` | caught by `wire_bytes_match_an_independent_python_build` |
| M85 | `runtime` | `…EXISTENTIAL_DEPOSIT: u128 = 10` → `…EXISTENTIAL_DEPOSIT: u128 = 11` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M86 | `runtime` | `…BASE_FEE: u128 = 1` → `…BASE_FEE: u128 = 2` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M87 | `runtime` | `self.nonce.encode_to(out); self.fre` → `self.free.encode_to(out); self.nonc` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M88 | `runtime` | `…"Account", Hasher::Blake2_128` → `…"Account", Hasher::Twox64` | caught by `get_storage_returns_hex_scale` |
| M89 | `runtime` | `blake2_256(&xts.to_vec().encode()` → `blake2_256(&[]` | caught by `extrinsics_root_commits_to_the_included_list` |
| M90 | `runtime` | `Compact(u128::from(self.number)).encode_to(&mut o);` → `∅` | caught by `header_encoding_layout` |
| M91 | `runtime` | `total += amount` → `total += 1` | caught by `a_transaction_after_its_sender_was_reaped_is_dropped` |
| M92 | `runtime` | `c.ctx.genesis = header.hash();` → `∅` | caught by `block_hash_and_header_by_hash` |
| M93 | `runtime` | `buf.extend_from_slice(&k.encode());` → `∅` | caught by `state_root_depends_on_keys_and_on_values` |
| M94 | `runtime` | `buf.extend_from_slice(&v.encode());` → `∅` | caught by `state_root_depends_on_keys_and_on_values` |
| M95 | `runtime` | `.filter(\|(k, _)\| k.starts_with(&prefix)` → `.filter(\|(_, _)\| true` | **equivalent**: without the prefix filter the `TotalIssuance` entry is also tried as an account, but it is 16 bytes and an account needs 20, so the decode fails and the entry is skipped; the filter is right but this test cannot tell, and it would matter the day the two types had the same size |
| M96 | `runtime` | `&s.signer == who && ` → `∅` | caught by `every_rejection_has_its_own_name_over_rpc` |
| M97 | `runtime` | `…s.extra.nonce <` → `…s.extra.nonce <=` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M98 | `runtime` | `…s.extra.nonce >` → `…s.extra.nonce >=` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M99 | `runtime` | `.free < BASE_FEE + s.extra.tip ` → `.free < BASE_FEE ` | caught by `pool_rejections` |
| M100 | `runtime` | `.free <` → `.free <=` | caught by `an_account_that_can_just_pay_the_fee_is_accepted_and_included` |
| M101 | `runtime` | `self.total_issuance() -` → `self.total_issuance() +` | caught by `a_transaction_after_its_sender_was_reaped_is_dropped` |
| M102 | `runtime` | `if u64::from(acc.nonce) != s.extra.nonc` → `if fals` | caught by `a_stale_nonce_after_reap_and_refund_is_not_executed` |
| M103 | `runtime` | `if acc.free <` → `if acc.free <=` | caught by `an_account_that_can_just_pay_the_fee_is_accepted_and_included` |
| M104 | `runtime` | `if acc.free < fe` → `if fals` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M105 | `runtime` | `acc.free -= fee;` → `∅` | caught by `a_transaction_after_its_sender_was_reaped_is_dropped` |
| M106 | `runtime` | `acc.nonce += 1;` → `∅` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M107 | `runtime` | `self.burn(fee);` → `∅` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M108 | `runtime` | `events.push(Event::Failed { who, error });` → `∅` | caught by `existential_deposit_on_the_receiver` |
| M109 | `runtime` | `self.reap(&who, events);` → `∅` | caught by `a_transaction_after_its_sender_was_reaped_is_dropped` |
| M110 | `runtime` | `if dest == who` → `if false` | caught by `conservation_holds_under_random_traffic` |
| M111 | `runtime` | `if from.free <` → `if from.free <=` | caught by `transferring_everything_after_the_fee_works` |
| M112 | `runtime` | `…to.free + value <` → `…to.free + value <=` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M113 | `runtime` | `…to.free += value` → `…to.free += value + 1` | caught by `a_transaction_that_became_unaffordable_is_dropped_not_included` |
| M114 | `runtime` | `if acc.free <` → `if acc.free <=` | caught by `an_account_left_with_exactly_the_deposit_stays` |
| M115 | `runtime` | `if acc.free >` → `if acc.free >=` | **equivalent**: `burn(0)` subtracts nothing, so `>= 0` and `> 0` behave alike |
| M116 | `runtime` | `…events).is_ok() ` → `…events).is_ok() \|\| true ` | caught by `a_transaction_after_its_sender_was_reaped_is_dropped` |
| M117 | `runtime` | `…parent.number + 1` → `…parent.number + 2` | caught by `block_hash_and_header_by_hash` |
| M118 | `runtime` | `parent_hash: parent.hash()` → `parent_hash: [0; 32]` | caught by `blocks_chain_by_hash` |
| M119 | `runtime` | `Ok(h` → `Ok([0; 32]` | caught by `submit_and_produce` |
| M120 | `runtime` | `if !xt.verify(&self.ctx)` → `if false` | caught by `every_rejection_has_its_own_name_over_rpc` |
| M121 | `runtime` | `len: r.len() as u32` → `len: 0` | caught by `remark_costs_only_the_fee` |
| M122 | `rpc` | `…PARSE_ERROR: i64 = -32700` → `…PARSE_ERROR: i64 = -32701` | caught by `json_rpc_error_code_values` |
| M123 | `rpc` | `…INVALID_TX: i64 = 1010` → `…INVALID_TX: i64 = 1011` | caught by `json_rpc_error_code_values` |
| M124 | `rpc` | `format!("0x` → `format!("` | caught by `every_rejection_has_its_own_name_over_rpc` |
| M125 | `rpc` | `if v.get("jsonrpc") != Some(&json!("2.0"))` → `if false` | caught by `protocol_errors` |
| M126 | `rpc` | `Invalid::BadSignature => "BadProof` → `Invalid::BadSignature => "BadSig` | caught by `every_rejection_has_its_own_name_over_rpc` |
| M127 | `rpc` | `format!("0x{:x` → `format!("{` | caught by `block_hash_and_header_by_hash` |
| M128 | `rpc` | `…Value::Null) => chain.best().header.number` → `…Value::Null) => 0` | caught by `block_hash_and_header_by_hash` |
| M129 | `rpc` | `…"extrinsics": b.extrinsics.len()` → `…"extrinsics": 0` | caught by `submit_and_produce` |
| M130 | `rpc` | `…ok(id, json!("Teaching ` → `…ok(id, json!("` | caught by `protocol_errors` |
| M131 | `rpc` | `…=> match n.as_u64(` → `…=> match n.as_i64().map(\|x\| x as u64` | caught by `params_are_validated` |
| M132 | `rpc` | `…).unwrap_or_else(\|\| json!([` → `…).unwrap_or_else(\|\| json!([1` | caught by `missing_params_means_an_empty_list` |

What the table does not show: it tests *these* edits, not every possible bug; the equivalent mutants could hide a real gap if my reasoning about them is wrong; and passing every mutant says nothing about the real stack, about security or about behavior on a node.

**Design notes.**

- Everything that touches bytes goes through `Encode`, `Decode` and `decode_all`, so "refuse trailing data" is written once.
- The state is a `BTreeMap` of real storage keys to SCALE values, so a test can read an account with the same key a client would compute.
- `submit` validates for the pool and `apply` validates again for execution; the second check is not redundant (§7.2, §7.4).
- Signing is deterministic from a seed so that known-answer tests are possible; production code should not derive keys this way.
- Not handled: sr25519 and ecdsa, mortal eras, weights, a trie, events as storage, multiple pallets beyond the two, WebAssembly, networking, consensus.

</details>

---

## 17. Key takeaways

- A Substrate client's job starts with exact bytes: SCALE is untagged, little-endian and compact, and a decoder must accept one encoding per value, refuse hostile lengths and trailing data, and be checked against a second implementation (✓ 3,000 random compact values and 500 nested values against `parity-scale-codec`).
- A storage key is a pure function of names: two `twox128` prefixes and, for maps, a hashed key; the hasher choice is a security decision for attacker-chosen keys (✓ Python known answers).
- SS58 is a spelling with a checksum and a network, not an identity; store 32-byte ids (✓ every single-character edit caught; a bug in the two-byte prefix reader that hid behind small prefixes).
- A signature binds the call, the extra fields and the chain and runtime it was made for; an immortal signature is only as good as the stored nonce (✓ Python-built wire bytes; the replay after reaping).
- A failed call is included and pays; execution re-validates what admission checked; supply must equal balances after every block (✓ 40 random rounds).
- A JSON-RPC front end is a small contract; pin the codes as literals and name every rejection (✓ six methods, five codes).
- Signatures dominate cost; the codec and the hashes are cheap (✓ 19 ns compact encode against 26 µs signing).
- Everything about real consensus, parachains, XCM, metadata-driven clients and real runtimes is from memory here; the model is a way to learn the byte-level contract, not a node.
- Mutation testing here pointed at real gaps: 5 of the 7 first survivors were real gaps and were closed with 5 new tests (a string longer than three bytes, one trailing byte, a call index, a stale nonce after reaping and refunding, missing parameters); 2 mutants remain, each with a stated reason for being equivalent.

**Connections.** [Lesson 48](48_crypto_primitives.md) supplied the hashes and signatures; [Lesson 49](49_merkle_transactions_consensus.md) the Merkle roots and the transaction and consensus vocabulary that §7.5 simplifies; [Lesson 51](51_rpc_nodes_indexers.md) and [Lesson 52](52_web3_reliability.md) are the client habits (finality, reorganizations, idempotency) that §8.1 repeats; [Lesson 53](53_solana_model.md) is the contrast (accounts that hold programs and callers who list every account, against a runtime that owns its state and a client that reads metadata); [Lesson 55](55_solana_security_testing_deploy.md) is where the snapshot, invariant and mutation habits used here were built. Lesson 57 turns to Ethereum-compatible chains from Rust.

---

## 18. References

- Substrate documentation (FRAME, pallets, runtime, the transaction format, the RPC methods): <https://docs.substrate.io/> (I remember this site being reorganized into the Polkadot developer documentation; search for the current location).
- Polkadot documentation (relay chain, parachains, XCM, finality): <https://docs.polkadot.com/>
- The SCALE codec specification and `parity-scale-codec`: <https://github.com/paritytech/parity-scale-codec> and <https://docs.rs/parity-scale-codec>
- The SS58 address format: <https://docs.substrate.io/reference/address-formats/> (the prefix registry is kept in a repository I remember as `paritytech/ss58-registry`).
- `subxt` (typed Rust client generated from metadata): <https://github.com/paritytech/subxt>
- `ed25519-dalek` documentation: <https://docs.rs/ed25519-dalek>; xxHash: <https://xxhash.com/>; BLAKE2 (RFC 7693): <https://www.rfc-editor.org/rfc/rfc7693>
- JSON-RPC 2.0 specification: <https://www.jsonrpc.org/specification>

The URLs and titles above are the standard locations as I remember them; I did not open them during this session (the sandbox cannot reach documentation sites), so if a link has moved, search for the title. **Unchecked or from memory:** everything about the **real** stack: BABE and GRANDPA, the relay chain and parachains, XCM, FRAME's pallet structure, the metadata format, `subxt`, `sp-core` and `polkadot-js`, the exact JSON-RPC method names and result shapes (legacy and `chainHead_v1`), error code 1010, the mortal-era encoding, sr25519, the Merkle-Patricia state trie and the extrinsic format version in force today. **Verified:** only what ran here, namely the SCALE codec against `parity-scale-codec`, the storage prefixes, SS58 addresses, ed25519 key and signature and the extrinsic bytes against separate Python programs, and the behavior of the model. **Environment caveats:** timings are single-machine micro-measurements (three runs of one example) on a shared virtual machine with two cores; nothing here ran on a node or network, so no figure applies to a real chain. Alice's address and the prefixes for the account map and the clock were values I remembered and then reproduced.
