---
layout: default
title: "Lesson 10 — References and borrowing: shared, mutable, aliasing, reborrowing"
---

# Lesson 10 — References and borrowing: shared, mutable, aliasing, reborrowing

**Module 4 · Borrowing and lifetimes** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs, error codes and assembly I actually ran; unmarked statements are documented behavior I did not run. Everything here is **stable Rust**; nothing needs nightly.

Builds on [Lesson 07](07_ownership_and_moves.md) (moves; why copying a `String` is a decision), [Lesson 08](08_stack_heap_drop.md) (fat pointers, `Drop`), and [Lesson 09](09_memory_layout.md) (a reference must be aligned and valid).

---

## 1. What a reference is

A reference `&T` or `&mut T` is, at run time, **just an address** (one word for a sized `T`; two words for slices, `str` and `dyn`, see Lesson 08). ✓ `size_of::<&String>()` and `size_of::<&mut String>()` are both 8 on this target. What differs is what the **compiler** knows and enforces about it:

- `&T` ("shared reference"): read access to a `T` that lives somewhere else. Any number of them may exist at once.
- `&mut T` ("mutable", better thought of as **exclusive** reference): read **and write** access, and *nobody else* may access that `T` while it lives.

```rust
fn len_of(s: &String) -> usize { s.len() }
fn append_bang(s: &mut String) { s.push('!'); }

fn main() {
    let mut s = String::from("hi");
    let n = len_of(&s);          // lend s, read only
    append_bang(&mut s);         // lend s, exclusively
    println!("{n} {s}");
    let r1 = &s;
    let r2 = &s;                 // many shared borrows: fine
    println!("{r1} {r2} {}", std::ptr::eq(r1, r2));
    let x = 10;
    let rx = &x;
    println!("{} {}", *rx + 1, rx + 1);   // explicit deref, or the operator impl for &i32
    let mut y = 1;
    let ry = &mut y;
    *ry += 5;                    // writing through a &mut needs an explicit *
    println!("{y}");
}
```

✓ Output: `2 hi!`, `hi! hi! true`, `11 11`, `6`.

**Why borrowing exists.** Lesson 07 gave you ownership: exactly one owner, a move transfers it. But passing a `String` to every function by value and getting it back is miserable. A borrow lets a function use a value **without** taking ownership, and the compiler proves the owner outlives the borrow (Lesson 11) and that access is safe (this lesson). Creating a reference is called *borrowing*; the reference ending is "giving it back". The notation `&` comes from C/C++ lineage; the *rules* are closer to a compile-time readers-writer lock (many readers or one writer) with zero run-time cost, which is the design bet of Rust's ownership system: pay in compile errors, not in locks or a garbage collector.

---

## 2. The one rule: aliasing XOR mutability

At any point in the program, for any piece of data, you may have **either**

- any number of shared references (`&T`), **or**
- exactly one exclusive reference (`&mut T`),

but **not both**, and never two `&mut`. (Compare: a mutex lets many threads wait; here the compiler rejects the program instead.)

### Why this rule, concretely

Mutation through one name while another name still points at the same data is the root of a family of bugs that other languages handle with discipline or runtime checks:

- **Iterator invalidation**: pushing to a vector while iterating it may reallocate and leave the iterator pointing at freed memory (C++ undefined behavior; Java's `ConcurrentModificationException` at run time).
- **Data races**: two threads, one writing (Lesson 27–29). `&mut`'s exclusivity is *the* reason safe Rust has no data races.
- **Surprising aliasing**: a function that writes to `a` and then reads `b` gets a different answer if `a` and `b` are the same location.

The compiler turns each of these into a compile-time error. The cost is that some correct programs are rejected (Section 8); the benefit is that every program that compiles is free of these bugs in safe code.

### The errors, with real codes ✓

Two `&mut` at once:

```rust
fn main() {
    let mut s = String::from("x");
    let a = &mut s;
    let b = &mut s;       // E0499
    a.push('1');
    b.push('2');
    println!("{s}");
}
```

✓ `error[E0499]: cannot borrow `s` as mutable more than once at a time`.

A shared borrow still alive when mutation starts:

```rust
fn main() {
    let mut s = String::from("x");
    let r = &s;
    s.push('!');          // E0502
    println!("{r}");
}
```

✓ `error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable`. (The same code covers the reverse: a `&mut` alive when a shared borrow starts.)

Assigning to, or moving out of, something that is borrowed:

```rust
fn main() {
    let mut n = 5;
    let r = &n;
    n = 6;                // E0506
    println!("{r} {n}");
}
```

✓ `error[E0506]: cannot assign to `n` because it is borrowed`.

```rust
fn main() {
    let s = String::from("x");
    let r = &s;
    let t = s;            // E0505
    println!("{r} {t}");
}
```

✓ `error[E0505]: cannot move out of `s` because it is borrowed`. A move may relocate the value (Lesson 07), which would leave `r` pointing at nothing, so it is forbidden while borrowed.

Mutating through something that isn't mutable:

```rust
fn add(v: &Vec<i32>) { v.push(1); }   // E0596: behind a `&` reference
fn main() { let mut v = vec![]; add(&v); println!("{v:?}"); }
```

✓ `error[E0596]: cannot borrow `*v` as mutable, as it is behind a `&` reference`. The compiler also warned that `let mut v` was unnecessary: nothing mutable ever happens to `v` itself.

```rust
fn main() { let s = String::from("a"); let r = &mut s; r.push('b'); println!("{r}"); }
```

✓ `error[E0596]: cannot borrow `s` as mutable, as it is not declared as mutable`. You can only take `&mut` of a place that is itself `mut` (or already reachable through a `&mut`). I did not trigger E0594 (assignment through a `&`) in this session; it is the same idea applied to `*r = value;` where `r: &T`.

**Rule of thumb for reading these errors:** find the two lines the compiler points to (where the first borrow was created, where the conflicting use happens), then find the **last use** of the first borrow. That last use is what keeps it alive (next section).

---

## 3. Non-lexical lifetimes (NLL): a borrow ends at its last use

Early Rust (before 2018) made borrows last until the end of the enclosing block, which rejected obviously fine code. Since Rust 2018 (NLL, stabilized in 1.31 for the 2018 edition, and used in all editions today), **a borrow lasts only as long as the reference is later used.**

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];
    println!("first = {first}");      // last use of the shared borrow
    v.push(4);                         // OK: the borrow has ended
    println!("{v:?}");
    let mut s = String::from("a");
    let r = &mut s;
    r.push('b');
    s.push('c');                       // OK: r is not used again
    println!("{s}");
}
```

✓ Output `first = 1`, `[1, 2, 3, 4]`, `abc`. Moving the `r.push('b')` **after** `s.push('c')` makes the same program fail with ✓ `E0499`, because now `r` is still live when `s` is used.

Important consequences:

- The scope of a reference *variable* (`{ ... }`) is not the same as the region where the borrow is **live**. NLL reasons about control flow: a borrow is live at a point if it may be used later along some path.
- A borrow that is only used in one `if` branch is not live in the other.
- **Types with `Drop` extend liveness.** If a struct holding a reference implements `Drop`, the destructor runs at scope end and counts as a use of the borrow (Lesson 08's drop order matters here). A plain `&T` has no destructor, so no extra use.
- NLL still cannot see through everything (Section 8.4: borrows returned from one branch and mutated in another are still rejected; the newer **Polonius** analysis aims to accept more, and as far as I know it is not what stable uses today).

---

## 4. Reborrowing

A `&mut T` is not `Copy`; passing it to a function **moves** it... except you can keep using it afterwards. Why? The compiler inserts an implicit **reborrow**: `bump(r)` really means `bump(&mut *r)`: a *new* exclusive borrow of the same place, valid for the call, after which `r` is usable again.

```rust
fn bump(x: &mut i32) { *x += 1; }
fn show(x: &i32) { println!("{x}"); }

fn main() {
    let mut n = 0;
    let r = &mut n;
    bump(r);                // implicit reborrow: r is still usable
    bump(r);
    show(&*r);              // explicit shared reborrow of a &mut
    let r2: &i32 = &*r;     // shared reborrow: r is "frozen" while r2 lives
    show(r2);
    *r += 10;               // OK: r2 no longer used
    println!("{n}");
    let mut v = vec![1, 2];
    v.push(v.len() as i32);  // two-phase borrow (Section 5)
    println!("{v:?}");
}
```

✓ Output `2`, `2`, `12`, `[1, 2, 2]`.

The model: while a reborrow is live, the **original** reference (and the place it points at) cannot be used in a conflicting way. When the reborrow ends, the parent regains full access. This is how a function can hand out `&mut` access to a part of its argument and keep going.

Where reborrowing is *not* automatic: when the target type is a generic parameter (`fn take<T>(x: T)`), passing `r` moves it, because there is no known "expected `&mut`" type to reborrow into. Write `take(&mut *r)` to be explicit. Also, **shared** references are `Copy`, so they're simply copied and need no reborrow.

---

## 5. Two-phase borrows

`v.push(v.len())` desugars to `Vec::push(&mut v, v.len())`: the `&mut v` is created first, and `v.len()` needs a shared borrow while it exists. That looks like E0502, but it compiles ✓ thanks to **two-phase borrows**: an autoref'd `&mut` for a method call starts as a *reservation* (acts like a shared borrow) and only becomes *active* at the call itself. Reads during the reservation are allowed.

This works only for the compiler-inserted autoref, not for a `&mut` you named yourself:

```rust
fn main() {
    let mut v = vec![1, 2];
    let r = &mut v;
    r.push(v.len() as i32);   // E0502: r is already an active &mut to v
    println!("{r:?}");
}
```

✓ `error[E0502]: cannot borrow `v` as immutable because it is also borrowed as mutable`. The explicit `let r = &mut v` activated the borrow immediately.

---

## 6. Disjoint borrows: what the compiler can and cannot split

The borrow checker tracks **places** (a path such as `p.a`, `*r`, `t.0`). Borrows of **different fields** are independent:

```rust
struct Pair { a: Vec<i32>, b: Vec<i32> }
fn main() {
    let mut p = Pair { a: vec![1], b: vec![2] };
    let ra = &mut p.a;
    let rb = &mut p.b;       // different field: fine
    ra.push(10);
    rb.push(20);
    println!("{:?} {:?}", p.a, p.b);
    let mut v = vec![1, 2, 3, 4];
    let (left, right) = v.split_at_mut(2);
    left[0] += right[1];
    right[0] *= 2;
    println!("{v:?}");
    v.swap(0, 3);
    println!("{v:?}");
    if let [first, .., last] = v.as_mut_slice() { std::mem::swap(first, last); }
    println!("{v:?}");
}
```

✓ Output `[1, 10] [2, 20]`, `[5, 2, 6, 4]`, `[4, 2, 6, 5]`, `[5, 2, 6, 4]`.

But **indexing is a method call** (`IndexMut::index_mut(&mut v, i)`), so the checker cannot know that `v[0]` and `v[1]` are different elements:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let a = &mut v[0];
    let b = &mut v[1];       // E0499: v borrowed twice
    std::mem::swap(a, b);
    println!("{v:?}");
}
```

