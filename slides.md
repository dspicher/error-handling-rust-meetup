---
title: Please `std::ops::Try` this at home
sub_title: Rust Meetup 13.03.25 @ BFH Biel
author: Dominik
---

Rust Error Handling
---
<!-- jump_to_middle -->
<!-- column_layout: [1,4, 1] -->
<!-- column: 1 -->

* Unrecoverable vs. recoverable errors
* `enum Result {}` and its goodies
* Tricks for `Err(E)`
* `?` & `std::ops::Try`
* The `std::error::Error` trait
* Infallibility
* Useful crates

<!-- end_slide -->

Unrecoverable vs. recoverable errors
---

* RFC 236: Conventions around error handling in Rust
* Two basic strategies for dealing with errors:
  * Task failure (`panic!()`) with stack unwinding
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
```rust +exec
fn fallible() -> Result<(), ()> {
    Ok(())
}

fn main() {
    fallible();
}
```


<!-- end_slide -->

Rust vs. Python
---
```rust +exec
use std::fs::File;
use std::path::Path;

fn main() {
    let maybe_file = File::open(Path::new("inexistent.txt"));
    println!("{maybe_file:?}");
}
```

<!-- pause -->
Compare with 
```python +exec
from pathlib import Path
contents = Path("inexistent.txt").read_text()
```
<!-- end_slide -->

What `Result` doesn't involve
---
<!-- jump_to_middle -->
<!-- column_layout: [1,4, 1] -->
<!-- column: 1 -->
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
<!-- jump_to_middle -->
<!-- column_layout: [1,4, 1] -->
<!-- column: 1 -->
`E` is an arbitrary, unconstrained type

=> flexible fallibility patterns

Quiz time!

<!-- end_slide -->

`Err(E)` tricks: Creating an array from mis-sized vector
---

```rust +exec
use std::convert::TryInto;

fn main() {
    let v = vec![1, 2, 3];
    let err: Result<[u8; 4], _> = v.try_into();
    eprintln!("{:?}", err);
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

`Err(E)` tricks: Binary search miss on a slice
---

```rust

impl [T] {
    fn binary_search(&self, &x) -> Result<usize, _>
}
```

```rust +exec
fn main() {
    let s = [0, 2, 3, 5, 8, 13];
    assert_eq!(s.binary_search(&2),  Ok(1));
    println!("{:?}", s.binary_search(&7))
}
```

<!-- pause -->
```rust
impl [T] {
    fn binary_search(&self, &x) -> Result<usize, usize>
}
```

<!-- end_slide -->

`Err(E)` tricks: Unsuccessful update of atomic types
---

```rust
impl AtomicU8 {
    fn compare_exchange(&self, current: u8, new: u8, ...) -> Result<u8, _>
}
```

```rust +exec
use std::sync::atomic::{AtomicU8, Ordering};

fn main() {
    let some_var = AtomicU8::new(5);
    assert_eq!(some_var.compare_exchange(5, 10, Ordering::SeqCst, Ordering::SeqCst), Ok(5));
    assert_eq!(some_var.load(Ordering::Relaxed), 10);
    println!("{:?}", some_var.compare_exchange(6, 12, Ordering::SeqCst, Ordering::SeqCst));
}
```

<!-- pause -->
```rust
impl AtomicU8 {
    fn compare_exchange(&self, current: u8, new: u8, ...) -> Result<u8, u8> {
        ...
        if old == current { Ok(old) } else { Err(old) }
    }
}
```

<!-- end_slide -->

`Err(E)` tricks: Locking a poisoned `Mutex`
---

```rust
impl Mutex<T> {
    fn lock(&self) -> LockResult<MutexGuard<'_, T>>
}

enum LockResult<T> {
    Ok(T),
    Err(_),
}
```

```rust +exec
use std::sync::{Arc, Mutex};

fn main() {
    let mutex = Arc::new(Mutex::new(41));
    let mutex_clone = Arc::clone(&mutex);
    let handle = std::thread::spawn(move || {
        let mut data = mutex_clone.lock().unwrap();
        *data += 1;
        panic!("");
    });
    let _ = handle.join();
    dbg!(*mutex.lock().unwrap_err().into_inner());
}
```

<!-- pause -->
```rust
enum LockResult<T> {
    Ok(T),
    Err(PoisonError<T>), // you have still acquired the mutex!
}
```

<!-- end_slide -->
`std::error::Error`
---

```rust
pub trait Error: Debug + Display {
    // Provided methods
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... } // deprecated
    fn description(&self) -> &str { ... } // deprecated
    fn cause(&self) -> Option<&dyn Error> { ... }
    fn provide<'a>(&'a self, request: &mut Request<'a>) { ... } // nightly
}
```
<!-- pause -->
* Almost a marker trait
  * See e.g. boring implementations for `std::io::Error` or `std::fmt::Error`
* Minimal expectations for errors: being able to print themselves
  * `Debug` (for developer) + `Display` (for user)
<!-- end_slide -->


`?` and `std::ops::Try`
---

`?` propagates error to the caller with an in-between `Into` step.

```rust
fn foo() -> Result<(), u16> {
    Err::<(), _>(0_u8)?;
    Ok(())
}
```
<!-- pause -->
`?` (also works for `Option`!) is powered by the nightly `std::ops::Try` trait:
```rust
pub trait Try: FromResidual {
    type Output;
    type Residual;

    // Required methods
    fn from_output(output: Self::Output) -> Self;
    fn branch(self) -> ControlFlow<Self::Residual, Self::Output>;
}
```
<!-- pause -->

allowing e.g.:
```rust
trait Iterator {
    fn try_fold<B, F, R>(&mut self, init: B, mut f: F) -> R
    where
        Self: Sized,
        F: FnMut(B, Self::Item) -> R,
        R: Try<Output = B>,
}
```
<!-- end_slide -->


Infallibility
---
```rust
pub enum Infallible {}
```

Where a `Result` is always `Ok`, e.g.:
```rust
impl<T, U> TryFrom<U> for T where U: Into<T>
```

<!-- pause -->
```rust +exec
fn main() {
    match Ok::<u8, std::convert::Infallible>(2) {
        Ok(_) => println!("ok"),
        // Err(_) => println!("err"),
    }
}
```
<!-- pause -->

`Infallibe` has the same role as the currently unstable "never" type `!`.

<!-- pause -->

Aside: empty impls are the most beautiful:
```rust
// in std::mem

/// This function is not magic
pub fn drop<T>(_x: T) {}
```
<!-- end_slide -->



Useful crates
---

`anyhow` for applications

```rust
impl<E> From<E> for Error
where
    E: StdError + Send + Sync + 'static,
```

allows you to `?` on any `std::error::Error` when returning an `anyhow::Error` result

check out the `eyre` fork (and `color-eyre`)!
<!-- pause -->


`thiserror` for libraries:

"a convenient derive macro for the standard library’s `std::error::Error` trait"

<!-- end_slide -->


<!-- jump_to_middle -->
Thanks!
---
