# Declaration provenance of returned objects

An object literal carries declaration provenance from its surviving member contracts. Returning the object can therefore make an otherwise unannotated factory participate in result inference. The same provenance determines whether the literal supplies a typed initializer to an immutable binding. Knowing the object's shape alone does not establish provenance.

```js
function make() {
  class P<T: type> { constructor(value: string) {} }
  class C {}
  function get() { const p: P.<C> = "x"; return p; }
  return { get };
}
const a = make(), b = make();
const holder = { get: a.get };
function replace(target: any) { target.get = b.get; }
replace(holder);
const extracted = holder.get; // TypeError: a different evaluated family
```

`get` publishes this activation's `P<C>` from its annotated local. The shorthand property retains that callable contract, and the object contribution anchors `make`. Its successful result and immutable initialization retain the established view. Replacing the backing property through `any` does not replace that view. The final extraction checks the obtained callable against the established contract; it does not invoke it. Replacing the property with `a.get` succeeds. This case is a missing factory-publication defect, not evidence that generic family identities should be compared differently.

Data properties carry their initializer's declaration provenance, subject to their written or adopted contract. A method carries its non-`any` written or contextual signature positions, or a participating inferred result. Getters and setters retain the provenance of their respective surviving contracts. An explicit `any` remains erased. A plain literal value or literal-only unannotated method supplies no anchor merely because its shape is known. An annotated `any` parameter can make a method participate and publish a known inferred result; that published result can then anchor the object. The annotation's presence and the resulting contract are separate questions.

Apply ordinary descriptor order before deciding whether an anchor survives. For example, the final value in `{ x: typedValue, x: "s" }` has no provenance from `typedValue`. Another surviving typed property can still anchor the aggregate. Replacing a getter retains an applicable setter contract, while a data definition replaces the accessor descriptor. Known primitive spreads replace the copied keys in their ordinary order. A nested object can carry its surviving provenance outward. This does not infer a storage type for an ordinary unannotated `let`.

## Requirements and decision

A solution must preserve contribution-local provenance, descriptor order, lexical and activation identity, explicit erasure, checking phases and receiving boundaries. It must leave literal-only results private and preserve independent reference permissions and liveness. No body, getter, default or initializer may execute to discover a type. Useful autocomplete must survive wrapping a typed result in an object, and an incompatible replacement must fail at the established boundary.

| Direction | Ergonomics and wrong-guess cost | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat every object literal as unanchored | Wrapping a typed callable loses result information and autocomplete; wrong replacements escape extraction checks. | Rust aggregates retain field types; TypeScript infers members and returns. Neither motivates this information loss. | Cheap classification discards already computed contracts; no shape or IC advantage. | Breaks the proposal's returned-contribution and independent-view rules. | Reject. |
| Publish every known object shape | Plain JavaScript factories gain enforced results, and wrong guesses can become new early errors without a declaration-derived result. | TypeScript's broad inference is erased; Rust's statically typed model does not supply the proposal's gradual participation policy. | Extends compilation and runtime admission to previously untyped code. | Violates annotation-seeded participation and compatibility. No liveness benefit. | Reject. |
| Require an annotated factory parameter or explicit result | Predictable but redundant annotations become necessary for an ordinary object wrapper. | Rust generally declares ordinary function results; adopting that policy here would withdraw an existing inference feature. | Reduces inference work by restricting supported code, without improving layouts. | Avoids the defect by changing the proposal's participation commitment. | Reject. |
| Remember any annotation anywhere in the literal | Overwritten properties or unrelated method-body annotations unexpectedly publish a result; autocomplete depends on discarded values. | Rust's retained field contracts and TypeScript inference do not justify treating discarded definitions as surviving data. | A syntax scan is cheap but yields the wrong dependency graph. | Confuses source syntax, control dependence and value provenance; can override erasure. | Reject. |
| Retain provenance with each object-contract query and combine only surviving member contracts | Equivalent shorthand, nested and extracted forms preserve useful contracts. Wrong replacements fail where the established view is consumed. | Rust closures retain typed captures; TypeScript infers member and return contracts. Neither has this exact runtime participation or class-activation model. | Compiler-owned provenance accompanies existing member construction. No public fields, wrappers, user-object shape changes or new inline-cache keys are required. | Preserves publication, descriptor order, independent admission, identity and liveness without source execution. | Choose, independently of implementation effort. |

