---
layout: default
title: "Lesson 53 — Solana execution model: accounts, programs, instructions, PDAs, CPI"
---

# Lesson 53 — Solana execution model: accounts, programs, instructions, PDAs, CPI

**Module 17 · Solana (first lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `sha2 0.11.0` and `curve25519-dalek 5.0.0` (22 packages in `Cargo.lock`, the crate itself included)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** Solana is a blockchain whose programs are **stateless**: a program is code, and everything it reads or writes lives in **accounts** that the *caller lists up front*. That one design decision explains most of what is unusual about writing Solana programs: why every instruction names its accounts, why a program must **check** every account it is handed, why **program-derived addresses** (PDAs) exist, why programs call each other with **cross-program invocation** (CPI) and not with function calls, and why transactions have hard limits on size and compute. The crate `solanalab` is a **teaching model of the runtime**, written in plain Rust with no Solana SDK: accounts with owners and rent, base58 keys, instructions and transactions with their wire-size arithmetic, an atomic transaction processor that **verifies after every instruction that the program changed only what it was allowed to change**, PDA derivation with a real curve-point check, CPI with privilege propagation, depth and re-entrancy limits, a compute meter, a system program and a small counter program. It is **not** the real runtime: it does not run BPF/SBF bytecode, does not schedule transactions in parallel, has no slots, no blockhashes, no signatures to verify (a transaction simply *lists* its signers), no sysvars, no address lookup tables, no priority fees, no program upgrade mechanism and no token program. Numbers such as the fee per signature, the rent rate, the CPI depth, the compute budget and the 1,232-byte transaction limit are **from memory** of the Solana documentation and may have changed; I could not open the documentation from the sandbox, and I did not install or run the Solana toolchain. **Nothing here is audited or secures anything.** Rust removes memory-safety bugs in your program; the failures in this lesson (a missing signer check, an account whose owner was never verified, a withdrawal that wraps around, a PDA derived with a non-canonical bump) are logic errors that compile cleanly. Lesson 54 covers Anchor and the token programs, and Lesson 55 security, testing and deployment; here I stay with the model.

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation and standards sites were not, so statements about the real runtime are labeled "from memory". Expected values for base58, for PDA derivation (which bumps are on the curve for which seeds) and for byte counts were computed by **separate Python programs** or by hand (✓), not by the code under test. Timings are single-machine micro-measurements.

Builds on [Lesson 52](52_web3_reliability.md) (signing, nonces, idempotency, simulation), [Lesson 51](51_rpc_nodes_indexers.md) (RPC and uncertain writes), [Lesson 50](50_networking_determinism.md) (limits, canonical encodings, sans-IO), [Lesson 49](49_merkle_transactions_consensus.md) (transactions, state, atomicity), [Lesson 48](48_crypto_primitives.md) (hashes, Ed25519, domain separation), [Lesson 46](46_databases_config_auth_jobs.md), [Lesson 41](41_benchmarking_profiling.md), [Lesson 40](40_property_testing_fuzzing.md), [Lesson 26](26_error_design.md), [Lesson 21](21_trait_objects_dispatch.md) (the `dyn Program` seam) and [Lesson 15](15_newtype_typestate.md). Lesson 54 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

Most smart-contract bugs are not clever cryptography; they are a program that **believed an account it was handed**. On Solana the caller chooses the accounts, so the caller is also the attacker. The lesson is organized around experiments in the crate that show where each rule comes from:

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| There is one data model: accounts | `Account { lamports, data, owner, executable }` is all the state there is; a wallet, a counter and a program are the same struct (§2) |
| Only the owner may change data or debit | a non-owner changing one byte, debiting one lamport, or editing a program is refused after the instruction runs; crediting is allowed (§4) |
| The runtime, not the program, enforces ownership | a program that tries to cheat is stopped by `verify_changes`, which is a pure function that is also called directly in a test (§4) |
| A transaction is atomic and failures still cost a fee | a failure in the second instruction undoes the first; the fee remains; a payer may not be left below the rent minimum (§3.4) |
| Size is arithmetic you can do by hand | a one-signer transfer is 203 bytes with empty data; 1,028 bytes of data fits in 1,232 and 1,029 does not; 21 transfers fit in one transaction and 22 do not (§3.3) |
| A PDA is an address with no private key | about half of random hashes are valid curve points; the derivation rejects those, and **no** derived address was ever on the curve (§5) |
| The *canonical* bump is the first off-curve one | for one program and authority, bumps 255 and 254 are on the curve, 253 is canonical, and 252 is a second valid but non-canonical address (§5.3) |
| CPI passes privileges, never invents them | a callee gets a signature only if the caller had it or the caller's own PDA signs; write access cannot be escalated; unknown accounts are refused (§6) |
| CPI is bounded | a fifth nested call fails, re-entering a program through another program fails, and each CPI costs compute units (§6.3) |
| A program must validate the accounts it receives | the counter checks owner, writability, size, signer and stored authority; an attacker-supplied lookalike is refused (§7) |
| Closing an account must reset it | close then re-create in one transaction works only if the closer reset the owner (a "revival" attack otherwise) (§7.3) |
| The whole machine conserves money | 200 seeds of random transfers keep total supply constant and every touched account rent-exempt (§8) |

Total in the practical challenge: one library crate, 7 modules, **77 tests**, a benchmark example, and **132 mutants** run against the test suite.

---

## 2. Accounts: the only state there is

### 2.1 What an account is ✓

```rust
/// Minimum balance for an account with `data_len` bytes to stay alive.
pub fn rent_exempt_minimum(data_len: usize) -> u64 {
    (ACCOUNT_OVERHEAD + data_len as u64) * LAMPORTS_PER_BYTE_YEAR * EXEMPTION_YEARS
}

/// The only kind of state on the chain: a balance, some bytes, and an owner.
#[derive(Debug, Clone, PartialEq, Eq, Default)]
pub struct Account {
    /// Balance in lamports (1 SOL = 10⁹ lamports).
    pub lamports: u64,
    /// Arbitrary state, writable only by the owner program.
    pub data: Vec<u8>,
    /// The program that may change `data` and debit `lamports`.
    pub owner: Pubkey,
    /// Is this account a program?
    pub executable: bool,
}
```

Four fields, from memory the same four the real runtime keeps: a **balance** in lamports (1 SOL = 10⁹ lamports), a **byte array** `data`, an **owner** (the key of a program), and an **executable** flag. Everything on the chain is one of these:

- A **wallet** is an account owned by the system program with empty data.
- A **program** is an account with `executable = true` whose data is code. In this model programs are Rust objects registered in the `Bank` with an executable account owned by a *loader* (`LOADER`, a placeholder key); in the real system the data is BPF/SBF bytecode and the owner is one of the loader programs (from memory).
- A **state account** (a counter, a token balance, an order book) is owned by the program that defines its layout; the bytes mean whatever that program says they mean.

A **public key** is 32 bytes and is displayed in **base58** (the Bitcoin alphabet, which has no `0`, `O`, `I` or `l`). The crate has its own encoder, because a lesson should not hide an algorithm behind a dependency, and it is checked against known answers computed by a Python implementation (✓ `base58_known_answers`): 32 zero bytes are `11111111111111111111111111111111` (each leading zero byte becomes the character `1`; the real system program's address has that form), 32 bytes of `0x01` are `4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi`, and `b"\0\0abc"` is `11ZiCa`. ✓ Decoding rejects characters outside the alphabet and reports the index (`base58_rejects_bad_characters`), and round-trips random keys.

### 2.2 Rent and the exemption minimum ✓

Storing bytes in every validator's memory costs something, so an account must hold a **minimum balance** that depends on its size, or it is removed. In this model (the rate is **from memory** of Solana's defaults: 3,480 lamports per byte-year, two years to be exempt, 128 bytes of per-account overhead):

`rent_exempt_minimum(len) = (128 + len) × 3,480 × 2 = (128 + len) × 6,960`

✓ `rent_minimums` pins the numbers: 890,880 lamports for 0 bytes, 1,169,280 for the 40-byte counter, 7,850,880 for 1,000 bytes (and the first value I wrote in that test was wrong by arithmetic, not by logic; the test caught my slip, not the code). The rule enforced by `Bank::process` after the instructions run: an account that was **changed** and still exists must have either **zero lamports** (it is deleted at the end of the transaction) or at least its minimum. ✓ `a_new_account_below_the_rent_minimum_is_refused`, `rent_is_checked_for_every_modified_account`, and an account emptied to zero lamports simply disappears (`an_account_emptied_to_zero_lamports_is_deleted`). The same rule applies to the **fee payer** (§3.4): paying the fee may not leave it with a balance that is neither zero nor rent-exempt. A property test found that gap in my first version (§8.2).

Exercise 1 asks for the inverse: the largest data size a given balance can pay for.

---

## 3. Instructions and transactions

### 3.1 The shape of a call ✓

```rust
/// One account an instruction touches, with the privileges it asks for.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct AccountMeta {
    /// Which account.
    pub pubkey: Pubkey,
    /// Must have authorized the transaction (or be a PDA signed for by its program).
    pub is_signer: bool,
    /// May be modified.
    pub is_writable: bool,
}

/// A call to one program: which program, which accounts, which bytes.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Instruction {
    /// The program to run.
    pub program_id: Pubkey,
    /// The accounts, in the order the program expects them.
    pub accounts: Vec<AccountMeta>,
    /// Program-specific arguments.
    pub data: Vec<u8>,
}

/// Instructions that succeed or fail together, authorized by `signers`.
/// The first signer pays the fee. (Signature verification is assumed done.)
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Transaction {
    /// Keys that signed; the first is the fee payer.
    pub signers: Vec<Pubkey>,
    /// Executed in order.
    pub instructions: Vec<Instruction>,
}
```

An **instruction** says: *run this program, with these accounts, on this data*. Each `AccountMeta` states two privileges the caller claims for that account: **`is_signer`** (the transaction carries a signature for this key) and **`is_writable`** (the program may modify it). A **transaction** is a list of instructions that succeed or fail **together**, plus the keys that signed it. The instruction data is opaque bytes whose layout the program defines (the system program uses a little-endian `u32` tag followed by fields; ✓ `transfer_data(5)` is `02 00 00 00 05 00 00 00 00 00 00 00`).

The consequence of "accounts are listed by the caller" is the most important idea of the lesson: **the program receives accounts, not names**. A program that expects "the vault of user X" gets *some account* in slot 3 of the instruction; whether it is really the vault, owned by this program, writable, and authorized by a signer is for the program to **check**. The runtime enforces only the generic rules of §4; the program-specific ones (which account is which) are the program's job. §7 shows a program doing it, and Lesson 54's Anchor automates it.

### 3.2 Account keys ✓

```rust
/// Distinct account keys: signers first, then every other key in first-use order.
pub fn account_keys(&self) -> Vec<Pubkey> {
    let mut seen = BTreeSet::new();
    let mut keys = Vec::new();
    let others = self.instructions.iter().flat_map(|ix| {
        ix.accounts
            .iter()
            .map(|m| m.pubkey)
            .chain(std::iter::once(ix.program_id))
    });
    for k in self.signers.iter().copied().chain(others) {
        if seen.insert(k) {
            keys.push(k);
        }
    }
    keys
}
```

Distinct keys, **signers first**, then every other key in order of first use, program ids included (a program is an account too, so a transaction must list it). In the real wire format the keys are sorted into four groups (writable signers, read-only signers, writable non-signers, read-only non-signers) and instructions refer to them by index (from memory); the model keeps first-use order, which is enough to count bytes. ✓ `account_keys_are_deduplicated_signers_first`.

### 3.3 Size is arithmetic ✓

A legacy transaction must fit in **1,232 bytes** (from memory: a 1,280-byte IPv6 minimum packet less headers), and that single number shapes program design. The model's serializer follows the legacy layout from memory:

```rust
/// Size of the legacy wire format: signatures, header, keys, recent
/// blockhash and compiled instructions (from memory of the layout).
pub fn serialized_len(&self) -> usize {
    let keys = self.account_keys().len();
    let mut n = shortvec_len(self.signers.len()) + 64 * self.signers.len();
    n += 3; // header
    n += shortvec_len(keys) + 32 * keys;
    n += 32; // recent blockhash
    n += shortvec_len(self.instructions.len());
    for ix in &self.instructions {
        n += 1 + shortvec_len(ix.accounts.len()) + ix.accounts.len();
        n += shortvec_len(ix.data.len()) + ix.data.len();
    }
    n
}
```

`shortvec_len` is the compact length encoding (1 byte below 128, 2 below 16,384, else 3). For one signer, the transaction is `1 + 64` (signatures) `+ 3` (header) `+ 1 + 32·keys` `+ 32` (recent blockhash) `+ 1` (instruction count) plus the instructions. ✓ **By hand**, for the shape of a system transfer (one signer, two accounts, the program, `d` bytes of data): keys = payer, recipient and the system program = 3, so signatures `1 + 64`, header `3`, keys `1 + 96`, blockhash `32` and instruction count `1` make `198`; the instruction itself is `1` (program index) `+ 1 + 2` (account list) `+ shortvec(d) + d`. The tests (`serialized_length_matches_the_hand_count`, `transaction_size_limit_is_1232_bytes`) pin **203 bytes when `d = 0`**, 330 when `d = 127`, 332 when `d = 128` (the length prefix widens to two bytes), and **1,232 when `d = 1,028` and 1,233 when `d = 1,029`**: the first is accepted by the size check, the second is refused with `TxTooLarge(1233)` and **no fee**. `shortvec_widens_at_16384_bytes_of_data` pins the second widening (a one-signer instruction with no accounts has 168 fixed bytes, then 1, 2 or 3 bytes of prefix and the data: lengths 127/128 and 16,383/16,384).

✓ Exercise 7 computes how many one-lamport transfers fit: each added recipient costs `32 (key) + 1 + 1 + 2 + 1 + 12 = 49` bytes on top of 166 fixed, so `166 + 49n ≤ 1,232` gives **n = 21** (1,195 bytes), and 22 is 1,244. The practical consequences, from memory: programs avoid large instruction data, transactions that touch many accounts need **address lookup tables** (versioned transactions replace 32-byte keys by 1-byte indexes into an on-chain table; not modeled), and anything big goes into an account that several transactions fill piece by piece.

### 3.4 Fees, atomicity and failure ✓

`Bank::process` runs in this order (the crate's whole transaction pipeline appears in §8.1):

1. The encoded size must be ≤ 1,232 (`TxTooLarge`), there must be at least one signer (`NoSigners`), and **every account marked `is_signer` in any instruction must be in the transaction's signer list** (`MissingSignature`). These checks fail *before* any fee: a malformed transaction costs nothing, because it never entered a block (from memory: such transactions are rejected at the door).
2. The **fee payer** (the first signer) pays `5,000 × signatures` lamports (from memory of the base fee). It must be able to afford it **and** keep a balance of zero or at least its rent minimum (`InsufficientFeePayer`; ✓ `fee_is_per_signature_and_payer_must_afford_it`, `the_fee_may_not_leave_the_payer_below_the_rent_minimum`).
3. All instructions run on a **copy** of the account map. If any instruction fails, the copy is thrown away, **and the fee stays charged**: ✓ `failed_transactions_pay_the_fee_and_change_nothing_else`. The same holds for a rent failure at the end.
4. If all succeed, the rent rule is checked on every changed account and the copy replaces the real state; accounts with 0 lamports are removed.

✓ `a_failure_in_a_later_instruction_undoes_an_earlier_program_s_changes`: the first instruction transfers lamports, the second fails, and the recipient has nothing. Atomicity across instructions is what lets a client compose "swap, then pay, then close" in one transaction without a half-finished state ever being visible (Lesson 49). The fee is the price of occupying block space whether or not you succeeded; the unconditional fee is also a spam defense.

---

## 4. The ownership rules and the post-instruction check

### 4.1 Six rules, enforced after the program runs ✓

A real runtime lets the program run, then compares every writable account before and after. The model does the same in one pure function:

```rust
/// Checks the changes made by an instruction run by `program_id`.
/// `writable` lists the instruction's distinct keys and whether any of its
/// metas for that key was writable.
pub fn verify_changes(
    program_id: &Pubkey,
    writable: &BTreeMap<Pubkey, bool>,
    before: &BTreeMap<Pubkey, Account>,
    after: &BTreeMap<Pubkey, Account>,
) -> Result<(), Error> {
    let mut sum_before: u128 = 0;
    let mut sum_after: u128 = 0;
    for (key, &is_writable) in writable {
        let pre = &before[key];
        let post = &after[key];
        sum_before += u128::from(pre.lamports);
        sum_after += u128::from(post.lamports);
        if !is_writable {
            if pre != post {
                return Err(Error::ReadonlyModified(*key));
            }
            continue;
        }
        if post.executable != pre.executable {
            return Err(Error::ExecutableChanged(*key));
        }
        let owns = pre.owner == *program_id;
        if post.lamports < pre.lamports && !owns {
            return Err(Error::UnauthorizedDebit(*key));
        }
        if post.data != pre.data && (!owns || pre.executable) {
            return Err(Error::UnauthorizedDataChange(*key));
        }
        if post.owner != pre.owner && (!owns || pre.executable || post.data.iter().any(|&b| b != 0))
        {
            return Err(Error::UnauthorizedOwnerChange(*key));
        }
    }
    if sum_before != sum_after {
        return Err(Error::LamportsNotConserved);
    }
    Ok(())
}
```

In words (and these six are, from memory, the heart of the real rules too):

1. A **read-only** account must be byte-for-byte unchanged.
2. The **executable** flag never changes inside a normal instruction.
3. **Lamports may be debited only by the owner program.** *Anyone* may credit an account (that is why you can send lamports to any address).
4. **Data may be changed only by the owner**, and never on an executable account.
5. The **owner** may be changed only by the current owner, only on a non-executable account, and only if the data is **all zeros** (so one program cannot reinterpret another's state by taking it over).
6. **Lamports are conserved**: the sum over the instruction's accounts before and after must be equal. A program cannot create or destroy money.

✓ `only_the_owner_may_change_data`, `only_the_owner_may_debit_but_anyone_may_credit`, `readonly_accounts_cannot_change_even_for_their_owner`, `lamports_are_conserved_across_an_instruction`, `owner_changes_need_ownership_and_zeroed_data`, `programs_cannot_be_edited_or_made_executable`, and `verify_changes_is_directly_callable`, which calls the function on hand-built maps. One rule needed a second test only after the mutation run: an **executable** account owned by the running program cannot be handed to another owner even with zeroed data (`a_program_cannot_hand_off_an_executable_account`; §15).

The design deserves a sentence: the checks come **after** the program ran. The program has `account_mut` and can write anything; the runtime compares and rolls back. That is how the real system keeps program code simple and the rules uniform, and it means a program's own checks are about *authorization* ("is this caller allowed?"), not about *physics* ("can lamports appear from nowhere?").

### 4.2 What the runtime does *not* check

Rule 3 says only the owner may debit, but nothing says *which* of the owner's accounts a given caller may debit. That is authorization, and it is entirely the program's. Suppose a vault program (the owner of all vaults) is called with vault A and an `authority` account that did not sign: the runtime lets the debit through, because the vault program owns A. This is why most real exploits are **missing checks inside the program**, and why the debugging challenge (§13) is a vault whose program forgot them.

The checklist, which §7 implements and Lesson 54 revisits, for every account a program receives:

- **Owner**: is `account.owner` the program (or the expected program) I think it is? Otherwise an attacker passes an account whose bytes *look* right but belong to someone else's program.
- **Signer**: if this account authorizes the action, is `is_signer` true?
- **Writable**: if I will modify it, was it passed writable? (The runtime would reject the write anyway, but an explicit error is better than a late one.)
- **Identity**: is this the *particular* account I expect (the authority stored in the vault; the PDA derived from these seeds)? A signer check proves *someone* signed, not that it was *the right someone*.
- **Size and layout**: is `data.len()` what I expect? Parsing a short slice by indexing panics; parsing the wrong type silently misreads.
- **Duplicates**: two slots may hold the *same* account (✓ `duplicate_accounts_alias_the_same_account`; `a_key_listed_twice_is_writable_if_either_meta_says_so`). A program that debits slot 0 and credits slot 1 and is handed one account twice may do something it never intended.

---

## 5. Program-derived addresses

### 5.1 The problem

A program wants to **own** accounts at addresses it can compute (the counter of a user, the vault of an order), and it wants to be able to **authorize** actions for them ("this vault may pay out"). But only the holder of a **private key** can sign, and a program has no private key. The solution: derive an address that **has no private key at all**, deterministically from the program's id and some seeds, and let the *runtime* accept a signature for it from the program that derived it.

### 5.2 The derivation ✓

An Ed25519 public key is a point on a curve, encoded in 32 bytes. Not every 32-byte string decodes to a valid point; roughly half do (✓ Python, 2,000 hashes, 0.4985 on the curve; ✓ `roughly_half_of_random_hashes_are_on_the_curve` requires between 850 and 1,149 of 2,000 hashes in the crate). A **program-derived address** is a SHA-256 hash of the seeds, the program id and a fixed marker that is **not** a valid point, so that no private key exists for it:

```rust
/// `SHA-256(seeds… ‖ program_id ‖ "ProgramDerivedAddress")`, accepted only
/// if the result is **not** a valid curve point.
pub fn create_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> Result<Pubkey, PdaError> {
    if seeds.len() > MAX_SEEDS {
        return Err(PdaError::TooManySeeds);
    }
    if seeds.iter().any(|s| s.len() > MAX_SEED_LEN) {
        return Err(PdaError::SeedTooLong);
    }
    let mut h = Sha256::new();
    for s in seeds {
        h.update(s);
    }
    h.update(program_id.0);
    h.update(MARKER);
    let key = Pubkey(h.finalize().into());
    if key.is_on_curve() {
        Err(PdaError::OnCurve)
    } else {
        Ok(key)
    }
}
```

`is_on_curve` is real: it asks `curve25519-dalek` to decompress the 32 bytes as an Edwards point (✓ `CompressedEdwardsY::decompress`, which succeeds exactly for valid encodings), and the derivation **rejects** hashes that decompress. Limits (from memory of the real values): at most 16 seeds, each at most 32 bytes (✓ `seed_limits`). The marker is the ASCII string `ProgramDerivedAddress`. Because the output is a hash, **any change to any seed or to the program id gives an unrelated address** (✓ `pda_is_never_on_the_curve_and_depends_on_every_input`).

### 5.3 Bumps and the canonical address ✓

Since half of all hashes fall on the curve, the programmer cannot count on the first try. The convention: append one extra seed, a **bump** byte, and search from 255 downward until the result is off the curve:

```rust
/// Tries bump seeds 255, 254, … and returns the first off-curve address and
/// the bump that produced it (the *canonical* bump).
pub fn find_program_address(
    seeds: &[&[u8]],
    program_id: &Pubkey,
) -> Result<(Pubkey, u8), PdaError> {
    if seeds.len() >= MAX_SEEDS {
        return Err(PdaError::TooManySeeds);
    }
    for bump in (0..=255u8).rev() {
        let b = [bump];
        let mut all: Vec<&[u8]> = seeds.to_vec();
        all.push(&b);
        match create_program_address(&all, program_id) {
            Ok(k) => return Ok((k, bump)),
            Err(PdaError::OnCurve) => continue,
            Err(e) => return Err(e),
        }
    }
    Err(PdaError::NoBump)
}
```

✓ The number of tries is geometric: over 2,000 seeds the histogram of "bumps tried" (1, 2, 3, …, 8 or more) is `[978, 521, 257, 134, 60, 29, 9, 12]`, which is close to 1000, 500, 250, 125… (about half the seeds succeed on the first try). The probability of exhausting all 256 bumps is about 2⁻²⁵⁶: negligible, but the function still returns `NoBump` instead of pretending (no test can reach it, so that path is checked by reading only).

✓ **Known answers** (computed by an independent Python implementation of the same hash and a pure-Python curve check): for program `[1; 32]` and the seed `b"counter"` the address is `6gxDfzF9Tr94UKgnThhCFLwYgj8z31YGKhKQkiTBJDAb` with bump 255; with the seeds `b"counter"` and `[2; 32]` it is `A8e1wdCK4icXgmfxNnVgkkbyXothp9JwHyqKGtT6azNj`, bump 255. For the counter program `[0xC0; 32]` and authority `[1; 32]`, bumps 255 and 254 are on the curve, **253 is the canonical bump**, and the address from bump 252 is *also* off the curve (a second valid address for the same seeds: ✓ `pda_known_answers_with_canonical_bumps`).

Why does that last fact matter? Two different bumps give two different valid addresses for the *same* logical seeds. If a program accepts a bump from the client without checking that it is the **canonical** (first valid) one, an attacker can create a second counter or vault for the same owner at the non-canonical address, and any logic that assumes "one account per owner" breaks. The model's counter therefore **recomputes** the canonical address and bump itself and compares both (`InvalidSeeds`; ✓ `the_pda_cannot_be_created_with_a_wrong_address_or_bump`). Exercise 2 is exactly this check. In real programs the finding is expensive (about 3.6 µs here per attempt, §8.4; in the real runtime a syscall with a compute cost, from memory in the low thousands of units), so programs store the canonical bump in the account at creation and verify it, rather than searching again every time.

### 5.4 Design notes for seeds

- **Put a type label in the seeds** (`b"counter"`), so that a counter and a vault for the same user cannot collide. This is domain separation (Lesson 48).
- **Seeds are public.** Anyone can derive the address; the PDA is a *name*, not a secret.
- **Do not concatenate variable-length data** into a single seed (`"ab" ‖ "c"` and `"a" ‖ "bc"` are the same bytes; Lesson 52 exercise 3). Separate seeds are hashed in order but *not* length-prefixed in this scheme (from memory, the real one is the same), so keep fixed-width fields or a separator in mind.
- A PDA **cannot sign a transaction**, because it has no key. It signs only through CPI, §6.

---

## 6. Cross-program invocation

### 6.1 Calling another program ✓

A program calls another by building an **instruction** and asking the runtime to run it, passing along accounts it already holds:

```rust
/// Cross-program invocation. `signer_seeds` lists PDAs of **this**
/// program that the callee may treat as signers.
pub fn invoke(
    &mut self,
    ix: &crate::instruction::Instruction,
    signer_seeds: &[&[&[u8]]],
) -> Result<(), Error> {
    self.consume(CPI_COST)?;
    let caller_id = self.program_id();
    let mut pda_signers = BTreeSet::new();
    for seeds in signer_seeds {
        let k = create_program_address(seeds, &caller_id).map_err(|_| Error::InvalidSeeds)?;
        pda_signers.insert(k);
    }
    for (n, m) in ix.accounts.iter().enumerate() {
        let held: Vec<&AccountMeta> = self
            .frame()
            .metas
            .iter()
            .filter(|c| c.pubkey == m.pubkey)
            .collect();
        if held.is_empty() {
            return Err(Error::MissingAccount(n));
        }
        if m.is_signer && !(held.iter().any(|c| c.is_signer) || pda_signers.contains(&m.pubkey))
        {
            return Err(Error::PrivilegeEscalation(m.pubkey));
        }
        if m.is_writable && !held.iter().any(|c| c.is_writable) {
            return Err(Error::PrivilegeEscalation(m.pubkey));
        }
    }
    self.run(ix.program_id, ix.accounts.clone(), &ix.data)
}
```

What `invoke` enforces, in order (the tests are named for each):

- **The caller must already hold every account it passes** (`MissingAccount`; ✓ `a_cpi_can_only_use_accounts_it_was_given`). A program cannot reach into the rest of the chain.
- **Signer privilege can be passed on, never invented.** A callee meta marked `is_signer` is accepted if the caller received that account as a signer (the user's signature *extends* through the call: ✓ `a_cpi_passes_on_the_users_signature`) **or** if the key is a **PDA the calling program derived** from the `signer_seeds` it supplies (`invoke_signed`). ✓ `a_pda_can_sign_only_for_its_own_program`: the runtime derives the address from the seeds **and the caller's program id**, so program B cannot sign for a PDA of program A, and a derivation that lands on the curve is refused (`seeds_that_derive_an_on_curve_address_are_refused`). ✓ `a_cpi_cannot_invent_a_signature` is the negative case.
- **Write privilege cannot be escalated**: a callee may mark an account writable only if the caller held it writable (`PrivilegeEscalation`; ✓ `a_cpi_cannot_gain_write_access`).

### 6.2 Verifying the callee's work ✓

`run` is the same function for top-level and nested instructions:

```rust
fn run(
    &mut self,
    program_id: Pubkey,
    metas: Vec<AccountMeta>,
    data: &[u8],
) -> Result<(), Error> {
    if self.stack.len() > MAX_CPI_DEPTH {
        return Err(Error::DepthExceeded);
    }
    if let Some(last) = self.stack.last()
        && last.program_id != program_id
        && self.stack.iter().any(|f| f.program_id == program_id)
    {
        return Err(Error::ReentrancyNotAllowed);
    }
    let programs = self.programs;
    let program = programs
        .get(&program_id)
        .ok_or(Error::UnknownProgram(program_id))?;
    if !self.work.get(&program_id).is_some_and(|a| a.executable) {
        return Err(Error::NotExecutable(program_id));
    }
    let mut writable: BTreeMap<Pubkey, bool> = BTreeMap::new();
    for m in &metas {
        *writable.entry(m.pubkey).or_insert(false) |= m.is_writable;
        self.work.entry(m.pubkey).or_default();
    }
    let before: BTreeMap<Pubkey, Account> = writable
        .keys()
        .map(|k| (*k, self.work[k].clone()))
        .collect();
    self.stack.push(Frame {
        program_id,
        metas,
        before: before.clone(),
    });
    let result = program.process(self, data);
    let frame = self.stack.pop().expect("pushed above");
    result?;
    // `frame.before` also reflects what nested calls legitimately changed
    verify_changes(&program_id, &writable, &frame.before, self.work)?;
    if let Some(caller) = self.stack.last_mut() {
        for k in before.keys() {
            if caller.before.contains_key(k) {
                caller.before.insert(*k, self.work[k].clone());
            }
        }
    }
    Ok(())
}
```

It snapshots the accounts the instruction can touch, pushes a **frame** with the instruction's metas, calls the program, pops the frame, and runs `verify_changes` with **that program's** identity against the snapshot. So a callee that debits an account its caller owns but it does not is stopped, at the callee's return, with `UnauthorizedDebit`; and the caller's own later checks see the callee's updates. After the callee returns, the caller's snapshot is refreshed for those keys, which matters: the first version of this model compared a nested call's changes against a snapshot taken **before** the call, and flagged legal chains of calls as unauthorized debits. That bug was mine; the CPI tests found it (§8.2).

### 6.3 Depth, re-entrancy and cost ✓

- **Depth**: from memory, the real runtime allows a call stack of a few frames (the model uses `MAX_CPI_DEPTH = 4` nested invocations beyond the top level). ✓ `recursion_depth_is_limited`: a program that calls itself four times (five frames with the top level) succeeds and a fifth nested call fails with `DepthExceeded`.
- **Re-entrancy**: a program may call itself directly (recursion) but may not be re-entered *through another program* (A → B → A). ✓ `reentrancy_through_another_program_is_refused`. This closes the classic family of bugs where a callee calls back into a caller that is half-way through changing its state.
- **Cost**: each CPI costs compute units (`CPI_COST = 1,000` here, from memory a similar order of magnitude in the real system), in addition to the callee's own work; ✓ `cpi_costs_compute_units`.

---

## 7. A program that validates: the counter

### 7.1 The system program ✓

The **system program** (id: 32 zero bytes, displayed as thirty-two `1`s) creates accounts, assigns owners, allocates space and moves lamports between accounts *it* owns:

```rust
impl Program for SystemProgram {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(COST)?;
        let tag = u32::from_le_bytes(
            data.get(..4)
                .ok_or(Error::InvalidInstructionData)?
                .try_into()
                .expect("4 bytes"),
        );
        match (tag, data.len()) {
            (0, 52) => {
                let (lamports, space) = (u64_at(data, 4), u64_at(data, 12));
                let owner = Pubkey(data[20..52].try_into().expect("32 bytes"));
                need_signer(inv, 0)?;
                need_signer(inv, 1)?;
                if space > MAX_DATA_LEN || inv.key(0)? == inv.key(1)? {
                    return Err(Error::InvalidInstructionData);
                }
                let new = inv.account(1)?;
                if !new.data.is_empty() || new.owner != Pubkey::default() {
                    return Err(Error::AlreadyInUse);
                }
                move_lamports(inv, 0, 1, lamports)?;
                let new = inv.account_mut(1)?;
                new.data = vec![0; space as usize];
                new.owner = owner;
                Ok(())
            }
            (1, 36) => {
                let owner = Pubkey(data[4..36].try_into().expect("32 bytes"));
                need_signer(inv, 0)?;
                if inv.account(0)?.owner != Pubkey::default() {
                    return Err(Error::WrongOwner(0));
                }
                inv.account_mut(0)?.owner = owner;
                Ok(())
            }
            (2, 12) => {
                need_signer(inv, 0)?;
                move_lamports(inv, 0, 1, u64_at(data, 4))
            }
            (8, 12) => {
                need_signer(inv, 0)?;
                let space = u64_at(data, 4);
                let acc: &Account = inv.account(0)?;
                if space > MAX_DATA_LEN || !acc.data.is_empty() || acc.owner != Pubkey::default() {
                    return Err(Error::AlreadyInUse);
                }
                inv.account_mut(0)?.data = vec![0; space as usize];
                Ok(())
            }
            _ => Err(Error::InvalidInstructionData),
        }
    }
}
```

Instruction data: `CreateAccount` (tag 0, 52 bytes: lamports, space, owner), `Assign` (1, 36 bytes), `Transfer` (2, 12 bytes), `Allocate` (8, 12 bytes), all little-endian (from memory of the real layout). Rules you can read in the code: `CreateAccount` needs **both** the funder and the new account to sign (the new address must have consented to being created: this is also why you cannot create an account at somebody else's wallet address); the new account must be empty and system-owned (`AlreadyInUse` otherwise); the transfer source must be system-owned **and have no data** (a data-carrying account is some program's state, not a wallet), must hold enough, and the destination credit uses a checked add. ✓ `transfer_source_rules`, `transfer_of_the_exact_balance_is_allowed_and_one_more_is_not`, `transfer_into_a_full_account_overflows_cleanly`, `create_account_needs_two_distinct_signers`, `create_account_refuses_an_account_in_use`, `assign_changes_the_owner_once`, `allocate_gives_an_empty_account_data`. The maximum data size is 10 MiB (from memory), and the test uses the literal number (not the constant) so that a changed constant cannot hide (§15).

How does a PDA, which has no private key, get created? It cannot sign `CreateAccount` itself. The program that derived it does: via CPI with `signer_seeds`, so the runtime vouches that the key belongs to the caller. That is the next program.

### 7.2 The counter: create, increment, close ✓

The counter stores `authority: 32 bytes ‖ count: u64 LE` (40 bytes) in a PDA derived from `[b"counter", authority]`. Instruction data is `[0, bump]` (create), `[1]` (increment) and `[2]` (close). Creation is the instructive path:

```rust
impl Program for CounterProgram {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(COST)?;
        match data {
            [0, bump] => {
                if !inv.is_signer(0)? {
                    return Err(Error::NotSigner(0));
                }
                if inv.key(2)? != Pubkey::default() {
                    return Err(Error::InvalidInstructionData);
                }
                let (payer, counter_key) = (inv.key(0)?, inv.key(1)?);
                inv.consume(1500)?; // searching for the canonical bump
                let (expected, canonical) = counter_address(&inv.program_id(), &payer);
                if expected != counter_key || canonical != *bump {
                    return Err(Error::InvalidSeeds);
                }
                let existing = inv.account(1)?;
                if !existing.data.is_empty() || existing.owner != Pubkey::default() {
                    return Err(Error::AlreadyInUse);
                }
                let lamports = rent_exempt_minimum(SPACE).saturating_sub(existing.lamports);
                let bump_seed = [*bump];
                let seeds: &[&[u8]] = &[b"counter", &payer.0, &bump_seed];
                debug_assert!(create_program_address(seeds, &inv.program_id()).is_ok());
                let ix = Instruction {
                    program_id: Pubkey::default(),
                    accounts: vec![
                        AccountMeta::writable(payer, true),
                        AccountMeta::writable(counter_key, true),
                    ],
                    data: system::create_account_data(lamports, SPACE as u64, &inv.program_id()),
                };
                inv.invoke(&ix, &[seeds])?;
                let acc = inv.account_mut(1)?;
                acc.data[..32].copy_from_slice(&payer.0);
                Ok(())
            }
            [1] => {
                check_counter(inv)?;
                let acc = inv.account_mut(0)?;
                let n = count_of(&acc.data).ok_or(Error::InvalidAccountData)?;
                let n = n.checked_add(1).ok_or(Error::ArithmeticOverflow)?;
                acc.data[32..40].copy_from_slice(&n.to_le_bytes());
                Ok(())
            }
            [2] => {
                check_counter(inv)?;
                if !inv.is_writable(1)? {
                    return Err(Error::NotWritable(1));
                }
                let lamports = inv.account(0)?.lamports;
                let to = inv.account(1)?.lamports;
                let total = to.checked_add(lamports).ok_or(Error::ArithmeticOverflow)?;
                inv.account_mut(1)?.lamports = total;
                let acc = inv.account_mut(0)?;
                acc.lamports = 0;
                acc.data = Vec::new();
                acc.owner = Pubkey::default();
                Ok(())
            }
            _ => Err(Error::InvalidInstructionData),
        }
    }
}
```

Read `Create` as a checklist. The payer must sign. The third account must be the real system program (otherwise the CPI target would be attacker-chosen). The program **recomputes** the canonical PDA and bump (charging 1,500 units for the search, as the real syscall would charge something similar) and compares both with what the client sent. It checks that the address is unused, **tops up** a prefunded address (anyone can send lamports to a PDA address in advance, so the program creates the account for only the missing difference: ✓ `create_with_a_prefunded_address_tops_up_only_the_difference`; a program that always sends the full minimum, or that refuses a nonzero starting balance, can be griefed), then CPIs the system program's `CreateAccount` with the payer signing through the user's signature and the counter signing through the PDA seeds. Only then does it write the authority into the data, which the runtime will accept because the account is now owned by this program.

`Increment` and `Close` share `check_counter`:

```rust
/// The checks shared by `Increment` and `Close`.
fn check_counter(inv: &Invoke<'_>) -> Result<(), Error> {
    let counter = inv.account(0)?;
    if counter.owner != inv.program_id() {
        return Err(Error::WrongOwner(0));
    }
    if !inv.is_writable(0)? {
        return Err(Error::NotWritable(0));
    }
    if counter.data.len() != SPACE {
        return Err(Error::InvalidAccountData);
    }
    if !inv.is_signer(1)? {
        return Err(Error::NotSigner(1));
    }
    if counter.data[..32] != inv.key(1)?.0 {
        return Err(Error::WrongAuthority);
    }
    Ok(())
}
```

Owner, writability, size, signer and **stored authority** are all checked, in that order, and each has a test that breaks exactly one (✓ `increment_checks_signer_authority_owner_and_size`). ✓ `an_attacker_cannot_hand_the_program_an_account_they_control`: an attacker creates an account with the same 40-byte layout and *their own* key as authority and passes it in place of the real counter; the `owner` check refuses it (the account belongs to the system program or another program, not this one). Without that line the attacker could pass a lookalike *owned by themselves* and read or write forged state through the real program's logic.

Arithmetic is checked: ✓ `counter_overflow_is_an_error_not_a_wrap` sets the count to `u64::MAX` and expects `ArithmeticOverflow`. (In release builds, Rust's integer overflow **wraps** unless you ask for checks; on-chain programs are built in release mode (from memory), so `checked_add` is not optional.)

### 7.3 Closing and the revival attack ✓

Closing moves all lamports to the authority, zeroes the data, and **resets the owner to the system program**. The account then has 0 lamports and disappears when the transaction ends. Why reset the owner and clear the data *as well*? Because the account lives until the end of the **transaction**: another instruction later in the same transaction can still see it. ✓ `close_then_create_in_one_transaction_works_only_if_close_resets_the_owner` runs `[close, create]` on the same address in one transaction: it succeeds because the closed account looks fresh. A close that only drains lamports leaves the program's data and owner in place, and a later instruction in the same transaction can **refund** the account's rent and use it again, now carrying stale state ("revival"; a documented class of bugs in the real system, from memory). Two mutations that skip the owner reset were caught by exactly this test (§15).

---

## 8. Putting it together

### 8.1 The pipeline ✓

```rust
/// Runs a transaction **atomically**: either every instruction succeeds
/// and all changes are kept, or only the fee is.
pub fn process(&mut self, tx: &Transaction) -> Result<Receipt, Error> {
    let size = tx.serialized_len();
    if size > MAX_TX_BYTES {
        return Err(Error::TxTooLarge(size));
    }
    let payer = *tx.signers.first().ok_or(Error::NoSigners)?;
    for ix in &tx.instructions {
        for m in &ix.accounts {
            if m.is_signer && !tx.signers.contains(&m.pubkey) {
                return Err(Error::MissingSignature(m.pubkey));
            }
        }
    }
    let fee = FEE_PER_SIGNATURE * tx.signers.len() as u64;
    let (have, data_len) = self
        .accounts
        .get(&payer)
        .map_or((0, 0), |a| (a.lamports, a.data.len()));
    match have.checked_sub(fee) {
        // the payer must keep a rent-exempt balance (or end at zero)
        Some(left) if left == 0 || left >= rent_exempt_minimum(data_len) => {}
        _ => return Err(Error::InsufficientFeePayer),
    }
    self.accounts.get_mut(&payer).expect("exists").lamports -= fee;
    self.fees_collected += fee;

    let total = (UNITS_PER_INSTRUCTION * tx.instructions.len() as u64).min(MAX_UNITS);
    let mut work = self.accounts.clone();
    let remaining = {
        let mut inv = Invoke {
            work: &mut work,
            programs: &self.programs,
            stack: Vec::new(),
            remaining: total,
        };
        for ix in &tx.instructions {
            inv.run(ix.program_id, ix.accounts.clone(), &ix.data)?;
        }
        inv.remaining
    };
    for (k, a) in &work {
        let old = self.accounts.get(k).cloned().unwrap_or_default();
        if *a != old && a.lamports != 0 && a.lamports < rent_exempt_minimum(a.data.len()) {
            return Err(Error::RentNotExempt(*k));
        }
    }
    for (k, a) in work {
        if a.lamports == 0 {
            self.accounts.remove(&k);
        } else {
            self.accounts.insert(k, a);
        }
    }
    Ok(Receipt {
        fee,
        units_used: total - remaining,
    })
}
```

Everything in §3.4 appears here: the checks, the fee, the working copy, the shared compute pool (`UNITS_PER_INSTRUCTION × instructions`, capped at `MAX_UNITS`; ✓ `compute_budget_scales_with_instructions_and_is_capped`, `one_instruction_gets_exactly_200_000_units`, `the_cap_is_exactly_1_400_000_even_for_one_hungry_instruction`), the rent check on changed accounts, and the commit. Compute is a **meter**: `consume` fails with `ComputeExceeded` when the pool runs out, and a failed transaction's work is discarded but its fee is not. (In the real system the budget is requested per transaction, with 200,000 units per instruction by default and 1,400,000 as the cap; from memory. Here the pool is shared by all instructions in the transaction, which is how the real one behaves for the total; I did not check how the per-instruction default combines with the cap in every case.)

### 8.2 Two bugs I wrote ✓

1. **A stale snapshot in CPI** (§6.2): legal chains of calls failed with `UnauthorizedDebit` because the verifier compared a nested call's result against the state from before the *outer* call. Fixed by keeping the snapshot in the frame and refreshing the caller's view after each callee returns.
2. **A fee payer below the rent minimum** (§2.2): `random_transactions_conserve_lamports_and_never_break_rent` (200 seeds) failed at seed 17: paying the fee left the payer with a balance that was neither zero nor rent-exempt, and the model's own invariant ("every account that exists is rent-exempt") was broken by the *fee*. I had checked rent for the accounts instructions touched but not for the payer. Fixed by checking the post-fee balance up front (`InsufficientFeePayer`) and adding a direct test.

Both were found by tests that checked an **invariant** (conservation, rent) over many random cases and not an example, which is the same lesson as the breaker of Lesson 52 §6.3: write the property.

### 8.3 What the model leaves out, and what that costs you

- **Parallelism.** Real Solana runs non-conflicting transactions in parallel; two transactions conflict if one *writes* an account the other reads or writes (from memory: account locks). That is why the writable flag is part of the instruction: it is the **scheduling information** as well as a security flag. It also means that one hot writable account (a global counter, a popular pool) serializes everything that touches it.
- **Blockhashes and signatures.** A real transaction is signed and contains a recent blockhash that expires after roughly a minute and a half (from memory); the model has neither, so replay and expiry (Lesson 52's subject) are not modeled here.
- **Sysvars** (clock, rent, recent slot hashes), **loaders and upgradeable programs** (a program account points to a data account holding the code, and an authority can replace it), **address lookup tables**, **priority fees** and **compute-budget instructions**, **realloc** of an account's data inside a program, and **the BPF/SBF virtual machine** with its memory limits and syscalls. Each is real and each can matter in production.
- **Tokens.** The token program and associated token accounts are Lesson 54.

### 8.4 Cost ✓

Release build, single machine, several runs of `examples/bench.rs` (ranges are the lowest and highest of the runs): encoding a 32-byte key to base58 takes about 680–730 ns and decoding 44 characters about 354–361 ns. The curve check `is_on_curve` takes **3.45–3.52 µs**; `create_program_address` 3.61–3.70 µs, so almost all of it (more than 95%) is that one decompression, and one try of `find_program_address` costs the same, 3.63–3.67 µs. Over 2,000 seeds the number of tries is geometric (§5.3), so the average search costs about twice a single try, around 7 µs. Processing a system-transfer transaction through the whole bank (size, fee, working copy, verifier, commit) takes about 0.86–0.90 µs; incrementing a counter 1.05–1.20 µs; creating and closing a counter (one PDA search, a CPI, rent) about 17.7–17.9 µs, of which the PDA search inside `Create` (at least one 3.6 µs attempt, usually about two) is the largest single piece I can name; I did not profile the rest. These numbers say the model is cheap to test, not that anything is fast on a validator: a real validator spends its time on signature verification, the virtual machine and the account database, none of which is here.

---

## 9. Rust notes

- **The `Program` trait object** (`Box<dyn Program>`, Lesson 21) is the seam that stands in for bytecode: the bank maps a program id to an object. `process(&self, &mut Invoke, &[u8])` is deliberately the shape of the real entrypoint (`program_id`, `accounts`, `instruction_data`): a program is a function from accounts and bytes to success or an error, with no memory of its own between calls.
- **Newtypes for keys** (Lesson 15): `Pubkey([u8; 32])` has `Display` and `Debug` in base58, `Ord` (so it can key a `BTreeMap`, which also gives deterministic iteration), and a `const fn filled`, so tests can say `Pubkey::filled(7)`.
- **Errors as an enum** (Lesson 26) with indexes where useful (`MissingAccount(usize)`, `NotSigner(usize)`, `WrongOwner(usize)`): the index is the evidence. Real Solana programs return `ProgramError` codes and, with Anchor, named custom errors.
- **Checked arithmetic.** Every balance and counter update in the crate goes through `checked_add`/`checked_sub`/`saturating_*` or an explicit comparison first. `u64` lamport balances can overflow in principle (a hostile account can hold nearly `u64::MAX`), and a wrapping subtraction is one of the bugs in the debugging challenge (§13).
- **Borrowing.** `account(i)` returns `&Account` and `account_mut(i)` `&mut Account`; both borrow the whole `Invoke`, so you cannot hold two accounts mutably at once. The real SDK splits the slice or uses `RefCell` (`account.data.borrow_mut()`); a pattern you will meet in Lesson 54 is to read all the numbers first, compute, then write.
- **Dev-profile dependencies**: `[profile.dev.package."*"] opt-level = 3` makes the curve decompressions fast in tests (Lesson 52 §7.6).
- **No `unsafe`, no async, no global state**: the bank is a plain struct; every test builds its own.

---

## 10. Common mistakes

1. Trusting an account because its data *looks* right, without checking its **owner** (§4.2, §7.2).
2. Treating `is_signer` as "the right person": checking a signature exists but not that it belongs to the **stored authority** (§4.2, §13).
3. Forgetting that **anyone can credit any address**, so "this account must be empty" or "balance must equal X" can be broken by a stranger (§7.2, prefunded PDAs).
4. Accepting a client-supplied bump or address without recomputing the **canonical** PDA (§5.3).
5. Using wrapping arithmetic on balances and counters in release mode (§7.2).
6. Closing an account by draining lamports without clearing data and owner (§7.3).
7. Passing the program a **duplicate** account in two slots, and not considering aliasing (§4.2).
8. Letting the client choose the **program** a CPI calls (the system program, the token program) instead of checking its id (§7.2).
9. Assuming a CPI keeps privileges it was never given; or assuming a callee cannot modify a writable account you passed (§6.1).
10. Ignoring rent: creating an account with too little, or draining one below the minimum (§2.2, §8.2).
11. Designing instructions that need more than ~1,200 bytes or 200,000 compute units and discovering it in production (§3.3, §8.1).
12. Putting everything in one global writable account, which serializes the whole program (§8.3).
13. Re-entering a program through a callee, or relying on callee state without re-reading it after the call (§6.3).
14. Seeds without a type label, so two kinds of account collide (§5.4).
15. Testing only the happy path: a missing-check bug never fails a test that uses honest inputs (§13).
16. Believing a green suite means the program is secure (§11).

---

## 11. Security notes and honest limits

- `solanalab` is a **model**. It has no signature verification, no real runtime, no BPF verifier, no concurrency and no economic model. A program that passes every test here can still be wrong on the real chain, and the reverse: do not take a rule of this crate as the authoritative statement of the real rule.
- **Every instruction is adversarial input.** The attacker chooses the accounts, their order, the duplicates, the writable flags the transaction allows, the instruction data and the *other instructions in the same transaction*. A program must be correct for *every* such combination, and the best test habit is to write, for each check, one test that breaks only that check (as `check_counter`'s tests do).
- **Programs are public.** Seeds, layouts and logic are visible. Security cannot depend on hiding how a program works.
- **Upgrade authority is a trust assumption** (from memory: a program deployed as upgradeable can be replaced by whoever holds its upgrade authority). Who holds it, how, and whether it is later made immutable is part of the system's security, not an implementation detail. Lesson 55.
- **Rent and lamports are money**: wrapping arithmetic, conservation bugs and rent-exemption mistakes are loss-of-funds bugs. The runtime's conservation rule (§4.1, rule 6) protects the *total* but not *who* gets the money.
- **Mutation testing is evidence of test strength, not of correctness** (§15 shows 129 of 132 caught and the remaining ones explained; it says nothing about the real chain).
- **The numbers are from memory.** Fee, rent, compute and size limits have changed over the history of the network and may have changed again; any production design must read them from the current documentation and, where possible, from the chain itself.

---

## 12. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Write `max_data_len(lamports)`: the largest data size an account can hold and stay rent-exempt with that balance, or `None` if even an empty account is too expensive.
2. **(Easy)** Write `canonical_bump(addr, seeds, program_id)`: the bump if `addr` is the *canonical* PDA of the seeds, else `None`. Show with a second valid address that "is a valid PDA" is not enough.
3. **(Easy)** Write a strict decoder for the system program's `Transfer` data (12 bytes, tag 2). Which malformed inputs must it distinguish?
4. **(Medium)** Write `first_escalation(caller, callee, pda_signers)`: the first callee account whose privileges exceed what the caller held.
5. **(Medium)** Write `broken_rule(program, writable, pre, post)`: a pure function naming the first ownership rule violated by a change of one account.
6. **(Medium-hard)** Write `parse_vault(program, account, index, authority, authority_signed)` for a 40-byte vault (`authority ‖ amount`) that performs the checklist of §4.2 in the right order and returns the amount.
7. **(Hard)** How many one-lamport system transfers to distinct recipients fit in one transaction? Compute it with the crate's size arithmetic and derive the formula by hand.
8. **(Hard)** Given a tree of calls (own cost plus nested calls), compute the total compute units with a per-CPI cost and the nesting depth. The counter's `Create` costs 100 + 1,500 and CPIs the system program (150): what is the total?

<details markdown="1">
<summary>Answers</summary>

1. A possible answer (✓ compiled and run in a scratch crate with tests):

   ```rust
   //! Exercise 1: the largest data size an account can hold and still be rent-exempt.
   use solanalab::account::{
       ACCOUNT_OVERHEAD, EXEMPTION_YEARS, LAMPORTS_PER_BYTE_YEAR, rent_exempt_minimum,
   };

   /// Largest `data_len` with `rent_exempt_minimum(data_len) <= lamports`,
   /// or `None` if even an empty account is too expensive.
   pub fn max_data_len(lamports: u64) -> Option<usize> {
       let per_byte = LAMPORTS_PER_BYTE_YEAR * EXEMPTION_YEARS;
       let bytes = lamports / per_byte;
       let data = bytes.checked_sub(ACCOUNT_OVERHEAD)?;
       let data = usize::try_from(data).ok()?;
       debug_assert!(rent_exempt_minimum(data) <= lamports);
       Some(data)
   }
   ```

   A balance of 890,879 buys nothing (`None`); 890,880 buys 0 bytes; each further 6,960 lamports buys one byte; 10,000,000 lamports pay for **1,308** bytes (`10,000,000 / 6,960 = 1,436`, minus the 128 overhead bytes). The test also checks the round trip `max_data_len(rent_exempt_minimum(n)) == Some(n)` for several `n`, and that one lamport less loses one byte (except at 0, where it loses the account).
2. A possible answer (✓):

   ```rust
   //! Exercise 2: is this address the *canonical* PDA for these seeds?
   use solanalab::pubkey::{Pubkey, find_program_address};

   /// `Some(bump)` if `addr` is the canonical program-derived address of
   /// `seeds` under `program_id`.
   pub fn canonical_bump(addr: &Pubkey, seeds: &[&[u8]], program_id: &Pubkey) -> Option<u8> {
       let (expected, bump) = find_program_address(seeds, program_id).ok()?;
       (expected == *addr).then_some(bump)
   }
   ```

   For program `[1; 32]` and seeds `[b"seed", [0]]` the canonical bump is **254**. The address from the *next* bump, 253, is also a valid off-curve address and differs from it, but `canonical_bump` rejects it, as it rejects the right address under other seeds or another program. A program that accepts any valid PDA lets a user create a second account "for the same owner".
3. A possible answer (✓):

   ```rust
   //! Exercise 3: strict decoding of system-program `Transfer` data.

   /// Why the bytes are not a transfer.
   #[derive(Debug, PartialEq, Eq)]
   pub enum ParseError {
       /// Not exactly 12 bytes.
       BadLength(usize),
       /// The tag is not 2.
       BadTag(u32),
   }

   /// `tag:u32 LE (= 2) ‖ lamports:u64 LE`, nothing more, nothing less.
   pub fn parse_transfer(data: &[u8]) -> Result<u64, ParseError> {
       let bytes: [u8; 12] = data
           .try_into()
           .map_err(|_| ParseError::BadLength(data.len()))?;
       let tag = u32::from_le_bytes(bytes[..4].try_into().expect("4 bytes"));
       if tag != 2 {
           return Err(ParseError::BadTag(tag));
       }
       Ok(u64::from_le_bytes(bytes[4..].try_into().expect("8 bytes")))
   }
   ```

   Distinguish **length** (anything but 12, including 11 and 13, because trailing bytes are an ambiguity attack on parsers that ignore them) from **tag** (a different instruction, or a tag with a high byte set such as `2 + 2²⁴`). The test also parses `u64::MAX` correctly: the parser does no arithmetic.
4. A possible answer (✓):

   ```rust
   //! Exercise 4: may a CPI ask for these privileges?
   use solanalab::instruction::AccountMeta;
   use solanalab::pubkey::Pubkey;

   /// The first callee account whose privileges exceed the caller's, if any.
   /// A callee may be a signer only if the caller held that key as a signer or
   /// the key is one of `pda_signers`; it may be writable only if the caller
   /// held it writable. A key the caller does not hold at all is also refused.
   pub fn first_escalation(
       caller: &[AccountMeta],
       callee: &[AccountMeta],
       pda_signers: &[Pubkey],
   ) -> Option<Pubkey> {
       callee.iter().find_map(|m| {
           let held: Vec<&AccountMeta> = caller.iter().filter(|c| c.pubkey == m.pubkey).collect();
           let ok = !held.is_empty()
               && (!m.is_signer
                   || held.iter().any(|c| c.is_signer)
                   || pda_signers.contains(&m.pubkey))
               && (!m.is_writable || held.iter().any(|c| c.is_writable));
           (!ok).then_some(m.pubkey)
       })
   }
   ```

   Duplicates in the caller's list **combine** their privileges (a key listed twice, once writable and once signer, counts as both: this mirrors `writable` in `Bank`). A key the caller does not hold at all is refused even if it is a PDA signer. The test walks through read-only→writable, unsigned→signer with and without a PDA signature, a writable signer not held writable, an unknown key, and "the first offender is reported".
5. A possible answer (✓):

   ```rust
   //! Exercise 5: the rules for one account, as a pure function.
   use solanalab::account::Account;
   use solanalab::pubkey::Pubkey;

   /// First rule broken by changing `pre` into `post` while `program` runs.
   pub fn broken_rule(
       program: &Pubkey,
       writable: bool,
       pre: &Account,
       post: &Account,
   ) -> Option<&'static str> {
       if !writable {
           return (pre != post).then_some("read-only account modified");
       }
       let owns = pre.owner == *program;
       if post.executable != pre.executable {
           Some("executable flag changed")
       } else if post.lamports < pre.lamports && !owns {
           Some("debit by a non-owner")
       } else if post.data != pre.data && (!owns || pre.executable) {
           Some("data changed by a non-owner")
       } else if post.owner != pre.owner
           && (!owns || pre.executable || post.data.iter().any(|&b| b != 0))
       {
           Some("illegal owner change")
       } else {
           None
       }
   }
   ```

   The order in which the rules are tested decides which violation is *reported* when several are broken; the runtime's order is the one in `verify_changes`. The test covers read-only accounts, the executable flag, a credit by a non-owner (allowed) against a debit (refused), data written by a non-owner, a program's own code (executable) edited by its owner (refused) and an owner change with and without zeroed data.
6. A possible answer (✓):

   ```rust
   //! Exercise 6: validating an account a program is handed.
   use solanalab::account::Account;
   use solanalab::pubkey::Pubkey;
   use solanalab::runtime::Error;

   /// A vault: `authority:32 ‖ amount:u64 LE`.
   pub fn parse_vault(
       program: &Pubkey,
       account: &Account,
       index: usize,
       authority: &Pubkey,
       authority_signed: bool,
   ) -> Result<u64, Error> {
       if account.owner != *program {
           return Err(Error::WrongOwner(index));
       }
       if account.data.len() != 40 {
           return Err(Error::InvalidAccountData);
       }
       if !authority_signed {
           return Err(Error::NotSigner(index + 1));
       }
       if account.data[..32] != authority.0 {
           return Err(Error::WrongAuthority);
       }
       Ok(u64::from_le_bytes(
           account.data[32..40].try_into().expect("8 bytes"),
       ))
   }
   ```

   The order is **owner, size, signer, identity**: the cheapest and most fundamental first. An account with the wrong owner is rejected before any bytes are interpreted, because the bytes of a foreign account mean nothing. Index arithmetic returns the *position* in the error (`NotSigner(index + 1)` refers to the authority slot).
7. A possible answer (✓):

   ```rust
   //! Exercise 7: how many one-recipient transfers fit in a single transaction?
   use solanalab::instruction::{AccountMeta, Instruction, MAX_TX_BYTES, Transaction};
   use solanalab::pubkey::Pubkey;
   use solanalab::system::transfer_data;

   fn transfer(from: Pubkey, to: Pubkey) -> Instruction {
       Instruction {
           program_id: Pubkey::default(),
           accounts: vec![
               AccountMeta::writable(from, true),
               AccountMeta::writable(to, false),
           ],
           data: transfer_data(1),
       }
   }

   /// Largest `n` such that `n` transfers to `n` distinct recipients fit.
   pub fn max_transfers() -> usize {
       let payer = Pubkey::filled(1);
       let mut n = 0;
       loop {
           let ixs: Vec<Instruction> = (0..=n)
               .map(|i| {
                   let mut k = [0u8; 32];
                   k[0] = 2;
                   k[1] = i as u8;
                   k[2] = (i >> 8) as u8;
                   transfer(payer, Pubkey(k))
               })
               .collect();
           let tx = Transaction {
               signers: vec![payer],
               instructions: ixs,
           };
           if tx.serialized_len() > MAX_TX_BYTES {
               return n;
           }
           n += 1;
       }
   }
   ```

   One signer, `n` distinct recipients and the system program give `n + 2` keys. Fixed part: `65 + 3 + 1 + 64 + 32 + 1 = 166` plus `32` per recipient key; each instruction adds `1 + 1 + 2 + 1 + 12 = 17`. So `size = 166 + 49n`, **21 transfers = 1,195 bytes** and 22 = 1,244 > 1,232. (A real client would lower the key cost with a lookup table, or pack recipients in a program's own data; the sums above are the legacy format from memory.)
8. A possible answer (✓):

   ```rust
   //! Exercise 8: compute units of a call tree.

   /// A call: its own cost, and the calls it makes.
   pub struct Call {
       /// Units the program itself consumes.
       pub own: u64,
       /// Nested invocations; each also costs `cpi_cost` to make.
       pub children: Vec<Call>,
   }

   /// Total units of the tree, charging `cpi_cost` per nested call.
   pub fn total_units(call: &Call, cpi_cost: u64) -> u64 {
       call.own
           + call
               .children
               .iter()
               .map(|c| cpi_cost + total_units(c, cpi_cost))
               .sum::<u64>()
   }

   /// Deepest nesting below the top level.
   pub fn depth(call: &Call) -> usize {
       call.children
           .iter()
           .map(|c| 1 + depth(c))
           .max()
           .unwrap_or(0)
   }
   ```

   The counter's `Create`: `1,600` own units (100 + 1,500), one CPI at `1,000`, the system program's `150`: **2,750** units, depth 1. The test also checks a three-deep chain (`1 + 100 + 2 + 100 + 3 + 100 + 10`) and a leaf (depth 0). Design consequence: a call tree's cost is the sum of its parts *plus* a toll per edge, so deep call chains are expensive twice (the toll and the work).

</details>

---

## 13. Debugging challenge

A vault program holds funds in accounts it owns (the vault's data is the 32-byte key of its **authority**). The instruction is `[0, amount: u64 LE]` with accounts `0 = vault (writable)`, `1 = authority`, `2 = destination (writable)`. It was shipped, and four reports arrive. (1) A user who is **not** the authority, and who did not even include the authority's signature, empties a vault. (2) A user who signs **as themselves** passes their own key in slot 1 and withdraws from *another* user's vault. (3) A withdrawal larger than the balance does not fail with a clear error; the transaction is rejected by the runtime with a message about lamports not being conserved. (4) A withdrawal that leaves the vault with 1,000,000 lamports fails with `RentNotExempt`, and the user is told nothing useful. Explain each cause and fix the code.

```rust
//! Debugging challenge: a vault program with three defects.
use solanalab::runtime::{Error, Invoke, Program};

/// Reads the amount from instruction data `[0, amount:u64 LE]`.
fn amount_of(data: &[u8]) -> Result<u64, Error> {
    match data {
        [0, rest @ ..] if rest.len() == 8 => {
            Ok(u64::from_le_bytes(rest.try_into().expect("8 bytes")))
        }
        _ => Err(Error::InvalidInstructionData),
    }
}

/// The faulty program. Accounts: 0 vault (writable, data = authority),
/// 1 authority, 2 destination (writable).
pub mod buggy {
    use super::*;

    /// The vault program, version 1.
    pub struct Vault;

    impl Program for Vault {
        fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
            let amount = amount_of(data)?;
            let vault = inv.account_mut(0)?;
            vault.lamports = vault.lamports.wrapping_sub(amount);
            let dest = inv.account_mut(2)?;
            dest.lamports = dest.lamports.wrapping_add(amount);
            Ok(())
        }
    }
}
```

<details markdown="1">
<summary>Solution</summary>

1. **No signer check.** The program never looks at whether slot 1 signed, so naming the authority's key is enough, and naming a key is free (✓ `buggy_lets_anyone_empty_the_vault_without_the_authority_signing`: the attacker signs only the fee and withdraws 48,000,000). Fix: `if !inv.is_signer(1)? { return Err(Error::NotSigner(1)) }` (✓ `fixed_requires_the_authority_signature`).
2. **A signer is not *the* signer.** The program does not compare the key in slot 1 with the authority **stored in the vault's data**, so any signer passes (✓ `buggy_lets_any_signer_withdraw_from_someone_elses_vault`). Fix: compare `vault.data` with `inv.key(1)` and return `WrongAuthority` (✓ `fixed_requires_the_stored_authority`). The same fix needs the **owner** check (`vault.owner == program_id`) and the **length** check (32 bytes) of §4.2, so that a lookalike vault from anywhere else is refused before its bytes are read.
3. **Wrapping subtraction.** `wrapping_sub` turns an overdraw into a huge balance. The *runtime* still stops the transaction, because the sum of lamports would change: `LamportsNotConserved` (✓ `buggy_overdraw_is_caught_only_by_the_runtime_conservation_check`). That is the safety net, not the design: a program should not rely on it, because it gives the user a message about the runtime's invariant instead of "insufficient funds", and because other bugs of the same family (a wrapped *counter*, a wrapped *price*) are **not** protected by conservation. Fix: `checked_sub`, with `InsufficientFunds` (✓ `fixed_refuses_to_overdraw_or_drop_below_rent`); and `checked_add` for the destination.
4. **Rent is a rule the program must know.** Taking the vault below its minimum is refused only at the *end* of the transaction, with `RentNotExempt` naming an account (I met this error myself while writing the test for report 1: my first amount of 49,000,000 left the vault with 1,000,000 lamports, below the 1,113,600 that 32 bytes need; ✓ the test now withdraws 48,000,000). A program should either leave the account above `rent_exempt_minimum` or close it fully. Fix: compute the balance after the withdrawal and return `InsufficientFunds` unless it is at least the minimum (✓ in the same test: the exact maximum, `balance − minimum`, works and one lamport more fails).

Fixed version (✓ compiled; four tests for it and three that demonstrate faults of the buggy one):

```rust
/// The corrected program.
pub mod fixed {
    use super::*;
    use solanalab::account::rent_exempt_minimum;

    /// The vault program, version 2.
    pub struct Vault;

    impl Program for Vault {
        fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
            let amount = amount_of(data)?;
            let vault = inv.account(0)?;
            if vault.owner != inv.program_id() {
                return Err(Error::WrongOwner(0));
            }
            if vault.data.len() != 32 {
                return Err(Error::InvalidAccountData);
            }
            if !inv.is_signer(1)? {
                return Err(Error::NotSigner(1));
            }
            if vault.data[..] != inv.key(1)?.0 {
                return Err(Error::WrongAuthority);
            }
            let left = vault
                .lamports
                .checked_sub(amount)
                .filter(|&l| l >= rent_exempt_minimum(vault.data.len()))
                .ok_or(Error::InsufficientFunds)?;
            let new_dest = inv
                .account(2)?
                .lamports
                .checked_add(amount)
                .ok_or(Error::ArithmeticOverflow)?;
            inv.account_mut(0)?.lamports = left;
            inv.account_mut(2)?.lamports = new_dest;
            Ok(())
        }
    }
}
```

Two notes: the fixed program **reads, validates and computes everything before it writes anything** (`left`, `new_dest`), so that a failure never leaves a half-modified account in a program that continues; and it still does not do everything a real vault must (no way to deposit, no close, no events, no limits). The runtime rolls back failed transactions anyway, but a program that calls others (CPI) should not rely on that for correctness.

</details>

---

## 14. Interview questions

1. *Why are Solana programs stateless? Where does state live?*
2. *What does the runtime check for you after a program runs, and what must the program check for itself?*
3. *What is a PDA? Why does it have no private key? How can it "sign"?*
4. *What is the canonical bump, and what goes wrong if you accept any bump?*
5. *What is a CPI? Which privileges are passed, which are checked, and what is `invoke_signed` for?*
6. *Why is the writable flag in the instruction?*
7. *What are rent and rent exemption? What happens to an account whose balance goes to zero?*
8. *What is the transaction size limit and how do you work with it?*
9. *What are compute units and what happens if you run out?*
10. *Name five checks a program should perform on an account it receives.*
11. *What is a "revival" or "closing" bug?*
12. *Why must arithmetic on balances be checked even though Rust has overflow detection?*
13. *A failed transaction: is the fee charged? Are state changes kept?*
14. *Why can the same account appear twice in an instruction, and why does that matter?*
15. *What can go wrong with a program-upgrade authority?*

<details markdown="1">
<summary>Answers</summary>

1. A program is code in an executable account, and every instruction names the accounts it reads and writes; the runtime can then run non-conflicting transactions in parallel and the chain's state is just a table of accounts (§2, §3.1, §8.3). Each state account is *owned* by the program that defines its bytes.
2. The runtime enforces ownership and money rules after the instruction: read-only unchanged, only the owner changes data or debits, executable and owner restrictions, conservation of lamports, rent (§4.1, §3.4). The program must check *authorization and identity*: owner of the passed accounts, signer, stored authority, layout, duplicates, the right program ids (§4.2).
3. A program-derived address is a hash of seeds, the program id and a marker, kept only if it is **off** the Ed25519 curve, so no private key exists. It "signs" when the owning program makes a CPI with the seeds; the runtime re-derives the address from the seeds and the *caller's* program id and, if it matches, treats it as a signer (§5, §6.1).
4. The first (highest) bump from 255 downward that yields an off-curve address. Accepting any valid bump means several valid addresses per logical key, so one can create duplicates of a "unique" account or bypass checks that assume uniqueness (§5.3; ✓ bumps 255 and 254 on the curve, 253 canonical, 252 also valid).
5. A call from one program to another. The caller's signer and writable privileges can be passed on if the caller has them; PDA signatures are added with `invoke_signed` and seeds; the runtime refuses escalation, unknown accounts, deep call chains and re-entrancy through another program, and charges compute units (§6).
6. Because it tells the scheduler which accounts a transaction may change, so transactions that do not conflict run in parallel; it also lets the runtime enforce that read-only accounts stay unchanged (§8.3, §4.1).
7. Accounts must hold a minimum balance proportional to their size (the exemption minimum) or they cannot exist; an account whose balance reaches zero is deleted at the end of the transaction, which is why a close must also clear data and owner (§2.2, §7.3).
8. 1,232 bytes for a legacy transaction (from memory). Keep instruction data small, use lookup tables for many accounts, split large uploads over several transactions, and count bytes (§3.3; ✓ 21 transfers fit, 22 do not).
9. A metering unit for execution; the transaction has a budget (default per instruction and a cap, from memory), and running out aborts the transaction, which still pays its fee (§8.1).
10. Owner, signer, writable, identity (the expected key or stored authority), data length/layout, discriminator (Lesson 54), and also the program ids of any CPI target (§4.2, §7.2).
11. Closing an account by draining lamports but leaving data and owner so that in the same transaction (or via later refunding) it can be used again with stale state; the fix is to zero data, reset the owner and ensure it cannot be revived (§7.3).
12. Release builds wrap on overflow by default, and on-chain programs run in release mode (from memory); wrapped balances turn a failed withdrawal into a huge one, and the runtime's conservation rule protects only the total (§7.2, debugging challenge).
13. The fee is charged; the state changes are discarded (§3.4).
14. Because the caller lists accounts freely. A program that adds slot 0 to slot 1 and gets the same account twice may double-count or alias a write; explicit checks (`key(0) != key(1)`) are needed where it matters (§4.2).
15. Whoever holds the upgrade authority can replace the code, so users trust that key (and its custody) as much as the code; teams freeze programs, move the authority to a multisig or governance, or time-lock upgrades (§11; Lesson 55).

</details>

---

## 15. Practical challenge: `solanalab`

Build a library crate that:

1. Provides **base58** encode/decode and a `Pubkey` newtype, with the PDA derivation (`create_program_address`, `find_program_address`) using SHA-256 and a real on-curve check.
2. Provides **accounts** and the rent-exemption rule, and **instructions and transactions** with their legacy wire-size arithmetic and the 1,232-byte limit.
3. Provides a **bank** that processes a transaction atomically: size and signature checks, the fee, a working copy, rent checks and commit.
4. Provides the **ownership verifier** (`verify_changes`) and runs it after every instruction, including nested ones.
5. Provides **CPI** with signer and writable privilege propagation, PDA signatures from seeds, depth and re-entrancy limits and a compute meter.
6. Provides a **system program** (create, assign, transfer, allocate) and a **counter program** that validates all of its accounts.
7. Has a benchmark example, passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

```toml
[package]
name = "solanalab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "Teaching model of the Solana execution model: accounts, programs, instructions, PDAs and CPI. Not audited; not the real runtime."

[dependencies]
sha2 = "0.11"
curve25519-dalek = "5"

[profile.dev.package."*"]
opt-level = 3
```

### `src/lib.rs`

```rust
//! A teaching model of the Solana execution model: accounts with owners and
//! rent, instructions, a runtime that enforces ownership rules after every
//! instruction, program-derived addresses, and cross-program invocation.
//! **A model written from memory of the design, not the real runtime; not
//! audited; not for production.**
#![deny(missing_docs)]

pub mod account;
pub mod base58;
pub mod counter;
pub mod instruction;
pub mod pubkey;
pub mod runtime;
pub mod system;
```

### `src/base58.rs`

```rust
//! Base58 (Bitcoin alphabet): the text form of 32-byte public keys.

const ALPHABET: &[u8; 58] = b"123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";

/// Why a string is not valid base58.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Base58Error {
    /// A character outside the alphabet (position in bytes).
    BadChar(usize),
}

/// Encodes bytes; each leading zero byte becomes a `1`.
pub fn encode(bytes: &[u8]) -> String {
    let zeros = bytes.iter().take_while(|&&b| b == 0).count();
    let mut digits: Vec<u8> = Vec::new(); // little-endian base-58 digits
    for &b in &bytes[zeros..] {
        let mut carry = u32::from(b);
        for d in digits.iter_mut() {
            carry += u32::from(*d) << 8;
            *d = (carry % 58) as u8;
            carry /= 58;
        }
        while carry > 0 {
            digits.push((carry % 58) as u8);
            carry /= 58;
        }
    }
    let mut s = String::with_capacity(zeros + digits.len());
    s.extend(std::iter::repeat_n('1', zeros));
    s.extend(digits.iter().rev().map(|&d| ALPHABET[d as usize] as char));
    s
}

/// Decodes a string; each leading `1` becomes a zero byte.
pub fn decode(s: &str) -> Result<Vec<u8>, Base58Error> {
    let zeros = s.bytes().take_while(|&c| c == b'1').count();
    let mut bytes: Vec<u8> = Vec::new(); // little-endian
    for (i, c) in s.bytes().enumerate().skip(zeros) {
        let v = ALPHABET
            .iter()
            .position(|&a| a == c)
            .ok_or(Base58Error::BadChar(i))?;
        let mut carry = v as u32;
        for b in bytes.iter_mut() {
            carry += u32::from(*b) * 58;
            *b = (carry & 0xff) as u8;
            carry >>= 8;
        }
        while carry > 0 {
            bytes.push((carry & 0xff) as u8);
            carry >>= 8;
        }
    }
    let mut out = vec![0u8; zeros];
    out.extend(bytes.iter().rev());
    Ok(out)
}
```

### `src/pubkey.rs`

```rust
//! Public keys and program-derived addresses (PDAs).

use crate::base58;
use curve25519_dalek::edwards::CompressedEdwardsY;
use sha2::{Digest, Sha256};
use std::fmt;

/// Most seeds a PDA derivation accepts.
pub const MAX_SEEDS: usize = 16;
/// Longest single seed in bytes.
pub const MAX_SEED_LEN: usize = 32;
const MARKER: &[u8] = b"ProgramDerivedAddress";

/// A 32-byte account address.
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
pub struct Pubkey(pub [u8; 32]);

impl Pubkey {
    /// The address with every byte equal to `b` (handy in tests).
    pub const fn filled(b: u8) -> Pubkey {
        Pubkey([b; 32])
    }

    /// Parses the base58 text form (exactly 32 bytes).
    pub fn from_base58(s: &str) -> Option<Pubkey> {
        let v = base58::decode(s).ok()?;
        Some(Pubkey(v.try_into().ok()?))
    }

    /// Is this a valid ed25519 point (so that a private key *could* exist)?
    pub fn is_on_curve(&self) -> bool {
        CompressedEdwardsY(self.0).decompress().is_some()
    }
}

impl fmt::Display for Pubkey {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(&base58::encode(&self.0))
    }
}

impl fmt::Debug for Pubkey {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Pubkey({self})")
    }
}

/// Why a derivation failed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PdaError {
    /// More than [`MAX_SEEDS`] seeds.
    TooManySeeds,
    /// A seed longer than [`MAX_SEED_LEN`].
    SeedTooLong,
    /// The hash landed on the curve, so it cannot be a PDA.
    OnCurve,
    /// No bump from 255 down to 0 produced an off-curve address.
    NoBump,
}

/// `SHA-256(seeds… ‖ program_id ‖ "ProgramDerivedAddress")`, accepted only
/// if the result is **not** a valid curve point.
pub fn create_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> Result<Pubkey, PdaError> {
    if seeds.len() > MAX_SEEDS {
        return Err(PdaError::TooManySeeds);
    }
    if seeds.iter().any(|s| s.len() > MAX_SEED_LEN) {
        return Err(PdaError::SeedTooLong);
    }
    let mut h = Sha256::new();
    for s in seeds {
        h.update(s);
    }
    h.update(program_id.0);
    h.update(MARKER);
    let key = Pubkey(h.finalize().into());
    if key.is_on_curve() {
        Err(PdaError::OnCurve)
    } else {
        Ok(key)
    }
}

/// Tries bump seeds 255, 254, … and returns the first off-curve address and
/// the bump that produced it (the *canonical* bump).
pub fn find_program_address(
    seeds: &[&[u8]],
    program_id: &Pubkey,
) -> Result<(Pubkey, u8), PdaError> {
    if seeds.len() >= MAX_SEEDS {
        return Err(PdaError::TooManySeeds);
    }
    for bump in (0..=255u8).rev() {
        let b = [bump];
        let mut all: Vec<&[u8]> = seeds.to_vec();
        all.push(&b);
        match create_program_address(&all, program_id) {
            Ok(k) => return Ok((k, bump)),
            Err(PdaError::OnCurve) => continue,
            Err(e) => return Err(e),
        }
    }
    Err(PdaError::NoBump)
}
```

### `src/account.rs`

```rust
//! Accounts and rent.

use crate::pubkey::Pubkey;

/// Bytes of per-account overhead counted by rent.
pub const ACCOUNT_OVERHEAD: u64 = 128;
/// Lamports per byte-year (from memory of the default rent parameters).
pub const LAMPORTS_PER_BYTE_YEAR: u64 = 3480;
/// Years of rent that make an account exempt.
pub const EXEMPTION_YEARS: u64 = 2;

/// Minimum balance for an account with `data_len` bytes to stay alive.
pub fn rent_exempt_minimum(data_len: usize) -> u64 {
    (ACCOUNT_OVERHEAD + data_len as u64) * LAMPORTS_PER_BYTE_YEAR * EXEMPTION_YEARS
}

/// The only kind of state on the chain: a balance, some bytes, and an owner.
#[derive(Debug, Clone, PartialEq, Eq, Default)]
pub struct Account {
    /// Balance in lamports (1 SOL = 10⁹ lamports).
    pub lamports: u64,
    /// Arbitrary state, writable only by the owner program.
    pub data: Vec<u8>,
    /// The program that may change `data` and debit `lamports`.
    pub owner: Pubkey,
    /// Is this account a program?
    pub executable: bool,
}

impl Account {
    /// A system-owned wallet account with `lamports`.
    pub fn wallet(lamports: u64) -> Account {
        Account {
            lamports,
            ..Account::default()
        }
    }
}
```

### `src/instruction.rs`

```rust
//! Instructions and transactions.

use crate::pubkey::Pubkey;
use std::collections::BTreeSet;

/// One account an instruction touches, with the privileges it asks for.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct AccountMeta {
    /// Which account.
    pub pubkey: Pubkey,
    /// Must have authorized the transaction (or be a PDA signed for by its program).
    pub is_signer: bool,
    /// May be modified.
    pub is_writable: bool,
}

impl AccountMeta {
    /// A writable account.
    pub fn writable(pubkey: Pubkey, is_signer: bool) -> AccountMeta {
        AccountMeta {
            pubkey,
            is_signer,
            is_writable: true,
        }
    }
    /// A read-only account.
    pub fn readonly(pubkey: Pubkey, is_signer: bool) -> AccountMeta {
        AccountMeta {
            pubkey,
            is_signer,
            is_writable: false,
        }
    }
}

/// A call to one program: which program, which accounts, which bytes.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Instruction {
    /// The program to run.
    pub program_id: Pubkey,
    /// The accounts, in the order the program expects them.
    pub accounts: Vec<AccountMeta>,
    /// Program-specific arguments.
    pub data: Vec<u8>,
}

/// Instructions that succeed or fail together, authorized by `signers`.
/// The first signer pays the fee. (Signature verification is assumed done.)
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Transaction {
    /// Keys that signed; the first is the fee payer.
    pub signers: Vec<Pubkey>,
    /// Executed in order.
    pub instructions: Vec<Instruction>,
}

/// Largest serialized transaction (from memory: the legacy packet limit).
pub const MAX_TX_BYTES: usize = 1232;

fn shortvec_len(n: usize) -> usize {
    match n {
        0..=0x7f => 1,
        0x80..=0x3fff => 2,
        _ => 3,
    }
}

impl Transaction {
    /// Distinct account keys: signers first, then every other key in first-use order.
    pub fn account_keys(&self) -> Vec<Pubkey> {
        let mut seen = BTreeSet::new();
        let mut keys = Vec::new();
        let others = self.instructions.iter().flat_map(|ix| {
            ix.accounts
                .iter()
                .map(|m| m.pubkey)
                .chain(std::iter::once(ix.program_id))
        });
        for k in self.signers.iter().copied().chain(others) {
            if seen.insert(k) {
                keys.push(k);
            }
        }
        keys
    }

    /// Size of the legacy wire format: signatures, header, keys, recent
    /// blockhash and compiled instructions (from memory of the layout).
    pub fn serialized_len(&self) -> usize {
        let keys = self.account_keys().len();
        let mut n = shortvec_len(self.signers.len()) + 64 * self.signers.len();
        n += 3; // header
        n += shortvec_len(keys) + 32 * keys;
        n += 32; // recent blockhash
        n += shortvec_len(self.instructions.len());
        for ix in &self.instructions {
            n += 1 + shortvec_len(ix.accounts.len()) + ix.accounts.len();
            n += shortvec_len(ix.data.len()) + ix.data.len();
        }
        n
    }
}
```

### `src/runtime.rs`

```rust
//! The runtime: loads accounts, runs programs, and **checks every change a
//! program made against the ownership rules** before accepting it.

use crate::account::{Account, rent_exempt_minimum};
use crate::instruction::{AccountMeta, MAX_TX_BYTES, Transaction};
use crate::pubkey::{Pubkey, create_program_address};
use std::collections::{BTreeMap, BTreeSet};

/// Fee per signature, in lamports (from memory of the default).
pub const FEE_PER_SIGNATURE: u64 = 5000;
/// How many CPI levels may be nested below the top-level instruction.
pub const MAX_CPI_DEPTH: usize = 4;
/// Compute units granted per top-level instruction (from memory).
pub const UNITS_PER_INSTRUCTION: u64 = 200_000;
/// Hard cap on the units of one transaction (from memory).
pub const MAX_UNITS: u64 = 1_400_000;
/// Units charged for making a cross-program invocation (model value).
pub const CPI_COST: u64 = 1000;

/// Everything that can go wrong, from the transaction envelope to a program.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Error {
    /// The serialized transaction exceeds [`MAX_TX_BYTES`].
    TxTooLarge(usize),
    /// A transaction needs at least one signer (the fee payer).
    NoSigners,
    /// The fee payer cannot pay the fee.
    InsufficientFeePayer,
    /// An instruction asks for a signature the transaction does not carry.
    MissingSignature(Pubkey),
    /// No program is registered under this id.
    UnknownProgram(Pubkey),
    /// The program account is not marked executable.
    NotExecutable(Pubkey),
    /// A read-only account was modified.
    ReadonlyModified(Pubkey),
    /// Lamports were taken from an account the program does not own.
    UnauthorizedDebit(Pubkey),
    /// Data was changed on an account the program does not own.
    UnauthorizedDataChange(Pubkey),
    /// The owner was changed illegally.
    UnauthorizedOwnerChange(Pubkey),
    /// The executable flag was changed.
    ExecutableChanged(Pubkey),
    /// The total of lamports across the instruction's accounts changed.
    LamportsNotConserved,
    /// A modified account is neither empty nor rent-exempt.
    RentNotExempt(Pubkey),
    /// A CPI asked for a signature or write privilege the caller lacks.
    PrivilegeEscalation(Pubkey),
    /// CPI nested too deeply.
    DepthExceeded,
    /// A program called back into a program already on the stack.
    ReentrancyNotAllowed,
    /// The compute budget ran out.
    ComputeExceeded,
    /// A CPI mentions an account the caller was not given.
    MissingAccount(usize),
    /// Malformed instruction data.
    InvalidInstructionData,
    /// Account `i` had to be a signer and was not.
    NotSigner(usize),
    /// Account `i` had to be writable and was not.
    NotWritable(usize),
    /// Account `i` has the wrong owner.
    WrongOwner(usize),
    /// Not enough lamports.
    InsufficientFunds,
    /// The target account is already in use.
    AlreadyInUse,
    /// The stored authority does not match the supplied one.
    WrongAuthority,
    /// PDA seeds or bump are wrong.
    InvalidSeeds,
    /// Account data has the wrong size or shape.
    InvalidAccountData,
    /// Checked arithmetic failed.
    ArithmeticOverflow,
}

/// A program: a function from (accounts, bytes) to success or an error.
pub trait Program {
    /// Runs one instruction. All account access goes through `inv`.
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error>;
}

struct Frame {
    program_id: Pubkey,
    metas: Vec<AccountMeta>,
    before: BTreeMap<Pubkey, Account>,
}

/// What a running program can see and do.
pub struct Invoke<'a> {
    work: &'a mut BTreeMap<Pubkey, Account>,
    programs: &'a BTreeMap<Pubkey, Box<dyn Program>>,
    stack: Vec<Frame>,
    remaining: u64,
}

/// Checks the changes made by an instruction run by `program_id`.
/// `writable` lists the instruction's distinct keys and whether any of its
/// metas for that key was writable.
pub fn verify_changes(
    program_id: &Pubkey,
    writable: &BTreeMap<Pubkey, bool>,
    before: &BTreeMap<Pubkey, Account>,
    after: &BTreeMap<Pubkey, Account>,
) -> Result<(), Error> {
    let mut sum_before: u128 = 0;
    let mut sum_after: u128 = 0;
    for (key, &is_writable) in writable {
        let pre = &before[key];
        let post = &after[key];
        sum_before += u128::from(pre.lamports);
        sum_after += u128::from(post.lamports);
        if !is_writable {
            if pre != post {
                return Err(Error::ReadonlyModified(*key));
            }
            continue;
        }
        if post.executable != pre.executable {
            return Err(Error::ExecutableChanged(*key));
        }
        let owns = pre.owner == *program_id;
        if post.lamports < pre.lamports && !owns {
            return Err(Error::UnauthorizedDebit(*key));
        }
        if post.data != pre.data && (!owns || pre.executable) {
            return Err(Error::UnauthorizedDataChange(*key));
        }
        if post.owner != pre.owner && (!owns || pre.executable || post.data.iter().any(|&b| b != 0))
        {
            return Err(Error::UnauthorizedOwnerChange(*key));
        }
    }
    if sum_before != sum_after {
        return Err(Error::LamportsNotConserved);
    }
    Ok(())
}

impl<'a> Invoke<'a> {
    fn frame(&self) -> &Frame {
        self.stack.last().expect("a program is running")
    }

    /// The id of the running program.
    pub fn program_id(&self) -> Pubkey {
        self.frame().program_id
    }

    /// Number of accounts passed to this instruction.
    pub fn len(&self) -> usize {
        self.frame().metas.len()
    }

    /// True if no accounts were passed.
    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }

    fn meta(&self, i: usize) -> Result<&AccountMeta, Error> {
        self.frame().metas.get(i).ok_or(Error::MissingAccount(i))
    }

    /// The address of account `i`.
    pub fn key(&self, i: usize) -> Result<Pubkey, Error> {
        Ok(self.meta(i)?.pubkey)
    }

    /// Did account `i` sign (or is it a PDA the caller signed for)?
    pub fn is_signer(&self, i: usize) -> Result<bool, Error> {
        Ok(self.meta(i)?.is_signer)
    }

    /// Was account `i` passed as writable?
    pub fn is_writable(&self, i: usize) -> Result<bool, Error> {
        Ok(self.meta(i)?.is_writable)
    }

    /// Read access to account `i`.
    pub fn account(&self, i: usize) -> Result<&Account, Error> {
        let k = self.key(i)?;
        Ok(&self.work[&k])
    }

    /// Write access to account `i`. The runtime does **not** check at this
    /// point; it checks everything when the instruction returns.
    pub fn account_mut(&mut self, i: usize) -> Result<&mut Account, Error> {
        let k = self.key(i)?;
        Ok(self.work.get_mut(&k).expect("loaded"))
    }

    /// Spends compute units.
    pub fn consume(&mut self, units: u64) -> Result<(), Error> {
        if units > self.remaining {
            self.remaining = 0;
            return Err(Error::ComputeExceeded);
        }
        self.remaining -= units;
        Ok(())
    }

    /// Cross-program invocation. `signer_seeds` lists PDAs of **this**
    /// program that the callee may treat as signers.
    pub fn invoke(
        &mut self,
        ix: &crate::instruction::Instruction,
        signer_seeds: &[&[&[u8]]],
    ) -> Result<(), Error> {
        self.consume(CPI_COST)?;
        let caller_id = self.program_id();
        let mut pda_signers = BTreeSet::new();
        for seeds in signer_seeds {
            let k = create_program_address(seeds, &caller_id).map_err(|_| Error::InvalidSeeds)?;
            pda_signers.insert(k);
        }
        for (n, m) in ix.accounts.iter().enumerate() {
            let held: Vec<&AccountMeta> = self
                .frame()
                .metas
                .iter()
                .filter(|c| c.pubkey == m.pubkey)
                .collect();
            if held.is_empty() {
                return Err(Error::MissingAccount(n));
            }
            if m.is_signer && !(held.iter().any(|c| c.is_signer) || pda_signers.contains(&m.pubkey))
            {
                return Err(Error::PrivilegeEscalation(m.pubkey));
            }
            if m.is_writable && !held.iter().any(|c| c.is_writable) {
                return Err(Error::PrivilegeEscalation(m.pubkey));
            }
        }
        self.run(ix.program_id, ix.accounts.clone(), &ix.data)
    }

    fn run(
        &mut self,
        program_id: Pubkey,
        metas: Vec<AccountMeta>,
        data: &[u8],
    ) -> Result<(), Error> {
        if self.stack.len() > MAX_CPI_DEPTH {
            return Err(Error::DepthExceeded);
        }
        if let Some(last) = self.stack.last()
            && last.program_id != program_id
            && self.stack.iter().any(|f| f.program_id == program_id)
        {
            return Err(Error::ReentrancyNotAllowed);
        }
        let programs = self.programs;
        let program = programs
            .get(&program_id)
            .ok_or(Error::UnknownProgram(program_id))?;
        if !self.work.get(&program_id).is_some_and(|a| a.executable) {
            return Err(Error::NotExecutable(program_id));
        }
        let mut writable: BTreeMap<Pubkey, bool> = BTreeMap::new();
        for m in &metas {
            *writable.entry(m.pubkey).or_insert(false) |= m.is_writable;
            self.work.entry(m.pubkey).or_default();
        }
        let before: BTreeMap<Pubkey, Account> = writable
            .keys()
            .map(|k| (*k, self.work[k].clone()))
            .collect();
        self.stack.push(Frame {
            program_id,
            metas,
            before: before.clone(),
        });
        let result = program.process(self, data);
        let frame = self.stack.pop().expect("pushed above");
        result?;
        // `frame.before` also reflects what nested calls legitimately changed
        verify_changes(&program_id, &writable, &frame.before, self.work)?;
        if let Some(caller) = self.stack.last_mut() {
            for k in before.keys() {
                if caller.before.contains_key(k) {
                    caller.before.insert(*k, self.work[k].clone());
                }
            }
        }
        Ok(())
    }
}

/// What a successful transaction cost.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Receipt {
    /// Fee charged.
    pub fee: u64,
    /// Compute units used.
    pub units_used: u64,
}

/// All accounts and registered programs.
pub struct Bank {
    accounts: BTreeMap<Pubkey, Account>,
    programs: BTreeMap<Pubkey, Box<dyn Program>>,
    /// Fees taken so far (they leave circulation in this model).
    pub fees_collected: u64,
}

/// The owner of every program account in this model.
pub const LOADER: Pubkey = Pubkey::filled(0xEE);

impl Default for Bank {
    fn default() -> Self {
        Bank::new()
    }
}

impl Bank {
    /// A bank with the system program installed.
    pub fn new() -> Bank {
        let mut b = Bank {
            accounts: BTreeMap::new(),
            programs: BTreeMap::new(),
            fees_collected: 0,
        };
        b.add_program(Pubkey::default(), Box::new(crate::system::SystemProgram));
        b
    }

    /// Registers a program and creates its executable account.
    pub fn add_program(&mut self, id: Pubkey, program: Box<dyn Program>) {
        self.programs.insert(id, program);
        self.accounts.insert(
            id,
            Account {
                lamports: 1,
                data: Vec::new(),
                owner: LOADER,
                executable: true,
            },
        );
    }

    /// Creates or replaces an account (test setup).
    pub fn set_account(&mut self, key: Pubkey, account: Account) {
        self.accounts.insert(key, account);
    }

    /// Looks up an account.
    pub fn account(&self, key: &Pubkey) -> Option<&Account> {
        self.accounts.get(key)
    }

    /// Balance of an account (0 if it does not exist).
    pub fn lamports(&self, key: &Pubkey) -> u64 {
        self.accounts.get(key).map_or(0, |a| a.lamports)
    }

    /// Total lamports in existence plus fees collected (conserved by every transaction).
    pub fn total_supply(&self) -> u128 {
        self.accounts
            .values()
            .map(|a| u128::from(a.lamports))
            .sum::<u128>()
            + u128::from(self.fees_collected)
    }

    /// Runs a transaction **atomically**: either every instruction succeeds
    /// and all changes are kept, or only the fee is.
    pub fn process(&mut self, tx: &Transaction) -> Result<Receipt, Error> {
        let size = tx.serialized_len();
        if size > MAX_TX_BYTES {
            return Err(Error::TxTooLarge(size));
        }
        let payer = *tx.signers.first().ok_or(Error::NoSigners)?;
        for ix in &tx.instructions {
            for m in &ix.accounts {
                if m.is_signer && !tx.signers.contains(&m.pubkey) {
                    return Err(Error::MissingSignature(m.pubkey));
                }
            }
        }
        let fee = FEE_PER_SIGNATURE * tx.signers.len() as u64;
        let (have, data_len) = self
            .accounts
            .get(&payer)
            .map_or((0, 0), |a| (a.lamports, a.data.len()));
        match have.checked_sub(fee) {
            // the payer must keep a rent-exempt balance (or end at zero)
            Some(left) if left == 0 || left >= rent_exempt_minimum(data_len) => {}
            _ => return Err(Error::InsufficientFeePayer),
        }
        self.accounts.get_mut(&payer).expect("exists").lamports -= fee;
        self.fees_collected += fee;

        let total = (UNITS_PER_INSTRUCTION * tx.instructions.len() as u64).min(MAX_UNITS);
        let mut work = self.accounts.clone();
        let remaining = {
            let mut inv = Invoke {
                work: &mut work,
                programs: &self.programs,
                stack: Vec::new(),
                remaining: total,
            };
            for ix in &tx.instructions {
                inv.run(ix.program_id, ix.accounts.clone(), &ix.data)?;
            }
            inv.remaining
        };
        for (k, a) in &work {
            let old = self.accounts.get(k).cloned().unwrap_or_default();
            if *a != old && a.lamports != 0 && a.lamports < rent_exempt_minimum(a.data.len()) {
                return Err(Error::RentNotExempt(*k));
            }
        }
        for (k, a) in work {
            if a.lamports == 0 {
                self.accounts.remove(&k);
            } else {
                self.accounts.insert(k, a);
            }
        }
        Ok(Receipt {
            fee,
            units_used: total - remaining,
        })
    }
}
```

### `src/system.rs`

```rust
//! The system program: creates accounts, assigns owners and moves lamports
//! between accounts it owns. Instruction data is a `u32` tag (little-endian)
//! followed by little-endian fields (from memory of the real layout).

use crate::account::Account;
use crate::pubkey::Pubkey;
use crate::runtime::{Error, Invoke, Program};

/// Largest account data the system program will allocate (from memory: 10 MiB).
pub const MAX_DATA_LEN: u64 = 10 * 1024 * 1024;
/// Compute units a system instruction costs in this model.
pub const COST: u64 = 150;

/// The system program (its id is the all-zero key).
pub struct SystemProgram;

fn u64_at(d: &[u8], at: usize) -> u64 {
    u64::from_le_bytes(d[at..at + 8].try_into().expect("8 bytes"))
}

/// Builds `CreateAccount` data.
pub fn create_account_data(lamports: u64, space: u64, owner: &Pubkey) -> Vec<u8> {
    let mut v = 0u32.to_le_bytes().to_vec();
    v.extend_from_slice(&lamports.to_le_bytes());
    v.extend_from_slice(&space.to_le_bytes());
    v.extend_from_slice(&owner.0);
    v
}

/// Builds `Assign` data.
pub fn assign_data(owner: &Pubkey) -> Vec<u8> {
    let mut v = 1u32.to_le_bytes().to_vec();
    v.extend_from_slice(&owner.0);
    v
}

/// Builds `Transfer` data.
pub fn transfer_data(lamports: u64) -> Vec<u8> {
    let mut v = 2u32.to_le_bytes().to_vec();
    v.extend_from_slice(&lamports.to_le_bytes());
    v
}

/// Builds `Allocate` data.
pub fn allocate_data(space: u64) -> Vec<u8> {
    let mut v = 8u32.to_le_bytes().to_vec();
    v.extend_from_slice(&space.to_le_bytes());
    v
}

fn need_signer(inv: &Invoke<'_>, i: usize) -> Result<(), Error> {
    if inv.is_signer(i)? {
        Ok(())
    } else {
        Err(Error::NotSigner(i))
    }
}

fn move_lamports(inv: &mut Invoke<'_>, from: usize, to: usize, amount: u64) -> Result<(), Error> {
    let src = inv.account(from)?;
    if src.owner != Pubkey::default() {
        return Err(Error::WrongOwner(from));
    }
    if !src.data.is_empty() {
        return Err(Error::InvalidAccountData);
    }
    if src.lamports < amount {
        return Err(Error::InsufficientFunds);
    }
    let dest = inv.account(to)?.lamports;
    let new_dest = dest.checked_add(amount).ok_or(Error::ArithmeticOverflow)?;
    inv.account_mut(from)?.lamports -= amount;
    inv.account_mut(to)?.lamports = new_dest;
    Ok(())
}

impl Program for SystemProgram {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(COST)?;
        let tag = u32::from_le_bytes(
            data.get(..4)
                .ok_or(Error::InvalidInstructionData)?
                .try_into()
                .expect("4 bytes"),
        );
        match (tag, data.len()) {
            (0, 52) => {
                let (lamports, space) = (u64_at(data, 4), u64_at(data, 12));
                let owner = Pubkey(data[20..52].try_into().expect("32 bytes"));
                need_signer(inv, 0)?;
                need_signer(inv, 1)?;
                if space > MAX_DATA_LEN || inv.key(0)? == inv.key(1)? {
                    return Err(Error::InvalidInstructionData);
                }
                let new = inv.account(1)?;
                if !new.data.is_empty() || new.owner != Pubkey::default() {
                    return Err(Error::AlreadyInUse);
                }
                move_lamports(inv, 0, 1, lamports)?;
                let new = inv.account_mut(1)?;
                new.data = vec![0; space as usize];
                new.owner = owner;
                Ok(())
            }
            (1, 36) => {
                let owner = Pubkey(data[4..36].try_into().expect("32 bytes"));
                need_signer(inv, 0)?;
                if inv.account(0)?.owner != Pubkey::default() {
                    return Err(Error::WrongOwner(0));
                }
                inv.account_mut(0)?.owner = owner;
                Ok(())
            }
            (2, 12) => {
                need_signer(inv, 0)?;
                move_lamports(inv, 0, 1, u64_at(data, 4))
            }
            (8, 12) => {
                need_signer(inv, 0)?;
                let space = u64_at(data, 4);
                let acc: &Account = inv.account(0)?;
                if space > MAX_DATA_LEN || !acc.data.is_empty() || acc.owner != Pubkey::default() {
                    return Err(Error::AlreadyInUse);
                }
                inv.account_mut(0)?.data = vec![0; space as usize];
                Ok(())
            }
            _ => Err(Error::InvalidInstructionData),
        }
    }
}
```

### `src/counter.rs`

```rust
//! An example program: a per-authority counter stored in a PDA account.
//! It shows the checks every program must make for itself (owner, signer,
//! authority, size) and a CPI signed by a PDA.

use crate::account::rent_exempt_minimum;
use crate::instruction::{AccountMeta, Instruction};
use crate::pubkey::{Pubkey, create_program_address, find_program_address};
use crate::runtime::{Error, Invoke, Program};
use crate::system;

/// Size of the counter's data: authority (32) + count (8).
pub const SPACE: usize = 40;
/// Compute units one counter instruction costs in this model.
pub const COST: u64 = 100;

/// The counter program.
pub struct CounterProgram;

/// The PDA of `authority`'s counter, and its canonical bump.
pub fn counter_address(program_id: &Pubkey, authority: &Pubkey) -> (Pubkey, u8) {
    find_program_address(&[b"counter", &authority.0], program_id).expect("a bump exists")
}

/// `Create` instruction.
pub fn create_ix(program_id: Pubkey, payer: Pubkey, counter: Pubkey, bump: u8) -> Instruction {
    Instruction {
        program_id,
        accounts: vec![
            AccountMeta::writable(payer, true),
            AccountMeta::writable(counter, false),
            AccountMeta::readonly(Pubkey::default(), false),
        ],
        data: vec![0, bump],
    }
}

/// `Increment` instruction.
pub fn increment_ix(program_id: Pubkey, counter: Pubkey, authority: Pubkey) -> Instruction {
    Instruction {
        program_id,
        accounts: vec![
            AccountMeta::writable(counter, false),
            AccountMeta::readonly(authority, true),
        ],
        data: vec![1],
    }
}

/// `Close` instruction (returns the lamports to the authority).
pub fn close_ix(program_id: Pubkey, counter: Pubkey, authority: Pubkey) -> Instruction {
    Instruction {
        program_id,
        accounts: vec![
            AccountMeta::writable(counter, false),
            AccountMeta::writable(authority, true),
        ],
        data: vec![2],
    }
}

/// Reads the count of a counter account.
pub fn count_of(data: &[u8]) -> Option<u64> {
    Some(u64::from_le_bytes(data.get(32..40)?.try_into().ok()?))
}

/// The checks shared by `Increment` and `Close`.
fn check_counter(inv: &Invoke<'_>) -> Result<(), Error> {
    let counter = inv.account(0)?;
    if counter.owner != inv.program_id() {
        return Err(Error::WrongOwner(0));
    }
    if !inv.is_writable(0)? {
        return Err(Error::NotWritable(0));
    }
    if counter.data.len() != SPACE {
        return Err(Error::InvalidAccountData);
    }
    if !inv.is_signer(1)? {
        return Err(Error::NotSigner(1));
    }
    if counter.data[..32] != inv.key(1)?.0 {
        return Err(Error::WrongAuthority);
    }
    Ok(())
}

impl Program for CounterProgram {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(COST)?;
        match data {
            [0, bump] => {
                if !inv.is_signer(0)? {
                    return Err(Error::NotSigner(0));
                }
                if inv.key(2)? != Pubkey::default() {
                    return Err(Error::InvalidInstructionData);
                }
                let (payer, counter_key) = (inv.key(0)?, inv.key(1)?);
                inv.consume(1500)?; // searching for the canonical bump
                let (expected, canonical) = counter_address(&inv.program_id(), &payer);
                if expected != counter_key || canonical != *bump {
                    return Err(Error::InvalidSeeds);
                }
                let existing = inv.account(1)?;
                if !existing.data.is_empty() || existing.owner != Pubkey::default() {
                    return Err(Error::AlreadyInUse);
                }
                let lamports = rent_exempt_minimum(SPACE).saturating_sub(existing.lamports);
                let bump_seed = [*bump];
                let seeds: &[&[u8]] = &[b"counter", &payer.0, &bump_seed];
                debug_assert!(create_program_address(seeds, &inv.program_id()).is_ok());
                let ix = Instruction {
                    program_id: Pubkey::default(),
                    accounts: vec![
                        AccountMeta::writable(payer, true),
                        AccountMeta::writable(counter_key, true),
                    ],
                    data: system::create_account_data(lamports, SPACE as u64, &inv.program_id()),
                };
                inv.invoke(&ix, &[seeds])?;
                let acc = inv.account_mut(1)?;
                acc.data[..32].copy_from_slice(&payer.0);
                Ok(())
            }
            [1] => {
                check_counter(inv)?;
                let acc = inv.account_mut(0)?;
                let n = count_of(&acc.data).ok_or(Error::InvalidAccountData)?;
                let n = n.checked_add(1).ok_or(Error::ArithmeticOverflow)?;
                acc.data[32..40].copy_from_slice(&n.to_le_bytes());
                Ok(())
            }
            [2] => {
                check_counter(inv)?;
                if !inv.is_writable(1)? {
                    return Err(Error::NotWritable(1));
                }
                let lamports = inv.account(0)?.lamports;
                let to = inv.account(1)?.lamports;
                let total = to.checked_add(lamports).ok_or(Error::ArithmeticOverflow)?;
                inv.account_mut(1)?.lamports = total;
                let acc = inv.account_mut(0)?;
                acc.lamports = 0;
                acc.data = Vec::new();
                acc.owner = Pubkey::default();
                Ok(())
            }
            _ => Err(Error::InvalidInstructionData),
        }
    }
}
```

### `tests/base58_pubkey.rs`

```rust
use solanalab::account::*;
use solanalab::base58::*;
use solanalab::pubkey::*;

fn hex(s: &str) -> Vec<u8> {
    (0..s.len() / 2)
        .map(|i| u8::from_str_radix(&s[2 * i..2 * i + 2], 16).unwrap())
        .collect()
}

#[test]
fn base58_known_answers() {
    assert_eq!(encode(&[]), "");
    assert_eq!(encode(&[0; 32]), "11111111111111111111111111111111");
    assert_eq!(
        encode(&[1; 32]),
        "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi"
    );
    assert_eq!(encode(b"\0\0abc"), "11ZiCa");
    let seq: Vec<u8> = (0..32).collect();
    assert_eq!(encode(&seq), "1thX6LZfHDZZKUs92febYZhYRcXddmzfzF2NvTkPNE");
}

#[test]
fn base58_decode_known_and_round_trip() {
    assert_eq!(decode("11ZiCa").unwrap(), b"\0\0abc");
    assert_eq!(decode("").unwrap(), Vec::<u8>::new());
    assert_eq!(decode("1").unwrap(), vec![0]);
    let mut x = 0x1234_5678_9abc_def1u64;
    for len in 0..70usize {
        let mut v = Vec::new();
        for _ in 0..len {
            x ^= x << 13;
            x ^= x >> 7;
            x ^= x << 17;
            v.push((x >> 24) as u8);
        }
        if len % 3 == 0 && len > 2 {
            v[0] = 0;
            v[1] = 0;
        }
        assert_eq!(decode(&encode(&v)).unwrap(), v, "len {len}");
    }
}

#[test]
fn base58_rejects_bad_characters() {
    for (s, at) in [
        ("0abc", 0),
        ("ab0c", 2),
        ("abOc", 2),
        ("abIc", 2),
        ("abl", 2),
        ("ab c", 2),
        ("é", 0),
    ] {
        assert_eq!(decode(s), Err(Base58Error::BadChar(at)), "{s}");
    }
}

#[test]
fn pubkey_text_form() {
    assert_eq!(
        Pubkey::default().to_string(),
        "11111111111111111111111111111111"
    );
    assert_eq!(
        format!("{:?}", Pubkey::filled(1)),
        "Pubkey(4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi)"
    );
    assert_eq!(
        Pubkey::from_base58("4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi"),
        Some(Pubkey::filled(1))
    );
    assert_eq!(Pubkey::from_base58("11ZiCa"), None); // wrong length
    assert_eq!(Pubkey::from_base58("0"), None);
    assert_eq!(Pubkey::from_base58(&"1".repeat(33)), None);
}

#[test]
fn rent_minimums() {
    assert_eq!(rent_exempt_minimum(0), 890_880);
    assert_eq!(rent_exempt_minimum(40), 1_169_280);
    assert_eq!(rent_exempt_minimum(1000), 7_850_880);
    assert_eq!(Account::wallet(5).lamports, 5);
}

#[test]
fn pda_known_answers_with_canonical_bumps() {
    let pid = Pubkey::filled(1);
    let (a, bump) = find_program_address(&[b"counter"], &pid).unwrap();
    assert_eq!(
        (a.to_string().as_str(), bump),
        ("6gxDfzF9Tr94UKgnThhCFLwYgj8z31YGKhKQkiTBJDAb", 255)
    );
    assert_eq!(
        a.0.to_vec(),
        hex("5486859cfff43f5b963168d98817e81262604ef3703610c8b1a7a81ae3120024")
    );
    let (c, bump) = find_program_address(&[b"counter", &[2; 32]], &pid).unwrap();
    assert_eq!(
        (c.to_string().as_str(), bump),
        ("A8e1wdCK4icXgmfxNnVgkkbyXothp9JwHyqKGtT6azNj", 255)
    );
    // seeds whose first candidate (bump 255) is on the curve
    for (i, want_bump, prefix) in [
        (0u8, 254u8, "7442f06a171de87d"),
        (3, 254, "d512d020010de8c8"),
        (1, 255, "13e486665ac01392"),
    ] {
        let (k, b) = find_program_address(&[b"seed", &[i]], &pid).unwrap();
        assert_eq!(b, want_bump, "seed {i}");
        assert_eq!(k.0[..8].to_vec(), hex(prefix), "seed {i}");
    }
}

#[test]
fn create_program_address_rejects_on_curve_results() {
    let pid = Pubkey::filled(1);
    // bump 255 is on the curve for these seeds (see the previous test), so it must fail
    assert_eq!(
        create_program_address(&[b"seed", &[0], &[255]], &pid),
        Err(PdaError::OnCurve)
    );
    assert!(create_program_address(&[b"seed", &[0], &[254]], &pid).is_ok());
    assert_eq!(
        create_program_address(&[b"seed", &[3], &[255]], &pid),
        Err(PdaError::OnCurve)
    );
}

#[test]
fn pda_is_never_on_the_curve_and_depends_on_every_input() {
    let pid = Pubkey::filled(7);
    let mut seen = std::collections::BTreeSet::new();
    for i in 0..100u8 {
        let (k, _) = find_program_address(&[b"x", &[i]], &pid).unwrap();
        assert!(!k.is_on_curve());
        assert!(seen.insert(k));
    }
    let (a, _) = find_program_address(&[b"x"], &pid).unwrap();
    let (b, _) = find_program_address(&[b"x"], &Pubkey::filled(8)).unwrap();
    let (c, _) = find_program_address(&[b"y"], &pid).unwrap();
    assert!(a != b && a != c);
}

#[test]
fn seed_limits() {
    let pid = Pubkey::filled(1);
    let long = [0u8; 33];
    assert_eq!(
        create_program_address(&[&long], &pid),
        Err(PdaError::SeedTooLong)
    );
    let ok32 = [0u8; 32];
    assert!(!matches!(
        create_program_address(&[&ok32], &pid),
        Err(PdaError::SeedTooLong)
    ));
    let seeds: Vec<&[u8]> = vec![b"a"; 17];
    assert_eq!(
        create_program_address(&seeds, &pid),
        Err(PdaError::TooManySeeds)
    );
    let sixteen: Vec<&[u8]> = vec![b"a"; 16];
    assert!(!matches!(
        create_program_address(&sixteen, &pid),
        Err(PdaError::TooManySeeds)
    ));
    // find_program_address needs room for the bump seed
    assert_eq!(
        find_program_address(&sixteen, &pid),
        Err(PdaError::TooManySeeds)
    );
    let fifteen: Vec<&[u8]> = vec![b"a"; 15];
    assert!(find_program_address(&fifteen, &pid).is_ok());
}

#[test]
fn roughly_half_of_random_hashes_are_on_the_curve() {
    let mut on = 0;
    for i in 0..2000u32 {
        use sha2::{Digest, Sha256};
        let h: [u8; 32] = Sha256::digest(i.to_le_bytes()).into();
        on += usize::from(Pubkey(h).is_on_curve());
    }
    assert!((850..1150).contains(&on), "{on}");
    assert!(
        Pubkey(
            hex("0100000000000000000000000000000000000000000000000000000000000000")
                .try_into()
                .unwrap()
        )
        .is_on_curve()
    );
}
```

### `tests/runtime_tests.rs`

```rust
use solanalab::account::*;
use solanalab::instruction::*;
use solanalab::pubkey::Pubkey;
use solanalab::runtime::*;
use solanalab::system::*;

const SYS: Pubkey = Pubkey([0; 32]);

fn k(n: u8) -> Pubkey {
    Pubkey::filled(n)
}

fn tx(signers: &[Pubkey], ixs: Vec<Instruction>) -> Transaction {
    Transaction {
        signers: signers.to_vec(),
        instructions: ixs,
    }
}

fn transfer(from: Pubkey, to: Pubkey, n: u64) -> Instruction {
    Instruction {
        program_id: SYS,
        accounts: vec![
            AccountMeta::writable(from, true),
            AccountMeta::writable(to, false),
        ],
        data: transfer_data(n),
    }
}

/// A program defined by a closure, for testing the runtime's rules.
struct F<T: Fn(&mut Invoke<'_>, &[u8]) -> Result<(), Error>>(T);
impl<T: Fn(&mut Invoke<'_>, &[u8]) -> Result<(), Error>> Program for F<T> {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        (self.0)(inv, data)
    }
}

const RICH: u64 = 100_000_000;

fn bank_with(accts: &[(u8, u64)]) -> Bank {
    let mut b = Bank::new();
    for &(n, l) in accts {
        b.set_account(k(n), Account::wallet(l));
    }
    b
}

#[test]
fn transfer_moves_lamports_and_charges_the_fee() {
    let mut b = bank_with(&[(1, RICH)]);
    let r = b
        .process(&tx(&[k(1)], vec![transfer(k(1), k(2), 2_000_000)]))
        .unwrap();
    assert_eq!(r.fee, 5000);
    assert_eq!(b.lamports(&k(1)), RICH - 2_000_000 - 5000);
    assert_eq!(b.lamports(&k(2)), 2_000_000);
    assert_eq!(b.fees_collected, 5000);
    assert_eq!(r.units_used, 150);
}

#[test]
fn a_new_account_below_the_rent_minimum_is_refused() {
    let mut b = bank_with(&[(1, RICH)]);
    let e = b
        .process(&tx(&[k(1)], vec![transfer(k(1), k(2), 890_879)]))
        .unwrap_err();
    assert_eq!(e, Error::RentNotExempt(k(2)));
    assert!(b.account(&k(2)).is_none());
    b.process(&tx(&[k(1)], vec![transfer(k(1), k(2), 890_880)]))
        .unwrap();
    assert_eq!(b.lamports(&k(2)), 890_880);
}

#[test]
fn failed_transactions_pay_the_fee_and_change_nothing_else() {
    let mut b = bank_with(&[(1, RICH)]);
    let before = b.lamports(&k(1));
    let t = tx(
        &[k(1)],
        vec![transfer(k(1), k(2), 2_000_000), transfer(k(1), k(3), RICH)],
    );
    assert_eq!(b.process(&t), Err(Error::InsufficientFunds));
    assert_eq!(b.lamports(&k(1)), before - 5000);
    assert!(
        b.account(&k(2)).is_none(),
        "the first transfer was rolled back"
    );
    assert_eq!(b.fees_collected, 5000);
}

#[test]
fn fee_is_per_signature_and_payer_must_afford_it() {
    let mut b = bank_with(&[(1, 9_999), (2, RICH)]);
    let t = tx(&[k(1), k(2)], vec![transfer(k(2), k(3), 1_000_000)]);
    assert_eq!(b.process(&t), Err(Error::InsufficientFeePayer));
    assert_eq!(b.lamports(&k(1)), 9_999);
    assert_eq!(b.fees_collected, 0);
    let mut b = bank_with(&[(1, 10_000), (2, RICH)]);
    assert_eq!(b.process(&t).unwrap().fee, 10_000);
    assert_eq!(b.lamports(&k(1)), 0);
    assert!(b.account(&k(1)).is_none(), "an empty account disappears");
}

#[test]
fn the_fee_may_not_leave_the_payer_below_the_rent_minimum() {
    let mut b = bank_with(&[(1, 890_880 + 4_999)]);
    assert_eq!(
        b.process(&tx(&[k(1)], vec![])),
        Err(Error::InsufficientFeePayer)
    );
    let mut b = bank_with(&[(1, 890_880 + 5_000)]);
    assert!(b.process(&tx(&[k(1)], vec![])).is_ok());
    assert_eq!(b.lamports(&k(1)), 890_880);
}

#[test]
fn signature_rules() {
    let mut b = bank_with(&[(1, RICH), (2, RICH)]);
    assert_eq!(b.process(&tx(&[], vec![])), Err(Error::NoSigners));
    // k(2) is asked to sign but did not
    assert_eq!(
        b.process(&tx(&[k(1)], vec![transfer(k(2), k(3), 1_000_000)])),
        Err(Error::MissingSignature(k(2)))
    );
    assert_eq!(
        b.fees_collected, 0,
        "a malformed transaction is not charged"
    );
}

fn sized(n: usize) -> Transaction {
    let mut ix = transfer(k(1), k(2), 1_000_000);
    ix.data = vec![0; n];
    tx(&[k(1)], vec![ix])
}

#[test]
fn serialized_length_matches_the_hand_count() {
    // 1+64 signatures, 3 header, 1+96 keys (payer, recipient, system program),
    // 32 blockhash, 1 instruction count, then 1+1+2 (program, account list)
    // and the data with its length prefix.
    assert_eq!(sized(0).serialized_len(), 203);
    assert_eq!(sized(127).serialized_len(), 330);
    assert_eq!(sized(128).serialized_len(), 332); // the length prefix grows to two bytes
}

#[test]
fn transaction_size_limit_is_1232_bytes() {
    assert_eq!(sized(1028).serialized_len(), 1232);
    assert_eq!(sized(1029).serialized_len(), 1233);
    let mut b = bank_with(&[(1, RICH)]);
    assert_eq!(b.process(&sized(1029)), Err(Error::TxTooLarge(1233)));
    assert_eq!(b.fees_collected, 0);
    // at the limit the size check passes (the program then rejects the malformed data)
    assert_eq!(b.process(&sized(1028)), Err(Error::InvalidInstructionData));
}

#[test]
fn account_keys_are_deduplicated_signers_first() {
    let t = tx(
        &[k(9), k(1)],
        vec![transfer(k(1), k(2), 1), transfer(k(2), k(1), 1)],
    );
    assert_eq!(t.account_keys(), vec![k(9), k(1), k(2), SYS]);
}

// ---------- the ownership rules, tested with small custom programs ----------

const P: Pubkey = Pubkey([0x50; 32]);

fn run_prog<T: Fn(&mut Invoke<'_>, &[u8]) -> Result<(), Error> + 'static>(
    f: T,
    accounts: Vec<(u8, Account)>,
    metas: Vec<AccountMeta>,
) -> (Bank, Result<Receipt, Error>) {
    let mut b = Bank::new();
    b.set_account(k(1), Account::wallet(RICH));
    b.add_program(P, Box::new(F(f)));
    for (n, a) in accounts {
        b.set_account(k(n), a);
    }
    let t = tx(
        &[k(1)],
        vec![Instruction {
            program_id: P,
            accounts: metas,
            data: vec![],
        }],
    );
    let r = b.process(&t);
    (b, r)
}

fn owned(owner: Pubkey, lamports: u64, data: Vec<u8>) -> Account {
    Account {
        lamports,
        data,
        owner,
        executable: false,
    }
}

#[test]
fn only_the_owner_may_change_data() {
    let w = |m: u8| vec![AccountMeta::writable(k(m), false)];
    let write = |inv: &mut Invoke<'_>, _: &[u8]| {
        inv.account_mut(0)?.data[0] = 9;
        Ok(())
    };
    let (b, r) = run_prog(write, vec![(2, owned(P, 2_000_000, vec![0; 4]))], w(2));
    assert!(r.is_ok());
    assert_eq!(b.account(&k(2)).unwrap().data[0], 9);
    let (b, r) = run_prog(
        write,
        vec![(2, owned(k(0x77), 2_000_000, vec![0; 4]))],
        w(2),
    );
    assert_eq!(r, Err(Error::UnauthorizedDataChange(k(2))));
    assert_eq!(b.account(&k(2)).unwrap().data[0], 0);
    // a wallet (system-owned) is not writable by a user program either
    let (_, r) = run_prog(write, vec![(2, owned(SYS, 2_000_000, vec![0; 4]))], w(2));
    assert_eq!(r, Err(Error::UnauthorizedDataChange(k(2))));
}

#[test]
fn only_the_owner_may_debit_but_anyone_may_credit() {
    let metas = vec![
        AccountMeta::writable(k(2), false),
        AccountMeta::writable(k(3), false),
    ];
    let take = |inv: &mut Invoke<'_>, _: &[u8]| {
        inv.account_mut(0)?.lamports -= 1000;
        inv.account_mut(1)?.lamports += 1000;
        Ok(())
    };
    // the program owns account 2 (debit) and not account 3 (credit): fine
    let (b, r) = run_prog(
        take,
        vec![
            (2, owned(P, 5_000_000, vec![])),
            (3, owned(k(0x77), 5_000_000, vec![])),
        ],
        metas.clone(),
    );
    assert!(r.is_ok());
    assert_eq!(b.lamports(&k(3)), 5_001_000);
    // the program does not own account 2: refused
    let (b, r) = run_prog(
        take,
        vec![
            (2, owned(k(0x77), 5_000_000, vec![])),
            (3, owned(P, 5_000_000, vec![])),
        ],
        metas.clone(),
    );
    assert_eq!(r, Err(Error::UnauthorizedDebit(k(2))));
    assert_eq!(b.lamports(&k(2)), 5_000_000);
    // the system program's wallets cannot be debited by a user program
    let (_, r) = run_prog(
        take,
        vec![
            (2, Account::wallet(5_000_000)),
            (3, Account::wallet(5_000_000)),
        ],
        metas,
    );
    assert_eq!(r, Err(Error::UnauthorizedDebit(k(2))));
}

#[test]
fn readonly_accounts_cannot_change_even_for_their_owner() {
    let metas = vec![AccountMeta::readonly(k(2), false)];
    let (_, r) = run_prog(
        |i, _| {
            i.account_mut(0)?.data[0] = 1;
            Ok(())
        },
        vec![(2, owned(P, 2_000_000, vec![0; 2]))],
        metas.clone(),
    );
    assert_eq!(r, Err(Error::ReadonlyModified(k(2))));
    let (_, r) = run_prog(
        |i, _| {
            i.account_mut(0)?.lamports += 1;
            Ok(())
        },
        vec![(2, owned(P, 2_000_000, vec![]))],
        metas.clone(),
    );
    assert_eq!(r, Err(Error::ReadonlyModified(k(2))));
    let (_, r) = run_prog(
        |i, _| {
            i.account_mut(0)?.owner = k(3);
            Ok(())
        },
        vec![(2, owned(P, 2_000_000, vec![]))],
        metas,
    );
    assert_eq!(r, Err(Error::ReadonlyModified(k(2))));
}

#[test]
fn lamports_are_conserved_across_an_instruction() {
    let metas = vec![
        AccountMeta::writable(k(2), false),
        AccountMeta::writable(k(3), false),
    ];
    let (_, r) = run_prog(
        |i, _| {
            i.account_mut(0)?.lamports -= 10;
            i.account_mut(1)?.lamports += 11;
            Ok(())
        },
        vec![
            (2, owned(P, 5_000_000, vec![])),
            (3, owned(P, 5_000_000, vec![])),
        ],
        metas.clone(),
    );
    assert_eq!(r, Err(Error::LamportsNotConserved));
    let (_, r) = run_prog(
        |i, _| {
            i.account_mut(0)?.lamports += 5;
            Ok(())
        },
        vec![
            (2, owned(P, 5_000_000, vec![])),
            (3, owned(P, 5_000_000, vec![])),
        ],
        metas,
    );
    assert_eq!(r, Err(Error::LamportsNotConserved));
}

#[test]
fn owner_changes_need_ownership_and_zeroed_data() {
    let metas = vec![AccountMeta::writable(k(2), false)];
    let give = |i: &mut Invoke<'_>, _: &[u8]| {
        i.account_mut(0)?.owner = k(0x66);
        Ok(())
    };
    let (b, r) = run_prog(
        give,
        vec![(2, owned(P, 2_000_000, vec![0; 8]))],
        metas.clone(),
    );
    assert!(r.is_ok());
    assert_eq!(b.account(&k(2)).unwrap().owner, k(0x66));
    let (_, r) = run_prog(
        give,
        vec![(2, owned(P, 2_000_000, vec![0, 0, 1]))],
        metas.clone(),
    );
    assert_eq!(r, Err(Error::UnauthorizedOwnerChange(k(2))));
    let (_, r) = run_prog(
        give,
        vec![(2, owned(k(0x77), 2_000_000, vec![0; 2]))],
        metas,
    );
    assert_eq!(r, Err(Error::UnauthorizedOwnerChange(k(2))));
}

#[test]
fn programs_cannot_be_edited_or_made_executable() {
    let metas = vec![AccountMeta::writable(k(2), false)];
    let mut prog_acct = owned(P, 2_000_000, vec![1]);
    prog_acct.executable = true;
    let (_, r) = run_prog(
        |i, _| {
            i.account_mut(0)?.data[0] = 2;
            Ok(())
        },
        vec![(2, prog_acct)],
        metas.clone(),
    );
    assert_eq!(r, Err(Error::UnauthorizedDataChange(k(2))));
    let (_, r) = run_prog(
        |i, _| {
            i.account_mut(0)?.executable = true;
            Ok(())
        },
        vec![(2, owned(P, 2_000_000, vec![]))],
        metas.clone(),
    );
    assert_eq!(r, Err(Error::ExecutableChanged(k(2))));
    let mut ex = owned(P, 2_000_000, vec![]);
    ex.executable = true;
    let (_, r) = run_prog(
        |i, _| {
            i.account_mut(0)?.executable = false;
            Ok(())
        },
        vec![(2, ex)],
        metas,
    );
    assert_eq!(r, Err(Error::ExecutableChanged(k(2))));
}

#[test]
fn rent_is_checked_for_every_modified_account() {
    let metas = vec![AccountMeta::writable(k(2), false)];
    // growing the data without adding lamports leaves the account below its minimum
    let (_, r) = run_prog(
        |i, _| {
            i.account_mut(0)?.data = vec![0; 100];
            Ok(())
        },
        vec![(2, owned(P, 890_880, vec![]))],
        metas.clone(),
    );
    assert_eq!(r, Err(Error::RentNotExempt(k(2))));
    // an untouched account that is already below the minimum is not an error
    let (_, r) = run_prog(|_, _| Ok(()), vec![(2, owned(P, 100, vec![]))], metas);
    assert!(r.is_ok());
}

#[test]
fn an_account_emptied_to_zero_lamports_is_deleted() {
    let metas = vec![
        AccountMeta::writable(k(2), false),
        AccountMeta::writable(k(3), false),
    ];
    let (b, r) = run_prog(
        |i, _| {
            let l = i.account(0)?.lamports;
            i.account_mut(0)?.lamports = 0;
            i.account_mut(1)?.lamports += l;
            Ok(())
        },
        vec![
            (2, owned(P, 3_000_000, vec![7; 5])),
            (3, Account::wallet(1_000_000)),
        ],
        metas,
    );
    assert!(r.is_ok());
    assert!(b.account(&k(2)).is_none());
    assert_eq!(b.lamports(&k(3)), 4_000_000);
}

#[test]
fn duplicate_accounts_alias_the_same_account() {
    let metas = vec![
        AccountMeta::readonly(k(2), false),
        AccountMeta::writable(k(2), false),
    ];
    let (b, r) = run_prog(
        |i, _| {
            i.account_mut(1)?.data[0] = 5;
            Ok(())
        },
        vec![(2, owned(P, 2_000_000, vec![0; 2]))],
        metas,
    );
    assert!(r.is_ok(), "writable if any meta for the key is writable");
    assert_eq!(b.account(&k(2)).unwrap().data[0], 5);
}

#[test]
fn account_indexes_are_bounds_checked() {
    let (_, r) = run_prog(
        |i, _| {
            i.key(3)?;
            Ok(())
        },
        vec![],
        vec![AccountMeta::readonly(k(2), false)],
    );
    assert_eq!(r, Err(Error::MissingAccount(3)));
}

#[test]
fn programs_must_exist_and_be_executable() {
    let mut b = bank_with(&[(1, RICH)]);
    let ix = Instruction {
        program_id: k(0x42),
        accounts: vec![],
        data: vec![],
    };
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix])),
        Err(Error::UnknownProgram(k(0x42)))
    );
    // a registered program whose account lost its executable flag
    b.add_program(P, Box::new(F(|_, _| Ok(()))));
    let mut a = b.account(&P).unwrap().clone();
    a.executable = false;
    b.set_account(P, a);
    let ix = Instruction {
        program_id: P,
        accounts: vec![],
        data: vec![],
    };
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix])),
        Err(Error::NotExecutable(P))
    );
}

#[test]
fn compute_budget_scales_with_instructions_and_is_capped() {
    let burn = |inv: &mut Invoke<'_>, d: &[u8]| inv.consume(u64::from(d[0]) * 1000);
    let mut b = Bank::new();
    b.set_account(k(1), Account::wallet(RICH));
    b.add_program(P, Box::new(F(burn)));
    let ix = |n: u8| Instruction {
        program_id: P,
        accounts: vec![],
        data: vec![n],
    };
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix(200)])).unwrap().units_used,
        200_000
    );
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix(201)])),
        Err(Error::ComputeExceeded)
    );
    // two instructions share 400,000 units
    assert!(b.process(&tx(&[k(1)], vec![ix(150), ix(250)])).is_ok());
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix(150), ix(251)])),
        Err(Error::ComputeExceeded)
    );
    // the cap: ten instructions would be 2,000,000, but only 1,400,000 are granted
    let ten: Vec<_> = (0..10).map(|_| ix(100)).collect();
    assert_eq!(b.process(&tx(&[k(1)], ten)).unwrap().units_used, 1_000_000);
    let mut many: Vec<_> = (0..7).map(|_| ix(200)).collect();
    assert_eq!(
        b.process(&tx(&[k(1)], many.clone())).unwrap().units_used,
        1_400_000
    );
    many.push(ix(1));
    assert_eq!(b.process(&tx(&[k(1)], many)), Err(Error::ComputeExceeded));
}

#[test]
fn a_failure_in_a_later_instruction_undoes_an_earlier_program_s_changes() {
    let mut b = Bank::new();
    b.set_account(k(1), Account::wallet(RICH));
    b.set_account(k(2), owned(P, 2_000_000, vec![0; 2]));
    b.add_program(
        P,
        Box::new(F(|i, d| {
            if d == [1] {
                return Err(Error::InvalidInstructionData);
            }
            i.account_mut(0)?.data[0] = 7;
            Ok(())
        })),
    );
    let ix = |d: u8| Instruction {
        program_id: P,
        accounts: vec![AccountMeta::writable(k(2), false)],
        data: vec![d],
    };
    assert!(b.process(&tx(&[k(1)], vec![ix(0)])).is_ok());
    assert_eq!(b.account(&k(2)).unwrap().data[0], 7);
    b.set_account(k(2), owned(P, 2_000_000, vec![0; 2]));
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix(0), ix(1)])),
        Err(Error::InvalidInstructionData)
    );
    assert_eq!(b.account(&k(2)).unwrap().data[0], 0);
}

// ---------- CPI ----------

fn cpi_to_system_transfer(
    inv: &mut Invoke<'_>,
    from: usize,
    to: usize,
    n: u64,
    seeds: &[&[&[u8]]],
) -> Result<(), Error> {
    let ix = Instruction {
        program_id: SYS,
        accounts: vec![
            AccountMeta::writable(inv.key(from)?, true),
            AccountMeta::writable(inv.key(to)?, false),
        ],
        data: transfer_data(n),
    };
    inv.invoke(&ix, seeds)
}

#[test]
fn a_cpi_passes_on_the_users_signature() {
    let mut b = bank_with(&[(1, RICH)]);
    b.add_program(
        P,
        Box::new(F(|i, _| cpi_to_system_transfer(i, 0, 1, 1_000_000, &[]))),
    );
    let ix = Instruction {
        program_id: P,
        accounts: vec![
            AccountMeta::writable(k(1), true),
            AccountMeta::writable(k(2), false),
            AccountMeta::readonly(SYS, false),
        ],
        data: vec![],
    };
    let r = b.process(&tx(&[k(1)], vec![ix])).unwrap();
    assert_eq!(b.lamports(&k(2)), 1_000_000);
    assert_eq!(r.units_used, 1000 + 150);
}

#[test]
fn a_cpi_cannot_invent_a_signature() {
    let mut b = bank_with(&[(1, RICH), (2, RICH)]);
    // the transaction did not sign for account 2, yet the program asks the system program to spend from it
    b.add_program(
        P,
        Box::new(F(|i, _| cpi_to_system_transfer(i, 1, 0, 1_000_000, &[]))),
    );
    let ix = Instruction {
        program_id: P,
        accounts: vec![
            AccountMeta::writable(k(1), true),
            AccountMeta::writable(k(2), false),
        ],
        data: vec![],
    };
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix])),
        Err(Error::PrivilegeEscalation(k(2)))
    );
    assert_eq!(b.lamports(&k(2)), RICH);
}

#[test]
fn a_cpi_cannot_gain_write_access() {
    let mut b = bank_with(&[(1, RICH), (2, RICH)]);
    b.add_program(
        P,
        Box::new(F(|i, _| cpi_to_system_transfer(i, 0, 1, 1_000_000, &[]))),
    );
    // account 1 is read-only for the caller
    let ix = Instruction {
        program_id: P,
        accounts: vec![
            AccountMeta::readonly(k(2), true),
            AccountMeta::readonly(k(1), false),
        ],
        data: vec![],
    };
    assert_eq!(
        b.process(&tx(&[k(1), k(2)], vec![ix])),
        Err(Error::PrivilegeEscalation(k(2)))
    );
}

#[test]
fn a_cpi_can_only_use_accounts_it_was_given() {
    let mut b = bank_with(&[(1, RICH)]);
    b.add_program(
        P,
        Box::new(F(|i, _| {
            let ix = Instruction {
                program_id: SYS,
                accounts: vec![
                    AccountMeta::writable(k(1), true),
                    AccountMeta::writable(k(9), false),
                ],
                data: transfer_data(1),
            };
            i.invoke(&ix, &[])
        })),
    );
    let ix = Instruction {
        program_id: P,
        accounts: vec![AccountMeta::writable(k(1), true)],
        data: vec![],
    };
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix])),
        Err(Error::MissingAccount(1))
    );
}

#[test]
fn a_pda_can_sign_only_for_its_own_program() {
    let seeds_for = |prog: Pubkey| {
        let (pda, bump) = solanalab::pubkey::find_program_address(&[b"vault"], &prog).unwrap();
        (pda, bump)
    };
    let (pda, bump) = seeds_for(P);
    let mut b = bank_with(&[(1, RICH)]);
    b.set_account(pda, Account::wallet(5_000_000));
    b.add_program(
        P,
        Box::new(F(move |i, _| {
            let bump_seed = [bump];
            let seeds: &[&[u8]] = &[b"vault", &bump_seed];
            cpi_to_system_transfer(i, 1, 0, 1_000_000, &[seeds])
        })),
    );
    let ix = Instruction {
        program_id: P,
        accounts: vec![
            AccountMeta::writable(k(1), true),
            AccountMeta::writable(pda, false),
            AccountMeta::readonly(SYS, false),
        ],
        data: vec![],
    };
    // the transfer is *from* the PDA (account 1 in the callee's order): signed by seeds
    assert!(b.process(&tx(&[k(1)], vec![ix.clone()])).is_ok());
    assert_eq!(b.lamports(&pda), 4_000_000);
    // the same seeds under a different program id derive a different address
    let other = Pubkey::filled(0x51);
    let (other_pda, other_bump) = seeds_for(other);
    assert_ne!(pda, other_pda);
    b.set_account(other_pda, Account::wallet(5_000_000));
    // P signs with the seeds of *its* vault but asks to spend from the other program's PDA: refused
    b.add_program(
        P,
        Box::new(F(move |i, _| {
            let bump_seed = [bump];
            let seeds: &[&[u8]] = &[b"vault", &bump_seed];
            cpi_to_system_transfer(i, 1, 0, 1_000_000, &[seeds])
        })),
    );
    let ix2 = Instruction {
        program_id: P,
        accounts: vec![
            AccountMeta::writable(k(1), true),
            AccountMeta::writable(other_pda, false),
            AccountMeta::readonly(SYS, false),
        ],
        data: vec![],
    };
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix2.clone()])),
        Err(Error::PrivilegeEscalation(other_pda))
    );
    // and P cannot even use the other program's bump: under P those seeds are not an address
    // (here the hash lands on the curve), so the derivation itself fails
    b.add_program(
        P,
        Box::new(F(move |i, _| {
            let bump_seed = [other_bump];
            let seeds: &[&[u8]] = &[b"vault", &bump_seed];
            cpi_to_system_transfer(i, 1, 0, 1_000_000, &[seeds])
        })),
    );
    assert_eq!(b.process(&tx(&[k(1)], vec![ix2])), Err(Error::InvalidSeeds));
    assert_eq!(b.lamports(&other_pda), 5_000_000);
}

#[test]
fn seeds_that_derive_an_on_curve_address_are_refused() {
    let mut b = bank_with(&[(1, RICH)]);
    b.add_program(
        P,
        Box::new(F(|i, _| {
            let ix = Instruction {
                program_id: SYS,
                accounts: vec![AccountMeta::writable(i.key(0)?, true)],
                data: assign_data(&Pubkey::default()),
            };
            let bump_seed = [255u8];
            let long = [0u8; 33];
            let seeds: &[&[u8]] = &[&long, &bump_seed];
            i.invoke(&ix, &[seeds])
        })),
    );
    let ix = Instruction {
        program_id: P,
        accounts: vec![AccountMeta::writable(k(1), true)],
        data: vec![],
    };
    assert_eq!(b.process(&tx(&[k(1)], vec![ix])), Err(Error::InvalidSeeds));
}

#[test]
fn recursion_depth_is_limited() {
    // a program that calls itself `data[0]` more times
    fn rec(inv: &mut Invoke<'_>, d: &[u8]) -> Result<(), Error> {
        if d[0] == 0 {
            return Ok(());
        }
        let ix = Instruction {
            program_id: inv.program_id(),
            accounts: vec![],
            data: vec![d[0] - 1],
        };
        inv.invoke(&ix, &[])
    }
    let mut b = bank_with(&[(1, RICH)]);
    b.add_program(P, Box::new(F(rec)));
    let ix = |n: u8| Instruction {
        program_id: P,
        accounts: vec![],
        data: vec![n],
    };
    assert!(
        b.process(&tx(&[k(1)], vec![ix(4)])).is_ok(),
        "top level plus four nested calls"
    );
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix(5)])),
        Err(Error::DepthExceeded)
    );
}

#[test]
fn reentrancy_through_another_program_is_refused() {
    let q = Pubkey::filled(0x52);
    let mut b = bank_with(&[(1, RICH)]);
    // P calls Q, and Q calls P back
    b.add_program(
        P,
        Box::new(F(move |inv, d| {
            if d == [1] {
                return Ok(());
            }
            inv.invoke(
                &Instruction {
                    program_id: q,
                    accounts: vec![],
                    data: vec![],
                },
                &[],
            )
        })),
    );
    b.add_program(
        q,
        Box::new(F(|inv, _| {
            inv.invoke(
                &Instruction {
                    program_id: P,
                    accounts: vec![],
                    data: vec![1],
                },
                &[],
            )
        })),
    );
    let ix = Instruction {
        program_id: P,
        accounts: vec![],
        data: vec![0],
    };
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix])),
        Err(Error::ReentrancyNotAllowed)
    );
}

#[test]
fn cpi_costs_compute_units() {
    let mut b = bank_with(&[(1, RICH)]);
    b.add_program(
        P,
        Box::new(F(|i, _| {
            i.consume(199_500)?;
            cpi_to_system_transfer(i, 0, 1, 1_000_000, &[])
        })),
    );
    let ix = Instruction {
        program_id: P,
        accounts: vec![
            AccountMeta::writable(k(1), true),
            AccountMeta::writable(k(2), false),
            AccountMeta::readonly(SYS, false),
        ],
        data: vec![],
    };
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix])),
        Err(Error::ComputeExceeded)
    );
}

#[test]
fn verify_changes_is_directly_callable() {
    use std::collections::BTreeMap;
    let before = BTreeMap::from([(k(2), owned(P, 10, vec![1]))]);
    let mut after = before.clone();
    let w = BTreeMap::from([(k(2), true)]);
    assert_eq!(verify_changes(&P, &w, &before, &after), Ok(()));
    after.get_mut(&k(2)).unwrap().data = vec![2];
    assert_eq!(verify_changes(&P, &w, &before, &after), Ok(()));
    assert_eq!(
        verify_changes(&k(3), &w, &before, &after),
        Err(Error::UnauthorizedDataChange(k(2)))
    );
}
```

### `tests/counter_tests.rs`

```rust
use solanalab::account::*;
use solanalab::counter::*;
use solanalab::instruction::*;
use solanalab::pubkey::*;
use solanalab::runtime::*;
use solanalab::system::*;

const SYS: Pubkey = Pubkey([0; 32]);
const CP: Pubkey = Pubkey([0xC0; 32]);
const RICH: u64 = 1_000_000_000;

fn k(n: u8) -> Pubkey {
    Pubkey::filled(n)
}

fn bank() -> Bank {
    let mut b = Bank::new();
    b.add_program(CP, Box::new(CounterProgram));
    b.set_account(k(1), Account::wallet(RICH));
    b.set_account(k(2), Account::wallet(RICH));
    b
}

fn tx(signers: &[Pubkey], ixs: Vec<Instruction>) -> Transaction {
    Transaction {
        signers: signers.to_vec(),
        instructions: ixs,
    }
}

fn created() -> (Bank, Pubkey) {
    let mut b = bank();
    let (pda, bump) = counter_address(&CP, &k(1));
    b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), pda, bump)]))
        .unwrap();
    (b, pda)
}

#[test]
fn create_makes_a_pda_account_owned_by_the_program() {
    let mut b = bank();
    let supply = b.total_supply();
    let (pda, bump) = counter_address(&CP, &k(1));
    // bumps 255 and 254 give points on the curve for these inputs (checked by an independent Python
    // implementation), so the canonical bump is 253
    assert_eq!(bump, 253);
    let r = b
        .process(&tx(&[k(1)], vec![create_ix(CP, k(1), pda, bump)]))
        .unwrap();
    let a = b.account(&pda).unwrap();
    assert_eq!(a.owner, CP);
    assert_eq!(a.lamports, 1_169_280);
    assert_eq!(a.data.len(), 40);
    assert_eq!(&a.data[..32], &k(1).0);
    assert_eq!(count_of(&a.data), Some(0));
    assert_eq!(b.lamports(&k(1)), RICH - 1_169_280 - 5000);
    assert_eq!(
        b.total_supply(),
        supply,
        "lamports are conserved (the fee is counted)"
    );
    // 100 (counter) + 1500 (bump search) + 1000 (CPI) + 150 (system)
    assert_eq!(r.units_used, 100 + 1500 + 1000 + 150);
}

#[test]
fn the_pda_cannot_be_created_with_a_wrong_address_or_bump() {
    let (pda, bump) = counter_address(&CP, &k(1));
    let (other, _) = counter_address(&CP, &k(2));
    let mut b = bank();
    // someone else's address
    assert_eq!(
        b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), other, bump)])),
        Err(Error::InvalidSeeds)
    );
    // a non-canonical bump (254 is a valid seed only if its address is off the curve; either way it is refused)
    assert_eq!(
        b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), pda, bump - 1)])),
        Err(Error::InvalidSeeds)
    );
    // a second, perfectly valid off-curve address for the same authority (bump 252): refused, because
    // accepting it would let one authority own several counters
    let second = create_program_address(&[b"counter", &k(1).0, &[252]], &CP).unwrap();
    assert_ne!(second, pda);
    assert_eq!(
        b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), second, 252)])),
        Err(Error::InvalidSeeds)
    );
    // an ordinary wallet address
    assert_eq!(
        b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), k(2), bump)])),
        Err(Error::InvalidSeeds)
    );
    assert!(b.account(&pda).is_none());
}

#[test]
fn create_needs_the_payer_signature_and_the_real_system_program() {
    let (pda, bump) = counter_address(&CP, &k(1));
    let mut b = bank();
    let mut ix = create_ix(CP, k(1), pda, bump);
    ix.accounts[0].is_signer = false;
    assert_eq!(b.process(&tx(&[k(2)], vec![ix])), Err(Error::NotSigner(0)));
    let mut ix = create_ix(CP, k(1), pda, bump);
    ix.accounts[2].pubkey = k(3);
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix])),
        Err(Error::InvalidInstructionData)
    );
}

#[test]
fn create_twice_is_refused() {
    let (mut b, pda) = created();
    let (_, bump) = counter_address(&CP, &k(1));
    assert_eq!(
        b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), pda, bump)])),
        Err(Error::AlreadyInUse)
    );
}

#[test]
fn create_with_a_prefunded_address_tops_up_only_the_difference() {
    let mut b = bank();
    let (pda, bump) = counter_address(&CP, &k(1));
    // anybody may send lamports to any address, including a PDA
    b.process(&tx(
        &[k(2)],
        vec![Instruction {
            program_id: SYS,
            accounts: vec![
                AccountMeta::writable(k(2), true),
                AccountMeta::writable(pda, false),
            ],
            data: transfer_data(1_000_000),
        }],
    ))
    .unwrap();
    b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), pda, bump)]))
        .unwrap();
    assert_eq!(b.lamports(&pda), 1_169_280);
    assert_eq!(b.lamports(&k(1)), RICH - (1_169_280 - 1_000_000) - 5000);
}

#[test]
fn increment_counts() {
    let (mut b, pda) = created();
    for n in 1..=3u64 {
        b.process(&tx(&[k(1)], vec![increment_ix(CP, pda, k(1))]))
            .unwrap();
        assert_eq!(count_of(&b.account(&pda).unwrap().data), Some(n));
    }
    // several increments in one transaction
    b.process(&tx(&[k(1)], vec![increment_ix(CP, pda, k(1)); 3]))
        .unwrap();
    assert_eq!(count_of(&b.account(&pda).unwrap().data), Some(6));
}

#[test]
fn increment_checks_signer_authority_owner_and_size() {
    let (mut b, pda) = created();
    // not signed by the authority
    let mut ix = increment_ix(CP, pda, k(1));
    ix.accounts[1].is_signer = false;
    assert_eq!(b.process(&tx(&[k(2)], vec![ix])), Err(Error::NotSigner(1)));
    // signed by someone who is not the stored authority
    assert_eq!(
        b.process(&tx(&[k(2)], vec![increment_ix(CP, pda, k(2))])),
        Err(Error::WrongAuthority)
    );
    // read-only counter
    let mut ix = increment_ix(CP, pda, k(1));
    ix.accounts[0].is_writable = false;
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix])),
        Err(Error::NotWritable(0))
    );
    // a counter-shaped account that the program does not own
    let mut fake = vec![0u8; 40];
    fake[..32].copy_from_slice(&k(2).0);
    b.set_account(
        k(8),
        Account {
            lamports: 2_000_000,
            data: fake.clone(),
            owner: k(0x99),
            executable: false,
        },
    );
    assert_eq!(
        b.process(&tx(&[k(2)], vec![increment_ix(CP, k(8), k(2))])),
        Err(Error::WrongOwner(0))
    );
    // right owner, wrong size
    b.set_account(
        k(9),
        Account {
            lamports: 2_000_000,
            data: vec![0; 39],
            owner: CP,
            executable: false,
        },
    );
    assert_eq!(
        b.process(&tx(&[k(2)], vec![increment_ix(CP, k(9), k(2))])),
        Err(Error::InvalidAccountData)
    );
    assert_eq!(count_of(&b.account(&pda).unwrap().data), Some(0));
}

#[test]
fn an_attacker_cannot_hand_the_program_an_account_they_control() {
    let (mut b, _) = created();
    // an account the attacker owns outright holds counter-looking data, but its owner is the system program,
    // so the program refuses it before reading the data
    let mut data = vec![0u8; 40];
    data[..32].copy_from_slice(&k(2).0);
    b.set_account(
        k(8),
        Account {
            lamports: 2_000_000,
            data,
            owner: SYS,
            executable: false,
        },
    );
    assert_eq!(
        b.process(&tx(&[k(2)], vec![increment_ix(CP, k(8), k(2))])),
        Err(Error::WrongOwner(0))
    );
    // and they cannot make the program own it: Assign moves ownership only for a system-owned account with
    // their signature, and a program-owned account cannot be written by anyone else
    b.set_account(k(7), Account::wallet(2_000_000));
    b.process(&tx(
        &[k(2), k(7)],
        vec![Instruction {
            program_id: SYS,
            accounts: vec![AccountMeta::writable(k(7), true)],
            data: assign_data(&CP),
        }],
    ))
    .unwrap();
    // the account is now owned by the counter program, but its data is all zeros: authority 0000…
    assert_eq!(
        b.process(&tx(&[k(2)], vec![increment_ix(CP, k(7), k(2))])),
        Err(Error::InvalidAccountData)
    );
}

#[test]
fn close_returns_the_lamports_and_deletes_the_account() {
    let (mut b, pda) = created();
    let supply = b.total_supply();
    let before = b.lamports(&k(1));
    b.process(&tx(&[k(1)], vec![close_ix(CP, pda, k(1))]))
        .unwrap();
    assert!(b.account(&pda).is_none());
    assert_eq!(b.lamports(&k(1)), before + 1_169_280 - 5000);
    assert_eq!(b.total_supply(), supply);
    // and the address can be used again
    let (_, bump) = counter_address(&CP, &k(1));
    b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), pda, bump)]))
        .unwrap();
    assert_eq!(count_of(&b.account(&pda).unwrap().data), Some(0));
}

#[test]
fn close_needs_the_authority() {
    let (mut b, pda) = created();
    assert_eq!(
        b.process(&tx(&[k(2)], vec![close_ix(CP, pda, k(2))])),
        Err(Error::WrongAuthority)
    );
    let mut ix = close_ix(CP, pda, k(1));
    ix.accounts[1].is_writable = false;
    assert_eq!(
        b.process(&tx(&[k(1)], vec![ix])),
        Err(Error::NotWritable(1))
    );
    assert!(b.account(&pda).is_some());
}

#[test]
fn unknown_instruction_data_is_refused() {
    let (mut b, pda) = created();
    for d in [
        vec![],
        vec![9],
        vec![1, 0],
        vec![0],
        vec![0, 1, 2],
        vec![2, 2],
    ] {
        let mut ix = increment_ix(CP, pda, k(1));
        ix.data = d.clone();
        assert_eq!(
            b.process(&tx(&[k(1)], vec![ix])),
            Err(Error::InvalidInstructionData),
            "{d:?}"
        );
    }
}

#[test]
fn a_counter_is_per_authority() {
    let mut b = bank();
    let (p1, b1) = counter_address(&CP, &k(1));
    let (p2, b2) = counter_address(&CP, &k(2));
    assert_ne!(p1, p2);
    b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), p1, b1)]))
        .unwrap();
    b.process(&tx(&[k(2)], vec![create_ix(CP, k(2), p2, b2)]))
        .unwrap();
    b.process(&tx(&[k(2)], vec![increment_ix(CP, p2, k(2))]))
        .unwrap();
    assert_eq!(count_of(&b.account(&p1).unwrap().data), Some(0));
    assert_eq!(count_of(&b.account(&p2).unwrap().data), Some(1));
}

#[test]
fn counter_overflow_is_an_error_not_a_wrap() {
    let (mut b, pda) = created();
    let mut a = b.account(&pda).unwrap().clone();
    a.data[32..40].copy_from_slice(&u64::MAX.to_le_bytes());
    b.set_account(pda, a);
    assert_eq!(
        b.process(&tx(&[k(1)], vec![increment_ix(CP, pda, k(1))])),
        Err(Error::ArithmeticOverflow)
    );
}

#[test]
fn the_pipeline_in_one_transaction() {
    // create, increment twice, in one atomic transaction
    let mut b = bank();
    let (pda, bump) = counter_address(&CP, &k(1));
    b.process(&tx(
        &[k(1)],
        vec![
            create_ix(CP, k(1), pda, bump),
            increment_ix(CP, pda, k(1)),
            increment_ix(CP, pda, k(1)),
        ],
    ))
    .unwrap();
    assert_eq!(count_of(&b.account(&pda).unwrap().data), Some(2));
    // if the last instruction fails, the creation is undone as well
    let mut b = bank();
    let t = tx(
        &[k(1)],
        vec![create_ix(CP, k(1), pda, bump), increment_ix(CP, pda, k(2))],
    );
    assert_eq!(b.process(&t), Err(Error::MissingSignature(k(2))));
    let mut t = tx(
        &[k(1), k(2)],
        vec![create_ix(CP, k(1), pda, bump), increment_ix(CP, pda, k(2))],
    );
    assert_eq!(b.process(&t), Err(Error::WrongAuthority));
    assert!(b.account(&pda).is_none());
    t.signers.pop();
}
```

### `tests/gap_tests.rs`

```rust
//! Tests added after the first mutation run: each closes a gap it exposed.
use std::collections::BTreeMap;

use solanalab::account::*;
use solanalab::counter::*;
use solanalab::instruction::*;
use solanalab::pubkey::*;
use solanalab::runtime::*;
use solanalab::system::*;

const SYS: Pubkey = Pubkey([0; 32]);
const CP: Pubkey = Pubkey([0xC0; 32]);
const BIG: u64 = 1_000_000_000_000;

fn k(n: u8) -> Pubkey {
    Pubkey::filled(n)
}

fn tx(signers: &[Pubkey], ixs: Vec<Instruction>) -> Transaction {
    Transaction {
        signers: signers.to_vec(),
        instructions: ixs,
    }
}

fn sys_ix(metas: Vec<AccountMeta>, data: Vec<u8>) -> Instruction {
    Instruction {
        program_id: SYS,
        accounts: metas,
        data,
    }
}

fn signer(n: u8) -> AccountMeta {
    AccountMeta::writable(k(n), true)
}

fn plain(n: u8) -> AccountMeta {
    AccountMeta::writable(k(n), false)
}

/// Bank with a rich fee payer (key 1) and a rich wallet (key 2).
fn bank() -> Bank {
    let mut b = Bank::new();
    b.set_account(k(1), Account::wallet(BIG));
    b.set_account(k(2), Account::wallet(BIG));
    b
}

fn run(b: &mut Bank, signers: &[u8], ix: Instruction) -> Result<Receipt, Error> {
    let s: Vec<Pubkey> = signers.iter().map(|&n| k(n)).collect();
    b.process(&tx(&s, vec![ix]))
}

// ---- system program ----

#[test]
fn create_account_accepts_max_space_and_refuses_one_more() {
    const TEN_MIB: u64 = 10 * 1024 * 1024;
    let mut b = bank();
    let ok = sys_ix(
        vec![signer(1), signer(3)],
        create_account_data(rent_exempt_minimum(TEN_MIB as usize), TEN_MIB, &k(9)),
    );
    run(&mut b, &[1, 3], ok).unwrap();
    assert_eq!(b.account(&k(3)).unwrap().data.len(), TEN_MIB as usize);
    let too_big = sys_ix(
        vec![signer(1), signer(4)],
        create_account_data(BIG / 2, TEN_MIB + 1, &k(9)),
    );
    assert_eq!(
        run(&mut b, &[1, 4], too_big),
        Err(Error::InvalidInstructionData)
    );
}

#[test]
fn create_account_needs_two_distinct_signers() {
    let mut b = bank();
    let same = sys_ix(
        vec![signer(1), signer(1)],
        create_account_data(1_000_000, 0, &SYS),
    );
    assert_eq!(run(&mut b, &[1], same), Err(Error::InvalidInstructionData));
    let unsigned_new = sys_ix(
        vec![signer(1), plain(3)],
        create_account_data(1_000_000, 0, &SYS),
    );
    assert_eq!(run(&mut b, &[1], unsigned_new), Err(Error::NotSigner(1)));
}

#[test]
fn create_account_refuses_an_account_in_use() {
    let mut b = bank();
    b.set_account(
        k(3),
        Account {
            lamports: 1_000_000,
            data: vec![1],
            owner: SYS,
            executable: false,
        },
    );
    b.set_account(
        k(4),
        Account {
            lamports: 1_000_000,
            data: vec![],
            owner: k(9),
            executable: false,
        },
    );
    for n in [3, 4] {
        let ix = sys_ix(
            vec![signer(1), signer(n)],
            create_account_data(1_000_000, 0, &SYS),
        );
        assert_eq!(
            run(&mut b, &[1, n], ix),
            Err(Error::AlreadyInUse),
            "key {n}"
        );
    }
}

#[test]
fn transfer_source_rules() {
    let mut b = bank();
    // owned by someone else
    b.set_account(
        k(3),
        Account {
            lamports: 5_000_000,
            data: vec![],
            owner: k(9),
            executable: false,
        },
    );
    let ix = sys_ix(vec![signer(3), plain(4)], transfer_data(1_000_000));
    assert_eq!(run(&mut b, &[1, 3], ix), Err(Error::WrongOwner(0)));
    // carrying data
    b.set_account(
        k(5),
        Account {
            lamports: 5_000_000,
            data: vec![1],
            owner: SYS,
            executable: false,
        },
    );
    let ix = sys_ix(vec![signer(5), plain(4)], transfer_data(1_000_000));
    assert_eq!(run(&mut b, &[1, 5], ix), Err(Error::InvalidAccountData));
}

#[test]
fn transfer_of_the_exact_balance_is_allowed_and_one_more_is_not() {
    let mut b = bank();
    b.set_account(k(3), Account::wallet(2_000_000));
    let more = sys_ix(vec![signer(3), plain(4)], transfer_data(2_000_001));
    assert_eq!(run(&mut b, &[1, 3], more), Err(Error::InsufficientFunds));
    let exact = sys_ix(vec![signer(3), plain(4)], transfer_data(2_000_000));
    run(&mut b, &[1, 3], exact).unwrap();
    assert!(b.account(&k(3)).is_none(), "an emptied account disappears");
    assert_eq!(b.lamports(&k(4)), 2_000_000);
}

#[test]
fn transfer_into_a_full_account_overflows_cleanly() {
    let mut b = bank();
    b.set_account(k(3), Account::wallet(u64::MAX - 5));
    let ix = sys_ix(vec![signer(2), plain(3)], transfer_data(1_000_000));
    assert_eq!(run(&mut b, &[1, 2], ix), Err(Error::ArithmeticOverflow));
}

#[test]
fn assign_changes_the_owner_once() {
    let mut b = bank();
    let ix = sys_ix(vec![signer(2)], assign_data(&k(9)));
    run(&mut b, &[1, 2], ix).unwrap();
    assert_eq!(b.account(&k(2)).unwrap().owner, k(9));
    // the system program no longer owns it, so it cannot reassign it
    let again = sys_ix(vec![signer(2)], assign_data(&k(8)));
    assert_eq!(run(&mut b, &[1, 2], again), Err(Error::WrongOwner(0)));
    // and without the signature nothing happens
    let unsigned = sys_ix(vec![plain(1)], assign_data(&k(8)));
    assert_eq!(run(&mut b, &[1], unsigned), Err(Error::NotSigner(0)));
}

#[test]
fn allocate_gives_an_empty_account_data() {
    let mut b = bank();
    run(&mut b, &[1, 2], sys_ix(vec![signer(2)], allocate_data(100))).unwrap();
    assert_eq!(b.account(&k(2)).unwrap().data, vec![0; 100]);
    // a second allocation finds the account in use
    assert_eq!(
        run(&mut b, &[1, 2], sys_ix(vec![signer(2)], allocate_data(50))),
        Err(Error::AlreadyInUse)
    );
    // too large
    assert_eq!(
        run(
            &mut b,
            &[1, 1],
            sys_ix(vec![signer(1)], allocate_data(10 * 1024 * 1024 + 1))
        ),
        Err(Error::AlreadyInUse)
    );
    // the maximum is fine
    run(
        &mut b,
        &[1],
        sys_ix(vec![signer(1)], allocate_data(10 * 1024 * 1024)),
    )
    .unwrap();
    // wrong length of data for the tag
    let mut d = allocate_data(1);
    d.push(0);
    assert_eq!(
        run(&mut b, &[1], sys_ix(vec![signer(1)], d)),
        Err(Error::InvalidInstructionData)
    );
}

#[test]
fn allocate_refuses_an_account_owned_by_another_program() {
    let mut b = bank();
    b.set_account(
        k(3),
        Account {
            lamports: BIG,
            data: vec![],
            owner: k(9),
            executable: false,
        },
    );
    assert_eq!(
        run(&mut b, &[1, 3], sys_ix(vec![signer(3)], allocate_data(8))),
        Err(Error::AlreadyInUse)
    );
}

// ---- runtime ----

struct Burn(u64);
impl Program for Burn {
    fn process(&self, inv: &mut Invoke<'_>, _data: &[u8]) -> Result<(), Error> {
        inv.consume(self.0)
    }
}

fn burn_bank(n: u64) -> Bank {
    let mut b = bank();
    b.add_program(k(0x70), Box::new(Burn(n)));
    b
}

fn burn_ix() -> Instruction {
    Instruction {
        program_id: k(0x70),
        accounts: vec![],
        data: vec![],
    }
}

#[test]
fn one_instruction_gets_exactly_200_000_units() {
    let r = run(&mut burn_bank(200_000), &[1], burn_ix()).unwrap();
    assert_eq!(r.units_used, 200_000);
    assert_eq!(
        run(&mut burn_bank(200_001), &[1], burn_ix()),
        Err(Error::ComputeExceeded)
    );
}

#[test]
fn a_transaction_never_gets_more_than_1_400_000_units() {
    let eight = |b: &mut Bank| b.process(&tx(&[k(1)], (0..8).map(|_| burn_ix()).collect()));
    // eight instructions would be entitled to 1_600_000, but the cap is 1_400_000;
    // the budget is shared, so four burns of 350_000 use exactly the cap
    struct Quarter;
    impl Program for Quarter {
        fn process(&self, inv: &mut Invoke<'_>, _d: &[u8]) -> Result<(), Error> {
            inv.consume(175_000)
        }
    }
    let mut b = bank();
    b.add_program(k(0x70), Box::new(Quarter));
    let r = eight(&mut b).unwrap();
    assert_eq!(r.units_used, 1_400_000);
    // a ninth instruction does not fit
    let mut b = bank();
    b.add_program(k(0x70), Box::new(Quarter));
    assert_eq!(
        b.process(&tx(&[k(1)], (0..9).map(|_| burn_ix()).collect())),
        Err(Error::ComputeExceeded)
    );
}

#[test]
fn duplicate_metas_are_writable_if_any_is() {
    struct Touch;
    impl Program for Touch {
        fn process(&self, inv: &mut Invoke<'_>, _d: &[u8]) -> Result<(), Error> {
            inv.account_mut(0)?.lamports += 1;
            inv.account_mut(1)?.lamports -= 1;
            Ok(())
        }
    }
    let mut b = bank();
    b.add_program(k(0x71), Box::new(Touch));
    // the same key twice: writable first, read-only second -- still writable.
    // The program does not own it, so a *credit and a debit of the same
    // account* net to zero and the change is legal.
    let ix = Instruction {
        program_id: k(0x71),
        accounts: vec![
            AccountMeta::writable(k(2), false),
            AccountMeta::readonly(k(2), false),
        ],
        data: vec![],
    };
    run(&mut b, &[1], ix).unwrap();
    // read-only first, writable second: same answer
    let ix = Instruction {
        program_id: k(0x71),
        accounts: vec![
            AccountMeta::readonly(k(2), false),
            AccountMeta::writable(k(2), false),
        ],
        data: vec![],
    };
    run(&mut b, &[1], ix).unwrap();
}

#[test]
fn a_program_cannot_hand_off_an_executable_account() {
    let p = k(0x50);
    let acct = |owner: Pubkey| Account {
        lamports: 1,
        data: vec![0; 4],
        owner,
        executable: true,
    };
    let mut before = BTreeMap::new();
    before.insert(k(1), acct(p));
    let mut after = before.clone();
    after.get_mut(&k(1)).unwrap().owner = k(9);
    let mut writable = BTreeMap::new();
    writable.insert(k(1), true);
    assert_eq!(
        verify_changes(&p, &writable, &before, &after),
        Err(Error::UnauthorizedOwnerChange(k(1)))
    );
    // the same change on a non-executable, zeroed account is allowed
    before.get_mut(&k(1)).unwrap().executable = false;
    after.get_mut(&k(1)).unwrap().executable = false;
    assert_eq!(verify_changes(&p, &writable, &before, &after), Ok(()));
}

#[test]
fn max_seeds_boundary_for_find_program_address() {
    let seed = [7u8];
    let sixteen: Vec<&[u8]> = (0..16).map(|_| &seed[..]).collect();
    let fifteen: Vec<&[u8]> = (0..15).map(|_| &seed[..]).collect();
    assert_eq!(
        find_program_address(&sixteen, &k(1)),
        Err(PdaError::TooManySeeds)
    );
    assert!(find_program_address(&fifteen, &k(1)).is_ok());
}

#[test]
fn shortvec_widens_at_16384_bytes_of_data() {
    let len = |n: usize| {
        tx(
            &[k(1)],
            vec![Instruction {
                program_id: k(2),
                accounts: vec![],
                data: vec![0; n],
            }],
        )
        .serialized_len()
    };
    // 168 fixed bytes (one signer, two keys, blockhash, one instruction with
    // no accounts) plus the shortvec length of the data plus the data itself
    assert_eq!(len(0), 168 + 1);
    assert_eq!(len(127), 168 + 1 + 127);
    assert_eq!(len(128), 168 + 2 + 128);
    assert_eq!(len(16_383), 168 + 2 + 16_383);
    assert_eq!(len(16_384), 168 + 3 + 16_384);
}

// ---- counter ----

fn counter_bank() -> Bank {
    let mut b = Bank::new();
    b.add_program(CP, Box::new(CounterProgram));
    b.set_account(k(1), Account::wallet(1_000_000_000));
    b
}

#[test]
fn create_refuses_an_address_owned_by_someone_else() {
    let mut b = counter_bank();
    let (pda, bump) = counter_address(&CP, &k(1));
    b.set_account(
        pda,
        Account {
            lamports: 5_000_000,
            data: vec![],
            owner: k(9),
            executable: false,
        },
    );
    assert_eq!(
        b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), pda, bump)])),
        Err(Error::AlreadyInUse)
    );
}

#[test]
fn close_then_create_in_one_transaction_works_only_if_close_resets_the_owner() {
    let mut b = counter_bank();
    let (pda, bump) = counter_address(&CP, &k(1));
    b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), pda, bump)]))
        .unwrap();
    b.process(&tx(
        &[k(1)],
        vec![close_ix(CP, pda, k(1)), create_ix(CP, k(1), pda, bump)],
    ))
    .unwrap();
    assert_eq!(count_of(&b.account(&pda).unwrap().data), Some(0));
}

#[test]
fn close_into_a_full_wallet_overflows_cleanly() {
    let mut b = counter_bank();
    let (pda, bump) = counter_address(&CP, &k(1));
    b.process(&tx(&[k(1)], vec![create_ix(CP, k(1), pda, bump)]))
        .unwrap();
    let mut a = b.account(&k(1)).unwrap().clone();
    a.lamports = u64::MAX - 1_000;
    b.set_account(k(1), a);
    assert_eq!(
        b.process(&tx(&[k(1)], vec![close_ix(CP, pda, k(1))])),
        Err(Error::ArithmeticOverflow)
    );
}

struct BurnData;
impl Program for BurnData {
    fn process(&self, inv: &mut Invoke<'_>, data: &[u8]) -> Result<(), Error> {
        inv.consume(u64::from_le_bytes(data.try_into().expect("8 bytes")))
    }
}

#[test]
fn the_cap_is_exactly_1_400_000_even_for_one_hungry_instruction() {
    let ixs = |first: u64| -> Vec<Instruction> {
        (0..8)
            .map(|i| Instruction {
                program_id: k(0x72),
                accounts: vec![],
                data: (if i == 0 { first } else { 0 }).to_le_bytes().to_vec(),
            })
            .collect()
    };
    let mut b = bank();
    b.add_program(k(0x72), Box::new(BurnData));
    assert_eq!(
        b.process(&tx(&[k(1)], ixs(1_400_001))),
        Err(Error::ComputeExceeded)
    );
    assert_eq!(
        b.process(&tx(&[k(1)], ixs(1_400_000))).unwrap().units_used,
        1_400_000
    );
}

struct Bump;
impl Program for Bump {
    fn process(&self, inv: &mut Invoke<'_>, _d: &[u8]) -> Result<(), Error> {
        inv.account_mut(0)?.data[0] += 1;
        Ok(())
    }
}

#[test]
fn a_key_listed_twice_is_writable_if_either_meta_says_so() {
    let owned = Account {
        lamports: 5_000_000,
        data: vec![0],
        owner: k(0x73),
        executable: false,
    };
    for metas in [
        vec![
            AccountMeta::writable(k(3), false),
            AccountMeta::readonly(k(3), false),
        ],
        vec![
            AccountMeta::readonly(k(3), false),
            AccountMeta::writable(k(3), false),
        ],
    ] {
        let mut b = bank();
        b.add_program(k(0x73), Box::new(Bump));
        b.set_account(k(3), owned.clone());
        let ix = Instruction {
            program_id: k(0x73),
            accounts: metas,
            data: vec![],
        };
        b.process(&tx(&[k(1)], vec![ix])).unwrap();
        assert_eq!(b.account(&k(3)).unwrap().data, vec![1]);
    }
}
```

### `tests/props.rs`

```rust
use solanalab::account::*;
use solanalab::instruction::*;
use solanalab::pubkey::Pubkey;
use solanalab::runtime::*;
use solanalab::system::*;

struct Rng(u64);
impl Rng {
    fn next(&mut self) -> u64 {
        self.0 ^= self.0 >> 12;
        self.0 ^= self.0 << 25;
        self.0 ^= self.0 >> 27;
        self.0.wrapping_mul(0x2545_F491_4F6C_DD1D)
    }
    fn below(&mut self, n: u64) -> u64 {
        self.next() % n
    }
}

fn k(n: u8) -> Pubkey {
    Pubkey::filled(n + 1)
}

#[test]
fn random_transactions_conserve_lamports_and_never_break_rent() {
    for seed in 1..=200u64 {
        let mut rng = Rng(seed * 0x9E37_79B9);
        let mut b = Bank::new();
        for n in 0..6u8 {
            b.set_account(k(n), Account::wallet(5_000_000 + rng.below(5_000_000)));
        }
        let supply = b.total_supply();
        let (mut ok, mut failed) = (0, 0);
        for _ in 0..60 {
            let mut ixs = Vec::new();
            for _ in 0..1 + rng.below(3) {
                let from = k(rng.below(6) as u8);
                let to = k(rng.below(8) as u8);
                let amount = if rng.below(4) == 0 {
                    rng.below(20_000_000)
                } else {
                    rng.below(2_000_000)
                };
                ixs.push(Instruction {
                    program_id: Pubkey::default(),
                    accounts: vec![
                        AccountMeta::writable(from, true),
                        AccountMeta::writable(to, false),
                    ],
                    data: transfer_data(amount),
                });
            }
            let signers: Vec<Pubkey> = {
                let mut s: Vec<Pubkey> = ixs.iter().map(|i| i.accounts[0].pubkey).collect();
                s.dedup();
                let mut seen = std::collections::BTreeSet::new();
                s.retain(|x| seen.insert(*x));
                s
            };
            match b.process(&Transaction {
                signers,
                instructions: ixs,
            }) {
                Ok(_) => ok += 1,
                Err(_) => failed += 1,
            }
            assert_eq!(b.total_supply(), supply, "seed {seed}");
        }
        assert!(
            ok > 5 && failed > 5,
            "seed {seed}: ok {ok}, failed {failed}"
        );
        // every surviving account is empty-of-lamports-free and rent-exempt
        for n in 0..8u8 {
            if let Some(a) = b.account(&k(n)) {
                assert!(
                    a.lamports >= rent_exempt_minimum(a.data.len()) || a.lamports == 0,
                    "seed {seed}"
                );
            }
        }
    }
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
//! Rough timings for the main operations (release build).
use solanalab::account::Account;
use solanalab::base58;
use solanalab::counter::*;
use solanalab::instruction::*;
use solanalab::pubkey::*;
use solanalab::runtime::*;
use solanalab::system::transfer_data;
use std::hint::black_box;
use std::time::Instant;

fn time<F: FnMut()>(name: &str, n: u32, mut f: F) {
    let t = Instant::now();
    for _ in 0..n {
        f();
    }
    let per = t.elapsed().as_nanos() as f64 / n as f64;
    println!("{name:<34} {per:>12.0} ns/op");
}

fn main() {
    let key = Pubkey::filled(7);
    let text = key.to_string();
    time("base58 encode (32 B)", 20000, || {
        black_box(base58::encode(black_box(&key.0)));
    });
    time("base58 decode (44 chars)", 20000, || {
        black_box(base58::decode(black_box(&text)).unwrap());
    });
    let pid = Pubkey::filled(1);
    time("is_on_curve", 20000, || {
        black_box(black_box(key).is_on_curve());
    });
    time("create_program_address", 20000, || {
        let _ = black_box(create_program_address(&[b"counter", &key.0, &[253]], &pid));
    });
    time("find_program_address (first try)", 5000, || {
        let _ = black_box(find_program_address(
            &[b"counter", &key.0],
            &Pubkey::filled(0xC0),
        ));
    });
    let mut hist = [0u32; 8];
    for i in 0..2000u32 {
        let (_, b) = find_program_address(&[b"x", &i.to_le_bytes()], &pid).unwrap();
        hist[(255 - b as usize).min(7)] += 1;
    }
    println!("bumps tried (1..8+) over 2000 seeds: {:?}", hist);

    let cp = Pubkey::filled(0xC0);
    let mut bank = Bank::new();
    bank.add_program(cp, Box::new(CounterProgram));
    bank.set_account(Pubkey::filled(1), Account::wallet(u64::MAX / 4));
    let a = Pubkey::filled(1);
    let b_ = Pubkey::filled(2);
    let transfer = Transaction {
        signers: vec![a],
        instructions: vec![Instruction {
            program_id: Pubkey::default(),
            accounts: vec![
                AccountMeta::writable(a, true),
                AccountMeta::writable(b_, false),
            ],
            data: transfer_data(1_000_000),
        }],
    };
    time("process: system transfer", 20000, || {
        bank.process(&transfer).unwrap();
    });
    let (pda, bump) = counter_address(&cp, &a);
    let create = Transaction {
        signers: vec![a],
        instructions: vec![create_ix(cp, a, pda, bump)],
    };
    let close = Transaction {
        signers: vec![a],
        instructions: vec![close_ix(cp, pda, a)],
    };
    time("process: create counter + close", 2000, || {
        bank.process(&create).unwrap();
        bank.process(&close).unwrap();
    });
    bank.process(&create).unwrap();
    let inc = Transaction {
        signers: vec![a],
        instructions: vec![increment_ix(cp, pda, a)],
    };
    time("process: increment", 20000, || {
        bank.process(&inc).unwrap();
    });
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **77 tests** (base58 and keys 10, runtime 32, counter 14, gap-closing tests 20, property test 1) and 8 consecutive full runs without a failure (nothing uses sockets, real time or OS randomness, so there is nothing to flake).

**Mutation check ✓.** I made 132 small source edits (a "mutant") across the seven source modules, each in its own copy of the crate, and ran the whole test suite against each. **129 were caught by a failing test, 2 are equivalent (M15, M118), and 1 does not compile (M129); none survive unexplained.** **What the first pass found.** Of the 132 mutants, 24 survived my first 57 tests and two did not compile (one of those, an attempt to keep every key, I rewrote so that it compiles; the other is kept in the table). Almost all pointed at the same place: **the system program's instructions were only exercised through `Transfer` and `CreateAccount` via the counter**, so `Assign` and `Allocate` had no direct test, nor did the rules that a source account must be system-owned with no data, that the new account must sign, or that the destination credit is checked; the exact compute constants (200,000 per instruction and the 1,400,000 cap) were only tested through scaled comparisons; the 16-seed and the 10 MiB limits were tested with the crate's own constants; the second shortvec width was never reached; a key listed twice with different privileges had no test; an **executable** account owned by the running program (the owner-change rule) had none; and the counter's close was never followed by a re-creation in the same transaction, nor closed into a wallet that would overflow. The cures were tests, not arguments: 20 tests in `tests/gap_tests.rs`. One mutant survived my first attempt to kill it because my new test read the *mutated constant* (`MAX_DATA_LEN`) from the crate, so the test and the mutant moved together; the second attempt uses the literal number 10 × 1024 × 1024. Another mutant (the duplicate-key writability merge) survived a test that credited and debited the same account, because the net change was zero; it needed a test in which the program changes a byte that only a writable account may change. Two mutants remain, each with a stated reason for being equivalent. The column "change" is generated from the edit itself (the part that differs, with a little context, whitespace collapsed); `…` marks omitted text. The tests ran in the dev profile.

| # | module | change | result |
|---|---|---|---|
| M1 | `base58` | `let zeros = bytes.iter().take_while(\|&&b\| b == 0).count()` → `let zeros = 0` | caught by `base58_decode_known_and_round_trip` |
| M2 | `base58` | `…u32::from(b)` → `…u32::from(b) + 1` | caught by `base58_decode_known_and_round_trip` |
| M3 | `base58` | `…u32::from(*d) << 8` → `…u32::from(*d) << 7` | caught by `base58_known_answers` |
| M4 | `base58` | `s.extend(std::iter::repeat_n('1', zeros));` → `∅` | caught by `base58_known_answers` |
| M5 | `base58` | `let zeros = s.bytes().take_while(\|&c\| c == b'1').count()` → `let zeros = 0` | caught by `base58_decode_known_and_round_trip` |
| M6 | `base58` | `…u32::from(*b) * 58` → `…u32::from(*b) * 57` | caught by `base58_decode_known_and_round_trip` |
| M7 | `base58` | `…carry = v as u32` → `…carry = v as u32 + 1` | caught by `base58_decode_known_and_round_trip` |
| M8 | `base58` | `…Base58Error::BadChar(i` → `…Base58Error::BadChar(i + 1` | caught by `base58_rejects_bad_characters` |
| M9 | `pubkey` | `…seeds.len() >` → `…seeds.len() >=` | caught by `seed_limits` |
| M10 | `pubkey` | `…\|s\| s.len() >` → `…\|s\| s.len() >=` | caught by `pda_known_answers_with_canonical_bumps` |
| M11 | `pubkey` | `h.update(program_id.0);` → `∅` | caught by `create_program_address_rejects_on_curve_results` |
| M12 | `pubkey` | `h.update(MARKER);` → `∅` | caught by `create_program_address_rejects_on_curve_results` |
| M13 | `pubkey` | `for s in seeds { h.update(s); }` → `∅` | caught by `create_program_address_rejects_on_curve_results` |
| M14 | `pubkey` | `if ` → `if !` | caught by `create_program_address_rejects_on_curve_results` |
| M15 | `pubkey` | `…seeds.len() >=` → `…seeds.len() >` | **equivalent**: `find_program_address` appends the bump as a 17th seed, and `create_program_address` already refuses 17 seeds with the same `TooManySeeds`, so the earlier check changes no observable result |
| M16 | `pubkey` | `for bump in (0..=255u8).rev()` → `for bump in 0..=255u8` | caught by `pda_known_answers_with_canonical_bumps` |
| M17 | `pubkey` | `…MAX_SEEDS: usize = 16` → `…MAX_SEEDS: usize = 17` | caught by `seed_limits` |
| M18 | `pubkey` | `…MAX_SEED_LEN: usize = 32` → `…MAX_SEED_LEN: usize = 33` | caught by `seed_limits` |
| M19 | `pubkey` | `…b"ProgramDerivedAddress` → `…b"ProgramDerivedAddres` | caught by `create_program_address_rejects_on_curve_results` |
| M20 | `pubkey` | `…PdaError::OnCurve) => continue` → `…PdaError::OnCurve) => return Err(PdaError::NoBump)` | caught by `pda_is_never_on_the_curve_and_depends_on_every_input` |
| M21 | `pubkey` | `all.push(&b` → `all.push(&[255]` | caught by `pda_is_never_on_the_curve_and_depends_on_every_input` |
| M22 | `pubkey` | `…v.try_into().ok()?` → `…v.try_into().unwrap_or([0; 32])` | caught by `pubkey_text_form` |
| M23 | `account` | `…LAMPORTS_PER_BYTE_YEAR * EXEMPTION_YEARS` → `…LAMPORTS_PER_BYTE_YEAR` | caught by `rent_minimums` |
| M24 | `account` | `…ACCOUNT_OVERHEAD: u64 = 128` → `…ACCOUNT_OVERHEAD: u64 = 127` | caught by `rent_minimums` |
| M25 | `account` | `…LAMPORTS_PER_BYTE_YEAR: u64 = 3480` → `…LAMPORTS_PER_BYTE_YEAR: u64 = 3481` | caught by `rent_minimums` |
| M26 | `account` | `…ACCOUNT_OVERHEAD + data_len as u64` → `…ACCOUNT_OVERHEAD` | caught by `rent_minimums` |
| M27 | `instruction` | `0..=0x7f` → `0..=0x7e` | caught by `serialized_length_matches_the_hand_count` |
| M28 | `instruction` | `0x80..=0x3fff` → `0x80..=0x3ffe` | caught by `shortvec_widens_at_16384_bytes_of_data` |
| M29 | `instruction` | `n += 3` → `n += 2` | caught by `transaction_size_limit_is_1232_bytes` |
| M30 | `instruction` | `n += 32; // recent blockhash` → `∅` | caught by `serialized_length_matches_the_hand_count` |
| M31 | `instruction` | `n += shortvec_len(self.instructions.len())` → `n += 0` | caught by `transaction_size_limit_is_1232_bytes` |
| M32 | `instruction` | `n += 1 + shortvec_len(ix.accounts.len()) + ` → `n += 1 + ` | caught by `transaction_size_limit_is_1232_bytes` |
| M33 | `instruction` | `n += shortvec_len(ix.data.len()) + ` → `n += ` | caught by `transaction_size_limit_is_1232_bytes` |
| M34 | `instruction` | `…self.signers.len()) + 64` → `…self.signers.len()) + 32` | caught by `transaction_size_limit_is_1232_bytes` |
| M35 | `instruction` | `…MAX_TX_BYTES: usize = 1232` → `…MAX_TX_BYTES: usize = 1233` | caught by `transaction_size_limit_is_1232_bytes` |
| M36 | `instruction` | `.chain(std::iter::once(ix.program_id` → `.chain(std::iter::empty(` | caught by `serialized_length_matches_the_hand_count` |
| M37 | `instruction` | `…seen.insert(k) ` → `…seen.insert(k) \|\| true ` | caught by `account_keys_are_deduplicated_signers_first` |
| M38 | `runtime` | `…!is_writable { if pre != post` → `…!is_writable { if false` | caught by `readonly_accounts_cannot_change_even_for_their_owner` |
| M39 | `runtime` | `if post.executable != pre.executabl` → `if fals` | caught by `programs_cannot_be_edited_or_made_executable` |
| M40 | `runtime` | `…pre.lamports && !owns` → `…pre.lamports && false` | caught by `only_the_owner_may_debit_but_anyone_may_credit` |
| M41 | `runtime` | `…post.lamports <` → `…post.lamports <=` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M42 | `runtime` | `…pre.data && (!owns \|\| pre.executable` → `…pre.data && (!owns` | caught by `programs_cannot_be_edited_or_made_executable` |
| M43 | `runtime` | `…pre.data && (!owns \|\| pre.executable)` → `…pre.data && false` | caught by `only_the_owner_may_change_data` |
| M44 | `runtime` | `if sum_before != sum_after` → `if false` | caught by `random_transactions_conserve_lamports_and_never_break_rent` |
| M45 | `runtime` | `…pre.executable \|\| post.data.iter().any(\|&b\| b != 0)` → `…pre.executable` | caught by `owner_changes_need_ownership_and_zeroed_data` |
| M46 | `runtime` | `(!owns \|\| pre.executable \|\| p` → `(!owns \|\| p` | caught by `a_program_cannot_hand_off_an_executable_account` |
| M47 | `runtime` | `(!owns \|\| ` → `(` | caught by `owner_changes_need_ownership_and_zeroed_data` |
| M48 | `runtime` | `if units >` → `if units >=` | caught by `compute_budget_scales_with_instructions_and_is_capped` |
| M49 | `runtime` | `…c.is_signer) \|\| pda_signers.contains(&m.pubkey)` → `…c.is_signer) \|\| true` | caught by `a_cpi_cannot_invent_a_signature` |
| M50 | `runtime` | `…m.is_signer && !(held.iter().any(\|c\| c.is_signer) \|\| pda_signe…` → `…m.is_signer && !pda_signers.contains(&m.pubkey` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M51 | `runtime` | `if m.is_writable && !held.iter().any(\|c\| c.is_writable)` → `if false` | caught by `a_cpi_cannot_gain_write_access` |
| M52 | `runtime` | `if held.is_empty()` → `if false` | caught by `a_cpi_can_only_use_accounts_it_was_given` |
| M53 | `runtime` | `…self.stack.len() >` → `…self.stack.len() >=` | caught by `recursion_depth_is_limited` |
| M54 | `runtime` | `…MAX_CPI_DEPTH ` → `…MAX_CPI_DEPTH + 1 ` | caught by `recursion_depth_is_limited` |
| M55 | `runtime` | `&& last.program_id != program_id` → `&& false` | caught by `reentrancy_through_another_program_is_refused` |
| M56 | `runtime` | `…last.program_id !` → `…last.program_id =` | caught by `recursion_depth_is_limited` |
| M57 | `runtime` | `if !self.work.get(&program_id).is_some_and(\|a\| a.executable)` → `if false` | caught by `programs_must_exist_and_be_executable` |
| M58 | `runtime` | `if caller.before.contains_key(k) { caller.before.insert(*k, self.wo…` → `let _ = k;` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M59 | `runtime` | `self.consume(CPI_COST` → `self.consume(CPI_COST - 1` | caught by `create_makes_a_pda_account_owned_by_the_program` |
| M60 | `runtime` | `…FEE_PER_SIGNATURE: u64 = 5000` → `…FEE_PER_SIGNATURE: u64 = 5001` | caught by `close_returns_the_lamports_and_deletes_the_account` |
| M61 | `runtime` | `…MAX_CPI_DEPTH: usize = 4` → `…MAX_CPI_DEPTH: usize = 5` | caught by `recursion_depth_is_limited` |
| M62 | `runtime` | `…u64 = 200_000` → `…u64 = 200_001` | caught by `one_instruction_gets_exactly_200_000_units` |
| M63 | `runtime` | `…u64 = 1_400_000` → `…u64 = 1_400_001` | caught by `the_cap_is_exactly_1_400_000_even_for_one_hungry_instruction` |
| M64 | `runtime` | `…CPI_COST: u64 = 1000` → `…CPI_COST: u64 = 1001` | caught by `create_makes_a_pda_account_owned_by_the_program` |
| M65 | `runtime` | `if size >` → `if size >=` | caught by `transaction_size_limit_is_1232_bytes` |
| M66 | `runtime` | `if m.is_signer && !tx.signers.contains(&m.pubkey)` → `if false` | caught by `the_pipeline_in_one_transaction` |
| M67 | `runtime` | `Some(left) if left == 0 \|\| left >= rent_exempt_minimum(data_len` → `Some(_` | caught by `random_transactions_conserve_lamports_and_never_break_rent` |
| M68 | `runtime` | `…== 0 \|\| left >=` → `…== 0 \|\| left >` | caught by `the_fee_may_not_leave_the_payer_below_the_rent_minimum` |
| M69 | `runtime` | `self.fees_collected += fee;` → `∅` | caught by `close_returns_the_lamports_and_deletes_the_account` |
| M70 | `runtime` | `if *a != old && a.lamports != 0 && a.lamports < rent_exempt_minimum…` → `if false` | caught by `random_transactions_conserve_lamports_and_never_break_rent` |
| M71 | `runtime` | `…a.lamports <` → `…a.lamports <=` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M72 | `runtime` | `if *a != old && ` → `if ` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M73 | `runtime` | `if a.lamports == 0` → `if false` | caught by `close_returns_the_lamports_and_deletes_the_account` |
| M74 | `runtime` | `let total = (UNITS_PER_INSTRUCTION * tx.instructions.len() as u64).…` → `let total = UNITS_PER_INSTRUCTION * tx.instructions.len() as u64` | caught by `compute_budget_scales_with_instructions_and_is_capped` |
| M75 | `runtime` | `units_used: total - remaining` → `units_used: total` | caught by `create_makes_a_pda_account_owned_by_the_program` |
| M76 | `runtime` | `…m.pubkey).or_insert(false) \|` → `…m.pubkey).or_insert(false) ` | caught by `a_key_listed_twice_is_writable_if_either_meta_says_so` |
| M77 | `runtime` | `…"exists").lamports -= fee` → `…"exists").lamports -= 0` | caught by `close_returns_the_lamports_and_deletes_the_account` |
| M78 | `runtime` | `…FEE_PER_SIGNATURE * tx.signers.len() as u64` → `…FEE_PER_SIGNATURE` | caught by `fee_is_per_signature_and_payer_must_afford_it` |
| M79 | `runtime` | `let size = tx.serialized_len(); if size > MAX_TX_BYTES { return Err…` → `let size = 0;` | caught by `transaction_size_limit_is_1232_bytes` |
| M80 | `runtime` | `…executable: tru` → `…executable: fals` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M81 | `system` | `…* 1024 * 1024` → `…* 1024 * 1024 + 1` | caught by `allocate_gives_an_empty_account_data` |
| M82 | `system` | `…COST: u64 = 150` → `…COST: u64 = 151` | caught by `create_makes_a_pda_account_owned_by_the_program` |
| M83 | `system` | `…MAX_DATA_LEN \|\| inv.key(0)? == inv.key(1)? ` → `…MAX_DATA_LEN ` | caught by `create_account_needs_two_distinct_signers` |
| M84 | `system` | `if space >` → `if space >=` | caught by `create_account_accepts_max_space_and_refuses_one_more` |
| M85 | `system` | `if !new.data.is_empty() \|\| new.owner != Pubkey::default()` → `if false` | caught by `create_account_refuses_an_account_in_use` |
| M86 | `system` | `if !new.data.is_empty() \|\| ` → `if ` | caught by `create_account_refuses_an_account_in_use` |
| M87 | `system` | `…!new.data.is_empty() \|\| new.owner != Pubkey::default() ` → `…!new.data.is_empty() ` | caught by `create_account_refuses_an_account_in_use` |
| M88 | `system` | `if src.owner != Pubkey::default()` → `if false` | caught by `transfer_source_rules` |
| M89 | `system` | `if !src.data.is_empty()` → `if false` | caught by `transfer_source_rules` |
| M90 | `system` | `…src.lamports <` → `…src.lamports <=` | caught by `transfer_of_the_exact_balance_is_allowed_and_one_more_is_not` |
| M91 | `system` | `…new_dest = dest.checked_add(amount).ok_or(Error::ArithmeticOverflow)?` → `…new_dest = dest.wrapping_add(amount)` | caught by `transfer_into_a_full_account_overflows_cleanly` |
| M92 | `system` | `new.data = vec![0; space as usize` → `new.data = vec![0; 0` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M93 | `system` | `new.owner` → `let _` | caught by `a_counter_is_per_authority` |
| M94 | `system` | `(0, 52` → `(0, 53` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M95 | `system` | `(1, 36` → `(1, 37` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M96 | `system` | `(2, 12` → `(2, 13` | caught by `create_with_a_prefunded_address_tops_up_only_the_difference` |
| M97 | `system` | `(8, 12` → `(8, 13` | caught by `allocate_gives_an_empty_account_data` |
| M98 | `system` | `if inv.account(0)?.owner != Pubkey::default()` → `if false` | caught by `assign_changes_the_owner_once` |
| M99 | `system` | `…MAX_DATA_LEN \|\| !acc.data.is_empty() \|\| ` → `…MAX_DATA_LEN \|\| ` | caught by `allocate_gives_an_empty_account_data` |
| M100 | `system` | `…u64_at(data, 4), u64_at(data, 12` → `…u64_at(data, 12), u64_at(data, 4` | caught by `a_counter_is_per_authority` |
| M101 | `system` | `…Pubkey(data[20..52` → `…Pubkey(data[19..51` | caught by `a_counter_is_per_authority` |
| M102 | `system` | `need_signer(inv, 1)?; ` → `∅` | caught by `create_account_needs_two_distinct_signers` |
| M103 | `system` | `inv.account_mut(0)?.owner` → `let _` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M104 | `system` | `…u64_at(data, 4` → `…u64_at(data, 8` | caught by `create_with_a_prefunded_address_tops_up_only_the_difference` |
| M105 | `system` | `…from)?.lamports -= amount` → `…from)?.lamports -= 0` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M106 | `counter` | `…SPACE: usize = 40` → `…SPACE: usize = 41` | caught by `close_returns_the_lamports_and_deletes_the_account` |
| M107 | `counter` | `…COST: u64 = 100` → `…COST: u64 = 101` | caught by `create_makes_a_pda_account_owned_by_the_program` |
| M108 | `counter` | `if counter.owner != inv.program_id()` → `if false` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M109 | `counter` | `if !inv.is_writable(0)?` → `if false` | caught by `increment_checks_signer_authority_owner_and_size` |
| M110 | `counter` | `if counter.data.len() != SPACE` → `if false` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M111 | `counter` | `if !inv.is_signer(1)?` → `if false` | caught by `increment_checks_signer_authority_owner_and_size` |
| M112 | `counter` | `if counter.data[..32] != inv.key(1)?.0` → `if false` | caught by `close_needs_the_authority` |
| M113 | `counter` | `if !inv.is_signer(0)?` → `if false` | caught by `create_needs_the_payer_signature_and_the_real_system_program` |
| M114 | `counter` | `if inv.key(2)? != Pubkey::default()` → `if false` | caught by `create_needs_the_payer_signature_and_the_real_system_program` |
| M115 | `counter` | `inv.consume(1500)?; // searching for the canonical bump` → `inv.consume(1499)?;` | caught by `create_makes_a_pda_account_owned_by_the_program` |
| M116 | `counter` | `…counter_key \|\| canonical != *bump ` → `…counter_key ` | caught by `the_pda_cannot_be_created_with_a_wrong_address_or_bump` |
| M117 | `counter` | `if expected != counter_key \|\| ` → `if ` | caught by `the_pda_cannot_be_created_with_a_wrong_address_or_bump` |
| M118 | `counter` | `if !existing.data.is_empty() \|\| existing.owner != Pubkey::default()` → `if false` | **equivalent**: the counter's own "address in use" check is a second line of defense; without it the CPI to the system program refuses the same account with the same `AlreadyInUse` |
| M119 | `counter` | `…rent_exempt_minimum(SPACE).saturating_sub(existing.lamports)` → `…rent_exempt_minimum(SPACE)` | caught by `create_with_a_prefunded_address_tops_up_only_the_difference` |
| M120 | `counter` | `inv.invoke(&ix, &[seeds` → `inv.invoke(&ix, &[` | caught by `a_counter_is_per_authority` |
| M121 | `counter` | `acc.data[..32].copy_from_slice(&payer` → `acc.data[..32].copy_from_slice(&counter_key` | caught by `a_counter_is_per_authority` |
| M122 | `counter` | `let n = n.checked_add(1).ok_or(Error::ArithmeticOverflow)?` → `let n = n.wrapping_add(1)` | caught by `counter_overflow_is_an_error_not_a_wrap` |
| M123 | `counter` | `acc.data[32..40].copy_from_slice(&n.to_l` → `acc.data[32..40].copy_from_slice(&n.to_b` | caught by `a_counter_is_per_authority` |
| M124 | `counter` | `if !inv.is_writable(1)?` → `if false` | caught by `close_needs_the_authority` |
| M125 | `counter` | `let total = to.checked_add(lamports).ok_or(Error::ArithmeticOverflow)?` → `let total = to.wrapping_add(lamports)` | caught by `close_into_a_full_wallet_overflows_cleanly` |
| M126 | `counter` | `acc.lamports = 0` → `acc.lamports = 1` | caught by `close_returns_the_lamports_and_deletes_the_account` |
| M127 | `counter` | `…= Vec::new(); acc.owner = Pubkey::default();` → `…= Vec::new();` | caught by `close_then_create_in_one_transaction_works_only_if_close_resets_the_owner` |
| M128 | `counter` | `acc.owner = Pubkey::default(` → `acc.owner = Pubkey::filled(9` | caught by `close_then_create_in_one_transaction_works_only_if_close_resets_the_owner` |
| M129 | `counter` | `[0, ` → `[0, _` | does not compile (the renamed binding `bump` is used later in the arm); not a usable mutant, kept for honesty |
| M130 | `counter` | `b"counter` → `b"counte` | caught by `an_attacker_cannot_hand_the_program_an_account_they_control` |
| M131 | `counter` | `…vec![0, bump` → `…vec![0, bump.wrapping_add(1)` | caught by `a_counter_is_per_authority` |
| M132 | `counter` | `…u64::from_le_bytes(data.get(32..40` → `…u64::from_le_bytes(data.get(31..39` | caught by `a_counter_is_per_authority` |

What the table does not show: it tests *these* edits, not every possible bug; the equivalent mutants could hide a real gap if my reasoning about them is wrong; and passing every mutant says nothing about the real runtime, about security or about behavior on a validator.

**Design notes.**

- Every dependency on the world is an argument: the accounts, the program registry, the compute budget. The logic is deterministic and fast to test (the property test runs 200 seeds in about a tenth of a second with optimized dependencies).
- The runtime enforces the **physics** (ownership, conservation, rent) once, in a pure function (`verify_changes`) that tests call directly; programs enforce the **policy** (who may do what) with explicit checks, each with a test that breaks only that check.
- Nested calls reuse the top-level path: one `run`, one verifier, one meter.
- Hashes, base58 strings, bumps and byte counts were checked against independent Python computations or by hand.
- Not handled: signatures, blockhashes, parallelism, sysvars, upgradeable programs, lookup tables, tokens, and everything in §8.3 and §11.

</details>

---

## 16. Key takeaways

- On Solana all state is in accounts (balance, bytes, owner, executable flag) and programs are stateless code that receive the accounts the **caller** chose (✓ one `Account` struct models wallets, programs and state).
- The runtime enforces the physics after each instruction: only the owner changes data or debits, anyone may credit, read-only stays unchanged, owner changes need zeroed data, lamports are conserved (✓ `verify_changes`, tested directly). The program enforces who is *allowed*: owner, signer, stored authority, layout, duplicates and program ids.
- A transaction is atomic, has a hard size limit that you can compute by hand, a compute budget and a fee that is paid even when it fails (✓ 203 bytes for the minimal case, 21 transfers per transaction, a failure in the second instruction undoes the first).
- A PDA is a hash that is off the curve, so nobody has its key; only the deriving program can sign for it, through CPI. Always use and verify the **canonical** bump (✓ bumps 255, 254 on the curve, 253 canonical, 252 a second valid address).
- CPI passes privileges and never invents them: signers come from the caller or from the caller's PDA, write access cannot be escalated, depth and re-entrancy are limited and each call costs compute (✓).
- Close accounts completely and check arithmetic: a drained account can be revived in the same transaction, and release-mode integers wrap (✓ revival test, overflow tests).
- Rent applies to the payer as well as to the accounts instructions touch (✓ found by the property test at seed 17).
- Mutation testing here pointed at real gaps: system-program instructions that no test exercised, unpinned limits and constants (including one test that was self-referential because it used the crate's own constant), and rules that had no test that broke them. Twenty-two real gaps were found and closed with 20 new tests (see the table in §15); 2 mutants remain, each with a stated reason for being equivalent, and 1 does not compile.

**Connections.** [Lesson 52](52_web3_reliability.md) built the off-chain side (signing, nonces, idempotency); this lesson is the on-chain side that those transactions run against, and the signer-and-authority checks of §4.2 are the on-chain version of the verifier's checks there. [Lesson 51](51_rpc_nodes_indexers.md) covers how a client reads accounts and submits transactions. [Lesson 50](50_networking_determinism.md) supplied limits and strict encodings, and [Lesson 49](49_merkle_transactions_consensus.md) the ideas of transactions, state and atomicity. [Lesson 48](48_crypto_primitives.md) provides the hash and curve arithmetic under PDAs. Lesson 54 introduces Anchor, which automates the account checklist of §4.2, and the token programs, which are programs of exactly the kind modeled here.

---

## 17. References

- Solana documentation: accounts, programs, transactions and instructions, program-derived addresses, cross-program invocation, fees, rent, compute budget, transaction limits: <https://solana.com/docs>
- The Solana Program Library and the `solana-program` crate documentation: <https://docs.rs/solana-program>
- RFC 8032, Edwards-Curve Digital Signature Algorithm (EdDSA): <https://www.rfc-editor.org/rfc/rfc8032>
- The documentation of `curve25519-dalek` (point decompression) and `sha2`: <https://github.com/dalek-cryptography/curve25519-dalek>, <https://github.com/RustCrypto/hashes>
- The Bitcoin base58 alphabet and algorithm (from memory): <https://en.bitcoin.it/wiki/Base58Check_encoding>
- Solana security write-ups and audit reports on missing owner, signer and account-validation checks (not consulted; search for current ones).

The URLs and titles above are the standard locations; I did not open them during this session (the sandbox cannot reach documentation and standards sites), so if a link has moved, search for the title. **Unchecked or from memory:** every figure about the **real** Solana runtime: the 1,232-byte transaction limit and the legacy wire layout, the 5,000-lamport signature fee, the rent parameters (3,480 lamports per byte-year, two years, 128 bytes overhead), the 10 MiB account limit, 16 seeds of 32 bytes, the `ProgramDerivedAddress` marker, the CPI depth, the compute budget (200,000 per instruction and 1,400,000 per transaction), the lamport conservation and ownership rules as stated, parallel execution by account locks, blockhash lifetime, address lookup tables, upgradeable-program authority, and the claim that on-chain programs run in release mode. The **model** values (constants in `runtime.rs`, `account.rs`, `system.rs`) are my reading of those from memory and are labeled as such where they appear. The base58 strings, the PDA addresses and bump facts, and the byte counts were verified by independent Python programs or by hand and by tests; the exercises' code in §12 and the two versions in §13 were compiled and run in a scratch crate (✓ 15 tests, `cargo clippy --all-targets -- -D warnings` clean). **Environment caveats:** timings are single-machine micro-measurements (several runs of one example) on a shared virtual machine with two cores; nothing here ran on a Solana validator or devnet, so no figure applies to a real deployment. Measurements and outcomes marked ✓ were observed with `rustc 1.97.0`, `sha2 0.11.0` and `curve25519-dalek 5.0.0`. **No part of this lesson makes any system secure.**
