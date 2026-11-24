---
title: "Lesson 09 — Memory layout: size, alignment, padding, niches, and repr"
---

# Lesson 09 — Memory layout: size, alignment, padding, niches, and `repr`

**Module 3 · Ownership and memory** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. **Every size and offset below is for 64-bit x86-64 Linux with this compiler.** Where the language guarantees a layout fact I say *guaranteed*; where I only observed it I say *observed*, because the compiler is free to change unspecified layouts between releases.

Builds on [Lesson 08](08_stack_heap_drop.md) (stack, heap, fat pointers), [Lesson 07](07_ownership_and_moves.md) (a move copies `size_of::<T>()` bytes), and [Lesson 01](01_foundations.md) (niche optimization previewed).

---

## 1. Size and alignment

Every type `T` has two layout numbers, both known at compile time for sized types:

- **`size_of::<T>()`**: how many bytes a value occupies (including any padding inside it).
- **`align_of::<T>()`**: the address a value of `T` must be placed at must be a **multiple of this**. Alignment is always a power of two.

Always true: `size` is a multiple of `align`, so arrays of `T` can place elements back to back (`stride == size_of::<T>()`) with every element still aligned.

**Why alignment exists.** CPUs load and store naturally aligned values efficiently, and some architectures fault or are slow on misaligned access; atomics generally require alignment. On x86-64 plain loads of misaligned integers usually work, which is exactly why misalignment bugs hide until the code runs on another platform or the optimizer assumes alignment. Rust treats a misaligned **reference** as undefined behavior (Section 7), not merely slow.

```rust
use std::mem::{align_of, size_of};
trait T {}
macro_rules! row { ($($t:ty),* $(,)?) => { $( println!("{:<22} size {:>3}  align {:>2}", stringify!($t), size_of::<$t>(), align_of::<$t>()); )* } }
fn main() {
    row!(u8, i8, bool, u16, u32, f32, char, u64, f64, usize, u128, i128, ());
    row!(&u8, &str, &[u8], &dyn T, Box<u8>, Box<dyn T>, Box<[u8]>, *const u8, fn(u8));
    row!(String, Vec<u8>, [u8; 3], [u32; 3], (u8, u32), (u8, u16, u8));
}
```

✓ output:

```text
u8                     size   1  align  1
i8                     size   1  align  1
bool                   size   1  align  1
u16                    size   2  align  2
u32                    size   4  align  4
f32                    size   4  align  4
char                   size   4  align  4
u64                    size   8  align  8
f64                    size   8  align  8
usize                  size   8  align  8
u128                   size  16  align 16
i128                   size  16  align 16
()                     size   0  align  1
&u8                    size   8  align  8
&str                   size  16  align  8
&[u8]                  size  16  align  8
&dyn T                 size  16  align  8
Box<u8>                size   8  align  8
Box<dyn T>             size  16  align  8
Box<[u8]>              size  16  align  8
*const u8              size   8  align  8
fn(u8)                 size   8  align  8
String                 size  24  align  8
Vec<u8>                size  24  align  8
[u8; 3]                size   3  align  1
[u32; 3]               size  12  align  4
(u8, u32)              size   8  align  4
(u8, u16, u8)          size   4  align  2
```

Things to notice:

- Primitive sizes are fixed by the language (`u32` is 4 bytes everywhere). `usize`/`isize` and pointers are the target's pointer width. The alignment of **`u128` is 16 on this target** ✓ (I recall that older compilers used 8 on this target, which is a good reason not to hard-code it; I did not check the old behavior).
- **Thin pointer = 8 bytes, fat pointer = 16:** `&u8`, `Box<u8>` are one word. `&str`, `&[u8]`, `Box<[u8]>` carry a length; `&dyn Trait`, `Box<dyn Trait>` carry a pointer to a vtable (a table of function pointers and size/alignment info, Lesson 21).
- `()` and other zero-sized types have size 0 (Section 6).
- `String` and `Vec<u8>` are three words (pointer, capacity, length). The **order** of those fields is unspecified; never assume it.
- `(u8, u16, u8)` is 4 bytes, not 6: the compiler reordered the fields. That is the next section.

---

## 2. Padding and field reordering

If the compiler laid fields out in declaration order, a `u8` followed by a `u32` would force three **padding** bytes in between so the `u32` lands on a multiple of 4. Reordering fields so that big-alignment ones come first often removes padding.

```rust
use std::mem::{align_of, offset_of, size_of};
struct R { a: u8, b: u32, c: u8 }
#[repr(C)] struct C1 { a: u8, b: u32, c: u8 }
#[repr(C)] struct C2 { b: u32, a: u8, c: u8 }
fn main() {
    println!("repr(Rust) {{u8,u32,u8}}: size {} align {}", size_of::<R>(), align_of::<R>());
    println!("repr(C)    {{u8,u32,u8}}: size {} align {}", size_of::<C1>(), align_of::<C1>());
    println!("repr(C)    {{u32,u8,u8}}: size {} align {}", size_of::<C2>(), align_of::<C2>());
    println!("C1 offsets: a={} b={} c={}", offset_of!(C1, a), offset_of!(C1, b), offset_of!(C1, c));
    println!("C2 offsets: b={} a={} c={}", offset_of!(C2, b), offset_of!(C2, a), offset_of!(C2, c));
    println!("R offsets: a={} b={} c={}", offset_of!(R, a), offset_of!(R, b), offset_of!(R, c));
    let _ = R { a: 0, b: 0, c: 0 }.a;
}
```

✓ output:

```text
repr(Rust) {u8,u32,u8}: size 8 align 4
repr(C)    {u8,u32,u8}: size 12 align 4
repr(C)    {u32,u8,u8}: size 8 align 4
C1 offsets: a=0 b=4 c=8
C2 offsets: b=0 a=4 c=5
R offsets: a=4 b=0 c=5
```

