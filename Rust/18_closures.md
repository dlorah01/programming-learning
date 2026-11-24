---
title: "Lesson 18 — Closures: Fn, FnMut, FnOnce"
---

# Lesson 18 — Closures: `Fn`, `FnMut`, `FnOnce`

**Module 6 · Collections and functional programming** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**.

Builds on [Lesson 17](17_iterators.md) (every adapter takes a closure), [Lesson 10](10_references_and_borrowing.md) (borrow rules), [Lesson 07](07_ownership_and_moves.md) (moves), [Lesson 11](11_lifetimes.md) and [Lesson 12](12_variance_hrtb_borrowck.md) (lifetimes, `for<'a>`), and [Lesson 09](09_memory_layout.md) (sizes, zero-sized types).

---

## 1. What a closure is

A **closure** is an anonymous function that can use variables from the scope where it was written. The syntax is `|parameters| body`:

```rust
fn add_one(x: i32) -> i32 { x + 1 }
fn main() {
    let a = |x: i32| -> i32 { x + 1 };
    let b = |x| x + 1;
    let c = |x: i32| x + 1;
    let d = || 42;
    let e = |a: i32, b: i32| a * b;
    println!("{} {} {} {} {} {}", add_one(1), a(1), b(1), c(1), d(), e(3, 4));
    let s = |x| x;
    println!("{}", s(String::from("hi")));
}
```

✓ Output `2 2 2 2 42 12` and `hi`. Parameter and return types are usually **inferred** from how the closure is used, so you can omit them (unlike `fn` items, whose signatures are always explicit). But the inferred types are fixed by the *first* use; a closure is not generic:

```rust
fn main() {
    let id = |x| x;
    println!("{}", id(5));
    println!("{}", id("five"));
}
```

✓ `error[E0308]: mismatched types` on the second call (expected integer, found `&str`). To be generic over types you need a function item (`fn id<T>(x: T) -> T`).

What the compiler does with `|x| x + n` is worth knowing, because it explains every rule below: it creates a **fresh, unnamed struct type** for this one closure expression, with one field for each captured variable, and implements a call operation on it. Two closures never have the same type, even if their text is identical (Section 8 shows the consequence). A closure with no captures is a zero-sized struct (✓ size 0, Section 7).

---

## 2. Capturing: borrow, mutably borrow, or move

A closure may use variables from its surroundings. The compiler looks at **how the body uses each variable** and captures it in the least demanding way that works:

| The body... | Captured by | Example |
|---|---|---|
| only reads it | shared reference `&T` | `\|\| println!("{name}")` |
| mutates it | mutable reference `&mut T` | `\|\| count += 1` |
| moves it out (consumes it) | by value (ownership moves into the closure) | `\|\| drop(s)` |

```rust
fn main() {
    let name = String::from("ana");
    let show = || println!("hi {name}");          // captures by shared reference
    show(); show();
    println!("still usable: {name}");

    let mut count = 0;
    let mut inc = || { count += 1; count };        // captures by mutable reference
    inc(); inc();
    println!("{}", count);                          // fine: inc's borrow ended after its last use

    let s = String::from("moved");
    let consume = || drop(s);                       // captures by value (needs ownership)
    consume();
}
```

✓ Output `hi ana`, `hi ana`, `still usable: ana`, `2`. These are the borrow rules of Lesson 10 applied to the captured variables, including NLL (the mutable borrow held by `inc` ends after its last call, so `count` is usable again). While a capturing closure is alive, its borrows are alive:

```rust
fn main() {
    let mut count = 0;
    let mut inc = || count += 1;
    println!("{count}");
    inc();
}
```

✓ `error[E0502]: cannot borrow `count` as immutable because it is also borrowed as mutable`: `inc` holds `&mut count` until its last use, which comes after the `println!`.

### 2.1 `move`: force by-value capture

Prefixing `move` makes the closure take **ownership** of every variable it uses (for `Copy` types, a copy):

```rust
fn main() {
    let s = String::from("hello");
    let f = move || println!("{s}");
    f();
    println!("{s}");
}
```

✓ `error[E0382]: borrow of moved value: `s`` — the `String` now lives inside `f`. With `Copy` values `move` copies, so the original is unaffected, and the closure keeps its own state ✓:

```rust
fn main() {
    let mut n = 10;
    let mut f = move || { n += 1; n };
    println!("{} {}", f(), f());
    println!("{n}");
    let v = vec![1, 2, 3];
    let g = move || v.len();
    println!("{}", g());
    println!("{}", g());
}
```

✓ Output `11 12`, `10` (the outer `n` is untouched: the closure mutated its own copy), then `3` and `3`. Two things to notice: `move` changes **how** variables are captured, not **what the closure can do** (the `move` closure `g` can still be called repeatedly because its body only reads `v`); and a `move` closure that mutates its captured state keeps that state between calls (this is how counters and generators work, Section 6).

When do you need `move`? Whenever the closure may **outlive** the variables it borrows: returning it from a function, storing it in a struct, or sending it to another thread.

```rust
use std::thread;
fn main() {
    let v = vec![1, 2, 3];
    let h = thread::spawn(|| println!("{v:?}"));
    h.join().unwrap();
}
```

✓ `error[E0373]: closure may outlive the current function, but it borrows `v`, which is owned by the current function`, with the help "to force the closure to take ownership of `v`, use the `move` keyword". The new thread may run longer than `main`'s stack frame, so a borrow would dangle. With `move` ✓:

```rust
use std::thread;
fn main() {
    let v = vec![1, 2, 3];
    let h = thread::spawn(move || v.iter().sum::<i32>());
    println!("{}", h.join().unwrap());
}
```

✓ Output `6`. (Lesson 27 covers threads. If the thread is guaranteed to finish before the data goes away, `std::thread::scope` lets the closure *borrow* instead ✓: a scoped thread summed `data` without `move`, and `data.push(4)` worked afterwards, printing `6 [1, 2, 3, 4]`.)

### 2.2 Edition 2021 and later: precise captures

Since the 2021 edition a closure captures the **individual fields** it uses, not the whole struct or tuple (RFC 2229, from memory), so it can coexist with other uses of the other fields ✓:

```rust
struct P { name: String, hits: u32 }
fn main() {
    let mut p = P { name: String::from("a"), hits: 0 };
    let name_ref = &p.name;
    let mut bump = || p.hits += 1;      // captures only p.hits
    bump(); bump();
    println!("{name_ref} {}", p.hits);
    let t = (String::from("x"), String::from("y"));
    let c = move || t.0.len();          // moves only t.0
    println!("{} {}", c(), t.1);
}
```

✓ Output `a 2` and `1 y`. In the 2018 edition the first closure would have borrowed all of `p` and conflicted with `name_ref` (from memory; I only ran this in the 2024 edition).

### 2.3 Captured values are dropped with the closure

A closure owns what it moved in, and drops it when the closure is dropped (Lesson 08) ✓:

```rust
struct D(&'static str);
impl Drop for D { fn drop(&mut self) { println!("drop {}", self.0); } }
fn main() {
    let a = D("a");
    let b = D("b");
    let f = move || { let _x = &a; };
    f();
    println!("after call");
    drop(f);
    println!("after drop(f)");
    let _keep = b;
}
```

✓ Output `after call`, `drop a`, `after drop(f)`, `drop b`: calling the closure did not drop `a`; dropping the closure did. A long-lived closure therefore keeps its captured data alive (and any lock guard or file handle it captured).

---

## 3. The three traits: `Fn`, `FnMut`, `FnOnce`

How a closure *uses* its captures decides which of three traits it implements. They are about **how many times and with what access the closure can be called**:

| Trait | Call takes | Can be called | The body... |
|---|---|---|---|
| `FnOnce` | `self` (by value) | at least once, maybe only once | may move captured values out (consume them) |
| `FnMut` | `&mut self` | many times, needs exclusive access | may mutate captured values, not move them out |
| `Fn` | `&self` | many times, even concurrently with shared access | only reads captured values |

They form a hierarchy: **every `Fn` is also `FnMut`, and every `FnMut` is also `FnOnce`** (`Fn: FnMut: FnOnce`). So a function that asks for `FnOnce` accepts *any* closure (the most permissive for the caller), while one that asks for `Fn` is the most demanding of the closure but the most useful for the callee (it can call it as often as it likes, even from several threads). The trait is chosen by the compiler from the body; `move` has no effect on it ✓:

