# Effective callable results at structural boundaries

A structural interface checks the callable value read from each method member using its effective signature. A declared return and an established published inferred return both constrain membership. An unannotated function that publishes no result retains the existing gradual catch-all, subject to the reference and narrowing rules.

```js
interface Has<T: type> { get(): T; }
class E {}
function text(): string { return "s"; }
let value = { get() { return text(); } };
value is Has.<E>;      // false
value is Has.<string>; // true
```

The checker has published `string` for `get`. Losing the receiver object's static type does not erase the function value's own contract. An explicit `get(): string` has the same result. In contrast, a wholly unannotated method that merely returns a string literal need not participate in inference; its unknown result still receives the existing gradual treatment. This repair does not make inference universal.

A return annotation alone makes a function typed. Its annotation is on the function's source declaration; absence of a copied annotation on an implementation's function object is not evidence that the source omitted it. This also matters when deciding whether a function is an untyped overload fallback.

## Requirements and decision

A valid solution must preserve published results, parameter and receiver compatibility, generic binder and nominal identity, ordinary property-read effects, function identity, existing inference participation, and independent reference permissions and liveness. It must not execute a candidate method or invent sample arguments. Both direct membership and membership through an object, getter, Proxy, nested object or array use the same callable relation.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Check callability only | Autocomplete promises members of `E` after a successful test even though the known result is a string; a wrong guess fails later | Rust's callable traits constrain the output; TypeScript also compares function results | Cheapest check, but no usable result guarantee for optimization | Discards explicit and published contracts; supplies no reference evidence | Rejected: contradicts existing membership rules |
| Check explicit annotations but ignore published results | Extracting a helper or omitting a redundant return annotation changes membership despite the same reported result | Rust closure output inference does not remove the output constraint; TypeScript checks inferred function results too | Saves a field lookup while wasting already established information | Conflicts with published-result reflection and subtyping | Rejected |
| Invoke the method with trial arguments | Tests unexpectedly perform application work and may throw, mutate or never return | Neither Rust callable compatibility nor TypeScript function compatibility samples executions | Arbitrary and potentially unbounded cost; disrupts optimization and observable execution order | One observed result proves no contract; arguments, receivers and references cannot safely be invented | Rejected |
| Install wrappers or instrument all future calls | Can offer a stronger gradual contract, but may change callable identity or where failures appear | Contract systems are a closer analogy than Rust; TypeScript supplies no such runtime wrapping | Wrapper allocation/dispatch or call-site guards; changes call-target inline-cache behavior | Could police unknown results, but requires a separate identity, aliasing and blame design; still cannot grant reference lifetime | Rejected for this question: unnecessary to compare information already known |
| Compare the effective signature, retaining the existing unknown-result rule | Runtime tests, reflection and established static facts agree; autocomplete can use the published result; incompatible known results fail at the boundary | Matches the relevant Rust and TypeScript result-compatibility principle, within the limits below | Reuse per-function signature metadata and normal relation caches; no new public fields, object shape changes or call wrappers | Preserves known results, captures, generic/nominal identity and ordinary receiver/reference checks; no body execution or lifetime extension | Recommended independently of implementation effort |

Choose the last direction. Runtime reflection may normalize an effective result into its signature's return field, while a declaration description keeps a separate inferred-result field. The catch-all must recognize either representation. Do not move inferred results into declaration identity or overload ranking to repair a runtime comparison. A return-only annotation is recognized syntactically even when resolving its type is deferred.

Rust's [closure expressions](https://doc.rust-lang.org/stable/reference/expressions/closure-expr.html) allow omitted annotations, and its [closure call traits](https://doc.rust-lang.org/reference/types/closure.html#call-traits-and-coercions) describe typed calls. This supports retaining a known output, but Rust does not have this proposal's arbitrary JavaScript object membership or gradual unknown-result catch-all. TypeScript's [function compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html#comparing-two-functions) likewise compares results structurally; its checks are static, not runtime property observations.

## Effects and scope

Membership reads the method property once under the existing structural rules. Accessor and Proxy effects remain observable, and abrupt property reads propagate. Signature comparison does not invoke the method, bind it, replace it or impose new storage on its containing object. A failed boundary keeps prior effects and leaves the failed destination uninitialized. A later independent check still uses the callable's own captured context. A successful check supplies no additional reference authority or lifetime.

The implementation repairs the catch-all tests before generic and concrete signature comparison and recognizes source return annotations when constructing overload signatures. The bounded corpus includes declared and inferred results, unknown controls, erased admission, direct functions, member receivers, imported and captured publications, getter/Proxy effects, nested aggregates, nominal identity, failure recovery and overload fallback classification. Broader inference convergence and checking-phase work remain separate.

## Regression repair: nested inferred result dependencies

The bounded regressions also reproduced a pre-existing failure for `const x = f()` when `f` publishes `P<C>` and `C` belongs to the producer's local activation. Resolving the known outer `P` before consuming the checked result dependency made the target appear complete while its `C` remained unresolved. Admission then treated the existing instance as a nonmember and attempted a second converting construction. Explicit `new P.<C>(...)` inside the producer had the same failure, so changing construction syntax would not repair the dependency.

A valid solution must preserve the source declaration and activation of each argument, retain written and already completed destination contracts, use only checked completion evidence, respect generic constraints/defaults in their declaration context, and perform no discovery through candidate values, constructors or references.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Erase unresolved arguments or identify classes by spelling | A seemingly valid assignment can silently mix unrelated local classes; autocomplete retains an unjustified type | Rust generic arguments are part of the instantiated type; spelling is not sufficient identity | Saves resolution work but removes useful specialization guards | Violates nominal invariance and activation identity | Rejected |
| Demand explicit destination annotations or explicit construction | Ordinary inferred results cease composing through wrappers; explicit construction already fails in the witness | Rust inference can determine concrete generic arguments; it does not justify requiring redundant syntax here | Moves burden to source code without addressing retained evidence | Avoids some uses but leaves the required inference contract incomplete | Rejected |
| Resolve the outer family first and regard any constructor-bearing record as fully complete | Fails at a later boundary with an unrelated constructor-parameter error | A known generic definition is not a completed argument list in Rust; the analogy does not specify runtime ordering | Can trigger an unnecessary second construction and its effects | Discards an unresolved nested dependency despite available checked evidence | Rejected |
| Complete matching checked result dependencies before argument normalization | Inferred values and forwarded results retain the producer's type; wrong actual identities still fail | Consistent with preserving concrete arguments in a generic instantiation; unlike Rust, this proposal must transport runtime activation identities | Reorders existing metadata work; no new public fields, shape transitions or inline-cache invalidation rules | Preserves completed targets and independent generic/borrow checks; no candidate-value authority, user-code discovery or lifetime extension | Recommended independently of implementation effort |

Choose the last direction for both inferred function returns and immutable initialization. A known outer generic family is still an authority check: only checked evidence from that same family can complete its unresolved nested arguments. A completed specialization or completed argument cannot be replaced by this operation. This changes neither the generic constraint judgment nor the unknown-result rule. It consumes only evidence retained after the producer's successful boundary; it does not choose a receiving contract from the returned object. See [Rust closure types](https://doc.rust-lang.org/reference/types/closure.html) for the limited captured-environment analogy; Rust does not expose the same dynamic class-declaration activation model.