✓ `error[E0499]`. Fixes, in order of preference: a method that does the job (`v.swap(0, 1)`), `split_at_mut`, a slice pattern (`if let [a, b, ..] = v.as_mut_slice()`), `iter_mut()`, or `split_first_mut`. `split_at_mut` is implemented with a small piece of `unsafe` plus a safe signature that *proves* the halves are disjoint; this is the standard pattern for hiding unsafe behind a safe API (Lesson 36).

The same applies to **methods**: `self.log.first()` as a direct field access borrows only `log`, while `self.first_log()` (a method taking `&self`) borrows **all of `self`**, because the signature is all the compiler sees at the call site. This is a major reason to prefer small structs and helper functions that take only the fields they need. Both behaviors are tested in the practical challenge (Section 13).

---

## 7. Auto-ref, auto-deref and deref coercion

You rarely write `&` and `*` for method calls: `s.len()` works whether `s` is `String`, `&String`, `&&String` or `Box<String>`. The compiler's method lookup tries the receiver type, then `&`, `&mut`, and repeatedly **dereferences** (`*`) to find a matching method. This is auto-ref/auto-deref and is why `.` replaces C's `->`.

**Deref coercion** is a separate, related rule: where a `&U` is expected and you have a `&T` with `T: Deref<Target = U>`, the compiler inserts `&*` for you. That is why `&String` works as `&str`, `&Vec<T>` as `&[T]`, and `&Box<T>` as `&T`:

```rust
fn takes_str(s: &str) -> usize { s.len() }
fn takes_slice(s: &[i32]) -> i32 { s.iter().sum() }
fn takes_i32(x: &i32) -> i32 { *x }
fn main() {
    let owned = String::from("hello");
    let boxed = Box::new(5);
    let v = vec![1, 2, 3];
    println!("{} {} {}", takes_str(&owned), takes_slice(&v), takes_i32(&boxed));
    let rr = &&&7;
    println!("{}", takes_i32(rr));      // &&&i32 coerces to &i32
    let words = vec![String::from("a"), String::from("bb")];
    let lens: Vec<usize> = words.iter().map(|w| w.len()).collect();
    println!("{lens:?}");
}
```

✓ Output `5 6 5`, `7`, `[1, 2]`.

**Design guidance:** take `&str` rather than `&String`, and `&[T]` rather than `&Vec<T>`, in function parameters. They accept strictly more callers (string literals, slices of arrays, `Box<str>`, ...) and say less about how the caller stores the data. Clippy warns about `&Vec<T>` and `&String` parameters (`ptr_arg`).

### Comparing references: `==` compares values, `ptr::eq` compares addresses

`==` on references compares **what they point to** (`&A == &B` uses `A: PartialEq<B>`); `std::ptr::eq` compares **addresses**:

```rust
fn main() {
    let a = String::from("same");
    let b = String::from("same");
    let ra = &a; let rb = &b; let ra2 = &a;
    println!("{} {} {}", ra == rb, std::ptr::eq(ra, rb), std::ptr::eq(ra, ra2));
    let x = 5;
    let y = 5;
    println!("{} {}", &x == &y, std::ptr::eq(&x, &y));
}
```

