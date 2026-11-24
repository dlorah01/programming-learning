---
layout: default
title: "Lesson 06 — Strings, text and bytes, conversions, and pattern matching"
---

# Lesson 06 — Strings, text and bytes, conversions, and pattern matching

**Module 2 · Syntax and fundamentals** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Sizes are for a **64-bit** target.

Builds on [Lesson 04](04_variables_and_types.md) (`char`, `as`, `TryFrom`) and [Lesson 05](05_functions_and_control_flow.md) (slices, bounds checks, `get`).

---

## 1. Text is UTF-8 bytes

Rust has two main string types, and both are **guaranteed valid UTF-8**:

| Type | What it is | Analogy |
|---|---|---|
| `str` | A dynamically sized run of UTF-8 bytes. Seen behind a reference as `&str` (pointer + length, 16 bytes ✓) | `&[u8]` with a validity promise |
| `String` | An owned, growable, heap-allocated buffer of UTF-8 (pointer + capacity + length, 24 bytes ✓) | `Vec<u8>` with a validity promise |

String literals have type `&'static str`: they are stored in the program image and live for the whole run.

**Design rationale.** Many languages choose UTF-16 or "bytes with no promise". Rust's designers chose UTF-8 plus a *type-level validity guarantee*, because UTF-8 is the dominant encoding on the web and in files, is compact for ASCII, and because a guaranteed-valid `str` lets every library assume valid text without re-checking. The cost: operations that sound simple (the "n-th character", the length in characters) are not O(1), and the language refuses to pretend they are.

```rust
fn main() {
    let lit: &str = "héllo";
    let mut s = String::from(lit);
    s.push_str(", wörld");
    s.push('!');
    println!("{s} len={} chars={} cap>={}", s.len(), s.chars().count(), s.capacity() >= s.len());
    let t = s.clone() + " ok";
    println!("{t}");
    let a = String::from("a");
    let b = String::from("b");
    let ab = a + &b;                 // a is moved; b is only borrowed
    println!("{ab} {b}");
    let f = format!("{}-{}", ab, 42);
    println!("{f}");
}
```

✓ output:

```text
héllo, wörld! len=15 chars=13 cap>=true
héllo, wörld! ok
ab b
ab-42
```

- `len()` is the **byte** length (15), not the character count (13).
- `a + &b` calls `String::add(self, &str)`: it takes `a` **by value** (moving it) and appends `b`, reusing `a`'s buffer. After the line, `a` is gone ✓ and using it is `E0382`:

  ```rust
  fn main() {
      let s1 = String::from("a");
      let s2 = String::from("b");
      let s3 = s1 + &s2;
      println!("{s3} {s2}");
      println!("{s1}");
  }
  ```

  ✓ `error[E0382]: borrow of moved value: `s1``. `&s2` is a `&String` that auto-derefs to `&str` (deref coercion, Lesson 23). For many pieces prefer `format!` or `push_str`.
- Ways to make a `String` from a `&str`: `String::from(s)`, `s.to_string()`, `s.to_owned()`, `s.into()`. They are equivalent in effect (one allocation and copy); choose one and be consistent.

### 1.1 Capacity and reallocation

`String` grows geometrically like `Vec` (Lesson 16). ✓ Pushing 40 single-byte characters into an empty `String` showed capacities `0 8 16 32 64` on this toolchain: a reallocation at 1, 9, 17, and 33 bytes. The exact growth policy is an implementation detail of the standard library, not a guarantee. If you know the size, `String::with_capacity(n)` avoids repeated reallocations ✓ (`len 3, capacity 100`).

---

## 2. Bytes, `char`s, and why there is no `s[0]`

```rust
fn main() { let s = String::from("hello"); let c = s[0]; println!("{c}"); }
```

✓ `error[E0277]: the type `str` cannot be indexed by `{integer}``. This is deliberate: a `str` is bytes, a "character" is a variable number of bytes, so `s[i]` would either return a byte (surprising) or need an O(n) scan (hidden cost). You must say which view you mean:

| You want | Use |
|---|---|
| The bytes | `s.as_bytes()` (`&[u8]`), `s.bytes()` |
| Unicode scalar values | `s.chars()` (iterator of `char`) |
| `char`s with their byte offset | `s.char_indices()` |
| A sub-string by byte range | `&s[a..b]` (panics if not on char boundaries) or `s.get(a..b)` (`Option`) |

```rust
fn main() {
    let s = "héllo";
    println!("{:?}", s.get(0..1));
    println!("{:?}", s.get(0..2));
    println!("{:?}", s.get(0..3));
    println!("{} {}", s.is_char_boundary(2), s.is_char_boundary(3));
    for (i, c) in s.char_indices() { print!("{i}:{c} "); }
    println!();
    let n = std::env::args().count() + 1;     // = 2 when run with no arguments
    println!("{}", &s[0..n]);
}
```

✓ output:

```text
Some("h")
None
Some("hé")
false true
0:h 1:é 3:l 4:l 5:o 

thread 'main' panicked at b03_boundary.rs:10:22:
end byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3 of string)
```

`é` occupies bytes 1 and 2, so a slice ending at byte 2 would cut it in half. Slicing a `str` at a non-boundary **panics** (it cannot return invalid UTF-8); `get` returns `None`.

**Security note.** A server that truncates user text with `&s[..100]` can be crashed by sending a multi-byte character at byte 99. Use `get`, `is_char_boundary`, `char_indices`, or the helper in the practical challenge.

### 2.1 `char`, code points, and "characters"

- A `char` is a Unicode **scalar value** (Lesson 04): 4 bytes in memory, 1 to 4 bytes when encoded as UTF-8 (`'a'`=1, `'é'`=2, `'€'`=3, `'😀'`=4).
- What a person calls a "character" may be several scalar values (a letter plus combining accents, emoji sequences). Those are *grapheme clusters*; the standard library has no grapheme API (the `unicode-segmentation` crate does).

