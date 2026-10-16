# Object-valued rests: prototype and descendant dependencies

Object-valued destinations remain outside the implemented scalar-rest specialization. Recognizing an intrinsic numeric payload represented by an interpreter Object does not establish support for a nominal or structural Object destination. The complete heterogeneous-rest design remains the recommendation; this note specifies its next proof obligations rather than changing class membership.

## Reproduced gaps

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

## Next implementation boundary

Before admitting nominal rests, identify the exact constructor/prototype dependency, including inherited links, and distinguish ordinary internal evidence from observable Proxy or constructor-property operations. A prospective prototype change must leave all affected acquired sequence and child obligations valid before publication. A valid changed chain must establish dependencies on its new links. Replacement/removal must retire obsolete relationships; weak dependency handling must avoid retaining dead arrays through surviving children. Independently completed nested boundaries remain in force.

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

Pure `is` does not acquire an obligation. Whole membership in `C | D` does not acquire an exact C obligation just because C was the successful alternative. Likewise, this helper is not permission to turn a heterogeneous rest's successful partition into permanent exact child contracts. The two original heterogeneous-rest witnesses above remain open; their whole-sequence admission does not yet use this mechanism. An independently selected conversion to C is an exact receiving operation and completes its own obligation before a later enclosing failure.

General composite descendant acquisition, correlated owner roles, generic class specialization, unresolved or replaceable constructor prototypes, initial Proxy/exotic chains and metadata remain separate coverage. Class construction without an exact receiving operation is not itself generalized by this milestone. Copies must acquire any required obligation on the actual published result; broader copy/adoption transport needs continued audit. Weak dependency metadata does not close portable evaluation-budget or cache-accounting gates.
