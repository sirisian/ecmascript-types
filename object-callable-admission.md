# Callable admission in object properties

A receiving object contract must retain the specialization required by a concrete callable property, even when the original generic function passes signature membership:

```js
function id<T: type>(x: T): T { return x; }
const raw: any = { f: id };
const view: { f: (x: string) => string } = raw;
view === raw;                 // true
raw.f === id.<string>;        // true
```

This is an in-place property conversion. It preserves the container, prototype, undeclared properties and aliases. A readonly receiving view does not make another alias readonly or grant permission to replace a non-writable property. The existing backing contract remains authoritative: a slot promising a generic callable cannot be overwritten with a monomorphic function that no longer satisfies that promise.

For the successful-membership path, retain each admitted property's object, key, value, resolved receiving contract and selected union member. Retain evidence from the selected union alternative and selected sequence assignment only. Complete membership before materializing the retained callables. Materialize in admission order, then use ordinary `Set` only when the value changes. Do not replay `Get`, redefine an accessor as a data property, or copy the container. Setters and proxy traps retain their normal effects and exceptions. A false write result produces the existing in-place-conversion TypeError. An unchanged value requires no write, including on frozen objects and getter-only properties.

Requirements reaching the same object/key apply together. Canonicalize their intersection for materialization and for a newly acquired backing contract. Keep each retained observation in admission order. Consecutive equal observations of that location share one materialization/write; a different intervening observed value makes a later observation distinct again. This coalesces duplicate evidence without running a setter once for each equivalent view. It does not coalesce different storage locations or manufacture an overload group from alternative union members.

Publish this admission's new property contracts only after its required materializations and writes succeed. On failure the receiving binding is not replaced, and no new contracts from that admission are published. Earlier completed writes and setter effects remain; this is not a transaction. Existing contracts and independently completed nested boundaries remain in force. Retaining observations does not freeze the object against ordinary getter, setter or declaration-check effects.

Pure membership and reference/overload viability checks neither specialize nor write. A backing contract proving membership does not alone prove that a concrete callable body has been retained. Elision requires that additional fact; a broad or generic backing contract may allow another alias to reintroduce a generic callable. No admission evidence grants a new reference-liveness permission.

## Decision: how the converted property reaches storage

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Keep the compatible generic function | Autocomplete promises a concrete result but an erased call can infer a different one | Rust separates instantiated function items from function-pointer coercion | Cheap admission, repeated inference and less stable call targets | Violates concrete callable crossing | Rejected |
| Specialize during membership | A Boolean `is` test can execute `where` checks or setters | Rust type compatibility does not perform mutable-container writes | Effectful speculative checks cannot be freely cached | Rejected union/sequence alternatives can change storage; viability probes become writes | Rejected |
| Rerun ordinary conversion after membership | A getter may run twice and return a different function | Rust coercion propagation does not justify replaying JS accessors | Duplicate property/IC work and checks | Converts a different observation from the admitted one | Rejected |
| Copy or wrap the object | Aliases stop agreeing; identity depends on the target property type | Rust aggregate coercion sites concern construction; they do not authorize copying an aliased JS object | Allocations, changed shapes or wrapper dispatch | Breaks established in-place conversion and reference locations | Rejected |
| Retain admission values; specialize and use ordinary Set | Concrete member behavior survives erasure; write failures identify the actual boundary | Rust's field/array/tuple coercion sites support propagating expectations, with no analogy for JS setter effects | Transient evidence plus existing specialization cache; no wrapper shape or per-call dispatch | Preserves identity, backing contracts, ordinary writes and liveness rules | **Selected** |

