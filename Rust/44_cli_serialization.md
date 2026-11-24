---
layout: default
title: "Lesson 44 — CLI tools and serialization"
---

# Lesson 44 — CLI tools and serialization

**Module 14 · Application development** · Status: generated · Edition used in examples: **2024** · External dependencies: `clap 4.6.7` (with `derive`), `serde 1.0.229` (with `derive`), `serde_json 1.0.154`

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux, 2 cores. As before, ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything runs on Linux only: I did not test Windows or macOS behavior (path handling, terminal colors, how a closed pipe is reported), and I did not run shell completions, man-page generation, color output, `serde` formats other than JSON (TOML, YAML, CBOR, bincode, ...), or any `serde` feature beyond the attributes shown. Timings are single measurements on one shared virtual machine. I could not open the documentation sites, so rules stated without ✓ are from memory; the closing paragraph lists them.

Builds on [Lesson 43](43_observability_ci_supply_chain.md) (logging to stderr, secrets, `--locked`, dependency policy), [Lesson 41](41_benchmarking_profiling.md) and [Lesson 42](42_performance_engineering.md) (how the measurements here were taken, binary-size knobs), [Lesson 39](39_testing.md) (testing), [Lesson 26](26_error_design.md) (error types and messages), [Lesson 35](35_proc_macros.md) (what `#[derive(Parser)]` and `#[derive(Serialize)]` expand to), and [Lesson 14](14_option_result_errors.md) (`Result` and `?`). Lesson 45 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

Two kinds of boundary decide whether a program is pleasant and safe to use:

1. **The command line**: how a human or a script talks to the program. Arguments, standard input and output, error messages, and *exit codes* are an interface other programs depend on, as much as any function signature.
2. **Serialization**: how data crosses time and process boundaries as bytes (a file, a pipe, a network message). Whatever the other side sends is **untrusted input**, and whatever you write today must still be readable by tomorrow's version.

The running example is `reclab`, a small tool that reads **JSON Lines** (one JSON object per line) order files, validates them, and summarizes them. It has three subcommands (`summarize`, `validate`, `cat`), documented exit codes, and constant memory use. Everything marked ✓ comes from it or from the small programs shown.

---

## 2. Designing a command-line program

### 2.1 The contract ✓

| Channel | Use it for | Why |
|---|---|---|
| **stdout** | the program's *result*, in a stable format | scripts pipe it into other programs |
| **stderr** | warnings, errors, progress, logs | stays out of the data stream (Lesson 43) |
| **exit code** | success or the *kind* of failure | `if`, `&&`, CI and schedulers read it |
| **arguments** | what to do and how | documented by `--help` |
| **stdin** | input when no file is given, or when the file is `-` | composes with pipes |

`reclab` defines four exit codes, tested:

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | fatal error: unreadable file, I/O failure, or a bad line in `--strict` mode |
| 2 | usage error (bad arguments; this is what `clap` uses) |
| 3 | the run finished but some lines were rejected |

✓ Real runs (abridged) on a seven-line file with three bad lines (an unknown field, malformed JSON, an empty customer):

```text
$ reclab summarize orders.jsonl
reclab: warning: line 5: unknown field `extra`, expected one of `id`, `customer`, `amountCents`, `tags`, `status` at line 1 column 63
reclab: warning: line 6: expected ident at line 1 column 2
reclab: warning: line 7: customer must be 1 to 64 characters, got 0
lines: 7 (accepted 4, rejected 3)
total: 79.49
pending: 1
paid: 2
refunded: 1
top 1: ada 62.50
top 2: bob 9.99
top 3: dee 7.00
[exit 3]

$ reclab summarize /nonexistent
reclab: error: cannot open '/nonexistent': No such file or directory (os error 2)
[exit 1]
```

Design rules that follow, all visible above:

- Results on stdout, diagnostics on stderr. ✓ A test checks that the JSON output (`--format json`) is a single valid object on stdout with the warnings elsewhere, and that a clean run leaves stderr empty.
- Errors say what failed, on what (`cannot open '/nonexistent'`) and why (the OS message). Line-level problems carry a line number.
- *Partial success is a state*, not a failure and not a success: exit 3 lets a script decide, and `--strict` turns the first problem into exit 1.
- Offer a machine-readable output format (`--format json`) next to the human one; keep the machine format stable and documented.
- Read from stdin when the file is `-` (or absent) so the tool composes in pipelines.
- Never require interaction when it is not a terminal; do not print progress bars to stdout.
- Do not pass secrets as arguments: command lines are visible to other users (`ps`, `/proc`, from memory; not shown here) and end up in shell history. Read secrets from a file, a descriptor, standard input, or an environment variable chosen deliberately (Lesson 43).

### 2.2 `main` stays thin ✓

```rust
fn main() -> ExitCode {
    let stdin = io::stdin();
    let mut input = stdin.lock();
    let mut output = BufWriter::new(io::stdout().lock());
    let mut errors = io::stderr().lock();
    let code = reclab::run(std::env::args_os(), &mut input, &mut output, &mut errors);
    ExitCode::from(code)
}
```

All logic is `reclab::run(args, stdin, stdout, stderr) -> u8`, generic over the streams. Tests call it with in-memory buffers: no process spawning, no temporary files, millisecond runtimes. ✓ 12 of the 14 tests in `tests/cli.rs` do exactly this (exit codes, exact stdout, stderr content for subcommands and error paths); the other two spawn the real binary through `env!("CARGO_BIN_EXE_reclab")`, which Cargo sets for integration tests, to prove the wiring (stdin, exit codes, the closed-pipe case below).

Prefer `args_os()` over `args()` at the boundary: `args()` panics on a non-UTF-8 argument, and file names on Unix need not be UTF-8. `clap` accepts `OsString`s and `PathBuf` for this reason.

### 2.3 The closed-pipe trap ✓

Run a producer into `head -1`: the reader exits after one line and closes the pipe. What does the writer see? Rust programs start with `SIGPIPE` ignored (from memory), so the next write fails with `ErrorKind::BrokenPipe` instead of killing the process silently as a C program would. The classic `println!` loop turns that error into a panic. ✓ `examples/sigpipe_demo.sh` on a 13.7 MB file (200,000 lines):

```text
== naive println! loop | head -1
{"id":0,"customer":"c0","amountCents":0,"status":"pending"}

thread 'main' panicked at library/std/src/io/stdio.rs:1166:9:
failed printing to stdout: Broken pipe (os error 32)
exit status of naive_cat: 101

== reclab cat | head -1
{"id":0,"customer":"c0","amountCents":0,"status":"pending"}
exit status of reclab: 0
```

A consumer that stops reading early is *not an error in the producer*; it is the normal way pipelines say "enough". `reclab::run` treats `BrokenPipe` on any write or on the final flush as a quiet exit 0, and any other write error (✓ tested with a writer that fails with "disk full") as exit 1 with a message. Two tests pin this: one with an in-memory writer that fails after ten bytes, one with the real binary whose output pipe is dropped after the first line (stderr must stay empty, exit code 0).

Related stdout facts: `println!` locks stdout on every call and, when stdout is a terminal or a pipe, flushes per line (stdout is line-buffered, from memory). For large outputs wrap `stdout().lock()` in a `BufWriter` as `main` does, and remember to flush before exiting (here `run` flushes and maps the error).

---

## 3. `clap`: declaring the interface ✓

`clap`'s derive API turns a struct into a parser (Lesson 35: the derive expands to code at compile time):

```rust
#[derive(Debug, Parser)]
#[command(name = "reclab", version, about = "Validate and summarize JSON Lines order files")]
pub struct Cli {
    #[command(subcommand)]
    command: Command,
}

#[derive(Debug, Subcommand)]
enum Command {
    /// Print totals and the top customers.
    Summarize(SummarizeArgs),
    /// Check every line and report the rejected ones.
    Validate(Common),
    /// Re-print accepted orders, one normalized JSON object per line.
    Cat(Common),
}

#[derive(Debug, Args)]
struct Common {
    /// Input file, or `-` for standard input.
    #[arg(default_value = "-")]
    file: PathBuf,
    /// Longest accepted line, in bytes.
    #[arg(long, default_value_t = 1_048_576, value_parser = clap::value_parser!(u32).range(16..=67_108_864))]
    max_line_bytes: u32,
    /// Stop at the first rejected line and exit with status 1.
    #[arg(long)]
    strict: bool,
}
```

What each piece does:

- `Parser` / `Subcommand` / `Args` / `ValueEnum` derive the parsing; doc comments (`///`) become the `--help` text, so the documentation cannot drift from the code.
- `#[command(flatten)]` (used for `SummarizeArgs`) reuses a group of options in several subcommands.
- `value_parser!(u8).range(1..=100)` validates at parse time. `ValueEnum` (`--format text|json`) lists the allowed values in the error message.
- `PathBuf` accepts any OS path; `default_value = "-"` makes the argument optional.
- `Cli::try_parse_from(args)` returns a `Result` instead of exiting the process, which is what makes the parser testable. The error value knows whether it belongs on stderr (`use_stderr()`), its text (`render()`) and its exit code (`exit_code()`). `Cli::parse()` is the same but prints and calls `exit` for you; fine in `main` of a small tool, awkward in tests.

✓ Parse errors and help (exit codes in brackets):

```text
$ reclab summarize --top 0
error: invalid value '0' for '--top <TOP>': 0 is not in 1..=100
For more information, try '--help'.                              [exit 2]

$ reclab summarize --format yaml
error: invalid value 'yaml' for '--format <FORMAT>'
  [possible values: text, json]                                  [exit 2]

$ reclab bogus
error: unrecognized subcommand 'bogus'                           [exit 2]

$ reclab --version
reclab 0.1.0                                                     [exit 0]
```

(The blank lines of the real output are omitted.) `--help` and `--version` go to **stdout** with exit 0 (a successful request for information); mistakes go to **stderr** with exit 2. `reclab` with no subcommand printed the help text and exited 2, both ✓ tested.

### 3.1 What `clap` costs ✓