✓ Output `true false true` and `true false`. (Clippy may suggest simplifying `&x == &y`; it's written that way here only to make the contrast explicit.) Two distinct variables never share an address while both are alive, even if they hold the same value. Do not use pointer equality for logic, except where identity truly is the question (caches, graph nodes, `Rc::ptr_eq`).

---

## 8. What the checker rejects, and why

### 8.1 Iterator invalidation

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    for x in &v {                 // shared borrow lives for the whole loop
        if *x == 2 { v.push(99); }  // E0502
    }
    println!("{v:?}");
}
```

✓ `E0502`. This is the exact bug that causes use-after-free in C++: `push` may reallocate and the loop's pointer would dangle. Here it is a compile error. Fixes: collect the changes and apply after the loop; iterate over indices (`for i in 0..v.len()`, mindful of elements being added); iterate over a clone; or use methods built for it (`retain`, `drain`, `extend`).

### 8.2 Passing the same data as `&mut` and `&`

```rust
fn add_all(dst: &mut Vec<i32>, src: &Vec<i32>) { for x in src { dst.push(*x); } }
fn main() {
    let mut v = vec![1, 2];
    add_all(&mut v, &v);          // E0502
    println!("{v:?}");
}
```

✓ `E0502`. If this were allowed, `dst.push` could reallocate while `src` is being iterated: the iterator-invalidation bug again, just hidden behind a function boundary. The signature *promises* the two never alias. Fix: clone first (`let copy = v.clone(); add_all(&mut v, &copy)`), or design the function differently (`v.extend_from_within(..)` for this exact case).

### 8.3 Dangling references (previewed; full treatment in Lesson 11)

```rust
fn dangle() -> &String { let s = String::from("x"); &s }
```

✓ `error[E0106]: missing lifetime specifier`. And:

```rust
fn main() {
    let r;
    { let x = 5; r = &x; }       // E0597: x does not live long enough
    println!("{r}");
}
```

✓ `E0597`. Both are about a reference **outliving** what it points at, which is the subject of the next lesson. This lesson's errors are about two references *conflicting*; Lesson 11's are about a reference *outliving*.

### 8.4 Correct programs the checker still rejects

The checker is conservative: it is a sound but not complete approximation. Known categories (all documented limitations, not bugs): methods that return a borrow in one branch and mutate in another (the case Polonius targets), self-referential structs (Lesson 25 on `Pin`), and graphs with cycles. The escape hatches are, in order: restructure the code (indices instead of references, separate the phases, split the struct), use a type that moves the check to run time (`RefCell`, `Mutex`: Lesson 24, 29), and only then `unsafe` with a safe wrapper.

---

## 9. Why `&mut` is exclusive: what the optimizer gets ✓

Exclusivity is not only about safety; it also lets the compiler generate better code. As far as I know, Rust tells LLVM that a `&mut T` (and a shared `&T` that has no interior mutability) parameter is `noalias` for the duration of the call. Consider:

```rust
pub fn add_twice_refs(a: &mut i32, b: &i32) { *a += *b; *a += *b; }
pub unsafe fn add_twice_raw(a: *mut i32, b: *const i32) { unsafe { *a += *b; *a += *b; } }
```

✓ Compiled with `rustc -O --emit asm` on this target, the reference version loads `*b` **once** and does:

```
movl (%rsi), %eax     ; load *b once
addl %eax, %eax       ; b + b
addl %eax, (%rdi)     ; *a += 2*b
```

while the raw-pointer version reloads `*b` after each store to `*a`, because `a` and `b` might point at the same `i32`. The compiler is entitled to optimize the first version *because* you promised (via the types) they don't alias. This is a hint of Lesson 36's undefined-behavior rules: if unsafe code **creates** two `&mut` to the same place, it breaks the promise the optimizer relied on, and the result is not just "a data race", it is undefined behavior. I only checked this one function on this one compiler version, and I make no claim about speed in general; measure real code (Lesson 41).

---

## 10. Common mistakes and edge cases

- **Fighting the checker with `.clone()` everywhere.** A clone fixes the error by making an independent copy; sometimes right (small data, a true snapshot), often a sign the borrows should be restructured. Know *why* before cloning.
- **Holding a borrow across a call you wanted to mutate.** `let n = v.len();` (a `usize`, no borrow) vs `let n = &v;` (a borrow). Copy out the small value you need, then mutate.
- **`&mut` is not "mutable variable".** `let mut x` lets you rebind or mutate `x` itself; `&mut T` is a *capability* to mutate the pointee. `let r = &mut x;` does not need `r` to be `mut` unless you reassign `r` to point elsewhere.
- **Shared references can still mutate**: through interior mutability (`Cell`, `RefCell`, `Mutex`, atomics, Lessons 24, 28, 29). "Shared" is better read as "possibly shared", "immutable" as "may not be mutated *by the compiler's rules* unless the type opts out". The aliasing rule still holds; it is enforced at run time (or by hardware atomics) instead.
- **Reference to a temporary.** `let r = &String::from("x");` works: temporary lifetime extension keeps the temporary alive as long as `r`. `let r = String::from("x").as_str();` does not (the temporary is dropped at the end of the statement): that's an E0716, Lesson 11. I did not run this one in this session.
- **Reborrow vs move of `&mut` into a generic.** See Section 4.
- **Indexing in a loop while holding a reference to the container**, or two indexes into the same container, see Section 6.
- **Equality**: `==` on references compares pointees, `ptr::eq` addresses.

---

## 11. Performance implications

- A reference is a pointer: passing `&T` costs one word regardless of `T`'s size. Passing a large struct by value copies `size_of::<T>()` bytes (Lesson 07, 09) unless optimized. For small `Copy` types (integers, `char`, small tuples), pass by value: a pointer is no cheaper than the value and adds an indirection.
- References add no run-time checks. Borrow checking is entirely at compile time and leaves no trace in the binary.
- Exclusivity enables `noalias` optimization (Section 9). Aliasing through raw pointers gives that up.
- Reborrows and the `&*` forms are free; they generate no code.
- Needless `.clone()` to satisfy the checker costs allocations; Lesson 42 shows how to find them with profilers.

---

## 12. Security and reliability

- Borrow rules remove **memory-safety** bug classes in safe code (use-after-free through invalidated iterators, data races, aliased mutation). They do not make a program correct or secure: logic errors, integer overflow in release builds, injection, authorization flaws and denial-of-service are all still possible. Rust does not guarantee security.
- `unsafe` can break the aliasing rules (two `&mut` to the same place), and the compiler will not warn you. Keep unsafe blocks tiny with a documented `// SAFETY:` comment (Lesson 36), and run [Miri](https://github.com/rust-lang/miri) over tests that touch them (Lesson 38).
- In long-running services (RPC nodes, indexers, validators; Lessons 51–52) a function that takes `&mut self` on shared state is a design signal: you will need synchronization to share it across tasks. Prefer designs where ownership of mutable state is clear (one task owns it, others send messages).
- When a borrow error pushes you toward `unsafe` or `RefCell`, stop and restructure first: the error is often pointing at a real design problem (two parts of the code assuming they own the same thing).