```rust
fn main() {
    let composed = "\u{e9}";            // é as one code point
    let decomposed = "e\u{301}";        // e + combining acute accent
    println!("{} {} {}", composed, decomposed, composed == decomposed);
    println!("{} {}", composed.chars().count(), decomposed.chars().count());
    println!("{} {}", composed.len(), decomposed.len());
    println!("{}", "straße".to_uppercase());
    println!("{} {}", "İ".to_lowercase().len(), "İ".len());
    println!("{}", "Hello".eq_ignore_ascii_case("hELLO"));
    println!("{:?}", "héllo".to_ascii_uppercase());
    let rev: String = "héllo".chars().rev().collect();
    println!("{rev}");
    println!("{}", "a😀".chars().count());
}
```

✓ output:

```text
é é false
1 2
2 3
STRASSE
3 2
true
"HéLLO"
olléh
2
```

Consequences worth memorizing:

- Two strings that **look identical can be unequal**: `"é"` composed vs decomposed compare `false` ✓. Comparing user-visible text (usernames, filenames, identifiers) requires normalization (the `unicode-normalization` crate), and is a classic source of spoofing and duplicate-account bugs.
- Case conversion can **change length**: `"straße".to_uppercase()` is `"STRASSE"` ✓ (`ß` becomes `SS`). That is why `char::to_uppercase` returns an *iterator*, not a `char`. Case-insensitive comparison of general Unicode is subtle; `eq_ignore_ascii_case` only handles ASCII (and `to_ascii_uppercase` leaves `é` untouched ✓).
- Reversing by `chars()` works for this example but breaks combining sequences; "reverse a string" has no single right answer for Unicode text.
- Byte `0..=127` is ASCII, and ASCII-only processing (`is_ascii`, `eq_ignore_ascii_case`, `as_bytes`) is fast and unambiguous. Use it for protocols and identifiers that are defined as ASCII.

### 2.2 From bytes to text: validation

Bytes from files, sockets, and the OS are not guaranteed to be UTF-8. Converting checks validity:

```rust
fn main() {
    println!("{:?}", String::from_utf8(vec![104, 105]));
    let bad = vec![0xff, 0x41];
    println!("{:?}", String::from_utf8(bad.clone()));
    println!("{}", String::from_utf8_lossy(&bad));
    println!("{:?}", std::str::from_utf8(&[0xe2, 0x82]));
    println!("{:?}", std::str::from_utf8(&[0xe2, 0x82, 0xac]));
    println!("{:?}", "€".as_bytes());
    println!("{:?}", b"hi\n");
    println!("{:?}", "a\tb\u{1F600}");
    println!("{}", r"raw\n");
}
```

✓ output (rustc also warned that `from_utf8` on the invalid literal `&[0xe2, 0x82]` always fails):

```text
Ok("hi")
Err(FromUtf8Error { bytes: [255, 65], error: Utf8Error { valid_up_to: 0, error_len: Some(1) } })
�A
Err(Utf8Error { valid_up_to: 0, error_len: None })
Ok("€")
[226, 130, 172]
[104, 105, 10]
"a\tb😀"
raw\n
```

- `String::from_utf8(Vec<u8>)` takes ownership and gives the bytes back inside the error. `str::from_utf8(&[u8])` borrows. Both validate in O(n).
- `error_len: None` means the input ended in the middle of a valid sequence (truncated input; more bytes might complete it): useful when decoding a stream in chunks. `Some(1)` means an invalid byte.
- `String::from_utf8_lossy` replaces invalid sequences with `�` (U+FFFD). Fine for logs and display; **wrong** for data that must round-trip or be authenticated (hashes, signatures, keys), where you should keep `Vec<u8>` and not pretend it is text (Lessons 48 and 51).
- `b"hi\n"` is a byte-string literal (`&[u8; 3]`); `r"..."` is a raw string with no escapes.
- File paths and OS strings are **not** necessarily UTF-8, which is why `std::path::Path` and `std::ffi::OsStr` exist. Do not push them through `String` unless you accept failure or lossy conversion.

---

## 3. The everyday string toolbox

```rust
fn main() {
    let line = "  name = Ada Lovelace ; age=36 ";
    for part in line.split(';') {
        if let Some((k, v)) = part.split_once('=') {
            println!("[{}] [{}]", k.trim(), v.trim());
        }
    }
    println!("{:?}", "a,b,,c".split(',').collect::<Vec<_>>());
    println!("{:?}", "one two  three".split_whitespace().collect::<Vec<_>>());
    println!("{:?}", "l1\nl2\r\nl3".lines().collect::<Vec<_>>());
    println!("{:?} {:?}", "prefix_x".strip_prefix("prefix_"), "x".strip_prefix("y"));
    println!("{} {}", "abc".starts_with("ab"), "abc".contains('c'));
    println!("{:?} {:?}", "hello".find('l'), "hello".rfind('l'));
    println!("{}", "a-b-c".replace('-', "+"));
    println!("{}", "ab".repeat(3));
}
```

✓ output:

```text
[name] [Ada Lovelace]
[age] [36]
["a", "b", "", "c"]
["one", "two", "three"]
["l1", "l2", "l3"]
Some("x") None
true true
Some(2) Some(3)
a+b+c
ababab
```

Notes:

- Most of these return **`&str` slices into the original** rather than new `String`s: `trim`, `split`, `split_once`, `strip_prefix`, `lines` allocate nothing. The results borrow from the input, so the input must outlive them (Lesson 11). This is why parsers in Rust can be allocation-free.
- `split(',')` keeps empty pieces (`"a,b,,c"` has four); `split_whitespace` collapses runs of whitespace. `lines()` handles both `\n` and `\r\n` ✓.
- A *pattern* argument can be a `char`, a `&str`, a closure `|c: char| c.is_ascii_digit()`, or a char slice.
- `find` returns a **byte** offset (✓ `Some(2)`), safe to use for slicing since it always lands on a boundary.

---

## 4. Parsing: `str::parse` and `FromStr`

