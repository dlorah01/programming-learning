---
title: "Lesson 58 — Cross-chain infrastructure, relayers and indexers"
---

# Lesson 58 — Cross-chain infrastructure, relayers and indexers

**Module 18 · Other blockchain ecosystems (third and last lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `ed25519-dalek 3.0.0`, `sha2 0.11.1`, and, for tests only, `hex 0.4.3` (26 packages in `Cargo.lock`, the crate itself included)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** Lessons 53, 56 and 57 each built a client for one chain. This one is about what sits **between** chains and between a chain and the applications that read it. A **bridge** (or cross-chain messaging protocol) lets an action on a *source* chain cause an effect on a *destination* chain; a **relayer** is the off-chain program that carries the message and its proof from one to the other; an **indexer** turns a chain's blocks into a queryable database and has to survive reorganizations. The lesson builds each as a small model with tests: a **message** with a canonical encoding and a domain-separated id, a **Merkle tree** with proofs that are bound to an index and a size, a **source chain** that escrows funds and can be reorganized, a **validator quorum** that signs finalized headers (ed25519), a **destination inbox** that executes each message at most once, an **untrusted relayer** that survives a network that loses, duplicates and half-applies calls, and a **reorganization-aware indexer**. What I did **not** do: use or read the code of any real bridge, messaging protocol, light client, relayer framework or indexing service, connect to any network, or build any smart contract. Every statement about real systems is labeled "from memory", and the lesson is about the **shape of the problem**: what must be verified, what must be idempotent, and what must be assumed.

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation sites were not. The Merkle roots and proofs, the message encoding and id, the header hash and an ed25519 public key and signature were computed by a **separate Python program** (`p58/py/ref.py`: `hashlib` and the `cryptography` package), not by the code under test.

Builds on [Lesson 49](49_merkle_transactions_consensus.md) (Merkle trees, transactions, finality), [Lesson 48](48_crypto_primitives.md) (hashes, signatures), [Lesson 50](50_networking_determinism.md) (determinism), [Lesson 51](51_rpc_nodes_indexers.md) (nodes and indexers), [Lesson 52](52_web3_reliability.md) (finality, retries, idempotency), [Lesson 53](53_solana_model.md), [Lesson 56](56_substrate_polkadot.md) and [Lesson 57](57_evm_rust.md) (three chains whose messages a bridge might carry), [Lesson 39](39_testing.md), [Lesson 40](40_property_testing_fuzzing.md), [Lesson 41](41_benchmarking_profiling.md) and [Lesson 26](26_error_design.md). The next lesson, Lesson 59, appears here only as plain text.

---

## 1. Why this lesson exists

Cross-chain systems concentrate risk: they hold the funds that back assets on another chain, and every check they skip is a check an attacker can walk through. The failures are old and boring: a proof that is not bound to what it proves, a message that executes twice, a "final" block that is not, a relayer that loses a message when the network hiccups. Each claim below has an experiment.

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| A message needs one canonical encoding and an id that covers its routing | the 74-byte header plus payload matches a Python build; every one of eight field changes changes the id; decoding refuses short, long and trailing input (§3) |
| A Merkle proof must be bound to an index and to a size | for every leaf of every tree of 1 to 33 leaves the proof verifies, and moving the index, growing the size, adding or removing a sibling or changing the leaf each break it; **a first version did not bind the size** (§4.2) |
| Promoting an odd node is safer than duplicating it | `[a,b,c]` and `[a,b,c,c]` have different roots; with promotion there is no equal-root pair (§4.1) |
| An index equal to the size must be refused explicitly | in a one-leaf tree the proof of "leaf 1" needs no siblings and would pass without the bound; a mutant that loosened it survived until a test was added (§4.2, §14) |
| A reorganization releases what the dropped blocks escrowed | after dropping two blocks, escrow and balances return to their earlier values and supply is conserved (§5) |
| A quorum threshold must exceed two thirds | thresholds 2 of 4 and 5 of 4 are refused; duplicates, unknown signers, bad signatures, signatures over another chain id or height all fail (§6) |
| Weak public keys can forge signatures under the plain verification equation | the identity point as key and as `R` "verifies" under non-strict checking; `verify_strict` refuses it (§6.2) |
| Nothing is trusted before an attestation, and a root cannot change | a delivery for an unattested height fails; a second attestation of a different root for a known height is an equivocation (§7) |
| Execution must be idempotent | replaying a delivery changes nothing; a cap and a pause stop minting; sums are checked, not wrapped (§7) |
| A relayer is untrusted and must survive a hostile network | 12 seeds of a link that loses calls, applies them and then reports failure, and duplicates them: every final message arrives exactly once and totals match (§8.3) |
| Shallow confirmations lose money | a message delivered after zero confirmations is minted; the block is then reorganized away; the destination holds 500 against nothing. With three confirmations it is not (§8.4) |
| An indexer must follow reorganizations | after 10 seeds of 80 rounds of random sends and reorganizations its hashes and event count equal the chain's (§9) |
| A feed of blocks cannot see a shorter chain | the indexer keeps a stale tip until it syncs against the chain (§9.3) |

Total in the practical challenge: one library crate, 8 modules, **53 tests**, a benchmark example and **95 mutants** run against the test suite.

---

## 2. The landscape (from memory)

Nothing in this section was run; it is the map, and §3 onward is the territory.

- **Why chains cannot just read each other.** A contract or program on one chain cannot make a network call. To act on another chain's state it must be handed a *claim* and a way to *verify* it. The ways differ in what they ask you to trust (my classification, from memory):
  - an **external validator set** (a multisig or a committee of signers) that attests to events: simple and cheap, and trust is in the committee's keys (this lesson's model);
  - a **light client** of the source chain running on the destination (the Inter-Blockchain Communication protocol, IBC, in the Cosmos ecosystem is the example I remember): the destination verifies the source's consensus itself, so trust is in the source chain and in correct client code;
  - **optimistic** schemes: a message is accepted after a challenge window unless someone proves it fraudulent, so trust is in at least one honest watcher being online;
  - **validity proofs**: a zero-knowledge proof that the source state transition is correct, so trust is in the proof system and the circuit.
- **Asset movement patterns:** *lock-and-mint* (lock on the source, mint a wrapped token on the destination, as modelled here), *burn-and-mint* (the token exists natively on both and total supply is conserved across them) and *liquidity pools* (a relayer or market maker pays out from a pool on the destination and is repaid). Each puts the risk somewhere different: the vault, the minting permission, or the pool.
- **Messaging protocols** I remember by name only: IBC, Wormhole, LayerZero, Axelar, Hyperlane, Chainlink CCIP. Their designs and trust models differ and change; check their current documentation.
- **Relayers** are usually permissionless carriers: they cost gas, may be paid a fee, and, in well-designed systems, **cannot forge** anything, only fail to deliver. **Oracles** and **keepers** are neighbours with the same shape (off-chain code that watches and acts).
- **Indexers** read blocks and logs and write them to a database for an API. Examples I remember: The Graph's subgraphs, custom extract-transform-load pipelines, and hosted APIs. Lesson 51 covered the basic loop; this lesson adds reorganizations and idempotency.
- **Incidents** I remember in outline (from memory; I did not check details or amounts): a 2022 bridge compromise in which an attacker obtained enough validator keys to meet the signing threshold; a 2022 case in which a signature-verification path could be bypassed so that an unbacked mint was accepted; a 2022 case in which a configuration or initialization mistake made nearly any message look valid. The pattern is always one of three things: **the keys**, **the verification code**, **the configuration**.

---

## 3. The message and its identity ✓

```rust
impl Message {
    /// Fixed-width big-endian fields, then a two-byte payload length and the payload.
    pub fn encode(&self) -> Vec<u8> {
        let mut o = Vec::with_capacity(HEADER_LEN + self.payload.len());
        o.extend(self.src.to_be_bytes());
        o.extend(self.dst.to_be_bytes());
        o.extend(self.nonce.to_be_bytes());
        o.extend(self.sender);
        o.extend(self.recipient);
        o.extend(self.amount.to_be_bytes());
        o.extend((self.payload.len() as u16).to_be_bytes());
        o.extend(&self.payload);
        o
    }

    /// Strict: the length field must match, the payload must be short enough, nothing may follow.
    pub fn decode(b: &[u8]) -> Result<Message, MsgError> {
        if b.len() < HEADER_LEN {
            return Err(MsgError::Short);
        }
        let plen = usize::from(u16::from_be_bytes([b[72], b[73]]));
        if plen > MAX_PAYLOAD {
            return Err(MsgError::PayloadTooLong);
        }
        if b.len() < HEADER_LEN + plen {
            return Err(MsgError::Short);
        }
        if b.len() > HEADER_LEN + plen {
            return Err(MsgError::Trailing);
        }
        Ok(Message {
            src: u32::from_be_bytes(b[0..4].try_into().expect("4")),
            dst: u32::from_be_bytes(b[4..8].try_into().expect("4")),
            nonce: u64::from_be_bytes(b[8..16].try_into().expect("8")),
            sender: b[16..36].try_into().expect("20"),
            recipient: b[36..56].try_into().expect("20"),
            amount: u128::from_be_bytes(b[56..72].try_into().expect("16")),
            payload: b[HEADER_LEN..].to_vec(),
        })
    }

    /// The message id: a domain-separated hash of the whole encoding, source and destination included.
    pub fn id(&self) -> [u8; 32] {
        sha256(b"msg-v1", &[&self.encode()])
    }
}
```

- **Fixed-width, big-endian, length-prefixed payload.** 4 + 4 + 8 + 20 + 20 + 16 + 2 bytes, then the payload (at most 256 bytes). There is exactly one encoding of a message, and decoding is **strict**: too short, too long, a payload length over the limit and trailing bytes are all refused (✓ `decoding_is_strict`).
- **The id covers routing.** Source chain, destination chain and nonce are inside the encoding, so the id of "send 5 to X" on one chain pair is not the id of the same words on another (✓ `every_field_changes_the_id`: eight single-field changes, eight different ids). The hash is **domain-separated** with a tag (`msg-v1`) and the tag's length, so an id cannot be confused with any other hash in the system (a Merkle leaf, a header hash).
- ✓ `message_encoding_and_id_match_python`: the encoding of a sample message and its id match a Python program that builds the bytes with `struct` and hashes with `hashlib`. ✓ 500 random messages round-trip and 4,000 random byte strings never panic the decoder.

---

## 4. Merkle commitments and proofs ✓

A source block commits to its messages with a **Merkle root**; the destination holds the root (attested, §6) and a relayer supplies one message plus a **proof**. The destination checks the proof without seeing the other messages (Lesson 49).

### 4.1 The tree

```rust
/// `SHA-256(0x00 || data)`.
pub fn leaf_hash(data: &[u8]) -> Hash {
    let mut h = Sha256::new();
    h.update([0x00]);
    h.update(data);
    h.finalize().into()
}

/// `SHA-256(0x01 || left || right)`.
pub fn node_hash(l: &Hash, r: &Hash) -> Hash {
    let mut h = Sha256::new();
    h.update([0x01]);
    h.update(l);
    h.update(r);
    h.finalize().into()
}

/// `SHA-256(0x02 || size as 8 bytes || top)`: binds the leaf count to the tree's top hash.
pub fn commit(size: u64, top: &Hash) -> Hash {
    let mut h = Sha256::new();
    h.update([0x02]);
    h.update(size.to_be_bytes());
    h.update(top);
    h.finalize().into()
}

/// The root of the tree over `leaves`.
pub fn root(leaves: &[Vec<u8>]) -> Hash {
    if leaves.is_empty() {
        return empty_root();
    }
    let mut level: Vec<Hash> = leaves.iter().map(|l| leaf_hash(l)).collect();
    while level.len() > 1 {
        level = next_level(&level);
    }
    commit(leaves.len() as u64, &level[0])
}

/// The proof for `leaves[index]`.
pub fn prove(leaves: &[Vec<u8>], index: usize) -> Option<Proof> {
    if index >= leaves.len() {
        return None;
    }
    let mut level: Vec<Hash> = leaves.iter().map(|l| leaf_hash(l)).collect();
    let mut idx = index;
    let mut siblings = Vec::new();
    while level.len() > 1 {
        if idx % 2 == 1 {
            siblings.push(level[idx - 1]);
        } else if idx + 1 < level.len() {
            siblings.push(level[idx + 1]);
        }
        idx /= 2;
        level = next_level(&level);
    }
    Some(Proof {
        index: index as u64,
        size: leaves.len() as u64,
        siblings,
    })
}

/// Checks that `leaf` is at `proof.index` of a tree of `proof.size` leaves with root `root`.
/// The shape of the path is derived from the index and size, so a proof cannot claim a different
/// depth, and every sibling must be used exactly once.
pub fn verify(root: &Hash, leaf: &[u8], proof: &Proof) -> bool {
    if proof.size == 0 || proof.index >= proof.size {
        return false;
    }
    let mut h = leaf_hash(leaf);
    let (mut idx, mut n, mut used) = (proof.index, proof.size, 0usize);
    while n > 1 {
        if idx % 2 == 1 {
            let Some(s) = proof.siblings.get(used) else {
                return false;
            };
            used += 1;
            h = node_hash(s, &h);
        } else if idx + 1 < n {
            let Some(s) = proof.siblings.get(used) else {
                return false;
            };
            used += 1;
            h = node_hash(&h, s);
        }
        idx /= 2;
        n = n.div_ceil(2);
    }
    used == proof.siblings.len() && commit(proof.size, &h) == *root
}
```

- **Domain separation.** A leaf is `SHA-256(0x00 ‖ data)` and an inner node `SHA-256(0x01 ‖ left ‖ right)`. Without the prefix, an inner node (64 bytes of two hashes) is indistinguishable from a leaf whose data is those 64 bytes, and an attacker can present a node as a leaf of a shallower tree (the second-preimage attack on Merkle trees). ✓ `a_leaf_cannot_pose_as_a_node_or_the_reverse`.
- **Promotion, not duplication.** If a level has an odd number of nodes, the last one is carried up unchanged. The common alternative, hashing it with itself, makes `[a,b,c]` and `[a,b,c,c]` share a root (Bitcoin had this property, from memory); with promotion they do not (✓ `a_duplicated_last_leaf_changes_the_root`, with the root of the second tree pinned from Python).
- ✓ Known answers from Python for the trees over `a`, `a,b`, `a,b,c` and `a,b,c,d,e`, the empty tree, and the sibling lists of two proofs (`roots_match_python`, `proofs_match_python`). The proof of the last leaf of a five-leaf tree has **one** sibling, not three, because that leaf is promoted twice.

### 4.2 Four things a proof must be bound to ✓

1. **The leaf** (obviously): the verifier recomputes the leaf hash from the bytes it was given.
2. **The index.** The path's *shape* (which side each sibling goes on, and where a node is promoted) is derived from `index` and `size`, not read from the proof. A proof cannot claim a different position, and every sibling must be used exactly once (✓ moving the index, adding or removing a sibling, or flipping a bit in one all fail, for every leaf of every size from 1 to 33: `every_leaf_of_every_size_proves_and_nothing_else_does`).
3. **The size.** My first version hashed up the tree and compared with the root, and a test that grew the claimed size **passed when it should have failed**: the proof of the *first* leaf of `[a,b,c]` is `[b, c]`, which is also a valid-looking path in a four-leaf tree, because the verifier never learned that the tree had three leaves. The fix is to make the root commit to the count: `root = SHA-256(0x02 ‖ size ‖ top)`, and the verifier recomputes it with `proof.size`. ✓ `the_root_commits_to_the_leaf_count` tries sizes 1, 2, 4, 5 and 8 against a three-leaf tree. (This matters when an *index* or *size* is used for anything else: a nonce, a sequence number, "is this the last message".)
4. **A bound on the index.** `index >= size` is refused before anything else. Without it, in a **one-leaf tree** the "proof" `{index: 1, size: 1, siblings: []}` has the same hash path as the real proof of leaf 0 and verifies. ✓ `an_index_equal_to_the_size_is_never_valid`. I did not see this one by reading the code; a mutant that loosened the check (`>=` to `>`) survived until I wrote the test (§17).

---

## 5. The source chain ✓

```rust
/// Locks `amount` of the sender's balance in escrow and queues a message for the next block.
pub fn send(
    &mut self,
    sender: Addr,
    recipient: Addr,
    dst: u32,
    amount: u128,
    payload: Vec<u8>,
) -> Result<Message, SendError> {
    if amount == 0 {
        return Err(SendError::ZeroAmount);
    }
    if payload.len() > MAX_PAYLOAD {
        return Err(SendError::PayloadTooLong);
    }
    let bal = self.balances.entry(sender).or_default();
    if *bal < amount {
        return Err(SendError::Insufficient);
    }
    *bal -= amount;
    self.escrow += amount;
    let m = Message {
        src: self.id,
        dst,
        nonce: self.next_nonce,
        sender,
        recipient,
        amount,
        payload,
    };
    self.next_nonce += 1;
    self.pending.push(m.clone());
    Ok(m)
}
```

```rust
/// Seals the pending messages into a new block.
pub fn seal(&mut self) -> &Block {
    let msgs = std::mem::take(&mut self.pending);
    let leaves: Vec<Vec<u8>> = msgs.iter().map(Message::encode).collect();
    let root = merkle::root(&leaves);
    let parent = self.blocks.last().expect("genesis").hash;
    let height = self.tip() + 1;
    let hash = header_hash(self.id, height, &parent, &root);
    self.blocks.push(Block {
        height,
        parent,
        root,
        hash,
        msgs,
    });
    self.blocks.last().expect("just pushed")
}
```

```rust
/// Drops the last `depth` blocks (never the genesis block) and returns the escrowed funds of
/// their messages to the senders: those transactions are no longer part of the chain.
pub fn reorg(&mut self, depth: u64) -> u64 {
    let mut dropped = 0;
    while dropped < depth && self.blocks.len() > 1 {
        let b = self.blocks.pop().expect("len > 1");
        for m in b.msgs {
            self.escrow -= m.amount;
            *self.balances.entry(m.sender).or_default() += m.amount;
        }
        dropped += 1;
    }
    dropped
}
```

- **Escrow.** Sending debits the sender and credits the **escrow** in the same step; the message is only a claim on the escrow. `total_supply` (balances plus escrow) never changes (✓ checked after every operation in the tests). A zero amount, an amount over the balance and a payload over 256 bytes are refused with distinct errors (`sends_are_refused_for_the_right_reason`).
- **The block commits to its messages.** The header hash is a domain-separated hash of the chain id, height, parent and message root (`hdr-v1`; ✓ one value pinned from Python), so a header names the chain it belongs to and the exact messages it contains.
- **A reorganization drops blocks and returns their escrow.** The transactions are no longer part of the chain, so the sender has their funds back (✓ `a_reorg_refunds_the_dropped_messages`: two blocks dropped, escrow 150 → 100, balances restored; the genesis block is never dropped). This is the single most important behavior to keep in mind for §8.4: **anything that acted on a message in a dropped block acted on something that no longer happened**.

---

## 6. Attestations: a quorum signs a finalized header ✓

```rust
/// The smallest threshold that is more than two thirds of `n`.
pub fn threshold_for(n: usize) -> usize {
    2 * n / 3 + 1
}

/// Signs the block at `height` of `chain` if it has at least `depth` confirmations.
pub fn attest(&self, chain: &SourceChain, height: u64) -> Option<(u32, [u8; 64])> {
    if height.checked_add(self.depth)? > chain.tip() {
        return None;
    }
    let b = chain.block(height)?;
    let sig = self
        .key
        .sign(&attest_bytes(chain.id, b.height, &b.hash, &b.root));
    Some((self.index, sig.to_bytes()))
}

/// Checks distinct, known signers with valid signatures, at least `threshold` of them.
pub fn verify(&self, chain: u32, a: &Attestation) -> Result<(), QuorumError> {
    let msg = attest_bytes(chain, a.height, &a.hash, &a.root);
    let mut seen = BTreeSet::new();
    for (i, sig) in &a.sigs {
        let key = self
            .keys
            .get(usize::try_from(*i).map_err(|_| QuorumError::UnknownSigner)?)
            .ok_or(QuorumError::UnknownSigner)?;
        if !seen.insert(*i) {
            return Err(QuorumError::Duplicate);
        }
        key.verify_strict(&msg, &Signature::from_bytes(sig))
            .map_err(|_| QuorumError::BadSignature)?;
    }
    if seen.len() < self.threshold {
        return Err(QuorumError::NotEnough);
    }
    Ok(())
}
```

### 6.1 What is signed and by whom

A validator signs `att-v1 ‖ chain id ‖ height ‖ header hash ‖ message root`. The chain id is inside, so an attestation for one source chain is not valid for another; height, hash and root are all inside, so nothing about the block can be swapped after signing (✓ the Python-computed public key and signature verify, and changing the chain id, height, root or hash makes them fail: `the_ed25519_vector_matches_python`). **A validator signs only blocks it considers final**: `attest` returns `None` unless the block has at least `depth` confirmations on the validator's own view of the chain (✓ `a_validator_signs_only_confirmed_blocks_of_its_own_chain_view`, including the height `u64::MAX` that would overflow a naive `height + depth`).

### 6.2 Verifying a quorum ✓

