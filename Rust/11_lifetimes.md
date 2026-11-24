---
title: "Lesson 11 — Lifetimes: annotations, elision, structs that borrow, and 'static"
---

# Lesson 11 — Lifetimes: annotations, elision, structs that borrow, and `'static`

**Module 4 · Borrowing and lifetimes** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Where behavior differs between edition 2021 and 2024 I say so (Section 7).

Builds on [Lesson 10](10_references_and_borrowing.md) (borrow conflicts; E0597/E0106/E0515 were previewed there), [Lesson 08](08_stack_heap_drop.md) (scopes, drop order) and [Lesson 07](07_ownership_and_moves.md) (what a move does to the old binding).

---

## 1. What a lifetime is (and is not)

Lesson 10 was about references **conflicting** with each other. This lesson is about a reference **outliving** the thing it points at, the dangling-pointer bug.

A **lifetime** is the compile-time name for a region of the program during which a reference is valid. Three facts to hold on to:

1. **Lifetimes exist only in the compiler.** They are erased before code generation; there is no run-time representation, no cost, no check. (Same bet as Lesson 10: pay in compile errors, not at run time.)
2. **Annotations describe; they never extend.** Writing `'a` cannot make a value live longer. It tells the compiler how the lifetimes of inputs and outputs *relate*, so it can verify every caller and the function body against that contract.
3. **The compiler infers almost all of them inside a function body.** You write lifetimes explicitly mainly at **function signatures** and **type definitions**, where the compiler needs a contract (it checks each function in isolation, never by looking at its callers' bodies).

```rust
fn longest(a: &str, b: &str) -> &str { if a.len() >= b.len() { a } else { b } }
fn main() { println!("{}", longest("ab", "c")); }
```

✓ `error[E0106]: missing lifetime specifier`. The returned reference must borrow from *something*; with two candidate inputs the compiler cannot guess which, and it will not guess. Without the contract, the caller could not know how long the result stays valid.

```rust
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str { if a.len() >= b.len() { a } else { b } }
fn main() {
    let s1 = String::from("long string");
    let result;
    {
        let s2 = String::from("xyz");
        result = longest(&s1, &s2);
        println!("inside: {result}");
    }
    println!("{}", longest("ab", "c"));
}
```

✓ Output `inside: long string`, `ab`. Read the signature as: "for some region `'a`: both inputs are valid for all of `'a`, and the output is valid for `'a`." At the call site `'a` becomes the **overlap** of the two argument lifetimes, so the result is valid only while *both* inputs are.

Use the result after the shorter-lived input is gone and you see why:

```rust
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str { if a.len() >= b.len() { a } else { b } }
fn main() {
    let s1 = String::from("long string");
    let result;
    {
        let s2 = String::from("xyz");
        result = longest(&s1, &s2);
    }                       // s2 dropped here
    println!("{result}");   // E0597: result might point at s2
}
```

✓ `error[E0597]: `s2` does not live long enough`. The compiler cannot know which branch ran, so it must assume the output could be `s2`. Remember the shape of E0597: *a value is dropped while a reference to it is still going to be used.*

**Why elision exists.** The design is "named lifetime parameters, with elision for the common cases", so that most code reads as if it had no lifetimes at all (Section 3). You will mostly *read* lifetime annotations, and write them when a signature genuinely relates several references. (I am not making claims here about how the design evolved before 1.0.)

---

## 2. Returning references to locals: E0515

The simplest dangling reference:

```rust
fn make() -> &'static str { let s = String::from("hi"); &s }
fn first_word(s: &str) -> &str { let owned = s.to_uppercase(); &owned[..1] }
fn main() { println!("{} {}", make(), first_word("abc")); }
```

✓ Both functions fail with `error[E0515]: cannot return reference to local variable `s`` / `cannot return value referencing local variable `owned``. A local is dropped when the function returns (Lesson 08), so a reference to it would dangle. Fixes: return the **owned** value (`String`), or return a reference derived from an *input* (not from a local you built), or return `Cow` when you sometimes need to allocate (Lesson 16). No amount of lifetime annotation can fix this; an annotation cannot make `owned` live longer.

---

## 3. Lifetime elision: why you rarely write them

The compiler applies three rules to function signatures. If, after applying them, every output lifetime is determined, you may omit the annotations; otherwise you get E0106.

1. Each elided input lifetime becomes a **distinct** lifetime parameter. (`fn f(a: &str, b: &str)` is `fn f<'1, '2>(a: &'1 str, b: &'2 str)`.)
2. If there is **exactly one** input lifetime (elided or not), it is assigned to all elided output lifetimes. (`fn first_word(s: &str) -> &str`.)
3. If one of the inputs is `&self` or `&mut self`, **its** lifetime is assigned to all elided outputs. (Methods almost always want this.)

```rust
// Pairs below are equivalent: elided vs explicit
fn first_word(s: &str) -> &str { s.split(' ').next().unwrap_or("") }
fn first_word_explicit<'a>(s: &'a str) -> &'a str { s.split(' ').next().unwrap_or("") }
struct Parser { buf: String }
impl Parser {
    fn head(&self, _other: &str) -> &str { &self.buf[..1] }   // rule 3: tied to &self
}
fn main() {
    let p = Parser { buf: "xyz".into() };
    println!("{} {} {}", first_word("a b"), first_word_explicit("c d"), p.head("zzz"));
}
```

✓ Output `a c x`. And when the rules can't decide:

```rust
fn pick(a: &str, b: &str) -> &str { a }   // two inputs, no self: rule 2 and 3 don't apply
fn main() { println!("{}", pick("a", "b")); }
```

✓ `E0106`. Elision is a **convenience, not inference**: the compiler never looks at the function body to decide the signature. Because of that, the signature is the whole contract, and a body that does not match the contract is an error:

```rust
fn pick<'a>(x: &'a str, y: &str) -> &'a str { if y.is_empty() { x } else { y } }
fn main() { println!("{}", pick("a", "b")); }
```

✓ `error[E0621]: explicit lifetime required in the type of `y``: the body returns `y`, but the signature promised only `x`'s lifetime. Conversely, `fn f<'a>(x: &'a i32) -> &'static i32 { x }` ✓ fails with "lifetime may not live long enough": you cannot promise `'static` from a shorter borrow.

**Choosing which inputs to relate is API design.** In `fn first_if_nonempty<'a>(x: &'a str, y: &str) -> Option<&'a str>` (in the practical challenge) the result borrows only from `x`, so callers may drop `y` immediately. Tying everything to one `'a` (as `longest` does) is simpler but demands more of callers. Give the compiler the *weakest* contract that is true; Clippy's `needless_lifetimes` ✓ even flagged my first draft, which named a `'b` that did nothing.

---

## 4. Structs that hold references

A struct that stores a reference needs a lifetime parameter, to relate the struct's validity to the data it borrows from:

```rust
struct Excerpt { part: &str }   // E0106
fn main() {}
```

✓ `E0106`. With the parameter:

```rust
#[derive(Debug)]
struct Excerpt<'a> { part: &'a str }
impl<'a> Excerpt<'a> {
    fn part(&self) -> &str { self.part }
    fn announce(&self, msg: &str) -> &str { println!("note: {msg}"); self.part }
    fn get(&self) -> &'a str { self.part }
}
fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first = novel.split('.').next().expect("a sentence");
    let e = Excerpt { part: first };
    println!("{:?} {} {}", e, e.part(), e.announce("hi"));
    let long_lived: &str = { let tmp = Excerpt { part: first }; tmp.get() };
    println!("{long_lived}");
}
```

✓ Output `note: hi`, `Excerpt { part: "Call me Ishmael" } Call me Ishmael Call me Ishmael`, `Call me Ishmael`. Notes:

- `impl<'a> Excerpt<'a>`: you declare `'a` after `impl` and use it in the type, like generics (Lesson 19).
- `announce` returns a `&str` tied (rule 3) to `&self`, *not* to `msg`. That is what you usually want.
- **`get` vs `part`:** `part(&self) -> &str` returns a borrow of the *struct* (shorter than `'a`); `get(&self) -> &'a str` returns a borrow of the *original data* and can **outlive the struct**. ✓ `long_lived` is used after `tmp` is dropped, which only compiles because `get` is declared with `'a`. The wrong choice is a classic papercut: with elision on `get`, the output is tied to the struct:

