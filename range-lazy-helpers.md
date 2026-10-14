# Lazy range mapping and filtering

`map` and `filter` forward to a fresh numeric iterator and return helper-bearing synchronous iterators. Use the item contribution `I` from [range-iterator-results.md](range-iterator-results.md), not the endpoint domain `T`. Mapping supplies `(value: I, index: uint64)` to its callback and yields the callback's inferred result `U`. Filtering requires a Boolean callback with those inputs and keeps `I`. Both helpers complete with `undefined`; their optional `next` input is `any`. Self-iteration has the same protocol arguments. Reading `.next().value` therefore includes exhaustion.

These are normal-result contracts, not guarantees of successful acquisition, callback completion, or eventual termination. Creation does not invoke the callback. Traversal invokes it on demand; filtering can consume multiple source items before yielding. The mapper's returned Promise is an item, not an instruction to await it. These rules add no narrowing-predicate inference to `filter`.

## Choosing the output contracts

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Give both helpers endpoint `T` | Misleads completion and rejects mapping to another type or yielding interior values. | Rust separates `Item` from `map`'s output. | Cheap but supplies invalid optimization facts; no shape benefit. | Confuses endpoints, source items and mapped outputs; establishes no borrow. | **Reject.** |
| Leave both helpers gradual | Allows known callback and result mistakes; loses useful chained completion. | Weaker than Rust's typed adapters. | Prevents valid specialization without removing an allocation. | Conservative but discards established contracts. | **Reject.** |
| Derive output by executing callbacks or inspecting yielded values | Types depend on effects and which branch runs first. | Neither Rust trait selection nor ECMAScript helper creation executes the mapper to discover a type. | Effectful checks obstruct caching and can consume an iterator. | Violates phases, laziness and failure order; may disturb live references. | **Reject.** |
| Map to inferred `U`; filter preserves `I`; include `undefined` completion | Predictable chains and direct result diagnostics. | Rust `map` changes `Item`; `filter` preserves it. ECMAScript supplies the mutable, lazy runtime protocol. | Reuses callable inference and helper objects; no new public shape or per-call wrapper. | Matches actual normal yields and exhaustion; grants no reference authority. | **Recommend.** |

Rust's `filter` predicate receives `&Item`, whereas these numeric callbacks receive values. Rust's adapters consume their iterator receiver, while a range here remains reusable by acquiring fresh cursors. Its `Option` termination protocol is analogous to the discriminated iterator result, not to a JavaScript `void` return. The Boolean predicate policy and contextual parameter admission remain those of [terminal helpers](range-terminal-helpers.md).

## Inferring the mapper without a circular obligation

A fresh callback literal can infer a callee-owned output parameter after adopting already established input positions. For a single callback signature `(I, uint64) => U` with an unbound result parameter `U`, first establish the inputs, infer the body result, then bind `U` and apply ordinary argument and result obligations. Do not demand that an identity body of type `I` already satisfy unresolved `U`. A written callback annotation still takes precedence. An existing callback contributes its established signature; passing it does not retroactively annotate it. An explicit `any` result stays gradual.

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Check the body against unresolved `U` first | Rejects `map(v => v)` despite known input; encourages unnecessary annotations. | Unlike Rust's inference from a closure output. | Cheap failure, no useful performance advantage. | Treats an inference variable as a completed obligation. | **Reject.** |
| Infer the literal without its known inputs | Identity and block bodies lose their result type; completion is weak. | Discards closure context used by Rust. | Loses specialization; no runtime shape advantage. | May be conservative but discards available evidence. | **Reject.** |
| Erase the whole callback to `any` permanently | Stops useful input and output errors. | Neither Rust nor the proposal treats an unsolved variable as a permanently unknown result. | Sacrifices optimization facts. | Loses adopted binding obligations and conflates pending with gradual. | **Reject.** |
| Establish independent inputs, infer result, then solve and check | Identity, block and function-expression mappers behave alike. | Closest to Rust's contextual closure inference, with dynamic admission added. | Static query work only; reuses normal entry/return checks and existing IC guards. | Preserves annotations, declaration captures and reference rules; executes no callback. | **Recommend.** Unsolved input dependencies still require the ordinary inference process. |

