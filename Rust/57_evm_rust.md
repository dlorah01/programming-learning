---
title: "Lesson 57 — EVM interaction from Rust"
---

# Lesson 57 — EVM interaction from Rust

**Module 18 · Other blockchain ecosystems (second lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `hex 0.4.3`, `k256 0.14.0`, `serde_json 1.0.154`, `sha3 0.12.0`, and, for tests only, `alloy-primitives 1.7.3`, `alloy-rlp 0.3.17` and `alloy-sol-types 1.7.3` (274 packages in `Cargo.lock`, the crate itself included)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** The Ethereum Virtual Machine (EVM) is the execution environment shared by Ethereum and a large family of other chains. A Rust program that talks to such a chain does five things well or badly: it computes **Keccak hashes** (selectors, event topics, transaction ids), it speaks **RLP** (the wire format of transactions), it speaks the **contract ABI** (the byte layout of calls, arguments, return values and logs), it builds and **signs transactions** (legacy with replay protection, and the fee-market type 2), and it talks **JSON-RPC** to a node. This lesson builds each of these small enough to read, plus a **toy node** (nonces, gas, base fee, burn and tip, a native token, logs) and an `eth_*` front end, so that every rule has a test. The codecs are cross-checked against the real `alloy-primitives`, `alloy-rlp` and `alloy-sol-types` crates, and the addresses, selectors, signing hashes and wire bytes were also computed by a separate pure-Python program. What I did **not** do: run an EVM, a node, `anvil`, `foundry`, `revm`, `reth`, `geth`, `alloy`'s providers or `ethers-rs`, or connect to any network. There are no smart contracts here: the "token" is native Rust code in the toy node.

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation sites were not. Statements about the real protocol that I did not run are labeled "from memory". The Python known answers (Keccak of the empty string, four selectors, the `Transfer` topic, EIP-55 vectors, three key-to-address pairs, a legacy and a type-2 transaction with their signing hashes, wire bytes and hashes) come from `p57/py/ref.py`, which uses its own Keccak, its own RLP and its own secp256k1 arithmetic, and shares no code with the crate. The Solidity documentation's ABI example is pinned as a literal.

Builds on [Lesson 50](50_networking_determinism.md) (determinism), [Lesson 51](51_rpc_nodes_indexers.md) (JSON-RPC, nodes, indexers), [Lesson 52](52_web3_reliability.md) (finality, retries, idempotency), [Lesson 49](49_merkle_transactions_consensus.md) (transactions, consensus), [Lesson 48](48_crypto_primitives.md) (hashes and signatures), [Lesson 55](55_solana_security_testing_deploy.md) and [Lesson 56](56_substrate_polkadot.md) (the same questions asked of two other chains), [Lesson 39](39_testing.md), [Lesson 40](40_property_testing_fuzzing.md), [Lesson 41](41_benchmarking_profiling.md), [Lesson 14](14_option_result_errors.md), [Lesson 15](15_newtype_typestate.md) and [Lesson 26](26_error_design.md). The next lesson, Lesson 58, appears here only as plain text.

---

## 1. Why this lesson exists

An EVM client is mostly **exact bytes with a checksum of habits around them**. The failures are rarely exotic: a selector computed from `uint` instead of `uint256`, an address compared as text, a quantity sent as `"0x00"`, a fee estimated for the wrong transaction type, a log decoded as a transfer that was really an NFT transfer. Each claim below has an experiment.

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| A selector is the first four bytes of the Keccak-256 of the *canonical* signature | `transfer(address,uint256)` → `a9059cbb`, `balanceOf(address)` → `70a08231`, `approve(address,uint256)` → `095ea7b3`; hashing the sloppy text `transfer(address,uint)` gives `6cb927d8`, a different selector (§3) |
| Keccak-256 here is not NIST SHA3-256 | `keccak256("")` = `c5d24601…a470` from Python; 11 random inputs around the 136-byte block boundary agree with `alloy-primitives` (§3) |
| RLP has exactly one encoding per value | the specification's examples; lengths 54–65,536; 500 random strings and integers agree with `alloy-rlp`; seven non-canonical forms are refused (§4) |
| A length prefix is attacker input | hostile lengths (a list claiming more than the input holds) are refused before any allocation; nesting beyond 32 levels is refused, 32 levels is accepted (§4.3) |
| ABI encoding is head and tail, padded to 32-byte words | the Solidity documentation's example for `f(uint256,uint32[],bytes10,bytes)` matches word for word; 400 random values agree with `alloy-sol-types` (§5) |
| A decoder that accepts what an encoder never produces is a hazard | dirty padding, overlong or out-of-range offsets and lengths are refused; the lenient decoder accepts aliased offsets and trailing junk, the strict one (decode, re-encode, compare) refuses them; 3,000 random garbage inputs never panic (§5.3) |
| An EIP-55 address checksum is a case pattern | four standard vectors; 300 random addresses agree with `alloy-primitives`; **all flips of a single letter's case in one address are rejected** (§6) |
| A transaction's sender is recovered, not stored | the legacy example (nonce 9, 20 gwei, chain 1) and a type-2 example reproduce a Python build byte for byte, including the signing hash and the transaction hash (§7) |
| The chain id is part of what is signed | changing it changes the signing hash; every field of both types does (§7.2) |
| A signature must be low-s and canonical | 600 random signatures are all low-s; high-s, zero r or s, `v` of 27/28 (unprotected) and unknown type bytes are refused; leading zeros of r and s are stripped on the wire and restored on parse (§7.3) |
| Any single bit flip of a transaction breaks it or changes it | all bits of both example transactions (§7.4) |
| Nodes check funds against the worst case, not the expected cost | a transaction whose cap times gas plus value exceeds the balance is refused though it would have been affordable at the real price (§8.2) |
| A failed call is mined and charged | a reverted token transfer costs gas and a nonce and leaves no log; running out of gas uses the whole limit (§8.3) |
| Ether is conserved | in 40 random rounds of traffic, balances plus burned fees equal the initial supply, and so do token balances (§8.5) |
| An `eth_*` front end has a small, testable contract | quantities are minimal hex, error codes are pinned as literals, every rejection has its own text, log filters follow the AND-of-ORs rule (§9) |
| A parser can have a bug of its own | `parse_quantity` accepted `"0x+1"`, because the standard library's radix parser allows a leading plus; found while planning mutants, before the first mutation run (§9.2) |

Total in the practical challenge: one library crate, 8 modules, **93 tests**, a benchmark example and **114 mutants** run against the test suite.

---

## 2. The stack at a glance (from memory)

Nothing in this section was run; it is the map, and §3 onward is the territory.

- The **EVM** is a stack machine that executes bytecode stored at an **account** that has code (a *contract*). Other accounts are controlled by a key (an *externally owned account*, EOA). State is a map from 20-byte addresses to account data (nonce, balance, code, storage). A **transaction** is signed by an EOA; it either creates a contract (empty `to`) or calls an address with calldata and optional value.
- **Gas** meters execution. A transaction names a gas limit; execution consumes gas; unused gas is refunded. Since the London upgrade (**EIP-1559**), each block has a **base fee** that is burned and a transaction pays a **priority fee** (tip) to the block producer on top (§8.2). I remember type-2 transactions as the ones that carry `max_fee_per_gas` and `max_priority_fee_per_gas`.
- **Logs** are the cheap, indexable output of contracts: up to four topics plus data. A node can filter them by block range, emitter and topics (`eth_getLogs`), which is how almost every indexer works (Lesson 51).
- **Rust libraries** I remember: **alloy** (the current general-purpose toolkit: primitives, RLP, ABI types, a `sol!` macro that generates Rust types from Solidity, providers over HTTP/WebSocket, signers), **ethers-rs** (its predecessor, which I remember as being in maintenance mode in favor of alloy), **revm** (an EVM interpreter in Rust, used for simulation and by clients), **reth** (an execution client in Rust). In the wider tooling: **Foundry** (`forge` to build and test Solidity, `cast` for command-line queries, `anvil` for a local node). I used none of these except three `alloy-*` crates as test oracles. Check current documentation for names, versions and APIs.
- **Layer 2s** and other EVM-compatible chains reuse the same transaction and ABI formats with different fee rules, extra transaction types and different finality. Treat "EVM-compatible" as "mostly the same bytes", never as "the same chain".

---

## 3. Keccak, selectors and topics ✓

Ethereum's hash is **Keccak-256**, the original Keccak submission, which differs from the standardized SHA3-256 in its padding byte; the two give different outputs for every input. The `sha3` crate exposes both, under the names `Keccak256` and `Sha3_256`. Using the wrong one is the classic first mistake.

```rust
//! Keccak-256 (the original Keccak padding, not NIST SHA3-256), event topics and selectors.

use sha3::{Digest, Keccak256};

/// Keccak-256 of `data`.
pub fn keccak256(data: &[u8]) -> [u8; 32] {
    Keccak256::digest(data).into()
}

/// The 4-byte function selector of a canonical signature such as `transfer(address,uint256)`.
pub fn selector(signature: &str) -> [u8; 4] {
    let h = keccak256(signature.as_bytes());
    [h[0], h[1], h[2], h[3]]
}

/// The topic 0 of an event with a canonical signature such as `Transfer(address,address,uint256)`.
pub fn event_topic(signature: &str) -> [u8; 32] {
    keccak256(signature.as_bytes())
}
```

- A **function selector** is the first four bytes of `keccak256(signature)`, where the signature is `name(type,type,…)` in **canonical** form: no spaces, no names, and `uint` written `uint256`, `int` written `int256`. ✓ The crate's `selector_of` parses the text, rewrites it canonically and then hashes, so the sloppy spelling cannot reach the hash:

```rust
/// The 4-byte selector of a signature, computed from its canonical form.
pub fn selector_of(sig: &str) -> Result<[u8; 4], AbiError> {
    let c = canonical_signature(sig)?;
    let h = keccak256(c.as_bytes());
    Ok([h[0], h[1], h[2], h[3]])
}
```

- An **event topic** is the full 32-byte Keccak of the event signature; for `Transfer(address,address,uint256)` it begins `ddf252ad` (✓ from Python; the full value is pinned in the tests). Indexed arguments of an event become further topics, left-padded to 32 bytes.

✓ Known answers, computed in Python (not by the code under test): `keccak256("")` = `c5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`; selectors `a9059cbb` (`transfer(address,uint256)`), `70a08231` (`balanceOf(address)`), `095ea7b3` (`approve(address,uint256)`) and `8be65246` (`f(uint256,uint32[],bytes10,bytes)`). The first three I also remembered from real contracts; the Python program reproduced them. The sloppy `transfer(address,uint)` hashes to `6cb927d8` (✓ Python), which no ERC-20 contract answers to: a call built that way is dispatched to the contract's fallback or reverts, depending on the contract (from memory).

---

## 4. RLP ✓

**Recursive Length Prefix** serializes nested lists of byte strings. There are no types, only bytes and lists; integers are big-endian byte strings with **no leading zero** (zero is the empty string). The rule is by the first byte:

| First byte | Meaning |
|---|---|
| `00`–`7f` | a single byte that is its own encoding |
| `80`–`b7` | a byte string of 0–55 bytes; length is `first − 0x80` |
| `b8`–`bf` | a byte string of 56 or more bytes; the next `first − 0xb7` bytes are its big-endian length |
| `c0`–`f7` | a list whose payload is 0–55 bytes; length is `first − 0xc0` |
| `f8`–`ff` | a list whose payload is 56 or more bytes; the next `first − 0xf7` bytes are its length |

### 4.1 The decoder ✓

```rust
impl Item {
    /// The encoding.
    pub fn encode(&self) -> Vec<u8> {
        let mut out = Vec::new();
        self.encode_to(&mut out);
        out
    }

    pub fn encode_to(&self, out: &mut Vec<u8>) {
        match self {
            Item::Bytes(b) if b.len() == 1 && b[0] < 0x80 => out.push(b[0]),
            Item::Bytes(b) => {
                length_prefix(b.len(), 0x80, out);
                out.extend_from_slice(b);
            }
            Item::List(items) => {
                let mut body = Vec::new();
                for i in items {
                    i.encode_to(&mut body);
                }
                length_prefix(body.len(), 0xc0, out);
                out.extend_from_slice(&body);
            }
        }
    }

    /// Reads one item that must use every byte of `bytes`.
    pub fn decode(bytes: &[u8]) -> Result<Item, RlpError> {
        let mut input = bytes;
        let item = read(&mut input, 0)?;
        if input.is_empty() {
            Ok(item)
        } else {
            Err(RlpError::Trailing)
        }
    }

    pub fn as_bytes(&self) -> Result<&[u8], RlpError> {
        match self {
            Item::Bytes(b) => Ok(b),
            Item::List(_) => Err(RlpError::WrongKind),
        }
    }

    pub fn as_list(&self) -> Result<&[Item], RlpError> {
        match self {
            Item::List(l) => Ok(l),
            Item::Bytes(_) => Err(RlpError::WrongKind),
        }
    }

    /// An unsigned integer, canonical: no leading zero byte, at most 8 bytes.
    pub fn as_u64(&self) -> Result<u64, RlpError> {
        let b = self.as_bytes()?;
        if b.len() > 8 || b.first() == Some(&0) {
            return Err(RlpError::BadInteger);
        }
        Ok(b.iter().fold(0u64, |a, x| (a << 8) | u64::from(*x)))
    }

    /// An unsigned integer of up to 16 bytes, canonical.
    pub fn as_u128(&self) -> Result<u128, RlpError> {
        let b = self.as_bytes()?;
        if b.len() > 16 || b.first() == Some(&0) {
            return Err(RlpError::BadInteger);
        }
        Ok(b.iter().fold(0u128, |a, x| (a << 8) | u128::from(*x)))
    }
}
```

```rust
// `len_of_len` is at most 8: the first byte is at most 0xbf or 0xff
fn long_len(input: &mut &[u8], len_of_len: usize) -> Result<usize, RlpError> {
    let b = take(input, len_of_len)?;
    if b[0] == 0 {
        return Err(RlpError::NonCanonical);
    }
    let len = b.iter().fold(0u64, |a, x| (a << 8) | u64::from(*x));
    if len < 56 {
        return Err(RlpError::NonCanonical);
    }
    usize::try_from(len).map_err(|_| RlpError::TooLong)
}
```

```rust
fn read(input: &mut &[u8], depth: usize) -> Result<Item, RlpError> {
    if depth > MAX_DEPTH {
        return Err(RlpError::TooDeep);
    }
    let first = take(input, 1)?[0];
    match first {
        0x00..=0x7f => Ok(Item::Bytes(vec![first])),
        0x80..=0xb7 => {
            let len = usize::from(first - 0x80);
            let b = take(input, len)?;
            if len == 1 && b[0] < 0x80 {
                return Err(RlpError::NonCanonical);
            }
            Ok(Item::Bytes(b.to_vec()))
        }
        0xb8..=0xbf => {
            let len = long_len(input, usize::from(first - 0xb7))?;
            if len > input.len() {
                return Err(RlpError::TooLong);
            }
            Ok(Item::Bytes(take(input, len)?.to_vec()))
        }
        0xc0..=0xf7 => list(input, usize::from(first - 0xc0), depth),
        0xf8..=0xff => {
            let len = long_len(input, usize::from(first - 0xf7))?;
            list(input, len, depth)
        }
    }
}

fn list(input: &mut &[u8], len: usize, depth: usize) -> Result<Item, RlpError> {
    if len > input.len() {
        return Err(RlpError::TooLong);
    }
    let mut body = take(input, len)?;
    let mut items = Vec::new();
    while !body.is_empty() {
        items.push(read(&mut body, depth + 1)?);
    }
    Ok(Item::List(items))
}
```

### 4.2 Canonical decoding ✓

The decoder refuses every second spelling of a value: a one-byte string below `0x80` written with a length prefix (`8105` instead of `05`), a length in long form that would have fit the short form (`b837…` for 55 bytes), a length with a leading zero byte, and an integer with a leading zero byte. The reason is the usual one: if two byte strings decode to the same value, then a **hash over the bytes** (a transaction id) and a **comparison of values** disagree, and the same transaction acquires two ids. ✓ `rlp_refuses_non_canonical_forms` feeds seven such inputs; `rlp_integers_are_canonical` covers integer widths; the gap tests pin the boundary at 55 and 56 and the leading-zero length field.

### 4.3 Hostile input ✓

A list or string whose declared length exceeds the remaining input is refused (`TooLong`) before anything is allocated from it. Nesting is limited to 32 levels (`MAX_DEPTH`): ✓ 32 levels decode, 33 give `TooDeep`, because a decoder without a limit lets a peer choose your stack depth. ✓ `rlp_matches_alloy_for_random_byte_strings_and_integers` compares 500 random inputs with `alloy-rlp`; `rlp_round_trip_of_random_trees` does 400 random trees; 4,000 random byte strings never panic.

---

## 5. The contract ABI ✓

The ABI turns typed values into bytes. Every static value takes **one 32-byte word**: integers are big-endian and left-padded (sign-extended for `int`), addresses are 20 bytes left-padded, `bool` is 0 or 1, `bytesN` is right-padded. Dynamic values (`bytes`, `string`, `T[]`, and any tuple or fixed array that contains one) live in a **tail**: the head holds a word with the **offset** of the tail entry, and the tail starts with a length (or count) word followed by the data right-padded to a multiple of 32.

### 5.1 The types and tokens

```rust
/// An ABI type.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Ty {
    /// `uintN`, N in 8..=256, multiple of 8.
    Uint(usize),
    /// `intN`.
    Int(usize),
    Address,
    Bool,
    /// `bytesN`, N in 1..=32.
    FixedBytes(usize),
    Bytes,
    String,
    /// `T[]`.
    Array(Box<Ty>),
    /// `T[N]`.
    FixedArray(Box<Ty>, usize),
    Tuple(Vec<Ty>),
}

/// A value of an ABI type.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Token {
    /// A big-endian 256-bit unsigned value.
    Uint(Word),
    /// A two's-complement 256-bit signed value (already sign-extended).
    Int(Word),
    Address([u8; 20]),
    Bool(bool),
    FixedBytes(Vec<u8>),
    Bytes(Vec<u8>),
    String(String),
    /// Used for both `T[]` and `T[N]`.
    Array(Vec<Token>),
    Tuple(Vec<Token>),
}
```

### 5.2 Encoding ✓

```rust
fn encode_seq(tys: &[Ty], toks: &[Token]) -> Result<Vec<u8>, AbiError> {
    if tys.len() != toks.len() {
        return Err(AbiError::Mismatch);
    }
    let head_len: usize = tys.iter().map(Ty::head_size).sum();
    let mut head = Vec::new();
    let mut tail = Vec::new();
    for (ty, tok) in tys.iter().zip(toks) {
        let enc = encode_one(ty, tok)?;
        if ty.is_dynamic() {
            head.extend_from_slice(&word_from_u128((head_len + tail.len()) as u128));
            tail.extend(enc);
        } else {
            head.extend(enc);
        }
    }
    head.extend(tail);
    Ok(head)
}

/// `abi.encode(...)` for a list of values: the arguments as one tuple.
pub fn encode(tys: &[Ty], toks: &[Token]) -> Result<Vec<u8>, AbiError> {
    encode_seq(tys, toks)
}

/// Calldata for a call: the selector, then the arguments.
pub fn encode_call(sig: &str, toks: &[Token]) -> Result<Vec<u8>, AbiError> {
    let (_, tys) = parse_signature(sig)?;
    let mut out = selector_of(sig)?.to_vec();
    out.extend(encode_seq(&tys, toks)?);
    Ok(out)
}
```

✓ The Solidity documentation's example, `f(uint256,uint32[],bytes10,bytes)` with `(0x123, [0x456, 0x789], "1234567890", "Hello, world!")`, encodes to ten words, in the order the documentation gives (selector `8be65246`, `0x123`, offset `0x80`, the `bytes10`, offset `0xe0`, count 2, `0x456`, `0x789`, length `0x0d`, the text); the test compares the whole string and decodes it back. 400 random values of mixed types agree with `alloy-sol-types`.

### 5.3 Decoding: lenient and strict ✓

```rust
fn decode_seq(tys: &[Ty], data: &[u8]) -> Result<Vec<Token>, AbiError> {
    let mut pos = 0usize;
    let mut out = Vec::with_capacity(tys.len().min(data.len() / 32 + 1));
    for ty in tys {
        if ty.is_dynamic() {
            let off = read_usize(word_at(data, pos)?)?;
            if off > data.len() {
                return Err(AbiError::BadOffset);
            }
            out.push(decode_one(ty, &data[off..])?);
            pos += 32;
        } else {
            out.push(decode_one(ty, data.get(pos..).ok_or(AbiError::Eof)?)?);
            pos += ty.head_size();
        }
    }
    Ok(out)
}

/// Decodes arguments. Checks bounds and padding, but (like many decoders) does not
/// insist that offsets are in the canonical order, or that there is nothing after the data.
pub fn decode(tys: &[Ty], data: &[u8]) -> Result<Vec<Token>, AbiError> {
    decode_seq(tys, data)
}
```

```rust
/// Decodes and then re-encodes: succeeds only if `data` is exactly the canonical encoding.
pub fn decode_strict(tys: &[Ty], data: &[u8]) -> Result<Vec<Token>, AbiError> {
    let toks = decode_seq(tys, data)?;
    if encode_seq(tys, &toks)? != data {
        return Err(AbiError::NonCanonical);
    }
    Ok(toks)
}
```

Two decoders, on purpose. The **lenient** one is what many libraries do: check bounds and padding, follow offsets, and stop. It accepts two offsets pointing at the same data (aliasing) and bytes after the end, neither of which an honest encoder produces. The **strict** one decodes and **re-encodes**, and requires the same bytes back (`NonCanonical` otherwise). Use the strict one wherever the bytes were signed or hashed, or where a second parser (a contract, a relayer) might read them differently. ✓ `lenient_decode_versus_strict_decode` builds both cases. ✓ Every padding byte is checked: a `bool` of 2, an address with a dirty byte at position 11, a `bytes4` with a dirty byte at position 4, an offset word with a stray high byte, an array claiming 32 elements behind 31 words (`TooLong`, not `Eof`).

```rust
/// Splits calldata into selector and arguments, checking the selector against `sig`.
pub fn decode_call(sig: &str, calldata: &[u8]) -> Result<Vec<Token>, AbiError> {
    let (_, tys) = parse_signature(sig)?;
    if calldata.len() < 4 {
        return Err(AbiError::Eof);
    }
    if calldata[..4] != selector_of(sig)? {
        return Err(AbiError::Mismatch);
    }
    decode_strict(&tys, &calldata[4..])
}
```

`decode_call` checks the four-byte selector against the signature first, then decodes strictly. ✓ 3,000 random garbage inputs through both decoders never panic.

---

## 6. Addresses and EIP-55 ✓

An address is the **last 20 bytes of the Keccak-256 of the 64-byte public key** (the uncompressed point without its `04` tag). **EIP-55** adds a checksum *without changing the length*: the hex digits are lower-cased, hashed with Keccak, and each letter is capitalized if the matching nibble of the hash is 8 or more.

```rust
/// The address of a public key: the last 20 bytes of Keccak-256 of the 64-byte
/// uncompressed key (without its `04` tag).
pub fn address_of(key: &VerifyingKey) -> Address {
    let point = key.to_sec1_point(false);
    let h = keccak256(&point.as_bytes()[1..]);
    h[12..].try_into().expect("20")
}

/// EIP-55: the hex address with capitals where the matching nibble of
/// Keccak-256 of the lower-case text is 8 or more.
pub fn to_checksum(a: &Address) -> String {
    let lower = hex::encode(a);
    let h = keccak256(lower.as_bytes());
    let mut out = String::from("0x");
    for (i, c) in lower.chars().enumerate() {
        let nibble = if i % 2 == 0 {
            h[i / 2] >> 4
        } else {
            h[i / 2] & 0x0f
        };
        out.push(if nibble >= 8 {
            c.to_ascii_uppercase()
        } else {
            c
        });
    }
    out
}

/// Parses `0x` plus 40 hex digits. All-lower-case and all-upper-case texts are accepted
/// unchecked; mixed case must match the checksum.
pub fn parse_address(text: &str) -> Result<Address, AddressError> {
    let body = text.strip_prefix("0x").ok_or(AddressError::BadHex)?;
    if body.len() != 40 {
        return Err(AddressError::BadLength);
    }
    let bytes = hex::decode(body).map_err(|_| AddressError::BadHex)?;
    let a: Address = bytes.try_into().expect("20");
    let has_lower = body.chars().any(|c| c.is_ascii_lowercase());
    let has_upper = body.chars().any(|c| c.is_ascii_uppercase());
    if has_lower && has_upper && to_checksum(&a)[2..] != *body {
        return Err(AddressError::BadChecksum);
    }
    Ok(a)
}
```

- ✓ Four standard vectors (`0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed`, `0xfB6916095ca1df60bB79Ce92cE3Ea74c37c5d359`, `0xdbF03B407c01E7cD3CBea99509d93f8DDDC8C6FB`, `0xD1220A0cf47c7B9Be7A2E6BA89F429762e7b9aDb`) and three key-to-address pairs from Python (the key `0x46…46` gives `0x9d8A62f6…5A4F`, the key `0x07…07` gives `0x4a623166…C569`, the key `1` gives `0x7E5F4552…5Bdf`).
- All-lower and all-upper texts are accepted **unchecked**; only mixed case is verified. So a checksum catches a typo only in a mixed-case spelling, and about **one error in `2^k`** gets through for an address with `k` letters (from memory of the EIP's own estimate; about 15 letters on average, so roughly one in 32,000, my arithmetic).
- ✓ Flipping the case of any one letter in a checksummed address makes it fail (`a_single_wrong_capital_is_caught`, every letter of one address).
- **Compare addresses as 20 bytes, never as text.** Two spellings of one address are two strings.

---

## 7. Transactions ✓

### 7.1 Two formats

A **legacy** transaction is an RLP list `[nonce, gas_price, gas_limit, to, value, data, v, r, s]`. With **EIP-155** replay protection, `v = 35 + 2·chain_id + y_parity`, and the *signing hash* is the Keccak of the RLP of the same list with `[chain_id, 0, 0]` in place of `[v, r, s]`. A `v` of 27 or 28 is the pre-EIP-155 form, which carries no chain id and can be replayed on another chain; this crate refuses it. A **type-2** transaction (EIP-1559) is the byte `0x02` followed by the RLP list `[chain_id, nonce, max_priority_fee, max_fee, gas_limit, to, value, data, access_list, y_parity, r, s]`; its signing hash is the Keccak of `0x02` followed by the RLP of the first nine fields. The transaction **hash** (its id) is the Keccak of the full wire bytes. (EIP-155 and EIP-1559 field orders are from memory; the Python program and `alloy-rlp` agree with the layout in the code.)

```rust
/// Fee fields, which differ by transaction type.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Fees {
    Legacy {
        gas_price: u128,
    },
    Eip1559 {
        max_priority_fee: u128,
        max_fee: u128,
        access_list: AccessList,
    },
}

/// An unsigned transaction. Values are `u128` here; the real fields are 256 bits.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Tx {
    pub chain_id: u64,
    pub nonce: u64,
    pub gas_limit: u64,
    /// `None` creates a contract.
    pub to: Option<Address>,
    pub value: u128,
    pub data: Vec<u8>,
    pub fees: Fees,
}
```

### 7.2 Signing and recovery ✓

```rust
/// The hash that gets signed.
pub fn signing_hash(&self) -> [u8; 32] {
    let (nonce, gas, to, value) = self.body_fields();
    let data = Item::Bytes(self.data.clone());
    match &self.fees {
        Fees::Legacy { gas_price } => {
            // EIP-155 appends (chain_id, 0, 0) so the chain is part of what is signed
            let l = Item::List(vec![
                nonce,
                uint(*gas_price),
                gas,
                to,
                value,
                data,
                uint(u128::from(self.chain_id)),
                uint(0),
                uint(0),
            ]);
            keccak256(&l.encode())
        }
        Fees::Eip1559 {
            max_priority_fee,
            max_fee,
            access_list,
        } => {
            let l = Item::List(vec![
                uint(u128::from(self.chain_id)),
                nonce,
                uint(*max_priority_fee),
                uint(*max_fee),
                gas,
                to,
                value,
                data,
                access_item(access_list),
            ]);
            let mut bytes = vec![TYPE_1559];
            bytes.extend(l.encode());
            keccak256(&bytes)
        }
    }
}
```

```rust
/// Signs with `key`. The nonce is deterministic (RFC 6979). The library returns the low `s`
/// (I checked 2,000 signatures: none was high), and the tests assert it on 600 more, so
/// there is no code here to flip `s`; if a library did not, this is where it would go.
pub fn sign(self, key: &SigningKey) -> SignedTx {
    let (sig, rec) = key.sign_prehash_recoverable(&self.signing_hash());
    let (r, s) = sig.split_bytes();
    SignedTx {
        tx: self,
        sig: Sig {
            y_parity: rec.to_byte() & 1,
            r: r.into(),
            s: s.into(),
        },
    }
}
```

```rust
/// The sender, recovered from the signature.
pub fn sender(&self) -> Result<Address, TxError> {
    let rec = RecoveryId::from_byte(self.sig.y_parity).ok_or(TxError::BadV)?;
    let sig =
        Signature::from_scalars(self.sig.r, self.sig.s).map_err(|_| TxError::ZeroSignature)?;
    let key = VerifyingKey::recover_from_prehash(&self.tx.signing_hash(), &sig, rec)
        .map_err(|_| TxError::Unrecoverable)?;
    Ok(address_of(&key))
}
```

The signature has no "from" field: the sender is **recovered** from `(signing hash, r, s, y_parity)`. ✓ The legacy example (nonce 9, 20 gwei, 21,000 gas, to `0x3535…35`, 1 ether, chain 1, signed with key `0x46…46`) has signing hash `daf5a779…8e53`, raw bytes `f86c0985…6d83` and hash `33469b22…0788`, all reproduced from Python; the type-2 example (chain 1337, nonce 3, tip 2 gwei, cap 30 gwei, an access list with one address and one key, key `0x07…07`) matches too (signing hash `93307d1c…bafe`, hash `bccf57ae…0ece`, `y_parity` 1). ✓ The chain id is part of what is signed; ✓ changing any one field of either type changes the signing hash (`every_field_changes_the_signing_hash`).

### 7.3 Strict parsing ✓

```rust
/// Parses a raw transaction, refusing anything with a second encoding or a weak signature.
pub fn from_raw(raw: &[u8]) -> Result<SignedTx, TxError> {
    match raw.first() {
        None => Err(RlpError::Eof.into()),
        Some(&TYPE_1559) => Self::from_1559(&raw[1..]),
        Some(&b) if b >= 0xc0 => Self::from_legacy(raw),
        Some(&b) => Err(TxError::UnknownType(b)),
    }
}
```

```rust
fn finish(tx: Tx, y_parity: u8, r: &Item, s: &Item) -> Result<SignedTx, TxError> {
    let r = pad32(r)?;
    let s = pad32(s)?;
    if r == [0; 32] || s == [0; 32] {
        return Err(TxError::ZeroSignature);
    }
    if s > HALF_ORDER {
        return Err(TxError::HighS);
    }
    let signed = SignedTx {
        tx,
        sig: Sig { y_parity, r, s },
    };
    signed.sender()?;
    Ok(signed)
}
```

- **Low-s.** For every valid `(r, s)`, `(r, n − s)` is also valid, so one transaction would have two ids; the network rule (EIP-2, from memory) accepts only `s ≤ n/2`. ✓ The crate refuses `s` above the half order, and the boundary is exact (`HALF_ORDER` passes, one more fails). ✓ 600 random signatures from 254 keys are all low-s: the library already returns low-s signatures, and a branch I wrote to flip `s` was dead (0 of 2,000 probes needed it), so I removed it.
- **Leading zeros.** r and s are 256-bit integers, so on the wire they are RLP integers with leading zeros stripped; the parser restores them to 32 bytes and refuses an encoding that still has a leading zero (✓ `a_padded_signature_integer_is_not_canonical`). ✓ `leading_zero_r_or_s_is_stripped_in_the_wire_form_and_restored_on_parse` finds real signatures for which this matters.
- **Everything else refused:** zero r or s, `v` below 35 (27/28 is reported as unprotected), `y_parity` above 1, an unknown type byte, the wrong number of fields (including a three-element access-list entry), a `to` of any length but 0 or 20, trailing bytes.

### 7.4 Every bit counts ✓

✓ `flipping_any_bit_of_a_raw_transaction_breaks_it_or_changes_it`: for every bit of both example transactions, the flipped bytes either fail to parse or parse to a **different** transaction (and sometimes a different recovered sender). No flip yields the same transaction under another spelling.

### 7.5 Gas and fees ✓

```rust
/// Intrinsic gas: 21,000, plus 16 per non-zero and 4 per zero data byte, plus 32,000 for
/// a creation, plus 2,400 per access-list address and 1,900 per key (all from memory).
pub fn intrinsic_gas(&self) -> u64 {
    let mut gas = 21_000u64;
    for b in &self.data {
        gas += if *b == 0 { 4 } else { 16 };
    }
    if self.to.is_none() {
        gas += 32_000;
    }
    if let Fees::Eip1559 { access_list, .. } = &self.fees {
        for (_, keys) in access_list {
            gas += 2_400 + 1_900 * keys.len() as u64;
        }
    }
    gas
}

/// The price per gas actually paid in a block with base fee `base_fee`.
pub fn effective_gas_price(&self, base_fee: u128) -> u128 {
    match &self.fees {
        Fees::Legacy { gas_price } => *gas_price,
        Fees::Eip1559 {
            max_priority_fee,
            max_fee,
            ..
        } => (*max_fee).min(base_fee.saturating_add(*max_priority_fee)),
    }
}
```

- **Intrinsic gas** (from memory of the Yellow Paper and EIPs; ✓ the formulas as written are pinned by tests): 21,000 base, 4 per zero byte and 16 per non-zero byte of data (the 16 is the figure since Istanbul, from memory), 32,000 more for a contract creation, and for each access-list entry 2,400 per address and 1,900 per storage key. A node rejects a transaction whose limit is below this (§8.2).
- **Effective gas price** for type 2: `min(max_fee, base_fee + max_priority_fee)`. A legacy transaction pays its `gas_price` (a legacy transaction also fits this rule with the price as both cap and tip, which is how the fee market treats it, from memory).
- **Worst-case cost** is `fee_cap · gas_limit + value`, which is what the balance must cover at submission (§8.2). The arithmetic saturates instead of overflowing, because the fields are attacker input (✓ `fee_arithmetic_saturates_instead_of_overflowing`).

---

## 8. A toy node ✓

`Node` has accounts (nonce, balance), a pool, blocks, receipts, a **native token** at a fixed address (a stand-in for an ERC-20 contract; there is no bytecode and no EVM), a constant base fee and a coinbase. It is a model of the **rules a client must understand**, not of an execution engine.

### 8.1 Submitting

```rust
/// Validates and queues a raw transaction. Returns its hash.
pub fn submit(&mut self, raw: &[u8]) -> Result<[u8; 32], SubmitError> {
    let stx = SignedTx::from_raw(raw).map_err(|_| SubmitError::Malformed)?;
    if stx.tx.chain_id != self.chain_id {
        return Err(SubmitError::WrongChain);
    }
    let from = stx.sender().map_err(|_| SubmitError::Malformed)?;
    let hash = stx.hash();
    if self.pool.iter().any(|(t, _)| t.hash() == hash) {
        return Err(SubmitError::AlreadyKnown);
    }
    let expected = self.pending_nonce(&from);
    if stx.tx.nonce < expected {
        return Err(SubmitError::NonceTooLow);
    }
    if stx.tx.nonce > expected {
        return Err(SubmitError::NonceTooHigh);
    }
    if stx.tx.gas_limit < stx.tx.intrinsic_gas() {
        return Err(SubmitError::IntrinsicGas);
    }
    if stx.tx.fee_cap() < self.base_fee {
        return Err(SubmitError::FeeCapBelowBaseFee);
    }
    if self.account(&from).balance < stx.tx.max_cost() {
        return Err(SubmitError::InsufficientFunds);
    }
    self.pool.push((stx, from));
    Ok(hash)
}
```

The order of checks matters to the caller, because each rejection tells the client something different: a **wrong chain** is a bug in the client's configuration; **nonce too low** means the transaction (or one with the same nonce) was already mined and the client should look for a receipt; **nonce too high** means a gap, so earlier transactions are missing or still pending; **intrinsic gas**, **fee cap below base fee** and **insufficient funds** can be fixed by re-pricing or waiting. ✓ `the_pool_rejects_for_a_named_reason` checks every reason; ✓ `pending_nonces_are_per_sender` checks that the expected nonce counts only the sender's own pool entries.

### 8.2 Money: base fee, burn, tip ✓

At mining, each transaction pays `gas_used · effective_price`. The part up to `base_fee` is **burned**; the rest, the **tip**, goes to the block producer (the coinbase here). ✓ A plain transfer at cap 30 gwei and tip 2 gwei with base fee 10 gwei pays 12 gwei per gas: 21,000 · 10 gwei burned, 21,000 · 2 gwei to the coinbase (`a_plain_transfer_pays_gas_at_the_effective_price_and_splits_it_into_burn_and_tip`); ✓ a transaction with a cap just above the base fee has its tip cut (`the_fee_cap_limits_the_price`); ✓ **only `gas_used` is charged, not `gas_limit`** (`an_unused_gas_allowance_is_not_charged`: limit 100,000, charged for 21,000). ✓ Funds are checked at submission against the worst case, so a transaction that would actually be affordable can still be refused (`funds_are_checked_against_the_worst_case_not_the_expected_cost`), and a transaction whose balance equals the worst case exactly is mined.

### 8.3 Execution and failure ✓

```rust
fn execute(&mut self, stx: &SignedTx, from: Address, block: u64) -> Option<Receipt> {
    let t = &stx.tx;
    let acct = self.account(&from);
    if acct.nonce != t.nonce || acct.balance < t.max_cost() {
        return None; // no longer valid: dropped, not included
    }
    let price = t.effective_gas_price(self.base_fee);
    let mut logs = Vec::new();
    let mut status = true;
    let mut gas_used = t.intrinsic_gas();
    let mut moves: Option<(Address, u128)> = None;
    match t.to {
        Some(to) if to == self.token => {
            gas_used += TOKEN_CALL_GAS;
            if gas_used > t.gas_limit {
                status = false;
                gas_used = t.gas_limit;
            } else if t.value > 0 {
                // the token has no payable function: value sent to it reverts the call
                status = false;
            } else {
                match self.token_transfer(&from, &t.data) {
                    Some(l) => logs.push(l),
                    None => status = false,
                }
            }
        }
        Some(to) => moves = Some((to, t.value)),
        None => status = false, // creation is not modelled
    }
    let fee = u128::from(gas_used) * price;
    let burn = u128::from(gas_used) * self.base_fee.min(price);
    let tip = fee - burn;
    let a = self.accounts.entry(from).or_default();
    a.nonce += 1;
    a.balance -= fee;
    self.burned += burn;
    self.accounts.entry(self.coinbase).or_default().balance += tip;
    if let (true, Some((to, value))) = (status, moves) {
        self.accounts.entry(from).or_default().balance -= value;
        self.accounts.entry(to).or_default().balance += value;
    }
    Some(Receipt {
        tx_hash: stx.hash(),
        block_number: block,
        from,
        status,
        gas_used,
        effective_gas_price: price,
        logs,
    })
}
```

- A transaction that is **no longer valid at mining time** (its nonce is not the account's nonce, or the balance fell below its worst case) is **dropped**, not included. ✓ `a_transaction_that_became_unaffordable_is_dropped_at_mining`; ✓ `a_dropped_transaction_drops_its_successors`: when nonce 1 is dropped, nonce 2 cannot run, and the account ends with nonce 1.
- A **token transfer** costs intrinsic gas plus a flat 30,000 (this model's number, not the EVM's). Not enough gas: status 0, `gas_used` equals the limit (✓ `running_out_of_gas_uses_all_the_gas_and_reverts`). A balance too small: status 0, **gas still charged, nonce still advanced, no log** (✓ `a_reverted_call_still_costs_gas_and_a_nonce_and_leaves_no_log`). Value sent to the token reverts, and so does a call to a creation (not modelled).
- A reverted call is in the block with `status` 0: **the receipt is the truth, not the submission**. A client that stops at "the node accepted my transaction" has not learned whether it worked.

### 8.4 Logs and filters ✓

```rust
/// Logs of blocks `from..=to` whose emitter and topics match.
pub fn logs(
    &self,
    from: u64,
    to: u64,
    address: Option<&Address>,
    topics: &[Vec<[u8; 32]>],
) -> Vec<(u64, Log)> {
    let mut out = Vec::new();
    for b in &self.blocks {
        if b.number < from || b.number > to {
            continue;
        }
        for h in &b.txs {
            for l in &self.receipts[h].logs {
                if address.is_some_and(|a| *a != l.address) {
                    continue;
                }
                let ok = topics.iter().enumerate().all(|(i, set)| {
                    set.is_empty() || l.topics.get(i).is_some_and(|t| set.contains(t))
                });
                if ok {
                    out.push((b.number, l.clone()));
                }
            }
        }
    }
    out
}
```

A filter is a block range, an optional emitter and a list of topic positions; each position holds a set of allowed values, and **an empty set matches anything**. A log must satisfy every position (AND across positions, OR within one). A log with fewer topics than a position requires does not match. ✓ `logs_are_filtered_by_range_emitter_and_topics` covers each case; ✓ the same filters run through `eth_getLogs` in §9.

```rust
/// Decodes a transfer log strictly: exactly three topics (a four-topic log with the same
/// first topic is an ERC-721 transfer), clean address padding and one 32-byte value.
pub fn decode_transfer(log: &Log) -> Result<Transfer, LogError> {
    if log.topics.first() != Some(&event_topic(TRANSFER_SIG)) {
        return Err(LogError::NotATransfer);
    }
    if log.topics.len() != 3 {
        return Err(LogError::WrongTopicCount);
    }
    let from = topic_address(&log.topics[1])?;
    let to = topic_address(&log.topics[2])?;
    let w: [u8; 32] = log
        .data
        .as_slice()
        .try_into()
        .map_err(|_| LogError::BadData)?;
    let value = word_to_u128(&w).ok_or(LogError::ValueTooLarge)?;
    Ok(Transfer {
        token: log.address,
        from,
        to,
        value,
    })
}
```

`decode_transfer` requires **exactly three topics** (the signature topic and two indexed addresses) and 32 bytes of data. ERC-721's `Transfer` has the same signature text and **four** topics (the token id is indexed), so a decoder that only looks at the first topic reads NFT transfers as token transfers with a wrong value. ✓ `transfer_logs_decode_strictly`.

### 8.5 Conservation ✓

`total_ether` is the sum of balances plus everything burned. ✓ In 40 random rounds of random plain transfers and token transfers, mined in random orders, it equals the initial supply after every block, and token balances sum to their initial total. A conservation test is the cheapest way to catch a fee-split bug.

---

## 9. A JSON-RPC front end ✓

```rust
/// Handles one request text and returns the response text.
pub fn handle(node: &mut Node, request: &str) -> String {
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
    call(node, method, params, id).to_string()
}
```

```rust
fn call(node: &mut Node, method: &str, params: &[Value], id: Value) -> Value {
    let bad = |id: Value| err(id, INVALID_PARAMS, "Invalid params");
    match method {
        "eth_chainId" => ok(id, json!(quantity(u128::from(node.chain_id)))),
        "eth_blockNumber" => ok(id, json!(quantity(u128::from(node.block_number())))),
        "eth_gasPrice" => ok(id, json!(quantity(node.base_fee))),
        "eth_getBalance" => match addr_param(params.first()) {
            Some(a) => ok(id, json!(quantity(node.account(&a).balance))),
            None => bad(id),
        },
        "eth_getTransactionCount" => {
            let Some(a) = addr_param(params.first()) else {
                return bad(id);
            };
            let n = match params.get(1).and_then(Value::as_str) {
                Some("pending") => node.pending_nonce(&a),
                Some("latest") | None => node.account(&a).nonce,
                Some(_) => return bad(id),
            };
            ok(id, json!(quantity(u128::from(n))))
        }
        "eth_sendRawTransaction" => {
            let Some(raw) = params.first().and_then(Value::as_str).and_then(parse_data) else {
                return bad(id);
            };
            match node.submit(&raw) {
                Ok(h) => ok(id, json!(data_hex(&h))),
                Err(e) => err(id, TX_REJECTED, submit_message(e)),
            }
        }
        "eth_call" => {
            let Some(obj) = params.first().and_then(Value::as_object) else {
                return bad(id);
            };
            let Some(to) = obj
                .get("to")
                .and_then(Value::as_str)
                .and_then(|s| parse_address(s).ok())
            else {
                return bad(id);
            };
            let Some(data) = obj
                .get("data")
                .or_else(|| obj.get("input"))
                .and_then(Value::as_str)
                .and_then(parse_data)
            else {
                return bad(id);
            };
            match node.call(&to, &data) {
                Ok(out) => ok(id, json!(data_hex(&out))),
                Err(Reverted) => err(id, EXECUTION_REVERTED, "execution reverted"),
            }
        }
        "eth_getTransactionReceipt" => {
            let Some(h) = params.first().and_then(Value::as_str).and_then(parse_data) else {
                return bad(id);
            };
            let Ok(h) = <[u8; 32]>::try_from(h) else {
                return bad(id);
            };
            match node.receipt(&h) {
                None => ok(id, Value::Null),
                Some(r) => ok(
                    id,
                    json!({
                        "transactionHash": data_hex(&r.tx_hash),
                        "blockNumber": quantity(u128::from(r.block_number)),
                        "from": crate::address::to_checksum(&r.from),
                        "status": if r.status { "0x1" } else { "0x0" },
                        "gasUsed": quantity(u128::from(r.gas_used)),
                        "effectiveGasPrice": quantity(r.effective_gas_price),
                        "logs": r.logs.iter().map(log_json).collect::<Vec<_>>(),
                    }),
                ),
            }
        }
        "eth_getLogs" => {
            let Some(f) = params.first().and_then(Value::as_object) else {
                return bad(id);
            };
            let Some(from) = block_param(node, f.get("fromBlock")) else {
                return bad(id);
            };
            let Some(to) = block_param(node, f.get("toBlock")) else {
                return bad(id);
            };
            let address = match f.get("address") {
                None | Some(Value::Null) => None,
                Some(v) => match v.as_str().and_then(|s| parse_address(s).ok()) {
                    Some(a) => Some(a),
                    None => return bad(id),
                },
            };
            let mut topics: Vec<Vec<[u8; 32]>> = Vec::new();
            if let Some(ts) = f.get("topics") {
                let Some(ts) = ts.as_array() else {
                    return bad(id);
                };
                for t in ts {
                    let set: Option<Vec<[u8; 32]>> = match t {
                        Value::Null => Some(vec![]),
                        Value::String(s) => parse_data(s)
                            .and_then(|b| b.try_into().ok())
                            .map(|x| vec![x]),
                        Value::Array(a) => a
                            .iter()
                            .map(|x| {
                                x.as_str()
                                    .and_then(parse_data)
                                    .and_then(|b| b.try_into().ok())
                            })
                            .collect(),
                        _ => None,
                    };
                    match set {
                        Some(s) => topics.push(s),
                        None => return bad(id),
                    }
                }
            }
            let found = node.logs(from, to, address.as_ref(), &topics);
            ok(
                id,
                json!(
                    found
                        .iter()
                        .map(|(n, l)| {
                            let mut j = log_json(l);
                            j["blockNumber"] = json!(quantity(u128::from(*n)));
                            j
                        })
                        .collect::<Vec<_>>()
                ),
            )
        }
        "evm_mine" => ok(id, json!(quantity(u128::from(node.mine())))),
        _ => err(id, METHOD_NOT_FOUND, "Method not found"),
    }
}
```

Methods: `eth_chainId`, `eth_blockNumber`, `eth_gasPrice`, `eth_getBalance`, `eth_getTransactionCount` (`latest` or `pending`), `eth_sendRawTransaction`, `eth_call`, `eth_getTransactionReceipt`, `eth_getLogs`, and the test-node method `evm_mine`. Real nodes have many more; the names and shapes here are the ones I remember.

### 9.1 Quantities and data

JSON-RPC numbers are **hex strings**: a *quantity* is `0x` plus the minimal digits (`0x0`, `0x1b4`, never `0x00` or `0x`), and *data* is `0x` plus an even number of hex digits. A node that receives `"0x00"` as a quantity may reject it (from memory: geth does; the standard says quantities have no leading zeros), so a client must send the minimal form.

### 9.2 The parser that accepted a plus sign ✓

```rust
/// Parses a quantity strictly: only hex digits (`from_str_radix` alone would accept a
/// leading `+`), no leading zeros, at least one digit.
pub fn parse_quantity(s: &str) -> Option<u128> {
    let digits = s.strip_prefix("0x")?;
    if !digits.bytes().all(|b| b.is_ascii_hexdigit())
        || (digits.len() > 1 && digits.starts_with('0'))
    {
        return None;
    }
    u128::from_str_radix(digits, 16).ok()
}
```

My first `parse_quantity` stripped `0x`, rejected leading zeros and called `u128::from_str_radix(digits, 16)`. That function accepts a **leading `+`** (and `-` for signed types), so `"0x+1"` parsed as 1. I found it while writing mutants, because the mutant that removed the zero check made me list what else the standard parser tolerates. The fix is a digit check before the conversion, and the test pins `0x+1`, `0x-1`, `0x1 ` (a trailing space), `0x1_0` and a non-ASCII digit as errors. The lesson: **a library parser's accepted language is wider than the protocol's.** Check the characters yourself.

### 9.3 Errors ✓

The codes are pinned in the tests as **literals** (`-32700` parse error, `-32600` invalid request, `-32601` method not found, `-32602` invalid params, from the JSON-RPC specification; `-32000` for a transaction the node refuses, `3` for a reverted call, which are conventions of real nodes as I remember them). A test that compares the code with the crate's own constant would not notice if the constant changed. Each pool rejection has its own text, modelled on what real nodes say (`nonce too low`, `nonce too high`, `insufficient funds for gas * price + value`, `intrinsic gas too low`, …; from memory), and ✓ `every_rejection_has_its_own_message` pins them. A failed token transfer has receipt `status` `0x0` and an empty log list (✓ `a_failed_call_has_receipt_status_zero`).

### 9.4 `eth_getLogs` ✓

The `topics` parameter is a list in which each position is `null` (anything), one value, or a list of values (any of them). `fromBlock`/`toBlock` accept a quantity or a tag (`latest`, `pending`, `safe`, `finalized` all mean the head in this model; `earliest` is block 0). ✓ `get_logs_with_filters` covers all three forms, the emitter and the ranges.

---

## 10. Cost ✓

`cargo run --release --example bench`, three runs, per operation (ranges over the three runs; release profile, 2 shared cores, so treat them as orders of magnitude):

| Operation | ns per operation |
|---|---|
| Keccak-256 of 32 bytes | 300–312 |
| Keccak-256 of 1 KiB | 2,310–2,388 |
| RLP decode of a type-2 body | 350–362 |
| ABI encode (the documentation example) | 804–931 |
| ABI `decode_strict` of the same | 135–154 |
| EIP-55 checksum | 497–543 |
| Sign a transaction | 25,255–27,190 |
| Parse a raw transaction and recover its sender | 113,496–143,564 |
| Mine a block of 50 token transfers | 8.85–11.05 ms |

The shape is the one every chain shows: **signature work dominates**. Recovering a sender costs about 120 µs, hundreds of times a hash. The last row is not mining alone: it builds a node, **signs** 50 transactions (about 1.3 ms by the signing row), **submits** them (50 parse-and-recover steps, about 6 ms by the recovery row; my arithmetic, not separately timed) and then mines. So almost all of the 10 ms is signature work, and mining itself, which does not recover again because the pool keeps each sender, is the small remainder. An indexer that re-verifies signatures it does not need to verify is paying for nothing; a service that must verify them should batch and parallelize.

---

## 11. Rust notes

- **Fixed arrays for fixed things.** `[u8; 20]`, `[u8; 32]` and `[u8; 4]` encode their size in the type, so a 19-byte "address" cannot compile. A **newtype** (Lesson 15) around an address would stop an address from being passed where a topic is expected; this crate uses aliases for readability, which is weaker.
- **`Result` with real error enums** (Lesson 26): `RlpError`, `AbiError`, `TxError`, `SubmitError`, `LogError`, `AddressError`. A caller can branch on *why* a transaction was refused. `Reverted` is a unit struct, not `()`, because clippy (rightly) objects to `Result<_, ()>` and because it documents itself.
- **`checked_add`, `saturating_*`, `u128::try_from`** wherever a length or a fee came from outside (`word_at` uses `checked_add` for the end of a slice).
- **The standard library's parsers are permissive** (`from_str_radix` and the leading plus, §9.2). Wrap them.
- **Ownership at the edge.** `sign(self, key)` takes the transaction by value: you cannot sign it and keep editing it by accident; clone first if you need both (a borrow-checker error I hit while writing the tests).
- **Public API of a model is small.** Only what a test or a caller needs is `pub`; `execute`, `token_transfer` and the RLP readers are private.
- **Test oracles as dev-dependencies.** `alloy-*` appear only under `[dev-dependencies]`, so the shipped crate does not depend on what checks it.

---

## 12. Common mistakes

1. Using SHA3-256 where Keccak-256 is meant (§3).
2. Computing a selector from `uint` or from text with spaces and names (§3).
3. Comparing addresses as strings, or trusting an all-lower-case address as checked (§6).
4. Accepting a non-canonical RLP or ABI encoding where the bytes will be hashed or signed (§4.2, §5.3).
5. Allocating from a length you just read (§4.3, §5.3).
6. Signing a legacy transaction without a chain id, so it can be replayed on another chain (§7.1).
7. Treating a high-s signature as valid, so one transaction has two ids (§7.3).
8. Using `gas_limit` instead of `gas_used` to compute the fee (§8.2), or the expected price instead of the cap to check funds (§8.2).
9. Sending a quantity as `"0x00"` or as a decimal number (§9.1).
10. Stopping at "the node accepted the transaction" without reading the receipt's `status` (§8.3).
11. Assuming nonce too low means failure: it often means the transaction already landed (§8.1; Lesson 52).
12. Decoding a log by its first topic only, so ERC-721 transfers look like ERC-20 transfers (§8.4).
13. Reading the best head instead of a finalized one for money decisions (Lesson 52).
14. Assuming `eth_estimateGas` or a simulation result is a promise: state changes before the transaction lands.
15. Calling the toy node "Ethereum". It has no EVM, no contracts, no trie, no uncle blocks, no reorganizations and one constant base fee.

---

## 13. Security notes and honest limits

- This is a **model written from memory**, with codecs that agree with the real `alloy-*` crates on every tested value and with Python on the fixed vectors, and with everything else unverified against a real node. I ran no node, no EVM and no real provider. A clean test run says nothing about a deployed system.
- Values are `u128`, not `u256`: ether amounts above about 3.4·10³⁸ wei (3.4·10²⁰ ether) cannot be represented, and neither can the largest token amounts a contract can emit. Real token logs carry 256-bit values; `decode_transfer` refuses a value above `u128` (`ValueTooLarge`) rather than truncate, which is a limit of the model, not a feature.
- **Private keys**: the tests derive keys from small seeds so that results can be pinned. Never do that outside a test. Real signing keys come from a secure random source, live in an HSM or hardware wallet or a vetted key store, and are never logged.
- **Deterministic signing**: `k256` signs with an RFC 6979 nonce (from memory of the crate's behavior; ✓ the signatures are reproducible in the known-answer tests). A signer that reuses or biases the nonce leaks the key.
- **Replay and phishing**: a signed transaction is a bearer instrument for as long as its nonce and chain are valid. A signature requested on an unfamiliar message may be a transaction in disguise. Typed-data signatures (EIP-712, from memory) exist to make what is signed readable; not covered here.
- **Approvals and allowances** are the most abused feature of token contracts (from memory): a stale unlimited approval is a standing permission. Not modelled.
- **Simulation is not a promise** (§12 item 14), **gas estimates are guesses**, and **the mempool is public**: anything you submit can be seen and reordered before inclusion (front-running and sandwiching, from memory).
- **Reorganizations and finality** (Lesson 52): a receipt from the head can disappear. The model has none.
- Mutation testing measures this crate's tests (§17), not the real stack.
- If you find a vulnerability in a real client or contract, report it privately to its maintainers and give them time to fix it. Never test against a network you do not own or have not been authorized to test; "testnet" does not make someone else's contract fair game.

---

## 14. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Write `rlp_int_len(n)`: how many bytes the RLP encoding of the integer `n` takes, without encoding it.
2. **(Easy)** Write `canonical_text(sig)`: rewrite a function signature so it can be hashed for a selector (`uint` and `int` become `uint256` and `int256`, whole words only, no spaces).
3. **(Easy)** Write `classify(text)` for address texts: checksummed, unchecked (one case), bad checksum, or malformed.
4. **(Medium)** Write `split(gas_used, base_fee, max_fee, max_priority)`: price, total paid, burned and tip for a type-2 transaction, or `None` if the cap is below the base fee.
5. **(Medium)** Write `next_nonce(account_nonce, pending)`: the next usable nonce when some transactions are already pending, given in any order.
6. **(Medium-hard)** Write `matches(filter, topics)`: does a log match an `eth_getLogs` topic filter?
7. **(Hard)** Write `split_static(data)`: the selector and the argument words of calldata for a function whose arguments are all static, or `None` if the length is wrong.
8. **(Hard)** Write `header(input)`: the kind (bytes or list), the payload offset and the payload length of the first RLP item, refusing non-canonical headers, without decoding any children.

<details markdown="1">
<summary>Answers</summary>

1. A possible answer (✓ compiled and run in a scratch crate with tests):

   ```rust
   //! Exercise 1: how many bytes does an integer take in RLP, without encoding it?

   /// The length of the RLP encoding of the integer `n`.
   pub fn rlp_int_len(n: u128) -> usize {
       let bytes = 16 - n.leading_zeros() as usize / 8;
       match (bytes, n) {
           (0, _) => 1,        // zero is the empty string: 0x80
           (1, 0..=0x7f) => 1, // a single byte below 0x80 is its own encoding
           _ => 1 + bytes,     // 0x80 + length, then the bytes (at most 16, so never long form)
       }
   }
   ```

   Zero is the empty string and takes one byte (`80`); a single byte below `0x80` is its own encoding; anything else is one prefix byte plus the bytes (never long form, since an integer here is at most 16 bytes). The test compares the function with the encoder for every power of two up to 2¹²⁷ and its neighbours.
2. A possible answer (✓):

   ```rust
   //! Exercise 2: spell a function signature the way selectors need it.

   /// Rewrites the type aliases `uint` and `int` as `uint256` and `int256`, whole words only,
   /// and removes spaces. Leaves `uint8`, `uint[]`'s element and names like `uinteger` alone.
   pub fn canonical_text(sig: &str) -> String {
       let mut out = String::new();
       let mut word = String::new();
       let flush = |word: &mut String, out: &mut String| {
           out.push_str(match word.as_str() {
               "uint" => "uint256",
               "int" => "int256",
               w => w,
           });
           word.clear();
       };
       for c in sig.chars().filter(|c| !c.is_whitespace()) {
           if c.is_ascii_alphanumeric() || c == '_' || c == '$' {
               word.push(c);
           } else {
               flush(&mut word, &mut out);
               out.push(c);
           }
       }
       flush(&mut word, &mut out);
       out
   }
   ```

   It works on **words**, not substrings: `uint8` and a name like `uinteger` are untouched, `uint[]` becomes `uint256[]`. The test checks it against the crate's `canonical_signature` and shows the trap (`6cb927d8` for the sloppy text, `a9059cbb` for the canonical one).
3. A possible answer (✓):

   ```rust
   //! Exercise 3: classify an address text.
   use evmlab::address::to_checksum;

   #[derive(Debug, PartialEq, Eq)]
   pub enum Kind {
       /// Mixed case and the capitals match EIP-55.
       Checksummed,
       /// All lower or all upper case: well formed, but nothing was checked.
       Unchecked,
       /// Mixed case and the capitals are wrong.
       BadChecksum,
       /// Not `0x` plus 40 hex digits.
       Malformed,
   }

   pub fn classify(text: &str) -> Kind {
       let Some(body) = text.strip_prefix("0x") else {
           return Kind::Malformed;
       };
       if body.len() != 40 || !body.bytes().all(|b| b.is_ascii_hexdigit()) {
           return Kind::Malformed;
       }
       let bytes: [u8; 20] = hex::decode(body).expect("checked").try_into().expect("20");
       let lower = body.bytes().any(|b| b.is_ascii_lowercase());
       let upper = body.bytes().any(|b| b.is_ascii_uppercase());
       if !(lower && upper) {
           Kind::Unchecked
       } else if to_checksum(&bytes)[2..] == *body {
           Kind::Checksummed
       } else {
           Kind::BadChecksum
       }
   }
   ```

   The digits-only address has no letters to check, so it is "unchecked" and is also a valid checksummed address; that is not a bug, it is the nature of the scheme. The test agrees with the crate's `parse_address` for 200 random addresses and every single-letter case flip.
4. A possible answer (✓):

   ```rust
   //! Exercise 4: who gets what from a type-2 transaction.

   #[derive(Debug, PartialEq, Eq)]
   pub struct Split {
       pub price: u128,
       pub paid: u128,
       pub burned: u128,
       pub tip: u128,
   }

   /// `None` if the fee cap is below the base fee (the transaction cannot be included).
   pub fn split(gas_used: u64, base_fee: u128, max_fee: u128, max_priority: u128) -> Option<Split> {
       if max_fee < base_fee {
           return None;
       }
       let priority = max_priority.min(max_fee - base_fee);
       let price = base_fee + priority;
       let gas = u128::from(gas_used);
       Some(Split {
           price,
           paid: gas * price,
           burned: gas * base_fee,
           tip: gas * priority,
       })
   }
   ```

   The priority actually paid is capped by what the cap leaves above the base fee. `paid = burned + tip` always; the test checks that on 2,000 random inputs and checks the price against the crate's `effective_gas_price`.
5. A possible answer (✓):

   ```rust
   //! Exercise 5: the next nonce to use, given what is already pending.

   /// The end of the unbroken run of pending nonces that starts at `account_nonce`.
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

   The next nonce is the end of the unbroken run that starts at the account's nonce: a pending nonce 7 does not help while 6 is missing; duplicates count once; entries below the account's nonce are stale and ignored.
6. A possible answer (✓):

   ```rust
   //! Exercise 6: does a log match an `eth_getLogs` topic filter?

   pub type Topic = [u8; 32];

   /// `filter[i]` is the set allowed at position `i`; an empty set means "anything".
   /// A log with fewer topics than the filter's last non-empty position cannot match.
   pub fn matches(filter: &[Vec<Topic>], topics: &[Topic]) -> bool {
       filter
           .iter()
           .enumerate()
           .all(|(i, set)| set.is_empty() || topics.get(i).is_some_and(|t| set.contains(t)))
   }
   ```

   An empty set is a wildcard; a position beyond the log's topics fails unless its set is empty. The test covers OR within a position, AND across positions, the longer-filter case and the empty log.
7. A possible answer (✓):

   ```rust
   //! Exercise 7: split calldata for a function whose arguments are all static words.

   /// The selector and the argument words, or `None` if the length is not `4 + 32k`.
   pub fn split_static(data: &[u8]) -> Option<([u8; 4], Vec<[u8; 32]>)> {
       if data.len() < 4 || !(data.len() - 4).is_multiple_of(32) {
           return None;
       }
       let selector = data[..4].try_into().ok()?;
       let words = data[4..]
           .chunks_exact(32)
           .map(|c| c.try_into().expect("32"))
           .collect();
       Some((selector, words))
   }
   ```

   The length must be `4 + 32k`; anything else is refused before slicing. It is the first check any decoder of static calldata should make, and the one whose absence has caused out-of-range reads in many languages.
8. A possible answer (✓):

   ```rust
   //! Exercise 8: read an RLP header without decoding the children.

   #[derive(Debug, PartialEq, Eq)]
   pub enum Kind {
       Bytes,
       List,
   }

   /// The kind, the offset where the payload starts and the payload length.
   /// `None` for anything the canonical rules refuse, or a payload that does not fit.
   pub fn header(input: &[u8]) -> Option<(Kind, usize, usize)> {
       let first = *input.first()?;
       let (kind, offset, len) = match first {
           0x00..=0x7f => (Kind::Bytes, 0, 1),
           0x80..=0xb7 => (Kind::Bytes, 1, usize::from(first - 0x80)),
           0xc0..=0xf7 => (Kind::List, 1, usize::from(first - 0xc0)),
           0xb8..=0xbf | 0xf8..=0xff => {
               let base = if first >= 0xf8 { 0xf7 } else { 0xb7 };
               let n = usize::from(first - base);
               let b = input.get(1..1 + n)?;
               if b[0] == 0 {
                   return None;
               }
               let len = usize::try_from(b.iter().fold(0u64, |a, x| (a << 8) | u64::from(*x))).ok()?;
               if len < 56 {
                   return None;
               }
               (
                   if first >= 0xf8 {
                       Kind::List
                   } else {
                       Kind::Bytes
                   },
                   1 + n,
                   len,
               )
           }
       };
       if offset + len > input.len() {
           return None;
       }
       if matches!(kind, Kind::Bytes) && first == 0x81 && input.get(1).is_some_and(|b| *b < 0x80) {
           return None;
       }
       Some((kind, offset, len))
   }
   ```

   It refuses a single byte below `0x80` written with a prefix, a long-form length below 56, a length with a leading zero, and a payload that does not fit the input. The test compares it with the crate's decoder on nine encodings and 20,000 random short inputs.

</details>

---

## 15. Debugging challenge

A developer writes the body of an ABI `bytes` value by hand for a small tool: a length word, the data, then zero padding up to the next multiple of 32. The tool works in all their tests, and then a counterparty's contract rejects calldata for one particular payload with "data too long". Find the bug and fix it.

```rust
/// The faulty program.
pub mod buggy {
    /// The ABI body of a `bytes` value: a length word, then the data padded to a multiple of 32.
    pub fn encode_bytes(data: &[u8]) -> Vec<u8> {
        let mut out = vec![0u8; 24];
        out.extend((data.len() as u64).to_be_bytes());
        out.extend_from_slice(data);
        let pad = 32 - data.len() % 32;
        out.extend(std::iter::repeat_n(0u8, pad));
        out
    }
}
```

<details markdown="1">
<summary>Solution</summary>

The padding is `32 - len % 32`. When the length is **already a multiple of 32** the remainder is 0, so the formula gives 32 and adds a **whole extra zero word**. The encoding is then 32 bytes too long; a strict decoder (or a hash that the other side recomputes) sees trailing data, and this is also why the bug hides: payloads of 1 to 31 bytes, or 33 to 63, work, and the developer's test strings were short and not multiples of 32. Length 0 is affected too (an empty `bytes` gets a padding word it should not have).

✓ The test measures it against the crate's encoder for every length from 0 to 200: the buggy version is wrong for exactly the multiples of 32 (0, 32, 64, 96, 128, 160, 192) and right for the other 194 lengths.

Fixed version (✓ compiled; equal to the crate's encoder for all 201 lengths):

```rust
/// The corrected program.
pub mod fixed {
    /// The ABI body of a `bytes` value: a length word, then the data padded to a multiple of 32.
    pub fn encode_bytes(data: &[u8]) -> Vec<u8> {
        let mut out = vec![0u8; 24];
        out.extend((data.len() as u64).to_be_bytes());
        out.extend_from_slice(data);
        // a length that is already a multiple of 32 needs no padding at all
        let pad = (32 - data.len() % 32) % 32;
        out.extend(std::iter::repeat_n(0u8, pad));
        out
    }
}
```

The general lesson: **"round up to a multiple" has a boundary at the multiple itself**, so test lengths on both sides of it and at zero. `(32 - len % 32) % 32` and `len.div_ceil(32) * 32` are the two safe spellings. When the domain is small enough, loop over all of it.

</details>

---

## 16. Interview questions

1. *What is the difference between Keccak-256 and SHA3-256? Where does Ethereum use Keccak?*
2. *How is a function selector computed? Why `uint256` and not `uint`?*
3. *Explain RLP in two minutes. Why must a decoder reject non-canonical forms?*
4. *Describe the head and tail of the ABI encoding for `f(uint256,bytes)`.*
5. *What does EIP-55 add to an address, and what does it not protect against?*
6. *How does a node know who sent a transaction that has no "from" field?*
7. *What do EIP-155 and the chain id protect against? What about type-2 transactions?*
8. *Why does a node require low-s signatures?*
9. *Explain the base fee, the priority fee and the max fee. Who pays what, and what is burned?*
10. *A transaction is mined with `status` 0. What happened, and what did the sender pay?*
11. *"Nonce too low" and "nonce too high": what does each tell a client to do?*
12. *How do you read ERC-20 transfers for an address without scanning every transaction? What goes wrong with NFTs?*
13. *Why check funds against the worst case, and why is `eth_estimateGas` not a promise?*
14. *How would you test client code for an EVM chain without running a node? What would the tests not tell you?*
15. *You are asked to review a decoder for a binary protocol. What do you check first?*

<details markdown="1">
<summary>Answers</summary>

1. Both use the same permutation; they differ in the padding byte, so outputs differ for every input. Ethereum uses the original Keccak-256 for addresses, selectors, event topics, transaction hashes and storage keys. Calling a SHA3-256 function by mistake compiles and gives plausible-looking wrong answers, so pin a known answer (`keccak256("")` starts `c5d24601`).
2. The first four bytes of the Keccak-256 of the canonical signature text: the function name and the canonical type names in parentheses, no spaces, no argument names. `uint` is an alias of `uint256`; hashing the alias gives a different selector, which no contract implements (✓ `6cb927d8` against `a9059cbb`).
3. RLP encodes byte strings and lists of them with a length prefix chosen by the length (single byte, short string, long string, short list, long list). Integers are big-endian without leading zeros. Each value has one valid encoding, so a decoder that accepts others lets two byte strings mean one value, and a hash over bytes and a comparison of values then disagree.
4. The head has one word per argument: for `uint256` the value, for `bytes` an offset (here `0x40`, counted from the start of the arguments). The tail, at that offset, is a length word and the data padded on the right to a multiple of 32.
5. A mixed-case pattern derived from the Keccak of the lower-case text, so typing errors in a mixed-case address are detected (not certainly: about one in `2^k` slips through for `k` letters). It does not authenticate the owner, it does nothing for an all-lower-case spelling, and a deliberate swap for another valid address is not caught.
6. By recovering the public key from the signature and the signing hash (the signature carries the recovery id, `y_parity`), then hashing the key and taking the last 20 bytes.
7. They put the chain id into the signed data, so a signature made for one chain is not valid on another. Type-2 transactions carry the chain id as a field, signed as well. A legacy transaction with `v` of 27 or 28 has no chain id and can be replayed (from memory: some nodes still accept those unless configured otherwise).
8. For every `(r, s)` the pair `(r, n − s)` also verifies, so a third party could change a transaction's hash without the sender's key. Accepting only low-s fixes the malleability (✓ the crate refuses high-s).
9. The base fee is set by the protocol per block and burned. The sender names a max fee and a max priority fee; the price is `min(max_fee, base_fee + max_priority_fee)`; the burned part is `gas_used · base_fee`, the rest goes to the block producer (✓ §8.2). The sender pays `gas_used · price`, not `gas_limit · price`.
10. The call reverted or ran out of gas: its state changes were undone, but the transaction is in the block, the nonce advanced and the sender paid for the gas used (all of the limit if gas ran out; ✓ §8.3). The receipt, not the send, tells you.
11. Too low: that nonce was already used; the transaction (or a replacement) probably landed; look for the receipt before resending. Too high: a gap; an earlier transaction is missing or pending; wait or fill it. Client-side nonce tracking must handle both (Lesson 52).
12. Query `eth_getLogs` with the `Transfer` topic and the address as the second or third topic (as a 32-byte left-padded value), over block ranges. ERC-721 has the same signature but four topics (the id is indexed); decode by topic count and data size, not by the signature alone (✓ `decode_transfer`).
13. A transaction may land when the price is highest, so the node needs the balance to cover `fee_cap · gas_limit + value`. An estimate runs against current state; state changes before the transaction is mined, so it can fail or cost more.
14. Known answers from an independent source, a second implementation as an oracle, a model node for nonce, fee and log rules, and recorded responses for the RPC layer. This does not test real contracts, real gas costs, the mempool, reorganizations or provider quirks.
15. Every length against the remaining input; every tag and offset; canonical forms; trailing bytes; integer widths and overflow; padding bytes; recursion depth; behavior on truncation; then fuzz it and cross-check with a second implementation.

</details>

---

## 17. Practical challenge: `evmlab`

Build a library crate that:

1. Implements **Keccak-256**, selectors and event topics.
2. Implements **RLP**: items, canonical-only decoding, depth limit, integers.
3. Implements the **contract ABI**: types and tokens, signature parsing and canonicalization, encoding, lenient and strict decoding, calldata decoding.
4. Implements **addresses**: derivation from a public key, EIP-55, strict parsing.
5. Implements **transactions**: legacy (EIP-155) and type 2, signing, sender recovery, strict parsing (low-s, no unprotected `v`), intrinsic gas, effective price.
6. Implements a **toy node** with nonces, base fee, burn and tip, a native token, logs and filters, and an **`eth_*` JSON-RPC front end**; passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, has a benchmark example and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

```toml
[package]
name = "evmlab"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
hex = "0.4.3"
k256 = "0.14.0"
serde_json = "1.0.154"
sha3 = "0.12.0"

[dev-dependencies]
alloy-primitives = "1.7.3"
alloy-rlp = "0.3.17"
alloy-sol-types = "1.7.3"

[profile.dev.package."*"]
opt-level = 3
```

### `src/lib.rs`

```rust
//! A teaching model of the Ethereum client side: Keccak-256 and selectors, RLP,
//! the contract ABI, addresses with EIP-55 checksums, legacy (EIP-155) and
//! EIP-1559 transactions with signing and sender recovery, event logs, and a
//! small in-process node behind a JSON-RPC front end. **A model written from
//! memory of the specifications; not the real stack; not audited; not for
//! production.**

pub mod abi;
pub mod address;
pub mod keccak;
pub mod logs;
pub mod node;
pub mod rlp;
pub mod rpc;
pub mod tx;
```

### `src/keccak.rs`

```rust
//! Keccak-256 (the original Keccak padding, not NIST SHA3-256), event topics and selectors.

use sha3::{Digest, Keccak256};

/// Keccak-256 of `data`.
pub fn keccak256(data: &[u8]) -> [u8; 32] {
    Keccak256::digest(data).into()
}

/// The 4-byte function selector of a canonical signature such as `transfer(address,uint256)`.
pub fn selector(signature: &str) -> [u8; 4] {
    let h = keccak256(signature.as_bytes());
    [h[0], h[1], h[2], h[3]]
}

/// The topic 0 of an event with a canonical signature such as `Transfer(address,address,uint256)`.
pub fn event_topic(signature: &str) -> [u8; 32] {
    keccak256(signature.as_bytes())
}
```

### `src/rlp.rs`

```rust
//! Recursive Length Prefix: the byte-string and list encoding of Ethereum transactions.
//! Decoding is strict: one encoding per value.

/// A decoded RLP value.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Item {
    Bytes(Vec<u8>),
    List(Vec<Item>),
}

/// Why an input is not valid RLP.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum RlpError {
    /// Input ended early.
    Eof,
    /// A valid value was followed by more bytes.
    Trailing,
    /// A second encoding of a value that has a shorter one.
    NonCanonical,
    /// A length that cannot fit in memory or in the input.
    TooLong,
    /// The item was not the kind (bytes or list) the caller needed.
    WrongKind,
    /// An integer with leading zero bytes, or wider than the target.
    BadInteger,
    /// Nesting deeper than the decoder allows.
    TooDeep,
}

/// Longest nesting the decoder accepts.
pub const MAX_DEPTH: usize = 32;

fn length_prefix(len: usize, short: u8, out: &mut Vec<u8>) {
    if len < 56 {
        out.push(short + len as u8);
    } else {
        let be = (len as u64).to_be_bytes();
        let skip = be.iter().take_while(|b| **b == 0).count();
        out.push(short + 55 + (8 - skip) as u8);
        out.extend_from_slice(&be[skip..]);
    }
}

impl Item {
    /// The encoding.
    pub fn encode(&self) -> Vec<u8> {
        let mut out = Vec::new();
        self.encode_to(&mut out);
        out
    }

    pub fn encode_to(&self, out: &mut Vec<u8>) {
        match self {
            Item::Bytes(b) if b.len() == 1 && b[0] < 0x80 => out.push(b[0]),
            Item::Bytes(b) => {
                length_prefix(b.len(), 0x80, out);
                out.extend_from_slice(b);
            }
            Item::List(items) => {
                let mut body = Vec::new();
                for i in items {
                    i.encode_to(&mut body);
                }
                length_prefix(body.len(), 0xc0, out);
                out.extend_from_slice(&body);
            }
        }
    }

    /// Reads one item that must use every byte of `bytes`.
    pub fn decode(bytes: &[u8]) -> Result<Item, RlpError> {
        let mut input = bytes;
        let item = read(&mut input, 0)?;
        if input.is_empty() {
            Ok(item)
        } else {
            Err(RlpError::Trailing)
        }
    }

    pub fn as_bytes(&self) -> Result<&[u8], RlpError> {
        match self {
            Item::Bytes(b) => Ok(b),
            Item::List(_) => Err(RlpError::WrongKind),
        }
    }

    pub fn as_list(&self) -> Result<&[Item], RlpError> {
        match self {
            Item::List(l) => Ok(l),
            Item::Bytes(_) => Err(RlpError::WrongKind),
        }
    }

    /// An unsigned integer, canonical: no leading zero byte, at most 8 bytes.
    pub fn as_u64(&self) -> Result<u64, RlpError> {
        let b = self.as_bytes()?;
        if b.len() > 8 || b.first() == Some(&0) {
            return Err(RlpError::BadInteger);
        }
        Ok(b.iter().fold(0u64, |a, x| (a << 8) | u64::from(*x)))
    }

    /// An unsigned integer of up to 16 bytes, canonical.
    pub fn as_u128(&self) -> Result<u128, RlpError> {
        let b = self.as_bytes()?;
        if b.len() > 16 || b.first() == Some(&0) {
            return Err(RlpError::BadInteger);
        }
        Ok(b.iter().fold(0u128, |a, x| (a << 8) | u128::from(*x)))
    }
}

/// The minimal big-endian bytes of an integer (zero is the empty string).
pub fn uint_bytes(v: u128) -> Vec<u8> {
    let be = v.to_be_bytes();
    let skip = be.iter().take_while(|b| **b == 0).count();
    be[skip..].to_vec()
}

/// An integer as an RLP item.
pub fn uint(v: u128) -> Item {
    Item::Bytes(uint_bytes(v))
}

fn take<'a>(input: &mut &'a [u8], n: usize) -> Result<&'a [u8], RlpError> {
    if input.len() < n {
        return Err(RlpError::Eof);
    }
    let (head, rest) = input.split_at(n);
    *input = rest;
    Ok(head)
}