```rust
struct S<'a> { r: &'a str }
impl<'a> S<'a> { fn get(&self) -> &str { self.r } }   // elided: output borrows from &self
fn main() {
    let owner = String::from("data");
    let out;
    {
        let s = S { r: &owner };
        out = s.get();
    }
    println!("{out}");     // E0597: `s` does not live long enough
}
```

✓ `E0597` on `s`. Fix: `fn get(&self) -> &'a str`. The same pattern is the heart of the debugging challenge (Section 14).

### A struct cannot outlive what it borrows

```rust
struct Excerpt<'a> { part: &'a str }
fn main() {
    let e;
    {
        let novel = String::from("Call me. Rest");
        e = Excerpt { part: novel.split('.').next().unwrap() };
    }
    println!("{}", e.part);   // E0597: novel dropped while e still used
}
```

✓ `E0597`. A struct with a lifetime parameter is "a borrow in a trench coat": everywhere the struct goes, the borrow goes.

### More than one lifetime

```rust
struct Pair<'a, 'b> { x: &'a str, y: &'b str }
impl<'a, 'b> Pair<'a, 'b> { fn x(&self) -> &'a str { self.x } }
fn main() {
    let a = String::from("a");
    let x_ref;
    {
        let b = String::from("b");
        let p = Pair { x: &a, y: &b };
        x_ref = p.x();
        println!("{}", p.y);
    }
    println!("{x_ref}");
}
```

✓ Output `b`, `a`. Because `x` and `y` have independent lifetimes, the result of `p.x()` may outlive `b`. Use separate lifetimes only when that independence is useful to callers; otherwise one `'a` is simpler (the compiler picks the overlap).

### Self-referential structs do not work this way

```rust
struct Bad { data: String, view: &str }   // E0106; and no lifetime can fix it
```