```rust
fn run_fn<F: Fn() -> usize>(f: F) -> usize { f() + f() }
fn run_mut<F: FnMut() -> usize>(mut f: F) -> usize { f() + f() }
fn run_once<F: FnOnce() -> usize>(f: F) -> usize { f() }
fn main() {
    let v = vec![1, 2, 3];
    let reads = || v.len();                 // Fn
    println!("{}", run_fn(reads));
    let mut calls = 0;
    let counts = || { calls += 1; calls };  // FnMut
    println!("{}", run_mut(counts));
    let owned = vec![9, 9];
    let consumes = move || owned.into_iter().count(); // FnOnce
    println!("{}", run_once(consumes));
    println!("{}", run_once(|| v.len()));   // an Fn closure is also FnOnce
    println!("{}", run_mut(|| v.len()));    // ... and FnMut
}
```

✓ Output `6`, `3` (1 + 2), `2`, `3`, `6`.

### 3.1 The errors you will see

**Calling a `FnOnce` closure twice**: it consumed something on the first call, so the closure value itself was moved by the call ✓:

```rust
fn main() {
    let s = String::from("x");
    let f = move || drop(s);
    f();
    f();
}
```

✓ `error[E0382]: use of moved value: `f``, with the note that `f` "cannot be invoked more than once because it moves the variable `s` out of its environment".

**Passing a `FnOnce` closure where `Fn` is required.** Which error you get depends on whether the closure is written at the call site or bound first ✓:

```rust
fn call_twice<F: Fn()>(f: F) { f(); f(); }
fn main() {
    let s = String::from("x");
    call_twice(move || drop(s));
}
```

✓ `error[E0507]: cannot move out of `s`, a captured variable in an `Fn` closure` (the compiler already knew the closure had to be `Fn` and complains about the body). Bind the closure to a variable first and the error moves to the call ✓:

```rust
fn call_twice<F: Fn()>(f: F) { f(); f(); }
fn main() {
    let s = String::from("x");
    let once = move || drop(s);
    call_twice(once);
}
```

✓ `error[E0525]: expected a closure that implements the `Fn` trait, but this closure only implements `FnOnce``. Fix: ask for `FnOnce` if you call it once, or change the body so it does not consume the capture (clone it, or borrow it).

**Mutating inside an `Fn` closure** ✓:

```rust
fn run_fn<F: Fn()>(f: F) { f() }
fn main() {
    let mut n = 0;
    run_fn(|| n += 1);
}
```

✓ `error[E0594]: cannot assign to `n`, as it is a captured variable in a `Fn` closure`. Ask for `FnMut` (and take the parameter as `mut f: F`) if the callee may mutate, or use interior mutability (`Cell`, Lesson 24) to mutate through `&self`.

**Calling an `FnMut` closure through a non-`mut` binding.** Calling it needs `&mut self`:

```rust
fn main() {
    let mut n = 0;
    let inc = || n += 1;
    inc();
}
```

✓ `error[E0596]: cannot borrow `inc` as mutable, as it is not declared as mutable`. Write `let mut inc`. The same applies to a `Box<dyn FnMut()>` (✓ `E0596` for `let f: Box<dyn FnMut() -> i32> = ...; f()`) and to a parameter: `fn run<F: FnMut()>(mut f: F)`.

### 3.2 `&mut F` is also `FnMut`

Because `&mut F` implements `FnMut` whenever `F: FnMut`, you can lend a closure to a function and keep using it afterwards ✓:

```rust
fn run_twice(f: &mut impl FnMut()) { f(); f(); }
fn run_dyn(f: &mut dyn FnMut()) { f(); }
fn consume<F: FnMut()>(mut f: F) { f(); }
fn main() {
    let mut n = 0;
    let mut inc = || n += 1;
    run_twice(&mut inc);
    run_dyn(&mut inc);
    consume(&mut inc);       // passes a borrow, so `inc` is still ours
    inc();
    println!("{n}");
}
```

✓ Output `5` (2 + 1 + 1 + 1 calls). This is the same trick as `by_ref()` for iterators (Lesson 17).

---

## 4. Closures as arguments

There are three ways to accept a closure, with different costs:

```rust
fn apply_generic<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 { f(x) }   // monomorphized
fn apply_impl(f: impl Fn(i32) -> i32, x: i32) -> i32 { f(x) }       // same thing, shorter
fn apply_dyn(f: &dyn Fn(i32) -> i32, x: i32) -> i32 { f(x) }        // dynamic dispatch
fn apply_ptr(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }             // function pointer
```

- **Generic / `impl Fn`**: the compiler generates a copy of the function for each closure type (monomorphization), so the call can be inlined. Fastest; code size grows with the number of distinct closures.
- **`&dyn Fn` / `Box<dyn Fn>`**: one copy of the function, and each call goes through a vtable (an indirect call); closures of different types can be mixed. Lesson 21 covers trait objects.
- **`fn(i32) -> i32`**: a plain function pointer (8 bytes). It accepts `fn` items and **non-capturing** closures only ✓:

```rust
fn double(x: i32) -> i32 { x * 2 }
fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn apply_gen<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 { f(x) }
fn main() {
    println!("{}", apply(double, 4));
    println!("{}", apply(|x| x + 1, 4));          // non-capturing closure coerces to fn pointer
    println!("{}", apply_gen(double, 4));          // fn items implement Fn
    println!("{:?}", ["1", "2"].iter().map(|s| s.parse::<i32>()).collect::<Result<Vec<_>, _>>());
    println!("{:?}", [Some(1), None].iter().map(Option::is_some).collect::<Vec<_>>());
    println!("{:?}", ["a", "b"].iter().map(ToString::to_string).collect::<Vec<_>>());
    let k = 3;
    println!("{}", apply(move |x| x + k, 1));
}
```

✓ This compiles the lines up to the last `apply`, which fails: `error[E0308]: mismatched types` with "expected fn pointer, found closure" and the note "closures can only be coerced to `fn` types if they do not capture any variables". Without that last line ✓ the output is `8`, `5`, `8`, `Ok([1, 2])`, `[true, false]`, `["a", "b"]`. Note that **functions and method paths can be used wherever a closure is expected** (`map(Option::is_some)`, `map(ToString::to_string)`, `.map(str::len)`), which is often clearer than `|x| x.is_some()`. Clippy's `redundant_closure` lint flags `|x| f(x)` (it fired in this lesson's practical crate: a test had `|a| Err(a)` and clippy asked for plain `Err`).

Prefer `impl Fn`/generics for hot, local code and `Box<dyn Fn>` when you need to **store** closures of different types together or hide the type.

### 4.1 `where` clauses and higher-ranked bounds

Bounds on closures that take or return references use the `for<'a>` form from Lesson 12; the compiler writes it for you in the `Fn(&str) -> &str` sugar, and you can spell it out ✓:

```rust
fn first_word(s: &str) -> &str { s.split(' ').next().unwrap_or("") }
fn apply<F>(f: F) -> usize where F: for<'a> Fn(&'a str) -> &'a str { f("hello world").len() }
fn main() {
    println!("{}", apply(first_word));
    println!("{}", apply(|s| s.split(' ').next().unwrap_or("")));
}
```

✓ Output `5` and `5`. A function item and a closure both satisfy the bound.

---

## 5. Closures stored in structs

To keep a closure in a struct you either make the struct generic over it, or box it.

```rust
struct Button<F: Fn(&str) -> String> { label: String, on_click: F }
impl<F: Fn(&str) -> String> Button<F> {
    fn click(&self) -> String { (self.on_click)(&self.label) }
}
struct Dyn { handlers: Vec<Box<dyn FnMut(i32)>> }
impl Dyn {
    fn fire(&mut self, x: i32) { for h in &mut self.handlers { h(x); } }
}
fn main() {
    let b = Button { label: "ok".into(), on_click: |l| format!("clicked {l}") };
    println!("{}", b.click());
    let total = std::rc::Rc::new(std::cell::Cell::new(0));
    let t2 = total.clone();
    let mut d = Dyn { handlers: vec![] };
    d.handlers.push(Box::new(move |x| t2.set(t2.get() + x)));
    d.handlers.push(Box::new(|x| println!("got {x}")));
    d.fire(3); d.fire(4);
    println!("{}", total.get());
}
```

