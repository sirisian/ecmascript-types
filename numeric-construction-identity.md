# Numeric metadata construction identity

An explicit construction `T(value)` or `value := T`, where `T` is a numeric metadata parameterization, produces a value carrying the admitted parameterization. A successful predicate check alone does not establish that provenance. This applies to decimal, rational, binary floating point, integer, BigInt, Number and complex payloads. It is the same construction boundary a brand already has.

For a source not already carrying the target, convert to the target's base and run its applicable defined validation judgments once. An absent validation hook permits construction; a rejecting or throwing hook prevents publication. Carry the target on the result only after admission. An explicitly written default metadata portion still names a distinct construction target. Ordinary membership and ordinary value admission retain their existing rules, including admission at a default requirement without acquiring a carrier. The proof of concept also corrects its BigInt default-admission shortcut, which had incorrectly reused a construction carrier.

If the value already carries the identical parameterization, construction is idempotent. If it carries a different parameterization of the same base, use the existing conversion protocol before any membership shortcut: subtype permission, scaling, quantization, unpinned-portion transport and final validation. This may carry additional transported metadata, as that protocol already specifies. Successful `validate` cannot substitute for permission to cross between two existing parameterizations. The same protocol already required at ordinary annotations, arguments and returns applies to every scalar numeric carrier, including a source passed through `any` or its bare base annotation.

Explicit conversion back to the immediate base erases the outer parameterization on the result. An implicit base annotation continues to preserve carried metadata. Both explicit spellings agree, including the one-argument rational constructor. Nested numeric brands use their immediate base at each construction boundary; layered intersections retain their full admitted record.

## Requirements and available directions

A valid solution must give explicit construction its promised runtime identity, preserve the source and exact payload, retain the metadata conversion protocol, distinguish predicate testing from construction, avoid repeated admission, and preserve evaluation order and reference liveness.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Leave a successful construction as a bare numeric value | A call typed as `T` suggests `T` operations, but reflection disagrees and a later range operation fails far from construction. | Rust newtype construction establishes the wrapper type; it does not return its bare field. | Avoids a carrier but causes repeated checks and invalidates downstream domain assumptions. | Violates construction identity; membership cannot repair provenance. | **Reject.** This is the observed defect. |
| Stamp every successful `is` test or implicit admission | A seemingly observational test changes what later calls accept; wrong guesses affect aliases. | Rust pattern/type checks do not construct a newtype. Python Decimal construction is distinct from predicate testing. Neither models gradual metadata admission exactly. | Adds writes to query paths and potential shape transitions or invalidated IC assumptions. | Violates the existing default-admission and reference-location rules. | **Reject.** |
| Mutate the numeric source's carrier at explicit construction | The result looks correct, but another binding to the source changes type without a conversion. | Rust value wrappers and immutable Python Decimal values do not retag an existing numeric alias. | Cheap locally, but shared boxed values require alias tracking or deoptimization; hidden shape/IC churn is avoidable. | Breaks value semantics and failed-construction isolation. | **Reject.** Internal object representation does not make a numeric value a mutable reference object. |
| Establish identity on the admitted numeric result, preserving the source | Both spellings match their static result, reflection and subsequent range operations. Errors occur at construction. | Rust's newtype is the closest provenance analogy; Python Decimal illustrates immutable payload and significance preservation. | Existing numeric layouts and canonical type descriptors suffice; a real engine can use tagged or unboxed values and eliminate copies when unobservable. No public properties or shape changes are needed. | Preserves exact payload, source identity, hook order and liveness; membership remains observational. | **Recommend.** Chosen for its semantics, independent of implementation effort. |

