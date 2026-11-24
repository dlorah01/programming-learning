---
title: "Lesson 59 — Architecture, code review, system design, production reliability"
---

# Lesson 59 — Architecture, code review, system design, production reliability

**Module 19 · Professional mastery (first of two lessons)** · Status: generated · Edition used in examples: **2024** · External dependencies: `sha2 0.11.1` only (11 packages in `Cargo.lock`, the crate itself included)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior, general engineering knowledge, or opinion, and I say which.

**What this lesson is, and is not.** The earlier lessons taught the language, the libraries and the domains. This one is about the decisions around the code: how to **structure** a system so that it can change, how to **review** a change so that it does not hurt, how to **design** a service on the back of an envelope, and how to keep it **reliable** when its dependencies are not. The reliability part is built, not just described: a small crate, `reliab`, with a **token-bucket rate limiter**, a **circuit breaker**, **retries with backoff, jitter and a budget**, **idempotency keys**, a **load-shedding queue**, a **consistent-hash ring** and the **arithmetic of SLOs and capacity**. Time is always an argument, never read from the system clock, so every test is deterministic and an hour of traffic runs in a millisecond. What I did **not** do: run a distributed system, measure a network, use any real service mesh, queue, cloud product or observability vendor, or test any of this under real concurrency (the components are single-threaded values; sharing them across threads is an exercise in wrapping, not something I measured). Statements about real organizations, products and published practices are labeled "from memory".

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation sites were not. The consistent-hash positions and lookups, and the backoff table, were computed by a **separate Python program** (`p59/py/ref.py`, `hashlib`), not by the code under test.

Builds on [Lesson 13](13_structs_enums_methods.md) and [Lesson 15](15_newtype_typestate.md) (modeling states), [Lesson 26](26_error_design.md) (errors), [Lesson 27](27_threads_channels.md) to [Lesson 32](32_cancellation_streams_backpressure.md) (concurrency, async, backpressure), [Lesson 33](33_modules_api_design_rustdoc.md) (API design), [Lesson 39](39_testing.md), [Lesson 40](40_property_testing_fuzzing.md), [Lesson 41](41_benchmarking_profiling.md), [Lesson 43](43_observability_ci_supply_chain.md) (observability), [Lesson 45](45_http_apis.md) and [Lesson 46](46_databases_config_auth_jobs.md) (services), [Lesson 51](51_rpc_nodes_indexers.md) and [Lesson 52](52_web3_reliability.md) (off-chain reliability). The next lesson, Lesson 60, appears here only as plain text.

---

## 1. Why this lesson exists

Most outages are not caused by a hard algorithm. They are caused by a retry that tripled the load on a struggling database, a timeout that was longer than the caller's, a queue that grew until the process died, a deploy nobody could roll back, or a review that checked the formatting and missed the missing check. The claims below each have an experiment in the crate.

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| A token bucket in integers is exact and tells the caller how long to wait | refill is `elapsed_ms × rate` thousandths of a token, so a rate of 3 per second waits exactly 334 ms for one token (rounded up), and the wait is honest: acquiring after it succeeds (§6) |
| A clock that goes backwards must not mint tokens or rewind the bucket | `bucket_survives_clock_going_backwards` (§6) |
| Retrying without a budget multiplies load | a 10% budget grants exactly 10 retries per 100 requests; with no budget a 3-attempt policy makes 3× the calls during a full outage (§7) |
| A retry must stop at a deadline, and must not spend budget on a retry it refuses | `run_respects_the_deadline_and_does_not_spend_budget_on_it` (§7) |
| A circuit breaker needs a half-open state with a probe limit | opens after N consecutive failures, refuses until the timer, admits at most `probes` trial calls, one failure re-opens (§8) |
| The same idempotency key with a different body is a client bug, not a retry | `same_key_different_body_is_a_conflict`; an attempt that never finished is taken over after a timeout (§9) |
| A bounded queue must drop the right work | expired work is dropped and counted; a full queue evicts the newest of the least important; equal importance is rejected (§10) |
| Consistent hashing moves about 1/N of the keys; `hash % N` moves most of them | ✓ 895 of 5000 keys move when a fifth node joins a ring of four (ideal 1000), against 4005 of 5000 with `hash % N` (§11) |
| Virtual nodes are what makes the load even | ✓ with 1, 10, 100 and 200 virtual nodes per node the busiest node carries 4050, 3030, 2216 and 2209 of 10 000 keys (ideal 2000) (§11) |
| An error budget and a burn rate are plain integer arithmetic | 14 failures in 1000 against 99.9% is a burn rate of exactly 14.0; paging needs a short *and* a long window to agree (§12) |

The crate has **62 tests**, passing in 8 full runs; `cargo clippy --all-targets -- -D warnings` is clean; and 86 mutants (§21) were run against it.

## 2. The landscape (from memory)

None of this section was checked against current documentation. It names ideas so that you can recognize them; check details before relying on them.

- **Architecture vocabulary.** *Layered*, *hexagonal* ("ports and adapters") and *clean* architecture all say the same thing in different words: the business rules sit in the middle and do not depend on the database, the web framework or the clock; the outer parts depend inward, never the other way. *Domain-driven design* adds a vocabulary (bounded context, aggregate, ubiquitous language). *Event sourcing* and *CQRS* are patterns for specific problems, not defaults. A **monolith** with clean module boundaries is the right start for most teams; **microservices** buy independent deployment and cost network failure, versioning and operations.
- **Decision records.** An *architecture decision record* (ADR) is a short file per decision: context, the options considered, the decision, the consequences. Their value is that someone in two years can see *why*.
- **Reliability vocabulary.** *SLI* (a measured indicator, such as "fraction of requests answered under 300 ms"), *SLO* (the target for it), *SLA* (a contract, usually with money attached), *error budget* (what the SLO allows to fail), *burn rate* (how fast the budget is being spent). The multi-window, multi-burn-rate alert is described in Google's SRE books; the numbers below (14.4×) are the well-known ones for a 30-day window, from memory.
- **Resilience patterns.** timeouts, retries with exponential backoff and jitter, circuit breakers, bulkheads, load shedding, backpressure, idempotency keys, hedged requests, graceful degradation, canary and blue-green deploys, feature flags. Libraries exist for several of these (in Rust, for example, `tower` layers and `governor` for rate limiting, from memory); this lesson builds the small cores so that you know what they do.
- **Incident practice.** On-call rotations, runbooks, blameless postmortems, game days. The common finding in postmortems, from memory of many public ones: a change, a retry storm, or a saturated shared resource; rarely a single bug.

## 3. Architecture: structure that lets code change

### 3.1 Dependency direction is the whole idea

Draw the modules of a program as boxes and the "uses" relation as arrows. A healthy picture has the **domain** (the rules of the business) at the center with no outgoing arrows to I/O: it takes data and time as arguments and returns data. The outer layer (HTTP handlers, database access, queue consumers) knows the domain and translates between the world and it. The test of the design is a question: *can I run the domain's tests without a network, a database or a clock?* Everything in `reliab` passes that test, because `now_ms` is a parameter. That is why 62 tests finish in a fraction of a second, and why the breaker's open timer can be tested to the millisecond (§8).

In Rust the dependency direction is enforced by the compiler where you let it be: a crate for the domain that does not list `tokio` or `sqlx` in `Cargo.toml` **cannot** use them. A workspace with `domain`, `adapters` and `app` crates turns a convention into an error message. The cost is more crates to maintain; for a small program a module boundary and discipline is enough.

### 3.2 Ports: traits at the boundary, not everywhere

A **port** is a trait the domain defines and an adapter implements, for example `trait Clock { fn now_ms(&self) -> u64; }` (see `src/clock.rs`). Use one where a dependency really varies (time, randomness, storage, a remote API), because there the trait buys testability. Do not put a trait in front of every struct "for mocking": a trait with one implementation costs reading time and gives nothing back. Generic parameters give static dispatch and larger binaries; `dyn Trait` gives one copy of the code and a vtable call ([Lesson 21](21_trait_objects_dispatch.md)); at a port that is called at human speed, either is fine, and `dyn` is usually simpler to read.

### 3.3 Make illegal states unrepresentable, again

[Lesson 15](15_newtype_typestate.md) argued this for values. The breaker shows it for a protocol: its state is an enum whose variants carry only the data that makes sense in them, so there is no "open but no deadline" or "closed with a probe count":

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum State {
    /// Calls flow; `failures` is the current run of consecutive failures.
    Closed { failures: u32 },
    /// Calls are refused until `until_ms`.
    Open { until_ms: u64 },
    /// A few trial calls are allowed; `in_flight` have been admitted and `successes` have returned well.
    HalfOpen { in_flight: u32, successes: u32 },
}
```

Before reaching for a boolean and two optional fields, ask whether the states are mutually exclusive; if they are, an enum is the model.

### 3.4 Write the decision down

A decision record is five lines, not a document:

```text
ADR-007: Retries live in the client, with a budget
Status: accepted
Context: the order service retried failed calls to inventory three times; during the
  last incident that tripled load on a database that was already slow.
Decision: one retry helper (retry::run) with a shared RetryBudget of 10%; no other code retries.
Consequences: a request can fail faster during an outage (good); a flaky call that used to
  be hidden now surfaces (also good); every new client must take a budget (cost).
```

The point is not the template; it is that "no other code retries" is a rule a reviewer can check.

### 3.5 Where architecture goes wrong

Over-abstraction before the second use ("we might need to swap the database"); a shared "common" crate that everything depends on and nobody owns; services split along org-chart lines instead of data ownership; a distributed transaction where a single database transaction would have done; and the reverse, one module that every change must touch. I give these as opinion and experience, not measurement.

## 4. Code review: what a reviewer is for

A review has three jobs: **find defects**, **spread knowledge** and **keep the design coherent**. Style is the formatter's and the linter's job (`cargo fmt --check`, `cargo clippy -- -D warnings` in CI, [Lesson 43](43_observability_ci_supply_chain.md)); a human who spends attention on spacing has none left for the missing check.

### 4.1 A checklist, in the order a reviewer should look

1. **What is this for?** Read the description and the tests first. A change that cannot say what it does cannot be reviewed.
2. **Failure paths.** What happens on timeout, on a duplicate, on an empty input, on a huge input, when the dependency is down? Is every error handled or deliberately propagated ([Lesson 26](26_error_design.md))?
3. **Limits.** Is every queue, map, retry loop and buffer bounded? Unbounded growth is a delayed outage.
4. **Arithmetic and units.** Overflow, rounding direction, milliseconds versus seconds, signed versus unsigned. In `reliab` every rate is integer thousandths or parts per million for this reason.
5. **Concurrency.** What is shared, what guards it, can a lock be held across an `.await` ([Lesson 29](29_locks_deadlocks.md), [Lesson 31](31_tokio.md))?
6. **Security.** Who can call this, with what input, and what does it cost them to be hostile ([Lesson 52](52_web3_reliability.md))?
7. **Tests.** Do they fail if the change is wrong? A test that cannot fail is decoration; §21 shows how to find out.
8. **Operations.** Can it be turned off, rolled back, observed? Is there a metric for the new failure?
9. **API and naming.** Will the next reader guess what it does ([Lesson 33](33_modules_api_design_rustdoc.md))?

### 4.2 A review, worked

This is the sort of function that passes a skim. It is the *buggy* backoff of §19 (it is the same code):

```rust
/// The faulty program.
pub mod buggy {
    /// Delay before retry number `attempt`: `base * 2^attempt`, capped.
    pub fn delay(base_ms: u64, cap_ms: u64, attempt: u32) -> u64 {
        (base_ms * (1u64 << attempt)).min(cap_ms)
    }
}
```

A reviewer working the list finds: *(3) limits*: the delay is capped, good; *(4) arithmetic*: `base_ms * (1 << attempt)` multiplies *before* the cap, so for `base_ms = 100` the product overflows at attempt 58 (`100 × 2^58 > 2^64`), and `1u64 << attempt` itself overflows at 64; in a debug build that is a panic, in release it silently wraps and a **long outage** produces a **short** delay at exactly the moment the dependency needs distance. ✓ `dbg_buggy_panics_in_a_long_outage_fixed_does_not` shows both attempts 60 and 64 panic in the buggy version. A good review comment says what, why, and how to prove it:

> `delay` overflows once `attempt` is large (a dependency down for a few minutes is enough at a 100 ms base). Please saturate the shift and the multiplication and cap after. I added a test that calls it with attempts 0 to 199.

### 4.3 How to review well

- Review the **smallest unit that can be understood**: ask authors for changes under a few hundred lines; a thousand-line diff gets a skim.
- Comment on the code, not the person; ask a question when unsure ("what happens if this is called twice?"); mark what blocks and what is only a preference.
- Verify claims: run the branch, run the new test against the old code and watch it fail.
- Respond fast. A review that waits two days costs more than any defect it finds.
- As an author: keep the change focused, explain the why, and point to the risky lines yourself.

## 5. System design: a method that fits on one page

The interview-style exercise "design X" is a real activity in miniature: turn a vague need into numbers, pick the simplest design that meets them, and say where it will break.

1. **Requirements.** Functional (what it does) and non-functional (latency, availability, durability, consistency, cost). Write the SLO you will promise (§12).
2. **Estimate.** Requests per second at peak, size per item, items stored, read/write ratio. Round hard; the point is the order of magnitude. Peak is often some multiple of average; pick one and say so.
3. **Concurrency by Little's law.** The number in flight is arrival rate × time each takes. ✓ `concurrency(1000, 200)` is 200: a service answering 1000 requests per second at 200 ms each has 200 requests in flight on average, which is the number of connections, threads or tasks you must be able to hold, and the size at which a queue starts to matter.
4. **Servers.** `servers_needed(1000, 100, 30)` is 13: 1000 requests per second at 100 per server, with 30% headroom, rounds up to 13. Headroom is not waste; it is what lets one server fail without the rest tipping over.
5. **Storage.** Size per item × items × replication, and what happens on growth. Decide the key you will shard on before you decide the database.
6. **The design**, simplest first: one service and one database; then what breaks first at 10×; then the specific mechanism (cache, queue, partition) that addresses *that*.
7. **Failure.** For each dependency: what if it is slow, down, wrong? What do callers see? What does the user see?

### 5.1 Availability arithmetic

Components that must all work multiply: ✓ two 99.9% dependencies in series give `serial_ppm(&[999_000, 999_000])` = 998 001 ppm, 99.8001%. A request that touches five 99.99% services is at best 99.95%. Redundancy that needs only one to work is the complement: ✓ two 90% components in parallel give 99% (`parallel_ppm`). The assumption behind the second formula is **independent failures**, which is often false (same rack, same deploy, same bug), so treat it as an upper bound. Related: with MTBF (mean time between failures) and MTTR (mean time to repair), availability is MTBF/(MTBF+MTTR) (Exercise 8); halving repair time helps exactly as much as doubling the time between failures, and is usually cheaper.

### 5.2 Consistency is a decision, not a default

Replicating data raises availability and raises the question of what a reader sees right after a write. The CAP and PACELC slogans (from memory: during a network partition choose between consistency and availability; otherwise choose between latency and consistency) are a prompt to ask *which operations need the latest value and which can accept a stale one*. A bank balance and a "likes" counter need different answers. Say which you chose and what the client sees.

## 6. Rate limiting: a token bucket in integers ✓

A rate limiter protects a service from a caller (or a caller from itself). The token bucket allows a sustained rate and a burst: tokens accumulate at `rate` per second up to `capacity`; each request takes one. In floating point this goes wrong in small ways (accumulated rounding, a bucket that never quite refills), so the crate counts **thousandths of a token**: a rate of `r` tokens per second adds exactly `r` thousandths per millisecond.

```rust
/// Takes `n` tokens, or says how many milliseconds to wait before `n` tokens exist
/// (`u64::MAX` if they never will).
pub fn try_acquire(&mut self, now_ms: u64, n: u64) -> Result<(), u64> {
    self.refill(now_ms);
    let need = u128::from(n) * 1000;
    if need <= self.milli_tokens {
        self.milli_tokens -= need;
        return Ok(());
    }
    if n > self.capacity || self.refill_per_sec == 0 {
        return Err(u64::MAX);
    }
    let missing = need - self.milli_tokens;
    let rate = u128::from(self.refill_per_sec);
    Err(u64::try_from(missing.div_ceil(rate)).unwrap_or(u64::MAX))
}
```

What the tests pin down, all ✓:

- A full bucket allows `capacity` requests and then refuses with an exact wait: capacity 5, 10 per second, six requests at time 0, the sixth is told `Err(100)` (`bucket_allows_burst_then_refuses`).
- The wait is honest and rounds up: 3 per second means 333.33 ms per token, the answer is 334, and acquiring at 334 succeeds (`bucket_wait_time_rounds_up_and_is_honest`). Rounding *down* would tell clients to come back too early and fail again.
- Partial tokens count: after 150 ms at 10 per second there are 1.5 tokens; asking for two says "50 ms" (`bucket_wait_accounts_for_partial_tokens`).
- A request bigger than the capacity can never succeed and says so (`u64::MAX`) rather than a wait that never ends; so does a bucket with rate 0 (`bucket_impossible_requests`). A request for exactly the capacity *can* succeed and gets a finite wait (`bucket_request_for_the_whole_capacity_waits_instead_of_failing_forever`, added after a mutation run found it unpinned).
- A clock that goes backwards adds nothing and does not move the origin backwards, so the next forward step is not double-counted (`bucket_survives_clock_going_backwards`).
- Large values do not overflow (`u128` intermediates).

Two design choices to notice. The limiter **returns the wait** instead of sleeping, which keeps it a pure function of time and lets the caller decide (return `429` with `Retry-After`, queue, or sleep). And it limits *one* key; a per-user limit is a map of buckets, which then needs its own eviction so that a million one-time callers do not become a memory leak.

**Why not a fixed window?** "100 per minute, counted per calendar minute" lets 100 requests through at 12:00:59 and 100 more at 12:01:00: twice the limit in two seconds. A sliding window (Exercise 2) or a bucket does not. ✓ `ex2_never_more_than_limit_in_any_window` shows the 3-per-second case with requests at 997, 998, 999 and 1000.

## 7. Timeouts, retries and budgets ✓

### 7.1 Timeouts come first

A call without a timeout can wait forever, holding a thread, a connection and a slot. Every network call has one, set from the **end-to-end deadline**, not from a constant: if the caller gives up after 1 s, a downstream call that takes up to 5 s is pure waste. Exercise 5 (`hop_timeout`) propagates a deadline: each hop gets what is left, minus what the caller needs to answer, never more than its own cap.

### 7.2 Backoff and jitter

Retrying immediately is right for a lost packet and wrong for an overloaded server. **Exponential backoff** spaces retries (`base × 2^attempt`, capped); **jitter** stops thousands of clients that failed together from retrying together. "Full jitter" draws the delay uniformly from `0..=backoff`.

```rust
impl Backoff {
    /// The delay before retry number `attempt` (0 for the first retry).
    pub fn delay(&self, attempt: u32) -> u64 {
        let factor = if attempt >= 64 {
            u64::MAX
        } else {
            1u64 << attempt
        };
        self.base_ms.saturating_mul(factor).min(self.cap_ms)
    }