✓ Output `clicked ok`, `got 3`, `got 4`, `7`. Notes:

- Calling a closure **stored in a field** needs parentheses: `(self.on_click)(...)`; `self.on_click(...)` would look for a *method* named `on_click`.
- The generic form has one closure type per `Button` (zero overhead, but all buttons in a `Vec` would need the same `F`). The boxed form lets different closure types live together (`Vec<Box<dyn FnMut(i32)>>`) at the cost of an allocation and an indirect call.
- `Box<dyn FnMut(i32)>` with no lifetime written means `Box<dyn FnMut(i32) + 'static>`: the closure cannot borrow local variables. To store a borrowing closure write the lifetime (`Box<dyn Fn() -> usize + 'a>`). Forgetting this gives an error with no code ✓:

```rust
fn make<'a>(s: &'a str) -> Box<dyn Fn() -> usize + 'a> { Box::new(move || s.len()) }
fn make_static(s: &str) -> Box<dyn Fn() -> usize> { Box::new(move || s.len()) }
fn main() { println!("{}", make("abc")()); let _ = make_static; }
```

✓ `error: lifetime may not live long enough` on `make_static` (the boxed closure borrows `s` but the return type promises `'static`). `make` compiles.

- Sharing state between a stored closure and the code that created it needs shared ownership plus interior mutability (`Rc<Cell<_>>` / `Rc<RefCell<_>>`, Lessons 23 and 24) ✓ (`total` above). Two closures sharing one log: three `Rc` strong counts (the log and two clones) and pushes from both appeared in order ✓ (`["a:x", "b:y", "a:z"] 3`). Reference cycles between a closure and what owns it leak memory (Lesson 25).
- A one-shot callback is `Option<Box<dyn FnOnce() -> T>>`: you `take()` it to call it, and it can only run once ✓:

```rust
struct Task { job: Option<Box<dyn FnOnce() -> String>> }
impl Task {
    fn run(&mut self) -> Option<String> { self.job.take().map(|j| j()) }
}
fn main() {
    let s = String::from("payload");
    let mut t = Task { job: Some(Box::new(move || s + "!")) };
    println!("{:?} {:?}", t.run(), t.run());
    let b: Box<dyn FnOnce() -> usize> = Box::new(|| 7);
    println!("{}", b());
}
```

✓ Output `Some("payload!") None` and `7`. Calling a `Box<dyn FnOnce>` consumes the box.

---

## 6. Returning closures

A closure's type has no name, so a function returns it as `impl Fn…` (one concrete type, no allocation) or `Box<dyn Fn…>` (allocated, can return different closures):

```rust
fn adder(n: i32) -> impl Fn(i32) -> i32 { move |x| x + n }
fn boxed(kind: &str) -> Box<dyn Fn(i32) -> i32> {
    match kind { "double" => Box::new(|x| x * 2), "neg" => Box::new(|x| -x), _ => Box::new(|x| x) }
}
fn counter() -> impl FnMut() -> u32 { let mut c = 0; move || { c += 1; c } }
fn main() {
    let add5 = adder(5);
    println!("{} {}", add5(1), add5(10));
    println!("{} {} {}", boxed("double")(4), boxed("neg")(4), boxed("?")(4));
    let mut c = counter();
    println!("{} {} {}", c(), c(), c());
    let mut d = counter();
    println!("{}", d());
}
```

✓ Output `6 15`, `8 -4 4`, `1 2 3`, and `1` (each `counter()` call makes an independent closure with its own state). The `move` is required: without it the closure would borrow the parameter `n`, which dies when `adder` returns ✓:

```rust
fn adder(n: i32) -> impl Fn(i32) -> i32 { |x| x + n }
fn main() { println!("{}", adder(1)(2)); }
```

✓ `error[E0373]: closure may outlive the current function, but it borrows `n``.

### 6.1 Two closures, two types

`impl Fn` means *one* hidden type. Returning two different **capturing** closures from `if`/`else` fails ✓:

```rust
fn pick(flag: bool, k: i32) -> impl Fn(i32) -> i32 {
    if flag { move |x| x + k } else { move |x| x + 2 * k }
}
fn main() { println!("{}", pick(true, 1)(1)); }
```

✓ `error[E0308]: `if` and `else` have incompatible types` ("no two closures, even if identical, have the same type"). Fix with `Box<dyn Fn(i32) -> i32>` and `Box::new(..)` in each branch, or by choosing the *data* in the branch and building one closure.

A surprise worth knowing: the **same** code with *non-capturing* closures compiles, because each coerces to a function pointer and both branches unify to `fn(i32) -> i32` ✓:

```rust
fn pick(flag: bool) -> impl Fn(i32) -> i32 {
    if flag { |x| x + 1 } else { |x| x + 2 }
}
fn main() { println!("{}", pick(true)(1)); }
```

✓ Output `2`. It works for this case only; don't rely on it once a closure starts capturing.

### 6.2 Composition and tables of closures

Closures compose into small combinators, and a `Vec`/`HashMap` of boxed closures gives you a table of behaviors ✓:

```rust
fn compose<A, B, C>(f: impl Fn(A) -> B, g: impl Fn(B) -> C) -> impl Fn(A) -> C { move |x| g(f(x)) }
fn apply_n(f: &dyn Fn(i32) -> i32, n: usize, x: i32) -> i32 { (0..n).fold(x, |acc, _| f(acc)) }
fn main() {
    let h = compose(|x: i32| x + 1, |y: i32| y * 10);
    println!("{}", h(4));
    println!("{}", apply_n(&|x| x * 2, 5, 1));
    let fs: Vec<Box<dyn Fn(i32) -> i32>> = vec![Box::new(|x| x + 1), Box::new(|x| x * x)];
    println!("{:?}", fs.iter().map(|f| f(5)).collect::<Vec<_>>());
}
```

✓ Output `50`, `32`, `[6, 25]`.

```rust
use std::collections::HashMap;
fn main() {
    let mut ops: HashMap<&str, Box<dyn Fn(f64, f64) -> Option<f64>>> = HashMap::new();
    ops.insert("+", Box::new(|a, b| Some(a + b)));
    ops.insert("/", Box::new(|a, b| if b == 0.0 { None } else { Some(a / b) }));
    let scale = 10.0;
    ops.insert("scaled", Box::new(move |a, b| Some((a + b) * scale)));
    let eval = |op: &str, a, b| ops.get(op).and_then(|f| f(a, b));
    println!("{:?} {:?} {:?} {:?} {:?}", eval("+", 1.0, 2.0), eval("/", 1.0, 0.0), eval("/", 6.0, 3.0), eval("scaled", 1.0, 1.0), eval("?", 1.0, 1.0));
}
```

✓ Output `Some(3.0) None Some(2.0) Some(20.0) None`: an unknown operator and a division by zero both become `None` instead of a panic.

### 6.3 Recursion

A closure cannot call itself by name, because its name is not in scope inside its own body ✓:

```rust
fn main() {
    let fact = |n: u32| -> u32 { if n == 0 { 1 } else { n * fact(n - 1) } };
    println!("{}", fact(5));
}
```

✓ `error[E0425]: cannot find function `fact` in this scope`. Use a nested `fn` item (which can recurse and cannot capture), or pass the function to itself through `&dyn Fn` ✓:

```rust
fn fact(n: u32) -> u32 { if n == 0 { 1 } else { n * fact(n - 1) } }
fn main() {
    fn inner(n: u32) -> u32 { if n <= 1 { 1 } else { n * inner(n - 1) } }
    println!("{} {}", fact(5), inner(5));
    fn fix(f: &dyn Fn(&dyn Fn(u32) -> u32, u32) -> u32, n: u32) -> u32 { f(&|k| fix(f, k), n) }
    println!("{}", fix(&|rec, n| if n == 0 { 1 } else { n * rec(n - 1) }, 5));
}
```

✓ Output `120 120` and `120`. In practice use a plain `fn` for recursion. The same applies to memoization: the usual shape is a function taking a `&mut HashMap` cache ✓ (`fib(50)` gave `12586269025` with 49 cached entries), or a small struct like the practical challenge's `Memo`. A non-recursive memoizing closure does work, as a stateful `FnMut` capturing the cache ✓:

