# Publishing generic callable factory results

A factory returning a generic callable publishes the callable's participating result contract in the callable's declaration environment. A local generic constructor is available to that query by lexical declaration identity before the ordinary source walk reaches the class. Its runtime binding is still uninitialized until ordinary execution initializes it.

```js
function make() {
  class P<T: type> {}
  function f<U: type>(value: U) { return new P.<U>(); }
  return f;
}
const f = make();
const first = f.<uint8>(1);
const second = f.<string>("x");
```

The result of `make` includes the nested callable result `P<U>`. Completing that publication does not call `f` or construct `P`. Ordinary execution supplies this activation's class identity. Another evaluation of `make` introduces another class identity even at the same type argument. An arbitrary returned value cannot redefine the contract checking it.

A bare callable group with an owner publishes its owner contract. Attached cases remain available for group selection; they are not additional promises of the unapplied generic value. Standalone-only groups and ordinary overload sets retain their existing rules.

```js
function make() {
  class P<N: uint32> {}
  function f<T: type>(value: T): any { return value; }
  function f<uint.<const N>>(value: uint.<N>) { return new P.<N>(); }
  return f;
}
const f = make();                  // owner's generic contract
const selected = f.<uint.<12>>;    // selected case, N = 12
const value = selected(1);         // this activation's P.<12>
```

The selected value describes the selected declaration with its completed captures, including its inferred result. A case's own capture shadows a same-named captured outer parameter. An unrelated caller's frame cannot instantiate an unapplied function or change a stored selection. Reading or reflecting either value executes no ordinary body or parameter initializer. Written `any` on the owner remains erased; the selected case has its own result under the existing selection rules.

## Requirements and directions

Every valid direction must preserve lexical declaration and runtime activation identity, the owner's public contract, selected captures, nested binder scope, independent admission evidence, ordinary checking timing, participation, declared erasure, reference permissions and liveness. Unfinished dependencies must not silently become completed unknown results. No direction may execute an ordinary body or default to discover a contract.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Keep a source-order name table and accept the initially missing nested result | Moving a class or extracting a factory changes inferred members and can reject valid code; autocomplete loses the result | Rust lexical items and generic scope are declaration based, not a last-visited name | Cheap initial lookup; stale publications require later repair anyway | Violates publication dependencies and lexical identity; runtime initialization is a separate issue | Rejected |
| Require explicit results or erase all returned generic callable results | Users repeat inferable contracts or lose useful result members; wrong guesses fail later | Rust normally requires item result annotations, but this proposal already commits to participating inference | Less inference work, at the cost of supported semantics | Withdraws established inference/admission guarantees; no justification from reference liveness | Rejected |
| Publish owner and all attached cases as an intersection | A bare group unexpectedly promises signatures with unbound captures and additive call shapes | Rust has no equivalent runtime overload group; its function-item identity does not justify this intersection | Extra signatures and compatibility work, potentially unbound queries | Contradicts the proposal's owner-value rule; exposes captures before selection | Rejected |
| Derive the receiving contract from the arbitrary returned callable or a same-spelled caller binding | Replacement values or unrelated calls change what a destination accepts | Rust item identity and lexical generic scope do not let a candidate redefine its destination | Adds reflection or unstable name lookups; weakens reusable guards | Circular admission or wrong binding authority; cannot establish liveness | Rejected |
| Complete declaration-scoped producer dependencies, publish the owner's value contract, and instantiate a selected case only from its retained application | Factory extraction preserves result members; selection/reflection agree; wrong values still fail admission | Rust distinguishes item identity and type arguments, and separates inner binders from outer names; limits below | Resolve the constructor by existing lexical binding identity, reuse declaration/type metadata, and bind the selected signature once per specialization. No public fields, user-object shape changes or new inline-cache invalidation rule | Preserves independent contracts, activation identity, captures, erasure and existing liveness; ordinary source execution order is unchanged | Recommended independently of implementation effort |

Choose the last direction. The public owner projection applies to value contributions, not to the declaration group's candidate table used for direct calls and explicit group selection. The selected application, rather than an ambient frame, supplies the bindings used to describe a selected signature. Restore temporary frames on every exit.

Rust's [function item types](https://doc.rust-lang.org/reference/types/function-item.html) include defining item and type arguments. Its [generic parameter scopes](https://doc.rust-lang.org/reference/names/scopes.html#generic-parameter-scopes) bind parameters to their declaring item and allow inner items to reuse outer generic names. This supports the identity/scoping comparison, not the whole design: Rust inner items cannot capture outer generic parameters, and Rust does not have this proposal's runtime class activations or attached callable cases. These differences make the existing proposal rules authoritative at publication and runtime selection.

## Implementation and bounded audit

The engine resolves an explicitly applied local class constructor through the contributing expression's lexical binding, as its ordinary construction path already does. A bare group contribution retains the owner's public signatures; direct calls and applications retain selection candidates. Published inferred results treat captures as the selected declaration's own binders. Selected signature derivation supplies the registered application frame after the captured outer frame, for both parameter and result descriptions. Invocation similarly leaves the selected case's own capture names out of the captured outer frame; otherwise correct reflection alone would still allow its body to use the outer argument.

The focused cases cover the two previously failing factories, separate activations and reused selections, own/captured arguments, immutable aliases and conditional returns, same-spelled classes and generic parameters, owner fallback, direct additive and standalone selections, ordinary overloads, intrinsic generic construction, reflection without body/default execution, failure cleanup, independent destination rejection, and Script/Module declaration failure timing. This is a bounded repair, not closure of the six broader audit gates. The subsequent [constructor-origin audit](generic-constructor-origins.md) covers unapplied immutable generic constructor aliases and class-expression bindings; stored constructor applications, extracted methods, remaining generic contexts and composite paths still require their own evidence.

See [stored specialization bindings](stored-specialization-bindings.md) for operation-owned completion evidence and [callable contract identity](callable-contract-identity.md) for declaration comparison.
