# Invariant locations at reference parameters

A reference parameter borrows a location with its established contract. The value currently in that location cannot narrow or widen the contract when the call arrives through `any`.

```javascript
const raw: any = { value: 1 };
const view: { value: number | string } = raw;
function take(ref value: number) {}
const invoke: any = take;
invoke(ref raw.value); // TypeError before take's body
```

The same refusal applies when the data is inherited. `view` established a contract on `raw.value`; it did not turn the supplying prototype into the destination. After refusal, the original location still accepts the values its established contract allows. A known `number` location likewise cannot become a `number | string` location merely because its present value satisfies the union.

## Required judgment

At the existing receiving check, read the referent with its normal effects and liveness checks. Resolve the receiving type in the current generic context and obtain any available declared location contract without another value read. Follow reference aliases and linked import indirections to the declaration that supplies that contract. Inspect only internal binding and backing metadata: lexical/global declarative bindings, typed object properties, Symbol keys, typed Array elements and tuple positions. Type lookup does not redirect the actual borrow, invoke a getter, enumerate a namespace or execute user code.

When the available location contract and receiving contract are both non-`any`, require `SameType`. Subtyping, current-value membership and display-name equality are insufficient. Then perform the existing referent-membership check without conversion. For a `ref`-marked formal, a mismatch throws before initializing its alias. The runtime judgment also applies at the existing receiving checks for reference-typed parameters and reference returns; it is not limited to scalar-rest-backed locations. Neither parameter spelling enters the body after a failed check.

Unknown or explicitly dynamic location types retain their existing runtime treatment. Do not infer a new general invariant contract from `RuntimeTypeOf` of the current value. Existing metadata-specific dynamic checks remain in force; this repair does not redesign every `any`/metadata combination or promise that all reference spellings have identical gradual behavior. A parameter written `n: ref (number | string)` has a reference to the union as its target; `n: ref number | string` instead denotes a union containing a reference type.

## Directions considered

Every valid solution preserves caller storage, location identity, existing permissions and liveness, resolved generic arguments, and the distinction between a location contract and one observed value.

| Direction | Ergonomics / autocomplete / wrong guesses | Prior art including Rust | Real-engine performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Compare available declared location contracts invariantly | The suggested writable parameter type describes the borrowed location. An erased caller's wrong guess fails before the body rather than acquiring a misleading narrower or wider view. | Rust mutable references are invariant in their referent type. Rust has a stronger borrowing/lifetime system and no equivalent gradual `any` boundary, so only the invariant-place analogy applies. | Existing binding/property/element metadata supplies the type; monomorphic checks can specialize to identity guards. No new public properties, shapes or getter IC cases. | Preserves the original location and backing, checks current membership separately and keeps liveness reads. | **Recommend**, independently of implementation effort. |
| Check only the current referent value | A union location containing a number appears to satisfy a number-only writable formal. A later alias write exposes the wrong guess. | Rust does not narrow a mutable place's declared type from its current value. Ordinary JavaScript value passing is not a typed borrow. | Cheap value test but omits required authority information. | Reproduces the defect; cannot establish an invariant location contract. | Reject. |
| Permit subtyping in either direction | Accepts useful-looking views but a writable parameter's possible stores and other aliases' stores no longer agree. Autocomplete overpromises one of those capabilities. | Rust's mutable referent invariance is the closer analogy than covariance of a shared read-only reference. | A subtype walk costs more than a known identity check and does not solve alias writes. | Covariance or contravariance alone cannot preserve both reads and writes. | Reject. |
| Narrow or replace the caller's backing | The call unexpectedly changes which stores old aliases permit. | A Rust borrow does not replace its place's declared type. | Requires metadata changes and alias/IC invalidation. | Violates established backing and may invalidate existing references. | Reject. |
| Create a converted temporary, possibly copy it back later | Writes stop targeting the caller directly; exceptions and later aliases see surprising results. | Neither this proposal's borrow nor a Rust mutable reference is a copy-in/copy-out adapter. | Copies, allocation and writeback bookkeeping; no shape benefit. | Changes identity, conversion policy, failure state and liveness. | Reject. |
| Reject all inherited, imported or dynamically reached locations | Refuses existing useful code for reasons a valid parameter suggestion cannot explain. | Rust has no JavaScript prototype/import-indirection analogue justifying a blanket refusal. | Easy refusal but no semantic advantage. | Discards supported locations instead of checking their actual contracts. | Reject. |