```rust
use std::str::FromStr;
fn main() {
    println!("{:?}", "42".parse::<i32>());
    println!("{:?}", " 42".parse::<i32>());
    println!("{:?}", "+42".parse::<i32>());
    println!("{:?}", "".parse::<i32>());
    println!("{:?}", "300".parse::<u8>());
    println!("{:?}", "-1".parse::<u8>());
    println!("{:?}", "3.14".parse::<f64>());
    println!("{:?}", "NaN".parse::<f64>().map(|x| x.is_nan()));
    println!("{:?}", "true".parse::<bool>());
    println!("{:?}", "x".parse::<char>());
    println!("{:?}", "xy".parse::<char>());
    println!("{:?}", u8::from_str("12"));
    println!("{:?}", i64::from_str_radix("ff", 16));
    match "abc".parse::<i32>() { Ok(n) => println!("{n}"), Err(e) => println!("error: {e}") }
}
```

✓ output:

```text
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
Ok(42)
Err(ParseIntError { kind: Empty })
Err(ParseIntError { kind: PosOverflow })
Err(ParseIntError { kind: InvalidDigit })
Ok(3.14)
Ok(true)
Ok(true)
Ok('x')
Err(ParseCharError { kind: TooManyChars })
Ok(12)
Ok(255)
error: invalid digit found in string
```

- `parse::<T>()` works for any `T: FromStr`. It returns `Result`, never panics. Handle the `Err` (Lesson 14).
- Parsing is **strict**: surrounding whitespace is an error ✓ (`" 42"`), a leading `+` is accepted ✓, an out-of-range value is `PosOverflow` ✓, and `"-1"` for an unsigned type is `InvalidDigit` ✓. Call `trim()` first if you want leniency.
- `"NaN"` parses as a float NaN ✓ — user-supplied floats can be NaN or infinite. Validate (`is_finite`) before using them.
- The type to parse into must be known: `parse::<i32>()` or an annotated binding. `ParseIntError` implements `Display` ✓ ("invalid digit found in string").

---

## 5. Conversion traits

Lesson 04 contrasted `as`, `From` and `TryFrom` for numbers. They work for all types:

| Trait | Fallible | Direction | Rule of thumb |
|---|---|---|---|
| `From<T>` / `Into<U>` | no | `U::from(t)` ≡ `t.into()` | Implement **`From`**; you get `Into` for free |
| `TryFrom<T>` / `TryInto<U>` | yes (`Result`) | `U::try_from(t)` ≡ `t.try_into()` | Use when some inputs are invalid |
| `FromStr` | yes | `T::from_str(s)` ≡ `s.parse::<T>()` | Parsing text into a type |
| `AsRef<T>` | no | borrow as `&T` cheaply | Generic over "anything string-like" |
| `ToString` (via `Display`) | no | value → `String` | Implement `Display`, not `ToString` |

```rust
use std::convert::TryFrom;
use std::fmt;

#[derive(Debug, PartialEq)]
struct Percent(u8);

#[derive(Debug, PartialEq)]
struct OutOfRange(i64);

impl fmt::Display for OutOfRange {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} is not in 0..=100", self.0)
    }
}

impl TryFrom<i64> for Percent {
    type Error = OutOfRange;
    fn try_from(v: i64) -> Result<Self, Self::Error> {
        if (0..=100).contains(&v) { Ok(Percent(v as u8)) } else { Err(OutOfRange(v)) }
    }
}

struct Celsius(f64);
struct Fahrenheit(f64);
impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self { Fahrenheit(c.0 * 9.0 / 5.0 + 32.0) }
}

fn takes_into(s: impl Into<String>) -> usize { s.into().len() }
fn takes_asref(s: impl AsRef<str>) -> usize { s.as_ref().len() }

fn main() {
    println!("{:?} {:?}", Percent::try_from(50), Percent::try_from(101));
    let p: Result<Percent, _> = 7i64.try_into();
    println!("{p:?}");
    let f: Fahrenheit = Celsius(100.0).into();
    println!("{}", f.0);
    println!("{} {}", takes_into("abc"), takes_into(String::from("hello")));
    println!("{} {}", takes_asref("abc"), takes_asref(String::from("hello")));
    let b: Box<str> = "boxed".into();
    let v: Vec<u8> = "hi".into();
    let s: String = 'c'.into();
    println!("{b} {v:?} {s}");
    println!("{}", OutOfRange(101));
}
```

✓ output:

```text
Ok(Percent(50)) Err(OutOfRange(101))
Ok(Percent(7))
212
3 5
3 5
boxed [104, 105] c
101 is not in 0..=100
```

Design guidance:

- **Make invalid states unrepresentable at the boundary.** `Percent::try_from` is the *only* way to build a `Percent` from an `i64`, so every `Percent` in the program is in range. That is the newtype pattern (Lesson 15) in miniature. (The `v as u8` inside is safe because the range was just checked; a comment saying so is good practice.)
- `impl Into<String>` accepts `&str` and `String`, and consumes the argument: a `String` input is reused without copying, a `&str` input is copied once. `impl AsRef<str>` only *borrows*, so it is better when you don't need ownership. Prefer plain `&str` for the common case and reach for these only for ergonomic flexibility; every generic parameter adds a monomorphized copy (Lesson 19).
- Implement `From` not `Into`, and `Display` not `ToString`: the blanket implementations give you the other for free.
- Conversions should be **cheap and obvious**. Anything expensive or surprising should be a named method (`to_vec()`, `into_boxed_slice()`) rather than `From`.
- `From` is also what makes `?` convert error types (Lesson 14).

---

## 6. Pattern matching

A **pattern** describes the shape of a value and (optionally) binds parts of it to names. Patterns appear in `match`, `let`, function parameters, `if let`, `while let`, `for`, and closures' parameters. Matching is checked **at compile time for exhaustiveness**.

### 6.1 `match`: ranges, alternatives, guards, bindings

