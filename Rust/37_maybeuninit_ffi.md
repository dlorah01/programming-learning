---
title: "Lesson 37 — MaybeUninit, FFI, repr(C), and C interoperability"
---

# Lesson 37 — `MaybeUninit`, FFI, `repr(C)`, and C interoperability

**Module 12 · Unsafe Rust and low-level programming** · Status: generated · Edition used in examples: **2024** · External dependencies: none in the library (`cc` as a *build* dependency in the practical crate)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, `gcc 13.3` and `valgrind`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. **Miri is not installed here** (Lesson 38 covers it), so nothing below was checked with Miri; I used the standard library's debug-build checks and `valgrind`. I could not open the Rust documentation sites, so the language rules are from memory plus what the compiler accepted; the closing paragraph lists them. All layout numbers are for this x86-64 Linux target; other targets may differ.

Builds on [Lesson 36](36_unsafe_raw_pointers.md) (unsafe contracts, raw pointers, UB, `SAFETY` comments), [Lesson 33](33_modules_api_design_rustdoc.md) (privacy as the soundness boundary), [Lesson 22](22_gats_send_sync_phantom.md) (`Send`/`Sync`, `PhantomData`), [Lesson 26](26_error_design.md) (error types for the wrappers), [Lesson 15](15_newtype_typestate.md) (newtypes), and [Lesson 03](03_cargo_projects.md) (build scripts and Cargo). Lesson 38 (`no_std`, embedded basics, Miri) is next; it appears here only as plain text.

---

## 1. Why this lesson exists

Almost every real system has C in it: the operating system, `libc`, OpenSSL, SQLite, GPU drivers, the Solana and Substrate runtimes' host interfaces. **FFI** (foreign function interface) is how Rust code calls such code and how C calls Rust. At the boundary the compiler stops helping, because it cannot see the other side. Two skills decide whether the boundary is sound:

1. **Layout**: both sides must agree, byte for byte, on what a value looks like. That is `repr(C)`.
2. **Initialization and ownership**: C hands you memory that is half-written, or memory you must free with *its* `free`. That is `MaybeUninit` and RAII wrappers.

The pattern of the whole lesson: keep the raw `extern` declarations private, and expose a small **safe wrapper** whose `SAFETY` comments prove every precondition (Lesson 36).

---

## 2. Memory layout and `repr`

By default Rust promises almost nothing about how a struct is laid out. The compiler may reorder fields to remove padding ✓:

```rust
use std::mem::{align_of, offset_of, size_of};

struct Auto { a: u8, b: u32, c: u8 }

#[repr(C)]
struct C { a: u8, b: u32, c: u8 }

#[repr(C, packed)]
struct Packed { a: u8, b: u32, c: u8 }

#[repr(C, align(16))]
struct Aligned { a: u8 }

#[repr(transparent)]
struct Meters(f64);

fn main() {
    println!("Auto:   size {} align {}", size_of::<Auto>(), align_of::<Auto>());
    println!("C:      size {} align {}", size_of::<C>(), align_of::<C>());
    println!("C offsets: a={} b={} c={}", offset_of!(C, a), offset_of!(C, b), offset_of!(C, c));
    println!("Packed: size {} align {}", size_of::<Packed>(), align_of::<Packed>());
    println!("Packed offsets: b={} c={}", offset_of!(Packed, b), offset_of!(Packed, c));
    println!("Aligned: size {} align {}", size_of::<Aligned>(), align_of::<Aligned>());
    println!("Meters: size {} align {}", size_of::<Meters>(), align_of::<Meters>());
}
```

```text
Auto:   size 8 align 4
C:      size 12 align 4
C offsets: a=0 b=4 c=8
Packed: size 6 align 1
Packed offsets: b=1 c=5
Aligned: size 16 align 16
Meters: size 8 align 8
```

I ran this and, separately, compiled the same struct in C with `gcc`: `sizeof` is 12 and `offsetof(b)`, `offsetof(c)` are 4 and 8 ✓, matching `repr(C)`.

| Attribute | Meaning | Use it for |
|---|---|---|
| none (`repr(Rust)`) | unspecified; the compiler may reorder fields | everything that never crosses a boundary |
| `repr(C)` | fields in declaration order, C padding rules | structs and unions shared with C, or whose layout you rely on |
| `repr(C, packed)` | no padding, alignment 1 | on-the-wire or file formats; **references to fields are a hazard** (§2.1) |
| `repr(C, align(N))` | at least alignment `N` | cache-line alignment, SIMD buffers |
| `repr(transparent)` | same layout and ABI as its single non-zero-sized field | newtypes passed to C as the inner type |
| `repr(u8)`, `repr(i32)`, … on an enum | the discriminant has that integer type | fieldless enums crossing the boundary |

For enums ✓: a plain fieldless enum was 1 byte, `repr(u8)` 1 byte, and `repr(C)` 4 bytes (a C `int`). The values of a C enum that Rust does not know are **not** a valid Rust enum: if C can return a value you did not list, receive it as the integer and convert with a checked `TryFrom` (Lesson 26), never as the enum type.

**Niches.** Some types have invalid bit patterns that Rust uses to store `None` for free. ✓ `Option<&u8>`, `Option<Box<u8>>` and `Option<extern "C" fn()>` are all 8 bytes, the size of a pointer, while `Option<u32>` is also 8 bytes, but only because it needs a separate tag (a `u32` has no spare bit pattern). This is why `Option<&T>` and `Option<extern "C" fn(..)>` are the correct FFI types for "pointer or NULL" and "callback or NULL": NULL *is* `None`. A plain `&T` or a plain `fn` must never be NULL.

### 2.1 `packed` and references

Taking `&packed.field` to a misaligned field is rejected or warned about, because a reference must be aligned. Read with a copy (`let v = packed.b;`) or through `&raw const packed.b` and `read_unaligned` (Lesson 36). `repr(packed)` buys compactness and costs correctness risk; use it only for byte-exact formats.

### 2.2 Make layout drift a compile error

Do not trust that your Rust struct matches the C header; assert it at compile time, as the practical crate does:

```rust
use std::mem::{align_of, offset_of, size_of};

#[repr(C)]
pub struct Point { x: i32, y: i32 }

const _: () = {
    assert!(size_of::<Point>() == 8);
    assert!(align_of::<Point>() == 4);
    assert!(offset_of!(Point, y) == 4);
};

fn main() {}
```

A failing `assert!` in a `const` is a compile error. The crate also compares against the *C compiler's* `sizeof`/`offsetof` in a test, so both sides are checked.

---

## 3. `MaybeUninit<T>`: memory that is not (yet) a `T`

Reading an uninitialized integer, `bool`, reference or enum is undefined behavior even if you "never look at it" (Lesson 36). So how do you hand C a buffer to fill in? Not with `let x: i32 = unsafe { mem::zeroed() }` style tricks, and never with the removed `mem::uninitialized`. You use **`MaybeUninit<T>`**: a type that has the size and alignment of `T` but makes *no* claim that it holds a valid `T`. Creating one, writing it, and passing a pointer to it are all fine; only `assume_init` (and its relatives) is `unsafe`, because *you* then promise it is initialized.

```rust
use std::mem::MaybeUninit;

fn main() {
    // 1. Write through a pointer, then claim it is initialized.
    let mut a = MaybeUninit::<u64>::uninit();
    unsafe { a.as_mut_ptr().write(0xDEAD_BEEF) };          // writing is fine
    let a = unsafe { a.assume_init() };                    // the promise
    println!("{a:#x}");

    // 2. The safe way to initialize: `write` returns &mut T.
    let mut b = MaybeUninit::<String>::uninit();
    let r: &mut String = b.write(String::from("hi"));
    r.push('!');
    let b = unsafe { b.assume_init() };
    println!("{b}");
}
```

✓ Output: `0xdeadbeef` and `hi!`.

**Key facts, all ✓ (small programs):**

- `MaybeUninit<T>` **never drops** its contents. A `MaybeUninit<Noisy>` that was initialized but never `assume_init`-ed silently leaks (no `Drop` output). Use `assume_init_drop()` to run the destructor in place, or `assume_init()` to move the value out and let it drop normally.
- `size_of::<MaybeUninit<u64>>()` is 8, while `Option<u64>` is 16. `MaybeUninit` has no tag: *you* are the tag. That is the whole point and the whole danger.
- `assume_init` on memory that is not fully initialized is UB, and it is a *validity* issue too: an initialized-but-invalid value (a `bool` of 2, a null `&T`) is also UB.

### 3.1 Arrays and partial structs

```rust
use std::mem::MaybeUninit;

struct Config { id: u32, name: [u8; 4], on: bool }

fn main() {
    // An array of uninitialized slots. `[const { .. }; N]` works for any T
    // because the const block is evaluated per element.
    let mut arr: [MaybeUninit<u8>; 3] = [const { MaybeUninit::uninit() }; 3];
    for (i, slot) in arr.iter_mut().enumerate() {
        slot.write(i as u8 * 10);
    }
    // SAFETY: all three slots were written; MaybeUninit<u8> and u8 have the same layout.
    let arr: [u8; 3] = unsafe { std::mem::transmute(arr) };
    println!("{arr:?}");

    // Initialize a struct field by field, without ever creating a reference
    // to uninitialized memory.
    let mut cfg = MaybeUninit::<Config>::uninit();
    let p = cfg.as_mut_ptr();
    unsafe {
        (&raw mut (*p).id).write(7);
        (&raw mut (*p).name).write(*b"abcd");
        (&raw mut (*p).on).write(true);
    }
    // SAFETY: every field was written above.
    let cfg = unsafe { cfg.assume_init() };
    println!("{} {:?} {}", cfg.id, cfg.name, cfg.on);
}
```