Reading the `repr(C)` layout of `C1`: `a` at offset 0, 3 padding bytes (1–3), `b` at 4–7, `c` at 8, then 3 trailing padding bytes (9–11) so the total size, 12, is a multiple of the alignment 4. Trailing padding exists so that the *next* element of an array also starts aligned.

The default representation, `repr(Rust)`, makes **no promise about field order** ✓ (observed: `R` stored `b` first at offset 0, then `c` and `a`, for a total of 8). `std::mem::offset_of!` (✓ available on this toolchain; I believe it was stabilized in 1.77 but did not check the release notes) reports the offsets the compiler actually chose. Since `repr(Rust)` is allowed to change, **never assume offsets** for it.

**Design rationale.** C fixed struct layout to declaration order, which then became the de facto ABI on each platform. Rust deliberately left the default layout unspecified so the compiler can reorder fields to shrink types, place niches (Section 5), and evolve; when you need a stable layout (FFI, memory-mapped data), you ask for it with `repr`.

✓ A footprint comparison: a struct with a `bool`, a `u64`, and a `bool`:

```rust
use std::mem::size_of;
struct Wide { a: bool, b: u64, c: bool }
#[repr(C)] struct WideC { a: bool, b: u64, c: bool }
#[repr(C)] struct Sorted { b: u64, a: bool, c: bool }
fn main() {
    println!("Wide {} WideC {} Sorted(repr C, big first) {}", size_of::<Wide>(), size_of::<WideC>(), size_of::<Sorted>());
    println!("1000 Wide = {} bytes; 1000 WideC = {} bytes", 1000 * size_of::<Wide>(), 1000 * size_of::<WideC>());
}
```

✓ `Wide 16 WideC 24 Sorted 16`, so 1000 elements take 16 000 versus 24 000 bytes. If you use `repr(C)`, **sort fields by descending alignment** yourself. This is a memory-footprint measurement, not a speed claim; smaller elements mean more of them per cache line (a cache line is typically 64 bytes on x86-64), which tends to help scans, but measure your workload (Lesson 41).

---

## 3. The `repr` attributes

| Attribute | Meaning | Use for |
|---|---|---|
| (none) = `repr(Rust)` | Unspecified: the compiler may reorder fields, choose enum tags, use niches | Everything by default |
| `repr(C)` | Fields in declaration order with C's padding rules; enums use C-compatible layout | FFI (Lesson 37), shared memory, anything where another language reads the bytes |
| `repr(u8)`, `repr(i32)`, ... on a fieldless `enum` | Tag has that integer type, discriminants are what you write | Wire/file values, FFI enums |
| `repr(transparent)` | A struct with one non-zero-sized field has exactly that field's layout and ABI | Newtypes passed to FFI (Lesson 15) |
| `repr(align(N))` | Raise the minimum alignment to `N` (a power of two) | Cache-line alignment, SIMD, hardware requirements |
| `repr(packed)` / `repr(packed(N))` | Remove padding by lowering alignment to 1 (or `N`) | Matching packed binary layouts; **dangerous** |

```rust
use std::mem::{align_of, size_of};
#[repr(u8)] #[derive(Clone, Copy, Debug)] enum Kind { A = 1, B = 2, C = 200 }
enum Plain { X, Y, Z }
#[repr(transparent)] struct Meters(f64);
#[repr(align(64))] struct Aligned(u8);
#[repr(C, packed)] struct Packed { a: u8, b: u32 }
#[repr(C)] struct Unpacked { a: u8, b: u32 }

fn main() {
    println!("Kind {} Plain {}", size_of::<Kind>(), size_of::<Plain>());
    println!("{} {} {:?}", Kind::C as u8, Kind::B as i32, Kind::A);
    println!("Meters {} vs f64 {}; align {} vs {}", size_of::<Meters>(), size_of::<f64>(), align_of::<Meters>(), align_of::<f64>());
    println!("Aligned size {} align {}", size_of::<Aligned>(), align_of::<Aligned>());
    println!("Packed size {} align {}; Unpacked size {} align {}", size_of::<Packed>(), align_of::<Packed>(), size_of::<Unpacked>(), align_of::<Unpacked>());
    let p = Packed { a: 1, b: 0x01020304 };
    let b_copy = p.b;           // copying the field out is fine
    println!("{} {}", p.a, b_copy);
    let arr = [Aligned(0), Aligned(1)];
    println!("{}", size_of_val(&arr));
}
```

✓ output (plus dead-code warnings):

```text
Kind 1 Plain 1
200 2 A
Meters 8 vs f64 8; align 8 vs 8
Aligned size 64 align 64
Packed size 5 align 1; Unpacked size 8 align 4
1 16909060
128
```

- `repr(u8)` gives the enum a one-byte tag with the discriminants you chose; `Kind::C as u8` is `200`. A plain `enum` with ≤ 256 variants is also one byte ✓ (observed), but only with `repr` is the value *specified*.
- `repr(align(64))` made the type 64 bytes with alignment 64, so an array of two is 128 bytes: padding is how alignment is satisfied. Use it deliberately (for example to keep per-thread counters on separate cache lines and avoid *false sharing*, Lesson 42) and know it costs memory.
- `repr(packed)` removed all padding (5 bytes) and set alignment 1. The price: fields may be misaligned, so references to them are rejected:

```rust
#[repr(C, packed)] struct Packed { a: u8, b: u32 }
fn main() { let p = Packed { a: 1, b: 2 }; let r = &p.b; println!("{}", r); }
```

✓ `error[E0793]: reference to field of packed struct is unaligned`, with notes: "creating a misaligned reference is undefined behavior (even if that reference is never dereferenced)" and the help "copy the field contents to a local variable, or replace the reference with a raw pointer and use `read_unaligned`/`write_unaligned`". Copying `p.b` out, as above, is fine. (`println!("{}", p.b)` takes a reference internally, which is why the example copies first.)