The production-cost assessment is a design analysis, not a benchmark or a V8 implementation prescription. [Rust closure types](https://doc.rust-lang.org/reference/types/closure.html) describe typed captured environments; their ownership and call-trait rules are not JavaScript's reference model. [TypeScript inference](https://www.typescriptlang.org/docs/handbook/type-inference.html) covers member initialization and function results, but its broader erased inference does not determine this proposal's annotation-seeded runtime boundaries.

## Implementation and audit scope

The checker previously classified every direct object literal as unanchored during result inference. It now records member provenance during the particular shape query, follows descriptor replacement, and uses the resulting provenance for both returned contributions and immutable initialization. It reuses existing declaration, callable-publication and dependency-completion judgments. It does not change runtime family comparison or derive a destination from the replacement value.

The bounded audit covers the original witness, nominal and generic results, same and different activations, nested objects, shorthand and methods, explicit erasure, descriptor overwrites, receiving failure and recovery, Script and Module phases, and ordinary getter effects. Return admission and an inferred immutable-object boundary may each read a getter under their existing rules; provenance computation adds no runtime read. General arrays, unknown spreads, recursive contribution convergence and all declaration-identity contexts remain broader audit work.

## Regression-test correction: specialization predicates

The selected callable-property suite contained an older test expecting a generic specialization predicate to append to an ambient mutable trace through a getter. Both the pulled baseline and repaired engine correctly reject that source before body evaluation under `sec-iscompiletimeevaluable` and `sec-generic-where`. This is a stale test expectation, not a regression in object provenance.

| Direction | Ergonomics | Prior art including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Permit ambient mutation in specialization predicates to retain the old trace | Specialization and cache hits could expose different effects and surprise callers. | Rust constant evaluation restricts external mutable state; its exact context rules differ. | Observable effects obstruct equivalent cached evaluation; shapes and ICs offer no remedy. | Violates the existing evaluability requirement and provides no liveness evidence. | Reject. |
| Change the expected error and drop ordering coverage | Correctly diagnoses the invalid declaration but leaves ordinary getter/setter ordering untested. | Rust's compile-time restrictions do not remove JavaScript's separate runtime observations. | No runtime cost; test coverage is weakened. | Correct negative case, incomplete regression coverage. | Reject as the complete test repair. |
| Retain the original source as an early-error test and add a valid ordering control | Invalid predicates fail before execution; legitimate property effects remain predictable. | Consistent with separating constant-evaluation restrictions from runtime operations; no claim of identical Rust semantics. | No runtime implementation change, public fields or IC changes. | Preserves the predicate rule and checks all reads before writes, including installed specialization identity. Reference rules stay unchanged. | Choose. |

The valid control uses `where true`, records the two getter reads and two setter writes, and verifies that each setter receives the interned `id.<string>` specialization. Its trace is `abAB`. [Rust constant evaluation](https://doc.rust-lang.org/reference/const_eval.html) is a limited comparison: its restrictions are context-dependent, while this proposal supplies its own evaluability judgment for specialization predicates.

## Regression repair: captured nominal member contexts

Two existing extracted-method tests also fail on the pulled baseline. Inside `make<T>()`, initializing an inferred object containing `f<U>(value: U) { return new P<T, U>(); }` can reject the freshly created method against its own displayed signature. The nominal family and explicit arguments agree, but the receiving contract still carries a symbolic captured member-environment input where the method's runtime signature has completed that input. A method-owned `U` shadowing the factory's `U` exposes the same defect.

The immutable initialization boundary must substitute the enclosing runtime frame by both parameter name and declaration identity. A retained nominal capture identifies its outer declaration even when an inner method reuses its spelling. Complete that capture while leaving the method's own binder unbound. Keep the established destination, separate activation identities, ordinary admission and reference liveness; do not execute the method or infer the target from its current value.

| Direction | Ergonomics | Prior art including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Erase generic object-method contracts inside generic factories | Ordinary composition loses result members and invalid replacements escape the intended boundary. | Rust's scoped generics do not justify discarding established contracts here. | Avoids required checks by removing the feature; no layout benefit. | Breaks existing inferred initialization and publication. | Reject. |
| Ignore captured member contexts in comparisons | Some equal-looking signatures pass, but unresolved contexts can be conflated. | Rust distinguishes generic declaration scopes; spelling is not sufficient identity. | A cheaper comparison discards determining inputs and weakens cache keys. | Conceals incomplete substitution and can merge distinct symbolic contexts. | Reject. |
| Substitute every same-spelled parameter from the current name map | Easy to predict only without shadowing; an inner generic method can silently become specialized. | Rust's inner generic scopes demonstrate why equal names need not denote one parameter; its inner items do not capture outer generics. | Small name map, but unsound keys. No shape or IC advantage. | Applies the method's own binder or leaves an identity-tagged outer capture unresolved. | Reject. |
| Use the enclosing frame's declaration bindings for captured inputs and preserve nested callable binders | Factory composition and shadowing retain the intended generic signature; wrong activations still fail extraction. | Consistent with lexical binder identity; the proposal separately defines capturing runtime environments. | Reuse the frame's existing declaration map and type-graph substitution. No new public fields, wrapper or IC key. | Completes the independently established target without relaxing comparison, admission or liveness. | Choose, independently of implementation effort. |

The implementation supplies the runtime frame's retained declaration bindings to the existing free-parameter substitution at inferred immutable initialization. This includes captures nested under object members. [Rust generic parameter scopes](https://doc.rust-lang.org/reference/names/scopes.html#generic-parameter-scopes) are a scoping comparison, not a model of this proposal's capturing factories. The original two tests remain unchanged; focused variants add nested storage, reused and distinct activations, mixed enclosing arguments and reflection.
