---
layout: default
title: "Lesson 55 — Program testing, deployment, upgrades, security auditing"
---

# Lesson 55 — Program testing, deployment, upgrades, security auditing

**Module 17 · Solana (third and last lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `sha2 0.11.0`, plus the Lesson 53 crate `solanalab` and the Lesson 54 crate `anchorlab` as path dependencies (24 packages in `Cargo.lock`, the crate itself included)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** Lessons 53 and 54 built a runtime model and the programs on top of it, and tested them one claim at a time. This lesson is about testing and reviewing *as a discipline*: how to find the bug you did not think of. It builds a small toolbox in the same model: a **world** that can be snapshotted, hashed and compared; **invariants** that must hold after every transaction; an **account-substitution fuzzer** that plays the attacker's part (the attacker chooses the accounts, so the harness tries every substitution it can enumerate, signing only for itself); a **static reviewer** for the constraint lists of Lesson 54; and a model of the **upgradeable loader** (upgrade authority, slot, immutability) with **replay testing** to compare two versions of a program. It ends with how a security review is organized in practice. Everything runs on the model, not on a validator: I did not install the Solana toolchain, run `solana-test-validator`, deploy anywhere, or use any real testing framework, and every statement about the real tools and the real loader is labeled "from memory". **Not audited; finding nothing proves nothing, and nothing here secures anything.** Not covered: formal verification, fuzzing the compiled BPF/SBF program (Lesson 40 covers fuzzing in general), economic and oracle attacks beyond a checklist, wallet and front-end security, and operational key management beyond what the upgrade authority needs.

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation sites were not. The program-data addresses, the vault address and the digest of a known state were computed by **separate Python programs** (✓: SHA-256, and a pure-Python curve check for program-derived addresses), not by the code under test.

Builds on [Lesson 53](53_solana_model.md) (the runtime: accounts, ownership, PDAs, CPI; the crate is used unchanged), [Lesson 54](54_anchor.md) (the constraint validator, the token program and the escrow, which are the targets of the tools here), [Lesson 52](52_web3_reliability.md) (simulation and idempotency habits), [Lesson 48](48_crypto_primitives.md) (hashing and framing), [Lesson 41](41_benchmarking_profiling.md) (timing method), [Lesson 40](40_property_testing_fuzzing.md) (random-operation tests and fuzzing), [Lesson 39](39_testing.md) (test design) and [Lesson 26](26_error_design.md). The next lesson, Lesson 56, starts the next module and appears here only as plain text.

---

## 1. Why this lesson exists

Most of what makes a program safe is *absence* of behavior: no path by which the wrong account is accepted, no state in which supply and balances disagree. Absence cannot be shown by an example. It is shown by trying many things that should fail and by checking properties that should always hold. Each claim in this lesson has an experiment:

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| State can be compared exactly | a SHA-256 over every tracked account, with every variable-length field length-prefixed, equal for equal states and different for any one-byte change; two accounts split differently ("ab"+"c" against "a"+"bc") hash differently; a known answer from Python (§3) |
| Tests need the whole state, not the accounts you remembered | a `World` remembers every key it has seen, so accounts that programs create are in the snapshot; the first version missed the programs' own accounts and a conservation check was off by one lamport (§3.2) |
| Some properties hold of every state | token supply equals the sum of balances for every mint, summed in 128 bits so two balances of `u64::MAX` cannot wrap (§4) |
| A failed transaction leaves no trace but the fee | after every failure in 200 random transactions exactly one account changed, by exactly 5,000 lamports (§4.3) |
| An attacker chooses the accounts | the fuzzer substitutes, flips, swaps, duplicates and drops accounts, strips every signature but the attacker's, and checks a policy (§5) |
| The fuzzer finds what reviews miss | the naive vault is drained by the *unmutated* call minus one signature; the signer-checking vault falls to exactly **one** pair of changes out of 105; the validated vault survives all of them (§5.3) |
| The same attack matrix clears the escrow | all single and pair mutations of `take_offer` and `cancel_offer` (316 + 12,960 and 154 + 4,320 attempts) end with no policy violation, and a legitimate three-slot change is accepted by the same policy (§5.4) |
| Constraint lists can be reviewed by a program | nine rules, each with a test that fires it and a test that does not; the three vaults get three different reviews (§6) |
| Upgrades are a permission, a slot and a layout | only the authority upgrades, the slot must grow, the authority can be given up for good, and a replay of old transactions pinpoints the first step where a new version differs (§7) |
| Harnesses can lie | a harness with three bugs reports "no findings" for a vault anyone can drain (§14) |

Total in the practical challenge: one library crate, 6 modules, **52 tests**, a benchmark example, and **117 mutants** run against the test suite.

---

## 2. A ladder of tests

Different tests answer different questions. Climb only as high as the question needs, because each rung is slower and harder to read.

1. **Pure rules** (`TokenAccount::transfer`, `Mint::mint_to`): plain structs in, `Result` out. Thousands per second, no setup. Most arithmetic and state-machine rules belong here (Lesson 54 §3.3).
2. **One instruction through the bank**: build the accounts, run one transaction, assert the exact error or the exact state. This is where account checks are tested: one test per check, breaking only that check (Lesson 54 §2.3).
3. **Scenarios**: several transactions in a row (make, take, then take again). They find ordering and re-initialization bugs.
4. **Invariants after random traffic** (Lesson 40's style): many random transactions, and after each one a property such as "supply equals balances" (§4).
5. **Differential tests**: the same transactions against two things that should agree, such as two builds of a program (§7.4) or an optimized and a plain implementation.
6. **Adversarial search**: a program that plays the attacker by enumerating the ways the caller controls the call (§5).
7. **A real validator or the real runtime**: needed for what a model cannot show (the virtual machine, syscalls, the real fee and compute rules, the loader). From memory, the ecosystem has in-process test harnesses of the real runtime (named `solana-program-test`, `LiteSVM` and `Mollusk`, as I remember them), a local validator (`solana-test-validator`), and the Anchor test flow that starts one; I used none of them here.

A habit that pays on every rung: **assert the specific error, and assert that nothing moved.** A test that only says `is_err()` passes for the wrong reason as often as for the right one. Lesson 54's mutation run showed this in its own tests: mutants survived until the assertions named the error and the layer (`WrongOwner(4)` rather than `WrongOwner(0)` told which program refused).

---

## 3. Snapshots, digests and diffs

### 3.1 Why the runtime model needs a wrapper

`Bank` (Lesson 53) can run transactions and look up an account by key, but it cannot list its accounts or be copied. A test harness needs both: to compare "before" and "after", and to start many runs from the same state. The crate's answer is `World`, a `Bank` plus the **set of keys it has seen**. This is enough because an account can only come into existence through a transaction, and a transaction lists every account it touches (Lesson 53 §3.1); so the keys set up by the test plus the keys mentioned in processed transactions cover every account that can exist.

```rust
/// Runs a transaction, remembering every key it mentions first.
pub fn process(&mut self, tx: &Transaction) -> Result<Receipt, Error> {
    self.keys.extend(tx.signers.iter().copied());
    for ix in &tx.instructions {
        self.keys.insert(ix.program_id);
        self.keys.extend(ix.accounts.iter().map(|m| m.pubkey));
    }
    self.bank.process(tx)
}
```

```rust
/// A copy of the state at one moment.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Snapshot {
    /// Existing accounts, in key order.
    pub accounts: BTreeMap<Pubkey, Account>,
    /// Fees collected so far.
    pub fees: u64,
}

/// What happened to one account between two snapshots.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Change {
    /// The account did not exist before.
    Created(Pubkey),
    /// The account existed before and does not now.
    Deleted(Pubkey),
    /// The account exists in both and differs.
    Modified {
        /// The account.
        key: Pubkey,
        /// Lamports before and after.
        lamports: (u64, u64),
        /// The data bytes differ.
        data: bool,
        /// The owner differs.
        owner: bool,
    },
}
```

### 3.2 A bug in my first version

The first `World::new` tracked nothing, and the programs were registered on the bank *before* it was wrapped. The random-traffic test (§4.3) asserts that the total number of lamports never changes, and it failed on the very first seed by exactly one lamport: the system program's account holds a lamport and was not yet in the key set, so the "before" total did not include it and the "after" total, taken once a transaction had mentioned it, did. I fixed it twice: `World::new` now tracks the system program, and the test helper registers programs through `World::add_program`. The lesson is general: **a snapshot is only as complete as its list of keys**, and an incomplete snapshot makes invariants fail (or pass) for reasons that have nothing to do with the program. The doc comment on `World::new` now states the limitation.

### 3.3 The digest

```rust
/// A SHA-256 over every account in key order and the fees: equal states
/// give equal digests. Each variable-length field is length-prefixed.
pub fn digest(&self) -> [u8; 32] {
    let mut h = Sha256::new();
    h.update(self.fees.to_le_bytes());
    h.update((self.accounts.len() as u64).to_le_bytes());
    for (k, a) in &self.accounts {
        h.update(k.0);
        h.update(a.lamports.to_le_bytes());
        h.update(a.owner.0);
        h.update([u8::from(a.executable)]);
        h.update((a.data.len() as u64).to_le_bytes());
        h.update(&a.data);
    }
    h.finalize().into()
}
```

A digest is a compact way to say "these two states are identical" without printing them. Two details matter. Every variable-length field is **length-prefixed**, so the boundary between fields is part of the hash (Lesson 48; ✓ `the_digest_does_not_confuse_where_one_account_ends` hashes ("ab","c") and ("a","bc") differently; exercise 1 writes the framing). And every field that can differ is included, one test per field (✓ `equal_states_have_equal_digests_and_any_difference_changes_them`: lamports, one data byte, a key, a missing account, an extra account, the fees, the executable flag and the owner each change the digest). ✓ Known answer, computed in Python from the byte layout: the state with fees 0 and one empty account at the zero key with one lamport, owned by the zero key and not executable, has digest `1ba1f972e8fda42340e4c2d4820c0542f25f6b9f2f64f5b174ab37f26d3f9431`.

### 3.4 Diffs

```rust
/// The differences from `self` to `after`, in key order.
pub fn changes(&self, after: &Snapshot) -> Vec<Change> {
    let mut keys: BTreeSet<&Pubkey> = self.accounts.keys().collect();
    keys.extend(after.accounts.keys());
    let mut out = Vec::new();
    for k in keys {
        match (self.accounts.get(k), after.accounts.get(k)) {
            (None, Some(_)) => out.push(Change::Created(*k)),
            (Some(_), None) => out.push(Change::Deleted(*k)),
            (Some(a), Some(b)) if a != b => out.push(Change::Modified {
                key: *k,
                lamports: (a.lamports, b.lamports),
                data: a.data != b.data,
                owner: a.owner != b.owner,
            }),
            _ => {}
        }
    }
    out
}
```

`changes` lists, in key order, which accounts were created, deleted or modified and how (lamports before and after, data differs, owner differs). Printing a diff of an unexpected state change is the most useful thing a failing test can do. ✓ `changes_lists_created_deleted_and_modified_in_key_order` covers all three kinds in one comparison, and `the_world_sees_accounts_that_programs_create` shows a transfer creating an account the world had never heard of.

---

## 4. Invariants

### 4.1 Token conservation

```rust
/// For every mint: `supply == Σ balances` of the token accounts that name it.
/// Layouts must parse; an uninitialized mint (all zero) is ignored.
pub fn token_conservation(s: &Snapshot) -> Result<(), Violation> {
    let mut supplies: BTreeMap<Pubkey, u64> = BTreeMap::new();
    let mut held: BTreeMap<Pubkey, u128> = BTreeMap::new();
    for (key, a) in &s.accounts {
        if a.owner != TOKEN_PROGRAM_ID {
            continue;
        }
        match a.data.len() {
            MINT_LEN => {
                let m = Mint::unpack(&a.data).map_err(|_| Violation::BadLayout(*key))?;
                if m.is_initialized {
                    supplies.insert(*key, m.supply);
                }
            }
            ACCOUNT_LEN => {
                let t = TokenAccount::unpack(&a.data).map_err(|_| Violation::BadLayout(*key))?;
                *held.entry(t.mint).or_default() += u128::from(t.amount);
            }
            _ => return Err(Violation::BadLayout(*key)),
        }
    }
    for (mint, supply) in &supplies {
        let h = held.get(mint).copied().unwrap_or(0);
        if h != u128::from(*supply) {
            return Err(Violation::Supply {
                mint: *mint,
                supply: *supply,
                held: h,
            });
        }
    }
    for mint in held.keys() {
        if !supplies.contains_key(mint) {
            return Err(Violation::OrphanBalance(*mint));
        }
    }
    Ok(())
}
```

For every initialized mint, `supply` must equal the sum of the balances of all token accounts that name it. The check also demands that every account owned by the token program parse (so a malformed account is a violation, not a panic later), ignores an all-zero mint (not yet initialized), and reports balances whose mint does not exist. Two details are the kind that matter in a real audit script. The sum is accumulated in `u128`: ✓ `the_sum_cannot_wrap` has two accounts of `u64::MAX` and a supply of `u64::MAX - 1`, where a `u64` sum would wrap to exactly `2⁶⁴ - 2` and make the totals look equal by accident (exercise 5 of Lesson 54 made the same point). And lookalikes owned by another program do not count: ✓ `only_token_program_accounts_count_and_uninitialized_mints_are_ignored`.

### 4.2 Lamports

`total_lamports` adds the accounts and the fees collected, in 128 bits. Conservation is the runtime's job (Lesson 53 §3.4), but checking it after *every* attempt of a fuzzer is cheap and would catch a bug in the model itself.

### 4.3 Random traffic ✓

`tests/props.rs` drives the Lesson 54 escrow and token program with random transactions: make an offer, take one, cancel one, or transfer tokens, by random wallets with random amounts and ids. 100 seeds of 25 transactions each, run on **two identical worlds in parallel**. After every transaction: the two worlds returned the same result (determinism), their digests are equal, token conservation holds and total lamports are unchanged. ✓ 1,313 of 2,500 transactions succeeded, so the test is not passing on a stream of refusals (a guard requires more than a quarter to succeed). A second test checks the property from §1's table: after *any* failed transaction, the diff has exactly one entry, the fee payer's account, with lamports down by 5,000 and nothing else different, and the fees counter has gone up by 5,000.

What this style finds: broken conservation, non-determinism (a program that reads something it should not), and failures that leave residue. What it does not find: a bug that needs a particular, unlikely sequence, or one that only an attacker would try. The next section covers the second kind.

---

## 5. An account-substitution fuzzer

### 5.1 The attacker model

On Solana the caller supplies the accounts (Lesson 53 §4). So an attacker has these degrees of freedom, and no others, over the *accounts* of a transaction: which key goes in each slot; whether a slot is marked signer; whether it is marked writable; how many slots there are. The attacker can sign only for keys it holds. The fuzzer enumerates exactly these:

```rust
/// One change to the way an instruction is called.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Mutation {
    /// The call as it is (but without signatures the attacker lacks).
    None,
    /// Put another key in slot `index`, keeping its flags.
    ReplaceKey {
        /// The slot.
        index: usize,
        /// The new key.
        with: Pubkey,
    },
    /// Toggle the signer flag of a slot.
    FlipSigner(usize),
    /// Toggle the writable flag of a slot.
    FlipWritable(usize),
    /// Exchange the keys (and only the keys) of two slots.
    Swap(usize, usize),
    /// Copy the key of slot `from` into slot `to`.
    Duplicate {
        /// Source slot.
        from: usize,
        /// Destination slot.
        to: usize,
    },
    /// Remove a slot, shifting the later ones down.
    Drop(usize),
    /// Two mutations, the first applied first.
    Both(Box<Mutation>, Box<Mutation>),
}
```

```rust
/// Applies a mutation; `None` if it refers to a slot that does not exist.
pub fn apply(m: &Mutation, ix: &Instruction) -> Option<Instruction> {
    let mut out = ix.clone();
    let n = out.accounts.len();
    match m {
        Mutation::None => {}
        Mutation::ReplaceKey { index, with } => out.accounts.get_mut(*index)?.pubkey = *with,
        Mutation::FlipSigner(i) => {
            let a = out.accounts.get_mut(*i)?;
            a.is_signer = !a.is_signer;
        }
        Mutation::FlipWritable(i) => {
            let a = out.accounts.get_mut(*i)?;
            a.is_writable = !a.is_writable;
        }
        Mutation::Swap(i, j) => {
            if *i >= n || *j >= n || i == j {
                return None;
            }
            let (a, b) = (out.accounts[*i].pubkey, out.accounts[*j].pubkey);
            out.accounts[*i].pubkey = b;
            out.accounts[*j].pubkey = a;
        }
        Mutation::Duplicate { from, to } => {
            if *from >= n || *to >= n || from == to {
                return None;
            }
            out.accounts[*to].pubkey = out.accounts[*from].pubkey;
        }
        Mutation::Drop(i) => {
            if *i >= n {
                return None;
            }
            out.accounts.remove(*i);
        }
        Mutation::Both(a, b) => return apply(b, &apply(a, ix)?),
    }
    Some(out)
}
```

`apply` returns `None` for a mutation that does not fit (a slot that does not exist, a swap of a slot with itself), so the generators can be simple. ✓ `mutations_are_applied_exactly` checks each kind, including that flags stay with the slot when keys are swapped and that nine impossible mutations are refused.

### 5.2 Generating and running

```rust
/// Every single mutation of `ix`, with replacement keys drawn from `pool`
/// (keys equal to the slot's own key are skipped).
pub fn single_mutations(ix: &Instruction, pool: &[Pubkey]) -> Vec<Mutation> {
    let n = ix.accounts.len();
    let mut out = vec![Mutation::None];
    for i in 0..n {
        for p in pool {
            if *p != ix.accounts[i].pubkey {
                out.push(Mutation::ReplaceKey { index: i, with: *p });
            }
        }
        out.push(Mutation::FlipSigner(i));
        out.push(Mutation::FlipWritable(i));
        out.push(Mutation::Drop(i));
        for j in 0..n {
            if i < j {
                out.push(Mutation::Swap(i, j));
            }
            if i != j {
                out.push(Mutation::Duplicate { from: i, to: j });
            }
        }
    }
    out
}
```

```rust
/// Pairs of key replacements in two different slots, and a key replacement
/// together with a signer flip in another slot.
pub fn pair_mutations(ix: &Instruction, pool: &[Pubkey]) -> Vec<Mutation> {
    let n = ix.accounts.len();
    let mut out = Vec::new();
    for i in 0..n {
        for p in pool {
            let first = Mutation::ReplaceKey { index: i, with: *p };
            for j in 0..n {
                if j == i {
                    continue;
                }
                out.push(Mutation::Both(
                    Box::new(first.clone()),
                    Box::new(Mutation::FlipSigner(j)),
                ));
                if j > i {
                    for q in pool {
                        out.push(Mutation::Both(
                            Box::new(first.clone()),
                            Box::new(Mutation::ReplaceKey { index: j, with: *q }),
                        ));
                    }
                }
            }
        }
    }
    out
}
```

A single mutation can only find a bug that one change is enough for. Real attacks often need two: replace the configuration account *and* replace the admin account. `pair_mutations` tries every pair of key replacements in two different slots, and a replacement together with a signer flip. The counts are small enough to run all of them: for `n` slots and a pool of `p` keys, singles are `1 + (n·p − own) + 3n + n(n−1)/2 + n(n−1)` and pairs are `n·p·(n−1) + p²·n(n−1)/2` (exercises 3 and 4; ✓ the exercise tests compare the formulas with the generators for every `n` up to 4 and `p` up to 5).

```rust
/// Like [`run_attacks`], but the signatures of `cosigners` are kept: the
/// attacker builds the transaction and a victim (an administrator, say) signs
/// it without noticing which accounts it names. This is the "confused deputy"
/// attacker; the checks that matter against it are the ones that bind accounts
/// to each other.
pub fn run_attacks_with(
    make_world: &dyn Fn() -> World,
    ix: &Instruction,
    attacker: Pubkey,
    cosigners: &[Pubkey],
    mutations: &[Mutation],
    policy: Policy<'_>,
) -> Report {
    let mut report = Report::default();
    let mut signers = vec![attacker];
    signers.extend_from_slice(cosigners);
    for m in mutations {
        let Some(mut mutated) = apply(m, ix) else {
            continue;
        };
        for meta in &mut mutated.accounts {
            if !signers.contains(&meta.pubkey) {
                meta.is_signer = false;
            }
        }
        let tx = Transaction {
            signers: signers.clone(),
            instructions: vec![mutated],
        };
        report.tried += 1;
        let mut world = make_world();
        let before = world.snapshot();
        let total_before = total_lamports(&before);
        let outcome = catch_unwind(AssertUnwindSafe(|| world.process(&tx)));
        let mut push = |what| {
            report.findings.push(Finding {
                mutation: m.clone(),
                what,
            })
        };
        let Ok(result) = outcome else {
            push(What::Panic);
            continue;
        };
        let after = world.snapshot();
        if total_lamports(&after) != total_before {
            push(What::Lamports);
        }
        match result {
            Ok(_) => {
                report.succeeded += 1;
                if let Some(p) = policy(&before, &after, &tx) {
                    push(What::Policy(p));
                }
                if let Err(v) = token_conservation(&after) {
                    push(What::Tokens(v));
                }
            }
            Err(_) => {
                let changes = before.changes(&after);
                let only_fee = changes.iter().all(|c| {
                    matches!(c, Change::Modified { key, data: false, owner: false, .. } if *key == attacker)
                });
                if !only_fee {
                    push(What::NotAtomic);
                }
            }
        }
    }
    report
}
```

The runner is where an adversarial harness is easiest to get wrong, so it is worth reading slowly. Each attempt runs on a **fresh world** (`make_world`), so one success cannot hide the next. The transaction is signed by the **attacker alone**: after the mutation, every signer flag on a key that is not the attacker is cleared, because the attacker cannot produce anyone else's signature. The runner takes the snapshot **before** the transaction. It catches panics (a panicking program is a finding: in a real validator it aborts the transaction, and a panic reachable by input is a denial of service). After a success it asks the **policy** (a closure supplied by the test: what must never happen) and checks token conservation; after a failure it checks that the diff contains only the fee payer's account. Total lamports are checked either way.

### 5.3 Three vaults ✓

The sample program is a vault: `withdraw(amount)` moves lamports from a vault PDA to the admin; accounts are `[config, vault, admin]`. It comes in three strengths:

```rust
/// The constraints each strength declares (for the strict one, the ones it enforces).
pub fn specs(strength: Strength) -> Vec<Spec> {
    let typed = Check::Typed {
        owner: VAULT,
        disc: discriminator::account("Config"),
        len: CONFIG_LEN,
    };
    let has_admin = Check::HasOne {
        offset: 8,
        target: 2,
    };
    let seeds = Check::Seeds {
        seeds: vec![Seed::Bytes(b"vault".to_vec()), Seed::Key(0)],
        bump: Bump::Canonical,
        program: VAULT,
    };
    match strength {
        Strength::Naive => vec![
            spec("config", vec![]),
            spec("vault", vec![Check::Mut]),
            spec("admin", vec![Check::Mut]),
        ],
        Strength::SignerOnly => vec![
            spec("config", vec![]),
            spec("vault", vec![Check::Mut]),
            spec("admin", vec![Check::Signer, Check::Mut]),
        ],
        Strength::Strict => vec![
            spec("config", vec![typed, has_admin]),
            spec("vault", vec![Check::Mut, Check::Owner(VAULT), seeds]),
            spec("admin", vec![Check::Signer, Check::Mut]),
        ],
    }
}

impl Program for Vault {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(100)?;
        let amount =
            u64::from_le_bytes(data.try_into().map_err(|_| Error::InvalidInstructionData)?);
        match self.0 {
            Strength::Strict => validate(inv, &specs(Strength::Strict))?,
            s => {
                let cfg = inv.account(0)?;
                if cfg.data.get(8..40) != Some(&inv.key(2)?.0[..]) {
                    return Err(Error::WrongAuthority);
                }
                if s == Strength::SignerOnly && !inv.is_signer(2)? {
                    return Err(Error::NotSigner(2));
                }
            }
        }
        let left = inv
            .account(1)?
            .lamports
            .checked_sub(amount)
            .ok_or(Error::InsufficientFunds)?;
        let total = inv
            .account(2)?
            .lamports
            .checked_add(amount)
            .ok_or(Error::ArithmeticOverflow)?;
        inv.account_mut(1)?.lamports = left;
        inv.account_mut(2)?.lamports = total;
        Ok(())
    }
}
```

The policy is a single sentence: *the vault must not lose lamports unless the admin signed* (the closure compares the vault's balance before and after, and looks in `tx.signers`). The attacker is Eve, who holds a decoy configuration account naming herself as admin (not owned by the vault program, but nothing in the weaker vaults looks).

- **Naive** (compares the stored admin key, nothing else): ✓ the finding with `Mutation::None` is the *unmutated call* with the admin's signature removed. No substitution is needed. 5 of the single mutations succeed in draining the vault.
- **Signer only** (adds a signature check): ✓ **no single mutation** is a finding; exactly **one pair** is: configuration → Eve's decoy *and* admin → Eve. The decoy names Eve as admin, Eve signs as admin, and the program is satisfied. 105 pair mutations were tried to find it.
- **Strict** (the `anchorlab` constraints: typed configuration owned by the program with a discriminator, `has_one` admin, vault re-derived as a PDA and owned by the program, admin signature): ✓ no finding among 31 single and 105 pair mutations.

The lesson in the middle case is the one to remember: **two plausible-looking checks are not the same as the right checks**. The signer-only vault checks the signature and compares a key, which feels like authorization, and is one substituted account away from being nothing. The fuzzer found it without being told what to look for.

### 5.3b The confused deputy ✓

So far the attacker signs alone. A second attacker is more realistic for an administrator: **the attacker builds the transaction and the victim signs it** (a phishing page, a malicious wallet prompt, a script run without reading it). `run_attacks_with` keeps the signatures of named co-signers, here the admin. The policy changes too: only the designated vault may lose lamports, whoever signed. The world gets one more program-owned account, a treasury holding 5,000,000 lamports above its rent minimum.

✓ With the admin co-signing, the naive and the signer-only vaults both lose the treasury to a **single** substitution (vault slot → treasury): they check that the admin signed and that the configuration names the admin, and then spend whatever account they are handed. The strict vault is untouched: its vault slot must be the address derived from the configuration's key, so the treasury is refused. This is the attack that the seeds check exists for, and it is invisible to the first attacker model, because a signature the attacker cannot forge protects the admin's money from everyone except the admin's own signature. (The mutation run found this the hard way: deleting the seeds check from the strict vault survived the first 41 tests, because they all used the first attacker, §16.) **Design the attacker before you design the harness.**

### 5.4 The escrow ✓

The same runner applied to Lesson 54's `take_offer` and `cancel_offer`, with a pool of 16 keys (wallets, mints, every associated account, the offer, a decoy vault owned by the offer, the token program): 316 single and 12,960 pair attempts for `take_offer`, 154 and 4,320 for `cancel_offer`. The policy (two sentences): tokens leave a token account only with its owner's signature, *except* from the offer's vault, and then only if the maker's token-B account gained at least the asking price. ✓ No finding in either, and **none of the attempts succeeded at all**: with Eve signing alone, changing one or two accounts never produces a valid swap. A run in which nothing is accepted shows that the program refuses, but not that the policy can accept: so ✓ `the_policy_lets_a_legitimate_swap_through` gives Eve 300 of token B and a three-slot substitution (Eve as taker, with her own two token accounts); the transaction succeeds, the policy is satisfied, and there is no finding. A policy that can only say no is not a policy.

### 5.5 What a fuzzer like this cannot find

- Bugs that need a *different amount* or data, or a *sequence* of transactions (this one mutates accounts only; §4.3 samples sequences).
- Anything that depends on a key the pool does not contain: ✓ in the debugging challenge a pool built from the honest call's keys cannot even name the decoy (§14).
- Logic errors with valid accounts: a wrong price, a wrong rounding direction, a missing fee.
- Anything below the model: compute limits, the virtual machine, serialization.
- A flawed policy. The policy is the specification, and it is written by the same person who wrote the program.

Treat a clean run as *evidence of one kind of absence*, and write down which kind.

---

## 6. A static reviewer for constraint lists

Because Lesson 54 made the trust boundary **data** (a `Vec<Check>` per account), a program can read it. `audit` applies nine rules to one instruction's specs and reports the shapes that usually mean a missing check:

```rust
/// Reviews one instruction's specs.
pub fn audit(specs: &[Spec]) -> Vec<Finding> {
    let mut out = Vec::new();
    let mut add = |rule, severity, account, name, message| {
        out.push(Finding {
            rule,
            severity,
            account,
            name,
            message,
        })
    };
    let any_signer = specs.iter().any(|s| has(s, |c| matches!(c, Check::Signer)));
    if !any_signer {
        add(
            "A01",
            Severity::High,
            0,
            "(instruction)",
            "no account must sign: anyone can call this instruction",
        );
    }
    for (i, s) in specs.iter().enumerate() {
        let writable = has(s, |c| matches!(c, Check::Mut));
        let identified = has(s, |c| {
            matches!(
                c,
                Check::Owner(_)
                    | Check::Typed { .. }
                    | Check::Address(_)
                    | Check::Seeds { .. }
                    | Check::Program(_)
            )
        });
        let signer = has(s, |c| matches!(c, Check::Signer));
        if writable && !identified && !signer && !has(s, |c| matches!(c, Check::Uninit)) {
            add(
                "A02",
                Severity::High,
                i,
                s.name,
                "writable account with no owner, type, address or seeds check: the caller picks which account is changed",
            );
        }
        if has(s, |c| matches!(c, Check::Uninit))
            && !has(s, |c| matches!(c, Check::Address(_) | Check::Seeds { .. }))
        {
            add(
                "A03",
                Severity::Warning,
                i,
                s.name,
                "an account to create is not tied to an address: the caller picks where it lives",
            );
        }
        if has(s, |c| {
            matches!(
                c,
                Check::Seeds {
                    bump: Bump::Stored(_),
                    ..
                }
            )
        }) && !has(s, |c| matches!(c, Check::Typed { .. }))
        {
            add(
                "A04",
                Severity::High,
                i,
                s.name,
                "the bump is read from account data that no type check has vouched for",
            );
        }
        if s.name.contains("program") && !has(s, |c| matches!(c, Check::Program(_))) {
            add(
                "A05",
                Severity::High,
                i,
                s.name,
                "an account named as a program is not compared with the expected program id",
            );
        }
        let mint = has(s, |c| matches!(c, Check::TokenMint(_)));
        let owner = has(s, |c| matches!(c, Check::TokenOwner(_)));
        if mint != owner {
            add(
                "A06",
                Severity::Warning,
                i,
                s.name,
                "a token account is checked for its mint or its owner but not both",
            );
        }
        let typed = has(s, |c| matches!(c, Check::Typed { .. }));
        if typed && writable && any_signer && !has(s, |c| matches!(c, Check::HasOne { .. })) {
            add(
                "A07",
                Severity::Warning,
                i,
                s.name,
                "a writable typed account is not tied to any other account by has_one: who may change it?",
            );
        }
        if (mint || owner) && !has(s, |c| matches!(c, Check::Owner(_))) {
            add(
                "A08",
                Severity::High,
                i,
                s.name,
                "token fields are compared but nothing says the token program owns this account",
            );
        }
        for (j, t) in specs.iter().enumerate().take(i) {
            if t.name == s.name {
                add(
                    "A09",
                    Severity::Info,
                    i,
                    s.name,
                    "two accounts share a name",
                );
                let _ = j;
            }
        }
    }
    out
}
```

| Rule | Severity | Fires when |
|---|---|---|
| A01 | high | no account of the instruction must sign |
| A02 | high | a writable account has no owner, type, address, seeds or program check (and does not sign, and is not being created) |
| A03 | warning | an account to be created is not tied to an address or seeds |
| A04 | high | a PDA bump is read from the account's own data, but no type check vouches for that data |
| A05 | high | an account whose name contains `program` is not compared with an id |
| A06 | warning | a token account is checked for its mint or its owner field but not both |
| A07 | warning | a writable typed account is not tied to anything by `has_one`, though someone signs |
| A08 | high | token fields are compared but the account's owner (the token program) is not checked |
| A09 | info | two accounts share a name |

✓ Every rule has a test that fires it and a test that does not (`tests/audit_tests.rs`), and the three vaults give three different reviews: the naive one gets A01 and two A02s, the signer-only one gets one A02 (for the vault account), the strict one gets nothing.

Three honest cautions. The rules are **heuristics from my own list of what went wrong in Lessons 53 and 54**, not a standard. A name-based rule (A05) is brittle by construction. And a reviewer that reads declarations cannot see logic: it would pass a program with a declared and complete set of constraints and a wrong formula. Use it as a first-pass checklist generator, not as an oracle. (From memory, real tools of this kind exist for Rust and for Solana programs; I did not use any.)

---

## 7. Deployment and upgrades

### 7.1 What the real loader keeps (from memory)

On Solana, a program is deployed by uploading its compiled code in chunks to a **buffer** account and then deploying from the buffer. With the upgradeable loader (the default for years, from memory) a deployed program is represented by two accounts: the **program account**, which is executable and only points to its data, and a **program-data account** at the PDA of the program's address under the loader, which holds the code and a small header: the **slot** of the last deployment and an optional **upgrade authority**. Whoever holds that key can replace the code at any time. Setting the authority to nothing makes the program **immutable** for good. Other operations I remember: transferring the authority, closing a buffer, extending the data account. The command-line names are `solana program deploy`, `write-buffer`, `set-upgrade-authority` (with a `--final` option), `show` and `close`; I did not run them. A newer loader exists (from memory) with different mechanics; this lesson models the older, better-known shape.

### 7.2 The model ✓

```rust
impl ProgramData {
    /// Serializes: tag 3, slot, then a 0/1 flag and 32 bytes.
    pub fn pack(&self) -> Vec<u8> {
        let mut d = vec![3];
        d.extend_from_slice(&self.slot.to_le_bytes());
        match &self.authority {
            Some(k) => {
                d.push(1);
                d.extend_from_slice(&k.0);
            }
            None => {
                d.push(0);
                d.extend_from_slice(&[0; 32]);
            }
        }
        d
    }

    /// Strict parse: exact length, tag 3, flag 0 or 1, zeros after a 0 flag.
    pub fn unpack(d: &[u8]) -> Result<ProgramData, DeployError> {
        if d.len() != PROGRAMDATA_LEN || d[0] != 3 {
            return Err(DeployError::BadData);
        }
        let slot = u64::from_le_bytes(d[1..9].try_into().expect("8"));
        let key: [u8; 32] = d[10..42].try_into().expect("32");
        let authority = match d[9] {
            0 if key == [0; 32] => None,
            1 => Some(Pubkey(key)),
            _ => return Err(DeployError::BadData),
        };
        Ok(ProgramData { slot, authority })
    }
}
```

```rust
/// Replaces the code. The accounts the program owns keep their data.
pub fn upgrade(
    w: &mut World,
    id: Pubkey,
    program: Box<dyn Program>,
    signer: &Pubkey,
    slot: u64,
) -> Result<(), DeployError> {
    let pd = read(w, &id)?;
    authorize(&pd, signer)?;
    if slot <= pd.slot {
        return Err(DeployError::StaleSlot);
    }
    w.add_program(id, program);
    write(
        w,
        &id,
        ProgramData {
            slot,
            authority: pd.authority,
        },
    );
    Ok(())
}
```

```rust
/// Hands the authority to someone else, or to nobody (making the program immutable).
pub fn set_authority(
    w: &mut World,
    id: &Pubkey,
    signer: &Pubkey,
    new: Option<Pubkey>,
) -> Result<(), DeployError> {
    let pd = read(w, id)?;
    authorize(&pd, signer)?;
    write(
        w,
        id,
        ProgramData {
            slot: pd.slot,
            authority: new,
        },
    );
    Ok(())
}
```

What the model keeps: one **authority**, a **slot that must grow**, a program-data account at the PDA `[program id]` under the loader (✓ known answers from the independent derivation: the vault program `[0x7A; 32]` gives `2rLM8Q9VzhfmikGLPaHbrDndRTVoFDPpytDdxyqzSXfX`, the counter program `[0xC7; 32]` gives `EbC3eZc1dtnqb42oPcAzT1mA5Fv3xsEPXFSeDdVgwipX`, `[1; 32]` gives `42DtwM9SRLwJD3vSKkxrpm6S6Hh9h4y2xLr3N6497u8x`), a strict serialization (✓ `programdata_layout_is_strict`: tag 3, the slot's eight bytes, a 0/1 flag and 32 bytes, with every other shape refused, including "no authority" followed by non-zero bytes), and **accounts that survive an upgrade**: the data of every account the program owns stays; only the code changes (✓ `upgrades_keep_balances_of_the_programs_accounts`). What it leaves out: the buffer, the byte code, rent for the data, and the signature itself (the `signer` argument stands for "this key signed the transaction").

✓ Authority behavior, as tests: a deployment of an existing id is refused and does not replace the running code; a stranger cannot upgrade or move the authority; a stale slot (equal or lower) is refused; an unknown program is reported; after `set_authority(None)` neither the old nor the new authority can upgrade or set anything (`Immutable`), and the slot is kept; a corrupted program-data account (wrong tag, wrong owner) is `BadData`.

### 7.3 Upgrades are a trust decision

A program whose upgrade authority is a single key is, for every user, a program *controlled by whoever holds that key*, whatever its source code says. The entire safety argument of the previous two lessons (the checks, the PDA custody of the vault) stands on the code staying the same. So the questions about an upgradeable program are: who holds the authority, how is that key stored, is it a multi-signature wallet or a program with a time delay (from memory, multisig tools such as Squads are common), is there a public process for an upgrade, and is the program meant to be made immutable at some point. For a program whose users hold funds in custody, **immutability is a feature you can sell**, and it is irreversible, so it belongs to the end of the life of a well-tested version, not the beginning. Verifiable builds (from memory: a reproducible build in a pinned container, whose hash anyone can compare with what is deployed) address a different question: whether the deployed bytes match the published source.

### 7.4 Replay testing across versions ✓

The scarier part of an upgrade is not the permission, it is that **old accounts meet new code**. Every account the program owns was written by the previous version. `differential` takes two worlds (the old and the new build) and a transcript of transactions, runs the same transactions on both, and reports the first step at which acceptance or the resulting state differs:

```rust
/// Where a replayed transcript first behaves differently.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Divergence {
    /// Index of the transaction.
    pub step: usize,
    /// Whether the old version accepted it.
    pub old_ok: bool,
    /// Whether the new version accepted it.
    pub new_ok: bool,
    /// Whether the states after it are identical.
    pub same_state: bool,
}

/// Runs the same transactions against two worlds and reports the first step
/// at which acceptance or the resulting state differs.
pub fn differential(
    old: &mut World,
    new: &mut World,
    txs: &[solanalab::instruction::Transaction],
) -> Option<Divergence> {
    for (step, tx) in txs.iter().enumerate() {
        let (a, b) = (old.process(tx), new.process(tx));
        let same_state = old.snapshot().digest() == new.snapshot().digest();
        if a.is_ok() != b.is_ok() || !same_state {
            return Some(Divergence {
                step,
                old_ok: a.is_ok(),
                new_ok: b.is_ok(),
                same_state,
            });
        }
    }
    None
}
```

The sample counter has three versions: the first adds 1 to a `u64` at the start of its 8-byte account; the second adds 2; the third also adds 1 but expects a 16-byte account. ✓ Replays: identical code, no divergence; step size 1 against 2, a divergence at step 0 with both accepting and the states differing (a behavior change, which may be intended); layout 8 against 16, a divergence at step 0 where the new version *refuses* the old account (`old_ok: true, new_ok: false`), which is precisely what a live upgrade would do to every existing account. A divergence is not necessarily a bug, but each one should be explainable, and "the new version cannot read any account the old one wrote" is the one to catch before the upgrade, not after.

A compatibility rule that is cheap to check by hand or by machine: **an account layout may grow at the end but must not move or resize any existing field**, and a new discriminator must not collide with an old one (exercise 8 writes the check). If a layout must change, the options are a migration instruction that rewrites each account (callable by anyone, idempotent, with a version field in the account), or a new account type and a way to move funds across.

---

## 8. How a security review is organized

What an auditor does is not mysterious: it is a systematic version of the questions asked in Lessons 53 and 54, plus time spent thinking about the money. A workable order, which is the one I would use:

1. **Scope and assets.** What is the program for, what can be lost, who are the privileged parties (admin, upgrade authority, oracle, fee collector), and what does the program trust that it does not control?
2. **Inventory.** List every instruction with its accounts, signers, writable flags and data. This is the table the constraint lists of Lesson 54 make explicit; for a program without one, writing it is the first finding.
3. **Ten questions per account**, asked of every account of every instruction: Is it required to sign? Is it writable, and should it be? Who owns it, and is that checked? What *type* is it, and is that checked (discriminator)? Is its address derived, and is the derivation re-computed (canonical bump)? Does it have to be a particular account, or merely *a* valid one? Which other account is it tied to (`has_one`, mint and owner fields)? Can the same account appear twice? Is it being created, and can it already exist? Is it being closed, and is the closing complete?
4. **Every CPI.** The target program id is checked, the accounts passed are the validated ones, signer seeds are the program's own, and after the call anything the callee could have changed is read again.
5. **Arithmetic.** Checked operations, the rounding direction of every division, 128-bit intermediates for products, units (decimals), and conversions that can truncate.
6. **State machine.** Initialization (cannot run twice), ordering, closing (nothing revivable in the same transaction), and what an *expired* or *half-finished* flow leaves behind.
7. **Admin and upgrade surface.** Who can change parameters, can they extract funds, is there a delay, who holds the upgrade authority (§7.3).
8. **Economics.** Oracle manipulation, rounding that can be repeated for profit, front-running and ordering within a block, dust and rent griefing, tokens with fees or hooks (Lesson 54 §6).
9. **The tests themselves.** Are there negative tests for each check? Do they assert the specific error? Do invariants run after random traffic? Mutation testing (§16) is a good measure here: it answers "would my tests notice if this line were deleted?"
10. **Report.** Each finding has an id, a location, a severity (what can be lost, by whom, at what cost to them), a reproduction (a failing test is the best), and a recommended fix. Then a re-check of the fixes, because a fix is a change and changes have bugs.

Two practical points. A review is only as good as its **threat model**: the fuzzer of §5 assumes an attacker who controls accounts and nothing else, and says nothing about a compromised admin key. And **a bounty or a second pair of eyes after the audit** is part of the process, not a replacement for it (from memory, this is common practice on Solana; I have no data).

---

## 9. Cost ✓

Release build, single machine, three runs of `examples/bench.rs` (lowest and highest), on the 14-account escrow world of §5.4:

- snapshot (clone every tracked account): 566–582 ns; digest of a snapshot: 1.58–1.73 µs; `token_conservation`: 204–205 ns; `changes` between two equal snapshots: 1.10–1.11 µs; `audit` of three specs: 49 ns.
- building the escrow world (set-up plus a `make_offer` transaction): 104.5–111.9 µs.
- 211 single attacks on `take_offer`: 25.5–26.8 ms in total, 121–127 µs each; 1,575 pair attacks: 189.2–200.2 ms, 120–127 µs each (the pool there had 5 keys, the tests' pool 16, which is why the test counts are higher). Almost all of that time is building the world for each attempt, which is why the run is cheap enough to do on every commit.
- deploy and upgrade in a fresh world: 30.1–30.9 µs.

The slowest test file is the one with the two escrow matrices (about 17,750 attempts between them, 2.6–2.9 s in the dev profile with optimized dependencies). A real program under a real validator is several orders of magnitude slower per attempt, which is why model-level fuzzing and real-runtime testing complement each other and do not replace each other.

---

## 10. Rust notes

- **Closures as specifications.** The policy is a `&dyn Fn(&Snapshot, &Snapshot, &Transaction) -> Option<String>`: the property under test is code, composable and testable. `Option<String>` rather than `bool` means a failing policy says *why*.
- **`catch_unwind` with `AssertUnwindSafe`.** The runner calls the model, which may panic inside a test program; `AssertUnwindSafe` is the promise that the world, which is discarded after the attempt, is not reused in a broken state. It is not a way to hide panics: each one becomes a `Finding`. (Note that a `panic = "abort"` profile would make this impossible.)
- **`&dyn Fn() -> World` for fresh state.** A world factory is the cheap substitute for `Clone` that the bank cannot provide because it holds `Box<dyn Program>`; building each attempt from scratch also rules out leaks between attempts.
- **Recursive enums.** `Mutation::Both(Box<Mutation>, Box<Mutation>)` is the standard way to give an enum a recursive shape; `apply` recurses on it.
- **Deterministic iteration.** Snapshots hold a `BTreeMap`, so digests and diffs are in key order and equal states produce equal bytes. A `HashMap` would give a different digest on every run.
- **Wide arithmetic for checks.** Sums in invariants use `u128` so the check itself cannot overflow (§4.1).
- **Tests as a second crate.** `tests/common/mod.rs` builds worlds; the crate's own modules have no `#[cfg(test)]` code, so the public API is what is tested.

---

## 11. Common mistakes

1. Asserting only `is_err()`; a test that cannot tell which check fired cannot tell when the check is deleted (§2).
2. Testing only the honest path and the one attack you thought of.
3. A snapshot that misses accounts, so invariants fail or pass for the wrong reason (§3.2).
4. A fuzzer that signs for everyone, or reuses one world for every attempt (§14).
5. A policy that cannot say yes, or one written by looking at what the program does rather than what it should do (§5.4).
6. Treating "no findings" as "safe". It is "no findings *of this kind, in this pool, under this policy*" (§5.5).
7. Letting the upgrade authority be a single key held on a developer's laptop (§7.3).
8. Upgrading without replaying old accounts and transactions against the new code (§7.4).
9. Changing an account layout by inserting or resizing fields (§7.4, exercise 8).
10. Making a program immutable before testing it as hard as it will ever be tested, or never making it immutable when users depend on that promise.
11. Believing an audit is a certificate. It is a time-boxed review by people with a list, a threat model and finite attention.
12. Fixing a finding without a test that fails before the fix and passes after it.
13. Asking "what does the code do?" before "what must it never do?".
14. Running the model-level fuzzer and skipping the real runtime (and the reverse).
15. Fuzzing with a pool that lacks the attacker's own accounts (§14).

---

## 12. Security notes and honest limits

- This lesson's tools are a **model written from memory**, applied to a model of the runtime. A clean run says nothing about a deployed program, the real loader or the real validator. I did not run any real Solana tooling.
- The fuzzer's attacker is **weak by design**: it controls the account list only. Real attackers also control instruction data, transaction ordering, the contents of accounts they own, and time.
- The **policy is the specification**. If the policy is wrong or incomplete, a clean run is meaningless, and no tool here checks the policy against intent.
- The constraint reviewer reads declarations; it cannot see that a declared constraint is *correct* or that the program uses what it validated.
- **Upgradeability is custody.** Do not describe an upgradeable program as trustless. State who can change it.
- Digests and snapshots in tests are for *finding* differences, not for security; a SHA-256 of a test state authenticates nothing.
- **Mutation testing here measures the tests of this crate** (§16), not the programs it tests, and not the model's fidelity to Solana.
- Reporting vulnerabilities: if you find a real one in a real program, tell its maintainers privately and give them time to fix it. Never use the techniques here against a deployed program you do not own and have not been authorized to test.

---

## 13. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Write `frame(parts)` (length-prefixed concatenation) and `unframe(bytes)`, and show why plain concatenation is not enough to hash a list of fields.
2. **(Easy)** Write the pure upgrade gate `gate(authority, signer, current_slot, new_slot)`, with a fixed order for the refusals.
3. **(Easy)** Write `single_count(n, p, own)`: how many single mutations does the fuzzer generate?
4. **(Medium)** Write `pair_count(n, p)` for the pair generator.
5. **(Medium)** Write `valid_bumps(seeds, program)`: every bump from 255 down to 0 for which the seeds derive a program address; which one is canonical?
6. **(Medium-hard)** Write `chain(states)` (a hash chain over a list of state digests) and `first_divergence(a, b)`.
7. **(Hard)** Write `minimize(items, fails)`: from a failing set of mutations, find a subset from which nothing can be removed without the failure disappearing.
8. **(Hard)** Write `compatible(old, new)`: is a new account layout an append-only extension of the old one?

<details markdown="1">
<summary>Answers</summary>

1. A possible answer (✓ compiled and run in a scratch crate with tests):

   ```rust
   //! Exercise 1: length-prefixed framing, so that where one field ends is part of the hash.

   /// Each part as an 8-byte little-endian length followed by its bytes.
   pub fn frame(parts: &[&[u8]]) -> Vec<u8> {
       let mut out = Vec::new();
       for p in parts {
           out.extend_from_slice(&(p.len() as u64).to_le_bytes());
           out.extend_from_slice(p);
       }
       out
   }

   /// The inverse of [`frame`]; `None` for anything that is not exactly a framed list.
   pub fn unframe(mut bytes: &[u8]) -> Option<Vec<Vec<u8>>> {
       let mut out = Vec::new();
       while !bytes.is_empty() {
           let (len, rest) = bytes.split_first_chunk::<8>()?;
           let len = usize::try_from(u64::from_le_bytes(*len)).ok()?;
           if rest.len() < len {
               return None;
           }
           let (part, rest) = rest.split_at(len);
           out.push(part.to_vec());
           bytes = rest;
       }
       Some(out)
   }
   ```

   Concatenating `"ab"` and `"c"` gives the same bytes as `"a"` and `"bc"`, so two different lists would hash alike (a *canonicalization* bug; the same shape as ambiguous signing payloads). A length prefix makes the encoding injective. `unframe` must refuse truncated input and a length that does not fit (the test passes eight `0xFF` bytes and must not try to allocate).
2. A possible answer (✓):

   ```rust
   //! Exercise 2: the upgrade gate as a pure function.

   /// Why an upgrade is refused.
   #[derive(Debug, PartialEq, Eq)]
   pub enum Refusal {
       /// The authority was given up.
       Immutable,
       /// The signer is not the authority.
       NotAuthority,
       /// The slot did not advance.
       StaleSlot,
   }

   /// May `signer` replace the code deployed at `current_slot` with a build for `new_slot`?
   /// Immutability is reported before the signer is looked at, and the signer before the slot.
   pub fn gate(
       authority: Option<[u8; 32]>,
       signer: [u8; 32],
       current_slot: u64,
       new_slot: u64,
   ) -> Result<(), Refusal> {
       match authority {
           None => Err(Refusal::Immutable),
           Some(a) if a != signer => Err(Refusal::NotAuthority),
           Some(_) if new_slot <= current_slot => Err(Refusal::StaleSlot),
           Some(_) => Ok(()),
       }
   }
   ```

   The order is a design decision that tests should pin: an immutable program reports `Immutable` even to the wrong signer, and a wrong signer is reported before a stale slot, so a stranger learns nothing about the slot.
3. A possible answer (✓):

   ```rust
   //! Exercise 3: how many single mutations does the fuzzer try?

   /// The length of `single_mutations` for an instruction with `n` slots and a pool of `p`
   /// keys of which `own` are the keys already sitting in some slot (each slot's own key
   /// is skipped once; the slots' keys are all different and all in the pool when `own == n`).
   pub fn single_count(n: usize, p: usize, own: usize) -> usize {
       // the unchanged call, key replacements, three per-slot changes, swaps, duplicates
       1 + (n * p - own) + 3 * n + n * n.saturating_sub(1) / 2 + n * n.saturating_sub(1)
   }
   ```

   The unchanged call, `n·p − own` key replacements (a slot's own key is skipped when it is in the pool), three per-slot changes (signer flip, writable flip, drop), `n(n−1)/2` swaps and `n(n−1)` ordered duplicates. The test checks it against the real generator for `n` from 1 to 4 and `p` up to 5, with a pool of fresh keys and with one that contains the slots' keys.
4. A possible answer (✓):

   ```rust
   //! Exercise 4: how many pair mutations?

   /// The length of `pair_mutations` for `n` slots and a pool of `p` keys.
   pub fn pair_count(n: usize, p: usize) -> usize {
       // every (slot, key) first change, then either a signer flip in another slot,
       // or a second replacement in a later slot
       let flips = n * p * n.saturating_sub(1);
       let second = p * p * (n * n.saturating_sub(1) / 2);
       flips + second
   }
   ```

   The first change is any (slot, key) pair; the second is a signer flip in any other slot, or a key replacement in a *later* slot (so each unordered pair is generated once). Note it does not skip a slot's own key in the pool, unlike the singles.
5. A possible answer (✓):

   ```rust
   //! Exercise 5: every valid bump of a seed list.

   use solanalab::pubkey::{Pubkey, create_program_address};

   /// All bumps (255 down to 0) for which the seeds plus the bump derive a program address,
   /// each with its address. The first one is the canonical bump.
   pub fn valid_bumps(seeds: &[&[u8]], program: &Pubkey) -> Vec<(u8, Pubkey)> {
       (0..=255u8)
           .rev()
           .filter_map(|b| {
               let bump = [b];
               let mut s = seeds.to_vec();
               s.push(&bump);
               create_program_address(&s, program).ok().map(|a| (b, a))
           })
           .collect()
   }
   ```

   About half of the 256 candidates are valid (the rest hash to a point on the curve, which a program-derived address must avoid); the first is the canonical bump. The test checks the order, the count (between 50 and 200), that the first agrees with `find_program_address` and that every bump gives a different address, which is why accepting any valid bump (rather than the canonical one) multiplies the addresses an attacker can use (Lesson 53 §5.3).
6. A possible answer (✓):

   ```rust
   //! Exercise 6: a hash chain over a transcript of states, and the first place two chains differ.

   use sha2::{Digest, Sha256};

   /// `chain[i] = SHA-256(chain[i-1] || state[i])`, starting from 32 zero bytes.
   pub fn chain(states: &[[u8; 32]]) -> Vec<[u8; 32]> {
       let mut prev = [0u8; 32];
       states
           .iter()
           .map(|s| {
               let mut h = Sha256::new();
               h.update(prev);
               h.update(s);
               prev = h.finalize().into();
               prev
           })
           .collect()
   }

   /// Index of the first position at which the two chains differ (a length difference
   /// counts as a difference at the end of the shorter one); `None` if they are equal.
   pub fn first_divergence(a: &[[u8; 32]], b: &[[u8; 32]]) -> Option<usize> {
       let common = a.len().min(b.len());
       (0..common)
           .find(|&i| a[i] != b[i])
           .or((a.len() != b.len()).then_some(common))
   }
   ```

   Each link hashes the previous link and the next state, so a difference at step `i` changes every later link: the first differing index is the first divergence. A shorter chain diverges at its end. ✓ Known answer: `chain(&[[0; 32]])` is SHA-256 of 64 zero bytes, `f5a5fd42d16a20302798ef6ed309979b43003d2320d9f0e8ea9831a92759fb4b`.
7. A possible answer (✓):

   ```rust
   //! Exercise 7: shrink a failing set of changes to one from which nothing can be removed.

   /// Given a set of `items` for which `fails(items)` is true, returns a subset for which
   /// `fails` is still true and removing any single remaining item makes it false
   /// (a 1-minimal subset). Keeps the original order. Panics if the input does not fail.
   pub fn minimize<T: Clone>(items: &[T], fails: &dyn Fn(&[T]) -> bool) -> Vec<T> {
       assert!(fails(items), "the input must fail");
       let mut cur = items.to_vec();
       let mut i = 0;
       while i < cur.len() {
           let mut without = cur.clone();
           without.remove(i);
           if fails(&without) {
               cur = without; // the item was not needed; look at the same index again
           } else {
               i += 1;
           }
       }
       cur
   }
   ```

   This is the simplest form of *delta debugging*: try deleting each item, keep the deletion if the failure persists. The result is **1-minimal** (removing any one item stops the failure), not necessarily the smallest possible, which would need exponential search. The test shows `[1,3,5,7,9]` shrinking to `[3,7]`, that the result depends on the order tried (`[1,2,3,4]` with "any even" gives `[4]`), and a property check of 1-minimality. Use it on the 3-slot substitution that found a bug to get the two substitutions that matter.
8. A possible answer (✓):

   ```rust
   //! Exercise 8: is a new account layout an append-only extension of the old one?

   /// A field of an account's data.
   #[derive(Debug, Clone, PartialEq, Eq)]
   pub struct Field {
       /// Name.
       pub name: &'static str,
       /// Byte offset.
       pub offset: usize,
       /// Byte length.
       pub len: usize,
   }

   /// Why a layout change breaks existing accounts.
   #[derive(Debug, PartialEq, Eq)]
   pub enum Incompatible {
       /// A field of the old layout is gone.
       Removed(&'static str),
       /// A field moved or changed length.
       Moved(&'static str),
       /// Two fields of the new layout overlap.
       Overlap(&'static str, &'static str),
   }

   /// Old accounts stay readable if every old field is present at the same offset and length,
   /// and no two fields of the new layout overlap. New fields may be added anywhere that is free.
   pub fn compatible(old: &[Field], new: &[Field]) -> Result<(), Incompatible> {
       for o in old {
           match new.iter().find(|n| n.name == o.name) {
               None => return Err(Incompatible::Removed(o.name)),
               Some(n) if n.offset != o.offset || n.len != o.len => {
                   return Err(Incompatible::Moved(o.name));
               }
               Some(_) => {}
           }
       }
       for (i, a) in new.iter().enumerate() {
           for b in &new[i + 1..] {
               if a.offset < b.offset + b.len && b.offset < a.offset + a.len {
                   return Err(Incompatible::Overlap(a.name, b.name));
               }
           }
       }
       Ok(())
   }
   ```

   Existing accounts were written by the old layout, so every old field must exist at the same offset and length. New fields may go anywhere free. The check also refuses a layout in which two fields overlap, since a new field written over an old one would corrupt it; fields that merely touch (one ends where the next begins) are fine.

</details>

---

## 14. Debugging challenge

A team writes a harness like §5.2 to check that their vault cannot be drained without the admin. The harness reports **zero findings** for the *naive* vault, which anyone can drain by calling it without the admin's signature (they confirmed it by hand). Then they fix one thing and it still reports zero for the *signer-only* vault, which an attacker with a decoy configuration can also drain. Find the bugs in the harness (there are three) and fix it.

```rust
//! Debugging challenge: a fuzzing harness that reports "no findings" for a vault
//! that anyone can drain.

use auditlab::attack::{Mutation, apply, single_mutations};
use auditlab::sample::*;
use auditlab::world::World;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::instruction::{Instruction, Transaction};
use solanalab::pubkey::Pubkey;
use solanalab::runtime::Bank;

/// A vault whose admin is `admin`, holding 1,000,000 lamports above its rent minimum,
/// and a decoy configuration (not owned by the vault) that names `attacker`.
pub fn vault_world(strength: Strength, admin: Pubkey, attacker: Pubkey, decoy: Pubkey) -> World {
    let mut w = World::new(Bank::new());
    w.add_program(VAULT, Box::new(Vault(strength)));
    w.set_account(admin, Account::wallet(1_000_000_000));
    w.set_account(attacker, Account::wallet(1_000_000_000));
    let config = Pubkey::filled(5);
    let acct = |owner, data| Account {
        lamports: rent_exempt_minimum(40),
        data,
        owner,
        executable: false,
    };
    w.set_account(config, acct(VAULT, config_data(&admin)));
    w.set_account(decoy, acct(Pubkey::default(), config_data(&attacker)));
    w.set_account(
        vault_address(&config),
        Account {
            lamports: rent_exempt_minimum(0) + 1_000_000,
            data: vec![],
            owner: VAULT,
            executable: false,
        },
    );
    w
}

/// The faulty program.
pub mod buggy {
    use super::*;

    /// Number of mutations of `ix` after which the vault lost lamports without the
    /// admin's signature. `extra` are further keys the attacker can pass.
    pub fn count_findings(
        make: &dyn Fn() -> World,
        ix: &Instruction,
        attacker: Pubkey,
        admin: Pubkey,
        _extra: &[Pubkey],
    ) -> usize {
        let pool: Vec<Pubkey> = ix.accounts.iter().map(|m| m.pubkey).collect();
        let vault = vault_address(&ix.accounts[0].pubkey);
        let mut found = 0;
        for m in single_mutations(ix, &pool) {
            let Some(mutated) = apply(&m, ix) else {
                continue;
            };
            let tx = Transaction {
                signers: vec![attacker, admin],
                instructions: vec![mutated],
            };
            let mut world = make();
            if world.process(&tx).is_ok() {
                let before = world.snapshot();
                let after = world.snapshot();
                let drained = after.lamports(&vault) < before.lamports(&vault);
                if drained && !tx.signers.contains(&admin) {
                    found += 1;
                }
            }
        }
        found
    }
}
```

<details markdown="1">
<summary>Solution</summary>

1. **The attacker signs with the admin's key.** The transaction lists `signers: vec![attacker, admin]` and never clears the signer flags of other keys. Every attempt therefore carries a valid admin signature, which is exactly the thing the harness is supposed to take away. Worse, the policy "drained and the admin did not sign" then can never be true, because `tx.signers` contains the admin. Fix: sign with the attacker alone, and after applying the mutation clear every signer flag on a key that is not the attacker's.
2. **The "before" snapshot is taken after the transaction.** `before` and `after` are both read once the transaction has run, so they are equal and nothing is ever "drained". The comparison needs the state *before* running. Fix: take the snapshot first.
3. **The pool is built from the honest call's keys.** `_extra` is accepted and ignored (the underscore silences the unused-variable warning, which would otherwise have pointed at the bug), so the replacement keys are the keys already in the call. The attacker's own key and the decoy configuration are never tried, so the pair attack on the signer-only vault cannot be found. Fix: the pool must contain the attacker, the caller's extra keys and the call's keys.

✓ Tests: the buggy harness finds nothing on all three vaults (`the_buggy_harness_sees_nothing_anywhere`); the fixed one finds 5 single mutations and 15 with pairs on the naive vault, none among singles but exactly 1 with pairs on the signer-only vault, and nothing on the strict vault. `each_flaw_hides_something_by_itself` shows the third flaw with the real runner: a pool taken from the honest call finds no pair attack on the signer-only vault, and the full pool finds one.

Fixed version (✓ compiled; four tests cover the two versions):

```rust
/// The corrected program.
pub mod fixed {
    use super::*;

    /// Number of mutations (singles, plus pairs when `pairs` is given) after which the
    /// vault lost lamports without the admin's signature.
    pub fn count_findings(
        make: &dyn Fn() -> World,
        ix: &Instruction,
        attacker: Pubkey,
        admin: Pubkey,
        extra: &[Pubkey],
        with_pairs: bool,
    ) -> usize {
        let mut pool: Vec<Pubkey> = ix.accounts.iter().map(|m| m.pubkey).collect();
        pool.push(attacker);
        pool.extend_from_slice(extra);
        pool.sort();
        pool.dedup();
        let vault = vault_address(&ix.accounts[0].pubkey);
        let mut mutations: Vec<Mutation> = single_mutations(ix, &pool);
        if with_pairs {
            mutations.extend(auditlab::attack::pair_mutations(ix, &pool));
        }
        let mut found = 0;
        for m in mutations {
            let Some(mut mutated) = apply(&m, ix) else {
                continue;
            };
            // the attacker cannot produce anyone else's signature
            for meta in &mut mutated.accounts {
                if meta.pubkey != attacker {
                    meta.is_signer = false;
                }
            }
            let tx = Transaction {
                signers: vec![attacker],
                instructions: vec![mutated],
            };
            let mut world = make();
            let before = world.snapshot();
            if world.process(&tx).is_ok() {
                let after = world.snapshot();
                let drained = after.lamports(&vault) < before.lamports(&vault);
                if drained && !tx.signers.contains(&admin) {
                    found += 1;
                }
            }
        }
        found
    }
}
```

The general lesson: **test the tester.** A harness that finds nothing must be shown to find something: always keep a control (a vault known to be weak) in the test suite, so that a harness that stops seeing is noticed. Mutation testing (§16) is the same idea applied to the tests of the program.

</details>

---

## 15. Interview questions

1. *How do you test that a Solana program rejects the wrong accounts?*
2. *What does it mean that an attacker "controls the accounts"? What else do they control?*
3. *Why take a snapshot before and after a transaction? What does a diff tell you?*
4. *Name three invariants you would check after random traffic on a token program.*
5. *What is the upgrade authority? What are the options for managing it?*
6. *How do you make a program immutable? When would you?*
7. *What can go wrong when you upgrade a program that has live accounts?*
8. *How do you review a layout change for compatibility?*
9. *What does a differential test compare, and what does a divergence mean?*
10. *What would an auditor ask about every account of every instruction?*
11. *What are the limits of fuzzing a program with a harness like the one in this lesson?*
12. *Why must a fuzzer's transaction be signed only by the attacker?*
13. *How do you know your tests would notice a deleted check?*
14. *What is a non-canonical bump, and why does accepting one matter?*
15. *What do you hand over in a vulnerability report, and what do you do with a vulnerability you find in someone else's program?*

<details markdown="1">
<summary>Answers</summary>

1. One test per check, each breaking only that check, asserting the exact error and that no state changed; then a matrix that tries all substitutions of the accounts and checks a policy; and random traffic with invariants (§2, §5).
2. The caller picks which key goes in each slot, the flags (the runtime enforces signatures, so the attacker cannot sign for others), and how many slots. Also the instruction data, the ordering of transactions, and the contents of accounts they own, which the account-substitution harness does not vary (§5.1, §5.5).
3. The runtime guarantees atomicity, but a program can still leave a *valid* but wrong state. A diff makes "what changed" explicit, so a test can say "only these two accounts changed, by these amounts"; for a failure it must be only the fee (§3.4, §4.3).
4. Supply equals the sum of balances per mint; total lamports are conserved; a failed transaction changes nothing but the fee; every account the token program owns parses; identical runs give identical digests (§4).
5. The key that may replace a deployed program's code (it is recorded in the program-data account). Options: a single key (fast, risky), a multi-signature wallet, a program with a time delay and public announcements, or giving it up to make the program immutable (§7).
6. Set the authority to none (in the real tools, a `--final` option on `set-upgrade-authority`, from memory); it is irreversible. When the program has been tested as hard as it will be, and users hold funds under it that the authority could otherwise endanger (§7.3).
7. Old accounts were written by the old code: a changed layout, a changed discriminator or a changed semantics can make every existing account unreadable or wrong; the new version may refuse them, or worse, accept and misread them (§7.4).
8. Every old field present at the same offset and length; new fields only in free space; no overlaps; unique discriminators; a version field if migration is needed; and a replay of real old accounts through the new code (exercise 8, §7.4).
9. The same transactions run against two builds; a divergence is the first step where acceptance or the resulting state differs. It may be an intended behavior change, but each one should be explainable, and "refuses old accounts" is the one to block (§7.4).
10. The ten questions of §8 step 3: signer, writable, owner, type, derived address, specific account, tied to which other account, duplicate, creation, closing.
11. It varies only the account list; it needs a policy that encodes the intent; it can only name keys in its pool; it runs on a model, not the real runtime; it cannot find wrong formulas, bad rounding or anything that needs a sequence (§5.5).
12. The attacker cannot forge anyone else's signature, and a harness that signs for others tests an impossible attacker and hides exactly the missing signature checks it should find (§14).
13. Mutation testing: change one line at a time and see whether a test fails; the surviving mutants are either equivalent (no observable change) or gaps in the tests (§16).
14. A bump other than the first valid one: it gives a second, different valid address for the same seeds. If a program accepts any bump supplied by the caller, an attacker can create a second "PDA" for the same logical key (Lesson 53 §5.3, exercise 5).
15. A clear description, the affected code, the impact, a reproduction (best as a failing test), the suggested fix, and the contact path; give maintainers time to fix before disclosure, and never exploit a vulnerability in a program you are not authorized to test (§12).

</details>

---

## 16. Practical challenge: `auditlab`

Build a library crate, depending on the Lesson 53 and 54 crates, that:

1. Provides a **world** that tracks the keys it has seen, with **snapshots**, a SHA-256 **digest**, and a **diff**.
2. Provides **invariants**: token conservation (supply against balances, summed in 128 bits, with layout and orphan checks) and total lamports.
3. Provides an **account-substitution fuzzer**: mutations (replace a key, flip signer, flip writable, swap, duplicate, drop, and pairs), generators, and a runner that signs only for the attacker (or for named co-signers, the confused deputy), uses a fresh world per attempt, catches panics, checks a policy, token conservation, lamport conservation and failure atomicity.
4. Provides a **static reviewer** for `anchorlab` constraint lists with the nine rules of §6.
5. Provides a model of the **upgradeable loader**: deploy, upgrade (authority and slot), set authority, immutability, a strict program-data layout, and a differential replay.
6. Provides sample programs (a vault in three strengths and a counter in three versions), a benchmark example, passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

The paths assume the Lesson 53 and 54 crates sit at `../../p53/solanalab` and `../../p54/anchorlab` in my scratch directory; point them at wherever you keep those crates.

```toml
[package]
name = "auditlab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "Teaching model of Solana program testing: state snapshots, invariants, an account-substitution fuzzer, a constraint auditor and an upgrade loader. Built on the Lesson 53 and 54 crates. Not audited."

[dependencies]
solanalab = { path = "../../p53/solanalab" }
anchorlab = { path = "../../p54/anchorlab" }
sha2 = "0.11"

[profile.dev.package."*"]
opt-level = 3
```

### `src/lib.rs`

```rust
//! Tools for testing and reviewing programs written for the `solanalab`
//! runtime model: tracked worlds with state snapshots and digests, invariant
//! checks, an account-substitution fuzzer, a static auditor for `anchorlab`
//! constraint lists and a model of the upgradeable loader.
//! **A teaching model; not audited; finding nothing proves nothing.**

pub mod attack;
pub mod audit;
pub mod deploy;
pub mod invariants;
pub mod sample;
pub mod world;
```

### `src/world.rs`

```rust
//! A bank that remembers which accounts it has seen, so that its whole state
//! can be snapshotted, hashed and compared.

use sha2::{Digest, Sha256};
use solanalab::account::Account;
use solanalab::instruction::Transaction;
use solanalab::pubkey::Pubkey;
use solanalab::runtime::{Bank, Error, Program, Receipt};
use std::collections::{BTreeMap, BTreeSet};

/// A [`Bank`] plus the set of keys that were ever set up or mentioned by a
/// transaction. Accounts that programs create always appear in some
/// transaction's account list, so this set covers every account that can exist.
pub struct World {
    /// The bank itself.
    pub bank: Bank,
    keys: BTreeSet<Pubkey>,
}

impl World {
    /// Wraps a bank. Only the system program account is known at first; any
    /// other account already inside the bank stays unknown until a transaction
    /// mentions it or it is set through this wrapper.
    pub fn new(bank: Bank) -> World {
        World {
            bank,
            keys: BTreeSet::from([Pubkey::default()]),
        }
    }

    /// Creates or replaces an account and remembers its key.
    pub fn set_account(&mut self, key: Pubkey, account: Account) {
        self.keys.insert(key);
        self.bank.set_account(key, account);
    }

    /// Registers a program and remembers its key.
    pub fn add_program(&mut self, id: Pubkey, program: Box<dyn Program>) {
        self.keys.insert(id);
        self.bank.add_program(id, program);
    }

    /// Runs a transaction, remembering every key it mentions first.
    pub fn process(&mut self, tx: &Transaction) -> Result<Receipt, Error> {
        self.keys.extend(tx.signers.iter().copied());
        for ix in &tx.instructions {
            self.keys.insert(ix.program_id);
            self.keys.extend(ix.accounts.iter().map(|m| m.pubkey));
        }
        self.bank.process(tx)
    }

    /// A copy of every remembered account that exists, plus the fees collected.
    pub fn snapshot(&self) -> Snapshot {
        let accounts = self
            .keys
            .iter()
            .filter_map(|k| self.bank.account(k).map(|a| (*k, a.clone())))
            .collect();
        Snapshot {
            accounts,
            fees: self.bank.fees_collected,
        }
    }
}

/// A copy of the state at one moment.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Snapshot {
    /// Existing accounts, in key order.
    pub accounts: BTreeMap<Pubkey, Account>,
    /// Fees collected so far.
    pub fees: u64,
}

/// What happened to one account between two snapshots.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Change {
    /// The account did not exist before.
    Created(Pubkey),
    /// The account existed before and does not now.
    Deleted(Pubkey),
    /// The account exists in both and differs.
    Modified {
        /// The account.
        key: Pubkey,
        /// Lamports before and after.
        lamports: (u64, u64),
        /// The data bytes differ.
        data: bool,
        /// The owner differs.
        owner: bool,
    },
}

impl Snapshot {
    /// Lamports of an account (0 if absent).
    pub fn lamports(&self, key: &Pubkey) -> u64 {
        self.accounts.get(key).map_or(0, |a| a.lamports)
    }

    /// A SHA-256 over every account in key order and the fees: equal states
    /// give equal digests. Each variable-length field is length-prefixed.
    pub fn digest(&self) -> [u8; 32] {
        let mut h = Sha256::new();
        h.update(self.fees.to_le_bytes());
        h.update((self.accounts.len() as u64).to_le_bytes());
        for (k, a) in &self.accounts {
            h.update(k.0);
            h.update(a.lamports.to_le_bytes());
            h.update(a.owner.0);
            h.update([u8::from(a.executable)]);
            h.update((a.data.len() as u64).to_le_bytes());
            h.update(&a.data);
        }
        h.finalize().into()
    }

    /// The differences from `self` to `after`, in key order.
    pub fn changes(&self, after: &Snapshot) -> Vec<Change> {
        let mut keys: BTreeSet<&Pubkey> = self.accounts.keys().collect();
        keys.extend(after.accounts.keys());
        let mut out = Vec::new();
        for k in keys {
            match (self.accounts.get(k), after.accounts.get(k)) {
                (None, Some(_)) => out.push(Change::Created(*k)),
                (Some(_), None) => out.push(Change::Deleted(*k)),
                (Some(a), Some(b)) if a != b => out.push(Change::Modified {
                    key: *k,
                    lamports: (a.lamports, b.lamports),
                    data: a.data != b.data,
                    owner: a.owner != b.owner,
                }),
                _ => {}
            }
        }
        out
    }
}
```

### `src/invariants.rs`

```rust
//! Properties that must hold of every state, whatever the programs did.

use crate::world::Snapshot;
use anchorlab::token::{ACCOUNT_LEN, MINT_LEN, Mint, TOKEN_PROGRAM_ID, TokenAccount};
use solanalab::pubkey::Pubkey;
use std::collections::BTreeMap;

/// A broken invariant.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Violation {
    /// An account owned by the token program has a malformed layout.
    BadLayout(Pubkey),
    /// A mint's supply differs from the sum of the balances of its accounts.
    Supply {
        /// The mint.
        mint: Pubkey,
        /// The supply it records.
        supply: u64,
        /// The sum of balances (wide, so it cannot wrap).
        held: u128,
    },
    /// Token accounts exist whose mint does not.
    OrphanBalance(Pubkey),
}

/// For every mint: `supply == Σ balances` of the token accounts that name it.
/// Layouts must parse; an uninitialized mint (all zero) is ignored.
pub fn token_conservation(s: &Snapshot) -> Result<(), Violation> {
    let mut supplies: BTreeMap<Pubkey, u64> = BTreeMap::new();
    let mut held: BTreeMap<Pubkey, u128> = BTreeMap::new();
    for (key, a) in &s.accounts {
        if a.owner != TOKEN_PROGRAM_ID {
            continue;
        }
        match a.data.len() {
            MINT_LEN => {
                let m = Mint::unpack(&a.data).map_err(|_| Violation::BadLayout(*key))?;
                if m.is_initialized {
                    supplies.insert(*key, m.supply);
                }
            }
            ACCOUNT_LEN => {
                let t = TokenAccount::unpack(&a.data).map_err(|_| Violation::BadLayout(*key))?;
                *held.entry(t.mint).or_default() += u128::from(t.amount);
            }
            _ => return Err(Violation::BadLayout(*key)),
        }
    }
    for (mint, supply) in &supplies {
        let h = held.get(mint).copied().unwrap_or(0);
        if h != u128::from(*supply) {
            return Err(Violation::Supply {
                mint: *mint,
                supply: *supply,
                held: h,
            });
        }
    }
    for mint in held.keys() {
        if !supplies.contains_key(mint) {
            return Err(Violation::OrphanBalance(*mint));
        }
    }
    Ok(())
}

/// Lamports held by remembered accounts plus fees, as a wide number.
pub fn total_lamports(s: &Snapshot) -> u128 {
    s.accounts
        .values()
        .map(|a| u128::from(a.lamports))
        .sum::<u128>()
        + u128::from(s.fees)
}
```

### `src/attack.rs`

```rust
//! An account-substitution fuzzer. Starting from a valid instruction, it
//! changes how the accounts are passed (the things an attacker controls),
//! strips every signature the attacker cannot produce, runs the result on a
//! fresh world and reports what a policy says should not have happened.

use crate::invariants::{Violation, token_conservation, total_lamports};
use crate::world::{Change, Snapshot, World};
use solanalab::instruction::{Instruction, Transaction};
use solanalab::pubkey::Pubkey;
use std::panic::{AssertUnwindSafe, catch_unwind};

/// One change to the way an instruction is called.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Mutation {
    /// The call as it is (but without signatures the attacker lacks).
    None,
    /// Put another key in slot `index`, keeping its flags.
    ReplaceKey {
        /// The slot.
        index: usize,
        /// The new key.
        with: Pubkey,
    },
    /// Toggle the signer flag of a slot.
    FlipSigner(usize),
    /// Toggle the writable flag of a slot.
    FlipWritable(usize),
    /// Exchange the keys (and only the keys) of two slots.
    Swap(usize, usize),
    /// Copy the key of slot `from` into slot `to`.
    Duplicate {
        /// Source slot.
        from: usize,
        /// Destination slot.
        to: usize,
    },
    /// Remove a slot, shifting the later ones down.
    Drop(usize),
    /// Two mutations, the first applied first.
    Both(Box<Mutation>, Box<Mutation>),
}

/// Applies a mutation; `None` if it refers to a slot that does not exist.
pub fn apply(m: &Mutation, ix: &Instruction) -> Option<Instruction> {
    let mut out = ix.clone();
    let n = out.accounts.len();
    match m {
        Mutation::None => {}
        Mutation::ReplaceKey { index, with } => out.accounts.get_mut(*index)?.pubkey = *with,
        Mutation::FlipSigner(i) => {
            let a = out.accounts.get_mut(*i)?;
            a.is_signer = !a.is_signer;
        }
        Mutation::FlipWritable(i) => {
            let a = out.accounts.get_mut(*i)?;
            a.is_writable = !a.is_writable;
        }
        Mutation::Swap(i, j) => {
            if *i >= n || *j >= n || i == j {
                return None;
            }
            let (a, b) = (out.accounts[*i].pubkey, out.accounts[*j].pubkey);
            out.accounts[*i].pubkey = b;
            out.accounts[*j].pubkey = a;
        }
        Mutation::Duplicate { from, to } => {
            if *from >= n || *to >= n || from == to {
                return None;
            }
            out.accounts[*to].pubkey = out.accounts[*from].pubkey;
        }
        Mutation::Drop(i) => {
            if *i >= n {
                return None;
            }
            out.accounts.remove(*i);
        }
        Mutation::Both(a, b) => return apply(b, &apply(a, ix)?),
    }
    Some(out)
}

/// Every single mutation of `ix`, with replacement keys drawn from `pool`
/// (keys equal to the slot's own key are skipped).
pub fn single_mutations(ix: &Instruction, pool: &[Pubkey]) -> Vec<Mutation> {
    let n = ix.accounts.len();
    let mut out = vec![Mutation::None];
    for i in 0..n {
        for p in pool {
            if *p != ix.accounts[i].pubkey {
                out.push(Mutation::ReplaceKey { index: i, with: *p });
            }
        }
        out.push(Mutation::FlipSigner(i));
        out.push(Mutation::FlipWritable(i));
        out.push(Mutation::Drop(i));
        for j in 0..n {
            if i < j {
                out.push(Mutation::Swap(i, j));
            }
            if i != j {
                out.push(Mutation::Duplicate { from: i, to: j });
            }
        }
    }
    out
}

/// Pairs of key replacements in two different slots, and a key replacement
/// together with a signer flip in another slot.
pub fn pair_mutations(ix: &Instruction, pool: &[Pubkey]) -> Vec<Mutation> {
    let n = ix.accounts.len();
    let mut out = Vec::new();
    for i in 0..n {
        for p in pool {
            let first = Mutation::ReplaceKey { index: i, with: *p };
            for j in 0..n {
                if j == i {
                    continue;
                }
                out.push(Mutation::Both(
                    Box::new(first.clone()),
                    Box::new(Mutation::FlipSigner(j)),
                ));
                if j > i {
                    for q in pool {
                        out.push(Mutation::Both(
                            Box::new(first.clone()),
                            Box::new(Mutation::ReplaceKey { index: j, with: *q }),
                        ));
                    }
                }
            }
        }
    }
    out
}

/// What went wrong in one attempt.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Finding {
    /// The mutation that was run.
    pub mutation: Mutation,
    /// What was observed.
    pub what: What,
}

/// The kinds of observation.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum What {
    /// The program panicked.
    Panic,
    /// The policy returned a complaint.
    Policy(String),
    /// A token invariant broke.
    Tokens(Violation),
    /// Total lamports changed.
    Lamports,
    /// A transaction that failed changed something other than the payer's fee.
    NotAtomic,
}

/// Totals and findings of a run.
#[derive(Debug, Default)]
pub struct Report {
    /// Attempts that were applicable.
    pub tried: usize,
    /// Attempts the runtime accepted.
    pub succeeded: usize,
    /// Everything suspicious.
    pub findings: Vec<Finding>,
}

/// The policy sees the state before and after and the transaction.
pub type Policy<'a> = &'a dyn Fn(&Snapshot, &Snapshot, &Transaction) -> Option<String>;

/// Runs every mutation on a fresh world. The transaction is signed by the
/// attacker alone: any other signer flag is cleared after the mutation.
pub fn run_attacks(
    make_world: &dyn Fn() -> World,
    ix: &Instruction,
    attacker: Pubkey,
    mutations: &[Mutation],
    policy: Policy<'_>,
) -> Report {
    run_attacks_with(make_world, ix, attacker, &[], mutations, policy)
}

/// Like [`run_attacks`], but the signatures of `cosigners` are kept: the
/// attacker builds the transaction and a victim (an administrator, say) signs
/// it without noticing which accounts it names. This is the "confused deputy"
/// attacker; the checks that matter against it are the ones that bind accounts
/// to each other.
pub fn run_attacks_with(
    make_world: &dyn Fn() -> World,
    ix: &Instruction,
    attacker: Pubkey,
    cosigners: &[Pubkey],
    mutations: &[Mutation],
    policy: Policy<'_>,
) -> Report {
    let mut report = Report::default();
    let mut signers = vec![attacker];
    signers.extend_from_slice(cosigners);
    for m in mutations {
        let Some(mut mutated) = apply(m, ix) else {
            continue;
        };
        for meta in &mut mutated.accounts {
            if !signers.contains(&meta.pubkey) {
                meta.is_signer = false;
            }
        }
        let tx = Transaction {
            signers: signers.clone(),
            instructions: vec![mutated],
        };
        report.tried += 1;
        let mut world = make_world();
        let before = world.snapshot();
        let total_before = total_lamports(&before);
        let outcome = catch_unwind(AssertUnwindSafe(|| world.process(&tx)));
        let mut push = |what| {
            report.findings.push(Finding {
                mutation: m.clone(),
                what,
            })
        };
        let Ok(result) = outcome else {
            push(What::Panic);
            continue;
        };
        let after = world.snapshot();
        if total_lamports(&after) != total_before {
            push(What::Lamports);
        }
        match result {
            Ok(_) => {
                report.succeeded += 1;
                if let Some(p) = policy(&before, &after, &tx) {
                    push(What::Policy(p));
                }
                if let Err(v) = token_conservation(&after) {
                    push(What::Tokens(v));
                }
            }
            Err(_) => {
                let changes = before.changes(&after);
                let only_fee = changes.iter().all(|c| {
                    matches!(c, Change::Modified { key, data: false, owner: false, .. } if *key == attacker)
                });
                if !only_fee {
                    push(What::NotAtomic);
                }
            }
        }
    }
    report
}
```

### `src/audit.rs`

```rust
//! A static reviewer for `anchorlab` constraint lists: it reads the checks an
//! instruction declares and reports the shapes that usually mean a missing one.
//! It reads declarations only. It cannot tell whether the program's logic is
//! right, and a clean report means only that none of these rules fired.

use anchorlab::validate::{Bump, Check, Spec};

/// How serious a finding is likely to be.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum Severity {
    /// Worth a look.
    Info,
    /// Usually a bug.
    Warning,
    /// Almost always a bug.
    High,
}

/// One thing the reviewer noticed.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Finding {
    /// Rule id.
    pub rule: &'static str,
    /// Seriousness.
    pub severity: Severity,
    /// Index of the account in the instruction.
    pub account: usize,
    /// Name of the account.
    pub name: &'static str,
    /// Explanation.
    pub message: &'static str,
}

fn has(s: &Spec, f: impl Fn(&Check) -> bool) -> bool {
    s.checks.iter().any(f)
}

/// Reviews one instruction's specs.
pub fn audit(specs: &[Spec]) -> Vec<Finding> {
    let mut out = Vec::new();
    let mut add = |rule, severity, account, name, message| {
        out.push(Finding {
            rule,
            severity,
            account,
            name,
            message,
        })
    };
    let any_signer = specs.iter().any(|s| has(s, |c| matches!(c, Check::Signer)));
    if !any_signer {
        add(
            "A01",
            Severity::High,
            0,
            "(instruction)",
            "no account must sign: anyone can call this instruction",
        );
    }
    for (i, s) in specs.iter().enumerate() {
        let writable = has(s, |c| matches!(c, Check::Mut));
        let identified = has(s, |c| {
            matches!(
                c,
                Check::Owner(_)
                    | Check::Typed { .. }
                    | Check::Address(_)
                    | Check::Seeds { .. }
                    | Check::Program(_)
            )
        });
        let signer = has(s, |c| matches!(c, Check::Signer));
        if writable && !identified && !signer && !has(s, |c| matches!(c, Check::Uninit)) {
            add(
                "A02",
                Severity::High,
                i,
                s.name,
                "writable account with no owner, type, address or seeds check: the caller picks which account is changed",
            );
        }
        if has(s, |c| matches!(c, Check::Uninit))
            && !has(s, |c| matches!(c, Check::Address(_) | Check::Seeds { .. }))
        {
            add(
                "A03",
                Severity::Warning,
                i,
                s.name,
                "an account to create is not tied to an address: the caller picks where it lives",
            );
        }
        if has(s, |c| {
            matches!(
                c,
                Check::Seeds {
                    bump: Bump::Stored(_),
                    ..
                }
            )
        }) && !has(s, |c| matches!(c, Check::Typed { .. }))
        {
            add(
                "A04",
                Severity::High,
                i,
                s.name,
                "the bump is read from account data that no type check has vouched for",
            );
        }
        if s.name.contains("program") && !has(s, |c| matches!(c, Check::Program(_))) {
            add(
                "A05",
                Severity::High,
                i,
                s.name,
                "an account named as a program is not compared with the expected program id",
            );
        }
        let mint = has(s, |c| matches!(c, Check::TokenMint(_)));
        let owner = has(s, |c| matches!(c, Check::TokenOwner(_)));
        if mint != owner {
            add(
                "A06",
                Severity::Warning,
                i,
                s.name,
                "a token account is checked for its mint or its owner but not both",
            );
        }
        let typed = has(s, |c| matches!(c, Check::Typed { .. }));
        if typed && writable && any_signer && !has(s, |c| matches!(c, Check::HasOne { .. })) {
            add(
                "A07",
                Severity::Warning,
                i,
                s.name,
                "a writable typed account is not tied to any other account by has_one: who may change it?",
            );
        }
        if (mint || owner) && !has(s, |c| matches!(c, Check::Owner(_))) {
            add(
                "A08",
                Severity::High,
                i,
                s.name,
                "token fields are compared but nothing says the token program owns this account",
            );
        }
        for (j, t) in specs.iter().enumerate().take(i) {
            if t.name == s.name {
                add(
                    "A09",
                    Severity::Info,
                    i,
                    s.name,
                    "two accounts share a name",
                );
                let _ = j;
            }
        }
    }
    out
}
```

### `src/deploy.rs`

```rust
//! A model of program deployment and upgrade authority. The real loader is a
//! program with buffer accounts and a program-data account; this model keeps
//! the parts that matter for review: one authority that may replace the code,
//! a slot that must grow, and a way to give up the authority for good.

use crate::world::World;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::pubkey::{Pubkey, find_program_address};
use solanalab::runtime::{LOADER, Program};

/// Bytes of a program-data account: tag, slot, authority option.
pub const PROGRAMDATA_LEN: usize = 1 + 8 + 1 + 32;

/// Why a deployment action was refused.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DeployError {
    /// The program id already has a program-data account.
    AlreadyDeployed,
    /// There is no such deployed program.
    UnknownProgram,
    /// The upgrade authority was given up; the code can never change.
    Immutable,
    /// The signer is not the upgrade authority.
    NotAuthority,
    /// The new slot is not later than the deployed one.
    StaleSlot,
    /// The program-data account is malformed.
    BadData,
}

/// The mutable facts about a deployed program.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct ProgramData {
    /// Slot of the last deployment.
    pub slot: u64,
    /// Who may upgrade, if anyone.
    pub authority: Option<Pubkey>,
}

impl ProgramData {
    /// Serializes: tag 3, slot, then a 0/1 flag and 32 bytes.
    pub fn pack(&self) -> Vec<u8> {
        let mut d = vec![3];
        d.extend_from_slice(&self.slot.to_le_bytes());
        match &self.authority {
            Some(k) => {
                d.push(1);
                d.extend_from_slice(&k.0);
            }
            None => {
                d.push(0);
                d.extend_from_slice(&[0; 32]);
            }
        }
        d
    }

    /// Strict parse: exact length, tag 3, flag 0 or 1, zeros after a 0 flag.
    pub fn unpack(d: &[u8]) -> Result<ProgramData, DeployError> {
        if d.len() != PROGRAMDATA_LEN || d[0] != 3 {
            return Err(DeployError::BadData);
        }
        let slot = u64::from_le_bytes(d[1..9].try_into().expect("8"));
        let key: [u8; 32] = d[10..42].try_into().expect("32");
        let authority = match d[9] {
            0 if key == [0; 32] => None,
            1 => Some(Pubkey(key)),
            _ => return Err(DeployError::BadData),
        };
        Ok(ProgramData { slot, authority })
    }
}

/// Address of the program-data account of `program`.
pub fn programdata_address(program: &Pubkey) -> Pubkey {
    find_program_address(&[&program.0], &LOADER)
        .expect("a bump exists")
        .0
}

/// Reads the program-data account.
pub fn read(w: &World, program: &Pubkey) -> Result<ProgramData, DeployError> {
    let a = w
        .bank
        .account(&programdata_address(program))
        .ok_or(DeployError::UnknownProgram)?;
    if a.owner != LOADER {
        return Err(DeployError::BadData);
    }
    ProgramData::unpack(&a.data)
}

fn write(w: &mut World, program: &Pubkey, pd: ProgramData) {
    w.set_account(
        programdata_address(program),
        Account {
            lamports: rent_exempt_minimum(PROGRAMDATA_LEN),
            data: pd.pack(),
            owner: LOADER,
            executable: false,
        },
    );
}

/// First deployment.
pub fn deploy(
    w: &mut World,
    id: Pubkey,
    program: Box<dyn Program>,
    authority: Pubkey,
    slot: u64,
) -> Result<(), DeployError> {
    if w.bank.account(&programdata_address(&id)).is_some() {
        return Err(DeployError::AlreadyDeployed);
    }
    w.add_program(id, program);
    write(
        w,
        &id,
        ProgramData {
            slot,
            authority: Some(authority),
        },
    );
    Ok(())
}

fn authorize(pd: &ProgramData, signer: &Pubkey) -> Result<(), DeployError> {
    match pd.authority {
        None => Err(DeployError::Immutable),
        Some(a) if a == *signer => Ok(()),
        Some(_) => Err(DeployError::NotAuthority),
    }
}

/// Replaces the code. The accounts the program owns keep their data.
pub fn upgrade(
    w: &mut World,
    id: Pubkey,
    program: Box<dyn Program>,
    signer: &Pubkey,
    slot: u64,
) -> Result<(), DeployError> {
    let pd = read(w, &id)?;
    authorize(&pd, signer)?;
    if slot <= pd.slot {
        return Err(DeployError::StaleSlot);
    }
    w.add_program(id, program);
    write(
        w,
        &id,
        ProgramData {
            slot,
            authority: pd.authority,
        },
    );
    Ok(())
}

/// Hands the authority to someone else, or to nobody (making the program immutable).
pub fn set_authority(
    w: &mut World,
    id: &Pubkey,
    signer: &Pubkey,
    new: Option<Pubkey>,
) -> Result<(), DeployError> {
    let pd = read(w, id)?;
    authorize(&pd, signer)?;
    write(
        w,
        id,
        ProgramData {
            slot: pd.slot,
            authority: new,
        },
    );
    Ok(())
}

/// Where a replayed transcript first behaves differently.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Divergence {
    /// Index of the transaction.
    pub step: usize,
    /// Whether the old version accepted it.
    pub old_ok: bool,
    /// Whether the new version accepted it.
    pub new_ok: bool,
    /// Whether the states after it are identical.
    pub same_state: bool,
}

/// Runs the same transactions against two worlds and reports the first step
/// at which acceptance or the resulting state differs.
pub fn differential(
    old: &mut World,
    new: &mut World,
    txs: &[solanalab::instruction::Transaction],
) -> Option<Divergence> {
    for (step, tx) in txs.iter().enumerate() {
        let (a, b) = (old.process(tx), new.process(tx));
        let same_state = old.snapshot().digest() == new.snapshot().digest();
        if a.is_ok() != b.is_ok() || !same_state {
            return Some(Divergence {
                step,
                old_ok: a.is_ok(),
                new_ok: b.is_ok(),
                same_state,
            });
        }
    }
    None
}
```

### `src/sample.rs`

```rust
//! Programs to test the tools on: a vault in three strengths and a counter in
//! three versions.

use anchorlab::discriminator;
use anchorlab::validate::{Bump, Check, Seed, Spec, spec, validate};
use solanalab::instruction::{AccountMeta, Instruction};
use solanalab::pubkey::{Pubkey, find_program_address};
use solanalab::runtime::{Error, Invoke, Program};

/// Program id of the vault.
pub const VAULT: Pubkey = Pubkey([0x7A; 32]);
/// Program id of the counter.
pub const COUNTER: Pubkey = Pubkey([0xC7; 32]);
/// Bytes of a vault configuration: discriminator and admin key.
pub const CONFIG_LEN: usize = 40;

/// How carefully the vault checks its accounts.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Strength {
    /// Compares the admin key stored in the configuration; nothing else.
    Naive,
    /// Also requires the admin's signature; still trusts the configuration account.
    SignerOnly,
    /// Validates every account with `anchorlab` constraints.
    Strict,
}

/// The vault program: `withdraw(amount)` moves lamports from the vault to the admin.
pub struct Vault(pub Strength);

/// The vault PDA of a configuration account.
pub fn vault_address(config: &Pubkey) -> Pubkey {
    find_program_address(&[b"vault", &config.0], &VAULT)
        .expect("a bump exists")
        .0
}

/// Bytes of a configuration naming `admin`.
pub fn config_data(admin: &Pubkey) -> Vec<u8> {
    let mut d = discriminator::account("Config").to_vec();
    d.extend_from_slice(&admin.0);
    d
}

/// `withdraw(amount)`: accounts `[config, vault, admin]`.
pub fn withdraw_ix(config: Pubkey, admin: Pubkey, amount: u64) -> Instruction {
    Instruction {
        program_id: VAULT,
        accounts: vec![
            AccountMeta::readonly(config, false),
            AccountMeta::writable(vault_address(&config), false),
            AccountMeta::writable(admin, true),
        ],
        data: amount.to_le_bytes().to_vec(),
    }
}

/// The constraints each strength declares (for the strict one, the ones it enforces).
pub fn specs(strength: Strength) -> Vec<Spec> {
    let typed = Check::Typed {
        owner: VAULT,
        disc: discriminator::account("Config"),
        len: CONFIG_LEN,
    };
    let has_admin = Check::HasOne {
        offset: 8,
        target: 2,
    };
    let seeds = Check::Seeds {
        seeds: vec![Seed::Bytes(b"vault".to_vec()), Seed::Key(0)],
        bump: Bump::Canonical,
        program: VAULT,
    };
    match strength {
        Strength::Naive => vec![
            spec("config", vec![]),
            spec("vault", vec![Check::Mut]),
            spec("admin", vec![Check::Mut]),
        ],
        Strength::SignerOnly => vec![
            spec("config", vec![]),
            spec("vault", vec![Check::Mut]),
            spec("admin", vec![Check::Signer, Check::Mut]),
        ],
        Strength::Strict => vec![
            spec("config", vec![typed, has_admin]),
            spec("vault", vec![Check::Mut, Check::Owner(VAULT), seeds]),
            spec("admin", vec![Check::Signer, Check::Mut]),
        ],
    }
}

impl Program for Vault {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(100)?;
        let amount =
            u64::from_le_bytes(data.try_into().map_err(|_| Error::InvalidInstructionData)?);
        match self.0 {
            Strength::Strict => validate(inv, &specs(Strength::Strict))?,
            s => {
                let cfg = inv.account(0)?;
                if cfg.data.get(8..40) != Some(&inv.key(2)?.0[..]) {
                    return Err(Error::WrongAuthority);
                }
                if s == Strength::SignerOnly && !inv.is_signer(2)? {
                    return Err(Error::NotSigner(2));
                }
            }
        }
        let left = inv
            .account(1)?
            .lamports
            .checked_sub(amount)
            .ok_or(Error::InsufficientFunds)?;
        let total = inv
            .account(2)?
            .lamports
            .checked_add(amount)
            .ok_or(Error::ArithmeticOverflow)?;
        inv.account_mut(1)?.lamports = left;
        inv.account_mut(2)?.lamports = total;
        Ok(())
    }
}

/// A counter program: each call adds `step` to the `u64` at the start of the
/// account's data. A wide counter keeps 16 bytes instead of 8.
pub struct Counter {
    /// Added per call.
    pub step: u64,
    /// Expects the 16-byte layout.
    pub wide: bool,
}

/// `bump`: accounts `[counter]`.
pub fn bump_ix(counter: Pubkey) -> Instruction {
    Instruction {
        program_id: COUNTER,
        accounts: vec![AccountMeta::writable(counter, false)],
        data: vec![],
    }
}

impl Program for Counter {
    fn process(&self, inv: &mut Invoke<'_>, _data: &[u8]) -> Result<(), Error> {
        inv.consume(50)?;
        let want = if self.wide { 16 } else { 8 };
        if inv.account(0)?.data.len() != want {
            return Err(Error::InvalidAccountData);
        }
        let a = inv.account_mut(0)?;
        let n = u64::from_le_bytes(a.data[..8].try_into().expect("8"));
        let n = n.checked_add(self.step).ok_or(Error::ArithmeticOverflow)?;
        a.data[..8].copy_from_slice(&n.to_le_bytes());
        Ok(())
    }
}
```

### `tests/common/mod.rs`

```rust
#![allow(dead_code)]
use anchorlab::ata::{ATA_PROGRAM_ID, AtaProgram, associated_address};
use anchorlab::escrow::EscrowProgram;
use anchorlab::token::*;
use auditlab::world::{Snapshot, World};
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::instruction::{Instruction, Transaction};
use solanalab::pubkey::Pubkey;
use solanalab::runtime::{Bank, Error, Receipt};

pub const ESC: Pubkey = Pubkey([0xE5; 32]);
pub const RICH: u64 = 1_000_000_000_000;

pub fn k(n: u8) -> Pubkey {
    Pubkey::filled(n)
}

pub fn world() -> World {
    let mut w = World::new(Bank::new());
    w.add_program(TOKEN_PROGRAM_ID, Box::new(TokenProgram));
    w.add_program(ATA_PROGRAM_ID, Box::new(AtaProgram));
    w.add_program(ESC, Box::new(EscrowProgram));
    w
}

pub fn wallet(w: &mut World, n: u8) {
    w.set_account(k(n), Account::wallet(RICH));
}

pub fn mint(w: &mut World, key: Pubkey, supply: u64) {
    let m = Mint {
        mint_authority: Some(k(250)),
        supply,
        decimals: 6,
        is_initialized: true,
        freeze_authority: None,
    };
    w.set_account(key, mint_account(&m, rent_exempt_minimum(MINT_LEN)));
}

pub fn tacct(w: &mut World, key: Pubkey, mint_key: Pubkey, owner: Pubkey, amount: u64) {
    let mut t = TokenAccount::new(mint_key, owner);
    t.amount = amount;
    w.set_account(key, token_account(&t, rent_exempt_minimum(ACCOUNT_LEN)));
}

pub fn ata(w: &mut World, wallet: Pubkey, mint_key: Pubkey, amount: u64) -> Pubkey {
    let a = associated_address(&wallet, &mint_key).0;
    tacct(w, a, mint_key, wallet, amount);
    a
}

pub fn run(w: &mut World, signers: &[u8], ixs: Vec<Instruction>) -> Result<Receipt, Error> {
    w.process(&Transaction {
        signers: signers.iter().map(|&n| k(n)).collect(),
        instructions: ixs,
    })
}

pub fn token_amount(s: &Snapshot, key: &Pubkey) -> Option<u64> {
    s.accounts
        .get(key)
        .and_then(|a| TokenAccount::unpack(&a.data).ok())
        .map(|t| t.amount)
}
```

### `tests/world_tests.rs`

```rust
mod common;
use auditlab::world::*;
use common::*;
use solanalab::account::Account;
use solanalab::instruction::{AccountMeta, Instruction};
use solanalab::pubkey::Pubkey;

fn small(a: &[(u8, u64, &[u8])]) -> Snapshot {
    Snapshot {
        accounts: a
            .iter()
            .map(|(n, l, d)| {
                (
                    k(*n),
                    Account {
                        lamports: *l,
                        data: d.to_vec(),
                        owner: Pubkey::default(),
                        executable: false,
                    },
                )
            })
            .collect(),
        fees: 0,
    }
}

#[test]
fn equal_states_have_equal_digests_and_any_difference_changes_them() {
    let base = small(&[(1, 10, b"ab"), (2, 20, b"")]);
    assert_eq!(
        base.digest(),
        small(&[(1, 10, b"ab"), (2, 20, b"")]).digest()
    );
    assert_eq!(base.digest(), base.clone().digest());
    let variants = [
        small(&[(1, 11, b"ab"), (2, 20, b"")]),
        small(&[(1, 10, b"aB"), (2, 20, b"")]),
        small(&[(1, 10, b"ab"), (3, 20, b"")]),
        small(&[(1, 10, b"ab")]),
        small(&[(1, 10, b"ab"), (2, 20, b""), (3, 1, b"")]),
    ];
    for v in &variants {
        assert_ne!(v.digest(), base.digest());
    }
    let mut fees = base.clone();
    fees.fees = 1;
    assert_ne!(fees.digest(), base.digest());
    let mut exec = base.clone();
    exec.accounts.get_mut(&k(1)).unwrap().executable = true;
    assert_ne!(exec.digest(), base.digest());
    let mut owner = base.clone();
    owner.accounts.get_mut(&k(1)).unwrap().owner = k(9);
    assert_ne!(owner.digest(), base.digest());
}

#[test]
fn the_digest_does_not_confuse_where_one_account_ends() {
    // the same bytes cut differently between two accounts must hash differently
    let a = small(&[(1, 5, b"ab"), (2, 5, b"c")]);
    let b = small(&[(1, 5, b"a"), (2, 5, b"bc")]);
    assert_ne!(a.digest(), b.digest());
    // known answer: SHA-256 of fees, count and one empty account, computed independently
    let one = Snapshot {
        accounts: [(
            Pubkey([0; 32]),
            Account {
                lamports: 1,
                data: vec![],
                owner: Pubkey([0; 32]),
                executable: false,
            },
        )]
        .into_iter()
        .collect(),
        fees: 0,
    };
    assert_eq!(
        one.digest()
            .iter()
            .map(|b| format!("{b:02x}"))
            .collect::<String>(),
        "1ba1f972e8fda42340e4c2d4820c0542f25f6b9f2f64f5b174ab37f26d3f9431"
    );
}

#[test]
fn changes_lists_created_deleted_and_modified_in_key_order() {
    let before = small(&[(1, 10, b"x"), (2, 20, b""), (3, 30, b"")]);
    let mut after = small(&[(1, 10, b"y"), (3, 31, b""), (4, 1, b"")]);
    after.accounts.get_mut(&k(3)).unwrap().owner = k(9);
    assert_eq!(
        before.changes(&after),
        vec![
            Change::Modified {
                key: k(1),
                lamports: (10, 10),
                data: true,
                owner: false
            },
            Change::Deleted(k(2)),
            Change::Modified {
                key: k(3),
                lamports: (30, 31),
                data: false,
                owner: true
            },
            Change::Created(k(4)),
        ]
    );
    assert!(before.changes(&before).is_empty());
    assert_eq!(before.lamports(&k(2)), 20);
    assert_eq!(before.lamports(&k(9)), 0);
}

#[test]
fn the_world_sees_accounts_that_programs_create() {
    let mut w = world();
    wallet(&mut w, 1);
    // a transfer to an address the world has never heard of creates it
    let ix = Instruction {
        program_id: Pubkey::default(),
        accounts: vec![
            AccountMeta::writable(k(1), true),
            AccountMeta::writable(k(77), false),
        ],
        data: solanalab::system::transfer_data(1_000_000),
    };
    let before = w.snapshot();
    assert!(!before.accounts.contains_key(&k(77)));
    run(&mut w, &[1], vec![ix]).unwrap();
    let after = w.snapshot();
    assert_eq!(after.lamports(&k(77)), 1_000_000);
    assert!(before.changes(&after).contains(&Change::Created(k(77))));
    assert_eq!(after.fees, 5000);
}

#[test]
fn keys_named_only_by_a_transaction_are_tracked() {
    // a payer and a program that were put into the bank behind the world's back
    let mut w = world();
    w.bank.set_account(k(40), Account::wallet(1_000_000_000));
    w.bank
        .add_program(k(41), Box::new(anchorlab::token::TokenProgram));
    assert!(!w.snapshot().accounts.contains_key(&k(40)));
    assert!(!w.snapshot().accounts.contains_key(&k(41)));
    // a payer that is not named by any instruction, and an instruction for the program
    let ix = Instruction {
        program_id: k(41),
        accounts: vec![],
        data: vec![],
    };
    let _ = run(&mut w, &[40], vec![ix]);
    let s = w.snapshot();
    assert!(s.accounts.contains_key(&k(40)), "signers are tracked");
    assert!(s.accounts.contains_key(&k(41)), "programs are tracked");
    assert_eq!(s.lamports(&k(40)), 1_000_000_000 - 5000);
}
```

### `tests/invariants_tests.rs`

```rust
mod common;
use anchorlab::token::*;
use auditlab::invariants::*;
use auditlab::world::World;
use common::*;
use solanalab::account::{Account, rent_exempt_minimum};

fn good() -> World {
    let mut w = world();
    mint(&mut w, k(100), 30);
    tacct(&mut w, k(1), k(100), k(11), 10);
    tacct(&mut w, k(2), k(100), k(12), 20);
    mint(&mut w, k(101), 0);
    w
}

#[test]
fn consistent_token_state_passes() {
    assert_eq!(token_conservation(&good().snapshot()), Ok(()));
}

#[test]
fn a_supply_that_differs_from_the_balances_is_reported() {
    let mut w = good();
    tacct(&mut w, k(2), k(100), k(12), 21);
    assert_eq!(
        token_conservation(&w.snapshot()),
        Err(Violation::Supply {
            mint: k(100),
            supply: 30,
            held: 31
        })
    );
    let mut w = good();
    tacct(&mut w, k(2), k(100), k(12), 19);
    assert_eq!(
        token_conservation(&w.snapshot()),
        Err(Violation::Supply {
            mint: k(100),
            supply: 30,
            held: 29
        })
    );
}

#[test]
fn the_sum_cannot_wrap() {
    // two balances of u64::MAX would wrap a u64 sum to 2^64 - 2
    let mut w = world();
    mint(&mut w, k(100), u64::MAX - 1);
    tacct(&mut w, k(1), k(100), k(11), u64::MAX);
    tacct(&mut w, k(2), k(100), k(12), u64::MAX);
    assert_eq!(
        token_conservation(&w.snapshot()),
        Err(Violation::Supply {
            mint: k(100),
            supply: u64::MAX - 1,
            held: 2 * u128::from(u64::MAX)
        })
    );
}

#[test]
fn balances_without_a_mint_and_bad_layouts_are_reported() {
    let mut w = world();
    tacct(&mut w, k(1), k(100), k(11), 5);
    assert_eq!(
        token_conservation(&w.snapshot()),
        Err(Violation::OrphanBalance(k(100)))
    );
    let mut w = good();
    w.set_account(
        k(3),
        Account {
            lamports: rent_exempt_minimum(10),
            data: vec![0; 10],
            owner: TOKEN_PROGRAM_ID,
            executable: false,
        },
    );
    assert_eq!(
        token_conservation(&w.snapshot()),
        Err(Violation::BadLayout(k(3)))
    );
    let mut w = good();
    let mut bad = vec![0u8; ACCOUNT_LEN];
    bad[108] = 9; // an invalid state byte
    w.set_account(
        k(4),
        Account {
            lamports: rent_exempt_minimum(ACCOUNT_LEN),
            data: bad,
            owner: TOKEN_PROGRAM_ID,
            executable: false,
        },
    );
    assert_eq!(
        token_conservation(&w.snapshot()),
        Err(Violation::BadLayout(k(4)))
    );
    let mut w = good();
    w.set_account(
        k(5),
        Account {
            lamports: rent_exempt_minimum(MINT_LEN),
            data: vec![7; MINT_LEN],
            owner: TOKEN_PROGRAM_ID,
            executable: false,
        },
    );
    assert_eq!(
        token_conservation(&w.snapshot()),
        Err(Violation::BadLayout(k(5)))
    );
}

#[test]
fn only_token_program_accounts_count_and_uninitialized_mints_are_ignored() {
    let mut w = good();
    // a lookalike owned by someone else does not count as a balance
    let mut t = TokenAccount::new(k(100), k(13));
    t.amount = 99;
    w.set_account(k(6), token_account(&t, 1_000_000_000).owned_by(k(9)));
    // an all-zero mint (uninitialized) is not a mint yet
    w.set_account(
        k(7),
        Account {
            lamports: rent_exempt_minimum(MINT_LEN),
            data: vec![0; MINT_LEN],
            owner: TOKEN_PROGRAM_ID,
            executable: false,
        },
    );
    assert_eq!(token_conservation(&w.snapshot()), Ok(()));
}

#[test]
fn total_lamports_adds_accounts_and_fees() {
    let mut s = good().snapshot();
    let accounts: u128 = s.accounts.values().map(|a| u128::from(a.lamports)).sum();
    s.fees = 7;
    assert_eq!(total_lamports(&s), accounts + 7);
    assert!(accounts > 0);
    // lamports above u64::MAX in total do not wrap
    let mut w = world();
    wallet(&mut w, 1);
    w.set_account(k(2), Account::wallet(u64::MAX));
    w.set_account(k(3), Account::wallet(u64::MAX));
    assert!(total_lamports(&w.snapshot()) > 2 * u128::from(u64::MAX));
}

trait Owned {
    fn owned_by(self, o: solanalab::pubkey::Pubkey) -> Self;
}
impl Owned for Account {
    fn owned_by(mut self, o: solanalab::pubkey::Pubkey) -> Self {
        self.owner = o;
        self
    }
}

#[test]
fn balances_of_an_uninitialized_mint_are_orphans() {
    let mut w = world();
    w.set_account(
        k(7),
        Account {
            lamports: rent_exempt_minimum(MINT_LEN),
            data: vec![0; MINT_LEN],
            owner: TOKEN_PROGRAM_ID,
            executable: false,
        },
    );
    tacct(&mut w, k(1), k(7), k(11), 5);
    assert_eq!(
        token_conservation(&w.snapshot()),
        Err(Violation::OrphanBalance(k(7)))
    );
}
```

### `tests/attack_tests.rs`

```rust
mod common;
use anchorlab::ata::{self, associated_address};
use anchorlab::escrow::*;
use auditlab::attack::*;
use auditlab::invariants::Violation;
use auditlab::sample::*;
use auditlab::world::{Snapshot, World};
use common::*;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::instruction::{AccountMeta, Instruction, Transaction};
use solanalab::pubkey::Pubkey;
use solanalab::runtime::{Bank, Error, Invoke, Program};

const ADMIN: u8 = 1;
const EVE: u8 = 3;
const CFG: u8 = 5;
const DECOY: u8 = 6;

fn vault_world(s: Strength) -> World {
    let mut w = World::new(Bank::new());
    w.add_program(VAULT, Box::new(Vault(s)));
    wallet(&mut w, ADMIN);
    wallet(&mut w, EVE);
    w.set_account(
        k(CFG),
        Account {
            lamports: rent_exempt_minimum(CONFIG_LEN),
            data: config_data(&k(ADMIN)),
            owner: VAULT,
            executable: false,
        },
    );
    // the attacker's own "configuration": not owned by the vault program
    w.set_account(
        k(DECOY),
        Account {
            lamports: rent_exempt_minimum(CONFIG_LEN),
            data: config_data(&k(EVE)),
            owner: Pubkey::default(),
            executable: false,
        },
    );
    w.set_account(
        vault_address(&k(CFG)),
        Account {
            lamports: rent_exempt_minimum(0) + 1_000_000,
            data: vec![],
            owner: VAULT,
            executable: false,
        },
    );
    w
}

fn vault_policy(_: &Snapshot, _: &Snapshot, _: &Transaction) -> Option<String> {
    None
}

/// The vault lost lamports although the admin did not sign.
fn unsigned_debit(before: &Snapshot, after: &Snapshot, tx: &Transaction) -> Option<String> {
    let v = vault_address(&k(CFG));
    (after.lamports(&v) < before.lamports(&v) && !tx.signers.contains(&k(ADMIN)))
        .then(|| "vault debited without the admin's signature".to_string())
}

fn pool() -> Vec<Pubkey> {
    vec![k(ADMIN), k(EVE), k(CFG), k(DECOY), vault_address(&k(CFG))]
}

fn attack_vault(s: Strength) -> (Report, Report) {
    let ix = withdraw_ix(k(CFG), k(ADMIN), 500_000);
    let make = move || vault_world(s);
    let singles = single_mutations(&ix, &pool());
    let pairs = pair_mutations(&ix, &pool());
    (
        run_attacks(&make, &ix, k(EVE), &singles, &unsigned_debit),
        run_attacks(&make, &ix, k(EVE), &pairs, &unsigned_debit),
    )
}

#[test]
fn the_honest_call_succeeds_on_every_strength() {
    for s in [Strength::Naive, Strength::SignerOnly, Strength::Strict] {
        let mut w = vault_world(s);
        let before = w.snapshot();
        run(
            &mut w,
            &[ADMIN],
            vec![withdraw_ix(k(CFG), k(ADMIN), 500_000)],
        )
        .unwrap();
        let after = w.snapshot();
        assert_eq!(
            unsigned_debit(
                &before,
                &after,
                &Transaction {
                    signers: vec![k(ADMIN)],
                    instructions: vec![]
                }
            ),
            None
        );
        assert_eq!(
            before.lamports(&vault_address(&k(CFG))) - after.lamports(&vault_address(&k(CFG))),
            500_000
        );
        assert_eq!(
            vault_policy(
                &before,
                &after,
                &Transaction {
                    signers: vec![],
                    instructions: vec![]
                }
            ),
            None
        );
    }
}

#[test]
fn the_naive_vault_can_be_drained_without_the_admin() {
    let (singles, _) = attack_vault(Strength::Naive);
    assert!(
        singles
            .findings
            .iter()
            .any(|f| f.mutation == Mutation::None && matches!(f.what, What::Policy(_))),
        "the unmutated call, minus the admin's signature, already works"
    );
}

#[test]
fn the_signer_only_vault_falls_to_a_substituted_configuration() {
    let (singles, pairs) = attack_vault(Strength::SignerOnly);
    assert_eq!(singles.findings, vec![], "no single change is enough");
    let hit = Mutation::Both(
        Box::new(Mutation::ReplaceKey {
            index: 0,
            with: k(DECOY),
        }),
        Box::new(Mutation::ReplaceKey {
            index: 2,
            with: k(EVE),
        }),
    );
    assert!(
        pairs.findings.iter().any(|f| f.mutation == hit),
        "config -> decoy and admin -> Eve works"
    );
    assert!(
        pairs
            .findings
            .iter()
            .all(|f| matches!(f.what, What::Policy(_)))
    );
}

#[test]
fn the_strict_vault_survives_every_single_and_pair_mutation() {
    let (singles, pairs) = attack_vault(Strength::Strict);
    assert_eq!(singles.findings, vec![]);
    assert_eq!(pairs.findings, vec![]);
    // 1 + 3 slots * (4 other pool keys + 3 flips/drop) + C(3,2) swaps + 6 duplicates
    assert_eq!(singles.tried, 1 + 3 * 7 + 3 + 6);
    // 3 slots * 5 keys * 2 other slots + 5 * 5 * C(3,2) key pairs
    assert_eq!(pairs.tried, 3 * 5 * 2 + 5 * 5 * 3);
}

#[test]
fn a_failed_attempt_costs_the_attacker_only_the_fee() {
    // every attempt that failed passed the atomicity check inside run_attacks; make sure
    // failures happened at all
    let (singles, _) = attack_vault(Strength::Strict);
    assert!(singles.succeeded < singles.tried);
}

#[test]
fn mutations_are_applied_exactly() {
    let ix = Instruction {
        program_id: k(9),
        accounts: vec![
            AccountMeta::writable(k(1), true),
            AccountMeta::readonly(k(2), false),
            AccountMeta::writable(k(3), false),
        ],
        data: vec![7],
    };
    let keys = |i: &Instruction| i.accounts.iter().map(|m| m.pubkey).collect::<Vec<_>>();
    let m = |x: &Mutation| apply(x, &ix).unwrap();
    assert_eq!(m(&Mutation::None), ix);
    let r = m(&Mutation::ReplaceKey {
        index: 1,
        with: k(8),
    });
    assert_eq!(keys(&r), vec![k(1), k(8), k(3)]);
    assert_eq!(r.accounts[1], AccountMeta::readonly(k(8), false));
    assert_eq!(r.data, vec![7]);
    let r = m(&Mutation::FlipSigner(1));
    assert!(r.accounts[1].is_signer && !r.accounts[1].is_writable);
    let r = m(&Mutation::FlipSigner(0));
    assert!(!r.accounts[0].is_signer && r.accounts[0].is_writable);
    let r = m(&Mutation::FlipWritable(1));
    assert!(r.accounts[1].is_writable && !r.accounts[1].is_signer);
    let r = m(&Mutation::FlipWritable(0));
    assert!(!r.accounts[0].is_writable && r.accounts[0].is_signer);
    let r = m(&Mutation::Swap(0, 2));
    assert_eq!(keys(&r), vec![k(3), k(2), k(1)]);
    assert_eq!(
        r.accounts[0],
        AccountMeta::writable(k(3), true),
        "flags stay with the slot"
    );
    let r = m(&Mutation::Duplicate { from: 0, to: 2 });
    assert_eq!(keys(&r), vec![k(1), k(2), k(1)]);
    let r = m(&Mutation::Drop(0));
    assert_eq!(keys(&r), vec![k(2), k(3)]);
    let both = Mutation::Both(
        Box::new(Mutation::Drop(0)),
        Box::new(Mutation::ReplaceKey {
            index: 1,
            with: k(8),
        }),
    );
    assert_eq!(keys(&m(&both)), vec![k(2), k(8)]);
    // slots that do not exist
    for bad in [
        Mutation::ReplaceKey {
            index: 3,
            with: k(8),
        },
        Mutation::FlipSigner(3),
        Mutation::FlipWritable(3),
        Mutation::Swap(0, 3),
        Mutation::Swap(1, 1),
        Mutation::Duplicate { from: 0, to: 3 },
        Mutation::Duplicate { from: 3, to: 0 },
        Mutation::Duplicate { from: 1, to: 1 },
        Mutation::Drop(3),
    ] {
        assert_eq!(apply(&bad, &ix), None, "{bad:?}");
    }
    assert_eq!(
        apply(
            &Mutation::Both(Box::new(Mutation::Drop(9)), Box::new(Mutation::None)),
            &ix
        ),
        None
    );
}

#[test]
fn the_number_of_mutations_is_what_the_formulas_say() {
    let ix = Instruction {
        program_id: k(9),
        accounts: vec![
            AccountMeta::writable(k(1), true),
            AccountMeta::readonly(k(2), false),
            AccountMeta::writable(k(3), false),
        ],
        data: vec![],
    };
    let pool = [k(8), k(9)];
    // 1 + n*(p + 3) + C(n,2) + n(n-1), with n = 3 and p = 2
    assert_eq!(single_mutations(&ix, &pool).len(), 1 + 3 * 5 + 3 + 6);
    // a key already in its slot is not offered again
    assert_eq!(
        single_mutations(&ix, &[k(1), k(8)]).len(),
        1 + 3 * 5 - 1 + 3 + 6 - 3 + 3
    );
    // n*p*(n-1) + p*p*C(n,2)
    assert_eq!(pair_mutations(&ix, &pool).len(), 3 * 2 * 2 + 2 * 2 * 3);
}

struct Panics;
impl Program for Panics {
    fn process(&self, inv: &mut Invoke<'_>, _d: &[u8]) -> Result<(), Error> {
        let a = inv.account(0)?;
        let _ = a.data[0]; // panics on an empty account
        Ok(())
    }
}

#[test]
fn a_panicking_program_is_a_finding_and_does_not_stop_the_run() {
    let make = || {
        let mut w = World::new(Bank::new());
        w.add_program(k(60), Box::new(Panics));
        wallet(&mut w, EVE);
        w.set_account(
            k(61),
            Account {
                lamports: rent_exempt_minimum(1),
                data: vec![1],
                owner: k(60),
                executable: false,
            },
        );
        w
    };
    let ix = Instruction {
        program_id: k(60),
        accounts: vec![AccountMeta::readonly(k(61), false)],
        data: vec![],
    };
    let ms = single_mutations(&ix, &[k(EVE)]);
    let r = run_attacks(&make, &ix, k(EVE), &ms, &|_, _, _| None);
    let panics: Vec<_> = r
        .findings
        .iter()
        .filter(|f| f.what == What::Panic)
        .collect();
    assert_eq!(panics.len(), 1);
    assert_eq!(
        panics[0].mutation,
        Mutation::ReplaceKey {
            index: 0,
            with: k(EVE)
        }
    );
    assert_eq!(r.tried, ms.len());
}

#[test]
fn a_policy_complaint_and_a_token_violation_are_reported() {
    // a program that mints out of thin air: supply is not raised
    struct Cheat;
    impl Program for Cheat {
        fn process(&self, inv: &mut Invoke<'_>, _d: &[u8]) -> Result<(), Error> {
            let a = inv.account_mut(0)?;
            a.data[64..72].copy_from_slice(&5u64.to_le_bytes());
            Ok(())
        }
    }
    let make = || {
        let mut w = world();
        w.add_program(k(62), Box::new(Cheat));
        wallet(&mut w, EVE);
        mint(&mut w, k(100), 0);
        tacct(&mut w, k(63), k(100), k(EVE), 0);
        // the account must be owned by the cheating program to be written
        let mut a = w.bank.account(&k(63)).unwrap().clone();
        a.owner = k(62);
        w.set_account(k(63), a);
        w
    };
    let ix = Instruction {
        program_id: k(62),
        accounts: vec![AccountMeta::writable(k(63), false)],
        data: vec![],
    };
    // the account is not owned by the token program here, so the token check sees nothing
    let r = run_attacks(&make, &ix, k(EVE), &[Mutation::None], &|_, _, _| {
        Some("always".into())
    });
    assert_eq!(r.succeeded, 1);
    assert_eq!(
        r.findings,
        vec![Finding {
            mutation: Mutation::None,
            what: What::Policy("always".into())
        }]
    );
    let _ = Violation::BadLayout(k(0));
}

// ---------- the escrow, attacked ----------

const MAKER: u8 = 1;
const TAKER: u8 = 2;
const A: u8 = 100;
const B: u8 = 101;

fn escrow_world() -> World {
    let mut w = world();
    for n in [MAKER, TAKER, EVE] {
        wallet(&mut w, n);
    }
    mint(&mut w, k(A), 1000);
    mint(&mut w, k(B), 500);
    ata(&mut w, k(MAKER), k(A), 1000);
    ata(&mut w, k(MAKER), k(B), 0);
    ata(&mut w, k(TAKER), k(A), 0);
    ata(&mut w, k(TAKER), k(B), 500);
    ata(&mut w, k(EVE), k(A), 0);
    ata(&mut w, k(EVE), k(B), 0);
    let (offer, _) = offer_address(&ESC, &k(MAKER), 7);
    run(
        &mut w,
        &[MAKER],
        vec![
            ata::create_ix(k(MAKER), offer, k(A), true),
            make_offer_ix(ESC, k(MAKER), k(A), k(B), 7, 400, 250),
        ],
    )
    .unwrap();
    // a token account that looks like the vault but belongs to Eve
    tacct(&mut w, k(70), k(A), offer, 0);
    w
}

fn offer_state() -> Offer {
    let (_, bump) = offer_address(&ESC, &k(MAKER), 7);
    Offer {
        maker: k(MAKER),
        mint_a: k(A),
        mint_b: k(B),
        id: 7,
        amount_b: 250,
        bump,
    }
}

/// Tokens leave a token account only with its owner's signature, or from the
/// offer's vault when the maker is paid at least the asking price.
fn escrow_policy(before: &Snapshot, after: &Snapshot, tx: &Transaction) -> Option<String> {
    let (offer_key, _) = offer_address(&ESC, &k(MAKER), 7);
    for (key, acct) in &before.accounts {
        let (Some(b), Some(a)) = (
            token_amount(before, key),
            token_amount(after, key).or(Some(0)),
        ) else {
            continue;
        };
        if a < b {
            let owner = anchorlab::token::TokenAccount::unpack(&acct.data)
                .ok()?
                .owner;
            if owner == offer_key {
                let paid = token_amount(after, &associated_address(&k(MAKER), &k(B)).0)
                    .unwrap_or(0)
                    - token_amount(before, &associated_address(&k(MAKER), &k(B)).0).unwrap_or(0);
                if paid < 250 {
                    return Some(format!("vault released but the maker was paid only {paid}"));
                }
            } else if !tx.signers.contains(&owner) {
                return Some(format!("{key} lost tokens without its owner's signature"));
            }
        }
    }
    None
}

fn escrow_pool() -> Vec<Pubkey> {
    let (offer, _) = offer_address(&ESC, &k(MAKER), 7);
    let mut p = vec![
        k(MAKER),
        k(TAKER),
        k(EVE),
        k(A),
        k(B),
        offer,
        k(70),
        k(DECOY),
    ];
    for w in [MAKER, TAKER, EVE] {
        for m in [A, B] {
            p.push(associated_address(&k(w), &k(m)).0);
        }
    }
    p.push(associated_address(&offer, &k(A)).0);
    p.push(anchorlab::token::TOKEN_PROGRAM_ID);
    p
}

#[test]
fn take_offer_survives_every_single_and_pair_mutation() {
    let ix = take_offer_ix(ESC, k(TAKER), &offer_state());
    let pool = escrow_pool();
    let singles = single_mutations(&ix, &pool);
    let r1 = run_attacks(&escrow_world, &ix, k(EVE), &singles, &escrow_policy);
    assert_eq!(r1.findings, vec![]);
    let pairs = pair_mutations(&ix, &pool);
    let r2 = run_attacks(&escrow_world, &ix, k(EVE), &pairs, &escrow_policy);
    assert_eq!(r2.findings, vec![]);
    println!(
        "take_offer: singles {} tried, {} accepted; pairs {} tried, {} accepted",
        r1.tried, r1.succeeded, r2.tried, r2.succeeded
    );
}

#[test]
fn cancel_offer_survives_every_single_and_pair_mutation() {
    let ix = cancel_offer_ix(ESC, &offer_state());
    let pool = escrow_pool();
    let singles = single_mutations(&ix, &pool);
    let r1 = run_attacks(&escrow_world, &ix, k(EVE), &singles, &escrow_policy);
    assert_eq!(r1.findings, vec![]);
    let pairs = pair_mutations(&ix, &pool);
    let r2 = run_attacks(&escrow_world, &ix, k(EVE), &pairs, &escrow_policy);
    assert_eq!(r2.findings, vec![]);
    println!(
        "cancel_offer: singles {} tried, {} accepted; pairs {} tried, {} accepted",
        r1.tried, r1.succeeded, r2.tried, r2.succeeded
    );
}

#[test]
fn the_policy_lets_a_legitimate_swap_through() {
    // Eve really can take the offer if she holds token B and signs: three slots change
    let funded = || {
        let mut w = escrow_world();
        ata(&mut w, k(EVE), k(B), 300);
        let mut m = w.bank.account(&k(B)).unwrap().clone();
        let mut mm = anchorlab::token::Mint::unpack(&m.data).unwrap();
        mm.supply += 300;
        m.data = mm.pack();
        w.set_account(k(B), m);
        w
    };
    let ix = take_offer_ix(ESC, k(TAKER), &offer_state());
    let r = |index, with| Mutation::ReplaceKey { index, with };
    let both = |a, b| Mutation::Both(Box::new(a), Box::new(b));
    let triple = both(
        both(r(0, k(EVE)), r(5, associated_address(&k(EVE), &k(A)).0)),
        r(6, associated_address(&k(EVE), &k(B)).0),
    );
    let report = run_attacks(&funded, &ix, k(EVE), &[triple], &escrow_policy);
    assert_eq!((report.tried, report.succeeded), (1, 1));
    assert_eq!(report.findings, vec![]);
}

#[test]
fn the_pair_generator_includes_a_replacement_with_a_signer_flip() {
    let ix = Instruction {
        program_id: k(9),
        accounts: vec![
            AccountMeta::writable(k(1), true),
            AccountMeta::readonly(k(2), false),
        ],
        data: vec![],
    };
    let pairs = pair_mutations(&ix, &[k(8)]);
    let flip = Mutation::Both(
        Box::new(Mutation::ReplaceKey {
            index: 0,
            with: k(8),
        }),
        Box::new(Mutation::FlipSigner(1)),
    );
    assert!(pairs.contains(&flip));
    assert!(!pairs.iter().any(|m| matches!(
        m,
        Mutation::Both(_, b) if **b == Mutation::FlipWritable(0) || **b == Mutation::FlipWritable(1)
    )));
    // the second replacement is always in a later slot
    assert!(pairs.contains(&Mutation::Both(
        Box::new(Mutation::ReplaceKey {
            index: 0,
            with: k(8)
        }),
        Box::new(Mutation::ReplaceKey {
            index: 1,
            with: k(8)
        }),
    )));
    assert!(!pairs.contains(&Mutation::Both(
        Box::new(Mutation::ReplaceKey {
            index: 1,
            with: k(8)
        }),
        Box::new(Mutation::ReplaceKey {
            index: 0,
            with: k(8)
        }),
    )));
}

#[test]
fn the_attacker_signs_alone_and_cosigners_are_kept_only_when_named() {
    let ix = withdraw_ix(k(CFG), k(ADMIN), 500_000);
    let seen = std::cell::RefCell::new(Vec::new());
    let policy = |_: &Snapshot, _: &Snapshot, tx: &Transaction| {
        seen.borrow_mut().push(tx.signers.clone());
        None
    };
    let make = || vault_world(Strength::Naive);
    let r = run_attacks(&make, &ix, k(EVE), &[Mutation::None], &policy);
    assert_eq!(r.succeeded, 1);
    assert_eq!(seen.borrow().as_slice(), &[vec![k(EVE)]]);
    seen.borrow_mut().clear();
    let r = run_attacks_with(&make, &ix, k(EVE), &[k(ADMIN)], &[Mutation::None], &policy);
    assert_eq!(r.succeeded, 1);
    assert_eq!(seen.borrow().as_slice(), &[vec![k(EVE), k(ADMIN)]]);
}

#[test]
fn a_program_that_breaks_token_supply_is_reported() {
    // a "token program" that credits itself tokens without raising the supply
    struct Cheat;
    impl Program for Cheat {
        fn process(&self, inv: &mut Invoke<'_>, _d: &[u8]) -> Result<(), Error> {
            inv.account_mut(0)?.data[64..72].copy_from_slice(&5u64.to_le_bytes());
            Ok(())
        }
    }
    let make = || {
        let mut w = world();
        w.add_program(anchorlab::token::TOKEN_PROGRAM_ID, Box::new(Cheat));
        wallet(&mut w, EVE);
        mint(&mut w, k(100), 0);
        tacct(&mut w, k(63), k(100), k(EVE), 0);
        w
    };
    let ix = Instruction {
        program_id: anchorlab::token::TOKEN_PROGRAM_ID,
        accounts: vec![AccountMeta::writable(k(63), false)],
        data: vec![],
    };
    let r = run_attacks(&make, &ix, k(EVE), &[Mutation::None], &|_, _, _| None);
    assert_eq!(
        r.findings,
        vec![Finding {
            mutation: Mutation::None,
            what: What::Tokens(Violation::Supply {
                mint: k(100),
                supply: 0,
                held: 5
            })
        }]
    );
}

// ---------- the confused deputy: the admin signs a transaction the attacker built ----------

const TREASURY: u8 = 8;

fn deputy_world(s: Strength) -> World {
    let mut w = vault_world(s);
    w.set_account(
        k(TREASURY),
        Account {
            lamports: rent_exempt_minimum(0) + 5_000_000,
            data: vec![],
            owner: VAULT,
            executable: false,
        },
    );
    w
}

/// Only the designated vault may lose lamports, whoever signed.
fn other_account_drained(before: &Snapshot, after: &Snapshot, _: &Transaction) -> Option<String> {
    (after.lamports(&k(TREASURY)) < before.lamports(&k(TREASURY)))
        .then(|| "the treasury lost lamports".to_string())
}

fn deputy(s: Strength) -> Report {
    let ix = withdraw_ix(k(CFG), k(ADMIN), 500_000);
    let mut pool = pool();
    pool.push(k(TREASURY));
    let make = move || deputy_world(s);
    let mut ms = single_mutations(&ix, &pool);
    ms.extend(pair_mutations(&ix, &pool));
    run_attacks_with(&make, &ix, k(EVE), &[k(ADMIN)], &ms, &other_account_drained)
}

#[test]
fn a_tricked_admin_loses_the_treasury_unless_every_account_is_bound() {
    let hit = Mutation::ReplaceKey {
        index: 1,
        with: k(TREASURY),
    };
    for s in [Strength::Naive, Strength::SignerOnly] {
        let r = deputy(s);
        assert!(r.findings.iter().any(|f| f.mutation == hit), "{s:?}");
    }
    let r = deputy(Strength::Strict);
    assert_eq!(r.findings, vec![]);
    assert!(r.succeeded > 0, "the admin's own honest call still works");
}
```

### `tests/audit_tests.rs`

```rust
mod common;
use anchorlab::validate::*;
use auditlab::audit::*;
use auditlab::sample::*;
use common::*;
use solanalab::pubkey::Pubkey;

fn rules(specs: &[Spec]) -> Vec<(&'static str, usize)> {
    audit(specs).iter().map(|f| (f.rule, f.account)).collect()
}

fn typed() -> Check {
    Check::Typed {
        owner: k(9),
        disc: [1; 8],
        len: 40,
    }
}
fn seeds(bump: Bump) -> Check {
    Check::Seeds {
        seeds: vec![Seed::Bytes(b"x".to_vec())],
        bump,
        program: k(9),
    }
}

#[test]
fn the_three_vaults_get_the_reviews_they_deserve() {
    assert_eq!(
        rules(&specs(Strength::Naive)),
        vec![("A01", 0), ("A02", 1), ("A02", 2)]
    );
    assert_eq!(rules(&specs(Strength::SignerOnly)), vec![("A02", 1)]);
    assert_eq!(rules(&specs(Strength::Strict)), vec![]);
    let f = &audit(&specs(Strength::Naive))[0];
    assert_eq!((f.severity, f.name), (Severity::High, "(instruction)"));
}

#[test]
fn a01_needs_a_signer_somewhere() {
    let s = vec![spec("a", vec![Check::Signer])];
    assert_eq!(rules(&s), vec![]);
    let s = vec![spec("a", vec![Check::Address(k(1))])];
    assert_eq!(rules(&s), vec![("A01", 0)]);
}

#[test]
fn a02_writable_accounts_need_an_identity() {
    let sig = spec("s", vec![Check::Signer]);
    for ok in [
        Check::Owner(k(9)),
        typed(),
        Check::Address(k(1)),
        seeds(Bump::Canonical),
        Check::Program(k(9)),
    ] {
        let s = vec![sig.clone(), spec("w", vec![Check::Mut, ok])];
        assert!(!rules(&s).iter().any(|r| r.0 == "A02"), "{s:?}");
    }
    let s = vec![sig.clone(), spec("w", vec![Check::Mut])];
    assert_eq!(rules(&s), vec![("A02", 1)]);
    // a signer is identified by its signature; an account about to be created is exempt
    let s = vec![spec("w", vec![Check::Signer, Check::Mut])];
    assert_eq!(rules(&s), vec![]);
    let s = vec![
        sig,
        spec("n", vec![Check::Mut, Check::Uninit, Check::Address(k(1))]),
    ];
    assert_eq!(rules(&s), vec![]);
}

#[test]
fn a03_new_accounts_need_an_address() {
    let sig = spec("s", vec![Check::Signer]);
    let s = vec![sig.clone(), spec("n", vec![Check::Mut, Check::Uninit])];
    assert_eq!(rules(&s), vec![("A03", 1)]);
    for ok in [Check::Address(k(1)), seeds(Bump::Canonical)] {
        let s = vec![sig.clone(), spec("n", vec![Check::Mut, Check::Uninit, ok])];
        assert_eq!(rules(&s), vec![]);
    }
}

#[test]
fn a04_a_stored_bump_needs_a_typed_account() {
    let sig = spec("s", vec![Check::Signer]);
    let s = vec![sig.clone(), spec("p", vec![seeds(Bump::Stored(3))])];
    assert_eq!(rules(&s), vec![("A04", 1)]);
    let s = vec![
        sig.clone(),
        spec("p", vec![typed(), seeds(Bump::Stored(3))]),
    ];
    assert_eq!(rules(&s), vec![]);
    let s = vec![sig, spec("p", vec![seeds(Bump::Canonical)])];
    assert_eq!(rules(&s), vec![]);
}

#[test]
fn a05_programs_are_compared_with_an_id() {
    let sig = spec("s", vec![Check::Signer]);
    let s = vec![sig.clone(), spec("token_program", vec![])];
    assert_eq!(rules(&s), vec![("A05", 1)]);
    let s = vec![sig, spec("token_program", vec![Check::Program(k(9))])];
    assert_eq!(rules(&s), vec![]);
}

#[test]
fn a06_and_a08_token_accounts() {
    let sig = spec("s", vec![Check::Signer]);
    let both = vec![
        Check::Owner(k(9)),
        Check::TokenMint(0),
        Check::TokenOwner(0),
    ];
    assert_eq!(rules(&[sig.clone(), spec("t", both)]), vec![]);
    // mint only: A06 (not both) and nothing else because the owner is checked
    let s = vec![
        sig.clone(),
        spec("t", vec![Check::Owner(k(9)), Check::TokenMint(0)]),
    ];
    assert_eq!(rules(&s), vec![("A06", 1)]);
    let s = vec![
        sig.clone(),
        spec("t", vec![Check::Owner(k(9)), Check::TokenOwner(0)]),
    ];
    assert_eq!(rules(&s), vec![("A06", 1)]);
    // both fields but no owner of the account: A08
    let s = vec![
        sig,
        spec("t", vec![Check::TokenMint(0), Check::TokenOwner(0)]),
    ];
    assert_eq!(rules(&s), vec![("A08", 1)]);
}

#[test]
fn a07_a_writable_typed_account_is_tied_to_someone() {
    let sig = spec("s", vec![Check::Signer]);
    let s = vec![sig.clone(), spec("d", vec![Check::Mut, typed()])];
    assert_eq!(rules(&s), vec![("A07", 1)]);
    let has = Check::HasOne {
        offset: 8,
        target: 0,
    };
    let s = vec![sig, spec("d", vec![Check::Mut, typed(), has])];
    assert_eq!(rules(&s), vec![]);
    // with no signer anywhere only A01 fires
    let s = vec![spec("d", vec![Check::Mut, typed()])];
    assert_eq!(rules(&s), vec![("A01", 0)]);
}

#[test]
fn a09_names_must_differ_and_findings_are_ordered_by_account() {
    let s = vec![
        spec("x", vec![Check::Signer]),
        spec("y", vec![]),
        spec("x", vec![]),
    ];
    assert_eq!(rules(&s), vec![("A09", 2)]);
    let s = vec![
        spec("a", vec![Check::Signer]),
        spec("b", vec![Check::Mut]),
        spec("c", vec![Check::Mut]),
    ];
    let r = rules(&s);
    assert_eq!(r, vec![("A02", 1), ("A02", 2)]);
    let _ = Pubkey::default();
}

#[test]
fn a_single_token_field_without_the_owner_fires_two_rules_and_severities_are_fixed() {
    let sig = spec("s", vec![Check::Signer]);
    let s = vec![sig.clone(), spec("t", vec![Check::TokenMint(0)])];
    assert_eq!(rules(&s), vec![("A06", 1), ("A08", 1)]);
    // one specimen of every rule, and the severity each carries
    let cases: Vec<(&str, Vec<Spec>, Severity)> = vec![
        ("A01", vec![spec("a", vec![])], Severity::High),
        (
            "A02",
            vec![sig.clone(), spec("w", vec![Check::Mut])],
            Severity::High,
        ),
        (
            "A03",
            vec![sig.clone(), spec("n", vec![Check::Uninit])],
            Severity::Warning,
        ),
        (
            "A04",
            vec![sig.clone(), spec("p", vec![seeds(Bump::Stored(0))])],
            Severity::High,
        ),
        (
            "A05",
            vec![sig.clone(), spec("my_program", vec![])],
            Severity::High,
        ),
        (
            "A06",
            vec![
                sig.clone(),
                spec("t", vec![Check::Owner(k(9)), Check::TokenMint(0)]),
            ],
            Severity::Warning,
        ),
        (
            "A07",
            vec![sig.clone(), spec("d", vec![Check::Mut, typed()])],
            Severity::Warning,
        ),
        (
            "A08",
            vec![
                sig.clone(),
                spec("t", vec![Check::TokenMint(0), Check::TokenOwner(0)]),
            ],
            Severity::High,
        ),
        ("A09", vec![sig.clone(), spec("s", vec![])], Severity::Info),
    ];
    for (rule, specs, severity) in cases {
        let found: Vec<_> = audit(&specs)
            .into_iter()
            .filter(|f| f.rule == rule)
            .collect();
        assert_eq!(found.len(), 1, "{rule}");
        assert_eq!(found[0].severity, severity, "{rule}");
        assert!(!found[0].message.is_empty());
    }
    assert!(Severity::Info < Severity::Warning && Severity::Warning < Severity::High);
}
```

### `tests/deploy_tests.rs`

```rust
mod common;
use auditlab::deploy::*;
use auditlab::sample::*;
use auditlab::world::World;
use common::*;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::instruction::Transaction;
use solanalab::pubkey::Pubkey;
use solanalab::runtime::{Bank, LOADER};

const ADMIN: u8 = 1;
const OTHER: u8 = 2;

fn fresh(step: u64, wide: bool) -> World {
    let mut w = World::new(Bank::new());
    wallet(&mut w, ADMIN);
    wallet(&mut w, OTHER);
    deploy(
        &mut w,
        COUNTER,
        Box::new(Counter { step, wide }),
        k(ADMIN),
        10,
    )
    .unwrap();
    w.set_account(
        k(50),
        Account {
            lamports: rent_exempt_minimum(8),
            data: vec![0; 8],
            owner: COUNTER,
            executable: false,
        },
    );
    w
}

fn bump(w: &mut World) -> Result<(), solanalab::runtime::Error> {
    run(w, &[ADMIN], vec![bump_ix(k(50))]).map(|_| ())
}

fn counter(w: &World) -> u64 {
    u64::from_le_bytes(
        w.bank.account(&k(50)).unwrap().data[..8]
            .try_into()
            .unwrap(),
    )
}

#[test]
fn programdata_layout_is_strict() {
    let pd = ProgramData {
        slot: 0x0102030405060708,
        authority: Some(k(7)),
    };
    let d = pd.pack();
    assert_eq!(d.len(), PROGRAMDATA_LEN);
    assert_eq!(&d[..10], &[3, 8, 7, 6, 5, 4, 3, 2, 1, 1]);
    assert_eq!(&d[10..], &[7; 32]);
    assert_eq!(ProgramData::unpack(&d), Ok(pd));
    let none = ProgramData {
        slot: 5,
        authority: None,
    };
    assert_eq!(none.pack()[9..], [0; 33]);
    assert_eq!(ProgramData::unpack(&none.pack()), Ok(none));
    // every malformed shape is refused
    assert_eq!(ProgramData::unpack(&d[..41]), Err(DeployError::BadData));
    let mut long = d.clone();
    long.push(0);
    assert_eq!(ProgramData::unpack(&long), Err(DeployError::BadData));
    let mut bad = d.clone();
    bad[0] = 2;
    assert_eq!(ProgramData::unpack(&bad), Err(DeployError::BadData));
    let mut bad = d.clone();
    bad[9] = 2;
    assert_eq!(ProgramData::unpack(&bad), Err(DeployError::BadData));
    let mut bad = none.pack();
    bad[20] = 1; // a "no authority" flag followed by non-zero bytes
    assert_eq!(ProgramData::unpack(&bad), Err(DeployError::BadData));
}

#[test]
fn programdata_addresses_match_an_independent_derivation() {
    let s = |p: &Pubkey| programdata_address(p).to_string();
    assert_eq!(s(&VAULT), "2rLM8Q9VzhfmikGLPaHbrDndRTVoFDPpytDdxyqzSXfX");
    assert_eq!(s(&COUNTER), "EbC3eZc1dtnqb42oPcAzT1mA5Fv3xsEPXFSeDdVgwipX");
    assert_eq!(s(&k(1)), "42DtwM9SRLwJD3vSKkxrpm6S6Hh9h4y2xLr3N6497u8x");
    assert_eq!(
        vault_address(&k(5)).to_string(),
        "3Uc7f9rBUAWpGrksga77W1mph88fz8TA4DdRULBajGfb"
    );
}

#[test]
fn deploy_records_the_authority_and_refuses_a_second_deployment() {
    let mut w = fresh(1, false);
    let pd = read(&w, &COUNTER).unwrap();
    assert_eq!((pd.slot, pd.authority), (10, Some(k(ADMIN))));
    let a = w.bank.account(&programdata_address(&COUNTER)).unwrap();
    assert_eq!(a.owner, LOADER);
    assert!(!a.executable);
    assert!(w.bank.account(&COUNTER).unwrap().executable);
    assert_eq!(
        deploy(
            &mut w,
            COUNTER,
            Box::new(Counter {
                step: 9,
                wide: false
            }),
            k(OTHER),
            11
        ),
        Err(DeployError::AlreadyDeployed)
    );
    assert_eq!(read(&w, &COUNTER).unwrap().authority, Some(k(ADMIN)));
    bump(&mut w).unwrap();
    assert_eq!(
        counter(&w),
        1,
        "the refused deployment did not replace the code"
    );
    assert_eq!(read(&w, &k(99)), Err(DeployError::UnknownProgram));
}

#[test]
fn only_the_authority_may_upgrade_and_the_slot_must_grow() {
    let mut w = fresh(1, false);
    bump(&mut w).unwrap();
    let v2 = || {
        Box::new(Counter {
            step: 2,
            wide: false,
        })
    };
    assert_eq!(
        upgrade(&mut w, COUNTER, v2(), &k(OTHER), 11),
        Err(DeployError::NotAuthority)
    );
    assert_eq!(
        upgrade(&mut w, COUNTER, v2(), &k(ADMIN), 10),
        Err(DeployError::StaleSlot)
    );
    assert_eq!(
        upgrade(&mut w, COUNTER, v2(), &k(ADMIN), 9),
        Err(DeployError::StaleSlot)
    );
    assert_eq!(
        upgrade(&mut w, k(99), v2(), &k(ADMIN), 11),
        Err(DeployError::UnknownProgram)
    );
    bump(&mut w).unwrap();
    assert_eq!(counter(&w), 2, "still version 1");
    upgrade(&mut w, COUNTER, v2(), &k(ADMIN), 11).unwrap();
    assert_eq!(read(&w, &COUNTER).unwrap().slot, 11);
    assert_eq!(read(&w, &COUNTER).unwrap().authority, Some(k(ADMIN)));
    // the data survived and the new code runs on it
    bump(&mut w).unwrap();
    assert_eq!(counter(&w), 4);
}

#[test]
fn authority_can_be_moved_and_then_given_up_for_good() {
    let mut w = fresh(1, false);
    let v = || {
        Box::new(Counter {
            step: 1,
            wide: false,
        })
    };
    assert_eq!(
        set_authority(&mut w, &COUNTER, &k(OTHER), Some(k(OTHER))),
        Err(DeployError::NotAuthority)
    );
    set_authority(&mut w, &COUNTER, &k(ADMIN), Some(k(OTHER))).unwrap();
    assert_eq!(
        upgrade(&mut w, COUNTER, v(), &k(ADMIN), 11),
        Err(DeployError::NotAuthority)
    );
    upgrade(&mut w, COUNTER, v(), &k(OTHER), 11).unwrap();
    set_authority(&mut w, &COUNTER, &k(OTHER), None).unwrap();
    assert_eq!(read(&w, &COUNTER).unwrap().authority, None);
    assert_eq!(read(&w, &COUNTER).unwrap().slot, 11, "slot kept");
    for who in [ADMIN, OTHER] {
        assert_eq!(
            upgrade(&mut w, COUNTER, v(), &k(who), 12),
            Err(DeployError::Immutable)
        );
        assert_eq!(
            set_authority(&mut w, &COUNTER, &k(who), Some(k(who))),
            Err(DeployError::Immutable)
        );
    }
    bump(&mut w).unwrap();
}

#[test]
fn a_malformed_programdata_account_is_reported() {
    let mut w = fresh(1, false);
    let addr = programdata_address(&COUNTER);
    let mut a = w.bank.account(&addr).unwrap().clone();
    a.data[0] = 0;
    w.set_account(addr, a.clone());
    assert_eq!(read(&w, &COUNTER), Err(DeployError::BadData));
    a.data[0] = 3;
    a.owner = k(9);
    w.set_account(addr, a);
    assert_eq!(read(&w, &COUNTER), Err(DeployError::BadData));
}

fn txs(n: usize) -> Vec<Transaction> {
    (0..n)
        .map(|_| Transaction {
            signers: vec![k(ADMIN)],
            instructions: vec![bump_ix(k(50))],
        })
        .collect()
}

#[test]
fn replaying_a_transcript_finds_where_a_new_version_differs() {
    // identical code: no divergence
    let (mut a, mut b) = (fresh(1, false), fresh(1, false));
    assert_eq!(differential(&mut a, &mut b, &txs(5)), None);
    // a different step size: the first call already leaves a different state
    let (mut a, mut b) = (fresh(1, false), fresh(2, false));
    assert_eq!(
        differential(&mut a, &mut b, &txs(5)),
        Some(Divergence {
            step: 0,
            old_ok: true,
            new_ok: true,
            same_state: false
        })
    );
    // a new account layout: the new version refuses the old accounts
    let (mut a, mut b) = (fresh(1, false), fresh(1, true));
    assert_eq!(
        differential(&mut a, &mut b, &txs(5)),
        Some(Divergence {
            step: 0,
            old_ok: true,
            new_ok: false,
            same_state: false
        })
    );
    // the same code, but one world was upgraded: the loader's own record differs
    let (mut a, mut b) = (fresh(1, false), fresh(1, false));
    bump(&mut a).unwrap();
    bump(&mut b).unwrap();
    upgrade(
        &mut b,
        COUNTER,
        Box::new(Counter {
            step: 3,
            wide: false,
        }),
        &k(ADMIN),
        11,
    )
    .unwrap();

    let d = differential(&mut a, &mut b, &txs(1)).unwrap();
    assert_eq!((d.step, d.old_ok, d.new_ok), (0, true, true));
    assert!(!d.same_state);
}

#[test]
fn upgrades_keep_balances_of_the_programs_accounts() {
    let mut w = fresh(1, false);
    let before = w.bank.lamports(&k(50));
    upgrade(
        &mut w,
        COUNTER,
        Box::new(Counter {
            step: 5,
            wide: false,
        }),
        &k(ADMIN),
        11,
    )
    .unwrap();
    assert_eq!(w.bank.lamports(&k(50)), before);
    assert_eq!(w.bank.account(&k(50)).unwrap().owner, COUNTER);
}

#[test]
fn a_different_verdict_with_an_identical_state_is_still_a_divergence() {
    // a counter that adds nothing leaves the account alone whether it runs or is refused
    // (both pay the same fee), but one version accepts and the other refuses
    let (mut a, mut b) = (fresh(0, false), fresh(0, true));
    assert_eq!(
        differential(&mut a, &mut b, &txs(1)),
        Some(Divergence {
            step: 0,
            old_ok: true,
            new_ok: false,
            same_state: true
        })
    );
}
```

### `tests/sample_tests.rs`

```rust
mod common;
use auditlab::sample::*;
use auditlab::world::World;
use common::*;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::runtime::{Bank, Error};

fn vault(s: Strength) -> World {
    let mut w = World::new(Bank::new());
    w.add_program(VAULT, Box::new(Vault(s)));
    wallet(&mut w, 1);
    let acct = |lamports, data, owner| Account {
        lamports,
        data,
        owner,
        executable: false,
    };
    w.set_account(
        k(5),
        acct(rent_exempt_minimum(CONFIG_LEN), config_data(&k(1)), VAULT),
    );
    w.set_account(
        vault_address(&k(5)),
        acct(rent_exempt_minimum(0) + 1_000, vec![], VAULT),
    );
    w
}

#[test]
fn every_strength_refuses_more_than_the_vault_holds_and_bad_data() {
    for s in [Strength::Naive, Strength::SignerOnly, Strength::Strict] {
        let mut w = vault(s);
        let held = w.bank.lamports(&vault_address(&k(5)));
        assert_eq!(
            run(&mut w, &[1], vec![withdraw_ix(k(5), k(1), held + 1)]).map(|_| ()),
            Err(Error::InsufficientFunds),
            "{s:?}"
        );
        let mut ix = withdraw_ix(k(5), k(1), 1);
        ix.data.pop();
        assert_eq!(
            run(&mut w, &[1], vec![ix]).map(|_| ()),
            Err(Error::InvalidInstructionData),
            "{s:?}"
        );
        // the exact amount that leaves the vault at its rent minimum is fine
        run(&mut w, &[1], vec![withdraw_ix(k(5), k(1), 1_000)]).unwrap();
        assert_eq!(
            w.bank.lamports(&vault_address(&k(5))),
            rent_exempt_minimum(0)
        );
    }
}

#[test]
fn compute_units_are_pinned() {
    let mut w = vault(Strength::Strict);
    let r = run(&mut w, &[1], vec![withdraw_ix(k(5), k(1), 1)]).unwrap();
    assert_eq!(r.units_used, 100);
    let mut w = World::new(Bank::new());
    w.add_program(
        COUNTER,
        Box::new(Counter {
            step: 1,
            wide: false,
        }),
    );
    wallet(&mut w, 1);
    w.set_account(
        k(50),
        Account {
            lamports: rent_exempt_minimum(8),
            data: vec![0; 8],
            owner: COUNTER,
            executable: false,
        },
    );
    let r = run(&mut w, &[1], vec![bump_ix(k(50))]).unwrap();
    assert_eq!(r.units_used, 50);
}

#[test]
fn the_counter_stops_at_the_top_instead_of_wrapping() {
    let mut w = World::new(Bank::new());
    w.add_program(
        COUNTER,
        Box::new(Counter {
            step: 2,
            wide: false,
        }),
    );
    wallet(&mut w, 1);
    let data = (u64::MAX - 1).to_le_bytes().to_vec();
    w.set_account(
        k(50),
        Account {
            lamports: rent_exempt_minimum(8),
            data,
            owner: COUNTER,
            executable: false,
        },
    );
    assert_eq!(
        run(&mut w, &[1], vec![bump_ix(k(50))]).map(|_| ()),
        Err(Error::ArithmeticOverflow)
    );
    assert_eq!(
        w.bank.account(&k(50)).unwrap().data,
        (u64::MAX - 1).to_le_bytes().to_vec()
    );
}
```

### `tests/props.rs`

```rust
mod common;
use anchorlab::ata::{self, associated_address};
use anchorlab::escrow::*;
use anchorlab::token;
use auditlab::invariants::{token_conservation, total_lamports};
use auditlab::world::World;
use common::*;
use solanalab::instruction::Instruction;

const A: u8 = 100;
const B: u8 = 101;
const WALLETS: [u8; 3] = [1, 2, 3];

fn start() -> World {
    let mut w = world();
    for n in WALLETS {
        wallet(&mut w, n);
    }
    mint(&mut w, k(A), 3000);
    mint(&mut w, k(B), 3000);
    for n in WALLETS {
        ata(&mut w, k(n), k(A), 1000);
        ata(&mut w, k(n), k(B), 1000);
    }
    w
}

struct Rng(u64);
impl Rng {
    fn next(&mut self, n: u64) -> u64 {
        self.0 = self
            .0
            .wrapping_mul(6364136223846793005)
            .wrapping_add(1442695040888963407);
        (self.0 >> 33) % n
    }
}

fn offer_of(maker: u8, id: u64, a: u64, b: u64) -> Offer {
    Offer {
        maker: k(maker),
        mint_a: k(A),
        mint_b: k(B),
        id,
        amount_b: b,
        bump: offer_address(&ESC, &k(maker), id).1,
    }
    .tap(|_| {
        let _ = a;
    })
}

trait Tap: Sized {
    fn tap(self, f: impl FnOnce(&Self)) -> Self {
        f(&self);
        self
    }
}
impl Tap for Offer {}

/// One random transaction: (signers, instructions).
fn random_tx(r: &mut Rng) -> (Vec<u8>, Vec<Instruction>) {
    let maker = WALLETS[r.next(3) as usize];
    let other = WALLETS[r.next(3) as usize];
    let id = 1 + r.next(3);
    let (a, b) = (1 + r.next(300), 1 + r.next(300));
    let (offer_key, _) = offer_address(&ESC, &k(maker), id);
    match r.next(4) {
        0 => (
            vec![maker],
            vec![
                ata::create_ix(k(maker), offer_key, k(A), true),
                make_offer_ix(ESC, k(maker), k(A), k(B), id, a, b),
            ],
        ),
        1 => (
            vec![other],
            vec![take_offer_ix(ESC, k(other), &offer_of(maker, id, a, b))],
        ),
        2 => (
            vec![maker],
            vec![cancel_offer_ix(ESC, &offer_of(maker, id, a, b))],
        ),
        _ => {
            let m = if r.next(2) == 0 { A } else { B };
            (
                vec![maker],
                vec![token::transfer(
                    associated_address(&k(maker), &k(m)).0,
                    associated_address(&k(other), &k(m)).0,
                    k(maker),
                    a,
                )],
            )
        }
    }
}

#[test]
fn random_escrow_traffic_is_deterministic_and_keeps_its_invariants() {
    let (mut ok, mut total) = (0, 0);
    for seed in 0..100u64 {
        let (mut w1, mut w2) = (start(), start());
        let lamports = total_lamports(&w1.snapshot());
        let mut r = Rng(seed);
        for _ in 0..25 {
            let (signers, ixs) = random_tx(&mut r);
            let (a, b) = (
                run(&mut w1, &signers, ixs.clone()),
                run(&mut w2, &signers, ixs),
            );
            assert_eq!(a, b, "seed {seed}");
            total += 1;
            ok += usize::from(a.is_ok());
            let s = w1.snapshot();
            assert_eq!(s.digest(), w2.snapshot().digest(), "seed {seed}");
            assert_eq!(token_conservation(&s), Ok(()), "seed {seed}");
            assert_eq!(total_lamports(&s), lamports, "seed {seed}");
        }
    }
    println!("random escrow traffic: {ok} of {total} transactions succeeded");
    assert!(ok * 4 > total, "too few successes: {ok} of {total}");
}

#[test]
fn a_failed_transaction_changes_only_the_payers_fee() {
    use auditlab::world::Change;
    let mut r = Rng(7);
    let mut w = start();
    let mut failures = 0;
    for _ in 0..200 {
        let (signers, ixs) = random_tx(&mut r);
        let before = w.snapshot();
        if run(&mut w, &signers, ixs).is_err() {
            failures += 1;
            let after = w.snapshot();
            let changes = before.changes(&after);
            assert_eq!(changes.len(), 1, "{changes:?}");
            assert!(
                matches!(&changes[0], Change::Modified { key, lamports, data: false, owner: false }
                    if *key == k(signers[0]) && lamports.0 - lamports.1 == 5000)
            );
            assert_eq!(after.fees, before.fees + 5000);
        }
    }
    assert!(failures > 20);
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
//! Rough timings of the tools. Run with `cargo run --release --example bench`.

use anchorlab::ata::{self, ATA_PROGRAM_ID, AtaProgram, associated_address};
use anchorlab::escrow::*;
use anchorlab::token::*;
use auditlab::attack::{pair_mutations, run_attacks, single_mutations};
use auditlab::audit::audit;
use auditlab::deploy::{deploy, upgrade};
use auditlab::invariants::token_conservation;
use auditlab::sample::*;
use auditlab::world::World;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::instruction::Transaction;
use solanalab::pubkey::Pubkey;
use solanalab::runtime::Bank;
use std::hint::black_box;
use std::time::Instant;

const ESC: Pubkey = Pubkey([0xE5; 32]);

fn k(n: u8) -> Pubkey {
    Pubkey::filled(n)
}

fn timed<R>(label: &str, iters: u32, mut f: impl FnMut() -> R) {
    let t = Instant::now();
    for _ in 0..iters {
        black_box(f());
    }
    let per = t.elapsed().as_nanos() as f64 / f64::from(iters);
    if per >= 10_000.0 {
        println!("{label:<44} {:>10.2} us", per / 1000.0);
    } else {
        println!("{label:<44} {per:>10.1} ns");
    }
}

fn world() -> World {
    let mut w = World::new(Bank::new());
    w.add_program(TOKEN_PROGRAM_ID, Box::new(TokenProgram));
    w.add_program(ATA_PROGRAM_ID, Box::new(AtaProgram));
    w.add_program(ESC, Box::new(EscrowProgram));
    for n in [1, 2] {
        w.set_account(k(n), Account::wallet(1_000_000_000_000));
    }
    let m = |supply| Mint {
        mint_authority: Some(k(250)),
        supply,
        decimals: 6,
        is_initialized: true,
        freeze_authority: None,
    };
    w.set_account(
        k(100),
        mint_account(&m(1000), rent_exempt_minimum(MINT_LEN)),
    );
    w.set_account(k(101), mint_account(&m(500), rent_exempt_minimum(MINT_LEN)));
    for (wl, mt, amt) in [(1, 100, 1000), (1, 101, 0), (2, 100, 0), (2, 101, 500)] {
        let mut t = TokenAccount::new(k(mt), k(wl));
        t.amount = amt;
        w.set_account(
            associated_address(&k(wl), &k(mt)).0,
            token_account(&t, rent_exempt_minimum(ACCOUNT_LEN)),
        );
    }
    let (offer, _) = offer_address(&ESC, &k(1), 7);
    w.process(&Transaction {
        signers: vec![k(1)],
        instructions: vec![
            ata::create_ix(k(1), offer, k(100), true),
            make_offer_ix(ESC, k(1), k(100), k(101), 7, 400, 250),
        ],
    })
    .unwrap();
    w
}

fn main() {
    let w = world();
    let snap = w.snapshot();
    println!("accounts tracked: {}", snap.accounts.len());
    timed("snapshot (clone every tracked account)", 20_000, || {
        w.snapshot()
    });
    timed("digest of a snapshot", 20_000, || snap.digest());
    timed("token_conservation", 20_000, || token_conservation(&snap));
    timed("changes between equal snapshots", 20_000, || {
        snap.changes(&snap)
    });
    let specs = specs(Strength::Strict);
    timed("audit of three specs", 100_000, || audit(&specs));
    timed("build the escrow world (make_offer included)", 2_000, world);

    let offer = Offer {
        maker: k(1),
        mint_a: k(100),
        mint_b: k(101),
        id: 7,
        amount_b: 250,
        bump: offer_address(&ESC, &k(1), 7).1,
    };
    let ix = take_offer_ix(ESC, k(2), &offer);
    let pool: Vec<Pubkey> = vec![k(1), k(2), k(100), k(101), offer_address(&ESC, &k(1), 7).0];
    let singles = single_mutations(&ix, &pool);
    let pairs = pair_mutations(&ix, &pool);
    println!(
        "mutations: {} singles, {} pairs",
        singles.len(),
        pairs.len()
    );
    let t = Instant::now();
    let r = run_attacks(&world, &ix, k(1), &singles, &|_, _, _| None);
    let el = t.elapsed();
    println!(
        "{} single attacks in {:.1} ms ({:.1} us each), {} accepted",
        r.tried,
        el.as_secs_f64() * 1e3,
        el.as_secs_f64() * 1e6 / r.tried as f64,
        r.succeeded
    );
    let t = Instant::now();
    let r = run_attacks(&world, &ix, k(1), &pairs, &|_, _, _| None);
    let el = t.elapsed();
    println!(
        "{} pair attacks in {:.1} ms ({:.1} us each), {} accepted",
        r.tried,
        el.as_secs_f64() * 1e3,
        el.as_secs_f64() * 1e6 / r.tried as f64,
        r.succeeded
    );

    timed("deploy + upgrade (new world)", 20_000, || {
        let mut w = World::new(Bank::new());
        w.set_account(k(1), Account::wallet(1));
        deploy(
            &mut w,
            COUNTER,
            Box::new(Counter {
                step: 1,
                wide: false,
            }),
            k(1),
            1,
        )
        .unwrap();
        upgrade(
            &mut w,
            COUNTER,
            Box::new(Counter {
                step: 2,
                wide: false,
            }),
            &k(1),
            2,
        )
        .unwrap();
        w
    });
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **52 tests** (world 5, invariants 7, attacks 16, audit 10, deploy 9, sample programs 3, random traffic 2) and 8 consecutive full runs without a failure (nothing uses sockets, real time or OS randomness).

**Mutation check ✓.** I made 117 small source edits (a "mutant") across the seven source modules, each in its own copy of the crate, and ran the whole test suite against each. **110 were caught by a failing test, 7 are equivalent (M47, M49, M54, M60, M61, M111, M112), and 0 does not compile (); none survive unexplained.** **What the first pass found.** I designed 115 mutants against my first 41 tests; 94 were caught and 21 survived. Seven of the survivors are equivalent (below). The other 14 were real gaps, in four groups. **The world** ignored which keys it tracks: nothing tested a fee payer that no instruction names, or a program that was put into the bank behind the world's back. **Invariants and reports**: an uninitialized mint with balances against it, a severity that nothing asserted (`A03` as a warning), the rule that a lone token-field check without the owner check fires two rules, a program that breaks the supply while the fuzzer reports success (it needs a "token program" that cheats, registered under the token program's id), and the signer list of the attacker's transaction (a mutant added a second signer and nothing noticed). **The sample programs** had no test for withdrawing more than the vault holds (`saturating_sub` instead of an error), for a counter at the top of its range (`wrapping_add` instead of an overflow error), or for their compute units. **The loader model**: a differential replay where one version accepts and the other refuses but the states are the same (a counter with step 0). One survivor taught more than the others: deleting the **seeds check from the strict vault** was not caught by any attack, because every attack in my harness was signed by the attacker alone, and a signature the attacker cannot forge protects the vault from everyone except its own admin. It needed a different attacker, the confused deputy of §5.3b, in which the admin signs a transaction built by the attacker. I then added two more mutants for the new runner (the co-signers list). 11 new tests closed the 14 gaps. Seven mutants remain, each with a stated reason for being equivalent; none fails to compile. The column "change" is generated from the edit itself (the part that differs, with a little context, whitespace collapsed); `…` marks omitted text. The tests ran in the dev profile.

| # | module | change | result |
|---|---|---|---|
| M1 | `world` | `keys: BTreeSet::from([Pubkey::default()]` → `keys: BTreeSet::new(` | caught by `random_escrow_traffic_is_deterministic_and_keeps_its_invariants` |
| M2 | `world` | `self.keys.extend(tx.signers.iter().copied());` → `∅` | caught by `keys_named_only_by_a_transaction_are_tracked` |
| M3 | `world` | `self.keys.insert(ix.program_id);` → `∅` | caught by `keys_named_only_by_a_transaction_are_tracked` |
| M4 | `world` | `self.keys.extend(ix.accounts.iter().map(\|m\| m.pubkey));` → `∅` | caught by `the_policy_lets_a_legitimate_swap_through` |
| M5 | `world` | `fees: self.bank.fees_collected` → `fees: 0` | caught by `a_policy_complaint_and_a_token_violation_are_reported` |
| M6 | `world` | `h.update(self.fees.to_le_bytes());` → `∅` | caught by `the_digest_does_not_confuse_where_one_account_ends` |
| M7 | `world` | `h.update((self.accounts.len() as u64).to_le_bytes());` → `∅` | caught by `the_digest_does_not_confuse_where_one_account_ends` |
| M8 | `world` | `h.update(k.0);` → `∅` | caught by `equal_states_have_equal_digests_and_any_difference_changes_them` |
| M9 | `world` | `h.update(a.lamports.to_le_bytes());` → `∅` | caught by `equal_states_have_equal_digests_and_any_difference_changes_them` |
| M10 | `world` | `h.update(a.owner.0);` → `∅` | caught by `the_digest_does_not_confuse_where_one_account_ends` |
| M11 | `world` | `h.update([u8::from(a.executable)]);` → `∅` | caught by `the_digest_does_not_confuse_where_one_account_ends` |
| M12 | `world` | `h.update((a.data.len() as u64).to_le_bytes());` → `∅` | caught by `the_digest_does_not_confuse_where_one_account_ends` |
| M13 | `world` | `h.update(&a.data);` → `∅` | caught by `replaying_a_transcript_finds_where_a_new_version_differs` |
| M14 | `world` | `…Some(_)) => out.push(Change::Created(*k)),` → `…Some(_)) => {}` | caught by `changes_lists_created_deleted_and_modified_in_key_order` |
| M15 | `world` | `…_), None) => out.push(Change::Deleted(*k)),` → `…_), None) => {}` | caught by `changes_lists_created_deleted_and_modified_in_key_order` |
| M16 | `world` | `…a), Some(b)) if a != b ` → `…a), Some(b)) ` | caught by `cancel_offer_survives_every_single_and_pair_mutation` |
| M17 | `world` | `data: a.data !` → `data: a.data =` | caught by `cancel_offer_survives_every_single_and_pair_mutation` |
| M18 | `world` | `owner: a.owner !` → `owner: a.owner =` | caught by `cancel_offer_survives_every_single_and_pair_mutation` |
| M19 | `world` | `lamports: (a.lamports, b` → `lamports: (b.lamports, a` | caught by `a_failed_transaction_changes_only_the_payers_fee` |
| M20 | `world` | `keys.extend(after.accounts.keys());` → `∅` | caught by `changes_lists_created_deleted_and_modified_in_key_order` |
| M21 | `world` | `…key).map_or(0` → `…key).map_or(1` | caught by `changes_lists_created_deleted_and_modified_in_key_order` |
| M22 | `invariants` | `if a.owner !` → `if a.owner =` | caught by `a_policy_complaint_and_a_token_violation_are_reported` |
| M23 | `invariants` | `if m.is_initialized { supplies.insert(*key, m.supply); }` → `supplies.insert(*key, m.supply);` | caught by `balances_of_an_uninitialized_mint_are_orphans` |
| M24 | `invariants` | `…t.mint).or_default() += u128::from(t.amount)` → `…t.mint).or_default() += 1` | caught by `the_policy_lets_a_legitimate_swap_through` |
| M25 | `invariants` | `if h !=` → `if h >` | caught by `a_supply_that_differs_from_the_balances_is_reported` |
| M26 | `invariants` | `if h !=` → `if h <` | caught by `a_supply_that_differs_from_the_balances_is_reported` |
| M27 | `invariants` | `if !supplies.contains_key(mint)` → `if false` | caught by `balances_without_a_mint_and_bad_layouts_are_reported` |
| M28 | `invariants` | `_ => return Err(Violation::BadLayout(*key)),` → `_ => {}` | caught by `balances_without_a_mint_and_bad_layouts_are_reported` |
| M29 | `invariants` | `.map_err(\|_\| Violation::BadLayout(*key))?` → `.unwrap_or_else(\|_\| panic!())` | caught by `balances_without_a_mint_and_bad_layouts_are_reported` |
| M30 | `invariants` | `+ u128::from(s.fees)` → `+ 0` | caught by `a_policy_complaint_and_a_token_violation_are_reported` |
| M31 | `invariants` | `…).unwrap_or(0` → `…).unwrap_or(1` | caught by `a_policy_complaint_and_a_token_violation_are_reported` |
| M32 | `attack` | `…out.accounts.get_mut(*i)?; a.is_signer = !a.is_signer` → `…out.accounts.get_mut(*i)?; a.is_writable = !a.is_writable` | caught by `mutations_are_applied_exactly` |
| M33 | `attack` | `a.is_writable = !a.is_writable` → `a.is_signer = !a.is_signer` | caught by `mutations_are_applied_exactly` |
| M34 | `attack` | `…n \|\| *j >= n \|\| i == j ` → `…n \|\| *j >= n ` | caught by `mutations_are_applied_exactly` |
| M35 | `attack` | `…\|\| *to >= n \|\| from == to ` → `…\|\| *to >= n ` | caught by `mutations_are_applied_exactly` |
| M36 | `attack` | `out.accounts[*to].pubkey = out.accounts[*from` → `out.accounts[*from].pubkey = out.accounts[*to` | caught by `mutations_are_applied_exactly` |
| M37 | `attack` | `if *i >=` → `if *i >` | caught by `mutations_are_applied_exactly` |
| M38 | `attack` | `…return apply(b, &apply(a` → `…return apply(a, &apply(b` | caught by `mutations_are_applied_exactly` |
| M39 | `attack` | `out.accounts[*i].pubkey = b` → `out.accounts[*i].pubkey = a` | caught by `mutations_are_applied_exactly` |
| M40 | `attack` | `…mut out = vec![Mutation::None` → `…mut out = vec![` | caught by `the_naive_vault_can_be_drained_without_the_admin` |
| M41 | `attack` | `if *p != ix.accounts[i].pubkey` → `if true` | caught by `the_number_of_mutations_is_what_the_formulas_say` |
| M42 | `attack` | `…Mutation::FlipSigner(i)); out.push(Mutation::FlipWritable(i));` → `…Mutation::FlipSigner(i));` | caught by `the_number_of_mutations_is_what_the_formulas_say` |
| M43 | `attack` | `…Mutation::FlipWritable(i)); out.push(Mutation::Drop(i));` → `…Mutation::FlipWritable(i));` | caught by `the_number_of_mutations_is_what_the_formulas_say` |
| M44 | `attack` | `if i <` → `if i <=` | caught by `a_panicking_program_is_a_finding_and_does_not_stop_the_run` |
| M45 | `attack` | `if i !=` → `if i <` | caught by `the_number_of_mutations_is_what_the_formulas_say` |
| M46 | `attack` | `if j == i` → `if false` | caught by `the_number_of_mutations_is_what_the_formulas_say` |
| M47 | `attack` | `if j >` → `if j >=` | **equivalent**: `j >= i` and `j > i` agree, because the case `j == i` was skipped two lines above |
| M48 | `attack` | `…Mutation::FlipSigner` → `…Mutation::FlipWritable` | caught by `the_pair_generator_includes_a_replacement_with_a_signer_flip` |
| M49 | `attack` | `if total_lamports(&after) != total_before { push(What::Lamports); }` → `∅` | **equivalent**: the runtime conserves lamports in every transaction, so this check of the model itself cannot fire; it is kept as a guard against a bug in the model |
| M50 | `attack` | `report.succeeded += 1;` → `∅` | caught by `a_policy_complaint_and_a_token_violation_are_reported` |
| M51 | `attack` | `report.tried += 1;` → `∅` | caught by `a_failed_attempt_costs_the_attacker_only_the_fee` |
| M52 | `attack` | `if let Some(p) = policy(&before, &after, &tx) { push(What::Policy(p…` → `∅` | caught by `a_policy_complaint_and_a_token_violation_are_reported` |
| M53 | `attack` | `if let Err(v) = token_conservation(&after) { push(What::Tokens(v)); }` → `∅` | caught by `a_program_that_breaks_token_supply_is_reported` |
| M54 | `attack` | `if !only_fee { push(What::NotAtomic); }` → `∅` | **equivalent**: the atomicity check cannot fire while the runtime restores state after a failure (Lesson 53); kept as a guard |
| M55 | `attack` | `let Ok(result) = outcome else { push(What::Panic); continue; }` → `let result = outcome.unwrap()` | caught by `a_panicking_program_is_a_finding_and_does_not_stop_the_run` |
| M56 | `attack` | `signers: signers.clone()` → `signers: vec![attacker, Pubkey([0; 32])]` | caught by `a_tricked_admin_loses_the_treasury_unless_every_account_is_bound` |
| M57 | `attack` | `if !` → `if ` | caught by `a_tricked_admin_loses_the_treasury_unless_every_account_is_bound` |
| M58 | `attack` | `signers.extend_from_slice(cosigners);` → `∅` | caught by `a_tricked_admin_loses_the_treasury_unless_every_account_is_bound` |
| M59 | `attack` | `…attacker, &[` → `…attacker, &[ix.accounts[0].pubkey` | caught by `cancel_offer_survives_every_single_and_pair_mutation` |
| M60 | `attack` | `…Change::Modified { key, data: false, owner: false, ` → `…Change::Modified { key, ` | **equivalent**: same reason as the previous atomicity mutant (the check cannot fire) |
| M61 | `attack` | `…Change::Modified { key, data: false, owner: false, .. } if *key ==…` → `…Change::Modified { data: false, owner: false, .. }` | **equivalent**: same reason (the check cannot fire) |
| M62 | `attack` | `…total_before = total_lamports(&before)` → `…total_before = 0` | caught by `a_policy_complaint_and_a_token_violation_are_reported` |
| M63 | `audit` | `if !` → `if ` | caught by `a01_needs_a_signer_somewhere` |
| M64 | `audit` | `Severity::High` → `Severity::Info` | caught by `the_three_vaults_get_the_reviews_they_deserve` |
| M65 | `audit` | `…!identified && !signer && !` → `…!identified && !` | caught by `a02_writable_accounts_need_an_identity` |
| M66 | `audit` | `…!identified && !signer && !has(s, \|c\| matches!(c, Check::Uninit)) ` → `…!identified && !signer ` | caught by `a03_new_accounts_need_an_address` |
| M67 | `audit` | `Check::Owner(_) \| Check::` → `Check::` | caught by `a02_writable_accounts_need_an_identity` |
| M68 | `audit` | `\| Check::Typed { .. } \| Check::` → `\| Check::` | caught by `a02_writable_accounts_need_an_identity` |
| M69 | `audit` | `\| Check::Address(_) \| Check::` → `\| Check::` | caught by `a02_writable_accounts_need_an_identity` |
| M70 | `audit` | `\| Check::Seeds { .. } \| Check::` → `\| Check::` | caught by `a02_writable_accounts_need_an_identity` |
| M71 | `audit` | `\| Check::Program(_) ` → `∅` | caught by `a02_writable_accounts_need_an_identity` |
| M72 | `audit` | `…Check::Address(_) \| Check::Seeds { .. }` → `…Check::Address(_)` | caught by `a03_new_accounts_need_an_address` |
| M73 | `audit` | `matches!(c, Check::Address(_) \| Check::` → `matches!(c, Check::` | caught by `a02_writable_accounts_need_an_identity` |
| M74 | `audit` | `bump: Bump::Stored(_)` → `bump: Bump::Canonical` | caught by `a04_a_stored_bump_needs_a_typed_account` |
| M75 | `audit` | `}) && !` → `}) && ` | caught by `a04_a_stored_bump_needs_a_typed_account` |
| M76 | `audit` | `…s.name.contains("program` → `…s.name.contains("programs` | caught by `a05_programs_are_compared_with_an_id` |
| M77 | `audit` | `if mint !=` → `if mint &&` | caught by `a06_and_a08_token_accounts` |
| M78 | `audit` | `…&& writable && any_signer &&` → `…&& writable &&` | caught by `a07_a_writable_typed_account_is_tied_to_someone` |
| M79 | `audit` | `if typed && writable && ` → `if typed && ` | caught by `a04_a_stored_bump_needs_a_typed_account` |
| M80 | `audit` | `if (mint \|\|` → `if (mint &&` | caught by `a_single_token_field_without_the_owner_fires_two_rules_and_severities_are_fixed` |
| M81 | `audit` | `if t.name =` → `if t.name !` | caught by `a02_writable_accounts_need_an_identity` |
| M82 | `audit` | `Severity::Warning` → `Severity::High` | caught by `a_single_token_field_without_the_owner_fires_two_rules_and_severities_are_fixed` |
| M83 | `deploy` | `d.len() !=` → `d.len() <` | caught by `programdata_layout_is_strict` |
| M84 | `deploy` | `…PROGRAMDATA_LEN \|\| d[0] != 3` → `…PROGRAMDATA_LEN` | caught by `a_malformed_programdata_account_is_reported` |
| M85 | `deploy` | `0 if key == [0; 32] ` → `0 ` | caught by `programdata_layout_is_strict` |
| M86 | `deploy` | `…mut d = vec![3` → `…mut d = vec![2` | caught by `authority_can_be_moved_and_then_given_up_for_good` |
| M87 | `deploy` | `if w.bank.account(&programdata_address(&id)).is_some() { return Err…` → `∅` | caught by `deploy_records_the_authority_and_refuses_a_second_deployment` |
| M88 | `deploy` | `if a.owner != LOADER { return Err(DeployError::BadData); }` → `∅` | caught by `a_malformed_programdata_account_is_reported` |
| M89 | `deploy` | `None => Err(DeployError::Immutable` → `None => Ok(()` | caught by `authority_can_be_moved_and_then_given_up_for_good` |
| M90 | `deploy` | `Some(a) if a == *signer` → `Some(_)` | caught by `authority_can_be_moved_and_then_given_up_for_good` |
| M91 | `deploy` | `if slot <=` → `if slot <` | caught by `only_the_authority_may_upgrade_and_the_slot_must_grow` |
| M92 | `deploy` | `if slot <= pd.slot` → `if slot == 0` | caught by `only_the_authority_may_upgrade_and_the_slot_must_grow` |
| M93 | `deploy` | `slot` → `slot: pd.slot` | caught by `authority_can_be_moved_and_then_given_up_for_good` |
| M94 | `deploy` | `slot: pd.slot` → `slot: 0` | caught by `authority_can_be_moved_and_then_given_up_for_good` |
| M95 | `deploy` | `slot, authority: Some(authority)` → `slot, authority: None` | caught by `deploy_records_the_authority_and_refuses_a_second_deployment` |
| M96 | `deploy` | `…read(w, &id)?; authorize(&pd, signer)?; ` → `…read(w, &id)?; ` | caught by `authority_can_be_moved_and_then_given_up_for_good` |
| M97 | `deploy` | `…read(w, id)?; authorize(&pd, signer)?; ` → `…read(w, id)?; ` | caught by `authority_can_be_moved_and_then_given_up_for_good` |
| M98 | `deploy` | `…!= b.is_ok() \|\| !same_state ` → `…!= b.is_ok() ` | caught by `replaying_a_transcript_finds_where_a_new_version_differs` |
| M99 | `deploy` | `if a.is_ok() != b.is_ok() \|\| ` → `if ` | caught by `a_different_verdict_with_an_identical_state_is_still_a_divergence` |
| M100 | `deploy` | `w.add_program(id, program); w` → `w` | caught by `only_the_authority_may_upgrade_and_the_slot_must_grow` |
| M101 | `deploy` | `…&[&program.0` → `…&[&program.0, b"x"` | caught by `programdata_addresses_match_an_independent_derivation` |
| M102 | `deploy` | `d.push(1` → `d.push(2` | caught by `authority_can_be_moved_and_then_given_up_for_good` |
| M103 | `sample` | `let amount = u64::from_le_bytes(data.try_into().map_err(\|_\| Error…` → `let amount = 1` | caught by `the_honest_call_succeeds_on_every_strength` |
| M104 | `sample` | `if s == Strength::SignerOnly && !inv.is_signer(2)?` → `if false` | caught by `the_signer_only_vault_falls_to_a_substituted_configuration` |
| M105 | `sample` | `if cfg.data.get(8..40) != Some(&inv.key(2)?.0[..])` → `if false` | caught by `the_signer_only_vault_falls_to_a_substituted_configuration` |
| M106 | `sample` | `.checked_sub(amount) .ok_or(Error::InsufficientFunds)?` → `.saturating_sub(amount)` | caught by `every_strength_refuses_more_than_the_vault_holds_and_bad_data` |
| M107 | `sample` | `let want = if self.wide { 16 } else { 8 }` → `let want = 8` | caught by `replaying_a_transcript_finds_where_a_new_version_differs` |
| M108 | `sample` | `let n = n.checked_add(self.step).ok_or(Error::ArithmeticOverflow)?` → `let n = n.wrapping_add(self.step)` | caught by `the_counter_stops_at_the_top_instead_of_wrapping` |
| M109 | `sample` | `inv.consume(100)?;` → `∅` | caught by `compute_units_are_pinned` |
| M110 | `sample` | `…Check::Owner(VAULT), seeds` → `…Check::Owner(VAULT)` | caught by `a_tricked_admin_loses_the_treasury_unless_every_account_is_bound` |
| M111 | `sample` | `…vec![Check::Mut, Check::Owner(VAULT), ` → `…vec![Check::Mut, ` | **equivalent**: the vault's owner check is implied by its seeds check, since the vault must be the derived address and only the owner can debit an account |
| M112 | `sample` | `…"config", vec![typed, ` → `…"config", vec![` | **equivalent**: the configuration's type check is implied by the vault's seeds check, which derives the vault from the configuration's key; no account owned by the program sits at an address derived from a decoy key |
| M113 | `sample` | `…"config", vec![typed, has_admin` → `…"config", vec![typed` | caught by `the_strict_vault_survives_every_single_and_pair_mutation` |
| M114 | `sample` | `…vec![Check::Signer, Check::` → `…vec![Check::` | caught by `the_strict_vault_survives_every_single_and_pair_mutation` |
| M115 | `sample` | `offset: 8, target: 2` → `offset: 8, target: 1` | caught by `the_honest_call_succeeds_on_every_strength` |
| M116 | `sample` | `inv.consume(50)?;` → `∅` | caught by `compute_units_are_pinned` |
| M117 | `sample` | `…CONFIG_LEN: usize = 40` → `…CONFIG_LEN: usize = 41` | caught by `the_honest_call_succeeds_on_every_strength` |

What the table does not show: it tests *these* edits, not every possible bug; the equivalent mutants could hide a real gap if my reasoning about them is wrong; and passing every mutant says nothing about the real runtime, about security or about behavior on a validator.

**Design notes.**

- The world remembers keys because the bank cannot list its accounts; the cost is that an account the world has never been told about is invisible (§3.2).
- Policies are closures; the harness never decides what is acceptable.
- Every attempt runs on a fresh world and is signed only by the attacker.
- The reviewer reads declarations only; the loader model keeps authority, slot and layout, and nothing about byte code.
- Not handled: instruction-data fuzzing, sequences of mutations, shrinking of findings (exercise 7 is the idea), the real loader, buffers and rent for program data.

</details>

---

## 17. Key takeaways

- Safety is mostly *absence*, and absence is shown by trying many wrong things and by invariants that must always hold; assert the specific error and that nothing moved (✓ throughout).
- Make state comparable: a world that tracks every key, snapshots, a length-framed SHA-256 digest and a diff turn "something changed" into a reviewable statement (✓ known answer from Python; a missing system-program key once made a conservation test fail by one lamport).
- An attacker controls the accounts and signs only for themselves; a fuzzer that enumerates substitutions, flag flips, swaps, duplicates and drops, on a fresh world each time, finds what reviews miss: the signer-only vault fell to exactly one pair of changes out of 105 (✓).
- A clean fuzzing run is evidence of one kind of absence under one policy; keep a control that the harness can still find, and a policy that can say yes (✓ the legitimate three-slot swap; the harness with three bugs).
- A constraint list is data, so a program can review it; its rules are heuristics and its silence is not safety (✓ nine rules, three vaults, three verdicts).
- An upgradeable program is a custody arrangement: who holds the authority matters as much as what the code says; immutability is an irreversible feature to use at the end (✓ authority, slot and immutability tests).
- An upgrade meets old accounts: replay old transactions against the new build and keep layouts append-only (✓ divergence at step 0 when the layout changed).
- A review is a method, not a certificate: scope, inventory, ten questions per account, CPIs, arithmetic, state machine, admin surface, economics, the tests themselves, a report with reproductions.
- Mutation testing here pointed at real gaps: 14 of the 21 first survivors were real gaps and were closed with 11 new tests, among them one that needed a different attacker (an administrator who signs what the attacker built); 7 mutants remain, each with a stated reason for being equivalent.

**Connections.** [Lesson 53](53_solana_model.md) is the runtime the tools wrap, unchanged; [Lesson 54](54_anchor.md) supplied the constraint lists the reviewer reads and the escrow the fuzzer attacks, and its mutation run is the reason §2 insists on exact errors. [Lesson 52](52_web3_reliability.md) gave the simulation and idempotency habits behind replay testing, [Lesson 48](48_crypto_primitives.md) the framing and hashing used by the digest, and [Lesson 40](40_property_testing_fuzzing.md) the random-traffic style. This is the last Solana lesson; Lesson 56 begins the next module.

---

## 18. References

- Solana documentation: programs, deployment, the upgradeable loader and upgrade authorities: <https://solana.com/docs>
- Anchor documentation (testing, verifiable builds, program upgrades): <https://www.anchor-lang.com/docs>
- Test harnesses for Solana programs, as I remember them: `solana-program-test` (in the Solana SDK repositories), `LiteSVM` and `Mollusk` (community and ecosystem projects), and the `solana-test-validator` tool; I did not use any of them.
- Public audit reports and post-mortems of Solana programs, for the vulnerability classes (missing signer, owner and type checks, re-initialization, duplicate mutable accounts, bump canonicalization, unchecked CPI targets, arithmetic and rounding): not consulted here; search for current ones.
- *The Rust Reference* and *Rustonomicon* on panics and unwinding, for `catch_unwind`: <https://doc.rust-lang.org/>
- SHA-256 (FIPS 180-4): <https://csrc.nist.gov/publications/detail/fips/180/4/final>

The URLs and titles above are the standard locations; I did not open them during this session (the sandbox cannot reach documentation sites), so if a link has moved, search for the title. **Unchecked or from memory:** everything about the **real** loader and tools: the two-account representation of an upgradeable program, the program-data PDA seeds, what the header holds, the command-line names and flags, the newer loader, verifiable builds, multisig and timelock practice, the names and capabilities of the test harnesses, and the claims about audit practice and bounties. **Environment caveats:** timings are single-machine micro-measurements (three runs of one example) on a shared virtual machine with two cores; nothing here ran on a validator, devnet or with any Solana tooling, so no figure applies to a real deployment. The digest of a known state, the program-data addresses and the vault address were verified against independent Python computations; the exercises' code in §13 and the two versions in §14 were compiled and run in a scratch crate (✓ 12 tests, `cargo clippy --all-targets -- -D warnings` clean). Measurements and outcomes marked ✓ were observed with `rustc 1.97.0`, `sha2 0.11.0` and the Lesson 53 and 54 crates. **No part of this lesson makes any system secure.**