Invalid combinations are compile errors, for example ✓ `#[repr(transparent)] struct Two(u8, u8);` gives `error[E0690]: transparent struct needs at most one field with non-trivial size or alignment, but has 2`, and ✓ `#[repr(u8)] struct S { a: u8 }` gives `error[E0517]: attribute should be applied to an enum`.

---

## 4. Enums: tag plus payload

An enum stores a **tag** (which variant) and enough space for the **largest** variant's payload, with alignment of the strictest variant. ✓ Observed:

```text
enum E1 { A, B, C }                          1   (tag only)
enum E2 { A(u8), B(u32) }                    8   (tag, padding, u32)
enum E3 { Small(u8), Large([u8; 100]) }    101   (tag + 100 bytes; align 1)
enum E4 { A(u32), B(u32) }                   8   (tag + u32; the tag cannot hide in the u32)
```

A `match` on an enum reads the tag and branches. Because size is determined by the largest variant, one huge variant makes *every* value huge (a reason to `Box` the rare large payload; Clippy's `large_enum_variant` lint flags this).

---

## 5. Niche optimization

A **niche** is a bit pattern that a type's valid values never use. The compiler can store *another* value's tag in a niche instead of adding a separate tag. The classic case is `Option<&T>`: a reference is never null, so `None` is represented as the all-zero pointer, and no extra byte is needed.

```rust
use std::mem::size_of;
use std::num::NonZeroU32;
macro_rules! row { ($($t:ty),* $(,)?) => { $( println!("{:<28} {:>3}", stringify!($t), size_of::<$t>()); )* } }
fn main() {
    row!(Option<u8>, Option<u32>, Option<u64>, Option<bool>, Option<char>, Option<Option<bool>>);
    row!(&u8, Option<&u8>, Box<u8>, Option<Box<u8>>, NonZeroU32, Option<NonZeroU32>);
    row!(String, Option<String>, Vec<u8>, Option<Vec<u8>>, Option<Option<String>>);
    row!(Result<u32, ()>, Result<(), String>, Result<u32, String>, Result<u8, u8>);
    row!(Option<fn()>, fn(), Option<std::cmp::Ordering>, std::cmp::Ordering);
}
```

✓ output:

```text
Option<u8>                     2
Option<u32>                    8
Option<u64>                   16
Option<bool>                   1
Option<char>                   4
Option<Option<bool>>           1
&u8                            8
Option<&u8>                    8
Box<u8>                        8
Option<Box<u8>>                8
NonZeroU32                     4
Option<NonZeroU32>             4
String                        24
Option<String>                24
Vec<u8>                       24
Option<Vec<u8>>               24
Option<Option<String>>        24
Result<u32, ()>                8
Result<(), String>            24
Result<u32, String>           24
Result<u8, u8>                 2
Option<fn()>                   8
fn()                           8
Option<std::cmp::Ordering>     1
std::cmp::Ordering             1
```

How to read it:

- `u8`, `u32`, `u64` use **every** bit pattern, so they have no niche: `Option<u32>` needs a separate tag (and padding to keep alignment): 8 bytes; `Option<u64>` doubles to 16.
- `bool` has 254 unused patterns, `char` has many (everything above `U+10FFFF` and the surrogate range), so `Option<bool>` is 1 byte and even `Option<Option<bool>>` is 1 byte. `Ordering` (three values) leaves room for `Option<Ordering>` in one byte.
- References, `Box`, function pointers, and `NonZero*` types have the "zero" niche. *Guaranteed* by the standard library's documentation of `Option`: `Option<&T>`, `Option<&mut T>`, `Option<Box<T>>`, `Option<NonZero*>`, and `Option<fn(..)>` have the same size as the wrapped type. This is why null pointers need no extra space in safe Rust and why `Option<NonZeroU32>` is a good "ID or absent" type.
- `Option<String>` and `Option<Vec<u8>>` are the same size as the unwrapped types ✓. That is **observed**, not guaranteed: it relies on the standard library marking part of `String`/`Vec` as never taking certain values.

A consequence for Lesson 07's cost model: moving an `Option<Box<T>>` is moving one pointer.

You can also pin layout facts at compile time with a const assertion, so a refactor or compiler upgrade can't silently break an assumption:

```rust
use std::mem::size_of;
#[repr(C)] struct Header { version: u8, kind: u8, length: u32 }
const _: () = assert!(size_of::<Header>() == 8);
const _: () = assert!(size_of::<Option<Box<u8>>>() == size_of::<Box<u8>>());
fn main() { println!("layout checks passed at compile time"); }
```

✓ compiles and runs. Changing the assertion for a `repr(Rust)` struct to a wrong value gives ✓ `error[E0080]: evaluation panicked: Header must be 6 bytes` with the message you wrote (for `const _: () = assert!(size_of::<Header>() == 6, "Header must be 6 bytes");` on a struct that is really 8 bytes).

---

## 6. Zero-sized types and dynamically sized types

**Zero-sized types (ZSTs):** `()`, unit structs, `[T; 0]`, `PhantomData<T>`, and structs of only ZSTs have size 0. They exist purely at the type level (markers, typestates, Lesson 15) and cost nothing.

```rust
use std::marker::PhantomData;
use std::mem::{size_of, size_of_val};
struct Unit;
struct Typed<T>(PhantomData<T>);
fn main() {
    println!("{} {} {} {}", size_of::<()>(), size_of::<Unit>(), size_of::<[u64; 0]>(), size_of::<Typed<String>>());
    let mut v: Vec<()> = Vec::new();
    for _ in 0..1000 { v.push(()); }
    println!("len {} capacity==usize::MAX {} size_of_val {}", v.len(), v.capacity() == usize::MAX, size_of_val(&v[..]));
    println!("{}", size_of_val(&[(); 100]));
}
```

✓ output: `0 0 0 0`, `len 1000 capacity==usize::MAX true size_of_val 0`, `0`. A `Vec<()>` of a thousand elements allocates nothing (✓ also `Box::new(())` allocated 0 times, Lesson 08).

**Dynamically sized types (DSTs):** `str`, `[T]`, and `dyn Trait` have no compile-time size. They exist only behind pointers, and the pointer carries the missing information (length or vtable): the fat pointers of Section 1. `size_of_val` asks for the size of the *pointee*:

```rust
use std::mem::{size_of, size_of_val};
trait Shape { fn area(&self) -> f64; }
struct Sq(f64);
struct Unit;
impl Shape for Sq { fn area(&self) -> f64 { self.0 * self.0 } }
impl Shape for Unit { fn area(&self) -> f64 { 1.0 } }
fn main() {
    let a: &dyn Shape = &Sq(2.0);
    let b: &dyn Shape = &Unit;
    println!("&dyn {} Box<dyn> {}", size_of::<&dyn Shape>(), size_of::<Box<dyn Shape>>());
    println!("pointee sizes via size_of_val: {} and {}", size_of_val(a), size_of_val(b));
    println!("areas {} {}", a.area(), b.area());
    let s: &str = "héllo";
    let sl: &[u16] = &[1, 2, 3];
    println!("{} {} {}", size_of_val(s), size_of_val(sl), size_of::<&str>());
}
```

✓ output: `&dyn 16 Box<dyn> 16`, `pointee sizes via size_of_val: 8 and 0`, `areas 4 1`, `6 6 16`. Both `&dyn Shape` values are 16 bytes though the pointees are 8 and 0 bytes; the vtable lets the program find out. `"héllo"` is 6 bytes (the `é` is 2) and `[1u16, 2, 3]` is 6.

---

## 7. Alignment in practice, and why you should not reinterpret bytes

Network and file data arrives as `&[u8]` at arbitrary offsets. Casting `&[u8]` bytes to `&u32` is wrong unless the address is a multiple of 4 and the bytes have the right endianness; **a misaligned reference is undefined behavior** (the compiler's note on E0793 above says so), and a `u32` view of bytes also depends on host byte order. The safe, portable way is to *build* the integer from bytes:

```rust
fn main() {
    let bytes: [u8; 8] = [1, 0, 0, 0, 2, 0, 0, 0];
    let a = u32::from_le_bytes(bytes[0..4].try_into().unwrap());
    let b = u32::from_le_bytes(bytes[4..8].try_into().unwrap());
    println!("{a} {b} {}", u32::from_be_bytes([0, 0, 1, 0]));
    let v: Vec<u8> = vec![0; 16];
    let base = v.as_ptr() as usize;
    println!("vec<u8> buffer address aligned to 4? {}", base % 4 == 0);
    println!("offset+1 aligned to 4? {}", (base + 1) % 4 == 0);
    println!("align_offset for u32 from base+1: {}", unsafe { v.as_ptr().add(1) }.align_offset(4) < 4);
}
```

✓ output: `1 2 256`, then `true`, `false`, `true`. (The first line is little-endian reads of `[1,0,0,0]` and `[2,0,0,0]` and a big-endian read of `[0,0,1,0]`.) A `Vec<u8>` buffer happened to be 4-aligned here, but `base + 1` is not; `u8` guarantees only alignment 1 for the *type*, so you may not rely on the buffer's address for anything wider. `unsafe` appears only for `ptr.add(1)`; Lesson 36 explains why.

`from_le_bytes`/`from_be_bytes`/`to_*_bytes` convert explicitly, handle endianness, need no unsafe, and typically compile to a plain load where that is valid (I did not inspect the assembly). Prefer them to `transmute` or pointer casts.

---

## 8. Common mistakes and edge cases

1. Using `size_of::<Struct>()` as the **serialized size** (see the debugging challenge).
2. Assuming `repr(Rust)` field order, offsets, or the size of `String`/`Vec` internals.
3. Using `repr(C)` for a "faster" layout: it only preserves declaration order, so unsorted fields can be larger than `repr(Rust)` (✓ 24 vs 16 bytes above).
4. Taking `&packed.field`, or casting a byte pointer to `*const u32` and dereferencing it without checking alignment.
5. Treating `align_of::<u128>()` or `usize` as target-independent.
6. Transmuting a struct to bytes (or the reverse): padding bytes are uninitialized, so reading them is undefined behavior and can leak memory contents.
7. Forgetting that `Option<T>` is bigger than `T` unless `T` has a niche (✓ `Option<u64>` is 16).
8. Putting a rarely used large variant in an enum and bloating every value.
9. Using `repr(align)` or `repr(packed)` without understanding the memory and correctness trade-offs.
10. Assuming ZSTs are exotic: `Vec<()>`, `HashMap<K, ()>`-style sets, and `PhantomData` are everyday uses.

## 9. Performance implications

- Smaller types mean more elements per cache line and less memory traffic. Reordering fields (let `repr(Rust)` do it, or sort descending by alignment under `repr(C)`) reduces size: ✓ 16 vs 24 bytes in the example. Whether that speeds up *your* program is a measurement question (Lessons 41–42); I have not benchmarked it here.
- Moving a value costs `size_of::<T>()` bytes (Lesson 07): large inline types (`[u8; 4096]` fields, big enum variants) are expensive to move; `Box` them.
- Niches make `Option<Box<T>>`, `Option<&T>`, and `Option<NonZero*>` free; prefer `NonZeroU32` for "ID or absent" fields where ID 0 is invalid.
- `repr(packed)` can make accesses slower or require fixup code on some targets and makes safe references impossible; use only when the layout is a requirement.
- `repr(align(64))` wastes memory (a 1-byte value occupies 64) but can prevent false sharing between threads; verify with a profiler.
- Array-of-structs versus struct-of-arrays is a layout decision that affects cache behavior; Lesson 42 revisits it with measurements.

## 10. Security and reliability

- **Never use in-memory layout as a protocol or file format.** Layout depends on the compiler version (for `repr(Rust)`), the target (alignment, pointer size, endianness), and `repr` attributes. Use explicit encoding with fixed-width integers and explicit endianness, validate lengths before reading, and return errors for short or malformed input (the practical challenge does exactly this).
- **Padding is uninitialized.** Copying a struct's raw bytes to a socket, file, or hash input can leak stale memory and makes hashes depend on garbage. Serialize field by field.
- **Misaligned references and out-of-range tags are undefined behavior**, not just crashes. Never create `&u32` from arbitrary bytes, and never `transmute` an arbitrary integer into an enum (only valid discriminants are allowed): use `TryFrom<u8>`, as `Kind` does.
- Hash/signature code that serializes structures (blockchain transactions, Lessons 48–55) must use a canonical, specified encoding: a layout-dependent encoding is a consensus bug waiting for a compiler upgrade.
- Pin layout assumptions you do depend on (FFI structs, `repr` enums) with `const _: () = assert!(...)` and tests, as in the practical challenge.

---

## 11. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Compute, by hand, the size, alignment and field offsets of `#[repr(C)] struct S { a: u8, b: u16, c: u8 }`, then check with `size_of`, `align_of`, and `offset_of!`. What is the size of the same struct under `repr(Rust)` on this compiler?
2. **(Easy)** Which is larger on this target, `Option<u32>` or `Option<NonZeroU32>`? By how much, and why?
3. **(Medium)** Predict `size_of::<Msg>()` for `enum Msg { Quit, Move { x: i32, y: i32 }, Write(String) }` before running it. Explain the number.
4. **(Medium)** Reorder the fields of `#[repr(C)] struct Bad { a: u8, b: u64, c: u16, d: u8, e: u32 }` to minimize its size. What is the minimum, and what does the same struct without `repr(C)` give?
5. **(Medium)** Add compile-time assertions that `Option<Box<T>>` has the same size as `Box<T>` and that your `Header` is a specific size. Show what the error looks like when an assertion fails.
6. **(Hard)** Explain why `std::mem::transmute::<Header, [u8; N]>` is a poor way to serialize a `repr(C)` struct that has padding, and write the safe alternative for `struct H { version: u8, length: u32, kind: u8 }`.
7. **(Hard)** What does `Vec::<()>::new()` report for `capacity()` after pushing 1000 `()` values, and why is that not an allocation problem?

<details markdown="1">
<summary>Answers</summary>

1. ✓ `repr(C)`: offsets `a = 0`, `b = 2` (aligned to 2, one padding byte at offset 1), `c = 4`; size 6 (one trailing padding byte so the size is a multiple of the alignment 2); alignment 2. ✓ Under `repr(Rust)` this compiler gave size **4** (reordered to `b, a, c`, observed, not guaranteed).
2. ✓ `Option<u32>` is 8 bytes, `Option<NonZeroU32>` is 4. `u32` uses all 2³² bit patterns, so `Option<u32>` needs a separate tag (padded to alignment 4); `NonZeroU32` never holds zero, so `None` is stored as zero in the same four bytes.
3. ✓ **24**. `Move` needs 8 bytes plus a tag, `Write(String)` needs the 24-byte `String`, but the compiler found a niche in `String`'s fields to encode the discriminant, so no extra tag space is added (observed on this compiler; don't rely on it).
4. ✓ Sorting by descending alignment: `b: u64, e: u32, c: u16, a: u8, d: u8` gives **16** bytes with no inner padding (the original `repr(C)` order is **24**). ✓ The plain `repr(Rust)` struct with the original field order is also **16**: the compiler does the reordering for you.
5. ✓ `const _: () = assert!(size_of::<Option<Box<u8>>>() == size_of::<Box<u8>>());` and `const _: () = assert!(size_of::<Header>() == 8);` (for a `repr(C)` `Header { version: u8, kind: u8, length: u32 }`) compile; a wrong expectation fails the build with `error[E0080]: evaluation panicked: <your message>`.
6. Padding bytes (and, for `repr(Rust)`, their positions) are not part of the value; they are uninitialized, so `transmute` to bytes produces uninitialized bytes (undefined behavior to read in general, and a way to leak stale memory), plus the result depends on endianness and layout. Safe alternative (✓ tested):
   ```rust
   const WIRE_LEN: usize = 6;
   fn encode(h: &H) -> [u8; WIRE_LEN] {
       let l = h.length.to_le_bytes();
       [h.version, l[0], l[1], l[2], l[3], h.kind]
   }
   ```
   The `H1` `repr(C)` struct was 12 bytes ✓ and the `repr(C, packed)` version 6 ✓, but the wire format is 6 bytes by *definition*, not because of any `repr`.
7. ✓ `capacity()` is `usize::MAX`: elements take no space, so the vector can "hold" as many as the length type allows without allocating. Pushing 1000 `()` values performed no allocation.

</details>

---

## 12. Debugging challenge

A teammate wrote the header encoder below. The test passes on their machine only because they never ran it: it should check that the encoded bytes match the size of `Header`, but it fails.

```rust
struct Header { version: u8, length: u32, kind: u8 }

fn encode(h: &Header) -> Vec<u8> {
    let mut out = Vec::with_capacity(std::mem::size_of::<Header>());
    out.push(h.version);
    out.extend(h.length.to_le_bytes());
    out.push(h.kind);
    out
}

fn main() {
    let bytes = encode(&Header { version: 1, length: 300, kind: 7 });
    println!("encoded {} bytes, size_of::<Header>() = {}", bytes.len(), std::mem::size_of::<Header>());
    assert_eq!(bytes.len(), std::mem::size_of::<Header>());
}
```

✓ Output: `encoded 6 bytes, size_of::<Header>() = 8` followed by `assertion `left == right` failed` (left 6, right 8). Explain the bug, why the "obvious" patches are wrong, and fix it properly.

<details markdown="1">
<summary>Solution</summary>

**Bug.** The code treats `size_of::<Header>()` (the in-memory size, 8 under today's `repr(Rust)`: the compiler reordered the fields and added padding) as the **wire size**. The wire format is 1 + 4 + 1 = 6 bytes, defined by the encoding code, not by memory layout. The two numbers are unrelated and can differ across compilers and targets.

**Wrong patches.**

- `#[repr(C)]` ✓ makes it 12, which still isn't 6.
- `#[repr(C, packed)]` ✓ makes it 6 on this compiler, so the assertion would pass, but it only moves the coincidence: it makes `&h.length` illegal (E0793), says nothing about endianness, and ties the file format to a compiler attribute. The test would still be asserting the wrong relationship.
- Changing the test to expect 8, or deleting the assertion.

**Proper fix.** Make the wire size an explicit constant, use it for the buffer capacity and the check, and keep layout out of it:

```rust
const WIRE_LEN: usize = 6;

struct Header { version: u8, length: u32, kind: u8 }

fn encode(h: &Header) -> Vec<u8> {
    let mut out = Vec::with_capacity(WIRE_LEN);
    out.push(h.version);
    out.extend(h.length.to_le_bytes());
    out.push(h.kind);
    out
}

fn main() {
    let bytes = encode(&Header { version: 1, length: 300, kind: 7 });
    assert_eq!(bytes.len(), WIRE_LEN);
    println!("{bytes:?}");
}
```

✓ prints `[1, 44, 1, 0, 0, 7]` (300 = `0x012C`, little-endian `2C 01 00 00`). The practical challenge builds the full encode/decode pair with this discipline, including a decoder that returns an error for short or malformed input.

</details>

---

## 13. Practical challenge: `layoutlab`

Create a library crate `layoutlab` (edition 2024) with:

1. A wire-format header: `version: u8`, `kind: Kind` (a `repr(u8)` enum with `Ping = 1`, `Data = 2`, `Close = 3`), `length: u32`. Wire layout: version (1 byte), kind (1 byte), length (4 bytes, little endian); `pub const HEADER_LEN: usize = 6`.
2. `Header::encode(&self) -> [u8; HEADER_LEN]` and `Header::decode(&[u8]) -> Result<Header, DecodeError>`; extra bytes are ignored; too-short input and unknown kinds are **errors, never panics**. Use `TryFrom<u8> for Kind` and a slice pattern in `decode`.
3. Tests for: round trip and the exact bytes; every input length shorter than 6; every possible kind byte (0..=255); extra trailing bytes.
4. **Layout tests** that pin down facts: `repr(C)` offsets and padding, size a multiple of alignment, `repr(u8)`, `repr(transparent)`, `repr(align(64))` including the array stride, `repr(packed)` with a safe copy-out, the guaranteed `Option` niches, ZSTs, and fat-pointer sizes. Put facts that are only *observed* (`repr(Rust)` field reordering, `Option<String>` niche) in separate tests named `observed_*` with a comment saying they are not promised by the language.
5. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! A tiny wire format, plus tests that pin down what Rust does and does not
//! promise about memory layout.

/// Size of the encoded header in bytes. This is a property of the *wire
/// format*, not of `size_of::<Header>()`.
pub const HEADER_LEN: usize = 6;

/// Message kind. `repr(u8)` fixes the in-memory representation to one byte
/// with the discriminants below, which are also the wire values.
#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Kind {
    Ping = 1,
    Data = 2,
    Close = 3,
}

#[derive(Debug, PartialEq, Eq)]
pub enum DecodeError {
    TooShort { needed: usize, got: usize },
    UnknownKind(u8),
}

impl TryFrom<u8> for Kind {
    type Error = DecodeError;

    fn try_from(b: u8) -> Result<Self, Self::Error> {
        match b {
            1 => Ok(Kind::Ping),
            2 => Ok(Kind::Data),
            3 => Ok(Kind::Close),
            other => Err(DecodeError::UnknownKind(other)),
        }
    }
}

/// Message header. Field order here is irrelevant to the wire format.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Header {
    pub version: u8,
    pub kind: Kind,
    pub length: u32,
}

