# Object-valued rests: prototype and descendant dependencies

General object-valued conversion planning remains incomplete; the bounded ordinary constructor extension below now supplies selected plans. Ordinary nominal tuple membership is now backed by the correlated prototype mechanism below. Recognizing an intrinsic numeric payload represented by an interpreter Object does not establish support for a nominal or structural Object destination. The complete heterogeneous-rest design remains the recommendation; this note specifies its next proof obligations rather than changing class membership.

## Original reproduced gaps

At engine262 commit `5f92785d37ecb497838c66e8b301188ede1a2677`, this converting boundary invokes the constructor twice and returns a sequence that is not a member of its annotation:

```js
let calls = 0;
class C { constructor(value: any) { calls++; } }
type T = [...[].<C>, ...[1].<string>, ...[].<boolean>];
const source: any = [1, "s"];
const admitted: T = source;
const result: any = admitted;
calls + "/" + String(result is T); // observed "2/false"
```

The required String cannot be consumed by the first rest. Removing its required count can legitimately let the greedy conversion select two C constructions, so constructor count alone is not the criterion.

Even a constructor-free successful admission needs more than retained Object identities:

```js
class C {}
type T = [...[].<C>, ...[].<string>];
const child: any = new C();
const source: any = [child, "s"];
const admitted: T = source;
Object.setPrototypeOf(child, null);
source is T; // observed false
```

The current nominal membership operation follows the object's prototype chain to the constructor's prototype. The example mutates that dependency without replacing either array element. A retained class Type Record or an earlier successful `is` is not evidence that this chain remains valid. These observations are discovery evidence, not accepted conformance outcomes. Neither defect is repaired by the numeric-source planning increment.

## Decision: evidence for an object role

| Direction | Ergonomics, completion and wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Treat Object identity or a carried class record as permanent membership | A nominal completion claim survives a prototype change even though `is` now disagrees; structural properties can likewise change behind an alias | Rust nominal identity does not depend on mutable JavaScript prototypes; importing that result without its model is invalid | Cheap cache, but grants unsound optimized type facts | Violates current membership and retained-descendant requirements; identity is not a deep snapshot | Rejected |
| Change all nominal membership to a permanent construction brand | Stable nominal completion becomes possible, but fabricated prototype instances and prototype changes receive new semantics | Rust's construction identity is closer, but this is a different language-wide membership rule | Potentially cheap brand checks and stable ICs | Could be coherent after a separate full design change; does not solve structural properties or metadata dependencies | Rejected as a local rest repair; no authorization from the existing prototype-chain rule |
| Freeze or copy every admitted object and its dependencies | Aliases unexpectedly become immutable or receive different objects; autocomplete cannot convey this hidden ownership change | Rust ownership is not this proposal's freely aliased checked storage; freezing a prototype graph is not a Rust borrow | Deep traversal, allocations and altered shapes; a shallow copy leaves shared dependencies | Changes committed identity and mutation behavior; metadata effects still need their own rule | Rejected as the general solution, independently of effort |
| Preserve selected backed obligations and track any remaining membership dependencies through mutation publication | Completion retains declared contracts; a mutation may fail when it would break an acquired obligation, while unrelated changes remain possible | Rust mutable-reference invariance motivates stable place contracts only; ECMAScript supplies prototype mutation behavior, not correlated guards | Share declaration descriptors and dependency summaries; prototype/property mutation invalidates relevant summaries and IC assumptions; dependency links must not retain dead owners | Supports current membership, all acquired guards and independent reference identities; requires coverage of every dependency and alias route | **Recommended** |