✓ Prints `[0, 10, 20]` and `7 [97, 98, 99, 100] true`. The `&raw mut (*p).field` syntax (Lesson 36) matters: `&mut (*p).field` would create a reference to an uninitialized place, which is itself UB for most types.

In everyday code, prefer `[T; N]::from_fn`/`std::array::from_fn`; the practical crate builds a tiny version by hand to show the **panic-safety** problem (§9.2).

### 3.2 When C fills a buffer for you

`Vec::with_capacity` plus `spare_capacity_mut()` gives you `&mut [MaybeUninit<T>]` of the unused capacity. You may pass its pointer to C, and after C reports how many bytes it wrote, call `set_len`. The count came from C, so **validate it** before `set_len` (§8). This avoids zero-filling a large buffer first.

---

## 4. Declaring and calling C functions

In edition 2024, a block of foreign declarations must be written `unsafe extern`. Each item is `unsafe` to call by default; an item you have verified to have no preconditions can be marked `safe fn` inside the block ✓:

```rust
use std::ffi::{CStr, CString, c_char, c_int, c_void};

unsafe extern "C" {
    fn strlen(s: *const c_char) -> usize;
    fn abs(x: c_int) -> c_int;
    safe fn labs(x: std::ffi::c_long) -> std::ffi::c_long;
    fn getenv(name: *const c_char) -> *const c_char;
    fn qsort(
        base: *mut c_void,
        n: usize,
        size: usize,
        cmp: extern "C" fn(*const c_void, *const c_void) -> c_int,
    );
}

extern "C" fn cmp(a: *const c_void, b: *const c_void) -> c_int {
    // SAFETY: qsort passes pointers to two live, aligned i32 elements.
    let (a, b) = unsafe { (*a.cast::<i32>(), *b.cast::<i32>()) };
    a.cmp(&b) as c_int
}

fn main() {
    let s = CString::new("héllo").unwrap();
    // SAFETY: `s` is NUL-terminated and outlives the call.
    println!("strlen = {}", unsafe { strlen(s.as_ptr()) });
    println!("chars  = {}", "héllo".chars().count());
    println!("abs    = {}", unsafe { abs(-5) });
    println!("labs   = {}", labs(-9));

    let mut v = [5, 3, 9, 1, 7];
    // SAFETY: valid array, correct element size, comparator matches the type.
    unsafe { qsort(v.as_mut_ptr().cast(), v.len(), size_of::<i32>(), cmp) };
    println!("{v:?}");

    let unset = CString::new("DEFINITELY_NOT_SET_12345").unwrap();
    let p = unsafe { getenv(unset.as_ptr()) };
    println!("unset is null: {}", p.is_null());
    let path = unsafe { getenv(c"PATH".as_ptr()) };
    if !path.is_null() {
        // SAFETY: getenv returns a NUL-terminated string valid until the environment changes.
        let _ = unsafe { CStr::from_ptr(path) }.to_string_lossy().len();
    }
}
```

✓ Output: `strlen = 6` (bytes, not 5 characters), `chars  = 5`, `abs = 5`, `labs = 9`, `[1, 3, 5, 7, 9]`, `unset is null: true`.

Points worth stating:

- **`extern "C"` selects the C calling convention**; it says nothing about the types. A Rust `extern "C" fn` can only be *called by C* safely if every parameter and return type is FFI-safe.
- The **FFI-safety lint** catches the common mistakes ✓. A function `extern "C" fn f(s: String)` (or `&str`, `Vec<u8>`) produces `warning: ... not FFI-safe` (`improper_ctypes_definitions`), with notes such as "string slices have no C equivalent" and "consider using `*const u8` and a length instead". `repr(C)` structs, integers, floats and pointers pass; so does `Option<&T>`.
- Calling an `unsafe extern "C"` function without an `unsafe` block is `E0133` ✓. A `safe fn` declaration is **your promise to every caller**; mark only functions with no pointer or state preconditions (`labs`).
- Use the type aliases `std::ffi::{c_int, c_char, c_long, c_void, ...}` instead of guessing: C's `int` is not necessarily `i32` on every platform, and `char` signedness differs.
- **Linking.** `libc` is already linked, so `strlen` and friends resolve. For your own C code you either add `#[link(name = "foo")]` / `cargo:rustc-link-lib=foo` in a build script (§9) or compile it with the `cc` crate.
- **Variadics** (`printf`) can be *declared* with `...` but are a minefield (types are promoted by C rules); prefer wrapping them in a small C function with fixed arguments.

---

## 5. Strings across the boundary

Rust `str` is UTF-8 with an explicit length and may contain `\0`. A C string is bytes ending at the first `\0`, with no encoding. `std::ffi` has a type for each side:

| Type | Owned? | Guarantees |
|---|---|---|
| `CString` | owned | ends in exactly one NUL, **no interior NUL** |
| `&CStr` | borrowed | NUL-terminated, no interior NUL; encoding unspecified |
| `c"literal"` | `&'static CStr` | compile-time checked |

✓ Observed behavior:

```rust
use std::ffi::{CStr, CString};

fn main() {
    assert_eq!(CString::new("hi").unwrap().as_bytes_with_nul(), b"hi\0");

    let err = CString::new("a\0b").unwrap_err();
    println!("nul at {}", err.nul_position());                     // 1

    let lit: &CStr = c"literal";                                    // checked at compile time
    println!("{}", lit.to_str().unwrap());

    let bytes = b"\xff\xfe\0";
    let s = CStr::from_bytes_with_nul(bytes).unwrap();              // OK: any bytes before the NUL
    println!("to_str is_err: {}", s.to_str().is_err());             // true: not UTF-8
    println!("lossy: {}", s.to_string_lossy());                     // replacement characters

    println!("no nul is_err: {}", CStr::from_bytes_with_nul(b"no nul").is_err());
    println!("until_nul len: {}", CStr::from_bytes_until_nul(b"ab\0cd\0").unwrap().count_bytes());
}
```

Prints `nul at 1`, `literal`, `to_str is_err: true`, `lossy: ��`, `no nul is_err: true`, `until_nul len: 2` ✓.

**Rules of thumb.**

1. Rust → C: build a `CString` (handle `NulError`: an interior NUL would silently truncate the string on the C side), keep it **alive in a named variable** for as long as C may read the pointer, and pass `.as_ptr()`.
2. C → Rust: `CStr::from_ptr(p)` (unsafe: `p` must be non-null, NUL-terminated, valid for the whole borrow), then `to_str()` (UTF-8 checked) or `to_string_lossy()`. Copy into a `String` if the C memory may change or be freed.
3. Never use `CString::new(..).unwrap().as_ptr()` in one expression: the temporary dies at the end of the statement and the pointer dangles. Since recent compilers this is a warning ✓ (`dangling_pointers_from_temporaries`), and §10 turns it into the debugging challenge.
4. A C `char *` that is *not* a text string (a byte buffer) should be `*const u8` plus a length, not a `CStr`.

---

## 6. Ownership across the boundary

C has no destructors; ownership is a convention. Decide, document, and encode it:

| Situation | Rust type for it |
|---|---|
| C returns a pointer you must free with **its** `free` function | a private RAII wrapper (`struct CAllocated(NonNull<c_char>)`) whose `Drop` calls the C free function |
| C returns an opaque handle (`counter_t *`) | a wrapper `struct Counter { raw: NonNull<counter_t>, .. }` with `Drop` |
| You lend a buffer for the duration of the call | `&mut [u8]` or `*mut u8` + length; C must not keep it |
| C keeps a pointer after the call returns | you must guarantee the pointee stays alive and unmoved (`Box::into_raw` / `Pin`), and say so in the API |