```rust
use std::collections::HashMap;
fn main() {
    let mut cache: HashMap<u64, u64> = HashMap::new();
    let mut slow_square = |n: u64| *cache.entry(n).or_insert_with(|| { println!("computing {n}"); n * n });
    println!("{} {} {}", slow_square(4), slow_square(4), slow_square(5));
    println!("{}", cache.len());
}
```

✓ Output `computing 4`, `computing 5` (the arguments are evaluated before the line prints), `16 16 25`, `2`. Note it must be `mut` and it holds `&mut cache` until its last use, so `cache.len()` comes after.

---

## 7. How it compiles and what it costs

A closure is a struct whose fields are the captured values (or references to them) plus compiler-generated code to call. Sizes follow directly ✓:

```rust
use std::mem::size_of_val;
fn main() {
    let a = 1u8; let b = 2u64; let s = String::from("hi"); let v = vec![1];
    let none = || 1;
    let by_ref = || a;
    let by_ref2 = || (a, b);
    let moved = move || (a, b);
    let owns_string = move || s.len();
    let owns_vec = move || v.len();
    println!("{} {} {} {} {} {}", size_of_val(&none), size_of_val(&by_ref), size_of_val(&by_ref2), size_of_val(&moved), size_of_val(&owns_string), size_of_val(&owns_vec));
    println!("{}", size_of_val(&(&none as &dyn Fn() -> i32)));
    let boxed: Box<dyn Fn() -> usize> = Box::new(owns_vec);
    println!("{}", size_of_val(&boxed));
    println!("{}", size_of_val(&(none as fn() -> i32)));
}
```

✓ Output `0 8 16 16 24 24`, `16`, `16`, `8`:

- A closure with no captures is **zero-sized** (0).
- Capturing by reference stores one reference per captured variable (8 bytes for `a`, 16 for `a` and `b`).
- `move` stores the values themselves: `(u8, u64)` is 16 with padding, a `String` or `Vec` is 24.
- `&dyn Fn()` and `Box<dyn Fn()>` are fat pointers (data pointer plus vtable pointer, 16 bytes); a `fn` pointer is 8.

These sizes are from this toolchain, not guarantees. Performance consequences:

- **Static dispatch** (`impl Fn`/generics): the call is a known function; the optimizer can inline it, and iterator chains become tight loops (Lesson 17).
- **Dynamic dispatch** (`dyn Fn`): an indirect call per invocation, no inlining across it, plus an allocation for `Box`. Use it for heterogeneous collections and plugin-style APIs, not inner loops.
- **Moving large values into a closure** copies them (a `memcpy` of the struct, Lesson 07); capture a reference or a `Box`/`Rc` if the value is large.
- **Every distinct closure is a new type**, so heavy use of generic `impl Fn` parameters can bloat compile times and binary size (monomorphization). Taking `&dyn Fn` for rarely used callbacks reduces that.

---

## 8. Common mistakes and edge cases

- **Forgetting `move`** when the closure outlives the scope (threads, returned closures, stored callbacks): E0373 ✓.
- **Using a variable after moving it into a closure** (E0382 ✓). Clone before the closure (`let t = s.clone(); move || t...`) or capture a reference.
- **Asking for the wrong trait bound**: `Fn` when the body mutates (E0594 ✓), `Fn` when it consumes (E0507/E0525 ✓), forgetting `mut` on an `FnMut` binding or parameter (E0596 ✓). Prefer the *weakest* trait the callee needs: `FnOnce` if called once, else `FnMut`, else `Fn`.
- **Treating `move` as "FnOnce"**: it only changes capture mode ✓ (the `move` closure `g` was called twice).
- **Expecting a closure to be generic**: its types are fixed by first use (E0308 ✓).
- **Two closures where one type is needed** (E0308 ✓): `Box<dyn Fn>` or compute data in the branch.
- **Closure in a struct field called as a method** (`self.f(x)`): write `(self.f)(x)`.
- **`Box<dyn Fn>` silently being `'static`**: it can't borrow locals (✓ error without code). Add `+ 'a`.
- **Holding a mutable borrow through a closure** longer than you think (E0502 ✓); end its last use before touching the variable again.
- **Reference cycles** through `Rc<RefCell<..>>` captured by a closure stored in that same object: memory is never freed (Lesson 25).
- **Hidden clones / allocations**: a `move` closure around a big `Vec` moves only the three words, but `.clone()` inside the body per call copies the data every time.
- **Panicking callbacks**: a panic in a handler unwinds through the caller; later handlers do not run, and shared state may be half-updated (Lesson 8 and the notes below).
- **Recursion** by name is not available to closures (E0425 ✓).

---

## 9. Security and reliability

- **Callbacks are arbitrary code.** Anything you accept as a closure can panic, block, leak, or capture something expensive. A panic in one `EventBus` handler stops the loop for the remaining handlers (documented unwinding behavior; I did not test it), so decide whether a failing handler should abort the publish, be isolated with `std::panic::catch_unwind` (which has its own caveats), or be written as returning `Result`.
- **Bound registrations.** An API that stores callbacks without a limit lets callers grow memory forever. The practical challenge's bus rejects the `max + 1`th handler (tested) and lets you unsubscribe.
- **Unbounded memoization is a memory (and DoS) risk.** A cache keyed by attacker-controlled input grows without limit; use a bounded cache (Lesson 16's LRU) for untrusted keys.
- **Retries need care.** Retrying a non-idempotent operation (a payment, a transaction broadcast, an email) can run it twice. Retry only idempotent work or use idempotency keys, cap attempts (the challenge's `retry` takes a `NonZeroU32` maximum and never loops forever), use exponential back-off with jitter, and give up immediately on permanent errors (`should_retry` returning `false`). The challenge function does not sleep or add jitter itself; that belongs in your `should_retry`.
- **Captured secrets stay alive** as long as the closure does. A boxed closure holding a key or token keeps it in memory until dropped; zeroizing and key handling are Lesson 48's subject. Capturing a lock guard in a long-lived closure can hold a lock far longer than intended (Lesson 29).
- **`move` into threads** transfers ownership safely, but it is the compiler (`Send`, Lesson 22) that prevents sending non-thread-safe captures like `Rc`; don't work around it with `unsafe`.
- **One-shot initialization** (`Lazy`) must define what happens if the initializer panics: the challenge's `Lazy` poisons and panics on later access rather than rerunning a consumed `FnOnce` (tested with `catch_unwind`).
- **Closures hide control flow.** A deeply nested callback pyramid is hard to audit; prefer named functions for security-critical logic (authorization checks, signature validation) so they can be tested and reviewed on their own.

---

## 10. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** For each closure say which of `Fn`, `FnMut`, `FnOnce` it implements: (a) `|| v.len()`; (b) `|| calls += 1`; (c) `move || owned.into_iter().count()`; (d) `move || v.len()`.
2. **(Easy)** Fix: `let h = thread::spawn(|| println!("{v:?}"));` (error E0373).
3. **(Medium)** Fix this: `let s = String::from("shared"); let h = thread::spawn(move || s.len()); println!("{} {}", h.join().unwrap(), s);`
4. **(Medium)** Write `make_counter() -> impl FnMut() -> u32` returning 1, 2, 3, ... and explain why two counters are independent.
5. **(Medium)** Write `compose(f, g)` returning `g(f(x))`, and `apply_n(f: &dyn Fn(i32) -> i32, n, x)` that applies `f` `n` times.
6. **(Medium)** Memoize Fibonacci. Why is a recursive *closure* the wrong tool, and what is a good shape?
7. **(Hard)** Build a calculator table: a map from operator names to boxed closures `Fn(f64, f64) -> Option<f64>`, with division by zero and unknown operators returning `None` instead of panicking.
8. **(Hard)** Why does `let fact = |n: u32| -> u32 { ... fact(n - 1) ... };` fail, and what are two fixes?

<details markdown="1">
<summary>Answers</summary>

1. ✓ (a) `Fn` (only reads `v`), and therefore also `FnMut` and `FnOnce`. (b) `FnMut` (mutates `calls`), and so `FnOnce`, but not `Fn`. (c) `FnOnce` only (it moves `owned` out by consuming it). (d) `Fn`: `move` only changes how `v` is captured; the body just reads it, so it can be called repeatedly (✓ called twice).
2. Add `move`: `thread::spawn(move || println!("{v:?}"))`. The new thread may outlive the function, so the closure must own `v` ✓. If the thread is guaranteed to end first, `thread::scope` lets you borrow instead ✓.
3. ✓ (run) `s` was moved into the thread, so the later `println!("{}", s)` is `E0382`. Clone before moving:
   ```rust
   let s = String::from("shared");
   let t = s.clone();
   let h = thread::spawn(move || t.len());
   println!("{} {}", h.join().unwrap(), s);   // 6 shared
   ```
   Or use `Arc` when the data is large or shared by many threads (Lesson 23).
4. ✓ (run)
   ```rust
   fn make_counter() -> impl FnMut() -> u32 { let mut c = 0; move || { c += 1; c } }
   ```
   Three calls give `1 2 3`, and a second counter starts again at `1`. Each call to `make_counter` creates a new closure value whose captured `c` is its own field, so two counters share nothing. `move` is needed so the closure owns `c` instead of borrowing a local that dies on return.
5. ✓ (run)
   ```rust
   fn compose<A, B, C>(f: impl Fn(A) -> B, g: impl Fn(B) -> C) -> impl Fn(A) -> C { move |x| g(f(x)) }
   fn apply_n(f: &dyn Fn(i32) -> i32, n: usize, x: i32) -> i32 { (0..n).fold(x, |acc, _| f(acc)) }
   ```
   `compose(|x: i32| x + 1, |y: i32| y * 10)(4)` is `50`; `apply_n(&|x| x * 2, 5, 1)` is `32`. `compose` returns `impl Fn` (static dispatch, no allocation); `move` hands the two closures to the returned one.
6. ✓ (run) A closure can't call itself (E0425), and a memoizing closure that is also recursive would need to borrow its own cache while being called. Use a function that takes the cache:
   ```rust
   use std::collections::HashMap;
   fn fib(n: u64, memo: &mut HashMap<u64, u64>) -> u64 {
       if n < 2 { return n; }
       if let Some(&v) = memo.get(&n) { return v; }
       let v = fib(n - 1, memo) + fib(n - 2, memo);
       memo.insert(n, v);
       v
   }
   ```
   `fib(50, &mut memo)` gave `12586269025` and `memo.len()` was 49. (`u64` overflows at `fib(94)`; use `checked_add` if `n` is untrusted.) For a reusable version, see `Memo` in the practical challenge, which wraps a pure `Fn` and a cache.
7. ✓ (run, output `Some(3.0) None Some(2.0) Some(20.0) None`) See the `HashMap<&str, Box<dyn Fn(f64, f64) -> Option<f64>>>` example in Section 6.2: division checks `b == 0.0` and returns `None`; the lookup uses `ops.get(op).and_then(|f| f(a, b))`, so an unknown operator is also `None`. The `scaled` entry uses `move` to capture `scale`.
8. ✓ A closure's name isn't in scope inside its own body, so `fact(n - 1)` is `E0425`. Fix 1: a plain (or nested) `fn fact(n: u32) -> u32` which can recurse (✓ `120`). Fix 2: pass the recursion in explicitly, e.g. `fn fix(f: &dyn Fn(&dyn Fn(u32) -> u32, u32) -> u32, n: u32) -> u32 { f(&|k| fix(f, k), n) }` and call `fix(&|rec, n| if n == 0 { 1 } else { n * rec(n - 1) }, 5)` (✓ `120`). Fix 1 is almost always the right one.

</details>

---

## 11. Debugging challenge

A teammate wants a list of callbacks and a shared hit counter:

```rust
fn main() {
    let mut handlers: Vec<Box<dyn Fn()>> = Vec::new();
    let mut hits = 0;
    handlers.push(Box::new(|| hits += 1));
    for h in &handlers { h(); }
    println!("{hits}");
}
```

✓ It fails with three errors: `E0594` (cannot assign to `hits`, as it is a captured variable in a `Fn` closure), `E0597` (`hits` does not live long enough) and `E0502` (cannot borrow `hits` as immutable because it is also borrowed as mutable). Explain each, then give two different fixes.

<details markdown="1">
<summary>Solution</summary>

1. **E0594.** `Box<dyn Fn()>` promises the closure only needs `&self`, but `|| hits += 1` mutates a captured variable, which makes it `FnMut`.
2. **E0597.** A `Box<dyn Fn()>` with no lifetime written is `Box<dyn Fn() + 'static>`, but the closure borrows the local `hits`, which does not live for `'static`.
3. **E0502.** Even if the closure were allowed, it keeps a mutable borrow of `hits` alive for as long as `handlers` is, so the `println!("{hits}")` at the end conflicts.

**Fix A: share ownership with interior mutability** (what you want when callbacks must be `'static`, e.g. stored in a long-lived bus) ✓:

```rust
use std::{cell::Cell, rc::Rc};
fn main() {
    let mut handlers: Vec<Box<dyn Fn()>> = Vec::new();
    let hits = Rc::new(Cell::new(0));
    let h = Rc::clone(&hits);
    handlers.push(Box::new(move || h.set(h.get() + 1)));
    for f in &handlers { f(); f(); }
    println!("{}", hits.get());
}
```

✓ Output `2`. The closure now owns an `Rc` clone and mutates through `&self` (a `Cell`), so `Fn` is true. For threads use `Arc<AtomicUsize>` or `Arc<Mutex<_>>` (Lessons 28 and 29).

**Fix B: use `FnMut` and keep the callbacks local**, so a plain borrow is enough ✓:

```rust
fn main() {
    let mut hits = 0;
    {
        let mut handlers: Vec<Box<dyn FnMut() + '_>> = Vec::new();
        handlers.push(Box::new(|| hits += 1));
        for f in &mut handlers { f(); f(); }
    }
    println!("{hits}");
}
```

✓ Output `2`. The trait object is `FnMut` (matching what the body does), the lifetime `'_` lets it borrow `hits`, the iteration uses `&mut handlers` (calling `FnMut` needs a mutable borrow), and the inner block ends the borrows before `hits` is read.

</details>

---

## 12. Practical challenge: `closurelab`

Create a library crate `closurelab` (edition 2024) with:

1. `retry(max_attempts: NonZeroU32, op: impl FnMut(u32) -> Result<T, E>, should_retry: impl FnMut(&E, u32) -> bool) -> Result<T, E>`: calls `op` with the 1-based attempt number, stops on success, stops at the limit or when `should_retry` says no, and returns the last error. It must not call `should_retry` after the final attempt, and must never loop forever.
2. `compose(f, g) -> impl Fn(A) -> C`.
3. `Lazy<T, F: FnOnce() -> T>` with `new`, `get(&mut self) -> &T` and `is_initialized`. The initializer runs at most once, can capture and consume values, and if it panics the `Lazy` is *poisoned* (later `get` calls panic instead of re-running a consumed closure).
4. `Memo<A, R, F: Fn(&A) -> R>` caching a pure function by argument, with `get`, `misses`, `len`, `is_empty`; the function is called once per distinct key.
5. `EventBus<E>` storing boxed `FnMut(&E)` handlers with a hard maximum: `subscribe` (returns a `HandlerId` or `TooManyHandlers`), `unsubscribe`, `publish` (calls handlers in order, returns how many ran), `len`, `is_empty`.
6. `Pipeline<T, E>` of boxed `Fn(T) -> Result<T, E>` stages: builder-style `stage`, `run` stopping at the first error, `len`, `is_empty`, `Default`.
7. Tests for each: retry success/limit/permanent errors/single attempt; lazy runs once, consumes a capture, poisons; memo hit counts for `u64` and `String` keys; bus order with private `FnMut` state, unsubscribe, limit, freeing a slot; pipeline order, short-circuit (a later stage is not called after an error), empty pipeline; compose. A usage doctest.
8. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! Closure-based building blocks that exercise `Fn`, `FnMut` and `FnOnce`:
//! `retry` (`FnMut`), `Lazy` (`FnOnce`), `Memo` (`Fn`), `EventBus` (boxed `FnMut`),
//! and `Pipeline` (boxed `Fn`).
//!
//! ```
//! use closurelab::{compose, retry, Lazy};
//! use std::num::NonZeroU32;
//!
//! let mut calls = 0;
//! let result: Result<u32, &str> = retry(
//!     NonZeroU32::new(3).unwrap(),
//!     |attempt| {
//!         calls += 1;
//!         if attempt < 3 { Err("not yet") } else { Ok(attempt) }
//!     },
//!     |_err, _attempt| true,
//! );
//! assert_eq!(result, Ok(3));
//! assert_eq!(calls, 3);
//!
//! let mut lazy = Lazy::new(|| "computed once".to_string());
//! assert_eq!(lazy.get(), "computed once");
//!
//! let inc_then_double = compose(|x: i32| x + 1, |y: i32| y * 2);
//! assert_eq!(inc_then_double(4), 10);
//! ```

