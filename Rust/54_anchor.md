---
layout: default
title: "Lesson 54 — Anchor, token programs, account validation"
---

# Lesson 54 — Anchor, token programs, account validation

**Module 17 · Solana (second lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `sha2 0.11.0`, plus the Lesson 53 crate `solanalab` as a path dependency (which brings `curve25519-dalek 5.0.0`; 23 packages in `Cargo.lock`, the crate itself included)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** Lesson 53 ended with a rule: *the caller chooses the accounts, so the program must check every one of them*. Writing those checks by hand, for every account of every instruction, is where most Solana security bugs come from, and it is what **Anchor** (the dominant Rust framework for Solana programs) automates: you describe each account's constraints in a struct, and generated code validates them before your function runs. This lesson builds that idea from the bottom, in the Lesson 53 runtime model: **discriminators** that tell account types apart, a **declarative constraint validator** (`validate`) with signer, mutability, owner, address, typed-account, `has_one`, seeds-and-bump, token-mint, token-authority, uninitialized and duplicate-account checks; a model of the **SPL token program** with the real byte layouts (an 82-byte mint, a 165-byte token account), delegates, freezing, closing and `TransferChecked`; **associated token accounts**; and an **escrow program** that uses all of it, locking tokens in a vault owned by a PDA and releasing them with a PDA signature. It is a **teaching model written from memory**: it does not use the real `anchor-lang` or `spl-token` crates, I did not install the Anchor toolchain or run anything on a validator, and the instruction layouts, error codes and program addresses are what I remember of the real ones, labeled "from memory" where they appear. **Not audited; nothing here secures anything.** Not covered: the macro machinery itself (procedural macros are Lesson 35), the IDL and TypeScript client, Token-2022 extensions (mentioned, not modeled), multisig token authorities, wrapped SOL, and testing and deployment (Lesson 55).

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation sites were not, so statements about the real Anchor and SPL token program are labeled "from memory". The discriminators, the associated-token-account and offer addresses and the bump numbers were computed by **separate Python programs** (✓: SHA-256 plus a pure-Python curve check), not by the code under test; the discriminator of `global:initialize`, `afaf6d1f0d989bed`, is also the value I remember from Anchor programs (✓ agreement). Timings are single-machine micro-measurements.

Builds on [Lesson 53](53_solana_model.md) (accounts, ownership rules, PDAs, CPI: this lesson's crate depends on it unchanged), [Lesson 52](52_web3_reliability.md) (idempotency, simulation), [Lesson 48](48_crypto_primitives.md) (hashes and domain separation), [Lesson 40](40_property_testing_fuzzing.md) (random-operation tests), [Lesson 35](35_proc_macros.md) is the background for what Anchor's derive macros do, [Lesson 26](26_error_design.md), [Lesson 21](21_trait_objects_dispatch.md) and [Lesson 15](15_newtype_typestate.md). Lesson 55 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

The programs that lost money on Solana mostly did not have clever bugs; they **forgot a check**. The lesson is organized around experiments that show what each check prevents:

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| Account types need a tag | 8-byte discriminators from `sha256("account:Name")`; a lookalike account with zeroed data is refused as `AccountDiscriminatorMismatch`; in the debugging challenge a "ticket" is accepted as a "pool" without one (§2.1, §11) |
| Constraints can be data | ten kinds of check, each tested to pass and to fail with its own error name and code; first failure wins (§2.3) |
| Validate first, then touch data | every escrow instruction runs `validate` before it reads or moves anything (§5) |
| A token account is 165 bytes and a mint 82 | the pack functions produce exact offsets and the unpackers refuse every wrong length and every invalid tag byte (§3.1) |
| Supply equals the sum of balances | 200 seeds of 60 random mint, burn, transfer, approve, revoke and freeze operations never break it, and a failed operation never changes state (§3.5) |
| Self-transfer must not change anything | my first version destroyed tokens on a transfer to oneself; found by a unit test (§3.4) |
| Delegates are limited | a delegate spends only its remaining allowance, which shrinks and disappears at zero; the owner is never limited (§3.3) |
| `TransferChecked` stops decimals mistakes | a wrong `decimals` value or a wrong mint is refused (§3.4) |
| The associated account is canonical | its address is a PDA of wallet, token program and mint; a wrong address, a foreign owner or a foreign mint is refused; creating twice is an error unless idempotent (§4) |
| A vault can belong to a PDA | the escrow's vault is owned by the offer PDA, which signs the release through CPI; no key can move it (§5.2) |
| Fakes fail | a fake token program, a substituted vault, a wrong mint, a forged offer and a taker who is also the maker are each refused (§5.4) |
| Everything is atomic | a taker who cannot pay leaves the vault, the offer and every balance untouched (§5.4) |

Total in the practical challenge: one library crate, 5 modules, **91 tests**, a benchmark example, and **192 mutants** run against the test suite.

---

## 2. Account validation

### 2.1 Discriminators: which kind of account is this?

An account is just bytes. A program that defines two account types of the same size cannot tell them apart by length, and if it checks only the owner (which is the program itself, for both) an attacker can hand it the wrong type. This is **type cosplay**. The usual defense, and Anchor's (from memory), is a **discriminator**: the first 8 bytes of every account the program creates, derived from the type's name:

```rust
/// Discriminator stored at the start of an account of type `name`
/// (`sha256("account:<Name>")[..8]`).
pub fn account(name: &str) -> [u8; 8] {
    disc("account", name)
}

/// Discriminator at the start of the data of instruction `name`
/// (`sha256("global:<snake_case_name>")[..8]`).
pub fn instruction(name: &str) -> [u8; 8] {
    disc("global", name)
}

fn disc(namespace: &str, name: &str) -> [u8; 8] {
    let mut h = Sha256::new();
    h.update(namespace.as_bytes());
    h.update(b":");
    h.update(name.as_bytes());
    let full = h.finalize();
    full[..8].try_into().expect("8 bytes")
}
```

`account("Offer")` is the first eight bytes of `SHA-256("account:Offer")`, and an instruction's discriminator is the first eight bytes of `SHA-256("global:<snake_case_name>")` (the namespaces keep an account called `initialize` from colliding with an instruction called `initialize`). ✓ Known answers computed with Python's `hashlib`: `global:initialize` → `afaf6d1f0d989bed` (the value I remember Anchor programs starting with), `global:make_offer` → `d66261233b0c2cb2`, `account:Offer` → `d7583c47aaa249e5` (`discriminator_known_answers`; `account_and_instruction_namespaces_differ` pins the separation). Eight bytes is a trade-off: with `n` names the chance of any two sharing a prefix is about `n²/2⁶⁵`, negligible for a program, whereas one byte collides for sure among 257 names (exercise 6 finds a collision with the crate's hash). The cost is 8 bytes of rent-paying space per account.

### 2.2 Anchor in one paragraph

You write (from memory of Anchor's syntax):

```text
#[derive(Accounts)]
pub struct TakeOffer<'info> {
    #[account(mut)] pub taker: Signer<'info>,
    #[account(mut)] pub maker: SystemAccount<'info>,
    #[account(mut, has_one = maker, has_one = mint_a, close = maker,
              seeds = [b"offer", maker.key().as_ref(), &offer.id.to_le_bytes()], bump = offer.bump)]
    pub offer: Account<'info, Offer>,
    ...
}
```

and the macro generates code that, before your instruction function runs, deserializes each account, checks its discriminator and owner (`Account<'info, T>`), checks `Signer`, mutability, `has_one` (a stored key equals another account's key), the PDA derivation (`seeds`, `bump`), token constraints (`token::mint`, `token::authority`), and, for `init`, creates the account with a CPI to the system program. Your function receives a typed context. Three consequences for a Rust engineer: the checks are **declarative**, so a reviewer reads one struct and sees the whole trust boundary; the checks run **before** your logic, in a fixed order; and the safety stops at the declaration: a constraint you forgot to write is a check that is not there, and `UncheckedAccount` (a raw account with no checks, which the documentation asks you to justify) means "I validate this myself". The model below is the same design with constraints as **data** instead of macro output, which makes it easy to test every check on its own.

### 2.3 The constraint validator ✓

```rust
/// One constraint on one account.
#[derive(Debug, Clone)]
pub enum Check {
    /// The account signed (`#[account(signer)]`).
    Signer,
    /// The account is writable (`#[account(mut)]`).
    Mut,
    /// The account is exactly this address (`address = …`).
    Address(Pubkey),
    /// The account is this executable program (`Program<'info, T>`).
    Program(Pubkey),
    /// The account is owned by this program.
    Owner(Pubkey),
    /// An account of a program-defined type: owner, exact length, discriminator
    /// (`Account<'info, T>`).
    Typed {
        /// Required owner.
        owner: Pubkey,
        /// Required first eight bytes.
        disc: [u8; 8],
        /// Required data length.
        len: usize,
    },
    /// 32 bytes at `offset` in this account's data equal the address of account
    /// `target` (`has_one = …`).
    HasOne {
        /// Offset of the stored key.
        offset: usize,
        /// Index of the account it must equal.
        target: usize,
    },
    /// This account is the PDA derived from `seeds` under `program`
    /// (`seeds = [...], bump`).
    Seeds {
        /// The seeds, in order.
        seeds: Vec<Seed>,
        /// Where the bump comes from.
        bump: Bump,
        /// The deriving program.
        program: Pubkey,
    },
    /// This token account's mint is account `target` (`token::mint`).
    TokenMint(usize),
    /// This token account's owner is account `target` (`token::authority`).
    TokenOwner(usize),
    /// The account is unused: empty data, system-owned (the precondition of `init`).
    Uninit,
}
```

```rust
/// Runs every constraint of every account. Returns the first failure, in
/// account order and check order. Also rejects the same writable account
/// appearing twice (`ConstraintDuplicateMutableAccount`).
pub fn validate(inv: &Invoke<'_>, specs: &[Spec]) -> Result<(), AnchorError> {
    if inv.len() < specs.len() {
        return Err(err(3005, "AccountNotEnoughKeys", inv.len(), "-"));
    }
    for (i, s) in specs.iter().enumerate() {
        let key = inv.key(i).expect("length checked");
        let acct = inv.account(i).expect("length checked");
        let f = s.name;
        let mut is_mut = false;
        for c in &s.checks {
            match c {
                Check::Signer => {
                    if !inv.is_signer(i).expect("i") {
                        return Err(err(2002, "ConstraintSigner", i, f));
                    }
                }
                Check::Mut => {
                    is_mut = true;
                    if !inv.is_writable(i).expect("i") {
                        return Err(err(2000, "ConstraintMut", i, f));
                    }
                }
                Check::Address(a) => {
                    if key != *a {
                        return Err(err(2012, "ConstraintAddress", i, f));
                    }
                }
                Check::Program(p) => {
                    if key != *p {
                        return Err(err(3008, "InvalidProgramId", i, f));
                    }
                    if !acct.executable {
                        return Err(err(3009, "InvalidProgramExecutable", i, f));
                    }
                }
                Check::Owner(o) => {
                    if acct.owner != *o {
                        return Err(err(2004, "ConstraintOwner", i, f));
                    }
                }
                Check::Typed { owner, disc, len } => {
                    if acct.owner != *owner {
                        return Err(err(3007, "AccountOwnedByWrongProgram", i, f));
                    }
                    if acct.data.len() < 8 {
                        return Err(err(3001, "AccountDiscriminatorNotFound", i, f));
                    }
                    if acct.data[..8] != disc[..] {
                        return Err(err(3002, "AccountDiscriminatorMismatch", i, f));
                    }
                    if acct.data.len() != *len {
                        return Err(err(3003, "AccountDidNotDeserialize", i, f));
                    }
                }
                Check::HasOne { offset, target } => {
                    let stored = acct.data.get(*offset..*offset + 32);
                    let want = inv
                        .key(*target)
                        .map_err(|_| err(3005, "AccountNotEnoughKeys", *target, f))?;
                    if stored != Some(&want.0[..]) {
                        return Err(err(2001, "ConstraintHasOne", i, f));
                    }
                }
                Check::Seeds {
                    seeds,
                    bump,
                    program,
                } => {
                    let mut parts: Vec<Vec<u8>> = Vec::new();
                    for s in seeds {
                        parts.push(match s {
                            Seed::Bytes(b) => b.clone(),
                            Seed::Key(j) => inv
                                .key(*j)
                                .map_err(|_| err(3005, "AccountNotEnoughKeys", *j, f))?
                                .0
                                .to_vec(),
                            Seed::OwnData(o, l) => acct
                                .data
                                .get(*o..*o + *l)
                                .ok_or_else(|| err(2006, "ConstraintSeeds", i, f))?
                                .to_vec(),
                        });
                    }
                    let refs: Vec<&[u8]> = parts.iter().map(Vec::as_slice).collect();
                    let derived = match bump {
                        Bump::Canonical => find_program_address(&refs, program).ok().map(|x| x.0),
                        Bump::Stored(o) => {
                            let b = acct.data.get(*o).copied();
                            b.and_then(|b| {
                                let bs = [b];
                                let mut with: Vec<&[u8]> = refs.clone();
                                with.push(&bs);
                                create_program_address(&with, program).ok()
                            })
                        }
                    };
                    if derived != Some(key) {
                        return Err(err(2006, "ConstraintSeeds", i, f));
                    }
                }
                Check::TokenMint(j) => {
                    let want = inv
                        .key(*j)
                        .map_err(|_| err(3005, "AccountNotEnoughKeys", *j, f))?;
                    if acct.data.get(0..32) != Some(&want.0[..]) {
                        return Err(err(2014, "ConstraintTokenMint", i, f));
                    }
                }
                Check::TokenOwner(j) => {
                    let want = inv
                        .key(*j)
                        .map_err(|_| err(3005, "AccountNotEnoughKeys", *j, f))?;
                    if acct.data.get(32..64) != Some(&want.0[..]) {
                        return Err(err(2015, "ConstraintTokenOwner", i, f));
                    }
                }
                Check::Uninit => {
                    if !acct.data.is_empty() || acct.owner != Pubkey::default() {
                        return Err(err(2010, "ConstraintInit", i, f));
                    }
                }
            }
        }
        if is_mut {
            for (j, other) in specs.iter().enumerate().take(i) {
                let other_mut = other.checks.iter().any(|c| matches!(c, Check::Mut));
                if other_mut && inv.key(j).expect("j") == key {
                    return Err(err(2040, "ConstraintDuplicateMutableAccount", i, f));
                }
            }
        }
    }
    Ok(())
}
```

Each `Spec` names an account and lists `Check`s; `validate` walks the accounts in order and returns the **first** failure as an `AnchorError { code, name, account, field }`. The error codes follow the numbering I remember from Anchor's `ErrorCode` (2000s for constraints, 3000s for account errors); treat the numbers as illustrative. Each check has a test that makes it pass and a test that makes it fail with the exact name, code, index and field (`tests/validate_tests.rs`, run through the `Bank` with a tiny program that calls `validate`):

- **`Signer`, `Mut`**: `ConstraintSigner` (2002), `ConstraintMut` (2000).
- **`Address`, `Program`, `Owner`**: the account is exactly this key; is this key *and* executable (`InvalidProgramId` 3008, `InvalidProgramExecutable` 3009); is owned by this program (`ConstraintOwner` 2004).
- **`Typed { owner, disc, len }`**: the model of `Account<'info, T>`: owner first (`AccountOwnedByWrongProgram` 3007), then that 8 bytes exist (`AccountDiscriminatorNotFound` 3001), then the discriminator (`AccountDiscriminatorMismatch` 3002), then the exact length (`AccountDidNotDeserialize` 3003). ✓ A 3-byte account is a clean error, not an index panic.
- **`HasOne { offset, target }`**: 32 bytes at `offset` in this account's data equal the *address* of account `target` (`ConstraintHasOne` 2001). ✓ An `offset` past the end of the data is an error, not a panic.
- **`Seeds { seeds, bump, program }`**: this account is the PDA derived from the seeds (`ConstraintSeeds` 2006). A seed can be fixed bytes, another account's key, or **bytes of this account's own data** (the offer's `id`). The bump is either searched (`Canonical`, about 3.7 µs) or **read from the account** (`Stored`, one hash plus one curve check), which is what you should do for accounts you created, as Lesson 53 §5.3 explained. ✓ A missing stored bump byte or an own-data range past the end fail cleanly.
- **`TokenMint(i)`, `TokenOwner(i)`**: the token account's mint field (bytes 0–32) or owner field (32–64) equals account `i` (`ConstraintTokenMint` 2014, `ConstraintTokenOwner` 2015). ✓ Swapped fields do not pass; a short account fails instead of panicking.
- **`Uninit`**: empty data and system-owned (the precondition for creating an account at this address).
- **Duplicates**: `validate` refuses the same *writable* account appearing in two writable slots (`ConstraintDuplicateMutableAccount`, 2040; from memory a check Anchor added in later versions). ✓ `the_same_writable_account_twice_is_refused`: two writable slots with one key fail, a read-only alias passes. Lesson 53's aliasing warning (§4.2) is why.
- **Not enough accounts**: `AccountNotEnoughKeys` (3005), reported before any other check.

`From<AnchorError> for Error` maps these onto the runtime's error type so a program can write `validate(inv, &specs)?` (✓ `anchor_errors_map_to_runtime_errors`). The model has no `constraint = <expression>` check (arbitrary predicate); in Anchor it is the escape hatch, and the usual advice is to prefer a named constraint when one exists.

### 2.4 What validation does not do

Validation answers "are these the accounts I expect?". It does not answer "is this action allowed *now*, in *this* state, with *this* amount?". That logic stays in your function: amounts, ordering, timing, arithmetic (checked), and state-machine rules. The most common failure after "forgot a constraint" is "validated the accounts and then trusted a **number**".

---

## 3. A token program

### 3.1 Mints and token accounts: two layouts ✓

A fungible token on Solana is not stored in a contract with a mapping from owner to balance; it is **one account per holder per token**, all owned by the token program, plus one **mint** account that holds the token's rules. The model uses the layouts I remember of the real program:

- **Mint (82 bytes)**: `mint_authority` as an optional key (a 4-byte tag, 0 or 1, then 32 bytes), `supply` (u64), `decimals` (u8), `is_initialized` (u8), `freeze_authority` (optional key). ✓ `mint_layout_is_82_bytes_with_known_offsets` checks every offset against hand-written bytes: the supply `0x0102030405060708` is `08 07 … 01` at 36–44, decimals at 44, the flag at 45.
- **Token account (165 bytes)**: `mint` (32), `owner` (32), `amount` (u64), `delegate` (optional key), `state` (0 uninitialized, 1 initialized, 2 frozen), `is_native` (optional u64), `delegated_amount` (u64), `close_authority` (optional key). ✓ `token_account_layout_is_165_bytes_with_known_offsets` pins offsets 0, 32, 64, 72, 108, 109, 121, 129 and 133.

Two ownerships are easy to mix up. The **account's owner** (the runtime field of Lesson 53) is the token program; the token account's **owner field** (bytes 32–64) is the wallet that controls the balance. A wallet "owns" a token account only in the second sense, and a program (a PDA) can be that owner, which is what makes vaults possible (§5).

Parsing is strict (✓ `unpacking_is_strict`): exact length, option tags 0 or 1 only, a flag byte 0 or 1, a state byte 0–2; every other value is `InvalidAccountData`. An all-zero 165-byte account is a valid *uninitialized* account, which is how `InitializeAccount` can tell a fresh account from a used one. *Decimals* are display only: the program counts the smallest units, a mint with 6 decimals shows `1_500_000` as `1.5` (exercise 2).

### 3.2 Instructions

The instruction set is small (tags from memory; I implemented the ones that exercise the ownership model and left out multisig, `SetAuthority` and wrapped SOL): `InitializeMint` (0), `InitializeAccount` (1), `Transfer` (3), `Approve` (4), `Revoke` (5), `MintTo` (7), `Burn` (8), `CloseAccount` (9), `FreezeAccount` (10), `ThawAccount` (11) and `TransferChecked` (12). Builders in `token.rs` produce the exact account lists. Every handler follows one shape: check the signer, load and **validate each account's owner and layout**, run a pure rule, write back. The rules live in pure functions so that they can be tested without a bank.

### 3.3 The rules as pure functions ✓

```rust
/// May `authority` spend `amount`? The owner always may; the delegate may
/// up to its allowance, which shrinks (and disappears at zero).
pub fn authorize(&mut self, authority: &Pubkey, amount: u64) -> Result<(), TokenError> {
    if self.owner == *authority {
        return Ok(());
    }
    if self.delegate == Some(*authority) {
        if self.delegated_amount < amount {
            return Err(TokenError::InsufficientFunds);
        }
        self.delegated_amount -= amount;
        if self.delegated_amount == 0 {
            self.delegate = None;
        }
        return Ok(());
    }
    Err(TokenError::OwnerMismatch)
}

/// Moves `amount` to `dest`. `checked` carries `(mint key, mint decimals, decimals given)`
/// for `TransferChecked`.
pub fn transfer(
    &mut self,
    dest: &mut TokenAccount,
    authority: &Pubkey,
    amount: u64,
    checked: Option<(&Pubkey, u8, u8)>,
) -> Result<(), TokenError> {
    if self.state == State::Uninitialized || dest.state == State::Uninitialized {
        return Err(TokenError::Uninitialized);
    }
    if self.mint != dest.mint {
        return Err(TokenError::MintMismatch);
    }
    if let Some((mint_key, mint_decimals, given)) = checked {
        if self.mint != *mint_key {
            return Err(TokenError::MintMismatch);
        }
        if mint_decimals != given {
            return Err(TokenError::DecimalsMismatch);
        }
    }
    if self.state == State::Frozen || dest.state == State::Frozen {
        return Err(TokenError::AccountFrozen);
    }
    if self.amount < amount {
        return Err(TokenError::InsufficientFunds);
    }
    let new_dest = dest
        .amount
        .checked_add(amount)
        .ok_or(TokenError::Overflow)?;
    self.authorize(authority, amount)?;
    self.amount -= amount;
    dest.amount = new_dest;
    Ok(())
}
```

- **Authority.** The owner may always spend. A delegate may spend up to its remaining allowance, and each spend reduces the allowance; at zero the delegate is cleared (✓ `delegates_spend_up_to_their_allowance`: allowance 5, a spend of 6 fails with `InsufficientFunds` and changes nothing, 3 leaves 2, 2 more clears the delegate, a further spend is `OwnerMismatch`).
- **Frozen accounts** neither send nor receive; **uninitialized** ones are refused first (`Uninitialized`). The order of the checks decides which error is reported when several apply, and the tests pin it.
- **Checks before effects.** `transfer` computes `new_dest` with a checked add and runs `authorize` (which can reduce the delegated allowance) only after every other check has passed, so a failing transfer leaves both structs unchanged (✓ `transfer_failures_change_nothing`: after an insufficient balance, a wrong authority and a destination at `u64::MAX`, the structs are equal to their copies).
- **Mint rules** (`mint_to`, `burn`): only the mint authority mints, a mint with `None` as authority can never mint again (a fixed supply), supply and balance use checked arithmetic, burns reduce both (✓ `minting_needs_the_authority_and_updates_supply_and_balance`, `burning_reduces_balance_and_supply`).

```rust
/// Creates tokens in `dest` (which must belong to `mint_key`).
pub fn mint_to(
    &mut self,
    mint_key: &Pubkey,
    dest: &mut TokenAccount,
    authority: &Pubkey,
    amount: u64,
) -> Result<(), TokenError> {
    if !self.is_initialized || dest.state == State::Uninitialized {
        return Err(TokenError::Uninitialized);
    }
    if dest.mint != *mint_key {
        return Err(TokenError::MintMismatch);
    }
    if dest.state == State::Frozen {
        return Err(TokenError::AccountFrozen);
    }
    if self.mint_authority != Some(*authority) {
        return Err(TokenError::OwnerMismatch);
    }
    let supply = self
        .supply
        .checked_add(amount)
        .ok_or(TokenError::Overflow)?;
    let balance = dest
        .amount
        .checked_add(amount)
        .ok_or(TokenError::Overflow)?;
    self.supply = supply;
    dest.amount = balance;
    Ok(())
}
```

### 3.4 `TransferChecked`, self-transfers and a bug I wrote ✓

`Transfer` takes an amount in raw units. A client that thinks a token has 6 decimals when it has 9 sends 1,000 times too little or too much, with no error. **`TransferChecked`** carries the *mint account* and the *decimals the client believes* and the program refuses if either disagrees (✓ `transfer_checked_compares_mint_and_decimals`, and through the bank `transfer_checked_validates_decimals_and_mint`). Use the checked variants for anything that moves value on behalf of a user; the same advice applies to a program that CPIs into the token program.

**The self-transfer bug.** The first version of the token program handled `source == destination` by running the transfer on a *copy* of the destination and then storing the *source*, which now held the debit and none of the credit: ✓ `self_transfer_checks_but_moves_nothing` failed with `left: 0, right: 10`: a transfer of 10 tokens to oneself destroyed all 10. The fix is to run every check and write nothing (the real program does the same, from memory). The cause is a trap in any design that loads two accounts by index: when two indexes name the same account, "load both, change both, write both" is wrong, and "the last write wins" is the best case. Lesson 53 §4.2 listed aliasing as a check to remember; this bug is what forgetting it looks like.

### 3.5 The program through the bank ✓

```rust
impl Program for TokenProgram {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(COST)?;
        let tag = *data.first().ok_or(Error::InvalidInstructionData)?;
        match tag {
            0 => {
                if data.len() != 1 + 1 + 32 + 1 + 32 && data.len() != 1 + 1 + 32 + 1 {
                    return Err(Error::InvalidInstructionData);
                }
                let freeze = match data[34] {
                    0 if data.len() == 35 => None,
                    1 if data.len() == 67 => Some(Pubkey(data[35..67].try_into().expect("32"))),
                    _ => return Err(Error::InvalidInstructionData),
                };
                let acct = inv.account(0)?;
                if acct.owner != inv.program_id() || acct.data.len() != MINT_LEN {
                    return Err(Error::InvalidAccountData);
                }
                if Mint::unpack(&acct.data)?.is_initialized {
                    return Err(TokenError::AlreadyInUse.into());
                }
                let mint = Mint {
                    mint_authority: Some(Pubkey(data[2..34].try_into().expect("32"))),
                    supply: 0,
                    decimals: data[1],
                    is_initialized: true,
                    freeze_authority: freeze,
                };
                store(inv, 0, mint.pack())
            }
            1 => {
                if data.len() != 1 {
                    return Err(Error::InvalidInstructionData);
                }
                let acct = inv.account(0)?;
                if acct.owner != inv.program_id() {
                    return Err(Error::WrongOwner(0));
                }
                if TokenAccount::unpack(&acct.data)?.state != State::Uninitialized {
                    return Err(TokenError::AlreadyInUse.into());
                }
                load_mint(inv, 1)
                    .ok()
                    .filter(|m| m.is_initialized)
                    .ok_or(Error::InvalidAccountData)?;
                let t = TokenAccount::new(inv.key(1)?, inv.key(2)?);
                store(inv, 0, t.pack())
            }
            3 | 12 => {
                let (s, d, a, mint_i) = if tag == 3 {
                    (0, 1, 2, None)
                } else {
                    (0, 2, 3, Some(1))
                };
                let amount = amount_arg(data, usize::from(tag == 12))?;
                need_signer(inv, a)?;
                let mut src = load_token(inv, s)?;
                let mut dst = load_token(inv, d)?;
                let authority = inv.key(a)?;
                let checked = match mint_i {
                    Some(m) => {
                        let mint = load_mint(inv, m)?;
                        Some((inv.key(m)?, mint.decimals, data[9]))
                    }
                    None => None,
                };
                let checked_ref = checked.as_ref().map(|(k, a, b)| (k, *a, *b));
                if inv.key(s)? == inv.key(d)? {
                    // all checks run, but nothing is written: moving tokens to
                    // oneself must not create or destroy any
                    src.transfer(&mut dst, &authority, amount, checked_ref)?;
                    return Ok(());
                }
                src.transfer(&mut dst, &authority, amount, checked_ref)?;
                store(inv, s, src.pack())?;
                store(inv, d, dst.pack())
            }
            4 => {
                let amount = amount_arg(data, 0)?;
                need_signer(inv, 2)?;
                let mut src = load_token(inv, 0)?;
                if src.owner != inv.key(2)? {
                    return Err(TokenError::OwnerMismatch.into());
                }
                if src.state == State::Frozen {
                    return Err(TokenError::AccountFrozen.into());
                }
                src.delegate = Some(inv.key(1)?);
                src.delegated_amount = amount;
                store(inv, 0, src.pack())
            }
            5 => {
                need_signer(inv, 1)?;
                let mut src = load_token(inv, 0)?;
                if src.owner != inv.key(1)? {
                    return Err(TokenError::OwnerMismatch.into());
                }
                src.delegate = None;
                src.delegated_amount = 0;
                store(inv, 0, src.pack())
            }
            7 => {
                let amount = amount_arg(data, 0)?;
                need_signer(inv, 2)?;
                let mut mint = load_mint(inv, 0)?;
                let mut dst = load_token(inv, 1)?;
                let (mk, auth) = (inv.key(0)?, inv.key(2)?);
                mint.mint_to(&mk, &mut dst, &auth, amount)?;
                store(inv, 0, mint.pack())?;
                store(inv, 1, dst.pack())
            }
            8 => {
                let amount = amount_arg(data, 0)?;
                need_signer(inv, 2)?;
                let mut acct = load_token(inv, 0)?;
                let mut mint = load_mint(inv, 1)?;
                let (mk, auth) = (inv.key(1)?, inv.key(2)?);
                mint.burn(&mk, &mut acct, &auth, amount)?;
                store(inv, 0, acct.pack())?;
                store(inv, 1, mint.pack())
            }
            9 => {
                if data.len() != 1 {
                    return Err(Error::InvalidInstructionData);
                }
                need_signer(inv, 2)?;
                let acct = load_token(inv, 0)?;
                if acct.state == State::Uninitialized {
                    return Err(TokenError::Uninitialized.into());
                }
                if acct.amount != 0 {
                    return Err(TokenError::NonZeroBalance.into());
                }
                let who = inv.key(2)?;
                if who != acct.close_authority.unwrap_or(acct.owner) {
                    return Err(TokenError::OwnerMismatch.into());
                }
                if inv.key(0)? == inv.key(1)? {
                    return Err(Error::InvalidInstructionData);
                }
                let lamports = inv.account(0)?.lamports;
                let total = inv
                    .account(1)?
                    .lamports
                    .checked_add(lamports)
                    .ok_or(Error::ArithmeticOverflow)?;
                inv.account_mut(1)?.lamports = total;
                let a = inv.account_mut(0)?;
                a.lamports = 0;
                a.data = Vec::new();
                a.owner = Pubkey::default();
                Ok(())
            }
            10 | 11 => {
                if data.len() != 1 {
                    return Err(Error::InvalidInstructionData);
                }
                need_signer(inv, 2)?;
                let mut acct = load_token(inv, 0)?;
                let mint = load_mint(inv, 1)?;
                if acct.mint != inv.key(1)? {
                    return Err(TokenError::MintMismatch.into());
                }
                let Some(fa) = mint.freeze_authority else {
                    return Err(TokenError::NoFreezeAuthority.into());
                };
                if fa != inv.key(2)? {
                    return Err(TokenError::OwnerMismatch.into());
                }
                acct.state = match (tag, acct.state) {
                    (_, State::Uninitialized) => return Err(TokenError::Uninitialized.into()),
                    (10, State::Frozen) | (11, State::Initialized) => {
                        return Err(Error::InvalidAccountData);
                    }
                    (10, _) => State::Frozen,
                    _ => State::Initialized,
                };
                store(inv, 0, acct.pack())
            }
            _ => Err(Error::InvalidInstructionData),
        }
    }
}
```

`tests/token_program.rs` runs every instruction through a `Bank`: minting 1,000 tokens, moving 400 and burning 150 leaves balances 600, 250 and supply 850; `InitializeMint` and `InitializeAccount` refuse a second initialization (`AlreadyInUse`), an uninitialized mint and an account that is not a mint; transfers by a signer who is not the owner give `WrongAuthority`, and by an owner who does not sign are stopped by the signature check; accounts owned by another program as source or destination give `WrongOwner(0)` or `WrongOwner(1)`; freezing blocks transfers until the freeze authority thaws, and the account owner cannot thaw; a mint without a freeze authority cannot freeze; `CloseAccount` needs a zero balance, the owner's signature, and moves the rent to the destination; malformed instruction data of five different shapes is refused.

✓ **Random operations** (`tests/props.rs`, the style of Lesson 40): 200 seeds of 60 operations each, chosen from `MintTo`, `Transfer` (twice as likely), `Burn`, `Approve`, `Revoke`, `Freeze`/`Thaw` and `TransferChecked`, with random signers (often the wrong one) over six token accounts of two mints. After every operation: for each mint, `supply == Σ balances`, total lamports are unchanged (fees go to a pot that `total_supply` counts), and **if the operation failed, every token account and mint is byte-for-byte unchanged**. 1,107 of the 12,000 operations succeeded (the rest were refused for the reasons above, which is the point of choosing signers at random); a second test checks that a simple mint-transfer-burn mix succeeds more than a third of the time, so the first test cannot pass vacuously.

---

## 4. Associated token accounts

### 4.1 The problem and the answer ✓

To send someone tokens you need *their token account for that mint*. If wallets could have any number of token accounts per mint at any address, the sender would have to ask. The convention (from memory, the **associated token account** standard) is one canonical account per (wallet, mint) at the **PDA** of the seeds `[wallet, token program id, mint]` under the associated-token-account program:

```rust
/// The address and bump of `wallet`'s associated token account for `mint`:
/// the PDA of `[wallet, token program, mint]` under the ATA program.
pub fn associated_address(wallet: &Pubkey, mint: &Pubkey) -> (Pubkey, u8) {
    find_program_address(&[&wallet.0, &TOKEN_PROGRAM_ID.0, &mint.0], &ATA_PROGRAM_ID)
        .expect("a bump exists")
}
```

The sender computes the address, and if the account does not exist yet, the sender (or anyone) creates it, paying the rent. ✓ Known answers from an independent derivation: wallet `[1;32]` and mint `[2;32]` give `CsYkfSfTUTWwnoeRkGchtai5kkYz2SC33kKJwA99wVr3` (bump 255); wallet `[7;32]` and mint `[9;32]` give `BjmJ1yi1Sc4s9xQaiv4DbRuUhgfjUSc8cYSuwsFqoS9` (bump 255) (`associated_addresses_match_an_independent_derivation`); swapping wallet and mint gives a different address. The program ids are the well-known ones (✓ `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` and `ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL` decode to the 32 bytes in the source and print back as these strings; I wrote them from memory).

### 4.2 Creating one is two CPIs ✓

```rust
impl Program for AtaProgram {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(COST)?;
        let idempotent = match data {
            [] => false,
            [1] => true,
            _ => return Err(Error::InvalidInstructionData),
        };
        if !inv.is_signer(0)? {
            return Err(Error::NotSigner(0));
        }
        if inv.key(4)? != Pubkey::default() || inv.key(5)? != TOKEN_PROGRAM_ID {
            return Err(Error::InvalidInstructionData);
        }
        let (payer, ata, wallet, mint) = (inv.key(0)?, inv.key(1)?, inv.key(2)?, inv.key(3)?);
        let m = inv.account(3)?;
        if m.owner != TOKEN_PROGRAM_ID || !token::Mint::unpack(&m.data)?.is_initialized {
            return Err(Error::InvalidAccountData);
        }
        inv.consume(1500)?; // the PDA search
        let (expected, bump) = associated_address(&wallet, &mint);
        if expected != ata {
            return Err(Error::InvalidSeeds);
        }
        let existing = inv.account(1)?;
        if existing.owner == TOKEN_PROGRAM_ID {
            if !idempotent {
                return Err(Error::AlreadyInUse);
            }
            let t = TokenAccount::unpack(&existing.data)?;
            if t.owner != wallet || t.mint != mint {
                return Err(Error::InvalidSeeds);
            }
            return Ok(());
        }
        if !existing.data.is_empty() || existing.owner != Pubkey::default() {
            return Err(Error::AlreadyInUse);
        }
        let lamports = rent_exempt_minimum(ACCOUNT_LEN).saturating_sub(existing.lamports);
        let bump_seed = [bump];
        let seeds: &[&[u8]] = &[&wallet.0, &TOKEN_PROGRAM_ID.0, &mint.0, &bump_seed];
        let create = Instruction {
            program_id: Pubkey::default(),
            accounts: vec![
                AccountMeta::writable(payer, true),
                AccountMeta::writable(ata, true),
            ],
            data: system::create_account_data(lamports, ACCOUNT_LEN as u64, &TOKEN_PROGRAM_ID),
        };
        inv.invoke(&create, &[seeds])?;
        inv.invoke(&token::initialize_account(ata, mint, wallet), &[])
    }
}
```

The program: requires the payer's signature; requires that the system and token program accounts are the real ones; requires that the mint is owned by the token program and initialized; **recomputes the PDA** (charging 1,500 units for the search) and refuses any other address (`InvalidSeeds`); if a token account is already there, `Create` fails with `AlreadyInUse` and `CreateIdempotent` succeeds only if its owner and mint are the right ones; otherwise it creates the account with the **system program** (`CreateAccount`, 165 bytes, owned by the token program, signed by the PDA's seeds) and then **initializes** it with the token program. ✓ Tests: the created account has owner, size, rent and contents exactly as expected and the payer pays rent plus fee; a **prefunded** address (anyone can send lamports there first) is topped up by the difference, and a prefund above the minimum is simply kept (as in Lesson 53 §7.2); a foreign owner or mint on an existing account is refused for the idempotent variant; foreign data or a foreign owner at the address is refused; creation, use and idempotent re-creation work inside **one transaction**.

A word on the trust the ATA program places in you: it creates *an* account for *that* wallet; it does not ask the wallet. Anyone can create your associated account for any mint (and make you pay nothing), which is harmless, but also means the existence of the account says nothing about whether you want that token.

---

## 5. An escrow built from these parts

### 5.1 Design ✓

A **maker** wants to swap `amount_a` of token A for `amount_b` of token B. The program: `make_offer` locks the A tokens in a **vault**, a token account whose *owner field* is the **offer PDA** (so no keypair can move them) and records the terms in an `Offer` account at that PDA; `take_offer` lets a **taker** pay `amount_b` of B to the maker and receive the vault's A tokens, then **closes** the vault and the offer, returning rent to the maker; `cancel_offer` lets the maker take the tokens back.

```rust
/// The decoded `Offer` account.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Offer {
    /// Who made the offer and receives token B.
    pub maker: Pubkey,
    /// The token locked in the vault.
    pub mint_a: Pubkey,
    /// The token asked for.
    pub mint_b: Pubkey,
    /// Chosen by the maker so one maker can have many offers.
    pub id: u64,
    /// How much token B the taker must pay.
    pub amount_b: u64,
    /// Canonical bump of the offer's PDA.
    pub bump: u8,
}
```

The `Offer` account is 121 bytes: an 8-byte discriminator, the maker, both mints, the maker's chosen `id` (so one maker can have many offers), the price and the bump (✓ `offer_packs_and_unpacks_strictly`: length, discriminator and every field). It lives at the PDA of `[b"offer", maker, id]`; ✓ independent derivations: maker `[1;32]` with id 7 gives `HbrjB4sbqSEh7jDKk8Kjw5jLHZ32aZ9HNhX1RgpCpkkq` with bump 254, id 8 gives `AW2gBw24cM6KEEFYk7qhzu6pgdzTANPeDH16BtjUv4o7`, maker `[2;32]` with id 7 gives `2X1W6tGJdNWdjvtk63ZJrtUvr87Q5zPoQQpw2HMCLWFZ` (program `[0xE5;32]`). The vault is the *associated* token account of the offer PDA for mint A, so its address is known from the offer alone.

### 5.2 Locking: `make_offer` ✓

```rust
fn make_offer(&self, inv: &mut Invoke<'_>, me: Pubkey, data: &[u8]) -> Result<(), Error> {
    if data.len() != 8 + 24 {
        return Err(Error::InvalidInstructionData);
    }
    let (id, amount_a, amount_b) = (u64_arg(data, 8), u64_arg(data, 16), u64_arg(data, 24));
    if amount_a == 0 || amount_b == 0 {
        return Err(Error::InvalidInstructionData);
    }
    inv.consume(1500)?; // two PDA searches would be charged here by the real runtime
    let (maker, mint_a) = (inv.key(0)?, inv.key(2)?);
    let (offer_key, bump) = offer_address(&me, &maker, id);
    let vault_key = associated_address(&offer_key, &mint_a).0;
    let specs: Vec<Spec> = vec![
        spec("maker", vec![Check::Signer, Check::Mut]),
        spec(
            "offer",
            vec![Check::Mut, Check::Address(offer_key), Check::Uninit],
        ),
        spec("mint_a", vec![Check::Owner(TOKEN_PROGRAM_ID)]),
        spec("mint_b", vec![Check::Owner(TOKEN_PROGRAM_ID)]),
        spec("maker_ata_a", token_acct(2, 0)),
        spec(
            "vault",
            [token_acct(2, 1), vec![Check::Address(vault_key)]].concat(),
        ),
        spec("token_program", vec![Check::Program(TOKEN_PROGRAM_ID)]),
        spec("system_program", vec![Check::Program(Pubkey::default())]),
    ];
    validate(inv, &specs)?;

    // create the offer account, signing for its PDA
    let seeds = offer_seeds(maker, id, bump);
    let seed_refs: Vec<&[u8]> = seeds.iter().map(Vec::as_slice).collect();
    let lamports = rent_exempt_minimum(OFFER_LEN).saturating_sub(inv.account(1)?.lamports);
    let create = Instruction {
        program_id: Pubkey::default(),
        accounts: vec![
            AccountMeta::writable(maker, true),
            AccountMeta::writable(offer_key, true),
        ],
        data: system::create_account_data(lamports, OFFER_LEN as u64, &me),
    };
    inv.invoke(&create, &[&seed_refs])?;
    let offer = Offer {
        maker,
        mint_a,
        mint_b: inv.key(3)?,
        id,
        amount_b,
        bump,
    };
    inv.account_mut(1)?.data = offer.pack();

    // lock token A in the vault; the maker's signature extends through the call
    let ix = token::transfer(inv.key(4)?, vault_key, maker, amount_a);
    inv.invoke(&ix, &[])
}
```

Everything is validated before anything happens: the maker signs and is writable; the offer is writable, at the **derived address** and **unused** (`Uninit`: a second offer with the same id is refused, which stops a maker, or an attacker, from overwriting the terms of a live offer: ✓ `make_twice_with_the_same_id_is_refused_and_leaves_the_vault_alone`); both mints are owned by the token program; the maker's token account is the maker's, for mint A; the vault is for mint A, owned by the offer, **at the associated address**; and the two programs are the real ones. Then the program creates the offer account through a CPI that the **offer PDA signs** (seeds `[b"offer", maker, id, bump]`), writes the terms, and CPIs the token program's `Transfer` from the maker's account to the vault: the maker's own signature extends through the call (Lesson 53 §6.1).

The client creates the vault first (an associated-account create in the same transaction, idempotent), because the escrow program only *checks* the vault; ✓ `creation_in_the_same_transaction_as_use` and the helper in the tests do exactly that. Zero amounts and malformed data are refused (✓ `make_offer_validation` runs thirteen bad variants: zero A, zero B, wrong data length, a wrong offer address, a vault that is not the associated account of the offer, a maker token account owned by someone else, a mint the token program does not own, a fake token program, a fake system program, an unsigned maker, and more tokens than the maker has, which fails inside the token CPI and leaves no offer behind).

### 5.3 Releasing: `take_offer` and `cancel_offer` ✓

```rust
fn take_offer(&self, inv: &mut Invoke<'_>, me: Pubkey) -> Result<(), Error> {
    let offer_key = inv.key(2)?;
    let mint_a_key = inv.key(3)?;
    let vault_key = associated_address(&offer_key, &mint_a_key).0;
    let specs: Vec<Spec> = vec![
        spec("taker", vec![Check::Signer, Check::Mut]),
        spec("maker", vec![Check::Mut]),
        spec(
            "offer",
            vec![
                Check::Mut,
                typed_offer(me),
                Check::HasOne {
                    offset: MAKER,
                    target: 1,
                },
                Check::HasOne {
                    offset: MINT_A,
                    target: 3,
                },
                Check::HasOne {
                    offset: MINT_B,
                    target: 4,
                },
                offer_pda_check(me, 1),
            ],
        ),
        spec("mint_a", vec![Check::Owner(TOKEN_PROGRAM_ID)]),
        spec("mint_b", vec![Check::Owner(TOKEN_PROGRAM_ID)]),
        spec("taker_ata_a", token_acct(3, 0)),
        spec("taker_ata_b", token_acct(4, 0)),
        spec("maker_ata_b", token_acct(4, 1)),
        spec(
            "vault",
            [token_acct(3, 2), vec![Check::Address(vault_key)]].concat(),
        ),
        spec("token_program", vec![Check::Program(TOKEN_PROGRAM_ID)]),
    ];
    validate(inv, &specs)?;
    let offer = Offer::unpack(&inv.account(2)?.data)?;
    let vault_amount = TokenAccount::unpack(&inv.account(8)?.data)?.amount;
    let (taker, maker) = (inv.key(0)?, inv.key(1)?);

    // taker pays the maker (taker's signature extends through the call)
    let pay = token::transfer(inv.key(6)?, inv.key(7)?, taker, offer.amount_b);
    inv.invoke(&pay, &[])?;

    // the offer PDA releases the vault and closes it
    let seeds = offer_seeds(maker, offer.id, offer.bump);
    let seed_refs: Vec<&[u8]> = seeds.iter().map(Vec::as_slice).collect();
    let release = token::transfer(inv.key(8)?, inv.key(5)?, offer_key, vault_amount);
    inv.invoke(&release, &[&seed_refs])?;
    let close = token::close_account(inv.key(8)?, maker, offer_key);
    inv.invoke(&close, &[&seed_refs])?;
    close_offer(inv, 2, 1)
}
```

After `validate`, the program decodes the offer, then makes three CPIs and one local change: (1) the **taker's** signature moves `amount_b` of B from the taker to the maker; (2) the **offer PDA** signs a token `Transfer` of the vault's *entire* balance to the taker; (3) the offer PDA signs `CloseAccount` for the vault, sending its rent to the maker; and finally the offer account itself is closed by hand (lamports to the maker, data cleared, owner reset, exactly the Lesson 53 §7.3 recipe, which matters again here because the vault close and the offer close happen inside one transaction). ✓ `take_offer_swaps_closes_and_refunds_rent`: taker ends with 400 A and 250 B less than he began, the maker with 600 A and 250 B, both closed accounts are gone, the maker's lamports rose by exactly the two rents, total lamports and A's supply are unchanged. `cancel_offer` is the same with the maker as the only party (✓ `cancel_returns_the_tokens_and_closes_everything`, and the id can then be reused).

### 5.4 The attacks, one by one ✓

Each of these is a test in `tests/escrow_tests.rs` that tries to make the program do something wrong and checks the error **and** that nothing moved:

| Attack | What stops it |
|---|---|
| The taker does not sign | `Signer` on the taker: `NotSigner(0)` |
| The "maker" account passed to `take_offer` is not the maker in the offer (to redirect the rent refund) | `HasOne` on the offer: `WrongAuthority` |
| Eve substitutes her own token account for the maker's payment destination | `TokenOwner` on `maker_ata_b`: the account's owner field is not the maker |
| Eve passes a token account holding token A as the vault | `Address` of the vault must be the associated address of the offer |
| A fake token program (so the "transfers" do nothing, or something else) | `Program(TOKEN_PROGRAM_ID)` |
| A different token B (a worthless mint of the same layout) | `HasOne` on `mint_b` against the offer |
| A **forged offer**: Eve assigns her own account to the escrow program, with zeroed data | the discriminator check (`AccountDiscriminatorMismatch`); she cannot write the data because only the program may |
| A forged offer with the right bytes at a different address | the seeds check, using the stored bump, finds the address is not the PDA |
| A second `take_offer` or a `cancel_offer` after a take | the offer account is gone, so its owner and discriminator fail |
| Maker and taker are the same wallet (aliasing) | the duplicate-writable check |
| The taker cannot pay | the token CPI fails with `InsufficientFunds`, the whole transaction is rolled back: the vault still holds 400, the offer exists (✓ `a_poor_taker_fails_atomically`) |
| Someone else cancels | `HasOne` against the maker, and `Signer`: ✓ `cancel_is_for_the_maker_only` |

Nothing in this table is exotic. Each is a missing line in a real program somewhere (from memory of public incident reports: I did not look any up). The point of a declarative validator is that the *list* of lines is short enough to review.

### 5.5 What this escrow is not

It takes the whole vault or nothing (no partial fills; exercise 8 computes a fair price for one); it has no expiry, no fee, no order book, and it trusts the maker's chosen price. It does not check that the vault is *empty* when the offer is made (a hostile maker could pre-fund the vault, which only helps the taker); the vault's `close_authority` and `delegate` fields are not examined (in the real token program a vault with a delegate set is a known hazard: a delegate could spend from it; an escrow should check that both are `None`, and this one does not). It does not handle tokens with transfer hooks or fees (Token-2022, below). And it is a model: no real signatures, no sysvars, no rent collection.

---

## 6. Beyond the model

- **Token-2022** (from memory): a newer token program with the same account structure plus *extensions* (transfer fees, interest-bearing amounts, non-transferable tokens, transfer hooks, confidential balances, and more) stored after the base layout. Programs that accept "any token" must decide which extensions to allow, because a transfer fee means the vault receives less than was sent and a transfer hook runs another program during your CPI. Exercise 7 is the arithmetic of a fee.
- **The Anchor toolchain** (not run here): `anchor init`, `anchor build`, `anchor test` (a local validator and a TypeScript test harness), the generated **IDL** (a JSON description of instructions and accounts that clients use to build instructions), `declare_id!`, `#[account]` structs with `InitSpace`, `require!`/`require_keys_eq!`, `emit!` events, `#[error_code]` custom errors, and `remaining_accounts` for a variable number of accounts. What you read in this lesson is what the macros expand to in spirit; reading the expansion (`cargo expand`) of a small program is the best next exercise on a machine that has the toolchain.
- **Alternatives**: writing programs directly against the `solana-program` crate (full control, no generated checks), or newer lighter frameworks; the checklist of §2.3 applies in all of them.
- **Testing** (Lesson 55): the real-runtime test harnesses (an in-process bank, a local validator, fuzzers), account snapshots and what to assert.

### 6.1 Cost ✓

Release build, single machine, three runs of `examples/bench.rs` (ranges are the lowest and highest): hashing a name for a discriminator takes about 59 ns; a PDA search for an associated address 3.68–3.71 µs (one curve check, §5.3 of Lesson 53, plus a hash); packing and unpacking a token account 20–21 ns; processing a one-instruction token transfer through the whole bank 2.20–2.25 µs and `transfer_checked` 2.41–2.47 µs; a `make_offer` (with vault creation) followed by a `cancel_offer` 63.5–64.7 µs, and a make followed by a take 66.4–72.0 µs. Most of the escrow's time is probably PDA searches (each instruction derives several addresses; I did not profile). Compute units in the model (✓ measured by the receipts): an associated-account creation 4,150, a `make_offer` 4,350, a `take_offer` 4,300, a `cancel_offer` 3,000, one token transfer 300; the whole make-with-vault-creation transaction 8,500, far below the 200,000 default per instruction (from memory); and the transactions are 475 bytes each (✓ `serialized_len`), well below 1,232. Real programs spend far more per instruction (deserialization, logging, the virtual machine); do not read these as predictions.

---

## 7. Rust notes

- **Constraints as data.** `Check` is an enum with fields, so a spec is a `Vec<Check>` that can be built at run time (the escrow computes the vault address from the offer's key and puts it in an `Address` check), compared, logged and tested. Anchor's macro produces the same logic as code at compile time, with the advantage that unused checks cost nothing; the cost of the data-driven version here is a few allocations per instruction (about the cost of a PDA search or less, from the benchmark).
- **Pure core, thin shell.** `TokenAccount::transfer`, `Mint::mint_to` and `burn` take plain structs and return `TokenError`; the program only loads, calls and stores. That is why the random test and the unit tests do not need a bank, and why the self-transfer bug (§3.4) lived in the shell and not the rules.
- **Strict decoders.** `unpack` functions return `Result` and check lengths first. Indexing a slice with a constant range panics on a short slice, and a panic in a program is an aborted transaction at best and a denial of service at worst; the tests include short and wrong-length inputs for every layout.
- **`Option<Pubkey>` and `COption`.** The on-chain format spells an optional key as a 4-byte tag plus 32 bytes (even when `None`). The `put_option`/`get_option` pair keeps that detail in one place and rejects tags other than 0 and 1.
- **Errors are enums, mapped at the edge.** `TokenError` and `AnchorError` are specific; `From` converts them to the runtime's `Error` at the boundary (Lesson 26). The cost is some loss of detail on the way out; the tests check the mapping table.
- **Borrowing in CPI code.** Building an `Instruction` needs keys copied out of `inv` first (`let (a, b) = (inv.key(0)?, inv.key(1)?)`), because `inv.invoke` borrows `inv` mutably; the escrow code does this everywhere, and it is the usual shape of CPI code in the real SDK too.
- **Seeds as `Vec<Vec<u8>>` then `&[&[u8]]`.** The seed list of the offer includes a temporary (the id's bytes), so the code owns the bytes in a `Vec` and borrows slices from it for the call.

---

## 8. Common mistakes

1. Checking the owner of an account but not its discriminator, or the reverse (§2.1).
2. Accepting a token account without checking its **mint** and its **owner field** (§2.3, §3.1).
3. Accepting the token program, the system program or any CPI target from the client without checking its address (§5.4).
4. Using `Transfer` where `TransferChecked` is possible, and trusting decimals from the client (§3.4).
5. Not checking that the vault is the account you think it is (the associated address of the PDA), and that it has no delegate or close authority (§5.2, §5.5).
6. `init` on an account that already holds data: re-initialization overwriting an admin or an offer (§5.2, the debugging challenge).
7. Forgetting that two slots can hold the same account (§3.4, §2.3).
8. Closing an account by transferring lamports out and leaving the data and owner (Lesson 53 §7.3; §5.3).
9. Trusting a bump or an address from the client instead of re-deriving, or storing the bump and not checking it on later calls (§2.3, Lesson 53 §5.3).
10. Writing logic before validation, so a half-validated state is visible to later code (§5.2).
11. Treating `UncheckedAccount` or a raw `AccountInfo` as "just a key" without writing the checks yourself (§2.2).
12. Validating accounts and then trusting a **number** from the instruction data or from an account (amounts, fees, decimals) without arithmetic checks (§2.4).
13. Assuming every token behaves like a plain token: fees, hooks and freezable accounts (§6).
14. Using the same signer PDA seeds for two purposes (§5.1; add a type label such as `b"offer"`).
15. Copying the layout constants (offsets, lengths) into several places instead of one definition (§5.1).
16. Believing that a program written with a framework is secure because the framework exists (§9).

---

## 9. Security notes and honest limits

- `anchorlab` is a **model written from memory**. It does not use the real `anchor-lang` or `spl-token` crates, and I did not compare its behavior with them. Its error codes, instruction tags, layouts and addresses are my recollection, labeled where they appear; verify against the real sources before relying on any.
- **Frameworks move the boundary; they do not remove it.** A constraint that is not declared is not checked, and `has_one` proves only that two keys are equal, not that the other account is the right *kind* of thing. Read the generated code (or the expansion) of any program you audit.
- **Every account is attacker-chosen**, including the ones that "must" be a program. The only identities you can trust are keys you compare against constants, PDAs you re-derive, and owners and discriminators you check.
- **Signing authority is the real secret.** A PDA signs only for the program that derived it, which is why an escrow's vault is safe from outsiders, and also why a bug *inside* the program is the only way to lose the funds. The program is the custodian; its upgrade authority (Lesson 53 §11) is the real custodian's key.
- **Arithmetic and rounding are money.** The partial-fill price rounds against the taker (exercise 8) and a fee rounds up (exercise 7); a rounding rule chosen carelessly is a systematic leak that a trader can repeat.
- **Passing the mutation tests is evidence about the tests** (§13), not about whether the model matches the real programs, and not about security.
- **Reentrancy and ordering.** The runtime forbids re-entering a program through another (Lesson 53 §6.3), which removes a class of attacks common elsewhere, but ordering inside one instruction still matters: move the tokens, then close, then refund, and re-read after a CPI anything the callee could have changed.

---

## 10. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Write `classify(data)`: is this token-program account data a mint, a token account, an uninitialized account or garbage? Why is the length not enough?
2. **(Easy)** Write `ui_amount(amount, decimals)`: the display string of a token amount without floating point (`1_500_000` with 6 decimals is `"1.5"`).
3. **(Easy)** Write `spendable_by(account, who)`: how much `who` may move out of a token account, given owner, delegate, allowance and state.
4. **(Medium)** Write `matches(data, mint, owner)`: the `token::mint` and `token::authority` checks on raw bytes, without ever panicking.
5. **(Medium)** Write `check_supply(supply, balances)`: does the sum of balances equal the supply? What can go wrong in the sum?
6. **(Medium-hard)** Write `first_prefix_collision(names, prefix_len)` using the crate's instruction discriminator, and use it to show why one byte is not enough.
7. **(Hard)** Write `transfer_fee(amount, bps, max_fee)` (a fee in basis points, rounded up, capped) and `net`, without overflow for any `u64`.
8. **(Hard)** Write `partial_fill(total_a, total_b, take_a)`: the price of a partial fill of an escrow offer, rounded so the maker is never short-changed.

<details markdown="1">
<summary>Answers</summary>

1. A possible answer (✓ compiled and run in a scratch crate with tests):

   ```rust
   //! Exercise 1: what kind of token-program account is this?
   use anchorlab::token::{ACCOUNT_LEN, MINT_LEN, Mint, State, TokenAccount};

   /// What the bytes of a token-program-owned account hold.
   #[derive(Debug, PartialEq, Eq)]
   pub enum Kind {
       /// An initialized mint.
       Mint,
       /// An initialized (or frozen) token account.
       TokenAccount,
       /// Right size, never initialized.
       Uninitialized,
       /// Anything else: wrong size or invalid fields.
       Unknown,
   }

   /// Classifies by exact length *and* by content; length alone is not enough.
   pub fn classify(data: &[u8]) -> Kind {
       match data.len() {
           MINT_LEN => match Mint::unpack(data) {
               Ok(m) if m.is_initialized => Kind::Mint,
               Ok(_) => Kind::Uninitialized,
               Err(_) => Kind::Unknown,
           },
           ACCOUNT_LEN => match TokenAccount::unpack(data) {
               Ok(t) if t.state == State::Uninitialized => Kind::Uninitialized,
               Ok(_) => Kind::TokenAccount,
               Err(_) => Kind::Unknown,
           },
           _ => Kind::Unknown,
       }
   }
   ```

   82 bytes with the initialized flag is a mint, but 82 zero bytes is an *uninitialized* mint, and 82 bytes with the flag byte set to 7 is garbage that happens to have the right length. Both layouts have a "state" byte whose valid values are few, so use the strict `unpack` and then look at the state. The test also tries lengths one off on each side.
2. A possible answer (✓):

   ```rust
   //! Exercise 2: show a token amount without floating point.

   /// `1_500_000` with 6 decimals is `"1.5"`; `1` is `"0.000001"`; `0` is `"0"`.
   pub fn ui_amount(amount: u64, decimals: u8) -> String {
       let d = usize::from(decimals);
       let digits = amount.to_string();
       if d == 0 {
           return digits;
       }
       let padded = format!("{digits:0>width$}", width = d + 1);
       let (int, frac) = padded.split_at(padded.len() - d);
       let frac = frac.trim_end_matches('0');
       if frac.is_empty() {
           int.to_string()
       } else {
           format!("{int}.{frac}")
       }
   }
   ```

   Pad the digit string with zeros up to `decimals + 1` characters, split off the last `decimals` digits as the fraction, and trim trailing zeros (and the dot if nothing is left). The tests include `1 → 0.000001`, `105` with 2 decimals `→ 1.05`, `u64::MAX` with 9 decimals `→ 18446744073.709551615`, and 20 decimals (more than a `u64` has digits).
3. A possible answer (✓):

   ```rust
   //! Exercise 3: how much may `who` move out of this token account?
   use anchorlab::token::{State, TokenAccount};
   use solanalab::pubkey::Pubkey;

   /// The owner may move the whole balance; a delegate at most its remaining
   /// allowance (and never more than the balance); nobody may move anything from
   /// a frozen or uninitialized account.
   pub fn spendable_by(acct: &TokenAccount, who: &Pubkey) -> u64 {
       if acct.state != State::Initialized {
           return 0;
       }
       if acct.owner == *who {
           acct.amount
       } else if acct.delegate == Some(*who) {
           acct.amount.min(acct.delegated_amount)
       } else {
           0
       }
   }
   ```

   A frozen or uninitialized account yields 0 for everybody, the owner gets the balance, the delegate `min(balance, allowance)` (the allowance can exceed the balance; the test uses 500 against 100).
4. A possible answer (✓):

   ```rust
   //! Exercise 4: `token::mint` and `token::authority` on raw bytes, without panics.
   use solanalab::pubkey::Pubkey;

   /// Why the bytes are not the expected token account.
   #[derive(Debug, PartialEq, Eq)]
   pub enum Mismatch {
       /// Not long enough to hold both keys.
       Short,
       /// The mint field differs.
       Mint,
       /// The owner field differs.
       Owner,
   }

   /// First 32 bytes = mint, next 32 = owner (the real layout's first 64 bytes).
   pub fn matches(data: &[u8], mint: &Pubkey, owner: &Pubkey) -> Result<(), Mismatch> {
       let head = data.get(..64).ok_or(Mismatch::Short)?;
       if head[..32] != mint.0 {
           return Err(Mismatch::Mint);
       }
       if head[32..] != owner.0 {
           return Err(Mismatch::Owner);
       }
       Ok(())
   }
   ```

   `data.get(..64)` turns a short slice into an error instead of a panic. The test includes swapped fields (the mint and owner positions are not interchangeable), a 63-byte slice and an empty slice.
5. A possible answer (✓):

   ```rust
   //! Exercise 5: does the supply equal the sum of the balances?

   /// Result of the check.
   #[derive(Debug, PartialEq, Eq)]
   pub struct Mismatch {
       /// What the mint says.
       pub supply: u64,
       /// What the accounts add up to (can exceed `u64`).
       pub sum: u128,
   }

   /// Sums in `u128` so the check cannot itself overflow.
   pub fn check_supply(supply: u64, balances: &[u64]) -> Result<(), Mismatch> {
       let sum: u128 = balances.iter().map(|&b| u128::from(b)).sum();
       if sum == u128::from(supply) {
           Ok(())
       } else {
           Err(Mismatch { supply, sum })
       }
   }
   ```

   Summing in `u64` can wrap and make a wrong total equal the supply by accident; the test has two accounts of `u64::MAX` whose true sum, `2 × (2⁶⁴ − 1)`, would wrap to `2⁶⁴ − 2`, which equals a supply of `u64::MAX − 1` only in the wrong arithmetic. In a real token program the invariant holds by construction (supply is checked on mint), but an off-chain auditor must not trust a sum that can wrap.
6. A possible answer (✓):

   ```rust
   //! Exercise 6: why discriminators are 8 bytes.
   use anchorlab::discriminator;

   /// Indexes `(i, j)`, `i < j`, of the first pair (in order of `j`) whose
   /// instruction discriminators agree on their first `prefix_len` bytes.
   pub fn first_prefix_collision(names: &[String], prefix_len: usize) -> Option<(usize, usize)> {
       let prefix_len = prefix_len.min(8);
       let discs: Vec<[u8; 8]> = names
           .iter()
           .map(|n| discriminator::instruction(n))
           .collect();
       for j in 1..discs.len() {
           for i in 0..j {
               if discs[i][..prefix_len] == discs[j][..prefix_len] {
                   return Some((i, j));
               }
           }
       }
       None
   }
   ```

   By pigeonhole, 257 names must contain two with the same first byte; the test finds such a pair among 400 names, checks that the full 8-byte discriminators differ, and finds none with all 8 bytes. With `prefix_len = 0` every pair "collides", so the answer is `(0, 1)`.
7. A possible answer (✓):

   ```rust
   //! Exercise 7: a transfer fee in basis points, rounded up and capped.

   /// `ceil(amount * bps / 10_000)`, at most `max_fee`. No overflow for any `u64`.
   pub fn transfer_fee(amount: u64, bps: u16, max_fee: u64) -> u64 {
       let raw = (u128::from(amount) * u128::from(bps)).div_ceil(10_000);
       u64::try_from(raw).unwrap_or(u64::MAX).min(max_fee)
   }

   /// What the receiver gets.
   pub fn net(amount: u64, bps: u16, max_fee: u64) -> u64 {
       amount - transfer_fee(amount, bps, max_fee).min(amount)
   }
   ```

   Compute in `u128` so `amount × bps` cannot overflow (`u64::MAX × 65_535` needs about 80 bits), round up with `div_ceil` so a tiny transfer still pays at least one unit when the rate is positive, then cap. The test checks that with `bps ≤ 10_000` the fee never exceeds the amount, that `10_001` at 50 bps gives 51, not 50, and that `u64::MAX` at the maximum rate does not overflow.
8. A possible answer (✓):

   ```rust
   //! Exercise 8: a partial fill of an escrow offer.

   /// The taker wants `take_a` of the `total_a` locked tokens. The price is
   /// proportional, rounded **up** so the maker is never short-changed:
   /// `ceil(total_b * take_a / total_a)`. `None` if the request is empty, larger
   /// than the offer, or the offer is empty.
   pub fn partial_fill(total_a: u64, total_b: u64, take_a: u64) -> Option<u64> {
       if total_a == 0 || take_a == 0 || take_a > total_a {
           return None;
       }
       let pay = (u128::from(total_b) * u128::from(take_a)).div_ceil(u128::from(total_a));
       u64::try_from(pay).ok()
   }
   ```

   The price is `ceil(total_b × take_a / total_a)` computed in `u128`. Rounding up means the taker never pays less than the proportional price: the test checks, for every `take_a` up to `total_a` over a grid of totals, that `pay × total_a ≥ total_b × take_a` and that the excess is less than `total_a`, i.e. less than one unit of token B per whole offer. The edge cases: an empty offer, a zero request and a request over the total return `None`.

</details>

---

## 11. Debugging challenge

A small treasury program keeps a `Pool` account (an admin key and a fee, 40 bytes) and `Ticket` accounts (a holder key and a credit, also 40 bytes; any user can mint one for themselves). The admin withdraws lamports from a program-owned **treasury** account with `payout(pool, admin, treasury, destination)`. Three reports arrive. (1) Someone who has never been the admin drains the treasury by passing **a ticket they minted** where the pool is expected. (2) Someone runs their own legitimate pool and drains a **stranger's** program-owned account by passing it as the treasury. (3) After the real admin set up the pool, an attacker calls the setup instruction again on the same pool account and becomes the admin. Explain each cause and fix the program.

```rust
//! Debugging challenge: a treasury program with three missing checks.
use solanalab::runtime::{Error, Invoke, Program};

/// Instruction data: `[0, admin:32]` init pool, `[1, holder:32]` mint ticket,
/// `[2, amount:u64 LE]` payout. Accounts of payout: pool, admin, treasury, destination.
/// The faulty program.
pub mod buggy {
    use super::*;

    /// Version 1.
    pub struct Treasury;

    fn write_key(inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        let me = inv.program_id();
        let acct = inv.account_mut(0)?;
        if acct.owner != me || acct.data.len() != 40 {
            return Err(Error::InvalidAccountData);
        }
        acct.data[..32].copy_from_slice(&data[1..33]);
        Ok(())
    }

    impl Program for Treasury {
        fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
            match data {
                [0, rest @ ..] | [1, rest @ ..] if rest.len() == 32 => write_key(inv, data),
                [2, rest @ ..] if rest.len() == 8 => {
                    let amount = u64::from_le_bytes(rest.try_into().expect("8"));
                    let me = inv.program_id();
                    let pool = inv.account(0)?;
                    if pool.owner != me || pool.data.len() != 40 {
                        return Err(Error::InvalidAccountData);
                    }
                    if !inv.is_signer(1)? || pool.data[..32] != inv.key(1)?.0 {
                        return Err(Error::WrongAuthority);
                    }
                    if inv.account(2)?.owner != me {
                        return Err(Error::WrongOwner(2));
                    }
                    let left = inv
                        .account(2)?
                        .lamports
                        .checked_sub(amount)
                        .ok_or(Error::InsufficientFunds)?;
                    let to = inv
                        .account(3)?
                        .lamports
                        .checked_add(amount)
                        .ok_or(Error::ArithmeticOverflow)?;
                    inv.account_mut(2)?.lamports = left;
                    inv.account_mut(3)?.lamports = to;
                    Ok(())
                }
                _ => Err(Error::InvalidInstructionData),
            }
        }
    }
}
```

<details markdown="1">
<summary>Solution</summary>

1. **Type cosplay.** The program checks that the "pool" is owned by itself and is 40 bytes long, but a ticket is *also* owned by the program and is *also* 40 bytes, and its first 32 bytes are the holder's key, exactly where `payout` looks for the admin (✓ `buggy_type_cosplay_a_ticket_passes_as_a_pool`: Eve mints a ticket for herself and withdraws 40,000,000 lamports). Fix: give each type an 8-byte discriminator written at creation and checked on every use (`Check::Typed`), so a ticket is refused as a pool (✓ `fixed_refuses_a_ticket_as_a_pool`).
2. **The treasury is not identified.** The program checks only that the treasury is program-owned, and the program owns every pool, ticket and stranger's account. Because the runtime allows the *owner* to debit its own accounts, "owned by this program" means "this program will debit it for whoever passes the authorization checks" (✓ `buggy_any_program_owned_account_can_be_named_as_the_treasury`). Fix: the treasury is a **PDA** of the program (`seeds = [b"treasury"]`), re-derived on every call (`Check::Seeds`), so there is exactly one valid treasury address (✓ `fixed_refuses_other_accounts_as_the_treasury`, `InvalidSeeds`). The fixed program also refuses to leave the treasury with a balance below the rent minimum unless it empties it (✓ `fixed_honest_flow_works_and_keeps_the_treasury_alive`).
3. **No `zero` check on initialization.** `init_pool` writes the admin into any program-owned 40-byte account, including one that is already in use (✓ `buggy_reinitialization_replaces_the_admin`). Fix: initialize only an account whose data is all zeros (Anchor's `init` and `zero` constraints do this), and write the discriminator at the same time (✓ `fixed_refuses_reinitialization`, `AlreadyInUse`; the attacker is then also refused as an admin, `WrongAuthority`).

Also in the fixed version: the admin is checked three ways, as the signer (`Signer`), as the key stored in the pool (`HasOne`) and implicitly as the account the pool names; and the instruction validates all accounts before it moves any lamport.

Fixed version (✓ compiled; seven tests for it and four that demonstrate faults of the buggy one):

```rust
/// The corrected program.
pub mod fixed {
    use super::*;
    use anchorlab::discriminator;
    use anchorlab::validate::{Bump, Check, Seed, spec, validate};
    use solanalab::account::rent_exempt_minimum;
    use solanalab::pubkey::Pubkey;

    /// Size of a pool or ticket: discriminator, key, number.
    pub const LEN: usize = 48;

    /// Version 2.
    pub struct Treasury;

    /// Writes `disc ‖ key` into a zeroed account owned by this program.
    fn init(inv: &mut Invoke<'_>, kind: &str, key: &[u8]) -> Result<(), Error> {
        let me = inv.program_id();
        let acct = inv.account(0)?;
        if acct.owner != me || acct.data.len() != LEN {
            return Err(Error::InvalidAccountData);
        }
        if acct.data.iter().any(|&b| b != 0) {
            return Err(Error::AlreadyInUse);
        }
        let acct = inv.account_mut(0)?;
        acct.data[..8].copy_from_slice(&discriminator::account(kind));
        acct.data[8..40].copy_from_slice(key);
        Ok(())
    }

    impl Program for Treasury {
        fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
            let me = inv.program_id();
            match data {
                [0, rest @ ..] if rest.len() == 32 => init(inv, "Pool", rest),
                [1, rest @ ..] if rest.len() == 32 => init(inv, "Ticket", rest),
                [2, rest @ ..] if rest.len() == 8 => {
                    let amount = u64::from_le_bytes(rest.try_into().expect("8"));
                    validate(
                        inv,
                        &[
                            spec(
                                "pool",
                                vec![
                                    Check::Typed {
                                        owner: me,
                                        disc: discriminator::account("Pool"),
                                        len: LEN,
                                    },
                                    Check::HasOne {
                                        offset: 8,
                                        target: 1,
                                    },
                                ],
                            ),
                            spec("admin", vec![Check::Signer]),
                            spec(
                                "treasury",
                                vec![
                                    Check::Mut,
                                    Check::Owner(me),
                                    Check::Seeds {
                                        seeds: vec![Seed::Bytes(b"treasury".to_vec())],
                                        bump: Bump::Canonical,
                                        program: me,
                                    },
                                ],
                            ),
                            spec("destination", vec![Check::Mut]),
                        ],
                    )?;
                    let treasury = inv.account(2)?;
                    let left = treasury
                        .lamports
                        .checked_sub(amount)
                        .filter(|&l| l == 0 || l >= rent_exempt_minimum(treasury.data.len()))
                        .ok_or(Error::InsufficientFunds)?;
                    let to = inv
                        .account(3)?
                        .lamports
                        .checked_add(amount)
                        .ok_or(Error::ArithmeticOverflow)?;
                    inv.account_mut(2)?.lamports = left;
                    inv.account_mut(3)?.lamports = to;
                    Ok(())
                }
                _ => Err(Error::InvalidInstructionData),
            }
        }
    }

    /// The treasury PDA of `program`.
    pub fn treasury_address(program: &Pubkey) -> Pubkey {
        solanalab::pubkey::find_program_address(&[b"treasury"], program)
            .expect("a bump exists")
            .0
    }
}
```

Two notes: the fixed program uses the crate's own `validate` so the trust boundary is one list that a reviewer can read; and `init` here only writes a key, whereas a real program would also create the account through the system program (as `make_offer` does), which brings the prefunded-address caveat of Lesson 53 §7.2 with it.

</details>

---

## 12. Interview questions

1. *What does Anchor do for you, and what does it not do?*
2. *What is an account discriminator? What attack does it prevent?*
3. *Name the checks you want on an account that holds a user's token balance.*
4. *What is the difference between the owner of a token account (as an account) and its owner field?*
5. *What is an associated token account and how is its address derived?*
6. *Why does an escrow's vault belong to a PDA? How does the program move tokens out of it?*
7. *What is `TransferChecked` for?*
8. *What is a delegate on a token account? What risk does it add for a program that holds tokens?*
9. *What can go wrong if two accounts passed to an instruction are the same account?*
10. *How do you prevent re-initialization of an account?*
11. *What is wrong with trusting a bump supplied by the client?*
12. *How do you close an account safely?*
13. *Why are fees and rounding security topics?*
14. *How would you test a program's account validation?*
15. *What changes if a token is a Token-2022 token?*

<details markdown="1">
<summary>Answers</summary>

1. It generates account-deserialization and constraint-checking code from a declaration: discriminator and owner (`Account<T>`), signer, mutability, `has_one`, seeds and bump, token constraints, account creation (`init`) and closing (`close`), plus an IDL for clients. It does not check your business logic, amounts or arithmetic, and any constraint you omit is not checked (§2.2, §2.4).
2. The first 8 bytes of an account, derived from the type name by hashing. It distinguishes account types that have the same owner and the same size so that a ticket cannot be passed as a pool (§2.1, §11).
3. Owned by the token program; exact length and an initialized state; the right **mint**; the right **owner field**; writable if it will change; and, in a program that holds the funds, that it is the expected address (associated or PDA) and has no unexpected delegate or close authority (§2.3, §5.5).
4. The account's owner is the program that may change its bytes (the token program); the owner *field* inside the data is the wallet or PDA that controls the balance (§3.1).
5. The canonical token account of a wallet for a mint, at the PDA of `[wallet, token program, mint]` under the associated-token-account program, created by a small program that makes the account and initializes it (§4).
6. So that no private key can move the tokens: the PDA is the vault's owner field, and only the escrow program can make the runtime accept a signature for it, by passing the seeds to a CPI (`invoke_signed`) (§5.1, §5.3, Lesson 53 §6.1).
7. It carries the mint and the decimals the caller believes, so that a wrong-decimals or wrong-mint mistake is an error instead of a transfer of a different value (§3.4).
8. A delegate may spend up to an allowance without the owner signing. If an attacker is a delegate of a vault, they can spend from it without the program's authorization; a program should refuse vaults with a delegate (or a close authority) it did not set (§3.3, §5.5).
9. A program that debits one slot and credits another may debit and credit the same account, double-count a balance, or lose a write; use explicit inequality checks or a duplicate-mutable-account check (§2.3, §3.4).
10. Initialize only an account that is empty or all zeros (a discriminator of zeros), or create it in the same instruction that writes it; refuse an account that already has a discriminator (§5.2, §11).
11. A client can supply a non-canonical bump and thereby a second valid address for the same logical key. Re-derive the canonical bump, or store it at creation and check the stored value on later calls (§2.3, Lesson 53 §5.3).
12. Move the lamports to the destination, zero the data (or write a closed-account discriminator), and reset the owner to the system program, so that nothing in the same transaction can revive it (§5.3, Lesson 53 §7.3).
13. A fee that rounds down to zero lets a trader avoid it by splitting transfers; a price that rounds in the taker's favor leaks value on every fill; sums that overflow can make an invariant check pass wrongly (exercises 5, 7, 8).
14. For each constraint write one test that breaks only that constraint and assert the exact error and that no state changed; add random-operation tests that assert invariants (supply conservation, no state change on failure); and test the aliasing and wrong-owner cases (§2.3, §3.5, §5.4).
15. The account layout has extensions after the base fields, transfers can charge fees (the vault receives less than was sent) and call hook programs, and some tokens can be non-transferable or have confidential balances; a program must decide which extensions it supports and measure received amounts, not assume them (§6).

</details>

---

## 13. Practical challenge: `anchorlab`

Build a library crate, depending on the Lesson 53 crate, that:

1. Provides 8-byte **discriminators** for accounts and instructions.
2. Provides a **constraint validator** (`validate`) with signer, mut, address, program, owner, typed-account, `has_one`, seeds-and-bump (searched or stored, with seeds from constants, other accounts and the account's own data), token mint and authority, uninitialized and duplicate-writable checks, and error codes.
3. Provides a **token program** with the real 82- and 165-byte layouts, strict decoders, pure rules and the instructions initialize mint/account, transfer, approve, revoke, mint, burn, close, freeze, thaw and transfer-checked.
4. Provides **associated token accounts**: derivation, creation by CPI (system program then token program), idempotent creation and prefunded-address handling.
5. Provides an **escrow** (make, take, cancel) that validates every account before acting, locks tokens in a PDA-owned vault, signs token CPIs with the offer's seeds and closes everything on completion.
6. Has a benchmark example, passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

The path assumes the Lesson 53 crate sits at `../../p53/solanalab` in my scratch directory; point it at wherever you keep that crate.

```toml
[package]
name = "anchorlab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "Teaching model of Anchor-style account validation, an SPL-token-like program, associated token accounts and an escrow. Built on the Lesson 53 crate. Not audited."

[dependencies]
solanalab = { path = "../../p53/solanalab" }
sha2 = "0.11"

[profile.dev.package."*"]
opt-level = 3
```

### `src/lib.rs`

```rust
//! Anchor-style account validation, an SPL-token-like program, associated
//! token accounts and an escrow, on top of the `solanalab` runtime model.
//! **A teaching model written from memory of the real programs; not audited;
//! not for production.**

pub mod ata;
pub mod discriminator;
pub mod escrow;
pub mod token;
pub mod validate;
```

### `src/discriminator.rs`

```rust
//! Eight-byte discriminators: the first bytes of a SHA-256 hash of a
//! namespaced name (from memory of how Anchor derives them).

use sha2::{Digest, Sha256};

fn disc(namespace: &str, name: &str) -> [u8; 8] {
    let mut h = Sha256::new();
    h.update(namespace.as_bytes());
    h.update(b":");
    h.update(name.as_bytes());
    let full = h.finalize();
    full[..8].try_into().expect("8 bytes")
}

/// Discriminator stored at the start of an account of type `name`
/// (`sha256("account:<Name>")[..8]`).
pub fn account(name: &str) -> [u8; 8] {
    disc("account", name)
}

/// Discriminator at the start of the data of instruction `name`
/// (`sha256("global:<snake_case_name>")[..8]`).
pub fn instruction(name: &str) -> [u8; 8] {
    disc("global", name)
}
```

### `src/validate.rs`

```rust
//! Declarative account validation in the style of Anchor's `#[derive(Accounts)]`:
//! each account has a name and a list of [`Check`]s, and [`validate`] runs them
//! all before any instruction logic touches the data.
//!
//! The error codes follow the numbering I remember from Anchor's `ErrorCode`
//! (constraint errors 2000s, account errors 3000s); treat them as illustrative.

use solanalab::pubkey::{Pubkey, create_program_address, find_program_address};
use solanalab::runtime::{Error, Invoke};

/// Where a PDA seed comes from.
#[derive(Debug, Clone)]
pub enum Seed {
    /// Fixed bytes.
    Bytes(Vec<u8>),
    /// The address of another account of the instruction (by index).
    Key(usize),
    /// Bytes of this same account's data (`offset`, `len`).
    OwnData(usize, usize),
}

/// Where the bump comes from.
#[derive(Debug, Clone, Copy)]
pub enum Bump {
    /// Search for it (the canonical bump).
    Canonical,
    /// Read one byte stored in this account's data at `offset`.
    Stored(usize),
}

/// One constraint on one account.
#[derive(Debug, Clone)]
pub enum Check {
    /// The account signed (`#[account(signer)]`).
    Signer,
    /// The account is writable (`#[account(mut)]`).
    Mut,
    /// The account is exactly this address (`address = …`).
    Address(Pubkey),
    /// The account is this executable program (`Program<'info, T>`).
    Program(Pubkey),
    /// The account is owned by this program.
    Owner(Pubkey),
    /// An account of a program-defined type: owner, exact length, discriminator
    /// (`Account<'info, T>`).
    Typed {
        /// Required owner.
        owner: Pubkey,
        /// Required first eight bytes.
        disc: [u8; 8],
        /// Required data length.
        len: usize,
    },
    /// 32 bytes at `offset` in this account's data equal the address of account
    /// `target` (`has_one = …`).
    HasOne {
        /// Offset of the stored key.
        offset: usize,
        /// Index of the account it must equal.
        target: usize,
    },
    /// This account is the PDA derived from `seeds` under `program`
    /// (`seeds = [...], bump`).
    Seeds {
        /// The seeds, in order.
        seeds: Vec<Seed>,
        /// Where the bump comes from.
        bump: Bump,
        /// The deriving program.
        program: Pubkey,
    },
    /// This token account's mint is account `target` (`token::mint`).
    TokenMint(usize),
    /// This token account's owner is account `target` (`token::authority`).
    TokenOwner(usize),
    /// The account is unused: empty data, system-owned (the precondition of `init`).
    Uninit,
}

/// The checks for one account.
#[derive(Debug, Clone)]
pub struct Spec {
    /// Field name, for error messages.
    pub name: &'static str,
    /// Constraints.
    pub checks: Vec<Check>,
}

/// Convenience constructor.
pub fn spec(name: &'static str, checks: Vec<Check>) -> Spec {
    Spec { name, checks }
}

/// A failed constraint.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct AnchorError {
    /// Numeric code (illustrative, see the module docs).
    pub code: u32,
    /// Error name.
    pub name: &'static str,
    /// Index of the account.
    pub account: usize,
    /// Field name of the account.
    pub field: &'static str,
}

impl From<AnchorError> for Error {
    fn from(e: AnchorError) -> Error {
        match e.name {
            "ConstraintMut" => Error::NotWritable(e.account),
            "ConstraintSigner" => Error::NotSigner(e.account),
            "ConstraintOwner" | "AccountOwnedByWrongProgram" => Error::WrongOwner(e.account),
            "ConstraintHasOne" => Error::WrongAuthority,
            "ConstraintSeeds" => Error::InvalidSeeds,
            "AccountNotEnoughKeys" => Error::MissingAccount(e.account),
            "ConstraintInit" => Error::AlreadyInUse,
            _ => Error::InvalidAccountData,
        }
    }
}

fn err(code: u32, name: &'static str, account: usize, field: &'static str) -> AnchorError {
    AnchorError {
        code,
        name,
        account,
        field,
    }
}

/// Runs every constraint of every account. Returns the first failure, in
/// account order and check order. Also rejects the same writable account
/// appearing twice (`ConstraintDuplicateMutableAccount`).
pub fn validate(inv: &Invoke<'_>, specs: &[Spec]) -> Result<(), AnchorError> {
    if inv.len() < specs.len() {
        return Err(err(3005, "AccountNotEnoughKeys", inv.len(), "-"));
    }
    for (i, s) in specs.iter().enumerate() {
        let key = inv.key(i).expect("length checked");
        let acct = inv.account(i).expect("length checked");
        let f = s.name;
        let mut is_mut = false;
        for c in &s.checks {
            match c {
                Check::Signer => {
                    if !inv.is_signer(i).expect("i") {
                        return Err(err(2002, "ConstraintSigner", i, f));
                    }
                }
                Check::Mut => {
                    is_mut = true;
                    if !inv.is_writable(i).expect("i") {
                        return Err(err(2000, "ConstraintMut", i, f));
                    }
                }
                Check::Address(a) => {
                    if key != *a {
                        return Err(err(2012, "ConstraintAddress", i, f));
                    }
                }
                Check::Program(p) => {
                    if key != *p {
                        return Err(err(3008, "InvalidProgramId", i, f));
                    }
                    if !acct.executable {
                        return Err(err(3009, "InvalidProgramExecutable", i, f));
                    }
                }
                Check::Owner(o) => {
                    if acct.owner != *o {
                        return Err(err(2004, "ConstraintOwner", i, f));
                    }
                }
                Check::Typed { owner, disc, len } => {
                    if acct.owner != *owner {
                        return Err(err(3007, "AccountOwnedByWrongProgram", i, f));
                    }
                    if acct.data.len() < 8 {
                        return Err(err(3001, "AccountDiscriminatorNotFound", i, f));
                    }
                    if acct.data[..8] != disc[..] {
                        return Err(err(3002, "AccountDiscriminatorMismatch", i, f));
                    }
                    if acct.data.len() != *len {
                        return Err(err(3003, "AccountDidNotDeserialize", i, f));
                    }
                }
                Check::HasOne { offset, target } => {
                    let stored = acct.data.get(*offset..*offset + 32);
                    let want = inv
                        .key(*target)
                        .map_err(|_| err(3005, "AccountNotEnoughKeys", *target, f))?;
                    if stored != Some(&want.0[..]) {
                        return Err(err(2001, "ConstraintHasOne", i, f));
                    }
                }
                Check::Seeds {
                    seeds,
                    bump,
                    program,
                } => {
                    let mut parts: Vec<Vec<u8>> = Vec::new();
                    for s in seeds {
                        parts.push(match s {
                            Seed::Bytes(b) => b.clone(),
                            Seed::Key(j) => inv
                                .key(*j)
                                .map_err(|_| err(3005, "AccountNotEnoughKeys", *j, f))?
                                .0
                                .to_vec(),
                            Seed::OwnData(o, l) => acct
                                .data
                                .get(*o..*o + *l)
                                .ok_or_else(|| err(2006, "ConstraintSeeds", i, f))?
                                .to_vec(),
                        });
                    }
                    let refs: Vec<&[u8]> = parts.iter().map(Vec::as_slice).collect();
                    let derived = match bump {
                        Bump::Canonical => find_program_address(&refs, program).ok().map(|x| x.0),
                        Bump::Stored(o) => {
                            let b = acct.data.get(*o).copied();
                            b.and_then(|b| {
                                let bs = [b];
                                let mut with: Vec<&[u8]> = refs.clone();
                                with.push(&bs);
                                create_program_address(&with, program).ok()
                            })
                        }
                    };
                    if derived != Some(key) {
                        return Err(err(2006, "ConstraintSeeds", i, f));
                    }
                }
                Check::TokenMint(j) => {
                    let want = inv
                        .key(*j)
                        .map_err(|_| err(3005, "AccountNotEnoughKeys", *j, f))?;
                    if acct.data.get(0..32) != Some(&want.0[..]) {
                        return Err(err(2014, "ConstraintTokenMint", i, f));
                    }
                }
                Check::TokenOwner(j) => {
                    let want = inv
                        .key(*j)
                        .map_err(|_| err(3005, "AccountNotEnoughKeys", *j, f))?;
                    if acct.data.get(32..64) != Some(&want.0[..]) {
                        return Err(err(2015, "ConstraintTokenOwner", i, f));
                    }
                }
                Check::Uninit => {
                    if !acct.data.is_empty() || acct.owner != Pubkey::default() {
                        return Err(err(2010, "ConstraintInit", i, f));
                    }
                }
            }
        }
        if is_mut {
            for (j, other) in specs.iter().enumerate().take(i) {
                let other_mut = other.checks.iter().any(|c| matches!(c, Check::Mut));
                if other_mut && inv.key(j).expect("j") == key {
                    return Err(err(2040, "ConstraintDuplicateMutableAccount", i, f));
                }
            }
        }
    }
    Ok(())
}
```

### `src/token.rs`

```rust
//! A model of the SPL token program: mints and token accounts with the real
//! byte layouts (from memory), plus the instructions that move, mint, burn,
//! delegate, freeze and close.
//!
//! The rules live in small pure functions on [`Mint`] and [`TokenAccount`]
//! that return [`TokenError`]; [`TokenProgram`] only parses accounts and data.

use solanalab::account::Account;
use solanalab::instruction::{AccountMeta, Instruction};
use solanalab::pubkey::Pubkey;
use solanalab::runtime::{Error, Invoke, Program};

/// The token program's address (`Tokenkeg…`, from memory).
pub const TOKEN_PROGRAM_ID: Pubkey = Pubkey([
    0x06, 0xdd, 0xf6, 0xe1, 0xd7, 0x65, 0xa1, 0x93, 0xd9, 0xcb, 0xe1, 0x46, 0xce, 0xeb, 0x79, 0xac,
    0x1c, 0xb4, 0x85, 0xed, 0x5f, 0x5b, 0x37, 0x91, 0x3a, 0x8c, 0xf5, 0x85, 0x7e, 0xff, 0x00, 0xa9,
]);
/// Size of a mint account.
pub const MINT_LEN: usize = 82;
/// Size of a token account.
pub const ACCOUNT_LEN: usize = 165;
/// Compute units charged per token instruction (model value).
pub const COST: u64 = 300;

/// Rule violations of the token program.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TokenError {
    /// Balance or delegated allowance too small.
    InsufficientFunds,
    /// The accounts belong to different mints.
    MintMismatch,
    /// The authority is neither the owner nor an allowed delegate.
    OwnerMismatch,
    /// A frozen account cannot send or receive.
    AccountFrozen,
    /// Initializing something that is already initialized.
    AlreadyInUse,
    /// Using something that was never initialized.
    Uninitialized,
    /// `TransferChecked` decimals differ from the mint's.
    DecimalsMismatch,
    /// Closing an account that still holds tokens.
    NonZeroBalance,
    /// The mint has no freeze authority.
    NoFreezeAuthority,
    /// Checked arithmetic failed.
    Overflow,
}

impl From<TokenError> for Error {
    fn from(e: TokenError) -> Error {
        match e {
            TokenError::InsufficientFunds => Error::InsufficientFunds,
            TokenError::OwnerMismatch => Error::WrongAuthority,
            TokenError::AlreadyInUse => Error::AlreadyInUse,
            TokenError::Overflow => Error::ArithmeticOverflow,
            _ => Error::InvalidAccountData,
        }
    }
}

fn put_option(out: &mut Vec<u8>, k: &Option<Pubkey>) {
    match k {
        Some(k) => {
            out.extend_from_slice(&1u32.to_le_bytes());
            out.extend_from_slice(&k.0);
        }
        None => {
            out.extend_from_slice(&0u32.to_le_bytes());
            out.extend_from_slice(&[0; 32]);
        }
    }
}

fn get_option(d: &[u8]) -> Result<Option<Pubkey>, Error> {
    let tag = u32::from_le_bytes(d[..4].try_into().expect("4"));
    let key = Pubkey(d[4..36].try_into().expect("32"));
    match tag {
        0 => Ok(None),
        1 => Ok(Some(key)),
        _ => Err(Error::InvalidAccountData),
    }
}

/// A token mint: who may create tokens, how many exist, how they are shown.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Mint {
    /// May mint more tokens; `None` fixes the supply for ever.
    pub mint_authority: Option<Pubkey>,
    /// Total tokens in existence (smallest units).
    pub supply: u64,
    /// Decimal places for display.
    pub decimals: u8,
    /// Set by `InitializeMint`.
    pub is_initialized: bool,
    /// May freeze and thaw token accounts of this mint.
    pub freeze_authority: Option<Pubkey>,
}

impl Mint {
    /// Layout: authority (4+32), supply (8), decimals (1), initialized (1), freeze (4+32).
    pub fn pack(&self) -> Vec<u8> {
        let mut v = Vec::with_capacity(MINT_LEN);
        put_option(&mut v, &self.mint_authority);
        v.extend_from_slice(&self.supply.to_le_bytes());
        v.push(self.decimals);
        v.push(u8::from(self.is_initialized));
        put_option(&mut v, &self.freeze_authority);
        v
    }

    /// Strict parse: exact length, valid option tags and flags.
    pub fn unpack(d: &[u8]) -> Result<Mint, Error> {
        if d.len() != MINT_LEN {
            return Err(Error::InvalidAccountData);
        }
        let init = match d[45] {
            0 => false,
            1 => true,
            _ => return Err(Error::InvalidAccountData),
        };
        Ok(Mint {
            mint_authority: get_option(&d[0..36])?,
            supply: u64::from_le_bytes(d[36..44].try_into().expect("8")),
            decimals: d[44],
            is_initialized: init,
            freeze_authority: get_option(&d[46..82])?,
        })
    }

    /// Creates tokens in `dest` (which must belong to `mint_key`).
    pub fn mint_to(
        &mut self,
        mint_key: &Pubkey,
        dest: &mut TokenAccount,
        authority: &Pubkey,
        amount: u64,
    ) -> Result<(), TokenError> {
        if !self.is_initialized || dest.state == State::Uninitialized {
            return Err(TokenError::Uninitialized);
        }
        if dest.mint != *mint_key {
            return Err(TokenError::MintMismatch);
        }
        if dest.state == State::Frozen {
            return Err(TokenError::AccountFrozen);
        }
        if self.mint_authority != Some(*authority) {
            return Err(TokenError::OwnerMismatch);
        }
        let supply = self
            .supply
            .checked_add(amount)
            .ok_or(TokenError::Overflow)?;
        let balance = dest
            .amount
            .checked_add(amount)
            .ok_or(TokenError::Overflow)?;
        self.supply = supply;
        dest.amount = balance;
        Ok(())
    }

    /// Destroys tokens held in `account`.
    pub fn burn(
        &mut self,
        mint_key: &Pubkey,
        account: &mut TokenAccount,
        authority: &Pubkey,
        amount: u64,
    ) -> Result<(), TokenError> {
        if !self.is_initialized || account.state == State::Uninitialized {
            return Err(TokenError::Uninitialized);
        }
        if account.mint != *mint_key {
            return Err(TokenError::MintMismatch);
        }
        if account.state == State::Frozen {
            return Err(TokenError::AccountFrozen);
        }
        if account.amount < amount {
            return Err(TokenError::InsufficientFunds);
        }
        account.authorize(authority, amount)?;
        account.amount -= amount;
        self.supply -= amount; // supply >= every balance, by construction
        Ok(())
    }
}

/// The state of a token account.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum State {
    /// Freshly allocated, zeroed.
    Uninitialized,
    /// Usable.
    Initialized,
    /// Frozen by the mint's freeze authority.
    Frozen,
}

/// A holder's balance of one mint.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct TokenAccount {
    /// Which token.
    pub mint: Pubkey,
    /// Who controls the balance (not who *owns the account*: that is the token program).
    pub owner: Pubkey,
    /// Balance in smallest units.
    pub amount: u64,
    /// Someone allowed to spend up to `delegated_amount`.
    pub delegate: Option<Pubkey>,
    /// Lifecycle state.
    pub state: State,
    /// Wrapped-SOL marker (kept so the layout is complete; not used).
    pub is_native: Option<u64>,
    /// Remaining allowance of the delegate.
    pub delegated_amount: u64,
    /// May close the account instead of the owner.
    pub close_authority: Option<Pubkey>,
}

impl TokenAccount {
    /// A new, initialized, empty account.
    pub fn new(mint: Pubkey, owner: Pubkey) -> TokenAccount {
        TokenAccount {
            mint,
            owner,
            amount: 0,
            delegate: None,
            state: State::Initialized,
            is_native: None,
            delegated_amount: 0,
            close_authority: None,
        }
    }

    /// Layout (165 bytes): mint, owner, amount, delegate (4+32), state, native (4+8),
    /// delegated amount, close authority (4+32).
    pub fn pack(&self) -> Vec<u8> {
        let mut v = Vec::with_capacity(ACCOUNT_LEN);
        v.extend_from_slice(&self.mint.0);
        v.extend_from_slice(&self.owner.0);
        v.extend_from_slice(&self.amount.to_le_bytes());
        put_option(&mut v, &self.delegate);
        v.push(match self.state {
            State::Uninitialized => 0,
            State::Initialized => 1,
            State::Frozen => 2,
        });
        match self.is_native {
            Some(n) => {
                v.extend_from_slice(&1u32.to_le_bytes());
                v.extend_from_slice(&n.to_le_bytes());
            }
            None => {
                v.extend_from_slice(&0u32.to_le_bytes());
                v.extend_from_slice(&[0; 8]);
            }
        }
        v.extend_from_slice(&self.delegated_amount.to_le_bytes());
        put_option(&mut v, &self.close_authority);
        v
    }

    /// Strict parse.
    pub fn unpack(d: &[u8]) -> Result<TokenAccount, Error> {
        if d.len() != ACCOUNT_LEN {
            return Err(Error::InvalidAccountData);
        }
        let state = match d[108] {
            0 => State::Uninitialized,
            1 => State::Initialized,
            2 => State::Frozen,
            _ => return Err(Error::InvalidAccountData),
        };
        let native_tag = u32::from_le_bytes(d[109..113].try_into().expect("4"));
        let is_native = match native_tag {
            0 => None,
            1 => Some(u64::from_le_bytes(d[113..121].try_into().expect("8"))),
            _ => return Err(Error::InvalidAccountData),
        };
        Ok(TokenAccount {
            mint: Pubkey(d[0..32].try_into().expect("32")),
            owner: Pubkey(d[32..64].try_into().expect("32")),
            amount: u64::from_le_bytes(d[64..72].try_into().expect("8")),
            delegate: get_option(&d[72..108])?,
            state,
            is_native,
            delegated_amount: u64::from_le_bytes(d[121..129].try_into().expect("8")),
            close_authority: get_option(&d[129..165])?,
        })
    }

    /// May `authority` spend `amount`? The owner always may; the delegate may
    /// up to its allowance, which shrinks (and disappears at zero).
    pub fn authorize(&mut self, authority: &Pubkey, amount: u64) -> Result<(), TokenError> {
        if self.owner == *authority {
            return Ok(());
        }
        if self.delegate == Some(*authority) {
            if self.delegated_amount < amount {
                return Err(TokenError::InsufficientFunds);
            }
            self.delegated_amount -= amount;
            if self.delegated_amount == 0 {
                self.delegate = None;
            }
            return Ok(());
        }
        Err(TokenError::OwnerMismatch)
    }

    /// Moves `amount` to `dest`. `checked` carries `(mint key, mint decimals, decimals given)`
    /// for `TransferChecked`.
    pub fn transfer(
        &mut self,
        dest: &mut TokenAccount,
        authority: &Pubkey,
        amount: u64,
        checked: Option<(&Pubkey, u8, u8)>,
    ) -> Result<(), TokenError> {
        if self.state == State::Uninitialized || dest.state == State::Uninitialized {
            return Err(TokenError::Uninitialized);
        }
        if self.mint != dest.mint {
            return Err(TokenError::MintMismatch);
        }
        if let Some((mint_key, mint_decimals, given)) = checked {
            if self.mint != *mint_key {
                return Err(TokenError::MintMismatch);
            }
            if mint_decimals != given {
                return Err(TokenError::DecimalsMismatch);
            }
        }
        if self.state == State::Frozen || dest.state == State::Frozen {
            return Err(TokenError::AccountFrozen);
        }
        if self.amount < amount {
            return Err(TokenError::InsufficientFunds);
        }
        let new_dest = dest
            .amount
            .checked_add(amount)
            .ok_or(TokenError::Overflow)?;
        self.authorize(authority, amount)?;
        self.amount -= amount;
        dest.amount = new_dest;
        Ok(())
    }
}

// ---------- instruction builders ----------

fn ix(accounts: Vec<AccountMeta>, data: Vec<u8>) -> Instruction {
    Instruction {
        program_id: TOKEN_PROGRAM_ID,
        accounts,
        data,
    }
}

fn opt_bytes(k: &Option<Pubkey>) -> Vec<u8> {
    match k {
        Some(k) => [&[1u8][..], &k.0].concat(),
        None => vec![0],
    }
}

/// `InitializeMint` (tag 0): accounts `[mint]`.
pub fn initialize_mint(
    mint: Pubkey,
    decimals: u8,
    authority: Pubkey,
    freeze: Option<Pubkey>,
) -> Instruction {
    let mut d = vec![0, decimals];
    d.extend_from_slice(&authority.0);
    d.extend(opt_bytes(&freeze));
    ix(vec![AccountMeta::writable(mint, false)], d)
}

/// `InitializeAccount` (tag 1): accounts `[account, mint, owner]`.
pub fn initialize_account(account: Pubkey, mint: Pubkey, owner: Pubkey) -> Instruction {
    ix(
        vec![
            AccountMeta::writable(account, false),
            AccountMeta::readonly(mint, false),
            AccountMeta::readonly(owner, false),
        ],
        vec![1],
    )
}

fn amount_data(tag: u8, amount: u64) -> Vec<u8> {
    let mut d = vec![tag];
    d.extend_from_slice(&amount.to_le_bytes());
    d
}

/// `Transfer` (tag 3): accounts `[source, destination, authority]`.
pub fn transfer(source: Pubkey, dest: Pubkey, authority: Pubkey, amount: u64) -> Instruction {
    ix(
        vec![
            AccountMeta::writable(source, false),
            AccountMeta::writable(dest, false),
            AccountMeta::readonly(authority, true),
        ],
        amount_data(3, amount),
    )
}

/// `Approve` (tag 4): accounts `[source, delegate, owner]`.
pub fn approve(source: Pubkey, delegate: Pubkey, owner: Pubkey, amount: u64) -> Instruction {
    ix(
        vec![
            AccountMeta::writable(source, false),
            AccountMeta::readonly(delegate, false),
            AccountMeta::readonly(owner, true),
        ],
        amount_data(4, amount),
    )
}

/// `Revoke` (tag 5): accounts `[source, owner]`.
pub fn revoke(source: Pubkey, owner: Pubkey) -> Instruction {
    ix(
        vec![
            AccountMeta::writable(source, false),
            AccountMeta::readonly(owner, true),
        ],
        vec![5],
    )
}

/// `MintTo` (tag 7): accounts `[mint, destination, mint authority]`.
pub fn mint_to(mint: Pubkey, dest: Pubkey, authority: Pubkey, amount: u64) -> Instruction {
    ix(
        vec![
            AccountMeta::writable(mint, false),
            AccountMeta::writable(dest, false),
            AccountMeta::readonly(authority, true),
        ],
        amount_data(7, amount),
    )
}

/// `Burn` (tag 8): accounts `[account, mint, authority]`.
pub fn burn(account: Pubkey, mint: Pubkey, authority: Pubkey, amount: u64) -> Instruction {
    ix(
        vec![
            AccountMeta::writable(account, false),
            AccountMeta::writable(mint, false),
            AccountMeta::readonly(authority, true),
        ],
        amount_data(8, amount),
    )
}

/// `CloseAccount` (tag 9): accounts `[account, destination, owner]`.
pub fn close_account(account: Pubkey, dest: Pubkey, owner: Pubkey) -> Instruction {
    ix(
        vec![
            AccountMeta::writable(account, false),
            AccountMeta::writable(dest, false),
            AccountMeta::readonly(owner, true),
        ],
        vec![9],
    )
}

/// `FreezeAccount` (tag 10) / `ThawAccount` (tag 11): accounts `[account, mint, freeze authority]`.
pub fn freeze_account(
    account: Pubkey,
    mint: Pubkey,
    authority: Pubkey,
    freeze: bool,
) -> Instruction {
    ix(
        vec![
            AccountMeta::writable(account, false),
            AccountMeta::readonly(mint, false),
            AccountMeta::readonly(authority, true),
        ],
        vec![if freeze { 10 } else { 11 }],
    )
}

/// `TransferChecked` (tag 12): accounts `[source, mint, destination, authority]`.
pub fn transfer_checked(
    source: Pubkey,
    mint: Pubkey,
    dest: Pubkey,
    authority: Pubkey,
    amount: u64,
    decimals: u8,
) -> Instruction {
    let mut d = amount_data(12, amount);
    d.push(decimals);
    ix(
        vec![
            AccountMeta::writable(source, false),
            AccountMeta::readonly(mint, false),
            AccountMeta::writable(dest, false),
            AccountMeta::readonly(authority, true),
        ],
        d,
    )
}

// ---------- the program ----------

/// The token program.
pub struct TokenProgram;

fn need_signer(inv: &Invoke<'_>, i: usize) -> Result<(), Error> {
    if inv.is_signer(i)? {
        Ok(())
    } else {
        Err(Error::NotSigner(i))
    }
}

fn load_token(inv: &Invoke<'_>, i: usize) -> Result<TokenAccount, Error> {
    let a = inv.account(i)?;
    if a.owner != inv.program_id() {
        return Err(Error::WrongOwner(i));
    }
    TokenAccount::unpack(&a.data)
}

fn load_mint(inv: &Invoke<'_>, i: usize) -> Result<Mint, Error> {
    let a = inv.account(i)?;
    if a.owner != inv.program_id() {
        return Err(Error::WrongOwner(i));
    }
    Mint::unpack(&a.data)
}

fn store(inv: &mut Invoke<'_>, i: usize, data: Vec<u8>) -> Result<(), Error> {
    if !inv.is_writable(i)? {
        return Err(Error::NotWritable(i));
    }
    inv.account_mut(i)?.data = data;
    Ok(())
}

fn amount_arg(data: &[u8], extra: usize) -> Result<u64, Error> {
    if data.len() != 9 + extra {
        return Err(Error::InvalidInstructionData);
    }
    Ok(u64::from_le_bytes(data[1..9].try_into().expect("8")))
}

impl Program for TokenProgram {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(COST)?;
        let tag = *data.first().ok_or(Error::InvalidInstructionData)?;
        match tag {
            0 => {
                if data.len() != 1 + 1 + 32 + 1 + 32 && data.len() != 1 + 1 + 32 + 1 {
                    return Err(Error::InvalidInstructionData);
                }
                let freeze = match data[34] {
                    0 if data.len() == 35 => None,
                    1 if data.len() == 67 => Some(Pubkey(data[35..67].try_into().expect("32"))),
                    _ => return Err(Error::InvalidInstructionData),
                };
                let acct = inv.account(0)?;
                if acct.owner != inv.program_id() || acct.data.len() != MINT_LEN {
                    return Err(Error::InvalidAccountData);
                }
                if Mint::unpack(&acct.data)?.is_initialized {
                    return Err(TokenError::AlreadyInUse.into());
                }
                let mint = Mint {
                    mint_authority: Some(Pubkey(data[2..34].try_into().expect("32"))),
                    supply: 0,
                    decimals: data[1],
                    is_initialized: true,
                    freeze_authority: freeze,
                };
                store(inv, 0, mint.pack())
            }
            1 => {
                if data.len() != 1 {
                    return Err(Error::InvalidInstructionData);
                }
                let acct = inv.account(0)?;
                if acct.owner != inv.program_id() {
                    return Err(Error::WrongOwner(0));
                }
                if TokenAccount::unpack(&acct.data)?.state != State::Uninitialized {
                    return Err(TokenError::AlreadyInUse.into());
                }
                load_mint(inv, 1)
                    .ok()
                    .filter(|m| m.is_initialized)
                    .ok_or(Error::InvalidAccountData)?;
                let t = TokenAccount::new(inv.key(1)?, inv.key(2)?);
                store(inv, 0, t.pack())
            }
            3 | 12 => {
                let (s, d, a, mint_i) = if tag == 3 {
                    (0, 1, 2, None)
                } else {
                    (0, 2, 3, Some(1))
                };
                let amount = amount_arg(data, usize::from(tag == 12))?;
                need_signer(inv, a)?;
                let mut src = load_token(inv, s)?;
                let mut dst = load_token(inv, d)?;
                let authority = inv.key(a)?;
                let checked = match mint_i {
                    Some(m) => {
                        let mint = load_mint(inv, m)?;
                        Some((inv.key(m)?, mint.decimals, data[9]))
                    }
                    None => None,
                };
                let checked_ref = checked.as_ref().map(|(k, a, b)| (k, *a, *b));
                if inv.key(s)? == inv.key(d)? {
                    // all checks run, but nothing is written: moving tokens to
                    // oneself must not create or destroy any
                    src.transfer(&mut dst, &authority, amount, checked_ref)?;
                    return Ok(());
                }
                src.transfer(&mut dst, &authority, amount, checked_ref)?;
                store(inv, s, src.pack())?;
                store(inv, d, dst.pack())
            }
            4 => {
                let amount = amount_arg(data, 0)?;
                need_signer(inv, 2)?;
                let mut src = load_token(inv, 0)?;
                if src.owner != inv.key(2)? {
                    return Err(TokenError::OwnerMismatch.into());
                }
                if src.state == State::Frozen {
                    return Err(TokenError::AccountFrozen.into());
                }
                src.delegate = Some(inv.key(1)?);
                src.delegated_amount = amount;
                store(inv, 0, src.pack())
            }
            5 => {
                need_signer(inv, 1)?;
                let mut src = load_token(inv, 0)?;
                if src.owner != inv.key(1)? {
                    return Err(TokenError::OwnerMismatch.into());
                }
                src.delegate = None;
                src.delegated_amount = 0;
                store(inv, 0, src.pack())
            }
            7 => {
                let amount = amount_arg(data, 0)?;
                need_signer(inv, 2)?;
                let mut mint = load_mint(inv, 0)?;
                let mut dst = load_token(inv, 1)?;
                let (mk, auth) = (inv.key(0)?, inv.key(2)?);
                mint.mint_to(&mk, &mut dst, &auth, amount)?;
                store(inv, 0, mint.pack())?;
                store(inv, 1, dst.pack())
            }
            8 => {
                let amount = amount_arg(data, 0)?;
                need_signer(inv, 2)?;
                let mut acct = load_token(inv, 0)?;
                let mut mint = load_mint(inv, 1)?;
                let (mk, auth) = (inv.key(1)?, inv.key(2)?);
                mint.burn(&mk, &mut acct, &auth, amount)?;
                store(inv, 0, acct.pack())?;
                store(inv, 1, mint.pack())
            }
            9 => {
                if data.len() != 1 {
                    return Err(Error::InvalidInstructionData);
                }
                need_signer(inv, 2)?;
                let acct = load_token(inv, 0)?;
                if acct.state == State::Uninitialized {
                    return Err(TokenError::Uninitialized.into());
                }
                if acct.amount != 0 {
                    return Err(TokenError::NonZeroBalance.into());
                }
                let who = inv.key(2)?;
                if who != acct.close_authority.unwrap_or(acct.owner) {
                    return Err(TokenError::OwnerMismatch.into());
                }
                if inv.key(0)? == inv.key(1)? {
                    return Err(Error::InvalidInstructionData);
                }
                let lamports = inv.account(0)?.lamports;
                let total = inv
                    .account(1)?
                    .lamports
                    .checked_add(lamports)
                    .ok_or(Error::ArithmeticOverflow)?;
                inv.account_mut(1)?.lamports = total;
                let a = inv.account_mut(0)?;
                a.lamports = 0;
                a.data = Vec::new();
                a.owner = Pubkey::default();
                Ok(())
            }
            10 | 11 => {
                if data.len() != 1 {
                    return Err(Error::InvalidInstructionData);
                }
                need_signer(inv, 2)?;
                let mut acct = load_token(inv, 0)?;
                let mint = load_mint(inv, 1)?;
                if acct.mint != inv.key(1)? {
                    return Err(TokenError::MintMismatch.into());
                }
                let Some(fa) = mint.freeze_authority else {
                    return Err(TokenError::NoFreezeAuthority.into());
                };
                if fa != inv.key(2)? {
                    return Err(TokenError::OwnerMismatch.into());
                }
                acct.state = match (tag, acct.state) {
                    (_, State::Uninitialized) => return Err(TokenError::Uninitialized.into()),
                    (10, State::Frozen) | (11, State::Initialized) => {
                        return Err(Error::InvalidAccountData);
                    }
                    (10, _) => State::Frozen,
                    _ => State::Initialized,
                };
                store(inv, 0, acct.pack())
            }
            _ => Err(Error::InvalidInstructionData),
        }
    }
}

/// Convenience for tests and clients: a rent-exempt mint account.
pub fn mint_account(mint: &Mint, lamports: u64) -> Account {
    Account {
        lamports,
        data: mint.pack(),
        owner: TOKEN_PROGRAM_ID,
        executable: false,
    }
}

/// Convenience: a rent-exempt token account.
pub fn token_account(t: &TokenAccount, lamports: u64) -> Account {
    Account {
        lamports,
        data: t.pack(),
        owner: TOKEN_PROGRAM_ID,
        executable: false,
    }
}
```

### `src/ata.rs`

```rust
//! Associated token accounts: the one canonical token account of a wallet for
//! a mint, at a program-derived address, created by a small program.

use crate::token::{self, ACCOUNT_LEN, TOKEN_PROGRAM_ID, TokenAccount};
use solanalab::account::rent_exempt_minimum;
use solanalab::instruction::{AccountMeta, Instruction};
use solanalab::pubkey::{Pubkey, find_program_address};
use solanalab::runtime::{Error, Invoke, Program};
use solanalab::system;

/// The associated-token-account program's address (`ATokenGPv…`, from memory).
pub const ATA_PROGRAM_ID: Pubkey = Pubkey([
    0x8c, 0x97, 0x25, 0x8f, 0x4e, 0x24, 0x89, 0xf1, 0xbb, 0x3d, 0x10, 0x29, 0x14, 0x8e, 0x0d, 0x83,
    0x0b, 0x5a, 0x13, 0x99, 0xda, 0xff, 0x10, 0x84, 0x04, 0x8e, 0x7b, 0xd8, 0xdb, 0xe9, 0xf8, 0x59,
]);
/// Units charged for the program's own work (model value).
pub const COST: u64 = 200;

/// The address and bump of `wallet`'s associated token account for `mint`:
/// the PDA of `[wallet, token program, mint]` under the ATA program.
pub fn associated_address(wallet: &Pubkey, mint: &Pubkey) -> (Pubkey, u8) {
    find_program_address(&[&wallet.0, &TOKEN_PROGRAM_ID.0, &mint.0], &ATA_PROGRAM_ID)
        .expect("a bump exists")
}

/// `Create` (data empty) or `CreateIdempotent` (data `[1]`).
/// Accounts: payer, ata, wallet, mint, system program, token program.
pub fn create_ix(payer: Pubkey, wallet: Pubkey, mint: Pubkey, idempotent: bool) -> Instruction {
    let (ata, _) = associated_address(&wallet, &mint);
    Instruction {
        program_id: ATA_PROGRAM_ID,
        accounts: vec![
            AccountMeta::writable(payer, true),
            AccountMeta::writable(ata, false),
            AccountMeta::readonly(wallet, false),
            AccountMeta::readonly(mint, false),
            AccountMeta::readonly(Pubkey::default(), false),
            AccountMeta::readonly(TOKEN_PROGRAM_ID, false),
        ],
        data: if idempotent { vec![1] } else { vec![] },
    }
}

/// The program that creates associated token accounts.
pub struct AtaProgram;

impl Program for AtaProgram {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(COST)?;
        let idempotent = match data {
            [] => false,
            [1] => true,
            _ => return Err(Error::InvalidInstructionData),
        };
        if !inv.is_signer(0)? {
            return Err(Error::NotSigner(0));
        }
        if inv.key(4)? != Pubkey::default() || inv.key(5)? != TOKEN_PROGRAM_ID {
            return Err(Error::InvalidInstructionData);
        }
        let (payer, ata, wallet, mint) = (inv.key(0)?, inv.key(1)?, inv.key(2)?, inv.key(3)?);
        let m = inv.account(3)?;
        if m.owner != TOKEN_PROGRAM_ID || !token::Mint::unpack(&m.data)?.is_initialized {
            return Err(Error::InvalidAccountData);
        }
        inv.consume(1500)?; // the PDA search
        let (expected, bump) = associated_address(&wallet, &mint);
        if expected != ata {
            return Err(Error::InvalidSeeds);
        }
        let existing = inv.account(1)?;
        if existing.owner == TOKEN_PROGRAM_ID {
            if !idempotent {
                return Err(Error::AlreadyInUse);
            }
            let t = TokenAccount::unpack(&existing.data)?;
            if t.owner != wallet || t.mint != mint {
                return Err(Error::InvalidSeeds);
            }
            return Ok(());
        }
        if !existing.data.is_empty() || existing.owner != Pubkey::default() {
            return Err(Error::AlreadyInUse);
        }
        let lamports = rent_exempt_minimum(ACCOUNT_LEN).saturating_sub(existing.lamports);
        let bump_seed = [bump];
        let seeds: &[&[u8]] = &[&wallet.0, &TOKEN_PROGRAM_ID.0, &mint.0, &bump_seed];
        let create = Instruction {
            program_id: Pubkey::default(),
            accounts: vec![
                AccountMeta::writable(payer, true),
                AccountMeta::writable(ata, true),
            ],
            data: system::create_account_data(lamports, ACCOUNT_LEN as u64, &TOKEN_PROGRAM_ID),
        };
        inv.invoke(&create, &[seeds])?;
        inv.invoke(&token::initialize_account(ata, mint, wallet), &[])
    }
}
```

### `src/escrow.rs`

```rust
//! An escrow built from the pieces of this lesson: a maker locks token A in a
//! vault owned by a program-derived address and asks for token B; a taker who
//! pays the asked amount receives the locked tokens; the maker may cancel.
//!
//! Every instruction begins with [`validate`] over a list of [`Spec`]s; only
//! then does it read data or move anything.

use crate::ata::associated_address;
use crate::discriminator;
use crate::token::{self, TOKEN_PROGRAM_ID, TokenAccount};
use crate::validate::{Bump, Check, Seed, Spec, spec, validate};
use solanalab::account::rent_exempt_minimum;
use solanalab::instruction::{AccountMeta, Instruction};
use solanalab::pubkey::{Pubkey, find_program_address};
use solanalab::runtime::{Error, Invoke, Program};
use solanalab::system;

/// Size of an `Offer` account: discriminator, three keys, id, price, bump.
pub const OFFER_LEN: usize = 8 + 32 * 3 + 8 + 8 + 1;
const MAKER: usize = 8;
const MINT_A: usize = 40;
const MINT_B: usize = 72;
const ID: usize = 104;
const AMOUNT_B: usize = 112;
const BUMP: usize = 120;
/// Compute units charged for the program's own work (model value).
pub const COST: u64 = 400;

/// The decoded `Offer` account.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Offer {
    /// Who made the offer and receives token B.
    pub maker: Pubkey,
    /// The token locked in the vault.
    pub mint_a: Pubkey,
    /// The token asked for.
    pub mint_b: Pubkey,
    /// Chosen by the maker so one maker can have many offers.
    pub id: u64,
    /// How much token B the taker must pay.
    pub amount_b: u64,
    /// Canonical bump of the offer's PDA.
    pub bump: u8,
}

