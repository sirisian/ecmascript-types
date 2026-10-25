# Construction at class-field initialization

A field annotation supplies the construction context of its initializer. If contextual checking selects the generic member of an overlapping structural union for a fresh object, initialization must check that selected member. It must not specialize the object to a different member merely because evaluation passes through the specification's synthetic initializer function.

```js
type Item = { fn: (value: string) => string }
          | { fn: <U: type>(value: U) => U };
class Example {
  item: Item = { fn(value: U): U { return value; } };
}
```

The pending choice belongs to this evaluation and receiving field. The stored contract remains `Item`. Reversing the union does not remove the generic capability chosen for the literal. Public, private, static, inherited and auto-accessor backing fields follow the same admission rule.

## Requirements and directions

Preserve declaration identities and captured arguments, once-only initializer evaluation, inferred function names, class-element ordering, object identity, already performed effects, and reference liveness. Admit before publishing the field. A selected-member failure cannot retry another union member. A later ordinary read, independently invoked callback or field-decorator replacement cannot borrow the skipped initializer's choice. Nested data contributions must still be live at admission, including after decorators.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Carry the producing completion through the synthetic return and call to field admission | The field behaves like a typed binding. Autocomplete can show the chosen generic signature; an invalid value fails when initialized. | Rust treats struct, union and enum field initializers as coercion sites. ECMAScript models a field initializer as a call. Rust does not model runtime structural unions or observable JavaScript getters. | Optional internal inputs can be registers or stack slots. No public properties, wrappers, shape changes or new polymorphic property reads. | Preserves the source occurrence, captured binders, existing order and liveness checks; admission remains at the field boundary. | **Recommend**, independently of implementation effort. |
| Admit inside the initializer's synthetic return | Similar happy-path result, but internal function machinery becomes an observable conversion boundary. | Rust's static coercions do not justify moving JavaScript's observable checks. | Can duplicate admission and getter work; introduces a second boundary to optimize. | Splits the field operation and risks changed failure order and contextual ownership. | Reject. The synthetic call transports the result. |
| Attach the choice permanently to the constructed object | Subsequent unrelated fields depend on hidden history that autocomplete cannot explain. | An explicit Rust enum discriminant belongs to its representation; this proposal's overlapping structural union has no such tag. | Tags can perturb shapes/ICs; side tables still retain history and require lifetime management. | Confuses value identity with source occurrence and invalidates replacement rules. | Reject. |
| Keep ambient state while running the initializer or checking the field | Simple cases work, but nested constructors and getters can consume another field's choice. | Neither Rust coercion sites nor ECMAScript's explicit field operation imply cross-call authority. | Requires nesting, abrupt-exit cleanup and callback masking on a hot path. | Violates reentry isolation and can mix declaration contexts. | Reject. |
| Discard the choice or require annotations only in class fields | A literal working in a binding loses generic capability in a field; a wrong guess can fail only when called. | Rust's field coercion sites support destination context, with the dynamic limitations above. | Less transport, but may perform wrong specialization or repeat checks. | Contradicts selected-construction admission and the field's contextual type. | Reject. |

Private storage exposed a separate implementation omission: initializers were published without the declared field check. Visibility cannot exempt storage from admission.

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Apply declared admission before publishing public and private/backing fields | A `uint8` field contains an admitted `uint8` regardless of visibility; wrong dynamic values fail during construction. | Rust checks private and public field initialization against the declared type; privacy controls access. JavaScript private storage still has its own publication rules. | Same declared check; no additional public shape or wrapper. | Consistent type invariant, original union storage contract and failure state; preserves private publication checks. | **Recommend.** This repairs a missing check. |
| Rely on the checker alone for private fields | Dynamic or erased inputs can leave invalid values behind. | Rust's static guarantee cannot be transplanted to this proposal's `any` boundaries. | Saves a runtime check at the expense of the invariant. | Unsound for unknown values and misses required numeric conversion. | Reject. |
| Check only on the first read or later write | Errors move away from construction and depend on whether the field is used. | Neither Rust's initialization rule nor this proposal's typed-storage contract makes private fields lazy. | Adds read-path guards and state, complicating IC assumptions. | Publishes invalid storage and changes effects and failure timing. | Reject. |

