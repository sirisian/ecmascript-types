# Terminal helpers on ranges

A range forwards `toArray`, `forEach`, `some`, `every`, and `find` to a fresh forward numeric iterator. These methods use the item contribution `I` from [range-iterator-results.md](range-iterator-results.md), including its endpoint-view, metadata, literal-widening, union and uncertainty rules. The endpoint domain `T` is not automatically the item domain.

| Method | Callback contract | Normal result |
|---|---|---|
| `toArray()` | None | `[].<I>` |
| `forEach(callback)` | `(value: I, index: uint64) => void` | `void` |
| `some(callback)` | `(value: I, index: uint64) => boolean` | `boolean` |
| `every(callback)` | `(value: I, index: uint64) => boolean` | `boolean` |
| `find(callback)` | `(value: I, index: uint64) => boolean` | `I | undefined` |

`find` includes absence even when the range is nonempty. A predicate need not match. `forEach` ignores its callback's result; its contextual `void` imposes no result-value obligation. Its own `void` result cannot initialize a value binding. None of these contracts proves that acquisition or traversal succeeds. `toArray` and `forEach` retain their refusal of an absent or positive-infinite stop before acquiring a cursor; `some`, `every` and `find` can short-circuit on an unbounded range. An empty traversal invokes no callback.

## Choosing the item and result contracts

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Use endpoint `T` throughout | Concise, but falsely excludes interior values for literal endpoint sets. | Rust's associated `Item` is not an endpoint-reading view with this proposal's fallback. | Cheap query, no shape change. | Unsound for literal and some metadata views; does not establish borrows. | **Reject.** |
| Leave every helper gradual | Accepts known result and callback mistakes; tools expose little. | Discards Rust's typed terminal-result contracts. | Misses specialization; no required allocation saving. | Conservative but loses established facts. | **Reject.** |
| Infer from actual endpoints or run a callback | Could appear more precise until effects or replacement change the answer. | Rust trait selection does not invoke predicates to discover types. | Effectful checking prevents stable caching and complicates IC assumptions. | Violates phase separation and can consume state or affect references. | **Reject.** |
| Reuse `I`, ordinary iterator callback contracts, and the results above | Predictable item completion and callback tools; wrong known types fail before execution. | Rust `collect`, `for_each`, `any`, `all`, and `find` separate item and terminal contracts; `Option<Item>` plays the absence role. | Reuses type queries and existing return representations; no new range/cursor shape. | Includes all supported carriers and absence; creates no reference authority. | **Recommend.** No dependency on implementation convenience. |

## Callback adoption is a runtime boundary

A fresh function or arrow expression adopts unannotated positions from its contextual signature. Admit each adopted parameter binding when initialized, after its own default if needed and before a later default or the body can read it. Use the effective optional/default/rest contract: omission can contribute `undefined`, and a rest binding describes its collected tuple or array. Keep the admitted binding's contract for later stores. Destructured bindings retain their existing extraction checks. Written annotations take precedence; reference binding and reference-rest rules remain separate.

This matters even when the original helper supplies valid numeric values. ECMAScript's helper counter is a Number; a contextual `uint64` parameter receives its checked numeric conversion. A replacement helper might call the same callback with a String, a missing index, or an out-of-range value. Admission rejects it before the callback body, using the existing TypeError or RangeError rules. If the helper saves the callback, later calls through `any` must still enforce the declaration's adopted bindings. Generic closures use their captured type context. Checking does not execute a callback or default initializer.

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Publish parameter types only to the checker | Tools show `uint64`, but reflection sees Number and a wrong injected value can reach code checked as numeric. | Rust's inferred closure parameter is a real parameter type, not a checker-only suggestion. | Cheapest entry but permits invalid optimization assumptions. | Unsound after replacement or an erased call; no liveness argument repairs it. | **Reject.** |
| Retag indices in every producer | Makes some built-in examples pass, while untyped callbacks unexpectedly see a different carrier. | ECMAScript counters are Numbers; Rust's `enumerate` explicitly adds `usize`, unlike these built-in callback positions. | Changes shared producer paths and representation assumptions. | Does not guard wrong items, later stores or retained callbacks. | **Reject.** |
| Wrap callbacks per helper invocation | Can check inputs but changes callback identity when a replacement observes or retains it. | Rust monomorphized closure calls do not imply JavaScript wrapper identity changes. | Extra closures, allocation and call IC variants unless proven removable. | A helper-only wrapper misses ordinary contextual function values and complicates reference handling. | **Reject.** |
| Enforce the created literal's adopted binding at initialization | Runtime values agree with the inferred body type; retained callbacks keep their contract. Uncontextualized and explicitly `any` positions remain gradual. | Closest to Rust closure type inference, with this proposal's dynamic conversion boundary added. Rust borrows and JavaScript by-value numeric arguments are distinct. | Reuses parameter admission; engines can specialize proven calls. No public property or function shape is required. | Guards body entry and later stores without rebinding callback identity or granting reference authority. | **Recommend.** General binding rule, not intrinsic-name trust. |

The implementation increment publishes missing simple and ordinary rest binding contracts to the existing inferred-binding path. It does not claim complete context transport for every collection callback. In particular, the current ordinary Array `forEach` path does not establish its index type in the observed static view; that separate transport audit remains open.

## Predicate results and method replacement

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Admit arbitrary truthy results in typed predicates | Familiar to untyped JavaScript, but contradicts the existing Boolean callback signature. | ECMAScript uses `ToBoolean`; Rust `any`, `all` and `find` require Boolean predicates. | Existing coercion is cheap; no shape effect. | Weakens an established typed contract without necessity; no reference benefit. | **Reject for this typed view.** Gradual calls keep existing behavior. |
| Require Boolean for predicate callbacks; ignore `forEach` results | Tools make the decision result explicit. A fresh literal returning an unknown value has a Boolean boundary; an existing function with explicit `any` return does not gain one by passing here. | Matches Rust's Boolean predicate requirement; `for_each` returns unit, whereas this proposal's contextual `void` accepts discarded results. Rust `find` may borrow items for the predicate, unlike these numeric value arguments. | Ordinary function result admission; no new result shape or wrapper. | Preserves existing standard-library signatures and declaration provenance. | **Recommend.** No new narrowing-predicate inference. |
| Treat `range.some` or `Iterator.prototype.some` spelling as proof of identity | Replacement mistakes evade result checks. | Rust trait dispatch is not a mutable JavaScript property read. | Unguarded speedup invalid under property/IC changes. | Unsound; effects and failures can change. | **Reject.** |
| Keep ordinary lookup and required receiving checks | Replacement remains observable; inferred initialization and return reject incompatible results. | ECMAScript mutable methods are the closer analogy here. | Existing property IC guards can support justified elision; no compulsory wrapper. | Preserves evaluation order, failed-binding TDZ, and reference liveness. | **Recommend.** Static views do not prove intrinsic identity. |

Established computed names, finite keys, optional access and extracted method views use the same member contract. Extraction does not bind `this`. A whole `any` receiver does not become participating solely from a familiar method name. Named intrinsic argument routing is a separate existing implementation gap; the parameter name in a static view does not prove that runtime routing is implemented.

Lazy `map`, `filter`, `flatMap`, reduction recurrence, the range's default `Symbol.iterator` static view, and general `bind`/`apply` transport remain separate increments. `take` and `drop` remain range-returning algebraic adapters.

Primary references: [Rust Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html) and [ECMAScript Iterator.prototype.forEach](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-iterator.prototype.foreach), including the neighboring terminal helper algorithms. Performance evaluations above are design analysis, not measured engine benchmarks.
