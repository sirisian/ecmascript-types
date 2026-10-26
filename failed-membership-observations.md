# Structural observations across failed membership

A receiving operation must convert the values it already observed. Failure of its first structural membership attempt is not permission to invoke the same getter again and accept a different answer.

```javascript
let reads = 0;
const source: any = { get x() { reads++; return reads === 1 ? "bad" : 1; } };
const out: { x: number } = source; // TypeError; reads === 1
```

The rule also matters when conversion succeeds. If `x` already satisfies its type and `y` needs numeric conversion, converting `y` must not reread `x`. A setter or Proxy write that changes `x` is different: the receiving operation must observe that write's result before claiming it satisfies the target.

## Input retention

A valid direction must retain actual input values without claiming that a failed type check succeeded; preserve implicit versus explicit conversion, union selection, ordinary write effects and refusal order; respect current data-path and reference-liveness checks; and keep independently invoked operations independent.

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Retain raw values by Object identity and property key within the receiving operation | A changing getter cannot repair a wrong value on a hidden retry. Autocomplete's member type still describes the value that must be accepted or converted; a wrong guess fails predictably. | ECMAScript Get may invoke a getter. Rust defines expression evaluation and effects, but has no equivalent structural getter admission; retaining evaluated inputs is the limited analogy. | An internal frame can retain values and ordinary path dependencies. No public property, shape transition or global IC entry is required. | Failed membership clears speculative admission/selection evidence, not input values. Every actual type, backing and liveness obligation still runs. | **Recommend**, independently of implementation effort. |
| Repeat every Get when membership fails or another union member is tried | Hidden retries change values, getter counts and exceptions. The same autocomplete result gives no warning about retry-dependent acceptance. | Ordinary Get permits another callback only when another read is specified; Rust supplies no reason to duplicate evaluation. | Repeats structural walks and callbacks; complicates effect-aware optimization. | A later answer can conceal the original invalid input and change failure precedence. | Reject. |
| Refuse immediately after failed membership | Predictable counts, but valid numeric/member conversions stop working. | Rust's static type rejection is not an analogy for this proposal's receiving conversions. | Cheap refusal; it removes required behavior. | Contradicts established conversion and union-selection rules. | Reject. |
| Share observations across operations or use an ambient context | A later assignment or reentrant callback can silently use an old answer. | A Rust value's type does not authorize a later access; JavaScript objects and callbacks remain mutable. | Global retention and invalidation cost; metadata may affect shapes and IC dependencies. | Confuses input observation with continuing authority and can skip fresh liveness checks. | Reject. |

Use an explicit internal argument through membership, member conversion, alternatives and completion of the same receiving operation. Raw observations are keyed by the actual source Object and property key, including Symbol identity and the empty string. Repeated aliases to the same source location share that operation's input. A nested member's internal conversion can use it; a setter, getter or user conversion that independently invokes another receiving operation starts a fresh one. Pure `is` checks do not acquire or share this receiving-operation record. Consecutive pure checks remain independent.

The record is neither successful admission nor backing, a selected construction, a stable snapshot of the whole object, or a reference-liveness certificate. Cached values still enter the same membership and conversion rules. Abrupt Get completions propagate; they are not stored as successful observations. Independent ordinary-data or lookup-path changes invalidate retained data evidence at its existing validation points. They cannot be accepted by rereading a convenient replacement. Earlier effects survive refusal.

## Presence and key domains

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Retain member values; keep HasProperty, own-key and descriptor-domain checks at their existing phases | A required property removed during conversion is still missing; a newly present optional property must satisfy its type. Autocomplete keeps the final shape contract. | ECMAScript distinguishes Get from presence, keys and descriptors. Rust fields have a static domain, so the dynamic-domain analogy is weak. | Separate value slots and existing shape/domain guards; an engine can optimize ordinary checks without changing trap behavior. | Preserves current shape admission and publication requirements while preventing value replay. | **Recommend.** |
| Snapshot presence and all keys with the initial values | A conversion can add an invalid optional member that is never checked, or remove a required one while admission succeeds. | Rust static fields do not justify freezing a JavaScript object's domain. | More retained state; fewer later checks but different semantics. | Weakens final shape obligations and observable presence/key phases. | Reject. |
| Restart both values and domains after every effect | Makes successful conversion depend on fresh getter answers; callbacks can cause unbounded restarts. | Neither language requires this retry model for a receiving operation. | Repeated IC misses, callbacks and validation work. | Loses the input-retention requirement and needs a new termination/failure policy. | Reject. |