impl Offer {
    /// Serializes with the `Offer` discriminator in front.
    pub fn pack(&self) -> Vec<u8> {
        let mut v = discriminator::account("Offer").to_vec();
        v.extend_from_slice(&self.maker.0);
        v.extend_from_slice(&self.mint_a.0);
        v.extend_from_slice(&self.mint_b.0);
        v.extend_from_slice(&self.id.to_le_bytes());
        v.extend_from_slice(&self.amount_b.to_le_bytes());
        v.push(self.bump);
        v
    }

    /// Strict parse (length and discriminator).
    pub fn unpack(d: &[u8]) -> Result<Offer, Error> {
        if d.len() != OFFER_LEN || d[..8] != discriminator::account("Offer") {
            return Err(Error::InvalidAccountData);
        }
        let key = |o: usize| Pubkey(d[o..o + 32].try_into().expect("32"));
        Ok(Offer {
            maker: key(MAKER),
            mint_a: key(MINT_A),
            mint_b: key(MINT_B),
            id: u64::from_le_bytes(d[ID..ID + 8].try_into().expect("8")),
            amount_b: u64::from_le_bytes(d[AMOUNT_B..AMOUNT_B + 8].try_into().expect("8")),
            bump: d[BUMP],
        })
    }
}

/// The offer PDA of `maker` and `id`, and its canonical bump.
pub fn offer_address(program: &Pubkey, maker: &Pubkey, id: u64) -> (Pubkey, u8) {
    find_program_address(&[b"offer", &maker.0, &id.to_le_bytes()], program).expect("a bump exists")
}

