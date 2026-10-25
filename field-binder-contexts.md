# Field construction and generic declaration context

A field initializer adopts the generic signature supplied by that field's annotation. Runtime type identity can ignore a renaming of bound parameters; construction must still use the declarations and captured arguments that supplied this source context.

```js
class C {
  f: <U: type>(value: U) => U = function(value: U): U { return value; };
  g: <V: type>(value: V) => V = function(value: V): V { return value; };
}
const c = new C();
const f: any = c.f, g: any = c.g;
f.<number>(7);
g.<string>("ok");
```

Both calls succeed. Interning `f`'s type first cannot make `V` disappear from `g`'s contextual signature. Reversing the fields, warming an equivalent Type Object, or constructing a second instance cannot change this outcome. The two generic function types retain their existing alpha-equivalent identity.

An alias must likewise retain the resolved source context associated with its actual initialized binding. For `type A = <U: type>(x: U) => U` and `type B = <V: type>(x: V) => V`, equal public Type Objects do not let A's declaration replace B's contextual binder. Alias chains and imports retain the source declaration context under their ordinary binding and export resolution. An arbitrary value binding containing an equal Type Object does not establish that provenance.

## Decision

A valid direction must preserve all of these: declaration-owned binders and captures; existing Type Object equivalence; ordinary annotation and initializer evaluation counts; TDZ and module failure order; independent activations and failed-construction recovery; field admission before publication; reference permissions and liveness.

| Direction | Ergonomics | Prior art and limits | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Retain resolved declaration context separately from public type identity | Autocomplete uses the written signature, including its own binders and defaults. A correct annotation does not fail because another type was formed first. | Rust scopes generic parameters to their declaring item and treats field initialization as a coercion site. It does not have this proposal's reified structural generic function Type Objects; nested Rust items also do not capture outer generic parameters. | A field descriptor or compiled initializer can retain an immutable context operand. Alias binding metadata can retain the already resolved record. No user-object properties, shapes, wrappers or additional polymorphic property reads are required. | Preserves alpha-equivalence, declaration identities, activation-specific captures, effects and existing liveness checks. | **Recommend**, independently of implementation effort. Retain only for the declaration/binding lifetime; no global source-name cache. |
| Split Type Objects by binder spelling or source declaration | Two otherwise equal signatures acquire surprising identities; tools cannot promise spelling-only refactors preserve equality. | Rust item identity does not justify making every structural function type nominal here. | Enlarges interning/cache populations; downstream guards and ICs see unnecessary distinctions. | Breaks existing alpha-equivalent type identity. | Reject. Construction context is not a new type identity. |
| Re-resolve the annotation for each initializer | Simple examples look correct, but getters, computed annotations, defaults or failures repeat unexpectedly. | Static Rust coercion does not justify repeated dynamic type evaluation. | Adds work and possible user calls per instance; weakens specialization and IC assumptions. | Can use the wrong environment, demand defaults early and change effects/failure order. | Reject. Reuse the established resolved context. |
| Rewrite body annotations to the interned representative's names | Autocomplete and written names cease to agree; collisions can silently select an outer binding. | Alpha-renaming in a compiler must preserve binding identity; arbitrary replacement by spelling is not that operation. | Requires mapping/rewriting and capture bookkeeping; no shape benefit over a retained operand. | Can confuse independent declarations, defaults and captured arguments. | Reject. Interning is not authority to rename this source's bindings. |
| Attach context to the public Type Object, constructed value, shared syntax or ambient name map | Outcomes depend on first or last use, reentry and another activation; a wrong guess fails far from the annotation. | Neither Rust lexical scopes nor explicit field coercion sites supply ambient invocation authority. | Value properties can perturb shapes/ICs; side tables still need lifetime/reentry isolation. | A shared representative cannot own several distinct source contexts. Mutable ambient state leaks across calls and failures. | Reject. Keep context with the actual declaration/binding and operation. |
| Keep the canonical signature or require explicit type parameters only in fields | A generic literal accepted at a typed binding fails as a field; users must repeat declarations despite contextual autocomplete. | Rust's field coercion sites support destination context, within its static model. | Saves a context operand but may cause wrong specialization or later failed calls. | Contradicts adopted generic signatures and makes cache history observable. | Reject. |

## Required sequencing

1. Resolve the field annotation in its existing class, private-name, receiver and generic environment. Retain the resulting construction context before public Type Object interning can select another alpha-equivalent representative.
2. Evaluate the initializer once using that context, including nested object/tuple contributions and ordinary anonymous-function naming. The function's explicitly written type-parameter list continues to take precedence over adoption.
3. Apply the existing field admission using the declared Type Object and the initializer's eligible selected-construction associations. Context retention does not add a check, conversion, reference exemption or publication point.
4. On an abrupt completion, preserve earlier effects and end the existing operation-local transport. A later instance or declaration uses its own context.
5. Publish alias context only after successful binding initialization. Ordinary binding reads still enforce TDZ and import rules; no getter is executed merely to discover contextual provenance. The public alias value, reflection and structural equality remain unchanged.

The engine retains a resolved context in the field descriptor and in the existing weak binding-publication metadata. This is one implementation of the requirements, not a prescribed object layout or cache strategy.

## Coverage and open work

The corpus covers both binder orders, functions/arrows/methods, public/private/static/accessor/readonly/inherited fields, nested objects/tuples/unions, captures, separate activations, warming, defaults, bounds, explicit own binders, naming, failure recovery, alias chains/shadowing, named imports and re-exports. Forward nominal alias admission remains checked.

Namespace-qualified imported aliases reveal a separate earlier checker omission: their annotation currently supplies no contextual signature before the function body is checked. A runtime-only namespace repair cannot fix that error, so it is not included. The recommended direction is to resolve their actual exported declarations without executing user code and retain the same declaration context; rejecting namespace aliases or inventing name-based fallback would violate the criteria above. Required successful outcomes for the direct and nested namespace cases remain preserved.

Two function-expression decorator probes fail during pre-evaluation while resolving adopted generic annotations. Class-field decorator coverage does not close these expression-decorator cases. Same-child reentrant admission and the imported-constructor guard also remain separate open findings. No broader gate is closed by this increment.

Sources: [Rust generic scopes](https://doc.rust-lang.org/reference/items/generics.html#generic-parameter-scopes), [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html#coercion-sites), and this proposal's contextual typing, type identity, alias publication and selected-construction admission rules.