    /// "Full jitter": a delay drawn uniformly from `0..=delay(attempt)`, given a random number.
    pub fn full_jitter(&self, attempt: u32, random: u64) -> u64 {
        let d = self.delay(attempt);
        if d == u64::MAX {
            random
        } else {
            random % (d + 1)
        }
    }
}
```

✓ The table of delays for base 100 ms, cap 5000 ms is `[100, 200, 400, 800, 1600, 3200, 5000, 5000]`, matching `py/ref.py`; `backoff_never_overflows` runs attempts up to 1000 with a base near `u64::MAX/2`; `full_jitter_stays_within_bounds` checks 2000 random inputs against the bound. The random number is a **parameter**: production passes a real generator, tests pass fixed values, which is why the deadline test can assert the exact sleeps `[100, 100]`.

### 7.3 A retry budget keeps retries from multiplying load

If every client retries three times and a dependency fails completely, it receives **3×** its normal traffic exactly when it is weakest (arithmetic, not measured). A **retry budget** says: retries may be at most some percentage of first attempts. Each original request deposits a fraction of a retry; each retry spends one.

```rust
impl RetryBudget {
    /// `ratio_pct` of 10 means one retry per ten requests; `reserve` retries are available at the start.
    pub fn new(ratio_pct: u64, reserve: u64) -> RetryBudget {
        RetryBudget {
            milli_tokens: reserve * 1000,
            max_milli: reserve.max(1) * 1000,
            deposit_milli: ratio_pct * 10,
        }
    }
    /// Call once per original request.
    pub fn on_request(&mut self) {
        self.milli_tokens = (self.milli_tokens + self.deposit_milli).min(self.max_milli);
    }
    /// Takes one retry from the budget if there is one.
    pub fn try_retry(&mut self) -> bool {
        if self.milli_tokens >= 1000 {
            self.milli_tokens -= 1000;
            true
        } else {
            false
        }
    }
    pub fn available(&self) -> u64 {
        self.milli_tokens / 1000
    }
}
```

✓ `budget_limits_retries_to_a_ratio`: with a ratio of 10 %, 100 requests that all fail grant exactly 10 retries. The reserve covers a cold start; the cap stops a quiet hour from banking a thousand retries to spend in a burst (`budget_reserve_and_cap`). The budget is shared across calls to one dependency (in practice per client process or per service), not per request.

### 7.4 Putting it together

```rust
/// Runs `op(attempt)` until it succeeds or the policy says stop. `random` supplies the jitter,
/// `sleep` is told how long to wait, `retryable` decides which errors are worth another try.
pub fn run<T, E>(
    policy: &RetryPolicy,
    budget: &mut RetryBudget,
    random: &mut dyn FnMut() -> u64,
    sleep: &mut dyn FnMut(u64),
    retryable: &dyn Fn(&E) -> bool,
    op: &mut dyn FnMut(u32) -> Result<T, E>,
) -> Result<T, RetryError<E>> {
    budget.on_request();
    let mut waited = 0u64;
    let mut attempt = 0u32;
    loop {
        let err = match op(attempt) {
            Ok(v) => return Ok(v),
            Err(e) => e,
        };
        if !retryable(&err) {
            return Err(RetryError::NotRetryable(err));
        }
        if attempt + 1 >= policy.max_attempts {
            return Err(RetryError::Exhausted(err));
        }
        let wait = policy.backoff.full_jitter(attempt, random());
        if waited.saturating_add(wait) > policy.deadline_ms {
            return Err(RetryError::DeadlineExceeded(err));
        }
        if !budget.try_retry() {
            return Err(RetryError::BudgetDenied(err));
        }
        sleep(wait);
        waited += wait;
        attempt += 1;
    }
}
```

The order of the checks matters and every one is tested: a non-retryable error returns at once; the attempt limit comes next; the deadline is checked **before** the budget is spent, so a retry that was never going to run does not use up a token (`run_respects_the_deadline_and_does_not_spend_budget_on_it`: waits `[100, 100]`, two retries spent, the refused third costs nothing, budget left is 3); a wait that lands *exactly* on the deadline is allowed (`waiting_exactly_the_deadline_is_allowed`, a gap the first mutation run exposed); and every request, including one that succeeds first time, earns credit (`every_request_earns_retry_credit`, also from the mutation run).

**Retry only what is safe.** A retry of a read is safe. A retry of a write is safe only if the write is idempotent (§9) or the failure proves it did not happen. "Timed out" proves nothing: the server may have done the work and lost the reply. Which errors are retryable is a decision the caller makes, which is why `retryable` is a parameter.

## 8. The circuit breaker ✓

When a dependency is failing, calling it harder does not help it or you. A breaker watches the calls and, after enough consecutive failures, **opens**: calls fail immediately without being made, which frees your threads and gives the dependency room to recover. After a delay it lets a few **probe** calls through (**half-open**): if they succeed it **closes**, if one fails it opens again for another full delay.

```rust
impl CircuitBreaker {
    /// Opens after `failure_threshold` consecutive failures, stays open `open_ms`, then admits up to
    /// `probes` trial calls; `probes` successes close it again.
    pub fn new(failure_threshold: u32, open_ms: u64, probes: u32) -> CircuitBreaker {
        assert!(failure_threshold > 0 && probes > 0);
        CircuitBreaker {
            failure_threshold,
            open_ms,
            probes,
            state: State::Closed { failures: 0 },
        }
    }

    pub fn state(&self) -> State {
        self.state
    }

    /// May a call be made now? Call `record_success` or `record_failure` for every call admitted.
    pub fn allow(&mut self, now_ms: u64) -> bool {
        match self.state {
            State::Closed { .. } => true,
            State::Open { until_ms } => {
                if now_ms >= until_ms {
                    self.state = State::HalfOpen {
                        in_flight: 1,
                        successes: 0,
                    };
                    true
                } else {
                    false
                }
            }
            State::HalfOpen {
                in_flight,
                successes,
            } => {
                if in_flight + successes < self.probes {
                    self.state = State::HalfOpen {
                        in_flight: in_flight + 1,
                        successes,
                    };
                    true
                } else {
                    false
                }
            }
        }
    }

    pub fn record_success(&mut self) {
        match self.state {
            State::Closed { .. } => self.state = State::Closed { failures: 0 },
            State::HalfOpen {
                in_flight,
                successes,
            } => {
                let successes = successes + 1;
                if successes >= self.probes {
                    self.state = State::Closed { failures: 0 };
                } else {
                    self.state = State::HalfOpen {
                        in_flight: in_flight.saturating_sub(1),
                        successes,
                    };
                }
            }
            State::Open { .. } => {} // a late result from before the circuit opened
        }
    }

    pub fn record_failure(&mut self, now_ms: u64) {
        match self.state {
            State::Closed { failures } => {
                let failures = failures + 1;
                if failures >= self.failure_threshold {
                    self.state = State::Open {
                        until_ms: now_ms.saturating_add(self.open_ms),
                    };
                } else {
                    self.state = State::Closed { failures };
                }
            }
            State::HalfOpen { .. } => {
                self.state = State::Open {
                    until_ms: now_ms.saturating_add(self.open_ms),
                }
            }
            State::Open { .. } => {}
        }
    }
}
```

✓ What the tests establish:

- It opens at exactly the threshold, not one early or one late (`breaker_opens_after_consecutive_failures`: three failures, `Open { until_ms: 1002 }`, refused at 1001, and the boundary at `until_ms` itself is where probing starts).
- A success in the closed state resets the run (`breaker_success_resets_the_run`): *consecutive* failures, not total.
- Half-open admits at most `probes` calls at once and needs `probes` successes to close (`breaker_half_open_limits_probes_and_closes`); freed slots can be used again (`breaker_probe_slots_free_up_after_success`).
- One failure while half-open re-opens it (`breaker_half_open_failure_reopens`).
- A late result from a call admitted *before* the circuit opened is ignored while open (`breaker_ignores_late_results_while_open`): otherwise a straggler's success would close a circuit that was opened for good reason.
- The open timer saturates instead of overflowing (`breaker_open_time_saturates`), and an impossible configuration (threshold 0, no probes) is refused with a panic at construction (`a_breaker_that_can_never_open_is_refused`, `a_breaker_without_probes_is_refused`).

What a breaker does **not** do: tell a slow dependency from a dead one (count slow calls as failures if you want that), or share state across processes (each instance has its own view, which is usually fine and sometimes confusing). Bulkheads (Exercise 7) are the partner idea: give each caller class its own slots so one cannot starve the others, and a breaker per dependency so that one failing dependency does not take down calls to the others.

## 9. Idempotency keys ✓

A client that times out does not know whether the server did the work. The honest protocol is: the client generates a **key** for the *intent*, sends it with every attempt, and the server guarantees that the effect happens once and that every attempt gets the same answer.

```rust
/// Decides what to do with a request carrying `key` and body `payload`.
pub fn begin(&mut self, key: &str, payload: &[u8], now_ms: u64) -> Begin {
    let fingerprint: [u8; 32] = Sha256::digest(payload).into();
    if let Some(e) = self.entries.get(key) {
        if self.expired(e, now_ms) {
            self.entries.remove(key);
        } else if e.fingerprint != fingerprint {
            return Begin::Conflict;
        } else {
            match &e.slot {
                Slot::Done { result, .. } => return Begin::Replay(result.clone()),
                Slot::Working { since_ms }
                    if now_ms.saturating_sub(*since_ms) < self.working_timeout_ms =>
                {
                    return Begin::InProgress;
                }
                Slot::Working { .. } => {} // presumed dead: fall through and take over
            }
        }
    }
    self.entries.insert(
        key.to_string(),
        Entry {
            fingerprint,
            slot: Slot::Working { since_ms: now_ms },
        },
    );
    Begin::New
}
```

Four outcomes, four tests (all ✓):

| Outcome | Meaning | Test |
|---|---|---|
| `New` | first sight: do the work, then `complete` | `first_request_is_new_then_in_progress_then_replay` |
| `InProgress` | another attempt is working: wait or ask again later | same test |
| `Replay(result)` | done: return the stored answer without repeating the effect | same test |
| `Conflict` | same key, **different** body: a client bug, never a retry | `same_key_different_body_is_a_conflict` |

The details that make it correct: the body is compared by SHA-256 **fingerprint** so a key cannot be reused for a different request; an attempt still "working" after a timeout is presumed dead and taken over (`dead_attempt_is_taken_over_after_timeout`, boundary at exactly the timeout); stored results expire after a TTL, and after expiry a *different* body is treated as a fresh start, not a conflict (`results_expire_after_the_ttl`); completing twice keeps the first result (`complete_errors`); and `abandon` forgets unfinished work but never a stored result (`abandon_forgets_only_unfinished_work`).

**What this model leaves out, and it matters.** The store is in memory, in one process. A real one must make "check key, do the work, store the result" **atomic with the effect**, usually in the same database transaction as the business change; otherwise a crash between "did it" and "recorded it" brings the duplicate back. Whether the stored result survives restarts and how long the TTL must be (longer than any client will keep retrying) are design decisions the model does not make for you.

## 10. Load shedding ✓

When demand exceeds capacity, something has to give. If the service accepts everything, queues grow, latency grows with them, and soon every response arrives after its caller has stopped waiting: the service is busy and useless. **Shedding** chooses to refuse some work early so that the rest finishes in time.

```rust
/// Offers an item. A higher `priority` number is more important. An item whose deadline is not
/// after `now_ms` is dropped at once (counted as expired).
pub fn push(&mut self, value: T, priority: u8, deadline_ms: u64, now_ms: u64) -> Admit<T> {
    if deadline_ms <= now_ms {
        self.expired += 1;
        return Admit::Rejected(value);
    }
    self.drop_expired(now_ms);
    if self.items.len() < self.capacity {
        self.items.push_back(Item {
            priority,
            deadline_ms,
            value,
        });
        return Admit::Queued;
    }
    // full: the newest of the least important items is the one to lose
    let victim = self
        .items
        .iter()
        .enumerate()
        .rev()
        .min_by_key(|(_, i)| i.priority)
        .map(|(k, i)| (k, i.priority));
    match victim {
        Some((k, p)) if p < priority => {
            let old = self.items.remove(k).expect("index from enumerate");
            self.items.push_back(Item {
                priority,
                deadline_ms,
                value,
            });
            Admit::Evicted(old.value)
        }
        _ => Admit::Rejected(value),
    }
}
```

The queue is **bounded** (it can never be the thing that runs the process out of memory), **deadline-aware** (work whose caller has already given up is dropped, and counted, because it is wasted effort), and **priority-aware**:

- ✓ at capacity, a more important arrival evicts the *newest of the least important* item (`important_work_evicts_the_least_important_newest`); newest, because it has waited least and the old item's sunk cost is higher; an arrival of equal importance is rejected, since evicting an equal gains nothing.
- ✓ a deadline equal to now is already expired (`deadline_equal_to_now_is_expired`).
- ✓ `pop` serves the highest priority, oldest first among equals (`pop_prefers_priority_then_age`).

Rejecting early is a kindness: the caller learns in microseconds and can try another replica or degrade gracefully. Pair it with the caller side: a `429` or `503` with a retry hint, and a client that treats it as a signal (a retry budget, §7.3) instead of a reason to retry harder. **Backpressure** ([Lesson 32](32_cancellation_streams_backpressure.md)) is the same idea inside a program: a bounded channel makes a fast producer wait for a slow consumer instead of buffering without limit.

## 11. Consistent hashing ✓

To spread keys over N servers, `hash(key) % N` works until N changes; then almost every key lands on a different server. ✓ Going from 4 to 5 servers moved **4005 of 5000** keys with modulo, which for a cache means a cold cache and for a database means moving nearly all the data.

**Consistent hashing** places servers and keys on the same circle (here the 64-bit space of the first eight bytes of SHA-256). A key belongs to the first server position at or after it, wrapping around. Adding a server takes over only the arc just before each of its positions; nothing else moves.

```rust
/// The first eight bytes of `SHA-256(data)`, big-endian.
pub fn hash64(data: &[u8]) -> u64 {
    let d = Sha256::digest(data);
    u64::from_be_bytes(d[..8].try_into().expect("8"))
}

