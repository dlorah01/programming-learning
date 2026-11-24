---
layout: default
title: "Lesson 45 — HTTP clients and servers, APIs"
---

# Lesson 45 — HTTP clients and servers, APIs

**Module 14 · Application development** · Status: generated · Edition used in examples: **2024** · External dependencies: `axum 0.8.9`, `tokio 1.54.1`, `reqwest 0.13.5` (no TLS, see §1), `tower-http 0.7.1`, `serde 1.0.229`, `serde_json 1.0.154`; dev only: `tower 0.5.3`, `http-body-util 0.1.5`

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux, 2 cores. As before, ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior I did not run. **Everything ran over plain HTTP on `127.0.0.1`.** I did not test TLS/HTTPS (the `reqwest` here is built without a TLS backend), HTTP/2, WebSockets, streaming bodies, cookies, CORS, redirects, proxies, any other web framework, any database, or the public Internet (the sandbox's network is restricted). Timings are single measurements on one shared virtual machine where the load generator and the server compete for the same two cores. I could not open the documentation sites, so rules stated without ✓ are from memory; the closing paragraph lists them.

Builds on [Lesson 44](44_cli_serialization.md) (serde, validation, untrusted input, exit codes), [Lesson 43](43_observability_ci_supply_chain.md) (secrets, log forging, duplicate dependencies), [Lesson 32](32_cancellation_streams_backpressure.md) (cancellation, backpressure), [Lesson 31](31_tokio.md) (Tokio), [Lesson 30](30_futures_async.md) (futures), [Lesson 29](29_locks_deadlocks.md) (locks), [Lesson 26](26_error_design.md) (error design), and [Lesson 39](39_testing.md) (testing). Lesson 46 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

Almost every Rust program that other people use ends up speaking HTTP, as a server, as a client, or both. The language gives you memory safety; it does not give you a *good API*. The decisions that matter are the same in any language, and they are mostly about the boundary:

- what the server does with a request it should not trust (too big, too slow, malformed, unauthenticated, duplicated);
- what the client does when the network or the server misbehaves (timeouts, retries, partial failure);
- what the contract looks like to everybody else (status codes, error bodies, pagination, compatibility).

The running example is `apilab`, a small orders API: `POST /orders`, `GET /orders/{id}`, `GET /orders` (paginated), `GET /health`, bearer-token auth on writes, idempotency keys, a typed client with retries, and graceful shutdown. It is in-memory on purpose; databases come in Lesson 46.

Scope of what I verified: 31 tests (16 in-memory, 15 over real sockets) plus the experiments quoted below. `reqwest` is used with `default-features = false`, so it has **no TLS backend** and cannot open `https://` URLs (✓ see §7.5). In production you would enable one, which changes compile time and dependencies in ways I did not measure.

---

## 2. HTTP in one page (from memory)

HTTP is a request/response protocol. Each request has a *method*, a *target* (path and query), *headers* and an optional *body*; each response has a *status*, headers and an optional body. Details are in RFC 9110 (semantics) and RFC 9112 (HTTP/1.1 messages); I did not open them here.

| Idea | What it means | Where it matters here |
|---|---|---|
| **Safe** methods (`GET`, `HEAD`, `OPTIONS`) | do not change server state | any client may call them freely, caches may keep them |
| **Idempotent** methods (safe ones, `PUT`, `DELETE`) | repeating the request has the same *effect* as sending it once | a client may retry them after an uncertain failure |
| `POST` | neither safe nor idempotent by definition | retry only with a mechanism like an idempotency key (§6.4) |
| Status classes | 2xx success, 3xx redirect, 4xx *client* mistake, 5xx *server* failure | the class tells a client whether trying again can help |
| `401` vs `403` | not authenticated (who are you?) vs authenticated but not allowed | `apilab` only has the first, §6.5 |
| `Content-Type` | how to read the body | a JSON endpoint should refuse other types with `415` |
| Keep-alive | one TCP connection carries many requests | connection reuse roughly halves the per-request cost here, §9 |

A fact that surprises people: the client cannot tell the difference between "the request never arrived" and "the server did the work but the response was lost". That is the entire reason idempotency matters.

---

## 3. The stack ✓

| Layer | Crate | Role |
|---|---|---|
| Protocol | `hyper 1.12` | HTTP/1 (and 2) parsing and connections |
| Composition | `tower 0.5`, `tower-http 0.7` | `Service` and `Layer`: timeouts, limits, middleware |
| Server framework | `axum 0.8.9` | routing, extractors, responses |
| Runtime | `tokio 1.54` | tasks, sockets, timers (Lessons 30–32) |
| Client | `reqwest 0.13.5` | pooled HTTP client |

✓ Cargo features used for the server: `axum` with `http1`, `json`, `tokio`, `query` only; `tower-http` with `timeout` and `limit`. The first cold `cargo build` of `apilab` (compiling every dependency, 2 cores) finished in **31.1 s** in the dev profile. ✓ `Cargo.lock` contains **two versions of `tower-http`** (0.6.11, pulled in by `reqwest`, and 0.7.1, which I asked for): exactly the kind of duplicate `cargo deny` reports (Lesson 43). It is harmless here and a reason to check versions when you pick crates.

---

## 4. The server

### 4.1 Handlers, extractors, responses ✓

An axum handler is an `async fn` whose arguments are *extractors* and whose return type implements `IntoResponse`:

```rust
async fn get_order(
    State(state): State<AppState>,       // shared state
    ApiPath(id): ApiPath<u64>,           // from /orders/{id}, parsed as u64
) -> Result<Json<Order>, ApiError> {     // Ok -> 200 JSON, Err -> the error envelope
    let store = state.store.lock().unwrap_or_else(|p| p.into_inner());
    store.orders.get(&id).cloned().map(Json).ok_or(ApiError::NotFound)
}
```

Rules enforced at compile time, with the exact diagnostics I got (Lesson 35 explains why the macro-free version cannot say more):

| Mistake | Without `#[axum::debug_handler]` | With it (needs axum's `macros` feature) ✓ |
|---|---|---|
| `Json<T>` before another extractor | ``the trait bound `fn(Json<Body>, HeaderMap) -> ... {handler}: Handler<_, _>` is not satisfied`` | ``Json<_> consumes the request body and thus must be the last argument to the handler function`` |
| returning a type without `IntoResponse` | the same `Handler<_, _>` error | ``the trait bound `NotAResponse: IntoResponse` is not satisfied`` |
| a `std::sync::MutexGuard` alive across `.await` | the same `Handler<_, _>` error | `future cannot be sent between threads safely`, ``the trait `Send` is not implemented for `std::sync::MutexGuard<'_, u32>` ``, and `future is not Send as this value is used across an await` |
| `Router<AppState>` passed to `serve` without `.with_state(...)` | `Router<AppState>: Service<IncomingStream<...>>` is not satisfied | (same) |

Use `#[axum::debug_handler]` while a handler will not compile; remove it afterwards if you do not want the feature in the final build.

### 4.2 One error type, one error format ✓

A client should be able to parse every failure the same way. `apilab` has one enum, `ApiError`, with a status, a stable machine-readable `code` and a human `message`, and implements `IntoResponse` once:

```json
{"error":{"code":"invalid_request","message":"customer must be 1 to 64 characters, got 0"}}
```

The part people miss: errors produced **outside your handlers** use the framework's own format. ✓ In my first version three of them did:

| Situation | Before | After (custom extractor or `map_response`) |
|---|---|---|
| `GET /orders/abc` (path does not parse) | `400`, `text/plain`: ``Invalid URL: Cannot parse `abc` to a `u64` `` | `400`, JSON envelope, same text as `message` |
| `DELETE /orders` (wrong method) | `405`, empty body | `405`, JSON envelope (the router still adds its `Allow` header) |
| handler too slow | `408`, empty body | `408`, JSON envelope |

The fixes are small: wrappers `ApiJson<T>`, `ApiQuery<T>`, `ApiPath<T>` that implement `FromRequest(Parts)` by calling the axum extractor and converting its rejection, and one `middleware::map_response` that rewrites the empty 405/408. Most error-path tests now assert the code, not just the status.

✓ All the response behaviors, from a real server over a real socket:

| Request | Status | `code` |
|---|---|---|
| `GET /health` | 200 | (body `ok`) |
| `GET /nope` | 404 | `not_found` |
| `DELETE /orders` | 405 | `method_not_allowed` |
| `POST /orders`, no token / wrong token | 401 plus `WWW-Authenticate: Bearer` | `unauthorized` |
| `POST /orders`, valid | 201 plus `Location: /orders/1` | |
| unknown field, missing field, wrong type, negative number | 400 | `invalid_request` |
| empty customer, amount above the limit | 400 | `invalid_request` |
| body is not JSON | 400 | `invalid_request` |
| no `Content-Type`, or `text/plain` | 415 | `unsupported_media_type` |
| body over the limit (also when sent chunked) | 413 | `body_too_large` |
| `GET /orders/99` | 404 | `not_found` |
| `GET /orders?limit=0`, `limit=abc`, unknown parameter | 400 | `invalid_request` |
| handler slower than the server timeout | 408 | `request_timeout` |

The 400 messages from deserialization come straight from `serde_json` and axum, for example ``Failed to deserialize the JSON body into the target type: x: unknown field `x`, expected `customer` or `amountCents` at line 1 column 37`` and ``Failed to deserialize query string: limit: invalid digit found in string``. They help clients, and they also reveal your type and field names. That is usually fine for a public API with a documented schema; decide deliberately for an internal one.

### 4.3 Shared state in async code ✓

```rust
#[derive(Clone)]
struct AppState {
    store: Arc<Mutex<Store>>,     // std::sync::Mutex, never held across an .await
    config: Arc<Config>,
}
```

- `State<T>` clones the state for each request, so it must be cheap to clone: an `Arc`.
- A `std::sync::Mutex` is fine when the critical section is short and contains no `.await`; the compiler enforces the second half (§4.1 table: the guard is not `Send`). Use `tokio::sync::Mutex` only when you must hold the lock across an await (from memory: it is slower and has no poisoning).
- `.lock().unwrap_or_else(|p| p.into_inner())` keeps the service alive after a thread panicked while holding the lock (poisoning, Lesson 24) at the price of reading possibly half-updated data. Whether that is right depends on your invariants; the mutation table (§15) shows that no test exercises this branch.
- ✓ Idempotency check and insert happen **under the same lock acquisition**. 32 concurrent requests with one key produced exactly one order (test `concurrent_requests_with_one_key_create_one_order`); 50 concurrent creates got ids 1..=50 with no gaps or duplicates.
- Design limits of this toy: the order map and the key map grow without bound; a real service needs eviction or a database (Lesson 46).

### 4.4 Limits, timeouts and what is *not* limited ✓

Layers wrap the whole router:

```rust
Router::new()
    .merge(writes).merge(reads).fallback(fallback)
    .layer(DefaultBodyLimit::max(MAX_BODY_BYTES))                      // 1024 bytes
    .layer(TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, timeout))
    .layer(middleware::map_response(envelope_framework_errors))
    .with_state(state)
```

What I measured:

- **Body size.** The limit is inclusive (a body of exactly 1024 bytes is accepted, 1025 gets `413`), and it also holds for a **chunked** body with no `Content-Length` (✓ a 2 KB chunked upload got `413` and `connection: close`). With no limit configured, axum's default still applied: a 1 MiB body was accepted and a 3 MiB body got `413` (✓ `Failed to buffer the request body: length limit exceeded`, text cut off in my output); the documented value is 2 MB, which I did not probe.
- **Handler time.** `TimeoutLayer` answered `408` after the configured time; a sleep shorter than the limit passed. It bounds the time to produce a response *once the request has been read and routed*.
- **Header size.** ✓ A request with a single 400,000-byte header got `200 OK`; with 1,000,000 bytes it got `431 Request Header Fields Too Large`. hyper's default is far larger than most proxies allow (I did not find the exact threshold).
- **A silent client.** ✓ I connected, sent `GET /health HTTP/1.1\r\nHost: x\r\n` (no blank line to end the headers) and waited: after **8 seconds** the server had neither answered nor closed the connection. `TimeoutLayer` does not help, because the request does not exist yet; the usual cure is a header-read timeout at the server layer, or a reverse proxy in front (`hyper`'s HTTP/1 builder has a `header_read_timeout` option, from memory; `axum::serve` does not expose it, and I did not wire it up). Treat "slow-header" clients as an unsolved risk of this toy.
- **Garbage.** ✓ `HELLO\r\n\r\n` got `400 Bad Request` with an empty body; an HTTP/1.0 request got an HTTP/1.0 response.
- **Concurrency.** Nothing limits the number of simultaneous connections or in-flight requests in `apilab`. A `tower::limit::ConcurrencyLimitLayer` or load shedding is the usual tool (from memory, not run; backpressure ideas are in Lesson 32).

### 4.5 Graceful shutdown ✓

```rust
pub async fn serve(listener: TcpListener, app: Router, shutdown: impl Future<Output = ()> + Send + 'static) -> io::Result<()> {
    axum::serve(listener, app).with_graceful_shutdown(shutdown).await
}
```

✓ Test: a request to `/sleep/500` is in flight; the shutdown signal fires after 100 ms; the request still completes with `200`; the `serve` future returns `Ok(())` within 3 s; a new request afterwards fails to connect. In `main` the signal is usually `tokio::signal::ctrl_c()` plus `SIGTERM` (from memory; `apilab` is a library, so I tested the mechanism with a channel). A shutdown with no deadline can wait forever for a stuck request; put a timeout around it in the caller.

---

## 5. A request that panics ✓

A handler that panics (here: `n.parse().unwrap()` on user input) does **not** bring the server down. Tokio catches the panic at the task boundary, the panic message goes to the server's stderr, and the connection is dropped without a response. ✓ Measured against the buggy server of §13:

```text
GET /square/12          -> 200 OK  body="144"
GET /square/abc         -> ERROR: error sending request for url (http://127.0.0.1:43189/square/abc)
                           (is_request=true, is_connect=false)
GET /square/99999999999 -> 200 OK  body="error: too big"        <- an error reported as success
server still alive: 200 OK
```

(server stderr: ``thread 'tokio-rt-worker' panicked at ...: called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }``.) So the *server* survives and the *client* sees an opaque transport error: it cannot tell a bug from a network failure, and it may retry a request that crashes the handler every time. With `tower_http::catch_panic::CatchPanicLayer` the same request returns `500 Internal Server Error` with the body `Service panicked` ✓, which at least tells the client the server answered. It is a safety net, not a substitute for returning errors: the panic still unwinds, still prints, and a poisoned `std::sync::Mutex` (if the panic happened while holding it) still poisons (Lesson 24). A build with `panic = "abort"` (Lesson 42's size recipe) kills the *whole process* on this panic instead (from memory; I did not test it here), so the profile you ship matters for a server.

---

## 6. API design

### 6.1 Resources and verbs

Name things, not actions: `POST /orders` creates, `GET /orders/{id}` reads, `GET /orders` lists. Use the status code for the outcome (`201` plus `Location` on creation, `404` for a missing resource, `409` for a conflict) and put machine-readable detail in the body, not in the message text clients will parse.

### 6.2 Validation is two steps ✓

1. **Shape**: serde plus `deny_unknown_fields` (Lesson 44) rejects wrong types, missing and unknown fields (`400`).
2. **Rules**: `NewOrder::validate` checks what types cannot say: 1 to 64 *characters* (✓ 64 `é` accepted, 65 `x` refused), amount at most 10¹² cents (✓ the limit itself is accepted, one above is not).

A request that fails validation must not change state: ✓ a rejected body does not consume an idempotency key (test `an_invalid_body_does_not_consume_the_idempotency_key`), and rejected unauthenticated calls create nothing.

### 6.3 Pagination: cursors, not offsets ✓

`GET /orders?after=<id>&limit=<n>` returns `{"items":[...],"nextAfter":<id>}`. The server fetches `limit + 1` rows to know whether another page exists, and omits `nextAfter` on the last page.

- ✓ Walking five orders with `limit=2` yields ids `[1,2,3,4,5]` in three pages; a last page that is *exactly* full has no `nextAfter` (the off-by-one is the whole reason for fetching `limit + 1`); `after=u64::MAX` returns an empty page instead of overflowing (`saturating_add`); the last page's JSON has no `nextAfter` key at all (not `null`); `limit` must be 1 to 100.
- An offset (`?page=3`) shifts when rows are inserted during the walk, so a client can skip or repeat items; a cursor on a stable, ordered key does not (from memory of the general argument; the toy store only ever appends).

### 6.4 Idempotency keys ✓

A client that times out on `POST /orders` does not know if the order exists. With an `Idempotency-Key` header the server remembers the first result for that key:

| Situation | Result ✓ |
|---|---|
| first request with key `k` | `201`, new order |
| same key, **same body** | `200`, the original order, nothing new stored |
| same key, **different body** | `409 idempotency_key_reused` |
| a different key, same body | `201`, another order (that is a new request) |
| no key | every request creates an order |
| key empty, longer than 64 bytes, or not visible ASCII | `400` before anything is stored |
| 32 concurrent requests, same key | one order, 32 identical answers |

Replays return `200`, not `201`, so a client can tell. A production version also expires keys and stores the response in a database in the same transaction as the order; this one keeps them forever in memory.

### 6.5 Authentication and authorization

`apilab` protects writes with a static bearer token compared by `tokens_equal`, which looks at every byte instead of stopping at the first difference. ✓ Tested: missing header, wrong scheme (`Basic`), no `Bearer ` prefix, lowercase `bearer`, a token that is one byte longer or shorter, and a differing last byte all get `401` with `WWW-Authenticate: Bearer`, and create nothing. That comparison is *hygiene against the simplest timing leak*, not a proven constant-time implementation; it is also not a real auth system (no users, expiry, rotation or `403`). Use an established mechanism for real services. `Config` has a manual `Debug` that prints `<redacted>` (✓ tested), so a stray `{:?}` cannot leak the token (Lesson 43).

### 6.6 Evolving the API (from memory)

Treat the JSON shape like the file format of Lesson 44: add optional fields, never change a field's meaning, keep error `code`s stable, and version in the URL or a header only when you must break something. Clients should ignore unknown fields (the server here uses `deny_unknown_fields` on *requests* but its responses can grow).

---

## 7. The client

### 7.1 Build one, share it ✓

`reqwest::Client` holds a connection pool. Build it once and clone it (cloning is cheap and shares the pool). ✓ Measured on loopback, one sequential client, `GET /health`:

| | per request |
|---|---:|
| reused connection (keep-alive) | 74.8 to 79.2 µs |
| a new TCP connection for every request | 139.6 to 156.1 µs |

So building a client per request roughly doubles the cost even on loopback; across a real network the connection and TLS setup cost far more (not measured).

### 7.2 Timeouts are not optional ✓

`reqwest` has **no overall timeout by default** (from memory of its documentation); `ApiClient` sets one for each attempt with `ClientBuilder::timeout` and `connect_timeout`. ✓ A server that sleeps 3 s against a 150 ms client timeout returned a `ClientError::Transport` whose `is_timeout()` was true, in well under 1.5 s. A connection to a closed port produced `is_connect()`.

### 7.3 Retries: what is safe ✓

```rust
let may_retry = method == Method::GET || idempotency_key.is_some();
let attempts = if may_retry { self.config.retries + 1 } else { 1 };
```

Rules implemented and tested:

- Retry only **idempotent** requests: `GET`, or `POST` that carries an idempotency key. ✓ `POST` without a key made exactly one attempt against a server answering `503`; with a key it retried and succeeded (2 calls).
- Retry only **transient** conditions: connection failures, timeouts, and statuses `429`, `502`, `503`, `504`. ✓ Table test over eight statuses: those four were retried (2 calls); `500`, `400`, `404`, `501` were not (1 call).
- **Bound** the retries: ✓ with `retries = 2` against a permanently failing server the client made exactly 3 calls and returned the last answer.
- **Back off**: delay `base × 2ⁿ`, capped at 2 s (✓ for base 100 ms: 100, 200, 400, 800, 1600, 2000, 2000 ms; a huge attempt number does not overflow). There is no jitter, so many clients that fail together will also retry together (the "thundering herd"); production clients add random jitter (from memory; not implemented here).
- ✓ Retrying covers a server that is not up yet: with the port closed, a client with 8 retries connected after the server started 120 ms later.
- Never retry blindly on `500`: the server may have done part of the work.

### 7.4 Errors a caller can act on ✓

`ClientError` separates three cases: `Transport` (no usable response), `Api { status, error }` (the server's JSON envelope; switch on `error.code`) and `Unexpected { status, body }` (a non-success answer that is not the envelope, such as a gateway's HTML page; ✓ the body is truncated to 200 characters). `ClientConfig` has a manual `Debug` that redacts the token.

### 7.5 TLS ✓

✓ With `default-features = false` this `reqwest` has no TLS backend; I did not try `https://` against a real server. A plain-HTTP server on loopback reached with an `https://` URL produced `error sending request`, caused by `client error (Connect)`, caused by **`invalid URL, scheme is not http`**: without a TLS backend the client simply refuses the scheme. Enabling `reqwest`'s default features (a TLS backend) fixes that and also pulls in a TLS library and, depending on the choice, a C or assembly build step; I did not build that configuration, so I can say nothing about its compile time or size. Wherever TLS is terminated, the application code in this lesson does not change.

---

## 8. Testing HTTP code

Two layers, both used here:

1. **In memory.** `Router` is a `tower::Service`; `ServiceExt::oneshot(request)` runs a request through the whole stack (routing, layers, extractors) without a socket. ✓ 16 tests, about 10 ms in total, one assertion style for status, `code`, headers and body. Cost per request: **1.54 to 1.79 µs** (100,000 `GET /health` calls, release build).
2. **Real sockets** on `127.0.0.1:0` (the OS picks a free port, so parallel tests do not collide). ✓ 15 tests, about 0.6 s in total, for things only a socket can show: timeouts, connection refused, graceful shutdown, concurrency, the typed client.

✓ Reliability: I ran the whole suite 30 times in a row and saw no failure. The time-based tests use margins of at least a factor of three (a 150 ms client timeout against a 3 s sleep, shutdown within 3 s for a 500 ms request); a test that races a timer against itself is a flake waiting for a slow CI machine.

Test doubles worth copying: a **flaky server** (an axum router with an atomic counter that fails the first *n* calls) to test retries, and a server that sleeps to test timeouts. They are ten lines each and test the *client* policy, not the network.

---

## 9. Performance ✓

`examples/bench.rs`, release build, loopback, client and server on the same two cores, `GET /health` unless noted; three runs each:

| Measurement | Result |
|---|---|
| in-process (`oneshot`), no sockets | 1.54 to 1.79 µs per request |
| 1 client, keep-alive, sequential | 74.8 to 79.2 µs per request (12.6k to 13.4k requests/s) |
| 1 client, new connection per request | 139.6 to 156.1 µs per request |
| 8 concurrent clients | 53.5k to 55.7k requests/s |
| 64 concurrent clients | 59.0k to 60.5k requests/s |
| 1 client, `POST /orders` (JSON in, JSON out, one locked insert) | 98.3 to 100.4 µs per request |

Reading it honestly:

- The in-process number is the framework's own cost; almost everything else is the kernel and the HTTP stack on both sides. I did not separate the client's share of the cost from the server's, and both run on the same machine.
- Concurrency raised throughput by about four times and then flattened (two cores, both busy running client and server). This says nothing about a real network, a real handler, or other hardware.
- A handler that blocks a worker thread (CPU work, a blocking call) stops one of the runtime's worker threads for its whole duration; with *w* worker threads and a blocking handler of *t* seconds, throughput cannot exceed *w ÷ t* requests/s (arithmetic, not measured). Move blocking work to `tokio::task::spawn_blocking` or a separate pool (Lesson 31).
- The response body `ok` is two bytes. I did not measure large bodies, compression, or JSON-heavy handlers.

---

## 10. Security checklist

1. **Limit everything the client controls**: body (✓ 1 KB), header size (✓ hyper's default is generous), time to send headers (✓ not limited here), time to handle, concurrent connections (not limited here), page size (✓ 100), string lengths (✓ 64 characters), numeric ranges (✓).
2. **Authenticate before parsing the body** where you can: here the token check is a layer around the write routes, so an unauthenticated request never reaches the JSON extractor (✓ 401 for a request with a valid body but no token; the order of the layers matters).
3. **Do not echo secrets**: manual `Debug` impls for configs and clients (✓), no tokens in URLs (they end up in logs and browser history), no tokens in command-line arguments (Lesson 44).
4. **Don't leak internals in errors**: the 400 messages here repeat serde's wording; a `500` should say "internal error" and log the details (Lesson 43), with a request id the client can quote (not implemented).
5. **Server-side request forgery (SSRF)**: if your server fetches a URL a user gave you, restrict schemes and destinations; `reqwest` will happily request `http://127.0.0.1/…` or a cloud metadata address (from memory, not run).
6. **CORS and CSRF** matter if browsers call your API (not covered, not run).
7. **TLS** terminates somewhere: in the process or in a reverse proxy (not covered, not run). Header-injection and request-smuggling classes of bugs are why you want a well-tested HTTP implementation (hyper) rather than parsing HTTP yourself (from memory).
8. **Dependency hygiene** (Lesson 43): this dependency tree is much bigger than the CLI's. ✓ `Cargo.lock` lists 99 packages (the CLI of Lesson 44 had 28), two versions of `tower-http` among them.

---

## 11. Common mistakes

1. Returning `200 OK` with `{"error": ...}` in the body: clients, caches and monitoring read the status.
2. `unwrap()` on user input in a handler (§5).
3. Different error formats from different layers (§4.2).
4. Holding a `std::sync::MutexGuard` across `.await` (§4.1).
5. A `reqwest::Client` per request, or no timeout at all (§7.1, §7.2).
6. Retrying non-idempotent requests, retrying on `4xx`, or retrying forever with no backoff (§7.3).
7. Offsets for pagination over data that changes (§6.3).
8. Treating validation as "serde accepted it" (§6.2).
9. Reading the whole body into memory with no limit (§4.4).
10. Using `Debug` formatting on a request or config that contains a token.
11. Testing only the happy path, or only with sockets (slow) or only without (misses timeouts and shutdown).
12. Letting the server hang on shutdown: no deadline for in-flight requests.

---

## 12. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Choose the status code for each: a new order is created; the same `POST` is replayed with the same idempotency key; the JSON is missing a field; no token; the order id does not exist; the database is unreachable; the client is sending too many requests.
2. **(Warm-up)** Which of `GET`, `PUT`, `DELETE`, `POST` may a client retry automatically after a timeout, and what must be true for `POST`?
3. **(Easy)** Write an axum router with `GET /hello/{name}` returning `hello, NAME` (upper-cased if `?shout=true`), and test it with `oneshot`.
4. **(Medium)** Write a `middleware::from_fn` layer that adds an `x-request-id` header (a counter) to every response and test it.
5. **(Medium)** This handler does not compile as a Tokio handler. Explain why and give two fixes.
6. **(Medium-hard)** A teammate writes the idempotency check and the insert as two separate lock acquisitions "to keep the critical section small". What goes wrong?
7. **(Hard)** A client sends `POST /orders` with a key, the request times out after 5 s, and it retries. List every possible server-side state before the retry, and show that the result is correct in each.
8. **(Hard)** A handler does 20 ms of CPU work per request and the runtime has 2 worker threads. What is the maximum throughput if it runs directly in the handler? How do you raise it?

<details markdown="1">
<summary>Answers</summary>

1. Created: `201` (with `Location`). Replay with the same key and body: `200` with the original. Missing field: `400`. No token: `401` (with `WWW-Authenticate`). Unknown id: `404`. Database unreachable: `503` (and the client may retry). Too many requests: `429` (often with `Retry-After`).
2. `GET`, `PUT` and `DELETE` are idempotent by definition (the method's semantics, if the server implements them correctly), so they may be retried. `POST` may be retried only if the server provides a way to make repeats harmless, such as an idempotency key (✓ `ApiClient` retries `POST` only when it has one).
3. A possible answer:

   ```rust
   // needs: axum (feature "query"), serde, tokio, tower (util)
   use axum::body::Body;
   use axum::extract::{Path, Query};
   use axum::http::{Request, StatusCode};
   use axum::{Router, routing::get};
   use serde::Deserialize;
   use tower::ServiceExt;

   #[derive(Deserialize)]
   struct Opts {
       #[serde(default)]
       shout: bool,
   }

   async fn hello(Path(name): Path<String>, Query(o): Query<Opts>) -> String {
       let text = format!("hello, {name}");
       if o.shout { text.to_uppercase() } else { text }
   }

   #[tokio::main]
   async fn main() {
       let app = Router::new().route("/hello/{name}", get(hello));
       for (uri, want) in [("/hello/ada", "hello, ada"), ("/hello/ada?shout=true", "HELLO, ADA")] {
           let r = app.clone().oneshot(Request::get(uri).body(Body::empty()).unwrap()).await.unwrap();
           assert_eq!(r.status(), StatusCode::OK);
           let bytes = axum::body::to_bytes(r.into_body(), 1024).await.unwrap();
           assert_eq!(bytes, want);
       }
       let r = app.oneshot(Request::get("/hello/ada?shout=maybe").body(Body::empty()).unwrap()).await.unwrap();
       assert_eq!(r.status(), StatusCode::BAD_REQUEST);
       println!("ok");
   }
   ```

   Note the default rejection of a bad query value is `400` with a plain-text body; wrap the extractor as `apilab` does if you want your own error format.
4. A possible answer:

   ```rust
   // needs: axum, tokio, tower (util)
   use std::sync::Arc;
   use std::sync::atomic::{AtomicU64, Ordering};

   use axum::body::Body;
   use axum::extract::{Request, State};
   use axum::http::HeaderValue;
   use axum::middleware::{self, Next};
   use axum::response::Response;
   use axum::{Router, routing::get};
   use tower::ServiceExt;

   async fn request_id(State(counter): State<Arc<AtomicU64>>, req: Request, next: Next) -> Response {
       let id = counter.fetch_add(1, Ordering::Relaxed) + 1;
       let mut resp = next.run(req).await;
       resp.headers_mut().insert("x-request-id", HeaderValue::from_str(&id.to_string()).unwrap());
       resp
   }

   #[tokio::main]
   async fn main() {
       let counter = Arc::new(AtomicU64::new(0));
       let app = Router::new()
           .route("/", get(|| async { "hi" }))
           .layer(middleware::from_fn_with_state(counter, request_id));
       let mut ids = Vec::new();
       for _ in 0..3 {
           let r = app.clone().oneshot(Request::get("/").body(Body::empty()).unwrap()).await.unwrap();
           ids.push(r.headers()["x-request-id"].to_str().unwrap().to_string());
       }
       assert_eq!(ids, ["1", "2", "3"]);
       println!("ok");
   }
   ```

   ✓ The layer also wraps the router's default `404` (the check at the end of the code passes). A counter is fine for a demo; a real request id should be unguessable or at least unique across restarts.
5. The guard of a `std::sync::Mutex` is not `Send`, and a guard that lives across an `.await` becomes part of the future, which Tokio's multi-threaded runtime must be able to move between threads (✓ the compiler error in §4.1). Fix 1: end the guard's scope before the await (copy the data out, drop the guard, then await). Fix 2: use `tokio::sync::Mutex`, whose guard may be held across an await (at a cost, and with the risk of holding a lock while waiting on I/O). Prefer fix 1.
6. Two requests with the same key can both pass the "not seen" check before either inserts, so both create an order: the key does not protect against concurrent duplicates, which are exactly the case a client retry produces (a slow first request plus an impatient retry). Checking and inserting under one lock makes the pair atomic (✓ 32 concurrent requests, one order). With a database the equivalent is a unique constraint on the key, or a transaction.
7. States before the retry: (a) the first request never reached the server: the retry is the first, so it creates the order (`201`). (b) it reached the server and the order was created, but the response was lost or slow: the retry finds the key and returns the original order (`200`), no second order. (c) the first request is still running when the retry arrives: with the check and insert under one lock, one of them goes first and the other sees the key (✓ the 32-way test). (d) the first request was rejected (`400`): it did not consume the key (✓ test), so a corrected retry is a first request. (e) the retry has a different body: `409`. In every state there is at most one order for the key.
8. With 2 worker threads, each request occupies a worker for 20 ms, so at most 2 ÷ 0.02 = 100 requests/s (reasoning, not measured), and while a worker is busy it cannot poll other tasks, so latency of unrelated requests grows too. Move the CPU work to `tokio::task::spawn_blocking` (a separate pool, whose size you control) or to a dedicated thread pool, and keep the handler `async` only for I/O; to raise the ceiling further you need more CPU cores or a cheaper computation.

</details>

---

## 13. Debugging challenge

A teammate's server "works in manual tests", but clients report that some requests get **no response at all** (the connection just closes), that failures come back as `200 OK` with a text body, and that every redeploy drops requests that were in flight. Find and fix the problems.

```rust
// needs: axum, tokio
use axum::{Router, extract::Path, routing::get};

async fn square(Path(n): Path<String>) -> String {
    let n: u64 = n.parse().unwrap();
    match n.checked_mul(n) {
        Some(v) => v.to_string(),
        None => "error: too big".to_string(),
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/square/{n}", get(square));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

<details markdown="1">
<summary>Solution</summary>

Problems, in the order a client meets them:

1. `Path<String>` followed by `parse().unwrap()` turns user input into a **panic**. ✓ The connection is dropped with no response (§5); the client sees a transport error, not a `400`. Fix: extract `Path<u64>`: axum then answers `400` for `abc`, `-3` and `1.5` itself.
2. The overflow case returns `200 OK` and the string `error: too big`. ✓ Clients, caches and monitoring read the status code, not the text. Fix: return a `Result` whose error carries `400` (or your `ApiError`).
3. `axum::serve(...).await` has no shutdown path: stopping the process cuts requests that are in flight. Fix: `with_graceful_shutdown` with a future that completes on `ctrl_c` (and `SIGTERM` in a container; from memory).
4. Defense in depth: add `CatchPanicLayer` so that a future bug in any handler becomes `500` instead of a silent disconnect (✓ `500 Service panicked`).

```rust
// needs: axum, tokio (signal), tower-http (catch-panic)
use axum::http::StatusCode;
use axum::{Router, extract::Path, routing::get};
use tower_http::catch_panic::CatchPanicLayer;

async fn square(Path(n): Path<u64>) -> Result<String, (StatusCode, String)> {
    n.checked_mul(n).map(|v| v.to_string()).ok_or_else(|| {
        (StatusCode::BAD_REQUEST, format!("{n} squared does not fit in 64 bits"))
    })
}

fn app() -> Router {
    Router::new()
        .route("/square/{n}", get(square))
        .layer(CatchPanicLayer::new()) // last line of defense: a bug becomes a 500, not a dropped connection
}

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app())
        .with_graceful_shutdown(async {
            let _ = tokio::signal::ctrl_c().await;
        })
        .await
        .unwrap();
}
```

✓ I ran the fixed `app()` on a real socket (port 0) and checked: `/square/12` gives `200` and `144`; `abc`, `-3` and `1.5` each give `400`; `/square/99999999999` gives `400` with a message containing "does not fit"; and a shutdown signal makes the server task finish. The `ctrl_c` future itself I compiled but did not trigger (no signal was sent), so `main` in the listing above is what you would ship and the test used a channel instead.

</details>

---

## 14. Interview questions

1. *What does "idempotent" mean for HTTP methods, and why does a retrying client care?*
2. *How do you make `POST` safe to retry?*
3. *Walk through the life of a request in axum: from socket to response.*
4. *Why must the `Json` extractor be last, and what does the compiler tell you if it is not?*
5. *Why should an API return one error format everywhere, and which errors escape your handlers?*
6. *What timeouts does an HTTP client and server each need?*
7. *How should a client retry, and what should it never retry?*
8. *Cursor or offset pagination?*
9. *Why can't you hold a `std::sync::MutexGuard` across `.await` in a handler?*
10. *How do you test an HTTP service quickly and also test timeouts and shutdown?*

<details markdown="1">
<summary>Answers</summary>

1. Repeating the request has the same effect on the server as sending it once (`GET`, `PUT`, `DELETE`, by their semantics). The network cannot tell a client whether a timed-out request ran, so the client may only resend requests whose repetition is harmless.
2. Give each logical operation a client-chosen key, store the first result under the key on the server, and return that result for repeats with the same body (`409` for a different body). The check and the write must be atomic, and invalid requests must not consume the key (✓ all three tested).
3. hyper accepts the connection and parses the request; the request passes through the stack of tower layers (body limit, timeout, middleware); the router matches method and path; extractors run in argument order, the body-consuming one last; the handler runs; its return value becomes a response through `IntoResponse`; the layers see the response on the way out; hyper writes it, and may keep the connection open for the next request.
4. It consumes the request body, which can be read only once, so any extractor after it would have nothing to read. Without `#[axum::debug_handler]` the compiler says the `Handler` trait is not implemented for the function; with the macro it says ``Json<_> consumes the request body and thus must be the last argument to the handler function`` (✓).
5. Clients write one error parser. Errors made outside your handlers (path and query rejections, 405 from the router, timeouts, body-limit failures) use the framework's format unless you wrap the extractors or rewrite the responses (✓ my first version had three of them in the wrong format).
6. Server: a limit on body size, on time to receive headers, on handler time, and on concurrency. Client: a connect timeout and a total timeout per attempt. Without them one slow peer holds a connection (✓ the server waited 8 s on a silent client; the client had no default total timeout).
7. Retry idempotent requests on transient conditions (connection failure, timeout, `429`, `502`, `503`, `504`), a bounded number of times, with exponential backoff and jitter, and honor `Retry-After` if present (jitter and `Retry-After` from memory, not implemented). Never retry `4xx` (the request is wrong), `500` blindly, or non-idempotent requests without a key.
8. Cursors (keyset): stable when rows are inserted or removed during the walk, and cheap for the database. Offsets are simple and allow random access to page *n*, but skip or repeat items under change and get slower with depth (from memory).
9. The guard is not `Send`; a future holding it across an await is not `Send`; Tokio's multi-threaded runtime requires handler futures to be `Send` because the task may resume on another thread (✓ the compiler error). Even a `Send` guard (a Tokio mutex) held across slow I/O serializes requests.
10. Run requests through the router in memory with `oneshot` for status, body and header checks (✓ 1.5 to 1.8 µs per call, 16 tests in about 10 ms); use real sockets on port 0 for timeouts, refused connections, concurrency and graceful shutdown (✓ 15 tests, about 0.6 s), with generous timing margins and a flaky-server helper for retry policy.

</details>

---

## 15. Practical challenge: `apilab`

Build the crate `apilab`. Requirements:

1. Types `NewOrder` (`deny_unknown_fields`, camelCase), `Order`, `Page`, and `NewOrder::validate` with the limits above.
2. One `ApiError` enum and `ErrorBody` envelope; `ApiJson`, `ApiQuery`, `ApiPath` wrappers; framework errors (405, 408) wrapped by a `map_response`.
3. Routes: `GET /health`, `GET /sleep/{ms}` (diagnostic, capped at 5 s), `POST /orders` (bearer token, optional `Idempotency-Key`, `201` plus `Location`), `GET /orders/{id}`, `GET /orders?after&limit`.
4. A 1 KiB body limit, a configurable request timeout, graceful shutdown through `serve`.
5. A client with per-attempt timeouts, bounded retries with capped exponential backoff, retries only for `GET` or keyed `POST`, and an error type that separates transport, API and unexpected responses.
6. Tests: in memory (`oneshot`) and over real sockets, including concurrency and shutdown.
7. Your own mutation checks.

<details markdown="1">
<summary>Solution</summary>

`Cargo.toml`:

```toml
[package]
name = "apilab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false

[dependencies]
axum = { version = "0.8", default-features = false, features = ["http1", "json", "tokio", "query"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net", "signal", "time", "sync"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.13", default-features = false, features = ["json"] }
tower-http = { version = "0.7", features = ["timeout", "limit"] }

[dev-dependencies]
tower = { version = "0.5", features = ["util"] }
http-body-util = "0.1"
tokio = { version = "1", features = ["io-util"] }
```

`src/lib.rs`:

````rust
//! `apilab`: a small JSON HTTP API (server and client) used to study request
//! handling, error design, limits, timeouts, idempotency and graceful shutdown.
#![deny(missing_docs)]

pub mod client;

use std::collections::{BTreeMap, HashMap};
use std::future::Future;
use std::sync::{Arc, Mutex};
use std::time::Duration;

use axum::extract::rejection::{JsonRejection, QueryRejection};
use axum::extract::{DefaultBodyLimit, FromRequest, FromRequestParts, Path, Request, State};
use axum::http::{HeaderMap, StatusCode, header};
use axum::middleware::{self, Next};
use axum::response::{IntoResponse, Response};
use axum::routing::{get, post};
use axum::{Json, Router};
use serde::{Deserialize, Serialize};
use tower_http::timeout::TimeoutLayer;

/// Largest accepted request body, in bytes.
pub const MAX_BODY_BYTES: usize = 1024;
/// Longest accepted customer name, in characters.
pub const MAX_CUSTOMER_CHARS: usize = 64;
/// Largest accepted amount, in cents.
pub const MAX_AMOUNT_CENTS: u64 = 1_000_000_000_000;
/// Longest accepted idempotency key, in bytes.
pub const MAX_KEY_BYTES: usize = 64;
/// Default and largest page size for listings.
pub const DEFAULT_LIMIT: usize = 20;
/// Largest page size.
pub const MAX_LIMIT: usize = 100;

// ---- data model ---------------------------------------------------------

/// The body of `POST /orders`.
#[derive(Debug, Clone, PartialEq, Eq, Deserialize, Serialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct NewOrder {
    /// Customer name, 1 to [`MAX_CUSTOMER_CHARS`] characters.
    pub customer: String,
    /// Amount in cents, at most [`MAX_AMOUNT_CENTS`].
    pub amount_cents: u64,
}

/// A stored order.
#[derive(Debug, Clone, PartialEq, Eq, Deserialize, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Order {
    /// Server-assigned identifier.
    pub id: u64,
    /// Customer name.
    pub customer: String,
    /// Amount in cents.
    pub amount_cents: u64,
}

/// One page of a listing.
#[derive(Debug, Clone, PartialEq, Eq, Deserialize, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Page {
    /// The orders on this page, ascending by id.
    pub items: Vec<Order>,
    /// Pass this as `after` to get the next page; absent on the last page.
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub next_after: Option<u64>,
}

