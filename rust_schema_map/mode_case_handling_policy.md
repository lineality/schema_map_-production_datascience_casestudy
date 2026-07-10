### Modes & Case Handling Framework
2026 Geoffrey Gordon Ashbrook


#### 'Power of 10' Style Modes and case-handling in Rust:

# Compile-Mode-Differentiated Case & Error Handling Policy & Framework: A Three-Mode System using Fieldless Error-Codes

Note: One of many terminology problems here is around the jargon for 'production' 'release' builds of code and products. In the wider world 'production' does not have the same meaning that it has in IT and software development. In film, for example, 'in production' means the film is still being shot, and 'post-production' refers to work after shooting scenes and before the film is finished to show in theaters (such as final editing, effects, etc.). For software, data-science, and cloud-deployments, 'production' is the conventional jargon term meaning 'in real-world use' where there are often phases 'development' 'staging' and 'production' environments. 'Production' software is the type used by real users, live in the real world. The phrase 'in production' is common to refer to either the production tech-environment or to 'production' meaning real-world-use, or both. (It not clear who decided to describe software that is finished and being used in the real world as 'still in production.')

Rust, and likely other languages, have a similar range of phases (and phrases), such as testing with cargo-tests, debug-builds, and another jargon-term "release" that is likely comparable to 'production.'

While not ideal, as the average person on the street would have no idea what 'production-release' is supposed to mean, I will try to consistently use the term production-release to refer to the area of final for-real-world-use code/software and product. This hopefully bridges the standard software 'production'  jargon and the Rust 'release' jargon.


## Three goals (in priority order):
1. Production-release never panics, never halts, never leaks data. Handle and move on.
2. Minimal memory-use in production: no heap in production-release error paths, no error strings compiled into production-release binaries.
3. Ideally: Every error is traceable to one exact location in the code.



## 1. The Three Modes of Output & Case/Error Handling
- Test Mode
- Debug Mode
- Production Mode

The same function source-code compiles into three behaviors. The function
signature and the returned error VALUE are byte-identical in all three
builds. Only side effects differ.

| Mode | Build flag | Panics? | Heap? | Output |
|---|---|---|---|---|
| **Test** (`cargo test`) | `#[cfg(test)]` | yes — `assert!` **in the test function only** | allowed | verbose; asserts on returned error values |
| **Debug** (debug build, not test) | #[cfg(all(debug_assertions, not(test)))] | invariant: yes (debug_assert!); input-validation: no | allowed | invariant: panic + message; input-validation: eprintln! + error returned |
| **Production** (release) | always-compiled path | **never** | **never** | terse 2-byte error code returned to caller; no terminal print |

Note: This framework assumes there is no additional foot-shot action such as turning debug-compilation (or test) on in your production-release.
```rust
// Please do not do this:
[profile.release]
debug-assertions = true
```

### Rules for Three-Modes

- `debug_assert!` in function bodies must be gated
  `#[cfg(all(debug_assertions, not(test)))]` so it cannot collide with tests
  of the production-release path.
- Verbose diagnostics at a failure site are `eprintln!` gated
  `#[cfg(debug_assertions)]`. They may use heap (`format!`, `.to_string()`)
  freely — they are excluded from production-release code.
- The production-release path (the `if`-check and the `return Err(...)`) is ALWAYS
  compiled, in every build. It is the only path that is used in production-release for real-world use.
- `assert!` for tests can live inside cargo tests: `#[cfg(test)] mod` test functions. A
  body test-assert panics before the production-release catch
  path runs, making the production-release path untestable. Tests can assert on
  the RETURNED error value (e.g. `assert_eq!(err, YourProjectError::ThisCondition)`), which
  exercises the production-release path directly.


### Disambiguation & Vocabulary

The original 2006 'Power of 10' rules contain a confusing use of the term "assert" regarding a number of critical valid points vital to the guidance in the paper.

