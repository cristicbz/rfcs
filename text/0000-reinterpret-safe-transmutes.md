- Feature Name: reinterpret_safe_transmutes
- Start Date: 2015-07-18
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Add a new marker trait, `std::marker::Reinterpret`, to the standard library
which encodes the fact that a type is truly POD: its values  can be serialized
and deserialized just by reinterpreting its memory as a byte string without any
memory unsafety. Such types can be safely transmuted between each other and a
(potentially mutable) reference to a `Reinterpret` type can be safely transmuted
to a (potentially mutable) reference to `[u8]`.

Since one can modify a value of a `Reinterpret` type arbitrarily by converting
it to `&mut [u8]`, the constraints on such a type are fairly strict: at the very
least any byte string of the same size must be a valid value.This is a stronger
bound than `Copy` since, among other things, it excludes references.

To support the `Reinterpret` trait, this proposal also introduces two more
marker traits which require compiler support:
 * `ReprPacked` -- Automatically implemented by any type defined with the
   `#[repr(packed)]` attribute.
 * `Public` -- Automatically defined by any type whose members are all defined
   as `pub`.

# Motivation

There many low level scenarios where treating a struct as a byte string, using
`sizeof` and casting it to and from `void*` is very common in C. These include:

  * Serializing and deserializing data [to and from devices][diwic-reddit-comment].
  * Memory mapping files which contain large, index-based data structures. This
    is a very common storage strategy for video games where it can drastically
    cut load times.
  * Vertex (and other kinds of) buffers that are shared between GPU and CPU in
    video games or other graphics and GPGPU applications.