| | measured on this machine |
|---|---|
| `reclab` release binary (default profile) | 1,359,472 bytes |
| `reclab` with `strip`, fat LTO, one codegen unit, `panic = "abort"` (Lesson 42's recipe) | 763,128 bytes |
| a std-only filter (`naive_cat`: read stdin lines, print them), default release | 455,992 bytes |
| cold `cargo build --release` of `reclab` (clap, serde, serde_json and their dependencies; 2 cores) | 14.4 seconds |

So the argument parser and JSON support cost roughly 0.9 MB of binary at default settings (0.3 MB after stripping and LTO) and a noticeable compile. For a tool with a handful of options a smaller parser (`pico-args`, `lexopt`, hand-rolled; from memory, not run) is a legitimate choice; for several subcommands with typed values and generated help, `clap` earns its cost. Measure before deciding, as in Lesson 41.

---

## 4. Serialization with `serde` and JSON

`serde` splits the problem in two: your types describe themselves with `#[derive(Serialize, Deserialize)]` (the *data model*), and a format crate (`serde_json`) turns that model into bytes and back. The same derive works with other formats (not run here). Attributes on the type define the **file format**, which makes them part of your public contract.

### 4.1 The model of `reclab` ✓

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Status { Pending, Paid, Refunded }

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct Order {
    pub id: u64,
    pub customer: String,
    pub amount_cents: u64,
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    pub tags: Vec<String>,
    pub status: Status,
}
```

- `rename_all`: Rust names stay idiomatic (`amount_cents`), the file uses `amountCents`. ✓ A test shows `amount_cents` in the input is *rejected* (`unknown field`), so the format is exactly one spelling.
- `deny_unknown_fields`: a misspelled key is an error, not silently ignored (see §4.4 for the trade-off).
- `default` + `skip_serializing_if`: `tags` may be omitted on input, and is omitted on output when empty. ✓ Round trip test: the serialized form of an order without tags has no `tags` key and parses back to an equal value.

### 4.2 Errors come with positions ✓

Error texts that appeared in this lesson's runs (`serde_json` 1.0.154):

| Input | Error |
|---|---|
| an extra key in an `Order` line | ``unknown field `extra`, expected one of `id`, `customer`, `amountCents`, `tags`, `status` at line 1 column 63`` |
| a line that is not JSON | `expected ident at line 1 column 2` |
| a wrong-typed value in the test suite | `column 38` is where the parser reported the position of the offending value (my first test guessed 37) |
| duplicate field in a struct | an error (a `Value` instead keeps the last one) |
| a newtype with `try_from` | your own message, plus the position: `name must be 1 to 8 bytes, got 10 at line 1 column 21` |
| `&str` field with an escaped string | `invalid type: string "a\nb", expected a borrowed string` |

Other mistakes (missing field, wrong type, negative into `u64`, trailing comma or characters) also produce messages with the offending type and a line and column; I did not record their exact wording, so run your own input to see it.

Good error messages are free here: type, expected type, location. In a line-oriented format the *column* is relative to the line, so `reclab` prefixes its own line number (`line 5: ... at line 1 column 63`). It reads slightly oddly (two "line"s), but both are accurate; a polished tool would rewrite the second part.

### 4.3 Numbers are where formats lie ✓

| Experiment | Result |
|---|---|
| `u64::MAX` serialize, parse back | `18446744073709551615`, exact |
| `18446744073709551616` (2⁶⁴) as `u64` | error: `invalid type: floating point `1.8446744073709552e+19`, expected u64` |
| `9007199254740993` (2⁵³+1) as `f64` | `9007199254740992.0` (rounded) |
| `1e400` as `f64` | error: `number out of range` |
| `f64::NAN` and `f64::INFINITY` serialized | **`null`** (silently!) |
| `0.1 + 0.2` serialized | not run; the usual float rounding applies |
| `-0` as `f64` | `-0.0` |

Consequences:

- JSON has one number type and no way to write NaN or infinity. `serde_json` serializes them as `null`, and reading that `null` back into an `f64` fails, so **a computation that produced NaN can write a file you cannot read**. Check `is_finite()` before serializing, or reject it in validation.
- Many other JSON consumers (JavaScript, for one, from memory) store numbers as 64-bit floats, so integers above 2⁵³ lose precision in them even though `serde_json` handles them exactly. Large ids that a browser must read are conventionally sent as strings.
- Money is stored as integer cents in `reclab` (`amountCents`) for this reason: no float rounding (`0.1 + 0.2`).
- `u64` fields reject negative numbers and fractions with a clear error: the type is the validation.

### 4.4 Strictness, evolution and compatibility ✓

By default `serde` **ignores unknown fields**. ✓ A reader struct with only `id` parsed `{"id":1,"added_later":true}` without complaint. That is what lets an old reader survive a newer writer that adds fields (forward compatibility); `deny_unknown_fields` gives it up in exchange for catching typos. Choose per type:

- Files written by *your own tool* and read back by it, or user-edited configuration where a typo should be loud: `deny_unknown_fields` (as `Order`).
- Data exchanged between independently upgraded programs: tolerate unknown fields, give new fields a `default`, and never reuse or repurpose a name. Add a `version` field if you may need to change meaning.

Other tools from the same family, all ✓ run:

- `#[serde(tag = "type", rename_all = "snake_case")]` on an enum gives an *internally tagged* representation: `{"type":"logout","user":"a","secs":5}`; a unit variant is `{"type":"ping"}`. Errors name the problem: `unknown variant `explode`, expected one of `login`, `logout`, `ping``, and `missing field `type``.
- `#[serde(untagged)]` tries each variant in order and on failure says only `data did not match any variant of untagged enum Either`, with no position or reason. Use it only when no tag is possible.
- `#[serde(alias = "nick", default = "anon")]` accepts an old name and supplies a default; `#[serde(flatten)] rest: BTreeMap<String, Value>` captures the leftover keys (`{"id":1,"nick":"n","color":"red","n":2}` gave `name: "n"` and `rest: {"color": ..., "n": ...}`).
- `Option<T>` fields accept both a missing key and `null` as `None`.
- Validation inside the type: `#[serde(try_from = "String", into = "String")]` on a newtype runs your `TryFrom` during deserialization: input `"abcdefghij"` produced `name must be 1 to 8 bytes, got 10 at line 1 column 21`. The `into` direction requires `Clone` (✓ the compiler said so). This pushes validation into the type, which is the idiom of Lesson 15; `reclab` instead validates after parsing in `parse_line`, which keeps all range checks in one visible place.
- Maps with non-string keys: `BTreeMap<u32, _>` serializes the key as a string (`{"1":"one"}`); a tuple key fails with `key must be a string`.

### 4.5 `Value`, borrowing and zero-copy ✓

- `serde_json::Value` is the "any JSON" tree. ✓ `{"b":1,"a":[1,2,{"z":null}]}` printed back as `{"a":[1,2,{"z":null}],"b":1}`: keys are **sorted** (a `BTreeMap`; an `indexmap`-based feature that preserves order exists, from memory). A duplicate key in a `Value` keeps the last one (`{"a":1,"a":2}` → `a: 2`), but into a struct it is an error (above). Prefer typed structs for known formats; use `Value` for inspection or pass-through.
- Borrowing: a struct field `Cow<'a, str>` with `#[serde(borrow)]` borrows from the input when the JSON string has no escapes and allocates when it does. ✓ `"plain"` stayed borrowed, `"a\nb"` did not. A plain `&'a str` field cannot represent an escaped string at all: `invalid type: string "a\nb", expected a borrowed string`. Borrowing saves allocations (Lesson 41/42) but ties the value's lifetime to the input buffer, which is why `reclab` deserializes owned `String`s from a reused line buffer.

### 4.6 Untrusted input ✓

Anything deserialized is attacker-controlled unless proven otherwise. The failure modes that matter:

- **Depth.** ✓ `serde_json` has a recursion limit (128 levels): 200 nested `[` into a `Value` failed at column 128 with the recursion-limit error; 100 levels parsed fine. A deeply nested input therefore cannot overflow the stack through `serde_json`'s own recursion (turning the limit off is a feature flag; leave it on).
- **Size.** Nothing limits how large a document or a line may be. `reclab` reads lines with `read_line_bounded`, a `Take`-based reader that never holds more than the limit in memory (§5). Without such a limit, `BufRead::read_line` on a stream with no newline buffers everything until memory runs out.
- **Breadth.** A flat object with millions of keys, or a huge array, costs memory proportional to its size. Bound the input, not just the depth.
- **Semantic validity.** Types cannot say "1 to 64 characters" or "at most ten billion dollars". ✓ `parse_line` checks both after parsing, counting characters not bytes (✓ 64 `é` characters, 128 bytes, are accepted; 65 are not), with inclusive-boundary tests.
- **Overflow.** ✓ Totals are `u128` because each amount may be up to 10¹² cents: a test sums 30,000 maximal orders (3·10¹⁶), and the type means no input can wrap the total. A `u64` accumulator would eventually wrap (in release builds) or panic (in debug).
- **Output injection.** Values you echo to a terminal or a log may contain control characters (Lesson 43's log-forging experiment). Formats that spreadsheets open (CSV) have their own injection risk with cells that start with `=` (from memory; not run here).

---

## 5. Streaming: constant memory ✓

The tool never loads the file. It reads one line at a time into a reused `Vec<u8>`, parses it, updates running totals, and forgets it. The only state that grows is one counter per distinct customer, which is the information the report needs.

The bounded reader is the interesting part (full code in the practical challenge). Plain `read_until` into a `Vec` is unbounded; wrapping the reader in `Take` caps what one call can buffer:

```rust
let limit = max as u64 + 2;                       // content + room for "\r\n"
let n = reader.by_ref().take(limit).read_until(b'\n', buf)?;
// ... if the limit was hit without finding a newline:
//     the line is too long: drop the buffer and skip to the end of the line
//     with fill_buf()/consume(), without storing it
```

✓ Behavior, all tested:

- A line of exactly `max` bytes is accepted with `\n`, with `\r\n`, and at end of input without a terminator; one byte more is `TooLong`. The terminator does not count toward the limit.
- After a too-long line the reader resumes at the next line, for every internal buffer size from 1 to 64 bytes (`BufReader::with_capacity` is used to force the "skip across several `fill_buf` chunks" path).
- ✓ A 10 MB line with a 16-byte limit leaves the reusable buffer with a capacity under 1 KiB.
- A lone `\r` inside a line is data; a `\r` at the very end of the input is dropped like the first half of `\r\n`.
- My first version failed its own CRLF test: it limited the read to `max + 1` bytes, so `"12345\r\n"` with `max = 5` was misjudged as too long (the `\r` used up the spare byte). The tests caught it; the fix reads up to `max + 2` bytes and treats the `\r` as part of the terminator. This is the kind of off-by-one that only boundary tests find.

✓ Memory and speed on this machine (release build, input files of 13.7 MB and 137 MB; `max RSS` is the peak resident memory of the process as reported by the OS):

| Command | 13.7 MB file (200,000 lines): time, min of 7 | throughput |
|---|---:|---:|
| `validate` | 28 to 31 ms | about 440 to 485 MB/s |
| `summarize` | 45 ms | about 300 MB/s |
| `cat` (re-serialize every order) | 55 to 58 ms | about 240 to 250 MB/s |

Peak memory (`max RSS`, the resident set size reported by the OS): `summarize` on the 13.7 MB file peaked at **11.3 MB**; on a file **ten times larger** (137 MB) it peaked at **11.45 MB**. Memory does not grow with input size. (The 11 MB is mostly the process and its libraries; I did not measure RSS for the other two commands.) These are single-machine, warm-cache numbers (the file was in the page cache) for this input shape, and I did not compare against other parsers or formats.

---

## 6. Common mistakes

1. Printing errors to stdout, or results to stderr.
2. Exiting 0 after a failure (scripts cannot tell) or exiting non-zero after success (`grep`-style "no match" conventions need documenting).
3. `unwrap()` on user input: a typo becomes a panic with a backtrace and exit code 101 instead of a message and a usage code.
4. A `println!` loop that panics when the reader closes the pipe (✓ exit 101).
5. `std::env::args()` where a path might not be UTF-8.
6. Secrets in arguments.
7. `read_line` / `read_to_string` on untrusted input with no limit.
8. `f64` for money; serializing NaN (✓ becomes `null`).
9. `deny_unknown_fields` on a format that independently upgraded programs share (or the opposite: silently ignoring a typo in user configuration).
10. Treating a serialized format as private: attribute changes (`rename_all`, field names, enum names) change the file format.
11. `#[serde(untagged)]` when a tag is possible (✓ error messages lose all detail).
12. Testing only the parser, not the exit code and stream separation.

---

## 7. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** A script runs `reclab validate data.jsonl && deploy`. Which of the four exit codes lets `deploy` run, and what is a reasonable alternative if you want to deploy even when some lines were rejected?
2. **(Warm-up)** Why does `println!` panic when stdout is a closed pipe, while a C program would just be killed silently? What should a filter do instead?
3. **(Easy)** Write a `clap` derive struct for a `grep-lite` program: a required `PATTERN`, any number of files (standard input when none), and an `-i`/`--ignore-case` flag. Show that it parses and that a missing pattern is a usage error with exit code 2.
4. **(Medium)** An API sometimes sends `"id": 5` and sometimes `"id": "5"`. Make a struct accept both with `serde`, without accepting garbage.
5. **(Medium)** You maintain a configuration file read by many installed versions of your tool. Should the config struct use `deny_unknown_fields`? What if it is a file users write by hand and typos are the common failure?
6. **(Medium-hard)** A report tool computes an average and serializes `{"avg": f64}`. Sometimes the input is empty and the average is `0.0 / 0.0`. What does the output look like, what breaks downstream, and what are two fixes?
7. **(Hard)** Explain why `reader.take(max + 1).read_until(b'\n', buf)` is not enough to implement "lines up to `max` bytes, CRLF allowed", and how `read_line_bounded` fixes it. What state tells you the limit was hit?
8. **(Hard)** Design how a CLI should accept an API token: compare an argument, an environment variable, a file, and standard input in terms of leakage and convenience.

<details markdown="1">
<summary>Answers</summary>

1. Only exit code 0 (no rejected lines). `&&` runs the next command only on success, so partial success (3) and fatal failure (1) both stop the deploy. To deploy despite rejected lines, test the code explicitly: `reclab validate f; rc=$?; [ "$rc" -eq 0 ] || [ "$rc" -eq 3 ] && deploy`, or use `||` with a check. The point of a distinct "partial" code is that the caller can choose.
2. Rust sets the `SIGPIPE` signal to be ignored at startup (from memory), so a write to a closed pipe returns `Err(BrokenPipe)` instead of terminating the process; `println!` panics on any write error (✓ `failed printing to stdout: Broken pipe`, exit 101). A filter should write with `writeln!` to a locked, buffered handle, propagate the `io::Error` with `?`, and in `main` treat `BrokenPipe` as a normal, quiet end (✓ exit 0, empty stderr).
3. A possible answer:

   ```rust
   // needs: clap
   use clap::Parser;
   use std::path::PathBuf;

   /// Print lines containing PATTERN.
   #[derive(Debug, Parser)]
   #[command(name = "grep-lite", version)]
   struct Args {
       /// Text to look for.
       pattern: String,
       /// Files to search; standard input when none are given.
       files: Vec<PathBuf>,
       /// Ignore case.
       #[arg(short, long)]
       ignore_case: bool,
   }

   fn main() {
       let a = Args::try_parse_from(["grep-lite", "-i", "needle", "a.txt", "b.txt"]).unwrap();
       assert!(a.ignore_case);
       assert_eq!(a.pattern, "needle");
       assert_eq!(a.files, vec![PathBuf::from("a.txt"), PathBuf::from("b.txt")]);

       let a = Args::try_parse_from(["grep-lite", "needle"]).unwrap();
       assert!(a.files.is_empty() && !a.ignore_case);

       let e = Args::try_parse_from(["grep-lite"]).unwrap_err();
       assert_eq!(e.exit_code(), 2);
       println!("ok");
   }
   ```

   A `Vec<PathBuf>` positional is optional and collects zero or more values; a bare `String` positional is required.
4. Deserialize through a small untagged helper and parse the string form:

   ```rust
   // needs: serde, serde_json
   use serde::{Deserialize, Deserializer};

   #[derive(Deserialize)]
   #[serde(untagged)]
   enum NumOrStr {
       Num(u64),
       Str(String),
   }

   fn id_from_either<'de, D: Deserializer<'de>>(d: D) -> Result<u64, D::Error> {
       match NumOrStr::deserialize(d)? {
           NumOrStr::Num(n) => Ok(n),
           NumOrStr::Str(s) => s.trim().parse().map_err(serde::de::Error::custom),
       }
   }

   #[derive(Debug, Deserialize)]
   struct Item {
       #[serde(deserialize_with = "id_from_either")]
       id: u64,
   }

   fn main() {
       let a: Item = serde_json::from_str(r#"{"id":5}"#).unwrap();
       let b: Item = serde_json::from_str(r#"{"id":"5"}"#).unwrap();
       assert_eq!(a.id, 5);
       assert_eq!(b.id, 5);
       assert!(serde_json::from_str::<Item>(r#"{"id":"five"}"#).is_err());
       assert!(serde_json::from_str::<Item>(r#"{"id":-1}"#).is_err());
       assert!(serde_json::from_str::<Item>(r#"{"id":1.5}"#).is_err());
       println!("ok");
   }
   ```

   Note the untagged helper's weak error messages (✓ `data did not match any variant`); wrap it so the error for a bad string is the parse error, as done here, and consider normalizing to one representation on output.
5. For a shared file read by many versions: **no**. Version N+1 may add a key that version N must ignore (✓ default behavior), and `deny_unknown_fields` would make old installs fail on a new file. For a hand-written file, typos are the main failure, so reject unknown keys, or better, keep tolerance but *warn* about unknown keys (collect them with `#[serde(flatten)] extra: BTreeMap<String, Value>` and report them on stderr), which gives both compatibility and typo detection.
6. `0.0 / 0.0` is NaN, and ✓ `serde_json` serializes it as `{"avg":null}` without error. A reader expecting `f64` then fails on `null` (or silently treats it as "missing"), so a perfectly valid empty input produces a file the next stage cannot parse. Fixes: model the case (`avg: Option<f64>` with `None` for no data, serialized as `null` on purpose and documented), or return an error before serializing when `!x.is_finite()`. Never rely on NaN round-tripping.
7. `take(max + 1)` caps the read at `max + 1` bytes, but a line of exactly `max` content bytes ending in `\r\n` is `max + 2` bytes long, so the `\r` consumes the spare byte, no `\n` is seen, and a valid line looks too long (✓ my first version failed this test). The fix reads up to `max + 2` bytes and decides afterwards: if the buffer ends in `\n` the line is complete (strip `\n`, then `\r`); if it does not, the limit was hit only when exactly `max + 2` bytes were read without a newline (fewer means end of input). Only then must the rest of the line be skipped with `fill_buf`/`consume`; otherwise the next call would start in the middle of the long line.
8. Argument (`--token X`): visible in `ps`/`/proc` to other users and stored in shell history; avoid. Environment variable: not in `ps` output for other users on typical systems but inherited by child processes, shown in some crash reports and container inspect output; acceptable with care. File (`--token-file path`): permissions control access, easy for services and orchestrators, not in history; a good default. Standard input (`--token-stdin`): not in history or `ps`, good for scripts piping from a secret manager, awkward if stdin is already the data. Whichever you choose: never echo the token, wrap it in a redacting type (Lesson 43), and document it. (From memory; I did not test platform-specific visibility.)

</details>

---

## 8. Debugging challenge

This tool is supposed to print the numbers `0` to `n-1`. Users report: a bare panic backtrace when they forget the argument, exit code 101 for a typo like `abc`, an error message in the output file when it fails, and a second panic when they pipe it into `head -1`. Fix all four.

```rust
fn main() {
    let args: Vec<String> = std::env::args().collect();
    let n: usize = args[1].parse().unwrap();
    for i in 0..n {
        println!("{i}");
    }
}
```

<details markdown="1">
<summary>Solution</summary>

Problems: (1) `args[1]` panics when the argument is missing (index out of bounds); (2) `parse().unwrap()` panics on bad input with exit code 101 and an unhelpful message; (3) the original reports must go to stderr with a documented non-zero code (a message in stdout would end up in the data file); (4) `println!` panics on a broken pipe. Fix: validate the arguments and return exit code 2 with a usage message on stderr, write through a locked `BufWriter`, and treat `BrokenPipe` as a normal end:

```rust
use std::io::{self, BufWriter, Write};
use std::process::ExitCode;

fn run(args: &[String], out: &mut dyn Write, err: &mut dyn Write) -> u8 {
    let Some(arg) = args.get(1) else {
        let _ = writeln!(err, "usage: count N");
        return 2;
    };
    let n: usize = match arg.parse() {
        Ok(n) => n,
        Err(e) => {
            let _ = writeln!(err, "count: invalid number '{arg}': {e}");
            return 2;
        }
    };
    let result = (0..n).try_for_each(|i| writeln!(out, "{i}")).and_then(|()| out.flush());
    match result {
        Ok(()) => 0,
        Err(e) if e.kind() == io::ErrorKind::BrokenPipe => 0,
        Err(e) => {
            let _ = writeln!(err, "count: cannot write output: {e}");
            1
        }
    }
}

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().collect();
    let mut out = BufWriter::new(io::stdout().lock());
    let mut err = io::stderr().lock();
    ExitCode::from(run(&args, &mut out, &mut err))
}
```

Checks (done by running the compiled program): no argument → usage on stderr, exit 2; `abc` → `invalid number 'abc': invalid digit found in string`, exit 2; `3` → `0 1 2`, exit 0; `count 100000000 | head -1` → one line, exit 0, nothing on stderr. Separating `run` from `main` also lets a test pass in-memory buffers, as `reclab` does.

</details>

---

## 9. Interview questions

1. *What belongs on stdout, what on stderr, and why do exit codes matter?*
2. *Why should `main` be thin?*
3. *What happens when a Rust program writes to a closed pipe, and how should a command-line tool respond?*
4. *What does `#[derive(Parser)]` generate from, and how do you test a `clap` interface?*
5. *What are the trade-offs between `deny_unknown_fields` and ignoring unknown fields?*
6. *Why can serializing a float lose information, and what does `serde_json` do with NaN?*
7. *How do you process a 100 GB file with a few MB of memory, and what limits do you apply to untrusted input?*
8. *What makes a serialized format a compatibility contract, and how do you evolve it?*

<details markdown="1">
<summary>Answers</summary>

1. stdout carries the result in a stable format so other programs can consume it; stderr carries diagnostics so they do not corrupt the data stream; the exit code is the only channel a shell, scheduler or CI job reads automatically, so it must distinguish success, failure and (if relevant) partial success, and be documented.
2. Logic in `main` can only be tested by spawning the process. A thin `main` that wires real streams and arguments to a function taking `args`, `stdin`, `stdout`, `stderr` (✓ `reclab::run`) makes every path testable in memory and quick.
3. Rust ignores `SIGPIPE`, so the write returns `BrokenPipe`; `println!` turns that into a panic (✓ exit 101). The tool should use `writeln!`/`write_all` with `?`, and exit quietly with 0 on `BrokenPipe` (the consumer chose to stop), reporting any other write error on stderr with a failure code.
4. The derive reads the struct, its field types, attributes and doc comments at compile time and generates the parser and the `--help` text. Test with `try_parse_from([...])`, which returns the error instead of exiting, and check the parsed values, the error's `exit_code()` and `render()` text; plus a few real-process tests for wiring.
5. `deny_unknown_fields` catches typos and enforces an exact format but breaks old readers when a writer adds a field. Ignoring unknown fields gives forward compatibility but hides typos (an optional field misspelled silently gets its default). A middle path: tolerate unknown fields and warn about them.
6. JSON has no NaN or infinity and one number type: `serde_json` writes NaN and infinity as `null` (✓), which does not read back as `f64`, and consumers that use 64-bit floats lose integer precision above 2⁵³ (✓ `9007199254740993` becomes `…992`). Validate finiteness, use integers (cents) or strings for exact values, and document the format.
7. Stream: read a bounded chunk or line, process, discard; keep only the aggregates the output needs (✓ 11.45 MB peak for a 137 MB file). For untrusted input apply limits at every layer: maximum line or document size (the `Take`-based reader), parser depth (✓ `recursion limit exceeded`), semantic bounds (string lengths, numeric ranges), and wide enough accumulators (✓ `u128` totals).
8. Other programs and old files depend on key names, enum spellings, number representations and optionality, so renaming a Rust field with `rename_all` or changing a type changes the contract. Evolve by adding optional fields with defaults, never reusing a name for a different meaning, tolerating unknown fields where independent upgrades occur, adding a version field when semantics change, and testing against saved sample files of old versions.

</details>

---

## 10. Practical challenge: `reclab`

Build the crate `reclab`. Requirements:

1. The `Order`/`Status` model with the serde attributes above, `parse_line` (types by serde, ranges by hand), and a `Summary` with per-status counts, a `u128` total and the top *n* customers (ties broken by name).
2. `read_line_bounded` (never buffers more than the limit plus a terminator; handles LF, CRLF and no final newline; resumes after a long line), `for_each_order` (blank lines ignored, line numbers include them, strict mode stops at the first rejection, invalid UTF-8 and over-long lines rejected rather than fatal) and `summarize`.
3. A `clap` interface with `summarize`, `validate`, `cat`, a shared option group, validated ranges and an enum for `--format`; `run(args, stdin, stdout, stderr) -> u8`; documented exit codes 0, 1, 2, 3; a closed output pipe is a quiet success.
4. Tests: serde behavior and error messages, boundaries (name length in characters, amount limit, every buffer size for the bounded reader), exact output text and JSON, exit codes, usage errors, help/version on stdout, the closed-pipe cases (in memory and with the real binary).
5. Your own mutation checks, reporting what survives and why.

<details markdown="1">
<summary>Solution</summary>

`Cargo.toml`:

```toml
[package]
name = "reclab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false

[dependencies]
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

`src/lib.rs`:

````rust
//! `reclab`: a small command-line tool that reads **JSON Lines** order files
//! (one JSON object per line), validates them, and summarizes them.
//!
//! The crate is split so that everything interesting is testable without
//! spawning a process:
//!
//! * the data model ([`Order`], [`Status`]) with `serde` attributes that
//!   define the file format,
//! * a bounded line reader ([`read_line_bounded`]) so one huge line cannot
//!   exhaust memory,
//! * [`summarize`], which folds a stream of lines into a [`Summary`],
//! * [`run`], which parses arguments with `clap` and writes to the streams it
//!   is given; `main` only wires it to the real standard streams.
//!
//! ```
//! use reclab::{parse_line, Status};
//!
//! let order = parse_line(
//!     r#"{"id":1,"customer":"ada","amountCents":250,"status":"paid"}"#,
//! )
//! .unwrap();
//! assert_eq!(order.status, Status::Paid);
//! assert!(parse_line(r#"{"id":1}"#).is_err());
//! ```

#![deny(missing_docs)]

use clap::{Args, Parser, Subcommand, ValueEnum};
use serde::{Deserialize, Serialize};
use std::collections::BTreeMap;
use std::ffi::OsString;
use std::fmt;
use std::fs::File;
use std::io::{self, BufRead, BufReader, Read, Write};
use std::path::{Path, PathBuf};

/// Longest customer name accepted by validation, in characters.
pub const MAX_CUSTOMER_CHARS: usize = 64;
/// Largest amount accepted by validation, in cents (ten billion dollars).
pub const MAX_AMOUNT_CENTS: u64 = 1_000_000_000_000;

/// Exit codes of the command-line tool.
pub mod exit {
    /// Everything succeeded.
    pub const OK: u8 = 0;
    /// A fatal error: unreadable file, I/O failure, or a bad line in `--strict` mode.
    pub const FAILURE: u8 = 1;
    /// Usage error (reported by `clap`).
    pub const USAGE: u8 = 2;
    /// The run finished, but some lines were rejected.
    pub const PARTIAL: u8 = 3;
}

/// Payment state of an order. Serialized as a lowercase string.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Status {
    /// Not yet paid.
    Pending,
    /// Paid in full.
    Paid,
    /// Money returned.
    Refunded,
}

/// One order. The file format is defined by these attributes: camelCase keys,
/// unknown keys rejected, `tags` optional.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct Order {
    /// Order number.
    pub id: u64,
    /// Customer name (1 to [`MAX_CUSTOMER_CHARS`] characters).
    pub customer: String,
    /// Amount in cents (at most [`MAX_AMOUNT_CENTS`]).
    pub amount_cents: u64,
    /// Free-form labels; may be omitted.
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    pub tags: Vec<String>,
    /// Payment state.
    pub status: Status,
}

