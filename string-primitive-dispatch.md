# String receivers in primitive-block dispatch

A String receiver reaches its `primitive string` definitions before the built-in fallback. For example, `primitive string { operator +(rhs:uint8):uint8 { return 7; } }` gives `"a" + (1 := uint8)` the value 7. A String literal or a literal/refined String carrier uses the String receiver family. The same selected definition determines the result for an annotated String, a String union at runtime, or an untyped value that is actually a String.

The protection for two plain values of the same primitive still applies. A broad `rhs:any` definition cannot replace plain String/String concatenation merely because one value carries a literal type. Strip the literal refinement for that protection; retain actual metadata parameterizations, which are eligible for metadata dispatch. An ordinary boxed String remains an object and follows its existing object/coercion rules. Do not convert an object to String to search for a String block.

Within the selected body, parameterized String receivers and effective operands lose their metadata just as numeric operands do. Nested operations use the built-in/raw path; after the body, the merged declared result metadata is installed on a String result. A bodyless definition likewise stamps the built-in String result. Operand evaluation, most-specific selection, ambiguity, conversion and abrupt restoration keep their existing order.

## Decision matrix

Required properties: accepted declarations are reachable; plain same-family pairs remain protected; carrier representation does not select different semantics; metadata and raw-body rules agree; selected result contracts and effects survive; checking cannot execute an operator or extend reference liveness.

| Direction | Ergonomics/autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Ban String blocks because lookup currently omits them | Removes accepted signatures and surprising-but-expressible mixed operations from completion | Rust Add supports a separate RHS and Output; it does not prescribe JS dispatch | Easy exclusion, no inherent shape/IC advantage | Requires a redesign of the general primitive rule | Reject |
| Accept the block but always concatenate | Completion reports a return that execution ignores; errors move downstream | Neither selected Rust Output nor ordinary operator contracts support ignoring the selected body | Incorrect fast-path and specialization assumptions | Loses values, effects and result contracts | Reject |
| Treat every carried literal record as a different primitive | A literal annotation changes whether an ordinary String pair calls user code | Rust type identity is not a precedent for changing the runtime family of a JS literal refinement | More dispatch identities and unstable specialization keys | Breaks same-plain-pair protection | Reject |
| Erase all String refinements and metadata before selection | Easy completion, but metadata overloads and result contracts vanish | Rust's distinct types do not justify erasing this proposal's metadata | Cheap lookup with invalid assumptions; no object-shape benefit | Loses metadata admission, conversion and composition | Reject |
| Lookup String by primitive representation; erase literal refinement only for plain-pair protection; use the existing selection/raw-body/result rules | The declared operation predicts values and completion; ordinary String pairs retain their meaning | Rust RHS/Output is a limited contract analogy; ECMAScript supplies fallback conversion order | Small primitive dispatch tag plus existing tables/guards; no program-object fields, shape transitions or property IC cases | Preserves selection, metadata, effect order and independent liveness | **Choose independently of effort** |

## Source boundaries and checking

A dynamic Function's synthetic checking root must expose its local operator declarations to the same syntax index as a Script. A registered block from another source can also run, even if the current checking pass cannot reconstruct its captured signature. Missing reconstructed syntax therefore cannot prove that the built-in String operation was selected. Keep the result unknown and the runtime return boundary in that case. This is a conservative implementation boundary, not a permanent language restriction on cross-source checking.

| Direction | Ergonomics/autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Assume no visible local syntax means no dispatch | Completion says String and can reject a valid numeric return | Rust item resolution does not equate absent local syntax with absent definitions | Unsound cached result and check elision | A registered body can return another type | Reject |
| Re-evaluate old source or resolve its captures in the new caller's scope | Suggestions depend on repeated effects or same-spelled shadows | Rust item identity is a limited analogy; JS environments decide captures | Repeated work, effects and potentially shape/IC changes | Violates identity, evaluation order and liveness | Reject |
| Permanently suppress cross-source result checking | Avoids false positives but never delivers available contracts | Discards useful declaration identity | Repeated runtime checking and lost optimization opportunities | Safe unknowns alone do not complete specified checking | Reject as completion |
| Recover declaration/captured-environment signatures by identity; until proved, retain unknown results and runtime checks | Precise where justified, honest uncertainty otherwise | Rust separates item identity from use sites; JS capture/publication rules remain specific | Compiler-owned summaries and dependency guards, no user-object metadata | Preserves the selected operation and prevents false built-in proof or elision | **Choose independently of effort** |

## Retained result-admission audit

Two witnesses remain outside this runtime repair: an unused invalid destination in a later Script does not receive its required static judgment from an earlier block, and a primitive body's obviously incompatible raw return is admitted. The raw-body rule erases metadata requirements, not the obligation that a result belongs to the declared primitive family. The next audit should compare declared, inferred and deferred results, ordinary and generated sources, bodyless composition, and failures before side effects.

| Direction | Ergonomics/autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Trust any raw body value and stamp the declared result | Completion can promise a numeric result for a String value | Rust Output still constrains the returned value | Cheap but unsound representation/layout assumptions | Metadata stamping cannot change a value's primitive category | Reject |
| Enforce all declared metadata inside the raw body | Re-enters casts or rejects the intended raw implementation | Rust has no directly analogous metadata-attachment phase | Redundant conversions and possible recursive work | Contradicts raw-body and post-body composition rules | Reject |
| Separate raw base-result admission from post-body metadata composition, with identity-correct cross-source signatures | Failures identify the actual obligation; completion matches selected output | Rust Output is analogous only to result admission; metadata remains proposal-specific | Reusable descriptors and ordinary guards; no new user properties or IC transitions | Preserves representation, phases, restoration and independent liveness | **Recommend independently of effort** |

Primary sources: [Rust Add](https://doc.rust-lang.org/std/ops/trait.Add.html) and [ECMAScript ApplyStringOrNumericBinaryOperator](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-applystringornumericbinaryoperator). Rust permits RHS and Output to differ from Self; its trait/coherence system is not this proposal's open primitive-block registry. ECMAScript supplies the built-in ToPrimitive/ToString fallback, while this proposal adds dispatch before it.
