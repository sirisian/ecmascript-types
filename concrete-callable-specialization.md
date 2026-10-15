# Concrete callable-boundary specialization

A generic function received by a concrete callable type must become the specialization inferred by signature compatibility. Compatibility alone does not perform that conversion:

```js
function id<T: type>(x: T): T { return x; }
const text: (x: string) => string = id;
text === id.<string>; // true

const r: any = 1..<3;
const mapText: (callback: (value: any, index: uint64) => string)
  => Iterator.<string, undefined, any> = r.map;
const callback: any = value => value;
const it: any = mapText.call(r, callback);
typeof it.next().value; // "string"
```

The source is the function actually held at the crossing, including a value reached through `any`. Infer the binding using the same signature-instantiation relation used for conformance. Inputs that already determine a type parameter are not rebound by a wider expected result. Check the declaration's bounds and `where` clauses, then publish its interned specialization. A failed crossing must not replace an existing binding. The function body does not run at the crossing.

The result has the same identity as the equivalent explicit application. Different receiving parameter names or defaults do not produce different specializations: those belong to the callable view used for a call. The source function retains its own defaults. Specialization metadata also retains declared narrowing guarantees, substituted by the same bindings; erasing a predicate does not discard its callable contract. A generic callable target retains the generic value. Ordinary concrete and gradual untyped functions retain their existing rules.

The specialization captures no receiver. Source declaration environments, private environments, realms and captured generic frames govern specialization-time declaration checks; a caller's shadowing binding does not replace a captured one. A later call supplies its receiver normally. Explicit and boundary-created source specializations share this environment rule and their cache. Reference permissions, decay, suspension and liveness rules remain unchanged.

## Decision: materializing a concrete boundary

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Keep the generic function and attach only a concrete view | Completion advertises a fixed result that can disappear after erasure; identity contradicts explicit application | Rust function items identify their type arguments before conversion to a matching function pointer; a JavaScript view is not that conversion | Cheapest store, but each call must infer again and cannot safely exploit the advertised output | Loses the committed binding; no liveness benefit | Rejected |
| Infer again from each call's values | Errors move from assignment to invocation; zero-input generics can lose all inference evidence | Rust does not reinfer a concrete function pointer's generic arguments at each call | Repeated matching and dispatch; misses the proposal's assignment-time optimization | A different call context can change the body or output type; ref modes still need checking | Rejected |
| Create a new target-specific forwarding wrapper | Calls can enforce the view, but aliases and explicit applications compare differently; default ownership becomes confusing | Rust pointer coercion does not create a new identity per parameter label | Extra allocation, call target and inline-cache pressure per receiving view | Duplicates specialization semantics and can accidentally bind receivers or extend captures | Rejected |
| Reuse signature inference and intern the held declaration's specialization at the crossing | Fixed output survives erasure; errors occur before publication; completion matches the resulting value | Closest to Rust's instantiated item and matching pointer signature; dynamic JavaScript callable identity and gradual admission remain distinct | One cached specialization per declaration/bindings; ordinary direct/indirect calls afterwards, with no receiver-dependent cache key or public shape changes | Preserves identity, constraints and the existing body/reference contract | **Selected** |

## Decision: compatibility versus conversion

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Specialize inside membership or the subtype relation | Asking `is` can unexpectedly evaluate a `where` clause or create a value | Rust type compatibility and value coercion are distinct; its static model does not authorize effectful JS membership | Pollutes relation caches and speculative comparisons with allocation/effects | A Boolean relation cannot return the converted value or publish it safely | Rejected |
| Duplicate inference in the runtime conversion | Looks correct until checking and execution choose different bindings | Rust's compiler has one coherent instantiation obligation, not independently evolving compatibility rules | Avoidable repeated inference code and maintenance cost | Divergent bounds, covariance or ref checks can admit the wrong body | Rejected |
| Return the inferred binding plan from the shared relation, then materialize at value admission | Same signature means the same binding; membership remains a question about compatibility | Uses the same distinction between compatibility and coercion, without importing Rust's lifetime or ownership model | Pure inference can be cached/elided; materialization uses the existing function cache | Preserves failure/publication order; successful compatibility is not proof that conversion is unnecessary | **Selected** |

## Decision: environment for specialization-time checks

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Read `where` names in the caller's scope | Extracting a function can produce ReferenceError or silently use a shadowing name | Unlike lexical binding in Rust or JavaScript closures | Cheap lookup in the wrong environment; no legitimate optimization benefit | Violates captured declaration identity and makes cache creation order observable | Rejected |
| Capture the current receiver and caller environment in each specialization | Hides some missing names but gives stored methods surprising receiver retention | Rust ownership is not a model for mutable JS method borrowing | More captures, receiver-dependent caches and polymorphic call targets | Changes receiver independence and may retain reference-bearing state unnecessarily | Rejected |
| Use the held function's declaration context and captured generic frame | Extracted and local uses agree; no caller-dependent name guesses | Consistent with lexical function binding; Rust does not settle JS realm/private-name details | Existing closure/context metadata, no new public shape | Checks the actual declaration and retains normal receiver and reference rules | **Selected** |

Primary references: [Rust function item types](https://doc.rust-lang.org/reference/types/function-item.html) and [Rust function pointer types](https://doc.rust-lang.org/reference/types/function-pointer.html). These support the instantiation/coercion analogy, not JavaScript specialization-object equality, dynamic `where` checks or gradual conversions. Performance discussion is design analysis, not benchmark evidence.

## Implementation coverage and remaining work

The current engine increment materializes a single generic callable at a single concrete signature: direct bindings, assignments, parameters, returns and fields, including equivalent callable interface/object-type spellings. Native Range/Iterator map uses its existing output-specialization cache. Source functions use their existing explicit-specialization cache. Proven-check elision still performs the materialization.

The rule above applies at every actual receiving boundary. [Union admission](union-callable-admission.md) now retains the selected member and materializes a direct callable at that contract. Container membership shortcuts can still skip constituent boundaries inside an already compatible object or array. They need an admission-plan audit that retains already-read values, performs required conversions at actual stores, and avoids repeated getters or mutation during speculative membership. Overloaded source groups, multi-signature targets and general bound/proxy/adopted callable transport also require further coverage; the new materializer does not choose a binding for those forms.

Those are implementation coverage limits, not an exception permitting a concrete view to ignore a required specialization. The specification's general boundary commitment remains in force.
