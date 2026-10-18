# Escaped local classes in inferred results

A closure can retain a local class correctly while its caller still loses that class in an inferred result contract. In this example the receiving annotation and constructor run successfully. The old implementation then checked the return from `use` by looking up `P` in the caller's environment and threw ReferenceError.

```javascript
let log = "";
function make<T: type>() {
  class P {
    constructor(v: boolean) { log += "b"; }
    constructor(v: T) { log += "t"; }
  }
  return (v: any) => { const p: P = v; return p; };
}
const convert = make.<string>();
function use<T: type>() { return convert("yes"); }
use.<boolean>();
log; // "t"
```

The same failure occurs without generics. It is an inferred-contract dependency failure, not a lost value capture or a constructor-ranking failure.

## Required properties

A solution must preserve the declaration and the applicable evaluated class identity, the original generic captures, nominal membership, existing conversions and their failure precedence. A same-spelled class in the caller cannot supply an identity. Another activation of a factory cannot replace an already captured class. Return admission must remain a check rather than infer its own target from the value being admitted. Reflection must not call a producer to discover its result. Uninitialized bindings and reference liveness remain independent obligations.

The contract must survive forwarding without making programmers expose an implementation-local class name. Autocomplete should preserve the published members and signatures. It must not promise a shape whose nominal obligations are silently erased.

## Available directions

| Direction | Ergonomics | Prior art | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Erase an escaping class result to `any` or its shape | Loses members or nominal precision; a wrong guess fails later, or passes incorrectly | Structural typing is useful for explicit interfaces; Rust's opaque returns still retain a concrete hidden type | Cheap but invalidates useful type-based optimization | Changes publication and nominal membership; removes an established boundary | Rejected: escape alone does not justify erasure |
| Reject every escaping local class result | Requires exposing or annotating an otherwise valid implementation detail | Rust can expose an opaque return satisfying traits; rejecting all such results is stronger than that analogy | Simple implementation; no runtime dependency state | Avoids guessing but rejects existing closure and publication commitments | Rejected: a representational limitation should not become a language restriction |
| Look up the name in the current caller, or keep the latest constructor for a Parse Node | Ordinary refactoring and call order change behavior; autocomplete cannot explain which class wins | Neither lexical closures nor Rust captures use unrelated caller bindings | A small cache, but invalidation spans activations and realms; unsafe specialization | Confuses declarations or activations; failure can contaminate later checks | Rejected: names and source sites alone are insufficient |
| Take the constructor from the returned object | Appears convenient until a wrong return chooses its own contract | Runtime type inspection is not a declared return contract; Rust's hidden return type is fixed independently of each value | Adds object/prototype inspection and may introduce exotic effects | Makes the checked value establish the target; can admit the wrong activation | Rejected: circular admission evidence |
| Retain the nominal dependency and instantiate it in its declaring environment | Preserves members, signatures and ordinary forwarding; failures remain at the actual boundary | Closest to lexical capture and Rust's retained generic captures, with the differences below | Retain environment/dependency identities; compiled lexical slots and dependency guards can avoid repeated walks without changing user object shapes | Preserves nominal checks and distinct activations; does not initialize bindings or prove references live | Recommended, independently of implementation effort |

Rust's [closure types](https://doc.rust-lang.org/reference/types/closure.html) retain the places their bodies capture. [Return-position impl Trait](https://doc.rust-lang.org/reference/types/impl-trait.html) retains relevant generic parameters for a hidden concrete type. These support preserving dependencies instead of resolving a spelling in the caller. They are not exact models: this proposal infers results for ordinary named functions, has JavaScript class evaluation identities, and uses its existing reference-liveness rules rather than Rust's borrow checker. No new opaque-type syntax is proposed here.

## Bounded engine implementation

For an unresolved, non-generic source ClassDeclaration at the root of an inferred result, first consult the function value's captured declarative environment. Accept a constructor only if its associated class record names the required declaration. Otherwise a single direct returned call through a captured, immutable `const` binding may supply its published nominal result. Follow such dependencies iteratively, with a cycle guard, and require the same declaration at every step. A captured class identity already present in a contract is preserved.

This work reads only initialized ordinary value bindings. It does not invoke a function, getter, Proxy trap, default initializer or reference read; it does not follow an indirect Module binding. It refuses a body-local binding as evidence from the function's outer environment, even when an outer binding has the same spelling. The syntactic dependency cache is weak and contains no concrete activation result. No global latest-constructor table is introduced. The ordinary nominal admission and prototype-liveness checks still enforce the completed contract.

This is a bounded implementation of the recommended direction. Multiple return paths, body-local dependency bindings, imported direct dependencies, generic class applications and nominal occurrences nested inside composite publications still require the appropriate dependency representation. They are not proved absent or inapplicable when this implementation has no evidence. Do not broaden it by guessing a constructor, by consulting the returned value, or by weakening a failed check.

The two measured next witnesses are a forwarding `const` created inside the caller and two return sites that both forward the same captured `const`. Both should preserve the result but still fail in the current interpreter. The next increment should extend dependency instantiation while preserving branch and activation identity; the five-direction decision above remains applicable.