The 2006 'Power of 10' rules use the single word "assert" when describing three importantly-differing meanings and situations (without providing a way to specify the differences):

1. The assert() macro, which halts the program on failure;
2. The general verb "assert a condition" — to check that something must hold, independent of what happens on failure;
3. A check that, on failure, takes a recovery action (e.g. returns an error) — the opposite of halting.

The 2006 power-of-10 rules demand all three at once:
1. Tight limits on conditional compilation — even though the standard assert is implemented by the preprocessor and DOES halt (case 1)
2. At least two "assertions" per function (case 2)
3. Never Halting in Production: Assertions that take a recovery action rather than halt (case 3)

Such terminology can lead to confusion (e.g. where you are both required to "assert" and prohibited from "assert"ing, because the term "assert" does not have one specific & consistent meaning), especially as there are no clarifying annotations for any use (let along all uses) of the term in the original 2006 power-of-10 paper. The spirit of the paper is clear, but the details become a quagmire.

The term "assert" cannot easily be used precisely here, so this framework aims to consistently use three descriptive terms instead:

1. **required-condition**: the rule itself (e.g. `len <= 32`), independent of build.
2. **requirement-check**: the code that tests a required-condition (the always-compiled `if`).
3. **reaction-to-check**: what happens upon failure, which varies by mode:
  test → panic in the test; debug → panic / verbose eprintln; production-release → terse code returned (possibly a retry-first) (never panic, never halt, never stop: the satellite must not fall out of the sky).

Similarly, this framework also aims to clarify the semantics around "errors" in the same context, where technically an "error" does not refer to all required ways of handling a ~case (in this context), and where there are three importantly distinct modes for how 'case-handling must (or must-not) be done. We need to clarify:

1. the build/compilation mode (production-release / debug / test), and
2. the difference between a case (any condition that must be handled) and an error (a case that is handled by returning Err).

Unfortunately there is no single common-language and technical-term that covers 'error-like-things' across all three modes that we need to manage: test-mode, debug-mode, production-release-mode. The somewhat bulky phrase 'cases that need to be handled' may be the closest we can come, though even that lacks an emphasis on erroneous or problematic-case handling (vs. every part of code that "handles" some logic in a neutral sense). In casual language the word "error" may be the simplest and most accurate term, but as with the term "assert", the word 'error' has been given specific technical meanings that make planning and communication fraught. Which meaning of 'assert' or 'error' is someone using in every instance? Colliding-terms make confusion unavoidable.

Not every "handled case" is technically an "error." The same case may be silently ignored, re-tried, or (in test mode) turned into a 'panic.' Keeping "case" and "error" distinct is a critical part of the source-code management that is implicit in the 2006 power-of-10 guidance, managing specifically different per-mode behaviors in the same 'cases.' We should strive to make more of the implicit wisdom of the spirit of the power-of-10 more explicit and robust for systems-programming and software education more broadly. There is a low probability of all readers of the original 2006 document following a uniform framework with a shared disambiguated lexicon.

## Defensive Programming: "expected" issues vs. "should-not-happen-check"

Another area that can be confusing is 'errors' or 'cases' that logically should-not happen but that in the real world (often) do happen. For whatever reason there is no standard terminology for this concept, and so important defensive-programming checks often appear to be silly mistakes made by a developer (why check for something that in a perfect world cannot happen?) and are then removed by later developers (and then the software crashes in production).

A solution that hopefully covers this is to standardize two
types of checks. We have "input-validation" that covers the range of 'expected-type'
problems (problems that we know are possible), such as a value being outside of an acceptable range.
And we have the "should-not-happen-check" for so-called "unexpected" problems (problems that are impossible in an ideal-fictional-world).

Another reason to define these two is to standardize where it may be usually best
to do each type of check.


### Input Validation ("expected") vs. Internal Invariant ("should-not-happen-check")