/// Why a line was rejected.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct LineError {
    /// 1-based line number.
    pub line: usize,
    /// Human-readable reason.
    pub message: String,
}

impl fmt::Display for LineError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "line {}: {}", self.line, self.message)
    }
}

impl std::error::Error for LineError {}

/// Parses and validates one line of the file.
///
/// Syntax and type errors come from `serde_json` (with column numbers);
/// range checks that types cannot express are done here.
pub fn parse_line(text: &str) -> Result<Order, String> {
    let order: Order = serde_json::from_str(text).map_err(|e| e.to_string())?;
    let chars = order.customer.chars().count();
    if chars == 0 || chars > MAX_CUSTOMER_CHARS {
        return Err(format!(
            "customer must be 1 to {MAX_CUSTOMER_CHARS} characters, got {chars}"
        ));
    }
    if order.amount_cents > MAX_AMOUNT_CENTS {
        return Err(format!(
            "amountCents {} exceeds {MAX_AMOUNT_CENTS}",
            order.amount_cents
        ));
    }
    Ok(order)
}

/// What [`read_line_bounded`] found.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum LineRead {
    /// End of input; nothing was read.
    Eof,
    /// A line was read into the buffer (without its line terminator).
    Line,
    /// The line was longer than the limit; it was skipped and the buffer is empty.
    TooLong,
}