**Never free memory with a deallocator that did not allocate it.** Memory from C's `malloc` must go back through C's `free` (or the library's own free function), not `Box::from_raw` or Rust's allocator, and vice versa; the two allocators may even be different libraries ✓ (the practical crate frees through `c_free_string`).

**Opaque types.** A C type you only hold by pointer is declared in Rust as a struct with a zero-size private field, so nobody can construct it or take its size:

```rust
#[repr(C)]
pub struct counter_t {
    _private: [u8; 0],
}
```

**`Send`/`Sync` are not automatic.** A raw pointer makes your wrapper `!Send + !Sync`, which is the *right default* for a C handle: nothing says the C library is thread-safe. The practical crate adds `PhantomData<*mut ()>` to make that explicit and tests it with a `compile_fail` doctest (Lesson 22). Add `unsafe impl Send` only when the C library documents it (Lesson 36 §4.3).

**Out-parameters** (`int f(.., T *out)`): the idiom is `let mut out = MaybeUninit::<T>::uninit(); let rc = f(.., out.as_mut_ptr()); if rc == 0 { out.assume_init() }`. Only assume initialization on the path where the C documentation says `out` was written ✓ (`Point::new` does this; on the error path C leaves `out` untouched, and the wrapper never reads it).

---

## 7. Callbacks, closures, and panics

C callbacks have the shape `int (*cb)(int x, void *user)`: a function pointer plus an opaque `user` pointer. Rust closures capture state, so pass the closure's address as `user` and use a **generic trampoline** that casts it back:

```rust
use std::ffi::{c_int, c_void};

extern "C" fn trampoline<F: FnMut(i32)>(x: i32, user: *mut c_void) -> c_int {
    // SAFETY: `user` is the `&mut F` we passed to C, alive for the whole call.
    let f = unsafe { &mut *user.cast::<F>() };
    f(x);
    0
}

/// Stands in for the C library: it only knows a function pointer and `user`.
fn fake_c_for_each(xs: &[i32], cb: extern "C" fn(i32, *mut c_void) -> c_int, user: *mut c_void) {
    for &x in xs {
        if cb(x, user) != 0 {
            break;
        }
    }
}

fn run<F: FnMut(i32)>(xs: &[i32], mut f: F) {
    fake_c_for_each(xs, trampoline::<F>, (&raw mut f).cast::<c_void>());
}

fn main() {
    let mut seen = Vec::new();
    run(&[1, 2, 3], |x| seen.push(x));
    println!("{seen:?}");
}
```

`trampoline::<F>` is a different function for each closure type `F`; that is how a stateless C function pointer reaches stateful Rust code.

### 7.1 A panic must not cross a plain `extern "C"` frame

Unwinding through C frames is not defined for the `"C"` ABI. Since Rust 1.81, a panic that tries to leave an `extern "C"` function **aborts the process**. I ran a program in which an `extern "C" fn` panics ✓: the process printed the panic message and `panic in a function that cannot unwind`, then died with exit status 134 (`SIGABRT`).

Two correct designs, both run ✓:

- **Catch at the boundary** (the default). Wrap the body in `std::panic::catch_unwind`, turn the failure into an error code, and (for callbacks) store the payload and `resume_unwind` it after C returns, as the practical crate's `for_each` does. The test in the crate shows that after the closure panics on the second element, C stops iterating and the caller sees the original panic message `"stop at 2"`.
- **`extern "C-unwind"`** declares that unwinding through this boundary is allowed, but it is only sound if *the other side also unwinds correctly* (C++ exceptions, or C compiled with unwind tables), and a Rust `catch_unwind` caller can then catch the panic. In my experiment ✓ this worked for a Rust-to-Rust call through a `"C-unwind"` function. Use it only when the foreign side is documented to cooperate.

Also remember that `catch_unwind` catches only unwinding panics: with `panic = "abort"` there is nothing to catch, the process aborts, and an FFI boundary does not change that.

---

## 8. Exporting Rust to C

To call Rust from C, export a function with an unmangled name and the C ABI. In edition 2024 the attribute must be written `#[unsafe(no_mangle)]`, because a duplicated symbol name can break soundness (from memory; the compiler enforced the new spelling ✓):

```rust
#[unsafe(no_mangle)]
pub extern "C" fn rs_add(a: i32, b: i32) -> i32 {
    a.wrapping_add(b)
}
```

I built a small `staticlib` exporting `rs_add`, `rs_sum(ptr, len)`, `rs_count_chars(cstr)` and `rs_checked_div(a, b, *mut out)`, wrote the header by hand, and called it from a C `main` ✓:

```text
$ rustc --edition 2024 --crate-type staticlib -C panic=unwind rs_export.rs -o libexp.a
$ gcc -Wall -o main main.c libexp.a -lpthread -ldl -lm
add = 5
sum = 10
chars = 5, null = -1, bad utf8 = -2
div ok = 0 (q=3)
div by zero = -1
```

`nm` listed `T rs_add`, `rs_checked_div`, `rs_count_chars`, `rs_sum` ✓. The debug static library was about 22 MB (it contains the whole standard library; release builds and LTO shrink it, and I did not measure those).

Design rules for exported functions:

- Take only FFI-safe types; use `*const T` + length for slices and return an **error code** (negative or non-zero) instead of panicking. `rs_count_chars` returns `-1` for NULL and `-2` for invalid UTF-8 ✓.
- Make a function that has unchecked preconditions `unsafe extern "C"` and write a `# Safety` section (Lesson 36); check for NULL anyway, as defense in depth, because C callers have no borrow checker.
- Use a `catch_unwind` wrapper if the body can panic (`rs_checked_div` ✓; division by zero returned `-1` and the panic message went to stderr).
- Writing the C header by hand drifts; real projects generate it (`cbindgen`) or generate the Rust side from the C header (`bindgen`). I did not run either, and they are third-party tools that I describe from memory.
- C evaluates function-call arguments in an unspecified order. In my first test program, `printf("%d (q=%d)", f(&q), q)` printed `q=0` because `q` was read before `f` ran ✓; call first, then print. This is a C lesson, but FFI tests hit it.

---

## 9. Build scripts and linking

A library that wraps C needs the C code compiled and linked. The simplest portable route is a **build script** using the `cc` crate: `build.rs` runs before your crate compiles and tells Cargo what to link.

```rust
// build.rs
fn main() {
    cc::Build::new()
        .file("c/ffilab.c")
        .warnings(true)
        .compile("ffilab_c");
    println!("cargo:rerun-if-changed=c/ffilab.c");
    println!("cargo:rustc-link-lib=m");
}
```

```toml
[package]
name = "ffilab"
version = "0.1.0"
edition = "2024"
build = "build.rs"

[build-dependencies]
cc = "1"
```

This was the setup used in §11 ✓ (the crate resolved `cc 1.11.0` here). Always emit `rerun-if-changed`, otherwise Cargo reruns the script on every build. Alternatives, from memory: link a system library with `#[link(name = "z")]`, discover it with `pkg-config`, or use a `*-sys` crate convention (a `foo-sys` crate with the raw bindings, and a `foo` crate with the safe API).

---

## 10. Performance, reliability, security

- **Crossing cost.** A call into C is an ordinary function call (no marshalling in Rust itself), so small calls are cheap; the costs are the copies and conversions you add (`CString` allocation, UTF-8 validation). I did not benchmark any of it; measure before optimizing. Batching work per call beats many tiny calls.
- **Inlining.** C code cannot be inlined into Rust without cross-language LTO (not covered); a hot one-line C function may cost more than the same Rust function.
- **Uninitialized buffers.** `with_capacity` + `spare_capacity_mut` avoids zero-filling large buffers. The saving is real only when the buffer is large and the zeroing shows in a profile.
- **Reliability.** Every FFI error path must free what it allocated; RAII wrappers make that automatic. Count on C returning NULL, negative codes and "partial success", and map each into a `Result` (Lesson 26).
- **Security.** Rust does not make C safe. A wrapper over an unsafe C library is exactly as safe as the C library and your `SAFETY` arguments. Treat every number from C as untrusted before using it as a length or index (`fill` checks `written <= cap` before `set_len`), never pass user-controlled data as a format string, and bound every size you pass. Never claim "it's Rust, so it's secure" about a crate with unsafe or FFI code.
- **Panics, again.** A panic across a plain `extern "C"` aborts the process: a reliability issue for servers and a denial-of-service issue if input can trigger it. Catch at the boundary.

---

## 11. Common mistakes

1. **Assuming default struct layout is C layout.** Without `repr(C)`, fields may be reordered; add the const assertions (§2.2).
2. **Using `String`/`&str`/`Vec` in `extern "C"` signatures.** The compiler warns; use pointer + length or `CStr`.
3. **Dangling `CString`**: `CString::new(s).unwrap().as_ptr()` in one expression (debugging challenge).
4. **Interior NUL**: unwrapping `CString::new` on user input; handle `NulError`.
5. **Freeing with the wrong allocator**, or freeing twice, or using after free; wrap C ownership in `Drop`.
6. **`assume_init` too early**: before C has written, or on the error path.
7. **Trusting a length from C** for `set_len` or `from_raw_parts`.
8. **Unwinding into C**: letting a panic escape a callback; catch it.
9. **Enum from C**: declaring a Rust enum parameter for a C enum that may carry unknown values.
10. **Marking a function `safe fn`** in an `unsafe extern` block when it has pointer preconditions: you just wrote an unsound safe API.
11. **Forgetting thread-safety**: sharing a non-thread-safe C handle across threads; leave the wrapper `!Send`.
12. **`MaybeUninit::uninit().assume_init()`** for a type with invalid bit patterns (`bool`, references, `NonZero*`, enums): UB, even if you "write it later".

---

## 12. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Why is `repr(C)` required for a struct passed by pointer to a C function, and what does `repr(transparent)` guarantee?
2. **(Warm-up)** What are `size_of::<Option<&u8>>()` and `size_of::<Option<u32>>()` on this target, and why does the first matter for FFI?
3. **(Easy)** Why does `strlen` of `"héllo"` return 6 while `.chars().count()` is 5, and what must you do to a `&str` before passing it to C?
4. **(Medium)** Write `fn parse_int_out(rc_fn: unsafe extern "C" fn(*mut i32) -> i32) -> Option<i32>` using `MaybeUninit`. On which path may you call `assume_init`?
5. **(Medium)** Given `unsafe extern "C" { fn make() -> *mut u8; fn release(p: *mut u8); }` (a C allocation with its own free function), write an owning wrapper type `Handle`. Why is `Handle` automatically `!Send`?
6. **(Medium-hard)** An `extern "C"` callback calls `unwrap()` on an `Option` that is sometimes `None`. What happens, and what are two ways to make it correct?
7. **(Hard)** Why is `&mut (*p).field` wrong when `p: *mut Config` points to uninitialized memory, but `&raw mut (*p).field` fine?
8. **(Hard)** Explain why `Vec::with_capacity(n)` + C writing into the spare capacity + `set_len(n)` is sound only if C actually wrote all `n` bytes, and what you should do when C reports a count.

<details markdown="1">
<summary>Answers</summary>

1. The default layout is unspecified, so Rust may reorder fields and C would read the wrong offsets. `repr(C)` fixes order and padding to the C rules (✓ `{u8,u32,u8}`: size 12 and offsets 0, 4, 8, equal to what `gcc` reports). `repr(transparent)` on a one-field newtype guarantees it has the same layout *and ABI* as the field, so `Meters(f64)` can be passed where C expects a `double`.
2. ✓ `Option<&u8>` is 8 bytes (a pointer; `None` is the null pointer, the niche) and `Option<u32>` is 8 bytes (tag plus value). For FFI, `Option<&T>`, `Option<Box<T>>`, `Option<NonNull<T>>` and `Option<extern "C" fn(..)>` map exactly onto "nullable pointer", so a nullable C pointer can be received as `Option<&T>` without any conversion; a plain `&T` must never be null.
3. `strlen` counts bytes up to the NUL, and `é` is two bytes in UTF-8 (✓ 6 vs 5). A `&str` is not NUL-terminated and may contain NUL, so convert it with `CString::new` (handle the `NulError`) and keep the `CString` alive in a variable while C uses the pointer.
4. ```rust
   use std::mem::MaybeUninit;

   fn parse_int_out(rc_fn: unsafe extern "C" fn(*mut i32) -> i32) -> Option<i32> {
       let mut out = MaybeUninit::<i32>::uninit();
       // SAFETY: `out` is valid for one i32 write; the callee only writes it.
       let rc = unsafe { rc_fn(out.as_mut_ptr()) };
       if rc == 0 {
           // SAFETY: rc == 0 is documented to mean `*out` was written.
           Some(unsafe { out.assume_init() })
       } else {
           None
       }
   }

   extern "C" fn fake(out: *mut i32) -> i32 {
       unsafe { out.write(42) };
       0
   }

   fn main() {
       assert_eq!(parse_int_out(fake), Some(42));
   }
   ```
   Only on the success path (`rc == 0`). On the error path the value may be uninitialized; reading it would be UB.
5. ```rust
   use std::ptr::NonNull;

   mod sys {
       unsafe extern "C" {
           pub fn make() -> *mut u8;
           pub fn release(p: *mut u8);
       }
   }

   pub struct Handle(NonNull<u8>);

   impl Handle {
       pub fn new() -> Option<Handle> {
           // SAFETY: no preconditions; null (failure) is handled.
           NonNull::new(unsafe { sys::make() }).map(Handle)
       }
   }

   impl Drop for Handle {
       fn drop(&mut self) {
           // SAFETY: from `make`, released exactly once, here.
           unsafe { sys::release(self.0.as_ptr()) }
       }
   }
   ```
   (It needs those C symbols to link, so it only compiles as a library check.) `NonNull<u8>` contains a raw pointer, and raw pointers are neither `Send` nor `Sync`, so `Handle` is `!Send + !Sync` automatically: the safe default for a C handle with unknown thread-safety.
6. A panic cannot unwind out of a plain `extern "C"` function, so the process **aborts** (✓ "panic in a function that cannot unwind", status 134). Fix 1: handle `None` explicitly and return an error code. Fix 2: wrap the body in `catch_unwind`, convert the panic to a code (or store the payload and `resume_unwind` on the Rust side of the call). `extern "C-unwind"` is a third option only if the caller is prepared to unwind.
7. `&mut (*p).field` creates a *reference*, and a reference to uninitialized memory violates the validity rule for most types (and creates a `&mut` the compiler may assume points to a valid value). `&raw mut (*p).field` creates a raw pointer to the field without making a reference; you then `write` through it ✓ (the `Config` example in §3.1).
8. `set_len(n)` tells the `Vec` its first `n` elements are initialized. If C wrote only `k < n` bytes, bytes `k..n` are uninitialized and reading them is UB. Hence: ask C how many bytes it wrote, check `written <= capacity` (a buggy or malicious C side could report more; `fill` returns `FillError::Overrun`), and `set_len(written)`.

</details>

---

## 13. Debugging challenge

A function asks C how long a string is. The author's test prints the right answer, and the compiler said something, but only as a warning.

```rust
use std::ffi::{CString, c_char};

unsafe extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    let p = CString::new("hello").unwrap().as_ptr();
    let n = unsafe { strlen(p) };
    println!("len = {n}");
}
```

✓ It printed `len = 5`. The compiler emitted `warning: this creates a dangling pointer because temporary `CString` is dropped at end of statement` (lint `dangling_pointers_from_temporaries`), with the help "bind the `CString` to a variable such that it outlives the pointer returned by `as_ptr`". Under `valgrind` the same program reported `Invalid read of size 1 ... at strlen`. Explain, and fix.

<details markdown="1">
<summary>Solution</summary>

`CString::new("hello").unwrap()` is a temporary: the `CString` owns the heap buffer, and it is dropped at the end of the `let p = ...;` statement, freeing the buffer. `as_ptr()` returned a raw pointer into that buffer, so `p` dangles, and `strlen` reads freed memory: undefined behavior (use after free). It printed 5 only because freed memory usually still contains the old bytes; that is luck, and valgrind's `Invalid read` is the real verdict. Raw pointers have no lifetime, which is why this compiles at all; the lint is a heuristic for exactly this pattern.

Fix: give the owner a name so it outlives the call (✓ this version ran and printed `len = 5` with no warning):

```rust
use std::ffi::{CString, c_char};

unsafe extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    let c = CString::new("hello").unwrap();
    // SAFETY: `c` is NUL-terminated and lives until the end of main.
    let n = unsafe { strlen(c.as_ptr()) };
    println!("len = {n}");
}
```

A better API removes the foot-gun: wrap the call in a safe function that takes `&CStr`, so the borrow, not a raw pointer, links the string to the call.

</details>

---

## 14. Interview questions

1. *What does `repr(C)` do, and when would you not use it?*
2. *What is `MaybeUninit` for, and why not `mem::zeroed()` or `mem::uninitialized()`?*
3. *How do you pass a Rust closure to a C callback API?*
4. *What happens if a Rust panic crosses an `extern "C"` function? What is `"C-unwind"`?*
5. *How do you represent a nullable function pointer or nullable pointer in an FFI signature?*
6. *Why do you need `CString`, and what are its two failure modes?*
7. *You wrap a C handle in a Rust struct. What must `Drop`, `Send` and `Sync` do?*
8. *How would you review an FFI wrapper for soundness?*

<details markdown="1">
<summary>Answers</summary>

1. It fixes field order and padding to the platform C ABI rules, so a pointer to the struct can be shared with C and its layout can be relied on. Do not use it for internal types: the default layout lets the compiler reorder fields to save space (✓ 8 bytes vs 12 for the same fields), and `repr(C)` also blocks some niche optimizations.
2. `MaybeUninit<T>` is a `T`-sized, `T`-aligned slot that does not claim to hold a valid `T`, so you can pass its pointer to C or build a value piece by piece. `mem::zeroed` is wrong for types where all-zero is invalid (references, `NonZero`, enums without a 0 discriminant, `Box`), and `mem::uninitialized` (deprecated and removed) produced a value the compiler treats as initialized, which is UB for most types. Only `assume_init` after the memory is really written.
3. Pass `&mut closure` cast to `*mut c_void` as the `user` pointer, and a generic `extern "C" fn trampoline<F: FnMut(..)>` that casts it back and calls it. The closure must outlive the call, and panics must be caught inside the trampoline.
4. With the plain `"C"` ABI the panic cannot propagate and the process aborts (✓ exit 134, "panic in a function that cannot unwind"). `extern "C-unwind"` permits unwinding across the boundary, which is sound only if the foreign frames can unwind correctly (C++ exceptions, compiled with unwind support). Default to catching at the boundary.
5. `Option<extern "C" fn(..)>` and `Option<&T>`/`Option<NonNull<T>>`/`Option<Box<T>>`, whose `None` is NULL by the niche rule (✓ same size as a pointer). A plain `fn`/`&T` is never null.
6. A `CString` guarantees the trailing NUL and no interior NUL. Failure modes: `CString::new` fails with `NulError` if the input contains `\0` (✓ `nul_position`), and the pointer from `as_ptr` dangles if the `CString` is dropped too early (✓ valgrind). Also: C strings are not necessarily UTF-8, so `CStr::to_str` can fail (✓).
7. `Drop` must free the resource exactly once with the correct deallocator and never touch it afterwards. A raw-pointer field makes the type `!Send + !Sync` by default; keep that unless the C library documents thread-safety, and then add `unsafe impl Send/Sync` with a comment citing it. Expose only `&mut self` methods if the C API is not reentrant.
8. For each `unsafe` block, check the preconditions of the C function (null, alignment, lifetime, aliasing, thread-safety, length) and where each is established; check that ownership is released exactly once on all paths, including errors and panics; check layout with const assertions and a C-side test; check that no panic can cross the boundary and that any number from C is validated before use; run the tests under valgrind or a sanitizer (and Miri for the Rust-only parts, Lesson 38).

</details>

---

## 15. Practical challenge: `ffilab`

Build a library crate `ffilab` that wraps a small C library with a **safe** API. Requirements:

1. A C file `c/ffilab.c`, compiled by `build.rs` with the `cc` crate, providing: a `point_t {int32_t x, y}` with a distance function taking structs **by value**, a scaling function taking `point_t *`, a constructor with an out-parameter that fails on negative coordinates and leaves `out` untouched; `sizeof`/`offsetof` helpers for a layout test; a version string; a `malloc`'d greeting plus its free function; an opaque counter handle; a `for_each` taking a callback and a user pointer; a function writing up to 64 bytes into a caller buffer and reporting the count; and a function that calls back into a **Rust-exported** function.
2. The Rust API: `Point` (`repr(C)`, const layout assertions), `greeting` (typed error enum), `Counter` (RAII, `!Send`), `for_each` (any `FnMut`, panics never unwind through C), `fill` (spare capacity + validated `set_len`), `sort_i32` through libc `qsort`, `version`, `apply_rust` and an exported `ffilab_rust_double`, and a panic-safe `array_from_fn_lite` built on `MaybeUninit`.
3. Every `unsafe` block has a `// SAFETY:` comment; `#![deny(missing_docs)]`, `#![deny(unsafe_op_in_unsafe_fn)]`; clippy with `-D warnings`.
4. Tests: layout against C, edge cases, closure state, panic stops iteration and is re-raised, drop counts for the array builder on success and panic.
5. Run the test binary under `valgrind`.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`:

```toml
[package]
name = "ffilab"
version = "0.1.0"
edition = "2024"
build = "build.rs"

[dependencies]

[build-dependencies]
cc = "1"
```

`build.rs`:

```rust
fn main() {
    println!("cargo:rerun-if-changed=c/ffilab.c");
    cc::Build::new()
        .file("c/ffilab.c")
        .warnings(true)
        .compile("ffilab_c");
    println!("cargo:rustc-link-lib=m");
}
```

`c/ffilab.c`:

```c
#include <math.h>
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int32_t x;
    int32_t y;
} point_t;