- **Input validation** (e.g. bad user input, missing file, oversized argument) :
  an EXPECTED case. Requirement-check + terse return
  only, plus debug eprintln.
  Default debugging for input-validation should be non-fatal (eprintln + return) to provide debug-data without panic-halting.
  Case by case, if a panic-backtrace provides the data needed then panic-assert (for debug) is an option.

- **Internal invariant** (a value that in a perfect-world 'cannot happen' but that in real life can be the result of a bitflip, hardware failure, adversarial attack, undefined behavior, or other corruption): all three modes apply — debug_assert, test assertion on the
  returned value, and the always-compiled production-release catch. a defensive should-not-happen-check

#### Classification heuristic to decide which pattern to use:
Ask the question "Can this condition fail when the code is correct and memory is sound?"
- Yes -> Input validation ("expected" type)
- No -> Internal invariant ("should-not-happen-check")



## 2. The Error Type: One Fieldless Enum

The project error type is a single fieldless `enum` with explicit `u16`
discriminants. The discriminant IS the error-code.


#### Single, Central YourProjectError Enum Across Module-Files
The ABCProjectError enum is defined once (e.g., in any .rs module file) and is
imported wherever errors need to be returned or handled. All modules use the
same enum variants, so error codes are globally unique and consistent.


```rust
// Debug printing (variant names as text) is compiled in for TEST and DEBUG
// builds only. In a production release build no Debug impl exists, so no
// variant-name strings are baked into the binary — and any stray `{:?}`
// on a YourProjectError in production-release code fails to compile (intended guard).
#[cfg_attr(any(test, debug_assertions), derive(Debug))]

// Always available, in every build:
// Copy/Clone — the error is a trivially-copyable 2-byte value (no heap).
// PartialEq/Eq — lets callers and tests compare codes (`err == YourProjectError::ThisCondition`,
// `assert_eq!`, `matches!`).
#[derive(Copy, Clone, PartialEq, Eq)]

// Fixes the in-memory representation to a u16 so the discriminant IS the
// 2-byte error-code, making `self as u16` well-defined and the size exactly
// 2 bytes.
#[repr(u16)]
pub enum YourProjectError {
    // io categories (mapped from io::ErrorKind at failure sites)
    IoInterrupted = 1,          // retryable
    IoWouldBlock  = 2,          // retryable
    IoTimedOut    = 3,          // retryable
    IoNotFound         = 50,
    IoPermissionDenied = 51,
    IoOther            = 99,

    // one block of codes per file/struct/feature — see Error-Code-Table rules below
    Fs32tFromStrInputTooLong   = 101,
    Fs32tAsStrLenExceedsBuffer = 102,
    Fs32tAsStrInvalidUtf8      = 103,

    RetryMaxAttemptsZero = 200,
}

impl YourProjectError {
    /// The terse numeric error-code. Available in ALL builds. No heap.
    pub fn code(self) -> u16 { self as u16 }
}

pub type Result<T> = std::result::Result<T, YourProjectError>;
```

### Reasons for using fieldless (strict properties that should not be changed)

- `Copy`, 2 bytes. No heap anywhere in the error path.
- NO `String` payloads. NO `io::Error` payloads (`io::Error` is not `Copy`
  and can itself hold a heap string containing paths/PII).
