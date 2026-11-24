---
layout: default
title: "Lesson 52 — Off-chain services, cryptographic integrations, fault tolerance"
---

# Lesson 52 — Off-chain services, cryptographic integrations, fault tolerance

**Module 16 · Blockchain and Web3 infrastructure, chain-agnostic (second lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `sha2 0.11.0`, `ed25519-dalek 3.0.0` and `hmac 0.13.0` (28 packages in `Cargo.lock`, the crate itself included)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** Almost every blockchain product has an **off-chain half**: a service that signs and submits transactions (a *relayer* or *bot*), receives callbacks from other systems (webhooks), keeps a to-do list that must survive a crash (a *job queue* or *outbox*), and keeps working while the network, the provider and its own process misbehave. The crate `relaylab` builds the core of that half at teaching scale: a **signed message envelope** with domain separation, nonces and expiry, a **signing service** with a policy and a nonce allocator that survives restarts, a verifier with **key rotation and revocation**; **HMAC-signed webhooks** with a freshness window; a **checksummed write-ahead journal**; a **durable job queue** rebuilt by replaying that journal, with idempotency keys, leases, backoff and a dead-letter state; a **circuit breaker**; and a seeded **relayer simulation** with crashes, timeouts that happen *after* the work was done, and a chain that may or may not deduplicate. It is teaching code: **not audited, not a wallet, not a key-management system, and not tied to any real network**. Not covered: real key custody (hardware modules, cloud key services, threshold signing), transaction formats of any specific chain, fee estimation and replacement transactions, a real database or file system (the journal lives in memory; fsync and torn-write behavior of real disks are discussed but not tested), HTTP servers and TLS (Lesson 45), clocks that disagree between machines, and monitoring dashboards (Lesson 43). **Nothing here makes a system secure.** Rust removes memory-safety bugs; the failures in this lesson (a nonce used twice after a restart, a payment sent twice after a timeout, a webhook replayed a week later, a circuit breaker that never closes again) are logic errors that compile cleanly, and one of them happened to me while writing the simulation (§6.3).

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation and standards sites were not, so statements about RFCs, about specific providers' webhook formats and about key-management products are labeled "from memory". There are no sockets, no real clocks and no OS randomness in the tests: time, faults and crashes are inputs, so every run is deterministic. Expected values for the HMAC tags, the journal checksums and the pseudo-random sequence were computed by **separate Python programs** (✓), and the two HMAC vectors of RFC 4231 were also checked against Python's `hmac` module (✓). Timings are single-machine micro-measurements.

Builds on [Lesson 51](51_rpc_nodes_indexers.md) (retries, backoff with jitter, uncertain writes, stable notice ids), [Lesson 50](50_networking_determinism.md) (limits, canonical encodings, sans-IO, seeded simulation), [Lesson 49](49_merkle_transactions_consensus.md) (transactions, nonces, finality), [Lesson 48](48_crypto_primitives.md) (hashes, MACs, signatures, domain separation, constant-time comparison), [Lesson 46](46_databases_config_auth_jobs.md) (transactions, jobs, secrets), [Lesson 45](45_http_apis.md) (idempotency keys), [Lesson 43](43_observability_ci_supply_chain.md), [Lesson 41](41_benchmarking_profiling.md), [Lesson 40](40_property_testing_fuzzing.md), [Lesson 26](26_error_design.md) and [Lesson 15](15_newtype_typestate.md). Lesson 53 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

A service that holds a signing key and pushes transactions to a chain is the most dangerous kind of program a Web3 team writes: **its mistakes cost money, they are often irreversible, and they happen at the worst time** (during an outage, a restart, a retry storm). The lesson is organized around experiments that show where naive designs break:

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| A signature must cover everything that gives the message its meaning | changing the nonce, the expiry, the payload or the signature itself makes verification fail; moving a signature to another chain id fails (§2.2) |
| A signing service must not sign arbitrary bytes | a policy refuses an amount over the limit and a payload too short to hold one; refused requests burn no nonce (§2.3) |
| A restarted signer must never reuse a nonce | 500 signings with random restarts, every nonce distinct, because a *reservation* is stored before any number in it is used (§2.4) |
| Verify with the strict ed25519 check | an identity-point "public key" with an all-zero signature is accepted by the ordinary check and refused by `verify_strict` (§2.6) |
| Keys must rotate without an outage | during the overlap both keys verify; after the old window closes only the new one does; a revoked key stops at once (§2.7) |
| A webhook MAC needs a clock | the same valid tag is accepted at 1,000 ms of skew and refused at 1,001 ms (§3.3) |
| Rotating a webhook secret needs two valid keys | a delivery signed with the old or the new secret verifies against `[new, old]` (§3.4) |
| A journal's tail may be torn | cut at *every* byte offset, recovery returns exactly the complete records before the cut; a flip of *any single bit* is noticed (§4.2) |
| A queue is its event log | after random operations the queue rebuilt from the journal bytes equals the live one, byte for byte (§5.4) |
| A stale worker must not finish a job | after a lease expires, the first holder's `complete` is refused with `StaleLease` (§5.3) |
| A breaker can get stuck | an unused half-open permit kept the breaker shut for ever in 2 of 200 seeds, until I fixed it (§6.3) |
| Retries plus timeouts double-apply on a chain that does not deduplicate | 200 of 200 seeds double-applied at least one transaction; on a deduplicating chain none did (§7.2) |
| At-least-once delivery plus an idempotent effect is exactly-once *in effect* | 200 seeds with 30% refusals, 20% lost acknowledgements and crashes: every one of 50 jobs applied once (§7.3) |

Total in the practical challenge: one library crate, 7 modules, **90 tests**, a benchmark example, and **131 mutants** run against the test suite.

---

## 2. Signing: messages, services and keys

### 2.1 What a signature protects, and what it does not

A digital signature (Lesson 48) says: *someone holding this private key produced a signature over exactly these bytes*. Everything else a system needs must be put **into** those bytes or enforced **around** them:

- **Who is it for?** A signature over `"pay 5 to Bob"` is valid on every network and every contract that uses the same key. Include a **domain** (what protocol, which version) and a **chain id** (which network).
- **Is it fresh?** Include an **expiry**. An old but valid message must stop working.
- **Is it a repeat?** Include a **nonce** that the verifier remembers (§2.5). Without one, anyone who saw the message can send it again.
- **Where does the payload end?** Include its **length**, so that two different field layouts cannot produce the same bytes (Lesson 48 §4).
- **Which key?** Include a **key id**, so that rotation (§2.7) is possible and the verifier knows which public key to try.

A signature does **not** say that the signer *should* have signed it, that the amount is sensible, or that the key was not stolen. Those are policy (§2.3) and custody (§2.8).

### 2.2 The envelope ✓

```rust
/// A signed message with the context that must not be replayable elsewhere.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Envelope {
    /// The network this message is for.
    pub chain_id: u32,
    /// Which key signed it.
    pub key_id: u32,
    /// Strictly increasing per key.
    pub nonce: u64,
    /// The message is invalid at or after this time (ms).
    pub expires_ms: u64,
    /// What is being authorized.
    pub payload: Vec<u8>,
    /// Ed25519 signature over [`Envelope::signing_bytes`].
    pub sig: [u8; 64],
}
```

The bytes that are signed (`signing_bytes`) are, in order: the length of the domain label as 8 bytes, the label `lab/envelope/v1`, then `chain_id` (4 bytes), `key_id` (4), `nonce` (8), `expires_ms` (8), the payload length (8) and the payload, all integers big-endian. The **wire format** (`encode`) is different on purpose: `chain ‖ key ‖ nonce ‖ expires ‖ len:u32 ‖ payload ‖ signature`, a 28-byte header, the payload and 64 bytes of signature. The signed bytes and the wire bytes are two separate encodings, so a change to the transport format can never change what is signed.

✓ Tests (`tests/sign_tests.rs`): `signing_bytes_layout` compares the exact byte string for a small envelope with one built by hand; `encode_decode_round_trip_and_layout` checks every header offset; `strict_decode` cuts a valid message at **every** length and requires `Truncated`, adds one trailing byte (`TrailingBytes`) and sets the length field to `u32::MAX` and to one past the limit (`PayloadTooLong`). The decoder refuses declared lengths over `MAX_PAYLOAD` (65,536) *before* allocating anything (Lesson 50 §3). `every_field_is_covered_by_the_signature` changes the nonce, the expiry and one signature byte in turn: each gives `BadSignature`. `domain_separation_between_chains` signs for chain 7, relabels the envelope as chain 8 and verifies on chain 8: refused. That last test is the reason the chain id is *inside* the signed bytes and not only a routing field around them.

### 2.3 The signing service and its policy ✓

A signing service is a function from **requests** to **signatures**, and its most important property is what it **refuses**:

```rust
/// Signs `payload`, valid for `ttl_ms` from `now_ms`.
pub fn sign(
    &mut self,
    payload: Vec<u8>,
    now_ms: u64,
    ttl_ms: u64,
) -> Result<Envelope, SignError> {
    if payload.len() > MAX_PAYLOAD {
        return Err(SignError::PayloadTooLong);
    }
    if ttl_ms == 0 || ttl_ms > MAX_TTL_MS {
        return Err(SignError::BadTtl(ttl_ms));
    }
    self.policy.check(&payload).map_err(SignError::Policy)?;
    let mut env = Envelope {
        chain_id: self.chain_id,
        key_id: self.signer.key_id(),
        nonce: self.nonces.allocate(),
        expires_ms: now_ms.saturating_add(ttl_ms),
        payload,
        sig: [0; 64],
    };
    env.sig = self.signer.sign(&env.signing_bytes());
    Ok(env)
}
```

The service checks, in this order: the payload size, the TTL (greater than 0 and at most `MAX_TTL_MS` = 10 minutes), and the **policy**. The policy here (`MaxAmount`) reads an 8-byte big-endian amount from the start of the payload and refuses larger values and payloads too short to contain one. ✓ `policy_and_size_checks` shows an amount of 101 refused with limit 100, exactly 100 accepted, a 2-byte payload refused, and a payload of 65,537 bytes refused as `PayloadTooLong` while exactly 65,536 is accepted (`payload_size_boundaries_use_literal_numbers`, which uses the *numbers* and not the constants, so that a changed constant cannot hide: see §14). TTL boundaries are pinned the same way (600,000 ms accepted, 600,001 refused, 0 refused).

Why does a policy belong in the signer? Because the code that *builds* requests is the code most likely to be attacked or buggy (it parses user input, it talks to the network). A signer that signs anything it is handed turns every bug upstream into a loss. The policy is a small, separately tested last line of defense: **limits per transaction, allowed destinations or contracts, rate and total limits per day, and an allowlist of message types**. The crate implements only the first; the others have the same shape. (A real design also logs every decision, with the reason, to a place the signer cannot rewrite.)

✓ `refused_requests_do_not_burn_nonces`: after a policy refusal the next envelope still has nonce 0. A nonce is only consumed when a signature is produced.

### 2.4 Nonces that survive a restart ✓

The dangerous case is not the first run; it is the **crash**. If a signer keeps its next nonce only in memory, a restart forgets it. Two outcomes are possible, and both are bad: it starts again from 0 (every new message is a replay and is refused, or worse, on a chain where nonces are sequence numbers, the transactions are stuck), or it is restored from a *stale* copy and issues numbers it has already used for *different* payloads.

The crate's answer is the **reservation**: the allocator hands out numbers from a block that has been reserved, and whenever it needs to start a new block it produces a *pending reservation* that the caller must store **durably before using any number in the block**. After a crash, a new allocator starts at the last stored reservation, so it skips the unused rest of the old block. Gaps are harmless; reuse is not.

```rust
/// Hands out increasing nonces without ever repeating one after a restart:
/// it *reserves* a block of numbers, the caller stores the reservation
/// durably **before** using any number from the block, and a restarted
/// allocator starts at the stored value. Gaps are harmless; reuse is not.
#[derive(Debug, Clone)]
pub struct NonceAllocator {
    next: u64,
    reserved: u64,
    block: u64,
    pending: Option<u64>,
}

impl NonceAllocator {
    /// `durable_high_water` is the last reservation that was stored (0 for a
    /// new key); `block` is how many numbers each reservation covers.
    pub fn new(durable_high_water: u64, block: u64) -> NonceAllocator {
        assert!(block > 0, "block must be positive");
        NonceAllocator {
            next: durable_high_water,
            reserved: durable_high_water,
            block,
            pending: None,
        }
    }

    /// The next nonce.
    pub fn allocate(&mut self) -> u64 {
        if self.next >= self.reserved {
            self.reserved = self
                .next
                .checked_add(self.block)
                .expect("nonce space exhausted");
            self.pending = Some(self.reserved);
        }
        let n = self.next;
        self.next += 1;
        n
    }

    /// A new reservation that must be stored durably before the nonces it
    /// covers are used.
    pub fn take_pending(&mut self) -> Option<u64> {
        self.pending.take()
    }
}
```

✓ `nonce_allocator_reserves_before_use_and_survives_restart` walks through block size 4: number 0 produces the pending reservation 4; numbers 1–3 produce none; number 4 produces 8; a restart at the stored value 8 continues with 8. ✓ `restart_never_reuses_a_nonce` signs 500 envelopes, restarting the service with probability 10% after each one from the *stored* reservation, and requires every nonce to be distinct. The cost of the design is visible in the simulation: ✓ a relayer that restarts after every single submission burns a whole block each time (`nonce_high_water == 16 × submissions`, §7.4), which is why the block size is a trade-off between wasted numbers and the frequency of durable writes. (On chains where the nonce is a **gapless sequence number per account**, from memory the model of several account-based chains, a gap is *not* harmless and this scheme does not apply as it stands: there the nonce comes from the chain, and the signer must ask for it and serialize its own sends. The envelope nonce in this lesson is an *application-level* replay counter.)

### 2.5 The verifier ✓

```rust
/// Checks everything and, only if all checks pass, records the nonce.
pub fn verify(&mut self, env: &Envelope, now_ms: u64) -> Result<(), VerifyError> {
    if env.chain_id != self.chain_id {
        return Err(VerifyError::WrongChain);
    }
    let key = self
        .keys
        .get(&env.key_id)
        .ok_or(VerifyError::UnknownKey(env.key_id))?;
    if key.revoked {
        return Err(VerifyError::Revoked(env.key_id));
    }
    if now_ms < key.valid_from_ms || now_ms >= key.valid_until_ms {
        return Err(VerifyError::KeyNotValidNow(env.key_id));
    }
    if now_ms >= env.expires_ms {
        return Err(VerifyError::Expired);
    }
    if env.expires_ms - now_ms > MAX_TTL_MS {
        return Err(VerifyError::TtlTooLong);
    }
    let vk = VerifyingKey::from_bytes(&key.public).map_err(|_| VerifyError::BadKey)?;
    vk.verify_strict(&env.signing_bytes(), &Signature::from_bytes(&env.sig))
        .map_err(|_| VerifyError::BadSignature)?;
    if let Some(&last) = self.last_nonce.get(&env.key_id) {
        if env.nonce <= last {
            return Err(VerifyError::Replay {
                last,
                got: env.nonce,
            });
        }
    }
    self.last_nonce.insert(env.key_id, env.nonce);
    Ok(())
}
```

The order of the checks matters and is deliberate:

1. **Cheap, public facts first**: the chain id, then whether the key id is known, revoked, or outside its validity window, then expiry and TTL. None of these needs a signature check, so garbage is refused without the most expensive step.
2. **The signature** with `verify_strict` (§2.6).
3. **The nonce last, and recorded only on success.** ✓ `bad_signature_does_not_advance_nonce`: an attacker who sends a forged envelope with a huge nonce must not be able to move the counter and lock out the real sender. If the nonce were recorded before the signature check, that is exactly what would happen. ✓ `replay_rejected_and_order_enforced` shows `Replay { last, got }` for the same envelope twice and for an older one.

Time boundaries are pinned with inclusive/exclusive tests: an envelope expiring at 1,000 is valid at 999 and `Expired` at 1,000 (`now >= expires`); a key valid in `[from, until)` works at `from` and `until − 1` and fails at `until` (✓ `key_windows_and_revocation`). The expiry check is *also* bounded above: an envelope that claims to live for more than 10 minutes is refused as `TtlTooLong` even if its signature is valid, because a signer that mistakenly (or maliciously) issues year-long envelopes would otherwise turn every stolen envelope into a long-lived credential.

The replay counter here is **per key id** and strictly increasing, which is the simplest correct rule and the one that fits a single ordered stream. If messages can arrive out of order, a *sliding window* of recently seen nonces is the usual refinement (exercise 2).

### 2.6 `verify_strict` and the identity key ✓

Ed25519 as originally specified has a few corner cases. Some public keys have **small order** (the identity point is one); with such a key an attacker can produce signatures that verify for *any* message. A library's ordinary `verify` accepts some of those; `verify_strict` rejects small-order keys and non-canonical encodings (from memory of the `ed25519-dalek` documentation, which says that strict verification is the one to use when signatures must be unique to a message; the exact set of rejected inputs I did not enumerate). ✓ `small_order_public_key_is_rejected_by_strict_verification`: the identity point as the registered public key, `R = identity` and `S = 0` as the signature, an arbitrary payload: the verifier answers `BadSignature`. ✓ The mutation that replaces `verify_strict` by the ordinary `verify` makes this test fail (§14, mutant M43), so the test really does pin the choice. Where the signature is only checked and not stored or used as an identifier, the weaker check is often harmless; where a signature is used as a **transaction id, a cache key or a dedup key**, signature malleability becomes a real bug. Choosing the strict check costs nothing here, so choose it.

### 2.7 Rotation and revocation ✓

Keys are not forever: employees leave, hosts are compromised, policies mandate rotation. A verifier therefore holds **several keys**, each with a validity window and a revoked flag, and the signer names the one it used. A safe rotation is a sequence:

1. Register the **new** public key with a window starting at the switch time (or earlier).
2. Switch the signer to the new key. Envelopes already in flight carry the old key id.
3. Keep the old key valid for **at least the maximum TTL** (here 10 minutes) so that nothing in flight is rejected.
4. Let the old window close, then remove the key.

✓ `rotation_with_overlap`: keys 1 (valid until 2,000) and 2 (valid from 1,000); at 1,500 both verify, with independent nonce counters; at 2,000 key 1's envelope is `KeyNotValidNow(1)` and key 2's still works. **Revocation** is the emergency version: `revoke(key_id)` makes that key refuse everything immediately, before its window ends (✓ `revocation_is_sticky_for_that_key_only`: the other key keeps working). Revocation only helps if the verifier *learns* of it; on a chain, "the verifier" may be a contract that cannot be updated quickly, which is why the maximum lifetime of anything signed (the TTL) is part of the security design, not a detail. Exercise 6 asks for a checker that finds gaps and too-short overlaps in a rotation plan.

### 2.8 Where the key lives

`LocalSigner` keeps the key in process memory. That is acceptable for a lesson and for low-value test keys. For real value the options, in increasing order of protection and cost (from memory of how teams commonly do this; I did not evaluate any product):

- **A key in an environment variable or file** (Lesson 46 §4): readable by anyone who can read the process or its backups. Acceptable only for low-value keys.
- **A cloud key-management service**: the key is created and kept inside the service, which signs on request, with access control and an audit log. The key never enters your process; a stolen credential can still *request signatures* until it is revoked, which is why the policy and rate limits of §2.3 matter even there.
- **A hardware security module**: the same idea with tamper-resistant hardware.
- **Threshold or multi-party signing**: the key is split so that no single machine can sign, and a quorum must cooperate.
- **A multi-signature or smart-contract account** (an on-chain construction, from memory): the chain itself requires several approvals, with limits and delays enforced in code that everyone can inspect.

The `Signer` trait in the crate (`key_id`, `sign`) is the seam where any of these plugs in; the service, the policy and the allocator do not change. Two rules hold for all of them: **the key must never appear in a log, an error message or a `Debug` print** (✓ `debug_output_never_contains_key_bytes` pins the exact `Debug` text, `LocalSigner { key_id: 3, key: <redacted> }`, and a mutation that prints the key bytes fails it), and **the thing that signs must not be reachable by the thing that is easiest to attack**.

---

## 3. Webhooks: authenticated callbacks

### 3.1 The problem

A provider tells your service "the payment arrived" by sending an HTTP request to a public URL. Anyone can send a request to that URL. The service must decide whether the request really came from the provider and whether it is **new**. The standard tool is a **shared secret** and an HMAC (Lesson 48 §5): the provider computes a MAC over the body with the secret and sends it in a header; the receiver recomputes and compares. A MAC alone has two gaps, which the crate closes: a captured request can be **replayed** later, and a naive comparison can **leak** the correct tag.

### 3.2 The header and the signed message ✓

The header is `t=<timestamp in ms>,v1=<64 lowercase hex digits>`, and the MAC is HMAC-SHA256 over `"<timestamp>." ‖ body` (from memory, a scheme of this shape is used by several payment and developer-platform providers; the details of any real provider differ, so read theirs):

```rust
/// Builds the header value `t=<timestamp>,v1=<64 hex digits>`.
pub fn sign(key: &[u8], timestamp_ms: u64, body: &[u8]) -> String {
    let tag = hmac_sha256(key, &signed_message(timestamp_ms, body));
    format!("t={timestamp_ms},v1={}", hex(&tag))
}
```

Putting the timestamp **inside the MAC input** is the point: an attacker who replays an old request with a fresh `t=` changes the signed message and the tag no longer matches (✓ `timestamp_is_covered`). ✓ Known answers: the two test vectors of **RFC 4231** (key `0x0b` repeated 20 times with data `Hi There` gives `b0344c61…cff7`; key `Jefe` with data `what do ya want for nothing?` gives `5bdcc146…3843`, from memory of the RFC and recomputed with Python's `hmac` module), and a header for key `k`, timestamp `1700000000000` and body `{"a":1}` whose tag `4284e27c…9b51` was computed by Python.

### 3.3 Verification: strict header, freshness, constant time ✓

```rust
/// Verifies a delivery against any of `keys` (so that a new key can be added
/// before the old one is removed).
pub fn verify(
    keys: &[&[u8]],
    header: &str,
    body: &[u8],
    now_ms: u64,
    tolerance_ms: u64,
) -> Result<(), WebhookError> {
    let rest = header
        .strip_prefix("t=")
        .ok_or(WebhookError::MalformedHeader)?;
    let (ts, rest) = rest
        .split_once(",v1=")
        .ok_or(WebhookError::MalformedHeader)?;
    if ts.is_empty()
        || (ts.len() > 1 && ts.starts_with('0'))
        || !ts.bytes().all(|c| c.is_ascii_digit())
    {
        return Err(WebhookError::MalformedHeader);
    }
    let timestamp: u64 = ts.parse().map_err(|_| WebhookError::MalformedHeader)?;
    let tag = unhex(rest).ok_or(WebhookError::MalformedHeader)?;
    if now_ms.abs_diff(timestamp) > tolerance_ms {
        return Err(WebhookError::Stale);
    }
    let msg = signed_message(timestamp, body);
    let mut ok = false;
    for key in keys {
        let mut m = <HmacSha256 as KeyInit>::new_from_slice(key).expect("any key length");
        m.update(&msg);
        // `verify_slice` compares in constant time
        ok |= m.verify_slice(&tag).is_ok();
    }
    if ok {
        Ok(())
    } else {
        Err(WebhookError::BadSignature)
    }
}
```

- **Strict parsing.** Exactly `t=`, a decimal timestamp with no sign, no spaces and no leading zeros, then `,v1=` and exactly 64 **lowercase** hex digits. ✓ `malformed_headers` runs fourteen bad shapes (empty, missing comma, empty or leading-zero timestamps, a `+`, a space, a timestamp too large for `u64`, 62 and 66 digits, uppercase, a non-hex character, wrong prefix, leading space), all `MalformedHeader`. Why so strict? The same reason as everywhere in this module: a header with several spellings is a header whose *signed form* is ambiguous.
- **Freshness**: `now.abs_diff(ts) > tolerance` is `Stale`. The check is in **both** directions, because a provider's clock can run ahead as well as behind. ✓ `tolerance_boundaries`: with a tolerance of 1,000 ms the delivery is accepted at ±1,000 and refused at ±1,001. ✓ `stale_is_checked_even_with_valid_mac`: a perfectly valid tag for an old timestamp is `Stale`. Choose the tolerance for your clocks (minutes is typical, from memory), and remember that freshness is a **replay window, not replay prevention**: within the window the same delivery can be replayed, so the handler must still be **idempotent** (§5.2: use the event id inside the body as the idempotency key).
- **Constant-time comparison.** `verify_slice` on the `hmac` crate's `Mac` compares in constant time (from `subtle`). A plain `==` on the tag bytes or on the hex strings stops at the first difference, and an attacker who can measure response times can, in principle, recover a valid tag one byte at a time. Exercise 1 builds `ct_eq` and shows a counter that exposes the early exit; the debugging challenge (§12) contains this fault.

### 3.4 Secret rotation ✓

`verify` takes a **slice of keys** and accepts a delivery if *any* of them produced the tag, and checks **all** of them (it ORs the results instead of stopping at the first success, so the time does not depend on which key matched). ✓ `rotation_accepts_either_key`: deliveries signed with `old` and with `new` both verify against `[new, old]`, and `old` alone is refused once only `[new]` is configured. A safe rotation is: add the new secret to the verifier; ask the provider to switch; wait until no old-key deliveries arrive (or past the tolerance); remove the old secret.

### 3.5 What the webhook receiver must still do

Authenticity says only that the provider sent it. The receiver must also: **limit the body size before reading it** (Lesson 50 §3), **parse it strictly**, **treat its content as an instruction to re-check, not as the truth** when money moves (for a payment notice, query the chain or the provider's API for the actual state before releasing goods: from memory, a common pattern, because a notice can be wrong, late or duplicated), **deduplicate by event id**, **respond quickly** (enqueue the work and acknowledge, which is exactly what §5 is for) and **never log the secret or the full header**.

### 3.6 Cost ✓

Verifying a 1 KiB body takes about 0.91–0.97 µs with one key and 1.61–1.71 µs with two (release build, three runs). Signature verification is far more expensive: ✓ **41.8–43.0 µs** for a strict ed25519 verification of an envelope, and 14.2–22.1 µs to sign one (256-byte payload). A service that verifies a thousand envelopes per second spends about 4% of one core on it; a service that accepts unauthenticated floods and verifies all of them does not, which is one reason the cheap checks come first (§2.5).

---

## 4. The journal: remembering across crashes

### 4.1 Why a log

A process that holds its only copy of important state in memory loses it when it dies. The simplest durable structure is an **append-only log** (a *journal* or *write-ahead log*): to change the state, first append a record that describes the change, then make the change. After a crash, replay the log. Appends are sequential writes, cheap and simple to reason about; no record is ever modified, so there is no "half-updated" row. This is the idea underneath databases (Lesson 46), consensus logs (Lesson 49) and event sourcing.

### 4.2 Records that detect damage ✓

A crash during an append can leave a **torn** record at the end of the file: the length was written but the payload only partly, or a few bytes of it are garbage. Disks and file systems can also corrupt data silently. Each record therefore carries a **length** and a **checksum**, and recovery keeps the **longest valid prefix**:

```rust
/// Reads records until the first one that is truncated, too long or fails its
/// checksum. Returns the records and the number of valid bytes.
pub fn recover(bytes: &[u8]) -> (Vec<Vec<u8>>, usize) {
    let mut records = Vec::new();
    let mut pos = 0;
    while pos + 4 <= bytes.len() {
        let len_be: [u8; 4] = bytes[pos..pos + 4].try_into().expect("4 bytes");
        let len = u32::from_be_bytes(len_be) as usize;
        if len > MAX_RECORD {
            break;
        }
        let end = pos + 4 + len + 4;
        if end > bytes.len() {
            break;
        }
        let payload = &bytes[pos + 4..pos + 4 + len];
        if bytes[pos + 4 + len..end] != checksum(len_be, payload) {
            break;
        }
        records.push(payload.to_vec());
        pos = end;
    }
    (records, pos)
}
```

The record is `len:u32 ‖ payload ‖ check:4`, where `check` is the first four bytes of `SHA-256("lab/journal/v1" ‖ len ‖ payload)`. A 4-byte truncated hash is a **checksum for accidents**, not an authenticator: anyone who can write the file can recompute it, and with only four bytes a random corrupt record passes with probability 2⁻³². If an attacker can modify the journal, use a MAC under a key they do not have (§3). ✓ Tests (`tests/journal_tests.rs`):

- `checksum_known_answers`: the bytes of the journal holding `abc` are `00 00 00 03 61 62 63 02 88 9c 56` and of the empty payload `00 00 00 00 e6 66 a6 79`, computed with Python's `hashlib`. Without a known-answer test, a checksum that silently drops the domain label or the length would pass every round-trip test (it did, for a mutant I made, until I wrote this test: §14).
- `truncation_at_every_offset_yields_a_prefix`: a journal of four records (including an empty one and one of 300 bytes) is cut at **every** byte offset from 0 to its length; for each cut, recovery returns exactly the records that fit entirely and reports the right number of valid bytes.
- `every_single_bit_flip_is_detected`: every one of the 8 × length bit flips is noticed, in the sense that the recovered list is a **strict prefix** of the original. That is stronger than "the checksum fails somewhere": it also shows that a flipped length field cannot be used to swallow later records.
- `oversize_length_stops_recovery` and `oversize_record_with_a_valid_checksum_is_still_rejected`: a record longer than `MAX_RECORD` (1 MiB) is refused *even if its checksum is correct*, because the limit is a rule about what we are willing to allocate while recovering.
- `from_bytes_drops_garbage_tail_so_appends_stay_valid`: after recovery the journal is truncated to its valid prefix, so the next append does not land after garbage (which would make everything after it unreachable on the *next* recovery).

### 4.3 What a real journal needs that this one does not

The crate's journal is a `Vec<u8>`. On a real disk, durability is a separate and subtle subject (from memory, and worth reading about in the documentation of your file system and database): an `append` is not durable until the data has been flushed (`File::sync_data` or `sync_all` in Rust) **and**, for a newly created file, the *directory entry* has been flushed too; drives and operating systems may reorder or cache writes unless told otherwise; and a successful `write` only means the kernel accepted the bytes. A real implementation must also decide **when to sync** (every record is safe and slow; batching is fast and loses the batch on a crash: *group commit*), handle **disk full**, **rotate** files, and **compact** old records (exercise 7). The shape of the recovery logic, "read until the first invalid record and stop", does not change. In most services the right answer is not to write this yourself but to use a database that already does (Lesson 46), and to keep the *lessons* of this section: a record format that detects damage, recovery that keeps a prefix, and replay through one code path.

---

## 5. A durable job queue (an outbox)

### 5.1 What problem it solves

When a request arrives ("send this payment"), the service must do two things: **remember the work** and **do the work**. If it does the work first and crashes, the work is done but unrecorded (so it may be done again); if it does neither atomically, it can lose requests. The usual structure is the **outbox**: write the *intent* durably first, acknowledge the caller, and let a separate loop perform the work and record the outcome. The loop may crash at any point, so it must be safe to run the same job more than once: delivery is **at least once**, and correctness comes from **idempotency** (§5.2, §7).

### 5.2 Idempotency keys ✓

The caller gives each logical job a **key** (an order id, a payment reference). `enqueue(key, payload, now)` returns `New(id)` the first time and `Duplicate(id)` for the same key afterwards, **without changing anything**. ✓ `duplicate_keys_are_ignored`: a second `enqueue` with different payload and a later time is `Duplicate` and leaves one pending job; the simulation enqueues every job twice, as a producer that retried its HTTP request would, and requires the second to be `Duplicate`. This is [Lesson 45](45_http_apis.md)'s idempotency key applied to the queue. Two more points: the key must come from the *meaning* of the request (exercise 3 shows why plain concatenation of fields is ambiguous, `("ab","c")` versus `("a","bc")`), and a duplicate with a **different payload** is a client bug that a real system should report instead of silently ignoring (the crate ignores it and says so).

### 5.3 Events, leases and tokens ✓

Every change to the queue is one of five **events**, and the events are the only thing ever written to the journal:

```rust
/// A state change; the only thing ever written to the journal.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Event {
    /// A job was added.
    Enqueued {
        /// Its id.
        id: JobId,
        /// Idempotency key.
        key: u64,
        /// Earliest start time.
        ready_at: u64,
        /// The work description.
        payload: Vec<u8>,
    },
    /// A worker took the job until `until`.
    Leased {
        /// Job.
        id: JobId,
        /// Lease expiry.
        until: u64,
    },
    /// The worker finished it.
    Completed {
        /// Job.
        id: JobId,
    },
    /// The attempt failed; retry at `retry_at`.
    Failed {
        /// Job.
        id: JobId,
        /// Earliest next attempt.
        retry_at: u64,
    },
    /// The job is given up.
    Died {
        /// Job.
        id: JobId,
    },
}
```

A worker takes a job with `lease(now, lease_ms)`. The job is *leased until* a time; if the worker neither completes nor fails it by then (it crashed, or hung), the job becomes available again: **the lease expires and the job is handed to the next worker**. This is the same idea as a timeout on a lock (Lesson 29), with the difference that the previous holder may still be alive, slow, and about to finish. The defense is a **fencing token**: each lease increments the job's attempt counter, the lease carries that number, and `complete(id, attempt)` and `fail(id, attempt, …)` check that it is still the current one.

```rust
/// Takes the oldest job that is ready (pending and due, or leased with an
/// expired lease). Exhausted jobs are reaped first.
pub fn lease(&mut self, now_ms: u64, lease_ms: u64) -> Option<Lease> {
    self.reap(now_ms);
    let id = self.jobs.iter().find_map(|(id, j)| {
        let ready = match j.status {
            Status::Pending { ready_at } => ready_at <= now_ms,
            Status::Leased { until } => until <= now_ms,
            Status::Done | Status::Dead => false,
        };
        ready.then_some(*id)
    })?;
    self.record(Event::Leased {
        id,
        until: now_ms.saturating_add(lease_ms),
    });
    let j = &self.jobs[&id];
    Some(Lease {
        id,
        payload: j.payload.clone(),
        attempt: j.attempts,
    })
}
```

✓ `expired_lease_is_retaken_and_old_holder_is_stale`: the first holder's lease runs out at 100; a second `lease` at 100 gets attempt 2; the first holder's `complete` and `fail` are refused with `StaleLease`; the second holder's `complete` is accepted. A completion that arrives *again* from the same holder is accepted as a no-op (✓ `complete_and_fail_errors`), because the caller may have retried the acknowledgement. Other outcomes: `Unknown` for a missing job and `NotLeased` for a job in any other state.

**Retries and the dead-letter state.** `fail(id, attempt, now, backoff)` schedules a retry at `now + backoff` (✓ `failure_backs_off_then_retries`) or, when `attempt >= max_attempts`, moves the job to `Dead` (✓ `dead_letter_after_max_attempts`: three attempts, then dead, and never leased again). A job whose last lease *expires* without any report is also declared dead, by `reap` (✓ `lease_that_expires_after_last_attempt_is_dead`, `reap_only_touches_exhausted_expired_leases`): otherwise a job that crashes its worker every time would sit "leased" for ever. The dead state is not an error to hide; it is a **queue for humans**, with an alert. The backoff itself comes from the caller (§7 uses a jittered exponential, Lesson 51 §5.2).

### 5.4 One code path for live and recovered state ✓

```rust
fn apply(&mut self, e: &Event) {
    match e {
        Event::Enqueued {
            id,
            key,
            ready_at,
            payload,
        } => {
            self.jobs.insert(
                *id,
                Job {
                    key: *key,
                    payload: payload.clone(),
                    attempts: 0,
                    status: Status::Pending {
                        ready_at: *ready_at,
                    },
                },
            );
            self.by_key.insert(*key, *id);
            self.next_id = self.next_id.max(id + 1);
        }
        Event::Leased { id, until } => {
            if let Some(j) = self.jobs.get_mut(id) {
                j.attempts += 1;
                j.status = Status::Leased { until: *until };
            }
        }
        Event::Completed { id } => {
            if let Some(j) = self.jobs.get_mut(id) {
                j.status = Status::Done;
            }
        }
        Event::Failed { id, retry_at } => {
            if let Some(j) = self.jobs.get_mut(id) {
                j.status = Status::Pending {
                    ready_at: *retry_at,
                };
            }
        }
        Event::Died { id } => {
            if let Some(j) = self.jobs.get_mut(id) {
                j.status = Status::Dead;
            }
        }
    }
}
```

`record(e)` appends the event to the journal and then calls `apply(e)`; `Queue::recover(bytes)` reads the valid records, decodes them and calls the **same** `apply` for each. There is no separate "recovery logic" that could drift from the live logic. ✓ `recovery_equals_live_state`: after six enqueues, a completion, a failure and a lease, the recovered queue has the same statistics, statuses, attempt counts, keys and **the same journal bytes** as the live one. ✓ `recovery_keeps_id_allocation_and_dedup`: after recovery the next id is correct and a repeated key is still a `Duplicate`. ✓ `recovery_from_every_truncation_gives_a_consistent_prefix`: recover from the journal cut at every byte; the number of known jobs never decreases as the cut moves right, and it ends at all four. ✓ `corrupted_tail_is_ignored_and_queue_continues` and `undecodable_record_stops_replay`: a damaged final record loses only that record, and the recovered queue can keep appending.

Event decoding is as strict as everything else: unknown tags, short input, **trailing bytes** and over-long payloads are `None` (✓ `event_encoding_round_trip_and_strict` cuts every encoding at every length). This matters more than it seems, because the journal is the one input that this program *trusts*, and trusted inputs are where a lax parser hurts later.

**Write-ahead order.** `record` journals before it applies in the sense of the protocol; in the crate both happen in one call and one thread, so the order is not observable (a mutation that swaps them is *equivalent*, §14). In a real system, with the journal on disk and the state in a database, the order and the sync are the whole point: **the intent must be durable before the caller is told "accepted"**, and the effect must be recorded durably before the acknowledgement to the next stage.

### 5.5 What this queue is not

It is a single-threaded, in-memory structure with a linear scan to find the next job (`lease` is O(jobs); a real queue uses an index by ready time), no priorities, no per-job timeouts beyond the lease, no batching and no persistence beyond what the caller does with `journal_bytes()`. It compares to a table in a database with a `FOR UPDATE SKIP LOCKED` style query or a managed queue (from memory, the two usual choices in practice); what carries over is the *contract*: idempotent enqueue, leases with fencing tokens, bounded retries, a dead-letter state, and a state that can be rebuilt from an event log.

### 5.6 Cost ✓

Enqueue, lease and complete one job in a fresh queue takes about 0.45–0.71 µs; appending a 64-byte journal record about 128–154 ns; recovering a journal of 20,000 such records (about 1.4 MB) 3.05–3.49 ms, or roughly 150–175 ns per record, probably mostly the SHA-256 per record (I did not profile it; release build, three runs). Recovery time is linear in the log, which is why real systems **snapshot** and truncate (exercise 7).

---

## 6. Fault tolerance: breakers, backoff and the failures that stay hidden

### 6.1 A circuit breaker ✓

Retries help against brief faults and make **long** faults worse: while the provider is down, every new job waits through its retries, the queue fills, and when the provider recovers it is hit by the whole backlog at once. A **circuit breaker** stops calling a dependency that is clearly failing, then tests it with a single call after a pause:

```rust
/// Counts consecutive failures; opens at a threshold; probes after a pause.
#[derive(Debug, Clone)]
pub struct Breaker {
    threshold: u32,
    open_ms: u64,
    failures: u32,
    opened_at: Option<u64>,
    probing_since: Option<u64>,
}

impl Breaker {
    /// Opens after `threshold` consecutive failures and stays open `open_ms`.
    pub fn new(threshold: u32, open_ms: u64) -> Breaker {
        assert!(threshold > 0, "threshold must be positive");
        Breaker {
            threshold,
            open_ms,
            failures: 0,
            opened_at: None,
            probing_since: None,
        }
    }

    /// The state at `now_ms`.
    pub fn state(&self, now_ms: u64) -> State {
        match self.opened_at {
            None => State::Closed,
            Some(t) if now_ms.saturating_sub(t) < self.open_ms => State::Open,
            Some(_) => State::HalfOpen,
        }
    }

    /// May a call be made now? In the half-open state only one trial is
    /// allowed until its result is recorded; a trial whose result never
    /// arrives is forgotten after another `open_ms`, so a lost probe cannot
    /// keep the breaker shut forever.
    pub fn allow(&mut self, now_ms: u64) -> bool {
        match self.state(now_ms) {
            State::Closed => true,
            State::Open => false,
            State::HalfOpen => match self.probing_since {
                Some(t) if now_ms.saturating_sub(t) < self.open_ms => false,
                _ => {
                    self.probing_since = Some(now_ms);
                    true
                }
            },
        }
    }

    /// Records a successful call.
    pub fn record_success(&mut self) {
        self.failures = 0;
        self.opened_at = None;
        self.probing_since = None;
    }

    /// Records a failed call.
    pub fn record_failure(&mut self, now_ms: u64) {
        if self.opened_at.is_some() {
            // a failed trial reopens for a full cool-down
            self.opened_at = Some(now_ms);
            self.probing_since = None;
            return;
        }
        self.failures += 1;
        if self.failures >= self.threshold {
            self.opened_at = Some(now_ms);
        }
    }
}
```

States (from the common pattern): **Closed** (calls flow; consecutive failures are counted), **Open** (calls are refused for `open_ms`), **HalfOpen** (after the pause, *one* trial call decides: success closes the circuit and clears the count; failure reopens it for a full pause). Time is an argument (`now_ms`), as in Lesson 50, so the tests need no clock. ✓ Tests (`tests/breaker_tests.rs`): it opens exactly at the threshold, not before; a success resets the count (two failures, a success, two failures stays closed); it is `Open` at `opened + open_ms − 1` and `HalfOpen` at `opened + open_ms`; only the first `allow` in the half-open state returns true; a failed trial reopens with the *full* cool-down measured from the failure; a zero threshold panics.

What a breaker does **not** do: choose good thresholds (too sensitive and it flaps, too lax and it never helps), distinguish *kinds* of failure (a 4xx for a bad request is not evidence that the provider is down), or protect a dependency that fails slowly (use timeouts, Lesson 50 §2.4, and count a timeout as a failure). It also fails **closed to the caller**: while it is open, work must wait somewhere, which is exactly what the durable queue gives you.

### 6.2 Backoff, jitter and deadlines

The relayer simulation retries failed jobs with the **full jitter** of Lesson 51 §5.2: after attempt *a* it waits a random time in `0..=min(2000, 50 · 2^(a−1))` ms, drawn from the seeded generator `Rng` (xorshift64\*; ✓ the first outputs for seed 1, `5180492295206395165`, `12380297144915551517`, `13389498078930870103`, `5599127315341312413`, and for seed 0 the first `973819730272012410`, were computed by an independent Python program). The queue stores the scheduled retry time as `ready_at`, so the schedule survives a restart. Remember the rule from Lesson 51 §5.3 about **writes**: a retry of a submission is only safe if it is idempotent, which is the subject of §7.

### 6.3 The breaker that never closed (a bug I wrote) ✓

My first version of the relayer loop asked the breaker for permission **before** it checked whether there was anything to send:

```text
if !breaker.allow(now) { skipped += 1 }
else if let Some(job) = queue.lease(now, ..) { ... }
```

In the half-open state `allow` grants **one** trial permit and then refuses everyone until the trial's result is recorded. If the permit was granted at a moment when every job was waiting out its backoff, `lease` returned nothing, **no call was made, no result was ever recorded, and the permit was never given back**. The breaker stayed half-open with its trial "in flight" for ever. ✓ Over 200 seeds, 2 (seeds 113 and 171) ended at the 600,000 ms time limit with one or two jobs undone and about 59,000 refused loop iterations each; `cargo test` reported `left: 49, right: 50`. A bug in a *test harness* revealed a bug in the *design* of the breaker's API, and it is a classic: **a permit that is acquired must be released on every path**.

Two fixes, both kept. First, the loop now asks `queue.has_ready(now)` before it asks the breaker, so a permit is only taken when a call will be made (✓ the 200-seed property test). Second, the breaker no longer trusts its callers: the permit is a **timestamp** (`probing_since`), and a trial that has not reported after another `open_ms` is forgotten and a new one is granted (✓ `lost_probe_is_forgotten_after_another_cooldown`: permit at 100, refused at 150 and 199, a new permit at 200). Real callers also crash, panic and get cancelled between `allow` and `record_*`; the second fix is the one that survives that. The mutation run showed an interesting consequence: the *explicit* clearing of the flag after a success or a failed trial became **redundant** once permits expire (mutants M87 and M88 are equivalent, §14). I left the clearing in, because it makes the intent obvious, and because redundant defenses are cheap.

### 6.4 Other failure modes this crate does not model

Partial failures with **no error** (a provider returning stale data with success), **clock jumps** (NTP steps, virtual machines paused and resumed), **overload** that makes every request slow instead of failing, **retry storms** across many service replicas (jitter helps; so do shared budgets), **poison jobs** that crash the process every time (the dead-letter state and the attempt limit are the defense), **deploys** that restart the process while jobs are leased (leases expire; that is what they are for), and **split brain**, where two instances both think they are the only relayer and both sign. The last one is worth a sentence: with a nonce reservation per instance and a leader lease, only one instance should be allowed to sign for a key; see exercise 5 for the crash points of a single instance, and note that the queue's fencing token protects the *job*, not the *key*.

---

## 7. The relayer: putting it together

### 7.1 The loop

`sim::run(seed, params)` simulates a relayer and a chain for up to 600 simulated seconds, with a tick of 10 ms. At each tick: if a job is ready and the breaker allows, **lease** it; **sign** its payload into an envelope with the next nonce (storing a nonce reservation first if one is pending); **submit** the envelope bytes to the simulated chain; then record the outcome: success → `complete` and `breaker.record_success()`; failure → `breaker.record_failure()` and `fail` with a jittered backoff. After each submission, with a configurable probability, the relayer **crashes between the submission and the recording of its result**; then the queue is rebuilt from the journal bytes (`Queue::recover`), the signing service is rebuilt from the last stored nonce reservation, and the breaker starts fresh.

```rust
/// Runs the simulation.
pub fn run(seed: u64, p: &Params) -> Report {
    let mut rng = Rng::new(seed);
    let signer = LocalSigner::from_seed([5; 32], 1);
    let mut verifier = Verifier::new(CHAIN_ID);
    verifier.add_key(
        1,
        KeyEntry {
            public: signer.public(),
            valid_from_ms: 0,
            valid_until_ms: u64::MAX,
            revoked: false,
        },
    );
    let mut chain = SimChain {
        effects: BTreeMap::new(),
        dedupe: p.dedupe,
        unavailable_pm: p.unavailable_pm,
        timeout_after_pm: p.timeout_after_pm,
        verifier,
        rng: Rng::new(seed ^ 0xA5A5_A5A5),
        rejected: 0,
    };
    let mut queue = Queue::new(p.max_attempts);
    for job in 0..p.jobs {
        queue.enqueue(job, payload_for(job), 0);
        // a producer that retried its request: the duplicate is ignored
        assert!(matches!(
            queue.enqueue(job, payload_for(job), 0),
            Enqueued::Duplicate(_)
        ));
    }
    let mut nonce_log: Vec<u64> = Vec::new();
    let new_service = |log: &[u64]| {
        SigningService::new(
            LocalSigner::from_seed([5; 32], 1),
            MaxAmount(1_000_000),
            CHAIN_ID,
            log.last().copied().unwrap_or(0),
            16,
        )
    };
    let mut service = new_service(&nonce_log);
    let mut breaker = Breaker::new(5, 500);
    let (mut submissions, mut crashes, mut skipped, mut total_wait) = (0u64, 0u64, 0u64, 0u64);
    let mut now = 0u64;
    loop {
        let st = queue.stats();
        if st.pending == 0 && st.leased == 0 {
            break;
        }
        if now > p.time_limit_ms {
            break;
        }
        queue.reap(now);
        if !queue.has_ready(now) {
            // nothing to send: do not ask the breaker for a permit we will not use
        } else if !breaker.allow(now) {
            skipped += 1;
        } else if let Some(lease) = queue.lease(now, 1_000) {
            let env = service
                .sign(lease.payload.clone(), now, 30_000)
                .expect("policy allows");
            if let Some(res) = service.take_reservation() {
                nonce_log.push(res); // stored durably before the envelope leaves
            }
            submissions += 1;
            let result = chain.submit(&env.encode(), tx_id(&lease.payload), now);
            // a crash between the submission and recording its result
            let crash_here = rng.chance(p.crash_pm);
            if !crash_here {
                match result {
                    Ok(_) => {
                        breaker.record_success();
                        queue.complete(lease.id, lease.attempt).expect("holder");
                    }
                    Err(_) => {
                        breaker.record_failure(now);
                        let ceiling = (50u64 << lease.attempt.saturating_sub(1).min(16)).min(2_000);
                        let wait = rng.below(ceiling + 1);
                        total_wait += wait;
                        let _ = queue
                            .fail(lease.id, lease.attempt, now, wait)
                            .expect("holder");
                    }
                }
            } else {
                crashes += 1;
                queue = Queue::recover(queue.journal_bytes(), p.max_attempts);
                service = new_service(&nonce_log);
                breaker = Breaker::new(5, 500);
            }
        }
        now += 10;
    }
    let st = queue.stats();
    Report {
        done: st.done as u64,
        dead: st.dead as u64,
        distinct_effects: chain.effects.len() as u64,
        double_applied: chain.effects.values().filter(|&&c| c > 1).count() as u64,
        submissions,
        crashes,
        skipped_by_breaker: skipped,
        rejected: chain.rejected,
        total_wait_ms: total_wait,
        nonce_high_water: nonce_log.last().copied().unwrap_or(0),
        finished_at: now,
    }
}
```

The simulated chain (`SimChain`) verifies each envelope with the real `Verifier` (so the signing path is exercised end to end), applies the transaction identified by `tx_id = SHA-256("lab/relay-tx/" ‖ payload)`, and injects two kinds of fault from its own seeded generator: **`Unavailable`**, a refusal *before* processing (nothing happened), and **`Timeout`**, no answer *after* processing (the transaction **was** applied but the relayer does not know). With `dedupe: true` the chain ignores a transaction id it already applied (`AlreadyKnown`, which counts as success); with `dedupe: false` it applies it again.

### 7.2 Timeouts that lie ✓

The `Timeout` fault is the whole difficulty. From the relayer's point of view, "no answer" and "the chain did it but the answer was lost" look identical. The only safe responses are to **check before repeating** (ask the chain whether the transaction id is known) or to make repeating **harmless** (the same bytes, the same id, a chain that deduplicates). ✓ Default parameters (50 jobs, 30% `Unavailable`, 20% lost answers, 3% crashes per tick, 30 attempts, 200 seeds):

| chain | seeds with a double-applied transaction | double-applied transactions in total |
|---|---|---|
| deduplicates by transaction id | **0 of 200** | **0** |
| does not deduplicate | **200 of 200** | **2,192** (up to 21 in one run of 50 jobs) |

✓ The same experiment with crashes switched off still gives 1,943 double-applied transactions for the blind chain, and with unavailability and crashes both off but lost answers on, 1,955 (every seed): `timeouts_alone_double_apply_on_a_blind_chain_but_not_on_a_deduplicating_one` pins it. So the lost answer alone is enough, and crashes add a second, independent way to the same result (a crash between submit and record, which exercise 5 enumerates).

### 7.3 Exactly once, in effect ✓

No network gives you exactly-once *delivery*. What you can build is **at-least-once delivery plus an idempotent effect**, whose result is indistinguishable from exactly-once. In this crate the idempotent effect is the chain's deduplication by transaction id, and the transaction id is the hash of the payload, which is **a function of the job** and not of the attempt. That is why every retry of a job re-signs the same payload with a *new nonce* (the envelope is new) but submits the *same transaction* (same id). ✓ `chaos_with_dedupe_is_exactly_once_effect`: for each of 200 seeds, `done == 50`, `dead == 0`, `distinct_effects == 50`, `double_applied == 0` and the chain rejected **no** envelope (`rejected == 0`, i.e. the verifier never saw a replay, a bad signature or an expired message, despite the restarts). ✓ `chaos_actually_exercises_the_faults` guards against a vacuous pass: over 50 seeds every run submits more than 50 times, there are more than 100 crashes in total, the breaker refuses some iterations, and the nonce high-water mark is at least the number of jobs. ✓ Over the 200 default seeds: 72–116 submissions per run (mean 91.3 for 50 jobs), 3.0 crashes and 71.1 breaker refusals on average, a mean of 2,922 simulated ms and at most 12,190 ms to finish.

### 7.4 Nonce gaps from crashes ✓

Each restart abandons the unused part of the current nonce block (§2.4). ✓ `a_relayer_that_restarts_after_every_submission_burns_a_whole_nonce_block_each_time`: with a crash after every submission the high-water mark is exactly `16 ×` the number of submissions; ✓ `without_crashes_the_nonce_block_is_used_up`: 40 jobs end at 48 (three blocks of 16). And yet ✓ `rejected == 0`: the verifier only demands that nonces **increase**, so gaps are free. Reuse would have shown up as a `Replay` rejection, which is why that counter is asserted to be zero.

### 7.5 More properties pinned by tests ✓

- `no_faults_each_job_once`: 20 jobs, no faults, one submission each.
- `deterministic_per_seed`: the same seed gives the same report; a different seed does not.
- `hopeless_chain_ends_in_dead_letters`: with every submission refused and 3 attempts, all 5 jobs end `Dead` and the breaker refused some iterations.
- `jobs_whose_every_attempt_ends_in_a_crash_become_dead_letters`: with a crash after every submission and 2 attempts, all 4 jobs end `Dead` and the run finishes in well under the time limit; this is `reap` at work.
- `the_time_limit_stops_a_run_that_cannot_finish`: the loop honors its limit.
- `a_breaker_that_forgets_failures_on_success_stays_closed_at_a_low_fault_rate`: at 5% refusals the breaker must not open in 20 seeds of 100 jobs, which holds only if successes reset the failure count.
- `backoff_waits_are_scheduled_only_after_failures`.

### 7.6 Cost ✓

One whole simulation of 50 jobs with chaos (on average about 91 signatures and verifications and 3 recoveries) takes 4.4–5.2 ms in a release build, which is dominated by the ed25519 operations (roughly 50–60 µs per submission, consistent with one signature at 14–22 µs plus one verification at about 42 µs). The test suite of the crate, with dependencies compiled at optimization level 3 in the dev profile (`[profile.dev.package."*"] opt-level = 3`), runs in about two seconds, and the 200-seed property test is most of that. Without that profile line the same suite took **99 seconds** in a debug build, almost all of it ed25519 in unoptimized code. That setting is a good habit for any project that tests cryptographic code.

---

## 8. Rust notes

- **Traits for seams**: `Signer` and `Policy` are the two extension points of the signing service; the service is generic over them (`SigningService<S, P>`), so tests use a local key and the production build could use a remote signer with no change to the logic. Generic parameters monomorphize (Lesson 21), which costs nothing at runtime; a `dyn Signer` would also work when the choice is made at run time.
- **One `apply`, many callers**: the queue's `apply(&Event)` takes `&mut self` and a borrowed event. It is private and deterministic; it is the only place that changes job state. Pure state-transition functions like this are the easiest code to test and to replay.
- **Errors as enums** (Lesson 26): `VerifyError` has one variant per reason (`Replay { last, got }` carries the evidence), `QueueError` three, `DecodeError` three, `WebhookError` three. Tests compare by value. Notice that `Debug` for `VerifyError` and `SignError` never contains secret material by construction (they hold counters and ids, never keys).
- **`Debug` and secrets**: `LocalSigner` implements `Debug` by hand and prints `<redacted>` instead of deriving it. The `ed25519-dalek` `SigningKey` type does not implement `Debug` in a way that would print the secret (I did not check the exact output); the hand-written impl makes the promise local, visible and testable (✓ the test compares the whole string). Consider crates such as `secrecy` or `zeroize` (from memory: wrappers that avoid accidental printing and clear memory on drop) for real keys; I did not use them here.
- **Constant-time APIs**: `Mac::verify_slice` is the right way to compare a MAC in the `hmac` crate (✓ used); `subtle::ConstantTimeEq` is the general tool (exercise 1 uses a hand-written loop to show the idea; prefer the library). Constant-time Rust is hard to *guarantee* because the optimizer may rewrite your loop; libraries take care (from memory: with barriers and, where available, tested assembly), which is another reason not to write your own.
- **`u64` arithmetic and time**: all times are `u64` milliseconds with `saturating_add`/`abs_diff` where overflow is possible, and the one raw subtraction (`env.expires_ms - now_ms`) is guarded by the earlier `now_ms >= env.expires_ms` check. Use `Duration` and `Instant` at real boundaries.
- **Dev-profile dependencies**: `[profile.dev.package."*"] opt-level = 3` compiles dependencies optimized and your own crate unoptimized (with debug assertions and overflow checks on), a good trade for crypto-heavy tests.
- **No `unsafe`, no `async`**: all state machines are synchronous and take time as an argument; an async runtime (Lesson 31) would call them from tasks. Be careful about holding a `std::sync::Mutex` across `.await` (Lesson 29); the queue should live behind one task or a lock held only for the duration of a method.

---

## 9. Common mistakes

1. Signing arbitrary bytes without a domain, a chain id, an expiry and a nonce inside them (§2.1, §2.2).
2. A signing service with no policy, so that every bug upstream becomes a payment (§2.3).
3. Keeping the next nonce only in memory, so that a restart reuses numbers (§2.4).
4. Recording the replay nonce **before** checking the signature, which lets a forger lock out the real sender (§2.5).
5. Accepting envelopes with unbounded lifetimes (§2.5).
6. Using the permissive signature check where signatures act as identifiers (§2.6).
7. A single key with no rotation plan, or a rotation with no overlap (§2.7).
8. Comparing MACs, tokens or API keys with `==` (§3.3).
9. A webhook MAC that does not cover the timestamp, or a timestamp that is never checked (§3.2, §3.3).
10. Trusting a webhook's content instead of re-checking the state when money moves (§3.5).
11. Treating the checksum of a journal as authentication (§4.2).
12. Replaying a log with different code than the live path, so recovered state differs from live state (§5.4).
13. Letting a worker complete a job after its lease was given to another worker (§5.3).
14. Retrying a timed-out write with a **new** transaction identity (§7.2).
15. A circuit-breaker permit that is acquired and not released on every path (§6.3).
16. A queue with no dead-letter state, so one poison job loops for ever (§5.3).
17. No test that kills the process (or simulates it) between "did the work" and "recorded the work" (§7.2, exercise 5).
18. Logging keys, full headers or signed payloads (§2.8, §3.5).
19. Believing a green test suite means the system is safe (§10).

---

## 10. Security notes and honest limits

- `relaylab` is teaching code. It has **no TLS, no HTTP, no authentication of callers to the signing service, no authorization beyond the amount policy, no rate limiting, no secure memory and no durable storage**. The signing service in a real system must authenticate its callers (who is allowed to ask for a signature?), log every request and decision to an append-only store the service cannot edit, and apply **cumulative** limits (per day, per destination), not only a per-request cap.
- **A signing oracle is a credential.** Anyone who can call `sign` can get signatures for anything the policy allows. The policy limits the damage; it does not remove the need to protect the call path.
- **Replay protection is only as good as the verifier's memory.** The `Verifier` keeps the last nonce per key in memory. If the verifier restarts and forgets, replay is possible until the TTL expires: that is another reason for the TTL cap, and a real verifier would persist the counter (§4–§5) or rely on a chain's own account nonces.
- **HMAC authenticates the sender among those who hold the secret.** Both the provider and you can produce a valid tag, so a MAC cannot prove to a third party who sent a message (for that you need a signature, §2). Keep webhook secrets in a secret store, not in source control (Lesson 46 §4; Lesson 43 for supply-chain concerns).
- **The 4-byte journal checksum catches accidents, not adversaries** (§4.2). A journal that an attacker can modify needs a MAC or must live in storage they cannot reach.
- **Idempotency is only as strong as the key.** If two different jobs get the same key, the second is silently dropped; if the same job gets two keys, it runs twice. Generating keys is part of the design (exercise 3).
- **A deterministic simulation proves the logic against the faults you scripted.** Real systems also meet slow disks, partial writes, reordered writes, clock steps, memory pressure, and humans. Add fault injection at the real boundaries (kill -9 tests, network partition tools, disk-full tests) and watch production with the metrics of Lesson 43: queue depth and oldest-job age, dead-letter count, breaker state changes, signature failures and nonce-gap size.
- **Mutation results are not proof of correctness** (§14).

---

## 11. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Write `ct_eq(a, b)`: true iff the slices are equal, looking at every byte when the lengths match. Why is it acceptable to return early when the lengths differ?
2. **(Easy)** Write a **replay window** that accepts a nonce at most once, accepts nonces that arrive out of order within the 64 most recent, and refuses anything older than that.
3. **(Easy)** Write `idempotency_key(order_id, recipient, amount)` so that two different requests can never collide because of how the fields run together. Show that plain concatenation collides.
4. **(Medium)** Write a **token bucket** with injected time: capacity *c*, refill rate *r* tokens per second, `try_take(now_ms)`. It must not use floating point and must cope with time going backwards.
5. **(Medium)** A relayer does three steps: (1) journal the intent, (2) submit to the chain, (3) journal the result. List the state after a crash at each of the four possible points and say what recovery must do for a chain that deduplicates and for one that does not.
6. **(Medium-hard)** Write `rotation_ok(windows, start, end, min_overlap)`: true iff the key windows `[from, until)` cover `[start, end)` without a gap and each hand-over overlaps by at least `min_overlap`.
7. **(Hard)** Write a **compaction** for the balance log of a ledger (events `Add(account, n)` and `Sub(account, n)`, where a debit that would underflow is ignored): produce a snapshot of the first `k` events plus the remaining tail, and show that replaying the snapshot and the tail equals replaying everything, for every `k`.
8. **(Hard)** Write a **health window**: the last *n* outcomes in a ring buffer, `failures()` and `unhealthy(max_failures)` (unhealthy only once the window is full). When is this a better signal than "consecutive failures" for a breaker?

<details markdown="1">
<summary>Answers</summary>

1. A possible answer (✓ compiled and run in a scratch crate with tests):

   ```rust
   //! Exercise 1: a comparison whose running time does not depend on where the
   //! first difference is.
   use std::hint::black_box;

   /// Equal length and equal bytes; always looks at every byte of `a`.
   pub fn ct_eq(a: &[u8], b: &[u8]) -> bool {
       if a.len() != b.len() {
           return false; // lengths are public
       }
       let mut diff = 0u8;
       for (x, y) in a.iter().zip(b) {
           diff |= x ^ y;
       }
       black_box(diff) == 0
   }

   /// The leaky version, kept to show what to avoid.
   pub fn leaky_eq(a: &[u8], b: &[u8]) -> (bool, usize) {
       let mut looked = 0;
       if a.len() != b.len() {
           return (false, looked);
       }
       for (x, y) in a.iter().zip(b) {
           looked += 1;
           if x != y {
               return (false, looked);
           }
       }
       (true, looked)
   }
   ```

   The test also runs the leaky version with a counter: `leaky_eq(b"xbc", b"abc")` looked at 1 byte, `leaky_eq(b"abx", b"abc")` at 3, and the match at 3, which is the timing signal an attacker uses. Returning early on different **lengths** is acceptable because the length of a tag is public (it is fixed by the algorithm); the secret is the content. `black_box` on the accumulated difference is a hint to the optimizer, not a guarantee; real code uses `subtle` (§8).
2. A possible answer (✓ compiled and run):

   ```rust
   //! Exercise 2: accept nonces out of order inside a 64-wide window, never twice.

   /// Sliding replay window anchored at the highest nonce seen.
   #[derive(Debug, Default)]
   pub struct ReplayWindow {
       highest: Option<u64>,
       seen: u64, // bit i set = (highest - i) was seen
   }

   /// Verdict for a nonce.
   #[derive(Debug, PartialEq, Eq)]
   pub enum Verdict {
       /// First time seen (and now recorded).
       Fresh,
       /// Seen before.
       Duplicate,
       /// Older than the window: cannot tell, so refuse.
       TooOld,
   }

   impl ReplayWindow {
       /// Checks and records `n`.
       pub fn check(&mut self, n: u64) -> Verdict {
           let Some(h) = self.highest else {
               self.highest = Some(n);
               self.seen = 1;
               return Verdict::Fresh;
           };
           if n > h {
               let shift = n - h;
               self.seen = if shift >= 64 { 0 } else { self.seen << shift };
               self.seen |= 1;
               self.highest = Some(n);
               return Verdict::Fresh;
           }
           let back = h - n;
           if back >= 64 {
               return Verdict::TooOld;
           }
           if self.seen & (1 << back) != 0 {
               return Verdict::Duplicate;
           }
           self.seen |= 1 << back;
           Verdict::Fresh
       }
   }
   ```

   `seen` is a 64-bit mask where bit *i* means "`highest − i` was seen". A new highest nonce shifts the mask left by the distance (clearing it entirely if the distance is 64 or more) and sets bit 0. An older nonce inside the window is `Duplicate` if its bit is set and `Fresh` otherwise; outside the window it is `TooOld`, because the window no longer remembers, and **refusing** is the safe choice. The test walks through `100, 100, 99, 99, 37, 36, 101, 100, 1000, 100, 1000` and checks `Fresh`, `Duplicate`, `Fresh`, `Duplicate`, `Fresh`, `TooOld`, `Fresh`, `Duplicate`, `Fresh`, `TooOld`, `Duplicate`.
3. A possible answer (✓ compiled and run):

   ```rust
   //! Exercise 3: derive an idempotency key from the *meaning* of a request,
   //! with unambiguous field framing.
   use sha2::{Digest, Sha256};

   /// Key for "pay `amount` to `recipient` for `order_id`".
   pub fn idempotency_key(order_id: &str, recipient: &[u8; 20], amount: u64) -> [u8; 32] {
       let mut h = Sha256::new();
       h.update(b"lab/idem/v1");
       h.update((order_id.len() as u64).to_be_bytes());
       h.update(order_id.as_bytes());
       h.update(recipient);
       h.update(amount.to_be_bytes());
       h.finalize().into()
   }

   /// The ambiguous version: plain concatenation.
   pub fn naive_key(order_id: &str, amount: &str) -> [u8; 32] {
       let mut h = Sha256::new();
       h.update(order_id.as_bytes());
       h.update(amount.as_bytes());
       h.finalize().into()
   }
   ```

   The key hashes a domain label, the **length** of the variable-size field, and fixed-width integers. The test shows that changing the amount, the order id or the recipient changes the key, and that `naive_key("ab", "c") == naive_key("a", "bc")`, so a key built by concatenation can merge two different requests, and the second would be dropped as a duplicate. (This is the length-prefix rule of Lesson 48 §4.)
4. A possible answer (✓ compiled and run):

   ```rust
   //! Exercise 4: a token bucket with injected time (rate limit for outbound calls).

   /// Refills `rate_per_sec` tokens per second up to `capacity`.
   #[derive(Debug)]
   pub struct TokenBucket {
       capacity: u64,
       rate_per_sec: u64,
       milli_tokens: u64, // tokens * 1000, to avoid floats
       last_ms: u64,
   }

   impl TokenBucket {
       /// A full bucket.
       pub fn new(capacity: u64, rate_per_sec: u64, now_ms: u64) -> Self {
           TokenBucket {
               capacity,
               rate_per_sec,
               milli_tokens: capacity * 1000,
               last_ms: now_ms,
           }
       }

       /// Takes one token if available.
       pub fn try_take(&mut self, now_ms: u64) -> bool {
           let dt = now_ms.saturating_sub(self.last_ms);
           self.last_ms = self.last_ms.max(now_ms);
           let add = dt.saturating_mul(self.rate_per_sec);
           self.milli_tokens = self
               .milli_tokens
               .saturating_add(add)
               .min(self.capacity * 1000);
           if self.milli_tokens >= 1000 {
               self.milli_tokens -= 1000;
               true
           } else {
               false
           }
       }
   }
   ```

   The bucket stores **milli-tokens** (tokens × 1000) so that a rate of *r* tokens per second adds exactly `dt_ms × r` milli-tokens, with no floats. `saturating_sub` makes a backwards clock add nothing, `max` keeps `last_ms` from moving back, and `min(capacity × 1000)` caps the bucket. The test: capacity 2, rate 1/s: two takes at t=0, a refusal at 0 and at 999, a success at 1,000, a refusal again, a long gap refills to the capacity and no further, and a call with an earlier time adds nothing.
5. A possible answer (✓ compiled and run; the table is the test's assertion):

   ```rust
   //! Exercise 5: enumerate every crash point of a "send once" protocol.
   //! Steps: (1) write intent, (2) call the chain, (3) write result.

   /// What is durable / on chain after a crash.
   #[derive(Debug, Clone, Copy, PartialEq, Eq)]
   pub struct World {
       /// Intent journalled.
       pub intent: bool,
       /// Chain applied the tx (count of effects).
       pub effects: u32,
       /// Result journalled.
       pub result: bool,
   }

   /// The state after executing the first `steps` steps (0..=3) then crashing.
   pub fn after(steps: u8) -> World {
       World {
           intent: steps >= 1,
           effects: u32::from(steps >= 2),
           result: steps >= 3,
       }
   }

   /// Recovery with a chain that deduplicates by id: always safe to re-run.
   pub fn recover_idempotent(w: World) -> World {
       let mut w = w;
       w.intent = true;
       w.effects = 1;
       w.result = true;
       w
   }

   /// Recovery with a chain that does NOT deduplicate: re-runs step 2 whenever
   /// the result is missing.
   pub fn recover_blind(w: World) -> World {
       let mut w = w;
       w.intent = true;
       if !w.result {
           w.effects += 1;
           w.result = true;
       }
       w
   }
   ```

   | crash after | intent journalled | effect on chain | result journalled |
   |---|---|---|---|
   | nothing | no | 0 | no |
   | step 1 | yes | 0 | no |
   | step 2 | yes | **1** | no |
   | step 3 | yes | 1 | yes |

   Recovery for a **deduplicating** chain is "re-run everything not marked done": the result is exactly one effect from every starting point (✓ `[1, 1, 1, 1]`). For a **blind** chain, recovery that re-runs step 2 whenever the result is missing gives effects `[1, 1, 2, 1]`: the crash between step 2 and step 3 doubles the effect. The fix is not in recovery but in the protocol: make step 2 idempotent (a transaction id the chain deduplicates), or ask the chain whether the transaction is known before re-submitting, or use a design where the chain itself rejects a repeated nonce. (If the crash is *before* step 1 completes, the caller never got an acknowledgement and will retry with the same idempotency key, §5.2.)
6. A possible answer (✓ compiled and run):

   ```rust
   //! Exercise 6: check that key validity windows cover a period without a gap
   //! and overlap long enough for in-flight messages.

   /// `[from, until)` window of one key.
   pub type Window = (u64, u64);

   /// Every instant in `[start, end)` is covered, and each hand-over overlaps by
   /// at least `min_overlap`.
   pub fn rotation_ok(mut ws: Vec<Window>, start: u64, end: u64, min_overlap: u64) -> bool {
       ws.sort();
       if ws.iter().any(|&(f, u)| f >= u) || ws.is_empty() {
           return false;
       }
       if ws[0].0 > start {
           return false;
       }
       let mut covered_to = ws[0].1;
       for &(f, u) in &ws[1..] {
           if f > covered_to {
               return false; // gap
           }
           if covered_to < end && covered_to - f < min_overlap {
               return false; // too little overlap before the old key ends
           }
           covered_to = covered_to.max(u);
       }
       covered_to >= end
   }
   ```

   The windows are sorted by start. The first must start at or before `start`. Walking forward with `covered_to` (the end of what is covered so far): the next window must begin at or before `covered_to` (no gap); and unless the old coverage already reaches `end`, the overlap `covered_to − from` must be at least `min_overlap`. The test checks a good plan, a plan with too little overlap (50 against 60 needed), a one-millisecond gap, a start that is too late, a plan that ends early, an empty plan, an empty window and an unsorted input.
7. A possible answer (✓ compiled and run; tested for every cut point from 0 to the length plus 2, including cuts beyond the end):

   ```rust
   //! Exercise 7: compact an event log into a snapshot plus a tail.
   use std::collections::BTreeMap;

   /// Events of a balance ledger.
   #[derive(Debug, Clone, PartialEq, Eq)]
   pub enum Ev {
       /// Credit.
       Add(u32, u64),
       /// Debit (ignored if it would underflow).
       Sub(u32, u64),
   }

   /// Applies one event.
   pub fn apply(state: &mut BTreeMap<u32, u64>, e: &Ev) {
       match e {
           Ev::Add(k, n) => *state.entry(*k).or_insert(0) += n,
           Ev::Sub(k, n) => {
               if let Some(v) = state.get_mut(k)
                   && *v >= *n
               {
                   *v -= n;
               }
           }
       }
   }

   /// Folds a log.
   pub fn replay(events: &[Ev]) -> BTreeMap<u32, u64> {
       let mut s = BTreeMap::new();
       events.iter().for_each(|e| apply(&mut s, e));
       s
   }

   /// Snapshot of the first `upto` events plus the remaining tail.
   pub fn compact(events: &[Ev], upto: usize) -> (BTreeMap<u32, u64>, Vec<Ev>) {
       let upto = upto.min(events.len());
       (replay(&events[..upto]), events[upto..].to_vec())
   }

   /// State from a snapshot and a tail (must equal the full replay).
   pub fn from_snapshot(snap: &BTreeMap<u32, u64>, tail: &[Ev]) -> BTreeMap<u32, u64> {
       let mut s = snap.clone();
       tail.iter().for_each(|e| apply(&mut s, e));
       s
   }
   ```

   Compaction is correct because replay is a **left fold**: `fold(events) = fold(tail) ∘ fold(prefix)`. The subtle case is the ignored debit: `Sub(2, 9)` against a balance of 5 does nothing, and `Sub(7, 1)` on an unknown account does nothing, both *in the snapshot and in the tail*. A real compaction must write the snapshot durably **before** deleting the old log, and record which log position the snapshot covers.
8. A possible answer (✓ compiled and run):

   ```rust
   //! Exercise 8: failure rate over the last N outcomes (a ring buffer).

   /// Remembers the last `n` outcomes.
   #[derive(Debug)]
   pub struct Health {
       ring: Vec<bool>,
       next: usize,
       filled: usize,
   }

   impl Health {
       /// Window of `n` outcomes.
       pub fn new(n: usize) -> Self {
           assert!(n > 0);
           Health {
               ring: vec![true; n],
               next: 0,
               filled: 0,
           }
       }

       /// Records an outcome.
       pub fn record(&mut self, ok: bool) {
           self.ring[self.next] = ok;
           self.next = (self.next + 1) % self.ring.len();
           self.filled = (self.filled + 1).min(self.ring.len());
       }

       /// Failures in the window.
       pub fn failures(&self) -> usize {
           self.ring
               .iter()
               .take(self.filled)
               .filter(|&&ok| !ok)
               .count()
       }

       /// Unhealthy once the window is full and at least `max_failures` failed.
       pub fn unhealthy(&self, max_failures: usize) -> bool {
           self.filled == self.ring.len() && self.failures() >= max_failures
       }
   }
   ```

   A ring buffer holds the last *n* outcomes; `unhealthy` requires a full window so a few early failures do not trip it. Compared with "N consecutive failures", a **failure-rate window** is a better signal when failures are **intermittent** (a provider that fails 60% of requests never produces five in a row but is clearly unhealthy) and a worse one when the dependency is fully down (consecutive counting reacts after N calls, a window after the window fills). Many systems use both. The test: a window of 4 with three failures is not yet unhealthy; after a fourth outcome (`true`) it has 3 failures and is unhealthy at threshold 3; one more success pushes the oldest failure out and it recovers.

</details>

---

## 12. Debugging challenge

A webhook receiver was written in a hurry. Four reports arrive: (1) a security reviewer takes a request body, attaches the timestamp `t=1` (decades ago) and a tag computed over the body alone, and it is accepted; (2) the integration test suite from the provider reports that *every genuine delivery fails* (but the receiver's own unit test, which uses the same helper to sign, passes); (3) a penetration tester shows that response times differ depending on how many leading characters of a forged tag are right; (4) a delivery signed with the string `test-secret`, which a developer hard-coded "temporarily", is accepted in production. Explain each cause and fix the code.

```rust
//! Debugging challenge: a webhook receiver with four defects.
use hmac::{Hmac, KeyInit, Mac};
use sha2::Sha256;

fn mac(key: &[u8], msg: &[u8]) -> Vec<u8> {
    let mut m = <Hmac<Sha256> as KeyInit>::new_from_slice(key).unwrap();
    m.update(msg);
    m.finalize().into_bytes().to_vec()
}

fn hex(b: &[u8]) -> String {
    b.iter().map(|x| format!("{x:02x}")).collect()
}

/// Sender side (correct): header `t=<ts>,v1=<hex>`.
pub fn sign(key: &[u8], ts: u64, body: &[u8]) -> String {
    let mut m = format!("{ts}.").into_bytes();
    m.extend_from_slice(body);
    format!("t={ts},v1={}", hex(&mac(key, &m)))
}

/// The faulty receiver.
pub mod buggy {
    use super::*;
    /// Returns true if the delivery is accepted.
    pub fn verify(key: &[u8], header: &str, body: &[u8], now: u64) -> bool {
        let _ = now; // (1) no freshness check
        let Some((t, v)) = header.split_once(",v1=") else {
            return false;
        };
        let _ts = t.trim_start_matches("t=");
        // (2) the timestamp is not part of the signed message
        let want = hex(&mac(key, body));
        // (3) variable-time comparison of the hex strings
        if want == v {
            return true;
        }
        // (4) accepts a *second* secret that was "temporarily" hard-coded
        hex(&mac(b"test-secret", body)) == v
    }
}
```

<details markdown="1">
<summary>Solution</summary>

1. **No freshness check, and the timestamp is not part of the MAC.** `now` is ignored (`let _ = now`), and the MAC covers only the body, so the `t=` value is decoration: an attacker can attach any timestamp (the oldest, the newest) to a body whose tag they have seen, and even if a freshness check existed it would check a number the attacker chose (✓ `buggy_accepts_an_ancient_timestamp_with_a_body_only_tag`). Fix: parse `t` as a number, refuse `now.abs_diff(ts) > tolerance`, and compute the MAC over `"<ts>." ‖ body` (✓ `fixed_rejects_replay_of_old_delivery`, `fixed_rejects_forged_timestamp`).
2. **The receiver signs a different message from the sender.** The sender (the provider; `sign` in the file) MACs `"<ts>." ‖ body`; the faulty receiver MACs the bare body, so it can never agree with a genuine delivery. The receiver's own unit test passed because it used the receiver's own (wrong) MAC to build the test input: **a test that shares the bug's code path proves nothing** ((✓ `buggy_has_all_the_flaws`: a genuine delivery from `sign` is refused by `buggy::verify`). Fix: one function for the signed message, used by both sides, and a test against a **known answer from an independent implementation** (as in §3.2).
3. **Variable-time comparison.** `want == v` compares two `String`s and stops at the first different byte. Fix: decode the tag to bytes (refusing malformed hex) and use `Mac::verify_slice` (✓ `fixed::verify`), which compares in constant time. Never compare secrets, MACs or tokens with `==`.
4. **A second, hard-coded secret.** The fallback `if hex(&mac(b"test-secret", body)) == v { return true }` is a backdoor. Fix: delete it; keep test secrets in test configuration, never in the code path (✓ `fixed_rejects_the_test_secret`). A grep for string literals near `mac(` in code review, and a CI check for known test secrets, would have caught it. If two secrets are needed for rotation (§3.4), they come from configuration and both are expected to be real.

Fixed version (✓ compiled; four tests for it and two that demonstrate faults of the buggy one):

```rust
/// The corrected receiver.
pub mod fixed {
    use super::*;
    /// Returns true if the delivery is accepted.
    pub fn verify(key: &[u8], header: &str, body: &[u8], now: u64, tolerance: u64) -> bool {
        let Some(rest) = header.strip_prefix("t=") else {
            return false;
        };
        let Some((t, v)) = rest.split_once(",v1=") else {
            return false;
        };
        let Ok(ts) = t.parse::<u64>() else {
            return false;
        };
        if now.abs_diff(ts) > tolerance {
            return false;
        }
        let mut m = format!("{ts}.").into_bytes();
        m.extend_from_slice(body);
        let Some(tag) = unhex(v) else { return false };
        let mut h = <Hmac<Sha256> as KeyInit>::new_from_slice(key).unwrap();
        h.update(&m);
        h.verify_slice(&tag).is_ok()
    }

    fn unhex(s: &str) -> Option<Vec<u8>> {
        if !s.len().is_multiple_of(2) || !s.is_ascii() {
            return None;
        }
        (0..s.len() / 2)
            .map(|i| u8::from_str_radix(&s[2 * i..2 * i + 2], 16).ok())
            .collect()
    }
}
```

Two notes: the fixed `unhex` rejects odd length and non-ASCII input before slicing, so it cannot panic on a multi-byte character in the middle of the header (slicing a `str` at a non-boundary panics); and the production version in `webhook.rs` is stricter still (§3.3: canonical timestamp, lowercase hex only, several keys, constant-time on all of them).

</details>

---

## 13. Interview questions

1. *What must be inside the bytes you sign, and why?*
2. *What is a replay attack, and how do nonces, expiry and chain ids each prevent a part of it?*
3. *Your signing service restarts. How do you make sure it never reuses a nonce?*
4. *Why check the signature before recording the nonce?*
5. *What is wrong with `if computed_mac == provided_mac`?*
6. *How do you verify a webhook? What does the timestamp do, and what does it not do?*
7. *How do you rotate a signing key or a webhook secret without downtime?*
8. *Why does a signing service need a policy, and what belongs in it?*
9. *What is an outbox? Why not just call the chain from the request handler?*
10. *Explain leases and fencing tokens in a job queue.*
11. *A request to submit a transaction times out. Walk through what you do.*
12. *"Exactly-once delivery" — is it possible?*
13. *What is a circuit breaker? Describe its states and a bug you could introduce in the half-open state.*
14. *How do you recover state after a crash, and how do you detect a torn write?*
15. *How would you test that a service survives a crash between two steps?*

<details markdown="1">
<summary>Answers</summary>

1. Everything that gives the message its meaning: a **domain/version label** (so a signature for one protocol is not valid in another), the **chain/network id**, the **key id** (rotation), a **nonce** (uniqueness), an **expiry**, and the **payload with its length** so layouts cannot be confused (§2.1, §2.2). Anything not signed can be changed by an attacker.
2. Replay is re-sending a valid message. A **nonce** (remembered by the verifier) stops reuse, an **expiry** bounds how long a captured message works and how long the verifier must remember, a **chain id** stops it being valid on another network or deployment (§2.5, §2.2).
3. Reserve a block of numbers, **store the reservation durably before using the block**, and start from the stored value after a restart; gaps are fine, reuse is not (§2.4). On chains where the nonce is a gapless account sequence, take the nonce from the chain and serialize sends.
4. Because recording first lets an attacker with a forged high nonce advance your counter and lock out the real sender; only authenticated messages may change state (✓ `bad_signature_does_not_advance_nonce`).
5. `==` returns at the first differing byte, so the time reveals how much of a guess is right; use a constant-time comparison (`Mac::verify_slice`, `subtle`) (§3.3).
6. Recompute HMAC over `timestamp "." body` with a shared secret, compare in constant time, parse the header strictly, and check the timestamp is within a tolerance **both ways**. The timestamp limits replay to a window; it does not prevent replay inside the window, so the handler must be idempotent by event id (§3.3, §5.2).
7. Support several keys with validity windows: add the new key, switch the signer, keep the old valid for at least the maximum lifetime of anything signed, then remove it; for webhooks, verify against both secrets during the overlap (§2.7, §3.4). Revocation handles the emergency case.
8. The code that builds requests is the code easiest to attack or break; the policy is a small, separate last line of defense: per-request and cumulative limits, allowed destinations/contracts/message types, rate limits, and logging of every decision (§2.3).
9. An outbox records the **intent** durably first, acknowledges the caller, and lets a separate loop do the work with retries; calling the chain inside the handler loses work on a crash, blocks on slow providers and gives the caller an ambiguous failure (§5.1).
10. A lease gives a job to a worker until a deadline; if the worker does not report, the job is given to another. A stale worker may still finish later, so each lease carries a **token** (the attempt number) that `complete`/`fail` must present, and an old token is refused (§5.3).
11. Do not re-sign with a new nonce or fee. Look the transaction up by its deterministic id; resend the **identical** signed bytes if it is unknown and the chain deduplicates; escalate (replace or alert) deliberately after a deadline. Model the outcome as `Uncertain` (Lesson 51 §5.3, §7.2).
12. Not as a property of the network: a message can be lost or duplicated. You build **at-least-once delivery + idempotent effects**, which behaves as exactly-once in effect (§7.3).
13. A breaker stops calling a failing dependency: Closed (count consecutive failures), Open (refuse for a pause), HalfOpen (allow one trial, close on success, reopen on failure). A classic bug: a half-open permit that is taken but never reported keeps the breaker shut for ever (§6.3); fix by only taking a permit when a call will be made and by expiring unreported permits.
14. Keep an append-only journal of events with a length and a checksum per record, replay through the same code as live operation, and keep the longest valid prefix, ignoring a torn tail (§4.2, §5.4). Take snapshots to bound replay time (exercise 7).
15. Simulate the crash at every step boundary (exercise 5's table), by dropping the in-memory state and recovering from the journal bytes, in a seeded loop over many schedules with faults and verify an invariant (every job's effect exactly once) (§7). For real services add process-kill tests.

</details>

---

## 14. Practical challenge: `relaylab`

Build a library crate that:

1. Provides a seeded generator and a **signed envelope** (domain, chain id, key id, nonce, expiry, payload) with a strict decoder.
2. Provides a **signing service** with a pluggable signer and policy, a nonce allocator that reserves blocks, a **verifier** with key windows, revocation, TTL limits, strict signature checking and replay protection that records a nonce only after full verification.
3. Provides **HMAC-signed webhooks** with a strict header, a freshness window in both directions, constant-time comparison and multi-key rotation.
4. Provides a **checksummed journal** whose recovery keeps the longest valid prefix.
5. Provides a **durable job queue** whose state is rebuilt by replaying journal events through one `apply`, with idempotency keys, leases with fencing tokens, backoff, a dead-letter state and a `reap` for exhausted leases.
6. Provides a **circuit breaker** with injected time and expiring half-open permits.
7. Provides a **relayer simulation** with a chain that can deduplicate or not, lost acknowledgements, refusals, crashes and recovery.
8. Has a benchmark example, passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

```toml
[package]
name = "relaylab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "Teaching code for Lesson 52: signing envelopes, signed webhooks, a write-ahead journal, a durable job queue, a circuit breaker and a relayer simulation. Not audited."

[dependencies]
sha2 = "0.11"
ed25519-dalek = "3"
hmac = "0.13"

[profile.dev.package."*"]
opt-level = 3
```

### `src/lib.rs`

```rust
//! Teaching code for Lesson 52: cryptographic integrations (signing envelopes,
//! signed webhooks), fault tolerance (write-ahead journal, durable job queue,
//! circuit breaker) and a relayer simulation that ties them together.
//! **Not audited; not for production.**
#![deny(missing_docs)]

pub mod breaker;
pub mod journal;
pub mod queue;
pub mod rng;
pub mod sign;
pub mod sim;
pub mod webhook;
```

### `src/rng.rs`

```rust
//! A small seeded generator (xorshift64*), **not** for secrets.

/// The generator.
#[derive(Debug, Clone)]
pub struct Rng(u64);

impl Rng {
    /// Seeds the generator; seed 0 is replaced by a fixed nonzero value.
    pub fn new(seed: u64) -> Rng {
        Rng(if seed == 0 {
            0x9E37_79B9_7F4A_7C15
        } else {
            seed
        })
    }

    /// The next 64 random bits.
    pub fn next_u64(&mut self) -> u64 {
        let mut x = self.0;
        x ^= x >> 12;
        x ^= x << 25;
        x ^= x >> 27;
        self.0 = x;
        x.wrapping_mul(0x2545_F491_4F6C_DD1D)
    }

    /// A value in `0..n` without modulo bias; `n` must be nonzero.
    pub fn below(&mut self, n: u64) -> u64 {
        assert!(n > 0, "below(0)");
        let limit = u64::MAX - u64::MAX % n;
        loop {
            let x = self.next_u64();
            if x < limit {
                return x % n;
            }
        }
    }

    /// True with probability `per_mille / 1000`.
    pub fn chance(&mut self, per_mille: u64) -> bool {
        self.below(1000) < per_mille
    }
}
```

### `src/sign.rs`

{% raw %}
```rust
//! Signing envelopes: what is signed, who may sign, how to verify, and how a
//! verifier refuses replays, expired messages, unknown and rotated keys.

use ed25519_dalek::{Signature, Signer as _, SigningKey, VerifyingKey};
use std::collections::BTreeMap;
use std::fmt;

/// Domain label prefixed (with its length) to every signed message.
pub const DOMAIN: &[u8] = b"lab/envelope/v1";
/// Largest payload an envelope may carry.
pub const MAX_PAYLOAD: usize = 64 * 1024;
/// Longest allowed lifetime of an envelope, in milliseconds.
pub const MAX_TTL_MS: u64 = 10 * 60 * 1000;

/// A signed message with the context that must not be replayable elsewhere.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Envelope {
    /// The network this message is for.
    pub chain_id: u32,
    /// Which key signed it.
    pub key_id: u32,
    /// Strictly increasing per key.
    pub nonce: u64,
    /// The message is invalid at or after this time (ms).
    pub expires_ms: u64,
    /// What is being authorized.
    pub payload: Vec<u8>,
    /// Ed25519 signature over [`Envelope::signing_bytes`].
    pub sig: [u8; 64],
}

/// Why bytes are not a valid envelope encoding.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum DecodeError {
    /// Fewer bytes than the format needs.
    Truncated,
    /// More bytes than the format uses.
    TrailingBytes,
    /// The payload length exceeds [`MAX_PAYLOAD`].
    PayloadTooLong(u64),
}

impl Envelope {
    /// The exact bytes that are signed: every field except the signature,
    /// fixed-width big-endian, with a domain label and a payload length.
    pub fn signing_bytes(&self) -> Vec<u8> {
        let mut v = Vec::with_capacity(64 + self.payload.len());
        v.extend_from_slice(&(DOMAIN.len() as u64).to_be_bytes());
        v.extend_from_slice(DOMAIN);
        v.extend_from_slice(&self.chain_id.to_be_bytes());
        v.extend_from_slice(&self.key_id.to_be_bytes());
        v.extend_from_slice(&self.nonce.to_be_bytes());
        v.extend_from_slice(&self.expires_ms.to_be_bytes());
        v.extend_from_slice(&(self.payload.len() as u64).to_be_bytes());
        v.extend_from_slice(&self.payload);
        v
    }

    /// Wire format: `chain_id ‖ key_id ‖ nonce ‖ expires ‖ len:u32 ‖ payload ‖ sig`.
    pub fn encode(&self) -> Vec<u8> {
        assert!(self.payload.len() <= MAX_PAYLOAD, "payload too long");
        let mut v = Vec::new();
        v.extend_from_slice(&self.chain_id.to_be_bytes());
        v.extend_from_slice(&self.key_id.to_be_bytes());
        v.extend_from_slice(&self.nonce.to_be_bytes());
        v.extend_from_slice(&self.expires_ms.to_be_bytes());
        v.extend_from_slice(&(self.payload.len() as u32).to_be_bytes());
        v.extend_from_slice(&self.payload);
        v.extend_from_slice(&self.sig);
        v
    }

    /// Strict decoder: accepts exactly what [`Envelope::encode`] produces.
    pub fn decode(b: &[u8]) -> Result<Envelope, DecodeError> {
        const HEAD: usize = 4 + 4 + 8 + 8 + 4;
        if b.len() < HEAD {
            return Err(DecodeError::Truncated);
        }
        let u32_at = |i: usize| u32::from_be_bytes(b[i..i + 4].try_into().expect("4 bytes"));
        let u64_at = |i: usize| u64::from_be_bytes(b[i..i + 8].try_into().expect("8 bytes"));
        let len = u64::from(u32_at(24));
        if len > MAX_PAYLOAD as u64 {
            return Err(DecodeError::PayloadTooLong(len));
        }
        let len = len as usize;
        let need = HEAD + len + 64;
        if b.len() < need {
            return Err(DecodeError::Truncated);
        }
        if b.len() > need {
            return Err(DecodeError::TrailingBytes);
        }
        Ok(Envelope {
            chain_id: u32_at(0),
            key_id: u32_at(4),
            nonce: u64_at(8),
            expires_ms: u64_at(16),
            payload: b[HEAD..HEAD + len].to_vec(),
            sig: b[HEAD + len..].try_into().expect("64 bytes"),
        })
    }
}

/// Anything that can sign: a local key, a remote service, a hardware module.
pub trait Signer {
    /// The id under which the matching public key is registered.
    fn key_id(&self) -> u32;
    /// Signs `msg`.
    fn sign(&self, msg: &[u8]) -> [u8; 64];
}

/// A key held in process memory.
pub struct LocalSigner {
    key: SigningKey,
    key_id: u32,
}

impl LocalSigner {
    /// Builds a signer from a 32-byte seed.
    pub fn from_seed(seed: [u8; 32], key_id: u32) -> LocalSigner {
        LocalSigner {
            key: SigningKey::from_bytes(&seed),
            key_id,
        }
    }

    /// The public key bytes.
    pub fn public(&self) -> [u8; 32] {
        self.key.verifying_key().to_bytes()
    }
}

impl fmt::Debug for LocalSigner {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // never print the key
        write!(
            f,
            "LocalSigner {{ key_id: {}, key: <redacted> }}",
            self.key_id
        )
    }
}

impl Signer for LocalSigner {
    fn key_id(&self) -> u32 {
        self.key_id
    }
    fn sign(&self, msg: &[u8]) -> [u8; 64] {
        self.key.sign(msg).to_bytes()
    }
}

/// A rule the signing service applies before it signs anything.
pub trait Policy {
    /// `Ok` if the payload may be signed.
    fn check(&self, payload: &[u8]) -> Result<(), String>;
}

/// Payloads start with an 8-byte big-endian amount, which must not exceed `max`.
pub struct MaxAmount(pub u64);

impl Policy for MaxAmount {
    fn check(&self, payload: &[u8]) -> Result<(), String> {
        let head: [u8; 8] = payload
            .get(..8)
            .and_then(|s| s.try_into().ok())
            .ok_or("payload shorter than an amount")?;
        let amount = u64::from_be_bytes(head);
        if amount > self.0 {
            return Err(format!("amount {amount} exceeds the limit {}", self.0));
        }
        Ok(())
    }
}

/// Why the service refused to sign.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum SignError {
    /// The payload is larger than [`MAX_PAYLOAD`].
    PayloadTooLong,
    /// The requested lifetime is zero or longer than [`MAX_TTL_MS`].
    BadTtl(u64),
    /// The policy refused.
    Policy(String),
}

/// Hands out increasing nonces without ever repeating one after a restart:
/// it *reserves* a block of numbers, the caller stores the reservation
/// durably **before** using any number from the block, and a restarted
/// allocator starts at the stored value. Gaps are harmless; reuse is not.
#[derive(Debug, Clone)]
pub struct NonceAllocator {
    next: u64,
    reserved: u64,
    block: u64,
    pending: Option<u64>,
}

impl NonceAllocator {
    /// `durable_high_water` is the last reservation that was stored (0 for a
    /// new key); `block` is how many numbers each reservation covers.
    pub fn new(durable_high_water: u64, block: u64) -> NonceAllocator {
        assert!(block > 0, "block must be positive");
        NonceAllocator {
            next: durable_high_water,
            reserved: durable_high_water,
            block,
            pending: None,
        }
    }

    /// The next nonce.
    pub fn allocate(&mut self) -> u64 {
        if self.next >= self.reserved {
            self.reserved = self
                .next
                .checked_add(self.block)
                .expect("nonce space exhausted");
            self.pending = Some(self.reserved);
        }
        let n = self.next;
        self.next += 1;
        n
    }

    /// A new reservation that must be stored durably before the nonces it
    /// covers are used.
    pub fn take_pending(&mut self) -> Option<u64> {
        self.pending.take()
    }
}

/// A signing service: applies a policy, assigns nonces, signs.
pub struct SigningService<S, P> {
    signer: S,
    policy: P,
    chain_id: u32,
    nonces: NonceAllocator,
}

impl<S: Signer, P: Policy> SigningService<S, P> {
    /// `durable_high_water` is the last nonce reservation that was stored;
    /// every nonce this key ever used is below it.
    pub fn new(signer: S, policy: P, chain_id: u32, durable_high_water: u64, block: u64) -> Self {
        SigningService {
            signer,
            policy,
            chain_id,
            nonces: NonceAllocator::new(durable_high_water, block),
        }
    }

    /// A nonce reservation to store durably before the envelope is used.
    pub fn take_reservation(&mut self) -> Option<u64> {
        self.nonces.take_pending()
    }

    /// Signs `payload`, valid for `ttl_ms` from `now_ms`.
    pub fn sign(
        &mut self,
        payload: Vec<u8>,
        now_ms: u64,
        ttl_ms: u64,
    ) -> Result<Envelope, SignError> {
        if payload.len() > MAX_PAYLOAD {
            return Err(SignError::PayloadTooLong);
        }
        if ttl_ms == 0 || ttl_ms > MAX_TTL_MS {
            return Err(SignError::BadTtl(ttl_ms));
        }
        self.policy.check(&payload).map_err(SignError::Policy)?;
        let mut env = Envelope {
            chain_id: self.chain_id,
            key_id: self.signer.key_id(),
            nonce: self.nonces.allocate(),
            expires_ms: now_ms.saturating_add(ttl_ms),
            payload,
            sig: [0; 64],
        };
        env.sig = self.signer.sign(&env.signing_bytes());
        Ok(env)
    }
}

/// One registered public key and when it may be used.
#[derive(Debug, Clone)]
pub struct KeyEntry {
    /// Public key bytes.
    pub public: [u8; 32],
    /// First instant (ms) at which signatures made with the key are accepted.
    pub valid_from_ms: u64,
    /// First instant at which they are no longer accepted.
    pub valid_until_ms: u64,
    /// Revoked keys are never accepted.
    pub revoked: bool,
}

/// Why verification failed. The order of the checks is part of the contract.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum VerifyError {
    /// The envelope is for another network.
    WrongChain,
    /// No such key id.
    UnknownKey(u32),
    /// The key was revoked.
    Revoked(u32),
    /// The key is not valid at this time.
    KeyNotValidNow(u32),
    /// `now >= expires`.
    Expired,
    /// The lifetime claimed is longer than [`MAX_TTL_MS`] from now.
    TtlTooLong,
    /// The key bytes are not a valid public key.
    BadKey,
    /// The signature does not verify.
    BadSignature,
    /// The nonce is not greater than the last accepted one.
    Replay {
        /// Last accepted nonce for this key.
        last: u64,
        /// Nonce received.
        got: u64,
    },
}

/// Verifies envelopes and remembers the highest nonce per key.
#[derive(Debug, Clone)]
pub struct Verifier {
    chain_id: u32,
    keys: BTreeMap<u32, KeyEntry>,
    last_nonce: BTreeMap<u32, u64>,
}

impl Verifier {
    /// A verifier for one network.
    pub fn new(chain_id: u32) -> Verifier {
        Verifier {
            chain_id,
            keys: BTreeMap::new(),
            last_nonce: BTreeMap::new(),
        }
    }

    /// Registers (or replaces) a key.
    pub fn add_key(&mut self, key_id: u32, entry: KeyEntry) {
        self.keys.insert(key_id, entry);
    }

    /// Revokes a key; returns false if it is unknown.
    pub fn revoke(&mut self, key_id: u32) -> bool {
        match self.keys.get_mut(&key_id) {
            Some(k) => {
                k.revoked = true;
                true
            }
            None => false,
        }
    }

    /// The highest nonce accepted for a key.
    pub fn last_nonce(&self, key_id: u32) -> Option<u64> {
        self.last_nonce.get(&key_id).copied()
    }

    /// Checks everything and, only if all checks pass, records the nonce.
    pub fn verify(&mut self, env: &Envelope, now_ms: u64) -> Result<(), VerifyError> {
        if env.chain_id != self.chain_id {
            return Err(VerifyError::WrongChain);
        }
        let key = self
            .keys
            .get(&env.key_id)
            .ok_or(VerifyError::UnknownKey(env.key_id))?;
        if key.revoked {
            return Err(VerifyError::Revoked(env.key_id));
        }
        if now_ms < key.valid_from_ms || now_ms >= key.valid_until_ms {
            return Err(VerifyError::KeyNotValidNow(env.key_id));
        }
        if now_ms >= env.expires_ms {
            return Err(VerifyError::Expired);
        }
        if env.expires_ms - now_ms > MAX_TTL_MS {
            return Err(VerifyError::TtlTooLong);
        }
        let vk = VerifyingKey::from_bytes(&key.public).map_err(|_| VerifyError::BadKey)?;
        vk.verify_strict(&env.signing_bytes(), &Signature::from_bytes(&env.sig))
            .map_err(|_| VerifyError::BadSignature)?;
        if let Some(&last) = self.last_nonce.get(&env.key_id) {
            if env.nonce <= last {
                return Err(VerifyError::Replay {
                    last,
                    got: env.nonce,
                });
            }
        }
        self.last_nonce.insert(env.key_id, env.nonce);
        Ok(())
    }
}
```
{% endraw %}

### `src/webhook.rs`

```rust
//! Signed webhooks: HMAC-SHA256 over `timestamp "." body`, a strict header
//! format, a freshness window, constant-time comparison and key rotation.

use hmac::{Hmac, KeyInit, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

/// HMAC-SHA256 of `msg` under `key`.
pub fn hmac_sha256(key: &[u8], msg: &[u8]) -> [u8; 32] {
    let mut m = <HmacSha256 as KeyInit>::new_from_slice(key).expect("any key length");
    m.update(msg);
    m.finalize().into_bytes().into()
}

fn hex(bytes: &[u8]) -> String {
    bytes.iter().map(|b| format!("{b:02x}")).collect()
}

fn unhex(s: &str) -> Option<[u8; 32]> {
    if s.len() != 64 {
        return None;
    }
    let mut out = [0u8; 32];
    for (i, o) in out.iter_mut().enumerate() {
        let pair = s.get(2 * i..2 * i + 2)?;
        if !pair.bytes().all(|c| matches!(c, b'0'..=b'9' | b'a'..=b'f')) {
            return None;
        }
        *o = u8::from_str_radix(pair, 16).ok()?;
    }
    Some(out)
}

fn signed_message(timestamp_ms: u64, body: &[u8]) -> Vec<u8> {
    let mut m = timestamp_ms.to_string().into_bytes();
    m.push(b'.');
    m.extend_from_slice(body);
    m
}

/// Builds the header value `t=<timestamp>,v1=<64 hex digits>`.
pub fn sign(key: &[u8], timestamp_ms: u64, body: &[u8]) -> String {
    let tag = hmac_sha256(key, &signed_message(timestamp_ms, body));
    format!("t={timestamp_ms},v1={}", hex(&tag))
}

/// Why a webhook was refused.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum WebhookError {
    /// The header is not exactly `t=<decimal>,v1=<64 lowercase hex>`.
    MalformedHeader,
    /// The timestamp differs from `now` by more than the tolerance.
    Stale,
    /// No key produced this tag.
    BadSignature,
}

/// Verifies a delivery against any of `keys` (so that a new key can be added
/// before the old one is removed).
pub fn verify(
    keys: &[&[u8]],
    header: &str,
    body: &[u8],
    now_ms: u64,
    tolerance_ms: u64,
) -> Result<(), WebhookError> {
    let rest = header
        .strip_prefix("t=")
        .ok_or(WebhookError::MalformedHeader)?;
    let (ts, rest) = rest
        .split_once(",v1=")
        .ok_or(WebhookError::MalformedHeader)?;
    if ts.is_empty()
        || (ts.len() > 1 && ts.starts_with('0'))
        || !ts.bytes().all(|c| c.is_ascii_digit())
    {
        return Err(WebhookError::MalformedHeader);
    }
    let timestamp: u64 = ts.parse().map_err(|_| WebhookError::MalformedHeader)?;
    let tag = unhex(rest).ok_or(WebhookError::MalformedHeader)?;
    if now_ms.abs_diff(timestamp) > tolerance_ms {
        return Err(WebhookError::Stale);
    }
    let msg = signed_message(timestamp, body);
    let mut ok = false;
    for key in keys {
        let mut m = <HmacSha256 as KeyInit>::new_from_slice(key).expect("any key length");
        m.update(&msg);
        // `verify_slice` compares in constant time
        ok |= m.verify_slice(&tag).is_ok();
    }
    if ok {
        Ok(())
    } else {
        Err(WebhookError::BadSignature)
    }
}
```

### `src/journal.rs`

```rust
//! An append-only journal with checksummed records. Recovery keeps the longest
//! valid prefix, so a torn final write is harmless.

use sha2::{Digest, Sha256};

/// Largest record payload.
pub const MAX_RECORD: usize = 1 << 20;

fn checksum(len_be: [u8; 4], payload: &[u8]) -> [u8; 4] {
    let mut h = Sha256::new();
    h.update(b"lab/journal/v1");
    h.update(len_be);
    h.update(payload);
    let d = h.finalize();
    [d[0], d[1], d[2], d[3]]
}

/// An in-memory journal (the bytes are what would be written to disk).
#[derive(Debug, Default, Clone)]
pub struct Journal {
    buf: Vec<u8>,
}

impl Journal {
    /// An empty journal.
    pub fn new() -> Journal {
        Journal::default()
    }

    /// Appends one record: `len:u32 ‖ payload ‖ checksum:4`.
    pub fn append(&mut self, payload: &[u8]) {
        assert!(payload.len() <= MAX_RECORD, "record too long");
        let len = (payload.len() as u32).to_be_bytes();
        self.buf.extend_from_slice(&len);
        self.buf.extend_from_slice(payload);
        self.buf.extend_from_slice(&checksum(len, payload));
    }

    /// The bytes written so far.
    pub fn bytes(&self) -> &[u8] {
        &self.buf
    }

    /// Rebuilds a journal from recovered bytes, discarding any invalid tail.
    pub fn from_bytes(bytes: &[u8]) -> (Journal, Vec<Vec<u8>>) {
        let (records, valid) = recover(bytes);
        (
            Journal {
                buf: bytes[..valid].to_vec(),
            },
            records,
        )
    }
}

/// Reads records until the first one that is truncated, too long or fails its
/// checksum. Returns the records and the number of valid bytes.
pub fn recover(bytes: &[u8]) -> (Vec<Vec<u8>>, usize) {
    let mut records = Vec::new();
    let mut pos = 0;
    while pos + 4 <= bytes.len() {
        let len_be: [u8; 4] = bytes[pos..pos + 4].try_into().expect("4 bytes");
        let len = u32::from_be_bytes(len_be) as usize;
        if len > MAX_RECORD {
            break;
        }
        let end = pos + 4 + len + 4;
        if end > bytes.len() {
            break;
        }
        let payload = &bytes[pos + 4..pos + 4 + len];
        if bytes[pos + 4 + len..end] != checksum(len_be, payload) {
            break;
        }
        records.push(payload.to_vec());
        pos = end;
    }
    (records, pos)
}
```

### `src/queue.rs`

```rust
//! A durable job queue (an outbox): every state change is an event appended to
//! a journal, and the in-memory state is rebuilt by replaying the events.
//! Delivery is at-least-once; deduplication is by an idempotency key.

use crate::journal::{Journal, MAX_RECORD};
use std::collections::BTreeMap;

/// Identifier of a job.
pub type JobId = u64;

/// Where a job is in its life.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Status {
    /// Waiting; may be leased once `ready_at` has passed.
    Pending {
        /// Earliest time (ms) the job may be leased.
        ready_at: u64,
    },
    /// Leased to a worker until `until`; after that it may be leased again.
    Leased {
        /// Lease expiry (ms).
        until: u64,
    },
    /// Finished successfully.
    Done,
    /// Gave up after too many attempts (the dead-letter state).
    Dead,
}

#[derive(Debug, Clone)]
struct Job {
    key: u64,
    payload: Vec<u8>,
    attempts: u32,
    status: Status,
}

/// A state change; the only thing ever written to the journal.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Event {
    /// A job was added.
    Enqueued {
        /// Its id.
        id: JobId,
        /// Idempotency key.
        key: u64,
        /// Earliest start time.
        ready_at: u64,
        /// The work description.
        payload: Vec<u8>,
    },
    /// A worker took the job until `until`.
    Leased {
        /// Job.
        id: JobId,
        /// Lease expiry.
        until: u64,
    },
    /// The worker finished it.
    Completed {
        /// Job.
        id: JobId,
    },
    /// The attempt failed; retry at `retry_at`.
    Failed {
        /// Job.
        id: JobId,
        /// Earliest next attempt.
        retry_at: u64,
    },
    /// The job is given up.
    Died {
        /// Job.
        id: JobId,
    },
}

impl Event {
    /// Binary encoding (tag byte, then fixed-width big-endian fields).
    pub fn encode(&self) -> Vec<u8> {
        let mut v = Vec::new();
        match self {
            Event::Enqueued {
                id,
                key,
                ready_at,
                payload,
            } => {
                v.push(1);
                v.extend_from_slice(&id.to_be_bytes());
                v.extend_from_slice(&key.to_be_bytes());
                v.extend_from_slice(&ready_at.to_be_bytes());
                v.extend_from_slice(&(payload.len() as u32).to_be_bytes());
                v.extend_from_slice(payload);
            }
            Event::Leased { id, until } => {
                v.push(2);
                v.extend_from_slice(&id.to_be_bytes());
                v.extend_from_slice(&until.to_be_bytes());
            }
            Event::Completed { id } => {
                v.push(3);
                v.extend_from_slice(&id.to_be_bytes());
            }
            Event::Failed { id, retry_at } => {
                v.push(4);
                v.extend_from_slice(&id.to_be_bytes());
                v.extend_from_slice(&retry_at.to_be_bytes());
            }
            Event::Died { id } => {
                v.push(5);
                v.extend_from_slice(&id.to_be_bytes());
            }
        }
        v
    }

    /// Strict decoder: unknown tags, short input and trailing bytes are errors.
    pub fn decode(b: &[u8]) -> Option<Event> {
        let (&tag, rest) = b.split_first()?;
        let u64_at = |i: usize| -> Option<u64> {
            Some(u64::from_be_bytes(rest.get(i..i + 8)?.try_into().ok()?))
        };
        let (ev, used) = match tag {
            1 => {
                let len = u32::from_be_bytes(rest.get(24..28)?.try_into().ok()?) as usize;
                if len > MAX_RECORD {
                    return None;
                }
                let payload = rest.get(28..28 + len)?.to_vec();
                (
                    Event::Enqueued {
                        id: u64_at(0)?,
                        key: u64_at(8)?,
                        ready_at: u64_at(16)?,
                        payload,
                    },
                    28 + len,
                )
            }
            2 => (
                Event::Leased {
                    id: u64_at(0)?,
                    until: u64_at(8)?,
                },
                16,
            ),
            3 => (Event::Completed { id: u64_at(0)? }, 8),
            4 => (
                Event::Failed {
                    id: u64_at(0)?,
                    retry_at: u64_at(8)?,
                },
                16,
            ),
            5 => (Event::Died { id: u64_at(0)? }, 8),
            _ => return None,
        };
        (rest.len() == used).then_some(ev)
    }
}

/// What `enqueue` did.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Enqueued {
    /// A new job was created.
    New(JobId),
    /// A job with this key already exists.
    Duplicate(JobId),
}

/// A job handed to a worker.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Lease {
    /// The job.
    pub id: JobId,
    /// Its work description.
    pub payload: Vec<u8>,
    /// Attempt number, starting at 1; also the token for `complete`/`fail`.
    pub attempt: u32,
}

/// Why `complete` or `fail` was refused.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum QueueError {
    /// No such job.
    Unknown,
    /// The job is not leased.
    NotLeased,
    /// Another attempt has started since; this holder's lease is stale.
    StaleLease,
}

/// What `fail` did.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum FailOutcome {
    /// Will be retried at this time.
    Retry(u64),
    /// Out of attempts: dead.
    Dead,
}

/// Counts per status.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub struct Stats {
    /// Waiting jobs.
    pub pending: usize,
    /// Leased jobs.
    pub leased: usize,
    /// Finished jobs.
    pub done: usize,
    /// Dead jobs.
    pub dead: usize,
}

/// The queue.
#[derive(Debug, Clone)]
pub struct Queue {
    jobs: BTreeMap<JobId, Job>,
    by_key: BTreeMap<u64, JobId>,
    journal: Journal,
    next_id: JobId,
    max_attempts: u32,
}

impl Queue {
    /// An empty queue; a job that fails `max_attempts` times is dead.
    pub fn new(max_attempts: u32) -> Queue {
        Queue {
            jobs: BTreeMap::new(),
            by_key: BTreeMap::new(),
            journal: Journal::new(),
            next_id: 1,
            max_attempts,
        }
    }

    /// Rebuilds a queue from journal bytes (an invalid tail is ignored).
    /// Every event goes through the same `apply` as in normal operation.
    pub fn recover(bytes: &[u8], max_attempts: u32) -> Queue {
        let (journal, records) = Journal::from_bytes(bytes);
        let mut q = Queue {
            jobs: BTreeMap::new(),
            by_key: BTreeMap::new(),
            journal,
            next_id: 1,
            max_attempts,
        };
        for r in records {
            match Event::decode(&r) {
                Some(e) => q.apply(&e),
                None => break,
            }
        }
        q
    }

    /// The journal bytes (what would be on disk).
    pub fn journal_bytes(&self) -> &[u8] {
        self.journal.bytes()
    }

    fn record(&mut self, e: Event) {
        self.journal.append(&e.encode());
        self.apply(&e);
    }

    fn apply(&mut self, e: &Event) {
        match e {
            Event::Enqueued {
                id,
                key,
                ready_at,
                payload,
            } => {
                self.jobs.insert(
                    *id,
                    Job {
                        key: *key,
                        payload: payload.clone(),
                        attempts: 0,
                        status: Status::Pending {
                            ready_at: *ready_at,
                        },
                    },
                );
                self.by_key.insert(*key, *id);
                self.next_id = self.next_id.max(id + 1);
            }
            Event::Leased { id, until } => {
                if let Some(j) = self.jobs.get_mut(id) {
                    j.attempts += 1;
                    j.status = Status::Leased { until: *until };
                }
            }
            Event::Completed { id } => {
                if let Some(j) = self.jobs.get_mut(id) {
                    j.status = Status::Done;
                }
            }
            Event::Failed { id, retry_at } => {
                if let Some(j) = self.jobs.get_mut(id) {
                    j.status = Status::Pending {
                        ready_at: *retry_at,
                    };
                }
            }
            Event::Died { id } => {
                if let Some(j) = self.jobs.get_mut(id) {
                    j.status = Status::Dead;
                }
            }
        }
    }

    /// Adds a job unless one with the same key exists.
    pub fn enqueue(&mut self, key: u64, payload: Vec<u8>, now_ms: u64) -> Enqueued {
        if let Some(&id) = self.by_key.get(&key) {
            return Enqueued::Duplicate(id);
        }
        let id = self.next_id;
        self.record(Event::Enqueued {
            id,
            key,
            ready_at: now_ms,
            payload,
        });
        Enqueued::New(id)
    }

    /// Declares dead every job whose lease expired after its last allowed
    /// attempt (no worker is left to report it).
    pub fn reap(&mut self, now_ms: u64) {
        let expired_out: Vec<JobId> = self
            .jobs
            .iter()
            .filter(|(_, j)| {
                matches!(j.status, Status::Leased { until } if until <= now_ms)
                    && j.attempts >= self.max_attempts
            })
            .map(|(id, _)| *id)
            .collect();
        for id in expired_out {
            self.record(Event::Died { id });
        }
    }

    /// Takes the oldest job that is ready (pending and due, or leased with an
    /// expired lease). Exhausted jobs are reaped first.
    pub fn lease(&mut self, now_ms: u64, lease_ms: u64) -> Option<Lease> {
        self.reap(now_ms);
        let id = self.jobs.iter().find_map(|(id, j)| {
            let ready = match j.status {
                Status::Pending { ready_at } => ready_at <= now_ms,
                Status::Leased { until } => until <= now_ms,
                Status::Done | Status::Dead => false,
            };
            ready.then_some(*id)
        })?;
        self.record(Event::Leased {
            id,
            until: now_ms.saturating_add(lease_ms),
        });
        let j = &self.jobs[&id];
        Some(Lease {
            id,
            payload: j.payload.clone(),
            attempt: j.attempts,
        })
    }

    /// Is there a job that `lease` would hand out at `now_ms`?
    pub fn has_ready(&self, now_ms: u64) -> bool {
        self.jobs.values().any(|j| match j.status {
            Status::Pending { ready_at } => ready_at <= now_ms,
            Status::Leased { until } => until <= now_ms && j.attempts < self.max_attempts,
            Status::Done | Status::Dead => false,
        })
    }

    fn holder(&self, id: JobId, attempt: u32) -> Result<(), QueueError> {
        let j = self.jobs.get(&id).ok_or(QueueError::Unknown)?;
        if !matches!(j.status, Status::Leased { .. }) {
            return Err(QueueError::NotLeased);
        }
        if j.attempts != attempt {
            return Err(QueueError::StaleLease);
        }
        Ok(())
    }

    /// Marks the leased job finished. Completing a job that is already done
    /// by the same attempt is accepted again (idempotent).
    pub fn complete(&mut self, id: JobId, attempt: u32) -> Result<(), QueueError> {
        if let Some(j) = self.jobs.get(&id) {
            if j.status == Status::Done && j.attempts == attempt {
                return Ok(());
            }
        }
        self.holder(id, attempt)?;
        self.record(Event::Completed { id });
        Ok(())
    }

    /// Records a failed attempt: retry after `backoff_ms`, or dead if no
    /// attempts remain.
    pub fn fail(
        &mut self,
        id: JobId,
        attempt: u32,
        now_ms: u64,
        backoff_ms: u64,
    ) -> Result<FailOutcome, QueueError> {
        self.holder(id, attempt)?;
        if attempt >= self.max_attempts {
            self.record(Event::Died { id });
            return Ok(FailOutcome::Dead);
        }
        let retry_at = now_ms.saturating_add(backoff_ms);
        self.record(Event::Failed { id, retry_at });
        Ok(FailOutcome::Retry(retry_at))
    }

    /// The status of a job.
    pub fn status(&self, id: JobId) -> Option<Status> {
        self.jobs.get(&id).map(|j| j.status)
    }

    /// Attempts started so far for a job.
    pub fn attempts(&self, id: JobId) -> Option<u32> {
        self.jobs.get(&id).map(|j| j.attempts)
    }

    /// The idempotency key of a job.
    pub fn key_of(&self, id: JobId) -> Option<u64> {
        self.jobs.get(&id).map(|j| j.key)
    }

    /// Number of jobs per status.
    pub fn stats(&self) -> Stats {
        let mut s = Stats::default();
        for j in self.jobs.values() {
            match j.status {
                Status::Pending { .. } => s.pending += 1,
                Status::Leased { .. } => s.leased += 1,
                Status::Done => s.done += 1,
                Status::Dead => s.dead += 1,
            }
        }
        s
    }
}
```

### `src/breaker.rs`

```rust
//! A circuit breaker with injected time.

/// The breaker's state at some instant.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum State {
    /// Calls flow; consecutive failures are counted.
    Closed,
    /// Calls are refused until the cool-down ends.
    Open,
    /// One trial call at a time decides whether to close or reopen.
    HalfOpen,
}

/// Counts consecutive failures; opens at a threshold; probes after a pause.
#[derive(Debug, Clone)]
pub struct Breaker {
    threshold: u32,
    open_ms: u64,
    failures: u32,
    opened_at: Option<u64>,
    probing_since: Option<u64>,
}

impl Breaker {
    /// Opens after `threshold` consecutive failures and stays open `open_ms`.
    pub fn new(threshold: u32, open_ms: u64) -> Breaker {
        assert!(threshold > 0, "threshold must be positive");
        Breaker {
            threshold,
            open_ms,
            failures: 0,
            opened_at: None,
            probing_since: None,
        }
    }

    /// The state at `now_ms`.
    pub fn state(&self, now_ms: u64) -> State {
        match self.opened_at {
            None => State::Closed,
            Some(t) if now_ms.saturating_sub(t) < self.open_ms => State::Open,
            Some(_) => State::HalfOpen,
        }
    }

    /// May a call be made now? In the half-open state only one trial is
    /// allowed until its result is recorded; a trial whose result never
    /// arrives is forgotten after another `open_ms`, so a lost probe cannot
    /// keep the breaker shut forever.
    pub fn allow(&mut self, now_ms: u64) -> bool {
        match self.state(now_ms) {
            State::Closed => true,
            State::Open => false,
            State::HalfOpen => match self.probing_since {
                Some(t) if now_ms.saturating_sub(t) < self.open_ms => false,
                _ => {
                    self.probing_since = Some(now_ms);
                    true
                }
            },
        }
    }

    /// Records a successful call.
    pub fn record_success(&mut self) {
        self.failures = 0;
        self.opened_at = None;
        self.probing_since = None;
    }

    /// Records a failed call.
    pub fn record_failure(&mut self, now_ms: u64) {
        if self.opened_at.is_some() {
            // a failed trial reopens for a full cool-down
            self.opened_at = Some(now_ms);
            self.probing_since = None;
            return;
        }
        self.failures += 1;
        if self.failures >= self.threshold {
            self.opened_at = Some(now_ms);
        }
    }
}
```

### `src/sim.rs`

```rust
//! A relayer simulation: jobs flow through the durable queue, a signing
//! service, a circuit breaker and a flaky "chain" that may time out *after*
//! processing a submission. Time, faults and crashes are seeded.

use crate::breaker::Breaker;
use crate::queue::{Enqueued, FailOutcome, Queue};
use crate::rng::Rng;
use crate::sign::{KeyEntry, LocalSigner, MaxAmount, Signer, SigningService, Verifier};
use sha2::{Digest, Sha256};
use std::collections::BTreeMap;

/// Network id used by the simulation.
pub const CHAIN_ID: u32 = 7;

/// Why a submission failed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SubmitError {
    /// Refused before processing: nothing happened.
    Unavailable,
    /// No answer — **but the submission may have been processed**.
    Timeout,
}

/// A successful submission.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Submitted {
    /// The chain accepted the transaction.
    Accepted,
    /// The chain had already seen this transaction id and ignored it.
    AlreadyKnown,
}

/// A chain that may or may not deduplicate submissions by transaction id.
#[derive(Debug, Clone)]
pub struct SimChain {
    /// How many times each transaction id took effect.
    pub effects: BTreeMap<[u8; 32], u32>,
    dedupe: bool,
    unavailable_pm: u64,
    timeout_after_pm: u64,
    verifier: Verifier,
    rng: Rng,
    /// Envelopes the chain rejected (should stay zero).
    pub rejected: u64,
}

impl SimChain {
    fn submit(
        &mut self,
        env_bytes: &[u8],
        tx_id: [u8; 32],
        now_ms: u64,
    ) -> Result<Submitted, SubmitError> {
        if self.rng.chance(self.unavailable_pm) {
            return Err(SubmitError::Unavailable);
        }
        let outcome = match crate::sign::Envelope::decode(env_bytes) {
            Ok(env) if self.verifier.verify(&env, now_ms).is_ok() => {
                if self.dedupe && self.effects.contains_key(&tx_id) {
                    Submitted::AlreadyKnown
                } else {
                    *self.effects.entry(tx_id).or_insert(0) += 1;
                    Submitted::Accepted
                }
            }
            _ => {
                self.rejected += 1;
                Submitted::AlreadyKnown
            }
        };
        if self.rng.chance(self.timeout_after_pm) {
            return Err(SubmitError::Timeout);
        }
        Ok(outcome)
    }
}

/// Parameters of a run.
#[derive(Debug, Clone)]
pub struct Params {
    /// Number of jobs.
    pub jobs: u64,
    /// Does the chain ignore a transaction id it already applied?
    pub dedupe: bool,
    /// Per-mille chance a submission is refused before processing.
    pub unavailable_pm: u64,
    /// Per-mille chance a processed submission times out.
    pub timeout_after_pm: u64,
    /// Per-mille chance the relayer crashes after each tick.
    pub crash_pm: u64,
    /// Attempts before a job is dead.
    pub max_attempts: u32,
    /// Limit of simulated time in milliseconds.
    pub time_limit_ms: u64,
}

impl Default for Params {
    fn default() -> Self {
        Params {
            jobs: 50,
            dedupe: true,
            unavailable_pm: 300,
            timeout_after_pm: 200,
            crash_pm: 30,
            max_attempts: 30,
            time_limit_ms: 600_000,
        }
    }
}

/// What a run produced.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Report {
    /// Jobs finished successfully.
    pub done: u64,
    /// Jobs given up.
    pub dead: u64,
    /// Transaction ids that took effect at least once.
    pub distinct_effects: u64,
    /// Transaction ids that took effect more than once.
    pub double_applied: u64,
    /// Submissions sent (including repeats).
    pub submissions: u64,
    /// Simulated crashes.
    pub crashes: u64,
    /// Times the breaker opened a call to the chain off.
    pub skipped_by_breaker: u64,
    /// Envelopes the chain rejected.
    pub rejected: u64,
    /// Sum of all backoff waits scheduled for failed attempts (ms).
    pub total_wait_ms: u64,
    /// Highest nonce reservation stored.
    pub nonce_high_water: u64,
    /// Simulated time when the run ended.
    pub finished_at: u64,
}

fn payload_for(job: u64) -> Vec<u8> {
    let mut p = (1000 + job).to_be_bytes().to_vec(); // amount
    p.extend_from_slice(&job.to_be_bytes());
    p
}

/// The transaction id of a payload (what the chain deduplicates on).
pub fn tx_id(payload: &[u8]) -> [u8; 32] {
    let mut h = Sha256::new();
    h.update(b"lab/relay-tx/");
    h.update(payload);
    h.finalize().into()
}

/// Runs the simulation.
pub fn run(seed: u64, p: &Params) -> Report {
    let mut rng = Rng::new(seed);
    let signer = LocalSigner::from_seed([5; 32], 1);
    let mut verifier = Verifier::new(CHAIN_ID);
    verifier.add_key(
        1,
        KeyEntry {
            public: signer.public(),
            valid_from_ms: 0,
            valid_until_ms: u64::MAX,
            revoked: false,
        },
    );
    let mut chain = SimChain {
        effects: BTreeMap::new(),
        dedupe: p.dedupe,
        unavailable_pm: p.unavailable_pm,
        timeout_after_pm: p.timeout_after_pm,
        verifier,
        rng: Rng::new(seed ^ 0xA5A5_A5A5),
        rejected: 0,
    };
    let mut queue = Queue::new(p.max_attempts);
    for job in 0..p.jobs {
        queue.enqueue(job, payload_for(job), 0);
        // a producer that retried its request: the duplicate is ignored
        assert!(matches!(
            queue.enqueue(job, payload_for(job), 0),
            Enqueued::Duplicate(_)
        ));
    }
    let mut nonce_log: Vec<u64> = Vec::new();
    let new_service = |log: &[u64]| {
        SigningService::new(
            LocalSigner::from_seed([5; 32], 1),
            MaxAmount(1_000_000),
            CHAIN_ID,
            log.last().copied().unwrap_or(0),
            16,
        )
    };
    let mut service = new_service(&nonce_log);
    let mut breaker = Breaker::new(5, 500);
    let (mut submissions, mut crashes, mut skipped, mut total_wait) = (0u64, 0u64, 0u64, 0u64);
    let mut now = 0u64;
    loop {
        let st = queue.stats();
        if st.pending == 0 && st.leased == 0 {
            break;
        }
        if now > p.time_limit_ms {
            break;
        }
        queue.reap(now);
        if !queue.has_ready(now) {
            // nothing to send: do not ask the breaker for a permit we will not use
        } else if !breaker.allow(now) {
            skipped += 1;
        } else if let Some(lease) = queue.lease(now, 1_000) {
            let env = service
                .sign(lease.payload.clone(), now, 30_000)
                .expect("policy allows");
            if let Some(res) = service.take_reservation() {
                nonce_log.push(res); // stored durably before the envelope leaves
            }
            submissions += 1;
            let result = chain.submit(&env.encode(), tx_id(&lease.payload), now);
            // a crash between the submission and recording its result
            let crash_here = rng.chance(p.crash_pm);
            if !crash_here {
                match result {
                    Ok(_) => {
                        breaker.record_success();
                        queue.complete(lease.id, lease.attempt).expect("holder");
                    }
                    Err(_) => {
                        breaker.record_failure(now);
                        let ceiling = (50u64 << lease.attempt.saturating_sub(1).min(16)).min(2_000);
                        let wait = rng.below(ceiling + 1);
                        total_wait += wait;
                        let _ = queue
                            .fail(lease.id, lease.attempt, now, wait)
                            .expect("holder");
                    }
                }
            } else {
                crashes += 1;
                queue = Queue::recover(queue.journal_bytes(), p.max_attempts);
                service = new_service(&nonce_log);
                breaker = Breaker::new(5, 500);
            }
        }
        now += 10;
    }
    let st = queue.stats();
    Report {
        done: st.done as u64,
        dead: st.dead as u64,
        distinct_effects: chain.effects.len() as u64,
        double_applied: chain.effects.values().filter(|&&c| c > 1).count() as u64,
        submissions,
        crashes,
        skipped_by_breaker: skipped,
        rejected: chain.rejected,
        total_wait_ms: total_wait,
        nonce_high_water: nonce_log.last().copied().unwrap_or(0),
        finished_at: now,
    }
}

#[allow(dead_code)]
fn _assert_signer_is_object_safe(_: &dyn Signer) {}

#[allow(dead_code)]
fn _unused(_: FailOutcome) {}
```

### `tests/sign_tests.rs`

```rust
use relaylab::rng::Rng;
use relaylab::sign::*;

fn pair() -> (SigningService<LocalSigner, MaxAmount>, Verifier) {
    let signer = LocalSigner::from_seed([9; 32], 1);
    let mut v = Verifier::new(7);
    v.add_key(
        1,
        KeyEntry {
            public: signer.public(),
            valid_from_ms: 0,
            valid_until_ms: u64::MAX,
            revoked: false,
        },
    );
    (SigningService::new(signer, MaxAmount(100), 7, 0, 4), v)
}
fn payload(amount: u64) -> Vec<u8> {
    let mut p = amount.to_be_bytes().to_vec();
    p.extend_from_slice(b"rest");
    p
}

#[test]
fn rng_known_vectors() {
    let mut r = Rng::new(1);
    let got: Vec<u64> = (0..4).map(|_| r.next_u64()).collect();
    assert_eq!(
        got,
        [
            5180492295206395165,
            12380297144915551517,
            13389498078930870103,
            5599127315341312413
        ]
    );
    assert_eq!(Rng::new(0).next_u64(), 973819730272012410);
}

#[test]
fn rng_below_and_chance() {
    let mut r = Rng::new(3);
    for _ in 0..1000 {
        assert!(r.below(7) < 7);
    }
    assert!(!r.chance(0));
    assert!(r.chance(1000));
    let hits = (0..10_000).filter(|_| r.chance(250)).count();
    assert!((2200..2800).contains(&hits), "{hits}");
}

#[test]
#[should_panic]
fn rng_below_zero_panics() {
    Rng::new(1).below(0);
}

#[test]
fn signing_bytes_layout() {
    let e = Envelope {
        chain_id: 1,
        key_id: 2,
        nonce: 3,
        expires_ms: 4,
        payload: vec![9, 9],
        sig: [0; 64],
    };
    let mut want = 15u64.to_be_bytes().to_vec();
    want.extend_from_slice(b"lab/envelope/v1");
    want.extend_from_slice(&1u32.to_be_bytes());
    want.extend_from_slice(&2u32.to_be_bytes());
    want.extend_from_slice(&3u64.to_be_bytes());
    want.extend_from_slice(&4u64.to_be_bytes());
    want.extend_from_slice(&2u64.to_be_bytes());
    want.extend_from_slice(&[9, 9]);
    assert_eq!(e.signing_bytes(), want);
}

#[test]
fn encode_decode_round_trip_and_layout() {
    let (mut s, _) = pair();
    let e = s.sign(payload(5), 1000, 5000).unwrap();
    let b = e.encode();
    assert_eq!(b.len(), 28 + e.payload.len() + 64);
    assert_eq!(&b[0..4], &7u32.to_be_bytes());
    assert_eq!(&b[4..8], &1u32.to_be_bytes());
    assert_eq!(&b[8..16], &0u64.to_be_bytes());
    assert_eq!(&b[16..24], &6000u64.to_be_bytes());
    assert_eq!(&b[24..28], &(e.payload.len() as u32).to_be_bytes());
    assert_eq!(Envelope::decode(&b).unwrap(), e);
}

#[test]
fn strict_decode() {
    let (mut s, _) = pair();
    let b = s.sign(payload(5), 0, 10).unwrap().encode();
    for n in 0..b.len() {
        assert_eq!(
            Envelope::decode(&b[..n]),
            Err(DecodeError::Truncated),
            "{n}"
        );
    }
    let mut t = b.clone();
    t.push(0);
    assert_eq!(Envelope::decode(&t), Err(DecodeError::TrailingBytes));
    let mut big = b.clone();
    big[24..28].copy_from_slice(&u32::MAX.to_be_bytes());
    assert_eq!(
        Envelope::decode(&big),
        Err(DecodeError::PayloadTooLong(u32::MAX as u64))
    );
    let mut edge = b.clone();
    edge[24..28].copy_from_slice(&((MAX_PAYLOAD + 1) as u32).to_be_bytes());
    assert_eq!(
        Envelope::decode(&edge),
        Err(DecodeError::PayloadTooLong(MAX_PAYLOAD as u64 + 1))
    );
}

#[test]
fn accepts_good_envelope() {
    let (mut s, mut v) = pair();
    let e = s.sign(payload(5), 1000, 5000).unwrap();
    assert_eq!(v.verify(&e, 1000), Ok(()));
    assert_eq!(v.last_nonce(1), Some(0));
}

#[test]
fn replay_rejected_and_order_enforced() {
    let (mut s, mut v) = pair();
    let a = s.sign(payload(1), 0, 1000).unwrap();
    let b = s.sign(payload(2), 0, 1000).unwrap();
    assert_eq!(v.verify(&b, 0), Ok(()));
    assert_eq!(
        v.verify(&b, 0),
        Err(VerifyError::Replay { last: 1, got: 1 })
    );
    assert_eq!(
        v.verify(&a, 0),
        Err(VerifyError::Replay { last: 1, got: 0 })
    );
}

#[test]
fn bad_signature_does_not_advance_nonce() {
    let (mut s, mut v) = pair();
    let mut e = s.sign(payload(1), 0, 1000).unwrap();
    let good = e.clone();
    e.payload[0] ^= 1;
    assert_eq!(v.verify(&e, 0), Err(VerifyError::BadSignature));
    assert_eq!(v.last_nonce(1), None);
    assert_eq!(v.verify(&good, 0), Ok(()));
}

#[test]
fn every_field_is_covered_by_the_signature() {
    let (mut s, mut v) = pair();
    let e = s.sign(payload(1), 100, 1000).unwrap();
    let mut a = e.clone();
    a.nonce += 1;
    assert_eq!(v.verify(&a, 100), Err(VerifyError::BadSignature));
    let mut b = e.clone();
    b.expires_ms += 1;
    assert_eq!(v.verify(&b, 100), Err(VerifyError::BadSignature));
    let mut c = e.clone();
    c.sig[0] ^= 1;
    assert_eq!(v.verify(&c, 100), Err(VerifyError::BadSignature));
    assert_eq!(v.verify(&e, 100), Ok(()));
}

#[test]
fn wrong_chain_and_unknown_key() {
    let (mut s, mut v) = pair();
    let mut e = s.sign(payload(1), 0, 1000).unwrap();
    e.chain_id = 8;
    assert_eq!(v.verify(&e, 0), Err(VerifyError::WrongChain));
    let mut e = s.sign(payload(1), 0, 1000).unwrap();
    e.key_id = 2;
    assert_eq!(v.verify(&e, 0), Err(VerifyError::UnknownKey(2)));
}

#[test]
fn domain_separation_between_chains() {
    let signer = LocalSigner::from_seed([9; 32], 1);
    let mut s7 = SigningService::new(LocalSigner::from_seed([9; 32], 1), MaxAmount(100), 7, 0, 4);
    let mut v8 = Verifier::new(8);
    v8.add_key(
        1,
        KeyEntry {
            public: signer.public(),
            valid_from_ms: 0,
            valid_until_ms: u64::MAX,
            revoked: false,
        },
    );
    let mut e = s7.sign(payload(1), 0, 1000).unwrap();
    e.chain_id = 8; // retargeting the same signature to another chain
    assert_eq!(v8.verify(&e, 0), Err(VerifyError::BadSignature));
}

#[test]
fn expiry_boundaries() {
    let (mut s, mut v) = pair();
    let e = s.sign(payload(1), 0, 1000).unwrap();
    assert_eq!(v.verify(&e, 999), Ok(()));
    let e2 = s.sign(payload(1), 0, 1000).unwrap();
    assert_eq!(v.verify(&e2, 1000), Err(VerifyError::Expired));
}

#[test]
fn ttl_limits() {
    let (mut s, mut v) = pair();
    assert_eq!(s.sign(payload(1), 0, 0).unwrap_err(), SignError::BadTtl(0));
    assert_eq!(
        s.sign(payload(1), 0, MAX_TTL_MS + 1).unwrap_err(),
        SignError::BadTtl(MAX_TTL_MS + 1)
    );
    let e = s.sign(payload(1), 0, MAX_TTL_MS).unwrap();
    assert_eq!(v.verify(&e, 0), Ok(()));
    // forged long-lived envelope (signed by the same key)
    let signer = LocalSigner::from_seed([9; 32], 1);
    let mut f = Envelope {
        chain_id: 7,
        key_id: 1,
        nonce: 99,
        expires_ms: MAX_TTL_MS + 1,
        payload: vec![],
        sig: [0; 64],
    };
    f.sig = signer.sign(&f.signing_bytes());
    assert_eq!(v.verify(&f, 0), Err(VerifyError::TtlTooLong));
}

#[test]
fn policy_and_size_checks() {
    let (mut s, _) = pair();
    assert!(matches!(
        s.sign(payload(101), 0, 10),
        Err(SignError::Policy(_))
    ));
    assert!(s.sign(payload(100), 0, 10).is_ok());
    assert!(matches!(
        s.sign(vec![1, 2], 0, 10),
        Err(SignError::Policy(_))
    ));
    assert_eq!(
        s.sign(vec![0; MAX_PAYLOAD + 1], 0, 10).unwrap_err(),
        SignError::PayloadTooLong
    );
}

#[test]
fn refused_requests_do_not_burn_nonces() {
    let (mut s, mut v) = pair();
    let _ = s.sign(payload(1000), 0, 10);
    let e = s.sign(payload(1), 0, 10).unwrap();
    assert_eq!(e.nonce, 0);
    assert_eq!(v.verify(&e, 0), Ok(()));
}

#[test]
fn key_windows_and_revocation() {
    let signer = LocalSigner::from_seed([9; 32], 1);
    let mut s = SigningService::new(LocalSigner::from_seed([9; 32], 1), MaxAmount(100), 7, 0, 4);
    let mut v = Verifier::new(7);
    v.add_key(
        1,
        KeyEntry {
            public: signer.public(),
            valid_from_ms: 100,
            valid_until_ms: 200,
            revoked: false,
        },
    );
    let e = s.sign(payload(1), 100, 50).unwrap();
    assert_eq!(v.verify(&e, 99), Err(VerifyError::KeyNotValidNow(1)));
    assert_eq!(v.verify(&e, 100), Ok(()));
    let e = s.sign(payload(1), 150, 1000).unwrap();
    assert_eq!(v.verify(&e, 199), Ok(()));
    let e = s.sign(payload(1), 150, 1000).unwrap();
    assert_eq!(v.verify(&e, 200), Err(VerifyError::KeyNotValidNow(1)));
    assert!(v.revoke(1));
    assert!(!v.revoke(5));
    let e = s.sign(payload(1), 150, 1000).unwrap();
    assert_eq!(v.verify(&e, 150), Err(VerifyError::Revoked(1)));
}

#[test]
fn rotation_with_overlap() {
    let old = LocalSigner::from_seed([1; 32], 1);
    let new = LocalSigner::from_seed([2; 32], 2);
    let mut v = Verifier::new(7);
    v.add_key(
        1,
        KeyEntry {
            public: old.public(),
            valid_from_ms: 0,
            valid_until_ms: 2000,
            revoked: false,
        },
    );
    v.add_key(
        2,
        KeyEntry {
            public: new.public(),
            valid_from_ms: 1000,
            valid_until_ms: u64::MAX,
            revoked: false,
        },
    );
    let mut so = SigningService::new(old, MaxAmount(9), 7, 0, 2);
    let mut sn = SigningService::new(new, MaxAmount(9), 7, 0, 2);
    // during the overlap both are accepted, with independent nonce counters
    assert_eq!(
        v.verify(&so.sign(payload(1), 1500, 100).unwrap(), 1500),
        Ok(())
    );
    assert_eq!(
        v.verify(&sn.sign(payload(1), 1500, 100).unwrap(), 1500),
        Ok(())
    );
    assert_eq!(v.last_nonce(1), Some(0));
    assert_eq!(v.last_nonce(2), Some(0));
    // after the old window closes only the new one works
    assert_eq!(
        v.verify(&so.sign(payload(1), 2000, 100).unwrap(), 2000),
        Err(VerifyError::KeyNotValidNow(1))
    );
    assert_eq!(
        v.verify(&sn.sign(payload(1), 2000, 100).unwrap(), 2000),
        Ok(())
    );
}

#[test]
fn nonce_allocator_reserves_before_use_and_survives_restart() {
    let mut a = NonceAllocator::new(0, 4);
    assert_eq!(a.take_pending(), None);
    assert_eq!(a.allocate(), 0);
    assert_eq!(a.take_pending(), Some(4));
    assert_eq!(a.take_pending(), None);
    assert_eq!([a.allocate(), a.allocate(), a.allocate()], [1, 2, 3]);
    assert_eq!(a.take_pending(), None);
    assert_eq!(a.allocate(), 4);
    assert_eq!(a.take_pending(), Some(8));
    // crash: the stored reservation is 8; a restarted allocator starts there
    let mut b = NonceAllocator::new(8, 4);
    assert_eq!(b.allocate(), 8);
    assert_eq!(b.take_pending(), Some(12));
}

#[test]
fn restart_never_reuses_a_nonce() {
    let mut rng = Rng::new(5);
    let mut stored = 0u64;
    let mut used = std::collections::BTreeSet::new();
    let mut s = SigningService::new(
        LocalSigner::from_seed([9; 32], 1),
        MaxAmount(9),
        7,
        stored,
        5,
    );
    for _ in 0..500 {
        let e = s.sign(payload(1), 0, 10).unwrap();
        if let Some(r) = s.take_reservation() {
            stored = r;
        }
        assert!(used.insert(e.nonce), "reused {}", e.nonce);
        if rng.chance(100) {
            s = SigningService::new(
                LocalSigner::from_seed([9; 32], 1),
                MaxAmount(9),
                7,
                stored,
                5,
            );
        }
    }
}

#[test]
fn constants_are_the_documented_values() {
    assert_eq!(MAX_PAYLOAD, 65_536);
    assert_eq!(MAX_TTL_MS, 600_000);
}

#[test]
fn payload_size_boundaries_use_literal_numbers() {
    let (mut s, _) = pair();
    let ok = s.sign(vec![0; 65_536], 0, 10).unwrap();
    assert_eq!(ok.payload.len(), 65_536);
    // and the maximum survives an encode/decode round trip
    assert_eq!(Envelope::decode(&ok.encode()).unwrap(), ok);
    assert_eq!(
        s.sign(vec![0; 65_537], 0, 10).unwrap_err(),
        SignError::PayloadTooLong
    );
    assert!(s.sign(payload(1), 0, 600_000).is_ok());
    assert_eq!(
        s.sign(payload(1), 0, 600_001).unwrap_err(),
        SignError::BadTtl(600_001)
    );
}

#[test]
#[should_panic(expected = "payload too long")]
fn encode_refuses_oversize_payload() {
    let e = Envelope {
        chain_id: 0,
        key_id: 0,
        nonce: 0,
        expires_ms: 0,
        payload: vec![0; 65_537],
        sig: [0; 64],
    };
    let _ = e.encode();
}

#[test]
fn expiry_is_now_plus_ttl() {
    let (mut s, _) = pair();
    assert_eq!(s.sign(payload(1), 1_000, 250).unwrap().expires_ms, 1_250);
}

#[test]
fn allocator_starts_at_the_stored_high_water_mark() {
    let mut a = NonceAllocator::new(40, 8);
    assert_eq!(a.allocate(), 40);
    assert_eq!(a.take_pending(), Some(48));
    assert_eq!(a.allocate(), 41);
}

#[test]
fn small_order_public_key_is_rejected_by_strict_verification() {
    // The identity point as a public key, with R = identity and S = 0, satisfies the
    // *cofactored-or-not* group equation for every message: a classic forgery that
    // `verify_strict` refuses.
    let mut pk = [0u8; 32];
    pk[0] = 1;
    let mut sig = [0u8; 64];
    sig[0] = 1;
    let mut v = Verifier::new(7);
    v.add_key(
        1,
        KeyEntry {
            public: pk,
            valid_from_ms: 0,
            valid_until_ms: u64::MAX,
            revoked: false,
        },
    );
    let e = Envelope {
        chain_id: 7,
        key_id: 1,
        nonce: 1,
        expires_ms: 100,
        payload: b"anything".to_vec(),
        sig,
    };
    assert_eq!(v.verify(&e, 0), Err(VerifyError::BadSignature));
}

#[test]
fn revocation_is_sticky_for_that_key_only() {
    let a = LocalSigner::from_seed([1; 32], 1);
    let b = LocalSigner::from_seed([2; 32], 2);
    let mut v = Verifier::new(7);
    v.add_key(
        1,
        KeyEntry {
            public: a.public(),
            valid_from_ms: 0,
            valid_until_ms: u64::MAX,
            revoked: false,
        },
    );
    v.add_key(
        2,
        KeyEntry {
            public: b.public(),
            valid_from_ms: 0,
            valid_until_ms: u64::MAX,
            revoked: false,
        },
    );
    assert!(v.revoke(1));
    let mut sa = SigningService::new(a, MaxAmount(9), 7, 0, 2);
    let mut sb = SigningService::new(b, MaxAmount(9), 7, 0, 2);
    assert_eq!(
        v.verify(&sa.sign(payload(1), 0, 10).unwrap(), 0),
        Err(VerifyError::Revoked(1))
    );
    assert_eq!(v.verify(&sb.sign(payload(1), 0, 10).unwrap(), 0), Ok(()));
}

#[test]
fn debug_output_never_contains_key_bytes() {
    let s = LocalSigner::from_seed([0x41; 32], 3);
    let d = format!("{s:?}");
    assert_eq!(d, "LocalSigner { key_id: 3, key: <redacted> }");
}
```

### `tests/rng_extra.rs`

```rust
use relaylab::rng::Rng;

#[test]
fn chance_zero_never_and_thousand_always() {
    let mut r = Rng::new(9);
    assert!((0..50_000).all(|_| !r.chance(0)));
    assert!((0..50_000).all(|_| r.chance(1000)));
}

#[test]
fn below_covers_the_whole_range() {
    let mut r = Rng::new(11);
    let mut seen = [false; 5];
    for _ in 0..200 {
        seen[r.below(5) as usize] = true;
    }
    assert!(seen.iter().all(|&s| s));
    assert_eq!(Rng::new(5).below(1), 0);
}
```

### `tests/webhook_tests.rs`

```rust
use relaylab::webhook::*;

#[test]
fn rfc4231_vectors() {
    let hex = |b: [u8; 32]| b.iter().map(|x| format!("{x:02x}")).collect::<String>();
    assert_eq!(
        hex(hmac_sha256(&[0x0b; 20], b"Hi There")),
        "b0344c61d8db38535ca8afceaf0bf12b881dc200c9833da726e9376c2e32cff7"
    );
    assert_eq!(
        hex(hmac_sha256(b"Jefe", b"what do ya want for nothing?")),
        "5bdcc146bf60754e6a042426089575c75a003f089d2739839dec58b964ec3843"
    );
}

#[test]
fn header_known_answer() {
    let h = sign(b"k", 1700000000000, br#"{"a":1}"#);
    assert_eq!(
        h,
        "t=1700000000000,v1=4284e27cb13c82554d08fc92e741b8ae1c126aa41c93a3060481666c36909b51"
    );
}

const NOW: u64 = 1_700_000_000_000;

#[test]
fn accepts_valid() {
    let h = sign(b"k", NOW, b"body");
    assert_eq!(verify(&[b"k"], &h, b"body", NOW, 300_000), Ok(()));
}

#[test]
fn rejects_wrong_body_key_and_tag() {
    let h = sign(b"k", NOW, b"body");
    assert_eq!(
        verify(&[b"k"], &h, b"bodY", NOW, 1000),
        Err(WebhookError::BadSignature)
    );
    assert_eq!(
        verify(&[b"j"], &h, b"body", NOW, 1000),
        Err(WebhookError::BadSignature)
    );
    let mut bad = h.clone();
    bad.pop();
    bad.push(if h.ends_with('0') { '1' } else { '0' });
    assert_eq!(
        verify(&[b"k"], &bad, b"body", NOW, 1000),
        Err(WebhookError::BadSignature)
    );
    assert_eq!(
        verify(&[], &h, b"body", NOW, 1000),
        Err(WebhookError::BadSignature)
    );
}

#[test]
fn timestamp_is_covered() {
    let h = sign(b"k", NOW, b"body");
    let forged = h.replacen(&NOW.to_string(), &(NOW + 1).to_string(), 1);
    assert_eq!(
        verify(&[b"k"], &forged, b"body", NOW, 1000),
        Err(WebhookError::BadSignature)
    );
}

#[test]
fn tolerance_boundaries() {
    let h = sign(b"k", NOW, b"b");
    assert_eq!(verify(&[b"k"], &h, b"b", NOW + 1000, 1000), Ok(()));
    assert_eq!(
        verify(&[b"k"], &h, b"b", NOW + 1001, 1000),
        Err(WebhookError::Stale)
    );
    assert_eq!(verify(&[b"k"], &h, b"b", NOW - 1000, 1000), Ok(()));
    assert_eq!(
        verify(&[b"k"], &h, b"b", NOW - 1001, 1000),
        Err(WebhookError::Stale)
    );
}

#[test]
fn stale_is_checked_even_with_valid_mac() {
    let h = sign(b"k", 5, b"b");
    assert_eq!(
        verify(&[b"k"], &h, b"b", 10_000_000, 1000),
        Err(WebhookError::Stale)
    );
}

#[test]
fn rotation_accepts_either_key() {
    let old = sign(b"old", NOW, b"b");
    let new = sign(b"new", NOW, b"b");
    for h in [&old, &new] {
        assert_eq!(verify(&[b"new", b"old"], h, b"b", NOW, 10), Ok(()));
    }
    assert_eq!(
        verify(&[b"new"], &old, b"b", NOW, 10),
        Err(WebhookError::BadSignature)
    );
}

#[test]
fn malformed_headers() {
    let good = sign(b"k", NOW, b"b");
    let tag = &good[good.len() - 64..];
    let cases = [
        String::new(),
        "garbage".into(),
        format!("t={NOW}v1={tag}"),
        format!("t=,v1={tag}"),
        format!("t=0{NOW},v1={tag}"),
        format!("t=+{NOW},v1={tag}"),
        format!("t= {NOW},v1={tag}"),
        format!("t=99999999999999999999999,v1={tag}"),
        format!("t={NOW},v1={}", &tag[..62]),
        format!("t={NOW},v1={tag}00"),
        format!("t={NOW},v1={}", tag.to_uppercase()),
        format!("t={NOW},v1={}g", &tag[..63]),
        format!("x={NOW},v1={tag}"),
        format!(" t={NOW},v1={tag}"),
    ];
    for c in cases {
        assert_eq!(
            verify(&[b"k"], &c, b"b", NOW, 10),
            Err(WebhookError::MalformedHeader),
            "{c:?}"
        );
    }
}

#[test]
fn zero_timestamp_is_allowed() {
    let h = sign(b"k", 0, b"b");
    assert!(h.starts_with("t=0,v1="));
    assert_eq!(verify(&[b"k"], &h, b"b", 0, 0), Ok(()));
}

#[test]
fn short_leading_zero_timestamps_are_malformed() {
    let tag = &sign(b"k", 5, b"b")[4..];
    for ts in ["05", "00", "007"] {
        let h = format!("t={ts},{}", tag.trim_start_matches("t=5,"));
        assert_eq!(
            verify(&[b"k"], &h, b"b", 5, 10),
            Err(WebhookError::MalformedHeader),
            "{h}"
        );
    }
}
```

### `tests/journal_tests.rs`

```rust
use relaylab::journal::*;

fn sample() -> (Journal, Vec<Vec<u8>>) {
    let recs: Vec<Vec<u8>> = vec![b"one".to_vec(), vec![], vec![7; 300], b"four".to_vec()];
    let mut j = Journal::new();
    for r in &recs {
        j.append(r);
    }
    (j, recs)
}

#[test]
fn round_trip_and_layout() {
    let (j, recs) = sample();
    assert_eq!(
        j.bytes().len(),
        recs.iter().map(|r| r.len() + 8).sum::<usize>()
    );
    assert_eq!(&j.bytes()[..4], &3u32.to_be_bytes());
    assert_eq!(&j.bytes()[4..7], b"one");
    let (got, valid) = recover(j.bytes());
    assert_eq!(got, recs);
    assert_eq!(valid, j.bytes().len());
}

#[test]
fn empty_input() {
    assert_eq!(recover(&[]), (vec![], 0));
    let (j, r) = Journal::from_bytes(&[]);
    assert!(j.bytes().is_empty() && r.is_empty());
}

#[test]
fn truncation_at_every_offset_yields_a_prefix() {
    let (j, recs) = sample();
    let b = j.bytes();
    let mut ends = vec![0];
    for r in &recs {
        ends.push(ends.last().unwrap() + r.len() + 8);
    }
    for cut in 0..=b.len() {
        let (got, valid) = recover(&b[..cut]);
        let n = ends.iter().filter(|&&e| e <= cut && e > 0).count();
        assert_eq!(got, recs[..n].to_vec(), "cut {cut}");
        assert_eq!(valid, ends[n], "cut {cut}");
    }
}

#[test]
fn every_single_bit_flip_is_detected() {
    let (j, recs) = sample();
    let b = j.bytes().to_vec();
    for i in 0..b.len() {
        for bit in 0..8 {
            let mut c = b.clone();
            c[i] ^= 1 << bit;
            let (got, _) = recover(&c);
            assert!(got.len() < recs.len(), "flip {i}:{bit} went unnoticed");
            // what is returned is still an untouched prefix
            assert_eq!(got, recs[..got.len()].to_vec(), "flip {i}:{bit}");
        }
    }
}

#[test]
fn oversize_length_stops_recovery() {
    let mut b = ((MAX_RECORD + 1) as u32).to_be_bytes().to_vec();
    b.extend_from_slice(&[0; 32]);
    assert_eq!(recover(&b), (vec![], 0));
}

#[test]
fn from_bytes_drops_garbage_tail_so_appends_stay_valid() {
    let (j, recs) = sample();
    let mut b = j.bytes().to_vec();
    b.extend_from_slice(b"\x00\x00\x00\x09partial");
    let (mut j2, got) = Journal::from_bytes(&b);
    assert_eq!(got, recs);
    j2.append(b"after");
    let (got2, valid) = recover(j2.bytes());
    assert_eq!(got2.len(), recs.len() + 1);
    assert_eq!(got2.last().unwrap(), b"after");
    assert_eq!(valid, j2.bytes().len());
}

#[test]
#[should_panic]
fn append_too_long_panics() {
    Journal::new().append(&vec![0; MAX_RECORD + 1]);
}

#[test]
fn max_size_record_ok() {
    let mut j = Journal::new();
    j.append(&vec![1; MAX_RECORD]);
    assert_eq!(recover(j.bytes()).0.len(), 1);
}

#[test]
fn checksum_known_answers() {
    // computed independently: first 4 bytes of SHA-256("lab/journal/v1" || len_be || payload)
    let mut j = Journal::new();
    j.append(b"abc");
    assert_eq!(
        j.bytes(),
        [0, 0, 0, 3, b'a', b'b', b'c', 0x02, 0x88, 0x9c, 0x56]
    );
    let mut e = Journal::new();
    e.append(b"");
    assert_eq!(e.bytes(), [0, 0, 0, 0, 0xe6, 0x66, 0xa6, 0x79]);
}

#[test]
fn oversize_record_with_a_valid_checksum_is_still_rejected() {
    use sha2::{Digest, Sha256};
    let payload = vec![5u8; MAX_RECORD + 1];
    let len = (payload.len() as u32).to_be_bytes();
    let mut h = Sha256::new();
    h.update(b"lab/journal/v1");
    h.update(len);
    h.update(&payload);
    let d = h.finalize();
    let mut b = len.to_vec();
    b.extend_from_slice(&payload);
    b.extend_from_slice(&d[..4]);
    assert_eq!(recover(&b), (vec![], 0));
}
```

### `tests/queue_tests.rs`

```rust
use relaylab::queue::*;

fn q() -> Queue {
    Queue::new(3)
}

#[test]
fn event_encoding_round_trip_and_strict() {
    let evs = [
        Event::Enqueued {
            id: 1,
            key: 2,
            ready_at: 3,
            payload: b"xyz".to_vec(),
        },
        Event::Enqueued {
            id: 1,
            key: 2,
            ready_at: 3,
            payload: vec![],
        },
        Event::Leased { id: 4, until: 5 },
        Event::Completed { id: 6 },
        Event::Failed { id: 7, retry_at: 8 },
        Event::Died { id: 9 },
    ];
    for e in evs {
        let b = e.encode();
        assert_eq!(Event::decode(&b), Some(e.clone()));
        for n in 0..b.len() {
            assert_eq!(Event::decode(&b[..n]), None, "{e:?} cut {n}");
        }
        let mut t = b.clone();
        t.push(0);
        assert_eq!(Event::decode(&t), None);
    }
    assert_eq!(Event::decode(&[0]), None);
    assert_eq!(Event::decode(&[6, 0, 0, 0, 0, 0, 0, 0, 0]), None);
    assert_eq!(
        Event::decode(&Event::Completed { id: 6 }.encode()[..]),
        Some(Event::Completed { id: 6 })
    );
    assert_eq!(
        Event::Leased { id: 1, until: 2 }.encode(),
        [2, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 2]
    );
}

#[test]
fn lifecycle() {
    let mut q = q();
    let Enqueued::New(id) = q.enqueue(10, b"p".to_vec(), 0) else {
        panic!()
    };
    assert_eq!(q.status(id), Some(Status::Pending { ready_at: 0 }));
    let l = q.lease(0, 100).unwrap();
    assert_eq!((l.id, l.attempt, l.payload.as_slice()), (id, 1, &b"p"[..]));
    assert_eq!(q.status(id), Some(Status::Leased { until: 100 }));
    assert_eq!(q.lease(0, 100), None);
    q.complete(id, 1).unwrap();
    assert_eq!(q.status(id), Some(Status::Done));
    assert_eq!(
        q.stats(),
        Stats {
            pending: 0,
            leased: 0,
            done: 1,
            dead: 0
        }
    );
    assert_eq!(q.lease(1000, 100), None);
}

#[test]
fn duplicate_keys_are_ignored() {
    let mut q = q();
    let Enqueued::New(a) = q.enqueue(1, b"a".to_vec(), 0) else {
        panic!()
    };
    assert_eq!(q.enqueue(1, b"other".to_vec(), 5), Enqueued::Duplicate(a));
    let Enqueued::New(b) = q.enqueue(2, b"b".to_vec(), 0) else {
        panic!()
    };
    assert_ne!(a, b);
    assert_eq!(q.stats().pending, 2);
    assert_eq!(q.key_of(b), Some(2));
}

#[test]
fn fifo_by_id_and_ready_time() {
    let mut q = q();
    q.enqueue(1, b"a".to_vec(), 50);
    q.enqueue(2, b"b".to_vec(), 0);
    assert_eq!(q.lease(10, 5).unwrap().payload, b"b");
    assert_eq!(q.lease(10, 5), None); // a is not due
    assert_eq!(q.lease(50, 5).unwrap().payload, b"a");
}

#[test]
fn failure_backs_off_then_retries() {
    let mut q = q();
    let Enqueued::New(id) = q.enqueue(1, vec![], 0) else {
        panic!()
    };
    let l = q.lease(0, 100).unwrap();
    assert_eq!(q.fail(id, l.attempt, 10, 40), Ok(FailOutcome::Retry(50)));
    assert_eq!(q.lease(49, 100), None);
    let l2 = q.lease(50, 100).unwrap();
    assert_eq!(l2.attempt, 2);
    assert_eq!(q.attempts(id), Some(2));
}

#[test]
fn dead_letter_after_max_attempts() {
    let mut q = q();
    let Enqueued::New(id) = q.enqueue(1, vec![], 0) else {
        panic!()
    };
    for n in 1..=3u32 {
        let l = q.lease(n as u64 * 1000, 10).unwrap();
        assert_eq!(l.attempt, n);
        let out = q.fail(id, n, n as u64 * 1000, 0).unwrap();
        assert_eq!(
            out,
            if n < 3 {
                FailOutcome::Retry(n as u64 * 1000)
            } else {
                FailOutcome::Dead
            }
        );
    }
    assert_eq!(q.status(id), Some(Status::Dead));
    assert_eq!(q.lease(99999, 10), None);
    assert_eq!(q.stats().dead, 1);
}

#[test]
fn expired_lease_is_retaken_and_old_holder_is_stale() {
    let mut q = q();
    let Enqueued::New(id) = q.enqueue(1, vec![], 0) else {
        panic!()
    };
    let first = q.lease(0, 100).unwrap();
    assert_eq!(q.lease(99, 100), None);
    let second = q.lease(100, 100).unwrap();
    assert_eq!(second.attempt, 2);
    assert_eq!(q.complete(id, first.attempt), Err(QueueError::StaleLease));
    assert_eq!(
        q.fail(id, first.attempt, 100, 0),
        Err(QueueError::StaleLease)
    );
    assert_eq!(q.complete(id, second.attempt), Ok(()));
}

#[test]
fn lease_that_expires_after_last_attempt_is_dead() {
    let mut q = Queue::new(2);
    let Enqueued::New(id) = q.enqueue(1, vec![], 0) else {
        panic!()
    };
    q.lease(0, 10).unwrap();
    q.lease(10, 10).unwrap(); // second attempt, never reported
    assert_eq!(q.lease(20, 10), None);
    assert_eq!(q.status(id), Some(Status::Dead));
}

#[test]
fn complete_and_fail_errors() {
    let mut q = q();
    assert_eq!(q.complete(9, 1), Err(QueueError::Unknown));
    assert_eq!(q.fail(9, 1, 0, 0), Err(QueueError::Unknown));
    let Enqueued::New(id) = q.enqueue(1, vec![], 0) else {
        panic!()
    };
    assert_eq!(q.complete(id, 1), Err(QueueError::NotLeased));
    let l = q.lease(0, 10).unwrap();
    q.complete(id, l.attempt).unwrap();
    assert_eq!(q.complete(id, l.attempt), Ok(())); // idempotent
    assert_eq!(q.fail(id, l.attempt, 0, 0), Err(QueueError::NotLeased));
    assert_eq!(q.complete(id, 5), Err(QueueError::NotLeased));
}

#[test]
fn recovery_equals_live_state() {
    let mut q = q();
    for k in 0..6 {
        q.enqueue(k, vec![k as u8], 0);
    }
    let a = q.lease(0, 100).unwrap();
    q.complete(a.id, a.attempt).unwrap();
    let b = q.lease(0, 100).unwrap();
    q.fail(b.id, b.attempt, 0, 30).unwrap();
    let c = q.lease(0, 100).unwrap();
    let _ = c;
    let r = Queue::recover(q.journal_bytes(), 3);
    assert_eq!(r.stats(), q.stats());
    for id in 1..=6 {
        assert_eq!(r.status(id), q.status(id));
        assert_eq!(r.attempts(id), q.attempts(id));
        assert_eq!(r.key_of(id), q.key_of(id));
    }
    assert_eq!(r.journal_bytes(), q.journal_bytes());
}

#[test]
fn recovery_keeps_id_allocation_and_dedup() {
    let mut q = q();
    q.enqueue(1, vec![], 0);
    q.enqueue(2, vec![], 0);
    let mut r = Queue::recover(q.journal_bytes(), 3);
    assert_eq!(r.enqueue(2, vec![], 0), Enqueued::Duplicate(2));
    assert_eq!(r.enqueue(3, vec![], 0), Enqueued::New(3));
}

#[test]
fn recovery_from_every_truncation_gives_a_consistent_prefix() {
    let mut q = q();
    for k in 0..4 {
        q.enqueue(k, vec![k as u8; 5], 0);
    }
    let l = q.lease(0, 10).unwrap();
    q.fail(l.id, l.attempt, 0, 5).unwrap();
    let l = q.lease(5, 10).unwrap();
    q.complete(l.id, l.attempt).unwrap();
    let bytes = q.journal_bytes().to_vec();
    let mut last_total = 0;
    for cut in 0..=bytes.len() {
        let r = Queue::recover(&bytes[..cut], 3);
        let s = r.stats();
        let total = s.pending + s.leased + s.done + s.dead;
        assert!(total >= last_total);
        last_total = total;
    }
    assert_eq!(last_total, 4);
}

#[test]
fn corrupted_tail_is_ignored_and_queue_continues() {
    let mut q = q();
    q.enqueue(1, vec![], 0);
    q.enqueue(2, vec![], 0);
    let mut b = q.journal_bytes().to_vec();
    let n = b.len();
    b[n - 1] ^= 0xff;
    let mut r = Queue::recover(&b, 3);
    assert_eq!(r.stats().pending, 1);
    assert_eq!(r.enqueue(2, vec![], 0), Enqueued::New(2));
    let r2 = Queue::recover(r.journal_bytes(), 3);
    assert_eq!(r2.stats().pending, 2);
}

#[test]
fn undecodable_record_stops_replay() {
    let mut j = relaylab::journal::Journal::new();
    j.append(
        &Event::Enqueued {
            id: 1,
            key: 1,
            ready_at: 0,
            payload: vec![],
        }
        .encode(),
    );
    j.append(&[99, 99]);
    j.append(
        &Event::Enqueued {
            id: 2,
            key: 2,
            ready_at: 0,
            payload: vec![],
        }
        .encode(),
    );
    let r = Queue::recover(j.bytes(), 3);
    assert_eq!(r.stats().pending, 1);
}

#[test]
fn has_ready_matches_lease() {
    let mut q = Queue::new(2);
    assert!(!q.has_ready(0));
    q.enqueue(1, vec![], 10);
    assert!(!q.has_ready(9));
    assert!(q.has_ready(10));
    q.lease(10, 5).unwrap();
    assert!(!q.has_ready(14));
    assert!(q.has_ready(15)); // expired lease
    q.lease(15, 5).unwrap(); // second and last attempt
    assert!(!q.has_ready(20)); // nothing left to retry
    q.reap(20);
    assert_eq!(q.stats().dead, 1);
}

#[test]
fn reap_only_touches_exhausted_expired_leases() {
    let mut q = Queue::new(1);
    q.enqueue(1, vec![], 0);
    q.enqueue(2, vec![], 0);
    q.lease(0, 10).unwrap();
    q.reap(9);
    assert_eq!(
        q.stats(),
        Stats {
            pending: 1,
            leased: 1,
            done: 0,
            dead: 0
        }
    );
    q.reap(10);
    assert_eq!(
        q.stats(),
        Stats {
            pending: 1,
            leased: 0,
            done: 0,
            dead: 1
        }
    );
}

#[test]
fn decode_accepts_a_payload_of_exactly_the_record_limit() {
    let mut b = vec![1u8];
    b.extend_from_slice(&1u64.to_be_bytes());
    b.extend_from_slice(&2u64.to_be_bytes());
    b.extend_from_slice(&3u64.to_be_bytes());
    b.extend_from_slice(&(relaylab::journal::MAX_RECORD as u32).to_be_bytes());
    b.extend_from_slice(&vec![0; relaylab::journal::MAX_RECORD]);
    assert!(Event::decode(&b).is_some());
    let n = b.len();
    b[n - relaylab::journal::MAX_RECORD - 4..n - relaylab::journal::MAX_RECORD]
        .copy_from_slice(&(relaylab::journal::MAX_RECORD as u32 + 1).to_be_bytes());
    b.push(0);
    assert!(Event::decode(&b).is_none());
}

#[test]
fn leased_event_has_exactly_16_bytes_after_the_tag() {
    let b = Event::Leased { id: 1, until: 2 }.encode();
    assert_eq!(b.len(), 17);
    assert_eq!(Event::decode(&b[..16]), None);
}
```

### `tests/breaker_tests.rs`

```rust
use relaylab::breaker::*;

#[test]
fn opens_at_threshold() {
    let mut b = Breaker::new(3, 100);
    for t in 0..2 {
        b.record_failure(t);
        assert_eq!(b.state(t), State::Closed);
        assert!(b.allow(t));
    }
    b.record_failure(2);
    assert_eq!(b.state(2), State::Open);
    assert!(!b.allow(2));
}

#[test]
fn success_resets_count() {
    let mut b = Breaker::new(3, 100);
    b.record_failure(0);
    b.record_failure(0);
    b.record_success();
    b.record_failure(0);
    b.record_failure(0);
    assert_eq!(b.state(0), State::Closed);
}

#[test]
fn half_open_after_cooldown_with_one_probe() {
    let mut b = Breaker::new(1, 100);
    b.record_failure(10);
    assert!(!b.allow(109));
    assert_eq!(b.state(109), State::Open);
    assert_eq!(b.state(110), State::HalfOpen);
    assert!(b.allow(110));
    assert!(!b.allow(111)); // only one probe in flight
    b.record_success();
    assert_eq!(b.state(111), State::Closed);
    assert!(b.allow(111) && b.allow(111));
}

#[test]
fn failed_probe_reopens_with_full_cooldown() {
    let mut b = Breaker::new(1, 100);
    b.record_failure(0);
    assert!(b.allow(100));
    b.record_failure(100);
    assert_eq!(b.state(199), State::Open);
    assert_eq!(b.state(200), State::HalfOpen);
    assert!(b.allow(200)); // probe flag was cleared
}

#[test]
#[should_panic]
fn zero_threshold_rejected() {
    Breaker::new(0, 1);
}

#[test]
fn lost_probe_is_forgotten_after_another_cooldown() {
    let mut b = Breaker::new(1, 100);
    b.record_failure(0);
    assert!(b.allow(100)); // probe taken, result never reported
    assert!(!b.allow(150));
    assert!(!b.allow(199));
    assert!(b.allow(200)); // a new probe
}
```

### `tests/sim_tests.rs`

```rust
use relaylab::sim::*;

#[test]
fn no_faults_each_job_once() {
    let p = Params {
        unavailable_pm: 0,
        timeout_after_pm: 0,
        crash_pm: 0,
        jobs: 20,
        ..Params::default()
    };
    let r = run(1, &p);
    assert_eq!(
        (r.done, r.dead, r.distinct_effects, r.double_applied),
        (20, 0, 20, 0)
    );
    assert_eq!(r.submissions, 20);
    assert_eq!(r.crashes, 0);
    assert_eq!(r.rejected, 0);
}

#[test]
fn deterministic_per_seed() {
    let p = Params::default();
    assert_eq!(run(42, &p), run(42, &p));
    assert_ne!(run(42, &p), run(43, &p));
}

#[test]
fn chaos_with_dedupe_is_exactly_once_effect() {
    let p = Params::default();
    for seed in 0..200 {
        let r = run(seed, &p);
        assert_eq!(r.dead, 0, "seed {seed}");
        assert_eq!(r.done, p.jobs, "seed {seed}");
        assert_eq!(r.distinct_effects, p.jobs, "seed {seed}");
        assert_eq!(r.double_applied, 0, "seed {seed}");
        assert_eq!(r.rejected, 0, "seed {seed}");
    }
}

#[test]
fn chaos_actually_exercises_the_faults() {
    let p = Params::default();
    let rs: Vec<Report> = (0..50).map(|s| run(s, &p)).collect();
    assert!(rs.iter().all(|r| r.submissions > p.jobs));
    assert!(rs.iter().map(|r| r.crashes).sum::<u64>() > 100);
    assert!(rs.iter().map(|r| r.skipped_by_breaker).sum::<u64>() > 0);
    assert!(rs.iter().all(|r| r.nonce_high_water >= p.jobs));
}

#[test]
fn without_dedupe_timeouts_double_apply() {
    let p = Params {
        dedupe: false,
        ..Params::default()
    };
    let total: u64 = (0..30).map(|s| run(s, &p).double_applied).sum();
    assert!(
        total > 0,
        "retrying a timed-out non-idempotent submission must double apply"
    );
}

#[test]
fn without_faults_dedupe_makes_no_difference() {
    let p = Params {
        dedupe: false,
        unavailable_pm: 0,
        timeout_after_pm: 0,
        crash_pm: 0,
        ..Params::default()
    };
    assert_eq!(run(3, &p).double_applied, 0);
}

#[test]
fn hopeless_chain_ends_in_dead_letters() {
    let p = Params {
        unavailable_pm: 1000,
        jobs: 5,
        max_attempts: 3,
        crash_pm: 0,
        ..Params::default()
    };
    let r = run(1, &p);
    assert_eq!((r.done, r.dead, r.distinct_effects), (0, 5, 0));
    assert!(r.skipped_by_breaker > 0);
}

#[test]
fn tx_id_is_stable_and_depends_on_payload() {
    assert_eq!(tx_id(b"a"), tx_id(b"a"));
    assert_ne!(tx_id(b"a"), tx_id(b"b"));
}

#[test]
fn timeouts_alone_double_apply_on_a_blind_chain_but_not_on_a_deduplicating_one() {
    let base = Params {
        unavailable_pm: 0,
        timeout_after_pm: 500,
        crash_pm: 0,
        jobs: 40,
        ..Params::default()
    };
    let blind = run(
        3,
        &Params {
            dedupe: false,
            ..base.clone()
        },
    );
    assert!(blind.double_applied > 0);
    let safe = run(
        3,
        &Params {
            dedupe: true,
            ..base
        },
    );
    assert_eq!(
        (safe.double_applied, safe.done, safe.distinct_effects),
        (0, 40, 40)
    );
    assert!(safe.submissions > 40);
}

#[test]
fn a_relayer_that_restarts_after_every_submission_burns_a_whole_nonce_block_each_time() {
    let p = Params {
        crash_pm: 1000,
        unavailable_pm: 0,
        timeout_after_pm: 0,
        jobs: 3,
        max_attempts: 4,
        ..Params::default()
    };
    let r = run(1, &p);
    assert!(r.submissions > 0);
    assert_eq!(r.nonce_high_water, 16 * r.submissions);
    assert_eq!(r.rejected, 0); // gaps are fine, reuse is not
}

#[test]
fn without_crashes_the_nonce_block_is_used_up() {
    let p = Params {
        crash_pm: 0,
        unavailable_pm: 0,
        timeout_after_pm: 0,
        jobs: 40,
        ..Params::default()
    };
    let r = run(1, &p);
    assert_eq!(r.nonce_high_water, 48); // 40 nonces use three blocks of 16
}

#[test]
fn a_breaker_that_forgets_failures_on_success_stays_closed_at_a_low_fault_rate() {
    let p = Params {
        unavailable_pm: 50,
        timeout_after_pm: 0,
        crash_pm: 0,
        jobs: 100,
        ..Params::default()
    };
    for seed in 0..20 {
        let r = run(seed, &p);
        assert_eq!(r.skipped_by_breaker, 0, "seed {seed}");
        assert_eq!(r.done, 100);
    }
}

#[test]
fn backoff_waits_are_scheduled_only_after_failures() {
    let quiet = Params {
        unavailable_pm: 0,
        timeout_after_pm: 0,
        crash_pm: 0,
        ..Params::default()
    };
    assert_eq!(run(1, &quiet).total_wait_ms, 0);
    assert!(run(1, &Params::default()).total_wait_ms > 0);
}

#[test]
fn the_time_limit_stops_a_run_that_cannot_finish() {
    let p = Params {
        unavailable_pm: 1000,
        max_attempts: 1_000_000,
        crash_pm: 0,
        time_limit_ms: 2_000,
        jobs: 3,
        ..Params::default()
    };
    let r = run(1, &p);
    assert_eq!(r.done, 0);
    assert!(
        r.finished_at > 2_000 && r.finished_at <= 2_010,
        "{}",
        r.finished_at
    );
}

#[test]
fn jobs_whose_every_attempt_ends_in_a_crash_become_dead_letters() {
    let p = Params {
        crash_pm: 1000,
        unavailable_pm: 0,
        timeout_after_pm: 0,
        max_attempts: 2,
        jobs: 4,
        ..Params::default()
    };
    let r = run(1, &p);
    assert_eq!((r.done, r.dead), (0, 4));
    assert!(r.finished_at < 20_000, "{}", r.finished_at);
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
//! Rough timings for the main operations (release build).
use relaylab::journal::{Journal, recover};
use relaylab::queue::Queue;
use relaylab::sign::*;
use relaylab::sim::{Params, run};
use relaylab::webhook;
use std::hint::black_box;
use std::time::Instant;

fn time<F: FnMut()>(name: &str, n: u32, mut f: F) {
    let t = Instant::now();
    for _ in 0..n {
        f();
    }
    let per = t.elapsed().as_nanos() as f64 / n as f64;
    println!("{name:<28} {per:>12.0} ns/op");
}

fn main() {
    let signer = LocalSigner::from_seed([9; 32], 1);
    let mut v = Verifier::new(7);
    v.add_key(
        1,
        KeyEntry {
            public: signer.public(),
            valid_from_ms: 0,
            valid_until_ms: u64::MAX,
            revoked: false,
        },
    );
    let mut s = SigningService::new(signer, MaxAmount(u64::MAX), 7, 0, 1024);
    let payload = vec![1u8; 256];
    time("sign envelope (256 B)", 2000, || {
        black_box(s.sign(payload.clone(), 0, 1000).unwrap());
    });
    let envs: Vec<Envelope> = (0..2000)
        .map(|_| s.sign(payload.clone(), 0, 1000).unwrap())
        .collect();
    let mut i = 0;
    time("verify envelope (strict)", 2000, || {
        v.verify(&envs[i], 0).unwrap();
        i += 1;
    });
    let body = vec![7u8; 1024];
    let hdr = webhook::sign(b"secret", 1000, &body);
    time("webhook verify (1 KiB)", 20000, || {
        webhook::verify(&[b"secret"], &hdr, &body, 1000, 10).unwrap();
    });
    time("webhook verify, 2 keys", 20000, || {
        webhook::verify(&[b"old", b"secret"], &hdr, &body, 1000, 10).unwrap();
    });
    let mut j = Journal::new();
    time("journal append (64 B)", 20000, || {
        j.append(black_box(&[3u8; 64]))
    });
    let bytes = j.bytes().to_vec();
    time("journal recover (20k recs)", 20, || {
        black_box(recover(black_box(&bytes)));
    });
    time("queue enqueue+lease+complete", 2000, || {
        let mut q = Queue::new(3);
        q.enqueue(1, vec![0; 64], 0);
        let l = q.lease(0, 10).unwrap();
        q.complete(l.id, l.attempt).unwrap();
    });
    let p = Params::default();
    time("sim: 50 jobs under chaos", 20, || {
        black_box(run(1, &p));
    });
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **90 tests** (rng 2, signing and envelopes 28, webhooks 11, journal 10, queue 18, breaker 6, simulation 15) and 8 consecutive full runs without a failure (nothing uses sockets, real time or OS randomness, so there is nothing to flake).

**Mutation check ✓.** I made 131 small source edits (a "mutant") across the seven source modules, each in its own copy of the crate, and ran the whole test suite against each. **118 were detected (117 by a failing or aborting test, 1 as a hang), 13 are equivalent (M6, M7, M9, M17, M32, M54, M74, M87, M88, M117, M118, M119, M123), none survive unexplained.** As in the previous lessons, **What the first pass found.** Against my first, 69 tests, 12 mutants survived that were **real gaps** and one more (the ordinary-versus-strict signature check) was a gap that the first harness could not even express; the cure for each was a test, not an argument: the signature and payload size limits and the TTL constants were only compared through the constants themselves (so I added tests that use the literal numbers 65,536, 65,537, 600,000 and 600,001), the journal checksum was never compared with a known answer (so a checksum that dropped its domain label, its length field or half its bytes passed every round-trip and bit-flip test; added `checksum_known_answers`), a record over the size limit was never presented with a *valid* checksum, `chance(0)` was never run enough times to catch an off-by-one, a leading-zero timestamp of two digits was not tried, and ordinary Ed25519 verification was not distinguished from the strict one until I added the identity-key test. The second, simulation-level harness found seven more gaps: the simulator never proved that lost acknowledgements alone double-apply on a blind chain, that a restart burns a nonce block, that successes reset the breaker, that backoff waits are scheduled, that the time limit is honored, that exhausted leases are reaped, and that a payload of exactly the record limit decodes. Ten mutants needed their patterns rewritten before they were valid (a line had been re-wrapped by `cargo fmt`, a mutant did not compile, or it was an exact copy of the original); they are included in the table with their final result. My harness treats `#[should_panic]` failures (`- should panic ... FAILED`) as failures, and a process abort as detection. The tests ran in the dev profile.

| # | module | change | result |
|---|---|---|---|
| M1 | `rng` | xorshift shift 11 instead of 12 | caught by `rng_known_vectors` |
| M2 | `rng` | xorshift shift 24 instead of 25 | caught by `rng_known_vectors` |
| M3 | `rng` | xorshift shift 26 instead of 27 | caught by `rng_known_vectors` |
| M4 | `rng` | multiplier constant changed | caught by `rng_known_vectors` |
| M5 | `rng` | seed 0 not remapped | caught by `rng_known_vectors` |
| M6 | `rng` | rejection-sampling limit removed | **equivalent**: the two limits differ only when a draw lands in the last `2^64 mod n` values, probability about 2⁻⁶⁴ |
| M7 | `rng` | rejection test `<=` instead of `<` | **equivalent**: `x == limit` happens with probability about 2⁻⁶⁴ |
| M8 | `rng` | `chance` uses `<=` | caught by `chance_zero_never_and_thousand_always` |
| M9 | `rng` | `below(0)` assertion removed | **equivalent**: with `n = 0` the next line, `u64::MAX % n`, panics anyway, so the observable behavior (a panic) is the same |
| M10 | `sign` | domain length not signed | caught by `signing_bytes_layout` |
| M11 | `sign` | domain label not signed | caught by `signing_bytes_layout` |
| M12 | `sign` | chain id not signed | caught by `domain_separation_between_chains` |
| M13 | `sign` | nonce not signed | caught by `every_field_is_covered_by_the_signature` |
| M14 | `sign` | expiry not signed | caught by `every_field_is_covered_by_the_signature` |
| M15 | `sign` | payload length not signed | caught by `signing_bytes_layout` |
| M16 | `sign` | `encode` accepts oversize payloads | caught by `encode_refuses_oversize_payload` |
| M17 | `sign` | `decode` header-length check `<=` instead of `<` | **equivalent**: an input of exactly the header length is short of the full message either way, and the next check returns the same `Truncated` |
| M18 | `sign` | payload limit `>=` instead of `>` in `decode` | caught by `payload_size_boundaries_use_literal_numbers` |
| M19 | `sign` | `decode` length check `<=` instead of `<` | caught by `encode_decode_round_trip_and_layout` |
| M20 | `sign` | trailing bytes accepted | caught by `strict_decode` |
| M21 | `sign` | header length constant off by one | caught by `encode_decode_round_trip_and_layout` |
| M22 | `sign` | `MAX_TTL_MS` is 600,600 | caught by `constants_are_the_documented_values` |
| M23 | `sign` | `MAX_PAYLOAD` is 65,537 | caught by `encode_refuses_oversize_payload` |
| M24 | `sign` | policy limit `>=` instead of `>` | caught by `policy_and_size_checks` |
| M25 | `sign` | policy reads 7 bytes of amount | caught by `accepts_good_envelope` |
| M26 | `sign` | payload limit `>=` instead of `>` in `sign` | caught by `payload_size_boundaries_use_literal_numbers` |
| M27 | `sign` | zero TTL accepted | caught by `ttl_limits` |
| M28 | `sign` | TTL limit `>=` instead of `>` | caught by `payload_size_boundaries_use_literal_numbers` |
| M29 | `sign` | policy not applied | caught by `policy_and_size_checks` |
| M30 | `sign` | expiry set to `now` instead of `now + ttl` | caught by `accepts_good_envelope` |
| M31 | `sign` | allocator ignores the stored high-water mark (`next`) | caught by `allocator_starts_at_the_stored_high_water_mark` |
| M32 | `sign` | allocator ignores the stored high-water mark (`reserved`) | **equivalent**: `reserved` is overwritten on the first allocation, so its initial value is never read |
| M33 | `sign` | reservation taken one number late | caught by `allocator_starts_at_the_stored_high_water_mark` |
| M34 | `sign` | reservation block size forced to 1 | caught by `allocator_starts_at_the_stored_high_water_mark` |
| M35 | `sign` | allocator skips every second number | caught by `allocator_starts_at_the_stored_high_water_mark` |
| M36 | `sign` | revoked keys accepted | caught by `key_windows_and_revocation` |
| M37 | `sign` | `valid_from` not checked | caught by `key_windows_and_revocation` |
| M38 | `sign` | `valid_until` not checked | caught by `key_windows_and_revocation` |
| M39 | `sign` | `valid_from` exclusive | caught by `bad_signature_does_not_advance_nonce` |
| M40 | `sign` | `valid_until` inclusive | caught by `key_windows_and_revocation` |
| M41 | `sign` | expiry inclusive | caught by `expiry_boundaries` |
| M42 | `sign` | TTL cap check `>=` instead of `>` | caught by `ttl_limits` |
| M43 | `sign` | ordinary `verify` instead of `verify_strict` | caught by `small_order_public_key_is_rejected_by_strict_verification` |
| M44 | `sign` | replay check `<` instead of `<=` | caught by `replay_rejected_and_order_enforced` |
| M45 | `sign` | wrong nonce recorded after verification | caught by `replay_rejected_and_order_enforced` |
| M46 | `sign` | chain id not checked | caught by `wrong_chain_and_unknown_key` |
| M47 | `sign` | `revoke` does not set the flag | caught by `key_windows_and_revocation` |
| M48 | `sign` | `Debug` prints the key bytes | caught by `debug_output_never_contains_key_bytes` |
| M49 | `webhook` | tag length check `<` instead of `!=` | caught by `malformed_headers` |
| M50 | `webhook` | uppercase hex accepted | caught by `malformed_headers` |
| M51 | `webhook` | separator in the signed message is `:` | caught by `header_known_answer` |
| M52 | `webhook` | header prefix `T=` | caught by `accepts_valid` |
| M53 | `webhook` | tag separator `,v0=` | caught by `accepts_valid` |
| M54 | `webhook` | empty timestamp check removed | **equivalent**: an empty timestamp is refused by `parse` right afterwards with the same error |
| M55 | `webhook` | leading-zero rule starts at three digits | caught by `short_leading_zero_timestamps_are_malformed` |
| M56 | `webhook` | non-digit timestamps accepted | caught by `malformed_headers` |
| M57 | `webhook` | tolerance `>=` instead of `>` | caught by `tolerance_boundaries` |
| M58 | `webhook` | freshness only checked in one direction | caught by `tolerance_boundaries` |
| M59 | `webhook` | freshness not checked | caught by `stale_is_checked_even_with_valid_mac` |
| M60 | `webhook` | result of the last key only (`=` instead of `\|=`) | caught by `rotation_accepts_either_key` |
| M61 | `webhook` | all keys must match (`&=`) | caught by `accepts_valid` |
| M62 | `webhook` | accumulator starts as `true` | caught by `rejects_wrong_body_key_and_tag` |
| M63 | `webhook` | MAC over the body only | caught by `accepts_valid` |
| M64 | `webhook` | hex without zero padding | caught by `header_known_answer` |
| M65 | `journal` | domain label not in the checksum | caught by `checksum_known_answers` |
| M66 | `journal` | length not in the checksum | caught by `checksum_known_answers` |
| M67 | `journal` | payload not in the checksum | caught by `checksum_known_answers` |
| M68 | `journal` | last checksum byte is zero | caught by `checksum_known_answers` |
| M69 | `journal` | first checksum byte is zero | caught by `checksum_known_answers` |
| M70 | `journal` | `append` accepts oversize records | caught by `append_too_long_panics` |
| M71 | `journal` | record limit `>=` instead of `>` | caught by `max_size_record_ok` |
| M72 | `journal` | record limit not enforced by `recover` | caught by `oversize_record_with_a_valid_checksum_is_still_rejected` |
| M73 | `journal` | end-of-input test `>=` instead of `>` | caught by `from_bytes_drops_garbage_tail_so_appends_stay_valid` |
| M74 | `journal` | loop condition `<` instead of `<=` for the length prefix | **equivalent**: a record needs at least 8 bytes, so when exactly 4 bytes remain the following length check fails the same way |
| M75 | `journal` | `from_bytes` keeps the invalid tail | caught by `from_bytes_drops_garbage_tail_so_appends_stay_valid` |
| M76 | `journal` | checksum never compared | caught by `every_single_bit_flip_is_detected` |
| M77 | `breaker` | open interval inclusive | caught by `failed_probe_reopens_with_full_cooldown` |
| M78 | `breaker` | opens one failure late | caught by `half_open_after_cooldown_with_one_probe` |
| M79 | `breaker` | success does not reset the failure count | caught by `success_resets_count` |
| M80 | `breaker` | success does not close the breaker | caught by `half_open_after_cooldown_with_one_probe` |
| M81 | `breaker` | failed probe does not restart the cool-down | caught by `failed_probe_reopens_with_full_cooldown` |
| M82 | `breaker` | a trial that never reports blocks for ever | caught by `lost_probe_is_forgotten_after_another_cooldown` |
| M83 | `breaker` | trial expiry inclusive | caught by `lost_probe_is_forgotten_after_another_cooldown` |
| M84 | `breaker` | trial start time not recorded | caught by `half_open_after_cooldown_with_one_probe` |
| M85 | `breaker` | open state allows calls | caught by `half_open_after_cooldown_with_one_probe` |
| M86 | `breaker` | zero threshold accepted | caught by `zero_threshold_rejected` |
| M87 | `breaker` | failed probe does not clear the trial flag | **equivalent**: since trial permits expire after `open_ms`, the flag cleared by this line would be ignored anyway when the next trial is allowed |
| M88 | `breaker` | success does not clear the trial flag | **equivalent**: after a success the breaker is closed, and the stale trial time is older than any cool-down by the next time it matters |
| M89 | `queue` | `Enqueued` tag byte changed | caught by `corrupted_tail_is_ignored_and_queue_continues` |
| M90 | `queue` | `Enqueued` encodes the id where the key belongs | caught by `event_encoding_round_trip_and_strict` |
| M91 | `queue` | `ready_at` not encoded | caught by `corrupted_tail_is_ignored_and_queue_continues` |
| M92 | `queue` | payload limit `>=` instead of `>` in `decode` | caught by `decode_accepts_a_payload_of_exactly_the_record_limit` |
| M93 | `queue` | trailing bytes accepted in `decode` | caught by `event_encoding_round_trip_and_strict` |
| M94 | `queue` | `Leased` expects 15 bytes | detected as a **hang** (killed after 200 s) |
| M95 | `queue` | idempotency keys not indexed | caught by `duplicate_keys_are_ignored` |
| M96 | `queue` | next id not updated by `apply` | caught by `corrupted_tail_is_ignored_and_queue_continues` |
| M97 | `queue` | attempts count by two | caught by `dead_letter_after_max_attempts` |
| M98 | `queue` | duplicate lookup uses the wrong key | caught by `duplicate_keys_are_ignored` |
| M99 | `queue` | `reap` needs more than `max_attempts` | caught by `has_ready_matches_lease` |
| M100 | `queue` | `reap` lease expiry exclusive | caught by `has_ready_matches_lease` |
| M101 | `queue` | pending jobs ready only after `ready_at` | caught by `complete_and_fail_errors` |
| M102 | `queue` | expired leases retaken one millisecond late | caught by `expired_lease_is_retaken_and_old_holder_is_stale` |
| M103 | `queue` | dead jobs are leased again | caught by `dead_letter_after_max_attempts` |
| M104 | `queue` | lease deadline equals now | caught by `expired_lease_is_retaken_and_old_holder_is_stale` |
| M105 | `queue` | `has_ready` ignores the attempt limit | caught by `has_ready_matches_lease` |
| M106 | `queue` | `has_ready` pending test exclusive | caught by `has_ready_matches_lease` |
| M107 | `queue` | `complete` and `fail` do not require a lease | caught by `complete_and_fail_errors` |
| M108 | `queue` | lease token not compared | caught by `expired_lease_is_retaken_and_old_holder_is_stale` |
| M109 | `queue` | idempotent `complete` ignores the attempt | caught by `complete_and_fail_errors` |
| M110 | `queue` | last attempt `>` instead of `>=` | caught by `dead_letter_after_max_attempts` |
| M111 | `queue` | retry time ignores the backoff | caught by `failure_backs_off_then_retries` |
| M112 | `queue` | leased jobs counted as pending | caught by `reap_only_touches_exhausted_expired_leases` |
| M113 | `queue` | undecodable record skipped instead of ending replay | caught by `undecodable_record_stops_replay` |
| M114 | `queue` | `Completed` marks the job dead | caught by `complete_and_fail_errors` |
| M115 | `queue` | recovery discards the journal | caught by `corrupted_tail_is_ignored_and_queue_continues` |
| M116 | `queue` | events not appended to the journal | caught by `corrupted_tail_is_ignored_and_queue_continues` |
| M117 | `queue` | `apply` increments the next id instead of taking the maximum | **equivalent**: ids are assigned in sequence starting at 1, so `next_id + 1` and `max(next_id, id + 1)` agree on every journal this crate can write |
| M118 | `queue` | journal append after `apply` | **equivalent**: both orders complete inside one call on one thread; the order matters only with a real disk (§5.4) |
| M119 | `sim` | duplicate enqueue not asserted | **equivalent**: the line only asserts a property of the library that `duplicate_keys_are_ignored` tests directly; removing it changes no behavior |
| M120 | `sim` | chain never deduplicates | caught by `chaos_with_dedupe_is_exactly_once_effect` |
| M121 | `sim` | lost acknowledgements never injected | caught by `timeouts_alone_double_apply_on_a_blind_chain_but_not_on_a_deduplicating_one` |
| M122 | `sim` | refusals never injected | caught by `hopeless_chain_ends_in_dead_letters` |
| M123 | `sim` | crash recovery keeps the live queue instead of replaying | **equivalent in outcome**: recovery equals the live state (tested directly in `recovery_equals_live_state`), so replacing it in the simulation changes no report field |
| M124 | `sim` | signing service survives the crash | caught by `a_relayer_that_restarts_after_every_submission_burns_a_whole_nonce_block_each_time` |
| M125 | `sim` | nonce reservations never stored | caught by `a_relayer_that_restarts_after_every_submission_burns_a_whole_nonce_block_each_time` |
| M126 | `sim` | backoff always zero | caught by `backoff_waits_are_scheduled_only_after_failures` |
| M127 | `sim` | breaker not told about successes | caught by `a_breaker_that_forgets_failures_on_success_stays_closed_at_a_low_fault_rate` |
| M128 | `sim` | breaker not told about failures | caught by `chaos_actually_exercises_the_faults` |
| M129 | `sim` | time limit multiplied by 1,000 | caught by `the_time_limit_stops_a_run_that_cannot_finish` |
| M130 | `sim` | nonce high-water mark not reported | caught by `a_relayer_that_restarts_after_every_submission_burns_a_whole_nonce_block_each_time` |
| M131 | `sim` | exhausted leases not reaped | caught by `jobs_whose_every_attempt_ends_in_a_crash_become_dead_letters` |

What the table does not show: it tests *these* edits, not every possible bug; the simulator is test infrastructure and is mutated at only a few spots; the equivalent mutants could hide a real gap if my reasoning about them is wrong; and passing every mutant says nothing about security, about real disks or about behavior against a real chain or key service.

**Design notes.**

- Every dependency on the world is an argument: time (`now_ms`), randomness (`Rng`), the signer (`Signer`), the policy (`Policy`), the chain (`SimChain`). The logic is deterministic and fast to test.
- State changes are events; the journal holds events; live and recovered state go through one function. That is the whole recovery story, and it is the reason `recovery_equals_live_state` can compare journal bytes.
- Every parser is strict and bounded (envelope, webhook header, journal record, queue event); every limit is pinned by a test that uses the literal number.
- Hashes, MAC tags, journal checksums and the generator's outputs are checked against independent Python computations.
- Not handled: real keys and custody, real storage and fsync, multiple workers, network protocols, chain-specific transaction formats and everything in §10.

</details>

---

## 15. Key takeaways

- A signature protects only what is inside the signed bytes: include a domain, the chain id, the key id, a nonce, an expiry and the payload length (✓ every field is covered; a signature moved to another chain fails).
- A signing service is a gate, not a function: a policy refuses what should never be signed, refused requests burn no nonce, and the key never appears in a log or a `Debug` (✓).
- A nonce must survive a restart: reserve a block, store the reservation before using it, and accept gaps (✓ 500 signings with random restarts, no reuse).
- Record a replay nonce only after the signature verifies, verify with the strict check, bound lifetimes and rotate keys with an overlap that is at least as long as anything in flight (✓).
- Authenticate webhooks with an HMAC over timestamp and body, parse the header strictly, check freshness both ways, compare in constant time, accept several secrets during rotation, and make the handler idempotent (✓ RFC 4231 vectors, boundary tests at ±1,000 ms).
- A journal needs a length and a checksum per record, and recovery keeps the longest valid prefix (✓ cut at every offset and flipped at every bit).
- A queue is its event log: one `apply` for live and recovered state, idempotency keys, leases with fencing tokens, bounded retries and a dead-letter state (✓ recovery equals live state byte for byte).
- Exactly-once is at-least-once plus an idempotent effect; a timeout after the work was done is the case that breaks naive retries (✓ 200 of 200 seeds double-applied on a blind chain, 0 of 200 on a deduplicating one).
- A permit acquired must be released on every path; make permits expire (✓ the breaker that never closed, found by a property test).
- Mutation testing here pointed at real gaps: unpinned constants, an unpinned checksum, an unpinned strict-verification choice and several unpinned simulator behaviors. Twelve real gaps and one strictness gap were found and closed in the first pass and seven more in the second (see the table in §14); 13 mutants remain, each with a stated reason for being equivalent.

**Connections.** [Lesson 51](51_rpc_nodes_indexers.md) supplied the retry client, the idea of an uncertain write and stable ids; this lesson builds the other side of that boundary: what a service must persist and verify so that retries are safe. [Lesson 50](50_networking_determinism.md) supplied limits, canonical encodings and the sans-IO, seeded-simulation style. [Lesson 49](49_merkle_transactions_consensus.md) explains nonces, transaction ids and finality, which determine what "the same transaction" means on a chain. [Lesson 48](48_crypto_primitives.md) provides the hash, the MAC, the signature and the constant-time comparison used throughout. [Lesson 46](46_databases_config_auth_jobs.md) covers the database transactions and job queues that a production outbox would use, and the handling of secrets. [Lesson 45](45_http_apis.md) covers idempotency keys at the HTTP level. [Lesson 40](40_property_testing_fuzzing.md) explains the testing style of the 200-seed properties. Lesson 53 begins the Solana module.

---

## 16. References

- RFC 4231, Identifiers and Test Vectors for HMAC-SHA-224, HMAC-SHA-256, HMAC-SHA-384, and HMAC-SHA-512: <https://www.rfc-editor.org/rfc/rfc4231>
- RFC 8032, Edwards-Curve Digital Signature Algorithm (EdDSA): <https://www.rfc-editor.org/rfc/rfc8032>
- The documentation of `ed25519-dalek` (strict verification), `hmac`, `sha2` and `subtle`: <https://github.com/dalek-cryptography/curve25519-dalek>, <https://github.com/RustCrypto/MACs>, <https://github.com/RustCrypto/hashes>
- The webhook-signing documentation of whichever provider you integrate with (not consulted; formats and tolerances differ).
- Martin Kleppmann, *Designing Data-Intensive Applications* (2017, from memory), for logs, idempotence and delivery guarantees; Michael Nygard, *Release It!* (from memory), for circuit breakers and the other stability patterns.
- Marc Brooker, "Exponential Backoff And Jitter", AWS Architecture Blog (2015, from memory).
- The documentation of your key-management service or hardware module, and of your database's write-ahead logging and durability settings (not consulted).

The URLs and titles above are the standard locations; I did not open them during this session (the sandbox cannot reach documentation and standards sites), so if a link has moved, search for the title. **Unchecked or from memory:** the content of RFC 4231 and RFC 8032 (the two HMAC test vectors were recomputed with Python's `hmac` module and agree with the values I remember; I did not re-read the RFC); the description of strict versus ordinary Ed25519 verification in `ed25519-dalek` (the behavior I rely on, the identity key being refused by `verify_strict`, was tested, but the full list of what strict mode rejects was not enumerated; I did not check whether the ordinary check in this version accepts that test input, except through the mutation in §14); that some payment providers use a `t=…,v1=…` webhook scheme; that real chains use sequential account nonces; the descriptions of cloud key services, hardware modules, threshold signing and multi-signature accounts; the durability behavior of file systems, drives and `fsync`; `FOR UPDATE SKIP LOCKED` and managed queues; the circuit-breaker pattern as described in the literature; the claims about `secrecy`, `zeroize` and `subtle` internals; and the AWS jitter article. **Environment caveats:** timings are single-machine micro-measurements (three runs of one example) on a shared virtual machine with two cores; nothing here talks to a real network, disk-durability layer or key service, so no latency, throughput or durability figure applies to a real deployment. Measurements and outcomes marked ✓ were observed with `rustc 1.97.0`, `sha2 0.11.0`, `ed25519-dalek 3.0.0` and `hmac 0.13.0`. The code blocks in exercises 1 to 8 and in the debugging challenge's two versions were compiled and run in a scratch crate (✓ 14 tests, with `cargo clippy --all-targets -- -D warnings` clean); the mutation results are those of the table above (a test process that aborted, failed an assertion or did not finish within 200 seconds counted as detecting the mutant). **No part of this lesson makes any system secure.**
