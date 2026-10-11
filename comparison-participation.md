# Comparison participation and current flow facts

This reconciles three old test expectations with existing rules; it does not change equality at runtime. The strict-comparison decision was already recorded in `inference-provenance.md` and the specification's `sec-narrowfrom`. Successful initialization/store facts and finite computed-key reads are already specified as well.

Keep four questions separate: the binding's permitted stores, its current value domain, the declaration provenance that makes an operand participate, and whether an expression decides a branch. An annotated binding narrowed to a singleton still participates and does not become a written literal. Its declared contract continues to govern later stores and captured storage.

```js
String((5 := uint8) === (5 := uint16)); // early type error
function equal(a, b) { return a === b; }
equal((5 := uint8), (5 := uint16));    // false: untyped parameter query

let a: uint8 | string = 1;
let b: string = "s";
a === b;                             // early type error: current a is uint8
a = "s";                             // valid under a's declared contract
a === b;                             // admitted

let o: { n: uint8 } = { n: 1 };
let k: string = "n";
let bad: string = o[k];               // early type error: known key selects uint8
```

The strict-comparison obligation applies to stored, returned, discarded and deciding results, including arguments to `String`. It is independent of the deciding-position rule for a pure membership query. `let a: uint8 | string = 1; a is uint8;` remains a permitted value query; using that settled query to decide an `if` is an error. Broad union parameters retain useful comparisons and membership tests.

A computed expression read uses its key's established current domain. A general `string` parameter without a singleton fact still leaves that read dynamic. Changing the key or invoking a function that can change a captured key invalidates the relevant fact. Type-position `T[K]` retains its separate admissibility rules. None of these facts executes a key or getter while checking, changes a store contract, or extends reference liveness.

## Decision: strict comparison consumers

Required properties are consistency with the already selected operator obligation, the literal-name/provenance distinction, gradual operands, independent membership-test rules, pre-body rejection, and unchanged runtime identity and liveness.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Reject disjoint strict comparisons only in deciding positions | Value queries become convenient, but storing an established typed mistake silently succeeds. | JavaScript identity is defined across types; Rust's equality availability is type-directed. Neither determines this proposal's participation policy. | Removes a check without a necessary layout or IC benefit. | Reverses an explicit existing operator obligation; membership-query permission does not imply it. | Reject for this reconciliation; a separate redesign would be required. |
| Exempt wrappers such as `String`, return or discard | Harmless extraction changes diagnostics and what completion promises. | Rust's operator contract is not selected by a result wrapper. | Adds consumer-sensitive compiler cases; no user-object benefit. | The same operand obligation receives conflicting answers; erroneous bodies can run. | Reject. |
| Reject every unequal or disjoint runtime query | Editors overpromise type evidence in ordinary JavaScript; legitimate probes fail. | Rust's static model cannot justify extending gradual checking to all JavaScript values. | More checking and invalid assumptions about unknown values. | Loses untyped/any routes, literal adoption and pure membership semantics. | Reject. |
| Preserve the obligation at every consumer, with its existing participation and literal exceptions | Stable operator diagnostics; explicit dynamic inputs retain runtime queries. Completion can explain the operand contract. | Rust offers a limited type-directed analogy; ECMAScript supplies runtime identity semantics. | Reuses compiler type/provenance records; no object fields, shape transitions or additional member IC cases. | Preserves current commitments, early phases and separate liveness checks. | **Choose independently of effort.** Reconcile the old wrapper expectation and retain a separate runtime-identity test. |

## Decision: declared contracts versus current facts

Required properties are source-position flow, declaration-based participation, valid later stores, effect invalidation, conservative unknown inputs, and agreement between exact-key and equivalent computed reads.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Ignore initialization facts and use declared types everywhere | Broad displayed contracts hide immediately known errors and known members. | Rust's declared types are useful contracts, but its model does not prescribe this proposal's union-flow analysis. | Discards reusable compiler facts; more dynamic checks, no shape benefit. | Reverses successful-store flow and finite-key commitments. | Reject. |
| Replace the binding's contract with its initializer type permanently | Later valid stores fail and completion misstates storage permissions. | Rust's binding type is not a license to substitute a transient fact for this proposal's declared union. | Conflates reusable storage metadata and transient facts; invalid cache assumptions. | Breaks captures, stores and effect invalidation; supplies no lifetime proof. | Reject. |
| Recover singleton values by executing keys or inspecting live bindings during checking | Diagnostics depend on evaluation and can trigger effects. | Neither Rust's type-directed comparison nor ECMAScript runtime lookup supplies that permission. | Runtime reads and getters defeat compiler isolation and cache validity. | Violates checking phases and declaration identity. | Reject. |
| Use established current facts for reads and retain declared contracts for stores/captures | Completion can show both the current member and the permitted storage type; wrong destinations fail early. | The distinction is this proposal's own flow policy; Rust is a limited contract analogy. | Compiler-owned flow/dependency records, conservative invalidation; no user-shape or IC changes. | Preserves gradual unknowns, evaluation order, allowed stores and independent liveness. | **Choose independently of effort.** Test initialized and genuinely broad inputs separately. |

Primary comparisons: [Rust PartialEq](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html) and [ECMAScript IsStrictlyEqual](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-isstrictlyequal). Rust permits different comparison operand types through the trait's right-hand type parameter; this is not the proposal's numeric literal adoption, gradual participation or flow policy.

## Evidence and limits

The focused corpus separates value/deciding positions, both strict operators, numeric families, annotated mutable/immutable bindings, broad and unknown inputs, stores, captured writes, lexical shadows, exact and general keys, and failure recovery. Function construction, importing Modules and feature-off controls supplement Scripts. Existing implementation behavior already satisfies these witnesses; changing expectations records a conformance reconciliation, not a runtime repair.

Object.is has a separate intrinsic-origin and argument judgment. This audit does not certify its currently unresolved consumer reach, constant publication across Scripts, primitive-block dispatch, general recursive inference, or the full suite.
