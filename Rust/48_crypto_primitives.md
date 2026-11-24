---
layout: default
title: "Lesson 48 — Hashes, signatures, key management"
---
{% raw %}

# Lesson 48 — Hashes, signatures, key management

**Module 15 · Cryptography and distributed systems (first lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies (all from crates.io, versions taken from `Cargo.lock`): `sha2 0.11.0`, `blake3 1.8.7`, `hmac 0.13.0`, `hkdf 0.13.0`, `subtle 2.6.1`, `zeroize 1.9.1`, `ed25519-dalek 3.0.0`, `chacha20poly1305 0.11.0`, `argon2 0.6.0`, `getrandom 0.4.3` (51 packages in total)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** It teaches how the *building blocks* of applied cryptography behave and how to use well-reviewed Rust crates for them without making the classic mistakes. It is **not** a cryptography course and **nothing here is audited**. The crate you will build (`cryptolab`) is teaching code: it combines vetted primitives into small protocols so you can see the rules at work, and the one hand-written primitive (a SHA-256, §3.4) exists only to show how the algorithm and its padding behave. **Do not copy `cryptolab` into a product.** Not covered: TLS and certificates (PKI), key exchange and Diffie–Hellman, ECDSA and secp256k1 (they arrive with blockchains in the next lessons), RSA, post-quantum schemes, FIPS validation, hardware security modules, side-channel testing beyond one crude timing experiment, and any legal or compliance question. **Rust does not make cryptography secure**: memory safety removes one class of bugs, while the design mistakes in this lesson (reused nonces, ambiguous encodings, missing replay protection, weak key handling) are logic errors that compile perfectly.

Environment notes: crates.io and GitHub were reachable from the sandbox, but the documentation websites (`docs.rs`, `doc.rust-lang.org`, standards sites) were not, so API facts come from the compiler and the crates' source in `~/.cargo/registry`, and statements about standards or about other libraries are labeled "from memory". Test vectors from RFCs were typed from memory of the RFCs **and then cross-checked by computing the same values independently with Python's `hashlib`, `hmac` and the `cryptography` package** (✓ where stated). Timings are single-machine micro-measurements (medians of several runs), not claims about other hardware; a timing difference measured on one machine is **not** a remote attack demonstration.

Builds on [Lesson 46](46_databases_config_auth_jobs.md) (passwords, sessions, secrets in configuration), [Lesson 45](45_http_apis.md) (HTTP and where tokens travel), [Lesson 44](44_cli_serialization.md) (binary formats and serialization), [Lesson 43](43_observability_ci_supply_chain.md) (supply chain, `cargo-audit`), [Lesson 36](36_unsafe_raw_pointers.md) (why `unsafe` in cryptographic crates matters), [Lesson 38](38_no_std_embedded_miri.md) (`no_std` crypto), [Lesson 37](37_maybeuninit_ffi.md) (uninitialized memory and secrets), [Lesson 28](28_atomics_memory_ordering.md) (compiler and CPU reordering, relevant to constant-time claims), [Lesson 27](27_threads_channels.md) (threads) and [Lesson 26](26_error_design.md) (error design). Lesson 49 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

Distributed systems and blockchains rest on three promises: *this data is the data you meant* (hashes), *this came from the holder of that key* (signatures), and *only the right parties can read or alter it* (encryption and key handling). Every later lesson in this module (Merkle trees, transactions, consensus, networking) assumes you can use those promises correctly and know exactly what they do not cover.

The recurring lesson of this field is that **primitives are rarely what breaks; the way they are combined is**. In this lesson every rule is attached to an experiment you can run:

| Rule | Experiment (all ✓ in the crate) |
|---|---|
| A hash with a long output is still only as strong as its birthday bound | truncate SHA-256 to 16, 24, 32 bits and count hashes until a collision (§2.3) |
| `SHA256(key ‖ message)` is not a MAC | forge a valid tag for an extended message *without the key* using the hand-written SHA-256 (§3.5) |
| Concatenation is ambiguous | `("ab","c")` and `("a","bc")` hash identically until lengths are encoded (§4) |
| Comparing secrets with `==` leaks position | 11,000-fold timing difference between "differs at first byte" and "at last byte" for early-exit compare (§5.3) |
| One key, one purpose | HKDF subkeys with distinct labels are unrelated (§6) |
| Signatures need context, freshness and replay protection | every field of the envelope is covered by a test that flips it (§9) |
| An AEAD nonce must never repeat under one key | reuse it and recover a plaintext from two ciphertexts with an XOR (§10.2) |
| Anything the file declares, the reader must distrust | a key file that asks for a terabyte of memory is refused in under 500 ms (§10.3) |

Total in the practical challenge: one library crate, `cryptolab`, 7 modules, 61 tests, plus an example that prints the measurements quoted below.

---

## 2. Hash functions

### 2.1 What a cryptographic hash promises

A hash function maps arbitrary bytes to a fixed-size digest (SHA-256: 32 bytes). Three properties are usually distinguished (from standard textbooks):

- **Preimage resistance**: given `d`, it is infeasible to find any `m` with `H(m) = d`. For an ideal `n`-bit hash the cost is about `2^n` evaluations.
- **Second-preimage resistance**: given `m1`, it is infeasible to find a different `m2` with `H(m1) = H(m2)`. Also about `2^n`.
- **Collision resistance**: it is infeasible to find *any* pair `m1 ≠ m2` with equal hashes. Because of the **birthday paradox**, the cost is only about `2^(n/2)` evaluations, so a 256-bit hash offers about 128 bits of collision security, and a 64-bit hash offers about 32 bits, which is trivial.

Which property you need depends on the use. A checksum that only guards against accidental corruption needs little. A content address (a file name derived from its hash, a Git object id, a Merkle root) needs **collision resistance**, because an attacker who can choose both versions of a document can otherwise swap one for the other. A password verifier needs something quite different (a slow, salted function; §7), because ordinary hashes are fast by design.

A hash is **not** encryption (nothing is recoverable by design), **not** a MAC (no key; anyone can compute it), and **not** a password hash (too fast).

### 2.2 Using hashes in Rust ✓

The RustCrypto `Digest` trait gives every algorithm the same shape. The crate wraps three of them in plain functions:

```rust
use cryptolab::{digest, hex};

let d = digest::sha256(b"abc");
assert_eq!(
    hex::encode(&d),
    "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad"
);
```

✓ The values for SHA-256 (empty input, `"abc"`, the 56-byte two-block test message, and one million `a`), SHA-512 (`"abc"`), BLAKE3 (empty, `"abc"`), HMAC-SHA256 (RFC 4231 cases 1, 2 and 6), HKDF-SHA256 (RFC 5869 case 1) and Ed25519 (RFC 8032 tests 1 and 2) are checked in `tests/vectors.rs`. A **known-answer test** like this is the first test to write for any cryptographic code: it catches a wrong algorithm, a wrong parameter, a wrong endianness and a wrong crate configuration in one line. I computed each expected value a second time with Python (`hashlib`, `hmac`, the `blake3` package and `cryptography`) rather than trusting memory; all matched.

Which hash should you pick? SHA-256 is the safe, ubiquitous default (and the one Bitcoin-style systems use, as you will see); SHA-512 is its larger sibling and is often cited as faster per byte on 64-bit CPUs without hardware SHA support (from memory) (✓ here it was **slower** for 1 MiB: 543–560 MB/s versus 1459–1516 MB/s; I did not investigate why, and the ranking can differ on other CPUs); BLAKE3 is a newer, very fast, parallelizable design (✓ 3946–6599 MB/s on 1 MiB, with large run-to-run variation, since it can use SIMD and several cores). SHA-3 is a different construction (a sponge, immune to length extension by design); I did not use or measure it (from memory).

| ✓ Measured (median of 5 runs, three separate runs of the whole program) | 64-byte input | 1 MiB input |
|---|---|---|
| SHA-256 (`sha2`) | 101–104 ns | 692–719 µs (1459–1516 MB/s) |
| SHA-512 (`sha2`) | 246 ns (one run) | 1.87–1.93 ms (543–560 MB/s) |
| BLAKE3 | 60–63 ns | 159–266 µs (3946–6599 MB/s) |
| hand-written teaching SHA-256 (§3.4) | 553–573 ns | 3.95–4.43 ms (237–265 MB/s) |

Single machine, release build, whatever SIMD this CPU provided; do not generalize. The conclusion that matters: **speed is a property of the implementation and the CPU, so measure on yours, and do not choose a hash for speed unless you have measured that hashing is your bottleneck**.

### 2.3 The birthday bound, observed ✓

The theory says that with an `n`-bit output a collision is expected after roughly `√(2^n)` samples. The crate truncates SHA-256 to its first `bits` bits (for `bits` in `1..=48`; anything else is refused) and counts how many inputs (`0, 1, 2, …`) it takes until two share a truncated value:

```rust
pub fn truncated_collision(bits: u32) -> (u64, u64, u64) {
    assert!(truncation_supported(bits), "bits must be 1..=48");
    let mask: u64 = (1u64 << bits) - 1;
    let mut seen = std::collections::HashMap::new();
    let mut i: u64 = 0;
    loop {
        let d = sha256(&i.to_be_bytes());
        let t = u64::from_be_bytes(d[..8].try_into().expect("8 bytes")) & mask;
        if let Some(&j) = seen.get(&t) {
            return (j, i, i + 1);
        }
        seen.insert(t, i);
        i += 1;
    }
}
```

✓ The run is deterministic (the inputs are fixed): 16 bits → first collision after **232** hashes (`√2^16 = 256`); 24 bits → **4,303** (`4,096`); 32 bits → **95,754** (`65,536`). One sample per size is not a statistic (the expected value of the first collision is about 1.25 times the square root; the spread is wide), but the scaling is clear: each 8 more bits multiplies the work by about 16, not 256. Extrapolate with care: this says that a **64-bit identifier** can be collided with about 2^32 hashes (a few hours on a modern machine is plausible, from memory, not measured here), which is why short hash prefixes are fine for display and wrong for security.

### 2.4 Hashing structured data ✓

Hashing a Rust value is not hashing its meaning. `std::hash::Hash` and `DefaultHasher` are for hash tables: they are not stable across Rust versions and platforms, and they are not cryptographic. To hash a structure, first **serialize it to a canonical byte string** (one and only one encoding per value), then hash the bytes. §4 shows what goes wrong otherwise.

---

## 3. Inside SHA-256, and why length extension exists

### 3.1 Merkle–Damgård in one paragraph

SHA-256 (from memory of FIPS 180-4, and ✓ confirmed by the implementation below) pads the message so its length is a multiple of 64 bytes, splits it into blocks, and feeds the blocks one by one through a **compression function** that updates a 256-bit internal state (eight 32-bit words). The initial state is fixed. The final state *is* the digest. This is the **Merkle–Damgård** construction. Its weakness is plain: the digest is the entire internal state, so anyone who knows `H(m)` and the length of `m` can restart the machine from that state and continue hashing more blocks, producing `H(m ‖ padding ‖ suffix)` without knowing `m`.

### 3.2 Padding

The padding appends a `0x80` byte, then zero bytes until the length is 56 mod 64, then the message length **in bits** as an 8-byte big-endian number. ✓ The teaching implementation computes it in one place and tests it for every length from 0 to 300 against the real crate:

```rust
// toy_sha256::padding(msg_len) -> Vec<u8>
// invariants checked in tests/toy_sha256.rs:
//   (msg_len + padding.len()) % 64 == 0
//   padding[0] == 0x80
//   the last 8 bytes are msg_len * 8 as big-endian
```

The edge cases are exactly where a hand-written hash breaks: lengths 55, 56, 63, 64 and 119/120, where the length field no longer fits in the current block and a whole extra block appears. They are tested explicitly.

### 3.3 Constants are not magic ✓

The 64 round constants of SHA-256 are the first 32 bits of the fractional parts of the cube roots of the first 64 primes, and the initial state is the same for square roots of the first 8 primes (from the standard's description). The teaching implementation does not paste a table; it **derives** them in `const fn` code with integer cube and square roots, and the known-answer tests would fail on a single wrong bit. A mutant that changes one constant is caught (§18).

### 3.4 A from-scratch SHA-256, for understanding only ✓

`toy_sha256.rs` (in the practical challenge) is about 250 lines. It exists for three reasons: to see the round function, to compare byte for byte with `sha2` on every length from 0 to 300 plus boundary lengths and chunked `update` calls (✓ 10 tests), and to implement the attack in §3.5. It is **not** constant-time-audited, **not** optimized (✓ roughly 6× slower than `sha2` for 1 MiB: 237–265 MB/s versus 1459–1516 MB/s here), and you must never use a hand-written hash in real software. The same advice applies to every primitive: **use a reviewed crate**.

### 3.5 Length extension, performed ✓

Take the tempting construction `tag = SHA256(key ‖ message)`. An attacker who sees one pair `(message, tag)` and knows the *length* of the key can compute, without the key, a valid tag for `message ‖ padding ‖ suffix`:

1. Treat `tag` as the internal state after hashing `key ‖ message ‖ padding` (a whole number of blocks).
2. Resume hashing from that state, pretending `len(key ‖ message ‖ padding)` bytes were processed, over `suffix`.

The crate does exactly this in `toy_sha256::extend(known_digest, unknown_len, suffix)`, which returns the padding "glue" the attacker must insert and the new digest. ✓ The test `length_extension_forges_a_tag_for_sha256_key_concat_message` verifies that the forged message and tag are accepted by the victim function built on the *real* `sha2` crate. Two further tests record the limits of the attack: it needs the correct secret length (✓ a wrong guess fails, so an attacker simply tries all plausible lengths), and it **does not work against HMAC** (✓ the same forged pair is rejected).

**Why HMAC resists it** (from the construction, ✓ in the sense that the tests show rejection): HMAC computes `H((K ⊕ opad) ‖ H((K ⊕ ipad) ‖ m))`. The attacker sees only the *outer* hash, whose input includes a key-derived block the attacker cannot reproduce, and whose own input is a fixed-size inner digest, so there is nothing to extend.

**Design rationale.** Merkle–Damgård hashes expose their state; SHA-3 (a sponge) and "truncated" variants such as SHA-512/256 do not (from memory), and BLAKE3 has a keyed mode (from memory, not used here). Whatever the hash, the correct habit is: **use the primitive designed for the job (HMAC, or a keyed hash), not a home-made composition of a hash and a key**.

---

## 4. Unambiguous encoding and domain separation ✓

Suppose a token is `SHA256(secret ‖ user ‖ role)`. Then `("bob", "admin")` and `("bo", "badmin")` produce the same input bytes, so a token for one is valid for the other (the debugging challenge in §16 makes you fix this). The general rule: **whenever you hash or sign several fields, encode each with its length** (or use a format with unambiguous framing), so that different field lists can never produce the same byte string.

`cryptolab::digest::hash_fields` does this and also adds a *domain label*, a string naming the purpose:

```rust
pub fn hash_fields(domain: &str, fields: &[&[u8]]) -> [u8; 32] {
    let mut h = Sha256::new();
    h.update((domain.len() as u64).to_be_bytes());
    h.update(domain.as_bytes());
    h.update((fields.len() as u64).to_be_bytes());
    for f in fields {
        h.update((f.len() as u64).to_be_bytes());
        h.update(f);
    }
    h.finalize().into()
}
```

- **Length prefixes make the encoding injective.** ✓ Tests: `("ab","c")` ≠ `("a","bc")`; an empty field is distinct from no field; order matters.
- **The domain label separates purposes.** The same fields hashed for "session id" and for "password reset" must not collide in meaning, or a value minted for one purpose might be replayed for another. ✓ Tested: same fields, different domain, different digest.
- **The field count is redundant** (length prefixes alone are already injective): ✓ removing it did not change any test result, and I list that mutant as *equivalent* (§18). It is kept as defense in depth. Also notice the subtle part: the *domain length prefix* is **not** redundant. ✓ A mutant that removed it survived my first test run; I then constructed an explicit collision (a domain string that embeds the next 16 bytes of the encoding) and added the test `hash_fields_domain_length_prefix_blocks_a_boundary_shift_collision`, which kills it. Without the prefix, two different calls produce identical bytes.

Binary formats from [Lesson 44](44_cli_serialization.md) (such as length-delimited framing) are the same idea; canonical JSON is notoriously harder (key order, number formatting, Unicode normalization; general knowledge).

---

## 5. Message authentication: HMAC and constant-time comparison

### 5.1 A MAC proves origin and integrity to a key holder

A **message authentication code** takes a secret key and a message and produces a tag. Anyone with the key can recompute the tag; nobody without it can forge one (that is the security goal). A MAC proves the message came from *someone holding the key* and was not altered. It does not prove *which* holder (both parties share the key), and it is not a signature in the transferable sense (§8).

```rust
use hmac::{Hmac, KeyInit, Mac};
use sha2::Sha256;

let mut m = <Hmac<Sha256> as KeyInit>::new_from_slice(key).expect("any length");
m.update(msg);
let tag = m.finalize().into_bytes();
```

(`hmac 0.13` needs `KeyInit` in scope for `new_from_slice`; omitting that import is the compile error I hit first.) ✓ Tests: RFC 4231 vectors 1, 2 and 6 (the last has a 131-byte key, longer than the block size, which exercises the key-hashing path); flipping any bit of the message, the tag or the key invalidates the tag; truncated tags and extension attempts fail.

### 5.2 Verify with the library, not with `==`

`Mac::verify_slice` compares in constant time and checks the length. The crate's `verify_hmac_sha256(key, msg, tag)` is a thin wrapper. Never write `computed == received` on secret-dependent data.

### 5.3 Why: early-exit comparison leaks, measured ✓

`naive_eq` returns at the first differing byte; `ct_eq` (from the `subtle` crate) examines every byte whatever the data. On two 64 KiB buffers, three runs of the benchmark:

| ✓ | differs at first byte | differs at last byte | ratio |
|---|---|---|---|
| `naive_eq` | 1–3 ns | 30.9–36.3 µs | about 11,000× to 22,000× |
| `ct_eq` | about 83–89 µs | about 83–89 µs | 1.0× to 1.1× |

Notice what this does and does not show. It shows the *mechanism* (the work depends on the position of the first difference) in a local microbenchmark with huge buffers. It does **not** show that an attacker over a network can recover a 32-byte tag byte by byte: real attacks fight noise, and the difference per byte of a 32-byte comparison is a few nanoseconds (from memory, remote timing attacks on such differences have been demonstrated in research under good network conditions; I did not test it). The right engineering conclusion is the cheap, absolute rule: **use a constant-time comparison for secrets, because it costs almost nothing and removes the question**.

Caveats about "constant time" in Rust (from memory): `subtle` uses tricks such as `black_box`-like barriers to keep the optimizer from turning the comparison back into an early exit, and the guarantee is best-effort and tied to the compiler version; I only measured one build. Also, `==` on a `[u8; N]` is allowed to return early, and comparing the *hex strings* is no better.

---

## 6. Key derivation: one key, one purpose ✓

A single master secret should not be used directly for two different things (encrypt and authenticate, or two protocols): a weakness in one use, or an accidental interaction between them, can compromise the other. **HKDF** (RFC 5869, from memory of the document, ✓ vector 1 checked) turns input key material into any number of independent keys: *extract* a pseudorandom key from the input (with an optional salt), then *expand* with an `info` label per purpose.

```rust
use hkdf::Hkdf;
use sha2::Sha256;

let hk = Hkdf::<Sha256>::new(None, master);
let mut enc = [0u8; 32];
let mut mac = [0u8; 32];
hk.expand(b"myapp/enc/v1", &mut enc).expect("valid length");
hk.expand(b"myapp/mac/v1", &mut mac).expect("valid length");
```

✓ Facts verified by tests in the crate: different `info` labels give unrelated keys; the same inputs always give the same keys; a shorter output is a *prefix* of a longer one for the same label (so never "extend" a key by asking for more bytes and calling it a new key); the maximum output is `255 × 32 = 8,160` bytes and asking for more fails; a zero-length request is refused by the crate's wrapper. The `subkey(master, purpose)` function returns `Zeroizing<[u8; 32]>`, a wrapper that overwrites the bytes on drop (§11.2).

HKDF is for inputs that already have **high entropy** (a random key, a shared secret). It is **not** for passwords: a guessable input stays guessable and HKDF is fast. For passwords use a deliberately slow memory-hard function (§7, §10).

---

## 7. Passwords are different (a reminder, and Argon2id)

[Lesson 46](46_databases_config_auth_jobs.md) covered storing password verifiers. The principle again: human passwords have little entropy, so the defender must make each guess **expensive**. **Argon2id** (the winner of the Password Hashing Competition, from memory; RFC 9106) takes a memory size `m`, a pass count `t` and a parallelism, and forces the attacker to spend `m` KiB of memory per guess. `cryptolab` uses it for one purpose, deriving a key-encryption key from a passphrase (§10). ✓ Measured on this machine (open time of a sealed key): `m = 8 KiB, t = 1`: 0.01–0.02 ms; `m = 19,456 KiB (19 MiB), t = 2` (the crate's default): 16.5–19.0 ms; `m = 65,536 KiB, t = 3`: 141.9–150.5 ms. The right parameters are "as large as your latency and memory budget allow" and should be revisited as hardware changes (guidance from memory; I did not consult the current OWASP recommendation, so check it).

---

## 8. Digital signatures: Ed25519

### 8.1 What a signature adds over a MAC

A signature scheme has a **private** signing key and a **public** verifying key. Anyone holding the public key can verify, but only the private-key holder can sign. This gives **non-repudiation in the technical sense** (the signer cannot claim someone else produced it unless the key leaked), and verifiers need no secret. That is why blockchains use signatures: thousands of nodes verify a transaction, and none can forge it.

**Ed25519** (RFC 8032, from memory; ✓ vectors 1 and 2 checked) is a modern elliptic-curve signature with 32-byte keys and 64-byte signatures. Properties relevant to engineering:

- **Deterministic**: signing the same message with the same key gives the same signature (✓ tested). That removes a whole class of failures where weak randomness during signing leaks the key (a flaw that affected other schemes; from memory).
- **Fast** (✓ here: sign a 100-byte message 14.0–17.6 µs; `verify_strict` 36.0–45.6 µs; key generation from a seed 11.9–15.3 µs).
- **Keys are 32-byte seeds**: the secret is the seed, and the public key is derived from it. `Keypair::from_seed` makes tests reproducible; `Keypair::generate` uses the operating system's random source through `getrandom`.

### 8.2 `verify_strict` and weak keys ✓

`ed25519-dalek` offers `verify` and `verify_strict`. The strict variant additionally rejects signatures that are valid only because the public key or a signature component is a **small-order point**, a degenerate value for which many messages "verify". Since this changes whether *the same bytes* are accepted, systems that must agree (like blockchain nodes) have to pick one rule everywhere. `cryptolab` uses `verify_strict`, **and** rejects weak public keys when they are registered (✓ test: the identity point `[1, 0, 0, …]` is refused by `trust`). Honest note: ✓ replacing `verify_strict` by `verify` is an **equivalent mutant** in my test suite, because the weak-key rejection at registration already makes the difference unreachable for my tests. A malicious-signature test for that difference would need hand-crafted small-order encodings, which I did not write; real systems should treat the choice as part of their specification.

### 8.3 The key id

Signed messages say which key signed them by an 8-byte `key_id` (the first 8 bytes of SHA-256 of the public key). ✓ For the RFC 8032 test-1 public key the id is `21fe31dfa154a261` (checked with Python and pinned by a known-answer test, which also kills the mutant that slices the wrong bytes). An id is a **lookup hint, not a security feature**: the verifier uses it to find a registered key, and the signature check does the real work. Eight bytes is fine for a lookup among your own few keys; two keys with equal ids simply cannot both be registered (✓ `trust` refuses duplicates).

---

## 9. Signed messages in practice: the envelope

A bare signature over a payload is not enough, because of what a valid signature *cannot* tell you. The following attacks need answers in the design:

| Attack | Defense in `cryptolab::sign` | ✓ tested |
|---|---|---|
| Reuse a signature from another protocol/purpose ("cross-protocol") | A **context string** and a fixed **protocol label** are part of the signed bytes | changing the context fails |
| Move the same bytes under another key | the signed bytes include the **key id** | changing it fails |
| Change the payload, time or nonce | all are inside the signature | each field is flipped in turn |
| Re-send an old valid message (**replay**) | a random 16-byte **nonce** plus a remembered set; also a **timestamp** and a freshness window | second delivery is `Replay` |
| Message from the far past or future | `max_age` and `max_skew` bounds | exact boundaries tested (inclusive) |
| Compromised or retired key | each key has a **validity window** `[not_before, not_after)`; overlap lets rotation happen without downtime | rotation overlap test |
| Memory exhaustion by flooding | replay cache has a **capacity**; forged messages never enter the cache because the cache is touched only *after* a valid signature | `CacheFull` and forged-message tests |
| Ambiguous framing of the signed bytes | each part is length-prefixed | different splits differ |
| Weak public keys | refused at registration | identity point refused |

### 9.1 The verification order matters ✓

`Verifier::verify` checks, in this order: key lookup → the key's validity window → timestamp not too far in the future → not too old → signature (`verify_strict`) → prune old nonces → replay check → capacity check → remember the nonce. Cheap checks come first; **state is changed only after a valid signature**, so an attacker with no key cannot fill the cache or make a legitimate message look like a replay.

Time is *injected* (`verify(context, msg, now)`) rather than read from the clock, so every boundary can be tested exactly (✓ age equal to `max_age` is accepted, one more second is not; skew is the same). Arithmetic on timestamps uses `saturating_add`/`saturating_sub`, because a hostile timestamp near `u64::MAX` must not wrap and look valid (✓ tests with timestamps near the limit; mutants that use plain `+` and `-` are caught).

### 9.2 Replay protection has costs

A nonce cache is state: it must be bounded (done), shared between all verifier instances behind a load balancer (not done: this teaching verifier is single-process), and it must survive restarts or the freshness window must be shorter than the restart time (a design question, not code). Pruning removes entries from the front of an insertion-ordered queue, so messages with out-of-order timestamps may linger until capacity or later pruning pushes them out (the capacity bound still holds). Production systems often prefer protocols with sequence numbers or one-time challenge values; the right choice depends on the system (general knowledge).

### 9.3 What the envelope still does not do

It does not hide the payload (no encryption), does not authenticate *who* the key belongs to (that is what certificates and registries do; out of scope), and does not protect against a stolen private key (which is the subject of §10 and §11).

---

## 10. Protecting a key at rest

### 10.1 The design ✓

A signing seed in a file must be encrypted under a key derived from a passphrase. `keys.rs` seals a seed as:

```
"CLK1" | version (1) | m_kib (u32 BE) | t (u32 BE) | salt (16) | nonce (24) | ciphertext+tag (48)
```

The header is 53 bytes and the whole blob 101 bytes. The key-encryption key is Argon2id (version 0x13, parallelism 1) over the passphrase and a fresh random 16-byte salt, producing 32 bytes; the cipher is **XChaCha20-Poly1305**, an AEAD (authenticated encryption with associated data), with a fresh random 24-byte nonce, and the header is passed as **associated data**, so it is authenticated but not encrypted.

### 10.2 AEAD, and why nonces matter ✓

An AEAD encrypts and authenticates in one step: decryption either returns the exact plaintext or fails, so you cannot get garbled output from a modified ciphertext. ✓ Test: ciphertext length is plaintext length + 16 (the tag); every single-byte modification of the 101-byte sealed blob (101 positions) is rejected; the same passphrase gives a different blob each time (fresh salt and nonce).

**The one rule: a (key, nonce) pair must never be used twice.** ✓ The test in `tests/lessons.rs` demonstrates why with ChaCha20-Poly1305: encrypting two 16-byte messages under the same key and nonce produces ciphertexts whose XOR equals the XOR of the plaintexts (the stream cipher keystream cancels), so knowing one plaintext reveals the other without the key; with distinct nonces that relation disappears. (The forgery side of nonce reuse, recovering the authentication key, is also a known consequence for Poly1305-based AEADs; from memory, not demonstrated.)

How many random nonces are safe? With `n` messages and a `b`-bit random nonce, the probability of any repeat is about `n² / 2^(b+1)`. For the 96-bit nonce of ChaCha20-Poly1305, a 2⁻³² risk is reached near `2^32.5` messages, a limit a busy service could hit; for the 192-bit nonce of XChaCha20, near `2^80.5` messages, which is why XChaCha20 is the usual choice for *random* nonces (✓ arithmetic checked in exercise 8). A counter nonce is the alternative, and has its own risk (restarts, replicas).

### 10.3 Distrust the file ✓

The header contains the Argon2 cost parameters. If `open_seed` simply used them, an attacker who can edit the file (or who hands you one) could request gigabytes of memory or a huge pass count and take your process down before any password check happens. So `open_seed` **range-checks `m` and `t` against fixed limits before running Argon2** (`MIN_M_KIB = 8`, `MAX_M_KIB = 262,144`, `MAX_T = 10`). ✓ Test: a blob with `m = u32::MAX`, `m = MAX+1`, `t = 0`, `t = MAX+1` and several others is refused with `ParamsOutOfRange` in under 500 ms each. The mutation run found two things here: removing the check made the test **hang** (the hostile parameters really do start work; I record those two mutants as caught by timeout, not by an assertion), and a mutant that changed the limit constant itself survived until I added `cost_limits_are_pinned_policy_values`, because tests that import the constant cannot detect a changed constant. Moral: **pin policy values with literal numbers in a test**.

Other properties, all ✓ tested: the error for a wrong passphrase and for a damaged file is the *same* error ("wrong passphrase or damaged key file"), so an attacker cannot tell which; the plaintext seed never appears in the blob; version and magic are checked; blobs of the wrong length are refused; an empty passphrase is refused when sealing.

Honest note about the associated data: authenticating the header is good practice, but ✓ in this format it is nearly redundant, since every header field either feeds the key derivation or the nonce or is range-checked (changing `m`, `t` or the salt changes the derived key, so decryption fails anyway). When I removed the header from the associated data on **one** side only (seal or open), the round-trip test caught it. I did not test removal on both sides, which I expect is behaviorally equivalent here; the AAD is defense in depth and protects future header fields.

### 10.4 Files on disk ✓

`write_key_file` creates the file with `create_new` (so it will not overwrite an existing key; ✓ tested) and mode `0o600`, then `sync_all`; `read_key_file` refuses a file whose mode has any group or other bits (`mode & 0o077 != 0`; ✓ tested with a table of modes). Unix only; on other systems this check is not implemented here. Even so, file permissions do not stop root, a backup tool, or another process of the same user.

---

## 11. Handling secrets in Rust

### 11.1 Do not print them

`Keypair`'s `Debug` implementation prints `Keypair { key_id: <hex>, secret: <redacted> }` (✓ tested; a mutant that leaks the seed is caught). Derive `Debug` on a struct holding a key and one stray `{:?}` or `tracing` call writes the key to the logs ([Lesson 43](43_observability_ci_supply_chain.md)). Errors in this crate never contain key material.

### 11.2 Zeroizing

`zeroize::Zeroizing<T>` overwrites its contents when dropped, using writes the optimizer is told not to remove. It reduces how long a secret sits in memory; it does **not** prevent copies made earlier (a `Vec` that grew and moved, a value passed by value, the kernel's swap or core dumps, the CPU registers). From memory, and consistent with the crate's documentation: treat zeroization as hygiene, not as a guarantee. The crate returns seeds and derived keys as `Zeroizing` so the common copies are cleaned.

### 11.3 Randomness

Use the operating system's random source (`getrandom`, which `ed25519-dalek` and the AEAD crates use through `rand_core`) for keys, salts, nonces. Never use a time-seeded generator or a non-cryptographic one (`fastrand`-style generators are for games). `random_bytes::<N>()` in the crate wraps `getrandom::fill` and panics only if the OS cannot provide randomness, a condition where continuing would be worse.

### 11.4 `unsafe` and constant time

Reviewed crypto crates contain `unsafe` or architecture intrinsics for speed ([Lesson 36](36_unsafe_raw_pointers.md) says how to read such code). Your job is to choose crates carefully ([Lesson 43](43_observability_ci_supply_chain.md): maintenance, audits, `cargo-audit`), pin versions with `Cargo.lock`, and update when advisories appear.

---

## 12. Key management: the part that decides outcomes

Algorithms rarely fail; operations do. Concepts (general practice, from memory unless marked; nothing here is tested):

- **Separation**: different keys for signing, encryption, authentication, environments (test versus production), and tenants. HKDF subkeys (§6) give separation from one master but do not reduce the blast radius of the master itself.
- **Generation**: with a proper random source, ideally where the key will be used.
- **Storage**: the secret should be as hard to exfiltrate as the system allows: encrypted at rest (§10), kept out of source control and logs ([Lesson 46](46_databases_config_auth_jobs.md)), held by a **key management service or hardware module** (cloud KMS, HSM, a smartcard) so that the application can ask for signatures without ever holding the key. I did not use any of these here.
- **Rotation**: change keys on a schedule and after any suspicion of compromise. The code supports it by validity windows with overlap (✓ test): the new key is registered with `not_before` before the old one's `not_after`, so messages in flight remain valid.
- **Revocation**: removing trust in a key before its window ends. A small verifier can simply stop trusting it; distributed systems need a way to propagate that (a revocation list, short-lived keys). The teaching verifier has no revoke call, which is a missing feature, not a feature.
- **Backup and recovery**: a key you cannot recover is a data-loss bug; a key backup is another place to steal it from.
- **Compromise plan**: write down in advance what you do when a key leaks (rotate, revoke, re-sign, notify), because the decisions are hard under pressure.

---

## 13. Common mistakes

1. `SHA256(key ‖ msg)` (or `SHA256(msg ‖ key)`) as a MAC. Use HMAC (✓ length extension, §3.5).
2. Comparing MACs, tokens or signatures with `==` (§5.3).
3. Hashing concatenated fields without lengths (§4).
4. One key for two jobs (§6).
5. Reusing a nonce under the same key, or a random 96-bit nonce at high volume (§10.2).
6. Using a fast hash for passwords (§7).
7. Trusting parameters inside a file or message before bounds-checking them (§10.3).
8. Verifying a signature but ignoring context, time and replay (§9).
9. Doing arithmetic on attacker-supplied numbers with `+` and `-` (✓ mutants §9.1).
10. Error messages that distinguish "no such user" from "bad password", or "bad padding" from "bad tag" (✓ one error for key files).
11. Logging secrets through `Debug` (§11.1).
12. Rolling your own primitive, or "improving" a standard one (§3.4).
13. Believing that Rust, a type system, or passing all tests makes the design secure.

---

## 14. Security notes and honest limits

- `cryptolab` has not been audited, was written for teaching, and has not been tested for timing side channels beyond the crude experiment in §5.3 (which tests *my comparison functions*, not the dependency crates).
- The crates used implement well-known algorithms; I did not review them, and I did not check them against advisory databases in this session beyond the fact that their versions were the newest available when I resolved the lock file.
- Test vectors prove an implementation agrees with the *vector*, not that it is free of side channels or misuse hazards.
- Quantum computers: Ed25519 and other elliptic-curve schemes would be broken by a large quantum computer (general knowledge); hash functions and symmetric ciphers lose some margin. Post-quantum schemes are out of scope.
- A signature proves possession of a key at signing time, not the identity, intent or sanity of the holder.

---

## 15. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Which hash property protects (a) a content address, (b) a stored digest of a file you downloaded from an untrusted mirror when you got the digest from the official site, (c) a password verifier? Which of the three is *not* mainly a hash-function question?
2. **(Warm-up)** About how many hashes does it take to find a collision in a hash truncated to 32 bits? To 64 bits? What did the experiment show for 32 bits?
3. **(Easy)** Write `tag(key, msg) -> [u8; 32]` and `verify(key, msg, tag) -> bool` with HMAC-SHA256 so that verification is constant time and a tag of the wrong length is simply `false`.
4. **(Medium)** Write `encode_fields(&[&[u8]]) -> Vec<u8>` (a 4-byte big-endian length before each field) and `decode_fields(&[u8]) -> Option<Vec<&[u8]>>` that rejects truncated input and any length that runs past the end, and never panics.
5. **(Medium)** Write `sign(key, context, msg)` and `verify(key, context, msg, sig)` for Ed25519 such that a signature made for one context fails in another, with unambiguous framing of the context and the message.
6. **(Medium-hard)** From one 32-byte master key derive an encryption key and a MAC key with HKDF so that neither reveals the other. Which two details of the call make them independent? What is wrong with deriving them as the first and second halves of one 64-byte expand?
7. **(Hard)** Design signing-key rotation for a service whose verifiers are deployed in 20 places and cannot all be updated at the same instant. What are the phases, what overlap do you need, and what happens to a leaked key?
8. **(Hard)** You encrypt each record with ChaCha20-Poly1305 (96-bit nonce) under one key and choose nonces at random. At what message count is the chance of any repeat about 2⁻³²? Repeat for XChaCha20-Poly1305 (192-bit nonce). Why does the repeat matter more than "a tiny chance of a wrong answer"?

<details markdown="1">
<summary>Answers</summary>

1. (a) A content address needs **collision resistance** (if an attacker can craft two inputs with one hash they can substitute one for the other). (b) Checking a download against a digest from a trusted source needs **second-preimage resistance** for that file (the attacker has the original and wants a different file with the same hash); it also assumes the digest itself came over a trusted channel, which is the real weak point, and a fast hash mismatch only proves integrity. (c) A password verifier needs **preimage resistance plus slowness and a salt**, so it is not mainly a hash-function question; it is a *cost* question (§7), because passwords are guessable and ordinary hashes are fast.
2. About `2^(n/2)`: roughly 2^16 = 65,536 for 32 bits (the constant is a bit above 1), and roughly 2^32 ≈ 4.3 billion for 64 bits. ✓ The experiment found a 32-bit collision after 95,754 hashes (one deterministic sample; the expected value is a little above 65,536).
3. A possible answer (✓ compiled and run):

   ```rust
   use hmac::{Hmac, KeyInit, Mac};
   use sha2::Sha256;

   pub fn tag(key: &[u8], msg: &[u8]) -> [u8; 32] {
       let mut m =
           <Hmac<Sha256> as KeyInit>::new_from_slice(key).expect("HMAC takes any key length");
       m.update(msg);
       m.finalize().into_bytes().into()
   }

   pub fn verify(key: &[u8], msg: &[u8], tag: &[u8]) -> bool {
       let mut m =
           <Hmac<Sha256> as KeyInit>::new_from_slice(key).expect("HMAC takes any key length");
       m.update(msg);
       m.verify_slice(tag).is_ok()
   }
   ```

   `verify_slice` compares in constant time and returns an error for a wrong length, so no length check of your own is needed; do not replace it with `tag == computed`.
4. A possible answer (✓ compiled and run, including an empty field, truncation and a huge length):

   ```rust
   pub fn encode_fields(fields: &[&[u8]]) -> Vec<u8> {
       let mut out = Vec::new();
       for f in fields {
           let len = u32::try_from(f.len()).expect("field longer than 4 GiB");
           out.extend_from_slice(&len.to_be_bytes());
           out.extend_from_slice(f);
       }
       out
   }

   pub fn decode_fields(mut bytes: &[u8]) -> Option<Vec<&[u8]>> {
       let mut out = Vec::new();
       while !bytes.is_empty() {
           let (head, rest) = bytes.split_first_chunk::<4>()?;
           let len = u32::from_be_bytes(*head) as usize;
           if rest.len() < len {
               return None;
           }
           let (field, rest) = rest.split_at(len);
           out.push(field);
           bytes = rest;
       }
       Some(out)
   }
   ```

   Edge cases worth a test: an empty field (distinct from no field), a length that claims more than remains, a truncated length prefix, and trailing bytes (the loop treats them as another field, so they either decode or fail, never silently ignored). `u32::try_from(f.len())` makes the 4 GiB limit explicit rather than silently truncating.
5. A possible answer (✓ compiled and run):

   ```rust
   use ed25519_dalek::{Signature, SigningKey, VerifyingKey};

   fn framed(context: &str, msg: &[u8]) -> Vec<u8> {
       let mut v = Vec::new();
       v.extend_from_slice(&(context.len() as u64).to_be_bytes());
       v.extend_from_slice(context.as_bytes());
       v.extend_from_slice(msg);
       v
   }

   pub fn sign(key: &SigningKey, context: &str, msg: &[u8]) -> Signature {
       use ed25519_dalek::Signer;
       key.sign(&framed(context, msg))
   }

   pub fn verify(key: &VerifyingKey, context: &str, msg: &[u8], sig: &Signature) -> bool {
       key.verify_strict(&framed(context, msg), sig).is_ok()
   }
   ```

   The 8-byte length of the context makes `("a", "bc…")` and `("ab", "c…")` different byte strings, so a signature cannot be re-read under another split. Ed25519 is deterministic, so the same inputs give the same signature (also tested).
6. A possible answer (✓ compiled and run):

   ```rust
   use hkdf::Hkdf;
   use sha2::Sha256;

   pub fn derive_two(master: &[u8; 32]) -> ([u8; 32], [u8; 32]) {
       let hk = Hkdf::<Sha256>::new(None, master);
       let mut enc = [0u8; 32];
       let mut mac = [0u8; 32];
       hk.expand(b"myapp/enc/v1", &mut enc)
           .expect("32 bytes is a valid length");
       hk.expand(b"myapp/mac/v1", &mut mac)
           .expect("32 bytes is a valid length");
       (enc, mac)
   }
   ```

   Independence comes from (1) the same extract step giving one pseudorandom key and (2) *different `info` labels*, which make the two outputs unrelated (in the HKDF security argument; from memory). Taking the first and second halves of one 64-byte expand also works technically, but then the two keys are positions in one stream, and an output of length 32 is a *prefix* of the 64-byte output (✓ tested in the crate), which makes it easy to confuse the two uses by asking for the wrong length. Labels name the purpose; halves do not. Also include a version in the label so you can change the derivation later.
7. Phases: (1) generate the new key pair; (2) **distribute the new public key to every verifier first**, with `not_before` = rollout time and while the old key remains valid; (3) wait until every verifier trusts both (monitor deployed versions; this is the step that needs the overlap equal to your slowest deployment plus the message lifetime `max_age`); (4) switch signers to the new key; (5) after `max_age` plus clock skew has passed, let the old key's `not_after` expire and remove it. The window `[not_before, not_after)` of the crate expresses exactly this, and the rotation test checks the overlap. A **leaked key** is different: you cannot wait. Remove trust in it everywhere as fast as possible (a push-based revocation or very short key lifetimes), accept that messages signed before the leak but unverified until later may need re-signing, and investigate what could have been signed in the meantime. Without a revocation channel, the damage window equals the verifiers' update time. (General practice; the crate has no revocation call.)
8. With `n` random messages and a `b`-bit nonce the repeat probability is about `n² / 2^(b+1)`, so `n ≈ 2^((b + 1 − 32)/2)`: for `b = 96`, `n ≈ 2^32.5` (about 6 billion messages); for `b = 192`, `n ≈ 2^80.5`. ✓ The arithmetic function from the crate's scratch crate returns exactly 32.5 and 80.5. A repeat matters because it is not "a wrong answer": with a stream cipher AEAD, two messages under the same key and nonce leak the XOR of their plaintexts (✓ shown in `tests/lessons.rs`), and for Poly1305-based AEADs it can also expose the authentication key (from memory), so one repeat can break confidentiality and integrity for that key.

</details>

---

## 16. Debugging challenge

A service issues session tokens as `hex(SHA256(secret ‖ user ‖ role))` and checks them with `==`. A penetration test reports three problems: (1) a user "bob" with role "admin" can present a token to be accepted as user "bo" with role "badmin"; (2) given one valid token and the secret's length, an attacker can produce tokens for messages they did not receive; (3) a timing difference between wrong guesses. Find all three, and fix the code, keeping the same inputs (the token type may change from `String` to raw bytes).

```rust
use sha2::{Digest, Sha256};

pub fn make_token(secret: &[u8], user: &str, role: &str) -> String {
    let mut h = Sha256::new();
    h.update(secret);
    h.update(user.as_bytes());
    h.update(role.as_bytes());
    h.finalize().iter().map(|b| format!("{b:02x}")).collect()
}

pub fn check_token(secret: &[u8], user: &str, role: &str, token: &str) -> bool {
    make_token(secret, user, role) == token
}
```

<details markdown="1">
<summary>Solution</summary>

1. **Ambiguous concatenation.** `user ‖ role` has no separator or lengths, so `"bob"+"admin"` and `"bo"+"badmin"` hash the same bytes (✓ the test `buggy_accepts_a_shifted_boundary` reproduces it). Fix: length-prefix every field.
2. **Not a MAC.** `SHA256(secret ‖ …)` allows length extension (§3.5): an attacker with one token and the secret's length can append bytes and compute the matching token. Fix: use HMAC-SHA256 keyed with the secret.
3. **Variable-time comparison**: `==` on strings returns at the first differing byte (§5.3). Fix: `verify_slice` (constant time) on the raw bytes, not the hex.

Also: hex strings are not needed inside the check, so the fix compares raw tags; and the secret should be a random key of at least 16–32 bytes, kept out of logs. Add a timestamp or an expiry to the signed data if tokens must expire (not shown), and note that a MAC token proves *the server issued it*, not that the user is still allowed to be an admin (revocation is a separate problem).

Fixed version (✓ compiled; the test `fixed_rejects_it` checks the shifted split, the wrong length and the correct token):

```rust
use hmac::{Hmac, KeyInit, Mac};
use sha2::Sha256;

fn mac(secret: &[u8], user: &str, role: &str) -> Hmac<Sha256> {
    let mut m = <Hmac<Sha256> as KeyInit>::new_from_slice(secret).expect("any length");
    m.update(&(user.len() as u64).to_be_bytes());
    m.update(user.as_bytes());
    m.update(&(role.len() as u64).to_be_bytes());
    m.update(role.as_bytes());
    m
}

pub fn make_token(secret: &[u8], user: &str, role: &str) -> Vec<u8> {
    mac(secret, user, role).finalize().into_bytes().to_vec()
}

pub fn check_token(secret: &[u8], user: &str, role: &str, token: &[u8]) -> bool {
    mac(secret, user, role).verify_slice(token).is_ok()
}
```

</details>

---

## 17. Interview questions

1. *Which properties does a cryptographic hash need? Which one decides the safety of a content address?*
2. *What is a length-extension attack, and why is HMAC not vulnerable?*
3. *Why compare MACs in constant time, and what is the limit of that argument?*
4. *Why is it dangerous to hash `a ‖ b` instead of a length-prefixed encoding? What is a domain separator for?*
5. *What is the difference between a MAC and a digital signature? When would you use each?*
6. *Why does Ed25519's determinism matter? What does `verify_strict` add?*
7. *A signed message is valid. List everything else a verifier should check before acting on it.*
8. *What is an AEAD nonce, and what happens if you reuse one?*
9. *How do you store a signing key on disk? What should the file reader never trust?*
10. *What is the difference between HKDF and Argon2? When do you use each?*
11. *How would you rotate a signing key across many verifiers? How would you handle a leaked key?*
12. *Does `Zeroizing` guarantee a secret is gone from memory?*

<details markdown="1">
<summary>Answers</summary>

1. Preimage, second-preimage and collision resistance (§2.1). A content address needs collision resistance, whose cost is only about `2^(n/2)` because of the birthday bound (✓ seen in the truncation experiment).
2. In Merkle–Damgård hashes the digest is the internal state, so `H(k ‖ m)` plus the key length lets an attacker continue hashing (✓ forged in §3.5). HMAC hashes twice with key-derived blocks, so the attacker never sees a state they can continue.
3. An early-exit compare leaks the position of the first difference through timing (✓ 11,000× in a local microbenchmark). The limit: exploiting it remotely is hard and noisy, but constant-time comparison is nearly free, so there is no reason to take the risk.
4. Different field lists can produce the same bytes (`("ab","c")` and `("a","bc")`), so one value can be passed off as another. A domain separator names the purpose so values minted for one use cannot be replayed for another.
5. A MAC uses a shared secret: only key holders can create or check tags; fast, but the verifier can also forge. A signature uses a key pair: anyone can verify, only the owner can sign. Use a MAC between two parties who trust each other; signatures when many verify or when the verifier must not be able to forge (blockchains, software updates).
6. Deterministic signing removes dependence on good randomness at signing time. `verify_strict` rejects signatures that verify only because of small-order points; when several implementations must agree on validity, the rule must be fixed in the specification.
7. Is the key known and active in the validity window; is the context right; is the timestamp fresh (and not from the far future); has this nonce been seen; does the payload parse and satisfy application rules; is the signer *authorized* for this action (a valid signature is not permission).
8. A number that must be unique per key. Reusing it with a stream-cipher AEAD leaks the XOR of the plaintexts (✓) and may expose the authentication key, breaking both secrecy and integrity.
9. Encrypt it under a key derived with a slow memory-hard function (Argon2id) and a fresh salt, with an AEAD and a fresh nonce; create the file with 0600 permissions and do not overwrite. The reader must never trust cost parameters or lengths in the file before checking them against limits (✓ §10.3).
10. HKDF derives several keys from an already high-entropy secret and is fast. Argon2 stretches a low-entropy secret (a password) and is intentionally slow and memory-hungry.
11. Register the new key before signers use it, keep both valid for an overlap (deployment time plus message lifetime), then retire the old. For a leak, remove trust immediately everywhere you can, because waiting is not possible; short lifetimes and a revocation channel make this possible.
12. No. It overwrites the value it owns when dropped, but copies made elsewhere (moves, reallocation, swap, dumps, registers) may remain. It is useful hygiene, not a guarantee.

</details>

---

## 18. Practical challenge: `cryptolab`

Build a library crate that:

1. Wraps SHA-256, SHA-512 and BLAKE3 and provides an **unambiguous, domain-separated** `hash_fields`; includes a truncated-hash collision experiment.
2. Provides HMAC-SHA256 (with verification in constant time), a constant-time comparison, a deliberately naive comparison and an insecure `SHA256(key ‖ message)` for demonstration; HKDF-SHA256 with limits and per-purpose subkeys.
3. Contains an educational SHA-256 from scratch with a length-extension helper, verified differentially against the real crate.
4. Provides Ed25519 keys with a redacted `Debug`; password-encrypted key storage (Argon2id + XChaCha20-Poly1305, authenticated header, hostile-parameter caps, safe file creation and permission check).
5. Provides signed messages with a context, key id, timestamp, nonce, replay cache, freshness window, key validity windows and weak-key rejection; time and nonces are injectable.
6. Has known-answer tests from RFCs, the experiments of this lesson, an example that prints the measurements, and passes `cargo fmt`, `cargo clippy --all-targets -- -D warnings` and mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

```toml
[package]
name = "cryptolab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "Worked examples of hashing, MACs, signatures and key handling with vetted crates"

[dependencies]
sha2 = "0.11"
blake3 = "1"
hmac = "0.13"
hkdf = "0.13"
subtle = "2"
zeroize = { version = "1", features = ["derive"] }
ed25519-dalek = "3"
chacha20poly1305 = "0.11"
argon2 = "0.6"
getrandom = "0.4"
```

### `src/lib.rs`

```rust
//! `cryptolab`: worked examples of hashing, message authentication, signatures
//! and key handling, built on vetted crates.
//!
//! **This crate is a teaching aid, not a product.** It has not been audited.
//! The one hand-written primitive, [`toy_sha256`], exists to explain how a
//! hash works and to demonstrate a famous attack; do not use it for real data.
#![deny(missing_docs)]

pub mod digest;
pub mod hex;
pub mod kdf;
pub mod keys;
pub mod mac;
pub mod sign;
pub mod toy_sha256;

/// Fills a fixed-size array from the operating system's random generator.
///
/// # Panics
/// Panics if the system cannot provide randomness, which is not recoverable
/// for key generation.
pub fn random_bytes<const N: usize>() -> [u8; N] {
    let mut out = [0u8; N];
    getrandom::fill(&mut out).expect("the operating system random generator failed");
    out
}
```

### `src/hex.rs`

```rust
//! Strict hexadecimal encoding.

use std::fmt;

/// Why a hex string was rejected.
#[derive(Debug, PartialEq, Eq)]
pub enum HexError {
    /// The string has an odd number of characters.
    OddLength,
    /// A character outside `0-9a-fA-F`, at this character index.
    InvalidChar(usize),
}

impl fmt::Display for HexError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            HexError::OddLength => write!(f, "hex string has an odd length"),
            HexError::InvalidChar(i) => write!(f, "invalid hex character at index {i}"),
        }
    }
}

impl std::error::Error for HexError {}

const DIGITS: &[u8; 16] = b"0123456789abcdef";

/// Encodes bytes as lower-case hex.
pub fn encode(bytes: &[u8]) -> String {
    let mut s = String::with_capacity(bytes.len() * 2);
    for &b in bytes {
        s.push(DIGITS[(b >> 4) as usize] as char);
        s.push(DIGITS[(b & 0x0f) as usize] as char);
    }
    s
}

fn nibble(c: u8) -> Option<u8> {
    match c {
        b'0'..=b'9' => Some(c - b'0'),
        b'a'..=b'f' => Some(c - b'a' + 10),
        b'A'..=b'F' => Some(c - b'A' + 10),
        _ => None,
    }
}

/// Decodes hex (either case). No prefix, no separators, no whitespace.
pub fn decode(text: &str) -> Result<Vec<u8>, HexError> {
    let b = text.as_bytes();
    if b.len() % 2 != 0 {
        return Err(HexError::OddLength);
    }
    let mut out = Vec::with_capacity(b.len() / 2);
    for (i, pair) in b.chunks_exact(2).enumerate() {
        let hi = nibble(pair[0]).ok_or(HexError::InvalidChar(2 * i))?;
        let lo = nibble(pair[1]).ok_or(HexError::InvalidChar(2 * i + 1))?;
        out.push((hi << 4) | lo);
    }
    Ok(out)
}

/// Decodes exactly `N` bytes of hex.
pub fn decode_array<const N: usize>(text: &str) -> Result<[u8; N], HexError> {
    let v = decode(text)?;
    v.try_into().map_err(|_| HexError::OddLength)
}
```

### `src/digest.rs`

```rust
//! Hash functions and unambiguous hashing of structured data.

use sha2::{Digest, Sha256, Sha512};

/// SHA-256 of `data`.
pub fn sha256(data: &[u8]) -> [u8; 32] {
    Sha256::digest(data).into()
}

/// SHA-512 of `data`.
pub fn sha512(data: &[u8]) -> [u8; 64] {
    Sha512::digest(data).into()
}

/// BLAKE3 of `data` (32-byte output).
pub fn blake3(data: &[u8]) -> [u8; 32] {
    *::blake3::hash(data).as_bytes()
}

/// Hashes a list of fields so that **different lists never produce the same
/// byte stream**, and so that different `domain`s never collide.
///
/// Naive concatenation is ambiguous: `H("ab" || "c") == H("a" || "bc")`. Here
/// every field (and the domain label) is preceded by its length as 8 bytes,
/// big-endian, so the encoding can be parsed back uniquely.
pub fn hash_fields(domain: &str, fields: &[&[u8]]) -> [u8; 32] {
    let mut h = Sha256::new();
    h.update((domain.len() as u64).to_be_bytes());
    h.update(domain.as_bytes());
    h.update((fields.len() as u64).to_be_bytes());
    for f in fields {
        h.update((f.len() as u64).to_be_bytes());
        h.update(f);
    }
    h.finalize().into()
}

/// Whether [`truncated_collision`] accepts this many bits.
pub fn truncation_supported(bits: u32) -> bool {
    (1..=48).contains(&bits)
}

/// Finds a collision of the hash truncated to `bits` bits (1..=48) by hashing
/// `0, 1, 2, ...` until two inputs share a truncated digest. Returns the two
/// inputs and how many hashes were computed. An experiment, not a tool: it
/// shows the birthday bound (about `2^(bits/2)` hashes).
pub fn truncated_collision(bits: u32) -> (u64, u64, u64) {
    assert!(truncation_supported(bits), "bits must be 1..=48");
    let mask: u64 = (1u64 << bits) - 1;
    let mut seen = std::collections::HashMap::new();
    let mut i: u64 = 0;
    loop {
        let d = sha256(&i.to_be_bytes());
        let t = u64::from_be_bytes(d[..8].try_into().expect("8 bytes")) & mask;
        if let Some(&j) = seen.get(&t) {
            return (j, i, i + 1);
        }
        seen.insert(t, i);
        i += 1;
    }
}
```

### `src/mac.rs`

```rust
//! Message authentication codes and constant-time comparison.

use hmac::{Hmac, KeyInit, Mac};
use sha2::{Digest, Sha256};
use subtle::ConstantTimeEq;

type HmacSha256 = Hmac<Sha256>;

/// HMAC-SHA256 of `msg` under `key` (any key length).
pub fn hmac_sha256(key: &[u8], msg: &[u8]) -> [u8; 32] {
    let mut m = HmacSha256::new_from_slice(key).expect("HMAC accepts any key length");
    m.update(msg);
    m.finalize().into_bytes().into()
}

/// Verifies an HMAC-SHA256 tag in constant time.
pub fn verify_hmac_sha256(key: &[u8], msg: &[u8], tag: &[u8]) -> bool {
    let mut m = HmacSha256::new_from_slice(key).expect("HMAC accepts any key length");
    m.update(msg);
    m.verify_slice(tag).is_ok()
}

/// Compares two byte strings without stopping at the first difference.
/// Different lengths compare unequal (the length is not secret here).
pub fn ct_eq(a: &[u8], b: &[u8]) -> bool {
    a.ct_eq(b).into()
}

/// The comparison you should **not** use for secrets: it returns at the first
/// differing byte, so its running time reveals how many leading bytes match.
pub fn naive_eq(a: &[u8], b: &[u8]) -> bool {
    if a.len() != b.len() {
        return false;
    }
    for i in 0..a.len() {
        if a[i] != b[i] {
            return false;
        }
    }
    true
}

/// `SHA-256(key || msg)`: a **broken** way to build a MAC from a Merkle–Damgård
/// hash (vulnerable to length extension). Present only to demonstrate the
/// attack and show that HMAC is not affected.
pub fn insecure_prefix_mac(key: &[u8], msg: &[u8]) -> [u8; 32] {
    let mut h = Sha256::new();
    h.update(key);
    h.update(msg);
    h.finalize().into()
}
```

### `src/kdf.rs`

```rust
//! Key derivation.

use hkdf::Hkdf;
use sha2::Sha256;
use zeroize::Zeroizing;

/// The longest output HKDF-SHA256 can produce: 255 blocks of 32 bytes.
pub const MAX_OUTPUT: usize = 255 * 32;

/// HKDF-SHA256 (extract and expand). `salt` may be empty.
/// Returns `None` if `len` is 0 or exceeds [`MAX_OUTPUT`].
pub fn hkdf_sha256(ikm: &[u8], salt: &[u8], info: &[u8], len: usize) -> Option<Zeroizing<Vec<u8>>> {
    if len == 0 || len > MAX_OUTPUT {
        return None;
    }
    let salt = if salt.is_empty() { None } else { Some(salt) };
    let hk = Hkdf::<Sha256>::new(salt, ikm);
    let mut out = Zeroizing::new(vec![0u8; len]);
    hk.expand(info, &mut out).ok()?;
    Some(out)
}

/// Derives an independent 32-byte subkey from a master key for one `purpose`
/// (for example `"mac"` or `"file-encryption"`). The same master key must never
/// be used directly for two different purposes.
pub fn subkey(master: &[u8; 32], purpose: &str) -> Zeroizing<[u8; 32]> {
    let hk = Hkdf::<Sha256>::new(None, master);
    let mut out = Zeroizing::new([0u8; 32]);
    hk.expand(purpose.as_bytes(), &mut *out)
        .expect("32 bytes is a valid HKDF output length");
    out
}
```

### `src/toy_sha256.rs`

```rust
//! A from-scratch SHA-256, written to show how a Merkle–Damgård hash works.
//!
//! **Not for production use**: use the `sha2` crate. This code is tested
//! against `sha2` on every input length around the padding boundaries, but it
//! has no optimizations and no review. It also exposes [`Sha256::from_state`],
//! which is what makes the length-extension attack of [`extend`] possible.

/// The first `N` prime numbers.
const fn primes<const N: usize>() -> [u64; N] {
    let mut out = [0u64; N];
    let (mut n, mut k) = (2u64, 0usize);
    while k < N {
        let mut is_prime = true;
        let mut d = 2u64;
        while d * d <= n {
            if n % d == 0 {
                is_prime = false;
                break;
            }
            d += 1;
        }
        if is_prime {
            out[k] = n;
            k += 1;
        }
        n += 1;
    }
    out
}

/// Largest `r` with `r * r * r <= x` (for `x < 2^105`).
const fn icbrt(x: u128) -> u128 {
    let (mut lo, mut hi) = (0u128, 1u128 << 35);
    while lo < hi {
        let mid = (lo + hi).div_ceil(2);
        if mid * mid * mid <= x {
            lo = mid;
        } else {
            hi = mid - 1;
        }
    }
    lo
}

/// Largest `r` with `r * r <= x` (for `x < 2^72`).
const fn isqrt(x: u128) -> u128 {
    let (mut lo, mut hi) = (0u128, 1u128 << 36);
    while lo < hi {
        let mid = (lo + hi).div_ceil(2);
        if mid * mid <= x {
            lo = mid;
        } else {
            hi = mid - 1;
        }
    }
    lo
}

/// Round constants: the first 32 bits of the fractional parts of the cube
/// roots of the first 64 primes, computed here with exact integer arithmetic.
const K: [u32; 64] = {
    let p = primes::<64>();
    let mut k = [0u32; 64];
    let mut i = 0;
    while i < 64 {
        k[i] = (icbrt((p[i] as u128) << 96) & 0xffff_ffff) as u32;
        i += 1;
    }
    k
};

/// Initial state: the first 32 bits of the fractional parts of the square
/// roots of the first 8 primes.
const H0: [u32; 8] = {
    let p = primes::<8>();
    let mut h = [0u32; 8];
    let mut i = 0;
    while i < 8 {
        h[i] = (isqrt((p[i] as u128) << 64) & 0xffff_ffff) as u32;
        i += 1;
    }
    h
};

fn compress(state: &mut [u32; 8], block: &[u8; 64]) {
    let mut w = [0u32; 64];
    for (i, chunk) in block.chunks_exact(4).enumerate() {
        w[i] = u32::from_be_bytes([chunk[0], chunk[1], chunk[2], chunk[3]]);
    }
    for i in 16..64 {
        let s0 = w[i - 15].rotate_right(7) ^ w[i - 15].rotate_right(18) ^ (w[i - 15] >> 3);
        let s1 = w[i - 2].rotate_right(17) ^ w[i - 2].rotate_right(19) ^ (w[i - 2] >> 10);
        w[i] = w[i - 16]
            .wrapping_add(s0)
            .wrapping_add(w[i - 7])
            .wrapping_add(s1);
    }
    let [mut a, mut b, mut c, mut d, mut e, mut f, mut g, mut h] = *state;
    for i in 0..64 {
        let s1 = e.rotate_right(6) ^ e.rotate_right(11) ^ e.rotate_right(25);
        let ch = (e & f) ^ (!e & g);
        let t1 = h
            .wrapping_add(s1)
            .wrapping_add(ch)
            .wrapping_add(K[i])
            .wrapping_add(w[i]);
        let s0 = a.rotate_right(2) ^ a.rotate_right(13) ^ a.rotate_right(22);
        let maj = (a & b) ^ (a & c) ^ (b & c);
        let t2 = s0.wrapping_add(maj);
        h = g;
        g = f;
        f = e;
        e = d.wrapping_add(t1);
        d = c;
        c = b;
        b = a;
        a = t1.wrapping_add(t2);
    }
    for (s, v) in state.iter_mut().zip([a, b, c, d, e, f, g, h]) {
        *s = s.wrapping_add(v);
    }
}

/// An incremental SHA-256 hasher.
#[derive(Clone)]
pub struct Sha256 {
    state: [u32; 8],
    buf: [u8; 64],
    buf_len: usize,
    total: u64,
}

impl Default for Sha256 {
    fn default() -> Self {
        Self::new()
    }
}

impl Sha256 {
    /// A fresh hasher.
    pub fn new() -> Self {
        Sha256 {
            state: H0,
            buf: [0; 64],
            buf_len: 0,
            total: 0,
        }
    }

    /// Resumes from a known internal `state` as if `processed_len` bytes had
    /// already been hashed. `processed_len` must be a multiple of 64.
    ///
    /// This is exactly what an attacker does in a length-extension attack: a
    /// SHA-256 *digest is* the internal state after the last block.
    pub fn from_state(state: [u32; 8], processed_len: u64) -> Self {
        assert!(
            processed_len % 64 == 0,
            "processed_len must be a whole number of blocks"
        );
        Sha256 {
            state,
            buf: [0; 64],
            buf_len: 0,
            total: processed_len,
        }
    }

    /// Feeds more bytes.
    pub fn update(&mut self, mut data: &[u8]) {
        self.total = self.total.wrapping_add(data.len() as u64);
        if self.buf_len > 0 {
            let take = (64 - self.buf_len).min(data.len());
            self.buf[self.buf_len..self.buf_len + take].copy_from_slice(&data[..take]);
            self.buf_len += take;
            data = &data[take..];
            if self.buf_len == 64 {
                let block = self.buf;
                compress(&mut self.state, &block);
                self.buf_len = 0;
            }
        }
        while data.len() >= 64 {
            let block: [u8; 64] = data[..64].try_into().expect("64 bytes");
            compress(&mut self.state, &block);
            data = &data[64..];
        }
        if !data.is_empty() {
            self.buf[..data.len()].copy_from_slice(data);
            self.buf_len = data.len();
        }
    }

    /// Pads, finishes and returns the digest.
    pub fn finalize(mut self) -> [u8; 32] {
        let pad = padding(self.total);
        let total = self.total;
        self.update(&pad);
        debug_assert_eq!(self.buf_len, 0);
        self.total = total;
        let mut out = [0u8; 32];
        for (chunk, word) in out.chunks_exact_mut(4).zip(self.state) {
            chunk.copy_from_slice(&word.to_be_bytes());
        }
        out
    }
}

/// The padding SHA-256 appends to a message of `msg_len` bytes: a `0x80`
/// byte, zeros up to 8 bytes short of a block boundary, then the message
/// length **in bits** as a 64-bit big-endian number.
pub fn padding(msg_len: u64) -> Vec<u8> {
    let zeros = (55u64.wrapping_sub(msg_len % 64)) % 64;
    let mut p = Vec::with_capacity(1 + zeros as usize + 8);
    p.push(0x80);
    p.extend(std::iter::repeat_n(0u8, zeros as usize));
    p.extend_from_slice(&(msg_len.wrapping_mul(8)).to_be_bytes());
    p
}

/// One-shot SHA-256.
pub fn digest(data: &[u8]) -> [u8; 32] {
    let mut h = Sha256::new();
    h.update(data);
    h.finalize()
}

/// Reads a digest back as the eight state words it came from.
pub fn state_from_digest(d: &[u8; 32]) -> [u32; 8] {
    let mut s = [0u32; 8];
    for (w, chunk) in s.iter_mut().zip(d.chunks_exact(4)) {
        *w = u32::from_be_bytes([chunk[0], chunk[1], chunk[2], chunk[3]]);
    }
    s
}

/// Length extension. Given the digest of an **unknown** message of
/// `unknown_len` bytes (for example `key || msg`) and nothing else, computes
/// the digest of `unknown || padding(unknown_len) || suffix` without knowing
/// the unknown part. Returns `(glue, new_digest)` where `glue` is what must
/// be appended after the original message: `padding(unknown_len) || suffix`.
pub fn extend(known_digest: &[u8; 32], unknown_len: u64, suffix: &[u8]) -> (Vec<u8>, [u8; 32]) {
    let glue_pad = padding(unknown_len);
    let processed = unknown_len + glue_pad.len() as u64;
    let mut h = Sha256::from_state(state_from_digest(known_digest), processed);
    h.update(suffix);
    let mut glue = glue_pad;
    glue.extend_from_slice(suffix);
    (glue, h.finalize())
}
```

### `src/keys.rs`

```rust
//! Key generation, identification and encrypted storage.

use std::fmt;
use std::io::{Read, Write};
use std::path::Path;

use argon2::{Algorithm, Argon2, Params, Version};
use chacha20poly1305::aead::{Aead, Payload};
use chacha20poly1305::{KeyInit, XChaCha20Poly1305};
use ed25519_dalek::{Signature, Signer, SigningKey, VerifyingKey};
use zeroize::Zeroizing;

use crate::digest::sha256;
use crate::hex;
use crate::random_bytes;

/// Length of a [`key_id`].
pub const KEY_ID_LEN: usize = 8;

/// A short identifier for a public key: the first 8 bytes of its SHA-256.
/// Used to say *which* key signed something; it is not a security boundary
/// (8 bytes can collide on purpose, so verification always uses the full key).
pub fn key_id(public: &VerifyingKey) -> [u8; KEY_ID_LEN] {
    let d = sha256(public.as_bytes());
    d[..KEY_ID_LEN].try_into().expect("8 bytes")
}

/// An Ed25519 key pair. The secret half is wiped from memory when this value
/// is dropped (best effort: see the lesson for what that does not cover) and
/// is never printed.
pub struct Keypair {
    signing: SigningKey,
}

impl Keypair {
    /// A new random key pair from the operating system's generator.
    pub fn generate() -> Self {
        Self::from_seed(&random_bytes::<32>())
    }

    /// Rebuilds a key pair from its 32-byte secret seed.
    pub fn from_seed(seed: &[u8; 32]) -> Self {
        Keypair {
            signing: SigningKey::from_bytes(seed),
        }
    }

    /// The secret seed, in a buffer that wipes itself on drop.
    pub fn seed(&self) -> Zeroizing<[u8; 32]> {
        Zeroizing::new(self.signing.to_bytes())
    }

    /// The public key.
    pub fn public(&self) -> VerifyingKey {
        self.signing.verifying_key()
    }

    /// This key's [`key_id`].
    pub fn id(&self) -> [u8; KEY_ID_LEN] {
        key_id(&self.public())
    }

    /// Signs `message` (deterministic: the same input gives the same signature).
    pub fn sign(&self, message: &[u8]) -> Signature {
        self.signing.sign(message)
    }
}

impl fmt::Debug for Keypair {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "Keypair {{ key_id: {}, secret: <redacted> }}",
            hex::encode(&self.id())
        )
    }
}

/// Memory-hard key-derivation cost for protecting a stored key.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct KdfParams {
    /// Memory in KiB.
    pub m_kib: u32,
    /// Number of passes.
    pub t: u32,
}

impl Default for KdfParams {
    fn default() -> Self {
        KdfParams {
            m_kib: 19_456,
            t: 2,
        }
    }
}

/// Largest memory cost [`open_seed`] will accept from a file (256 MiB).
pub const MAX_M_KIB: u32 = 262_144;
/// Largest pass count [`open_seed`] will accept from a file.
pub const MAX_T: u32 = 10;
/// Smallest memory cost accepted (Argon2 requires at least 8 KiB).
pub const MIN_M_KIB: u32 = 8;

/// What can go wrong handling stored keys.
#[derive(Debug)]
pub enum KeyError {
    /// The blob is not in the expected format (wrong magic or length).
    BadFormat,
    /// The format version is not supported.
    UnsupportedVersion(u8),
    /// The stored cost parameters are outside the accepted range.
    ParamsOutOfRange,
    /// An empty passphrase was supplied.
    EmptyPassphrase,
    /// Wrong passphrase, or the data was modified. These cannot be told apart.
    Decrypt,
    /// The key-derivation function failed.
    Kdf,
    /// The key file is readable by group or others (mode shown).
    InsecurePermissions(u32),
    /// An I/O error.
    Io(std::io::Error),
}

impl fmt::Display for KeyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            KeyError::BadFormat => write!(f, "not a key file"),
            KeyError::UnsupportedVersion(v) => write!(f, "unsupported key file version {v}"),
            KeyError::ParamsOutOfRange => write!(f, "key derivation parameters out of range"),
            KeyError::EmptyPassphrase => write!(f, "empty passphrase"),
            KeyError::Decrypt => write!(f, "wrong passphrase or damaged key file"),
            KeyError::Kdf => write!(f, "key derivation failed"),
            KeyError::InsecurePermissions(m) => {
                write!(f, "key file permissions {m:o} allow group or other access")
            }
            KeyError::Io(e) => write!(f, "i/o error: {e}"),
        }
    }
}

impl std::error::Error for KeyError {}

impl From<std::io::Error> for KeyError {
    fn from(e: std::io::Error) -> Self {
        KeyError::Io(e)
    }
}

const MAGIC: &[u8; 4] = b"CLK1";
const VERSION: u8 = 1;
const SALT_LEN: usize = 16;
const NONCE_LEN: usize = 24;
/// Bytes before the ciphertext: magic, version, m, t, salt, nonce.
pub const HEADER_LEN: usize = 4 + 1 + 4 + 4 + SALT_LEN + NONCE_LEN;
/// Total size of a sealed 32-byte seed: header + 32 bytes + 16-byte tag.
pub const SEALED_LEN: usize = HEADER_LEN + 32 + 16;

fn derive(
    passphrase: &[u8],
    salt: &[u8],
    m_kib: u32,
    t: u32,
) -> Result<Zeroizing<[u8; 32]>, KeyError> {
    let params = Params::new(m_kib, t, 1, Some(32)).map_err(|_| KeyError::ParamsOutOfRange)?;
    let argon = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);
    let mut key = Zeroizing::new([0u8; 32]);
    argon
        .hash_password_into(passphrase, salt, &mut *key)
        .map_err(|_| KeyError::Kdf)?;
    Ok(key)
}

fn cipher(key: &[u8; 32]) -> XChaCha20Poly1305 {
    XChaCha20Poly1305::new(key.into())
}

/// Encrypts a 32-byte seed under a passphrase.
///
/// Layout: `"CLK1" | version | m_kib | t | salt(16) | nonce(24) | ciphertext+tag`.
/// The whole header is authenticated as associated data, so changing the cost
/// parameters or the salt makes decryption fail.
pub fn seal_seed(
    seed: &[u8; 32],
    passphrase: &[u8],
    params: KdfParams,
) -> Result<Vec<u8>, KeyError> {
    seal_with(
        seed,
        passphrase,
        params,
        random_bytes::<SALT_LEN>(),
        random_bytes::<NONCE_LEN>(),
    )
}

/// [`seal_seed`] with a caller-chosen salt and nonce (for tests and vectors).
/// **Never reuse a nonce with the same key.**
pub fn seal_with(
    seed: &[u8; 32],
    passphrase: &[u8],
    params: KdfParams,
    salt: [u8; SALT_LEN],
    nonce: [u8; NONCE_LEN],
) -> Result<Vec<u8>, KeyError> {
    if passphrase.is_empty() {
        return Err(KeyError::EmptyPassphrase);
    }
    if !(MIN_M_KIB..=MAX_M_KIB).contains(&params.m_kib) || !(1..=MAX_T).contains(&params.t) {
        return Err(KeyError::ParamsOutOfRange);
    }
    let mut out = Vec::with_capacity(SEALED_LEN);
    out.extend_from_slice(MAGIC);
    out.push(VERSION);
    out.extend_from_slice(&params.m_kib.to_be_bytes());
    out.extend_from_slice(&params.t.to_be_bytes());
    out.extend_from_slice(&salt);
    out.extend_from_slice(&nonce);
    let key = derive(passphrase, &salt, params.m_kib, params.t)?;
    let ct = cipher(&key)
        .encrypt(
            (&nonce).into(),
            Payload {
                msg: seed,
                aad: &out,
            },
        )
        .map_err(|_| KeyError::Decrypt)?;
    out.extend_from_slice(&ct);
    Ok(out)
}

/// Decrypts a blob made by [`seal_seed`].
///
/// The cost parameters come from the (untrusted) file, so they are range
/// checked **before** any expensive work: a hostile file must not be able to
/// make us allocate gigabytes.
pub fn open_seed(blob: &[u8], passphrase: &[u8]) -> Result<Zeroizing<[u8; 32]>, KeyError> {
    if blob.len() != SEALED_LEN || &blob[..4] != MAGIC {
        return Err(KeyError::BadFormat);
    }
    if blob[4] != VERSION {
        return Err(KeyError::UnsupportedVersion(blob[4]));
    }
    if passphrase.is_empty() {
        return Err(KeyError::EmptyPassphrase);
    }
    let m = u32::from_be_bytes(blob[5..9].try_into().expect("4 bytes"));
    let t = u32::from_be_bytes(blob[9..13].try_into().expect("4 bytes"));
    if !(MIN_M_KIB..=MAX_M_KIB).contains(&m) || !(1..=MAX_T).contains(&t) {
        return Err(KeyError::ParamsOutOfRange);
    }
    let salt = &blob[13..13 + SALT_LEN];
    let nonce: [u8; NONCE_LEN] = blob[13 + SALT_LEN..HEADER_LEN]
        .try_into()
        .expect("24 bytes");
    let key = derive(passphrase, salt, m, t)?;
    let pt = cipher(&key)
        .decrypt(
            (&nonce).into(),
            Payload {
                msg: &blob[HEADER_LEN..],
                aad: &blob[..HEADER_LEN],
            },
        )
        .map_err(|_| KeyError::Decrypt)?;
    let pt = Zeroizing::new(pt);
    let seed: [u8; 32] = pt[..].try_into().map_err(|_| KeyError::BadFormat)?;
    Ok(Zeroizing::new(seed))
}

/// Writes a key file that did not exist, readable only by its owner (0600).
/// Fails if the file already exists, so a key is never silently overwritten.
#[cfg(unix)]
pub fn write_key_file(path: &Path, blob: &[u8]) -> Result<(), KeyError> {
    use std::os::unix::fs::OpenOptionsExt;
    let mut f = std::fs::OpenOptions::new()
        .write(true)
        .create_new(true)
        .mode(0o600)
        .open(path)?;
    f.write_all(blob)?;
    f.sync_all()?;
    Ok(())
}

/// Reads a key file, refusing one that group or others can access.
#[cfg(unix)]
pub fn read_key_file(path: &Path) -> Result<Vec<u8>, KeyError> {
    use std::os::unix::fs::PermissionsExt;
    let mut f = std::fs::File::open(path)?;
    let mode = f.metadata()?.permissions().mode() & 0o777;
    if mode & 0o077 != 0 {
        return Err(KeyError::InsecurePermissions(mode));
    }
    let mut v = Vec::new();
    f.read_to_end(&mut v)?;
    Ok(v)
}
```

### `src/sign.rs`

```rust
//! Signed messages with domain separation, freshness, replay protection and
//! key rotation windows.

use std::collections::{HashSet, VecDeque};

use ed25519_dalek::{Signature, VerifyingKey};

use crate::keys::{KEY_ID_LEN, Keypair, key_id};
use crate::random_bytes;

/// Length of the per-message random nonce.
pub const NONCE_LEN: usize = 16;
/// Protocol label mixed into everything that is signed.
pub const PROTOCOL: &str = "cryptolab/signed-message/v1";

/// A message together with the data needed to verify it.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct SignedMessage {
    /// Which key signed (see [`key_id`]).
    pub key_id: [u8; KEY_ID_LEN],
    /// Signing time, seconds since the Unix epoch.
    pub timestamp: u64,
    /// Random value that makes each message unique.
    pub nonce: [u8; NONCE_LEN],
    /// The signed content.
    pub payload: Vec<u8>,
    /// Ed25519 signature over [`signing_input`].
    pub signature: [u8; 64],
}

fn put(out: &mut Vec<u8>, bytes: &[u8]) {
    out.extend_from_slice(&(bytes.len() as u64).to_be_bytes());
    out.extend_from_slice(bytes);
}

/// The exact bytes that are signed. Every component is length-prefixed so no
/// two different `(context, id, time, nonce, payload)` tuples encode alike,
/// and `context` (an application label such as `"login-challenge"`) keeps a
/// signature made for one purpose from being accepted for another.
pub fn signing_input(
    context: &str,
    key_id: &[u8; KEY_ID_LEN],
    timestamp: u64,
    nonce: &[u8; NONCE_LEN],
    payload: &[u8],
) -> Vec<u8> {
    let mut out = Vec::with_capacity(payload.len() + 96);
    put(&mut out, PROTOCOL.as_bytes());
    put(&mut out, context.as_bytes());
    put(&mut out, key_id);
    put(&mut out, &timestamp.to_be_bytes());
    put(&mut out, nonce);
    put(&mut out, payload);
    out
}

/// Signs `payload` with a caller-supplied nonce (for tests and vectors).
pub fn sign_with_nonce(
    kp: &Keypair,
    context: &str,
    payload: &[u8],
    now: u64,
    nonce: [u8; NONCE_LEN],
) -> SignedMessage {
    let id = kp.id();
    let sig = kp.sign(&signing_input(context, &id, now, &nonce, payload));
    SignedMessage {
        key_id: id,
        timestamp: now,
        nonce,
        payload: payload.to_vec(),
        signature: sig.to_bytes(),
    }
}

/// Signs `payload` with a fresh random nonce.
pub fn sign(kp: &Keypair, context: &str, payload: &[u8], now: u64) -> SignedMessage {
    sign_with_nonce(kp, context, payload, now, random_bytes::<NONCE_LEN>())
}

/// Why verification failed. Callers should treat all of these as "reject".
#[derive(Debug, PartialEq, Eq)]
pub enum VerifyError {
    /// No trusted key has this id.
    UnknownKey,
    /// The key exists but is not valid at this time (not yet active, or retired).
    KeyNotActive,
    /// The key is a small-order ("weak") point and cannot be trusted.
    WeakKey,
    /// The message is older than the allowed age.
    Expired,
    /// The message claims a time too far in the future.
    FromFuture,
    /// The signature does not match the message and context.
    BadSignature,
    /// This nonce was already used within the freshness window.
    Replay,
    /// The replay cache is full of still-valid entries; try again later.
    CacheFull,
}

struct KeyEntry {
    id: [u8; KEY_ID_LEN],
    key: VerifyingKey,
    not_before: u64,
    not_after: u64,
}

/// Verification policy and state: trusted keys with validity windows (which is
/// how rotation works: the new key becomes valid before the old one retires),
/// freshness limits, and a bounded memory of recent nonces.
pub struct Verifier {
    keys: Vec<KeyEntry>,
    max_age: u64,
    max_skew: u64,
    capacity: usize,
    seen: HashSet<[u8; NONCE_LEN]>,
    order: VecDeque<(u64, [u8; NONCE_LEN])>,
}

impl Verifier {
    /// `max_age`: how old a message may be (seconds). `max_skew`: how far in
    /// the future a timestamp may be, to tolerate clock differences.
    /// `capacity`: how many recent nonces to remember.
    pub fn new(max_age: u64, max_skew: u64, capacity: usize) -> Self {
        Verifier {
            keys: Vec::new(),
            max_age,
            max_skew,
            capacity,
            seen: HashSet::new(),
            order: VecDeque::new(),
        }
    }

    /// Trusts `key` for times in `not_before..not_after`.
    /// Returns an error for weak keys, empty windows and duplicate ids.
    pub fn trust(
        &mut self,
        key: VerifyingKey,
        not_before: u64,
        not_after: u64,
    ) -> Result<(), VerifyError> {
        if key.is_weak() {
            return Err(VerifyError::WeakKey);
        }
        let id = key_id(&key);
        if not_before >= not_after || self.keys.iter().any(|k| k.id == id) {
            return Err(VerifyError::UnknownKey);
        }
        self.keys.push(KeyEntry {
            id,
            key,
            not_before,
            not_after,
        });
        Ok(())
    }

    fn prune(&mut self, now: u64) {
        while let Some(&(ts, nonce)) = self.order.front() {
            if ts.saturating_add(self.max_age) < now {
                self.order.pop_front();
                self.seen.remove(&nonce);
            } else {
                break;
            }
        }
    }

    /// Verifies `msg` for `context` at time `now`. On success the nonce is
    /// remembered, so the same message cannot be accepted twice.
    ///
    /// The order of checks matters: cheap, unauthenticated checks first, the
    /// signature next, and the replay cache is touched **only after** a valid
    /// signature, so forged messages cannot fill it.
    pub fn verify(
        &mut self,
        context: &str,
        msg: &SignedMessage,
        now: u64,
    ) -> Result<(), VerifyError> {
        let entry = self
            .keys
            .iter()
            .find(|k| k.id == msg.key_id)
            .ok_or(VerifyError::UnknownKey)?;
        if now < entry.not_before || now >= entry.not_after {
            return Err(VerifyError::KeyNotActive);
        }
        if msg.timestamp > now.saturating_add(self.max_skew) {
            return Err(VerifyError::FromFuture);
        }
        if now.saturating_sub(msg.timestamp) > self.max_age {
            return Err(VerifyError::Expired);
        }
        let input = signing_input(
            context,
            &msg.key_id,
            msg.timestamp,
            &msg.nonce,
            &msg.payload,
        );
        let sig = Signature::from_bytes(&msg.signature);
        entry
            .key
            .verify_strict(&input, &sig)
            .map_err(|_| VerifyError::BadSignature)?;
        self.prune(now);
        if self.seen.contains(&msg.nonce) {
            return Err(VerifyError::Replay);
        }
        if self.seen.len() >= self.capacity {
            return Err(VerifyError::CacheFull);
        }
        self.seen.insert(msg.nonce);
        self.order.push_back((msg.timestamp, msg.nonce));
        Ok(())
    }

    /// How many nonces are currently remembered.
    pub fn remembered(&self) -> usize {
        self.seen.len()
    }
}
```

### `tests/vectors.rs`

```rust
//! Known-answer tests. The expected values come from published test vectors
//! (RFC 4231, RFC 5869, RFC 8032, FIPS 180) and were re-computed with Python's
//! `hashlib`, `hmac`, `blake3` and `cryptography` packages.

use cryptolab::digest::{blake3, sha256, sha512};
use cryptolab::hex;
use cryptolab::kdf::hkdf_sha256;
use cryptolab::keys::Keypair;
use cryptolab::mac::hmac_sha256;
use cryptolab::toy_sha256;

fn h(s: &str) -> Vec<u8> {
    hex::decode(s).unwrap()
}

#[test]
fn sha256_known_answers() {
    let cases: [(&[u8], &str); 3] = [
        (
            b"",
            "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
        ),
        (
            b"abc",
            "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad",
        ),
        (
            b"abcdbcdecdefdefgefghfghighijhijkijkljklmklmnlmnomnopnopq",
            "248d6a61d20638b8e5c026930c3e6039a33ce45964ff2167f6ecedd419db06c1",
        ),
    ];
    for (input, want) in cases {
        assert_eq!(hex::encode(&sha256(input)), want);
        assert_eq!(
            hex::encode(&toy_sha256::digest(input)),
            want,
            "toy implementation"
        );
    }
    let million = vec![b'a'; 1_000_000];
    let want = "cdc76e5c9914fb9281a1c7e284d73e67f1809a48a497200e046d39ccc7112cd0";
    assert_eq!(hex::encode(&sha256(&million)), want);
    assert_eq!(hex::encode(&toy_sha256::digest(&million)), want);
}

#[test]
fn sha512_and_blake3_known_answers() {
    assert_eq!(
        hex::encode(&sha512(b"abc")),
        "ddaf35a193617abacc417349ae20413112e6fa4e89a97ea20a9eeee64b55d39a\
         2192992a274fc1a836ba3c23a3feebbd454d4423643ce80e2a9ac94fa54ca49f"
    );
    assert_eq!(
        hex::encode(&blake3(b"")),
        "af1349b9f5f9a1a6a0404dea36dcc9499bcb25c9adc112b7cc9a93cae41f3262"
    );
    assert_eq!(
        hex::encode(&blake3(b"abc")),
        "6437b3ac38465133ffb63b75273a8db548c558465d79db03fd359c6cd5bd9d85"
    );
}

#[test]
fn hmac_sha256_rfc4231_cases() {
    // Case 1
    assert_eq!(
        hex::encode(&hmac_sha256(&[0x0b; 20], b"Hi There")),
        "b0344c61d8db38535ca8afceaf0bf12b881dc200c9833da726e9376c2e32cff7"
    );
    // Case 2
    assert_eq!(
        hex::encode(&hmac_sha256(b"Jefe", b"what do ya want for nothing?")),
        "5bdcc146bf60754e6a042426089575c75a003f089d2739839dec58b964ec3843"
    );
    // Case 6: a key longer than the 64-byte block is hashed first
    assert_eq!(
        hex::encode(&hmac_sha256(
            &[0xaa; 131],
            b"Test Using Larger Than Block-Size Key - Hash Key First"
        )),
        "60e431591ee0b67f0d8a26aacbf5b77f8e0bc6213728c5140546040f0ee37f54"
    );
}

#[test]
fn hkdf_rfc5869_case_1() {
    let okm = hkdf_sha256(
        &[0x0b; 22],
        &h("000102030405060708090a0b0c"),
        &h("f0f1f2f3f4f5f6f7f8f9"),
        42,
    )
    .unwrap();
    assert_eq!(
        hex::encode(&okm),
        "3cb25f25faacd57a90434f64d0362f2a2d2d0a90cf1a5a4c5db02d56ecc4c5bf34007208d5b887185865"
    );
}

#[test]
fn ed25519_rfc8032_test_vectors() {
    let cases = [
        (
            "9d61b19deffd5a60ba844af492ec2cc44449c5697b326919703bac031cae7f60",
            "d75a980182b10ab7d54bfed3c964073a0ee172f3daa62325af021a68f707511a",
            "",
            "e5564300c360ac729086e2cc806e828a84877f1eb8e5d974d873e06522490155\
             5fb8821590a33bacc61e39701cf9b46bd25bf5f0595bbe24655141438e7a100b",
        ),
        (
            "4ccd089b28ff96da9db6c346ec114e0f5b8a319f35aba624da8cf6ed4fb8a6fb",
            "3d4017c3e843895a92b70aa74d1b7ebc9c982ccf2ec4968cc0cd55f12af4660c",
            "72",
            "92a009a9f0d4cab8720e820b5f642540a2b27b5416503f8fb3762223ebdb69da\
             085ac1e43e15996e458f3613d0f11d8c387b2eaeb4302aeeb00d291612bb0c00",
        ),
    ];
    for (seed, public, msg, sig) in cases {
        let kp = Keypair::from_seed(&hex::decode_array::<32>(seed).unwrap());
        assert_eq!(hex::encode(kp.public().as_bytes()), public);
        let m = h(msg);
        assert_eq!(
            hex::encode(&kp.sign(&m).to_bytes()),
            sig,
            "signatures are deterministic"
        );
        let sig = ed25519_dalek::Signature::from_bytes(&hex::decode_array::<64>(sig).unwrap());
        assert!(kp.public().verify_strict(&m, &sig).is_ok());
        let mut other = m.clone();
        other.push(0);
        assert!(kp.public().verify_strict(&other, &sig).is_err());
    }
}

#[test]
fn key_ids_are_the_first_eight_bytes_of_the_public_key_hash() {
    // RFC 8032 test 1 public key; SHA-256 of it computed with Python's hashlib.
    let kp = Keypair::from_seed(
        &hex::decode_array::<32>(
            "9d61b19deffd5a60ba844af492ec2cc44449c5697b326919703bac031cae7f60",
        )
        .unwrap(),
    );
    assert_eq!(hex::encode(&kp.id()), "21fe31dfa154a261");
}
```

### `tests/toy_sha256.rs`

```rust
use cryptolab::mac::{hmac_sha256, insecure_prefix_mac, verify_hmac_sha256};
use cryptolab::{digest, toy_sha256};

fn pattern(n: usize) -> Vec<u8> {
    (0..n).map(|i| (i * 31 + 7) as u8).collect()
}

#[test]
fn matches_the_sha2_crate_for_every_length_up_to_300() {
    for n in 0..=300 {
        let data = pattern(n);
        assert_eq!(
            toy_sha256::digest(&data),
            digest::sha256(&data),
            "length {n}"
        );
    }
}

#[test]
fn matches_the_sha2_crate_around_block_and_padding_boundaries() {
    for n in [
        55, 56, 57, 63, 64, 65, 119, 120, 121, 127, 128, 129, 1023, 1024, 4096, 65_537,
    ] {
        let data = pattern(n);
        assert_eq!(
            toy_sha256::digest(&data),
            digest::sha256(&data),
            "length {n}"
        );
    }
}

#[test]
fn feeding_data_in_pieces_changes_nothing() {
    let data = pattern(700);
    let want = digest::sha256(&data);
    for step in [1, 2, 3, 7, 31, 63, 64, 65, 100, 699, 700] {
        let mut h = toy_sha256::Sha256::new();
        for chunk in data.chunks(step) {
            h.update(chunk);
        }
        assert_eq!(h.finalize(), want, "chunk size {step}");
    }
    let mut h = toy_sha256::Sha256::new();
    h.update(&[]);
    h.update(&data[..10]);
    h.update(&[]);
    h.update(&data[10..]);
    assert_eq!(h.finalize(), want);
}

#[test]
fn padding_fills_to_a_block_boundary_and_ends_with_the_bit_length() {
    for n in 0u64..400 {
        let p = toy_sha256::padding(n);
        assert_eq!((n + p.len() as u64) % 64, 0, "length {n}");
        assert_eq!(p[0], 0x80);
        assert!(p[1..p.len() - 8].iter().all(|&b| b == 0));
        assert_eq!(&p[p.len() - 8..], &(n * 8).to_be_bytes());
        assert!(p.len() >= 9 && p.len() <= 72, "length {n}: {}", p.len());
    }
    assert_eq!(toy_sha256::padding(0).len(), 64);
    assert_eq!(toy_sha256::padding(55).len(), 9);
    assert_eq!(toy_sha256::padding(56).len(), 72);
    assert_eq!(toy_sha256::padding(64).len(), 64);
}

#[test]
#[should_panic(expected = "whole number of blocks")]
fn resuming_requires_a_whole_number_of_blocks() {
    let _ = toy_sha256::Sha256::from_state([0; 8], 63);
}

#[test]
#[should_panic(expected = "whole number of blocks")]
fn resuming_rejects_half_a_block_too() {
    let _ = toy_sha256::Sha256::from_state([0; 8], 32);
}

#[test]
fn resuming_accepts_whole_blocks() {
    let _ = toy_sha256::Sha256::from_state([0; 8], 0);
    let _ = toy_sha256::Sha256::from_state([0; 8], 64);
    let _ = toy_sha256::Sha256::from_state([0; 8], 128);
}

#[test]
fn length_extension_forges_a_tag_for_sha256_key_concat_message() {
    let key = b"a secret the attacker never sees";
    let msg = b"amount=100&to=alice";
    let tag = insecure_prefix_mac(key, msg); // all the attacker sees: msg and tag
    let suffix = b"&to=mallory";

    // The attacker only needs the *length* of the secret (or guesses it).
    let (glue, forged_tag) = toy_sha256::extend(&tag, (key.len() + msg.len()) as u64, suffix);
    let mut forged_msg = msg.to_vec();
    forged_msg.extend_from_slice(&glue);

    // The server recomputes the "MAC" over the forged message with the real key.
    assert_eq!(
        insecure_prefix_mac(key, &forged_msg),
        forged_tag,
        "forgery accepted"
    );
    assert!(forged_msg.starts_with(msg) && forged_msg.ends_with(suffix));
}

#[test]
fn the_forgery_needs_the_right_secret_length() {
    let key = b"0123456789";
    let msg = b"hello";
    let tag = insecure_prefix_mac(key, msg);
    for wrong_len in [0usize, 5, 9, 11, 20] {
        let (glue, forged) = toy_sha256::extend(&tag, (wrong_len + msg.len()) as u64, b"!");
        let mut m = msg.to_vec();
        m.extend_from_slice(&glue);
        assert_ne!(insecure_prefix_mac(key, &m), forged, "guess {wrong_len}");
    }
}

#[test]
fn hmac_is_not_vulnerable_to_the_same_attack() {
    let key = b"a secret the attacker never sees";
    let msg = b"amount=100&to=alice";
    let tag = hmac_sha256(key, msg);
    let (glue, forged) = toy_sha256::extend(&tag, (key.len() + msg.len()) as u64, b"&to=mallory");
    let mut m = msg.to_vec();
    m.extend_from_slice(&glue);
    assert!(!verify_hmac_sha256(key, &m, &forged));
}
```

### `tests/hashing.rs`

```rust
use cryptolab::digest::{hash_fields, sha256, truncated_collision, truncation_supported};
use cryptolab::hex::{self, HexError};
use cryptolab::kdf::{MAX_OUTPUT, hkdf_sha256, subkey};
use cryptolab::mac::{ct_eq, hmac_sha256, naive_eq, verify_hmac_sha256};

// ---------- hex ----------

#[test]
fn hex_round_trips_every_byte_value() {
    let all: Vec<u8> = (0..=255).collect();
    let text = hex::encode(&all);
    assert_eq!(text.len(), 512);
    assert!(text.starts_with("000102") && text.ends_with("fdfeff"));
    assert_eq!(hex::decode(&text).unwrap(), all);
    assert_eq!(
        hex::decode(&text.to_uppercase()).unwrap(),
        all,
        "either case is accepted"
    );
    assert_eq!(hex::encode(&[]), "");
    assert_eq!(hex::decode("").unwrap(), Vec::<u8>::new());
}

#[test]
fn hex_decode_rejects_bad_input_with_a_position() {
    assert_eq!(hex::decode("abc"), Err(HexError::OddLength));
    assert_eq!(hex::decode("0x00"), Err(HexError::InvalidChar(1)));
    assert_eq!(hex::decode("zz"), Err(HexError::InvalidChar(0)));
    assert_eq!(hex::decode("0g"), Err(HexError::InvalidChar(1)));
    assert_eq!(hex::decode("00  11"), Err(HexError::InvalidChar(2)));
    assert_eq!(
        hex::decode("00 11"),
        Err(HexError::OddLength),
        "odd length is reported first"
    );
    assert_eq!(hex::decode("0011\n"), Err(HexError::OddLength));
    assert_eq!(
        hex::decode("é0"),
        Err(HexError::OddLength),
        "length counts bytes: é is two"
    );
    assert_eq!(hex::decode("éé"), Err(HexError::InvalidChar(0)));
    assert!(hex::decode("00ff").is_ok());
    assert!(hex::decode_array::<2>("00ff").is_ok());
    assert!(hex::decode_array::<2>("00ff00").is_err());
    assert!(hex::decode_array::<2>("00").is_err());
    assert_eq!(
        HexError::InvalidChar(3).to_string(),
        "invalid hex character at index 3"
    );
}

// ---------- unambiguous hashing ----------

#[test]
fn plain_concatenation_is_ambiguous_but_hash_fields_is_not() {
    let naive = |a: &[u8], b: &[u8]| sha256(&[a, b].concat());
    assert_eq!(naive(b"ab", b"c"), naive(b"a", b"bc"), "the ambiguity");
    assert_ne!(
        hash_fields("d", &[b"ab", b"c"]),
        hash_fields("d", &[b"a", b"bc"]),
        "length prefixes remove it"
    );
}

#[test]
fn hash_fields_separates_domains_counts_and_empty_fields() {
    let f: [&[u8]; 2] = [b"x", b"y"];
    assert_eq!(hash_fields("d", &f), hash_fields("d", &f), "deterministic");
    assert_ne!(hash_fields("d1", &f), hash_fields("d2", &f), "domains");
    assert_ne!(
        hash_fields("d", &[b"a", b""]),
        hash_fields("d", &[b"a"]),
        "trailing empty field"
    );
    assert_ne!(
        hash_fields("d", &[b"", b"a"]),
        hash_fields("d", &[b"a", b""]),
        "order"
    );
    assert_ne!(
        hash_fields("d", &[]),
        hash_fields("d", &[b""]),
        "no fields vs one empty field"
    );
    assert_ne!(
        hash_fields("ab", &[b"c"]),
        hash_fields("a", &[b"bc"]),
        "domain/field boundary"
    );
}

#[test]
fn truncated_hashes_collide_after_about_the_birthday_bound() {
    for bits in [8u32, 12, 16, 20] {
        let (a, b, hashes) = truncated_collision(bits);
        assert_ne!(a, b, "two different inputs");
        let mask = (1u64 << bits) - 1;
        let t =
            |x: u64| u64::from_be_bytes(sha256(&x.to_be_bytes())[..8].try_into().unwrap()) & mask;
        assert_eq!(t(a), t(b), "the truncated digests are equal");
        let bound = 1u64 << (bits / 2);
        assert!(hashes >= 2, "{bits} bits: {hashes}");
        assert!(
            hashes <= 40 * bound,
            "{bits} bits: {hashes} hashes vs bound {bound}"
        );
    }
}

#[test]
#[should_panic(expected = "bits must be 1..=48")]
fn truncated_collision_rejects_zero_bits() {
    let _ = truncated_collision(0);
}

#[test]
#[should_panic(expected = "bits must be 1..=48")]
fn truncated_collision_rejects_more_than_48_bits() {
    let _ = truncated_collision(49);
}

// ---------- MAC and comparison ----------

#[test]
fn hmac_verifies_only_the_exact_tag() {
    let key = b"k";
    let msg = b"message";
    let tag = hmac_sha256(key, msg);
    assert!(verify_hmac_sha256(key, msg, &tag));
    for i in 0..32 {
        for bit in 0..8 {
            let mut bad = tag;
            bad[i] ^= 1 << bit;
            assert!(!verify_hmac_sha256(key, msg, &bad), "byte {i} bit {bit}");
        }
    }
    assert!(!verify_hmac_sha256(key, msg, &tag[..31]), "truncated tag");
    assert!(
        !verify_hmac_sha256(key, msg, &[tag.as_slice(), &[0]].concat()),
        "extended tag"
    );
    assert!(!verify_hmac_sha256(key, msg, &[]), "empty tag");
    assert!(!verify_hmac_sha256(b"K", msg, &tag), "other key");
    assert!(!verify_hmac_sha256(key, b"messagE", &tag), "other message");
}

#[test]
fn hmac_key_and_message_are_not_interchangeable() {
    assert_ne!(hmac_sha256(b"ab", b"c"), hmac_sha256(b"a", b"bc"));
    assert_ne!(hmac_sha256(b"", b"x"), hmac_sha256(b"x", b""));
    assert_eq!(hmac_sha256(b"", b"x"), hmac_sha256(b"", b"x"));
}

#[test]
fn comparison_functions_agree_on_equality() {
    let cases: [(&[u8], &[u8], bool); 7] = [
        (b"", b"", true),
        (b"a", b"a", true),
        (b"a", b"b", false),
        (b"ab", b"a", false),
        (b"a", b"ab", false),
        (b"abc", b"abd", false),
        (b"xbc", b"abc", false),
    ];
    for (a, b, want) in cases {
        assert_eq!(ct_eq(a, b), want, "ct_eq {a:?} {b:?}");
        assert_eq!(naive_eq(a, b), want, "naive_eq {a:?} {b:?}");
    }
}

// ---------- key derivation ----------

#[test]
fn hkdf_limits_and_prefix_property() {
    assert!(hkdf_sha256(b"ikm", b"salt", b"info", 0).is_none());
    assert!(hkdf_sha256(b"ikm", b"salt", b"info", 1).is_some());
    assert_eq!(
        hkdf_sha256(b"ikm", b"salt", b"info", MAX_OUTPUT)
            .unwrap()
            .len(),
        MAX_OUTPUT
    );
    assert!(hkdf_sha256(b"ikm", b"salt", b"info", MAX_OUTPUT + 1).is_none());
    assert_eq!(MAX_OUTPUT, 8160);
    let long = hkdf_sha256(b"ikm", b"salt", b"info", 100).unwrap();
    let short = hkdf_sha256(b"ikm", b"salt", b"info", 40).unwrap();
    assert_eq!(
        &long[..40],
        &short[..],
        "shorter output is a prefix of a longer one"
    );
    let other_info = hkdf_sha256(b"ikm", b"salt", b"other", 40).unwrap();
    let other_salt = hkdf_sha256(b"ikm", b"pepper", b"info", 40).unwrap();
    let other_ikm = hkdf_sha256(b"ikM", b"salt", b"info", 40).unwrap();
    assert_ne!(&short[..], &other_info[..]);
    assert_ne!(&short[..], &other_salt[..]);
    assert_ne!(&short[..], &other_ikm[..]);
    let no_salt = hkdf_sha256(b"ikm", b"", b"info", 40).unwrap();
    assert_ne!(&short[..], &no_salt[..]);
}

#[test]
fn subkeys_are_deterministic_and_independent_per_purpose() {
    let master = [7u8; 32];
    let mac = subkey(&master, "mac");
    let enc = subkey(&master, "file-encryption");
    assert_eq!(*mac, *subkey(&master, "mac"));
    assert_ne!(*mac, *enc);
    assert_ne!(*mac, master);
    assert_ne!(*mac, *subkey(&[8u8; 32], "mac"));
    assert_ne!(
        *subkey(&master, "mac"),
        *subkey(&master, "mac "),
        "no trimming"
    );
}

#[test]
fn the_supported_truncation_range_is_exact() {
    assert!(!truncation_supported(0) && truncation_supported(1));
    assert!(truncation_supported(48) && !truncation_supported(49));
}

#[test]
fn hash_fields_domain_length_prefix_blocks_a_boundary_shift_collision() {
    // Without the domain length prefix, a domain that embeds the rest of an
    // encoding could make two different calls hash the same byte string.
    let mut f = Vec::new();
    f.extend_from_slice(&1u64.to_be_bytes());
    f.extend_from_slice(&0u64.to_be_bytes());
    let mut dom = String::from("A");
    dom.push_str(std::str::from_utf8(&1u64.to_be_bytes()).unwrap());
    dom.push_str(std::str::from_utf8(&16u64.to_be_bytes()).unwrap());
    let a = hash_fields("A", &[&f]);
    let b = hash_fields(&dom, &[&[]]);
    assert_ne!(a, b);
}

#[test]
fn truncated_collision_reports_the_number_of_hashes_it_computed() {
    for bits in [8, 12, 16] {
        let (a, b, n) = truncated_collision(bits);
        assert!(a < b);
        assert_eq!(n, b + 1, "inputs 0..=b were hashed, so b + 1 hashes");
    }
}
```

### `tests/sign.rs`

```rust
use cryptolab::keys::{Keypair, key_id};
use cryptolab::sign::{SignedMessage, Verifier, VerifyError, sign_with_nonce, signing_input};

const CTX: &str = "login-challenge";
const T0: u64 = 1_000_000;

fn kp(n: u8) -> Keypair {
    Keypair::from_seed(&[n; 32])
}

fn verifier(max_age: u64, max_skew: u64, cap: usize) -> Verifier {
    let mut v = Verifier::new(max_age, max_skew, cap);
    v.trust(kp(1).public(), 0, u64::MAX).unwrap();
    v
}

fn msg(nonce: u8, ts: u64) -> SignedMessage {
    sign_with_nonce(&kp(1), CTX, b"payload", ts, [nonce; 16])
}

#[test]
fn a_valid_message_verifies_once() {
    let mut v = verifier(60, 5, 100);
    let m = msg(1, T0);
    assert_eq!(v.verify(CTX, &m, T0), Ok(()));
    assert_eq!(
        v.verify(CTX, &m, T0),
        Err(VerifyError::Replay),
        "same message again"
    );
    assert_eq!(v.verify(CTX, &msg(2, T0), T0), Ok(()), "new nonce is fine");
}

#[test]
fn signing_is_deterministic_for_a_fixed_nonce() {
    assert_eq!(msg(5, T0), msg(5, T0));
    assert_ne!(msg(5, T0).signature, msg(6, T0).signature);
}

#[test]
fn every_field_is_covered_by_the_signature() {
    let mut v = verifier(60, 5, 100);
    let good = msg(1, T0);

    let mut m = good.clone();
    m.payload.push(b'!');
    assert_eq!(
        v.verify(CTX, &m, T0),
        Err(VerifyError::BadSignature),
        "payload"
    );

    let mut m = good.clone();
    m.timestamp += 1;
    assert_eq!(
        v.verify(CTX, &m, T0 + 1),
        Err(VerifyError::BadSignature),
        "timestamp"
    );

    let mut m = good.clone();
    m.nonce[0] ^= 1;
    assert_eq!(
        v.verify(CTX, &m, T0),
        Err(VerifyError::BadSignature),
        "nonce"
    );

    let mut m = good.clone();
    m.key_id[0] ^= 1;
    assert_eq!(
        v.verify(CTX, &m, T0),
        Err(VerifyError::UnknownKey),
        "key id"
    );

    for byte in 0..64 {
        let mut m = good.clone();
        m.signature[byte] ^= 0x01;
        assert_eq!(
            v.verify(CTX, &m, T0),
            Err(VerifyError::BadSignature),
            "signature byte {byte}"
        );
    }
    assert_eq!(
        v.remembered(),
        0,
        "rejected messages leave no trace in the replay cache"
    );
    assert_eq!(
        v.verify(CTX, &good, T0),
        Ok(()),
        "the untouched message still works"
    );
}

#[test]
fn a_signature_is_only_valid_for_its_context() {
    let mut v = verifier(60, 5, 100);
    let m = msg(1, T0);
    assert_eq!(
        v.verify("other-purpose", &m, T0),
        Err(VerifyError::BadSignature)
    );
    assert_eq!(
        v.verify("login-challenge ", &m, T0),
        Err(VerifyError::BadSignature)
    );
    assert_eq!(v.verify("", &m, T0), Err(VerifyError::BadSignature));
    assert_eq!(v.verify(CTX, &m, T0), Ok(()));
}

#[test]
fn signing_input_is_unambiguous() {
    let id = [1u8; 8];
    let n = [2u8; 16];
    let a = signing_input("ab", &id, 5, &n, b"c");
    let b = signing_input("a", &id, 5, &n, b"bc");
    assert_ne!(a, b, "bytes moved from the context into the payload");
    assert_ne!(
        signing_input("x", &id, 5, &n, b"p"),
        signing_input("x", &id, 6, &n, b"p")
    );
    assert_ne!(
        signing_input("x", &id, 5, &n, b"p"),
        signing_input("x", &[3; 8], 5, &n, b"p")
    );
    assert_ne!(
        signing_input("x", &id, 5, &n, b"p"),
        signing_input("x", &id, 5, &[3; 16], b"p")
    );
    assert!(
        a.starts_with(&(27u64).to_be_bytes()),
        "protocol label comes first"
    );
}

#[test]
fn age_limit_is_inclusive_and_enforced() {
    let mut v = verifier(60, 5, 100);
    assert_eq!(
        v.verify(CTX, &msg(1, T0), T0 + 60),
        Ok(()),
        "exactly max_age old"
    );
    assert_eq!(
        v.verify(CTX, &msg(2, T0), T0 + 61),
        Err(VerifyError::Expired),
        "one second older"
    );
}

#[test]
fn future_skew_limit_is_inclusive_and_enforced() {
    let mut v = verifier(60, 5, 100);
    assert_eq!(
        v.verify(CTX, &msg(1, T0 + 5), T0),
        Ok(()),
        "exactly max_skew ahead"
    );
    assert_eq!(
        v.verify(CTX, &msg(2, T0 + 6), T0),
        Err(VerifyError::FromFuture)
    );
}

#[test]
fn timestamps_near_the_numeric_limits_do_not_overflow() {
    let mut v = verifier(60, 5, 100);
    assert_eq!(
        v.verify(CTX, &msg(1, u64::MAX), 10),
        Err(VerifyError::FromFuture)
    );
    assert_eq!(
        v.verify(CTX, &msg(2, 0), u64::MAX - 1),
        Err(VerifyError::Expired)
    );
}

#[test]
fn key_validity_window_is_start_inclusive_end_exclusive() {
    let mut v = Verifier::new(1_000_000, 5, 100);
    v.trust(kp(1).public(), 100, 200).unwrap();
    assert_eq!(
        v.verify(CTX, &msg(1, 100), 99),
        Err(VerifyError::KeyNotActive)
    );
    assert_eq!(v.verify(CTX, &msg(1, 100), 100), Ok(()));
    assert_eq!(v.verify(CTX, &msg(2, 100), 199), Ok(()));
    assert_eq!(
        v.verify(CTX, &msg(3, 100), 200),
        Err(VerifyError::KeyNotActive)
    );
}

#[test]
fn rotation_with_overlap_accepts_both_keys_then_only_the_new_one() {
    let (old, new) = (kp(1), kp(2));
    let mut v = Verifier::new(1_000, 5, 100);
    v.trust(old.public(), 0, 2_000).unwrap();
    v.trust(new.public(), 1_500, u64::MAX).unwrap();
    let by = |k: &Keypair, n: u8, t: u64| sign_with_nonce(k, CTX, b"p", t, [n; 16]);
    assert_eq!(v.verify(CTX, &by(&old, 1, 1_000), 1_000), Ok(()));
    assert_eq!(
        v.verify(CTX, &by(&new, 2, 1_000), 1_000),
        Err(VerifyError::KeyNotActive),
        "not yet"
    );
    assert_eq!(
        v.verify(CTX, &by(&old, 3, 1_600), 1_600),
        Ok(()),
        "overlap: old still valid"
    );
    assert_eq!(
        v.verify(CTX, &by(&new, 4, 1_600), 1_600),
        Ok(()),
        "overlap: new valid"
    );
    assert_eq!(
        v.verify(CTX, &by(&old, 5, 2_000), 2_000),
        Err(VerifyError::KeyNotActive),
        "old retired"
    );
    assert_eq!(v.verify(CTX, &by(&new, 6, 2_000), 2_000), Ok(()));
}

#[test]
fn trust_rejects_weak_keys_empty_windows_and_duplicates() {
    let mut v = Verifier::new(60, 5, 10);
    // The identity point (y = 1) is a small-order key: any signature "verifies" under some rules.
    let mut id_bytes = [0u8; 32];
    id_bytes[0] = 1;
    let weak = ed25519_dalek::VerifyingKey::from_bytes(&id_bytes).unwrap();
    assert!(weak.is_weak());
    assert_eq!(v.trust(weak, 0, 10), Err(VerifyError::WeakKey));
    assert!(v.trust(kp(1).public(), 10, 10).is_err(), "empty window");
    assert!(v.trust(kp(1).public(), 20, 10).is_err(), "inverted window");
    assert!(v.trust(kp(1).public(), 0, 10).is_ok());
    assert!(v.trust(kp(1).public(), 0, 10).is_err(), "duplicate key id");
    assert!(v.trust(kp(2).public(), 0, 10).is_ok());
}

#[test]
fn unknown_signers_are_refused() {
    let mut v = verifier(60, 5, 100);
    let stranger = sign_with_nonce(&kp(9), CTX, b"p", T0, [1; 16]);
    assert_eq!(v.verify(CTX, &stranger, T0), Err(VerifyError::UnknownKey));
    assert_ne!(key_id(&kp(9).public()), key_id(&kp(1).public()));
}

#[test]
fn the_replay_cache_forgets_old_nonces_but_not_valid_ones() {
    let mut v = verifier(60, 5, 100);
    for n in 0..10u8 {
        assert_eq!(v.verify(CTX, &msg(n, T0), T0), Ok(()));
    }
    assert_eq!(v.remembered(), 10);
    // 61 seconds later the old messages are Expired anyway; the cache drops them.
    assert_eq!(v.verify(CTX, &msg(200, T0 + 61), T0 + 61), Ok(()));
    assert_eq!(v.remembered(), 1);
    // Exactly at max_age the entries are still remembered (replay still detected).
    let mut v = verifier(60, 5, 100);
    assert_eq!(v.verify(CTX, &msg(1, T0), T0), Ok(()));
    assert_eq!(v.verify(CTX, &msg(2, T0 + 60), T0 + 60), Ok(()));
    assert_eq!(v.remembered(), 2, "an entry max_age old is not pruned yet");
    assert_eq!(
        v.verify(CTX, &msg(1, T0), T0 + 60),
        Err(VerifyError::Replay)
    );
}

#[test]
fn a_full_cache_refuses_new_messages_instead_of_forgetting_valid_ones() {
    let mut v = verifier(60, 5, 3);
    for n in 0..3u8 {
        assert_eq!(v.verify(CTX, &msg(n, T0), T0), Ok(()));
    }
    assert_eq!(v.verify(CTX, &msg(3, T0), T0), Err(VerifyError::CacheFull));
    assert_eq!(
        v.verify(CTX, &msg(0, T0), T0),
        Err(VerifyError::Replay),
        "old entries kept"
    );
    // after the window passes there is room again
    assert_eq!(v.verify(CTX, &msg(3, T0 + 100), T0 + 100), Ok(()));
}

#[test]
fn forged_messages_cannot_fill_the_replay_cache() {
    let mut v = verifier(60, 5, 3);
    for n in 0..50u8 {
        let mut m = msg(n, T0);
        m.signature[0] ^= 1;
        assert_eq!(v.verify(CTX, &m, T0), Err(VerifyError::BadSignature));
    }
    assert_eq!(v.remembered(), 0);
    assert_eq!(v.verify(CTX, &msg(1, T0), T0), Ok(()));
}
```

### `tests/keys.rs`

```rust
use std::os::unix::fs::PermissionsExt;
use std::path::PathBuf;
use std::time::{Duration, Instant};

use cryptolab::hex;
use cryptolab::keys::{
    HEADER_LEN, KEY_ID_LEN, KdfParams, KeyError, Keypair, MAX_M_KIB, MAX_T, SEALED_LEN, open_seed,
    read_key_file, seal_seed, seal_with, write_key_file,
};

const CHEAP: KdfParams = KdfParams { m_kib: 8, t: 1 };

struct TempDir(PathBuf);
impl TempDir {
    fn new(tag: &str) -> Self {
        let p = std::env::temp_dir().join(format!(
            "cryptolab-{tag}-{}-{:?}",
            std::process::id(),
            std::thread::current().id()
        ));
        let _ = std::fs::remove_dir_all(&p);
        std::fs::create_dir_all(&p).unwrap();
        TempDir(p)
    }
}
impl Drop for TempDir {
    fn drop(&mut self) {
        let _ = std::fs::remove_dir_all(&self.0);
    }
}

#[test]
fn a_sealed_seed_opens_with_the_right_passphrase_only() {
    let seed = [42u8; 32];
    let blob = seal_seed(&seed, b"correct horse", CHEAP).unwrap();
    assert_eq!(blob.len(), SEALED_LEN);
    assert_eq!(SEALED_LEN, 101);
    assert_eq!(*open_seed(&blob, b"correct horse").unwrap(), seed);
    for wrong in [
        &b"correct hors"[..],
        b"correct horsE",
        b"correct horse ",
        b"x",
    ] {
        assert!(
            matches!(open_seed(&blob, wrong), Err(KeyError::Decrypt)),
            "{wrong:?}"
        );
    }
}

#[test]
fn the_plaintext_seed_and_passphrase_do_not_appear_in_the_blob() {
    let seed = [0x5au8; 32];
    let blob = seal_seed(&seed, b"pass phrase 123", CHEAP).unwrap();
    assert!(!blob.windows(32).any(|w| w == seed));
    assert!(!blob.windows(15).any(|w| w == b"pass phrase 123"));
    assert_eq!(&blob[..4], b"CLK1");
    assert_eq!(blob[4], 1);
    assert_eq!(u32::from_be_bytes(blob[5..9].try_into().unwrap()), 8);
    assert_eq!(u32::from_be_bytes(blob[9..13].try_into().unwrap()), 1);
    assert_eq!(HEADER_LEN, 53);
}

#[test]
fn every_single_byte_change_is_detected() {
    let blob = seal_seed(&[1u8; 32], b"pw", CHEAP).unwrap();
    for i in 0..blob.len() {
        let mut bad = blob.clone();
        bad[i] ^= 0x01;
        assert!(
            open_seed(&bad, b"pw").is_err(),
            "flipping a bit of byte {i} went unnoticed"
        );
    }
}

#[test]
fn malformed_blobs_are_classified() {
    let blob = seal_seed(&[1u8; 32], b"pw", CHEAP).unwrap();
    assert!(matches!(
        open_seed(&blob[..blob.len() - 1], b"pw"),
        Err(KeyError::BadFormat)
    ));
    assert!(matches!(
        open_seed(&[blob.clone(), vec![0]].concat(), b"pw"),
        Err(KeyError::BadFormat)
    ));
    assert!(matches!(open_seed(&[], b"pw"), Err(KeyError::BadFormat)));
    let mut v2 = blob.clone();
    v2[4] = 2;
    assert!(matches!(
        open_seed(&v2, b"pw"),
        Err(KeyError::UnsupportedVersion(2))
    ));
    let mut magic = blob.clone();
    magic[0] = b'X';
    assert!(matches!(open_seed(&magic, b"pw"), Err(KeyError::BadFormat)));
    assert!(matches!(
        open_seed(&blob, b""),
        Err(KeyError::EmptyPassphrase)
    ));
}

#[test]
fn hostile_cost_parameters_are_refused_before_any_expensive_work() {
    let blob = seal_seed(&[1u8; 32], b"pw", CHEAP).unwrap();
    for (m, t) in [
        (MAX_M_KIB + 1, 1u32),
        (u32::MAX, 1),
        (7, 1),
        (0, 1),
        (8, 0),
        (8, MAX_T + 1),
        (8, u32::MAX),
    ] {
        let mut bad = blob.clone();
        bad[5..9].copy_from_slice(&m.to_be_bytes());
        bad[9..13].copy_from_slice(&t.to_be_bytes());
        let t0 = Instant::now();
        assert!(
            matches!(open_seed(&bad, b"pw"), Err(KeyError::ParamsOutOfRange)),
            "m={m} t={t}"
        );
        assert!(
            t0.elapsed() < Duration::from_millis(500),
            "m={m} t={t} took {:?}",
            t0.elapsed()
        );
    }
}

#[test]
fn sealing_validates_parameters_and_passphrase_at_both_ends() {
    let s = [0u8; 32];
    assert!(matches!(
        seal_seed(&s, b"", CHEAP),
        Err(KeyError::EmptyPassphrase)
    ));
    assert!(seal_seed(&s, b"p", KdfParams { m_kib: 8, t: 1 }).is_ok());
    assert!(matches!(
        seal_seed(&s, b"p", KdfParams { m_kib: 7, t: 1 }),
        Err(KeyError::ParamsOutOfRange)
    ));
    assert!(matches!(
        seal_seed(&s, b"p", KdfParams { m_kib: 8, t: 0 }),
        Err(KeyError::ParamsOutOfRange)
    ));
    assert!(matches!(
        seal_seed(
            &s,
            b"p",
            KdfParams {
                m_kib: MAX_M_KIB + 1,
                t: 1
            }
        ),
        Err(KeyError::ParamsOutOfRange)
    ));
    assert!(matches!(
        seal_seed(
            &s,
            b"p",
            KdfParams {
                m_kib: 8,
                t: MAX_T + 1
            }
        ),
        Err(KeyError::ParamsOutOfRange)
    ));
    assert!(seal_seed(&s, b"p", KdfParams { m_kib: 8, t: MAX_T }).is_ok());
    assert_eq!(
        KdfParams::default(),
        KdfParams {
            m_kib: 19_456,
            t: 2
        }
    );
}

#[test]
fn salt_and_nonce_are_fresh_for_every_seal() {
    let a = seal_seed(&[1u8; 32], b"pw", CHEAP).unwrap();
    let b = seal_seed(&[1u8; 32], b"pw", CHEAP).unwrap();
    assert_ne!(a[13..29], b[13..29], "salt");
    assert_ne!(a[29..53], b[29..53], "nonce");
    assert_ne!(a[53..], b[53..], "ciphertext");
    let fixed = |salt, nonce| seal_with(&[1u8; 32], b"pw", CHEAP, salt, nonce).unwrap();
    assert_eq!(
        fixed([1; 16], [2; 24]),
        fixed([1; 16], [2; 24]),
        "deterministic given salt and nonce"
    );
    assert_ne!(fixed([1; 16], [2; 24]), fixed([9; 16], [2; 24]));
    assert_ne!(fixed([1; 16], [2; 24]), fixed([1; 16], [9; 24]));
}

#[test]
fn key_files_are_private_and_never_overwritten() {
    let dir = TempDir::new("files");
    let path = dir.0.join("key.clk");
    let blob = seal_seed(&[3u8; 32], b"pw", CHEAP).unwrap();
    write_key_file(&path, &blob).unwrap();
    let mode = std::fs::metadata(&path).unwrap().permissions().mode() & 0o777;
    assert_eq!(mode, 0o600);
    assert_eq!(read_key_file(&path).unwrap(), blob);
    assert!(
        matches!(write_key_file(&path, b"other"), Err(KeyError::Io(_))),
        "no overwrite"
    );
    assert_eq!(read_key_file(&path).unwrap(), blob, "original untouched");
}

#[test]
fn key_files_readable_by_others_are_refused() {
    let dir = TempDir::new("perm");
    let path = dir.0.join("key.clk");
    write_key_file(&path, b"data").unwrap();
    for (mode, bad) in [
        (0o600, false),
        (0o400, false),
        (0o640, true),
        (0o604, true),
        (0o644, true),
        (0o660, true),
        (0o666, true),
        (0o700, false),
        (0o710, true),
    ] {
        std::fs::set_permissions(&path, std::fs::Permissions::from_mode(mode)).unwrap();
        match read_key_file(&path) {
            Err(KeyError::InsecurePermissions(m)) => assert!(bad && m == mode, "mode {mode:o}"),
            Ok(_) => assert!(!bad, "mode {mode:o} should have been refused"),
            Err(e) => panic!("{e}"),
        }
    }
    assert!(matches!(
        read_key_file(&dir.0.join("missing")),
        Err(KeyError::Io(_))
    ));
}

#[test]
fn keypairs_identify_themselves_and_hide_their_secret() {
    let kp = Keypair::from_seed(&[7u8; 32]);
    assert_eq!(kp.id().len(), KEY_ID_LEN);
    assert_eq!(kp.id(), Keypair::from_seed(&[7u8; 32]).id());
    assert_ne!(kp.id(), Keypair::from_seed(&[8u8; 32]).id());
    assert_eq!(*kp.seed(), [7u8; 32]);
    let shown = format!("{kp:?}");
    assert!(shown.contains("<redacted>") && shown.contains(&hex::encode(&kp.id())));
    assert!(
        !shown.contains(&hex::encode(&[7u8; 32])) && !shown.contains("0707"),
        "{shown}"
    );
    let (a, b) = (Keypair::generate(), Keypair::generate());
    assert_ne!(a.id(), b.id());
    assert_ne!(*a.seed(), *b.seed());
    let sig = a.sign(b"m");
    assert!(a.public().verify_strict(b"m", &sig).is_ok());
    assert!(b.public().verify_strict(b"m", &sig).is_err());
}

#[test]
fn error_messages_do_not_reveal_which_part_was_wrong() {
    assert_eq!(
        KeyError::Decrypt.to_string(),
        "wrong passphrase or damaged key file"
    );
    assert_eq!(
        KeyError::InsecurePermissions(0o644).to_string(),
        "key file permissions 644 allow group or other access"
    );
    assert_eq!(
        KeyError::UnsupportedVersion(9).to_string(),
        "unsupported key file version 9"
    );
}

#[test]
fn cost_limits_are_pinned_policy_values() {
    assert_eq!(MAX_M_KIB, 262_144, "256 MiB");
    assert_eq!(MAX_T, 10);
}
```

### `tests/lessons.rs`

```rust
//! Experiments that illustrate why the rules exist.

use chacha20poly1305::aead::Aead;
use chacha20poly1305::{ChaCha20Poly1305, KeyInit};

#[test]
fn reusing_a_nonce_leaks_the_xor_of_the_plaintexts() {
    let key = [9u8; 32];
    let nonce = [1u8; 12];
    let cipher = ChaCha20Poly1305::new((&key).into());
    let p1 = b"attack at dawn!!"; // 16 bytes
    let p2 = b"retreat at noon!"; // 16 bytes
    let c1 = cipher.encrypt((&nonce).into(), &p1[..]).unwrap();
    let c2 = cipher.encrypt((&nonce).into(), &p2[..]).unwrap();
    // The first 16 bytes are plaintext XOR keystream; the keystream is the
    // same for both messages, so it cancels out.
    for i in 0..16 {
        assert_eq!(c1[i] ^ c2[i], p1[i] ^ p2[i], "byte {i}");
    }
    // Knowing one plaintext reveals the other without the key:
    let recovered: Vec<u8> = (0..16).map(|i| c1[i] ^ c2[i] ^ p1[i]).collect();
    assert_eq!(&recovered[..], &p2[..]);
}

#[test]
fn with_distinct_nonces_the_xor_relation_disappears() {
    let key = [9u8; 32];
    let cipher = ChaCha20Poly1305::new((&key).into());
    let p1 = b"attack at dawn!!";
    let p2 = b"retreat at noon!";
    let c1 = cipher.encrypt((&[1u8; 12]).into(), &p1[..]).unwrap();
    let c2 = cipher.encrypt((&[2u8; 12]).into(), &p2[..]).unwrap();
    assert!((0..16).any(|i| c1[i] ^ c2[i] != p1[i] ^ p2[i]));
}

#[test]
fn an_authenticated_cipher_rejects_any_modification() {
    let cipher = ChaCha20Poly1305::new((&[5u8; 32]).into());
    let nonce = [3u8; 12];
    let ct = cipher
        .encrypt((&nonce).into(), &b"hello world"[..])
        .unwrap();
    assert_eq!(
        ct.len(),
        11 + 16,
        "ciphertext is 16 bytes longer than the plaintext (the tag)"
    );
    for i in 0..ct.len() {
        let mut bad = ct.clone();
        bad[i] ^= 1;
        assert!(
            cipher.decrypt((&nonce).into(), &bad[..]).is_err(),
            "byte {i}"
        );
    }
    assert!(
        cipher.decrypt((&[4u8; 12]).into(), &ct[..]).is_err(),
        "wrong nonce"
    );
    assert_eq!(
        cipher.decrypt((&nonce).into(), &ct[..]).unwrap(),
        b"hello world"
    );
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
//! Single-machine measurements quoted in the lesson (release build).
use std::hint::black_box;
use std::time::Instant;

use cryptolab::digest::{blake3, sha256, sha512, truncated_collision};
use cryptolab::keys::{KdfParams, Keypair, open_seed, seal_seed};
use cryptolab::mac::{ct_eq, hmac_sha256, naive_eq};
use cryptolab::{sign, toy_sha256};

type Op<'a> = (&'a str, Box<dyn Fn() + 'a>);

fn median(mut v: Vec<f64>) -> f64 {
    v.sort_by(|a, b| a.partial_cmp(b).unwrap());
    v[v.len() / 2]
}

fn time_ns<F: FnMut()>(iters: u32, mut f: F) -> f64 {
    let runs: Vec<f64> = (0..5)
        .map(|_| {
            let t = Instant::now();
            for _ in 0..iters {
                f();
            }
            t.elapsed().as_nanos() as f64 / f64::from(iters)
        })
        .collect();
    median(runs)
}

fn main() {
    println!("== hash throughput (median of 5 runs)");
    for &size in &[64usize, 1024, 1 << 20] {
        let data = vec![0x5au8; size];
        let iters = if size >= 1 << 20 { 50 } else { 20_000 };
        let rows: [Op<'_>; 4] = [
            (
                "sha256",
                Box::new(|| {
                    black_box(sha256(black_box(&data)));
                }),
            ),
            (
                "sha512",
                Box::new(|| {
                    black_box(sha512(black_box(&data)));
                }),
            ),
            (
                "blake3",
                Box::new(|| {
                    black_box(blake3(black_box(&data)));
                }),
            ),
            (
                "toy_sha256",
                Box::new(|| {
                    black_box(toy_sha256::digest(black_box(&data)));
                }),
            ),
        ];
        for (name, f) in rows {
            let ns = time_ns(iters, &f);
            let mbs = size as f64 / ns * 1e3;
            println!("{name:10} {size:>8} B: {ns:>12.0} ns/op  {mbs:>8.0} MB/s");
        }
    }

    println!("== MAC and comparison (32-byte values, 64 KiB inputs for compare)");
    println!(
        "hmac_sha256 64 B message: {:.0} ns",
        time_ns(20_000, || {
            black_box(hmac_sha256(black_box(b"key"), black_box(&[1u8; 64])));
        })
    );
    let a = vec![7u8; 65_536];
    let mut first_diff = a.clone();
    first_diff[0] ^= 1;
    let mut last_diff = a.clone();
    *last_diff.last_mut().unwrap() ^= 1;
    for (name, f) in [
        ("naive_eq", naive_eq as fn(&[u8], &[u8]) -> bool),
        ("ct_eq", ct_eq as fn(&[u8], &[u8]) -> bool),
    ] {
        let early = time_ns(20_000, || {
            black_box(f(black_box(&a), black_box(&first_diff)));
        });
        let late = time_ns(20_000, || {
            black_box(f(black_box(&a), black_box(&last_diff)));
        });
        println!(
            "{name:9} differs at first byte: {early:>9.0} ns   at last byte: {late:>9.0} ns   ratio {:.1}x",
            late / early
        );
    }

    println!("== Ed25519");
    let kp = Keypair::from_seed(&[1u8; 32]);
    let msg = [9u8; 100];
    let sig = kp.sign(&msg);
    println!(
        "sign 100 B:           {:.1} us",
        time_ns(5_000, || {
            black_box(kp.sign(black_box(&msg)));
        }) / 1e3
    );
    println!(
        "verify_strict 100 B:  {:.1} us",
        time_ns(5_000, || {
            kp.public().verify_strict(black_box(&msg), &sig).unwrap();
        }) / 1e3
    );
    println!(
        "keygen (from seed):   {:.1} us",
        time_ns(5_000, || {
            black_box(Keypair::from_seed(black_box(&[3u8; 32])).public());
        }) / 1e3
    );
    let m = sign::sign_with_nonce(&kp, "ctx", b"payload", 1000, [1; 16]);
    println!(
        "signed message built: {:.1} us",
        time_ns(5_000, || {
            black_box(sign::sign_with_nonce(&kp, "ctx", b"payload", 1000, [1; 16]));
        }) / 1e3
    );
    let _ = m;

    println!("== key sealing (Argon2id + XChaCha20-Poly1305), open() time");
    for (m_kib, t) in [(8u32, 1u32), (19_456, 2), (65_536, 3)] {
        let blob = seal_seed(&[1u8; 32], b"pw", KdfParams { m_kib, t }).unwrap();
        let ms = time_ns(5, || {
            black_box(open_seed(&blob, b"pw").unwrap());
        }) / 1e6;
        println!("m = {m_kib:>6} KiB, t = {t}: {ms:>8.2} ms");
    }

    println!("== birthday experiment (truncated SHA-256)");
    for bits in [16u32, 24, 32] {
        let runs: Vec<u64> = (0..1).map(|_| truncated_collision(bits).2).collect();
        println!(
            "{bits:>2} bits: first collision after {} hashes (sqrt(2^bits) = {})",
            runs[0],
            1u64 << (bits / 2)
        );
    }
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **61 tests** (vectors 6, toy SHA-256 10, hashing 15, signed messages 15, key storage 12, lessons 3); 10 consecutive full runs without a failure on the 54-test version, then 5 more on the final 61-test suite (timing-sensitive tests use generous bounds, such as 500 ms for refusing hostile parameters that would otherwise take seconds). The first cold build of the crate took 29.65 s (dev profile, 2 cores, dependencies included).

**Mutation check ✓.** I made 66 one-line changes (a "mutant") across the seven source modules, each in its own copy of the crate, and ran the whole test suite against each. **63 were detected (61 by a failing or aborting test, 2 as hangs), 3 are equivalent (M7, M20, M41), none survive.** Six mutants survived or went unclassified in the *first* run; I looked at each, and five exposed real gaps that I closed with a new test before re-running: M6 (the domain length prefix: my tests only covered field boundaries, so I built an explicit boundary-shift collision), M12 (the number of hashes in the collision experiment was never checked), M27 (a changed limit constant is invisible to tests that import the constant; the limits are now pinned with literal numbers), M62 (the whole-block assertion was tested only with 63, not with 32) and M22/M23 (my first mutation patterns did not match the source formatting, so they were reported *invalid*; after fixing the patterns both were caught). Two more things about the harness: M62 was first reported as "survived" because `#[should_panic]` failures print `- should panic ... FAILED`, which my classifier did not recognise (I read the log and fixed the claim; the other three survivors were checked the same way and really have no failing test); and M41 first failed to compile because `verify` needs a trait import, so I rewrote the mutant with a fully qualified call. Two mutants (M24, M26) made the hostile-parameter test run Argon2 with absurd costs and did not finish; I count a hang as detection, but it is a weaker signal than an assertion. The tests ran in the dev profile.

| # | module | change | result |
|---|---|---|---|
| M1 | `hex` | digit table typo (`e` at the end) | caught by `hex_round_trips_every_byte_value` |
| M2 | `hex` | odd-length check removed | caught by `hex_decode_rejects_bad_input_with_a_position` |
| M3 | `hex` | upper-case `A`–`F` no longer accepted | caught by `hex_round_trips_every_byte_value` |
| M4 | `hex` | `InvalidChar` reports the high-nibble index | caught by `hex_decode_rejects_bad_input_with_a_position` |
| M5 | `hex` | lower-case `a`–`f` value off by one | caught by `hex_round_trips_every_byte_value` |
| M6 | `digest` | domain length prefix removed | caught by `hash_fields_domain_length_prefix_blocks_a_boundary_shift_collision` |
| M7 | `digest` | field count removed | **equivalent**: length-prefixing every field already makes the encoding injective, so the count adds nothing observable (kept as defense in depth) |
| M8 | `digest` | field length prefix removed | caught by `hash_fields_separates_domains_counts_and_empty_fields` |
| M9 | `digest` | collision mask `1 << bits` instead of `(1 << bits) - 1` | caught by `truncated_hashes_collide_after_about_the_birthday_bound` |
| M10 | `digest` | `truncation_supported` excludes 48 bits | caught by `the_supported_truncation_range_is_exact` |
| M11 | `digest` | `truncation_supported` excludes 1 bit | caught by `the_supported_truncation_range_is_exact` |
| M12 | `digest` | hash count reported as `i` instead of `i + 1` | caught by `truncated_collision_reports_the_number_of_hashes_it_computed` |
| M13 | `mac` | `verify_hmac_sha256` always true | caught by `hmac_verifies_only_the_exact_tag` |
| M14 | `mac` | `ct_eq` compares lengths only | caught by `comparison_functions_agree_on_equality` |
| M15 | `mac` | `naive_eq` length check removed | caught by `comparison_functions_agree_on_equality` |
| M16 | `mac` | `naive_eq` skips the last byte | caught by `comparison_functions_agree_on_equality` |
| M17 | `mac` | insecure prefix MAC hashes message before key | caught by `length_extension_forges_a_tag_for_sha256_key_concat_message` |
| M18 | `kdf` | `MAX_OUTPUT` one too large | caught by `hkdf_limits_and_prefix_property` |
| M19 | `kdf` | zero-length output accepted | caught by `hkdf_limits_and_prefix_property` |
| M20 | `kdf` | empty salt passed as `Some(&[])` instead of `None` | **equivalent**: HKDF treats an absent salt and an empty salt identically (a zero-filled salt of hash length) |
| M21 | `kdf` | `subkey` ignores the purpose label | caught by `subkeys_are_deterministic_and_independent_per_purpose` |
| M22 | `keys` | seal: header not passed as associated data | caught by `a_sealed_seed_opens_with_the_right_passphrase_only` |
| M23 | `keys` | open: header not passed as associated data | caught by `a_sealed_seed_opens_with_the_right_passphrase_only` |
| M24 | `keys` | open: parameter range check removed | caught as a **hang**: the hostile-parameter test never finished (killed after 300 s) |
| M25 | `keys` | seal: parameter range check removed | caught by `sealing_validates_parameters_and_passphrase_at_both_ends` |
| M26 | `keys` | open: only the memory limit checked, not `t` | caught as a **hang**: the hostile-parameter test never finished (killed after 300 s) |
| M27 | `keys` | `MAX_M_KIB` one too large | caught by `cost_limits_are_pinned_policy_values` |
| M28 | `keys` | seal: empty-passphrase check removed | caught by `sealing_validates_parameters_and_passphrase_at_both_ends` |
| M29 | `keys` | blob length check `<` instead of `!=` | caught by `malformed_blobs_are_classified` |
| M30 | `keys` | `create_new` replaced by `create` (overwrites) | caught by `key_files_are_private_and_never_overwritten` |
| M31 | `keys` | file mode 0o644 instead of 0o600 | caught by `key_files_are_private_and_never_overwritten` |
| M32 | `keys` | read check looks only at "other" bits | caught by `key_files_readable_by_others_are_refused` |
| M33 | `keys` | `Debug` prints the seed | caught by `keypairs_identify_themselves_and_hide_their_secret` |
| M34 | `keys` | key id taken from bytes 1..=8 | caught by `key_ids_are_the_first_eight_bytes_of_the_public_key_hash` |
| M35 | `keys` | version byte not checked | caught by `malformed_blobs_are_classified` |
| M36 | `sign` | context not in the signed bytes | caught by `a_signature_is_only_valid_for_its_context` |
| M37 | `sign` | key id not in the signed bytes | caught by `signing_input_is_unambiguous` |
| M38 | `sign` | timestamp not in the signed bytes | caught by `every_field_is_covered_by_the_signature` |
| M39 | `sign` | nonce not in the signed bytes | caught by `signing_input_is_unambiguous` |
| M40 | `sign` | payload not in the signed bytes | caught by `every_field_is_covered_by_the_signature` |
| M41 | `sign` | `verify` instead of `verify_strict` | **equivalent in my suite**: weak keys are refused at registration, so the cases where `verify` and `verify_strict` differ are unreachable by these tests (see §8.2) |
| M42 | `sign` | age check `>=` instead of `>` | caught by `age_limit_is_inclusive_and_enforced` |
| M43 | `sign` | skew check `>=` instead of `>` | caught by `future_skew_limit_is_inclusive_and_enforced` |
| M44 | `sign` | key window start made exclusive | caught by `key_validity_window_is_start_inclusive_end_exclusive` |
| M45 | `sign` | key window end made inclusive | caught by `key_validity_window_is_start_inclusive_end_exclusive` |
| M46 | `sign` | prune `<=` instead of `<` | caught by `the_replay_cache_forgets_old_nonces_but_not_valid_ones` |
| M47 | `sign` | capacity check `>` instead of `>=` | caught by `a_full_cache_refuses_new_messages_instead_of_forgetting_valid_ones` |
| M48 | `sign` | replay check removed | caught by `a_full_cache_refuses_new_messages_instead_of_forgetting_valid_ones` |
| M49 | `sign` | weak-key check removed | caught by `trust_rejects_weak_keys_empty_windows_and_duplicates` |
| M50 | `sign` | duplicate key id check removed | caught by `trust_rejects_weak_keys_empty_windows_and_duplicates` |
| M51 | `sign` | empty window check `>` instead of `>=` | caught by `trust_rejects_weak_keys_empty_windows_and_duplicates` |
| M52 | `sign` | plain `+` instead of `saturating_add` for skew | caught by `timestamps_near_the_numeric_limits_do_not_overflow` |
| M53 | `sign` | plain `-` instead of `saturating_sub` for age | caught by `future_skew_limit_is_inclusive_and_enforced` |
| M54 | `sign` | pruning removed | caught by `a_full_cache_refuses_new_messages_instead_of_forgetting_valid_ones` |
| M55 | `sign` | nonce never remembered | caught by `a_valid_message_verifies_once` |
| M56 | `toy_sha256` | padding: 56 instead of 55 in zero count | caught by `hmac_is_not_vulnerable_to_the_same_attack` |
| M57 | `toy_sha256` | length field in bytes instead of bits | caught by `length_extension_forges_a_tag_for_sha256_key_concat_message` |
| M58 | `toy_sha256` | Σ1 rotation 24 instead of 25 | caught by `length_extension_forges_a_tag_for_sha256_key_concat_message` |
| M59 | `toy_sha256` | `Ch` uses `e & g` instead of `!e & g` | caught by `length_extension_forges_a_tag_for_sha256_key_concat_message` |
| M60 | `toy_sha256` | `Maj` missing the `b & c` term | caught by `length_extension_forges_a_tag_for_sha256_key_concat_message` |
| M61 | `toy_sha256` | buffer flushed at 63 bytes | caught by `feeding_data_in_pieces_changes_nothing` |
| M62 | `toy_sha256` | `from_state` accepts multiples of 32 | caught by `resuming_rejects_half_a_block_too` |
| M63 | `toy_sha256` | `extend` forgets the padding length | caught by `hmac_is_not_vulnerable_to_the_same_attack` |
| M64 | `toy_sha256` | round constants: low bit cleared | caught by `feeding_data_in_pieces_changes_nothing` |
| M65 | `toy_sha256` | σ0 shift 4 instead of 3 | caught by `length_extension_forges_a_tag_for_sha256_key_concat_message` |
| M66 | `toy_sha256` | bulk loop `>` instead of `>=` 64 | caught by `feeding_data_in_pieces_changes_nothing` |

What the table does not show: it tests *these* edits, not every possible bug (it contains no mutant for the dependencies, for `getrandom` failures, or for non-Unix code); the timing-based tests rely on wide margins; and passing every mutant says nothing about security.

**Design notes.**

- Policy values (limits, window rules, label strings) are plain constants and small functions that tests call directly. Time and nonces are injected (`sign_with_nonce`, the `now` argument) so every boundary can be tested exactly; randomness is used only at the edges (`Keypair::generate`, `sign`, `seal_seed`).
- The crate uses reviewed implementations for every primitive that matters. The hand-written SHA-256 is isolated in `toy_sha256` and is used only for the length-extension demonstration and as a teaching aid.
- Errors are specific where that helps the caller (`Replay`, `Expired`) and deliberately uniform where specificity would help an attacker (`Decrypt` for wrong passphrase or damage).
- Not handled: constant-time guarantees of the dependencies, side-channel tests beyond one microbenchmark, multi-process replay caches, revocation, a signing service or HSM interface, key exchange, certificates, other signature schemes, other platforms' file permission models, password policy, rate limiting of unlock attempts, and secure memory (locking pages, avoiding swap).

</details>

---

## 19. Key takeaways

- A hash protects integrity only when you got the digest from somewhere trusted; collision resistance costs `2^(n/2)`, and the birthday bound is not theory (✓ 232, 4,303 and 95,754 hashes for 16, 24 and 32 bits).
- `SHA256(key ‖ message)` is forgeable without the key (✓ performed with a differentially tested hand-written SHA-256); use HMAC. Compare secrets in constant time (✓ 11,000× difference for the naive compare in a local microbenchmark; the right rule is cheap, not the right exploit).
- Encode fields unambiguously and separate domains; the length prefix on the domain was not redundant, the field count was (✓ mutation testing told them apart).
- One key, one purpose: derive subkeys with labelled HKDF. Passwords need a slow memory-hard function (Argon2id), not a fast hash or HKDF.
- Ed25519 gives deterministic, fast signatures (✓ about 14–18 µs to sign, 36–46 µs to verify strictly); a valid signature is only the start: add context, key identity, freshness, replay protection and key windows, check cheap things first, and change state only after the signature is valid.
- A nonce must never repeat under a key (✓ plaintext XOR leaked); choose the nonce size by the message volume you will actually reach.
- A file or message that tells you how much work to do is hostile input (✓ refused in under 500 ms); pin policy numbers literally in tests.
- Key management (separation, rotation with overlap, revocation, recovery, storage, compromise plans) decides real outcomes; the code can support it but not replace it.
- Mutation testing again exposed gaps: a missing boundary test, a constant that tests could not detect, a counter returned wrong, and revealed which parts of a design are genuinely redundant (✓ three equivalent mutants, honestly labeled).

**Connections.** [Lesson 46](46_databases_config_auth_jobs.md) is where passwords, sessions and secret configuration were introduced; §7 and §10 supply the cryptographic detail behind them. [Lesson 45](45_http_apis.md) shows where tokens travel and why limits matter; [Lesson 44](44_cli_serialization.md) covers the binary framing used for unambiguous encoding; [Lesson 43](43_observability_ci_supply_chain.md) covers choosing and auditing dependencies and keeping secrets out of logs; [Lessons 36 and 37](37_maybeuninit_ffi.md) explain the `unsafe` and raw-memory concerns inside cryptographic crates; [Lesson 28](28_atomics_memory_ordering.md) explains why the optimizer may reorder code, which is why constant-time code relies on barriers; [Lessons 26 and 27](26_error_design.md) cover error design and threads. Lesson 49 builds on hashes and signatures: Merkle trees, transaction models, consensus and trust assumptions.

---

## 20. References

- RFC 8032 (EdDSA / Ed25519), RFC 5869 (HKDF), RFC 4231 (HMAC-SHA test vectors), RFC 2104 (HMAC), RFC 9106 (Argon2), RFC 8439 (ChaCha20-Poly1305): <https://www.rfc-editor.org/>
- NIST FIPS 180-4 (SHA-2) and FIPS 202 (SHA-3): <https://csrc.nist.gov/publications>
- The BLAKE3 specification and reference implementation: <https://github.com/BLAKE3-team/BLAKE3>
- The RustCrypto organization (crates `sha2`, `hmac`, `hkdf`, `argon2`, `chacha20poly1305`, `zeroize`): <https://github.com/RustCrypto>
- `ed25519-dalek` and `curve25519-dalek`: <https://github.com/dalek-cryptography>
- `subtle` (constant-time helpers): <https://github.com/dalek-cryptography/subtle>
- `getrandom`: <https://docs.rs/getrandom>
- Cryptographic Right Answers (Latacora): general guidance on choosing primitives (from memory).
- OWASP Password Storage Cheat Sheet: <https://cheatsheetseries.owasp.org/>

The URLs above are the standard locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the title. **Unchecked or from memory:** the descriptions of the standards (padding rule and constant derivation of SHA-256, HKDF's extract/expand, the HMAC construction, Argon2's parameters, Ed25519's properties, `verify` versus `verify_strict` semantics); that SHA-3 is a sponge unaffected by length extension and that SHA-512/256 and BLAKE3's keyed mode are alternatives; the claim that a 64-bit hash can be collided in hours; remote timing attacks being demonstrated in research; how `subtle` defends against the optimizer; what current Argon2 parameter guidance recommends (check OWASP); that Poly1305-based AEADs can expose their authentication key on nonce reuse; that Ed25519 is broken by large quantum computers; everything in §12 about KMS, HSMs, revocation and rotation practice; the limits of `zeroize` (swap, dumps, registers); the Unix permission semantics of `0o600`; the nonce-collision probability formula (derived from the birthday approximation and ✓ evaluated, not independently simulated). **Environment caveats:** timings are single-machine micro-measurements from three runs of an example on a shared virtual machine; test vectors were cross-checked with Python's `hashlib`, `hmac`, `blake3` and `cryptography` packages; the library versions are those recorded in `Cargo.lock` on 2026-11-10. Measurements and outcomes marked ✓ were observed with `rustc 1.97.0`. The code blocks in exercises 3 to 6 and in the debugging challenge were compiled and run in a scratch crate (✓ 7 tests); the exercise 8 formula was evaluated by a test; the mutation results are those of the table above (the harness classified a test process as detecting a mutant when a test failed, when it aborted, or, for two mutants, when it did not finish within five minutes). **No part of this lesson makes any system secure.**
{% endraw %}