---

## 13. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Which of these compile? Predict first, then check. (a) `let mut s = String::new(); let r = &s; s.push('a'); println!("{r}");` (b) `let mut s = String::new(); let r = &s; println!("{r}"); s.push('a');` (c) `let s = String::new(); let r = &mut s;`
2. **(Easy)** Write `fn swap_first_last(xs: &mut [i32])` using a slice pattern, with no indexing. What happens for slices with 0 or 1 elements?
3. **(Medium)** Write `fn sum(xs: &[i32]) -> i32` and `fn increment_all(xs: &mut [i32])`. Call them on one `Vec` in sequence (`increment_all(&mut v); let t = sum(&v);`). Then explain whether `let first = &v[0]; increment_all(&mut v); println!("{first}");` compiles.
4. **(Medium)** This fails with E0499: `let a = &mut v[0]; let b = &mut v[1]; *a += *b;`. Give three different compiling versions.
5. **(Medium)** Explain why `v.push(v.len())` compiles but `let r = &mut v; r.push(v.len());` does not.
6. **(Hard)** Why does this compile? `let mut s = String::from("a"); let r = &mut s; r.push('b'); s.push('c');` What single change to its last lines makes it fail, and why?
7. **(Hard)** Write a function `fn longest_run_mut(xs: &mut [i32]) -> &mut [i32]` returning the longest run of equal adjacent values, and explain how elision gave the output the lifetime of the input. (Lifetimes are Lesson 11; just state what the signature promises.)

<details markdown="1">
<summary>Answers</summary>

1. ✓ (a) fails with E0502; (b) compiles (NLL: `r` is dead before the `push`); (c) fails with E0596 (cannot borrow `s` as mutable, not declared `mut`).
2. ✓ Tested in `borrowlab` as `swap_ends`:
   ```rust
   pub fn swap_ends(xs: &mut [i32]) {
       if let [first, .., last] = xs {
           std::mem::swap(first, last);
       }
   }
   ```
   The pattern `[first, .., last]` needs at least two elements, so 0- and 1-element slices fall through and are left unchanged. No indexing, hence no panic path.
3. The sequential calls compile: each `&mut`/`&` borrow ends when its call returns. ✓ The second program fails with E0502, because `first` (a shared borrow into `v`) is still used after `increment_all` needs `&mut v`. (I tested the E0502 shape with an equivalent `push` example, Section 2, not this exact snippet.)
4. ✓ (E0499 for the original verified.) Compiling forms: `v.swap(0, 1)` when swapping; `let (a, b) = v.split_at_mut(1); a[0] += b[0];`; `if let [a, b, ..] = v.as_mut_slice() { *a += *b; }`. A fourth: copy the value out first: `let y = v[1]; v[0] += y;`.
5. Two-phase borrows (Section 5): the autoref'd `&mut v` for the method call is only a *reservation* until the call executes, so `v.len()` may read `v` meanwhile. With an explicit `let r = &mut v;` the `&mut` is active from that line, so `v.len()` conflicts. ✓ Both verified.
6. ✓ It compiles because `r` has no use after `r.push('b')`, so the exclusive borrow is dead by `s.push('c')`. Moving a use of `r` (for example `r.push('b')` after `s.push('c')`) makes it fail with E0499, because `r` is then live across the second use of `s`.
7. The signature promises that the returned slice borrows from `xs`: with exactly one input reference, elision gives the output the same lifetime, so `xs` stays exclusively borrowed as long as the result lives. Sketch: scan with indices to find `(start, len)` of the longest run, then `&mut xs[start..start + len]`. I did not write or test this one.

</details>

---

## 14. Debugging challenge

A teammate wants to remove all negative numbers in place:

```rust
fn drop_negatives(v: &mut Vec<i32>) {
    for (i, x) in v.iter().enumerate() {
        if *x < 0 { v.remove(i); }
    }
}
fn main() { let mut a = vec![1, -1, -2, 3]; drop_negatives(&mut a); println!("{a:?}"); }
```

✓ It fails with `error[E0502]: cannot borrow `*v` as mutable because it is also borrowed as immutable`. The `for` loop holds a shared borrow of `*v` (through the iterator) for its whole duration while `v.remove` needs `&mut`.

