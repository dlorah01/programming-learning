---
layout: default
title: "Lesson 46 — Databases, configuration, authentication, background jobs"
---
{% raw %}

# Lesson 46 — Databases, configuration, authentication, background jobs

**Module 14 · Application development** · Status: generated · Edition used in examples: **2024** · External dependencies: `rusqlite 0.41.1` (with `bundled`, SQLite **3.54.1**), `toml 1.2.1`, `serde 1.0.229`, `serde_json 1.0.154`, `argon2 0.6.0`, `sha2 0.11.0`, `getrandom 0.4.3`

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux, 2 cores. As before, ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior I did not run. **Everything here is SQLite on one machine.** I did not test PostgreSQL, MySQL, any ORM or query builder (`diesel`, `sea-orm`, `sqlx`), connection pools, async database access, replication, backups, OAuth/OIDC, JWT, cookies, MFA, password reset, rate limiting, an external queue (Redis, a cloud queue), cron-style schedulers, or any Windows/macOS behavior. Timings are single measurements on one shared virtual machine whose disk behavior is not that of your hardware (the `fsync` numbers in particular). I could not open the documentation sites, so rules stated without ✓ are from memory; the closing paragraph lists them. **Nothing in this lesson makes a system secure**: the authentication code is a worked example of good habits, not an audited product.

Builds on [Lesson 45](45_http_apis.md) (the API that would call this code, idempotency keys, 401/403, retries), [Lesson 44](44_cli_serialization.md) (serde, validation, secrets in arguments), [Lesson 43](43_observability_ci_supply_chain.md) (secrets and logs, dependency policy), [Lesson 39](39_testing.md) (testing), [Lesson 29](29_locks_deadlocks.md) (locks, deadlocks), [Lesson 27](27_threads_channels.md) (threads), [Lesson 26](26_error_design.md) (error design) and [Lesson 24](24_interior_mutability.md) (interior mutability). Lesson 47 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

A server that forgets everything when it restarts is a demo. Real programs keep **state** (a database), take their **settings** from the outside world (configuration), decide **who is calling** (authentication), and do **work that should not run inside a request** (background jobs). These four share one theme: each is a boundary between your process and something that outlives it or that you do not control, so the same discipline applies as in Lessons 44 and 45: validate what comes in, limit what it can cost, make failure explicit, and test the failure paths.

The running example is `datalab`, a small crate with four modules and 57 tests (✓ `cargo test` passes): `config` (defaults, TOML file, environment, validation), `db` (SQLite, pragmas, versioned migrations), `auth` (registration, Argon2id passwords, hashed session tokens) and `jobs` (a queue stored in the database, with leases, retries, a fencing token and a worker pool). Registering a user creates the user **and** a welcome-mail job in one transaction.

---

## 2. Choosing a store (from memory)

| | SQLite (this lesson) | A client/server database (PostgreSQL, MySQL) |
|---|---|---|
| Where it runs | inside your process, one file | a separate server, reached over the network |
| Writers | **one at a time** (readers can run alongside in WAL mode) | many, with row-level locking |
| Operations | nothing to run; back up a file (carefully) | an extra service to run, secure, upgrade and back up |
| Fits | tools, small services, tests, embedded and edge deployments, single-node apps | several application servers sharing data, heavy write concurrency, large datasets |

This is the textbook comparison, not something I benchmarked. The practical point for a learner: **SQLite is a real database** with transactions, constraints and indexes, and nearly everything below (parameters, transactions, migrations, idempotent writes, leases) transfers to the others. What changes is mostly the driver (`sqlx`, `tokio-postgres`, `diesel`: all from memory, not run), connection pooling, and error codes.

---

## 3. SQLite from Rust

### 3.1 Connecting, building ✓

`rusqlite` with the `bundled` feature compiles SQLite's C source into your binary. ✓ That needs a C compiler (`gcc` was present) and made a cold `cargo build` of `datalab` (dev profile, 2 cores, all dependencies) take **16.5 s** wall clock. It also means the SQLite version is fixed by your lockfile instead of the operating system: ✓ `rusqlite::version()` printed `3.54.1`. `Connection` is not `Sync`; give each thread its own connection (the worker pool does).

### 3.2 Parameters, not string building ✓

```rust
db.execute("INSERT INTO users(email, name) VALUES (?1, ?2)", params![email, name])?;
let n: i64 = db.query_row("SELECT count(*) FROM users WHERE email = ?1", [email], |r| r.get(0))?;
```

SQL text and values travel separately, so a value can never become part of the statement. ✓ The experiments, on a table with two users:

| Code | Input | Result |
|---|---|---|
| `format!("... WHERE email = '{evil}'")` + `query_row` | `x' OR '1'='1` | **2 rows** (the condition was rewritten) |
| `WHERE email = ?1` with the same text | `x' OR '1'='1` | 0 rows |
| `execute_batch(&format!("INSERT ... VALUES ('{evil}','n')"))` | ``z','n'); DELETE FROM users; --`` | `Ok(())`, **0 rows left** |
| `execute("INSERT ...; DELETE FROM users", [..])` | two statements in one string | `Err("Multiple statements provided")` |

Two lessons: `execute_batch` runs *several* statements, which is exactly what an injection needs, so never feed it anything built from input; and the driver's refusal of multiple statements in `execute` is a safety net, not a defense. Bound parameters work for values only, not for table or column names; if a name must vary, pick it from a fixed list in your code (✓ no test of this, general rule).

### 3.3 Types and constraints do the validation they can ✓

- ✓ A plain `INTEGER` column **accepted the text `'abc'`** (SQLite uses type *affinity*: `'abc'` stayed text, `'42'` became the integer 42, `1.5` stayed real). A `STRICT` table refused it: ``cannot store TEXT value in INTEGER column s.n``. Use `STRICT` for new tables.
- ✓ Constraint violations are errors you can match on: `UNIQUE constraint failed: users.email` (`ErrorCode::ConstraintViolation`, extended code **2067**, which `rusqlite::ffi::SQLITE_CONSTRAINT_UNIQUE` names), `NOT NULL constraint failed: users.email`, `CHECK constraint failed: bal >= 0`, `FOREIGN KEY constraint failed`. `datalab::auth::register` turns the unique violation into `AuthError::EmailTaken`: it is the *database* that guarantees uniqueness (two racing requests cannot both pass a "does it exist?" check), the Rust code only translates the error.
- ✓ Foreign keys: SQLite's own default is to ignore them, but the bundled build here **already enforced** them before I ran `PRAGMA foreign_keys` (the violation was reported with the default settings). `Db::open` still sets the pragma explicitly, because relying on a build default is how code breaks when the build changes (the mutation table shows that removing the line changes nothing in *this* build).
- ✓ Missing rows are an error variant, not `None`: `query_row` returns `Error::QueryReturnedNoRows`; use `.optional()` (`OptionalExtension`) when absence is normal. Reading a column as the wrong type gives ``Invalid column type Text at index: 0, name: name``.

### 3.4 Transactions ✓

A transaction makes several statements succeed or fail together. `rusqlite`'s `Transaction` **rolls back when dropped** without `commit()`:

```rust
let tx = db.transaction()?;
tx.execute("UPDATE acct SET bal = bal - 30 WHERE id = 1", [])?;
// early return or `?` here drops `tx` => rollback
tx.commit()?;
```

✓ Tests: a dropped transaction leaves no rows, and another connection cannot see uncommitted rows. A `CHECK (bal >= 0)` rejected an overdraft with `CHECK constraint failed: bal >= 0`, and dropping the transaction restored the balance (100). This is why `register` can insert the user, enqueue the mail job, and fail anywhere in between without leaving half a registration (✓ test: with the `jobs` table dropped, `register` fails and **no user row exists**).

### 3.5 Locks, `busy_timeout`, and `IMMEDIATE` ✓

SQLite allows **one writer at a time**. A second connection that tries to write gets `database is locked` (`SQLITE_BUSY`) unless it is told to wait. ✓ With one connection holding an open write transaction:

| `busy_timeout` of the second writer | Result |
|---|---|
| 0 ms | `Err("database is locked")` after 86 µs |
| 50 ms | the same error after 51 ms |
| 300 ms | the same error after 302 ms |
| 2 s, and the holder commits after 200 ms | **success** after 232 ms |

(Note: the `rusqlite` default timeout is five seconds, from memory of its documentation: ✓ a mutant that sets the timeout to zero made my 150 ms wait test fail, so the setting matters; `Db::open` sets it explicitly from the configuration.)

The classic trap: a **deferred** transaction that reads and then writes.

```text
tx = x.transaction()            // DEFERRED: no lock yet
tx.query_row("SELECT v FROM k") // takes a read snapshot
y.execute("UPDATE k SET v = 5") // another connection commits a write
tx.execute("UPDATE k SET v = v + 1")   // -> Err("database is locked")   ✓
```

The write cannot be applied on top of a snapshot that is now stale, and waiting would not help, so SQLite fails at once even with a busy timeout. The cure is to **declare the intent up front**: `transaction_with_behavior(TransactionBehavior::Immediate)` takes the write lock at `BEGIN`, so the read-then-write sequence is atomic (✓ the same sequence succeeded and read the value 5). `datalab` uses `Immediate` for migrations and registration.

✓ I also compared journal modes (`DELETE` vs `WAL`): with a writer holding an open, uncommitted write transaction, a reader on another connection was **not** blocked in either mode (both returned the old count in about 100 µs). The documented difference between them appears while the writer *commits*, which this test did not exercise; `Db::open` uses WAL (✓ test asserts `PRAGMA journal_mode` returns `wal`), the usual choice for read-heavy concurrent use.

### 3.6 What a commit costs ✓

Inserting 2,000 rows into a file database (WAL mode), one statement per implicit transaction versus all in one transaction:

| `PRAGMA synchronous` | 2,000 separate commits | all in one transaction |
|---|---:|---:|
| `FULL` (the default in my runs) | 324.7 ms (162.4 µs each) | 2.3 ms (1.15 µs each) |
| `NORMAL` | 27.5 ms (13.8 µs each) | 2.0 ms (0.99 µs each) |
| `OFF` | 11.4 ms (5.7 µs each) | 2.0 ms (0.98 µs each) |

(Single run on this virtual disk.) Two readings: **batching writes into one transaction is the biggest lever** (about 140 times in the `FULL` row), and the durability setting is a real trade-off. From memory: `FULL` waits for data to reach stable storage at each commit; `NORMAL` in WAL mode can lose the most recent commits (but not corrupt the database) after a power failure; `OFF` can also corrupt on a crash. Don't change it without understanding that. `datalab` keeps the default.

### 3.7 Migrations ✓

The schema changes over time; the program must upgrade old files itself. `datalab::db` keeps an ordered list of SQL strings and records progress in SQLite's `user_version` pragma:

```rust
for (i, sql) in migrations.iter().enumerate() {
    let tx = conn.transaction_with_behavior(TransactionBehavior::Immediate)?;
    if user_version(&tx)? != i as i64 { continue; }          // someone else did it
    tx.execute_batch(&format!("{sql}\n;\nPRAGMA user_version = {};", i + 1))
        .map_err(|source| DbError::Migration { version: i + 1, source })?;
    tx.commit()?;
}
```

Rules, each backed by a test (✓ all 13 database tests pass):

- **Each step is its own transaction, and DDL is transactional in SQLite.** A migration `CREATE TABLE b; CREATE TABLE a` where `a` already exists failed with ``table a already exists`` and **table `b` was rolled back with it**; the version stayed at 1. (Some other databases do not roll back schema changes; from memory.)
- **Take the lock first, then check the version.** Eight threads opening the same new file at once all succeeded and the version ended at 2; without the re-check inside the lock a second process could try to create an existing table. (Mutation results for this rule are in §13: removing the re-check was caught by the reopen tests, and switching `IMMEDIATE` to `DEFERRED` by the 8-thread test.)
- **Refuse a database from the future.** A file at version 9 made a program that knows 2 migrations return `TooNew { found: 9, supported: 2 }` and touch nothing, instead of running old code on a schema it does not understand.
- **Never edit a shipped migration; append a new one.** Existing databases already ran the old text.
- A bug my own test found: appending `PRAGMA user_version = N` to a migration that did not end in a semicolon produced ``near "user_version": syntax error``. The runner now inserts `;` itself.

### 3.8 Indexes ✓

A table of 200,000 rows, `SELECT count(*) FROM ev WHERE user = 42`, 200 repetitions, release build:

| | plan (`EXPLAIN QUERY PLAN`) | per query |
|---|---|---:|
| no index | `SCAN ev` | 3415.1 µs |
| `CREATE INDEX ev_user ON ev(user)` | `SEARCH ev USING COVERING INDEX ev_user (user=?)` | 2.8 µs |

About 1,200 times faster for one extra line. The job queue relies on the same idea: `jobs_ready ON jobs(state, run_at)` is what `claim` searches (✓ a test reads the plan and requires `jobs_ready`; plan wording can change between SQLite versions, so treat that test as a tripwire, not a specification). Indexes cost space and slow writes; add them for queries you actually run.

---

## 4. Configuration

### 4.1 Layers ✓

`Config::load(file_text, env)` applies, in this order: **defaults → TOML file → environment → validation**. Each layer overrides the previous one; a key missing from a layer keeps the earlier value.

```toml
# datalab.toml
[db]
path = "/var/lib/app/app.db"
[jobs]
workers = 8
```

```text
DATALAB_JOBS_WORKERS=16 datalab ...      # the environment wins: 16
```

✓ Tests cover precedence (file over default, environment over file), every environment variable mapped to the right field (each test value is distinct, so a swapped pair fails), and that variables without the `DATALAB_` prefix are ignored. The loader takes the environment as an *iterator of pairs*, not by calling `std::env::vars()` itself, so tests need no global state (setting environment variables in tests races with other threads).

### 4.2 Strict on purpose ✓

- **Unknown keys are errors** in the file (`deny_unknown_fields`) *and* in the environment: a misspelled `DATALAB_JOBS_WORKER` is `ConfigError::UnknownEnv`, because a setting that silently does nothing is the worst kind. Variables that merely look similar (`DATALABX`, `MY_DATALAB_JOBS`) are not ours and are ignored.
- **Errors say where and what.** TOML errors carry positions; ✓ the parser's messages:

| Input | Message |
|---|---|
| table `db` missing | ``TOML parse error at line 1, column 1`` + ``missing field `db` `` (when the schema has no defaults) |
| `busy_ms = 'soon'` for a `u64` | ``invalid type: string "soon", expected u64`` at line 4, column 9 |
| `busy_ms = -5` | ``invalid value: integer `-5`, expected u64`` |
| extra key | ``unknown field `listen_port`, expected one of `listen`, `workers`, `db` `` |
| `listen = ` (no value) | ``string values must be quoted, expected literal string`` at line 1, column 10 |
| a key twice | ``duplicate key`` at line 2, column 1 |