## Boundary behaviour

1. Use the field's established declaration context. Evaluate its initializer once, with ordinary `this`, `super`, captured generic environment and anonymous-function naming.
2. Preserve the initializer result through the internal return and call. That transport performs no admission. An ordinary stored-source read supplies no fresh pending choice.
3. At the field's declared-type check, provide that result's eligible associations explicitly. Validate live nested data contributions, require the selected member without fallback, and preserve the declared union as the field's storage contract.
4. Publish only the admitted value. Private and auto-accessor backing fields have the same obligation as public fields. An initializer or admission failure prevents this field and subsequent elements from being initialized; earlier fields and effects remain. Ordinary private-name, extensibility, placement and reference-liveness rules continue to apply.
5. End pending transport on every exit. A nested constructor or getter starts its own operation. Field-decorator replacement values and type defaults are ordinary sources at their existing checks, with no choice inherited from an initializer that did not run.

## Coverage and limits

The regression corpus covers both union orders and function, arrow and method literals, the field visibility/static/accessor forms, nested objects and tuples, naming, generic captures, imported declarations, identity, replacement, reentry, dynamic numeric admission and failure state. Two expanded probes remain open: differently named generic binders on adjacent field signatures, and reentrant admission of the same escaped child during a sibling getter. Their required outcomes remain recorded separately; passing independent-source reentry does not close them. It exercises this proposal's decorator semantics, where a class-field decorator can replace the initial value. It does not claim interoperability with the separate experimental TC39-decorators feature flag, nor a complete audit of buffer/SoA storage, reference-field admission, recursive inference or all publication rollback.

Sources: [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html#coercion-sites), [ECMAScript DefineField](https://tc39.es/ecma262/#sec-definefield), and this proposal's selected-construction admission, typed classes, defaults and decorator replacement rules. Rust provides a useful destination-context and privacy analogy; the operation-local runtime association is specific to this proposal.

## Regression repair: selected owner bindings

The selected regression suite exposed an older dispatch defect: an explicit call with a captured specialization case could select the primary owner's fallback, then lose its already supplied type argument during body inference. Rebuilding the selected frame had kept the argument's spelling but omitted its declaration identity. The implementation now binds both type and value parameters to their actual declarations when constructing that frame; type arguments require no numeric conversion.

| Direction | Ergonomics | Prior art, including Rust | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Retain the selected declaration's argument identity in the rebuilt frame | An explicit argument remains explicit for direct and stored calls, including under a same-named outer parameter. | Rust generic arguments bind a particular item declaration; the analogy supports lexical identity, not this proposal's runtime case dispatch. | Internal metadata only, with no object shape/IC change or new user-code evaluation. | Preserves existing selection, constraints, defaults, captures and liveness; prevents accidental re-inference. | **Recommend.** This implements the existing identity rule. |
| Treat any same-spelled ambient parameter as the argument | Simple calls may work but nested calls silently use another declaration's argument. | Rust lexical generic scope does not equate binders by spelling. | Cheap lookup, incorrect sharing. | Violates declaration identity and shadowing. | Reject. |
| Infer again from value arguments or defaults | Explicit no-argument calls can fail or change meaning. | Rust does not discard explicit arguments when a body is selected. | Repeats inference and potentially constraint/default work. | Loses supplied arguments and can change failure/effect order. | Reject. |
| Re-run selection or tag the callable value with an ambient argument | Hidden history affects later calls or repeats selection effects. | Rust item identity is not ambient invocation state. | Extra dispatch or retained mutable metadata; poor call-site specialization. | Conflates distinct invocations and can change captured contexts. | Reject. |
