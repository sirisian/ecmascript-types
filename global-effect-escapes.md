# Global effects through stored and returned values

October 31, 2026. Sixth implementation increment of the [eval and stable names plan](eval-name-stability-plan.md), continuing milestone 5. This extends [global callable origins](global-call-origins.md) to stored values, source callable returns and exposed containers. The [implementation record](global-effect-escapes-implementation.md) gives validation, patch bases and remaining gaps.

## Problem and direction

```js
function f(): string { return "old"; }
const box = { world: globalThis };
box.world.f = () => 1;
const n: number = f();
```

The earlier alias inventory recognizes `const world = globalThis` but loses the same value when it is stored in `box.world`. The old declaration's string result then produces a false Early Error. Returning that alias from a source function, getter or callback creates related routes. None changes the actual storage being written.

**Recommendation:** extend the mutation footprint with a finite value-flow graph that distinguishes allocation sites and property names. Use it only to discover possible effects and withdraw stale declaration-result assumptions. Keep three different facts separate: a container's identity, a value stored inside it, and the authority that invoking or exposing that value may give code. Preserve the existing independent-contract and runtime-enforcement rules.

In particular, `box.f = ...` is not `globalThis.f = ...` merely because `box.world` holds the global object. A broad label that merges those objects would repair the motivating case at the cost of unnecessary changes to ordinary code's error phase. Autocomplete should keep the relevant declared API information without turning uncertain current-value evidence into an unconditional result promise.

## Validity requirements

