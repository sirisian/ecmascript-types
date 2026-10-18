# Checked class dependencies at generic calls

A checked generic call can infer a type whose class applications still name source declarations. Before the runtime binder compares that type with an evaluated constraint, instantiate its class dependencies in the lexical activation to which those declarations belong. Do this recursively through application arguments and containing type shapes. Keep an already captured family identity; an ambient evaluation of the same declaration cannot replace it. Keep the shared checking result symbolic.

```js
class Pair<A: type, B: type> {}
function f<T: type extends Pair.<any, string>>(x: T) { return T; }
f(new Pair.<uint8, string>()); // T remains Pair.<uint8, string>

class A<T: type> {}
class B<T: type> {}
function g<T: type extends A.<B.<_>>>(x: T) { return T; }
g(new A.<B.<uint8>>()); // both application heads retain their families
```

The first constraint is an ordinary concrete view. The `any` in `Pair.<any, _>` is instead a fixed family-pattern entry and does not match `uint8`. Completing a declaration dependency changes neither rule, the inferred arguments, metadata, nor a value's storage contracts.

## Requirements

1. The checked and runtime bindings identify the same evaluated families, including nested arguments and instantiated ancestry.
2. Repeated evaluation of one declaration does not share an activation's completed binding with another activation. Captured identities remain fixed.
3. The receiving constraint is not selected from the argument value. Ordinary admission still runs, including after gradual calls.
4. Completion performs no public property read, conversion, constructor execution or mutable source-name lookup. It does not alter argument effects, failures or reference liveness.
5. Static shape information survives; an interface view or optional member must not disappear because an argument's current object has less information.

## Directions considered

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Use the symbolic checking record unchanged | Autocomplete promises a call that can fail despite matching arguments | Rust resolves declaration-bound types before execution; it does not compare an unresolved declaration with a runtime instantiation | Cheap transport, but the result cannot justify specialization or an inline cache | Violates requirements 1 and 2 when a later fallback completes only the outer head | Rejected: the two reproduced failures |
| Ignore missing family identity in equality or matching | Makes simple calls pass but hides cross-activation mistakes until state is used | Rust's item identity and .NET's retained generic type information do not justify equating distinct declarations or applications | Cheap comparison with unsound cache equivalences and layout assumptions | Violates 1, 2 and 3; equality would depend on whether one side was incomplete | Rejected |
| Discard checked bindings and infer again from runtime values | Loses declared optional members and other information visible in autocomplete; inference becomes value-dependent | Rust infers from static types. Runtime reflection in .NET exposes an actual type but is not a substitute for the caller's declared contract | Can require object inspection and repeated inference, with observable property access | Violates 3 and 5; can also violate 4 | Rejected |
| Resolve class names again or memoize the latest constructor on source syntax | A shadow or reassignment silently changes a previously determined type; interleaved closures disagree | Rust resolves paths by scope, not the latest execution of an item. Its nested items do not capture outer generic parameters, so that part is not analogous | Name lookup can encounter object environments; a syntax-global cache has the wrong key | Violates 2 and 4; a name alone does not identify the originating declaration instance | Rejected |
| Reject every inferred type with an unresolved class dependency | Requires explicit arguments for ordinary nested generics even when the lexical dependency is available | Rust and .NET both support inferred uses without treating every generic application as unknown | Simple rejection, no runtime cost for rejected programs | Conservative but needlessly rejects requirement 1's valid programs | Rejected |
| Instantiate declaration dependencies for this use from retained lexical family identities | The inferred type matches the written explicit application and autocomplete; wrong family arguments still fail at admission | Follows Rust's declaration-bound substitution principle and .NET's retained constructed-type identity. Neither is a direct model of fresh JavaScript class families per lexical activation | A compiler can lower a dependency to an environment slot and cache on family plus arguments. No new instance fields, shape transitions or user-property inline-cache invalidations are required | Meets all five requirements when the dependency context is available; preserves completed identities and ordinary checks | Recommended independently of implementation effort |

Rust's generic scope rules: https://doc.rust-lang.org/reference/items/generics.html and https://doc.rust-lang.org/reference/names/scopes.html. .NET runtime generic information: https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/generics/generics-in-the-run-time. These comparisons motivate declaration-bound identity, not a claim that either language has this proposal's `any` views or anonymous family constraints.

## Implementation boundary

The proof of concept records evaluated class declarations by declaration and lexical environment, with weak environment keys. A checked call creates a private copy of its binding graph, retaining cycles, and resolves available class dependencies from that environment chain. Specializations do not overwrite a family's entry. Reassignment of the source class binding does not change the retained declaration association. Existing completed nominal records are kept intact. Runtime heritage is taken from the resolved family, rather than reconstructed in the caller's scope.

This repair covers lexical class declarations and their available application dependencies. It does not establish complete transport for escaped class expressions, imported aliases, arbitrary aggregate provenance or every generic-context path. Unavailable evidence remains unresolved; neither a same-spelled class nor the tested value is used to manufacture a match. Those paths remain in the declaration-identity audit. No reference is dereferenced, extended or granted new permissions by this operation.
