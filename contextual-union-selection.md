# Contextual selection of complete object alternatives

An object literal at a union must keep the evidence for one alternative together. For example:

```js
const adapter:
  { tag: "text", fn: (value: string) => string }
  | { tag: "generic", fn: <U: type>(value: U) => U }
  = { tag: "generic", fn: (value: U): U => value };
```

The discriminant and callable describe the generic alternative, regardless of arm or property order. Its effective generic declarations govern the written `U` annotations, body, created value, specialization and reflection. Construction alone is insufficient evidence: a function that passes checking but fails its first generic call has not implemented this rule.

## Selection and checking

Determine compatibility with a complete alternative, including the surviving descriptors, required members and callable contracts. A fresh method with a written default may accept more calls than its destination requires; test callable assignability before treating the writable property's declared contract as invariant. Last writers determine the value stored at a key. Earlier initializers still execute normally and their independent checking obligations remain.

An unresolved signature annotation is not affirmative compatibility. Trial contexts, adopted declarations, deferred annotation obligations and source publications must be restored when a trial loses, including after a nested trial succeeds or an abrupt completion occurs. Closed body declarations retain their required failure phase; they do not turn an otherwise known surrounding signature into an unknown signature. The selected body's defaults, returns and local declarations still need ordinary validation.

A selected alternative must reach evaluation as well as checking. Anonymous property functions use NamedEvaluation and receive the same property context as an expression evaluated ordinarily. Keep effective generic declarations and captured arguments with the created callable. Preserve ordinary evaluation order, function identity, reference liveness and the existing boundary rules; selection does not execute a body, getter, initializer or iterator to obtain evidence.

The union-wide freshness rule remains separate: choosing an alternative does not make a key declared by another union arm an excess key. Nested literals retain their own freshness checks.

## Directions considered

