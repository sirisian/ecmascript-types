# Stored specialization binding dependencies

A completed specialization retains its callable contract when stored in an inferred immutable binding. A captured local class does not become a caller-side name lookup merely because a direct call is split into selection and invocation.

```js
function make() {
  class C {}
  function f<T: type>(value: T) { return new C(); }
  function f<uint8>(value: uint8) { return new C(); }
  return f;
}
const f = make();
const selected = f.<uint8>;
const result = selected(1); // this factory activation's C
```

The application owns the selected declaration, completed arguments or case captures, and lexical dependencies. Its resulting completion retains that contract so the initializer can complete corresponding unresolved dependencies before ordinary admission. Each evaluation supplies its own completion, including a specialization-cache hit. The shared checking result and existing destination contracts are unchanged.

## Requirements and alternatives

A solution must keep immutable initialization admission, declaration and activation identities, written erasure, modes and defaults, receiver independence, specialization interning, existing checking timing and reference-liveness rules. It must neither run a function body to learn its signature nor select arguments again when a stored value is called. A wrong replacement must not redefine the target that checks it.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Erase the inferred callable type or skip immutable initialization admission | Storage loses autocomplete precision or accepts a wrong replacement | Rust function items retain function identity and type arguments when used as values | Cheap initially; loses useful type and call guards | Discards the destination obligation and evidence of distinct activations | Rejected |
| Reject captured result types in stored specializations | A valid direct call cannot be factored into selection and invocation | Rust supports storing a function item with specified arguments; it does not model this proposal's runtime class activations | Simple refusal excludes useful programs | Avoids conflation through an unnecessary restriction | Rejected |
| Derive every initializer target from the tested callable's current reflected type | A replacement can change what the destination accepts | Rust's identity-bearing function item does not authorize arbitrary values to redefine another destination | Repeated derivation and unstable guard assumptions | Circular admission; no reference permission follows | Rejected |
| Resolve a same-spelled caller binding or use the latest class for the source syntax | Unrelated calls alter which result type is admitted | Contradicts lexical generic scope in Rust and JavaScript | Small lookup with unsound identity/cache keys | Conflates declarations or activations | Rejected |
| Retain the successful selection operation's independently established instantiated contract and complete only corresponding unresolved destination dependencies | Storing, aliasing and invoking preserve usable members; incompatible replacements still fail at the boundary | Rust function items carry the selected function and type arguments; this proposal defines capturing classes and runtime overload selection | Reuse specialization and signature metadata; transfer one internal contract reference per completion. No new public property or user-object shape; existing cache identity remains authoritative | Preserves destination authority, erasure, declaration identity, checks and reference liveness | Recommended independently of implementation effort |

Choose the final direction. Runtime selection establishes a new operation-owned contract; an ordinary read of an arbitrary callable does not. For a generic specialization, its signature includes the declared return, or otherwise the published inferred return closed over its declaration environment and completed application. Written `any` remains `any`. A selected case uses that case's declaration and captures. The normal signature correspondence rules determine which unresolved dependencies can be completed; an already completed target cannot be retargeted.

Rust's [function item types](https://doc.rust-lang.org/reference/types/function-item.html) include the item identity and type arguments, which supports the expectation that storing an explicitly applied function preserves its identity and contract. Its [generic scope](https://doc.rust-lang.org/reference/items/generics.html) is a limited analogy: nested Rust items do not capture enclosing generic parameters, and Rust does not supply this proposal's per-evaluation class identity or reified overload-set semantics. The proposal's existing capturing and gradual-admission rules remain authoritative.

## Calling and observing the stored value

A transparent specialization implementation wrapper preserves the original invocation's checked result completion, including abrupt outcomes and frame cleanup. It is not another declared result boundary. This is the same [overload result transport rule](overloaded-result-dependencies.md) applied to the ordinary generic-specialization wrapper. ECMAScript's [completion propagation](https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html#sec-completion-record-specification-type) supplies the control-flow model; the additional dependency is proposal-specific internal metadata.

Creation and reflection do not execute the selected body or ordinary parameter initializers. Reflection describes the completed non-generic callable and its captured inferred result. Cache hits preserve specialization identity and still retain the contract on the current expression result. Calls retain ordinary argument admission and reference permissions. None of this makes an arbitrary reflective call, optional call or user-written wrapper transparent.

## Scope

This increment addresses stored case and owner selections and ordinary generic specializations, with written or inferred captured local-class results. Broader captured constraints/default computations, composite transport, resumable paths, convergence, failure precedence, budget/cache accounting and final full verification remain separate gates.
