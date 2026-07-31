# `generic_per_mono`

By default a generic imported function has its type parameters **erased**: every
`T` is passed across the ABI as a `JsValue`, and the single JS binding that is
generated works for all instantiations. That is described in
[Working with wasm-bindgen Generics](../../working-with-generics.md), and it is
why `T` normally has to be a JS type (`JsGeneric`) rather than a Rust one.

`generic_per_mono` opts a single import out of erasure. Instead of one erased
binding, `wasm-bindgen` generates **one binding per monomorphisation**, each with
its own descriptor, so arguments and return values are marshalled at their
concrete types:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console, generic_per_mono)]
    fn log<T>(value: T);
}

log(42u32);        // crosses as a number
log("hello");      // crosses as a string
log(true);         // crosses as a boolean
```

Each of those three calls gets its own JS shim. Because nothing is boxed into a
`JsValue`, `T` can be an ordinary Rust type — `u32`, `f64`, `bool`, `String` —
which the erasure path does not allow.

## When to use it

Reach for `generic_per_mono` when you want one Rust signature to serve several
*Rust* types and you care about how they marshal. Reach for the default erasure
path when you are modelling JS generics (`Array<T>`, `Promise<T>`) and want a
single binding for all of them.

The trade-off is code size: one JS shim and one descriptor per instantiation. A
generic import instantiated at a dozen types produces a dozen shims, so prefer
erasure when the concrete marshalling does not matter.

## Trait bounds

Bounds you declare are part of the import's contract. They are carried through to
the generated wrapper, so callers must satisfy them, and they also reach the
generated shim — which means a shim signature may project an associated type off
a bounded parameter. Inline bounds, `where` predicates, and higher-ranked
predicates all work:

```rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(generic_per_mono)]
    fn sum_items<T>(items: T) -> f64
    where
        T: IntoIterator<Item = u32>;

    #[wasm_bindgen(generic_per_mono)]
    fn sum_by_ref<T>(items: &T) -> f64
    where
        for<'a> &'a T: IntoIterator<Item = &'a u32>;
}
```

## Other attributes

`generic_per_mono` composes with the usual import attributes — `method`,
`static_method_of`, `constructor`, `getter`, `setter`, `structural`,
`js_namespace`, `js_name`, `catch`, `variadic`, and `slice_to_array` — and the
resulting JS binding is shaped exactly as it would be for the equivalent
non-generic import.

`async` is supported, and returns a future in the usual way:

```rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(generic_per_mono)]
    async fn round_trip<T>(value: T) -> T;
}
```

## Unsupported shapes

These are rejected at compile time with a diagnostic pointing at the offending
declaration. Each generally keeps working on the type-erasure path, so the fix is
usually to drop `generic_per_mono`:

* **Lifetime or const generic parameters**, and generic parameters on the
  imported *type* (class-level generics).
* **`&mut T`** where `T` is a type parameter, and a reference to a type parameter
  **nested inside another type** (e.g. `Option<&T>`). A bare `&T` *is* supported.
* **Returning a reference.**
* **A bare type parameter as the `variadic` argument**, since it may
  monomorphise to a scalar, which is not spreadable.
* **A type parameter in the error position of a `catch` import**
  (`Result<T, E>` with generic `E`): only the `Ok` type is monomorphised, and the
  error type is always `JsValue`.
* **`slice_to_array` on a slice whose element type mentions a type parameter**
  (`&[T]`, `&[Vec<T>]`, `Option<&[T]>`). `VectorRefIntoWasmAbi` is implemented
  per concrete ABI shape, so no bound the caller can write makes an arbitrary `T`
  satisfy it; the element type must be concrete. See
  [`slice_to_array`](./slice_to_array.md).
* **`reexport`**, which has no well-defined target when one binding is
  manufactured per monomorphisation.

## Note on `&T` arguments

A bare `&T` argument is supported, and requires the referent to satisfy the bound
`wasm-bindgen` needs to marshal it — either a scalar (via `ScalarIntoWasmAbi`) or
a JS handle type. Passing `&SomeStruct` for a plain Rust struct is rejected,
since there is no ABI representation for it; take it by value, or pass a JS type.