double c_dist(point_t a, point_t b) {
    double dx = (double)a.x - (double)b.x;
    double dy = (double)a.y - (double)b.y;
    return sqrt(dx * dx + dy * dy);
}

void c_scale(point_t *p, int32_t k) {
    p->x *= k;
    p->y *= k;
}

/* Returns 0 and fills *out, or -1 (and leaves *out untouched) if a coordinate is negative. */
int32_t c_make_point(int32_t x, int32_t y, point_t *out) {
    if (x < 0 || y < 0) {
        return -1;
    }
    out->x = x;
    out->y = y;
    return 0;
}

size_t c_sizeof_point(void) { return sizeof(point_t); }
size_t c_offsetof_y(void) { return offsetof(point_t, y); }

const char *c_version(void) { return "ffilab-c 1.0"; }

/* Returns a malloc'd "Hello, <name>!" or NULL if name is NULL/empty or allocation fails.
   The caller must release it with c_free_string. */
char *c_make_greeting(const char *name) {
    if (name == NULL || name[0] == '\0') {
        return NULL;
    }
    size_t n = strlen(name) + sizeof("Hello, !");
    char *s = malloc(n);
    if (s == NULL) {
        return NULL;
    }
    strcpy(s, "Hello, ");
    strcat(s, name);
    strcat(s, "!");
    return s;
}