/// The ring position of virtual node `i` of `node`: the hash of `node`, `#` and the decimal `i`.
pub fn point(node: &str, i: u32) -> u64 {
    hash64(format!("{node}#{i}").as_bytes())
}

impl Ring {
    pub fn new(vnodes: u32) -> Ring {
        assert!(vnodes > 0);
        Ring {
            vnodes,
            points: BTreeMap::new(),
            nodes: BTreeSet::new(),
        }
    }

    pub fn nodes(&self) -> impl Iterator<Item = &str> {
        self.nodes.iter().map(String::as_str)
    }
    pub fn len(&self) -> usize {
        self.nodes.len()
    }
    pub fn is_empty(&self) -> bool {
        self.nodes.is_empty()
    }

    /// Adds a node. Returns false if it was already there. A position already taken by another
    /// node is left to that node, so the ring never depends on insertion order for ties
    /// only when no tie exists; ties between 64-bit hashes are astronomically rare.
    pub fn add(&mut self, node: &str) -> bool {
        if !self.nodes.insert(node.to_string()) {
            return false;
        }
        for i in 0..self.vnodes {
            self.points
                .entry(point(node, i))
                .or_insert_with(|| node.to_string());
        }
        true
    }

    /// Removes a node and its positions. Returns false if it was not there.
    pub fn remove(&mut self, node: &str) -> bool {
        if !self.nodes.remove(node) {
            return false;
        }
        self.points.retain(|_, n| n != node);
        true
    }

    /// The node responsible for `key`: the first position at or after the key's hash, wrapping around.
    pub fn lookup(&self, key: &str) -> Option<&str> {
        let h = hash64(key.as_bytes());
        self.points
            .range(h..)
            .next()
            .or_else(|| self.points.iter().next())
            .map(|(_, n)| n.as_str())
    }
}
```

✓ Known answers computed independently in Python (`py/ref.py`): `hash64("a")` is `0xca978112ca1bbdca`; the position of virtual node 0 of `n1` is `0x36ab20d02d20c204`; and for a ring of `n1`, `n2`, `n3` with one position each, the keys `alpha`, `beta`, `gamma`, `delta`, `k1`, `k2`, `k3` belong to `n3`, `n1`, `n3`, `n3`, `n3`, `n1`, `n1` (`lookup_known_answers_with_one_vnode`).

- ✓ **Minimal movement.** Adding a fifth node to four (100 virtual nodes each), every key that moved moved *to the new node*, and 895 of 5000 moved against an ideal of 1000 (`adding_a_node_moves_keys_only_to_the_new_node`). Removing a node moves only that node's keys (`removing_a_node_moves_only_its_keys`).
- ✓ **Virtual nodes even out the load.** With one position per node, five nodes carried between 349 and 4050 of 10 000 keys; 10 positions gave 1337 to 3030; 100 gave 1779 to 2216; 200 gave 1851 to 2209 (ideal 2000). More positions cost memory and a slower add: ✓ adding one node with 100 positions takes about 18 to 20 µs (§14).
- ✓ A key that hashes **exactly** onto a position belongs to that position (`a_key_that_hashes_exactly_to_a_position_belongs_to_that_position`: the key `"n2#0"` has the same hash as virtual node 0 of `n2`). That boundary is a one-character difference (`..h` versus `h+1..`) that the first mutation run showed was not covered.
- ✓ The ring does not depend on the order in which nodes were added (`ring_is_independent_of_insertion_order`), which matters when two processes must agree without talking.

Limits: it does not by itself give **replication** (Exercise 4 walks the ring to find the next distinct nodes), balance by *capacity* (give bigger servers more positions), or protection from **hot keys** (a single popular key lands on one node however you hash). Real systems that use rings differ in the details (from memory: some use fixed slots instead, and some use "rendezvous" hashing, which needs no ring at all).

## 12. SLOs, error budgets and burn rates ✓

An **SLO** turns "reliable" into a number a team can plan with: *99.9% of requests succeed in under 300 ms over 30 days*. The complement is the **error budget**: the amount of failure you are allowed. ✓ `error_budget(999_000, 1_000_000)` is 1000 failures of a million; `allowed_downtime_secs(999_000, 30)` (Exercise 1) is 2592 s, 43.2 minutes a month. The budget changes the conversation: spent, you slow risky launches and fix reliability; unspent, you may ship faster.

```rust
/// How many of `total` requests may fail and still meet an objective of `slo_ppm`.
pub fn error_budget(slo_ppm: u64, total: u64) -> u64 {
    let allowed_ppm = PPM.saturating_sub(slo_ppm);
    (u128::from(total) * u128::from(allowed_ppm) / u128::from(PPM)) as u64
}

/// How fast the budget is being spent, in thousandths: 1000 means exactly on budget, 14400 means
/// 14.4 times too fast. `None` without traffic or with an objective of 100% (no budget at all).
pub fn burn_rate_milli(slo_ppm: u64, total: u64, bad: u64) -> Option<u64> {
    let allowed_ppm = PPM.checked_sub(slo_ppm).filter(|a| *a > 0)?;
    if total == 0 {
        return None;
    }
    let bad_ppm = u128::from(bad.min(total)) * u128::from(PPM) / u128::from(total);
    Some((bad_ppm * 1000 / u128::from(allowed_ppm)) as u64)
}

/// Pages only when a short and a long window both burn at least `threshold_milli`: the short window
/// shows it is happening now, the long one that it is not a blip.
pub fn should_page(
    short: (u64, u64),
    long: (u64, u64),
    slo_ppm: u64,
    threshold_milli: u64,
) -> bool {
    let hot =
        |w: (u64, u64)| burn_rate_milli(slo_ppm, w.0, w.1).is_some_and(|b| b >= threshold_milli);
    hot(short) && hot(long)
}
```

The **burn rate** is how fast the budget is being used relative to an even spend: 1.0 means the budget lasts exactly the window. ✓ 14 failures in 1000 against 99.9% is `Some(14_000)` thousandths, 14.0× (`burn_rate_milli(999_000, 1000, 14)`), and it also works at the strict end, 1 failure in a million against 99.9999% is exactly 1.0 (`burn_rate_works_for_a_very_strict_objective`, found by mutation).

**Alert on burn, not on a single bad minute.** A short window shows it is happening now; a long window shows it is not a blip. Page only when **both** exceed a threshold: ✓ `should_page` (`paging_needs_both_windows`) refuses when only one window is hot, pages when both are, and counts "exactly at the threshold" as hot. From memory, the commonly cited starting point for a 30-day SLO is to page at a burn rate of about 14.4 over one hour (confirmed by five minutes), which would spend 2% of the monthly budget in that hour; ticket at lower rates over longer windows. Treat those numbers as a starting point to tune, not as a rule.

An SLO needs an **SLI you can actually measure**, ideally at the user's side (a load balancer or a client), because the service that is down cannot report that it is down.

## 13. Observability and incidents (from memory)

- **Three signals.** Metrics (cheap, aggregated: rate, errors, duration), logs (events with context), traces (one request across services). Start with the "RED" set per endpoint (rate, errors, duration) and the "USE" set per resource (utilization, saturation, errors). Use histograms or percentiles for latency, never the mean: Exercise 6 ✓ shows one slow request in a hundred is invisible at the 99th percentile (`10`) and visible at the 100th (`5000`).
- **Instrument the reliability machinery itself.** Breaker state changes, retries taken and refused (`BudgetDenied`), shed and expired counts (`SheddingQueue::expired`), rate-limit rejections. These are the numbers that tell you a protection is working or is hiding a problem.
- **Deployments.** Small, frequent, reversible changes beat large rare ones. Canary (a small share of traffic first), feature flags (separate "deployed" from "enabled") and a rollback that is rehearsed. A change you cannot roll back needs a much higher bar.
- **Incidents.** Stabilize first (roll back, shed load, fail over), diagnose second. Write a blameless postmortem: timeline, contributing causes, what detected it and how late, what will change, with owners. The test of a postmortem is whether it produces a change in code, alert or process.

## 14. Cost ✓

Release build, x86-64 Linux, 2 cores, `black_box` around results, three runs of `examples/bench.rs` after the final source. Ranges are min to max of the three:

| Operation | ns per operation (3 runs) |
|---|---|
| token bucket `try_acquire` | 4.2 to 5.2 |
| breaker `allow` + `record_success` | 0.3 to 0.5 |
| idempotency `begin` (10 000 keys; new, then replays) | 170.7 to 189.5 |
| ring `lookup` (10 nodes × 100 positions; includes formatting the key) | 180.4 to 187.4 |
| ring `add` of one node with 100 positions | 18 391 to 20 264 |

Read these as *shape*, not as promises. The protection logic costs a few nanoseconds to a few hundred; the idempotency and ring numbers include a SHA-256 and a `String` allocation each, which are most of that time. Nothing here will ever be the reason a service is slow: the network call these guard costs microseconds to milliseconds, 3 to 6 orders of magnitude more. The breaker's 0.3 ns is below what this micro-benchmark can meaningfully resolve (the loop may be partly optimized); I report it, but do not draw conclusions from it.

## 15. Rust notes

- **Injecting time and randomness as arguments** (or through a tiny trait) is the single most useful testability habit in this lesson. `Cell` in `ManualClock` gives interior mutability for a single-threaded test clock ([Lesson 24](24_interior_mutability.md)); a real clock would be a separate type behind the same trait.
- **Integer arithmetic with wide intermediates.** `u128::from(a) * u128::from(b)` cannot overflow for `u64` inputs; the narrowing `as u64` after a division by a large number is where a bug could hide, and is why the tests include values near `u64::MAX`.
- **`div_ceil`** (stable on unsigned integers) says "round up" in one word, replacing the `(a + b - 1) / b` idiom that can itself overflow.
- **Closures as dependencies.** `retry::run` takes `&mut dyn FnMut` and `&dyn Fn` for the random source, the sleeper, the retry predicate and the operation. That keeps it non-generic (one copy of the code) and lets a test pass a closure that records the sleeps. In async code the same function would take an async sleeper; the structure, and the tests, would not change.
- **Returning a typed outcome instead of acting.** `Admit`, `Begin` and `RetryError` carry the reason; the caller decides what to do. That is more code than returning `bool` and prevents the caller from guessing.
- **`#[should_panic]` for constructors** documents preconditions (`Ring::new(0)`); for fallible inputs from outside the program, return an error instead ([Lesson 26](26_error_design.md)).

## 16. Common mistakes

1. **No timeout** on a network call, or a default one nobody chose.
2. **A timeout longer than the caller's**: the work finishes after nobody is waiting.
3. **Retrying everything**, including non-idempotent writes and errors that will never succeed (a `400`).
4. **Retrying without backoff or jitter**: a synchronized stampede on recovery.
5. **Retrying at every layer**: three layers with three attempts each make 27 calls for one request.
6. **No retry budget**, so an outage multiplies load by the retry count.
7. **Unbounded queues and maps**, including per-user limiter state that is never evicted.
8. **Measuring the mean latency** and missing the tail.
9. **Alerting on causes (CPU 80%) instead of symptoms** (users failing), and paging on a single noisy sample.
10. **A breaker that counts total failures** instead of consecutive ones or a rate, or that never probes.
11. **An idempotency key that is not tied to the request body**, so a reused key silently returns the wrong answer.
12. **Idempotency check and effect in separate transactions**, so a crash between them duplicates the effect.
13. **`hash % N` sharding** and then growing N.
14. **Reviewing style instead of failure paths**, and approving tests that cannot fail.
15. **Treating availability numbers as independent** when the components share a failure cause, and designing for 99.99% on top of a 99.9% dependency.

## 17. Security notes and honest limits

- **Rate limiting is a defense, not authentication.** Limit by something the attacker cannot cheaply change; an IP address is weak behind shared NATs and cheap to rotate; a header the client sets is no key at all.
- **Idempotency keys chosen by clients are scoped by the authenticated caller** in any real system; otherwise one user's key could replay another's result. The model has one namespace and says nothing about authorization.
- **Error messages and `Retry-After` headers leak** capacity and internal state; decide what to reveal.
- **Backpressure and shedding are also availability defenses** against deliberate overload, but they do not distinguish a hostile flood from a popular launch.
- **Limits of what I verified.** Everything here is single-threaded, in memory, deterministic. I did not test under concurrent load, with a real clock, across processes, over a network, or with persistence; the mutation testing shows the tests detect 86 small edits, not that the models match any production system; and the numbers in §14 are one machine's micro-benchmarks.

## 18. Exercises

Each solution is in the `exp` crate with tests (12 tests, 8 full runs). Answers are collapsed; try first.

**Exercise 1.** Write `allowed_downtime_secs(slo_ppm, days)`: how many seconds of total outage does the objective allow over a window? A 100% objective allows none.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 1: how many seconds of total outage fit in the error budget of a time window?

   /// Seconds of full outage allowed in `days` days by an availability objective of `slo_ppm`
   /// (999_900 is 99.99%). Rounds down; an objective of 100% or more allows none.
   pub fn allowed_downtime_secs(slo_ppm: u64, days: u64) -> u64 {
       let allowed_ppm = 1_000_000u64.saturating_sub(slo_ppm);
       let total = u128::from(days) * 86_400;
       (total * u128::from(allowed_ppm) / 1_000_000) as u64
   }
```

Widening to `u128` before multiplying avoids overflow; integer division rounds down, which is the safe direction for a budget (never promise a second you do not have). ✓ 99.9% over 30 days is 2592 s; 99.99% is 259 s.
</details>

**Exercise 2.** A fixed window of "3 per second" lets 6 requests through in two milliseconds across a boundary. Write a limiter that never allows more than `limit` requests in *any* window of `window_ms`.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 2: a sliding-window-log rate limiter: at most `limit` requests in any window.
   use std::collections::VecDeque;

   /// Remembers the time of every admitted request in the last `window_ms`.
   pub struct SlidingLog {
       limit: usize,
       window_ms: u64,
       times: VecDeque<u64>,
   }

   impl SlidingLog {
       pub fn new(limit: usize, window_ms: u64) -> SlidingLog {
           SlidingLog {
               limit,
               window_ms,
               times: VecDeque::new(),
           }
       }

       /// Admits a request at `now_ms` if fewer than `limit` were admitted in `(now_ms - window_ms, now_ms]`.
       pub fn allow(&mut self, now_ms: u64) -> bool {
           while let Some(&t) = self.times.front() {
               if now_ms.saturating_sub(t) >= self.window_ms {
                   self.times.pop_front();
               } else {
                   break;
               }
           }
           if self.times.len() < self.limit {
               self.times.push_back(now_ms);
               true
           } else {
               false
           }
       }
   }
```

The log keeps one timestamp per admitted request, so memory is proportional to the limit; the token bucket does the same job in constant memory but with a slightly different rule (rate plus burst). Expiry is `elapsed >= window_ms`, so a request at 0 stops counting at exactly 1000.
</details>

**Exercise 3.** "Equal jitter" keeps half the backoff and randomizes the other half. Implement it and say what it trades against full jitter.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 3: "equal jitter": half the backoff is kept, the other half is random.
   use reliab::retry::Backoff;

   /// `delay/2 + random % (delay/2 + 1)`, which lies in `delay/2 ..= delay` (rounding the half down).
   pub fn equal_jitter(b: &Backoff, attempt: u32, random: u64) -> u64 {
       let d = b.delay(attempt);
       let half = d / 2;
       half + random % (d - half + 1)
   }
```

Result is in `delay/2 ..= delay`. Compared with full jitter it can never choose a near-zero delay, so it spreads clients less but never retries almost immediately. ✓ the test checks every value in the upper half is reachable (`ex3_covers_the_whole_upper_half`).
</details>

**Exercise 4.** For replication, return the `n` distinct nodes responsible for a key by walking the ring clockwise from the key's position.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 4: the `n` distinct nodes that should hold a key: walk the ring clockwise and skip
   //! positions of nodes already chosen.
   use reliab::hashring::{hash64, point};

   pub fn replicas(nodes: &[&str], vnodes: u32, key: &str, n: usize) -> Vec<String> {
       if n == 0 {
           return Vec::new();
       }
       let mut points: Vec<(u64, &str)> = Vec::new();
       for node in nodes {
           for i in 0..vnodes {
               points.push((point(node, i), node));
           }
       }
       points.sort();
       points.dedup_by_key(|p| p.0);
       let h = hash64(key.as_bytes());
       let start = points.partition_point(|(p, _)| *p < h);
       let mut out: Vec<String> = Vec::new();
       for k in 0..points.len() {
           let node = points[(start + k) % points.len()].1;
           if !out.iter().any(|o| o == node) {
               out.push(node.to_string());
               if out.len() == n {
                   break;
               }
           }
       }
       out
   }
```