## Decision: write capability and failure precedence

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Preflight descriptors, then bypass Set | Predicts incorrectly for inherited accessors and proxies | Rust's static mutability is not a proxy-trap model | Extra descriptor walks and invalidation dependencies | Bypasses setters and backing-store checks | Rejected |
| Always write, even when identity is unchanged | Re-admitting a frozen compatible object unexpectedly fails | Rust has no requirement to reassign an unchanged JS property | Extra IC/setter activity | Adds unnecessary observable effects | Rejected |
| Materialize first; Set only changed values; preserve normal failure | A failed `where` precedes a write attempt; unchanged frozen values still pass | Existing JS conversion and property operations are the close analogy; Rust contributes no dynamic order rule | One required write; stable descriptor/IC machinery | Honors inherited setters, proxies, readonly storage and the existing backing type | **Selected** |
| Roll back all earlier writes if a later step fails | Suggests atomicity that arbitrary setters cannot provide | Rust ownership does not undo external effects or panic side effects | Requires logging and often impossible compensating operations | Contradicts existing conversion effects and cannot restore arbitrary user code | Rejected; preserve earlier completed effects |

## Decision: overlapping paths to one location

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| First or last receiving requirement wins | A path silently invalidates another path's promised call signature | Rust does not merge aliased mutable obligations by traversal order | Simple but incorrect | Loses an admitted requirement and can replace a backing contract | Rejected |
| Allocate a specialization per view or copy the shared child | Views no longer share the same property | Rust's aliasing restrictions cannot be imported as JS identity changes | Wrappers/copies and polymorphic calls | Breaks the shared object/key and reference semantics | Rejected |
| Combine requirements and retain observation order | Equivalent paths agree; genuinely combined call signatures remain a combined obligation | Rust supplies no structural-overlap algorithm; the proposal's existing intersection algebra is controlling | A transient object/key grouping; implementations may fuse statically proved groups | No last-writer choice of contract; existing backing type still wins | **Selected** |
| Repeat Set for equal evidence from every equivalent path | Adding an alias changes setter count although its required value is identical | Rust coercion sites do not require duplicate JS writes | Redundant setter/IC work | Introduces effects solely to replay the same acquired obligation | Rejected; coalesce equal consecutive observations of that location |

## Decision: check elision

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Treat any compatible backing type as complete admission | A typed alias can skip the body selection required by its member | Rust compatibility and coercion are distinct | Cheap fast path with wrong semantics | A generic or broad backing contract can admit unmaterialized values | Rejected |
| Disable every structural fast path | Repeated crossings can introduce unnecessary getter effects | Rust does not require rerunning proved coercions | Unnecessary shape walks and IC traffic | Loses established no-evaluation behavior when admission is already complete | Rejected |
| Elide only with membership, storage and materialization established | A concrete view behaves consistently across checked and erased paths | Rust's distinction supports the proof obligation, not the JS runtime mechanism | Engines can retain type/shape facts; this interpreter conservatively examines type records without property Get | Preserves contract acquisition and body selection; no new borrow permission | **Selected** |

Primary sources: [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html#coercion-sites), [Rust function item types](https://doc.rust-lang.org/reference/types/function-item.html), and the proposal's existing in-place conversion, typed-storage, union-selection and reference-liveness clauses. Rust's expected-type propagation is a limited analogy: JavaScript aliases, accessors, proxies and runtime `where` checks require the explicit decisions above. Performance assessments are design analysis, not benchmark measurements.

## Implementation boundary and next work

This increment covers named and index-signature object properties, inherited properties, nested structural objects and object properties reached through admitted arrays/tuples. It retains union-member choices for each property. The existing single-source/single-concrete-signature materializer is reused, including source/native specialization identity and captured declaration context.

An intersection requiring several callable signatures is still an open materializer capability. This increment groups that obligation instead of destructively choosing its first or last signature; the current interpreter can still leave the generic function in that case. This is an implementation gap, not permission for the specification to omit required specialization.

Direct homogeneous-array elements now use retained admission evidence; see [array callable admission](array-callable-admission.md). Direct tuple elements remain open because their selected sequence assignment must agree with later mutations and reference checks. Combined callable signatures remain a separate open obligation.