(From a scratch program with a different schema; `datalab`'s own test checks ``unknown field `workerz` `` and `line 2`.)

- **Ranges are validated after merging**, because a value can be fine in the file and wrong after an override. Every range is tested at both ends (✓ `workers` 1 and 64 accepted, 0 and 65 refused, and likewise for the other eight). The error names the setting and the rule: ``invalid setting jobs.workers: 0 is not in 1..=64``.
- **An unparsable environment value names the variable but not the value** (✓ test with `12x`): the value of a *secret* variable that fails to parse would otherwise end up in logs.

### 4.3 Secrets ✓

`Secret(String)` has a hand-written `Debug` that prints `Secret(<redacted>)`, and no `Display` at all. ✓ A test loads the API key from the environment and from the file and checks that `{config:?}` contains neither. Habits that go with it (Lessons 43 and 44): secrets come from the environment or a file, **never from command-line arguments** (visible to other users via the process list) or from the repository; they are not echoed in error messages; and the type system, not good intentions, prevents the accidental `println!("{config:?}")`.

What this does not give you: rotation, a secrets manager, encryption at rest, or protection from someone who can read the process environment (from memory: on Linux that is `/proc/<pid>/environ`, readable by the same user and by root; I did not test it here).

---

## 5. Authentication

Authentication answers "who is this?"; authorization answers "may they do this?". `datalab` implements the first only.

### 5.1 Passwords ✓

Never store passwords; store a **password hash** made by a deliberately slow, memory-hard function with a random salt. `datalab` uses Argon2id (the `argon2` crate, pure Rust) and the standard PHC string format:

```text
$argon2id$v=19$m=19456,t=2,p=1$<salt: 22 chars>$<hash: 43 chars>      (97 bytes in total)
```

The string carries the algorithm, version, parameters and salt, so a hash can be verified later even after you change the defaults. ✓ Observed:

- Two hashes of the *same* password differ (random salt); verification of the right password succeeds, a wrong one fails, and a garbage stored string yields ``invalid encoding`` which `verify_password` treats as a mismatch, never as success.
- ✓ **Cost** (release build, one run, this machine): defaults (19,456 KiB, 2 passes) took **15 to 19 ms** to verify (27 ms for the first hash, which includes page faults); 8 KiB / 1 pass took 25 µs; 64 MiB / 3 passes took 122 ms. The cost is the point: it is what an attacker with a stolen database pays *per guess*, and what a legitimate login pays once. A single SHA-256 took about 47 ns here, so a fast hash would let an attacker try roughly 300,000 times as many guesses per second per core (15 ms ÷ 47 ns; arithmetic, not a benchmark of an attack).
- ✓ **Debug builds are about 27 times slower**: 414 to 434 ms per default hash. Tests that hash with default parameters crawl. Two fixes: use cheap parameters in tests (`datalab`'s test configuration uses 8 KiB, 1 pass, giving the PHC prefix `m=8,t=1,p=1`, and a test checks the parameters are really applied), or optimize just this dependency in development: `[profile.dev.package.argon2] opt-level = 3` brought the default hash from 433 ms to 29 ms (✓ measured).
- Which parameters to choose is a capacity question: choose the strongest your login latency and server memory allow (each concurrent login uses about `m` KiB while hashing), and revisit it as hardware changes. The defaults of the crate match the minimum configuration of OWASP's password-storage guidance as I remember it; I did not re-read it.
- ✓ Passwords are bounded to 8–128 bytes (boundaries tested): an unbounded password lets a client make the server hash megabytes. `login` also rejects an over-long password before hashing.

### 5.2 Login without telling attackers more than they know ✓

- **One error for every failure.** Unknown e-mail, malformed e-mail, wrong password and an over-long password all return `InvalidCredentials` with the identical text `invalid e-mail or password` (✓ test over five cases).
- **Equal work for both paths.** When the e-mail is unknown, `login` still verifies against a precomputed dummy hash, so it takes about as long as a wrong password. ✓ Medians of nine logins at default parameters: right password 15.5 to 19.2 ms, wrong password 18.0 to 18.9 ms, **unknown e-mail 18.1 to 22.0 ms**. A test (cheaper parameters, minimum of three tries) fails if the unknown path takes less than a third of the known one; with the dummy verification removed the difference is thousands of times, so the test catches that mutation reliably.
- **Registration is the other leak.** `register` returns `EmailTaken` for an existing address, which tells a caller the account exists. That is usually accepted for usability; if it is not acceptable, registration must say the same thing either way and send the real answer by e-mail. A design choice, not a bug.
- Not implemented here but needed in a real system: rate limiting and lockout per account and per client, password reset, MFA, breach checks, and audit logging (from memory).

### 5.3 Sessions and tokens ✓

After login the server issues a **session token**: 32 bytes from the operating system's random generator (`getrandom`), written as 64 hex characters. The client sends it as `Authorization: Bearer <token>` (Lesson 45).

- **Store only the digest.** The `sessions` table holds `SHA-256(token)`; the token itself is returned once and kept nowhere on the server (✓ test recomputes the digest and checks the raw token is absent). A leaked database then contains no usable tokens. A fast hash is right *here* and wrong for passwords: a 256-bit random token cannot be guessed or brute-forced, so slowness adds nothing (reasoning from memory); a human-chosen password can be, so it needs the slow function.
- **Lookup by digest** in an indexed column: ✓ `authenticate` (SHA-256 plus lookup) took **4.7 to 5.1 µs** per call.
- **Expiry is exact**: valid at `expires_at - 1`, invalid at `expires_at` (✓ boundary test; time is a parameter, `now_ms`, so no sleeping). `purge_expired` deletes them; `logout` deletes one; `delete_user` removes the user and, through `ON DELETE CASCADE`, their sessions (✓ tests).
- `Session` has a `Debug` that hides the token (`Secret`), and tokens are compared by database equality on the digest, not by comparing secrets in Rust.
- Every flavor of wrong token (empty, one character changed, truncated, extended, upper-cased, leading space) is `Unauthorized` (✓).

Cookies, CSRF protection, JWTs, token refresh, OAuth and OpenID Connect are outside this lesson; each has its own pitfalls.

---

## 6. Background jobs

Anything slow or flaky that the user does not need to wait for (sending e-mail, resizing an image, calling another service) should happen **outside the request**. The simplest reliable queue is a table in the database you already have.

### 6.1 The table and the guarantees ✓

```sql
CREATE TABLE jobs(
  id INTEGER PRIMARY KEY, kind TEXT NOT NULL, payload TEXT NOT NULL,
  state TEXT NOT NULL CHECK (state IN ('queued','running','done','dead')),
  attempts INTEGER NOT NULL DEFAULT 0, max_attempts INTEGER NOT NULL CHECK (max_attempts >= 1),
  run_at INTEGER NOT NULL, lease_until INTEGER, last_error TEXT, created_at INTEGER NOT NULL
) STRICT;
```

The guarantee is **at least once**: a job runs one or more times, never silently zero. "Exactly once" is not something a queue alone can give, because a worker can finish the side effect and crash before recording it. So handlers must be **idempotent** (§6.5).

### 6.2 Claiming without double delivery ✓

```sql
UPDATE jobs SET state = 'running', attempts = attempts + 1, lease_until = ?now + ?lease
WHERE id = (SELECT id FROM jobs
            WHERE (state = 'queued' AND run_at <= ?now)
               OR (state = 'running' AND lease_until <= ?now)
            ORDER BY run_at, id LIMIT 1)
RETURNING id, kind, payload, attempts, max_attempts
```

One statement does *find* and *take*, and SQLite runs a statement under the write lock, so two workers cannot take the same row. ✓ 8 threads, each with its own connection, claimed from 400 jobs: **400 claims, 400 distinct ids**. The ordering is by `run_at` then `id` (✓ test with three jobs; a mutant ordering by `id` alone fails). A job is runnable when it is due, or when its lease has **expired** (its worker is presumed dead).

### 6.3 Leases, crashes and the fencing token ✓

A claim is a **lease**: the job belongs to that worker until `lease_until`. If the worker crashes, the lease expires and another worker claims the same job. ✓ Test: claim at time 0 with a 1,000 ms lease and "crash"; at 999 nothing is claimable; at **1,000** the job comes back as attempt 2.

That creates a second hazard: the original worker may not be dead, only slow, and later report its result. The `attempts` counter doubles as a **fencing token**: `complete` and `fail` only change the row `WHERE state = 'running' AND attempts = <my attempt>`. ✓ After the redelivery, the first worker's `complete` and `fail` both return `false` and change nothing; the second worker's `complete` succeeds. Without the fence a late success from attempt 1 could overwrite attempt 2's state.

A job whose lease expires while on its **last** attempt is declared `dead` with the message `lease expired during the final attempt` (✓ test), instead of running a never-ending extra attempt.

A handler that legitimately takes longer than the lease needs either a longer lease or a heartbeat that extends it (not implemented here); otherwise it will be run twice concurrently.

### 6.4 Retries with backoff ✓

A failed attempt goes back to `queued` with `run_at = now + base × 2^(attempt-1)`, capped at one hour, until `max_attempts` is reached and the job becomes `dead` (kept for inspection, with `last_error`, truncated to 500 characters, ✓ tested with multi-byte text). ✓ With base 100 ms the retries were due at +100, +200, +400 ms and not one millisecond earlier; the pure function gives 1,000, 2,000, 4,000, 8,000, 16,000, 32,000 for base 1,000, hits the one-hour cap at attempt 13 and does not overflow for `u32::MAX` attempts or a `u64::MAX` base. There is no jitter (random spread to avoid synchronized retries; from memory, not implemented).

A handler that **panics** is a failed attempt, not a dead worker thread: `run_one` wraps the call in `catch_unwind` (✓ test: the job is re-queued with `last_error = "handler panicked"`). Remember that with `panic = "abort"` (Lesson 42) this cannot work (from memory).

### 6.5 Idempotent handlers ✓

The queue may call the handler twice for one job (crash after the side effect, lease expiry). Make the effect itself idempotent, usually by a natural key and a unique constraint:

```sql
INSERT OR IGNORE INTO welcome_sent(user_id) VALUES (?1)    -- second run does nothing
```

✓ Test: the same job is delivered twice; the handler ran **2 times** and the effect was applied **once**. For external effects (an e-mail), pass an idempotency key to the provider if it supports one (Lesson 45), or record "sent" before you send and accept that a crash in between can lose or duplicate one message; there is no free lunch.

### 6.6 The transactional outbox ✓

The reason jobs live in the *same database* as the data: "create the user **and** schedule the welcome mail" becomes one transaction. If the process dies between two separate steps, you either have a user with no mail scheduled, or a mail scheduled for a user that was rolled back. ✓ `register` inserts both in one `IMMEDIATE` transaction; tests show a duplicate e-mail creates neither a second user nor a second job, and a failure on the job insert rolls the user back. With a separate message broker you need extra machinery (an outbox table and a relay) to get this property; that is the standard reason to start with a table.

### 6.7 The worker pool ✓

`WorkerPool::start(&config, handler)` opens one connection per worker **before** spawning (so a bad database path fails `start` instead of killing threads silently: ✓ test), then each thread loops: claim, run, record; sleep `poll_ms` when idle. Dropping the pool sets a stop flag and joins the threads; a worker **finishes its current job first** (✓ test: a 300 ms handler was running when `shutdown` was called; shutdown waited at least 150 ms and the job ended `done`; an idle pool stopped in under 2 s). Other verified behaviors: 100 jobs on 4 workers were each processed exactly once, and jobs that failed twice then succeeded ended `done` with `attempts = 3` and their last error still recorded.

Design limits: a database error in a worker is swallowed and retried after the poll interval (a real worker logs it, Lesson 43); idle workers poll instead of being woken (a notification channel would reduce latency; from memory); no scheduled/recurring jobs; no priorities; no per-kind routing (the handler receives `kind` and dispatches).

### 6.8 How fast, and why more workers did not help ✓

Release build, file database, default `synchronous`; three runs:

| Measurement | Result |
|---|---|
| enqueue, one commit per job | 242.6 to 300.7 µs per job |
| enqueue, 20,000 jobs in one transaction | 5.7 to 7.0 µs per job |
| 1 worker, empty handler | 1,645 to 2,137 jobs/s |
| 2 workers, empty handler | 1,567 to 2,040 jobs/s |
| 4 workers, empty handler | 1,501 to 2,144 jobs/s |

Throughput **did not scale** with the number of workers, and the runs overlap. Each job costs two commits (claim, then complete), commits are serialized by the single-writer rule, and each costs an `fsync` on this disk. So the ceiling is about the commit rate, not CPU. More workers help when the *handler* spends its time waiting (network, sleeping) outside the database, which an empty handler does not do. And batching (§3.6) is what makes enqueue 40 to 50 times cheaper per job. If you need far more throughput, you need a different store or a broker; if you need hundreds of jobs per second, this is plenty.

---

## 7. Testing code that touches the world ✓

- **Time is a parameter** (`now_ms`). Expiry, leases and backoff are tested at exact boundaries (`expires_at - 1` and `expires_at`) without sleeping. Only the worker loop calls the clock.
- **Databases in tests**: `Db::open_in_memory()` for logic that needs one connection; a unique temporary directory per test (removed on drop) when several connections or threads are involved, because every in-memory connection is a separate database.
- **Cheap hashing in tests** (§5.1), a real concurrency test (8 threads, 400 jobs) instead of reasoning about it, and a stress test of the migration race.
- **Timing-based tests have margins**: the unknown-e-mail test compares against a third of the known time, not equality; waits poll up to 20 s instead of sleeping once; the busy-timeout test accepts 120 ms to 3 s for a 150 ms setting. ✓ I ran the complete final suite 15 times in a row (after 25 clean runs of an earlier 55-test version): no failures. (While writing it I fixed several of my own tests: a claim at a time when a lease had already expired correctly found it, and a user count was off by one.)

---

## 8. Security checklist

1. SQL only with bound parameters; never `format!` input into SQL (✓ 2 rows leaked, a stacked `DELETE` succeeded).
2. Passwords hashed with Argon2id (or an equivalent vetted algorithm), unique salt, bounded length, parameters you can raise later (they live in the hash string).
3. Same response and similar cost for "no such user" and "wrong password" (✓ timing medians).
4. Session tokens from the OS random generator, long, stored only as digests, with expiry and revocation.
5. Secrets in a type that cannot print itself; none in arguments, logs, or error messages (✓ the value of a bad numeric setting is not echoed).
6. Validate configuration after merging all layers; reject unknown keys and unknown variables.
7. Least privilege for the database file (permissions, directory) and backups: not covered; a database file is only as private as its directory.
8. Migrations are code too: review them, test them against a copy of production data, and never edit shipped ones.
9. Dependency size: ✓ `Cargo.lock` lists 57 packages (the HTTP crate of Lesson 45 had 99, the CLI of Lesson 44 had 28) and no duplicate versions; `libsqlite3-sys` compiles a C library, so your supply-chain surface includes that C source (Lesson 43).
10. Treat every claim above as a *design discipline*, not an assurance. Passing these tests means those behaviors hold on this machine, not that the system resists attack.

---

## 9. Common mistakes

1. Building SQL with `format!` (§3.2).
2. Forgetting that `INTEGER` columns accept text unless the table is `STRICT`.
3. Doing a read-then-write in a `DEFERRED` transaction and being surprised by `database is locked` (§3.5).
4. One statement per commit when a batch would do (§3.6).
5. Holding a transaction (the write lock) open while doing slow work: ✓ `register` hashes the password *before* opening the transaction.
6. No `busy_timeout`, or one that hides a real deadlock; no limit on how long a job may run.
7. A queue consumer that is not idempotent, or that marks a job done *before* doing the work.
8. Missing the fence: letting a slow worker overwrite a newer attempt.
9. Different errors for unknown user and wrong password; plaintext or fast-hashed passwords; storing raw session tokens.
10. Config that silently ignores typos, or that prints secrets in `Debug`.
11. Testing password code with production-strength parameters in a debug build (✓ 27 times slower).
12. Editing an already-shipped migration.

---

## 10. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** A transfer moves 30 from account 1 to account 2 with two `UPDATE`s. Where must the transaction begin and end, what happens if the second update fails, and what does `CHECK (bal >= 0)` add?
2. **(Warm-up)** Why did `execute_batch(&format!(...))` let a `DELETE` through while `execute` refused two statements? Which one would you put near user input, and with what?
3. **(Easy)** Create an in-memory `STRICT` table of books, insert rows with bound parameters (one with a missing year), and read back a `Vec<Book>` with `query_map` and `Option<i32>`.
4. **(Medium)** Write `hit(db, page) -> i64` that counts page views with an upsert and `RETURNING`. Is the explicit transaction needed?
5. **(Medium)** Reproduce the `DEFERRED` upgrade failure with two connections, then fix it.
6. **(Medium-hard)** How would you choose Argon2 parameters for a service, and what do you do when you raise them next year and a user logs in?
7. **(Hard)** A job handler sometimes takes 90 seconds; the lease is 30 seconds. What happens, how does the fence limit the damage, and what are two real fixes?
8. **(Hard)** `claim` takes `now_ms` from each worker's own clock. What can go wrong when two machines share one database, and what would you change?

<details markdown="1">
<summary>Answers</summary>

1. `BEGIN` before the first `UPDATE`, `COMMIT` after the second; with `rusqlite`, `let tx = db.transaction()?` ... `tx.commit()?`. If the second `UPDATE` fails (or returns an error you propagate with `?`), the `Transaction` is dropped and **both** updates roll back, so money is neither lost nor created. `CHECK (bal >= 0)` makes the database itself reject an overdraft (✓ ``CHECK constraint failed: bal >= 0``), so the rule holds even if some other code path forgets to check. For concurrent transfers use `Immediate` if you read the balance first (§3.5).
2. `execute_batch` is designed to run a script of several statements, so injected `; DELETE ...; --` is executed (✓ 0 rows left); `execute` accepts exactly one statement and reports ``Multiple statements provided``. Neither is a defense. Near user input use `execute`/`query_row` with **bound parameters**, so the input is never parsed as SQL.
3. A possible answer:

   ```rust
   // needs: rusqlite (feature "bundled")
   use rusqlite::{Connection, params};

   #[derive(Debug, PartialEq)]
   struct Book {
       id: i64,
       title: String,
       year: Option<i32>,
   }

   fn main() -> rusqlite::Result<()> {
       let db = Connection::open_in_memory()?;
       db.execute_batch("CREATE TABLE books(id INTEGER PRIMARY KEY, title TEXT NOT NULL, year INTEGER) STRICT")?;
       for (title, year) in [("Dune", Some(1965)), ("Anon", None), ("It's", Some(1986))] {
           db.execute("INSERT INTO books(title, year) VALUES (?1, ?2)", params![title, year])?;
       }
       let mut st = db.prepare("SELECT id, title, year FROM books WHERE year IS NULL OR year > ?1 ORDER BY id")?;
       let books: Vec<Book> = st
           .query_map([1970], |r| Ok(Book { id: r.get(0)?, title: r.get(1)?, year: r.get(2)? }))?
           .collect::<Result<_, _>>()?;
       assert_eq!(books.len(), 2);
       assert_eq!(books[0], Book { id: 2, title: "Anon".into(), year: None });
       assert_eq!(books[1].title, "It's"); // the apostrophe needed no escaping
       println!("ok");
       Ok(())
   }
   ```

   `Option<i32>` maps to SQL `NULL`; the apostrophe in `It's` shows that bound values need no quoting.
4. A possible answer:

   ```rust
   // needs: rusqlite (feature "bundled")
   use rusqlite::Connection;

   fn hit(db: &Connection, page: &str) -> rusqlite::Result<i64> {
       db.query_row(
           "INSERT INTO hits(page, n) VALUES (?1, 1)
            ON CONFLICT(page) DO UPDATE SET n = n + 1 RETURNING n",
           [page],
           |r| r.get(0),
       )
   }

   fn main() -> rusqlite::Result<()> {
       let db = Connection::open_in_memory()?;
       db.execute_batch("CREATE TABLE hits(page TEXT PRIMARY KEY, n INTEGER NOT NULL) STRICT")?;
       assert_eq!((hit(&db, "/a")?, hit(&db, "/a")?, hit(&db, "/a")?), (1, 2, 3));
       assert_eq!(hit(&db, "/b")?, 1);
       println!("ok");
       Ok(())
   }
   ```

   No explicit transaction is needed: a single SQL statement is atomic, and the upsert avoids the racy "select, then insert or update" pair. You need a transaction when *several* statements must succeed together.
5. A possible answer (the second connection commits between the first one's read and write):

   ```rust
   // needs: rusqlite (feature "bundled")
   use rusqlite::{Connection, TransactionBehavior};
   use std::time::Duration;

   fn main() -> rusqlite::Result<()> {
       let path = std::env::temp_dir().join(format!("deferred-{}.db", std::process::id()));
       let mut x = Connection::open(&path)?;
       let y = Connection::open(&path)?;
       x.execute_batch("PRAGMA journal_mode=WAL; CREATE TABLE k(v INTEGER); INSERT INTO k VALUES (0)")?;
       x.busy_timeout(Duration::from_millis(100))?;
       y.busy_timeout(Duration::from_millis(100))?;

       let tx = x.transaction()?; // DEFERRED
       let _: i64 = tx.query_row("SELECT v FROM k", [], |r| r.get(0))?;
       y.execute("UPDATE k SET v = 5", [])?;
       let err = tx.execute("UPDATE k SET v = v + 1", []).unwrap_err();
       assert!(err.to_string().contains("database is locked"), "{err}");
       drop(tx);

       let tx = x.transaction_with_behavior(TransactionBehavior::Immediate)?;
       let v: i64 = tx.query_row("SELECT v FROM k", [], |r| r.get(0))?;
       tx.execute("UPDATE k SET v = v + 1", [])?;
       tx.commit()?;
       assert_eq!(v, 5);
       std::fs::remove_file(&path).ok();
       for ext in ["-wal", "-shm"] {
           std::fs::remove_file(format!("{}{ext}", path.display())).ok();
       }
       println!("ok");
       Ok(())
   }
   ```

   The deferred transaction holds a snapshot from before `y`'s write; applying the update would overwrite a state it never saw, so SQLite refuses at once. `Immediate` takes the write lock first, so the read and the write see the same state. (The error text and the immediate failure are what I observed in WAL mode; other journal modes may behave differently.)
6. Choose the strongest parameters your login latency and memory budget tolerate: measure on production-like hardware (✓ here 15 to 19 ms at 19 MiB / 2 passes, 122 ms at 64 MiB / 3 passes) and remember that every concurrent login needs about `m` KiB while hashing. Parameters are stored **in each hash string**, so raising them does not break old users: verify with the parameters in the stored string, and when a login succeeds and the stored parameters are weaker than the current ones, re-hash the password you just verified and update the row (a "rehash on login"). (The rehash step is the standard approach as I remember it; `datalab` does not implement it.) Re-hash costs one extra hash at login time; do it outside any open transaction.
7. At 30 s the lease expires while the first worker is still running, so a second worker claims the job (attempt 2) and the handler runs **concurrently**. The fence makes the first worker's later `complete`/`fail` no-ops (✓ both return `false`), so the database state is not corrupted, but the *side effects* happen twice, possibly at the same time. Fixes: (a) set the lease longer than the worst case, with a margin; (b) extend the lease while working (heartbeat: `UPDATE jobs SET lease_until = ?now + ?lease WHERE id = ?1 AND attempts = ?2`, which is fenced the same way); (c) split the work into smaller jobs; and in all cases keep the handler idempotent.
8. Leases and due times are compared with the timestamps workers pass in. If machine B's clock is two minutes ahead, it will consider machine A's live leases expired and steal running jobs, or run jobs "from the future" early; if it is behind, it will not retry on time. Fixes: use one clock, the database's (SQLite has none shared between machines, which is one reason it suits a single host; with PostgreSQL use `now()` inside the statement), or keep clocks synchronized and leases much longer than the expected skew. In `datalab` the workers share one machine, so this does not arise; I did not test multi-machine use (SQLite on a network file system is a separate warning).

</details>

---

## 11. Debugging challenge

This login function "works": the right password logs in and a wrong one is refused. A reviewer says it has three serious problems. Find them, show the worst one with an input, and fix all three.

```rust
// needs: rusqlite (feature "bundled")
use rusqlite::Connection;

fn setup() -> Connection {
    let db = Connection::open_in_memory().unwrap();
    db.execute_batch(
        "CREATE TABLE users(id INTEGER PRIMARY KEY, email TEXT NOT NULL UNIQUE, password TEXT NOT NULL);
         INSERT INTO users(email, password) VALUES ('ada@example.com', 'tulip'), ('bob@example.com', 'hunter2');",
    )
    .unwrap();
    db
}

fn login(db: &Connection, email: &str, password: &str) -> Result<i64, String> {
    let sql = format!("SELECT id, password FROM users WHERE email = '{email}'");
    let row: Option<(i64, String)> = db
        .query_row(&sql, [], |r| Ok((r.get(0)?, r.get(1)?)))
        .ok();
    match row {
        None => Err("no such user".into()),
        Some((id, stored)) if stored == password => Ok(id),
        Some(_) => Err("wrong password".into()),
    }
}

fn main() {
    let db = setup();
    assert_eq!(login(&db, "ada@example.com", "tulip"), Ok(1));
    assert_eq!(login(&db, "ada@example.com", "nope"), Err("wrong password".into()));
}
```

<details markdown="1">
<summary>Solution</summary>

Three problems, each real:

1. **SQL injection.** `format!` puts the e-mail into the SQL text. The input ``x' UNION SELECT 99, 'pw' --`` with the password `pw` makes the query return `(99, 'pw')`, so `login` returns `Ok(99)`: a login as a user that does not exist, with no password at all (✓ I ran it). Worse variants (`execute_batch`, other tables) exist; the fix is a bound parameter.
2. **Passwords stored in plaintext and compared with `==`.** Anyone who reads the database (a backup, a leaked file, SQL injection) has every password, and people reuse them. Store an Argon2id hash instead and verify with the library. (`==` on strings also stops at the first different byte; verification inside the hashing library avoids hand-rolled comparison.)
3. **Different errors for unknown user and wrong password** (`"no such user"` vs `"wrong password"`), which lets anyone test which e-mails have accounts. Return one message for both, and do the same amount of work (verify against a dummy hash) so the response *time* does not give it away.

A fixed version (✓ compiled and run with `rusqlite 0.41.1`, `argon2 0.6.0`; default Argon2 parameters, so it takes about 20 ms per login in a release build and much longer in a debug build, as measured above):

```rust
// needs: rusqlite (feature "bundled"), argon2 0.6
use argon2::password_hash::{PasswordHasher, PasswordVerifier};
use argon2::Argon2;
use rusqlite::{Connection, OptionalExtension};

fn hash(password: &str) -> String {
    Argon2::default().hash_password(password.as_bytes()).unwrap().to_string()
}

fn setup() -> (Connection, String) {
    let db = Connection::open_in_memory().unwrap();
    db.execute_batch(
        "CREATE TABLE users(id INTEGER PRIMARY KEY, email TEXT NOT NULL UNIQUE, password_hash TEXT NOT NULL) STRICT",
    )
    .unwrap();
    for (e, p) in [("ada@example.com", "tulip"), ("bob@example.com", "hunter2")] {
        db.execute("INSERT INTO users(email, password_hash) VALUES (?1, ?2)", (e, hash(p))).unwrap();
    }
    let dummy = hash("not a real password");
    (db, dummy)
}

fn login(db: &Connection, dummy: &str, email: &str, password: &str) -> Result<i64, &'static str> {
    const FAIL: &str = "invalid e-mail or password";
    let row: Option<(i64, String)> = db
        .query_row("SELECT id, password_hash FROM users WHERE email = ?1", [email], |r| {
            Ok((r.get(0)?, r.get(1)?))
        })
        .optional()
        .map_err(|_| "internal error")?;
    let (id, stored) = match &row {
        Some((id, h)) => (Some(*id), h.as_str()),
        None => (None, dummy), // same amount of work for an unknown address
    };
    let ok = Argon2::default().verify_password(password.as_bytes(), stored).is_ok();
    match (id, ok) {
        (Some(id), true) => Ok(id),
        _ => Err(FAIL),
    }
}

fn main() {
    let (db, dummy) = setup();
    assert_eq!(login(&db, &dummy, "ada@example.com", "tulip"), Ok(1));
    let fail = Err("invalid e-mail or password");
    assert_eq!(login(&db, &dummy, "ada@example.com", "nope"), fail);
    assert_eq!(login(&db, &dummy, "nobody@example.com", "tulip"), fail);
    assert_eq!(login(&db, &dummy, "x' UNION SELECT 99, 'pw' --", "pw"), fail);
    let stored: String = db.query_row("SELECT password_hash FROM users WHERE id = 1", [], |r| r.get(0)).unwrap();
    assert!(stored.starts_with("$argon2id$") && !stored.contains("tulip"));
    println!("ok");
}
```

Notes: `.optional()` turns "no row" into `None`, so a database failure is a different, internal error (not "no such user"); the unknown-address path verifies against `dummy` and cannot succeed; the injection string is now just an e-mail that matches nothing. Still missing, as in every example here: rate limiting, session creation, and a policy for password length.

</details>

---

## 12. Interview questions

1. *Why use bound parameters, and what can't they do?*
2. *What does a transaction guarantee, and what happens to it when a Rust `Transaction` is dropped?*
3. *Why does SQLite say "database is locked", and how do `busy_timeout` and `BEGIN IMMEDIATE` help?*
4. *How do you run schema migrations safely, including when two instances start at once?*
5. *How should a service load configuration, and how do you keep secrets out of logs?*
6. *How do you store passwords and session tokens, and why differently?*
7. *Why must "unknown user" and "wrong password" look the same?*
8. *What does "at least once" mean for a job queue, and what must handlers do about it?*
9. *What are a lease and a fencing token?*
10. *Why did adding workers not increase throughput in your queue?*

<details markdown="1">
<summary>Answers</summary>

1. They keep the SQL text and the values apart, so a value can never change the statement's structure (✓ `x' OR '1'='1` matched 0 rows with a parameter and 2 rows when pasted in). They only stand in for **values**; identifiers (table and column names) and keywords cannot be parameters, so choose those from a fixed list in code.
2. Atomicity (all or nothing), consistency with constraints, isolation from other connections' uncommitted work, and durability once committed. A `rusqlite::Transaction` that goes out of scope without `commit()` is **rolled back** (✓ tested), so an early return or a `?` leaves the database unchanged.
3. SQLite allows one writer; a second writer (or a stale reader trying to write) gets `SQLITE_BUSY`. `busy_timeout` makes a statement wait for the lock up to that long (✓ 86 µs at 0 ms, 51 ms at 50 ms, success after 232 ms when the holder released). A deferred transaction that read first cannot be upgraded if someone wrote in between and fails immediately (✓); `BEGIN IMMEDIATE` takes the write lock up front so read-then-write is atomic.
4. Ordered, append-only migrations with the applied version stored in the database; each in its own transaction (schema changes roll back with it in SQLite); take the write lock, **then** re-read the version; refuse a database newer than the program; test migrations on a copy of real data. ✓ Eight threads migrating one new file all succeeded.
5. Layers (defaults, file, environment) merged and then validated, strict about unknown keys and variables, errors that name the setting but not the secret value, secrets in a type with a redacting `Debug` (✓), and never in command-line arguments.
6. Passwords: a slow, memory-hard, salted hash (Argon2id), verified with the parameters stored in the hash; a human-chosen secret must make each guess expensive (✓ 15 ms vs 47 ns). Session tokens: long random values stored only as a fast digest (SHA-256), because a 256-bit random token cannot be guessed, so slowness buys nothing and lookup speed matters (✓ about 5 µs).
7. Different answers (or different timing) let an attacker discover which addresses have accounts. Return one error and do equal work (verify against a dummy hash) so both paths cost the same (✓ 18.1 to 22.0 ms unknown vs 18.0 to 18.9 ms wrong password).
8. A job may run more than once (a worker can crash after the side effect but before recording success; a lease can expire on a slow worker), never zero times. Handlers must be idempotent: a natural key with a unique constraint (✓ `INSERT OR IGNORE`: ran twice, applied once), or an idempotency key passed to the external service.
9. A lease is a time-limited claim on a job: if the worker does not finish before it expires, another worker may take the job. A fencing token (here the `attempts` counter) is included in the completion update so a worker that lost its lease cannot overwrite the new owner's result (✓ stale `complete` and `fail` returned `false`).
10. Each job costs two commits, commits are serialized by the single write lock and each waits for storage, so the commit rate is the ceiling (✓ 1.5k to 2.1k jobs/s at 1, 2 and 4 workers, with overlapping ranges). Workers help only when handlers wait on something else (network, sleep); batching claims or enqueues, or a different store, raises the ceiling.

</details>

---

## 13. Practical challenge: `datalab`

Build the crate `datalab`. Requirements:

1. `config`: `Config` with `db`, `auth` and `jobs` sections, defaults, `from_toml`, `apply_env` (`DATALAB_*`, unknown names rejected), `validate` with ranges, `load` in that order, a `Secret` type with redacted `Debug`.
2. `db`: `Db::open` (busy timeout, foreign keys, WAL) and `open_in_memory`; versioned migrations in `IMMEDIATE` transactions with a re-check, a refusal of newer databases, and a `migrate_with` for tests.
3. `auth`: `Authenticator` with Argon2id from the configuration, `register` (validation, hash outside the transaction, user plus welcome job in one transaction, unique-violation to `EmailTaken`), `login` (uniform error, dummy verification, random token stored as SHA-256), `authenticate` with exact expiry, `logout`, `purge_expired`, `delete_user`.
4. `jobs`: `enqueue`, atomic `claim` with leases, `complete` and `fail` fenced by the attempt number, exponential `backoff_ms`, dead state, `run_one` (panics are failures) and `WorkerPool` (one connection per worker, graceful stop).
5. Tests for each rule, including boundaries, concurrency and failure paths, and your own mutation checks.

<details markdown="1">
<summary>Solution</summary>

`Cargo.toml`:

```toml
[package]
name = "datalab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false

[dependencies]
rusqlite = { version = "0.41", features = ["bundled"] }
serde = { version = "1", features = ["derive"] }
toml = "1"
argon2 = "0.6"
sha2 = "0.11"
getrandom = "0.4"
serde_json = "1"
```

`src/lib.rs`:

```rust
//! `datalab`: configuration, SQLite storage, password and session
//! authentication, and a database-backed job queue, in one small crate.
#![deny(missing_docs)]

pub mod auth;
pub mod config;
pub mod db;
pub mod jobs;

pub use config::{Config, ConfigError, Secret};
pub use db::{Db, DbError};

/// Milliseconds since the Unix epoch. Library functions take `now_ms` as a
/// parameter so tests control time; only the worker loop calls this.
pub fn now_ms() -> i64 {
    use std::time::{SystemTime, UNIX_EPOCH};
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .map(|d| d.as_millis() as i64)
        .unwrap_or(0)
}
```

`src/config.rs`:

````rust
//! Layered configuration: defaults, then a TOML file, then environment
//! variables, then validation. Secrets never appear in `Debug` output.

use std::fmt;

use serde::Deserialize;

/// A string that must not be printed. `Debug` shows `<redacted>`.
#[derive(Clone, Deserialize)]
#[serde(transparent)]
pub struct Secret(String);

impl Secret {
    /// Wraps a secret value.
    pub fn new(value: impl Into<String>) -> Self {
        Secret(value.into())
    }

    /// The secret itself. Call this only where the value is really needed.
    pub fn expose(&self) -> &str {
        &self.0
    }
}

impl fmt::Debug for Secret {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str("Secret(<redacted>)")
    }
}

/// Everything that can go wrong while building a [`Config`].
#[derive(Debug)]
pub enum ConfigError {
    /// The TOML text is malformed or does not match the schema.
    Toml(toml::de::Error),
    /// An environment variable has a value that cannot be parsed.
    Env {
        /// Variable name.
        var: String,
        /// What was wrong with the value (never the value of a secret).
        message: String,
    },
    /// A `DATALAB_*` variable that names no setting (usually a typo).
    UnknownEnv(String),
    /// A value is outside its allowed range.
    Invalid {
        /// Dotted setting name, e.g. `jobs.workers`.
        field: &'static str,
        /// The rule that was broken.
        message: String,
    },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Toml(e) => write!(f, "invalid configuration file: {e}"),
            ConfigError::Env { var, message } => write!(f, "environment variable {var}: {message}"),
            ConfigError::UnknownEnv(var) => write!(f, "unknown environment variable {var}"),
            ConfigError::Invalid { field, message } => {
                write!(f, "invalid setting {field}: {message}")
            }
        }
    }
}

