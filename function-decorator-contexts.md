# Context for decorated function expressions

A decorator on a function literal does not remove the construction context supplied by its position. The literal's own annotations, body and created closure use that context before the decorator observes it.

```javascript
function inspect(context: Reflect.Function) {}
class C {
  f: <U: type>(value: U) => U =
    @inspect function(value: U): U { return value; };
}
const first = new C(), second = new C();
```

Each initializer creates its own generic function and invokes the decorator once. `U` is the effective own binder supplied by this field, not an unresolved global name or another field's alpha-equivalent binder. A thrown decorator leaves earlier effects intact and does not supply an initializer result. Another instance uses its own construction context.

## Required behavior

Carry the destination's established context through the decorated operand in checking and evaluation, including parentheses. Do not pass it into decorator expressions or their written arguments. Explicitly written type parameters retain precedence. Captured generic arguments retain their declaring identities and activation values. A decorator's potential replacement does not excuse an invalid original body or defer its required pre-evaluation diagnostic.

A decorated function literal receives `Reflect.Function`, including the actual function, its name where written, metadata and effective signature reflection. Read the signature using the created closure's own/adopted parameters and captured frame. Unwritten parameters and results use their established effective contracts; reflection must not silently omit an adopted type because a bare source annotation was resolved outside its binder scope. Ordinary reflection does not execute the function body or its parameter initializers.

Apply decorators in their existing evaluation/application order. Returning `undefined` keeps the current function. Each replacement must satisfy the original function contract before the next decorator can receive it; a mismatch throws `TypeError`. An outer decorator's `type` field names the accepted inner replacement. Updating that public field uses ordinary own-data-property definition; refusal, such as a decorator freezing the context, throws before another application rather than silently exposing a stale function. The receiving binding, field, argument or return still performs its ordinary admission. Do not contextually recreate a replacement function in the decorated literal's scope or attach the original construction choice to it, even if the decorator returns the same function object.

The original producing completion retains its construction associations when no replacement occurs. Such associations remain requests for receiving admission, not certificates that decorators preserved the value's backing. Failure closes existing operation-local state without undoing prior effects or extending reference lifetimes.

## Directions considered

Every valid solution preserves declaration identity, existing type equality, original-body checking, decorator order, ordinary receiving checks, independent activations, and reference permission/liveness.

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art, including Rust | Real-engine performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Propagate the position's context into the literal and reflect its effective closure signature | Adding a metadata decorator preserves the parameters suggested by the typed position. Wrong original bodies fail at their existing phase. | Rust propagates expected types at specified coercion sites and scopes generic binders to their declarations. It has no equivalent runtime function decorator or this proposal's adopted generic closure signatures. | Reuse compiler context operands and resolved records; no user-object shape change, wrapper function or extra polymorphic property access. | Keeps source binders, captures and runtime adoption aligned. No additional borrow authority. | **Recommend**, independently of effort. |
| Require explicit type-parameter lists whenever a function is decorated | A harmless decoration invalidates otherwise supported code and duplicates the surrounding signature. | Rust's lexical scope rules do not justify a decorator-specific loss of context. | Saves a context edge but adds no shape/IC benefit. | Contradicts contextual construction at the existing destination. | Reject. |
| Suppress the unresolved annotation or check only after decoration | Autocomplete advertises a generic body that is not checked; a wrong body may fail only when called. | Rust rejects invalid generic bodies independently of whether they execute. Runtime decorators are not Rust macros. | Delays work and complicates optimization assumptions. | Can admit invalid unused bodies or lose required diagnostic ordering. | Reject. |
| Resolve reflection directly from syntax in the surrounding scope | Binder names may resolve to a same-spelled outer declaration or disappear. | Rust generic names belong to their declaration; token spelling is not binding identity. | Repeats resolution and risks needless type-builder work; no IC advantage. | Loses adopted binders and activation-specific captures. | Reject. |
| Rename binders to the canonical Type Object or store ambient context on shared syntax | Equivalent types become dependent on which source or instance ran first. | Rust item scopes do not make a shared structural type representative the owner of other declarations. | Introduces mutable caches/invalidation or extra identity distinctions. | Breaks alpha-equivalence or isolation after failed/reentrant construction. | Reject. |

## Replacement and transport choices

| Direction | Ergonomics / wrong guesses | Prior art, including Rust | Real-engine performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Validate each replacement, expose it to the next decorator, then retain receiving admission | A bad inner decorator fails before an outer decorator consumes its incompatible result. Tools may continue to expose the original contract. | Rust has no runtime replacement decorator; typed function boundaries are the limited analogy. | One contract guard per actual replacement; engines can specialize known signatures. No wrapper or changed function-object shape is required. | Preserves chain order and original contract without transferring stale construction evidence. | **Recommend.** |
| Ignore replacements or label every expression as Object | `Reflect.Function` overloads do not run and valid replacement decorators have no effect. | No relevant Rust feature supports erasing a declared function context. | Cheap but implements the wrong target. | Violates the Function row of the existing reflection/replacement tables. | Reject. |
| Validate only the final value at its destination | An incompatible inner result can reach another decorator; an untyped destination can accept a non-function. | Rust typed boundaries do not justify removing the intermediate contract here. | Fewer guards but different observable effects. | Violates the replacement contract before the destination is reached. | Reject. |
| Reuse the original literal's construction choice for any replacement with equal identity or type | A decorator's new contribution can silently inherit an earlier union choice. | Rust coercion sites belong to source expressions, not arbitrary equal values. | Avoids transport bookkeeping but can authorize stale assumptions. | Confuses value identity with the producing contribution and may evade required checks. | Reject. |

These are semantic requirements, not a mandated engine representation. Cost statements are design assessments, not benchmarks. The repair covers whole function-literal decoration and established contextual paths. It does not settle the separate TC39-decorators flag, all function sub-target decorations, function naming propagation, every replacement context family, or arbitrary stored callable expressions. Standalone callable-union adoption is not established by the selected object-union coverage.

Sources: [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html#coercion-sites), [Rust generic scopes](https://doc.rust-lang.org/reference/items/generics.html#generic-parameter-scopes), [Decorators](decorators.md), [Field binder contexts](field-binder-contexts.md), and [Contextual union selection](contextual-union-selection.md).