impl NewOrder {
    /// Checks the rules the types cannot express.
    pub fn validate(&self) -> Result<(), ApiError> {
        let chars = self.customer.chars().count();
        if chars == 0 || chars > MAX_CUSTOMER_CHARS {
            return Err(ApiError::invalid(format!(
                "customer must be 1 to {MAX_CUSTOMER_CHARS} characters, got {chars}"
            )));
        }
        if self.amount_cents > MAX_AMOUNT_CENTS {
            return Err(ApiError::invalid(format!(
                "amountCents must be at most {MAX_AMOUNT_CENTS}"
            )));
        }
        Ok(())
    }
}

// ---- errors ---------------------------------------------------------------

/// Every way a request can fail, with one stable machine-readable code each.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ApiError {
    /// The request is malformed or breaks a rule (400).
    Invalid(String),
    /// Missing or wrong credentials (401).
    Unauthorized,
    /// No such resource (404).
    NotFound,
    /// The idempotency key was already used with a different body (409).
    KeyConflict,
    /// The body is larger than [`MAX_BODY_BYTES`] (413).
    TooLarge,
    /// The body is not JSON (415).
    UnsupportedMediaType,
    /// The path exists but not for this method (405).
    MethodNotAllowed,
    /// The handler did not finish in time (408).
    Timeout,
}