✓ The first replica always equals `Ring::lookup`. Skipping repeats matters: with virtual nodes the next position clockwise is often the same node. In real systems you would also skip nodes in the same rack or zone.
</details>

**Exercise 5.** Propagate a deadline: given the whole request's budget, the time already spent, a reserve for the caller's own work and a per-hop cap, return the timeout for the next call, or `None` if nothing worthwhile is left.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 5: deadline propagation: each hop gets what is left, minus what the caller needs to
   //! answer, never more than its own cap.

   /// The timeout for the next call, or `None` if there is no time left worth spending.
   /// `total_ms` is the whole request's budget, `elapsed_ms` is spent so far, `reserve_ms` is kept
   /// for the caller's own work, `cap_ms` is the most this hop may take.
   pub fn hop_timeout(total_ms: u64, elapsed_ms: u64, reserve_ms: u64, cap_ms: u64) -> Option<u64> {
       let left = total_ms.checked_sub(elapsed_ms)?.checked_sub(reserve_ms)?;
       if left == 0 {
           None
       } else {
           Some(left.min(cap_ms))
       }
   }
```

`checked_sub` makes "already past the deadline" a `None` instead of an underflow. A timeout of 1 ms may be technically positive and practically useless; many systems choose a minimum below which they fail fast.
</details>

**Exercise 6.** Implement the nearest-rank percentile. Show that one slow request in a hundred is invisible at p99.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 6: the nearest-rank percentile.

   /// The smallest value such that at least `pct` percent of the samples are at or below it.
   /// `pct` is in `1..=100`. `None` for no samples or a bad `pct`.
   pub fn percentile(samples: &[u64], pct: u32) -> Option<u64> {
       if samples.is_empty() || pct == 0 || pct > 100 {
           return None;
       }
       let mut v = samples.to_vec();
       v.sort_unstable();
       let rank = (v.len() as u64 * u64::from(pct)).div_ceil(100) as usize; // 1-based
       Some(v[rank - 1])
   }
```

✓ With 99 samples at 10 ms and one at 5000 ms, p99 is 10 and p100 is 5000. Percentiles of percentiles (averaging p99 across servers) are not the p99 of the whole; merge histograms instead.
</details>

**Exercise 7.** Write a bulkhead: a total number of slots and a per-class limit, so that one class of caller cannot use them all.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 7: a bulkhead: one caller class cannot use all the slots.
   use std::collections::HashMap;

   pub struct Bulkhead {
       total: usize,
       per_class: usize,
       used: HashMap<String, usize>,
   }

   impl Bulkhead {
       pub fn new(total: usize, per_class: usize) -> Bulkhead {
           Bulkhead {
               total,
               per_class,
               used: HashMap::new(),
           }
       }
       fn in_use(&self) -> usize {
           self.used.values().sum()
       }
       /// Takes a slot for `class` if both the class limit and the total limit allow.
       pub fn enter(&mut self, class: &str) -> bool {
           if self.in_use() >= self.total {
               return false;
           }
           let n = self.used.entry(class.to_string()).or_insert(0);
           if *n >= self.per_class {
               return false;
           }
           *n += 1;
           true
       }
       /// Gives a slot back. Leaving without entering is ignored.
       pub fn leave(&mut self, class: &str) {
           if let Some(n) = self.used.get_mut(class) {
               *n = n.saturating_sub(1);
           }
       }
   }
```

The total is checked before the class so a full system refuses everyone; an extra `leave` is ignored rather than corrupting the count.
</details>

**Exercise 8.** Compute availability from MTBF and MTTR in parts per million.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 8: availability from mean time between failures and mean time to repair.

   /// `MTBF / (MTBF + MTTR)` in parts per million, rounded down. `None` if both are zero.
   pub fn availability_ppm(mtbf_hours: u64, mttr_hours: u64) -> Option<u64> {
       let total = u128::from(mtbf_hours) + u128::from(mttr_hours);
       if total == 0 {
           return None;
       }
       Some((u128::from(mtbf_hours) * 1_000_000 / total) as u64)
   }
```

✓ MTBF 999 h and MTTR 1 h give 999 000 ppm. `u128` keeps `u64::MAX` inputs from overflowing; zero over zero is `None` rather than a division by zero.
</details>

## 19. Debugging challenge

This function is in a production client. It works for months. During a 15-minute outage the process starts crashing in debug builds and, in release builds, retries a dead dependency with tiny delays. What is wrong?

```rust
/// The faulty program.
pub mod buggy {
    /// Delay before retry number `attempt`: `base * 2^attempt`, capped.
    pub fn delay(base_ms: u64, cap_ms: u64, attempt: u32) -> u64 {
        (base_ms * (1u64 << attempt)).min(cap_ms)
    }
}
```

<details markdown="1">
<summary>Solution</summary>

`1u64 << attempt` overflows when `attempt >= 64`, and `base_ms * factor` overflows long before that (`100 × 2^58`). In a debug build both panic; in release the shift is masked and the product wraps, so a long outage yields *short* delays, the opposite of what backoff is for. ✓ The test shows attempts 60 and 64 panic in the buggy version and that the fixed one equals the saturating formula for attempts 0 to 199.

```rust
/// The corrected program.
pub mod fixed {
    /// Delay before retry number `attempt`: `base * 2^attempt`, capped.
    pub fn delay(base_ms: u64, cap_ms: u64, attempt: u32) -> u64 {
        // the doubling saturates, so a long outage keeps the delay at the cap instead of wrapping
        let factor = if attempt >= 64 {
            u64::MAX
        } else {
            1u64 << attempt
        };
        base_ms.saturating_mul(factor).min(cap_ms)
    }
}
```

The lesson is the review point of §4.2: arithmetic on values that grow with *time* needs saturation or a check, and the test must use the extreme case, not the first few attempts.
</details>

## 20. Interview questions

Answers are collapsed.

1. **What is an error budget and how is it used?**
<details markdown="1"><summary>Answer</summary>

The failure the SLO permits (100% minus the objective, over the window). Spent budget justifies slowing releases and prioritizing reliability work; unspent budget is permission to take risk. It turns an argument about opinion into a number.
</details>

2. **Why is retrying dangerous, and how do you make it safe?**
<details markdown="1"><summary>Answer</summary>

It multiplies load on a failing dependency and repeats non-idempotent effects. Retry only safe or idempotent operations, back off exponentially with jitter, stop at a deadline, cap retries with a budget, and retry in one layer only.
</details>

3. **Why add jitter?**
<details markdown="1"><summary>Answer</summary>

Clients that failed together would otherwise retry together at every step, creating synchronized spikes. Randomizing the delay spreads the load.
</details>

4. **Explain a circuit breaker's three states.**
<details markdown="1"><summary>Answer</summary>

Closed: calls pass and consecutive failures are counted. Open: calls are refused until a timer. Half-open: a limited number of probe calls test recovery; success closes, failure re-opens.
</details>

5. **How does an idempotency key work, and what can go wrong?**
<details markdown="1"><summary>Answer</summary>

The client sends a stable key per intent; the server stores key, request fingerprint and result, and replays the result on repeats. Wrong: key reuse with a different body (reject), concurrent duplicates (in-progress state), crash between effect and record (do both in one transaction), unbounded storage (TTL).
</details>

6. **Token bucket versus fixed window?**
<details markdown="1"><summary>Answer</summary>

A fixed window can admit twice the limit across a boundary. A bucket allows a sustained rate plus a bounded burst and smooths the boundary. A sliding log is exact but stores one timestamp per request.
</details>

7. **What is load shedding and when do you do it?**
<details markdown="1"><summary>Answer</summary>

Refusing some work early when demand exceeds capacity so the rest meets its deadline. Do it at the edge, by priority and deadline, before queues grow; unbounded queues convert overload into latency for everyone.
</details>

8. **Why does `hash(key) % N` fail when N changes, and what is the fix?**
<details markdown="1"><summary>Answer</summary>