✓ `E0106`. Even with a lifetime parameter, you cannot make `view` borrow from `data` in the same struct in safe code: moving the struct would move `data` while `view` still points at the old address (Lesson 07's "a move is a memcpy"). The standard answers are to store **indices/ranges** instead of references, to split the owner and the view into two values, or to use `Pin` plus `unsafe` (Lesson 25); crates that automate it exist, with their own trade-offs.

### A destructor counts as a use

If a type with a lifetime parameter implements `Drop`, its destructor runs at scope end and may use the borrow, so the borrow stays live until then:

```rust
struct Guard<'a>(&'a mut Vec<i32>);
impl Drop for Guard<'_> { fn drop(&mut self) { self.0.push(99); } }
fn main() {
    let mut v = vec![1];
    let _g = Guard(&mut v);
    println!("{}", v.len());    // E0502: _g's borrow is live until its drop at end of main
}
```

✓ `E0502`. Delete the `impl Drop` and ✓ the same program compiles (only a dead-code warning), because without a destructor the borrow is dead after construction. Lesson 08's drop order is what decides *when* the "use" happens.

---

## 5. `'static`: two different things

`'static` is the lifetime of the whole program. It appears in two roles that beginners conflate:

**(a) A reference that is valid for the entire program: `&'static T`.** String literals (`"hello"` is `&'static str`, typically stored in the binary's read-only data), `static` items, and leaked allocations:

```rust
use std::thread;
static GREETING: &str = "hello";
fn main() {
    let lit: &'static str = "literal";
    println!("{lit} {GREETING}");
    let leaked: &'static mut Vec<i32> = Box::leak(Box::new(vec![1, 2]));
    leaked.push(3);
    println!("{leaked:?}");
    let owned = String::from("owned");   // String contains no borrows, so it is 'static as a *type*
    let h = thread::spawn(move || println!("{owned}"));
    h.join().unwrap();
}
```

✓ Output `literal hello`, `[1, 2, 3]`, `owned`. `Box::leak` (Lesson 08) turns an owned allocation into a `&'static mut` by promising never to free it.

**(b) A bound on a type: `T: 'static`.** It means "`T` contains **no non-`'static` references**." Owned types like `String`, `Vec<i32>`, `i32` all satisfy it, even though a `String` is dropped long before the program ends. `'static` as a *bound* says "does not borrow from anything that can disappear", not "lives forever".

The bound is why `thread::spawn` demands `'static`: a new thread may outlive the function that spawned it, so the closure can't hold borrows of locals.

```rust
use std::thread;
fn main() {
    let s = String::from("x");
    let r = &s;
    let h = thread::spawn(move || println!("{r}"));
    h.join().unwrap();
}
```

✓ `error[E0597]: `s` does not live long enough` with the note "argument requires that `s` is borrowed for `'static`". Fixes: move an owned value in (`move || println!("{s}")`), or use **scoped threads** (`thread::scope`, Lesson 27), which prove the threads end before the borrow does. The same bound explains

```rust
fn needs_static<T: 'static>(_t: T) {}
fn main() {
    let s = String::from("x");
    needs_static(s.clone());   // owned: fine
    let r = &s;
    needs_static(r);           // &String borrowed from a local: E0597
}
```

✓ `E0597` on `needs_static(r)`.

**Beware the "just add `'static`" reflex.** Error messages and blog posts sometimes suggest `'static`; it usually means "I now can't accept borrowed data", or "I must leak memory". Fix the design (owned data, or a lifetime parameter) before reaching for it. Also `&'static mut T` from `Box::leak` is a real memory leak for the program's lifetime: fine for global config built once at startup, a bug in a loop.

### Trait objects have a default lifetime too

`Box<dyn Trait>` in a signature means `Box<dyn Trait + 'static>` unless you say otherwise:

```rust
use std::fmt::Display;
fn boxed(x: &i32) -> Box<dyn Display> { Box::new(x) }   // wants x to be 'static
fn main() { let n = 7; println!("{}", boxed(&n)); }
```

✓ `error: lifetime may not live long enough` ("returning this value requires that `'1` must outlive `'static`"), and the compiler suggests the fix: `Box<dyn Display + '_>`. With that fix ✓ it compiles and prints `7 8` alongside an owned version. Similarly, a generic `T` boxed into `Box<dyn Display>` without a bound:

```rust
use std::fmt::Display;
fn boxed<T: Display>(t: T) -> Box<dyn Display> { Box::new(t) }
fn main() { println!("{}", boxed(5)); }
```

✓ `error[E0310]: the parameter type `T` may not live long enough` ("this `dyn Trait` has an implicit `'static` lifetime bound"). Fix: `T: Display + 'static` (accept only owned data) or return `Box<dyn Display + 'a>` with `T: 'a`.

---

## 6. Lifetime bounds: `T: 'a` and `'b: 'a`

- `'b: 'a` reads "`'b` outlives `'a`" (colon = "at least as long as").
- `T: 'a` reads "every reference inside `T` is valid for at least `'a`".

```rust
fn pick<'a, 'b: 'a>(x: &'a str, y: &'b str, first: bool) -> &'a str { if first { x } else { y } }
fn main() {
    let a = String::from("short-lived");
    let r;
    {
        let b = String::from("longer?");
        r = pick(&a, &b, false);
        println!("{r}");
    }
}
```

✓ Output `longer?`. Here `'b: 'a` lets the function return `y` where `'a` is expected, because `y` lives at least as long. Since Rust 2018 the compiler **infers** `T: 'a` for a struct field like `r: &'a T` (✓ `struct Holder<'a, T> { r: &'a T }` compiles with no explicit bound), so you rarely write it, except on functions and traits where it is not implied.

Why this works at all: references are **covariant** in their lifetime, meaning a longer-lived reference can be used where a shorter-lived one is expected ("a `&'long T` is a `&'short T`"). ✓ In a test, assigning an outer `&String` to a variable of an inner, shorter lifetime compiled. Covariance, contravariance, invariance (`&mut T` is invariant in `T`!) and higher-ranked bounds (`for<'a> Fn(&'a str)`) are the subject of Lesson 12; today it is enough to know that the compiler can *shorten* a lifetime when it must.

---

## 7. Lifetimes in `impl Trait`, iterators, and closures

A function returning an iterator or closure that borrows from an argument has to say so:

```rust
fn evens<'a>(xs: &'a [i32]) -> impl Iterator<Item = &'a i32> + 'a { xs.iter().filter(|x| **x % 2 == 0) }
fn evens2(xs: &[i32]) -> impl Iterator<Item = &i32> { xs.iter().filter(|x| **x % 2 == 0) }
fn main() {
    let v = vec![1, 2, 3, 4];
    println!("{:?} {:?}", evens(&v).collect::<Vec<_>>(), evens2(&v).collect::<Vec<_>>());
}
```

✓ Output `[2, 4] [2, 4]` for both. In the second, the lifetime appears in the `Item = &i32` bound, so the returned type captures it.

**Edition difference ✓.** A `-> impl Fn(i32) -> i32` that captures a borrowed argument, with no lifetime mentioned anywhere in the return type:

```rust
fn adder(n: &i32) -> impl Fn(i32) -> i32 { move |x| x + *n }
fn main() { let n = 3; let f = adder(&n); println!("{}", f(4)); }
```

compiles and prints `7` in **edition 2024**, but with `--edition 2021` I got ✓ `error[E0700]: hidden type for `impl Fn(i32) -> i32` captures lifetime that does not appear in bounds`. Edition 2024 makes return-position `impl Trait` capture **all** in-scope lifetimes by default; in older editions you had to write `+ '_` (or `+ 'a`). If you *don't* want to capture the borrow, 2024 lets you say so with **precise capturing**:

```rust
fn keep_only_n(n: &i32) -> impl Fn(i32) -> i32 + use<> { let k = *n; move |x| x + k }
fn main() { let f = { let n = 3; keep_only_n(&n) }; println!("{}", f(1)); }
```

✓ Output `4`. `use<>` means "captures no lifetimes", so `f` outlives `n`. (I believe `use<..>` stabilized in 1.82; I did not check the exact version, and it compiled on 1.97.)

---

## 8. The cases the borrow checker still rejects ("Problem case #3")

NLL (Lesson 10) ends borrows at last use, but a borrow that is **returned from a function** must last for the whole caller-provided lifetime, even on paths that did not return it:

```rust
use std::collections::HashMap;
fn get_or_insert(map: &mut HashMap<u32, String>, k: u32) -> &String {
    if let Some(v) = map.get(&k) { return v; }   // borrow flows into the return value...
    map.insert(k, format!("v{k}"));              // ...so it is still live here: E0502
    &map[&k]
}
fn main() {
    let mut m = HashMap::new();
    println!("{}", get_or_insert(&mut m, 1));
}
```

✓ `E0502`. It is *correct* code (on the `None` path nothing is borrowed), but today's checker can't see that. Idiomatic fix, ✓ compiles and prints `v1` twice when called twice:

```rust
fn get_or_insert(map: &mut HashMap<u32, String>, k: u32) -> &String {
    map.entry(k).or_insert_with(|| format!("v{k}"))
}
```

The in-progress **Polonius** analysis is designed to accept the first version; as far as I know it is not enabled on stable. Related, and not a bug:

```rust
fn first_mut(xs: &mut Vec<i32>) -> &mut i32 { &mut xs[0] }
fn main() {
    let mut v = vec![1, 2];
    let a = first_mut(&mut v);
    v.push(3);       // E0499: `a` borrows v through first_mut's signature
    *a += 1;
    println!("{v:?}");
}
```

✓ `E0499`. The signature says "output borrows from the `&mut` input", so as long as `a` lives, `v` is frozen. That is exactly the contract working.

---

## 9. Temporaries and lifetime extension

A temporary normally dies at the end of its statement. `let x = &temp();` is special: the compiler **extends** the temporary's life to the end of the enclosing block.

```rust
fn main() {
    let r = &String::from("ext");        // extended
    println!("{r}");
}
```

✓ Output `ext`. But extension applies only to specific syntactic forms. Calling a method on the temporary and keeping the result is not one of them:

```rust
fn main() {
    let s = String::from("a b").as_str();   // temporary dropped at end of statement
    println!("{s}");
}
```

✓ `error[E0716]: temporary value dropped while borrowed`. Fix: bind the owner first (`let owned = String::from("a b"); let s = owned.as_str();`). Edition 2024 also changed when temporaries in `if let` and a block's tail expression are dropped (Lesson 08 covered this).

---

## 10. Common mistakes and edge cases

- **Annotating everything `'a`.** One lifetime on every reference forces all of them to be related. Callers then hit confusing errors. Relate only what the function truly relates.
- **Expecting an annotation to fix E0515/E0597.** It can't. The data must live somewhere that outlives the reference: return an owned value, move the owner up a scope, or change the design.
- **Struct returning `&self`-bound data when it should return `&'a`** (Section 4). If callers will keep the result after dropping the struct, return `&'a`.
- **Storing `&str` in long-lived structures** (caches, services) that must outlive the input. Store `String`/`Arc<str>` (Lesson 23); borrow for short-lived processing.
- **`T: 'static` read as "lives forever".** It means "owns everything it holds".
- **`&'static` from `Box::leak` in a loop** leaks memory every iteration.
- **Fighting E0502 in "get or insert" shapes.** Use `entry`, split the check from the insertion, or restructure.
- **A lifetime parameter that appears only on a struct but never in a field** (E0392 *unused parameter*; I did not run it this session): use `PhantomData<&'a T>` (Lesson 22).
- **Thinking lifetimes affect the machine code.** They don't; the same function with different annotations produces identical code. They affect which programs compile.

---

## 11. Performance implications

- Lifetimes are free at run time: no code, no memory, no checks.
- What they enable is **zero-copy** design: a parser that returns `&str` slices of the input (like `Tokens` and `parse_kv` in the practical challenge) avoids one heap allocation and copy per token compared with returning `String`s. I did not benchmark it; whether it matters for your workload is a question for Lesson 41, but the allocation count is at least visibly smaller: `parse_kv` allocates only the result `Vec`.
- The price is that the output is tied to the input buffer. A tiny token that borrows from a 100 MB buffer keeps all 100 MB alive. When results must be stored long-term, copy the part you keep.
- `Cow<'a, str>` (Lesson 16) borrows when it can and allocates only when it must.

---

## 12. Security and reliability

- Borrowing from the input buffer eliminates a family of use-after-free and double-free bugs in safe Rust; it does not make parsing correct or safe against malicious input. Untrusted input still needs length limits, validation, and error handling instead of panics (`parse_kv` returns `Result`; there is no `unwrap` or indexing on input).
- A parser that holds `&str` slices into a request buffer must not outlive that buffer: the compiler enforces it. If you later need to hand the data to another task or thread, the `'static` bound will force you to make an owned copy; that copy is a good place to enforce size limits.
- `Box::leak` and `'static` references in servers can hide unbounded memory growth. Treat each leak as a deliberate, documented decision.
- With `unsafe` (Lesson 36), you take over the compiler's job: **unbounded lifetimes** produced by dereferencing raw pointers can silently become `'static` or any lifetime you like. Constrain them back to the true owner's lifetime in a safe wrapper's signature.

---

## 13. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Add lifetimes by hand to `fn longer_of_three(a: &str, b: &str, c: &str) -> &str`. How long is the result valid for?
2. **(Easy)** What are the elided lifetimes of `fn f(&self, a: &str) -> &str` and of `fn g(a: &str, b: &str) -> &str`? Which compiles?
3. **(Medium)** Make `fn pick<'a>(x: &'a str, y: &str) -> &'a str { if y.is_empty() { x } else { y } }` compile in two different ways, and say which changes the contract.
4. **(Medium)** Write `struct Words<'a>` that iterates over the words of a `&'a str`. Why must `Item = &'a str` and not `&str` (tied to `&mut self`)?
5. **(Medium)** `thread::spawn(|| println!("{}", v.len()))` with `v: Vec<i32>` local fails. Give two fixes and say how each works with lifetimes.
6. **(Hard)** Explain why `if let Some(v) = map.get(&k) { return v; } map.insert(..)` is rejected, and give a version that compiles.
7. **(Hard)** What is the difference between `&'static str` and `T: 'static`? Give a type that satisfies the bound but is not itself a reference.

<details markdown="1">
<summary>Answers</summary>

1. `fn longer_of_three<'a>(a: &'a str, b: &'a str, c: &'a str) -> &'a str`. The result is valid only while all three inputs are (the overlap, i.e. the shortest). Same reasoning ✓ as `longest`.
2. `f` is `fn f<'s, 'a>(&'s self, a: &'a str) -> &'s str` (rule 3) and compiles; `g` has two input lifetimes, no `self`, so ✓ `E0106`.
3. ✓ (E0621 for the original verified.) (a) Add `'a` to `y`: `y: &'a str`. This **changes the contract** (callers must now keep `y` alive as long as the result). (b) Stop returning `y` (return `x` on both paths), or return an owned `String` built from either. Option (a) is usually what was meant; (b) changes the function's behavior or cost. Decide which promise you want to make.
4. `struct Words<'a> { rest: &'a str }` with `impl<'a> Iterator for Words<'a> { type Item = &'a str; ... }` (the `Tokens` type in the practical challenge). If `Item` were tied to `&mut self` (as in a hand-written `next_word(&mut self) -> &str`) you would get E0499 as soon as you held two words at once (the debugging challenge). `Iterator::next` cannot express "borrows from self" (the trait has no lifetime link between `&mut self` and `Item`), so items must borrow from the **underlying data** with `'a`.
5. (a) `move` the vector into the closure: `thread::spawn(move || ...)`; the closure now owns it, so it is `'static`. (b) Use `thread::scope(|s| { s.spawn(|| ...); })`, which guarantees the threads finish before the scope returns, so borrows are allowed. (Neither version was run in this lesson; the E0597 failure with a borrowed `r` was.)
6. ✓ E0502 verified. The returned `v` forces the borrow from `map.get` to last for the whole function's output lifetime, including the path that falls through to `map.insert`. Use `map.entry(k).or_insert_with(...)` ✓.
7. `&'static str` is a reference that is valid for the entire program (a literal, a leaked allocation). `T: 'static` means `T` holds no non-`'static` borrows. `String`, `Vec<u8>`, `i32` ✓ satisfy it (the `thread::spawn` example moves a `String`) and are dropped long before program end.

</details>

---

## 14. Debugging challenge

A teammate wrote a word scanner and wants two words at once:

```rust
struct Scanner<'a> { rest: &'a str }
impl<'a> Scanner<'a> {
    fn new(s: &'a str) -> Self { Scanner { rest: s } }
    fn next_word(&mut self) -> &str {
        let t = self.rest.trim_start();
        let end = t.find(' ').unwrap_or(t.len());
        let (w, r) = t.split_at(end);
        self.rest = r;
        w
    }
}
fn main() {
    let mut sc = Scanner::new("alpha beta");
    let a = sc.next_word();
    let b = sc.next_word();
    println!("{a} {b}");
}
```

✓ It fails with `error[E0499]: cannot borrow `sc` as mutable more than once at a time`, pointing at the second `next_word` and at the later use of `a`. The function body is perfectly fine (the words really do borrow from the original string, not from the scanner). Why does the compiler complain, and what is the one-token change?

<details markdown="1">
<summary>Solution</summary>

**Elision rule 3** gave `next_word` the signature `fn next_word<'s>(&'s mut self) -> &'s str`: the returned word is tied to the *exclusive borrow of the scanner*. So `a` keeps `sc` mutably borrowed, and the second call is a second `&mut`. The signature *understates* what the body does: `w` is a slice of the original text, which has lifetime `'a`.

Fix: promise the true lifetime.

```rust
fn next_word(&mut self) -> &'a str {
```

✓ Prints `alpha beta`. The same reasoning explains `Tokens::next` returning `Option<&'a str>` in the practical challenge, and why `Iterator::next` items usually borrow from the underlying data (`'a`), not from the iterator.

</details>

---

## 15. Practical challenge: `lifelab`

Create a library crate `lifelab` (edition 2024) with:

1. `longest<'a>(a: &'a str, b: &'a str) -> &'a str` (first on ties) with a `compile_fail` doctest that uses the result after the shorter input is dropped.
2. `first_if_nonempty` returning `Option<&str>` borrowed from only the first argument (the second argument needs no named lifetime), and `split_first_word(&str) -> (&str, &str)`.
3. A zero-copy `Tokens<'a>` iterator over whitespace-separated words with `Item = &'a str`, and `remainder(&self) -> &'a str` that may outlive the iterator.
4. `parse_kv(&str) -> Result<Vec<(&str, &str)>, ParseError>` for `key=value` lines: skip blanks and `#` comments, trim, split at the **first** `=`, and return typed errors with 1-based line numbers (no panics, no `unwrap`/indexing on input). Implement `Display` and `Error` for `ParseError`.
5. `get_or_insert_with` using `HashMap::entry`, and `longest_line(&str) -> Option<Line<'_>>`.
6. Tests for each: ties, empty input, Unicode whitespace, "tokens outlive the tokenizer", error cases. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! lifelab: zero-copy parsing helpers that show how lifetimes tie outputs to inputs.

use std::collections::HashMap;
use std::fmt;

/// Returns the longer of two strings; on a tie, the first.
///
/// The output may borrow from either input, so it can only be used while *both* are alive:
///
/// ```compile_fail
/// let a = String::from("long string");
/// let result;
/// {
///     let b = String::from("xyz");
///     result = lifelab::longest(&a, &b);
/// }
/// println!("{result}");
/// ```
pub fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if b.len() > a.len() { b } else { a }
}

