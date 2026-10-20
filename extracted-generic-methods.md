# Publishing extracted generic methods

A participating method result remains part of the method's callable contract when the method is returned, stored, reflected or specialized. Open generic parameters are retained until the corresponding application supplies them. A factory's local class declaration is resolved in its lexical environment even when inference requests the method before the ordinary source walk visits that class.

```js
function make() {
  class P<T: type> {}
  class C {
    static create<U: type>(value: U) { return new P.<U>(); }
  }
  return C.create;
}
const first = make();
const second = make();
const holder = { create: first.<uint8> };
holder.create(1);
```

The two factory calls produce different `P` families. Replacing `holder.create` through an erased alias with `second.<uint8>` cannot change the established destination contract; its subsequent checked read rejects that replacement. Reusing the same activation and specialization succeeds. The same publication principle applies to object methods, instance methods, and the class identities nested in async or generator result protocols.

Extraction and specialization do not bind `this`. A class method keeps the receiver requirement already present in its static signature, including in its runtime signature. A free call remains an error under the existing receiver rules. Preserving a receiver requirement is separate from inferring a result and from establishing a particular runtime receiver.

## Requirements and directions

A valid solution preserves declaration and activation identity, generic binder scope, participation and explicit erasure, independent destination evidence, receiver requirements, invariant arguments, default information, checking phases, and reference permissions and liveness. Reading a contract must not execute an ordinary body, getter or parameter default. Runtime initialization and the temporal dead zone remain unchanged.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real engine performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Require explicit results, or erase every result containing an open parameter | Method extraction loses useful members or forces redundant annotations; wrong replacements fail later | Rust item returns are generally explicit, but this proposal already supports participating inference; that difference does not justify selective erasure | Reduces initial inference work but removes promised behavior | Breaks existing publication and identity guarantees; supplies no liveness benefit | Rejected |
| Resolve classes by the last visited name, or discover a destination contract from the replacement callable | Source order and unrelated scopes change autocomplete and acceptance; a wrong guess can silently change the destination | Rust function items retain their defining item and arguments; neither Rust nor ordinary ECMAScript lexical lookup uses a caller's same-spelled declaration | Name lookup is cheap, but stale contracts and value-dependent guards inhibit reuse | Wrong declaration authority or circular admission; candidate values cannot prove an independent destination | Rejected |
| Bind the original receiver automatically, or discard its requirement when extracting | Calling the extracted value becomes convenient but changes ordinary JavaScript behavior or delays a missing-receiver failure | Rust method receivers are explicit `self` parameters; JavaScript extraction does not implicitly bind a receiver | Binding needs extra callable objects and receiver retention; dropping requirements hides necessary guards | Changes the existing receiver and lifetime rules and can retain an object unexpectedly | Rejected |
| Complete the declaration-scoped result and preserve its generic and receiver metadata through storage and specialization | Result members survive extraction; same-activation reuse works and wrong activations fail at the established boundary | Rust function-item types distinguish item and arguments; associated functions and methods distinguish receiver requirements. These are limited analogies, described below | Reuse declaration/type metadata and specialization caches. No public properties, user-object shape transitions or new inline-cache invalidation rule are required. An implementation can canonicalize concrete arguments once per cached contract | Preserves independent evidence, activation identity, nested binders, invariant comparison, receiver requirements and liveness; defaults remain unevaluated during inspection | Recommended independently of implementation effort |

Choose the last direction. A captured environment substitutes only free parameters of a stored generic callable. Its own binder with the same spelling remains its own binder. Both declared and inferred result positions participate in this substitution. Runtime source-class dependencies nested inside a protocol are closed from the callable's declaring environment, rather than from an arbitrary value supplied to a destination.

Concrete value arguments in nested callable contracts must use the same canonical domain representation as direct class results before invariant comparison. The engine reifies an unconverted Number literal from the same declaration position's already checked value argument only when the numeric value matches exactly. A type parameter does not authorize that value-domain conversion. Preserve the fact that a parameter has a declared default without evaluating that default. Neither requirement weakens argument invariance or changes overload identity and selection.

Re-running generic argument binding across every nested contract was also considered. It could make results consistent for users, but has no closer Rust analogy than preserving the completed application, performs redundant domain work in a real engine, and risks resolving a captured domain outside its declaring context or evaluating it again. It adds machinery without strengthening liveness or admission. Reject that direction in favor of exact checked-position transfer; this choice concerns authority and observable work, independently of implementation effort.

Rust's [function item types](https://doc.rust-lang.org/reference/types/function-item.html) identify the item and its generic arguments, while coercion to a function pointer loses item identity. Its [associated items](https://doc.rust-lang.org/reference/items/associated-items.html) distinguish associated functions from methods with an explicit receiver. Rust does not model this proposal's runtime class activations, generic callable objects, or ordinary JavaScript receiver binding; those parts follow this proposal and ECMAScript, not the analogy.

## Bounded implementation audit

The engine resolves direct class-object reads by lexical declaration, retains symbolic object-method publications, and preserves class-method receiver markers in runtime callable signatures and specialization metadata. Inferred binding admission substitutes free inputs, and protocol publications retain their nested lexical class dependencies. The focused cases cover independent and reused activations, type and value parameters, captured and shadowing binders, inherited and computed static access, reflection without body/default execution, receiver rejection, explicit erasure, temporal-dead-zone behavior, and Script/Module failure timing.

This is an extracted-method increment against the published repository bases. It is separate from the earlier stored-constructor-application patch, which was not present in those bases. Integration with that patch and the broader generic-context, convergence, checking-state, resource-accounting, specification-determinacy and final regression gates remain distinct work.

See [generic callable publication](generic-callable-publication.md) and [generic constructor origins](generic-constructor-origins.md).
