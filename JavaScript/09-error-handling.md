# Chapter 9 — Error Handling

> Depends on: Ch.6 (async error propagation relies on how Promises/async-await handle rejections).
> Feeds into: Ch.11 (graceful degradation is a performance/reliability concern), real-world production reliability.

---

## 9.1 `try`/`catch`/`finally`

**Intuitive version:** `try` runs code that might fail; `catch` handles the failure if it happens; `finally` runs regardless of whether an error occurred — for cleanup that must always happen.

**Formal version:**
```js
try {
  riskyOperation();
} catch (err) {
  console.error("Handled:", err.message);
} finally {
  cleanup(); // ALWAYS runs — even if catch itself throws, even if try has a `return`
}
```

**The `finally` + `return` interaction (a genuine gotcha, common interview question):**
```js
function test() {
  try {
    return "try";
  } finally {
    console.log("finally runs"); // runs BEFORE the function actually returns
  }
}
test(); // logs "finally runs", THEN returns "try"

function test2() {
  try {
    return "try";
  } finally {
    return "finally"; // OVERRIDES the try's return value entirely!
  }
}
test2(); // returns "finally" — a return inside finally always wins, considered a serious code smell
```

**Why `finally` exists:** guarantees cleanup code (closing a file handle, releasing a lock, hiding a loading spinner) runs no matter how the `try` block exits — normal completion, an exception, or even a `return`/`break`/`continue`.

**Interview questions:**
- *Beginner:* Does `finally` run if the `try` block has a `return` statement?
- *Senior/FAANG:* What does `test2()` above return, and why is a `return` inside `finally` considered dangerous? (It silently swallows any exception from `try`/`catch` too, not just overriding return values.)

---

## 9.2 `throw` and the Error Object Hierarchy

**Intuitive version:** `throw` immediately stops normal execution and starts unwinding the call stack, looking for the nearest enclosing `catch` — if none exists anywhere up the stack, the program crashes (or in Node, the process exits; in a browser, an uncaught error is logged).

**Formal version — you can `throw` *any* value, not just an `Error`:**
```js
throw "a string";          // valid but bad practice — no stack trace
throw { code: 500 };        // valid but bad practice
throw new Error("proper");   // best practice — includes .message, .stack, .name
```

**Built-in Error subtypes (know these for interviews):**
```js
new TypeError("wrong type");        // e.g., calling a non-function, accessing property of null
new RangeError("out of range");      // e.g., invalid array length, number out of allowed range
new ReferenceError("not defined");    // e.g., accessing an undeclared variable
new SyntaxError("bad syntax");         // usually thrown by the parser itself, before execution
```

**Why always throw `Error` objects, never plain values:** `Error` instances automatically capture a `.stack` trace at the point of creation — critical for debugging — and integrate correctly with debugging tools, source maps, and error-tracking services (Sentry, etc.). A thrown string has none of this.

**Interview questions:**
- *Beginner:* Can you `throw` something that isn't an `Error`? Should you?
- *Mid:* Name the four most common built-in `Error` subtypes and a scenario for each.

---

## 9.3 Custom Error Types

**Intuitive version:** Extending the built-in `Error` class lets you create domain-specific error types (`ValidationError`, `NotFoundError`) that carry extra structured data and can be distinguished with `instanceof` — much more robust than checking `error.message` strings.

**Formal version:**
```js
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError"; // otherwise shows generic "Error" in stack traces
    this.field = field;
    // Note: Error.captureStackTrace(this, ValidationError) is a V8-specific API
    // to exclude the constructor itself from the stack trace — nice-to-have, not required
  }
}

function validateAge(age) {
  if (age < 0) throw new ValidationError("Age cannot be negative", "age");
}

try {
  validateAge(-5);
} catch (err) {
  if (err instanceof ValidationError) {
    console.log(`Validation failed on field: ${err.field}`);
  } else {
    throw err; // re-throw anything you don't specifically know how to handle
  }
}
```

**Best practice — never swallow errors you don't understand:** the `else { throw err; }` above is important — catching broadly but only *handling* what you specifically expect, and re-throwing everything else, prevents silently hiding real bugs.

**Interview questions:**
- *Mid:* Why extend `Error` instead of just throwing a plain object with a `type` field?
- *Senior:* Why is re-throwing unrecognized errors in a `catch` block considered a best practice rather than silently swallowing everything?

---

## 9.4 Error Handling in Async Code

**Intuitive version:** Errors in async code don't propagate the same way as synchronous errors — a `throw` inside a `.then()` callback or an `async function` doesn't crash your program synchronously; it turns the Promise into a rejected one, which needs its own handling.

**Formal version:**
```js
// Promise chain — .catch() handles rejection from ANY earlier .then() in the chain
fetchUser(id)
  .then(user => fetchPosts(user.id))
  .then(posts => console.log(posts))
  .catch(err => console.error("Something failed:", err));

// async/await — try/catch works because `await` converts a rejection into a thrown exception
async function loadUser(id) {
  try {
    const user = await fetchUser(id);
    const posts = await fetchPosts(user.id);
    return posts;
  } catch (err) {
    console.error("Something failed:", err);
  }
}
```

**Unhandled promise rejections — a real production hazard:**
```js
async function risky() { throw new Error("oops"); }
risky(); // no .catch, no try/catch around the call — this is an UNHANDLED REJECTION
// Node: logs a warning and (in modern versions) can crash the process
// Browser: fires a global "unhandledrejection" event, visible in DevTools console
```

**Best practice — global safety nets (defense in depth, not a substitute for proper handling):**
```js
// Browser
window.addEventListener("unhandledrejection", event => {
  console.error("Unhandled rejection:", event.reason);
});

// Node
process.on("unhandledRejection", (reason) => {
  console.error("Unhandled rejection:", reason);
});
```

**Common mistake — mixing `.then()` and `await` inconsistently, or forgetting `await` entirely:**
```js
async function broken() {
  fetchUser(id).then(user => console.log(user)); // missing await — errors here are NOT caught
  // by any surrounding try/catch, because this promise chain is unrelated/unattached to the
  // async function's own control flow
}
```

**Interview questions:**
- *Mid:* Why does a `throw` inside an `async function` not immediately crash the program the way a synchronous `throw` at the top level would?
- *Senior:* What is an unhandled promise rejection, and what are two ways to guard against it in production?
- *FAANG-style debugging:* a production Node service occasionally crashes with no stack trace pointing to application code, only "UnhandledPromiseRejectionWarning." Walk through how you'd track down the source (enable stricter rejection handling flags, add global listeners with full `reason` logging, audit for missing `await`/`.catch()` on fire-and-forget promises).

---

## Practical exercises — Chapter 9

1. Predict the exact return value of a function whose `try` block returns `"A"` and whose `finally` block returns `"B"`. Explain why this pattern is considered dangerous in code review.
2. **Debugging exercise:** find and fix the bug in `broken()` (§9.4) so errors from `fetchUser` are properly caught.
3. **Small challenge:** implement a `NetworkError` and `ValidationError` custom error hierarchy (both extending a shared `AppError` base class), and a single `catch` block that handles each type differently using `instanceof`.
4. **Advanced challenge:** implement a `safeAsync(fn)` wrapper that catches any error from an async function and returns a consistent `[error, result]` tuple (Go-style error handling) instead of requiring try/catch at every call site.

---

**Next:** Chapter 10 — Advanced / Metaprogramming (Symbols, iterators/iterables, generators as iterators, deeper Proxy/Reflect patterns, the decorators proposal). Say **"next"** when ready.