/// Returns (a view of) `x`; `y` is only inspected, so it needs no named lifetime
/// (Clippy's `needless_lifetimes` asks you to elide it).
pub fn first_if_nonempty<'a>(x: &'a str, y: &str) -> Option<&'a str> {
    if y.is_empty() { None } else { Some(x) }
}

/// Splits at the first space. Both halves borrow from `s` (elision: one input lifetime).
pub fn split_first_word(s: &str) -> (&str, &str) {
    match s.split_once(' ') {
        Some((head, tail)) => (head, tail),
        None => (s, ""),
    }
}

/// A zero-copy tokenizer: yields whitespace-separated words that borrow from the input.
#[derive(Debug, Clone)]
pub struct Tokens<'a> {
    rest: &'a str,
}

impl<'a> Tokens<'a> {
    pub fn new(input: &'a str) -> Self {
        Tokens { rest: input }
    }

    /// What is left to scan. Returns `&'a str`, not a borrow of `self`,
    /// so the result can outlive this `Tokens` value.
    pub fn remainder(&self) -> &'a str {
        self.rest
    }
}

impl<'a> Iterator for Tokens<'a> {
    type Item = &'a str;

    fn next(&mut self) -> Option<&'a str> {
        let trimmed = self.rest.trim_start();
        if trimmed.is_empty() {
            self.rest = trimmed;
            return None;
        }
        let end = trimmed.find(char::is_whitespace).unwrap_or(trimmed.len());
        let (word, rest) = trimmed.split_at(end);
        self.rest = rest;
        Some(word)
    }
}

