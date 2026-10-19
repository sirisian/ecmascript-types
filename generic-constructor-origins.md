# Unapplied generic constructor aliases

A stable immutable value alias of an unapplied generic constructor preserves the constructor's lexical class origin when computing a construction's contribution. This includes anonymous and named class expressions bound by ordinary immutable declarations, alias chains and parentheses. The construction supplies the specialization under the existing explicit, contextual, inferred and default argument rules.

```js
function make() {
  const P = class<T: type> {};
  const Alias = P;
  function f<U: type>(value: U) { return new Alias.<U>(); }
  return f;
}
const first = make();
const second = make();
const holder = { f: first.<uint8> };
function replace(value: any) { value.f = second.<uint8>; }
replace(holder);
const wrong = holder.f; // TypeError: the independently inferred result belongs to first's P
```

The alias creates no new class identity. Another execution of `make` does. Publishing `f` must retain the result contract even when the contribution query precedes the ordinary walk of the class expression or alias declaration. Computing that contract executes neither initializer nor constructor. Runtime initialization and temporal-dead-zone failures keep their ordinary order.

A known incompatible consumer, such as `const wrong: string = new Alias.<uint8>()`, is rejected at the source checking boundary. Erased or unstable constructor origins keep their existing rules. A local alias named `Proxy` or `Map` denotes its lexical class, not a built-in selected by spelling. Autocomplete can therefore retain the constructed class's members through a stable alias; an incorrect assumption about a different factory activation still fails at admission.

## Requirements and directions

Every valid direction must retain lexical declaration identity, runtime activation identity, the source scope of written arguments, the construction argument ladder, independent destination evidence, participation and declared erasure. It must preserve initialization timing, receiver requirements, reference permissions and liveness, and execute no user code to recover a contract. Missing an established origin during an early query cannot silently complete that query with an unknown result.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Keep only direct constructor names and erase results reached through aliases | A small refactoring removes result members and permits wrong replacements; autocomplete becomes weaker without an explicit erasure | Rust's item identity survives value references; its type-alias syntax is a different facility | Cheap lookup, but incomplete publication dependencies still need repair | Loses established contracts and independent admission evidence | Rejected |
| Require explicit annotations or disallow generic constructor value aliases | Users repeat inferable contracts or lose an existing expression form | Rust item results are normally annotated, but that does not justify removing this proposal's participating inference or runtime constructor values | Less inference work by withdrawing supported behavior | Conflicts with existing alias and contribution rules; no liveness benefit | Rejected |
| Resolve a constructor by spelling in a collected name table, or use the most recently evaluated constructor | Same-spelled local classes and built-in shadows change behavior unexpectedly | Rust's lexical scopes and item identity do not select an unrelated same-named item | A name map is cheap; mutable last-value associations defeat stable guards and activation isolation | Wrong declaration or activation; cannot establish reference validity | Rejected |
| Recover the receiving contract from an arbitrary returned callable or constructed value | An incompatible replacement can redefine what its destination accepts | Rust's destination type is not inferred anew from a replacement value | Adds runtime inspection and unstable dependency recovery | Circular admission; cannot establish independent identity or liveness | Rejected |
| Follow an established stable lexical origin and complete the construction's producer dependencies | Aliases preserve result members and failure timing; distinct activations remain distinct | Rust separates item identity, generic arguments and lexical scope; important differences below | Reuse binding/declaration metadata and memoized origin dependencies. No public fields, user-object shape changes or new inline-cache invalidation rule are required | Preserves independent targets, argument scope, erasure, initialization and existing reference rules | Recommended independently of implementation effort |

Choose the last direction. Origin recovery is a checking dependency, not permission to execute an initializer or to reuse one activation's evaluated class in another. Built-in contracts require an established built-in origin. Parentheses do not change an origin; a same-spelled binding does.

Rust's [function item types](https://doc.rust-lang.org/reference/types/function-item.html) distinguish defining items and their generic arguments. Its [generic parameter scopes](https://doc.rust-lang.org/reference/names/scopes.html#generic-parameter-scopes) tie parameters to declaring items. These support the identity and scope comparison. Rust inner items cannot capture outer generic parameters, and Rust has no directly corresponding runtime class-expression activation. Also, Rust [type aliases](https://doc.rust-lang.org/reference/items/type-aliases.html) do not alias tuple-struct constructors. The aliases here are values, not a claim that Rust accepts the same syntax or constructor rules.

## Stored applications are a separate dependency

An alias that already stores a specialization needs more than the class's source declaration. For example, in `function make<T: type>() { class P<V: type> {} const Alias = P.<T>; function f<T: type>(value: T) { return new Alias(); } return f; }`, the argument of `Alias` belongs to `make`, although the later function also declares `T`. Re-reading that argument under the later function's frame is incorrect. A prototype doing so rejected a valid `make.<uint8>().<string>("x")` call by describing its result as `P<string>`.

| Direction for stored applications | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Replay initializer argument syntax in the later query's current name environment | Renaming an inner binder changes an already selected constructor | Rust scopes distinguish declaring items; the runtime capture analogy is limited | Little retained metadata, but repeated resolution and unstable dependencies | Rebinds captured arguments to the wrong declaration; cannot establish liveness | Rejected |
| Erase every stored specialization or require manual result annotations | Loses result members or imposes annotations on an established typed value | Rust's annotation rules do not imply erasing a known selected item | Lower implementation cost at the expense of promised checks | Fails independent admission and publication requirements | Rejected as a design; the remaining implementation gap is not approval of erasure |
| Retain the application's declaration-aware binding dependencies and complete them in its owning context | Stored selections remain stable under extraction and shadowing | Rust's item/argument distinction supports the identity aspect; this proposal defines runtime capture | Intern immutable dependency records and reuse completed selections; no user-object shape requirement | Preserves source binding authority and existing reference rules without candidate-derived evidence | Recommended; requires separate implementation and boundary coverage |

The same independent producer-contract requirement applies to extracted generic method values, while their receiver obligations must remain intact. Static-method and object-method probes currently lose a result identity at publication and accept a wrong-activation replacement. Treating extraction as automatic erasure, deriving the target from the replacement, or borrowing a caller's bindings remains rejected by the first matrix. Complete the method's own publication dependencies instead.

## Bounded implementation and remaining evidence

The engine uses its existing stable class-origin analysis in construction contribution queries. Unapplied immutable aliases supply the declaration template, while arguments written at the construction remain in that use's generic context. Bare construction uses the existing inference/default ladder. Explicit construction normalizes parentheses and respects lexical shadowing before considering built-in contracts. The ordinary constructor-argument check also requires an intrinsic origin before imposing `Proxy`'s built-in parameters.

Focused Script and Module cases cover separate activations, compatible replacement, explicit and inferred/defaulted construction, alias chains, class-expression names, type and value arguments, outer and own parameters, intrinsic shadows, reflection without execution, erased/mutable-origin controls and checking versus temporal-dead-zone timing. The carried probes for stored constructor applications and extracted static/object methods remain failing and are recorded with paired baseline/current observations. A freshly evaluated inline generic class expression also needs its own publication/context analysis; it is not made precise by borrowing a different evaluation's class. This increment does not claim closure of the broader identity, convergence, failure-state, budget, specification or regression gates.

See [generic callable publication](generic-callable-publication.md), [stored specialization bindings](stored-specialization-bindings.md) and [callable contract identity](callable-contract-identity.md).