void c_free_string(char *s) { free(s); }

typedef struct counter {
    int32_t next;
} counter_t;

counter_t *counter_new(int32_t start) {
    if (start < 0) {
        return NULL;
    }
    counter_t *c = malloc(sizeof *c);
    if (c != NULL) {
        c->next = start;
    }
    return c;
}

int32_t counter_next(counter_t *c) { return c->next++; }
void counter_free(counter_t *c) { free(c); }

/* Calls cb for each element; stops early if cb returns non-zero. Returns the number of calls made. */
size_t c_for_each(const int32_t *xs, size_t n, int (*cb)(int32_t, void *), void *user) {
    size_t calls = 0;
    for (size_t i = 0; i < n; i++) {
        calls++;
        if (cb(xs[i], user) != 0) {
            break;
        }
    }
    return calls;
}

/* Writes min(cap, 64) bytes of the pattern i % 251 and stores the count in *written.
   Returns -1 if cap == 0. */
int32_t c_fill(uint8_t *buf, size_t cap, size_t *written) {
    if (cap == 0) {
        return -1;
    }
    size_t n = cap < 64 ? cap : 64;
    for (size_t i = 0; i < n; i++) {
        buf[i] = (uint8_t)(i % 251);
    }
    *written = n;
    return 0;
}

/* Implemented in Rust (see src/lib.rs). */
extern int32_t ffilab_rust_double(int32_t x);

int32_t c_apply_rust(int32_t x) { return ffilab_rust_double(x) + 1; }
```

`src/lib.rs`:

```rust
//! `ffilab`: safe Rust wrappers around a small C library (`c/ffilab.c`,
//! compiled by `build.rs`), plus a pure-Rust `MaybeUninit` exercise.
//!
//! Every wrapper follows the same recipe: convert and validate Rust values in
//! safe code, make exactly one unsafe call with a `// SAFETY:` comment, and
//! turn the C result (null, error code, out-parameter) into `Option`/`Result`.
//!
//! ```
//! use ffilab::{greeting, Counter, Point};
//! assert_eq!(greeting("Ada").unwrap(), "Hello, Ada!");
//! let mut c = Counter::new(5).unwrap();
//! assert_eq!((c.next_value(), c.next_value()), (5, 6));
//! assert_eq!(Point::new(3, 4).unwrap().dist(Point::new(0, 0).unwrap()), 5.0);
//! ```
//!
//! A counter owns a C allocation and is tied to the thread that made it:
//!
//! ```compile_fail,E0277
//! fn needs_send<T: Send>(_: T) {}
//! needs_send(ffilab::Counter::new(0).unwrap());
//! ```

#![deny(missing_docs)]
#![deny(unsafe_op_in_unsafe_fn)]
#![warn(clippy::undocumented_unsafe_blocks)]

use std::any::Any;
use std::ffi::{CStr, CString, c_char, c_int, c_void};
use std::fmt;
use std::marker::PhantomData;
use std::mem::{self, MaybeUninit};
use std::panic::{AssertUnwindSafe, catch_unwind, resume_unwind};
use std::ptr::NonNull;

/// The raw declarations. Private: callers only see the safe wrappers.
mod sys {
    use super::Point;
    use std::ffi::{c_char, c_int, c_void};

    /// Opaque C type: only ever used behind a pointer.
    #[repr(C)]
    pub struct counter_t {
        _private: [u8; 0],
    }

    unsafe extern "C" {
        pub fn c_dist(a: Point, b: Point) -> f64;
        pub fn c_scale(p: *mut Point, k: i32);
        pub fn c_make_point(x: i32, y: i32, out: *mut Point) -> i32;
        pub fn c_sizeof_point() -> usize;
        pub fn c_offsetof_y() -> usize;
        pub fn c_version() -> *const c_char;
        pub fn c_make_greeting(name: *const c_char) -> *mut c_char;
        pub fn c_free_string(s: *mut c_char);
        pub fn counter_new(start: i32) -> *mut counter_t;
        pub fn counter_next(c: *mut counter_t) -> i32;
        pub fn counter_free(c: *mut counter_t);
        pub fn c_for_each(
            xs: *const i32,
            n: usize,
            cb: extern "C" fn(i32, *mut c_void) -> c_int,
            user: *mut c_void,
        ) -> usize;
        pub fn c_fill(buf: *mut u8, cap: usize, written: *mut usize) -> i32;
        pub fn c_apply_rust(x: i32) -> i32;
        pub fn qsort(
            base: *mut c_void,
            n: usize,
            size: usize,
            cmp: extern "C" fn(*const c_void, *const c_void) -> c_int,
        );
    }
}

/// A point with the same layout as the C `point_t`. Coordinates are
/// non-negative (the C constructor rejects negative ones).
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(C)]
pub struct Point {
    x: i32,
    y: i32,
}

// If the Rust definition ever drifts from the C one, fail the build, not a run.
const _: () = {
    assert!(size_of::<Point>() == 8);
    assert!(align_of::<Point>() == 4);
    assert!(mem::offset_of!(Point, x) == 0);
    assert!(mem::offset_of!(Point, y) == 4);
};