impl std::error::Error for ConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ConfigError::Toml(e) => Some(e),
            _ => None,
        }
    }
}

/// Database settings.
#[derive(Debug, Clone, Deserialize)]
#[serde(default, deny_unknown_fields)]
pub struct DbConfig {
    /// File path of the SQLite database.
    pub path: String,
    /// How long a statement waits for a lock held by another connection.
    pub busy_timeout_ms: u64,
}

/// Authentication settings.
#[derive(Debug, Clone, Deserialize)]
#[serde(default, deny_unknown_fields)]
pub struct AuthConfig {
    /// Lifetime of a session token.
    pub session_ttl_secs: u64,
    /// Argon2 memory cost in KiB.
    pub argon2_memory_kib: u32,
    /// Argon2 number of passes.
    pub argon2_iterations: u32,
}

/// Background job settings.
#[derive(Debug, Clone, Deserialize)]
#[serde(default, deny_unknown_fields)]
pub struct JobsConfig {
    /// Number of worker threads.
    pub workers: u32,
    /// Attempts before a job is declared dead.
    pub max_attempts: u32,
    /// How long a claimed job belongs to its worker before another may take it.
    pub lease_ms: u64,
    /// First retry delay; doubles with each attempt.
    pub backoff_base_ms: u64,
    /// How long an idle worker sleeps between polls.
    pub poll_ms: u64,
    /// Credential for the (imaginary) mail provider; never printed.
    pub api_key: Option<Secret>,
}

/// The whole configuration.
#[derive(Debug, Clone, Default, Deserialize)]
#[serde(default, deny_unknown_fields)]
pub struct Config {
    /// Database section.
    pub db: DbConfig,
    /// Authentication section.
    pub auth: AuthConfig,
    /// Jobs section.
    pub jobs: JobsConfig,
}

impl Default for DbConfig {
    fn default() -> Self {
        DbConfig {
            path: "datalab.db".into(),
            busy_timeout_ms: 5_000,
        }
    }
}

impl Default for AuthConfig {
    fn default() -> Self {
        AuthConfig {
            session_ttl_secs: 7 * 24 * 3600,
            argon2_memory_kib: 19_456,
            argon2_iterations: 2,
        }
    }
}

impl Default for JobsConfig {
    fn default() -> Self {
        JobsConfig {
            workers: 2,
            max_attempts: 5,
            lease_ms: 30_000,
            backoff_base_ms: 1_000,
            poll_ms: 200,
            api_key: None,
        }
    }
}

fn parse_env<T: std::str::FromStr>(var: &str, value: &str) -> Result<T, ConfigError>
where
    T::Err: fmt::Display,
{
    value.parse().map_err(|e: T::Err| ConfigError::Env {
        var: var.to_string(),
        message: format!("cannot parse the value: {e}"),
    })
}

fn check<T: PartialOrd + fmt::Display>(
    field: &'static str,
    value: T,
    lo: T,
    hi: T,
) -> Result<(), ConfigError> {
    if value < lo || value > hi {
        return Err(ConfigError::Invalid {
            field,
            message: format!("{value} is not in {lo}..={hi}"),
        });
    }
    Ok(())
}

impl Config {
    /// Parses a TOML document over the defaults (missing keys keep their default).
    pub fn from_toml(text: &str) -> Result<Config, ConfigError> {
        toml::from_str(text).map_err(ConfigError::Toml)
    }

    /// Applies `DATALAB_*` variables. Other variables are ignored; an unknown
    /// `DATALAB_*` name is an error so that a typo cannot silently do nothing.
    pub fn apply_env<I>(&mut self, vars: I) -> Result<(), ConfigError>
    where
        I: IntoIterator<Item = (String, String)>,
    {
        for (name, value) in vars {
            let Some(key) = name.strip_prefix("DATALAB_") else {
                continue;
            };
            match key {
                "DB_PATH" => self.db.path = value,
                "DB_BUSY_TIMEOUT_MS" => self.db.busy_timeout_ms = parse_env(&name, &value)?,
                "AUTH_SESSION_TTL_SECS" => self.auth.session_ttl_secs = parse_env(&name, &value)?,
                "AUTH_ARGON2_MEMORY_KIB" => self.auth.argon2_memory_kib = parse_env(&name, &value)?,
                "AUTH_ARGON2_ITERATIONS" => self.auth.argon2_iterations = parse_env(&name, &value)?,
                "JOBS_WORKERS" => self.jobs.workers = parse_env(&name, &value)?,
                "JOBS_MAX_ATTEMPTS" => self.jobs.max_attempts = parse_env(&name, &value)?,
                "JOBS_LEASE_MS" => self.jobs.lease_ms = parse_env(&name, &value)?,
                "JOBS_BACKOFF_BASE_MS" => self.jobs.backoff_base_ms = parse_env(&name, &value)?,
                "JOBS_POLL_MS" => self.jobs.poll_ms = parse_env(&name, &value)?,
                "JOBS_API_KEY" => self.jobs.api_key = Some(Secret::new(value)),
                _ => return Err(ConfigError::UnknownEnv(name)),
            }
        }
        Ok(())
    }

    /// Checks every range.
    pub fn validate(&self) -> Result<(), ConfigError> {
        if self.db.path.is_empty() {
            return Err(ConfigError::Invalid {
                field: "db.path",
                message: "must not be empty".into(),
            });
        }
        check("db.busy_timeout_ms", self.db.busy_timeout_ms, 0, 60_000)?;
        check(
            "auth.session_ttl_secs",
            self.auth.session_ttl_secs,
            60,
            30 * 24 * 3600,
        )?;
        check(
            "auth.argon2_memory_kib",
            self.auth.argon2_memory_kib,
            8,
            1 << 20,
        )?;
        check("auth.argon2_iterations", self.auth.argon2_iterations, 1, 10)?;
        check("jobs.workers", self.jobs.workers, 1, 64)?;
        check("jobs.max_attempts", self.jobs.max_attempts, 1, 20)?;
        check("jobs.lease_ms", self.jobs.lease_ms, 1, 3_600_000)?;
        check("jobs.backoff_base_ms", self.jobs.backoff_base_ms, 1, 60_000)?;
        check("jobs.poll_ms", self.jobs.poll_ms, 1, 60_000)?;
        Ok(())
    }