```rust
fn describe(n: i32) -> &'static str {
    match n {
        i32::MIN..=-1 => "negative",
        0 => "zero",
        1 | 2 | 3 => "small",
        x if x % 2 == 0 => "big even",
        _ => "big odd",
    }
}

fn bucket(age: u8) -> String {
    match age {
        n @ 0..=12 => format!("child {n}"),
        n @ 13..=19 => format!("teen {n}"),
        n => format!("adult {n}"),
    }
}

fn main() {
    for n in [-5, 0, 2, 10, 11] { print!("{} ", describe(n)); }
    println!();
    println!("{} {} {}", bucket(5), bucket(15), bucket(40));
    let pair = (3, -3);
    match pair {
        (x, y) if x + y == 0 => println!("opposites"),
        (x, _) if x % 2 == 1 => println!("odd x"),
        _ => println!("other"),
    }
    let c = 'k';
    println!("{}", match c { 'a'..='j' => 1, 'k'..='z' => 2, _ => 3 });
    println!("{}", matches!(Some(5), Some(n) if n > 3));
}
```

✓ output:

```text
negative zero small big even big odd 
child 5 teen 15 adult 40
opposites
2
true
```

- `|` combines alternatives; `a..=b` is an inclusive range pattern (integers and `char`); `x @ pattern` binds the matched value to `x` while also testing `pattern`; `if cond` is a **guard** evaluated after the pattern matches. `_` matches anything and binds nothing; `..` ignores "the rest" of a tuple, array, or struct.
- Arms are tried **top to bottom**; the first match wins. A `match` is an expression, so all arms must have the same type.
- Guards are not considered by exhaustiveness checking: the compiler cannot reason about arbitrary `if` conditions, so `match` with only guarded arms usually needs a final catch-all.
- `matches!(value, pattern)` is a macro that returns a `bool`.

### 6.2 Exhaustiveness: the compiler as a reviewer

```rust
fn main() {
    let n = 3u8;
    let s = match n {
        0 => "zero",
        1..=100 => "small",
    };
    println!("{s}");
}
```

✓ `error[E0004]: non-exhaustive patterns: `101_u8..=u8::MAX` not covered`. The compiler computes which values are missing and names them. This is why `enum` matches (Lesson 13) are so valuable: add a variant, and every `match` that forgot it stops compiling. Use `_ =>` only when you really mean "everything else"; a wildcard silently swallows future variants.

In the other direction, an arm that can never match is a warning:

```rust
fn main() {
    let n = 3;
    match n {
        _ => println!("any"),
        3 => println!("three"),
    }
}
```

✓ `warning: unreachable pattern` (the `3` arm is dead because `_` came first).

### 6.3 Destructuring tuples, arrays, and slices

```rust
fn describe(v: &[i32]) -> String {
    match v {
        [] => "empty".to_string(),
        [x] => format!("one: {x}"),
        [first, .., last] if first == last => format!("same ends {first}"),
        [first, rest @ ..] => format!("first {first}, {} more", rest.len()),
    }
}

fn main() {
    println!("{}", describe(&[]));
    println!("{}", describe(&[7]));
    println!("{}", describe(&[1, 2, 1]));
    println!("{}", describe(&[1, 2, 3, 4]));
    let [a, b, c] = [1, 2, 3];
    println!("{a} {b} {c}");
    let ((x, y), z) = ((1, 2), 3);
    println!("{x} {y} {z}");
    let [head, ..] = [9, 8, 7];
    println!("{head}");
}
```

✓ output:

```text
empty
one: 7
same ends 1
first 1, 3 more
1 2 3
1 2 3
9
```

Slice patterns (`[first, .., last]`, `[first, rest @ ..]`) are bounds-safe by construction: the compiler knows the length each arm requires, so no index can go out of range. They are often clearer and safer than `v[0]`, `v[v.len() - 1]` (Lesson 05).

### 6.4 `if let`, `while let`, `let else`

```rust
fn parse_kv(s: &str) -> Option<(&str, i32)> {
    let (k, v) = s.split_once('=')?;
    let Ok(n) = v.trim().parse::<i32>() else { return None; };
    Some((k.trim(), n))
}

fn main() {
    println!("{:?} {:?} {:?}", parse_kv("a = 5"), parse_kv("a=x"), parse_kv("nokey"));
    let mut stack = vec![1, 2, 3];
    while let Some(top) = stack.pop() { print!("{top} "); }
    println!();
    if let Some(n) = "12".parse::<i32>().ok() { println!("got {n}"); } else { println!("none"); }
}
```

✓ output:

```text
Some(("a", 5)) None None
3 2 1 
got 12
```

- `if let PATTERN = expr { ... } else { ... }`: run a block only if the value matches.
- `while let`: loop while the pattern keeps matching (here, until `pop()` returns `None`).
- `let PATTERN = expr else { diverge };`: bind on success, otherwise the `else` block **must diverge** (`return`, `break`, `continue`, `panic!`). It keeps the happy path unindented — Rust's idiom for guard clauses on patterns.
- Plain `let` requires an **irrefutable** pattern (one that always matches). A refutable pattern in `let` is an error:

  ```rust
  fn main() {
      let o: Option<i32> = Some(1);
      let Some(x) = o;
      println!("{x}");
  }
  ```

  ✓ `error[E0005]: refutable pattern in local binding` (the compiler suggests `let ... else` or `if let`).

### 6.5 Matching moves unless you borrow

Binding a non-`Copy` value by name in a pattern **moves** it out:

```rust
fn main() {
    let opt = Some(String::from("hi"));
    if let Some(s) = opt { println!("{s}"); }
    println!("{opt:?}");
}
```

✓ `error[E0382]: borrow of partially moved value: `opt``. The `String` inside moved into `s`. Fix by matching on a reference, `if let Some(s) = &opt`, or with `ref`:

```rust
fn main() {
    let opt = Some(String::from("hi"));
    if let Some(ref s) = opt { println!("{s}"); }
    println!("{opt:?}");
}
```

✓ prints `hi` then `Some("hi")`. Since Rust 1.26 ("match ergonomics", also called default binding modes), matching `&opt` against `Some(s)` makes `s` a reference automatically, so `if let Some(s) = &opt` is the usual spelling; explicit `ref` is rarely needed in new code (✓ in edition 2024, writing `ref` inside a pattern that is already implicitly borrowing, such as `if let Some(ref s) = r` where `r: &Option<String>`, is an error: "explicit `ref` binding modifier not allowed when implicitly borrowing"; so prefer matching on `&opt`). Move semantics are the subject of Lesson 07.