Almost every key maps to a different server (4005 of 5000 in the lesson's run for 4 to 5). Consistent hashing puts nodes and keys on a ring so only about 1/N of the keys move; virtual nodes even out the load.
</details>

9. **State Little's law and use it.**
<details markdown="1"><summary>Answer</summary>

Average number in the system = arrival rate × average time in the system. 1000 requests per second at 200 ms means about 200 in flight, so you need at least that many connections or workers.
</details>

10. **Why use percentiles instead of the mean?**
<details markdown="1"><summary>Answer</summary>

The mean hides the tail that users feel and that cascades through fan-out calls; a service with a good mean can have a terrible p99. Keep histograms; do not average percentiles.
</details>

11. **What makes an alert good?**
<details markdown="1"><summary>Answer</summary>

It signals user-visible symptoms, is actionable, and has few false positives. Multi-window burn-rate alerts page only when both a short and a long window burn the budget fast.
</details>

12. **What do you look for first in a code review?**
<details markdown="1"><summary>Answer</summary>

What the change is for, then failure paths, limits, arithmetic and units, concurrency, security, whether the tests can fail, and how it is operated. Style is automated.
</details>

13. **Monolith or microservices?**
<details markdown="1"><summary>Answer</summary>

Start with a modular monolith unless independent deployment or scaling is a real need; microservices add network failure, versioning and operational cost. Split along data ownership, not org charts.
</details>

14. **Two 99.9% services in series: what is the availability?**
<details markdown="1"><summary>Answer</summary>

About 99.8% (998 001 ppm), if failures are independent. Redundant components in parallel multiply the *unavailability* instead, but shared causes break the independence assumption.
</details>

15. **How would you design a rate limiter for 1M users?**
<details markdown="1"><summary>Answer</summary>

A token bucket per key in a sharded store (consistent hashing on the key), with eviction of idle keys, an atomic check-and-take, a local approximate layer to cut remote calls, a clear behavior when the store is down (fail open or closed, chosen deliberately), and metrics on rejections.
</details>

## 21. Practical challenge: `reliab`

The crate below is the whole lesson. Build it with `cargo test`; the benchmark is `cargo run --release --example bench`. Layout: `Cargo.toml`, `src/{lib,clock,ratelimit,breaker,retry,idempotency,shed,hashring,slo}.rs`, `tests/{core_tests,service_tests,gap_tests}.rs`, `examples/bench.rs`.

### `Cargo.toml`

```toml
[package]
name = "reliab"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
sha2 = "0.11.1"

[profile.dev.package."*"]
opt-level = 3
```

### `src/lib.rs`

```rust
//! Small, deterministic building blocks of reliable services: time-injected rate limiting, a circuit
//! breaker, retries with a budget, idempotency keys, load shedding, consistent hashing and the
//! arithmetic of service-level objectives. Every component takes the time as an argument, so a test
//! can run an hour of traffic in a millisecond. A teaching model; not audited.
pub mod breaker;
pub mod clock;
pub mod hashring;
pub mod idempotency;
pub mod ratelimit;
pub mod retry;
pub mod shed;
pub mod slo;
```

### `src/clock.rs`

```rust
//! Time as a value that tests control.
use std::cell::Cell;

/// A source of milliseconds since some fixed origin.
pub trait Clock {
    fn now_ms(&self) -> u64;
}

/// A clock that moves only when told to.
#[derive(Debug, Default)]
pub struct ManualClock(Cell<u64>);

impl ManualClock {
    pub fn new(start_ms: u64) -> ManualClock {
        ManualClock(Cell::new(start_ms))
    }
    pub fn advance(&self, ms: u64) {
        self.0.set(self.0.get().saturating_add(ms));
    }
}

impl Clock for ManualClock {
    fn now_ms(&self) -> u64 {
        self.0.get()
    }
}
```

### `src/ratelimit.rs`

```rust
//! A token bucket in integer arithmetic.
//!
//! Tokens are counted in thousandths. A rate of `r` tokens per second adds exactly `r` thousandths
//! per millisecond, so no fraction is ever rounded away.

/// Refuses requests beyond `refill_per_sec` on average, with bursts up to `capacity`.
#[derive(Debug, Clone)]
pub struct TokenBucket {
    capacity: u64,
    refill_per_sec: u64,
    milli_tokens: u128,
    last_ms: u64,
}

impl TokenBucket {
    /// A full bucket at time `now_ms`.
    pub fn new(capacity: u64, refill_per_sec: u64, now_ms: u64) -> TokenBucket {
        TokenBucket {
            capacity,
            refill_per_sec,
            milli_tokens: u128::from(capacity) * 1000,
            last_ms: now_ms,
        }
    }

    fn refill(&mut self, now_ms: u64) {
        // a clock that goes backwards adds nothing and does not rewind the bucket
        let elapsed = now_ms.saturating_sub(self.last_ms);
        self.last_ms = self.last_ms.max(now_ms);
        let max = u128::from(self.capacity) * 1000;
        self.milli_tokens =
            (self.milli_tokens + u128::from(elapsed) * u128::from(self.refill_per_sec)).min(max);
    }

    /// Takes `n` tokens, or says how many milliseconds to wait before `n` tokens exist
    /// (`u64::MAX` if they never will).
    pub fn try_acquire(&mut self, now_ms: u64, n: u64) -> Result<(), u64> {
        self.refill(now_ms);
        let need = u128::from(n) * 1000;
        if need <= self.milli_tokens {
            self.milli_tokens -= need;
            return Ok(());
        }
        if n > self.capacity || self.refill_per_sec == 0 {
            return Err(u64::MAX);
        }
        let missing = need - self.milli_tokens;
        let rate = u128::from(self.refill_per_sec);
        Err(u64::try_from(missing.div_ceil(rate)).unwrap_or(u64::MAX))
    }

    /// Whole tokens available right now.
    pub fn available(&mut self, now_ms: u64) -> u64 {
        self.refill(now_ms);
        (self.milli_tokens / 1000) as u64
    }
}
```

### `src/breaker.rs`

```rust
//! A circuit breaker: stop calling a dependency that is failing, and probe it carefully.

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum State {
    /// Calls flow; `failures` is the current run of consecutive failures.
    Closed { failures: u32 },
    /// Calls are refused until `until_ms`.
    Open { until_ms: u64 },
    /// A few trial calls are allowed; `in_flight` have been admitted and `successes` have returned well.
    HalfOpen { in_flight: u32, successes: u32 },
}

#[derive(Debug, Clone)]
pub struct CircuitBreaker {
    failure_threshold: u32,
    open_ms: u64,
    probes: u32,
    state: State,
}

impl CircuitBreaker {
    /// Opens after `failure_threshold` consecutive failures, stays open `open_ms`, then admits up to
    /// `probes` trial calls; `probes` successes close it again.
    pub fn new(failure_threshold: u32, open_ms: u64, probes: u32) -> CircuitBreaker {
        assert!(failure_threshold > 0 && probes > 0);
        CircuitBreaker {
            failure_threshold,
            open_ms,
            probes,
            state: State::Closed { failures: 0 },
        }
    }

    pub fn state(&self) -> State {
        self.state
    }

    /// May a call be made now? Call `record_success` or `record_failure` for every call admitted.
    pub fn allow(&mut self, now_ms: u64) -> bool {
        match self.state {
            State::Closed { .. } => true,
            State::Open { until_ms } => {
                if now_ms >= until_ms {
                    self.state = State::HalfOpen {
                        in_flight: 1,
                        successes: 0,
                    };
                    true
                } else {
                    false
                }
            }
            State::HalfOpen {
                in_flight,
                successes,
            } => {
                if in_flight + successes < self.probes {
                    self.state = State::HalfOpen {
                        in_flight: in_flight + 1,
                        successes,
                    };
                    true
                } else {
                    false
                }
            }
        }
    }

    pub fn record_success(&mut self) {
        match self.state {
            State::Closed { .. } => self.state = State::Closed { failures: 0 },
            State::HalfOpen {
                in_flight,
                successes,
            } => {
                let successes = successes + 1;
                if successes >= self.probes {
                    self.state = State::Closed { failures: 0 };
                } else {
                    self.state = State::HalfOpen {
                        in_flight: in_flight.saturating_sub(1),
                        successes,
                    };
                }
            }
            State::Open { .. } => {} // a late result from before the circuit opened
        }
    }

    pub fn record_failure(&mut self, now_ms: u64) {
        match self.state {
            State::Closed { failures } => {
                let failures = failures + 1;
                if failures >= self.failure_threshold {
                    self.state = State::Open {
                        until_ms: now_ms.saturating_add(self.open_ms),
                    };
                } else {
                    self.state = State::Closed { failures };
                }
            }
            State::HalfOpen { .. } => {
                self.state = State::Open {
                    until_ms: now_ms.saturating_add(self.open_ms),
                }
            }
            State::Open { .. } => {}
        }
    }
}
```

### `src/retry.rs`

```rust
//! Retries that help: exponential backoff with jitter, a budget so that retries cannot multiply
//! load, and a deadline.

/// Exponential backoff: `base * 2^attempt`, never more than `cap`, never an overflow.
#[derive(Debug, Clone, Copy)]
pub struct Backoff {
    pub base_ms: u64,
    pub cap_ms: u64,
}

impl Backoff {
    /// The delay before retry number `attempt` (0 for the first retry).
    pub fn delay(&self, attempt: u32) -> u64 {
        let factor = if attempt >= 64 {
            u64::MAX
        } else {
            1u64 << attempt
        };
        self.base_ms.saturating_mul(factor).min(self.cap_ms)
    }

    /// "Full jitter": a delay drawn uniformly from `0..=delay(attempt)`, given a random number.
    pub fn full_jitter(&self, attempt: u32, random: u64) -> u64 {
        let d = self.delay(attempt);
        if d == u64::MAX {
            random
        } else {
            random % (d + 1)
        }
    }
}

/// Lets retries be at most `ratio_pct` percent of first attempts (plus a small reserve), so that a
/// failing dependency sees at most a bounded multiple of the normal load.
#[derive(Debug, Clone)]
pub struct RetryBudget {
    milli_tokens: u64,
    max_milli: u64,
    deposit_milli: u64,
}

impl RetryBudget {
    /// `ratio_pct` of 10 means one retry per ten requests; `reserve` retries are available at the start.
    pub fn new(ratio_pct: u64, reserve: u64) -> RetryBudget {
        RetryBudget {
            milli_tokens: reserve * 1000,
            max_milli: reserve.max(1) * 1000,
            deposit_milli: ratio_pct * 10,
        }
    }
    /// Call once per original request.
    pub fn on_request(&mut self) {
        self.milli_tokens = (self.milli_tokens + self.deposit_milli).min(self.max_milli);
    }
    /// Takes one retry from the budget if there is one.
    pub fn try_retry(&mut self) -> bool {
        if self.milli_tokens >= 1000 {
            self.milli_tokens -= 1000;
            true
        } else {
            false
        }
    }
    pub fn available(&self) -> u64 {
        self.milli_tokens / 1000
    }
}

#[derive(Debug, PartialEq, Eq)]
pub enum RetryError<E> {
    /// The error is not worth retrying.
    NotRetryable(E),
    /// Every attempt failed.
    Exhausted(E),
    /// The budget has no retries left.
    BudgetDenied(E),
    /// Waiting for the next attempt would pass the deadline.
    DeadlineExceeded(E),
}

#[derive(Debug, Clone, Copy)]
pub struct RetryPolicy {
    pub max_attempts: u32,
    pub backoff: Backoff,
    /// Total time allowed for all waits.
    pub deadline_ms: u64,
}

/// Runs `op(attempt)` until it succeeds or the policy says stop. `random` supplies the jitter,
/// `sleep` is told how long to wait, `retryable` decides which errors are worth another try.
pub fn run<T, E>(
    policy: &RetryPolicy,
    budget: &mut RetryBudget,
    random: &mut dyn FnMut() -> u64,
    sleep: &mut dyn FnMut(u64),
    retryable: &dyn Fn(&E) -> bool,
    op: &mut dyn FnMut(u32) -> Result<T, E>,
) -> Result<T, RetryError<E>> {
    budget.on_request();
    let mut waited = 0u64;
    let mut attempt = 0u32;
    loop {
        let err = match op(attempt) {
            Ok(v) => return Ok(v),
            Err(e) => e,
        };
        if !retryable(&err) {
            return Err(RetryError::NotRetryable(err));
        }
        if attempt + 1 >= policy.max_attempts {
            return Err(RetryError::Exhausted(err));
        }
        let wait = policy.backoff.full_jitter(attempt, random());
        if waited.saturating_add(wait) > policy.deadline_ms {
            return Err(RetryError::DeadlineExceeded(err));
        }
        if !budget.try_retry() {
            return Err(RetryError::BudgetDenied(err));
        }
        sleep(wait);
        waited += wait;
        attempt += 1;
    }
}
```

### `src/idempotency.rs`

```rust
//! Idempotency keys: the same request, repeated, has one effect and one answer.
use sha2::{Digest, Sha256};
use std::collections::HashMap;

#[derive(Debug, PartialEq, Eq)]
pub enum Begin {
    /// First time (or the earlier attempt is presumed dead): do the work, then call `complete`.
    New,
    /// Another attempt is working on it right now.
    InProgress,
    /// Already done: return this stored result instead of doing the work again.
    Replay(Vec<u8>),
    /// The key was used before with a different request: a client bug, never a retry.
    Conflict,
}

#[derive(Debug, PartialEq, Eq)]
pub enum CompleteError {
    UnknownKey,
    AlreadyDone,
}

enum Slot {
    Working { since_ms: u64 },
    Done { result: Vec<u8>, at_ms: u64 },
}

struct Entry {
    fingerprint: [u8; 32],
    slot: Slot,
}

pub struct IdempotencyStore {
    ttl_ms: u64,
    working_timeout_ms: u64,
    entries: HashMap<String, Entry>,
}

impl IdempotencyStore {
    /// Results are kept `ttl_ms`; an attempt still "working" after `working_timeout_ms` is presumed dead.
    pub fn new(ttl_ms: u64, working_timeout_ms: u64) -> IdempotencyStore {
        IdempotencyStore {
            ttl_ms,
            working_timeout_ms,
            entries: HashMap::new(),
        }
    }

    pub fn len(&self) -> usize {
        self.entries.len()
    }
    pub fn is_empty(&self) -> bool {
        self.entries.is_empty()
    }

    fn expired(&self, e: &Entry, now_ms: u64) -> bool {
        match &e.slot {
            Slot::Done { at_ms, .. } => now_ms.saturating_sub(*at_ms) >= self.ttl_ms,
            Slot::Working { .. } => false,
        }
    }

    /// Decides what to do with a request carrying `key` and body `payload`.
    pub fn begin(&mut self, key: &str, payload: &[u8], now_ms: u64) -> Begin {
        let fingerprint: [u8; 32] = Sha256::digest(payload).into();
        if let Some(e) = self.entries.get(key) {
            if self.expired(e, now_ms) {
                self.entries.remove(key);
            } else if e.fingerprint != fingerprint {
                return Begin::Conflict;
            } else {
                match &e.slot {
                    Slot::Done { result, .. } => return Begin::Replay(result.clone()),
                    Slot::Working { since_ms }
                        if now_ms.saturating_sub(*since_ms) < self.working_timeout_ms =>
                    {
                        return Begin::InProgress;
                    }
                    Slot::Working { .. } => {} // presumed dead: fall through and take over
                }
            }
        }
        self.entries.insert(
            key.to_string(),
            Entry {
                fingerprint,
                slot: Slot::Working { since_ms: now_ms },
            },
        );
        Begin::New
    }

    /// Stores the result of the work started by `begin`.
    pub fn complete(
        &mut self,
        key: &str,
        result: Vec<u8>,
        now_ms: u64,
    ) -> Result<(), CompleteError> {
        let e = self.entries.get_mut(key).ok_or(CompleteError::UnknownKey)?;
        if matches!(e.slot, Slot::Done { .. }) {
            return Err(CompleteError::AlreadyDone);
        }
        e.slot = Slot::Done {
            result,
            at_ms: now_ms,
        };
        Ok(())
    }

    /// The work failed in a way that should be retried from scratch: forget the key.
    pub fn abandon(&mut self, key: &str) {
        if matches!(
            self.entries.get(key).map(|e| &e.slot),
            Some(Slot::Working { .. })
        ) {
            self.entries.remove(key);
        }
    }
}
```

### `src/shed.rs`

```rust
//! A bounded queue that sheds load on purpose: it never grows, it drops work that nobody is
//! waiting for any more, and when it is full the least important work loses.
use std::collections::VecDeque;

struct Item<T> {
    priority: u8,
    deadline_ms: u64,
    value: T,
}

#[derive(Debug, PartialEq, Eq)]
pub enum Admit<T> {
    /// Accepted.
    Queued,
    /// Accepted; this lower-priority item was dropped to make room.
    Evicted(T),
    /// Not accepted: the queue is full of equal or more important work.
    Rejected(T),
}

pub struct SheddingQueue<T> {
    capacity: usize,
    items: VecDeque<Item<T>>,
    expired: u64,
}

impl<T> SheddingQueue<T> {
    pub fn new(capacity: usize) -> SheddingQueue<T> {
        SheddingQueue {
            capacity,
            items: VecDeque::new(),
            expired: 0,
        }
    }
    pub fn len(&self) -> usize {
        self.items.len()
    }
    pub fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
    /// How many items were dropped because their deadline passed.
    pub fn expired(&self) -> u64 {
        self.expired
    }

    fn drop_expired(&mut self, now_ms: u64) {
        let before = self.items.len();
        self.items.retain(|i| i.deadline_ms > now_ms);
        self.expired += (before - self.items.len()) as u64;
    }

    /// Offers an item. A higher `priority` number is more important. An item whose deadline is not
    /// after `now_ms` is dropped at once (counted as expired).
    pub fn push(&mut self, value: T, priority: u8, deadline_ms: u64, now_ms: u64) -> Admit<T> {
        if deadline_ms <= now_ms {
            self.expired += 1;
            return Admit::Rejected(value);
        }
        self.drop_expired(now_ms);
        if self.items.len() < self.capacity {
            self.items.push_back(Item {
                priority,
                deadline_ms,
                value,
            });
            return Admit::Queued;
        }
        // full: the newest of the least important items is the one to lose
        let victim = self
            .items
            .iter()
            .enumerate()
            .rev()
            .min_by_key(|(_, i)| i.priority)
            .map(|(k, i)| (k, i.priority));
        match victim {
            Some((k, p)) if p < priority => {
                let old = self.items.remove(k).expect("index from enumerate");
                self.items.push_back(Item {
                    priority,
                    deadline_ms,
                    value,
                });
                Admit::Evicted(old.value)
            }
            _ => Admit::Rejected(value),
        }
    }

    /// The most important live item; the oldest of equals. Expired items are dropped on the way.
    pub fn pop(&mut self, now_ms: u64) -> Option<T> {
        self.drop_expired(now_ms);
        let best = self
            .items
            .iter()
            .enumerate()
            .max_by_key(|(k, i)| (i.priority, std::cmp::Reverse(*k)))
            .map(|(k, _)| k)?;
        self.items.remove(best).map(|i| i.value)
    }
}
```

### `src/hashring.rs`

```rust
//! Consistent hashing with virtual nodes: adding or removing a node moves only a fair share of keys.
use sha2::{Digest, Sha256};
use std::collections::{BTreeMap, BTreeSet};

/// The first eight bytes of `SHA-256(data)`, big-endian.
pub fn hash64(data: &[u8]) -> u64 {
    let d = Sha256::digest(data);
    u64::from_be_bytes(d[..8].try_into().expect("8"))
}

/// The ring position of virtual node `i` of `node`: the hash of `node`, `#` and the decimal `i`.
pub fn point(node: &str, i: u32) -> u64 {
    hash64(format!("{node}#{i}").as_bytes())
}

pub struct Ring {
    vnodes: u32,
    points: BTreeMap<u64, String>,
    nodes: BTreeSet<String>,
}

impl Ring {
    pub fn new(vnodes: u32) -> Ring {
        assert!(vnodes > 0);
        Ring {
            vnodes,
            points: BTreeMap::new(),
            nodes: BTreeSet::new(),
        }
    }

    pub fn nodes(&self) -> impl Iterator<Item = &str> {
        self.nodes.iter().map(String::as_str)
    }
    pub fn len(&self) -> usize {
        self.nodes.len()
    }
    pub fn is_empty(&self) -> bool {
        self.nodes.is_empty()
    }

    /// Adds a node. Returns false if it was already there. A position already taken by another
    /// node is left to that node, so the ring never depends on insertion order for ties
    /// only when no tie exists; ties between 64-bit hashes are astronomically rare.
    pub fn add(&mut self, node: &str) -> bool {
        if !self.nodes.insert(node.to_string()) {
            return false;
        }
        for i in 0..self.vnodes {
            self.points
                .entry(point(node, i))
                .or_insert_with(|| node.to_string());
        }
        true
    }

    /// Removes a node and its positions. Returns false if it was not there.
    pub fn remove(&mut self, node: &str) -> bool {
        if !self.nodes.remove(node) {
            return false;
        }
        self.points.retain(|_, n| n != node);
        true
    }

    /// The node responsible for `key`: the first position at or after the key's hash, wrapping around.
    pub fn lookup(&self, key: &str) -> Option<&str> {
        let h = hash64(key.as_bytes());
        self.points
            .range(h..)
            .next()
            .or_else(|| self.points.iter().next())
            .map(|(_, n)| n.as_str())
    }
}
```

### `src/slo.rs`

```rust
//! The arithmetic of service-level objectives and capacity, in integers.
//!
//! Ratios are parts per million ("ppm"): 999_000 is 99.9%.

pub const PPM: u64 = 1_000_000;

/// How many of `total` requests may fail and still meet an objective of `slo_ppm`.
pub fn error_budget(slo_ppm: u64, total: u64) -> u64 {
    let allowed_ppm = PPM.saturating_sub(slo_ppm);
    (u128::from(total) * u128::from(allowed_ppm) / u128::from(PPM)) as u64
}

/// How fast the budget is being spent, in thousandths: 1000 means exactly on budget, 14400 means
/// 14.4 times too fast. `None` without traffic or with an objective of 100% (no budget at all).
pub fn burn_rate_milli(slo_ppm: u64, total: u64, bad: u64) -> Option<u64> {
    let allowed_ppm = PPM.checked_sub(slo_ppm).filter(|a| *a > 0)?;
    if total == 0 {
        return None;
    }
    let bad_ppm = u128::from(bad.min(total)) * u128::from(PPM) / u128::from(total);
    Some((bad_ppm * 1000 / u128::from(allowed_ppm)) as u64)
}

/// Pages only when a short and a long window both burn at least `threshold_milli`: the short window
/// shows it is happening now, the long one that it is not a blip.
pub fn should_page(
    short: (u64, u64),
    long: (u64, u64),
    slo_ppm: u64,
    threshold_milli: u64,
) -> bool {
    let hot =
        |w: (u64, u64)| burn_rate_milli(slo_ppm, w.0, w.1).is_some_and(|b| b >= threshold_milli);
    hot(short) && hot(long)
}

/// Little's law: the average number of requests in flight is the arrival rate times the time each
/// takes. Rounded up.
pub fn concurrency(rate_per_sec: u64, latency_ms: u64) -> u64 {
    (u128::from(rate_per_sec) * u128::from(latency_ms)).div_ceil(1000) as u64
}

/// Servers needed for `peak_rps`, each good for `per_server_rps`, keeping `headroom_pct` percent spare.
pub fn servers_needed(peak_rps: u64, per_server_rps: u64, headroom_pct: u64) -> Option<u64> {
    if per_server_rps == 0 {
        return None;
    }
    let with_headroom = u128::from(peak_rps) * u128::from(100 + headroom_pct);
    Some(
        (with_headroom
            .div_ceil(100)
            .div_ceil(u128::from(per_server_rps))) as u64,
    )
}

/// Availability of components that must all work (in series), in ppm.
pub fn serial_ppm(parts: &[u64]) -> u64 {
    parts.iter().fold(PPM, |acc, p| {
        (u128::from(acc) * u128::from(*p) / u128::from(PPM)) as u64
    })
}

/// Availability of redundant components where one is enough (in parallel), in ppm.
pub fn parallel_ppm(parts: &[u64]) -> u64 {
    let all_down = parts.iter().fold(PPM, |acc, p| {
        let down = PPM.saturating_sub(*p);
        (u128::from(acc) * u128::from(down) / u128::from(PPM)) as u64
    });
    PPM - all_down
}
```

### `tests/core_tests.rs`

```rust
use reliab::breaker::{CircuitBreaker, State};
use reliab::clock::{Clock, ManualClock};
use reliab::ratelimit::TokenBucket;
use reliab::retry::{Backoff, RetryBudget, RetryError, RetryPolicy, run};

#[test]
fn clock_moves_only_when_told() {
    let c = ManualClock::new(5);
    assert_eq!(c.now_ms(), 5);
    c.advance(10);
    assert_eq!(c.now_ms(), 15);
    c.advance(u64::MAX);
    assert_eq!(c.now_ms(), u64::MAX);
}

#[test]
fn bucket_allows_burst_then_refuses() {
    let mut b = TokenBucket::new(5, 10, 0);
    for _ in 0..5 {
        assert_eq!(b.try_acquire(0, 1), Ok(()));
    }
    assert_eq!(b.try_acquire(0, 1), Err(100));
}

#[test]
fn bucket_refills_exactly() {
    let mut b = TokenBucket::new(5, 10, 0);
    assert_eq!(b.try_acquire(0, 5), Ok(()));
    assert_eq!(b.available(99), 0);
    assert_eq!(b.available(100), 1);
    assert_eq!(b.try_acquire(100, 1), Ok(()));
    assert_eq!(b.available(100), 0);
}

#[test]
fn bucket_never_exceeds_capacity() {
    let mut b = TokenBucket::new(3, 1000, 0);
    assert_eq!(b.available(1_000_000), 3);
}

#[test]
fn bucket_wait_time_rounds_up_and_is_honest() {
    let mut b = TokenBucket::new(10, 3, 0); // 3 per second: 333.33 ms per token
    assert_eq!(b.try_acquire(0, 10), Ok(()));
    let wait = b.try_acquire(0, 1).unwrap_err();
    assert_eq!(wait, 334);
    assert_eq!(b.try_acquire(wait, 1), Ok(()));
}

#[test]
fn bucket_wait_accounts_for_partial_tokens() {
    let mut b = TokenBucket::new(10, 10, 0);
    assert_eq!(b.try_acquire(0, 10), Ok(()));
    assert_eq!(b.available(150), 1); // 1.5 tokens
    assert_eq!(b.try_acquire(150, 2), Err(50));
}

#[test]
fn bucket_impossible_requests() {
    let mut b = TokenBucket::new(5, 10, 0);
    assert_eq!(b.try_acquire(0, 6), Err(u64::MAX));
    let mut z = TokenBucket::new(5, 0, 0);
    assert_eq!(z.try_acquire(0, 5), Ok(()));
    assert_eq!(z.try_acquire(10, 1), Err(u64::MAX));
}

#[test]
fn bucket_survives_clock_going_backwards() {
    let mut b = TokenBucket::new(5, 10, 1000);
    assert_eq!(b.try_acquire(1000, 5), Ok(()));
    assert_eq!(b.available(0), 0); // earlier time adds nothing
    assert_eq!(b.available(1100), 1); // and did not rewind the origin
}

#[test]
fn bucket_zero_tokens_always_ok() {
    let mut b = TokenBucket::new(1, 1, 0);
    assert_eq!(b.try_acquire(0, 0), Ok(()));
}

#[test]
fn bucket_large_rates_do_not_overflow() {
    let mut b = TokenBucket::new(u64::MAX / 2000, u64::MAX, 0);
    assert!(b.try_acquire(u64::MAX, 1).is_ok());
}

// ---- breaker ----

#[test]
fn breaker_opens_after_consecutive_failures() {
    let mut b = CircuitBreaker::new(3, 1000, 2);
    for t in 0..2 {
        assert!(b.allow(t));
        b.record_failure(t);
    }
    assert_eq!(b.state(), State::Closed { failures: 2 });
    assert!(b.allow(2));
    b.record_failure(2);
    assert_eq!(b.state(), State::Open { until_ms: 1002 });
    assert!(!b.allow(500));
    assert!(!b.allow(1001));
}

#[test]
fn breaker_success_resets_the_run() {
    let mut b = CircuitBreaker::new(3, 1000, 1);
    b.record_failure(0);
    b.record_failure(0);
    b.record_success();
    assert_eq!(b.state(), State::Closed { failures: 0 });
    b.record_failure(0);
    b.record_failure(0);
    assert_eq!(b.state(), State::Closed { failures: 2 });
}

#[test]
fn breaker_half_open_limits_probes_and_closes() {
    let mut b = CircuitBreaker::new(1, 100, 2);
    b.record_failure(0);
    assert!(b.allow(100)); // first probe
    assert_eq!(
        b.state(),
        State::HalfOpen {
            in_flight: 1,
            successes: 0
        }
    );
    assert!(b.allow(100)); // second probe
    assert!(!b.allow(100)); // no third
    b.record_success();
    assert_eq!(
        b.state(),
        State::HalfOpen {
            in_flight: 1,
            successes: 1
        }
    );
    b.record_success();
    assert_eq!(b.state(), State::Closed { failures: 0 });
}

#[test]
fn breaker_half_open_failure_reopens() {
    let mut b = CircuitBreaker::new(1, 100, 2);
    b.record_failure(0);
    assert!(b.allow(100));
    b.record_failure(150);
    assert_eq!(b.state(), State::Open { until_ms: 250 });
    assert!(!b.allow(200));
    assert!(b.allow(250));
}

#[test]
fn breaker_ignores_late_results_while_open() {
    let mut b = CircuitBreaker::new(1, 100, 1);
    b.record_failure(0);
    b.record_success();
    b.record_failure(50);
    assert_eq!(b.state(), State::Open { until_ms: 100 });
}

#[test]
fn breaker_probe_slots_free_up_after_success() {
    let mut b = CircuitBreaker::new(1, 10, 3);
    b.record_failure(0);
    assert!(b.allow(10));
    b.record_success();
    assert!(b.allow(10));
    assert!(b.allow(10));
    assert!(!b.allow(10));
}

#[test]
fn breaker_open_time_saturates() {
    let mut b = CircuitBreaker::new(1, u64::MAX, 1);
    b.record_failure(10);
    assert_eq!(b.state(), State::Open { until_ms: u64::MAX });
}

// ---- retry ----

#[test]
fn backoff_doubles_and_caps() {
    let b = Backoff {
        base_ms: 100,
        cap_ms: 5000,
    };
    let got: Vec<u64> = (0..8).map(|a| b.delay(a)).collect();
    assert_eq!(got, vec![100, 200, 400, 800, 1600, 3200, 5000, 5000]); // py/ref.py
}

#[test]
fn backoff_never_overflows() {
    let b = Backoff {
        base_ms: u64::MAX / 2,
        cap_ms: u64::MAX,
    };
    assert_eq!(b.delay(1), u64::MAX - 1);
    assert_eq!(b.delay(2), u64::MAX);
    assert_eq!(b.delay(63), u64::MAX);
    assert_eq!(b.delay(64), u64::MAX);
    assert_eq!(b.delay(1000), u64::MAX);
    let z = Backoff {
        base_ms: 0,
        cap_ms: 10,
    };
    assert_eq!(z.delay(70), 0);
}

#[test]
fn full_jitter_stays_within_bounds() {
    let b = Backoff {
        base_ms: 100,
        cap_ms: 5000,
    };
    assert_eq!(b.full_jitter(2, 0), 0);
    assert_eq!(b.full_jitter(2, 400), 400);
    assert_eq!(b.full_jitter(2, 401), 0);
    assert_eq!(b.full_jitter(2, 12345), 12345 % 401);
    for r in 0..2000 {
        assert!(b.full_jitter(3, r) <= 800);
    }
    let big = Backoff {
        base_ms: 1,
        cap_ms: u64::MAX,
    };
    assert_eq!(big.full_jitter(64, 77), 77);
}

#[test]
fn budget_limits_retries_to_a_ratio() {
    let mut b = RetryBudget::new(10, 0);
    let mut granted = 0;
    for _ in 0..100 {
        b.on_request();
        if b.try_retry() {
            granted += 1;
        }
    }
    assert_eq!(granted, 10);
}

#[test]
fn budget_reserve_and_cap() {
    let mut b = RetryBudget::new(10, 2);
    assert_eq!(b.available(), 2);
    assert!(b.try_retry());
    assert!(b.try_retry());
    assert!(!b.try_retry());
    for _ in 0..1000 {
        b.on_request();
    }
    assert_eq!(b.available(), 2); // never banks more than the reserve
    let mut z = RetryBudget::new(100, 0);
    for _ in 0..5 {
        z.on_request();
    }
    assert_eq!(z.available(), 1); // a cap of at least one retry
}

fn policy(max: u32, deadline: u64) -> RetryPolicy {
    RetryPolicy {
        max_attempts: max,
        backoff: Backoff {
            base_ms: 100,
            cap_ms: 1000,
        },
        deadline_ms: deadline,
    }
}

#[test]
fn run_succeeds_after_transient_failures() {
    let mut budget = RetryBudget::new(0, 5);
    let mut slept = vec![];
    let r: Result<u32, RetryError<&str>> = run(
        &policy(5, 10_000),
        &mut budget,
        &mut || u64::MAX, // jitter: always the maximum allowed delay
        &mut |ms| slept.push(ms),
        &|_| true,
        &mut |attempt| {
            if attempt < 2 {
                Err("busy")
            } else {
                Ok(attempt)
            }
        },
    );
    assert_eq!(r, Ok(2));
    assert_eq!(slept.len(), 2);
    assert_eq!(slept[0], u64::MAX % 101);
    assert_eq!(slept[1], u64::MAX % 201);
}

#[test]
fn run_stops_on_non_retryable() {
    let mut budget = RetryBudget::new(0, 5);
    let mut calls = 0;
    let r: Result<(), RetryError<&str>> = run(
        &policy(5, 10_000),
        &mut budget,
        &mut || 0,
        &mut |_| {},
        &|e| *e != "fatal",
        &mut |_| {
            calls += 1;
            Err("fatal")
        },
    );
    assert_eq!(r, Err(RetryError::NotRetryable("fatal")));
    assert_eq!(calls, 1);
}

#[test]
fn run_exhausts_attempts() {
    let mut budget = RetryBudget::new(0, 50);
    let mut calls = 0;
    let r: Result<(), RetryError<u8>> = run(
        &policy(3, 10_000),
        &mut budget,
        &mut || 0,
        &mut |_| {},
        &|_| true,
        &mut |_| {
            calls += 1;
            Err(7)
        },
    );
    assert_eq!(r, Err(RetryError::Exhausted(7)));
    assert_eq!(calls, 3);
}

#[test]
fn run_respects_the_budget() {
    let mut budget = RetryBudget::new(0, 1);
    let mut calls = 0;
    let r: Result<(), RetryError<u8>> = run(
        &policy(10, 10_000),
        &mut budget,
        &mut || 0,
        &mut |_| {},
        &|_| true,
        &mut |_| {
            calls += 1;
            Err(1)
        },
    );
    assert_eq!(r, Err(RetryError::BudgetDenied(1)));
    assert_eq!(calls, 2); // one original, one budgeted retry
}

#[test]
fn run_respects_the_deadline_and_does_not_spend_budget_on_it() {
    let mut budget = RetryBudget::new(0, 5);
    let mut slept = vec![];
    let r: Result<(), RetryError<u8>> = run(
        &policy(10, 250),
        &mut budget,
        &mut || 100,
        &mut |ms| slept.push(ms),
        &|_| true,
        &mut |_| Err(1),
    );
    assert_eq!(r, Err(RetryError::DeadlineExceeded(1)));
    assert_eq!(slept, vec![100, 100]); // a third wait of 100 would reach 300 > 250
    assert_eq!(budget.available(), 3); // two retries spent; the refused third cost nothing
}

#[test]
fn run_zero_attempts_policy_still_calls_once() {
    let mut budget = RetryBudget::new(0, 5);
    let mut calls = 0;
    let r: Result<(), RetryError<u8>> = run(
        &policy(0, 1000),
        &mut budget,
        &mut || 0,
        &mut |_| {},
        &|_| true,
        &mut |_| {
            calls += 1;
            Err(1)
        },
    );
    assert_eq!(r, Err(RetryError::Exhausted(1)));
    assert_eq!(calls, 1);
}
```

### `tests/service_tests.rs`

```rust
use reliab::hashring::{Ring, hash64, point};
use reliab::idempotency::{Begin, CompleteError, IdempotencyStore};
use reliab::shed::{Admit, SheddingQueue};
use reliab::slo::*;

// ---- idempotency ----

#[test]
fn first_request_is_new_then_in_progress_then_replay() {
    let mut s = IdempotencyStore::new(1000, 100);
    assert_eq!(s.begin("k", b"pay 5", 0), Begin::New);
    assert_eq!(s.begin("k", b"pay 5", 10), Begin::InProgress);
    assert_eq!(s.complete("k", b"receipt".to_vec(), 20), Ok(()));
    assert_eq!(
        s.begin("k", b"pay 5", 30),
        Begin::Replay(b"receipt".to_vec())
    );
    assert_eq!(
        s.begin("k", b"pay 5", 31),
        Begin::Replay(b"receipt".to_vec())
    );
}

#[test]
fn same_key_different_body_is_a_conflict() {
    let mut s = IdempotencyStore::new(1000, 100);
    assert_eq!(s.begin("k", b"pay 5", 0), Begin::New);
    assert_eq!(s.begin("k", b"pay 6", 1), Begin::Conflict);
    s.complete("k", vec![1], 2).unwrap();
    assert_eq!(s.begin("k", b"pay 6", 3), Begin::Conflict);
}

#[test]
fn different_keys_are_independent() {
    let mut s = IdempotencyStore::new(1000, 100);
    assert_eq!(s.begin("a", b"x", 0), Begin::New);
    assert_eq!(s.begin("b", b"x", 0), Begin::New);
    assert_eq!(s.len(), 2);
}

#[test]
fn dead_attempt_is_taken_over_after_timeout() {
    let mut s = IdempotencyStore::new(1000, 100);
    assert_eq!(s.begin("k", b"x", 0), Begin::New);
    assert_eq!(s.begin("k", b"x", 99), Begin::InProgress);
    assert_eq!(s.begin("k", b"x", 100), Begin::New);
    assert_eq!(s.begin("k", b"x", 150), Begin::InProgress); // the new attempt started at 100
}

#[test]
fn results_expire_after_the_ttl() {
    let mut s = IdempotencyStore::new(1000, 100);
    s.begin("k", b"x", 0);
    s.complete("k", vec![9], 50).unwrap();
    assert_eq!(s.begin("k", b"x", 1049), Begin::Replay(vec![9]));
    assert_eq!(s.begin("k", b"x", 1050), Begin::New);
    // after expiry even a different body is a fresh start, not a conflict
    s.complete("k", vec![8], 1060).unwrap();
    assert_eq!(s.begin("k", b"y", 2060), Begin::New);
}

#[test]
fn complete_errors() {
    let mut s = IdempotencyStore::new(1000, 100);
    assert_eq!(
        s.complete("nope", vec![], 0),
        Err(CompleteError::UnknownKey)
    );
    s.begin("k", b"x", 0);
    s.complete("k", vec![1], 1).unwrap();
    assert_eq!(s.complete("k", vec![2], 2), Err(CompleteError::AlreadyDone));
    assert_eq!(s.begin("k", b"x", 3), Begin::Replay(vec![1])); // the first result stands
}

#[test]
fn abandon_forgets_only_unfinished_work() {
    let mut s = IdempotencyStore::new(1000, 100);
    s.begin("k", b"x", 0);
    s.abandon("k");
    assert!(s.is_empty());
    assert_eq!(s.begin("k", b"x", 1), Begin::New);
    s.complete("k", vec![1], 2).unwrap();
    s.abandon("k");
    assert_eq!(s.len(), 1);
    assert_eq!(s.begin("k", b"x", 3), Begin::Replay(vec![1]));
}

// ---- shedding ----

#[test]
fn queue_is_bounded_and_fifo_within_priority() {
    let mut q = SheddingQueue::new(2);
    assert_eq!(q.push("a", 5, 1000, 0), Admit::Queued);
    assert_eq!(q.push("b", 5, 1000, 0), Admit::Queued);
    assert_eq!(q.push("c", 5, 1000, 0), Admit::Rejected("c"));
    assert_eq!(q.len(), 2);
    assert_eq!(q.pop(0), Some("a"));
    assert_eq!(q.pop(0), Some("b"));
    assert_eq!(q.pop(0), None);
    assert!(q.is_empty());
}

#[test]
fn important_work_evicts_the_least_important_newest() {
    let mut q = SheddingQueue::new(3);
    q.push("low1", 1, 1000, 0);
    q.push("low2", 1, 1000, 0);
    q.push("mid", 5, 1000, 0);
    assert_eq!(q.push("high", 9, 1000, 0), Admit::Evicted("low2"));
    assert_eq!(q.push("equal", 1, 1000, 0), Admit::Rejected("equal"));
    assert_eq!(q.pop(0), Some("high"));
    assert_eq!(q.pop(0), Some("mid"));
    assert_eq!(q.pop(0), Some("low1"));
}

#[test]
fn lower_priority_than_everything_is_rejected() {
    let mut q = SheddingQueue::new(1);
    q.push("mid", 5, 1000, 0);
    assert_eq!(q.push("low", 4, 1000, 0), Admit::Rejected("low"));
    assert_eq!(q.pop(0), Some("mid"));
}

#[test]
fn expired_work_is_dropped_and_counted() {
    let mut q = SheddingQueue::new(2);
    q.push("a", 5, 100, 0);
    q.push("b", 5, 200, 0);
    assert_eq!(q.push("late", 5, 50, 60), Admit::Rejected("late")); // deadline 50 already passed at 60
    assert_eq!(q.expired(), 1);
    // at 100 "a" is dead; its slot is free again
    assert_eq!(q.push("c", 5, 300, 100), Admit::Queued);
    assert_eq!(q.expired(), 2);
    assert_eq!(q.pop(100), Some("b"));
    assert_eq!(q.pop(200), Some("c")); // "b" had been popped; "c" live until 300
}

#[test]
fn deadline_equal_to_now_is_expired() {
    let mut q = SheddingQueue::new(2);
    assert_eq!(q.push("x", 1, 10, 10), Admit::Rejected("x"));
    q.push("y", 1, 10, 9);
    assert_eq!(q.pop(10), None);
    assert_eq!(q.expired(), 2);
}

#[test]
fn pop_prefers_priority_then_age() {
    let mut q = SheddingQueue::new(5);
    q.push("a", 1, 1000, 0);
    q.push("b", 3, 1000, 0);
    q.push("c", 3, 1000, 0);
    q.push("d", 2, 1000, 0);
    assert_eq!(q.pop(0), Some("b"));
    assert_eq!(q.pop(0), Some("c"));
    assert_eq!(q.pop(0), Some("d"));
    assert_eq!(q.pop(0), Some("a"));
}

// ---- hash ring ----

#[test]
fn hash_known_answers() {
    // py/ref.py: first 8 bytes of SHA-256, big endian
    assert_eq!(hash64(b"a"), 0xca97_8112_ca1b_bdca);
    assert_eq!(hash64(b"abc"), 0xba78_16bf_8f01_cfea);
    assert_eq!(point("n1", 0), 0x36ab_20d0_2d20_c204);
    assert_eq!(point("n1", 1), 0x1412_fdcd_00d6_426e);
}

#[test]
fn lookup_known_answers_with_one_vnode() {
    let mut r = Ring::new(1);
    for n in ["n1", "n2", "n3"] {
        assert!(r.add(n));
    }
    // positions: n1 0x36ab.., n2 0x4955.. , n3 0xe45e..; expectations from py/ref.py
    let want = [
        ("alpha", "n3"),
        ("beta", "n1"),
        ("gamma", "n3"),
        ("delta", "n3"),
        ("k1", "n3"),
        ("k2", "n1"),
        ("k3", "n1"),
    ];
    for (k, n) in want {
        assert_eq!(r.lookup(k), Some(n), "key {k}");
    }
}

#[test]
fn empty_ring_and_duplicates() {
    let mut r = Ring::new(4);
    assert_eq!(r.lookup("x"), None);
    assert!(r.is_empty());
    assert!(r.add("a"));
    assert!(!r.add("a"));
    assert_eq!(r.len(), 1);
    assert!(r.remove("a"));
    assert!(!r.remove("a"));
    assert_eq!(r.lookup("x"), None);
    assert_eq!(r.nodes().count(), 0);
}

#[test]
fn single_node_owns_everything() {
    let mut r = Ring::new(8);
    r.add("only");
    for i in 0..100 {
        assert_eq!(r.lookup(&format!("key{i}")), Some("only"));
    }
}

fn owners(r: &Ring, n: usize) -> Vec<String> {
    (0..n)
        .map(|i| r.lookup(&format!("key{i}")).unwrap().to_string())
        .collect()
}

#[test]
fn adding_a_node_moves_keys_only_to_the_new_node() {
    let mut r = Ring::new(100);
    for n in ["a", "b", "c", "d"] {
        r.add(n);
    }
    let before = owners(&r, 5000);
    r.add("e");
    let after = owners(&r, 5000);
    let mut moved = 0;
    for (b, a) in before.iter().zip(&after) {
        if b != a {
            assert_eq!(a, "e");
            moved += 1;
        }
    }
    // about 1/5 of 5000 = 1000 should move; allow generous slack, but far below "rehash everything"
    assert!((600..1500).contains(&moved), "moved {moved}");
}

#[test]
fn removing_a_node_moves_only_its_keys() {
    let mut r = Ring::new(100);
    for n in ["a", "b", "c", "d"] {
        r.add(n);
    }
    let before = owners(&r, 5000);
    r.remove("c");
    let after = owners(&r, 5000);
    for (b, a) in before.iter().zip(&after) {
        if b != "c" {
            assert_eq!(a, b);
        } else {
            assert_ne!(a, "c");
        }
    }
}

#[test]
fn more_virtual_nodes_balance_better() {
    let spread = |v: u32| {
        let mut r = Ring::new(v);
        for n in ["a", "b", "c", "d", "e"] {
            r.add(n);
        }
        let mut counts = std::collections::HashMap::new();
        for o in owners(&r, 10_000) {
            *counts.entry(o).or_insert(0u32) += 1;
        }
        let max = *counts.values().max().unwrap();
        let min = *counts.values().min().unwrap();
        (max, min)
    };
    let (max1, min1) = spread(1);
    let (max200, min200) = spread(200);
    assert!(
        max200 - min200 < max1 - min1,
        "{:?} vs {:?}",
        (max200, min200),
        (max1, min1)
    );
    assert!(max200 < 2800 && min200 > 1300, "{max200} {min200}"); // ideal 2000 each
}

#[test]
fn ring_is_independent_of_insertion_order() {
    let mut r1 = Ring::new(20);
    let mut r2 = Ring::new(20);
    for n in ["a", "b", "c"] {
        r1.add(n);
    }
    for n in ["c", "a", "b"] {
        r2.add(n);
    }
    assert_eq!(owners(&r1, 500), owners(&r2, 500));
}

// ---- slo ----

#[test]
fn error_budget_values() {
    assert_eq!(error_budget(999_000, 1_000_000), 1000);
    assert_eq!(error_budget(999_900, 10_000_000), 1000);
    assert_eq!(error_budget(PPM, 1_000_000), 0);
    assert_eq!(error_budget(0, 500), 500);
    assert_eq!(error_budget(2 * PPM, 500), 0);
    assert_eq!(
        error_budget(999_000, u64::MAX),
        (u128::from(u64::MAX) * 1000 / 1_000_000) as u64
    );
}

#[test]
fn burn_rate_values() {
    // 14 bad of 1000 against 99.9%: 1.4% / 0.1% = 14.0x
    assert_eq!(burn_rate_milli(999_000, 1000, 14), Some(14_000));
    assert_eq!(burn_rate_milli(999_000, 1000, 1), Some(1000));
    assert_eq!(burn_rate_milli(999_000, 1000, 0), Some(0));
    assert_eq!(burn_rate_milli(999_000, 0, 0), None);
    assert_eq!(burn_rate_milli(PPM, 1000, 1), None);
    assert_eq!(burn_rate_milli(PPM + 5, 1000, 1), None);
    assert_eq!(burn_rate_milli(990_000, 100, 500), Some(100_000)); // bad is clamped to total: 100% / 1%
}

#[test]
fn paging_needs_both_windows() {
    let slo = 999_000;
    let hot = (1000, 15); // 15x
    let cold = (1000, 2); // 2x
    assert!(should_page(hot, hot, slo, 14_400 - 400));
    assert!(!should_page(hot, cold, slo, 14_000));
    assert!(!should_page(cold, hot, slo, 14_000));
    assert!(should_page((100, 2), (100, 2), slo, 14_000)); // 20x each
    assert!(!should_page((0, 0), hot, slo, 1)); // no traffic in a window: no page
    assert!(should_page((1000, 14), (1000, 14), slo, 14_000)); // exactly at the threshold counts
    assert!(!should_page((1000, 13), (1000, 14), slo, 14_000));
}

#[test]
fn littles_law_and_capacity() {
    assert_eq!(concurrency(1000, 200), 200);
    assert_eq!(concurrency(1, 1), 1); // rounds up
    assert_eq!(concurrency(0, 500), 0);
    assert_eq!(concurrency(3, 100), 1);
    assert_eq!(servers_needed(1000, 100, 0), Some(10));
    assert_eq!(servers_needed(1000, 100, 30), Some(13));
    assert_eq!(servers_needed(1001, 100, 0), Some(11));
    assert_eq!(servers_needed(10, 0, 0), None);
    assert_eq!(servers_needed(0, 100, 50), Some(0));
}

#[test]
fn availability_composition() {
    assert_eq!(serial_ppm(&[999_000, 999_000]), 998_001);
    assert_eq!(serial_ppm(&[]), PPM);
    assert_eq!(parallel_ppm(&[900_000, 900_000]), 990_000);
    assert_eq!(parallel_ppm(&[]), 0);
    assert_eq!(parallel_ppm(&[999_000]), 999_000);
    assert!(serial_ppm(&[999_900; 5]) < 999_900);
}
```

### `tests/gap_tests.rs`

```rust
//! Tests added after the first mutation run showed what the ordinary tests did not pin down.
use reliab::breaker::CircuitBreaker;
use reliab::hashring::Ring;
use reliab::ratelimit::TokenBucket;
use reliab::retry::{Backoff, RetryBudget, RetryError, RetryPolicy, run};
use reliab::slo::burn_rate_milli;

#[test]
fn bucket_request_for_the_whole_capacity_waits_instead_of_failing_forever() {
    let mut b = TokenBucket::new(5, 10, 0);
    assert_eq!(b.try_acquire(0, 5), Ok(()));
    assert_eq!(b.try_acquire(0, 5), Err(500)); // possible, just not yet
    assert_eq!(b.try_acquire(500, 5), Ok(()));
}

fn policy(max: u32, deadline: u64) -> RetryPolicy {
    RetryPolicy {
        max_attempts: max,
        backoff: Backoff {
            base_ms: 100,
            cap_ms: 1000,
        },
        deadline_ms: deadline,
    }
}

#[test]
fn waiting_exactly_the_deadline_is_allowed() {
    let mut budget = RetryBudget::new(0, 5);
    let mut slept = vec![];
    let r: Result<(), RetryError<u8>> = run(
        &policy(3, 200),
        &mut budget,
        &mut || 100,
        &mut |ms| slept.push(ms),
        &|_| true,
        &mut |_| Err(1),
    );
    assert_eq!(r, Err(RetryError::Exhausted(1)));
    assert_eq!(slept, vec![100, 100]); // 200 of 200 used; nothing was refused
}

#[test]
fn every_request_earns_retry_credit() {
    let mut budget = RetryBudget::new(100, 0);
    let r: Result<u8, RetryError<u8>> = run(
        &policy(3, 10_000),
        &mut budget,
        &mut || 0,
        &mut |_| {},
        &|_| true,
        &mut |attempt| if attempt == 0 { Err(1) } else { Ok(2) },
    );
    assert_eq!(r, Ok(2)); // a 100% ratio funds one retry per request, with no reserve at all
}

#[test]
fn a_key_that_hashes_exactly_to_a_position_belongs_to_that_position() {
    let mut r = Ring::new(1);
    r.add("n1");
    r.add("n2");
    r.add("n3");
    // the key "n2#0" hashes to the same 64 bits as virtual node 0 of n2
    assert_eq!(r.lookup("n2#0"), Some("n2"));
    assert_eq!(r.lookup("n1#0"), Some("n1"));
    assert_eq!(r.lookup("n3#0"), Some("n3"));
}

#[test]
#[should_panic]
fn a_ring_without_virtual_nodes_is_refused() {
    let _ = Ring::new(0);
}

#[test]
#[should_panic]
fn a_breaker_that_can_never_open_is_refused() {
    let _ = CircuitBreaker::new(0, 10, 1);
}

#[test]
#[should_panic]
fn a_breaker_without_probes_is_refused() {
    let _ = CircuitBreaker::new(1, 10, 0);
}

#[test]
fn burn_rate_works_for_a_very_strict_objective() {
    // 1 ppm of error allowed; one failure in a million is exactly on budget
    assert_eq!(burn_rate_milli(999_999, 1_000_000, 1), Some(1000));
}
```

### `examples/bench.rs`

```rust
use reliab::breaker::CircuitBreaker;
use reliab::hashring::Ring;
use reliab::idempotency::IdempotencyStore;
use reliab::ratelimit::TokenBucket;
use std::hint::black_box;
use std::time::Instant;

fn time<F: FnMut()>(name: &str, n: u64, mut f: F) {
    let t = Instant::now();
    f();
    let ns = t.elapsed().as_nanos() as f64 / n as f64;
    println!("{name:<32} {ns:>10.1} ns/op");
}

fn main() {
    let n = 1_000_000u64;
    let mut b = TokenBucket::new(1_000_000, 1_000_000, 0);
    time("token bucket try_acquire", n, || {
        for i in 0..n {
            black_box(b.try_acquire(i, 1).is_ok());
        }
    });
    let mut cb = CircuitBreaker::new(5, 1000, 2);
    time("breaker allow+success", n, || {
        for i in 0..n {
            if cb.allow(i) {
                cb.record_success();
            }
        }
    });
    let mut s = IdempotencyStore::new(1_000_000, 1000);
    let keys: Vec<String> = (0..10_000).map(|i| format!("key-{i}")).collect();
    time("idempotency begin (new+replay)", 200_000, || {
        for r in 0..20 {
            for (i, k) in keys.iter().enumerate() {
                black_box(s.begin(k, b"payload", r));
                if r == 0 {
                    s.complete(k, vec![i as u8], 0).unwrap();
                }
            }
        }
    });
    let mut ring = Ring::new(100);
    for i in 0..10 {
        ring.add(&format!("node{i}"));
    }
    time("ring lookup (10 nodes x 100)", n / 4, || {
        for i in 0..n / 4 {
            black_box(ring.lookup(&format!("key{i}")));
        }
    });
    time("ring add node (100 vnodes)", 100, || {
        for i in 0..100 {
            let mut r = Ring::new(100);
            black_box(r.add(&format!("n{i}")));
        }
    });
}
```

### `py/ref.py` (independent known answers)

```python
import hashlib
def h64(b): return int.from_bytes(hashlib.sha256(b).digest()[:8],'big')
print("hash64(a)=",hex(h64(b"a")))
print("point(n1,0)=",hex(h64(b"n1#0")),"point(n1,1)=",hex(h64(b"n1#1")))
# ring with n1,n2,n3 vnodes=1 lookup of keys
pts=sorted((h64(f"{n}#0".encode()),n) for n in ["n1","n2","n3"])
print(pts)
def look(k,pts):
    h=h64(k.encode())
    for p,n in pts:
        if p>=h: return n
    return pts[0][1]
for k in ["alpha","beta","gamma","delta","k1","k2","k3"]: print(k,hex(h64(k.encode())),look(k,pts))
print("sha256(abc) first8",hex(h64(b"abc")))
# backoff
print([min(100*2**a,5000) for a in range(8)])
# burn rate: slo 99.9%, 1000 total 14 bad -> 14/1000=1.4% /0.1% =14.0
print(14000000//1000*1000//1000)
```

### Checks and mutation testing

**Mutation check ✓.** I made 86 small source edits (a "mutant") across eight source modules, each in its own copy of the crate, and ran the whole test suite against each. **85 were caught by a failing test and 1 is equivalent (M69); none fails to compile and none survives unexplained.** **What the first pass found.** I ran the mutants against the 54 tests of `core_tests.rs` and `service_tests.rs` before writing any gap test, so every gap test below was written *after* seeing a survivor; I did not predict them. Of 86 mutants, 67 were caught at once and 19 were not. **Thirteen of the nineteen were mistakes in my mutants, not in the tests**: eleven patterns no longer matched because `rustfmt` had re-wrapped the source (the harness reports these as "invalid", and I rewrote them against the formatted text) and two did not compile (`u128::from(100)` is ambiguous without a type, and `.div(1000)` needs the `Div` trait in scope). After fixing the patterns, all thirteen were caught except one, which is equivalent (below). **Six were real gaps**, each closed with one test in `gap_tests.rs`: a request for exactly the bucket's capacity with an empty bucket (the `>` versus `>=` bound), a retry wait that lands exactly on the deadline, a retry budget with a 100% ratio and no reserve (the per-request deposit had no test that needed it), a key that hashes exactly onto a ring position (`range(h..)` versus `range(h+1..)`), the ring constructor with zero virtual nodes (a precondition no test violated), and a very strict objective (999 999 ppm) for the burn rate. I added two more tests for the two breaker constructor preconditions at the same time as the ring one; they pinned no surviving mutant. The pattern in the real gaps is the usual one: every one is a **boundary** (equal to the capacity, equal to the deadline, equal to a position, a ratio of exactly 100%, a divisor of exactly 1). The ordinary tests exercised both sides of each boundary and missed the point on it. The column "change" is generated from the edit itself (the part that differs, with a little context, whitespace collapsed); `…` marks omitted text. The tests ran in the dev profile with optimized dependencies.

| # | module | change | result |
|---|---|---|---|
| M1 | `ratelimit` | `self.last_ms = self.last_ms.max(now_ms)` → `self.last_ms = now_ms` | caught by `bucket_survives_clock_going_backwards` |
| M2 | `ratelimit` | `now_ms.saturat` → `now_ms.wrapp` | caught by `bucket_survives_clock_going_backwards` |
| M3 | `ratelimit` | `if need <=` → `if need <` | caught by `bucket_impossible_requests` |
| M4 | `ratelimit` | `…self.capacity \|\|` → `…self.capacity &&` | caught by `bucket_impossible_requests` |
| M5 | `ratelimit` | `missing.div_ceil(rate)` → `missing / rate` | caught by `bucket_wait_time_rounds_up_and_is_honest` |
| M6 | `ratelimit` | `u128::from(elapsed) *` → `u128::from(elapsed) +` | caught by `bucket_allows_burst_then_refuses` |
| M7 | `ratelimit` | `.min(max)` → `∅` | caught by `bucket_never_exceeds_capacity` |
| M8 | `ratelimit` | `self.milli_tokens -= need` → `self.milli_tokens = 0` | caught by `bucket_allows_burst_then_refuses` |
| M9 | `ratelimit` | `n >` → `n >=` | caught by `bucket_request_for_the_whole_capacity_waits_instead_of_failing_forever` |
| M10 | `ratelimit` | `…self.milli_tokens / 1000` → `…self.milli_tokens / 100` | caught by `bucket_never_exceeds_capacity` |
| M11 | `ratelimit` | `…missing = need - self.milli_tokens` → `…missing = need` | caught by `bucket_wait_accounts_for_partial_tokens` |
| M12 | `breaker` | `failures >=` → `failures >` | caught by `breaker_half_open_limits_probes_and_closes` |
| M13 | `breaker` | `if now_ms >=` → `if now_ms >` | caught by `breaker_half_open_failure_reopens` |
| M14 | `breaker` | `…+ successes <` → `…+ successes <=` | caught by `breaker_half_open_limits_probes_and_closes` |
| M15 | `breaker` | `State::Closed { .. } => self.state = State::Closed { failures: 0 },` → `State::Closed { .. } => {}` | caught by `breaker_success_resets_the_run` |
| M16 | `breaker` | `if successes >=` → `if successes >` | caught by `breaker_half_open_limits_probes_and_closes` |
| M17 | `breaker` | `in_flight: in_flight.saturating_sub(1)` → `in_flight` | caught by `breaker_half_open_limits_probes_and_closes` |
| M18 | `breaker` | `State::HalfOpen { .. } => {` → `State::HalfOpen { .. } => { return;` | caught by `breaker_half_open_failure_reopens` |
| M19 | `breaker` | `State::Open { .. } => {} // a late result from before the circuit o…` → `State::Open { .. } => self.state = State::Closed { failures: 0 },` | caught by `breaker_ignores_late_results_while_open` |
| M20 | `breaker` | `State::Open { .. } => {}` → `State::Open { .. } => self.state = State::Open { until_ms: now_ms.s…` | caught by `breaker_ignores_late_results_while_open` |
| M21 | `breaker` | `in_flight: 1` → `in_flight: 0` | caught by `breaker_half_open_limits_probes_and_closes` |
| M22 | `breaker` | `until_ms: now_ms.saturating_add(self.open_ms)` → `until_ms: now_ms` | caught by `breaker_open_time_saturates` |
| M23 | `retry` | `base_ms.saturat` → `base_ms.wrapp` | caught by `backoff_never_overflows` |
| M24 | `retry` | `if attempt >=` → `if attempt >` | caught by `backoff_never_overflows` |
| M25 | `retry` | `random % (d + 1)` → `random % d` | caught by `full_jitter_stays_within_bounds` |
| M26 | `retry` | `if d == u64::MAX` → `if false` | caught by `full_jitter_stays_within_bounds` |
| M27 | `retry` | `self.milli_tokens >=` → `self.milli_tokens >` | caught by `budget_reserve_and_cap` |
| M28 | `retry` | `…ratio_pct * 10` → `…ratio_pct * 100` | caught by `budget_limits_retries_to_a_ratio` |
| M29 | `retry` | `reserve.max(1)` → `reserve` | caught by `budget_limits_retries_to_a_ratio` |
| M30 | `retry` | `attempt + 1 >=` → `attempt + 1 >` | caught by `run_exhausts_attempts` |
| M31 | `retry` | `waited.saturating_add(wait) >` → `waited.saturating_add(wait) >=` | caught by `waiting_exactly_the_deadline_is_allowed` |
| M32 | `retry` | `waited +` → `waited ` | caught by `run_respects_the_deadline_and_does_not_spend_budget_on_it` |
| M33 | `retry` | `if !` → `if ` | caught by `run_exhausts_attempts` |
| M34 | `retry` | `budget.on_request(); ` → `∅` | caught by `every_request_earns_retry_credit` |
| M35 | `retry` | `self.milli_tokens -= 100` → `self.milli_tokens = ` | caught by `budget_reserve_and_cap` |
| M36 | `retry` | `.min(self.cap_ms)` → `∅` | caught by `backoff_doubles_and_caps` |
| M37 | `retry` | `if !budget.try_retry() { return Err(RetryError::BudgetDenied(err)); } ` → `∅` | caught by `run_respects_the_budget` |
| M38 | `retry` | `…policy.backoff.full_jitter(attempt, random()` → `…policy.backoff.delay(attempt` | caught by `run_respects_the_deadline_and_does_not_spend_budget_on_it` |
| M39 | `idempotency` | `…now_ms.saturating_sub(*at_ms) >=` → `…now_ms.saturating_sub(*at_ms) >` | caught by `results_expire_after_the_ttl` |
| M40 | `idempotency` | `} else if e.fingerprint != fingerprint` → `} else if false` | caught by `same_key_different_body_is_a_conflict` |
| M41 | `idempotency` | `…*since_ms) <` → `…*since_ms) <=` | caught by `dead_attempt_is_taken_over_after_timeout` |
| M42 | `idempotency` | `if matches!(e.slot, Slot::Done { .. }) { return Err(CompleteError::…` → `∅` | caught by `complete_errors` |
| M43 | `idempotency` | `…CompleteError::UnknownKey` → `…CompleteError::AlreadyDone` | caught by `complete_errors` |
| M44 | `idempotency` | `Some(Slot::Working { .. }` → `Some(_` | caught by `abandon_forgets_only_unfinished_work` |
| M45 | `idempotency` | `Slot::Working { .. } => fals` → `Slot::Working { .. } => tru` | caught by `dead_attempt_is_taken_over_after_timeout` |
| M46 | `idempotency` | `Slot::Working { .. } => {} // presumed dead: fall through and take …` → `Slot::Working { .. } => return Begin::InProgress,` | caught by `dead_attempt_is_taken_over_after_timeout` |
| M47 | `idempotency` | `Slot::Done { result, .. } => return Begin::Replay(result.clone()` → `Slot::Done { .. } => return Begin::Replay(vec![]` | caught by `abandon_forgets_only_unfinished_work` |
| M48 | `idempotency` | `Sha256::digest(payload).into()` → `[0u8; 32]` | caught by `same_key_different_body_is_a_conflict` |
| M49 | `idempotency` | `self.entries.remove(key); ` → `self.entries.remove(key); return Begin::New; ` | caught by `results_expire_after_the_ttl` |
| M50 | `shed` | `…deadline_ms <=` → `…deadline_ms <` | caught by `deadline_equal_to_now_is_expired` |
| M51 | `shed` | `…i.deadline_ms >` → `…i.deadline_ms >=` | caught by `deadline_equal_to_now_is_expired` |
| M52 | `shed` | `…self.items.len() <` → `…self.items.len() <=` | caught by `important_work_evicts_the_least_important_newest` |
| M53 | `shed` | `…k, p)) if p <` → `…k, p)) if p <=` | caught by `important_work_evicts_the_least_important_newest` |
| M54 | `shed` | `.rev()` → `∅` | caught by `important_work_evicts_the_least_important_newest` |
| M55 | `shed` | `…i.priority, std::cmp::Reverse(*k)` → `…i.priority, *k` | caught by `expired_work_is_dropped_and_counted` |
| M56 | `shed` | `self.expired += (before - self.items.len()) as u64` → `self.expired += 1` | caught by `deadline_equal_to_now_is_expired` |
| M57 | `shed` | `self.drop_expired(now_ms); ` → `∅` | caught by `expired_work_is_dropped_and_counted` |
| M58 | `shed` | `self.drop_expired(now_ms); ` → `∅` | caught by `deadline_equal_to_now_is_expired` |
| M59 | `shed` | `self.expired += 1; ` → `∅` | caught by `deadline_equal_to_now_is_expired` |
| M60 | `shed` | `self.items.push_back` → `drop` | caught by `important_work_evicts_the_least_important_newest` |
| M61 | `hashring` | `d[..8` → `d[8..16` | caught by `hash_known_answers` |
| M62 | `hashring` | `format!("{node}#` → `format!("{node}` | caught by `hash_known_answers` |
| M63 | `hashring` | `for i in 0` → `for i in 1` | caught by `lookup_known_answers_with_one_vnode` |
| M64 | `hashring` | `.range(h` → `.range((h+1)` | caught by `a_key_that_hashes_exactly_to_a_position_belongs_to_that_position` |
| M65 | `hashring` | `…self.points.iter().next` → `…self.points.iter().next_back` | caught by `lookup_known_answers_with_one_vnode` |
| M66 | `hashring` | `self.points.retain(\|_, n\| n !` → `self.points.retain(\|_, n\| n =` | caught by `empty_ring_and_duplicates` |
| M67 | `hashring` | `if !self.nodes.insert(node.to_string()) { return false; }` → `self.nodes.insert(node.to_string());` | caught by `empty_ring_and_duplicates` |
| M68 | `hashring` | `if !self.nodes.remove(node) { return false; }` → `self.nodes.remove(node);` | caught by `empty_ring_and_duplicates` |
| M69 | `hashring` | `.` → `.and_modify(\|n\| *n = node.to_string()).` | **equivalent**: `or_insert_with` keeps the first node at a position and the mutant replaces it with the later one; the two differ only when two different virtual-node labels hash to the same 64-bit position, which needs a SHA-256 prefix collision and which no test can construct |
| M70 | `hashring` | `assert!(vnodes > 0);` → `∅` | caught by `a_ring_without_virtual_nodes_is_refused` |
| M71 | `slo` | `PPM.saturating_sub` → `PPM.saturating_add` | caught by `error_budget_values` |
| M72 | `slo` | `filter(\|a\| *a > 0` → `filter(\|a\| *a > 1` | caught by `burn_rate_works_for_a_very_strict_objective` |
| M73 | `slo` | `bad.min(total)` → `bad` | caught by `burn_rate_values` |
| M74 | `slo` | `bad_ppm * 1000` → `bad_ppm * 100` | caught by `burn_rate_values` |
| M75 | `slo` | `b >=` → `b >` | caught by `paging_needs_both_windows` |
| M76 | `slo` | `hot(short) &&` → `hot(short) \|\|` | caught by `paging_needs_both_windows` |
| M77 | `slo` | `…u128::from(latency_ms)).div_ceil(` → `…u128::from(latency_ms) / ` | caught by `littles_law_and_capacity` |
| M78 | `slo` | `…rate_per_sec) *` → `…rate_per_sec) +` | caught by `littles_law_and_capacity` |
| M79 | `slo` | `u128::from(100 + headroom_pct` → `u128::from(100u64` | caught by `littles_law_and_capacity` |
| M80 | `slo` | `.div_ceil(u128::from(per_server_rps)` → `/ u128::from(per_server_rps` | caught by `littles_law_and_capacity` |
| M81 | `slo` | `if per_server_rps == 0 { return None; }` → `∅` | caught by `littles_law_and_capacity` |
| M82 | `slo` | `PPM - ` → `∅` | caught by `availability_composition` |
| M83 | `slo` | `(u128::from(acc) * u128::from(*p) / u128::from(PPM)) as u64` → `acc.min(*p)` | caught by `availability_composition` |
| M84 | `slo` | `let down = PPM.saturating_sub(*p)` → `let down = *p` | caught by `availability_composition` |
| M85 | `slo` | `if total == 0 { return None; }` → `∅` | caught by `burn_rate_values` |
| M86 | `clock` | `self.0.set(self.0.get().saturating_add(ms)` → `self.0.set(ms` | caught by `clock_moves_only_when_told` |

What the table does not show: it tests *these* edits, not every possible bug; the one equivalent mutant could hide a gap if my reasoning about it is wrong; the `lib.rs` file has no mutants because it only declares modules; the `ManualClock` has one mutant because it is a test helper; and passing every mutant says nothing about behavior under concurrency, with a real clock, or in production.

### Design notes

- **Everything takes time as an argument.** There is no `Instant::now()` in the crate; `ManualClock` exists for tests that want a clock object.
- **Integers only.** Thousandths of a token, thousandths of a burn rate, parts per million. No float appears, so no rounding surprise and no `NaN`.
- **Outcomes are types.** `Begin`, `Admit`, `RetryError`, `State` say what happened.
- **Nothing is shared across threads.** To share a limiter, wrap it in a `Mutex` and keep the critical section to the call; the model deliberately does not choose a locking strategy.

## Key takeaways

- Keep the domain free of I/O and clocks and the tests become fast and exact; enforce the boundary with crates where it is worth the cost.
- A review looks for failure paths, limits, arithmetic and unfalsifiable tests before style.
- Every protection has a number: a rate, a threshold, a budget, a deadline, a TTL, a bound. Pick them, write them down, test the boundary on both sides.
- Retries need backoff, jitter, a deadline, a budget, idempotency and a single layer. Without them they are an outage multiplier.
- A breaker, a bulkhead and a shedding queue exist so that a failure stays small.
- Consistent hashing moves about 1/N of the keys instead of nearly all; virtual nodes make the load even.
- An SLO is a number; its complement is a budget; alerting on burn over two windows pages on real problems.
- Mutation testing result: 85 of 86 mutants were caught and 1 is equivalent; the 8 tests added afterward (6 closing real gaps, 2 for constructor preconditions) are all at boundaries (equal to a capacity, a deadline, a ring position, a ratio of 100%), and thirteen "survivors" were my own broken mutants, not weak tests.

**Connections.** Backward: [Lesson 52](52_web3_reliability.md) used retries and idempotency for off-chain services; this lesson generalizes them and adds the budget and the breaker. [Lesson 58](58_cross_chain_relayers.md) built a relayer whose idempotency was the retry strategy; the idempotency store here is the general form. [Lesson 32](32_cancellation_streams_backpressure.md) introduced backpressure; shedding is its policy side. Forward: Lesson 60 turns these topics into interview answers and capstone projects.

## References

Everything here is from memory; I could not reach documentation sites from the sandbox. Check current versions before relying on a detail.

- Google, *Site Reliability Engineering* and *The Site Reliability Workbook* (SLOs, error budgets, multi-window burn-rate alerts).
- Marc Brooker's writing on timeouts, retries and backoff with jitter (the "full jitter" name), from memory of posts on the AWS Architecture Blog.
- Karger et al., "Consistent hashing and random trees" (1997).
- Michael Nygard, *Release It!* (circuit breaker, bulkhead, timeouts).
- Stripe's public API documentation on idempotency keys (the practice of a client-supplied key and replayed responses).
- Little, "A proof for the queuing formula L = λW" (1961).
- Alistair Cockburn, "Hexagonal architecture" (ports and adapters); Michael Nygard, "Documenting architecture decisions" (ADRs).
- Rust: the documentation of `u64::div_ceil`, `std::cell::Cell`, and the `tower` and `governor` crates, none of which this lesson uses.

**Unchecked or from memory.** The architecture vocabulary and citations of §2 and the reference list; the 14.4× burn-rate figure; the claim that postmortems usually find a change, a retry storm or saturation; the descriptions of how real systems use rings and idempotency keys; CAP and PACELC wording; library names. Not tested: any concurrent use, any real clock, any network, any persistence, any real service.
