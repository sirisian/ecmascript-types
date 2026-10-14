# Range and synchronous iterator flattening

`flatMap` receives `(value: I, index: uint64) => R`. A range contributes its cursor item `I` under [range-iterator-results.md](range-iterator-results.md); an established helper-bearing synchronous iterator contributes its item. Establish those callback inputs before inferring `R`, as for [map](range-lazy-helpers.md). Derive the flattened item `F` from the reached iteration protocol of `R`. The result is a helper-bearing synchronous iterator with item `F`, `undefined` completion and optional `any` input to `next`. Its self-iteration has the same protocol arguments. The inner iterator's terminal value is discarded.

## Eligibility and item contribution

Eligibility follows ECMAScript's `GetIteratorFlattenable` with `reject-primitives`. The callback must return an Object. A non-nullish `Symbol.iterator` hook is called to acquire an iterator; a nullish hook selects the callback result itself as a direct iterator. All primitive results, including Strings, fail. This is not Array.prototype.flatMap's scalar-result rule.

The checker reports an eligibility error only when every established callback result alternative fails the reached protocol. A viable or unknown alternative prevents that negative proof. Independently, normal item inference follows these rules without executing callbacks, getters or iterator methods:

- For an established iterable hook, inspect its declared or published callable result, then the returned iterator's `next` result. For an explicitly nullish hook, inspect the original object's `next` result.
- Join all established alternatives. A proved throwing alternative has no normal items and contributes `never`. This includes a selected hook or `next` signature whose argument or receiver contract proves that its actual zero-argument invocation cannot enter; use the same proof as eligibility. A viable or unknown signature still contributes normally. Unknown alternatives keep the item gradual. Empty unions and protocols whose only established steps have required `done: true` contribute `never`.
- A required literal `done: true` excludes that step's terminal value. Other Object steps contribute the read type of `value`, including `undefined` when optional. An undeclared `value` on an open shape is unknown, not necessarily `undefined`.
- An optional hook includes the direct-iterator fallback. An optional `next` includes a throwing absent-method branch. An undeclared hook on an open shape is unknown; absence cannot be assumed.
- A synchronous generator contributes its established yield argument. A finite Composite tuple contributes its positions. An ordinary array or tuple's indexed storage does not establish its replaceable iteration protocol. A separately established protocol contract can supply that evidence.
- Unresolved protocol generics, unknown calls and recursive protocol evidence without a completed contribution remain gradual. An established yielded type parameter, such as `T` in `Generator.<T, R, N>`, can remain symbolic. Do not invent a yielded `T` merely because the whole callback result has type `T`.

A protocol callable returning `void` is unknown evidence, not a proof that its actual result is primitive. A `void`-returning receiving type can hide a function that actually returns an Object. The ordinary prohibition on consuming a known `void` callback result remains independent.

Only one iteration level is flattened. A yielded array or Promise remains an item. An async callback's Promise is not awaited to discover another iterable.

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Use callback result `R` as the item | Completion shows the inner iterable instead of its items; wrong annotations fail late. | Unlike Rust `flat_map`, whose output is the `IntoIterator` item. | Supplies invalid specialization facts; no allocation benefit. | Confuses yielded and terminal values; provides no borrow. | **Reject.** |
| Flatten array storage, retain other results as scalars | Familiar to Array.flatMap users but silently describes a different operation. | ECMAScript iterator flatMap rejects primitives. Rust requires `IntoIterator`. | Cheap storage lookup cannot justify protocol IC assumptions. | Ignores iterator replacement and direct iterator objects. | **Reject.** |
| Require a strict Iterable contract for every alternative | Useful completion but rejects direct iterators and gradual code that may succeed. | Rust has a strict trait bound, but no open JS shapes or mutable hooks. | Easier static dispatch, purchased by changing existing dynamic behavior. | Violates the proposal's conservative negative-proof rule. | **Reject.** |
| Separate eligibility from normal protocol items | Completion reflects actual known yields; uncertain protocols visibly remain gradual. | Rust separates returned `U` from `U::Item`; ECMAScript determines acquisition and failure. | Static traversal only; reusable protocol facts need existing invalidation guards, no new public shape. | Keeps unknown distinct from empty and terminal distinct from yielded; grants no reference authority. | **Recommend.** |

## Unions and incomplete evidence

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Ignore every unknown or invalid arm | Attractive precision hides possible yields from unknown objects. | Rust cannot silently drop an unconstrained trait obligation. | Unsound specialization and check elision; no shape advantage. | Discards reachable normal items. | **Reject.** |
| Let even proved throwing arms erase all precision | Safe but a primitive alternative hides a known iterable's useful item type. | Rust's `Result` is explicit; JS abrupt completion is not a yielded item. | Loses specialization without reducing dynamic work. | Conservative but conflates failure with unknown normal values. | **Reject.** |
| Omit proved throwing arms; unknown arms poison the contribution | Known normal results remain useful without promising traversal will succeed. | Closest to normal-completion typing; Rust's associated item remains the limited analogy. | No new object shape or runtime probing; union work stays in checking. | Accounts for every possible normal yield; reference rules remain independent. | **Recommend.** |

A primitive-only callback still receives an eligibility error. A callback that cannot return normally can have item `never` without such a proof. Either way, helper exhaustion remains `undefined`. An item contract is not a claim of termination or successful traversal.

