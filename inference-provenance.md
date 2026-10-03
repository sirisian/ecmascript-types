# Return inference, declaration provenance and captured contracts

The result type of a contribution and its declaration provenance answer different questions. A singleton type may come from an annotation. Conversely, widening a literal to `string` does not create an annotation. Preserve these facts separately, and read captured storage through its declared or inferred binding contract when establishing a published return.

This specifies the binding-read cases of `sec-anchored-contributions` and preserves `sec-published-return-types`: publication affects result checking and typed consumers, while signature identity and overload ranking remain declared-only. It neither infers types for ordinary unannotated `let` bindings nor changes which units are checked.

## Required outcomes

- `let s: string = "s"; function g(){ return s; } const n: number = g();` has an early return-contract error, including in a nested scope or uncalled body.
- A declared singleton, `null` or `undefined` can anchor a contribution. Parentheses do not change that result.
- `function g(){ const s = "s"; return s; }` has no annotation-derived anchor. Widening or naming the initializer alone does not activate publication.
- An untyped shadowing parameter or local cannot borrow the outer binding's annotation.
- A captured `string | number` contributes that contract. A valid later Number store must not violate a published `string` contract fabricated from the declaration-time value.
- A narrowing comparison on a union-typed parameter remains useful. The same comparison immediately after a singleton initialization remains subject to the existing impossible-test rule.

## Available directions

The matrix evaluates all directions against the same criteria. Prior-art comparisons below describe analogies, not claims that another language implements this proposal's participation policy.

| Direction | Ergonomics | Prior art | Production performance | Correctness and reference liveness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Infer anchoring from the resulting type category | An annotation can disappear from autocomplete and errors after narrowing; a literal can acquire a contract after a harmless local extraction. | Rust, TypeScript and C# all distinguish declared information from inferred expression types; none motivates this category shortcut. | Cheap, but its speed buys inconsistent contracts. | Confuses singleton information with provenance and loses typed obligations. It supplies no valid lifetime proof. | Reject: type representation must not determine participation. |
| Publish every known return | Predictable inference in isolation, but ordinary JavaScript functions acquire enforced boundaries; a wrong guess can turn a catchable runtime mismatch into source rejection. | Rust closures and TypeScript infer broadly; C# infers lambda returns. Their compatibility and enforcement models differ. | Requires additional work across legacy JavaScript; no shape or inline-cache benefit justifies the semantic change. | Violates annotation-seeded participation and legacy behavior. Does not resolve captured-storage stability. | Reject: changes the proposal's compatibility contract. |
| Preserve provenance and captured binding contracts | An annotation remains useful through narrowing and local extraction. Tools can show the contract and the actual declaration source. Unannotated literal-only functions stay dynamic. | Rust separates capture semantics from closure type inference; TypeScript separates observed narrowing from declared assignment types; C# captures variables rather than freezing their values. | A compiler can retain an origin flag or dependency edge beside the type and query binding contracts. This needs no program-object property, hidden-class transition or polymorphic inline-cache entry. The proof-of-concept's temporary flow snapshots are not a required engine strategy. | Keeps shadowing, allowed stores, publication and capture lifetimes distinct. No reference becomes live longer and no runtime check is elided on inferred-return evidence. | **Choose.** Specify the semantic evidence independently of compiler representation and traversal order. |
| Specialize the published result to each closure-creation value | Potentially narrower autocomplete, but results depend on creation order and later writes. | Rust move captures are an ownership operation, not a precedent for silently freezing JavaScript reference captures. Neither TypeScript nor C# changes ordinary capture semantics this way. | Requires guards, invalidation or distinct contracts per activation, adding cache and specialization pressure. | Speculative feedback cannot determine mandatory early errors. Rechecking or deoptimizing cannot justify an already published incompatible return boundary. | Reject as a language rule. Ordinary execution optimizations remain possible under the existing semantics. |

The chosen direction also applies to transparent local aliases: carry the initializer's provenance alongside its contribution type. A binding with an explicit `any`, an unresolved annotation or no usable initializer does not borrow an outer annotation. Control dependence alone remains insufficient.

## Signed-literal regression expectation

There are three directions: remove initialization-derived impossibility checks; exempt signed literals; or retain the rule and test subtraction using an input with both alternatives possible. The first two make results less predictable, lose existing correctness guarantees, and add exceptions with no capture, reference-liveness, shape or inline-cache benefit. TypeScript's separation of declared and observed types supports retaining the distinction; Rust and C# do not supply a close analogue for this proposal's JavaScript numeric-literal union rule. Their absence is no reason to invent an exemption.

Choose the third direction. Use a `uint64 | -1` parameter to test equality and inequality narrowing. Separately test that the same comparison after `let r: uint64 | -1 = -1` is rejected with `RT_IMPOSSIBLE_TEST`. This makes both useful behavior and mandatory rejection explicit without weakening either rule.

## Sources

- [Rust book: closures](https://doc.rust-lang.org/book/ch13-01-closures.html) and [Rust Reference: closure captures](https://doc.rust-lang.org/reference/types/closure.html): closure type inference and capture modes are separate concepts; ownership and borrowing make the analogy narrower than JavaScript capture.
- [TypeScript: narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html): observed types change through assignments while declared types govern allowable assignments. TypeScript's erased checks differ from enforced return boundaries here.
- [C# specification: expressions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions): anonymous-function inference and captured outer variables. Ordinary named-method return annotations and C#'s static compilation model limit the analogy.
