---
title: "Lesson 49 — Merkle trees, transaction models, consensus, trust assumptions"
---

# Lesson 49 — Merkle trees, transaction models, consensus, trust assumptions

**Module 15 · Cryptography and distributed systems (second lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `sha2 0.11.0` and `ed25519-dalek 3.0.0` (25 packages in `Cargo.lock`, the practical crate only)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** It builds, in a small library crate (`merklelab`), the four ideas that every blockchain-style system is made of: a **Merkle tree** (one hash that commits to a list, with short proofs of membership), two **transaction models** (UTXO and accounts), a **hash-linked chain of blocks** with a toy proof-of-work and a most-work fork-choice rule, and the **arithmetic of agreement** (quorums, fault thresholds) together with the **trust assumptions** each design leans on. It is teaching code: **not audited, not a blockchain, not for production**. Not covered: networking and gossip (Lesson 50), real consensus protocols (leader election, view changes, Raft, PBFT, Tendermint, Nakamoto consensus in full), difficulty retargeting, mempools and fee markets, smart-contract execution, Merkle–Patricia and sparse Merkle tries, consistency proofs between tree versions, zero-knowledge proofs, economic security, and anything about specific live networks. **Nothing here makes a system secure.** Rust removes memory-safety bugs, but the failures in this lesson (an ambiguous tree, a replayable transfer, a chain accepted with too little work, a quorum that is too small) are logic errors that compile cleanly.

Environment notes: crates.io and GitHub were reachable from the sandbox, but the documentation websites and standards sites were not, so statements about RFCs, papers, Bitcoin and Ethereum are labeled "from memory". Expected Merkle roots in the known-answer tests were computed by a **separate Python implementation** of the same construction (✓). Timings are single-machine micro-measurements.

Builds on [Lesson 48](48_crypto_primitives.md) (hashes, domain separation, Ed25519, `verify_strict`, nonces and replay), [Lesson 46](46_databases_config_auth_jobs.md) (transactions and atomicity in databases), [Lesson 45](45_http_apis.md) (idempotency, limits), [Lesson 44](44_cli_serialization.md) (canonical binary encodings), [Lesson 43](43_observability_ci_supply_chain.md) (supply chain), [Lesson 41](41_benchmarking_profiling.md) (measuring), [Lesson 40](40_property_testing_fuzzing.md) (property tests), [Lesson 39](39_testing.md) (tests and mutation habits), [Lesson 29](29_locks_deadlocks.md) (races), [Lesson 26](26_error_design.md) (error enums) and [Lesson 16](16_collections.md) (`HashMap` and `HashSet`). Lesson 50 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

Lesson 48 gave you hashes, MACs, signatures and key handling. This lesson shows what you get when you stack them:

1. **A Merkle tree** turns "a million transactions" into 32 bytes plus proofs of about 20 hashes (✓ 17 hashes, 544 bytes, for 100,000 leaves).
2. **A transaction model** says what a valid state change is: who may spend what, and how double spending and replay are prevented.
3. **A chain of blocks** orders those changes and makes history expensive to rewrite.
4. **Consensus** is how independent machines agree on one history when some of them fail or lie; **trust assumptions** are the list of things that must be true for the result to be believable.

The lesson is organized around experiments that show where each idea breaks:

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| A Merkle root that hashes leaves and nodes alike is forgeable | a 64-byte "leaf" produces the root of a two-leaf tree (§2.4) |
| Duplicating the last node on odd levels makes different lists share a root | `[a,b,c]` and `[a,b,c,c]` collide, `[a..e]` and `[a..e,e]` too (§2.5) |
| A root does not commit to the number of leaves | a proof made for 3 leaves verifies when it claims 4 (§2.6) |
| A transfer without a nonce can be replayed; without a chain id it can be moved to another network | both rejected by tests (§3.4) |
| Sums of amounts must be checked, not wrapped | two inputs of 2⁶³ overflow `u64`; outputs summing past `u64::MAX` are refused (§3.3) |
| Mining cost is `2^bits` hashes | measured 190 / 3,818 / 55,774 / 883,796 tries against 256 / 4,096 / 65,536 / 1,048,576 (§4.3) |
| The best chain is the one with the most *work*, not the most blocks | a 3-block branch overtakes a 2-block one and reports a reorg of depth 2 (§4.4) |
| With `n` nodes and `f` Byzantine ones, quorums must be bigger than a majority | brute force over every subset for `n ≤ 10`, and a closed form checked up to `n = 24` (§5.3) |
| One more faulty node than the threshold breaks safety | `n = 4, f = 2` allows two conflicting decisions (§5.4) |

Total in the practical challenge: one library crate, 5 modules, **74 tests**, an example that prints the measurements quoted below, and **100 mutants** run against the test suite.

---

## 2. Merkle trees

### 2.1 The problem

You have a list of items (transactions in a block, files in a backup, entries in a log) and you want:

- one short value that **commits** to the whole list (change any item or the order and the value changes), and
- a way to show **"this item is in that list"** to someone who holds only the short value, **without sending the list**.

Hashing the whole list in one go (`H(item1 ‖ item2 ‖ …)`) gives the first property but not the second: to check one item you would need all the others. A **Merkle tree** (Ralph Merkle, 1979, from memory) arranges the hashes in a binary tree: each leaf is the hash of an item, each interior node is the hash of its two children, and the **root** is the commitment. A membership proof is the list of **sibling hashes along the path** from the leaf to the root; the verifier recomputes the path and compares with the root it already trusts.

```
               root = H(01 ‖ N1 ‖ N2)
              /                      \
   N1 = H(01 ‖ L0 ‖ L1)         N2 = H(01 ‖ L2 ‖ L3)
      /        \                     /        \
   L0=H(00‖t0) L1=H(00‖t1)       L2=H(00‖t2) L3=H(00‖t3)

proof for t2:  [ L3 , N1 ]   (siblings, bottom first)
verify:        L2' = H(00‖t2);  N2' = H(01‖L2'‖L3);  root' = H(01‖N1‖N2');  root' == root ?
```

### 2.2 Cost ✓

For `n` leaves the tree needs `n` leaf hashes plus `n − 1` interior hashes (`2n − 1` in total), and a proof has `⌈log₂ n⌉` hashes at most. ✓ Measured on this machine (release build, about 52-byte leaves, median of five runs, three separate runs of the example):

| leaves | build the root | proof size | verify one proof |
|---|---|---|---|
| 1,000 | 0.23 ms | 10 hashes (320 B) | 1.50–1.57 µs |
| 10,000 | 2.11–2.43 ms | 14 hashes (448 B) | 2.03–2.19 µs |
| 100,000 | 23.07–24.52 ms | 17 hashes (544 B) | 2.49–2.60 µs |

A million transactions would need a proof of 20 hashes, 640 bytes (arithmetic, not measured). Verification grows with the *logarithm* of the list, which is the point: a phone can check one payment against a block header it trusts without downloading the block (this is the "light client" idea; from memory, Bitcoin calls it SPV).

**A limitation of my implementation, stated plainly:** `Tree::prove` recomputes sibling subtrees from the leaves, so one proof costs `O(n)` ✓ (about 0.14 ms for 1,000 leaves and about 14.3–14.6 ms for 100,000). A real implementation stores the levels (or builds all proofs in one pass) and answers in `O(log n)`. I kept the simple version because the lesson is about correctness of the construction, and the tests do not depend on it.

### 2.3 The construction used here ✓

The crate follows the layout of RFC 6962 and RFC 9162 (Certificate Transparency; from memory, the algorithms were re-derived and then **checked against an independent Python implementation** for sizes 0, 1, 2, 3, 5, 7, 8 and 13, ✓ `roots_match_an_independent_python_implementation`):

- leaf hash: `SHA-256(0x00 ‖ data)`; interior hash: `SHA-256(0x01 ‖ left ‖ right)`;
- the empty tree's root is `SHA-256("")`;
- for `n ≥ 2` leaves, split at the **largest power of two strictly smaller than `n`**: the left subtree is a full tree of `k` leaves, the right one holds the remaining `n − k`. An odd leaf is never duplicated and never paired with itself.

```rust
fn split_point(n: usize) -> usize {
    assert!(n >= 2, "split_point needs at least two leaves");
    1usize << (usize::BITS - 1 - (n - 1).leading_zeros())
}
```

(✓ for every `n` from 2 to 40 the tests exercise the proof path of every leaf; a mutant that used the largest power of two `≤ n` made the process abort and was caught.)

### 2.4 Pitfall 1: leaves and nodes must not look alike ✓

If a leaf is `H(data)` and a node is `H(left ‖ right)` (no prefixes), a leaf whose data is the concatenation of two hashes hashes **exactly like** an interior node. The attacker takes a real two-transaction tree, forms a single "transaction" `H(tx0) ‖ H(tx1)` (64 bytes) and offers a one-leaf tree with the *same root*. ✓ The test `without_prefixes_an_interior_node_can_pose_as_a_leaf` performs this against `unprefixed_root` (a deliberately flawed function in the crate) and succeeds; with the `0x00`/`0x01` prefixes the same forgery produces different hashes (✓ `leaves_and_nodes_are_hashed_with_different_prefixes`). This is the Merkle-tree version of the domain separation from Lesson 48 §4: **different roles in a structure get different labels**.

### 2.5 Pitfall 2: duplicating the odd node ✓

A common way to handle an odd number of nodes at a level is to pair the last node with a copy of itself (this is what Bitcoin does, from memory). It makes `[a, b, c]` and `[a, b, c, c]` produce the **same root** ✓ (`duplicating_the_last_node_makes_different_lists_share_a_root`, also for five and six leaves). From memory: this was reported in 2012 as CVE-2012-2459 for Bitcoin, where two different blocks could share a Merkle root, and the countermeasure is a validity rule that rejects blocks with duplicate transactions at the right place; I did not verify the details or the rule's wording. The RFC 6962 layout above avoids the problem by construction, and ✓ the same two lists give different roots in `merklelab`.

### 2.6 Pitfall 3: the root does not commit to the size ✓

My first draft of the proof tests assumed that changing `proof.size` would always make verification fail. It does not, and the test failed with a clear message. ✓ A proof made for a tree of **3** leaves, index 0, also verifies when it claims **4** leaves, because the walk the verifier performs is the same. The root hash alone therefore does not tell you how many leaves the tree had. Systems that need that (an append-only log) publish the size **together with the root and sign both** (from memory, Certificate Transparency signs a "tree head" with the size and the root). ✓ `the_root_alone_does_not_commit_to_the_tree_size` documents the surprise and shows that many other size changes are caught by the path length. Lesson: *test the properties you assume*, because wrong assumptions in a verifier are security bugs.

### 2.7 Verification, line by line ✓

`verify` follows the iterative algorithm of RFC 9162 (from memory): keep the leaf index `f` and the last index `s = size − 1`; at each step, if `f` is odd or equals `s`, the sibling is on the left; otherwise on the right; shift both right by one; at the end `s` must be zero and the computed hash must equal the root. The crate pins the details with tests: an index equal to the size is refused (✓ `an_index_equal_to_the_size_is_refused_even_when_the_path_is_empty`, a mutant that allowed it survived until I added the one-leaf case where the path is empty), every leaf of every tree up to 40 leaves verifies, and a changed leaf, root, path element, truncated path, extended path or claimed position is rejected for every tree up to 17 leaves.

Four mutants of this function **survive** and are *equivalent*, not gaps (§13): the `size == 0` test is covered by the index test; the early `s == 0` return is also covered because an extra path element changes the hash, so the final comparison fails anyway; and two details of the RFC's inner loop (the `f != 0` guard and the `f & 1 == 0` test) are unreachable once the other checks hold. They are harmless defensive code.

### 2.8 What Merkle trees are used for

Commitments to the transactions in a block (§4), light-client proofs, content-addressed stores (Git uses hash-linked objects, a related but different structure; from memory), append-only audit logs with consistency proofs (not built here), and authenticated key-value maps (Merkle–Patricia and sparse Merkle tries; from memory, not built here). In every case the security comes from the hash function (Lesson 48 §2) *and* from getting the encoding, the odd-node rule and the size commitment right.

---

## 3. Transaction models

A ledger records who owns what. Two designs dominate (from memory, Bitcoin and Ethereum are the best-known examples of the first and second).

### 3.1 UTXO: coins are objects ✓

State is a set of **unspent transaction outputs** (UTXOs). Each output says "this amount may be spent by this key". A transaction **consumes** whole outputs and **creates** new ones; whatever the inputs hold and the outputs do not claim is the **fee**.

```rust
pub struct OutPoint { pub txid: Hash, pub index: u32 }
pub struct Output   { pub owner: PubKey, pub amount: u64 }
pub struct Tx {
    pub inputs: Vec<OutPoint>,
    pub outputs: Vec<Output>,
    pub extra: u64,
    pub sigs: Vec<[u8; 64]>,   // one per input, over the sighash
}
```

Validation (`UtxoSet::apply`), in the order the crate checks it: has inputs and outputs; one signature per input; no zero outputs; **no input listed twice**; the txid is new; every input exists (unspent) and its signature verifies against the *owner of the spent output*; the sum of inputs does not overflow; the sum of outputs does not overflow and does not exceed the inputs. Only then is the set changed. ✓ A failed transaction leaves the set untouched (`structural_errors` checks the total after each failure), and ✓ value is conserved: after two spends the remaining total plus the fees equals the original coinbase.

Double spending is prevented **structurally**: once an output is spent it is gone, so a second spend finds no such output (✓ `spending_the_same_output_twice_is_refused`), and listing it twice in one transaction is refused explicitly (✓; a mutant that skipped that check was caught). Parallelism is natural: transactions that touch disjoint outputs cannot interfere (from memory, a commonly cited advantage).

**Coinbase** transactions (no inputs) create new coins by rule. In this toy they are accepted without any check of who may create them; in a real chain that rule is part of consensus (block reward). Two otherwise identical coinbases would have the same id, so the `extra` field exists (✓ a duplicate id is refused).

### 3.2 Account model: balances and nonces ✓

State is a map from public key to `(balance, nonce)`. A transfer says "from A to B, amount X, nonce N, on chain C" and is signed by A.

```rust
pub struct Transfer {
    pub chain_id: u32, pub from: PubKey, pub to: PubKey,
    pub amount: u64, pub nonce: u64, pub sig: [u8; 64],
}
```

`Accounts::apply` checks the chain id, the signature (`verify_strict`), a non-zero amount, the **nonce equals the account's current nonce**, the sender can pay (`checked_sub`), and the recipient's balance cannot overflow (`checked_add`); it computes both new balances first and then writes them. ✓ Transfers to yourself consume the nonce and change no balance (a mutant that skipped that special case corrupted the balance and was caught).

### 3.3 Arithmetic is part of validation ✓

Amounts are `u64`, and attacker-controlled sums can wrap. ✓ Two outputs of `u64::MAX` and `101` would wrap to a small total and pass an "outputs ≤ inputs" test; the crate uses `checked_add` and rejects with `Overflow` (✓ `sums_are_checked_not_wrapped`; a mutant that used `wrapping_add` for the input sum was caught). The same test creates two coinbase outputs of 2⁶³ and shows that the sum, 2⁶⁴, does not fit a `u64` while a `u128` total still reports it. This is the Lesson 48 §9.1 rule ("saturating or checked arithmetic on hostile numbers") applied to money. In a consensus system the rule is stricter: **every node must overflow the same way**, so you do not leave it to debug-versus-release behavior ([Lesson 4](04_variables_and_types.md) covers integer overflow behavior).

### 3.4 Replay protection ✓

A valid signed transfer is just bytes; anyone who sees it can send it again. The account model uses two defenses, both ✓ tested:

- the **nonce** makes each transfer valid exactly once and in order (✓ a replay returns `BadNonce { expected: 1, got: 0 }`; a gap or an out-of-order nonce is refused until the missing one arrives);
- the **chain id** is inside the signed bytes, so a transfer signed for a test network cannot be applied on the main one (✓ `a_transfer_for_another_network_is_refused`, and relabeling the chain id breaks the signature).

In the UTXO model the spent output itself is the replay protection (the same transaction applied twice finds its inputs gone; ✓ the crate reports the second application as `DuplicateTxid`, because that check comes first, a precedence the test now pins). Cross-chain replay is possible in any model where signatures do not name the chain; the chain id (or a hash of the genesis block in the signed data) fixes it. This is the "context" idea of Lesson 48 §9.

### 3.5 Transaction ids and malleability ✓

The id of a transaction should not change when someone alters only the *signature encoding*. The crate computes `txid` over the body **excluding signatures** and signs a *different* digest (`sighash`) using a different domain label (✓ `the_txid_ignores_signatures_but_the_sighash_does_not_ignore_the_body`; a mutant that gave both digests the same label was caught). With `verify_strict` and Ed25519's fixed-size signatures the practical malleability is small; from memory, Bitcoin's older signature format allowed third parties to alter signatures and thereby the txid, which is why later designs moved signatures out of the id computation. I did not verify details.

### 3.6 Weak keys ✓

An output locked to an invalid or small-order key must be unspendable. ✓ The test uses the identity point as the owner and a signature made of the identity point and a zero scalar, which a non-strict verifier accepts for **any** message under that key; `verify_strict` refuses it (a mutant that used plain `verify` was caught). In a real chain the choice of strict versus lenient verification must be identical on every node, because it decides which blocks are valid (Lesson 48 §8.2).

### 3.7 Comparison

| | UTXO | Account |
|---|---|---|
| State | set of unspent outputs | map key → balance, nonce |
| Replay protection | the spent output disappears | nonce (and chain id) |
| Double spend | second spend finds nothing | nonce and balance checks, in order |
| Validity of a transaction depends on | only the outputs it names | the sender's current nonce and balance (so order matters) |
| Parallel validation | easy for disjoint inputs | harder: same-sender transfers must be ordered |
| Storage | grows with unspent outputs | grows with accounts |
| Natural fit | payments, simple scripts | stateful contracts (from memory) |

Neither is "better"; each moves complexity somewhere else. What matters for you: **write down the state, the validity rule and the replay defense before writing code**, and test each rule with an input that violates only that rule.

---

## 4. Blocks and chains

### 4.1 The header ✓

```rust
pub struct Header {
    pub height: u64, pub prev: Hash, pub merkle_root: Hash,
    pub time: u64, pub bits: u32, pub nonce: u64,
}
```

The **block hash** is the SHA-256 of a fixed-width encoding of all six fields (✓ every field is covered: changing any one changes the hash, and a mutant that dropped `nonce` from the hash made mining never succeed, so that test **hung** and the mutant counts as detected by timeout). `prev` is the hash of the parent header: the chain is a linked list by hash, so altering an old block changes its hash, which no longer matches the `prev` of its child. ✓ `changing_an_old_block_breaks_every_later_link` forges a replacement of block 1 with different transactions, re-mines it, and shows that block 2 no longer attaches (it becomes an orphan). To rewrite history an attacker must redo the work of every block from the changed one up, and then outpace the honest chain.

The header commits to the transactions through `merkle_root`. ✓ A light client holding only headers can check one transaction with a Merkle proof (`the_header_commits_to_transactions_through_the_merkle_root`).

### 4.2 Proof of work ✓

Anyone can propose a block, so something must make proposing costly. In proof of work (from memory, Bitcoin's design, using Hashcash) the header hash, read as a number, must be below a target; here the equivalent is "at least `bits` leading zero bits". A miner varies `nonce` until a hash qualifies:

```rust
pub fn mine(mut h: Header, max_tries: u64) -> Option<(Header, u64)> {
    for n in 0..max_tries {
        h.nonce = n;
        if h.meets_target() {
            return Some((h, n + 1));
        }
    }
    None
}
```

Each try succeeds with probability `2^-bits`, so the expected number of tries is `2^bits`, and the *work* a valid block represents is taken to be `2^bits` hashes (✓ `work(bits) = 1 << bits`).

### 4.3 The measurement ✓

Mean number of hashes until success over several different headers (the nonce search is deterministic, so this is reproducible):

| bits | headers | mean tries | `2^bits` | rate |
|---|---|---|---|---|
| 8 | 20 | 190 | 256 | too short to time |
| 12 | 20 | 3,818 | 4,096 | 6.28–6.56 Mhash/s |
| 16 | 20 | 55,774 | 65,536 | 6.08–6.49 Mhash/s |
| 20 | 8 | 883,796 | 1,048,576 | 6.44–6.55 Mhash/s |

Twenty samples are few (the standard deviation of a geometric variable equals its mean, so a mean of 20 samples has about 22% relative error), which is why the means are within that range of `2^bits`, not exactly on it. One thread of this machine does about 6.1–6.6 million header hashes per second; the *same* 20 bits would take real mining hardware a negligible time, and real networks use far more than 20 bits (from memory, the real figures are much larger; I did not check them). The point to remember is the **cost model**: adding one bit doubles the expected work, and verification costs one hash, so work is cheap to check and expensive to produce.

### 4.4 Fork choice: most cumulative work ✓

Two miners may find blocks at the same time, creating a **fork**. Every node stores both branches and follows the one with the **greatest total work**, summing `2^bits` over the blocks, in a `u128` so the sum cannot overflow for the supported `bits ≤ 64`. Ties keep the branch seen first. ✓ Tests: an equal-work branch does not replace the current tip (`SideBranch`); a longer branch does, and the store reports a **reorg** with `depth` = how many blocks of the old chain were abandoned (✓ depth 2 and depth 1 cases); blocks that arrive before their parent are refused as orphans and accepted once the parent arrives (a real node would keep them in a pool; this toy does not). A mutant that compared work with `>=` instead of `>` was caught: it would make the tip flip between equal branches.

**Why work and not length?** If blocks could have different difficulty, a long chain of easy blocks would beat a short chain of hard ones. ✓ arithmetic in the exercises: 6 blocks of 8 bits are 1,536 units of work, 2 blocks of 10 bits are 2,048. In this toy the difficulty is a constant (no retargeting), so work is proportional to length, but the rule is the right one for networks that adjust difficulty.

### 4.5 The rules a block must satisfy ✓

`Store::add` accepts a block only if: it is new; its parent is known; its height is parent + 1; its `bits` equal the network's difficulty; its hash meets the target; its Merkle root matches its transactions; its time is later than its parent's; and its time is not more than `max_future` seconds ahead of the local clock. ✓ Each rule has a test that violates **only** that rule (the test mines the offending block properly so that the other rules still pass) and a boundary test where one is available (the future limit is inclusive, and `saturating_add` keeps a hostile timestamp of `u64::MAX` from wrapping). Rejected blocks are not stored (✓ `len()` unchanged).

What is *not* validated here: the transactions themselves. A real node validates each transaction against the state (the previous section's `apply`) before accepting the block, and ✓ the lesson keeps the two layers separate so each can be tested alone.

### 4.6 Confirmations and the attacker's odds ✓

When a payment is included in a block it has one confirmation; each later block on top adds one (`Store::confirmations`, ✓ tests). How many are enough? The simplest model (from the Bitcoin paper, from memory; I did not re-derive the full analysis) treats the race between the honest chain and an attacker with fraction `q` of the total hash power as a gambler's-ruin problem: if `q < 1/2`, the probability that the attacker ever catches up from `z` blocks behind is `(q / (1 − q))^z`; if `q ≥ 1/2` it is 1. ✓ The function is tested against that formula and gives (computed by the example):

| attacker share | z = 1 | z = 3 | z = 6 | z = 10 |
|---|---|---|---|---|
| q = 0.1 | 0.111111 | 0.001372 | 0.000002 | ≈ 0 |
| q = 0.3 | 0.428571 | 0.078717 | 0.006196 | 0.000209 |

Read it as a *model* of catching up from behind, not as a guarantee: it assumes independent random block production, ignores network delay, the attacker's earlier head start (if they mined in secret), and economic incentives. Two things hold regardless of the model: confirmations reduce risk but never to zero, and a majority attacker (`q ≥ 1/2`) wins in the long run, which is the meaning of the "51% attack" (§6).

### 4.7 What this toy leaves out

Difficulty retargeting (so the block rate stays steady as hash power changes), the validation of transactions inside blocks, uncle/orphan handling beyond refusing orphans, a mempool, block size limits and fees, the coinbase reward rule, reorg handling of the ledger state (undoing and redoing transactions), pruning, and persistence. Each is a real engineering subject. In particular **reorgs force the state to roll back**, which is why a ledger implementation should be able to undo a block (store deltas) or recompute from a checkpoint.

---

## 5. Consensus

### 5.1 The problem

Several machines each hold a copy of the ledger. They receive transactions in different orders, some messages are lost or delayed, and some machines crash or are malicious. **Consensus** is a protocol that lets the correct machines agree on one sequence of transactions. Two properties are always discussed:

- **Safety**: nothing bad happens; in particular, no two correct nodes decide different values for the same position ("no forks among decisions").
- **Liveness**: something good eventually happens; the system keeps deciding new values.

(From memory) A famous result (Fischer, Lynch and Paterson, 1985) says that in a fully asynchronous network where even one node may crash, no deterministic protocol can guarantee both safety and liveness; practical protocols give up *guaranteed* liveness under asynchrony (they stay safe, and make progress when the network behaves) or use randomness or time-outs. I did not verify the statement's exact hypotheses; look up the paper before relying on details.

### 5.2 Fault models

- **Crash faults**: a node stops and does nothing wrong. Protocols such as Raft and Paxos (from memory) tolerate `f` crashes with `n = 2f + 1` nodes; decisions need a **majority** quorum.
- **Byzantine faults**: a node may do anything, including sending different messages to different peers (**equivocation**). Classical protocols tolerate `f` Byzantine nodes with `n ≥ 3f + 1`; decisions need a quorum of `⌊2n/3⌋ + 1`.
- **Proof-of-work (Nakamoto) consensus** replaces "count the nodes" with "count the work": the assumption is that honest participants control **more than half of the hash power**, and agreement is probabilistic (§4.6) rather than final after one round. Participation is open: anyone can mine, no membership list is needed.

### 5.3 Quorum arithmetic ✓

Why `3f + 1`? Consider a vote where every correct node votes for at most one value and Byzantine nodes may vote for both of two conflicting values. For two conflicting decisions to happen, two quorums of size `q` must be reachable. The two quorums overlap in at least `2q − n` nodes (✓ `min_overlap`, checked by **brute force over all pairs of subsets** for every `n ≤ 10` and every `q`). If that overlap is larger than `f`, the overlap contains at least one **correct** node, which would have had to vote twice: impossible. So safety needs `2q − n > f`. Liveness needs the correct nodes alone to be able to form a quorum: `n − f ≥ q`. With `q = ⌊2n/3⌋ + 1`, both hold exactly when `n ≥ 3f + 1`:

```rust
pub fn max_faulty(n: usize) -> usize { n.saturating_sub(1) / 3 }
pub fn bft_quorum(n: usize) -> usize { (2 * n) / 3 + 1 }
pub fn min_overlap(n: usize, q: usize) -> usize { (2 * q).saturating_sub(n) }
```

✓ Tests: the table (`n = 4` tolerates 1 with quorum 3; `n = 7` tolerates 2 with quorum 5; `n = 100` tolerates 33 with quorum 67); for every `n` from 1 to 60, at the maximum number of faults the quorums share an honest node, honest nodes can form a quorum, and **no conflicting decision is possible** (checked by enumerating every way the correct nodes can split their votes); and a closed form (`2·max(q − f, 0) ≤ n − f` is the condition for a conflict) agrees with the enumeration for all `n ≤ 24`, `f ≤ n`, `q ≤ n`.

### 5.4 One fault too many ✓

- `n = 4`, `f = 2` (the threshold is 1): two conflicting values can both collect 3 votes, so safety is broken (✓ `one_fault_too_many_breaks_safety_or_liveness`).
- `n = 3`, `f = 1` where the faulty node simply stays silent: the two correct nodes cannot form the quorum of 3, so the system **stops** (liveness lost) but never decides wrongly. Which of the two failures you see depends on what the faulty nodes do; the thresholds are the same.
- `n = 10` (a company runs 3 of the validators): ✓ with `f = 3` no conflict is possible and the 7 others can still form the quorum of 7; with `f = 4` both a conflicting decision (`2·(7 − 4) = 6 ≤ 10 − 4 = 6`) and a stall become possible. A single operator controlling more than a third of the validators is therefore a violation of the assumption, even if no one ever misbehaves.

A majority quorum is **not** enough against liars: ✓ with `n = 5`, `q = 3` and one equivocating node, conflicting decisions become possible, while with no Byzantine node they are impossible (`a_simple_majority_is_not_enough_against_equivocation`). That is the difference between the crash and Byzantine worlds.

### 5.5 What a real protocol adds

The functions above say *how big* a quorum must be. A protocol also needs: who proposes (a leader or rotating proposer), how nodes recover when the proposer is faulty (view change), how votes are authenticated (signatures from Lesson 48, with replay protection per round and per chain), how validators join and leave, and a rule for what is final. Finality differs by design: PoW chains offer **probabilistic** finality (the older a block, the less likely it is reverted), voting protocols can offer **deterministic** finality (a block with a quorum of signatures is final as long as fewer than a third are faulty), at the cost of a known, bounded validator set (from memory, general characterization). I implemented none of this; the lesson's claim is only the arithmetic and the experiments above.

---

## 6. Trust assumptions

Every security claim has the shape *"if these things hold, then this is true"*. Making the list explicit is the main skill of this lesson, because the failures are almost always in an assumption nobody wrote down.

| Mechanism | What it lets you conclude | What you must trust |
|---|---|---|
| Hash (Lesson 48) | "this data is the data the hash was computed from" | the hash function is collision/second-preimage resistant; **you obtained the hash from a trusted place** |
| Merkle root | "this item is the `i`-th of the committed list" | the root came from a trusted source; the size is authenticated if it matters (§2.6); the construction separates leaves and nodes (§2.4) |
| Signature (Lesson 48) | "the holder of this key signed this" | the key belongs to who you think; the key was not stolen; the verification rule (strict or not) matches everyone else's |
| Transaction validity | "this transfer follows the rules" | every node applies **the same** rules, including arithmetic and signature verification |
| Proof-of-work chain | "this history cost at least this much work" | honest hash power is a majority (and network delay is small relative to block time) |
| Quorum voting | "a quorum of validators signed this decision" | fewer than a third of validators are faulty (`f ≤ (n−1)/3`); validator set membership is correct |
| Light client | "this transaction is in a block of the chain I follow" | the headers you follow are the best chain (you did not verify all the transactions); you are connected to at least one honest peer |
| Software | "the program does what the spec says" | the implementation and its dependencies are correct and reviewed ([Lesson 43](43_observability_ci_supply_chain.md)); **different nodes running the same bug agree on the wrong answer** |
| Clocks | "this block is not from the future" | local clocks are roughly right; the `max_future` rule limits how far they can be abused |
| Keys | "only the owner can spend" | key generation, storage and backup (Lesson 48 §10 and §12) |

A few consequences worth stating:

1. **Trust is not eliminated, it is relocated.** "Trustless" systems replace trusting a named institution with trusting a majority of work or a supermajority of validators, plus the software, plus the key holders. Whether that is a better bargain depends on who the adversaries are.
2. **Thresholds are cliffs, not slopes.** Below a third (voting) or a half (work), the guarantees hold; above, the protocol may fail completely (§5.4).
3. **Independence is part of the assumption.** "At most one third faulty" assumes failures are not correlated: ten validators running the same client, hosted by the same provider, are not ten independent nodes.
4. **Social and legal layers exist** (what the community does after a theft, who may change the software). I make no claim about them.
5. **The edge of your model matters.** Everything in §4.6 and §5 assumes a particular network and adversary; if your deployment differs, the numbers may not apply.

---

## 7. Rust notes for consensus-critical code

- **Determinism is a requirement**, not a nicety (Lesson 50 develops this): every node must compute the same result from the same input. In this lesson's terms: no floating point in validity rules (the one `f64` function, `catch_up_probability`, is for *analysis* and is never used by the store), no `HashMap` iteration order in anything that is hashed (`HashMap` in `std` uses a random hasher, so iteration order differs between runs and machines; ✓ the crate only uses maps for lookup and sums, and sums are order-independent), no dependence on the platform's `usize` size (use fixed-width integers for formats), and explicit byte order (big-endian here) for every encoded field.
- **Checked arithmetic everywhere money or work is summed** (§3.3); decide between `checked_*` (refuse), `saturating_*` (clamp) and `wrapping_*` (modular, as hashes do) on purpose.
- **Errors as data**: `TxError`, `TransferError`, `BlockError` are enums with one variant per rule ([Lesson 26](26_error_design.md)). That makes tests precise (`assert_eq!(…, Err(TxError::DuplicateInput(op)))`) and lets a node log *which* rule rejected a block. One caution: **the order of checks is observable** (a replayed transaction reports `DuplicateTxid`, not `UnknownInput`), so an error-code change can be a compatibility break in a system where nodes compare reasons.
- **Validate, then mutate.** Both ledgers compute everything that can fail before writing anything, so a rejected transaction cannot leave half an update (✓ tested). With locks or threads ([Lesson 29](29_locks_deadlocks.md)) this becomes "validate and commit under one critical section".
- **Newtypes** ([Lesson 15](15_newtype_typestate.md)) would make `Hash`, `PubKey` and `TxId` impossible to confuse; the crate uses type aliases for brevity, which is a weaker choice I would not make in a larger codebase.

---

## 8. Common mistakes

1. Hashing leaves and interior nodes the same way (§2.4).
2. Duplicating the last node on odd levels (§2.5).
3. Treating a Merkle root as a commitment to the number of leaves (§2.6).
4. Accepting a proof without checking the index against the size.
5. Summing amounts with `+` on `u64`, or comparing sums that may have wrapped (§3.3).
6. Applying a transaction partway: debiting before checking the recipient (§3.2, §7).
7. Signatures without a chain id, or transfers without a nonce (§3.4).
8. Including signatures in the transaction id (§3.5).
9. Choosing the best chain by block count instead of work (§4.4).
10. Trusting a block's own claim of its difficulty (the store compares `bits` with the network setting, ✓).
11. Using a majority quorum against Byzantine faults, or `3f` nodes instead of `3f + 1` (§5.3).
12. Counting correlated nodes as independent (§6).
13. Reading "N confirmations" as a guarantee (§4.6).
14. Using floating point or `HashMap` order inside a validity rule (§7).
15. Believing that because the tests pass and the code is in Rust, the system is safe.

---

## 9. Security notes and honest limits

- `merklelab` is teaching code. Its consensus is a **single-process store**, not a network protocol; it has no peer-to-peer layer (Lesson 50), no mempool, no transaction validation inside blocks, no difficulty adjustment, and no persistence.
- The mining experiment measures one thread on one machine; it says nothing about real networks.
- The attacker-odds table is a simple model (§4.6). Real analyses account for network delay, strategy and economics (from memory).
- The Byzantine arithmetic assumes each correct node votes for **at most one** value per round and that quorum votes are authenticated; real protocols spend most of their complexity on making those assumptions true.
- Cryptographic caveats from Lesson 48 apply unchanged: no audit, no side-channel testing, no post-quantum claim.
- Nothing here addresses privacy: both transaction models expose amounts and keys.

---

## 10. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** For a Merkle tree over 1,000,000 leaves, how many hashes does it take to build the root, and how many hashes (and bytes) are in an inclusion proof? Why can a verifier get by with the proof alone?
2. **(Warm-up)** In the UTXO model, what stops an owner from spending the same output twice? In the account model, what stops them from re-sending a payment someone else recorded?
3. **(Easy)** Write `merkle_root(&[&[u8]]) -> [u8; 32]` with the `0x00`/`0x01` prefixes and the largest-power-of-two split, with the empty tree hashing the empty string.
4. **(Medium)** Write `verify_pow2(root, leaf, index, siblings)` for trees whose leaf count is a power of two (no size parameter): it must reject an index too large for the proof's height.
5. **(Medium)** Write `confirmations_needed(q, risk) -> Option<u32>`: the smallest `z` with catch-up probability below `risk`, or `None` if `q ≥ 0.5`.
6. **(Medium-hard)** Explain why `[a, b, c]` and `[a, b, c, c]` have the same Bitcoin-style root, why that is dangerous for a validator that rejects blocks with duplicate transactions, and how the layout of §2.3 avoids it.
7. **(Hard)** Branch A has 6 blocks at 8 bits; branch B has 2 blocks at 10 bits. Which is the best chain under "most work"? Which under "longest"? What would happen to a client that used "longest" when difficulty can vary?
8. **(Hard)** A network of 10 validators needs a quorum of 7. Three validators belong to one company. Another four are hosted by one cloud provider. Analyze what each party can do alone, and what happens if both collude or fail together.

<details markdown="1">
<summary>Answers</summary>

1. Building needs `n` leaf hashes plus `n − 1` interior hashes: 1,999,999 hashes. A proof has `⌈log₂ 1,000,000⌉ = 20` hashes, 640 bytes. The verifier already trusts the root, recomputes 21 hashes (one leaf, 20 interior), and compares. If the leaf were not in the tree it would need a second preimage or a collision in SHA-256 to match (Lesson 48 §2.1). ✓ for the scaling: 100,000 leaves took 23–25 ms to build with 17-hash proofs.
2. UTXO: the first spend **removes** the output from the set, so a second spend names an output that does not exist (and listing it twice inside one transaction is refused explicitly). Accounts: the **nonce** inside the signed bytes must equal the account's current nonce and then increases, so a replay carries an old nonce and fails; the **chain id** stops it being replayed on another network. ✓ both tested.
3. A possible answer (✓ compiled and run, compared with the crate's tree for all sizes 0 to 40):

   ```rust
   use sha2::{Digest, Sha256};

   pub type Hash = [u8; 32];

   fn h(prefix: u8, parts: &[&[u8]]) -> Hash {
       let mut s = Sha256::new();
       s.update([prefix]);
       for p in parts {
           s.update(p);
       }
       s.finalize().into()
   }

   pub fn merkle_root(leaves: &[&[u8]]) -> Hash {
       match leaves.len() {
           0 => Sha256::digest([]).into(),
           1 => h(0x00, &[leaves[0]]),
           n => {
               let k = 1usize << (n - 1).ilog2(); // largest power of two below n
               let left = merkle_root(&leaves[..k]);
               let right = merkle_root(&leaves[k..]);
               h(0x01, &[&left, &right])
           }
       }
   }
   ```

   The expression `1usize << (n - 1).ilog2()` is the largest power of two below `n` for `n ≥ 2` (`ilog2` of `n − 1`). My first attempt used `(n - 1).next_power_of_two() / 2`, which gives `0` for `n = 2` and recursed forever (a stack overflow in the test), a bug the compile-and-run check caught.
4. A possible answer (✓ compiled and run against proofs from the crate):

   ```rust
   use sha2::{Digest, Sha256};

   pub type Hash = [u8; 32];

   fn h(prefix: u8, a: &[u8], b: &[u8]) -> Hash {
       let mut s = Sha256::new();
       s.update([prefix]);
       s.update(a);
       s.update(b);
       s.finalize().into()
   }

   /// Verifies a proof for a tree whose leaf count is a power of two.
   /// `index` is the leaf position; `siblings` go from the leaf upwards.
   pub fn verify_pow2(root: &Hash, leaf: &[u8], mut index: usize, siblings: &[Hash]) -> bool {
       if index >> siblings.len() != 0 {
           return false; // the index does not fit in a tree of this height
       }
       let mut acc = h(0x00, leaf, &[]);
       for sib in siblings {
           acc = if index & 1 == 0 {
               h(0x01, &acc, sib)
           } else {
               h(0x01, sib, &acc)
           };
           index >>= 1;
       }
       &acc == root
   }
   ```

   Because the height is `siblings.len()`, an index with any bit above that position cannot belong to the tree (the first line). A verifier that ignored this would accept the same proof for `index` and `index + 2^height`, for example.
5. A possible answer (✓ compiled and run against the crate's formula):

   ```rust
   /// Smallest number of confirmations `z` such that an attacker with hash-power
   /// share `q` catches up with probability below `risk`; `None` if `q >= 0.5`
   /// (the attacker eventually wins whatever `z` is).
   pub fn confirmations_needed(q: f64, risk: f64) -> Option<u32> {
       assert!((0.0..1.0).contains(&q) && risk > 0.0 && risk < 1.0);
       if q >= 0.5 {
           return None;
       }
       if q == 0.0 {
           return Some(0);
       }
       let ratio = q / (1.0 - q);
       let mut z = 0;
       let mut p = 1.0;
       while p >= risk {
           z += 1;
           p *= ratio;
       }
       Some(z)
   }
   ```

   The loop multiplies by `q/(1 − q)` until the product falls below the risk; the result is small for small `q` and grows as `q` approaches one half. It inherits all the limits of the model in §4.6.
6. With the duplication rule, the odd node at the bottom level (`c`) is paired with itself, so `[a, b, c]` becomes `[a, b, c, c]` before hashing and both lists produce the same tree (✓ the same root for three/four and five/six leaves). A validator that tests "no duplicate transaction ids" on the *block's list* would reject the padded block `[a, b, c, c]` but accept the genuine `[a, b, c]` with the same root; if it then remembers "this root is invalid" it can wrongly mark the genuine block invalid. (From memory this is the shape of the 2012 Bitcoin issue; I did not check how it was handled.) The RFC 6962 layout splits at a power of two and never pairs a node with itself, and leaf/node prefixes keep the levels distinct, so different lists give different roots (✓ tested).
7. Work: A = 6 × 2⁸ = 1,536; B = 2 × 2¹⁰ = 2,048, so B is the best chain under "most work" even though A is longer (✓ checked in code). Under "longest" A wins. A client using "longest" could be fed a long chain of cheap blocks (an attacker needs little hash power per block when difficulty is low) and would follow a chain that the rest of the network, using work, ignores. (In this toy difficulty is constant, so the two rules agree; the rule matters once difficulty varies.)
8. Using `n = 10`: `max_faulty = 3`, `q = 7`. ✓ (computed with the crate's functions in a test) with `f = 3`: no conflicting decisions are possible, and the 7 others can form a quorum, so **the company alone** (3) can neither break safety nor stop the system. With `f = 4` both become possible, so **the cloud provider alone** (4 validators) is above the threshold: a provider outage stops progress (the 6 remaining cannot reach 7), and a provider that behaved maliciously could cause conflicting decisions. If both collude, 7 of 10 is a quorum by themselves. The lesson: count **fault domains** (operators, providers, regions, software versions), not machines, and treat correlated failure as one failure. Mitigations (not implemented): spread validators across providers, diversify clients, raise `n`.

</details>

---

## 11. Debugging challenge

A block builder computes the Merkle root with the function below. Three bug reports arrive: (1) a block with no transactions crashes the node; (2) two blocks, one with transactions `[a, b, c]` and one with `[a, b, c, c]`, have the same root; (3) a researcher shows that a single 64-byte "transaction" built from two transaction hashes gives the same root as a genuine two-transaction block. Explain each and fix the function.

```rust
use sha2::{Digest, Sha256};

fn h(parts: &[&[u8]]) -> [u8; 32] {
    let mut s = Sha256::new();
    for p in parts {
        s.update(p);
    }
    s.finalize().into()
}

pub fn root(leaves: &[Vec<u8>]) -> [u8; 32] {
    let mut level: Vec<[u8; 32]> = leaves.iter().map(|l| h(&[l])).collect();
    while level.len() > 1 {
        if level.len() % 2 == 1 {
            level.push(*level.last().unwrap());
        }
        level = level.chunks(2).map(|c| h(&[&c[0], &c[1]])).collect();
    }
    level[0]
}

pub fn txids_unique(leaves: &[Vec<u8>]) -> bool {
    let mut seen = std::collections::HashSet::new();
    leaves.iter().all(|l| seen.insert(h(&[l])))
}
```

<details markdown="1">
<summary>Solution</summary>

1. **Empty input.** `level[0]` is read when `level` is empty, so the call panics with an index out of bounds (✓ the test `buggy_panics_on_an_empty_block`). A node that panics on a valid (if unusual) block can be crashed remotely. Define the root of an empty list (RFC 6962 uses the hash of the empty string) and never panic on network input.
2. **Duplication of the odd node.** When a level has an odd length, the last hash is pushed again, so `[a, b, c]` and `[a, b, c, c]` build identical levels (✓ `buggy_has_two_blocks_with_one_root`). The `txids_unique` check, which flags the padded list, is not a fix: it only accepts one of the two and leaves the root ambiguous.
3. **No leaf/node domain separation.** A leaf is `H(data)` and a node is `H(left ‖ right)`, so the 64-byte "transaction" `H(a) ‖ H(b)` hashes like the node above `a` and `b` (✓ `buggy_lets_a_node_pose_as_a_leaf`). The attacker presents a block with a single fake transaction and the same root.

Fixed version (✓ compiled; the test `fixed_separates_all_three` checks the empty case, the three/four case and the fake leaf): hash leaves and nodes with different prefix bytes, split at the largest power of two below the length instead of duplicating, and give the empty list a defined root.

```rust
use sha2::{Digest, Sha256};

fn h(prefix: u8, parts: &[&[u8]]) -> [u8; 32] {
    let mut s = Sha256::new();
    s.update([prefix]);
    for p in parts {
        s.update(p);
    }
    s.finalize().into()
}

pub fn root(leaves: &[Vec<u8>]) -> [u8; 32] {
    fn go(hashes: &[[u8; 32]]) -> [u8; 32] {
        match hashes.len() {
            0 => Sha256::digest([]).into(),
            1 => hashes[0],
            n => {
                let k = 1usize << (n - 1).ilog2();
                h(0x01, &[&go(&hashes[..k]), &go(&hashes[k..])])
            }
        }
    }
    let hashed: Vec<[u8; 32]> = leaves.iter().map(|l| h(0x00, &[l])).collect();
    go(&hashed)
}
```

Two notes: the fixed function recomputes hashes recursively on slices, which is `O(n)` and fine for a block; and the *encoding* of each transaction before it is hashed still has to be canonical (Lesson 48 §4), which this function cannot guarantee.

</details>

---

## 12. Interview questions

1. *What is a Merkle tree, and what does an inclusion proof contain? How big is it for a million leaves?*
2. *Why do leaf hashes and node hashes need different prefixes? What attack does that prevent?*
3. *What goes wrong when the last node of an odd level is duplicated?*
4. *Compare the UTXO and account models. How does each prevent double spending and replay?*
5. *Why must a transaction's signature cover the chain id and a nonce?*
6. *Why should transaction ids exclude signatures? What is malleability?*
7. *How does a hash-linked chain make history tamper-evident, and what does proof of work add?*
8. *How does a node choose between two valid forks? Why work, not length?*
9. *What does "6 confirmations" mean? What does it guarantee?*
10. *Safety versus liveness: define both. Why does a Byzantine protocol need `3f + 1` nodes?*
11. *What is the difference between crash faults and Byzantine faults? What is equivocation?*
12. *What does "decentralized" assume about the participants? Give three examples of correlated failures.*
13. *Why must consensus-critical code be deterministic? Name three ways Rust code can accidentally be non-deterministic.*

<details markdown="1">
<summary>Answers</summary>

1. A binary tree of hashes whose root commits to the list; the proof is the sibling hashes along the path from a leaf to the root, `⌈log₂ n⌉` of them: 20 hashes (640 bytes) for a million leaves.
2. Without prefixes a leaf whose data is two concatenated hashes hashes like an interior node, so an attacker can present a shorter or different list with the same root (✓ performed in §2.4). Prefixes (domain separation) make the roles distinguishable.
3. Different lists can share a root (`[a,b,c]` and `[a,b,c,c]`), which makes block identity ambiguous and complicates validation (✓ §2.5; the Bitcoin CVE-2012-2459 history is from memory).
4. UTXO: outputs are consumed whole; once spent they are gone, so a second spend finds nothing. Account: balances plus a per-account nonce signed into each transfer; replay has an old nonce. Both: signatures name the chain, so a transfer cannot be moved to another network.
5. Without a nonce a valid transfer can be replayed; without a chain id it can be replayed on another network that uses the same keys.
6. A third party who can change the signature encoding without invalidating it changes the id of the transaction (malleability), which breaks anything that refers to it (a dependent transaction). Excluding signatures from the id removes the problem (✓ §3.5).
7. Each header contains the parent's hash, so changing an old block changes every later hash; proof of work makes recomputing those blocks expensive, and the Merkle root ties the header to the transactions.
8. By greatest cumulative work (first seen wins ties). With variable difficulty, length would let an attacker win with many cheap blocks (✓ exercise 7).
9. The transaction is in a block with five more blocks on top. It makes reversal less likely (in the simple model, exponentially so for an attacker below half of the hash power), but it is a probabilistic statement with assumptions (§4.6), not a guarantee.
10. Safety: no two correct nodes decide differently. Liveness: the system keeps making decisions. Byzantine nodes can vote for both of two conflicting values, so a quorum must be large enough that any two overlap in a correct node (`2q − n > f`) while correct nodes can still form one (`n − f ≥ q`); both hold iff `n ≥ 3f + 1` (✓ brute-forced).
11. A crashed node stops; a Byzantine node may behave arbitrarily. Equivocation is sending conflicting messages to different peers (voting for two values).
12. That the faulty fraction is below the threshold and failures are independent. Correlated failures: the same cloud provider or data centre, the same software bug or client, the same operator or key custodian, the same jurisdiction, the same time source.
13. Every node must reach the same result from the same input or the network splits. Examples: floating-point arithmetic and library functions that differ by platform; iterating a `HashMap` (random order) while hashing or serializing; relying on `usize` width or on overflow behavior that differs between debug and release; reading the clock or a random source inside validation; threads that finish in different orders.

</details>

---

## 13. Practical challenge: `merklelab`

Build a library crate that:

1. Computes RFC 6962-style Merkle roots (prefixes, largest-power-of-two split, empty tree), builds inclusion proofs, and verifies them (including bad indexes, sizes and paths); also provides two deliberately flawed constructions (`unprefixed_root`, `duplicating_root`) for the attack tests.
2. Implements a UTXO ledger and an account ledger with signed transactions, checked arithmetic, replay protection, atomic application and strict signature verification.
3. Implements a toy proof-of-work block store with all the block rules of §4.5, a most-work fork choice, reorg reporting and confirmations.
4. Provides the quorum functions of §5 and the attacker catch-up probability, with brute-force tests of the closed forms.
5. Has an example that prints the measurements, passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

```toml
[package]
name = "merklelab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "Teaching code for Lesson 49: Merkle trees, ledgers, a toy chain and quorum arithmetic. Not audited."

[dependencies]
sha2 = "0.11"
ed25519-dalek = "3"
```

### `src/lib.rs`

```rust
//! Teaching code for Lesson 49: Merkle trees, two transaction models, a toy
//! proof-of-work chain and quorum arithmetic. **Not audited; not for production.**
#![deny(missing_docs)]

pub mod chain;
pub mod hash;
pub mod ledger;
pub mod merkle;
pub mod quorum;
```

### `src/hash.rs`

```rust
//! Hash helpers shared by the other modules.

use sha2::{Digest, Sha256};

/// A 32-byte SHA-256 digest.
pub type Hash = [u8; 32];

/// SHA-256 of the concatenation of `parts`.
pub fn sha256(parts: &[&[u8]]) -> Hash {
    let mut h = Sha256::new();
    for p in parts {
        h.update(p);
    }
    h.finalize().into()
}

/// Lower-case hex of a digest.
pub fn hex(h: &[u8]) -> String {
    h.iter().map(|b| format!("{b:02x}")).collect()
}

/// Number of leading zero *bits* of a digest.
pub fn leading_zero_bits(h: &Hash) -> u32 {
    let mut n = 0;
    for b in h {
        if *b == 0 {
            n += 8;
        } else {
            n += b.leading_zeros();
            break;
        }
    }
    n
}
```

### `src/merkle.rs`

```rust
//! Merkle trees in the style of RFC 6962 / RFC 9162 (as remembered): leaves and
//! interior nodes are hashed with different prefixes, and a tree of `n` leaves
//! splits at the largest power of two smaller than `n`, so nothing is duplicated.
//! Also two *deliberately flawed* constructions used by the lesson's attacks.

use crate::hash::{Hash, sha256};

/// Prefix byte for leaf hashes.
pub const LEAF_PREFIX: u8 = 0x00;
/// Prefix byte for interior-node hashes.
pub const NODE_PREFIX: u8 = 0x01;

/// Hash of a leaf: `SHA-256(0x00 || data)`.
pub fn leaf_hash(data: &[u8]) -> Hash {
    sha256(&[&[LEAF_PREFIX], data])
}

/// Hash of an interior node: `SHA-256(0x01 || left || right)`.
pub fn node_hash(left: &Hash, right: &Hash) -> Hash {
    sha256(&[&[NODE_PREFIX], left, right])
}

/// Root of the empty tree: `SHA-256("")`.
pub fn empty_root() -> Hash {
    sha256(&[])
}

/// Largest power of two strictly smaller than `n` (requires `n >= 2`).
fn split_point(n: usize) -> usize {
    assert!(n >= 2, "split_point needs at least two leaves");
    1usize << (usize::BITS - 1 - (n - 1).leading_zeros())
}

fn mth(leaves: &[Hash]) -> Hash {
    match leaves.len() {
        0 => empty_root(),
        1 => leaves[0],
        n => {
            let k = split_point(n);
            node_hash(&mth(&leaves[..k]), &mth(&leaves[k..]))
        }
    }
}

/// An inclusion proof: the sibling hashes from the leaf up to the root.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Proof {
    /// Index of the leaf (0-based).
    pub index: usize,
    /// Number of leaves in the tree the proof was made for.
    pub size: usize,
    /// Sibling hashes, bottom first.
    pub path: Vec<Hash>,
}

/// A Merkle tree over a list of leaves (kept as leaf hashes).
#[derive(Clone, Debug)]
pub struct Tree {
    leaves: Vec<Hash>,
}

impl Tree {
    /// Builds a tree over `leaves` (any number, including zero).
    pub fn new(leaves: &[&[u8]]) -> Self {
        Tree {
            leaves: leaves.iter().map(|l| leaf_hash(l)).collect(),
        }
    }

    /// Number of leaves.
    pub fn len(&self) -> usize {
        self.leaves.len()
    }

    /// Whether the tree has no leaves.
    pub fn is_empty(&self) -> bool {
        self.leaves.is_empty()
    }

    /// The root hash.
    pub fn root(&self) -> Hash {
        mth(&self.leaves)
    }

    /// Inclusion proof for leaf `index`, or `None` if out of range.
    pub fn prove(&self, index: usize) -> Option<Proof> {
        if index >= self.leaves.len() {
            return None;
        }
        let mut path = Vec::new();
        let mut slice = &self.leaves[..];
        let mut m = index;
        while slice.len() > 1 {
            let k = split_point(slice.len());
            if m < k {
                path.push(mth(&slice[k..]));
                slice = &slice[..k];
            } else {
                path.push(mth(&slice[..k]));
                slice = &slice[k..];
                m -= k;
            }
        }
        path.reverse();
        Some(Proof {
            index,
            size: self.leaves.len(),
            path,
        })
    }
}

/// Root of a tree over `leaves` without building a [`Tree`].
pub fn root_of(leaves: &[&[u8]]) -> Hash {
    Tree::new(leaves).root()
}

/// Checks that `leaf` is leaf number `proof.index` of a tree of `proof.size`
/// leaves whose root is `root`. Iterative algorithm in the style of RFC 9162.
pub fn verify(root: &Hash, leaf: &[u8], proof: &Proof) -> bool {
    if proof.size == 0 || proof.index >= proof.size {
        return false;
    }
    let mut f = proof.index;
    let mut s = proof.size - 1;
    let mut r = leaf_hash(leaf);
    for p in &proof.path {
        if s == 0 {
            return false;
        }
        if f & 1 == 1 || f == s {
            r = node_hash(p, &r);
            if f & 1 == 0 {
                while f & 1 == 0 && f != 0 {
                    f >>= 1;
                    s >>= 1;
                }
            }
        } else {
            r = node_hash(&r, p);
        }
        f >>= 1;
        s >>= 1;
    }
    s == 0 && &r == root
}

/// **Flawed on purpose.** A Bitcoin-style root: no prefixes, and an odd node at
/// any level is paired with a copy of itself.
pub fn duplicating_root(leaves: &[&[u8]]) -> Hash {
    if leaves.is_empty() {
        return empty_root();
    }
    let mut level: Vec<Hash> = leaves.iter().map(|l| sha256(&[l])).collect();
    while level.len() > 1 {
        if level.len() % 2 == 1 {
            let last = *level.last().expect("non-empty");
            level.push(last);
        }
        level = level.chunks(2).map(|c| sha256(&[&c[0], &c[1]])).collect();
    }
    level[0]
}

/// **Flawed on purpose.** The same split as [`root_of`] but with no prefixes,
/// so a leaf and an interior node are indistinguishable.
pub fn unprefixed_root(leaves: &[&[u8]]) -> Hash {
    fn go(h: &[Hash]) -> Hash {
        match h.len() {
            0 => empty_root(),
            1 => h[0],
            n => {
                let k = split_point(n);
                sha256(&[&go(&h[..k]), &go(&h[k..])])
            }
        }
    }
    let hashes: Vec<Hash> = leaves.iter().map(|l| sha256(&[l])).collect();
    go(&hashes)
}
```

### `src/ledger.rs`

```rust
//! Two ways to record who owns what: a UTXO model (coins are objects that are
//! spent whole) and an account model (balances plus a per-account nonce).
//! Both verify Ed25519 signatures with `verify_strict`.

use std::collections::{HashMap, HashSet};

use ed25519_dalek::{Signature, Signer, SigningKey, VerifyingKey};

use crate::hash::{Hash, sha256};

/// A 32-byte Ed25519 public key.
pub type PubKey = [u8; 32];

fn verify_sig(owner: &PubKey, msg: &[u8], sig: &[u8; 64]) -> bool {
    match VerifyingKey::from_bytes(owner) {
        Ok(vk) => vk.verify_strict(msg, &Signature::from_bytes(sig)).is_ok(),
        Err(_) => false,
    }
}

// ---------------------------------------------------------------- UTXO model

/// Reference to one output of an earlier transaction.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct OutPoint {
    /// Id of the transaction that created the output.
    pub txid: Hash,
    /// Position of the output in that transaction.
    pub index: u32,
}

/// An amount locked to a public key.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct Output {
    /// Who may spend it.
    pub owner: PubKey,
    /// How much.
    pub amount: u64,
}

/// A UTXO transaction.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Tx {
    /// Outputs being spent (empty only for a coinbase).
    pub inputs: Vec<OutPoint>,
    /// Outputs being created.
    pub outputs: Vec<Output>,
    /// Free field so that two otherwise identical coinbases get different ids.
    pub extra: u64,
    /// One signature per input, over [`Tx::sighash`].
    pub sigs: Vec<[u8; 64]>,
}

const TXID_DOMAIN: &[u8] = b"lab/utxo/txid/v1";
const SIG_DOMAIN: &[u8] = b"lab/utxo/sighash/v1";

impl Tx {
    fn body(&self) -> Vec<u8> {
        let mut v = Vec::new();
        v.extend_from_slice(&(self.inputs.len() as u64).to_be_bytes());
        for i in &self.inputs {
            v.extend_from_slice(&i.txid);
            v.extend_from_slice(&i.index.to_be_bytes());
        }
        v.extend_from_slice(&(self.outputs.len() as u64).to_be_bytes());
        for o in &self.outputs {
            v.extend_from_slice(&o.owner);
            v.extend_from_slice(&o.amount.to_be_bytes());
        }
        v.extend_from_slice(&self.extra.to_be_bytes());
        v
    }

    /// Transaction id: a hash of the body, **excluding** the signatures, so
    /// changing a signature's encoding cannot change the id.
    pub fn txid(&self) -> Hash {
        sha256(&[
            &(TXID_DOMAIN.len() as u64).to_be_bytes(),
            TXID_DOMAIN,
            &self.body(),
        ])
    }

    /// The digest every input signature covers.
    pub fn sighash(&self) -> Hash {
        sha256(&[
            &(SIG_DOMAIN.len() as u64).to_be_bytes(),
            SIG_DOMAIN,
            &self.body(),
        ])
    }

    /// Fills `sigs` by signing the sighash with one key per input.
    pub fn sign(&mut self, keys: &[&SigningKey]) {
        let h = self.sighash();
        self.sigs = keys.iter().map(|k| k.sign(&h).to_bytes()).collect();
    }
}

/// Why a UTXO transaction was refused.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum TxError {
    /// A non-coinbase transaction with no inputs.
    NoInputs,
    /// A transaction with no outputs.
    NoOutputs,
    /// The same output is listed twice in one transaction.
    DuplicateInput(OutPoint),
    /// The output does not exist or was already spent.
    UnknownInput(OutPoint),
    /// The number of signatures differs from the number of inputs.
    SigCount,
    /// The signature on this input index does not verify.
    BadSignature(usize),
    /// An output with amount zero.
    ZeroOutput,
    /// An amount sum overflowed.
    Overflow,
    /// Outputs exceed inputs.
    Inflation {
        /// Sum of inputs.
        inputs: u64,
        /// Sum of outputs.
        outputs: u64,
    },
    /// A transaction with this id is already in the set.
    DuplicateTxid,
}

/// The set of unspent outputs.
#[derive(Clone, Debug, Default)]
pub struct UtxoSet {
    map: HashMap<OutPoint, Output>,
    known_ids: HashSet<Hash>,
}

impl UtxoSet {
    /// An empty set.
    pub fn new() -> Self {
        Self::default()
    }

    /// Adds a coinbase transaction (no inputs): new coins created by rule.
    pub fn apply_coinbase(&mut self, tx: &Tx) -> Result<(), TxError> {
        if !tx.inputs.is_empty() || !tx.sigs.is_empty() {
            return Err(TxError::SigCount);
        }
        if tx.outputs.is_empty() {
            return Err(TxError::NoOutputs);
        }
        if tx.outputs.iter().any(|o| o.amount == 0) {
            return Err(TxError::ZeroOutput);
        }
        let id = tx.txid();
        if self.known_ids.contains(&id) {
            return Err(TxError::DuplicateTxid);
        }
        tx.outputs
            .iter()
            .try_fold(0u64, |a, o| a.checked_add(o.amount))
            .ok_or(TxError::Overflow)?;
        self.insert_outputs(tx);
        Ok(())
    }

    fn insert_outputs(&mut self, tx: &Tx) {
        let id = tx.txid();
        self.known_ids.insert(id);
        for (i, o) in tx.outputs.iter().enumerate() {
            self.map.insert(
                OutPoint {
                    txid: id,
                    index: i as u32,
                },
                *o,
            );
        }
    }

    /// Validates and applies `tx`; returns the fee. Nothing changes on error.
    pub fn apply(&mut self, tx: &Tx) -> Result<u64, TxError> {
        if tx.inputs.is_empty() {
            return Err(TxError::NoInputs);
        }
        if tx.outputs.is_empty() {
            return Err(TxError::NoOutputs);
        }
        if tx.sigs.len() != tx.inputs.len() {
            return Err(TxError::SigCount);
        }
        if tx.outputs.iter().any(|o| o.amount == 0) {
            return Err(TxError::ZeroOutput);
        }
        let mut seen = HashSet::new();
        for i in &tx.inputs {
            if !seen.insert(*i) {
                return Err(TxError::DuplicateInput(*i));
            }
        }
        let id = tx.txid();
        if self.known_ids.contains(&id) {
            return Err(TxError::DuplicateTxid);
        }
        let sighash = tx.sighash();
        let mut total_in: u64 = 0;
        for (n, i) in tx.inputs.iter().enumerate() {
            let out = self.map.get(i).ok_or(TxError::UnknownInput(*i))?;
            if !verify_sig(&out.owner, &sighash, &tx.sigs[n]) {
                return Err(TxError::BadSignature(n));
            }
            total_in = total_in.checked_add(out.amount).ok_or(TxError::Overflow)?;
        }
        let total_out = tx
            .outputs
            .iter()
            .try_fold(0u64, |a, o| a.checked_add(o.amount))
            .ok_or(TxError::Overflow)?;
        if total_out > total_in {
            return Err(TxError::Inflation {
                inputs: total_in,
                outputs: total_out,
            });
        }
        for i in &tx.inputs {
            self.map.remove(i);
        }
        self.insert_outputs(tx);
        Ok(total_in - total_out)
    }

    /// The unspent output at `p`, if any.
    pub fn get(&self, p: &OutPoint) -> Option<&Output> {
        self.map.get(p)
    }

    /// Sum of all unspent outputs owned by `owner`.
    pub fn balance(&self, owner: &PubKey) -> u128 {
        self.map
            .values()
            .filter(|o| &o.owner == owner)
            .map(|o| o.amount as u128)
            .sum()
    }

    /// Sum of every unspent output.
    pub fn total(&self) -> u128 {
        self.map.values().map(|o| o.amount as u128).sum()
    }

    /// Number of unspent outputs.
    pub fn len(&self) -> usize {
        self.map.len()
    }

    /// Whether there are no unspent outputs.
    pub fn is_empty(&self) -> bool {
        self.map.is_empty()
    }
}

// -------------------------------------------------------------- account model

/// A signed transfer between accounts.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Transfer {
    /// Which network this transfer is for.
    pub chain_id: u32,
    /// Sender (and signer).
    pub from: PubKey,
    /// Recipient.
    pub to: PubKey,
    /// Amount to move.
    pub amount: u64,
    /// Must equal the sender's current nonce.
    pub nonce: u64,
    /// Signature over [`Transfer::signing_bytes`].
    pub sig: [u8; 64],
}

const TRANSFER_DOMAIN: &[u8] = b"lab/account/transfer/v1";

impl Transfer {
    /// The bytes that are signed (everything except the signature).
    pub fn signing_bytes(&self) -> Vec<u8> {
        let mut v = Vec::new();
        v.extend_from_slice(&(TRANSFER_DOMAIN.len() as u64).to_be_bytes());
        v.extend_from_slice(TRANSFER_DOMAIN);
        v.extend_from_slice(&self.chain_id.to_be_bytes());
        v.extend_from_slice(&self.from);
        v.extend_from_slice(&self.to);
        v.extend_from_slice(&self.amount.to_be_bytes());
        v.extend_from_slice(&self.nonce.to_be_bytes());
        v
    }

    /// Creates and signs a transfer from `key`'s account.
    pub fn new(key: &SigningKey, chain_id: u32, to: PubKey, amount: u64, nonce: u64) -> Self {
        let mut t = Transfer {
            chain_id,
            from: key.verifying_key().to_bytes(),
            to,
            amount,
            nonce,
            sig: [0; 64],
        };
        t.sig = key.sign(&t.signing_bytes()).to_bytes();
        t
    }
}

/// Why a transfer was refused.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum TransferError {
    /// The transfer names another network.
    WrongChain,
    /// The signature does not verify.
    BadSignature,
    /// Amount zero.
    ZeroAmount,
    /// The nonce is not the sender's next nonce (replay or gap).
    BadNonce {
        /// Nonce the account expects.
        expected: u64,
        /// Nonce the transfer carries.
        got: u64,
    },
    /// The sender's balance is too small.
    InsufficientFunds,
    /// The recipient's balance would overflow.
    Overflow,
}

/// Balance and next-nonce of one account.
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub struct Account {
    /// Current balance.
    pub balance: u64,
    /// Nonce the next transfer must carry.
    pub nonce: u64,
}

/// A ledger of accounts for one network.
#[derive(Clone, Debug)]
pub struct Accounts {
    chain_id: u32,
    map: HashMap<PubKey, Account>,
}

impl Accounts {
    /// An empty ledger for network `chain_id`.
    pub fn new(chain_id: u32) -> Self {
        Accounts {
            chain_id,
            map: HashMap::new(),
        }
    }

    /// Credits `amount` out of thin air (genesis allocation in this toy).
    pub fn fund(&mut self, who: PubKey, amount: u64) {
        let a = self.map.entry(who).or_default();
        a.balance = a.balance.checked_add(amount).expect("genesis overflow");
    }

    /// The account, or an all-zero default.
    pub fn get(&self, who: &PubKey) -> Account {
        self.map.get(who).copied().unwrap_or_default()
    }

    /// Sum of all balances.
    pub fn total(&self) -> u128 {
        self.map.values().map(|a| a.balance as u128).sum()
    }

    /// Validates and applies a transfer. Nothing changes on error.
    pub fn apply(&mut self, t: &Transfer) -> Result<(), TransferError> {
        if t.chain_id != self.chain_id {
            return Err(TransferError::WrongChain);
        }
        if !verify_sig(&t.from, &t.signing_bytes(), &t.sig) {
            return Err(TransferError::BadSignature);
        }
        if t.amount == 0 {
            return Err(TransferError::ZeroAmount);
        }
        let from = self.get(&t.from);
        if t.nonce != from.nonce {
            return Err(TransferError::BadNonce {
                expected: from.nonce,
                got: t.nonce,
            });
        }
        let from_balance = from
            .balance
            .checked_sub(t.amount)
            .ok_or(TransferError::InsufficientFunds)?;
        if t.from == t.to {
            self.map.insert(
                t.from,
                Account {
                    balance: from.balance,
                    nonce: from.nonce + 1,
                },
            );
            return Ok(());
        }
        let to = self.get(&t.to);
        let to_balance = to
            .balance
            .checked_add(t.amount)
            .ok_or(TransferError::Overflow)?;
        self.map.insert(
            t.from,
            Account {
                balance: from_balance,
                nonce: from.nonce + 1,
            },
        );
        self.map.insert(
            t.to,
            Account {
                balance: to_balance,
                nonce: to.nonce,
            },
        );
        Ok(())
    }
}
```

### `src/chain.rs`

```rust
//! A toy proof-of-work block store with a cumulative-work fork-choice rule.
//! Fixed difficulty, no retargeting, no transactions semantics: a block carries
//! opaque byte strings and commits to them with a Merkle root.

use std::collections::HashMap;

use crate::hash::{Hash, leading_zero_bits, sha256};
use crate::merkle;

/// A block header.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct Header {
    /// Distance from genesis.
    pub height: u64,
    /// Hash of the parent header.
    pub prev: Hash,
    /// Merkle root of the block's transactions.
    pub merkle_root: Hash,
    /// Seconds since some epoch, chosen by the miner.
    pub time: u64,
    /// Required leading zero bits of the header hash.
    pub bits: u32,
    /// Free counter varied by the miner.
    pub nonce: u64,
}

impl Header {
    /// The header hash (fixed-width fields, so the encoding is unambiguous).
    pub fn hash(&self) -> Hash {
        sha256(&[
            b"lab/header/v1",
            &self.height.to_be_bytes(),
            &self.prev,
            &self.merkle_root,
            &self.time.to_be_bytes(),
            &self.bits.to_be_bytes(),
            &self.nonce.to_be_bytes(),
        ])
    }

    /// Whether the hash meets the header's own `bits` target.
    pub fn meets_target(&self) -> bool {
        leading_zero_bits(&self.hash()) >= self.bits
    }
}

/// A header plus the transactions it commits to.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Block {
    /// The header.
    pub header: Header,
    /// Opaque transactions.
    pub txs: Vec<Vec<u8>>,
}

/// Searches nonces `0..max_tries`; returns the solved header and the number of
/// hashes tried (the successful nonce plus one).
pub fn mine(mut h: Header, max_tries: u64) -> Option<(Header, u64)> {
    for n in 0..max_tries {
        h.nonce = n;
        if h.meets_target() {
            return Some((h, n + 1));
        }
    }
    None
}

/// Builds and mines a block on top of `prev` (`None` for genesis).
pub fn mine_block(prev: Option<&Header>, txs: Vec<Vec<u8>>, time: u64, bits: u32) -> Option<Block> {
    let refs: Vec<&[u8]> = txs.iter().map(|t| t.as_slice()).collect();
    let template = Header {
        height: prev.map_or(0, |p| p.height + 1),
        prev: prev.map_or([0; 32], |p| p.hash()),
        merkle_root: merkle::root_of(&refs),
        time,
        bits,
        nonce: 0,
    };
    let (header, _) = mine(template, 1 << 26)?;
    Some(Block { header, txs })
}

/// Why a block was refused.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum BlockError {
    /// The parent is unknown.
    Orphan(Hash),
    /// The block is already stored.
    Duplicate,
    /// `height` is not parent height + 1.
    BadHeight,
    /// The header's `bits` differ from the network's fixed difficulty.
    WrongDifficulty,
    /// The header hash does not meet its target.
    InsufficientWork,
    /// The Merkle root does not match the transactions.
    BadMerkleRoot,
    /// The time is not later than the parent's.
    TimeNotIncreasing,
    /// The time is too far ahead of the local clock.
    TooFarInFuture,
    /// A genesis block was offered where one already exists, or with a wrong shape.
    BadGenesis,
}

/// What adding a block did to the best chain.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Outcome {
    /// The block extends the current best chain.
    Extended,
    /// The block is stored on a side branch that has less (or equal) work.
    SideBranch,
    /// The best chain switched; `depth` blocks of the old chain were abandoned.
    Reorg {
        /// Number of old-chain blocks that are no longer on the best chain.
        depth: u64,
    },
}

/// Work represented by one block of `bits` difficulty: `2^bits`.
pub fn work(bits: u32) -> u128 {
    1u128 << bits
}

struct Entry {
    block: Block,
    total_work: u128,
}

/// Block store with fork choice by most cumulative work (first seen wins ties).
pub struct Store {
    bits: u32,
    max_future: u64,
    blocks: HashMap<Hash, Entry>,
    tip: Hash,
}

impl Store {
    /// Starts a store from `genesis`, which must be a mined block at height 0
    /// with the all-zero parent, `bits` difficulty and an honest Merkle root.
    /// `bits` must be at most 64.
    pub fn new(genesis: Block, bits: u32, max_future: u64) -> Result<Self, BlockError> {
        assert!(bits <= 64, "difficulty above 64 bits is not supported");
        let h = genesis.header;
        let refs: Vec<&[u8]> = genesis.txs.iter().map(|t| t.as_slice()).collect();
        if h.height != 0
            || h.prev != [0; 32]
            || h.bits != bits
            || !h.meets_target()
            || h.merkle_root != merkle::root_of(&refs)
        {
            return Err(BlockError::BadGenesis);
        }
        let id = h.hash();
        let mut blocks = HashMap::new();
        blocks.insert(
            id,
            Entry {
                block: genesis,
                total_work: work(bits),
            },
        );
        Ok(Store {
            bits,
            max_future,
            blocks,
            tip: id,
        })
    }

    /// Hash of the best tip.
    pub fn tip(&self) -> Hash {
        self.tip
    }

    /// Height of the best tip.
    pub fn height(&self) -> u64 {
        self.blocks[&self.tip].block.header.height
    }

    /// Cumulative work of the best tip.
    pub fn best_work(&self) -> u128 {
        self.blocks[&self.tip].total_work
    }

    /// Number of stored blocks (all branches).
    pub fn len(&self) -> usize {
        self.blocks.len()
    }

    /// Always false: a store holds at least the genesis block.
    pub fn is_empty(&self) -> bool {
        false
    }

    /// The stored block with this hash.
    pub fn get(&self, id: &Hash) -> Option<&Block> {
        self.blocks.get(id).map(|e| &e.block)
    }

    /// Hashes from the best tip back to genesis (tip first).
    pub fn best_chain(&self) -> Vec<Hash> {
        let mut out = vec![self.tip];
        let mut cur = self.tip;
        while cur != [0; 32] {
            let prev = self.blocks[&cur].block.header.prev;
            if prev == [0; 32] {
                break;
            }
            out.push(prev);
            cur = prev;
        }
        out
    }

    /// Confirmations of `id` on the best chain (the tip has 1), or `None` if
    /// the block is not on the best chain.
    pub fn confirmations(&self, id: &Hash) -> Option<u64> {
        let chain = self.best_chain();
        chain.iter().position(|h| h == id).map(|p| p as u64 + 1)
    }

    /// Validates and stores `block`; `now` is the local clock in seconds.
    pub fn add(&mut self, block: Block, now: u64) -> Result<Outcome, BlockError> {
        let h = block.header;
        let id = h.hash();
        if self.blocks.contains_key(&id) {
            return Err(BlockError::Duplicate);
        }
        let parent = self.blocks.get(&h.prev).ok_or(BlockError::Orphan(h.prev))?;
        if h.height != parent.block.header.height + 1 {
            return Err(BlockError::BadHeight);
        }
        if h.bits != self.bits {
            return Err(BlockError::WrongDifficulty);
        }
        if !h.meets_target() {
            return Err(BlockError::InsufficientWork);
        }
        let refs: Vec<&[u8]> = block.txs.iter().map(|t| t.as_slice()).collect();
        if h.merkle_root != merkle::root_of(&refs) {
            return Err(BlockError::BadMerkleRoot);
        }
        if h.time <= parent.block.header.time {
            return Err(BlockError::TimeNotIncreasing);
        }
        if h.time > now.saturating_add(self.max_future) {
            return Err(BlockError::TooFarInFuture);
        }
        let total_work = parent.total_work + work(h.bits);
        let old_tip = self.tip;
        let extends = h.prev == old_tip;
        self.blocks.insert(id, Entry { block, total_work });
        if total_work > self.blocks[&old_tip].total_work {
            self.tip = id;
            if extends {
                return Ok(Outcome::Extended);
            }
            // reorg: walk both branches back to the fork point
            let new_chain: std::collections::HashSet<Hash> = {
                let mut s = std::collections::HashSet::new();
                let mut c = id;
                loop {
                    s.insert(c);
                    let p = self.blocks[&c].block.header.prev;
                    if p == [0; 32] {
                        break;
                    }
                    c = p;
                }
                s
            };
            let mut depth = 0;
            let mut c = old_tip;
            while !new_chain.contains(&c) {
                depth += 1;
                c = self.blocks[&c].block.header.prev;
            }
            Ok(Outcome::Reorg { depth })
        } else {
            Ok(Outcome::SideBranch)
        }
    }
}
```

### `src/quorum.rs`

```rust
//! Quorum arithmetic for crash-fault and Byzantine-fault agreement, and the
//! "attacker catches up" probability for proof-of-work confirmations.

/// Largest number of faulty nodes a Byzantine protocol tolerates with `n` nodes:
/// `floor((n - 1) / 3)` (zero for `n <= 1`).
pub fn max_faulty(n: usize) -> usize {
    n.saturating_sub(1) / 3
}

/// Smallest majority: `floor(n / 2) + 1`.
pub fn majority(n: usize) -> usize {
    n / 2 + 1
}

/// Byzantine quorum size `floor(2n / 3) + 1`.
pub fn bft_quorum(n: usize) -> usize {
    (2 * n) / 3 + 1
}

/// Smallest possible overlap of two quorums of size `q` among `n` nodes.
pub fn min_overlap(n: usize, q: usize) -> usize {
    (2 * q).saturating_sub(n)
}

/// Whether any two `q`-quorums must share at least one *honest* node when up to
/// `f` nodes are Byzantine (overlap greater than `f`).
pub fn quorums_share_an_honest_node(n: usize, q: usize, f: usize) -> bool {
    min_overlap(n, q) > f
}

/// Whether the honest nodes alone can form a quorum (liveness).
pub fn honest_can_form_quorum(n: usize, q: usize, f: usize) -> bool {
    n.saturating_sub(f) >= q
}

/// Votes a value can collect: `honest` honest votes plus every Byzantine node
/// (which may vote for both of two conflicting values).
pub fn votes_with_equivocators(honest_for_value: usize, f: usize) -> usize {
    honest_for_value + f
}

/// Brute force over every split of the honest nodes between two conflicting
/// values (each honest node votes for at most one): can *both* reach `q` votes?
pub fn conflicting_decisions_possible(n: usize, q: usize, f: usize) -> bool {
    let honest = n.saturating_sub(f);
    (0..=honest).any(|a| {
        (0..=honest - a)
            .any(|b| votes_with_equivocators(a, f) >= q && votes_with_equivocators(b, f) >= q)
    })
}

/// Probability that an attacker with fraction `q` of the hash power ever catches
/// up from `z` blocks behind, in the simple model of the Bitcoin paper (from
/// memory): `1` if `q >= 1/2`, else `(q / (1 - q))^z`.
pub fn catch_up_probability(q: f64, z: u32) -> f64 {
    assert!((0.0..=1.0).contains(&q), "q must be a fraction");
    if q >= 0.5 {
        1.0
    } else {
        (q / (1.0 - q)).powi(z as i32)
    }
}
```

### `tests/hash.rs`

```rust
use merklelab::hash::{hex, leading_zero_bits, sha256};

#[test]
fn sha256_hashes_the_concatenation_of_its_parts() {
    assert_eq!(
        hex(&sha256(&[b"abc"])),
        "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad"
    );
    assert_eq!(sha256(&[b"a", b"bc"]), sha256(&[b"abc"]));
    assert_eq!(sha256(&[b"", b"abc", b""]), sha256(&[b"abc"]));
}

#[test]
fn hex_pads_every_byte_to_two_digits() {
    assert_eq!(hex(&[0x00, 0x0a, 0xff, 0x10]), "000aff10");
    assert_eq!(hex(&[]), "");
}

#[test]
fn leading_zero_bits_counts_bits_not_bytes() {
    let mut h = [0xffu8; 32];
    assert_eq!(leading_zero_bits(&h), 0);
    h[0] = 0x7f;
    assert_eq!(leading_zero_bits(&h), 1);
    h[0] = 0x01;
    assert_eq!(leading_zero_bits(&h), 7);
    h[0] = 0x00;
    h[1] = 0x80;
    assert_eq!(leading_zero_bits(&h), 8);
    h[1] = 0x0f;
    assert_eq!(leading_zero_bits(&h), 12);
    assert_eq!(leading_zero_bits(&[0; 32]), 256);
    let mut last = [0u8; 32];
    last[31] = 1;
    assert_eq!(leading_zero_bits(&last), 255);
}
```

### `tests/merkle.rs`

```rust
use merklelab::hash::{hex, sha256};
use merklelab::merkle::{
    Proof, Tree, duplicating_root, empty_root, leaf_hash, node_hash, root_of, unprefixed_root,
    verify,
};

fn leaves(n: usize) -> Vec<Vec<u8>> {
    (0..n).map(|i| format!("leaf-{i}").into_bytes()).collect()
}

fn refs(v: &[Vec<u8>]) -> Vec<&[u8]> {
    v.iter().map(|x| x.as_slice()).collect()
}

fn tree(n: usize) -> (Vec<Vec<u8>>, Tree) {
    let l = leaves(n);
    let t = Tree::new(&refs(&l));
    (l, t)
}

#[test]
fn roots_match_an_independent_python_implementation() {
    // Computed with a separate RFC 6962 style implementation in Python.
    let expected = [
        (
            0,
            "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
        ),
        (
            1,
            "305df59f9590c3c9ac63d2b2743c388e3792449078cebf7fb3dbe6471643b2b7",
        ),
        (
            2,
            "60a53eed0de87a90c8e59427c59c46253c33a76a09502a51801300927b7e6bdc",
        ),
        (
            3,
            "cf763a041c81ceef1578a6083f75c61bef2e0014f2a3e683a97fcfca5be7f19a",
        ),
        (
            5,
            "00d21829a5503145348abcf712513eacf2a274211ad83e970202bb5b6d80b286",
        ),
        (
            7,
            "0b007fb915eb9b2a146f54b1c86ec53b664f8e455b7660b0b6ee13edc0d921c0",
        ),
        (
            8,
            "ca6b7b3e674ac86c1027b59c87c064fc3bc27b313294c75f83bd05fdd13f0dcf",
        ),
        (
            13,
            "a8ef4844c8e1d5ba49c811cdb86e95791f5d32ca7d9709afda28fdf65e949a53",
        ),
    ];
    for (n, want) in expected {
        let (_, t) = tree(n);
        assert_eq!(hex(&t.root()), want, "n = {n}");
    }
}

#[test]
fn small_shapes_are_what_the_definition_says() {
    let (l, t) = tree(3);
    let a = leaf_hash(&l[0]);
    let b = leaf_hash(&l[1]);
    let c = leaf_hash(&l[2]);
    // three leaves split as (2, 1): the odd leaf is NOT duplicated
    assert_eq!(t.root(), node_hash(&node_hash(&a, &b), &c));
    assert_eq!(tree(1).1.root(), leaf_hash(&l[0]));
    assert_eq!(Tree::new(&[]).root(), empty_root());
    assert!(Tree::new(&[]).is_empty());
    assert_eq!(t.len(), 3);
    assert_eq!(root_of(&refs(&l)), t.root());
}

#[test]
fn every_leaf_of_every_tree_up_to_forty_leaves_has_a_valid_proof() {
    for n in 1..=40 {
        let (l, t) = tree(n);
        let root = t.root();
        let max_len = (n as f64).log2().ceil() as usize;
        for (i, leaf) in l.iter().enumerate() {
            let p = t.prove(i).expect("in range");
            assert_eq!((p.index, p.size), (i, n));
            assert!(
                p.path.len() <= max_len,
                "n={n} i={i}: {} > {max_len}",
                p.path.len()
            );
            assert!(verify(&root, leaf, &p), "n={n} i={i}");
        }
    }
}

#[test]
fn proof_lengths_are_logarithmic() {
    let (_, t) = tree(1000);
    assert!(t.prove(0).unwrap().path.len() <= 10);
    assert!(t.prove(999).unwrap().path.len() <= 10);
    assert_eq!(tree(1).1.prove(0).unwrap().path.len(), 0);
    assert_eq!(tree(2).1.prove(1).unwrap().path.len(), 1);
}

#[test]
fn out_of_range_proofs_do_not_exist() {
    let (_, t) = tree(5);
    assert!(t.prove(5).is_none());
    assert!(Tree::new(&[]).prove(0).is_none());
}

#[test]
fn any_change_to_the_leaf_the_proof_or_the_claim_is_rejected() {
    for n in 2..=17 {
        let (l, t) = tree(n);
        let root = t.root();
        for (i, leaf) in l.iter().enumerate() {
            let good = t.prove(i).unwrap();
            assert!(
                !verify(&root, b"not the leaf", &good),
                "n={n} i={i} wrong leaf"
            );
            assert!(!verify(&[0; 32], leaf, &good), "n={n} i={i} wrong root");
            for k in 0..good.path.len() {
                let mut bad = good.clone();
                bad.path[k][0] ^= 1;
                assert!(!verify(&root, leaf, &bad), "n={n} i={i} path[{k}]");
            }
            let mut short = good.clone();
            short.path.pop();
            assert!(!verify(&root, leaf, &short), "n={n} i={i} truncated");
            let mut long = good.clone();
            long.path.push([7; 32]);
            assert!(!verify(&root, leaf, &long), "n={n} i={i} extended");
            let other = (i + 1) % n;
            let moved = Proof {
                index: other,
                ..good.clone()
            };
            assert!(
                !verify(&root, leaf, &moved),
                "n={n} i={i} claimed at {other}"
            );
        }
    }
}

#[test]
fn degenerate_proofs_are_refused() {
    let (l, t) = tree(4);
    let root = t.root();
    let p = t.prove(1).unwrap();
    assert!(!verify(
        &root,
        &l[1],
        &Proof {
            size: 0,
            ..p.clone()
        }
    ));
    assert!(!verify(
        &root,
        &l[1],
        &Proof {
            index: 4,
            ..p.clone()
        }
    ));
    assert!(!verify(
        &root,
        &l[1],
        &Proof {
            index: usize::MAX,
            ..p
        }
    ));
    // a one-leaf tree: only the empty path works
    let one = tree(1);
    let pr = one.1.prove(0).unwrap();
    assert!(verify(&one.1.root(), &one.0[0], &pr));
    assert!(!verify(
        &one.1.root(),
        &one.0[0],
        &Proof {
            path: vec![[0; 32]],
            ..pr
        }
    ));
}

#[test]
fn order_and_content_change_the_root() {
    let (l, t) = tree(6);
    let mut swapped = l.clone();
    swapped.swap(1, 4);
    assert_ne!(Tree::new(&refs(&swapped)).root(), t.root());
    let mut more = l.clone();
    more.push(b"extra".to_vec());
    assert_ne!(Tree::new(&refs(&more)).root(), t.root());
    let mut edited = l.clone();
    edited[3].push(b'!');
    assert_ne!(Tree::new(&refs(&edited)).root(), t.root());
}

#[test]
fn leaves_and_nodes_are_hashed_with_different_prefixes() {
    let a = leaf_hash(b"a");
    let b = leaf_hash(b"b");
    // a leaf whose content looks like two child hashes is not the node above them
    let mut glued = Vec::new();
    glued.extend_from_slice(&a);
    glued.extend_from_slice(&b);
    assert_ne!(leaf_hash(&glued), node_hash(&a, &b));
    assert_ne!(root_of(&[&glued]), root_of(&[b"a", b"b"]));
}

#[test]
fn without_prefixes_an_interior_node_can_pose_as_a_leaf() {
    // Second-preimage attack on a tree that hashes leaves and nodes alike.
    let honest = unprefixed_root(&[b"pay alice 5", b"pay bob 7"]);
    let mut forged_leaf = Vec::new();
    forged_leaf.extend_from_slice(&sha256(&[b"pay alice 5"]));
    forged_leaf.extend_from_slice(&sha256(&[b"pay bob 7"]));
    // one 64-byte "leaf" produces the same root as the two real leaves
    assert_eq!(unprefixed_root(&[&forged_leaf]), honest);
}

#[test]
fn duplicating_the_last_node_makes_different_lists_share_a_root() {
    let a = duplicating_root(&[b"a", b"b", b"c"]);
    let b = duplicating_root(&[b"a", b"b", b"c", b"c"]);
    assert_eq!(a, b, "three and four leaves, same root");
    let five = duplicating_root(&[b"a", b"b", b"c", b"d", b"e"]);
    let six = duplicating_root(&[b"a", b"b", b"c", b"d", b"e", b"e"]);
    assert_eq!(five, six);
    // the RFC 6962 layout keeps them apart
    assert_ne!(
        root_of(&[b"a", b"b", b"c"]),
        root_of(&[b"a", b"b", b"c", b"c"])
    );
    assert_ne!(
        root_of(&[b"a", b"b", b"c", b"d", b"e"]),
        root_of(&[b"a", b"b", b"c", b"d", b"e", b"e"])
    );
}

#[test]
fn the_flawed_roots_agree_with_themselves_on_edge_cases() {
    assert_eq!(duplicating_root(&[]), empty_root());
    assert_eq!(duplicating_root(&[b"x"]), sha256(&[b"x"]));
    assert_eq!(unprefixed_root(&[]), empty_root());
    assert_eq!(unprefixed_root(&[b"x"]), sha256(&[b"x"]));
    assert_ne!(
        duplicating_root(&[b"a", b"b"]),
        duplicating_root(&[b"b", b"a"])
    );
}

#[test]
fn the_root_alone_does_not_commit_to_the_tree_size() {
    // A proof made for 3 leaves also verifies when it claims 4 leaves, because
    // the root of a 3-leaf tree has the same shape as one particular 4-leaf walk.
    // So the size must be authenticated separately (e.g. signed with the root).
    let (l, t) = tree(3);
    let p = t.prove(0).unwrap();
    assert!(verify(&t.root(), &l[0], &p));
    assert!(verify(&t.root(), &l[0], &Proof { size: 4, ..p }));
    // but many size changes are caught by the path length alone
    let (l4, t4) = tree(4);
    let p4 = t4.prove(0).unwrap();
    assert!(!verify(
        &t4.root(),
        &l4[0],
        &Proof {
            size: 5,
            ..p4.clone()
        }
    ));
    assert!(!verify(&t4.root(), &l4[0], &Proof { size: 2, ..p4 }));
}

#[test]
fn an_index_equal_to_the_size_is_refused_even_when_the_path_is_empty() {
    // For a one-leaf tree the "proof" is empty, so only the range check can
    // tell leaf number 1 of 1 from leaf number 0 of 1.
    let (l, t) = tree(1);
    let p = Proof {
        index: 1,
        size: 1,
        path: vec![],
    };
    assert!(!verify(&t.root(), &l[0], &p));
    assert!(verify(&t.root(), &l[0], &Proof { index: 0, ..p }));
}
```

### `tests/ledger.rs`

```rust
use ed25519_dalek::SigningKey;
use merklelab::ledger::*;

fn key(n: u8) -> SigningKey {
    SigningKey::from_bytes(&[n; 32])
}

fn pk(k: &SigningKey) -> PubKey {
    k.verifying_key().to_bytes()
}

fn coinbase(owner: PubKey, amount: u64, extra: u64) -> Tx {
    Tx {
        inputs: vec![],
        outputs: vec![Output { owner, amount }],
        extra,
        sigs: vec![],
    }
}

fn spend(from: &[(&SigningKey, OutPoint)], outs: &[(PubKey, u64)], extra: u64) -> Tx {
    let mut tx = Tx {
        inputs: from.iter().map(|(_, p)| *p).collect(),
        outputs: outs
            .iter()
            .map(|(o, a)| Output {
                owner: *o,
                amount: *a,
            })
            .collect(),
        extra,
        sigs: vec![],
    };
    let keys: Vec<&SigningKey> = from.iter().map(|(k, _)| *k).collect();
    tx.sign(&keys);
    tx
}

fn funded(owner: &SigningKey, amount: u64) -> (UtxoSet, OutPoint) {
    let mut set = UtxoSet::new();
    let cb = coinbase(pk(owner), amount, 0);
    set.apply_coinbase(&cb).unwrap();
    (
        set,
        OutPoint {
            txid: cb.txid(),
            index: 0,
        },
    )
}

// ------------------------------------------------------------------ UTXO

#[test]
fn a_signed_spend_moves_value_and_reports_the_fee() {
    let (alice, bob) = (key(1), key(2));
    let (mut set, op) = funded(&alice, 100);
    let tx = spend(&[(&alice, op)], &[(pk(&bob), 60), (pk(&alice), 35)], 0);
    assert_eq!(set.apply(&tx), Ok(5));
    assert_eq!(set.balance(&pk(&bob)), 60);
    assert_eq!(set.balance(&pk(&alice)), 35);
    assert_eq!(
        set.total(),
        95,
        "the fee left the set (it belongs to the miner)"
    );
    assert!(set.get(&op).is_none(), "the spent output is gone");
    assert_eq!(set.len(), 2);
    assert!(!set.is_empty());
}

#[test]
fn spending_the_same_output_twice_is_refused() {
    let (alice, bob) = (key(1), key(2));
    let (mut set, op) = funded(&alice, 100);
    let first = spend(&[(&alice, op)], &[(pk(&bob), 100)], 0);
    let second = spend(&[(&alice, op)], &[(pk(&alice), 100)], 1);
    set.apply(&first).unwrap();
    assert_eq!(set.apply(&second), Err(TxError::UnknownInput(op)));
    assert_eq!(set.balance(&pk(&bob)), 100);
}

#[test]
fn the_same_output_twice_inside_one_transaction_is_refused() {
    let (alice, bob) = (key(1), key(2));
    let (mut set, op) = funded(&alice, 100);
    let tx = spend(&[(&alice, op), (&alice, op)], &[(pk(&bob), 200)], 0);
    assert_eq!(set.apply(&tx), Err(TxError::DuplicateInput(op)));
    assert_eq!(set.total(), 100);
}

#[test]
fn outputs_may_not_exceed_inputs() {
    let (alice, bob) = (key(1), key(2));
    let (mut set, op) = funded(&alice, 100);
    let tx = spend(&[(&alice, op)], &[(pk(&bob), 101)], 0);
    assert_eq!(
        set.apply(&tx),
        Err(TxError::Inflation {
            inputs: 100,
            outputs: 101
        })
    );
    let exact = spend(&[(&alice, op)], &[(pk(&bob), 100)], 0);
    assert_eq!(set.apply(&exact), Ok(0), "zero fee is allowed");
}

#[test]
fn sums_are_checked_not_wrapped() {
    let (alice, bob) = (key(1), key(2));
    // outputs that overflow u64 when added would wrap to a small number
    let (mut set, op) = funded(&alice, 100);
    let tx = spend(&[(&alice, op)], &[(pk(&bob), u64::MAX), (pk(&bob), 101)], 0);
    assert_eq!(set.apply(&tx), Err(TxError::Overflow));
    // inputs that overflow
    let mut set = UtxoSet::new();
    let a = coinbase(pk(&alice), 1 << 63, 1);
    let b = coinbase(pk(&alice), 1 << 63, 2);
    set.apply_coinbase(&a).unwrap();
    set.apply_coinbase(&b).unwrap();
    let ins = [
        (
            &alice,
            OutPoint {
                txid: a.txid(),
                index: 0,
            },
        ),
        (
            &alice,
            OutPoint {
                txid: b.txid(),
                index: 0,
            },
        ),
    ];
    let tx = spend(&ins, &[(pk(&bob), 1)], 0);
    assert_eq!(set.apply(&tx), Err(TxError::Overflow));
    assert_eq!(
        set.total(),
        1u128 << 64,
        "u128 total shows nothing was lost"
    );
}

#[test]
fn only_the_owner_can_spend() {
    let (alice, bob, mallory) = (key(1), key(2), key(3));
    let (mut set, op) = funded(&alice, 100);
    let theft = spend(&[(&mallory, op)], &[(pk(&mallory), 100)], 0);
    assert_eq!(set.apply(&theft), Err(TxError::BadSignature(0)));
    // a valid signature, then the outputs are edited
    let mut tx = spend(&[(&alice, op)], &[(pk(&bob), 10)], 0);
    tx.outputs[0].owner = pk(&mallory);
    assert_eq!(set.apply(&tx), Err(TxError::BadSignature(0)));
    let mut tx = spend(&[(&alice, op)], &[(pk(&bob), 10)], 0);
    tx.outputs[0].amount = 99;
    assert_eq!(set.apply(&tx), Err(TxError::BadSignature(0)));
    let mut tx = spend(&[(&alice, op)], &[(pk(&bob), 10)], 0);
    tx.extra = 7;
    assert_eq!(set.apply(&tx), Err(TxError::BadSignature(0)));
    assert_eq!(set.balance(&pk(&alice)), 100);
}

#[test]
fn the_second_input_can_be_the_bad_one() {
    let (alice, bob, mallory) = (key(1), key(2), key(3));
    let mut set = UtxoSet::new();
    let a = coinbase(pk(&alice), 10, 1);
    let b = coinbase(pk(&bob), 10, 2);
    set.apply_coinbase(&a).unwrap();
    set.apply_coinbase(&b).unwrap();
    let ins = [
        (
            &alice,
            OutPoint {
                txid: a.txid(),
                index: 0,
            },
        ),
        (
            &mallory,
            OutPoint {
                txid: b.txid(),
                index: 0,
            },
        ),
    ];
    let tx = spend(&ins, &[(pk(&mallory), 20)], 0);
    assert_eq!(set.apply(&tx), Err(TxError::BadSignature(1)));
    assert_eq!(set.total(), 20);
}

#[test]
fn structural_errors() {
    let (alice, bob) = (key(1), key(2));
    let (mut set, op) = funded(&alice, 100);
    let empty_in = Tx {
        inputs: vec![],
        outputs: vec![Output {
            owner: pk(&bob),
            amount: 1,
        }],
        extra: 0,
        sigs: vec![],
    };
    assert_eq!(set.apply(&empty_in), Err(TxError::NoInputs));
    let mut no_out = spend(&[(&alice, op)], &[], 0);
    no_out.sigs.truncate(1);
    assert_eq!(set.apply(&no_out), Err(TxError::NoOutputs));
    let mut missing_sig = spend(&[(&alice, op)], &[(pk(&bob), 1)], 0);
    missing_sig.sigs.clear();
    assert_eq!(set.apply(&missing_sig), Err(TxError::SigCount));
    let mut extra_sig = spend(&[(&alice, op)], &[(pk(&bob), 1)], 0);
    extra_sig.sigs.push([0; 64]);
    assert_eq!(set.apply(&extra_sig), Err(TxError::SigCount));
    let zero = spend(&[(&alice, op)], &[(pk(&bob), 0)], 0);
    assert_eq!(set.apply(&zero), Err(TxError::ZeroOutput));
    let unknown = OutPoint {
        txid: [9; 32],
        index: 0,
    };
    let tx = spend(&[(&alice, unknown)], &[(pk(&bob), 1)], 0);
    assert_eq!(set.apply(&tx), Err(TxError::UnknownInput(unknown)));
    let bad_index = OutPoint {
        txid: op.txid,
        index: 1,
    };
    let tx = spend(&[(&alice, bad_index)], &[(pk(&bob), 1)], 0);
    assert_eq!(set.apply(&tx), Err(TxError::UnknownInput(bad_index)));
    assert_eq!(set.total(), 100, "every failure left the set untouched");
}

#[test]
fn coinbase_rules() {
    let alice = key(1);
    let mut set = UtxoSet::new();
    let cb = coinbase(pk(&alice), 50, 0);
    set.apply_coinbase(&cb).unwrap();
    assert_eq!(set.apply_coinbase(&cb), Err(TxError::DuplicateTxid));
    set.apply_coinbase(&coinbase(pk(&alice), 50, 1)).unwrap();
    assert_eq!(set.total(), 100);
    assert_eq!(
        set.apply_coinbase(&coinbase(pk(&alice), 0, 2)),
        Err(TxError::ZeroOutput)
    );
    let none = Tx {
        inputs: vec![],
        outputs: vec![],
        extra: 3,
        sigs: vec![],
    };
    assert_eq!(set.apply_coinbase(&none), Err(TxError::NoOutputs));
    let with_input = Tx {
        inputs: vec![OutPoint {
            txid: [1; 32],
            index: 0,
        }],
        outputs: vec![Output {
            owner: pk(&alice),
            amount: 1,
        }],
        extra: 4,
        sigs: vec![],
    };
    assert_eq!(set.apply_coinbase(&with_input), Err(TxError::SigCount));
    let overflow = Tx {
        inputs: vec![],
        outputs: vec![
            Output {
                owner: pk(&alice),
                amount: u64::MAX,
            },
            Output {
                owner: pk(&alice),
                amount: 1,
            },
        ],
        extra: 5,
        sigs: vec![],
    };
    assert_eq!(set.apply_coinbase(&overflow), Err(TxError::Overflow));
    assert_eq!(set.total(), 100);
}

#[test]
fn the_txid_ignores_signatures_but_the_sighash_does_not_ignore_the_body() {
    let (alice, bob) = (key(1), key(2));
    let (_, op) = funded(&alice, 100);
    let tx = spend(&[(&alice, op)], &[(pk(&bob), 10)], 0);
    let mut resigned = tx.clone();
    resigned.sigs[0][0] ^= 0xff;
    assert_eq!(
        tx.txid(),
        resigned.txid(),
        "a changed signature cannot change the id"
    );
    let mut other = tx.clone();
    other.outputs[0].amount = 11;
    assert_ne!(tx.txid(), other.txid());
    assert_ne!(tx.sighash(), other.sighash());
    assert_ne!(
        tx.txid(),
        tx.sighash(),
        "the two digests use different domain labels"
    );
    let mut moved = tx.clone();
    moved.inputs[0].index = 1;
    assert_ne!(tx.txid(), moved.txid());
    let mut extra = tx.clone();
    extra.extra = 1;
    assert_ne!(tx.txid(), extra.txid());
}

#[test]
fn outputs_locked_to_weak_or_invalid_keys_cannot_be_spent() {
    let alice = key(1);
    let mut identity_sig = [0u8; 64];
    identity_sig[0] = 1;
    for owner in [
        {
            let mut k = [0u8; 32];
            k[0] = 1;
            k
        },
        [0xff; 32],
    ] {
        let mut set = UtxoSet::new();
        let cb = coinbase(owner, 10, 0);
        set.apply_coinbase(&cb).unwrap();
        let op = OutPoint {
            txid: cb.txid(),
            index: 0,
        };
        let tx = Tx {
            inputs: vec![op],
            outputs: vec![Output {
                owner: pk(&alice),
                amount: 10,
            }],
            extra: 0,
            // R = identity point, S = 0: valid for ANY message under a small-order
            // key if the verifier is not strict
            sigs: vec![identity_sig],
        };
        assert_eq!(
            set.apply(&tx),
            Err(TxError::BadSignature(0)),
            "owner {owner:?}"
        );
    }
}

#[test]
fn value_is_conserved_across_a_chain_of_spends() {
    let (a, b, c) = (key(1), key(2), key(3));
    let (mut set, op) = funded(&a, 1000);
    let t1 = spend(&[(&a, op)], &[(pk(&b), 400), (pk(&a), 590)], 0);
    let fee1 = set.apply(&t1).unwrap();
    let t2 = spend(
        &[(
            &b,
            OutPoint {
                txid: t1.txid(),
                index: 0,
            },
        )],
        &[(pk(&c), 399)],
        0,
    );
    let fee2 = set.apply(&t2).unwrap();
    assert_eq!(set.total() + (fee1 + fee2) as u128, 1000);
    assert_eq!((fee1, fee2), (10, 1));
}

// --------------------------------------------------------------- accounts

const CHAIN: u32 = 1;

fn funded_accounts() -> (Accounts, SigningKey, SigningKey) {
    let (a, b) = (key(1), key(2));
    let mut l = Accounts::new(CHAIN);
    l.fund(pk(&a), 100);
    (l, a, b)
}

#[test]
fn a_transfer_moves_balance_and_bumps_the_nonce() {
    let (mut l, a, b) = funded_accounts();
    l.apply(&Transfer::new(&a, CHAIN, pk(&b), 30, 0)).unwrap();
    assert_eq!(
        l.get(&pk(&a)),
        Account {
            balance: 70,
            nonce: 1
        }
    );
    assert_eq!(
        l.get(&pk(&b)),
        Account {
            balance: 30,
            nonce: 0
        }
    );
    assert_eq!(l.total(), 100);
    assert_eq!(l.get(&pk(&key(9))), Account::default());
}

#[test]
fn replaying_a_transfer_fails_on_the_nonce() {
    let (mut l, a, b) = funded_accounts();
    let t = Transfer::new(&a, CHAIN, pk(&b), 30, 0);
    l.apply(&t).unwrap();
    assert_eq!(
        l.apply(&t),
        Err(TransferError::BadNonce {
            expected: 1,
            got: 0
        })
    );
    assert_eq!(l.get(&pk(&b)).balance, 30);
}

#[test]
fn nonces_must_be_used_in_order_without_gaps() {
    let (mut l, a, b) = funded_accounts();
    let t0 = Transfer::new(&a, CHAIN, pk(&b), 10, 0);
    let t1 = Transfer::new(&a, CHAIN, pk(&b), 10, 1);
    let t5 = Transfer::new(&a, CHAIN, pk(&b), 10, 5);
    assert_eq!(
        l.apply(&t1),
        Err(TransferError::BadNonce {
            expected: 0,
            got: 1
        })
    );
    assert_eq!(
        l.apply(&t5),
        Err(TransferError::BadNonce {
            expected: 0,
            got: 5
        })
    );
    l.apply(&t0).unwrap();
    l.apply(&t1).unwrap();
    assert_eq!(l.get(&pk(&a)).nonce, 2);
}

#[test]
fn a_transfer_for_another_network_is_refused() {
    let (mut l, a, b) = funded_accounts();
    let t = Transfer::new(&a, 2, pk(&b), 10, 0);
    assert_eq!(l.apply(&t), Err(TransferError::WrongChain));
    // and the chain id is signed: relabelling it breaks the signature
    let mut relabel = t.clone();
    relabel.chain_id = CHAIN;
    assert_eq!(l.apply(&relabel), Err(TransferError::BadSignature));
}

#[test]
fn every_signed_field_is_covered() {
    let (mut l, a, b) = funded_accounts();
    let good = Transfer::new(&a, CHAIN, pk(&b), 10, 0);
    let mut t = good.clone();
    t.amount = 11;
    assert_eq!(l.apply(&t), Err(TransferError::BadSignature));
    let mut t = good.clone();
    t.to = pk(&key(7));
    assert_eq!(l.apply(&t), Err(TransferError::BadSignature));
    let mut t = good.clone();
    t.nonce = 1;
    assert_eq!(l.apply(&t), Err(TransferError::BadSignature));
    let mut t = good.clone();
    t.from = pk(&key(7));
    assert_eq!(l.apply(&t), Err(TransferError::BadSignature));
    let mut t = good.clone();
    t.sig[63] ^= 1;
    assert_eq!(l.apply(&t), Err(TransferError::BadSignature));
    assert_eq!(l.get(&pk(&a)).balance, 100);
    l.apply(&good).unwrap();
}

#[test]
fn balance_rules_and_atomicity() {
    let (mut l, a, b) = funded_accounts();
    let before = (l.get(&pk(&a)), l.get(&pk(&b)));
    assert_eq!(
        l.apply(&Transfer::new(&a, CHAIN, pk(&b), 101, 0)),
        Err(TransferError::InsufficientFunds)
    );
    assert_eq!(
        l.apply(&Transfer::new(&a, CHAIN, pk(&b), 0, 0)),
        Err(TransferError::ZeroAmount)
    );
    assert_eq!(
        (l.get(&pk(&a)), l.get(&pk(&b))),
        before,
        "failures change nothing"
    );
    l.apply(&Transfer::new(&a, CHAIN, pk(&b), 100, 0)).unwrap();
    assert_eq!(l.get(&pk(&a)).balance, 0);
}

#[test]
fn a_recipient_balance_cannot_overflow() {
    let (a, b) = (key(1), key(2));
    let mut l = Accounts::new(CHAIN);
    l.fund(pk(&a), 10);
    l.fund(pk(&b), u64::MAX - 5);
    let t = Transfer::new(&a, CHAIN, pk(&b), 6, 0);
    assert_eq!(l.apply(&t), Err(TransferError::Overflow));
    assert_eq!(
        l.get(&pk(&a)),
        Account {
            balance: 10,
            nonce: 0
        },
        "sender untouched"
    );
    l.apply(&Transfer::new(&a, CHAIN, pk(&b), 5, 0)).unwrap();
    assert_eq!(l.get(&pk(&b)).balance, u64::MAX);
}

#[test]
fn a_transfer_to_yourself_only_consumes_the_nonce() {
    let (mut l, a, _) = funded_accounts();
    l.apply(&Transfer::new(&a, CHAIN, pk(&a), 40, 0)).unwrap();
    assert_eq!(
        l.get(&pk(&a)),
        Account {
            balance: 100,
            nonce: 1
        }
    );
    assert_eq!(
        l.apply(&Transfer::new(&a, CHAIN, pk(&a), 101, 1)),
        Err(TransferError::InsufficientFunds)
    );
}

#[test]
fn invalid_sender_keys_are_just_bad_signatures() {
    let (mut l, _, b) = funded_accounts();
    let t = Transfer {
        chain_id: CHAIN,
        from: [0xff; 32],
        to: pk(&b),
        amount: 1,
        nonce: 0,
        sig: [0; 64],
    };
    assert_eq!(l.apply(&t), Err(TransferError::BadSignature));
}

#[test]
fn signing_bytes_are_framed_and_deterministic() {
    let (_, a, b) = funded_accounts();
    let t = Transfer::new(&a, CHAIN, pk(&b), 10, 0);
    assert_eq!(t.sig, Transfer::new(&a, CHAIN, pk(&b), 10, 0).sig);
    let mut other = t.clone();
    other.amount = 10 + 256;
    assert_ne!(t.signing_bytes(), other.signing_bytes());
    assert_eq!(t.signing_bytes().len(), 8 + 23 + 4 + 32 + 32 + 8 + 8);
}

#[test]
fn replaying_an_applied_transaction_is_reported_as_a_duplicate() {
    let (alice, bob) = (key(1), key(2));
    let (mut set, op) = funded(&alice, 100);
    let tx = spend(&[(&alice, op)], &[(pk(&bob), 100)], 0);
    set.apply(&tx).unwrap();
    assert_eq!(set.apply(&tx), Err(TxError::DuplicateTxid));
}

#[test]
fn funding_accumulates() {
    let mut l = Accounts::new(CHAIN);
    let who = pk(&key(1));
    l.fund(who, 40);
    l.fund(who, 2);
    assert_eq!(l.get(&who).balance, 42);
}

#[test]
#[should_panic(expected = "genesis overflow")]
fn funding_past_the_maximum_panics() {
    let mut l = Accounts::new(CHAIN);
    let who = pk(&key(1));
    l.fund(who, u64::MAX);
    l.fund(who, 1);
}
```

### `tests/chain.rs`

```rust
use merklelab::chain::*;
use merklelab::hash::Hash;
use merklelab::merkle::{Tree, verify};

const BITS: u32 = 8;
const T0: u64 = 1_000;
const NOW: u64 = 1_000_000;

fn txs(tag: &str, n: usize) -> Vec<Vec<u8>> {
    (0..n).map(|i| format!("{tag}-{i}").into_bytes()).collect()
}

fn child(parent: &Block, tag: &str, dt: u64) -> Block {
    mine_block(
        Some(&parent.header),
        txs(tag, 2),
        parent.header.time + dt,
        BITS,
    )
    .unwrap()
}

fn setup() -> (Store, Block) {
    let g = mine_block(None, txs("genesis", 1), T0, BITS).unwrap();
    (Store::new(g.clone(), BITS, 7200).unwrap(), g)
}

#[test]
fn mining_finds_a_nonce_that_meets_the_target() {
    let h = Header {
        height: 1,
        prev: [3; 32],
        merkle_root: [4; 32],
        time: 5,
        bits: BITS,
        nonce: 0,
    };
    let (solved, tries) = mine(h, 1 << 20).unwrap();
    assert!(solved.meets_target());
    assert_eq!(tries, solved.nonce + 1);
    assert_eq!(
        mine(h, 1 << 20).unwrap().0,
        solved,
        "mining is deterministic"
    );
    assert!(merklelab::hash::leading_zero_bits(&solved.hash()) >= BITS);
}

#[test]
fn a_tiny_budget_can_fail() {
    let h = Header {
        height: 1,
        prev: [3; 32],
        merkle_root: [4; 32],
        time: 5,
        bits: 24,
        nonce: 0,
    };
    assert!(mine(h, 50).is_none());
    assert!(mine(h, 0).is_none());
    let easy = Header { bits: 0, ..h };
    assert_eq!(
        mine(easy, 5).unwrap().1,
        1,
        "zero bits: the first nonce works"
    );
}

#[test]
fn every_header_field_changes_the_hash() {
    let h = Header {
        height: 1,
        prev: [3; 32],
        merkle_root: [4; 32],
        time: 5,
        bits: 6,
        nonce: 7,
    };
    let base = h.hash();
    let variants = [
        Header { height: 2, ..h },
        Header { prev: [9; 32], ..h },
        Header {
            merkle_root: [9; 32],
            ..h
        },
        Header { time: 6, ..h },
        Header { bits: 7, ..h },
        Header { nonce: 8, ..h },
    ];
    for v in variants {
        assert_ne!(v.hash(), base, "{v:?}");
    }
}

#[test]
fn work_doubles_with_every_bit() {
    assert_eq!(work(0), 1);
    assert_eq!(work(8), 256);
    assert_eq!(work(64), 1u128 << 64);
}

#[test]
fn a_chain_grows_one_block_at_a_time() {
    let (mut s, g) = setup();
    assert_eq!((s.height(), s.len(), s.best_work()), (0, 1, 256));
    assert!(!s.is_empty());
    let mut tip = g;
    for i in 1..=5 {
        let b = child(&tip, "main", 10);
        assert_eq!(s.add(b.clone(), NOW), Ok(Outcome::Extended), "block {i}");
        tip = b;
    }
    assert_eq!(s.height(), 5);
    assert_eq!(s.best_work(), 6 * 256);
    assert_eq!(s.tip(), tip.header.hash());
    assert_eq!(s.best_chain().len(), 6);
    assert_eq!(s.best_chain()[0], s.tip());
    assert_eq!(s.get(&s.tip()).unwrap(), &tip);
}

#[test]
fn confirmations_count_from_the_tip() {
    let (mut s, g) = setup();
    let b1 = child(&g, "a", 10);
    let b2 = child(&b1, "a", 10);
    let b3 = child(&b2, "a", 10);
    for b in [&b1, &b2, &b3] {
        s.add(b.clone(), NOW).unwrap();
    }
    assert_eq!(s.confirmations(&b3.header.hash()), Some(1));
    assert_eq!(s.confirmations(&b1.header.hash()), Some(3));
    assert_eq!(s.confirmations(&g.header.hash()), Some(4));
    assert_eq!(s.confirmations(&[7; 32]), None);
}

#[test]
fn an_equal_work_branch_does_not_replace_the_first_seen_chain() {
    let (mut s, g) = setup();
    let a1 = child(&g, "a", 10);
    let b1 = child(&g, "b", 11);
    assert_eq!(s.add(a1.clone(), NOW), Ok(Outcome::Extended));
    assert_eq!(s.add(b1.clone(), NOW), Ok(Outcome::SideBranch));
    assert_eq!(s.tip(), a1.header.hash());
    assert_eq!(s.confirmations(&b1.header.hash()), None);
    assert_eq!(s.len(), 3);
}

#[test]
fn a_longer_branch_causes_a_reorg_and_reports_its_depth() {
    let (mut s, g) = setup();
    let a1 = child(&g, "a", 10);
    let a2 = child(&a1, "a", 10);
    let b1 = child(&g, "b", 11);
    let b2 = child(&b1, "b", 11);
    let b3 = child(&b2, "b", 11);
    assert_eq!(s.add(a1.clone(), NOW), Ok(Outcome::Extended));
    assert_eq!(s.add(a2.clone(), NOW), Ok(Outcome::Extended));
    assert_eq!(s.add(b1, NOW), Ok(Outcome::SideBranch));
    assert_eq!(
        s.add(b2.clone(), NOW),
        Ok(Outcome::SideBranch),
        "tie in work: first seen stays"
    );
    assert_eq!(s.add(b3.clone(), NOW), Ok(Outcome::Reorg { depth: 2 }));
    assert_eq!(s.tip(), b3.header.hash());
    assert_eq!(
        s.confirmations(&a1.header.hash()),
        None,
        "the old chain lost its confirmations"
    );
    assert_eq!(s.confirmations(&g.header.hash()), Some(4));
}

#[test]
fn a_reorg_of_depth_one() {
    let (mut s, g) = setup();
    let a1 = child(&g, "a", 10);
    let b1 = child(&g, "b", 11);
    let b2 = child(&b1, "b", 11);
    s.add(a1, NOW).unwrap();
    s.add(b1, NOW).unwrap();
    assert_eq!(s.add(b2, NOW), Ok(Outcome::Reorg { depth: 1 }));
}

#[test]
fn blocks_can_arrive_before_their_parent() {
    let (mut s, g) = setup();
    let b1 = child(&g, "a", 10);
    let b2 = child(&b1, "a", 10);
    assert_eq!(
        s.add(b2.clone(), NOW),
        Err(BlockError::Orphan(b1.header.hash()))
    );
    s.add(b1, NOW).unwrap();
    assert_eq!(s.add(b2, NOW), Ok(Outcome::Extended));
}

#[test]
fn duplicates_are_refused() {
    let (mut s, g) = setup();
    let b = child(&g, "a", 10);
    s.add(b.clone(), NOW).unwrap();
    assert_eq!(s.add(b, NOW), Err(BlockError::Duplicate));
    assert_eq!(s.add(g, NOW), Err(BlockError::Duplicate));
}

fn remine(mut h: Header) -> Header {
    h.nonce = 0;
    mine(h, 1 << 22).unwrap().0
}

#[test]
fn a_header_must_meet_every_consensus_rule() {
    let (mut s, g) = setup();
    let good = child(&g, "a", 10);

    // wrong height (properly mined, so only the height is wrong)
    let h = remine(Header {
        height: 5,
        ..good.header
    });
    assert_eq!(
        s.add(
            Block {
                header: h,
                txs: good.txs.clone()
            },
            NOW
        ),
        Err(BlockError::BadHeight)
    );

    // wrong difficulty, even though the work is real
    let h = mine(
        Header {
            bits: 4,
            nonce: 0,
            ..good.header
        },
        1 << 20,
    )
    .unwrap()
    .0;
    assert_eq!(
        s.add(
            Block {
                header: h,
                txs: good.txs.clone()
            },
            NOW
        ),
        Err(BlockError::WrongDifficulty)
    );

    // not enough work: find a nonce that does NOT meet the target
    let mut h = good.header;
    for n in 0u64.. {
        h.nonce = n;
        if !h.meets_target() {
            break;
        }
    }
    assert_eq!(
        s.add(
            Block {
                header: h,
                txs: good.txs.clone()
            },
            NOW
        ),
        Err(BlockError::InsufficientWork)
    );

    // transactions that do not match the committed root
    let mut tampered = good.clone();
    tampered.txs[0].push(b'!');
    assert_eq!(s.add(tampered, NOW), Err(BlockError::BadMerkleRoot));
    let mut fewer = good.clone();
    fewer.txs.pop();
    assert_eq!(s.add(fewer, NOW), Err(BlockError::BadMerkleRoot));

    // time not later than the parent's
    let h = remine(Header {
        time: T0,
        ..good.header
    });
    assert_eq!(
        s.add(
            Block {
                header: h,
                txs: good.txs.clone()
            },
            NOW
        ),
        Err(BlockError::TimeNotIncreasing)
    );
    let h = remine(Header {
        time: T0 - 1,
        ..good.header
    });
    assert_eq!(
        s.add(
            Block {
                header: h,
                txs: good.txs.clone()
            },
            NOW
        ),
        Err(BlockError::TimeNotIncreasing)
    );

    assert_eq!(s.len(), 1, "nothing invalid was stored");
    assert_eq!(s.add(good, NOW), Ok(Outcome::Extended));
}

#[test]
fn the_future_limit_is_inclusive() {
    let (mut s, g) = setup();
    let now = 5_000;
    let at_limit = mine_block(Some(&g.header), txs("x", 1), now + 7200, BITS).unwrap();
    let over = mine_block(Some(&g.header), txs("y", 1), now + 7201, BITS).unwrap();
    assert_eq!(s.add(over, now), Err(BlockError::TooFarInFuture));
    assert_eq!(s.add(at_limit, now), Ok(Outcome::Extended));
}

#[test]
fn the_future_limit_does_not_overflow() {
    let (mut s, g) = setup();
    let b = mine_block(Some(&g.header), txs("z", 1), u64::MAX, BITS).unwrap();
    assert_eq!(s.add(b.clone(), u64::MAX), Ok(Outcome::Extended));
    let (mut s2, g2) = setup();
    let b2 = mine_block(Some(&g2.header), txs("z", 1), u64::MAX, BITS).unwrap();
    assert_eq!(s2.add(b2, 0), Err(BlockError::TooFarInFuture));
    drop((s, b));
}

#[test]
fn genesis_must_be_well_formed() {
    let g = mine_block(None, txs("g", 1), T0, BITS).unwrap();
    assert!(Store::new(g.clone(), BITS, 10).is_ok());
    assert_eq!(
        Store::new(g.clone(), BITS + 1, 10).err(),
        Some(BlockError::BadGenesis)
    );
    let mut bad = g.clone();
    bad.txs.push(b"extra".to_vec());
    assert_eq!(
        Store::new(bad, BITS, 10).err(),
        Some(BlockError::BadGenesis)
    );
    let h = remine(Header {
        height: 1,
        ..g.header
    });
    assert_eq!(
        Store::new(
            Block {
                header: h,
                txs: g.txs.clone()
            },
            BITS,
            10
        )
        .err(),
        Some(BlockError::BadGenesis)
    );
    let h = remine(Header {
        prev: [1; 32],
        ..g.header
    });
    assert_eq!(
        Store::new(
            Block {
                header: h,
                txs: g.txs.clone()
            },
            BITS,
            10
        )
        .err(),
        Some(BlockError::BadGenesis)
    );
    let mut unsolved = g.header;
    for n in 0u64.. {
        unsolved.nonce = n;
        if !unsolved.meets_target() {
            break;
        }
    }
    assert_eq!(
        Store::new(
            Block {
                header: unsolved,
                txs: g.txs
            },
            BITS,
            10
        )
        .err(),
        Some(BlockError::BadGenesis)
    );
}

#[test]
#[should_panic(expected = "difficulty above 64 bits")]
fn absurd_difficulty_is_refused_up_front() {
    let g = mine_block(None, txs("g", 1), T0, BITS).unwrap();
    let _ = Store::new(g, 65, 10);
}

#[test]
fn the_header_commits_to_transactions_through_the_merkle_root() {
    let (_, g) = setup();
    let b = mine_block(Some(&g.header), txs("tx", 5), T0 + 1, BITS).unwrap();
    let refs: Vec<&[u8]> = b.txs.iter().map(|t| t.as_slice()).collect();
    let tree = Tree::new(&refs);
    assert_eq!(tree.root(), b.header.merkle_root);
    let proof = tree.prove(3).unwrap();
    assert!(
        verify(&b.header.merkle_root, &b.txs[3], &proof),
        "a light client needs only header + proof"
    );
    assert!(!verify(&b.header.merkle_root, b"tx-forged", &proof));
}

#[test]
fn changing_an_old_block_breaks_every_later_link() {
    let (mut s, g) = setup();
    let b1 = child(&g, "a", 10);
    let b2 = child(&b1, "a", 10);
    s.add(b1.clone(), NOW).unwrap();
    s.add(b2.clone(), NOW).unwrap();
    // An attacker rewrites b1's transactions and re-mines it: it has a new hash,
    // so b2 (which names the old hash) no longer attaches to it.
    let mut forged = Block {
        header: b1.header,
        txs: txs("evil", 2),
    };
    let refs: Vec<&[u8]> = forged.txs.iter().map(|t| t.as_slice()).collect();
    forged.header.merkle_root = Tree::new(&refs).root();
    forged.header = remine(forged.header);
    assert_ne!(forged.header.hash(), b1.header.hash());
    assert_ne!(forged.header.hash(), b2.header.prev);
    let mut fresh = Store::new(g, BITS, 7200).unwrap();
    fresh.add(forged, NOW).unwrap();
    assert_eq!(
        fresh.add(b2, NOW),
        Err(BlockError::Orphan(b1.header.hash()))
    );
}

#[test]
fn a_longer_chain_of_cheap_blocks_does_not_beat_fewer_expensive_ones() {
    // Fork choice counts work, not blocks. Two bits more per block is 4x the work.
    // (Different difficulty is not valid in one network; this checks the arithmetic.)
    assert!(work(10) > 3 * work(8));
    let _: Hash = [0; 32];
}

#[test]
fn a_hash_with_exactly_the_required_zero_bits_is_enough() {
    let bits = 4;
    let mut exact = 0;
    for k in 0..40u64 {
        let h = Header {
            height: k,
            prev: [k as u8; 32],
            merkle_root: [1; 32],
            time: 1,
            bits,
            nonce: 0,
        };
        let (s, _) = mine(h, 1 << 20).unwrap();
        assert!(s.meets_target());
        if merklelab::hash::leading_zero_bits(&s.hash()) == bits {
            exact += 1;
        }
    }
    assert!(exact > 0, "some solutions have exactly {bits} zero bits");
}
```

### `tests/quorum.rs`

```rust
use merklelab::quorum::*;

#[test]
fn tolerated_faults_follow_n_equals_3f_plus_1() {
    let table = [
        (0, 0),
        (1, 0),
        (2, 0),
        (3, 0),
        (4, 1),
        (5, 1),
        (6, 1),
        (7, 2),
        (10, 3),
        (100, 33),
    ];
    for (n, f) in table {
        assert_eq!(max_faulty(n), f, "n = {n}");
    }
}

#[test]
fn quorum_sizes() {
    let bft = [
        (1, 1),
        (2, 2),
        (3, 3),
        (4, 3),
        (5, 4),
        (6, 5),
        (7, 5),
        (10, 7),
        (100, 67),
    ];
    for (n, q) in bft {
        assert_eq!(bft_quorum(n), q, "bft n = {n}");
    }
    let maj = [(1, 1), (2, 2), (3, 2), (4, 3), (5, 3), (10, 6), (101, 51)];
    for (n, q) in maj {
        assert_eq!(majority(n), q, "majority n = {n}");
    }
}

#[test]
fn min_overlap_matches_brute_force_over_all_subsets() {
    for n in 1..=10usize {
        for q in 1..=n {
            let subsets: Vec<u32> = (0u32..(1 << n))
                .filter(|m| m.count_ones() as usize == q)
                .collect();
            let mut least = usize::MAX;
            for a in &subsets {
                for b in &subsets {
                    least = least.min((a & b).count_ones() as usize);
                }
            }
            assert_eq!(least, min_overlap(n, q), "n={n} q={q}");
        }
    }
}

#[test]
fn two_majorities_always_share_a_node() {
    for n in 1..=200 {
        assert!(min_overlap(n, majority(n)) >= 1, "n = {n}");
    }
}

#[test]
fn bft_quorums_are_safe_and_live_up_to_the_threshold() {
    for n in 1..=60 {
        let f = max_faulty(n);
        let q = bft_quorum(n);
        assert!(quorums_share_an_honest_node(n, q, f), "n={n}");
        assert!(honest_can_form_quorum(n, q, f), "n={n}");
        assert!(!conflicting_decisions_possible(n, q, f), "n={n}");
    }
}

#[test]
fn one_fault_too_many_breaks_safety_or_liveness() {
    // n = 4, tolerates 1. With 2 faulty, two conflicting decisions are possible.
    let (n, q) = (4, bft_quorum(4));
    assert!(conflicting_decisions_possible(n, q, 2));
    assert!(!quorums_share_an_honest_node(n, q, 2));
    // n = 3 tolerates 0. With 1 faulty (a node that just goes silent) the honest
    // nodes can no longer form a quorum, so progress stops: liveness is lost first.
    assert!(!honest_can_form_quorum(3, bft_quorum(3), 1));
    assert!(!conflicting_decisions_possible(3, bft_quorum(3), 1));
    // exactly at the threshold: n = 7, f = 2 is fine, f = 3 is not
    assert!(!conflicting_decisions_possible(7, bft_quorum(7), 2));
    assert!(conflicting_decisions_possible(7, bft_quorum(7), 3));
}

#[test]
fn the_closed_form_agrees_with_the_enumeration() {
    for n in 1..=24usize {
        for f in 0..=n {
            for q in 1..=n {
                let closed = 2 * q.saturating_sub(f) <= n - f;
                assert_eq!(
                    conflicting_decisions_possible(n, q, f),
                    closed,
                    "n={n} f={f} q={q}"
                );
            }
        }
    }
}

#[test]
fn a_simple_majority_is_not_enough_against_equivocation() {
    // Crash-fault majority quorums (n = 5, q = 3) are not safe if one node lies.
    assert!(!conflicting_decisions_possible(5, majority(5), 0));
    assert!(conflicting_decisions_possible(5, majority(5), 1));
}

#[test]
fn votes_count_every_equivocator_for_both_sides() {
    assert_eq!(votes_with_equivocators(2, 1), 3);
    assert_eq!(votes_with_equivocators(0, 0), 0);
}

#[test]
fn catch_up_probability_follows_the_closed_form() {
    assert_eq!(catch_up_probability(0.1, 0), 1.0);
    let p5 = catch_up_probability(0.1, 5);
    assert!((p5 - (0.1f64 / 0.9).powi(5)).abs() < 1e-15);
    assert!(p5 > 1.6e-5 && p5 < 1.8e-5, "{p5}");
    assert!(catch_up_probability(0.3, 6) < catch_up_probability(0.3, 3));
    assert_eq!(catch_up_probability(0.5, 100), 1.0);
    assert_eq!(catch_up_probability(0.9, 100), 1.0);
    assert_eq!(catch_up_probability(0.0, 3), 0.0);
    assert_eq!(catch_up_probability(1.0, 3), 1.0);
}

#[test]
#[should_panic(expected = "q must be a fraction")]
fn catch_up_probability_rejects_a_fraction_above_one() {
    let _ = catch_up_probability(1.5, 1);
}

#[test]
#[should_panic(expected = "q must be a fraction")]
fn catch_up_probability_rejects_a_negative_fraction() {
    let _ = catch_up_probability(-0.1, 1);
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
//! Prints the measurements quoted in Lesson 49.
//! Run with `cargo run --release --example bench`.

use std::hint::black_box;
use std::time::Instant;

use merklelab::chain::{Header, mine};
use merklelab::merkle::{Tree, verify};
use merklelab::quorum::catch_up_probability;

fn leaves(n: usize) -> Vec<Vec<u8>> {
    (0..n)
        .map(|i| format!("transaction-{i:08}-padding-padding-padding-padding").into_bytes())
        .collect()
}

fn median(mut v: Vec<f64>) -> f64 {
    v.sort_by(|a, b| a.partial_cmp(b).expect("no NaN"));
    v[v.len() / 2]
}

fn main() {
    println!("== Merkle tree (64-byte leaves, median of 5 runs)");
    for n in [1_000usize, 10_000, 100_000] {
        let data = leaves(n);
        let refs: Vec<&[u8]> = data.iter().map(|d| d.as_slice()).collect();
        let build = median(
            (0..5)
                .map(|_| {
                    let t = Instant::now();
                    black_box(Tree::new(black_box(&refs)).root());
                    t.elapsed().as_secs_f64() * 1e3
                })
                .collect(),
        );
        let tree = Tree::new(&refs);
        let root = tree.root();
        let prove = median(
            (0..5)
                .map(|_| {
                    let t = Instant::now();
                    black_box(tree.prove(black_box(n / 3)));
                    t.elapsed().as_secs_f64() * 1e6
                })
                .collect(),
        );
        let proof = tree.prove(n / 3).expect("in range");
        let iters = 20_000;
        let t = Instant::now();
        for _ in 0..iters {
            assert!(verify(
                black_box(&root),
                black_box(&data[n / 3]),
                black_box(&proof)
            ));
        }
        let ver = t.elapsed().as_secs_f64() * 1e9 / iters as f64;
        println!(
            "n = {n:>7}: build root {build:>8.2} ms | prove one leaf {prove:>9.1} us | proof = {} hashes ({} bytes) | verify {ver:>7.0} ns",
            proof.path.len(),
            proof.path.len() * 32
        );
    }
    println!(
        "== proof of work (header hash with N leading zero bits), 20 different headers per size"
    );
    for bits in [8u32, 12, 16, 20] {
        let runs = if bits == 20 { 8 } else { 20 };
        let mut tries = Vec::new();
        let t = Instant::now();
        for k in 0..runs {
            let h = Header {
                height: k,
                prev: [k as u8; 32],
                merkle_root: [7; 32],
                time: 1,
                bits,
                nonce: 0,
            };
            let (_, n) = mine(h, 1 << 26).expect("found");
            tries.push(n as f64);
        }
        let secs = t.elapsed().as_secs_f64();
        let total: f64 = tries.iter().sum();
        println!(
            "bits = {bits:>2}: mean tries {:>10.0} (2^bits = {:>8}) | {:.2} Mhash/s",
            total / runs as f64,
            1u64 << bits,
            total / secs / 1e6
        );
    }
    println!("== attacker catch-up probability (simple model)");
    for q in [0.1f64, 0.3] {
        let row: Vec<String> = [1u32, 3, 6, 10]
            .iter()
            .map(|&z| format!("z={z}: {:.6}", catch_up_probability(q, z)))
            .collect();
        println!("q = {q}: {}", row.join(" | "));
    }
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **74 tests** (hash 3, Merkle 14, ledger 25, chain 20, quorum 12) and 8 consecutive full runs without a failure (proof-of-work tests use `bits = 8`, about 256 hashes per block, so they are fast and deterministic).

**Mutation check ✓.** I made 100 one-line changes (a "mutant") across the five source modules, each in its own copy of the crate, and ran the whole test suite against each. **94 were detected (93 by a failing or aborting test, 1 as a hang), 6 are equivalent (M10, M14, M16, M17, M34, M98), none survive.** Ten mutants survived the *first* run; I looked at each, and four exposed real gaps that I closed with a new test before re-running them: M9 (an index equal to the size is only distinguishable from a valid one when the path is empty, so I added the one-leaf case), M32 (the error for replaying an applied transaction was not pinned), M55 (funding was only ever tested once per account, so "overwrite" looked the same as "add") and M57 (no test used a header with exactly the required number of zero bits). The other six first-run survivors (M10, M14, M16, M17, M34 and M98) are equivalent mutants, for the reasons in the table: they change code that no input can reach or whose effect another check already covers. One mutant (M4) made the test process abort (an infinite recursion), which counts as detected. My harness first reported `#[should_panic]` failures as survivors because their line reads `- should panic ... FAILED`; I fixed the classifier before running this table (the same classifier flaw I found in the previous lesson), and I checked the logs of every survivor for the word FAILED. Mutants that were not valid Rust at the point of change were rewritten, not counted. The tests ran in the dev profile.

| # | module | change | result |
|---|---|---|---|
| M1 | `merkle` | leaf prefix `0x01` (same as node prefix) | caught by `leaves_and_nodes_are_hashed_with_different_prefixes` |
| M2 | `merkle` | node prefix `0x00` (same as leaf prefix) | caught by `leaves_and_nodes_are_hashed_with_different_prefixes` |
| M3 | `merkle` | leaf hash without a prefix | caught by `roots_match_an_independent_python_implementation` |
| M4 | `merkle` | split at the largest power of two `<= n` instead of `< n` | caught (the test process aborted) |
| M5 | `merkle` | empty-tree root is the hash of a zero byte | caught by `roots_match_an_independent_python_implementation` |
| M6 | `merkle` | `prove`: `m <= k` instead of `m < k` | caught by `the_header_commits_to_transactions_through_the_merkle_root` |
| M7 | `merkle` | `prove`: path not reversed to bottom-first | caught by `the_header_commits_to_transactions_through_the_merkle_root` |
| M8 | `merkle` | `prove`: index not rebased in the right subtree | caught by `every_leaf_of_every_tree_up_to_forty_leaves_has_a_valid_proof` |
| M9 | `merkle` | `verify`: index equal to size allowed | caught by `an_index_equal_to_the_size_is_refused_even_when_the_path_is_empty` |
| M10 | `merkle` | `verify`: early `s == 0` return removed | **equivalent**: an extra path element changes the hash, so the final comparison fails anyway |
| M11 | `merkle` | `verify`: `f == s` case removed | caught by `every_leaf_of_every_tree_up_to_forty_leaves_has_a_valid_proof` |
| M12 | `merkle` | `verify`: final `s == 0` check removed | caught by `the_root_alone_does_not_commit_to_the_tree_size` |
| M13 | `merkle` | `verify`: sibling on the wrong side | caught by `the_header_commits_to_transactions_through_the_merkle_root` |
| M14 | `merkle` | `verify`: explicit `size == 0` check removed | **equivalent**: `index >= size` already rejects every index when the size is zero |
| M15 | `merkle` | `duplicating_root` pairs with the first node, not the last | caught by `duplicating_the_last_node_makes_different_lists_share_a_root` |
| M16 | `merkle` | `verify`: `f != 0` guard removed in the inner loop | **equivalent**: the loop is reached only when `f == s` and `s != 0`, so `f` never becomes zero |
| M17 | `merkle` | `verify`: inner loop entered even when `f` is odd | **equivalent**: the inner loop tests the same condition, so entering it with an odd `f` does nothing |
| M18 | `merkle` | `prove`: `index > len` instead of `>=` | caught by `out_of_range_proofs_do_not_exist` |
| M19 | `hash` | `leading_zero_bits` counts one too many | caught by `leading_zero_bits_counts_bits_not_bytes` |
| M20 | `hash` | whole zero byte counted as 7 bits | caught by `leading_zero_bits_counts_bits_not_bytes` |
| M21 | `hash` | `hex` does not pad to two digits | caught by `hex_pads_every_byte_to_two_digits` |
| M22 | `ledger` | input sum uses `wrapping_add` | caught by `sums_are_checked_not_wrapped` |
| M23 | `ledger` | inflation check `>=` instead of `>` | caught by `outputs_may_not_exceed_inputs` |
| M24 | `ledger` | fee reported as the input total | caught by `a_signed_spend_moves_value_and_reports_the_fee` |
| M25 | `ledger` | duplicate input inside one transaction not detected | caught by `the_same_output_twice_inside_one_transaction_is_refused` |
| M26 | `ledger` | spent inputs are not removed | caught by `a_signed_spend_moves_value_and_reports_the_fee` |
| M27 | `ledger` | input signature check skipped | caught by `only_the_owner_can_spend` |
| M28 | `ledger` | signature count check `<` instead of `!=` | caught by `structural_errors` |
| M29 | `ledger` | coinbase: zero-amount output allowed | caught by `coinbase_rules` |
| M30 | `ledger` | spend: zero-amount output allowed | caught by `structural_errors` |
| M31 | `ledger` | coinbase: duplicate id allowed | caught by `coinbase_rules` |
| M32 | `ledger` | spend: duplicate-id check removed | caught by `replaying_an_applied_transaction_is_reported_as_a_duplicate` |
| M33 | `ledger` | `extra` not part of the transaction body | caught by `coinbase_rules` |
| M34 | `ledger` | output count not part of the transaction body | **equivalent**: with fixed-width entries and a count for the inputs, the length of the rest already fixes the number of outputs (the encoding stays injective) |
| M35 | `ledger` | sighash uses the same domain label as the txid | caught by `the_txid_ignores_signatures_but_the_sighash_does_not_ignore_the_body` |
| M36 | `ledger` | `balance` counts other owners | caught by `a_signed_spend_moves_value_and_reports_the_fee` |
| M37 | `ledger` | output index off by one | caught by `a_signed_spend_moves_value_and_reports_the_fee` |
| M38 | `ledger` | `verify` instead of `verify_strict` | caught by `outputs_locked_to_weak_or_invalid_keys_cannot_be_spent` |
| M39 | `ledger` | coinbase may have inputs or signatures | caught by `coinbase_rules` |
| M40 | `ledger` | transfer: chain id not checked | caught by `a_transfer_for_another_network_is_refused` |
| M41 | `ledger` | transfer: signature not checked | caught by `a_transfer_for_another_network_is_refused` |
| M42 | `ledger` | nonce check `<` instead of `!=` | caught by `nonces_must_be_used_in_order_without_gaps` |
| M43 | `ledger` | zero-amount transfer allowed | caught by `balance_rules_and_atomicity` |
| M44 | `ledger` | self-transfer does not bump the nonce | caught by `a_transfer_to_yourself_only_consumes_the_nonce` |
| M45 | `ledger` | normal transfer does not bump the nonce | caught by `a_transfer_moves_balance_and_bumps_the_nonce` |
| M46 | `ledger` | recipient overflow silently becomes 0 | caught by `a_recipient_balance_cannot_overflow` |
| M47 | `ledger` | insufficient funds silently becomes 0 | caught by `balance_rules_and_atomicity` |
| M48 | `ledger` | self-transfer special case removed | caught by `a_transfer_to_yourself_only_consumes_the_nonce` |
| M49 | `ledger` | chain id not in the signed bytes | caught by `a_transfer_for_another_network_is_refused` |
| M50 | `ledger` | sender not in the signed bytes | caught by `signing_bytes_are_framed_and_deterministic` |
| M51 | `ledger` | recipient not in the signed bytes | caught by `every_signed_field_is_covered` |
| M52 | `ledger` | amount not in the signed bytes | caught by `every_signed_field_is_covered` |
| M53 | `ledger` | nonce not in the signed bytes | caught by `every_signed_field_is_covered` |
| M54 | `ledger` | domain label length prefix removed | caught by `signing_bytes_are_framed_and_deterministic` |
| M55 | `ledger` | `fund` overwrites instead of adding | caught by `funding_past_the_maximum_panics` |
| M56 | `chain` | `mine` reports `n` tries instead of `n + 1` | caught by `a_tiny_budget_can_fail` |
| M57 | `chain` | target check `>` instead of `>=` | caught by `a_hash_with_exactly_the_required_zero_bits_is_enough` |
| M58 | `chain` | height not in the header hash | caught by `every_header_field_changes_the_hash` |
| M59 | `chain` | `prev` not in the header hash | caught by `every_header_field_changes_the_hash` |
| M60 | `chain` | Merkle root not in the header hash | caught by `changing_an_old_block_breaks_every_later_link` |
| M61 | `chain` | time not in the header hash | caught by `every_header_field_changes_the_hash` |
| M62 | `chain` | `bits` not in the header hash | caught by `every_header_field_changes_the_hash` |
| M63 | `chain` | nonce not in the header hash | detected as a **hang**: mining never succeeds without the nonce in the hash (killed after 240 s) |
| M64 | `chain` | total work ignores the parent | caught by `a_chain_grows_one_block_at_a_time` |
| M65 | `chain` | time rule `<` instead of `<=` | caught by `a_header_must_meet_every_consensus_rule` |
| M66 | `chain` | future limit `>=` instead of `>` | caught by `the_future_limit_does_not_overflow` |
| M67 | `chain` | future limit uses `+` instead of `saturating_add` | caught by `the_future_limit_does_not_overflow` |
| M68 | `chain` | fork choice `>=` instead of `>` | caught by `a_reorg_of_depth_one` |
| M69 | `chain` | reorg depth counts 2 per block | caught by `a_longer_branch_causes_a_reorg_and_reports_its_depth` |
| M70 | `chain` | `work(bits)` is `2^(bits+1)` | caught by `a_chain_grows_one_block_at_a_time` |
| M71 | `chain` | height rule `<` instead of `!=` | caught by `a_header_must_meet_every_consensus_rule` |
| M72 | `chain` | difficulty rule `>` instead of `!=` | caught by `a_header_must_meet_every_consensus_rule` |
| M73 | `chain` | proof-of-work check skipped | caught by `a_header_must_meet_every_consensus_rule` |
| M74 | `chain` | Merkle root check skipped | caught by `a_header_must_meet_every_consensus_rule` |
| M75 | `chain` | duplicate block check skipped | caught by `duplicates_are_refused` |
| M76 | `chain` | genesis: difficulty not checked | caught by `genesis_must_be_well_formed` |
| M77 | `chain` | genesis: work not checked | caught by `genesis_must_be_well_formed` |
| M78 | `chain` | genesis: Merkle root not checked | caught by `genesis_must_be_well_formed` |
| M79 | `chain` | genesis: height not checked | caught by `genesis_must_be_well_formed` |
| M80 | `chain` | genesis: parent not checked | caught by `genesis_must_be_well_formed` |
| M81 | `chain` | difficulty cap 65 instead of 64 | caught by `absurd_difficulty_is_refused_up_front` |
| M82 | `chain` | confirmations off by one | caught by `a_longer_branch_causes_a_reorg_and_reports_its_depth` |
| M83 | `chain` | `mine_block` height not parent + 1 | caught by `a_chain_grows_one_block_at_a_time` |
| M84 | `chain` | every new block treated as extending the tip | caught by `a_reorg_of_depth_one` |
| M85 | `quorum` | `max_faulty` divides by 2 | caught by `bft_quorums_are_safe_and_live_up_to_the_threshold` |
| M86 | `quorum` | `max_faulty` without the `- 1` | caught by `bft_quorums_are_safe_and_live_up_to_the_threshold` |
| M87 | `quorum` | majority is `n / 2` | caught by `a_simple_majority_is_not_enough_against_equivocation` |
| M88 | `quorum` | BFT quorum without the `+ 1` | caught by `bft_quorums_are_safe_and_live_up_to_the_threshold` |
| M89 | `quorum` | BFT quorum `+ 2` | caught by `bft_quorums_are_safe_and_live_up_to_the_threshold` |
| M90 | `quorum` | overlap uses `q` instead of `2q` | caught by `bft_quorums_are_safe_and_live_up_to_the_threshold` |
| M91 | `quorum` | safety condition `>=` instead of `>` | caught by `one_fault_too_many_breaks_safety_or_liveness` |
| M92 | `quorum` | liveness condition `>` instead of `>=` | caught by `bft_quorums_are_safe_and_live_up_to_the_threshold` |
| M93 | `quorum` | equivocators not counted | caught by `a_simple_majority_is_not_enough_against_equivocation` |
| M94 | `quorum` | enumeration misses all-honest-for-A | caught by `the_closed_form_agrees_with_the_enumeration` |
| M95 | `quorum` | enumeration misses all-remaining-for-B | caught by `a_simple_majority_is_not_enough_against_equivocation` |
| M96 | `quorum` | first side needs `> q` votes | caught by `a_simple_majority_is_not_enough_against_equivocation` |
| M97 | `quorum` | second side needs `> q` votes | caught by `a_simple_majority_is_not_enough_against_equivocation` |
| M98 | `quorum` | catch-up: `q > 0.5` instead of `>=` | **equivalent**: at `q = 0.5` the formula gives `(0.5/0.5)^z = 1`, the same value |
| M99 | `quorum` | catch-up ratio `q` instead of `q/(1-q)` | caught by `catch_up_probability_follows_the_closed_form` |
| M100 | `quorum` | `q = 1.0` rejected | caught by `catch_up_probability_follows_the_closed_form` |

What the table does not show: it tests *these* edits, not every possible bug; there is no mutant for the dependencies, for networks, or for the ledger-state rollback that a real node needs; the equivalent mutants could hide a real gap if my reasoning about them is wrong; and passing every mutant says nothing about security.

**Design notes.**

- Rules are small functions and enums (`merkle::verify`, `UtxoSet::apply`, `Accounts::apply`, `Store::add`, `quorum::*`) so each can be tested alone, and each test violates exactly one rule. The tests for block rules **mine the offending block properly** so that only the rule under test fails.
- The Merkle tree, the ledgers, the chain and the quorum arithmetic do not depend on each other except through `hash.rs`; the block store treats transactions as opaque bytes and commits to them with `merkle::root_of`. A real node connects them: validate each transaction against the ledger, then accept the block.
- Time is passed in (`Store::add(block, now)`) and nonces are searched deterministically, so tests never read the clock or a random source.
- Not handled: network protocols, transaction validation inside blocks, difficulty retargeting, mempool, persistence, ledger rollback on reorg, block size limits, consistency proofs, sparse Merkle trees, validator-set changes, signature aggregation, and everything in §9.

</details>

---

## 14. Key takeaways

- A Merkle root commits to a list and gives logarithmic membership proofs (✓ 17 hashes for 100,000 leaves, verified in about 2.5 µs); it must separate leaves from nodes (✓ forged without prefixes), never duplicate an odd node (✓ collision), and does not by itself commit to the size (✓ a proof for 3 leaves verified as 4).
- A transaction model is a state, a validity rule and a replay defense. UTXO uses the spent output; accounts use a nonce and a chain id; both need checked arithmetic, strict signature verification and all-or-nothing application (✓ every rule has a test that breaks only that rule).
- A hash-linked chain makes edits detectable; proof of work makes rewriting expensive (✓ `2^bits` tries on average: 190, 3,818, 55,774 and 883,796 against 256, 4,096, 65,536 and 1,048,576); nodes follow the branch with the most cumulative work, and a reorg undoes blocks.
- Confirmations lower risk without removing it; in the simple model the attacker's odds fall exponentially with depth when `q < 1/2` and are certain when `q ≥ 1/2`.
- Byzantine agreement needs `n ≥ 3f + 1` and quorums of `⌊2n/3⌋ + 1`; the overlap argument was checked by brute force, and one fault too many breaks safety or liveness (✓ `n = 4, f = 2`; `n = 10, f = 4`).
- Every guarantee comes with trust assumptions; write them down, check that they are independent, and count fault domains rather than machines.
- Consensus-critical code must be deterministic: fixed-width integers, explicit byte order, checked arithmetic, no `HashMap` order or floats in rules.
- Mutation testing again found real gaps (an empty-path index case, an unpinned error precedence, funding that did not accumulate, a boundary on the target) and four equivalent mutants in the verifier that are defensive code (✓ 100 mutants, none surviving that I could not explain).

**Connections.** [Lesson 48](48_crypto_primitives.md) supplies every primitive used here: domain separation (the Merkle prefixes and the `txid`/`sighash` labels), `verify_strict`, nonces and replay windows (the account nonce and chain id), and the discipline of pinning policy values in tests. [Lesson 46](46_databases_config_auth_jobs.md) covers transactions and atomicity in databases, the same "validate, then commit" idea as §7. [Lesson 45](45_http_apis.md) covers idempotency, the practical cousin of replay protection. [Lesson 44](44_cli_serialization.md) is where canonical encodings were introduced. [Lessons 39 and 40](40_property_testing_fuzzing.md) explain testing styles used here (the brute-force and closed-form comparison is a property test in spirit). [Lesson 29](29_locks_deadlocks.md) covers the races that appear when several threads apply transactions. Lesson 50 moves from agreement to the network: peer-to-peer messaging and determinism.

---

## 15. References

- Ralph C. Merkle, "A Digital Signature Based on a Conventional Encryption Function" (1987) and the earlier patent/paper on hash trees (from memory).
- RFC 6962 and RFC 9162, Certificate Transparency (Merkle tree hash, audit paths): <https://www.rfc-editor.org/>
- Satoshi Nakamoto, "Bitcoin: A Peer-to-Peer Electronic Cash System" (2008), including the section on calculations (the catch-up probability): <https://bitcoin.org/bitcoin.pdf>
- Adam Back, "Hashcash — A Denial of Service Counter-Measure" (2002).
- Michael J. Fischer, Nancy A. Lynch, Michael S. Paterson, "Impossibility of Distributed Consensus with One Faulty Process" (1985).
- Leslie Lamport, Robert Shostak, Marshall Pease, "The Byzantine Generals Problem" (1982).
- Miguel Castro and Barbara Liskov, "Practical Byzantine Fault Tolerance" (1999).
- Diego Ongaro and John Ousterhout, "In Search of an Understandable Consensus Algorithm" (Raft).
- Vitalik Buterin et al., Ethereum yellow paper / documentation on accounts and nonces (from memory, for the account model).
- RustCrypto `sha2`: <https://github.com/RustCrypto/hashes>; `ed25519-dalek`: <https://github.com/dalek-cryptography/curve25519-dalek>
- The Rust Reference on integer overflow and the `checked_*`/`saturating_*`/`wrapping_*` methods: <https://doc.rust-lang.org/std/>

The URLs and titles above are the standard locations and papers; I did not open them during this session (the sandbox cannot reach the documentation or standards sites), so if a link has moved, search for the title. **Unchecked or from memory:** the descriptions of RFC 6962 and RFC 9162 (algorithms were re-derived and the roots checked against an independent Python implementation, but not against the RFC text or its published test vectors); Merkle's original constructions; the history and details of CVE-2012-2459 and how Bitcoin Core handles duplicates; Bitcoin's duplication rule, its signature malleability history and SPV; the account/UTXO characterizations of Bitcoin and Ethereum; the FLP statement; Raft and Paxos fault tolerance; PBFT's `3f + 1`; probabilistic versus deterministic finality; the catch-up probability formula (the formula is implemented and tested against itself, and I did not re-derive it); real-network hash rates and difficulty; Certificate Transparency's signed tree heads; that Git is a related hash-linked structure; the claim that parallel validation favors UTXO; and everything in §6 beyond the arithmetic. **Environment caveats:** timings are single-machine micro-measurements from three runs of one example on a shared virtual machine with two cores; the mining statistics are means over 20 headers (8 for 20 bits), not estimates with confidence intervals. Measurements and outcomes marked ✓ were observed with `rustc 1.97.0`, `sha2 0.11.0` and `ed25519-dalek 3.0.0`. The code blocks in exercises 3, 4 and 5 and in the debugging challenge's two versions were compiled and run in a scratch crate (✓ 9 tests, including a stack overflow that my first version of exercise 3 triggered and that I fixed); the numbers in exercises 7 and 8 were evaluated by tests; the mutation results are those of the table above (a test process that aborted, failed an assertion or did not finish within four minutes counted as detecting the mutant). **No part of this lesson makes any system secure.**