use std::collections::HashMap;
use std::collections::hash_map::Entry;
use std::hash::Hash;
use std::num::NonZeroU32;

/// Calls `op` up to `max_attempts` times, passing the 1-based attempt number.
///
/// After a failure, `should_retry(&error, attempt)` decides whether to try again
/// (use it to give up on permanent errors). The last error is returned if every
/// attempt fails or `should_retry` says stop. Both closures are `FnMut`, so they may
/// keep state such as counters or logs. This function does not sleep: put any
/// back-off delay inside `should_retry`.
pub fn retry<T, E>(
    max_attempts: NonZeroU32,
    mut op: impl FnMut(u32) -> Result<T, E>,
    mut should_retry: impl FnMut(&E, u32) -> bool,
) -> Result<T, E> {
    let mut attempt = 1;
    loop {
        match op(attempt) {
            Ok(value) => return Ok(value),
            Err(err) => {
                if attempt >= max_attempts.get() || !should_retry(&err, attempt) {
                    return Err(err);
                }
            }
        }
        // Cannot overflow: we only get here when `attempt < max_attempts <= u32::MAX`.
        attempt += 1;
    }
}

/// Returns a closure that applies `f` and then `g`.
pub fn compose<A, B, C>(f: impl Fn(A) -> B, g: impl Fn(B) -> C) -> impl Fn(A) -> C {
    move |x| g(f(x))
}

enum LazyState<T, F> {
    Pending(F),
    Ready(T),
    Poisoned,
}

/// A value computed on first access by a one-shot (`FnOnce`) initializer.
///
/// If the initializer panics, the `Lazy` is *poisoned*: later calls to [`Lazy::get`]
/// panic instead of running the (already consumed) initializer again.
pub struct Lazy<T, F> {
    state: LazyState<T, F>,
}

impl<T, F: FnOnce() -> T> Lazy<T, F> {
    /// Creates a lazy value; nothing runs yet.
    pub fn new(init: F) -> Self {
        Lazy {
            state: LazyState::Pending(init),
        }
    }

    /// True once the initializer has run successfully.
    pub fn is_initialized(&self) -> bool {
        matches!(self.state, LazyState::Ready(_))
    }

    /// Returns the value, running the initializer on the first call.
    ///
    /// # Panics
    /// Panics if an earlier initializer call panicked.
    pub fn get(&mut self) -> &T {
        // Move the state out first: `FnOnce` consumes the closure, so we must own it.
        let state = std::mem::replace(&mut self.state, LazyState::Poisoned);
        self.state = match state {
            LazyState::Pending(init) => LazyState::Ready(init()),
            other => other,
        };
        match &self.state {
            LazyState::Ready(value) => value,
            _ => panic!("Lazy value is poisoned: its initializer panicked"),
        }
    }
}

/// Caches the results of a pure function (`Fn`) by argument.
///
/// The cache is unbounded: do not feed it attacker-controlled keys without a limit.
pub struct Memo<A, R, F> {
    f: F,
    cache: HashMap<A, R>,
    misses: u64,
}

impl<A: Hash + Eq, R, F: Fn(&A) -> R> Memo<A, R, F> {
    /// Wraps `f` with an empty cache.
    pub fn new(f: F) -> Self {
        Memo {
            f,
            cache: HashMap::new(),
            misses: 0,
        }
    }

    /// Returns the cached result for `arg`, computing it on the first request.
    pub fn get(&mut self, arg: A) -> &R {
        match self.cache.entry(arg) {
            Entry::Occupied(slot) => slot.into_mut(),
            Entry::Vacant(slot) => {
                self.misses += 1;
                let value = (self.f)(slot.key());
                slot.insert(value)
            }
        }
    }

    /// How many times the underlying function was actually called.
    pub fn misses(&self) -> u64 {
        self.misses
    }

    /// Number of cached entries.
    pub fn len(&self) -> usize {
        self.cache.len()
    }

    /// True if nothing is cached.
    pub fn is_empty(&self) -> bool {
        self.cache.is_empty()
    }
}

/// Identifies a subscription so it can be removed later.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct HandlerId(u64);

/// Returned when the bus already holds its maximum number of handlers.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct TooManyHandlers {
    /// The configured limit.
    pub max: usize,
}

type Handler<E> = Box<dyn FnMut(&E)>;

/// A synchronous event bus holding boxed `FnMut` handlers, with a hard size limit.
///
/// Handlers cannot publish to the bus they are registered on (publishing needs
/// `&mut self`, which the bus holds while calling them); that is the borrow checker
/// ruling out re-entrancy.
pub struct EventBus<E> {
    max_handlers: usize,
    next_id: u64,
    handlers: Vec<(HandlerId, Handler<E>)>,
}

impl<E> EventBus<E> {
    /// Creates a bus that accepts at most `max_handlers` subscriptions.
    pub fn new(max_handlers: usize) -> Self {
        EventBus {
            max_handlers,
            next_id: 0,
            handlers: Vec::new(),
        }
    }

    /// Registers a handler. It must be `'static` because the bus stores it.
    pub fn subscribe(
        &mut self,
        handler: impl FnMut(&E) + 'static,
    ) -> Result<HandlerId, TooManyHandlers> {
        if self.handlers.len() >= self.max_handlers {
            return Err(TooManyHandlers {
                max: self.max_handlers,
            });
        }
        let id = HandlerId(self.next_id);
        // A u64 id counter will not wrap in practice (2^64 subscriptions).
        self.next_id += 1;
        self.handlers.push((id, Box::new(handler)));
        Ok(id)
    }

    /// Removes a handler; returns whether it existed.
    pub fn unsubscribe(&mut self, id: HandlerId) -> bool {
        let before = self.handlers.len();
        self.handlers.retain(|(h, _)| *h != id);
        self.handlers.len() != before
    }

    /// Calls every handler, in subscription order, with `event`. Returns how many ran.
    pub fn publish(&mut self, event: &E) -> usize {
        for (_, handler) in &mut self.handlers {
            handler(event);
        }
        self.handlers.len()
    }

    /// Number of registered handlers.
    pub fn len(&self) -> usize {
        self.handlers.len()
    }

    /// True if no handlers are registered.
    pub fn is_empty(&self) -> bool {
        self.handlers.is_empty()
    }
}

type Stage<T, E> = Box<dyn Fn(T) -> Result<T, E>>;

/// An ordered list of fallible stages; the first error stops the run.
pub struct Pipeline<T, E> {
    stages: Vec<Stage<T, E>>,
}

impl<T, E> Default for Pipeline<T, E> {
    fn default() -> Self {
        Pipeline { stages: Vec::new() }
    }
}

impl<T, E> Pipeline<T, E> {
    /// An empty pipeline (passes its input through unchanged).
    pub fn new() -> Self {
        Self::default()
    }

    /// Appends a stage.
    pub fn stage(mut self, f: impl Fn(T) -> Result<T, E> + 'static) -> Self {
        self.stages.push(Box::new(f));
        self
    }

    /// Number of stages.
    pub fn len(&self) -> usize {
        self.stages.len()
    }

    /// True if there are no stages.
    pub fn is_empty(&self) -> bool {
        self.stages.is_empty()
    }

