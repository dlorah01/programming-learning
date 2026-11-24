---
title: "Lesson 51 — RPC, nodes, indexers, transaction monitoring, serialization"
---

# Lesson 51 — RPC, nodes, indexers, transaction monitoring, serialization

**Module 16 · Blockchain and Web3 infrastructure, chain-agnostic (first lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `serde 1.0.229` (with `derive`), `serde_json 1.0.154` and `sha2 0.11.0` (22 packages in `Cargo.lock`, the crate itself included)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** Services that talk to a blockchain, whichever one, have the same shape: they ask a **node** questions over **RPC**, they follow the chain with an **indexer** that must survive **reorganizations**, they **monitor transactions** until they are final, and everything that crosses a boundary is **serialized**. The crate `rpclab` builds each of those at teaching scale, against a **mock in-memory chain**: strict serialization helpers, a **JSON-RPC 2.0** server and client codec with limits and batch matching, a **retrying, failing-over client** with backoff, jitter and deadlines, a **chain indexer** that detects and follows reorgs, and a **transaction watcher** that emits each state change once. It is teaching code: **not audited, not a node, not a wallet, and not tied to any real network**. Not covered: real chains' wire formats and method names (the names here, like `getBlockByNumber`, are my own), WebSocket subscriptions and log filters, mempool and fee APIs, transaction construction and signing (Lesson 48 has the primitives), a real HTTP stack (Lesson 45), persistent storage (Lesson 46 and the notes in §6), consensus rules, light-client verification, and anything about specific providers or networks. **Nothing here makes a system secure.** Rust removes memory-safety bugs; the failures in this lesson (a double-counted payment after a reorg, a retried write that executes twice, a balance that silently lost precision) are logic errors that compile cleanly.

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation and standards sites were not, so statements about the JSON-RPC specification and about specific blockchains' conventions are labeled "from memory". All "network" behavior is simulated in-process, deterministically; there are no sockets in this lesson's tests. Expected values for block hashes, notice ids and the retry jitter were computed by **separate Python programs** (✓). Timings are single-machine micro-measurements.

Builds on [Lesson 50](50_networking_determinism.md) (framing, limits, timeouts, sans-IO design, seeded simulation), [Lesson 49](49_merkle_transactions_consensus.md) (blocks, forks, confirmations, trust assumptions), [Lesson 48](48_crypto_primitives.md) (hashes, domain separation), [Lesson 46](46_databases_config_auth_jobs.md) (transactions, idempotent jobs), [Lesson 45](45_http_apis.md) (idempotency, rate limits, timeouts), [Lesson 44](44_cli_serialization.md) (`serde`), [Lesson 43](43_observability_ci_supply_chain.md), [Lesson 41](41_benchmarking_profiling.md), [Lesson 40](40_property_testing_fuzzing.md), [Lesson 26](26_error_design.md) and [Lesson 14](14_option_result_errors.md). Lesson 52 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

An application that reads from a chain is a **distributed system with an unreliable, partly untrusted peer** (the node or provider) and a **data source that can change its mind** (the chain, after a reorg). The lesson is organized around experiments that show where naive code breaks:

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| JSON numbers lose integers above 2⁵³ in many consumers | `2⁵³ + 1` does not survive a round trip through `f64`; as a decimal string it is exact (§2.2) |
| A hex parser should accept one spelling | `0xAB`, `AB`, `0X00`, odd lengths and non-ASCII are all refused (§2.1) |
| JSON-RPC has quiet corner cases | `result: null` is a result, `id: null` is not a notification, a batch of notifications gets no reply, an empty batch is an error (§3) |
| Batch replies come in any order | replies are matched by id; missing, unknown and duplicated ids are refused (§3.5) |
| Retrying everything is wrong | a bad-params error is returned at once; a timed-out *write* is reported as uncertain, not repeated (§5.3) |
| Backoff needs jitter and a ceiling | ceilings 100, 200, 400, 800, 1600, 2000, 2000 ms; the jitter for seed 1 is 70, 101, 124, 735 ms, confirmed by Python (§5.2) |
| A provider that is behind is not a reorg | the indexer waits instead of rolling back (§6.4) |
| Reorgs must be followed exactly | 200 random seeds of mining and reorgs, and after each sync the indexer's chain equals the node's (§6.3) |
| A node's data must be checked | a block whose transactions were altered without fixing the hash is rejected (§6.5) |
| Notifications are at-least-once | every fact has a stable 32-byte id; replaying the same notices applies each effect once (§7.3) |

Total in the practical challenge: one library crate, 6 modules, **97 tests**, a benchmark example, and **116 mutants** run against the test suite.

---

## 2. Serialization at the boundary

### 2.1 Be strict about what you accept ✓

Data crossing a boundary is untrusted ([Lesson 50](50_networking_determinism.md) §3). For text formats that means picking **one canonical spelling** for each value and refusing the rest. The crate's byte strings are `0x` followed by an even number of **lowercase** hex digits:

```rust
/// Decodes the canonical form produced by [`to_hex`] and nothing else.
pub fn from_hex(s: &str) -> Result<Vec<u8>, HexError> {
    let body = s.strip_prefix("0x").ok_or(HexError::MissingPrefix)?;
    let b = body.as_bytes();
    if b.len() % 2 != 0 {
        return Err(HexError::OddLength);
    }
    let mut out = Vec::with_capacity(b.len() / 2);
    for (i, pair) in b.chunks(2).enumerate() {
        let hi = digit(pair[0]).ok_or(HexError::BadDigit(2 * i))?;
        let lo = digit(pair[1]).ok_or(HexError::BadDigit(2 * i + 1))?;
        out.push(hi << 4 | lo);
    }
    Ok(out)
}
```

✓ Tests (`tests/ser.rs`): the empty string `0x` decodes to no bytes; all 256 byte values round-trip; and the decoder refuses a missing prefix (`00ab`, `0X00ab`), an odd number of digits, uppercase (`0xAB`), non-hex characters, spaces, and non-ASCII text (the length is counted in **bytes**, so `0xé` is two bytes of UTF-8 and fails at digit 0, while `0xé1` is three bytes and fails as odd-length). Error positions count digits from zero. Why so strict? Because two spellings of one value (`0xAB` and `0xab`) are two different strings to a hash, a database key, a cache or a deduplication set. This is the canonical-encoding rule of Lesson 50 §3.3 applied to text. (Real networks have their own conventions for hex quantities; from memory some of them allow variants I deliberately refuse here. The point is to choose, document and enforce one.)

### 2.2 Integers as decimal strings ✓

JSON has one number type. Parsers in many languages read it as an IEEE-754 double, which represents every integer up to 2⁵³ exactly and many above it not at all. ✓ In Rust: `9_007_199_254_740_993u64 as f64 as u64` is not equal to `9_007_199_254_740_993` (2⁵³ + 1). Balances in the smallest unit of a token, block numbers and timestamps in nanoseconds can exceed that, so the crate puts every `u64` on the wire as a **canonical decimal string**:

```rust
/// Parses a canonical decimal `u64`: digits only, no sign, no spaces, no
/// leading zeros (except `"0"`), and it must fit.
pub fn parse_u64(s: &str) -> Option<u64> {
    if s.is_empty() || (s.len() > 1 && s.starts_with('0')) {
        return None;
    }
    if !s.bytes().all(|c| c.is_ascii_digit()) {
        return None;
    }
    s.parse().ok()
}
```

✓ `parse_u64` accepts `"0"` and `"18446744073709551615"`, and refuses `"18446744073709551616"` (one past `u64::MAX`), the empty string, `"00"`, `"01"`, `"+1"`, `"-1"`, spaces, `"1.0"`, `"1e3"`, `"0x10"` and a full-width digit. A `serde` adapter (`#[serde(with = "ser::u64_string")]`) applies it, and a JSON *number* where a string is expected is an error (✓ `serde_adapters_refuse_numbers_and_bad_strings`). The same applies to `u128` and to anything in a currency: **never put money in a float** (Lesson 49 §5.3 and Lesson 50 §5.3). (From memory, several real chains' RPC conventions use hex strings or decimal strings for the same reason; I do not claim any particular one.)

### 2.3 `serde` as a validator ✓

Deserialization is the place to enforce shape. The wire block is:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(deny_unknown_fields, rename_all = "camelCase")]
pub struct Block {
    /// Height; genesis is 0.
    #[serde(with = "ser::u64_string")]
    pub number: u64,
    /// Hash of the block contents (see [`Block::compute_hash`]).
    #[serde(with = "ser::hash_hex")]
    pub hash: Hash,
    /// Hash of the parent block.
    #[serde(with = "ser::hash_hex")]
    pub parent_hash: Hash,
    /// Transaction ids included in the block.
    #[serde(with = "ser::hash_list")]
    pub transactions: Vec<Hash>,
}
```

`#[serde(deny_unknown_fields)]` makes a typo or an unexpected extra field an error instead of being ignored (✓ `blocks_refuse_unknown_fields_and_wrong_types_on_the_wire`). The trade-off is real and worth stating: **strictness helps when you control both sides** and want to catch mistakes early, and **hurts forward compatibility** when a server may add fields your client has not heard of (a lenient client ignores them). Choose per boundary: be strict on what you *emit* and on messages that are hashed or signed, and decide deliberately about what you tolerate on input from a third party. Adapters (`hash_hex`, `hash_list`) check the length (exactly 32 bytes) as part of parsing, so a short or long hash can never reach the rest of the program ([Lesson 15](15_newtype_typestate.md): parse, don't validate later).

### 2.4 Never trust a hash you did not compute ✓

A block carries its own `hash` field. A node (or whoever sits between you and it) can send contents that do not match. `Block::compute_hash` recomputes it:

`SHA-256(len("lab/block/v1") ‖ "lab/block/v1" ‖ number ‖ parent ‖ count ‖ txs)`, all integers as 8-byte big-endian, with the domain label and the transaction **count** included (Lesson 48 §4: without a count, a block whose transaction list is split differently could produce the same byte string). ✓ The hash of the genesis block (`0xefdbefc5…aef5`) and of a block with two transactions (`0x68253073…07d7`) were computed by an independent Python implementation; a test changes each field in turn and shows the hash no longer matches (`the_hash_covers_every_field`). The indexer (§6) calls `is_self_consistent()` on every block it receives. This does not prove the block is on the *best* chain (that needs consensus knowledge, Lesson 49), but it removes an easy lie.

### 2.5 JSON versus binary

JSON is readable, debuggable with `curl`, and universal; it costs bytes and time. ✓ A block with 20 transactions is 1,572 bytes of JSON; encoding takes about 1.9–2.0 µs and decoding about 2.3 µs on this machine, against about 0.55 µs to recompute its hash (release build, medians of five). A binary format (the style of Lesson 50 §3) would be smaller and faster, and fixed-width fields avoid the number problem. JSON is nonetheless what most node APIs speak (from memory), so a service must be good at it. Whatever the format, the rules are the same: one canonical form, limits before allocation, exact integers, validation at parse time.

---

## 3. JSON-RPC 2.0

### 3.1 The protocol in one page

(From memory of the specification at jsonrpc.org; I could not open it from the sandbox, so the rules below are what the crate implements and tests, which I believe match it.)

- A **request** is a JSON object with `"jsonrpc": "2.0"`, a `"method"` string, optional `"params"` (an array for positional or an object for named arguments), and an `"id"` (a string, a number or null). Without an `id`, it is a **notification**: the server must not reply.
- A **response** has `"jsonrpc": "2.0"`, the same `"id"`, and **exactly one** of `"result"` and `"error"`. An error is `{ "code": integer, "message": string, "data"?: any }`.
- A **batch** is an array of requests; the reply is an array of responses (in any order, omitting notifications), or nothing if every entry was a notification.
- Reserved codes: `-32700` parse error, `-32600` invalid request, `-32601` method not found, `-32602` invalid params, `-32603` internal error; `-32000` to `-32099` are left to the implementation.

JSON-RPC does not say anything about transport (HTTP, WebSocket, a pipe), authentication, rate limits, timeouts or retries. Those are your problem, and §5 is about them.

### 3.2 Parsing a request strictly ✓

```rust
/// Validates one request object.
///
/// On failure returns the error to send and the id to send it with
/// (`Id::Null` when the id could not be determined).
pub fn parse_request(v: &Value) -> Result<Request, (Id, RpcError)> {
    let bad = |id: Id, msg: &str| Err((id, RpcError::new(INVALID_REQUEST, msg)));
    let Some(obj) = v.as_object() else {
        return bad(Id::Null, "request must be an object");
    };
    let id = match obj.get("id") {
        None => None,
        Some(x) => match Id::from_value(x) {
            Some(id) => Some(id),
            None => return bad(Id::Null, "id must be an integer, a string or null"),
        },
    };
    let reply_id = id.clone().unwrap_or(Id::Null);
    if obj.get("jsonrpc") != Some(&Value::String("2.0".into())) {
        return bad(reply_id, "jsonrpc must be \"2.0\"");
    }
    let Some(method) = obj.get("method").and_then(Value::as_str) else {
        return bad(reply_id, "method must be a string");
    };
    let params = match obj.get("params") {
        None => Params::None,
        Some(Value::Array(a)) => Params::Array(a.clone()),
        Some(Value::Object(o)) => Params::Object(o.clone()),
        Some(_) => return bad(reply_id, "params must be an array or an object"),
    };
    Ok(Request {
        id,
        method: method.to_string(),
        params,
    })
}
```

The function returns, on failure, **the error and the best id available** for the reply (the spec says to use `null` when the id could not be determined). ✓ `invalid_requests_are_refused_with_the_best_id_available` runs fourteen malformed inputs: a numeric `method`, a missing or wrong `jsonrpc`, `params` that is a number or a string, ids that are fractional (`1.5`), arrays, booleans or larger than `i64` (which cannot be represented by the crate's `Id::Num(i64)`), and top-level values that are not objects; each gets `-32600`, with the id echoed when the request had a valid one. Note that `i64` ids are a **choice**: a client that sends 2⁶⁴ − 1 gets an error rather than a silently rounded id.

### 3.3 The server loop and its limits ✓

`handle(handler, input, limits)` returns `Option<String>`:

1. input larger than `max_bytes` → `-32600 "request too large"` before parsing anything;
2. not valid JSON → `-32700` with `id: null`;
3. a batch: empty → `-32600`; larger than `max_batch` → `-32600` **without running any method**; otherwise each entry is processed in order, and an entry that is a notification produces nothing;
4. a batch whose entries were all notifications → `None` (no reply at all).

✓ `limits_are_enforced_before_any_method_runs` checks the sizes at, one under and one over the limit, and that a rejected batch executed **zero** calls. The defaults, `1,048,576` bytes and 100 requests, are pinned by a test written after mutation testing showed they were unpinned (§13). A server without these limits lets one request carry a million sub-requests, which is the JSON-RPC version of the unbounded `Vec::with_capacity` from Lesson 50. (Real providers also limit batch size and cost per call, from memory; the numbers differ.)

Quiet corner cases, each a test: `"result": null` is a valid successful result (✓ `a_null_result_is_still_a_result`); `"id": null` is **not** a notification (✓); a failing notification is still silent (✓); an error object keeps its `data`.

### 3.4 Validating a response ✓

The client must be as strict as the server. `parse_response` demands the version, a valid id, and exactly one of `result` and `error`, with a well-typed error object; ✓ eleven bad replies (both members, neither, wrong version, missing id, a string error, a code that is a string, a fractional id, an array, not JSON) are refused. A client that accepts "anything with a `result`" will accept replies from the wrong request, from a captive-portal HTML page, or from a proxy that rewrote the body.

### 3.5 Batches and matching by id ✓

Because replies may come in any order, a client must **match by id**, not by position:

```rust
/// Matches a batch reply to the ids that were sent, returning the outcomes in
/// the order of `sent` (servers may answer in any order).
pub fn match_batch(sent: &[i64], reply: &str) -> Result<Vec<Result<Value, RpcError>>, BatchError> {
    let v: Value = serde_json::from_str(reply)
        .map_err(|e| BatchError::Malformed(format!("invalid json: {e}")))?;
    let items = match v {
        Value::Array(a) => a,
        single => {
            return match parse_response_value(&single) {
                Ok(Response {
                    outcome: Err(e), ..
                }) => Err(BatchError::Rejected(e)),
                Ok(_) => Err(BatchError::Malformed("expected an array".into())),
                Err(m) => Err(BatchError::Malformed(m)),
            };
        }
    };
    let mut seen = BTreeSet::new();
    let mut by_id = std::collections::BTreeMap::new();
    for item in &items {
        let r = parse_response_value(item).map_err(BatchError::Malformed)?;
        if !seen.insert(r.id.clone()) {
            return Err(BatchError::DuplicateId(r.id));
        }
        match r.id {
            Id::Num(n) if sent.contains(&n) => {
                by_id.insert(n, r.outcome);
            }
            other => return Err(BatchError::UnknownId(other)),
        }
    }
    let mut out = Vec::with_capacity(sent.len());
    for n in sent {
        match by_id.remove(n) {
            Some(o) => out.push(o),
            None => return Err(BatchError::Missing(Id::Num(*n))),
        }
    }
    Ok(out)
}
```

✓ Tests: replies `[12, 10, 11]` for requests `[10, 11, 12]` come back in request order; a **missing** reply, an **unknown** id, a **duplicate** id and a `null` id are each refused with a specific `BatchError`; and a single error object for the whole batch (what a server sends for "batch too large") is reported as `Rejected`, not as a malformed reply. Positional matching is a classic bug: when the server drops one entry, every later result is attributed to the wrong request.

---

## 4. Nodes and providers

### 4.1 What you are talking to

A **node** is a program that follows the chain and answers queries; a **provider** is a company running many of them behind one URL (from memory, the usual deployment for applications). Facts that shape client design:

- The node may be **behind** the real tip, or on a **different fork** from the one other nodes see, or temporarily **unavailable**. Two providers can disagree about the latest block for a few seconds without anyone being faulty.
- Providers **rate-limit** and **meter**. A client that ignores limits gets throttled or banned.
- Some nodes keep **full history**, some prune old state (from memory: "archive" versus "full" nodes); a query that works on one may fail on another.
- You usually cannot verify everything the node says. You can verify **self-consistency** (the hash matches the contents, §2.4), **linkage** (each block's parent hash matches the previous block), and, with a light client, headers against a trusted checkpoint (not built here). This is the trust table of Lesson 49 §6 in practice: an indexer that trusts one provider inherits that provider's view and its failures.

### 4.2 The mock node ✓

`chain::MockChain` keeps a canonical `Vec<Block>` plus the orphaned blocks, and can **mine** and **reorg** (`reorg(depth, replacement)` removes the last `depth` blocks and mines new ones). `chain::Node` serves it through the `Handler` trait with three methods of my own naming: `getBlockNumber`, `getBlockByNumber` (a decimal string or `"latest"`) and `getBlockByHash`. Parameters are validated: wrong count, wrong type, `"01"`, `"-1"`, a short hash or **named** parameters all return `-32602` (✓ `bad_parameters_are_reported_as_invalid_params`). An orphaned block can still be fetched **by hash** after a reorg (✓), which is also what real nodes do, from memory, and is why a hash lookup does not tell you a block is canonical.

`Node.lag` makes the node behave like a provider that is that many blocks behind: it hides the newest blocks from every method (✓ `a_lagging_node_hides_recent_blocks`). Together with scripted faults (§5) it reproduces the failures that matter without a network.

### 4.3 Simulated transport and faults ✓

`client::SimTransport` is a set of endpoints, each a `Node` with a queue of scripted `Fault`s consumed one per request:

| Fault | What the client sees |
|---|---|
| `Timeout` | no answer; the request **may have been processed** |
| `Unavailable` | refused; the request was **not** processed |
| `RateLimited(ms)` | refused with a "retry after" hint; not processed |
| `Garbage` | an HTML error page instead of JSON |
| `WrongId` | a valid response for a different request id |
| `InternalError` | JSON-RPC error `-32603` |

Time is a `Clock` trait with a `FakeClock` whose `sleep_ms` advances a counter, and randomness is a seeded generator, so every test is deterministic and takes microseconds (the whole suite of 97 tests runs in well under a second). This is the sans-IO idea of Lesson 50 §4.1 again: **time, randomness and the network are inputs**.

---

## 5. A reliable client

### 5.1 What to retry

A failed call falls in one of three classes:

1. **Will fail again**: unknown method, invalid params, invalid request. Retrying wastes time and quota. The client returns these at once (✓ `permanent_errors_are_returned_at_once`: one request each, no sleeping, no failover).
2. **Might work next time**: timeouts, refused connections, rate limits, garbage replies, a reply to the wrong id, and server errors `-32603` or in `-32000..=-32099` (✓ `retryable_codes_are_the_internal_and_server_ranges` checks both ends of the range and the codes next to it). From memory, real providers use the implementation-defined range for things like "limit exceeded" or "header not found", which is why it is retried here; check your provider's documentation, because some of those codes are permanent.
3. **Unknown whether it ran** (timeouts on writes): see §5.3.

### 5.2 Exponential backoff with full jitter ✓

After the `a`-th failed attempt the client waits a random time in `0..=min(cap, base · 2^(a−1))`:

```rust
/// The un-jittered backoff after `attempt` failed attempts (1-based):
/// `min(cap, base * 2^(attempt-1))`.
pub fn ceiling(&self, attempt: u32) -> u64 {
    let shift = (attempt.saturating_sub(1)).min(32);
    self.base_ms.saturating_mul(1u64 << shift).min(self.cap_ms)
}
```

✓ With `base = 100 ms` and `cap = 2000 ms` the ceilings are 100, 200, 400, 800, 1600, 2000, 2000 ms, no overflow for huge attempt numbers or a `u64::MAX` base (`saturating_mul`, shift capped at 32). **Why jitter?** If a thousand clients lose their connection at the same instant and all retry after exactly 200 ms, they hit the recovering server together again and again (a "thundering herd"). Randomizing the wait spreads them out (from memory, this "full jitter" variant is the one analyzed in an AWS Architecture blog post; I did not re-read it). **Why a cap?** Without it the wait grows without bound. **Why a deadline?** A caller needs an answer or an error within a known time, so the total (sleeps included) is limited: ✓ with `deadline_ms = 150` the client stops before sleeping past the deadline and reports `DeadlineExceeded`; ✓ the check is inclusive (a wait that ends exactly at the deadline is allowed, one millisecond more is not).

The generator is seeded, so jitter is reproducible: ✓ for seed 1 the waits for ceilings 100, 200, 400, 800 are **70, 101, 124 and 735 ms**, which equal the first four outputs of xorshift64* modulo `ceiling + 1` computed in Python. In production you seed from the operating system (Lesson 48 §6); jitter needs unpredictability across clients, not cryptographic strength. A **rate-limit hint** is a floor: the client waits `max(jitter, retry_after)` (✓ a hint of 1,500 ms is honored; a hint of 1 ms does not shorten the jitter).

The default policy (5 attempts, 100 ms base, 2 s cap, 10 s deadline) sleeps at most 100 + 200 + 400 + 800 = **1,500 ms** in total over its 5 attempts (✓ computed by the example); the sleeps are the only time the client adds, so real latency is that plus the time of the attempts.

### 5.3 Writes are different ✓

If a **read** times out, asking again is harmless. If a request that **submits a transaction** times out, you do not know whether the node received it. Blindly resending may create a second transaction, depending on how the chain identifies transactions. The crate encodes the caution in the type of the call: `call_with(method, params, Idempotency::Unsafe)` retries only when the failure proves the request was **not processed** (`Unavailable`, `RateLimited`), and returns `CallError::Uncertain` for everything else (✓ timeouts, garbage replies, internal errors and wrong ids all become `Uncertain` after exactly one request).

What the caller does with `Uncertain` is application logic, and the safe pattern is: **look it up by a deterministic identifier**. If the transaction id is the hash of the signed bytes (as in Lesson 49 §3.5), the caller can ask the node or the indexer "is transaction X known?" instead of resending, and resending the **identical signed bytes** is idempotent on chains where a duplicate is rejected as "already known" (from memory, common behavior; I did not check any network). What must never happen is **re-signing with a new nonce or fee** because of a timeout, which can pay twice. This is the idempotency-key idea from Lesson 45 and Lesson 46's job queues, with the transaction id as the key.

### 5.4 Failover and ids ✓

On each retry the client moves to the next endpoint (round robin), and **remembers the endpoint that worked** as the first choice for the next call (✓ `preferred()`). Every attempt uses a **new request id** (✓ the ids seen by the transport are 1, 2, 3, 4 across two calls with two timeouts), and the reply's id must match the current attempt (✓ `WrongId` is retried; an old reply to a timed-out attempt cannot be mistaken for the current one). Failover has a cost that §6.4 explains: the next endpoint may be on a different fork or behind.

### 5.5 What the client does not do

No circuit breaker (stop calling an endpoint that keeps failing), no health scores or hedged requests (send to two endpoints, use the first answer), no concurrency limits or queues, no connection pooling, no TLS, no authentication, and no cross-checking of providers (asking two providers and comparing). Each is a real engineering topic; the crate's job is to make the basic loop correct and testable.

### 5.6 Cost ✓

Parse, dispatch and encode of one `getBlockByNumber` request take about 5.2–5.3 µs; a batch of 50 takes 249–275 µs, about 5.0–5.5 µs per request (release build, medians of five, three runs). The in-process "network" costs nothing, so real latency will be dominated by the round trip to the provider (milliseconds), which is why batching and a sensible retry policy matter more than parsing speed.

---

## 6. Indexers

### 6.1 What an indexer is for

Nodes answer questions about **the chain by position** (block N, transaction by hash). Applications ask questions about **meaning**: all payments to this address, the balance history of this account, every event of this contract. An **indexer** reads blocks as they appear and writes them into a store designed for those queries (a relational database, Lesson 46; a search engine, a key-value store). It is the bridge between a log that can be rewritten near its head and an application that wants stable answers.

The central difficulty is the sentence "can be rewritten near its head": **the data you stored may stop being canonical**. (From memory, chains with probabilistic finality, Lesson 49 §4, can reorganize the last few blocks; chains with deterministic finality have a point after which blocks cannot change; either way, an indexer needs a policy.)

### 6.2 The algorithm ✓

The crate's `Indexer` keeps a local copy of the canonical chain in a `BTreeMap<u64, Block>` and a map from transaction id to the height that includes it. `sync` brings it in line with a `BlockSource`:

```rust
/// Brings the local chain in line with the source and returns the events.
///
/// The algorithm: first make sure the local tip is still on the source's
/// chain (undo it otherwise; if the source has no block at that height it
/// is behind and we simply stop); then append source blocks one by one while
/// each links to the local tip; a block that does not link means the
/// source switched branches, so undo the local tip and look again.
pub fn sync<S: BlockSource>(&mut self, source: &mut S) -> Result<Vec<Event>, SyncError> {
    let mut events = Vec::new();
    let mut undone = 0u64;
    let mut verified = false;
    loop {
        if !verified {
            if let Some(tip) = self.tip().cloned() {
                // A source with no block at our tip height is *behind*, not
                // proof of a reorg: wait instead of rolling back.
                let Some(b) = source.block_at(tip.number)? else {
                    return Ok(events);
                };
                if self.checked(b, tip.number)?.hash != tip.hash {
                    self.undo_one(&mut undone, &mut events)?;
                    continue;
                }
            }
            verified = true;
        }
        let next = self.tip().map_or(0, |b| b.number + 1);
        let Some(b) = source.block_at(next)? else {
            return Ok(events);
        };
        let b = self.checked(b, next)?;
        let expected_parent = self.tip().map_or(ZERO, |t| t.hash);
        if b.parent_hash == expected_parent {
            self.apply(b, &mut events);
        } else if self.is_empty() {
            return Err(SyncError::BadGenesis);
        } else {
            self.undo_one(&mut undone, &mut events)?;
            verified = false;
        }
    }
}
```

The reasoning, step by step:

1. **Check the tip first.** Ask the source for the block at our tip's height. If its hash differs, our tip is no longer canonical: undo it and look again. If the source has **no block** at that height, it is behind (§6.4): stop, change nothing.
2. **Append while blocks link.** Ask for block `tip + 1`. If its `parent_hash` equals our tip's hash, apply it. If it does not, the source switched branches somewhere below: undo our tip, re-verify, repeat. Each rollback walks one step back toward the fork point.
3. **Stop** when the source has no block `tip + 1`.

Rollback emits `Event::RolledBack(block)` for the newest block first; applying emits `Event::Applied(block)` oldest first. ✓ For a 3-deep reorg to a longer branch the events are `-6 -5 -4 +4 +5 +6 +7` (`a_deeper_and_longer_reorg_undoes_newest_first_and_applies_oldest_first`), the fork point is not touched (✓ `the_fork_point_is_not_touched`), syncing twice produces no events (idempotent, ✓), and an incremental sync only reports new blocks (✓).

### 6.3 Why it is right: a property test ✓

Hand-written reorg cases are not enough: the interesting bugs live in sequences nobody wrote down. ✓ `random_reorgs_always_end_with_the_same_chain_as_the_source` runs **200 seeds**; each does twelve random operations (mine a block, or reorganize the last 1–4 blocks with a replacement at least as long), syncing the indexer after only about half of them, and afterwards requires that (a) the indexer's chain equals the node's block for block, and (b) every transaction in the node's canonical chain is reported by `status` at exactly the height where it is. The test found a bug in the *test* itself: its generator reused a transaction id in two blocks of one chain, and the indexer, which records one height per transaction id, then forgot the earlier occurrence when the later block was rolled back. A repeated id in one chain is invalid data to begin with (the indexer keeps only the most recent height, which is a stated simplification), so I fixed the generator; ✓ `rolling_back_a_later_block_keeps_the_transactions_of_earlier_blocks` pins the normal case.

### 6.4 A source that is behind is not a reorg ✓

My first draft treated "the source has no block at my tip height" as a mismatch and rolled back. That is wrong with several providers: a client that fails over to a lagging endpoint would make the indexer undo its tip, then re-apply it when the client returned to a healthy endpoint, emitting **spurious `Reorged` notices** to every watcher. The fix is the rule in step 1 above; ✓ `a_source_that_is_behind_does_not_cause_a_rollback` and, through the full JSON-RPC stack with a failover to a node that lags by 3 blocks, `a_lagging_provider_after_failover_causes_no_spurious_events`.

The price is a real limitation, tested and documented: if the true chain reorganizes to a **shorter** branch, the indexer cannot tell that from lag until the source has a block at our tip's height; ✓ `a_reorg_to_a_shorter_branch_is_noticed_once_the_source_has_blocks_at_our_height` shows it waiting and then rolling back. A production indexer would add a **freshness check** (compare against a second provider, or a time bound on "behind").

### 6.5 Trust, bounds and errors ✓

- **Verify blocks** (§2.4): a block whose hash does not match its contents gives `SyncError::BadBlock`; a block with a different number than requested gives `WrongHeight`; both leave the earlier, valid blocks in place (✓).
- **The first block must have the zero parent**: otherwise `BadGenesis` (✓), instead of rolling back forever.
- **Bound reorg depth**: `max_reorg` is the deepest rollback the indexer will perform. A source that asks for more gets `ReorgTooDeep { depth, limit }`, and the state is still a valid chain prefix: exactly `limit` blocks were undone (✓ `a_reorg_deeper_than_the_limit_is_refused…`; ✓ exactly at the limit is followed). The right value is a policy decision about the chain's finality, and exceeding it should wake a human, not silently rewrite a month of history. (From memory, some chains have documented maximum reorg depths in practice and some designs make deeper reorgs impossible; I make no claim about any network.)
- **Errors propagate** (`SyncError::Source`) and progress made before the error is kept.

The `RpcSource` adapter fetches through the retrying client, parses the JSON into `Block` (so all the strictness of §2 applies), and treats a malformed block as a source error (✓).

### 6.6 Transaction status and finality ✓

`Indexer::status(tx)` answers `Unknown`, `Included { height, confirmations }` or `Confirmed { height, confirmations }`, where confirmations count the block itself (a transaction in the tip has 1) and `Confirmed` means at least `required` (✓ boundary tests). A reorg can **drop** a transaction (back to `Unknown`) or **move** it to a different height (✓ `a_reorg_can_drop_a_transaction_or_move_it`). How many confirmations are enough is the question of Lesson 49 §4.6: it depends on the chain, the value at stake and the adversary; the code only needs the number as a parameter.

### 6.7 Cost ✓

Indexing 2,001 blocks of 20 transactions each took 10.0–11.4 ms straight from memory, and 39.7–42.5 ms through the whole JSON-RPC stack (about 20–21 µs per block, 2,002 requests; release build, three runs). Following a reorg of depth 50 produced 100 events in 0.46–0.64 ms. The CPU cost of the indexer is small; **real cost is in the round trips** (one request per block here; real systems batch ranges, §6.8) and in the store.

### 6.8 What the crate leaves out, and what a real indexer needs

- **Persistence and the cursor.** The indexer's state is in memory. A real one writes blocks and its position (the *cursor*) to a database **in one transaction** ([Lesson 46](46_databases_config_auth_jobs.md)): if it wrote the data and crashed before the cursor, it would re-ingest on restart, so writes must be **idempotent** (keyed by block hash or by `(height, hash)`, using an upsert); if it wrote the cursor first, it would skip blocks. On restart it must also **verify the stored tip** against the node, because a reorg can happen while it is down. The algorithm of §6.2 already does that.
- **Rollback in the store.** `RolledBack` events must undo the derived rows (the crate hands them to the caller in order). Storing the *undo information* (or deriving rows from blocks by pure functions of the block) makes this possible.
- **Backfill versus live.** Catching up on a million old blocks wants parallel fetching of ranges and batching (§3.5); following the head wants low latency and reorg handling. They are different modes.
- **Rate limits and cost** of the provider, concurrency limits, back-pressure ([Lesson 32](32_cancellation_streams_backpressure.md)), observability ([Lesson 43](43_observability_ci_supply_chain.md)): lag in blocks, last successful sync time, reorg count, error rates.
- **Cross-checking providers** to detect a provider on a wrong fork or lying.

---

## 7. Monitoring transactions

### 7.1 States and notices ✓

The `Watcher` turns indexer events into **notices**. For each watched transaction it keeps one of: waiting (since some height), included (in a given block), confirmed, expired. Transitions:

```
Waiting --block with the tx applied--> Included --enough confirmations--> Confirmed
Waiting --expire_after blocks without inclusion--> Expired --(still may be included later)--> Included
Included or Confirmed --its block rolled back--> Waiting (clock restarts) and a Reorged notice
```

`process(events, indexer)` handles the events **in order**, then evaluates confirmations and expiry against the indexer's tip. ✓ Tests: inclusion then confirmation, each reported once; inclusion and confirmation arriving **together** are reported in that order (`required = 1`); unwatched transactions are ignored; a reorg before confirmation gives `Reorged` and then a new `Included` (and `Confirmed` later); a reorg **after** confirmation still gives `Reorged`; an expired transaction that appears later is still reported; the expiry clock restarts after a reorg drops the transaction; watching twice does not restart the clock.

A subtle case, ✓ `a_block_that_is_rolled_back_and_not_the_one_we_hold_is_ignored`: a rollback event for a block that merely lists the same transaction id but is not the block the watcher recorded is ignored. The state records the **block hash**, not just the height.

### 7.2 What "dropped" means

`Expired` here means "not seen in a block within N blocks of when you started watching". It cannot distinguish a transaction that was **dropped** from the node's pool, one that is **still pending** with a fee too low, and one that was **replaced** (from memory, many chains let a sender replace a pending transaction with the same nonce and a higher fee). To know, you need the node's view of pending transactions (not built here) and the account's nonce. The honest statement for a user interface is "not included after N blocks", not "failed".

### 7.3 At-least-once delivery and stable ids ✓

A watcher feeds something else: a webhook, a queue, an email. Those channels fail and retry, so the consumer will see some notices **twice** (at-least-once delivery). The cure is not to hope for exactly-once delivery but to make **effects idempotent**: every notice carries an `id`, the SHA-256 of `"lab/notice/v1" ‖ tx ‖ kind ‖ height ‖ block hash`, so the same fact always has the same id and a different fact (another block, another kind) has another. ✓ Known answers computed by an independent Python program: `Included` for transaction 1 in block `0x68253073…07d7` at height 1 has id `0xee1f6a74…24ad`, and `Confirmed` for the same fact `0xf81118a8…7546`. ✓ A consumer that remembers ids applied three notices once each when the channel delivered them all twice. This is [Lesson 46](46_databases_config_auth_jobs.md)'s idempotent job and [Lesson 45](45_http_apis.md)'s idempotency key in a new setting; the consumer's "seen ids" set must itself be stored durably, with the effect, in one transaction.

### 7.4 What a good user experience does with this

Show *included* quickly (it is informative and cheap), show *confirmed* when the policy says so, and treat `Reorged` as a first-class event: a payment that was shown as received may need to be un-shown, an order released and then held. Never treat *included* as *final* for anything irreversible, and never ship a feature that assumes reorgs "don't happen".

---

## 8. Rust notes

- **Errors as data** ([Lesson 26](26_error_design.md)): `RpcError`, `CallError`, `AttemptError`, `TransportError`, `SyncError`, `HexError` and `BatchError` are enums with one variant per situation, compared by value in tests. `CallError::Uncertain` exists *only* so the type system forces a caller to think about it.
- **Traits for seams**: `Transport`, `Clock`, `Handler` and `BlockSource` are the places where the world enters; everything else is deterministic. Generic types (`Client<T, C>`) cost nothing at runtime and are easy to fake in tests; a `dyn` object would also work ([Lesson 21](21_trait_objects_dispatch.md)) when a program must choose at run time.
- **`serde` attributes** are part of the wire contract: `rename_all = "camelCase"`, `deny_unknown_fields`, and `with = "…"` adapters. Test them against literal JSON text, not only round trips, because a round trip passes even when both directions are wrong in the same way.
- **Integer ids and arithmetic**: ids are `i64` and attempt counters `u32`; waits use `saturating_*` and `Duration`-free milliseconds as `u64` to keep tests exact. In real code, prefer `std::time::Duration` at API boundaries ([Lesson 41](41_benchmarking_profiling.md) used it for measurements).
- **Cloning for isolation**: `parse_request` clones the `params` out of the parsed `Value`, which is simple and costs a copy; for hot paths take ownership of the `Value` instead.
- **No `unsafe`, no `async`**: the same logic runs under an async runtime ([Lesson 31](31_tokio.md)) if `Transport::send` and `Clock::sleep_ms` become `async fn`s; the retry loop would not change shape. Keeping it synchronous keeps the tests simple.

---

## 9. Common mistakes

1. Putting 64-bit integers or amounts in JSON numbers, or going through `f64` (§2.2).
2. Accepting several spellings of one value (hex case, leading zeros, a `0X` prefix) (§2.1).
3. Treating `null` ids as notifications, or `result: null` as a missing result (§3.3).
4. Matching batch replies by position instead of by id (§3.5).
5. No limits on request size and batch length on a server you run (§3.3).
6. Retrying permanent errors, or retrying everything with no backoff and no deadline (§5).
7. Backoff without jitter, so that all clients retry together (§5.2).
8. Retrying a **write** after a timeout as if it were a read (§5.3).
9. Re-signing a transaction with a new nonce because the first submission "timed out" (§5.3).
10. Trusting block hashes (or any self-reported identifier) the node sends (§2.4).
11. Storing the chain without parent-hash links, so that a reorg leaves stale blocks (§11).
12. Reading "the provider has no such block" as a reorg (§6.4).
13. No bound on how deep a reorg may rewrite stored data (§6.5).
14. Counting confirmations with an off-by-one, or letting the subtraction underflow (§11).
15. Writing the data and the cursor in separate steps (§6.8).
16. Delivering notifications with no stable id, so that retries create duplicates (§7.3).
17. Showing "included" as "paid" (§7.4).
18. Assuming a second provider agrees with the first (§4.1).

---

## 10. Security notes and honest limits

- `rpclab` is teaching code. It has **no TLS, no authentication, no authorization, no rate limiting on the server side, and no secrets handling**. A real client must authenticate to its provider (API keys are secrets: Lesson 46 §4, Lesson 48 §10) and must not log them; a real server must not expose administrative methods on a public port.
- A **malicious or compromised provider** can lie about anything the client cannot verify: it can hide transactions, show a fork, or delay blocks. Self-consistency checks (§2.4) catch only crude lies. Cross-checking several independent providers, or running your own node, or verifying headers with a light client, are the real defenses (not built).
- The retry client can amplify load. A policy with many attempts and no jitter turns an outage into an attack on your own provider; a call that fans out across endpoints multiplies the cost.
- JSON parsing of untrusted input can still be expensive (deeply nested values, huge strings). The crate bounds the **input size** in the server; `serde_json` also has a recursion limit (from memory, 128), which I did not test.
- Simulated faults are a small, friendly sample. Real failures include partial responses, slow trickles (Lesson 50 §2.4), TLS errors, DNS failures and providers that return success with stale data.
- Nothing here addresses privacy: the provider sees every address and transaction you query.
- A deterministic test suite proves the logic against *these* scripted faults; it does not prove the system survives real ones.

---

## 11. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** For each input, say what a JSON-RPC 2.0 server replies (a result, an error with which code and id, or nothing): (a) `{"jsonrpc":"2.0","method":"m","id":1}` for a method that returns `null`; (b) `{"jsonrpc":"2.0","method":"m"}`; (c) `{"jsonrpc":"2.0","method":"m","id":null}`; (d) `{"jsonrpc":"2.0","method":"m","id":1.5}`; (e) `{"jsonrpc":"2.0","method":"m","id":"x"` (unterminated); (f) `[]`; (g) a batch of two notifications.
2. **(Warm-up)** Why are 64-bit integers sent as decimal strings in this lesson? What is the smallest positive integer that a double cannot represent exactly?
3. **(Easy)** Write `decode_hex(&str) -> Result<Vec<u8>, String>` accepting only `0x` plus an even number of lowercase hex digits.
4. **(Medium)** Write `backoff_ms(attempt, base, cap, rand_below)` implementing full jitter without overflow, where `rand_below(n)` returns a value in `0..n`.
5. **(Medium)** Given the block ids of a local chain and of the source's chain (one `u64` per height), write `first_difference` and `rollback_depth`; a source that is merely behind must not cause a rollback.
6. **(Medium-hard)** Trace the notices produced for one watched transaction (confirmations required: 2) in this scenario: the transaction is mined in block 2, then block 3 is mined, then blocks 2 and 3 are replaced by two blocks where the transaction is in the new block 3, then block 4 is mined. The indexer syncs after each step.
7. **(Hard)** A payment service submits a signed transaction and the call times out. Describe what it does next, what it must not do, and what the indexer and the watcher contribute.
8. **(Hard)** Design the persistence of an indexer that writes rows to a database: what is written together, what is keyed how, what happens if the process dies between any two steps, and what happens on a reorg that happens while the process is down?

<details markdown="1">
<summary>Answers</summary>

1. (a) A **result** reply: `{"jsonrpc":"2.0","id":1,"result":null}` — `null` is a valid result. (b) **Nothing**: no `id` means notification (the method still runs). (c) A reply with `"id":null` and `"result":null`: an explicit `null` id is **not** a notification. (d) `-32600` invalid request with `id: null` (ids should not be fractional). (e) `-32700` parse error with `id: null`. (f) `-32600` invalid request (an empty batch is invalid), as a single error object, not an array. (g) **Nothing at all**: no reply to a batch of only notifications. ✓ each case is a test in `tests/jsonrpc.rs`.
2. Many JSON consumers parse every number as an IEEE-754 double, which has 53 bits of significand, so integers above 2⁵³ may be rounded; sending them as strings keeps them exact. The smallest positive integer that a double cannot represent exactly is **2⁵³ + 1 = 9,007,199,254,740,993** (✓ `9_007_199_254_740_993u64 as f64 as u64` is not equal to the original).
3. A possible answer (✓ compiled and run in a scratch crate, with tests for the accepted and rejected forms and a round trip of all 256 byte values):

   ```rust
   /// Decode `0x` + an even number of lowercase hex digits; nothing else.
   pub fn decode_hex(s: &str) -> Result<Vec<u8>, String> {
       let body = s.strip_prefix("0x").ok_or("missing 0x prefix")?;
       if body.len() % 2 != 0 {
           return Err("odd number of digits".into());
       }
       let digit = |c: u8| match c {
           b'0'..=b'9' => Some(c - b'0'),
           b'a'..=b'f' => Some(c - b'a' + 10),
           _ => None,
       };
       body.as_bytes()
           .chunks(2)
           .enumerate()
           .map(|(i, p)| match (digit(p[0]), digit(p[1])) {
               (Some(h), Some(l)) => Ok(h << 4 | l),
               _ => Err(format!("bad digit in byte {i}")),
           })
           .collect()
   }
   ```

   The length is checked in **bytes** before the digit loop, so a multi-byte character cannot slip through as "two digits"; uppercase is refused by `digit`, since only `a..=f` is accepted.
4. A possible answer (✓ compiled and run; the test uses a "random" source that always returns its largest value to get the ceiling for each attempt):

   ```rust
   /// "Full jitter": a uniformly random wait between 0 and `min(cap, base * 2^(attempt-1))`.
   /// `rand_below(n)` must return a value in `0..n`.
   pub fn backoff_ms(attempt: u32, base: u64, cap: u64, mut rand_below: impl FnMut(u64) -> u64) -> u64 {
       let shift = attempt.saturating_sub(1).min(32);
       let ceiling = base.saturating_mul(1u64 << shift).min(cap);
       rand_below(ceiling.saturating_add(1))
   }
   ```

   `attempt.saturating_sub(1).min(32)` caps the shift so `1u64 << shift` cannot overflow; `saturating_mul` and `min(cap)` keep the ceiling bounded; `saturating_add(1)` makes the upper bound inclusive without wrapping. With a source that always returns its maximum the results are 100, 200, 400, 800, 1600, 2000, 2000, 2000 for `base = 100`, `cap = 2000`.
5. A possible answer (✓ compiled and run):

   ```rust
   /// Index of the first height at which two chains (given as block ids by height)
   /// differ, i.e. the number of blocks they share. `None` if they are identical
   /// over the shorter length and have the same length.
   pub fn first_difference(local: &[u64], remote: &[u64]) -> Option<usize> {
       let common = local.iter().zip(remote).take_while(|(a, b)| a == b).count();
       if common == local.len() && common == remote.len() {
           None
       } else {
           Some(common)
       }
   }

   /// How many local blocks must be rolled back to follow `remote`.
   pub fn rollback_depth(local: &[u64], remote: &[u64]) -> usize {
       match first_difference(local, remote) {
           None => 0,
           // a remote that is merely a prefix (behind) needs no rollback
           Some(c) if c == remote.len() => 0,
           Some(c) => local.len() - c,
       }
   }
   ```

   `first_difference` counts the common prefix; if one chain is a prefix of the other, the shorter one is *behind*, not forked. `rollback_depth` is `local.len() - common` only when the remote chain actually has a different block at height `common`, which is the same decision the indexer makes with its "no block at my tip height → wait" rule.
6. ✓ (run with the crate; `required = 2`, watched from height 0): after block 2 is mined and synced: **Included at height 2**. After block 3: **Confirmed at height 2** (block 2 has 2 confirmations). After the reorg that replaces blocks 2 and 3 (the transaction is now in the new block 3): **Reorged at height 2**, then **Included at height 3**, and no `Confirmed` yet because block 3 is the tip and has 1 confirmation. After block 4: **Confirmed at height 3**. A further sync with no new events yields nothing. The `Reorged` notice follows the *old* block (id derived from its hash), and the new `Included` has a different id, so a consumer can store both facts.
7. First, **do not resend blindly and do not re-sign**. The transaction's id is the hash of the signed bytes, so the service already knows it; it asks the node (or its own indexer) whether that id is known or included. If it is unknown, resending the **identical signed bytes** is safe on chains that reject duplicates as "already known" (from memory; check your chain's behavior), and within the limits of the nonce and fee already chosen. If a new transaction is needed (the old one is stuck), it must be an explicit decision that **replaces** the old one (same nonce on account-model chains, from memory) and not an additional payment. The **indexer** tells the service whether and where the transaction was included; the **watcher** turns that into `Included`, `Confirmed` and `Reorged` notices, with an expiry notice if nothing appears within N blocks, which prompts the human or policy decision above. The client type `CallError::Uncertain` exists to force this branch.
8. In **one database transaction** per block (or per batch): upsert the derived rows keyed by `(block hash, index)` or by natural keys that include the hash, and update the cursor (`height` and `hash` of the last ingested block). Then a crash before the commit loses the whole step (nothing changed) and a crash after it loses nothing; re-ingesting a block is harmless because the upserts are idempotent. For reorgs, keep enough information to **undo** a block: either delete rows by block hash, or mark rows with the block hash and only expose rows whose block is on the current chain. On restart, read the stored cursor and **verify its hash against the node** (the algorithm of §6.2 does this): if the node has a different block at that height, roll back stored blocks one by one (applying the same undo) until the hashes match, within the `max_reorg` bound, and alert a human when the bound is exceeded. Notices are produced from the same transaction (an outbox table) so that "stored" and "to be announced" cannot diverge; consumers dedupe by notice id.

</details>

---

## 12. Debugging challenge

A service follows a chain and reports payments. Four reports arrive: (1) after a reorganization, the database contains a block 5 whose parent is not the stored block 4, and a payment that no longer exists in the chain is still shown as received; (2) for a block at the tip the service reports "0 confirmations", and for a block number the service has not seen yet it panics with `attempt to subtract with overflow`; (3) when the provider is down, the service sends thousands of requests per second and never stops; (4) an amount of 9007199254740993 is displayed as 9007199254740992. Explain each cause and fix the code.

```rust
use rpclab::chain::Block;
use serde_json::Value;
use std::collections::BTreeMap;

/// Keeps the blocks seen so far.
#[derive(Default)]
pub struct Follower {
    pub blocks: BTreeMap<u64, Block>,
    pub last: u64,
}

impl Follower {
    pub fn ingest(&mut self, b: Block) {
        self.last = b.number;
        self.blocks.insert(b.number, b);
    }

    pub fn confirmations(&self, number: u64) -> u64 {
        self.last - number
    }
}

/// Retries until it works.
pub fn retry<T>(mut f: impl FnMut() -> Result<T, String>) -> T {
    loop {
        if let Ok(v) = f() {
            return v;
        }
    }
}

/// Reads an amount from a JSON-RPC result.
pub fn amount(v: &Value) -> u64 {
    v.as_f64().unwrap() as u64
}
```

<details markdown="1">
<summary>Solution</summary>

1. **No linkage check.** `ingest` overwrites `blocks[number]` and sets `last`, whatever the block's parent. After a reorg the node serves the new branch from height 4, the follower stores new block 4 over old block 4 and keeps old block 5 (✓ `buggy_follower_keeps_stale_blocks_after_a_reorg`: block 5 comes from the abandoned branch and its `parent_hash` is not the hash of the stored block 4). Fix: append only a block whose `number` is `tip + 1` and whose `parent_hash` equals the tip's hash; otherwise **drop the tip** and have the caller fetch again, repeating until the fork point (✓ `fixed_follower_never_stores_a_broken_chain` walks this through). Also verify `is_self_consistent()`.
2. **Off-by-one and unchecked subtraction.** A block's own confirmations are `tip − number + 1`, not `tip − number` (✓ `buggy_confirmations_are_off_by_one`), and `last - number` underflows when the number is above the tip (✓ the debug-build panic message is `attempt to subtract with overflow`; in a release build it would **wrap** to a huge number, which is worse: the service would report an enormous confirmation count). Fix: look up the block, compute from the tip, and return 0 when unknown (✓ `fixed_confirmations_count_the_block_itself_and_never_underflow`).
3. **Unbounded retry with no pause.** `retry` loops forever and calls the function again immediately (✓ `buggy_retry_never_pauses_and_never_gives_up`: a thousand instant calls against a down server). Fix: a maximum number of attempts, a pause that grows between attempts (with jitter), and a returned error (✓ `fixed_retry_is_bounded_and_pauses_between_attempts`: four attempts, pauses after attempts 1, 2 and 3). The full version is the client of §5.
4. **Precision loss through `f64`.** `as_f64()` converts the number to a double, which cannot hold 2⁵³ + 1 (✓ `buggy_amount_loses_precision`; note that `serde_json` itself parsed the number exactly, `as_u64()` returns it; the loss is in the conversion). Fix: send and parse amounts as canonical decimal **strings** (✓ `fixed_amount_uses_exact_strings`: the exact value is parsed, a JSON number is refused, and `"007"` is refused).

Fixed version (✓ compiled; four tests for it and five that demonstrate the faults above):

```rust
use rpclab::chain::{Block, ZERO};
use rpclab::ser::parse_u64;
use serde_json::Value;
use std::collections::BTreeMap;

#[derive(Debug, PartialEq)]
pub enum Ingest {
    Applied,
    /// Does not extend the tip: the caller must fetch the parent and retry.
    Gap,
}

#[derive(Default)]
pub struct Follower {
    blocks: BTreeMap<u64, Block>,
}

impl Follower {
    pub fn tip(&self) -> Option<&Block> {
        self.blocks.values().next_back()
    }

    pub fn len(&self) -> usize {
        self.blocks.len()
    }

    /// Appends only a block that links to the tip; rolls back and reports a
    /// gap when the parent does not match.
    pub fn ingest(&mut self, b: Block) -> Ingest {
        if !b.is_self_consistent() {
            return Ingest::Gap;
        }
        let want = self.tip().map_or(ZERO, |t| t.hash);
        if b.number == self.tip().map_or(0, |t| t.number + 1) && b.parent_hash == want {
            self.blocks.insert(b.number, b);
            Ingest::Applied
        } else {
            // our tip is not an ancestor of `b`: drop it and let the caller refetch
            self.blocks.pop_last();
            Ingest::Gap
        }
    }

    /// Blocks on top of `number`, counting its own block; 0 if unknown.
    pub fn confirmations(&self, number: u64) -> u64 {
        match (self.blocks.contains_key(&number), self.tip()) {
            (true, Some(t)) => t.number - number + 1,
            _ => 0,
        }
    }
}

/// At most `max` attempts, with a growing pause between them (passed in so
/// tests need not sleep).
pub fn retry<T, E>(
    max: u32,
    mut pause: impl FnMut(u32),
    mut f: impl FnMut() -> Result<T, E>,
) -> Result<T, E> {
    let mut attempt = 1;
    loop {
        match f() {
            Ok(v) => return Ok(v),
            Err(e) if attempt >= max => return Err(e),
            Err(_) => {
                pause(attempt);
                attempt += 1;
            }
        }
    }
}

/// Amounts travel as canonical decimal strings.
pub fn amount(v: &Value) -> Option<u64> {
    parse_u64(v.as_str()?)
}
```

Two notes: `Ingest::Gap` only says "fetch the parent and try again"; the crate's indexer does the fetching and bounds the rollback depth (§6.2, §6.5). And the fixed `retry` takes the pause as a closure so tests do not sleep; in production the closure sleeps for a jittered, capped time (§5.2).

</details>

---

## 13. Interview questions

1. *What is JSON-RPC 2.0? How do requests, responses, notifications and batches work?*
2. *A server receives `{"jsonrpc":"2.0","method":"m","id":null}`. Is it a notification?*
3. *How should a client match the replies of a batch?*
4. *Why send large integers as strings in JSON?*
5. *Which RPC errors should a client retry? Which not? What is different about writes?*
6. *Explain exponential backoff with jitter. Why the cap and the deadline?*
7. *A transaction submission times out. What do you do?*
8. *What is a blockchain reorganization? How does an indexer handle it?*
9. *Your provider returns no block at your tip's height. Is that a reorg?*
10. *How do you decide that a transaction is final? What can go wrong with "N confirmations"?*
11. *How do you make notifications to a webhook safe against duplicates?*
12. *Design the database writes of an indexer so that a crash at any point is safe.*
13. *What can a malicious RPC provider do to you, and what can you verify?*

<details markdown="1">
<summary>Answers</summary>

1. A small stateless protocol: a request has `jsonrpc: "2.0"`, `method`, optional `params` and an `id`; a response echoes the id and has exactly one of `result` and `error` (`code`, `message`, optional `data`); a request without an id is a notification and gets no reply; a batch is an array of requests answered by an array of responses in any order (§3).
2. No. An explicit `null` id is a normal request (the spec discourages it), so it gets a reply with `id: null`; a notification has **no** `id` member (✓ tested).
3. By `id`, never by position: replies may be reordered and entries may be missing; reject unknown, duplicate and missing ids (§3.5).
4. Many consumers parse JSON numbers as doubles, which are exact only up to 2⁵³; a decimal string is exact and can be validated canonically (§2.2).
5. Retry transport failures, rate limits, garbage replies and server errors (`-32603`, `-32000..-32099`, with the caveat that providers differ); do not retry invalid request, unknown method or invalid params. A write that timed out may have been executed, so it is not blindly retried: look it up by its deterministic id instead (§5.3).
6. Wait a random time between 0 and `min(cap, base·2^(n−1))` after the n-th failure. The exponential growth gives a struggling server room; jitter prevents synchronized retries; the cap bounds the wait; the deadline bounds the whole call (§5.2).
7. Do not resend with a new nonce/fee. Look the transaction up by id (node, indexer); if it is unknown, rebroadcast the same signed bytes where duplicates are rejected as known; if it must be replaced, do so deliberately; surface `Uncertain` to the caller (§5.3, exercise 7).
8. Nodes switch to a different best branch, so blocks near the head stop being canonical. An indexer verifies parent-hash links, undoes its own blocks back to the fork point (emitting rollback events so derived data is undone), and applies the new branch; it bounds the depth and alerts when the bound is exceeded (§6.2).
9. Not necessarily: the provider may be behind. Rolling back on that evidence creates spurious reorgs after a failover. Wait, or cross-check with another provider (§6.4).
10. Finality is a policy: a number of confirmations for probabilistic-finality chains, or a protocol-defined finalized marker where one exists (from memory). Confirmations reduce the risk but do not remove it, they assume a threat model (Lesson 49 §4.6), and an indexer must still handle a deeper reorg than expected.
11. Give each notice a stable id derived from the fact (transaction, kind, height, block hash), and make the consumer idempotent by storing seen ids together with the effect (§7.3).
12. One database transaction per block or batch: idempotent upserts keyed by block hash plus the cursor; undo data for rollbacks; on restart verify the stored tip against the node and roll back to the fork point (exercise 8).
13. It can lie, delay, hide, serve a stale or forked chain, or rate-limit you. You can verify self-consistency (hash matches contents), linkage (parent hashes), and, with extra work, compare several providers or verify headers against a trusted checkpoint; you cannot verify that a chain is the best one from a single untrusted source (§4.1, §10).

</details>

---

## 14. Practical challenge: `rpclab`

Build a library crate that:

1. Provides strict serialization helpers: canonical `0x` lowercase hex, canonical decimal `u64`s, and `serde` adapters for them.
2. Implements JSON-RPC 2.0: strict request parsing with the best id for errors, a dispatcher with size and batch limits and notification semantics, strict response parsing, and batch matching by id.
3. Provides a mock chain with mining and reorgs, and a node serving it through a handler (including a lag setting).
4. Provides a client with retries (classified errors), exponential backoff with full jitter from a seeded generator, a deadline, endpoint failover and an explicit idempotency mode for writes, against a simulated transport with scripted faults.
5. Provides an indexer that follows the chain through any block source, handles reorgs with a depth bound, verifies blocks, and answers transaction status with confirmations, and a watcher that emits each state change once with a stable id.
6. Has a benchmark example, passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

```toml
[package]
name = "rpclab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "Teaching code for Lesson 51: JSON-RPC 2.0, a retrying client, a chain indexer with reorg handling, a transaction watcher and strict serialization. Not audited."

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sha2 = "0.11"
```

### `src/lib.rs`

```rust
//! Teaching code for Lesson 51: strict serialization, JSON-RPC 2.0, a mock
//! node, a retrying failover client, a chain indexer with reorg handling and a
//! transaction watcher. **Not audited; not for production.**
#![deny(missing_docs)]

pub mod chain;
pub mod client;
pub mod indexer;
pub mod jsonrpc;
pub mod ser;
pub mod watch;
```

### `src/ser.rs`

```rust
//! Strict serialization helpers: canonical hex and decimal-string integers.
//!
//! Wire rules used throughout the crate:
//! * byte strings are `0x` followed by an even number of **lowercase** hex digits;
//! * 64-bit integers travel as **decimal strings** (no sign, no leading zeros),
//!   because JSON numbers are doubles in many consumers and lose precision
//!   above 2^53.

use std::fmt;

/// Why a hex string was refused.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum HexError {
    /// The string does not start with `0x`.
    MissingPrefix,
    /// The number of digits is odd.
    OddLength,
    /// A character that is not a lowercase hex digit, at this digit index.
    BadDigit(usize),
    /// The decoded length differs from the one required.
    WrongLength {
        /// Required number of bytes.
        expected: usize,
        /// Number of bytes found.
        got: usize,
    },
}

impl fmt::Display for HexError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            HexError::MissingPrefix => write!(f, "hex string must start with 0x"),
            HexError::OddLength => write!(f, "hex string has an odd number of digits"),
            HexError::BadDigit(i) => write!(f, "invalid hex digit at position {i}"),
            HexError::WrongLength { expected, got } => {
                write!(f, "expected {expected} bytes, got {got}")
            }
        }
    }
}

impl std::error::Error for HexError {}

const DIGITS: &[u8; 16] = b"0123456789abcdef";

/// Encodes bytes as `0x` + lowercase hex.
pub fn to_hex(bytes: &[u8]) -> String {
    let mut s = String::with_capacity(2 + bytes.len() * 2);
    s.push_str("0x");
    for b in bytes {
        s.push(DIGITS[(b >> 4) as usize] as char);
        s.push(DIGITS[(b & 15) as usize] as char);
    }
    s
}

fn digit(c: u8) -> Option<u8> {
    match c {
        b'0'..=b'9' => Some(c - b'0'),
        b'a'..=b'f' => Some(c - b'a' + 10),
        _ => None,
    }
}

/// Decodes the canonical form produced by [`to_hex`] and nothing else.
pub fn from_hex(s: &str) -> Result<Vec<u8>, HexError> {
    let body = s.strip_prefix("0x").ok_or(HexError::MissingPrefix)?;
    let b = body.as_bytes();
    if b.len() % 2 != 0 {
        return Err(HexError::OddLength);
    }
    let mut out = Vec::with_capacity(b.len() / 2);
    for (i, pair) in b.chunks(2).enumerate() {
        let hi = digit(pair[0]).ok_or(HexError::BadDigit(2 * i))?;
        let lo = digit(pair[1]).ok_or(HexError::BadDigit(2 * i + 1))?;
        out.push(hi << 4 | lo);
    }
    Ok(out)
}

/// Decodes a hex string of exactly `N` bytes.
pub fn hex_array<const N: usize>(s: &str) -> Result<[u8; N], HexError> {
    let v = from_hex(s)?;
    v.try_into().map_err(|v: Vec<u8>| HexError::WrongLength {
        expected: N,
        got: v.len(),
    })
}

/// Parses a canonical decimal `u64`: digits only, no sign, no spaces, no
/// leading zeros (except `"0"`), and it must fit.
pub fn parse_u64(s: &str) -> Option<u64> {
    if s.is_empty() || (s.len() > 1 && s.starts_with('0')) {
        return None;
    }
    if !s.bytes().all(|c| c.is_ascii_digit()) {
        return None;
    }
    s.parse().ok()
}

/// `serde` adapter: `u64` as a canonical decimal string.
pub mod u64_string {
    use serde::{Deserialize, Deserializer, Serializer, de::Error};

    /// Serializes as a decimal string.
    pub fn serialize<S: Serializer>(v: &u64, s: S) -> Result<S::Ok, S::Error> {
        s.serialize_str(&v.to_string())
    }

    /// Deserializes only the canonical decimal string.
    pub fn deserialize<'de, D: Deserializer<'de>>(d: D) -> Result<u64, D::Error> {
        let s = String::deserialize(d)?;
        super::parse_u64(&s).ok_or_else(|| D::Error::custom("not a canonical decimal u64"))
    }
}

/// `serde` adapter: a 32-byte hash as `0x` hex.
pub mod hash_hex {
    use serde::{Deserialize, Deserializer, Serializer, de::Error};

    /// Serializes as `0x` + 64 lowercase hex digits.
    pub fn serialize<S: Serializer>(v: &[u8; 32], s: S) -> Result<S::Ok, S::Error> {
        s.serialize_str(&super::to_hex(v))
    }

    /// Deserializes exactly 32 bytes of canonical hex.
    pub fn deserialize<'de, D: Deserializer<'de>>(d: D) -> Result<[u8; 32], D::Error> {
        let s = String::deserialize(d)?;
        super::hex_array::<32>(&s).map_err(D::Error::custom)
    }
}

/// `serde` adapter: a list of 32-byte hashes.
pub mod hash_list {
    use serde::{Deserialize, Deserializer, Serializer, de::Error, ser::SerializeSeq};

    /// Serializes as an array of hex strings.
    pub fn serialize<S: Serializer>(v: &[[u8; 32]], s: S) -> Result<S::Ok, S::Error> {
        let mut seq = s.serialize_seq(Some(v.len()))?;
        for h in v {
            seq.serialize_element(&super::to_hex(h))?;
        }
        seq.end()
    }

    /// Deserializes an array of canonical hex hashes.
    pub fn deserialize<'de, D: Deserializer<'de>>(d: D) -> Result<Vec<[u8; 32]>, D::Error> {
        let v = Vec::<String>::deserialize(d)?;
        v.iter()
            .map(|s| super::hex_array::<32>(s).map_err(D::Error::custom))
            .collect()
    }
}
```

### `src/jsonrpc.rs`

```rust
//! JSON-RPC 2.0: strict request parsing, a server-side dispatcher with limits,
//! response parsing and batch matching for clients.

use serde_json::{Map, Value, json};
use std::collections::BTreeSet;
use std::fmt;

/// Invalid JSON was received.
pub const PARSE_ERROR: i64 = -32700;
/// The JSON is not a valid request object.
pub const INVALID_REQUEST: i64 = -32600;
/// The method does not exist.
pub const METHOD_NOT_FOUND: i64 = -32601;
/// Invalid method parameters.
pub const INVALID_PARAMS: i64 = -32602;
/// Internal error.
pub const INTERNAL_ERROR: i64 = -32603;

/// A request or response identifier.
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum Id {
    /// An integer id.
    Num(i64),
    /// A string id.
    Str(String),
    /// A null id (used in error responses when the id is unknown).
    Null,
}

impl Id {
    fn to_value(&self) -> Value {
        match self {
            Id::Num(n) => json!(n),
            Id::Str(s) => json!(s),
            Id::Null => Value::Null,
        }
    }

    fn from_value(v: &Value) -> Option<Id> {
        match v {
            Value::Null => Some(Id::Null),
            Value::String(s) => Some(Id::Str(s.clone())),
            Value::Number(n) => n.as_i64().map(Id::Num),
            _ => None,
        }
    }
}

/// An error object.
#[derive(Debug, Clone, PartialEq)]
pub struct RpcError {
    /// Numeric code.
    pub code: i64,
    /// Short description.
    pub message: String,
    /// Optional extra data.
    pub data: Option<Value>,
}

impl RpcError {
    /// Builds an error without data.
    pub fn new(code: i64, message: impl Into<String>) -> RpcError {
        RpcError {
            code,
            message: message.into(),
            data: None,
        }
    }

    /// `-32601`.
    pub fn method_not_found(method: &str) -> RpcError {
        RpcError::new(METHOD_NOT_FOUND, format!("method not found: {method}"))
    }

    /// `-32602`.
    pub fn invalid_params(msg: &str) -> RpcError {
        RpcError::new(INVALID_PARAMS, format!("invalid params: {msg}"))
    }

    fn to_value(&self) -> Value {
        let mut m = Map::new();
        m.insert("code".into(), json!(self.code));
        m.insert("message".into(), json!(self.message));
        if let Some(d) = &self.data {
            m.insert("data".into(), d.clone());
        }
        Value::Object(m)
    }
}

impl fmt::Display for RpcError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "rpc error {}: {}", self.code, self.message)
    }
}

impl std::error::Error for RpcError {}

/// The `params` member.
#[derive(Debug, Clone, PartialEq)]
pub enum Params {
    /// Absent.
    None,
    /// Positional.
    Array(Vec<Value>),
    /// Named.
    Object(Map<String, Value>),
}

impl Params {
    /// The positional arguments; an absent `params` counts as empty.
    pub fn as_array(&self) -> Option<&[Value]> {
        match self {
            Params::None => Some(&[]),
            Params::Array(v) => Some(v),
            Params::Object(_) => None,
        }
    }
}

/// A parsed request. `id == None` means a notification (no response allowed).
#[derive(Debug, Clone, PartialEq)]
pub struct Request {
    /// Identifier; absent for notifications.
    pub id: Option<Id>,
    /// Method name.
    pub method: String,
    /// Parameters.
    pub params: Params,
}

/// Validates one request object.
///
/// On failure returns the error to send and the id to send it with
/// (`Id::Null` when the id could not be determined).
pub fn parse_request(v: &Value) -> Result<Request, (Id, RpcError)> {
    let bad = |id: Id, msg: &str| Err((id, RpcError::new(INVALID_REQUEST, msg)));
    let Some(obj) = v.as_object() else {
        return bad(Id::Null, "request must be an object");
    };
    let id = match obj.get("id") {
        None => None,
        Some(x) => match Id::from_value(x) {
            Some(id) => Some(id),
            None => return bad(Id::Null, "id must be an integer, a string or null"),
        },
    };
    let reply_id = id.clone().unwrap_or(Id::Null);
    if obj.get("jsonrpc") != Some(&Value::String("2.0".into())) {
        return bad(reply_id, "jsonrpc must be \"2.0\"");
    }
    let Some(method) = obj.get("method").and_then(Value::as_str) else {
        return bad(reply_id, "method must be a string");
    };
    let params = match obj.get("params") {
        None => Params::None,
        Some(Value::Array(a)) => Params::Array(a.clone()),
        Some(Value::Object(o)) => Params::Object(o.clone()),
        Some(_) => return bad(reply_id, "params must be an array or an object"),
    };
    Ok(Request {
        id,
        method: method.to_string(),
        params,
    })
}

/// Implemented by whatever answers requests.
pub trait Handler {
    /// Runs one method.
    fn call(&mut self, method: &str, params: &Params) -> Result<Value, RpcError>;
}

/// Server-side limits.
#[derive(Debug, Clone, Copy)]
pub struct Limits {
    /// Largest accepted input in bytes.
    pub max_bytes: usize,
    /// Largest accepted batch.
    pub max_batch: usize,
}

impl Default for Limits {
    fn default() -> Self {
        Limits {
            max_bytes: 1 << 20,
            max_batch: 100,
        }
    }
}

fn error_response(id: &Id, e: &RpcError) -> Value {
    json!({"jsonrpc": "2.0", "id": id.to_value(), "error": e.to_value()})
}

fn run_one<H: Handler>(h: &mut H, v: &Value) -> Option<Value> {
    match parse_request(v) {
        Err((id, e)) => Some(error_response(&id, &e)),
        Ok(req) => {
            let out = h.call(&req.method, &req.params);
            let id = req.id?; // notification: the result is discarded
            Some(match out {
                Ok(result) => json!({"jsonrpc": "2.0", "id": id.to_value(), "result": result}),
                Err(e) => error_response(&id, &e),
            })
        }
    }
}

/// Handles one text input and returns the text to send back, if any.
pub fn handle<H: Handler>(h: &mut H, input: &str, limits: &Limits) -> Option<String> {
    if input.len() > limits.max_bytes {
        let e = RpcError::new(INVALID_REQUEST, "request too large");
        return Some(error_response(&Id::Null, &e).to_string());
    }
    let v: Value = match serde_json::from_str(input) {
        Ok(v) => v,
        Err(_) => {
            let e = RpcError::new(PARSE_ERROR, "parse error");
            return Some(error_response(&Id::Null, &e).to_string());
        }
    };
    match v {
        Value::Array(items) => {
            if items.is_empty() {
                let e = RpcError::new(INVALID_REQUEST, "empty batch");
                return Some(error_response(&Id::Null, &e).to_string());
            }
            if items.len() > limits.max_batch {
                let e = RpcError::new(INVALID_REQUEST, "batch too large");
                return Some(error_response(&Id::Null, &e).to_string());
            }
            let out: Vec<Value> = items.iter().filter_map(|x| run_one(h, x)).collect();
            if out.is_empty() {
                None
            } else {
                Some(Value::Array(out).to_string())
            }
        }
        other => run_one(h, &other).map(|v| v.to_string()),
    }
}

/// A parsed response.
#[derive(Debug, Clone, PartialEq)]
pub struct Response {
    /// The id it answers (`Null` if the server could not tell).
    pub id: Id,
    /// The outcome.
    pub outcome: Result<Value, RpcError>,
}

/// Validates one response object: version, an id, and exactly one of
/// `result` and `error`.
pub fn parse_response_value(v: &Value) -> Result<Response, String> {
    let obj = v.as_object().ok_or("response must be an object")?;
    if obj.get("jsonrpc") != Some(&Value::String("2.0".into())) {
        return Err("jsonrpc must be \"2.0\"".into());
    }
    let id = obj
        .get("id")
        .and_then(Id::from_value)
        .ok_or("missing or invalid id")?;
    match (obj.get("result"), obj.get("error")) {
        (Some(r), None) => Ok(Response {
            id,
            outcome: Ok(r.clone()),
        }),
        (None, Some(e)) => {
            let eo = e.as_object().ok_or("error must be an object")?;
            let code = eo
                .get("code")
                .and_then(Value::as_i64)
                .ok_or("error code must be an integer")?;
            let message = eo
                .get("message")
                .and_then(Value::as_str)
                .ok_or("error message must be a string")?
                .to_string();
            Ok(Response {
                id,
                outcome: Err(RpcError {
                    code,
                    message,
                    data: eo.get("data").cloned(),
                }),
            })
        }
        (Some(_), Some(_)) => Err("response has both result and error".into()),
        (None, None) => Err("response has neither result nor error".into()),
    }
}

/// Parses the text of a single response.
pub fn parse_response(text: &str) -> Result<Response, String> {
    let v: Value = serde_json::from_str(text).map_err(|e| format!("invalid json: {e}"))?;
    parse_response_value(&v)
}

/// Why a batch reply could not be matched to the batch that was sent.
#[derive(Debug, Clone, PartialEq)]
pub enum BatchError {
    /// The server answered the whole batch with a single error.
    Rejected(RpcError),
    /// The reply was not a valid response (or array of responses).
    Malformed(String),
    /// A response carried an id that was not sent.
    UnknownId(Id),
    /// Two responses carried the same id.
    DuplicateId(Id),
    /// A request got no response.
    Missing(Id),
}

/// Matches a batch reply to the ids that were sent, returning the outcomes in
/// the order of `sent` (servers may answer in any order).
pub fn match_batch(sent: &[i64], reply: &str) -> Result<Vec<Result<Value, RpcError>>, BatchError> {
    let v: Value = serde_json::from_str(reply)
        .map_err(|e| BatchError::Malformed(format!("invalid json: {e}")))?;
    let items = match v {
        Value::Array(a) => a,
        single => {
            return match parse_response_value(&single) {
                Ok(Response {
                    outcome: Err(e), ..
                }) => Err(BatchError::Rejected(e)),
                Ok(_) => Err(BatchError::Malformed("expected an array".into())),
                Err(m) => Err(BatchError::Malformed(m)),
            };
        }
    };
    let mut seen = BTreeSet::new();
    let mut by_id = std::collections::BTreeMap::new();
    for item in &items {
        let r = parse_response_value(item).map_err(BatchError::Malformed)?;
        if !seen.insert(r.id.clone()) {
            return Err(BatchError::DuplicateId(r.id));
        }
        match r.id {
            Id::Num(n) if sent.contains(&n) => {
                by_id.insert(n, r.outcome);
            }
            other => return Err(BatchError::UnknownId(other)),
        }
    }
    let mut out = Vec::with_capacity(sent.len());
    for n in sent {
        match by_id.remove(n) {
            Some(o) => out.push(o),
            None => return Err(BatchError::Missing(Id::Num(*n))),
        }
    }
    Ok(out)
}

/// Builds the text of a request with positional parameters.
pub fn request_text(id: i64, method: &str, params: Vec<Value>) -> String {
    json!({"jsonrpc": "2.0", "id": id, "method": method, "params": params}).to_string()
}
```

### `src/chain.rs`

```rust
//! A tiny in-memory chain and a mock node that serves it over JSON-RPC.

use crate::jsonrpc::{Handler, Params, RpcError};
use crate::ser;
use serde::{Deserialize, Serialize};
use serde_json::{Value, json};
use sha2::{Digest, Sha256};
use std::collections::BTreeMap;

/// A 32-byte hash.
pub type Hash = [u8; 32];

/// The all-zero parent of the genesis block.
pub const ZERO: Hash = [0; 32];

/// A block as it travels over the wire.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(deny_unknown_fields, rename_all = "camelCase")]
pub struct Block {
    /// Height; genesis is 0.
    #[serde(with = "ser::u64_string")]
    pub number: u64,
    /// Hash of the block contents (see [`Block::compute_hash`]).
    #[serde(with = "ser::hash_hex")]
    pub hash: Hash,
    /// Hash of the parent block.
    #[serde(with = "ser::hash_hex")]
    pub parent_hash: Hash,
    /// Transaction ids included in the block.
    #[serde(with = "ser::hash_list")]
    pub transactions: Vec<Hash>,
}

impl Block {
    /// Builds a block and fills in its hash.
    pub fn new(number: u64, parent_hash: Hash, transactions: Vec<Hash>) -> Block {
        let mut b = Block {
            number,
            hash: ZERO,
            parent_hash,
            transactions,
        };
        b.hash = b.compute_hash();
        b
    }

    /// `SHA-256(len("lab/block/v1") ‖ "lab/block/v1" ‖ number ‖ parent ‖ count ‖ txs)`
    /// with all integers as 8-byte big-endian.
    pub fn compute_hash(&self) -> Hash {
        const DOMAIN: &[u8] = b"lab/block/v1";
        let mut h = Sha256::new();
        h.update((DOMAIN.len() as u64).to_be_bytes());
        h.update(DOMAIN);
        h.update(self.number.to_be_bytes());
        h.update(self.parent_hash);
        h.update((self.transactions.len() as u64).to_be_bytes());
        for t in &self.transactions {
            h.update(t);
        }
        h.finalize().into()
    }

    /// True if the stored hash matches the contents.
    pub fn is_self_consistent(&self) -> bool {
        self.hash == self.compute_hash()
    }
}

/// A transaction id made from a number (for tests and demos).
pub fn tx_hash(n: u64) -> Hash {
    let mut h = Sha256::new();
    h.update(b"lab/tx/");
    h.update(n.to_be_bytes());
    h.finalize().into()
}

/// An in-memory chain with a canonical branch and the orphaned blocks.
#[derive(Debug, Clone)]
pub struct MockChain {
    canonical: Vec<Block>,
    orphans: BTreeMap<Hash, Block>,
}

impl Default for MockChain {
    fn default() -> Self {
        Self::new()
    }
}

impl MockChain {
    /// A chain holding only the genesis block.
    pub fn new() -> MockChain {
        MockChain {
            canonical: vec![Block::new(0, ZERO, vec![])],
            orphans: BTreeMap::new(),
        }
    }

    /// The best block.
    pub fn tip(&self) -> &Block {
        self.canonical.last().expect("genesis always exists")
    }

    /// Appends a block with these transactions.
    pub fn mine(&mut self, txs: Vec<Hash>) -> Block {
        let tip = self.tip();
        let b = Block::new(tip.number + 1, tip.hash, txs);
        self.canonical.push(b.clone());
        b
    }

    /// Removes the last `depth` blocks (they become orphans) and mines one
    /// replacement block per entry of `replacement`.
    pub fn reorg(&mut self, depth: usize, replacement: Vec<Vec<Hash>>) {
        assert!(depth < self.canonical.len(), "cannot remove genesis");
        for _ in 0..depth {
            let b = self.canonical.pop().expect("checked above");
            self.orphans.insert(b.hash, b);
        }
        for txs in replacement {
            self.mine(txs);
        }
    }

    /// The canonical block at this height.
    pub fn get(&self, number: u64) -> Option<&Block> {
        self.canonical.get(usize::try_from(number).ok()?)
    }

    /// Any known block (canonical or orphaned) with this hash.
    pub fn by_hash(&self, hash: &Hash) -> Option<&Block> {
        self.canonical
            .iter()
            .rev()
            .find(|b| &b.hash == hash)
            .or_else(|| self.orphans.get(hash))
    }
}

/// A node serving a [`MockChain`]. `lag` makes it behave like a provider that
/// is that many blocks behind the real tip.
#[derive(Debug, Clone)]
pub struct Node {
    /// The chain served.
    pub chain: MockChain,
    /// How many blocks behind the node pretends to be.
    pub lag: u64,
}

impl Node {
    /// A node with no lag.
    pub fn new(chain: MockChain) -> Node {
        Node { chain, lag: 0 }
    }

    fn visible_tip(&self) -> u64 {
        self.chain.tip().number.saturating_sub(self.lag)
    }
}

fn invalid_params(msg: &str) -> RpcError {
    RpcError::invalid_params(msg)
}

impl Handler for Node {
    fn call(&mut self, method: &str, params: &Params) -> Result<Value, RpcError> {
        let args = params
            .as_array()
            .ok_or_else(|| invalid_params("positional params required"))?;
        match method {
            "getBlockNumber" => {
                if !args.is_empty() {
                    return Err(invalid_params("expected no params"));
                }
                Ok(json!(self.visible_tip().to_string()))
            }
            "getBlockByNumber" => {
                let [arg] = args else {
                    return Err(invalid_params("expected one param"));
                };
                let s = arg
                    .as_str()
                    .ok_or_else(|| invalid_params("param must be a string"))?;
                let n = if s == "latest" {
                    self.visible_tip()
                } else {
                    ser::parse_u64(s).ok_or_else(|| invalid_params("bad block number"))?
                };
                if n > self.visible_tip() {
                    return Ok(Value::Null);
                }
                Ok(self.chain.get(n).map(|b| json!(b)).unwrap_or(Value::Null))
            }
            "getBlockByHash" => {
                let [arg] = args else {
                    return Err(invalid_params("expected one param"));
                };
                let s = arg
                    .as_str()
                    .ok_or_else(|| invalid_params("param must be a string"))?;
                let h = ser::hex_array::<32>(s).map_err(|e| invalid_params(&e.to_string()))?;
                match self.chain.by_hash(&h) {
                    Some(b) if b.number <= self.visible_tip() => Ok(json!(b)),
                    _ => Ok(Value::Null),
                }
            }
            _ => Err(RpcError::method_not_found(method)),
        }
    }
}
```

### `src/client.rs`

```rust
//! A JSON-RPC client with retries, exponential backoff with jitter, a total
//! deadline and failover between endpoints. Time and randomness are injected.

use crate::chain::Node;
use crate::jsonrpc::{self, Id, RpcError};
use serde_json::Value;
use std::collections::VecDeque;
use std::fmt;

/// A failure below the JSON-RPC layer.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum TransportError {
    /// No answer in time. The request **may or may not** have been processed.
    Timeout,
    /// The endpoint refused or could not be reached; the request was not processed.
    Unavailable,
    /// The endpoint asked us to slow down; the request was not processed.
    RateLimited {
        /// Suggested wait before the next attempt.
        retry_after_ms: u64,
    },
}

/// Sends one request text to one endpoint and returns the reply text.
pub trait Transport {
    /// Number of endpoints available.
    fn endpoints(&self) -> usize;
    /// Sends `request` to endpoint `endpoint`.
    fn send(&mut self, endpoint: usize, request: &str) -> Result<String, TransportError>;
}

/// Time source and sleeper.
pub trait Clock {
    /// Milliseconds since an arbitrary start.
    fn now_ms(&self) -> u64;
    /// Waits.
    fn sleep_ms(&mut self, ms: u64);
}

/// A clock that only moves when somebody sleeps.
#[derive(Debug, Default, Clone)]
pub struct FakeClock {
    /// Current time.
    pub now: u64,
    /// Every sleep that was requested, in order.
    pub sleeps: Vec<u64>,
}

impl Clock for FakeClock {
    fn now_ms(&self) -> u64 {
        self.now
    }
    fn sleep_ms(&mut self, ms: u64) {
        self.sleeps.push(ms);
        self.now = self.now.saturating_add(ms);
    }
}

/// A scripted misbehavior of a simulated endpoint, consumed one per request.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Fault {
    /// Return [`TransportError::Timeout`].
    Timeout,
    /// Return [`TransportError::Unavailable`].
    Unavailable,
    /// Return [`TransportError::RateLimited`].
    RateLimited(u64),
    /// Reply with text that is not JSON.
    Garbage,
    /// Process the request but reply with a different id.
    WrongId,
    /// Reply with a JSON-RPC internal error.
    InternalError,
}

/// A simulated set of endpoints, each a [`Node`] with a queue of faults.
#[derive(Debug, Clone)]
pub struct SimTransport {
    /// The endpoints.
    pub nodes: Vec<Node>,
    /// Faults still to be applied, per endpoint.
    pub faults: Vec<VecDeque<Fault>>,
    /// Requests received, per endpoint.
    pub requests: Vec<u64>,
}

impl SimTransport {
    /// Endpoints serving the given nodes, with no faults scripted.
    pub fn new(nodes: Vec<Node>) -> SimTransport {
        let n = nodes.len();
        SimTransport {
            nodes,
            faults: vec![VecDeque::new(); n],
            requests: vec![0; n],
        }
    }

    /// Queues faults for an endpoint.
    pub fn script(&mut self, endpoint: usize, faults: impl IntoIterator<Item = Fault>) {
        self.faults[endpoint].extend(faults);
    }
}

impl Transport for SimTransport {
    fn endpoints(&self) -> usize {
        self.nodes.len()
    }

    fn send(&mut self, endpoint: usize, request: &str) -> Result<String, TransportError> {
        self.requests[endpoint] += 1;
        let fault = self.faults[endpoint].pop_front();
        match fault {
            Some(Fault::Timeout) => return Err(TransportError::Timeout),
            Some(Fault::Unavailable) => return Err(TransportError::Unavailable),
            Some(Fault::RateLimited(ms)) => {
                return Err(TransportError::RateLimited { retry_after_ms: ms });
            }
            Some(Fault::Garbage) => return Ok("<html>502 Bad Gateway</html>".into()),
            Some(Fault::InternalError) => {
                let id = request_id(request);
                let e = RpcError::new(jsonrpc::INTERNAL_ERROR, "internal error");
                return Ok(serde_json::json!({"jsonrpc":"2.0","id":id,"error":{"code":e.code,"message":e.message}}).to_string());
            }
            _ => {}
        }
        let reply = jsonrpc::handle(
            &mut self.nodes[endpoint],
            request,
            &jsonrpc::Limits::default(),
        )
        .unwrap_or_default();
        if fault == Some(Fault::WrongId) {
            let mut v: Value = serde_json::from_str(&reply).unwrap_or(Value::Null);
            if let Some(o) = v.as_object_mut() {
                o.insert("id".into(), serde_json::json!(999_999));
            }
            return Ok(v.to_string());
        }
        Ok(reply)
    }
}

fn request_id(request: &str) -> Value {
    serde_json::from_str::<Value>(request)
        .ok()
        .and_then(|v| v.get("id").cloned())
        .unwrap_or(Value::Null)
}

/// A small seeded generator (xorshift64*), not for security.
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

    /// A value in `0..=max`, with rejection sampling so there is no modulo bias.
    pub fn up_to(&mut self, max: u64) -> u64 {
        if max == u64::MAX {
            return self.next_u64();
        }
        let n = max + 1;
        let limit = u64::MAX - u64::MAX % n;
        loop {
            let x = self.next_u64();
            if x < limit {
                return x % n;
            }
        }
    }
}

/// Retry settings.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct RetryPolicy {
    /// Maximum number of attempts, including the first.
    pub max_attempts: u32,
    /// Backoff before the second attempt, before jitter.
    pub base_ms: u64,
    /// Upper bound for the backoff.
    pub cap_ms: u64,
    /// Total time allowed for the whole call, sleeps included.
    pub deadline_ms: u64,
}

impl Default for RetryPolicy {
    fn default() -> Self {
        RetryPolicy {
            max_attempts: 5,
            base_ms: 100,
            cap_ms: 2_000,
            deadline_ms: 10_000,
        }
    }
}

impl RetryPolicy {
    /// The un-jittered backoff after `attempt` failed attempts (1-based):
    /// `min(cap, base * 2^(attempt-1))`.
    pub fn ceiling(&self, attempt: u32) -> u64 {
        let shift = (attempt.saturating_sub(1)).min(32);
        self.base_ms.saturating_mul(1u64 << shift).min(self.cap_ms)
    }
}

/// Whether repeating a request is safe.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Idempotency {
    /// Reads: repeating changes nothing.
    Safe,
    /// Writes: repeat only when the request certainly was **not** processed.
    Unsafe,
}

/// What went wrong in one attempt.
#[derive(Debug, Clone, PartialEq)]
pub enum AttemptError {
    /// The transport failed.
    Transport(TransportError),
    /// The server answered with a retryable error.
    Rpc(RpcError),
    /// The reply was not a valid answer to this request.
    Bad(String),
}

/// Why a call failed.
#[derive(Debug, Clone, PartialEq)]
pub enum CallError {
    /// The server rejected the request in a way retrying cannot fix.
    Rpc(RpcError),
    /// All attempts failed.
    Exhausted {
        /// Attempts made.
        attempts: u32,
        /// The last failure.
        last: AttemptError,
    },
    /// The deadline would have been exceeded by waiting for another attempt.
    DeadlineExceeded {
        /// Attempts made.
        attempts: u32,
        /// The last failure.
        last: AttemptError,
    },
    /// A write may have been processed; the caller must check before repeating it.
    Uncertain(AttemptError),
}

impl fmt::Display for CallError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            CallError::Rpc(e) => write!(f, "{e}"),
            CallError::Exhausted { attempts, last } => {
                write!(f, "gave up after {attempts} attempts: {last:?}")
            }
            CallError::DeadlineExceeded { attempts, last } => {
                write!(f, "deadline exceeded after {attempts} attempts: {last:?}")
            }
            CallError::Uncertain(last) => write!(f, "outcome unknown: {last:?}"),
        }
    }
}

impl std::error::Error for CallError {}

/// Server errors worth retrying: internal errors and the implementation-defined
/// server range `-32099..=-32000`. Everything else (bad request, unknown
/// method, bad params) will fail the same way again.
pub fn is_retryable_code(code: i64) -> bool {
    code == jsonrpc::INTERNAL_ERROR || (-32099..=-32000).contains(&code)
}

/// The client.
#[derive(Debug)]
pub struct Client<T, C> {
    /// The transport.
    pub transport: T,
    /// The clock.
    pub clock: C,
    policy: RetryPolicy,
    rng: Rng,
    next_id: i64,
    preferred: usize,
}

impl<T: Transport, C: Clock> Client<T, C> {
    /// Creates a client; `seed` drives the jitter.
    pub fn new(transport: T, clock: C, policy: RetryPolicy, seed: u64) -> Self {
        Client {
            transport,
            clock,
            policy,
            rng: Rng::new(seed),
            next_id: 1,
            preferred: 0,
        }
    }

    /// The endpoint tried first by the next call.
    pub fn preferred(&self) -> usize {
        self.preferred
    }

    /// A read call (safe to repeat).
    pub fn call(&mut self, method: &str, params: Vec<Value>) -> Result<Value, CallError> {
        self.call_with(method, params, Idempotency::Safe)
    }

    /// A call with explicit idempotency.
    pub fn call_with(
        &mut self,
        method: &str,
        params: Vec<Value>,
        idem: Idempotency,
    ) -> Result<Value, CallError> {
        let start = self.clock.now_ms();
        let n = self.transport.endpoints();
        let mut endpoint = self.preferred % n;
        let mut attempts = 0u32;
        loop {
            attempts += 1;
            let id = self.next_id;
            self.next_id += 1;
            let text = jsonrpc::request_text(id, method, params.clone());
            let failure = match self.transport.send(endpoint, &text) {
                Err(e) => AttemptError::Transport(e),
                Ok(reply) => match jsonrpc::parse_response(&reply) {
                    Err(m) => AttemptError::Bad(m),
                    Ok(r) if r.id != Id::Num(id) => {
                        AttemptError::Bad(format!("id mismatch: {:?}", r.id))
                    }
                    Ok(r) => match r.outcome {
                        Ok(v) => {
                            self.preferred = endpoint;
                            return Ok(v);
                        }
                        Err(e) if is_retryable_code(e.code) => AttemptError::Rpc(e),
                        Err(e) => return Err(CallError::Rpc(e)),
                    },
                },
            };
            if idem == Idempotency::Unsafe && !certainly_not_processed(&failure) {
                return Err(CallError::Uncertain(failure));
            }
            if attempts >= self.policy.max_attempts {
                return Err(CallError::Exhausted {
                    attempts,
                    last: failure,
                });
            }
            let mut wait = self.rng.up_to(self.policy.ceiling(attempts));
            if let AttemptError::Transport(TransportError::RateLimited { retry_after_ms }) =
                &failure
            {
                wait = wait.max(*retry_after_ms);
            }
            let elapsed = self.clock.now_ms().saturating_sub(start);
            if elapsed.saturating_add(wait) > self.policy.deadline_ms {
                return Err(CallError::DeadlineExceeded {
                    attempts,
                    last: failure,
                });
            }
            self.clock.sleep_ms(wait);
            endpoint = (endpoint + 1) % n;
        }
    }
}

fn certainly_not_processed(e: &AttemptError) -> bool {
    matches!(
        e,
        AttemptError::Transport(TransportError::Unavailable | TransportError::RateLimited { .. })
    )
}
```

### `src/indexer.rs`

```rust
//! A chain follower: keeps a local copy of the canonical chain, detects
//! reorganizations by parent-hash links and reports what changed.

use crate::chain::{Block, Hash, ZERO};
use crate::client::{CallError, Client, Clock, Transport};
use crate::ser;
use serde_json::{Value, json};
use std::collections::BTreeMap;
use std::fmt;

/// Something the indexer changed.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Event {
    /// A block was added to the local chain.
    Applied(Block),
    /// A block was removed from the local chain because of a reorganization.
    RolledBack(Block),
}

/// Why synchronization stopped.
#[derive(Debug, Clone, PartialEq)]
pub enum SyncError {
    /// The source failed.
    Source(String),
    /// The source sent a block whose hash does not match its contents.
    BadBlock(u64),
    /// The source sent a block for another height than requested.
    WrongHeight {
        /// Height asked for.
        asked: u64,
        /// Height received.
        got: u64,
    },
    /// The source's first block does not have the all-zero parent.
    BadGenesis,
    /// Following the source would require undoing more blocks than allowed.
    ReorgTooDeep {
        /// Blocks that would have to be undone.
        depth: u64,
        /// The configured limit.
        limit: u64,
    },
}

impl fmt::Display for SyncError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            SyncError::Source(m) => write!(f, "source error: {m}"),
            SyncError::BadBlock(n) => write!(f, "block {n} does not match its hash"),
            SyncError::WrongHeight { asked, got } => {
                write!(f, "asked for block {asked}, received block {got}")
            }
            SyncError::BadGenesis => write!(f, "block 0 must have the zero parent hash"),
            SyncError::ReorgTooDeep { depth, limit } => {
                write!(f, "reorg of depth {depth} exceeds the limit of {limit}")
            }
        }
    }
}

impl std::error::Error for SyncError {}

/// Where blocks come from.
pub trait BlockSource {
    /// The canonical block at `number`, or `None` if the source has none.
    fn block_at(&mut self, number: u64) -> Result<Option<Block>, SyncError>;
}

/// A [`BlockSource`] that asks a node through a [`Client`].
pub struct RpcSource<'a, T, C> {
    /// The client used for the calls.
    pub client: &'a mut Client<T, C>,
}

impl<T: Transport, C: Clock> BlockSource for RpcSource<'_, T, C> {
    fn block_at(&mut self, number: u64) -> Result<Option<Block>, SyncError> {
        let v = self
            .client
            .call("getBlockByNumber", vec![json!(number.to_string())])
            .map_err(|e: CallError| SyncError::Source(e.to_string()))?;
        if v == Value::Null {
            return Ok(None);
        }
        let b: Block = serde_json::from_value(v).map_err(|e| SyncError::Source(e.to_string()))?;
        Ok(Some(b))
    }
}

/// Status of a transaction as the indexer sees it.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TxStatus {
    /// Not in any block of the local chain.
    Unknown,
    /// In a block, with fewer confirmations than required.
    Included {
        /// Height of the block that contains it.
        height: u64,
        /// Blocks on top of it, counting its own block as 1.
        confirmations: u64,
    },
    /// In a block with at least the required number of confirmations.
    Confirmed {
        /// Height of the block that contains it.
        height: u64,
        /// Confirmations.
        confirmations: u64,
    },
}

/// The follower.
#[derive(Debug, Clone)]
pub struct Indexer {
    chain: BTreeMap<u64, Block>,
    txs: BTreeMap<Hash, u64>,
    required: u64,
    max_reorg: u64,
}

impl Indexer {
    /// `required` confirmations count a transaction as final; a reorg deeper
    /// than `max_reorg` blocks is refused instead of followed.
    pub fn new(required: u64, max_reorg: u64) -> Indexer {
        Indexer {
            chain: BTreeMap::new(),
            txs: BTreeMap::new(),
            required,
            max_reorg,
        }
    }

    /// The local tip.
    pub fn tip(&self) -> Option<&Block> {
        self.chain.values().next_back()
    }

    /// The local block at a height.
    pub fn block(&self, number: u64) -> Option<&Block> {
        self.chain.get(&number)
    }

    /// Number of blocks held.
    pub fn len(&self) -> usize {
        self.chain.len()
    }

    /// True if nothing has been indexed yet.
    pub fn is_empty(&self) -> bool {
        self.chain.is_empty()
    }

    /// Where the transaction is, as far as the local chain knows.
    pub fn status(&self, tx: &Hash) -> TxStatus {
        let Some(&height) = self.txs.get(tx) else {
            return TxStatus::Unknown;
        };
        let tip = self.tip().map_or(0, |b| b.number);
        let confirmations = tip - height + 1;
        if confirmations >= self.required {
            TxStatus::Confirmed {
                height,
                confirmations,
            }
        } else {
            TxStatus::Included {
                height,
                confirmations,
            }
        }
    }

    fn apply(&mut self, b: Block, events: &mut Vec<Event>) {
        for t in &b.transactions {
            self.txs.insert(*t, b.number);
        }
        self.chain.insert(b.number, b.clone());
        events.push(Event::Applied(b));
    }

    fn roll_back(&mut self, events: &mut Vec<Event>) {
        if let Some((_, b)) = self.chain.pop_last() {
            for t in &b.transactions {
                if self.txs.get(t) == Some(&b.number) {
                    self.txs.remove(t);
                }
            }
            events.push(Event::RolledBack(b));
        }
    }

    /// Brings the local chain in line with the source and returns the events.
    ///
    /// The algorithm: first make sure the local tip is still on the source's
    /// chain (undo it otherwise; if the source has no block at that height it
    /// is behind and we simply stop); then append source blocks one by one while
    /// each links to the local tip; a block that does not link means the
    /// source switched branches, so undo the local tip and look again.
    pub fn sync<S: BlockSource>(&mut self, source: &mut S) -> Result<Vec<Event>, SyncError> {
        let mut events = Vec::new();
        let mut undone = 0u64;
        let mut verified = false;
        loop {
            if !verified {
                if let Some(tip) = self.tip().cloned() {
                    // A source with no block at our tip height is *behind*, not
                    // proof of a reorg: wait instead of rolling back.
                    let Some(b) = source.block_at(tip.number)? else {
                        return Ok(events);
                    };
                    if self.checked(b, tip.number)?.hash != tip.hash {
                        self.undo_one(&mut undone, &mut events)?;
                        continue;
                    }
                }
                verified = true;
            }
            let next = self.tip().map_or(0, |b| b.number + 1);
            let Some(b) = source.block_at(next)? else {
                return Ok(events);
            };
            let b = self.checked(b, next)?;
            let expected_parent = self.tip().map_or(ZERO, |t| t.hash);
            if b.parent_hash == expected_parent {
                self.apply(b, &mut events);
            } else if self.is_empty() {
                return Err(SyncError::BadGenesis);
            } else {
                self.undo_one(&mut undone, &mut events)?;
                verified = false;
            }
        }
    }

    fn undo_one(&mut self, undone: &mut u64, events: &mut Vec<Event>) -> Result<(), SyncError> {
        *undone += 1;
        if *undone > self.max_reorg {
            return Err(SyncError::ReorgTooDeep {
                depth: *undone,
                limit: self.max_reorg,
            });
        }
        self.roll_back(events);
        Ok(())
    }

    fn checked(&self, b: Block, asked: u64) -> Result<Block, SyncError> {
        if b.number != asked {
            return Err(SyncError::WrongHeight {
                asked,
                got: b.number,
            });
        }
        if !b.is_self_consistent() {
            return Err(SyncError::BadBlock(b.number));
        }
        Ok(b)
    }
}

/// Parses a hex block hash received in a configuration or a user request.
pub fn parse_hash(s: &str) -> Result<Hash, ser::HexError> {
    ser::hex_array::<32>(s)
}
```

### `src/watch.rs`

```rust
//! A transaction watcher: turns indexer events into at-most-one notice per
//! state change, with ids that let a consumer discard duplicates.

use crate::chain::Hash;
use crate::indexer::{Event, Indexer, TxStatus};
use sha2::{Digest, Sha256};
use std::collections::BTreeMap;

/// What the watcher tells its consumer.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Kind {
    /// The transaction appeared in a block.
    Included,
    /// The transaction reached the required confirmations.
    Confirmed,
    /// The block that held it was rolled back.
    Reorged,
    /// It was not included within the allowed number of blocks.
    Expired,
}

/// A notification.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Notice {
    /// The transaction.
    pub tx: Hash,
    /// What happened.
    pub kind: Kind,
    /// Height involved (0 for `Expired`).
    pub height: u64,
    /// A stable identifier: the same fact always has the same id.
    pub id: Hash,
}

fn notice(tx: Hash, kind: Kind, height: u64, block: &Hash) -> Notice {
    let mut h = Sha256::new();
    h.update(b"lab/notice/v1");
    h.update(tx);
    h.update([kind as u8]);
    h.update(height.to_be_bytes());
    h.update(block);
    Notice {
        tx,
        kind,
        height,
        id: h.finalize().into(),
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum State {
    Waiting { since: u64 },
    Included { height: u64, block: Hash },
    Confirmed { height: u64, block: Hash },
    Expired,
}

/// Watches a set of transactions.
#[derive(Debug, Clone)]
pub struct Watcher {
    watched: BTreeMap<Hash, State>,
    expire_after: u64,
}

impl Watcher {
    /// A transaction not included `expire_after` blocks after it was added is
    /// reported as expired.
    pub fn new(expire_after: u64) -> Watcher {
        Watcher {
            watched: BTreeMap::new(),
            expire_after,
        }
    }

    /// Starts watching a transaction at local height `now`.
    pub fn watch(&mut self, tx: Hash, now: u64) {
        self.watched
            .entry(tx)
            .or_insert(State::Waiting { since: now });
    }

    /// Number of transactions in the table.
    pub fn len(&self) -> usize {
        self.watched.len()
    }

    /// True if nothing is watched.
    pub fn is_empty(&self) -> bool {
        self.watched.is_empty()
    }

    /// Processes events from [`Indexer::sync`] (in order) and returns the notices.
    ///
    /// After the events, confirmations and expiry are evaluated against the
    /// indexer's current tip. Calling it again with no events returns nothing.
    pub fn process(&mut self, events: &[Event], idx: &Indexer) -> Vec<Notice> {
        let mut out = Vec::new();
        let tip = idx.tip().map_or(0, |b| b.number);
        for e in events {
            match e {
                Event::RolledBack(b) => {
                    for t in &b.transactions {
                        let Some(st) = self.watched.get_mut(t) else {
                            continue;
                        };
                        if let State::Included { block, .. } | State::Confirmed { block, .. } = *st
                        {
                            if block == b.hash {
                                out.push(notice(*t, Kind::Reorged, b.number, &b.hash));
                                *st = State::Waiting { since: tip };
                            }
                        }
                    }
                }
                Event::Applied(b) => {
                    for t in &b.transactions {
                        let Some(st) = self.watched.get_mut(t) else {
                            continue;
                        };
                        if matches!(*st, State::Waiting { .. } | State::Expired) {
                            out.push(notice(*t, Kind::Included, b.number, &b.hash));
                            *st = State::Included {
                                height: b.number,
                                block: b.hash,
                            };
                        }
                    }
                }
            }
        }
        for (t, st) in self.watched.iter_mut() {
            match *st {
                State::Included { height, block }
                    if matches!(idx.status(t), TxStatus::Confirmed { .. }) =>
                {
                    out.push(notice(*t, Kind::Confirmed, height, &block));
                    *st = State::Confirmed { height, block };
                }
                State::Waiting { since } if tip.saturating_sub(since) >= self.expire_after => {
                    out.push(notice(*t, Kind::Expired, 0, &[0; 32]));
                    *st = State::Expired;
                }
                _ => {}
            }
        }
        out
    }
}
```

### `tests/ser.rs`

{% raw %}
```rust
use rpclab::ser::*;
use serde::{Deserialize, Serialize};

#[test]
fn hex_is_prefixed_lowercase_and_even() {
    assert_eq!(to_hex(&[]), "0x");
    assert_eq!(to_hex(&[0x00, 0xab, 0xff]), "0x00abff");
    assert_eq!(from_hex("0x00abff"), Ok(vec![0, 0xab, 0xff]));
    assert_eq!(from_hex("0x"), Ok(vec![]));
}

#[test]
fn hex_round_trips_every_byte() {
    let all: Vec<u8> = (0..=255).collect();
    assert_eq!(from_hex(&to_hex(&all)), Ok(all));
}

#[test]
fn hex_refuses_everything_but_the_canonical_form() {
    assert_eq!(from_hex("00ab"), Err(HexError::MissingPrefix));
    assert_eq!(from_hex("0X00ab"), Err(HexError::MissingPrefix));
    assert_eq!(from_hex("0xabc"), Err(HexError::OddLength));
    assert_eq!(
        from_hex("0xAB"),
        Err(HexError::BadDigit(0)),
        "uppercase is not canonical"
    );
    assert_eq!(from_hex("0xaB"), Err(HexError::BadDigit(1)));
    assert_eq!(from_hex("0x0g"), Err(HexError::BadDigit(1)));
    assert_eq!(from_hex("0x 0"), Err(HexError::BadDigit(0)));
    assert_eq!(
        from_hex("0xé"),
        Err(HexError::BadDigit(0)),
        "two bytes of UTF-8, neither a digit"
    );
    assert_eq!(
        from_hex("0xé1"),
        Err(HexError::OddLength),
        "three bytes: the length is counted in bytes"
    );
}

#[test]
fn bad_digit_positions_count_digits_from_zero() {
    assert_eq!(from_hex("0x00zz"), Err(HexError::BadDigit(2)));
    assert_eq!(from_hex("0x000z"), Err(HexError::BadDigit(3)));
}

#[test]
fn fixed_length_arrays() {
    assert_eq!(hex_array::<2>("0x0102"), Ok([1, 2]));
    assert_eq!(
        hex_array::<2>("0x010203"),
        Err(HexError::WrongLength {
            expected: 2,
            got: 3
        })
    );
    assert_eq!(
        hex_array::<2>("0x01"),
        Err(HexError::WrongLength {
            expected: 2,
            got: 1
        })
    );
}

#[test]
fn error_messages_are_readable() {
    assert_eq!(
        HexError::MissingPrefix.to_string(),
        "hex string must start with 0x"
    );
    assert_eq!(
        HexError::OddLength.to_string(),
        "hex string has an odd number of digits"
    );
    assert_eq!(
        HexError::BadDigit(3).to_string(),
        "invalid hex digit at position 3"
    );
    assert_eq!(
        HexError::WrongLength {
            expected: 32,
            got: 3
        }
        .to_string(),
        "expected 32 bytes, got 3"
    );
}

#[test]
fn decimal_u64_is_canonical() {
    assert_eq!(parse_u64("0"), Some(0));
    assert_eq!(parse_u64("7"), Some(7));
    assert_eq!(parse_u64("18446744073709551615"), Some(u64::MAX));
    assert_eq!(parse_u64("18446744073709551616"), None, "one past u64::MAX");
    for bad in [
        "", "00", "01", "+1", "-1", " 1", "1 ", "1.0", "1e3", "0x10", "１",
    ] {
        assert_eq!(parse_u64(bad), None, "{bad:?}");
    }
}

#[derive(Debug, PartialEq, Serialize, Deserialize)]
struct Wire {
    #[serde(with = "u64_string")]
    n: u64,
    #[serde(with = "hash_hex")]
    h: [u8; 32],
    #[serde(with = "hash_list")]
    l: Vec<[u8; 32]>,
}

#[test]
fn serde_adapters_round_trip_and_fix_the_text() {
    let w = Wire {
        n: u64::MAX,
        h: [1; 32],
        l: vec![[2; 32]],
    };
    let text = serde_json::to_string(&w).unwrap();
    assert_eq!(
        text,
        format!(
            r#"{{"n":"18446744073709551615","h":"0x{}","l":["0x{}"]}}"#,
            "01".repeat(32),
            "02".repeat(32)
        )
    );
    assert_eq!(serde_json::from_str::<Wire>(&text).unwrap(), w);
}

#[test]
fn serde_adapters_refuse_numbers_and_bad_strings() {
    let h = format!("0x{}", "00".repeat(32));
    let ok = format!(r#"{{"n":"1","h":"{h}","l":[]}}"#);
    assert!(serde_json::from_str::<Wire>(&ok).is_ok());
    let number = format!(r#"{{"n":1,"h":"{h}","l":[]}}"#);
    assert!(
        serde_json::from_str::<Wire>(&number).is_err(),
        "a JSON number is not accepted"
    );
    let padded = format!(r#"{{"n":"01","h":"{h}","l":[]}}"#);
    assert!(serde_json::from_str::<Wire>(&padded).is_err());
    let short = r#"{"n":"1","h":"0x00","l":[]}"#;
    assert!(serde_json::from_str::<Wire>(short).is_err());
    let bad_list = format!(r#"{{"n":"1","h":"{h}","l":["0x00"]}}"#);
    assert!(serde_json::from_str::<Wire>(&bad_list).is_err());
}

#[test]
fn large_integers_do_not_survive_a_double() {
    // What a JavaScript consumer would do with a JSON number above 2^53.
    let big: u64 = 9_007_199_254_740_993; // 2^53 + 1
    assert_ne!(big as f64 as u64, big);
    // As a decimal string it is exact.
    let w = Wire {
        n: big,
        h: [0; 32],
        l: vec![],
    };
    let back: Wire = serde_json::from_str(&serde_json::to_string(&w).unwrap()).unwrap();
    assert_eq!(back.n, big);
}
```
{% endraw %}

### `tests/jsonrpc.rs`

{% raw %}
```rust
use rpclab::jsonrpc::*;
use serde_json::{Value, json};

/// A handler with a few methods for exercising the dispatcher.
struct Calc {
    calls: Vec<String>,
}

impl Handler for Calc {
    fn call(&mut self, method: &str, params: &Params) -> Result<Value, RpcError> {
        self.calls.push(method.to_string());
        match method {
            "subtract" => {
                let a = params
                    .as_array()
                    .ok_or_else(|| RpcError::invalid_params("array"))?;
                match a {
                    [x, y] => Ok(json!(x.as_i64().unwrap() - y.as_i64().unwrap())),
                    _ => Err(RpcError::invalid_params("two numbers")),
                }
            }
            "null" => Ok(Value::Null),
            "boom" => Err(RpcError {
                code: -32000,
                message: "boom".into(),
                data: Some(json!({"why": "because"})),
            }),
            _ => Err(RpcError::method_not_found(method)),
        }
    }
}

fn run(input: &str) -> Option<Value> {
    let mut h = Calc { calls: vec![] };
    handle(&mut h, input, &Limits::default()).map(|s| serde_json::from_str(&s).unwrap())
}

#[test]
fn a_valid_call_gets_a_result_with_the_same_id() {
    let r = run(r#"{"jsonrpc":"2.0","method":"subtract","params":[42,23],"id":1}"#).unwrap();
    assert_eq!(r, json!({"jsonrpc":"2.0","result":19,"id":1}));
    let r = run(r#"{"jsonrpc":"2.0","method":"subtract","params":[1,1],"id":"abc"}"#).unwrap();
    assert_eq!(r["id"], json!("abc"));
}

#[test]
fn a_null_result_is_still_a_result() {
    let r = run(r#"{"jsonrpc":"2.0","method":"null","id":3}"#).unwrap();
    assert_eq!(r, json!({"jsonrpc":"2.0","result":null,"id":3}));
}

#[test]
fn an_error_object_keeps_its_data() {
    let r = run(r#"{"jsonrpc":"2.0","method":"boom","id":4}"#).unwrap();
    assert_eq!(r["error"]["code"], json!(-32000));
    assert_eq!(r["error"]["message"], json!("boom"));
    assert_eq!(r["error"]["data"], json!({"why":"because"}));
    assert!(r.get("result").is_none());
}

#[test]
fn unknown_methods_and_bad_params_have_their_codes() {
    let r = run(r#"{"jsonrpc":"2.0","method":"foobar","id":"1"}"#).unwrap();
    assert_eq!(r["error"]["code"], json!(METHOD_NOT_FOUND));
    let r = run(r#"{"jsonrpc":"2.0","method":"subtract","params":[1],"id":2}"#).unwrap();
    assert_eq!(r["error"]["code"], json!(INVALID_PARAMS));
}

#[test]
fn a_notification_gets_no_reply_but_is_executed() {
    let mut h = Calc { calls: vec![] };
    let out = handle(
        &mut h,
        r#"{"jsonrpc":"2.0","method":"subtract","params":[1,1]}"#,
        &Limits::default(),
    );
    assert_eq!(out, None);
    assert_eq!(h.calls, vec!["subtract"]);
    // even a failing notification is silent
    let out = handle(
        &mut h,
        r#"{"jsonrpc":"2.0","method":"nope"}"#,
        &Limits::default(),
    );
    assert_eq!(out, None);
}

#[test]
fn an_explicit_null_id_is_not_a_notification() {
    let r = run(r#"{"jsonrpc":"2.0","method":"null","id":null}"#).unwrap();
    assert_eq!(r["id"], Value::Null);
    assert_eq!(r["result"], Value::Null);
}

#[test]
fn parse_errors_answer_with_a_null_id() {
    for bad in [
        "",
        "{",
        r#"{"jsonrpc":"2.0","method":"foobar,"params":"bar","baz]"#,
        "nul",
    ] {
        let r = run(bad).unwrap();
        assert_eq!(r["error"]["code"], json!(PARSE_ERROR), "{bad:?}");
        assert_eq!(r["id"], Value::Null);
    }
}

#[test]
fn invalid_requests_are_refused_with_the_best_id_available() {
    let cases = [
        (
            r#"{"jsonrpc":"2.0","method":1,"params":"bar"}"#,
            Value::Null,
        ),
        (r#"{"jsonrpc":"2.0","method":1,"id":9}"#, json!(9)),
        (r#"{"method":"null","id":5}"#, json!(5)),
        (r#"{"jsonrpc":"1.0","method":"null","id":6}"#, json!(6)),
        (r#"{"jsonrpc":2.0,"method":"null","id":7}"#, json!(7)),
        (
            r#"{"jsonrpc":"2.0","method":"null","params":5,"id":8}"#,
            json!(8),
        ),
        (
            r#"{"jsonrpc":"2.0","method":"null","params":"x","id":10}"#,
            json!(10),
        ),
        (r#"{"jsonrpc":"2.0","method":"null","id":1.5}"#, Value::Null),
        (r#"{"jsonrpc":"2.0","method":"null","id":[1]}"#, Value::Null),
        (
            r#"{"jsonrpc":"2.0","method":"null","id":true}"#,
            Value::Null,
        ),
        (
            r#"{"jsonrpc":"2.0","method":"null","id":18446744073709551615}"#,
            Value::Null,
        ),
        ("1", Value::Null),
        (r#""x""#, Value::Null),
        ("null", Value::Null),
    ];
    for (input, id) in cases {
        let r = run(input).unwrap_or_else(|| panic!("no reply for {input}"));
        assert_eq!(r["error"]["code"], json!(INVALID_REQUEST), "{input}");
        assert_eq!(r["id"], id, "{input}");
    }
}

#[test]
fn a_batch_answers_each_request_and_skips_notifications() {
    let input = r#"[
        {"jsonrpc":"2.0","method":"subtract","params":[10,3],"id":"1"},
        {"jsonrpc":"2.0","method":"subtract","params":[1,1]},
        {"jsonrpc":"2.0","method":"foo.get","params":{"name":"myself"},"id":"5"},
        {"foo":"boo"},
        {"jsonrpc":"2.0","method":"null","id":"9"}
    ]"#;
    let r = run(input).unwrap();
    let a = r.as_array().unwrap();
    assert_eq!(a.len(), 4);
    assert_eq!(a[0], json!({"jsonrpc":"2.0","result":7,"id":"1"}));
    assert_eq!(a[1]["error"]["code"], json!(METHOD_NOT_FOUND));
    assert_eq!(a[1]["id"], json!("5"));
    assert_eq!(a[2]["error"]["code"], json!(INVALID_REQUEST));
    assert_eq!(a[2]["id"], Value::Null);
    assert_eq!(a[3]["result"], Value::Null);
    assert_eq!(a[3]["id"], json!("9"));
}

#[test]
fn a_batch_of_only_notifications_gets_no_reply() {
    let input = r#"[{"jsonrpc":"2.0","method":"null"},{"jsonrpc":"2.0","method":"null"}]"#;
    assert_eq!(run(input), None);
}

#[test]
fn an_empty_batch_is_an_invalid_request() {
    let r = run("[]").unwrap();
    assert_eq!(r["error"]["code"], json!(INVALID_REQUEST));
    assert!(!r.is_array());
}

#[test]
fn a_batch_of_invalid_entries_gets_one_error_each() {
    let r = run("[1,2,3]").unwrap();
    let a = r.as_array().unwrap();
    assert_eq!(a.len(), 3);
    assert!(
        a.iter()
            .all(|e| e["error"]["code"] == json!(INVALID_REQUEST))
    );
}

#[test]
fn limits_are_enforced_before_any_method_runs() {
    let limits = Limits {
        max_bytes: 100,
        max_batch: 2,
    };
    let mut h = Calc { calls: vec![] };
    let big = format!(
        r#"{{"jsonrpc":"2.0","method":"null","id":1,"x":"{}"}}"#,
        "a".repeat(100)
    );
    let r: Value = serde_json::from_str(&handle(&mut h, &big, &limits).unwrap()).unwrap();
    assert_eq!(r["error"]["message"], json!("request too large"));
    let three = r#"[{"jsonrpc":"2.0","method":"null","id":1},{"jsonrpc":"2.0","method":"null","id":2},{"jsonrpc":"2.0","method":"null","id":3}]"#;
    let limits = Limits {
        max_bytes: 10_000,
        max_batch: 2,
    };
    let r: Value = serde_json::from_str(&handle(&mut h, three, &limits).unwrap()).unwrap();
    assert_eq!(r["error"]["message"], json!("batch too large"));
    assert!(h.calls.is_empty(), "nothing was executed");
    // exactly at the limits is fine
    let two =
        r#"[{"jsonrpc":"2.0","method":"null","id":1},{"jsonrpc":"2.0","method":"null","id":2}]"#;
    let r: Value = serde_json::from_str(&handle(&mut h, two, &limits).unwrap()).unwrap();
    assert_eq!(r.as_array().unwrap().len(), 2);
    let exact = Limits {
        max_bytes: two.len(),
        max_batch: 2,
    };
    assert!(handle(&mut h, two, &exact).unwrap().contains("result"));
    let under = Limits {
        max_bytes: two.len() - 1,
        max_batch: 2,
    };
    assert!(handle(&mut h, two, &under).unwrap().contains("too large"));
}

#[test]
fn large_ids_are_kept_exactly() {
    let r = run(&format!(
        r#"{{"jsonrpc":"2.0","method":"null","id":{}}}"#,
        i64::MAX
    ))
    .unwrap();
    assert_eq!(r["id"].as_i64(), Some(i64::MAX));
    let r = run(&format!(
        r#"{{"jsonrpc":"2.0","method":"null","id":{}}}"#,
        i64::MIN
    ))
    .unwrap();
    assert_eq!(r["id"].as_i64(), Some(i64::MIN));
}

#[test]
fn request_text_round_trips_through_the_parser() {
    let text = request_text(7, "subtract", vec![json!(5), json!(2)]);
    let v: Value = serde_json::from_str(&text).unwrap();
    let req = parse_request(&v).unwrap();
    assert_eq!(req.id, Some(Id::Num(7)));
    assert_eq!(req.method, "subtract");
    assert_eq!(req.params, Params::Array(vec![json!(5), json!(2)]));
}

#[test]
fn params_may_be_absent_an_array_or_an_object() {
    let p = |s: &str| {
        parse_request(&serde_json::from_str(s).unwrap())
            .unwrap()
            .params
    };
    assert_eq!(p(r#"{"jsonrpc":"2.0","method":"m"}"#), Params::None);
    assert_eq!(
        p(r#"{"jsonrpc":"2.0","method":"m","params":[]}"#),
        Params::Array(vec![])
    );
    assert!(matches!(
        p(r#"{"jsonrpc":"2.0","method":"m","params":{"a":1}}"#),
        Params::Object(_)
    ));
    assert_eq!(Params::None.as_array(), Some(&[][..]));
    assert!(Params::Object(Default::default()).as_array().is_none());
}

// ---- responses and batches, the client's side

#[test]
fn responses_need_exactly_one_of_result_and_error() {
    let ok = parse_response(r#"{"jsonrpc":"2.0","id":1,"result":5}"#).unwrap();
    assert_eq!(
        ok,
        Response {
            id: Id::Num(1),
            outcome: Ok(json!(5))
        }
    );
    let null = parse_response(r#"{"jsonrpc":"2.0","id":1,"result":null}"#).unwrap();
    assert_eq!(null.outcome, Ok(Value::Null));
    let err = parse_response(
        r#"{"jsonrpc":"2.0","id":"a","error":{"code":-1,"message":"m","data":[1]}}"#,
    )
    .unwrap();
    assert_eq!(
        err.outcome,
        Err(RpcError {
            code: -1,
            message: "m".into(),
            data: Some(json!([1]))
        })
    );
    for bad in [
        r#"{"jsonrpc":"2.0","id":1}"#,
        r#"{"jsonrpc":"2.0","id":1,"result":1,"error":{"code":1,"message":"x"}}"#,
        r#"{"jsonrpc":"2.0","result":1}"#,
        r#"{"jsonrpc":"1.0","id":1,"result":1}"#,
        r#"{"id":1,"result":1}"#,
        r#"{"jsonrpc":"2.0","id":1,"error":"x"}"#,
        r#"{"jsonrpc":"2.0","id":1,"error":{"code":"1","message":"x"}}"#,
        r#"{"jsonrpc":"2.0","id":1,"error":{"code":1}}"#,
        r#"{"jsonrpc":"2.0","id":1.5,"result":1}"#,
        "[1]",
        "not json",
    ] {
        assert!(parse_response(bad).is_err(), "{bad}");
    }
}

#[test]
fn batch_replies_are_matched_by_id_not_by_position() {
    let reply = r#"[
        {"jsonrpc":"2.0","id":12,"result":"c"},
        {"jsonrpc":"2.0","id":10,"result":"a"},
        {"jsonrpc":"2.0","id":11,"error":{"code":-32601,"message":"nope"}}
    ]"#;
    let out = match_batch(&[10, 11, 12], reply).unwrap();
    assert_eq!(out[0], Ok(json!("a")));
    assert_eq!(out[1].as_ref().unwrap_err().code, -32601);
    assert_eq!(out[2], Ok(json!("c")));
}

#[test]
fn a_batch_reply_with_a_missing_unknown_or_repeated_id_is_refused() {
    let one = r#"[{"jsonrpc":"2.0","id":1,"result":1}]"#;
    assert_eq!(
        match_batch(&[1, 2], one),
        Err(BatchError::Missing(Id::Num(2)))
    );
    let stranger = r#"[{"jsonrpc":"2.0","id":1,"result":1},{"jsonrpc":"2.0","id":99,"result":1}]"#;
    assert_eq!(
        match_batch(&[1], stranger),
        Err(BatchError::UnknownId(Id::Num(99)))
    );
    let twice = r#"[{"jsonrpc":"2.0","id":1,"result":1},{"jsonrpc":"2.0","id":1,"result":2}]"#;
    assert_eq!(
        match_batch(&[1], twice),
        Err(BatchError::DuplicateId(Id::Num(1)))
    );
    let null_id = r#"[{"jsonrpc":"2.0","id":null,"result":1}]"#;
    assert_eq!(
        match_batch(&[1], null_id),
        Err(BatchError::UnknownId(Id::Null))
    );
    assert!(matches!(
        match_batch(&[1], "[1]"),
        Err(BatchError::Malformed(_))
    ));
    assert!(matches!(
        match_batch(&[1], "xx"),
        Err(BatchError::Malformed(_))
    ));
}

#[test]
fn a_single_error_for_a_whole_batch_is_reported_as_rejection() {
    let r = r#"{"jsonrpc":"2.0","id":null,"error":{"code":-32600,"message":"batch too large"}}"#;
    match match_batch(&[1, 2], r) {
        Err(BatchError::Rejected(e)) => assert_eq!(e.code, -32600),
        other => panic!("{other:?}"),
    }
    let ok = r#"{"jsonrpc":"2.0","id":1,"result":1}"#;
    assert!(matches!(
        match_batch(&[1], ok),
        Err(BatchError::Malformed(_))
    ));
}

#[test]
fn rpc_errors_display_their_code() {
    assert_eq!(
        RpcError::new(-32000, "x").to_string(),
        "rpc error -32000: x"
    );
    assert_eq!(
        RpcError::method_not_found("m").message,
        "method not found: m"
    );
    assert_eq!(RpcError::invalid_params("p").message, "invalid params: p");
}

#[test]
fn the_default_limits_are_pinned_to_literal_numbers() {
    let l = Limits::default();
    assert_eq!((l.max_bytes, l.max_batch), (1_048_576, 100));
    assert_eq!(
        (
            PARSE_ERROR,
            INVALID_REQUEST,
            METHOD_NOT_FOUND,
            INVALID_PARAMS,
            INTERNAL_ERROR
        ),
        (-32700, -32600, -32601, -32602, -32603)
    );
}
```
{% endraw %}

### `tests/chain.rs`

```rust
use rpclab::chain::*;
use rpclab::jsonrpc::{self, Limits};
use rpclab::ser::to_hex;
use serde_json::{Value, json};

const GENESIS: &str = "0xefdbefc50eec7ec5d8316b713a8da13df22e5138bd0567bbc86408ec1d1eaef5";
const TX1: &str = "0x6e4cf8c33cbb6816c5c14e205631f93435688320cc87d85da7617d57d70d00da";
const BLOCK1: &str = "0x682530731c6d0627a6c456339ccb50d90fd86492d8ddfc4b5d99e54a8c2507d7";

fn ask(node: &mut Node, method: &str, params: Vec<Value>) -> Value {
    let text = jsonrpc::request_text(1, method, params);
    let reply = jsonrpc::handle(node, &text, &Limits::default()).unwrap();
    serde_json::from_str(&reply).unwrap()
}

#[test]
fn block_hashes_match_an_independent_python_implementation() {
    let g = Block::new(0, ZERO, vec![]);
    assert_eq!(to_hex(&g.hash), GENESIS);
    assert_eq!(to_hex(&tx_hash(1)), TX1);
    let b1 = Block::new(1, g.hash, vec![tx_hash(1), tx_hash(2)]);
    assert_eq!(to_hex(&b1.hash), BLOCK1);
}

#[test]
fn the_hash_covers_every_field() {
    let b = Block::new(5, [7; 32], vec![tx_hash(1)]);
    assert!(b.is_self_consistent());
    let mut c = b.clone();
    c.number = 6;
    assert!(!c.is_self_consistent());
    let mut c = b.clone();
    c.parent_hash = [8; 32];
    assert!(!c.is_self_consistent());
    let mut c = b.clone();
    c.transactions.push(tx_hash(2));
    assert!(!c.is_self_consistent());
    let mut c = b.clone();
    c.transactions[0] = tx_hash(9);
    assert!(!c.is_self_consistent());
    let mut c = b;
    c.hash = [0; 32];
    assert!(!c.is_self_consistent());
}

#[test]
fn the_transaction_count_is_part_of_the_hash() {
    // two blocks whose concatenated tx bytes are equal would collide without a count
    let a = Block::new(1, ZERO, vec![[1; 32], [2; 32]]);
    let b = Block::new(1, ZERO, vec![[1; 32]]);
    assert_ne!(a.hash, b.hash);
}

#[test]
fn mining_extends_the_chain_and_reorg_replaces_the_tail() {
    let mut c = MockChain::new();
    let b1 = c.mine(vec![tx_hash(1)]);
    let b2 = c.mine(vec![tx_hash(2)]);
    assert_eq!((b1.number, b2.number), (1, 2));
    assert_eq!(b2.parent_hash, b1.hash);
    assert_eq!(c.tip().number, 2);
    c.reorg(2, vec![vec![tx_hash(3)], vec![tx_hash(4)], vec![]]);
    assert_eq!(c.tip().number, 3);
    assert_eq!(c.get(1).unwrap().transactions, vec![tx_hash(3)]);
    assert_ne!(c.get(1).unwrap().hash, b1.hash);
    // the old blocks are still known by hash, but are no longer canonical
    assert_eq!(c.by_hash(&b1.hash), Some(&b1));
    assert_eq!(c.by_hash(&b2.hash), Some(&b2));
    assert!(c.get(4).is_none());
}

#[test]
#[should_panic(expected = "cannot remove genesis")]
fn genesis_cannot_be_reorged() {
    let mut c = MockChain::new();
    c.reorg(1, vec![]);
}

#[test]
fn the_node_serves_block_numbers_as_decimal_strings() {
    let mut c = MockChain::new();
    for i in 1..=12 {
        c.mine(vec![tx_hash(i)]);
    }
    let mut n = Node::new(c);
    assert_eq!(ask(&mut n, "getBlockNumber", vec![])["result"], json!("12"));
    let r = ask(&mut n, "getBlockByNumber", vec![json!("latest")]);
    assert_eq!(r["result"]["number"], json!("12"));
    let r = ask(&mut n, "getBlockByNumber", vec![json!("0")]);
    assert_eq!(r["result"]["hash"], json!(GENESIS));
    assert_eq!(
        r["result"]["parentHash"],
        json!(format!("0x{}", "00".repeat(32)))
    );
    assert_eq!(
        ask(&mut n, "getBlockByNumber", vec![json!("13")])["result"],
        Value::Null
    );
}

#[test]
fn the_node_serves_blocks_by_hash_including_orphans() {
    let mut c = MockChain::new();
    let b1 = c.mine(vec![tx_hash(1)]);
    c.mine(vec![]);
    c.reorg(2, vec![vec![tx_hash(5)]]);
    let mut n = Node::new(c);
    let r = ask(&mut n, "getBlockByHash", vec![json!(to_hex(&b1.hash))]);
    assert_eq!(
        r["result"]["number"],
        json!("1"),
        "an orphaned block can still be fetched by hash"
    );
    let r = ask(
        &mut n,
        "getBlockByHash",
        vec![json!(format!("0x{}", "11".repeat(32)))],
    );
    assert_eq!(r["result"], Value::Null);
}

#[test]
fn bad_parameters_are_reported_as_invalid_params() {
    let mut n = Node::new(MockChain::new());
    for (m, p) in [
        ("getBlockNumber", vec![json!("x")]),
        ("getBlockByNumber", vec![]),
        ("getBlockByNumber", vec![json!(1)]),
        ("getBlockByNumber", vec![json!("01")]),
        ("getBlockByNumber", vec![json!("-1")]),
        ("getBlockByNumber", vec![json!("0"), json!("0")]),
        ("getBlockByHash", vec![json!("0x00")]),
        ("getBlockByHash", vec![json!(5)]),
        ("getBlockByHash", vec![]),
    ] {
        let r = ask(&mut n, m, p.clone());
        assert_eq!(r["error"]["code"], json!(-32602), "{m} {p:?}");
    }
    let named = jsonrpc::handle(
        &mut n,
        r#"{"jsonrpc":"2.0","id":1,"method":"getBlockNumber","params":{}}"#,
        &Limits::default(),
    )
    .unwrap();
    assert!(
        named.contains("-32602"),
        "named params are not supported by this node"
    );
    assert!(ask(&mut n, "nope", vec![])["error"]["code"] == json!(-32601));
}

#[test]
fn a_lagging_node_hides_recent_blocks() {
    let mut c = MockChain::new();
    for i in 1..=10 {
        c.mine(vec![tx_hash(i)]);
    }
    let tip = c.tip().hash;
    let mut n = Node::new(c);
    n.lag = 3;
    assert_eq!(ask(&mut n, "getBlockNumber", vec![])["result"], json!("7"));
    assert_eq!(
        ask(&mut n, "getBlockByNumber", vec![json!("8")])["result"],
        Value::Null
    );
    assert_ne!(
        ask(&mut n, "getBlockByNumber", vec![json!("7")])["result"],
        Value::Null
    );
    assert_eq!(
        ask(&mut n, "getBlockByHash", vec![json!(to_hex(&tip))])["result"],
        Value::Null
    );
}

#[test]
fn blocks_refuse_unknown_fields_and_wrong_types_on_the_wire() {
    let good = serde_json::to_value(Block::new(1, ZERO, vec![])).unwrap();
    assert!(serde_json::from_value::<Block>(good.clone()).is_ok());
    let mut extra = good.clone();
    extra["extra"] = json!(1);
    assert!(serde_json::from_value::<Block>(extra).is_err());
    let mut num = good;
    num["number"] = json!(1);
    assert!(serde_json::from_value::<Block>(num).is_err());
}
```

### `tests/client.rs`

```rust
use rpclab::chain::{MockChain, Node, tx_hash};
use rpclab::client::*;
use serde_json::{Value, json};

fn nodes(n: usize) -> Vec<Node> {
    (0..n)
        .map(|_| {
            let mut c = MockChain::new();
            for i in 1..=5 {
                c.mine(vec![tx_hash(i)]);
            }
            Node::new(c)
        })
        .collect()
}

fn client(n: usize, policy: RetryPolicy, seed: u64) -> Client<SimTransport, FakeClock> {
    Client::new(
        SimTransport::new(nodes(n)),
        FakeClock::default(),
        policy,
        seed,
    )
}

const POLICY: RetryPolicy = RetryPolicy {
    max_attempts: 5,
    base_ms: 100,
    cap_ms: 2_000,
    deadline_ms: 10_000,
};

#[test]
fn a_healthy_call_needs_one_request_and_no_waiting() {
    let mut c = client(2, POLICY, 1);
    assert_eq!(c.call("getBlockNumber", vec![]), Ok(json!("5")));
    assert_eq!(c.transport.requests, vec![1, 0]);
    assert!(c.clock.sleeps.is_empty());
}

#[test]
fn a_timeout_fails_over_to_the_next_endpoint_after_a_backoff() {
    let mut c = client(2, POLICY, 1);
    c.transport.script(0, [Fault::Timeout]);
    assert_eq!(c.call("getBlockNumber", vec![]), Ok(json!("5")));
    assert_eq!(c.transport.requests, vec![1, 1]);
    assert_eq!(c.clock.sleeps.len(), 1);
    assert_eq!(
        c.preferred(),
        1,
        "the endpoint that worked is tried first next time"
    );
    c.call("getBlockNumber", vec![]).unwrap();
    assert_eq!(c.transport.requests, vec![1, 2]);
}

#[test]
fn the_backoff_ceiling_doubles_up_to_the_cap() {
    let p = POLICY;
    let got: Vec<u64> = (1..=7).map(|a| p.ceiling(a)).collect();
    assert_eq!(got, vec![100, 200, 400, 800, 1600, 2000, 2000]);
    assert_eq!(p.ceiling(0), 100, "attempt 0 is treated like attempt 1");
    assert_eq!(p.ceiling(1000), 2000, "large attempt numbers saturate");
    let huge = RetryPolicy {
        base_ms: u64::MAX,
        cap_ms: u64::MAX,
        ..POLICY
    };
    assert_eq!(huge.ceiling(40), u64::MAX, "no overflow");
    let zero = RetryPolicy {
        base_ms: 0,
        ..POLICY
    };
    assert_eq!(zero.ceiling(3), 0);
}

#[test]
fn jitter_is_within_the_ceiling_and_reproducible() {
    let run = |seed| {
        let mut c = client(
            1,
            RetryPolicy {
                max_attempts: 8,
                deadline_ms: u64::MAX,
                ..POLICY
            },
            seed,
        );
        c.transport
            .script(0, std::iter::repeat_n(Fault::Unavailable, 6));
        c.call("getBlockNumber", vec![]).unwrap();
        c.clock.sleeps.clone()
    };
    let s = run(7);
    assert_eq!(s.len(), 6);
    for (i, w) in s.iter().enumerate() {
        assert!(*w <= POLICY.ceiling(i as u32 + 1), "sleep {i} = {w}");
    }
    assert_eq!(s, run(7), "same seed, same sleeps");
    assert_ne!(s, run(8), "another seed, other sleeps");
}

#[test]
fn jitter_values_match_an_independent_python_computation() {
    let mut c = client(1, POLICY, 1);
    c.transport
        .script(0, std::iter::repeat_n(Fault::Unavailable, 4));
    c.call("getBlockNumber", vec![]).unwrap();
    // xorshift64* outputs for seed 1 reduced modulo ceiling + 1: 100, 200, 400, 800
    assert_eq!(c.clock.sleeps, vec![70, 101, 124, 735]);
}

#[test]
fn a_rate_limited_answer_is_waited_out() {
    let mut c = client(1, POLICY, 1);
    c.transport.script(0, [Fault::RateLimited(1_500)]);
    c.call("getBlockNumber", vec![]).unwrap();
    assert!(c.clock.sleeps[0] >= 1_500, "{:?}", c.clock.sleeps);
    // and a hint smaller than the jitter does not shorten it
    let mut c = client(1, POLICY, 1);
    c.transport.script(0, [Fault::RateLimited(1)]);
    c.call("getBlockNumber", vec![]).unwrap();
    assert_eq!(c.clock.sleeps, vec![70]);
}

#[test]
fn permanent_errors_are_returned_at_once() {
    let mut c = client(2, POLICY, 1);
    match c.call("nope", vec![]) {
        Err(CallError::Rpc(e)) => assert_eq!(e.code, -32601),
        other => panic!("{other:?}"),
    }
    match c.call("getBlockByNumber", vec![json!(5)]) {
        Err(CallError::Rpc(e)) => assert_eq!(e.code, -32602),
        other => panic!("{other:?}"),
    }
    assert_eq!(c.transport.requests, vec![2, 0], "no retry, no failover");
    assert!(c.clock.sleeps.is_empty());
}

#[test]
fn server_errors_garbage_and_wrong_ids_are_retried() {
    for fault in [Fault::InternalError, Fault::Garbage, Fault::WrongId] {
        let mut c = client(2, POLICY, 1);
        c.transport.script(0, [fault.clone()]);
        assert_eq!(
            c.call("getBlockNumber", vec![]),
            Ok(json!("5")),
            "{fault:?}"
        );
        assert_eq!(c.transport.requests, vec![1, 1], "{fault:?}");
    }
}

#[test]
fn giving_up_reports_the_attempts_and_the_last_failure() {
    let mut c = client(
        1,
        RetryPolicy {
            max_attempts: 3,
            ..POLICY
        },
        1,
    );
    c.transport.script(
        0,
        [
            Fault::Timeout,
            Fault::Unavailable,
            Fault::Timeout,
            Fault::Timeout,
        ],
    );
    match c.call("getBlockNumber", vec![]) {
        Err(CallError::Exhausted { attempts, last }) => {
            assert_eq!(attempts, 3);
            assert_eq!(last, AttemptError::Transport(TransportError::Timeout));
        }
        other => panic!("{other:?}"),
    }
    assert_eq!(c.transport.requests, vec![3]);
    assert_eq!(c.clock.sleeps.len(), 2, "no sleep after the final attempt");
}

#[test]
fn the_deadline_stops_the_retries_before_sleeping_past_it() {
    let mut c = client(
        1,
        RetryPolicy {
            max_attempts: 50,
            deadline_ms: 150,
            ..POLICY
        },
        1,
    );
    c.transport
        .script(0, std::iter::repeat_n(Fault::Timeout, 50));
    match c.call("getBlockNumber", vec![]) {
        Err(CallError::DeadlineExceeded { attempts, .. }) => {
            assert!((2..50).contains(&attempts))
        }
        other => panic!("{other:?}"),
    }
    assert!(
        c.clock.now <= 150,
        "never slept past the deadline: {}",
        c.clock.now
    );
}

#[test]
fn the_deadline_is_inclusive() {
    // first jitter for seed 1 is 70 ms: a deadline of exactly 70 allows the sleep, 69 does not
    let mk = |deadline| {
        let mut c = client(
            1,
            RetryPolicy {
                deadline_ms: deadline,
                ..POLICY
            },
            1,
        );
        c.transport.script(0, [Fault::Timeout]);
        c.call("getBlockNumber", vec![])
    };
    assert!(mk(70).is_ok());
    assert!(matches!(mk(69), Err(CallError::DeadlineExceeded { .. })));
}

#[test]
fn writes_are_repeated_only_when_they_certainly_did_not_run() {
    let mut c = client(2, POLICY, 1);
    c.transport.script(0, [Fault::Timeout]);
    let r = c.call_with("getBlockNumber", vec![], Idempotency::Unsafe);
    assert_eq!(
        r,
        Err(CallError::Uncertain(AttemptError::Transport(
            TransportError::Timeout
        )))
    );
    assert_eq!(c.transport.requests, vec![1, 0]);

    let mut c = client(2, POLICY, 1);
    c.transport.script(0, [Fault::Unavailable]);
    assert!(
        c.call_with("getBlockNumber", vec![], Idempotency::Unsafe)
            .is_ok()
    );
    assert_eq!(c.transport.requests, vec![1, 1]);

    let mut c = client(2, POLICY, 1);
    c.transport.script(0, [Fault::RateLimited(10)]);
    assert!(
        c.call_with("getBlockNumber", vec![], Idempotency::Unsafe)
            .is_ok()
    );

    for fault in [Fault::Garbage, Fault::InternalError, Fault::WrongId] {
        let mut c = client(2, POLICY, 1);
        c.transport.script(0, [fault.clone()]);
        let r = c.call_with("getBlockNumber", vec![], Idempotency::Unsafe);
        assert!(
            matches!(r, Err(CallError::Uncertain(_))),
            "{fault:?}: {r:?}"
        );
    }
}

#[test]
fn retryable_codes_are_the_internal_and_server_ranges() {
    for (code, want) in [
        (-32603, true),
        (-32000, true),
        (-32099, true),
        (-32050, true),
        (-32100, false),
        (-31999, false),
        (-32601, false),
        (-32602, false),
        (-32600, false),
        (-32700, false),
        (0, false),
        (3, false),
    ] {
        assert_eq!(is_retryable_code(code), want, "{code}");
    }
}

/// A transport that records the ids of the requests it sees.
struct Recording {
    inner: SimTransport,
    ids: Vec<Value>,
}

impl Transport for Recording {
    fn endpoints(&self) -> usize {
        self.inner.endpoints()
    }
    fn send(&mut self, endpoint: usize, request: &str) -> Result<String, TransportError> {
        let v: Value = serde_json::from_str(request).unwrap();
        self.ids.push(v["id"].clone());
        self.inner.send(endpoint, request)
    }
}

#[test]
fn every_attempt_uses_a_new_id() {
    let mut inner = SimTransport::new(nodes(1));
    inner.script(0, [Fault::Timeout, Fault::Timeout]);
    let mut c = Client::new(
        Recording { inner, ids: vec![] },
        FakeClock::default(),
        POLICY,
        1,
    );
    c.call("getBlockNumber", vec![]).unwrap();
    c.call("getBlockNumber", vec![]).unwrap();
    assert_eq!(
        c.transport.ids,
        vec![json!(1), json!(2), json!(3), json!(4)]
    );
}

#[test]
fn a_late_reply_to_an_old_attempt_cannot_be_mistaken_for_the_current_one() {
    // WrongId models a reply that belongs to another request
    let mut c = client(1, POLICY, 1);
    c.transport.script(0, [Fault::WrongId, Fault::WrongId]);
    assert_eq!(c.call("getBlockNumber", vec![]), Ok(json!("5")));
    assert_eq!(c.transport.requests, vec![3]);
}

#[test]
fn the_generator_matches_the_known_sequence_and_stays_in_range() {
    let mut r = Rng::new(1);
    let got: Vec<u64> = (0..4).map(|_| r.next_u64()).collect();
    assert_eq!(
        got,
        vec![
            5180492295206395165,
            12380297144915551517,
            13389498078930870103,
            5599127315341312413
        ]
    );
    assert_eq!(Rng::new(0).next_u64(), 973819730272012410);
    let mut r = Rng::new(9);
    for max in [0u64, 1, 2, 9, 100, 1_000_000] {
        for _ in 0..200 {
            assert!(r.up_to(max) <= max);
        }
    }
    assert_eq!(Rng::new(1).up_to(0), 0);
    let _ = Rng::new(3).up_to(u64::MAX);
    // every value of a small range appears
    let mut seen = [false; 4];
    let mut r = Rng::new(5);
    for _ in 0..200 {
        seen[r.up_to(3) as usize] = true;
    }
    assert!(seen.iter().all(|s| *s));
}

#[test]
fn the_fake_clock_only_moves_when_slept() {
    let mut c = FakeClock::default();
    assert_eq!(c.now_ms(), 0);
    c.sleep_ms(5);
    c.sleep_ms(7);
    assert_eq!((c.now_ms(), c.sleeps.clone()), (12, vec![5, 7]));
    c.sleep_ms(u64::MAX);
    assert_eq!(c.now_ms(), u64::MAX, "saturates");
}

#[test]
fn errors_are_readable() {
    let e = CallError::Exhausted {
        attempts: 3,
        last: AttemptError::Bad("x".into()),
    };
    assert_eq!(e.to_string(), "gave up after 3 attempts: Bad(\"x\")");
    let e = CallError::DeadlineExceeded {
        attempts: 2,
        last: AttemptError::Bad("y".into()),
    };
    assert_eq!(
        e.to_string(),
        "deadline exceeded after 2 attempts: Bad(\"y\")"
    );
    let e = CallError::Uncertain(AttemptError::Bad("z".into()));
    assert_eq!(e.to_string(), "outcome unknown: Bad(\"z\")");
    let e = CallError::Rpc(jsonrpc_err());
    assert_eq!(e.to_string(), "rpc error -1: m");
}

fn jsonrpc_err() -> rpclab::jsonrpc::RpcError {
    rpclab::jsonrpc::RpcError::new(-1, "m")
}

#[test]
fn a_provider_that_is_behind_still_answers_but_with_older_data() {
    let mut ns = nodes(2);
    ns[0].lag = 2;
    let mut c = Client::new(SimTransport::new(ns), FakeClock::default(), POLICY, 1);
    assert_eq!(
        c.call("getBlockNumber", vec![]),
        Ok(json!("3")),
        "endpoint 0 is two blocks behind"
    );
}

#[test]
fn the_default_retry_policy_is_pinned_to_literal_numbers() {
    let p = RetryPolicy::default();
    assert_eq!(
        (p.max_attempts, p.base_ms, p.cap_ms, p.deadline_ms),
        (5, 100, 2_000, 10_000)
    );
}
```

### `tests/indexer.rs`

```rust
use rpclab::chain::*;
use rpclab::client::*;
use rpclab::indexer::*;
use serde_json::json;

/// A source backed directly by a chain.
struct Direct<'a>(&'a MockChain);

impl BlockSource for Direct<'_> {
    fn block_at(&mut self, n: u64) -> Result<Option<Block>, SyncError> {
        Ok(self.0.get(n).cloned())
    }
}

fn chain_of(n: u64) -> MockChain {
    let mut c = MockChain::new();
    for i in 1..=n {
        c.mine(vec![tx_hash(i)]);
    }
    c
}

fn numbers(ev: &[Event]) -> Vec<(char, u64)> {
    ev.iter()
        .map(|e| match e {
            Event::Applied(b) => ('+', b.number),
            Event::RolledBack(b) => ('-', b.number),
        })
        .collect()
}

fn same_as(idx: &Indexer, c: &MockChain) -> bool {
    idx.len() as u64 == c.tip().number + 1 && (0..=c.tip().number).all(|n| idx.block(n) == c.get(n))
}

#[test]
fn a_fresh_indexer_follows_the_chain_from_genesis() {
    let c = chain_of(5);
    let mut idx = Indexer::new(3, 10);
    assert!(idx.is_empty() && idx.tip().is_none());
    let ev = idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(numbers(&ev), (0..=5).map(|n| ('+', n)).collect::<Vec<_>>());
    assert!(same_as(&idx, &c));
    assert_eq!(idx.tip().unwrap().number, 5);
}

#[test]
fn syncing_again_changes_nothing_and_new_blocks_arrive_incrementally() {
    let mut c = chain_of(3);
    let mut idx = Indexer::new(3, 10);
    idx.sync(&mut Direct(&c)).unwrap();
    assert!(idx.sync(&mut Direct(&c)).unwrap().is_empty(), "idempotent");
    c.mine(vec![]);
    c.mine(vec![]);
    let ev = idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(numbers(&ev), vec![('+', 4), ('+', 5)]);
}

#[test]
fn a_one_block_reorg_rolls_back_and_reapplies() {
    let mut c = chain_of(5);
    let mut idx = Indexer::new(3, 10);
    idx.sync(&mut Direct(&c)).unwrap();
    c.reorg(1, vec![vec![tx_hash(100)]]);
    let ev = idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(numbers(&ev), vec![('-', 5), ('+', 5)]);
    assert!(same_as(&idx, &c));
}

#[test]
fn a_deeper_and_longer_reorg_undoes_newest_first_and_applies_oldest_first() {
    let mut c = chain_of(6);
    let mut idx = Indexer::new(3, 10);
    idx.sync(&mut Direct(&c)).unwrap();
    c.reorg(
        3,
        vec![vec![tx_hash(100)], vec![tx_hash(101)], vec![], vec![]],
    );
    let ev = idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(
        numbers(&ev),
        vec![
            ('-', 6),
            ('-', 5),
            ('-', 4),
            ('+', 4),
            ('+', 5),
            ('+', 6),
            ('+', 7)
        ]
    );
    assert!(same_as(&idx, &c));
}

#[test]
fn the_fork_point_is_not_touched() {
    let mut c = chain_of(6);
    let mut idx = Indexer::new(3, 10);
    idx.sync(&mut Direct(&c)).unwrap();
    let kept = idx.block(3).cloned().unwrap();
    c.reorg(2, vec![vec![tx_hash(100)], vec![tx_hash(101)]]);
    idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(idx.block(3), Some(&kept));
    assert_ne!(idx.block(4).unwrap().hash, kept.hash);
}

#[test]
fn a_source_that_is_behind_does_not_cause_a_rollback() {
    let long = chain_of(8);
    let mut idx = Indexer::new(3, 10);
    idx.sync(&mut Direct(&long)).unwrap();
    let behind = chain_of(5); // same blocks up to 5, nothing after
    let ev = idx.sync(&mut Direct(&behind)).unwrap();
    assert!(ev.is_empty());
    assert_eq!(
        idx.tip().unwrap().number,
        8,
        "waits for the source to catch up"
    );
}

#[test]
fn a_reorg_to_a_shorter_branch_is_noticed_once_the_source_has_blocks_at_our_height() {
    let mut c = chain_of(5);
    let mut idx = Indexer::new(3, 10);
    idx.sync(&mut Direct(&c)).unwrap();
    c.reorg(3, vec![vec![tx_hash(100)]]); // now only 3 blocks tall
    assert!(
        idx.sync(&mut Direct(&c)).unwrap().is_empty(),
        "cannot tell reorg from lag yet"
    );
    for i in 0..3 {
        c.mine(vec![tx_hash(200 + i)]);
    }
    let ev = idx.sync(&mut Direct(&c)).unwrap();
    assert!(ev.iter().any(|e| matches!(e, Event::RolledBack(_))));
    assert!(same_as(&idx, &c));
}

#[test]
fn a_reorg_deeper_than_the_limit_is_refused_and_the_state_is_still_a_valid_prefix() {
    let mut c = chain_of(6);
    let mut idx = Indexer::new(3, 2);
    idx.sync(&mut Direct(&c)).unwrap();
    c.reorg(
        3,
        vec![vec![tx_hash(100)], vec![tx_hash(101)], vec![], vec![]],
    );
    let err = idx.sync(&mut Direct(&c)).unwrap_err();
    assert_eq!(err, SyncError::ReorgTooDeep { depth: 3, limit: 2 });
    assert_eq!(
        idx.tip().unwrap().number,
        4,
        "exactly `limit` blocks were undone"
    );
    // the old chain's blocks 0..=4 are still a consistent chain
    for n in 1..=4 {
        assert_eq!(
            idx.block(n).unwrap().parent_hash,
            idx.block(n - 1).unwrap().hash
        );
    }
    assert_eq!(err.to_string(), "reorg of depth 3 exceeds the limit of 2");
}

#[test]
fn a_reorg_exactly_at_the_limit_is_followed() {
    let mut c = chain_of(6);
    let mut idx = Indexer::new(3, 2);
    idx.sync(&mut Direct(&c)).unwrap();
    c.reorg(2, vec![vec![tx_hash(100)], vec![tx_hash(101)], vec![]]);
    idx.sync(&mut Direct(&c)).unwrap();
    assert!(same_as(&idx, &c));
}

struct Tamper<'a> {
    chain: &'a MockChain,
    mode: u8,
}

impl BlockSource for Tamper<'_> {
    fn block_at(&mut self, n: u64) -> Result<Option<Block>, SyncError> {
        let Some(mut b) = self.chain.get(n).cloned() else {
            return Ok(None);
        };
        if n == 3 {
            match self.mode {
                0 => b.transactions.push(tx_hash(999)), // contents changed, hash not
                1 => b.number = 4,                      // wrong height
                _ => return Err(SyncError::Source("down".into())),
            }
        }
        Ok(Some(b))
    }
}

#[test]
fn blocks_that_do_not_match_their_hash_or_height_are_refused() {
    let c = chain_of(5);
    let mut idx = Indexer::new(3, 10);
    let err = idx.sync(&mut Tamper { chain: &c, mode: 0 }).unwrap_err();
    assert_eq!(err, SyncError::BadBlock(3));
    assert_eq!(
        idx.tip().unwrap().number,
        2,
        "blocks before the bad one were kept"
    );
    let mut idx = Indexer::new(3, 10);
    let err = idx.sync(&mut Tamper { chain: &c, mode: 1 }).unwrap_err();
    assert_eq!(err, SyncError::WrongHeight { asked: 3, got: 4 });
    assert_eq!(err.to_string(), "asked for block 3, received block 4");
    let mut idx = Indexer::new(3, 10);
    let err = idx.sync(&mut Tamper { chain: &c, mode: 2 }).unwrap_err();
    assert_eq!(err, SyncError::Source("down".into()));
    assert_eq!(err.to_string(), "source error: down");
    assert_eq!(idx.tip().unwrap().number, 2);
    assert_eq!(
        SyncError::BadBlock(3).to_string(),
        "block 3 does not match its hash"
    );
}

#[test]
fn the_first_block_must_have_the_zero_parent() {
    struct Evil;
    impl BlockSource for Evil {
        fn block_at(&mut self, n: u64) -> Result<Option<Block>, SyncError> {
            Ok((n == 0).then(|| Block::new(0, [9; 32], vec![])))
        }
    }
    let mut idx = Indexer::new(3, 10);
    assert_eq!(idx.sync(&mut Evil), Err(SyncError::BadGenesis));
    assert!(idx.is_empty());
    assert_eq!(
        SyncError::BadGenesis.to_string(),
        "block 0 must have the zero parent hash"
    );
}

#[test]
fn transaction_status_follows_inclusion_and_confirmations() {
    let mut c = chain_of(1);
    let t = tx_hash(1);
    let mut idx = Indexer::new(3, 10);
    assert_eq!(idx.status(&t), TxStatus::Unknown);
    idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(
        idx.status(&t),
        TxStatus::Included {
            height: 1,
            confirmations: 1
        }
    );
    c.mine(vec![]);
    idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(
        idx.status(&t),
        TxStatus::Included {
            height: 1,
            confirmations: 2
        }
    );
    c.mine(vec![]);
    idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(
        idx.status(&t),
        TxStatus::Confirmed {
            height: 1,
            confirmations: 3
        }
    );
    c.mine(vec![]);
    idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(
        idx.status(&t),
        TxStatus::Confirmed {
            height: 1,
            confirmations: 4
        }
    );
    assert_eq!(idx.status(&tx_hash(77)), TxStatus::Unknown);
}

#[test]
fn a_reorg_can_drop_a_transaction_or_move_it() {
    let mut c = chain_of(4);
    let mut idx = Indexer::new(2, 10);
    idx.sync(&mut Direct(&c)).unwrap();
    let t4 = tx_hash(4);
    assert!(matches!(
        idx.status(&t4),
        TxStatus::Included { height: 4, .. }
    ));
    // dropped: the replacement block does not contain it
    c.reorg(1, vec![vec![]]);
    idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(idx.status(&t4), TxStatus::Unknown);
    // moved: it comes back one block later
    c.reorg(1, vec![vec![t4], vec![]]);
    idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(
        idx.status(&t4),
        TxStatus::Confirmed {
            height: 4,
            confirmations: 2
        }
    );
    c.reorg(2, vec![vec![], vec![t4]]);
    idx.sync(&mut Direct(&c)).unwrap();
    assert_eq!(
        idx.status(&t4),
        TxStatus::Included {
            height: 5,
            confirmations: 1
        }
    );
}

#[test]
fn rolling_back_a_later_block_keeps_the_transactions_of_earlier_blocks() {
    let t = tx_hash(1);
    let mut c = MockChain::new();
    c.mine(vec![t]);
    c.mine(vec![]);
    let mut idx = Indexer::new(5, 10);
    idx.sync(&mut Direct(&c)).unwrap();
    c.reorg(1, vec![vec![]]);
    idx.sync(&mut Direct(&c)).unwrap();
    assert!(
        matches!(idx.status(&t), TxStatus::Included { height: 1, .. }),
        "block 1 is untouched"
    );
}

#[test]
fn random_reorgs_always_end_with_the_same_chain_as_the_source() {
    for seed in 1..=200u64 {
        let mut rng = Rng::new(seed);
        let mut c = chain_of(3);
        let mut idx = Indexer::new(3, 1_000);
        let mut next_tx = 1000;
        for _ in 0..12 {
            match rng.up_to(2) {
                0 => {
                    next_tx += 1;
                    c.mine(vec![tx_hash(next_tx)]);
                }
                _ => {
                    let depth = rng.up_to((c.tip().number).min(4).saturating_sub(1)) as usize + 1;
                    let depth = depth.min(c.tip().number as usize);
                    let n = rng.up_to(3) as usize + depth; // never shorter than before
                    let repl: Vec<_> = (0..n)
                        .map(|_| {
                            next_tx += 1;
                            vec![tx_hash(next_tx)]
                        })
                        .collect();
                    c.reorg(depth, repl);
                }
            }
            if rng.up_to(1) == 0 {
                idx.sync(&mut Direct(&c)).unwrap();
            }
        }
        idx.sync(&mut Direct(&c)).unwrap();
        assert!(same_as(&idx, &c), "seed {seed}");
        // and every transaction the indexer reports is really in the chain
        for h in 0..=c.tip().number {
            for t in &c.get(h).unwrap().transactions {
                assert!(
                    matches!(idx.status(t), TxStatus::Included { height, .. } | TxStatus::Confirmed { height, .. } if height == h),
                    "seed {seed} tx {t:?} h {h} status {:?}",
                    idx.status(t)
                );
            }
        }
    }
}

// ---- through the real JSON-RPC stack

fn rpc_client(nodes: Vec<Node>) -> Client<SimTransport, FakeClock> {
    Client::new(
        SimTransport::new(nodes),
        FakeClock::default(),
        RetryPolicy::default(),
        1,
    )
}

#[test]
fn syncing_through_json_rpc_survives_faults() {
    let c = chain_of(6);
    let mut cl = rpc_client(vec![Node::new(c.clone()), Node::new(c.clone())]);
    cl.transport.script(0, [Fault::Timeout, Fault::Garbage]);
    cl.transport.script(1, [Fault::RateLimited(50)]);
    let mut idx = Indexer::new(3, 10);
    let ev = idx.sync(&mut RpcSource { client: &mut cl }).unwrap();
    assert_eq!(ev.len(), 7);
    assert!(same_as(&idx, &c));
}

#[test]
fn a_provider_that_lies_about_a_hash_is_caught() {
    let c = chain_of(4);
    let mut node = Node::new(c.clone());
    // serve a block whose contents were altered without fixing the hash
    let mut bad = node.chain.clone();
    let _ = &mut bad;
    let mut cl = rpc_client(vec![node.clone()]);
    let mut idx = Indexer::new(3, 10);
    idx.sync(&mut RpcSource { client: &mut cl }).unwrap();
    // now a raw reply with a forged transaction list for block 5
    struct Forger(Node);
    impl Transport for Forger {
        fn endpoints(&self) -> usize {
            1
        }
        fn send(&mut self, e: usize, r: &str) -> Result<String, TransportError> {
            let mut inner = SimTransport::new(vec![self.0.clone()]);
            let mut out = inner.send(e, r)?;
            if r.contains("\"5\"") {
                out = out.replace(
                    "\"transactions\":[",
                    &format!("\"transactions\":[\"0x{}\",", "ab".repeat(32)),
                );
            }
            Ok(out)
        }
    }
    node.chain.mine(vec![tx_hash(5)]);
    let mut cl = Client::new(
        Forger(node),
        FakeClock::default(),
        RetryPolicy::default(),
        1,
    );
    let err = idx.sync(&mut RpcSource { client: &mut cl }).unwrap_err();
    assert_eq!(err, SyncError::BadBlock(5));
}

#[test]
fn a_lagging_provider_after_failover_causes_no_spurious_events() {
    let c = chain_of(8);
    let mut lagging = Node::new(c.clone());
    lagging.lag = 3;
    let mut cl = rpc_client(vec![Node::new(c.clone()), lagging]);
    let mut idx = Indexer::new(3, 10);
    idx.sync(&mut RpcSource { client: &mut cl }).unwrap();
    // endpoint 0 now fails for a while, so the client moves to the lagging one
    cl.transport
        .script(0, std::iter::repeat_n(Fault::Unavailable, 3));
    let ev = idx.sync(&mut RpcSource { client: &mut cl }).unwrap();
    assert!(ev.is_empty(), "{ev:?}");
    assert_eq!(idx.tip().unwrap().number, 8);
}

#[test]
fn rpc_source_reports_a_malformed_block_as_a_source_error() {
    struct Weird;
    impl Transport for Weird {
        fn endpoints(&self) -> usize {
            1
        }
        fn send(&mut self, _: usize, r: &str) -> Result<String, TransportError> {
            let v: serde_json::Value = serde_json::from_str(r).unwrap();
            Ok(json!({"jsonrpc":"2.0","id":v["id"],"result":{"number":1}}).to_string())
        }
    }
    let mut cl = Client::new(Weird, FakeClock::default(), RetryPolicy::default(), 1);
    let err = RpcSource { client: &mut cl }.block_at(0).unwrap_err();
    assert!(matches!(err, SyncError::Source(_)));
}
```

### `tests/watch.rs`

```rust
use rpclab::chain::*;
use rpclab::indexer::*;
use rpclab::ser::to_hex;
use rpclab::watch::*;
use std::collections::BTreeSet;

struct Direct<'a>(&'a MockChain);

impl BlockSource for Direct<'_> {
    fn block_at(&mut self, n: u64) -> Result<Option<Block>, SyncError> {
        Ok(self.0.get(n).cloned())
    }
}

struct Rig {
    chain: MockChain,
    idx: Indexer,
    w: Watcher,
}

impl Rig {
    fn new(required: u64, expire: u64) -> Rig {
        Rig {
            chain: MockChain::new(),
            idx: Indexer::new(required, 100),
            w: Watcher::new(expire),
        }
    }

    fn step(&mut self) -> Vec<Notice> {
        let ev = self.idx.sync(&mut Direct(&self.chain)).unwrap();
        self.w.process(&ev, &self.idx)
    }

    fn kinds(n: &[Notice]) -> Vec<(Kind, u64)> {
        n.iter().map(|x| (x.kind, x.height)).collect()
    }
}

#[test]
fn a_transaction_is_reported_when_included_and_when_confirmed() {
    let mut r = Rig::new(3, 100);
    let t = tx_hash(1);
    r.step();
    r.w.watch(t, 0);
    assert!(r.step().is_empty());
    r.chain.mine(vec![t]);
    assert_eq!(Rig::kinds(&r.step()), vec![(Kind::Included, 1)]);
    r.chain.mine(vec![]);
    assert!(r.step().is_empty());
    r.chain.mine(vec![]);
    let n = r.step();
    assert_eq!(Rig::kinds(&n), vec![(Kind::Confirmed, 1)]);
    assert_eq!(n[0].tx, t);
    r.chain.mine(vec![]);
    assert!(r.step().is_empty(), "each fact is reported once");
    assert!(r.step().is_empty());
}

#[test]
fn inclusion_and_confirmation_can_arrive_together_in_order() {
    let mut r = Rig::new(1, 100);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![t]);
    assert_eq!(
        Rig::kinds(&r.step()),
        vec![(Kind::Included, 1), (Kind::Confirmed, 1)]
    );
}

#[test]
fn transactions_that_are_not_watched_are_ignored() {
    let mut r = Rig::new(1, 100);
    r.chain.mine(vec![tx_hash(1), tx_hash(2)]);
    assert!(r.step().is_empty());
    assert!(r.w.is_empty());
    r.w.watch(tx_hash(9), 0);
    assert_eq!(r.w.len(), 1);
}

#[test]
fn a_reorg_before_confirmation_is_reported_and_the_new_inclusion_too() {
    let mut r = Rig::new(3, 100);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![t]);
    r.chain.mine(vec![]);
    let first = r.step();
    assert_eq!(Rig::kinds(&first), vec![(Kind::Included, 1)]);
    r.chain.reorg(2, vec![vec![], vec![t], vec![]]);
    let n = r.step();
    assert_eq!(
        Rig::kinds(&n),
        vec![(Kind::Reorged, 1), (Kind::Included, 2)],
        "the new block has only 2 of the 3 confirmations"
    );
    r.chain.mine(vec![]);
    assert_eq!(Rig::kinds(&r.step()), vec![(Kind::Confirmed, 2)]);
    assert_ne!(
        n[0].id, first[0].id,
        "a different block gives a different notice"
    );
    assert_eq!(n[0].id.len(), 32);
}

#[test]
fn a_reorg_after_confirmation_is_still_reported() {
    let mut r = Rig::new(2, 100);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![t]);
    r.chain.mine(vec![]);
    assert_eq!(
        Rig::kinds(&r.step()),
        vec![(Kind::Included, 1), (Kind::Confirmed, 1)]
    );
    r.chain.reorg(2, vec![vec![], vec![], vec![]]);
    assert_eq!(Rig::kinds(&r.step()), vec![(Kind::Reorged, 1)]);
    assert_eq!(r.idx.status(&t), TxStatus::Unknown);
}

#[test]
fn a_transaction_that_never_appears_expires_once() {
    let mut r = Rig::new(2, 3);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![]);
    r.chain.mine(vec![]);
    assert!(r.step().is_empty(), "2 blocks since watching: not yet");
    r.chain.mine(vec![]);
    assert_eq!(Rig::kinds(&r.step()), vec![(Kind::Expired, 0)]);
    r.chain.mine(vec![]);
    assert!(r.step().is_empty(), "reported once");
}

#[test]
fn an_expired_transaction_that_shows_up_later_is_still_reported() {
    let mut r = Rig::new(1, 1);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![]);
    assert_eq!(Rig::kinds(&r.step()), vec![(Kind::Expired, 0)]);
    r.chain.mine(vec![t]);
    assert_eq!(
        Rig::kinds(&r.step()),
        vec![(Kind::Included, 2), (Kind::Confirmed, 2)]
    );
}

#[test]
fn the_expiry_clock_restarts_after_a_reorg_drops_the_transaction() {
    let mut r = Rig::new(5, 3);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![t]);
    r.step();
    r.chain.mine(vec![]);
    r.chain.mine(vec![]);
    r.step(); // tip 3, included, would be expired by height alone
    r.chain.reorg(3, vec![vec![], vec![], vec![]]);
    let n = r.step();
    assert_eq!(Rig::kinds(&n), vec![(Kind::Reorged, 1)]);
    r.chain.mine(vec![]);
    r.chain.mine(vec![]);
    assert!(r.step().is_empty(), "two blocks after the reorg");
    r.chain.mine(vec![]);
    assert_eq!(Rig::kinds(&r.step()), vec![(Kind::Expired, 0)]);
}

#[test]
fn watching_twice_does_not_restart_the_clock() {
    let mut r = Rig::new(5, 2);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![]);
    r.step();
    r.w.watch(t, 1);
    assert_eq!(r.w.len(), 1);
    r.chain.mine(vec![]);
    assert_eq!(Rig::kinds(&r.step()), vec![(Kind::Expired, 0)]);
}

#[test]
fn notice_ids_are_stable_and_match_an_independent_computation() {
    let mut r = Rig::new(1, 100);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![t, tx_hash(2)]);
    let n = r.step();
    assert_eq!(
        to_hex(&n[0].id),
        "0xee1f6a744947a4ee6680b343be75e0232468a0c540b445cd67a7761dec0824ad"
    );
    assert_eq!(
        to_hex(&n[1].id),
        "0xf81118a8f916479d9fa2587f8d8fc8a3c06260b0bdf93f7578520e3f366e7546"
    );
    // a second, independent watcher computes the same ids
    let mut r2 = Rig::new(1, 100);
    r2.w.watch(t, 0);
    r2.chain.mine(vec![t, tx_hash(2)]);
    assert_eq!(r2.step(), n);
}

#[test]
fn a_consumer_that_remembers_ids_applies_each_notice_once() {
    let mut r = Rig::new(2, 100);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![t]);
    r.chain.mine(vec![]);
    let notices = r.step();
    // an at-least-once channel delivers everything twice
    let delivered: Vec<&Notice> = notices.iter().chain(notices.iter()).collect();
    let mut seen = BTreeSet::new();
    let mut applied = 0;
    for n in delivered {
        if seen.insert(n.id) {
            applied += 1;
        }
    }
    assert_eq!(applied, notices.len());
}

#[test]
fn a_block_that_is_rolled_back_and_not_the_one_we_hold_is_ignored() {
    let mut r = Rig::new(5, 100);
    let t = tx_hash(1);
    r.w.watch(t, 0);
    r.chain.mine(vec![t]);
    r.step();
    // a rollback event for a different block that happens to list the same tx
    let other = Block::new(7, [3; 32], vec![t]);
    let n = r.w.process(&[Event::RolledBack(other)], &r.idx);
    assert!(n.is_empty());
    assert!(matches!(r.idx.status(&t), TxStatus::Included { .. }));
}

#[test]
fn many_transactions_are_tracked_independently() {
    let mut r = Rig::new(2, 100);
    for i in 1..=50 {
        r.w.watch(tx_hash(i), 0);
    }
    for i in 1..=10u64 {
        r.chain.mine((0..5).map(|j| tx_hash(i * 5 - j)).collect());
    }
    let n = r.step();
    let included = n.iter().filter(|x| x.kind == Kind::Included).count();
    let confirmed = n.iter().filter(|x| x.kind == Kind::Confirmed).count();
    assert_eq!(
        (included, confirmed),
        (50, 45),
        "the 5 transactions of the newest block have 1 of the 2 confirmations"
    );
}
```

### `tests/e2e.rs`

```rust
//! The whole stack: mock node -> JSON-RPC -> retrying client -> indexer -> watcher.

use rpclab::chain::*;
use rpclab::client::*;
use rpclab::indexer::*;
use rpclab::watch::*;

fn client(nodes: Vec<Node>) -> Client<SimTransport, FakeClock> {
    Client::new(
        SimTransport::new(nodes),
        FakeClock::default(),
        RetryPolicy::default(),
        3,
    )
}

#[test]
fn a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation() {
    let pay = tx_hash(42);
    let mut chain = MockChain::new();
    chain.mine(vec![tx_hash(1)]);
    let mut cl = client(vec![Node::new(chain.clone()), Node::new(chain.clone())]);
    let mut idx = Indexer::new(3, 10);
    let mut w = Watcher::new(20);
    let mut log: Vec<Kind> = Vec::new();

    let tick = |cl: &mut Client<SimTransport, FakeClock>,
                chain: &MockChain,
                idx: &mut Indexer,
                w: &mut Watcher,
                log: &mut Vec<Kind>| {
        for n in &mut cl.transport.nodes {
            n.chain = chain.clone();
        }
        let ev = idx.sync(&mut RpcSource { client: cl }).unwrap();
        log.extend(w.process(&ev, idx).iter().map(|n| n.kind));
    };

    tick(&mut cl, &chain, &mut idx, &mut w, &mut log);
    w.watch(pay, idx.tip().unwrap().number);
    chain.mine(vec![pay]);
    cl.transport.script(0, [Fault::Timeout]);
    tick(&mut cl, &chain, &mut idx, &mut w, &mut log);
    chain.mine(vec![]);
    tick(&mut cl, &chain, &mut idx, &mut w, &mut log);
    // the block with the payment is replaced by one that does not hold it
    chain.reorg(2, vec![vec![], vec![], vec![]]);
    cl.transport
        .script(1, [Fault::Garbage, Fault::RateLimited(100)]);
    tick(&mut cl, &chain, &mut idx, &mut w, &mut log);
    // the payment is mined again
    chain.mine(vec![pay]);
    tick(&mut cl, &chain, &mut idx, &mut w, &mut log);
    chain.mine(vec![]);
    chain.mine(vec![]);
    tick(&mut cl, &chain, &mut idx, &mut w, &mut log);

    assert_eq!(
        log,
        vec![
            Kind::Included,
            Kind::Reorged,
            Kind::Included,
            Kind::Confirmed
        ]
    );
    assert!(matches!(
        idx.status(&pay),
        TxStatus::Confirmed {
            height: 5,
            confirmations: 3
        }
    ));
}

#[test]
fn two_indexers_on_different_providers_end_with_identical_chains() {
    let mut chain = MockChain::new();
    for i in 1..=20 {
        chain.mine(vec![tx_hash(i)]);
    }
    chain.reorg(3, vec![vec![tx_hash(100)]; 5]);
    let mut a_nodes = vec![Node::new(chain.clone())];
    a_nodes[0].lag = 0;
    let mut a = client(a_nodes);
    let mut b = client(vec![Node::new(chain.clone()), Node::new(chain.clone())]);
    b.transport
        .script(0, [Fault::InternalError, Fault::WrongId]);
    let mut ia = Indexer::new(3, 10);
    let mut ib = Indexer::new(3, 10);
    ia.sync(&mut RpcSource { client: &mut a }).unwrap();
    ib.sync(&mut RpcSource { client: &mut b }).unwrap();
    for n in 0..=chain.tip().number {
        assert_eq!(ia.block(n), ib.block(n));
    }
    assert_eq!(ia.len(), ib.len());
}

#[test]
fn all_endpoints_down_is_an_error_not_a_hang() {
    let mut cl = client(vec![Node::new(MockChain::new())]);
    cl.transport
        .script(0, std::iter::repeat_n(Fault::Unavailable, 100));
    let mut idx = Indexer::new(1, 1);
    let err = idx.sync(&mut RpcSource { client: &mut cl }).unwrap_err();
    assert!(matches!(err, SyncError::Source(_)));
    assert!(idx.is_empty());
    assert!(
        cl.clock.now <= 10_000,
        "bounded by the deadline: {}",
        cl.clock.now
    );
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
//! Micro-measurements for Lesson 51. Run with `cargo run --release --example bench`.

use rpclab::chain::*;
use rpclab::client::*;
use rpclab::indexer::*;
use rpclab::jsonrpc::{self, Limits};
use serde_json::json;
use std::hint::black_box;
use std::time::Instant;

fn median_ns<F: FnMut()>(mut f: F, iters: u32) -> f64 {
    let mut runs: Vec<f64> = (0..5)
        .map(|_| {
            let t = Instant::now();
            for _ in 0..iters {
                f();
            }
            t.elapsed().as_nanos() as f64 / iters as f64
        })
        .collect();
    runs.sort_by(|a, b| a.partial_cmp(b).unwrap());
    runs[2]
}

fn main() {
    let mut chain = MockChain::new();
    for i in 1..=2000 {
        chain.mine((0..20).map(|j| tx_hash(i * 100 + j)).collect());
    }
    let block = chain.get(1000).unwrap().clone();
    let text = serde_json::to_string(&block).unwrap();
    println!("block with 20 txs: {} bytes of JSON", text.len());

    let enc = median_ns(
        || {
            black_box(serde_json::to_string(black_box(&block)).unwrap());
        },
        20_000,
    );
    let dec = median_ns(
        || {
            black_box(serde_json::from_str::<Block>(black_box(&text)).unwrap());
        },
        20_000,
    );
    let hash = median_ns(
        || {
            black_box(black_box(&block).compute_hash());
        },
        100_000,
    );
    println!("block encode {enc:.0} ns, decode {dec:.0} ns, verify hash {hash:.0} ns");

    let mut node = Node::new(chain.clone());
    let req = jsonrpc::request_text(1, "getBlockByNumber", vec![json!("1000")]);
    let limits = Limits::default();
    let one = median_ns(
        || {
            black_box(jsonrpc::handle(&mut node, black_box(&req), &limits));
        },
        20_000,
    );
    println!("single request, parse + dispatch + encode: {one:.0} ns");

    let batch: Vec<_> = (0..50)
        .map(|i| json!({"jsonrpc":"2.0","id":i,"method":"getBlockByNumber","params":[i.to_string()]}))
        .collect();
    let batch_text = serde_json::Value::Array(batch).to_string();
    let b50 = median_ns(
        || {
            black_box(jsonrpc::handle(&mut node, black_box(&batch_text), &limits));
        },
        2_000,
    );
    println!(
        "batch of 50 requests: {b50:.0} ns ({:.0} ns each)",
        b50 / 50.0
    );

    // indexing 2000 blocks straight from memory and through the JSON-RPC stack
    struct Direct<'a>(&'a MockChain);
    impl BlockSource for Direct<'_> {
        fn block_at(&mut self, n: u64) -> Result<Option<Block>, SyncError> {
            Ok(self.0.get(n).cloned())
        }
    }
    let t = Instant::now();
    let mut idx = Indexer::new(6, 100);
    idx.sync(&mut Direct(&chain)).unwrap();
    let direct = t.elapsed();
    let t = Instant::now();
    let mut cl = Client::new(
        SimTransport::new(vec![Node::new(chain.clone())]),
        FakeClock::default(),
        RetryPolicy::default(),
        1,
    );
    let mut idx2 = Indexer::new(6, 100);
    idx2.sync(&mut RpcSource { client: &mut cl }).unwrap();
    let rpc = t.elapsed();
    println!(
        "index {} blocks (20 txs each): in memory {:.2} ms, through JSON-RPC {:.2} ms ({:.1} µs per block), {} requests",
        idx.len(),
        direct.as_secs_f64() * 1e3,
        rpc.as_secs_f64() * 1e3,
        rpc.as_secs_f64() * 1e6 / idx2.len() as f64,
        cl.transport.requests[0]
    );

    // reorg handling
    let mut c2 = chain.clone();
    c2.reorg(50, vec![vec![]; 50]);
    let t = Instant::now();
    let ev = idx.sync(&mut Direct(&c2)).unwrap();
    println!(
        "reorg of depth 50: {} events in {:.3} ms",
        ev.len(),
        t.elapsed().as_secs_f64() * 1e3
    );

    // retry arithmetic
    let p = RetryPolicy::default();
    let total: u64 = (1..p.max_attempts).map(|a| p.ceiling(a)).sum();
    println!(
        "default policy: worst-case total sleep without jitter reduction = {total} ms over {} attempts",
        p.max_attempts
    );
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **97 tests** (ser 10, JSON-RPC 22, chain 10, client 20, indexer 19, watcher 13, end-to-end 3) and 8 consecutive full runs without a failure (nothing uses sockets, real time or the OS random source, so there is nothing to flake).

**Mutation check ✓.** I made 116 one-line changes (a "mutant") across the six source modules, each in its own copy of the crate, and ran the whole test suite against each. **112 were detected (111 by a failing or aborting test, 1 as a hang), 4 are equivalent (M62, M63, M87, M94), none survive unexplained.** Four mutants survived the *first* run. I looked at each: M62 and M63 change the rejection-sampling limit of the jitter generator and differ only with probability about 2⁻⁶⁴; M87 changes a guard that is always true, because rollback removes the newest block and a transaction in it is recorded at that height; and M94 only removes a redundant re-check, so the final state is the same and only the number of fetches differs. All four are equivalent, so I added no tests for them. Six other mutants were not valid at first (two failed to compile for lack of a type annotation, three no longer matched the source after `cargo fmt` rewrapped it, and one was a nonsense edit I dropped); I rewrote the five worth keeping and re-ran them, and all are caught. Before the run I had already pinned the default limits, the retry policy and the protocol error codes with literal-number tests, because the same gap appeared in Lesson 50. Mutants that hang (M101) loop forever in a test that waits for progress, which counts as detected by timeout. As in the previous lessons, my harness treats `#[should_panic]` failures (`- should panic ... FAILED`) as failures, and a process abort as detection. The tests ran in the dev profile.

| # | module | change | result |
|---|---|---|---|
| M1 | `ser` | odd-length check removed | caught by `hex_refuses_everything_but_the_canonical_form` |
| M2 | `ser` | missing `0x` prefix accepted | caught by `hex_refuses_everything_but_the_canonical_form` |
| M3 | `ser` | digits encoded in uppercase | caught by `block_hashes_match_an_independent_python_implementation` |
| M4 | `ser` | high nibble shifted by 3 | caught by `a_lagging_node_hides_recent_blocks` |
| M5 | `ser` | low nibble masked with 7 | caught by `block_hashes_match_an_independent_python_implementation` |
| M6 | `ser` | `a` decodes to 9 instead of 10 | caught by `the_node_serves_blocks_by_hash_including_orphans` |
| M7 | `ser` | `g` accepted as a hex digit | caught by `hex_refuses_everything_but_the_canonical_form` |
| M8 | `ser` | decoded nibbles combined with a shift of 3 | caught by `the_node_serves_blocks_by_hash_including_orphans` |
| M9 | `ser` | `BadDigit` position of the low digit off by one | caught by `bad_digit_positions_count_digits_from_zero` |
| M10 | `ser` | `BadDigit` position of the high digit off by one | caught by `bad_digit_positions_count_digits_from_zero` |
| M11 | `ser` | `WrongLength` reports the wrong `got` | caught by `fixed_length_arrays` |
| M12 | `ser` | leading zeros accepted in `parse_u64` | caught by `bad_parameters_are_reported_as_invalid_params` |
| M13 | `ser` | leading-zero rule applies only above two digits | caught by `bad_parameters_are_reported_as_invalid_params` |
| M14 | `ser` | non-digit characters accepted in `parse_u64` | caught by `decimal_u64_is_canonical` |
| M15 | `ser` | `Display` text of `MissingPrefix` changed | caught by `error_messages_are_readable` |
| M16 | `ser` | `Display` text of `BadDigit` changed | caught by `error_messages_are_readable` |
| M17 | `jsonrpc` | request version not checked | caught by `invalid_requests_are_refused_with_the_best_id_available` |
| M18 | `jsonrpc` | response version not checked | caught by `responses_need_exactly_one_of_result_and_error` |
| M19 | `jsonrpc` | fractional ids accepted | caught by `invalid_requests_are_refused_with_the_best_id_available` |
| M20 | `jsonrpc` | notifications get a reply | caught by `a_batch_answers_each_request_and_skips_notifications` |
| M21 | `jsonrpc` | size limit `>=` instead of `>` | caught by `limits_are_enforced_before_any_method_runs` |
| M22 | `jsonrpc` | empty batch accepted | caught by `an_empty_batch_is_an_invalid_request` |
| M23 | `jsonrpc` | batch limit `>=` instead of `>` | caught by `limits_are_enforced_before_any_method_runs` |
| M24 | `jsonrpc` | empty batch reply sent as `[]` | caught by `a_batch_of_only_notifications_gets_no_reply` |
| M25 | `jsonrpc` | duplicate response ids not detected | caught by `a_batch_reply_with_a_missing_unknown_or_repeated_id_is_refused` |
| M26 | `jsonrpc` | unknown numeric ids accepted in a batch reply | caught by `a_batch_reply_with_a_missing_unknown_or_repeated_id_is_refused` |
| M27 | `jsonrpc` | missing batch replies skipped | caught by `a_batch_reply_with_a_missing_unknown_or_repeated_id_is_refused` |
| M28 | `jsonrpc` | whole-batch rejection reported as malformed | caught by `a_single_error_for_a_whole_batch_is_reported_as_rejection` |
| M29 | `jsonrpc` | `PARSE_ERROR` is -32701 | caught by `the_default_limits_are_pinned_to_literal_numbers` |
| M30 | `jsonrpc` | `INVALID_REQUEST` is -32601 | caught by `the_default_limits_are_pinned_to_literal_numbers` |
| M31 | `jsonrpc` | `METHOD_NOT_FOUND` is -32600 | caught by `bad_parameters_are_reported_as_invalid_params` |
| M32 | `jsonrpc` | `INVALID_PARAMS` is -32603 | caught by `bad_parameters_are_reported_as_invalid_params` |
| M33 | `jsonrpc` | invalid requests always answered with a null id | caught by `invalid_requests_are_refused_with_the_best_id_available` |
| M34 | `jsonrpc` | default `max_bytes` is 2 MiB | caught by `the_default_limits_are_pinned_to_literal_numbers` |
| M35 | `jsonrpc` | default `max_batch` is 101 | caught by `the_default_limits_are_pinned_to_literal_numbers` |
| M36 | `jsonrpc` | response with neither result nor error accepted | caught by `responses_need_exactly_one_of_result_and_error` |
| M37 | `jsonrpc` | array `params` dropped | caught by `bad_parameters_are_reported_as_invalid_params` |
| M38 | `jsonrpc` | every id replaced by null | caught by `a_late_reply_to_an_old_attempt_cannot_be_mistaken_for_the_current_one` |
| M39 | `chain` | block number not in the hash | caught by `block_hashes_match_an_independent_python_implementation` |
| M40 | `chain` | parent hash not in the hash | caught by `block_hashes_match_an_independent_python_implementation` |
| M41 | `chain` | transaction count not in the hash | caught by `block_hashes_match_an_independent_python_implementation` |
| M42 | `chain` | transactions not in the hash | caught by `block_hashes_match_an_independent_python_implementation` |
| M43 | `chain` | domain label not in the hash | caught by `block_hashes_match_an_independent_python_implementation` |
| M44 | `chain` | domain length not in the hash | caught by `block_hashes_match_an_independent_python_implementation` |
| M45 | `chain` | `reorg` may remove genesis | caught by `genesis_cannot_be_reorged` |
| M46 | `chain` | orphaned blocks cannot be fetched by hash | caught by `mining_extends_the_chain_and_reorg_replaces_the_tail` |
| M47 | `chain` | `getBlockNumber` accepts parameters | caught by `bad_parameters_are_reported_as_invalid_params` |
| M48 | `chain` | `getBlockByNumber` hides the visible tip | caught by `a_lagging_node_hides_recent_blocks` |
| M49 | `chain` | `getBlockByHash` hides the visible tip | caught by `the_node_serves_blocks_by_hash_including_orphans` |
| M50 | `chain` | `lag` makes the node ahead instead of behind | caught by `a_lagging_node_hides_recent_blocks` |
| M51 | `chain` | `"latest"` spelled `"late"` | caught by `the_node_serves_block_numbers_as_decimal_strings` |
| M52 | `chain` | mined block links to the zero hash | caught by `mining_extends_the_chain_and_reorg_replaces_the_tail` |
| M53 | `chain` | `is_self_consistent` inverted | caught by `the_hash_covers_every_field` |
| M54 | `client` | backoff shift capped at 64 | caught by `the_backoff_ceiling_doubles_up_to_the_cap` |
| M55 | `client` | backoff multiplication wraps | caught by `the_backoff_ceiling_doubles_up_to_the_cap` |
| M56 | `client` | backoff not capped | caught by `the_backoff_ceiling_doubles_up_to_the_cap` |
| M57 | `client` | xorshift shift 11 instead of 12 | caught by `jitter_values_match_an_independent_python_computation` |
| M58 | `client` | xorshift shift 24 instead of 25 | caught by `a_rate_limited_answer_is_waited_out` |
| M59 | `client` | xorshift shift 26 instead of 27 | caught by `jitter_values_match_an_independent_python_computation` |
| M60 | `client` | even multiplier constant | caught by `a_rate_limited_answer_is_waited_out` |
| M61 | `client` | seed 0 not remapped | caught by `a_rate_limited_answer_is_waited_out` |
| M62 | `client` | rejection-sampling limit removed | **equivalent**: the two limits differ only when a draw lands in the last `2^64 mod n` values, probability about 2⁻⁶⁴ |
| M63 | `client` | rejection test `<=` instead of `<` | **equivalent**: `x == limit` happens with probability about 2⁻⁶⁴ |
| M64 | `client` | `up_to(u64::MAX)` special case removed | caught by `the_generator_matches_the_known_sequence_and_stays_in_range` |
| M65 | `client` | `up_to` returns `x % (n + 1)` | caught by `a_rate_limited_answer_is_waited_out` |
| M66 | `client` | fake clock wraps instead of saturating | caught by `the_fake_clock_only_moves_when_slept` |
| M67 | `client` | fake clock does not record sleeps | caught by `a_rate_limited_answer_is_waited_out` |
| M68 | `client` | `-32603` not retryable | caught by `retryable_codes_are_the_internal_and_server_ranges` |
| M69 | `client` | `-32000` not retryable | caught by `retryable_codes_are_the_internal_and_server_ranges` |
| M70 | `client` | `-32099` not retryable | caught by `retryable_codes_are_the_internal_and_server_ranges` |
| M71 | `client` | working endpoint not remembered | caught by `a_timeout_fails_over_to_the_next_endpoint_after_a_backoff` |
| M72 | `client` | server errors never retried | caught by `server_errors_garbage_and_wrong_ids_are_retried` |
| M73 | `client` | writes retried like reads | caught by `writes_are_repeated_only_when_they_certainly_did_not_run` |
| M74 | `client` | one attempt too many | caught by `giving_up_reports_the_attempts_and_the_last_failure` |
| M75 | `client` | jitter uses the next attempt's ceiling | caught by `a_rate_limited_answer_is_waited_out` |
| M76 | `client` | retry-after hint shortens the wait | caught by `a_rate_limited_answer_is_waited_out` |
| M77 | `client` | deadline `>=` instead of `>` | caught by `the_deadline_is_inclusive` |
| M78 | `client` | no failover to the next endpoint | caught by `a_timeout_fails_over_to_the_next_endpoint_after_a_backoff` |
| M79 | `client` | reply id not checked | caught by `a_late_reply_to_an_old_attempt_cannot_be_mistaken_for_the_current_one` |
| M80 | `client` | request id not incremented | caught by `every_attempt_uses_a_new_id` |
| M81 | `client` | rate-limited writes treated as uncertain | caught by `writes_are_repeated_only_when_they_certainly_did_not_run` |
| M82 | `client` | elapsed time always 0 | caught by `the_deadline_stops_the_retries_before_sleeping_past_it` |
| M83 | `client` | wrong-id fault disabled in the simulator | caught by `a_late_reply_to_an_old_attempt_cannot_be_mistaken_for_the_current_one` |
| M84 | `indexer` | confirmations off by one | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M85 | `indexer` | `Confirmed` needs more than the required confirmations | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M86 | `indexer` | transactions not recorded | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M87 | `indexer` | rollback forgets a transaction unconditionally | **equivalent**: rollback always removes the newest block, and a transaction in it is recorded at that height, so the guard is always true |
| M88 | `indexer` | rollback does not forget transactions | caught by `a_reorg_can_drop_a_transaction_or_move_it` |
| M89 | `indexer` | tip never re-verified | caught by `a_one_block_reorg_rolls_back_and_reapplies` |
| M90 | `indexer` | tip comparison inverted | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M91 | `indexer` | next block requested two ahead | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M92 | `indexer` | parent link checked against the zero hash | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M93 | `indexer` | bad genesis not detected | caught by `the_first_block_must_have_the_zero_parent` |
| M94 | `indexer` | tip not re-verified after a rollback | **equivalent in outcome**: without the re-check the loop fetches the next block, finds the link broken and undoes again; the final state is the same, only the number of fetches differs (not pinned by a test) |
| M95 | `indexer` | reorg bound `>=` instead of `>` | caught by `a_reorg_deeper_than_the_limit_is_refused_and_the_state_is_still_a_valid_prefix` |
| M96 | `indexer` | rollback counter not incremented | caught by `a_reorg_deeper_than_the_limit_is_refused_and_the_state_is_still_a_valid_prefix` |
| M97 | `indexer` | block height not checked | caught by `blocks_that_do_not_match_their_hash_or_height_are_refused` |
| M98 | `indexer` | block hash not checked | caught by `a_provider_that_lies_about_a_hash_is_caught` |
| M99 | `indexer` | null replies parsed as blocks | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M100 | `indexer` | rollback removes block 0 | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M101 | `indexer` | applied blocks not stored | detected as a **hang** (killed after 200 s) |
| M102 | `indexer` | applied blocks reported as rolled back | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M103 | `indexer` | rolled-back blocks reported as applied | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M104 | `watch` | notice kind not in the id | caught by `a_consumer_that_remembers_ids_applies_each_notice_once` |
| M105 | `watch` | height not in the id | caught by `notice_ids_are_stable_and_match_an_independent_computation` |
| M106 | `watch` | block hash not in the id | caught by `notice_ids_are_stable_and_match_an_independent_computation` |
| M107 | `watch` | transaction id not in the id | caught by `notice_ids_are_stable_and_match_an_independent_computation` |
| M108 | `watch` | id domain label changed | caught by `notice_ids_are_stable_and_match_an_independent_computation` |
| M109 | `watch` | rollback of any block with the same transaction accepted | caught by `a_block_that_is_rolled_back_and_not_the_one_we_hold_is_ignored` |
| M110 | `watch` | expiry clock not restarted after a reorg | caught by `the_expiry_clock_restarts_after_a_reorg_drops_the_transaction` |
| M111 | `watch` | expired transactions are never reported as included | caught by `an_expired_transaction_that_shows_up_later_is_still_reported` |
| M112 | `watch` | confirmation reported without checking the indexer | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M113 | `watch` | expiry `>` instead of `>=` | caught by `a_transaction_that_never_appears_expires_once` |
| M114 | `watch` | watching twice restarts the clock | caught by `watching_twice_does_not_restart_the_clock` |
| M115 | `watch` | reorg reported as an inclusion | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |
| M116 | `watch` | confirmations never reported | caught by `a_payment_is_followed_from_inclusion_through_a_reorg_to_confirmation` |

What the table does not show: it tests *these* edits, not every possible bug; there is no mutant for the simulated transport itself beyond two spots (it is test infrastructure); the equivalent mutants could hide a real gap if my reasoning about them is wrong; and passing every mutant says nothing about security or about behavior against a real provider.

**Design notes.**

- Every seam is a small trait (`Handler`, `Transport`, `Clock`, `BlockSource`); the logic around them is deterministic and fast to test, and the full stack runs in-process (`tests/e2e.rs`).
- Rules are separate from I/O: `parse_request` and `parse_response_value` are pure; `handle` adds limits; the client adds time and failure policy; the indexer adds chain semantics; the watcher adds notification semantics. Each layer's tests violate exactly one rule.
- Hashes, notice ids and jitter values are checked against independent Python computations rather than only against the code that produces them.
- Not handled: real transports, subscriptions, pending-transaction APIs, signing and submission, persistence, provider cross-checking, circuit breakers, and everything in §10.

</details>

---

## 15. Key takeaways

- A boundary needs one canonical spelling for each value, exact integers (strings, not doubles) and validation at parse time (✓ `2⁵³ + 1` does not survive an `f64`; hex has one spelling).
- JSON-RPC's corner cases are where clients and servers disagree: `null` results, `null` ids, notifications, empty batches. Limit the input before running anything, and match batch replies by id (✓ 22 tests).
- Retry what can succeed, not what cannot; add exponential backoff with jitter, a cap and a deadline; treat a timed-out write as **uncertain** and look it up by id instead of repeating it (✓ jitter 70, 101, 124, 735 ms for seed 1).
- Never trust what you did not compute: recompute hashes, check parent links, bound reorg depth (✓ a forged transaction list is rejected as `BadBlock`).
- An indexer is a loop that checks its tip, appends while blocks link, and rolls back when they do not; a source that is behind is not a reorg (✓ 200 random seeds end with the node's chain; a lagging failover target causes no spurious events).
- Finality is a policy parameter; `included` is not `final`, `dropped` is not knowable from blocks alone, and every user-facing state needs a `reorged` case.
- Notifications are at-least-once; make the effects idempotent with stable ids derived from the fact (✓ known-answer ids; duplicates applied once).
- Persist data and cursor in one transaction, use idempotent upserts, and verify the stored tip against the node on restart.
- Mutation testing found no new test gaps this time, because I applied the lesson of Lesson 50 first (pin limits and defaults as literal numbers); the four survivors of the first run are equivalent mutants with stated reasons, and five mutants needed their patterns fixed before they were valid. That is a cheaper outcome than in Lessons 48 to 50, not proof that the tests are complete.

**Connections.** [Lesson 50](50_networking_determinism.md) supplied the ingredients (limits before allocation, sans-IO with injected time, seeded simulation, canonical encodings) that this lesson applies to RPC and to chain following. [Lesson 49](49_merkle_transactions_consensus.md) explains blocks, forks, confirmations and trust assumptions; the indexer and the watcher implement their operational consequences. [Lesson 48](48_crypto_primitives.md) provides the hash and the domain separation used for block hashes and notice ids. [Lessons 45 and 46](45_http_apis.md) cover HTTP semantics, idempotency and database transactions, which §5.3, §6.8 and §7.3 reuse. [Lesson 44](44_cli_serialization.md) introduced `serde`. [Lesson 40](40_property_testing_fuzzing.md) explains the testing style of the random-reorg test. Lesson 52 continues with off-chain services, cryptographic integrations and fault tolerance.

---

## 16. References

- The JSON-RPC 2.0 specification: <https://www.jsonrpc.org/specification>
- RFC 8259, The JSON Data Interchange Format (for number handling and interoperability): <https://www.rfc-editor.org/>
- IEEE 754 floating-point arithmetic (from memory, for the 2⁵³ statement).
- Marc Brooker, "Exponential Backoff And Jitter", AWS Architecture Blog (2015, from memory), for the full-jitter variant.
- `serde` documentation (container and field attributes, `deny_unknown_fields`, `with`): <https://serde.rs/>
- `serde_json`: <https://github.com/serde-rs/json>; RustCrypto `sha2`: <https://github.com/RustCrypto/hashes>
- Documentation of the JSON-RPC interfaces of specific blockchain nodes and providers (not consulted; check the one you use for method names, error codes and limits).

The URLs and titles above are the standard locations; I did not open them during this session (the sandbox cannot reach the documentation or standards sites), so if a link has moved, search for the title. **Unchecked or from memory:** the text of the JSON-RPC 2.0 specification (the crate implements and tests the rules I recall, including the reserved error codes and the batch rules, but I did not compare them with the specification); the claims that real chains and providers use hex or decimal strings for large integers, lag behind, prune history, rate-limit, use the `-32000..-32099` range for operational errors, drop or replace pending transactions, reject duplicate transactions as "already known" and allow replacement with the same nonce; that some chains have documented practical reorg depths; the "full jitter" analysis in the AWS blog post; the `serde_json` recursion limit; and everything about the finality models of specific networks. **Environment caveats:** timings are single-machine micro-measurements (medians of five runs, three separate runs of one example) on a shared virtual machine with two cores; there are no real network calls in this lesson, so no latency or reliability figure applies to any real provider. Measurements and outcomes marked ✓ were observed with `rustc 1.97.0`, `serde 1.0.229`, `serde_json 1.0.154` and `sha2 0.11.0`. The code blocks in exercises 3, 4 and 5 and in the debugging challenge's two versions were compiled and run in a scratch crate (✓ 14 tests, repeated three times without a failure; the buggy version intentionally contains the faults described); the trace of exercise 6 was produced by a test; the mutation results are those of the table above (a test process that aborted, failed an assertion or did not finish within 150 seconds counted as detecting the mutant). **No part of this lesson makes any system secure.**