This rule does not promise that HasProperty, OwnPropertyKeys or descriptor operations execute only once. Their phase-specific behavior remains live. Existing sequence length/element observations remain a separate mechanism; an object member reached inside an array or tuple conversion uses this structural-value rule without merging unrelated sequence operations.

## Writes and completion

| Direction after an operation-owned write | Ergonomics | Prior art including Rust | Real-engine performance | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Advance verified ordinary data; discard a setter/Proxy input so required post-write validation observes the result | Numeric conversion, callable specialization and tuple defaults remain usable. An ignored setter cannot pretend to have installed a numeric result. | ECMAScript setters and Proxy writes need not store their argument. Rust direct field writes lack those interception hooks; retaining a direct written value is the closer analogy. | Ordinary writes can update an internal slot after descriptor/path guards; effectful writes pay the required new Get. | Validate an inherited supplier/path before accepting its new own shadow. Preserve fresh result and liveness checks for effectful writes. | **Recommend.** |
| Trust every successful Set as proof of its argument | A setter that ignores the argument can produce a falsely typed result. | Contradicts JavaScript interception semantics; Rust direct writes are not equivalent. | Cheap but unsound. | Set success alone does not prove stored representation or type. | Reject. |
| Always reread ordinary data as well | Adds unnecessary work and would be unsafe if generalized to unchanged accessor inputs. | JavaScript permits direct internal descriptor validation; Rust does not require reevaluating a source after a direct write. | Extra loads and potentially callbacks instead of using verified data. | Can be made correct for ordinary data, but gives no stronger guarantee than path/value validation. | Reject in favor of the precise direct-data rule. |
| Keep the old input after successful conversion/materialization | A valid in-place specialization or tuple replacement later looks like an unrelated conflicting mutation. | Neither direct assignment nor Rust value production preserves the overwritten value as current storage. | Small record, but rejects valid programs. | The ledger no longer describes this operation's own completed write. | Reject. |

Advance only the receiving operation's record. A nested callable specialization, value copy, defaulted-tuple replacement or missing-own-element initialization that belongs to its completion must update the same record when it writes. An independently suspended operation's record is unchanged. For an ordinary inherited value, validate the old supplying path while allowing only the new receiver shadow, then retain the verified own value. For a setter or Proxy, remove that key's retained input; the existing post-write membership phase must inspect the result. This does not add a new universal post-write policy to every materialization boundary.

Failed alternatives retain completed ordinary effects, as before. They discard speculative success/selection records but retain raw inputs and this operation's verified writes. No failed outer admission publishes its pending contracts; independently completed nested boundaries keep their effects and contracts. Refusal does not itself remove, relocate or revive a reference.

An explicit cast followed by a separate inferred receiving binding can perform two receiving operations. The one-input rule applies inside each operation, not to every syntactic statement as a whole. Successful typed-store forwarding has its own stricter callback-barrier rule in [typed-store-observations.md](typed-store-observations.md); raw inputs here must never stand in for that successful evidence.

Sources: [ECMAScript ordinary Get](https://tc39.es/ecma262/#sec-ordinaryget), [ordinary Set](https://tc39.es/ecma262/#sec-ordinarysetwithowndescriptor), [Proxy Set](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots-set-p-v-receiver), and [Rust expression evaluation](https://doc.rust-lang.org/reference/expressions.html#evaluation-order-of-operands). Real-engine cost statements are design judgments, not measurements of a production implementation.