// `len_of_len` is at most 8: the first byte is at most 0xbf or 0xff
fn long_len(input: &mut &[u8], len_of_len: usize) -> Result<usize, RlpError> {
    let b = take(input, len_of_len)?;
    if b[0] == 0 {
        return Err(RlpError::NonCanonical);
    }
    let len = b.iter().fold(0u64, |a, x| (a << 8) | u64::from(*x));
    if len < 56 {
        return Err(RlpError::NonCanonical);
    }
    usize::try_from(len).map_err(|_| RlpError::TooLong)
}

fn read(input: &mut &[u8], depth: usize) -> Result<Item, RlpError> {
    if depth > MAX_DEPTH {
        return Err(RlpError::TooDeep);
    }
    let first = take(input, 1)?[0];
    match first {
        0x00..=0x7f => Ok(Item::Bytes(vec![first])),
        0x80..=0xb7 => {
            let len = usize::from(first - 0x80);
            let b = take(input, len)?;
            if len == 1 && b[0] < 0x80 {
                return Err(RlpError::NonCanonical);
            }
            Ok(Item::Bytes(b.to_vec()))
        }
        0xb8..=0xbf => {
            let len = long_len(input, usize::from(first - 0xb7))?;
            if len > input.len() {
                return Err(RlpError::TooLong);
            }
            Ok(Item::Bytes(take(input, len)?.to_vec()))
        }
        0xc0..=0xf7 => list(input, usize::from(first - 0xc0), depth),
        0xf8..=0xff => {
            let len = long_len(input, usize::from(first - 0xf7))?;
            list(input, len, depth)
        }
    }
}

