# Python Mastery Course — Module 10: Error Handling & Context Managers

---

## 10.1 Exceptions as the Primary Control-Flow Mechanism for Errors

**Intuition/philosophy:** Python leans harder into "exceptions as normal control flow" than most JS code does in practice — the community mantra is **EAFP** ("Easier to Ask Forgiveness than Permission") over **LBYL** ("Look Before You Leap"):

```python
# LBYL — check first, then act (closer to typical defensive JS style)
if key in d:
    value = d[key]
else:
    value = default

# EAFP — idiomatic Python — try it, handle failure
try:
    value = d[key]
except KeyError:
    value = default
```

**Why EAFP is preferred, concretely, not just "different taste":** LBYL has a genuine race condition in concurrent contexts (the key could be removed between the check and the access — TOCTOU, "time of check to time of use") that EAFP structurally avoids; and for the common case (key usually present), EAFP has *zero* extra overhead on the success path, while LBYL always pays for the membership check even when it's about to succeed. This is why idiomatic Python style (and PEP 8 itself) explicitly favors `try/except` over defensive pre-checks for exactly this class of problem — a genuine, common code-review nudge for JS transplants who default to guard-clause-heavy defensive style.

**JS comparison:** JS also has `try/catch`, but JS idiom leans more LBYL/defensive by convention (`if (obj && obj.prop)`, optional chaining `?.`) partly *because* `undefined`/missing-property access doesn't throw in JS (Module 2) — there's less to "ask forgiveness for." Python's stricter "missing things throw loudly" design (also Module 2) is precisely what makes EAFP both natural and necessary.

---

## 10.2 `try`/`except`/`else`/`finally` — All Four Clauses, Precisely

```python
def read_config(path):
    try:
        f = open(path)
    except FileNotFoundError:
        print("using defaults")
        return {}
    except PermissionError as e:
        print(f"cannot read: {e}")
        raise                        # re-raises the SAME exception, preserving its original traceback
    else:
        # runs ONLY if the try block raised NOTHING — genuinely useful, often skipped/unknown
        data = f.read()
        f.close()
        return parse(data)
    finally:
        # ALWAYS runs — success, handled exception, unhandled exception, even a `return` in try/except
        print("attempted config read")
```

**Why `else` exists and is worth using (a real, under-known feature):** code in `else` runs only on success — the value proposition is keeping the `try` block itself as narrow as possible (containing *only* the specific call that might raise), while success-path logic that shouldn't accidentally have its own exceptions silently caught by the surrounding `except` clauses lives in `else` instead. Putting `data = f.read()` inside the `try` block would mean an unexpected `IOError` from `.read()` itself gets swallowed by the same `except` clauses meant only for the `open()` call — `else` avoids that scope-creep. This distinction ("keep the try block minimal, use else for the success path") is a genuine, checkable code-review-level best practice, not a rarely-used curiosity.

**`finally` always runs, even across `return`:**

```python
def f():
    try:
        return 1
    finally:
        print("cleanup")   # prints BEFORE the function actually returns

print(f())   # prints "cleanup" then 1
```

A real, common gotcha: a `return` *inside* `finally` silently **overrides** any return/exception from the `try`/`except` block — considered a genuine anti-pattern precisely because it can silently swallow exceptions:

```python
def dangerous():
    try:
        raise ValueError("real problem")
    finally:
        return "masked"   # the ValueError is COMPLETELY discarded — no trace of it, ever

print(dangerous())   # "masked" — the exception vanished silently
```

Flagged by every linter worth using; know it exists specifically to recognize and avoid it, and because "what happens if `finally` also returns" is a genuine interview trap question.

---

## 10.3 Custom Exception Hierarchies — Real API Design

```python
class AppError(Exception):
    """Base class for all application-specific errors."""

class ValidationError(AppError):
    def __init__(self, field, message):
        self.field = field
        super().__init__(f"{field}: {message}")

class NotFoundError(AppError):
    pass

try:
    raise ValidationError("email", "invalid format")
except AppError as e:                # catches ValidationError, NotFoundError, and any future subclass
    print(f"app error: {e}")
```

