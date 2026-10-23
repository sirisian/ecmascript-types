# Instance initializer flow

An instance field initializer is a deferred callable body. Checking the class does not execute it. Construction runs it in the ordinary order, with its ordinary assignments, updates, reads, throws and reference-liveness checks. A source-position counter fact established before construction cannot survive a possible initializer write merely because the class has no explicit constructor.

Keep three boundaries distinct:

1. Class evaluation evaluates its computed names, decorators, static fields and static blocks in their specified contexts.
2. Checking an instance initializer uses captured declaration contracts and facts proved stable for later invocations. A mutable captured value or a reference's declaration-time location is not such a proof. An immutable ordinary value can retain its established fact; constness does not freeze object members or a reference's referent. Checking restores the enclosing flow afterward.
3. Construction transfers the possible effects of constructors and instance initializers, including writes through aliases. The transfer applies in loop widening, exceptional entry and finalizers too. Declaring a class in a loop does not run its instance initializer.

This is checking information, not an instruction to run source early. It changes neither field initialization order nor receiving checks, storage rules, or reference lifetime. An initializer that throws retains its already performed runtime effects; a failed static check performs none of them.

## Checked-code classification is separate

A unit is checked when its own syntax contains proposal syntax or it inherits checking from an enclosing unit. Discovery excludes nested units. Thus a typed class does not by itself make its enclosing module checked. Importing a typed value does not add proposal syntax to the importing module either.

A later top-level annotation does make the entire module checked, including earlier guards. The earlier construction audit's typed-field and typed-getter consumers exposed stale facts by activating those diagnostics. This source-order-independent classification is intentional; the stale facts were the bug. Add a top-level `type Audit = number;` when a control must explicitly test required module-level narrowing errors without changing runtime counter effects.

The two earlier import-precision witnesses also had incorrect expectations: their original importing modules were unchecked, so fulfillment was required. Their counterparts with an explicit top-level type declaration correctly reject the redundant guard. This closes those two findings by correcting the audit, not by changing checking policy.

## Decision

A valid solution must invalidate possible construction writes, preserve unaffected enclosing facts during declaration, exclude unstable declaration-time captures from initializer checking, preserve proved immutable value facts, retain static/computed-name timing, and leave checked-code classification, receiving checks and reference liveness intact.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Production-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Count only explicit constructors and functions | Completion can retain a stale counter or captured value; valid guards fail | ECMAScript fields have implicit initializer functions; Rust constructs fields through expressions rather than this class mechanism | Cheap incomplete inventory; can authorize invalid load reuse | Misses implicit callable writes | Rejected |
| Transfer instance-body effects at class declaration | Completion loses facts before any instance exists; wrong timing obscures errors | Neither ECMAScript's deferred instance initialization nor Rust's expression evaluation justifies running a future initializer at declaration | Simple scan but needlessly broad invalidation | Confuses declaration and invocation; leaks effects and captured assumptions | Rejected |
| Discard every fact at class declaration and construction | Avoids some stale facts but loses required diagnostics and useful completion for independent values | Conservative analyses are familiar; Rust's guarantees do not require erasing unrelated immutable values | More checks and reduced optimization opportunities, without a shape/IC benefit | Loses valid preservation proofs and the declaration/instance distinction | Rejected |
| Execute or snapshot initializers while checking | Completion depends on source execution and future instances cease to observe live captures | Rust type checking does not execute ordinary initialization expressions to discover future values; ECMAScript initializes each instance at runtime | Arbitrary execution/allocation cost; possible shape and IC disturbance | Breaks phases, counter/throw order, initialization and reference liveness | Rejected |
| Make enclosing/importing modules checked automatically | Previously ordinary JavaScript starts rejecting guards because of a nested class or an import; completion knowledge becomes a checking-mode switch | Rust has no analogous opt-in proposal-syntax boundary; that analogy cannot override this proposal's explicit rule | More checking, no repair to effect summaries or shapes/ICs | Contradicts checked-unit discovery and still leaves stale effects | Rejected |
| Exempt post-construction guards from diagnostics | Hides incorrect guesses even when immutable facts prove the answer | Rust's diagnostic policy differs and cannot justify an ad hoc exception here | Cheap syntax special case, with no valid optimization information | Removes required errors rather than fixing their inputs; liveness remains unrelated | Rejected |
| Isolate initializer checking, retain stable captures, and transfer possible writes at construction | Completion reflects each source position; correct guards work and proved redundant guards still fail | Follows ECMAScript's initializer-function boundary; Rust is useful for separating values, storage and evaluation, but has no equivalent JavaScript class field mechanism or alias guarantees here | Compiler-owned effect/dependency summaries; existing constructor/field lowering and IC guards remain; no required object fields, wrapper or shape/IC-key change | Preserves source order, static timing, captured contracts, alias invalidation, failure state and liveness | Recommended independently of implementation effort |

Changing all enclosing modules to checked, or exempting post-construction guards from diagnostics, is also rejected. Both would change which existing JavaScript programs receive type errors while leaving the effect defect unresolved. Autocomplete may know imported contracts without that knowledge changing a source unit's checked status.

The interpreter carries these distinctions in existing flow snapshots and the captured-write inventory. A production engine may use sparse effects or versioned dependencies; the required judgments do not depend on that representation.

Sources: [ECMAScript ClassFieldDefinitionEvaluation](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-runtime-semantics-classfielddefinitionevaluation), [ECMAScript InitializeInstanceElements](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-initializeinstanceelements), [Rust struct expressions](https://doc.rust-lang.org/reference/expressions/struct-expr.html), [Rust expression evaluation order](https://doc.rust-lang.org/reference/expressions.html#evaluation-order-of-operands). Rust's explicit field expressions are a limited analogy, not this proposal's runtime class model.