fn list(input: &mut &[u8], len: usize, depth: usize) -> Result<Item, RlpError> {
    if len > input.len() {
        return Err(RlpError::TooLong);
    }
    let mut body = take(input, len)?;
    let mut items = Vec::new();
    while !body.is_empty() {
        items.push(read(&mut body, depth + 1)?);
    }
    Ok(Item::List(items))
}
```

### `src/abi.rs`

```rust
//! The contract ABI: how call arguments, return values and event data become bytes.
//! Every value is padded to 32-byte words; dynamic values live in a "tail" and are
//! pointed at by offsets in the "head".

use crate::keccak::keccak256;

/// One 32-byte ABI word.
pub type Word = [u8; 32];

/// An ABI type.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Ty {
    /// `uintN`, N in 8..=256, multiple of 8.
    Uint(usize),
    /// `intN`.
    Int(usize),
    Address,
    Bool,
    /// `bytesN`, N in 1..=32.
    FixedBytes(usize),
    Bytes,
    String,
    /// `T[]`.
    Array(Box<Ty>),
    /// `T[N]`.
    FixedArray(Box<Ty>, usize),
    Tuple(Vec<Ty>),
}

/// A value of an ABI type.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Token {
    /// A big-endian 256-bit unsigned value.
    Uint(Word),
    /// A two's-complement 256-bit signed value (already sign-extended).
    Int(Word),
    Address([u8; 20]),
    Bool(bool),
    FixedBytes(Vec<u8>),
    Bytes(Vec<u8>),
    String(String),
    /// Used for both `T[]` and `T[N]`.
    Array(Vec<Token>),
    Tuple(Vec<Token>),
}

/// Why encoding, decoding or parsing failed.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum AbiError {
    /// Input ended early.
    Eof,
    /// Padding bytes that must be zero are not.
    DirtyPadding,
    /// An offset points outside the data.
    BadOffset,
    /// A length that cannot fit in the data.
    TooLong,
    /// A value does not fit its type (`uint8` holding 256, and so on).
    ValueTooWide,
    /// The token does not match the type.
    Mismatch,
    /// Not a canonical encoding (found by re-encoding).
    NonCanonical,
    /// A bad type or signature string.
    BadSignature,
    /// A string field that is not UTF-8.
    Utf8,
}

/// A word holding a small unsigned number.
pub fn word_from_u128(v: u128) -> Word {
    let mut w = [0u8; 32];
    w[16..].copy_from_slice(&v.to_be_bytes());
    w
}

/// A sign-extended word holding a small signed number.
pub fn word_from_i128(v: i128) -> Word {
    let mut w = if v < 0 { [0xff; 32] } else { [0; 32] };
    w[16..].copy_from_slice(&v.to_be_bytes());
    w
}

/// The value of a word if it fits in 128 bits.
pub fn word_to_u128(w: &Word) -> Option<u128> {
    if w[..16].iter().any(|b| *b != 0) {
        return None;
    }
    Some(u128::from_be_bytes(w[16..].try_into().expect("16")))
}

fn read_usize(w: &[u8]) -> Result<usize, AbiError> {
    if w[..24].iter().any(|b| *b != 0) {
        return Err(AbiError::TooLong);
    }
    usize::try_from(u64::from_be_bytes(w[24..32].try_into().expect("8")))
        .map_err(|_| AbiError::TooLong)
}

impl Ty {
    /// Whether the encoding has a variable size (and so goes in the tail).
    pub fn is_dynamic(&self) -> bool {
        match self {
            Ty::Bytes | Ty::String | Ty::Array(_) => true,
            Ty::FixedArray(t, n) => *n > 0 && t.is_dynamic(),
            Ty::Tuple(ts) => ts.iter().any(Ty::is_dynamic),
            _ => false,
        }
    }

    /// Bytes taken in the head: 32 for a dynamic type, the whole size for a static one.
    pub fn head_size(&self) -> usize {
        if self.is_dynamic() {
            return 32;
        }
        match self {
            Ty::FixedArray(t, n) => n * t.head_size(),
            Ty::Tuple(ts) => ts.iter().map(Ty::head_size).sum(),
            _ => 32,
        }
    }

    /// The canonical spelling used in selectors: `uint256`, never `uint`.
    pub fn canonical(&self) -> String {
        match self {
            Ty::Uint(n) => format!("uint{n}"),
            Ty::Int(n) => format!("int{n}"),
            Ty::Address => "address".into(),
            Ty::Bool => "bool".into(),
            Ty::FixedBytes(n) => format!("bytes{n}"),
            Ty::Bytes => "bytes".into(),
            Ty::String => "string".into(),
            Ty::Array(t) => format!("{}[]", t.canonical()),
            Ty::FixedArray(t, n) => format!("{}[{n}]", t.canonical()),
            Ty::Tuple(ts) => format!(
                "({})",
                ts.iter().map(Ty::canonical).collect::<Vec<_>>().join(",")
            ),
        }
    }
}

fn split_top(s: &str) -> Result<Vec<&str>, AbiError> {
    let mut out = Vec::new();
    let mut depth = 0i32;
    let mut start = 0;
    for (i, c) in s.char_indices() {
        match c {
            '(' => depth += 1,
            ')' => {
                depth -= 1;
                if depth < 0 {
                    return Err(AbiError::BadSignature);
                }
            }
            ',' if depth == 0 => {
                out.push(&s[start..i]);
                start = i + 1;
            }
            _ => {}
        }
    }
    if depth != 0 {
        return Err(AbiError::BadSignature);
    }
    if !s.is_empty() {
        out.push(&s[start..]);
    }
    Ok(out)
}

/// Parses one type such as `uint`, `bytes32`, `(address,uint256)[]` or `uint8[3]`.
pub fn parse_type(s: &str) -> Result<Ty, AbiError> {
    if let Some(inner) = s.strip_suffix("[]") {
        return Ok(Ty::Array(Box::new(parse_type(inner)?)));
    }
    if s.ends_with(']') {
        let open = s.rfind('[').ok_or(AbiError::BadSignature)?;
        let n: usize = s[open + 1..s.len() - 1]
            .parse()
            .map_err(|_| AbiError::BadSignature)?;
        return Ok(Ty::FixedArray(Box::new(parse_type(&s[..open])?), n));
    }
    if let Some(inner) = s.strip_prefix('(').and_then(|r| r.strip_suffix(')')) {
        let parts = split_top(inner)?;
        return Ok(Ty::Tuple(
            parts
                .into_iter()
                .map(parse_type)
                .collect::<Result<_, _>>()?,
        ));
    }
    let bits = |rest: &str| -> Result<usize, AbiError> {
        if rest.is_empty() {
            return Ok(256);
        }
        if rest.starts_with('0') {
            return Err(AbiError::BadSignature);
        }
        let n: usize = rest.parse().map_err(|_| AbiError::BadSignature)?;
        if n == 0 || n > 256 || !n.is_multiple_of(8) {
            return Err(AbiError::BadSignature);
        }
        Ok(n)
    };
    match s {
        "address" => Ok(Ty::Address),
        "bool" => Ok(Ty::Bool),
        "bytes" => Ok(Ty::Bytes),
        "string" => Ok(Ty::String),
        _ => {
            if let Some(r) = s.strip_prefix("uint") {
                return Ok(Ty::Uint(bits(r)?));
            }
            if let Some(r) = s.strip_prefix("int") {
                return Ok(Ty::Int(bits(r)?));
            }
            if let Some(r) = s.strip_prefix("bytes") {
                let n: usize = r.parse().map_err(|_| AbiError::BadSignature)?;
                if (1..=32).contains(&n) && !r.starts_with('0') {
                    return Ok(Ty::FixedBytes(n));
                }
            }
            Err(AbiError::BadSignature)
        }
    }
}

/// Parses `name(type,type,...)` into the name and its argument types.
pub fn parse_signature(sig: &str) -> Result<(String, Vec<Ty>), AbiError> {
    let open = sig.find('(').ok_or(AbiError::BadSignature)?;
    let name = &sig[..open];
    let valid = !name.is_empty()
        && !name.starts_with(|c: char| c.is_ascii_digit())
        && name
            .chars()
            .all(|c| c.is_ascii_alphanumeric() || c == '_' || c == '$');
    if !valid || !sig.ends_with(')') {
        return Err(AbiError::BadSignature);
    }
    let inner = &sig[open + 1..sig.len() - 1];
    let tys = split_top(inner)?
        .into_iter()
        .map(parse_type)
        .collect::<Result<Vec<_>, _>>()?;
    Ok((name.to_string(), tys))
}

/// The canonical signature string of a possibly sloppy one (`uint` becomes `uint256`).
pub fn canonical_signature(sig: &str) -> Result<String, AbiError> {
    let (name, tys) = parse_signature(sig)?;
    Ok(format!(
        "{name}({})",
        tys.iter().map(Ty::canonical).collect::<Vec<_>>().join(",")
    ))
}

/// The 4-byte selector of a signature, computed from its canonical form.
pub fn selector_of(sig: &str) -> Result<[u8; 4], AbiError> {
    let c = canonical_signature(sig)?;
    let h = keccak256(c.as_bytes());
    Ok([h[0], h[1], h[2], h[3]])
}

fn pad_right(data: &[u8], out: &mut Vec<u8>) {
    out.extend_from_slice(data);
    let rem = data.len() % 32;
    if rem != 0 {
        out.extend(std::iter::repeat_n(0u8, 32 - rem));
    }
}

fn encode_one(ty: &Ty, tok: &Token) -> Result<Vec<u8>, AbiError> {
    let mut out = Vec::new();
    match (ty, tok) {
        (Ty::Uint(bits), Token::Uint(w)) => {
            let keep = bits / 8;
            if w[..32 - keep].iter().any(|b| *b != 0) {
                return Err(AbiError::ValueTooWide);
            }
            out.extend_from_slice(w);
        }
        (Ty::Int(bits), Token::Int(w)) => {
            let keep = bits / 8;
            let ext = if w[32 - keep] & 0x80 != 0 { 0xff } else { 0x00 };
            if w[..32 - keep].iter().any(|b| *b != ext) {
                return Err(AbiError::ValueTooWide);
            }
            out.extend_from_slice(w);
        }
        (Ty::Address, Token::Address(a)) => {
            out.extend_from_slice(&[0; 12]);
            out.extend_from_slice(a);
        }
        (Ty::Bool, Token::Bool(b)) => {
            out.extend_from_slice(&word_from_u128(u128::from(*b)));
        }
        (Ty::FixedBytes(n), Token::FixedBytes(b)) => {
            if b.len() != *n {
                return Err(AbiError::Mismatch);
            }
            pad_right(b, &mut out);
        }
        (Ty::Bytes, Token::Bytes(b)) => {
            out.extend_from_slice(&word_from_u128(b.len() as u128));
            pad_right(b, &mut out);
        }
        (Ty::String, Token::String(s)) => {
            out.extend_from_slice(&word_from_u128(s.len() as u128));
            pad_right(s.as_bytes(), &mut out);
        }
        (Ty::Array(t), Token::Array(items)) => {
            out.extend_from_slice(&word_from_u128(items.len() as u128));
            let tys = vec![(**t).clone(); items.len()];
            out.extend(encode_seq(&tys, items)?);
        }
        (Ty::FixedArray(t, n), Token::Array(items)) => {
            if items.len() != *n {
                return Err(AbiError::Mismatch);
            }
            let tys = vec![(**t).clone(); *n];
            out.extend(encode_seq(&tys, items)?);
        }
        (Ty::Tuple(ts), Token::Tuple(items)) => {
            if ts.len() != items.len() {
                return Err(AbiError::Mismatch);
            }
            out.extend(encode_seq(ts, items)?);
        }
        _ => return Err(AbiError::Mismatch),
    }
    Ok(out)
}

fn encode_seq(tys: &[Ty], toks: &[Token]) -> Result<Vec<u8>, AbiError> {
    if tys.len() != toks.len() {
        return Err(AbiError::Mismatch);
    }
    let head_len: usize = tys.iter().map(Ty::head_size).sum();
    let mut head = Vec::new();
    let mut tail = Vec::new();
    for (ty, tok) in tys.iter().zip(toks) {
        let enc = encode_one(ty, tok)?;
        if ty.is_dynamic() {
            head.extend_from_slice(&word_from_u128((head_len + tail.len()) as u128));
            tail.extend(enc);
        } else {
            head.extend(enc);
        }
    }
    head.extend(tail);
    Ok(head)
}

/// `abi.encode(...)` for a list of values: the arguments as one tuple.
pub fn encode(tys: &[Ty], toks: &[Token]) -> Result<Vec<u8>, AbiError> {
    encode_seq(tys, toks)
}

/// Calldata for a call: the selector, then the arguments.
pub fn encode_call(sig: &str, toks: &[Token]) -> Result<Vec<u8>, AbiError> {
    let (_, tys) = parse_signature(sig)?;
    let mut out = selector_of(sig)?.to_vec();
    out.extend(encode_seq(&tys, toks)?);
    Ok(out)
}

fn word_at(data: &[u8], pos: usize) -> Result<&[u8], AbiError> {
    data.get(pos..pos.checked_add(32).ok_or(AbiError::Eof)?)
        .ok_or(AbiError::Eof)
}

fn decode_one(ty: &Ty, data: &[u8]) -> Result<Token, AbiError> {
    match ty {
        Ty::Uint(bits) => {
            let w: Word = word_at(data, 0)?.try_into().expect("32");
            if w[..32 - bits / 8].iter().any(|b| *b != 0) {
                return Err(AbiError::DirtyPadding);
            }
            Ok(Token::Uint(w))
        }
        Ty::Int(bits) => {
            let w: Word = word_at(data, 0)?.try_into().expect("32");
            let keep = bits / 8;
            let ext = if w[32 - keep] & 0x80 != 0 { 0xff } else { 0x00 };
            if w[..32 - keep].iter().any(|b| *b != ext) {
                return Err(AbiError::DirtyPadding);
            }
            Ok(Token::Int(w))
        }
        Ty::Address => {
            let w = word_at(data, 0)?;
            if w[..12].iter().any(|b| *b != 0) {
                return Err(AbiError::DirtyPadding);
            }
            Ok(Token::Address(w[12..].try_into().expect("20")))
        }
        Ty::Bool => {
            let w = word_at(data, 0)?;
            if w[..31].iter().any(|b| *b != 0) || w[31] > 1 {
                return Err(AbiError::DirtyPadding);
            }
            Ok(Token::Bool(w[31] == 1))
        }
        Ty::FixedBytes(n) => {
            let w = word_at(data, 0)?;
            if w[*n..].iter().any(|b| *b != 0) {
                return Err(AbiError::DirtyPadding);
            }
            Ok(Token::FixedBytes(w[..*n].to_vec()))
        }
        Ty::Bytes | Ty::String => {
            let len = read_usize(word_at(data, 0)?)?;
            if len > data.len() - 32 {
                return Err(AbiError::TooLong);
            }
            let padded = len.div_ceil(32) * 32;
            let body = data.get(32..32 + padded).ok_or(AbiError::Eof)?;
            if body[len..].iter().any(|b| *b != 0) {
                return Err(AbiError::DirtyPadding);
            }
            let bytes = body[..len].to_vec();
            if matches!(ty, Ty::Bytes) {
                Ok(Token::Bytes(bytes))
            } else {
                Ok(Token::String(
                    String::from_utf8(bytes).map_err(|_| AbiError::Utf8)?,
                ))
            }
        }
        Ty::Array(t) => {
            let n = read_usize(word_at(data, 0)?)?;
            // every element takes at least 32 bytes in the head, so more cannot fit
            if n > (data.len() - 32) / 32 {
                return Err(AbiError::TooLong);
            }
            let tys = vec![(**t).clone(); n];
            Ok(Token::Array(decode_seq(&tys, &data[32..])?))
        }
        Ty::FixedArray(t, n) => {
            let tys = vec![(**t).clone(); *n];
            Ok(Token::Array(decode_seq(&tys, data)?))
        }
        Ty::Tuple(ts) => Ok(Token::Tuple(decode_seq(ts, data)?)),
    }
}

fn decode_seq(tys: &[Ty], data: &[u8]) -> Result<Vec<Token>, AbiError> {
    let mut pos = 0usize;
    let mut out = Vec::with_capacity(tys.len().min(data.len() / 32 + 1));
    for ty in tys {
        if ty.is_dynamic() {
            let off = read_usize(word_at(data, pos)?)?;
            if off > data.len() {
                return Err(AbiError::BadOffset);
            }
            out.push(decode_one(ty, &data[off..])?);
            pos += 32;
        } else {
            out.push(decode_one(ty, data.get(pos..).ok_or(AbiError::Eof)?)?);
            pos += ty.head_size();
        }
    }
    Ok(out)
}

/// Decodes arguments. Checks bounds and padding, but (like many decoders) does not
/// insist that offsets are in the canonical order, or that there is nothing after the data.
pub fn decode(tys: &[Ty], data: &[u8]) -> Result<Vec<Token>, AbiError> {
    decode_seq(tys, data)
}

/// Decodes and then re-encodes: succeeds only if `data` is exactly the canonical encoding.
pub fn decode_strict(tys: &[Ty], data: &[u8]) -> Result<Vec<Token>, AbiError> {
    let toks = decode_seq(tys, data)?;
    if encode_seq(tys, &toks)? != data {
        return Err(AbiError::NonCanonical);
    }
    Ok(toks)
}

/// Splits calldata into selector and arguments, checking the selector against `sig`.
pub fn decode_call(sig: &str, calldata: &[u8]) -> Result<Vec<Token>, AbiError> {
    let (_, tys) = parse_signature(sig)?;
    if calldata.len() < 4 {
        return Err(AbiError::Eof);
    }
    if calldata[..4] != selector_of(sig)? {
        return Err(AbiError::Mismatch);
    }
    decode_strict(&tys, &calldata[4..])
}
```

### `src/address.rs`

```rust
//! Keys, addresses and EIP-55 checksums.

use crate::keccak::keccak256;
use k256::ecdsa::{SigningKey, VerifyingKey};

/// A 20-byte account address.
pub type Address = [u8; 20];

/// Why an address text was rejected.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AddressError {
    BadLength,
    BadHex,
    /// Mixed-case text whose capitals do not match the checksum.
    BadChecksum,
}

/// The address of a public key: the last 20 bytes of Keccak-256 of the 64-byte
/// uncompressed key (without its `04` tag).
pub fn address_of(key: &VerifyingKey) -> Address {
    let point = key.to_sec1_point(false);
    let h = keccak256(&point.as_bytes()[1..]);
    h[12..].try_into().expect("20")
}

/// A signing key from 32 secret bytes, or `None` for zero or a value at or above the curve order.
pub fn signing_key(secret: &[u8; 32]) -> Option<SigningKey> {
    SigningKey::from_bytes(secret.into()).ok()
}

/// EIP-55: the hex address with capitals where the matching nibble of
/// Keccak-256 of the lower-case text is 8 or more.
pub fn to_checksum(a: &Address) -> String {
    let lower = hex::encode(a);
    let h = keccak256(lower.as_bytes());
    let mut out = String::from("0x");
    for (i, c) in lower.chars().enumerate() {
        let nibble = if i % 2 == 0 {
            h[i / 2] >> 4
        } else {
            h[i / 2] & 0x0f
        };
        out.push(if nibble >= 8 {
            c.to_ascii_uppercase()
        } else {
            c
        });
    }
    out
}

/// Parses `0x` plus 40 hex digits. All-lower-case and all-upper-case texts are accepted
/// unchecked; mixed case must match the checksum.
pub fn parse_address(text: &str) -> Result<Address, AddressError> {
    let body = text.strip_prefix("0x").ok_or(AddressError::BadHex)?;
    if body.len() != 40 {
        return Err(AddressError::BadLength);
    }
    let bytes = hex::decode(body).map_err(|_| AddressError::BadHex)?;
    let a: Address = bytes.try_into().expect("20");
    let has_lower = body.chars().any(|c| c.is_ascii_lowercase());
    let has_upper = body.chars().any(|c| c.is_ascii_uppercase());
    if has_lower && has_upper && to_checksum(&a)[2..] != *body {
        return Err(AddressError::BadChecksum);
    }
    Ok(a)
}
```

### `src/logs.rs`

```rust
//! Event logs and the decoding of an ERC-20 style `Transfer`.

use crate::abi::{word_from_u128, word_to_u128};
use crate::address::Address;
use crate::keccak::event_topic;

/// A log entry: who emitted it, up to four topics, and unindexed data.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Log {
    pub address: Address,
    pub topics: Vec<[u8; 32]>,
    pub data: Vec<u8>,
}

/// A decoded `Transfer(address indexed from, address indexed to, uint256 value)`.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Transfer {
    pub token: Address,
    pub from: Address,
    pub to: Address,
    pub value: u128,
}

/// Why a log is not a well-formed transfer.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum LogError {
    NotATransfer,
    WrongTopicCount,
    DirtyTopic,
    BadData,
    ValueTooLarge,
}

pub const TRANSFER_SIG: &str = "Transfer(address,address,uint256)";

fn topic_address(t: &[u8; 32]) -> Result<Address, LogError> {
    if t[..12].iter().any(|b| *b != 0) {
        return Err(LogError::DirtyTopic);
    }
    Ok(t[12..].try_into().expect("20"))
}