impl Header {
    /// Encodes as: version (1 byte), kind (1 byte), length (4 bytes, little endian).
    pub fn encode(&self) -> [u8; HEADER_LEN] {
        let l = self.length.to_le_bytes();
        [self.version, self.kind as u8, l[0], l[1], l[2], l[3]]
    }

    /// Decodes the first `HEADER_LEN` bytes; extra bytes are ignored.
    pub fn decode(bytes: &[u8]) -> Result<Header, DecodeError> {
        let [version, kind, l0, l1, l2, l3, ..] = *bytes else {
            return Err(DecodeError::TooShort {
                needed: HEADER_LEN,
                got: bytes.len(),
            });
        };
        Ok(Header {
            version,
            kind: Kind::try_from(kind)?,
            length: u32::from_le_bytes([l0, l1, l2, l3]),
        })
    }
}

const _: () = assert!(HEADER_LEN == 6);

#[cfg(test)]
mod tests {
    use super::*;
    use std::marker::PhantomData;
    use std::mem::{align_of, offset_of, size_of};
    use std::num::NonZeroU32;

    // ---------- the wire format ----------

    #[test]
    fn round_trip() {
        let h = Header {
            version: 1,
            kind: Kind::Data,
            length: 300,
        };
        let bytes = h.encode();
        assert_eq!(bytes, [1, 2, 44, 1, 0, 0]);
        assert_eq!(Header::decode(&bytes), Ok(h));
    }