fn offer_seeds(maker: Pubkey, id: u64, bump: u8) -> [Vec<u8>; 4] {
    [
        b"offer".to_vec(),
        maker.0.to_vec(),
        id.to_le_bytes().to_vec(),
        vec![bump],
    ]
}

// ---------- client-side instruction builders ----------

/// `make_offer(id, amount_a, amount_b)`.
/// Accounts: maker, offer, mint_a, mint_b, maker_ata_a, vault, token program, system program.
pub fn make_offer_ix(
    program: Pubkey,
    maker: Pubkey,
    mint_a: Pubkey,
    mint_b: Pubkey,
    id: u64,
    amount_a: u64,
    amount_b: u64,
) -> Instruction {
    let (offer, _) = offer_address(&program, &maker, id);
    let (maker_ata_a, _) = associated_address(&maker, &mint_a);
    let (vault, _) = associated_address(&offer, &mint_a);
    let mut data = discriminator::instruction("make_offer").to_vec();
    data.extend_from_slice(&id.to_le_bytes());
    data.extend_from_slice(&amount_a.to_le_bytes());
    data.extend_from_slice(&amount_b.to_le_bytes());
    Instruction {
        program_id: program,
        accounts: vec![
            AccountMeta::writable(maker, true),
            AccountMeta::writable(offer, false),
            AccountMeta::readonly(mint_a, false),
            AccountMeta::readonly(mint_b, false),
            AccountMeta::writable(maker_ata_a, false),
            AccountMeta::writable(vault, false),
            AccountMeta::readonly(TOKEN_PROGRAM_ID, false),
            AccountMeta::readonly(Pubkey::default(), false),
        ],
        data,
    }
}