- The threshold must be **more than two thirds** of the set (`2n/3 + 1`), so that two conflicting quorums must share a signer, and if a third or fewer of the validators are faulty, two quorums cannot both be made of honest signatures on different blocks (the usual Byzantine argument, from memory). ✓ `thresholds_must_exceed_two_thirds`: for four validators 2 is refused, 3 and 4 accepted, 5 refused; the thresholds for eight set sizes between 1 and 9 are pinned.
- Signers are **distinct** (the same signature three times is one signer: `Duplicate`), **known** (an index outside the set fails) and **valid**; a signature by validator 1 presented as validator 0 fails (✓ `quorum_checks_each_rule`).
- **Strict verification.** `verify_strict` rejects weak (small-order) public keys (and, from my memory of its documentation, non-canonical signature encodings; only the weak-key case is tested here). Under the plain ed25519 equation the *identity point* as a public key, with the identity as `R` and a zero `S`, is a signature on **every** message. A validator set with such a key lets anyone forge that member. ✓ `a_small_order_key_cannot_forge_a_signature`: the same bytes are refused here. I wrote this test **before** the first mutation run, because I knew the mutant that replaces `verify_strict` with `verify` would survive without it (§17).

---

## 7. The destination inbox ✓

```rust
/// Accepts the (hash, root) of a block if a quorum signed it. Repeating the same attestation
/// is fine; a different root for a known height is an equivocation and is refused.
pub fn submit_attestation(&mut self, a: &Attestation) -> Result<(), BridgeError> {
    self.set
        .verify(self.source, a)
        .map_err(BridgeError::Quorum)?;
    match self.roots.get(&a.height) {
        Some((h, r)) if *h == a.hash && *r == a.root => Ok(()),
        Some(_) => Err(BridgeError::Equivocation),
        None => {
            self.roots.insert(a.height, (a.hash, a.root));
            Ok(())
        }
    }
}
```

```rust
/// Executes one message: decode, route, find the root for `height`, check the proof, check the
/// id was not used, check the cap, then mint. The id is recorded before anything else changes.
pub fn deliver(
    &mut self,
    encoded: &[u8],
    height: u64,
    proof: &Proof,
) -> Result<Message, BridgeError> {
    if self.paused {
        return Err(BridgeError::Paused);
    }
    let m = Message::decode(encoded).map_err(BridgeError::Malformed)?;
    if m.src != self.source {
        return Err(BridgeError::WrongSource);
    }
    if m.dst != self.chain {
        return Err(BridgeError::WrongDestination);
    }
    let (_, root) = self.roots.get(&height).ok_or(BridgeError::UnknownHeight)?;
    if !merkle::verify(root, encoded, proof) {
        return Err(BridgeError::BadProof);
    }
    let id = m.id();
    if self.executed.contains(&id) {
        return Err(BridgeError::AlreadyExecuted);
    }
    let total = self
        .total_minted
        .checked_add(m.amount)
        .filter(|t| *t <= self.cap)
        .ok_or(BridgeError::OverCap)?;
    self.executed.insert(id);
    self.total_minted = total;
    *self.minted.entry(m.recipient).or_default() += m.amount;
    Ok(m)
}
```

The order of the checks in `deliver` is the whole design:

1. **Not paused** (a circuit breaker an operator can pull; ✓ `pause_and_cap_stop_minting`).
2. **Decode strictly**, then **route**: the message must come from the configured source and be addressed to this chain. A message for another chain pair is refused even if its proof is valid (✓ `a_message_for_another_chain_pair_is_not_replayable_here`: a block attested by the same validators, with a message for chain 3, fails with `WrongDestination`).
3. **A trusted root for that height**: nothing is accepted for a height nobody attested (✓ `nothing_is_trusted_before_an_attestation`).
4. **Verify the proof** against that root. A changed amount, recipient or leaf position fails (✓ `a_forged_message_or_proof_is_refused`).
5. **Execute once**: the id must not be in the executed set. The id is recorded **before** the state changes, in the same step, so there is no window in which a second call can pass the check.
6. **Cap**: the running total is computed with `checked_add` and compared with the cap (a total, as a circuit breaker, from memory of how real systems limit loss). The boundary is exact: a cap equal to the total passes, one less refuses the last message (✓), and amounts near `u128::MAX` do not wrap (✓ `huge_amounts_are_compared_without_overflow`).

`submit_attestation` has the other half of the safety argument: **a height maps to at most one root**. Repeating an attestation is harmless; a different root for a known height is an `Equivocation` and is refused (✓ `attestations_need_a_quorum_and_cannot_conflict`, and the gap test that checks a conflict in the root *only* or the hash *only*). In a real system an equivocation is also evidence: two signatures by the same quorum on different blocks at one height is proof that the validators misbehaved, and should trigger the pause.

---

## 8. The relayer ✓

### 8.1 What a relayer is allowed to do

A relayer holds **no keys that matter**. It reads the source chain, asks validators for signatures, and carries `(attestation, message, proof)` to the destination. Everything it sends is verified by the inbox, so the worst a malicious relayer can do is **not deliver** or deliver late. That property is what makes it safe to let anyone run one, and it is why a protocol that lets the relayer *choose* what is true (instead of what is proven) is broken. ✓ `a_relayer_cannot_deliver_what_validators_did_not_sign`: with two of four validators online the relayer makes no progress and delivers nothing.

```rust
/// How a relayer reaches the destination.
pub trait Link {
    fn attest(&mut self, a: &Attestation) -> Result<(), LinkError>;
    fn deliver(&mut self, encoded: &[u8], height: u64, proof: &Proof) -> Result<(), LinkError>;
}
```

```rust
/// Processes every block the validators will attest to, in order, from the cursor.
/// A transient failure stops the run with the cursor on the unfinished block; calling again
/// repeats the block, which is safe because every step is idempotent at the inbox.
pub fn run_once(
    &mut self,
    chain: &SourceChain,
    validators: &[Validator],
    link: &mut impl Link,
) -> Result<Report, RelayError> {
    let mut rep = Report::default();
    if let Some((h, hash)) = self.last
        && chain.block(h).map(|b| b.hash) != Some(hash)
    {
        return Err(RelayError::FinalityViolation { height: h });
    }
    while self.cursor <= chain.tip() {
        let h = self.cursor;
        let sigs: Vec<_> = validators
            .iter()
            .filter_map(|v| v.attest(chain, h))
            .collect();
        let Some(block) = chain.block(h) else { break };
        if sigs.is_empty() {
            break; // not final yet for anyone
        }
        let att = Attestation {
            height: h,
            hash: block.hash,
            root: block.root,
            sigs,
        };
        match link.attest(&att) {
            Ok(()) => {}
            Err(LinkError::Transient) => {
                rep.stopped = true;
                return Ok(rep);
            }
            Err(LinkError::Rejected(BridgeError::Quorum(_))) => break, // not enough signatures yet
            Err(LinkError::Rejected(_)) => {
                rep.rejected += 1;
                break;
            }
        }
        for i in 0..block.msgs.len() {
            if self.poisoned.contains(&(h, i)) {
                continue;
            }
            let (enc, proof) = chain.proof_for(h, i).expect("index in range");
            match link.deliver(&enc, h, &proof) {
                Ok(()) => rep.delivered += 1,
                Err(LinkError::Rejected(BridgeError::AlreadyExecuted)) => {}
                Err(LinkError::Transient) => {
                    rep.stopped = true;
                    return Ok(rep);
                }
                Err(LinkError::Rejected(_)) => {
                    self.poisoned.push((h, i));
                    rep.rejected += 1;
                }
            }
        }
        self.last = Some((h, block.hash));
        self.cursor = h + 1;
    }
    Ok(rep)
}
```

### 8.2 Idempotency is the retry strategy

`Link` has two methods and one honest signature for failure: `LinkError::Transient` means "I do not know whether it was applied". That is the real network: a timeout after the destination has already executed the call looks the same as a call that never arrived. The relayer's answer is to make every step safe to repeat:

- **Attestations** are idempotent at the inbox (the same root again is `Ok`).
- **Deliveries** are idempotent because of the id set: a repeat returns `AlreadyExecuted`, which the relayer treats as **success** (the message is there).
- The relayer **stops at the first transient failure** and keeps its cursor on the unfinished block, so calling `run_once` again repeats that block. It never advances past a block it did not finish.
- A **poisoned** message (one the inbox refuses for a reason retrying cannot fix, here a wrong destination chain) is recorded in the relayer's own list and skipped from then on; it does not block the queue. It stays in escrow on the source: stuck, but visible (✓ `a_poisoned_message_does_not_block_the_queue`, and the gap test that shows a repeated block *skipping* it instead of being rejected again).
- A relayer started with no memory and run from block 1 repeats everything and changes nothing (✓ `rerunning_a_relayer_from_scratch_is_harmless`). A crash, a restart and a second relayer running at the same time are all the same case.

### 8.3 A hostile network ✓

The test link (`Flaky`) does, per call, one of: **loses** the call before it arrives (15–20%), **applies** it and then reports a transient failure (10–15%), **applies it twice** (10%), or behaves. ✓ For each of 12 seeds, 40 rounds of random sends from two senders to three recipients, each followed by a relayer run on this link, then repeated runs until quiet: every message in a block with at least two confirmations is executed **exactly once** (`executed_count` equals the number of messages), the destination's total equals the sum of their amounts, the source's supply is conserved, and each seed really did exercise all three failure kinds (`a_flaky_network_never_loses_or_repeats_a_message`). Fixed seeds make the failure reproducible.

### 8.4 Finality is a policy, and shallow policy loses money ✓

In `shallow_confirmations_lose_money_and_deep_ones_do_not` the validators are configured with **zero** confirmations. Block 1 holds a 500 transfer; it is attested and delivered; 500 is minted on the destination. Then block 1 is reorganized away: the source returns the escrow to the sender, and the destination still holds 500 minted against nothing (`inbox.total_minted() > chain.escrow()`). With **three** confirmations the same one-block reorganization happens before anyone signs, and nothing is minted. A cross-chain system is only as safe as its finality rule is *true*: a depth that a reorganization can exceed is a double-spend machine. The relayer also checks itself: if the last block it processed is no longer in the chain, `run_once` returns `FinalityViolation` and stops (✓ `a_reorg_below_the_cursor_stops_the_relayer`) rather than continuing on a history that has changed under it. ✓ Over 60 random rounds with reorganizations of up to two blocks and validators waiting for three, the destination never holds more than the source has escrowed (`conservation_holds_over_random_traffic_with_shallow_reorgs`).

---

## 9. The indexer ✓

```rust
fn roll_back_to(&mut self, keep: u64) {
    self.blocks.retain(|h, _| *h <= keep);
    self.events.retain(|(h, _), _| *h <= keep);
}

/// Ingests one block, given in any order and any number of times.
pub fn ingest(&mut self, b: &Block) -> Outcome {
    if b.height == 0 {
        return if self.hash_at(0) == Some(b.hash) {
            Outcome::Duplicate
        } else {
            Outcome::Orphan
        };
    }
    if let Some(h) = self.hash_at(b.height)
        && h == b.hash
    {
        return Outcome::Duplicate;
    }
    if b.height > self.tip() + 1 {
        return Outcome::Gap;
    }
    if self.hash_at(b.height - 1) != Some(b.parent) {
        return Outcome::Orphan;
    }
    let existing = self.hash_at(b.height).is_some();
    let depth = self.tip() - (b.height - 1);
    if existing || depth > 0 {
        self.roll_back_to(b.height - 1);
        self.store(b);
        return Outcome::Reorged { depth };
    }
    self.store(b);
    Outcome::Appended
}
```

```rust
/// Brings the index in line with `chain`, walking back to the common ancestor if needed.
pub fn sync(&mut self, chain: &SourceChain) {
    // find the highest indexed height whose hash still matches the chain
    let mut h = self.tip().min(chain.tip());
    while h > 0 && self.hash_at(h) != chain.block(h).map(|b| b.hash) {
        h -= 1;
    }
    self.roll_back_to(h);
    for height in h + 1..=chain.tip() {
        let b = chain.block(height).expect("in range");
        self.ingest(b);
    }
}
```

### 9.1 Blocks are stored with their parent

The index holds `height → (hash, parent)` and the events (here, the messages) of each block keyed by `(height, position)`. `ingest` classifies every block it is given: `Appended` (extends the tip), `Duplicate` (already indexed: the feed may repeat), `Gap` (a block above the next height: fetch the missing ones first), `Orphan` (its parent is not what we have) or `Reorged { depth }` (it replaces blocks we have; everything above its parent is rolled back, events included, then it is stored). Because every classification is a pure function of the stored hashes, feeding the same blocks twice, or out of order and then in order, converges (✓ `blocks_fed_one_at_a_time_in_order_converge_even_with_repeats`: each block delivered up to three times over 60 rounds with reorganizations).

### 9.2 Rolling back is part of every write

Events are **derived from blocks and deleted with them**: when a block is rolled back, its events go in the same step (`roll_back_to`). An indexer that appends events to a table and keeps a separate "latest block" cursor will, after the first reorganization, serve events from blocks that no longer exist. ✓ `a_competing_block_rolls_back_the_old_branch`: after a depth-2 rollback the event from the old block 3 is gone and the query for the new recipient shows the new event at its new height. In SQL terms: write the block row and its events in one transaction, key events by block hash or `(height, index)` with a foreign key, and `DELETE … WHERE height > ancestor` in the same transaction as the insert of the new branch.

### 9.3 Syncing against the chain ✓

`sync` walks back from the lower of the two tips to the **highest height whose hash still matches** (the common ancestor), rolls back to it and ingests forward. ✓ After 10 seeds of 80 rounds of random sends, seals and reorganizations of up to four blocks (synced about two rounds in three), every height's hash equals the chain's, the tips are equal and the event count equals the number of messages in the chain (`after_random_reorgs_the_index_equals_the_chain`). A **feed of blocks** (a subscription) cannot always do this by itself: if the chain becomes *shorter* and no competing block arrives, the indexer's tip is stale, because nothing was ever delivered that contradicts it. ✓ `a_feed_of_blocks_cannot_see_a_shorter_chain_but_sync_can`. Real feeds usually send removal notices or the new head; subscribe to *heads*, not only to new blocks, and re-sync on reconnect (Lesson 52).

### 9.4 What to expose

`events_for(address, max_height)` answers "what involves this address up to this height", and `finalized_height(depth)` says which part of the answer is unlikely to change. An API that returns data from the head without saying so is promising more than a chain can give: let the caller ask for "finalized only" and say how many confirmations that means (✓ `queries_respect_the_height_bound_and_finality`).

---

## 10. Cost ✓

`cargo run --release --example bench`, three runs, ranges over the three runs, release profile on 2 shared cores (orders of magnitude):

| Operation | per operation |
|---|---|
| Message encode and id (106 bytes) | 121–173 ns |
| Merkle root of 64 leaves | 17,651–18,346 ns |
| Merkle prove (64 leaves) | 18,373–20,009 ns |
| Merkle verify (depth 6) | 935–1,266 ns |
| Validator attest (sign) | 17,998–21,617 ns |
| Quorum verify, 5 of 7 signatures | 222,213–306,787 ns |
| Relay 100 messages end to end (10 blocks, 4 validators) | 2.68–3.89 ms |
| Index 1,000 blocks, one message each | 230,742–340,808 ns |

Hashes are cheap and **signatures dominate**: one signature verification costs roughly 45–60 µs here (the quorum row divided by five; my arithmetic), against about 1 µs to verify a Merkle proof. That is why a design that verifies a quorum **once per block** and a Merkle proof **per message** scales with messages, while one that verifies a signature per message does not. In the end-to-end row almost all of the time is attesting (signing by 4 validators, verifying 3 of those signatures at the inbox, for each of 10 blocks); the hashing is a small part.

---

## 11. Rust notes

- **Make illegal states hard to build.** `ValidatorSet::new` returns `Result` and refuses a weak threshold, so there is no way to hold a set that violates the two-thirds rule (Lesson 15).
- **A trait for the thing that fails.** `Link` abstracts "how I reach the destination" so that the reliable inbox and the hostile test link are the same type to the relayer; the test double is 40 lines in `tests/common/mod.rs`. Design for the double you will want.
- **Distinguish "no" from "I do not know".** `LinkError::Transient` versus `Rejected(_)` is the most important enum in the crate; collapsing them into one error loses messages or repeats them (Lesson 26).
- **Let-chains.** The relayer's and indexer's guards use `if let Some(x) = … && condition` (stable in edition 2024 on this toolchain), which keeps a check and the binding it needs in one place.
- **`checked_add`, `saturating_sub`, `div_ceil`.** Every length, height, depth and sum that can come from outside goes through a checked or saturating operation.
- **`BTreeMap` for deterministic iteration.** The state uses ordered maps so that two runs on the same input produce the same output (Lesson 50).
- **Test helpers belong in `tests/common/mod.rs`**, not `tests/common.rs`, or Cargo treats the helper file as a test of its own.

---

## 12. Common mistakes

1. Trusting a relayer's claim instead of a proof against an attested root (§8.1).
2. Executing a message without recording its id first, or recording the id in a different step from the state change (§7).
3. Keying the executed set on something shorter than the whole identity: a nonce alone, or a hash of the payload without routing (§15, the debugging challenge).
4. Hashing leaves and nodes the same way, or duplicating the odd node (§4.1).
5. Taking an index or a size from a proof and not binding it to the root (§4.2).
6. Verifying signatures without strict checks, so that a weak key forges (§6.2).
7. A quorum threshold at or below two thirds, or counting the same signer twice (§6.2).
8. Calling a block final because it is deep *on the node you asked* (§8.4); finality is a property of the source chain's consensus (Lesson 52).
9. Treating a timeout as failure and not retrying idempotently, or treating it as success and not retrying at all (§8.2).
10. Letting one malformed message block the queue behind it (§8.2).
11. An indexer that appends and never rolls back, or that rolls back events but not the cursor (§9.2).
12. Subscribing to blocks but not to heads, so a shorter chain is invisible (§9.3).
13. No circuit breaker: no cap, no pause, no monitoring of minted versus escrowed (§7).
14. A permanently upgradeable bridge contract controlled by one key, or a validator set that never rotates (§13).
15. Calling this model "a bridge". It has no fees, no timeouts, no refunds of undeliverable messages, no ordering guarantees and no real chains.

---

## 13. Security notes and honest limits

- This is a **model written from memory**. It was not compared with any real protocol. The cryptographic vectors were computed independently in Python; nothing else was checked against a real system. A clean test run says nothing about a deployed bridge.
- **Trust assumptions are the product.** The model trusts that more than two thirds of the validators are honest and keep their keys safe, that their notion of finality is true, and that the inbox's configuration (validator keys, source id, cap) is right. Break any of the three and the money is gone. The incidents in §2 are all one of these.
- **Key management** is the biggest real-world risk and is absent from the model: where validator keys live (hardware modules, threshold signing, from memory), how they are rotated, how many are held by one operator, and how a compromised key is removed. A set of five validators run by one company is one validator.
- **Rotation and upgrades.** The inbox's validator set is fixed at construction. A real system needs a way to change it that is itself protected by the old set, and a way to pause that an attacker cannot use against users. Both are governance and are not modelled.
- **Replay across chains and deployments.** The domain tags, the chain ids inside the attestation, and the routing inside the message are what stop an attestation for one deployment from working on another (✓ `a_message_for_another_chain_pair_is_not_replayable_here`). Test them whenever you add a chain.
- **Ordering, fees, timeouts and refunds.** Messages here may execute in any order, cost nothing and never expire. A real protocol decides each of these; a message that cannot be delivered (§8.2, poisoned) needs a refund path, which the model lacks.
- **Denial of service.** An attacker can fill blocks with small messages so the relayer spends more than a message is worth. Fees, minimum amounts and rate limits are the response and are not modelled.
- **Monitoring is a security control.** Alert on `minted > escrowed`, on equivocation, on a stuck queue and on a relayer's `FinalityViolation`; each is cheap to compute (the tests compute the first of them).
- **Unsafe shortcuts in the model:** `u128` amounts, ed25519 keys derived from small seeds in tests (never do that outside a test), no persistence, no concurrency.
- Mutation testing measures this crate's tests (§17), not any real bridge.
- If you find a vulnerability in a real bridge or relayer, report it privately to its maintainers and give them time to fix it. Never test against a network you do not own or have not been authorized to test.

---

## 14. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Write `proof_len(index, size)`: how many siblings the proof for a leaf has, with odd nodes promoted, without building the tree.
2. **(Easy)** Write `distinct_signers(indexes, n)`: the number of distinct signers, or `None` if an index is out of range.
3. **(Easy)** Write `retry_delay(attempt, base, cap)`: exponential backoff in milliseconds that never overflows.
4. **(Medium)** Write `reorg_depth(old, new)`: how many indexed blocks must be rolled back when the chain switches to `new`.
5. **(Medium)** Write `final_height(tip, depth)` and `confirmations(tip, height)`.
6. **(Medium-hard)** Write `fresh(seen, batch)`: the events of a batch not seen before, in order, recording them.
7. **(Hard)** Write `batches(amounts, limit)`: split transfers into consecutive groups none of which moves more than `limit`.
8. **(Hard)** Write `missing(seen)`: the nonces never seen between 0 and the largest one.

<details markdown="1">
<summary>Answers</summary>