impl Point {
    /// Creates a point through the C constructor; `None` if a coordinate is negative.
    pub fn new(x: i32, y: i32) -> Option<Point> {
        let mut out = MaybeUninit::<Point>::uninit();
        // SAFETY: `out` is valid for writing one `Point` (same layout as the
        // C struct, checked above); on a zero return C has written both fields.
        let rc = unsafe { sys::c_make_point(x, y, out.as_mut_ptr()) };
        if rc != 0 {
            return None;
        }
        // SAFETY: rc == 0 means C initialized `*out` completely, and every
        // bit pattern of two `i32`s is a valid `Point`.
        Some(unsafe { out.assume_init() })
    }

    /// The x coordinate.
    pub fn x(&self) -> i32 {
        self.x
    }

    /// The y coordinate.
    pub fn y(&self) -> i32 {
        self.y
    }

    /// Euclidean distance, computed in C (the struct is passed by value).
    pub fn dist(self, other: Point) -> f64 {
        // SAFETY: plain by-value call with `repr(C)` arguments; no pointers.
        unsafe { sys::c_dist(self, other) }
    }

    /// Multiplies both coordinates by `k` in C.
    ///
    /// Wraps on overflow in release and panics in debug if the C code is
    /// compiled with trapping arithmetic; C signed overflow is undefined, so
    /// `k` is limited to results that fit.
    pub fn scale(&mut self, k: i32) -> Result<(), RangeError> {
        let fits = |v: i32| v.checked_mul(k).is_some();
        if !(fits(self.x) && fits(self.y)) {
            return Err(RangeError);
        }
        // SAFETY: `self` is a valid exclusive `Point`; the multiplication was
        // checked above to not overflow, so the C code has no signed overflow.
        unsafe { sys::c_scale(self, k) };
        Ok(())
    }
}

/// A scaled coordinate would overflow `i32`.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct RangeError;

impl fmt::Display for RangeError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str("scaled coordinate does not fit in i32")
    }
}

impl std::error::Error for RangeError {}

/// The size and the offset of `y` of the C `point_t`, as reported by C.
pub fn c_point_layout() -> (usize, usize) {
    // SAFETY: argument-free functions that only evaluate `sizeof`/`offsetof`.
    unsafe { (sys::c_sizeof_point(), sys::c_offsetof_y()) }
}

/// The version string of the C library.
pub fn version() -> &'static str {
    // SAFETY: `c_version` returns a pointer to a string literal inside the C
    // library: valid, NUL-terminated and immutable for the whole program, so
    // a `'static` borrow is sound.
    let s = unsafe { CStr::from_ptr(sys::c_version()) };
    s.to_str().unwrap_or("<invalid utf-8>")
}

/// Why [`greeting`] failed.
#[derive(Debug, Clone, PartialEq, Eq)]
#[non_exhaustive]
pub enum GreetError {
    /// The name contains a NUL byte at this position (C strings end there).
    InteriorNul {
        /// Index of the first NUL byte.
        position: usize,
    },
    /// The C library returned NULL (an empty name or an allocation failure).
    Rejected,
    /// The C library returned bytes that are not UTF-8.
    InvalidUtf8,
}

impl fmt::Display for GreetError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            GreetError::InteriorNul { position } => {
                write!(f, "name contains a NUL byte at position {position}")
            }
            GreetError::Rejected => f.write_str("the C library rejected the name"),
            GreetError::InvalidUtf8 => f.write_str("the C library returned invalid UTF-8"),
        }
    }
}

impl std::error::Error for GreetError {}

/// A string allocated by the C library, freed with the C library's `free`.
struct CAllocated(NonNull<c_char>);

impl CAllocated {
    fn new(raw: *mut c_char) -> Option<CAllocated> {
        NonNull::new(raw).map(CAllocated)
    }

    fn as_cstr(&self) -> &CStr {
        // SAFETY: the pointer came from `c_make_greeting`, which returns a
        // NUL-terminated string that stays valid until `c_free_string`, which
        // only runs in `drop`; the borrow is tied to `&self`.
        unsafe { CStr::from_ptr(self.0.as_ptr()) }
    }
}

impl Drop for CAllocated {
    fn drop(&mut self) {
        // SAFETY: the pointer was returned by `c_make_greeting`, is freed
        // exactly once (here), and is not used afterwards.
        unsafe { sys::c_free_string(self.0.as_ptr()) }
    }
}

/// Builds `"Hello, <name>!"` in C and copies it into a Rust `String`.
pub fn greeting(name: &str) -> Result<String, GreetError> {
    let c_name = CString::new(name).map_err(|e| GreetError::InteriorNul {
        position: e.nul_position(),
    })?;
    // SAFETY: `c_name` is a valid NUL-terminated string that outlives the call.
    let raw = unsafe { sys::c_make_greeting(c_name.as_ptr()) };
    let owned = CAllocated::new(raw).ok_or(GreetError::Rejected)?;
    owned
        .as_cstr()
        .to_str()
        .map(str::to_owned)
        .map_err(|_| GreetError::InvalidUtf8)
}

/// A counter living in C memory. It is neither `Send` nor `Sync`: nothing
/// promises that the C code may be used from another thread.
pub struct Counter {
    raw: NonNull<sys::counter_t>,
    _not_send_sync: PhantomData<*mut ()>,
}

impl Counter {
    /// Starts counting at `start`; `None` if `start` is negative (C refuses)
    /// or allocation fails.
    pub fn new(start: i32) -> Option<Counter> {
        // SAFETY: plain by-value call; a null result is handled below.
        let raw = unsafe { sys::counter_new(start) };
        NonNull::new(raw).map(|raw| Counter {
            raw,
            _not_send_sync: PhantomData,
        })
    }

    /// Returns the current value and advances.
    pub fn next_value(&mut self) -> i32 {
        // SAFETY: `raw` came from `counter_new`, is not freed before `drop`,
        // and `&mut self` guarantees exclusive access.
        unsafe { sys::counter_next(self.raw.as_ptr()) }
    }
}

impl Drop for Counter {
    fn drop(&mut self) {
        // SAFETY: `raw` is freed exactly once, here.
        unsafe { sys::counter_free(self.raw.as_ptr()) }
    }
}

struct ForEachState<F> {
    f: F,
    panic: Option<Box<dyn Any + Send>>,
}

extern "C" fn for_each_trampoline<F: FnMut(i32)>(x: i32, user: *mut c_void) -> c_int {
    // SAFETY: `user` is the `&mut ForEachState<F>` that `for_each` passed to
    // C; C hands it back unchanged, and it is alive and unaliased for the
    // duration of the call.
    let state = unsafe { &mut *user.cast::<ForEachState<F>>() };
    // A panic must not unwind into C: catch it, remember it, ask C to stop.
    match catch_unwind(AssertUnwindSafe(|| (state.f)(x))) {
        Ok(()) => 0,
        Err(payload) => {
            state.panic = Some(payload);
            1
        }
    }
}

/// Calls `f` for every element, through C. If `f` panics, iteration stops
/// and the panic is re-raised after C has returned. Returns the number of
/// calls made.
pub fn for_each<F: FnMut(i32)>(xs: &[i32], f: F) -> usize {
    let mut state = ForEachState { f, panic: None };
    // SAFETY: `xs` is valid for `xs.len()` reads; the callback is the
    // matching instantiation of the trampoline; `state` outlives the call and
    // is only accessed through the pointer while C runs.
    let calls = unsafe {
        sys::c_for_each(
            xs.as_ptr(),
            xs.len(),
            for_each_trampoline::<F>,
            (&raw mut state).cast::<c_void>(),
        )
    };
    if let Some(payload) = state.panic {
        resume_unwind(payload);
    }
    calls
}

/// Why [`fill`] failed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[non_exhaustive]
pub enum FillError {
    /// A capacity of zero was requested.
    Empty,
    /// C reported writing more bytes than the buffer holds (a bug in C).
    Overrun {
        /// What C claims to have written.
        written: usize,
        /// The capacity offered.
        cap: usize,
    },
}

impl fmt::Display for FillError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            FillError::Empty => f.write_str("capacity must be positive"),
            FillError::Overrun { written, cap } => {
                write!(f, "C reported {written} bytes written into {cap}")
            }
        }
    }
}

impl std::error::Error for FillError {}

/// Lets C write up to `cap` bytes straight into a `Vec`'s spare capacity.
pub fn fill(cap: usize) -> Result<Vec<u8>, FillError> {
    let mut v: Vec<u8> = Vec::with_capacity(cap);
    let mut written = 0usize;
    // SAFETY: `spare_capacity_mut` has room for at least `cap` bytes, which C
    // may write but never read; `written` is a valid `usize` out-parameter.
    let rc = unsafe {
        sys::c_fill(
            v.spare_capacity_mut().as_mut_ptr().cast::<u8>(),
            cap,
            &mut written,
        )
    };
    if rc != 0 {
        return Err(FillError::Empty);
    }
    if written > cap {
        // Never trust a count from C when it feeds `set_len`.
        return Err(FillError::Overrun { written, cap });
    }
    // SAFETY: C initialized the first `written` bytes (it reported success),
    // and `written <= cap <= capacity`.
    unsafe { v.set_len(written) };
    Ok(v)
}

extern "C" fn cmp_i32(a: *const c_void, b: *const c_void) -> c_int {
    // SAFETY: `qsort` calls the comparator with pointers to two live,
    // aligned `i32` elements of the array being sorted.
    let (a, b) = unsafe { (*a.cast::<i32>(), *b.cast::<i32>()) };
    a.cmp(&b) as c_int
}