- The compiler enforces both name uniqueness and discriminant-value uniqueness, so no two variants can collide on a name or a code. (See https://doc.rust-lang.org/error_codes/E0081.html)
- Callers can `match` exhaustively.
- Variant payloads are forbidden. If a caller needs a fact (a length, a
  position), the fact justifies a distinct variant, not a payload.
  Payloads are the leak vector this policy exists to close.


### Error-Code-Table rules (in the enum's doc comment)

1. **APPEND-ONLY.** A code number is never reused or renumbered, ever.
   A code in a years-old field log must still mean one thing.
2. A file/struct/feature can get a reserved hundred-block
   (e.g. `100–199 FixedSize32Timestamp`). Blocks are listed in the enum doc.
3. Variant names are `AcronymFunctionCondition` (e.g.
   `Fs32tFromStrInputTooLong`) — unique, traceable, greppable.
4. The number to meaning table lives in the enum's doc comment. It is
   DOCUMENTATION, not shipped binary data.

### Display: debug builds only

```rust
/// Human-readable text — DEBUG/TEST BUILDS **ONLY**. Never ships.
#[cfg(debug_assertions)]
impl std::fmt::Display for YourProjectError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let text = match self {
            YourProjectError::Fs32tFromStrInputTooLong => "FS32T-101 from_str: input too long",
            // ... one arm per variant ...
        };
        write!(f, "{}", text)
    }
}
```

Implicit in the power-of-10 goals for production-release software is an interesting unique-per-project design question: for software that never stops, halts, panics, etc., how is 'error' information reported? If there is a UI or terminal display, the information may be printed to terminal (without heap) or shown in an info-bar. Or the error-data may be logged or saved to a database. Also, information about the error or 'case-handled' should be as terse as possible, this to manage memory (and avoid heap) but also to prevent any exfiltration of potentially sensitive, private, or security-related data.

Production-Release binaries therefore contain no error strings.
A production-release-build log or an info-bar shown to the user can show the error-code-number (e.g. `101`). The number can be looked-up in the Error-Code-Table.

Render the number without heap:

```rust
/// u16 → decimal ASCII in a stack buffer. No heap. Panic-free.
fn u16_to_decimal(mut value: u16, buf: &mut [u8; 5]) -> &str {
    let mut start = buf.len();
    if value == 0 { start -= 1; buf[start] = b'0'; }
    else {
        while value > 0 && start > 0 {
            start -= 1;
            buf[start] = b'0' + (value % 10) as u8;
            value /= 10;
        }
    }
    std::str::from_utf8(&buf[start..]).unwrap_or("?")
}
```


### Error-Code Rendering responsibility: The boundary renders. Functions return the error-code but do not render.

Error-producing functions return the error-code, they never print, never log, never touch an 'info bar' in production. Converting a code to human-facing output (u16_to_decimal to 'info bar,' or the log writer) happens at exactly one place per output surface: the top-level handler that owns state, and the logging facility. Rationale: one rendering site per surface means one place to audit for PII and one place to change formatting.


## 3. Retry-or-Permanent Is Defined Per Error-Code (not number-ranges)

Transience (whether an error is retryable) is a property of the error code,
defined using a single explicit method (for a single source of truth).
Behavior is defined in one place, making it easy to audit and update.

The is_retryable() method is required to exhaustively list which errors are
"transient" (able to be retried).
- Avoids silent misclassification.
- Requires developers to decide for each error whether it is retryable.


```rust
impl YourProjectError {
    /// Transient failure worth retrying?
    /// EXPLICIT list. Deliberately NOT inferred from number ranges:
    /// range rules ("codes < 50 are retryable") silently mis-classify
    /// appended codes and nothing fails to compile. Numbers are identity
    /// only; behavior lives here, in readable code.
    pub fn is_retryable(self) -> bool {
        match self {
            YourProjectError::IoInterrupted
            | YourProjectError::IoWouldBlock
            | YourProjectError::IoTimedOut => true,

            YourProjectError::IoNotFound
            | YourProjectError::IoPermissionDenied
            | YourProjectError::IoOther
            | YourProjectError::Fs32tFromStrInputTooLong
            | ... => false,
            // NO wildcard -> adding a variant is
            // a compile error until classified
    }
}
}
```

Retry loops branch on the output of is_retryable() (bounded loop, production-release catch for bad arguments):

```rust
fn run_with_retry(max_attempts: u32) -> Result<T> {
    // Requirement-check: max_attempts >= 1.
    // INPUT VALIDATION — produces code 200, never panics.
    // This guard guarantees the loop runs at least once, so `attempt >= max_attempts`
    // cannot be spuriously true on the first iteration.
    if max_attempts == 0 {
        #[cfg(debug_assertions)]
        eprintln!("RETRY-200: run_with_retry: max_attempts == 0");
        return Err(YourProjectError::RetryMaxAttemptsZero);
    }

    let mut attempt = 1;
    loop {
        match operation() {
            Ok(v) => return Ok(v),
            Err(e) if attempt >= max_attempts => return Err(e), // exhausted: return last error
            Err(e) if e.is_retryable() => { /* sleep / backoff */ attempt += 1; }
            Err(e) => return Err(e),                            // permanent: fail fast
        }
    }
}
```

### The io boundary rule

`io::Error` is converted to a terse code at the failure site.
`io::Error` (and any path/PII text inside it) is DROPPED at the failure site.
Only `ErrorKind` is kept, as a discriminant.

Two rules for mapping:

1. Retry-relevant kinds get distinct codes. IoOther is non-retryable.

2. `IoOther` is an explicit policy bucket, not a catch-all
   `IoOther` means "no distinct code assigned, and **non-retryable**."
   Mapping a kind to `IoOther` = no retrying.
   If sometime should be retried, it needs its own code.

`io::ErrorKind` is `#[non_exhaustive]`, so the wildcard arm (`_ => IoOther`) is
**required** by the compiler — new kinds can appear in future std versions.
This means you cannot rely on the compiler to force classification of a *newly
added std kind* (unlike your own enum, where the no-wildcard `match` in
`is_retryable` does force it). E.g. Re-audit mapping when updating Rust version.

```rust
impl YourProjectError {
    /// io::Error → terse code. The io::Error (and any path/PII text inside
    /// it) is DROPPED here; only the kind survives as a discriminant.
    ///
    /// Classification note: kinds mapped to `IoOther` are treated as
    /// NON-RETRYABLE by policy (see `is_retryable`). Kinds that should be
    /// retried MUST get their own code, not `IoOther`.
    pub fn from_io_kind(kind: io::ErrorKind) -> YourProjectError {
        match kind {
            // --- transient / retryable: distinct codes required ---
            io::ErrorKind::Interrupted       => YourProjectError::IoInterrupted,
            io::ErrorKind::WouldBlock        => YourProjectError::IoWouldBlock,
            io::ErrorKind::TimedOut          => YourProjectError::IoTimedOut,
            io::ErrorKind::ConnectionReset   => YourProjectError::IoConnectionReset,
            io::ErrorKind::ConnectionAborted => YourProjectError::IoConnectionAborted,
            io::ErrorKind::BrokenPipe        => YourProjectError::IoBrokenPipe,

            // --- permanent: distinct codes for common cases ---
            io::ErrorKind::NotFound          => YourProjectError::IoNotFound,
            io::ErrorKind::PermissionDenied  => YourProjectError::IoPermissionDenied,
            io::ErrorKind::AlreadyExists     => YourProjectError::IoAlreadyExists,
            io::ErrorKind::InvalidInput      => YourProjectError::IoInvalidInput,
            io::ErrorKind::InvalidData       => YourProjectError::IoInvalidData,
            io::ErrorKind::UnexpectedEof     => YourProjectError::IoUnexpectedEof,

            // --- explicit policy bucket: NO distinct code, NON-RETRYABLE ---
            // Required wildcard (io::ErrorKind is #[non_exhaustive]).
            // Everything here is a deliberate "treat as permanent" decision.
            _                                => YourProjectError::IoOther,
        }
    }
}
```

#### Arbitrary examples of Corresponding additions to the enum's io block:
(Append new numbers, do not renumber.)

```rust
    // io categories (mapped from io::ErrorKind at failure sites)
    IoInterrupted      = 1,   // retryable
    IoWouldBlock       = 2,   // retryable
    IoTimedOut         = 3,   // retryable
    IoConnectionReset  = 4,   // retryable
    IoConnectionAborted = 5,  // retryable
    IoBrokenPipe       = 6,   // retryable

    IoNotFound         = 50,
    IoPermissionDenied = 51,
    IoAlreadyExists    = 52,
    IoInvalidInput     = 53,
    IoInvalidData      = 54,
    IoUnexpectedEof    = 55,
    IoOther            = 99,   // explicit policy bucket: non-retryable by rule
```

Matching arms in `is_retryable`:
(no-wildcard `match` forces these to be classified)

```rust
    pub fn is_retryable(self) -> bool {
        match self {
            YourProjectError::IoInterrupted
            | YourProjectError::IoWouldBlock
            | YourProjectError::IoTimedOut
            | YourProjectError::IoConnectionReset
            | YourProjectError::IoConnectionAborted
            | YourProjectError::IoBrokenPipe => true,

            YourProjectError::IoNotFound
            | YourProjectError::IoPermissionDenied
            | YourProjectError::IoAlreadyExists
            | YourProjectError::IoInvalidInput
            | YourProjectError::IoInvalidData
            | YourProjectError::IoUnexpectedEof
            | YourProjectError::IoOther
            /* | ...remaining non-io variants... */ => false,
            // NO wildcard -> adding a variant is a compile error until classified.
        }
    }
```

### Notes:
- Retry classifications above are arbitrary examples, not framework-rules
- `io::ErrorKind` is `#[non_exhaustive]`,  it needs the human re-audit


#### More Disambiguation: "Error Site" & "Catch"
Naming things is hard, so "error site" frequently has one
(hopefully just one) of two different meanings:
1. The file, function, or line of code where an issue, failure or bug occurred.
2. The place in the code where an issue is detected (and responded-to / handled etc.).


A detection point, as meant here, is the place in a function where a case is detected and handled, for example where the code decides "this is wrong, so we should return error code N."

The commonly used term "Catch" is sometimes roughly equivalent in spirit to how
'detect' is used here, but details vary.

JS/TS, Java, and c# use the common 'Try / Catch / finally' system.
cpp and php use only 'Try / Catch'

Zig uses the terms 'try' and 'catch' but the meanings are different:
- zig try: immediately propagate any error up the stack (return that error)
```zig
const value = try mightFail(false);
```

- zig catch can work in a few different ways:
1. "catch" an error to replace it with a specified default value.
If the function-call works, great!
If the function call fails, then we should do this other thing instead.
```zig
const a: u32 = addTwenty(44) catch 22;␤
```
2. "catch" the error to run a block of code when that happens.
const x = mightFail(true) catch |err| {
// run this code block if it fails
}

As with the word 'assert,' it is important to note collisions between one or more
jargon and common-use meanings.


## 4. Two Standardized Patterns for Detection

We will define two patterns, because there are two ways that code
can detect an issue (such as an 'error' or a 'case to handle'):

Both patterns cover the three-modes of behavior (test / debug / production-release).
Consistently using these two patterns should help reviews, audits, and readability.

1. The If-Detection-Pattern:
Use an if statement where you can ask as an 'if' question:
E.g. "is this length ≤ 32?" or "is this denominator zero?"


2. The Match-A-Function-Call-Pattern
Use a 'match' for functions that can fail (and, coding defensively,
all functions in this system should return a result).
E.g.  Some function you called told you that it failed.
You called a function such as from_utf8 or opened a file
and the check happened inside that function.
The function you called handed you a 'success-or-failure' result.


### Never Use '?'
Use of '?' is in tension or conflict with at least two of this framework's  goals / mechanisms (per-site variants, mode-gated diagnostics). Using '?' silently reintroduces untraceable, unaudited propagation. Real life tends to be full of strange rare edge cases, but even so I am not aware:
- There are no areas where '?' is required-rigor in Rust.
- '?' is always a convenience-shortcut that is not compatible with this framework.

Matching each call with a clear error pointing to that case improves transparency and communication.


### The If-Detection-Pattern
This pattern is for checks that are a condition written as an if statement.
e.g. ("is X length ≤ 42?", "is Y denominator 0?")

// --- Check: <the required condition, in words> ---
// <input validation | internal invariant> — <one line: why this check exists>

// Internal invariants ONLY (omit this line for input validation):
// panics in debug builds; compiled out of test and production-release builds.
#[cfg(all(debug_assertions, not(test)))]
debug_assert!(condition, "ACRO-NNN: fn_name: detail {}", value);

if !condition {
    // Debug/test diagnostic. May format text (heap OK) because this line
    // is compiled out of production-release builds entirely.
    #[cfg(debug_assertions)]
    eprintln!("ACRO-NNN: fn_name: detail: {}", value);

    // Compiled in EVERY build; the only response that exists in
    // production-release: return the 2-byte code. No print, no heap.
    return Err(YourProjectError::AcroFnCondition);
}

Every detection-site gets a cargo test asserting the returned value:

```rust
#[test]
fn acro_fn_rejects_condition_with_code_nnn() {
    let err = the_function(bad_input).unwrap_err();
    assert_eq!(err, YourProjectError::AcroFnCondition);
    assert_eq!(err.code(), NNN);
}
```

Tests for a function's detection sites should live in the same file as the function,
in that file's `#[cfg(test)] mod <filename>_tests` block, so the
Error-Code-Table doc-comment, the detection sites, and their tests are auditable.

Test names should follow the form
`<acronym>_<fn>_<condition>_with_code_<nnn>`
so that failures can be more easily and clearly traced.


### Testing invariant catch paths (unreachable via the public API)

A production-release catch for an internal invariant is normally unreachable
through public constructors — that is what makes it an invariant. Its
test must construct the invalid state DIRECTLY, which is possible
because tests live in the same file/module and can access
private fields:

```rust
#[test]
fn fs32t_as_str_catches_corrupt_len_with_code_102() {
    // Construct an invalid value directly (same-module private access).
    // This simulates post-construction corruption (e.g. bitflip).
    let corrupt = FixedSize32Timestamp { data: [0u8; 32], len: 99 };
    let err = corrupt.as_str().unwrap_err();
    assert_eq!(err, YourProjectError::Fs32tAsStrLenExceedsBuffer);
    assert_eq!(err.code(), 102);
}
```

Do not add public test-only constructors or pub(crate) fields to enable other tests, that would widen the API surface that the invariant protects.


### The Match-A-Function-Call-Pattern: failure detected via a returned `Result` (not a testable bool)

When the required-condition is checked BY a fallible call (e.g.
`std::str::from_utf8`, a parse, a syscall wrapper), the requirement-check
is the `match` on that call's result (no requirement to test the condition
separately with a duplicate check).