---

## 7. Common mistakes and edge cases

1. `&s[..n]` with a user-controlled `n`: panics on out-of-range or mid-character indices. Use `get`, `is_char_boundary`, or char-based logic.
2. Treating `len()` as a character count, or the n-th `char` as the n-th "letter".
3. Comparing user-visible text for equality without normalization; assuming case conversion preserves length.
4. `String::from_utf8_lossy` on data that must round-trip.
5. `parse()` without trimming, or `unwrap()` on user input.
6. Assuming `"NaN"`, `"inf"` cannot parse as floats.
7. `a + &b` when you still need `a`; building strings in a loop with `+` instead of `push_str` / `format!`.
8. A `match` with a catch-all `_` that hides new enum variants.
9. A refutable pattern in plain `let` (E0005); a guard-only `match` without a fallback.
10. Matching a non-`Copy` field by value and then using the original (E0382).
11. Forgetting that most `&str` helpers return borrowed slices tied to the input.
12. Pushing OS paths and file names through `String`.

## 8. Performance implications

- `String` operations allocate; slicing, `trim`, `split` and friends do not. Prefer borrowing (`&str`) in function parameters and return `String` only when you create new text.
- `chars().count()`, `char_indices`, and `s.chars().nth(i)` are O(n); `len()`, slicing at known boundaries, and `as_bytes()[i]` are O(1).
- Building a string of known size: `String::with_capacity`. Repeated `s = s + ...` or `format!` in a loop allocates more than `push_str` onto one buffer.
- ASCII-only work is cheaper and simpler: `bytes()` or `as_bytes()` avoid UTF-8 decoding.
- `match` on integers and `char` ranges compiles to jump tables or comparisons; matching is not slower than hand-written `if` chains, and the compiler may optimize it better. I did not measure this; benchmark before choosing a structure for speed (Lesson 41).
- `impl Into<String>` and `impl AsRef<str>` generate one copy of the function per concrete type (monomorphization, Lesson 19). Use `&str` in code that is hot or numerous.

## 9. Security and reliability

- Text is **untrusted input** in almost every service. Slicing panics, invalid UTF-8, huge inputs, NaN floats, and confusable look-alike characters (homoglyphs, combining marks) are all reachable from the network.
- Enforce limits in **bytes** before allocating or parsing; then validate encoding; then normalize if identity matters; then parse. Cap line lengths and field counts.
- Never use lossy decoding on anything that is hashed, signed, or compared for authentication.
- Do not build SQL, shell commands, or file paths by string concatenation of user input; use parameterized APIs (Lessons 45–46).
- Exhaustive `match` is a reliability tool: prefer it to catch-alls so that new cases force a decision at compile time.
- Error messages that echo input can leak data or enable log injection (embedded newlines): the `{:?}` Debug format escapes control characters ✓ (`"a\tb😀"`), which is why error types often print inputs with `{:?}`.

---

## 10. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Print the number of bytes and the number of `char`s of `"naïve café"`. Explain the difference.
2. **(Easy)** Write `fn is_palindrome_ascii(s: &str) -> bool` that ignores case and any non-alphanumeric ASCII characters.
3. **(Medium)** Write `fn word_count(s: &str) -> usize` without allocating a `Vec`.
4. **(Medium)** Implement `FromStr` for `struct Point { x: i32, y: i32 }` from the text `"3,-4"` (whitespace around numbers allowed), with a dedicated error type.
5. **(Medium)** Write `fn classify(v: &[i32]) -> &'static str` using slice patterns: `"empty"`, `"single"`, `"pair"`, `"many"`.
6. **(Hard)** Write `fn first_n_chars(s: &str, n: usize) -> &str` that returns the first `n` `char`s (not bytes) of `s` without allocating and without panicking for any `n`.
7. **(Hard)** Show with code why `"é"` (composed) and `"e\u{301}"` (decomposed) are different strings, then describe in two sentences what you would do in a signup form to avoid duplicate usernames that differ only in this way.

<details markdown="1">
<summary>Answers</summary>

1. ✓ `"naïve café"` is 10 chars and 12 bytes (`ï` and `é` are 2 bytes each). Bytes are the UTF-8 encoding length; `chars()` counts Unicode scalar values. (Printing: `println!("{} {}", s.len(), s.chars().count());`.)
2. ```rust
   fn is_palindrome_ascii(s: &str) -> bool {
       let mut it = s.chars().filter(|c| c.is_ascii_alphanumeric()).map(|c| c.to_ascii_lowercase());
       // compare from both ends without collecting
       loop {
           match (it.next(), it.next_back()) {
               (Some(a), Some(b)) if a != b => return false,
               (Some(_), Some(_)) => {}
               _ => return true,
           }
       }
   }
   ```
   (`next_back` requires a double-ended iterator; `filter` and `map` preserve that.) Odd-length strings end when the middle element is consumed from the front and `next_back` returns `None`. ✓ Tested: the Panama sentence → `true`, `"hello"` → `false`, `""` → `true`, `"ab"` → `false`.
3. ✓ `s.split_whitespace().count()`: it iterates lazily and does not allocate (`"  a  b c "` → 3).
4. ✓ Tested.
   ```rust
   #[derive(Debug, PartialEq)]
   struct Point { x: i32, y: i32 }
   #[derive(Debug, PartialEq)]
   enum PointError { MissingComma, BadX(ParseIntError), BadY(ParseIntError) }
   impl FromStr for Point {
       type Err = PointError;
       fn from_str(s: &str) -> Result<Self, Self::Err> {
           let (a, b) = s.split_once(',').ok_or(PointError::MissingComma)?;
           let x = a.trim().parse().map_err(PointError::BadX)?;
           let y = b.trim().parse().map_err(PointError::BadY)?;
           Ok(Point { x, y })
       }
   }
   ```
   (with `use std::num::ParseIntError; use std::str::FromStr;`). `"3,-4"` and `" 1 , 2 "` parse; `"3;4"` gives `MissingComma`; `"x,1"` gives `BadX(..)`.