/// Why a `key=value` line could not be parsed. Carries the 1-based line number.
#[derive(Debug, PartialEq, Eq)]
pub enum ParseError {
    MissingEquals { line: usize },
    EmptyKey { line: usize },
}

impl fmt::Display for ParseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ParseError::MissingEquals { line } => write!(f, "line {line}: expected `key=value`"),
            ParseError::EmptyKey { line } => write!(f, "line {line}: empty key"),
        }
    }
}

impl std::error::Error for ParseError {}

/// Parses `key=value` lines into borrowed pairs. Blank lines and `#` comments are skipped.
/// The returned slices borrow from `input`; nothing is allocated except the `Vec`.
pub fn parse_kv(input: &str) -> Result<Vec<(&str, &str)>, ParseError> {
    let mut out = Vec::new();
    for (i, raw) in input.lines().enumerate() {
        let line = raw.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        let Some((k, v)) = line.split_once('=') else {
            return Err(ParseError::MissingEquals { line: i + 1 });
        };
        let k = k.trim();
        if k.is_empty() {
            return Err(ParseError::EmptyKey { line: i + 1 });
        }
        out.push((k, v.trim()));
    }
    Ok(out)
}

/// Returns the value for `k`, inserting a default first if absent.
/// (The `if let Some(v) = map.get(..) { return v; } map.insert(..)` version is rejected
/// by today's borrow checker; `entry` is the idiomatic way around it.)
pub fn get_or_insert_with<'m>(
    map: &'m mut HashMap<String, String>,
    k: &str,
    make: impl FnOnce() -> String,
) -> &'m String {
    map.entry(k.to_owned()).or_insert_with(make)
}