/// `take_offer`. Accounts: taker, maker, offer, mint_a, mint_b, taker_ata_a,
/// taker_ata_b, maker_ata_b, vault, token program.
pub fn take_offer_ix(program: Pubkey, taker: Pubkey, offer_state: &Offer) -> Instruction {
    let (offer, _) = offer_address(&program, &offer_state.maker, offer_state.id);
    Instruction {
        program_id: program,
        accounts: vec![
            AccountMeta::writable(taker, true),
            AccountMeta::writable(offer_state.maker, false),
            AccountMeta::writable(offer, false),
            AccountMeta::readonly(offer_state.mint_a, false),
            AccountMeta::readonly(offer_state.mint_b, false),
            AccountMeta::writable(associated_address(&taker, &offer_state.mint_a).0, false),
            AccountMeta::writable(associated_address(&taker, &offer_state.mint_b).0, false),
            AccountMeta::writable(
                associated_address(&offer_state.maker, &offer_state.mint_b).0,
                false,
            ),
            AccountMeta::writable(associated_address(&offer, &offer_state.mint_a).0, false),
            AccountMeta::readonly(TOKEN_PROGRAM_ID, false),
        ],
        data: discriminator::instruction("take_offer").to_vec(),
    }
}

/// `cancel_offer`. Accounts: maker, offer, mint_a, maker_ata_a, vault, token program.
pub fn cancel_offer_ix(program: Pubkey, offer_state: &Offer) -> Instruction {
    let (offer, _) = offer_address(&program, &offer_state.maker, offer_state.id);
    Instruction {
        program_id: program,
        accounts: vec![
            AccountMeta::writable(offer_state.maker, true),
            AccountMeta::writable(offer, false),
            AccountMeta::readonly(offer_state.mint_a, false),
            AccountMeta::writable(
                associated_address(&offer_state.maker, &offer_state.mint_a).0,
                false,
            ),
            AccountMeta::writable(associated_address(&offer, &offer_state.mint_a).0, false),
            AccountMeta::readonly(TOKEN_PROGRAM_ID, false),
        ],
        data: discriminator::instruction("cancel_offer").to_vec(),
    }
}

// ---------- the program ----------

/// The escrow program.
pub struct EscrowProgram;

fn token_acct(mint: usize, owner: usize) -> Vec<Check> {
    vec![
        Check::Mut,
        Check::Owner(TOKEN_PROGRAM_ID),
        Check::TokenMint(mint),
        Check::TokenOwner(owner),
    ]
}

fn typed_offer(program: Pubkey) -> Check {
    Check::Typed {
        owner: program,
        disc: discriminator::account("Offer"),
        len: OFFER_LEN,
    }
}

fn offer_pda_check(program: Pubkey, maker_index: usize) -> Check {
    Check::Seeds {
        seeds: vec![
            Seed::Bytes(b"offer".to_vec()),
            Seed::Key(maker_index),
            Seed::OwnData(ID, 8),
        ],
        bump: Bump::Stored(BUMP),
        program,
    }
}

fn close_offer(inv: &mut Invoke<'_>, offer: usize, to: usize) -> Result<(), Error> {
    let lamports = inv.account(offer)?.lamports;
    let total = inv
        .account(to)?
        .lamports
        .checked_add(lamports)
        .ok_or(Error::ArithmeticOverflow)?;
    inv.account_mut(to)?.lamports = total;
    let a = inv.account_mut(offer)?;
    a.lamports = 0;
    a.data = Vec::new();
    a.owner = Pubkey::default();
    Ok(())
}

fn u64_arg(d: &[u8], at: usize) -> u64 {
    u64::from_le_bytes(d[at..at + 8].try_into().expect("8"))
}

impl Program for EscrowProgram {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(COST)?;
        let me = inv.program_id();
        let disc: [u8; 8] = data
            .get(..8)
            .ok_or(Error::InvalidInstructionData)?
            .try_into()
            .expect("8");
        if disc == discriminator::instruction("make_offer") {
            self.make_offer(inv, me, data)
        } else if disc == discriminator::instruction("take_offer") && data.len() == 8 {
            self.take_offer(inv, me)
        } else if disc == discriminator::instruction("cancel_offer") && data.len() == 8 {
            self.cancel_offer(inv, me)
        } else {
            Err(Error::InvalidInstructionData)
        }
    }
}

impl EscrowProgram {
    fn make_offer(&self, inv: &mut Invoke<'_>, me: Pubkey, data: &[u8]) -> Result<(), Error> {
        if data.len() != 8 + 24 {
            return Err(Error::InvalidInstructionData);
        }
        let (id, amount_a, amount_b) = (u64_arg(data, 8), u64_arg(data, 16), u64_arg(data, 24));
        if amount_a == 0 || amount_b == 0 {
            return Err(Error::InvalidInstructionData);
        }
        inv.consume(1500)?; // two PDA searches would be charged here by the real runtime
        let (maker, mint_a) = (inv.key(0)?, inv.key(2)?);
        let (offer_key, bump) = offer_address(&me, &maker, id);
        let vault_key = associated_address(&offer_key, &mint_a).0;
        let specs: Vec<Spec> = vec![
            spec("maker", vec![Check::Signer, Check::Mut]),
            spec(
                "offer",
                vec![Check::Mut, Check::Address(offer_key), Check::Uninit],
            ),
            spec("mint_a", vec![Check::Owner(TOKEN_PROGRAM_ID)]),
            spec("mint_b", vec![Check::Owner(TOKEN_PROGRAM_ID)]),
            spec("maker_ata_a", token_acct(2, 0)),
            spec(
                "vault",
                [token_acct(2, 1), vec![Check::Address(vault_key)]].concat(),
            ),
            spec("token_program", vec![Check::Program(TOKEN_PROGRAM_ID)]),
            spec("system_program", vec![Check::Program(Pubkey::default())]),
        ];
        validate(inv, &specs)?;

        // create the offer account, signing for its PDA
        let seeds = offer_seeds(maker, id, bump);
        let seed_refs: Vec<&[u8]> = seeds.iter().map(Vec::as_slice).collect();
        let lamports = rent_exempt_minimum(OFFER_LEN).saturating_sub(inv.account(1)?.lamports);
        let create = Instruction {
            program_id: Pubkey::default(),
            accounts: vec![
                AccountMeta::writable(maker, true),
                AccountMeta::writable(offer_key, true),
            ],
            data: system::create_account_data(lamports, OFFER_LEN as u64, &me),
        };
        inv.invoke(&create, &[&seed_refs])?;
        let offer = Offer {
            maker,
            mint_a,
            mint_b: inv.key(3)?,
            id,
            amount_b,
            bump,
        };
        inv.account_mut(1)?.data = offer.pack();

        // lock token A in the vault; the maker's signature extends through the call
        let ix = token::transfer(inv.key(4)?, vault_key, maker, amount_a);
        inv.invoke(&ix, &[])
    }

    fn take_offer(&self, inv: &mut Invoke<'_>, me: Pubkey) -> Result<(), Error> {
        let offer_key = inv.key(2)?;
        let mint_a_key = inv.key(3)?;
        let vault_key = associated_address(&offer_key, &mint_a_key).0;
        let specs: Vec<Spec> = vec![
            spec("taker", vec![Check::Signer, Check::Mut]),
            spec("maker", vec![Check::Mut]),
            spec(
                "offer",
                vec![
                    Check::Mut,
                    typed_offer(me),
                    Check::HasOne {
                        offset: MAKER,
                        target: 1,
                    },
                    Check::HasOne {
                        offset: MINT_A,
                        target: 3,
                    },
                    Check::HasOne {
                        offset: MINT_B,
                        target: 4,
                    },
                    offer_pda_check(me, 1),
                ],
            ),
            spec("mint_a", vec![Check::Owner(TOKEN_PROGRAM_ID)]),
            spec("mint_b", vec![Check::Owner(TOKEN_PROGRAM_ID)]),
            spec("taker_ata_a", token_acct(3, 0)),
            spec("taker_ata_b", token_acct(4, 0)),
            spec("maker_ata_b", token_acct(4, 1)),
            spec(
                "vault",
                [token_acct(3, 2), vec![Check::Address(vault_key)]].concat(),
            ),
            spec("token_program", vec![Check::Program(TOKEN_PROGRAM_ID)]),
        ];
        validate(inv, &specs)?;
        let offer = Offer::unpack(&inv.account(2)?.data)?;
        let vault_amount = TokenAccount::unpack(&inv.account(8)?.data)?.amount;
        let (taker, maker) = (inv.key(0)?, inv.key(1)?);

        // taker pays the maker (taker's signature extends through the call)
        let pay = token::transfer(inv.key(6)?, inv.key(7)?, taker, offer.amount_b);
        inv.invoke(&pay, &[])?;

        // the offer PDA releases the vault and closes it
        let seeds = offer_seeds(maker, offer.id, offer.bump);
        let seed_refs: Vec<&[u8]> = seeds.iter().map(Vec::as_slice).collect();
        let release = token::transfer(inv.key(8)?, inv.key(5)?, offer_key, vault_amount);
        inv.invoke(&release, &[&seed_refs])?;
        let close = token::close_account(inv.key(8)?, maker, offer_key);
        inv.invoke(&close, &[&seed_refs])?;
        close_offer(inv, 2, 1)
    }

    fn cancel_offer(&self, inv: &mut Invoke<'_>, me: Pubkey) -> Result<(), Error> {
        let offer_key = inv.key(1)?;
        let mint_a_key = inv.key(2)?;
        let vault_key = associated_address(&offer_key, &mint_a_key).0;
        let specs: Vec<Spec> = vec![
            spec("maker", vec![Check::Signer, Check::Mut]),
            spec(
                "offer",
                vec![
                    Check::Mut,
                    typed_offer(me),
                    Check::HasOne {
                        offset: MAKER,
                        target: 0,
                    },
                    Check::HasOne {
                        offset: MINT_A,
                        target: 2,
                    },
                    offer_pda_check(me, 0),
                ],
            ),
            spec("mint_a", vec![Check::Owner(TOKEN_PROGRAM_ID)]),
            spec("maker_ata_a", token_acct(2, 0)),
            spec(
                "vault",
                [token_acct(2, 1), vec![Check::Address(vault_key)]].concat(),
            ),
            spec("token_program", vec![Check::Program(TOKEN_PROGRAM_ID)]),
        ];
        validate(inv, &specs)?;
        let offer = Offer::unpack(&inv.account(1)?.data)?;
        let vault_amount = TokenAccount::unpack(&inv.account(4)?.data)?.amount;
        let maker = inv.key(0)?;
        let seeds = offer_seeds(maker, offer.id, offer.bump);
        let seed_refs: Vec<&[u8]> = seeds.iter().map(Vec::as_slice).collect();
        let back = token::transfer(inv.key(4)?, inv.key(3)?, offer_key, vault_amount);
        inv.invoke(&back, &[&seed_refs])?;
        let close = token::close_account(inv.key(4)?, maker, offer_key);
        inv.invoke(&close, &[&seed_refs])?;
        close_offer(inv, 1, 0)
    }
}
```

### `tests/common/mod.rs`

```rust
#![allow(dead_code)]
use anchorlab::ata::{ATA_PROGRAM_ID, AtaProgram, associated_address};
use anchorlab::escrow::EscrowProgram;
use anchorlab::token::*;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::instruction::{Instruction, Transaction};
use solanalab::pubkey::Pubkey;
use solanalab::runtime::{Bank, Error, Receipt};

pub const ESC: Pubkey = Pubkey([0xE5; 32]);
pub const RICH: u64 = 1_000_000_000_000;

pub fn k(n: u8) -> Pubkey {
    Pubkey::filled(n)
}

pub fn bank() -> Bank {
    let mut b = Bank::new();
    b.add_program(TOKEN_PROGRAM_ID, Box::new(TokenProgram));
    b.add_program(ATA_PROGRAM_ID, Box::new(AtaProgram));
    b.add_program(ESC, Box::new(EscrowProgram));
    b
}