    #[test]
    fn short_input_is_an_error_not_a_panic() {
        let full = Header {
            version: 9,
            kind: Kind::Ping,
            length: 1,
        }
        .encode();
        for n in 0..HEADER_LEN {
            assert_eq!(
                Header::decode(&full[..n]),
                Err(DecodeError::TooShort {
                    needed: HEADER_LEN,
                    got: n
                })
            );
        }
    }

    #[test]
    fn every_kind_byte_is_handled() {
        for b in 0..=255u8 {
            let r = Header::decode(&[0, b, 0, 0, 0, 0]);
            if (1..=3).contains(&b) {
                assert!(r.is_ok(), "byte {b}");
            } else {
                assert_eq!(r, Err(DecodeError::UnknownKind(b)));
            }
        }
    }

    #[test]
    fn extra_bytes_are_ignored() {
        let mut v = Header {
            version: 1,
            kind: Kind::Close,
            length: 5,
        }
        .encode()
        .to_vec();
        v.extend_from_slice(&[0xAA, 0xBB]);
        assert_eq!(Header::decode(&v).unwrap().length, 5);
    }

    // ---------- layout facts that ARE guaranteed ----------

    #[repr(C)]
    struct C1 {
        a: u8,
        b: u32,
        c: u8,
    }
    #[repr(C)]
    struct C2 {
        b: u32,
        a: u8,
        c: u8,
    }