They "fix" it with an index loop:

```rust
fn drop_negatives_index(v: &mut Vec<i32>) {
    let mut i = 0;
    while i < v.len() {
        if v[i] < 0 { v.remove(i); }
        i += 1;
    }
}
```

This compiles, but is it correct? Test it on `[1, -1, -2, 3]`, find the bug, explain it, and give the idiomatic fix.

<details markdown="1">
<summary>Solution</summary>

✓ `drop_negatives_index` on `[1, -1, -2, 3]` returns `[1, -2, 3]`: **wrong**, a negative remains. After `remove(1)` the next element (`-2`) shifts into index 1, but `i` still advances to 2, so `-2` is skipped. The borrow checker only guarantees memory safety, not that the algorithm is right; this is also O(n²) because each `remove` shifts the tail.

Fix, idiomatic and O(n):

```rust
fn drop_negatives(v: &mut Vec<i32>) {
    v.retain(|x| *x >= 0);
}
```

✓ Returns `[1, 3]`. `retain` is implemented to do the removal in a single pass with the vector's internals, which is precisely what the borrow checker prevented you from improvising from outside. (If you must keep the manual loop, only advance `i` when you did **not** remove.)

</details>

---

## 15. Practical challenge: `borrowlab`

Create a library crate `borrowlab` (edition 2024) with:

1. `swap_ends(&mut [i32])` using a slice pattern, `clamp_all(&mut [i32], lo, hi)`, and `first_and_last_mut(&mut [i32]) -> Option<(&mut i32, &mut i32)>`.
2. `split_even_odd_mut(&mut [i32]) -> (Vec<&mut i32>, Vec<&mut i32>)` returning two disjoint groups of mutable references into the same slice.
3. `bump_pair(&mut u32, &mut u32)` with a `compile_fail` doctest showing that passing the same variable twice is rejected.
4. A `Counter { count, log }` whose method mutates `count` while holding a borrow into `log`, plus a method `first_log(&self)` and a `compile_fail` doctest showing it borrows all of the struct.
5. `push_len` (two-phase borrow), `drop_negatives` (`retain`), and a test showing `==` vs `ptr::eq`.
6. Tests for each, including edge cases (empty and one-element slices, an invalid clamp range panics). It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! borrowlab: small functions that exercise shared and mutable borrowing.

/// Swaps the first and last elements. Empty and one-element slices are left alone.
pub fn swap_ends(xs: &mut [i32]) {
    if let [first, .., last] = xs {
        std::mem::swap(first, last);
    }
}

/// Clamps every element into `lo..=hi`.
///
/// # Panics
/// Panics if `lo > hi` (the behaviour of `i32::clamp`).
pub fn clamp_all(xs: &mut [i32], lo: i32, hi: i32) {
    for x in xs.iter_mut() {
        *x = (*x).clamp(lo, hi);
    }
}

/// Splits mutable access to a slice into two disjoint groups: even values and odd values.
/// The returned references all borrow from `xs`, so `xs` is frozen while they live.
pub fn split_even_odd_mut(xs: &mut [i32]) -> (Vec<&mut i32>, Vec<&mut i32>) {
    let mut evens = Vec::new();
    let mut odds = Vec::new();
    for x in xs.iter_mut() {
        if *x % 2 == 0 {
            evens.push(x);
        } else {
            odds.push(x);
        }
    }
    (evens, odds)
}

/// Adds one to both counters.
///
/// Two `&mut` parameters are guaranteed by the compiler not to alias:
///
/// ```compile_fail
/// let mut n = 0u32;
/// borrowlab::bump_pair(&mut n, &mut n);
/// ```
pub fn bump_pair(a: &mut u32, b: &mut u32) {
    *a += 1;
    *b += 1;
}

/// Returns mutable references to the first and last elements, or `None` if fewer than two.
pub fn first_and_last_mut(xs: &mut [i32]) -> Option<(&mut i32, &mut i32)> {
    match xs {
        [first, .., last] => Some((first, last)),
        _ => None,
    }
}

/// A counter with a log. The two fields can be borrowed independently *inside* a method.
#[derive(Debug, Default)]
pub struct Counter {
    pub count: u32,
    pub log: Vec<String>,
}

impl Counter {
    /// Disjoint field borrows: `self.log` is read while `self.count` is mutated.
    pub fn bump_and_record(&mut self) {
        let last = self.log.last(); // shared borrow of one field
        let note = match last {
            Some(prev) => format!("after {prev}"),
            None => String::from("first"),
        };
        self.count += 1; // mutate another field while `last` was alive: fine
        self.log.push(note);
    }

    /// Borrowing through a method borrows the *whole* struct.
    ///
    /// ```compile_fail
    /// let mut c = borrowlab::Counter::default();
    /// c.log.push("a".to_string());
    /// let first = c.first_log();      // borrows all of `c`
    /// c.bump_and_record();            // needs &mut c while `first` is alive
    /// println!("{first:?}");
    /// ```
    pub fn first_log(&self) -> Option<&String> {
        self.log.first()
    }
}

