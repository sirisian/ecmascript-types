# Observations within one typed store

A successful receiving check belongs to the store that performed it. Ordinary internal prototype delegation and the final own-property definition must not turn that one store into several observable admissions.

```js
const raw: any = { a: { x: 0 } };
const view: { a: { x: number } } = raw;
let reads = 0;
const candidate: any = { get x() { reads++; return 1; } };
raw.a = candidate;
// reads === 1; view.a === candidate
```

The same rule applies when the property was inherited, when Reflect.set supplies a different lookup target and receiver, and when an owned Array or fixed tuple performs its final definition. Prototype depth must not multiply observations. A later independent assignment of candidate is another receiving operation and performs its own admission.

## Direction

A valid solution preserves the first receiving check's position, converted result and selected construction, ordinary permission/refusal order, independent callback effects, established backing, and reference liveness. It must identify the destination receiver and key rather than the current lookup object. It must not equate a construction request with successful admission or reuse evidence after an intervening effectful operation.

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Carry successful admission through the same callback-free internal store continuation | Getter counts and exception order do not depend on prototype depth. Autocomplete still describes the same destination contract; a wrong value fails at its existing receiving check. | ECMAScript ordinary Set delegates internally and eventually defines on the receiver. Rust specifies expression effects and evaluation order, but has no prototype/getter-based runtime admission; only the distinction between one operation and implementation forwarding is analogous. | A checked value and destination/type evidence can remain in registers or an internal frame. No public property, shape transition or global IC dependency. | Preserve exact receiver/key, converted value and SameType backing. Ordinary permission and liveness checks still run. | **Recommend**, independently of implementation effort. |
| Move every check to the final own-property definition | Avoids some repeats, but getters run later or not at all when a nonwritable property or setter intercepts the operation. | Ordinary JavaScript permits setters; Rust field assignment does not supply an equivalent receiving callback protocol. | Few checks but requires a different effect and failure-order model. | Changes existing admission-before-refusal and setter-input guarantees. | Reject. |
| Repeat admission at each internal delegation and definition | The same statement can read a getter two, three or more times solely because the chain changed; autocomplete cannot explain the extra effects. | Neither language's internal implementation steps prescribe these extra source observations. | Repeated graph walks, conversions, copies and specialization work. | Can introduce an exception after an already successful check without an intervening user operation. | Reject. |
| Cache successful admission globally by source value or receiver | Later independent stores can silently skip their getters, and aliases can reuse stale authority. | Rust access validity is contextual; value equality does not grant a fresh borrow. JavaScript objects can change between operations. | Invalidation, retention and cache-accounting burden; may add object metadata/shape effects. | Loses operation identity, destination key and intervening effects. | Reject. |
| Reject structural getters at typed stores | Removes established useful receiving operations without an autocomplete-based explanation. | Rust's lack of prototype getters is not grounds to remove a JavaScript feature. | Cheap refusal at the expense of valid programs. | Changes the structural admission rules rather than fixing forwarding. | Reject. |

## Scope of successful evidence

The receiving Object and ordinary property key must be identical, including Symbol identity and empty-string keys. The value forwarded is the actual result of the successful admission, including numeric conversion, value copying or callable specialization. The required backing must be SameType; a subtype relation alone is insufficient. A fresh construction selection remains input to admission, not evidence of success.

An ordinary internal descriptor/prototype walk can forward this evidence without additional reads of the candidate. Continue checking readonly fields, extensibility, writable/configurable attributes, tuple position/extent, inherited-storage dependencies and reference liveness at their existing phases. Admission can itself run a getter that freezes or changes the destination; its later write must observe that change. Earlier nested stores and effects are not rolled back when the outer operation fails.

Do not carry evidence through a setter invocation, a Proxy operation, an unknown internal-method override, or a representation-specific property observation that can read through backing. These are barriers even when a Proxy ultimately forwards to an ordinary target: selecting a trap can itself call a getter. A separately invoked store or definition starts without the outer store's evidence, including reentry on the same receiver and key. Distinct receiving checks that can execute user code do not accumulate a reusable cache of earlier checks.

This preserves the existing Proxy policy. Its incoming check, a store independently invoked by a trap, and its post-trap check are separated by user code and are distinct. A successful ordinary store inside the trap still performs its own check once. The rule does not reinterpret three such checks as internal forwarding or allow a trap to carry evidence from before its callback.

| Transport direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Explicit private continuation argument | No observable syntax or metadata; internal forwarding has predictable effects. | Similar to retaining an intermediate checked result in an expression evaluation, not a Rust borrow token exposed to JavaScript. | Optimizers can erase the carrier; the interpreter can use a small private record. | Only audited forwarding sites receive it; callback calls cannot inherit it accidentally. | **Recommend.** |
| Ambient receiver or execution-context flag | Usually looks equivalent until a callback recursively stores the same value. | A function frame is not Rust's ownership/lifetime proof and can contain independent operations. | Cheap lookup, but requires fragile reentry restoration and barriers everywhere. | Same-frame native/reflection callbacks can accidentally reuse authority. | Reject. |
| Tag values or user-visible descriptors | Risks retaining evidence across unrelated uses or reflective reconstruction. | A Rust value's type does not encode whether this particular JavaScript destination check ran. | Adds metadata, retention and invalidation work. | Confuses the returned value or descriptor with the receiving operation. | Reject. |

Owned Array and fixed-tuple element checks can use this continuation, including their ordinary final definition. Scalar-rest planning, different simultaneous backing requirements, arbitrary exotics and backing-specific stores retain their separate observation and publication requirements; this is not permission to bypass them.

## Separate failed-membership question

The repair concerns forwarding an already successful check. A separate preserved program shows a getter read twice inside one RequireType when structural membership fails and conversion is then attempted. No successful store evidence exists at that point. Reusing the source observation for that conversion needs its own failure/conversion ledger, including conversion selection, reentry and failure ordering; it is not justified by this continuation. The follow-up retains a one-observation requirement and compares directions separately rather than treating two reads as a passing result.

Sources: [ECMAScript ordinary Set](https://tc39.es/ecma262/#sec-ordinarysetwithowndescriptor), [Rust expression evaluation](https://doc.rust-lang.org/reference/expressions.html#evaluation-order-of-operands), and the proposal's receiving checks, selected construction, inherited storage and reference-liveness rules. Performance recommendations are design judgments.