    /// Defaults, then the optional file text, then the environment, then validation.
    pub fn load<I>(toml_text: Option<&str>, env: I) -> Result<Config, ConfigError>
    where
        I: IntoIterator<Item = (String, String)>,
    {
        let mut config = match toml_text {
            Some(text) => Config::from_toml(text)?,
            None => Config::default(),
        };
        config.apply_env(env)?;
        config.validate()?;
        Ok(config)
    }
}
````

`src/db.rs`:

````rust
//! SQLite access: opening, pragmas and versioned migrations.

use std::fmt;
use std::time::Duration;

use rusqlite::{Connection, TransactionBehavior};

use crate::config::DbConfig;

/// Schema changes in order. Entry `i` upgrades a database from version `i` to `i + 1`.
/// Never edit an entry that has shipped; append a new one.
pub const MIGRATIONS: &[&str] = &[
    "CREATE TABLE users(
         id            INTEGER PRIMARY KEY,
         email         TEXT NOT NULL UNIQUE,
         name          TEXT NOT NULL,
         password_hash TEXT NOT NULL,
         created_at    INTEGER NOT NULL
     ) STRICT;
     CREATE TABLE sessions(
         token_hash TEXT PRIMARY KEY,
         user_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
         expires_at INTEGER NOT NULL
     ) STRICT;
     CREATE INDEX sessions_user ON sessions(user_id);",
    "CREATE TABLE jobs(
         id           INTEGER PRIMARY KEY,
         kind         TEXT NOT NULL,
         payload      TEXT NOT NULL,
         state        TEXT NOT NULL CHECK (state IN ('queued','running','done','dead')),
         attempts     INTEGER NOT NULL DEFAULT 0,
         max_attempts INTEGER NOT NULL CHECK (max_attempts >= 1),
         run_at       INTEGER NOT NULL,
         lease_until  INTEGER,
         last_error   TEXT,
         created_at   INTEGER NOT NULL
     ) STRICT;
     CREATE INDEX jobs_ready ON jobs(state, run_at);",
];

/// Errors from opening or migrating a database.
#[derive(Debug)]
pub enum DbError {
    /// SQLite reported an error.
    Sqlite(rusqlite::Error),
    /// Migration number `version` (1-based) failed and was rolled back.
    Migration {
        /// The migration that failed.
        version: usize,
        /// Why.
        source: rusqlite::Error,
    },
    /// The file was written by a newer program than this one.
    TooNew {
        /// Version stored in the file.
        found: i64,
        /// Newest version this program knows.
        supported: usize,
    },
}

impl fmt::Display for DbError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DbError::Sqlite(e) => write!(f, "database error: {e}"),
            DbError::Migration { version, source } => {
                write!(
                    f,
                    "migration {version} failed and was rolled back: {source}"
                )
            }
            DbError::TooNew { found, supported } => write!(
                f,
                "database schema version {found} is newer than this program supports ({supported})"
            ),
        }
    }
}

impl std::error::Error for DbError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            DbError::Sqlite(e) | DbError::Migration { source: e, .. } => Some(e),
            DbError::TooNew { .. } => None,
        }
    }
}

impl From<rusqlite::Error> for DbError {
    fn from(e: rusqlite::Error) -> Self {
        DbError::Sqlite(e)
    }
}

/// An open, migrated database connection. One connection must not be shared
/// between threads; each thread opens its own.
pub struct Db {
    conn: Connection,
}

fn user_version(conn: &Connection) -> rusqlite::Result<i64> {
    conn.query_row("PRAGMA user_version", [], |r| r.get(0))
}

/// Applies the pending entries of `migrations`, each in its own `IMMEDIATE`
/// transaction, so two processes starting together cannot both apply one.
pub fn migrate_with(conn: &mut Connection, migrations: &[&str]) -> Result<(), DbError> {
    let found = user_version(conn)?;
    if found > migrations.len() as i64 {
        return Err(DbError::TooNew {
            found,
            supported: migrations.len(),
        });
    }
    for (i, sql) in migrations.iter().enumerate() {
        let tx = conn.transaction_with_behavior(TransactionBehavior::Immediate)?;
        // Re-read inside the lock: another process may have applied it meanwhile.
        if user_version(&tx)? != i as i64 {
            continue;
        }
        // The extra ";" makes a migration that forgot its final semicolon work.
        let step = format!("{sql}\n;\nPRAGMA user_version = {};", i + 1);
        tx.execute_batch(&step)
            .map_err(|source| DbError::Migration {
                version: i + 1,
                source,
            })?;
        tx.commit()?;
    }
    Ok(())
}

impl Db {
    /// Opens (creating if needed) the database in `cfg`, configures the
    /// connection and applies all migrations.
    pub fn open(cfg: &DbConfig) -> Result<Db, DbError> {
        let mut conn = Connection::open(&cfg.path)?;
        conn.busy_timeout(Duration::from_millis(cfg.busy_timeout_ms))?;
        conn.execute_batch("PRAGMA foreign_keys = ON;")?;
        // journal_mode returns a row, so it cannot go through execute_batch.
        let _mode: String = conn.query_row("PRAGMA journal_mode = WAL", [], |r| r.get(0))?;
        migrate_with(&mut conn, MIGRATIONS)?;
        Ok(Db { conn })
    }

    /// A private in-memory database (tests, experiments). Not shareable.
    pub fn open_in_memory() -> Result<Db, DbError> {
        let mut conn = Connection::open_in_memory()?;
        conn.execute_batch("PRAGMA foreign_keys = ON;")?;
        migrate_with(&mut conn, MIGRATIONS)?;
        Ok(Db { conn })
    }

    /// The underlying connection.
    pub fn conn(&self) -> &Connection {
        &self.conn
    }

    /// The underlying connection, mutably (needed to start transactions).
    pub fn conn_mut(&mut self) -> &mut Connection {
        &mut self.conn
    }

    /// The schema version stored in the file.
    pub fn schema_version(&self) -> Result<i64, DbError> {
        Ok(user_version(&self.conn)?)
    }
}
````

`src/auth.rs`:

````rust
//! Registration, login and sessions.
//!
//! Passwords are stored as Argon2id hashes in PHC format. Session tokens are
//! 256 random bits; only their SHA-256 digest is stored, so a stolen database
//! does not contain usable tokens.

use std::fmt;

use argon2::password_hash::{PasswordHasher, PasswordVerifier};
use argon2::{Algorithm, Argon2, Params, Version};
use rusqlite::{OptionalExtension, TransactionBehavior, params};
use sha2::{Digest, Sha256};

use crate::config::{AuthConfig, Config, Secret};
use crate::db::Db;
use crate::jobs;

/// Shortest accepted password, in bytes.
pub const MIN_PASSWORD_BYTES: usize = 8;
/// Longest accepted password, in bytes (bounds the hashing work an attacker can request).
pub const MAX_PASSWORD_BYTES: usize = 128;
/// Longest accepted display name, in characters.
pub const MAX_NAME_CHARS: usize = 100;
/// Longest accepted e-mail address, in bytes.
pub const MAX_EMAIL_BYTES: usize = 254;
/// Kind of the job enqueued when a user registers.
pub const WELCOME_JOB: &str = "welcome_email";

/// Why an authentication operation failed.
#[derive(Debug)]
pub enum AuthError {
    /// The e-mail address is not acceptable.
    InvalidEmail,
    /// The password is too short or too long.
    WeakPassword,
    /// The display name is empty or too long.
    InvalidName,
    /// An account with this e-mail already exists.
    EmailTaken,
    /// Unknown e-mail or wrong password; deliberately indistinguishable.
    InvalidCredentials,
    /// Missing, unknown or expired session token.
    Unauthorized,
    /// The password hasher failed (bad parameters, internal error).
    Hashing(String),
    /// The system random number generator failed.
    Random(String),
    /// Database failure.
    Db(rusqlite::Error),
}

impl fmt::Display for AuthError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AuthError::InvalidEmail => f.write_str("invalid e-mail address"),
            AuthError::WeakPassword => write!(
                f,
                "password must be {MIN_PASSWORD_BYTES} to {MAX_PASSWORD_BYTES} bytes"
            ),
            AuthError::InvalidName => write!(f, "name must be 1 to {MAX_NAME_CHARS} characters"),
            AuthError::EmailTaken => f.write_str("an account with this e-mail already exists"),
            AuthError::InvalidCredentials => f.write_str("invalid e-mail or password"),
            AuthError::Unauthorized => f.write_str("missing, unknown or expired session"),
            AuthError::Hashing(m) => write!(f, "password hashing failed: {m}"),
            AuthError::Random(m) => write!(f, "random number generator failed: {m}"),
            AuthError::Db(e) => write!(f, "database error: {e}"),
        }
    }
}

impl std::error::Error for AuthError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AuthError::Db(e) => Some(e),
            _ => None,
        }
    }
}

impl From<rusqlite::Error> for AuthError {
    fn from(e: rusqlite::Error) -> Self {
        AuthError::Db(e)
    }
}

/// A freshly created session.
#[derive(Debug)]
pub struct Session {
    /// The bearer token to give to the client. Shown only once; not stored.
    pub token: Secret,
    /// The user it belongs to.
    pub user_id: i64,
    /// Expiry, milliseconds since the epoch.
    pub expires_at_ms: i64,
}

/// Normalizes an e-mail address (trim, ASCII lowercase) and checks its shape.
///
/// This is a sanity check, not the full RFC grammar; the only real proof that
/// an address works is sending mail to it.
pub fn normalize_email(raw: &str) -> Result<String, AuthError> {
    let email = raw.trim().to_ascii_lowercase();
    let ok = email.len() <= MAX_EMAIL_BYTES
        && email.chars().all(|c| !c.is_whitespace() && !c.is_control())
        && match email.split_once('@') {
            Some((local, domain)) => {
                !local.is_empty()
                    && !domain.contains('@')
                    && domain.contains('.')
                    && !domain.starts_with('.')
                    && !domain.ends_with('.')
            }
            None => false,
        };
    if ok {
        Ok(email)
    } else {
        Err(AuthError::InvalidEmail)
    }
}

fn hex(bytes: &[u8]) -> String {
    bytes.iter().map(|b| format!("{b:02x}")).collect()
}

fn token_digest(token: &str) -> String {
    hex(&Sha256::digest(token.as_bytes()))
}

/// Hashes and verifies passwords and manages sessions.
pub struct Authenticator {
    cfg: AuthConfig,
    argon2: Argon2<'static>,
    /// Verified against when the e-mail is unknown, so that both failure
    /// paths cost the same time.
    dummy_hash: String,
}

impl Authenticator {
    /// Builds an authenticator with the Argon2id parameters from `cfg`.
    pub fn new(cfg: &AuthConfig) -> Result<Self, AuthError> {
        let params = Params::new(cfg.argon2_memory_kib, cfg.argon2_iterations, 1, None)
            .map_err(|e| AuthError::Hashing(e.to_string()))?;
        let argon2 = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);
        let dummy_hash = argon2
            .hash_password(b"dummy password for timing")
            .map_err(|e| AuthError::Hashing(e.to_string()))?
            .to_string();
        Ok(Authenticator {
            cfg: cfg.clone(),
            argon2,
            dummy_hash,
        })
    }

    /// Hashes a password with a fresh random salt. Takes tens of milliseconds
    /// by design; do not call it while holding a database lock.
    pub fn hash_password(&self, password: &str) -> Result<String, AuthError> {
        self.argon2
            .hash_password(password.as_bytes())
            .map(|h| h.to_string())
            .map_err(|e| AuthError::Hashing(e.to_string()))
    }

    /// Checks `password` against a stored PHC string. A malformed stored hash
    /// is treated as a mismatch.
    pub fn verify_password(&self, password: &str, phc: &str) -> bool {
        self.argon2
            .verify_password(password.as_bytes(), phc)
            .is_ok()
    }

    /// Creates a user and, in the same transaction, a welcome-mail job.
    pub fn register(
        &self,
        db: &mut Db,
        cfg: &Config,
        email: &str,
        name: &str,
        password: &str,
        now_ms: i64,
    ) -> Result<i64, AuthError> {
        let email = normalize_email(email)?;
        let name = name.trim();
        if name.is_empty() || name.chars().count() > MAX_NAME_CHARS {
            return Err(AuthError::InvalidName);
        }
        if !(MIN_PASSWORD_BYTES..=MAX_PASSWORD_BYTES).contains(&password.len()) {
            return Err(AuthError::WeakPassword);
        }
        // Hash first, outside any transaction: the write lock is held only briefly.
        let hash = self.hash_password(password)?;
        let tx = db
            .conn_mut()
            .transaction_with_behavior(TransactionBehavior::Immediate)?;
        let inserted = tx.execute(
            "INSERT INTO users(email, name, password_hash, created_at) VALUES (?1, ?2, ?3, ?4)",
            params![email, name, hash, now_ms],
        );
        match inserted {
            Ok(_) => {}
            Err(rusqlite::Error::SqliteFailure(e, _))
                if e.extended_code == rusqlite::ffi::SQLITE_CONSTRAINT_UNIQUE =>
            {
                return Err(AuthError::EmailTaken);
            }
            Err(e) => return Err(e.into()),
        }
        let id = tx.last_insert_rowid();
        let payload = serde_json::json!({ "userId": id }).to_string();
        jobs::enqueue(
            &tx,
            WELCOME_JOB,
            &payload,
            now_ms,
            cfg.jobs.max_attempts,
            now_ms,
        )?;
        tx.commit()?;
        Ok(id)
    }

    /// Verifies the credentials and starts a session.
    pub fn login(
        &self,
        db: &Db,
        email: &str,
        password: &str,
        now_ms: i64,
    ) -> Result<Session, AuthError> {
        // A malformed address cannot belong to an account, but must not be
        // distinguishable from a wrong password.
        let email = normalize_email(email).map_err(|_| AuthError::InvalidCredentials)?;
        if password.len() > MAX_PASSWORD_BYTES {
            return Err(AuthError::InvalidCredentials);
        }
        let row: Option<(i64, String)> = db
            .conn()
            .query_row(
                "SELECT id, password_hash FROM users WHERE email = ?1",
                [&email],
                |r| Ok((r.get(0)?, r.get(1)?)),
            )
            .optional()?;
        let (user_id, ok) = match row {
            Some((id, phc)) => (id, self.verify_password(password, &phc)),
            None => {
                let _ = self.verify_password(password, &self.dummy_hash);
                (0, false)
            }
        };
        if !ok {
            return Err(AuthError::InvalidCredentials);
        }
        let mut raw = [0u8; 32];
        getrandom::fill(&mut raw).map_err(|e| AuthError::Random(e.to_string()))?;
        let token = hex(&raw);
        let expires_at_ms =
            now_ms.saturating_add((self.cfg.session_ttl_secs as i64).saturating_mul(1000));
        db.conn().execute(
            "INSERT INTO sessions(token_hash, user_id, expires_at) VALUES (?1, ?2, ?3)",
            params![token_digest(&token), user_id, expires_at_ms],
        )?;
        Ok(Session {
            token: Secret::new(token),
            user_id,
            expires_at_ms,
        })
    }

    /// Returns the user a token belongs to, if the session exists and has not expired.
    pub fn authenticate(&self, db: &Db, token: &str, now_ms: i64) -> Result<i64, AuthError> {
        db.conn()
            .query_row(
                "SELECT user_id FROM sessions WHERE token_hash = ?1 AND expires_at > ?2",
                params![token_digest(token), now_ms],
                |r| r.get(0),
            )
            .optional()?
            .ok_or(AuthError::Unauthorized)
    }

    /// Ends one session. Returns whether it existed.
    pub fn logout(&self, db: &Db, token: &str) -> Result<bool, AuthError> {
        let n = db.conn().execute(
            "DELETE FROM sessions WHERE token_hash = ?1",
            [token_digest(token)],
        )?;
        Ok(n == 1)
    }

    /// Deletes expired sessions and returns how many there were.
    pub fn purge_expired(&self, db: &Db, now_ms: i64) -> Result<usize, AuthError> {
        Ok(db
            .conn()
            .execute("DELETE FROM sessions WHERE expires_at <= ?1", [now_ms])?)
    }

    /// Deletes a user; their sessions disappear with them (`ON DELETE CASCADE`).
    pub fn delete_user(&self, db: &Db, user_id: i64) -> Result<bool, AuthError> {
        Ok(db
            .conn()
            .execute("DELETE FROM users WHERE id = ?1", [user_id])?
            == 1)
    }
}
````

`src/jobs.rs`:

````rust
//! A job queue stored in the database.
//!
//! Delivery is **at least once**: a worker that dies after claiming a job
//! loses its lease, and another worker runs the job again. Handlers must
//! therefore be idempotent. The `attempts` column doubles as a fencing token so
//! a worker that lost its lease cannot overwrite the new owner's result.

use std::collections::BTreeMap;
use std::panic::{AssertUnwindSafe, catch_unwind};
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::thread::JoinHandle;
use std::time::Duration;

use rusqlite::{Connection, OptionalExtension, params};

use crate::config::Config;
use crate::db::{Db, DbError};
use crate::now_ms;

/// A claimed job.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Job {
    /// Row id.
    pub id: i64,
    /// Which handler should run it.
    pub kind: String,
    /// Handler input, usually JSON.
    pub payload: String,
    /// Which attempt this is (1 for the first run). Also the fencing token.
    pub attempts: u32,
    /// Attempts allowed before the job is declared dead.
    pub max_attempts: u32,
}

/// Longest stored error text, in characters.
pub const MAX_ERROR_CHARS: usize = 500;

/// Inserts a queued job that becomes runnable at `run_at_ms`. Call it with a
/// `Transaction` to make "change data and schedule follow-up work" atomic.
pub fn enqueue(
    conn: &Connection,
    kind: &str,
    payload: &str,
    run_at_ms: i64,
    max_attempts: u32,
    now_ms: i64,
) -> rusqlite::Result<i64> {
    conn.execute(
        "INSERT INTO jobs(kind, payload, state, max_attempts, run_at, created_at)
         VALUES (?1, ?2, 'queued', ?3, ?4, ?5)",
        params![kind, payload, max_attempts, run_at_ms, now_ms],
    )?;
    Ok(conn.last_insert_rowid())
}

/// Delay before retry number `attempts`: `base * 2^(attempts - 1)`, at most one hour.
pub fn backoff_ms(base_ms: u64, attempts: u32) -> u64 {
    let shift = attempts.saturating_sub(1).min(32);
    base_ms.saturating_mul(1u64 << shift).min(3_600_000)
}