    #[test]
    fn repr_c_uses_declaration_order_and_padding() {
        assert_eq!(
            (offset_of!(C1, a), offset_of!(C1, b), offset_of!(C1, c)),
            (0, 4, 8)
        );
        assert_eq!(size_of::<C1>(), 12);
        assert_eq!(
            (offset_of!(C2, b), offset_of!(C2, a), offset_of!(C2, c)),
            (0, 4, 5)
        );
        assert_eq!(size_of::<C2>(), 8);
        assert_eq!(align_of::<C1>(), 4);
    }

    #[test]
    fn size_is_a_multiple_of_alignment() {
        assert_eq!(size_of::<C1>() % align_of::<C1>(), 0);
        assert_eq!(size_of::<C2>() % align_of::<C2>(), 0);
    }

    #[repr(transparent)]
    struct Meters(f64);

    #[repr(align(64))]
    struct CacheLine(#[allow(dead_code)] u8);

    #[repr(C, packed)]
    struct Packed {
        a: u8,
        b: u32,
    }

    #[test]
    fn repr_attributes_do_what_they_say() {
        assert_eq!(size_of::<Kind>(), 1);
        assert_eq!(Kind::Data as u8, 2);
        assert_eq!(size_of::<Meters>(), size_of::<f64>());
        assert_eq!(align_of::<Meters>(), align_of::<f64>());
        assert_eq!((size_of::<CacheLine>(), align_of::<CacheLine>()), (64, 64));
        assert_eq!(size_of::<[CacheLine; 3]>(), 192);
        assert_eq!((size_of::<Packed>(), align_of::<Packed>()), (5, 1));
        let p = Packed {
            a: 1,
            b: 0x0102_0304,
        };
        let copy = p.b; // copying a packed field is fine; `&p.b` would be E0793
        assert_eq!((p.a, copy), (1, 0x0102_0304));
    }

    #[test]
    fn documented_niche_guarantees() {
        // std documents these Option layouts as guaranteed.
        assert_eq!(size_of::<Option<&u8>>(), size_of::<&u8>());
        assert_eq!(size_of::<Option<Box<u8>>>(), size_of::<Box<u8>>());
        assert_eq!(size_of::<Option<NonZeroU32>>(), size_of::<u32>());
        assert_eq!(size_of::<Option<fn()>>(), size_of::<fn()>());
    }

    #[test]
    fn zero_sized_types_take_no_space() {
        struct Unit;
        assert_eq!(size_of::<()>(), 0);
        assert_eq!(size_of::<Unit>(), 0);
        assert_eq!(size_of::<[u64; 0]>(), 0);
        assert_eq!(size_of::<PhantomData<String>>(), 0);
    }

    #[test]
    fn fat_pointers_are_two_words() {
        assert_eq!(size_of::<&[u8]>(), 2 * size_of::<usize>());
        assert_eq!(size_of::<&str>(), 2 * size_of::<usize>());
        assert_eq!(size_of::<Box<dyn Fn()>>(), 2 * size_of::<usize>());
    }

    // ---------- layout facts that are only OBSERVED on this toolchain ----------
    // These document current behavior; the language does not promise them,
    // so a failure after a compiler upgrade is information, not necessarily a bug.

    #[test]
    fn observed_repr_rust_reorders_fields() {
        struct R {
            _a: u8,
            _b: u32,
            _c: u8,
        }
        assert_eq!(size_of::<R>(), 8); // repr(C) with the same fields is 12
    }

    #[test]
    fn observed_niche_in_option_string_and_bool() {
        assert_eq!(size_of::<Option<String>>(), size_of::<String>());
        assert_eq!(size_of::<Option<bool>>(), 1);
        assert_eq!(size_of::<Option<u32>>(), 8); // u32 has no niche
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 12 passed.

**Design notes.**

- `decode` uses `let [version, kind, l0, l1, l2, l3, ..] = *bytes else { ... }`: a slice pattern (Lesson 06) that matches *only* if there are at least six bytes, so no index can go out of range. The `else` branch must diverge, and here it returns `TooShort`. No `unwrap`, no `as`, no unsafe.
- The wire size is the constant `HEADER_LEN`. The test names and comments keep "layout facts" separate from "protocol facts", which is the lesson of the debugging challenge. The `const _: () = assert!(HEADER_LEN == 6);` is a compile-time reminder that changing it changes the protocol.
- `Kind::try_from` is the only way to turn a byte into a `Kind`; invalid values become errors instead of undefined behavior (never `transmute` a byte into an enum).
- The `observed_*` tests document what this compiler does today. If a future compiler changes them, that test failing is information about layout, not a bug in the lesson's code.
- `CacheLine(#[allow(dead_code)] u8)` exists only to give the type a size; the field is never read, so it carries a narrow `allow` rather than a crate-wide one.

</details>

---

## 14. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What do `size_of` and `align_of` mean?** Size is the number of bytes a value occupies, including inner and trailing padding; alignment is the power of two its address must be a multiple of. Size is always a multiple of alignment.
2. **Why does Rust reorder struct fields?** `repr(Rust)` is unspecified so the compiler can minimize padding and create niches; a stable layout needs `repr(C)` or another `repr`.
3. **What does `repr(C)` guarantee?** Fields in declaration order with C's padding and alignment rules (and C-compatible enum layout). It does not minimize size.
4. **What is `repr(transparent)` for?** A newtype with one non-zero-sized field has exactly that field's layout and ABI, so it can be passed across FFI as the inner type.
5. **What is a niche and why is `Option<Box<T>>` the size of `Box<T>`?** A niche is a bit pattern the type never uses (null for `Box`); `None` is stored in it, so no extra tag is needed. This is guaranteed for `Option` of references, `Box`, `NonZero*`, and function pointers.
6. **Why is `Option<u64>` 16 bytes?** `u64` has no spare bit patterns, so a separate tag is needed, and padding for alignment makes it 16.
7. **How big is an enum?** Tag plus the largest payload, rounded to the strictest alignment, minus anything saved by niches.
8. **What is a fat pointer?** A pointer plus metadata: a length for slices and `str`, a vtable pointer for `dyn Trait`. Two words on 64-bit.
9. **What is a zero-sized type and why use it?** A type with size 0 (`()`, unit structs, `PhantomData<T>`, `[T; 0]`): compile-time markers and typestates at no runtime cost; collections of them don't allocate.
10. **Why can't you take a reference to a field of a `repr(packed)` struct?** The field may be misaligned and a misaligned reference is undefined behavior (E0793). Copy the field out or use `read_unaligned` through a raw pointer.
11. **Why is `transmute`-based serialization wrong?** Padding bytes are uninitialized, layout and endianness are platform-dependent, and `repr(Rust)` layout can change between compiler versions; encode explicitly instead.
12. **How would you assert a layout assumption?** `const _: () = assert!(size_of::<T>() == N);` (compile-time) plus tests; use `offset_of!` for field offsets of `repr(C)` types.
13. **How do you sort fields to reduce size under `repr(C)`?** Descending alignment, largest first; trailing padding then only appears at the end.
14. **Why `u32::from_le_bytes` rather than casting a `&[u8]` pointer?** Casting requires alignment and matching endianness and is undefined behavior if misaligned; `from_le_bytes` is safe, portable, and equally fast where a plain load is valid.

</details>

---

## 15. Key takeaways

- Every type has a size and an alignment; size is a multiple of alignment; padding fills the gaps. Check with `size_of`, `align_of`, and `offset_of!`.
- `repr(Rust)` (default) is **unspecified**: the compiler reorders fields and uses niches. Use `repr(C)` for FFI-stable layout (and sort fields yourself), `repr(u8)` for fieldless enum values, `repr(transparent)` for newtypes, `repr(align(N))` to raise alignment, and `repr(packed)` only when forced, with care.
- Niches make `Option<&T>`, `Option<Box<T>>`, `Option<NonZero*>` free; types that use every bit pattern (`u32`, `u64`) don't have one.
- Fat pointers (slices, `str`, `dyn`) are two words; ZSTs take no space; `size_of_val` measures the pointee.
- Never reinterpret memory as a protocol: encode explicitly with `to_le_bytes`/`from_le_bytes`, validate lengths, and handle malformed input as errors. Padding is uninitialized; misaligned references are undefined behavior.
- Pin the layout facts you rely on with const assertions and tests, and label the ones that are merely observed.

**Connections.** Lesson 01 introduced niche optimization and `Option<&u32>`; Lesson 07 said a move copies `size_of::<T>()` bytes and showed `Option<String>` at 24 bytes; Lesson 08 introduced fat pointers and the stack/heap split. Next: Lesson 10 starts Module 4 (references, shared and mutable borrowing, aliasing, reborrowing), where "a reference is an aligned, valid pointer" becomes the formal rule. Lesson 15 builds newtypes on `repr(transparent)` and typestates on ZSTs; Lesson 21 explains vtables; Lesson 36–37 cover unsafe code, `MaybeUninit`, FFI, and `repr(C)` in depth; Lesson 42 revisits cache effects and false sharing with measurement; Lessons 48–49 depend on canonical serialization. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 16. References

- *The Rust Reference*, type layout (size, alignment, `repr` attributes, layout of primitives, enums, and closures): <https://doc.rust-lang.org/reference/type-layout.html>
- *The Rustonomicon*, "Data Representation in Rust" (alignment, exotically sized types, repr(C), other reprs): <https://doc.rust-lang.org/nomicon/data.html>
- Rust standard library documentation, `std::mem::size_of`, `align_of`, `size_of_val`, and `offset_of!`: <https://doc.rust-lang.org/std/mem/fn.size_of.html>, <https://doc.rust-lang.org/std/mem/macro.offset_of.html>
- Rust standard library documentation, `Option` ("Representation": the guaranteed niche layouts): <https://doc.rust-lang.org/std/option/index.html#representation>
- Rust standard library documentation, `NonZeroU32` and the other `NonZero*` types: <https://doc.rust-lang.org/std/num/struct.NonZero.html>
- Rust error code index: E0793 (packed references), E0690 (`repr(transparent)`), E0517 (misapplied `repr`): <https://doc.rust-lang.org/error_codes/E0793.html>
- Rust standard library documentation, integer `to_le_bytes` / `from_le_bytes`: <https://doc.rust-lang.org/std/primitive.u32.html#method.from_le_bytes>
- *Rust unsafe code guidelines* (layout and validity of types; still a work in progress, so treat it as guidance rather than specification): <https://rust-lang.github.io/unsafe-code-guidelines/layout.html>
- *Clippy lint list*, `large_enum_variant`: <https://rust-lang.github.io/rust-clippy/master/index.html#large_enum_variant>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title.