/// Reads one line into `buf` (cleared first), keeping **at most `max + 2`
/// bytes of it in memory** (the content plus room for a `\r\n` terminator).
/// A longer line is skipped to its end, so one enormous line cannot exhaust
/// memory. The terminator (`\n` or `\r\n`) is not stored and does not count
/// against `max`.
pub fn read_line_bounded<R: BufRead>(
    reader: &mut R,
    buf: &mut Vec<u8>,
    max: usize,
) -> io::Result<LineRead> {
    buf.clear();
    let limit = max as u64 + 2;
    let n = reader.by_ref().take(limit).read_until(b'\n', buf)?;
    if n == 0 {
        return Ok(LineRead::Eof);
    }
    let terminated = buf.last() == Some(&b'\n');
    // Without a newline we stopped either at the end of the input (fewer
    // bytes than the limit) or because the limit was reached.
    let hit_limit = !terminated && n as u64 == limit;
    if terminated {
        buf.pop();
    }
    if buf.last() == Some(&b'\r') && !hit_limit {
        buf.pop();
    }
    if buf.len() <= max && !hit_limit {
        return Ok(LineRead::Line);
    }
    buf.clear();
    if hit_limit {
        skip_rest_of_line(reader)?;
    }
    Ok(LineRead::TooLong)
}

/// Discards input up to and including the next newline without storing it.
fn skip_rest_of_line<R: BufRead>(reader: &mut R) -> io::Result<()> {
    loop {
        let chunk = reader.fill_buf()?;
        if chunk.is_empty() {
            return Ok(());
        }
        match chunk.iter().position(|&b| b == b'\n') {
            Some(i) => {
                reader.consume(i + 1);
                return Ok(());
            }
            None => {
                let len = chunk.len();
                reader.consume(len);
            }
        }
    }
}

/// Totals over all accepted orders.
#[derive(Debug, Clone, Default, PartialEq, Eq, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Summary {
    /// Non-empty lines seen.
    pub lines: usize,
    /// Lines accepted.
    pub accepted: usize,
    /// Lines rejected.
    pub rejected: usize,
    /// Sum of `amountCents` over accepted orders.
    pub total_cents: u128,
    /// Accepted orders per status (only statuses that occurred).
    pub by_status: BTreeMap<Status, usize>,
    /// Customers with the highest totals, largest first (ties by name).
    pub top_customers: Vec<CustomerTotal>,
}

/// One entry of [`Summary::top_customers`].
#[derive(Debug, Clone, PartialEq, Eq, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct CustomerTotal {
    /// Customer name.
    pub customer: String,
    /// Their total in cents.
    pub total_cents: u128,
}

/// Settings for [`summarize`].
#[derive(Debug, Clone, Copy)]
pub struct Options {
    /// Longest accepted line in bytes.
    pub max_line_bytes: usize,
    /// How many top customers to keep.
    pub top: usize,
    /// Stop at the first rejected line instead of skipping it.
    pub strict: bool,
}

/// Reads every line from `reader`, calling `on_error` for rejected lines and
/// `on_order` for accepted ones, and returns the counts of lines seen and
/// rejected. Blank lines are ignored. In strict mode it stops after the first
/// rejected line.
pub fn for_each_order<R: BufRead>(
    reader: &mut R,
    opts: &Options,
    mut on_order: impl FnMut(Order) -> io::Result<()>,
    mut on_error: impl FnMut(LineError),
) -> io::Result<(usize, usize)> {
    let mut buf = Vec::new();
    let (mut seen, mut rejected, mut number) = (0usize, 0usize, 0usize);
    loop {
        let read = read_line_bounded(reader, &mut buf, opts.max_line_bytes)?;
        if read == LineRead::Eof {
            break;
        }
        number += 1;
        let outcome: Result<Order, String> = match read {
            LineRead::TooLong => Err(format!("longer than {} bytes", opts.max_line_bytes)),
            _ => match std::str::from_utf8(&buf) {
                Err(_) => Err("not valid UTF-8".to_string()),
                Ok(text) if text.trim().is_empty() => continue,
                Ok(text) => parse_line(text),
            },
        };
        seen += 1;
        match outcome {
            Ok(order) => on_order(order)?,
            Err(message) => {
                rejected += 1;
                on_error(LineError {
                    line: number,
                    message,
                });
                if opts.strict {
                    break;
                }
            }
        }
    }
    Ok((seen, rejected))
}