pub fn wallet(b: &mut Bank, n: u8) {
    b.set_account(k(n), Account::wallet(RICH));
}

pub fn mint(b: &mut Bank, key: Pubkey, decimals: u8, authority: Pubkey, freeze: Option<Pubkey>) {
    let m = Mint {
        mint_authority: Some(authority),
        supply: 0,
        decimals,
        is_initialized: true,
        freeze_authority: freeze,
    };
    b.set_account(key, mint_account(&m, rent_exempt_minimum(MINT_LEN)));
}

/// A token account at an arbitrary address; also raises the mint's supply by `amount`.
pub fn tacct(b: &mut Bank, key: Pubkey, mint_key: Pubkey, owner: Pubkey, amount: u64) {
    let mut t = TokenAccount::new(mint_key, owner);
    t.amount = amount;
    b.set_account(key, token_account(&t, rent_exempt_minimum(ACCOUNT_LEN)));
    if amount > 0 {
        let mut m = get_mint(b, &mint_key);
        m.supply += amount;
        let lam = b.lamports(&mint_key);
        b.set_account(mint_key, mint_account(&m, lam));
    }
}

/// A token account at the associated address.
pub fn ata(b: &mut Bank, wallet: Pubkey, mint_key: Pubkey, amount: u64) -> Pubkey {
    let (a, _) = associated_address(&wallet, &mint_key);
    tacct(b, a, mint_key, wallet, amount);
    a
}

pub fn get_token(b: &Bank, key: &Pubkey) -> TokenAccount {
    TokenAccount::unpack(&b.account(key).expect("exists").data).expect("valid")
}

pub fn get_mint(b: &Bank, key: &Pubkey) -> Mint {
    Mint::unpack(&b.account(key).expect("exists").data).expect("valid")
}

pub fn amount(b: &Bank, key: &Pubkey) -> u64 {
    get_token(b, key).amount
}

pub fn tx(signers: &[u8], ixs: Vec<Instruction>) -> Transaction {
    Transaction {
        signers: signers.iter().map(|&n| k(n)).collect(),
        instructions: ixs,
    }
}

pub fn run(b: &mut Bank, signers: &[u8], ixs: Vec<Instruction>) -> Result<Receipt, Error> {
    b.process(&tx(signers, ixs))
}
```

### `tests/disc_layout.rs`

```rust
use anchorlab::ata::*;
use anchorlab::discriminator;
use anchorlab::escrow::*;
use anchorlab::token::*;
use solanalab::pubkey::Pubkey;

fn k(n: u8) -> Pubkey {
    Pubkey::filled(n)
}
fn hex(s: &str) -> Vec<u8> {
    (0..s.len() / 2)
        .map(|i| u8::from_str_radix(&s[2 * i..2 * i + 2], 16).unwrap())
        .collect()
}

#[test]
fn discriminator_known_answers() {
    // computed with Python's hashlib; "global:initialize" is the value I remember from Anchor
    assert_eq!(
        discriminator::instruction("initialize").to_vec(),
        hex("afaf6d1f0d989bed")
    );
    assert_eq!(
        discriminator::instruction("make_offer").to_vec(),
        hex("d66261233b0c2cb2")
    );
    assert_eq!(
        discriminator::instruction("take_offer").to_vec(),
        hex("809cf2cfedc067f0")
    );
    assert_eq!(
        discriminator::instruction("cancel_offer").to_vec(),
        hex("5ccbdf285c593577")
    );
    assert_eq!(
        discriminator::account("Offer").to_vec(),
        hex("d7583c47aaa249e5")
    );
}

#[test]
fn account_and_instruction_namespaces_differ() {
    assert_ne!(
        discriminator::account("Offer"),
        discriminator::instruction("Offer")
    );
    assert_ne!(
        discriminator::account("Offer"),
        discriminator::account("offer")
    );
}

#[test]
fn program_ids_have_their_well_known_text_form() {
    assert_eq!(
        TOKEN_PROGRAM_ID.to_string(),
        "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA"
    );
    assert_eq!(
        ATA_PROGRAM_ID.to_string(),
        "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
    );
}

#[test]
fn mint_layout_is_82_bytes_with_known_offsets() {
    let m = Mint {
        mint_authority: Some(k(1)),
        supply: 0x0102_0304_0506_0708,
        decimals: 9,
        is_initialized: true,
        freeze_authority: None,
    };
    let d = m.pack();
    assert_eq!(d.len(), 82);
    assert_eq!(&d[0..4], &[1, 0, 0, 0]);
    assert_eq!(&d[4..36], &[1; 32]);
    assert_eq!(&d[36..44], &[8, 7, 6, 5, 4, 3, 2, 1]);
    assert_eq!(d[44], 9);
    assert_eq!(d[45], 1);
    assert_eq!(&d[46..50], &[0, 0, 0, 0]);
    assert_eq!(&d[50..82], &[0; 32]);
    assert_eq!(Mint::unpack(&d).unwrap(), m);
}

#[test]
fn token_account_layout_is_165_bytes_with_known_offsets() {
    let t = TokenAccount {
        mint: k(1),
        owner: k(2),
        amount: 258,
        delegate: Some(k(3)),
        state: State::Frozen,
        is_native: Some(5),
        delegated_amount: 7,
        close_authority: Some(k(4)),
    };
    let d = t.pack();
    assert_eq!(d.len(), 165);
    assert_eq!(&d[0..32], &[1; 32]);
    assert_eq!(&d[32..64], &[2; 32]);
    assert_eq!(&d[64..72], &[2, 1, 0, 0, 0, 0, 0, 0]);
    assert_eq!(&d[72..76], &[1, 0, 0, 0]);
    assert_eq!(&d[76..108], &[3; 32]);
    assert_eq!(d[108], 2);
    assert_eq!(&d[109..113], &[1, 0, 0, 0]);
    assert_eq!(&d[113..121], &[5, 0, 0, 0, 0, 0, 0, 0]);
    assert_eq!(&d[121..129], &[7, 0, 0, 0, 0, 0, 0, 0]);
    assert_eq!(&d[129..133], &[1, 0, 0, 0]);
    assert_eq!(&d[133..165], &[4; 32]);
    assert_eq!(TokenAccount::unpack(&d).unwrap(), t);
    // an all-zero account is the uninitialized state
    assert_eq!(
        TokenAccount::unpack(&[0; 165]).unwrap().state,
        State::Uninitialized
    );
}

#[test]
fn unpacking_is_strict() {
    let m = Mint {
        mint_authority: None,
        supply: 0,
        decimals: 0,
        is_initialized: true,
        freeze_authority: None,
    }
    .pack();
    assert!(Mint::unpack(&m[..81]).is_err());
    let mut long = m.clone();
    long.push(0);
    assert!(Mint::unpack(&long).is_err());
    for at in [45usize, 0, 46] {
        let mut bad = m.clone();
        bad[at] = 2;
        assert!(Mint::unpack(&bad).is_err(), "byte {at}");
    }
    let t = TokenAccount::new(k(1), k(2)).pack();
    assert!(TokenAccount::unpack(&t[..164]).is_err());
    for at in [108usize, 72, 109, 129] {
        let mut bad = t.clone();
        bad[at] = 3;
        assert!(TokenAccount::unpack(&bad).is_err(), "byte {at}");
    }
}

#[test]
fn associated_addresses_match_an_independent_derivation() {
    // Python: sha256(seeds || bump || program || "ProgramDerivedAddress") with a pure-Python curve check
    let (a, bump) = associated_address(&k(1), &k(2));
    assert_eq!(
        a.to_string(),
        "CsYkfSfTUTWwnoeRkGchtai5kkYz2SC33kKJwA99wVr3"
    );
    assert_eq!(bump, 255);
    let (a, bump) = associated_address(&k(7), &k(9));
    assert_eq!(a.to_string(), "BjmJ1yi1Sc4s9xQaiv4DbRuUhgfjUSc8cYSuwsFqoS9");
    assert_eq!(bump, 255);
    assert_ne!(
        associated_address(&k(1), &k(2)).0,
        associated_address(&k(2), &k(1)).0
    );
}

#[test]
fn offer_addresses_match_an_independent_derivation() {
    let esc = Pubkey::filled(0xE5);
    let (a, bump) = offer_address(&esc, &k(1), 7);
    assert_eq!(
        a.to_string(),
        "HbrjB4sbqSEh7jDKk8Kjw5jLHZ32aZ9HNhX1RgpCpkkq"
    );
    assert_eq!(bump, 254);
    assert_eq!(
        offer_address(&esc, &k(1), 8).0.to_string(),
        "AW2gBw24cM6KEEFYk7qhzu6pgdzTANPeDH16BtjUv4o7"
    );
    assert_eq!(
        offer_address(&esc, &k(2), 7).0.to_string(),
        "2X1W6tGJdNWdjvtk63ZJrtUvr87Q5zPoQQpw2HMCLWFZ"
    );
}

#[test]
fn offer_packs_and_unpacks_strictly() {
    let o = Offer {
        maker: k(1),
        mint_a: k(2),
        mint_b: k(3),
        id: 9,
        amount_b: 77,
        bump: 254,
    };
    let d = o.pack();
    assert_eq!(d.len(), OFFER_LEN);
    assert_eq!(OFFER_LEN, 121);
    assert_eq!(&d[..8], &hex("d7583c47aaa249e5")[..]);
    assert_eq!(Offer::unpack(&d).unwrap(), o);
    assert!(Offer::unpack(&d[..120]).is_err());
    let mut bad = d.clone();
    bad[0] ^= 1;
    assert!(Offer::unpack(&bad).is_err());
}
```

### `tests/validate_tests.rs`

```rust
mod common;
use anchorlab::validate::*;
use common::*;
use solanalab::account::Account;
use solanalab::instruction::{AccountMeta, Instruction, Transaction};
use solanalab::pubkey::{Pubkey, find_program_address};
use solanalab::runtime::{Error, Invoke, Program};
use std::cell::RefCell;
use std::rc::Rc;

const P: Pubkey = Pubkey([0x77; 32]);

struct V {
    specs: Vec<Spec>,
    out: Rc<RefCell<Option<AnchorError>>>,
}
impl Program for V {
    fn process(&self, inv: &mut Invoke<'_>, _d: &[u8]) -> Result<(), Error> {
        match validate(inv, &self.specs) {
            Ok(()) => Ok(()),
            Err(e) => {
                *self.out.borrow_mut() = Some(e.clone());
                Err(e.into())
            }
        }
    }
}

type Metas = Vec<(Pubkey, bool, bool, Account)>; // key, signer, writable, account

/// Runs `validate` inside a one-instruction transaction and returns its verdict.
fn check(metas: Metas, specs: Vec<Spec>) -> Result<(), AnchorError> {
    let out = Rc::new(RefCell::new(None));
    let mut b = bank();
    b.add_program(
        P,
        Box::new(V {
            specs,
            out: out.clone(),
        }),
    );
    wallet(&mut b, 1);
    let mut accounts = vec![];
    for (key, signer, writable, acct) in &metas {
        b.set_account(*key, acct.clone());
        accounts.push(if *writable {
            AccountMeta::writable(*key, *signer)
        } else {
            AccountMeta::readonly(*key, *signer)
        });
    }
    let mut signers = vec![k(1)];
    for (key, signer, _, _) in &metas {
        if *signer && !signers.contains(key) {
            signers.push(*key);
        }
    }
    let t = Transaction {
        signers,
        instructions: vec![Instruction {
            program_id: P,
            accounts,
            data: vec![],
        }],
    };
    let r = b.process(&t);
    let e = out.borrow().clone();
    match e {
        Some(e) => Err(e),
        None => {
            r.expect("validation passed, so the instruction succeeds");
            Ok(())
        }
    }
}

fn w(lamports: u64) -> Account {
    Account::wallet(lamports)
}
fn data_acct(owner: Pubkey, data: Vec<u8>) -> Account {
    Account {
        lamports: 10_000_000,
        data,
        owner,
        executable: false,
    }
}
fn fail(r: Result<(), AnchorError>) -> (u32, &'static str, usize, &'static str) {
    let e = r.expect_err("must fail");
    (e.code, e.name, e.account, e.field)
}

#[test]
fn signer_and_mut() {
    let ok = check(
        vec![(k(2), true, true, w(5))],
        vec![spec("a", vec![Check::Signer, Check::Mut])],
    );
    assert_eq!(ok, Ok(()));
    let r = check(
        vec![(k(2), false, true, w(5))],
        vec![spec("a", vec![Check::Signer])],
    );
    assert_eq!(fail(r), (2002, "ConstraintSigner", 0, "a"));
    let r = check(
        vec![(k(2), true, false, w(5))],
        vec![spec("a", vec![Check::Mut])],
    );
    assert_eq!(fail(r), (2000, "ConstraintMut", 0, "a"));
}

#[test]
fn address_and_program_and_owner() {
    let m = || vec![(k(2), false, false, w(5))];
    assert_eq!(
        check(m(), vec![spec("a", vec![Check::Address(k(2))])]),
        Ok(())
    );
    assert_eq!(
        fail(check(m(), vec![spec("a", vec![Check::Address(k(3))])])),
        (2012, "ConstraintAddress", 0, "a")
    );
    assert_eq!(
        fail(check(m(), vec![spec("a", vec![Check::Program(k(3))])])),
        (3008, "InvalidProgramId", 0, "a")
    );
    // right key but not executable
    assert_eq!(
        fail(check(m(), vec![spec("a", vec![Check::Program(k(2))])])),
        (3009, "InvalidProgramExecutable", 0, "a")
    );
    let exec = Account {
        executable: true,
        ..w(1)
    };
    assert_eq!(
        check(
            vec![(k(2), false, false, exec)],
            vec![spec("a", vec![Check::Program(k(2))])]
        ),
        Ok(())
    );
    assert_eq!(
        check(m(), vec![spec("a", vec![Check::Owner(Pubkey::default())])]),
        Ok(())
    );
    assert_eq!(
        fail(check(m(), vec![spec("a", vec![Check::Owner(k(9))])])),
        (2004, "ConstraintOwner", 0, "a")
    );
}

#[test]
fn typed_accounts_check_owner_discriminator_and_length() {
    let disc = [1, 2, 3, 4, 5, 6, 7, 8];
    let typed = |len| Check::Typed {
        owner: k(9),
        disc,
        len,
    };
    let good = data_acct(k(9), [&disc[..], &[0; 4]].concat());
    let s = |len| vec![spec("t", vec![typed(len)])];
    assert_eq!(
        check(vec![(k(2), false, false, good.clone())], s(12)),
        Ok(())
    );
    assert_eq!(
        fail(check(vec![(k(2), false, false, good.clone())], s(13))),
        (3003, "AccountDidNotDeserialize", 0, "t")
    );
    let wrong_owner = Account {
        owner: k(8),
        ..good.clone()
    };
    assert_eq!(
        fail(check(vec![(k(2), false, false, wrong_owner)], s(12))),
        (3007, "AccountOwnedByWrongProgram", 0, "t")
    );
    let wrong_disc = data_acct(k(9), [&[9u8; 8][..], &[0; 4]].concat());
    assert_eq!(
        fail(check(vec![(k(2), false, false, wrong_disc)], s(12))),
        (3002, "AccountDiscriminatorMismatch", 0, "t")
    );
    let short = data_acct(k(9), vec![1, 2, 3]);
    assert_eq!(
        fail(check(vec![(k(2), false, false, short)], s(12))),
        (3001, "AccountDiscriminatorNotFound", 0, "t")
    );
    // exactly 8 bytes with the right discriminator and len 8 is fine
    let eight = data_acct(k(9), disc.to_vec());
    assert_eq!(check(vec![(k(2), false, false, eight)], s(8)), Ok(()));
}

#[test]
fn has_one_compares_a_stored_key_with_another_account() {
    let mut d = vec![0u8; 8];
    d.extend_from_slice(&k(5).0);
    let holder = data_acct(k(9), d);
    let metas = |other: u8| {
        vec![
            (k(2), false, false, holder.clone()),
            (k(other), false, false, w(1)),
        ]
    };
    let s = vec![
        spec(
            "holder",
            vec![Check::HasOne {
                offset: 8,
                target: 1,
            }],
        ),
        spec("other", vec![]),
    ];
    assert_eq!(check(metas(5), s.clone()), Ok(()));
    assert_eq!(
        fail(check(metas(6), s.clone())),
        (2001, "ConstraintHasOne", 0, "holder")
    );
    // an offset past the data is a failure, not a panic
    let s2 = vec![
        spec(
            "holder",
            vec![Check::HasOne {
                offset: 30,
                target: 1,
            }],
        ),
        spec("other", vec![]),
    ];
    assert_eq!(
        fail(check(metas(5), s2)),
        (2001, "ConstraintHasOne", 0, "holder")
    );
    // a target that does not exist
    let s3 = vec![spec(
        "holder",
        vec![Check::HasOne {
            offset: 8,
            target: 7,
        }],
    )];
    assert_eq!(fail(check(metas(5), s3)).1, "AccountNotEnoughKeys");
}

#[test]
fn seeds_with_canonical_stored_and_own_data_parts() {
    let (pda, bump) = find_program_address(&[b"vault", &k(5).0], &P).unwrap();
    let canon = vec![
        spec(
            "pda",
            vec![Check::Seeds {
                seeds: vec![Seed::Bytes(b"vault".to_vec()), Seed::Key(1)],
                bump: Bump::Canonical,
                program: P,
            }],
        ),
        spec("user", vec![]),
    ];
    let m = |key: Pubkey, user: u8| vec![(key, false, false, w(1)), (k(user), false, false, w(1))];
    assert_eq!(check(m(pda, 5), canon.clone()), Ok(()));
    assert_eq!(
        fail(check(m(pda, 6), canon.clone())),
        (2006, "ConstraintSeeds", 0, "pda")
    );
    assert_eq!(fail(check(m(k(40), 5), canon)).1, "ConstraintSeeds");
    // stored bump: the bump byte lives in the account's data at offset 3; one seed comes from the data
    let mut d = vec![0u8; 4];
    d[0] = 9;
    d[3] = bump;
    let (pda2, bump2) = find_program_address(&[b"vault", &[9]], &P).unwrap();
    d[3] = bump2;
    let stored = |key: Pubkey, d: &Vec<u8>| vec![(key, false, false, data_acct(P, d.clone()))];
    let s = vec![spec(
        "pda",
        vec![Check::Seeds {
            seeds: vec![Seed::Bytes(b"vault".to_vec()), Seed::OwnData(0, 1)],
            bump: Bump::Stored(3),
            program: P,
        }],
    )];
    assert_eq!(check(stored(pda2, &d), s.clone()), Ok(()));
    d[3] = bump2.wrapping_sub(1);
    assert_eq!(
        fail(check(stored(pda2, &d), s.clone())).1,
        "ConstraintSeeds"
    );
    // an own-data seed range past the end
    let s_bad = vec![spec(
        "pda",
        vec![Check::Seeds {
            seeds: vec![Seed::OwnData(2, 9)],
            bump: Bump::Stored(3),
            program: P,
        }],
    )];
    assert_eq!(fail(check(stored(pda2, &d), s_bad)).1, "ConstraintSeeds");
    // a stored bump byte that is missing
    d.truncate(3);
    assert_eq!(fail(check(stored(pda2, &d), s)).1, "ConstraintSeeds");
}

#[test]
fn token_mint_and_owner_fields() {
    let mut d = vec![0u8; 165];
    d[0..32].copy_from_slice(&k(5).0);
    d[32..64].copy_from_slice(&k(6).0);
    let t = data_acct(k(9), d);
    let s = vec![
        spec("t", vec![Check::TokenMint(1), Check::TokenOwner(2)]),
        spec("mint", vec![]),
        spec("owner", vec![]),
    ];
    let m = |mint, owner| {
        vec![
            (k(2), false, false, t.clone()),
            (k(mint), false, false, w(1)),
            (k(owner), false, false, w(1)),
        ]
    };
    assert_eq!(check(m(5, 6), s.clone()), Ok(()));
    assert_eq!(
        fail(check(m(7, 6), s.clone())),
        (2014, "ConstraintTokenMint", 0, "t")
    );
    assert_eq!(
        fail(check(m(5, 7), s.clone())),
        (2015, "ConstraintTokenOwner", 0, "t")
    );
    // swapped fields do not pass
    assert_eq!(fail(check(m(6, 5), s)).1, "ConstraintTokenMint");
    // a short account fails instead of panicking
    let short = data_acct(k(9), vec![0; 40]);
    let s2 = vec![spec("t", vec![Check::TokenOwner(1)]), spec("o", vec![])];
    assert_eq!(
        fail(check(
            vec![(k(2), false, false, short), (k(6), false, false, w(1))],
            s2
        ))
        .1,
        "ConstraintTokenOwner"
    );
}

#[test]
fn uninit_means_empty_and_system_owned() {
    let s = vec![spec("a", vec![Check::Uninit])];
    assert_eq!(check(vec![(k(2), false, false, w(5))], s.clone()), Ok(()));
    assert_eq!(
        fail(check(
            vec![(k(2), false, false, data_acct(Pubkey::default(), vec![1]))],
            s.clone()
        )),
        (2010, "ConstraintInit", 0, "a")
    );
    assert_eq!(
        fail(check(
            vec![(k(2), false, false, data_acct(k(9), vec![]))],
            s
        ))
        .1,
        "ConstraintInit"
    );
}

#[test]
fn the_same_writable_account_twice_is_refused() {
    let s = vec![spec("a", vec![Check::Mut]), spec("b", vec![Check::Mut])];
    let dup = vec![(k(2), false, true, w(5)), (k(2), false, true, w(5))];
    assert_eq!(
        fail(check(dup, s.clone())),
        (2040, "ConstraintDuplicateMutableAccount", 1, "b")
    );
    let distinct = vec![(k(2), false, true, w(5)), (k(3), false, true, w(5))];
    assert_eq!(check(distinct, s), Ok(()));
    // one read-only alias is allowed by this rule
    let s2 = vec![spec("a", vec![Check::Mut]), spec("b", vec![])];
    assert_eq!(
        check(
            vec![(k(2), false, true, w(5)), (k(2), false, false, w(5))],
            s2
        ),
        Ok(())
    );
}

#[test]
fn too_few_accounts_and_check_order() {
    let s = vec![spec("a", vec![]), spec("b", vec![])];
    assert_eq!(
        fail(check(vec![(k(2), false, false, w(5))], s)),
        (3005, "AccountNotEnoughKeys", 1, "-")
    );
    // the first failing check of the first failing account is reported
    let s = vec![
        spec("a", vec![Check::Signer, Check::Mut]),
        spec("b", vec![Check::Mut]),
    ];
    let metas = vec![(k(2), false, false, w(5)), (k(3), false, false, w(5))];
    assert_eq!(fail(check(metas, s)), (2002, "ConstraintSigner", 0, "a"));
}

#[test]
fn anchor_errors_map_to_runtime_errors() {
    let e = |name| AnchorError {
        code: 0,
        name,
        account: 3,
        field: "x",
    };
    assert_eq!(Error::from(e("ConstraintMut")), Error::NotWritable(3));
    assert_eq!(Error::from(e("ConstraintSigner")), Error::NotSigner(3));
    assert_eq!(Error::from(e("ConstraintOwner")), Error::WrongOwner(3));
    assert_eq!(
        Error::from(e("AccountOwnedByWrongProgram")),
        Error::WrongOwner(3)
    );
    assert_eq!(Error::from(e("ConstraintHasOne")), Error::WrongAuthority);
    assert_eq!(Error::from(e("ConstraintSeeds")), Error::InvalidSeeds);
    assert_eq!(
        Error::from(e("AccountNotEnoughKeys")),
        Error::MissingAccount(3)
    );
    assert_eq!(Error::from(e("ConstraintInit")), Error::AlreadyInUse);
    assert_eq!(
        Error::from(e("ConstraintTokenMint")),
        Error::InvalidAccountData
    );
}

#[test]
fn a_seven_byte_account_has_no_discriminator() {
    let disc = [7u8; 8];
    let s = vec![spec(
        "t",
        vec![Check::Typed {
            owner: k(9),
            disc,
            len: 8,
        }],
    )];
    // seven bytes: one short of a discriminator
    let short = data_acct(k(9), vec![7; 7]);
    assert_eq!(
        fail(check(vec![(k(2), false, false, short)], s.clone())),
        (3001, "AccountDiscriminatorNotFound", 0, "t")
    );
    // exactly eight bytes is enough to look at
    let exact = data_acct(k(9), vec![7; 8]);
    assert_eq!(check(vec![(k(2), false, false, exact)], s), Ok(()));
}

#[test]
fn only_declared_mutable_slots_count_for_duplicates() {
    // slot 0 does not declare `Mut`, slot 1 does; the same key in both is allowed by this rule
    let s = vec![spec("a", vec![]), spec("b", vec![Check::Mut])];
    let metas = vec![(k(2), false, true, w(5)), (k(2), false, true, w(5))];
    assert_eq!(check(metas, s), Ok(()));
}

#[test]
fn a_stored_bump_may_be_non_canonical_and_a_searched_one_may_not() {
    // the same logical PDA has several valid bumps; `Stored` accepts the one in the data
    // (safe only because the program wrote it), `Canonical` accepts only the first found
    let (canon_addr, canon) = find_program_address(&[b"alt"], &P).unwrap();
    let other = (0..canon)
        .rev()
        .find_map(|b| {
            solanalab::pubkey::create_program_address(&[b"alt", &[b]], &P)
                .ok()
                .map(|a| (a, b))
        })
        .expect("a lower valid bump exists");
    let stored = vec![spec(
        "pda",
        vec![Check::Seeds {
            seeds: vec![Seed::Bytes(b"alt".to_vec())],
            bump: Bump::Stored(0),
            program: P,
        }],
    )];
    let searched = vec![spec(
        "pda",
        vec![Check::Seeds {
            seeds: vec![Seed::Bytes(b"alt".to_vec())],
            bump: Bump::Canonical,
            program: P,
        }],
    )];
    let acct = |b: u8| data_acct(P, vec![b]);
    assert_eq!(
        check(vec![(other.0, false, false, acct(other.1))], stored.clone()),
        Ok(())
    );
    assert_eq!(
        check(vec![(canon_addr, false, false, acct(canon))], stored),
        Ok(())
    );
    assert_eq!(
        fail(check(
            vec![(other.0, false, false, acct(other.1))],
            searched.clone()
        )),
        (2006, "ConstraintSeeds", 0, "pda")
    );
    assert_eq!(
        check(vec![(canon_addr, false, false, acct(canon))], searched),
        Ok(())
    );
}
```

### `tests/token_core.rs`

```rust
//! The token rules as pure functions.
use anchorlab::token::*;
use solanalab::pubkey::Pubkey;

fn k(n: u8) -> Pubkey {
    Pubkey::filled(n)
}
const M: u8 = 100;

fn acct(owner: u8, amount: u64) -> TokenAccount {
    let mut t = TokenAccount::new(k(M), k(owner));
    t.amount = amount;
    t
}

fn mint(authority: Option<u8>, supply: u64) -> Mint {
    Mint {
        mint_authority: authority.map(k),
        supply,
        decimals: 6,
        is_initialized: true,
        freeze_authority: None,
    }
}

#[test]
fn owner_transfers() {
    let (mut a, mut b) = (acct(1, 10), acct(2, 5));
    a.transfer(&mut b, &k(1), 4, None).unwrap();
    assert_eq!((a.amount, b.amount), (6, 9));
    a.transfer(&mut b, &k(1), 6, None).unwrap();
    assert_eq!((a.amount, b.amount), (0, 15));
    a.transfer(&mut b, &k(1), 0, None).unwrap();
}

#[test]
fn transfer_failures_change_nothing() {
    let (a0, b0) = (acct(1, 10), acct(2, 5));
    let cases: Vec<(Pubkey, u64, TokenError)> = vec![
        (k(1), 11, TokenError::InsufficientFunds),
        (k(3), 1, TokenError::OwnerMismatch),
    ];
    for (who, n, want) in cases {
        let (mut a, mut b) = (a0.clone(), b0.clone());
        assert_eq!(a.transfer(&mut b, &who, n, None), Err(want));
        assert_eq!((&a, &b), (&a0, &b0));
    }
    // overflow on the receiving side
    let (mut a, mut c) = (acct(1, 10), acct(2, u64::MAX));
    assert_eq!(
        a.transfer(&mut c, &k(1), 1, None),
        Err(TokenError::Overflow)
    );
    assert_eq!(a.amount, 10);
}

#[test]
fn mints_must_match_and_accounts_must_be_initialized() {
    let (mut a, mut b) = (acct(1, 10), acct(2, 0));
    b.mint = k(101);
    assert_eq!(
        a.transfer(&mut b, &k(1), 1, None),
        Err(TokenError::MintMismatch)
    );
    let mut u = acct(2, 0);
    u.state = State::Uninitialized;
    assert_eq!(
        a.transfer(&mut u, &k(1), 1, None),
        Err(TokenError::Uninitialized)
    );
    let mut s = acct(1, 10);
    s.state = State::Uninitialized;
    let mut ok = acct(2, 0);
    assert_eq!(
        s.transfer(&mut ok, &k(1), 1, None),
        Err(TokenError::Uninitialized)
    );
}

#[test]
fn frozen_accounts_neither_send_nor_receive() {
    let (mut a, mut b) = (acct(1, 10), acct(2, 0));
    a.state = State::Frozen;
    assert_eq!(
        a.transfer(&mut b, &k(1), 1, None),
        Err(TokenError::AccountFrozen)
    );
    a.state = State::Initialized;
    b.state = State::Frozen;
    assert_eq!(
        a.transfer(&mut b, &k(1), 1, None),
        Err(TokenError::AccountFrozen)
    );
}

#[test]
fn delegates_spend_up_to_their_allowance() {
    let mut a = acct(1, 10);
    let mut b = acct(2, 0);
    a.delegate = Some(k(9));
    a.delegated_amount = 5;
    assert_eq!(
        a.transfer(&mut b, &k(9), 6, None),
        Err(TokenError::InsufficientFunds)
    );
    assert_eq!(a.delegated_amount, 5);
    a.transfer(&mut b, &k(9), 3, None).unwrap();
    assert_eq!(
        (a.amount, a.delegated_amount, a.delegate),
        (7, 2, Some(k(9)))
    );
    a.transfer(&mut b, &k(9), 2, None).unwrap();
    assert_eq!((a.amount, a.delegated_amount, a.delegate), (5, 0, None));
    assert_eq!(
        a.transfer(&mut b, &k(9), 1, None),
        Err(TokenError::OwnerMismatch)
    );
    // the owner is never limited by the allowance
    a.transfer(&mut b, &k(1), 5, None).unwrap();
}

#[test]
fn transfer_checked_compares_mint_and_decimals() {
    let (mut a, mut b) = (acct(1, 10), acct(2, 0));
    let m = k(M);
    a.transfer(&mut b, &k(1), 1, Some((&m, 6, 6))).unwrap();
    assert_eq!(
        a.transfer(&mut b, &k(1), 1, Some((&m, 6, 5))),
        Err(TokenError::DecimalsMismatch)
    );
    let other = k(101);
    assert_eq!(
        a.transfer(&mut b, &k(1), 1, Some((&other, 6, 6))),
        Err(TokenError::MintMismatch)
    );
}

#[test]
fn minting_needs_the_authority_and_updates_supply_and_balance() {
    let mut m = mint(Some(1), 100);
    let mut a = acct(7, 5);
    m.mint_to(&k(M), &mut a, &k(1), 20).unwrap();
    assert_eq!((m.supply, a.amount), (120, 25));
    assert_eq!(
        m.mint_to(&k(M), &mut a, &k(2), 1),
        Err(TokenError::OwnerMismatch)
    );
    let mut fixed = mint(None, 100);
    assert_eq!(
        fixed.mint_to(&k(M), &mut a, &k(1), 1),
        Err(TokenError::OwnerMismatch)
    );
    assert_eq!(
        m.mint_to(&k(101), &mut a, &k(1), 1),
        Err(TokenError::MintMismatch)
    );
    a.state = State::Frozen;
    assert_eq!(
        m.mint_to(&k(M), &mut a, &k(1), 1),
        Err(TokenError::AccountFrozen)
    );
    a.state = State::Initialized;
    let mut big = mint(Some(1), u64::MAX);
    assert_eq!(
        big.mint_to(&k(M), &mut a, &k(1), 1),
        Err(TokenError::Overflow)
    );
    let mut b2 = acct(8, u64::MAX);
    let mut m2 = mint(Some(1), 0);
    assert_eq!(
        m2.mint_to(&k(M), &mut b2, &k(1), 1),
        Err(TokenError::Overflow)
    );
    assert_eq!(m2.supply, 0);
    let mut uninit = mint(Some(1), 0);
    uninit.is_initialized = false;
    assert_eq!(
        uninit.mint_to(&k(M), &mut a, &k(1), 1),
        Err(TokenError::Uninitialized)
    );
}

#[test]
fn burning_reduces_balance_and_supply() {
    let mut m = mint(Some(1), 100);
    let mut a = acct(7, 30);
    m.burn(&k(M), &mut a, &k(7), 10).unwrap();
    assert_eq!((m.supply, a.amount), (90, 20));
    assert_eq!(
        m.burn(&k(M), &mut a, &k(7), 21),
        Err(TokenError::InsufficientFunds)
    );
    assert_eq!(
        m.burn(&k(M), &mut a, &k(8), 1),
        Err(TokenError::OwnerMismatch)
    );
    assert_eq!(
        m.burn(&k(101), &mut a, &k(7), 1),
        Err(TokenError::MintMismatch)
    );
    a.delegate = Some(k(9));
    a.delegated_amount = 3;
    m.burn(&k(M), &mut a, &k(9), 3).unwrap();
    assert_eq!((m.supply, a.amount, a.delegate), (87, 17, None));
    a.state = State::Frozen;
    assert_eq!(
        m.burn(&k(M), &mut a, &k(7), 1),
        Err(TokenError::AccountFrozen)
    );
}