/// An address as a 32-byte topic.
pub fn address_topic(a: &Address) -> [u8; 32] {
    let mut t = [0u8; 32];
    t[12..].copy_from_slice(a);
    t
}

/// Builds the log of a transfer.
pub fn transfer_log(token: Address, from: Address, to: Address, value: u128) -> Log {
    Log {
        address: token,
        topics: vec![
            event_topic(TRANSFER_SIG),
            address_topic(&from),
            address_topic(&to),
        ],
        data: word_from_u128(value).to_vec(),
    }
}

/// Decodes a transfer log strictly: exactly three topics (a four-topic log with the same
/// first topic is an ERC-721 transfer), clean address padding and one 32-byte value.
pub fn decode_transfer(log: &Log) -> Result<Transfer, LogError> {
    if log.topics.first() != Some(&event_topic(TRANSFER_SIG)) {
        return Err(LogError::NotATransfer);
    }
    if log.topics.len() != 3 {
        return Err(LogError::WrongTopicCount);
    }
    let from = topic_address(&log.topics[1])?;
    let to = topic_address(&log.topics[2])?;
    let w: [u8; 32] = log
        .data
        .as_slice()
        .try_into()
        .map_err(|_| LogError::BadData)?;
    let value = word_to_u128(&w).ok_or(LogError::ValueTooLarge)?;
    Ok(Transfer {
        token: log.address,
        from,
        to,
        value,
    })
}
```

### `src/tx.rs`

```rust
//! Transactions: legacy with replay protection (EIP-155) and type 2 (EIP-1559),
//! their signing hashes, signatures with sender recovery, and strict parsing.

use crate::address::{Address, address_of};
use crate::keccak::keccak256;
use crate::rlp::{Item, RlpError, uint};
use k256::ecdsa::{RecoveryId, Signature, SigningKey, VerifyingKey};

/// Half the order of secp256k1: signatures with `s` above it are refused (EIP-2).
const HALF_ORDER: [u8; 32] = [
    0x7f, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff,
    0x5d, 0x57, 0x6e, 0x73, 0x57, 0xa4, 0x50, 0x1d, 0xdf, 0xe9, 0x2f, 0x46, 0x68, 0x1b, 0x20, 0xa0,
];

/// The type byte of an EIP-1559 transaction.
pub const TYPE_1559: u8 = 0x02;

/// `(address, storage keys)` pairs of an access list.
pub type AccessList = Vec<(Address, Vec<[u8; 32]>)>;

/// Fee fields, which differ by transaction type.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Fees {
    Legacy {
        gas_price: u128,
    },
    Eip1559 {
        max_priority_fee: u128,
        max_fee: u128,
        access_list: AccessList,
    },
}

/// An unsigned transaction. Values are `u128` here; the real fields are 256 bits.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Tx {
    pub chain_id: u64,
    pub nonce: u64,
    pub gas_limit: u64,
    /// `None` creates a contract.
    pub to: Option<Address>,
    pub value: u128,
    pub data: Vec<u8>,
    pub fees: Fees,
}

/// A recoverable ECDSA signature.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Sig {
    /// 0 or 1: the parity of the y coordinate of the nonce point.
    pub y_parity: u8,
    pub r: [u8; 32],
    pub s: [u8; 32],
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct SignedTx {
    pub tx: Tx,
    pub sig: Sig,
}

/// Why a raw transaction was refused.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TxError {
    Rlp(RlpError),
    /// A type byte this model does not know.
    UnknownType(u8),
    /// The wrong number of fields.
    FieldCount,
    /// A field of the wrong size (the `to` address, `r`, `s`).
    BadField,
    /// `v` of a legacy transaction without chain id (replayable on any chain).
    Unprotected,
    /// A `v` or `y_parity` that is not valid.
    BadV,
    /// `r` or `s` is zero.
    ZeroSignature,
    /// `s` is in the upper half of the group order.
    HighS,
    /// The signature does not recover a public key.
    Unrecoverable,
}

impl From<RlpError> for TxError {
    fn from(e: RlpError) -> Self {
        TxError::Rlp(e)
    }
}

fn to_item(a: &Option<Address>) -> Item {
    Item::Bytes(a.map(|x| x.to_vec()).unwrap_or_default())
}

fn access_item(al: &AccessList) -> Item {
    Item::List(
        al.iter()
            .map(|(a, keys)| {
                Item::List(vec![
                    Item::Bytes(a.to_vec()),
                    Item::List(keys.iter().map(|k| Item::Bytes(k.to_vec())).collect()),
                ])
            })
            .collect(),
    )
}

fn strip(b: &[u8; 32]) -> Item {
    let skip = b.iter().take_while(|x| **x == 0).count();
    Item::Bytes(b[skip..].to_vec())
}

fn pad32(item: &Item) -> Result<[u8; 32], TxError> {
    let b = item.as_bytes()?;
    if b.len() > 32 || b.first() == Some(&0) {
        return Err(TxError::BadField);
    }
    let mut out = [0u8; 32];
    out[32 - b.len()..].copy_from_slice(b);
    Ok(out)
}

impl Tx {
    /// The RLP fields every type shares between `nonce` and `data`, minus fees.
    fn body_fields(&self) -> (Item, Item, Item, Item) {
        (
            uint(u128::from(self.nonce)),
            uint(u128::from(self.gas_limit)),
            to_item(&self.to),
            uint(self.value),
        )
    }

    /// The hash that gets signed.
    pub fn signing_hash(&self) -> [u8; 32] {
        let (nonce, gas, to, value) = self.body_fields();
        let data = Item::Bytes(self.data.clone());
        match &self.fees {
            Fees::Legacy { gas_price } => {
                // EIP-155 appends (chain_id, 0, 0) so the chain is part of what is signed
                let l = Item::List(vec![
                    nonce,
                    uint(*gas_price),
                    gas,
                    to,
                    value,
                    data,
                    uint(u128::from(self.chain_id)),
                    uint(0),
                    uint(0),
                ]);
                keccak256(&l.encode())
            }
            Fees::Eip1559 {
                max_priority_fee,
                max_fee,
                access_list,
            } => {
                let l = Item::List(vec![
                    uint(u128::from(self.chain_id)),
                    nonce,
                    uint(*max_priority_fee),
                    uint(*max_fee),
                    gas,
                    to,
                    value,
                    data,
                    access_item(access_list),
                ]);
                let mut bytes = vec![TYPE_1559];
                bytes.extend(l.encode());
                keccak256(&bytes)
            }
        }
    }

    /// Signs with `key`. The nonce is deterministic (RFC 6979). The library returns the low `s`
    /// (I checked 2,000 signatures: none was high), and the tests assert it on 600 more, so
    /// there is no code here to flip `s`; if a library did not, this is where it would go.
    pub fn sign(self, key: &SigningKey) -> SignedTx {
        let (sig, rec) = key.sign_prehash_recoverable(&self.signing_hash());
        let (r, s) = sig.split_bytes();
        SignedTx {
            tx: self,
            sig: Sig {
                y_parity: rec.to_byte() & 1,
                r: r.into(),
                s: s.into(),
            },
        }
    }

    /// Intrinsic gas: 21,000, plus 16 per non-zero and 4 per zero data byte, plus 32,000 for
    /// a creation, plus 2,400 per access-list address and 1,900 per key (all from memory).
    pub fn intrinsic_gas(&self) -> u64 {
        let mut gas = 21_000u64;
        for b in &self.data {
            gas += if *b == 0 { 4 } else { 16 };
        }
        if self.to.is_none() {
            gas += 32_000;
        }
        if let Fees::Eip1559 { access_list, .. } = &self.fees {
            for (_, keys) in access_list {
                gas += 2_400 + 1_900 * keys.len() as u64;
            }
        }
        gas
    }

    /// The price per gas actually paid in a block with base fee `base_fee`.
    pub fn effective_gas_price(&self, base_fee: u128) -> u128 {
        match &self.fees {
            Fees::Legacy { gas_price } => *gas_price,
            Fees::Eip1559 {
                max_priority_fee,
                max_fee,
                ..
            } => (*max_fee).min(base_fee.saturating_add(*max_priority_fee)),
        }
    }

    /// The highest price per gas the sender accepts.
    pub fn fee_cap(&self) -> u128 {
        match &self.fees {
            Fees::Legacy { gas_price } => *gas_price,
            Fees::Eip1559 { max_fee, .. } => *max_fee,
        }
    }

    /// The most the sender may be charged: gas limit times the fee cap, plus the value.
    pub fn max_cost(&self) -> u128 {
        self.fee_cap()
            .saturating_mul(u128::from(self.gas_limit))
            .saturating_add(self.value)
    }
}

impl SignedTx {
    /// The wire form: the RLP list for legacy, the type byte and RLP list for type 2.
    pub fn raw(&self) -> Vec<u8> {
        let t = &self.tx;
        let (nonce, gas, to, value) = t.body_fields();
        let data = Item::Bytes(t.data.clone());
        match &t.fees {
            Fees::Legacy { gas_price } => {
                let v = u128::from(self.sig.y_parity) + 35 + 2 * u128::from(t.chain_id);
                Item::List(vec![
                    nonce,
                    uint(*gas_price),
                    gas,
                    to,
                    value,
                    data,
                    uint(v),
                    strip(&self.sig.r),
                    strip(&self.sig.s),
                ])
                .encode()
            }
            Fees::Eip1559 {
                max_priority_fee,
                max_fee,
                access_list,
            } => {
                let l = Item::List(vec![
                    uint(u128::from(t.chain_id)),
                    nonce,
                    uint(*max_priority_fee),
                    uint(*max_fee),
                    gas,
                    to,
                    value,
                    data,
                    access_item(access_list),
                    uint(u128::from(self.sig.y_parity)),
                    strip(&self.sig.r),
                    strip(&self.sig.s),
                ]);
                let mut out = vec![TYPE_1559];
                out.extend(l.encode());
                out
            }
        }
    }

    /// The transaction hash: Keccak-256 of the wire form.
    pub fn hash(&self) -> [u8; 32] {
        keccak256(&self.raw())
    }

    /// The sender, recovered from the signature.
    pub fn sender(&self) -> Result<Address, TxError> {
        let rec = RecoveryId::from_byte(self.sig.y_parity).ok_or(TxError::BadV)?;
        let sig =
            Signature::from_scalars(self.sig.r, self.sig.s).map_err(|_| TxError::ZeroSignature)?;
        let key = VerifyingKey::recover_from_prehash(&self.tx.signing_hash(), &sig, rec)
            .map_err(|_| TxError::Unrecoverable)?;
        Ok(address_of(&key))
    }

    /// Parses a raw transaction, refusing anything with a second encoding or a weak signature.
    pub fn from_raw(raw: &[u8]) -> Result<SignedTx, TxError> {
        match raw.first() {
            None => Err(RlpError::Eof.into()),
            Some(&TYPE_1559) => Self::from_1559(&raw[1..]),
            Some(&b) if b >= 0xc0 => Self::from_legacy(raw),
            Some(&b) => Err(TxError::UnknownType(b)),
        }
    }

    fn from_legacy(raw: &[u8]) -> Result<SignedTx, TxError> {
        let item = Item::decode(raw)?;
        let f = item.as_list()?;
        if f.len() != 9 {
            return Err(TxError::FieldCount);
        }
        let v = f[6].as_u128()?;
        if v < 35 {
            return Err(if v == 27 || v == 28 {
                TxError::Unprotected
            } else {
                TxError::BadV
            });
        }
        let chain_id = u64::try_from((v - 35) / 2).map_err(|_| TxError::BadV)?;
        let y_parity = ((v - 35) % 2) as u8;
        let tx = Tx {
            chain_id,
            nonce: f[0].as_u64()?,
            gas_limit: f[2].as_u64()?,
            to: parse_to(&f[3])?,
            value: f[4].as_u128()?,
            data: f[5].as_bytes()?.to_vec(),
            fees: Fees::Legacy {
                gas_price: f[1].as_u128()?,
            },
        };
        finish(tx, y_parity, &f[7], &f[8])
    }

    fn from_1559(body: &[u8]) -> Result<SignedTx, TxError> {
        let item = Item::decode(body)?;
        let f = item.as_list()?;
        if f.len() != 12 {
            return Err(TxError::FieldCount);
        }
        let y = f[9].as_u64()?;
        if y > 1 {
            return Err(TxError::BadV);
        }
        let tx = Tx {
            chain_id: f[0].as_u64()?,
            nonce: f[1].as_u64()?,
            gas_limit: f[4].as_u64()?,
            to: parse_to(&f[5])?,
            value: f[6].as_u128()?,
            data: f[7].as_bytes()?.to_vec(),
            fees: Fees::Eip1559 {
                max_priority_fee: f[2].as_u128()?,
                max_fee: f[3].as_u128()?,
                access_list: parse_access(&f[8])?,
            },
        };
        finish(tx, y as u8, &f[10], &f[11])
    }
}

fn parse_to(i: &Item) -> Result<Option<Address>, TxError> {
    let b = i.as_bytes()?;
    match b.len() {
        0 => Ok(None),
        20 => Ok(Some(b.try_into().expect("20"))),
        _ => Err(TxError::BadField),
    }
}

fn parse_access(i: &Item) -> Result<AccessList, TxError> {
    i.as_list()?
        .iter()
        .map(|entry| {
            let e = entry.as_list()?;
            if e.len() != 2 {
                return Err(TxError::FieldCount);
            }
            let a: Address = e[0].as_bytes()?.try_into().map_err(|_| TxError::BadField)?;
            let keys = e[1]
                .as_list()?
                .iter()
                .map(|k| k.as_bytes()?.try_into().map_err(|_| TxError::BadField))
                .collect::<Result<Vec<[u8; 32]>, _>>()?;
            Ok((a, keys))
        })
        .collect()
}

fn finish(tx: Tx, y_parity: u8, r: &Item, s: &Item) -> Result<SignedTx, TxError> {
    let r = pad32(r)?;
    let s = pad32(s)?;
    if r == [0; 32] || s == [0; 32] {
        return Err(TxError::ZeroSignature);
    }
    if s > HALF_ORDER {
        return Err(TxError::HighS);
    }
    let signed = SignedTx {
        tx,
        sig: Sig { y_parity, r, s },
    };
    signed.sender()?;
    Ok(signed)
}
```

### `src/node.rs`

```rust
//! A small in-process node: balances and nonces, one native ERC-20 style token,
//! a transaction pool, blocks, receipts and logs. Gas for a token call is a flat
//! number; there is no EVM.

use crate::abi::{self, Token, Ty, Word, word_from_u128, word_to_u128};
use crate::address::Address;
use crate::logs::{Log, transfer_log};
use crate::tx::SignedTx;
use std::collections::BTreeMap;

/// Gas charged on top of the intrinsic gas for a call to the token contract.
pub const TOKEN_CALL_GAS: u64 = 30_000;
/// The signature of the token's only state-changing function.
pub const TRANSFER_SIG: &str = "transfer(address,uint256)";
/// The signature of its only view function.
pub const BALANCE_OF_SIG: &str = "balanceOf(address)";

/// A read-only call that reverted.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Reverted;

#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
pub struct Account {
    pub nonce: u64,
    pub balance: u128,
}

/// Why the pool refused a transaction.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SubmitError {
    Malformed,
    WrongChain,
    NonceTooLow,
    NonceTooHigh,
    IntrinsicGas,
    FeeCapBelowBaseFee,
    InsufficientFunds,
    AlreadyKnown,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Receipt {
    pub tx_hash: [u8; 32],
    pub block_number: u64,
    pub from: Address,
    pub status: bool,
    pub gas_used: u64,
    pub effective_gas_price: u128,
    pub logs: Vec<Log>,
}

#[derive(Debug, Clone)]
pub struct Block {
    pub number: u64,
    pub base_fee: u128,
    pub txs: Vec<[u8; 32]>,
}

pub struct Node {
    pub chain_id: u64,
    pub base_fee: u128,
    pub coinbase: Address,
    pub token: Address,
    accounts: BTreeMap<Address, Account>,
    token_balances: BTreeMap<Address, u128>,
    pool: Vec<(SignedTx, Address)>,
    blocks: Vec<Block>,
    receipts: BTreeMap<[u8; 32], Receipt>,
    burned: u128,
    initial: u128,
}

impl Node {
    /// A chain at block 0 with funded accounts and token holders.
    pub fn new(
        chain_id: u64,
        base_fee: u128,
        funded: &[(Address, u128)],
        token_holders: &[(Address, u128)],
    ) -> Node {
        let accounts: BTreeMap<_, _> = funded
            .iter()
            .map(|(a, b)| {
                (
                    *a,
                    Account {
                        nonce: 0,
                        balance: *b,
                    },
                )
            })
            .collect();
        let initial = accounts.values().map(|a| a.balance).sum();
        Node {
            chain_id,
            base_fee,
            coinbase: [0xc0; 20],
            token: [0xee; 20],
            accounts,
            token_balances: token_holders.iter().copied().collect(),
            pool: Vec::new(),
            blocks: vec![Block {
                number: 0,
                base_fee,
                txs: vec![],
            }],
            receipts: BTreeMap::new(),
            burned: 0,
            initial,
        }
    }

    pub fn account(&self, a: &Address) -> Account {
        self.accounts.get(a).copied().unwrap_or_default()
    }
    pub fn token_balance(&self, a: &Address) -> u128 {
        self.token_balances.get(a).copied().unwrap_or(0)
    }
    pub fn block_number(&self) -> u64 {
        self.blocks.last().expect("genesis").number
    }
    pub fn receipt(&self, hash: &[u8; 32]) -> Option<&Receipt> {
        self.receipts.get(hash)
    }
    pub fn pool_len(&self) -> usize {
        self.pool.len()
    }
    pub fn burned(&self) -> u128 {
        self.burned
    }

    /// Ether in accounts plus ether burned: constant for the life of the chain.
    pub fn total_ether(&self) -> u128 {
        self.accounts.values().map(|a| a.balance).sum::<u128>() + self.burned
    }
    pub fn initial_ether(&self) -> u128 {
        self.initial
    }

    /// The nonce a new transaction from `a` should use, counting the pool.
    pub fn pending_nonce(&self, a: &Address) -> u64 {
        self.account(a).nonce + self.pool.iter().filter(|(_, s)| s == a).count() as u64
    }

    /// Validates and queues a raw transaction. Returns its hash.
    pub fn submit(&mut self, raw: &[u8]) -> Result<[u8; 32], SubmitError> {
        let stx = SignedTx::from_raw(raw).map_err(|_| SubmitError::Malformed)?;
        if stx.tx.chain_id != self.chain_id {
            return Err(SubmitError::WrongChain);
        }
        let from = stx.sender().map_err(|_| SubmitError::Malformed)?;
        let hash = stx.hash();
        if self.pool.iter().any(|(t, _)| t.hash() == hash) {
            return Err(SubmitError::AlreadyKnown);
        }
        let expected = self.pending_nonce(&from);
        if stx.tx.nonce < expected {
            return Err(SubmitError::NonceTooLow);
        }
        if stx.tx.nonce > expected {
            return Err(SubmitError::NonceTooHigh);
        }
        if stx.tx.gas_limit < stx.tx.intrinsic_gas() {
            return Err(SubmitError::IntrinsicGas);
        }
        if stx.tx.fee_cap() < self.base_fee {
            return Err(SubmitError::FeeCapBelowBaseFee);
        }
        if self.account(&from).balance < stx.tx.max_cost() {
            return Err(SubmitError::InsufficientFunds);
        }
        self.pool.push((stx, from));
        Ok(hash)
    }

    /// Read-only call: `balanceOf` on the token, anything else returns nothing.
    pub fn call(&self, to: &Address, data: &[u8]) -> Result<Vec<u8>, Reverted> {
        if *to != self.token {
            return Ok(vec![]);
        }
        let args = abi::decode_call(BALANCE_OF_SIG, data).map_err(|_| Reverted)?;
        let Token::Address(who) = &args[0] else {
            return Err(Reverted);
        };
        Ok(word_from_u128(self.token_balance(who)).to_vec())
    }

    /// Mines the pool into a block, in order, and returns the new block number.
    pub fn mine(&mut self) -> u64 {
        let number = self.block_number() + 1;
        let pool = std::mem::take(&mut self.pool);
        let mut hashes = Vec::new();
        for (stx, from) in pool {
            if let Some(r) = self.execute(&stx, from, number) {
                hashes.push(r.tx_hash);
                self.receipts.insert(r.tx_hash, r);
            }
        }
        self.blocks.push(Block {
            number,
            base_fee: self.base_fee,
            txs: hashes,
        });
        number
    }

    fn execute(&mut self, stx: &SignedTx, from: Address, block: u64) -> Option<Receipt> {
        let t = &stx.tx;
        let acct = self.account(&from);
        if acct.nonce != t.nonce || acct.balance < t.max_cost() {
            return None; // no longer valid: dropped, not included
        }
        let price = t.effective_gas_price(self.base_fee);
        let mut logs = Vec::new();
        let mut status = true;
        let mut gas_used = t.intrinsic_gas();
        let mut moves: Option<(Address, u128)> = None;
        match t.to {
            Some(to) if to == self.token => {
                gas_used += TOKEN_CALL_GAS;
                if gas_used > t.gas_limit {
                    status = false;
                    gas_used = t.gas_limit;
                } else if t.value > 0 {
                    // the token has no payable function: value sent to it reverts the call
                    status = false;
                } else {
                    match self.token_transfer(&from, &t.data) {
                        Some(l) => logs.push(l),
                        None => status = false,
                    }
                }
            }
            Some(to) => moves = Some((to, t.value)),
            None => status = false, // creation is not modelled
        }
        let fee = u128::from(gas_used) * price;
        let burn = u128::from(gas_used) * self.base_fee.min(price);
        let tip = fee - burn;
        let a = self.accounts.entry(from).or_default();
        a.nonce += 1;
        a.balance -= fee;
        self.burned += burn;
        self.accounts.entry(self.coinbase).or_default().balance += tip;
        if let (true, Some((to, value))) = (status, moves) {
            self.accounts.entry(from).or_default().balance -= value;
            self.accounts.entry(to).or_default().balance += value;
        }
        Some(Receipt {
            tx_hash: stx.hash(),
            block_number: block,
            from,
            status,
            gas_used,
            effective_gas_price: price,
            logs,
        })
    }

    fn token_transfer(&mut self, from: &Address, data: &[u8]) -> Option<Log> {
        let args = abi::decode_call(TRANSFER_SIG, data).ok()?;
        let (Token::Address(to), Token::Uint(w)) = (&args[0], &args[1]) else {
            return None;
        };
        let amount = word_to_u128(w)?;
        let bal = self.token_balance(from);
        if bal < amount {
            return None;
        }
        *self.token_balances.entry(*from).or_default() -= amount;
        *self.token_balances.entry(*to).or_default() += amount;
        Some(transfer_log(self.token, *from, *to, amount))
    }

    /// Logs of blocks `from..=to` whose emitter and topics match.
    pub fn logs(
        &self,
        from: u64,
        to: u64,
        address: Option<&Address>,
        topics: &[Vec<[u8; 32]>],
    ) -> Vec<(u64, Log)> {
        let mut out = Vec::new();
        for b in &self.blocks {
            if b.number < from || b.number > to {
                continue;
            }
            for h in &b.txs {
                for l in &self.receipts[h].logs {
                    if address.is_some_and(|a| *a != l.address) {
                        continue;
                    }
                    let ok = topics.iter().enumerate().all(|(i, set)| {
                        set.is_empty() || l.topics.get(i).is_some_and(|t| set.contains(t))
                    });
                    if ok {
                        out.push((b.number, l.clone()));
                    }
                }
            }
        }
        out
    }
}

/// `transfer(to, amount)` calldata.
pub fn transfer_calldata(to: Address, amount: u128) -> Vec<u8> {
    abi::encode_call(
        TRANSFER_SIG,
        &[Token::Address(to), Token::Uint(word_from_u128(amount))],
    )
    .expect("well-typed")
}

/// `balanceOf(who)` calldata.
pub fn balance_of_calldata(who: Address) -> Vec<u8> {
    abi::encode_call(BALANCE_OF_SIG, &[Token::Address(who)]).expect("well-typed")
}

/// Decodes a single `uint256` return value.
pub fn decode_uint_return(data: &[u8]) -> Option<Word> {
    match abi::decode_strict(&[Ty::Uint(256)], data).ok()?.pop()? {
        Token::Uint(w) => Some(w),
        _ => None,
    }
}
```

### `src/rpc.rs`

```rust
//! A JSON-RPC front end shaped like an Ethereum node's `eth_*` methods.
//! Quantities are hex with no leading zeros; data is hex of even length.

use crate::address::{Address, parse_address};
use crate::node::{Node, Reverted, SubmitError};
use serde_json::{Value, json};

pub const PARSE_ERROR: i64 = -32700;
pub const INVALID_REQUEST: i64 = -32600;
pub const METHOD_NOT_FOUND: i64 = -32601;
pub const INVALID_PARAMS: i64 = -32602;
/// Geth-style codes for rejected transactions, from memory.
pub const TX_REJECTED: i64 = -32000;
pub const EXECUTION_REVERTED: i64 = 3;

/// A quantity: `0x` and the minimal hex digits (`0x0` for zero).
pub fn quantity(v: u128) -> String {
    format!("0x{v:x}")
}

/// Parses a quantity strictly: only hex digits (`from_str_radix` alone would accept a
/// leading `+`), no leading zeros, at least one digit.
pub fn parse_quantity(s: &str) -> Option<u128> {
    let digits = s.strip_prefix("0x")?;
    if !digits.bytes().all(|b| b.is_ascii_hexdigit())
        || (digits.len() > 1 && digits.starts_with('0'))
    {
        return None;
    }
    u128::from_str_radix(digits, 16).ok()
}

/// Data: `0x` and an even number of hex digits.
pub fn data_hex(b: &[u8]) -> String {
    format!("0x{}", hex::encode(b))
}

pub fn parse_data(s: &str) -> Option<Vec<u8>> {
    hex::decode(s.strip_prefix("0x")?).ok()
}

fn err(id: Value, code: i64, message: &str) -> Value {
    json!({"jsonrpc":"2.0","id":id,"error":{"code":code,"message":message}})
}
fn ok(id: Value, result: Value) -> Value {
    json!({"jsonrpc":"2.0","id":id,"result":result})
}

fn submit_message(e: SubmitError) -> &'static str {
    match e {
        SubmitError::Malformed => "invalid transaction",
        SubmitError::WrongChain => "invalid chain id for signer",
        SubmitError::NonceTooLow => "nonce too low",
        SubmitError::NonceTooHigh => "nonce too high",
        SubmitError::IntrinsicGas => "intrinsic gas too low",
        SubmitError::FeeCapBelowBaseFee => "max fee per gas less than block base fee",
        SubmitError::InsufficientFunds => "insufficient funds for gas * price + value",
        SubmitError::AlreadyKnown => "already known",
    }
}

/// Handles one request text and returns the response text.
pub fn handle(node: &mut Node, request: &str) -> String {
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
    call(node, method, params, id).to_string()
}

fn addr_param(p: Option<&Value>) -> Option<Address> {
    parse_address(p?.as_str()?).ok()
}

/// A block tag or number, resolved against the chain head.
fn block_param(node: &Node, p: Option<&Value>) -> Option<u64> {
    match p {
        None | Some(Value::Null) => Some(node.block_number()),
        Some(Value::String(s)) => match s.as_str() {
            "latest" | "pending" | "safe" | "finalized" => Some(node.block_number()),
            "earliest" => Some(0),
            q => u64::try_from(parse_quantity(q)?).ok(),
        },
        _ => None,
    }
}

fn call(node: &mut Node, method: &str, params: &[Value], id: Value) -> Value {
    let bad = |id: Value| err(id, INVALID_PARAMS, "Invalid params");
    match method {
        "eth_chainId" => ok(id, json!(quantity(u128::from(node.chain_id)))),
        "eth_blockNumber" => ok(id, json!(quantity(u128::from(node.block_number())))),
        "eth_gasPrice" => ok(id, json!(quantity(node.base_fee))),
        "eth_getBalance" => match addr_param(params.first()) {
            Some(a) => ok(id, json!(quantity(node.account(&a).balance))),
            None => bad(id),
        },
        "eth_getTransactionCount" => {
            let Some(a) = addr_param(params.first()) else {
                return bad(id);
            };
            let n = match params.get(1).and_then(Value::as_str) {
                Some("pending") => node.pending_nonce(&a),
                Some("latest") | None => node.account(&a).nonce,
                Some(_) => return bad(id),
            };
            ok(id, json!(quantity(u128::from(n))))
        }
        "eth_sendRawTransaction" => {
            let Some(raw) = params.first().and_then(Value::as_str).and_then(parse_data) else {
                return bad(id);
            };
            match node.submit(&raw) {
                Ok(h) => ok(id, json!(data_hex(&h))),
                Err(e) => err(id, TX_REJECTED, submit_message(e)),
            }
        }
        "eth_call" => {
            let Some(obj) = params.first().and_then(Value::as_object) else {
                return bad(id);
            };
            let Some(to) = obj
                .get("to")
                .and_then(Value::as_str)
                .and_then(|s| parse_address(s).ok())
            else {
                return bad(id);
            };
            let Some(data) = obj
                .get("data")
                .or_else(|| obj.get("input"))
                .and_then(Value::as_str)
                .and_then(parse_data)
            else {
                return bad(id);
            };
            match node.call(&to, &data) {
                Ok(out) => ok(id, json!(data_hex(&out))),
                Err(Reverted) => err(id, EXECUTION_REVERTED, "execution reverted"),
            }
        }
        "eth_getTransactionReceipt" => {
            let Some(h) = params.first().and_then(Value::as_str).and_then(parse_data) else {
                return bad(id);
            };
            let Ok(h) = <[u8; 32]>::try_from(h) else {
                return bad(id);
            };
            match node.receipt(&h) {
                None => ok(id, Value::Null),
                Some(r) => ok(
                    id,
                    json!({
                        "transactionHash": data_hex(&r.tx_hash),
                        "blockNumber": quantity(u128::from(r.block_number)),
                        "from": crate::address::to_checksum(&r.from),
                        "status": if r.status { "0x1" } else { "0x0" },
                        "gasUsed": quantity(u128::from(r.gas_used)),
                        "effectiveGasPrice": quantity(r.effective_gas_price),
                        "logs": r.logs.iter().map(log_json).collect::<Vec<_>>(),
                    }),
                ),
            }
        }
        "eth_getLogs" => {
            let Some(f) = params.first().and_then(Value::as_object) else {
                return bad(id);
            };
            let Some(from) = block_param(node, f.get("fromBlock")) else {
                return bad(id);
            };
            let Some(to) = block_param(node, f.get("toBlock")) else {
                return bad(id);
            };
            let address = match f.get("address") {
                None | Some(Value::Null) => None,
                Some(v) => match v.as_str().and_then(|s| parse_address(s).ok()) {
                    Some(a) => Some(a),
                    None => return bad(id),
                },
            };
            let mut topics: Vec<Vec<[u8; 32]>> = Vec::new();
            if let Some(ts) = f.get("topics") {
                let Some(ts) = ts.as_array() else {
                    return bad(id);
                };
                for t in ts {
                    let set: Option<Vec<[u8; 32]>> = match t {
                        Value::Null => Some(vec![]),
                        Value::String(s) => parse_data(s)
                            .and_then(|b| b.try_into().ok())
                            .map(|x| vec![x]),
                        Value::Array(a) => a
                            .iter()
                            .map(|x| {
                                x.as_str()
                                    .and_then(parse_data)
                                    .and_then(|b| b.try_into().ok())
                            })
                            .collect(),
                        _ => None,
                    };
                    match set {
                        Some(s) => topics.push(s),
                        None => return bad(id),
                    }
                }
            }
            let found = node.logs(from, to, address.as_ref(), &topics);
            ok(
                id,
                json!(
                    found
                        .iter()
                        .map(|(n, l)| {
                            let mut j = log_json(l);
                            j["blockNumber"] = json!(quantity(u128::from(*n)));
                            j
                        })
                        .collect::<Vec<_>>()
                ),
            )
        }
        "evm_mine" => ok(id, json!(quantity(u128::from(node.mine())))),
        _ => err(id, METHOD_NOT_FOUND, "Method not found"),
    }
}

fn log_json(l: &crate::logs::Log) -> Value {
    json!({
        "address": crate::address::to_checksum(&l.address),
        "topics": l.topics.iter().map(|t| data_hex(t)).collect::<Vec<_>>(),
        "data": data_hex(&l.data),
    })
}
```

### `tests/common/mod.rs`

```rust
#![allow(dead_code)]
use evmlab::address::{Address, address_of, signing_key};
use k256::ecdsa::SigningKey;

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
    pub fn word(&mut self) -> [u8; 32] {
        self.bytes(32).try_into().unwrap()
    }
    pub fn addr(&mut self) -> Address {
        self.bytes(20).try_into().unwrap()
    }
}

/// A key from a small seed byte (repeated 32 times); seeds 1..=254 are valid (255 is above the group order).
pub fn key(seed: u8) -> SigningKey {
    signing_key(&[seed; 32]).expect("valid")
}

pub fn addr_of_seed(seed: u8) -> Address {
    address_of(key(seed).verifying_key())
}
```

### `tests/keccak_rlp_tests.rs`

```rust
mod common;
use alloy_rlp::Encodable as _;
use common::*;
use evmlab::keccak::*;
use evmlab::rlp::*;

fn h(x: impl AsRef<[u8]>) -> String {
    hex::encode(x)
}

#[test]
fn keccak_known_answers_from_python() {
    assert_eq!(
        h(keccak256(b"")),
        "c5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470"
    );
    // not the NIST SHA3-256 of the empty string
    assert_ne!(
        h(keccak256(b"")),
        "a7ffc6f8bf1ed76651c14756a061d662f580ff4de43b49fa82d80a4b80f8434a"
    );
    assert_eq!(h(selector("transfer(address,uint256)")), "a9059cbb");
    assert_eq!(h(selector("balanceOf(address)")), "70a08231");
    assert_eq!(h(selector("approve(address,uint256)")), "095ea7b3");
    assert_eq!(
        h(event_topic("Transfer(address,address,uint256)")),
        "ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"
    );
}