/// Folds a whole input into a [`Summary`].
pub fn summarize<R: BufRead>(
    reader: &mut R,
    opts: &Options,
    on_error: impl FnMut(LineError),
) -> io::Result<Summary> {
    let mut summary = Summary::default();
    let mut per_customer: BTreeMap<String, u128> = BTreeMap::new();
    let (seen, rejected) = for_each_order(
        reader,
        opts,
        |o| {
            summary.accepted += 1;
            summary.total_cents += u128::from(o.amount_cents);
            *summary.by_status.entry(o.status).or_insert(0) += 1;
            *per_customer.entry(o.customer).or_insert(0) += u128::from(o.amount_cents);
            Ok(())
        },
        on_error,
    )?;
    summary.lines = seen;
    summary.rejected = rejected;
    let mut ranked: Vec<CustomerTotal> = per_customer
        .into_iter()
        .map(|(customer, total_cents)| CustomerTotal {
            customer,
            total_cents,
        })
        .collect();
    ranked.sort_by(|a, b| {
        b.total_cents
            .cmp(&a.total_cents)
            .then_with(|| a.customer.cmp(&b.customer))
    });
    ranked.truncate(opts.top);
    summary.top_customers = ranked;
    Ok(summary)
}

/// Command-line interface.
#[derive(Debug, Parser)]
#[command(
    name = "reclab",
    version,
    about = "Validate and summarize JSON Lines order files"
)]
pub struct Cli {
    #[command(subcommand)]
    command: Command,
}

#[derive(Debug, Subcommand)]
enum Command {
    /// Print totals and the top customers.
    Summarize(SummarizeArgs),
    /// Check every line and report the rejected ones.
    Validate(Common),
    /// Re-print accepted orders, one normalized JSON object per line.
    Cat(Common),
}

#[derive(Debug, Args)]
struct Common {
    /// Input file, or `-` for standard input.
    #[arg(default_value = "-")]
    file: PathBuf,
    /// Longest accepted line, in bytes.
    #[arg(long, default_value_t = 1_048_576, value_parser = clap::value_parser!(u32).range(16..=67_108_864))]
    max_line_bytes: u32,
    /// Stop at the first rejected line and exit with status 1.
    #[arg(long)]
    strict: bool,
}

#[derive(Debug, Args)]
struct SummarizeArgs {
    #[command(flatten)]
    common: Common,
    /// Output format.
    #[arg(long, value_enum, default_value_t = Format::Text)]
    format: Format,
    /// How many top customers to list.
    #[arg(long, default_value_t = 3, value_parser = clap::value_parser!(u8).range(1..=100))]
    top: u8,
}

#[derive(Debug, Clone, Copy, ValueEnum)]
enum Format {
    /// Human-readable text.
    Text,
    /// A single JSON object.
    Json,
}

fn open<'a>(file: &Path, stdin: &'a mut dyn BufRead) -> io::Result<Box<dyn BufRead + 'a>> {
    if file.as_os_str() == "-" {
        Ok(Box::new(stdin))
    } else {
        Ok(Box::new(BufReader::new(File::open(file)?)))
    }
}

fn render_text(s: &Summary) -> String {
    let mut out = String::new();
    out.push_str(&format!(
        "lines: {} (accepted {}, rejected {})\n",
        s.lines, s.accepted, s.rejected
    ));
    out.push_str(&format!(
        "total: {}.{:02}\n",
        s.total_cents / 100,
        s.total_cents % 100
    ));
    for (status, n) in &s.by_status {
        out.push_str(&format!("{status:?}: {n}\n").to_lowercase());
    }
    for (i, c) in s.top_customers.iter().enumerate() {
        out.push_str(&format!(
            "top {}: {} {}.{:02}\n",
            i + 1,
            c.customer,
            c.total_cents / 100,
            c.total_cents % 100
        ));
    }
    out
}

/// Runs the tool: parses `args` (the first is the program name), reads from
/// `stdin` or a file, writes results to `stdout` and diagnostics to `stderr`,
/// and returns the process exit code (see [`exit`]).
///
/// A closed output pipe (for example `reclab cat big | head -1`) is a normal
/// way for a consumer to say "enough": it ends the run with status 0 and no
/// message.
pub fn run<I, T>(
    args: I,
    stdin: &mut dyn BufRead,
    stdout: &mut dyn Write,
    stderr: &mut dyn Write,
) -> u8
where
    I: IntoIterator<Item = T>,
    T: Into<OsString> + Clone,
{
    let cli = match Cli::try_parse_from(args) {
        Ok(cli) => cli,
        Err(e) => {
            let text = e.render().to_string();
            let _ = if e.use_stderr() {
                write!(stderr, "{text}")
            } else {
                write!(stdout, "{text}")
            };
            return u8::try_from(e.exit_code()).unwrap_or(exit::USAGE);
        }
    };
    match execute(cli, stdin, stdout, stderr) {
        Ok(code) => match stdout.flush() {
            Ok(()) => code,
            Err(e) if e.kind() == io::ErrorKind::BrokenPipe => exit::OK,
            Err(e) => fatal(stderr, &format!("cannot write output: {e}")),
        },
        Err(e) if e.kind() == io::ErrorKind::BrokenPipe => exit::OK,
        Err(e) => fatal(stderr, &e.to_string()),
    }
}

fn fatal(stderr: &mut dyn Write, message: &str) -> u8 {
    let _ = writeln!(stderr, "reclab: error: {message}");
    exit::FAILURE
}

fn execute(
    cli: Cli,
    stdin: &mut dyn BufRead,
    stdout: &mut dyn Write,
    stderr: &mut dyn Write,
) -> io::Result<u8> {
    let (common, top) = match &cli.command {
        Command::Summarize(a) => (&a.common, usize::from(a.top)),
        Command::Validate(c) | Command::Cat(c) => (c, 0),
    };
    let opts = Options {
        max_line_bytes: common.max_line_bytes as usize,
        top,
        strict: common.strict,
    };
    let mut input = open(&common.file, stdin).map_err(|e| {
        io::Error::new(
            e.kind(),
            format!("cannot open '{}': {e}", common.file.display()),
        )
    })?;
    let mut warn = |e: LineError| {
        let _ = writeln!(stderr, "reclab: warning: {e}");
    };
    let rejected = match &cli.command {
        Command::Summarize(a) => {
            let s = summarize(&mut input, &opts, &mut warn)?;
            match a.format {
                Format::Text => stdout.write_all(render_text(&s).as_bytes())?,
                Format::Json => {
                    serde_json::to_writer(&mut *stdout, &s).map_err(io::Error::from)?;
                    stdout.write_all(b"\n")?;
                }
            }
            s.rejected
        }
        Command::Cat(_) => {
            let (_, rejected) = for_each_order(
                &mut input,
                &opts,
                |o| {
                    serde_json::to_writer(&mut *stdout, &o).map_err(io::Error::from)?;
                    stdout.write_all(b"\n")
                },
                &mut warn,
            )?;
            rejected
        }
        Command::Validate(_) => {
            let mut accepted = 0usize;
            let (seen, rejected) = for_each_order(
                &mut input,
                &opts,
                |_| {
                    accepted += 1;
                    Ok(())
                },
                &mut warn,
            )?;
            writeln!(
                stdout,
                "{seen} lines checked, {accepted} accepted, {rejected} rejected"
            )?;
            rejected
        }
    };
    Ok(match (rejected, common.strict) {
        (0, _) => exit::OK,
        (_, true) => exit::FAILURE,
        (_, false) => exit::PARTIAL,
    })
}

/// Reads all of `reader` into a string, refusing more than `limit` bytes.
/// Useful for small whole-file inputs such as configuration.
pub fn read_to_string_limited<R: Read>(reader: R, limit: u64) -> io::Result<String> {
    let mut text = String::new();
    reader.take(limit + 1).read_to_string(&mut text)?;
    if text.len() as u64 > limit {
        return Err(io::Error::new(
            io::ErrorKind::InvalidData,
            format!("input larger than {limit} bytes"),
        ));
    }
    Ok(text)
}
````

`src/main.rs`:

```rust
//! Thin binary: wires [`reclab::run`] to the real standard streams.

use std::io::{self, BufWriter};
use std::process::ExitCode;

fn main() -> ExitCode {
    let stdin = io::stdin();
    let mut input = stdin.lock();
    let mut output = BufWriter::new(io::stdout().lock());
    let mut errors = io::stderr().lock();
    let code = reclab::run(std::env::args_os(), &mut input, &mut output, &mut errors);
    ExitCode::from(code)
}
```

`tests/format.rs`:

```rust
use reclab::{
    LineRead, MAX_AMOUNT_CENTS, MAX_CUSTOMER_CHARS, Options, Order, Status, for_each_order,
    parse_line, read_line_bounded, read_to_string_limited, summarize,
};
use std::io::{BufRead, BufReader, Cursor};

const GOOD: &str = r#"{"id":7,"customer":"ada","amountCents":250,"status":"paid"}"#;

#[test]
fn parses_a_minimal_order_with_defaults() {
    let o = parse_line(GOOD).unwrap();
    assert_eq!(
        o,
        Order {
            id: 7,
            customer: "ada".into(),
            amount_cents: 250,
            tags: vec![],
            status: Status::Paid
        }
    );
}

#[test]
fn camel_case_keys_and_lowercase_statuses_are_the_format() {
    let o = parse_line(
        r#"{"id":1,"customer":"x","amountCents":1,"tags":["a","b"],"status":"refunded"}"#,
    )
    .unwrap();
    assert_eq!(o.tags, vec!["a", "b"]);
    assert_eq!(o.status, Status::Refunded);
    let err =
        parse_line(r#"{"id":1,"customer":"x","amount_cents":1,"status":"paid"}"#).unwrap_err();
    assert!(err.contains("unknown field `amount_cents`"), "{err}");
    let err = parse_line(r#"{"id":1,"customer":"x","amountCents":1,"status":"Paid"}"#).unwrap_err();
    assert!(err.contains("unknown variant `Paid`"), "{err}");
}

#[test]
fn serialization_omits_empty_tags_and_round_trips() {
    let o = parse_line(GOOD).unwrap();
    let text = serde_json::to_string(&o).unwrap();
    assert_eq!(
        text,
        r#"{"id":7,"customer":"ada","amountCents":250,"status":"paid"}"#
    );
    assert_eq!(parse_line(&text).unwrap(), o);
    let with_tags = Order {
        tags: vec!["t".into()],
        ..o
    };
    let text = serde_json::to_string(&with_tags).unwrap();
    assert!(text.contains(r#""tags":["t"]"#), "{text}");
    assert_eq!(parse_line(&text).unwrap(), with_tags);
}

#[test]
fn rejects_bad_json_with_a_position() {
    let err =
        parse_line(r#"{"id":1,"customer":"x","amountCents":oops,"status":"paid"}"#).unwrap_err();
    assert!(err.contains("column 38"), "{err}");
    assert!(parse_line("").is_err());
    assert!(parse_line("[]").is_err());
    assert!(
        parse_line(&format!("{GOOD} x"))
            .unwrap_err()
            .contains("trailing characters")
    );
    assert!(
        parse_line(r#"{"id":1,"id":2,"customer":"x","amountCents":1,"status":"paid"}"#)
            .unwrap_err()
            .contains("duplicate field `id`")
    );
}

#[test]
fn rejects_wrong_types_and_missing_fields() {
    let err =
        parse_line(r#"{"id":-1,"customer":"x","amountCents":1,"status":"paid"}"#).unwrap_err();
    assert!(err.contains("expected u64"), "{err}");
    let err = parse_line(r#"{"id":1,"customer":"x","status":"paid"}"#).unwrap_err();
    assert!(err.contains("missing field `amountCents`"), "{err}");
    let err =
        parse_line(r#"{"id":1.5,"customer":"x","amountCents":1,"status":"paid"}"#).unwrap_err();
    assert!(err.contains("floating point"), "{err}");
}

#[test]
fn large_ids_survive_exactly() {
    let o = parse_line(&format!(
        r#"{{"id":{},"customer":"x","amountCents":1,"status":"paid"}}"#,
        u64::MAX
    ))
    .unwrap();
    assert_eq!(o.id, u64::MAX);
    assert!(
        serde_json::to_string(&o)
            .unwrap()
            .contains("18446744073709551615")
    );
    assert!(
        parse_line(r#"{"id":18446744073709551616,"customer":"x","amountCents":1,"status":"paid"}"#)
            .is_err()
    );
}

#[test]
fn customer_length_is_counted_in_characters_not_bytes() {
    let line =
        |name: &str| format!(r#"{{"id":1,"customer":"{name}","amountCents":1,"status":"paid"}}"#);
    assert!(parse_line(&line(&"a".repeat(MAX_CUSTOMER_CHARS))).is_ok());
    assert!(parse_line(&line(&"a".repeat(MAX_CUSTOMER_CHARS + 1))).is_err());
    // Each é is two bytes but one character: 64 of them are fine.
    assert!(parse_line(&line(&"é".repeat(MAX_CUSTOMER_CHARS))).is_ok());
    assert!(parse_line(&line(&"é".repeat(MAX_CUSTOMER_CHARS + 1))).is_err());
    let err = parse_line(&line("")).unwrap_err();
    assert!(err.contains("got 0"), "{err}");
}

#[test]
fn amount_limit_is_inclusive() {
    let line = |n: u64| format!(r#"{{"id":1,"customer":"x","amountCents":{n},"status":"paid"}}"#);
    assert!(parse_line(&line(0)).is_ok());
    assert!(parse_line(&line(MAX_AMOUNT_CENTS)).is_ok());
    let err = parse_line(&line(MAX_AMOUNT_CENTS + 1)).unwrap_err();
    assert!(err.contains("exceeds"), "{err}");
}

fn read_all(input: &[u8], cap: usize, max: usize) -> Vec<(LineRead, Vec<u8>)> {
    let mut r = BufReader::with_capacity(cap, input);
    let mut buf = Vec::new();
    let mut out = Vec::new();
    loop {
        let k = read_line_bounded(&mut r, &mut buf, max).unwrap();
        if k == LineRead::Eof {
            return out;
        }
        out.push((k, buf.clone()));
    }
}

#[test]
fn bounded_reader_handles_terminators_and_the_final_line() {
    let got = read_all(b"ab\ncd\r\nef", 64, 10);
    let kinds: Vec<_> = got.iter().map(|(k, _)| *k).collect();
    assert_eq!(kinds, vec![LineRead::Line; 3]);
    assert_eq!(got[0].1, b"ab");
    assert_eq!(got[1].1, b"cd");
    assert_eq!(got[2].1, b"ef");
    assert!(read_all(b"", 64, 10).is_empty());
    assert_eq!(read_all(b"\n\n", 64, 10).len(), 2);
}

#[test]
fn bounded_reader_limit_is_exact() {
    // Exactly `max` content bytes (with or without a terminator) is accepted.
    for input in [&b"12345\nx"[..], b"12345"] {
        let got = read_all(input, 64, 5);
        assert_eq!(got[0], (LineRead::Line, b"12345".to_vec()));
    }
    // One byte more is too long.
    for input in [&b"123456\nx"[..], b"123456"] {
        let got = read_all(input, 64, 5);
        assert_eq!(got[0].0, LineRead::TooLong);
        assert!(got[0].1.is_empty());
    }
    // CRLF does not count against the limit.
    assert_eq!(
        read_all(b"12345\r\n", 64, 5)[0],
        (LineRead::Line, b"12345".to_vec())
    );
}

#[test]
fn bounded_reader_carriage_return_edge_cases() {
    // CR LF with content exactly at the limit, and one byte over.
    assert_eq!(
        read_all(b"12345\r\nz\n", 64, 5)[0],
        (LineRead::Line, b"12345".to_vec())
    );
    let over = read_all(b"123456\r\nz\n", 64, 5);
    assert_eq!(over[0].0, LineRead::TooLong);
    assert_eq!(
        over[1],
        (LineRead::Line, b"z".to_vec()),
        "the rest of the long line is skipped"
    );
    // A lone CR inside a line is data, not a terminator.
    assert_eq!(
        read_all(b"a\rb\n", 64, 5)[0],
        (LineRead::Line, b"a\rb".to_vec())
    );
    // A CR at the very end of the input is dropped like the first half of CRLF.
    assert_eq!(
        read_all(b"12345\r", 64, 5)[0],
        (LineRead::Line, b"12345".to_vec())
    );
    assert_eq!(read_all(b"123456\r", 64, 5)[0].0, LineRead::TooLong);
    // Every split of the input across the reader's internal buffer behaves the same.
    for cap in 1..9 {
        let got = read_all(b"12345\r\n123456\r\nok\r\n", cap, 5);
        let kinds: Vec<_> = got.iter().map(|(k, _)| *k).collect();
        assert_eq!(
            kinds,
            vec![LineRead::Line, LineRead::TooLong, LineRead::Line],
            "cap {cap}"
        );
        assert_eq!(got[2].1, b"ok");
    }
}

#[test]
fn bounded_reader_skips_a_long_line_and_resumes_on_the_next() {
    for cap in [1usize, 2, 3, 7, 64] {
        let mut input = vec![b'x'; 1000];
        input.extend_from_slice(b"\nok\n");
        let got = read_all(&input, cap, 10);
        assert_eq!(got.len(), 2, "cap {cap}");
        assert_eq!(got[0].0, LineRead::TooLong, "cap {cap}");
        assert_eq!(got[1], (LineRead::Line, b"ok".to_vec()), "cap {cap}");
    }
    // A long line at the very end, without a terminator.
    let got = read_all(&vec![b'y'; 500], 8, 10);
    assert_eq!(got, vec![(LineRead::TooLong, vec![])]);
}

#[test]
fn bounded_reader_never_buffers_more_than_the_limit() {
    // A 10 MB line with a limit of 16 bytes: the buffer must stay tiny.
    let big = vec![b'z'; 10_000_000];
    let mut r = BufReader::with_capacity(4096, &big[..]);
    let mut buf = Vec::new();
    let k = read_line_bounded(&mut r, &mut buf, 16).unwrap();
    assert_eq!(k, LineRead::TooLong);
    assert!(buf.capacity() < 1024, "capacity {}", buf.capacity());
}

fn opts(strict: bool) -> Options {
    Options {
        max_line_bytes: 200,
        top: 2,
        strict,
    }
}

#[test]
fn for_each_order_reports_line_numbers_and_skips_blank_lines() {
    let text = format!("{GOOD}\n\n   \nnot json\n{GOOD}\n");
    let mut r = Cursor::new(text.into_bytes());
    let (mut ok, mut errs) = (0, Vec::new());
    let (seen, rejected) = for_each_order(
        &mut r,
        &opts(false),
        |_| {
            ok += 1;
            Ok(())
        },
        |e| errs.push(e),
    )
    .unwrap();
    assert_eq!((seen, rejected, ok), (3, 1, 2));
    assert_eq!(errs.len(), 1);
    assert_eq!(errs[0].line, 4, "numbers count blank lines too");
    assert!(errs[0].to_string().starts_with("line 4: "));
}

#[test]
fn strict_mode_stops_at_the_first_bad_line() {
    let text = format!("{GOOD}\nbad\n{GOOD}\n{GOOD}\n");
    let mut r = Cursor::new(text.into_bytes());
    let mut ok = 0;
    let (seen, rejected) = for_each_order(
        &mut r,
        &opts(true),
        |_| {
            ok += 1;
            Ok(())
        },
        |_| {},
    )
    .unwrap();
    assert_eq!((seen, rejected, ok), (2, 1, 1));
}

#[test]
fn invalid_utf8_and_overlong_lines_are_rejected_not_fatal() {
    let mut input = Vec::new();
    input.extend_from_slice(GOOD.as_bytes());
    input.push(b'\n');
    input.extend_from_slice(b"{\"id\":\xff}\n");
    input.extend_from_slice(&vec![b'a'; 300]);
    input.push(b'\n');
    input.extend_from_slice(GOOD.as_bytes());
    let mut errs = Vec::new();
    let s = summarize(&mut Cursor::new(input), &opts(false), |e| errs.push(e)).unwrap();
    assert_eq!((s.lines, s.accepted, s.rejected), (4, 2, 2));
    assert_eq!(errs[0].message, "not valid UTF-8");
    assert_eq!(errs[1].message, "longer than 200 bytes");
    assert_eq!(errs[1].line, 3);
}

#[test]
fn summary_totals_statuses_and_ranking() {
    let lines = [
        r#"{"id":1,"customer":"bob","amountCents":500,"status":"paid"}"#,
        r#"{"id":2,"customer":"ada","amountCents":500,"status":"pending"}"#,
        r#"{"id":3,"customer":"cy","amountCents":900,"status":"paid"}"#,
        r#"{"id":4,"customer":"ada","amountCents":1,"status":"refunded"}"#,
    ];
    let s = summarize(&mut Cursor::new(lines.join("\n")), &opts(false), |_| {}).unwrap();
    assert_eq!(s.accepted, 4);
    assert_eq!(s.total_cents, 1901);
    assert_eq!(s.by_status[&Status::Paid], 2);
    assert_eq!(s.by_status[&Status::Pending], 1);
    assert_eq!(s.by_status[&Status::Refunded], 1);
    let names: Vec<_> = s
        .top_customers
        .iter()
        .map(|c| (c.customer.as_str(), c.total_cents))
        .collect();
    // cy 900, then ada 501 beats bob 500; only the top 2 are kept.
    assert_eq!(names, vec![("cy", 900), ("ada", 501)]);
}

#[test]
fn ties_in_the_ranking_are_broken_by_name() {
    let lines = [
        r#"{"id":1,"customer":"zed","amountCents":10,"status":"paid"}"#,
        r#"{"id":2,"customer":"amy","amountCents":10,"status":"paid"}"#,
    ];
    let s = summarize(&mut Cursor::new(lines.join("\n")), &opts(false), |_| {}).unwrap();
    assert_eq!(s.top_customers[0].customer, "amy");
}

#[test]
fn totals_do_not_overflow_u64() {
    let big =
        format!(r#"{{"id":1,"customer":"x","amountCents":{MAX_AMOUNT_CENTS},"status":"paid"}}"#);
    let text = std::iter::repeat_n(big, 30_000)
        .collect::<Vec<_>>()
        .join("\n");
    let o = Options {
        max_line_bytes: 200,
        top: 1,
        strict: false,
    };
    let s = summarize(&mut Cursor::new(text), &o, |_| {}).unwrap();
    assert_eq!(s.total_cents, 30_000u128 * u128::from(MAX_AMOUNT_CENTS));
    assert!(s.total_cents > u128::from(u64::MAX) / 1000);
}

#[test]
fn empty_input_is_a_valid_empty_summary() {
    let s = summarize(&mut Cursor::new(Vec::new()), &opts(false), |_| {}).unwrap();
    assert_eq!(
        (s.lines, s.accepted, s.rejected, s.total_cents),
        (0, 0, 0, 0)
    );
    assert!(s.by_status.is_empty() && s.top_customers.is_empty());
}

#[test]
fn limited_reader_refuses_oversized_input() {
    assert_eq!(read_to_string_limited(&b"hello"[..], 5).unwrap(), "hello");
    let err = read_to_string_limited(&b"hello!"[..], 5).unwrap_err();
    assert_eq!(err.kind(), std::io::ErrorKind::InvalidData);
    assert_eq!(read_to_string_limited(&b""[..], 0).unwrap(), "");
    let mut r = BufReader::new(&b"abc"[..]);
    let _ = r.fill_buf();
}
```