/// Atomically takes the oldest runnable job and leases it for `lease_ms`.
///
/// A job is runnable if it is queued and due, or running with an expired
/// lease (its worker is presumed dead). An expired job that already used its
/// last attempt is declared dead instead of being run again.
pub fn claim(conn: &Connection, now_ms: i64, lease_ms: i64) -> rusqlite::Result<Option<Job>> {
    conn.execute(
        "UPDATE jobs SET state = 'dead', lease_until = NULL,
                last_error = 'lease expired during the final attempt'
         WHERE state = 'running' AND lease_until <= ?1 AND attempts >= max_attempts",
        params![now_ms],
    )?;
    // One statement: SQLite takes the write lock for all of it, so two
    // workers can never receive the same job.
    conn.query_row(
        "UPDATE jobs SET state = 'running', attempts = attempts + 1, lease_until = ?1 + ?2
         WHERE id = (
             SELECT id FROM jobs
             WHERE (state = 'queued' AND run_at <= ?1)
                OR (state = 'running' AND lease_until <= ?1)
             ORDER BY run_at, id LIMIT 1)
         RETURNING id, kind, payload, attempts, max_attempts",
        params![now_ms, lease_ms],
        |r| {
            Ok(Job {
                id: r.get(0)?,
                kind: r.get(1)?,
                payload: r.get(2)?,
                attempts: r.get(3)?,
                max_attempts: r.get(4)?,
            })
        },
    )
    .optional()
}

/// Marks `job` done. Returns `false` if the lease was lost (the job is no
/// longer running as this attempt), in which case nothing is changed.
pub fn complete(conn: &Connection, job: &Job) -> rusqlite::Result<bool> {
    let n = conn.execute(
        "UPDATE jobs SET state = 'done', lease_until = NULL
         WHERE id = ?1 AND state = 'running' AND attempts = ?2",
        params![job.id, job.attempts],
    )?;
    Ok(n == 1)
}

/// Records a failed attempt: re-queues with backoff, or declares the job dead
/// when it has used all its attempts. Returns `false` if the lease was lost.
pub fn fail(
    conn: &Connection,
    job: &Job,
    error: &str,
    now_ms: i64,
    backoff_base_ms: u64,
) -> rusqlite::Result<bool> {
    let error: String = error.chars().take(MAX_ERROR_CHARS).collect();
    let n = if job.attempts >= job.max_attempts {
        conn.execute(
            "UPDATE jobs SET state = 'dead', lease_until = NULL, last_error = ?3
             WHERE id = ?1 AND state = 'running' AND attempts = ?2",
            params![job.id, job.attempts, error],
        )?
    } else {
        let run_at = now_ms.saturating_add(backoff_ms(backoff_base_ms, job.attempts) as i64);
        conn.execute(
            "UPDATE jobs SET state = 'queued', lease_until = NULL, last_error = ?3, run_at = ?4
             WHERE id = ?1 AND state = 'running' AND attempts = ?2",
            params![job.id, job.attempts, error, run_at],
        )?
    };
    Ok(n == 1)
}

/// Number of jobs in each state.
pub fn counts(conn: &Connection) -> rusqlite::Result<BTreeMap<String, i64>> {
    let mut st = conn.prepare("SELECT state, count(*) FROM jobs GROUP BY state")?;
    let rows = st.query_map([], |r| Ok((r.get::<_, String>(0)?, r.get::<_, i64>(1)?)))?;
    rows.collect()
}

/// Runs one job through `handler` and records the outcome. Returns whether a
/// job was found. A panic in the handler counts as a failed attempt.
pub fn run_one<H>(
    conn: &Connection,
    cfg: &Config,
    handler: &H,
    now_ms: i64,
) -> rusqlite::Result<bool>
where
    H: Fn(&Job) -> Result<(), String>,
{
    let Some(job) = claim(conn, now_ms, cfg.jobs.lease_ms as i64)? else {
        return Ok(false);
    };
    let outcome = catch_unwind(AssertUnwindSafe(|| handler(&job)))
        .unwrap_or_else(|_| Err("handler panicked".to_string()));
    match outcome {
        Ok(()) => {
            complete(conn, &job)?;
        }
        Err(message) => {
            fail(conn, &job, &message, now_ms, cfg.jobs.backoff_base_ms)?;
        }
    }
    Ok(true)
}

/// A set of worker threads, each with its own database connection.
/// Dropping the pool asks the workers to stop and waits for them; a worker
/// finishes the job it is running first.
pub struct WorkerPool {
    stop: Arc<AtomicBool>,
    handles: Vec<JoinHandle<()>>,
}

impl WorkerPool {
    /// Starts `cfg.jobs.workers` threads running `handler`.
    pub fn start<H>(cfg: &Config, handler: H) -> Result<WorkerPool, DbError>
    where
        H: Fn(&Job) -> Result<(), String> + Send + Sync + 'static,
    {
        let stop = Arc::new(AtomicBool::new(false));
        let handler = Arc::new(handler);
        let mut handles = Vec::new();
        for _ in 0..cfg.jobs.workers {
            let db = Db::open(&cfg.db)?; // opened here so a bad path fails `start`
            let (stop, handler, cfg) = (stop.clone(), handler.clone(), cfg.clone());
            handles.push(std::thread::spawn(move || {
                while !stop.load(Ordering::Acquire) {
                    match run_one(db.conn(), &cfg, &*handler, now_ms()) {
                        Ok(true) => {}
                        // Nothing to do, or a database error (lock timeout, disk full):
                        // wait and try again. A real worker would also log the error.
                        Ok(false) | Err(_) => {
                            std::thread::sleep(Duration::from_millis(cfg.jobs.poll_ms))
                        }
                    }
                }
            }));
        }
        Ok(WorkerPool { stop, handles })
    }

    /// Stops the workers and waits for them (same as dropping the pool).
    pub fn shutdown(self) {}
}

impl Drop for WorkerPool {
    fn drop(&mut self) {
        self.stop.store(true, Ordering::Release);
        for h in self.handles.drain(..) {
            let _ = h.join();
        }
    }
}
````

`tests/common/mod.rs`:

```rust
//! Helpers shared by the integration tests.
#![allow(dead_code)]

use std::path::PathBuf;
use std::sync::atomic::{AtomicU32, Ordering};

use datalab::Config;

static NEXT: AtomicU32 = AtomicU32::new(0);

/// A unique temporary directory, removed on drop.
pub struct TempDir(pub PathBuf);

impl TempDir {
    pub fn new() -> TempDir {
        let n = NEXT.fetch_add(1, Ordering::Relaxed);
        let dir = std::env::temp_dir().join(format!("datalab-{}-{n}", std::process::id()));
        std::fs::create_dir_all(&dir).unwrap();
        TempDir(dir)
    }

    pub fn file(&self, name: &str) -> String {
        self.0.join(name).to_string_lossy().into_owned()
    }
}

impl Drop for TempDir {
    fn drop(&mut self) {
        let _ = std::fs::remove_dir_all(&self.0);
    }
}

/// A configuration for tests: a database file in `dir`, cheap password hashing,
/// fast job polling.
pub fn test_config(dir: &TempDir) -> Config {
    let mut c = Config::default();
    c.db.path = dir.file("test.db");
    c.db.busy_timeout_ms = 5_000;
    c.auth.argon2_memory_kib = 8;
    c.auth.argon2_iterations = 1;
    c.jobs.poll_ms = 2;
    c.jobs.backoff_base_ms = 10;
    c.jobs.lease_ms = 60_000;
    c
}
```

`tests/config.rs`:

````rust
use datalab::{Config, ConfigError, Secret};

fn env(pairs: &[(&str, &str)]) -> Vec<(String, String)> {
    pairs
        .iter()
        .map(|(k, v)| (k.to_string(), v.to_string()))
        .collect()
}

#[test]
fn defaults_are_valid() {
    Config::default().validate().unwrap();
    let c = Config::load(None, env(&[])).unwrap();
    assert_eq!(c.jobs.workers, 2);
    assert_eq!(c.db.busy_timeout_ms, 5_000);
    assert_eq!(c.auth.argon2_memory_kib, 19_456);
    assert_eq!(c.auth.argon2_iterations, 2);
    assert!(c.jobs.api_key.is_none());
}

#[test]
fn toml_overrides_defaults_and_missing_keys_keep_them() {
    let c = Config::load(
        Some("[jobs]\nworkers = 8\n[db]\npath = \"/var/lib/x.db\"\n"),
        env(&[]),
    )
    .unwrap();
    assert_eq!(c.jobs.workers, 8);
    assert_eq!(c.db.path, "/var/lib/x.db");
    assert_eq!(c.jobs.max_attempts, 5, "untouched key keeps its default");
}

#[test]
fn environment_beats_the_file_and_the_file_beats_defaults() {
    let toml = "[jobs]\nworkers = 8\nmax_attempts = 3\n";
    let c = Config::load(
        Some(toml),
        env(&[("DATALAB_JOBS_WORKERS", "16"), ("HOME", "/root")]),
    )
    .unwrap();
    assert_eq!(c.jobs.workers, 16, "env > file");
    assert_eq!(c.jobs.max_attempts, 3, "file > default");
    assert_eq!(c.jobs.lease_ms, 30_000, "default");
}

#[test]
fn every_environment_setting_is_wired_to_its_field() {
    let c = Config::load(
        None,
        env(&[
            ("DATALAB_DB_PATH", "/tmp/a.db"),
            ("DATALAB_DB_BUSY_TIMEOUT_MS", "11"),
            ("DATALAB_AUTH_SESSION_TTL_SECS", "3600"),
            ("DATALAB_AUTH_ARGON2_MEMORY_KIB", "64"),
            ("DATALAB_AUTH_ARGON2_ITERATIONS", "3"),
            ("DATALAB_JOBS_WORKERS", "4"),
            ("DATALAB_JOBS_MAX_ATTEMPTS", "7"),
            ("DATALAB_JOBS_LEASE_MS", "123"),
            ("DATALAB_JOBS_BACKOFF_BASE_MS", "45"),
            ("DATALAB_JOBS_POLL_MS", "67"),
            ("DATALAB_JOBS_API_KEY", "k-123"),
        ]),
    )
    .unwrap();
    assert_eq!(
        (c.db.path.as_str(), c.db.busy_timeout_ms),
        ("/tmp/a.db", 11)
    );
    assert_eq!(
        (
            c.auth.session_ttl_secs,
            c.auth.argon2_memory_kib,
            c.auth.argon2_iterations
        ),
        (3600, 64, 3)
    );
    assert_eq!(
        (c.jobs.workers, c.jobs.max_attempts, c.jobs.lease_ms),
        (4, 7, 123)
    );
    assert_eq!((c.jobs.backoff_base_ms, c.jobs.poll_ms), (45, 67));
    assert_eq!(c.jobs.api_key.as_ref().unwrap().expose(), "k-123");
}

#[test]
fn a_misspelled_datalab_variable_is_an_error_but_other_variables_are_ignored() {
    let e = Config::load(None, env(&[("DATALAB_JOBS_WORKER", "4")])).unwrap_err();
    assert!(
        matches!(&e, ConfigError::UnknownEnv(v) if v == "DATALAB_JOBS_WORKER"),
        "{e}"
    );
    Config::load(
        None,
        env(&[
            ("PATH", "/bin"),
            ("DATALABX", "1"),
            ("MY_DATALAB_JOBS", "1"),
        ]),
    )
    .unwrap();
}

#[test]
fn unparsable_environment_values_name_the_variable_but_not_the_value() {
    let e = Config::load(None, env(&[("DATALAB_JOBS_WORKERS", "12x")])).unwrap_err();
    let text = e.to_string();
    assert!(text.contains("DATALAB_JOBS_WORKERS"), "{text}");
    assert!(
        !text.contains("12x"),
        "the value must not be echoed: {text}"
    );
    for bad in ["-1", "", "1.5", "99999999999999999999"] {
        assert!(
            Config::load(None, env(&[("DATALAB_JOBS_WORKERS", bad)])).is_err(),
            "{bad:?}"
        );
    }
}

#[test]
fn toml_errors_carry_a_position_and_unknown_keys_are_rejected() {
    let e = Config::load(Some("[jobs]\nworkerz = 3\n"), env(&[])).unwrap_err();
    let text = e.to_string();
    assert!(matches!(e, ConfigError::Toml(_)));
    assert!(
        text.contains("unknown field `workerz`") && text.contains("line 2"),
        "{text}"
    );
    let e = Config::load(Some("[jobs]\nworkers = \"many\"\n"), env(&[])).unwrap_err();
    assert!(e.to_string().contains("invalid type: string"), "{e}");
    let e = Config::load(Some("[jobs\nworkers = 1\n"), env(&[])).unwrap_err();
    assert!(e.to_string().contains("line 1"), "{e}");
    assert!(
        std::error::Error::source(&e).is_some(),
        "the parser error is the source"
    );
}

fn invalid(toml: &str) -> String {
    match Config::load(Some(toml), env(&[])) {
        Err(ConfigError::Invalid { field, message }) => format!("{field}: {message}"),
        other => panic!("expected Invalid for {toml:?}, got {other:?}"),
    }
}

fn valid(toml: &str) {
    Config::load(Some(toml), env(&[])).unwrap_or_else(|e| panic!("{toml:?}: {e}"));
}

#[test]
fn every_range_is_enforced_at_both_ends() {
    let cases: &[(&str, &str, u64, u64)] = &[
        ("db", "busy_timeout_ms", 0, 60_000),
        ("auth", "session_ttl_secs", 60, 30 * 24 * 3600),
        ("auth", "argon2_memory_kib", 8, 1 << 20),
        ("auth", "argon2_iterations", 1, 10),
        ("jobs", "workers", 1, 64),
        ("jobs", "max_attempts", 1, 20),
        ("jobs", "lease_ms", 1, 3_600_000),
        ("jobs", "backoff_base_ms", 1, 60_000),
        ("jobs", "poll_ms", 1, 60_000),
    ];
    for &(table, key, lo, hi) in cases {
        valid(&format!("[{table}]\n{key} = {lo}\n"));
        valid(&format!("[{table}]\n{key} = {hi}\n"));
        if lo > 0 {
            let below = invalid(&format!("[{table}]\n{key} = {}\n", lo - 1));
            assert!(below.starts_with(&format!("{table}.{key}:")), "{below}");
        } else {
            let e = Config::load(Some(&format!("[{table}]\n{key} = -1\n")), env(&[])).unwrap_err();
            assert!(
                matches!(e, ConfigError::Toml(_)),
                "negative numbers fail at parse time: {e}"
            );
        }
        let above = invalid(&format!("[{table}]\n{key} = {}\n", hi + 1));
        assert!(
            above.starts_with(&format!("{table}.{key}:")) && above.contains("is not in"),
            "{above}"
        );
    }
    assert!(invalid("[db]\npath = \"\"\n").starts_with("db.path"));
}

#[test]
fn secrets_never_appear_in_debug_output() {
    let c = Config::load(None, env(&[("DATALAB_JOBS_API_KEY", "super-secret-value")])).unwrap();
    let text = format!("{c:?}");
    assert!(!text.contains("super-secret-value"), "{text}");
    assert!(text.contains("Secret(<redacted>)"), "{text}");
    let from_file = Config::from_toml("[jobs]\napi_key = \"another-secret\"\n").unwrap();
    assert!(!format!("{from_file:?}").contains("another-secret"));
    assert_eq!(format!("{:?}", Secret::new("x")), "Secret(<redacted>)");
}

#[test]
fn error_messages_name_the_setting() {
    let e = Config::load(None, env(&[("DATALAB_JOBS_WORKERS", "0")])).unwrap_err();
    assert_eq!(
        e.to_string(),
        "invalid setting jobs.workers: 0 is not in 1..=64"
    );
}
````

`tests/db.rs`:

````rust
mod common;

use std::sync::{Arc, Barrier};
use std::time::{Duration, Instant};

use common::{TempDir, test_config};
use datalab::db::{MIGRATIONS, migrate_with};
use datalab::{Db, DbError};
use rusqlite::{Connection, TransactionBehavior, params};

fn version(c: &Connection) -> i64 {
    c.query_row("PRAGMA user_version", [], |r| r.get(0))
        .unwrap()
}

fn tables(c: &Connection) -> Vec<String> {
    let mut st = c
        .prepare("SELECT name FROM sqlite_master WHERE type = 'table' ORDER BY name")
        .unwrap();
    st.query_map([], |r| r.get(0))
        .unwrap()
        .map(Result::unwrap)
        .collect()
}

#[test]
fn a_fresh_database_gets_every_migration() {
    let db = Db::open_in_memory().unwrap();
    assert_eq!(db.schema_version().unwrap(), MIGRATIONS.len() as i64);
    assert_eq!(tables(db.conn()), ["jobs", "sessions", "users"]);
}

#[test]
fn reopening_a_file_database_changes_nothing() {
    let dir = TempDir::new();
    let cfg = test_config(&dir);
    {
        let db = Db::open(&cfg.db).unwrap();
        db.conn()
            .execute(
                "INSERT INTO users(email,name,password_hash,created_at) VALUES ('a@b.c','A','h',1)",
                [],
            )
            .unwrap();
    }
    let db = Db::open(&cfg.db).unwrap();
    assert_eq!(db.schema_version().unwrap(), MIGRATIONS.len() as i64);
    let n: i64 = db
        .conn()
        .query_row("SELECT count(*) FROM users", [], |r| r.get(0))
        .unwrap();
    assert_eq!(n, 1, "data survives a reopen");
}

#[test]
fn file_databases_use_wal_and_enforce_foreign_keys() {
    let dir = TempDir::new();
    let db = Db::open(&test_config(&dir).db).unwrap();
    let mode: String = db
        .conn()
        .query_row("PRAGMA journal_mode", [], |r| r.get(0))
        .unwrap();
    assert_eq!(mode, "wal");
    let fk: i64 = db
        .conn()
        .query_row("PRAGMA foreign_keys", [], |r| r.get(0))
        .unwrap();
    assert_eq!(fk, 1);
    let e = db
        .conn()
        .execute(
            "INSERT INTO sessions(token_hash,user_id,expires_at) VALUES ('t', 999, 1)",
            [],
        )
        .unwrap_err();
    assert!(e.to_string().contains("FOREIGN KEY"), "{e}");
}

#[test]
fn only_pending_migrations_run() {
    let mut c = Connection::open_in_memory().unwrap();
    let steps = [
        "CREATE TABLE a(x)",
        "CREATE TABLE b(x)",
        "CREATE TABLE c(x)",
    ];
    migrate_with(&mut c, &steps[..1]).unwrap();
    assert_eq!((version(&c), tables(&c)), (1, vec!["a".to_string()]));
    migrate_with(&mut c, &steps).unwrap(); // would fail on "a" if step 1 ran again
    assert_eq!(
        (version(&c), tables(&c)),
        (3, vec!["a".into(), "b".into(), "c".into()])
    );
    migrate_with(&mut c, &steps).unwrap(); // idempotent
    assert_eq!(version(&c), 3);
}