#[test]
fn keccak_agrees_with_alloy_on_random_inputs_at_every_block_boundary() {
    let mut r = Rng(11);
    for len in [0usize, 1, 31, 32, 135, 136, 137, 271, 272, 273, 1000] {
        let data = r.bytes(len);
        assert_eq!(
            keccak256(&data),
            alloy_primitives::keccak256(&data).0,
            "len {len}"
        );
    }
}

#[test]
fn rlp_spec_examples() {
    let e = |i: &Item| h(i.encode());
    let b = |s: &[u8]| Item::Bytes(s.to_vec());
    assert_eq!(e(&b(b"dog")), "83646f67");
    assert_eq!(
        e(&Item::List(vec![b(b"cat"), b(b"dog")])),
        "c88363617483646f67"
    );
    assert_eq!(e(&b(b"")), "80");
    assert_eq!(e(&Item::List(vec![])), "c0");
    assert_eq!(e(&uint(0)), "80");
    assert_eq!(e(&b(&[0])), "00");
    assert_eq!(e(&uint(15)), "0f");
    assert_eq!(e(&uint(1024)), "820400");
    assert_eq!(e(&b(&[0x7f])), "7f");
    assert_eq!(e(&b(&[0x80])), "8180");
    let empty = || Item::List(vec![]);
    let set3 = Item::List(vec![
        empty(),
        Item::List(vec![empty()]),
        Item::List(vec![empty(), Item::List(vec![empty()])]),
    ]);
    assert_eq!(e(&set3), "c7c0c1c0c3c0c1c0");
    let lorem = b(b"Lorem ipsum dolor sit amet, consectetur adipisicing elit");
    assert_eq!(
        e(&lorem),
        format!(
            "b838{}",
            h(b"Lorem ipsum dolor sit amet, consectetur adipisicing elit")
        )
    );
}

#[test]
fn rlp_length_boundaries() {
    for len in [54usize, 55, 56, 57, 255, 256, 65535, 65536] {
        let data = vec![0xabu8; len];
        let enc = Item::Bytes(data.clone()).encode();
        let prefix = if len < 56 {
            1
        } else {
            1 + (usize::BITS - len.leading_zeros()).div_ceil(8) as usize
        };
        assert_eq!(enc.len(), len + prefix, "len {len}");
        assert_eq!(Item::decode(&enc), Ok(Item::Bytes(data.clone())));
        let l = Item::List(vec![Item::Bytes(data)]);
        assert_eq!(Item::decode(&l.encode()), Ok(l));
    }
}

#[test]
fn rlp_matches_alloy_for_random_byte_strings_and_integers() {
    let mut r = Rng(5);
    for _ in 0..500 {
        let n = r.below(300) as usize;
        let data = r.bytes(n);
        let mut theirs = Vec::new();
        data.as_slice().encode(&mut theirs);
        assert_eq!(Item::Bytes(data).encode(), theirs);
        let v = (u128::from(r.next_u64()) << 64 | u128::from(r.next_u64())) >> r.below(128);
        let mut theirs = Vec::new();
        v.encode(&mut theirs);
        assert_eq!(uint(v).encode(), theirs, "{v}");
        assert_eq!(Item::decode(&theirs).unwrap().as_u128(), Ok(v));
    }
}

#[test]
fn rlp_refuses_non_canonical_forms() {
    for (hexs, why) in [
        ("8100", "single byte below 0x80 with a prefix"),
        ("817f", "single byte 0x7f with a prefix"),
        ("b80100", "long form for a short string"),
        ("b8020000", "long form for a short string"),
        ("b90001aa", "length with a leading zero"),
        ("f80100", "long list form for a short list"),
        ("f9000100", "list length with a leading zero"),
    ] {
        let b = hex::decode(hexs).unwrap();
        assert!(
            matches!(
                Item::decode(&b),
                Err(RlpError::NonCanonical) | Err(RlpError::Eof)
            ),
            "{why}"
        );
    }
    assert_eq!(
        Item::decode(&hex::decode("8100").unwrap()),
        Err(RlpError::NonCanonical)
    );
    assert_eq!(
        Item::decode(&hex::decode("b80100").unwrap()),
        Err(RlpError::NonCanonical)
    );
    assert_eq!(
        Item::decode(&hex::decode("b90001aa").unwrap()),
        Err(RlpError::NonCanonical)
    );
    assert_eq!(
        Item::decode(&hex::decode("f80100").unwrap()),
        Err(RlpError::NonCanonical)
    );
}

#[test]
fn rlp_hostile_and_malformed_input() {
    assert_eq!(Item::decode(&[]), Err(RlpError::Eof));
    assert_eq!(Item::decode(&[0x83, 1, 2]), Err(RlpError::Eof));
    assert_eq!(Item::decode(&[0x01, 0x02]), Err(RlpError::Trailing));
    // a string that claims 2^32 bytes
    assert_eq!(
        Item::decode(&hex::decode("bb0100000000").unwrap()),
        Err(RlpError::TooLong)
    );
    // a length of 9 bytes cannot be a usize
    assert_eq!(
        Item::decode(&hex::decode("bf010000000000000000").unwrap()),
        Err(RlpError::TooLong)
    );
    // a list that claims more than it has
    assert_eq!(Item::decode(&[0xc5, 0x01]), Err(RlpError::TooLong));
    // an item inside a list that runs past the list
    assert_eq!(Item::decode(&[0xc2, 0x83, 0x01]), Err(RlpError::Eof));
    // nesting
    let mut deep_item = Item::List(vec![]);
    for _ in 0..40 {
        deep_item = Item::List(vec![deep_item]);
    }
    assert_eq!(Item::decode(&deep_item.encode()), Err(RlpError::TooDeep));
    let mut ok = Item::List(vec![]);
    for _ in 0..30 {
        ok = Item::List(vec![ok]);
    }
    assert_eq!(Item::decode(&ok.encode()), Ok(ok));
}

#[test]
fn rlp_integers_are_canonical() {
    assert_eq!(Item::Bytes(vec![0, 1]).as_u64(), Err(RlpError::BadInteger));
    assert_eq!(Item::Bytes(vec![0]).as_u64(), Err(RlpError::BadInteger));
    assert_eq!(Item::Bytes(vec![]).as_u64(), Ok(0));
    assert_eq!(Item::Bytes(vec![1; 9]).as_u64(), Err(RlpError::BadInteger));
    assert_eq!(Item::Bytes(vec![1; 8]).as_u64(), Ok(0x0101010101010101));
    assert_eq!(
        Item::Bytes(vec![1; 17]).as_u128(),
        Err(RlpError::BadInteger)
    );
    assert_eq!(Item::List(vec![]).as_u64(), Err(RlpError::WrongKind));
    assert_eq!(Item::Bytes(vec![]).as_list(), Err(RlpError::WrongKind));
    assert_eq!(uint_bytes(0), Vec::<u8>::new());
    assert_eq!(uint_bytes(256), vec![1, 0]);
}

#[test]
fn rlp_round_trip_of_random_trees() {
    fn tree(r: &mut Rng, depth: u32) -> Item {
        if depth == 0 || r.below(3) == 0 {
            let n = r.below(70) as usize;
            Item::Bytes(r.bytes(n))
        } else {
            let n = r.below(5) as usize;
            Item::List((0..n).map(|_| tree(r, depth - 1)).collect())
        }
    }
    let mut r = Rng(77);
    for _ in 0..400 {
        let t = tree(&mut r, 4);
        assert_eq!(Item::decode(&t.encode()), Ok(t));
    }
    // garbage never panics
    for _ in 0..4000 {
        let n = r.below(40) as usize;
        let b = r.bytes(n);
        let _ = Item::decode(&b);
    }
}
```

### `tests/abi_tests.rs`

```rust
mod common;
use alloy_primitives::{Address as AAddr, Bytes as ABytes, U256};
use alloy_sol_types::SolValue;
use common::*;
use evmlab::abi::*;

fn u(v: u128) -> Token {
    Token::Uint(word_from_u128(v))
}
fn h(x: &[u8]) -> String {
    hex::encode(x)
}

#[test]
fn selectors_and_the_uint_alias_trap() {
    assert_eq!(
        h(&selector_of("transfer(address,uint256)").unwrap()),
        "a9059cbb"
    );
    // `uint` is an alias: the selector is computed from the canonical `uint256`
    assert_eq!(
        selector_of("transfer(address,uint)").unwrap(),
        selector_of("transfer(address,uint256)").unwrap()
    );
    // hashing the sloppy text directly gives a different, wrong selector (6cb927d8, from Python)
    assert_eq!(
        h(&evmlab::keccak::selector("transfer(address,uint)")),
        "6cb927d8"
    );
    assert_eq!(
        canonical_signature("f(uint,int,(uint8,bytes)[],address[3])").unwrap(),
        "f(uint256,int256,(uint8,bytes)[],address[3])"
    );
    assert_eq!(
        h(&selector_of("f(uint256,uint32[],bytes10,bytes)").unwrap()),
        "8be65246"
    );
}

#[test]
fn the_solidity_documentation_example() {
    // f(uint256,uint32[],bytes10,bytes) with (0x123, [0x456, 0x789], "1234567890", "Hello, world!")
    let toks = [
        u(0x123),
        Token::Array(vec![u(0x456), u(0x789)]),
        Token::FixedBytes(b"1234567890".to_vec()),
        Token::Bytes(b"Hello, world!".to_vec()),
    ];
    let data = encode_call("f(uint256,uint32[],bytes10,bytes)", &toks).unwrap();
    let expected = [
        "8be65246",
        "0000000000000000000000000000000000000000000000000000000000000123",
        "0000000000000000000000000000000000000000000000000000000000000080",
        "3132333435363738393000000000000000000000000000000000000000000000",
        "00000000000000000000000000000000000000000000000000000000000000e0",
        "0000000000000000000000000000000000000000000000000000000000000002",
        "0000000000000000000000000000000000000000000000000000000000000456",
        "0000000000000000000000000000000000000000000000000000000000000789",
        "000000000000000000000000000000000000000000000000000000000000000d",
        "48656c6c6f2c20776f726c642100000000000000000000000000000000000000",
    ]
    .concat();
    assert_eq!(h(&data), expected);
    assert_eq!(
        decode_call("f(uint256,uint32[],bytes10,bytes)", &data).unwrap(),
        toks
    );
}

#[test]
fn static_types_are_one_word_each() {
    let a = [0x11u8; 20];
    let data = encode(
        &[
            Ty::Address,
            Ty::Bool,
            Ty::Uint(8),
            Ty::Int(8),
            Ty::FixedBytes(4),
        ],
        &[
            Token::Address(a),
            Token::Bool(true),
            u(255),
            Token::Int(word_from_i128(-1)),
            Token::FixedBytes(vec![1, 2, 3, 4]),
        ],
    )
    .unwrap();
    assert_eq!(data.len(), 5 * 32);
    assert_eq!(&data[..12], &[0; 12]); // address is left-padded
    assert_eq!(&data[12..32], &a);
    assert_eq!(data[63], 1);
    assert_eq!(data[95], 255);
    assert_eq!(&data[96..128], &[0xff; 32]); // -1 is sign-extended
    assert_eq!(&data[128..132], &[1, 2, 3, 4]); // fixed bytes are right-padded
    assert_eq!(&data[132..160], &[0; 28]);
}

#[test]
fn values_must_fit_their_types() {
    assert_eq!(
        encode(&[Ty::Uint(8)], &[u(256)]),
        Err(AbiError::ValueTooWide)
    );
    assert!(encode(&[Ty::Uint(8)], &[u(255)]).is_ok());
    assert_eq!(
        encode(&[Ty::Int(8)], &[Token::Int(word_from_i128(128))]),
        Err(AbiError::ValueTooWide)
    );
    assert_eq!(
        encode(&[Ty::Int(8)], &[Token::Int(word_from_i128(-129))]),
        Err(AbiError::ValueTooWide)
    );
    assert!(encode(&[Ty::Int(8)], &[Token::Int(word_from_i128(-128))]).is_ok());
    assert!(encode(&[Ty::Int(8)], &[Token::Int(word_from_i128(127))]).is_ok());
    assert_eq!(
        encode(&[Ty::Uint(256)], &[Token::Bool(true)]),
        Err(AbiError::Mismatch)
    );
    assert_eq!(
        encode(&[Ty::FixedBytes(4)], &[Token::FixedBytes(vec![1, 2, 3])]),
        Err(AbiError::Mismatch)
    );
    assert_eq!(encode(&[Ty::Bool], &[]), Err(AbiError::Mismatch));
    assert_eq!(
        encode(
            &[Ty::FixedArray(Box::new(Ty::Bool), 2)],
            &[Token::Array(vec![Token::Bool(true)])]
        ),
        Err(AbiError::Mismatch)
    );
}

#[test]
fn dynamic_and_nested_layouts() {
    // (uint256, string, uint256[]): head = value, offset, offset
    let tys = [
        Ty::Uint(256),
        Ty::String,
        Ty::Array(Box::new(Ty::Uint(256))),
    ];
    let toks = [
        u(7),
        Token::String("hi".into()),
        Token::Array(vec![u(1), u(2), u(3)]),
    ];
    let data = encode(&tys, &toks).unwrap();
    assert_eq!(word_to_u128(data[32..64].try_into().unwrap()), Some(96)); // first offset after the 3-word head
    assert_eq!(
        word_to_u128(data[64..96].try_into().unwrap()),
        Some(96 + 64)
    ); // after len + padded "hi"
    assert_eq!(decode_strict(&tys, &data).unwrap(), toks);
    // an empty dynamic array and an empty string each take one length word
    let e = encode(&[Ty::Array(Box::new(Ty::Uint(8)))], &[Token::Array(vec![])]).unwrap();
    assert_eq!(e.len(), 64);
    assert_eq!(
        encode(&[Ty::String], &[Token::String(String::new())])
            .unwrap()
            .len(),
        64
    );
    // a static tuple is inline, a dynamic one is behind an offset
    let st = Ty::Tuple(vec![Ty::Uint(256), Ty::Bool]);
    assert!(!st.is_dynamic());
    assert_eq!(st.head_size(), 64);
    let dt = Ty::Tuple(vec![Ty::Uint(256), Ty::Bytes]);
    assert!(dt.is_dynamic());
    assert_eq!(dt.head_size(), 32);
    let tys = [Ty::Array(Box::new(dt.clone())), st.clone()];
    let toks = [
        Token::Array(vec![
            Token::Tuple(vec![u(1), Token::Bytes(vec![9; 40])]),
            Token::Tuple(vec![u(2), Token::Bytes(vec![])]),
        ]),
        Token::Tuple(vec![u(3), Token::Bool(false)]),
    ];
    let data = encode(&tys, &toks).unwrap();
    assert_eq!(decode_strict(&tys, &data).unwrap(), toks);
    // fixed arrays of dynamic elements are dynamic; of static elements, inline
    let fa = Ty::FixedArray(Box::new(Ty::String), 2);
    assert!(fa.is_dynamic());
    assert!(!Ty::FixedArray(Box::new(Ty::Uint(8)), 3).is_dynamic());
    assert_eq!(Ty::FixedArray(Box::new(Ty::Uint(8)), 3).head_size(), 96);
    assert!(!Ty::FixedArray(Box::new(Ty::String), 0).is_dynamic());
    let toks = [Token::Array(vec![
        Token::String("a".into()),
        Token::String("bc".into()),
    ])];
    assert_eq!(
        decode_strict(
            std::slice::from_ref(&fa),
            &encode(std::slice::from_ref(&fa), &toks).unwrap()
        )
        .unwrap(),
        toks
    );
}

#[test]
fn encoding_agrees_with_alloy_on_random_values() {
    let mut r = Rng(31);
    for _ in 0..400 {
        let v = U256::from_be_bytes(r.word());
        let a = r.addr();
        let slen = r.below(90) as usize;
        let s: String = (0..slen)
            .map(|_| (b'a' + r.below(26) as u8) as char)
            .collect();
        let blen = r.below(100) as usize;
        let b = r.bytes(blen);
        let alen = r.below(6) as usize;
        let arr: Vec<U256> = (0..alen).map(|_| U256::from_be_bytes(r.word())).collect();
        let theirs = (
            v,
            AAddr::from(a),
            s.clone(),
            ABytes::from(b.clone()),
            arr.clone(),
        )
            .abi_encode_params();
        let tys = [
            Ty::Uint(256),
            Ty::Address,
            Ty::String,
            Ty::Bytes,
            Ty::Array(Box::new(Ty::Uint(256))),
        ];
        let toks = [
            Token::Uint(v.to_be_bytes()),
            Token::Address(a),
            Token::String(s),
            Token::Bytes(b),
            Token::Array(arr.iter().map(|x| Token::Uint(x.to_be_bytes())).collect()),
        ];
        let ours = encode(&tys, &toks).unwrap();
        assert_eq!(ours, theirs);
        assert_eq!(decode_strict(&tys, &theirs).unwrap(), toks);
    }
}

#[test]
fn decoding_refuses_dirty_padding() {
    let ok = encode(&[Ty::Address], &[Token::Address([1; 20])]).unwrap();
    let mut bad = ok.clone();
    bad[0] = 1;
    assert_eq!(decode(&[Ty::Address], &bad), Err(AbiError::DirtyPadding));
    // bool with value 2
    let mut b = vec![0u8; 32];
    b[31] = 2;
    assert_eq!(decode(&[Ty::Bool], &b), Err(AbiError::DirtyPadding));
    // uint8 with a high byte set
    let mut b = vec![0u8; 32];
    b[30] = 1;
    assert_eq!(decode(&[Ty::Uint(8)], &b), Err(AbiError::DirtyPadding));
    // int8 whose upper bytes do not match the sign
    let mut b = vec![0u8; 32];
    b[31] = 0x80;
    assert_eq!(decode(&[Ty::Int(8)], &b), Err(AbiError::DirtyPadding));
    // bytes4 with junk after the 4 bytes
    let mut b = vec![0u8; 32];
    b[10] = 1;
    assert_eq!(
        decode(&[Ty::FixedBytes(4)], &b),
        Err(AbiError::DirtyPadding)
    );
    // bytes with junk in the padding of its last word
    let mut enc = encode(&[Ty::Bytes], &[Token::Bytes(vec![1, 2, 3])]).unwrap();
    *enc.last_mut().unwrap() = 1;
    assert_eq!(decode(&[Ty::Bytes], &enc), Err(AbiError::DirtyPadding));
}

#[test]
fn decoding_refuses_hostile_lengths_and_offsets() {
    // an offset that points past the data
    let mut data = encode(&[Ty::Bytes], &[Token::Bytes(vec![1])]).unwrap();
    data[31] = 0xff;
    assert_eq!(decode(&[Ty::Bytes], &data), Err(AbiError::BadOffset));
    // an offset with high bytes set
    let mut data = encode(&[Ty::Bytes], &[Token::Bytes(vec![1])]).unwrap();
    data[0] = 1;
    assert_eq!(decode(&[Ty::Bytes], &data), Err(AbiError::TooLong));
    // a bytes length larger than the data
    let mut data = encode(&[Ty::Bytes], &[Token::Bytes(vec![1])]).unwrap();
    data[63] = 200;
    assert_eq!(decode(&[Ty::Bytes], &data), Err(AbiError::TooLong));
    // an array that claims 2^64-1 elements
    let mut data = encode(
        &[Ty::Array(Box::new(Ty::Uint(8)))],
        &[Token::Array(vec![u(1)])],
    )
    .unwrap();
    for b in &mut data[56..64] {
        *b = 0xff;
    }
    assert_eq!(
        decode(&[Ty::Array(Box::new(Ty::Uint(8)))], &data),
        Err(AbiError::TooLong)
    );
    // an array that claims one more element than there are words
    let mut data = encode(
        &[Ty::Array(Box::new(Ty::Uint(8)))],
        &[Token::Array(vec![u(1)])],
    )
    .unwrap();
    data[63] = 2;
    assert_eq!(
        decode(&[Ty::Array(Box::new(Ty::Uint(8)))], &data),
        Err(AbiError::TooLong)
    );
    // truncated static data
    assert_eq!(
        decode(&[Ty::Uint(256), Ty::Uint(256)], &[0; 63]),
        Err(AbiError::Eof)
    );
    assert_eq!(decode(&[Ty::Uint(256)], &[]), Err(AbiError::Eof));
    // invalid UTF-8 in a string
    let mut enc = encode(&[Ty::String], &[Token::String("a".into())]).unwrap();
    enc[64] = 0xff;
    assert_eq!(decode(&[Ty::String], &enc), Err(AbiError::Utf8));
}

#[test]
fn lenient_decode_versus_strict_decode() {
    let tys = [Ty::Bytes, Ty::Bytes];
    let canonical = encode(&tys, &[Token::Bytes(vec![7]), Token::Bytes(vec![7])]).unwrap();
    assert_eq!(decode_strict(&tys, &canonical).unwrap().len(), 2);
    // both offsets point at the same bytes: a lenient decoder reads two values from one copy
    let mut alias = canonical[..64].to_vec();
    alias[63] = 64; // second offset = first offset
    alias[31] = 64;
    alias.extend_from_slice(&canonical[canonical.len() - 64..]);
    assert_eq!(
        decode(&tys, &alias).unwrap(),
        vec![Token::Bytes(vec![7]), Token::Bytes(vec![7])]
    );
    assert_eq!(decode_strict(&tys, &alias), Err(AbiError::NonCanonical));
    // junk after the data is accepted by `decode` and refused by `decode_strict`
    let mut extra = canonical.clone();
    extra.extend_from_slice(&[0xaa; 5]);
    assert!(decode(&tys, &extra).is_ok());
    assert_eq!(decode_strict(&tys, &extra), Err(AbiError::NonCanonical));
}

#[test]
fn decode_call_checks_the_selector() {
    let data = encode_call(
        "transfer(address,uint256)",
        &[Token::Address([2; 20]), u(5)],
    )
    .unwrap();
    assert!(decode_call("transfer(address,uint256)", &data).is_ok());
    assert_eq!(
        decode_call("approve(address,uint256)", &data),
        Err(AbiError::Mismatch)
    );
    assert_eq!(
        decode_call("transfer(address,uint256)", &data[..3]),
        Err(AbiError::Eof)
    );
    assert_eq!(
        decode_call("transfer(address,uint256)", &data[..data.len() - 1]),
        Err(AbiError::Eof)
    );
    let mut extra = data.clone();
    extra.push(0);
    assert_eq!(
        decode_call("transfer(address,uint256)", &extra),
        Err(AbiError::NonCanonical)
    );
}

#[test]
fn signature_parsing() {
    for bad in [
        "",
        "f",
        "f(",
        "f)",
        "(uint256)",
        "1f(uint256)",
        "f(uint7)",
        "f(uint0)",
        "f(uint264)",
        "f(bytes0)",
        "f(bytes33)",
        "f(uint256,)",
        "f(,uint256)",
        "f(uint256[)",
        "f(uint256[x])",
        "f((uint256)",
        "f(uint256)x",
        "f(uint08)",
        "f(int)x",
        "f(foo)",
        "f g(uint256)",
    ] {
        assert!(parse_signature(bad).is_err(), "{bad}");
    }
    let (n, t) = parse_signature("f((uint8,bytes)[2],string)").unwrap();
    assert_eq!(n, "f");
    assert_eq!(
        t[0],
        Ty::FixedArray(Box::new(Ty::Tuple(vec![Ty::Uint(8), Ty::Bytes])), 2)
    );
    assert_eq!(t[1], Ty::String);
    assert_eq!(parse_signature("f()").unwrap().1, vec![]);
    assert_eq!(parse_type("uint").unwrap(), Ty::Uint(256));
    assert_eq!(parse_type("int128").unwrap(), Ty::Int(128));
    assert_eq!(
        parse_type("uint8[][3]").unwrap(),
        Ty::FixedArray(Box::new(Ty::Array(Box::new(Ty::Uint(8)))), 3)
    );
    assert_eq!(parse_type("bytes32").unwrap(), Ty::FixedBytes(32));
    assert_eq!(parse_type("()").unwrap(), Ty::Tuple(vec![]));
}

#[test]
fn canonical_names_round_trip_through_the_parser() {
    for s in [
        "uint8",
        "int256",
        "address",
        "bool",
        "bytes",
        "string",
        "bytes7",
        "uint16[]",
        "(address,uint256)[4]",
        "((bool,bytes),string[])[]",
    ] {
        assert_eq!(parse_type(s).unwrap().canonical(), s);
    }
}

#[test]
fn word_helpers() {
    assert_eq!(word_to_u128(&word_from_u128(u128::MAX)), Some(u128::MAX));
    let mut w = word_from_u128(1);
    w[15] = 1;
    assert_eq!(word_to_u128(&w), None);
    assert_eq!(word_from_i128(-2)[..16], [0xff; 16]);
    assert_eq!(word_from_i128(2)[..16], [0; 16]);
    assert_eq!(h(&word_from_i128(-1)), "ff".repeat(32));
}

#[test]
fn random_garbage_never_panics() {
    let mut r = Rng(3);
    let tys = [
        Ty::Uint(256),
        Ty::String,
        Ty::Array(Box::new(Ty::Tuple(vec![Ty::Bytes, Ty::Bool]))),
        Ty::FixedArray(Box::new(Ty::Bytes), 2),
    ];
    for _ in 0..3000 {
        let n = r.below(400) as usize;
        let mut b = r.bytes(n);
        // make some of it look structured
        if n >= 64 && r.below(2) == 0 {
            b[0..30].fill(0);
            b[32..62].fill(0);
        }
        let _ = decode(&tys, &b);
        let _ = decode_strict(&tys, &b);
    }
}
```

### `tests/address_tests.rs`

```rust
mod common;
use alloy_primitives::Address as AAddr;
use common::*;
use evmlab::address::*;

#[test]
fn eip55_known_answers() {
    for s in [
        "0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed",
        "0xfB6916095ca1df60bB79Ce92cE3Ea74c37c5d359",
        "0xdbF03B407c01E7cD3CBea99509d93f8DDDC8C6FB",
        "0xD1220A0cf47c7B9Be7A2E6BA89F429762e7b9aDb",
    ] {
        let a = parse_address(s).unwrap();
        assert_eq!(to_checksum(&a), s);
        assert_eq!(parse_address(&s.to_lowercase()), Ok(a));
        assert_eq!(
            parse_address(&format!("0x{}", s[2..].to_uppercase())),
            Ok(a)
        );
    }
}

#[test]
fn checksum_matches_alloy_on_random_addresses() {
    let mut r = Rng(9);
    for _ in 0..300 {
        let a = r.addr();
        assert_eq!(to_checksum(&a), AAddr::from(a).to_checksum(None));
    }
}

#[test]
fn a_single_wrong_capital_is_caught() {
    let s = "0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed";
    let mut caught = 0;
    let mut letters = 0;
    for (i, c) in s.char_indices().skip(2) {
        if !c.is_ascii_alphabetic() {
            continue;
        }
        letters += 1;
        let flipped = if c.is_ascii_uppercase() {
            c.to_ascii_lowercase()
        } else {
            c.to_ascii_uppercase()
        };
        let mut t = s.to_string();
        t.replace_range(i..i + 1, &flipped.to_string());
        if parse_address(&t) == Err(AddressError::BadChecksum) {
            caught += 1;
        }
    }
    assert_eq!(caught, letters);
    assert!(letters > 10);
}

#[test]
fn parse_rejects_bad_text() {
    assert_eq!(
        parse_address("5aaeb6053f3e94c9b9a09f33669435e7ef1beaed"),
        Err(AddressError::BadHex)
    );
    assert_eq!(
        parse_address("0x5aaeb6053f3e94c9b9a09f33669435e7ef1beae"),
        Err(AddressError::BadLength)
    );
    assert_eq!(
        parse_address("0x5aaeb6053f3e94c9b9a09f33669435e7ef1beaedd"),
        Err(AddressError::BadLength)
    );
    assert_eq!(
        parse_address("0xzzaeb6053f3e94c9b9a09f33669435e7ef1beaed"),
        Err(AddressError::BadHex)
    );
    assert_eq!(parse_address(""), Err(AddressError::BadHex));
    assert_eq!(parse_address("0x"), Err(AddressError::BadLength));
    // digits only: no letters, so no case to check
    assert_eq!(
        parse_address(&format!("0x{}", "1".repeat(40))),
        Ok([0x11; 20])
    );
}

#[test]
fn addresses_of_keys_computed_in_python() {
    let a =
        |secret: [u8; 32]| to_checksum(&address_of(signing_key(&secret).unwrap().verifying_key()));
    assert_eq!(a([0x46; 32]), "0x9d8A62f656a8d1615C1294fd71e9CFb3E4855A4F");
    assert_eq!(a([0x07; 32]), "0x4a62316623ad457F02cDC5D997deD67a383EC569");
    let mut one = [0u8; 32];
    one[31] = 1;
    assert_eq!(a(one), "0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf");
}

#[test]
fn invalid_secret_keys() {
    assert!(signing_key(&[0; 32]).is_none());
    assert!(signing_key(&[0xff; 32]).is_none()); // above the group order
    let n_minus_1 =
        hex::decode("fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364140").unwrap();
    assert!(signing_key(&n_minus_1.try_into().unwrap()).is_some());
    let n =
        hex::decode("fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141").unwrap();
    assert!(signing_key(&n.try_into().unwrap()).is_none());
}
```

### `tests/tx_tests.rs`

```rust
mod common;
use common::*;
use evmlab::address::{Address, to_checksum};
use evmlab::keccak::keccak256;
use evmlab::rlp::{Item, RlpError, uint};
use evmlab::tx::*;

const EIP155_SIGHASH: &str = "daf5a779ae972f972197303d7b574746c7ef83eadac0f2791ad23db92e4c8e53";
const EIP155_RAW: &str = "f86c098504a817c800825208943535353535353535353535353535353535353535880de0b6b3a76400008025a028ef61340bd939bc2195fe537567866003e1a15d3c71ff63e1590620aa636276a067cbe9d8997f761aecb703304b3800ccf555c9f3dc64214b297fb1966a3b6d83";
const EIP155_HASH: &str = "33469b22e9f636356c4160a87eb19df52b7412e8eac32a4a55ffe88ea8350788";
const T2_SIGHASH: &str = "93307d1c98e601f878479c1c525125c6347105ceebdd122c0674e94be765bafe";
const T2_RAW: &str = "02f8a88205390384773594008506fc23ac0082520894222222222222222222222222222222222222222282303980f838f7943333333333333333333333333333333333333333e1a0010101010101010101010101010101010101010101010101010101010101010101a0b45c95a591641f8d2914b7562c6cb577a076869bcaf0017f5a07d358c462e1a6a03e8989e739445afeb7b08cd1c5c8e1bdea90ea2725135966c44be1e1b4d2d675";
const T2_HASH: &str = "bccf57aee3db5b149ccf2f3a2a5ed530d5f793c74de6310ffdb30a67b63e0ece";

fn eip155_tx() -> Tx {
    Tx {
        chain_id: 1,
        nonce: 9,
        gas_limit: 21_000,
        to: Some([0x35; 20]),
        value: 1_000_000_000_000_000_000,
        data: vec![],
        fees: Fees::Legacy {
            gas_price: 20_000_000_000,
        },
    }
}

fn t2_tx() -> Tx {
    Tx {
        chain_id: 1337,
        nonce: 3,
        gas_limit: 21_000,
        to: Some([0x22; 20]),
        value: 12345,
        data: vec![],
        fees: Fees::Eip1559 {
            max_priority_fee: 2_000_000_000,
            max_fee: 30_000_000_000,
            access_list: vec![([0x33; 20], vec![[1; 32]])],
        },
    }
}

#[test]
fn the_eip155_example_matches_python_byte_for_byte() {
    let tx = eip155_tx();
    assert_eq!(hex::encode(tx.signing_hash()), EIP155_SIGHASH);
    let signed = tx.sign(&key_from([0x46; 32]));
    assert_eq!(hex::encode(signed.raw()), EIP155_RAW);
    assert_eq!(hex::encode(signed.hash()), EIP155_HASH);
    assert_eq!(
        to_checksum(&signed.sender().unwrap()),
        "0x9d8A62f656a8d1615C1294fd71e9CFb3E4855A4F"
    );
    assert_eq!(signed.sig.y_parity, 0);
}

fn key_from(s: [u8; 32]) -> k256::ecdsa::SigningKey {
    evmlab::address::signing_key(&s).unwrap()
}

#[test]
fn the_type2_example_matches_python_byte_for_byte() {
    let tx = t2_tx();
    assert_eq!(hex::encode(tx.signing_hash()), T2_SIGHASH);
    let signed = tx.sign(&key_from([7; 32]));
    assert_eq!(hex::encode(signed.raw()), T2_RAW);
    assert_eq!(hex::encode(signed.hash()), T2_HASH);
    assert_eq!(signed.sig.y_parity, 1);
    assert_eq!(
        to_checksum(&signed.sender().unwrap()),
        "0x4a62316623ad457F02cDC5D997deD67a383EC569"
    );
    assert_eq!(signed.raw()[0], TYPE_1559);
}

#[test]
fn parsing_what_python_built() {
    let a = SignedTx::from_raw(&hex::decode(EIP155_RAW).unwrap()).unwrap();
    assert_eq!(a.tx, eip155_tx());
    assert_eq!(a.hash().to_vec(), hex::decode(EIP155_HASH).unwrap());
    let b = SignedTx::from_raw(&hex::decode(T2_RAW).unwrap()).unwrap();
    assert_eq!(b.tx, t2_tx());
    assert_eq!(b.sender(), Ok(addr_of_seed(7)));
}