/// A view into a larger text. Has two lifetimes only if it needs them; here one is enough.
#[derive(Debug, PartialEq, Eq)]
pub struct Line<'t> {
    pub number: usize,
    pub text: &'t str,
}

/// Returns the line containing the longest text, borrowing from `input`.
pub fn longest_line(input: &str) -> Option<Line<'_>> {
    let mut best: Option<Line<'_>> = None;
    for (i, text) in input.lines().enumerate() {
        if best.as_ref().is_none_or(|b| text.len() > b.text.len()) {
            best = Some(Line {
                number: i + 1,
                text,
            });
        }
    }
    best
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn longest_prefers_first_on_tie() {
        assert_eq!(longest("ab", "cd"), "ab");
        assert_eq!(longest("a", "bc"), "bc");
        assert_eq!(longest("", ""), "");
    }

    #[test]
    fn independent_lifetimes() {
        let kept = String::from("kept");
        let out;
        {
            let temp = String::from("temp");
            out = first_if_nonempty(&kept, &temp);
        }
        assert_eq!(out, Some("kept"));
        assert_eq!(first_if_nonempty("x", ""), None);
    }

    #[test]
    fn split_first_word_cases() {
        assert_eq!(split_first_word("hello big world"), ("hello", "big world"));
        assert_eq!(split_first_word("single"), ("single", ""));
        assert_eq!(split_first_word(""), ("", ""));
    }

    #[test]
    fn tokens_basic_and_whitespace() {
        let words: Vec<_> = Tokens::new("  a  bb\tccc\n d ").collect();
        assert_eq!(words, ["a", "bb", "ccc", "d"]);
        assert_eq!(Tokens::new("").count(), 0);
        assert_eq!(Tokens::new("   \n ").count(), 0);
    }

    #[test]
    fn tokens_outlive_the_tokenizer() {
        let text = String::from("one two three");
        let words: Vec<&str>;
        {
            let toks = Tokens::new(&text);
            words = toks.collect(); // `toks` is consumed and gone; the words borrow from `text`
        }
        assert_eq!(words, ["one", "two", "three"]);
    }

    #[test]
    fn remainder_outlives_tokens() {
        let text = String::from("a b c");
        let rest;
        {
            let mut t = Tokens::new(&text);
            t.next();
            rest = t.remainder();
        }
        assert_eq!(rest, " b c");
    }

    #[test]
    fn tokens_handles_unicode_whitespace() {
        // U+00A0 NO-BREAK SPACE is whitespace per `char::is_whitespace`.
        let words: Vec<_> = Tokens::new("é\u{a0}ü").collect();
        assert_eq!(words, ["é", "ü"]);
    }

    #[test]
    fn parse_kv_ok() {
        let text = "# comment\n\nname = lifelab\nmode=fast\n  x =  1 = 2 \n";
        assert_eq!(
            parse_kv(text).unwrap(),
            [("name", "lifelab"), ("mode", "fast"), ("x", "1 = 2")]
        );
        assert!(parse_kv("").unwrap().is_empty());
    }

    #[test]
    fn parse_kv_errors_never_panic() {
        assert_eq!(
            parse_kv("a=1\nbogus"),
            Err(ParseError::MissingEquals { line: 2 })
        );
        assert_eq!(parse_kv("=1"), Err(ParseError::EmptyKey { line: 1 }));
        assert_eq!(parse_kv("  = 1"), Err(ParseError::EmptyKey { line: 1 }));
        assert_eq!(
            ParseError::MissingEquals { line: 3 }.to_string(),
            "line 3: expected `key=value`"
        );
    }

    #[test]
    fn get_or_insert_with_runs_make_once() {
        let mut m = HashMap::new();
        let mut calls = 0;
        let v = get_or_insert_with(&mut m, "k", || {
            calls += 1;
            "v".to_string()
        });
        assert_eq!(v, "v");
        let v2 = get_or_insert_with(&mut m, "k", || unreachable!("already present"));
        assert_eq!(v2, "v");
        assert_eq!(calls, 1);
    }

    #[test]
    fn longest_line_cases() {
        assert_eq!(longest_line(""), None);
        assert_eq!(
            longest_line("ab\nabcd\nxy\nabcd"),
            Some(Line {
                number: 2,
                text: "abcd"
            })
        );
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 11 unit tests and 1 doctest passed.

**Design notes.**

- Clippy initially rejected my first draft of `first_if_nonempty<'a, 'b>`: the `'b` was needless. The fix is the version above.
- `Tokens::next` returns `Option<&'a str>`, and `remainder` returns `&'a str`, both tied to the **input**, not to the `Tokens` value. The tests `tokens_outlive_the_tokenizer` and `remainder_outlives_tokens` prove it: the tokenizer is gone, the words remain valid.
- `parse_kv` splits with `split_once('=')`, so values may contain `=` (the `x =  1 = 2` test). The slices it returns borrow from `input`; the only allocation is the `Vec`. Errors carry line numbers instead of panicking, since the input is untrusted.
- `get_or_insert_with` is the `entry` version of Section 8; the test uses `unreachable!` in the second closure to prove `make` isn't called when the key exists.
- `longest_line` uses `Option::is_none_or`, which compiled on the toolchain used here; I believe it is newer than Rust 1.80 but did not check the stabilization version, so on an older compiler use `map_or(true, ...)`.
- The doctest is `compile_fail`: it passes if the snippet fails to compile *for any reason*. I confirmed the identical shape fails with E0597 in Section 1, but did not check the code inside the doctest itself.

</details>

---

## 16. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is a lifetime?** A compile-time name for the region where a reference is valid. It has no run-time representation and annotations never extend a value's life.
2. **Why does `fn longest(a: &str, b: &str) -> &str` fail?** E0106: with two input references the compiler can't tell which the output borrows from, and it will not guess; add `<'a>` to relate them.
3. **What does `fn f<'a>(x: &'a str, y: &'a str) -> &'a str` promise?** The output is valid for a region contained in both inputs' lifetimes, i.e. the overlap of the two.
4. **State the elision rules.** (1) Each elided input lifetime is a distinct parameter; (2) if exactly one input lifetime, it is given to all elided outputs; (3) if there's `&self`/`&mut self`, its lifetime is given to elided outputs.
5. **Why can a function not return a reference to a local?** The local is dropped on return; the reference would dangle (E0515). Return an owned value or a borrow of an input.
6. **What does `struct S<'a> { r: &'a str }` mean for users of `S`?** Every `S` value is valid only while the borrowed data is; the compiler checks that the struct never outlives its source.
7. **What does `'static` mean as a reference and as a bound?** `&'static T` is valid for the whole program (literals, statics, leaked memory); `T: 'static` means `T` contains no non-static borrows, so owned types satisfy it.
8. **Why does `thread::spawn` require `'static`?** The thread may outlive the spawning function, so its closure must not borrow locals; use `move` with owned data or `thread::scope`.
9. **What is `Box<dyn Trait>`'s default lifetime bound?** `'static` in a signature without other context; write `Box<dyn Trait + '_>` or `+ 'a` to allow borrowed data.
10. **What is the difference between `fn get(&self) -> &T` and `fn get(&self) -> &'a T` on `impl<'a> S<'a>`?** The first borrows from the struct; the second borrows from the data the struct borrows, so it can outlive the struct.
11. **What is `'b: 'a`?** `'b` outlives `'a`. Used to let a longer-lived reference be returned where a shorter lifetime is promised.
12. **Why is `if let Some(v) = map.get(k) { return v; } map.insert(...)` rejected, and how do you fix it?** The returned borrow must last for the output lifetime, so it's live on the fall-through path too (a limitation NLL doesn't remove). Use `entry`, or restructure; Polonius targets this.
13. **What changed for `impl Trait` returns in edition 2024?** They capture all in-scope lifetimes by default; `use<..>` opts out precisely. In 2021 a closure capturing a borrowed argument without `+ '_` gave E0700.
14. **Do lifetimes affect run-time performance?** No: they're erased; they only decide which programs compile, and enable zero-copy designs.

