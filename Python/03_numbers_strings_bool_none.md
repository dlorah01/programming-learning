---
layout: default
title: "Module 2: Numbers, Strings, Booleans, None, Truthiness"
---

# Python Mastery Course — Module 2: Numbers, Strings, Booleans, None, Truthiness

---

## 2.1 Numbers — `int`, `float`, `complex`, and What JS Doesn't Have

**The core divergence from JS in one sentence:** JS has exactly one numeric type (`number`, an IEEE-754 double, plus `BigInt` bolted on later); Python has always had a real distinction between `int` (arbitrary precision, no overflow, ever) and `float` (IEEE-754 double, same as JS's `number`).

**`int` internals.** Python's `int` is not a fixed-width machine integer. CPython implements it as a variable-length array of "digits" (base 2**30 chunks internally) — it grows automatically as the value grows. This means:

```python
print(2 ** 1000)   # a full, exact, 302-digit integer — no overflow, no precision loss
```

There's no `int` overflow bug class in Python at all — a real, structural difference from languages with fixed-width ints, and different from JS pre-`BigInt` where `Number.MAX_SAFE_INTEGER` (2^53 - 1) silently loses precision beyond that point. **The cost:** arbitrary-precision arithmetic is measurably slower than fixed-width machine arithmetic for small numbers — every `int` operation pays some overhead for this flexibility, part of why tight numeric loops are slow in CPython (ties back to Module 0's "no JIT" discussion — there's no escape hatch to unbox into a machine `int64` the way a JIT could).

**`float` internals.** Standard 64-bit IEEE-754 double, identical bit layout to JS's `number`. This means Python has the exact same classic floating-point gotcha as JS:

```python
print(0.1 + 0.2)          # 0.30000000000000004
print(0.1 + 0.2 == 0.3)   # False
```
Same root cause as JS, same fix philosophy (compare with tolerance, e.g. `math.isclose(a, b)`, or use `decimal.Decimal` for money/precision-critical work — never native `float` for currency).

**Division — the sharpest, most-tested divergence from JS:**

```python
print(7 / 2)     # 3.5   — true division, always returns float (Python 3; this differs from Python 2!)
print(7 // 2)    # 3     — floor division, truncates toward negative infinity
print(-7 // 2)   # -4    — NOT -3! Floors toward -infinity, doesn't truncate toward zero
print(7 % 2)     # 1
print(-7 % 2)    # 1     — NOT -1! Python's % always returns a result with the SAME SIGN AS THE DIVISOR
```

JS's `%` is a *remainder* operator (sign follows the dividend): `-7 % 2` in JS is `-1`. Python's `%` is a true *modulo* (sign follows the divisor): `-7 % 2` in Python is `1`. This is a frequent, genuine bug source when porting numeric code between the two languages — flag it every time you translate modular-arithmetic logic (hashing, circular buffers, clock arithmetic).

**Interview trap:** `//` is not "integer division that truncates" — it's *floor* division. `7 // 2 == 3` looks like truncation and floor agree for positive numbers, but they diverge for negatives, exactly as shown above. Say "floor division" out loud in an interview, not "integer division."

**`complex`:** built-in, first-class (`3 + 4j`), rarely used outside scientific/DSP code — JS has no built-in equivalent at all. Mention it exists; don't over-invest here.

---

## 2.2 Strings — Immutable Unicode Sequences

**Core model:** Python 3 `str` is an immutable sequence of Unicode code points (this was the headline Python 2→3 break — Python 2's `str` was bytes by default, with `unicode` as a separate opt-in type; Python 3 unified this so text is *always* Unicode `str`, and raw bytes are the explicitly separate `bytes` type). JS strings are also effectively immutable-from-the-language's-perspective UTF-16 sequences — conceptually close, but Python's Unicode handling is more consistent (JS's UTF-16 surrogate-pair handling for characters outside the BMP is a well-known JS wart; Python 3's `str` since 3.3 uses a flexible internal representation — PEP 393 — that stores each string in the narrowest fixed-width encoding that fits its actual content, avoiding that class of bug entirely).

**Every "mutation" method returns a new string** (immutability from Module 1, applied concretely):

```python
s = "hello"
s.upper()          # returns "HELLO" — s itself is UNCHANGED
print(s)            # "hello"
s = s.upper()        # must rebind to observe the change
```

This trips up JS developers used to some in-place-feeling array/string methods; in Python, *zero* string methods mutate — there is no such thing as a mutating string method, ever, because `str` is immutable, full stop.

**Building strings efficiently — a real performance consideration, not just style:**

```python
# ANTI-PATTERN — O(n^2) in the worst case
result = ""
for word in big_list_of_words:
    result += word    # each += creates a brand-new string, copying everything so far

# IDIOMATIC — O(n)
result = "".join(big_list_of_words)
```

Because `str` is immutable, `result += word` in a loop reallocates and copies the entire accumulated string on every iteration (CPython has a narrow special-case optimization for exactly this pattern in some circumstances, but you should never *rely* on it) — `"".join(...)` builds the final string in one pass and is the idiomatic, expected-in-code-review way to concatenate many pieces. This is one of the most common "not idiomatic Python" flags in code review for JS transplants used to `arr.join()` being the exception rather than knowing `+=` accumulation is the anti-pattern here specifically.

**f-strings (3.6+) — the idiomatic formatting mechanism, direct analog to JS template literals:**

```python
name, age = "Ada", 30
print(f"{name} is {age} years old")      # like `${name} is ${age} years old` in JS
print(f"{age * 2 = }")                   # 3.8+ debug specifier: prints "age * 2 = 60"
print(f"{3.14159:.2f}")                  # format-spec mini-language: "3.14"
```

f-strings are compiled directly into concatenation + formatting bytecode — no runtime parsing of the format string happens (unlike `str.format()` or `%`-formatting, both of which still exist for legacy/specific reasons but which you should default away from in new code).

**Common pitfall:** slicing and indexing operate on *code points*, not grapheme clusters — `len("👨‍👩‍👧‍👦")` is not `1` (that emoji is several code points joined with zero-width joiners); this matters for genuinely internationalized text processing, rarely for everyday code, but know it exists rather than being blindsided.

---

## 2.3 Booleans — `bool` Is a Subclass of `int`

**This one genuinely surprises people, including experienced devs:**

```python
print(True == 1)     # True
print(True + True)   # 2
print(isinstance(True, int))   # True — bool literally subclasses int
```

**Why:** historically, Python had no dedicated boolean type until 2.3 (2003) — code used `0`/`1` directly, or user-defined pseudo-booleans. When `bool` was added, it was retrofitted as an `int` subclass specifically to preserve backward compatibility with the enormous amount of existing code that used plain integers as truthy/falsy flags. This is a pure historical-compatibility artifact, not a design ideal — but it's permanent now, and it means `True`/`False` can genuinely participate in arithmetic and indexing (`[10, 20][True]` is legal and returns `20`) — clever code sometimes exploits this; code review will generally still want you to write it explicitly instead.

**JS comparison:** JS's `true`/`false` are a wholly separate primitive type with no arithmetic identity (`true + true` is `2` in JS too, actually, via coercion — so this one behaves similarly in both languages, just for different underlying reasons: JS coerces at the operator; Python's `bool` genuinely *is* numeric).

---

## 2.4 `None` — Python's Only Null-ish Value (No `undefined`, No Distinction)

**The biggest simplification vs. JS here:** JS has *two* "absence" values — `undefined` (unset/missing) and `null` (deliberately empty) — with famously inconsistent rules about which appears where (`obj.missingProp` → `undefined`; `JSON.parse` absence → often `null`; `==` treats them as equal, `===` doesn't). Python has exactly **one**: `None`, a true singleton (there is exactly one `None` object for the entire process lifetime — this is why `is None` works reliably as identity, per Module 1).

- A function with no explicit `return` returns `None` (equivalent-ish to a JS function falling off the end implicitly returning `undefined`).
- There's no equivalent of "accessing a property that was never declared" silently giving you a sentinel — accessing a missing dict key raises `KeyError`; accessing a missing attribute raises `AttributeError`. This is the Zen of Python's "errors should never pass silently" principle made concrete (Module 0) — Python would rather throw loudly than hand you a `None`-like value you might not check for.
- `dict.get(key)` is the explicit, opt-in way to get `None` (or a supplied default) instead of an exception for a missing key — mirroring JS's optional-chaining instinct, but you ask for it per-call rather than getting silent-`undefined` by default.

```python
config = {"host": "localhost"}
print(config["port"])          # KeyError — loud, immediate
print(config.get("port"))      # None — you opted in
print(config.get("port", 8080))  # 8080 — you opted in with a default
```

---

## 2.5 Truthiness — Broader Than JS, One Consistent Rule

**The rule:** every object has a truth value. `bool(x)` is `False` for: `None`, `False`, numeric zero of any numeric type (`0`, `0.0`, `0j`), and any **empty** container/sequence (`""`, `[]`, `{}`, `()`, `set()`). Everything else is truthy — including, notably, the string `"False"` (a non-empty string is always truthy regardless of its content — a classic gotcha for developers reflexively expecting the string `"False"` to be falsy).

```python
if []: print("truthy")          # doesn't print — empty list is falsy
if [0]: print("truthy")         # prints — non-empty list, even containing a falsy element, IS truthy
if "False": print("truthy")     # prints — non-empty string, regardless of content
```

**JS comparison — where they align and where they diverge:** JS's falsy set is `false, 0, -0, 0n, "", null, undefined, NaN`. Python's aligns closely for numbers/strings/None, but **diverges sharply on containers**: an empty JS array `[]` and empty object `{}` are both **truthy** in JS (`if ([]) {}` runs the block!) — this is a real, frequent bug source for JS developers writing Python who reflexively assume containers need an explicit `.length` check, and separately for Python developers writing JS who get bitten by `if (arr)` always being true. Internalize: **Python containers participate in truthiness by emptiness; JS containers do not.**

**Under the hood:** `bool(x)` first tries `x.__bool__()`; if that's not defined, it falls back to `len(x) != 0` via `__len__()`; if neither is defined, every instance is truthy by default. This is why you can make your own classes participate in `if my_object:` checks meaningfully by implementing `__bool__` or `__len__` (Module 7 data-model territory) — another instance of "everything is just a protocol method dispatch," a theme that will recur constantly.

**Idiomatic style this produces, that reads oddly to JS eyes at first:**

```python
# Idiomatic Python — relies on truthiness of emptiness
if not my_list:
    print("empty")

# Non-idiomatic (works, but code review will flag it as "not Pythonic")
if len(my_list) == 0:
    print("empty")
```

---

## 2.6 Common Mistakes / Interview Traps — Consolidated

- Assuming `%` behaves like JS's remainder operator for negative operands — it's modulo, sign follows the divisor (§2.1).
- Calling `//` "integer division" instead of "floor division" — matters the moment negatives are involved.
- String-building with `+=` in a loop instead of `"".join(...)` — works, but is the textbook "not idiomatic / O(n²)" code review flag.
- Forgetting `bool` is an `int` subclass, then being confused why `True + True == 2` or why `[10,20][some_bool]` type-checks at all.
- Using `== None` instead of `is None` (carried over from Module 1, resurfaces constantly here).
- Assuming empty containers are truthy because that's JS's behavior — an inverted-logic bug that's genuinely common when porting JS conditionals.
- Comparing floats with `==` instead of `math.isclose` for anything derived from arithmetic.

---

## 2.7 Exercises

**Predict the output:**
```python
print(-7 // 2, -7 % 2, 7 // -2, 7 % -2)
print(bool(""), bool("0"), bool([0]), bool({}))
print(True + True + True == 3)
```

**Core:** Write a function that safely divides two numbers, returning `None` on division by zero instead of letting `ZeroDivisionError` propagate, and using the `is None` idiom correctly in a caller that consumes it.

**Debugging exercise:** A teammate's currency-summing function uses native `float` and their totals are off by fractions of a cent in production. Explain the root cause precisely (don't just say "floating point is imprecise" — say *why*, in IEEE-754 terms) and name the correct fix (`decimal.Decimal`).

**Interview — junior:** "What does `bool([])` return, and why? Contrast with JavaScript's `Boolean([])`."

**Interview — mid:** "Explain the difference between `//` and `/` in Python 3, and what happens with negative operands specifically."

**Interview — senior:** "A function receives a dict that may or may not have a `'count'` key, where a present-but-zero count is meaningfully different from an absent key. Show how `dict.get` alone is insufficient here, and how you'd distinguish the two cases correctly." (Hint: this is a "explicit is better than implicit" design question, and it foreshadows the sentinel-object pattern.)

**Advanced / FAANG-style:** "Why is `bool` a subclass of `int` in Python, historically — and what real, exploitable consequence does that have for someone writing `sum(1 for x in items if predicate(x))` style counting code?" (Connects forward to generator expressions in Module 6.)

---

*Next: Module 3 — Control Flow: `if`/loops, comprehensions, `match`/`case`, and the iteration protocol at a conceptual level (full depth on iterators/generators comes in Module 6).*
