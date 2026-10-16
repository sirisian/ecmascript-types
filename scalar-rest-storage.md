# Scalar rest storage with required finite runs

This increment implements a bounded part of [heterogeneous rest storage](heterogeneous-rest-storage.md): owned Arrays admitted at tuples with scalar dynamic rests and required fixed or finite runs, such as `[...[].<number>, ...[].<string>]`. The complete heterogeneous-rest design remains the recommendation. This specialization does not complete its six general gates.

## Supported domain and observable behavior

The new path applies when the existing stable positional-domain path does not apply, and ordered tuple expansion consists of required fixed positions, finite array rests, or dynamic array rests over an unparameterized `number`, `string`, `boolean`, `bigint`, `symbol`, `null`, `undefined`, `float16`, `float32`, `float64`, or resolved `int<N>`/`uint<N>`. Dynamic runs may be empty; required positions and finite rests retain their exact counts. Nested tuple spreads are included, with no active recursive expansion or unresolved/unsafe count. Three or more runs and repeated nonadjacent types are included. Defaults attached to positions, union-valued rests, object-valued rests and metadata-parameterized destinations are outside this specialization. Existing stable-domain paths stay in place.

The declaration supplies an ordered guard and a declaration-derived per-index union projection at each admitted length. The union permits ordinary location typing; the guard preserves the correlation between positions. Existing narrower element, tuple and property contracts remain authoritative. Acquiring another guard adds an obligation rather than replacing the backing or freezing a partition.

```js
type T = [...[].<number>, ...[].<string>];
const raw: any = [1, "s"];
const t: T = raw;
raw[1] = "t"; // valid String replacement
raw[0] = "a"; // valid repartition: two Strings
raw[1] = 2;   // ordinary String conversion: stores "2"
raw is T;     // true
```

Direct, nested, union, intersection, parameter and return admission retain the same guard when they admit an owned Array. Pure `is` acquires none. Failed admission publishes none of that admission's contracts; independently completed boundaries and effects remain. Guard acquisition revalidates the current own data elements, including when intervening work established different backing. A retained object or old observation is not proof that the current contents still qualify.

An implicit or explicit conversion first retains complete membership success. Otherwise it selects the existing leftmost-greedy count assignment from viable scalar destinations, executes only the selected effectful conversions once, validates their results and returns a backed result. Whole-sequence membership takes precedence over the conversion ranking: `[1,"s",2]` already fits `[number...,string...,number...]` and must not turn its last Number into a String. Implicit numeric-family restrictions still apply; explicit conversion keeps wrapping behavior. Pure intrinsic numeric conversion facts can give an earlier rest's item back to a later rest, including when the interpreter represents the source numeric value as an object.

A single store, reflective definition, gap initialization or length growth plans against the entire prospective sequence and all acquired obligations. Unsupplied new positions use defaults at the selected destinations; an actual `undefined` value is supplied, not missing. Default construction gives dynamic runs zero elements and independently initializes every required scalar position; the result carries its guard. Zero-count runs request no default. `symbol` has no fabricated default. The proof-of-concept planner retains the current values and guard/backing identities, then revalidates them after conversion. A callback's independent write is preserved, while the receiving store fails on changed consulted state. Identical guard reacquisition does not itself invalidate a plan.

## Decision: pure conversion facts during selection

The previous decision forbids speculative user conversions. It leaves a practical question: whether an intrinsically pure scalar range check may participate in assignment. All alternatives below retain whole membership first and execute user code only after selecting its destination.

| Direction | Ergonomics and completion | Prior art | Production-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Select only by source category, then propagate even a predictable range failure | `[1,300]` cannot use the available uint16 rest after uint8; completion suggests a valid destination that is never tried | Rust has destination-directed coercion but no dynamic greedy rest assignment | Cheapest category tests; avoids useful exact range facts | Safe but loses this proposal's value-sensitive viable assignment; no reference advantage | Rejected: predictable pure failure need not select a losing assignment |
| Run all conversions, including operators, before selecting | A wrong guess invokes unused operators; effects make retry surprising | Rust coercion sites do not authorize speculative JavaScript calls | Repeated calls, allocations and invalidation harm IC stability | Cannot undo effects or preserve the selected-operation rule | Rejected independently of implementation effort |
| Retain pure scalar results or exact failure facts; classify effectful candidates without executing them | Range giveback works; a selected operator can still throw without hidden retries; completion can explain scalar alternatives | Rust supplies the destination-before-conversion analogy only; the greedy selection is this proposal's rule | Pure checks can be folded/shared; only selected calls require effect barriers | No discarded user effects or stale-reference authority; every selected result is checked | **Recommended** |