## Void protocol results

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Treat a protocol `void` result as empty or primitive | Shows no items even when a hidden Object yields values; rejects valid gradual consumers. | Rust's unit is an actual value, unlike this proposal's unusable-result contract. | Creates false specialization facts without a shape benefit. | Violates the existing void callable-type rule; no liveness gain. | **Reject.** |
| Infer a hidden result from the function that once initialized the slot | Precise until the slot or method changes; wrong guesses become effect-sensitive. | Rust's declared callable output does not require inspecting a mutable JS slot history. | Needs value and property guards unrelated to this judgment. | A discarded result contract does not establish a stable yielded type. | **Reject.** |
| Keep void protocol results unknown | Honest gradual completion; actual reached runtime protocol decides. | Preserves this proposal's distinct void semantics rather than importing Rust's unit. | No added shape, wrapper or runtime probing. | Cannot invent empty yields from discarded type evidence; preserves reference rules. | **Recommend.** |

## Carrying the contribution

Computed names, optional access and extracted method views use the same operation contribution when the member is established. Extraction does not bind the receiver. The contribution is computed after the callback result is established; the receiving item's expected type must not be treated as if it were the callback's entire result type.

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Special-case only direct `.flatMap(...)` syntax | Equivalent extracted and computed calls lose item completion and checks. | Rust method references preserve callable meaning. | Cheap parser shortcut, no runtime shape benefit. | Makes source spelling change established contracts. | **Reject.** |
| Always expose a gradual result | No useful item completion even for generators with known yields. | Weaker than Rust's associated-item adapters. | Forfeits available optimization facts. | Safe but discards completed evidence. | **Reject.** |
| Add a new public dependent type operator | Could expose unresolved flattening in arbitrary user signatures. | Analogous to Rust associated types, but this proposal has no equivalent protocol trait obligation. | Requires general substitution, interning and convergence semantics; no runtime shape benefit by itself. | Would require a separate complete type-system design, not just this built-in judgment. | **Reject for this built-in contract.** It is not needed to preserve established call contributions. |
| Carry a built-in operation contribution through method views and derive the call result | Direct and extracted calls agree; unknown protocol evidence remains gradual. | Preserves the returned-type/item distinction without introducing Rust's trait system. | Ordinary inference and substitution; no callback wrapper, eager traversal or new hidden-class requirement. | Completed evidence only, ordinary mutable-binding stability and receiving admission still apply. | **Recommend.** Choice is semantic, not based on implementation effort. |

The proof of concept observes the callback's declared or published result and keeps an internal flattening-operation marker on the signature. It uses the ordinary inference dependency machinery without adopting the inferred `R` as a new return annotation. Its broad unspecialized return view is gradual. This marker is not a reflected public type operator or proof that a replaceable member is the intrinsic. General user-defined dependent flattening and improved unresolved protocol constraints remain separate work. General generic intrinsic application and named intrinsic argument routing remain separate, as recorded for map.

## Result evidence must not create a return obligation

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Adopt inferred `R` as a new callback return annotation | Item completion is useful, but an unannotated callback unexpectedly triggers extra getter reads. | Rust associated item inference does not run a structural validator; JS getters make the difference observable. | Adds structural checks and can disturb property IC behavior. | Changes effects and failure ordering merely to gather evidence; no lifetime benefit. | **Reject.** A development probe observed two iterator getter reads instead of one. |
| Avoid inference and keep all items gradual | Preserves effects but loses useful completion for established protocols. | Weaker than Rust's item inference. | Forfeits static facts without reducing required runtime operations. | Safe but unnecessarily imprecise. | **Reject.** |
| Adopt established inputs and observe declared or published results | Keeps completion and normal input diagnostics without extra inferred-result admission. | Matches the distinction between Rust's type evidence and ECMAScript's effectful acquisition. | Static dependency tracking only; runtime retains existing callback and iterator shapes. | Written return annotations and required receiving checks still apply; inference adds no getter or iterator execution. | **Recommend.** |

A contextual `any` callback result also accepts an async callback's Promise without requiring a written Promise annotation or awaiting it. The async function's explicitly written return annotation still follows the ordinary Promise annotation rule.

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Reject async callbacks at contextual `any` | A gradual result unexpectedly requires extra annotation; completion cannot explain the restriction. | Rust has explicit future outputs but no gradual `any`; the analogy does not justify rejection. | Saves no required runtime operation or shape. | Treats absent result constraints as a written non-Promise annotation. | **Reject.** |
| Synthesize a new Promise return annotation for all such contexts | Accepts the callback but can add obligations the position did not request. | Rust future inference does not imply runtime structural admission. | Adds metadata and potential checks without an IC benefit. | Risks inventing result or rejection contracts; no reference benefit. | **Reject.** |
| Keep contextual `any` gradual and preserve written annotation rules | Async results remain Promises; wrong flattening still fails when reached. | ECMAScript async returns stay intact; Rust's future distinction is only an analogy. | No wrapper, awaiting or shape change. | Preserves context versus declaration provenance and normal reference rules. | **Recommend.** |

## Effects and lifetime

Creation acquires the source iterator but does not call the mapper. Traversal invokes the callback, acquires each inner iterator and yields its items on demand. Preserve ordinary lookup, failures, closing and all prior effects. An inferred result must still pass required receiving admission after replacement; a failed initialization leaves its binding uninitialized. A replacement that saves a fresh contextually typed callback also saves its adopted parameter obligations. Existing callbacks keep their identity and written `any` remains gradual.

None of this consumes an iterator during checking, creates a reference, permits escape, or extends reference liveness. Rust adapters commonly consume their receiver and may carry borrows; these reusable numeric ranges acquire fresh cursors, and this proposal's own reference rules govern any references involved.

Primary references: [ECMAScript GetIteratorFlattenable](https://tc39.es/ecma262/2026/multipage/abstract-operations.html#sec-getiteratorflattenable), [Iterator.prototype.flatMap](https://tc39.es/ecma262/2026/multipage/control-abstraction-objects.html#sec-iterator.prototype.flatmap), and [Rust Iterator::flat_map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.flat_map). Performance assessments are design analysis, not measurements.