**Why a custom hierarchy, not just `raise Exception("...")` everywhere — a real, common interview/design question:** a well-designed exception hierarchy lets callers catch precisely the granularity they care about — a caller who only cares "did *anything* app-specific go wrong" catches `AppError`; a caller who needs to handle validation differently from not-found catches the specific subclasses. This mirrors good custom `Error` subclassing discipline in JS/TS (`class ValidationError extends Error`) — same underlying design principle, Python's built-in exception hierarchy (`Exception` → many built-ins like `ValueError`, `KeyError`, `TypeError`) is just deeper and more standardized than JS's flatter, less-conventionalized `Error` ecosystem.

**Real anti-pattern, worth calling out explicitly:** `except Exception:` (or worse, a bare `except:`) with no re-raise — a "catch everything and silently move on" pattern that violates Module 0's "errors should never pass silently" principle directly, and is one of the most common real production bug sources (masks genuine bugs as silent no-ops, makes debugging production incidents far harder). A bare `except:` additionally catches things you almost certainly don't want to catch, like `KeyboardInterrupt` and `SystemExit` (both inherit from `BaseException`, not `Exception` — see next).

**`Exception` vs `BaseException` — a real, precise distinction:** `BaseException` is the true root; `SystemExit`, `KeyboardInterrupt`, and `GeneratorExit` inherit directly from `BaseException`, deliberately *outside* the `Exception` branch, specifically so that a broad `except Exception:` (the idiomatic "catch application errors broadly" pattern) doesn't accidentally swallow Ctrl-C or a deliberate `sys.exit()` call. Always catch `Exception`, essentially never catch bare `BaseException` (or a bare `except:`, which is exactly equivalent to `except BaseException:`) unless you have a very specific, deliberate reason.

**`raise ... from ...` — exception chaining, worth knowing exists:**

```python
try:
    int("not a number")
except ValueError as e:
    raise RuntimeError("config parsing failed") from e   # preserves the original as __cause__
```

Produces a traceback showing both exceptions with "the above exception was the direct cause of" — genuinely useful for not losing root-cause information when translating a low-level exception into a higher-level, more meaningful one at an API boundary.

---

## 10.4 Context Managers — `with`, Built From First Principles

**Intuition:** the `with` statement guarantees a cleanup action runs, no matter how the block exits (normal completion, `return`, exception) — same underlying motivation as `finally`, but scoped tightly to a specific resource and expressed declaratively.

```python
with open("file.txt") as f:
    data = f.read()
# file is guaranteed closed here, even if .read() raised
```

**What `with` desugars to, precisely — this is the actual mechanism, per Module 7.1's "everything is dunder dispatch" theme:**

```python
manager = open("file.txt")
f = manager.__enter__()          # __enter__'s return value is what `as f` binds to
try:
    data = f.read()
finally:
    manager.__exit__(*sys.exc_info())   # always called — exc_info is (None, None, None) on success
```