Currently these scenarios require unsafe code and the blunt instrument that is
`std::mem::transmute`. Rust could (and should, in the author's opinion) open the
door to more low level systems programming within its safe subset, enabling more
people to write this sort of code. Especially since this can be done with very
little compiler support.

# Detailed design

This RFC proposes the addition of the following (safe) functions to the
`std::mem`:

```rust
  pub fn reinterpret<S, D>(source: S) -> D;
          where S: Reinterpret, D: Reinterpret;
  pub fn reinterpret_bytes<S>(source: &S) -> &[u8];
      where S: Reinterpret + ?Sized;
  pub fn reinterpret_bytes_mut<S>(source: &mut S) -> &mut [u8];
      where S: Reinterpret + ?Sized;
```

All these functions can be implemented with a simple
`unsafe { mem::transmute(source) }`. To ensure that they are actually memory
safe, however, we need the following need to be true of a `Reinterpret` type:

  1. It needs to be `Copy`. If, for whatever reason, it is not legal to copy a
     value by `memcpy` then it's trivially invalid to do anything we'd want to
     do with a `Reinterpret` type.
  2. All its members must be public. Otherwise `reinterpret_bytes_mut` would
     allow modifying private fields, which is considered [unsafe][ref-unsafety].
  3. The type must be defined with the `#[repr(packed)]` attribute. This is for
     two reasons:
       a. First, Rust's default struct layout is undefined, so a type needs to
          be at least `#[repr(C)]` to not invoke undefined behaviour when
          reinterpreting it as an array of bytes.
       b. Second, reading out of and writing to the padding fields of a struct
          is undefined behaviour. LLVM is allowed to put its register spill into
          the padding of a struct stored on stack.
  4. All its members must also be `Reinterpret`. Otherwise we could clobber a
     member's type private members, or otherwise mess up its invariants.

To support constraints 2 and 3, this RFC proposes the introduction of two
additional built-in `std::marker` traits:

```rust
  /// The `Public` trait is automatically implemented for any types whose
  /// members are all public (defined as `pub`).
  ///
  ///
  #[lang_item = "public"]
  pub unsafe trait Public {}

  /// The `ReprPacked` trait is automatically implemented for any type which
  /// is defined with the `#[repr(packed)]` attribute, or:
  ///  * It is a primitive type (e.g. `u8`, `i32`, `f64`, `bool` etc.)
  ///  * It is an array of a `ReprPacked` type (`[T]: ReprPacked` iff
  ///    `T: ReprPacked`.
  ///  * It is a C-like enum (e.g. defined with `#[repr(u32)]`).
  ///
  /// Notes:
  ///  * Tuples are not `ReprPacked` since they may introduce padding.
  ///  * Non-C-like enum-s are not `ReprPacked` since their layout is
  ///    unspecified.
  #[lang_item = "repr_packed"]
  pub unsafe trait ReprPacked {}
```

Using these two traits we can now define `Reinterpret` as an opt-in built-in
trait (OIBIT):

```rust
  pub unsafe trait Reinterpret: Public + ReprPacked + Copy {}
  unsafe impl Reinterpret for .. {}

  /// References of any kind are non-`Reinterpret`. Otherwise, by mapping to
  /// `&mut [u8]` they could be made to an arbitrary location.
  impl<'a, T> !Reinterpret for &'a T {}
  impl<'a, T> !Reinterpret for &'a mut T {}

  /// `bool`-s are not `Reinterpret` since the only valid values are 0 and 1.
  impl !Reinterpret for bool {}

  /// `char`-s are not `Reinterpret` since they need to enfore UTF-8 invariants
  /// (0xff, for instance, is not a valid char).
  impl !Reinterpret for char {}
```

Note that this definition inherits many of its constraints from ReprPacked:
arrays 'inherit' `Reinterpret`, tuples do not, enum-s are `Reinterpret` only if
they're C-like.

# Drawbacks

* More proliferation of unsafe OIBIT-s. These are viral and the fact that they
  are magically implemented is a source of surprise to newcomers.
  Marker traits (even `Copy`) are already a common source of confusion in the
  language.
* This design is not as rich as one might hope. In particular one cannot
  transmute between `Reinterpret` references.  This rules out interpreting
  `&[SimdVec4]` as `&[f32]` for instance and similar patterns. See unresolved
  questions for why a reference transmuting function was not included. Perhaps a
  more complex design would allow for this.

# Alternatives

* Don't do this. Writing certain kinds of low level code remains limited to
  unsafe code and the use of `mem::transmute` which can cause a lot of
  collateral damage (binding to the wrong lifetime amongst other things).
* Provide only `ReprPacked` and `Public` and implement `Reinterpret` and
  associated functions in a separate crate; this would rely on OIBIT-s being
  stabilized. Furthermore, a trait like `Reinterpret` seems fundamental
  and would enable many abstractions to be built on top of it (e.g. `Read::read`
  could accept any `&mut [T]` with `T: Reinterpret`). An external crate might
  cause fragmentation.
* Instead of `ReprPacked`, support `Repr<Packed>`, `Repr<C>`, `Repr<Rust>`,
  `Repr<u32>` etc. More complexity, but perhaps clearer and more complete. We
  can punt on stabilizing the `ReprPacked` and `Public` traits, while still
  stabilizing `Reinterpret`.
* Instead of separately providing these three traits, we could simply have
  `Reinterpret` as `lang_item` trait.  `lang_item` trait, `Reinterpret` which
  encompasses all of them. Would require fewer stabilization decisions, but more
  magic in the compiler which seems undesirable.
* A previous design required an explicit opt-in even when all the constraints
  were satisfied. However, when all members of a type are `pub` anyway, the
  explicit opt-in doesn't prevent anything: one still cannot rely on any
  invariants being preserved since users of the type can clobber its fields to
  their heart's content.
* `Reinterpret` needs a good bikeshedding. `Pod` was avoided because it is
  overloaded in the `Copy: Pod: Clone` debate. Other names: `BitSafe`, `Bytes`,
  `MapBytes`, `Mappable`, `StdLayout`, `SafeTransmute`, `Transmute`,
  `ActuallyPod`, `ReallyReallyPod`.

# Unresolved questions

* Patrick Walton (pcwalton@) raised the point that transmuting to `f32` or `f64`
  is undefined behaviour. Is that true? I couldn't find any reference supporting
  that, but he generally knows what he's talking about. If it really is the case,
  we can, of course provide negative `impl-s` for floating point types, but that
  does reduce the usefulness of this design for GPU buffers by quite a bit.

* Should we disallow zero sized structs?

* We could relax the restriction on `reinterpret_bytes` to `ReprPacked` only.
  Technically, reading a type's private fields is not unsafe and there's no need
  for the type to be `Copy`.

* Is there a way to make transmuting between references to `Reinterpret` types
  safe?

```rust
  fn reinterpret_ref<S>(source: &S) -> &D
          where S: Reinterpret + ?Sized, D: Reinterpret + ?Sized {
      unsafe {
        assert_eq!((source as *const _) as usize % align_of::<D>(), 0);
        transmute(source)
      }
  }

  fn reinterpret_ref_mut<S>(source: &mut S) -> &mut D
          where S: Reinterpret + ?Sized, D: Reinterpret + ?Sized {
      unsafe {
        assert_eq!((source as *mut _) as usize % align_of::<D>(), 0);
        transmute(source)
      }
  }
```

In C and C++ these transformations would violate the [strict aliasing
rule][cpp-ref-reinterpret]:

> When a pointer or reference to object of type T1 is reinterpret_cast (or
> C-style cast) to a pointer or reference to object of a different type T2, the
> cast always succeeds, but the resulting pointer or reference may only be
> accessed if both T1 and T2 are standard-layout types and one of the following is
> true:
>
>  1. T2 is the (possibly cv-qualified) dynamic type of the object
>  2. T2 and T1 are both (possibly multi-level, possibly cv-qualified at each
>     level) pointers to  the same type T3 (since C++11)
>  3. T2 is the (possibly cv-qualified) signed or unsigned variant of the dynamic
>     type of the object
>  4. T2 is an aggregate type or a union type which holds one of the
>     aforementioned types as an element or non-static member (including,
>     recursively elements of subaggregates and non-static data members of the
>     contained unions): this makes it safe to cast from the first member of a
>     struct and from an element of a union to the struct/union that contains
>     it
>  5. T2 is a (possibly cv-qualified) base class of the dynamic type of the
>     object
>  6. T2 is char or unsigned char
>
> If T2 does not satisfy these requirements, accessing the object through the
> new pointer or reference invokes undefined behavior. This is known as the
> _strict aliasing rule_ and applies to both C++ and C programming languages.

So we can cast to `&[u8]` or `&[i8]` (6), but not to anything else without
violating strict aliasing. _Maybe_ thanks to Rust's ability to rule out
simultaneous mutability and aliasing, these rules could be relaxed (since they
generally concern optimisations which allow skipping loads after stores of
unrelated types). If not, maybe some mechanism could be used to signal to LLVM
when aliasing like this is possible.

To reiterate this would be hugely useful for e.g. `&[SimdVec4]` as `&[f64]` or
`&[u8]` to `&[u32]`.

# Acknowledgements

I first posted this as a [pre-RFC][internals-pre-rfc] on the internals forum
where I received hugely useful feedback from a number of people: cmr@,
steven099@, pcwalton@, tbu@, rkuppe@, frankmcsherry@, diwic@. As a result of
their feedback the design was drastically simplified and essentially rewritten
twice.

[diwic-reddit-comment]: https://www.reddit.com/r/rust/comments/3adl7k/viewing_integer_slice_as_u8
[cpp-ref-reinterpret]: http://en.cppreference.com/w/cpp/language/reinterpret_cast
[ref-unsafety]: http://doc.rust-lang.org/book/unsafe.html#what-does-%E2%80%98safe%E2%80%99-mean?
[internals-pre-rfc]: https://internals.rust-lang.org/t/pre-rfc-explicit-opt-in-oibit-for-truly-pod-data-and-safe-transmutes/2361
