---
date: 2024-02-05
---

Rust uses macros for situations where a variable number of arguments is needed (no *function overloading* and *default arguments*)

Macros being *hygienic* means they don't accidentally capture identifiers from the scope they are used in. Rust macros are actually only partially hygienic (local variables, labels, `$crate`).

Benefits of Rust:

- *Compile time memory safety* (C++ RAII)
  - No uninitialized variables.
  - No double-frees.
  - No use-after-free.
  - No `NULL` pointers.
  - No forgotten locked mutexes.
  - No data races between threads.
  - No iterator invalidation.

- *No undefined runtime behavior* - what a Rust statement does is never left unspecified
  - Array access is bounds checked.
  - Integer overflow is defined (panic, wrap-around, saturating).

- *Modern language features* - as expressive and ergonomic as higher-level languages
  - Enums and pattern matching.
  - Generics.
  - No overhead FFI.
  - Zero-cost abstractions.
  - Great compiler errors.
  - Built-in dependency manage.
  - Built-in support for testing.
  - Excellent Language Server Protocol support.

Variables:

- Type must be known at compile time
- Immutable by default (add `mut` to override) 
- Type inference.

Types:

- Signed integers `i<N>`: `i8`, `i16`, `i32`, `64`, `i128`, `isize`
- Unsigned integers `u<N>`: `u8`, `u16`, `u32`, `64`, `u128`, `usize`
- Floating point numbers: `f32`, `f64`
- Unicode scalar values: `char`
- Booleans: `bool`

Width of types:

- `isize` and `usize` are the width of a pointer.
- `char` is 4 bytes wide.
- `bool` is 1 byte wide.

Blocks:

A block in Rust contains a sequence of expressions, enclosed by braces `{}`. Each block has a value and a type, which are those of the last expression of the block. If the last expression ends with `;`, then the resulting value is `()`.

Macros:

Macros are expanded into Rust code during compilation, and can take a variable number of arguments.