`__exit__(self, exc_type, exc_value, traceback)` receives details of any exception that occurred inside the block; if it **returns `True`**, the exception is considered handled and is **suppressed** (doesn't propagate further) — a genuinely important, sometimes-surprising detail (returning `None`/`False`, the default, lets the exception propagate normally after cleanup still runs).

**Writing your own context manager, class-based:**

```python
class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self
    def __exit__(self, exc_type, exc_value, tb):
        print(f"elapsed: {time.perf_counter() - self.start:.4f}s")
        return False    # don't suppress exceptions — let them propagate after logging the time

with Timer():
    do_expensive_work()
```

**`contextlib.contextmanager` — the idiomatic, much more common shortcut using a generator:**

```python
from contextlib import contextmanager

@contextmanager
def timer():
    start = time.perf_counter()
    try:
        yield              # everything before yield is __enter__; everything after is __exit__
    finally:
        print(f"elapsed: {time.perf_counter() - start:.4f}s")

with timer():
    do_expensive_work()
```

The generator's single `yield` point is where the `with` block's body actually executes — code before `yield` runs on entry, code after (typically in a `finally`, to guarantee it runs even on exception) runs on exit. This is genuinely the idiomatic, most-used way to write simple context managers in real code — the class-based `__enter__`/`__exit__` form is reserved for cases needing more state or multiple reusable entry points. Recognizing this pattern is directly connected to Module 6's generator mechanics — same `yield`-pauses-execution idea, repurposed here as a structural tool rather than a data-producing one.

**Multiple context managers, and why nesting is rarely necessary:**

```python
with open("in.txt") as fin, open("out.txt", "w") as fout:   # both guaranteed closed correctly
    fout.write(fin.read())
```

---

## 10.5 `exception*` / Exception Groups (3.11+) — Brief, Worth Knowing Exists

```python
try:
    raise ExceptionGroup("multiple failures", [ValueError("a"), TypeError("b")])
except* ValueError as eg:
    print(f"handled value errors: {eg.exceptions}")
except* TypeError as eg:
    print(f"handled type errors: {eg.exceptions}")
```

Added specifically to support scenarios where multiple independent operations can fail *simultaneously* (concurrent task groups in `asyncio`, Module 13, being the primary motivating use case) and a single `try/except` couldn't previously represent "several different exceptions happened at once, from parallel work." Niche outside concurrent code, but a legitimate "are you current on the language" signal at senior level.

---

## 10.6 Common Mistakes / Interview Traps — Consolidated

- Bare `except:` (or overly broad `except Exception:` with no re-raise/logging) silently swallowing real bugs.
- Forgetting `except Exception` doesn't catch `KeyboardInterrupt`/`SystemExit` — a deliberate, correct design, not an oversight, but worth being able to explain.
- Putting a `return` inside `finally`, silently discarding an in-flight exception.
- Writing success-path logic inside `try` instead of `else`, accidentally widening what a given `except` clause can catch.
- Reaching for LBYL defensive checks reflexively (JS habit) where idiomatic Python EAFP would be simpler and avoid a TOCTOU race.
- Forgetting `with` guarantees cleanup even on early `return`/exception — writing manual `try/finally` boilerplate for resource cleanup where `with` already exists and is idiomatic.

---

## 10.7 Exercises

**Predict the output:**
```python
def f():
    try:
        raise ValueError("oops")
    except ValueError:
        return "caught"
    finally:
        print("finally ran")

print(f())
```

**Core:** Design a small exception hierarchy for a payment-processing module (`PaymentError` base, `InsufficientFundsError`, `CardDeclinedError` subclasses), and write a caller that handles `InsufficientFundsError` specifically but lets other `PaymentError` subclasses propagate to a generic handler.

**Debugging exercise:** A codebase has `except Exception: pass` scattered through a data pipeline, and a silent data-corruption bug has gone undetected for weeks. Explain, precisely, why this pattern makes bugs like this both possible and hard to diagnose, and rewrite it to fail loudly while still allowing genuinely recoverable cases to continue.

**Interview — junior:** "What's the difference between `except Exception:` and a bare `except:`? Which should you almost always prefer, and why?"

**Interview — mid:** "Explain what the `else` clause of a `try` statement is for, and rewrite a `try` block that currently puts success-path logic inside `try` to use `else` correctly instead."

**Interview — senior:** "Write a `@contextmanager`-based context manager that acquires a database connection, ensures it's released even on exception, and — separately — decide and justify whether it should suppress exceptions raised inside the `with` block or let them propagate."

**Advanced / FAANG-style:** "Design error handling for a batch job that processes 10,000 independent records, where some records may fail validation. Compare: (a) letting the first failure abort the whole batch, (b) catching and logging per-record failures while continuing, and (c) using exception groups to fail the whole batch loudly at the end while still processing everything. Discuss the tradeoffs and which you'd choose for a nightly batch vs. a user-facing synchronous request."

---

*Next: Module 11 — Functional Tools: `map`/`filter`/`reduce`, `itertools`, and `functools` — the toolbox that replaces most manual loop-and-accumulator code with composable, often lazily-evaluated building blocks.*