#[test]
fn a_failing_migration_is_rolled_back_completely() {
    let mut c = Connection::open_in_memory().unwrap();
    migrate_with(&mut c, &["CREATE TABLE a(x)"]).unwrap();
    let steps = ["CREATE TABLE a(x)", "CREATE TABLE b(x); CREATE TABLE a(x)"];
    match migrate_with(&mut c, &steps) {
        Err(DbError::Migration { version: 2, source }) => {
            assert!(source.to_string().contains("already exists"))
        }
        other => panic!("{other:?}"),
    }
    assert_eq!(version(&c), 1, "the version did not advance");
    assert_eq!(
        tables(&c),
        ["a"],
        "table b was rolled back with the failed step"
    );
    let msg = migrate_with(&mut c, &steps).unwrap_err().to_string();
    assert!(
        msg.starts_with("migration 2 failed and was rolled back"),
        "{msg}"
    );
}

#[test]
fn a_database_from_the_future_is_refused() {
    let mut c = Connection::open_in_memory().unwrap();
    c.execute_batch("PRAGMA user_version = 9").unwrap();
    match migrate_with(&mut c, MIGRATIONS) {
        Err(DbError::TooNew {
            found: 9,
            supported,
        }) => assert_eq!(supported, MIGRATIONS.len()),
        other => panic!("{other:?}"),
    }
    assert_eq!(version(&c), 9, "nothing was touched");
}

#[test]
fn processes_starting_together_migrate_exactly_once() {
    let dir = TempDir::new();
    let path = dir.file("race.db");
    let barrier = Arc::new(Barrier::new(8));
    let handles: Vec<_> = (0..8)
        .map(|_| {
            let (path, barrier) = (path.clone(), barrier.clone());
            std::thread::spawn(move || {
                let mut c = Connection::open(&path).unwrap();
                c.busy_timeout(Duration::from_secs(10)).unwrap();
                barrier.wait();
                migrate_with(&mut c, MIGRATIONS)
            })
        })
        .collect();
    for h in handles {
        h.join()
            .unwrap()
            .expect("no thread may fail with 'table already exists'");
    }
    let c = Connection::open(&path).unwrap();
    assert_eq!(version(&c), MIGRATIONS.len() as i64);
}

#[test]
fn bound_parameters_store_hostile_text_verbatim() {
    let db = Db::open_in_memory().unwrap();
    let evil = "x'); DROP TABLE users; --";
    db.conn()
        .execute(
            "INSERT INTO users(email,name,password_hash,created_at) VALUES (?1,?2,'h',1)",
            params!["e@x.io", evil],
        )
        .unwrap();
    let name: String = db
        .conn()
        .query_row("SELECT name FROM users", [], |r| r.get(0))
        .unwrap();
    assert_eq!(name, evil);
    assert!(tables(db.conn()).contains(&"users".to_string()));
    let n: i64 = db
        .conn()
        .query_row(
            "SELECT count(*) FROM users WHERE name = ?1",
            ["x' OR '1'='1"],
            |r| r.get(0),
        )
        .unwrap();
    assert_eq!(n, 0);
}

#[test]
fn string_built_sql_is_injectable_documentation_of_the_bug() {
    // This is the pattern NOT to use: it exists to show why.
    let db = Db::open_in_memory().unwrap();
    for e in ["a@x.io", "b@x.io"] {
        db.conn()
            .execute(
                "INSERT INTO users(email,name,password_hash,created_at) VALUES (?1,'n','h',1)",
                [e],
            )
            .unwrap();
    }
    let evil = "nobody' OR '1'='1";
    let sql = format!("SELECT count(*) FROM users WHERE email = '{evil}'");
    let n: i64 = db.conn().query_row(&sql, [], |r| r.get(0)).unwrap();
    assert_eq!(n, 2, "the condition was rewritten by the input");
}

#[test]
fn constraints_reject_bad_rows() {
    let db = Db::open_in_memory().unwrap();
    let ins = "INSERT INTO users(email,name,password_hash,created_at) VALUES (?1,?2,'h',?3)";
    db.conn().execute(ins, params!["a@x.io", "A", 1]).unwrap();
    let dup = db
        .conn()
        .execute(ins, params!["a@x.io", "B", 1])
        .unwrap_err();
    assert!(
        dup.to_string()
            .contains("UNIQUE constraint failed: users.email"),
        "{dup}"
    );
    let null = db
        .conn()
        .execute(ins, params![rusqlite::types::Null, "B", 1])
        .unwrap_err();
    assert!(
        null.to_string().contains("NOT NULL constraint failed"),
        "{null}"
    );
    let text_in_int = db
        .conn()
        .execute(ins, params!["c@x.io", "C", "yesterday"])
        .unwrap_err();
    assert!(
        text_in_int
            .to_string()
            .contains("cannot store TEXT value in INTEGER column"),
        "{text_in_int}"
    );
    let bad_state = db
        .conn()
        .execute(
            "INSERT INTO jobs(kind,payload,state,max_attempts,run_at,created_at) VALUES ('k','p','bogus',1,0,0)",
            [],
        )
        .unwrap_err();
    assert!(
        bad_state.to_string().contains("CHECK constraint failed"),
        "{bad_state}"
    );
}

#[test]
fn a_dropped_transaction_rolls_back_and_a_committed_one_persists() {
    let dir = TempDir::new();
    let mut a = Db::open(&test_config(&dir).db).unwrap();
    let b = Db::open(&test_config(&dir).db).unwrap();
    let count = |db: &Db| -> i64 {
        db.conn()
            .query_row("SELECT count(*) FROM users", [], |r| r.get(0))
            .unwrap()
    };
    {
        let tx = a.conn_mut().transaction().unwrap();
        tx.execute(
            "INSERT INTO users(email,name,password_hash,created_at) VALUES ('t@x.io','T','h',1)",
            [],
        )
        .unwrap();
        assert_eq!(
            b_count(&b),
            0,
            "another connection does not see uncommitted rows"
        );
    } // dropped without commit
    assert_eq!(count(&a), 0);
    let tx = a.conn_mut().transaction().unwrap();
    tx.execute(
        "INSERT INTO users(email,name,password_hash,created_at) VALUES ('t@x.io','T','h',1)",
        [],
    )
    .unwrap();
    tx.commit().unwrap();
    assert_eq!(count(&b), 1);
}

fn b_count(db: &Db) -> i64 {
    db.conn()
        .query_row("SELECT count(*) FROM users", [], |r| r.get(0))
        .unwrap()
}

#[test]
fn a_second_writer_waits_for_the_busy_timeout_then_fails() {
    let dir = TempDir::new();
    let mut cfg = test_config(&dir);
    cfg.db.busy_timeout_ms = 150;
    let mut holder = Db::open(&cfg.db).unwrap();
    let other = Db::open(&cfg.db).unwrap();
    let tx = holder
        .conn_mut()
        .transaction_with_behavior(TransactionBehavior::Immediate)
        .unwrap();
    let t = Instant::now();
    let e = other
        .conn()
        .execute(
            "INSERT INTO users(email,name,password_hash,created_at) VALUES ('w@x.io','W','h',1)",
            [],
        )
        .unwrap_err();
    let waited = t.elapsed();
    assert!(e.to_string().contains("database is locked"), "{e}");
    assert!(
        waited >= Duration::from_millis(120) && waited < Duration::from_secs(3),
        "{waited:?}"
    );
    drop(tx);
    other
        .conn()
        .execute(
            "INSERT INTO users(email,name,password_hash,created_at) VALUES ('w@x.io','W','h',1)",
            [],
        )
        .expect("succeeds once the lock is released");
}

#[test]
fn the_job_queue_query_uses_the_index() {
    let db = Db::open_in_memory().unwrap();
    let plan: Vec<String> = db
        .conn()
        .prepare("EXPLAIN QUERY PLAN SELECT id FROM jobs WHERE state = 'queued' AND run_at <= 5 ORDER BY run_at, id LIMIT 1")
        .unwrap()
        .query_map([], |r| r.get::<_, String>(3))
        .unwrap()
        .map(Result::unwrap)
        .collect();
    assert!(plan.iter().any(|l| l.contains("jobs_ready")), "{plan:?}");
}
````

`tests/auth.rs`:

````rust
mod common;

use std::time::Instant;

use common::{TempDir, test_config};
use datalab::auth::{
    AuthError, Authenticator, MAX_EMAIL_BYTES, MAX_NAME_CHARS, MAX_PASSWORD_BYTES,
    MIN_PASSWORD_BYTES, WELCOME_JOB, normalize_email,
};
use datalab::{Config, Db};
use sha2::{Digest, Sha256};

const NOW: i64 = 1_700_000_000_000;

fn setup() -> (Db, Authenticator, Config) {
    let dir = TempDir::new();
    let cfg = test_config(&dir);
    let db = Db::open_in_memory().unwrap();
    let auth = Authenticator::new(&cfg.auth).unwrap();
    std::mem::forget(dir); // in-memory database: the directory is never used
    (db, auth, cfg)
}

fn count(db: &Db, table: &str) -> i64 {
    db.conn()
        .query_row(&format!("SELECT count(*) FROM {table}"), [], |r| r.get(0))
        .unwrap()
}

#[test]
fn email_normalization_table() {
    for (input, want) in [
        ("ada@example.com", "ada@example.com"),
        ("  Ada@Example.COM \n", "ada@example.com"),
        ("a+tag@sub.example.org", "a+tag@sub.example.org"),
    ] {
        assert_eq!(normalize_email(input).unwrap(), want, "{input:?}");
    }
    for bad in [
        "",
        "plain",
        "@example.com",
        "ada@",
        "ada@example",
        "ada@@example.com",
        "a@b@c.com",
        "ada@.com",
        "ada@example.",
        "a da@example.com",
        "ada@exa mple.com",
        "ada@example.com\0",
        "ada\n@example.com",
    ] {
        assert!(
            matches!(normalize_email(bad), Err(AuthError::InvalidEmail)),
            "{bad:?}"
        );
    }
}

#[test]
fn email_length_limit_is_inclusive() {
    let domain = "@x.co";
    let ok = format!("{}{domain}", "a".repeat(MAX_EMAIL_BYTES - domain.len()));
    assert_eq!(ok.len(), MAX_EMAIL_BYTES);
    assert!(normalize_email(&ok).is_ok());
    assert!(normalize_email(&format!("a{ok}")).is_err());
}

#[test]
fn register_creates_the_user_and_a_welcome_job_together() {
    let (mut db, auth, cfg) = setup();
    let id = auth
        .register(
            &mut db,
            &cfg,
            "Ada@Example.com",
            " Ada ",
            "correct horse",
            NOW,
        )
        .unwrap();
    let (email, name): (String, String) = db
        .conn()
        .query_row("SELECT email, name FROM users WHERE id = ?1", [id], |r| {
            Ok((r.get(0)?, r.get(1)?))
        })
        .unwrap();
    assert_eq!((email.as_str(), name.as_str()), ("ada@example.com", "Ada"));
    let (kind, payload, state, max, run_at): (String, String, String, u32, i64) = db
        .conn()
        .query_row(
            "SELECT kind, payload, state, max_attempts, run_at FROM jobs",
            [],
            |r| Ok((r.get(0)?, r.get(1)?, r.get(2)?, r.get(3)?, r.get(4)?)),
        )
        .unwrap();
    assert_eq!(kind, WELCOME_JOB);
    assert_eq!(payload, format!("{{\"userId\":{id}}}"));
    assert_eq!(
        (state.as_str(), max, run_at),
        ("queued", cfg.jobs.max_attempts, NOW)
    );
}

#[test]
fn a_failure_after_the_user_insert_rolls_the_user_back() {
    let (mut db, auth, cfg) = setup();
    db.conn().execute_batch("DROP TABLE jobs").unwrap();
    let e = auth
        .register(&mut db, &cfg, "a@x.io", "A", "password1", NOW)
        .unwrap_err();
    assert!(matches!(e, AuthError::Db(_)), "{e}");
    assert_eq!(
        count(&db, "users"),
        0,
        "the user insert was part of the same transaction"
    );
}

#[test]
fn duplicate_emails_are_refused_in_any_letter_case_without_a_second_job() {
    let (mut db, auth, cfg) = setup();
    auth.register(&mut db, &cfg, "ada@example.com", "Ada", "password1", NOW)
        .unwrap();
    for again in ["ada@example.com", "ADA@EXAMPLE.COM", "  Ada@Example.Com"] {
        let e = auth
            .register(&mut db, &cfg, again, "Other", "password2", NOW)
            .unwrap_err();
        assert!(matches!(e, AuthError::EmailTaken), "{again:?}: {e}");
    }
    assert_eq!((count(&db, "users"), count(&db, "jobs")), (1, 1));
}

#[test]
fn registration_validation_boundaries() {
    let (mut db, auth, cfg) = setup();
    let reg = |db: &mut Db, email: &str, name: &str, pw: &str| {
        auth.register(db, &cfg, email, name, pw, NOW)
    };
    let pw_min = "p".repeat(MIN_PASSWORD_BYTES);
    let pw_max = "p".repeat(MAX_PASSWORD_BYTES);
    assert!(reg(&mut db, "1@x.io", "n", &pw_min).is_ok());
    assert!(reg(&mut db, "2@x.io", "n", &pw_max).is_ok());
    for pw in [
        "p".repeat(MIN_PASSWORD_BYTES - 1),
        "p".repeat(MAX_PASSWORD_BYTES + 1),
        String::new(),
    ] {
        assert!(
            matches!(
                reg(&mut db, "3@x.io", "n", &pw),
                Err(AuthError::WeakPassword)
            ),
            "{} bytes",
            pw.len()
        );
    }
    let name_max = "é".repeat(MAX_NAME_CHARS);
    assert!(
        reg(&mut db, "4@x.io", &name_max, &pw_min).is_ok(),
        "100 two-byte characters"
    );
    for name in [format!("{name_max}é"), String::new(), "   ".to_string()] {
        assert!(
            matches!(
                reg(&mut db, "5@x.io", &name, &pw_min),
                Err(AuthError::InvalidName)
            ),
            "{name:?}"
        );
    }
    assert!(matches!(
        reg(&mut db, "bad", "n", &pw_min),
        Err(AuthError::InvalidEmail)
    ));
    assert_eq!(
        count(&db, "users"),
        3,
        "only the three accepted requests stored anything"
    );
    assert_eq!(count(&db, "jobs"), 3);
}

#[test]
fn passwords_are_stored_as_salted_argon2id_hashes() {
    let (mut db, auth, cfg) = setup();
    auth.register(&mut db, &cfg, "a@x.io", "A", "same password", NOW)
        .unwrap();
    auth.register(&mut db, &cfg, "b@x.io", "B", "same password", NOW)
        .unwrap();
    let hashes: Vec<String> = db
        .conn()
        .prepare("SELECT password_hash FROM users ORDER BY id")
        .unwrap()
        .query_map([], |r| r.get(0))
        .unwrap()
        .map(Result::unwrap)
        .collect();
    assert!(
        hashes
            .iter()
            .all(|h| h.starts_with("$argon2id$v=19$m=8,t=1,p=1$")),
        "{hashes:?}"
    );
    assert!(hashes.iter().all(|h| !h.contains("same password")));
    assert_ne!(
        hashes[0], hashes[1],
        "a random salt makes equal passwords hash differently"
    );
    assert!(auth.verify_password("same password", &hashes[0]));
    assert!(!auth.verify_password("same passworD", &hashes[0]));
    assert!(
        !auth.verify_password("same password", "not a hash"),
        "a malformed stored hash is a mismatch"
    );
    assert!(!auth.verify_password("same password", ""));
}

#[test]
fn login_issues_a_random_token_and_stores_only_its_digest() {
    let (mut db, auth, cfg) = setup();
    let id = auth
        .register(&mut db, &cfg, "a@x.io", "A", "password1", NOW)
        .unwrap();
    let s1 = auth.login(&db, "A@X.IO", "password1", NOW).unwrap();
    let s2 = auth.login(&db, "a@x.io", "password1", NOW).unwrap();
    let t1 = s1.token.expose().to_string();
    assert_eq!(t1.len(), 64);
    assert!(t1.chars().all(|c| c.is_ascii_hexdigit()));
    assert_ne!(t1, s2.token.expose(), "every login gets a fresh token");
    assert_eq!(
        (s1.user_id, s1.expires_at_ms),
        (id, NOW + 7 * 24 * 3600 * 1000)
    );
    let stored: Vec<String> = db
        .conn()
        .prepare("SELECT token_hash FROM sessions")
        .unwrap()
        .query_map([], |r| r.get(0))
        .unwrap()
        .map(Result::unwrap)
        .collect();
    let want: String = Sha256::digest(t1.as_bytes())
        .iter()
        .map(|b| format!("{b:02x}"))
        .collect();
    assert!(stored.contains(&want));
    assert!(!stored.contains(&t1), "the raw token is never stored");
    assert!(
        !format!("{s1:?}").contains(&t1),
        "Debug output hides the token"
    );
}

#[test]
fn all_login_failures_look_the_same() {
    let (mut db, auth, cfg) = setup();
    auth.register(&mut db, &cfg, "a@x.io", "A", "password1", NOW)
        .unwrap();
    let long = "p".repeat(MAX_PASSWORD_BYTES + 1);
    let cases = [
        ("a@x.io", "wrong-password"),
        ("nobody@x.io", "password1"),
        ("not an email", "password1"),
        ("a@x.io", long.as_str()),
        ("", ""),
    ];
    let messages: Vec<String> = cases
        .iter()
        .map(|(e, p)| {
            let err = auth.login(&db, e, p, NOW).unwrap_err();
            assert!(matches!(err, AuthError::InvalidCredentials), "{e:?}");
            err.to_string()
        })
        .collect();
    assert!(
        messages.iter().all(|m| m == "invalid e-mail or password"),
        "{messages:?}"
    );
    assert_eq!(count(&db, "sessions"), 0);
}

#[test]
fn an_unknown_email_costs_about_as_much_as_a_wrong_password() {
    let dir = TempDir::new();
    let mut cfg = test_config(&dir);
    cfg.auth.argon2_memory_kib = 2048; // expensive enough that skipping the hash is obvious
    cfg.auth.argon2_iterations = 2;
    let auth = Authenticator::new(&cfg.auth).unwrap();
    let mut db = Db::open_in_memory().unwrap();
    auth.register(&mut db, &cfg, "a@x.io", "A", "password1", NOW)
        .unwrap();
    let best = |email: &str| {
        (0..3)
            .map(|_| {
                let t = Instant::now();
                let _ = auth.login(&db, email, "wrong-password", NOW);
                t.elapsed()
            })
            .min()
            .unwrap()
    };
    let (known, unknown) = (best("a@x.io"), best("nobody@x.io"));
    assert!(
        unknown * 3 >= known,
        "unknown {unknown:?} vs known {known:?}: user enumeration by timing"
    );
}

#[test]
fn sessions_expire_exactly_at_their_deadline() {
    let (mut db, auth, cfg) = setup();
    let id = auth
        .register(&mut db, &cfg, "a@x.io", "A", "password1", NOW)
        .unwrap();
    let s = auth.login(&db, "a@x.io", "password1", NOW).unwrap();
    let tok = s.token.expose();
    assert_eq!(auth.authenticate(&db, tok, NOW).unwrap(), id);
    assert_eq!(
        auth.authenticate(&db, tok, s.expires_at_ms - 1).unwrap(),
        id
    );
    for at in [s.expires_at_ms, s.expires_at_ms + 1] {
        assert!(
            matches!(
                auth.authenticate(&db, tok, at),
                Err(AuthError::Unauthorized)
            ),
            "at {at}"
        );
    }
}

