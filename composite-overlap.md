# Composite overlap and checking

`Composite` denotes every composite. A value created as `Composite.<{x:uint8}>({x:1})` can therefore be identical to one returned by `Composite({x:uint8(1)})`. The latter call's broad static type does not describe a different kind of value.

The implementation represents both types using primitive Type Records. That representation must not feed the scalar rule that distinct primitive types have no common values. Composite values are frozen Objects with overlapping structural contracts. Their type objects remain distinct, and overlap does not imply equality of two particular values.

## Shared judgment

Handle Composite before the scalar cases in AreDisjoint. Use that relation consistently for strict comparison, narrowing and intersection checking/canonicalization; do not special-case only an equality expression. Top Composite overlaps each shaped Composite. A structural object, tuple or interface view may also describe the same value. A Composite is neither callable nor a scalar/enum value, and the proposal's mutable array contracts exclude it.

For two known shapes, these finite negative proofs apply:

- Record and tuple kinds cannot be one composite; an array shape describes the variable-length tuple kind.
- Record membership allows extra keys. Different declared key sets are not closed construction schemas. A shared property with disjoint types proves failure if either shape requires it. Two optional properties can both be absent.
- For tuples without rests, disjoint admitted length intervals prove failure. Otherwise only positions present at every common admitted length can prove a member conflict. Declared defaults establish possible omission without being evaluated.
- A tuple with a required position can conflict with the element contract of an array shape. Two array shapes can share the empty tuple, even when their element types differ.
- Nested Composite members use the same judgment. A repeated active pair supplies no proof of disjointness; independently established conflicts are still visited. Unresolved shapes, rest partitions, index-signature interactions and unknown leaf types remain conservative.

These rules inspect type descriptors only. They do not read properties, run getters, initialize defaults, convert members, merge type objects, inspect interned values, or grant reference permissions. Intervening effects still require the existing reference-liveness checks. Runtime equality and interning are unchanged.

## Direction matrix

All directions are evaluated against the same criteria. The recommendation is independent of implementation effort.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance, shapes and ICs | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat distinct Composite Type Records as disjoint scalar types | A broad annotation prevents comparisons with its own subtype; displayed member types are misleading | Rust PartialEq may relate different types; its trait-based equality is not this proposal's identity test | Cheap but based on a representation accident; no useful IC advantage | Contradicts family-top containment and structural membership | Reject |
| Suppress only the equality diagnostic or erase types in tests | One spelling works while narrowing or an intersection still fails; autocomplete loses useful contracts | Neither Rust's typed comparison contracts nor JS identity supports losing types to repair a relation | Scattered checks, no shape advantage | Leaves inconsistent consumers and wrong empty-type conclusions | Reject |
| Declare every Composite pair overlapping | Avoids false rejections but misses clear mistakes in required fields and kinds | Rust preserves comparison contracts; upstream Composite intern keys distinguish stored contents | Simple but loses optimization/diagnostic facts; no shape benefit over descriptor checks | Ignores established contradictions | Reject as the complete rule |
| Reject every pair without a subtype relation | Two different open shapes can share a wider record; wrong guesses reject legitimate code | Rust's restricted subtyping does not model JS structural overlap | May run two expensive subtype checks, still learns the wrong fact | Neither type need contain the other for a common value to exist | Reject |
| Compare evaluated values or construct candidate composites | Getters/defaults run during checking; failures change program effects | Rust comparison code runs at execution, not as implicit checking; JS creation is observable | Allocations, interning and getter/IC effects during checking | Violates phase separation and can invalidate references | Reject |
| Family-top rule plus sound kind/member/length proofs, unresolved cases deferred | Broad and narrow views compose; known member contracts stay useful in completion and diagnostics | Rust PartialEq is a limited example of cross-type comparison; this proposal's structural membership and JS interned identity determine the actual rule | Compiler-owned descriptors and per-query recursion guards; no user-object shape transitions or extra property ICs | Consistent shared relation, retained conflicts, no conversion or new borrow rights | Choose |

## Recursive and unresolved inputs

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat a recursion revisit or trial-depth limit as disjoint | Recursive contracts fail unpredictably; a wrong guess rejects valid code | Rust's explicit trait obligations are not proof that an arbitrary recursive structural type is empty | Fast cutoff but depth-dependent results; no shape benefit | Invents a negative proof | Reject |
| Evaluate or enumerate values until a witness is found | Unbounded delay, hidden effects and poor diagnostics | Neither Rust type checking nor JS identity requires enumerating runtime values | Unbounded work and possible allocation/interning | Nontermination and phase/liveness violations | Reject |
| Revisited pair contributes no negative proof; check other established facts | Predictable conservative checking; remaining contracts still constrain completion | Like conservative compiler analyses, but Rust's nominal/lifetime subtyping is not imported | Finite descriptor traversal on the current graph; production implementations can memoize dependency-aware results | Sound necessary proofs; false means no disjointness proof, not an inhabitation guarantee | Choose |

The proof-of-concept uses an invocation-local active-pair list and no arbitrary depth or count cutoff. This is not closure of the general portable evaluation-budget or recursive-type formation gates. A production implementation should share completed relation work without treating a cycle-dependent incomplete result as a final global cache entry.

## Verification and remaining boundaries

A finite independent value model checks 2,025 ordered pairs across 45 top/record/tuple/array Composite contracts, including absent optional members and empty tuples. Script, generated Function and importing Module tests exercise positive and negative checking. Recursive record graphs and unresolved rests also have direct relation controls.

Formation and consumer reach are distinct from the relation itself. Existing probes still show trouble constructing an empty array-shaped Composite, assigning a Composite over a named interface to that interface, and forming a Composite tuple with a rest whose collected shape is transformed into a Composite. An Object.is probe also remains outside the established diagnostic path. These witnesses stay recorded in the handoff; successful overlap tests do not close them.

Primary references: [Rust PartialEq](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html), [Rust subtyping and variance](https://doc.rust-lang.org/reference/subtyping.html), and [TC39 Composites](https://github.com/tc39/proposal-composites). Rust's cross-type comparison is explicit trait dispatch, and its subtype system is not structural width subtyping. Neither is a direct substitute for this proposal's Composite contracts.