impl ApiError {
    fn invalid(message: impl Into<String>) -> Self {
        ApiError::Invalid(message.into())
    }

    /// The HTTP status of this error.
    pub fn status(&self) -> StatusCode {
        match self {
            ApiError::Invalid(_) => StatusCode::BAD_REQUEST,
            ApiError::Unauthorized => StatusCode::UNAUTHORIZED,
            ApiError::NotFound => StatusCode::NOT_FOUND,
            ApiError::KeyConflict => StatusCode::CONFLICT,
            ApiError::TooLarge => StatusCode::PAYLOAD_TOO_LARGE,
            ApiError::UnsupportedMediaType => StatusCode::UNSUPPORTED_MEDIA_TYPE,
            ApiError::MethodNotAllowed => StatusCode::METHOD_NOT_ALLOWED,
            ApiError::Timeout => StatusCode::REQUEST_TIMEOUT,
        }
    }

    /// The stable code clients should switch on.
    pub fn code(&self) -> &'static str {
        match self {
            ApiError::Invalid(_) => "invalid_request",
            ApiError::Unauthorized => "unauthorized",
            ApiError::NotFound => "not_found",
            ApiError::KeyConflict => "idempotency_key_reused",
            ApiError::TooLarge => "body_too_large",
            ApiError::UnsupportedMediaType => "unsupported_media_type",
            ApiError::MethodNotAllowed => "method_not_allowed",
            ApiError::Timeout => "request_timeout",
        }
    }

    fn message(&self) -> String {
        match self {
            ApiError::Invalid(m) => m.clone(),
            ApiError::Unauthorized => "missing or invalid bearer token".into(),
            ApiError::NotFound => "no such resource".into(),
            ApiError::KeyConflict => "this Idempotency-Key was used with a different body".into(),
            ApiError::TooLarge => format!("body is larger than {MAX_BODY_BYTES} bytes"),
            ApiError::UnsupportedMediaType => "content-type must be application/json".into(),
            ApiError::MethodNotAllowed => "method not allowed for this path".into(),
            ApiError::Timeout => "the request took too long".into(),
        }
    }
}

