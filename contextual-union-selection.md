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

This is not a general solver for all competing contextual bodies. Immediate admission now preserves the selected complete alternative for the covered construction paths, including overlapping callable alternatives without a discriminant. Receiving-position coverage remains an explicit audit. A compatible shape is not a completed body judgment. Cases whose full applicability differs only after checking a body, unresolved alternatives, conflicting numeric preferences, and diagnostic/cache reuse remain part of that follow-up. The checking report records required outcomes and observations rather than relabelling a runtime failure as success.

### Overlapping callable admission

The probes without a tag require a callable usable through its selected generic signature. The initial six function, arrow and method probes now pass in either arm order. Unlike the discriminated case, the created generic function can also satisfy a nongeneric arm, so the ordinary union boundary's first successful admission may specialize it to that arm. This is a boundary-policy question as well as an implementation question; completing source context selection alone does not settle it.

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Let the later boundary choose the first admitting arm independently | A generic context accepted for construction can disappear before the first call; completion and runtime capability diverge. | Rust's explicit variant construction retains the chosen variant; structural admission here is a weaker analogy. | Reuses the boundary but can repeat work and specialization. | Closest to the current SelectUnionMember algorithm, but does not preserve the selected construction contract. | Reject as the desired contextual-literal rule; document the existing algorithm's conflict. |
| Prefer every generic arm globally | Makes context-free crossings surprising when a nongeneric destination was intended. | Neither Rust variants nor TypeScript contextual typing establishes this preference. | Adds selection work to unrelated runtime crossings. | Changes general union admission beyond the construction evidence. | Reject. |
| Require explicit intermediate annotations for every overlap | Predictable but discards an already established construction choice; extra syntax can still encounter the same later boundary. | Rust explicitly chooses variants, but this proposal already supplies structural construction context. | Simple checker path; extra boundary operations may remain. | Does not explain how a later crossing should retain or discard that choice. | Reject as a complete solution. |
| Carry the selected complete alternative into the immediate construction admission | The created callable and completion remain consistent; an unrelated later crossing still follows its own target contract. | Rust's retained variant identity supports the principle, while its nominal representation does not dictate the mechanism. | Retain a compiler/admission fact without wrappers or new object fields; avoid repeated observable admission. | Requires an explicit SelectUnionMember refinement and proof that admission, specialization and reference liveness are preserved. | Recommend, independent of the work needed. The implementation now carries this choice through the covered immediate receiving paths. |

## How long the construction choice lasts

The choice belongs to one evaluation of a source literal and its immediate receiving operation. It requests admission against the selected complete member; it is not proof that admission succeeds. Check at the original boundary, after the same earlier evaluation effects. On failure, propagate that selected member's failure rather than trying a different union arm. Keep the ordinary failure effects and reference-liveness checks.

Carry the source result through value-preserving expression/completion operations. A fresh enclosing object or array can carry a child's choice while that child's producing data-property contribution still supplies the slot. Replacement or deletion ends that association, even when a replacement stores the identical value. For deeper nesting, every intervening contribution must still be present when the receiving check uses the choice. A literal's own selected contract still needs checking after decorators have run.

The selected member governs admission and required callable materialization. It does not narrow the declared union contract of the enclosing storage slot. Successful admission retains its ordinary evidence so acquisition and specialization do not reread admitted properties. Failed admission follows the same checking/conversion sequence as an explicit selected destination; this rule does not erase earlier effects or promise one read across separate failed attempts.

An ordinary binding/property read, an unrelated later boundary, and an independent call made by a getter do not inherit a pending construction choice. Read-only membership queries do not consume it or acquire storage. Completion of the immediate boundary ends the pending choice; an admitted contract may survive under the separate result-dependency rules. No public tag, function wrapper, changed identity or extended borrow lifetime is introduced.

### Transport directions considered

| Direction | Ergonomics / autocomplete / wrong-guess cost | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Attach the choice permanently to the value | A later explicit destination unexpectedly inherits old completion assumptions. Escaped values can retain a choice from a failed initializer. | Rust enum identity is part of a nominal value; that is not this structural union's representation. TypeScript supplies no runtime tag. | A public tag can alter shapes/ICs; a hidden table adds lifetime and lookup costs. | Confuses source provenance with value identity and cannot invalidate a replaced contribution. | Reject. |
| Put the choice on a dynamically scoped admission stack | Source completion looks right until a getter enters another boundary. Errors depend on unrelated nesting. | Neither Rust constructors nor TypeScript contextual typing specifies ambient runtime preference. | Cheap push/pop, but reentrant operations need isolation anyway. | A getter can apply the outer choice to the same value at an independent crossing. | Reject. |
| Admit immediately at literal creation | Later arguments/initializers observe earlier getter or failure effects than the ordinary boundary allows. | Rust explicit construction does not justify moving JavaScript accessor effects; erased TypeScript cannot settle this. | May duplicate checks and materialization. | Moves failure precedence and liveness-sensitive effects. | Reject. |
| Retain the selected member as the enclosing slot's declared type | Completion would falsely forbid a later value admitted by another declared union arm. | Rust stores an enum in an enum-typed slot; choosing a variant does not redefine the slot's type. | Narrows storage cheaply but changes legal stores. | Conflates selected admission with the original backing contract. | Reject. |
| Carry an operation-local source result and live aggregate contribution paths to the existing boundary | Completion and first use agree; errors remain at the receiving boundary. Later reads use their own contracts. | Rust supports coherence of an explicitly selected constructor; the source transport mechanism here is this proposal's design. TypeScript contextual typing is an analogy for source context only. | A compiler can pass the chosen target directly and guard changed contributions; no wrappers or new object properties are required. The interpreter's private metadata is not a proposed object layout. | Checks the selected member, retains original storage types, isolates reentrant calls, and preserves ordinary effects and reference liveness. | Recommend. More implementation work does not weaken the choice. |

The primary sources linked above were rechecked for this refinement. The transport recommendation is a design inference, not a claim that Rust or TypeScript implements these runtime structural rules.

### Coverage still required

The current increment covers lexical and `var` initialization, transparent expression results, selected `do` completion paths, written returns, and fresh object/array nesting. It checks generic calls, reflection, local captures, replacement/deep replacement, abrupt escape, reentrant admission and live-borrow failures. It does not close every receiving position. The audit retains required-success cases for tuple element context, call-argument context/admission, parameter defaults and assignment writes. Complete body/default applicability, unresolved alternatives, conflicting numeric preferences, rollback and cached diagnostic/flow reuse remain the broader contextual-selection gate.
