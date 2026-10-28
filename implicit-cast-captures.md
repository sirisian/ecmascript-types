# Capture bindings in implicit-cast bodies

A selected implicit cast must bind its header captures before its body runs. A capture referenced in the destination type reads the requested destination; a header capture absent from that destination keeps its receiver role. These are the projection roles already specified in [Primitive Operators](primitivemetadata.md#primitive-operators). The missing runtime binding made an admitted `operator float32.<X>()` throw `ReferenceError` as soon as its body read `X`.

```js
type Dim = { m: int32 };
meta Dim {
  default = { m: 0 };
  subtype(a, b) { return a.m === b.m; }
}
primitive float32<const X: Dim> {
  operator float32.<X>() {
    // X is the requested destination's completed Dim portion.
    return Number(this) + Number(X.m);
  }
}
const source: float32 = 3;
const a: float32.<{ m: 1 }> = source; // value 4
const b: float32.<{ m: 2 }> = source; // value 5
```

This does not make a cast change the base-admission rules. A statically incompatible component type or rational width remains an error at the receiving boundary. The body still computes a raw result, which must satisfy the actual target base; the receiving boundary's metadata validation follows.

## Decision and alternatives

Required properties: the binding follows its declaration and source/destination role; metadata is a completed portion of its owning meta type and reads with the declared field domains; components use the actual selected argument; callers cannot supply missing names; closures retain the correct invocation; failure restores the previous context; required effects, base admission, and reference permissions and liveness are preserved.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance, shapes and inline caches | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Bind every capture from the source or its default | The destination written beside `operator` disagrees with the value used to construct it; a units conversion can silently use the wrong coefficient. | Rust `From<T> for U` distinguishes the input from the result; C# conversion declarations also distinguish both types. Neither implies that destination inputs come from the source. | Cheap projection, but the wrong argument invalidates specialization assumptions. | Violates the existing destination-capture rule. | Reject. |
| Bind every header capture from the destination | Repairs the common spelling but changes a capture that occurs only in the receiver header. Completion would conceal this dependence on unrelated destination fields. | Rust and C# retain the input's identity separately from the result type. Their conversions do not justify erasing a receiver role here. | Equally cheap to cache, so cost provides no reason to merge the roles. | Violates receiver projection for captures absent from the destination. | Reject. |
| Search active generic frames by name | Identical cast declarations behave differently depending on their caller; an unrelated same-named parameter appears usable. | Rust generic parameters are declaration-scoped; ordinary ECMAScript lexical scope is not dynamic caller scope. | Requires caller-sensitive guards and unstable specialization keys. | Loses declaration and invocation identity; an unavailable argument can acquire an unrelated value. | Reject. |
| Re-evaluate the destination or its domain at each body entry | A type read can call a producer again, observe a replacement binding, or change which failure wins. | Rust and C# conversion signatures identify their types independently of a caller's spelling; neither supplies an analogy for replaying JavaScript effects. | Repeated resolution and calls are unnecessary when the boundary already has the determining descriptors. | Violates retained declaration context and effect order. | Reject as a way to recover already established arguments. |
| Add ordinary arguments or public properties to carry captures | Requires a new calling convention in source or exposes accidental mutable storage; autocomplete suggests operations the capture does not permit. | Rust and C# have explicit source-value parameters, but that does not require exposing this proposal's generic binding machinery as mutable objects. | Public properties change shapes and affect inline caches; promoted storage adds lifetime obligations. | Introduces writes and possible reference authority absent from application. | Reject. |
| Retain a declaration-owned binding recipe and instantiate its source/destination arguments for the selected call | The destination and header retain their stated meanings; errors occur at the relevant boundary; a saved closure keeps the argument it saw. | Rust's distinct input/result types and declaration-owned generics are limited analogies. ECMAScript supplies closure lifetime and lexical resolution. | Private descriptor slots and argument projections; compiled code can specialize them without new public properties or shape transitions. No producer replay is required. | Preserves scope, views, effects, base checks, and independent reference permissions and liveness. | **Choose independently of implementation effort.** |

Rust's `From<T>` consumes a source value and returns `Self`; its infallibility expectation differs from these checked JavaScript boundaries. C# allows conversion declarations on a source or target type, and its implicit conversions have their own restrictions. Neither language has this proposal's runtime metadata projection, so the choice of projection roles comes from this proposal, not from those analogies.

## Implementation and verification

Registration retains the actual capture declarations, the already resolved metadata domains, component positions, and whether a lexical reference in the destination selects that capture. This includes captures in a second header list. A same-spelled ordinary binding or a distinct nested binder is not evidence for the header capture.

For a selected call, metadata arguments use `MetadataPortion` and its completed defaults, normalized with the owning shape at the selected base. Their value views retain declared field types, including exact wide integers. Component arguments come from the selected subject's component slots. The frame binds declaration identities before the call and is removed on normal or abrupt completion. The function's retained parameter environment keeps escaped closures alive with their own arguments. Captures remain read-only and do not provide a borrowable location.

The portable `implicit-cast-captures.json` corpus covers metadata values and field domains, completed portions, unrelated metadata, changing arguments across calls, component types and widths, receiver-only captures, lexical shadows, direct and indirect eval, closures, imports and declaration helpers, assignment/argument/return/field/default boundaries, failed writes and borrows, and recovery after a throwing body. Width and component mismatches remain negative controls. The delivery's verification record gives exact test and baseline results.

A separate composition defect is preserved with a paired reproducer: when two meta types require separate identity casts, both the pulled baseline and this implementation invoke only the first body. The existing per-meta-type invocation requirement remains authoritative; this change does not treat that short circuit as correct. Repairing composition must retain the argument ownership, raw-result admission and effect rules above.

The broader implicit-cast applicability/composition audit and the imported-constructor `instanceof`/hook-effect audit remain separate work. Passing these cases does not certify unsupported cast patterns, vector cast dispatch, or all identity, resource and conformance gates.

Sources: [Rust From](https://doc.rust-lang.org/std/convert/trait.From.html), [Rust generic parameters](https://doc.rust-lang.org/reference/items/generics.html), and [C# conversion operators](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/user-defined-conversion-operators).