/// The JSON error envelope: `{"error":{"code":"...","message":"..."}}`.
#[derive(Debug, Clone, PartialEq, Eq, Deserialize, Serialize)]
pub struct ErrorBody {
    /// The error details.
    pub error: ErrorDetail,
}

/// Code and message inside [`ErrorBody`].
#[derive(Debug, Clone, PartialEq, Eq, Deserialize, Serialize)]
pub struct ErrorDetail {
    /// Stable machine-readable code.
    pub code: String,
    /// Human-readable explanation; never relied on by programs.
    pub message: String,
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let body = ErrorBody {
            error: ErrorDetail {
                code: self.code().into(),
                message: self.message(),
            },
        };
        let mut response = (self.status(), Json(body)).into_response();
        if self == ApiError::Unauthorized {
            response
                .headers_mut()
                .insert(header::WWW_AUTHENTICATE, "Bearer".parse().unwrap());
        }
        response
    }
}

// ---- extractors that speak our error format --------------------------------

/// `Json<T>` whose rejections use [`ApiError`].
pub struct ApiJson<T>(pub T);

impl<S: Send + Sync, T: serde::de::DeserializeOwned> FromRequest<S> for ApiJson<T> {
    type Rejection = ApiError;

    async fn from_request(req: Request, state: &S) -> Result<Self, ApiError> {
        match Json::<T>::from_request(req, state).await {
            Ok(Json(v)) => Ok(ApiJson(v)),
            Err(JsonRejection::MissingJsonContentType(_)) => Err(ApiError::UnsupportedMediaType),
            Err(JsonRejection::BytesRejection(e))
                if e.status() == StatusCode::PAYLOAD_TOO_LARGE =>
            {
                Err(ApiError::TooLarge)
            }
            Err(e) => Err(ApiError::invalid(e.body_text())),
        }
    }
}

/// `Query<T>` whose rejections use [`ApiError`].
pub struct ApiQuery<T>(pub T);

impl<S: Send + Sync, T: serde::de::DeserializeOwned> FromRequestParts<S> for ApiQuery<T> {
    type Rejection = ApiError;

    async fn from_request_parts(
        parts: &mut axum::http::request::Parts,
        state: &S,
    ) -> Result<Self, ApiError> {
        axum::extract::Query::<T>::from_request_parts(parts, state)
            .await
            .map(|axum::extract::Query(v)| ApiQuery(v))
            .map_err(|e: QueryRejection| ApiError::invalid(e.body_text()))
    }
}

/// `Path<T>` whose rejections use [`ApiError`].
pub struct ApiPath<T>(pub T);

impl<S: Send + Sync, T: serde::de::DeserializeOwned + Send> FromRequestParts<S> for ApiPath<T> {
    type Rejection = ApiError;

    async fn from_request_parts(
        parts: &mut axum::http::request::Parts,
        state: &S,
    ) -> Result<Self, ApiError> {
        Path::<T>::from_request_parts(parts, state)
            .await
            .map(|Path(v)| ApiPath(v))
            .map_err(|e| ApiError::invalid(e.body_text()))
    }
}

/// Gives the empty 405 and 408 responses produced by the router and the
/// timeout layer the same JSON envelope as every other error. (The router adds
/// the `Allow` header to its 405 answers itself, after this runs.)
async fn envelope_framework_errors(response: Response) -> Response {
    match response.status() {
        StatusCode::METHOD_NOT_ALLOWED => ApiError::MethodNotAllowed.into_response(),
        StatusCode::REQUEST_TIMEOUT => ApiError::Timeout.into_response(),
        _ => response,
    }
}

// ---- state -------------------------------------------------------------------

#[derive(Default)]
struct Store {
    next_id: u64,
    orders: BTreeMap<u64, Order>,
    keys: HashMap<String, (NewOrder, u64)>,
}

/// Server configuration. `Debug` never prints the token.
#[derive(Clone)]
pub struct Config {
    /// The bearer token required by write endpoints.
    pub token: String,
    /// Per-request time limit for the whole handler.
    pub request_timeout: Duration,
}

impl std::fmt::Debug for Config {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Config")
            .field("token", &"<redacted>")
            .field("request_timeout", &self.request_timeout)
            .finish()
    }
}

#[derive(Clone)]
struct AppState {
    store: Arc<Mutex<Store>>,
    config: Arc<Config>,
}

/// Compares two byte strings without stopping at the first difference.
///
/// This removes the most obvious timing difference; it is hygiene, not a
/// proof of constant-time behavior on every platform.
pub fn tokens_equal(a: &[u8], b: &[u8]) -> bool {
    let mut diff = (a.len() ^ b.len()) as u64;
    for i in 0..a.len().max(b.len()) {
        let x = a.get(i).copied().unwrap_or(0);
        let y = b.get(i).copied().unwrap_or(0);
        diff |= u64::from(x ^ y);
    }
    diff == 0
}

fn bearer(headers: &HeaderMap) -> Option<&str> {
    headers
        .get(header::AUTHORIZATION)?
        .to_str()
        .ok()?
        .strip_prefix("Bearer ")
}

async fn require_token(State(state): State<AppState>, req: Request, next: Next) -> Response {
    match bearer(req.headers()) {
        Some(t) if tokens_equal(t.as_bytes(), state.config.token.as_bytes()) => next.run(req).await,
        _ => ApiError::Unauthorized.into_response(),
    }
}

// ---- handlers ----------------------------------------------------------------

async fn health() -> &'static str {
    "ok"
}