#[test]
fn the_chain_id_is_part_of_what_is_signed() {
    let signed = eip155_tx().sign(&key(5));
    let from = signed.sender().unwrap();
    assert_eq!(from, addr_of_seed(5));
    // the same signature presented for another chain recovers a different, unrelated sender
    let mut moved = signed.clone();
    moved.tx.chain_id = 5;
    assert_ne!(moved.sender().ok(), Some(from));
    let mut moved = signed.clone();
    moved.tx.nonce += 1;
    assert_ne!(moved.sender().ok(), Some(from));
    let t2 = t2_tx().sign(&key(5));
    let mut moved = t2.clone();
    moved.tx.chain_id = 1;
    assert_ne!(moved.sender().ok(), Some(addr_of_seed(5)));
    let mut moved = t2.clone();
    moved.tx.value += 1;
    assert_ne!(moved.sender().ok(), Some(addr_of_seed(5)));
}

#[test]
fn every_field_changes_the_signing_hash() {
    for base in [eip155_tx(), t2_tx()] {
        let h = base.signing_hash();
        let mut v: Vec<Tx> = vec![];
        let mut t = base.clone();
        t.chain_id += 1;
        v.push(t);
        let mut t = base.clone();
        t.nonce += 1;
        v.push(t);
        let mut t = base.clone();
        t.gas_limit += 1;
        v.push(t);
        let mut t = base.clone();
        t.to = Some([1; 20]);
        v.push(t);
        let mut t = base.clone();
        t.to = None;
        v.push(t);
        let mut t = base.clone();
        t.value += 1;
        v.push(t);
        let mut t = base.clone();
        t.data = vec![0];
        v.push(t);
        let mut t = base.clone();
        match &mut t.fees {
            Fees::Legacy { gas_price } => *gas_price += 1,
            Fees::Eip1559 { max_fee, .. } => *max_fee += 1,
        }
        v.push(t);
        if let Fees::Eip1559 {
            max_priority_fee,
            access_list,
            max_fee,
        } = &base.fees
        {
            let mut t = base.clone();
            t.fees = Fees::Eip1559 {
                max_priority_fee: max_priority_fee + 1,
                max_fee: *max_fee,
                access_list: access_list.clone(),
            };
            v.push(t);
            let mut t = base.clone();
            t.fees = Fees::Eip1559 {
                max_priority_fee: *max_priority_fee,
                max_fee: *max_fee,
                access_list: vec![],
            };
            v.push(t);
            let mut t = base.clone();
            t.fees = Fees::Eip1559 {
                max_priority_fee: *max_priority_fee,
                max_fee: *max_fee,
                access_list: vec![([0; 20], vec![])],
            };
            v.push(t);
        }
        for x in v {
            assert_ne!(x.signing_hash(), h, "{x:?}");
        }
    }
}

#[test]
fn flipping_any_bit_of_a_raw_transaction_breaks_it_or_changes_it() {
    for raw_hex in [EIP155_RAW, T2_RAW] {
        let raw = hex::decode(raw_hex).unwrap();
        let orig = SignedTx::from_raw(&raw).unwrap();
        let mut changed_sender = 0;
        for i in 0..raw.len() {
            for bit in 0..8 {
                let mut b = raw.clone();
                b[i] ^= 1 << bit;
                if let Ok(x) = SignedTx::from_raw(&b) {
                    assert!(x != orig, "byte {i} bit {bit} gave the same transaction");
                    if x.sender().ok() != orig.sender().ok() {
                        changed_sender += 1;
                    }
                }
            }
        }
        assert!(changed_sender > 0);
    }
}

#[test]
fn signatures_are_low_s_and_round_trip_for_many_keys_and_nonces() {
    let mut r = Rng(1234);
    let mut short_r = 0;
    let mut short_s = 0;
    for i in 0..600u64 {
        let seed = 1 + r.below(254) as u8; // 0xff...ff is above the group order
        let mut tx = if i % 2 == 0 { eip155_tx() } else { t2_tx() };
        tx.nonce = r.next_u64() >> r.below(60);
        tx.value = u128::from(r.next_u64());
        let n = r.below(40) as usize;
        tx.data = r.bytes(n);
        let signed = tx.sign(&key(seed));
        assert!(signed.sig.s <= HALF);
        assert!(signed.sig.y_parity <= 1);
        let raw = signed.raw();
        let back = SignedTx::from_raw(&raw).unwrap();
        assert_eq!(back, signed);
        assert_eq!(back.sender(), Ok(addr_of_seed(seed)));
        if signed.sig.r[0] == 0 {
            short_r += 1;
        }
        if signed.sig.s[0] == 0 {
            short_s += 1;
        }
    }
    // about 1 in 256 signatures has a leading zero byte in r or s; with 600 trials we expect a few
    assert!(
        short_r + short_s >= 1,
        "no leading-zero r or s in 600 signatures; the stripping path is not exercised"
    );
}

const HALF: [u8; 32] = [
    0x7f, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff,
    0x5d, 0x57, 0x6e, 0x73, 0x57, 0xa4, 0x50, 0x1d, 0xdf, 0xe9, 0x2f, 0x46, 0x68, 0x1b, 0x20, 0xa0,
];

#[test]
fn leading_zero_r_or_s_is_stripped_in_the_wire_form_and_restored_on_parse() {
    let mut found = 0;
    for nonce in 0..3000u64 {
        let mut tx = eip155_tx();
        tx.nonce = nonce;
        let signed = tx.sign(&key(3));
        if signed.sig.r[0] != 0 && signed.sig.s[0] != 0 {
            continue;
        }
        found += 1;
        let item = Item::decode(&signed.raw()).unwrap();
        let f = item.as_list().unwrap();
        for (idx, v) in [(7usize, signed.sig.r), (8, signed.sig.s)] {
            let b = f[idx].as_bytes().unwrap();
            assert_eq!(b, &v[v.iter().take_while(|x| **x == 0).count()..]);
            assert_ne!(b.first(), Some(&0));
        }
        assert_eq!(SignedTx::from_raw(&signed.raw()).unwrap(), signed);
        if found == 3 {
            break;
        }
    }
    assert!(found >= 1);
}

fn legacy_fields(signed: &SignedTx) -> Vec<Item> {
    Item::decode(&signed.raw())
        .unwrap()
        .as_list()
        .unwrap()
        .to_vec()
}

fn relist(f: Vec<Item>) -> Vec<u8> {
    Item::List(f).encode()
}

#[test]
fn parsing_refuses_weak_or_malformed_signatures() {
    let signed = eip155_tx().sign(&key(4));
    // high s: n - s is an equivalent signature that EIP-2 forbids
    let mut f = legacy_fields(&signed);
    let n =
        hex::decode("fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141").unwrap();
    let s = signed.sig.s;
    let mut high = [0u8; 32];
    let mut borrow = 0i16;
    for i in (0..32).rev() {
        let mut d = i16::from(n[i]) - i16::from(s[i]) - borrow;
        borrow = 0;
        if d < 0 {
            d += 256;
            borrow = 1;
        }
        high[i] = d as u8;
    }
    f[8] = Item::Bytes(high.iter().skip_while(|b| **b == 0).copied().collect());
    assert_eq!(SignedTx::from_raw(&relist(f)), Err(TxError::HighS));
    // zero r, zero s
    let mut f = legacy_fields(&signed);
    f[7] = Item::Bytes(vec![]);
    assert_eq!(SignedTx::from_raw(&relist(f)), Err(TxError::ZeroSignature));
    let mut f = legacy_fields(&signed);
    f[8] = Item::Bytes(vec![]);
    assert_eq!(SignedTx::from_raw(&relist(f)), Err(TxError::ZeroSignature));
    // r with a leading zero byte (a second encoding)
    let mut f = legacy_fields(&signed);
    let mut rr = vec![0];
    rr.extend_from_slice(&signed.sig.r);
    f[7] = Item::Bytes(rr);
    assert_eq!(SignedTx::from_raw(&relist(f)), Err(TxError::BadField));
    // r longer than 32 bytes
    let mut f = legacy_fields(&signed);
    f[7] = Item::Bytes(vec![1; 33]);
    assert_eq!(SignedTx::from_raw(&relist(f)), Err(TxError::BadField));
}

#[test]
fn parsing_refuses_unprotected_and_invalid_v() {
    let signed = eip155_tx().sign(&key(4));
    for v in [27u128, 28] {
        let mut f = legacy_fields(&signed);
        f[6] = uint(v);
        assert_eq!(
            SignedTx::from_raw(&relist(f)),
            Err(TxError::Unprotected),
            "v={v}"
        );
    }
    for v in [0u128, 1, 26, 29, 34] {
        let mut f = legacy_fields(&signed);
        f[6] = uint(v);
        assert_eq!(SignedTx::from_raw(&relist(f)), Err(TxError::BadV), "v={v}");
    }
    // y_parity 2 in a type 2 transaction
    let t2 = t2_tx().sign(&key(4));
    let raw = t2.raw();
    let mut f = Item::decode(&raw[1..]).unwrap().as_list().unwrap().to_vec();
    f[9] = uint(2);
    let mut bad = vec![TYPE_1559];
    bad.extend(relist(f));
    assert_eq!(SignedTx::from_raw(&bad), Err(TxError::BadV));
}

#[test]
fn parsing_refuses_wrong_shapes() {
    assert_eq!(SignedTx::from_raw(&[]), Err(TxError::Rlp(RlpError::Eof)));
    assert_eq!(
        SignedTx::from_raw(&[0x01, 0xc0]),
        Err(TxError::UnknownType(1))
    );
    assert_eq!(
        SignedTx::from_raw(&[0x03, 0xc0]),
        Err(TxError::UnknownType(3))
    );
    assert_eq!(SignedTx::from_raw(&[0x80]), Err(TxError::UnknownType(0x80)));
    assert_eq!(SignedTx::from_raw(&[0xc0]), Err(TxError::FieldCount));
    assert_eq!(
        SignedTx::from_raw(&[TYPE_1559, 0xc0]),
        Err(TxError::FieldCount)
    );
    // trailing byte
    let mut raw = hex::decode(EIP155_RAW).unwrap();
    raw.push(0);
    assert_eq!(
        SignedTx::from_raw(&raw),
        Err(TxError::Rlp(RlpError::Trailing))
    );
    let mut raw = hex::decode(T2_RAW).unwrap();
    raw.push(0);
    assert_eq!(
        SignedTx::from_raw(&raw),
        Err(TxError::Rlp(RlpError::Trailing))
    );
    // a `to` of 19 bytes
    let signed = eip155_tx().sign(&key(4));
    let mut f = legacy_fields(&signed);
    f[3] = Item::Bytes(vec![1; 19]);
    assert_eq!(SignedTx::from_raw(&relist(f)), Err(TxError::BadField));
    // a nonce with a leading zero
    let mut f = legacy_fields(&signed);
    f[0] = Item::Bytes(vec![0, 9]);
    assert_eq!(
        SignedTx::from_raw(&relist(f)),
        Err(TxError::Rlp(RlpError::BadInteger))
    );
    // access list entries of the wrong shape
    let t2 = t2_tx().sign(&key(4));
    let mut f = Item::decode(&t2.raw()[1..])
        .unwrap()
        .as_list()
        .unwrap()
        .to_vec();
    f[8] = Item::List(vec![Item::List(vec![Item::Bytes(vec![1; 20])])]);
    let mut bad = vec![TYPE_1559];
    bad.extend(relist(f.clone()));
    assert_eq!(SignedTx::from_raw(&bad), Err(TxError::FieldCount));
    f[8] = Item::List(vec![Item::List(vec![
        Item::Bytes(vec![1; 19]),
        Item::List(vec![]),
    ])]);
    let mut bad = vec![TYPE_1559];
    bad.extend(relist(f.clone()));
    assert_eq!(SignedTx::from_raw(&bad), Err(TxError::BadField));
    f[8] = Item::List(vec![Item::List(vec![
        Item::Bytes(vec![1; 20]),
        Item::List(vec![Item::Bytes(vec![1; 31])]),
    ])]);
    let mut bad = vec![TYPE_1559];
    bad.extend(relist(f));
    assert_eq!(SignedTx::from_raw(&bad), Err(TxError::BadField));
}

#[test]
fn contract_creation_has_an_empty_to() {
    let mut tx = eip155_tx();
    tx.to = None;
    tx.data = vec![0x60, 0x00];
    let signed = tx.sign(&key(2));
    let back = SignedTx::from_raw(&signed.raw()).unwrap();
    assert_eq!(back.tx.to, None);
    assert_eq!(back.sender(), Ok(addr_of_seed(2)));
}

#[test]
fn gas_formulas() {
    let mut t = eip155_tx();
    assert_eq!(t.intrinsic_gas(), 21_000);
    t.data = vec![0, 0, 1, 2, 0];
    assert_eq!(t.intrinsic_gas(), 21_000 + 3 * 4 + 2 * 16);
    t.to = None;
    assert_eq!(t.intrinsic_gas(), 21_000 + 3 * 4 + 2 * 16 + 32_000);
    let mut t2 = t2_tx();
    assert_eq!(t2.intrinsic_gas(), 21_000 + 2_400 + 1_900);
    if let Fees::Eip1559 { access_list, .. } = &mut t2.fees {
        access_list.push(([5; 20], vec![]));
    }
    assert_eq!(t2.intrinsic_gas(), 21_000 + 2_400 + 1_900 + 2_400);
    // effective price: min(max_fee, base_fee + priority)
    let t2 = t2_tx();
    assert_eq!(t2.effective_gas_price(10_000_000_000), 12_000_000_000);
    assert_eq!(t2.effective_gas_price(29_000_000_000), 30_000_000_000);
    assert_eq!(t2.effective_gas_price(100_000_000_000), 30_000_000_000);
    assert_eq!(eip155_tx().effective_gas_price(5), 20_000_000_000);
    assert_eq!(t2.fee_cap(), 30_000_000_000);
    assert_eq!(t2.max_cost(), 30_000_000_000 * 21_000 + 12345);
    assert_eq!(
        eip155_tx().max_cost(),
        20_000_000_000 * 21_000 + 1_000_000_000_000_000_000
    );
}

#[test]
fn the_hash_is_keccak_of_the_wire_bytes() {
    let signed = t2_tx().sign(&key(9));
    assert_eq!(signed.hash(), keccak256(&signed.raw()));
    let a: Address = signed.sender().unwrap();
    assert_eq!(a, addr_of_seed(9));
}
```

### `tests/node_tests.rs`

```rust
mod common;
use common::*;
use evmlab::abi::word_from_u128;
use evmlab::address::Address;
use evmlab::keccak::event_topic;
use evmlab::logs::*;
use evmlab::node::{
    Account, Node, SubmitError, TOKEN_CALL_GAS, balance_of_calldata, decode_uint_return,
    transfer_calldata,
};
use evmlab::tx::*;

const GWEI: u128 = 1_000_000_000;
const ETH: u128 = 1_000_000_000_000_000_000;
const CHAIN: u64 = 1337;
const BASE: u128 = 10 * GWEI;

fn node() -> Node {
    Node::new(
        CHAIN,
        BASE,
        &[(addr_of_seed(1), 10 * ETH), (addr_of_seed(2), ETH)],
        &[(addr_of_seed(1), 5_000)],
    )
}

fn eth_tx(node: &Node, from: u8, nonce: u64, to: Address, value: u128) -> SignedTx {
    let _ = node;
    Tx {
        chain_id: CHAIN,
        nonce,
        gas_limit: 21_000,
        to: Some(to),
        value,
        data: vec![],
        fees: Fees::Eip1559 {
            max_priority_fee: 2 * GWEI,
            max_fee: 30 * GWEI,
            access_list: vec![],
        },
    }
    .sign(&key(from))
}

fn token_tx(node: &Node, from: u8, nonce: u64, to: Address, amount: u128) -> SignedTx {
    let data = transfer_calldata(to, amount);
    let mut t = Tx {
        chain_id: CHAIN,
        nonce,
        gas_limit: 100_000,
        to: Some(node.token),
        value: 0,
        data,
        fees: Fees::Eip1559 {
            max_priority_fee: GWEI,
            max_fee: 20 * GWEI,
            access_list: vec![],
        },
    };
    t.gas_limit = t.intrinsic_gas() + TOKEN_CALL_GAS;
    t.sign(&key(from))
}

#[test]
fn genesis() {
    let n = node();
    assert_eq!(n.block_number(), 0);
    assert_eq!(
        n.account(&addr_of_seed(1)),
        Account {
            nonce: 0,
            balance: 10 * ETH
        }
    );
    assert_eq!(n.account(&addr_of_seed(9)), Account::default());
    assert_eq!(n.token_balance(&addr_of_seed(1)), 5_000);
    assert_eq!(n.total_ether(), 11 * ETH);
    assert_eq!(n.initial_ether(), 11 * ETH);
}

#[test]
fn a_plain_transfer_pays_gas_at_the_effective_price_and_splits_it_into_burn_and_tip() {
    let mut n = node();
    let to = addr_of_seed(3);
    let stx = eth_tx(&n, 1, 0, to, ETH);
    let hash = n.submit(&stx.raw()).unwrap();
    assert_eq!(hash, stx.hash());
    assert_eq!(n.pool_len(), 1);
    assert_eq!(n.mine(), 1);
    let r = n.receipt(&hash).unwrap();
    assert!(r.status);
    assert_eq!(r.gas_used, 21_000);
    assert_eq!(r.effective_gas_price, 12 * GWEI); // base 10 + tip 2, under the 30 cap
    assert_eq!(r.block_number, 1);
    assert_eq!(r.from, addr_of_seed(1));
    let fee = 21_000 * 12 * GWEI;
    assert_eq!(
        n.account(&addr_of_seed(1)),
        Account {
            nonce: 1,
            balance: 10 * ETH - ETH - fee
        }
    );
    assert_eq!(n.account(&to).balance, ETH);
    assert_eq!(n.burned(), 21_000 * BASE);
    assert_eq!(n.account(&n.coinbase).balance, 21_000 * 2 * GWEI);
    assert_eq!(n.total_ether(), n.initial_ether());
    assert_eq!(n.pool_len(), 0);
}

#[test]
fn the_fee_cap_limits_the_price() {
    let mut n = Node::new(CHAIN, 29 * GWEI, &[(addr_of_seed(1), 10 * ETH)], &[]);
    let stx = eth_tx(&n, 1, 0, addr_of_seed(3), 1);
    let h = n.submit(&stx.raw()).unwrap();
    n.mine();
    assert_eq!(n.receipt(&h).unwrap().effective_gas_price, 30 * GWEI); // 29 + 2 would be 31
    assert_eq!(n.burned(), 21_000 * 29 * GWEI);
    assert_eq!(n.account(&n.coinbase).balance, 21_000 * GWEI);
}

#[test]
fn a_legacy_transaction_pays_its_gas_price() {
    let mut n = node();
    let stx = Tx {
        chain_id: CHAIN,
        nonce: 0,
        gas_limit: 21_000,
        to: Some(addr_of_seed(3)),
        value: 5,
        data: vec![],
        fees: Fees::Legacy {
            gas_price: 15 * GWEI,
        },
    }
    .sign(&key(1));
    let h = n.submit(&stx.raw()).unwrap();
    n.mine();
    assert_eq!(n.receipt(&h).unwrap().effective_gas_price, 15 * GWEI);
    assert_eq!(n.burned(), 21_000 * BASE);
    assert_eq!(n.account(&n.coinbase).balance, 21_000 * 5 * GWEI);
    assert_eq!(n.total_ether(), n.initial_ether());
}

#[test]
fn the_pool_rejects_for_a_named_reason() {
    let mut n = node();
    let to = addr_of_seed(3);
    assert_eq!(n.submit(&[1, 2, 3]), Err(SubmitError::Malformed));
    // wrong chain
    let mut t = eth_tx(&n, 1, 0, to, 1).tx;
    t.chain_id = 1;
    assert_eq!(
        n.submit(&t.sign(&key(1)).raw()),
        Err(SubmitError::WrongChain)
    );
    // nonce too high / low
    assert_eq!(
        n.submit(&eth_tx(&n, 1, 1, to, 1).raw()),
        Err(SubmitError::NonceTooHigh)
    );
    let first = eth_tx(&n, 1, 0, to, 1);
    n.submit(&first.raw()).unwrap();
    assert_eq!(n.submit(&first.raw()), Err(SubmitError::AlreadyKnown));
    // another transaction with the same nonce but different content
    assert_eq!(
        n.submit(&eth_tx(&n, 1, 0, to, 2).raw()),
        Err(SubmitError::NonceTooLow)
    );
    n.submit(&eth_tx(&n, 1, 1, to, 1).raw()).unwrap();
    assert_eq!(n.pending_nonce(&addr_of_seed(1)), 2);
    n.mine();
    assert_eq!(n.submit(&first.raw()), Err(SubmitError::NonceTooLow));
    // intrinsic gas
    let mut t = eth_tx(&n, 1, 2, to, 1).tx;
    t.gas_limit = 20_999;
    assert_eq!(
        n.submit(&t.sign(&key(1)).raw()),
        Err(SubmitError::IntrinsicGas)
    );
    t = eth_tx(&n, 1, 2, to, 1).tx;
    t.gas_limit = 21_000;
    t.data = vec![1];
    assert_eq!(
        n.submit(&t.sign(&key(1)).raw()),
        Err(SubmitError::IntrinsicGas)
    );
    // fee cap below the base fee, both types
    let mut t = eth_tx(&n, 1, 2, to, 1).tx;
    t.fees = Fees::Eip1559 {
        max_priority_fee: 0,
        max_fee: BASE - 1,
        access_list: vec![],
    };
    assert_eq!(
        n.submit(&t.clone().sign(&key(1)).raw()),
        Err(SubmitError::FeeCapBelowBaseFee)
    );
    t.fees = Fees::Legacy {
        gas_price: BASE - 1,
    };
    assert_eq!(
        n.submit(&t.clone().sign(&key(1)).raw()),
        Err(SubmitError::FeeCapBelowBaseFee)
    );
    t.fees = Fees::Legacy { gas_price: BASE };
    assert!(n.submit(&t.sign(&key(1)).raw()).is_ok());
}

#[test]
fn funds_are_checked_against_the_worst_case_not_the_expected_cost() {
    let mut n = Node::new(
        CHAIN,
        BASE,
        &[(addr_of_seed(1), 21_000 * 30 * GWEI + 100)],
        &[],
    );
    // value 100 plus gas limit times the fee cap is exactly the balance: accepted
    let ok = eth_tx(&n, 1, 0, addr_of_seed(3), 100);
    assert!(n.submit(&ok.raw()).is_ok());
    // one more wei of value: refused, even though the actual price would be much lower than the cap
    let mut n = Node::new(
        CHAIN,
        BASE,
        &[(addr_of_seed(1), 21_000 * 30 * GWEI + 100)],
        &[],
    );
    let too_much = eth_tx(&n, 1, 0, addr_of_seed(3), 101);
    assert_eq!(
        n.submit(&too_much.raw()),
        Err(SubmitError::InsufficientFunds)
    );
    assert_eq!(
        n.submit(&eth_tx(&n, 7, 0, addr_of_seed(3), 0).raw()),
        Err(SubmitError::InsufficientFunds)
    );
}

#[test]
fn a_token_transfer_moves_tokens_emits_a_log_and_costs_more_gas() {
    let mut n = node();
    let to = addr_of_seed(4);
    let stx = token_tx(&n, 1, 0, to, 1_000);
    let h = n.submit(&stx.raw()).unwrap();
    n.mine();
    let r = n.receipt(&h).unwrap().clone();
    assert!(r.status);
    assert_eq!(r.gas_used, stx.tx.intrinsic_gas() + TOKEN_CALL_GAS);
    assert!(r.gas_used > 21_000 + 30_000);
    assert_eq!(n.token_balance(&addr_of_seed(1)), 4_000);
    assert_eq!(n.token_balance(&to), 1_000);
    assert_eq!(r.logs.len(), 1);
    assert_eq!(
        decode_transfer(&r.logs[0]),
        Ok(Transfer {
            token: n.token,
            from: addr_of_seed(1),
            to,
            value: 1_000
        })
    );
    // no ether moved, only gas was paid
    assert_eq!(n.account(&to).balance, 0);
    assert_eq!(n.total_ether(), n.initial_ether());
}

#[test]
fn a_reverted_call_still_costs_gas_and_a_nonce_and_leaves_no_log() {
    let mut n = node();
    let stx = token_tx(&n, 2, 0, addr_of_seed(4), 1); // seed 2 holds no tokens
    let before = n.account(&addr_of_seed(2)).balance;
    let h = n.submit(&stx.raw()).unwrap();
    n.mine();
    let r = n.receipt(&h).unwrap();
    assert!(!r.status);
    assert!(r.logs.is_empty());
    assert_eq!(r.gas_used, stx.tx.intrinsic_gas() + TOKEN_CALL_GAS);
    assert_eq!(n.account(&addr_of_seed(2)).nonce, 1);
    assert_eq!(
        before - n.account(&addr_of_seed(2)).balance,
        u128::from(r.gas_used) * r.effective_gas_price
    );
    assert_eq!(n.token_balance(&addr_of_seed(4)), 0);
    assert_eq!(n.total_ether(), n.initial_ether());
}

#[test]
fn running_out_of_gas_uses_all_the_gas_and_reverts() {
    let mut n = node();
    let mut stx = token_tx(&n, 1, 0, addr_of_seed(4), 10).tx;
    stx.gas_limit = stx.intrinsic_gas() + TOKEN_CALL_GAS - 1;
    let signed = stx.sign(&key(1));
    let h = n.submit(&signed.raw()).unwrap();
    n.mine();
    let r = n.receipt(&h).unwrap();
    assert!(!r.status);
    assert_eq!(r.gas_used, signed.tx.gas_limit);
    assert_eq!(n.token_balance(&addr_of_seed(4)), 0);
    assert_eq!(n.token_balance(&addr_of_seed(1)), 5_000);
    // one more unit of gas and it succeeds
    let mut stx = token_tx(&n, 1, 1, addr_of_seed(4), 10).tx;
    stx.gas_limit = stx.intrinsic_gas() + TOKEN_CALL_GAS;
    let h = n.submit(&stx.sign(&key(1)).raw()).unwrap();
    n.mine();
    assert!(n.receipt(&h).unwrap().status);
}

#[test]
fn value_sent_to_the_token_and_unknown_calls_revert() {
    let mut n = node();
    let mut t = token_tx(&n, 1, 0, addr_of_seed(4), 10).tx;
    t.value = 1;
    t.gas_limit += 10;
    let h = n.submit(&t.sign(&key(1)).raw()).unwrap();
    // a call to an unknown function
    let mut t = token_tx(&n, 1, 1, addr_of_seed(4), 10).tx;
    t.data = vec![0xde, 0xad, 0xbe, 0xef];
    t.gas_limit = t.intrinsic_gas() + TOKEN_CALL_GAS;
    let h2 = n.submit(&t.sign(&key(1)).raw()).unwrap();
    // a contract creation is not modelled
    let mut t = eth_tx(&n, 1, 2, addr_of_seed(4), 0).tx;
    t.to = None;
    t.gas_limit = t.intrinsic_gas();
    let h3 = n.submit(&t.sign(&key(1)).raw()).unwrap();
    n.mine();
    for h in [h, h2, h3] {
        assert!(!n.receipt(&h).unwrap().status);
    }
    assert_eq!(n.token_balance(&addr_of_seed(1)), 5_000);
    assert_eq!(n.token_balance(&addr_of_seed(4)), 0);
    assert_eq!(n.account(&n.token).balance, 0);
    assert_eq!(n.account(&addr_of_seed(1)).nonce, 3);
    assert_eq!(n.total_ether(), n.initial_ether());
}

#[test]
fn a_transaction_that_became_unaffordable_is_dropped_at_mining() {
    let cap = 21_000 * 30 * GWEI;
    let mut n = Node::new(CHAIN, BASE, &[(addr_of_seed(1), 3 * cap)], &[]);
    // each passes the pool on its own (worst case cap + value <= balance); together they cannot both run
    let a = eth_tx(&n, 1, 0, addr_of_seed(3), cap + cap / 2);
    let b = eth_tx(&n, 1, 1, addr_of_seed(3), cap / 2);
    n.submit(&a.raw()).unwrap();
    // b's worst case is 1 cap of gas + 0.5 cap of value = 1.5 cap, within the 3 cap balance now;
    // after a runs (1.5 cap of value + 0.4 cap of gas) only 1.1 cap is left
    n.submit(&b.raw()).unwrap();
    n.mine();
    assert!(n.receipt(&a.hash()).is_some());
    assert!(n.receipt(&b.hash()).is_none());
    assert_eq!(n.account(&addr_of_seed(1)).nonce, 1);
    assert_eq!(n.pool_len(), 0);
    assert_eq!(n.total_ether(), n.initial_ether());
}

#[test]
fn view_calls() {
    let n = node();
    let out = n
        .call(&n.token, &balance_of_calldata(addr_of_seed(1)))
        .unwrap();
    assert_eq!(out, word_from_u128(5_000).to_vec());
    assert_eq!(decode_uint_return(&out), Some(word_from_u128(5_000)));
    assert_eq!(decode_uint_return(&out[..31]), None);
    assert_eq!(
        n.call(&n.token, &balance_of_calldata(addr_of_seed(9)))
            .unwrap(),
        word_from_u128(0).to_vec()
    );
    assert!(n.call(&n.token, &[1, 2, 3, 4]).is_err());
    assert!(n.call(&n.token, &transfer_calldata([0; 20], 1)).is_err());
    // a call to anything else returns nothing
    assert_eq!(
        n.call(&addr_of_seed(1), &balance_of_calldata(addr_of_seed(1)))
            .unwrap(),
        Vec::<u8>::new()
    );
}

#[test]
fn logs_are_filtered_by_range_emitter_and_topics() {
    let mut n = node();
    let (a, b) = (addr_of_seed(4), addr_of_seed(5));
    n.submit(&token_tx(&n, 1, 0, a, 100).raw()).unwrap();
    n.submit(&token_tx(&n, 1, 1, b, 200).raw()).unwrap();
    n.mine(); // block 1
    n.submit(&token_tx(&n, 1, 2, a, 300).raw()).unwrap();
    n.mine(); // block 2
    n.mine(); // block 3, empty
    let topic0 = event_topic(evmlab::logs::TRANSFER_SIG);
    assert_eq!(n.logs(0, 3, None, &[]).len(), 3);
    assert_eq!(n.logs(1, 1, None, &[]).len(), 2);
    assert_eq!(n.logs(2, 2, None, &[]).len(), 1);
    assert_eq!(n.logs(3, 9, None, &[]).len(), 0);
    assert_eq!(n.logs(2, 1, None, &[]).len(), 0);
    assert_eq!(n.logs(0, 3, Some(&n.token), &[]).len(), 3);
    assert_eq!(n.logs(0, 3, Some(&a), &[]).len(), 0);
    assert_eq!(n.logs(0, 3, None, &[vec![topic0]]).len(), 3);
    assert_eq!(n.logs(0, 3, None, &[vec![[9; 32]]]).len(), 0);
    // topic 2 is the recipient: a only
    let to_a = vec![vec![], vec![], vec![address_topic(&a)]];
    let found = n.logs(0, 3, None, &to_a);
    assert_eq!(found.len(), 2);
    assert_eq!(
        found.iter().map(|(blk, _)| *blk).collect::<Vec<_>>(),
        vec![1, 2]
    );
    // an OR list at one position
    let either = vec![vec![], vec![], vec![address_topic(&a), address_topic(&b)]];
    assert_eq!(n.logs(0, 3, None, &either).len(), 3);
    // a filter with more positions than the log has topics matches nothing
    assert_eq!(
        n.logs(0, 3, None, &[vec![], vec![], vec![], vec![[1; 32]]])
            .len(),
        0
    );
}

#[test]
fn ether_and_tokens_are_conserved_under_random_traffic() {
    let mut r = Rng(555);
    let mut n = Node::new(
        CHAIN,
        BASE,
        &[
            (addr_of_seed(1), 5 * ETH),
            (addr_of_seed(2), 5 * ETH),
            (addr_of_seed(3), 1_000),
        ],
        &[(addr_of_seed(1), 10_000), (addr_of_seed(2), 10_000)],
    );
    let token_total = 20_000u128;
    for round in 0..40 {
        for _ in 0..r.below(6) {
            let from = 1 + r.below(3) as u8;
            let nonce = n.pending_nonce(&addr_of_seed(from));
            let to = addr_of_seed(1 + r.below(6) as u8);
            let stx = if r.below(2) == 0 {
                eth_tx(&n, from, nonce, to, u128::from(r.below(1_000_000_000)))
            } else {
                token_tx(&n, from, nonce, to, u128::from(r.below(8_000)))
            };
            let _ = n.submit(&stx.raw());
        }
        n.mine();
        assert_eq!(n.total_ether(), n.initial_ether(), "round {round}");
        let tokens: u128 = (1..=6).map(|s| n.token_balance(&addr_of_seed(s))).sum();
        assert_eq!(tokens, token_total, "round {round}");
        assert_eq!(n.block_number(), round + 1);
    }
}

#[test]
fn transfer_logs_decode_strictly() {
    let (f, t) = (addr_of_seed(1), addr_of_seed(2));
    let good = transfer_log([9; 20], f, t, 77);
    assert_eq!(
        decode_transfer(&good),
        Ok(Transfer {
            token: [9; 20],
            from: f,
            to: t,
            value: 77
        })
    );
    let mut l = good.clone();
    l.topics[0] = [1; 32];
    assert_eq!(decode_transfer(&l), Err(LogError::NotATransfer));
    let mut l = good.clone();
    l.topics.clear();
    assert_eq!(decode_transfer(&l), Err(LogError::NotATransfer));
    // four topics: an ERC-721 transfer has the same first topic
    let mut l = good.clone();
    l.topics.push([0; 32]);
    assert_eq!(decode_transfer(&l), Err(LogError::WrongTopicCount));
    let mut l = good.clone();
    l.topics.pop();
    assert_eq!(decode_transfer(&l), Err(LogError::WrongTopicCount));
    let mut l = good.clone();
    l.topics[1][0] = 1;
    assert_eq!(decode_transfer(&l), Err(LogError::DirtyTopic));
    let mut l = good.clone();
    l.topics[2][11] = 1;
    assert_eq!(decode_transfer(&l), Err(LogError::DirtyTopic));
    let mut l = good.clone();
    l.data.pop();
    assert_eq!(decode_transfer(&l), Err(LogError::BadData));
    let mut l = good.clone();
    l.data.push(0);
    assert_eq!(decode_transfer(&l), Err(LogError::BadData));
    let mut l = good.clone();
    l.data[0] = 1;
    assert_eq!(decode_transfer(&l), Err(LogError::ValueTooLarge));
    assert_eq!(address_topic(&f)[12..], f);
    assert_eq!(address_topic(&f)[..12], [0; 12]);
}
```

### `tests/rpc_tests.rs`

```rust
mod common;
use common::*;
use evmlab::address::to_checksum;
use evmlab::node::*;
use evmlab::rpc::*;
use evmlab::tx::*;
use serde_json::{Value, json};