The same question arises when a source already carries metadata:

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Reconstruct from a stripped base whenever `T` is called | A unit-changing call silently keeps the old magnitude; autocomplete cannot reveal that error. | Rust unit wrappers require an explicit conversion implementation; stripping fields is not that conversion. | Cheap, but loses required work rather than optimizing it. | Bypasses subtype permission, factors and quantization. | **Reject.** |
| Re-run construction and hooks even for identical types | Repeated calls can unexpectedly quantize again. | Rust identity conversion does not construct a different numeric quantity. | Repeated hook evaluation and allocations with no new proof. | Conflicts with same-type conversion identity. | **Reject.** |
| Identity for identical carried types; existing protocol for different same-base parameterizations | One predictable conversion rule across numeric families and both spellings. | Comparable to explicit typed conversions in Rust; this proposal's metadata hooks and runtime reflection have no direct Rust equivalent. | Canonical type comparisons and existing conversion plans; no new persistent cache or layout. | Preserves permission, exact scaling, validation and full metadata transport. | **Recommend.** |

## Nested metadata depth

A type's ultimate numeric base is a structural fact. A fixed helper cutoff must not make a deeply nested numeric brand take the mutable-object construction path.

| Direction | Ergonomics | Prior art | Performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Keep or raise a silent depth cutoff | Reflection and alias behavior change at an invisible nesting threshold; autocomplete cannot explain it. | Rust nested newtypes do not silently become mutable reference values at a helper threshold. | Bounded walk, but incorrect carrier selection and potential alias/IC invalidation. | Violates source preservation; a larger number only moves the failure. | **Reject.** |
| Introduce a source-level nesting rejection at that cutoff | Predictable only after learning a new arbitrary restriction. | Rust implementation recursion limits are explicit resource limits, not permission to retag a value. | Bounds the query but unnecessarily rejects finite types. | Would require a new portable limit and diagnostic; does not implement existing construction semantics. | **Reject.** |
| Follow finite base chains with cycle detection | Uniform behavior at every admitted finite nesting depth. | Consistent with nested Rust wrappers; this proposal's type graph and resource accounting remain its own. | Linear in visited descriptors, with no persistent cache or public shape change; plain numeric conversions bypass this parameterization query. | Preserves carrier selection and terminates a cyclic query without claiming a numeric base. Existing type-evaluation limits remain in force. | **Recommend.** |

## Payloads, failures and source effects

An explicit return to `number` produces the ordinary Number representation, so equality with an ordinary Number observes the preserved value and zero sign. Attaching or removing the outer metadata changes no numeric representation: decimal width, coefficient, exponent and special value; binary zero sign and special value; exact wide integer; canonical rational numerator/denominator; complex components. A conversion to a different numeric base still performs that base conversion's specified rounding or loss before admission. Literal operands retain their existing exact contextual evaluation, including decimal significance. Construction does not parse a String.

A failed base conversion runs no target validation. A failed validation publishes no target value and leaves the source unchanged. Source expressions execute once. Existing effects before failure remain; a failed initializer is not initialized, and a following Script can recover without acquiring the failed target's identity. Constructing from a value does not convert a reference location, authorize a stale reference read, or bypass a temporal dead zone.

This repair is independent of implicit-boundary coverage, default-value construction and the broader inference gates. It does not assert that those audits are finished. The range reproducer now constructs decimal endpoints directly without a declared identity cast; arithmetic's full endpoint-domain agreement remains enforced.

## Prior-art sources and limits

- [Rust Book: tuple structs](https://doc.rust-lang.org/book/ch05-01-defining-structs.html) and [advanced types](https://doc.rust-lang.org/book/ch20-03-advanced-types.html): tuple structs establish distinct types, and newtypes can describe units. Rust has no corresponding runtime metadata validation or JavaScript `any` boundary, so this supports construction provenance rather than prescribing the admission algorithm.
- [Python Decimal documentation](https://docs.python.org/3/library/decimal.html): constructed decimals are immutable and retain significant trailing zeros. Python does not supply this proposal's parameterized numeric type identity; the analogy concerns payload preservation only.

- [Rust Reference: limits](https://doc.rust-lang.org/reference/attributes/limits.html): explicit compile-time recursion/resource controls; these do not prescribe this proposal's portable limits or alter value identity.