    /// Runs the input through each stage in order, stopping at the first error.
    pub fn run(&self, input: T) -> Result<T, E> {
        self.stages
            .iter()
            .try_fold(input, |value, stage| stage(value))
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::{Cell, RefCell};
    use std::panic::{AssertUnwindSafe, catch_unwind};
    use std::rc::Rc;

    fn attempts(n: u32) -> NonZeroU32 {
        NonZeroU32::new(n).unwrap()
    }

    #[test]
    fn retry_succeeds_after_failures() {
        let mut calls = 0;
        let r: Result<u32, String> = retry(
            attempts(5),
            |attempt| {
                calls += 1;
                if attempt < 3 {
                    Err(format!("fail {attempt}"))
                } else {
                    Ok(attempt * 10)
                }
            },
            |_, _| true,
        );
        assert_eq!(r, Ok(30));
        assert_eq!(calls, 3);
    }

    #[test]
    fn retry_returns_last_error_at_the_limit() {
        let mut seen = Vec::new();
        let r: Result<(), u32> = retry(attempts(3), Err, |e, _| {
            seen.push(*e);
            true
        });
        assert_eq!(r, Err(3));
        assert_eq!(
            seen,
            vec![1, 2],
            "should_retry is not asked after the final attempt"
        );
    }

    #[test]
    fn retry_stops_on_permanent_errors_and_single_attempt() {
        let mut calls = 0;
        let r: Result<(), &str> = retry(
            attempts(10),
            |_| {
                calls += 1;
                Err("permanent")
            },
            |_, _| false,
        );
        assert_eq!(r, Err("permanent"));
        assert_eq!(calls, 1);
        let mut calls = 0;
        let _: Result<(), ()> = retry(
            attempts(1),
            |_| {
                calls += 1;
                Err(())
            },
            |_, _| true,
        );
        assert_eq!(calls, 1);
    }

    #[test]
    fn lazy_runs_the_fnonce_initializer_once() {
        let owned = String::from("moved into the closure");
        let mut lazy = Lazy::new(move || owned); // consumes the captured String
        assert!(!lazy.is_initialized());
        assert_eq!(lazy.get(), "moved into the closure");
        assert!(lazy.is_initialized());
        assert_eq!(lazy.get(), "moved into the closure");
        let runs = Cell::new(0);
        let mut counted = Lazy::new(|| {
            runs.set(runs.get() + 1);
            5
        });
        assert_eq!(runs.get(), 0);
        counted.get();
        counted.get();
        assert_eq!(runs.get(), 1);
    }

    #[test]
    fn lazy_is_poisoned_if_the_initializer_panics() {
        let mut lazy: Lazy<i32, _> = Lazy::new(|| panic!("boom"));
        assert!(catch_unwind(AssertUnwindSafe(|| *lazy.get())).is_err());
        assert!(!lazy.is_initialized());
        let again = catch_unwind(AssertUnwindSafe(|| *lazy.get()));
        assert!(
            again.is_err(),
            "second call must panic, not re-run the consumed initializer"
        );
    }

    #[test]
    fn memo_calls_the_function_once_per_key() {
        let mut m = Memo::new(|n: &u64| n * n);
        assert_eq!(*m.get(4), 16);
        assert_eq!(*m.get(4), 16);
        assert_eq!(*m.get(5), 25);
        assert_eq!(m.misses(), 2);
        assert_eq!(m.len(), 2);
        assert!(!m.is_empty());
        let mut words = Memo::new(|s: &String| s.len());
        assert_eq!(*words.get("abc".to_string()), 3);
        assert_eq!(*words.get("abc".to_string()), 3);
        assert_eq!(words.misses(), 1);
    }

    #[test]
    fn event_bus_calls_handlers_in_order_with_state() {
        let log = Rc::new(RefCell::new(Vec::new()));
        let mut bus: EventBus<i32> = EventBus::new(4);
        let l1 = Rc::clone(&log);
        bus.subscribe(move |e| l1.borrow_mut().push(format!("a{e}")))
            .unwrap();
        let l2 = Rc::clone(&log);
        let mut total = 0; // private FnMut state, owned by the handler
        bus.subscribe(move |e| {
            total += e;
            l2.borrow_mut().push(format!("sum{total}"));
        })
        .unwrap();
        assert_eq!(bus.publish(&1), 2);
        assert_eq!(bus.publish(&10), 2);
        assert_eq!(*log.borrow(), vec!["a1", "sum1", "a10", "sum11"]);
    }

    #[test]
    fn event_bus_unsubscribe_and_limit() {
        let hits = Rc::new(Cell::new(0));
        let mut bus: EventBus<()> = EventBus::new(2);
        let h1 = Rc::clone(&hits);
        let id = bus.subscribe(move |_| h1.set(h1.get() + 1)).unwrap();
        let h2 = Rc::clone(&hits);
        bus.subscribe(move |_| h2.set(h2.get() + 100)).unwrap();
        assert_eq!(bus.subscribe(|_| {}), Err(TooManyHandlers { max: 2 }));
        bus.publish(&());
        assert_eq!(hits.get(), 101);
        assert!(bus.unsubscribe(id));
        assert!(!bus.unsubscribe(id), "second removal finds nothing");
        bus.publish(&());
        assert_eq!(hits.get(), 201);
        assert_eq!(bus.len(), 1);
        assert!(bus.subscribe(|_| {}).is_ok(), "space freed by unsubscribe");
        assert!(!bus.is_empty());
    }

    #[test]
    fn pipeline_runs_in_order_and_stops_at_first_error() {
        let later_called = Rc::new(Cell::new(false));
        let flag = Rc::clone(&later_called);
        let p: Pipeline<i32, String> = Pipeline::new()
            .stage(|x| Ok(x + 1))
            .stage(|x| {
                if x > 10 {
                    Err(format!("{x} too big"))
                } else {
                    Ok(x * 2)
                }
            })
            .stage(move |x| {
                flag.set(true);
                Ok(x)
            });
        assert_eq!(p.len(), 3);
        assert_eq!(p.run(1), Ok(4));
        assert!(later_called.get());
        later_called.set(false);
        assert_eq!(p.run(10), Err("11 too big".to_string()));
        assert!(
            !later_called.get(),
            "the last stage must not run after an error"
        );
        let empty: Pipeline<i32, ()> = Pipeline::new();
        assert!(empty.is_empty());
        assert_eq!(empty.run(7), Ok(7));
    }

    #[test]
    fn compose_applies_left_to_right() {
        let f = compose(|x: i32| x + 1, |y: i32| y.to_string());
        assert_eq!(f(4), "5");
        let g = compose(str::len, |n: usize| n * 2);
        assert_eq!(g("abc"), 6);
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 10 unit tests and 1 doctest passed. (On the first run clippy's `redundant_closure` lint flagged `|a| Err(a)` in a test; replacing it with `Err` is exactly the "use a function where a closure is expected" point from Section 4.)

**Design notes.**

- **Pick the weakest trait that works.** `retry` takes `FnMut` for both closures (they may count or log, and are called many times); `Lazy` takes `FnOnce` (it is called at most once and may consume its captures); `Memo` takes `Fn` (a *pure* function whose result is safe to cache); the bus stores `FnMut` handlers; the pipeline stores `Fn` stages so `run(&self)` can be called repeatedly and shared.
- **`Lazy` and `FnOnce`.** Calling a `FnOnce` consumes it, so `get` first moves the whole state out with `mem::replace` (leaving `Poisoned`), runs the initializer, and writes `Ready` back. If the initializer panics, the assignment never happens and the state stays `Poisoned`; the test confirms the second `get` panics rather than running a closure that no longer exists.
- **`Memo` and disjoint borrows.** In `get`, `self.cache.entry(..)`, `self.misses += 1` and `(self.f)(slot.key())` touch three different fields, so the borrow checker accepts them together (Lesson 10). The `Entry` API hashes once.
- **`EventBus` and `'static`.** `Box<dyn FnMut(&E)>` is `'static`, so `subscribe` requires `impl FnMut(&E) + 'static`; tests share state via `Rc<RefCell<_>>`/`Rc<Cell<_>>`. Handlers cannot re-enter the bus because `publish` holds `&mut self`: the compiler rules out re-entrancy bugs. The `max_handlers` limit and `unsubscribe` bound memory.
- **`Pipeline::run`** is `try_fold`: the `?`-like early exit of Lesson 17 means a failing stage prevents later stages from running (tested with a flag).
- **`retry` and overflow.** `attempt += 1` can only run when `attempt < max_attempts <= u32::MAX`, so it cannot overflow. There is no sleep, jitter or idempotency check in the function; those are the caller's policy (Section 9).
- **Not production code.** No async support, no thread safety (`Rc` and non-`Send` handlers), no panic isolation in `publish`, no cache eviction in `Memo`.

</details>

---

## 13. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is a closure in Rust?** An anonymous function that can capture variables from its environment. The compiler makes a unique struct type holding the captures and implementing one or more of `Fn`, `FnMut`, `FnOnce`.
2. **How do closures capture variables?** By shared reference, mutable reference, or by value, whichever the body needs (the least restrictive that works); `move` forces by-value capture of everything used.
3. **Explain `Fn`, `FnMut`, `FnOnce`.** `Fn` can be called through `&self` repeatedly (only reads captures); `FnMut` needs `&mut self` (may mutate); `FnOnce` takes `self` (may consume captures, can be called once). Every `Fn` is `FnMut`, and every `FnMut` is `FnOnce`.
4. **Does `move` make a closure `FnOnce`?** No. `move` only changes capture mode; the trait depends on what the body does with the captures.
5. **When is `move` required?** When the closure may outlive the borrowed variables: spawning threads, returning closures, storing them in `'static` containers.
6. **How do you return a closure?** `impl Fn(..)` for a single concrete type (no allocation), or `Box<dyn Fn(..)>` to return different closures or erase the type.
7. **Why can't two closures be the same type, and what does that mean for `if/else` returning closures?** Each closure expression has its own unnamed type; branches must unify, so use `Box<dyn Fn>` (or build one closure from data). Non-capturing closures can coerce to `fn` pointers.
8. **What's the difference between `fn` pointers and closures?** `fn(i32) -> i32` is a plain pointer type; it accepts functions and non-capturing closures. Closures that capture are not `fn` pointers (E0308), only `Fn*` implementors.
9. **`impl Fn` vs `dyn Fn` as a parameter?** `impl Fn`/generics are monomorphized (static dispatch, inlinable, code duplication); `&dyn Fn`/`Box<dyn Fn>` use a vtable (dynamic dispatch, one copy, heterogeneous collections).
10. **How big is a closure?** The sum of its captures: zero for non-capturing, a reference per borrowed variable, the value itself for `move`-captured ones. A `Box<dyn Fn>` is a 16-byte fat pointer plus the heap data.
11. **How do you store a closure in a struct?** Make the struct generic over `F: Fn…` (zero cost) or hold `Box<dyn Fn…>` (flexible); call it as `(self.field)(args)`.
12. **Why is `Box<dyn Fn()>` `'static` by default, and how do you borrow locals?** The default object lifetime bound in `Box<dyn Trait>` is `'static`; write `Box<dyn Fn() + 'a>` to allow borrowing.
13. **Why can't a closure call itself?** Its name isn't in scope in its own body; use a `fn` item or pass the function in.
14. **Which error do you get calling a `FnOnce` twice?** `E0382` "use of moved value" on the closure (the call moved it). Passing it where `Fn` is required gives `E0507` or `E0525`.
15. **What does edition 2021 change for closures?** Closures capture disjoint fields rather than whole variables, so they conflict less with other borrows (RFC 2229).
16. **How would you share mutable state between a closure and surrounding code?** `Rc<RefCell<T>>`/`Rc<Cell<T>>` on one thread, `Arc<Mutex<T>>`/atomics across threads; beware reference cycles.
17. **What are the risks of accepting callbacks in an API?** Panics, blocking, re-entrancy, unbounded registration, captured secrets and locks living too long. Bound, document and isolate them.
18. **How do closures relate to iterators?** Adapters like `map`/`filter`/`fold` take closures as generic parameters; because the closure type is known, the whole chain is inlined (Lesson 17).

</details>

---

## 14. Key takeaways

- A closure is an anonymous struct holding its captures plus a call; each closure has its own type. Types are inferred from the first use and are not generic.
- Capture mode comes from the body: read means `&`, mutate means `&mut`, consume means by value. `move` forces by-value capture; it is required when the closure outlives its scope (threads, returned or stored closures, E0373).
- `Fn` (read, `&self`) is a subtrait of `FnMut` (mutate, `&mut self`) is a subtrait of `FnOnce` (consume, `self`). Ask for the weakest bound the callee needs; the compiler infers the strongest the closure supports.
- Know the errors: E0382 (reuse after move or a second `FnOnce` call), E0507/E0525 (`FnOnce` where `Fn` is needed), E0594 (mutation in `Fn`), E0596 (missing `mut`), E0373, E0502, E0308 (two closure types, or capturing closure to `fn`).
- Parameters: generics/`impl Fn` (fast, monomorphized), `&dyn Fn`/`Box<dyn Fn>` (flexible, indirect), `fn` pointers (non-capturing only). Return `impl Fn` or `Box<dyn Fn>`; `Box<dyn Fn>` is `'static` unless you write a lifetime.
- Closures are zero-sized when they capture nothing, and as large as their captures otherwise; sharing state needs `Rc`/`Arc` plus interior mutability.
- Callbacks are arbitrary code: bound their number, think about panics and re-entrancy, and use named functions for critical logic.

**Connections.** Lesson 17's adapters all take closures; Lessons 7, 8 and 10 explain moves, drops and borrows of captures; Lesson 12's `for<'a>` appears in `Fn(&str) -> &str`. Next: Module 7, starting with Lesson 19 (generics and trait bounds, which explain `F: Fn(A) -> B`, `where` clauses and monomorphization), then Lesson 20 (associated types and coherence; `Fn*` use associated types for the return type), Lesson 21 (trait objects: `dyn Fn` and vtables) and Lesson 22 (GATs, `Send`/`Sync`: why `Rc` closures can't cross threads). Lessons 23 to 25 cover `Rc`/`Arc`/`RefCell`/`Weak`, Lesson 27 threads and `move` closures, Lesson 30 async blocks (which are closure-like state machines), and Lesson 34 macros (a macro can do what a closure cannot, e.g. take names). See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 15. References

- *The Rust Programming Language*, ch. 13.1 "Closures: Anonymous Functions that Capture Their Environment": <https://doc.rust-lang.org/book/ch13-01-closures.html>
- *The Rust Reference*, "Closure expressions" and "Closure types" (capture modes, `move`, the `Fn` traits implemented): <https://doc.rust-lang.org/reference/expressions/closure-expr.html>, <https://doc.rust-lang.org/reference/types/closure.html>
- `std::ops::Fn`, `FnMut`, `FnOnce`: <https://doc.rust-lang.org/std/ops/trait.Fn.html>, <https://doc.rust-lang.org/std/ops/trait.FnMut.html>, <https://doc.rust-lang.org/std/ops/trait.FnOnce.html>
- *Rust By Example*, "Closures" (capturing, as input parameters, type anonymity, `Fn` families): <https://doc.rust-lang.org/rust-by-example/fn/closures.html>
- `std::thread::scope` and `std::thread::spawn`: <https://doc.rust-lang.org/std/thread/fn.scope.html>, <https://doc.rust-lang.org/std/thread/fn.spawn.html>
- RFC 2229 "Closures capture disjoint fields" (edition 2021 capture rules): <https://rust-lang.github.io/rfcs/2229-capture-disjoint-fields.html>
- *The Rustonomicon*, "Higher-Rank Trait Bounds": <https://doc.rust-lang.org/nomicon/hrtb.html>
- Rust error code index: E0373, E0382, E0507, E0525, E0594, E0596: <https://doc.rust-lang.org/error_codes/E0373.html>, <https://doc.rust-lang.org/error_codes/E0525.html>, <https://doc.rust-lang.org/error_codes/E0507.html>
- Clippy lint `redundant_closure`: <https://rust-lang.github.io/rust-clippy/master/index.html#redundant_closure>

The URLs above are the standard documentation locations; I did not open every one during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the RFC 2229 number and URL, the Rust Book chapter and Reference page names, that the 2018 edition captured whole variables (I only ran the 2024 edition), that `Box<dyn FnMut>` handler panics skip later handlers and cause partial updates (documented unwinding behavior, not tested), and the `catch_unwind` caveats.