const ETH: u128 = 1_000_000_000_000_000_000;
const GWEI: u128 = 1_000_000_000;

fn node() -> Node {
    Node::new(
        1337,
        10 * GWEI,
        &[(addr_of_seed(1), 10 * ETH)],
        &[(addr_of_seed(1), 5_000)],
    )
}
fn req(n: &mut Node, method: &str, params: Value) -> Value {
    let r = json!({"jsonrpc":"2.0","id":7,"method":method,"params":params}).to_string();
    serde_json::from_str(&handle(n, &r)).unwrap()
}
fn ck(seed: u8) -> String {
    to_checksum(&addr_of_seed(seed))
}
fn send(from: u8, nonce: u64, to: [u8; 20], value: u128) -> Vec<u8> {
    Tx {
        chain_id: 1337,
        nonce,
        gas_limit: 21_000,
        to: Some(to),
        value,
        data: vec![],
        fees: Fees::Eip1559 {
            max_priority_fee: 2 * GWEI,
            max_fee: 30 * GWEI,
            access_list: vec![],
        },
    }
    .sign(&key(from))
    .raw()
}

#[test]
fn quantities_are_minimal_hex() {
    assert_eq!(quantity(0), "0x0");
    assert_eq!(quantity(1), "0x1");
    assert_eq!(quantity(255), "0xff");
    assert_eq!(quantity(256), "0x100");
    assert_eq!(parse_quantity("0x0"), Some(0));
    assert_eq!(parse_quantity("0x1F"), Some(31));
    assert_eq!(parse_quantity("0xff"), Some(255));
    for bad in [
        "0x", "0x00", "0x01", "1", "", "0xg", "-0x1", "0x 1", "0x1_0",
    ] {
        assert_eq!(parse_quantity(bad), None, "{bad}");
    }
    assert_eq!(
        parse_quantity(&format!("0x{}", "f".repeat(32))),
        Some(u128::MAX)
    );
    assert_eq!(parse_quantity(&format!("0x1{}", "0".repeat(32))), None);
    assert_eq!(data_hex(&[0, 1]), "0x0001");
    assert_eq!(data_hex(&[]), "0x");
    assert_eq!(parse_data("0x0001"), Some(vec![0, 1]));
    assert_eq!(parse_data("0x"), Some(vec![]));
    assert_eq!(parse_data("0x1"), None);
    assert_eq!(parse_data("01"), None);
}

#[test]
fn reads() {
    let mut n = node();
    assert_eq!(
        req(&mut n, "eth_chainId", json!([]))["result"],
        json!("0x539")
    );
    assert_eq!(
        req(&mut n, "eth_blockNumber", json!([]))["result"],
        json!("0x0")
    );
    assert_eq!(
        req(&mut n, "eth_gasPrice", json!([]))["result"],
        json!(quantity(10 * GWEI))
    );
    let r = req(&mut n, "eth_getBalance", json!([ck(1), "latest"]));
    assert_eq!(r["result"], json!(quantity(10 * ETH)));
    assert_eq!(r["id"], json!(7));
    assert_eq!(
        req(&mut n, "eth_getBalance", json!([ck(9)]))["result"],
        json!("0x0")
    );
    assert_eq!(
        req(&mut n, "eth_getBalance", json!([ck(1).to_lowercase()]))["result"],
        json!(quantity(10 * ETH))
    );
}

#[test]
fn send_mine_receipt_and_nonce_views() {
    let mut n = node();
    let raw = send(1, 0, addr_of_seed(2), ETH);
    let r = req(&mut n, "eth_sendRawTransaction", json!([data_hex(&raw)]));
    let hash = r["result"].as_str().unwrap().to_string();
    assert_eq!(hash.len(), 66);
    assert_eq!(
        req(&mut n, "eth_getTransactionCount", json!([ck(1), "latest"]))["result"],
        json!("0x0")
    );
    assert_eq!(
        req(&mut n, "eth_getTransactionCount", json!([ck(1), "pending"]))["result"],
        json!("0x1")
    );
    assert_eq!(
        req(&mut n, "eth_getTransactionReceipt", json!([hash.clone()]))["result"],
        Value::Null
    );
    assert_eq!(req(&mut n, "evm_mine", json!([]))["result"], json!("0x1"));
    assert_eq!(
        req(&mut n, "eth_blockNumber", json!([]))["result"],
        json!("0x1")
    );
    assert_eq!(
        req(&mut n, "eth_getTransactionCount", json!([ck(1)]))["result"],
        json!("0x1")
    );
    let rc = req(&mut n, "eth_getTransactionReceipt", json!([hash.clone()]))["result"].clone();
    assert_eq!(rc["status"], json!("0x1"));
    assert_eq!(rc["gasUsed"], json!("0x5208"));
    assert_eq!(rc["blockNumber"], json!("0x1"));
    assert_eq!(rc["from"], json!(ck(1)));
    assert_eq!(rc["effectiveGasPrice"], json!(quantity(12 * GWEI)));
    assert_eq!(rc["transactionHash"], json!(hash));
    assert_eq!(rc["logs"], json!([]));
    assert_eq!(
        req(&mut n, "eth_getBalance", json!([ck(2)]))["result"],
        json!(quantity(ETH))
    );
}

#[test]
fn rejections_use_the_node_style_code_and_text() {
    let mut n = node();
    let raw = send(1, 1, addr_of_seed(2), 1);
    let r = req(&mut n, "eth_sendRawTransaction", json!([data_hex(&raw)]));
    assert_eq!(r["error"]["code"], json!(TX_REJECTED));
    assert_eq!(r["error"]["message"], json!("nonce too high"));
    let ok = send(1, 0, addr_of_seed(2), 1);
    req(&mut n, "eth_sendRawTransaction", json!([data_hex(&ok)]));
    let r = req(&mut n, "eth_sendRawTransaction", json!([data_hex(&ok)]));
    assert_eq!(r["error"]["message"], json!("already known"));
    let r = req(&mut n, "eth_sendRawTransaction", json!(["0x0102"]));
    assert_eq!(r["error"]["message"], json!("invalid transaction"));
    let poor = send(9, 0, addr_of_seed(2), 1);
    let r = req(&mut n, "eth_sendRawTransaction", json!([data_hex(&poor)]));
    assert_eq!(
        r["error"]["message"],
        json!("insufficient funds for gas * price + value")
    );
    req(&mut n, "evm_mine", json!([]));
    let r = req(&mut n, "eth_sendRawTransaction", json!([data_hex(&ok)]));
    assert_eq!(r["error"]["message"], json!("nonce too low"));
}

#[test]
fn eth_call_reads_the_token_balance() {
    let mut n = node();
    let tk = to_checksum(&n.token);
    let data = data_hex(&balance_of_calldata(addr_of_seed(1)));
    let r = req(
        &mut n,
        "eth_call",
        json!([{"to": tk, "data": data.clone()}, "latest"]),
    );
    assert_eq!(
        r["result"],
        json!(data_hex(&evmlab::abi::word_from_u128(5_000)))
    );
    // `input` is accepted as an alias of `data`
    let r = req(&mut n, "eth_call", json!([{"to": tk, "input": data}]));
    assert_eq!(
        r["result"],
        json!(data_hex(&evmlab::abi::word_from_u128(5_000)))
    );
    let r = req(
        &mut n,
        "eth_call",
        json!([{"to": tk, "data": "0x01020304"}]),
    );
    assert_eq!(r["error"]["code"], json!(EXECUTION_REVERTED));
    assert_eq!(r["error"]["message"], json!("execution reverted"));
    assert_eq!(
        req(&mut n, "eth_call", json!([{"data": "0x"}]))["error"]["code"],
        json!(INVALID_PARAMS)
    );
    assert_eq!(
        req(&mut n, "eth_call", json!([{"to": ck(1)}]))["error"]["code"],
        json!(INVALID_PARAMS)
    );
    assert_eq!(
        req(&mut n, "eth_call", json!(["x"]))["error"]["code"],
        json!(INVALID_PARAMS)
    );
}

fn token_send(n: &mut Node, nonce: u64, to: [u8; 20], amount: u128) -> Vec<u8> {
    let data = transfer_calldata(to, amount);
    let mut t = Tx {
        chain_id: 1337,
        nonce,
        gas_limit: 0,
        to: Some(n.token),
        value: 0,
        data,
        fees: Fees::Eip1559 {
            max_priority_fee: GWEI,
            max_fee: 20 * GWEI,
            access_list: vec![],
        },
    };
    t.gas_limit = t.intrinsic_gas() + TOKEN_CALL_GAS;
    t.sign(&key(1)).raw()
}

#[test]
fn get_logs_with_filters() {
    let mut n = node();
    let tk = to_checksum(&n.token);
    let (a, b) = (addr_of_seed(4), addr_of_seed(5));
    for (i, (to, amt)) in [(a, 10u128), (b, 20), (a, 30)].into_iter().enumerate() {
        let raw = token_send(&mut n, i as u64, to, amt);
        req(&mut n, "eth_sendRawTransaction", json!([data_hex(&raw)]));
    }
    req(&mut n, "evm_mine", json!([]));
    let topic0 = data_hex(&evmlab::keccak::event_topic(
        "Transfer(address,address,uint256)",
    ));
    let r = req(
        &mut n,
        "eth_getLogs",
        json!([{"fromBlock": "0x1", "toBlock": "latest"}]),
    );
    let logs = r["result"].as_array().unwrap();
    assert_eq!(logs.len(), 3);
    assert_eq!(logs[0]["blockNumber"], json!("0x1"));
    assert_eq!(logs[0]["address"], json!(tk));
    assert_eq!(logs[0]["topics"][0], json!(topic0));
    assert_eq!(
        logs[0]["data"],
        json!(data_hex(&evmlab::abi::word_from_u128(10)))
    );
    let to_a = data_hex(&evmlab::logs::address_topic(&a));
    let r = req(
        &mut n,
        "eth_getLogs",
        json!([{"fromBlock": "earliest", "toBlock": "latest", "topics": [topic0.clone(), null, to_a.clone()]}]),
    );
    assert_eq!(r["result"].as_array().unwrap().len(), 2);
    let r = req(
        &mut n,
        "eth_getLogs",
        json!([{"topics": [[topic0.clone(), "0x".to_string() + &"11".repeat(32)], null, [to_a, data_hex(&evmlab::logs::address_topic(&b))]]}]),
    );
    assert_eq!(r["result"].as_array().unwrap().len(), 3);
    let r = req(
        &mut n,
        "eth_getLogs",
        json!([{"address": tk, "fromBlock": "0x2"}]),
    );
    assert_eq!(r["result"], json!([]));
    let r = req(&mut n, "eth_getLogs", json!([{"address": ck(7)}]));
    assert_eq!(r["result"], json!([]));
    // receipts carry the same logs
    let raw = token_send(&mut n, 3, a, 1);
    let h = req(&mut n, "eth_sendRawTransaction", json!([data_hex(&raw)]))["result"].clone();
    req(&mut n, "evm_mine", json!([]));
    let rc = req(&mut n, "eth_getTransactionReceipt", json!([h]));
    assert_eq!(rc["result"]["logs"].as_array().unwrap().len(), 1);
    assert_eq!(rc["result"]["status"], json!("0x1"));
}