1. A possible answer (✓ compiled and run in a scratch crate with tests):

   ```rust
   //! Exercise 1: how many siblings does the proof for one leaf contain?

   /// The number of siblings in the proof for leaf `index` of a tree of `size` leaves, where an odd
   /// node at the end of a level is promoted without a sibling. `None` if `index >= size`.
   pub fn proof_len(index: u64, size: u64) -> Option<usize> {
       if index >= size {
           return None;
       }
       let (mut idx, mut n, mut len) = (index, size, 0);
       while n > 1 {
           if idx % 2 == 1 || idx + 1 < n {
               len += 1;
           }
           idx /= 2;
           n = n.div_ceil(2);
       }
       Some(len)
   }
   ```

   A node at an odd position always has a left sibling; a node at an even position has a right sibling unless it is the last of an odd level (then it is promoted). The test compares the function with the real prover for **every leaf of every tree of 1 to 70 leaves**.
2. A possible answer (✓):

   ```rust
   //! Exercise 2: count the distinct signers of an attestation.

   /// The number of distinct signer indexes, or `None` if any index is `>= n`.
   /// A signer who appears twice counts once.
   pub fn distinct_signers(indexes: &[u32], n: u32) -> Option<usize> {
       let mut seen = vec![false; n as usize];
       let mut count = 0;
       for &i in indexes {
           let slot = seen.get_mut(i as usize)?;
           if !*slot {
               *slot = true;
               count += 1;
           }
       }
       Some(count)
   }
   ```

   It uses a `Vec<bool>` of length `n` and `get_mut`, so an out-of-range index becomes `None` through `?` and nothing indexes out of bounds. It never allocates from the *signature count*, only from the set size, which the verifier owns.
3. A possible answer (✓):

   ```rust
   //! Exercise 3: how long to wait before the next retry.

   /// `base * 2^attempt` milliseconds, never more than `cap`, and never an overflow, whatever `attempt` is.
   pub fn retry_delay(attempt: u32, base: u64, cap: u64) -> u64 {
       let shifted = if attempt >= 64 {
           u64::MAX
       } else {
           base.saturating_mul(1u64 << attempt)
       };
       shifted.min(cap)
   }
   ```

   `1u64 << attempt` overflows for 64 or more, so that case is handled first; the multiplication saturates; the cap applies last. The test runs 200 attempts, checks monotonicity and the cap, and `retry_delay(u32::MAX, u64::MAX, u64::MAX)`. In production add jitter, so that many relayers do not retry in step (Lesson 52).
4. A possible answer (✓):

   ```rust
   //! Exercise 4: how many indexed blocks must be rolled back?

   /// `old` and `new` are block hashes from height 0 upward. The depth of the reorganization is the
   /// number of blocks of `old` that are not a prefix of `new`.
   pub fn reorg_depth<T: PartialEq>(old: &[T], new: &[T]) -> usize {
       let common = old.iter().zip(new).take_while(|(a, b)| a == b).count();
       old.len() - common
   }
   ```

   The common prefix of the two hash lists is the shared history; everything after it in `old` must go. The test also builds a real six-block chain, reorganizes three blocks, and checks that the **indexer** reports `Reorged { depth: 3 }` for the same pair.
5. A possible answer (✓):

   ```rust
   //! Exercise 5: which block is final?

   /// The highest block with at least `depth` blocks on top of it, or `None` if there is none
   /// (a chain with `tip <= depth` has no final block above genesis; genesis counts as final only
   /// when it has `depth` blocks on top).
   pub fn final_height(tip: u64, depth: u64) -> Option<u64> {
       tip.checked_sub(depth)
   }

   /// How many blocks of the chain with tip `tip` confirm `height` (itself included); zero if the
   /// block is above the tip.
   pub fn confirmations(tip: u64, height: u64) -> u64 {
       if height > tip { 0 } else { tip - height + 1 }
   }
   ```

   `checked_sub` makes "not enough blocks yet" a `None` instead of an underflow. The test checks that a block at `final_height` has exactly `depth + 1` confirmations (it counts itself).
6. A possible answer (✓):

   ```rust
   //! Exercise 6: apply each event once.
   use std::collections::HashSet;

   /// Returns the events of `batch` that have not been seen before, in order, and records them.
   /// An event repeated inside the batch counts once. The key is the whole identity of the event.
   pub fn fresh<K: std::hash::Hash + Eq + Clone>(seen: &mut HashSet<K>, batch: &[K]) -> Vec<K> {
       batch
           .iter()
           .filter(|k| seen.insert((*k).clone()))
           .cloned()
           .collect()
   }
   ```

   `HashSet::insert` returns whether the value was new, so the filter and the recording are one operation, and a repeat inside the same batch is caught too. The key type is generic so the same function works for ids and for `(sender, nonce)` pairs, which is the point of §15.
7. A possible answer (✓):

   ```rust
   //! Exercise 7: group transfers so that no group moves more than a limit.

   /// Splits `amounts` into consecutive groups whose sums are at most `limit`, keeping the order.
   /// `None` if a single amount exceeds the limit. Sums never overflow.
   pub fn batches(amounts: &[u128], limit: u128) -> Option<Vec<Vec<u128>>> {
       let mut out: Vec<Vec<u128>> = Vec::new();
       let mut sum = 0u128;
       for &a in amounts {
           if a > limit {
               return None;
           }
           match out.last_mut() {
               Some(group) if sum <= limit - a => {
                   group.push(a);
                   sum += a;
               }
               _ => {
                   out.push(vec![a]);
                   sum = a;
               }
           }
       }
       Some(out)
   }
   ```

   The comparison is `sum <= limit - a` rather than `sum + a <= limit`, so nothing can overflow near `u128::MAX` (the test uses `u128::MAX - 1`). A single amount over the limit is an error, not a group of its own.
8. A possible answer (✓):

   ```rust
   //! Exercise 8: find the nonces that never arrived.

   /// The missing values between 0 and the largest nonce seen, in increasing order.
   /// Duplicates and any input order are fine. An empty input has no gaps.
   pub fn missing(seen: &[u64]) -> Vec<u64> {
       let Some(&max) = seen.iter().max() else {
           return vec![];
       };
       let mut sorted: Vec<u64> = seen.to_vec();
       sorted.sort_unstable();
       sorted.dedup();
       let mut out = Vec::new();
       let mut next = 0u64;
       for n in sorted {
           out.extend(next..n);
           next = n + 1;
       }
       debug_assert!(next == max + 1);
       out
   }
   ```

   Sort, remove duplicates and walk, emitting the ranges between consecutive values. A monitor that runs this over executed nonces per sender finds messages that were never delivered; the debug assertion checks the walk ended at the maximum.

</details>

---

## 15. Debugging challenge

A destination inbox protects against replays with a set of "already seen" nonces. In testing, with one test sender, it works. On a network with two senders, half of the users report that their transfers "never arrive", although the relayer's logs show every message was delivered. Find the bug and fix it.

```rust
/// The faulty program.
pub mod buggy {
    use std::collections::HashSet;

    /// Accepts each message at most once. A message is `(sender, nonce, amount)`.
    #[derive(Default)]
    pub struct Inbox {
        seen: HashSet<u64>,
        pub minted: u128,
    }

    impl Inbox {
        /// Returns whether the message was applied.
        pub fn deliver(&mut self, _sender: u8, nonce: u64, amount: u128) -> bool {
            if !self.seen.insert(nonce) {
                return false;
            }
            self.minted += amount;
            true
        }
    }
}
```

<details markdown="1">
<summary>Solution</summary>

The set is keyed on `nonce` alone, but a nonce is **per sender**: sender 1's message 0 and sender 2's message 0 have the same nonce and are different messages. The second one is treated as a replay and silently dropped (`deliver` returns `false`, which the relayer's logs read as "already done"). With one sender every nonce is unique, so the tests passed; with two, every message of the later sender that reuses a nonce is lost, and the destination mints less than the source escrowed.

✓ The test sends five messages (nonces 0 to 4) from each of two senders: the buggy inbox applies **5 of 10** and mints 50 instead of 100; the fixed one applies all 10, still refuses a true replay, and behaves identically to the buggy one when there is one sender.

Fixed version (✓ compiled):

```rust
/// The corrected program.
pub mod fixed {
    use std::collections::HashSet;

    /// Accepts each message at most once. A message is `(sender, nonce, amount)`.
    #[derive(Default)]
    pub struct Inbox {
        seen: HashSet<(u8, u64)>,
        pub minted: u128,
    }

    impl Inbox {
        /// Returns whether the message was applied.
        pub fn deliver(&mut self, sender: u8, nonce: u64, amount: u128) -> bool {
            // a nonce counts per sender, so the identity of a message is the pair
            if !self.seen.insert((sender, nonce)) {
                return false;
            }
            self.minted += amount;
            true
        }
    }
}
```

The general lesson: **the key of a deduplication set must be the whole identity of the thing**. In the crate that identity is the message id, a hash over source, destination, nonce, sender, recipient, amount and payload; in this reduced example it is at least `(sender, nonce)`. Test deduplication with two writers that use the same counter values, not one.

</details>

---

## 16. Interview questions

1. *Why can't a contract on one chain just read the state of another? What are the main ways around it?*
2. *Describe lock-and-mint, burn-and-mint and a liquidity-pool bridge. Where is the risk in each?*
3. *What does a Merkle proof prove, and what must a verifier check besides the hashes?*
4. *Why prefix leaves and nodes differently? What is wrong with duplicating the last node of an odd level?*
5. *Why must a quorum threshold exceed two thirds? What does a quorum signature cover, and why does it include the chain id?*
6. *What is a relayer allowed to do? What happens if every relayer is malicious?*
7. *A relayer's call to the destination times out. What can have happened, and how do you handle it?*
8. *How do you make message execution idempotent? What is the key of the "already executed" set?*
9. *What is finality, and why is "N confirmations" a policy and not a fact? What can go wrong if N is too small?*
10. *What is an equivocation, and what should a bridge do on seeing one?*
11. *A message can never be delivered (the recipient reverts every time). What should the system do?*
12. *Describe how an indexer survives a reorganization, step by step.*
13. *What invariants would you monitor on a bridge in production?*
14. *Name three ways a bridge has been drained in practice, and what control would have prevented each.*
15. *You are asked to review a cross-chain contract. What do you check first?*

<details markdown="1">
<summary>Answers</summary>

1. A contract cannot open a network connection, and a chain's state is not exposed to others. Something must carry a claim and something must verify it: an external validator set (trust in its keys), a light client of the source chain running on the destination (trust in the source's consensus and the client code), optimistic fraud proofs with a challenge window (trust in an honest watcher) or validity proofs (trust in the proof system). Each trades cost, latency and trust (from memory).
2. Lock-and-mint locks assets in a vault on the source and mints a wrapped token on the destination: the risk is the vault and the minting permission. Burn-and-mint keeps total supply across chains constant by burning on one and minting on the other: the risk is the minting permission. Liquidity pools pay out from a pool on the destination and settle later: the risk is the pool and the settlement. In all three, whoever can create a valid-looking message controls the money.
3. That a leaf is in a tree with a given root at a given position. A verifier must also check that the root itself is trusted (attested), that the leaf is the exact message it will act on, that the index and size are bound (so the path is the right shape) and that the message is addressed to it; and it must refuse proofs with extra or missing siblings (✓ §4.2).
4. Domain separation stops a node from being presented as a leaf (a second-preimage attack). Duplicating the last node makes different leaf lists share a root (`[a,b,c]` and `[a,b,c,c]`), so a proof for the second does not prove the first is not the same tree; promotion has no such pair (✓ §4.1).
5. With a threshold above two thirds, any two quorums share at least one signer, so with fewer than a third faulty two conflicting blocks cannot both be signed by honest validators. The signature covers the chain id, height, header hash and message root, so it cannot be replayed on another chain or moved to another block (✓ §6).
6. Carry data. Everything it sends is verified, so it can only fail to deliver or deliver late. If every relayer is malicious, nothing moves, but nothing is stolen; liveness fails and safety holds. That is why relaying can be permissionless and why anyone should be able to run one.
7. Either the call never arrived, or it arrived and was applied and the response was lost. The caller cannot tell. The fix is idempotency at the receiver (an already-executed set keyed by the message id), a retry that treats `AlreadyExecuted` as success, and a cursor that does not advance past unfinished work (✓ §8.2).
8. Record the id before changing state, in the same atomic step, and make the id cover the full identity: source, destination, nonce, sender, recipient, amount and payload. A key shorter than the identity (a nonce alone) loses messages (§15).
9. Finality is the point after which the source chain's consensus will not revert a block (probabilistically for some chains, absolutely for others; Lesson 52). "N confirmations" is your estimate of that point. If a reorganization can be deeper than N, an action taken on a block that is later dropped (minting, paying out) has no backing: ✓ the 500 minted against nothing in §8.4.
10. Two conflicting signed statements for the same slot (here, two different roots for one height). It proves misbehavior by the signers. A bridge should refuse the second, alert, and usually pause; slashing is the economic response in systems that have stake (from memory).
11. Do not let it block the queue (skip it after bounded retries, record it), keep its funds safe and visible, and offer a refund or manual redemption path on the source after proof of non-delivery or a timeout. The model records it in `poisoned` and leaves the escrow in place (§8.2); a refund path is a gap I state in §13.
12. Store blocks by hash with their parent and derive events from them. On a block whose parent is not the indexed block below it, find the common ancestor, delete everything above it (blocks and events, in the same transaction), and ingest the new branch. Make each write idempotent. Keep a finalized cursor and let the API say whether a result is final (✓ §9).
13. `minted <= escrowed` (the main one), no equivocations, the queue's age and length, the relayer's cursor against the source's finalized height, validator signing participation, the pause state and the balance and fee budget of the relayer. Alerts should page a human.
14. From memory: compromised validator keys met the threshold (control: more independent signers, hardware-protected keys, rate limits); a verification bug accepted forged messages (control: audits, differential tests, caps); a bad initialization or configuration accepted any message (control: deployment checks, invariants monitored after each change, staged rollouts). I did not verify the details of any incident.
15. Where trust enters: who can add or replace validators, who can upgrade or pause, what exactly a signature covers, what keys the executed-set uses, whether every arithmetic step is checked, whether finality assumptions are explicit, and whether there is a cap and a way to stop the system. Then try to make the same message execute twice, on two chains, and to execute one nobody signed.

</details>

---

## 17. Practical challenge: `relaylab`

Build a library crate that:

1. Defines a **message** with a canonical fixed-width encoding, strict decoding and a domain-separated id.
2. Implements a **Merkle tree** with domain-separated leaves and nodes, promotion of odd nodes, a root that commits to the leaf count, and proofs bound to index and size.
3. Implements a **source chain** with escrow, sealed blocks that commit to their messages, header hashes and reorganizations that return escrow.
4. Implements **validator attestations** with ed25519: distinct, known, strictly verified signers and a threshold above two thirds; validators sign only blocks with enough confirmations.
5. Implements a **destination inbox**: trusted roots from attestations (no equivocation), routing checks, proof verification, execute-once with a cap and a pause.
6. Implements an **untrusted relayer** over a `Link` trait that survives transient and ambiguous failures, skips poisoned messages and detects a finality violation, and a **reorganization-aware indexer**; passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, has a benchmark example and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

```toml
[package]
name = "relaylab"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
ed25519-dalek = "3.0.0"
sha2 = "0.11.1"

[profile.dev.package."*"]
opt-level = 3

[dev-dependencies]
hex = "0.4.3"
```

### `src/lib.rs`

```rust
//! A teaching model of cross-chain infrastructure: a source chain that escrows funds and emits
//! messages, a quorum of validators that attest to finalized blocks, a destination inbox that
//! verifies Merkle proofs and executes each message once, an untrusted relayer, and a
//! reorganization-aware indexer. Not the real stack; not audited.
pub mod bridge;
pub mod chain;
pub mod hash;
pub mod indexer;
pub mod merkle;
pub mod msg;
pub mod quorum;
pub mod relayer;
```

### `src/hash.rs`

```rust
//! Domain-separated SHA-256.
use sha2::{Digest, Sha256};

/// `SHA-256(len(domain) as one byte || domain || parts...)`. Callers pass fixed-width parts, so the
/// concatenation cannot be re-read as a different split; the domain tag keeps unrelated uses apart.
pub fn sha256(domain: &[u8], parts: &[&[u8]]) -> [u8; 32] {
    assert!(domain.len() < 256);
    let mut h = Sha256::new();
    h.update([domain.len() as u8]);
    h.update(domain);
    for p in parts {
        h.update(p);
    }
    h.finalize().into()
}
```

### `src/merkle.rs`

```rust
//! A binary Merkle tree with domain-separated leaves and nodes, and index-bound proofs.
//! The root also commits to the number of leaves, so the size in a proof cannot be chosen freely.
//!
//! A level with an odd number of nodes promotes its last node unchanged; it is never duplicated,
//! because duplicating makes `[a, b, c]` and `[a, b, c, c]` share a root.
use sha2::{Digest, Sha256};

pub type Hash = [u8; 32];

/// `SHA-256(0x00 || data)`.
pub fn leaf_hash(data: &[u8]) -> Hash {
    let mut h = Sha256::new();
    h.update([0x00]);
    h.update(data);
    h.finalize().into()
}

/// `SHA-256(0x01 || left || right)`.
pub fn node_hash(l: &Hash, r: &Hash) -> Hash {
    let mut h = Sha256::new();
    h.update([0x01]);
    h.update(l);
    h.update(r);
    h.finalize().into()
}

/// `SHA-256(0x02 || size as 8 bytes || top)`: binds the leaf count to the tree's top hash.
pub fn commit(size: u64, top: &Hash) -> Hash {
    let mut h = Sha256::new();
    h.update([0x02]);
    h.update(size.to_be_bytes());
    h.update(top);
    h.finalize().into()
}

/// The root of an empty tree.
pub fn empty_root() -> Hash {
    commit(0, &Sha256::new().finalize().into())
}

fn next_level(level: &[Hash]) -> Vec<Hash> {
    level
        .chunks(2)
        .map(|c| {
            if c.len() == 2 {
                node_hash(&c[0], &c[1])
            } else {
                c[0]
            }
        })
        .collect()
}

/// The root of the tree over `leaves`.
pub fn root(leaves: &[Vec<u8>]) -> Hash {
    if leaves.is_empty() {
        return empty_root();
    }
    let mut level: Vec<Hash> = leaves.iter().map(|l| leaf_hash(l)).collect();
    while level.len() > 1 {
        level = next_level(&level);
    }
    commit(leaves.len() as u64, &level[0])
}

/// A proof that one leaf sits at `index` in a tree of `size` leaves.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Proof {
    pub index: u64,
    pub size: u64,
    pub siblings: Vec<Hash>,
}

/// The proof for `leaves[index]`.
pub fn prove(leaves: &[Vec<u8>], index: usize) -> Option<Proof> {
    if index >= leaves.len() {
        return None;
    }
    let mut level: Vec<Hash> = leaves.iter().map(|l| leaf_hash(l)).collect();
    let mut idx = index;
    let mut siblings = Vec::new();
    while level.len() > 1 {
        if idx % 2 == 1 {
            siblings.push(level[idx - 1]);
        } else if idx + 1 < level.len() {
            siblings.push(level[idx + 1]);
        }
        idx /= 2;
        level = next_level(&level);
    }
    Some(Proof {
        index: index as u64,
        size: leaves.len() as u64,
        siblings,
    })
}

/// Checks that `leaf` is at `proof.index` of a tree of `proof.size` leaves with root `root`.
/// The shape of the path is derived from the index and size, so a proof cannot claim a different
/// depth, and every sibling must be used exactly once.
pub fn verify(root: &Hash, leaf: &[u8], proof: &Proof) -> bool {
    if proof.size == 0 || proof.index >= proof.size {
        return false;
    }
    let mut h = leaf_hash(leaf);
    let (mut idx, mut n, mut used) = (proof.index, proof.size, 0usize);
    while n > 1 {
        if idx % 2 == 1 {
            let Some(s) = proof.siblings.get(used) else {
                return false;
            };
            used += 1;
            h = node_hash(s, &h);
        } else if idx + 1 < n {
            let Some(s) = proof.siblings.get(used) else {
                return false;
            };
            used += 1;
            h = node_hash(&h, s);
        }
        idx /= 2;
        n = n.div_ceil(2);
    }
    used == proof.siblings.len() && commit(proof.size, &h) == *root
}
```

### `src/msg.rs`

```rust
//! The cross-chain message and its canonical encoding.
use crate::hash::sha256;

pub type Addr = [u8; 20];
/// Longest payload a message may carry.
pub const MAX_PAYLOAD: usize = 256;
/// Bytes before the payload: 4 + 4 + 8 + 20 + 20 + 16 + 2.
pub const HEADER_LEN: usize = 74;

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Message {
    pub src: u32,
    pub dst: u32,
    pub nonce: u64,
    pub sender: Addr,
    pub recipient: Addr,
    pub amount: u128,
    pub payload: Vec<u8>,
}

#[derive(Debug, PartialEq, Eq)]
pub enum MsgError {
    Short,
    PayloadTooLong,
    Trailing,
}

impl Message {
    /// Fixed-width big-endian fields, then a two-byte payload length and the payload.
    pub fn encode(&self) -> Vec<u8> {
        let mut o = Vec::with_capacity(HEADER_LEN + self.payload.len());
        o.extend(self.src.to_be_bytes());
        o.extend(self.dst.to_be_bytes());
        o.extend(self.nonce.to_be_bytes());
        o.extend(self.sender);
        o.extend(self.recipient);
        o.extend(self.amount.to_be_bytes());
        o.extend((self.payload.len() as u16).to_be_bytes());
        o.extend(&self.payload);
        o
    }

    /// Strict: the length field must match, the payload must be short enough, nothing may follow.
    pub fn decode(b: &[u8]) -> Result<Message, MsgError> {
        if b.len() < HEADER_LEN {
            return Err(MsgError::Short);
        }
        let plen = usize::from(u16::from_be_bytes([b[72], b[73]]));
        if plen > MAX_PAYLOAD {
            return Err(MsgError::PayloadTooLong);
        }
        if b.len() < HEADER_LEN + plen {
            return Err(MsgError::Short);
        }
        if b.len() > HEADER_LEN + plen {
            return Err(MsgError::Trailing);
        }
        Ok(Message {
            src: u32::from_be_bytes(b[0..4].try_into().expect("4")),
            dst: u32::from_be_bytes(b[4..8].try_into().expect("4")),
            nonce: u64::from_be_bytes(b[8..16].try_into().expect("8")),
            sender: b[16..36].try_into().expect("20"),
            recipient: b[36..56].try_into().expect("20"),
            amount: u128::from_be_bytes(b[56..72].try_into().expect("16")),
            payload: b[HEADER_LEN..].to_vec(),
        })
    }

    /// The message id: a domain-separated hash of the whole encoding, source and destination included.
    pub fn id(&self) -> [u8; 32] {
        sha256(b"msg-v1", &[&self.encode()])
    }
}
```