5. ✓ Tested.
   ```rust
   fn classify(v: &[i32]) -> &'static str {
       match v {
           [] => "empty",
           [_] => "single",
           [_, _] => "pair",
           _ => "many",
       }
   }
   ```
6. ✓ Tested (`"aé€😀z"` with `n` = 0, 3, 5, 99 → `""`, `"aé€"`, whole string, whole string).
   ```rust
   fn first_n_chars(s: &str, n: usize) -> &str {
       match s.char_indices().nth(n) {
           Some((byte_idx, _)) => &s[..byte_idx],
           None => s,
       }
   }
   ```
   `char_indices().nth(n)` yields the byte offset where the `(n+1)`-th char starts (always a boundary), or `None` if the string has `n` or fewer chars.
7. ✓ The program in Section 2.1 prints `false` for `composed == decomposed`, with 1 vs 2 chars and 2 vs 3 bytes. For usernames: normalize the input to a single Unicode normalization form (for example NFKC, via a crate such as `unicode-normalization`) before comparing and before storing, and consider restricting the allowed character set (and blocking mixed-script look-alikes) per the guidance in Unicode's security recommendations (UTS #39). I did not run a normalization crate in this lesson.

</details>

---

## 11. Debugging challenge

This function is meant to uppercase the first letter of a word:

```rust
fn capitalize(s: &str) -> String {
    let mut out = s[..1].to_uppercase();
    out.push_str(&s[1..]);
    out
}
```

It works for `"hello"` (✓ `Hello`). Find **two inputs that make it panic**, explain why in terms of bytes and `char` boundaries, and fix it. Then explain one more property of Unicode case conversion that a fix based on `to_ascii_uppercase` would get wrong.

<details markdown="1">
<summary>Solution</summary>

1. **Empty string.** `capitalize("")`: `s[..1]` asks for byte 1 of a 0-byte string. ✓ `panicked: end byte index 1 is out of bounds for string of length 0`.
2. **A first character wider than one byte.** `capitalize("élan")` (and `"ß"`): `é` is 2 bytes, so `..1` ends inside it. ✓ `panicked: end byte index 1 is not a char boundary; it is inside 'é' (bytes 0..2 of string)`.

The root cause is the assumption "one character = one byte", that is, treating a byte index as a character index.

Fix: work with `char`s, which also handles the empty string:

```rust
fn capitalize(s: &str) -> String {
    let mut chars = s.chars();
    match chars.next() {
        None => String::new(),
        Some(first) => first.to_uppercase().chain(chars).collect(),
    }
}
```

✓ Tested in the `textlab` crate (practical challenge): `""` → `""`, `"hello"` → `"Hello"`, `"élan"` → `"Élan"`, `"ßa"` → `"SSa"`.

The extra Unicode property: **uppercasing one `char` can produce several** (`ß` → `SS`). That is why `char::to_uppercase` returns an iterator (which the fix chains onto the rest of the string), and why a fix that converts a single `char` with `to_ascii_uppercase` would leave `é` and `ß` unchanged ✓ (`"héllo".to_ascii_uppercase()` is `"HéLLO"`). Also, "capitalize" is locale-dependent in some languages (for example Turkish dotted/dotless i), which the standard library's locale-independent conversion does not handle.

</details>

---

## 12. Practical challenge: `textlab`

Create a library crate `textlab` (edition 2024) with:

1. `truncate_bytes(s: &str, max_bytes: usize) -> &str`: the longest prefix of at most `max_bytes` bytes that ends on a `char` boundary; never panics.
2. `capitalize` from the debugging challenge.
3. `parse_config(input: &str) -> Result<Vec<(String, i64)>, ConfigError>`: lines of `key = integer`; blank lines and `#` comments ignored; keys must be unique; errors carry **line numbers** (1-based) and implement `Display` and `std::error::Error`.
4. A tiny command language `Cmd` implementing `FromStr`: `add A B`, `neg A`, `sum A B C ...` (zero or more numbers), parsed using **slice patterns** on the whitespace-separated tokens, with a `CmdError` for empty input, unknown command, wrong arity, and bad numbers.
5. Tests for the edge cases above, including a loop that calls `truncate_bytes` on `"aé€😀z"` with *every* limit from 0 to `len + 2`.
6. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! Text-handling helpers: boundary-safe truncation, a tiny config parser,
//! and a command parser built on slice patterns.

use std::fmt;
use std::num::ParseIntError;
use std::str::FromStr;

/// Returns the longest prefix of `s` that is at most `max_bytes` long and
/// ends on a `char` boundary.
pub fn truncate_bytes(s: &str, max_bytes: usize) -> &str {
    if max_bytes >= s.len() {
        return s;
    }
    let mut end = max_bytes;
    while !s.is_char_boundary(end) {
        end -= 1; // index 0 is always a boundary, so this terminates
    }
    &s[..end]
}

/// Uppercases the first character (which may become several characters).
pub fn capitalize(s: &str) -> String {
    let mut chars = s.chars();
    match chars.next() {
        None => String::new(),
        Some(first) => first.to_uppercase().chain(chars).collect(),
    }
}

/// Errors from [`parse_config`]. Line numbers start at 1.
#[derive(Debug, PartialEq)]
pub enum ConfigError {
    MissingEquals { line: usize },
    EmptyKey { line: usize },
    BadNumber { line: usize, source: ParseIntError },
    DuplicateKey { line: usize, key: String },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::MissingEquals { line } => write!(f, "line {line}: expected key=value"),
            ConfigError::EmptyKey { line } => write!(f, "line {line}: empty key"),
            ConfigError::BadNumber { line, source } => write!(f, "line {line}: {source}"),
            ConfigError::DuplicateKey { line, key } => {
                write!(f, "line {line}: duplicate key {key:?}")
            }
        }
    }
}

impl std::error::Error for ConfigError {}