</details>

---

## 17. Key takeaways

- A lifetime is a compile-time label for how long a reference is valid. Annotations relate inputs and outputs; they never change how long anything lives, and they leave no trace at run time.
- The error family: **E0106** (missing contract), **E0597** (value dropped while still borrowed), **E0515** (return of a reference to a local), **E0716** (temporary dropped), **E0310** (generic type may not outlive `'static`), **E0621** (body disagrees with signature).
- Elision handles most signatures; with several inputs and no `self`, you must say which input the output borrows from. Give the weakest true contract.
- Structs that hold references carry a lifetime parameter; return `&'a T` (not `&self`-bound) when the result should outlive the struct. Self-referential structs need a different design.
- `&'static T` (valid for the whole program) is not `T: 'static` (owns everything it holds). Don't reach for `'static` or `Box::leak` to silence an error.
- Idioms: return owned data, borrow from inputs, use `entry` for get-or-insert, scoped threads for borrowing across threads, and zero-copy parsing with `Item = &'a str`.
- Edition 2024: return-position `impl Trait` captures all lifetimes; use `use<..>` for exact control.

**Connections.** Lesson 10's "references conflict" now pairs with "references outlive"; Lesson 08's drop order explains E0597, `Drop` extending borrows, and `Box::leak`; Lesson 07's moves explain why self-referential structs are unsound. Next: Lesson 12 formalizes **variance** (why `&'a T` is covariant and `&mut T` invariant), **higher-ranked trait bounds** (`for<'a>`) and gives the borrow checker a deeper model. Lesson 16 uses `Cow`; Lesson 17 iterators with borrowed items; Lesson 18 closures capturing by reference; Lesson 23 `Arc` for sharing owned data; Lesson 25 `Pin`; Lesson 27 scoped threads; Lesson 30 async (futures that borrow across `.await`). See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 18. References