### `src/chain.rs`

```rust
//! The source chain: balances, an escrow, sealed blocks that commit to their messages, reorgs.
use crate::hash::sha256;
use crate::merkle::{self, Hash, Proof};
use crate::msg::{Addr, MAX_PAYLOAD, Message};
use std::collections::BTreeMap;

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Block {
    pub height: u64,
    pub parent: Hash,
    pub root: Hash,
    pub hash: Hash,
    pub msgs: Vec<Message>,
}

#[derive(Debug, PartialEq, Eq)]
pub enum SendError {
    Insufficient,
    ZeroAmount,
    PayloadTooLong,
}

/// The header hash binds the chain id, height, parent and message root.
pub fn header_hash(chain: u32, height: u64, parent: &Hash, root: &Hash) -> Hash {
    sha256(
        b"hdr-v1",
        &[&chain.to_be_bytes(), &height.to_be_bytes(), parent, root],
    )
}

pub struct SourceChain {
    pub id: u32,
    balances: BTreeMap<Addr, u128>,
    escrow: u128,
    blocks: Vec<Block>,
    pending: Vec<Message>,
    next_nonce: u64,
    initial: u128,
}

impl SourceChain {
    pub fn new(id: u32, funded: &[(Addr, u128)]) -> SourceChain {
        let balances: BTreeMap<Addr, u128> = funded.iter().copied().collect();
        let initial = balances.values().sum();
        let root = merkle::empty_root();
        let genesis = Block {
            height: 0,
            parent: [0; 32],
            root,
            hash: header_hash(id, 0, &[0; 32], &root),
            msgs: vec![],
        };
        SourceChain {
            id,
            balances,
            escrow: 0,
            blocks: vec![genesis],
            pending: vec![],
            next_nonce: 0,
            initial,
        }
    }

    pub fn tip(&self) -> u64 {
        self.blocks.len() as u64 - 1
    }
    pub fn block(&self, height: u64) -> Option<&Block> {
        self.blocks.get(usize::try_from(height).ok()?)
    }
    pub fn balance(&self, a: &Addr) -> u128 {
        self.balances.get(a).copied().unwrap_or(0)
    }
    pub fn escrow(&self) -> u128 {
        self.escrow
    }
    pub fn pending_len(&self) -> usize {
        self.pending.len()
    }
    /// Balances plus escrow: conserved by every operation.
    pub fn total_supply(&self) -> u128 {
        self.balances.values().sum::<u128>() + self.escrow
    }
    pub fn initial_supply(&self) -> u128 {
        self.initial
    }
    /// The sum of the amounts of every message currently in a sealed block.
    pub fn escrowed_in_blocks(&self) -> u128 {
        self.blocks
            .iter()
            .flat_map(|b| &b.msgs)
            .map(|m| m.amount)
            .sum()
    }

    /// Locks `amount` of the sender's balance in escrow and queues a message for the next block.
    pub fn send(
        &mut self,
        sender: Addr,
        recipient: Addr,
        dst: u32,
        amount: u128,
        payload: Vec<u8>,
    ) -> Result<Message, SendError> {
        if amount == 0 {
            return Err(SendError::ZeroAmount);
        }
        if payload.len() > MAX_PAYLOAD {
            return Err(SendError::PayloadTooLong);
        }
        let bal = self.balances.entry(sender).or_default();
        if *bal < amount {
            return Err(SendError::Insufficient);
        }
        *bal -= amount;
        self.escrow += amount;
        let m = Message {
            src: self.id,
            dst,
            nonce: self.next_nonce,
            sender,
            recipient,
            amount,
            payload,
        };
        self.next_nonce += 1;
        self.pending.push(m.clone());
        Ok(m)
    }

    /// Seals the pending messages into a new block.
    pub fn seal(&mut self) -> &Block {
        let msgs = std::mem::take(&mut self.pending);
        let leaves: Vec<Vec<u8>> = msgs.iter().map(Message::encode).collect();
        let root = merkle::root(&leaves);
        let parent = self.blocks.last().expect("genesis").hash;
        let height = self.tip() + 1;
        let hash = header_hash(self.id, height, &parent, &root);
        self.blocks.push(Block {
            height,
            parent,
            root,
            hash,
            msgs,
        });
        self.blocks.last().expect("just pushed")
    }

    /// Drops the last `depth` blocks (never the genesis block) and returns the escrowed funds of
    /// their messages to the senders: those transactions are no longer part of the chain.
    pub fn reorg(&mut self, depth: u64) -> u64 {
        let mut dropped = 0;
        while dropped < depth && self.blocks.len() > 1 {
            let b = self.blocks.pop().expect("len > 1");
            for m in b.msgs {
                self.escrow -= m.amount;
                *self.balances.entry(m.sender).or_default() += m.amount;
            }
            dropped += 1;
        }
        dropped
    }

    /// The encoded message at `index` of block `height`, with its proof against the block's root.
    pub fn proof_for(&self, height: u64, index: usize) -> Option<(Vec<u8>, Proof)> {
        let b = self.block(height)?;
        let leaves: Vec<Vec<u8>> = b.msgs.iter().map(Message::encode).collect();
        let p = merkle::prove(&leaves, index)?;
        Some((leaves[index].clone(), p))
    }
}
```

### `src/quorum.rs`

```rust
//! Validator attestations: a threshold of ed25519 signatures over one finalized header.
use crate::chain::SourceChain;
use crate::merkle::Hash;
use ed25519_dalek::{Signature, Signer, SigningKey, VerifyingKey};
use std::collections::BTreeSet;

/// The bytes a validator signs: domain tag, chain id, height, header hash, message root.
pub fn attest_bytes(chain: u32, height: u64, hash: &Hash, root: &Hash) -> Vec<u8> {
    let mut o = b"att-v1".to_vec();
    o.extend(chain.to_be_bytes());
    o.extend(height.to_be_bytes());
    o.extend(hash);
    o.extend(root);
    o
}

/// The smallest threshold that is more than two thirds of `n`.
pub fn threshold_for(n: usize) -> usize {
    2 * n / 3 + 1
}

#[derive(Debug, PartialEq, Eq)]
pub enum QuorumError {
    WeakThreshold,
    UnknownSigner,
    Duplicate,
    BadSignature,
    NotEnough,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Attestation {
    pub height: u64,
    pub hash: Hash,
    pub root: Hash,
    pub sigs: Vec<(u32, [u8; 64])>,
}

pub struct Validator {
    pub index: u32,
    key: SigningKey,
    /// Confirmations this validator requires before it signs a block.
    pub depth: u64,
}

impl Validator {
    pub fn from_seed(index: u32, seed: [u8; 32], depth: u64) -> Validator {
        Validator {
            index,
            key: SigningKey::from_bytes(&seed),
            depth,
        }
    }
    pub fn public(&self) -> [u8; 32] {
        self.key.verifying_key().to_bytes()
    }
    /// Signs the block at `height` of `chain` if it has at least `depth` confirmations.
    pub fn attest(&self, chain: &SourceChain, height: u64) -> Option<(u32, [u8; 64])> {
        if height.checked_add(self.depth)? > chain.tip() {
            return None;
        }
        let b = chain.block(height)?;
        let sig = self
            .key
            .sign(&attest_bytes(chain.id, b.height, &b.hash, &b.root));
        Some((self.index, sig.to_bytes()))
    }
}

pub struct ValidatorSet {
    keys: Vec<VerifyingKey>,
    threshold: usize,
}

impl ValidatorSet {
    /// A set needs a threshold above two thirds, so two conflicting quorums share a validator.
    pub fn new(keys: &[[u8; 32]], threshold: usize) -> Result<ValidatorSet, QuorumError> {
        if keys.is_empty() || threshold < threshold_for(keys.len()) || threshold > keys.len() {
            return Err(QuorumError::WeakThreshold);
        }
        let keys = keys
            .iter()
            .map(|k| VerifyingKey::from_bytes(k).map_err(|_| QuorumError::UnknownSigner))
            .collect::<Result<_, _>>()?;
        Ok(ValidatorSet { keys, threshold })
    }
    pub fn len(&self) -> usize {
        self.keys.len()
    }
    pub fn is_empty(&self) -> bool {
        self.keys.is_empty()
    }
    pub fn threshold(&self) -> usize {
        self.threshold
    }

    /// Checks distinct, known signers with valid signatures, at least `threshold` of them.
    pub fn verify(&self, chain: u32, a: &Attestation) -> Result<(), QuorumError> {
        let msg = attest_bytes(chain, a.height, &a.hash, &a.root);
        let mut seen = BTreeSet::new();
        for (i, sig) in &a.sigs {
            let key = self
                .keys
                .get(usize::try_from(*i).map_err(|_| QuorumError::UnknownSigner)?)
                .ok_or(QuorumError::UnknownSigner)?;
            if !seen.insert(*i) {
                return Err(QuorumError::Duplicate);
            }
            key.verify_strict(&msg, &Signature::from_bytes(sig))
                .map_err(|_| QuorumError::BadSignature)?;
        }
        if seen.len() < self.threshold {
            return Err(QuorumError::NotEnough);
        }
        Ok(())
    }
}
```

### `src/bridge.rs`

```rust
//! The destination side: trusted roots from attestations, proof checking, execute-once minting.
use crate::merkle::{self, Hash, Proof};
use crate::msg::{Addr, Message, MsgError};
use crate::quorum::{Attestation, QuorumError, ValidatorSet};
use std::collections::{BTreeMap, BTreeSet};

#[derive(Debug, PartialEq, Eq)]
pub enum BridgeError {
    Quorum(QuorumError),
    /// A different root was already accepted for this height: the validators signed two histories.
    Equivocation,
    Malformed(MsgError),
    WrongSource,
    WrongDestination,
    UnknownHeight,
    BadProof,
    AlreadyExecuted,
    Paused,
    OverCap,
}

pub struct Inbox {
    /// This chain's id (messages must be addressed to it).
    pub chain: u32,
    /// The chain whose messages are accepted.
    pub source: u32,
    set: ValidatorSet,
    roots: BTreeMap<u64, (Hash, Hash)>,
    executed: BTreeSet<[u8; 32]>,
    minted: BTreeMap<Addr, u128>,
    total_minted: u128,
    paused: bool,
    cap: u128,
}

impl Inbox {
    /// `cap` is the most that may ever be minted in total (a circuit breaker).
    pub fn new(chain: u32, source: u32, set: ValidatorSet, cap: u128) -> Inbox {
        Inbox {
            chain,
            source,
            set,
            roots: BTreeMap::new(),
            executed: BTreeSet::new(),
            minted: BTreeMap::new(),
            total_minted: 0,
            paused: false,
            cap,
        }
    }
    pub fn pause(&mut self) {
        self.paused = true;
    }
    pub fn unpause(&mut self) {
        self.paused = false;
    }
    pub fn total_minted(&self) -> u128 {
        self.total_minted
    }
    pub fn balance(&self, a: &Addr) -> u128 {
        self.minted.get(a).copied().unwrap_or(0)
    }
    pub fn executed_count(&self) -> usize {
        self.executed.len()
    }
    pub fn has_executed(&self, id: &[u8; 32]) -> bool {
        self.executed.contains(id)
    }
    pub fn trusted_root(&self, height: u64) -> Option<Hash> {
        self.roots.get(&height).map(|(_, r)| *r)
    }

    /// Accepts the (hash, root) of a block if a quorum signed it. Repeating the same attestation
    /// is fine; a different root for a known height is an equivocation and is refused.
    pub fn submit_attestation(&mut self, a: &Attestation) -> Result<(), BridgeError> {
        self.set
            .verify(self.source, a)
            .map_err(BridgeError::Quorum)?;
        match self.roots.get(&a.height) {
            Some((h, r)) if *h == a.hash && *r == a.root => Ok(()),
            Some(_) => Err(BridgeError::Equivocation),
            None => {
                self.roots.insert(a.height, (a.hash, a.root));
                Ok(())
            }
        }
    }

    /// Executes one message: decode, route, find the root for `height`, check the proof, check the
    /// id was not used, check the cap, then mint. The id is recorded before anything else changes.
    pub fn deliver(
        &mut self,
        encoded: &[u8],
        height: u64,
        proof: &Proof,
    ) -> Result<Message, BridgeError> {
        if self.paused {
            return Err(BridgeError::Paused);
        }
        let m = Message::decode(encoded).map_err(BridgeError::Malformed)?;
        if m.src != self.source {
            return Err(BridgeError::WrongSource);
        }
        if m.dst != self.chain {
            return Err(BridgeError::WrongDestination);
        }
        let (_, root) = self.roots.get(&height).ok_or(BridgeError::UnknownHeight)?;
        if !merkle::verify(root, encoded, proof) {
            return Err(BridgeError::BadProof);
        }
        let id = m.id();
        if self.executed.contains(&id) {
            return Err(BridgeError::AlreadyExecuted);
        }
        let total = self
            .total_minted
            .checked_add(m.amount)
            .filter(|t| *t <= self.cap)
            .ok_or(BridgeError::OverCap)?;
        self.executed.insert(id);
        self.total_minted = total;
        *self.minted.entry(m.recipient).or_default() += m.amount;
        Ok(m)
    }
}
```

### `src/relayer.rs`

```rust
//! An untrusted relayer: collects attestations, carries proofs, retries, and is idempotent.
use crate::bridge::{BridgeError, Inbox};
use crate::chain::SourceChain;
use crate::merkle::{Hash, Proof};
use crate::quorum::{Attestation, Validator};

#[derive(Debug, PartialEq, Eq)]
pub enum LinkError {
    /// The call may or may not have been applied: the caller cannot tell.
    Transient,
    Rejected(BridgeError),
}

/// How a relayer reaches the destination.
pub trait Link {
    fn attest(&mut self, a: &Attestation) -> Result<(), LinkError>;
    fn deliver(&mut self, encoded: &[u8], height: u64, proof: &Proof) -> Result<(), LinkError>;
}

/// The reliable link: straight into an inbox.
impl Link for Inbox {
    fn attest(&mut self, a: &Attestation) -> Result<(), LinkError> {
        self.submit_attestation(a).map_err(LinkError::Rejected)
    }
    fn deliver(&mut self, encoded: &[u8], height: u64, proof: &Proof) -> Result<(), LinkError> {
        Inbox::deliver(self, encoded, height, proof)
            .map(|_| ())
            .map_err(LinkError::Rejected)
    }
}

#[derive(Debug, Default, PartialEq, Eq)]
pub struct Report {
    pub delivered: usize,
    /// Messages the inbox refused for a reason that retrying will not fix.
    pub rejected: usize,
    /// True if the run stopped early on a transient failure (call again).
    pub stopped: bool,
}

#[derive(Debug, PartialEq, Eq)]
pub enum RelayError {
    /// The block the relayer already processed is no longer in the source chain.
    FinalityViolation { height: u64 },
}

pub struct Relayer {
    cursor: u64,
    last: Option<(u64, Hash)>,
    /// The relayer's own list of poisoned messages (height, index), skipped from now on.
    pub poisoned: Vec<(u64, usize)>,
}

impl Default for Relayer {
    fn default() -> Self {
        Self::new()
    }
}

impl Relayer {
    pub fn new() -> Relayer {
        Relayer {
            cursor: 1,
            last: None,
            poisoned: vec![],
        }
    }
    pub fn cursor(&self) -> u64 {
        self.cursor
    }

    /// Processes every block the validators will attest to, in order, from the cursor.
    /// A transient failure stops the run with the cursor on the unfinished block; calling again
    /// repeats the block, which is safe because every step is idempotent at the inbox.
    pub fn run_once(
        &mut self,
        chain: &SourceChain,
        validators: &[Validator],
        link: &mut impl Link,
    ) -> Result<Report, RelayError> {
        let mut rep = Report::default();
        if let Some((h, hash)) = self.last
            && chain.block(h).map(|b| b.hash) != Some(hash)
        {
            return Err(RelayError::FinalityViolation { height: h });
        }
        while self.cursor <= chain.tip() {
            let h = self.cursor;
            let sigs: Vec<_> = validators
                .iter()
                .filter_map(|v| v.attest(chain, h))
                .collect();
            let Some(block) = chain.block(h) else { break };
            if sigs.is_empty() {
                break; // not final yet for anyone
            }
            let att = Attestation {
                height: h,
                hash: block.hash,
                root: block.root,
                sigs,
            };
            match link.attest(&att) {
                Ok(()) => {}
                Err(LinkError::Transient) => {
                    rep.stopped = true;
                    return Ok(rep);
                }
                Err(LinkError::Rejected(BridgeError::Quorum(_))) => break, // not enough signatures yet
                Err(LinkError::Rejected(_)) => {
                    rep.rejected += 1;
                    break;
                }
            }
            for i in 0..block.msgs.len() {
                if self.poisoned.contains(&(h, i)) {
                    continue;
                }
                let (enc, proof) = chain.proof_for(h, i).expect("index in range");
                match link.deliver(&enc, h, &proof) {
                    Ok(()) => rep.delivered += 1,
                    Err(LinkError::Rejected(BridgeError::AlreadyExecuted)) => {}
                    Err(LinkError::Transient) => {
                        rep.stopped = true;
                        return Ok(rep);
                    }
                    Err(LinkError::Rejected(_)) => {
                        self.poisoned.push((h, i));
                        rep.rejected += 1;
                    }
                }
            }
            self.last = Some((h, block.hash));
            self.cursor = h + 1;
        }
        Ok(rep)
    }
}
```

### `src/indexer.rs`

```rust
//! A reorganization-aware indexer: blocks stored by height with their parent, events derived from
//! them, rollback to the common ancestor, idempotent ingestion.
use crate::chain::{Block, SourceChain};
use crate::merkle::Hash;
use crate::msg::{Addr, Message};
use std::collections::BTreeMap;

#[derive(Debug, PartialEq, Eq)]
pub enum Outcome {
    /// The block extended the indexed chain.
    Appended,
    /// The same block was already indexed.
    Duplicate,
    /// The block's parent was not the indexed block below it; the indexer rolled back this many
    /// blocks and indexed the block on the new branch.
    Reorged { depth: u64 },
    /// The block is above the next expected height; fetch the missing ones first.
    Gap,
    /// The block conflicts with the indexed chain and its parent is unknown.
    Orphan,
}

#[derive(Default)]
pub struct Indexer {
    blocks: BTreeMap<u64, (Hash, Hash)>, // height -> (hash, parent)
    events: BTreeMap<(u64, usize), Message>,
}

impl Indexer {
    pub fn new(genesis: &Block) -> Indexer {
        let mut i = Indexer::default();
        i.blocks.insert(0, (genesis.hash, genesis.parent));
        i
    }
    pub fn tip(&self) -> u64 {
        *self.blocks.keys().next_back().expect("genesis")
    }
    pub fn hash_at(&self, height: u64) -> Option<Hash> {
        self.blocks.get(&height).map(|(h, _)| *h)
    }
    pub fn event_count(&self) -> usize {
        self.events.len()
    }

    fn roll_back_to(&mut self, keep: u64) {
        self.blocks.retain(|h, _| *h <= keep);
        self.events.retain(|(h, _), _| *h <= keep);
    }

    fn store(&mut self, b: &Block) {
        self.blocks.insert(b.height, (b.hash, b.parent));
        for (i, m) in b.msgs.iter().enumerate() {
            self.events.insert((b.height, i), m.clone());
        }
    }

    /// Ingests one block, given in any order and any number of times.
    pub fn ingest(&mut self, b: &Block) -> Outcome {
        if b.height == 0 {
            return if self.hash_at(0) == Some(b.hash) {
                Outcome::Duplicate
            } else {
                Outcome::Orphan
            };
        }
        if let Some(h) = self.hash_at(b.height)
            && h == b.hash
        {
            return Outcome::Duplicate;
        }
        if b.height > self.tip() + 1 {
            return Outcome::Gap;
        }
        if self.hash_at(b.height - 1) != Some(b.parent) {
            return Outcome::Orphan;
        }
        let existing = self.hash_at(b.height).is_some();
        let depth = self.tip() - (b.height - 1);
        if existing || depth > 0 {
            self.roll_back_to(b.height - 1);
            self.store(b);
            return Outcome::Reorged { depth };
        }
        self.store(b);
        Outcome::Appended
    }

    /// Brings the index in line with `chain`, walking back to the common ancestor if needed.
    pub fn sync(&mut self, chain: &SourceChain) {
        // find the highest indexed height whose hash still matches the chain
        let mut h = self.tip().min(chain.tip());
        while h > 0 && self.hash_at(h) != chain.block(h).map(|b| b.hash) {
            h -= 1;
        }
        self.roll_back_to(h);
        for height in h + 1..=chain.tip() {
            let b = chain.block(height).expect("in range");
            self.ingest(b);
        }
    }

    /// Events (height, index, message) that involve `a` as sender or recipient, up to `max_height`.
    pub fn events_for(&self, a: &Addr, max_height: u64) -> Vec<(u64, usize, &Message)> {
        self.events
            .iter()
            .filter(|((h, _), m)| *h <= max_height && (m.sender == *a || m.recipient == *a))
            .map(|((h, i), m)| (*h, *i, m))
            .collect()
    }

    /// The highest height with at least `depth` confirmations on the indexed chain.
    pub fn finalized_height(&self, depth: u64) -> u64 {
        self.tip().saturating_sub(depth)
    }
}
```