async fn create_order(
    State(state): State<AppState>,
    headers: HeaderMap,
    ApiJson(new): ApiJson<NewOrder>,
) -> Result<Response, ApiError> {
    new.validate()?;
    let key = match headers.get("idempotency-key") {
        None => None,
        Some(v) => {
            let k = v
                .to_str()
                .ok()
                .filter(|k| !k.is_empty() && k.len() <= MAX_KEY_BYTES)
                .ok_or_else(|| {
                    ApiError::invalid(format!(
                        "Idempotency-Key must be 1 to {MAX_KEY_BYTES} visible ASCII bytes"
                    ))
                })?;
            Some(k.to_owned())
        }
    };
    let mut store = state.store.lock().unwrap_or_else(|p| p.into_inner());
    if let Some(k) = &key {
        if let Some((first, id)) = store.keys.get(k) {
            if *first != new {
                return Err(ApiError::KeyConflict);
            }
            let order = store.orders[id].clone();
            return Ok((StatusCode::OK, Json(order)).into_response());
        }
    }
    store.next_id += 1;
    let id = store.next_id;
    let order = Order {
        id,
        customer: new.customer.clone(),
        amount_cents: new.amount_cents,
    };
    store.orders.insert(id, order.clone());
    if let Some(k) = key {
        store.keys.insert(k, (new, id));
    }
    let location = format!("/orders/{id}");
    Ok((
        StatusCode::CREATED,
        [(header::LOCATION, location)],
        Json(order),
    )
        .into_response())
}

async fn get_order(
    State(state): State<AppState>,
    ApiPath(id): ApiPath<u64>,
) -> Result<Json<Order>, ApiError> {
    let store = state.store.lock().unwrap_or_else(|p| p.into_inner());
    store
        .orders
        .get(&id)
        .cloned()
        .map(Json)
        .ok_or(ApiError::NotFound)
}

#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
struct ListParams {
    after: Option<u64>,
    limit: Option<usize>,
}

async fn list_orders(
    State(state): State<AppState>,
    ApiQuery(p): ApiQuery<ListParams>,
) -> Result<Json<Page>, ApiError> {
    let limit = p.limit.unwrap_or(DEFAULT_LIMIT);
    if !(1..=MAX_LIMIT).contains(&limit) {
        return Err(ApiError::invalid(format!("limit must be 1 to {MAX_LIMIT}")));
    }
    let store = state.store.lock().unwrap_or_else(|p| p.into_inner());
    let start = p.after.unwrap_or(0);
    let mut items: Vec<Order> = store
        .orders
        .range(start.saturating_add(1)..)
        .take(limit + 1)
        .map(|(_, o)| o.clone())
        .collect();
    let next_after = if items.len() > limit {
        items.truncate(limit);
        items.last().map(|o| o.id)
    } else {
        None
    };
    Ok(Json(Page { items, next_after }))
}

/// Longest sleep the diagnostics route `/sleep/{ms}` accepts.
pub const MAX_SLEEP_MS: u64 = 5_000;

async fn sleep_ms(ApiPath(ms): ApiPath<u64>) -> Result<&'static str, ApiError> {
    if ms > MAX_SLEEP_MS {
        return Err(ApiError::invalid(format!(
            "ms must be at most {MAX_SLEEP_MS}"
        )));
    }
    tokio::time::sleep(Duration::from_millis(ms)).await;
    Ok("slept")
}

async fn fallback() -> ApiError {
    ApiError::NotFound
}

/// Builds the application router.
pub fn app(config: Config) -> Router {
    let timeout = config.request_timeout;
    let state = AppState {
        store: Arc::default(),
        config: Arc::new(config),
    };
    let writes = Router::new()
        .route("/orders", post(create_order))
        .route_layer(middleware::from_fn_with_state(state.clone(), require_token));
    let reads = Router::new()
        .route("/health", get(health))
        .route("/sleep/{ms}", get(sleep_ms))
        .route("/orders", get(list_orders))
        .route("/orders/{id}", get(get_order));
    Router::new()
        .merge(writes)
        .merge(reads)
        .fallback(fallback)
        .layer(DefaultBodyLimit::max(MAX_BODY_BYTES))
        .layer(TimeoutLayer::with_status_code(
            StatusCode::REQUEST_TIMEOUT,
            timeout,
        ))
        .layer(middleware::map_response(envelope_framework_errors))
        .with_state(state)
}

/// Serves `app` on `listener` until `shutdown` completes, then lets in-flight
/// requests finish.
pub async fn serve(
    listener: tokio::net::TcpListener,
    app: Router,
    shutdown: impl Future<Output = ()> + Send + 'static,
) -> std::io::Result<()> {
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown)
        .await
}
````

`src/client.rs`:

````rust
//! A small typed client with timeouts, bounded retries and exponential backoff.

use std::time::Duration;

use reqwest::{Method, StatusCode};

use crate::{ErrorDetail, NewOrder, Order, Page};

/// What can go wrong when calling the API.
#[derive(Debug)]
pub enum ClientError {
    /// No usable HTTP response: connection refused, timeout, reset, bad body.
    Transport(reqwest::Error),
    /// The server answered with its JSON error envelope.
    Api {
        /// HTTP status.
        status: u16,
        /// Code and message from the server.
        error: ErrorDetail,
    },
    /// A non-success status whose body was not the error envelope (proxies, gateways).
    Unexpected {
        /// HTTP status.
        status: u16,
        /// First bytes of the body, for diagnostics.
        body: String,
    },
}

impl std::fmt::Display for ClientError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ClientError::Transport(e) => write!(f, "transport error: {e}"),
            ClientError::Api { status, error } => {
                write!(f, "api error {status} {}: {}", error.code, error.message)
            }
            ClientError::Unexpected { status, body } => {
                write!(f, "unexpected response {status}: {body}")
            }
        }
    }
}

impl std::error::Error for ClientError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ClientError::Transport(e) => Some(e),
            _ => None,
        }
    }
}

/// Client settings.
#[derive(Clone)]
pub struct ClientConfig {
    /// Base URL without a trailing slash, e.g. `http://127.0.0.1:3000`.
    pub base_url: String,
    /// Bearer token for write calls.
    pub token: Option<String>,
    /// Time limit for one attempt, connect to last byte.
    pub timeout: Duration,
    /// Extra attempts after the first (0 = no retry).
    pub retries: u32,
    /// Delay before the first retry; doubles each time, capped at 2 seconds.
    pub backoff: Duration,
}

impl std::fmt::Debug for ClientConfig {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("ClientConfig")
            .field("base_url", &self.base_url)
            .field("token", &self.token.as_ref().map(|_| "<redacted>"))
            .field("timeout", &self.timeout)
            .field("retries", &self.retries)
            .field("backoff", &self.backoff)
            .finish()
    }
}

/// The API client. Cloning is cheap and shares the connection pool.
#[derive(Debug, Clone)]
pub struct ApiClient {
    config: ClientConfig,
    http: reqwest::Client,
}

/// Whether another attempt could change the outcome.
fn retryable_status(status: StatusCode) -> bool {
    matches!(status.as_u16(), 429 | 502 | 503 | 504)
}

/// Delay before retry number `attempt` (0-based): `base * 2^attempt`, at most 2 s.
pub fn backoff_delay(base: Duration, attempt: u32) -> Duration {
    let factor = 1u32.checked_shl(attempt).unwrap_or(u32::MAX);
    base.saturating_mul(factor).min(Duration::from_secs(2))
}

impl ApiClient {
    /// Builds a client; fails only if the HTTP stack cannot be initialized.
    pub fn new(config: ClientConfig) -> Result<Self, ClientError> {
        let http = reqwest::Client::builder()
            .timeout(config.timeout)
            .connect_timeout(config.timeout)
            .build()
            .map_err(ClientError::Transport)?;
        Ok(ApiClient { config, http })
    }

    async fn send<B: serde::Serialize + ?Sized>(
        &self,
        method: Method,
        path: &str,
        body: Option<&B>,
        idempotency_key: Option<&str>,
    ) -> Result<reqwest::Response, ClientError> {
        // A request may be repeated only if repeating it is harmless.
        let may_retry = method == Method::GET || idempotency_key.is_some();
        let attempts = if may_retry {
            self.config.retries + 1
        } else {
            1
        };
        let mut last = None;
        for attempt in 0..attempts {
            if attempt > 0 {
                tokio::time::sleep(backoff_delay(self.config.backoff, attempt - 1)).await;
            }
            let mut req = self
                .http
                .request(method.clone(), format!("{}{path}", self.config.base_url));
            if let Some(t) = &self.config.token {
                req = req.bearer_auth(t);
            }
            if let Some(k) = idempotency_key {
                req = req.header("Idempotency-Key", k);
            }
            if let Some(b) = body {
                req = req.json(b);
            }
            match req.send().await {
                Ok(resp) if retryable_status(resp.status()) && attempt + 1 < attempts => {
                    last = Some(Ok(resp));
                }
                Ok(resp) => return Ok(resp),
                Err(e) if attempt + 1 < attempts && (e.is_connect() || e.is_timeout()) => {
                    last = Some(Err(e));
                }
                Err(e) => return Err(ClientError::Transport(e)),
            }
        }
        match last {
            Some(Ok(resp)) => Ok(resp),
            Some(Err(e)) => Err(ClientError::Transport(e)),
            None => unreachable!("at least one attempt is always made"),
        }
    }

    async fn decode<T: serde::de::DeserializeOwned>(
        resp: reqwest::Response,
    ) -> Result<T, ClientError> {
        let status = resp.status();
        if status.is_success() {
            return resp.json::<T>().await.map_err(ClientError::Transport);
        }
        let text = resp.text().await.map_err(ClientError::Transport)?;
        match serde_json::from_str::<crate::ErrorBody>(&text) {
            Ok(b) => Err(ClientError::Api {
                status: status.as_u16(),
                error: b.error,
            }),
            Err(_) => Err(ClientError::Unexpected {
                status: status.as_u16(),
                body: text.chars().take(200).collect(),
            }),
        }
    }

    /// `POST /orders`. Pass a key to make retries (and your own re-sends) safe.
    pub async fn create_order(
        &self,
        new: &NewOrder,
        idempotency_key: Option<&str>,
    ) -> Result<Order, ClientError> {
        let resp = self
            .send(Method::POST, "/orders", Some(new), idempotency_key)
            .await?;
        Self::decode(resp).await
    }

    /// `GET /orders/{id}`.
    pub async fn get_order(&self, id: u64) -> Result<Order, ClientError> {
        let resp = self
            .send::<()>(Method::GET, &format!("/orders/{id}"), None, None)
            .await?;
        Self::decode(resp).await
    }

    /// `GET /orders?after=..&limit=..`.
    pub async fn list_orders(
        &self,
        after: Option<u64>,
        limit: Option<usize>,
    ) -> Result<Page, ClientError> {
        let mut path = String::from("/orders");
        let mut sep = '?';
        if let Some(a) = after {
            path.push_str(&format!("{sep}after={a}"));
            sep = '&';
        }
        if let Some(l) = limit {
            path.push_str(&format!("{sep}limit={l}"));
        }
        let resp = self.send::<()>(Method::GET, &path, None, None).await?;
        Self::decode(resp).await
    }
}
````

`tests/api.rs`:

````rust
//! In-memory tests: requests go straight into the router, no sockets.

use std::time::Duration;

use apilab::{
    ApiError, Config, ErrorBody, MAX_BODY_BYTES, MAX_CUSTOMER_CHARS, Order, Page, app,
    client::backoff_delay, tokens_equal,
};
use axum::Router;
use axum::body::Body;
use axum::http::{HeaderMap, Method, Request, StatusCode};
use http_body_util::BodyExt;
use serde_json::json;
use tower::ServiceExt;

const TOKEN: &str = "s3cret-token";

fn router() -> Router {
    app(Config {
        token: TOKEN.into(),
        request_timeout: Duration::from_secs(5),
    })
}

struct Reply {
    status: StatusCode,
    headers: HeaderMap,
    body: Vec<u8>,
}

impl Reply {
    fn json<T: serde::de::DeserializeOwned>(&self) -> T {
        serde_json::from_slice(&self.body)
            .unwrap_or_else(|e| panic!("{e}: {}", String::from_utf8_lossy(&self.body)))
    }
    fn code(&self) -> String {
        self.json::<ErrorBody>().error.code
    }
}

async fn call(
    app: &Router,
    method: Method,
    uri: &str,
    extra: &[(&str, &str)],
    body: Option<Vec<u8>>,
) -> Reply {
    let mut req = Request::builder().method(method).uri(uri);
    for (k, v) in extra {
        req = req.header(*k, *v);
    }
    let req = req.body(Body::from(body.unwrap_or_default())).unwrap();
    let resp = app.clone().oneshot(req).await.unwrap();
    let (parts, body) = resp.into_parts();
    Reply {
        status: parts.status,
        headers: parts.headers,
        body: body.collect().await.unwrap().to_bytes().to_vec(),
    }
}

async fn get(app: &Router, uri: &str) -> Reply {
    call(app, Method::GET, uri, &[], None).await
}