- *The Rust Programming Language*, ch. 10.3 "Validating References with Lifetimes": <https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html>
- *The Rust Reference*, "Lifetime elision" and "Trait object lifetime bounds": <https://doc.rust-lang.org/reference/lifetime-elision.html>, <https://doc.rust-lang.org/reference/types/trait-object.html#trait-object-lifetime-bounds>
- *The Rustonomicon*, "Lifetimes", "Limits of Lifetimes", "Unbounded Lifetimes": <https://doc.rust-lang.org/nomicon/lifetimes.html>, <https://doc.rust-lang.org/nomicon/unbounded-lifetimes.html>
- *The Rust Reference*, "Temporary lifetime extension": <https://doc.rust-lang.org/reference/destructors.html#temporary-lifetime-extension>
- *The Rust Edition Guide*, "RPIT lifetime capture rules" (edition 2024): <https://doc.rust-lang.org/edition-guide/rust-2024/rpit-lifetime-capture.html>
- Rust RFC 2094 "Non-lexical lifetimes" (includes the "Problem case #3" discussion): <https://rust-lang.github.io/rfcs/2094-nll.html>
- Rust error code index: E0106, E0310, E0515, E0597, E0621, E0700, E0716: <https://doc.rust-lang.org/error_codes/E0106.html>, <https://doc.rust-lang.org/error_codes/E0310.html>, <https://doc.rust-lang.org/error_codes/E0515.html>, <https://doc.rust-lang.org/error_codes/E0597.html>, <https://doc.rust-lang.org/error_codes/E0621.html>, <https://doc.rust-lang.org/error_codes/E0700.html>, <https://doc.rust-lang.org/error_codes/E0716.html>
- Rust standard library documentation: `std::thread::spawn` and `std::thread::scope`; `Box::leak`; `HashMap::entry`: <https://doc.rust-lang.org/std/thread/fn.spawn.html>, <https://doc.rust-lang.org/std/thread/fn.scope.html>, <https://doc.rust-lang.org/std/boxed/struct.Box.html#method.leak>, <https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.entry>
- Clippy lint `needless_lifetimes`: <https://rust-lang.github.io/rust-clippy/master/index.html#needless_lifetimes>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title. Unchecked from memory: the stabilization versions of `use<..>` and `Option::is_none_or`, and the exact current status of Polonius.