### `tests/common/mod.rs`

```rust
#![allow(dead_code)]
use relaylab::bridge::Inbox;
use relaylab::merkle::Proof;
use relaylab::msg::Addr;
use relaylab::quorum::{Attestation, Validator, ValidatorSet, threshold_for};
use relaylab::relayer::{Link, LinkError};

pub const SRC: u32 = 1;
pub const DST: u32 = 2;

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

pub fn addr(n: u8) -> Addr {
    [n; 20]
}

pub fn h(b: &[u8]) -> String {
    hex::encode(b)
}

pub fn hx<const N: usize>(s: &str) -> [u8; N] {
    hex::decode(s).unwrap().try_into().unwrap()
}

/// `n` validators with seeds `[i+1; 32]`, each requiring `depth` confirmations.
pub fn validators(n: u32, depth: u64) -> Vec<Validator> {
    (0..n)
        .map(|i| Validator::from_seed(i, [i as u8 + 1; 32], depth))
        .collect()
}

pub fn set_of(v: &[Validator]) -> ValidatorSet {
    let keys: Vec<[u8; 32]> = v.iter().map(Validator::public).collect();
    ValidatorSet::new(&keys, threshold_for(keys.len())).unwrap()
}

pub fn inbox(v: &[Validator], cap: u128) -> Inbox {
    Inbox::new(DST, SRC, set_of(v), cap)
}

/// A link that loses calls before they arrive, applies calls and then reports a failure, and
/// delivers some calls twice: the three things a network does.
pub struct Flaky {
    pub inner: Inbox,
    pub rng: Rng,
    pub lost: u32,
    pub ambiguous: u32,
    pub duplicated: u32,
}

impl Flaky {
    pub fn new(inner: Inbox, seed: u64) -> Flaky {
        Flaky {
            inner,
            rng: Rng(seed),
            lost: 0,
            ambiguous: 0,
            duplicated: 0,
        }
    }
    fn fate(&mut self) -> u64 {
        self.rng.below(100)
    }
}

impl Link for Flaky {
    fn attest(&mut self, a: &Attestation) -> Result<(), LinkError> {
        let f = self.fate();
        if f < 15 {
            self.lost += 1;
            return Err(LinkError::Transient);
        }
        let r = Link::attest(&mut self.inner, a);
        if f < 25 {
            self.ambiguous += 1;
            return Err(LinkError::Transient);
        }
        r
    }
    fn deliver(&mut self, e: &[u8], height: u64, p: &Proof) -> Result<(), LinkError> {
        let f = self.fate();
        if f < 20 {
            self.lost += 1;
            return Err(LinkError::Transient);
        }
        let r = Link::deliver(&mut self.inner, e, height, p);
        if f < 35 {
            self.ambiguous += 1;
            return Err(LinkError::Transient);
        }
        if f < 45 {
            self.duplicated += 1;
            let _ = Link::deliver(&mut self.inner, e, height, p);
        }
        r
    }
}
```

### `tests/merkle_msg_tests.rs`

```rust
mod common;
use common::*;
use relaylab::merkle::*;
use relaylab::msg::*;

fn leaves(n: usize) -> Vec<Vec<u8>> {
    (0..n).map(|i| format!("leaf-{i}").into_bytes()).collect()
}

#[test]
fn roots_match_python() {
    let abcde: Vec<Vec<u8>> = ["a", "b", "c", "d", "e"]
        .iter()
        .map(|s| s.as_bytes().to_vec())
        .collect();
    assert_eq!(
        h(&root(&[])),
        "f0c1e0b4cd1983b9c92909f8145cc102993e4c797489c0ba98639fd93056b82f"
    );
    assert_eq!(
        h(&root(&abcde[..1])),
        "a94fe96b1c38a1bdfb72939d6cd46bc31a31bfe1ad0d960804189a287b6558a0"
    );
    assert_eq!(
        h(&root(&abcde[..2])),
        "4c18b87ee623a3dfbcfefc3b002d3334c0554f545f9ef9dadec271daec0b399b"
    );
    assert_eq!(
        h(&root(&abcde[..3])),
        "7d36854b375536f2bb96cbc87f12825048cc9888d457bfa7a25e1b30df5f6da8"
    );
    assert_eq!(
        h(&root(&abcde)),
        "18949a0a0f9894ca630d9057e57e45b148a1d82c2a303251f3dd06a4bca92752"
    );
}

#[test]
fn proofs_match_python() {
    let abcde: Vec<Vec<u8>> = ["a", "b", "c", "d", "e"]
        .iter()
        .map(|s| s.as_bytes().to_vec())
        .collect();
    let p = prove(&abcde, 2).unwrap();
    let sib: Vec<String> = p.siblings.iter().map(|s| h(s)).collect();
    assert_eq!(
        sib,
        [
            "d070dc5b8da9aea7dc0f5ad4c29d89965200059c9a0ceca3abd5da2492dcb71d",
            "b137985ff484fb600db93107c77b0365c80d78f5b429ded0fd97361d077999eb",
            "2824a7ccda2caa720c85c9fba1e8b5b735eecfdb03878e4f8dfe6c3625030bc4"
        ]
    );
    // the odd leaf at the end of a level is promoted, so it has one sibling, not three
    let p4 = prove(&abcde, 4).unwrap();
    assert_eq!(p4.siblings.len(), 1);
    assert_eq!(
        h(&p4.siblings[0]),
        "33376a3bd63e9993708a84ddfe6c28ae58b83505dd1fed711bd924ec5a6239f0"
    );
}

#[test]
fn every_leaf_of_every_size_proves_and_nothing_else_does() {
    for n in 1..=33usize {
        let ls = leaves(n);
        let r = root(&ls);
        for i in 0..n {
            let p = prove(&ls, i).unwrap();
            assert!(verify(&r, &ls[i], &p), "n={n} i={i}");
            // a different leaf at the same place
            assert!(!verify(&r, b"other", &p));
            // the same leaf claimed at another index or in a tree of another size
            let mut q = p.clone();
            q.index = (p.index + 1) % n as u64;
            if n > 1 {
                assert!(!verify(&r, &ls[i], &q), "n={n} i={i} moved");
            }
            let mut q = p.clone();
            q.size += 1;
            assert!(!verify(&r, &ls[i], &q), "n={n} i={i} grown");
            // one sibling too many or too few
            let mut q = p.clone();
            q.siblings.push([0; 32]);
            assert!(!verify(&r, &ls[i], &q));
            if !p.siblings.is_empty() {
                let mut q = p.clone();
                q.siblings.pop();
                assert!(!verify(&r, &ls[i], &q));
                let mut q = p.clone();
                q.siblings[0][0] ^= 1;
                assert!(!verify(&r, &ls[i], &q));
            }
        }
        assert!(prove(&ls, n).is_none());
    }
}

#[test]
fn a_duplicated_last_leaf_changes_the_root() {
    // Bitcoin-style duplication would make these equal; promotion does not
    let abc: Vec<Vec<u8>> = vec![b"a".to_vec(), b"b".to_vec(), b"c".to_vec()];
    let mut abcc = abc.clone();
    abcc.push(b"c".to_vec());
    assert_ne!(root(&abc), root(&abcc));
    assert_eq!(
        h(&root(&abcc)),
        "4e8ae05ab0d47478b2dd8cf13b4e8526a0411652ddfbe788f7eb03346dc81521"
    );
}

#[test]
fn a_leaf_cannot_pose_as_a_node_or_the_reverse() {
    let l = leaves(2);
    let r = root(&l);
    // a 64-byte "leaf" made of the two child hashes is not the same as the tree over them
    let mut fake = Vec::new();
    fake.extend(leaf_hash(&l[0]));
    fake.extend(leaf_hash(&l[1]));
    assert_ne!(root(&[fake.clone()]), r);
    assert_ne!(leaf_hash(&fake), r);
    assert_ne!(leaf_hash(b""), empty_root());
}

#[test]
fn proofs_for_empty_and_degenerate_shapes() {
    let r = root(&leaves(1));
    let p = prove(&leaves(1), 0).unwrap();
    assert!(p.siblings.is_empty());
    assert!(verify(&r, &leaves(1)[0], &p));
    let bad = Proof {
        index: 0,
        size: 0,
        siblings: vec![],
    };
    assert!(!verify(&empty_root(), b"x", &bad));
    let bad = Proof {
        index: 3,
        size: 3,
        siblings: vec![],
    };
    assert!(!verify(&r, b"x", &bad));
}

fn msg() -> Message {
    Message {
        src: 1,
        dst: 2,
        nonce: 7,
        sender: addr(0x11),
        recipient: addr(0x22),
        amount: 1_000_000,
        payload: b"hi".to_vec(),
    }
}

#[test]
fn message_encoding_and_id_match_python() {
    let m = msg();
    assert_eq!(
        h(&m.encode()),
        "0000000100000002000000000000000711111111111111111111111111111111111111112222222222222222222222222222222222222222000000000000000000000000000f424000026869"
    );
    assert_eq!(
        h(&m.id()),
        "e7a114f5a0e12209727555ff85bb8b37d1ca8f18646644e316fa0552b4a167be"
    );
    assert_eq!(m.encode().len(), HEADER_LEN + 2);
    assert_eq!(Message::decode(&m.encode()), Ok(m));
}

#[test]
fn every_field_changes_the_id() {
    let m = msg();
    let variants = [
        Message { src: 9, ..msg() },
        Message { dst: 9, ..msg() },
        Message { nonce: 8, ..msg() },
        Message {
            sender: addr(0x12),
            ..msg()
        },
        Message {
            recipient: addr(0x23),
            ..msg()
        },
        Message {
            amount: 1_000_001,
            ..msg()
        },
        Message {
            payload: b"ho".to_vec(),
            ..msg()
        },
        Message {
            payload: vec![],
            ..msg()
        },
    ];
    for v in &variants {
        assert_ne!(v.id(), m.id());
    }
    // the id is not the plain hash of the encoding
    assert_ne!(
        m.id().to_vec(),
        relaylab::merkle::leaf_hash(&m.encode()).to_vec()
    );
}

#[test]
fn decoding_is_strict() {
    let enc = msg().encode();
    assert_eq!(
        Message::decode(&enc[..HEADER_LEN - 1]),
        Err(MsgError::Short)
    );
    assert_eq!(Message::decode(&enc[..enc.len() - 1]), Err(MsgError::Short));
    let mut t = enc.clone();
    t.push(0);
    assert_eq!(Message::decode(&t), Err(MsgError::Trailing));
    let mut big = Message {
        payload: vec![1; MAX_PAYLOAD],
        ..msg()
    }
    .encode();
    assert!(Message::decode(&big).is_ok());
    big[72] = 0x01;
    big[73] = 0x01; // claims 257 bytes
    big.push(1);
    assert_eq!(Message::decode(&big), Err(MsgError::PayloadTooLong));
    assert_eq!(Message::decode(&[]), Err(MsgError::Short));
}

#[test]
fn random_messages_round_trip() {
    let mut r = Rng(5);
    for _ in 0..500 {
        let n = r.below(MAX_PAYLOAD as u64 + 1) as usize;
        let m = Message {
            src: r.next_u64() as u32,
            dst: r.next_u64() as u32,
            nonce: r.next_u64(),
            sender: r.bytes(20).try_into().unwrap(),
            recipient: r.bytes(20).try_into().unwrap(),
            amount: (u128::from(r.next_u64()) << 64) | u128::from(r.next_u64()),
            payload: r.bytes(n),
        };
        assert_eq!(Message::decode(&m.encode()), Ok(m));
    }
    for _ in 0..4000 {
        let n = r.below(120) as usize;
        let _ = Message::decode(&r.bytes(n));
    }
}

#[test]
fn the_root_commits_to_the_leaf_count() {
    // without the count, the proof of the first leaf of [a,b,c] also "proves" it in a 4-leaf tree
    let ls = leaves(3);
    let r = root(&ls);
    let p = prove(&ls, 0).unwrap();
    assert!(verify(&r, &ls[0], &p));
    for size in [1, 2, 4, 5, 8] {
        assert!(
            !verify(&r, &ls[0], &Proof { size, ..p.clone() }),
            "size {size}"
        );
    }
    assert_eq!(commit(0, &[0; 32]).len(), 32);
    assert_ne!(commit(1, &[0; 32]), commit(2, &[0; 32]));
}
```

### `tests/chain_quorum_tests.rs`

```rust
mod common;
use common::*;
use relaylab::chain::*;
use relaylab::merkle;
use relaylab::quorum::*;

fn chain() -> SourceChain {
    SourceChain::new(SRC, &[(addr(1), 1_000), (addr(2), 500)])
}

#[test]
fn header_hash_matches_python() {
    assert_eq!(
        h(&header_hash(1, 5, &[0x33; 32], &[0x44; 32])),
        "2708eac1111880a8a29b9df35fcfe6ef8ed2d6462b31cf7484abda797d73950f"
    );
}

#[test]
fn sending_locks_funds_and_sealing_commits_them() {
    let mut c = chain();
    assert_eq!(c.tip(), 0);
    let m = c.send(addr(1), addr(9), DST, 300, b"x".to_vec()).unwrap();
    assert_eq!(
        (c.balance(&addr(1)), c.escrow(), c.pending_len()),
        (700, 300, 1)
    );
    assert_eq!(m.nonce, 0);
    let m2 = c.send(addr(1), addr(9), DST, 100, vec![]).unwrap();
    assert_eq!(m2.nonce, 1);
    let b = c.seal().clone();
    assert_eq!(b.height, 1);
    assert_eq!(b.parent, c.block(0).unwrap().hash);
    assert_eq!(b.msgs, vec![m, m2]);
    assert_eq!(c.pending_len(), 0);
    assert_eq!(b.hash, header_hash(SRC, 1, &b.parent, &b.root));
    assert_eq!(c.total_supply(), c.initial_supply());
    // an empty block commits to the empty root
    let e = c.seal().clone();
    assert_eq!(e.root, merkle::empty_root());
    assert_eq!(e.parent, b.hash);
}

#[test]
fn sends_are_refused_for_the_right_reason() {
    let mut c = chain();
    assert_eq!(
        c.send(addr(1), addr(9), DST, 0, vec![]),
        Err(SendError::ZeroAmount)
    );
    assert_eq!(
        c.send(addr(1), addr(9), DST, 1_001, vec![]),
        Err(SendError::Insufficient)
    );
    assert_eq!(
        c.send(addr(1), addr(9), DST, 1_000, vec![]).map(|_| ()),
        Ok(())
    );
    assert_eq!(
        c.send(addr(1), addr(9), DST, 1, vec![]),
        Err(SendError::Insufficient)
    );
    assert_eq!(
        c.send(addr(2), addr(9), DST, 1, vec![0; 257]),
        Err(SendError::PayloadTooLong)
    );
    assert_eq!(
        c.send(addr(2), addr(9), DST, 1, vec![0; 256]).map(|_| ()),
        Ok(())
    );
    assert_eq!(
        c.send(addr(77), addr(9), DST, 1, vec![]),
        Err(SendError::Insufficient)
    );
    assert_eq!(c.total_supply(), c.initial_supply());
}

#[test]
fn proofs_from_the_chain_verify_against_the_block_root() {
    let mut c = chain();
    for i in 0..5u8 {
        c.send(addr(1), addr(9), DST, 10 + u128::from(i), vec![i])
            .unwrap();
    }
    c.seal();
    let root = c.block(1).unwrap().root;
    for i in 0..5 {
        let (enc, p) = c.proof_for(1, i).unwrap();
        assert!(merkle::verify(&root, &enc, &p));
    }
    assert!(c.proof_for(1, 5).is_none());
    assert!(c.proof_for(2, 0).is_none());
    assert!(c.proof_for(0, 0).is_none());
}

#[test]
fn a_reorg_refunds_the_dropped_messages() {
    let mut c = chain();
    c.send(addr(1), addr(9), DST, 100, vec![]).unwrap();
    c.seal();
    c.send(addr(2), addr(9), DST, 50, vec![]).unwrap();
    c.seal();
    c.seal();
    assert_eq!(c.escrow(), 150);
    assert_eq!(c.reorg(2), 2);
    assert_eq!(c.tip(), 1);
    assert_eq!(
        (c.escrow(), c.balance(&addr(1)), c.balance(&addr(2))),
        (100, 900, 500)
    );
    assert_eq!(c.total_supply(), c.initial_supply());
    assert_eq!(c.escrowed_in_blocks(), c.escrow());
    // the genesis block is never dropped
    assert_eq!(c.reorg(99), 1);
    assert_eq!(c.tip(), 0);
    assert_eq!((c.escrow(), c.balance(&addr(1))), (0, 1_000));
}

#[test]
fn the_ed25519_vector_matches_python() {
    let v = Validator::from_seed(0, [7; 32], 0);
    assert_eq!(
        h(&v.public()),
        "ea4a6c63e29c520abef5507b132ec5f9954776aebebe7b92421eea691446d22c"
    );
    // sign through a chain whose block hash and root we control is not possible, so check the
    // signed bytes and verify the Python signature with the set
    let sig: [u8; 64] = hx(
        "215ff688a694e9be84663c7165500b44d59dbbb70d969ab808941da4cfa1cd58630b9b3423ee232af52f446222109c23eaed1f14a6669ab9aeac2a5781be7602",
    );
    let set = ValidatorSet::new(&[v.public()], 1).unwrap();
    let a = Attestation {
        height: 5,
        hash: [0x55; 32],
        root: [0x66; 32],
        sigs: vec![(0, sig)],
    };
    assert_eq!(set.verify(1, &a), Ok(()));
    assert_eq!(
        set.verify(2, &a),
        Err(QuorumError::BadSignature),
        "the chain id is signed"
    );
    let mut b = a.clone();
    b.height = 6;
    assert_eq!(set.verify(1, &b), Err(QuorumError::BadSignature));
    let mut b = a.clone();
    b.root[0] ^= 1;
    assert_eq!(set.verify(1, &b), Err(QuorumError::BadSignature));
    let mut b = a.clone();
    b.hash[0] ^= 1;
    assert_eq!(set.verify(1, &b), Err(QuorumError::BadSignature));
    assert_eq!(
        &attest_bytes(1, 5, &[0x55; 32], &[0x66; 32])[..6],
        b"att-v1"
    );
}

#[test]
fn thresholds_must_exceed_two_thirds() {
    assert_eq!(
        [1, 2, 3, 4, 5, 6, 7, 9].map(threshold_for),
        [1, 2, 3, 3, 4, 5, 5, 7]
    );
    let v = validators(4, 0);
    let keys: Vec<[u8; 32]> = v.iter().map(Validator::public).collect();
    assert_eq!(
        ValidatorSet::new(&keys, 2).err(),
        Some(QuorumError::WeakThreshold)
    );
    assert!(ValidatorSet::new(&keys, 3).is_ok());
    assert!(ValidatorSet::new(&keys, 4).is_ok());
    assert_eq!(
        ValidatorSet::new(&keys, 5).err(),
        Some(QuorumError::WeakThreshold)
    );
    assert_eq!(
        ValidatorSet::new(&[], 1).err(),
        Some(QuorumError::WeakThreshold)
    );
    assert_eq!(
        ValidatorSet::new(&[[2; 32]], 1).err(),
        Some(QuorumError::UnknownSigner)
    );
    let s = ValidatorSet::new(&keys, 3).unwrap();
    assert_eq!((s.len(), s.threshold(), s.is_empty()), (4, 3, false));
}

fn attested(c: &SourceChain, v: &[Validator], height: u64) -> Attestation {
    let b = c.block(height).unwrap();
    Attestation {
        height,
        hash: b.hash,
        root: b.root,
        sigs: v.iter().filter_map(|x| x.attest(c, height)).collect(),
    }
}

#[test]
fn quorum_checks_each_rule() {
    let v = validators(4, 0);
    let set = set_of(&v);
    let mut c = chain();
    c.send(addr(1), addr(9), DST, 5, vec![]).unwrap();
    c.seal();
    let a = attested(&c, &v, 1);
    assert_eq!(a.sigs.len(), 4);
    assert_eq!(set.verify(SRC, &a), Ok(()));
    let mut three = a.clone();
    three.sigs.truncate(3);
    assert_eq!(set.verify(SRC, &three), Ok(()));
    let mut two = a.clone();
    two.sigs.truncate(2);
    assert_eq!(set.verify(SRC, &two), Err(QuorumError::NotEnough));
    // the same signer three times is one signer
    let mut dup = a.clone();
    dup.sigs = vec![a.sigs[0], a.sigs[0], a.sigs[0]];
    assert_eq!(set.verify(SRC, &dup), Err(QuorumError::Duplicate));
    let mut unknown = a.clone();
    unknown.sigs[0].0 = 9;
    assert_eq!(set.verify(SRC, &unknown), Err(QuorumError::UnknownSigner));
    let mut bad = a.clone();
    bad.sigs[1].1[0] ^= 1;
    assert_eq!(set.verify(SRC, &bad), Err(QuorumError::BadSignature));
    // a signature by validator 1 presented as validator 0
    let mut swapped = a.clone();
    swapped.sigs[0].1 = a.sigs[1].1;
    assert_eq!(set.verify(SRC, &swapped), Err(QuorumError::BadSignature));
    // attestations by a different set of keys
    let other = validators(4, 0)
        .into_iter()
        .map(|x| Validator::from_seed(x.index, [x.index as u8 + 50; 32], 0))
        .collect::<Vec<_>>();
    assert_eq!(
        set.verify(SRC, &attested(&c, &other, 1)),
        Err(QuorumError::BadSignature)
    );
}

#[test]
fn a_validator_signs_only_confirmed_blocks_of_its_own_chain_view() {
    let v = Validator::from_seed(0, [1; 32], 2);
    let mut c = chain();
    c.seal();
    c.seal();
    assert!(
        v.attest(&c, 1).is_none(),
        "tip is 2, block 1 has 1 confirmation"
    );
    c.seal();
    assert!(v.attest(&c, 1).is_some());
    assert!(v.attest(&c, 2).is_none());
    assert!(v.attest(&c, 9).is_none());
    assert!(
        v.attest(&c, u64::MAX).is_none(),
        "no overflow on the height check"
    );
    let zero = Validator::from_seed(0, [1; 32], 0);
    assert!(zero.attest(&c, 3).is_some());
    assert!(zero.attest(&c, 4).is_none());
}
```

