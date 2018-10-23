# `BitVec` – Managing memory bit by bit

This crate provides packed bit-level analogues to `[T]` and `Vec<T>`. The slice
type `BitSlice` and the vector type `BitVec` allow bitwise access to a region of
memory in any endian ordering or underlying primitive type. This permits
construction of space-efficient sets or fine-grained control over the values in
a region of memory.

`BitVec` is a strict expansion of `BitSlice` to include allocation management.
Since `BitVec` is shorter to type, the rest of this document will use it by
default, and mark out sections that apply *only* to the vector type and not to
the slice type. Unless marked, assume that the text applies to both.

`BitVec` is generic over an ordering cursor, using the trait `Endian`, and the
primitive type, using the trait `Bits`. This means that `BitVec` structures can
be built with a great deal of flexibility over how they manage their memory and
translate between the in-memory representation and their semantic contents.

`BitVec` acts as closely to a standard `Vec` as possible, and can be assumed by
default to be what a `Vec<u1>` would be if such a type were possible to express
in Rust. It has stack semantics, in that push and pop operations take place only
on one end of the `BitVec`’s buffer. It supports iteration, bitwise operations,
and rendering for `Display` and `Debug`.

## Usage

I wrote this crate because I was unhappy with the other bit-vector crates
available. I specifically need to manage raw memory in bit-level precision, and
this is not a behavior pattern the other bit-vector crates made easily available
to me. This served as the guiding star for my development process on this crate,
and remains the crate’s primary goal.

To this end, the default type parameters for the `BitVec` type use `u8` as the
storage primitive and use big-endian ordering of bits: the forwards direction is
from MSb to LSb, and the backwards direction is from LSb to MSb.

To use this crate, you need to depend on it in `Cargo.toml`:

```toml
[dependencies]
bitvec = "0.6"
```

and include it in your crate root `src/main.rs` or `src/lib.rs`:

```rust,no-run
#[macro_use]
extern crate bitvec;

use bitvec::*;
```

> Note: For 2018 edition Rust, use `"0.6.0-e2018"` as your version string, and
> elide the `#[macro_use]` import directive. The `use bitvec::*;` import is
> still recommended for using the `bitvec!` macro while I figure out how to
> properly use, but conceal, implementation details of the macro suite.

This gives you access to the `bitvec!` macro for building `BitVec` types
similarly to the `vec!` macro, and imports the following symbols:

- `BitSlice<E: Endian, T: Bits>` – the actual bit-slice reference type It is
  generic over a cursor type (`E`) and storage type (`T`). Note that `BitSlice`
  is unsized, and can never be held directly; it must always be behind a
  reference such as `&BitSlice` or `&mut BitSlice`.

  Furthermore, it is *impossible* to put `BitSlice` into any kind of intelligent
  pointer such as a `Box` or `Rc`! Any work that involves managing the memory
  behind a bitwise type *must* go through `BitVec` instead. This may change in
  the future as I learn how to better manage this library, but for now this
  limitation stands.

- `BitVec<E: Endian, T: Bits>` – the actual bit-vector structure type. It is
  generic over a cursor type (`E`) and storage type (`T`).

- `Endian` – an open trait that defines an ordering schema for `BitVec` to use.
  Little and big endian orderings are provided by default. If you wish to
  implement other ordering types, the `Endian` trait requires one function:

  - `fn curr<T: Bits>(index: u8) -> u8` takes a semantic index and computes a
    bit offset into the primitive `T` for it.

  and provides default implementations for two functions that you may need to
  override:

  - `fn next<T: Bits>(index: u8) -> (u8, bool)`
  - `fn prev<T: Bits>(index: u8) -> (u8, bool)`

  These two functions compute the next and previous, respectively, semantic
  indices and overflow markers from a given semantic index. The boolean flag
  indicates that moving the index forward or backward would cross into a new
  primitive storage element.

- `BigEndian` – a zero-sized struct that implements `Endian` by defining the
  forward direction as towards LSb and the backward direction as towards MSb.

- `LittleEndian` – a zero-sized struct that implements `Endian` by defining the
  forward direction as towards MSb and the backward direction as towards LSb.

- `Bits` – a sealed trait that provides generic access to the four Rust
  primitives usable as storage types: `u8`, `u16`, `u32`, and `u64`. `usize`
  and the signed integers do *not* implement `Bits` and cannot be used as the
  storage type. `u128` also does not implement `Bits`, as I am not confident in
  its memory representation, and dropping support for it allowed me to support
  older compilers.

`BitVec` has largely the same API as `Vec`, and should be easy to use.

The `bitvec!` macro requires type information as its first two arguments.
Because macros do not have access to the type checker, this currently only
accepts the literal tokens `BigEndian` or `LittleEndian` as the first argument,
one of the four unsigned integer primitives as the second argument, and then as
many values as you wish to insert into the `BitVec`. It accepts any integer
value, and maps them to bits by comparing against 0. `0` becomes `0` and any
other integer, whether it is odd or not, becomes `1`. While the syntax is loose,
you should only use `0` and `1` to fill the macro, for readability and lack of
surprise.

## Example

```rust
#[macro_use]
extern crate bitvec;

use bitvec::*;

use std::iter::repeat;

fn main() {
    let mut bv = bitvec![BigEndian, u8; 0, 1, 0, 1];
    bv.reserve(8);
    for bit in repeat(false).take(4).chain(repeat(true).take(4)) {
        bv.push(bit);
    }

    //  Memory access
    assert_eq!(bv.as_ref(), &[0b0101_0000, 0b1111_0000]);
    //                 index 0 -^               ^- index 11
    assert_eq!(bv.len(), 12);
    assert!(bv.capacity() >= 16);

    //  Arithmetic operations
    bv &= repeat(true);
    bv = bv | repeat(false);
    bv ^= repeat(false);
    bv = !bv;

    //  Borrowing iteration
    let mut iter = bv.iter();
    //  index 0
    if let Some(false) = iter.next() {} else { panic!() };
    //  index 11
    if let Some(true) = iter.next_back() {} else { panic!() };
    assert_eq!(iter.len(), 10);
}
```

Immutable and mutable access to the underlying memory is provided by the `AsRef`
and `AsMut` implementations, so the `BitVec` can be readily passed to transport
functions.

`BitVec` implements `Borrow` down to `BitSlice`, and `BitSlice` implements
`ToOwned` up to `BitVec`, so they can be used in a `Cow` or wherever this API
is desired. Any case where a `Vec`/`[T]` pair cannot be replaced with a
`BitVec`/`BitSlice` pair is a bug in this library, and a bug report is
appropriate.

`BitVec` can relinquish its owned memory as a `Box<[T]>` via the
`.into_boxed_slice()` method, and `BitSlice` can relinquish access to its memory
simply by going out of scope.

## Planned Features

- `#![no_std]` support that uses core libraries for allocation, and
  `#![no_core]` support that strips the vector type entirely and only provides
  the slice type.

- A `Box<BitSlice>` type that corresponds to `Box<[T]>` between `&[T]` and
  `Vec<T>`.