/// Sorts with the C library's `qsort` and a Rust comparator.
pub fn sort_i32(xs: &mut [i32]) {
    if xs.len() < 2 {
        return;
    }
    // SAFETY: `xs` is a valid array of `len` elements of `size_of::<i32>()`
    // bytes; the comparator matches the element type and cannot panic.
    unsafe {
        sys::qsort(
            xs.as_mut_ptr().cast::<c_void>(),
            xs.len(),
            size_of::<i32>(),
            cmp_i32,
        );
    }
}

/// A Rust function exported to C: `c_apply_rust` calls it.
#[unsafe(no_mangle)]
pub extern "C" fn ffilab_rust_double(x: i32) -> i32 {
    x.wrapping_mul(2)
}

/// Calls C, which calls back into [`ffilab_rust_double`] and adds one.
pub fn apply_rust(x: i32) -> i32 {
    // SAFETY: by-value call; the C function calls `ffilab_rust_double`, which
    // is defined above with a matching signature.
    unsafe { sys::c_apply_rust(x) }
}

/// Builds `[f(0), f(1), ..., f(N - 1)]` without requiring `T: Default`.
///
/// If `f` panics, the elements created so far are dropped exactly once.
pub fn array_from_fn_lite<T, const N: usize>(mut f: impl FnMut(usize) -> T) -> [T; N] {
    struct Guard<'a, T> {
        slots: &'a mut [MaybeUninit<T>],
        initialized: usize,
    }

    impl<T> Drop for Guard<'_, T> {
        fn drop(&mut self) {
            for slot in &mut self.slots[..self.initialized] {
                // SAFETY: exactly the first `initialized` slots were written
                // and none of them has been dropped or moved out.
                unsafe { slot.assume_init_drop() };
            }
        }
    }

    let mut arr: [MaybeUninit<T>; N] = [const { MaybeUninit::uninit() }; N];
    let mut guard = Guard {
        slots: &mut arr,
        initialized: 0,
    };
    for i in 0..N {
        let value = f(i);
        guard.slots[i].write(value);
        guard.initialized += 1;
    }
    // Everything is initialized: disarm the guard so it does not drop them.
    mem::forget(guard);
    // SAFETY: all `N` slots were written above, `MaybeUninit<T>` has the same
    // layout as `T`, and `arr` has no destructor, so reading the array out
    // moves the values exactly once.
    unsafe { std::ptr::read(arr.as_ptr().cast::<[T; N]>()) }
}
```

`tests/api.rs`:

```rust
//! Tests through the public API: the C library, the callbacks in both
//! directions, ownership and layout.

use ffilab::{
    Counter, FillError, GreetError, Point, RangeError, apply_rust, array_from_fn_lite,
    c_point_layout, fill, for_each, greeting, sort_i32, version,
};
use std::cell::Cell;
use std::panic::{AssertUnwindSafe, catch_unwind};
use std::rc::Rc;

#[test]
fn layout_matches_the_c_struct() {
    assert_eq!(c_point_layout(), (size_of::<Point>(), 4));
}

#[test]
fn point_constructor_validates_in_c() {
    let p = Point::new(3, 4).unwrap();
    assert_eq!((p.x(), p.y()), (3, 4));
    assert_eq!(Point::new(0, 0).map(|p| (p.x(), p.y())), Some((0, 0)));
    assert!(Point::new(-1, 0).is_none());
    assert!(Point::new(0, -1).is_none());
    assert_eq!(Point::new(1, 2), Point::new(1, 2));
}

#[test]
fn point_distance_is_computed_with_by_value_structs() {
    let o = Point::new(0, 0).unwrap();
    assert_eq!(Point::new(3, 4).unwrap().dist(o), 5.0);
    assert_eq!(o.dist(Point::new(3, 4).unwrap()), 5.0);
    assert_eq!(o.dist(o), 0.0);
    let far = Point::new(i32::MAX, 0).unwrap();
    assert_eq!(far.dist(o), f64::from(i32::MAX));
}

#[test]
fn point_scale_updates_through_the_pointer_and_checks_overflow() {
    let mut p = Point::new(2, 3).unwrap();
    p.scale(4).unwrap();
    assert_eq!((p.x(), p.y()), (8, 12));
    let mut q = Point::new(i32::MAX, 1).unwrap();
    assert_eq!(q.scale(2), Err(RangeError));
    assert_eq!((q.x(), q.y()), (i32::MAX, 1), "unchanged on error");
    assert_eq!(q.scale(1), Ok(()));
    assert!(RangeError.to_string().contains("i32"));
}

#[test]
fn version_is_a_static_utf8_string() {
    let v: &'static str = version();
    assert_eq!(v, "ffilab-c 1.0");
}

#[test]
fn greeting_round_trips_through_c_memory() {
    assert_eq!(greeting("Ada").unwrap(), "Hello, Ada!");
    assert_eq!(greeting("héllo ✓").unwrap(), "Hello, héllo ✓!");
    assert_eq!(greeting(&"x".repeat(10_000)).unwrap().len(), 10_000 + 8);
    for _ in 0..1000 {
        greeting("loop").unwrap();
    }
}

#[test]
fn greeting_errors() {
    assert_eq!(
        greeting("a\0b"),
        Err(GreetError::InteriorNul { position: 1 })
    );
    assert_eq!(greeting(""), Err(GreetError::Rejected));
    assert_eq!(
        greeting("a\0b").unwrap_err().to_string(),
        "name contains a NUL byte at position 1"
    );
    assert!(greeting("\0").is_err());
}

#[test]
fn counter_counts_and_rejects_negative_start() {
    let mut c = Counter::new(5).unwrap();
    assert_eq!((c.next_value(), c.next_value(), c.next_value()), (5, 6, 7));
    assert!(Counter::new(-1).is_none());
    let mut z = Counter::new(0).unwrap();
    assert_eq!(z.next_value(), 0);
    let many: Vec<Counter> = (0..100).filter_map(Counter::new).collect();
    assert_eq!(many.len(), 100);
}

#[test]
fn for_each_visits_in_order_with_a_stateful_closure() {
    let mut seen = Vec::new();
    let calls = for_each(&[3, 1, 4, 1, 5], |x| seen.push(x));
    assert_eq!(calls, 5);
    assert_eq!(seen, [3, 1, 4, 1, 5]);
    assert_eq!(for_each(&[], |_| unreachable!()), 0);
    let mut sum = 0;
    for_each(&[1, 2, 3], |x| sum += x);
    assert_eq!(sum, 6);
}

#[test]
fn for_each_panics_do_not_cross_the_c_frames_and_stop_iteration() {
    let visited = Cell::new(0);
    let r = catch_unwind(AssertUnwindSafe(|| {
        for_each(&[1, 2, 3, 4], |x| {
            visited.set(visited.get() + 1);
            if x == 2 {
                panic!("stop at {x}");
            }
        })
    }));
    let payload = r.unwrap_err();
    assert_eq!(visited.get(), 2, "C stopped after the panicking call");
    let msg = payload.downcast_ref::<String>().unwrap();
    assert_eq!(msg, "stop at 2");
}

#[test]
fn fill_lets_c_write_into_spare_capacity() {
    assert_eq!(fill(0), Err(FillError::Empty));
    assert_eq!(fill(1).unwrap(), [0]);
    assert_eq!(fill(5).unwrap(), [0, 1, 2, 3, 4]);
    let v = fill(64).unwrap();
    assert_eq!(v.len(), 64);
    assert!(v.iter().enumerate().all(|(i, &b)| b == (i % 251) as u8));
    assert_eq!(fill(1000).unwrap().len(), 64, "C caps at 64");
    assert_eq!(FillError::Empty.to_string(), "capacity must be positive");
}

#[test]
fn sort_matches_std_sort() {
    let mut s = 12345u64;
    for len in 0..40 {
        let mut a: Vec<i32> = (0..len)
            .map(|_| {
                s = s.wrapping_mul(6364136223846793005).wrapping_add(1);
                (s >> 40) as i32 - 5000
            })
            .collect();
        let mut b = a.clone();
        sort_i32(&mut a);
        b.sort();
        assert_eq!(a, b);
    }
    let mut extremes = [i32::MAX, i32::MIN, 0, -1, 1];
    sort_i32(&mut extremes);
    assert_eq!(extremes, [i32::MIN, -1, 0, 1, i32::MAX]);
}

#[test]
fn rust_and_c_can_call_each_other() {
    assert_eq!(apply_rust(4), 9);
    assert_eq!(apply_rust(-3), -5);
    assert_eq!(apply_rust(i32::MAX), -2 + 1);
}

#[test]
fn array_from_fn_lite_builds_arrays() {
    let a: [usize; 5] = array_from_fn_lite(|i| i * i);
    assert_eq!(a, [0, 1, 4, 9, 16]);
    let e: [String; 0] = array_from_fn_lite(|_| unreachable!());
    assert!(e.is_empty());
    let z: [(); 4] = array_from_fn_lite(|_| ());
    assert_eq!(z.len(), 4);
    let s: [String; 3] = array_from_fn_lite(|i| format!("s{i}"));
    assert_eq!(s, ["s0", "s1", "s2"]);
}