#[test]
fn only_the_exact_token_authenticates() {
    let (mut db, auth, cfg) = setup();
    auth.register(&mut db, &cfg, "a@x.io", "A", "password1", NOW)
        .unwrap();
    let s = auth.login(&db, "a@x.io", "password1", NOW).unwrap();
    let tok = s.token.expose().to_string();
    let mut flipped = tok.clone();
    flipped.replace_range(0..1, if tok.starts_with('0') { "1" } else { "0" });
    for bad in [
        "",
        "x",
        &flipped,
        &tok[1..],
        &format!("{tok}0"),
        &tok.to_uppercase(),
        &format!(" {tok}"),
    ] {
        let r = auth.authenticate(&db, bad, NOW);
        assert!(matches!(r, Err(AuthError::Unauthorized)), "{bad:?}");
    }
    assert!(auth.authenticate(&db, &tok, NOW).is_ok());
}

#[test]
fn logout_purge_and_delete_user() {
    let (mut db, auth, cfg) = setup();
    let id = auth
        .register(&mut db, &cfg, "a@x.io", "A", "password1", NOW)
        .unwrap();
    let keep = auth.login(&db, "a@x.io", "password1", NOW).unwrap();
    let gone = auth.login(&db, "a@x.io", "password1", NOW).unwrap();
    assert!(auth.logout(&db, gone.token.expose()).unwrap());
    assert!(
        !auth.logout(&db, gone.token.expose()).unwrap(),
        "second logout finds nothing"
    );
    assert!(auth.authenticate(&db, gone.token.expose(), NOW).is_err());
    assert!(auth.authenticate(&db, keep.token.expose(), NOW).is_ok());

    assert_eq!(auth.purge_expired(&db, NOW).unwrap(), 0);
    assert_eq!(
        auth.purge_expired(&db, keep.expires_at_ms).unwrap(),
        1,
        "expiry instant counts as expired"
    );
    assert_eq!(count(&db, "sessions"), 0);

    auth.login(&db, "a@x.io", "password1", NOW).unwrap();
    assert_eq!(count(&db, "sessions"), 1);
    assert!(auth.delete_user(&db, id).unwrap());
    assert_eq!(
        count(&db, "sessions"),
        0,
        "ON DELETE CASCADE removed the session"
    );
    assert!(!auth.delete_user(&db, id).unwrap());
}

#[test]
fn the_hasher_rejects_impossible_parameters() {
    let dir = TempDir::new();
    let mut cfg = test_config(&dir);
    cfg.auth.argon2_memory_kib = 1; // below the algorithm's minimum
    assert!(matches!(
        Authenticator::new(&cfg.auth),
        Err(AuthError::Hashing(_))
    ));
}
````

`tests/jobs.rs`:

````rust
mod common;

use std::collections::BTreeSet;
use std::sync::atomic::{AtomicBool, AtomicU32, Ordering};
use std::sync::{Arc, Mutex};
use std::time::{Duration, Instant};

use common::{TempDir, test_config};
use datalab::jobs::{
    self, Job, MAX_ERROR_CHARS, WorkerPool, backoff_ms, claim, complete, counts, enqueue, fail,
    run_one,
};
use datalab::{Config, Db};
use rusqlite::{Connection, params};

const LEASE: i64 = 1_000;

fn mem() -> Db {
    Db::open_in_memory().unwrap()
}

fn add(c: &Connection, kind: &str, run_at: i64, max: u32) -> i64 {
    enqueue(c, kind, "{}", run_at, max, 0).unwrap()
}

fn state(c: &Connection, id: i64) -> (String, u32, Option<String>, i64) {
    c.query_row(
        "SELECT state, attempts, last_error, run_at FROM jobs WHERE id = ?1",
        [id],
        |r| Ok((r.get(0)?, r.get(1)?, r.get(2)?, r.get(3)?)),
    )
    .unwrap()
}

#[test]
fn among_due_jobs_the_earliest_run_at_goes_first_even_with_a_higher_id() {
    let db = mem();
    let c = db.conn();
    let _a = add(c, "a", 300, 3);
    let b = add(c, "b", 200, 3);
    assert_eq!(claim(c, 300, LEASE).unwrap().unwrap().id, b);
}

#[test]
fn claims_follow_run_at_then_id_and_wait_until_due() {
    let db = mem();
    let c = db.conn();
    let late = add(c, "late", 500, 3);
    let first = add(c, "first", 100, 3);
    let second = add(c, "second", 100, 3);
    assert_eq!(claim(c, 99, LEASE).unwrap(), None, "nothing is due yet");
    assert_eq!(claim(c, 100, LEASE).unwrap().unwrap().id, first);
    assert_eq!(claim(c, 100, LEASE).unwrap().unwrap().id, second);
    assert_eq!(claim(c, 499, LEASE).unwrap(), None);
    let j = claim(c, 500, LEASE).unwrap().unwrap();
    assert_eq!(
        (j.id, j.kind.as_str(), j.attempts, j.max_attempts),
        (late, "late", 1, 3)
    );
    assert_eq!(
        claim(c, 600, LEASE).unwrap(),
        None,
        "running jobs with a live lease are not handed out"
    );
}

#[test]
fn complete_marks_done_once() {
    let db = mem();
    let id = add(db.conn(), "k", 0, 3);
    let job = claim(db.conn(), 0, LEASE).unwrap().unwrap();
    assert!(complete(db.conn(), &job).unwrap());
    assert_eq!(state(db.conn(), id).0, "done");
    assert!(
        !complete(db.conn(), &job).unwrap(),
        "a finished job cannot be finished again"
    );
    assert_eq!(claim(db.conn(), 99_999, LEASE).unwrap(), None);
}

#[test]
fn a_failed_attempt_is_requeued_with_exponential_backoff() {
    let db = mem();
    let c = db.conn();
    let id = add(c, "k", 0, 4);
    let mut now = 1_000;
    for (attempt, delay) in [(1u32, 100i64), (2, 200), (3, 400)] {
        let job = claim(c, now, LEASE).unwrap().unwrap();
        assert_eq!(job.attempts, attempt);
        assert!(fail(c, &job, "boom", now, 100).unwrap());
        let (st, att, err, run_at) = state(c, id);
        assert_eq!(
            (st.as_str(), att, err.as_deref(), run_at),
            ("queued", attempt, Some("boom"), now + delay)
        );
        assert_eq!(
            claim(c, now + delay - 1, LEASE).unwrap(),
            None,
            "not before the backoff elapsed"
        );
        now += delay;
    }
    let last = claim(c, now, LEASE).unwrap().unwrap();
    assert_eq!(last.attempts, 4);
    assert!(fail(c, &last, "final", now, 100).unwrap());
    let (st, att, err, _) = state(c, id);
    assert_eq!(
        (st.as_str(), att, err.as_deref()),
        ("dead", 4, Some("final"))
    );
    assert_eq!(
        claim(c, i64::MAX / 2, LEASE).unwrap(),
        None,
        "dead jobs never run again"
    );
}

#[test]
fn long_error_messages_are_truncated_by_characters() {
    let db = mem();
    let id = add(db.conn(), "k", 0, 1);
    let job = claim(db.conn(), 0, LEASE).unwrap().unwrap();
    let long = "é".repeat(MAX_ERROR_CHARS + 50);
    fail(db.conn(), &job, &long, 0, 10).unwrap();
    let stored = state(db.conn(), id).2.unwrap();
    assert_eq!(stored.chars().count(), MAX_ERROR_CHARS);
}

#[test]
fn backoff_table_cap_and_overflow() {
    let ms: Vec<u64> = (1..=6).map(|a| backoff_ms(1_000, a)).collect();
    assert_eq!(ms, [1_000, 2_000, 4_000, 8_000, 16_000, 32_000]);
    assert_eq!(
        backoff_ms(1_000, 0),
        1_000,
        "attempt 0 behaves like attempt 1"
    );
    assert_eq!(backoff_ms(1_000, 12), 2_048_000);
    assert_eq!(backoff_ms(1_000, 13), 3_600_000, "capped at one hour");
    assert_eq!(backoff_ms(1_000, u32::MAX), 3_600_000);
    assert_eq!(backoff_ms(u64::MAX, 40), 3_600_000);
    assert_eq!(backoff_ms(60_000, 1), 60_000);
}

#[test]
fn a_dead_workers_job_is_redelivered_after_its_lease_expires() {
    let db = mem();
    let c = db.conn();
    let id = add(c, "k", 0, 3);
    let first = claim(c, 0, LEASE).unwrap().unwrap(); // the worker "crashes" here
    assert_eq!(
        claim(c, LEASE - 1, LEASE).unwrap(),
        None,
        "lease still valid"
    );
    let again = claim(c, LEASE, LEASE)
        .unwrap()
        .expect("lease expired exactly now");
    assert_eq!(
        (again.id, again.attempts),
        (id, 2),
        "same job, second attempt"
    );
    // fencing: the slow original worker finally reports back
    assert!(!complete(c, &first).unwrap(), "stale completion is ignored");
    assert!(
        !fail(c, &first, "late failure", LEASE, 10).unwrap(),
        "stale failure is ignored"
    );
    assert_eq!(state(c, id).0, "running");
    assert!(complete(c, &again).unwrap());
    assert_eq!(state(c, id).0, "done");
}

#[test]
fn a_lease_that_expires_on_the_final_attempt_kills_the_job() {
    let db = mem();
    let c = db.conn();
    let id = add(c, "k", 0, 2);
    claim(c, 0, LEASE).unwrap().unwrap();
    claim(c, LEASE, LEASE).unwrap().unwrap(); // attempt 2 of 2, then crash again
    assert_eq!(
        claim(c, 2 * LEASE, LEASE).unwrap(),
        None,
        "no third attempt"
    );
    let (st, att, err, _) = state(c, id);
    assert_eq!((st.as_str(), att), ("dead", 2));
    assert!(err.unwrap().contains("lease expired"));
}

#[test]
fn counts_group_by_state() {
    let db = mem();
    let c = db.conn();
    let a = add(c, "a", 0, 1);
    add(c, "b", 0, 1);
    add(c, "c", 1_000_000, 1);
    let j = claim(c, 0, LEASE).unwrap().unwrap();
    assert_eq!(j.id, a);
    complete(c, &j).unwrap();
    let n = counts(c).unwrap();
    assert_eq!(
        (n.get("done"), n.get("queued"), n.get("dead")),
        (Some(&1), Some(&2), None)
    );
}

#[test]
fn enqueue_inside_a_rolled_back_transaction_leaves_no_job() {
    let mut db = mem();
    {
        let tx = db.conn_mut().transaction().unwrap();
        enqueue(&tx, "k", "{}", 0, 3, 0).unwrap();
    } // dropped: rolled back, together with whatever else the transaction did
    assert!(counts(db.conn()).unwrap().is_empty());
    let tx = db.conn_mut().transaction().unwrap();
    enqueue(&tx, "k", "{}", 0, 3, 0).unwrap();
    tx.commit().unwrap();
    assert_eq!(counts(db.conn()).unwrap().get("queued"), Some(&1));
}

#[test]
fn run_one_records_success_failure_and_panic() {
    let db = mem();
    let c = db.conn();
    let mut cfg = Config::default();
    cfg.jobs.backoff_base_ms = 50;
    let ok = add(c, "ok", 0, 3);
    let err = add(c, "err", 0, 3);
    let boom = add(c, "boom", 0, 3);
    let handler = |j: &Job| match j.kind.as_str() {
        "ok" => Ok(()),
        "err" => Err("bad input".to_string()),
        _ => panic!("handler bug"),
    };
    for _ in 0..3 {
        assert!(run_one(c, &cfg, &handler, 0).unwrap());
    }
    assert!(
        !run_one(c, &cfg, &handler, 0).unwrap(),
        "failed jobs wait for their backoff"
    );
    assert_eq!(state(c, ok).0, "done");
    let (st, att, e, run_at) = state(c, err);
    assert_eq!(
        (st.as_str(), att, e.as_deref(), run_at),
        ("queued", 1, Some("bad input"), 50)
    );
    let (st, _, e, _) = state(c, boom);
    assert_eq!(
        (st.as_str(), e.as_deref()),
        ("queued", Some("handler panicked"))
    );
}

#[test]
fn concurrent_workers_never_receive_the_same_job() {
    let dir = TempDir::new();
    let cfg = test_config(&dir);
    {
        let db = Db::open(&cfg.db).unwrap();
        let tx = db.conn().unchecked_transaction().unwrap();
        for i in 0..400 {
            enqueue(&tx, "k", &i.to_string(), 0, 3, 0).unwrap();
        }
        tx.commit().unwrap();
    }
    let claimed = Arc::new(Mutex::new(Vec::new()));
    let handles: Vec<_> = (0..8)
        .map(|_| {
            let (cfg, claimed) = (cfg.clone(), claimed.clone());
            std::thread::spawn(move || {
                let db = Db::open(&cfg.db).unwrap();
                while let Some(job) = claim(db.conn(), 1, 60_000).unwrap() {
                    claimed.lock().unwrap().push(job.id);
                }
            })
        })
        .collect();
    for h in handles {
        h.join().unwrap();
    }
    let ids = claimed.lock().unwrap();
    let unique: BTreeSet<i64> = ids.iter().copied().collect();
    assert_eq!(
        (ids.len(), unique.len()),
        (400, 400),
        "every job exactly once"
    );
}

fn wait_until(what: &str, mut cond: impl FnMut() -> bool) {
    let t = Instant::now();
    while !cond() {
        assert!(
            t.elapsed() < Duration::from_secs(20),
            "timed out waiting for {what}"
        );
        std::thread::sleep(Duration::from_millis(5));
    }
}

#[test]
fn a_worker_pool_processes_every_job_once() {
    let dir = TempDir::new();
    let mut cfg = test_config(&dir);
    cfg.jobs.workers = 4;
    let db = Db::open(&cfg.db).unwrap();
    let tx = db.conn().unchecked_transaction().unwrap();
    for i in 0..100 {
        enqueue(&tx, "k", &i.to_string(), 0, 3, 0).unwrap();
    }
    tx.commit().unwrap();
    let seen = Arc::new(Mutex::new(Vec::new()));
    let s = seen.clone();
    let pool = WorkerPool::start(&cfg, move |j| {
        s.lock().unwrap().push(j.id);
        Ok(())
    })
    .unwrap();
    wait_until("all jobs done", || {
        counts(db.conn()).unwrap().get("done") == Some(&100)
    });
    drop(pool);
    let seen = seen.lock().unwrap();
    let unique: BTreeSet<i64> = seen.iter().copied().collect();
    assert_eq!((seen.len(), unique.len()), (100, 100));
}

#[test]
fn the_configured_number_of_workers_really_runs_in_parallel() {
    let dir = TempDir::new();
    let mut cfg = test_config(&dir);
    cfg.jobs.workers = 3;
    let db = Db::open(&cfg.db).unwrap();
    for _ in 0..3 {
        add(db.conn(), "slow", 0, 3);
    }
    let (running, peak) = (Arc::new(AtomicU32::new(0)), Arc::new(AtomicU32::new(0)));
    let (r, p) = (running.clone(), peak.clone());
    let pool = WorkerPool::start(&cfg, move |_| {
        let now = r.fetch_add(1, Ordering::SeqCst) + 1;
        p.fetch_max(now, Ordering::SeqCst);
        std::thread::sleep(Duration::from_millis(200));
        r.fetch_sub(1, Ordering::SeqCst);
        Ok(())
    })
    .unwrap();
    wait_until("all jobs done", || {
        counts(db.conn()).unwrap().get("done") == Some(&3)
    });
    drop(pool);
    assert!(peak.load(Ordering::SeqCst) >= 2, "jobs never overlapped");
}

#[test]
fn failed_jobs_are_retried_by_the_pool_until_they_succeed() {
    let dir = TempDir::new();
    let mut cfg = test_config(&dir);
    cfg.jobs.workers = 2;
    cfg.jobs.backoff_base_ms = 1;
    let db = Db::open(&cfg.db).unwrap();
    let ids: Vec<i64> = (0..10).map(|_| add(db.conn(), "flaky", 0, 5)).collect();
    let pool = WorkerPool::start(&cfg, |j| {
        if j.attempts < 3 {
            Err(format!("attempt {} failed", j.attempts))
        } else {
            Ok(())
        }
    })
    .unwrap();
    wait_until("all jobs done", || {
        counts(db.conn()).unwrap().get("done") == Some(&10)
    });
    drop(pool);
    for id in ids {
        let (st, att, err, _) = state(db.conn(), id);
        assert_eq!((st.as_str(), att), ("done", 3));
        assert_eq!(
            err.as_deref(),
            Some("attempt 2 failed"),
            "the last error stays visible"
        );
    }
}

#[test]
fn dropping_the_pool_lets_the_running_job_finish() {
    let dir = TempDir::new();
    let mut cfg = test_config(&dir);
    cfg.jobs.workers = 1;
    let db = Db::open(&cfg.db).unwrap();
    let id = add(db.conn(), "slow", 0, 3);
    let started = Arc::new(AtomicBool::new(false));
    let s = started.clone();
    let pool = WorkerPool::start(&cfg, move |_| {
        s.store(true, Ordering::SeqCst);
        std::thread::sleep(Duration::from_millis(300));
        Ok(())
    })
    .unwrap();
    wait_until("the handler to start", || started.load(Ordering::SeqCst));
    let t = Instant::now();
    pool.shutdown();
    assert!(
        t.elapsed() >= Duration::from_millis(150),
        "shutdown waited for the job"
    );
    assert_eq!(
        state(db.conn(), id).0,
        "done",
        "the job finished before the workers exited"
    );
}

#[test]
fn an_idle_pool_shuts_down_promptly() {
    let dir = TempDir::new();
    let cfg = test_config(&dir);
    let pool = WorkerPool::start(&cfg, |_| Ok(())).unwrap();
    let t = Instant::now();
    pool.shutdown();
    assert!(t.elapsed() < Duration::from_secs(2), "{:?}", t.elapsed());
}

#[test]
fn starting_a_pool_on_an_unusable_path_fails_immediately() {
    let mut cfg = Config::default();
    cfg.db.path = "/nonexistent-dir-for-datalab/x.db".into();
    assert!(WorkerPool::start(&cfg, |_| Ok(())).is_err());
}