The earlier [complete-context decision](adopted-generic-annotations.md#complete-union-contexts) remains applicable. This refinement makes the scope and remaining ambiguity explicit.

| Direction | Ergonomics and wrong-guess cost | Prior art | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep the first member-local success | Autocomplete can offer a callable inconsistent with the object's tag; changing arm order changes admission. | Neither Rust's explicit variants nor TypeScript contextual typing supports mixing contracts from unrelated alternatives. | Cheap trials save compiler work but can publish unusable contracts. | Missing immediate diagnostics can hide a deferred failure; member correlation is lost. | Reject. |
| Merge the offered callable signatures and binders | Completion invents a contract no arm supplies, and errors move away from the user's guess. | Rust variants retain their own fields. TypeScript supplies context from a destination; it does not establish this proposal's runtime generic declarations. | More union and specialization work; no justified runtime benefit. | Invented declaration correspondences can change capture and admission. | Reject. |
| Require an explicit intermediate annotation for every union literal | Predictable, but repeats information already established by the complete literal. | Explicit variant constructors fit Rust's nominal enums; requiring them here would change structural contextual typing. | Simpler implementation is not a semantic reason to reject a determined literal. | Avoids selection by dropping a contextual obligation. | Reject as the general rule; an explicit annotation remains useful when context is actually ambiguous. |
| Choose a uniquely compatible complete alternative and validate its effective contract | Completion follows the whole object; a wrong body is diagnosed at its normal check. | Rust supports keeping variant-specific contracts coherent; its explicit nominal construction does not specify structural inference. TypeScript supplies contextual callback parameter types, but erases them. | Compiler trials can share immutable records and prune incompatible shapes. Retaining selection as compiler metadata requires no wrapper or extra object properties, so it need not alter shapes or inline caches. | Keeps correlated members and declaration identity; rollback must include obligations and runtime-facing publications. | Recommend. Implementation effort does not change this choice. |

Primary prior art: [Rust enum constructors](https://doc.rust-lang.org/reference/items/enumerations.html), [Rust struct expressions](https://doc.rust-lang.org/reference/expressions/struct-expr.html), and [TypeScript contextual typing](https://www.typescriptlang.org/docs/handbook/type-inference.html#contextual-typing). The conclusions about this proposal are design inferences from those analogies, not claims that either language implements runtime structural generic adoption.

## Numeric preferences between complete alternatives

Existing literal propagation requires `{ x: 1 }` at `{ x: int32 } | { x: uint8 }` to choose `uint8`, including the runtime value reported by reflection. Selecting that type only in the checker leaves an ordinary Number that a later union boundary can convert through the wider alternative first.

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Choose the first complete arm | A harmless reorder can change arithmetic and reflection. | Rust's explicit variant and literal context do not justify a source-order guess here. | Cheap, but makes source order observable without a specified reason. | Violates the existing narrower-fitting-member rule. | Reject. |
| Independently choose each member's narrowest type | Simple completion per field can invent an object no arm admits. | Neither explicit Rust variants nor contextual TypeScript destinations justify this combination. | Local work is cheap; later rejection or conversion wastes the result. | Loses correlation across numeric fields. | Reject. |
| Require explicit numeric annotations in all multi-arm cases | Clear but makes an already specified scalar preference disappear one level down. | Rust permits explicit suffixes; that does not require abandoning this proposal's numeric adaptation. | Low implementation cost. | Contradicts the existing single-field rule. | Reject. |
| Prefer a complete candidate when every differing numeric position prefers it and all other positions agree | Preserves the scalar expectation without making source order or property order a tie-breaker. | Rust supplies only the general analogy that numeric literals consume context; the precise ordering is this proposal's own rule. | Pairwise compiler comparisons suffice; ordinary literal creation needs no object-layout change. | Keeps a whole alternative and the existing numeric ordering together. Conflicting preferences do not establish a winner. | Recommend for this determined case. |

## Bounded implementation and follow-up

The implementation trials complete fresh shapes, rolls back candidate contexts and deferred annotations, and publishes a uniquely compatible or uniformly preferred numeric alternative. It then performs the selected member/body checks. Numeric literal creation preserves the selected representation, and NamedEvaluation preserves the selected callable context.

This is not a general solver for all competing contextual bodies. Overlapping callable alternatives without a discriminant still need a separate audit of union canonicalization, generic admission and the contract retained after crossing the union boundary. A compatible shape is not a completed body judgment. Cases whose full applicability differs only after checking a body, unresolved alternatives, conflicting numeric preferences, and diagnostic/cache reuse remain part of that follow-up. The checking report records required outcomes and observations rather than relabelling a runtime failure as success.

### Follow-up: overlapping callable admission

The probes without a tag retain the desired outcome of a callable usable through its selected generic signature. They currently fail at runtime. Unlike the discriminated case, the created generic function can also satisfy a nongeneric arm, so the ordinary union boundary's first successful admission may specialize it to that arm. This is a boundary-policy question as well as an implementation question; completing source context selection alone does not settle it.

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Let the later boundary choose the first admitting arm independently | A generic context accepted for construction can disappear before the first call; completion and runtime capability diverge. | Rust's explicit variant construction retains the chosen variant; structural admission here is a weaker analogy. | Reuses the boundary but can repeat work and specialization. | Closest to the current SelectUnionMember algorithm, but does not preserve the selected construction contract. | Reject as the desired contextual-literal rule; document the existing algorithm's conflict. |
| Prefer every generic arm globally | Makes context-free crossings surprising when a nongeneric destination was intended. | Neither Rust variants nor TypeScript contextual typing establishes this preference. | Adds selection work to unrelated runtime crossings. | Changes general union admission beyond the construction evidence. | Reject. |
| Require explicit intermediate annotations for every overlap | Predictable but discards an already established construction choice; extra syntax can still encounter the same later boundary. | Rust explicitly chooses variants, but this proposal already supplies structural construction context. | Simple checker path; extra boundary operations may remain. | Does not explain how a later crossing should retain or discard that choice. | Reject as a complete solution. |
| Carry the selected complete alternative into the immediate construction admission | The created callable and completion remain consistent; an unrelated later crossing still follows its own target contract. | Rust's retained variant identity supports the principle, while its nominal representation does not dictate the mechanism. | Retain a compiler/admission fact without wrappers or new object fields; avoid repeated observable admission. | Requires an explicit SelectUnionMember refinement and proof that admission, specialization and reference liveness are preserved. | Recommend for the next repair, independent of the work needed. The current patch does not claim to implement it. |