#[test]
fn token_errors_map_to_runtime_errors() {
    use solanalab::runtime::Error;
    assert_eq!(
        Error::from(TokenError::InsufficientFunds),
        Error::InsufficientFunds
    );
    assert_eq!(
        Error::from(TokenError::OwnerMismatch),
        Error::WrongAuthority
    );
    assert_eq!(Error::from(TokenError::AlreadyInUse), Error::AlreadyInUse);
    assert_eq!(Error::from(TokenError::Overflow), Error::ArithmeticOverflow);
    assert_eq!(
        Error::from(TokenError::AccountFrozen),
        Error::InvalidAccountData
    );
}
```

### `tests/token_program.rs`

```rust
//! The token program through the bank.
mod common;
use anchorlab::token::*;
use common::*;
use solanalab::account::Account;
use solanalab::runtime::Error;

const M: u8 = 100;

fn setup() -> solanalab::runtime::Bank {
    let mut b = bank();
    for n in 1..=4 {
        wallet(&mut b, n);
    }
    mint(&mut b, k(M), 6, k(1), Some(k(1)));
    tacct(&mut b, k(11), k(M), k(1), 0);
    tacct(&mut b, k(12), k(M), k(2), 0);
    b
}

#[test]
fn mint_transfer_burn_and_supply_accounting() {
    let mut b = setup();
    run(&mut b, &[1], vec![mint_to(k(M), k(11), k(1), 1000)]).unwrap();
    assert_eq!(
        (amount(&b, &k(11)), get_mint(&b, &k(M)).supply),
        (1000, 1000)
    );
    run(&mut b, &[1], vec![transfer(k(11), k(12), k(1), 400)]).unwrap();
    assert_eq!((amount(&b, &k(11)), amount(&b, &k(12))), (600, 400));
    run(&mut b, &[2], vec![burn(k(12), k(M), k(2), 150)]).unwrap();
    assert_eq!((amount(&b, &k(12)), get_mint(&b, &k(M)).supply), (250, 850));
}

#[test]
fn initialize_mint_and_account() {
    let mut b = bank();
    wallet(&mut b, 1);
    // allocate a zeroed mint and a zeroed token account owned by the token program
    let blank = |len: usize| Account {
        lamports: 10_000_000,
        data: vec![0; len],
        owner: TOKEN_PROGRAM_ID,
        executable: false,
    };
    b.set_account(k(50), blank(MINT_LEN));
    b.set_account(k(51), blank(ACCOUNT_LEN));
    run(&mut b, &[1], vec![initialize_mint(k(50), 9, k(1), None)]).unwrap();
    let m = get_mint(&b, &k(50));
    assert_eq!(
        (
            m.decimals,
            m.mint_authority,
            m.freeze_authority,
            m.supply,
            m.is_initialized
        ),
        (9, Some(k(1)), None, 0, true)
    );
    assert_eq!(
        run(&mut b, &[1], vec![initialize_mint(k(50), 9, k(1), None)]),
        Err(Error::AlreadyInUse)
    );
    run(&mut b, &[1], vec![initialize_account(k(51), k(50), k(1))]).unwrap();
    assert_eq!(get_token(&b, &k(51)), TokenAccount::new(k(50), k(1)));
    assert_eq!(
        run(&mut b, &[1], vec![initialize_account(k(51), k(50), k(2))]),
        Err(Error::AlreadyInUse)
    );
    // with a freeze authority
    b.set_account(k(52), blank(MINT_LEN));
    run(
        &mut b,
        &[1],
        vec![initialize_mint(k(52), 0, k(1), Some(k(2)))],
    )
    .unwrap();
    assert_eq!(get_mint(&b, &k(52)).freeze_authority, Some(k(2)));
    // an account that is not a mint, or an uninitialized mint, cannot back a token account
    b.set_account(k(53), blank(ACCOUNT_LEN));
    b.set_account(k(54), blank(MINT_LEN));
    assert_eq!(
        run(&mut b, &[1], vec![initialize_account(k(53), k(54), k(1))]),
        Err(Error::InvalidAccountData)
    );
    b.set_account(k(55), blank(MINT_LEN - 1));
    assert_eq!(
        run(&mut b, &[1], vec![initialize_mint(k(55), 0, k(1), None)]),
        Err(Error::InvalidAccountData)
    );
}

#[test]
fn only_the_signing_authority_moves_tokens() {
    let mut b = setup();
    tacct(&mut b, k(11), k(M), k(1), 100);
    // signer is not the owner
    assert_eq!(
        run(&mut b, &[2], vec![transfer(k(11), k(12), k(2), 1)]),
        Err(Error::WrongAuthority)
    );
    // owner named but not signing: the transaction itself is refused
    let mut ix = transfer(k(11), k(12), k(1), 1);
    ix.accounts[2].is_signer = false;
    assert_eq!(run(&mut b, &[2], vec![ix]), Err(Error::NotSigner(2)));
    // mint authority only
    assert_eq!(
        run(&mut b, &[2], vec![mint_to(k(M), k(12), k(2), 1)]),
        Err(Error::WrongAuthority)
    );
    assert_eq!(amount(&b, &k(11)), 100);
}

#[test]
fn accounts_must_be_owned_by_the_token_program() {
    let mut b = setup();
    tacct(&mut b, k(11), k(M), k(1), 100);
    // a lookalike token account owned by someone else
    let mut fake = b.account(&k(12)).unwrap().clone();
    fake.owner = k(77);
    b.set_account(k(13), fake);
    assert_eq!(
        run(&mut b, &[1], vec![transfer(k(11), k(13), k(1), 1)]),
        Err(Error::WrongOwner(1))
    );
    assert_eq!(
        run(&mut b, &[1], vec![transfer(k(13), k(11), k(1), 1)]),
        Err(Error::WrongOwner(0))
    );
}

#[test]
fn approve_delegate_spends_and_revoke_ends_it() {
    let mut b = setup();
    tacct(&mut b, k(11), k(M), k(1), 100);
    run(&mut b, &[1], vec![approve(k(11), k(3), k(1), 30)]).unwrap();
    let t = get_token(&b, &k(11));
    assert_eq!((t.delegate, t.delegated_amount), (Some(k(3)), 30));
    run(&mut b, &[3], vec![transfer(k(11), k(12), k(3), 20)]).unwrap();
    assert_eq!(get_token(&b, &k(11)).delegated_amount, 10);
    assert_eq!(
        run(&mut b, &[3], vec![transfer(k(11), k(12), k(3), 11)]),
        Err(Error::InsufficientFunds)
    );
    run(&mut b, &[1], vec![revoke(k(11), k(1))]).unwrap();
    assert_eq!(
        run(&mut b, &[3], vec![transfer(k(11), k(12), k(3), 1)]),
        Err(Error::WrongAuthority)
    );
    // only the owner can approve or revoke
    assert_eq!(
        run(&mut b, &[2], vec![approve(k(11), k(2), k(2), 5)]),
        Err(Error::WrongAuthority)
    );
    assert_eq!(
        run(&mut b, &[2], vec![revoke(k(11), k(2))]),
        Err(Error::WrongAuthority)
    );
}

#[test]
fn freeze_and_thaw() {
    let mut b = setup();
    tacct(&mut b, k(11), k(M), k(1), 100);
    run(&mut b, &[1], vec![freeze_account(k(11), k(M), k(1), true)]).unwrap();
    assert_eq!(get_token(&b, &k(11)).state, State::Frozen);
    assert_eq!(
        run(&mut b, &[1], vec![transfer(k(11), k(12), k(1), 1)]),
        Err(Error::InvalidAccountData)
    );
    assert_eq!(
        run(&mut b, &[1], vec![freeze_account(k(11), k(M), k(1), true)]),
        Err(Error::InvalidAccountData)
    );
    // the owner cannot thaw; only the freeze authority can
    assert_eq!(
        run(&mut b, &[2], vec![freeze_account(k(11), k(M), k(2), false)]),
        Err(Error::WrongAuthority)
    );
    run(&mut b, &[1], vec![freeze_account(k(11), k(M), k(1), false)]).unwrap();
    run(&mut b, &[1], vec![transfer(k(11), k(12), k(1), 1)]).unwrap();
    // a mint without a freeze authority cannot freeze
    mint(&mut b, k(60), 0, k(1), None);
    tacct(&mut b, k(61), k(60), k(1), 0);
    assert_eq!(
        run(&mut b, &[1], vec![freeze_account(k(61), k(60), k(1), true)]),
        Err(Error::InvalidAccountData)
    );
}

#[test]
fn close_needs_zero_balance_and_the_owner() {
    let mut b = setup();
    tacct(&mut b, k(11), k(M), k(1), 5);
    assert_eq!(
        run(&mut b, &[1], vec![close_account(k(11), k(3), k(1))]),
        Err(Error::InvalidAccountData)
    );
    run(&mut b, &[1], vec![transfer(k(11), k(12), k(1), 5)]).unwrap();
    assert_eq!(
        run(&mut b, &[2], vec![close_account(k(11), k(3), k(2))]),
        Err(Error::WrongAuthority)
    );
    let before = b.lamports(&k(3));
    let rent = b.lamports(&k(11));
    run(&mut b, &[1], vec![close_account(k(11), k(3), k(1))]).unwrap();
    assert!(b.account(&k(11)).is_none());
    assert_eq!(b.lamports(&k(3)), before + rent);
    // closing into itself is refused
    tacct(&mut b, k(13), k(M), k(2), 0);
    assert_eq!(
        run(&mut b, &[2], vec![close_account(k(13), k(13), k(2))]),
        Err(Error::InvalidInstructionData)
    );
}

#[test]
fn transfer_checked_validates_decimals_and_mint() {
    let mut b = setup();
    tacct(&mut b, k(11), k(M), k(1), 100);
    run(
        &mut b,
        &[1],
        vec![transfer_checked(k(11), k(M), k(12), k(1), 10, 6)],
    )
    .unwrap();
    assert_eq!(
        run(
            &mut b,
            &[1],
            vec![transfer_checked(k(11), k(M), k(12), k(1), 10, 5)]
        ),
        Err(Error::InvalidAccountData)
    );
    mint(&mut b, k(62), 6, k(1), None);
    assert_eq!(
        run(
            &mut b,
            &[1],
            vec![transfer_checked(k(11), k(62), k(12), k(1), 10, 6)]
        ),
        Err(Error::InvalidAccountData)
    );
    assert_eq!(amount(&b, &k(11)), 90);
}

#[test]
fn self_transfer_checks_but_moves_nothing() {
    let mut b = setup();
    tacct(&mut b, k(11), k(M), k(1), 10);
    run(&mut b, &[1], vec![transfer(k(11), k(11), k(1), 10)]).unwrap();
    assert_eq!(amount(&b, &k(11)), 10);
    assert_eq!(
        run(&mut b, &[1], vec![transfer(k(11), k(11), k(1), 11)]),
        Err(Error::InsufficientFunds)
    );
}

#[test]
fn malformed_instruction_data_is_refused() {
    let mut b = setup();
    tacct(&mut b, k(11), k(M), k(1), 10);
    for data in [
        vec![],
        vec![3],
        vec![3, 1, 2, 3, 4, 5, 6, 7],
        vec![3, 0, 0, 0, 0, 0, 0, 0, 0, 0],
        vec![99],
    ] {
        let mut ix = transfer(k(11), k(12), k(1), 1);
        ix.data = data;
        assert_eq!(
            run(&mut b, &[1], vec![ix]),
            Err(Error::InvalidInstructionData)
        );
    }
    let mut ix = initialize_mint(k(M), 0, k(1), None);
    ix.data.push(0);
    assert_eq!(
        run(&mut b, &[1], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
}

#[test]
fn readonly_destination_is_refused() {
    let mut b = setup();
    tacct(&mut b, k(11), k(M), k(1), 10);
    let mut ix = transfer(k(11), k(12), k(1), 1);
    ix.accounts[1].is_writable = false;
    assert!(run(&mut b, &[1], vec![ix]).is_err());
    assert_eq!(amount(&b, &k(11)), 10);
}
```

### `tests/ata_tests.rs`

```rust
mod common;
use anchorlab::ata::*;
use anchorlab::token::*;
use common::*;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::pubkey::Pubkey;
use solanalab::runtime::Error;

const M: u8 = 100;

fn setup() -> solanalab::runtime::Bank {
    let mut b = bank();
    for n in 1..=3 {
        wallet(&mut b, n);
    }
    mint(&mut b, k(M), 6, k(1), None);
    b
}

#[test]
fn create_makes_the_associated_account() {
    let mut b = setup();
    let (a, _) = associated_address(&k(2), &k(M));
    let payer_before = b.lamports(&k(1));
    run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), false)]).unwrap();
    let acct = b.account(&a).unwrap();
    assert_eq!(acct.owner, TOKEN_PROGRAM_ID);
    assert_eq!(acct.data.len(), 165);
    assert_eq!(acct.lamports, rent_exempt_minimum(165));
    assert_eq!(get_token(&b, &a), TokenAccount::new(k(M), k(2)));
    assert_eq!(
        payer_before - b.lamports(&k(1)),
        rent_exempt_minimum(165) + 5000
    );
}

#[test]
fn create_twice_is_refused_but_idempotent_create_is_not() {
    let mut b = setup();
    run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), false)]).unwrap();
    assert_eq!(
        run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), false)]),
        Err(Error::AlreadyInUse)
    );
    run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), true)]).unwrap();
}

#[test]
fn idempotent_create_on_a_foreign_account_is_refused() {
    let mut b = setup();
    let (a, _) = associated_address(&k(2), &k(M));
    // an account at the right address owned by the token program but for a different wallet
    tacct(&mut b, a, k(M), k(3), 0);
    assert_eq!(
        run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), true)]),
        Err(Error::InvalidSeeds)
    );
    // or for a different mint
    mint(&mut b, k(101), 0, k(1), None);
    tacct(&mut b, a, k(101), k(2), 0);
    assert_eq!(
        run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), true)]),
        Err(Error::InvalidSeeds)
    );
}

#[test]
fn create_tops_up_a_prefunded_address() {
    let mut b = setup();
    let (a, _) = associated_address(&k(2), &k(M));
    b.set_account(a, Account::wallet(1_000_000));
    run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), false)]).unwrap();
    assert_eq!(b.lamports(&a), rent_exempt_minimum(165));
    // a prefund above the minimum is simply kept
    let mut b = setup();
    b.set_account(a, Account::wallet(rent_exempt_minimum(165) + 77));
    run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), false)]).unwrap();
    assert_eq!(b.lamports(&a), rent_exempt_minimum(165) + 77);
}

#[test]
fn the_address_must_be_the_derived_one() {
    let mut b = setup();
    let mut ix = create_ix(k(1), k(2), k(M), false);
    ix.accounts[1].pubkey = k(50);
    assert_eq!(run(&mut b, &[1], vec![ix]), Err(Error::InvalidSeeds));
    // the address for another wallet does not work for this one
    let mut ix = create_ix(k(1), k(2), k(M), false);
    ix.accounts[1].pubkey = associated_address(&k(3), &k(M)).0;
    assert_eq!(run(&mut b, &[1], vec![ix]), Err(Error::InvalidSeeds));
}

#[test]
fn payer_signature_and_real_programs_are_required() {
    let mut b = setup();
    let mut ix = create_ix(k(1), k(2), k(M), false);
    ix.accounts[0].is_signer = false;
    assert_eq!(run(&mut b, &[2], vec![ix]), Err(Error::NotSigner(0)));
    let mut ix = create_ix(k(1), k(2), k(M), false);
    ix.accounts[5].pubkey = k(66);
    assert_eq!(
        run(&mut b, &[1], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
    let mut ix = create_ix(k(1), k(2), k(M), false);
    ix.accounts[4].pubkey = k(66);
    assert_eq!(
        run(&mut b, &[1], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
    let mut ix = create_ix(k(1), k(2), k(M), false);
    ix.data = vec![2];
    assert_eq!(
        run(&mut b, &[1], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
}

#[test]
fn the_mint_must_be_a_real_initialized_mint() {
    let mut b = setup();
    // not owned by the token program
    b.set_account(
        k(70),
        Account {
            lamports: 10_000_000,
            data: vec![0; 82],
            owner: k(9),
            executable: false,
        },
    );
    assert_eq!(
        run(&mut b, &[1], vec![create_ix(k(1), k(2), k(70), false)]),
        Err(Error::InvalidAccountData)
    );
    // token-owned but not initialized
    b.set_account(
        k(71),
        Account {
            lamports: 10_000_000,
            data: vec![0; 82],
            owner: TOKEN_PROGRAM_ID,
            executable: false,
        },
    );
    assert_eq!(
        run(&mut b, &[1], vec![create_ix(k(1), k(2), k(71), false)]),
        Err(Error::InvalidAccountData)
    );
}

#[test]
fn an_address_holding_foreign_data_is_not_overwritten() {
    let mut b = setup();
    let (a, _) = associated_address(&k(2), &k(M));
    b.set_account(
        a,
        Account {
            lamports: 5_000_000,
            data: vec![1, 2, 3],
            owner: Pubkey::default(),
            executable: false,
        },
    );
    assert_eq!(
        run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), false)]),
        Err(Error::AlreadyInUse)
    );
    b.set_account(
        a,
        Account {
            lamports: 5_000_000,
            data: vec![],
            owner: k(9),
            executable: false,
        },
    );
    assert_eq!(
        run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), false)]),
        Err(Error::AlreadyInUse)
    );
}

#[test]
fn a_created_account_is_immediately_usable() {
    let mut b = setup();
    tacct(&mut b, k(80), k(M), k(1), 50);
    run(&mut b, &[1], vec![create_ix(k(1), k(2), k(M), false)]).unwrap();
    let (a, _) = associated_address(&k(2), &k(M));
    run(&mut b, &[1], vec![transfer(k(80), a, k(1), 20)]).unwrap();
    assert_eq!(amount(&b, &a), 20);
}

#[test]
fn creation_in_the_same_transaction_as_use() {
    let mut b = setup();
    tacct(&mut b, k(80), k(M), k(1), 50);
    let (a, _) = associated_address(&k(2), &k(M));
    run(
        &mut b,
        &[1],
        vec![
            create_ix(k(1), k(2), k(M), true),
            transfer(k(80), a, k(1), 5),
            create_ix(k(1), k(2), k(M), true),
        ],
    )
    .unwrap();
    assert_eq!(amount(&b, &a), 5);
}
```

### `tests/escrow_tests.rs`

```rust
mod common;
use anchorlab::ata::{self, associated_address};
use anchorlab::escrow::*;
use anchorlab::token::*;
use common::*;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::pubkey::Pubkey;
use solanalab::runtime::{Bank, Error};

const A: u8 = 100;
const B: u8 = 101;
const MAKER: u8 = 1;
const TAKER: u8 = 2;
const EVE: u8 = 3;

struct World {
    b: Bank,
    offer: Offer,
    offer_key: Pubkey,
    vault: Pubkey,
}

fn base() -> Bank {
    let mut b = bank();
    for n in [MAKER, TAKER, EVE] {
        wallet(&mut b, n);
    }
    mint(&mut b, k(A), 6, k(MAKER), None);
    mint(&mut b, k(B), 6, k(TAKER), None);
    ata(&mut b, k(MAKER), k(A), 1000);
    ata(&mut b, k(MAKER), k(B), 0);
    ata(&mut b, k(TAKER), k(A), 0);
    ata(&mut b, k(TAKER), k(B), 500);
    b
}

fn make(b: &mut Bank, id: u64, a: u64, bb: u64) -> Result<(), Error> {
    let (offer, _) = offer_address(&ESC, &k(MAKER), id);
    run(
        b,
        &[MAKER],
        vec![
            ata::create_ix(k(MAKER), offer, k(A), true),
            make_offer_ix(ESC, k(MAKER), k(A), k(B), id, a, bb),
        ],
    )
    .map(|_| ())
}

fn world() -> World {
    let mut b = base();
    make(&mut b, 7, 400, 250).unwrap();
    let (offer_key, bump) = offer_address(&ESC, &k(MAKER), 7);
    let offer = Offer {
        maker: k(MAKER),
        mint_a: k(A),
        mint_b: k(B),
        id: 7,
        amount_b: 250,
        bump,
    };
    let vault = associated_address(&offer_key, &k(A)).0;
    World {
        b,
        offer,
        offer_key,
        vault,
    }
}

fn lamports_total(b: &Bank) -> u128 {
    b.total_supply()
}

#[test]
fn make_offer_locks_tokens_and_records_the_offer() {
    let w = world();
    assert_eq!(amount(&w.b, &associated_address(&k(MAKER), &k(A)).0), 600);
    assert_eq!(amount(&w.b, &w.vault), 400);
    let acct = w.b.account(&w.offer_key).unwrap();
    assert_eq!(acct.owner, ESC);
    assert_eq!(acct.lamports, rent_exempt_minimum(OFFER_LEN));
    assert_eq!(Offer::unpack(&acct.data).unwrap(), w.offer);
    // the vault belongs to the offer's PDA, not to the maker
    assert_eq!(get_token(&w.b, &w.vault).owner, w.offer_key);
}

#[test]
fn take_offer_swaps_closes_and_refunds_rent() {
    let mut w = world();
    let supply_before = lamports_total(&w.b);
    let maker_before = w.b.lamports(&k(MAKER));
    let vault_rent = w.b.lamports(&w.vault);
    let offer_rent = w.b.lamports(&w.offer_key);
    run(
        &mut w.b,
        &[TAKER],
        vec![take_offer_ix(ESC, k(TAKER), &w.offer)],
    )
    .unwrap();
    let ta = |m| associated_address(&k(TAKER), &k(m)).0;
    let ma = |m| associated_address(&k(MAKER), &k(m)).0;
    assert_eq!((amount(&w.b, &ta(A)), amount(&w.b, &ta(B))), (400, 250));
    assert_eq!((amount(&w.b, &ma(A)), amount(&w.b, &ma(B))), (600, 250));
    assert!(w.b.account(&w.vault).is_none());
    assert!(w.b.account(&w.offer_key).is_none());
    assert_eq!(
        w.b.lamports(&k(MAKER)),
        maker_before + vault_rent + offer_rent
    );
    assert_eq!(
        lamports_total(&w.b),
        supply_before,
        "fees move to the fee pot, which total_supply counts"
    );
    assert_eq!(get_mint(&w.b, &k(A)).supply, 1000);
}

#[test]
fn cancel_returns_the_tokens_and_closes_everything() {
    let mut w = world();
    run(&mut w.b, &[MAKER], vec![cancel_offer_ix(ESC, &w.offer)]).unwrap();
    assert_eq!(amount(&w.b, &associated_address(&k(MAKER), &k(A)).0), 1000);
    assert!(w.b.account(&w.vault).is_none());
    assert!(w.b.account(&w.offer_key).is_none());
    // the id can be used again
    make(&mut w.b, 7, 100, 10).unwrap();
}

#[test]
fn cancel_is_for_the_maker_only() {
    let mut w = world();
    // Eve signs as "maker" of the real offer: the stored maker differs
    let mut ix = cancel_offer_ix(ESC, &w.offer);
    ix.accounts[0].pubkey = k(EVE);
    assert_eq!(run(&mut w.b, &[EVE], vec![ix]), Err(Error::WrongAuthority));
    // the maker named correctly but not signing
    let mut ix = cancel_offer_ix(ESC, &w.offer);
    ix.accounts[0].is_signer = false;
    assert_eq!(run(&mut w.b, &[EVE], vec![ix]), Err(Error::NotSigner(0)));
    assert_eq!(amount(&w.b, &w.vault), 400);
}

#[test]
fn a_second_take_or_cancel_finds_nothing() {
    let mut w = world();
    run(
        &mut w.b,
        &[TAKER],
        vec![take_offer_ix(ESC, k(TAKER), &w.offer)],
    )
    .unwrap();
    assert!(
        run(
            &mut w.b,
            &[TAKER],
            vec![take_offer_ix(ESC, k(TAKER), &w.offer)]
        )
        .is_err()
    );
    assert!(run(&mut w.b, &[MAKER], vec![cancel_offer_ix(ESC, &w.offer)]).is_err());
}

#[test]
fn make_twice_with_the_same_id_is_refused_and_leaves_the_vault_alone() {
    let mut w = world();
    assert_eq!(make(&mut w.b, 7, 100, 1), Err(Error::AlreadyInUse));
    assert_eq!(amount(&w.b, &w.vault), 400);
    assert_eq!(
        Offer::unpack(&w.b.account(&w.offer_key).unwrap().data)
            .unwrap()
            .amount_b,
        250
    );
}

#[test]
fn make_offer_validation() {
    let mut b = base();
    let good = || make_offer_ix(ESC, k(MAKER), k(A), k(B), 1, 100, 10);
    let attempt = |b: &mut Bank, ix| {
        run(
            b,
            &[MAKER],
            vec![
                ata::create_ix(k(MAKER), offer_address(&ESC, &k(MAKER), 1).0, k(A), true),
                ix,
            ],
        )
    };
    // zero amounts
    let mut ix = good();
    ix.data[16..24].copy_from_slice(&0u64.to_le_bytes());
    assert_eq!(attempt(&mut b, ix), Err(Error::InvalidInstructionData));
    let mut ix = good();
    ix.data[24..32].copy_from_slice(&0u64.to_le_bytes());
    assert_eq!(attempt(&mut b, ix), Err(Error::InvalidInstructionData));
    // wrong length
    let mut ix = good();
    ix.data.push(0);
    assert_eq!(attempt(&mut b, ix), Err(Error::InvalidInstructionData));
    // offer address not the derived one
    let mut ix = good();
    ix.accounts[1].pubkey = k(60);
    assert_eq!(attempt(&mut b, ix), Err(Error::InvalidAccountData));
    // vault not the associated account of the offer
    let mut ix = good();
    tacct(&mut b, k(61), k(A), offer_address(&ESC, &k(MAKER), 1).0, 0);
    ix.accounts[5].pubkey = k(61);
    assert_eq!(attempt(&mut b, ix), Err(Error::InvalidAccountData));
    // maker's token account belongs to someone else
    let mut ix = good();
    ix.accounts[4].pubkey = associated_address(&k(TAKER), &k(A)).0;
    assert_eq!(attempt(&mut b, ix), Err(Error::InvalidAccountData));
    // a mint account the token program does not own
    let mut ix = good();
    b.set_account(
        k(62),
        Account {
            lamports: 1,
            data: vec![0; 82],
            owner: k(9),
            executable: false,
        },
    );
    ix.accounts[3].pubkey = k(62);
    assert_eq!(attempt(&mut b, ix), Err(Error::WrongOwner(3)));
    // fake token program and fake system program
    let mut ix = good();
    ix.accounts[6].pubkey = k(63);
    assert_eq!(attempt(&mut b, ix), Err(Error::InvalidAccountData));
    let mut ix = good();
    ix.accounts[7].pubkey = k(63);
    assert_eq!(attempt(&mut b, ix), Err(Error::InvalidAccountData));
    // maker not a signer
    let mut ix = good();
    ix.accounts[0].is_signer = false;
    assert_eq!(attempt(&mut b, ix), Err(Error::NotSigner(0)));
    // not enough tokens: the transfer inside fails and nothing is created
    let mut ix = good();
    ix.data[16..24].copy_from_slice(&1001u64.to_le_bytes());
    assert_eq!(attempt(&mut b, ix), Err(Error::InsufficientFunds));
    assert!(b.account(&offer_address(&ESC, &k(MAKER), 1).0).is_none());
    // and the honest one still works
    attempt(&mut b, good()).unwrap();
}

#[test]
fn a_prefunded_offer_address_still_works() {
    let mut b = base();
    let (offer, _) = offer_address(&ESC, &k(MAKER), 1);
    b.set_account(offer, Account::wallet(1_000_000));
    make(&mut b, 1, 10, 10).unwrap();
    assert_eq!(b.lamports(&offer), rent_exempt_minimum(OFFER_LEN));
}

#[test]
fn take_offer_attacks() {
    let w = world();
    let try_ix = |f: &dyn Fn(&mut solanalab::instruction::Instruction, &mut Bank)| {
        let mut w2 = world();
        let mut ix = take_offer_ix(ESC, k(TAKER), &w2.offer);
        f(&mut ix, &mut w2.b);
        let r = run(&mut w2.b, &[TAKER], vec![ix]);
        (r, w2)
    };
    // the taker does not sign
    let (r, _) = try_ix(&|ix, _| ix.accounts[0].is_signer = false);
    assert_eq!(r, Err(Error::NotSigner(0)));
    // a different maker account (receives the rent refund) than the one in the offer
    let (r, _) = try_ix(&|ix, _| ix.accounts[1].pubkey = k(EVE));
    assert_eq!(r, Err(Error::WrongAuthority));
    // Eve substitutes her own token account for the maker's payment destination
    let (r, _) = try_ix(&|ix, b| {
        tacct(b, k(70), k(B), k(EVE), 0);
        ix.accounts[7].pubkey = k(70);
    });
    assert_eq!(r, Err(Error::InvalidAccountData));
    // the taker's token-A account belongs to someone else: tokens would leak
    let (r, _) = try_ix(&|ix, _| ix.accounts[5].pubkey = associated_address(&k(EVE), &k(A)).0);
    assert!(r.is_err());
    // vault replaced by Eve's own token account holding A
    let (r, _) = try_ix(&|ix, b| {
        tacct(b, k(71), k(A), k(EVE), 0);
        ix.accounts[8].pubkey = k(71);
    });
    assert_eq!(r, Err(Error::InvalidAccountData));
    // a fake token program
    let (r, _) = try_ix(&|ix, _| ix.accounts[9].pubkey = k(72));
    assert_eq!(r, Err(Error::InvalidAccountData));
    // wrong mint B for the payment (a worthless token of the same layout)
    let (r, _) = try_ix(&|ix, b| {
        mint(b, k(73), 6, k(EVE), None);
        ix.accounts[4].pubkey = k(73);
    });
    assert_eq!(r, Err(Error::WrongAuthority));
    // nothing moved in any of these
    let w3 = world();
    assert_eq!(amount(&w.b, &w.vault), amount(&w3.b, &w3.vault));
}

#[test]
fn a_lookalike_offer_cannot_be_forged() {
    // Eve assigns an account of her own to the escrow program; she cannot write its data
    let mut w = world();
    let mut forged = Account {
        lamports: rent_exempt_minimum(OFFER_LEN),
        data: vec![0; OFFER_LEN],
        owner: ESC,
        executable: false,
    };
    w.b.set_account(k(80), forged.clone());
    let mut ix = take_offer_ix(ESC, k(TAKER), &w.offer);
    ix.accounts[2].pubkey = k(80);
    assert_eq!(
        run(&mut w.b, &[TAKER], vec![ix]),
        Err(Error::InvalidAccountData)
    );
    // even with a correct discriminator and fields, the address is not the derived one
    forged.data = w.b.account(&w.offer_key).unwrap().data.clone();
    w.b.set_account(k(81), forged);
    let mut ix = take_offer_ix(ESC, k(TAKER), &w.offer);
    ix.accounts[2].pubkey = k(81);
    assert!(run(&mut w.b, &[TAKER], vec![ix]).is_err());
}

#[test]
fn the_maker_cannot_take_their_own_offer_through_aliasing() {
    let mut w = world();
    ata(&mut w.b, k(MAKER), k(A), 0);
    let mut ix = take_offer_ix(ESC, k(MAKER), &w.offer);
    ix.accounts[0].is_signer = true;
    assert_eq!(
        run(&mut w.b, &[MAKER], vec![ix]),
        Err(Error::InvalidAccountData)
    );
}

#[test]
fn a_poor_taker_fails_atomically() {
    let mut w = world();
    let tb = associated_address(&k(TAKER), &k(B)).0;
    let mut t = get_token(&w.b, &tb);
    t.amount = 249;
    let lam = w.b.lamports(&tb);
    w.b.set_account(tb, token_account(&t, lam));
    assert_eq!(
        run(
            &mut w.b,
            &[TAKER],
            vec![take_offer_ix(ESC, k(TAKER), &w.offer)]
        ),
        Err(Error::InsufficientFunds)
    );
    assert_eq!(amount(&w.b, &w.vault), 400);
    assert!(w.b.account(&w.offer_key).is_some());
}

#[test]
fn many_offers_are_independent() {
    let mut b = base();
    make(&mut b, 1, 100, 10).unwrap();
    make(&mut b, 2, 200, 20).unwrap();
    let (o1, o2) = (
        offer_address(&ESC, &k(MAKER), 1).0,
        offer_address(&ESC, &k(MAKER), 2).0,
    );
    assert_ne!(o1, o2);
    let v1 = associated_address(&o1, &k(A)).0;
    let v2 = associated_address(&o2, &k(A)).0;
    assert_eq!((amount(&b, &v1), amount(&b, &v2)), (100, 200));
    let offer1 = Offer::unpack(&b.account(&o1).unwrap().data).unwrap();
    run(
        &mut b,
        &[TAKER],
        vec![take_offer_ix(ESC, k(TAKER), &offer1)],
    )
    .unwrap();
    assert!(b.account(&o1).is_none());
    assert_eq!(amount(&b, &v2), 200);
}

