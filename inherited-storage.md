# Inherited storage after admission

Successful structural admission can establish a property contract without creating an own property. That contract must survive ordinary mutations of the supplier and the lookup path.

```js
const proto: any = { a: 1 };
const raw: any = {};
Object.setPrototypeOf(raw, proto);
const out: { a: number } = raw;
proto.a = "bad"; // TypeError; proto.a remains 1
Object.setPrototypeOf(raw, { a: "bad" }); // TypeError; old prototype remains
```

[Pending admission](inherited-data-admission.md) validates retained observations before publication. This continuation retains the successfully established receiver/key contract after publication. Compatible changes remain possible: `proto.a = 2`, retargeting `raw` to ordinary data holding a number, and an ordinary own write through `raw.a` can all succeed.

## Enforcement decision

A valid direction preserves receiver identity and ordinary property keys, existing backing types, every dependent receiver's obligations, ordinary exception/effect order and reference liveness. It must not replay getters or traps to discover an affected supplier, silently choose one dependent view's conversions, or keep receivers alive merely because a prototype remains reachable.

| Direction | Ergonomics | Prior art and limits | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Retain receiver/key lookup dependencies and check affected ordinary mutations before publication | Failure identifies the mutation that would break the contract. An untyped supplier alias can get TypeError without helpful autocomplete; explicit writes through the typed receiver retain their usual admission. | Rust RefCell rejects conflicting dynamic access, but has explicit borrow exclusivity and no prototypes. ECMAScript distinguishes ordinary definition, deletion and prototype change; it supplies no runtime-type policy. | Slot/absence/prototype watchpoints keep ordinary reads fast. Mutation fanout is real cost; share dependencies where possible. No public bookkeeping property or avoidable shape transition. | Enforces all dependent contracts without assigning one receiver's declaration to a shared prototype. No new reference authority or lifetime. | **Recommend**, independently of work. |
| Permit incompatible mutations and check later reads | Failure moves to a later use whose autocomplete still promises the member. | Rust checked access is an analogy, not this proposal's storage rule; ECMAScript getters can fail but ordinary data has no such implicit check. | Requires every consuming path and escaped reference to respect invalidated proofs. | Changes the existing storage-boundary policy and complicates invariant references and check elision. | Reject as the language repair. |
| Stamp only the supplying property with one receiver's type | Unrelated instances become constrained by whichever view arrived first. | A Rust field's declaration belongs to its storage; a shared JavaScript supplier is not each receiver's declaration. | Cheap supplier check, but topology changes still need tracking. | Misses shadowing and retargeting; can replace or overconstrain existing backing. | Reject. |
| Copy inherited properties, freeze the graph, or reject inherited members | Adds own keys, removes ordinary alias permissions, or rejects familiar prototype methods. | Rust explicit copying/ownership does not imply JavaScript copying or freezing. | Adds stores, shape changes or graph locks to unchanged-value admission. | Violates no-write-for-unchanged-value and ordinary identity/lookup rules. | Reject. |
| Trust the initial check forever | Values can contradict the declared property and autocomplete. | Neither Rust location types nor the proposal's own-storage contracts justify stale evidence. | Fast but unsound type/IC assumptions. | Reproduces the two preserved failures. | Reject. |

## Which conversions belong to an indirect mutation?

The actual destination keeps its existing conversions. A write through a typed receiver is such a destination. An untyped prototype serving several receivers does not acquire a chosen receiver's converter merely because their reads depend on it.

| Direction | Ergonomics | Prior art and limits | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Apply the actual destination's existing admission, then require effect-free evidence for every dependent lookup | An indirect replacement can fail where a typed receiver write succeeds. Users can perform the receiving admission explicitly. A raw structural object whose getters would need inspection is not an already established contract. | Rust separates explicit conversions from location obligations; its ownership model does not choose among JavaScript receiver views. | Checks reuse primitive membership and established backing/type evidence. No additional getter, constraint callback, constructor or specialization search. | Preserves source backing and conjunction of receivers. SameValue to the previously admitted value preserves that observation; a changed value needs established membership/storage without pending materialization. | **Recommend.** The refusal is intentional when the indirect operation lacks this evidence. |
| Convert the shared value through one dependent view | Often convenient until adding another receiver changes conversion or effects. | Neither Rust nor ordinary JavaScript selects a converter from unrelated future reads. | Can allocate, call user code and change slot representations according to registration order. | Wrong destination context; one view can erase another's requirements. | Reject. |
| Run a fresh combined structural admission for every indirect change | Accepts more raw values, but prototype changes/deletions can start running nested getters, defaults and conversions. A wrong guess may leave those effects. | Rust has no effectful structural-admission counterpart. JavaScript prototype mutation does not read the members of proposed data values. | Requires preparation, reentry invalidation and potentially many new graph walks on topology changes. | Could define a different effect model, but does not preserve the chosen no-extra-observation policy or identify storage for a converted fallback value. | Reject as the implicit policy. Use an explicit receiving operation when admission is required. |
| Require SameValue for every future mutation | Simple, but rejects ordinary compatible number changes. | Dynamic borrow exclusion is not a rule that a structural property can never change. | Cheapest guards. | Preserves old evidence but unnecessarily removes valid ordinary writes. | Reject. |