### `tests/bridge_tests.rs`

```rust
mod common;
use common::*;
use relaylab::bridge::*;
use relaylab::chain::SourceChain;
use relaylab::merkle::Proof;
use relaylab::msg::Message;
use relaylab::quorum::{Attestation, Validator};

struct Env {
    chain: SourceChain,
    vals: Vec<Validator>,
    inbox: Inbox,
}

fn env() -> Env {
    let vals = validators(4, 0);
    let inbox = inbox(&vals, 10_000);
    let mut chain = SourceChain::new(SRC, &[(addr(1), 1_000), (addr(2), 1_000)]);
    chain
        .send(addr(1), addr(9), DST, 100, b"a".to_vec())
        .unwrap();
    chain.send(addr(2), addr(8), DST, 200, vec![]).unwrap();
    chain.send(addr(1), addr(9), DST, 50, vec![]).unwrap();
    chain.seal();
    Env { chain, vals, inbox }
}

fn att(e: &Env, height: u64) -> Attestation {
    let b = e.chain.block(height).unwrap();
    Attestation {
        height,
        hash: b.hash,
        root: b.root,
        sigs: e
            .vals
            .iter()
            .filter_map(|v| v.attest(&e.chain, height))
            .collect(),
    }
}

#[test]
fn a_message_is_minted_once_with_a_valid_proof() {
    let mut e = env();
    let a = att(&e, 1);
    e.inbox.submit_attestation(&a).unwrap();
    let (enc, p) = e.chain.proof_for(1, 1).unwrap();
    let m = e.inbox.deliver(&enc, 1, &p).unwrap();
    assert_eq!((m.amount, m.recipient), (200, addr(8)));
    assert_eq!(
        (
            e.inbox.balance(&addr(8)),
            e.inbox.total_minted(),
            e.inbox.executed_count()
        ),
        (200, 200, 1)
    );
    assert!(e.inbox.has_executed(&m.id()));
    // a second delivery is refused and changes nothing
    assert_eq!(
        e.inbox.deliver(&enc, 1, &p).err(),
        Some(BridgeError::AlreadyExecuted)
    );
    assert_eq!(
        (e.inbox.balance(&addr(8)), e.inbox.total_minted()),
        (200, 200)
    );
    // the other two still go through, in any order
    for i in [2, 0] {
        let (enc, p) = e.chain.proof_for(1, i).unwrap();
        e.inbox.deliver(&enc, 1, &p).unwrap();
    }
    assert_eq!(
        (e.inbox.balance(&addr(9)), e.inbox.total_minted()),
        (150, 350)
    );
    assert_eq!(e.inbox.total_minted(), e.chain.escrow());
}

#[test]
fn nothing_is_trusted_before_an_attestation() {
    let mut e = env();
    let (enc, p) = e.chain.proof_for(1, 0).unwrap();
    assert_eq!(
        e.inbox.deliver(&enc, 1, &p).err(),
        Some(BridgeError::UnknownHeight)
    );
    assert_eq!(e.inbox.trusted_root(1), None);
    e.inbox.submit_attestation(&att(&e, 1)).unwrap();
    assert_eq!(
        e.inbox.trusted_root(1),
        Some(e.chain.block(1).unwrap().root)
    );
    assert_eq!(
        e.inbox.deliver(&enc, 2, &p).err(),
        Some(BridgeError::UnknownHeight)
    );
}

#[test]
fn a_forged_message_or_proof_is_refused() {
    let mut e = env();
    e.inbox.submit_attestation(&att(&e, 1)).unwrap();
    let (enc, p) = e.chain.proof_for(1, 0).unwrap();
    let mut m = Message::decode(&enc).unwrap();
    m.amount = 9_000;
    assert_eq!(
        e.inbox.deliver(&m.encode(), 1, &p).err(),
        Some(BridgeError::BadProof)
    );
    m = Message::decode(&enc).unwrap();
    m.recipient = addr(66);
    assert_eq!(
        e.inbox.deliver(&m.encode(), 1, &p).err(),
        Some(BridgeError::BadProof)
    );
    // a proof for another leaf
    let (_, p2) = e.chain.proof_for(1, 1).unwrap();
    assert_eq!(
        e.inbox.deliver(&enc, 1, &p2).err(),
        Some(BridgeError::BadProof)
    );
    let bad = Proof {
        siblings: vec![],
        ..p.clone()
    };
    assert_eq!(
        e.inbox.deliver(&enc, 1, &bad).err(),
        Some(BridgeError::BadProof)
    );
    assert_eq!(e.inbox.total_minted(), 0);
}

#[test]
fn malformed_and_misrouted_messages_are_refused_before_the_proof_matters() {
    let mut e = env();
    e.inbox.submit_attestation(&att(&e, 1)).unwrap();
    let (enc, p) = e.chain.proof_for(1, 0).unwrap();
    assert!(matches!(
        e.inbox.deliver(&enc[..10], 1, &p),
        Err(BridgeError::Malformed(_))
    ));
    let mut m = Message::decode(&enc).unwrap();
    m.dst = 77;
    assert_eq!(
        e.inbox.deliver(&m.encode(), 1, &p).err(),
        Some(BridgeError::WrongDestination)
    );
    m = Message::decode(&enc).unwrap();
    m.src = 77;
    assert_eq!(
        e.inbox.deliver(&m.encode(), 1, &p).err(),
        Some(BridgeError::WrongSource)
    );
}

#[test]
fn a_message_for_another_chain_pair_is_not_replayable_here() {
    // the same bytes minus the routing would hash to the same leaf in another bridge; the routing is inside the message
    let mut e = env();
    let mut other = SourceChain::new(SRC, &[(addr(1), 1_000)]);
    other.send(addr(1), addr(9), 3, 100, vec![]).unwrap();
    other.seal();
    let a = {
        let b = other.block(1).unwrap();
        Attestation {
            height: 1,
            hash: b.hash,
            root: b.root,
            sigs: e.vals.iter().filter_map(|v| v.attest(&other, 1)).collect(),
        }
    };
    // the quorum signs this block too (a validator set serves many bridges), but its message is for chain 3
    assert!(e.inbox.submit_attestation(&a).is_ok());
    let (enc, p) = other.proof_for(1, 0).unwrap();
    assert_eq!(
        e.inbox.deliver(&enc, 1, &p).err(),
        Some(BridgeError::WrongDestination)
    );
}

#[test]
fn attestations_need_a_quorum_and_cannot_conflict() {
    let mut e = env();
    let mut a = att(&e, 1);
    a.sigs.truncate(2);
    assert_eq!(
        e.inbox.submit_attestation(&a),
        Err(BridgeError::Quorum(
            relaylab::quorum::QuorumError::NotEnough
        ))
    );
    assert_eq!(e.inbox.trusted_root(1), None);
    let a = att(&e, 1);
    assert_eq!(e.inbox.submit_attestation(&a), Ok(()));
    assert_eq!(
        e.inbox.submit_attestation(&a),
        Ok(()),
        "repeating is harmless"
    );
    // the same validators sign a different history for height 1
    let mut fork = SourceChain::new(SRC, &[(addr(1), 1_000)]);
    fork.send(addr(1), addr(5), DST, 999, vec![]).unwrap();
    fork.seal();
    let b = fork.block(1).unwrap();
    let fa = Attestation {
        height: 1,
        hash: b.hash,
        root: b.root,
        sigs: e.vals.iter().filter_map(|v| v.attest(&fork, 1)).collect(),
    };
    assert_eq!(
        e.inbox.submit_attestation(&fa),
        Err(BridgeError::Equivocation)
    );
    assert_eq!(
        e.inbox.trusted_root(1),
        Some(e.chain.block(1).unwrap().root)
    );
}

#[test]
fn pause_and_cap_stop_minting() {
    let mut e = env();
    e.inbox.submit_attestation(&att(&e, 1)).unwrap();
    e.inbox.pause();
    let (enc, p) = e.chain.proof_for(1, 0).unwrap();
    assert_eq!(
        e.inbox.deliver(&enc, 1, &p).err(),
        Some(BridgeError::Paused)
    );
    e.inbox.unpause();
    assert!(e.inbox.deliver(&enc, 1, &p).is_ok());
    // a cap of exactly the total lets everything through; one less refuses the last message
    let mut e2 = env();
    e2.inbox = inbox(&e2.vals, 349);
    e2.inbox.submit_attestation(&att(&e2, 1)).unwrap();
    let mut results = vec![];
    for i in 0..3 {
        let (enc, p) = e2.chain.proof_for(1, i).unwrap();
        results.push(e2.inbox.deliver(&enc, 1, &p).err());
    }
    assert_eq!(results, vec![None, None, Some(BridgeError::OverCap)]);
    assert_eq!(e2.inbox.total_minted(), 300);
    let mut e3 = env();
    e3.inbox = inbox(&e3.vals, 350);
    e3.inbox.submit_attestation(&att(&e3, 1)).unwrap();
    for i in 0..3 {
        let (enc, p) = e3.chain.proof_for(1, i).unwrap();
        assert!(e3.inbox.deliver(&enc, 1, &p).is_ok());
    }
    assert_eq!(e3.inbox.total_minted(), 350);
}

#[test]
fn huge_amounts_are_compared_without_overflow() {
    let vals = validators(4, 0);
    let big = u128::MAX - 10;
    let mut inb = inbox(&vals, big);
    let mut c = SourceChain::new(SRC, &[(addr(1), big), (addr(2), 10)]);
    c.send(addr(1), addr(9), DST, big, vec![]).unwrap();
    c.send(addr(2), addr(9), DST, 1, vec![]).unwrap();
    c.seal();
    let b = c.block(1).unwrap();
    let a = Attestation {
        height: 1,
        hash: b.hash,
        root: b.root,
        sigs: vals.iter().filter_map(|v| v.attest(&c, 1)).collect(),
    };
    inb.submit_attestation(&a).unwrap();
    let (e1, p1) = c.proof_for(1, 1).unwrap();
    let (e0, p0) = c.proof_for(1, 0).unwrap();
    assert!(inb.deliver(&e0, 1, &p0).is_ok());
    assert_eq!(inb.deliver(&e1, 1, &p1).err(), Some(BridgeError::OverCap));
    assert_eq!(inb.total_minted(), big);
    // with no cap at all the sum would still not wrap: the addition is checked
    let mut inb = inbox(&vals, u128::MAX);
    inb.submit_attestation(&a).unwrap();
    assert!(inb.deliver(&e0, 1, &p0).is_ok());
    assert!(inb.deliver(&e1, 1, &p1).is_ok());
    assert_eq!(inb.total_minted(), big + 1);
}
```

### `tests/relayer_tests.rs`

```rust
mod common;
use common::*;
use relaylab::bridge::{BridgeError, Inbox};
use relaylab::chain::SourceChain;
use relaylab::relayer::*;

fn setup(depth: u64) -> (SourceChain, Vec<relaylab::quorum::Validator>, Inbox) {
    let vals = validators(4, depth);
    let inb = inbox(&vals, u128::MAX);
    (
        SourceChain::new(SRC, &[(addr(1), 10_000), (addr(2), 10_000)]),
        vals,
        inb,
    )
}

#[test]
fn the_relayer_delivers_final_blocks_in_order() {
    let (mut c, v, mut inb) = setup(2);
    let mut r = Relayer::new();
    for i in 0..3u8 {
        c.send(addr(1), addr(9), DST, 10 + u128::from(i), vec![])
            .unwrap();
        c.seal();
    }
    // tip 3 and depth 2: only block 1 is final
    let rep = r.run_once(&c, &v, &mut inb).unwrap();
    assert_eq!((rep.delivered, rep.rejected, rep.stopped), (1, 0, false));
    assert_eq!((r.cursor(), inb.total_minted()), (2, 10));
    // nothing new: nothing happens
    assert_eq!(r.run_once(&c, &v, &mut inb).unwrap(), Report::default());
    c.seal();
    c.seal();
    let rep = r.run_once(&c, &v, &mut inb).unwrap();
    assert_eq!(rep.delivered, 2);
    assert_eq!(r.cursor(), 4);
    assert_eq!(inb.total_minted(), 10 + 11 + 12);
    assert_eq!(inb.total_minted(), c.escrow());
}

#[test]
fn rerunning_a_relayer_from_scratch_is_harmless() {
    let (mut c, v, mut inb) = setup(0);
    for _ in 0..3 {
        c.send(addr(1), addr(9), DST, 7, vec![]).unwrap();
        c.seal();
    }
    Relayer::new().run_once(&c, &v, &mut inb).unwrap();
    assert_eq!(inb.total_minted(), 21);
    // a second relayer, started later with no memory, repeats everything; the inbox says "already executed"
    let rep = Relayer::new().run_once(&c, &v, &mut inb).unwrap();
    assert_eq!((rep.delivered, rep.rejected), (0, 0));
    assert_eq!(inb.total_minted(), 21);
    assert_eq!(inb.executed_count(), 3);
}

#[test]
fn a_flaky_network_never_loses_or_repeats_a_message() {
    for seed in 1..=12u64 {
        let (mut c, v, inb) = setup(2);
        let mut link = Flaky::new(inb, seed * 7919);
        let mut r = Relayer::new();
        let mut rng = Rng(seed);
        let mut sent = 0u128;
        let mut calls = 0;
        for _ in 0..40 {
            for _ in 0..rng.below(4) {
                let amt = 1 + u128::from(rng.below(50));
                if c.send(
                    addr(1 + (rng.below(2) as u8)),
                    addr(9 + rng.below(3) as u8),
                    DST,
                    amt,
                    vec![],
                )
                .is_ok()
                {
                    sent += amt;
                }
            }
            c.seal();
            let _ = r.run_once(&c, &v, &mut link).unwrap();
            calls += 1;
        }
        // let the network settle: keep calling until a run completes without stopping and nothing moves
        c.seal();
        c.seal();
        let mut quiet = 0;
        while quiet < 3 && calls < 5_000 {
            let rep = r.run_once(&c, &v, &mut link).unwrap();
            quiet = if !rep.stopped && rep.delivered == 0 {
                quiet + 1
            } else {
                0
            };
            calls += 1;
        }
        assert!(
            link.lost > 0 && link.ambiguous > 0 && link.duplicated > 0,
            "seed {seed}: the network was not hostile"
        );
        // everything in a final block (tip - 2 or lower) arrived, exactly once
        let final_h = c.tip() - 2;
        let expected: u128 = (1..=final_h)
            .flat_map(|h| c.block(h).unwrap().msgs.iter().map(|m| m.amount))
            .sum();
        let count: usize = (1..=final_h).map(|h| c.block(h).unwrap().msgs.len()).sum();
        assert_eq!(link.inner.total_minted(), expected, "seed {seed}");
        assert_eq!(link.inner.executed_count(), count, "seed {seed}");
        assert!(expected <= sent);
        assert_eq!(c.total_supply(), c.initial_supply());
    }
}

#[test]
fn a_poisoned_message_does_not_block_the_queue() {
    let (mut c, v, mut inb) = setup(0);
    c.send(addr(1), addr(9), 77, 5, vec![]).unwrap(); // wrong destination chain
    c.send(addr(1), addr(9), DST, 6, vec![]).unwrap();
    c.seal();
    let mut r = Relayer::new();
    let rep = r.run_once(&c, &v, &mut inb).unwrap();
    assert_eq!((rep.delivered, rep.rejected), (1, 1));
    assert_eq!(r.poisoned, vec![(1, 0)]);
    assert_eq!(inb.total_minted(), 6);
    // the message stays in escrow on the source: stuck, visible, and not delivered again
    assert_eq!(c.escrow(), 11);
    c.seal();
    let rep = r.run_once(&c, &v, &mut inb).unwrap();
    assert_eq!(rep, Report::default());
}

#[test]
fn a_reorg_below_the_cursor_stops_the_relayer() {
    let (mut c, v, mut inb) = setup(0);
    c.send(addr(1), addr(9), DST, 5, vec![]).unwrap();
    c.seal();
    let mut r = Relayer::new();
    r.run_once(&c, &v, &mut inb).unwrap();
    c.reorg(1);
    c.send(addr(1), addr(9), DST, 6, vec![]).unwrap();
    c.seal();
    assert_eq!(
        r.run_once(&c, &v, &mut inb),
        Err(RelayError::FinalityViolation { height: 1 })
    );
}

#[test]
fn shallow_confirmations_lose_money_and_deep_ones_do_not() {
    // depth 0: the validators sign block 1 at once, the message is minted, then block 1 is reorganized away
    let (mut c, v, mut inb) = setup(0);
    c.send(addr(1), addr(9), DST, 500, vec![]).unwrap();
    c.seal();
    Relayer::new().run_once(&c, &v, &mut inb).unwrap();
    assert_eq!(inb.total_minted(), 500);
    c.reorg(1);
    assert_eq!(c.escrow(), 0, "the source gave the funds back");
    assert!(
        inb.total_minted() > c.escrow(),
        "the destination still holds 500 minted against nothing"
    );
    // depth 2: a one-block reorg happens before anyone signs
    let (mut c, v, mut inb) = setup(2);
    c.send(addr(1), addr(9), DST, 500, vec![]).unwrap();
    c.seal();
    let mut r = Relayer::new();
    r.run_once(&c, &v, &mut inb).unwrap();
    assert_eq!(inb.total_minted(), 0);
    c.reorg(1);
    c.seal();
    c.seal();
    c.seal();
    r.run_once(&c, &v, &mut inb).unwrap();
    assert_eq!(inb.total_minted(), 0);
    assert_eq!(c.escrow(), 0);
}

#[test]
fn a_relayer_cannot_deliver_what_validators_did_not_sign() {
    let (mut c, v, mut inb) = setup(0);
    c.send(addr(1), addr(9), DST, 5, vec![]).unwrap();
    c.seal();
    // only two of four validators are online: below the threshold of three
    let mut r = Relayer::new();
    let rep = r.run_once(&c, &v[..2], &mut inb).unwrap();
    assert_eq!((rep.delivered, rep.rejected, r.cursor()), (0, 0, 1));
    assert_eq!(inb.total_minted(), 0);
    let rep = r.run_once(&c, &v[..3], &mut inb).unwrap();
    assert_eq!((rep.delivered, r.cursor()), (1, 2));
}

#[test]
fn the_inbox_is_a_link_and_reports_rejections() {
    let (mut c, v, mut inb) = setup(0);
    c.send(addr(1), addr(9), DST, 5, vec![]).unwrap();
    c.seal();
    let b = c.block(1).unwrap();
    let a = relaylab::quorum::Attestation {
        height: 1,
        hash: b.hash,
        root: b.root,
        sigs: v.iter().filter_map(|x| x.attest(&c, 1)).collect(),
    };
    assert_eq!(Link::attest(&mut inb, &a), Ok(()));
    let (e, p) = c.proof_for(1, 0).unwrap();
    assert_eq!(Link::deliver(&mut inb, &e, 1, &p), Ok(()));
    assert_eq!(
        Link::deliver(&mut inb, &e, 1, &p),
        Err(LinkError::Rejected(BridgeError::AlreadyExecuted))
    );
}

#[test]
fn conservation_holds_over_random_traffic_with_shallow_reorgs() {
    let mut rng = Rng(2024);
    let (mut c, v, inb) = setup(3);
    let mut link = Flaky::new(inb, 99);
    let mut r = Relayer::new();
    for round in 0..60 {
        for _ in 0..rng.below(4) {
            let _ = c.send(
                addr(1 + rng.below(2) as u8),
                addr(9),
                DST,
                1 + u128::from(rng.below(40)),
                vec![],
            );
        }
        c.seal();
        if rng.below(5) == 0 {
            c.reorg(1 + rng.below(2)); // at most 2 deep, validators wait for 3
        }
        let _ = r.run_once(&c, &v, &mut link).unwrap();
        assert_eq!(c.total_supply(), c.initial_supply(), "round {round}");
        assert!(
            link.inner.total_minted() <= c.escrow(),
            "round {round}: minted more than is escrowed"
        );
    }
    for _ in 0..4 {
        c.seal();
    }
    for _ in 0..400 {
        let _ = r.run_once(&c, &v, &mut link).unwrap();
    }
    let fin = c.tip() - 3;
    let expected: u128 = (1..=fin)
        .flat_map(|h| c.block(h).unwrap().msgs.iter().map(|m| m.amount))
        .sum();
    assert_eq!(link.inner.total_minted(), expected);
}
```