#[test]
fn protocol_and_parameter_errors() {
    let mut n = node();
    let r: Value = serde_json::from_str(&handle(&mut n, "{nope")).unwrap();
    assert_eq!(r["error"]["code"], json!(PARSE_ERROR));
    assert_eq!(r["id"], Value::Null);
    let r: Value = serde_json::from_str(&handle(&mut n, r#"{"jsonrpc":"2.0","id":1}"#)).unwrap();
    assert_eq!(r["error"]["code"], json!(INVALID_REQUEST));
    let r: Value = serde_json::from_str(&handle(
        &mut n,
        r#"{"jsonrpc":"1.0","id":1,"method":"eth_chainId"}"#,
    ))
    .unwrap();
    assert_eq!(r["error"]["code"], json!(INVALID_REQUEST));
    assert_eq!(
        req(&mut n, "eth_nope", json!([]))["error"]["code"],
        json!(METHOD_NOT_FOUND)
    );
    let r: Value = serde_json::from_str(&handle(
        &mut n,
        r#"{"jsonrpc":"2.0","id":"x","method":"eth_chainId"}"#,
    ))
    .unwrap();
    assert_eq!(r["result"], json!("0x539"));
    assert_eq!(r["id"], json!("x"));
    let r: Value = serde_json::from_str(&handle(
        &mut n,
        r#"{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":{}}"#,
    ))
    .unwrap();
    assert_eq!(r["error"]["code"], json!(INVALID_PARAMS));
    for (m, p) in [
        ("eth_getBalance", json!([])),
        ("eth_getBalance", json!(["0x12"])),
        ("eth_getBalance", json!([5])),
        (
            "eth_getBalance",
            json!(["0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAEd"]),
        ), // wrong capital: bad checksum
        ("eth_getTransactionCount", json!([ck(1), "yesterday"])),
        ("eth_sendRawTransaction", json!([])),
        ("eth_sendRawTransaction", json!(["0xzz"])),
        ("eth_getTransactionReceipt", json!(["0x1234"])),
        ("eth_getLogs", json!([])),
        ("eth_getLogs", json!([{"fromBlock": "0x01"}])),
        ("eth_getLogs", json!([{"fromBlock": 1}])),
        ("eth_getLogs", json!([{"address": "nope"}])),
        ("eth_getLogs", json!([{"topics": "0x00"}])),
        ("eth_getLogs", json!([{"topics": [5]}])),
        ("eth_getLogs", json!([{"topics": ["0x1234"]}])),
        ("eth_getLogs", json!([{"topics": [["0x1234"]]}])),
    ] {
        assert_eq!(
            req(&mut n, m, p.clone())["error"]["code"],
            json!(INVALID_PARAMS),
            "{m} {p}"
        );
    }
}

#[test]
fn block_tags() {
    let mut n = node();
    req(&mut n, "evm_mine", json!([]));
    req(&mut n, "evm_mine", json!([]));
    for (tag, count_at_most) in [
        ("earliest", 0),
        ("latest", 0),
        ("pending", 0),
        ("safe", 0),
        ("finalized", 0),
        ("0x1", 0),
    ] {
        let r = req(
            &mut n,
            "eth_getLogs",
            json!([{"fromBlock": tag, "toBlock": tag}]),
        );
        assert_eq!(
            r["result"].as_array().unwrap().len(),
            count_at_most,
            "{tag}"
        );
    }
}
```

### `tests/gap_tests.rs`

```rust
//! Tests written by reading the planned mutant list: each pins something the earlier tests left loose.
mod common;
use common::*;
use evmlab::abi::*;
use evmlab::node::*;
use evmlab::rlp::*;
use evmlab::rpc::*;
use evmlab::tx::*;
use serde_json::{Value, json};

const GWEI: u128 = 1_000_000_000;
const ETH: u128 = 1_000_000_000_000_000_000;

fn nested(levels: usize) -> Item {
    let mut it = Item::List(vec![]);
    for _ in 0..levels {
        it = Item::List(vec![it]);
    }
    it
}

#[test]
fn nesting_limit_is_exactly_32_levels_below_the_top() {
    assert_eq!(MAX_DEPTH, 32);
    assert_eq!(Item::decode(&nested(32).encode()), Ok(nested(32)));
    assert_eq!(Item::decode(&nested(33).encode()), Err(RlpError::TooDeep));
}

#[test]
fn zero_argument_calls_and_zero_words() {
    let data = encode_call("ping()", &[]).unwrap();
    assert_eq!(data.len(), 4);
    assert_eq!(decode_call("ping()", &data), Ok(vec![]));
    assert_eq!(word_from_i128(0), [0; 32]);
    assert_eq!(word_from_i128(-1), [0xff; 32]);
    assert_eq!(word_from_i128(i128::MIN)[..16], [0xff; 16]);
    assert_eq!(word_from_u128(0), [0; 32]);
}

fn base_tx() -> Tx {
    Tx {
        chain_id: 1337,
        nonce: 1,
        gas_limit: 21_000,
        to: Some([0x22; 20]),
        value: 1,
        data: vec![],
        fees: Fees::Eip1559 {
            max_priority_fee: GWEI,
            max_fee: 30 * GWEI,
            access_list: vec![],
        },
    }
}

fn with_s(signed: &SignedTx, s: [u8; 32]) -> Vec<u8> {
    let mut x = signed.clone();
    x.sig.s = s;
    x.raw()
}

#[test]
fn the_s_boundary_is_exact() {
    let signed = base_tx().sign(&key(6));
    let half: [u8; 32] =
        hex::decode("7fffffffffffffffffffffffffffffff5d576e7357a4501ddfe92f46681b20a0")
            .unwrap()
            .try_into()
            .unwrap();
    let mut above = half;
    above[31] += 1;
    assert!(SignedTx::from_raw(&with_s(&signed, half)).is_ok());
    assert_eq!(
        SignedTx::from_raw(&with_s(&signed, above)),
        Err(TxError::HighS)
    );
}

#[test]
fn field_counts_are_exact() {
    let signed = base_tx().sign(&key(6));
    let mut f = Item::decode(&signed.raw()[1..])
        .unwrap()
        .as_list()
        .unwrap()
        .to_vec();
    f.push(Item::Bytes(vec![]));
    let mut raw = vec![TYPE_1559];
    raw.extend(Item::List(f).encode());
    assert_eq!(SignedTx::from_raw(&raw), Err(TxError::FieldCount));
    let legacy = Tx {
        fees: Fees::Legacy {
            gas_price: 20 * GWEI,
        },
        ..base_tx()
    }
    .sign(&key(6));
    let mut f = Item::decode(&legacy.raw())
        .unwrap()
        .as_list()
        .unwrap()
        .to_vec();
    f.push(Item::Bytes(vec![]));
    assert_eq!(
        SignedTx::from_raw(&Item::List(f).encode()),
        Err(TxError::FieldCount)
    );
    // an access-list entry with three elements
    let mut f = Item::decode(&signed.raw()[1..])
        .unwrap()
        .as_list()
        .unwrap()
        .to_vec();
    f[8] = Item::List(vec![Item::List(vec![
        Item::Bytes(vec![1; 20]),
        Item::List(vec![]),
        Item::List(vec![]),
    ])]);
    let mut raw = vec![TYPE_1559];
    raw.extend(Item::List(f).encode());
    assert_eq!(SignedTx::from_raw(&raw), Err(TxError::FieldCount));
}

#[test]
fn chain_id_zero_is_the_smallest_protected_legacy_v() {
    let t = Tx {
        chain_id: 0,
        fees: Fees::Legacy { gas_price: GWEI },
        ..base_tx()
    }
    .sign(&key(6));
    let item = Item::decode(&t.raw()).unwrap();
    let v = item.as_list().unwrap()[6].as_u64().unwrap();
    assert!(v == 35 || v == 36);
    assert_eq!(SignedTx::from_raw(&t.raw()), Ok(t));
}

#[test]
fn access_list_gas_counts_every_key() {
    let mut t = base_tx();
    t.fees = Fees::Eip1559 {
        max_priority_fee: 0,
        max_fee: 0,
        access_list: vec![
            ([1; 20], vec![[1; 32], [2; 32], [3; 32]]),
            ([2; 20], vec![[1; 32]]),
        ],
    };
    assert_eq!(
        t.intrinsic_gas(),
        21_000 + 2_400 + 3 * 1_900 + 2_400 + 1_900
    );
    assert_eq!(
        Tx {
            data: vec![0],
            ..base_tx()
        }
        .intrinsic_gas(),
        21_004
    );
    assert_eq!(
        Tx {
            data: vec![1],
            ..base_tx()
        }
        .intrinsic_gas(),
        21_016
    );
    assert_eq!(
        Tx {
            to: None,
            ..base_tx()
        }
        .intrinsic_gas(),
        53_000
    );
    assert_eq!(base_tx().intrinsic_gas(), 21_000);
}

#[test]
fn fee_arithmetic_saturates_instead_of_overflowing() {
    let t = Tx {
        fees: Fees::Eip1559 {
            max_priority_fee: u128::MAX,
            max_fee: 7,
            access_list: vec![],
        },
        ..base_tx()
    };
    assert_eq!(t.effective_gas_price(u128::MAX), 7);
    let t = Tx {
        fees: Fees::Eip1559 {
            max_priority_fee: 1,
            max_fee: u128::MAX,
            access_list: vec![],
        },
        value: u128::MAX,
        ..base_tx()
    };
    assert_eq!(t.max_cost(), u128::MAX);
    assert_eq!(t.effective_gas_price(u128::MAX), u128::MAX);
    let t = Tx {
        fees: Fees::Legacy {
            gas_price: u128::MAX,
        },
        value: 0,
        ..base_tx()
    };
    assert_eq!(t.max_cost(), u128::MAX);
}

#[test]
fn the_flat_token_call_gas_is_thirty_thousand() {
    assert_eq!(TOKEN_CALL_GAS, 30_000);
    assert_eq!(TRANSFER_SIG, "transfer(address,uint256)");
    assert_eq!(BALANCE_OF_SIG, "balanceOf(address)");
}

fn eth_tx(from: u8, nonce: u64, to: [u8; 20], value: u128) -> SignedTx {
    Tx {
        nonce,
        to: Some(to),
        value,
        ..base_tx()
    }
    .sign(&key(from))
}

#[test]
fn pending_nonces_are_per_sender() {
    let mut n = Node::new(
        1337,
        10 * GWEI,
        &[(addr_of_seed(1), ETH), (addr_of_seed(2), ETH)],
        &[],
    );
    n.submit(&eth_tx(1, 0, addr_of_seed(3), 1).raw()).unwrap();
    n.submit(&eth_tx(1, 1, addr_of_seed(3), 1).raw()).unwrap();
    assert_eq!(n.pending_nonce(&addr_of_seed(1)), 2);
    assert_eq!(n.pending_nonce(&addr_of_seed(2)), 0);
    n.submit(&eth_tx(2, 0, addr_of_seed(3), 1).raw()).unwrap();
    assert_eq!(n.pending_nonce(&addr_of_seed(2)), 1);
    assert_eq!(n.pending_nonce(&addr_of_seed(1)), 2);
}

#[test]
fn a_transaction_whose_worst_case_cost_equals_the_balance_is_mined() {
    let max = 21_000 * 30 * GWEI + 5;
    let mut n = Node::new(1337, 10 * GWEI, &[(addr_of_seed(1), max)], &[]);
    let t = eth_tx(1, 0, addr_of_seed(3), 5);
    let h = n.submit(&t.raw()).unwrap();
    n.mine();
    assert!(n.receipt(&h).is_some_and(|r| r.status));
    assert_eq!(n.account(&addr_of_seed(3)).balance, 5);
}

#[test]
fn a_token_holder_can_send_everything() {
    let mut n = Node::new(
        1337,
        10 * GWEI,
        &[(addr_of_seed(1), ETH)],
        &[(addr_of_seed(1), 5_000)],
    );
    let mut t = Tx {
        nonce: 0,
        to: Some(n.token),
        value: 0,
        data: transfer_calldata(addr_of_seed(4), 5_000),
        ..base_tx()
    };
    t.gas_limit = t.intrinsic_gas() + TOKEN_CALL_GAS;
    let h = n.submit(&t.sign(&key(1)).raw()).unwrap();
    n.mine();
    assert!(n.receipt(&h).unwrap().status);
    assert_eq!(n.token_balance(&addr_of_seed(1)), 0);
    assert_eq!(n.token_balance(&addr_of_seed(4)), 5_000);
    // one more than the balance reverts
    let mut t = Tx {
        nonce: 1,
        to: Some(n.token),
        value: 0,
        data: transfer_calldata(addr_of_seed(4), 1),
        ..base_tx()
    };
    t.gas_limit = t.intrinsic_gas() + TOKEN_CALL_GAS;
    let h = n.submit(&t.sign(&key(1)).raw()).unwrap();
    n.mine();
    assert!(!n.receipt(&h).unwrap().status);
}

fn req(n: &mut Node, method: &str, params: Value) -> Value {
    let r = json!({"jsonrpc":"2.0","id":1,"method":method,"params":params}).to_string();
    serde_json::from_str(&handle(n, &r)).unwrap()
}

#[test]
fn json_rpc_error_code_values() {
    assert_eq!(PARSE_ERROR, -32700);
    assert_eq!(INVALID_REQUEST, -32600);
    assert_eq!(METHOD_NOT_FOUND, -32601);
    assert_eq!(INVALID_PARAMS, -32602);
    assert_eq!(TX_REJECTED, -32000);
    assert_eq!(EXECUTION_REVERTED, 3);
}

#[test]
fn every_rejection_has_its_own_message() {
    let mut n = Node::new(1337, 10 * GWEI, &[(addr_of_seed(1), ETH)], &[]);
    let msg = |n: &mut Node, t: &Tx| -> String {
        let r = req(
            n,
            "eth_sendRawTransaction",
            json!([data_hex(&t.clone().sign(&key(1)).raw())]),
        );
        r["error"]["message"].as_str().unwrap().to_string()
    };
    let t = Tx {
        nonce: 0,
        ..base_tx()
    };
    assert_eq!(
        msg(
            &mut n,
            &Tx {
                chain_id: 1,
                ..t.clone()
            }
        ),
        "invalid chain id for signer"
    );
    assert_eq!(
        msg(
            &mut n,
            &Tx {
                gas_limit: 20_999,
                ..t.clone()
            }
        ),
        "intrinsic gas too low"
    );
    let low = Tx {
        fees: Fees::Eip1559 {
            max_priority_fee: 0,
            max_fee: GWEI,
            access_list: vec![],
        },
        ..t.clone()
    };
    assert_eq!(
        msg(&mut n, &low),
        "max fee per gas less than block base fee"
    );
    assert_eq!(
        msg(
            &mut n,
            &Tx {
                value: 2 * ETH,
                ..t.clone()
            }
        ),
        "insufficient funds for gas * price + value"
    );
    assert_eq!(
        msg(
            &mut n,
            &Tx {
                nonce: 5,
                ..t.clone()
            }
        ),
        "nonce too high"
    );
}

#[test]
fn a_failed_call_has_receipt_status_zero() {
    let mut n = Node::new(1337, 10 * GWEI, &[(addr_of_seed(1), ETH)], &[]);
    let mut t = Tx {
        nonce: 0,
        to: Some(n.token),
        value: 0,
        data: transfer_calldata(addr_of_seed(4), 1),
        ..base_tx()
    };
    t.gas_limit = t.intrinsic_gas() + TOKEN_CALL_GAS;
    let h = req(
        &mut n,
        "eth_sendRawTransaction",
        json!([data_hex(&t.sign(&key(1)).raw())]),
    )["result"]
        .clone();
    req(&mut n, "evm_mine", json!([]));
    let r = req(&mut n, "eth_getTransactionReceipt", json!([h]));
    assert_eq!(r["result"]["status"], json!("0x0"));
    assert_eq!(r["result"]["logs"], json!([]));
}

#[test]
fn quantities_with_a_plus_sign_or_non_hex_are_refused() {
    for bad in ["0x+1", "0x+", "0x-0", "0x1z", "0xé"] {
        assert_eq!(parse_quantity(bad), None, "{bad}");
    }
    assert_eq!(parse_quantity("0xAbC"), Some(0xabc));
}
```

### `tests/gap2_tests.rs`

```rust
//! Second round: tests added after the first mutation run left 24 survivors (see the lesson).
mod common;
use common::*;
use evmlab::abi::*;
use evmlab::node::*;
use evmlab::rlp::*;
use evmlab::rpc::*;
use evmlab::tx::*;
use serde_json::{Value, json};

const GWEI: u128 = 1_000_000_000;
const ETH: u128 = 1_000_000_000_000_000_000;

fn u(v: u128) -> Token {
    Token::Uint(word_from_u128(v))
}

#[test]
fn as_u128_rejects_a_leading_zero() {
    assert_eq!(Item::Bytes(vec![0, 1]).as_u128(), Err(RlpError::BadInteger));
    assert_eq!(Item::Bytes(vec![0, 1]).as_u64(), Err(RlpError::BadInteger));
    assert!(Item::Bytes(vec![1; 16]).as_u128().is_ok());
    assert_eq!(
        Item::Bytes(vec![1; 17]).as_u128(),
        Err(RlpError::BadInteger)
    );
    assert_eq!(Item::Bytes(vec![1; 9]).as_u64(), Err(RlpError::BadInteger));
}

#[test]
fn long_form_lengths_must_be_minimal() {
    // a leading zero in the length field, though the length itself is >= 56
    let mut a = vec![0xb9, 0x00, 60];
    a.extend(vec![7u8; 60]);
    assert_eq!(Item::decode(&a), Err(RlpError::NonCanonical));
    // length 55 written in long form
    let mut b = vec![0xb8, 55];
    b.extend(vec![7u8; 55]);
    assert_eq!(Item::decode(&b), Err(RlpError::NonCanonical));
    // length 56 in long form is the first legal one
    let mut c = vec![0xb8, 56];
    c.extend(vec![7u8; 56]);
    assert_eq!(Item::decode(&c), Ok(Item::Bytes(vec![7; 56])));
    let mut d = vec![0xf8, 55];
    d.extend(vec![0x80u8; 55]);
    assert_eq!(Item::decode(&d), Err(RlpError::NonCanonical));
}

fn dirty(mut w: Word, idx: usize) -> Word {
    w[idx] = 1;
    w
}

#[test]
fn every_byte_of_a_padded_word_is_checked() {
    let ok = word_from_u128(5);
    // offset word with a stray byte just above the low 64 bits
    let mut data = dirty(word_from_u128(32), 23).to_vec();
    data.extend(word_from_u128(0));
    assert!(decode(&[Ty::Bytes], &data).is_err());
    // address: bytes 0..12 must be zero
    assert_eq!(
        decode(&[Ty::Address], &dirty(ok, 11)),
        Err(AbiError::DirtyPadding)
    );
    assert!(decode(&[Ty::Address], &dirty([0; 32], 12)).is_ok());
    // bytes4: the byte right after the value must be zero
    let mut w = [0u8; 32];
    w[..4].copy_from_slice(&[1, 2, 3, 4]);
    assert!(decode(&[Ty::FixedBytes(4)], &w).is_ok());
    assert_eq!(
        decode(&[Ty::FixedBytes(4)], &dirty(w, 4)),
        Err(AbiError::DirtyPadding)
    );
    // bool: 2 is not a bool
    assert!(decode(&[Ty::Bool], &word_from_u128(2)).is_err());
    assert!(decode(&[Ty::Bool], &word_from_u128(1)).is_ok());
}

#[test]
fn type_names_are_parsed_strictly() {
    for bad in [
        "uint12", "uint0", "uint264", "int7", "bytes01", "bytes0", "bytes33", "uint08",
    ] {
        assert!(parse_type(bad).is_err(), "{bad}");
    }
    assert_eq!(parse_type("uint8"), Ok(Ty::Uint(8)));
    assert_eq!(parse_type("uint256"), Ok(Ty::Uint(256)));
    assert_eq!(parse_type("bytes32"), Ok(Ty::FixedBytes(32)));
    assert_eq!(parse_type("bytes1"), Ok(Ty::FixedBytes(1)));
}

#[test]
fn static_composites_take_the_right_head_space() {
    let pair = Ty::Tuple(vec![Ty::Uint(256), Ty::Uint(256)]);
    let nested = Ty::Tuple(vec![Ty::Uint(256), pair.clone()]);
    assert_eq!(nested.head_size(), 96);
    let grid = Ty::FixedArray(Box::new(pair.clone()), 2);
    assert_eq!(grid.head_size(), 128);
    // a static aggregate followed by another argument decodes in place
    let tys = [nested.clone(), grid.clone(), Ty::Uint(256)];
    let toks = vec![
        Token::Tuple(vec![u(1), Token::Tuple(vec![u(2), u(3)])]),
        Token::Array(vec![
            Token::Tuple(vec![u(4), u(5)]),
            Token::Tuple(vec![u(6), u(7)]),
        ]),
        u(8),
    ];
    let enc = encode(&tys, &toks).unwrap();
    assert_eq!(enc.len(), 96 + 128 + 32);
    assert_eq!(decode_strict(&tys, &enc), Ok(toks));
}

#[test]
fn length_and_offset_errors_are_the_exact_ones() {
    // an offset equal to the data length passes the bound check and then finds nothing to read
    let data = word_from_u128(32).to_vec();
    assert_eq!(decode(&[Ty::Bytes], &data), Err(AbiError::Eof));
    // an array claiming 32 elements with only 31 words behind its length
    let mut d = word_from_u128(32).to_vec();
    d.extend(word_from_u128(32));
    for _ in 0..31 {
        d.extend(word_from_u128(1));
    }
    assert_eq!(
        decode(&[Ty::Array(Box::new(Ty::Uint(256)))], &d),
        Err(AbiError::TooLong)
    );
}

#[test]
fn encoders_refuse_wrong_lengths() {
    let ty = Ty::FixedArray(Box::new(Ty::Uint(256)), 2);
    let three = Token::Array(vec![u(1), u(2), u(3)]);
    assert_eq!(
        encode(std::slice::from_ref(&ty), &[three]),
        Err(AbiError::Mismatch)
    );
    assert_eq!(
        encode(&[Ty::Uint(256)], &[u(1), u(2)]),
        Err(AbiError::Mismatch)
    );
    assert_eq!(
        encode(&[Ty::Uint(256), Ty::Bool], &[u(1)]),
        Err(AbiError::Mismatch)
    );
}

fn tx(chain: u64, nonce: u64) -> Tx {
    Tx {
        chain_id: chain,
        nonce,
        gas_limit: 21_000,
        to: Some([0x22; 20]),
        value: 1,
        data: vec![],
        fees: Fees::Legacy { gas_price: GWEI },
    }
}

#[test]
fn chain_zero_legacy_v_35_is_accepted() {
    let mut seen = [false; 2];
    for s in 1..60u8 {
        let signed = tx(0, 0).sign(&key(s));
        let v = Item::decode(&signed.raw()).unwrap().as_list().unwrap()[6]
            .as_u64()
            .unwrap();
        seen[(v - 35) as usize] = true;
        assert_eq!(SignedTx::from_raw(&signed.raw()), Ok(signed));
    }
    assert_eq!(seen, [true, true]);
}

#[test]
fn a_padded_signature_integer_is_not_canonical() {
    let signed = tx(1, 0).sign(&key(5));
    let mut f = Item::decode(&signed.raw())
        .unwrap()
        .as_list()
        .unwrap()
        .to_vec();
    let padded = vec![0u8, 5];
    f[7] = Item::Bytes(padded);
    assert_eq!(
        SignedTx::from_raw(&Item::List(f).encode()),
        Err(TxError::BadField)
    );
}

fn fresh() -> Node {
    Node::new(1337, 10 * GWEI, &[(addr_of_seed(1), ETH)], &[])
}

#[test]
fn an_unused_gas_allowance_is_not_charged() {
    let mut n = fresh();
    let t = Tx {
        chain_id: 1337,
        gas_limit: 100_000,
        fees: Fees::Legacy {
            gas_price: 20 * GWEI,
        },
        ..tx(1337, 0)
    };
    let h = n.submit(&t.sign(&key(1)).raw()).unwrap();
    n.mine();
    let r = n.receipt(&h).unwrap();
    assert_eq!(r.gas_used, 21_000);
    assert_eq!(
        n.account(&addr_of_seed(1)).balance,
        ETH - 21_000 * 20 * GWEI - 1
    );
    assert_eq!(n.burned(), 21_000 * 10 * GWEI);
    assert_eq!(n.total_ether(), n.initial_ether());
}

#[test]
fn a_dropped_transaction_drops_its_successors() {
    let mut n = Node::new(1337, 10 * GWEI, &[(addr_of_seed(1), 10 * ETH)], &[]);
    let cap = 20 * GWEI;
    let mk = |nonce: u64, value: u128| {
        let mut t = Tx {
            value,
            ..tx(1337, nonce)
        };
        t.fees = Fees::Legacy { gas_price: cap };
        t.sign(&key(1))
    };
    let fee = 21_000 * cap;
    // 0 leaves a little; 1 asked for a lot (fine when submitted); 2 is cheap
    let a = mk(0, 10 * ETH - fee - 21_000 * cap - 5);
    let b = mk(1, 9 * ETH);
    let c = mk(2, 1);
    n.submit(&a.raw()).unwrap();
    n.submit(&b.raw()).unwrap();
    n.submit(&c.raw()).unwrap();
    n.mine();
    assert!(n.receipt(&a.hash()).is_some());
    assert!(n.receipt(&b.hash()).is_none());
    assert!(
        n.receipt(&c.hash()).is_none(),
        "nonce 2 cannot follow a missing nonce 1"
    );
    assert_eq!(n.account(&addr_of_seed(1)).nonce, 1);
}

#[test]
fn earliest_means_block_zero_for_logs() {
    let mut n = Node::new(
        1337,
        10 * GWEI,
        &[(addr_of_seed(1), ETH)],
        &[(addr_of_seed(1), 100)],
    );
    let mut t = Tx {
        to: Some(n.token),
        value: 0,
        data: transfer_calldata(addr_of_seed(2), 7),
        ..tx(1337, 0)
    };
    t.gas_limit = t.intrinsic_gas() + TOKEN_CALL_GAS;
    t.fees = Fees::Legacy {
        gas_price: 20 * GWEI,
    };
    n.submit(&t.sign(&key(1)).raw()).unwrap();
    n.mine();
    let q = |n: &mut Node, from: &str| {
        let r = json!({"jsonrpc":"2.0","id":1,"method":"eth_getLogs",
            "params":[{"fromBlock": from, "toBlock": "latest"}]})
        .to_string();
        let v: Value = serde_json::from_str(&handle(n, &r)).unwrap();
        v["result"].as_array().unwrap().len()
    };
    assert_eq!(q(&mut n, "earliest"), 1);
    assert_eq!(q(&mut n, "0x1"), 1);
    assert_eq!(q(&mut n, "0x2"), 0);
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
use evmlab::abi::{self, Token, Ty, word_from_u128};
use evmlab::address::{address_of, signing_key, to_checksum};
use evmlab::keccak::keccak256;
use evmlab::node::{Node, TOKEN_CALL_GAS, transfer_calldata};
use evmlab::rlp::Item;
use evmlab::tx::{Fees, SignedTx, Tx};
use std::hint::black_box;
use std::time::Instant;

fn time<F: FnMut()>(name: &str, iters: u32, mut f: F) {
    for _ in 0..(iters / 10).max(1) {
        f();
    }
    let t = Instant::now();
    for _ in 0..iters {
        f();
    }
    let ns = t.elapsed().as_nanos() as f64 / f64::from(iters);
    println!("{name:<38} {ns:>12.0} ns/op");
}

fn main() {
    let key = signing_key(&[7; 32]).unwrap();
    let tx = Tx {
        chain_id: 1337,
        nonce: 3,
        gas_limit: 21_000,
        to: Some([0x22; 20]),
        value: 12345,
        data: vec![],
        fees: Fees::Eip1559 {
            max_priority_fee: 2_000_000_000,
            max_fee: 30_000_000_000,
            access_list: vec![],
        },
    };
    let signed = tx.clone().sign(&key);
    let raw = signed.raw();
    let kb = vec![0xabu8; 1024];
    let data = transfer_calldata([5; 20], 1000);
    let tys = [Ty::Address, Ty::Uint(256)];
    time("keccak256 of 32 bytes", 1_000_000, || {
        black_box(keccak256(black_box(&[1u8; 32])));
    });
    time("keccak256 of 1 KiB", 200_000, || {
        black_box(keccak256(black_box(&kb)));
    });
    time("rlp decode of a type-2 tx body", 500_000, || {
        black_box(Item::decode(black_box(&raw[1..])).unwrap());
    });
    time("abi encode transfer(address,uint256)", 500_000, || {
        black_box(
            abi::encode_call(
                "transfer(address,uint256)",
                &[Token::Address([5; 20]), Token::Uint(word_from_u128(1000))],
            )
            .unwrap(),
        );
    });
    time("abi decode_strict of the arguments", 500_000, || {
        black_box(abi::decode_strict(&tys, black_box(&data[4..])).unwrap());
    });
    time("eip-55 checksum", 200_000, || {
        black_box(to_checksum(black_box(&[9; 20])));
    });
    time("tx sign (secp256k1, RFC 6979)", 2_000, || {
        black_box(tx.clone().sign(&key));
    });
    time("tx parse + recover sender", 2_000, || {
        let s = SignedTx::from_raw(black_box(&raw)).unwrap();
        black_box(s.sender().unwrap());
    });
    let keys: Vec<_> = (1..=50u8).map(|s| signing_key(&[s; 32]).unwrap()).collect();
    let funded: Vec<_> = keys
        .iter()
        .map(|k| (address_of(k.verifying_key()), 10u128.pow(20)))
        .collect();
    time("block of 50 token transfers", 20, || {
        let mut n = Node::new(1337, 10_000_000_000, &funded, &funded);
        for (i, k) in keys.iter().enumerate() {
            let mut t = Tx {
                chain_id: 1337,
                nonce: 0,
                gas_limit: 0,
                to: Some(n.token),
                value: 0,
                data: transfer_calldata(funded[(i + 1) % 50].0, 10),
                fees: Fees::Eip1559 {
                    max_priority_fee: 1_000_000_000,
                    max_fee: 20_000_000_000,
                    access_list: vec![],
                },
            };
            t.gas_limit = t.intrinsic_gas() + TOKEN_CALL_GAS;
            n.submit(&t.sign(k).raw()).unwrap();
        }
        black_box(n.mine());
    });
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **93 tests** (keccak and RLP 9, ABI 14, address 6, transaction 14, node 15, RPC 8, gap tests 15 and 12) and 8 consecutive full runs without a failure (nothing uses sockets, real time or OS randomness; the random tests use a fixed-seed generator).

**Mutation check ✓.** I made 114 small source edits (a "mutant") across seven of the eight source modules, each in its own copy of the crate, and ran the whole test suite against each. **107 were caught by a failing test and 7 are equivalent (M29, M50, M71, M72, M97, M101, M111); none fails to compile and none survives unexplained.** **What the first pass found.** While planning the mutants I found a real bug (§9.2: `parse_quantity` accepted `0x+1`) and removed one dead check (a length-of-length test in the RLP reader that the first byte already guarantees); both were fixed before the first run, and the second needed no test. I then wrote 15 tests (in `gap_tests.rs`) before the first run, to pin things I could already see were loose: the nesting limit, zero-argument calls, the half-order boundary, exact field counts, chain id 0, access-list gas, saturating fee arithmetic, the token gas constant, per-sender pending nonces, the exact-funds boundary, a full-balance token transfer, the literal error codes and texts, and the `0x+1` family. Against the 81 tests then in place, 90 of the 114 mutants were caught and 24 survived. Seven are equivalent (below). **Seventeen were real gaps**, closed by new tests in `gap2_tests.rs` (12 tests were added; two of them aimed at mutants that proved equivalent): **the leading-zero rule for 128-bit integers**, **a zero byte in a long-form length**, **a length of exactly 55 in long form**, **a stray byte above the low 64 bits of an offset**, **the head size of nested static tuples and fixed arrays** (three mutants, because the only static aggregates tested held plain words), **type-name parsing** (`uint12`, `bytes01`), **the position of the first dirty padding byte** for addresses and `bytesN`, **the array-count guard** (the error must be `TooLong`, not `Eof`), **an offset equal to the data length**, **a stale nonce after an earlier transaction of the same account was dropped**, **fees charged on `gas_used` rather than `gas_limit`** (every test had used a limit equal to the usage) and **`v` = 35**, the smallest protected value, which only appears for chain id 0 and one parity of the signature. One of my own new tests failed to kill its mutant: the test for a padded signature integer padded a *32-byte* value to 33 bytes, which the length check refuses anyway; it killed the mutant only after I changed it to a short value with a leading zero (`00 05`). Another had a wrong expectation (I wrote `BadOffset` where the code reports `Eof`, since an offset equal to the length passes the bound and then finds nothing to read), which running it showed. Two of the equivalent mutants are redundant checks that I left in place on purpose. The column "change" is generated from the edit itself (the part that differs, with a little context, whitespace collapsed); `…` marks omitted text. The tests ran in the dev profile with optimized dependencies.

| # | module | change | result |
|---|---|---|---|
| M1 | `rlp` | `if len <` → `if len <=` | caught by `rlp_length_boundaries` |
| M2 | `rlp` | `out.push(short + 55` → `out.push(short + 54` | caught by `a_failed_call_has_receipt_status_zero` |
| M3 | `rlp` | `…== 1 && b[0] <` → `…== 1 && b[0] <=` | caught by `rlp_spec_examples` |
| M4 | `rlp` | `…MAX_DEPTH: usize = 32` → `…MAX_DEPTH: usize = 33` | caught by `nesting_limit_is_exactly_32_levels_below_the_top` |
| M5 | `rlp` | `if depth >` → `if depth >=` | caught by `nesting_limit_is_exactly_32_levels_below_the_top` |
| M6 | `rlp` | `if input.is_empty()` → `if true` | caught by `rlp_hostile_and_malformed_input` |
| M7 | `rlp` | `if b.len() > 8` → `if b.len() > 9` | caught by `rlp_integers_are_canonical` |
| M8 | `rlp` | `if b.len() > 8 \|\| b.first() == Some(&0)` → `if b.len() > 8` | caught by `rlp_integers_are_canonical` |
| M9 | `rlp` | `if b.len() > 16` → `if b.len() > 17` | caught by `rlp_integers_are_canonical` |
| M10 | `rlp` | `…b.len() > 16 \|\| b.first() == Some(&0)` → `…b.len() > 16` | caught by `as_u128_rejects_a_leading_zero` |
| M11 | `rlp` | `…== 0).count()` → `…== 0).count().min(15)` | caught by `a_token_holder_can_send_everything` |
| M12 | `rlp` | `if b[0] == 0` → `if false` | caught by `long_form_lengths_must_be_minimal` |
| M13 | `rlp` | `if len < 56` → `if len < 55` | caught by `long_form_lengths_must_be_minimal` |
| M14 | `rlp` | `…&& b[0] < 0x80` → `…&& b[0] < 0x7f` | caught by `rlp_refuses_non_canonical_forms` |
| M15 | `rlp` | `if len == 1 && b[0] < 0x80` → `if false` | caught by `rlp_refuses_non_canonical_forms` |
| M16 | `rlp` | `…usize::from(first - 0x80` → `…usize::from(first - 0x81` | caught by `a_token_holder_can_send_everything` |
| M17 | `rlp` | `…usize::from(first - 0xc0` → `…usize::from(first - 0xc1` | caught by `a_token_holder_can_send_everything` |
| M18 | `rlp` | `if len > input.len()` → `if false` | caught by `rlp_hostile_and_malformed_input` |
| M19 | `rlp` | `…&mut body, depth + 1` → `…&mut body, depth` | caught by `nesting_limit_is_exactly_32_levels_below_the_top` |
| M20 | `rlp` | `…input.len() <` → `…input.len() <=` | caught by `a_token_holder_can_send_everything` |
| M21 | `rlp` | `…usize::from(first - 0xb7` → `…usize::from(first - 0xb6` | caught by `a_token_holder_can_send_everything` |
| M22 | `abi` | `if w[..16` → `if w[..15` | caught by `word_helpers` |
| M23 | `abi` | `if w[..24` → `if w[..23` | caught by `every_byte_of_a_padded_word_is_checked` |
| M24 | `abi` | `Ty::FixedArray(t, n) => *n > 0 && ` → `Ty::FixedArray(t, n) => ` | caught by `dynamic_and_nested_layouts` |
| M25 | `abi` | `…=> ts.iter().any` → `…=> ts.iter().all` | caught by `dynamic_and_nested_layouts` |
| M26 | `abi` | `Ty::FixedArray(t, n) => n * t.head_size()` → `Ty::FixedArray(_, n) => n * 32` | caught by `static_composites_take_the_right_head_space` |
| M27 | `abi` | `Ty::Tuple(ts) => ts.iter().map(Ty::head_size).sum()` → `Ty::Tuple(ts) => ts.len() * 32` | caught by `static_composites_take_the_right_head_space` |
| M28 | `abi` | `…!n.is_multiple_of(8` → `…!n.is_multiple_of(4` | caught by `type_names_are_parsed_strictly` |
| M29 | `abi` | `if n == 0 \|\| n ` → `if n ` | **equivalent**: `uint0` is already refused one line earlier by the leading-zero check, so the `n == 0` test is unreachable; a dead check, kept as a second line of defence |
| M30 | `abi` | `…n == 0 \|\| n >` → `…n == 0 \|\| n >=` | caught by `canonical_names_round_trip_through_the_parser` |
| M31 | `abi` | `if (1..=` → `if (1..` | caught by `signature_parsing` |
| M32 | `abi` | `…1..=32).contains(&n) && !r.starts_with('0')` → `…1..=32).contains(&n)` | caught by `type_names_are_parsed_strictly` |
| M33 | `abi` | `if rem != 0` → `if rem != 1` | caught by `dynamic_and_nested_layouts` |
| M34 | `abi` | `…std::iter::repeat_n(0u8, 32 - ` → `…std::iter::repeat_n(0u8, ` | caught by `decoding_refuses_dirty_padding` |
| M35 | `abi` | `…/ 8; if w[..32` → `…/ 8; if w[..31` | caught by `decode_call_checks_the_selector` |
| M36 | `abi` | `let ext = if w[32 - keep] & 0x80 != 0 { 0xff } else { 0x00 }` → `let ext = 0x00` | caught by `static_types_are_one_word_each` |
| M37 | `abi` | `if w[..12` → `if w[..11` | caught by `every_byte_of_a_padded_word_is_checked` |
| M38 | `abi` | `…0) \|\| w[31] > 1` → `…0) \|\| w[31] > 2` | caught by `decoding_refuses_dirty_padding` |
| M39 | `abi` | `…\|b\| *b != 0) \|\| w[31] > 1 ` → `…\|b\| *b != 0) ` | caught by `decoding_refuses_dirty_padding` |
| M40 | `abi` | `if w[*n` → `if w[*n + 1` | caught by `every_byte_of_a_padded_word_is_checked` |
| M41 | `abi` | `if len >` → `if len >=` | caught by `dynamic_and_nested_layouts` |
| M42 | `abi` | `…len.div_ceil(32) * 32` → `…len.div_ceil(32) * 32 + 32` | caught by `decoding_refuses_dirty_padding` |
| M43 | `abi` | `if body[len` → `if body[len + 1` | caught by `dynamic_and_nested_layouts` |
| M44 | `abi` | `…data.len() - 32) / 32` → `…data.len() - 32) / 31` | caught by `length_and_offset_errors_are_the_exact_ones` |
| M45 | `abi` | `if off >` → `if off >=` | caught by `length_and_offset_errors_are_the_exact_ones` |
| M46 | `abi` | `…&data[off..])?); pos += 32` → `…&data[off..])?); pos += 31` | caught by `encoding_agrees_with_alloy_on_random_values` |
| M47 | `abi` | `pos += ty.head_size()` → `pos += 32` | caught by `static_composites_take_the_right_head_space` |
| M48 | `abi` | `…calldata.len() < 4` → `…calldata.len() < 5` | caught by `zero_argument_calls_and_zero_words` |
| M49 | `abi` | `word_from_u128((head_len + tail.len())` → `word_from_u128(head_len` | caught by `dynamic_and_nested_layouts` |
| M50 | `abi` | `…items.len() !=` → `…items.len() <` | **equivalent**: with too many items the next call, `encode_seq`, compares the type count with the item count and refuses with the same error; the length check is duplicated on purpose |
| M51 | `abi` | `if tys.len() !=` → `if tys.len() <` | caught by `values_must_fit_their_types` |
| M52 | `address` | `if nibble >=` → `if nibble >` | caught by `checksum_matches_alloy_on_random_addresses` |
| M53 | `address` | `h[i / 2] >> 4` → `h[i / 2] & 0x0f` | caught by `checksum_matches_alloy_on_random_addresses` |
| M54 | `address` | `if body.len() !=` → `if body.len() <` | caught by `parse_rejects_bad_text` |
| M55 | `address` | `…has_lower && has_upper && ` → `…has_lower && ` | caught by `eip55_known_answers` |
| M56 | `address` | `…to_checksum(&a)[2..] !` → `…to_checksum(&a)[2..] =` | caught by `a_single_wrong_capital_is_caught` |
| M57 | `tx` | `…mut gas = 21_000` → `…mut gas = 21_001` | caught by `a_transaction_whose_worst_case_cost_equals_the_balance_is_mined` |
| M58 | `tx` | `…{ 4 } else { 16` → `…{ 4 } else { 68` | caught by `access_list_gas_counts_every_key` |
| M59 | `tx` | `…if *b == 0 { 4 } else { 16` → `…if *b == 0 { 16 } else { 4` | caught by `access_list_gas_counts_every_key` |
| M60 | `tx` | `gas += 32_0` → `gas += 3_2` | caught by `access_list_gas_counts_every_key` |
| M61 | `tx` | `…2_400 + 1_900 * keys.len() as u64` → `…2_400 + 1_900` | caught by `access_list_gas_counts_every_key` |
| M62 | `tx` | `gas += 2_400 + ` → `gas += ` | caught by `access_list_gas_counts_every_key` |
| M63 | `tx` | `…self.sig.y_parity) + 35` → `…self.sig.y_parity) + 27` | caught by `chain_id_zero_is_the_smallest_protected_legacy_v` |
| M64 | `tx` | `…self.sig.y_parity) + 35 + 2 * ` → `…self.sig.y_parity) + 35 + ` | caught by `a_legacy_transaction_pays_its_gas_price` |
| M65 | `tx` | `if v < 35` → `if v < 36` | caught by `chain_zero_legacy_v_35_is_accepted` |
| M66 | `tx` | `…u64::try_from((v - 35) / 2` → `…u64::try_from((v - 35) / 3` | caught by `a_legacy_transaction_pays_its_gas_price` |
| M67 | `tx` | `…((v - 35) % 2` → `…((v - 35) % 3` | caught by `a_legacy_transaction_pays_its_gas_price` |
| M68 | `tx` | `if y > 1` → `if y > 2` | caught by `parsing_refuses_unprotected_and_invalid_v` |
| M69 | `tx` | `if s >` → `if s >=` | caught by `the_s_boundary_is_exact` |
| M70 | `tx` | `if s > HALF_ORDER` → `if false` | caught by `the_s_boundary_is_exact` |
| M71 | `tx` | `if r == [0; 32] \|\| ` → `if ` | **equivalent**: a zero `r` is also refused by `sender()`, because the library's `Signature::from_scalars` rejects zero, and it maps to the same error (`ZeroSignature`); the explicit check in `finish` is redundant but gives the error before the recovery cost |
| M72 | `tx` | `…r == [0; 32] \|\| s == [0; 32] ` → `…r == [0; 32] ` | **equivalent**: the same reason as the previous row, for `s` |
| M73 | `tx` | `if f.len() !=` → `if f.len() <` | caught by `field_counts_are_exact` |
| M74 | `tx` | `if f.len() !=` → `if f.len() <` | caught by `field_counts_are_exact` |
| M75 | `tx` | `if e.len() !=` → `if e.len() <` | caught by `field_counts_are_exact` |
| M76 | `tx` | `Some(&b) if b >=` → `Some(&b) if b >` | caught by `parsing_refuses_wrong_shapes` |
| M77 | `tx` | `…(*max_fee).min` → `…(*max_fee).max` | caught by `fee_arithmetic_saturates_instead_of_overflowing` |
| M78 | `tx` | `…*max_fee).min(base_fee.saturating_add(*max_priority_fee)` → `…*max_fee).min(base_fee` | caught by `a_plain_transfer_pays_gas_at_the_effective_price_and_splits_it_into_burn_and_tip` |
| M79 | `tx` | `.saturat` → `.wrapp` | caught by `fee_arithmetic_saturates_instead_of_overflowing` |
| M80 | `tx` | `.saturat` → `.wrapp` | caught by `fee_arithmetic_saturates_instead_of_overflowing` |
| M81 | `tx` | `y_parity: rec.to_byte() & 1` → `y_parity: 0` | caught by `a_token_holder_can_send_everything` |
| M82 | `tx` | `…b.len() > 32 \|\| b.first() == Some(&0) ` → `…b.len() > 32 ` | caught by `a_padded_signature_integer_is_not_canonical` |
| M83 | `tx` | `if b.len() > 32` → `if b.len() > 33` | caught by `parsing_refuses_weak_or_malformed_signatures` |
| M84 | `node` | `…stx.tx.nonce <` → `…stx.tx.nonce <=` | caught by `a_token_holder_can_send_everything` |
| M85 | `node` | `…stx.tx.nonce >` → `…stx.tx.nonce >=` | caught by `a_token_holder_can_send_everything` |
| M86 | `node` | `…stx.tx.gas_limit <` → `…stx.tx.gas_limit <=` | caught by `a_transaction_whose_worst_case_cost_equals_the_balance_is_mined` |
| M87 | `node` | `…stx.tx.fee_cap() <` → `…stx.tx.fee_cap() <=` | caught by `the_pool_rejects_for_a_named_reason` |
| M88 | `node` | `…&from).balance <` → `…&from).balance <=` | caught by `a_transaction_whose_worst_case_cost_equals_the_balance_is_mined` |
| M89 | `node` | `…&from).balance < stx.tx.max_cost()` → `…&from).balance < stx.tx.value` | caught by `funds_are_checked_against_the_worst_case_not_the_expected_cost` |
| M90 | `node` | `if stx.tx.chain_id != self.chain_id` → `if false` | caught by `every_rejection_has_its_own_message` |
| M91 | `node` | `if self.pool.iter().any(\|(t, _)\| t.hash() == hash)` → `if false` | caught by `the_pool_rejects_for_a_named_reason` |
| M92 | `node` | `…+ self.pool.iter().filter(\|(_, s)\| s == a).count` → `…+ self.pool.len` | caught by `pending_nonces_are_per_sender` |
| M93 | `node` | `if acct.nonce != t.nonce \|\| acct.` → `if acct.` | caught by `a_dropped_transaction_drops_its_successors` |
| M94 | `node` | `…acct.nonce != t.nonce \|\| acct.balance < t.max_cost() ` → `…acct.nonce != t.nonce ` | caught by `a_transaction_that_became_unaffordable_is_dropped_at_mining` |
| M95 | `node` | `if gas_used >` → `if gas_used >=` | caught by `a_token_holder_can_send_everything` |
| M96 | `node` | `gas_used += TOKEN_CALL_GAS` → `gas_used += 0` | caught by `a_reverted_call_still_costs_gas_and_a_nonce_and_leaves_no_log` |
| M97 | `node` | `…self.base_fee.min(price)` → `…self.base_fee` | **equivalent**: `submit` refuses a fee cap below the base fee and the base fee never changes, so the price is never below it and `min(base_fee, price)` is always `base_fee`; the `min` would matter the day the base fee moved between submission and mining |
| M98 | `node` | `let burn = u128::from(gas_used) * self.base_fee.min(price)` → `let burn = 0` | caught by `a_legacy_transaction_pays_its_gas_price` |
| M99 | `node` | `let tip = fee - burn` → `let tip = 0` | caught by `a_legacy_transaction_pays_its_gas_price` |
| M100 | `node` | `a.nonce += 1` → `a.nonce += 0` | caught by `a_token_holder_can_send_everything` |
| M101 | `node` | `if let (true` → `if let (_` | **equivalent**: `moves` is only set for a plain transfer, whose status is still `true` at that point; the status test guards a case that cannot happen yet (a plain transfer cannot fail in this model) |
| M102 | `node` | `if bal <` → `if bal <=` | caught by `a_token_holder_can_send_everything` |
| M103 | `node` | `if b.number <` → `if b.number <=` | caught by `logs_are_filtered_by_range_emitter_and_topics` |
| M104 | `node` | `…\|\| b.number >` → `…\|\| b.number >=` | caught by `logs_are_filtered_by_range_emitter_and_topics` |
| M105 | `node` | `…l.topics.get(i).is_some_and` → `…l.topics.get(i).is_none_or` | caught by `logs_are_filtered_by_range_emitter_and_topics` |
| M106 | `node` | `set.is_empty() \|\| ` → `∅` | caught by `logs_are_filtered_by_range_emitter_and_topics` |
| M107 | `node` | `…address.is_some_and(\|a\| *a !` → `…address.is_some_and(\|a\| *a =` | caught by `logs_are_filtered_by_range_emitter_and_topics` |
| M108 | `node` | `…= u128::from(gas_used` → `…= u128::from(t.gas_limit` | caught by `an_unused_gas_allowance_is_not_charged` |
| M109 | `rpc` | `…b.is_ascii_hexdigit()) \|\| (digits.len() > 1 && digits.starts_wit…` → `…b.is_ascii_hexdigit()) ` | caught by `protocol_and_parameter_errors` |
| M110 | `rpc` | `if !digits.bytes().all(\|b\| b.is_ascii_hexdigit()) \|\| (digits.le…` → `if digits.len() > 1 && digits.starts_with('0'` | caught by `quantities_with_a_plus_sign_or_non_hex_are_refused` |
| M111 | `rpc` | `"earliest" => Some(0` → `"earliest" => Some(1` | **equivalent**: block 0 has no logs, so starting `eth_getLogs` at block 0 or at block 1 returns the same entries; `earliest` would matter for any method that reads state at a block |
| M112 | `rpc` | `…"pending") => node.pending_nonce(&a)` → `…"pending") => node.account(&a).nonce` | caught by `send_mine_receipt_and_nonce_views` |
| M113 | `rpc` | `…Value::Null, PARSE_ERROR` → `…Value::Null, INVALID_REQUEST` | caught by `protocol_and_parameter_errors` |
| M114 | `logs` | `topics.len() !=` → `topics.len() <` | caught by `transfer_logs_decode_strictly` |

What the table does not show: it tests *these* edits, not every possible bug; the equivalent mutants could hide a real gap if my reasoning about them is wrong; the `keccak` module has no mutants because it is three one-line wrappers whose outputs are pinned by Python known answers; and passing every mutant says nothing about the real EVM, about security or about behavior on a node.

**Design notes.**

- Everything that touches bytes returns an error enum; nothing in the parsers panics on input (✓ random garbage tests).
- The sender is **recovered** and cached nowhere: `sender()` is called by `from_raw` (so a parsed transaction is known to have a recoverable signature) and by `submit`; the cost is visible in §10.
- `submit` validates for the pool and `execute` validates again at mining; the second check is not redundant (§8.3).
- Values are `u128`; a real client needs `u256` (alloy's `U256`, from memory).
- Not handled: the EVM, contract creation, contracts of any kind, other transaction types (type 1 access-list transactions, type 3 blob transactions, type 4 authorization transactions: names from memory), the real gas schedule, a trie, reorganizations, a mempool with replacement and eviction, EIP-712, WebSocket subscriptions, networking.

</details>

---

## 18. Key takeaways

- An EVM client's job starts with exact bytes: Keccak (not SHA3), canonical signatures for selectors, RLP and ABI decoders that accept one encoding per value, refuse hostile lengths and are checked against a second implementation (✓ `alloy-*` as oracles, Python for the fixed vectors).
- An address is 20 bytes; EIP-55 is a typo detector in mixed-case spellings only; compare bytes, not text (✓ every single-letter flip caught).
- A transaction's sender is recovered from the signature; the chain id, the fee fields and every other field are signed; low-s and canonical integers keep the id unique (✓ Python-built wire bytes; every bit flip).
- A node checks funds against the worst case and charges for gas used; a mined transaction can still have failed, so read the receipt (✓ the burn/tip split, reverted calls, conservation over 40 random rounds).
- Logs are the indexer's input: filter by AND-of-ORs, decode by topic count and data size, and keep amounts wide (✓ ERC-721 versus ERC-20, `u128` limit stated).
- A JSON-RPC front end is a small contract: minimal hex quantities, pinned error codes, a distinct text per rejection; wrap the standard library's permissive parsers (✓ the `0x+1` bug).
- Signatures dominate cost; hashes and codecs are cheap (✓ about 120 µs to recover a sender against about 300 ns for a hash).
- Everything about the real EVM, real nodes, alloy's providers, ethers-rs, revm, reth and Foundry is from memory here; the model teaches the byte-level and rule-level contract, not an execution engine.
- Mutation testing here pointed at real gaps: 17 of the 24 first survivors were real gaps and were closed with new tests (integer and length canonical forms, padding positions, head sizes of static aggregates, type-name parsing, a stale nonce, fees on gas used rather than the limit, `v` = 35), and planning the mutants had already exposed the `0x+1` parser bug; 7 mutants remain, each with a stated reason for being equivalent.

**Connections.** [Lesson 48](48_crypto_primitives.md) supplied the hashes and signatures; [Lesson 49](49_merkle_transactions_consensus.md) the transaction vocabulary that §7 makes exact; [Lesson 51](51_rpc_nodes_indexers.md) and [Lesson 52](52_web3_reliability.md) are the client habits (finality, retries, idempotency, nonces) that §8 and §9 repeat; [Lesson 55](55_solana_security_testing_deploy.md) and [Lesson 56](56_substrate_polkadot.md) ask the same questions of two other chains; [Lesson 39](39_testing.md) and [Lesson 40](40_property_testing_fuzzing.md) are the testing habits behind the oracles and the random tests. Lesson 58 turns from one chain to the infrastructure that connects them: cross-chain messaging, relayers and indexers.

---

## 19. References

- Ethereum documentation (accounts, transactions, gas, the JSON-RPC API): <https://ethereum.org/en/developers/docs/> and the execution API specification, which I remember as the repository `ethereum/execution-apis`.
- Ethereum Improvement Proposals: <https://eips.ethereum.org/> — EIP-55 (checksum addresses), EIP-155 (replay protection), EIP-1559 (fee market), EIP-2 (low-s, in the Homestead changes), EIP-2718 (typed transactions), EIP-2930 (access lists), EIP-712 (typed data).
- The Solidity documentation, "Contract ABI Specification": <https://docs.soliditylang.org/> (the `f(uint256,uint32[],bytes10,bytes)` example is from there).
- The RLP page in the Ethereum developer documentation (under data structures and encoding).
- `alloy`: <https://github.com/alloy-rs/alloy> and <https://alloy.rs>; `revm`: <https://github.com/bluealloy/revm>; `reth`: <https://github.com/paradigmxyz/reth>; Foundry: <https://book.getfoundry.sh/>
- `k256` and `sha3` (RustCrypto): <https://docs.rs/k256>, <https://docs.rs/sha3>; RFC 6979 (deterministic ECDSA): <https://www.rfc-editor.org/rfc/rfc6979>
- JSON-RPC 2.0 specification: <https://www.jsonrpc.org/specification>

The URLs and titles above are the standard locations as I remember them; I did not open them during this session (the sandbox cannot reach documentation sites), so if a link has moved, search for the title. **Unchecked or from memory:** everything about the **real** stack: the EVM's execution model and opcode costs, the real gas schedule (the intrinsic-gas constants are from memory of the protocol and the 16-gas figure for non-zero bytes), EIP numbers and field orders (they agree with Python and `alloy-rlp` on the examples tested, which is not a check of the EIPs themselves), geth's behavior on quantities, error texts and codes of real nodes, mempool behavior and front-running, ERC-20 and ERC-721 details, `alloy`, `ethers-rs`, `revm`, `reth` and Foundry, layer-2 differences, and typed-data signing. Everything marked ✓ was run in this session, against the crate in this lesson.