The interpreter currently retains candidate results for distinct declared scalar records. A production implementation may retain a pure conversion recipe or range fact instead. This is not permission to call getters, metadata hooks, constructors or user conversion operators while searching. If no assignment is viable, propagate the first retained pure scalar conversion failure in input-index and written-run order, or TypeError if none exists. Once a selected effectful conversion begins, its failure is propagated without choosing another assignment.

## Intrinsic numeric values represented as objects

```js
type T = [...[].<bigint>, ...[].<number>];
const source: any = [float128(7), float128(Infinity)];
const result: any = (source := T); // [7n, Infinity]
result is T;                    // true, with acquired scalar backing
```

The first BigInt conversion succeeds. The second has a pure RangeError, so greedy assignment gives that item back to the Number run. The same rule applies to unparameterized decimal and rational intrinsic payloads. A required BigInt position still requires an item; explicit finite integer wrapping and implicit numeric-family prohibitions retain their existing rules. Whole membership still wins: a later Number store that already leaves a valid sequence does not have to become a BigInt.

| Direction | Ergonomics, completion and wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Treat every interpreter Object as effectful | The same advertised numeric destinations behave differently for Number and a heap-represented numeric value; a predictable failing choice hides a valid later run | Rust's coercion sites determine destinations, not behavior from interpreter object allocation; it has no greedy runtime rest analogue | Cheap classification but representation leaks into semantics and obstructs unboxing | Prevents speculative calls but loses the existing pure-failure giveback rule; no liveness benefit | Rejected |
| Treat every numeric-looking Object as pure | A getter, Proxy or user operator can run for an unused candidate; wrong guesses have visible effects | Rust coercion sites do not justify speculative JavaScript coercion | Shape guesses and arbitrary calls disrupt ICs and deoptimization assumptions | Object identity or visible slot names do not prove intrinsic payload or effect freedom | Rejected independently of implementation effort |
| Classify the actual intrinsic payload conversion, excluding metadata and user dispatch | Numeric candidates use the same pure-failure rule; selected user conversions still run once and may throw | Rust provides the destination-first analogy only; this proposal defines greedy selection and effect barriers | Internal numeric tags permit range checks without ordinary property access; shared plans do not create per-value object shapes | Preserves existing conversions, backing checks and final reference revalidation | **Recommended** |

The engine's bounded implementation recognizes its own decimal, rational and float128 payload records, requires no carried metadata, and distinguishes an explicit intrinsic dispatch from a candidate that could invoke a class conversion entry. An unrelated class operator does not make a known intrinsic explicit cast effectful. It does not trust public property names, proxy targets, `valueOf` or `Symbol.toPrimitive`. Existing primitive-value planning remains in place. General metadata operations and user-class applicability are not completed here. The pure candidate is retained under this receiving operation, not cached indefinitely by source identity.