async fn post_raw(app: &Router, extra: &[(&str, &str)], body: Vec<u8>) -> Reply {
    let mut headers = vec![("authorization", format!("Bearer {TOKEN}"))];
    headers.push(("content-type", "application/json".into()));
    let mut all: Vec<(&str, &str)> = headers.iter().map(|(k, v)| (*k, v.as_str())).collect();
    all.extend_from_slice(extra);
    call(app, Method::POST, "/orders", &all, Some(body)).await
}

async fn post(app: &Router, body: serde_json::Value) -> Reply {
    post_raw(app, &[], serde_json::to_vec(&body).unwrap()).await
}

fn order(customer: &str, cents: u64) -> serde_json::Value {
    json!({"customer": customer, "amountCents": cents})
}

#[tokio::test]
async fn health_and_unknown_paths() {
    let app = router();
    let r = get(&app, "/health").await;
    assert_eq!((r.status, r.body.as_slice()), (StatusCode::OK, &b"ok"[..]));
    let r = get(&app, "/nope").await;
    assert_eq!(r.status, StatusCode::NOT_FOUND);
    assert_eq!(r.code(), "not_found");
}

#[tokio::test]
async fn wrong_method_gets_the_json_envelope_and_keeps_allow() {
    let r = call(&router(), Method::DELETE, "/orders", &[], None).await;
    assert_eq!(r.status, StatusCode::METHOD_NOT_ALLOWED);
    assert_eq!(r.code(), "method_not_allowed");
    let allow = r
        .headers
        .get("allow")
        .expect("Allow header kept")
        .to_str()
        .unwrap();
    assert!(allow.contains("GET") && allow.contains("POST"), "{allow}");
}

#[tokio::test]
async fn writes_need_the_right_bearer_token_reads_do_not() {
    let app = router();
    let body = serde_json::to_vec(&order("ada", 1)).unwrap();
    for auth in [
        None,
        Some("Bearer nope"),
        Some("Basic abc"),
        Some(TOKEN),
        Some("Bearer s3cret-tokeN"),
        Some("Bearer s3cret-toke"),
        Some("bearer s3cret-token"),
    ] {
        let mut h = vec![("content-type", "application/json")];
        if let Some(a) = auth {
            h.push(("authorization", a));
        }
        let r = call(&app, Method::POST, "/orders", &h, Some(body.clone())).await;
        assert_eq!(r.status, StatusCode::UNAUTHORIZED, "{auth:?}");
        assert_eq!(r.headers["www-authenticate"], "Bearer");
        assert_eq!(r.code(), "unauthorized");
    }
    assert_eq!(get(&app, "/orders").await.status, StatusCode::OK);
    // nothing was created by the rejected calls
    assert!(get(&app, "/orders").await.json::<Page>().items.is_empty());
}

#[tokio::test]
async fn create_then_get_and_list() {
    let app = router();
    let r = post(&app, order("ada", 1250)).await;
    assert_eq!(r.status, StatusCode::CREATED);
    assert_eq!(r.headers["location"], "/orders/1");
    let created: Order = r.json();
    assert_eq!(
        created,
        Order {
            id: 1,
            customer: "ada".into(),
            amount_cents: 1250
        }
    );
    assert_eq!(get(&app, "/orders/1").await.json::<Order>(), created);
    let missing = get(&app, "/orders/2").await;
    assert_eq!(
        (missing.status, missing.code().as_str()),
        (StatusCode::NOT_FOUND, "not_found")
    );
    let bad = get(&app, "/orders/abc").await;
    assert_eq!(bad.status, StatusCode::BAD_REQUEST);
    assert_eq!(bad.code(), "invalid_request");
}

#[tokio::test]
async fn pagination_walks_every_order_exactly_once() {
    let app = router();
    for i in 0..5 {
        post(&app, order(&format!("c{i}"), i)).await;
    }
    let mut seen = Vec::new();
    let mut after: Option<u64> = None;
    let mut pages = 0;
    loop {
        let uri = match after {
            Some(a) => format!("/orders?after={a}&limit=2"),
            None => "/orders?limit=2".into(),
        };
        let page: Page = get(&app, &uri).await.json();
        assert!(page.items.len() <= 2);
        seen.extend(page.items.iter().map(|o| o.id));
        pages += 1;
        match page.next_after {
            Some(n) => after = Some(n),
            None => break,
        }
    }
    assert_eq!(seen, vec![1, 2, 3, 4, 5]);
    assert_eq!(pages, 3);
    // an exactly full last page has no next link
    let page: Page = get(&app, "/orders?limit=5").await.json();
    assert_eq!((page.items.len(), page.next_after), (5, None));
    // the last page has no "nextAfter" key at all (not even null)
    let raw: serde_json::Value = get(&app, "/orders?limit=5").await.json();
    assert!(raw.get("nextAfter").is_none(), "{raw}");
    let raw: serde_json::Value = get(&app, "/orders?limit=2").await.json();
    assert_eq!(raw["nextAfter"], 2);
    // past the end and at the integer limit
    for after in ["5", "100", &u64::MAX.to_string()] {
        let page: Page = get(&app, &format!("/orders?after={after}")).await.json();
        assert!(
            page.items.is_empty() && page.next_after.is_none(),
            "after={after}"
        );
    }
}

#[tokio::test]
async fn query_validation() {
    let app = router();
    for ok in ["limit=1", "limit=100", "after=0&limit=20"] {
        assert_eq!(
            get(&app, &format!("/orders?{ok}")).await.status,
            StatusCode::OK,
            "{ok}"
        );
    }
    for bad in [
        "limit=0",
        "limit=101",
        "limit=-1",
        "limit=abc",
        "after=x",
        "foo=1",
        "limit=1&limit=2x",
    ] {
        let r = get(&app, &format!("/orders?{bad}")).await;
        assert_eq!(r.status, StatusCode::BAD_REQUEST, "{bad}");
        assert_eq!(r.code(), "invalid_request", "{bad}");
    }
}

#[tokio::test]
async fn body_validation_boundaries() {
    let app = router();
    let name64 = "é".repeat(MAX_CUSTOMER_CHARS);
    assert_eq!(
        post(&app, order(&name64, 1)).await.status,
        StatusCode::CREATED
    );
    for (name, cents) in [("", 1), (&"x".repeat(65), 1), ("a", 1_000_000_000_001)] {
        let r = post(&app, order(name, cents)).await;
        assert_eq!(
            (r.status, r.code().as_str()),
            (StatusCode::BAD_REQUEST, "invalid_request")
        );
    }
    assert_eq!(
        post(&app, order("a", 1_000_000_000_000)).await.status,
        StatusCode::CREATED
    );
    for bad in [
        json!({"customer": "a"}),
        json!({"customer": "a", "amountCents": "1"}),
        json!({"customer": "a", "amountCents": -1}),
        json!({"customer": "a", "amountCents": 1.5}),
        json!({"customer": "a", "amountCents": 1, "extra": true}),
        json!({"customer": null, "amountCents": 1}),
        json!([]),
    ] {
        let r = post(&app, bad.clone()).await;
        assert_eq!(
            (r.status, r.code().as_str()),
            (StatusCode::BAD_REQUEST, "invalid_request"),
            "{bad}"
        );
    }
    let r = post_raw(&app, &[], b"{oops".to_vec()).await;
    assert_eq!(
        (r.status, r.code().as_str()),
        (StatusCode::BAD_REQUEST, "invalid_request")
    );
    // only the two valid creations above were stored
    assert_eq!(get(&app, "/orders").await.json::<Page>().items.len(), 2);
}

#[tokio::test]
async fn content_type_is_required_to_be_json() {
    let app = router();
    let body = serde_json::to_vec(&order("a", 1)).unwrap();
    for ct in [
        None,
        Some("text/plain"),
        Some("application/x-www-form-urlencoded"),
    ] {
        let mut h = vec![("authorization", "Bearer s3cret-token")];
        if let Some(c) = ct {
            h.push(("content-type", c));
        }
        let r = call(&app, Method::POST, "/orders", &h, Some(body.clone())).await;
        assert_eq!(
            (r.status, r.code().as_str()),
            (StatusCode::UNSUPPORTED_MEDIA_TYPE, "unsupported_media_type"),
            "{ct:?}"
        );
    }
    let h = [
        ("authorization", "Bearer s3cret-token"),
        ("content-type", "application/json; charset=utf-8"),
    ];
    let r = call(&app, Method::POST, "/orders", &h, Some(body)).await;
    assert_eq!(r.status, StatusCode::CREATED);
}

/// A valid order padded with trailing spaces to exactly `len` bytes.
fn padded(len: usize) -> Vec<u8> {
    let mut v = serde_json::to_vec(&order("a", 1)).unwrap();
    assert!(v.len() <= len);
    v.resize(len, b' ');
    v
}

#[tokio::test]
async fn body_limit_is_inclusive() {
    let app = router();
    let at = post_raw(&app, &[], padded(MAX_BODY_BYTES)).await;
    assert_eq!(
        at.status,
        StatusCode::CREATED,
        "exactly the limit is accepted"
    );
    let over = post_raw(&app, &[], padded(MAX_BODY_BYTES + 1)).await;
    assert_eq!(
        (over.status, over.code().as_str()),
        (StatusCode::PAYLOAD_TOO_LARGE, "body_too_large")
    );
}

#[tokio::test]
async fn idempotency_key_makes_a_retry_return_the_first_result() {
    let app = router();
    let k = [("idempotency-key", "abc-123")];
    let first = post_raw(&app, &k, serde_json::to_vec(&order("ada", 5)).unwrap()).await;
    assert_eq!(first.status, StatusCode::CREATED);
    let again = post_raw(&app, &k, serde_json::to_vec(&order("ada", 5)).unwrap()).await;
    assert_eq!(again.status, StatusCode::OK, "a replay is 200, not 201");
    assert_eq!(first.json::<Order>(), again.json::<Order>());
    let clash = post_raw(&app, &k, serde_json::to_vec(&order("ada", 6)).unwrap()).await;
    assert_eq!(
        (clash.status, clash.code().as_str()),
        (StatusCode::CONFLICT, "idempotency_key_reused")
    );
    let other = post_raw(
        &app,
        &[("idempotency-key", "different")],
        serde_json::to_vec(&order("ada", 5)).unwrap(),
    )
    .await;
    assert_eq!(other.status, StatusCode::CREATED);
    assert_eq!(get(&app, "/orders").await.json::<Page>().items.len(), 2);
    // without a key every call creates a new order
    post(&app, order("ada", 5)).await;
    post(&app, order("ada", 5)).await;
    assert_eq!(get(&app, "/orders").await.json::<Page>().items.len(), 4);
}

#[tokio::test]
async fn bad_idempotency_keys_are_rejected_before_anything_is_stored() {
    let app = router();
    let long = "k".repeat(65);
    let ok64 = "k".repeat(64);
    for bad in ["", long.as_str()] {
        let r = post_raw(
            &app,
            &[("idempotency-key", bad)],
            serde_json::to_vec(&order("a", 1)).unwrap(),
        )
        .await;
        assert_eq!(r.status, StatusCode::BAD_REQUEST, "{bad:?}");
    }
    let r = post_raw(
        &app,
        &[("idempotency-key", ok64.as_str())],
        serde_json::to_vec(&order("a", 1)).unwrap(),
    )
    .await;
    assert_eq!(r.status, StatusCode::CREATED);
    assert_eq!(get(&app, "/orders").await.json::<Page>().items.len(), 1);
}

#[tokio::test]
async fn an_invalid_body_does_not_consume_the_idempotency_key() {
    let app = router();
    let k = [("idempotency-key", "reusable")];
    let r = post_raw(&app, &k, serde_json::to_vec(&order("", 1)).unwrap()).await;
    assert_eq!(r.status, StatusCode::BAD_REQUEST);
    let r = post_raw(&app, &k, serde_json::to_vec(&order("fixed", 1)).unwrap()).await;
    assert_eq!(
        r.status,
        StatusCode::CREATED,
        "the corrected request is a first request"
    );
}

#[test]
fn tokens_equal_is_exact() {
    assert!(tokens_equal(b"", b""));
    assert!(tokens_equal(b"abc", b"abc"));
    assert!(!tokens_equal(b"abc", b"abd"));
    assert!(
        !tokens_equal(b"xbc", b"abc"),
        "a difference in the first byte counts"
    );
    assert!(
        !tokens_equal(b"axc", b"abc"),
        "a difference in the middle counts"
    );
    assert!(!tokens_equal(b"abc", b"abcd"), "a prefix is not equal");
    assert!(!tokens_equal(b"abcd", b"abc"));
    assert!(!tokens_equal(b"", b"a"));
    assert!(
        !tokens_equal(b"a\0", b"a"),
        "a trailing zero byte is a difference"
    );
}

#[test]
fn config_debug_never_prints_the_token() {
    let c = Config {
        token: TOKEN.into(),
        request_timeout: Duration::from_secs(1),
    };
    let text = format!("{c:?}");
    assert!(
        !text.contains(TOKEN) && text.contains("<redacted>"),
        "{text}"
    );
}