### `tests/indexer_tests.rs`

```rust
mod common;
use common::*;
use relaylab::chain::SourceChain;
use relaylab::indexer::*;

fn chain() -> SourceChain {
    SourceChain::new(SRC, &[(addr(1), 10_000), (addr(2), 10_000)])
}

#[test]
fn appending_duplicates_gaps_and_orphans() {
    let mut c = chain();
    for i in 0..3u8 {
        c.send(addr(1), addr(9), DST, 1 + u128::from(i), vec![])
            .unwrap();
        c.seal();
    }
    let mut ix = Indexer::new(c.block(0).unwrap());
    assert_eq!(ix.tip(), 0);
    assert_eq!(ix.ingest(c.block(0).unwrap()), Outcome::Duplicate);
    assert_eq!(ix.ingest(c.block(2).unwrap()), Outcome::Gap);
    assert_eq!(ix.ingest(c.block(1).unwrap()), Outcome::Appended);
    assert_eq!(ix.ingest(c.block(1).unwrap()), Outcome::Duplicate);
    assert_eq!(ix.ingest(c.block(2).unwrap()), Outcome::Appended);
    assert_eq!((ix.tip(), ix.event_count()), (2, 2));
    // a block from a different history whose parent is unknown
    let mut other = SourceChain::new(SRC, &[(addr(5), 100)]);
    other.send(addr(5), addr(6), DST, 1, vec![]).unwrap();
    other.seal();
    other.seal();
    assert_eq!(ix.ingest(other.block(2).unwrap()), Outcome::Orphan);
    assert_eq!(
        ix.ingest(other.block(0).unwrap()),
        Outcome::Duplicate,
        "same genesis"
    );
    assert_eq!((ix.tip(), ix.event_count()), (2, 2));
}

#[test]
fn a_competing_block_rolls_back_the_old_branch() {
    let mut c = chain();
    c.send(addr(1), addr(9), DST, 10, vec![]).unwrap();
    c.seal();
    c.send(addr(1), addr(9), DST, 20, vec![]).unwrap();
    c.seal();
    c.send(addr(1), addr(9), DST, 30, vec![]).unwrap();
    c.seal();
    let mut ix = Indexer::new(c.block(0).unwrap());
    for hgt in 1..=3 {
        ix.ingest(c.block(hgt).unwrap());
    }
    assert_eq!(ix.event_count(), 3);
    let old2 = ix.hash_at(2).unwrap();
    // the chain drops blocks 2 and 3 and builds another block 2
    c.reorg(2);
    c.send(addr(2), addr(8), DST, 77, vec![]).unwrap();
    c.seal();
    assert_eq!(
        ix.ingest(c.block(2).unwrap()),
        Outcome::Reorged { depth: 1 + 1 }
    );
    assert_ne!(ix.hash_at(2).unwrap(), old2);
    assert_eq!(ix.tip(), 2, "the old block 3 is gone");
    assert_eq!(ix.event_count(), 2);
    let evs = ix.events_for(&addr(8), 10);
    assert_eq!(evs.len(), 1);
    assert_eq!((evs[0].0, evs[0].1, evs[0].2.amount), (2, 0, 77));
    assert!(ix.events_for(&addr(1), 10).iter().all(|(h, _, _)| *h == 1));
}

#[test]
fn a_replacement_at_the_tip_is_a_reorg_of_depth_one() {
    let mut c = chain();
    c.send(addr(1), addr(9), DST, 10, vec![]).unwrap();
    c.seal();
    let mut ix = Indexer::new(c.block(0).unwrap());
    ix.ingest(c.block(1).unwrap());
    c.reorg(1);
    c.send(addr(1), addr(9), DST, 11, vec![]).unwrap();
    c.seal();
    assert_eq!(
        ix.ingest(c.block(1).unwrap()),
        Outcome::Reorged { depth: 1 }
    );
    assert_eq!(ix.events_for(&addr(9), 9).len(), 1);
    assert_eq!(ix.events_for(&addr(9), 9)[0].2.amount, 11);
}

#[test]
fn queries_respect_the_height_bound_and_finality() {
    let mut c = chain();
    for i in 0..5u8 {
        c.send(addr(1), addr(9), DST, 1 + u128::from(i), vec![])
            .unwrap();
        c.seal();
    }
    let mut ix = Indexer::new(c.block(0).unwrap());
    ix.sync(&c);
    assert_eq!(ix.tip(), 5);
    assert_eq!(ix.events_for(&addr(9), 3).len(), 3);
    assert_eq!(ix.events_for(&addr(9), 99).len(), 5);
    assert_eq!(ix.events_for(&addr(1), 99).len(), 5, "sender matches too");
    assert_eq!(ix.events_for(&addr(7), 99).len(), 0);
    assert_eq!(ix.finalized_height(2), 3);
    assert_eq!(ix.finalized_height(9), 0);
}

#[test]
fn syncing_twice_changes_nothing() {
    let mut c = chain();
    for _ in 0..4 {
        c.send(addr(1), addr(9), DST, 1, vec![]).unwrap();
        c.seal();
    }
    let mut ix = Indexer::new(c.block(0).unwrap());
    ix.sync(&c);
    let (t, n) = (ix.tip(), ix.event_count());
    ix.sync(&c);
    ix.sync(&c);
    assert_eq!((ix.tip(), ix.event_count()), (t, n));
}

#[test]
fn after_random_reorgs_the_index_equals_the_chain() {
    for seed in 1..=10u64 {
        let mut rng = Rng(seed);
        let mut c = chain();
        let mut ix = Indexer::new(c.block(0).unwrap());
        for round in 0..80 {
            for _ in 0..rng.below(3) {
                let _ = c.send(
                    addr(1 + rng.below(2) as u8),
                    addr(9 + rng.below(3) as u8),
                    DST,
                    1 + u128::from(rng.below(9)),
                    vec![],
                );
            }
            c.seal();
            if rng.below(4) == 0 {
                c.reorg(1 + rng.below(4));
            }
            if rng.below(3) != 0 {
                ix.sync(&c);
                for h in 0..=c.tip() {
                    assert_eq!(
                        ix.hash_at(h),
                        c.block(h).map(|b| b.hash),
                        "seed {seed} round {round} height {h}"
                    );
                }
                assert_eq!(ix.tip(), c.tip());
                let want: usize = (1..=c.tip()).map(|h| c.block(h).unwrap().msgs.len()).sum();
                assert_eq!(ix.event_count(), want, "seed {seed} round {round}");
            }
        }
    }
}

#[test]
fn blocks_fed_one_at_a_time_in_order_converge_even_with_repeats() {
    let mut rng = Rng(8);
    let mut c = chain();
    let mut ix = Indexer::new(c.block(0).unwrap());
    for _ in 0..60 {
        let _ = c.send(addr(1), addr(9), DST, 1 + u128::from(rng.below(5)), vec![]);
        c.seal();
        if rng.below(4) == 0 {
            // the new branch is longer than the old one, as it is when a heavier branch wins
            let d = 1 + rng.below(2);
            c.reorg(d);
            for _ in 0..=d {
                c.seal();
            }
        }
        for h in 1..=c.tip() {
            // deliver each block up to three times, as a flaky feed might
            for _ in 0..1 + rng.below(3) {
                let o = ix.ingest(c.block(h).unwrap());
                assert_ne!(o, Outcome::Orphan);
            }
        }
        assert_eq!(ix.tip(), c.tip());
        let want: usize = (1..=c.tip()).map(|h| c.block(h).unwrap().msgs.len()).sum();
        assert_eq!(ix.event_count(), want);
    }
}

#[test]
fn a_feed_of_blocks_cannot_see_a_shorter_chain_but_sync_can() {
    let mut c = chain();
    for _ in 0..4 {
        c.send(addr(1), addr(9), DST, 1, vec![]).unwrap();
        c.seal();
    }
    let mut ix = Indexer::new(c.block(0).unwrap());
    ix.sync(&c);
    c.reorg(4);
    c.seal();
    // the feed delivers the only block the chain has at height 1; it replaces the old blocks 1..4
    assert_eq!(
        ix.ingest(c.block(1).unwrap()),
        Outcome::Reorged { depth: 4 }
    );
    assert_eq!(
        ix.tip(),
        1,
        "ingest of a competing block rolls back everything above it"
    );
    // but if the chain just gets shorter with no new block, nothing is ever ingested
    let mut ix2 = Indexer::new(c.block(0).unwrap());
    let mut d = chain();
    for _ in 0..4 {
        d.send(addr(1), addr(9), DST, 1, vec![]).unwrap();
        d.seal();
    }
    ix2.sync(&d);
    d.reorg(2);
    assert_eq!(
        ix2.tip(),
        4,
        "a stale tip: no block arrived to say otherwise"
    );
    ix2.sync(&d);
    assert_eq!(ix2.tip(), 2);
    assert_eq!(ix2.event_count(), 2);
}
```

### `tests/gap_tests.rs`