#[test]
fn duplicate_delivery_is_harmless_when_the_effect_is_idempotent() {
    // The same job runs twice (lease expiry). The effect is keyed by the
    // business identity, so the second run changes nothing.
    let db = mem();
    let c = db.conn();
    c.execute_batch("CREATE TABLE welcome_sent(user_id INTEGER PRIMARY KEY)")
        .unwrap();
    add(c, "welcome", 0, 3);
    let runs = AtomicU32::new(0);
    let handler = |_: &Job| -> Result<(), String> {
        runs.fetch_add(1, Ordering::SeqCst);
        c.execute(
            "INSERT OR IGNORE INTO welcome_sent(user_id) VALUES (?1)",
            params![42],
        )
        .map_err(|e| e.to_string())?;
        Ok(())
    };
    let first = claim(c, 0, LEASE).unwrap().unwrap();
    handler(&first).unwrap(); // effect happens, worker dies before complete()
    let second = claim(c, LEASE, LEASE).unwrap().unwrap();
    handler(&second).unwrap();
    assert!(complete(c, &second).unwrap());
    let sent: i64 = c
        .query_row("SELECT count(*) FROM welcome_sent", [], |r| r.get(0))
        .unwrap();
    assert_eq!(
        (runs.load(Ordering::SeqCst), sent),
        (2, 1),
        "ran twice, effect applied once"
    );
}

#[test]
fn the_module_is_reachable_by_its_documented_path() {
    // keeps the `jobs::` re-export honest
    assert_eq!(jobs::backoff_ms(5, 1), 5);
}
````

`examples/bench.rs`:

````rust
//! Rough costs of the pieces. Single measurements on a shared machine.
use std::time::{Duration, Instant};

use datalab::auth::Authenticator;
use datalab::jobs::{WorkerPool, counts, enqueue};
use datalab::{Config, Db};

fn median(mut v: Vec<Duration>) -> Duration {
    v.sort();
    v[v.len() / 2]
}

fn main() {
    let dir = std::env::temp_dir().join(format!("datalab-bench-{}", std::process::id()));
    std::fs::create_dir_all(&dir).unwrap();
    let mut cfg = Config::default();
    cfg.db.path = dir.join("bench.db").to_string_lossy().into_owned();
    cfg.jobs.poll_ms = 1;

    // --- authentication, default Argon2 parameters ---
    let auth = Authenticator::new(&cfg.auth).unwrap();
    let mut db = Db::open(&cfg.db).unwrap();
    let t = Instant::now();
    auth.register(&mut db, &cfg, "ada@example.com", "Ada", "correct horse", 0)
        .unwrap();
    println!(
        "register (hash + insert, includes first-use page faults): {:?}",
        t.elapsed()
    );
    let time = |email: &str, pw: &str| {
        median(
            (0..9)
                .map(|_| {
                    let t = Instant::now();
                    let _ = auth.login(&db, email, pw, 0);
                    t.elapsed()
                })
                .collect(),
        )
    };
    println!(
        "login, right password:      median {:?}",
        time("ada@example.com", "correct horse")
    );
    println!(
        "login, wrong password:      median {:?}",
        time("ada@example.com", "wrong password")
    );
    println!(
        "login, unknown e-mail:      median {:?}",
        time("nobody@example.com", "wrong password")
    );
    let session = auth
        .login(&db, "ada@example.com", "correct horse", 0)
        .unwrap();
    let tok = session.token.expose().to_string();
    let n = 20_000;
    let t = Instant::now();
    for _ in 0..n {
        auth.authenticate(&db, &tok, 0).unwrap();
    }
    println!(
        "authenticate (sha256 + indexed lookup): {:?} per call",
        t.elapsed() / n
    );

    // --- job queue ---
    let jobs_db = Db::open(&cfg.db).unwrap();
    let c = jobs_db.conn();
    let n = 1_000u32;
    let t = Instant::now();
    for i in 0..n {
        enqueue(c, "k", &i.to_string(), 0, 3, 0).unwrap();
    }
    let one_by_one = t.elapsed();
    let n2 = 20_000u32;
    let t = Instant::now();
    let tx = c.unchecked_transaction().unwrap();
    for i in 0..n2 {
        enqueue(&tx, "k", &i.to_string(), 0, 3, 0).unwrap();
    }
    tx.commit().unwrap();
    let batched = t.elapsed();
    println!(
        "enqueue, one commit each:   {:?} per job ({n} jobs)",
        one_by_one / n
    );
    println!(
        "enqueue, one transaction:   {:?} per job ({n2} jobs)",
        batched / n2
    );
    c.execute("DELETE FROM jobs", []).unwrap();

    for workers in [1u32, 2, 4] {
        cfg.jobs.workers = workers;
        let total = 2_000;
        let tx = c.unchecked_transaction().unwrap();
        for i in 0..total {
            enqueue(&tx, "k", &i.to_string(), 0, 3, 0).unwrap();
        }
        tx.commit().unwrap();
        let t = Instant::now();
        let pool = WorkerPool::start(&cfg, |_| Ok(())).unwrap();
        while counts(c).unwrap().get("done").copied().unwrap_or(0) < total {
            std::thread::sleep(Duration::from_millis(2));
        }
        let el = t.elapsed();
        drop(pool);
        println!(
            "{workers} worker(s), empty handler: {total} jobs in {el:?} = {:.0} jobs/s",
            total as f64 / el.as_secs_f64()
        );
        c.execute("DELETE FROM jobs", []).unwrap();
    }
    std::fs::remove_dir_all(&dir).ok();
}
````

✓ Checks I ran: `cargo fmt`, `cargo clippy --all-targets -- -D warnings` (clean), and `cargo test --locked`: `tests/config.rs` 10 tests, `tests/db.rs` 13, `tests/auth.rs` 14, `tests/jobs.rs` 20 (57 in all), plus 15 consecutive full runs without a failure.

**Mutation check ✓.** I made 57 one-line changes to the crate (a "mutant"), each in its own copy, and ran the whole suite against each: **55 were caught, 2 are equivalent (M16 and M32 below), none survive.** Two mutants (M43, M57) survived the *first* run, which showed real gaps: no test required `ORDER BY run_at` to beat id order, and none required more than one worker thread; I added a test for each and re-ran those two (both now caught). Three mutant patterns I had first written did not match the source text (they were reported as invalid, not as caught); I rewrote them and re-ran them. The hang mutant (M54) never finishes shutting down, so I count a timeout as detection. The tests ran in dev profile; the table lists the first failing test.

| # | module | change | result |
|---|---|---|---|
| M1 | `config` | swap `JOBS_LEASE_MS` target with `poll_ms` | caught by `every_environment_setting_is_wired_to_its_field` |
| M2 | `config` | a variable without the `DATALAB_` prefix is an error instead of ignored | caught by `a_misspelled_datalab_variable_is_an_error_but_other_variables_are_ignored` |
| M3 | `config` | unknown `DATALAB_*` variable silently ignored | caught by `a_misspelled_datalab_variable_is_an_error_but_other_variables_are_ignored` |
| M4 | `config` | range check `<=` on the low end | caught by `every_range_is_enforced_at_both_ends` |
| M5 | `config` | range check `>=` on the high end | caught by `every_range_is_enforced_at_both_ends` |
| M6 | `config` | `workers` upper bound 65 | caught by `error_messages_name_the_setting` |
| M7 | `config` | session TTL range check removed | caught by `every_range_is_enforced_at_both_ends` |
| M8 | `config` | environment layer skipped in `load` | caught by `a_misspelled_datalab_variable_is_an_error_but_other_variables_are_ignored` |
| M9 | `config` | `Secret` `Debug` prints the value | caught by `login_issues_a_random_token_and_stores_only_its_digest` |
| M10 | `config` | default busy timeout 4000 instead of 5000 | caught by `defaults_are_valid` |
| M11 | `config` | `deny_unknown_fields` removed on `JobsConfig` | caught by `toml_errors_carry_a_position_and_unknown_keys_are_rejected` |
| M12 | `config` | env parse error echoes the value | caught by `unparsable_environment_values_name_the_variable_but_not_the_value` |
| M13 | `db` | version re-check inside the lock removed | caught by `a_failing_migration_is_rolled_back_completely` |
| M14 | `db` | `TooNew` check `>=` instead of `>` | caught by `a_dropped_transaction_rolls_back_and_a_committed_one_persists` |
| M15 | `db` | `DEFERRED` instead of `IMMEDIATE` for migrations | caught by `processes_starting_together_migrate_exactly_once` |
| M16 | `db` | `PRAGMA foreign_keys = ON` removed from `open` | **equivalent here**: the bundled build already enforces foreign keys, so behavior is identical |
| M17 | `db` | busy timeout forced to 0 | caught by `a_second_writer_waits_for_the_busy_timeout_then_fails` |
| M18 | `db` | WAL pragma removed | caught by `file_databases_use_wal_and_enforce_foreign_keys` |
| M19 | `db` | `PRAGMA user_version` not written | caught by `a_failure_after_the_user_insert_rolls_the_user_back` |
| M20 | `db` | `ON DELETE CASCADE` removed | caught by `logout_purge_and_delete_user` |
| M21 | `db` | `STRICT` removed from `users` | caught by `constraints_reject_bad_rows` |
| M22 | `db` | `CHECK` on `jobs.state` removed | caught by `a_failure_after_the_user_insert_rolls_the_user_back` |
| M23 | `db` | `jobs_ready` index removed | caught by `the_job_queue_query_uses_the_index` |
| M24 | `db` | `UNIQUE` on `users.email` removed | caught by `duplicate_emails_are_refused_in_any_letter_case_without_a_second_job` |
| M25 | `auth` | password range excludes the maximum | caught by `registration_validation_boundaries` |
| M26 | `auth` | name length counted in bytes, not characters | caught by `registration_validation_boundaries` |
| M27 | `auth` | name not trimmed | caught by `register_creates_the_user_and_a_welcome_job_together` |
| M28 | `auth` | unique violation not recognized (`false`) | caught by `duplicate_emails_are_refused_in_any_letter_case_without_a_second_job` |
| M29 | `auth` | welcome job not enqueued | caught by `a_failure_after_the_user_insert_rolls_the_user_back` |
| M30 | `auth` | dummy verification removed (unknown e-mail returns fast) | caught by `an_unknown_email_costs_about_as_much_as_a_wrong_password` |
| M31 | `auth` | wrong password no longer rejected (`if false`) | caught by `all_login_failures_look_the_same` |
| M32 | `auth` | over-long password guard removed in `login` | **equivalent**: an over-long password still fails verification (only the cost of hashing differs, and no test measures it) |
| M33 | `auth` | e-mail shape error propagated from `login` | caught by `all_login_failures_look_the_same` |
| M34 | `auth` | session valid at `expires_at` (`>=`) | caught by `sessions_expire_exactly_at_their_deadline` |
| M35 | `auth` | raw token stored instead of its digest | caught by `login_issues_a_random_token_and_stores_only_its_digest` |
| M36 | `auth` | TTL multiplied by 100 instead of 1000 | caught by `login_issues_a_random_token_and_stores_only_its_digest` |
| M37 | `auth` | e-mail not lower-cased | caught by `duplicate_emails_are_refused_in_any_letter_case_without_a_second_job` |
| M38 | `auth` | `purge_expired` uses `<` (misses the boundary) | caught by `logout_purge_and_delete_user` |
| M39 | `auth` | `logout` always returns `true` | caught by `logout_purge_and_delete_user` |
| M40 | `auth` | e-mail length limit exclusive | caught by `email_length_limit_is_inclusive` |
| M41 | `auth` | trailing-dot domain accepted | caught by `email_normalization_table` |
| M42 | `auth` | domain without a dot accepted | caught by `email_normalization_table` |
| M43 | `jobs` | claim ordered by `id` only | caught by `among_due_jobs_the_earliest_run_at_goes_first_even_with_a_higher_id` (added after the first run left J1 alive) |
| M44 | `jobs` | due check `run_at <` (misses the boundary) | caught by `a_dead_workers_job_is_redelivered_after_its_lease_expires` |
| M45 | `jobs` | lease check `lease_until <` | caught by `a_dead_workers_job_is_redelivered_after_its_lease_expires` |
| M46 | `jobs` | attempts not incremented on claim | caught by `a_dead_workers_job_is_redelivered_after_its_lease_expires` |
| M47 | `jobs` | `complete` fence `attempts >=` instead of `=` | caught by `a_dead_workers_job_is_redelivered_after_its_lease_expires` |
| M48 | `jobs` | dead only when `attempts >` max | caught by `a_failed_attempt_is_requeued_with_exponential_backoff` |
| M49 | `jobs` | backoff off by one (`attempts` not `attempts-1`) | caught by `a_failed_attempt_is_requeued_with_exponential_backoff` |
| M50 | `jobs` | backoff cap 2 h instead of 1 h | caught by `backoff_table_cap_and_overflow` |
| M51 | `jobs` | shift cap 64 (overflow) | caught by `backoff_table_cap_and_overflow` |
| M52 | `jobs` | error truncated to 501 characters | caught by `long_error_messages_are_truncated_by_characters` |
| M53 | `jobs` | a panic treated as success | caught by `run_one_records_success_failure_and_panic` |
| M54 | `jobs` | pool `shutdown` never sets the stop flag | caught (hang: the run timed out) |
| M55 | `jobs` | final-attempt lease expiry `attempts >` max | caught by `a_lease_that_expires_on_the_final_attempt_kills_the_job` |
| M56 | `jobs` | `fail` returns `true` regardless | caught by `a_dead_workers_job_is_redelivered_after_its_lease_expires` |
| M57 | `jobs` | pool starts one worker regardless of config | caught by `the_configured_number_of_workers_really_runs_in_parallel` (added after the first run left J16 alive) |

What the table does not show: it tests *these* 57 edits, not every possible bug; two timing tests (M30) rely on margins, so a very loaded machine could make them flaky (I saw none in 15 runs); and passing mutants says nothing about security.

**Design notes.**

- The library functions take `now_ms` instead of reading the clock, so every time-dependent rule is tested at its exact boundary without sleeping.
- `Db` owns one `Connection` and is not shared between threads; the worker pool opens one per thread. `migrate_with` is public so tests can try failing and partial migration lists.
- `Authenticator` is built once (it precomputes the dummy hash) and reused; its parameters come from `AuthConfig`, and the PHC string stores them so old hashes keep verifying after a change.
- The session token is returned inside a `Secret`; the database never sees it.
- Jobs: claim is one statement; `attempts` is the fence; the lease check reuses the same query so crash recovery needs no separate reaper.
- Not handled: PostgreSQL/MySQL, connection pools, async access, password reset, rate limiting and lockout, MFA, cookies and CSRF, authorization (roles, permissions), rehash-on-login, lease extension, jitter in backoff, recurring jobs, job priorities, multi-machine clocks, backups, metrics and logging of worker errors, and encryption at rest.

</details>

---

## 14. Key takeaways

- Treat the database as a boundary: bound parameters only (✓ string-built SQL leaked rows and ran a stacked `DELETE`), `STRICT` tables and constraints so the database rejects bad data (✓), and errors matched by code, not by text.
- Transactions are the unit of correctness: commit-or-rollback by scope (✓ dropped means rolled back), `IMMEDIATE` for read-then-write (✓ the deferred upgrade failed), a busy timeout for contention (✓), and batching for speed (✓ about 140 times at `synchronous=FULL`).
- Migrations are ordered, transactional, re-checked under the lock, and refuse the future (✓ 8-thread race; a missing semicolon was a real bug).
- Configuration is layered and validated after merging, strict about typos, and never prints secrets (✓).
- Store password hashes made by a slow salted function and session tokens as digests of long random values; return the same error and do the same work for every login failure (✓ about 15 to 19 ms per Argon2 verification, 5 µs per token check, equal timing for unknown and wrong).
- A database-backed queue is at-least-once: claim in one statement, lease the job, fence the result, back off retries, keep handlers idempotent, and schedule follow-up work in the same transaction as the change that needs it (✓ 400 jobs, 8 threads, each claimed once; ✓ the stale worker's results were ignored).
- Measure the real ceiling: here it was the commit rate, so more workers changed nothing (✓).

**Connections.** [Lesson 45](45_http_apis.md) is the layer above: its idempotency keys, its bearer tokens and its 401/403 map directly onto `register`, `login` and `authenticate`; [Lesson 44](44_cli_serialization.md) supplies serde validation and the rule about secrets in arguments; [Lesson 43](43_observability_ci_supply_chain.md) covers logging without leaking and the dependency policy for a crate that compiles C; [Lessons 27 and 29](27_threads_channels.md) explain the threads and locks under the pool; [Lesson 26](26_error_design.md) is the model for `ConfigError`, `DbError` and `AuthError`; [Lesson 39](39_testing.md) covers the test techniques used. Next: Lesson 47 covers deployment and WebAssembly.

---

## 15. References

- `rusqlite` documentation (`Connection`, `Transaction`, `TransactionBehavior`, `params!`, `OptionalExtension`, `busy_timeout`, features such as `bundled`): <https://docs.rs/rusqlite>
- SQLite documentation: STRICT tables, type affinity, `PRAGMA user_version`, WAL mode, `PRAGMA synchronous`, `BEGIN IMMEDIATE`, `UPDATE ... RETURNING`, upsert, `EXPLAIN QUERY PLAN`, result codes: <https://www.sqlite.org/docs.html>
- `toml` crate documentation: <https://docs.rs/toml>
- `argon2` crate documentation (parameters, PHC strings, `PasswordHasher`, `PasswordVerifier`) and the PHC string format specification: <https://docs.rs/argon2>, <https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md>
- RFC 9106, "Argon2 Memory-Hard Function for Password Hashing and Proof-of-Work Applications": <https://www.rfc-editor.org/rfc/rfc9106>
- OWASP cheat sheets: Password Storage, Authentication, Session Management, SQL Injection Prevention: <https://cheatsheetseries.owasp.org/>
- `sha2` and `getrandom` crate documentation: <https://docs.rs/sha2>, <https://docs.rs/getrandom>
- The transactional outbox and idempotent receiver patterns (general descriptions): <https://microservices.io/patterns/data/transactional-outbox.html>
- The Twelve-Factor App, "Config": <https://12factor.net/config>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: the SQLite versus client/server comparison; `sqlx`, `diesel` and `tokio-postgres` as alternatives; what `synchronous = FULL`, `NORMAL` and `OFF` guarantee after a power failure; the rusqlite default busy timeout of five seconds; that other databases do not roll back schema changes; that WAL's reader/writer advantages show at commit time (my reader test showed no difference before commit); OWASP's minimum Argon2 parameters; that Linux exposes a process's environment in `/proc/<pid>/environ` to the same user and root; that a fast hash is the right tool for high-entropy tokens; the rehash-on-login procedure; heartbeat leases; jitter; `panic = "abort"` defeating `catch_unwind`; polling versus notification for queues; PostgreSQL's `now()` for shared time; and everything about cookies, CSRF, JWT, OAuth, MFA, rate limiting and network file systems. The measurements marked ✓ come from one shared virtual machine: the benchmark program is a release build, the tests are dev builds, medians and ranges are from three runs (nine logins per median), and the `fsync` costs reflect this virtual disk only. Exercise answers 1, 6, 7 and 8 are reasoning from memory; the code blocks in answers 3, 4 and 5 and in the debugging challenge were compiled and run; the SQL injection outputs, error texts, timings, the migration, lease, fencing and concurrency behaviors marked ✓ were observed with `rustc 1.97.0`, SQLite 3.54.1, `rusqlite 0.41.1`, `argon2 0.6.0` and `toml 1.2.1`.
{% endraw %}
