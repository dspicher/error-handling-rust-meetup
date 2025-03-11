---
title: Please `std::ops::Try` this at home
sub_title: Rust Meetup 13.03.25 @ BFH Biel
author: Dominik
---

Rust Error Handling
---

* Unrecoverable vs. recoverable errors
* `enum Result {}` and lots of goodies
* A brief Python excursion
* `?` & `std::ops::Try`
* The `std::error::Error` trait
* Infallibility
* Useful crates

<!-- end_slide -->

Unrecoverable vs. recoverable errors
---

* RFC 236: Conventions around error handling in Rust
* Rust provides two basic strategies for dealing with errors:
  * Task failure (`panic!()`), which involves stack unwinding
    * For catastrophic errors (e.g. out of memory) and contract violations (e.g. out of bounds indexing)
    * Recovered at a "coarse grain" (the task boundary)
<!-- pause -->
  * The `Result` type
    * For "obstructions" (e.g. file not found)
    * Recovered at a "fine grain" (the call site)
<!-- pause -->
* This talk is only about the second one!

<!-- end_slide -->

Rust's `Result` type
---

The central error type is a "normal", double-generic, trivial? enum!

```rust {2-7}
#[must_use]
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```
<!-- pause -->
Getting the result requires acknowledging the possiblity of an error:
```rust
let res = some_fallible_operation();
res.unwrap();
res.unwrap_or_else(|| ...);
match res { ... }
```
<!-- pause -->
Even if you don't want the result, the compiler will warn you:
```rust
some_fallible_operation(); // unused `Result` that must be used
```

<!-- pause -->
Compare with 
```python
from pathlib import Path
contents = Path("inexistent.txt").read_text()
```

<!-- end_slide -->

What `Result` doesn't involve
---
* Special language support
  * `?` is just syntactic sugar
<!-- pause -->
* Control flow impact
  * No `try {} catch {}`
<!-- pause -->
* Stack unwinding which can be expensive
<!-- pause -->
* The `std::error::Error` trait 🙃

<!-- end_slide -->

Some neat examples of `Err(E)`
---

`E` is an arbitrary, unconstrained type.

=> flexible fallibility patterns

Quiz time!

<!-- end_slide -->

Turning a vector into a fixed-size array
---

```rust
impl TryFrom<Vec<T>> for [T; N] {
    fn try_from(mut vec: Vec<T>) -> Result<[T; N], _>;
}
```

<!-- pause -->
```rust
impl TryFrom<Vec<T>> for [T; N] {
    fn try_from(mut vec: Vec<T>) -> Result<[T; N], Vec<T>>;
}
```
<!-- pause -->

Similar patterns:
 * `String::from_utf8(vec: Vec<u8>) -> Result<String, FromUtf8Error>` with `FromUtf8Error::into_bytes`
 * `OsString::into_string(self) -> Result<String, OsString>`
 * ...

<!-- end_slide -->

Binary search on a slice
---

```rust
[T]::binary_search(&x) -> Result<usize, _>
```

<!-- pause -->
```rust
[T]::binary_search(&x) -> Result<usize, usize>
```

(insertion index)

<!-- end_slide -->

Update of an `AtomicBool`
---

```rust
AtomicBool::compare_exchange(&self, current: bool, new: bool, ...) -> Result<bool, _>
```

<!-- pause -->
```rust
AtomicBool::compare_exchange(&self, current: bool, new: bool, ...) -> Result<bool, bool> {
    ...
    if old == current { Ok(old) } else { Err(old) }
}
```