#[test]
fn unknown_instructions_are_refused() {
    let mut w = world();
    let mut ix = take_offer_ix(ESC, k(TAKER), &w.offer);
    ix.data[0] ^= 1;
    assert_eq!(
        run(&mut w.b, &[TAKER], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
    let mut ix = take_offer_ix(ESC, k(TAKER), &w.offer);
    ix.data.push(0);
    assert_eq!(
        run(&mut w.b, &[TAKER], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
    let mut ix = take_offer_ix(ESC, k(TAKER), &w.offer);
    ix.data.truncate(4);
    assert_eq!(
        run(&mut w.b, &[TAKER], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
}

#[test]
fn compute_units_stay_far_below_the_budget() {
    let mut b = base();
    let (offer, _) = offer_address(&ESC, &k(MAKER), 7);
    let r = run(
        &mut b,
        &[MAKER],
        vec![
            ata::create_ix(k(MAKER), offer, k(A), true),
            make_offer_ix(ESC, k(MAKER), k(A), k(B), 7, 400, 250),
        ],
    )
    .unwrap();
    assert!(
        r.units_used > 1500 && r.units_used < 20_000,
        "{}",
        r.units_used
    );
}
```

### `tests/gap_tests.rs`

Written after the first mutation run (§13, mutation table).

```rust
//! Tests added after the first mutation run showed what the earlier ones missed.
mod common;
use anchorlab::ata::{self, associated_address};
use anchorlab::escrow::*;
use anchorlab::token::{self, *};
use common::*;
use solanalab::account::{Account, rent_exempt_minimum};
use solanalab::instruction::Instruction;
use solanalab::pubkey::Pubkey;
use solanalab::runtime::{Bank, Error};

const A: u8 = 100;
const B: u8 = 101;
const MAKER: u8 = 1;
const TAKER: u8 = 2;

fn base() -> Bank {
    let mut b = bank();
    for n in [MAKER, TAKER, 3] {
        wallet(&mut b, n);
    }
    mint(&mut b, k(A), 6, k(MAKER), Some(k(MAKER)));
    mint(&mut b, k(B), 6, k(TAKER), Some(k(MAKER)));
    ata(&mut b, k(MAKER), k(A), 1000);
    ata(&mut b, k(MAKER), k(B), 0);
    ata(&mut b, k(TAKER), k(A), 0);
    ata(&mut b, k(TAKER), k(B), 500);
    b
}

fn live_offer(b: &mut Bank, id: u64) -> (Offer, Pubkey, Pubkey) {
    let (offer_key, bump) = offer_address(&ESC, &k(MAKER), id);
    run(
        b,
        &[MAKER],
        vec![
            ata::create_ix(k(MAKER), offer_key, k(A), true),
            make_offer_ix(ESC, k(MAKER), k(A), k(B), id, 400, 250),
        ],
    )
    .unwrap();
    let offer = Offer {
        maker: k(MAKER),
        mint_a: k(A),
        mint_b: k(B),
        id,
        amount_b: 250,
        bump,
    };
    let vault = associated_address(&offer_key, &k(A)).0;
    (offer, offer_key, vault)
}

fn raw_token(b: &mut Bank, key: Pubkey, mint_key: Pubkey, owner: Pubkey, amount: u64) {
    let mut t = TokenAccount::new(mint_key, owner);
    t.amount = amount;
    b.set_account(key, token_account(&t, rent_exempt_minimum(ACCOUNT_LEN)));
}

// ---------- pure token rules ----------

#[test]
fn minting_into_an_uninitialized_account_is_refused() {
    let mut m = Mint {
        mint_authority: Some(k(1)),
        supply: 0,
        decimals: 0,
        is_initialized: true,
        freeze_authority: None,
    };
    let mut dest = TokenAccount::new(k(50), k(2));
    dest.state = State::Uninitialized;
    assert_eq!(
        m.mint_to(&k(50), &mut dest, &k(1), 5),
        Err(TokenError::Uninitialized)
    );
    assert_eq!((m.supply, dest.amount), (0, 0));
}

#[test]
fn burning_the_whole_balance_is_allowed() {
    let mut m = Mint {
        mint_authority: Some(k(1)),
        supply: 10,
        decimals: 0,
        is_initialized: true,
        freeze_authority: None,
    };
    let mut acct = TokenAccount::new(k(50), k(2));
    acct.amount = 10;
    assert_eq!(m.burn(&k(50), &mut acct, &k(2), 10), Ok(()));
    assert_eq!((m.supply, acct.amount), (0, 0));
    assert_eq!(
        m.burn(&k(50), &mut acct, &k(2), 1),
        Err(TokenError::InsufficientFunds)
    );
}

#[test]
fn a_stranger_is_not_the_delegate() {
    let mut a = TokenAccount::new(k(50), k(2));
    a.amount = 10;
    a.delegate = Some(k(5));
    a.delegated_amount = 10;
    assert_eq!(a.authorize(&k(6), 1), Err(TokenError::OwnerMismatch));
    assert_eq!((a.delegate, a.delegated_amount), (Some(k(5)), 10));
}

#[test]
fn a_token_account_with_trailing_bytes_is_refused() {
    let mut d = TokenAccount::new(k(50), k(2)).pack();
    d.push(0);
    assert!(TokenAccount::unpack(&d).is_err());
    assert!(TokenAccount::unpack(&d[..164]).is_err());
}

// ---------- the token program through the bank ----------

#[test]
fn initialize_mint_needs_a_token_owned_account_and_exact_data() {
    let mut b = base();
    // an 82-byte account owned by another program
    b.set_account(
        k(60),
        Account {
            lamports: rent_exempt_minimum(MINT_LEN),
            data: vec![0; MINT_LEN],
            owner: k(9),
            executable: false,
        },
    );
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![token::initialize_mint(k(60), 6, k(MAKER), None)]
        ),
        Err(Error::InvalidAccountData)
    );
    // a "no freeze authority" flag with the length of a freeze authority is malformed
    b.set_account(
        k(61),
        Account {
            lamports: rent_exempt_minimum(MINT_LEN),
            data: vec![0; MINT_LEN],
            owner: TOKEN_PROGRAM_ID,
            executable: false,
        },
    );
    let mut ix = token::initialize_mint(k(61), 6, k(MAKER), Some(k(3)));
    ix.data[34] = 0;
    assert_eq!(
        run(&mut b, &[MAKER], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
}

#[test]
fn approve_on_a_frozen_account_is_refused_and_revoke_clears_the_allowance() {
    let mut b = base();
    let src = associated_address(&k(MAKER), &k(A)).0;
    run(
        &mut b,
        &[MAKER],
        vec![token::approve(src, k(3), k(MAKER), 50)],
    )
    .unwrap();
    let t = get_token(&b, &src);
    assert_eq!((t.delegate, t.delegated_amount), (Some(k(3)), 50));
    run(&mut b, &[MAKER], vec![token::revoke(src, k(MAKER))]).unwrap();
    let t = get_token(&b, &src);
    assert_eq!((t.delegate, t.delegated_amount), (None, 0));
    run(
        &mut b,
        &[MAKER],
        vec![token::freeze_account(src, k(A), k(MAKER), true)],
    )
    .unwrap();
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![token::approve(src, k(3), k(MAKER), 5)]
        ),
        Err(Error::InvalidAccountData)
    );
}

#[test]
fn freezing_needs_the_matching_mint_and_thawing_needs_a_frozen_account() {
    let mut b = base();
    let src = associated_address(&k(MAKER), &k(A)).0;
    // the account is mint A; mint B (whose freeze authority is also the maker) is the wrong mint
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![token::freeze_account(src, k(B), k(MAKER), true)]
        ),
        Err(Error::InvalidAccountData)
    );
    // thawing an account that is not frozen
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![token::freeze_account(src, k(A), k(MAKER), false)]
        ),
        Err(Error::InvalidAccountData)
    );
    // freezing twice
    run(
        &mut b,
        &[MAKER],
        vec![token::freeze_account(src, k(A), k(MAKER), true)],
    )
    .unwrap();
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![token::freeze_account(src, k(A), k(MAKER), true)]
        ),
        Err(Error::InvalidAccountData)
    );
    run(
        &mut b,
        &[MAKER],
        vec![token::freeze_account(src, k(A), k(MAKER), false)],
    )
    .unwrap();
}

#[test]
fn a_read_only_destination_cannot_be_credited() {
    let mut b = base();
    let src = associated_address(&k(MAKER), &k(A)).0;
    let dst = associated_address(&k(TAKER), &k(A)).0;
    let mut ix = token::transfer(src, dst, k(MAKER), 10);
    ix.accounts[1].is_writable = false;
    assert_eq!(run(&mut b, &[MAKER], vec![ix]), Err(Error::NotWritable(1)));
    assert_eq!(amount(&b, &src), 1000);
}

#[test]
fn mints_owned_by_another_program_are_refused() {
    let mut b = base();
    let src = associated_address(&k(MAKER), &k(A)).0;
    let dst = associated_address(&k(TAKER), &k(A)).0;
    let fake = Account {
        lamports: rent_exempt_minimum(MINT_LEN),
        data: get_mint_bytes(&b, &k(A)),
        owner: k(9),
        executable: false,
    };
    b.set_account(k(62), fake);
    raw_token(&mut b, k(63), k(62), k(MAKER), 5);
    raw_token(&mut b, k(64), k(62), k(TAKER), 0);
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![token::mint_to(k(62), k(64), k(MAKER), 1)]
        ),
        Err(Error::WrongOwner(0))
    );
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![token::burn(k(63), k(62), k(MAKER), 1)]
        ),
        Err(Error::WrongOwner(1))
    );
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![token::transfer_checked(k(63), k(62), k(64), k(MAKER), 1, 6)]
        ),
        Err(Error::WrongOwner(1))
    );
    let _ = (src, dst);
}

fn get_mint_bytes(b: &Bank, key: &Pubkey) -> Vec<u8> {
    b.account(key).unwrap().data.clone()
}

#[test]
fn closing_an_uninitialized_account_is_refused() {
    let mut b = base();
    b.set_account(
        k(65),
        Account {
            lamports: rent_exempt_minimum(ACCOUNT_LEN),
            data: vec![0; ACCOUNT_LEN],
            owner: TOKEN_PROGRAM_ID,
            executable: false,
        },
    );
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![token::close_account(k(65), k(MAKER), k(MAKER))]
        ),
        Err(Error::InvalidAccountData)
    );
}

// ---------- compute units are part of the model ----------

#[test]
fn exact_compute_units() {
    let mut b = base();
    let src = associated_address(&k(MAKER), &k(A)).0;
    let dst = associated_address(&k(TAKER), &k(A)).0;
    let r = run(
        &mut b,
        &[MAKER],
        vec![token::transfer(src, dst, k(MAKER), 1)],
    )
    .unwrap();
    assert_eq!(r.units_used, 300);
    let (offer_key, _) = offer_address(&ESC, &k(MAKER), 7);
    let r = run(
        &mut b,
        &[MAKER],
        vec![ata::create_ix(k(MAKER), offer_key, k(A), false)],
    )
    .unwrap();
    assert_eq!(r.units_used, 4150);
    let r = run(
        &mut b,
        &[MAKER],
        vec![make_offer_ix(ESC, k(MAKER), k(A), k(B), 7, 400, 250)],
    )
    .unwrap();
    assert_eq!(r.units_used, 4350);
    let offer = Offer {
        maker: k(MAKER),
        mint_a: k(A),
        mint_b: k(B),
        id: 7,
        amount_b: 250,
        bump: offer_address(&ESC, &k(MAKER), 7).1,
    };
    let r = run(&mut b, &[MAKER], vec![cancel_offer_ix(ESC, &offer)]).unwrap();
    assert_eq!(r.units_used, 3000);
    let mut b2 = base();
    live_offer(&mut b2, 7);
    let r = run(
        &mut b2,
        &[TAKER],
        vec![take_offer_ix(ESC, k(TAKER), &offer)],
    )
    .unwrap();
    assert_eq!(r.units_used, 4300);
}

// ---------- associated accounts ----------

#[test]
fn an_existing_account_for_an_unusable_mint_is_not_accepted_as_idempotent_success() {
    let mut b = base();
    // an uninitialized mint (right owner, zero bytes) with a token account already at the address
    b.set_account(
        k(70),
        Account {
            lamports: rent_exempt_minimum(MINT_LEN),
            data: vec![0; MINT_LEN],
            owner: TOKEN_PROGRAM_ID,
            executable: false,
        },
    );
    let addr = associated_address(&k(TAKER), &k(70)).0;
    raw_token(&mut b, addr, k(70), k(TAKER), 0);
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![ata::create_ix(k(MAKER), k(TAKER), k(70), true)]
        ),
        Err(Error::InvalidAccountData)
    );
    // a well-formed mint owned by some other program
    let mut data = get_mint_bytes(&b, &k(A));
    data.truncate(MINT_LEN);
    b.set_account(
        k(71),
        Account {
            lamports: rent_exempt_minimum(MINT_LEN),
            data,
            owner: k(9),
            executable: false,
        },
    );
    let addr = associated_address(&k(TAKER), &k(71)).0;
    raw_token(&mut b, addr, k(71), k(TAKER), 0);
    assert_eq!(
        run(
            &mut b,
            &[MAKER],
            vec![ata::create_ix(k(MAKER), k(TAKER), k(71), true)]
        ),
        Err(Error::InvalidAccountData)
    );
}

// ---------- the escrow ----------

fn fake_mint_world() -> (Bank, Pubkey, Pubkey) {
    let mut b = base();
    b.set_account(
        k(90),
        Account {
            lamports: rent_exempt_minimum(MINT_LEN),
            data: get_mint_bytes(&b, &k(A)),
            owner: k(9),
            executable: false,
        },
    );
    let (offer_key, _) = offer_address(&ESC, &k(MAKER), 1);
    let maker_ata = associated_address(&k(MAKER), &k(90)).0;
    let vault = associated_address(&offer_key, &k(90)).0;
    raw_token(&mut b, maker_ata, k(90), k(MAKER), 100);
    raw_token(&mut b, vault, k(90), offer_key, 0);
    (b, maker_ata, vault)
}

#[test]
fn make_offer_refuses_mints_the_token_program_does_not_own() {
    let (mut b, _, _) = fake_mint_world();
    let ix = make_offer_ix(ESC, k(MAKER), k(90), k(B), 1, 100, 10);
    assert_eq!(run(&mut b, &[MAKER], vec![ix]), Err(Error::WrongOwner(2)));
    assert!(b.account(&offer_address(&ESC, &k(MAKER), 1).0).is_none());
}

#[test]
fn escrow_token_accounts_must_be_owned_by_the_token_program() {
    let mut b = base();
    let (offer, _, _) = live_offer(&mut b, 7);
    // make: the maker's source account is a lookalike owned by someone else
    let mut b2 = base();
    let src = associated_address(&k(MAKER), &k(A)).0;
    let mut acct = b2.account(&src).unwrap().clone();
    acct.owner = k(9);
    b2.set_account(src, acct);
    let (offer_key, _) = offer_address(&ESC, &k(MAKER), 1);
    let r = run(
        &mut b2,
        &[MAKER],
        vec![
            ata::create_ix(k(MAKER), offer_key, k(A), true),
            make_offer_ix(ESC, k(MAKER), k(A), k(B), 1, 100, 10),
        ],
    );
    assert_eq!(r, Err(Error::WrongOwner(4)));
    // take: the maker's payment account is a lookalike owned by someone else
    let dst = associated_address(&k(MAKER), &k(B)).0;
    let mut acct = b.account(&dst).unwrap().clone();
    acct.owner = k(9);
    b.set_account(dst, acct);
    assert_eq!(
        run(&mut b, &[TAKER], vec![take_offer_ix(ESC, k(TAKER), &offer)]),
        Err(Error::WrongOwner(7))
    );
}

#[test]
fn a_vault_that_is_not_the_associated_account_is_refused() {
    // Eve funds a second token account owned by the offer PDA; releasing it would
    // close the offer and strand the real vault
    let mut b = base();
    let (offer, offer_key, vault) = live_offer(&mut b, 7);
    raw_token(&mut b, k(95), k(A), offer_key, 5);
    let mut take = take_offer_ix(ESC, k(TAKER), &offer);
    take.accounts[8].pubkey = k(95);
    assert_eq!(
        run(&mut b, &[TAKER], vec![take]),
        Err(Error::InvalidAccountData)
    );
    let mut cancel = cancel_offer_ix(ESC, &offer);
    cancel.accounts[4].pubkey = k(95);
    assert_eq!(
        run(&mut b, &[MAKER], vec![cancel]),
        Err(Error::InvalidAccountData)
    );
    assert_eq!(amount(&b, &vault), 400);
    assert!(b.account(&offer_key).is_some());
}

#[test]
fn the_offer_must_name_the_mint_it_is_released_in() {
    let mut b = base();
    let (offer, offer_key, _) = live_offer(&mut b, 7);
    mint(&mut b, k(96), 6, k(3), None);
    let fake_vault = associated_address(&offer_key, &k(96)).0;
    raw_token(&mut b, fake_vault, k(96), offer_key, 1);
    raw_token(
        &mut b,
        associated_address(&k(TAKER), &k(96)).0,
        k(96),
        k(TAKER),
        0,
    );
    let mut take = take_offer_ix(ESC, k(TAKER), &offer);
    take.accounts[3].pubkey = k(96);
    take.accounts[5].pubkey = associated_address(&k(TAKER), &k(96)).0;
    take.accounts[8].pubkey = fake_vault;
    assert_eq!(
        run(&mut b, &[TAKER], vec![take]),
        Err(Error::WrongAuthority)
    );
    raw_token(
        &mut b,
        associated_address(&k(MAKER), &k(96)).0,
        k(96),
        k(MAKER),
        0,
    );
    let mut cancel = cancel_offer_ix(ESC, &offer);
    cancel.accounts[2].pubkey = k(96);
    cancel.accounts[3].pubkey = associated_address(&k(MAKER), &k(96)).0;
    cancel.accounts[4].pubkey = fake_vault;
    assert_eq!(
        run(&mut b, &[MAKER], vec![cancel]),
        Err(Error::WrongAuthority)
    );
}

#[test]
fn an_offer_copied_to_another_address_is_not_a_valid_offer() {
    let mut b = base();
    let (offer, offer_key, _) = live_offer(&mut b, 7);
    let mut copy = b.account(&offer_key).unwrap().clone();
    copy.lamports = rent_exempt_minimum(OFFER_LEN);
    b.set_account(k(81), copy);
    let fake_vault = associated_address(&k(81), &k(A)).0;
    raw_token(&mut b, fake_vault, k(A), k(81), 5);
    let mut take = take_offer_ix(ESC, k(TAKER), &offer);
    take.accounts[2].pubkey = k(81);
    take.accounts[8].pubkey = fake_vault;
    assert_eq!(run(&mut b, &[TAKER], vec![take]), Err(Error::InvalidSeeds));
    let mut cancel = cancel_offer_ix(ESC, &offer);
    cancel.accounts[1].pubkey = k(81);
    cancel.accounts[4].pubkey = fake_vault;
    assert_eq!(
        run(&mut b, &[MAKER], vec![cancel]),
        Err(Error::InvalidSeeds)
    );
}

