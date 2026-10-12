# Metadata declaration and computed-result admission

This audit separates three formerly failing tests from the judgments they try to exercise. The specification and implementation are both subject to review; an old assertion is evidence of intent, not authority.

## Required boundaries

A metadata declaration claims its shape's property keys once, before it has a particular governing base. A parameter such as `T: type extends Ordered.<T>` is a binder at that point. Treating its placeholder as an ordinary alias argument attempts to read or validate a specialization that has not happened.

A selected operator result such as `float32.<mul(X, Y)>` becomes closed when the operand contracts determine `X` and `Y`. Its builder then supplies an ordinary result contract before source entry. The same requirement applies when that result closes a containing expression or a later local binding. Component counts and metadata arguments belong to the declaration that captures them. An unselected value body's builder is not evaluated.

An unused function containing a known bad string literal must fail statically when its target pattern is available. The old literal fixture obtains the standard helpers from an ordinary IIFE and destructuring, so it does not establish that premise. Its corrected check uses the exact `suffixed` and `prefixed` definitions from the standard source as hoisted declarations and asserts StaticTypeError. Existing runtime-kit crossing tests remain. The correction does not claim to implement general effect-dependent builder availability.

## Direction 1: generic declaration versus application

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Apply a generic shape to invented concrete arguments | A valid declaration fails before a user supplies a type; choosing a dummy argument leaks into member descriptions | Rust permits a generic declaration under its bounds; it does not require a representative concrete argument | Repeated invented specializations and checks do unnecessary work; no useful object-shape advantage | Confuses binders with arguments and dependent bounds with closed judgments | Reject |
| Ignore bounds for every alias application | Declarations appear to work, but invalid uses get misleading field completions and fail later | Rust's generic admission does not support erasing obligations | Removes useful specialization assumptions; may force runtime guards | Unsound for actual arguments; would alter ordinary generic contracts | Reject |
| Resolve the declaration's shape with identity-bearing binders; admit real applications separately | Keys are available consistently and concrete uses retain their declared constraints | Rust generic scope and bounded definitions are a close analogy for binders, not for this metadata protocol | Compiler-owned declaration descriptors; no user property or hidden-class changes; specialize actual applications normally | Preserves declaring captures and ordinary admission without inventing values or reference liveness | Recommend, independent of implementation effort |

The implementation adds a declaration-shape resolver using the alias's captured environment and abstract parameter frame. It retains metering and leaves ordinary InstantiateGenericAlias admission in place. Optional parameter-valued fields and captured outer parameters are covered; arbitrary dependent field builders and every specialization-family shape are not certified.

## Direction 2: computed operator results

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
| Leave every computed result for runtime | Nested operations lose useful result completions; an obvious wrong dimension runs effects before failing | Rust's required compile-time computations are not silently postponed; the analogy is limited by this proposal's runtime types | Cheap checker but extra runtime checks and weaker optimized contracts | Violates the pre-body rule when captures and builder are available | Reject |
| Evaluate every candidate or execute ordinary producers to discover inputs | A losing overload or unrelated initializer can reject code; wrong guesses have effect-order costs | Rust const contexts do not authorize executing arbitrary runtime code | Unneeded evaluation and cache pollution; observable effects obstruct reuse | Breaks selection order, availability and source effects; cannot prove reference liveness | Reject |
| Resolve only required closed result queries and follow their dependencies | Exact result contracts reach assignments, calls, returns and chains; bad guesses fail at the source boundary | Rust const evaluation illustrates required compile-time work; JS metadata composition remains proposal-specific | Interned result descriptors and identity/argument-keyed compiler caches, with no enumerable metadata or shape/IC mutation; dependency work can be scheduled incrementally | Preserves lexical identity, selected-body rules and the common evaluation budget; no value body is run to obtain a type | Recommend, independent of implementation effort |

The former implementation detached only the annotation's outer node to request evaluation. Its children still identified the original generic binders, so the closed application was dropped as open. Explicit queries now retain the real declaration node. Local and previously registered definitions share site/definition/capture keying, and selected value bodies enqueue their result only after selection. The checking pass rechecks after relevant declaration preparation and drains newly closed dependencies before crossings. A completed query is not repeatedly evaluated in the same pass. This is not a complete portable accounting or cache invalidation proof.

## Direction 3: the literal fixture

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
| Execute the kit's IIFE during checking to preserve its old assertion | Conceals the availability premise and makes arbitrary source effects surprising | Rust compile-time eligibility does not permit arbitrary runtime initialization | Requires speculative execution or rollback; no useful shape/IC benefit | Changes source order and cannot generally undo effects or make references live | Reject |
| Weaken literal admission or mark the test expected-failure | Bad literal assignments receive misleading feedback; wrong guesses move to runtime | No supporting Rust analogy for dropping a known obligation | Avoids checking at the cost of reliable contracts | Fails the already-required closed case | Reject |
| Give the fixture available standard declarations and verify phase; retain the unavailable case separately | The test measures its stated rule and still checks the actual standard helper code | Explicit compile-time availability is a useful limited Rust analogy | No runtime or object-shape changes; only the fixture setup changes | Keeps the known-literal rule without executing ordinary producers | Recommend |

## Retained questions and evidence

A generic shape's conflicting key is now claimed, but one independently probed second declaration still reports a runtime TypeError after body entry. This is not accepted as conforming to the early-error rule. The exact baseline/current observations are retained as boundary evidence, outside passing conformance counts.

| Direction | Ergonomics | Prior art, including Rust | Performance | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Accept runtime conflict timing | Lets unrelated effects run before an already established declaration error | Rust rejects established declaration conflicts before runtime, though the namespace differs | Avoids an early check but supplies no shape advantage | Contradicts the second-declaration early-error requirement | Reject |
| Throw every failed preparation attempt immediately | Gives early feedback, but unavailable input failures may be mistaken for closed invalidity | Rust's known const contexts are narrower than these JS availability cases | Simple but risks repeated preparation or disabled caching | Conflates unavailable inputs with failed closed judgments and may change precedence | Reject as a blanket repair |
| Classify availability, then publish valid claims and report closed conflicts at the required boundary | Predictable errors and member metadata; wrong guesses do not run the source | Rust declaration checking supports the boundary, not this protocol's details | Validate compiler-owned descriptors before publication; no source-object shape changes | Must preserve ordering, failed-state restoration and liveness separately | Recommend next audit |

The IIFE helper boundary also remains open: possible directions are speculative execution (rejected above), arbitrary permanent prohibition (rejected because an earlier evaluated source can supply the helper), or dependency/availability-based checking at the specified boundary (recommended). Judge inputs by lexical identity, not a same-spelled helper or the fact that a test string happens to place an initializer earlier.

Source: Rust Reference [generic parameters](https://doc.rust-lang.org/reference/items/generics.html), [bounds](https://doc.rust-lang.org/reference/trait-bounds.html), and [constant evaluation](https://doc.rust-lang.org/reference/const_eval.html), consulted 2026-10-12. These are analogies, not evidence that Rust implements runtime metadata or this proposal's reference rules.