/// Parses lines of `key = integer`. Blank lines and lines starting with `#`
/// are ignored. Keys must be unique.
pub fn parse_config(input: &str) -> Result<Vec<(String, i64)>, ConfigError> {
    let mut out: Vec<(String, i64)> = Vec::new();
    for (idx, raw) in input.lines().enumerate() {
        let line = idx + 1;
        let text = raw.trim();
        if text.is_empty() || text.starts_with('#') {
            continue;
        }
        let Some((key, value)) = text.split_once('=') else {
            return Err(ConfigError::MissingEquals { line });
        };
        let key = key.trim();
        if key.is_empty() {
            return Err(ConfigError::EmptyKey { line });
        }
        let value = value
            .trim()
            .parse::<i64>()
            .map_err(|source| ConfigError::BadNumber { line, source })?;
        if out.iter().any(|(k, _)| k == key) {
            return Err(ConfigError::DuplicateKey {
                line,
                key: key.to_string(),
            });
        }
        out.push((key.to_string(), value));
    }
    Ok(out)
}

/// A tiny command language: `add A B`, `neg A`, `sum A B C ...`.
#[derive(Debug, PartialEq)]
pub enum Cmd {
    Add(i64, i64),
    Neg(i64),
    Sum(Vec<i64>),
}

#[derive(Debug, PartialEq)]
pub enum CmdError {
    Empty,
    UnknownCommand(String),
    WrongArity {
        name: &'static str,
        expected: usize,
        got: usize,
    },
    BadNumber(String),
}

fn num(s: &str) -> Result<i64, CmdError> {
    s.parse().map_err(|_| CmdError::BadNumber(s.to_string()))
}