```rust
//! Tests written by reading the planned mutant list: each pins something the main tests left loose.
mod common;
use common::*;
use relaylab::bridge::*;
use relaylab::chain::SourceChain;
use relaylab::merkle::Proof;
use relaylab::quorum::*;
use relaylab::relayer::*;

#[test]
fn a_small_order_key_cannot_forge_a_signature() {
    // The identity point as a public key, with the identity as R and a zero S, "verifies" any
    // message under the plain ed25519 equation. `verify_strict` rejects such weak keys.
    let mut identity = [0u8; 32];
    identity[0] = 1;
    let mut sig = [0u8; 64];
    sig[0] = 1;
    let set = ValidatorSet::new(&[identity], 1).unwrap();
    let a = Attestation {
        height: 5,
        hash: [0x55; 32],
        root: [0x66; 32],
        sigs: vec![(0, sig)],
    };
    assert_eq!(set.verify(1, &a), Err(QuorumError::BadSignature));
}

fn signed_by_all(
    seed_base: u8,
    n: u32,
    chain: u32,
    height: u64,
    hash: [u8; 32],
    root: [u8; 32],
) -> Attestation {
    use ed25519_dalek::{Signer, SigningKey};
    let sigs = (0..n)
        .map(|i| {
            let k = SigningKey::from_bytes(&[seed_base + i as u8; 32]);
            (
                i,
                k.sign(&attest_bytes(chain, height, &hash, &root))
                    .to_bytes(),
            )
        })
        .collect();
    Attestation {
        height,
        hash,
        root,
        sigs,
    }
}

#[test]
fn an_attestation_that_matches_on_only_the_hash_or_only_the_root_is_a_conflict() {
    let vals = validators(4, 0);
    let mut inb = inbox(&vals, 100);
    let a = signed_by_all(1, 4, SRC, 3, [1; 32], [2; 32]);
    assert_eq!(inb.submit_attestation(&a), Ok(()));
    let same_hash = signed_by_all(1, 4, SRC, 3, [1; 32], [9; 32]);
    assert_eq!(
        inb.submit_attestation(&same_hash),
        Err(BridgeError::Equivocation)
    );
    let same_root = signed_by_all(1, 4, SRC, 3, [9; 32], [2; 32]);
    assert_eq!(
        inb.submit_attestation(&same_root),
        Err(BridgeError::Equivocation)
    );
    assert_eq!(inb.trusted_root(3), Some([2; 32]));
}

/// Fails the second call of `deliver` once, then behaves.
struct Hiccup {
    inner: Inbox,
    delivers: u32,
}
impl Link for Hiccup {
    fn attest(&mut self, a: &Attestation) -> Result<(), LinkError> {
        Link::attest(&mut self.inner, a)
    }
    fn deliver(&mut self, e: &[u8], h: u64, p: &Proof) -> Result<(), LinkError> {
        self.delivers += 1;
        if self.delivers == 3 {
            return Err(LinkError::Transient);
        }
        Link::deliver(&mut self.inner, e, h, p)
    }
}

#[test]
fn a_poisoned_message_is_skipped_when_a_block_is_repeated() {
    let vals = validators(4, 0);
    let mut c = SourceChain::new(SRC, &[(addr(1), 1_000)]);
    c.send(addr(1), addr(9), 77, 5, vec![]).unwrap(); // poisoned: wrong destination
    c.send(addr(1), addr(9), DST, 6, vec![]).unwrap();
    c.send(addr(1), addr(9), DST, 7, vec![]).unwrap();
    c.seal();
    let mut link = Hiccup {
        inner: inbox(&vals, 100),
        delivers: 0,
    };
    let mut r = Relayer::new();
    let first = r.run_once(&c, &vals, &mut link).unwrap();
    assert_eq!(
        (first.delivered, first.rejected, first.stopped),
        (1, 1, true)
    );
    let second = r.run_once(&c, &vals, &mut link).unwrap();
    // the repeat skips the poisoned message instead of being rejected again, and does not redeliver the first good one
    assert_eq!(
        (second.delivered, second.rejected, second.stopped),
        (1, 0, false)
    );
    assert_eq!(link.inner.total_minted(), 13);
    assert_eq!(r.poisoned, vec![(1, 0)]);
}

#[test]
fn a_stopped_run_says_so() {
    let vals = validators(4, 0);
    let mut c = SourceChain::new(SRC, &[(addr(1), 1_000)]);
    c.send(addr(1), addr(9), DST, 6, vec![]).unwrap();
    c.seal();
    let mut link = Flaky::new(inbox(&vals, 100), 4);
    link.rng.0 = 1; // any state; the loop below finds a stopping run
    let mut saw_stop = false;
    let mut r = Relayer::new();
    for _ in 0..50 {
        let rep = r.run_once(&c, &vals, &mut link).unwrap();
        saw_stop |= rep.stopped;
        if !rep.stopped && r.cursor() == 2 {
            break;
        }
    }
    assert!(saw_stop);
    assert_eq!(link.inner.total_minted(), 6);
}

#[test]
fn the_cursor_starts_after_genesis() {
    assert_eq!(Relayer::new().cursor(), 1);
    assert_eq!(Relayer::default().cursor(), 1);
}

#[test]
fn a_chain_of_one_block_has_no_blocks_to_relay_and_tip_zero() {
    let vals = validators(4, 0);
    let c = SourceChain::new(SRC, &[(addr(1), 1)]);
    let mut inb = inbox(&vals, 1);
    assert_eq!(c.tip(), 0);
    assert_eq!(
        Relayer::new().run_once(&c, &vals, &mut inb).unwrap(),
        Report::default()
    );
}

#[test]
fn nonces_increase_and_the_exact_balance_can_be_sent() {
    let mut c = SourceChain::new(SRC, &[(addr(1), 10)]);
    let a = c.send(addr(1), addr(2), DST, 10, vec![]).unwrap();
    assert_eq!((a.nonce, c.balance(&addr(1)), c.escrow()), (0, 0, 10));
    assert_eq!(
        c.send(addr(1), addr(2), DST, 1, vec![]),
        Err(relaylab::chain::SendError::Insufficient)
    );
    let mut d = SourceChain::new(SRC, &[(addr(1), 10)]);
    let m1 = d.send(addr(1), addr(2), DST, 1, vec![]).unwrap();
    let m2 = d.send(addr(1), addr(2), DST, 1, vec![]).unwrap();
    assert_eq!((m1.nonce, m2.nonce), (0, 1));
    assert_ne!(m1.id(), m2.id());
    // a payload of exactly the limit is fine and one over is not
    assert!(d.send(addr(1), addr(2), DST, 1, vec![0; 256]).is_ok());
    assert!(d.send(addr(1), addr(2), DST, 1, vec![0; 257]).is_err());
}

#[test]
fn an_index_equal_to_the_size_is_never_valid() {
    use relaylab::merkle::{Proof, prove, root, verify};
    // in a one-leaf tree the proof of "leaf 1" would need no siblings at all, and the top hash is
    // the leaf hash: only the explicit bound on the index stops it
    let ls = vec![b"only".to_vec()];
    let r = root(&ls);
    assert!(verify(&r, &ls[0], &prove(&ls, 0).unwrap()));
    assert!(!verify(
        &r,
        &ls[0],
        &Proof {
            index: 1,
            size: 1,
            siblings: vec![]
        }
    ));
    for n in 1..=9usize {
        let ls: Vec<Vec<u8>> = (0..n).map(|i| vec![i as u8]).collect();
        let r = root(&ls);
        let p = prove(&ls, n - 1).unwrap();
        for last in 0..n {
            let q = Proof {
                index: n as u64,
                ..prove(&ls, last).unwrap()
            };
            assert!(!verify(&r, &ls[last], &q), "n={n}");
        }
        assert!(verify(&r, &ls[n - 1], &p));
    }
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
//! Timings for the pieces of a relay path. Run with `cargo run --release --example bench`.
use relaylab::bridge::Inbox;
use relaylab::chain::SourceChain;
use relaylab::indexer::Indexer;
use relaylab::merkle::{prove, root, verify};
use relaylab::msg::Message;
use relaylab::quorum::{Attestation, Validator, ValidatorSet, threshold_for};
use relaylab::relayer::Relayer;
use std::hint::black_box;
use std::time::Instant;

fn time(name: &str, iters: u32, mut f: impl FnMut()) {
    f();
    let t = Instant::now();
    for _ in 0..iters {
        f();
    }
    let ns = t.elapsed().as_nanos() as f64 / f64::from(iters);
    if ns > 1e6 {
        println!("{name:<44} {:>10.2} ms", ns / 1e6);
    } else {
        println!("{name:<44} {ns:>10.0} ns");
    }
}

fn main() {
    let m = Message {
        src: 1,
        dst: 2,
        nonce: 7,
        sender: [1; 20],
        recipient: [2; 20],
        amount: 5,
        payload: vec![0; 32],
    };
    time("message encode + id", 200_000, || {
        black_box(black_box(&m).id());
    });
    let leaves: Vec<Vec<u8>> = (0..64u8).map(|i| vec![i; 106]).collect();
    time("merkle root of 64 leaves", 20_000, || {
        black_box(root(black_box(&leaves)));
    });
    let r = root(&leaves);
    let p = prove(&leaves, 37).unwrap();
    time("merkle prove (64 leaves)", 20_000, || {
        black_box(prove(black_box(&leaves), 37));
    });
    time("merkle verify (depth 6)", 200_000, || {
        black_box(verify(&r, &leaves[37], black_box(&p)));
    });

    let vals: Vec<Validator> = (0..7)
        .map(|i| Validator::from_seed(i, [i as u8 + 1; 32], 0))
        .collect();
    let keys: Vec<[u8; 32]> = vals.iter().map(Validator::public).collect();
    let set = ValidatorSet::new(&keys, threshold_for(7)).unwrap();
    let mut c = SourceChain::new(1, &[([1; 20], u128::MAX / 2)]);
    c.send([1; 20], [2; 20], 2, 5, vec![]).unwrap();
    c.seal();
    let b = c.block(1).unwrap().clone();
    let sigs: Vec<_> = vals.iter().filter_map(|v| v.attest(&c, 1)).collect();
    time("validator attest (sign)", 5_000, || {
        black_box(vals[0].attest(black_box(&c), 1));
    });
    let a = Attestation {
        height: 1,
        hash: b.hash,
        root: b.root,
        sigs: sigs[..5].to_vec(),
    };
    time("quorum verify, 5 of 7 signatures", 2_000, || {
        set.verify(1, black_box(&a)).unwrap();
    });

    time("relay 100 messages end to end", 20, || {
        let mut c = SourceChain::new(1, &[([1; 20], u128::MAX / 2)]);
        for i in 0..100u8 {
            c.send([1; 20], [i; 20], 2, 5, vec![]).unwrap();
            if i % 10 == 9 {
                c.seal();
            }
        }
        let vals: Vec<Validator> = (0..4)
            .map(|i| Validator::from_seed(i, [i as u8 + 1; 32], 0))
            .collect();
        let keys: Vec<[u8; 32]> = vals.iter().map(Validator::public).collect();
        let mut inbox = Inbox::new(2, 1, ValidatorSet::new(&keys, 3).unwrap(), u128::MAX);
        let rep = Relayer::new().run_once(&c, &vals, &mut inbox).unwrap();
        assert_eq!(rep.delivered, 100);
    });
    let mut big = SourceChain::new(1, &[([1; 20], u128::MAX / 2)]);
    for i in 0..1000u32 {
        big.send([1; 20], [(i % 250) as u8; 20], 2, 1, vec![])
            .unwrap();
        big.seal();
    }
    time("index 1,000 blocks (one message each)", 20, || {
        let mut ix = Indexer::new(big.block(0).unwrap());
        ix.sync(&big);
        black_box(ix.event_count());
    });
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **53 tests** (merkle and message 11, chain and quorum 9, bridge 8, relayer 9, indexer 8, gap tests 8) and 8 consecutive full runs without a failure (nothing uses sockets, real time or OS randomness; the random tests use a fixed-seed generator, and the hostile link draws from one).

**Mutation check ✓.** I made 95 small source edits (a "mutant") across seven of the eight source modules, each in its own copy of the crate, and ran the whole test suite against each. **90 were caught by a failing test and 5 are equivalent (M7, M47, M69, M84, M91); none fails to compile and none survives unexplained.** **What the first pass found.** Before the mutants I had already found one real bug by an ordinary test (§4.2: the proof's size was not authenticated) and fixed it. I then wrote 8 tests (in `gap_tests.rs`) before the first run, to pin things I could already see were loose: the weak-key forgery under non-strict verification, a conflict in only the hash or only the root of an attestation, the skip of a poisoned message when a block is repeated, the "stopped" flag of a run, the starting cursor, a chain with no blocks beyond genesis, nonces and the exact-balance boundary. Against the 52 tests then in place, 88 of the 95 mutants were caught and 7 were not. Five of the seven are equivalent (below). One was a mistake in my mutant: replacing `verify_strict` with `verify` did not compile, because the `Verifier` trait was not imported; I rewrote it with the full path, and it was then caught by the weak-key test written beforehand. **One was a real gap**: relaxing the proof's index bound from `index >= size` to `index > size` survived, because no test used an index equal to the size, and in a tree of one leaf the proof `{index: 1, size: 1, siblings: []}` then verifies; one new test (`an_index_equal_to_the_size_is_never_valid`) closes it. The lesson in the real gap is the usual one: the existing proof tests moved the index to *other valid* positions, and none tried the one position that is just out of range. Five mutants remain, each with a stated reason for being equivalent (several are checks I kept on purpose as a second line of defence); none fails to compile. The column "change" is generated from the edit itself (the part that differs, with a little context, whitespace collapsed); `…` marks omitted text. The tests ran in the dev profile with optimized dependencies.

| # | module | change | result |
|---|---|---|---|
| M1 | `merkle` | `if c.len() == 2` → `if c.len() == 1` | caught by `a_forged_message_or_proof_is_refused` |
| M2 | `merkle` | `…level.len() > 1` → `…level.len() > 2` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M3 | `merkle` | `commit(leaves.len() as u64` → `commit(0` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M4 | `merkle` | `if index >=` → `if index >` | caught by `proofs_from_the_chain_verify_against_the_block_root` |
| M5 | `merkle` | `if idx % 2 == 1` → `if idx % 2 == 0` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M6 | `merkle` | `…if idx + 1 <` → `…if idx + 1 <=` | caught by `pause_and_cap_stop_minting` |
| M7 | `merkle` | `if proof.size == 0 \|\| proof.` → `if proof.` | **equivalent**: a proof with `size` 0 also has `index >= size` for every index, so the explicit zero test is redundant; kept for readability |
| M8 | `merkle` | `…proof.index >=` → `…proof.index >` | caught by `an_index_equal_to_the_size_is_never_valid` |
| M9 | `merkle` | `…if idx + 1 <` → `…if idx + 1 <=` | caught by `pause_and_cap_stop_minting` |
| M10 | `merkle` | `n = n.div_ceil(2)` → `n /= 2` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M11 | `merkle` | `used =` → `used <` | caught by `every_leaf_of_every_size_proves_and_nothing_else_does` |
| M12 | `merkle` | `…proof.siblings.len() && commit(proof.size, &h)` → `…proof.siblings.len() && h` | caught by `huge_amounts_are_compared_without_overflow` |
| M13 | `merkle` | `h.update([0x00` → `h.update([0x01` | caught by `proofs_match_python` |
| M14 | `merkle` | `h.update([0x01` → `h.update([0x00` | caught by `a_duplicated_last_leaf_changes_the_root` |
| M15 | `merkle` | `h.update([0x02` → `h.update([0x00` | caught by `roots_match_python` |
| M16 | `merkle` | `h.update(size` → `h.update(0u64` | caught by `a_duplicated_last_leaf_changes_the_root` |
| M17 | `msg` | `if b.len() <` → `if b.len() <=` | caught by `a_message_for_another_chain_pair_is_not_replayable_here` |
| M18 | `msg` | `if plen >` → `if plen >=` | caught by `decoding_is_strict` |
| M19 | `msg` | `if b.len() <` → `if b.len() <=` | caught by `a_forged_message_or_proof_is_refused` |
| M20 | `msg` | `if b.len() >` → `if b.len() >=` | caught by `a_message_for_another_chain_pair_is_not_replayable_here` |
| M21 | `msg` | `o.extend(self.src` → `o.extend(self.dst` | caught by `a_forged_message_or_proof_is_refused` |
| M22 | `msg` | `…self.nonce.to_b` → `…self.nonce.to_l` | caught by `message_encoding_and_id_match_python` |
| M23 | `msg` | `…HEADER_LEN: usize = 74` → `…HEADER_LEN: usize = 75` | caught by `a_forged_message_or_proof_is_refused` |
| M24 | `msg` | `sha256(b"msg-v1` → `sha256(b"msg-v2` | caught by `message_encoding_and_id_match_python` |
| M25 | `msg` | `…self.payload.len() as u` → `…self.payload.len() as u8 as u` | caught by `decoding_is_strict` |
| M26 | `msg` | `…MAX_PAYLOAD: usize = 256` → `…MAX_PAYLOAD: usize = 255` | caught by `sends_are_refused_for_the_right_reason` |
| M27 | `msg` | `o.extend(self.recipient` → `o.extend(self.sender` | caught by `a_forged_message_or_proof_is_refused` |
| M28 | `chain` | `if amount == 0` → `if amount == 1` | caught by `huge_amounts_are_compared_without_overflow` |
| M29 | `chain` | `…payload.len() >` → `…payload.len() >=` | caught by `sends_are_refused_for_the_right_reason` |
| M30 | `chain` | `if *bal <` → `if *bal <=` | caught by `huge_amounts_are_compared_without_overflow` |
| M31 | `chain` | `*bal -= amount` → `*bal -= 0` | caught by `a_reorg_refunds_the_dropped_messages` |
| M32 | `chain` | `self.escrow += amount` → `self.escrow += 0` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M33 | `chain` | `self.next_nonce += 1` → `self.next_nonce += 0` | caught by `sending_locks_funds_and_sealing_commits_them` |
| M34 | `chain` | `while dropped <` → `while dropped <=` | caught by `a_reorg_refunds_the_dropped_messages` |
| M35 | `chain` | `…self.blocks.len() > 1` → `…self.blocks.len() > 0` | caught by `a_reorg_refunds_the_dropped_messages` |
| M36 | `chain` | `self.escrow -= m.amount` → `self.escrow -= 0` | caught by `a_reorg_refunds_the_dropped_messages` |
| M37 | `chain` | `…m.sender).or_default() += m.amount` → `…m.sender).or_default() += 0` | caught by `a_reorg_refunds_the_dropped_messages` |
| M38 | `chain` | `b"hdr-v1` → `b"hdr-v2` | caught by `header_hash_matches_python` |
| M39 | `chain` | `self.blocks.len() as u64 - 1` → `self.blocks.len() as u64` | caught by `a_forged_message_or_proof_is_refused` |
| M40 | `chain` | `…self.tip() + 1` → `…self.tip() + 2` | caught by `a_forged_message_or_proof_is_refused` |
| M41 | `chain` | `let parent = self.blocks.last().expect("genesis").hash` → `let parent = [0u8; 32]` | caught by `sending_locks_funds_and_sealing_commits_them` |
| M42 | `chain` | `&height.to_be_bytes(), parent, roo` → `&height.to_be_bytes(), root, paren` | caught by `header_hash_matches_python` |
| M43 | `chain` | `…).sum::<u128>() + self.escrow` → `…).sum::<u128>()` | caught by `a_reorg_refunds_the_dropped_messages` |
| M44 | `quorum` | `2 * n / 3 + 1` → `2 * n / 3` | caught by `attestations_need_a_quorum_and_cannot_conflict` |
| M45 | `quorum` | `2 * n / 3` → `n / 2` | caught by `thresholds_must_exceed_two_thirds` |
| M46 | `quorum` | `…self.depth)? >` → `…self.depth)? >=` | caught by `a_forged_message_or_proof_is_refused` |
| M47 | `quorum` | `keys.is_empty() \|\| ` → `∅` | **equivalent**: an empty key list also fails the next two tests (the threshold must be at least 1 and at most the key count), with the same error; the explicit test is redundant |
| M48 | `quorum` | `…\|\| threshold <` → `…\|\| threshold <=` | caught by `a_forged_message_or_proof_is_refused` |
| M49 | `quorum` | `…\|\| threshold >` → `…\|\| threshold >=` | caught by `the_ed25519_vector_matches_python` |
| M50 | `quorum` | `…seen.len() <` → `…seen.len() <=` | caught by `the_ed25519_vector_matches_python` |
| M51 | `quorum` | `…!seen.insert(*i) ` → `…!seen.insert(*i) && false ` | caught by `quorum_checks_each_rule` |
| M52 | `quorum` | `key.verify_strict(` → `ed25519_dalek::Verifier::verify(key,` | caught by `a_small_order_key_cannot_forge_a_signature` |
| M53 | `quorum` | `…mut o = b"att-v1` → `…mut o = b"att-v2` | caught by `the_ed25519_vector_matches_python` |
| M54 | `quorum` | `o.extend(chain` → `o.extend(0u32` | caught by `the_ed25519_vector_matches_python` |
| M55 | `quorum` | `o.extend(hash); o.extend(root` → `o.extend(root); o.extend(hash` | caught by `the_ed25519_vector_matches_python` |
| M56 | `bridge` | `Some((h, r)) if *h == a.hash && *r == a.root` → `Some((h, _)) if *h == a.hash` | caught by `an_attestation_that_matches_on_only_the_hash_or_only_the_root_is_a_conflict` |
| M57 | `bridge` | `Some((h, r)) if *h == a.hash &&` → `Some((_, r)) if` | caught by `an_attestation_that_matches_on_only_the_hash_or_only_the_root_is_a_conflict` |
| M58 | `bridge` | `if ` → `if !` | caught by `a_message_for_another_chain_pair_is_not_replayable_here` |
| M59 | `bridge` | `if m.src !` → `if m.src =` | caught by `a_forged_message_or_proof_is_refused` |
| M60 | `bridge` | `if m.dst !` → `if m.dst =` | caught by `a_forged_message_or_proof_is_refused` |
| M61 | `bridge` | `if !` → `if ` | caught by `a_forged_message_or_proof_is_refused` |
| M62 | `bridge` | `if self.executed.contains(&id)` → `if false` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M63 | `bridge` | `.filter(\|t\| *t <=` → `.filter(\|t\| *t <` | caught by `huge_amounts_are_compared_without_overflow` |
| M64 | `bridge` | `self.executed.insert(id); self.` → `self.` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M65 | `bridge` | `…m.recipient).or_default() += m.amount` → `…m.recipient).or_default() += 0` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M66 | `bridge` | `Some(_) => Err(BridgeError::Equivocation` → `Some(_) => Ok(()` | caught by `attestations_need_a_quorum_and_cannot_conflict` |
| M67 | `bridge` | `…a.height, (a.hash, a.root` → `…a.height, (a.hash, a.hash` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M68 | `bridge` | `self.total_minted = total` → `self.total_minted += 0` | caught by `a_message_is_minted_once_with_a_valid_proof` |
| M69 | `relayer` | `if sigs.is_empty()` → `if false` | **equivalent**: with no signatures the inbox answers `Quorum(NotEnough)`, which the relayer already handles by stopping at that block, so the early exit saves a call and changes no result |
| M70 | `relayer` | `…self.cursor <=` → `…self.cursor <` | caught by `a_poisoned_message_is_skipped_when_a_block_is_repeated` |
| M71 | `relayer` | `…BridgeError::Quorum(_))) => break, // not enough signatures yet` → `…BridgeError::Quorum(_))) => { rep.rejected += 1; break; }` | caught by `a_relayer_cannot_deliver_what_validators_did_not_sign` |
| M72 | `relayer` | `…rep.delivered += 1` → `…rep.delivered += 2` | caught by `a_poisoned_message_is_skipped_when_a_block_is_repeated` |
| M73 | `relayer` | `…BridgeError::AlreadyExecuted)) => {}` → `…BridgeError::AlreadyExecuted)) => rep.delivered += 1,` | caught by `a_poisoned_message_is_skipped_when_a_block_is_repeated` |
| M74 | `relayer` | `self.cursor = h + 1` → `self.cursor = h + 2` | caught by `a_relayer_cannot_deliver_what_validators_did_not_sign` |
| M75 | `relayer` | `self.last = Some((h, block.hash))` → `self.last = None` | caught by `a_reorg_below_the_cursor_stops_the_relayer` |
| M76 | `relayer` | `self.poisoned.push((h, i)` → `let _ = (h, i` | caught by `a_poisoned_message_is_skipped_when_a_block_is_repeated` |
| M77 | `relayer` | `if self.poisoned.contains(&(h, i))` → `if false` | caught by `a_poisoned_message_is_skipped_when_a_block_is_repeated` |
| M78 | `relayer` | `…\|b\| b.hash) !` → `…\|b\| b.hash) =` | caught by `a_flaky_network_never_loses_or_repeats_a_message` |
| M79 | `relayer` | `Relayer { cursor: 1` → `Relayer { cursor: 0` | caught by `the_cursor_starts_after_genesis` |
| M80 | `relayer` | `…LinkError::Transient) => { rep.stopped = true; re` → `…LinkError::Transient) => { re` | caught by `a_flaky_network_never_loses_or_repeats_a_message` |
| M81 | `indexer` | `if b.height >` → `if b.height >=` | caught by `a_competing_block_rolls_back_the_old_branch` |
| M82 | `indexer` | `…b.height - 1) !` → `…b.height - 1) =` | caught by `a_competing_block_rolls_back_the_old_branch` |
| M83 | `indexer` | `…self.tip() - (b.height - 1)` → `…self.tip() - b.height` | caught by `a_competing_block_rolls_back_the_old_branch` |
| M84 | `indexer` | `if existing \|\| depth > 0 ` → `if existing ` | **equivalent**: a height that already holds a block always has indexed blocks above its parent, so `existing` and `depth > 0` are the same condition |
| M85 | `indexer` | `self.blocks.retain(\|h, _\| *h <=` → `self.blocks.retain(\|h, _\| *h <` | caught by `a_feed_of_blocks_cannot_see_a_shorter_chain_but_sync_can` |
| M86 | `indexer` | `…h, _), _\| *h <=` → `…h, _), _\| *h <` | caught by `a_competing_block_rolls_back_the_old_branch` |
| M87 | `indexer` | `*h <=` → `*h <` | caught by `queries_respect_the_height_bound_and_finality` |
| M88 | `indexer` | `…m.sender == *a \|\|` → `…m.sender == *a &&` | caught by `a_competing_block_rolls_back_the_old_branch` |
| M89 | `indexer` | `self.tip().saturating_sub(depth)` → `self.tip() - depth` | caught by `queries_respect_the_height_bound_and_finality` |
| M90 | `indexer` | `while h > 0` → `while h > 1` | caught by `after_random_reorgs_the_index_equals_the_chain` |
| M91 | `indexer` | `…= self.tip().min(chain.tip())` → `…= self.tip()` | **equivalent**: starting the walk at the indexer's own tip instead of the lower of the two tips changes nothing, because a height above the chain's tip has no block there and the loop steps down past it anyway |
| M92 | `indexer` | `…height in h + 1` → `…height in h + 2` | caught by `a_feed_of_blocks_cannot_see_a_shorter_chain_but_sync_can` |
| M93 | `indexer` | `…b.height == 0` → `…b.height == 1` | caught by `a_competing_block_rolls_back_the_old_branch` |
| M94 | `indexer` | `self.events.insert((b.height, i` → `self.events.insert((b.height, 0` | caught by `after_random_reorgs_the_index_equals_the_chain` |
| M95 | `indexer` | `&& h =` → `&& h !` | caught by `a_competing_block_rolls_back_the_old_branch` |

What the table does not show: it tests *these* edits, not every possible bug; the equivalent mutants could hide a real gap if my reasoning about them is wrong; the `hash` module has no mutants because it is a twelve-line wrapper whose outputs are covered through the Python known answers of the modules that use it; and passing every mutant says nothing about any real bridge, about security or about behavior on a network.

**Design notes.**

- Everything that decides money goes through one function, `Inbox::deliver`, in a fixed order: pause, decode, route, trusted root, proof, execute-once, cap, mint.
- The relayer holds no secrets and the inbox trusts nothing the relayer says; the quorum and the proof carry the weight.
- The executed set stores **ids** (hashes of the full encoding), not nonces.
- The root commits to the leaf count, so the numbers in a proof are authenticated.
- Not handled: fees, expiry, ordering, refunds of undeliverable messages, validator rotation, persistence, concurrency, real chains, light clients, optimistic or zero-knowledge verification.

</details>

---

## 18. Key takeaways

- A cross-chain message needs a canonical encoding and an id that covers its routing; the executed set is keyed by that id (✓ eight field changes, eight ids; the nonce-only set that lost half the transfers).
- A Merkle proof must be bound to the leaf, the index and the size, the tree must separate leaves from nodes and promote rather than duplicate, and an index equal to the size must be refused (✓ every leaf of every tree to 33 leaves; the size field my first version left unauthenticated).
- A validator quorum needs a threshold above two thirds, distinct and strictly verified signers and a signature over chain, height, hash and root; validators sign only what they consider final (✓ the identity key that forges under the plain equation).
- The destination checks in a fixed order and executes once: attested root, proof, id, cap; a height has one root (✓ replay, equivocation, cap, pause).
- A relayer is untrusted and idempotent: transient failure means "do not know", `AlreadyExecuted` is success, a poisoned message is skipped, and a changed history stops the relayer (✓ 12 hostile seeds; 60 random rounds of conservation).
- Finality is a policy; if a reorganization can be deeper than the policy, minting on it loses money (✓ 500 minted against nothing).
- An indexer stores blocks with parents, deletes events with their blocks, and syncs against the chain; a feed of blocks alone cannot see a shorter chain (✓ 10 seeds of 80 rounds).
- Signatures dominate cost, hashes do not; verify a quorum per block and a proof per message (✓ about 50 µs a signature against about 1 µs a proof).
- Everything about real bridges, messaging protocols, light clients, oracles and incidents is from memory here; the model teaches what must be verified and what must be idempotent, not any protocol.
- Mutation testing here pointed at real gaps and at redundant code: 1 real gap was closed with 1 new test (an index equal to the size), one mutant was a mistake in my own mutant, and 5 mutants remain, each with a stated reason for being equivalent; the more useful finding came from an ordinary test, before any mutant, when a proof for the first leaf of a three-leaf tree was accepted for a four-leaf tree.

**Connections.** [Lesson 49](49_merkle_transactions_consensus.md) supplied the Merkle trees and finality vocabulary that §4 and §8.4 make exact; [Lesson 48](48_crypto_primitives.md) the hashes and signatures; [Lesson 51](51_rpc_nodes_indexers.md) and [Lesson 52](52_web3_reliability.md) are the client habits (finality, retries, idempotency, reorganizations) that §8 and §9 repeat; [Lessons 53](53_solana_model.md), [56](56_substrate_polkadot.md) and [57](57_evm_rust.md) built the three chains whose messages such a system carries; [Lesson 39](39_testing.md) and [Lesson 40](40_property_testing_fuzzing.md) are the testing habits behind the hostile link and the random tests. Lesson 59 steps back from blockchain to the skills that make any of this survivable in production: architecture, code review, system design and reliability.

---

## 19. References

- The Inter-Blockchain Communication protocol (IBC), specification and documentation: <https://github.com/cosmos/ibc> and <https://ibc.cosmos.network/> (as I remember the locations).
- Documentation of messaging protocols I remember by name (check each for current behavior): Wormhole <https://wormhole.com/docs/>, LayerZero <https://docs.layerzero.network/>, Axelar <https://docs.axelar.dev/>, Hyperlane <https://docs.hyperlane.xyz/>, Chainlink CCIP <https://docs.chain.link/ccip>.
- Ethereum's description of bridges and of rollups' messaging: <https://ethereum.org/en/developers/docs/bridges/>
- Ed25519: RFC 8032 <https://www.rfc-editor.org/rfc/rfc8032>; `ed25519-dalek` documentation (including `verify_strict`): <https://docs.rs/ed25519-dalek>; `sha2`: <https://docs.rs/sha2>
- Merkle trees and the second-preimage problem: RFC 6962 (Certificate Transparency), which uses distinct leaf and node prefixes: <https://www.rfc-editor.org/rfc/rfc6962>
- The Graph (subgraphs) <https://thegraph.com/docs/> as one example of an indexing service.
- Practical Byzantine Fault Tolerance (Castro and Liskov, 1999) for the two-thirds argument.

The URLs and titles above are the standard locations as I remember them; I did not open them during this session (the sandbox cannot reach documentation sites), so if a link has moved, search for the title. **Unchecked or from memory:** everything about **real** systems: the taxonomy of bridge designs, IBC, Wormhole, LayerZero, Axelar, Hyperlane and CCIP, the incidents and their causes, claims about Bitcoin's duplicate-leaf property, light-client and optimistic and zero-knowledge designs, the details of ed25519 weak keys beyond the behavior tested here, the practices attributed to production operators (hardware key storage, threshold signing, slashing, monitoring) and the indexing services named. Everything marked ✓ was run in this session, against the crate in this lesson.