This is an inference from the existing proposal. [Rust's variance rules](https://doc.rust-lang.org/reference/subtyping.html) do not establish a JavaScript ownership discipline. [ECMAScript OrdinarySetPrototypeOf](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinarysetprototypeof) permits ordinary prototype replacement under its own checks; adding an acquired type obligation requires a specified prepublication check, not a retrospective rollback.

## General implementation boundary

The ordinary exact-class and tuple-membership mechanisms below identify fixed constructor/prototype dependencies, including inherited links. General nominal conversion and exotic backing must additionally distinguish their applicability and persistent evidence from observable Proxy or constructor-property operations. A prospective prototype change must leave all affected acquired sequence and child obligations valid before publication. A valid changed chain must establish dependencies on its new links. Replacement/removal must retire obsolete relationships; weak dependency handling must avoid retaining dead arrays through surviving children. Independently completed nested boundaries remain in force.

Conversion planning must reserve finite runs without invoking constructors or operators while searching. Constructor applicability, selected execution, and successful backed result are distinct facts. Retain operation observations; execute only the selected conversions in index order; propagate a selected failure without retry; revalidate earlier results and relevant backing after later user effects. General constructor signatures, specialization, proxy membership and metadata cannot be replaced by a blanket one-parameter constructor test.

Structural roles additionally need the selected property/callable/default obligations and their alias relationships. Do not infer a child's writable property contract merely from its observed value or silently broaden existing backing. Establish which evidence remains valid under child writes and which dependency must be checked. Metadata judgments require their existing effect discipline; unchanged ordinary properties are not a permanent cache key.

Validation must cover the two witnesses above, inherited prototype mutations, valid changes, unrelated changes, multiple parents and guards, replacement/removal, constructor failure and reentrancy, Proxy routes, and invariant references with independent removal/relocation liveness. Until that coherent domain is implemented and specified, keep these object destinations out of the pure scalar specialization. This is a staged implementation boundary, not a recommendation to reject all heterogeneous object tuples in the finished language.

## Implemented prerequisite: exact ordinary class boundaries

An exact receiving operation for a resolved, non-generic source class now retains the class prototype it requires. This includes ordinary checked and explicit conversions, parameter and return boundaries that perform them, and the corresponding elided annotation path. The target constructor must have a nonwritable, nonconfigurable own prototype identity; the instance's consulted prototype prefix must be ordinary. A completed conversion cannot publish an ordinary result whose chain misses that target, including when a converting constructor returned an unrelated object. Earlier constructor effects and abrupt completions retain their order.

```js
class C {}
const source: any = Object.create(C.prototype);
const admitted: C = source;
Reflect.setPrototypeOf(source, null); // false, before changing the link
source is C;                        // true
```

An exact class contract is retained on the received value, like an acquired storage obligation. It is not released merely because one binding goes out of scope or an alias changes: other typed aliases may still rely on it. Additional exact class admissions are conjunctive. This does not replace membership with a permanent brand; membership still follows the prototype chain, and an unadmitted object can still change that chain.

### Decision: retaining an exact class promise

| Direction | Ergonomics, completion and wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Check only when the value next crosses another boundary | Completion promises C inside the current body, but an alias can invalidate it before a use | Rust's invariant mutable references motivate stable contracts, not deferred checks after invalidation | Cheap initial admission, unsound optimized class facts between checks | Does not preserve an already acquired exact contract | Rejected |
| Release the obligation when its originating binding is overwritten or exits | Prototype changes depend on which alias originally received C; an escaped alias can retain the same promise | Rust lifetimes track borrowing under a separate ownership discipline; this proposal has gradual aliases without that release protocol | Requires alias/lifetime accounting; incorrect release invalidates IC assumptions | A single binding's lifetime cannot prove no remaining reliance; release after a complete proof could be an invisible optimization | Rejected as an observable release rule |
| Freeze the entire prototype chain | Simple failures, but inserting a harmless intermediate link or changing links beyond C is unexpectedly prohibited | Rust has no mutable JavaScript prototype chain to freeze | Simple guards but changes extensibility and restricts useful shapes | Safe but stronger than preserving C; does not justify changing reference lifetimes | Rejected |
| Retain exact reached-prototype obligations while allowing valid chain changes | Completion's C promise survives; a wrong change fails at the mutation, while valid replacements remain possible | Rust supplies the stable-place analogy only; ECMAScript supplies ordinary mutation ordering | Reverse dependency links target affected instances; stable class contracts permit IC guarding without per-binding shapes | Preserves all acquired exact contracts and leaves allocation/removal reference rules independent | **Recommended**, independently of implementation effort |

The dependency prefix includes the instance and each intermediate object whose outgoing link is needed to reach the target prototype. It excludes the target's outgoing link. Replacing a prefix with another valid one updates the dependencies; obsolete links immediately stop constraining that instance. Shared links remain constrained by other live instances. Weak instance/dependency links and cleanup prevent a surviving class prototype from retaining all admitted instances. The interpreter uses host weak references and finalization for metadata cleanup. This does not promise prompt release after an instance becomes unreachable: a surviving intermediate prototype can remain constrained until the weak instance is cleared. Programs must not rely on collection occurring before a prototype mutation. Finalizer records hold the target prototype weakly too, since a user property on that target can point back to the instance.

### Decision: retiring unreachable instance dependencies

| Direction | Ergonomics, completion and wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Strongly retain every dependent instance through its surviving prototypes | Mutation results stay predictable, but a class prototype silently keeps all admitted instances alive | Unlike Rust Weak, which does not retain its payload | Retains entire instance graphs and IC dependency metadata | Preserves contracts but violates the weak ownership requirement | Rejected |
| Keep permanent restrictions on links after instances disappear | A link can remain unusable forever although no remaining instance needs it | Neither Rust borrowing nor ECMAScript ordinary prototypes impose this permanent history | Small summaries can still retain target graphs; unrelated future shapes stay restricted | Safe but stronger than retaining actual acquired value obligations | Rejected |
| Require exact reachability discovery before every affected mutation | Would release irrelevant restrictions promptly, but success now requires a global liveness decision | Rust Rc/Weak destruction relies on reference counting and a different cycle/ownership model; it does not solve JavaScript reachability | Global tracing on prototype mutation defeats targeted dependency checking; semantic liveness can require conservative approximation | Reference counting alone misses cycles, and binding exit misses escaped aliases | Rejected as a portable mutation requirement, independently of implementation effort |
| Retain obligations until weak instance references clear; perform no user cleanup callbacks | Live promises remain stable; a shared prototype may still refuse after the last apparent use, so collection cannot be used as a mutation protocol | ECMAScript weak references permit delayed collection; Rust Weak is a limited non-owning analogy, not a matching release schedule | Collector-integrated weak edges and targeted cleanup avoid retaining instance graphs or changing instance shapes | Never releases a potentially observable instance's contract; storage-reference removal rules remain independent | **Recommended**, with the timing limitation explicit |

This decision intentionally permits a prototype mutation result to reflect delayed reclamation of an otherwise unreachable dependent. It does not permit dropping a contract while the instance can still be observed again. [ECMAScript's weak-target processing model](https://tc39.es/ecma262/multipage/managing-memory.html#sec-processing-model-of-weakref-and-finalizationregistry-targets) supplies the delayed-clearance analogy; [Rust Weak](https://doc.rust-lang.org/std/rc/struct.Weak.html) supplies only the non-owning relationship. Tests keep relevant instances observable and do not assert a collection deadline. The host finalizer runs metadata cleanup only, never interpreted program code.

### Decision: publishing a prototype change

| Direction | Ergonomics, completion and wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Change the link, then validate and roll back | A wrong change can expose a broken C promise or transiently invalidate dependent code | ECMAScript has an ordinary link publication, not a transaction; Rust is not a rollback model | Requires restoration and additional invalidation work | Validation after publication is too late; unrelated effects cannot generally be rolled back | Rejected |
| Run arbitrary prototype getters/traps while checking the prospective chain | Failure can invoke unrelated user code even when no link is written | ECMAScript's ordinary mutation checks do not justify extra prototype-hook calls; Rust supplies no such protocol | Reentrant calls invalidate plans and IC assumptions | A Proxy's current answer is not persistent evidence of its future answers | Rejected for this ordinary specialization; general exotic backing needs separate semantics |
| Check every affected ordinary dependency before writing, then update links without user code | Reflect returns false and Object.setPrototypeOf throws for a refused change; valid changes retain ordinary behavior | Follows ECMAScript's Boolean internal result and ordinary abrupt handling; Rust remains a limited invariant-place analogy | Check only dependent prefixes; commit changes once and update targeted reverse links | No invalid prototype is published; no reference is removed, relocated or retargeted | **Recommended** |

Keep the ordinary same-prototype, nonextensible and cycle checks in their existing order. A new Proxy/exotic link in a required prefix cannot provide this specialization's ordinary persistent proof and is refused without executing its prototype trap. Forwarding Proxy aliases of an already guarded instance reach the same target check; a trap that reports success without changing the target has not invalidated its membership. Independently completed class boundaries survive later throws.

### Boundary of this milestone

Pure `is` does not acquire an obligation. Whole membership in `C | D` does not acquire an exact C obligation just because C was the successful alternative. Likewise, this helper is not permission to turn a heterogeneous rest's successful partition into permanent exact child contracts. At the exact-class prerequisite milestone, both original heterogeneous-rest witnesses remained open. The subsequent ordinary tuple-membership extension below closes the prototype invalidation examples; constructor planning remains open. An independently selected conversion to C is an exact receiving operation and completes its own obligation before a later enclosing failure.

General composite descendant acquisition, generic class specialization, unresolved or replaceable constructor prototypes, initial Proxy/exotic chains and metadata remain separate coverage. The next section covers correlated ordinary nominal tuple owners. Class construction without an exact receiving operation is not itself generalized by this milestone. Copies must acquire any required obligation on the actual published result; broader copy/adoption transport needs continued audit. Weak dependency metadata does not close portable evaluation-budget or cache-accounting gates.

## Implemented extension: ordinary nominal tuple membership

Successful owned-Array membership admission now retains an ordered guard for tuple patterns containing ordinary identity classes and the supported unparameterized scalar types. Required positions, finite runs and dynamic runs keep their counts; zero contributes no role. Nominal targets must be resolved, non-generic source classes with the permanent prototype identity described above. Value classes, metadata, unions used as element types, defaulted positions, unknown counts and initially exotic nominal evidence remain outside this extension. Stable fixed/prefix/suffix tuple shapes keep their declared location contracts and acquire the same nominal child protection.

```js
class C {}
class D {}
type T = [...[].<C>, ...[].<D>];
const child: any = new C();
const source: any = [child];
const received: T = source;
Reflect.setPrototypeOf(child, D.prototype); // true: the whole tuple still fits
Reflect.setPrototypeOf(child, null);        // false: no valid role remains
```

For `[a, b]` initially containing two C instances, changing only `a` to D is refused when `b` still requires C: `[D, C]` violates the ordering. Changing `b` to D succeeds. Thus this is a sequence promise, not an independently writable element union or an exact class stamp on each child.

### Decision: protecting nominal roles without fixing a partition

| Direction | Ergonomics, completion and wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Acquire an exact permanent class obligation for each current role | Completion's declared alternatives no longer explain why a valid class-role change fails; removed children remain pinned | Rust nominal identity is not JavaScript prototype membership; Rust borrowing does not choose an observed tuple partition | Cheap exact guards, but retains stronger instance facts than the declaration promised | Prevents valid repartition and conflates owner membership with independently completed class boundaries | Rejected |
| Recheck only on later reads or after the prototype write | A mutation can invalidate a currently held tuple promise before failure is reported | Neither Rust stable-place reasoning nor ECMAScript property ordering supplies retrospective repair | Late checks weaken optimized facts and can require rollback or deoptimization after exposure | Can publish invalid state and cannot undo effects or repair reference authority | Rejected |
| Scan every acquired array on every prototype change | Correct results are possible, but unrelated arrays impose mutation latency | Rust supplies no global JavaScript alias scan; ECMAScript prototype operations have no such scan | Global owner traversal instead of targeted IC/dependency invalidation; weak registry maintenance still required | Can preserve contracts if all owners are visited; no correctness advantage over complete reverse dependencies | Rejected as the production design, independently of implementation effort |
| Track relevant ordinary prototype paths and recheck each affected owner's whole ordered guard before publication | Completion retains declared alternatives; a wrong order fails at mutation, while valid repartition and removal work | Rust's invariant-place analogy applies to location contracts only; the correlated runtime guard is this proposal's addition | Shared run descriptors, targeted weak reverse edges, no per-partition object shapes; a check can still cost sequence and prototype-depth work | Preserves every acquired owner guard and independent exact class obligation; references keep their declaration-derived type and ordinary removal liveness | **Recommended** |

The guard checks all acquired patterns against the same prospective values. A child may satisfy several classes, and two owners may use it differently. Matching one class in one successful assignment is not a permanent choice. Each affected owner re-evaluates its complete pattern using ordinary internal links with the proposed replacement substituted. These checks invoke no getters, conversion operators, constructors or Proxy traps. If no ordinary witness remains, the mutation fails before publishing the link or changing live dependencies. A valid change installs the new dependencies and retires obsolete ones. Negative roles do not need independent watchers: losing an unused possibility cannot invalidate the retained accepting witness; any change to that witness triggers a fresh whole-pattern check.

Element replacement and successful length publication refresh dependencies from actual owned data. Removing a child retires that owner's relationship, including when a failed truncation removes only a suffix. Another occurrence, another owner or an independently completed exact C boundary can still constrain it. Newly inserted or reflectively replaced children acquire the owner relationship without a hidden narrow class stamp at a broad projected location. Non-index property changes require no sequence scan; staged growth attaches new child dependencies at length publication rather than rescanning once per unpublished tail slot. No user code runs between these publications and dependency updates.

Admission revalidates retained values and their current ordinary roles before publishing a new guard. A prior getter can change a prototype; retaining the same Object identity does not validate the earlier class role. During later effectful scalar stores, the owner's guards already protect its published state, and the prospective converted sequence is rechecked after effects. A failed outer check preserves valid earlier effects and independently completed guards. Pure `is` acquires nothing.

Owner links and dependency paths are weak. The inspector is an engine operation that reads the currently retained owner; its finalizer record captures no array, element or constructor graph. The previous weak-cleanup timing decision applies to unreachable owners too. This does not change language-level weak-target eligibility or reference removal/relocation rules.

### Remaining boundary

The original direct and inherited nominal-rest prototype witnesses are repaired for this ordinary family. At the membership milestone, nominal constructor planning remained open and the required-String witness still called C twice and returned a nonmember. The following extension repairs that witness and supports selected simple constructors at converting boundaries and stores. Nominal defaults and general constructor/operator applicability remain separate work.

Initial Proxy/array-like acquisition, value-class copy transport, metadata judgments, general structural descendants, custom species and contextual fresh-literal transport still need their own proof. Forwarding aliases of an already guarded child or indexed Array store reach the same target guard. Proxy-receiver intrinsic removal remains a separate existing limitation: a forwarding Proxy's `pop` can reach the direct typed-element deletion refusal instead of the owned-array removal plan. Portable evaluation and cache-accounting gates remain open.

## Implemented extension: selected ordinary constructors

The required-String witness now returns `1/true`: one C construction, the retained String, and a result that satisfies T. A complete unchanged sequence still wins before conversion planning. Otherwise, declared counts reserve every required run before any constructor executes. Making the String run optional can legitimately select more C conversions under the existing leftmost-greedy rule.

### Decision: constructor applicability during rest assignment

| Direction | Ergonomics, completion and wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Treat every one-parameter constructor as applicable | Completion cannot explain why a Boolean parameter captures a Number that a later String run could accept; a wrong guess can execute unrelated code | Rust conversion implementations have source types, not just arity; its From/TryFrom distinction does not authorize ignoring a parameter | Cheap classification but bad plans cause unnecessary calls, allocation and unstable result shapes | Does not establish parameter admission and can select the wrong assignment | Rejected |
| Execute constructors while searching and retry after failure | Backtracking repeats user effects and can swallow the intended error; completion exposes no retry protocol | Rust TryFrom reports a selected conversion's failure; it does not specify rollback or implicit alternative search | Repeated allocation and arbitrary user calls defeat a bounded candidate phase | Earlier effects and completed boundaries cannot be rolled back; retained references may have changed liveness | Rejected |
| Convert using the first rest and check the whole result afterwards | Invalid results are caught, but a valid required-String assignment still fails after extra constructor effects | Neither Rust's typed conversion traits nor ECMAScript call ordering supplies this count-blind rule | Avoids planning at the cost of unnecessary allocation and avoidable throws | Final validation is necessary but does not repair incorrect selection or invocation count | Rejected as the selection algorithm |
| Use declaration-resolved parameter evidence, reserve counts, then execute the selected plan once and revalidate | Completion follows declared alternatives; a selected constructor's error propagates with its ordinary effects; wrong guesses fail at a defined boundary | Rust supplies a useful separation between source compatibility and fallible execution; this proposal additionally defines runtime greedy sequence assignment and mutable prototype obligations | Share immutable signature/run descriptors; keep per-operation candidate data, without speculative allocation or a shape per partition; invalidate only relevant membership dependencies | Preserves required counts, selected-error precedence, backing, exact conversion obligations, and independent reference authority/liveness | **Recommended** |

[Rust From](https://doc.rust-lang.org/std/convert/trait.From.html) is intended for infallible conversions and points fallible conversions to TryFrom. This proposal's converting constructors can throw, so neither trait provides its implicit greedy ranking. [Rust variance](https://doc.rust-lang.org/reference/subtyping.html) remains a limited analogy for stable mutable-place types. ECMAScript's existing call and prototype publication order remains relevant; the runtime ordered guard is an additional proposal rule.

The implemented applicability proof is deliberately bounded. The destination is an ordinary identity class in the membership family above. It declares exactly one simple binding parameter with no optional marker, default, rest, reference, or destructuring. An unannotated or resolved any parameter is admissible. A resolved primitive parameter may admit an existing member or an intrinsically pure checked conversion; an ordinary identity-class parameter may admit an existing member. Read the parameter's retained declaration-context Type Record. Do not resolve its spelling in the caller's scope or infer any from an unavailable signature. Generic destinations, metadata/structural/union parameter checks, effectful parameter conversions, constructor defaults and general source/inbound operators are not classified by this proof.

A rejected pure parameter conversion supplies a retained failure fact, in the same input/run order as scalar failures. It does not preempt a viable complete assignment. A selected constructor executes once in input order; its actual parameter boundary remains checked even when the enclosing tuple uses an explicit cast. A selected throw propagates without trying another class or partition. A returned value outside the selected ordinary class throws TypeError. A successful selected conversion establishes its own exact class obligation, which survives later enclosing failure or removal from an array. Existing unchanged children acquire only the correlated owner guard.

After all selected operations, revalidate the complete sequence against every pattern. Later effects may validly repartition an earlier unchanged child, or may invalidate it despite retaining its identity. Refuse an invalid result before publishing the new Array and attach ordinary nominal dependencies before publishing a successful one. Source Gets remain operation-local retained observations. Stores also revalidate the consulted owner, length and backing after effects and preserve independent writes on failure.

Only the value supplied by the current insertion write can convert. A previously prepared private-tail entry, including a relocated element or an earlier inserted value, is already fixed for this write. Searching a later insertion must not reconstruct it or execute discarded conversion candidates. Movement retains SameType location checks and value identity; geometric failures still precede construction. Reflective data definitions use the same selected boundary conversion rules as existing scalar storage.

### Remaining boundary after constructor selection

The required-String witness and the ordinary prototype witnesses are now repaired. This extension is not general heterogeneous conversion completion. Next, extend the plan representation to source and inbound operators, effectful/composite parameter admission, defaults and generic/captured specialization while preserving operation identity, declared precedence and single execution. Metadata, value-class copying, general structural descendants, contextual fresh literals, custom species and initially exotic storage retain the previously recorded obligations. Proxy-receiver intrinsic removal and portable limits/cache accounting remain open.
