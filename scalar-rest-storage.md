# Dynamic scalar rest storage

This increment implements a bounded part of [heterogeneous rest storage](heterogeneous-rest-storage.md): owned Arrays admitted at tuples made entirely of dynamic scalar array rests, such as `[...[].<number>, ...[].<string>]`. The complete heterogeneous-rest design remains the recommendation. This specialization does not complete its six general gates.

## Supported domain and observable behavior

The new path applies when the existing stable positional-domain path does not apply, and every written element is a dynamic array rest over an unparameterized `number`, `string`, `boolean`, `bigint`, `symbol`, `null`, `undefined`, `float16`, `float32`, `float64`, or resolved `int<N>`/`uint<N>`. Each run may be empty. Three or more runs and repeated nonadjacent types are included. Fixed positions, finite rests, defaults attached to positions, union-valued rests, object-valued rests and metadata-parameterized destinations are outside this specialization. Existing stable-domain paths stay in place.

The declaration supplies an ordered guard and a stable per-index union projection. The union permits ordinary location typing; the guard preserves the correlation between positions. Existing narrower element, tuple and property contracts remain authoritative. Acquiring another guard adds an obligation rather than replacing the backing or freezing a partition.

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

An implicit or explicit conversion first retains complete membership success. Otherwise it selects the existing leftmost-greedy count assignment from viable scalar destinations, executes only the selected effectful conversions once, validates their results and returns a backed result. Whole-sequence membership takes precedence over the conversion ranking: `[1,"s",2]` already fits `[number...,string...,number...]` and must not turn its last Number into a String. Implicit numeric-family restrictions still apply; explicit conversion keeps wrapping behavior. Range checks on pure primitive conversions can give an earlier rest's item back to a later rest.

A single store, reflective definition, gap initialization or length growth plans against the entire prospective sequence and all acquired obligations. Unsupplied new positions use defaults at the selected destinations; an actual `undefined` value is supplied, not missing. An empty default carries its guard. `symbol` has no fabricated default. The proof-of-concept planner retains the current values and guard/backing identities, then revalidates them after conversion. A callback's independent write is preserved, while the receiving store fails on changed consulted state. Identical guard reacquisition does not itself invalidate a plan.

## Decision: pure conversion facts during selection

The previous decision forbids speculative user conversions. It leaves a practical question: whether an intrinsically pure scalar range check may participate in assignment. All alternatives below retain whole membership first and execute user code only after selecting its destination.

| Direction | Ergonomics and completion | Prior art | Production-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Select only by source category, then propagate even a predictable range failure | `[1,300]` cannot use the available uint16 rest after uint8; completion suggests a valid destination that is never tried | Rust has destination-directed coercion but no dynamic greedy rest assignment | Cheapest category tests; avoids useful exact range facts | Safe but loses this proposal's value-sensitive viable assignment; no reference advantage | Rejected: predictable pure failure need not select a losing assignment |
| Run all conversions, including operators, before selecting | A wrong guess invokes unused operators; effects make retry surprising | Rust coercion sites do not authorize speculative JavaScript calls | Repeated calls, allocations and invalidation harm IC stability | Cannot undo effects or preserve the selected-operation rule | Rejected independently of implementation effort |
| Retain pure scalar results or exact failure facts; classify effectful candidates without executing them | Range giveback works; a selected operator can still throw without hidden retries; completion can explain scalar alternatives | Rust supplies the destination-before-conversion analogy only; the greedy selection is this proposal's rule | Pure checks can be folded/shared; only selected calls require effect barriers | No discarded user effects or stale-reference authority; every selected result is checked | **Recommended** |

The interpreter currently retains candidate results for distinct declared scalar records. A production implementation may retain a pure conversion recipe or range fact instead. This is not permission to call getters, metadata hooks, constructors or user conversion operators while searching. If no assignment is viable, propagate the first retained pure scalar conversion failure in input-index and written-run order, or TypeError if none exists. Once a selected effectful conversion begins, its failure is propagated without choosing another assignment.

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

## Costs, proof limits and next work

A scalar pattern stores written runs, not a current partition or expanded finite count. This interpreter can scan current owned data and allocate candidate rows on a guarded store; memoized assignment tracks the other guards' run states as well as the first pattern's run and input index. There is no claim that this implementation meets V8 performance targets. A real engine can share declaration descriptors, guard store ICs on those descriptors and maintain invalidation-aware run summaries. Repeated guards and long alternating patterns still need broader complexity and resource-accounting work.

The new conformance corpus tests desired outcomes rather than preserving old failures. The delivery report records exact build, paired-baseline, bounded regression, typecheck and patch-application evidence. This document deliberately does not mark general contextual literal distribution, fixed/finite heterogeneous conversion, object/metadata effects, recursive inference, portable evaluation budgets or full regression completion as done.

Prior art used above: [Rust mutable-reference variance](https://doc.rust-lang.org/reference/subtyping.html), [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html), and [ECMAScript reverse property-operation order](https://tc39.es/ecma262/multipage/indexed-collections.html#sec-array.prototype.reverse). The recommendations are inferences for this proposal; none of these sources implements its runtime rest guards.