#[test]
fn cancel_needs_the_real_token_program_and_exact_data() {
    let mut b = base();
    let (offer, _, _) = live_offer(&mut b, 7);
    let mut ix = cancel_offer_ix(ESC, &offer);
    ix.accounts[5].pubkey = k(72);
    assert_eq!(
        run(&mut b, &[MAKER], vec![ix]),
        Err(Error::InvalidAccountData)
    );
    let mut ix = cancel_offer_ix(ESC, &offer);
    ix.data.push(0);
    assert_eq!(
        run(&mut b, &[MAKER], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
    let mut ix = take_offer_ix(ESC, k(TAKER), &offer);
    ix.data.push(0);
    assert_eq!(
        run(&mut b, &[TAKER], vec![ix]),
        Err(Error::InvalidInstructionData)
    );
}

#[test]
fn take_needs_a_writable_maker() {
    let mut b = base();
    let (offer, _, _) = live_offer(&mut b, 7);
    let mut ix = take_offer_ix(ESC, k(TAKER), &offer);
    ix.accounts[1].is_writable = false;
    assert_eq!(run(&mut b, &[TAKER], vec![ix]), Err(Error::NotWritable(1)));
}

#[test]
fn a_closed_offer_cannot_be_revived_inside_the_same_transaction() {
    // take, then make the same offer again, in one transaction: the closed account
    // must look like a fresh system account
    let mut b = base();
    let (offer, offer_key, _) = live_offer(&mut b, 7);
    let ixs: Vec<Instruction> = vec![
        take_offer_ix(ESC, k(TAKER), &offer),
        ata::create_ix(k(MAKER), offer_key, k(A), true),
        make_offer_ix(ESC, k(MAKER), k(A), k(B), 7, 100, 10),
    ];
    run(&mut b, &[MAKER, TAKER], ixs).unwrap();
    assert_eq!(
        Offer::unpack(&b.account(&offer_key).unwrap().data)
            .unwrap()
            .amount_b,
        10
    );
}

#[test]
fn closing_into_a_wallet_that_would_overflow_is_an_error() {
    let mut b = base();
    let (offer, offer_key, vault) = live_offer(&mut b, 7);
    let (vr, or) = (b.lamports(&vault), b.lamports(&offer_key));
    b.set_account(k(MAKER), Account::wallet(u64::MAX - vr - or + 1));
    assert_eq!(
        run(&mut b, &[TAKER], vec![take_offer_ix(ESC, k(TAKER), &offer)]),
        Err(Error::ArithmeticOverflow)
    );
}

#[test]
fn the_escrow_trusts_the_bump_it_stored_itself() {
    // Only the program can write an offer, and it always stores the canonical bump. If an
    // offer did exist at the address of a lower valid bump, the stored bump (not a fresh
    // search) decides the address, and the offer signs for itself with that bump.
    let mut b = base();
    let (offer, _, _) = live_offer(&mut b, 7);
    let seeds: [&[u8]; 3] = [b"offer", &k(MAKER).0, &7u64.to_le_bytes()];
    let (_, canonical) = solanalab::pubkey::find_program_address(&seeds, &ESC).unwrap();
    let (alt, alt_bump) = (0..canonical)
        .rev()
        .find_map(|bump| {
            let mut s = seeds.to_vec();
            let bs = [bump];
            s.push(&bs);
            solanalab::pubkey::create_program_address(&s, &ESC)
                .ok()
                .map(|a| (a, bump))
        })
        .expect("a lower valid bump exists");
    let moved = Offer {
        bump: alt_bump,
        ..offer.clone()
    };
    b.set_account(
        alt,
        Account {
            lamports: rent_exempt_minimum(OFFER_LEN),
            data: moved.pack(),
            owner: ESC,
            executable: false,
        },
    );
    let vault = associated_address(&alt, &k(A)).0;
    raw_token(&mut b, vault, k(A), alt, 30);
    let mut ix = cancel_offer_ix(ESC, &moved);
    ix.accounts[1].pubkey = alt;
    ix.accounts[4].pubkey = vault;
    let before = amount(&b, &associated_address(&k(MAKER), &k(A)).0);
    run(&mut b, &[MAKER], vec![ix]).unwrap();
    assert_eq!(
        amount(&b, &associated_address(&k(MAKER), &k(A)).0),
        before + 30
    );
}
```

### `tests/props.rs`

```rust
//! Random operations never create or destroy tokens, and a failed operation
//! changes nothing but the fee.
mod common;
use anchorlab::token::*;
use common::*;
use solanalab::account::Account;
use solanalab::pubkey::Pubkey;

struct Rng(u64);
impl Rng {
    fn next(&mut self) -> u64 {
        self.0 ^= self.0 << 13;
        self.0 ^= self.0 >> 7;
        self.0 ^= self.0 << 17;
        self.0
    }
    fn below(&mut self, n: u64) -> u64 {
        self.next() % n
    }
}

#[test]
fn random_token_operations_conserve_supply() {
    let mut successes = 0u32;
    for seed in 1..=200u64 {
        let mut rng = Rng(seed.wrapping_mul(0x9E37_79B9_7F4A_7C15) | 1);
        let mut b = bank();
        for n in 1..=4 {
            wallet(&mut b, n);
        }
        let mints = [k(100), k(101)];
        mint(&mut b, mints[0], 6, k(1), Some(k(2)));
        mint(&mut b, mints[1], 3, k(2), Some(k(2)));
        let accts: Vec<Pubkey> = (10..16).map(k).collect();
        for (i, a) in accts.iter().enumerate() {
            tacct(&mut b, *a, mints[i % 2], k(1 + (i as u8 % 4)), 0);
        }
        let total_lamports = b.total_supply();
        for _ in 0..60 {
            let (i, j) = (rng.below(6) as usize, rng.below(6) as usize);
            let signer = 1 + rng.below(4) as u8;
            let amt = rng.below(50);
            let m = rng.below(2) as usize;
            let ix = match rng.below(8) {
                0 => mint_to(mints[m], accts[i], k(signer), amt),
                1 | 2 => transfer(accts[i], accts[j], k(signer), amt),
                3 => burn(accts[i], mints[m], k(signer), amt),
                4 => approve(accts[i], k(1 + rng.below(4) as u8), k(signer), amt),
                5 => revoke(accts[i], k(signer)),
                6 => freeze_account(accts[i], mints[m], k(signer), rng.below(2) == 0),
                _ => transfer_checked(
                    accts[i],
                    mints[m],
                    accts[j],
                    k(signer),
                    amt,
                    if m == 0 { 6 } else { 3 },
                ),
            };
            let before: Vec<Option<Account>> =
                accts.iter().map(|a| b.account(a).cloned()).collect();
            let mint_before: Vec<Option<Account>> =
                mints.iter().map(|a| b.account(a).cloned()).collect();
            match run(&mut b, &[signer], vec![ix]) {
                Ok(_) => successes += 1,
                Err(_) => {
                    let after: Vec<Option<Account>> =
                        accts.iter().map(|a| b.account(a).cloned()).collect();
                    let mint_after: Vec<Option<Account>> =
                        mints.iter().map(|a| b.account(a).cloned()).collect();
                    assert_eq!(
                        (before, mint_before),
                        (after, mint_after),
                        "seed {seed}: a failure changed state"
                    );
                }
            }
            for (mi, mk) in mints.iter().enumerate() {
                let sum: u128 = accts
                    .iter()
                    .filter_map(|a| b.account(a))
                    .map(|a| TokenAccount::unpack(&a.data).unwrap())
                    .filter(|t| t.mint == *mk)
                    .map(|t| u128::from(t.amount))
                    .sum();
                assert_eq!(
                    sum,
                    u128::from(get_mint(&b, mk).supply),
                    "seed {seed} mint {mi}"
                );
            }
            assert_eq!(b.total_supply(), total_lamports, "seed {seed}: lamports");
        }
    }
    println!("successes: {successes}");
    assert!(successes > 500, "{successes}");
}

#[test]
fn the_property_test_actually_exercises_successes() {
    // across many seeds, a healthy fraction of random operations succeed
    let mut good = 0;
    let mut total = 0;
    for seed in 1..=40u64 {
        let mut rng = Rng(seed.wrapping_mul(0x9E37_79B9_7F4A_7C15) | 1);
        let mut b = bank();
        wallet(&mut b, 1);
        mint(&mut b, k(100), 6, k(1), None);
        tacct(&mut b, k(10), k(100), k(1), 0);
        tacct(&mut b, k(11), k(100), k(1), 0);
        for _ in 0..30 {
            let amt = 1 + rng.below(20);
            let ix = match rng.below(3) {
                0 => mint_to(k(100), k(10), k(1), amt),
                1 => transfer(k(10), k(11), k(1), amt),
                _ => burn(k(11), k(100), k(1), amt),
            };
            total += 1;
            good += usize::from(run(&mut b, &[1], vec![ix]).is_ok());
        }
    }
    assert!(good * 3 > total, "{good} of {total}");
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
//! Micro-benchmarks. Run with `cargo run --release --example bench`.
use anchorlab::ata::{self, ATA_PROGRAM_ID, AtaProgram, associated_address};
use anchorlab::discriminator;
use anchorlab::escrow::*;
use anchorlab::token::*;
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

fn time<F: FnMut()>(label: &str, iters: u32, mut f: F) {
    for _ in 0..(iters / 10).max(1) {
        f();
    }
    let t = Instant::now();
    for _ in 0..iters {
        f();
    }
    let ns = t.elapsed().as_nanos() as f64 / f64::from(iters);
    println!("{label:<44}{ns:>10.0} ns/op");
}

fn base() -> Bank {
    let mut b = Bank::new();
    b.add_program(TOKEN_PROGRAM_ID, Box::new(TokenProgram));
    b.add_program(ATA_PROGRAM_ID, Box::new(AtaProgram));
    b.add_program(ESC, Box::new(EscrowProgram));
    for n in 1..=3 {
        b.set_account(k(n), Account::wallet(1_000_000_000_000));
    }
    let m = |auth| Mint {
        mint_authority: Some(k(auth)),
        supply: 0,
        decimals: 6,
        is_initialized: true,
        freeze_authority: None,
    };
    b.set_account(k(100), mint_account(&m(1), rent_exempt_minimum(MINT_LEN)));
    b.set_account(k(101), mint_account(&m(2), rent_exempt_minimum(MINT_LEN)));
    for (w, mint, amt) in [
        (1u8, 100u8, 1_000_000u64),
        (1, 101, 0),
        (2, 100, 0),
        (2, 101, 1_000_000),
    ] {
        let mut t = TokenAccount::new(k(mint), k(w));
        t.amount = amt;
        b.set_account(
            associated_address(&k(w), &k(mint)).0,
            token_account(&t, rent_exempt_minimum(ACCOUNT_LEN)),
        );
    }
    b
}

fn main() {
    time("discriminator (sha256 of a name)", 200_000, || {
        black_box(discriminator::instruction(black_box("make_offer")));
    });
    time("associated_address (PDA search)", 20_000, || {
        black_box(associated_address(black_box(&k(1)), black_box(&k(100))));
    });
    time("TokenAccount pack + unpack", 500_000, || {
        let t = TokenAccount::new(k(1), k(2));
        black_box(TokenAccount::unpack(&black_box(t.pack())).unwrap());
    });
    let mut b = base();
    let a1 = associated_address(&k(1), &k(100)).0;
    let a2 = associated_address(&k(2), &k(100)).0;
    let t = Transaction {
        signers: vec![k(1)],
        instructions: vec![transfer(a1, a2, k(1), 1)],
    };
    time("process: token transfer", 50_000, || {
        black_box(b.process(&t).unwrap());
    });
    let t = Transaction {
        signers: vec![k(1)],
        instructions: vec![transfer_checked(a1, k(100), a2, k(1), 1, 6)],
    };
    time("process: token transfer_checked", 50_000, || {
        black_box(b.process(&t).unwrap());
    });
    let mut b = base();
    let (offer, _) = offer_address(&ESC, &k(1), 7);
    let make = Transaction {
        signers: vec![k(1)],
        instructions: vec![
            ata::create_ix(k(1), offer, k(100), true),
            make_offer_ix(ESC, k(1), k(100), k(101), 7, 400, 250),
        ],
    };
    let state = Offer {
        maker: k(1),
        mint_a: k(100),
        mint_b: k(101),
        id: 7,
        amount_b: 250,
        bump: 0,
    };
    let take = Transaction {
        signers: vec![k(2)],
        instructions: vec![take_offer_ix(ESC, k(2), &state)],
    };
    let cancel = Transaction {
        signers: vec![k(1)],
        instructions: vec![cancel_offer_ix(ESC, &state)],
    };
    time("process: make_offer + cancel_offer", 2_000, || {
        black_box(b.process(&make).unwrap());
        black_box(b.process(&cancel).unwrap());
    });
    time("process: make_offer + take_offer", 2_000, || {
        black_box(b.process(&make).unwrap());
        black_box(b.process(&take).unwrap());
    });
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **91 tests** (discriminators and layouts 9, validator 13, token rules 9, token program 11, associated accounts 10, escrow 15, gap tests 22, random operations 2) and 8 consecutive full runs without a failure (nothing uses sockets, real time or OS randomness, so there is nothing to flake).

**Mutation check ✓.** I made 192 small source edits (a "mutant") across the seven source modules, each in its own copy of the crate, and ran the whole test suite against each. **183 were caught by a failing test, 9 are equivalent (M51, M88, M89, M144, M145, M147, M148, M150, M153), and 0 does not compile (); none survive unexplained.** **What the first pass found.** Of the 192 mutants, 46 survived my first 66 tests. They fell into five groups. **Compute constants**: the 300, 200, 400 and 1,500 units were never asserted exactly, only bounded (`exact_compute_units` now pins 300, 4,150, 4,350, 3,000 and 4,300). **Checks in the token program that other checks hide**: a mint owned by another program (reached through `MintTo`, `Burn` and `TransferChecked`), freezing with the wrong mint, thawing an account that is not frozen, approving on a frozen account, a read-only destination, closing an uninitialized account, trailing bytes in a token account, the exact flag/length pairing of `InitializeMint`, the delegate being *a stranger*, burning the entire balance (the `<` against `<=` boundary), and `revoke` clearing the allowance. **Escrow checks that a later step repeats**: a vault that is not the associated address but is owned by the offer PDA (a real attack: it would close the offer and strand the real vault), an offer that names a different mint than the vault, an offer copied to another address, a fake token program in `cancel_offer`, a lookalike token account owned by someone else (the error index tells which layer refused it), a mint that the token program does not own in `make_offer`, extra instruction data on `cancel_offer` and `take_offer`, a read-only maker, and an overflow when the refund is added. **The associated-account program** accepted an existing account for an uninitialized or foreign mint when the call was idempotent. **The validator**: seven-byte data, the rule that only slots that *declare* `Mut` count as duplicates, and the stored bump against the searched one. Two of the cures needed care: closing an offer and making it again *in the same transaction* (it killed the mutant that skipped resetting the owner, because the account stays visible until the transaction ends); and the stored-bump mutant needed an offer placed by the test at the address of a lower valid bump, which documents that the escrow trusts the bump it stored itself. 22 tests in `tests/gap_tests.rs` and 3 in `tests/validate_tests.rs`. Nine mutants remain, each with a stated reason for being equivalent; none fails to compile. The column "change" is generated from the edit itself (the part that differs, with a little context, whitespace collapsed); `…` marks omitted text. The tests ran in the dev profile.

| # | module | change | result |
|---|---|---|---|
| M1 | `discriminator` | `h.update(b":");` → `∅` | caught by `discriminator_known_answers` |
| M2 | `discriminator` | `full[..8` → `full[1..9` | caught by `discriminator_known_answers` |
| M3 | `discriminator` | `disc("account` → `disc("accounts` | caught by `discriminator_known_answers` |
| M4 | `discriminator` | `disc("glob` → `disc("loc` | caught by `discriminator_known_answers` |
| M5 | `discriminator` | `h.update(namespace.as_bytes());` → `∅` | caught by `account_and_instruction_namespaces_differ` |
| M6 | `token` | `v.push(u8::from(self.is_initialized)` → `v.push(0` | caught by `a_created_account_is_immediately_usable` |
| M7 | `token` | `1 => tru` → `1 => fals` | caught by `a_created_account_is_immediately_usable` |
| M8 | `token` | `if d.len() !=` → `if d.len() <` | caught by `unpacking_is_strict` |
| M9 | `token` | `1 => Ok(Some(key)` → `1 => Ok(None` | caught by `mint_layout_is_82_bytes_with_known_offsets` |
| M10 | `token` | `0 => Ok(None` → `0 => Ok(Some(key)` | caught by `create_makes_the_associated_account` |
| M11 | `token` | `…!self.is_initialized \|\| dest.state == State::Uninitialized ` → `…!self.is_initialized ` | caught by `minting_into_an_uninitialized_account_is_refused` |
| M12 | `token` | `if !self.is_initialized \|\| ` → `if ` | caught by `minting_needs_the_authority_and_updates_supply_and_balance` |
| M13 | `token` | `if dest.mint != *mint_key` → `if false` | caught by `random_token_operations_conserve_supply` |
| M14 | `token` | `if dest.state == State::Frozen` → `if false` | caught by `minting_needs_the_authority_and_updates_supply_and_balance` |
| M15 | `token` | `if self.mint_authority != Some(*authority)` → `if false` | caught by `minting_needs_the_authority_and_updates_supply_and_balance` |
| M16 | `token` | `.supply .checked_add(amount) .ok_or(TokenError::Overflow)?` → `.supply .wrapping_add(amount)` | caught by `minting_needs_the_authority_and_updates_supply_and_balance` |
| M17 | `token` | `.amount .checked_add(amount) .ok_or(TokenError::Overflow)?` → `.amount .wrapping_add(amount)` | caught by `minting_needs_the_authority_and_updates_supply_and_balance` |
| M18 | `token` | `self.supply = supply; dest.amount = balance;` → `self.supply = supply;` | caught by `random_token_operations_conserve_supply` |
| M19 | `token` | `self.supply = supply; ` → `∅` | caught by `the_property_test_actually_exercises_successes` |
| M20 | `token` | `if account.mint != *mint_key` → `if false` | caught by `random_token_operations_conserve_supply` |
| M21 | `token` | `if account.state == State::Frozen` → `if false` | caught by `burning_reduces_balance_and_supply` |
| M22 | `token` | `…account.amount <` → `…account.amount <=` | caught by `burning_the_whole_balance_is_allowed` |
| M23 | `token` | `account.authorize(authority, amount)?;` → `∅` | caught by `burning_reduces_balance_and_supply` |
| M24 | `token` | `self.supply -= amount; // supply >= every balance, by construction` → `∅` | caught by `random_token_operations_conserve_supply` |
| M25 | `token` | `account.amount -= amount;` → `∅` | caught by `the_property_test_actually_exercises_successes` |
| M26 | `token` | `if self.owner == *authority` → `if true` | caught by `burning_reduces_balance_and_supply` |
| M27 | `token` | `…self.delegated_amount <` → `…self.delegated_amount <=` | caught by `delegates_spend_up_to_their_allowance` |
| M28 | `token` | `if self.delegated_amount == 0` → `if false` | caught by `delegates_spend_up_to_their_allowance` |
| M29 | `token` | `…self.delegate == Some(*authority` → `…self.delegate.is_some(` | caught by `a_stranger_is_not_the_delegate` |
| M30 | `token` | `…State::Uninitialized \|\| dest.state == State::Uninitialized ` → `…State::Uninitialized ` | caught by `mints_must_match_and_accounts_must_be_initialized` |
| M31 | `token` | `if self.state == State::Uninitialized \|\| ` → `if ` | caught by `mints_must_match_and_accounts_must_be_initialized` |
| M32 | `token` | `if self.mint != dest.mint` → `if false` | caught by `random_token_operations_conserve_supply` |
| M33 | `token` | `if self.mint != *mint_key` → `if false` | caught by `transfer_checked_compares_mint_and_decimals` |
| M34 | `token` | `if mint_decimals != given` → `if false` | caught by `transfer_checked_compares_mint_and_decimals` |
| M35 | `token` | `…State::Frozen \|\| dest.state == State::Frozen ` → `…State::Frozen ` | caught by `frozen_accounts_neither_send_nor_receive` |
| M36 | `token` | `if self.state == State::Frozen \|\| ` → `if ` | caught by `frozen_accounts_neither_send_nor_receive` |
| M37 | `token` | `…self.amount <` → `…self.amount <=` | caught by `cancel_returns_the_tokens_and_closes_everything` |
| M38 | `token` | `.amount .checked_add(amount) .ok_or(TokenError::Overflow)?` → `.amount .wrapping_add(amount)` | caught by `transfer_failures_change_nothing` |
| M39 | `token` | `dest.amount = new_dest;` → `∅` | caught by `a_created_account_is_immediately_usable` |
| M40 | `token` | `self.authorize(authority, amount)?;` → `∅` | caught by `delegates_spend_up_to_their_allowance` |
| M41 | `token` | `TokenError::OwnerMismatch => Error::WrongAuthority` → `TokenError::OwnerMismatch => Error::InvalidAccountData` | caught by `token_errors_map_to_runtime_errors` |
| M42 | `token` | `…=> Error::InsufficientFunds` → `…=> Error::InvalidAccountData` | caught by `a_poor_taker_fails_atomically` |
| M43 | `token` | `TokenError::Overflow => Error::ArithmeticOverflow` → `TokenError::Overflow => Error::InvalidAccountData` | caught by `token_errors_map_to_runtime_errors` |
| M44 | `token` | `TokenError::AlreadyInUse => Error::AlreadyInUse` → `TokenError::AlreadyInUse => Error::InvalidAccountData` | caught by `token_errors_map_to_runtime_errors` |
| M45 | `token` | `State::Frozen => 2` → `State::Frozen => 1` | caught by `token_account_layout_is_165_bytes_with_known_offsets` |
| M46 | `token` | `2 => State::Frozen` → `2 => State::Initialized` | caught by `token_account_layout_is_165_bytes_with_known_offsets` |
| M47 | `token` | `v.extend_from_slice(&self.delegated_amount.to_le_bytes());` → `∅` | caught by `a_created_account_is_immediately_usable` |
| M48 | `token` | `if d.len() !=` → `if d.len() <` | caught by `a_token_account_with_trailing_bytes_is_refused` |
| M49 | `token` | `d[108` → `d[107` | caught by `a_created_account_is_immediately_usable` |
| M50 | `token` | `if acct.owner != inv.program_id() \|\| acct.` → `if acct.` | caught by `initialize_mint_needs_a_token_owned_account_and_exact_data` |
| M51 | `token` | `…inv.program_id() \|\| acct.data.len() != MINT_LEN ` → `…inv.program_id() ` | **equivalent**: the length test in `InitializeMint` is a second line of defense; `Mint::unpack` refuses any other length with the same `InvalidAccountData` |
| M52 | `token` | `if Mint::unpack(&acct.data)?.is_initialized` → `if false` | caught by `initialize_mint_and_account` |
| M53 | `token` | `if TokenAccount::unpack(&acct.data)?.state != State::Uninitialized` → `if false` | caught by `initialize_mint_and_account` |
| M54 | `token` | `.filter(\|m\| m.is_initialized` → `.filter(\|_\| true` | caught by `initialize_mint_and_account` |
| M55 | `token` | `need_signer(inv, a)?;` → `∅` | caught by `only_the_signing_authority_moves_tokens` |
| M56 | `token` | `src.delegated_amount = amount` → `src.delegated_amount = 0` | caught by `approve_delegate_spends_and_revoke_ends_it` |
| M57 | `token` | `if src.owner != inv.key(2)?` → `if false` | caught by `approve_delegate_spends_and_revoke_ends_it` |
| M58 | `token` | `if src.owner != inv.key(1)?` → `if false` | caught by `approve_delegate_spends_and_revoke_ends_it` |
| M59 | `token` | `if src.state == State::Frozen` → `if false` | caught by `approve_on_a_frozen_account_is_refused_and_revoke_clears_the_allowance` |
| M60 | `token` | `src.delegate = None; src.delegated_amount = 0;` → `src.delegate = None;` | caught by `approve_on_a_frozen_account_is_refused_and_revoke_clears_the_allowance` |
| M61 | `token` | `if acct.amount != 0` → `if false` | caught by `close_needs_zero_balance_and_the_owner` |
| M62 | `token` | `if who != acct.close_authority.unwrap_or(acct.owner)` → `if false` | caught by `close_needs_zero_balance_and_the_owner` |
| M63 | `token` | `if inv.key(0)? == inv.key(1)?` → `if false` | caught by `close_needs_zero_balance_and_the_owner` |
| M64 | `token` | `if acct.mint != inv.key(1)?` → `if false` | caught by `freezing_needs_the_matching_mint_and_thawing_needs_a_frozen_account` |
| M65 | `token` | `if fa != inv.key(2)?` → `if false` | caught by `freeze_and_thaw` |
| M66 | `token` | `…State::Frozen) \| (11, State::Initialized) ` → `…State::Frozen) ` | caught by `freezing_needs_the_matching_mint_and_thawing_needs_a_frozen_account` |
| M67 | `token` | `if data.len() !=` → `if data.len() <` | caught by `malformed_instruction_data_is_refused` |
| M68 | `token` | `if !inv.is_writable(i)?` → `if false` | caught by `a_read_only_destination_cannot_be_credited` |
| M69 | `token` | `if a.owner != inv.program_id()` → `if false` | caught by `accounts_must_be_owned_by_the_token_program` |
| M70 | `token` | `if a.owner != inv.program_id()` → `if false` | caught by `mints_owned_by_another_program_are_refused` |
| M71 | `token` | `inv.consume(COST)?;` → `∅` | caught by `exact_compute_units` |
| M72 | `token` | `&&` → `\|\|` | caught by `initialize_mint_and_account` |
| M73 | `token` | `0 if data.len() == 35 ` → `0 ` | caught by `initialize_mint_needs_a_token_owned_account_and_exact_data` |
| M74 | `token` | `return Ok(` → `return store(inv, s, src.pack` | caught by `random_token_operations_conserve_supply` |
| M75 | `token` | `if acct.state == State::Uninitialized { return Err(TokenError::Unin…` → `if acct.` | caught by `closing_an_uninitialized_account_is_refused` |
| M76 | `token` | `…COST: u64 = 300` → `…COST: u64 = 301` | caught by `exact_compute_units` |
| M77 | `ata` | `[1] => tru` → `[1] => fals` | caught by `create_twice_is_refused_but_idempotent_create_is_not` |
| M78 | `ata` | `if !inv.is_signer(0)?` → `if false` | caught by `payer_signature_and_real_programs_are_required` |
| M79 | `ata` | `if inv.key(4)? != Pubkey::default() \|\| inv.key(` → `if inv.key(` | caught by `payer_signature_and_real_programs_are_required` |
| M80 | `ata` | `…Pubkey::default() \|\| inv.key(5)? != TOKEN_PROGRAM_ID ` → `…Pubkey::default() ` | caught by `payer_signature_and_real_programs_are_required` |
| M81 | `ata` | `…TOKEN_PROGRAM_ID \|\| !token::Mint::unpack(&m.data)?.is_initialized ` → `…TOKEN_PROGRAM_ID ` | caught by `an_existing_account_for_an_unusable_mint_is_not_accepted_as_idempotent_success` |
| M82 | `ata` | `if m.owner != TOKEN_PROGRAM_ID \|\| ` → `if ` | caught by `an_existing_account_for_an_unusable_mint_is_not_accepted_as_idempotent_success` |
| M83 | `ata` | `if expected != ata` → `if false` | caught by `the_address_must_be_the_derived_one` |
| M84 | `ata` | `if existing.owner == TOKEN_PROGRAM_ID` → `if false` | caught by `create_twice_is_refused_but_idempotent_create_is_not` |
| M85 | `ata` | `if !` → `if ` | caught by `create_twice_is_refused_but_idempotent_create_is_not` |
| M86 | `ata` | `…t.owner != wallet \|\| t.mint != mint ` → `…t.owner != wallet ` | caught by `idempotent_create_on_a_foreign_account_is_refused` |
| M87 | `ata` | `if t.owner != wallet \|\| t.` → `if t.` | caught by `idempotent_create_on_a_foreign_account_is_refused` |
| M88 | `ata` | `if !existing.data.is_empty() \|\| ` → `if ` | **equivalent**: an address holding data is also refused by the system program's `CreateAccount` with the same `AlreadyInUse` |
| M89 | `ata` | `…!existing.data.is_empty() \|\| existing.owner != Pubkey::default() ` → `…!existing.data.is_empty() ` | **equivalent**: an address with a foreign owner is also refused by the system program's `CreateAccount` with the same `AlreadyInUse` |
| M90 | `ata` | `…ACCOUNT_LEN).saturating_sub(existing.lamports)` → `…ACCOUNT_LEN)` | caught by `create_tops_up_a_prefunded_address` |
| M91 | `ata` | `inv.invoke(&create, &[seeds` → `inv.invoke(&create, &[` | caught by `a_created_account_is_immediately_usable` |
| M92 | `ata` | `inv.invoke(&token::initialize_account(ata, mint, wallet), &[]` → `Ok(()` | caught by `a_created_account_is_immediately_usable` |
| M93 | `ata` | `&wallet.0, &TOKEN_PROGRAM_ID.0, &min` → `&mint.0, &TOKEN_PROGRAM_ID.0, &walle` | caught by `a_created_account_is_immediately_usable` |
| M94 | `ata` | `&wallet.0, &TOKEN_PROGRAM_ID.0, &min` → `&mint.0, &TOKEN_PROGRAM_ID.0, &walle` | caught by `a_created_account_is_immediately_usable` |
| M95 | `ata` | `inv.consume(1500)?; // the PDA search` → `∅` | caught by `exact_compute_units` |
| M96 | `ata` | `create_account_data(lamports, ACCOUNT_LEN as u64` → `create_account_data(lamports, 100` | caught by `a_created_account_is_immediately_usable` |
| M97 | `ata` | `[] => fals` → `[] => tru` | caught by `create_twice_is_refused_but_idempotent_create_is_not` |
| M98 | `ata` | `AccountMeta::writable(ata, tru` → `AccountMeta::writable(ata, fals` | caught by `a_created_account_is_immediately_usable` |
| M99 | `ata` | `…COST: u64 = 200` → `…COST: u64 = 201` | caught by `exact_compute_units` |
| M100 | `validate` | `if inv.len() <` → `if inv.len() <=` | caught by `a_poor_taker_fails_atomically` |
| M101 | `validate` | `if !` → `if ` | caught by `a_lookalike_offer_cannot_be_forged` |
| M102 | `validate` | `if !inv.is_writable(i).expect("i")` → `if false` | caught by `signer_and_mut` |
| M103 | `validate` | `if key !` → `if key =` | caught by `a_lookalike_offer_cannot_be_forged` |
| M104 | `validate` | `if key !` → `if key =` | caught by `a_poor_taker_fails_atomically` |
| M105 | `validate` | `if !acct.executabl` → `if fals` | caught by `address_and_program_and_owner` |
| M106 | `validate` | `if acct.owner != *o` → `if false` | caught by `make_offer_validation` |
| M107 | `validate` | `if acct.owner != *owner` → `if false` | caught by `typed_accounts_check_owner_discriminator_and_length` |
| M108 | `validate` | `…acct.data.len() < 8` → `…acct.data.len() < 7` | caught by `a_seven_byte_account_has_no_discriminator` |
| M109 | `validate` | `if acct.data[..8] != disc[..]` → `if false` | caught by `a_lookalike_offer_cannot_be_forged` |
| M110 | `validate` | `if acct.data.len() != *len` → `if false` | caught by `typed_accounts_check_owner_discriminator_and_length` |
| M111 | `validate` | `if stored != Some(&want.0[..])` → `if false` | caught by `cancel_is_for_the_maker_only` |
| M112 | `validate` | `…*offset..*offset + 32` → `…*offset..*offset + 31` | caught by `a_poor_taker_fails_atomically` |
| M113 | `validate` | `Seed::Bytes(b) => b.clone` → `Seed::Bytes(_) => Vec::new` | caught by `a_poor_taker_fails_atomically` |
| M114 | `validate` | `.get(*o..*o + *l` → `.get(*o..*o + *l + 1` | caught by `a_poor_taker_fails_atomically` |
| M115 | `validate` | `…program).ok().map(\|x\| x.0` → `…program).ok().map(\|_\| key` | caught by `seeds_with_canonical_stored_and_own_data_parts` |
| M116 | `validate` | `…acct.data.get(*o` → `…acct.data.get(*o + 1` | caught by `a_poor_taker_fails_atomically` |
| M117 | `validate` | `if derived != Some(key)` → `if false` | caught by `seeds_with_canonical_stored_and_own_data_parts` |
| M118 | `validate` | `acct.data.get(0..32` → `acct.data.get(32..64` | caught by `a_lookalike_offer_cannot_be_forged` |
| M119 | `validate` | `acct.data.get(32..64` → `acct.data.get(0..32` | caught by `a_lookalike_offer_cannot_be_forged` |
| M120 | `validate` | `…!acct.data.is_empty() \|\| acct.owner != Pubkey::default() ` → `…!acct.data.is_empty() ` | caught by `uninit_means_empty_and_system_owned` |
| M121 | `validate` | `if !acct.data.is_empty() \|\| ` → `if ` | caught by `uninit_means_empty_and_system_owned` |
| M122 | `validate` | `if is_mut` → `if false` | caught by `the_maker_cannot_take_their_own_offer_through_aliasing` |
| M123 | `validate` | `.take(i` → `.take(i + 1` | caught by `a_poor_taker_fails_atomically` |
| M124 | `validate` | `if other_mut && ` → `if ` | caught by `only_declared_mutable_slots_count_for_duplicates` |
| M125 | `validate` | `err(2000` → `err(2001` | caught by `signer_and_mut` |
| M126 | `validate` | `err(2002` → `err(2003` | caught by `signer_and_mut` |
| M127 | `validate` | `err(3002` → `err(3003` | caught by `typed_accounts_check_owner_discriminator_and_length` |
| M128 | `validate` | `…Error::NotWritable(e.account` → `…Error::NotWritable(0` | caught by `anchor_errors_map_to_runtime_errors` |
| M129 | `validate` | `"ConstraintHasOne" => Error::WrongAuthority` → `"ConstraintHasOne" => Error::InvalidSeeds` | caught by `cancel_is_for_the_maker_only` |
| M130 | `validate` | `"ConstraintInit" => Error::AlreadyInUse` → `"ConstraintInit" => Error::InvalidAccountData` | caught by `make_twice_with_the_same_id_is_refused_and_leaves_the_vault_alone` |
| M131 | `validate` | `…"AccountNotEnoughKeys", inv.len()` → `…"AccountNotEnoughKeys", 0` | caught by `too_few_accounts_and_check_order` |
| M132 | `escrow` | `…MAKER: usize = 8` → `…MAKER: usize = 9` | caught by `offer_packs_and_unpacks_strictly` |
| M133 | `escrow` | `…MINT_B: usize = 72` → `…MINT_B: usize = 73` | caught by `offer_packs_and_unpacks_strictly` |
| M134 | `escrow` | `…ID: usize = 104` → `…ID: usize = 105` | caught by `offer_packs_and_unpacks_strictly` |
| M135 | `escrow` | `…AMOUNT_B: usize = 112` → `…AMOUNT_B: usize = 113` | caught by `offer_packs_and_unpacks_strictly` |
| M136 | `escrow` | `…BUMP: usize = 120` → `…BUMP: usize = 119` | caught by `offer_packs_and_unpacks_strictly` |
| M137 | `escrow` | `if d.len() !=` → `if d.len() >` | caught by `offer_packs_and_unpacks_strictly` |
| M138 | `escrow` | `if data.len() !=` → `if data.len() <` | caught by `make_offer_validation` |
| M139 | `escrow` | `…amount_a == 0 \|\| amount_b == 0 ` → `…amount_a == 0 ` | caught by `make_offer_validation` |
| M140 | `escrow` | `if amount_a == 0 \|\| amount_` → `if amount_` | caught by `make_offer_validation` |
| M141 | `escrow` | `…vec![Check::Signer, Check::` → `…vec![Check::` | caught by `make_offer_validation` |
| M142 | `escrow` | `…vec![Check::Signer, Check::` → `…vec![Check::` | caught by `cancel_is_for_the_maker_only` |
| M143 | `escrow` | `…vec![Check::Signer, Check::` → `…vec![Check::` | caught by `take_offer_attacks` |
| M144 | `escrow` | `…Check::Address(offer_key), Check::Uninit` → `…Check::Address(offer_key)` | **equivalent**: the `Uninit` check is repeated by the system program's `CreateAccount`, which gives the same `AlreadyInUse` |
| M145 | `escrow` | `vec![Check::Mut, Check::Address(offer_key), Check::` → `vec![Check::Mut, Check::` | **equivalent**: a wrong offer address fails at the next step, because the CPI names the derived address, which is not among the instruction's accounts, and the error is the same `InvalidAccountData` |
| M146 | `escrow` | `…"mint_a", vec![Check::Owner(TOKEN_PROGRAM_ID)` → `…"mint_a", vec![` | caught by `make_offer_refuses_mints_the_token_program_does_not_own` |
| M147 | `escrow` | `…"mint_a", vec![Check::Owner(TOKEN_PROGRAM_ID)` → `…"mint_a", vec![` | **equivalent** (unreachable): `take_offer` also requires that the mint account is the key stored in the offer, and `make_offer` stored only mints that passed the owner check |
| M148 | `escrow` | `…"mint_a", vec![Check::Owner(TOKEN_PROGRAM_ID)` → `…"mint_a", vec![` | **equivalent** (unreachable): the same argument for `cancel_offer` |
| M149 | `escrow` | `…"mint_b", vec![Check::Owner(TOKEN_PROGRAM_ID)` → `…"mint_b", vec![` | caught by `make_offer_validation` |
| M150 | `escrow` | `…"mint_b", vec![Check::Owner(TOKEN_PROGRAM_ID)` → `…"mint_b", vec![` | **equivalent** (unreachable): the same argument for mint B in `take_offer` |
| M151 | `escrow` | `…token_acct(2, 0` → `…token_acct(2, 1` | caught by `a_lookalike_offer_cannot_be_forged` |
| M152 | `escrow` | `…token_acct(2, 0` → `…token_acct(2, 1` | caught by `cancel_returns_the_tokens_and_closes_everything` |
| M153 | `escrow` | `Check::TokenMint(mint)` → `Check::Mut` | **equivalent**: every escrow token account is then used in a token transfer, whose own mint comparison fails with the same `InvalidAccountData` |
| M154 | `escrow` | `Check::TokenOwner(owner)` → `Check::Mut` | caught by `make_offer_validation` |
| M155 | `escrow` | `Check::Owner(TOKEN_PROGRAM_ID), Check::` → `Check::` | caught by `escrow_token_accounts_must_be_owned_by_the_token_program` |
| M156 | `escrow` | `vec![Check::Address(vault_key)` → `vec![` | caught by `make_offer_validation` |
| M157 | `escrow` | `vec![Check::Address(vault_key)` → `vec![` | caught by `a_vault_that_is_not_the_associated_account_is_refused` |
| M158 | `escrow` | `vec![Check::Address(vault_key)` → `vec![` | caught by `a_vault_that_is_not_the_associated_account_is_refused` |
| M159 | `escrow` | `Check::HasOne { offset: MAKER, target: 1, },` → `∅` | caught by `take_offer_attacks` |
| M160 | `escrow` | `Check::HasOne { offset: MINT_A, target: 3, },` → `∅` | caught by `the_offer_must_name_the_mint_it_is_released_in` |
| M161 | `escrow` | `Check::HasOne { offset: MINT_B, target: 4, },` → `∅` | caught by `take_offer_attacks` |
| M162 | `escrow` | `Check::HasOne { offset: MAKER, target: 0, },` → `∅` | caught by `cancel_is_for_the_maker_only` |
| M163 | `escrow` | `Check::HasOne { offset: MINT_A, target: 2, },` → `∅` | caught by `the_offer_must_name_the_mint_it_is_released_in` |
| M164 | `escrow` | `offer_pda_check(me, 1),` → `∅` | caught by `an_offer_copied_to_another_address_is_not_a_valid_offer` |
| M165 | `escrow` | `offer_pda_check(me, 0),` → `∅` | caught by `an_offer_copied_to_another_address_is_not_a_valid_offer` |
| M166 | `escrow` | `Seed::Bytes(b"offer` → `Seed::Bytes(b"offe` | caught by `a_poor_taker_fails_atomically` |
| M167 | `escrow` | `bump: Bump::Stored(BUMP)` → `bump: Bump::Canonical` | caught by `the_escrow_trusts_the_bump_it_stored_itself` |
| M168 | `escrow` | `…"token_program", vec![Check::Program(TOKEN_PROGRAM_ID)` → `…"token_program", vec![` | caught by `make_offer_validation` |
| M169 | `escrow` | `…"token_program", vec![Check::Program(TOKEN_PROGRAM_ID)` → `…"token_program", vec![` | caught by `take_offer_attacks` |
| M170 | `escrow` | `…"token_program", vec![Check::Program(TOKEN_PROGRAM_ID)` → `…"token_program", vec![` | caught by `cancel_needs_the_real_token_program_and_exact_data` |
| M171 | `escrow` | `…"system_program", vec![Check::Program(Pubkey::default())` → `…"system_program", vec![` | caught by `make_offer_validation` |
| M172 | `escrow` | `…"maker", vec![Check::Mut` → `…"maker", vec![` | caught by `take_needs_a_writable_maker` |
| M173 | `escrow` | `…token_acct(3, 0` → `…token_acct(3, 1` | caught by `a_poor_taker_fails_atomically` |
| M174 | `escrow` | `…token_acct(4, 0` → `…token_acct(4, 1` | caught by `a_poor_taker_fails_atomically` |
| M175 | `escrow` | `…token_acct(4, 1` → `…token_acct(4, 0` | caught by `a_poor_taker_fails_atomically` |
| M176 | `escrow` | `offer.amount_b` → `offer.amount_b - 1` | caught by `a_poor_taker_fails_atomically` |
| M177 | `escrow` | `…vault_amount` → `…vault_amount - 1` | caught by `a_second_take_or_cancel_finds_nothing` |
| M178 | `escrow` | `inv.invoke(&close, &[&seed_refs])?;` → `∅` | caught by `take_offer_swaps_closes_and_refunds_rent` |
| M179 | `escrow` | `inv.invoke(&close, &[&seed_refs])?;` → `∅` | caught by `cancel_returns_the_tokens_and_closes_everything` |
| M180 | `escrow` | `close_offer(inv, 2, 1` → `Ok(()` | caught by `many_offers_are_independent` |
| M181 | `escrow` | `close_offer(inv, 1, 0` → `Ok(()` | caught by `cancel_returns_the_tokens_and_closes_everything` |
| M182 | `escrow` | `a.owner = Pubkey::default();` → `∅` | caught by `a_closed_offer_cannot_be_revived_inside_the_same_transaction` |
| M183 | `escrow` | `a.data = Vec::new();` → `∅` | caught by `a_second_take_or_cancel_finds_nothing` |
| M184 | `escrow` | `inv.consume(1500)?; // two PDA searches would be charged here by th…` → `∅` | caught by `exact_compute_units` |
| M185 | `escrow` | `.saturating_sub(inv.account(1)?.lamports)` → `∅` | caught by `a_prefunded_offer_address_still_works` |
| M186 | `escrow` | `…OFFER_LEN as u64, &me` → `…OFFER_LEN as u64, &TOKEN_PROGRAM_ID` | caught by `a_poor_taker_fails_atomically` |
| M187 | `escrow` | `mint_b: inv.key(3` → `mint_b: inv.key(2` | caught by `a_poor_taker_fails_atomically` |
| M188 | `escrow` | `…maker, amount_a` → `…maker, amount_b` | caught by `a_poor_taker_fails_atomically` |
| M189 | `escrow` | `…"take_offer") && data.len() == 8` → `…"take_offer")` | caught by `unknown_instructions_are_refused` |
| M190 | `escrow` | `…"cancel_offer") && data.len() == 8` → `…"cancel_offer")` | caught by `cancel_needs_the_real_token_program_and_exact_data` |
| M191 | `escrow` | `inv.consume(COST)?;` → `∅` | caught by `exact_compute_units` |
| M192 | `escrow` | `let total = inv .account(to)? .lamports .checked_add(lamports) .ok_…` → `let total = inv.account(to)?.lamports.wrapping_add(lamports)` | caught by `closing_into_a_wallet_that_would_overflow_is_an_error` |

What the table does not show: it tests *these* edits, not every possible bug; the equivalent mutants could hide a real gap if my reasoning about them is wrong; and passing every mutant says nothing about the real runtime, about security or about behavior on a validator.

**Design notes.**

- Rules are pure functions on plain structs; programs are thin shells that load, call and store. Constraints are data, validated before any logic runs.
- Every parser is strict (exact lengths, valid tags) and returns an error on short input; no test input makes any of them panic.
- Failure leaves no trace: every program path validates and checks before it writes, and the runtime rolls back the rest (✓ the random test asserts byte-for-byte equality after every failed operation).
- Addresses, discriminators and bumps were checked against independent Python computations.
- Not handled: multisig, wrapped SOL, `SetAuthority`, Token-2022 extensions, account reallocation, events, an IDL and everything in §6 and §9.

</details>

---

## 14. Key takeaways

- Most Solana program bugs are missing account checks, so Anchor's central idea is to declare each account's constraints once and check them all before the logic runs; the model is a list of `Check`s per account, and each check has a test that breaks only it (✓).
- A discriminator (the first 8 bytes of a hash of the type name) is what stops one account type from standing in for another of the same size and owner; zeroed lookalikes are refused (✓ `AccountDiscriminatorMismatch`, and the debugging challenge's ticket-as-pool).
- A token is one mint account (82 bytes) plus one token account per holder (165 bytes); a token account has two owners that must not be confused: the program that owns the account and the wallet or PDA that controls the balance (✓ exact offsets; strict decoders).
- Keep the rules pure, check everything before changing anything, and test invariants with random operations: supply equals the sum of balances and a failed operation changes nothing (✓ 200 seeds, 12,000 operations; a self-transfer bug found by a unit test).
- Use the checked token instructions for anything that moves value for a user, and refuse vaults with a delegate or close authority you did not set.
- An associated token account is a PDA of wallet, token program and mint; creating it is two CPIs, can be made idempotent, and must handle a prefunded address (✓).
- An escrow vault owned by a PDA cannot be moved by any key; the program moves it by signing token CPIs with the offer's seeds, and every account the instruction receives (taker, maker, mints, vault, offer, programs) is validated first (✓ twelve attacks, each refused with no state change).
- Validation checks identities; amounts, rounding, fees and ordering are still your logic and still money (exercises 7 and 8).
- Mutation testing here pointed at real gaps: 37 of the 46 first survivors were real gaps and were closed with 25 new tests, among them a vault that is not the associated account, a mint that the token program does not own, and a closed offer revived within one transaction; 9 mutants remain, each with a stated reason for being equivalent.

**Connections.** [Lesson 53](53_solana_model.md) provided the runtime this crate runs in unchanged: ownership rules, PDAs with canonical bumps, CPI with PDA signing and the closing recipe; every check in §2.3 is the programmatic form of the checklist in that lesson's §4.2. [Lesson 52](52_web3_reliability.md) supplied the idempotency and simulation habits (the idempotent create, the random-operation test). [Lesson 48](48_crypto_primitives.md) supplied the hashing and domain separation used by discriminators, and [Lesson 35](35_proc_macros.md) is the background for how Anchor's derive macros generate this code. Lesson 55 covers security review, testing and deployment of Solana programs.

---

## 15. References

- Anchor documentation (accounts, constraints, errors, `declare_id!`, space, IDL): <https://www.anchor-lang.com/docs>
- The SPL token program: source and documentation: <https://github.com/solana-program/token> and <https://spl.solana.com/token>
- The associated token account program and the Token-2022 program and its extensions: <https://github.com/solana-program/associated-token-account>, <https://github.com/solana-program/token-2022>
- Solana documentation: accounts, programs, PDAs and CPI: <https://solana.com/docs>
- Public write-ups of Solana vulnerabilities by auditors: missing owner checks, signer checks, type cosplay, re-initialization, duplicate mutable accounts, bump canonicalization (not consulted; search for current ones).
- SHA-256 (FIPS 180-4), used for discriminators: <https://csrc.nist.gov/publications/detail/fips/180/4/final>

The URLs and titles above are the standard locations; I did not open them during this session (the sandbox cannot reach documentation sites), so if a link has moved, search for the title. **Unchecked or from memory:** everything about the **real** Anchor and SPL programs: the discriminator derivation (`sha256("account:Name")[..8]`, `sha256("global:name")[..8]`; only the value `afaf6d1f0d989bed` for `initialize` agrees with what I remember of real programs), the numbers of Anchor's error codes, the account layouts of the real mint and token account (82 and 165 bytes) and their field order, the instruction tags, the existence and behavior of `TransferChecked`, delegates, freezing and closing, the program ids (decoded from base58 text I remembered, ✓ 32 bytes each), the associated-token-account seeds, Anchor's duplicate-mutable-account check, the `init`/`zero`/`close`/`has_one` constraint semantics, the claims about Token-2022 extensions, `UncheckedAccount` guidance, and the public history of Solana vulnerabilities. **Environment caveats:** timings are single-machine micro-measurements (three runs of one example) on a shared virtual machine with two cores; nothing here ran on a validator, devnet or with the Anchor toolchain, so no figure applies to a real deployment. The discriminators, derived addresses and bumps were verified against independent Python programs; the exercises' code in §10 and the two versions in §11 were compiled and run in a scratch crate (✓ 17 tests, `cargo clippy --all-targets -- -D warnings` clean). Measurements and outcomes marked ✓ were observed with `rustc 1.97.0`, `sha2 0.11.0` and the Lesson 53 crate (`curve25519-dalek 5.0.0`). **No part of this lesson makes any system secure.**