/// Two-phase borrow: `v.push(v.len())` is accepted.
pub fn push_len(v: &mut Vec<usize>) {
    v.push(v.len());
}

/// Removes negatives in place without invalidating an iterator.
pub fn drop_negatives(v: &mut Vec<i32>) {
    v.retain(|x| *x >= 0);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn swap_ends_cases() {
        let mut a = [1, 2, 3];
        swap_ends(&mut a);
        assert_eq!(a, [3, 2, 1]);
        let mut one = [7];
        swap_ends(&mut one);
        assert_eq!(one, [7]);
        let mut empty: [i32; 0] = [];
        swap_ends(&mut empty);
    }

    #[test]
    fn clamp_all_works() {
        let mut a = [-5, 0, 5, 50];
        clamp_all(&mut a, 0, 10);
        assert_eq!(a, [0, 0, 5, 10]);
    }

    #[test]
    #[should_panic]
    fn clamp_all_bad_range_panics() {
        clamp_all(&mut [1], 5, 1);
    }

    #[test]
    fn even_odd_groups_write_through() {
        let mut a = [1, 2, 3, 4, 5];
        {
            let (evens, odds) = split_even_odd_mut(&mut a);
            for e in evens {
                *e *= 10;
            }
            for o in odds {
                *o = 0;
            }
        }
        assert_eq!(a, [0, 20, 0, 40, 0]);
    }

    #[test]
    fn bump_pair_distinct() {
        let (mut a, mut b) = (1, 10);
        bump_pair(&mut a, &mut b);
        assert_eq!((a, b), (2, 11));
    }

    #[test]
    fn first_last() {
        let mut a = [1, 2, 3];
        if let Some((f, l)) = first_and_last_mut(&mut a) {
            *f += 100;
            *l += 200;
        }
        assert_eq!(a, [101, 2, 203]);
        assert!(first_and_last_mut(&mut [1]).is_none());
        assert!(first_and_last_mut(&mut []).is_none());
    }

    #[test]
    fn counter_disjoint_fields() {
        let mut c = Counter::default();
        c.bump_and_record();
        c.bump_and_record();
        assert_eq!(c.count, 2);
        assert_eq!(c.log, ["first", "after first"]);
        assert_eq!(c.first_log().map(String::as_str), Some("first"));
    }

    #[test]
    fn push_len_two_phase() {
        let mut v = vec![];
        push_len(&mut v);
        push_len(&mut v);
        assert_eq!(v, [0, 1]);
    }

    #[test]
    fn ptr_eq_vs_eq() {
        let a = String::from("x");
        let b = String::from("x");
        assert!(a == b);
        assert!(!std::ptr::eq(&a, &b));
        assert!(std::ptr::eq(&a, &a));
    }

    #[test]
    fn drop_negatives_keeps_order() {
        let mut v = vec![1, -1, -2, 3];
        drop_negatives(&mut v);
        assert_eq!(v, [1, 3]);
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 10 unit tests and 2 doctests passed.

**Design notes.**

- `split_even_odd_mut` has one reference input, so elision ties both returned vectors' references to the input slice. The loop uses `iter_mut()`, which yields **disjoint** `&mut i32`s; the borrow checker accepts storing them because each comes from a different iteration, and the iterator itself is the unsafe-internally, safe-externally proof of disjointness.
- `bump_pair`'s doctest uses `compile_fail`: it passes if the snippet *fails to compile*, for any reason. I confirmed the `bump_pair(&mut n, &mut n)` shape errors with E0499 using the equivalent two-borrow example in Section 2, but I did not check the specific error code inside the doctest, so if you edit it, double-check it still fails for the *intended* reason. The same applies to the `first_log` doctest, where the intended error is E0502.
- `bump_and_record` shows disjoint field borrows inside one method: `self.log.last()` borrows only the `log` field, so `self.count += 1` is allowed while `last` is live (`note` is computed before `push` so no borrow remains).
- `first_log(&self)` returns a reference that borrows all of `*self`, which is why calling `bump_and_record(&mut self)` while it lives is an error.

</details>

---

## 16. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **State Rust's borrowing rules.** At any time, either any number of shared `&T` or exactly one `&mut T` to a given place, and every reference must be valid (point at live, aligned data) for as long as it is used.
2. **Why can't you have two `&mut` to the same value?** Mutation through one would be visible through the other without either knowing, causing iterator invalidation, data races and invalid assumptions (and breaking `noalias` optimizations).
3. **What is NLL?** Non-lexical lifetimes: a borrow lasts until its last use along control-flow paths, not until the end of the lexical block.
4. **What is a reborrow?** A new borrow derived from an existing reference (`&mut *r`, `&*r`). It is implicit when you pass a `&mut` to a function expecting `&mut`; the original is usable again after the reborrow ends.
5. **What are two-phase borrows?** An autoref'd `&mut` method receiver is first a reservation, allowing reads of the same place while evaluating arguments, then activates at the call. It is why `v.push(v.len())` compiles.
6. **Why do `v[0]` and `v[1]` as `&mut` conflict but `p.a` and `p.b` do not?** Fields are distinct places the checker can see; indexing is an `IndexMut` method call on the whole container, and the checker can't prove the indices differ. Use `split_at_mut`, `swap`, or slice patterns.
7. **What does `&mut self` vs `&self` say about a method?** `&self` promises not to mutate (barring interior mutability) and can be called while others read; `&mut self` needs exclusive access and calls conflict with other live borrows of the value.
8. **`&String` or `&str` for a parameter?** `&str`: deref coercion lets callers pass `&String`, literals, and slices, and the function does not care about ownership.
9. **Is a shared reference always immutable?** No: types with interior mutability (`Cell`, `RefCell`, `Mutex`, atomics) allow mutation through `&`; they enforce the aliasing discipline at run time or with atomics.
10. **How does `==` differ from `ptr::eq` on references?** `==` compares pointed-to values via `PartialEq`; `ptr::eq` compares addresses.
11. **Why can't you push to a vector while iterating over it?** `for x in &v` holds a shared borrow for the loop; `push` needs `&mut v` and could reallocate, invalidating the iterator; the compiler rejects it (E0502).
12. **What does it mean that `&mut T` is not `Copy`?** Copying it would produce two exclusive references to the same place. Passing it moves or reborrows it.
13. **What does the optimizer gain from exclusivity?** It may assume a `&mut` argument does not alias other pointers, so it can keep values in registers and avoid reloads (✓ shown in Section 9 for one small function).
14. **Does Rust's borrow checker prevent all bugs?** No: it prevents memory-safety and data-race bugs in safe code; logic errors (like the skipped element in the debugging challenge), overflow in release builds, and security flaws remain.

</details>

---

## 17. Key takeaways

- A reference is a pointer plus a compile-time permission: `&T` read-only and shareable, `&mut T` exclusive. Rule: many readers **or** one writer, never both.
- The checker reports conflicts as E0499 (two `&mut`), E0502 (`&mut` vs `&`), E0506 (assign while borrowed), E0505 (move while borrowed), and E0596 (borrowing mutably from something not mutable). Find the first borrow and its **last use**.
- NLL ends a borrow at its last use; reborrows (`&mut *r`) give temporary access without giving up the original; two-phase borrows allow `v.push(v.len())`.
- Disjoint **fields** can be borrowed separately, but indexing and `&self` method calls borrow the whole value; use `split_at_mut`, slice patterns, `swap`, or smaller helper functions.
- Prefer `&str`/`&[T]` parameters, pass small `Copy` values by value, and don't `.clone()` just to silence an error without understanding it.
- Exclusivity is both a safety rule and an optimization contract; unsafe code that breaks it is undefined behavior.
- The borrow checker proves memory safety, not correctness: the index-loop bug still compiled.

**Connections.** Lesson 07's moves become "a move is forbidden while borrowed" (E0505); Lesson 08's `Drop` explains why a destructor counts as a use of a borrow; Lesson 09's validity and alignment rules are what a reference must satisfy. Next: Lesson 11 (lifetimes) covers references that *outlive* their data (E0106, E0597, E0515), lifetime annotations and elision. Lesson 12 formalizes variance and higher-ranked bounds; Lesson 24 (interior mutability) and 29 (locks) move this same rule to run time; Lesson 36 shows how `unsafe` can lift it, and what you owe in return. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 18. References

- *The Rust Programming Language*, ch. 4.2 "References and Borrowing": <https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html>
- *The Rust Reference*, "Behavior considered undefined" (the aliasing and validity rules for references): <https://doc.rust-lang.org/reference/behavior-considered-undefined.html>
- *The Rustonomicon*, "Ownership and Lifetimes", "Aliasing", "Splitting borrows": <https://doc.rust-lang.org/nomicon/aliasing.html>, <https://doc.rust-lang.org/nomicon/borrow-splitting.html>
- *The Rust Reference*, "Method-call expressions" (auto-ref/auto-deref) and "Type coercions" (deref coercion): <https://doc.rust-lang.org/reference/expressions/method-call-expr.html>, <https://doc.rust-lang.org/reference/type-coercions.html>
- Rust RFC 2094, "Non-lexical lifetimes": <https://rust-lang.github.io/rfcs/2094-nll.html>
- Rust RFC 2025, "Nested method calls" (two-phase borrows): <https://rust-lang.github.io/rfcs/2025-nested-method-calls.html>
- Rust error code index: E0499, E0502, E0505, E0506, E0596: <https://doc.rust-lang.org/error_codes/E0499.html>, <https://doc.rust-lang.org/error_codes/E0502.html>, <https://doc.rust-lang.org/error_codes/E0505.html>, <https://doc.rust-lang.org/error_codes/E0506.html>, <https://doc.rust-lang.org/error_codes/E0596.html>
- Rust standard library documentation: `slice::split_at_mut`, `Vec::retain`, `std::ptr::eq`: <https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut>, <https://doc.rust-lang.org/std/vec/struct.Vec.html#method.retain>, <https://doc.rust-lang.org/std/ptr/fn.eq.html>
- Miri, an interpreter that detects undefined behavior including aliasing violations: <https://github.com/rust-lang/miri>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title. The NLL stabilization version (1.31, 2018 edition) and the two-phase-borrow RFC number are from memory and unchecked.
