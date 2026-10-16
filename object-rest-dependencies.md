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
