---
title: "Lesson 50 — Networking and deterministic execution"
---

# Lesson 50 — Networking and deterministic execution

**Module 15 · Cryptography and distributed systems (third lesson)** · Status: generated · Edition used in examples: **2024** · External dependencies: `sha2 0.11` only (the exercises and the crate use nothing else; 25 packages are in the `Cargo.lock` I copied from Lesson 49, the practical crate only needs `sha2`'s tree)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior or general knowledge I did not run.

**What this lesson is, and is not.** Two questions that every peer-to-peer or replicated system must answer: *how do two machines exchange messages over an unreliable byte stream without trusting each other?* and *how do many machines, running the same code on the same input, end up with exactly the same state?* The crate `netlab` answers both at teaching scale: **length-prefixed framing**, a **canonical binary protocol**, a **sans-IO peer session** (handshake, ping/pong, rate limiting, misbehavior scoring, timeouts, all with injected time), a **deterministic state machine** with canonical snapshots and a **metered VM**, a **seeded network simulator** for gossip, and real **loopback TCP** tests. It is teaching code: **not audited, not a node, not for production**. Not covered: TLS or any encryption of the stream (Lesson 48's primitives are the starting point, but I built no secure channel), authentication of peers, async runtimes, peer discovery, NAT traversal, DHTs, bandwidth management, real consensus messages, and anything about specific live networks. The simulator is **not a model of the real Internet**, and loopback timings are not wide-area timings. **Nothing here makes a system secure.** Rust removes memory-safety bugs; the failures in this lesson (a 4 GiB allocation requested by a stranger, a half-read message, two nodes hashing the same map into different bytes) are logic and design errors that compile cleanly.

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation and standards sites were not, so statements about RFCs, protocols and specific systems are labeled "from memory". Loopback TCP works in the sandbox, so the socket tests are real. The expected values of the random-number generator, the state hashes and the simulator reports were computed by **separate Python implementations** (✓). Timings are single-machine micro-measurements.

Builds on [Lesson 49](49_merkle_transactions_consensus.md) (agreement, trust assumptions, the determinism requirement), [Lesson 48](48_crypto_primitives.md) (hashes, domain separation, replay), [Lesson 47](47_deployment_wasm.md) (deterministic execution and fuel in Wasm), [Lesson 46](46_databases_config_auth_jobs.md) (atomic transactions), [Lesson 45](45_http_apis.md) (limits and timeouts), [Lesson 44](44_cli_serialization.md) (binary formats), [Lesson 43](43_observability_ci_supply_chain.md), [Lesson 41](41_benchmarking_profiling.md), [Lesson 40](40_property_testing_fuzzing.md), [Lesson 29](29_locks_deadlocks.md), [Lesson 27](27_threads_channels.md), [Lesson 26](26_error_design.md) and [Lesson 16](16_collections.md) (`HashMap` and `BTreeMap`). Lesson 51 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

A node on a peer-to-peer network reads bytes from strangers. A node that takes part in consensus must also compute **bit-identical** results from the same inputs as thousands of other nodes. The first is a *defensive* problem (hostile input, slow peers, resource exhaustion); the second is a *discipline* problem (no hidden sources of difference). The lesson is organized around experiments:

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| TCP delivers a stream, not messages | frames sent one byte at a time and two frames in one write are both read correctly (§2.2) |
| A length prefix is attacker-controlled input | a peer announcing 4 GiB is refused right after 4 bytes, before any allocation (§2.3) |
| One `read` call does not read a whole message | a single `read` returns a partial body padded with zeros (§11, debugging challenge) |
| A decoder should accept exactly what the encoder produces | for random inputs, `decode(b) = Ok(m)` implies `encode(m) = b`; flipping any single bit never yields a non-canonical message (§3.3) |
| Peers need budgets | a token bucket in integer milli-tokens refills at exactly the configured rate and survives odd clocks (§4.3) |
| `HashMap` order is not deterministic | fifty equal maps hash to more than one value in map order (§5.2) |
| Floating point depends on the order | `0.1 + 0.2 + 0.3` is `0.6000000000000001` but `0.3 + 0.2 + 0.1` is `0.6` (§5.3) |
| Infinite loops must stop the same way everywhere | a metered VM traps with `OutOfFuel` after exactly the same instruction on every run (§5.6) |
| Replicas that apply the same log agree | identical state hashes for every prefix of the log (§5.5) |
| Gossip without retransmission is unreliable under loss | with 30% loss, a ring of 10 reached everyone in 25 of 200 runs, a complete graph of 10 in 200 of 200 (§6.4) |

Total in the practical challenge: one library crate, 5 modules, **104 tests** (including 8 over real loopback sockets), a benchmark example, and **124 mutants** run against the test suite.

---

## 2. Framing: messages on a byte stream

### 2.1 The problem

TCP gives you a reliable, ordered **stream of bytes** (from memory, RFC 9293). It does **not** preserve the boundaries of your `write` calls: ten bytes written in two calls may arrive in one read or in ten, and two messages may arrive in a single read. Code that assumes "one `read` = one message" works on a developer's laptop and fails in production. The fix is **framing**: the sender prefixes each message with its length, and the receiver reassembles. This crate's frame is `4-byte big-endian length ‖ payload`.

```
  write(frame A)   write(frame B)
  [00 00 00 03 | a a a] [00 00 00 02 | b b]
        |  the network may deliver, in any split:  |
  read 1: [00 00]      read 2: [00 03 a a]   read 3: [a 00 00 00 02 b b]
```

Two library traits do the real work: `Read::read` returns *some* bytes (possibly fewer than requested, possibly interrupted by a signal as `ErrorKind::Interrupted`), `Read::read_exact` loops until the buffer is full or it fails ([Lesson 27](27_threads_channels.md) and the standard library documentation; the behavior of `read_exact` on `Interrupted` is from memory and I do not rely on it: the crate retries on its own).

### 2.2 The reader and the writer ✓

`read_frame` has three jobs: assemble the 4-byte header (which may itself arrive in pieces), decide *before reading the body* whether the announced length is acceptable, then read exactly that many bytes. It distinguishes **three endings**: a clean close between frames (`Closed`, normal), a close in the middle of a frame (`Truncated`, an error), and an announced length over the limit (`TooLarge`).

```rust
/// The limit is checked **before** any payload is read, and the buffer grows
/// only as bytes actually arrive, so a peer that announces a large frame and
/// then sends nothing costs almost no memory.
pub fn read_frame<R: Read>(r: &mut R, max: usize) -> Result<Vec<u8>, FrameError> {
    let mut header = [0u8; 4];
    let mut got = 0;
    while got < 4 {
        match r.read(&mut header[got..]) {
            Ok(0) if got == 0 => return Err(FrameError::Closed),
            Ok(0) => return Err(FrameError::Truncated),
            Ok(n) => got += n,
            Err(e) if e.kind() == io::ErrorKind::Interrupted => {}
            Err(e) => return Err(e.into()),
        }
    }
    let len = u32::from_be_bytes(header) as u64;
    if len > max as u64 {
        return Err(FrameError::TooLarge { len, max });
    }
    let mut buf = Vec::new();
    r.take(len).read_to_end(&mut buf)?;
    if (buf.len() as u64) < len {
        return Err(FrameError::Truncated);
    }
    Ok(buf)
}
```

✓ Tests (12 in `tests/frame.rs`): the format is fixed (`00 00 00 03 61 62 63`); frames round-trip and stay separate in one stream; reading works when the stream delivers **one byte at a time**; writing works when the sink accepts one byte at a time; `Interrupted` reads are retried; closing between frames is not an error but closing in the middle is; the limit is inclusive and checked before the body is read; a zero limit allows only empty frames; oversize payloads are refused without writing anything; only `WouldBlock` and `TimedOut` count as timeouts; and the reader **never consumes bytes past the frame** (a second frame can still be read). The same logic over **real sockets** (✓ `tests/tcp.rs`): a full exchange, two frames in one `write` still arrive as two messages, a message sent one byte at a time is reassembled, and a peer that vanishes mid-frame produces `Truncated`, not `Closed`.

### 2.3 The length is untrusted input ✓

The header is read before anything else is known about the peer. A naive reader does `vec![0; len]` and then reads; a stranger who sends `FF FF FF FF` makes the process ask for 4 GiB. Two defenses, both in the code above:

1. **Check the limit first.** A protocol has a maximum message size; compare `len` with it before allocating. ✓ A real socket test announces a 4 GiB frame and the reader returns `TooLarge { len: 4294967295, max: … }` immediately, after only four bytes arrived.
2. **Do not trust the limit as a pre-allocation size either.** The limit may be large (1 MiB here). If you allocate `max` bytes for every connection, a thousand idle connections cost a gigabyte. The reader instead grows the buffer as bytes actually arrive: `r.take(len).read_to_end(&mut buf)` stops at `len` bytes and allocates in proportion to what has been received, so memory tracks the data the peer really sent. A peer that announces 1 MiB and sends 10 bytes costs 10 bytes (plus the connection) until a timeout closes it.

This is the same rule as Lesson 48 §9.1 ("hostile numbers are checked before use") applied to memory. Clippy helps here too: ✓ `clippy::unused_io_amount` is a deny-by-default lint and it **rejected** the `stream.read(&mut buf)?;` calls of the buggy reader in §11 at compile-check time.

### 2.4 Timeouts and stalled peers ✓

A limit on size is not a limit on time. A peer can send the header and then nothing. With blocking sockets the defense is a **read timeout**: `TcpStream::set_read_timeout(Some(d))` makes `read` fail with `WouldBlock` or `TimedOut` (which one depends on the platform; from memory, Unix gives `WouldBlock`), and `FrameError::is_timeout()` classifies both. ✓ A silent loopback peer hits the timeout and the error is classified as one (the test uses a generous margin, not a tight one, so it does not flake on a loaded machine). What to do next is policy: close the connection, or keep it and try again if you are between frames. Mid-frame, a timeout normally means *close*, because the partial frame cannot be resumed unless you keep the buffered state yourself.

### 2.5 Cost ✓

Single-machine micro-measurements, release build, three runs (`examples/bench.rs`):

| operation | time |
|---|---|
| `read_frame` of a 100-byte payload from memory | about 59–60 ns |
| loopback TCP ping/pong round trip (9-byte payloads, `TCP_NODELAY`, 2000 per run) | 34.3–39.2 µs |

The round trip is dominated by the kernel and two thread wakeups, not by the code. `TCP_NODELAY` disables Nagle's algorithm (from memory, which delays small writes to coalesce them); for a request/response protocol made of small frames you usually want it. A WAN round trip would be thousands of times larger; do not read the loopback number as a network number.

---

## 3. A canonical binary protocol

### 3.1 The messages ✓

```
tag 1  Hello      version:u16  chain_id:u32  node_id:[u8;8]
tag 2  Ping       nonce:u64
tag 3  Pong       nonce:u64
tag 4  Tx         len:u32  bytes                      len ≤ 65,536
tag 5  GetBlocks  from_height:u64  max:u16            1 ≤ max ≤ 100
tag 6  Inv        count:u16  count × 32-byte hashes   count ≤ 1000
tag 7  Blocks     count:u16  count × (len:u32 bytes)  count ≤ 100, each ≤ 1 MiB
```

All integers are **big-endian and fixed width** ([Lesson 44](44_cli_serialization.md)); nothing depends on `usize`, struct layout or the host's byte order. A `Ping` is 9 bytes; a `Tx` of 300 bytes is 305 bytes (✓ measured). The frame layer (§2) carries these bytes; the two layers are independent so each can be tested alone.

### 3.2 Strict decoding ✓

Input from a stranger can be short, long, mislabeled or hostile. The decoder in `msg.rs` is deliberately strict:

- **Truncation** at every possible cut is detected (✓ `truncation_is_detected_at_every_cut` cuts every valid message at every length).
- **Trailing bytes** are an error (✓ `trailing_bytes_are_refused`): a message that decodes with extra bytes is not the message the encoder would have produced.
- **Unknown tags** are an error (✓), not "ignore and continue": a future version needs a version bump, not silent skipping.
- **Counts and lengths are checked against limits before anything is allocated** (✓ `counts_and_lengths_are_checked_before_anything_is_allocated`): an `Inv` that claims 1000 hashes but carries 3 is refused by comparing the *remaining byte count* with `count × 32` before building the vector; `Vec::with_capacity(count)` on an untrusted `u16`/`u32` is the classic allocation bug.
- **Semantic ranges** are part of decoding: `GetBlocks.max` must be in `1..=100` (✓ `block_counts_in_get_blocks_must_be_in_range`).
- **Limits are pinned by literal numbers** in a test (✓ `the_protocol_limits_are_pinned_to_literal_numbers` asserts `MAX_TX == 65_536`, and so on, and checks one byte past each limit). I added this test *because mutation testing showed that a constant changed from 65,536 to 65,537 went unnoticed* (§13): the tests used the constant to compute the boundary, so changing the constant moved the boundary and the test with it. A protocol limit is a compatibility promise; write it down as a number.

### 3.3 Canonical means one encoding per message ✓

An encoding is **canonical** if each message has exactly one valid byte string. Without it, two nodes can disagree about whether `bytes_a` and `bytes_b` are "the same message", hashes of messages stop being identifiers (Lesson 48 §4), and an attacker can malleate messages. This crate enforces it with two property tests (Lesson 40 style):

- ✓ `random_bytes_never_panic_and_accepted_input_is_canonical`: for a large number of pseudo-random inputs, `decode` never panics, and whenever `decode(b) == Ok(m)`, `encode(m) == b`;
- ✓ `flipping_any_single_bit_never_yields_a_non_canonical_message`: take every message, flip each bit in turn, and require that the result either fails to decode or re-encodes to exactly the flipped bytes.

`encode` itself panics when a *local* program violates a limit (`"transaction too long"`, and so on): that is a bug in our own code, not an input from the network, so it must be loud (Lesson 26). `decode` never panics; it returns `DecodeError` with the specific reason.

### 3.4 Cost ✓

| message | encode | decode |
|---|---|---|
| Ping (9 B) | 21 ns | 9 ns |
| Tx, 300 B (305 B on the wire) | 36–42 ns | 23–26 ns |
| Inv, 1000 hashes (32,003 B) | 1,247–1,402 ns | 1,460–1,539 ns |

Decoding a 32 KiB inventory in 1.5 µs is a copy and a few comparisons; encoding and decoding are never the bottleneck of a node (hashing and signature verification, Lessons 48 and 49, are far more expensive).

---

## 4. The peer session: sans-IO

### 4.1 Sans-IO ✓

The `Session` type does **no I/O and never reads a clock**. You feed it events and it returns **actions**:

```rust
pub enum Action {
    Send(Message),        // write this to the peer
    Deliver(Message),     // hand this to the application
    Disconnect(Reason),   // close the connection, here is why
}
// session.start(now_ms)                 -> Vec<Action>   (sends our Hello)
// session.on_message(msg, now_ms)       -> Vec<Action>
// session.on_decode_error(now_ms)       -> Vec<Action>
// session.tick(now_ms)                  -> Vec<Action>   (timers: handshake, ping)
```

Why this shape? (1) **Testability**: every behavior, including "the ping times out after 30 seconds", is tested by calling methods with chosen millisecond values, with no sleeping and no sockets (✓ 19 tests in `tests/peer.rs` run in milliseconds). (2) **Determinism**: time and the peer's behavior are inputs, so a test (or the simulator of §6) can replay any scenario exactly. (3) **Freedom**: the same state machine works with blocking threads, an async runtime or a simulator (from memory, this "sans-IO" style is used by several Rust and Python protocol libraries; I did not verify which). The socket test `the_session_type_is_usable_without_any_socket` makes the point, and `a_misbehaving_peer_is_dropped_by_the_session` runs the session over a real socket.

### 4.2 The state machine ✓

```
AwaitingHello --valid Hello--> Ready --(ban | timeout | protocol violation)--> Closed
      \--(anything else, bad version, wrong chain, our own id)--> Closed
```

Rules, in the order the code applies them: **rate limit first** (a message over budget is dropped and costs score; this protects every later rule); before the handshake only `Hello` is acceptable (✓ `anything_before_hello_ends_the_session`); a `Hello` must carry our protocol version, our chain id and a node id different from ours (✓ `a_bad_hello_ends_the_session_with_the_right_reason`, with three distinct `Reason`s: `UnsupportedVersion`, `WrongChain`, `SelfConnection`); a second `Hello` is a violation (✓); `Ping(n)` is answered with `Pong(n)` (✓); a `Pong` must match the **outstanding** ping nonce, otherwise it costs score (✓ `a_pong_with_the_wrong_nonce_does_not_count`); everything else is delivered to the application.

The chain id in the handshake is the same idea as in Lesson 49 §3.4: two networks that share a wire format must not accept each other's peers or messages by accident.

### 4.3 Rate limiting: a token bucket in integers ✓

A peer must not be able to make you work without bound. A **token bucket** has a capacity (the burst allowed) and a refill rate; each message costs one token. Floating point is a poor fit (Lesson 49 §7, and §5 below), so the bucket counts **milli-tokens** in `u64`:

```rust
pub struct RateLimiter {
    capacity_milli: u64,
    tokens_milli: u64,
    refill_per_sec: u64,
    last_ms: u64,
}

impl RateLimiter {
    /// A full bucket of `capacity` tokens refilling at `refill_per_sec`.
    pub fn new(capacity: u32, refill_per_sec: u32, now_ms: u64) -> Self {
        let cap = capacity as u64 * 1000;
        RateLimiter {
            capacity_milli: cap,
            tokens_milli: cap,
            refill_per_sec: refill_per_sec as u64,
            last_ms: now_ms,
        }
    }

    /// Takes one token if available. A clock that goes backwards adds nothing.
    pub fn allow(&mut self, now_ms: u64) -> bool {
        let elapsed = now_ms.saturating_sub(self.last_ms);
        if now_ms > self.last_ms {
            self.last_ms = now_ms;
        }
        // elapsed milliseconds * tokens per second = milli-tokens
        let added = elapsed.saturating_mul(self.refill_per_sec);
        self.tokens_milli = self
            .tokens_milli
            .saturating_add(added)
            .min(self.capacity_milli);
        if self.tokens_milli >= 1000 {
            self.tokens_milli -= 1000;
            true
        } else {
            false
        }
    }
}
```

✓ The tests pin: a burst of exactly `capacity` is allowed and the next message is not; refills arrive **exactly** at the configured rate (not one millisecond early); a patient peer is never limited; and the bucket survives odd clocks: time moving **backwards** adds nothing, a pause of years does not exceed the capacity (`min(capacity)`), and `u64::MAX` rates and timestamps saturate instead of wrapping (✓ `the_token_bucket_survives_odd_clocks_and_huge_numbers`). A monotonic clock (`Instant`) should feed the session in production; the injected `now_ms` only needs to be non-decreasing, and the code tolerates it when it is not.

### 4.4 Misbehavior scoring and timeouts ✓

Not every protocol violation deserves a ban; a flaky network produces garbage occasionally. The session keeps a **score** and bans (disconnects with `Misbehaving`) when it reaches the configured threshold. In the tests' configuration: rate-limited message = 20, undecodable input = 50, unsolicited `Pong` = 10; the ban is inclusive (✓ `the_ban_threshold_is_inclusive`: a score *equal to* the threshold bans). Two timers run in `tick`: the **handshake deadline** (a peer that connects and says nothing is dropped; ✓ `the_handshake_has_a_deadline`) and **liveness**: an idle connection is pinged after `ping_interval`, and must answer within `ping_timeout` or it is disconnected with `PingTimeout`; each new ping uses a fresh nonce (✓).

Scores are *policy*, and the numbers above are teaching values. Real nodes also decay scores over time, ban by address for a period, and have to worry about an attacker who burns through many addresses (from memory; not built here). Two honest limits: scoring does not stop a determined adversary with many identities, and a bad threshold either bans honest peers on a bad network or lets abusers stay. Measure with real traffic.

---

## 5. Deterministic execution

### 5.1 Why every node must compute the same bytes

In a replicated state machine ([Lesson 49](49_merkle_transactions_consensus.md) §5), many nodes apply the same ordered log of commands to the same initial state. They agree *only if* applying a command is a **pure function** of (state, command). One node that differs by one bit produces a different state hash, rejects blocks the others accept, and drops off the network or, worse, splits it. The sources of difference are the same everywhere:

| Source of non-determinism | What goes wrong | Deterministic alternative |
|---|---|---|
| `HashMap`/`HashSet` iteration order | each map instance has a random hasher; order differs between runs and machines | `BTreeMap`/`BTreeSet`, or sort the keys before hashing or serializing |
| Floating point | addition is not associative; library functions and compilers may differ by platform (from memory) | integers and fixed-point; `checked_*` |
| Integer overflow | debug builds panic, release builds wrap, other targets differ in width for `usize` | fixed-width integers, `checked_*` that returns an error |
| Byte order and layout | `to_ne_bytes`, `repr(Rust)` struct layout | explicit big-endian (or little-endian) fixed-width encoding |
| Clocks | each node reads a different time | time is an input (a block timestamp), never read inside validation |
| Randomness | each node draws different values | none in validation; a seed that is *part of the input* |
| Threads | results arrive in different orders | a single deterministic order, or a deterministic merge |
| Unbounded loops/recursion | one node runs out of memory or time, another does not | **metering**: a fuel budget counted in the same units everywhere (§5.6) |
| Errors | different messages/ordering of checks | one error type, one check order, compared by value |
| Dependencies and compilers | two library versions disagree on an edge case | pinned versions (`Cargo.lock`), the same code on every node |

(From memory, blockchain virtual machines such as the Wasm-based ones avoid floating point or define it strictly, and meter execution; see [Lesson 47](47_deployment_wasm.md) for what Wasm itself does and does not guarantee. I did not verify the details of any specific system.)

### 5.2 `HashMap` order ✓

`std::collections::HashMap` seeds each map with a random hasher on purpose, to resist hash-flooding attacks ([Lesson 16](16_collections.md); from memory). The price is that iteration order is unspecified and differs from one map to another, *even in the same process*. ✓ The crate's `hash_in_map_order` hashes entries in iteration order, and the test `hashing_a_hashmap_in_iteration_order_is_not_deterministic` builds equal maps and obtains different hashes. The same experiment in the scratch crate: 50 maps with identical contents gave more than one distinct hash. Two fixes, both in the crate: **sort the keys** before hashing (`hash_in_key_order`), or store the data in a `BTreeMap` from the start (`Machine` does). The ordered hash is independent of insertion order but not of content (✓ `the_ordered_hash_ignores_insertion_order_but_not_content`).

### 5.3 Floating point ✓

Floating-point addition is not associative, so the *order* of a sum changes the result:

```rust
assert_eq!(sum_f64(&[0.1, 0.2, 0.3]), 0.6000000000000001);
assert_eq!(sum_f64(&[0.3, 0.2, 0.1]), 0.6);   // same numbers, other order
```

✓ (`floating_point_addition_depends_on_the_order`.) On one machine and one binary the result is repeatable; the danger in a distributed system is that *different* nodes may sum in different orders (a parallel reduction, a different iteration order, a different compiler or library) or evaluate functions such as `sin` differently (from memory: IEEE 754 requires correct rounding only for basic operations, not all math functions). Rule: **no floating point in validity rules**; use integers (fixed-point: amounts in the smallest unit) and `checked_add`, which also reports overflow (✓ `integer_sums_are_exact_in_any_order_and_report_overflow`).

### 5.4 The state machine ✓

`Machine` holds `BTreeMap<u64, u64>` balances and a height. Commands: `Credit { acct, amount }` and `Transfer { from, to, amount }`. Every rule is explicit and checked: zero amounts are refused, balances cannot overflow (`checked_add`) and cannot go negative (`checked_sub`), a transfer to yourself still needs the funds and changes nothing else (✓ `a_transfer_to_yourself_changes_nothing_but_still_needs_the_funds`), and a **block** of commands is applied **atomically**: the machine clones its state, applies each command to the copy, and replaces the real state only if every command succeeded, returning the index of the first failing command otherwise (✓ `a_block_is_all_or_nothing`). Cloning is the simplest way to get atomicity; a real node would use an undo log or a copy-on-write structure. **Order matters** for commands that do not commute (✓ `order_matters_for_commands_that_do_not_commute`: transfer-then-credit differs from credit-then-transfer), which is exactly why the log order is the thing consensus provides.

### 5.5 Canonical snapshots and state hashes ✓

To compare nodes without sending the state, hash it. The hash must be defined by a **canonical byte string**:

```
snapshot = u64(len("lab/state/v1")) ‖ "lab/state/v1" ‖ height:u64 ‖ count:u64 ‖ (account:u64 ‖ balance:u64) × count   (ascending accounts)
```

All fields are fixed-width big-endian, accounts come in ascending order (a `BTreeMap` gives that for free), and the domain label and a length prefix keep this hash from colliding with other uses of SHA-256 (Lesson 48 §4). ✓ Known answers computed by an independent Python implementation: the empty machine hashes to `f03a3748…49b6` and the machine `{1: 70, 2: 80}` at height 1 (a 68-byte snapshot) to `1f973434…f277` (`state_hashes_match_independent_values`). ✓ `the_hash_covers_height_and_every_balance`, and `replicas_that_apply_the_same_log_agree_on_every_hash`: two machines applying the same log have the same hash after every block.

`restore` is the inverse and is as strict as a network decoder, since a snapshot may come from a peer: it checks the domain, uses `count.checked_mul(16)` so a hostile count cannot overflow, checks that the body length equals exactly what the count promises, and requires **strictly ascending keys** and **no zero balances**, i.e. it refuses everything that `snapshot` never produces (✓ `restore_refuses_everything_that_snapshot_never_produces`, `snapshots_round_trip_exactly`). The canonical-form rule of §3.3 applies to state, too: a state with two possible encodings would have two hashes.

Cost ✓: applying 10,000 credits takes about 1.0 ms (9.4–10.1 million commands per second, a map insertion each); hashing 10,000 accounts (160,036 bytes) takes about 109–115 µs. Recomputing the whole hash after every block does not scale; real systems keep an authenticated tree so that an update touches `O(log n)` nodes (Lesson 49 §2, and from memory, Merkle–Patricia tries); the flat hash here is a teaching simplification.

### 5.6 A metered virtual machine ✓

Smart-contract platforms must run code from strangers on every node. Two requirements: it must be **deterministic** and it must **terminate**. The second cannot be decided in advance (the halting problem), so execution is **metered**: each instruction costs *fuel*, the caller supplies a budget, and running out is a defined, deterministic failure.

```rust
pub enum Op { Push(i64), Add, Sub, Mul, Div, Dup, Swap, Pop, Jmp(usize), Jnz(usize), Halt }
// cost: 1 for every instruction, 3 for Mul, 5 for Div
// run(program, fuel) -> Result<Outcome { top, fuel_used, steps }, Trap>
// Trap: OutOfFuel, StackUnderflow, StackOverflow (depth 256), Overflow, DivideByZero, BadJump(target)
```

Design points, each pinned by a test: **fuel is checked before the instruction executes**, so a program with fuel `n` runs exactly the instructions whose cumulative cost fits (✓ `fuel_is_exact`); an endless loop `[Jmp(0)]` stops with `OutOfFuel` for any budget (✓ `an_infinite_loop_is_stopped_by_the_fuel_limit`); a counting loop costs exactly four instructions per iteration (✓); arithmetic overflow is a **trap**, never a wrap, and division truncates toward zero and traps on zero and on `i64::MIN / -1` (✓ two tests); the stack depth is bounded; a jump to an index past the end of the program is a `BadJump`, and a jump *to* the end halts normally (✓ the test I added after mutation testing); running twice gives identical results (✓). Expensive operations cost more (`Mul` 3, `Div` 5): fuel prices are a protocol constant, and changing one is a consensus change (✓ `division_costs_five_and_multiplication_three`, also added after mutation testing).

Measured ✓ (release build, three runs): 4,000,002 instructions in 7.04–7.54 ms, about 531–568 million instructions per second. [Lesson 47](47_deployment_wasm.md) discussed the same fuel idea inside Wasm runtimes; the point here is that *you* can write the rule in 100 lines and test it exhaustively.

**What metering does not give you:** it bounds time in *instructions*, not seconds; a cheap-looking operation that is slow in the host (a huge memory copy) must be priced to its real cost or it becomes a denial-of-service lever; and it does not bound memory unless you meter memory too (the stack limit here does, nothing else is allocated by programs).

---

## 6. Simulating a network

### 6.1 Why simulate

Testing a protocol on real networks is slow, flaky and unrepeatable. A **simulator** turns the network into a function: given a topology, loss, duplication and delay settings, and a seed, it returns *exactly* the same history every time. Bugs found at seed 4,271 can be replayed at seed 4,271. This is the idea behind "deterministic simulation testing" (from memory, used by some database and distributed-systems teams; I did not verify which).

### 6.2 A seeded generator ✓

Randomness inside a simulation is a *pure function of a seed*. `Rng` is xorshift64* (shifts 12, 25, 27; multiplier `0x2545F4914F6CDD1D`; seed 0 is replaced by a fixed nonzero constant). It is **not** cryptographically secure and must never be used for keys or nonces (Lesson 48 §6). ✓ Its outputs were checked against an independent Python implementation (`the_generator_matches_an_independent_implementation`; for seed 1 the first output is `5180492295206395165`). `below(n)` uses **rejection sampling** (draws above the largest multiple of `n` are redrawn) so the result has no modulo bias; ✓ it stays in range and is roughly uniform.

### 6.3 The flooding simulation ✓

`flood(network, link, seed, origin)` models gossip: the origin sends a message to all its neighbors; each node that receives it **for the first time** forwards it to every neighbor except the one it came from. Each send may be lost (`drop_per_mille`), delayed by `1..=max_delay` ticks, and duplicated (`dup_per_mille`, with its own delay). Events are processed from a priority queue ordered by `(time, sequence number, from, to)`, so ties are broken deterministically. The random draws per send happen in a **fixed order** (loss, delay, duplicate decision, duplicate delay), even when a draw is not needed; otherwise the stream would shift and a different seed-to-history mapping would result when a parameter changed. The `Report` records first-receipt time per node, messages sent, dropped, duplicated, events processed and the time of the last event.

✓ The simulator is a pure function (`a_run_is_a_pure_function_of_its_inputs`) and agrees with an **independent Python simulation**: for a ring of 6 with 10% loss and 10% duplication, max delay 5, seed 7, origin 0, the first-receipt times are `[0, 4, 5, 8, 9, 12]`, 7 sends, 1 drop, 0 duplicates, 6 events, finished at tick 16; for a complete graph of 5 with 30% loss, max delay 4, seed 11, origin 2: `[5, 1, 0, 2, 4]`, 16 sends, 7 drops, 9 events, finished at 8; and a duplication case (ring of 6, 60% duplication, max delay 9, seed 3): `[0, 2, 4, 5, 9, 1]`, 7 sends, 4 duplicates, 11 events, finished at tick 12 (`duplicate_copies_get_their_own_delay`, a test added after mutation testing).

### 6.4 What gossip costs, and what it does not guarantee ✓

With no loss, the number of sends follows from the forwarding rule (✓ `message_counts_follow_from_the_forwarding_rule`): a **ring** of `n` nodes costs `n + 1` sends (the origin sends 2; every other node forwards once, except that the last two nodes each send a copy to each other), and a **complete graph** of `n` nodes costs `(n − 1)²` (the origin sends `n − 1`; each of the other `n − 1` forwards to `n − 2`). ✓ Measured by the benchmark: ring of 1000 → 1001 sends, finished at tick 2748; complete graph of 100 → 9801 sends, finished at tick 13. Redundancy is the price of robustness: the complete graph is slow in messages (quadratic) and fast in time; the ring is cheap and slow.

Gossip **without retransmission** is not reliable under loss. ✓ `gossip_without_retransmission_is_not_reliable_under_loss`: with 30% loss per send over 200 seeds, a ring of 10 nodes delivered to everyone in only **25 of 200** runs (one lost message cuts the ring in a place with no alternative path), while a complete graph of 10 delivered in **200 of 200** (many independent paths). In a partitioned network only the origin's component is reached (✓). Real protocols add acknowledgements, announcements ("I have X") followed by requests, periodic anti-entropy exchanges and random peer selection (from memory); the `Inv` and `GetBlocks` messages of §3 are the announce-then-request pattern, but I did not simulate it.

Cost ✓: the simulation of a 100-node complete graph (9,801 events) takes 0.82–1.01 ms; a 1000-node ring takes about 0.02 ms.

### 6.5 Honest limits of the simulator

It models loss, duplication and delay with independent uniform draws. Real networks have correlated loss, bursts, asymmetric links, reordering by path, congestion, and adversaries who choose *which* messages to delay. The simulator is a **test tool for logic**, not a prediction of behavior. It also simulates one message and one origin; there are no node failures, no clocks that drift, and no byzantine nodes.

---

## 7. Rust notes for network and consensus code

- **Never panic on input from the network** ([Lesson 26](26_error_design.md)): decode returns `Result`, with an error enum that says which rule failed. Panic is for *our own* invariant violations (`encode` of an oversized message).
- **`#![deny(missing_docs)]`** in a library that others implement against forces every public item to say what it promises; for a protocol crate that is part of the specification.
- **Errors implement `Display` and keep their source** (`FrameError::Io(io::Error)`), and the tests check the messages (`error_messages_are_readable`) because logs are how operators find out why a peer was dropped.
- **Use the type system for states** where it helps ([Lesson 15](15_newtype_typestate.md)): the session's `State` and `Reason` are enums, so a `match` that forgets a state does not compile. A typestate design (`Session<AwaitingHello>` versus `Session<Ready>`) is possible, but each message may move the session from one state to another, and with a runtime socket loop that becomes awkward; I chose an enum and tests.
- **Arithmetic on peer-controlled numbers:** `checked_*`/`saturating_*` everywhere (the bucket uses `saturating_mul`; the snapshot decoder uses `checked_mul`), never `as` casts that truncate silently; `u32::try_from(len)` is used when writing a frame ([Lesson 4](04_variables_and_types.md)).
- **Threads and sockets** ([Lesson 27](27_threads_channels.md), [Lesson 29](29_locks_deadlocks.md)): the TCP tests use one thread per side and generous timeouts. A blocking read with no timeout can hang a thread forever; a lock held across a blocking read can hang *every* thread that needs the lock. Keep I/O outside critical sections.
- **No `unsafe` is used** anywhere in the crate; Rust's memory safety is not the issue in this lesson.

---

## 8. Common mistakes

1. Assuming one `read` returns one message, or that `read` fills the buffer (§2.1, §11).
2. Allocating from a length the peer announced (`Vec::with_capacity(len)`) before checking it (§2.3).
3. Checking a limit after reading the body instead of before (§2.3).
4. No read timeout, so one silent peer holds a thread forever (§2.4).
5. Treating a close in the middle of a frame as a normal close (§2.2).
6. Accepting trailing bytes, unknown tags or non-canonical encodings (§3).
7. Using the platform's native integer width or byte order on the wire (§3.1).
8. Calling the clock inside protocol logic, which makes it untestable (§4.1).
9. Floating point for budgets, rates or money (§4.3, §5.3).
10. Hashing or serializing a `HashMap` in iteration order (§5.2).
11. Letting a failed command leave half an update behind (§5.4).
12. A non-canonical state encoding, or one without a domain label (§5.5).
13. Running untrusted code with no fuel limit, or with fuel prices that do not match real cost (§5.6).
14. Believing a gossip protocol is reliable because it worked once on a clean network (§6.4).
15. Using a simulation's output as a statement about the real Internet (§6.5).
16. Limits defined by constants that the tests also use to compute boundaries (§3.2).

---

## 9. Security notes and honest limits

- `netlab` is teaching code. **There is no encryption and no authentication of peers**: anyone on the path can read, modify or inject frames. A real node runs the protocol inside an authenticated, encrypted channel (TLS or a Noise-based handshake; from memory) and binds the peer's identity to a key (Lesson 48).
- The ban score and rate limit slow down one abusive connection. They do not stop an adversary who opens thousands of connections from thousands of addresses; connection limits, per-address limits and cost to join are separate defenses not built here.
- Loopback timings and the simulator say nothing about real network behavior (§2.5, §6.5).
- The state hash is a flat SHA-256 over a snapshot: it is deterministic, but it is not a Merkle structure and cannot give membership proofs.
- The VM is a toy. A real contract platform needs memory metering, host-call pricing, a verifier for programs, versioning of fuel costs, and a security review that this lesson did not do.
- A deterministic program is not a correct program: if every node runs the same bug, every node agrees on the wrong answer (Lesson 49 §6).
- Nothing here addresses privacy: addresses, timing and message sizes leak information.

---

## 10. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Why does a TCP receiver need framing? List three things that can happen to two consecutive `write` calls of 5 bytes each. Why is the length checked before the body is read?
2. **(Warm-up)** Give four sources of non-determinism in Rust code that a blockchain node must avoid, and a replacement for each.
3. **(Easy)** Write `read_frame<R: Read>(r, max) -> io::Result<Option<Vec<u8>>>` for a 4-byte big-endian length prefix: it returns `Ok(None)` on a clean close before a frame, an error on truncation, and refuses lengths over `max` without allocating.
4. **(Medium)** Write `encode`/`decode` for `enum Msg { Ping(u64), Data(Vec<u8>) }` with tags 1 and 2, `Data` having a `u16` length and a 1024-byte limit. `decode` must reject truncation, unknown tags, over-long data and trailing bytes, and `decode(b) == Ok(m)` must imply `encode(m) == b`.
5. **(Medium)** Write a token bucket with integer arithmetic and injected time: `Bucket::new(capacity, refill_per_sec, now_ms)` and `allow(now_ms) -> bool`. It must never exceed its capacity, ignore a clock that moves backwards and not overflow.
6. **(Medium-hard)** Flooding gossip: derive the number of messages sent in a lossless **ring** of `n` nodes and in a **complete graph** of `n` nodes, when each node forwards a first-received message to all neighbors except the sender.
7. **(Hard)** For the program `[Push(5), Push(7), Mul, Push(2), Div, Halt]`, compute the fuel used and the result. Then explain why `[Jmp(0)]` terminates for *every* fuel budget and what this property gives a blockchain.
8. **(Hard)** Design question: a node gets a message announcing 1 MiB from a peer that then sends 10 bytes per second. List the resources at risk, the defenses (with the number you would choose and why), and what you would log.

<details markdown="1">
<summary>Answers</summary>

1. TCP is a stream: the two writes may arrive as **one read of 10 bytes**, as **two reads of 5**, or as pieces that split a write (for example 3 + 7 bytes); the receiver cannot tell message boundaries from the stream alone. A length prefix lets it reassemble. The length is checked **before** the body is read (and before any allocation) because it comes from the peer: an unchecked 4 GiB announcement would make the node allocate or wait for 4 GiB (✓ refused after four bytes on a real socket).
2. (a) `HashMap` iteration order → `BTreeMap` or sorted keys. (b) Floating point → integers and fixed-point with `checked_*`. (c) Reading the clock or random numbers inside validation → pass time (and a seed) in as part of the input. (d) `usize` width / native byte order / struct layout in a format → fixed-width integers and explicit big-endian bytes. (Others: threads finishing in different orders, overflow behavior that differs between debug and release, unbounded loops.)
3. A possible answer (✓ compiled and run in a scratch crate with three tests, including a reader that returns one byte at a time):

   ```rust
   use std::io::{self, Read};

   /// Read one frame: 4-byte big-endian length, then that many bytes.
   /// `Ok(None)` means the peer closed cleanly before a new frame began.
   pub fn read_frame<R: Read>(r: &mut R, max: usize) -> io::Result<Option<Vec<u8>>> {
       let mut head = [0u8; 4];
       let mut got = 0;
       while got < 4 {
           match r.read(&mut head[got..]) {
               Ok(0) if got == 0 => return Ok(None),
               Ok(0) => return Err(io::ErrorKind::UnexpectedEof.into()),
               Ok(n) => got += n,
               Err(e) if e.kind() == io::ErrorKind::Interrupted => {}
               Err(e) => return Err(e),
           }
       }
       let len = u32::from_be_bytes(head) as usize;
       if len > max {
           return Err(io::Error::new(io::ErrorKind::InvalidData, "frame too large"));
       }
       let mut body = Vec::new();
       r.take(len as u64).read_to_end(&mut body)?;
       if body.len() != len {
           return Err(io::ErrorKind::UnexpectedEof.into());
       }
       Ok(Some(body))
   }
   ```

   The header loop is needed because `read` may return fewer than four bytes; `Ok(0)` with nothing read yet is a clean close, with something read it is a truncation. `r.take(len).read_to_end(...)` grows the buffer with the bytes that really arrive, so a peer that announces `max` and sends nothing costs nothing. (`Interrupted` is retried; `WouldBlock`/`TimedOut` are returned to the caller.)
4. A possible answer (✓ compiled and run; the test also checks `encode(decode(b)) == b` over 262,144 short inputs, each cut at every length):

   ```rust
   #[derive(Debug, PartialEq, Eq, Clone)]
   pub enum Msg {
       Ping(u64),
       Data(Vec<u8>),
   }

   #[derive(Debug, PartialEq, Eq)]
   pub enum Error {
       Truncated,
       UnknownTag(u8),
       TooLong,
       Trailing,
   }

   const MAX_DATA: usize = 1024;

   pub fn encode(m: &Msg) -> Vec<u8> {
       match m {
           Msg::Ping(n) => {
               let mut v = vec![1];
               v.extend_from_slice(&n.to_be_bytes());
               v
           }
           Msg::Data(d) => {
               assert!(d.len() <= MAX_DATA);
               let mut v = vec![2];
               v.extend_from_slice(&(d.len() as u16).to_be_bytes());
               v.extend_from_slice(d);
               v
           }
       }
   }

   pub fn decode(b: &[u8]) -> Result<Msg, Error> {
       let (&tag, rest) = b.split_first().ok_or(Error::Truncated)?;
       let (msg, used) = match tag {
           1 => {
               let n: [u8; 8] = rest.get(..8).ok_or(Error::Truncated)?.try_into().unwrap();
               (Msg::Ping(u64::from_be_bytes(n)), 8)
           }
           2 => {
               let l: [u8; 2] = rest.get(..2).ok_or(Error::Truncated)?.try_into().unwrap();
               let len = u16::from_be_bytes(l) as usize;
               if len > MAX_DATA {
                   return Err(Error::TooLong);
               }
               let d = rest.get(2..2 + len).ok_or(Error::Truncated)?;
               (Msg::Data(d.to_vec()), 2 + len)
           }
           t => return Err(Error::UnknownTag(t)),
       };
       if rest.len() != used {
           return Err(Error::Trailing);
       }
       Ok(msg)
   }
   ```

   The line `if rest.len() != used` is what rejects trailing bytes; without it, `[1, 0,0,0,0,0,0,0,1, 0]` would decode to `Ping(1)` and re-encode to a shorter string, breaking canonicality. The length is compared with `MAX_DATA` **before** slicing, and `rest.get(2..2 + len)` cannot panic.
5. A possible answer (✓ compiled and run):

   ```rust
   /// Token bucket in integer milli-tokens; time is passed in.
   pub struct Bucket {
       capacity_milli: u64,
       refill_per_sec: u64,
       milli: u64,
       last_ms: u64,
   }

   impl Bucket {
       pub fn new(capacity: u64, refill_per_sec: u64, now_ms: u64) -> Self {
           let capacity_milli = capacity.saturating_mul(1000);
           Bucket { capacity_milli, refill_per_sec, milli: capacity_milli, last_ms: now_ms }
       }

       pub fn allow(&mut self, now_ms: u64) -> bool {
           if now_ms > self.last_ms {
               // refill_per_sec tokens per 1000 ms is exactly refill_per_sec milli-tokens per ms
               let add = (now_ms - self.last_ms).saturating_mul(self.refill_per_sec);
               self.milli = self.milli.saturating_add(add).min(self.capacity_milli);
               self.last_ms = now_ms;
           }
           if self.milli >= 1000 {
               self.milli -= 1000;
               true
           } else {
               false
           }
       }
   }
   ```

   `refill_per_sec` tokens per 1000 ms is exactly `refill_per_sec` **milli**-tokens per millisecond, so no division and no rounding. Time only moves forward (`now_ms > last_ms`), the multiplication and the addition saturate, and `min(capacity)` caps the stored amount.
6. Ring: the origin sends 2 messages. Each of the other `n − 1` nodes forwards a first receipt to its one other neighbor, which makes `n − 1` more sends, for a total of `n + 1` (the last two nodes send to each other, and those copies are ignored as duplicates). Complete graph: the origin sends `n − 1`; each of the `n − 1` receivers forwards to the `n − 2` neighbors that are neither itself nor the sender: `(n − 1) + (n − 1)(n − 2) = (n − 1)²`. ✓ Both formulas were verified by a test for `n = 4, 10, 100` (`ring(1000)` gave 1001 and `complete(100)` gave 9801).
7. Costs: `Push` 1, `Push` 1, `Mul` 3, `Push` 1, `Div` 5, `Halt` 1 → ✓ **12** fuel and 6 steps; the result is `5 × 7 / 2 = 35 / 2 = 17` (division truncates; ✓ `top = Some(17)`). `[Jmp(0)]` costs 1 per iteration, so it spends any finite budget and stops with `OutOfFuel` (✓ for budgets 1, 10 and 1000). For a blockchain this means a hostile contract cannot stall validators: every validator stops at the *same* instruction and charges the same fee, so they still agree on the outcome (a failed transaction).
8. At risk: a connection slot and a thread (or task), memory (the partial frame), and the node's patience. Defenses: (1) frame limit checked before reading, so the announcement itself is capped (1 MiB here, chosen as a few times the largest legitimate message); (2) incremental buffer growth, so only 10 B/s of memory is used; (3) a **read deadline for the whole frame**, not per `read` (for example 30 s from the header to the last byte, so a trickle does not reset the clock), then disconnect; (4) a per-peer budget of connections and of in-flight bytes; (5) score the peer as misbehaving, and ban after repetition. Log: peer address, the announced length, bytes received and time, and the reason for the disconnect; do not log the payload. In this crate the per-`read` timeout of §2.4 does *not* stop a slow trickle by itself; a whole-frame deadline is needed on top (not built in `netlab`).

</details>

---

## 11. Debugging challenge

A node reads framed messages and keeps balances in a map; it also computes a state hash to compare with its peers. Three reports arrive: (1) under load, some messages arrive with zeros at the end or with the wrong content; (2) a peer that sends the header `FF FF FF FF` makes the process use gigabytes of memory, and a peer that connects and sends nothing blocks a thread forever; (3) two nodes with identical balances publish different state hashes, and a restart changes the hash too. Explain each cause and fix the code.

```rust
use sha2::{Digest, Sha256};
use std::collections::HashMap;
use std::io::Read;
use std::net::TcpStream;

/// Reads one length-prefixed message from the peer.
pub fn read_message(stream: &mut TcpStream) -> std::io::Result<Vec<u8>> {
    let mut head = [0u8; 4];
    stream.read(&mut head)?;
    let len = u32::from_be_bytes(head) as usize;
    let mut body = vec![0u8; len];
    stream.read(&mut body)?;
    Ok(body)
}

/// Hash of all balances, used to compare nodes.
pub fn state_hash(balances: &HashMap<u64, u64>) -> [u8; 32] {
    let mut h = Sha256::new();
    for (k, v) in balances {
        h.update(k.to_be_bytes());
        h.update(v.to_be_bytes());
    }
    h.finalize().into()
}
```

<details markdown="1">
<summary>Solution</summary>

1. **`read` is not `read_exact`.** `stream.read(&mut body)?` returns however many bytes are available *now*, and the code ignores the count; the rest of the buffer stays zero (✓ `buggy_read_loses_data_when_the_body_arrives_in_pieces`: the body is sent in two pieces, the buggy reader returns six bytes of which three are zeros). The header read has the same bug: it may return fewer than four bytes. Clippy flags both calls with the deny-by-default lint `unused_io_amount` (✓ the scratch crate does not compile under `cargo clippy` until they are fixed).
2. **Unbounded, untimed reads.** `vec![0u8; len]` allocates whatever the peer announces (up to 4 GiB), and there is no read timeout, so a silent peer blocks the thread. Fix: a maximum message size checked **before** allocating, growth with the bytes that really arrive (`take(len).read_to_end`), and `set_read_timeout` (✓ `fixed_read_refuses_a_huge_announcement_immediately` returns `InvalidData` at once).
3. **`HashMap` iteration order.** Each map has a random hasher, so identical contents iterate in different orders and give different hashes (✓ `buggy_state_hash_depends_on_the_map_instance`: 50 equal maps gave several different hashes), and a restart builds a new map with a new order. Fix: a `BTreeMap` (or sorted keys), fixed-width big-endian fields, and a leading count so that the encoding is unambiguous (✓ `fixed_state_hash_is_stable`: the same entries inserted in the opposite order hash equal, and a different content hashes differently).

Fixed version (✓ compiled; four tests, including reassembly of a body sent in two pieces 100 ms apart):

```rust
use sha2::{Digest, Sha256};
use std::collections::BTreeMap;
use std::io::{self, Read};
use std::net::TcpStream;
use std::time::Duration;

const MAX_MESSAGE: usize = 1 << 20;

/// Reads one length-prefixed message; bounded in size and in time.
pub fn read_message(stream: &mut TcpStream) -> io::Result<Vec<u8>> {
    stream.set_read_timeout(Some(Duration::from_secs(10)))?;
    let mut head = [0u8; 4];
    stream.read_exact(&mut head)?;
    let len = u32::from_be_bytes(head) as usize;
    if len > MAX_MESSAGE {
        return Err(io::Error::new(io::ErrorKind::InvalidData, "message too large"));
    }
    let mut body = Vec::new();
    stream.take(len as u64).read_to_end(&mut body)?;
    if body.len() != len {
        return Err(io::ErrorKind::UnexpectedEof.into());
    }
    Ok(body)
}

/// Hash of all balances: sorted keys, fixed-width big-endian fields, a count up front.
pub fn state_hash(balances: &BTreeMap<u64, u64>) -> [u8; 32] {
    let mut h = Sha256::new();
    h.update((balances.len() as u64).to_be_bytes());
    for (k, v) in balances {
        h.update(k.to_be_bytes());
        h.update(v.to_be_bytes());
    }
    h.finalize().into()
}
```

Two notes: the per-`read` timeout still lets a peer who sends one byte every nine seconds hold a thread for hours (§10, exercise 8); a whole-frame deadline or a non-blocking design is the complete answer. And the fixed hash is still a flat hash with a domain-free encoding; §5.5's snapshot adds a domain label.

</details>

---

## 12. Interview questions

1. *Why does TCP code need message framing? What are the usual framing schemes?*
2. *What is the difference between `Read::read` and `Read::read_exact`? What can go wrong with a single `read`?*
3. *A peer sends a length prefix of 4 GiB. What should the node do, and in what order?*
4. *What does it mean for an encoding to be canonical? Why does it matter for hashing and consensus?*
5. *What is a "sans-IO" protocol implementation? What are its advantages for testing?*
6. *How does a token bucket work? Why implement it with integers and injected time?*
7. *Why does `HashMap` iteration order differ between runs? How do you hash a map deterministically?*
8. *Why is floating point a problem in consensus-critical code?*
9. *What is a metered VM? Why is a gas or fuel limit necessary, and why must it be deterministic?*
10. *Why must a block of transactions be applied atomically?*
11. *How would you test a gossip protocol without a real network? What does determinism buy you?*
12. *What does a read timeout protect against, and what does it not protect against?*
13. *What guarantees does a gossip protocol without retransmission give under message loss?*

<details markdown="1">
<summary>Answers</summary>

1. TCP is a byte stream with no message boundaries; two writes may be read together or split. Common schemes: a length prefix (this lesson), a delimiter (newline, with escaping), or fixed-size messages. Length prefixes are the usual choice for binary protocols.
2. `read` returns *some* number of bytes (0 means end of stream); `read_exact` loops until the buffer is full or an error (including unexpected end of stream) occurs. Ignoring the count of a single `read` yields partial or zero-padded data (✓ §11).
3. Read only the four-byte header, compare with the protocol maximum **before** allocating or reading the body, refuse and disconnect (and likely score the peer). ✓ Refused after four bytes on a real socket.
4. One valid byte string per value. Otherwise two encodings of the same message have different hashes and signatures, nodes can disagree about equality, and attackers can malleate. Enforce it by rejecting trailing bytes, unknown tags, out-of-order or duplicate entries, and by testing `decode(b) = Ok(m) ⇒ encode(m) = b` (§3.3).
5. The protocol logic does no I/O and does not read a clock: it consumes events (bytes in, time) and returns actions (bytes out, disconnect). Tests call it with chosen inputs and timestamps, with no sleeping or sockets; the same code runs under threads, async or a simulator.
6. A bucket holds up to `capacity` tokens and gains tokens at a fixed rate; each message costs one; if there is none, the message is rejected. Integers avoid float rounding and platform differences, and an injected `now` makes it deterministic and testable; the code must handle time not advancing, going backwards, and huge values.
7. Rust's `HashMap` seeds each map with a random hasher (to resist hash-flooding), so order is unspecified. Hash in sorted key order or use a `BTreeMap`, with fixed-width big-endian fields.
8. Addition is not associative and library functions can differ across platforms or compilers, so nodes can compute slightly different results and then disagree on validity or state. Use integers or fixed-point and checked arithmetic (✓ `0.1 + 0.2 + 0.3` depends on the order).
9. A VM that charges for every instruction against a budget supplied with the transaction. Without it a loop never ends (the halting problem cannot be solved in advance); it must be deterministic (same prices, same checks before executing) so that every node stops at the same instruction and agrees on the outcome and the fee.
10. If a block applies half its commands and then fails, the node's state is neither the old nor the new one, and two nodes that fail at different points would disagree. Validate all, or apply to a copy or an undo log, and commit only if everything succeeded (✓ `a_block_is_all_or_nothing`).
11. Use a simulator driven by a seeded random generator and a priority queue of events; run many seeds, replay a failing seed exactly, and compare with an independent implementation. Determinism makes failures reproducible and tests fast, but the simulator only models what you put in it.
12. It bounds how long a single blocking `read` waits, so a silent peer cannot hold a thread forever. It does not bound the total time of a message that trickles in (each `read` resets the clock), nor memory use if you allocate up front, nor the number of connections.
13. Only best-effort delivery: a lost message may leave nodes that never receive it, particularly in sparse topologies (✓ a ring of 10 at 30% loss delivered to everyone in 25 of 200 runs). Reliability needs retransmission, acknowledgements or periodic reconciliation.

</details>

---

## 13. Practical challenge: `netlab`

Build a library crate that:

1. Reads and writes **length-prefixed frames** with a size limit checked before the body, handles partial reads and writes, retries `Interrupted`, and distinguishes a clean close, a truncated frame and an oversized frame.
2. Defines a **canonical binary message format** (Hello, Ping/Pong, Tx, GetBlocks, Inv, Blocks) with strict decoding: truncation, trailing bytes, unknown tags, counts and lengths checked before allocation.
3. Implements a **sans-IO peer session** with handshake rules, ping/pong liveness, a token-bucket rate limiter in integers, misbehavior scoring and timeouts, all driven by injected time.
4. Implements a **deterministic state machine** (ordered maps, checked arithmetic, atomic blocks, canonical snapshots, a SHA-256 state hash and a strict `restore`), demonstrations of the non-deterministic alternatives, and a **metered VM**.
5. Implements a **seeded network simulator** (xorshift generator, topologies, loss, duplication, delay, flooding) that is a pure function of its inputs.
6. Has real loopback TCP tests, a benchmark example, passes `cargo fmt` and `cargo clippy --all-targets -- -D warnings`, and survives mutation testing.

<details markdown="1">
<summary>Solution</summary>

### `Cargo.toml`

```toml
[package]
name = "netlab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "Teaching code for Lesson 50: framing, a peer protocol, deterministic execution and a seeded network simulator. Not audited."

[dependencies]
sha2 = "0.11"
```

### `src/lib.rs`

```rust
//! Teaching code for Lesson 50: length-prefixed framing, a binary peer
//! protocol, a sans-IO peer session, a deterministic state machine with a tiny
//! metered VM, and a seeded network simulator. **Not audited; not for production.**
#![deny(missing_docs)]

pub mod det;
pub mod frame;
pub mod msg;
pub mod peer;
pub mod sim;
```

### `src/frame.rs`

```rust
//! Length-prefixed framing over any `Read`/`Write`: a 4-byte big-endian length,
//! then that many bytes. TCP is a byte stream, not a message stream, so message
//! boundaries must be rebuilt by the application.

use std::fmt;
use std::io::{self, Read, Write};

/// Why a frame could not be read or written.
#[derive(Debug)]
pub enum FrameError {
    /// The peer closed the stream cleanly between frames.
    Closed,
    /// The stream ended in the middle of a frame.
    Truncated,
    /// The announced (or supplied) length exceeds the limit.
    TooLarge {
        /// The announced length.
        len: u64,
        /// The limit in force.
        max: usize,
    },
    /// Any other I/O error (including timeouts).
    Io(io::Error),
}

impl FrameError {
    /// Whether this is a read/write timeout (`WouldBlock` or `TimedOut`).
    pub fn is_timeout(&self) -> bool {
        matches!(self, FrameError::Io(e)
            if matches!(e.kind(), io::ErrorKind::WouldBlock | io::ErrorKind::TimedOut))
    }
}

impl fmt::Display for FrameError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            FrameError::Closed => write!(f, "connection closed"),
            FrameError::Truncated => write!(f, "connection closed in the middle of a frame"),
            FrameError::TooLarge { len, max } => {
                write!(f, "frame of {len} bytes exceeds the limit of {max}")
            }
            FrameError::Io(e) => write!(f, "i/o error: {e}"),
        }
    }
}

impl std::error::Error for FrameError {}

impl From<io::Error> for FrameError {
    fn from(e: io::Error) -> Self {
        FrameError::Io(e)
    }
}

/// Writes one frame. Refuses payloads longer than `max`.
pub fn write_frame<W: Write>(w: &mut W, payload: &[u8], max: usize) -> Result<(), FrameError> {
    if payload.len() > max || u32::try_from(payload.len()).is_err() {
        return Err(FrameError::TooLarge {
            len: payload.len() as u64,
            max,
        });
    }
    w.write_all(&(payload.len() as u32).to_be_bytes())?;
    w.write_all(payload)?;
    Ok(())
}

/// Reads one frame of at most `max` bytes.
///
/// The limit is checked **before** any payload is read, and the buffer grows
/// only as bytes actually arrive, so a peer that announces a large frame and
/// then sends nothing costs almost no memory.
pub fn read_frame<R: Read>(r: &mut R, max: usize) -> Result<Vec<u8>, FrameError> {
    let mut header = [0u8; 4];
    let mut got = 0;
    while got < 4 {
        match r.read(&mut header[got..]) {
            Ok(0) if got == 0 => return Err(FrameError::Closed),
            Ok(0) => return Err(FrameError::Truncated),
            Ok(n) => got += n,
            Err(e) if e.kind() == io::ErrorKind::Interrupted => {}
            Err(e) => return Err(e.into()),
        }
    }
    let len = u32::from_be_bytes(header) as u64;
    if len > max as u64 {
        return Err(FrameError::TooLarge { len, max });
    }
    let mut buf = Vec::new();
    r.take(len).read_to_end(&mut buf)?;
    if (buf.len() as u64) < len {
        return Err(FrameError::Truncated);
    }
    Ok(buf)
}
```

### `src/msg.rs`

```rust
//! A small binary peer protocol with a **canonical** encoding: every message has
//! exactly one byte representation, and the decoder rejects everything else
//! (unknown tags, trailing bytes, impossible counts, oversize items).

use std::fmt;

/// Longest transaction accepted in a `Tx` message.
pub const MAX_TX: usize = 64 * 1024;
/// Longest block accepted in a `Blocks` message.
pub const MAX_BLOCK: usize = 1024 * 1024;
/// Most hashes in one `Inv`.
pub const MAX_INV: usize = 1000;
/// Most blocks in one `Blocks` reply or `GetBlocks` request.
pub const MAX_BLOCKS: usize = 100;

/// A protocol message.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Message {
    /// First message each side sends.
    Hello {
        /// Protocol version.
        version: u16,
        /// Network identifier.
        chain_id: u32,
        /// Random identifier of the sending node.
        node_id: [u8; 8],
    },
    /// Liveness probe carrying a nonce.
    Ping(u64),
    /// Reply to a `Ping`, echoing its nonce.
    Pong(u64),
    /// A transaction.
    Tx(Vec<u8>),
    /// Request for up to `max` blocks starting at `from_height`.
    GetBlocks {
        /// First wanted height.
        from_height: u64,
        /// How many blocks, `1..=MAX_BLOCKS`.
        max: u16,
    },
    /// Announcement of known object hashes.
    Inv(Vec<[u8; 32]>),
    /// A batch of blocks.
    Blocks(Vec<Vec<u8>>),
}

/// Why bytes were not a valid message.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum DecodeError {
    /// The input ended before the message did.
    Truncated,
    /// The first byte is not a known tag.
    UnknownTag(u8),
    /// Bytes remain after a complete message.
    TrailingBytes,
    /// A count exceeds its limit.
    TooMany {
        /// The announced count.
        count: usize,
        /// The limit.
        max: usize,
    },
    /// An item is longer than its limit.
    TooLong {
        /// The announced length.
        len: usize,
        /// The limit.
        max: usize,
    },
    /// A field has a value the protocol forbids.
    BadValue(&'static str),
}

impl fmt::Display for DecodeError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DecodeError::Truncated => write!(f, "message is truncated"),
            DecodeError::UnknownTag(t) => write!(f, "unknown message tag {t}"),
            DecodeError::TrailingBytes => write!(f, "trailing bytes after message"),
            DecodeError::TooMany { count, max } => write!(f, "{count} items, limit {max}"),
            DecodeError::TooLong { len, max } => write!(f, "item of {len} bytes, limit {max}"),
            DecodeError::BadValue(what) => write!(f, "bad value: {what}"),
        }
    }
}

impl std::error::Error for DecodeError {}

struct Reader<'a> {
    rest: &'a [u8],
}

impl<'a> Reader<'a> {
    fn take(&mut self, n: usize) -> Result<&'a [u8], DecodeError> {
        if self.rest.len() < n {
            return Err(DecodeError::Truncated);
        }
        let (head, tail) = self.rest.split_at(n);
        self.rest = tail;
        Ok(head)
    }

    fn u16(&mut self) -> Result<u16, DecodeError> {
        Ok(u16::from_be_bytes(
            self.take(2)?.try_into().expect("2 bytes"),
        ))
    }

    fn u32(&mut self) -> Result<u32, DecodeError> {
        Ok(u32::from_be_bytes(
            self.take(4)?.try_into().expect("4 bytes"),
        ))
    }

    fn u64(&mut self) -> Result<u64, DecodeError> {
        Ok(u64::from_be_bytes(
            self.take(8)?.try_into().expect("8 bytes"),
        ))
    }

    /// A length-prefixed byte string no longer than `max`.
    fn bytes(&mut self, max: usize) -> Result<Vec<u8>, DecodeError> {
        let len = self.u32()? as usize;
        if len > max {
            return Err(DecodeError::TooLong { len, max });
        }
        Ok(self.take(len)?.to_vec())
    }
}

fn put_bytes(out: &mut Vec<u8>, b: &[u8]) {
    out.extend_from_slice(&(b.len() as u32).to_be_bytes());
    out.extend_from_slice(b);
}

impl Message {
    /// The canonical encoding.
    ///
    /// # Panics
    /// If the message violates the protocol limits (an oversize item or too
    /// many items): a local bug, since the decoder would refuse it.
    pub fn encode(&self) -> Vec<u8> {
        let mut out = Vec::new();
        match self {
            Message::Hello {
                version,
                chain_id,
                node_id,
            } => {
                out.push(1);
                out.extend_from_slice(&version.to_be_bytes());
                out.extend_from_slice(&chain_id.to_be_bytes());
                out.extend_from_slice(node_id);
            }
            Message::Ping(n) => {
                out.push(2);
                out.extend_from_slice(&n.to_be_bytes());
            }
            Message::Pong(n) => {
                out.push(3);
                out.extend_from_slice(&n.to_be_bytes());
            }
            Message::Tx(tx) => {
                assert!(tx.len() <= MAX_TX, "transaction too long");
                out.push(4);
                put_bytes(&mut out, tx);
            }
            Message::GetBlocks { from_height, max } => {
                assert!((1..=MAX_BLOCKS as u16).contains(max), "bad block count");
                out.push(5);
                out.extend_from_slice(&from_height.to_be_bytes());
                out.extend_from_slice(&max.to_be_bytes());
            }
            Message::Inv(hashes) => {
                assert!(hashes.len() <= MAX_INV, "too many hashes");
                out.push(6);
                out.extend_from_slice(&(hashes.len() as u16).to_be_bytes());
                for h in hashes {
                    out.extend_from_slice(h);
                }
            }
            Message::Blocks(blocks) => {
                assert!(blocks.len() <= MAX_BLOCKS, "too many blocks");
                out.push(7);
                out.extend_from_slice(&(blocks.len() as u16).to_be_bytes());
                for b in blocks {
                    assert!(b.len() <= MAX_BLOCK, "block too long");
                    put_bytes(&mut out, b);
                }
            }
        }
        out
    }

    /// Strict decoder: accepts exactly the bytes [`Message::encode`] produces.
    pub fn decode(bytes: &[u8]) -> Result<Message, DecodeError> {
        let mut r = Reader { rest: bytes };
        let tag = r.take(1)?[0];
        let msg = match tag {
            1 => {
                let version = r.u16()?;
                let chain_id = r.u32()?;
                let node_id = r.take(8)?.try_into().expect("8 bytes");
                Message::Hello {
                    version,
                    chain_id,
                    node_id,
                }
            }
            2 => Message::Ping(r.u64()?),
            3 => Message::Pong(r.u64()?),
            4 => Message::Tx(r.bytes(MAX_TX)?),
            5 => {
                let from_height = r.u64()?;
                let max = r.u16()?;
                if max == 0 || max as usize > MAX_BLOCKS {
                    return Err(DecodeError::BadValue("block count out of range"));
                }
                Message::GetBlocks { from_height, max }
            }
            6 => {
                let count = r.u16()? as usize;
                if count > MAX_INV {
                    return Err(DecodeError::TooMany {
                        count,
                        max: MAX_INV,
                    });
                }
                // check the bytes exist BEFORE allocating for `count` items
                if r.rest.len() < count * 32 {
                    return Err(DecodeError::Truncated);
                }
                let mut v = Vec::with_capacity(count);
                for _ in 0..count {
                    v.push(r.take(32)?.try_into().expect("32 bytes"));
                }
                Message::Inv(v)
            }
            7 => {
                let count = r.u16()? as usize;
                if count > MAX_BLOCKS {
                    return Err(DecodeError::TooMany {
                        count,
                        max: MAX_BLOCKS,
                    });
                }
                let mut v = Vec::new();
                for _ in 0..count {
                    v.push(r.bytes(MAX_BLOCK)?);
                }
                Message::Blocks(v)
            }
            t => return Err(DecodeError::UnknownTag(t)),
        };
        if !r.rest.is_empty() {
            return Err(DecodeError::TrailingBytes);
        }
        Ok(msg)
    }
}
```

### `src/peer.rs`

```rust
//! A "sans-IO" peer session: a state machine that is fed messages, decode
//! errors and clock ticks and returns actions. It never touches a socket or the
//! clock, so every rule can be tested exactly.

use crate::msg::{DecodeError, Message};

/// The only protocol version this node speaks.
pub const PROTOCOL_VERSION: u16 = 1;

/// Score added for a message dropped by the rate limiter.
pub const SCORE_RATE_LIMITED: u32 = 20;
/// Score added for bytes that did not decode.
pub const SCORE_BAD_MESSAGE: u32 = 50;
/// Score added for a `Pong` nobody asked for.
pub const SCORE_UNSOLICITED_PONG: u32 = 10;

/// Where the session is.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum State {
    /// Waiting for the peer's `Hello`.
    AwaitingHello,
    /// Handshake complete.
    Ready,
    /// The session ended; all further input is ignored.
    Closed,
}

/// Why the session ended.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Reason {
    /// The peer sent something other than `Hello` first.
    UnexpectedBeforeHello,
    /// The peer's version differs from ours.
    UnsupportedVersion(u16),
    /// The peer is on another network.
    WrongChain(u32),
    /// A second `Hello`.
    DuplicateHello,
    /// The peer has our own node id.
    SelfConnection,
    /// The misbehavior score reached the limit.
    Misbehaving,
    /// A `Ping` was not answered in time.
    PingTimeout,
    /// No `Hello` arrived in time.
    HandshakeTimeout,
}

/// What the caller must do.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Action {
    /// Send this message to the peer.
    Send(Message),
    /// Hand this message to the application layer.
    Deliver(Message),
    /// Close the connection.
    Disconnect(Reason),
}

/// Session settings.
#[derive(Clone, Debug)]
pub struct Config {
    /// Network identifier we accept.
    pub chain_id: u32,
    /// Our own node id.
    pub node_id: [u8; 8],
    /// Messages the peer may burst.
    pub bucket_capacity: u32,
    /// Messages per second the peer may sustain.
    pub refill_per_sec: u32,
    /// Misbehavior score at which we disconnect.
    pub ban_score: u32,
    /// Idle time before we ping, in milliseconds.
    pub ping_interval_ms: u64,
    /// Time allowed for the `Pong`, in milliseconds.
    pub ping_timeout_ms: u64,
    /// Time allowed for the `Hello`, in milliseconds.
    pub handshake_timeout_ms: u64,
}

/// Token bucket with integer arithmetic (milli-tokens) and injected time.
#[derive(Clone, Debug)]
pub struct RateLimiter {
    capacity_milli: u64,
    tokens_milli: u64,
    refill_per_sec: u64,
    last_ms: u64,
}

impl RateLimiter {
    /// A full bucket of `capacity` tokens refilling at `refill_per_sec`.
    pub fn new(capacity: u32, refill_per_sec: u32, now_ms: u64) -> Self {
        let cap = capacity as u64 * 1000;
        RateLimiter {
            capacity_milli: cap,
            tokens_milli: cap,
            refill_per_sec: refill_per_sec as u64,
            last_ms: now_ms,
        }
    }

    /// Takes one token if available. A clock that goes backwards adds nothing.
    pub fn allow(&mut self, now_ms: u64) -> bool {
        let elapsed = now_ms.saturating_sub(self.last_ms);
        if now_ms > self.last_ms {
            self.last_ms = now_ms;
        }
        // elapsed milliseconds * tokens per second = milli-tokens
        let added = elapsed.saturating_mul(self.refill_per_sec);
        self.tokens_milli = self
            .tokens_milli
            .saturating_add(added)
            .min(self.capacity_milli);
        if self.tokens_milli >= 1000 {
            self.tokens_milli -= 1000;
            true
        } else {
            false
        }
    }
}

/// One peer connection's protocol state.
#[derive(Debug)]
pub struct Session {
    cfg: Config,
    state: State,
    score: u32,
    limiter: RateLimiter,
    started_ms: u64,
    last_activity_ms: u64,
    outstanding: Option<(u64, u64)>,
    next_nonce: u64,
}

impl Session {
    /// A new session created at time `now_ms`.
    pub fn new(cfg: Config, now_ms: u64) -> Self {
        let limiter = RateLimiter::new(cfg.bucket_capacity, cfg.refill_per_sec, now_ms);
        Session {
            cfg,
            state: State::AwaitingHello,
            score: 0,
            limiter,
            started_ms: now_ms,
            last_activity_ms: now_ms,
            outstanding: None,
            next_nonce: 1,
        }
    }

    /// Current state.
    pub fn state(&self) -> State {
        self.state
    }

    /// Current misbehavior score.
    pub fn score(&self) -> u32 {
        self.score
    }

    /// The `Hello` we send when the connection opens.
    pub fn start(&self) -> Vec<Action> {
        vec![Action::Send(Message::Hello {
            version: PROTOCOL_VERSION,
            chain_id: self.cfg.chain_id,
            node_id: self.cfg.node_id,
        })]
    }

    fn close(&mut self, why: Reason) -> Vec<Action> {
        self.state = State::Closed;
        vec![Action::Disconnect(why)]
    }

    fn add_score(&mut self, points: u32) -> Vec<Action> {
        self.score = self.score.saturating_add(points);
        if self.score >= self.cfg.ban_score {
            self.close(Reason::Misbehaving)
        } else {
            vec![]
        }
    }

    /// Handles a decoded message received at `now_ms`.
    pub fn on_message(&mut self, msg: Message, now_ms: u64) -> Vec<Action> {
        if self.state == State::Closed {
            return vec![];
        }
        self.last_activity_ms = now_ms;
        if !self.limiter.allow(now_ms) {
            return self.add_score(SCORE_RATE_LIMITED);
        }
        match (self.state, msg) {
            (
                State::AwaitingHello,
                Message::Hello {
                    version,
                    chain_id,
                    node_id,
                },
            ) => {
                if version != PROTOCOL_VERSION {
                    self.close(Reason::UnsupportedVersion(version))
                } else if chain_id != self.cfg.chain_id {
                    self.close(Reason::WrongChain(chain_id))
                } else if node_id == self.cfg.node_id {
                    self.close(Reason::SelfConnection)
                } else {
                    self.state = State::Ready;
                    vec![]
                }
            }
            (State::AwaitingHello, _) => self.close(Reason::UnexpectedBeforeHello),
            (_, Message::Hello { .. }) => self.close(Reason::DuplicateHello),
            (_, Message::Ping(n)) => vec![Action::Send(Message::Pong(n))],
            (_, Message::Pong(n)) => match self.outstanding {
                Some((expected, _)) if expected == n => {
                    self.outstanding = None;
                    vec![]
                }
                _ => self.add_score(SCORE_UNSOLICITED_PONG),
            },
            (_, other) => vec![Action::Deliver(other)],
        }
    }

    /// Handles bytes that failed to decode.
    pub fn on_decode_error(&mut self, _err: &DecodeError) -> Vec<Action> {
        if self.state == State::Closed {
            return vec![];
        }
        self.add_score(SCORE_BAD_MESSAGE)
    }

    /// Advances timers to `now_ms`.
    pub fn tick(&mut self, now_ms: u64) -> Vec<Action> {
        match self.state {
            State::Closed => vec![],
            State::AwaitingHello => {
                if now_ms.saturating_sub(self.started_ms) > self.cfg.handshake_timeout_ms {
                    self.close(Reason::HandshakeTimeout)
                } else {
                    vec![]
                }
            }
            State::Ready => match self.outstanding {
                Some((_, sent)) => {
                    if now_ms.saturating_sub(sent) > self.cfg.ping_timeout_ms {
                        self.close(Reason::PingTimeout)
                    } else {
                        vec![]
                    }
                }
                None => {
                    if now_ms.saturating_sub(self.last_activity_ms) >= self.cfg.ping_interval_ms {
                        let n = self.next_nonce;
                        self.next_nonce += 1;
                        self.outstanding = Some((n, now_ms));
                        vec![Action::Send(Message::Ping(n))]
                    } else {
                        vec![]
                    }
                }
            },
        }
    }
}
```

### `src/det.rs`

```rust
//! Deterministic execution: a ledger state machine with a canonical state hash
//! and snapshots, a tiny metered virtual machine, and small demonstrations of
//! the usual sources of non-determinism.

use std::collections::{BTreeMap, HashMap};

use sha2::{Digest, Sha256};

/// A 32-byte digest.
pub type Hash = [u8; 32];

// ------------------------------------------------------------ state machine

/// A command for the ledger state machine.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Cmd {
    /// Create `amount` in `acct`.
    Credit {
        /// Account.
        acct: u64,
        /// Amount.
        amount: u64,
    },
    /// Move `amount` between accounts.
    Transfer {
        /// Source.
        from: u64,
        /// Destination.
        to: u64,
        /// Amount.
        amount: u64,
    },
}

/// Why a command failed.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum CmdError {
    /// Amount zero.
    ZeroAmount,
    /// The source cannot pay (or does not exist).
    Insufficient,
    /// A balance would overflow.
    Overflow,
}

/// Deterministic ledger state: ordered map plus a block height.
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub struct Machine {
    accounts: BTreeMap<u64, u64>,
    height: u64,
}

const STATE_DOMAIN: &[u8] = b"lab/state/v1";

impl Machine {
    /// An empty machine at height 0.
    pub fn new() -> Self {
        Self::default()
    }

    /// Balance of `acct` (0 if unknown).
    pub fn balance(&self, acct: u64) -> u64 {
        self.accounts.get(&acct).copied().unwrap_or(0)
    }

    /// Number of blocks applied.
    pub fn height(&self) -> u64 {
        self.height
    }

    /// Sum of all balances.
    pub fn total(&self) -> u128 {
        self.accounts.values().map(|v| *v as u128).sum()
    }

    /// Applies one command; on error nothing changes.
    pub fn apply(&mut self, cmd: Cmd) -> Result<(), CmdError> {
        match cmd {
            Cmd::Credit { acct, amount } => {
                if amount == 0 {
                    return Err(CmdError::ZeroAmount);
                }
                let new = self
                    .balance(acct)
                    .checked_add(amount)
                    .ok_or(CmdError::Overflow)?;
                self.accounts.insert(acct, new);
            }
            Cmd::Transfer { from, to, amount } => {
                if amount == 0 {
                    return Err(CmdError::ZeroAmount);
                }
                let from_new = self
                    .balance(from)
                    .checked_sub(amount)
                    .ok_or(CmdError::Insufficient)?;
                if from == to {
                    return Ok(());
                }
                let to_new = self
                    .balance(to)
                    .checked_add(amount)
                    .ok_or(CmdError::Overflow)?;
                self.accounts.insert(from, from_new);
                self.accounts.insert(to, to_new);
            }
        }
        Ok(())
    }

    /// Applies a block of commands atomically: either every command succeeds
    /// and the height increases by one, or nothing changes and the index of the
    /// failing command is returned.
    pub fn apply_block(&mut self, cmds: &[Cmd]) -> Result<(), (usize, CmdError)> {
        let mut staged = self.clone();
        for (i, c) in cmds.iter().enumerate() {
            staged.apply(*c).map_err(|e| (i, e))?;
        }
        staged.height += 1;
        *self = staged;
        Ok(())
    }

    /// Hash of the whole state: domain, height, count, then every
    /// `(account, balance)` pair in ascending account order.
    pub fn state_hash(&self) -> Hash {
        Sha256::digest(self.snapshot()).into()
    }

    /// Canonical serialization of the state.
    pub fn snapshot(&self) -> Vec<u8> {
        let mut v = Vec::new();
        v.extend_from_slice(&(STATE_DOMAIN.len() as u64).to_be_bytes());
        v.extend_from_slice(STATE_DOMAIN);
        v.extend_from_slice(&self.height.to_be_bytes());
        v.extend_from_slice(&(self.accounts.len() as u64).to_be_bytes());
        for (k, b) in &self.accounts {
            v.extend_from_slice(&k.to_be_bytes());
            v.extend_from_slice(&b.to_be_bytes());
        }
        v
    }

    /// Strict inverse of [`Machine::snapshot`]: wrong domain, trailing bytes,
    /// unsorted or duplicate accounts, zero balances and impossible counts are
    /// all refused.
    pub fn restore(bytes: &[u8]) -> Option<Machine> {
        let head = 8 + STATE_DOMAIN.len();
        if bytes.len() < head + 16 {
            return None;
        }
        if bytes[..8] != (STATE_DOMAIN.len() as u64).to_be_bytes()
            || &bytes[8..head] != STATE_DOMAIN
        {
            return None;
        }
        let height = u64::from_be_bytes(bytes[head..head + 8].try_into().ok()?);
        let count = u64::from_be_bytes(bytes[head + 8..head + 16].try_into().ok()?);
        let body = &bytes[head + 16..];
        if body.len() as u64 != count.checked_mul(16)? {
            return None;
        }
        let mut accounts = BTreeMap::new();
        let mut prev: Option<u64> = None;
        for chunk in body.chunks_exact(16) {
            let k = u64::from_be_bytes(chunk[..8].try_into().ok()?);
            let b = u64::from_be_bytes(chunk[8..].try_into().ok()?);
            if b == 0 || prev.is_some_and(|p| p >= k) {
                return None;
            }
            prev = Some(k);
            accounts.insert(k, b);
        }
        Some(Machine { accounts, height })
    }
}

// ------------------------------------------------- sources of non-determinism

/// **Flawed on purpose**: hashes a `HashMap` in iteration order. The standard
/// `HashMap` uses a randomly keyed hasher, so two maps with equal contents can
/// iterate differently.
pub fn hash_in_map_order(map: &HashMap<u64, u64>) -> Hash {
    let mut h = Sha256::new();
    for (k, v) in map {
        h.update(k.to_be_bytes());
        h.update(v.to_be_bytes());
    }
    h.finalize().into()
}

/// Hashes a `HashMap` in ascending key order, which is deterministic.
pub fn hash_in_key_order(map: &HashMap<u64, u64>) -> Hash {
    let mut keys: Vec<&u64> = map.keys().collect();
    keys.sort_unstable();
    let mut h = Sha256::new();
    for k in keys {
        h.update(k.to_be_bytes());
        h.update(map[k].to_be_bytes());
    }
    h.finalize().into()
}

/// Left-to-right floating-point sum. Floating-point addition is not
/// associative, so the result depends on the order.
pub fn sum_f64(xs: &[f64]) -> f64 {
    xs.iter().fold(0.0, |a, x| a + x)
}

/// Fixed-point (integer) sum: exact, order-independent, and `None` on overflow.
pub fn sum_fixed(xs: &[i64]) -> Option<i64> {
    xs.iter().try_fold(0i64, |a, x| a.checked_add(*x))
}

// ------------------------------------------------------------------ metered VM

/// A VM instruction.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Op {
    /// Push a constant.
    Push(i64),
    /// Pop `b`, pop `a`, push `a + b`.
    Add,
    /// Pop `b`, pop `a`, push `a - b`.
    Sub,
    /// Pop `b`, pop `a`, push `a * b`.
    Mul,
    /// Pop `b`, pop `a`, push `a / b` (truncating toward zero).
    Div,
    /// Duplicate the top.
    Dup,
    /// Swap the top two.
    Swap,
    /// Drop the top.
    Pop,
    /// Jump to an instruction index (`len` means halt).
    Jmp(usize),
    /// Pop a value; jump if it is not zero.
    Jnz(usize),
    /// Stop.
    Halt,
}

/// Why execution stopped abnormally. Every trap is deterministic.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Trap {
    /// Not enough fuel for the next instruction.
    OutOfFuel,
    /// Popped from an empty (or too short) stack.
    StackUnderflow,
    /// The stack would exceed [`MAX_STACK`].
    StackOverflow,
    /// Checked arithmetic overflowed.
    Overflow,
    /// Division by zero.
    DivideByZero,
    /// Jump target beyond the end of the program.
    BadJump(usize),
}

/// Maximum stack depth.
pub const MAX_STACK: usize = 256;

/// Result of a successful run.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct Outcome {
    /// Top of the stack when execution stopped, if any.
    pub top: Option<i64>,
    /// Fuel consumed.
    pub fuel_used: u64,
    /// Instructions executed.
    pub steps: u64,
}

fn cost(op: &Op) -> u64 {
    match op {
        Op::Mul => 3,
        Op::Div => 5,
        _ => 1,
    }
}

/// Runs `program` with `fuel` units. Falling off the end is a normal halt.
pub fn run(program: &[Op], fuel: u64) -> Result<Outcome, Trap> {
    let mut stack: Vec<i64> = Vec::new();
    let mut pc = 0usize;
    let mut left = fuel;
    let mut steps = 0u64;

    fn pop(stack: &mut Vec<i64>) -> Result<i64, Trap> {
        stack.pop().ok_or(Trap::StackUnderflow)
    }
    fn push(stack: &mut Vec<i64>, v: i64) -> Result<(), Trap> {
        if stack.len() >= MAX_STACK {
            return Err(Trap::StackOverflow);
        }
        stack.push(v);
        Ok(())
    }

    while pc < program.len() {
        let op = program[pc];
        let c = cost(&op);
        if left < c {
            return Err(Trap::OutOfFuel);
        }
        left -= c;
        steps += 1;
        pc += 1;
        match op {
            Op::Push(v) => push(&mut stack, v)?,
            Op::Add => {
                let b = pop(&mut stack)?;
                let a = pop(&mut stack)?;
                push(&mut stack, a.checked_add(b).ok_or(Trap::Overflow)?)?;
            }
            Op::Sub => {
                let b = pop(&mut stack)?;
                let a = pop(&mut stack)?;
                push(&mut stack, a.checked_sub(b).ok_or(Trap::Overflow)?)?;
            }
            Op::Mul => {
                let b = pop(&mut stack)?;
                let a = pop(&mut stack)?;
                push(&mut stack, a.checked_mul(b).ok_or(Trap::Overflow)?)?;
            }
            Op::Div => {
                let b = pop(&mut stack)?;
                let a = pop(&mut stack)?;
                if b == 0 {
                    return Err(Trap::DivideByZero);
                }
                push(&mut stack, a.checked_div(b).ok_or(Trap::Overflow)?)?;
            }
            Op::Dup => {
                let a = *stack.last().ok_or(Trap::StackUnderflow)?;
                push(&mut stack, a)?;
            }
            Op::Swap => {
                let n = stack.len();
                if n < 2 {
                    return Err(Trap::StackUnderflow);
                }
                stack.swap(n - 1, n - 2);
            }
            Op::Pop => {
                pop(&mut stack)?;
            }
            Op::Jmp(t) => {
                if t > program.len() {
                    return Err(Trap::BadJump(t));
                }
                pc = t;
            }
            Op::Jnz(t) => {
                let c = pop(&mut stack)?;
                if c != 0 {
                    if t > program.len() {
                        return Err(Trap::BadJump(t));
                    }
                    pc = t;
                }
            }
            Op::Halt => break,
        }
    }
    Ok(Outcome {
        top: stack.last().copied(),
        fuel_used: fuel - left,
        steps,
    })
}
```

### `src/sim.rs`

```rust
//! A seeded, single-threaded network simulator: messages are events in a queue
//! ordered by (time, sequence number), and loss, duplication and delay come from
//! a small PRNG, so a run is a pure function of its inputs.

use std::cmp::Reverse;
use std::collections::BinaryHeap;

/// xorshift64* pseudo-random generator (Vigna's variant, as remembered).
#[derive(Clone, Debug)]
pub struct Rng(u64);

impl Rng {
    /// A generator from `seed` (zero is replaced by a fixed non-zero constant,
    /// since xorshift generators cannot leave the all-zero state).
    pub fn new(seed: u64) -> Self {
        Rng(if seed == 0 {
            0x9E37_79B9_7F4A_7C15
        } else {
            seed
        })
    }

    /// The next 64-bit value.
    pub fn next_u64(&mut self) -> u64 {
        let mut x = self.0;
        x ^= x >> 12;
        x ^= x << 25;
        x ^= x >> 27;
        self.0 = x;
        x.wrapping_mul(0x2545_F491_4F6C_DD1D)
    }

    /// A value in `0..n`, without modulo bias (rejection sampling).
    pub fn below(&mut self, n: u64) -> u64 {
        assert!(n > 0, "below(0) is meaningless");
        let limit = u64::MAX - u64::MAX % n;
        loop {
            let x = self.next_u64();
            if x < limit {
                return x % n;
            }
        }
    }
}

/// An undirected network of `n` nodes.
#[derive(Clone, Debug)]
pub struct Network {
    adj: Vec<Vec<usize>>,
}

impl Network {
    /// Builds a network from undirected edges. Self-loops and duplicates are ignored.
    pub fn from_edges(n: usize, edges: &[(usize, usize)]) -> Self {
        let mut adj = vec![Vec::new(); n];
        for &(a, b) in edges {
            assert!(a < n && b < n, "edge endpoint out of range");
            if a != b && !adj[a].contains(&b) {
                adj[a].push(b);
                adj[b].push(a);
            }
        }
        for l in &mut adj {
            l.sort_unstable();
        }
        Network { adj }
    }

    /// A ring of `n` nodes.
    pub fn ring(n: usize) -> Self {
        let edges: Vec<_> = (0..n).map(|i| (i, (i + 1) % n)).collect();
        Self::from_edges(n, &edges)
    }

    /// A line of `n` nodes.
    pub fn line(n: usize) -> Self {
        let edges: Vec<_> = (1..n).map(|i| (i - 1, i)).collect();
        Self::from_edges(n, &edges)
    }

    /// Every pair of `n` nodes connected.
    pub fn complete(n: usize) -> Self {
        let mut edges = Vec::new();
        for a in 0..n {
            for b in a + 1..n {
                edges.push((a, b));
            }
        }
        Self::from_edges(n, &edges)
    }

    /// Number of nodes.
    pub fn len(&self) -> usize {
        self.adj.len()
    }

    /// Whether there are no nodes.
    pub fn is_empty(&self) -> bool {
        self.adj.is_empty()
    }

    /// Number of undirected edges.
    pub fn edges(&self) -> usize {
        self.adj.iter().map(|l| l.len()).sum::<usize>() / 2
    }
}

/// Link behavior.
#[derive(Clone, Copy, Debug)]
pub struct Link {
    /// Chance in 1/1000 that a message is lost.
    pub drop_per_mille: u32,
    /// Chance in 1/1000 that a delivered message arrives twice.
    pub dup_per_mille: u32,
    /// Delay is uniform in `1..=max_delay` ticks.
    pub max_delay: u64,
}

/// What a flood run produced.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Report {
    /// Tick of first receipt at each node (`None` if never reached).
    pub first_receipt: Vec<Option<u64>>,
    /// Messages handed to the network (before loss).
    pub sent: u64,
    /// Messages lost.
    pub dropped: u64,
    /// Extra copies created.
    pub duplicated: u64,
    /// Events processed.
    pub events: u64,
    /// Tick of the last event.
    pub finished_at: u64,
}

impl Report {
    /// Number of nodes that received the item (the origin counts).
    pub fn reached(&self) -> usize {
        self.first_receipt.iter().filter(|t| t.is_some()).count()
    }
}

/// Floods one item from `origin`: a node that receives it for the first time
/// forwards it to every neighbor except the one it came from.
pub fn flood(net: &Network, link: Link, seed: u64, origin: usize) -> Report {
    assert!(origin < net.len(), "origin out of range");
    assert!(link.max_delay >= 1, "max_delay must be at least 1");
    let mut rng = Rng::new(seed);
    let mut queue: BinaryHeap<Reverse<(u64, u64, usize, usize)>> = BinaryHeap::new();
    let mut seq = 0u64;
    let mut rep = Report {
        first_receipt: vec![None; net.len()],
        sent: 0,
        dropped: 0,
        duplicated: 0,
        events: 0,
        finished_at: 0,
    };

    let mut send = |from: usize,
                    to: usize,
                    now: u64,
                    rep: &mut Report,
                    rng: &mut Rng,
                    queue: &mut BinaryHeap<Reverse<(u64, u64, usize, usize)>>| {
        rep.sent += 1;
        // the draws happen in a fixed order whether or not they matter
        let lost = rng.below(1000) < link.drop_per_mille as u64;
        let delay = 1 + rng.below(link.max_delay);
        let twice = rng.below(1000) < link.dup_per_mille as u64;
        let delay2 = 1 + rng.below(link.max_delay);
        if lost {
            rep.dropped += 1;
            return;
        }
        queue.push(Reverse((now + delay, seq, from, to)));
        seq += 1;
        if twice {
            rep.duplicated += 1;
            queue.push(Reverse((now + delay2, seq, from, to)));
            seq += 1;
        }
    };

    rep.first_receipt[origin] = Some(0);
    for &n in &net.adj[origin] {
        send(origin, n, 0, &mut rep, &mut rng, &mut queue);
    }
    while let Some(Reverse((t, _, from, to))) = queue.pop() {
        rep.events += 1;
        rep.finished_at = t;
        if rep.first_receipt[to].is_some() {
            continue;
        }
        rep.first_receipt[to] = Some(t);
        for &n in &net.adj[to] {
            if n != from {
                send(to, n, t, &mut rep, &mut rng, &mut queue);
            }
        }
    }
    rep
}
```

### `tests/frame.rs`

```rust
use std::io::{self, Cursor, Read, Write};

use netlab::frame::{FrameError, read_frame, write_frame};

/// Returns at most one byte per `read` call.
struct Dribble<R>(R);
impl<R: Read> Read for Dribble<R> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let n = buf.len().min(1);
        self.0.read(&mut buf[..n])
    }
}

/// Accepts at most one byte per `write` call.
struct SlowSink(Vec<u8>);
impl Write for SlowSink {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        let n = buf.len().min(1);
        self.0.extend_from_slice(&buf[..n]);
        Ok(n)
    }
    fn flush(&mut self) -> io::Result<()> {
        Ok(())
    }
}

/// Fails every other call with `Interrupted`.
struct Flaky<R> {
    inner: R,
    toggle: bool,
}
impl<R: Read> Read for Flaky<R> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        self.toggle = !self.toggle;
        if self.toggle {
            Err(io::Error::from(io::ErrorKind::Interrupted))
        } else {
            self.inner.read(buf)
        }
    }
}

struct Failing(io::ErrorKind);
impl Read for Failing {
    fn read(&mut self, _: &mut [u8]) -> io::Result<usize> {
        Err(io::Error::from(self.0))
    }
}

fn framed(payload: &[u8]) -> Vec<u8> {
    let mut v = Vec::new();
    write_frame(&mut v, payload, 1 << 20).unwrap();
    v
}

#[test]
fn a_frame_is_a_big_endian_length_then_the_bytes() {
    assert_eq!(framed(b"abc"), [0, 0, 0, 3, b'a', b'b', b'c']);
    assert_eq!(framed(b""), [0, 0, 0, 0]);
    let big = vec![7u8; 258];
    assert_eq!(&framed(&big)[..4], [0, 0, 1, 2]);
}

#[test]
fn frames_round_trip_and_stay_separate_in_one_stream() {
    let mut stream = Vec::new();
    for p in [&b"one"[..], b"", b"three!"] {
        write_frame(&mut stream, p, 100).unwrap();
    }
    let mut r = Cursor::new(stream);
    assert_eq!(read_frame(&mut r, 100).unwrap(), b"one");
    assert_eq!(read_frame(&mut r, 100).unwrap(), b"");
    assert_eq!(read_frame(&mut r, 100).unwrap(), b"three!");
    assert!(matches!(read_frame(&mut r, 100), Err(FrameError::Closed)));
}

#[test]
fn reading_works_when_the_stream_delivers_one_byte_at_a_time() {
    let mut r = Dribble(Cursor::new(framed(b"hello world")));
    assert_eq!(read_frame(&mut r, 100).unwrap(), b"hello world");
}

#[test]
fn writing_works_when_the_sink_accepts_one_byte_at_a_time() {
    let mut sink = SlowSink(Vec::new());
    write_frame(&mut sink, b"hello world", 100).unwrap();
    assert_eq!(sink.0, framed(b"hello world"));
}

#[test]
fn interrupted_reads_are_retried() {
    let mut r = Flaky {
        inner: Cursor::new(framed(b"payload")),
        toggle: false,
    };
    assert_eq!(read_frame(&mut r, 100).unwrap(), b"payload");
}

#[test]
fn closing_between_frames_is_not_an_error_in_the_middle_is() {
    assert!(matches!(
        read_frame(&mut Cursor::new(vec![]), 10),
        Err(FrameError::Closed)
    ));
    for cut in 1..4 {
        let data = framed(b"abc")[..cut].to_vec();
        assert!(
            matches!(
                read_frame(&mut Cursor::new(data), 10),
                Err(FrameError::Truncated)
            ),
            "cut {cut}"
        );
    }
    for cut in 4..7 {
        let data = framed(b"abc")[..cut].to_vec();
        assert!(
            matches!(
                read_frame(&mut Cursor::new(data), 10),
                Err(FrameError::Truncated)
            ),
            "cut {cut}"
        );
    }
}

#[test]
fn the_limit_is_inclusive_and_checked_before_the_body_is_read() {
    assert!(read_frame(&mut Cursor::new(framed(&[1; 10])), 10).is_ok());
    let err = read_frame(&mut Cursor::new(framed(&[1; 11])), 10).unwrap_err();
    assert!(
        matches!(err, FrameError::TooLarge { len: 11, max: 10 }),
        "{err:?}"
    );
    // a header alone is enough to refuse: no body follows
    let header_only = u32::MAX.to_be_bytes().to_vec();
    let err = read_frame(&mut Cursor::new(header_only), 1 << 20).unwrap_err();
    assert!(
        matches!(err, FrameError::TooLarge { len, .. } if len == u32::MAX as u64),
        "{err:?}"
    );
}

#[test]
fn a_zero_limit_allows_only_empty_frames() {
    assert_eq!(read_frame(&mut Cursor::new(framed(b"")), 0).unwrap(), b"");
    assert!(matches!(
        read_frame(&mut Cursor::new(framed(b"x")), 0),
        Err(FrameError::TooLarge { .. })
    ));
}

#[test]
fn writing_refuses_oversize_payloads_and_writes_nothing() {
    let mut sink = Vec::new();
    let err = write_frame(&mut sink, &[0; 11], 10).unwrap_err();
    assert!(matches!(err, FrameError::TooLarge { len: 11, max: 10 }));
    assert!(sink.is_empty());
    write_frame(&mut sink, &[0; 10], 10).unwrap();
    assert_eq!(sink.len(), 14);
}

#[test]
fn only_would_block_and_timed_out_count_as_timeouts() {
    for (kind, timeout) in [
        (io::ErrorKind::WouldBlock, true),
        (io::ErrorKind::TimedOut, true),
        (io::ErrorKind::ConnectionReset, false),
        (io::ErrorKind::UnexpectedEof, false),
    ] {
        let e = read_frame(&mut Failing(kind), 10).unwrap_err();
        assert_eq!(e.is_timeout(), timeout, "{kind:?}");
        assert!(matches!(e, FrameError::Io(_)));
    }
    assert!(!FrameError::Closed.is_timeout());
    assert!(!FrameError::Truncated.is_timeout());
}

#[test]
fn error_messages_are_readable() {
    assert_eq!(FrameError::Closed.to_string(), "connection closed");
    assert_eq!(
        FrameError::Truncated.to_string(),
        "connection closed in the middle of a frame"
    );
    assert_eq!(
        FrameError::TooLarge { len: 11, max: 10 }.to_string(),
        "frame of 11 bytes exceeds the limit of 10"
    );
    assert!(
        FrameError::Io(io::Error::from(io::ErrorKind::TimedOut))
            .to_string()
            .starts_with("i/o error: ")
    );
}

#[test]
fn the_reader_never_consumes_bytes_past_the_frame() {
    let mut stream = framed(b"first");
    stream.extend_from_slice(b"NEXT");
    let mut r = Cursor::new(stream);
    assert_eq!(read_frame(&mut r, 100).unwrap(), b"first");
    assert_eq!(r.position(), 9);
}
```

### `tests/msg.rs`

```rust
use netlab::msg::*;
use netlab::sim::Rng;

fn samples() -> Vec<Message> {
    vec![
        Message::Hello {
            version: 1,
            chain_id: 7,
            node_id: [1, 2, 3, 4, 5, 6, 7, 8],
        },
        Message::Ping(0),
        Message::Ping(u64::MAX),
        Message::Pong(42),
        Message::Tx(vec![]),
        Message::Tx(vec![9; 300]),
        Message::GetBlocks {
            from_height: 123_456,
            max: 1,
        },
        Message::GetBlocks {
            from_height: 0,
            max: 100,
        },
        Message::Inv(vec![]),
        Message::Inv(vec![[0xab; 32], [0xcd; 32]]),
        Message::Blocks(vec![]),
        Message::Blocks(vec![vec![], vec![1, 2, 3]]),
    ]
}

#[test]
fn every_message_round_trips() {
    for m in samples() {
        assert_eq!(Message::decode(&m.encode()).as_ref(), Ok(&m), "{m:?}");
    }
}

#[test]
fn the_wire_format_is_fixed() {
    let hello = Message::Hello {
        version: 1,
        chain_id: 0x0102_0304,
        node_id: [9; 8],
    };
    assert_eq!(
        hello.encode(),
        [1, 0, 1, 1, 2, 3, 4, 9, 9, 9, 9, 9, 9, 9, 9]
    );
    assert_eq!(Message::Ping(0x0102).encode(), [2, 0, 0, 0, 0, 0, 0, 1, 2]);
    assert_eq!(Message::Pong(1).encode(), [3, 0, 0, 0, 0, 0, 0, 0, 1]);
    assert_eq!(
        Message::Tx(vec![0xaa, 0xbb]).encode(),
        [4, 0, 0, 0, 2, 0xaa, 0xbb]
    );
    assert_eq!(
        Message::GetBlocks {
            from_height: 5,
            max: 3
        }
        .encode(),
        [5, 0, 0, 0, 0, 0, 0, 0, 5, 0, 3]
    );
    let mut inv = vec![6, 0, 1];
    inv.extend_from_slice(&[7; 32]);
    assert_eq!(Message::Inv(vec![[7; 32]]).encode(), inv);
    assert_eq!(
        Message::Blocks(vec![vec![1]]).encode(),
        [7, 0, 1, 0, 0, 0, 1, 1]
    );
}

#[test]
fn truncation_is_detected_at_every_cut() {
    for m in samples() {
        let bytes = m.encode();
        for cut in 0..bytes.len() {
            assert!(
                Message::decode(&bytes[..cut]).is_err(),
                "{m:?} cut at {cut}"
            );
        }
    }
}

#[test]
fn trailing_bytes_are_refused() {
    for m in samples() {
        let mut bytes = m.encode();
        bytes.push(0);
        assert_eq!(
            Message::decode(&bytes),
            Err(DecodeError::TrailingBytes),
            "{m:?}"
        );
    }
}

#[test]
fn unknown_tags_are_refused() {
    for t in [0u8, 8, 9, 100, 255] {
        assert_eq!(
            Message::decode(&[t, 0, 0, 0, 0]),
            Err(DecodeError::UnknownTag(t))
        );
    }
    assert_eq!(Message::decode(&[]), Err(DecodeError::Truncated));
}

#[test]
fn counts_and_lengths_are_checked_before_anything_is_allocated() {
    // 1001 hashes announced, none present: refused by the limit, not by running out of bytes
    assert_eq!(
        Message::decode(&[6, 0x03, 0xe9]),
        Err(DecodeError::TooMany {
            count: 1001,
            max: 1000
        })
    );
    // 1000 announced and absent: truncated (no 32 KB allocation is needed to find out)
    assert_eq!(
        Message::decode(&[6, 0x03, 0xe8]),
        Err(DecodeError::Truncated)
    );
    // 65535 announced
    assert_eq!(
        Message::decode(&[6, 0xff, 0xff]),
        Err(DecodeError::TooMany {
            count: 65535,
            max: 1000
        })
    );
    assert_eq!(
        Message::decode(&[7, 0, 101]),
        Err(DecodeError::TooMany {
            count: 101,
            max: 100
        })
    );
    assert_eq!(Message::decode(&[7, 0, 100]), Err(DecodeError::Truncated));
    // a 4 GiB transaction or block is refused by its length field alone
    assert_eq!(
        Message::decode(&[4, 0xff, 0xff, 0xff, 0xff]),
        Err(DecodeError::TooLong {
            len: u32::MAX as usize,
            max: MAX_TX
        })
    );
    let mut tx = vec![4];
    tx.extend_from_slice(&((MAX_TX + 1) as u32).to_be_bytes());
    assert_eq!(
        Message::decode(&tx),
        Err(DecodeError::TooLong {
            len: MAX_TX + 1,
            max: MAX_TX
        })
    );
    let mut blocks = vec![7, 0, 1];
    blocks.extend_from_slice(&((MAX_BLOCK + 1) as u32).to_be_bytes());
    assert_eq!(
        Message::decode(&blocks),
        Err(DecodeError::TooLong {
            len: MAX_BLOCK + 1,
            max: MAX_BLOCK
        })
    );
}

#[test]
fn the_limits_themselves_are_accepted() {
    let big = Message::Tx(vec![0; MAX_TX]);
    assert_eq!(Message::decode(&big.encode()), Ok(big));
    let inv = Message::Inv(vec![[1; 32]; MAX_INV]);
    assert_eq!(Message::decode(&inv.encode()), Ok(inv));
    let blocks = Message::Blocks(vec![vec![2; 10]; MAX_BLOCKS]);
    assert_eq!(Message::decode(&blocks.encode()), Ok(blocks));
}

#[test]
fn block_counts_in_get_blocks_must_be_in_range() {
    let mk = |max: u16| {
        let mut b = vec![5, 0, 0, 0, 0, 0, 0, 0, 1];
        b.extend_from_slice(&max.to_be_bytes());
        b
    };
    assert_eq!(
        Message::decode(&mk(0)),
        Err(DecodeError::BadValue("block count out of range"))
    );
    assert_eq!(
        Message::decode(&mk(101)),
        Err(DecodeError::BadValue("block count out of range"))
    );
    assert!(Message::decode(&mk(1)).is_ok());
    assert!(Message::decode(&mk(100)).is_ok());
}

#[test]
fn random_bytes_never_panic_and_accepted_input_is_canonical() {
    // The decoder accepts exactly the encoder's output: decode(b) = Ok(m) implies encode(m) = b.
    let mut rng = Rng::new(2024);
    let mut accepted = 0;
    for i in 0..60_000 {
        let len = rng.below(40) as usize;
        let mut b: Vec<u8> = (0..len).map(|_| rng.next_u64() as u8).collect();
        if i % 2 == 0 && !b.is_empty() {
            b[0] = 1 + (rng.below(7) as u8); // bias toward real tags
        }
        if let Ok(m) = Message::decode(&b) {
            accepted += 1;
            assert_eq!(m.encode(), b, "non-canonical input accepted: {b:?}");
        }
    }
    assert!(
        accepted > 100,
        "the test should reach the Ok path ({accepted})"
    );
}

#[test]
fn flipping_any_single_bit_never_yields_a_non_canonical_message() {
    for m in samples() {
        let bytes = m.encode();
        for i in 0..bytes.len() * 8 {
            let mut b = bytes.clone();
            b[i / 8] ^= 1 << (i % 8);
            if let Ok(m2) = Message::decode(&b) {
                assert_eq!(m2.encode(), b);
                assert_ne!(m2, m, "a flipped bit cannot decode to the same message");
            }
        }
    }
}

#[test]
fn error_messages_are_readable() {
    assert_eq!(DecodeError::Truncated.to_string(), "message is truncated");
    assert_eq!(
        DecodeError::UnknownTag(9).to_string(),
        "unknown message tag 9"
    );
    assert_eq!(
        DecodeError::TrailingBytes.to_string(),
        "trailing bytes after message"
    );
    assert_eq!(
        DecodeError::TooMany { count: 5, max: 4 }.to_string(),
        "5 items, limit 4"
    );
    assert_eq!(
        DecodeError::TooLong { len: 5, max: 4 }.to_string(),
        "item of 5 bytes, limit 4"
    );
    assert_eq!(DecodeError::BadValue("x").to_string(), "bad value: x");
}

#[test]
#[should_panic(expected = "transaction too long")]
fn encoding_an_oversize_transaction_is_a_local_bug() {
    let _ = Message::Tx(vec![0; MAX_TX + 1]).encode();
}

#[test]
#[should_panic(expected = "bad block count")]
fn encoding_a_zero_block_request_is_a_local_bug() {
    let _ = Message::GetBlocks {
        from_height: 0,
        max: 0,
    }
    .encode();
}

#[test]
#[should_panic(expected = "bad block count")]
fn encoding_an_oversize_block_request_is_a_local_bug() {
    let _ = Message::GetBlocks {
        from_height: 0,
        max: 101,
    }
    .encode();
}

#[test]
#[should_panic(expected = "too many hashes")]
fn encoding_too_many_hashes_is_a_local_bug() {
    let _ = Message::Inv(vec![[0; 32]; MAX_INV + 1]).encode();
}

#[test]
#[should_panic(expected = "too many blocks")]
fn encoding_too_many_blocks_is_a_local_bug() {
    let _ = Message::Blocks(vec![vec![]; MAX_BLOCKS + 1]).encode();
}

#[test]
#[should_panic(expected = "block too long")]
fn encoding_an_oversize_block_is_a_local_bug() {
    let _ = Message::Blocks(vec![vec![0; MAX_BLOCK + 1]]).encode();
}

#[test]
fn the_protocol_limits_are_pinned_to_literal_numbers() {
    assert_eq!(MAX_TX, 65_536);
    assert_eq!(MAX_BLOCK, 1_048_576);
    assert_eq!(MAX_INV, 1000);
    assert_eq!(MAX_BLOCKS, 100);
    // one past each limit, written as plain numbers
    let mut tx = vec![4];
    tx.extend_from_slice(&65_537u32.to_be_bytes());
    assert_eq!(
        Message::decode(&tx),
        Err(DecodeError::TooLong {
            len: 65_537,
            max: 65_536
        })
    );
    let mut blocks = vec![7, 0, 1];
    blocks.extend_from_slice(&1_048_577u32.to_be_bytes());
    assert_eq!(
        Message::decode(&blocks),
        Err(DecodeError::TooLong {
            len: 1_048_577,
            max: 1_048_576
        })
    );
    // a block of exactly the limit encodes and decodes
    let m = Message::Blocks(vec![vec![5; 1_048_576]]);
    assert_eq!(Message::decode(&m.encode()), Ok(m));
}
```

### `tests/peer.rs`

```rust
use netlab::msg::{DecodeError, Message};
use netlab::peer::*;

const ME: [u8; 8] = [1; 8];
const THEM: [u8; 8] = [2; 8];

fn cfg() -> Config {
    Config {
        chain_id: 7,
        node_id: ME,
        bucket_capacity: 5,
        refill_per_sec: 1,
        ban_score: 100,
        ping_interval_ms: 1000,
        ping_timeout_ms: 500,
        handshake_timeout_ms: 300,
    }
}

fn hello(version: u16, chain_id: u32, node_id: [u8; 8]) -> Message {
    Message::Hello {
        version,
        chain_id,
        node_id,
    }
}

fn ready() -> Session {
    let mut s = Session::new(cfg(), 0);
    assert!(s.on_message(hello(PROTOCOL_VERSION, 7, THEM), 0).is_empty());
    assert_eq!(s.state(), State::Ready);
    s
}

#[test]
fn the_session_opens_with_our_hello() {
    let s = Session::new(cfg(), 0);
    assert_eq!(s.state(), State::AwaitingHello);
    assert_eq!(
        s.start(),
        vec![Action::Send(hello(PROTOCOL_VERSION, 7, ME))]
    );
    assert_eq!(s.score(), 0);
}

#[test]
fn a_matching_hello_completes_the_handshake() {
    let _ = ready();
}

#[test]
fn a_bad_hello_ends_the_session_with_the_right_reason() {
    let cases = [
        (hello(2, 7, THEM), Reason::UnsupportedVersion(2)),
        (hello(0, 7, THEM), Reason::UnsupportedVersion(0)),
        (hello(PROTOCOL_VERSION, 8, THEM), Reason::WrongChain(8)),
        (hello(PROTOCOL_VERSION, 7, ME), Reason::SelfConnection),
    ];
    for (m, why) in cases {
        let mut s = Session::new(cfg(), 0);
        assert_eq!(
            s.on_message(m, 0),
            vec![Action::Disconnect(why.clone())],
            "{why:?}"
        );
        assert_eq!(s.state(), State::Closed);
    }
}

#[test]
fn anything_before_hello_ends_the_session() {
    for m in [
        Message::Ping(1),
        Message::Pong(1),
        Message::Tx(vec![1]),
        Message::Inv(vec![]),
    ] {
        let mut s = Session::new(cfg(), 0);
        assert_eq!(
            s.on_message(m, 0),
            vec![Action::Disconnect(Reason::UnexpectedBeforeHello)]
        );
        assert_eq!(s.state(), State::Closed);
    }
}

#[test]
fn a_second_hello_ends_the_session() {
    let mut s = ready();
    assert_eq!(
        s.on_message(hello(PROTOCOL_VERSION, 7, THEM), 1),
        vec![Action::Disconnect(Reason::DuplicateHello)]
    );
    assert_eq!(s.state(), State::Closed);
}

#[test]
fn pings_are_answered_with_the_same_nonce() {
    let mut s = ready();
    assert_eq!(
        s.on_message(Message::Ping(99), 1),
        vec![Action::Send(Message::Pong(99))]
    );
    assert_eq!(
        s.on_message(Message::Ping(0), 2),
        vec![Action::Send(Message::Pong(0))]
    );
}

#[test]
fn application_messages_are_delivered_unchanged() {
    let mut s = ready();
    for m in [
        Message::Tx(vec![1, 2]),
        Message::Inv(vec![[3; 32]]),
        Message::GetBlocks {
            from_height: 4,
            max: 5,
        },
        Message::Blocks(vec![vec![6]]),
    ] {
        assert_eq!(s.on_message(m.clone(), 1), vec![Action::Deliver(m)]);
    }
}

#[test]
fn an_unsolicited_pong_costs_score_and_enough_of_them_ban_the_peer() {
    let mut c = cfg();
    c.bucket_capacity = 1000; // keep the rate limiter out of this test
    let mut s = Session::new(c, 0);
    s.on_message(hello(PROTOCOL_VERSION, 7, THEM), 0);
    for i in 1..=9 {
        assert!(s.on_message(Message::Pong(i), 0).is_empty(), "pong {i}");
        assert_eq!(s.score(), SCORE_UNSOLICITED_PONG * i as u32);
    }
    assert_eq!(s.state(), State::Ready);
    assert_eq!(
        s.on_message(Message::Pong(10), 0),
        vec![Action::Disconnect(Reason::Misbehaving)],
        "the 10th brings the score to the limit of 100"
    );
}

#[test]
fn undecodable_input_costs_fifty_and_two_strikes_ban() {
    let mut s = ready();
    assert!(s.on_decode_error(&DecodeError::Truncated).is_empty());
    assert_eq!(s.score(), SCORE_BAD_MESSAGE);
    assert_eq!(
        s.on_decode_error(&DecodeError::UnknownTag(9)),
        vec![Action::Disconnect(Reason::Misbehaving)]
    );
    assert_eq!(s.state(), State::Closed);
    // a closed session ignores everything, including further errors
    assert!(s.on_decode_error(&DecodeError::Truncated).is_empty());
    assert!(s.on_message(Message::Ping(1), 5).is_empty());
    assert!(s.tick(1_000_000).is_empty());
}

#[test]
fn the_ban_threshold_is_inclusive() {
    let mut c = cfg();
    c.ban_score = 50;
    let mut s = Session::new(c, 0);
    s.on_message(hello(PROTOCOL_VERSION, 7, THEM), 0);
    assert_eq!(
        s.on_decode_error(&DecodeError::Truncated),
        vec![Action::Disconnect(Reason::Misbehaving)],
        "score 50 with a limit of 50"
    );
}

#[test]
fn flooding_is_rate_limited_and_eventually_bans() {
    let mut s = ready(); // the Hello used 1 of 5 tokens
    for i in 0..4 {
        assert_eq!(
            s.on_message(Message::Ping(i), 0),
            vec![Action::Send(Message::Pong(i))]
        );
    }
    // bucket empty: messages are dropped and cost 20 each; the 5th such drop reaches 100
    for i in 1..=4 {
        assert!(s.on_message(Message::Ping(9), 0).is_empty(), "drop {i}");
        assert_eq!(s.score(), 20 * i);
    }
    assert_eq!(
        s.on_message(Message::Ping(9), 0),
        vec![Action::Disconnect(Reason::Misbehaving)]
    );
}

#[test]
fn a_patient_peer_is_never_limited() {
    let mut s = ready();
    for i in 0..50u64 {
        let t = (i + 1) * 1000; // one message per second = the refill rate
        assert_eq!(
            s.on_message(Message::Ping(i), t),
            vec![Action::Send(Message::Pong(i))]
        );
    }
    assert_eq!(s.score(), 0);
}

#[test]
fn the_token_bucket_refills_exactly_at_the_configured_rate() {
    let mut b = RateLimiter::new(2, 2, 0); // 2 tokens, +2 per second
    assert!(b.allow(0));
    assert!(b.allow(0));
    assert!(!b.allow(0));
    assert!(!b.allow(499), "499 ms gives 0.998 tokens");
    assert!(b.allow(500), "500 ms gives one whole token");
    assert!(!b.allow(500));
    // refill is capped at capacity
    assert!(b.allow(1_000_000));
    assert!(b.allow(1_000_000));
    assert!(!b.allow(1_000_000));
}

#[test]
fn the_token_bucket_survives_odd_clocks_and_huge_numbers() {
    let mut b = RateLimiter::new(1, 1, 10_000);
    assert!(b.allow(10_000));
    assert!(!b.allow(5_000), "a clock that goes backwards adds nothing");
    assert!(!b.allow(10_500), "and does not reset the reference time");
    assert!(b.allow(11_000));
    let mut huge = RateLimiter::new(u32::MAX, u32::MAX, 0);
    assert!(huge.allow(u64::MAX));
    assert!(huge.allow(0));
    let mut zero = RateLimiter::new(0, 5, 0);
    assert!(!zero.allow(u64::MAX), "a zero-capacity bucket never allows");
}

#[test]
fn the_handshake_has_a_deadline() {
    let mut s = Session::new(cfg(), 100);
    assert!(
        s.tick(400).is_empty(),
        "300 ms after the start is still within the limit"
    );
    assert_eq!(
        s.tick(401),
        vec![Action::Disconnect(Reason::HandshakeTimeout)]
    );
    assert_eq!(s.state(), State::Closed);
    // a completed handshake cancels the deadline
    let mut ok = Session::new(cfg(), 0);
    ok.on_message(hello(PROTOCOL_VERSION, 7, THEM), 10);
    assert!(ok.tick(299).is_empty());
}

#[test]
fn idle_connections_are_pinged_and_must_answer() {
    let mut s = ready();
    assert!(s.tick(999).is_empty(), "not idle long enough");
    assert_eq!(s.tick(1000), vec![Action::Send(Message::Ping(1))]);
    assert!(
        s.tick(1001).is_empty(),
        "only one ping is outstanding at a time"
    );
    assert!(
        s.tick(1500).is_empty(),
        "500 ms after the ping is still within the limit"
    );
    assert_eq!(s.tick(1501), vec![Action::Disconnect(Reason::PingTimeout)]);
}

#[test]
fn the_right_pong_clears_the_ping_and_the_next_ping_has_a_new_nonce() {
    let mut s = ready();
    assert_eq!(s.tick(1000), vec![Action::Send(Message::Ping(1))]);
    assert!(s.on_message(Message::Pong(1), 1100).is_empty());
    assert_eq!(s.score(), 0);
    assert!(
        s.tick(2099).is_empty(),
        "activity at 1100 restarts the idle clock"
    );
    assert_eq!(s.tick(2100), vec![Action::Send(Message::Ping(2))]);
}

#[test]
fn a_pong_with_the_wrong_nonce_does_not_count() {
    let mut s = ready();
    s.tick(1000);
    assert!(s.on_message(Message::Pong(7), 1100).is_empty());
    assert_eq!(s.score(), SCORE_UNSOLICITED_PONG);
    assert_eq!(s.tick(1501), vec![Action::Disconnect(Reason::PingTimeout)]);
}

#[test]
fn any_message_counts_as_activity() {
    let mut s = ready();
    s.on_message(Message::Tx(vec![1]), 900);
    assert!(s.tick(1000).is_empty());
    assert!(s.tick(1899).is_empty());
    assert_eq!(s.tick(1900), vec![Action::Send(Message::Ping(1))]);
}
```

### `tests/det.rs`

```rust
use std::collections::HashMap;

use netlab::det::*;

fn credit(acct: u64, amount: u64) -> Cmd {
    Cmd::Credit { acct, amount }
}
fn transfer(from: u64, to: u64, amount: u64) -> Cmd {
    Cmd::Transfer { from, to, amount }
}

fn hex(h: &[u8]) -> String {
    h.iter().map(|b| format!("{b:02x}")).collect()
}

// -------------------------------------------------------- state machine

#[test]
fn commands_change_balances() {
    let mut m = Machine::new();
    m.apply(credit(1, 100)).unwrap();
    m.apply(transfer(1, 2, 30)).unwrap();
    assert_eq!((m.balance(1), m.balance(2), m.balance(3)), (70, 30, 0));
    assert_eq!(m.total(), 100);
    assert_eq!(m.height(), 0, "single commands do not advance the height");
}

#[test]
fn invalid_commands_fail_without_changing_anything() {
    let mut m = Machine::new();
    m.apply(credit(1, 10)).unwrap();
    let before = m.clone();
    assert_eq!(m.apply(credit(1, 0)), Err(CmdError::ZeroAmount));
    assert_eq!(m.apply(transfer(1, 2, 0)), Err(CmdError::ZeroAmount));
    assert_eq!(m.apply(transfer(1, 2, 11)), Err(CmdError::Insufficient));
    assert_eq!(
        m.apply(transfer(9, 2, 1)),
        Err(CmdError::Insufficient),
        "unknown source"
    );
    assert_eq!(m, before);
}

#[test]
fn balances_cannot_overflow() {
    let mut m = Machine::new();
    m.apply(credit(1, u64::MAX)).unwrap();
    assert_eq!(m.apply(credit(1, 1)), Err(CmdError::Overflow));
    m.apply(credit(2, 1)).unwrap();
    assert_eq!(m.apply(transfer(2, 1, 1)), Err(CmdError::Overflow));
    assert_eq!((m.balance(1), m.balance(2)), (u64::MAX, 1), "nothing moved");
}

#[test]
fn a_transfer_to_yourself_changes_nothing_but_still_needs_the_funds() {
    let mut m = Machine::new();
    m.apply(credit(1, 5)).unwrap();
    m.apply(transfer(1, 1, 5)).unwrap();
    assert_eq!(m.balance(1), 5);
    assert_eq!(m.apply(transfer(1, 1, 6)), Err(CmdError::Insufficient));
}

#[test]
fn a_block_is_all_or_nothing() {
    let mut m = Machine::new();
    m.apply_block(&[credit(1, 100), transfer(1, 2, 30)])
        .unwrap();
    assert_eq!(m.height(), 1);
    let before = (m.clone(), m.state_hash());
    let err = m.apply_block(&[
        transfer(1, 2, 10),
        transfer(1, 3, 10),
        transfer(2, 9, 999),
        credit(5, 5),
    ]);
    assert_eq!(
        err,
        Err((2, CmdError::Insufficient)),
        "reports the index of the failing command"
    );
    assert_eq!(
        (m.clone(), m.state_hash()),
        before,
        "the first two commands were rolled back"
    );
    assert_eq!(m.apply_block(&[]), Ok(()));
    assert_eq!(m.height(), 2, "an empty block still advances the height");
}

#[test]
fn order_matters_for_commands_that_do_not_commute() {
    let a = [credit(1, 10), transfer(1, 2, 10)];
    let b = [transfer(1, 2, 10), credit(1, 10)];
    let mut ma = Machine::new();
    assert!(ma.apply_block(&a).is_ok());
    let mut mb = Machine::new();
    assert!(
        mb.apply_block(&b).is_err(),
        "the transfer comes before the money exists"
    );
}

#[test]
fn replicas_that_apply_the_same_log_agree_on_every_hash() {
    let log: Vec<Vec<Cmd>> = (0..50u64)
        .map(|i| {
            vec![
                credit(i % 7, 10 + i),
                transfer(i % 7, (i + 3) % 7, 5),
                credit(100 + i, 1),
            ]
        })
        .collect();
    let mut x = Machine::new();
    let mut y = Machine::new();
    for block in &log {
        x.apply_block(block).unwrap();
        y.apply_block(block).unwrap();
        assert_eq!(x.state_hash(), y.state_hash());
    }
    assert_eq!(x.height(), 50);
    // a replica that restarts from a snapshot halfway converges to the same state
    let mut z = Machine::new();
    for block in &log[..25] {
        z.apply_block(block).unwrap();
    }
    let mut z = Machine::restore(&z.snapshot()).expect("valid snapshot");
    for block in &log[25..] {
        z.apply_block(block).unwrap();
    }
    assert_eq!(z, x);
    assert_eq!(z.state_hash(), x.state_hash());
}

#[test]
fn state_hashes_match_independent_values() {
    // Computed with a separate Python implementation of the snapshot layout.
    assert_eq!(
        hex(&Machine::new().state_hash()),
        "f03a3748500911156632afe2c0de57f46a1dfbc4c1d3928f69b2397470a649b6"
    );
    let mut m = Machine::new();
    m.apply_block(&[credit(1, 100), credit(2, 50), transfer(1, 2, 30)])
        .unwrap();
    assert_eq!(m.snapshot().len(), 68);
    assert_eq!(
        hex(&m.state_hash()),
        "1f9734347372a03edf92e2fc8647c6ae6cc86588f4185d103788e91ca6c6f277"
    );
}

#[test]
fn the_hash_covers_height_and_every_balance() {
    let mut a = Machine::new();
    a.apply_block(&[credit(1, 10)]).unwrap();
    let mut b = Machine::new();
    b.apply(credit(1, 10)).unwrap();
    assert_ne!(
        a.state_hash(),
        b.state_hash(),
        "same balances, different height"
    );
    let mut c = Machine::new();
    c.apply_block(&[credit(1, 11)]).unwrap();
    assert_ne!(a.state_hash(), c.state_hash());
    let mut d = Machine::new();
    d.apply_block(&[credit(2, 10)]).unwrap();
    assert_ne!(a.state_hash(), d.state_hash());
}

#[test]
fn snapshots_round_trip_exactly() {
    let mut m = Machine::new();
    for i in 0..20 {
        m.apply_block(&[credit(i * 7 % 13, i + 1)]).unwrap();
    }
    let s = m.snapshot();
    let back = Machine::restore(&s).unwrap();
    assert_eq!(back, m);
    assert_eq!(back.snapshot(), s);
    assert_eq!(
        Machine::restore(&Machine::new().snapshot()),
        Some(Machine::new())
    );
}

#[test]
fn restore_refuses_everything_that_snapshot_never_produces() {
    let mut m = Machine::new();
    m.apply_block(&[credit(1, 5), credit(2, 6)]).unwrap();
    let good = m.snapshot();
    assert!(Machine::restore(&good).is_some());
    // truncated at every length
    for cut in 0..good.len() {
        assert!(Machine::restore(&good[..cut]).is_none(), "cut {cut}");
    }
    // trailing bytes
    let mut t = good.clone();
    t.push(0);
    assert!(Machine::restore(&t).is_none());
    // wrong domain label
    let mut d = good.clone();
    d[8] ^= 1;
    assert!(Machine::restore(&d).is_none());
    let mut d = good.clone();
    d[7] ^= 1; // the label length
    assert!(Machine::restore(&d).is_none());
    let head = 8 + 12;
    // a count that disagrees with the body (including one that overflows when multiplied)
    // (2^60 + 2) * 16 wraps around to 32, the real body length
    for count in [0u64, 1, 3, u64::MAX, 1 << 60, (1 << 60) + 2] {
        let mut c = good.clone();
        c[head + 8..head + 16].copy_from_slice(&count.to_be_bytes());
        assert!(Machine::restore(&c).is_none(), "count {count}");
    }
    // duplicate or unsorted accounts, and a zero balance
    let mut u = good.clone();
    u[head + 16..head + 24].copy_from_slice(&5u64.to_be_bytes());
    assert!(
        Machine::restore(&u).is_none(),
        "keys 5, 2 are not ascending"
    );
    let mut dup = good.clone();
    dup[head + 32..head + 40].copy_from_slice(&1u64.to_be_bytes());
    assert!(Machine::restore(&dup).is_none(), "duplicate key");
    let mut z = good.clone();
    z[head + 24..head + 32].copy_from_slice(&0u64.to_be_bytes());
    assert!(Machine::restore(&z).is_none(), "zero balance");
}

// ------------------------------------------------- sources of non-determinism

#[test]
fn hashing_a_hashmap_in_iteration_order_is_not_deterministic() {
    let entries: Vec<(u64, u64)> = (0..64).map(|i| (i, i * i)).collect();
    let mut flawed = std::collections::HashSet::new();
    let mut fixed = std::collections::HashSet::new();
    for _ in 0..40 {
        let map: HashMap<u64, u64> = entries.iter().copied().collect();
        flawed.insert(hash_in_map_order(&map));
        fixed.insert(hash_in_key_order(&map));
    }
    assert!(
        flawed.len() > 1,
        "equal maps hashed to {} different values",
        flawed.len()
    );
    assert_eq!(fixed.len(), 1, "key order gives one answer");
}

#[test]
fn the_ordered_hash_ignores_insertion_order_but_not_content() {
    let a: HashMap<u64, u64> = [(1, 2), (3, 4)].into_iter().collect();
    let b: HashMap<u64, u64> = [(3, 4), (1, 2)].into_iter().collect();
    let c: HashMap<u64, u64> = [(1, 2), (3, 5)].into_iter().collect();
    assert_eq!(hash_in_key_order(&a), hash_in_key_order(&b));
    assert_ne!(hash_in_key_order(&a), hash_in_key_order(&c));
    assert_ne!(hash_in_key_order(&HashMap::new()), hash_in_key_order(&a));
}

#[test]
fn floating_point_addition_depends_on_the_order() {
    assert_eq!(sum_f64(&[0.1, 0.2, 0.3]), 0.6000000000000001);
    assert_eq!(sum_f64(&[0.3, 0.2, 0.1]), 0.6);
    assert_ne!(sum_f64(&[0.1, 0.2, 0.3]), sum_f64(&[0.3, 0.2, 0.1]));
    assert_eq!(sum_f64(&[]), 0.0);
}

#[test]
fn integer_sums_are_exact_in_any_order_and_report_overflow() {
    let xs = [100, 200, 300, -50];
    let mut ys = xs;
    ys.reverse();
    assert_eq!(sum_fixed(&xs), Some(550));
    assert_eq!(sum_fixed(&xs), sum_fixed(&ys));
    assert_eq!(sum_fixed(&[i64::MAX, 1]), None);
    assert_eq!(
        sum_fixed(&[i64::MAX, 1, -1]),
        None,
        "an overflow is not forgiven by a later negative"
    );
    assert_eq!(sum_fixed(&[]), Some(0));
}

// ------------------------------------------------------------------- the VM

use Op::*;

#[test]
fn arithmetic_runs_on_a_stack() {
    // 2 + 3 * 4
    let p = [Push(2), Push(3), Push(4), Mul, Add];
    let o = run(&p, 100).unwrap();
    assert_eq!(o.top, Some(14));
    assert_eq!((o.steps, o.fuel_used), (5, 1 + 1 + 1 + 3 + 1));
    assert_eq!(
        run(&[Push(10), Push(3), Sub], 10).unwrap().top,
        Some(7),
        "a - b with b on top"
    );
    assert_eq!(
        run(&[], 0).unwrap(),
        Outcome {
            top: None,
            fuel_used: 0,
            steps: 0
        }
    );
}

#[test]
fn division_truncates_toward_zero_and_traps_on_zero_and_overflow() {
    assert_eq!(run(&[Push(-7), Push(2), Div], 10).unwrap().top, Some(-3));
    assert_eq!(run(&[Push(7), Push(-2), Div], 10).unwrap().top, Some(-3));
    assert_eq!(run(&[Push(1), Push(0), Div], 10), Err(Trap::DivideByZero));
    assert_eq!(
        run(&[Push(i64::MIN), Push(-1), Div], 10),
        Err(Trap::Overflow)
    );
    assert_eq!(
        run(&[Push(5), Push(0), Div], 4),
        Err(Trap::OutOfFuel),
        "the division costs 5; fuel is checked first"
    );
}

#[test]
fn arithmetic_overflow_is_a_trap_never_a_wrap() {
    assert_eq!(
        run(&[Push(i64::MAX), Push(1), Add], 10),
        Err(Trap::Overflow)
    );
    assert_eq!(
        run(&[Push(i64::MIN), Push(1), Sub], 10),
        Err(Trap::Overflow)
    );
    assert_eq!(
        run(&[Push(i64::MAX), Push(2), Mul], 10),
        Err(Trap::Overflow)
    );
    assert_eq!(
        run(&[Push(i64::MAX), Push(1), Mul], 10).unwrap().top,
        Some(i64::MAX)
    );
}

#[test]
fn fuel_is_exact() {
    let p = [Push(1), Push(2), Add, Dup, Mul]; // cost 1+1+1+1+3 = 7
    assert_eq!(run(&p, 7).unwrap().fuel_used, 7);
    assert_eq!(run(&p, 6), Err(Trap::OutOfFuel));
    assert_eq!(run(&p, 0), Err(Trap::OutOfFuel));
    assert_eq!(run(&p, u64::MAX).unwrap().fuel_used, 7);
}

#[test]
fn an_infinite_loop_is_stopped_by_the_fuel_limit() {
    for fuel in [0u64, 1, 10, 1000] {
        assert_eq!(run(&[Jmp(0)], fuel), Err(Trap::OutOfFuel), "fuel {fuel}");
    }
}

#[test]
fn stack_operations() {
    assert_eq!(run(&[Push(1), Push(2), Swap], 10).unwrap().top, Some(1));
    assert_eq!(run(&[Push(1), Push(2), Pop], 10).unwrap().top, Some(1));
    assert_eq!(run(&[Push(3), Dup, Add], 10).unwrap().top, Some(6));
    for p in [
        vec![Pop],
        vec![Add],
        vec![Sub],
        vec![Mul],
        vec![Div],
        vec![Dup],
        vec![Swap],
        vec![Jnz(0)],
    ] {
        assert_eq!(run(&p, 10), Err(Trap::StackUnderflow), "{p:?}");
    }
    assert_eq!(run(&[Push(1), Add], 10), Err(Trap::StackUnderflow));
    assert_eq!(run(&[Push(1), Swap], 10), Err(Trap::StackUnderflow));
}

#[test]
fn the_stack_depth_is_bounded() {
    let ok: Vec<Op> = (0..MAX_STACK).map(|_| Push(1)).collect();
    assert_eq!(run(&ok, 1000).unwrap().steps, MAX_STACK as u64);
    let too_many: Vec<Op> = (0..=MAX_STACK).map(|_| Push(1)).collect();
    assert_eq!(run(&too_many, 1000), Err(Trap::StackOverflow));
    let mut dup = vec![Push(1); MAX_STACK];
    dup.push(Dup);
    assert_eq!(run(&dup, 1000), Err(Trap::StackOverflow));
}

#[test]
fn jumps_and_halting() {
    assert_eq!(
        run(&[Push(1), Jmp(3), Push(99), Push(2)], 10).unwrap().top,
        Some(2)
    );
    assert_eq!(run(&[Jmp(1)], 10).unwrap().steps, 1, "jumping to len halts");
    assert_eq!(run(&[Jmp(2)], 10), Err(Trap::BadJump(2)));
    assert_eq!(run(&[Push(1), Jnz(7)], 10), Err(Trap::BadJump(7)));
    assert_eq!(
        run(&[Push(0), Jnz(7), Push(5)], 10).unwrap().top,
        Some(5),
        "an untaken jump is never checked"
    );
    assert_eq!(run(&[Push(1), Halt, Push(2)], 10).unwrap().top, Some(1));
    assert_eq!(run(&[Push(1), Halt, Push(2)], 10).unwrap().steps, 2);
}

#[test]
fn a_counting_loop_costs_exactly_four_instructions_per_iteration() {
    // push n; loop: push 1, sub, dup, jnz loop
    for n in [1i64, 2, 10, 1000] {
        let p = [Push(n), Push(1), Sub, Dup, Jnz(1), Halt];
        let o = run(&p, 100_000).unwrap();
        assert_eq!(o.top, Some(0));
        assert_eq!(o.steps, 4 * n as u64 + 2, "n = {n}");
        assert_eq!(o.fuel_used, o.steps);
        assert_eq!(
            run(&p, o.fuel_used - 1),
            Err(Trap::OutOfFuel),
            "one unit short"
        );
        assert_eq!(run(&p, o.fuel_used), Ok(o));
    }
}

#[test]
fn running_twice_gives_identical_results() {
    let p = [Push(1000), Push(1), Sub, Dup, Jnz(1), Push(7), Push(6), Mul];
    let first = run(&p, 50_000);
    for _ in 0..20 {
        assert_eq!(run(&p, 50_000), first);
    }
    assert_eq!(first.unwrap().top, Some(42));
}

#[test]
fn division_costs_five_and_multiplication_three() {
    let o = run(&[Push(6), Push(3), Div], 100).unwrap();
    assert_eq!((o.top, o.fuel_used), (Some(2), 1 + 1 + 5));
    assert_eq!(run(&[Push(6), Push(3), Div], 6), Err(Trap::OutOfFuel));
    assert_eq!(run(&[Push(6), Push(3), Mul], 4), Err(Trap::OutOfFuel));
    assert_eq!(run(&[Push(6), Push(3), Mul], 5).unwrap().top, Some(18));
}

#[test]
fn a_taken_conditional_jump_may_target_the_end_of_the_program() {
    assert_eq!(run(&[Push(1), Jnz(2)], 10).unwrap().top, None);
    assert_eq!(run(&[Push(1), Jnz(3)], 10), Err(Trap::BadJump(3)));
}
```

### `tests/sim.rs`

```rust
use netlab::sim::*;

const CLEAN: Link = Link {
    drop_per_mille: 0,
    dup_per_mille: 0,
    max_delay: 5,
};

#[test]
fn the_generator_matches_an_independent_implementation() {
    // Values from a separate Python implementation of xorshift64*.
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
    assert_eq!(
        Rng::new(0).next_u64(),
        973819730272012410,
        "seed 0 is replaced"
    );
    let mut r = Rng::new(42);
    let got: Vec<u64> = (0..8).map(|_| r.below(10)).collect();
    assert_eq!(got, [0, 8, 6, 5, 8, 5, 2, 3]);
}

#[test]
fn below_stays_in_range_and_is_roughly_uniform() {
    let mut r = Rng::new(99);
    let mut counts = [0u32; 10];
    for _ in 0..10_000 {
        let v = r.below(10) as usize;
        counts[v] += 1;
    }
    for (i, c) in counts.iter().enumerate() {
        assert!((800..1200).contains(c), "bucket {i}: {c}");
    }
    assert_eq!(Rng::new(5).below(1), 0);
    for _ in 0..100 {
        assert!(r.below(3) < 3);
    }
}

#[test]
#[should_panic(expected = "below(0)")]
fn below_zero_is_meaningless() {
    let _ = Rng::new(1).below(0);
}

#[test]
fn the_same_seed_gives_the_same_stream() {
    let a: Vec<u64> = {
        let mut r = Rng::new(7);
        (0..20).map(|_| r.next_u64()).collect()
    };
    let b: Vec<u64> = {
        let mut r = Rng::new(7);
        (0..20).map(|_| r.next_u64()).collect()
    };
    let c: Vec<u64> = {
        let mut r = Rng::new(8);
        (0..20).map(|_| r.next_u64()).collect()
    };
    assert_eq!(a, b);
    assert_ne!(a, c);
}

#[test]
fn the_constructors_build_the_expected_graphs() {
    assert_eq!((Network::ring(6).len(), Network::ring(6).edges()), (6, 6));
    assert_eq!(Network::line(6).edges(), 5);
    assert_eq!(Network::complete(6).edges(), 15);
    let n = Network::from_edges(4, &[(0, 1), (1, 0), (2, 2), (1, 2)]);
    assert_eq!(n.edges(), 2, "duplicates and self-loops are ignored");
    assert!(Network::from_edges(0, &[]).is_empty());
    assert!(!n.is_empty());
}

#[test]
#[should_panic(expected = "edge endpoint out of range")]
fn edges_must_name_real_nodes() {
    let _ = Network::from_edges(3, &[(0, 3)]);
}

#[test]
fn a_clean_flood_reaches_everyone_in_a_connected_network() {
    for net in [Network::ring(10), Network::line(10), Network::complete(10)] {
        for origin in [0, 4, 9] {
            let r = flood(&net, CLEAN, 1, origin);
            assert_eq!(r.reached(), 10);
            assert_eq!((r.dropped, r.duplicated), (0, 0));
            assert_eq!(r.events, r.sent, "every message arrives exactly once");
        }
    }
}

#[test]
fn message_counts_follow_from_the_forwarding_rule() {
    // ring: the origin sends 2, every other node forwards once => n + 1
    for n in [3usize, 4, 10, 25] {
        assert_eq!(
            flood(&Network::ring(n), CLEAN, 3, 0).sent,
            n as u64 + 1,
            "ring {n}"
        );
    }
    // complete graph: (n - 1) from the origin, each of the others forwards to n - 2 => (n - 1)^2
    for n in [3usize, 5, 8] {
        let want = ((n - 1) * (n - 1)) as u64;
        assert_eq!(
            flood(&Network::complete(n), CLEAN, 3, 0).sent,
            want,
            "complete {n}"
        );
    }
    // line from one end: each node forwards to the next, n - 1 messages
    assert_eq!(flood(&Network::line(7), CLEAN, 3, 0).sent, 6);
    // line from the middle: both directions
    assert_eq!(flood(&Network::line(7), CLEAN, 3, 3).sent, 6);
}

#[test]
fn with_unit_delay_receipt_time_is_the_hop_distance() {
    let link = Link {
        max_delay: 1,
        ..CLEAN
    };
    let r = flood(&Network::ring(8), link, 1, 0);
    let want: Vec<Option<u64>> = (0..8u64).map(|i| Some(i.min(8 - i))).collect();
    assert_eq!(r.first_receipt, want);
    // the farthest node forwards once more; those copies arrive one tick later
    assert_eq!(r.finished_at, 5);
    let r = flood(&Network::line(5), link, 9, 2);
    assert_eq!(
        r.first_receipt,
        [Some(2), Some(1), Some(0), Some(1), Some(2)]
    );
}

#[test]
fn delays_stay_within_the_configured_bounds() {
    let net = Network::line(20);
    let r = flood(
        &net,
        Link {
            max_delay: 7,
            ..CLEAN
        },
        5,
        0,
    );
    let mut prev = 0;
    for t in r.first_receipt.iter().skip(1) {
        let t = t.expect("reached");
        assert!(t > prev && t - prev <= 7, "step {prev} -> {t}");
        prev = t;
    }
    assert_eq!(r.finished_at, prev);
}

#[test]
fn a_partitioned_network_reaches_only_the_origin_component() {
    let net = Network::from_edges(6, &[(0, 1), (1, 2), (3, 4), (4, 5)]);
    let r = flood(&net, CLEAN, 1, 0);
    assert_eq!(r.reached(), 3);
    assert!(r.first_receipt[3].is_none() && r.first_receipt[5].is_none());
    let alone = flood(&Network::from_edges(1, &[]), CLEAN, 1, 0);
    assert_eq!((alone.reached(), alone.sent, alone.events), (1, 0, 0));
}

#[test]
fn a_run_is_a_pure_function_of_its_inputs() {
    let lossy = Link {
        drop_per_mille: 200,
        dup_per_mille: 200,
        max_delay: 9,
    };
    let net = Network::ring(12);
    for seed in 0..20 {
        assert_eq!(
            flood(&net, lossy, seed, 0),
            flood(&net, lossy, seed, 0),
            "seed {seed}"
        );
    }
    let distinct: std::collections::HashSet<_> = (0..20)
        .map(|s| format!("{:?}", flood(&net, lossy, s, 0)))
        .collect();
    assert!(
        distinct.len() > 10,
        "different seeds give different histories ({})",
        distinct.len()
    );
}

#[test]
fn reports_match_an_independent_python_simulation() {
    let ring6 = Link {
        drop_per_mille: 100,
        dup_per_mille: 100,
        max_delay: 5,
    };
    let r = flood(&Network::ring(6), ring6, 7, 0);
    assert_eq!(
        r.first_receipt,
        [Some(0), Some(4), Some(5), Some(8), Some(9), Some(12)]
    );
    assert_eq!(
        (r.sent, r.dropped, r.duplicated, r.events, r.finished_at),
        (7, 1, 0, 6, 16)
    );
    let k5 = Link {
        drop_per_mille: 300,
        dup_per_mille: 0,
        max_delay: 4,
    };
    let r = flood(&Network::complete(5), k5, 11, 2);
    assert_eq!(
        r.first_receipt,
        [Some(5), Some(1), Some(0), Some(2), Some(4)]
    );
    assert_eq!(
        (r.sent, r.dropped, r.duplicated, r.events, r.finished_at),
        (16, 7, 0, 9, 8)
    );
}

#[test]
fn losing_everything_leaves_only_the_origin() {
    let r = flood(
        &Network::ring(6),
        Link {
            drop_per_mille: 1000,
            ..CLEAN
        },
        3,
        0,
    );
    assert_eq!((r.reached(), r.sent, r.dropped, r.events), (1, 2, 2, 0));
}

#[test]
fn duplication_doubles_the_arrivals_but_not_the_receipts() {
    let link = Link {
        dup_per_mille: 1000,
        ..CLEAN
    };
    let r = flood(&Network::complete(6), link, 4, 0);
    assert_eq!(r.duplicated, r.sent);
    assert_eq!(r.events, 2 * r.sent);
    assert_eq!(r.reached(), 6);
}

#[test]
fn gossip_without_retransmission_is_not_reliable_under_loss() {
    // Over 200 fixed seeds (so these counts are exact and repeatable):
    let lossy = Link {
        drop_per_mille: 300,
        ..CLEAN
    };
    let full = |net: &Network| {
        (0..200u64)
            .filter(|s| flood(net, lossy, *s, 0).reached() == net.len())
            .count()
    };
    let ring = full(&Network::ring(10));
    let complete = full(&Network::complete(10));
    assert_eq!((ring, complete), (25, 200));
    assert!(complete > ring, "more paths mean better delivery");
    assert!(ring < 200, "a ring loses the item in some runs");
}

#[test]
#[should_panic(expected = "origin out of range")]
fn the_origin_must_exist() {
    let _ = flood(&Network::ring(3), CLEAN, 1, 3);
}

#[test]
#[should_panic(expected = "max_delay must be at least 1")]
fn the_delay_range_cannot_be_empty() {
    let _ = flood(
        &Network::ring(3),
        Link {
            max_delay: 0,
            ..CLEAN
        },
        1,
        0,
    );
}

/// First seed in `0..` whose first `below(1000)` draw equals `want`.
fn seed_with_first_draw(want: u64) -> u64 {
    (1u64..)
        .find(|s| Rng::new(*s).below(1000) == want)
        .expect("exists")
}

/// First seed whose first draw is anything and whose third draw (the duplicate
/// decision of the first message) equals `want`.
fn seed_with_third_draw(want: u64) -> u64 {
    (1u64..)
        .find(|s| {
            let mut r = Rng::new(*s);
            r.below(1000);
            r.below(5);
            r.below(1000) == want
        })
        .expect("exists")
}

#[test]
fn per_mille_probabilities_use_a_strict_comparison() {
    let line = Network::line(2);
    // drop: the first draw decides the fate of the origin's only message
    let s = seed_with_first_draw(0);
    let none = Link {
        drop_per_mille: 0,
        dup_per_mille: 0,
        max_delay: 5,
    };
    assert_eq!(
        flood(&line, none, s, 0).dropped,
        0,
        "a draw of 0 is not below 0"
    );
    let one = Link {
        drop_per_mille: 1,
        ..none
    };
    assert_eq!(flood(&line, one, s, 0).dropped, 1, "a draw of 0 is below 1");
    let s = seed_with_first_draw(7);
    assert_eq!(
        flood(
            &line,
            Link {
                drop_per_mille: 7,
                ..none
            },
            s,
            0
        )
        .dropped,
        0
    );
    assert_eq!(
        flood(
            &line,
            Link {
                drop_per_mille: 8,
                ..none
            },
            s,
            0
        )
        .dropped,
        1
    );
    // duplication: the third draw
    let s = seed_with_third_draw(0);
    assert_eq!(flood(&line, none, s, 0).duplicated, 0);
    assert_eq!(
        flood(
            &line,
            Link {
                dup_per_mille: 1,
                ..none
            },
            s,
            0
        )
        .duplicated,
        1
    );
    let s = seed_with_third_draw(9);
    assert_eq!(
        flood(
            &line,
            Link {
                dup_per_mille: 9,
                ..none
            },
            s,
            0
        )
        .duplicated,
        0
    );
    assert_eq!(
        flood(
            &line,
            Link {
                dup_per_mille: 10,
                ..none
            },
            s,
            0
        )
        .duplicated,
        1
    );
}

#[test]
fn duplicate_copies_get_their_own_delay() {
    // Checked against the independent Python simulation.
    let link = Link {
        drop_per_mille: 0,
        dup_per_mille: 600,
        max_delay: 9,
    };
    let r = flood(&Network::ring(6), link, 3, 0);
    assert_eq!(
        r.first_receipt,
        [Some(0), Some(2), Some(4), Some(5), Some(9), Some(1)]
    );
    assert_eq!(
        (r.sent, r.dropped, r.duplicated, r.events, r.finished_at),
        (7, 0, 4, 11, 12)
    );
}
```

### `tests/tcp.rs`

```rust
//! Real loopback TCP: the same framing and messages over an actual socket.

use std::io::Write;
use std::net::{TcpListener, TcpStream};
use std::thread::{self, JoinHandle};
use std::time::{Duration, Instant};

use netlab::frame::{FrameError, read_frame, write_frame};
use netlab::msg::Message;
use netlab::peer::{Action, Config, PROTOCOL_VERSION, Session, State};

const LIMIT: usize = 1 << 20;
const PATIENCE: Duration = Duration::from_secs(5);

fn listener() -> (TcpListener, std::net::SocketAddr) {
    let l = TcpListener::bind("127.0.0.1:0").expect("bind loopback");
    let a = l.local_addr().unwrap();
    (l, a)
}

fn connect(addr: std::net::SocketAddr) -> TcpStream {
    let s = TcpStream::connect(addr).expect("connect");
    s.set_read_timeout(Some(PATIENCE)).unwrap();
    s.set_write_timeout(Some(PATIENCE)).unwrap();
    s.set_nodelay(true).unwrap();
    s
}

fn cfg(node: u8) -> Config {
    Config {
        chain_id: 7,
        node_id: [node; 8],
        bucket_capacity: 100,
        refill_per_sec: 100,
        ban_score: 100,
        ping_interval_ms: 10_000,
        ping_timeout_ms: 10_000,
        handshake_timeout_ms: 10_000,
    }
}

/// A server that runs a `Session` over one connection and returns what it delivered.
fn serve_one(l: TcpListener) -> JoinHandle<Vec<Message>> {
    thread::spawn(move || {
        let (mut sock, _) = l.accept().unwrap();
        sock.set_read_timeout(Some(PATIENCE)).unwrap();
        let start = Instant::now();
        let mut session = Session::new(cfg(1), 0);
        let mut delivered = Vec::new();
        for a in session.start() {
            if let Action::Send(m) = a {
                write_frame(&mut sock, &m.encode(), LIMIT).unwrap();
            }
        }
        loop {
            let now = start.elapsed().as_millis() as u64;
            let actions = match read_frame(&mut sock, LIMIT) {
                Ok(bytes) => match Message::decode(&bytes) {
                    Ok(m) => session.on_message(m, now),
                    Err(e) => session.on_decode_error(&e),
                },
                Err(FrameError::Closed) => return delivered,
                Err(e) => panic!("server read failed: {e}"),
            };
            for a in actions {
                match a {
                    Action::Send(m) => write_frame(&mut sock, &m.encode(), LIMIT).unwrap(),
                    Action::Deliver(m) => delivered.push(m),
                    Action::Disconnect(_) => return delivered,
                }
            }
        }
    })
}

fn send(s: &mut TcpStream, m: &Message) {
    write_frame(s, &m.encode(), LIMIT).unwrap();
}

fn recv(s: &mut TcpStream) -> Message {
    Message::decode(&read_frame(s, LIMIT).unwrap()).unwrap()
}

fn hello() -> Message {
    Message::Hello {
        version: PROTOCOL_VERSION,
        chain_id: 7,
        node_id: [2; 8],
    }
}

#[test]
fn a_full_exchange_over_a_real_socket() {
    let (l, addr) = listener();
    let server = serve_one(l);
    let mut c = connect(addr);
    assert_eq!(
        recv(&mut c),
        Message::Hello {
            version: PROTOCOL_VERSION,
            chain_id: 7,
            node_id: [1; 8]
        }
    );
    send(&mut c, &hello());
    send(&mut c, &Message::Ping(5));
    assert_eq!(recv(&mut c), Message::Pong(5));
    send(&mut c, &Message::Tx(vec![1, 2, 3]));
    send(&mut c, &Message::Inv(vec![[9; 32]]));
    drop(c);
    assert_eq!(
        server.join().unwrap(),
        vec![Message::Tx(vec![1, 2, 3]), Message::Inv(vec![[9; 32]])]
    );
}

#[test]
fn two_frames_in_one_write_are_still_two_messages() {
    let (l, addr) = listener();
    let server = serve_one(l);
    let mut c = connect(addr);
    recv(&mut c);
    let mut both = Vec::new();
    for m in [hello(), Message::Tx(vec![7]), Message::Tx(vec![8])] {
        write_frame(&mut both, &m.encode(), LIMIT).unwrap();
    }
    c.write_all(&both).unwrap();
    drop(c);
    assert_eq!(
        server.join().unwrap(),
        vec![Message::Tx(vec![7]), Message::Tx(vec![8])]
    );
}

#[test]
fn a_message_sent_one_byte_at_a_time_is_reassembled() {
    let (l, addr) = listener();
    let server = serve_one(l);
    let mut c = connect(addr);
    recv(&mut c);
    send(&mut c, &hello());
    let mut bytes = Vec::new();
    write_frame(&mut bytes, &Message::Tx(vec![0xaa; 50]).encode(), LIMIT).unwrap();
    for b in &bytes {
        c.write_all(std::slice::from_ref(b)).unwrap();
        c.flush().unwrap();
    }
    drop(c);
    assert_eq!(server.join().unwrap(), vec![Message::Tx(vec![0xaa; 50])]);
}

#[test]
fn an_announced_four_gigabyte_frame_is_refused_immediately() {
    let (l, addr) = listener();
    let t = thread::spawn(move || {
        let (mut sock, _) = l.accept().unwrap();
        sock.set_read_timeout(Some(PATIENCE)).unwrap();
        let start = Instant::now();
        let r = read_frame(&mut sock, LIMIT);
        (r, start.elapsed())
    });
    let mut c = connect(addr);
    c.write_all(&u32::MAX.to_be_bytes()).unwrap(); // the header only; the body never comes
    let (r, took) = t.join().unwrap();
    assert!(
        matches!(r, Err(FrameError::TooLarge { len, .. }) if len == u32::MAX as u64),
        "{r:?}"
    );
    assert!(took < Duration::from_millis(1500), "took {took:?}");
    drop(c);
}

#[test]
fn a_silent_peer_hits_the_read_timeout() {
    let (l, addr) = listener();
    let t = thread::spawn(move || {
        let (mut sock, _) = l.accept().unwrap();
        sock.set_read_timeout(Some(Duration::from_millis(200)))
            .unwrap();
        let start = Instant::now();
        let r = read_frame(&mut sock, LIMIT);
        (r, start.elapsed())
    });
    let c = connect(addr); // connects, says nothing
    let (r, took) = t.join().unwrap();
    let err = r.unwrap_err();
    assert!(err.is_timeout(), "{err:?}");
    assert!(
        took >= Duration::from_millis(150) && took < Duration::from_secs(3),
        "took {took:?}"
    );
    drop(c);
}

#[test]
fn a_peer_that_vanishes_mid_frame_is_truncated_not_closed() {
    let (l, addr) = listener();
    let t = thread::spawn(move || {
        let (mut sock, _) = l.accept().unwrap();
        sock.set_read_timeout(Some(PATIENCE)).unwrap();
        read_frame(&mut sock, LIMIT)
    });
    let mut c = connect(addr);
    c.write_all(&[0, 0, 0, 10, 1, 2, 3]).unwrap(); // promises 10 bytes, sends 3
    drop(c);
    assert!(matches!(t.join().unwrap(), Err(FrameError::Truncated)));
}

#[test]
fn a_misbehaving_peer_is_dropped_by_the_session() {
    let (l, addr) = listener();
    let server = serve_one(l);
    let mut c = connect(addr);
    recv(&mut c);
    send(&mut c, &Message::Ping(1)); // not a Hello
    // the server closes the connection; our next read sees the end of the stream
    let r = read_frame(&mut c, LIMIT);
    assert!(
        matches!(
            r,
            Err(FrameError::Closed) | Err(FrameError::Truncated) | Err(FrameError::Io(_))
        ),
        "{r:?}"
    );
    assert!(server.join().unwrap().is_empty());
}

#[test]
fn the_session_type_is_usable_without_any_socket() {
    let mut s = Session::new(cfg(1), 0);
    assert_eq!(s.state(), State::AwaitingHello);
    assert!(s.on_message(hello(), 0).is_empty());
    assert_eq!(s.state(), State::Ready);
}
```

### `examples/bench.rs`

Run with `cargo run --release --example bench`.

```rust
//! Prints the measurements quoted in Lesson 50.
//! Run with `cargo run --release --example bench`.

use std::hint::black_box;
use std::io::Cursor;
use std::net::{TcpListener, TcpStream};
use std::thread;
use std::time::Instant;

use netlab::det::{Cmd, Machine, Op, run};
use netlab::frame::{read_frame, write_frame};
use netlab::msg::Message;
use netlab::sim::{Link, Network, flood};

fn median(mut v: Vec<f64>) -> f64 {
    v.sort_by(|a, b| a.partial_cmp(b).expect("no NaN"));
    v[v.len() / 2]
}

fn time_ns<F: FnMut()>(iters: u32, mut f: F) -> f64 {
    median(
        (0..5)
            .map(|_| {
                let t = Instant::now();
                for _ in 0..iters {
                    f();
                }
                t.elapsed().as_secs_f64() * 1e9 / iters as f64
            })
            .collect(),
    )
}

fn main() {
    println!("== message codec (median of 5 runs)");
    let msgs: Vec<(&str, Message)> = vec![
        ("Ping", Message::Ping(7)),
        ("Tx 300 B", Message::Tx(vec![5; 300])),
        ("Inv 1000 hashes", Message::Inv(vec![[3; 32]; 1000])),
    ];
    for (name, m) in &msgs {
        let bytes = m.encode();
        let enc = time_ns(20_000, || {
            black_box(black_box(m).encode());
        });
        let dec = time_ns(20_000, || {
            black_box(Message::decode(black_box(&bytes)).expect("valid"));
        });
        println!(
            "{name:<16} {:>6} bytes | encode {enc:>9.0} ns | decode {dec:>9.0} ns",
            bytes.len()
        );
    }
    println!("== framing over memory");
    let payload = vec![1u8; 100];
    let mut framed = Vec::new();
    write_frame(&mut framed, &payload, 1 << 20).expect("fits");
    let rd = time_ns(200_000, || {
        let mut c = Cursor::new(black_box(&framed));
        black_box(read_frame(&mut c, 1 << 20).expect("frame"));
    });
    println!("read_frame of a 100-byte payload: {rd:.0} ns");

    println!("== state machine");
    let cmds: Vec<Cmd> = (0..10_000u64)
        .map(|i| Cmd::Credit {
            acct: i,
            amount: 1 + i % 5,
        })
        .collect();
    let t = Instant::now();
    let mut m = Machine::new();
    for c in &cmds {
        m.apply(*c).expect("valid");
    }
    let apply = t.elapsed().as_secs_f64();
    println!(
        "apply 10,000 credits (new accounts): {:.2} ms ({:.1} M cmds/s)",
        apply * 1e3,
        10_000.0 / apply / 1e6
    );
    let h = time_ns(50, || {
        black_box(m.state_hash());
    });
    println!(
        "state_hash of 10,000 accounts: {:.1} us ({} bytes hashed)",
        h / 1e3,
        m.snapshot().len()
    );

    println!("== metered VM");
    let p = [
        Op::Push(1_000_000),
        Op::Push(1),
        Op::Sub,
        Op::Dup,
        Op::Jnz(1),
        Op::Halt,
    ];
    let t = Instant::now();
    let o = run(black_box(&p), u64::MAX).expect("terminates");
    let dt = t.elapsed().as_secs_f64();
    println!(
        "{} instructions in {:.2} ms ({:.0} M instr/s)",
        o.steps,
        dt * 1e3,
        o.steps as f64 / dt / 1e6
    );

    println!("== simulator");
    for (name, net) in [
        ("ring 1000", Network::ring(1000)),
        ("complete 100", Network::complete(100)),
    ] {
        let link = Link {
            drop_per_mille: 0,
            dup_per_mille: 0,
            max_delay: 10,
        };
        let mut rep = None;
        let ms = time_ns(20, || {
            rep = Some(flood(black_box(&net), link, 1, 0));
        }) / 1e6;
        let r = rep.expect("ran");
        println!(
            "{name:<13} {ms:>7.2} ms | sent {} | events {} | finished at tick {}",
            r.sent, r.events, r.finished_at
        );
    }

    println!("== loopback TCP (127.0.0.1)");
    let l = TcpListener::bind("127.0.0.1:0").expect("bind");
    let addr = l.local_addr().expect("addr");
    let server = thread::spawn(move || {
        let (mut s, _) = l.accept().expect("accept");
        s.set_nodelay(true).expect("nodelay");
        while let Ok(f) = read_frame(&mut s, 1 << 20) {
            write_frame(&mut s, &f, 1 << 20).expect("echo");
        }
    });
    let mut c = TcpStream::connect(addr).expect("connect");
    c.set_nodelay(true).expect("nodelay");
    let ping = Message::Ping(1).encode();
    let mut rtts = Vec::new();
    for _ in 0..5 {
        let t = Instant::now();
        for _ in 0..2000 {
            write_frame(&mut c, &ping, 1 << 20).expect("send");
            black_box(read_frame(&mut c, 1 << 20).expect("echo"));
        }
        rtts.push(t.elapsed().as_secs_f64() * 1e6 / 2000.0);
    }
    println!(
        "ping/pong round trip (9-byte payload, 2000 per run): {:.1} us",
        median(rtts)
    );
    drop(c);
    server.join().expect("server");
}
```

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean; **104 tests** (frame 12, msg 18, peer 19, det 27, sim 20, tcp 8) and 8 consecutive full runs without a failure (the socket tests use loopback with generous timeouts and no fixed ports: each binds port 0).

**Mutation check ✓.** I made 124 one-line changes (a "mutant") across the five source modules, each in its own copy of the crate, and ran the whole test suite against each. **119 were detected (118 by a failing or aborting test, 1 as a hang), 5 are equivalent or untestable (M2, M18, M104, M105, M108), none survive unexplained.** Eleven mutants survived the *first* run. Six exposed real gaps that I closed with a new test before re-running them: M27, M29 and M34 (the tests computed their boundaries from the same constants as the code, so changing a limit moved the test with it; I added a test that pins `MAX_TX`, `MAX_BLOCK`, `MAX_INV` and `MAX_BLOCKS` to literal numbers and checks one byte past each), M81 (the price of `Div` was not pinned), M90 (a conditional jump to the end of the program was not tested) and M124 (a duplicated message used the first copy's delay; the existing tests did not pin the exact report of a duplication run, so I added a vector computed by the independent Python simulation). The other five first-run survivors (M2, M18, M104, M105, M108) are equivalent or untestable, for the reasons in the table. Two mutants (M32, M33) were first written with a pattern that no longer matched the formatted source (so they were not valid mutants); I rewrote them rather than counting them. One mutant (M102) makes the seed-search tests loop forever, which counts as detected by timeout; one (M119) made the test process abort, also counted as detected. As in the previous two lessons, my harness had to treat `#[should_panic]` failures (`- should panic ... FAILED`) as failures; I checked the logs of every survivor for the word FAILED. The tests ran in the dev profile.

| # | module | change | result |
|---|---|---|---|
| M1 | `frame` | write: `payload.len() >= max` instead of `>` | caught by `writing_refuses_oversize_payloads_and_writes_nothing` |
| M2 | `frame` | write: the `u32::try_from` length guard removed | **untestable / equivalent**: it only matters for a payload over 4 GiB, which a test cannot allocate; the `max` check already refuses it for every realistic limit |
| M3 | `frame` | read: a close with nothing read is not distinguished from a partial header | caught by `closing_between_frames_is_not_an_error_in_the_middle_is` |
| M4 | `frame` | read: a partial header reported as `Closed` | caught by `closing_between_frames_is_not_an_error_in_the_middle_is` |
| M5 | `frame` | read: limit `len >= max` instead of `>` | caught by `a_zero_limit_allows_only_empty_frames` |
| M6 | `frame` | read: short body not detected | caught by `closing_between_frames_is_not_an_error_in_the_middle_is` |
| M7 | `frame` | read: header loop stops after 3 bytes | caught by `closing_between_frames_is_not_an_error_in_the_middle_is` |
| M8 | `frame` | read: `Interrupted` no longer retried | caught by `interrupted_reads_are_retried` |
| M9 | `frame` | read: header decoded little-endian | caught by `closing_between_frames_is_not_an_error_in_the_middle_is` |
| M10 | `frame` | read: body read without `take(len)` | caught by `frames_round_trip_and_stay_separate_in_one_stream` |
| M11 | `frame` | `is_timeout` ignores `TimedOut` | caught by `only_would_block_and_timed_out_count_as_timeouts` |
| M12 | `frame` | write: length header not written | caught by `a_zero_limit_allows_only_empty_frames` |
| M13 | `frame` | `Display` text of `Closed` changed | caught by `error_messages_are_readable` |
| M14 | `msg` | Tx decode: length limit `>=` instead of `>` | caught by `the_limits_themselves_are_accepted` |
| M15 | `msg` | GetBlocks: `max == 0` allowed | caught by `block_counts_in_get_blocks_must_be_in_range` |
| M16 | `msg` | GetBlocks: `max == 100` refused | caught by `block_counts_in_get_blocks_must_be_in_range` |
| M17 | `msg` | Inv: count limit `>=` instead of `>` | caught by `counts_and_lengths_are_checked_before_anything_is_allocated` |
| M18 | `msg` | Inv: early remaining-bytes check removed | **equivalent**: the early check only avoids an allocation; without it the later length-checked read fails with the same error |
| M19 | `msg` | Blocks: count limit `>=` instead of `>` | caught by `counts_and_lengths_are_checked_before_anything_is_allocated` |
| M20 | `msg` | trailing bytes accepted | caught by `flipping_any_single_bit_never_yields_a_non_canonical_message` |
| M21 | `msg` | tag 2 written as 3 (Ping encodes as Pong) | caught by `every_message_round_trips` |
| M22 | `msg` | Inv: count not written | caught by `every_message_round_trips` |
| M23 | `msg` | encode Tx: limit `<` instead of `<=` | caught by `the_limits_themselves_are_accepted` |
| M24 | `msg` | encode GetBlocks: `max = 100` refused | caught by `every_message_round_trips` |
| M25 | `msg` | encode Inv: limit `<` instead of `<=` | caught by `the_limits_themselves_are_accepted` |
| M26 | `msg` | encode Blocks: limit `<` instead of `<=` | caught by `the_limits_themselves_are_accepted` |
| M27 | `msg` | encode Blocks: block limit `<` instead of `<=` | caught by `the_protocol_limits_are_pinned_to_literal_numbers` |
| M28 | `msg` | Tx decode uses the block limit | caught by `counts_and_lengths_are_checked_before_anything_is_allocated` |
| M29 | `msg` | `MAX_TX` is 65,537 | caught by `the_protocol_limits_are_pinned_to_literal_numbers` |
| M30 | `msg` | `MAX_INV` is 1001 | caught by `counts_and_lengths_are_checked_before_anything_is_allocated` |
| M31 | `msg` | `MAX_BLOCKS` is 101 | caught by `counts_and_lengths_are_checked_before_anything_is_allocated` |
| M32 | `msg` | `Reader::u32` little-endian | caught by `counts_and_lengths_are_checked_before_anything_is_allocated` |
| M33 | `msg` | `Reader::u16` little-endian | caught by `block_counts_in_get_blocks_must_be_in_range` |
| M34 | `msg` | `MAX_BLOCK` is 1 MiB + 1 | caught by `the_protocol_limits_are_pinned_to_literal_numbers` |
| M35 | `peer` | rate-limit penalty 21 | caught by `flooding_is_rate_limited_and_eventually_bans` |
| M36 | `peer` | bad-message penalty 49 | caught by `the_ban_threshold_is_inclusive` |
| M37 | `peer` | unsolicited-pong penalty 9 | caught by `an_unsolicited_pong_costs_score_and_enough_of_them_ban_the_peer` |
| M38 | `peer` | ban when score `>` threshold (not `>=`) | caught by `an_unsolicited_pong_costs_score_and_enough_of_them_ban_the_peer` |
| M39 | `peer` | protocol version not checked | caught by `a_bad_hello_ends_the_session_with_the_right_reason` |
| M40 | `peer` | chain id not checked | caught by `a_bad_hello_ends_the_session_with_the_right_reason` |
| M41 | `peer` | self-connection not detected | caught by `a_bad_hello_ends_the_session_with_the_right_reason` |
| M42 | `peer` | messages before `Hello` ignored instead of closing | caught by `anything_before_hello_ends_the_session` |
| M43 | `peer` | duplicate `Hello` ignored | caught by `a_second_hello_ends_the_session` |
| M44 | `peer` | `Ping(n)` answered with `Pong(0)` | caught by `a_patient_peer_is_never_limited` |
| M45 | `peer` | any `Pong` accepted as the answer | caught by `a_pong_with_the_wrong_nonce_does_not_count` |
| M46 | `peer` | activity time not updated | caught by `any_message_counts_as_activity` |
| M47 | `peer` | rate limiter bypassed | caught by `flooding_is_rate_limited_and_eventually_bans` |
| M48 | `peer` | handshake timeout `>=` instead of `>` | caught by `the_handshake_has_a_deadline` |
| M49 | `peer` | ping timeout `>=` instead of `>` | caught by `idle_connections_are_pinged_and_must_answer` |
| M50 | `peer` | ping interval `>` instead of `>=` | caught by `a_pong_with_the_wrong_nonce_does_not_count` |
| M51 | `peer` | ping nonce not incremented | caught by `the_right_pong_clears_the_ping_and_the_next_ping_has_a_new_nonce` |
| M52 | `peer` | outstanding ping not recorded | caught by `a_pong_with_the_wrong_nonce_does_not_count` |
| M53 | `peer` | outstanding ping not cleared by the right pong | caught by `the_right_pong_clears_the_ping_and_the_next_ping_has_a_new_nonce` |
| M54 | `peer` | our `Hello` advertises version 0 | caught by `the_session_opens_with_our_hello` |
| M55 | `peer` | bucket capacity 999 milli-tokens per token | caught by `application_messages_are_delivered_unchanged` |
| M56 | `peer` | refill even when time did not advance | caught by `the_token_bucket_survives_odd_clocks_and_huge_numbers` |
| M57 | `peer` | refill with unchecked multiplication | caught by `the_token_bucket_survives_odd_clocks_and_huge_numbers` |
| M58 | `peer` | bucket not capped at capacity | caught by `the_token_bucket_refills_exactly_at_the_configured_rate` |
| M59 | `peer` | token cost test `>` instead of `>=` | caught by `flooding_is_rate_limited_and_eventually_bans` |
| M60 | `peer` | elapsed time with unchecked subtraction | caught by `the_token_bucket_survives_odd_clocks_and_huge_numbers` |
| M61 | `peer` | a message costs 999 milli-tokens | caught by `the_token_bucket_refills_exactly_at_the_configured_rate` |
| M62 | `det` | `Credit` with zero amount allowed | caught by `invalid_commands_fail_without_changing_anything` |
| M63 | `det` | `Transfer` with zero amount allowed | caught by `invalid_commands_fail_without_changing_anything` |
| M64 | `det` | `Credit` overflow becomes 0 | caught by `balances_cannot_overflow` |
| M65 | `det` | `Transfer` recipient overflow becomes 0 | caught by `balances_cannot_overflow` |
| M66 | `det` | insufficient funds becomes 0 | caught by `a_block_is_all_or_nothing` |
| M67 | `det` | self-transfer special case removed | caught by `a_transfer_to_yourself_changes_nothing_but_still_needs_the_funds` |
| M68 | `det` | height not incremented by a block | caught by `a_block_is_all_or_nothing` |
| M69 | `det` | a successful block is not committed | caught by `a_block_is_all_or_nothing` |
| M70 | `det` | failing command index off by one | caught by `a_block_is_all_or_nothing` |
| M71 | `det` | height not in the snapshot | caught by `restore_refuses_everything_that_snapshot_never_produces` |
| M72 | `det` | account count not in the snapshot | caught by `replicas_that_apply_the_same_log_agree_on_every_hash` |
| M73 | `det` | domain length not in the snapshot | caught by `replicas_that_apply_the_same_log_agree_on_every_hash` |
| M74 | `det` | `restore`: length check `<=` | caught by `snapshots_round_trip_exactly` |
| M75 | `det` | `restore`: duplicate keys allowed | caught by `restore_refuses_everything_that_snapshot_never_produces` |
| M76 | `det` | `restore`: zero balances allowed | caught by `restore_refuses_everything_that_snapshot_never_produces` |
| M77 | `det` | `restore`: `wrapping_mul` instead of `checked_mul` | caught by `restore_refuses_everything_that_snapshot_never_produces` |
| M78 | `det` | `hash_in_key_order` does not sort | caught by `hashing_a_hashmap_in_iteration_order_is_not_deterministic` |
| M79 | `det` | `sum_fixed` wraps | caught by `integer_sums_are_exact_in_any_order_and_report_overflow` |
| M80 | `vm` | `Mul` costs 2 | caught by `arithmetic_runs_on_a_stack` |
| M81 | `vm` | `Div` costs 4 | caught by `division_costs_five_and_multiplication_three` |
| M82 | `vm` | fuel check `<=` instead of `<` | caught by `a_counting_loop_costs_exactly_four_instructions_per_iteration` |
| M83 | `vm` | stack limit off by one | caught by `the_stack_depth_is_bounded` |
| M84 | `vm` | `Add` wraps | caught by `arithmetic_overflow_is_a_trap_never_a_wrap` |
| M85 | `vm` | `Sub` wraps | caught by `arithmetic_overflow_is_a_trap_never_a_wrap` |
| M86 | `vm` | `Mul` wraps | caught by `arithmetic_overflow_is_a_trap_never_a_wrap` |
| M87 | `vm` | division by zero not trapped | caught by `division_truncates_toward_zero_and_traps_on_zero_and_overflow` |
| M88 | `vm` | `i64::MIN / -1` wraps | caught by `division_truncates_toward_zero_and_traps_on_zero_and_overflow` |
| M89 | `vm` | `Jmp` to the end of the program is a trap | caught by `jumps_and_halting` |
| M90 | `vm` | `Jnz` to the end of the program is a trap | caught by `a_taken_conditional_jump_may_target_the_end_of_the_program` |
| M91 | `vm` | `Jnz` jumps on zero | caught by `a_counting_loop_costs_exactly_four_instructions_per_iteration` |
| M92 | `vm` | `Swap` does nothing | caught by `stack_operations` |
| M93 | `vm` | `Swap` with one element allowed | caught by `stack_operations` |
| M94 | `vm` | `Halt` does not halt | caught by `jumps_and_halting` |
| M95 | `vm` | `fuel_used` reports the whole budget | caught by `a_counting_loop_costs_exactly_four_instructions_per_iteration` |
| M96 | `vm` | `steps` not counted | caught by `a_counting_loop_costs_exactly_four_instructions_per_iteration` |
| M97 | `vm` | `Jmp` target off by one | caught by `an_infinite_loop_is_stopped_by_the_fuel_limit` |
| M98 | `vm` | `Jnz` target off by one | caught by `a_counting_loop_costs_exactly_four_instructions_per_iteration` |
| M99 | `sim` | xorshift shift 11 instead of 12 | caught by `reports_match_an_independent_python_simulation` |
| M100 | `sim` | xorshift shift 24 instead of 25 | caught by `reports_match_an_independent_python_simulation` |
| M101 | `sim` | xorshift shift 26 instead of 27 | caught by `reports_match_an_independent_python_simulation` |
| M102 | `sim` | even multiplier constant | detected as a **hang**: with an even multiplier the seed-search tests never finish (killed after 200 s) |
| M103 | `sim` | seed 0 not remapped | caught by `the_generator_matches_an_independent_implementation` |
| M104 | `sim` | rejection-sampling limit removed | **equivalent**: the two limits differ only when a draw lands in the last `2^64 mod n` values, probability about 2⁻⁶⁴ |
| M105 | `sim` | rejection test `<=` instead of `<` | **equivalent**: `x == limit` happens with probability about 2⁻⁶⁴ |
| M106 | `sim` | `from_edges`: duplicate edges kept | caught by `the_constructors_build_the_expected_graphs` |
| M107 | `sim` | ring connects `i` to `i + 2` | caught by `a_clean_flood_reaches_everyone_in_a_connected_network` |
| M108 | `sim` | `complete`: self-loops generated | **equivalent**: `from_edges` ignores self-loops, so adding them changes nothing |
| M109 | `sim` | origin receipt time not recorded | caught by `a_partitioned_network_reaches_only_the_origin_component` |
| M110 | `sim` | forwarding back to the sender | caught by `delays_stay_within_the_configured_bounds` |
| M111 | `sim` | loss test `<=` instead of `<` | caught by `per_mille_probabilities_use_a_strict_comparison` |
| M112 | `sim` | delay may be 0 | caught by `delays_stay_within_the_configured_bounds` |
| M113 | `sim` | delivery time ignores the delay | caught by `delays_stay_within_the_configured_bounds` |
| M114 | `sim` | `sent` not counted | caught by `a_clean_flood_reaches_everyone_in_a_connected_network` |
| M115 | `sim` | `dropped` not counted | caught by `losing_everything_leaves_only_the_origin` |
| M116 | `sim` | `duplicated` not counted | caught by `duplication_doubles_the_arrivals_but_not_the_receipts` |
| M117 | `sim` | `events` not counted | caught by `a_clean_flood_reaches_everyone_in_a_connected_network` |
| M118 | `sim` | `finished_at` not updated | caught by `delays_stay_within_the_configured_bounds` |
| M119 | `sim` | first-receipt check removed (nodes forward every copy) | caught (the test process aborted) |
| M120 | `sim` | duplication test `<=` instead of `<` | caught by `per_mille_probabilities_use_a_strict_comparison` |
| M121 | `sim` | duplicate uses the first copy's delay | caught by `reports_match_an_independent_python_simulation` |
| M122 | `sim` | `reached()` counts nodes not reached | caught by `a_partitioned_network_reaches_only_the_origin_component` |
| M123 | `sim` | `edges()` counts each edge twice | caught by `the_constructors_build_the_expected_graphs` |
| M124 | `sim` | duplicate copy scheduled with the first delay | caught by `duplicate_copies_get_their_own_delay` |

What the table does not show: it tests *these* edits, not every possible bug; there is no mutant for the socket layer's timing behavior beyond what the loopback tests observe; the equivalent mutants could hide a real gap if my reasoning about them is wrong; and passing every mutant says nothing about security or about behavior on a real network.

**Design notes.**

- Layers are separate: `frame` knows nothing about messages, `msg` nothing about sockets, `peer` nothing about either (it consumes decoded `Message` values and time), `det` and `sim` depend on nothing else. Each layer is tested alone, and `tests/tcp.rs` joins them over real sockets.
- Time is always passed in, randomness is always seeded, and no function reads the clock or the environment, so tests never sleep (except the socket tests, which wait on real I/O with long timeouts).
- Error enums are compared by value in tests (`assert_eq!(…, Err(DecodeError::TooMany { count, max }))`), and Display strings are tested because they end up in logs.
- Not handled: encryption and authentication, async I/O, whole-frame deadlines, peer discovery, connection limits, persistent bans, a real VM memory model, Merkle state commitments, and everything in §9.

</details>

---

## 14. Key takeaways

- TCP is a byte stream: frame your messages, loop until a frame is complete, and treat the length as hostile (✓ a 4 GiB announcement refused after four bytes; a single `read` loses data when the body arrives in pieces).
- Decoding must be strict and canonical: reject trailing bytes and unknown tags, check counts against the remaining bytes before allocating, and test `decode(b) = Ok(m) ⇒ encode(m) = b` (✓ random inputs and every single-bit flip).
- Write protocols **sans-IO**: events in, actions out, time injected. Then handshakes, timeouts, rate limits and bans are tested in microseconds without sockets (✓ 19 tests).
- A read timeout bounds one `read`, not a whole message; use a whole-frame deadline too.
- Deterministic execution means no hidden inputs: ordered maps, fixed-width integers with explicit byte order, checked arithmetic, no floats, no clock or randomness in validation (✓ `HashMap` order and `0.1 + 0.2 + 0.3` both differ).
- Run untrusted code under a fuel meter with prices fixed by the protocol; a stop at the same instruction on every node is itself a deterministic result (✓ `[Jmp(0)]` stops for every budget).
- Apply blocks atomically; hash state from a canonical encoding with a domain label (✓ two replicas agree after every block, hashes match an independent Python implementation).
- A seeded simulator turns a network into a function (✓ matches an independent simulation); gossip without retransmission is cheap and unreliable under loss (✓ 25/200 for a ring, 200/200 for a complete graph at 30% loss).
- Mutation testing again found real gaps (limits that the tests computed from the same constants as the code, an unpinned instruction price, a jump to the end of the program, and a duplicate copy that reused the first delay), and the survivors that remain are explained, not ignored.

**Connections.** [Lesson 49](49_merkle_transactions_consensus.md) said consensus-critical code must be deterministic; this lesson builds the machinery (ordered state, canonical snapshots, metering) and the message layer that carries blocks and transactions between nodes. [Lesson 48](48_crypto_primitives.md) supplies the hash and the domain separation used for the state hash; an authenticated channel built from its primitives is the missing security layer. [Lesson 47](47_deployment_wasm.md) covers fuel and determinism in Wasm. [Lessons 44 and 45](45_http_apis.md) cover binary formats, limits and timeouts at the HTTP level. [Lesson 40](40_property_testing_fuzzing.md) explains the testing styles used for the codec, and [Lessons 27 and 29](29_locks_deadlocks.md) the thread issues around socket loops. Lesson 51 moves up a layer: RPC, nodes, indexers, transaction monitoring and serialization.

---

## 15. References

- Rust standard library: `std::io::{Read, Write}`, `Read::take`, `Read::read_exact`, `std::net::{TcpListener, TcpStream}`, `TcpStream::set_read_timeout` and `set_nodelay`, `BTreeMap`, `HashMap`: <https://doc.rust-lang.org/std/>
- RFC 9293, Transmission Control Protocol (from memory, for the stream semantics and Nagle's algorithm): <https://www.rfc-editor.org/>
- David Blackman and Sebastiano Vigna, "Scrambled linear pseudorandom number generators" (xorshift* family, from memory); George Marsaglia, "Xorshift RNGs" (2003, from memory).
- IEEE 754 floating-point arithmetic (from memory, for the statements about which operations are exactly specified).
- Bitcoin developer documentation on the peer-to-peer network and on misbehavior scores (from memory; I did not check the current numbers).
- Documentation of sans-IO style protocol libraries (for example `h11` and `quinn-proto`, from memory).
- FoundationDB's and TigerBeetle's descriptions of deterministic simulation testing (from memory).
- RustSec/clippy documentation of `clippy::unused_io_amount`: <https://rust-lang.github.io/rust-clippy/>
- RustCrypto `sha2`: <https://github.com/RustCrypto/hashes>

The URLs and titles above are the standard locations; I did not open them during this session (the sandbox cannot reach the documentation or standards sites), so if a link has moved, search for the title. **Unchecked or from memory:** the statements about TCP and `TCP_NODELAY`/Nagle; which error (`WouldBlock` or `TimedOut`) a timed-out read gives on each platform; how `read_exact` treats `Interrupted`; the claims about Rust's `HashMap` hasher and hash-flooding; IEEE 754 and platform differences of math functions; the properties of xorshift64* beyond what the tests check (it is not cryptographically secure); sans-IO libraries, deterministic simulation testing and Bitcoin's ban scores; blockchain virtual machines and their metering; Merkle–Patricia tries; TLS and Noise as channel options. **Environment caveats:** timings are single-machine micro-measurements from three runs of one example on a shared virtual machine with two cores, with loopback TCP; the gossip delivery rates are counts over 200 seeds at one loss setting, not estimates for any real network. Measurements and outcomes marked ✓ were observed with `rustc 1.97.0` and `sha2 0.11`. The code blocks in exercises 3, 4 and 5 and in the debugging challenge's two versions were compiled and run in a scratch crate (✓ 14 tests, repeated three times without a failure; the buggy version intentionally fails `cargo clippy`); the numbers in exercises 6 and 7 were evaluated by tests; the mutation results are those of the table above (a test process that aborted, failed an assertion or did not finish within 200 seconds counted as detecting the mutant). **No part of this lesson makes any system secure.**