The implementation covers a monomorphic callback whose return is a bare open parameter of the callee and whose input types do not depend on the callee's remaining unsolved parameters. This is a sufficient case, not permission to substitute `any` for arbitrary recursive dependencies or infer higher-rank callbacks by guessing. Ordinary user generic functions with this shape use the same rule.

## Receiving contracts, substitution and replacement

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Expose only an internal carrier name or close only its argument list | Direct `.next` can disagree with self-iteration; admission can search for a nonexistent global constructor. | Internal Rust adapter types still implement the full iterator protocol. | Superficially cheap, but cannot justify optimized protocol access. | Leaves unresolved variables in a supposedly closed result. | **Reject.** |
| Replace all results with a bare gradual protocol | Avoids the unresolved variable but destroys chains and item checks. | Weaker than either typed adapters or the proposal's established views. | Removes useful static facts with no necessary allocation saving. | Conservative but incomplete. | **Reject.** |
| Close the carrier arguments and its expanded receiving shape together | Direct, self-iteration and annotated receiving paths agree. | Preserves the adapter/protocol relation; Rust has no mutable JS prototype analogy. | Static substitution only; no runtime shape or wrapper requirement. | Keeps the full structural contract and existing runtime admission. User declaration templates retain their own binder handling. | **Recommend.** |

Ordinary property lookup and effects remain observable. A method name or inherited method does not prove intrinsic identity. Keep required inferred initialization and return admission; incompatible replacement results cannot initialize a checked binding, and prior effects remain after failure. A replaced helper that saves a fresh contextual callback also saves its adopted parameter obligations. The callback keeps its identity. No checking step consumes the helper, grants reference access, or extends a lifetime.

## Boundaries still open

`flatMap` eligibility and established flattened contributions are specified in [range-flatmap.md](range-flatmap.md); reduction preserves the shared accumulator contract in [range-reduce.md](range-reduce.md). The default iterator is specified in [range-default-iterator.md](range-default-iterator.md); named intrinsic arguments now follow [named-intrinsic-routing.md](named-intrinsic-routing.md), while general `bind`/`apply` transport remains separate. This increment improves shared synchronous helper signatures, but does not establish async helper semantics.

Explicit mapper application now follows [intrinsic-generic-map.md](intrinsic-generic-map.md), including stored specializations, lazy output admission and ordinary generic delegation. The original alternatives below remain recorded; the selected direction is implemented for `map`, while other intrinsic generic capabilities remain separate.

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art | Real-engine performance | Correctness / reference liveness | Other / decision |
|---|---|---|---|---|---|
| Permanently forbid explicit output arguments | An advertised generic method cannot use ordinary generic syntax. | Rust permits explicit generic arguments; the existing proposal promises generic callable consistency. | Avoids work but supplies no semantic optimization benefit. | Leaves static and runtime callable capabilities inconsistent; no liveness gain. | **Reject as the design.** Current implementation limitation only. |
| Register the intrinsic and ignore supplied arguments | Appears to work until a result violates the written output contract. | Explicit Rust type arguments impose obligations. | Cheap identity check but no justified shape/IC advantage. | Omits admission and may trust replaced methods; no liveness gain. | **Reject.** |
| Carry a real generic callable contract through application | Ordinary syntax has its ordinary meaning; wrong outputs fail at their required boundary. | Closest to the proposal's generic functions, with Rust as a type-argument analogy only. | Requires callable metadata and existing lookup guards; no necessary public shape or wrapper change. | Must preserve lookup, identity, effects, result admission and reference rules. | **Recommend.** Implemented for explicit `map` application; see the linked contract. |

Primary references: [Rust Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map), including `filter`, and [ECMAScript Iterator.prototype.map](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-iterator.prototype.map), including `filter`. Performance assessments above are design analysis, not benchmarks.