The [Rust coercion-site rules](https://doc.rust-lang.org/reference/type-coercions.html) establish the limited prior-art comparison above. The recommendation is an inference from this proposal's existing pure-candidate rule. Object-valued **destinations** have the separate unresolved dependencies recorded in [object rest dependencies](object-rest-dependencies.md).

## Decision: several acquired guards and conversion priority

All guards are conjunctive. Their ordering must not silently become an ordering of Type Object identities. The open choice is which declared greedy order ranks a store's otherwise valid conversion assignments.

| Direction | Ergonomics and completion | Prior art | Production-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| The newest view replaces the earlier ranking or backing | Passing through a view changes existing aliases' conversion preference; autocomplete no longer explains the original storage | Rust's invariant mutable place supports retaining the established destination, not replacing it through a view | Rewrites descriptors and invalidates store ICs on every view | Replacing backing can invalidate reference authority; ranking-only replacement still changes existing alias behavior | Rejected |
| Canonically reorder all acquired declarations | Acquisition order disappears, but an unrelated spelling/normalization order can decide which conversion happens | Rust has no comparable runtime conjunction-ranking rule | Requires a canonical combined descriptor and potentially new shape/IC variants | Could be safe with all guards, but changes the original declared greedy preference without a type-system reason | Rejected |
| Refuse every store with multiple possible scalar conversions | Completion advertises alternatives that become unusable through another view | Rust does not need this runtime ambiguity policy | Simple refusal, lost valid stores | Safe but stronger than the committed greedy assignment rule | Rejected |
| Keep the earliest acquired guard's written greedy ranking and constrain it by every guard and existing backing contract | The established storage keeps its preference; adding a view may restrict valid writes but cannot replace its preference; wrong guesses fail at the same checked store | Rust supports stable invariant destination contracts, not this precise runtime ranking | Stable owner descriptor with added guard dependencies; no per-value partition shapes | Preserves established location authority and all guards; liveness remains independent | **Recommended** |

For an owned Array acquired first as `[uint8...,string...]` and then as `[string...,uint8...]`, appending Number 1 converts it to uint8. Reversing the acquisitions chooses String. Both satisfy both guards. This is storage history, like retaining an earlier narrower backing, not unequal intersection types. Reacquiring an identical guard does not reorder it. Guards obtained by one successful compound admission keep that admission's selected traversal order; membership and Type Object equality remain independent of this operational ranking. General effectful intersection conversion remains open.

## References, intrinsic effects and transport

A freshly acquired `[number...,string...]` position has invariant `number|string` storage. Its current Number or String does not justify a narrow reference parameter. An earlier `[number,number]` backing still permits a `ref number`. Proxy aliases of an already guarded owned Array use the owner's location type, allocation generation and removal identity. Replacement preserves identity; removal/recreation and growth relocation retain the existing TypeError liveness checks.

Intrinsic movement preserves values: it cannot repair an invalid permutation by turning a Number into a String. Existing declaration-based movement preflight and ordinary property order remain in effect. Each published intermediate write must satisfy the guard. `reverse()` on `[1,"s"]` may therefore write `"s"` at index 0 and fail on the next write, leaving `["s","s"]`. This is a valid guarded state with ordinary earlier effects, not an atomic rollback. Growing insertion retains its existing private-tail publication rules, and failure discards only the unpublished tail.

Copying operations whose unbounded source has no determined shape retain the existing ordinary untyped-result rule. The tests check that `slice`, `toReversed`, `with` and `toSorted` do not invent a guard for such a copy. Fresh Proxy/array-like acquisition, species transport into custom typed receivers, full metadata transport and nested object-role dependencies are not completed by this work. Proxy aliases of an already acquired owner are covered separately.


## Required finite runs and length-sensitive projections

```js
type T = [...[].<uint8>, ...[1].<string>, ...[].<boolean>];
const source: any = [1, 2, 3];
const t = (source := T); // [uint8(1), uint8(2), "3"]
t is T;                 // true
```

Required counts participate in greedy assignment before any selected user conversion. They are never consumed by the first rest's fallback. Finite array counts stay compact; fixed positions contribute count one, nested tuple spreads preserve order, and zero extents contribute no positions. The planner retains all viable states of every additional guard: the earliest matching run alone cannot prove membership when an identical scalar must also fill a later required position. An impossible total length fails before indexed Gets.

For an index `i` at admitted length `n`, the position contract is the union of the scalar destinations that can occupy `i` in any complete count assignment of length `n`, without considering current values. For `[number...,string{1},boolean...]` at length three, these contracts are `number|string`, `number|string|boolean`, and `string|boolean`. The required String may move without freezing a partition. The actual values must also satisfy the complete ordered guard.

### Decision: contracts across a finite-run resize

| Direction | Ergonomics / autocomplete / wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Freeze the current partition or current length | A value replacement or append unexpectedly changes what completion permits; an observed scalar suggests an unjustifiably narrow borrow | Rust preserves declared element types; it does not derive mutable contracts from current values | Many value-dependent shapes and IC invalidations | Loses declared alternatives or changes live reference authority | Rejected: observational partitions are not declarations |
| Give every position the union of every written scalar forever | Simple completion, but advertises Boolean at a first position that cannot contain it and forces unnecessarily broad reference parameters | Rust arrays have a homogeneous declared element type; that does not justify erasing a heterogeneous tuple's position information | Simple stable descriptor, weaker specialization | Guards preserve membership, but the location type discards declaration-proved exclusions | Rejected: use the exact declaration projection |
| Change surviving location contracts on resize and invalidate affected references | More resizing succeeds, but a valid borrowed location can change type through another alias | Rust prevents incompatible mutation while borrowed; it does not silently retag mutable places | Requires position retagging and additional invalidation/IC variants | Conflicts with the existing SameType rule for surviving locations | Rejected unless the general reference/storage design is changed first |
| Derive each length's exact projection and require SameType at surviving indices | Completion reflects all declaration-possible destinations; a wrong resize fails before element conversion | Rust supports stable declared mutable places, not these runtime count automata | Shared compact run descriptors; boundary comparisons can skip finite extents; no value-partition shapes | Preserves the existing resizing and independent liveness rules | **Recommended**, independently of implementation effort |

Consequently a newly backed `[number...,string{1},boolean...]` of length three cannot grow to four: its old last position would broaden. A previously established broader homogeneous backing can permit that growth, provided every guard still accepts the result. New views retain that earlier backing. Intrinsic preflight checks required length minima even when earlier backing is broader. Indexed growth checks geometry before a supplied conversion operator runs. Refused growth neither relocates storage nor invalidates an otherwise live reference.

### Decision: a truncation blocked after some deletions

Finite-run languages need not be prefix-closed. For `[number...,string...,number{2}]`, `[1,2,"s",3,4]` and its length-two prefix are members, but the length-three prefix is not. If index 2 is nonconfigurable or independently protected by a property contract, ordinary truncation to two would stop at three after deleting indices 4 and 3.

| Direction | Ergonomics / autocomplete / wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and references | Other / decision |
| Validate only the requested final prefix | Requested length looks valid, but failure leaves aliases reading a nonmember | ECMAScript permits partial truncation; Rust's homogeneous arrays do not have this correlated state | Cheapest check | Publishes invalid guarded storage and may remove live locations before discovering it | Rejected |
| Delete, then roll back if the stopping prefix is invalid | Conceals earlier deletion effects and complicates reference identity | Neither ECMAScript partial deletion nor Rust stable places supplies this rollback rule | Requires descriptors/identities to be saved and restored | Conflicts with earlier-effects and removal-identity rules | Rejected |
| Require every shorter intermediate prefix to be valid | Rejects successful ordinary truncation even when no barrier exists; completion cannot explain an actual obstacle | Rust's homogeneous domain is prefix-closed, so the analogy does not establish this restriction | Pure but overrestrictive | Safe, but unnecessarily excludes declaration-valid operations | Rejected |
| Inspect actual owned descriptor/property barriers, validate requested and actual stopping prefixes before deletion | A wrong truncation fails without this operation's deletions; a valid blocked result retains ordinary partial deletion | Keeps ECMAScript's barrier behavior when type-safe; Rust motivates valid storage, not this exact algorithm | Pure own-descriptor scan can share the existing deletion-key walk; no getters or new object shapes | Both possible published outcomes satisfy guards and surviving-location contracts; no rollback or stale-reference revival | **Recommended** |

Apply this check after the existing nonwritable-length refusal, with current owned descriptors and acquired property contracts. If either prospective prefix is invalid, throw TypeError before deleting. When the blocked prefix is valid, retain ordinary descending deletion, restored length, the original failure, and requested writability handling. Independent effects of earlier length conversion remain; the preflight does not undo them. An intrinsic's earlier valid element writes likewise remain if its eventual truncation is refused.

Rust's [fixed arrays](https://doc.rust-lang.org/reference/types/array.html) and [destination-directed coercions](https://doc.rust-lang.org/reference/type-coercions.html) are limited analogies: neither provides this heterogeneous runtime sequence language. The barrier comparison uses [ECMAScript ArraySetLength](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-arraysetlength). These recommendations are proposal-specific inferences.

## Costs, proof limits and next work

A scalar pattern stores written runs, not a current partition or expanded finite count. This interpreter can scan current owned data and allocate candidate rows on a guarded store; memoized assignment tracks the other guards' sets of reachable run/count states as well as the first pattern's run and input index. There is no claim that this implementation meets V8 performance targets. A real engine can share declaration descriptors, guard store ICs on those descriptors and maintain invalidation-aware run summaries. Repeated guards and long alternating patterns still need broader complexity and resource-accounting work.

The new conformance corpus tests desired outcomes rather than preserving old failures. The delivery report records exact build, paired-baseline, bounded regression, typecheck and patch-application evidence. This document deliberately does not mark general contextual literal distribution, general heterogeneous conversion beyond the required scalar family, object/metadata effects, recursive inference, portable evaluation budgets or full regression completion as done.

Prior art used above: [Rust mutable-reference variance](https://doc.rust-lang.org/reference/subtyping.html), [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html), and [ECMAScript reverse property-operation order](https://tc39.es/ecma262/multipage/indexed-collections.html#sec-array.prototype.reverse). The recommendations are inferences for this proposal; none of these sources implements its runtime rest guards.