#[test]
fn error_table_is_stable() {
    let table = [
        (ApiError::Unauthorized, 401, "unauthorized"),
        (ApiError::NotFound, 404, "not_found"),
        (ApiError::KeyConflict, 409, "idempotency_key_reused"),
        (ApiError::TooLarge, 413, "body_too_large"),
        (
            ApiError::UnsupportedMediaType,
            415,
            "unsupported_media_type",
        ),
        (ApiError::MethodNotAllowed, 405, "method_not_allowed"),
        (ApiError::Timeout, 408, "request_timeout"),
    ];
    for (e, status, code) in table {
        assert_eq!((e.status().as_u16(), e.code()), (status, code), "{e:?}");
    }
}

#[test]
fn backoff_doubles_and_is_capped() {
    let base = Duration::from_millis(100);
    let ms: Vec<u128> = (0..7).map(|a| backoff_delay(base, a).as_millis()).collect();
    assert_eq!(ms, vec![100, 200, 400, 800, 1600, 2000, 2000]);
    assert_eq!(
        backoff_delay(base, 40),
        Duration::from_secs(2),
        "a huge attempt number does not overflow"
    );
    assert_eq!(backoff_delay(Duration::ZERO, 3), Duration::ZERO);
}
````

`tests/live.rs`:

````rust
//! Tests with real sockets on 127.0.0.1: client, retries, timeouts, shutdown.

use std::net::SocketAddr;
use std::sync::Arc;
use std::sync::atomic::{AtomicU32, Ordering};
use std::time::{Duration, Instant};

use apilab::client::{ApiClient, ClientConfig, ClientError};
use apilab::{Config, NewOrder, app, serve};
use axum::Router;
use axum::http::StatusCode;
use axum::http::header::CONTENT_TYPE;
use axum::response::IntoResponse;
use axum::routing::{get, post};
use tokio::net::TcpListener;
use tokio::sync::oneshot;

const TOKEN: &str = "live-token";

async fn bind() -> (TcpListener, SocketAddr) {
    let l = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let a = l.local_addr().unwrap();
    (l, a)
}

fn config(timeout_ms: u64) -> Config {
    Config {
        token: TOKEN.into(),
        request_timeout: Duration::from_millis(timeout_ms),
    }
}

/// Starts the real app; the returned sender triggers graceful shutdown.
async fn start(
    cfg: Config,
) -> (
    SocketAddr,
    oneshot::Sender<()>,
    tokio::task::JoinHandle<std::io::Result<()>>,
) {
    let (l, addr) = bind().await;
    let (tx, rx) = oneshot::channel();
    let h = tokio::spawn(serve(l, app(cfg), async move {
        let _ = rx.await;
    }));
    (addr, tx, h)
}

fn client(addr: SocketAddr, retries: u32, timeout_ms: u64) -> ApiClient {
    ApiClient::new(ClientConfig {
        base_url: format!("http://{addr}"),
        token: Some(TOKEN.into()),
        timeout: Duration::from_millis(timeout_ms),
        retries,
        backoff: Duration::from_millis(5),
    })
    .unwrap()
}

fn new_order(c: &str, cents: u64) -> NewOrder {
    NewOrder {
        customer: c.into(),
        amount_cents: cents,
    }
}

#[tokio::test]
async fn client_round_trip_against_the_real_server() {
    let (addr, _tx, _h) = start(config(2000)).await;
    let c = client(addr, 0, 1000);
    let created = c.create_order(&new_order("ada", 700), None).await.unwrap();
    assert_eq!(created.id, 1);
    assert_eq!(c.get_order(1).await.unwrap(), created);
    for i in 0..4 {
        c.create_order(&new_order(&format!("c{i}"), i), None)
            .await
            .unwrap();
    }
    let p1 = c.list_orders(None, Some(3)).await.unwrap();
    assert_eq!(p1.items.len(), 3);
    let p2 = c.list_orders(p1.next_after, Some(3)).await.unwrap();
    assert_eq!(p2.items.len(), 2);
    assert!(p2.next_after.is_none());
}

#[tokio::test]
async fn api_errors_surface_with_status_and_code() {
    let (addr, _tx, _h) = start(config(2000)).await;
    let c = client(addr, 0, 1000);
    match c.get_order(42).await {
        Err(ClientError::Api { status: 404, error }) => assert_eq!(error.code, "not_found"),
        other => panic!("{other:?}"),
    }
    match c.create_order(&new_order("", 1), None).await {
        Err(ClientError::Api { status: 400, error }) => assert_eq!(error.code, "invalid_request"),
        other => panic!("{other:?}"),
    }
    let anon = ApiClient::new(ClientConfig {
        token: None,
        ..cfg_of(addr)
    })
    .unwrap();
    match anon.create_order(&new_order("a", 1), None).await {
        Err(ClientError::Api { status: 401, .. }) => {}
        other => panic!("{other:?}"),
    }
}

fn cfg_of(addr: SocketAddr) -> ClientConfig {
    ClientConfig {
        base_url: format!("http://{addr}"),
        token: Some(TOKEN.into()),
        timeout: Duration::from_millis(1000),
        retries: 0,
        backoff: Duration::from_millis(5),
    }
}

#[test]
fn client_config_debug_redacts_the_token() {
    let cfg = cfg_of("127.0.0.1:1".parse().unwrap());
    let text = format!("{cfg:?}");
    assert!(
        !text.contains(TOKEN) && text.contains("<redacted>"),
        "{text}"
    );
}

/// A server whose `GET /orders/1` and `POST /orders` answer 503 the first
/// `fail_first` times and a valid order afterwards.
async fn flaky(fail_first: u32) -> (SocketAddr, Arc<AtomicU32>) {
    flaky_with(fail_first, StatusCode::SERVICE_UNAVAILABLE).await
}

async fn flaky_with(fail_first: u32, failure: StatusCode) -> (SocketAddr, Arc<AtomicU32>) {
    let calls = Arc::new(AtomicU32::new(0));
    async fn answer(
        calls: Arc<AtomicU32>,
        fail_first: u32,
        failure: StatusCode,
    ) -> impl IntoResponse {
        let n = calls.fetch_add(1, Ordering::SeqCst) + 1;
        let (status, body) = if n <= fail_first {
            (failure, "upstream busy")
        } else {
            (
                StatusCode::OK,
                r#"{"id":1,"customer":"ok","amountCents":1}"#,
            )
        };
        (status, [(CONTENT_TYPE, "application/json")], body)
    }
    let (c1, c2) = (calls.clone(), calls.clone());
    let router = Router::new()
        .route(
            "/orders/1",
            get(move || answer(c1.clone(), fail_first, failure)),
        )
        .route(
            "/orders",
            post(move || answer(c2.clone(), fail_first, failure)),
        );
    let (l, addr) = bind().await;
    tokio::spawn(async move { axum::serve(l, router).await });
    (addr, calls)
}

#[tokio::test]
async fn get_is_retried_on_503_until_it_succeeds() {
    let (addr, calls) = flaky(2).await;
    let o = client(addr, 3, 1000).get_order(1).await.unwrap();
    assert_eq!(o.customer, "ok");
    assert_eq!(
        calls.load(Ordering::SeqCst),
        3,
        "two failures and one success"
    );
}

#[tokio::test]
async fn retries_are_bounded_and_the_last_response_is_reported() {
    let (addr, calls) = flaky(100).await;
    match client(addr, 2, 1000).get_order(1).await {
        Err(ClientError::Unexpected { status: 503, body }) => assert_eq!(body, "upstream busy"),
        other => panic!("{other:?}"),
    }
    assert_eq!(
        calls.load(Ordering::SeqCst),
        3,
        "1 attempt + 2 retries, no more"
    );
}

#[tokio::test]
async fn post_is_retried_only_with_an_idempotency_key() {
    let (addr, calls) = flaky(1).await;
    let c = client(addr, 3, 1000);
    let r = c.create_order(&new_order("a", 1), None).await;
    assert!(
        matches!(r, Err(ClientError::Unexpected { status: 503, .. })),
        "{r:?}"
    );
    assert_eq!(
        calls.load(Ordering::SeqCst),
        1,
        "no key: exactly one attempt"
    );

    let (addr, calls) = flaky(1).await;
    let c = client(addr, 3, 1000);
    c.create_order(&new_order("a", 1), Some("k1"))
        .await
        .unwrap();
    assert_eq!(
        calls.load(Ordering::SeqCst),
        2,
        "with a key the retry happens"
    );
}

#[tokio::test]
async fn a_client_timeout_is_a_transport_error_and_bounded_in_time() {
    let router = Router::new().route(
        "/orders/1",
        get(|| async {
            tokio::time::sleep(Duration::from_secs(3)).await;
            "too late"
        }),
    );
    let (l, addr) = bind().await;
    tokio::spawn(async move { axum::serve(l, router).await });
    let t = Instant::now();
    match client(addr, 0, 150).get_order(1).await {
        Err(ClientError::Transport(e)) => assert!(e.is_timeout(), "{e}"),
        other => panic!("{other:?}"),
    }
    assert!(
        t.elapsed() < Duration::from_millis(1500),
        "gave up long before the server finished"
    );
}

#[tokio::test]
async fn the_server_answers_408_when_the_handler_is_too_slow() {
    let (addr, _tx, _h) = start(config(150)).await;
    let r = reqwest::get(format!("http://{addr}/sleep/1500"))
        .await
        .unwrap();
    assert_eq!(r.status(), StatusCode::REQUEST_TIMEOUT);
    let body: apilab::ErrorBody = r.json().await.unwrap();
    assert_eq!(body.error.code, "request_timeout");
    let ok = reqwest::get(format!("http://{addr}/sleep/20"))
        .await
        .unwrap();
    assert_eq!(ok.status(), StatusCode::OK);
}

#[tokio::test]
async fn connection_refused_is_a_connect_error() {
    let (l, addr) = bind().await;
    drop(l);
    match client(addr, 0, 500).get_order(1).await {
        Err(ClientError::Transport(e)) => assert!(e.is_connect(), "{e}"),
        other => panic!("{other:?}"),
    }
}

#[tokio::test]
async fn retries_cover_a_server_that_starts_late() {
    let (l, addr) = bind().await;
    drop(l);
    let late = tokio::spawn(async move {
        tokio::time::sleep(Duration::from_millis(120)).await;
        let l = TcpListener::bind(addr).await.unwrap();
        let _ = serve(l, app(config(2000)), std::future::pending()).await;
    });
    let c = ApiClient::new(ClientConfig {
        retries: 8,
        backoff: Duration::from_millis(40),
        ..cfg_of(addr)
    })
    .unwrap();
    let page = c
        .list_orders(None, None)
        .await
        .expect("a later attempt connects");
    assert!(page.items.is_empty());
    late.abort();
}

#[tokio::test]
async fn graceful_shutdown_finishes_in_flight_requests_and_refuses_new_ones() {
    let (addr, tx, handle) = start(config(5000)).await;
    let slow = tokio::spawn(async move {
        reqwest::get(format!("http://{addr}/sleep/500"))
            .await
            .unwrap()
            .status()
    });
    tokio::time::sleep(Duration::from_millis(100)).await;
    tx.send(()).unwrap();
    assert_eq!(
        slow.await.unwrap(),
        StatusCode::OK,
        "the in-flight request completed"
    );
    tokio::time::timeout(Duration::from_secs(3), handle)
        .await
        .expect("server stopped")
        .unwrap()
        .unwrap();
    let after = reqwest::Client::builder()
        .timeout(Duration::from_millis(300))
        .build()
        .unwrap()
        .get(format!("http://{addr}/health"))
        .send()
        .await;
    assert!(after.is_err(), "no new requests are served after shutdown");
}

#[tokio::test]
async fn concurrent_requests_with_one_key_create_one_order() {
    let (addr, _tx, _h) = start(config(5000)).await;
    let c = client(addr, 0, 3000);
    let mut tasks = Vec::new();
    for _ in 0..32 {
        let c = c.clone();
        tasks.push(tokio::spawn(async move {
            c.create_order(&new_order("ada", 9), Some("same-key"))
                .await
                .unwrap()
        }));
    }
    let mut ids = Vec::new();
    for t in tasks {
        ids.push(t.await.unwrap().id);
    }
    assert!(ids.iter().all(|&i| i == ids[0]), "{ids:?}");
    let page = c.list_orders(None, Some(100)).await.unwrap();
    assert_eq!(page.items.len(), 1);
}

#[tokio::test]
async fn concurrent_creates_get_unique_consecutive_ids() {
    let (addr, _tx, _h) = start(config(5000)).await;
    let c = client(addr, 0, 3000);
    let mut tasks = Vec::new();
    for i in 0..50u64 {
        let c = c.clone();
        tasks.push(tokio::spawn(async move {
            c.create_order(&new_order("x", i), None).await.unwrap().id
        }));
    }
    let mut ids = Vec::new();
    for t in tasks {
        ids.push(t.await.unwrap());
    }
    ids.sort_unstable();
    assert_eq!(ids, (1..=50).collect::<Vec<_>>());
}