impl FromStr for Cmd {
    type Err = CmdError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let tokens: Vec<&str> = s.split_whitespace().collect();
        match tokens.as_slice() {
            [] => Err(CmdError::Empty),
            ["add", a, b] => Ok(Cmd::Add(num(a)?, num(b)?)),
            ["add", rest @ ..] => Err(CmdError::WrongArity {
                name: "add",
                expected: 2,
                got: rest.len(),
            }),
            ["neg", a] => Ok(Cmd::Neg(num(a)?)),
            ["neg", rest @ ..] => Err(CmdError::WrongArity {
                name: "neg",
                expected: 1,
                got: rest.len(),
            }),
            ["sum", rest @ ..] => Ok(Cmd::Sum(
                rest.iter().map(|t| num(t)).collect::<Result<_, _>>()?,
            )),
            [other, ..] => Err(CmdError::UnknownCommand((*other).to_string())),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn truncate_respects_boundaries() {
        assert_eq!(truncate_bytes("héllo", 0), "");
        assert_eq!(truncate_bytes("héllo", 1), "h");
        assert_eq!(truncate_bytes("héllo", 2), "h"); // inside 'é'
        assert_eq!(truncate_bytes("héllo", 3), "hé");
        assert_eq!(truncate_bytes("héllo", 99), "héllo");
        assert_eq!(truncate_bytes("😀", 3), "");
        assert_eq!(truncate_bytes("😀", 4), "😀");
    }

    #[test]
    fn truncate_never_panics_for_any_limit() {
        let s = "aé€😀z";
        for n in 0..=s.len() + 2 {
            let t = truncate_bytes(s, n);
            assert!(t.len() <= n && s.starts_with(t));
        }
    }

    #[test]
    fn capitalize_cases() {
        assert_eq!(capitalize(""), "");
        assert_eq!(capitalize("hello"), "Hello");
        assert_eq!(capitalize("élan"), "Élan");
        assert_eq!(capitalize("ßa"), "SSa");
    }

    #[test]
    fn config_ok() {
        let got = parse_config("# c\n\n a = 1 \nb=-2\n").unwrap();
        assert_eq!(got, vec![("a".to_string(), 1), ("b".to_string(), -2)]);
    }

    #[test]
    fn config_errors() {
        assert_eq!(
            parse_config("x"),
            Err(ConfigError::MissingEquals { line: 1 })
        );
        assert_eq!(
            parse_config("a=1\n=2"),
            Err(ConfigError::EmptyKey { line: 2 })
        );
        assert!(matches!(
            parse_config("a=zz"),
            Err(ConfigError::BadNumber { line: 1, .. })
        ));
        assert_eq!(
            parse_config("a=1\na=2"),
            Err(ConfigError::DuplicateKey {
                line: 2,
                key: "a".into()
            })
        );
        let msg = parse_config("a=zz").unwrap_err().to_string();
        assert_eq!(msg, "line 1: invalid digit found in string");
    }

    #[test]
    fn commands() {
        assert_eq!("add 1 2".parse(), Ok(Cmd::Add(1, 2)));
        assert_eq!("neg -5".parse(), Ok(Cmd::Neg(-5)));
        assert_eq!("sum".parse(), Ok(Cmd::Sum(vec![])));
        assert_eq!("sum 1 2 3".parse(), Ok(Cmd::Sum(vec![1, 2, 3])));
        assert_eq!("".parse::<Cmd>(), Err(CmdError::Empty));
        assert_eq!(
            "add 1".parse::<Cmd>(),
            Err(CmdError::WrongArity {
                name: "add",
                expected: 2,
                got: 1
            })
        );
        assert_eq!("neg x".parse::<Cmd>(), Err(CmdError::BadNumber("x".into())));
        assert_eq!(
            "mul 1 2".parse::<Cmd>(),
            Err(CmdError::UnknownCommand("mul".into()))
        );
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 6 passed.

**Design notes.**

- `truncate_bytes` loops downward from `max_bytes` until `is_char_boundary` is true. Index 0 is always a boundary, so the loop terminates; if `max_bytes >= len` the whole string is returned. The result borrows from the input (no allocation).
- `ConfigError` carries the data needed to report the problem (line number, offending key, the underlying `ParseIntError`); `Display` produces the human message and the `Debug` derive provides the programmatic one. In a real application you would add `source()` for `BadNumber` so error chains can be walked (Lesson 26).
- `parse_config` uses `let ... else` for the missing `=` case and `map_err` to attach the line number to the parse error. Duplicate detection is a linear scan: O(n²) for n keys, fine for small files; a `HashSet` or `BTreeSet` is the next step for large inputs (Lesson 16).
- `Cmd::from_str` matches on `tokens.as_slice()`. The arm order matters: the specific `["add", a, b]` comes before the catch-all `["add", rest @ ..]` that reports the arity error, and the final `[other, ..]` handles unknown commands. The compiler's exhaustiveness check confirms every token list is covered (the `[]` arm is first).
- Numbers are parsed as `i64`; arithmetic on them (when you evaluate commands) must use `checked_add` / `checked_neg` (Lesson 04): `neg i64::MIN` overflows.

</details>

---

## 13. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **`String` vs `&str`?** `String` owns a growable heap buffer; `&str` is a borrowed view (pointer + length) of UTF-8 bytes owned elsewhere (a `String`, a literal, a file buffer). Prefer `&str` for parameters.
2. **Why can't you index a `String` with an integer?** A character occupies 1 to 4 bytes in UTF-8, so a byte index is not a character index and a character lookup would be O(n). You must choose `bytes()`, `chars()`, or a byte-range slice.
3. **What happens if you slice a string in the middle of a character?** `&s[a..b]` panics; `s.get(a..b)` returns `None`.
4. **Is `s.len()` the number of characters?** No: the number of bytes. `s.chars().count()` counts Unicode scalar values, and even that is not always what users call characters (grapheme clusters).
5. **Why is `"é" == "e\u{301}"` false?** String equality compares bytes; the two are different byte sequences representing canonically equivalent text. Normalization is required to compare them as text.
6. **Can `to_uppercase` change the length of a string?** Yes (`"ß"` becomes `"SS"`), which is why `char::to_uppercase` returns an iterator.
7. **`From` vs `TryFrom` vs `as`?** `From` is infallible and lossless; `TryFrom` returns `Result` for conversions that can fail; `as` is a primitive cast that silently truncates or reinterprets (Lesson 04).
8. **Why implement `From` rather than `Into`?** A `From<T> for U` impl automatically provides `Into<U> for T`; the reverse is not true. Similarly implement `Display` and get `ToString`.
9. **What does `parse::<i32>()` do on `" 42"`?** It returns `Err(InvalidDigit)`; parsing does not trim.
10. **What does exhaustiveness checking give you?** The compiler proves that a `match` covers every possible value, and tells you which are missing. Adding a new enum variant breaks every non-exhaustive `match`, which prevents forgotten cases.
11. **Difference between `if let` and `let ... else`?** `if let` runs a block when the pattern matches. `let else` binds names for the rest of the scope when it matches, and otherwise runs a block that must diverge. The second avoids rightward drift for guard clauses.
12. **Why does `if let Some(s) = opt` followed by use of `opt` fail?** The pattern binds `s` by value, moving the `String` out of `opt`. Match on `&opt` (or use `ref`) to borrow instead.
13. **When is `String::from_utf8_lossy` inappropriate?** Whenever data must be preserved exactly (hashing, signing, round-tripping, binary formats): it replaces invalid bytes with U+FFFD, destroying information.

</details>

---

## 14. Key takeaways

- `str`/`String` are valid UTF-8 by construction; `len()` counts bytes, `chars()` scalar values, and neither equals user-perceived characters. No integer indexing: choose the view you mean.
- Byte-range slicing must land on char boundaries or it panics; use `get`/`is_char_boundary` for untrusted indices.
- Most `&str` helpers return borrowed slices (no allocation); `String` is for new or owned text; `+` moves its left operand.
- Validate bytes → text with `from_utf8`; use lossy conversion only for display. Parse with `parse::<T>()`/`FromStr` (strict, returns `Result`).
- Implement `From`/`TryFrom` (not `Into`) and `Display` (not `ToString`); use `TryFrom` to make invalid values unrepresentable.
- Patterns destructure and test shape: ranges, `|`, guards, `@`, tuple/array/slice patterns. `match` is checked for exhaustiveness; plain `let` needs an irrefutable pattern; `if let`/`while let`/`let else` handle the refutable cases. Binding by value moves.

**Connections.** Lesson 04's `char`, `TryFrom` and overflow checks reappear in parsing and `Cmd`; Lesson 05's slices and `get` are what `&str` really is. Next: Lesson 07 (ownership, moves, `Copy`, `Clone`) explains the `E0382` errors you met here (`a + &b`, matching a `String` out of an `Option`). Lesson 13 brings enums and exhaustive matching to your own types; Lesson 14 handles `Option`, `Result`, and `?` (which uses `From`); Lesson 16 covers `String` capacity and collections; Lesson 26 covers error design; Lessons 44–45 apply these parsing rules at API boundaries. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 15. References

- *The Rust Programming Language*, ch. 8.2 "Storing UTF-8 Encoded Text with Strings": <https://doc.rust-lang.org/book/ch08-02-strings.html>
- *The Rust Programming Language*, ch. 6 "Enums and Pattern Matching" and ch. 19 "Patterns and Matching": <https://doc.rust-lang.org/book/ch06-00-enums.html>, <https://doc.rust-lang.org/book/ch19-00-patterns.html>
- *The Rust Reference*, patterns: <https://doc.rust-lang.org/reference/patterns.html>
- *The Rust Reference*, `let` statements (including `let else`): <https://doc.rust-lang.org/reference/statements.html#let-statements>
- Rust standard library documentation, `str`: <https://doc.rust-lang.org/std/primitive.str.html>
- Rust standard library documentation, `String`: <https://doc.rust-lang.org/std/string/struct.String.html>
- Rust standard library documentation, `From`, `TryFrom`, `AsRef`, and `FromStr`: <https://doc.rust-lang.org/std/convert/trait.From.html>, <https://doc.rust-lang.org/std/convert/trait.TryFrom.html>, <https://doc.rust-lang.org/std/convert/trait.AsRef.html>, <https://doc.rust-lang.org/std/str/trait.FromStr.html>
- Rust API Guidelines, conversions (`as_`, `to_`, `into_` naming): <https://rust-lang.github.io/api-guidelines/naming.html>
- The Unicode Standard, UTF-8 and normalization forms (UAX #15): <https://www.unicode.org/reports/tr15/>
- Unicode Technical Standard #39, Unicode Security Mechanisms: <https://www.unicode.org/reports/tr39/>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title.