`tests/cli.rs`:

```rust
use reclab::{exit, run};
use std::io::{BufRead, BufReader, Cursor, Read, Write};
use std::path::PathBuf;
use std::process::{Command, Stdio};

const FILE: &str = concat!(
    r#"{"id":1,"customer":"ada","amountCents":1250,"status":"paid"}"#,
    "\n",
    r#"{"id":2,"customer":"bob","amountCents":999,"status":"pending","tags":["gift"]}"#,
    "\n",
    "\n",
    r#"{"id":3,"customer":"ada","amountCents":5000,"status":"refunded"}"#,
    "\n",
    r#"{"id":4,"customer":"cy","amountCents":1,"status":"paid","extra":true}"#,
    "\n",
    "not json at all\n",
    r#"{"id":6,"customer":"dee","amountCents":700,"status":"paid"}"#,
    "\n",
);

/// Runs the tool in memory. Returns (exit code, stdout, stderr).
fn cli(args: &[&str], stdin: &str) -> (u8, String, String) {
    let mut input = Cursor::new(stdin.as_bytes().to_vec());
    let (mut out, mut err) = (Vec::new(), Vec::new());
    let mut argv = vec!["reclab"];
    argv.extend_from_slice(args);
    let code = run(argv, &mut input, &mut out, &mut err);
    (
        code,
        String::from_utf8(out).unwrap(),
        String::from_utf8(err).unwrap(),
    )
}

#[test]
fn summarize_text_output_and_partial_exit_code() {
    let (code, out, err) = cli(&["summarize"], FILE);
    assert_eq!(code, exit::PARTIAL);
    assert_eq!(
        out,
        "lines: 6 (accepted 4, rejected 2)\ntotal: 79.49\npending: 1\npaid: 2\nrefunded: 1\n\
         top 1: ada 62.50\ntop 2: bob 9.99\ntop 3: dee 7.00\n"
    );
    assert_eq!(err.lines().count(), 2, "{err}");
    assert!(
        err.contains("reclab: warning: line 5: unknown field `extra`"),
        "{err}"
    );
    assert!(err.contains("reclab: warning: line 6: "), "{err}");
}

#[test]
fn summarize_json_output_is_one_valid_object_on_stdout_only() {
    let (code, out, _err) = cli(&["summarize", "--format", "json", "--top", "1"], FILE);
    assert_eq!(code, exit::PARTIAL);
    assert_eq!(out.lines().count(), 1);
    let v: serde_json::Value = serde_json::from_str(&out).unwrap();
    assert_eq!(v["accepted"], 4);
    assert_eq!(v["totalCents"], 7949);
    assert_eq!(v["byStatus"]["paid"], 2);
    assert_eq!(v["topCustomers"].as_array().unwrap().len(), 1);
    assert_eq!(v["topCustomers"][0]["customer"], "ada");
}

#[test]
fn clean_input_exits_zero_with_nothing_on_stderr() {
    let good = r#"{"id":1,"customer":"ada","amountCents":100,"status":"paid"}"#;
    let (code, out, err) = cli(&["summarize"], good);
    assert_eq!((code, err.as_str()), (exit::OK, ""));
    assert!(out.contains("total: 1.00"), "{out}");
    let (code, out, err) = cli(&["validate"], good);
    assert_eq!((code, err.as_str()), (exit::OK, ""));
    assert_eq!(out, "1 lines checked, 1 accepted, 0 rejected\n");
}

#[test]
fn validate_reports_every_bad_line_and_strict_stops_at_the_first() {
    let (code, out, err) = cli(&["validate"], FILE);
    assert_eq!(code, exit::PARTIAL);
    assert_eq!(out, "6 lines checked, 4 accepted, 2 rejected\n");
    assert_eq!(err.lines().count(), 2);
    let (code, out, err) = cli(&["validate", "--strict"], FILE);
    assert_eq!(code, exit::FAILURE);
    assert_eq!(out, "4 lines checked, 3 accepted, 1 rejected\n");
    assert_eq!(err.lines().count(), 1);
}

#[test]
fn cat_normalizes_and_emits_only_accepted_orders() {
    let (code, out, _) = cli(&["cat"], FILE);
    assert_eq!(code, exit::PARTIAL);
    let lines: Vec<_> = out.lines().collect();
    assert_eq!(lines.len(), 4);
    assert_eq!(
        lines[1],
        r#"{"id":2,"customer":"bob","amountCents":999,"tags":["gift"],"status":"pending"}"#
    );
    for l in lines {
        assert!(reclab::parse_line(l).is_ok(), "{l}");
    }
}

#[test]
fn usage_errors_exit_2_with_a_message_on_stderr() {
    for args in [
        &["bogus"][..],
        &[][..],
        &["summarize", "--top", "0"],
        &["summarize", "--top", "101"],
        &["summarize", "--top", "x"],
        &["summarize", "--format", "yaml"],
        &["summarize", "--max-line-bytes", "3"],
        &["summarize", "--nope"],
    ] {
        let (code, out, err) = cli(args, "");
        assert_eq!(code, exit::USAGE, "{args:?}");
        assert!(out.is_empty(), "{args:?}: {out}");
        assert!(!err.is_empty(), "{args:?}");
    }
    let (_, _, err) = cli(&["summarize", "--top", "0"], "");
    assert!(err.contains("0 is not in 1..=100"), "{err}");
    let (_, _, err) = cli(&["summarize", "--format", "yaml"], "");
    assert!(err.contains("[possible values: text, json]"), "{err}");
}

#[test]
fn help_and_version_go_to_stdout_and_exit_zero() {
    let (code, out, err) = cli(&["--help"], "");
    assert_eq!((code, err.as_str()), (exit::OK, ""));
    assert!(
        out.contains("Validate and summarize JSON Lines order files"),
        "{out}"
    );
    assert!(out.contains("summarize") && out.contains("validate") && out.contains("cat"));
    let (code, out, _) = cli(&["--version"], "");
    assert_eq!(code, exit::OK);
    assert_eq!(out, format!("reclab {}\n", env!("CARGO_PKG_VERSION")));
    let (code, out, _) = cli(&["summarize", "--help"], "");
    assert_eq!(code, exit::OK);
    assert!(
        out.contains("--format") && out.contains("--top") && out.contains("--strict"),
        "{out}"
    );
}

#[test]
fn missing_file_is_a_fatal_error_with_a_clear_message() {
    let (code, out, err) = cli(&["summarize", "/definitely/not/here.jsonl"], "");
    assert_eq!(code, exit::FAILURE);
    assert!(out.is_empty());
    assert!(
        err.starts_with("reclab: error: cannot open '/definitely/not/here.jsonl': "),
        "{err}"
    );
}

#[test]
fn max_line_bytes_option_rejects_long_lines() {
    let line = r#"{"id":1,"customer":"ada","amountCents":100,"status":"paid"}"#;
    let (code, _, err) = cli(&["validate", "--max-line-bytes", "20"], line);
    assert_eq!(code, exit::PARTIAL);
    assert!(err.contains("longer than 20 bytes"), "{err}");
    let (code, _, _) = cli(&["validate", "--max-line-bytes", "60"], line);
    assert_eq!(code, exit::OK, "the line is 59 bytes");
}

/// A writer that fails like a closed pipe after `limit` bytes.
struct BrokenPipeAfter(usize);

impl Write for BrokenPipeAfter {
    fn write(&mut self, buf: &[u8]) -> std::io::Result<usize> {
        if self.0 == 0 {
            return Err(std::io::ErrorKind::BrokenPipe.into());
        }
        let n = buf.len().min(self.0);
        self.0 -= n;
        Ok(n)
    }
    fn flush(&mut self) -> std::io::Result<()> {
        if self.0 == 0 {
            Err(std::io::ErrorKind::BrokenPipe.into())
        } else {
            Ok(())
        }
    }
}

#[test]
fn a_closed_output_pipe_is_a_quiet_success() {
    let mut input = Cursor::new(FILE.as_bytes().to_vec());
    let mut err = Vec::new();
    let code = run(
        ["reclab", "cat"],
        &mut input,
        &mut BrokenPipeAfter(10),
        &mut err,
    );
    assert_eq!(code, exit::OK);
    assert!(err.is_empty(), "{}", String::from_utf8_lossy(&err));
}

#[test]
fn a_closed_pipe_discovered_only_by_the_final_flush_is_also_a_quiet_success() {
    // Buffered writers often accept every write and report the closed pipe on flush.
    struct FailsOnFlush;
    impl Write for FailsOnFlush {
        fn write(&mut self, buf: &[u8]) -> std::io::Result<usize> {
            Ok(buf.len())
        }
        fn flush(&mut self) -> std::io::Result<()> {
            Err(std::io::ErrorKind::BrokenPipe.into())
        }
    }
    let good = r#"{"id":1,"customer":"ada","amountCents":5,"status":"paid"}"#;
    let mut input = Cursor::new(good.as_bytes().to_vec());
    let mut err = Vec::new();
    let code = run(["reclab", "cat"], &mut input, &mut FailsOnFlush, &mut err);
    assert_eq!(code, exit::OK);
    assert!(err.is_empty(), "{}", String::from_utf8_lossy(&err));
}

#[test]
fn another_write_error_is_reported_as_failure() {
    struct Full;
    impl Write for Full {
        fn write(&mut self, _: &[u8]) -> std::io::Result<usize> {
            Err(std::io::Error::other("disk full"))
        }
        fn flush(&mut self) -> std::io::Result<()> {
            Ok(())
        }
    }
    let mut input = Cursor::new(FILE.as_bytes().to_vec());
    let mut err = Vec::new();
    let code = run(["reclab", "cat"], &mut input, &mut Full, &mut err);
    assert_eq!(code, exit::FAILURE);
    assert!(String::from_utf8_lossy(&err).contains("disk full"));
}

// ---- real process tests -------------------------------------------------

fn tmp_file(name: &str, contents: &str) -> PathBuf {
    let path = std::env::temp_dir().join(format!("reclab-test-{}-{name}", std::process::id()));
    std::fs::write(&path, contents).unwrap();
    path
}

#[test]
fn the_real_binary_reads_stdin_and_returns_exit_codes() {
    let exe = env!("CARGO_BIN_EXE_reclab");
    let mut child = Command::new(exe)
        .args(["validate", "-"])
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()
        .unwrap();
    child
        .stdin
        .take()
        .unwrap()
        .write_all(FILE.as_bytes())
        .unwrap();
    let out = child.wait_with_output().unwrap();
    assert_eq!(out.status.code(), Some(3));
    assert_eq!(
        String::from_utf8_lossy(&out.stdout),
        "6 lines checked, 4 accepted, 2 rejected\n"
    );
    let bad = Command::new(exe).arg("bogus").output().unwrap();
    assert_eq!(bad.status.code(), Some(2));
    assert!(bad.stdout.is_empty());
}

#[test]
fn the_real_binary_survives_a_reader_that_quits_early() {
    let mut big = String::new();
    for i in 0..100_000 {
        big.push_str(&format!(
            r#"{{"id":{i},"customer":"c{}","amountCents":{},"status":"paid"}}"#,
            i % 100,
            i % 777
        ));
        big.push('\n');
    }
    let path = tmp_file("big.jsonl", &big);
    let mut child = Command::new(env!("CARGO_BIN_EXE_reclab"))
        .arg("cat")
        .arg(&path)
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()
        .unwrap();
    let mut first = String::new();
    {
        let mut out = BufReader::new(child.stdout.take().unwrap());
        out.read_line(&mut first).unwrap();
        // Dropping the reader closes the pipe while the tool still has output.
    }
    let mut err = String::new();
    child
        .stderr
        .take()
        .unwrap()
        .read_to_string(&mut err)
        .unwrap();
    let status = child.wait().unwrap();
    std::fs::remove_file(&path).ok();
    assert!(first.starts_with(r#"{"id":0,"#), "{first}");
    assert_eq!(status.code(), Some(0), "stderr: {err}");
    assert_eq!(err, "");
}
```