The proof retains the proposal's existing distinctions: primitive/literal membership, union alternatives and conjunctive intersections, established structural backing, and absence of remaining callable/default materialization. Nominal subtyping alone does not establish storage representability: a value-class subclass adding storage is refused at a base-class contract, while a same-width subclass and an explicit prior widening conversion remain eligible. A shape resembling an object type is insufficient without its storage obligations. This is not a new general effect-free inference algorithm; defaulted/recursive/index-domain and exotic evidence retain their separate proof requirements. An ordinary direct typed store may perform their required admission instead. Do not treat a new arbitrary getter result or Proxy result as ordinary data evidence.

## Mutation results and lifecycle

Definitions are checked after existing descriptor conversion, destination admission and ordinary descriptor compatibility, before their own data publication. A new inherited-contract conflict throws TypeError. A nonconfigurable ordinary refusal still returns false where the original operation did so. Deletion checks occur after ordinary presence/configurability checks and before deletion or removal-identity changes. A compatible ordinary data fallback is allowed; a missing value, accessor or unproved lookup is refused.

Prototype changes retain the existing ordinary refusal protocol: same-prototype success, nonextensibility and cycle checks precede dependency validation; a contract conflict makes the internal operation return false. Object.setPrototypeOf consequently throws, while Reflect.setPrototypeOf returns false. Earlier argument/descriptor effects and earlier successful definitions are retained. No callback runs between final validation and mutation publication.

This choice preserves the established ECMAScript Boolean prototype protocol and the proposal's typed-property exception protocol. Making every prototype refusal throw would change Reflect semantics; silently accepting a bad mutation would break storage. Rust's Result/panic choice does not determine the JavaScript protocol. All three choices have comparable guard costs; the existing protocols win on compatibility and consistency.

On success, replace old lookup dependencies with the new path. The old supplier becomes unconstrained by that receiver when it is no longer consulted. If an ordinary write creates an own property, its existing backing owns later stores and deletion; retire the inherited dependency. Several receivers sharing a supplier remain conjunctive. Unrelated keys and a supplier's own outgoing prototype do not affect a completed data lookup. Pure membership and failed admission publish no dependency.

A reflected `type` descriptor cannot change an existing property's backing type. Require SameType; supplying the same type remains permitted under ordinary permissions. Allowing a subtype or arbitrary replacement could invalidate existing aliases and invariant references, even after an inherited property becomes own. Rust's declared place types support retaining the declaration, although JavaScript descriptors are a different mechanism. This costs one type identity check at an explicit typed definition, not a read-time shape change. Replacing backing or allowing retyping only when no borrow is observed are rejected: backing identity does not depend on whether a reference happens to be live.

Array publication must obey the same dependencies, including internal batched initialization. Validate the complete pending definitions before an atomic group starts. Array shortening retains its existing ordered deletion/partial-failure behaviour. Acquiring typed Array backing can also change the representation of length or derived byteLength; refuse acquisition when that would invalidate established ordinary inherited-data evidence. Ordinary indexed data and an explicit data byteLength are not excluded just because their holder is an Array.

Dependency bookkeeping must not make a live prototype retain a dead receiver, its values or its type graph. Reverse edges and finalizer holdings are weak; receiver-owned state holds the accepted value. Strong reverse ownership is rejected for its observable retention cost. Polling all objects at each mutation is rejected for its unbounded global cost. Weak dependency indexing preserves the same semantics without adding user-visible fields.

## Scope

This increment covers completed ordinary-data lookup dependencies and their mutation publication. It does not settle arbitrary accessor/Proxy or buffer/SoA contracts, all nested type-role proofs, or the broader checking/inference gates. It neither creates a reference nor revives a removed location. Previously acquired backing remains authoritative.

Sources: [Rust RefCell](https://doc.rust-lang.org/std/cell/struct.RefCell.html), [ECMAScript ordinary object internal methods](https://tc39.es/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots), and this proposal's typed-storage, selected-admission and reference-liveness rules. Performance choices and recommendations are design judgments.