struct Tracked(Rc<Cell<usize>>);
impl Drop for Tracked {
    fn drop(&mut self) {
        self.0.set(self.0.get() + 1);
    }
}

#[test]
fn array_from_fn_lite_drops_exactly_the_initialized_prefix_on_panic() {
    let drops = Rc::new(Cell::new(0));
    let r = catch_unwind(AssertUnwindSafe(|| {
        let _: [Tracked; 5] = array_from_fn_lite(|i| {
            if i == 3 {
                panic!("fail at 3");
            }
            Tracked(drops.clone())
        });
    }));
    assert!(r.is_err());
    assert_eq!(drops.get(), 3, "elements 0, 1, 2 dropped once each");
}

#[test]
fn array_from_fn_lite_does_not_double_drop_on_success() {
    let drops = Rc::new(Cell::new(0));
    {
        let arr: [Tracked; 4] = array_from_fn_lite(|_| Tracked(drops.clone()));
        assert_eq!(drops.get(), 0, "nothing dropped while building");
        assert_eq!(arr.len(), 4);
    }
    assert_eq!(drops.get(), 4);
    assert_eq!(Rc::strong_count(&drops), 1);
}
```

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, `cargo test --locked` and `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps` pass: 16 integration tests and 2 doctests (one `compile_fail,E0277` showing `Counter` is not `Send`). Under `valgrind --leak-check=full --errors-for-leak-kinds=definite` the integration binary reported no errors (exit status 0). Without that flag, valgrind reported a "possibly lost" 48-byte block that originates in the standard test harness's thread bookkeeping and mentions none of the crate's functions; I treated that as harness noise. **Not run:** Miri, AddressSanitizer for this crate, other targets, big-endian, or Windows.

**Mutation checks ✓** (each applied to a copy, with its own target directory; a "caught" mutant failed at least one test):

| # | Mutation | Result |
|---|---|---|
| M1 | `Point::new` returns `None` on success instead of failure (`rc == 0`) | **caught** |
| M2 | `scale` overflow check uses `||` instead of `&&` | **caught** |
| M3 | `fill`'s overrun check loosened to `written > cap + 1000` | **survived**: the C function never reports more than `cap`, so no test input can reach that branch. This is a defense against a *buggy C library*; testing it needs a fake C implementation |
| M4 | the array guard is `drop`ped instead of `mem::forget`-ed on success (double drop) | **caught** |
| M5 | the guard drops one fewer element on panic (leak) | **caught** |
| M6 | trampoline returns 0 after a panic (C keeps iterating) | **caught** |
| M7 | stored panic payload is never re-raised | **caught** |
| M8 | `qsort` comparator reversed | **caught** |
| M9 | `ffilab_rust_double` multiplies by 3 | **caught** |
| M10 | `InteriorNul` always reports position 0 | **caught** |
| M11 | `fill` calls `set_len(cap)` instead of `set_len(written)` | **caught** |
| M12 | `sort_i32` early-return guard `len < 2` weakened to `len < 1` | **survived**: with 0 or 1 elements `qsort` does nothing observable, so the mutant is behaviorally equivalent (the guard just avoids a pointless FFI call) |
| M13 | `Counter::drop` no longer calls `counter_free` | **survived the test suite** (a leak is not an assertion failure), but **valgrind reported `definitely lost` at `counter_new`** ✓ for the mutant. This is why memory tools belong in the pipeline, not just tests |
| M14 | the guard's `initialized` counter is never incremented | **caught** |
| M15 | `fill`'s error check weakened to `rc > 0` (C returns `-1`) | **caught** |

Totals: 15 mutants, 12 caught by tests, 1 more caught by valgrind (M13), 2 survived (M3 unreachable without a faulty C, M12 equivalent). While running these I made two harness mistakes worth noting: my first "caught" detector matched the text `0 failed` in a *passing* run, and a first batch of mutants was recorded as compile errors because the detector matched `error:` in ordinary test output. I fixed the detector and reran everything; the table is from the rerun.

**Design notes.**

- The raw `extern` declarations live in a private `mod sys`, so a caller cannot reach an unsafe function; each safe function discharges the preconditions with a `SAFETY` comment (null handled, lifetimes tied to borrows).
- `Point::new` uses `MaybeUninit<Point>` as the out-parameter and calls `assume_init` only when C returned 0.
- `greeting` copies the C string into a `String` and frees the C memory through a private `CAllocated` guard, so every path (including a UTF-8 error) frees exactly once.
- `Counter` is `!Send + !Sync` through `NonNull` plus `PhantomData<*mut ()>`; the `compile_fail` doctest locks that in.
- `for_each` keeps the closure and the panic payload in one state struct. The trampoline catches the panic, returns non-zero (C stops), and the wrapper calls `resume_unwind` after C returns. `AssertUnwindSafe` is used because the state is dropped/re-raised right after, never observed in a half-updated state by other code (an argument you should re-check whenever you copy this pattern).
- `fill` never trusts `written`; `array_from_fn_lite` uses a guard so a panic at index 3 drops exactly three elements (tested), and `mem::forget`s the guard on success so nothing is dropped twice.
- Not handled: Windows/MSVC linkage, a C library that retains pointers after the call, thread-safe sharing of `Counter`, 32-bit targets, big-endian, a generated header or `bindgen`, and any benchmark. Layout was compared with the C compiler only on this one target.

</details>

---

## 16. Key takeaways

- Default Rust layout is unspecified; `repr(C)` is the contract with C. Assert size, alignment and offsets at compile time, and compare with the C compiler in a test (✓ 8 vs 12 bytes).
- `MaybeUninit<T>` is how uninitialized memory is represented: never read it, write through `write`/`&raw mut`, and call `assume_init` only on the path where initialization is guaranteed (✓). It never drops.
- `unsafe extern "C"` blocks (edition 2024), `safe fn` only for functions with no preconditions; use `std::ffi` types (`c_int`, `CStr`, `CString`, `c"..."`) and respect NUL rules (✓).
- Ownership is a convention in C: encode it as RAII wrappers (`Drop` calls the library's own free function), and keep wrappers `!Send` unless documented.
- Callbacks need a generic trampoline plus a `user` pointer; a panic must never cross a plain `extern "C"` frame (✓ abort 134); catch it and re-raise on the Rust side.
- Exported functions use `#[unsafe(no_mangle)] pub extern "C"`, FFI-safe types, error codes, and NULL checks (✓ a C program linked and ran).
- Validate every length or code returned by C; Rust's guarantees stop at the boundary. Use valgrind (✓ it caught a leak the tests did not) and, from Lesson 38, Miri.

**Connections.** [Lesson 36](36_unsafe_raw_pointers.md) supplies the contract style and raw-pointer rules used throughout; [Lesson 33](33_modules_api_design_rustdoc.md) and [Lesson 15](15_newtype_typestate.md) explain why the private `sys` module and wrapper types keep the API sound; [Lesson 22](22_gats_send_sync_phantom.md) explains `!Send` through `PhantomData`; [Lesson 26](26_error_design.md) shapes `GreetError` and `FillError`; [Lesson 03](03_cargo_projects.md) introduced Cargo features that build scripts extend. Next: Lesson 38 covers `no_std`, embedded basics and Miri.

---

## 17. References

- The Rustonomicon, chapters on FFI, "Working with Unsafe", uninitialized memory and unwinding: <https://doc.rust-lang.org/nomicon/>
- The Rust Reference, "Type layout" (`repr`), "External blocks", "Application binary interface" and "Panic" sections: <https://doc.rust-lang.org/reference/>
- `std::mem::MaybeUninit`, `std::ffi` (`CStr`, `CString`, `c_int`, ...), `std::panic::catch_unwind`, and `Vec::spare_capacity_mut` documentation: <https://doc.rust-lang.org/std/>
- The Edition Guide, "Rust 2024": `unsafe extern`, `unsafe` attributes: <https://doc.rust-lang.org/edition-guide/rust-2024/>
- The Cargo Book, "Build Scripts": <https://doc.rust-lang.org/cargo/reference/build-scripts.html>
- The `cc` crate (compiling C from `build.rs`): <https://docs.rs/cc>
- `bindgen` and `cbindgen` (generating bindings and headers; not run here): <https://rust-lang.github.io/rust-bindgen/> and <https://github.com/mozilla/cbindgen>
- RFC 2945 ("C-unwind ABI") for the design of `extern "C-unwind"`: <https://rust-lang.github.io/rfcs/2945-c-unwind-abi.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: that the Rust and C layouts match on targets other than this x86-64 Linux; the exact `repr` guarantees for `transparent`, `packed` and enums; that references to packed fields are an error/warning in the form described; the statement that a panic escaping `extern "C"` aborts "since Rust 1.81" (I observed the abort on 1.97 but not the version boundary); the semantics of `"C-unwind"` beyond my small Rust-to-Rust experiment; that `#[unsafe(no_mangle)]` is required in edition 2024; the claims about `bindgen`, `cbindgen`, `pkg-config`, `*-sys` conventions and cross-language LTO (not run); that `mem::uninitialized` was removed; and the whole of Miri's behavior (not installed). Exercise answers 5 and 8 combine ✓-run facts with reasoning from memory. Everything marked ✓ was run on this machine's `rustc 1.97.0`, with the C parts compiled by `gcc 13.3` and `valgrind` as installed in the sandbox.