#### Two different match-handling error values:

The Match-A-Function-Call-Pattern involves two different error values of different kinds. This is important for clear handling, for naming and communication, and (if relevant) for heap and memory management.


#### Match-error type 1: The error-code for this system.
This is more clearly no-heap, slim, and predictable.
In every build-mode, only this 2-byte error-code leaves the function.


#### Match-error type 2: The callee's error, which depends on what function is called. Hopefully it is usually standard a small fixed-size struct the fallible function returns on failure.
E.g. the Utf8Error is two integers, and stack-only)
```rust
struct Utf8Error {
    valid_up_to: usize,      // the byte position where the input stopped being valid UTF-8
    error_len: Option<u8>,   // length of the invalid byte sequence
}
```
But this is just one clean example, the details are up to each function.
The most memory-use-defensive we can be in production is to '_details' whatever this error is (note: even if memory-use is not a concern, this error-data might contain privacy or even PII related information that should not be exfiltrated in production builds).


The callee-function's error value has whatever type that function declared. Many std errors are small stack-only structs, but other std library, and especially 3rd party, errors may own heap allocated inside the callee.

When using any standard-library or third-party function not built 'natively' within a project, heap-use is likely:
- io::Error can sometimes contain an N-length message (using heap)
- String`, `Box<dyn Error>` & `anyhow::Error` can use heap
- Custom library errors with `String`/`Vec` fields can/will use heap.

This framework does not / cannot prevent a dependency from using heap,
but it can drop what is returned and allocates nothing further to heap.


All three modes live in the
`Err` arm:

```rust
match fallible_call(input) {
    Ok(value) => Ok(value),   // or continue using `value`
    Err(_detail) => {
        // Debug Mode: panic in debug builds ONLY (not test, not production-release).
        // `debug_assert!(false, ...)` is the standard way to panic from
        // inside an Err arm; it compiles out of production-release builds.
        #[cfg(all(debug_assertions, not(test)))]
        debug_assert!(false, "ACRO-NNN: fn_name: {}", _detail);

        // Verbose diagnostic, heap OK, compiled out of production-release.
        // For an internal invariant this runs in TEST builds only: in a debug build
        // the debug_assert!(false, …) above panics first, so execution never reaches here.
        #[cfg(debug_assertions)]
        eprintln!("ACRO-NNN: fn_name: {}", _detail);


        // Production Mode: terse code. No heap. No PII. No print.
        Err(YourProjectError::AcroFnCondition)
    }
}
```


**Input-validation version** — identical to the above, but with the `debug_assert!(false, …)` line removed, because a failure on untrusted input is expected, not a bug, for where Debug should not panic:

```rust
// INPUT VALIDATION version — for a call that can legitimately fail on
// untrusted input (e.g. from_utf8 on external bytes). Debug must NOT panic.
match fallible_call(input) {
    Ok(value) => Ok(value),
    Err(_detail) => {
        // NO debug_assert!(false, …) here — this failure is expected, not a bug.

        // Verbose diagnostic, heap OK, compiled out of production-release.
        #[cfg(debug_assertions)]
        eprintln!("ACRO-NNN: fn_name: {}", _detail);

        // Production Mode: terse code. No heap. No PII. No print.
        Err(YourProjectError::AcroFnCondition)
    }
}
```




## 5. Things Production-Release Code Must Not Do

### Banned in Production-Release Error Paths

- `format!`, `to_string`, `String`, `Box<dyn Error>`, any use of heap
- `io::Error` stored inside the project error type.
- `unwrap` / `expect` / `panic!` / body `assert!`.
- Error text containing paths, file contents, env vars, user data, PII,
  or implementation detail. (Production-Release ships no error text at all.)
- Number-range behavior rules ("codes 1–49 mean X").
- Reusing or renumbering an existing error-code.

### Avoiding Production-Release Panic
(Note, this section is likely not 100% exhaustive.)

These operations also panic with no opt-out (so, in production as well), so they should be avoided or handled correctly when used in case-handling:
let x = buf[i];        // panics if i >= buf.len()   — ALL profiles
let s = &data[a..b];   // panics if a > b or b > len — ALL profiles
let q = num / den;     // panics if den == 0         — ALL profiles
let r = num % den;     // panics if den == 0         — ALL profiles

When indexing, slicing, or dividing in production-release, use non-panicking versions of operations: .get() / .get_mut() instead of [], checked_div / checked_rem instead of / and %, split_at_checked instead of split_at.

#### Plain Arithmetic Operators
Rust's plain arithmetic operators (`+`, `-`, `*`, `<<`, `-x` negation, etc.) have build-profile-dependent behavior that is controlled by `overflow-checks` setting:

| Profile | `overflow-checks` Default | Behavior on Overflow |
|---|---|---|
| dev / debug | `true` | **panics** ("attempt to add with overflow") |
| release / production | `false` | **silently wraps** (two's-complement wraparound) |

With default settings, using plain arithmetic operators:
- In debug build, `255u8 + 1` panics.
- In production-release build, `255u8 + 1` evaluates to `0` and execution continues with a wrong value.

So: use checked_add / checked_sub / checked_mul / checked_shl in production.