## Incomplete metadata and lookup

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Share a non-observing contract lookup; preserve existing dynamic fallback | String and Symbol keys, arrays, tuples and aliases agree when their contracts are known. Ordinary untyped objects keep their dynamic behavior. | Rust has typed places rather than this missing-metadata case; ECMAScript module imports use internal binding indirections. | Reuse declared records, follow internal aliases with cycle guards, and avoid user property reads. No persistent cache or new retention graph. | The original ReferenceRecord still determines reads, writes and permission. A missing contract is not invented from one value. | **Recommend.** |
| Keep special-case lookups per receiving path | Single/rest parameters or Symbol/string properties can disagree despite equivalent declarations. | Neither prior art calls for key-spelling or internal-binder differences in storage authority. | Duplicate code and inconsistent optimizations; no justified shape benefit. | The previous single-formal lookup missed Symbol/tuple cases and treated array element metadata as the wrong representation. | Reject. |
| Use the current runtime value type as every missing location contract | Untyped locations become accidentally rigid, making an otherwise valid dynamic union formal fail. | Rust statically knows the place type; runtime observation is not a substitute for that fact here. | Avoids metadata traversal but may synthesize structural types and reject valid code. | Confuses a value with the location's admissible future values. | Reject as a new general rule; preserve existing narrower metadata checks. |
| Reflectively query getters, descriptors or namespace exports to discover the type | Type checking can run extra user effects before the call. | ECMAScript provides internal module links; Rust type metadata does not require executing a value getter. | Additional callbacks, observable lookups and IC work. | Alters failure order and still cannot prove backing from a returned value. | Reject. |

## Checking phases and effects

Keep the ordinary receiving read before the new invariant refusal, rather than moving the refusal ahead of that read. The alternative would avoid a read for a known mismatch, but would change getter exceptions and stale-reference precedence. Rust statically prevents many such calls; that is not a precedent for suppressing JavaScript runtime effects. Retaining the phase costs only the already-required read and preserves the proposal's existing failure behavior. Do not defer the check to first body use: even an empty body must not enter with an incompatible contract, and delaying it would allow unrelated body effects first.

Earlier parameter defaults keep their effects; later defaults do not run after a failed `ref`-marked binding. A parameter whose annotation is a reference type keeps its existing annotation-enforcement phase after parameter initialization, so intervening defaults retain their effects before its refusal. Moving all such checks earlier would make the spellings look more alike but change established default/annotation ordering; deferring either to first body use would permit body effects under an invalid contract. Preserve the existing phases while sharing the invariant judgment. This choice has no shape/IC advantage to trade away, and Rust's static rejection does not determine JavaScript default effects.

A check during argument distribution does not replace final binding's liveness check after defaults. An ordinary function's unmapped `arguments` snapshot has its own referent read; an arrow lacks that snapshot. The contract lookup adds no read to either form. A throwing getter, removed array element or relocated storage keeps its ordinary abrupt completion. Type agreement grants no write permission to an immutable imported binding, no lifetime extension, and no backing conversion.

The patch covers the available runtime contracts above. Arbitrary exotic storage, namespace-property provenance, other unknown location representations and a complete redesign of gradual reference compatibility remain separate audits. Reference-rest distribution retains its existing partitioning policy; this increment shares its metadata source rather than changing how all unknown reference candidates are partitioned.

Sources: [Rust variance](https://doc.rust-lang.org/reference/subtyping.html), [ECMAScript module bindings](https://tc39.es/ecma262/#sec-module-environment-records-getbindingvalue-n-s), and the proposal's reference, argument-distribution, metadata and liveness rules. Production cost assessments are design judgments, not benchmarks.