#[tokio::test]
async fn only_transient_statuses_are_retried() {
    for (status, retried) in [
        (429u16, true),
        (502, true),
        (503, true),
        (504, true),
        (500, false),
        (400, false),
        (404, false),
        (501, false),
    ] {
        let (addr, calls) = flaky_with(1, StatusCode::from_u16(status).unwrap()).await;
        let r = client(addr, 2, 1000).get_order(1).await;
        let expected = if retried { 2 } else { 1 };
        assert_eq!(calls.load(Ordering::SeqCst), expected, "status {status}");
        assert_eq!(r.is_ok(), retried, "status {status}: {r:?}");
    }
}

#[tokio::test]
async fn the_sleep_diagnostic_route_is_capped() {
    let (addr, _tx, _h) = start(config(100)).await;
    let at_cap = reqwest::get(format!("http://{addr}/sleep/5000"))
        .await
        .unwrap();
    assert_eq!(
        at_cap.status(),
        StatusCode::REQUEST_TIMEOUT,
        "accepted, then cut off by the 100 ms server timeout"
    );
    let over = reqwest::get(format!("http://{addr}/sleep/5001"))
        .await
        .unwrap();
    assert_eq!(over.status(), StatusCode::BAD_REQUEST);
}
````

`examples/bench.rs`:

```rust
//! Rough request cost on loopback. Client and server share this machine's CPUs.
use std::time::{Duration, Instant};

use apilab::{Config, app, serve};
use axum::body::Body;
use axum::http::Request;
use tower::ServiceExt;

#[tokio::main]
async fn main() {
    let cfg = Config {
        token: "t".into(),
        request_timeout: Duration::from_secs(5),
    };

    // 1. in-process: no sockets, no HTTP parsing
    let router = app(cfg.clone());
    let n = 100_000;
    let t = Instant::now();
    for _ in 0..n {
        let req = Request::get("/health").body(Body::empty()).unwrap();
        let resp = router.clone().oneshot(req).await.unwrap();
        assert!(resp.status().is_success());
    }
    let d = t.elapsed();
    println!(
        "in-process GET /health:     {:>7.2} us/request ({} requests)",
        d.as_secs_f64() * 1e6 / n as f64,
        n
    );

    // 2. real sockets
    let l = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = l.local_addr().unwrap();
    tokio::spawn(serve(l, app(cfg), std::future::pending()));
    let client = reqwest::Client::new();
    let url = format!("http://{addr}/health");
    for _ in 0..200 {
        client
            .get(&url)
            .send()
            .await
            .unwrap()
            .bytes()
            .await
            .unwrap();
    }
    let n = 20_000;
    let t = Instant::now();
    for _ in 0..n {
        client
            .get(&url)
            .send()
            .await
            .unwrap()
            .bytes()
            .await
            .unwrap();
    }
    let d = t.elapsed();
    println!(
        "loopback, 1 client, keep-alive: {:>7.1} us/request ({} requests, {:.0} req/s)",
        d.as_secs_f64() * 1e6 / n as f64,
        n,
        n as f64 / d.as_secs_f64()
    );

    // new TCP connection for every request (no connection reuse)
    let fresh = reqwest::Client::builder()
        .pool_max_idle_per_host(0)
        .build()
        .unwrap();
    let n = 5_000;
    let t = Instant::now();
    for _ in 0..n {
        fresh.get(&url).send().await.unwrap().bytes().await.unwrap();
    }
    let d = t.elapsed();
    println!(
        "loopback, 1 client, new connection each time: {:>7.1} us/request ({} requests)",
        d.as_secs_f64() * 1e6 / n as f64,
        n
    );

    for conc in [8usize, 64] {
        let per = 10_000 / conc * 4;
        let t = Instant::now();
        let mut tasks = Vec::new();
        for _ in 0..conc {
            let c = client.clone();
            let u = url.clone();
            tasks.push(tokio::spawn(async move {
                for _ in 0..per {
                    c.get(&u).send().await.unwrap().bytes().await.unwrap();
                }
            }));
        }
        for t in tasks {
            t.await.unwrap();
        }
        let d = t.elapsed();
        let total = per * conc;
        println!(
            "loopback, {conc:>2} concurrent clients:  {:>7.0} req/s ({total} requests)",
            total as f64 / d.as_secs_f64()
        );
    }

    // 3. POST /orders (JSON in, JSON out, mutex-protected insert)
    let body = serde_json::json!({"customer":"ada","amountCents":1250});
    let n = 10_000;
    let t = Instant::now();
    for _ in 0..n {
        let r = client
            .post(format!("http://{addr}/orders"))
            .bearer_auth("t")
            .json(&body)
            .send()
            .await
            .unwrap();
        assert_eq!(r.status(), 201);
        r.bytes().await.unwrap();
    }
    let d = t.elapsed();
    println!(
        "loopback POST /orders, 1 client: {:>7.1} us/request ({} requests)",
        d.as_secs_f64() * 1e6 / n as f64,
        n
    );
}
```

✓ Checks I ran: `cargo fmt`, `cargo clippy --all-targets -- -D warnings` (clean), `cargo test --locked`: `tests/api.rs` 16 tests, `tests/live.rs` 15 tests, 30 consecutive full runs without a failure.

✓ **Mutation checks.** I applied 31 hand-made mutations one at a time to a copy of the crate and ran the whole suite (`cargo test --locked`). Result: **28 caught, 2 survived, 1 equivalent.** Before running them I looked at the list and added four tests I expected to be needed (`xbc`/`axc` token cases for M6, the `/sleep/5000` boundary for M20, the status table for M23, the "no `nextAfter` key" check for M30); I did not run the earlier suite against those four, so I cannot say how many of them it would have missed.

| # | Mutation | Result |
|---|---|---|
| M1 | body limit one byte larger | caught: `body_limit_is_inclusive` |
| M2 | customer length `>` → `>=` | caught |
| M3 | drop the "empty customer" check | caught |
| M4 | amount `>` → `>=` | caught |
| M5 | `tokens_equal` ignores length | caught |
| M6 | `tokens_equal` keeps only the last byte difference | caught (needed the new first-byte and middle-byte cases) |
| M7 | any bearer token accepted | caught |
| M8 | no `WWW-Authenticate` header | caught |
| M9 | idempotency body comparison always "same" | caught |
| M10 | replay answers `201` instead of `200` | caught |
| M11 | key length `<=` → `<` | caught |
| M12 | `limit` range excludes 100 | caught |
| M13 | fetch `limit` rows instead of `limit + 1` | caught (also fails the client round trip) |
| M14 | `after` inclusive (`range(start..)`) | caught |
| M15 | `next_after` when the page is exactly full | caught |
| M16 | keys never stored | caught |
| M17 | `lock().unwrap()` instead of recovering from poisoning | **survived**: nothing poisons the lock, so no test exercises the branch |
| M18 | 408 not rewritten into the envelope | caught |
| M19 | do not copy `Allow` into the rewritten 405 | **equivalent**: the router adds `Allow` after the layer runs (✓ the header was there with the copy removed); I deleted the dead copy from the code |
| M20 | sleep cap `>` → `>=` | caught (needed the `/sleep/5000` case) |
| M21 | retry every request, keyed or not | caught |
| M22 | backoff index `attempt - 1` → `attempt` | **survived**: only the length of a sleep changes (5 ms vs 10 ms in tests); a test that asserted delays would be timing-flaky |
| M23 | 429 not retried | caught (needed the status table) |
| M24 | backoff cap 2 s → 3 s | caught |
| M25 | every status treated as success when decoding | caught |
| M26 | wrong `Location` path | caught |
| M27 | body-limit layer removed | caught |
| M28 | shutdown signal ignored | caught |
| M29 | missing-content-type arm removed | caught |
| M30 | `nextAfter: null` serialized on the last page | caught (needed the raw-JSON check) |
| M31 | 405 not rewritten into the envelope | caught |

The script I used first labelled every mutant "compile error": its pattern matched cargo's own line `error: test failed`. I caught that by reading one log, and re-classified all logs by hand: a mutant counts as caught only if a named test printed `FAILED`.

**Design notes.**

- `AppState` holds an `Arc<Mutex<Store>>`; every critical section is short and contains no `.await` (the compiler checks the second half).
- Each rule that can fail has one `ApiError` variant with one code; clients switch on `code`, not on `message`.
- The write routes and the read routes are separate routers so the auth layer wraps only the writes; the body limit and timeout wrap everything.
- `serve` takes the shutdown future as a parameter so tests (and `main`) decide what triggers it.
- Not handled: TLS, HTTP/2, a header-read timeout, a connection limit, rate limiting, request ids, structured logging and metrics (Lesson 43's tools apply), persistent storage and key expiry (Lesson 46), real authentication, CORS, compression, streaming bodies, WebSockets, and OpenAPI documents.

</details>

---

## 16. Key takeaways

- HTTP is a boundary: limit what clients control (✓ body, page size, string lengths; ✓ headers are generous by default; ✓ a silent client is never timed out), authenticate before parsing, and make failures machine-readable.
- Use one error type and one JSON envelope, and wrap the extractors and framework errors so *every* failure uses it (✓ three escaped in my first version).
- Axum's compile errors improve a lot with `#[axum::debug_handler]` (✓): body extractor last, `IntoResponse` result, no `std` guard across an await.
- Cursor pagination with `limit + 1` lookahead, idempotency keys with an atomic check-and-insert, and no state change on invalid input make a retrying client safe (✓ concurrency test).
- Clients need a shared `Client`, timeouts, and bounded, backed-off retries of idempotent requests only (✓ 75 to 79 µs against 140 to 156 µs with and without connection reuse; ✓ eight statuses tested).
- Test in memory for speed (✓ 1.5 to 1.8 µs per request) and over sockets for time-dependent behavior; give timing tests wide margins (✓ 30 clean runs).
- A handler panic is one dropped connection, not a dead server, but the client gets no reply (§5).

**Connections.** [Lesson 44](44_cli_serialization.md) supplies the serde and validation habits used for every request body; [Lesson 43](43_observability_ci_supply_chain.md) covers logging, secrets and dependency policy, which matter more with a dependency tree this size; [Lessons 30–32](30_futures_async.md) explain the runtime under every handler, cancellation (a dropped request drops its future) and backpressure; [Lesson 29](29_locks_deadlocks.md) and [Lesson 24](24_interior_mutability.md) cover the mutex and its poisoning; [Lesson 26](26_error_design.md) is the model for `ApiError` and `ClientError`. Next: Lesson 46 covers databases, configuration, authentication and background jobs.

---

## 17. References

- axum documentation (routing, extractors, `IntoResponse`, `middleware::from_fn`, `serve`, `with_graceful_shutdown`, `DefaultBodyLimit`, `debug_handler`): <https://docs.rs/axum>
- `tower` and `tower-http` (`Service`, `Layer`, `ServiceExt::oneshot`, `TimeoutLayer`, `RequestBodyLimitLayer`, `CatchPanicLayer`): <https://docs.rs/tower>, <https://docs.rs/tower-http>
- `hyper` guides and API (HTTP/1 server builder options such as header read timeout): <https://hyper.rs/>
- `reqwest` documentation (`Client`, `ClientBuilder::timeout`, features, TLS backends): <https://docs.rs/reqwest>
- RFC 9110, "HTTP Semantics" (methods, status codes, idempotency, safe methods): <https://www.rfc-editor.org/rfc/rfc9110>
- RFC 9112, "HTTP/1.1" (message syntax, connections, chunked transfer): <https://www.rfc-editor.org/rfc/rfc9112>
- RFC 6585 (status 429, 431), RFC 9457 "Problem Details for HTTP APIs" (a standard error format; `apilab` uses its own simpler envelope): <https://www.rfc-editor.org/rfc/rfc6585>, <https://www.rfc-editor.org/rfc/rfc9457>
- The IETF "Idempotency-Key HTTP Header Field" draft (the idea implemented in §6.4; I did not read the current draft): search for the title on <https://datatracker.ietf.org/>
- Tokio tutorial and the `tokio::sync::Mutex` documentation: <https://tokio.rs/tokio/tutorial>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: RFC contents and the safe/idempotent classification; the exact 2 MB value of axum's default body limit; `reqwest` having no default overall timeout; hyper's `header_read_timeout` option and the fact that `axum::serve` does not expose it; `ConcurrencyLimitLayer`; `spawn_blocking` behavior; `tokio::sync::Mutex` costs and lack of poisoning; the argument against offset pagination; jitter and `Retry-After` handling; SSRF, CORS, CSRF, TLS and request-smuggling remarks; and everything about HTTP/2, WebSockets, proxies and the public Internet. The measurements marked ✓ are from one shared virtual machine with the load generator and the server on the same two cores, over loopback, with a release build for the benchmark (`examples/bench.rs`) and a dev build for the tests; run each of them on your own hardware before drawing conclusions. The compile errors quoted are from `rustc 1.97.0` with `axum 0.8.9`; their wording may change. Exercise answers 1, 2, 6, 7 and 8 are reasoning (the idempotency scenarios in 7 are covered by the tests cited); the code blocks in answers 3 and 4 and the challenge fix in §13 were compiled and run.