The previous seven [requirements](global-call-origins.md#requirements-for-a-valid-solution) still apply. This extension adds four tests of a proposed solution:

1. Distinguish two separate containers with the same property names, and two different known properties or indices of one container.
2. Follow writes through aliases of the same container, including supported reflective writes, without assuming that an overwritten slot has never held its earlier value.
3. Connect supported arguments, receivers, returned values and captures without running a function, getter, constructor or dynamic string during checking.
4. Terminate on cyclic storage and recursive callable relationships. Exhausted analysis must not manufacture proof that a global binding was unaffected.

## Decision 1: how should stored aliases be represented?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Stop at binding-to-binding aliases | A harmless refactor into an object changes acceptance | Rust distinguishes a reference value from its storage; ECMAScript property access plainly reloads the stored value | Cheapest scan | Leaves the reproduced false Early Error | Reject |
| Mark every container holding a global alias as the global object | Removes the example's false rejection, but makes `box.f` affect unrelated global `f` | Neither Rust places nor ECMAScript objects have this identity rule | Compact metadata, extensive unnecessary invalidation | Conflates storage and contained values | Reject |
| Merge all objects by property spelling | Two unrelated `{ world: ... }` objects contaminate each other's diagnostics | Rust place identity is a useful analogy, not a JavaScript heap analysis | Cheap field sets, poor specialization and precision | Fails the distinct-allocation requirement | Reject |
| Require exact runtime heap interpretation | Apparently precise completions, but checking can execute or depend on arbitrary behavior | Rust does not solve JavaScript's unrestricted heap and eval model this way | Unbounded heap/context analysis or actual execution is unsuitable for required checking | Getter/call execution violates phase and effect rules | Reject as a mandatory language rule |
| Use allocation-site and property-sensitive may-flow, with wildcard slots for unknown keys | Preserves useful distinctions; unknown keys deliberately reduce precision | ECMAScript value/storage semantics are direct; Rust supplies only the place/alias comparison | Private compiler graph and worklist, no runtime object fields or new shapes | Repairs supported storage routes without asserting exact runtime identity | **Recommend**, independently of implementation effort |

Object and array literal occurrences supply abstract allocation identities. Aliases flow to those identities. Named stores contribute to the matching slot; unknown-key stores contribute to a wildcard visible to every slot. Reads from unknown keys join the possible slot values. Values accumulate rather than being removed on an overwrite or delete. Cycles therefore describe possible reachability, not an instruction to recursively execute or copy an object.

The global object has a distinguished storage summary. Its property writes can feed subsequent global-object binding reads, subject to the existing lexical identity filter. A same-named Script lexical or module binding is not connected to that object property. Numeric property keys use their property-key spelling, so array index `1` is distinct from index `0`.

## Decision 2: how should calls, getters and exposure participate?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Ignore all returned values and callback forwarding | A trivial `id(globalThis)` or source getter loses alias evidence | A Rust function pointer's declared type is not knowledge of the returned JavaScript object | Very cheap, but leaves repeated blind spots | Fails supported value-flow cases | Reject |
| Execute a visible function/getter to discover its result | Checking triggers counters, exceptions and external effects | Neither cited language model authorizes speculative user-code execution here | Potentially unbounded runtime work during compilation | Violates effect ordering and no-execution requirements | Reject |
| Treat every function result and receiver as the global object | Precision disappears for ordinary factories and local objects | Rust callable types can be useful without exact function identity; that does not make every result the same object | Small analysis, broad loss of typed optimization and diagnostics | Conflates unknown values, global identity and independently established contracts | Reject |
| Connect visible callable parameters/receivers/returns; summarize exposure of reachable capabilities separately | Refactoring through source functions, methods and callbacks retains useful behavior. Opaque exposure remains conservative | ECMAScript supplies argument/receiver/return behavior. Rust's function-pointer distinction is limited by this proposal's shallow admission | A finite call/value graph can share lexical infrastructure; no wrapper or extra invocation | Adds possible effect evidence without proving a callable result contract or implementation | **Recommend** |

The graph joins candidate arguments into a source callable's parameter bindings, its receiver into applicable `this` reads, and return expressions into the call result. Concise arrows, ordinary returns, source object getters/methods, callbacks with available source candidates, `.call`, bound source calls and nonliteral `.apply` containers participate. No fixed small argument prefix is silently dropped for `.apply`.

A getter's source return flow is a possible returned value, not a proof that the getter runs, succeeds, is still installed, or has no other effects. Getter execution remains at the ordinary runtime property access. Tests count that access and require it to occur once. The existing conservative treatment of deferred-body writes remains in force.

Exposing a tracked aggregate can expose the global object or a dynamic-source callable nested inside it. This is a reachability summary, not aggregate identity: mutation of `box.f` alone is local, while passing `box` to opaque code can expose `box.world`. Explicitly exposed eval/Function/generated-function capabilities can reach mutable global lexicals; an object-property footprint alone does not replace a same-named lexical binding. The summary discovers potential effects; it neither grants reference authority nor extends a lifetime.

Supported reflective stores also feed the graph: direct `Reflect.set`, `Object.defineProperty`, `Object.defineProperties` and `Object.assign` routes. Aggregate spreads and some reflective copies deliberately join properties into a wildcard instead of claiming an exact copied layout. These routes are still subject to ordinary execution, exceptions and getters at runtime.

## Decision 3: how much temporal and call-context precision is required?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| One undifferentiated global hazard for the entire source | Unrelated local edits change all callable results | Rust places and ECMAScript captures do not imply this collapse | Cheap scan, lost specialization | Erases distinctions and risks repeating blanket contract loss | Reject |
| Fully version every heap slot and every call context as a mandatory rule | Best potential precision, but complex behavior around joins, re-entry and unknown code | Rust's borrowing guarantees are stronger; they do not transfer to arbitrary JavaScript aliases | More compiler memory, joins and contexts; recursive contexts require a finite abstraction anyway | Can be correct with a prescribed model, but is not an exact universal oracle | Do not require unbounded precision; bounded prescribed refinements remain eligible later |
| Use monotone, context-merged may-flow for this effect screen; retain the prior read-position/capture judgment | Predictable conservatism with explicit precision costs; ordinary distinct allocations and fields remain useful | ECMAScript capture order is the direct requirement; Rust offers no matching mandatory diagnostic rule | Finite candidates and reusable edges; avoids per-invocation heap snapshots | Conservative within the modeled routes and separate from receiving proof | **Recommend** as the current finite screen |

This recommendation is a performance and semantic-boundary choice, not an excuse based on development effort. It does sacrifice precision. For example, calling `id({})` and `id(globalThis)` joins both values in the same source callable's parameter/result summary. A later write through the first result can consequently withdraw global result evidence even though that particular runtime value is ordinary. A probe confirms that this changes an incompatible receiver's failure from an Early Error to runtime admission. This is not claimed to be exact context-sensitive inference.

The footprint still applies at the contributing callee read. An immutable copy captured before a later direct heap-mediated write retains its earlier evidence; a copy afterward cannot regain the old declaration result. Argument-side mutation does not replace the callee already obtained for that invocation. The existing whole-source rule for loops and deferred regions remains conservative. Value-flow joins do not authorize replaying a getter, argument or call.

Future prescribed flow/context precision must be compared against these costs and accompanied by diagnostic-phase tests. An optimizer may use stronger runtime guards independently; it cannot retroactively justify a mandatory Early Error. The longer-term positive static-evidence/independent-contract separation remains the design target.

## Decision 4: what happens with cycles, limits and prior-source values?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Recursively expand until the host stack fails | Ordinary cyclic objects can crash checking | Neither Rust's reference model nor ECMAScript object identity demands recursive copying | Unbounded time/stack | Fails termination | Reject |
| Treat a cycle or exhausted search as no possible origin | Apparently precise but stale diagnostics depend on analysis limits | A missing Rust lifetime/alias proof would not establish safety; the analogy stops there | Fast cutoff | Turns lack of evidence into a false negative-effect proof | Reject |
| Inspect arbitrary runtime getters, collection internals and generated bodies to recover precision | Can move user code and errors into checking | Rust interior mutability is explicitly controlled; this proposal does not acquire that guarantee from a readonly view | Unbounded inspection or extra user execution; awkward engine/host coupling | Violates the available-evidence boundary | Reject as the general rule |
| Use finite sets/worklist edges and conservative exhaustion; inspect only permitted initialized ordinary own data | Cyclic supported storage terminates; opaque source boundaries remain explicitly limited | ECMAScript descriptors/environments supply the direct model. Rust's alias guarantees are stronger and not imported | Bounded private analysis; no getters, wrappers or user-visible metadata | Exhaustion withdraws affected global assumptions rather than establishing absence; independent local contracts remain separate | **Recommend** |

The engine deduplicates graph edges and candidates. It bounds graph expansion depth and propagation work. If that analysis exhausts, the caller withdraws mutable global declaration-result assumptions across the source, including reads whose individual queries ran before exhaustion. It does not reinterpret a partially built graph as complete. These internal limits are not a promise that every other checker operation is stack-safe: the stress experiment described in the implementation record found a separate existing overflow.

Initialized earlier-source ordinary own-data values can seed aggregate summaries without invoking getters or exotic operations. Binding lookup retains the prior TDZ/import/reference barriers. This increment does not read hidden collection entries or execute an earlier getter or generated function to recover provenance.

## Coverage and remaining gate

The motivating heap-alias example now succeeds, as do supported nested records/arrays, aliases of containers, stores and reloads, patterns, spreads, reflective stores, returned aliases, source getters/methods, callbacks, bound calls and nonliteral apply arrays. Controls preserve distinct allocation/field/index identities, isolated local and global lexical names, independent contracts, immutable capture, receiving failure and single getter execution.

The counterexamples left by this increment, repaired by [opaque global effects](opaque-global-effects.md), require a precise source-boundary setup: create an opaque carrier in an earlier Script, then declare `f` in a later Script and replace it through that carrier. At this increment's baseline, probes reproduce false Early Errors with an earlier `Map` holding the global object, an earlier accessor returning it, and an earlier generated callable returning it. A generic claim that every use of these objects fails would be wrong: some same-source examples are already covered by exposure/dynamic-source footprints, and earlier declarations may have different available result evidence.

Arbitrary intrinsic callbacks, class/prototype paths, opaque imported/host values, all reflective/bound combinations, and other static consumers still need the positive-evidence audit. The graph is not a universal absence-of-effects proof. A missing edge cannot license runtime check elision, and preserving runtime checks does not excuse a false Early Error. Milestone 5 and the integrated milestone 6 gate remain open.

## Prior art and engine consequences

[Rust function pointers](https://doc.rust-lang.org/reference/types/function-pointer.html) separate a callable type from knowledge of a particular function. [Rust interior mutability](https://doc.rust-lang.org/reference/interior-mutability.html) also shows why guarantees about aliases depend on the language's storage rules. This proposal's ordinary JavaScript objects, shallow callable admission and runtime references supply different guarantees; a `const` or readonly view is not a Rust borrow.

[ECMAScript object initialization and property access](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-object-initializer) supply the actual values and effects. The graph is a proposal-specific static abstraction of possible routes, not a change to those operations. [V8's Maps and dependency discussion](https://v8.dev/docs/hidden-classes) motivates keeping compiler metadata separate from user objects: this analysis adds no fields, wrappers or shape transitions. Its effect on ICs is indirect, through which optimizations retain admission checks. Production engines can use existing graph/CFG infrastructure and guarded dependency mechanisms; that is an engineering inference, not a benchmark of this interpreter or a claim that V8 implements the proposal.
