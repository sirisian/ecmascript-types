# Metadata root and written-value admission

A metadata record has a record at its root. A nested list remains a valid metadata value; an Array supplied as the root does not become a record because it is empty or has named properties. The same distinction applies to default records, computed records, reflected descriptions and hook portions. A published base-form meta declaration waives unclaimed-key checking, not the record domain or the closed value language.

Determine the root kind before taking entry snapshots. Do not read Array indices, getters or prototype fields to try to turn a rejected root into a record. An ordinary record whose prototype happens to be an Array remains an ordinary record: internal kind, not prototype resemblance, determines the distinction. Existing unsupported-object admission and nested list traversal are unchanged.

Written metadata projection must retain complete, pending and invalid outcomes. An invalid nested leaf invalidates the supplied record; dropping a field changes its value and can collapse different types into one. A known invalid field is still invalid beside an unresolved generic field. Pending values retain their declaration and application dependencies and never publish a partial record as complete. An optional shape member permits absence, not a supplied undefined. An any constraint does not widen the closed metadata language.

A meta-type family argument denotes the registered constraint shape itself. A record is not that family merely because it contains a key the family owns. Gradual mutual assignability through any is not structural identity. Keep ordinary generic type arguments separate from metadata value projection; a tuple type supplied to `Box<T:type>` remains a type, and a named meta family remains a family.

## Root admission directions

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Accept empty lists or enumerate their named properties as a record | Adding an element can change whether the same expression is accepted; Array completions misdescribe record input | Rust arrays are positional values; ECMAScript distinguishes Array internal kind from ordinary property storage | Cheap projection creates inconsistent canonical/cache keys; no valid shape benefit | Violates the required outer record and origin independence; gives no liveness proof | Reject |
| Reject lists everywhere or require literal records only | Removes useful nested values, builders and reflection; completion offers supported forms that cannot be used | Rust const evaluation admits aggregate construction, not only literal leaves; it has no equivalent runtime metadata protocol | Easy restriction, but no necessary IC improvement and prevents reusable computations | Changes the established value language to avoid checking it | Reject |
| Validate the outer record role and admit nested values by their own rules | Wrong roots fail at their source; nested list indexing and reflection remain predictable | Rust's array/aggregate distinction is a limited role analogy; ECMAScript internal kinds supply the direct object distinction | One root kind check before traversal, existing indexed/record access paths; no user-object fields or shape transitions | Preserves all origins, effect order, root requirements and independent reference permissions/liveness | Recommend independently of implementation effort |

## Written projection directions

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Omit unsupported or unresolved leaves | Completion promises a written field that vanishes; a typo can silently become a different interned type | Rust required constant computations report invalid values; they do not silently remove aggregate members | Fewer immediate checks but unsound cache identity and downstream assumptions | Collapses invalid, pending and absent; violates the closed language | Reject |
| Reject every dependent record | Useful generic builders fail even when specialization supplies valid values | Rust const parameters motivate retaining dependencies; its evaluation rules are not this proposal's runtime rules | Simple rejection removes specialization opportunities without improving shapes | Unavailability is not invalidity; contradicts valid dependent construction | Reject |
| Preserve complete/pending/invalid status and fail closed invalid leaves at the required boundary | Errors name invalid supplied values; valid dependent members keep their expected completion | Rust separates required const evaluation from runtime contexts; an architectural analogy, not an imported syntax or ownership rule | Compiler-owned projection results and dependency records; cache only completed values under determining inputs, no user-object IC churn | Retains members, precedence, source context and runtime admission without changing liveness | Recommend independently of effort |

## Family discrimination directions

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat any nonliteral owned-key record as its meta family, or use gradual compatibility | A nested value or invalid field can silently become a family; completion cannot explain lost data | Rust distinguishes naming a generic item from supplying a value; it has no gradual-any equivalence | Saves a structural identity check at the cost of wrong specialization/cache identities | Confuses membership/compatibility with the denoted constraint shape | Reject |
| Ban named families to make every object a value | Existing family conversions and aliases stop working | Rust's type/value distinction provides no justification for deleting this proposal's family form | Simplifies one branch, no inherent runtime shape advantage | Removes an existing language commitment | Reject |
| Identify the registered shape structurally and otherwise project supplied values | Named families work; nested records retain their fields and invalid values fail | Rust's distinction is limited prior art; this proposal's structural identity governs aliases | Reuse canonical/compiler-owned shape identity and existing owner tables; no object-shape mutation | Preserves family semantics, ordinary generic type arguments and independent reference rules | Recommend regardless of implementation effort |

## Verification boundary

Cover root lists/atoms/unsupported objects, base-form exemptions, rejection before indexed getter reads, ordinary records with unusual prototypes, nested values, aliases, nominal and chained bases, Symbols, defaults, computed results, named families, ordinary generic type arguments, dependent valid values and closed invalid applications. Compare native errors, diagnostic phase, body entry and actual reflected content. This increment does not claim full recursive projection/resource accounting, general effect-dependent availability, every family activation or the full extension suite.

Primary sources: [Rust constant evaluation](https://doc.rust-lang.org/reference/const_eval.html), [Rust array types](https://doc.rust-lang.org/reference/types/array.html), [ECMAScript IsArray](https://tc39.es/ecma262/2025/multipage/abstract-operations.html#sec-isarray).
