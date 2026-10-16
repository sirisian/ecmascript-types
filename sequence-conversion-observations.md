# Source observations during sequence conversion

A concrete array or tuple receiving operation keeps the source values it has
already observed when membership fails and elementwise conversion follows.
Its membership test and conversion do not each get a new reading of `length`
or of an indexed property. This applies to implicit and explicit conversion.

```js
let reads = 0;
const raw: any = [];
Object.defineProperty(raw, "0", { get() { return ++reads; } });
const value: [uint8] = raw;
// reads === 1; value[0] is uint8(1).
```

The previous implementation read the element during membership, rejected the
plain Number as a member of `uint8`, then read it again during conversion. The
result contained 2; a getter refusing a second call could instead throw.

## Required properties

1. A successful source Get is retained for the rest of this concrete operation.
   Retain actual `undefined` and object identities, not only truthy results.
2. Retention is lazy. It must not cause a later getter to run before an earlier
   conversion fails, or before effects of an earlier converting constructor.
3. Preserve existing membership order. Established positional domains and
   homogeneous arrays stop at the first incompatible position. General rest
   matching obtains its indexed input in order before searching assignments;
   conversion must not replay that input.
4. Retain the first length value and any length coercion already performed by
   this operation. Later source growth or shrinkage does not replace that count.
   This does not change the accepted count range or conversion rules.
5. Observation is not admission. Failed speculative storage evidence is still
   discarded, and a produced result must complete its own backing before
   publication. Earlier independently completed nested boundaries keep their
   effects, even if the outer conversion later fails.
6. Separate and reentrant receiving operations get separate observations.
   Standalone `is` is independent. No observation authorizes a reference borrow,
   proves liveness, or snapshots an object's descendants.

## Directions considered

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Read again after failed membership | Autocomplete gives no hint that one boundary can call a getter twice; a wrong assumption changes values or throws | Ordinary ECMAScript Gets are independent, but the duplicate phase is internal; Rust iteration does not imply rereading consumed items | No retention buffer, but duplicated observable calls prevent elimination and can dominate cost | Checks and conversion use different values; replay frustrates later rest selection | Reject: preserves the demonstrated defect |
| Eagerly snapshot the whole source before checking | Easy single-read story, but an unreachable later getter can now throw or perform effects | Explicit collection exists in Rust, but a lazy iterator is not eagerly collected; Array methods read later positions as visited | O(n) live values even on an immediate failure; unnecessary calls and GC roots | Changes failure precedence and ordering with user conversions | Reject: violates lazy evaluation and existing order |
| One lazy observation record per concrete receiving operation | One source read supplies both phases; no API or autocomplete change; independent calls remain observable | Rust Peekable retains a produced item, while ECMAScript array traversal captures length and visits positions in order; neither is an exact conversion analogue | O(k) retained observed values on fallback paths; no source shape transition or extra object IC state; effects-free ordinary storage can use guarded optimized paths | Preserves first values, lazy reads, failure effects, independent boundaries and reference rules | Recommend independently of implementation effort |
| Persistent cache attached to a source or shared by all active boundaries | Later or reentrant calls can unexpectedly see stale values, with no syntax exposing the cache | Rust iterator state belongs to an iterator instance, not a global cache of arbitrary objects | Source properties would change shapes; side tables still add lookup, invalidation and GC cost | Conflates separate operations, aliases and reentrancy; can falsely imply backing or liveness | Reject: wrong lifetime |
| Remove membership and always construct a converted Array | Avoids phase replay, but changes identity for already compatible inputs | Rust collecting into a new container is explicit; it does not justify changing this proposal's established admission | Allocates/copies compatible inputs and loses in-place fast paths | Breaks identity and established backing/reference behavior | Reject: changes an independent contract |

Rust's [`Iterator::peekable`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.peekable)
is a limited analogy: the first peek invokes `next`, including its effects, and
later consumption reuses the item. Rust has no JavaScript property getters and
this proposal uses indexed Gets, not the iterator protocol. ECMAScript's
[`Array.prototype.map`](https://tc39.es/ecma262/multipage/indexed-collections.html#sec-array.prototype.map)
captures length and reads values as positions are visited; its hole policy and
callback algorithm are not imported into typed conversion.

## Scope and implementation

The engine carries a `SequenceObservation` through the direct concrete
array/tuple membership and conversion phases. It retains the raw length,
optional converted count and indexed observations. Storage-admission rollback
does not erase source observations. The record is neither installed on the
source nor reused by a subsequent invocation. The existing successful-admission
path retains its separate descriptor/evidence validation.

This repairs root sequence reads, including the current general-rest path. It
does not solve heterogeneous rest partition selection or correlated backing.
`[...number, ...string]` still has direct and nested backing defects. Union,
intersection, metadata and nested structural traversal require their own
transport audit: this patch does not share one record across distinct candidate,
member or child operations. Rechecking descendant properties is not covered by
retaining their containing indexed object's identity. Source object changes
remain observable wherever the existing algorithm has not yet read a value.

The conformance corpus observes getters, length coercion, holes, failure order,
identity, later source mutation, independent/reentrant boundaries and completed
nested admission. Module, cross-realm Proxy, Function, eval and disabled-feature
controls use the same implementation. No existing expectation is weakened.

## Regression found during verification

An existing target-typed-construction test failed on the pulled baseline:
admitting an Array of constructed vectors falsely reported an independent
element definition. The engine's core value-category `SameType` omitted
`VectorValue`, so `SameValue(v, v)` was false even for the very same retained
vector. The category switch now recognizes vectors, as it already recognizes
typed numbers. This is not the proposal Type Record relation and introduces no
conversion between vector specializations. The admission conflict check stays
in place; an independently replaced vector still invalidates its observation.
This repairs the demonstrated reflexivity/admission failure without claiming a
general audit of vector equality.

The expanded vector subset also exposed a stale acceptance test for discarded
`string === boolean`. The current disjoint-comparison obligation explicitly
applies to discarded results. The test now requires a static error for each of
`===` and `!==`, with compatible and unknown-operand controls. Neither the
runtime nor the specification is changed for this test correction.

The existing accessor-conversion corpus also recorded two getter calls. Its
expected count is now one, as required by this change; its copying, converted
value and preserved-source-accessor assertions are unchanged.