`examples/naive_cat.rs`:

```rust
//! The naive way: `println!` per line. Try: `naive_cat file | head -1`.
use std::io::{self, BufRead};

fn main() {
    let stdin = io::stdin();
    for line in stdin.lock().lines() {
        println!("{}", line.expect("read error"));
    }
}
```

`examples/sigpipe_demo.sh`:

```bash
#!/usr/bin/env bash
# Compare a naive filter with reclab when the reader quits early.
set -u
bin=target/debug
file=$1
echo "== naive println! loop | head -1"
$bin/examples/naive_cat < "$file" | head -1
echo "exit status of naive_cat: ${PIPESTATUS[0]}"
echo
echo "== reclab cat | head -1"
$bin/reclab cat "$file" | head -1
echo "exit status of reclab: ${PIPESTATUS[0]}"
```

✓ Checks I ran: `cargo fmt`, `cargo clippy --all-targets -- -D warnings` (clean), and `cargo test --locked`: `tests/cli.rs` 14 tests, `tests/format.rs` 21 tests, 1 doctest, all passing, plus the experiments quoted in the lesson.

✓ **Mutation checks.** I applied 29 hand-made mutations one at a time to a copy of the crate and ran the full suite (`cargo test --locked`). Of 29: 27 were caught, 1 was an equivalent mutant (R8), and R17 survived the first suite and was fixed by a new test (so 28 caught after the fix, counting R17). R13 ran with a corrected pattern. Mutants that make most tests fail also left the early-quitting-reader process test hanging (every line fails, so the child blocks writing warnings to an unread stderr pipe); I stopped those runs by timeout, and the failing tests are what I counted.

| # | Mutation | Result |
|---|---|---|
| R1 | customer length `>` → `>=` the limit | caught: characters-not-bytes boundary test |
| R2 | drop the "empty customer" check | caught |
| R3 | amount `>` → `>=` the limit | caught: inclusive-limit test |
| R4 | count bytes instead of characters | caught |
| R5 | read limit `max + 2` → `max + 1` | caught: CRLF edge cases |
| R6 | accept `len < max` instead of `<=` | caught: exact-limit test |
| R7 | do not consume the newline when skipping | caught |
| R8 | `hit_limit` uses `>=` instead of `==` | **equivalent**: `take(limit)` never returns more than `limit` bytes, so `==` and `>=` are the same |
| R9 | strict mode no longer stops | caught |
| R10 | blank-line check without `trim` | caught |
| R11 | cast the amount to `u32` before summing | caught |
| R12 | reverse the tie-break by name | caught |
| R13 | rank ascending instead of descending | caught |
| R14 | do not truncate to `top` | caught |
| R15 | strict failure returns 3 instead of 1 | caught |
| R16 | closed pipe during execution is a failure | caught (in-memory and real-process tests) |
| R17 | closed pipe on the final flush is a failure | **survived** the first suite; caught after adding `a_closed_pipe_discovered_only_by_the_final_flush_is_also_a_quiet_success` |
| R18 | cents formatted with `% 10` | caught |
| R19 | `--top` range allows 0 | caught |
| R20 | `--max-line-bytes` minimum 16 → 2 | caught |
| R21 | remove `deny_unknown_fields` | caught |
| R22 | remove `rename_all = "camelCase"` | caught (10 tests fail) |
| R23 | remove `rename_all = "lowercase"` on `Status` | caught (9 or more tests fail) |
| R24 | serialize empty `tags` | caught |
| R25 | `-` is no longer standard input | caught |
| R26 | warning prefix changed | caught |
| R27 | never strip a trailing `\r` | caught |
| R28 | remove `default` on `tags` | caught (9 tests fail) |
| R29 | status names not lowercased in the text report | caught |

The lesson of R17: my first suite tested a writer that fails *during* writes but not one that fails only when flushed, which is exactly how a `BufWriter` behaves (it accepts writes into memory and learns about the closed pipe later). The test double has to fail where the real thing fails.

**Design notes.**

- `run` takes `&mut dyn BufRead`/`&mut dyn Write` rather than being generic: the tool is not performance-critical at this boundary and trait objects keep the signature simple (Lesson 21). The inner loops are generic over `BufRead`.
- `read_line_bounded` returns `LineRead::{Eof, Line, TooLong}` instead of an error for over-long lines, so one hostile line is a *rejected line*, not a fatal error.
- Validation lives in one function (`parse_line`) so range rules are not scattered.
- `Summary.total_cents` is `u128` (§4.6). `serde_json` serializes `u128` as a number (✓ tested through the JSON output), which a 64-bit-float consumer would round if it ever exceeded 2⁵³.
- Customer totals are held in a `BTreeMap`, so memory grows with the number of *distinct customers*; a hostile file with millions of distinct names grows it without bound. A production tool would cap the number of distinct keys or switch to an on-disk or approximate structure.
- Not handled: gzip or other compressed input, CSV or other formats, colored output and terminal detection, shell completions, localized messages, configuration files and environment variables, reading several files in one run, Windows path and console behavior, and non-UTF-8 customer names (they are rejected as invalid UTF-8).

</details>

---

## 11. Key takeaways

- A CLI is an interface: results on stdout, diagnostics on stderr, meaningful documented exit codes, and `-`/stdin for composition. Keep `main` thin and test `run(args, stdin, stdout, stderr)` in memory.
- A closed pipe is normal: `println!` panics on it (✓ exit 101), a filter built on `writeln!` and `io::Result` exits 0 quietly (✓).
- `clap`'s derive gives parsing, validation (✓ ranges, enums), help and exit code 2 from declarations, and `try_parse_from` makes it testable. It costs binary size and compile time (✓ 1.36 MB and 14.4 s here), so choose consciously.
- Serde attributes are your file format. Know the defaults (✓ unknown fields ignored, `Option` accepts missing and `null`), the error messages (✓ position and expected type), and the traps: NaN becomes `null` (✓), integers above 2⁵³ are fragile in other languages (✓ `…993` became `…992` as `f64`), `untagged` errors say nothing (✓).
- Untrusted input needs limits at every layer: bounded line length (✓ about 11.3 to 11.5 MB for 13.7 MB and 137 MB files), parser depth (✓ recursion limit), semantic ranges, and wide accumulators (✓ `u128`). Boundary tests find the off-by-ones (✓ the CRLF bug).
- Don't pass secrets as arguments; don't trust output formats to be private contracts; evolve formats with optional fields and defaults.

**Connections.** [Lesson 43](43_observability_ci_supply_chain.md) gives stderr discipline, redaction and log forging, and the CI and dependency checks `reclab` should ship with; [Lesson 42](42_performance_engineering.md) explains the binary-size knobs used in §3.1; [Lesson 41](41_benchmarking_profiling.md) is the measurement method; [Lesson 35](35_proc_macros.md) explains what the derives expand to; [Lesson 26](26_error_design.md) and [Lesson 14](14_option_result_errors.md) cover error values and `?`. Next: Lesson 45 covers HTTP clients and servers and API design.

---

## 12. References

- `clap` documentation and derive tutorial (`Parser`, `Subcommand`, `Args`, `ValueEnum`, `value_parser!`, `try_parse_from`): <https://docs.rs/clap>
- Command Line Interface Guidelines (stdout and stderr, exit codes, help, composability): <https://clig.dev/>
- The Rust Command Line Book (arguments, exit codes, output, signals): <https://rust-cli.github.io/book/>
- `serde` documentation and attribute reference (container, variant and field attributes): <https://serde.rs/>
- `serde_json` documentation (`from_str`, `to_writer`, `Value`, `StreamDeserializer`, recursion limit, features): <https://docs.rs/serde_json>
- JSON Lines format description: <https://jsonlines.org/>
- RFC 8259, "The JavaScript Object Notation (JSON) Data Interchange Format" (numbers, NaN, interoperability advice about integers): <https://www.rfc-editor.org/rfc/rfc8259>
- `std::io::BufRead` and `std::io::Read::take`: <https://doc.rust-lang.org/std/io/trait.BufRead.html>
- `std::process::ExitCode`: <https://doc.rust-lang.org/std/process/struct.ExitCode.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: that Rust sets `SIGPIPE` to ignore at start-up (the observable behavior, `BrokenPipe` and the panic, is ✓); that stdout is line-buffered (I did not test it); command-line visibility of arguments in `ps` and `/proc`, and what shell history stores; environment-variable inheritance and exposure claims in exercise 8; `pico-args`, `lexopt` and `argh` as alternatives and their sizes (not run); that other languages (JavaScript, for one) store JSON numbers as 64-bit floats; CSV formula injection; the `indexmap`-based order-preserving feature and the feature flag that disables the recursion limit; Windows and macOS behavior of anything here; and the idea that `Cli::parse()` exits the process (I used `try_parse_from` throughout). The measurements marked ✓ are from one shared virtual machine: timings are the minimum (or minimum and median) of 7 runs per command with the input in the page cache, peak memory is the resident set size reported by the operating system's `wait4` accounting, and binary sizes are for this toolchain and these dependency versions. Exercise answers 1, 5 and 8 are reasoning from memory; the code blocks in answers 3 and 4 and in the debugging challenge were compiled and run (the challenge's four behaviors by running the program). Everything marked ✓ was run on this machine's `rustc 1.97.0` with `clap 4.6.7`, `serde 1.0.229` and `serde_json 1.0.154`.
