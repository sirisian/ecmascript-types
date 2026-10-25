# Reentrant admission and retained data observations

A structural boundary can observe a child, then run a sibling getter which independently changes that child's storage. Successful observation of the old value is not permission to publish the changed value under the old contract.

```js
type Item = { fn: (value: string) => string }
          | { fn: <U: type>(value: U) => U };
let escaped: any;
function save(c: Reflect.Object) { escaped = c.objectContext; }
function cross(): number {
  const independent: Item = escaped;
  return 1;
}
class C {
  value: { child: Item, readonly z: number } = {
    child: @save { fn(value: U): U { return value; } },
    get z(): number { return cross(); }
  };
}
```

The fresh child's context selects the generic alternative. The independent boundary in `cross` has no pending construction choice; ordinary canonical admission specializes its `fn` at the concrete string signature. If the outer check already observed the generic `fn`, that observation is now stale. Constructing C must fail with TypeError before publishing `value`. The completed independent admission and earlier effects remain.

## Decision

A valid direction must preserve the selected member, ordinary source identity and width subtyping, once-only successful observations, independent reentry, declared storage contracts, failure precedence, previously completed effects, and reference permissions/liveness. It must not silently replace the nested operation's result or make checking an unbounded fixed-point process.

| Direction | Ergonomics | Prior art and limits | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Validate retained data observations before using and publishing them; fail on conflict | The outer boundary reports that its evidence became stale. Autocomplete continues to show the declared contract. Even a compatible replacement requires a later independent attempt; this predictable cost avoids hidden retries. | Rust's RefCell detects conflicting dynamic access, but blocks the conflicting borrow itself. Here callbacks remain allowed and the receiving admission detects its invalidated evidence. Ordinary ECMAScript data inspection supplies values without executing accessors. | Linear retained-location bookkeeping; engines can use ordinary slot/shape guards and invalidate them at calls. No public property, wrapper or user-object shape change; no extra getter or Proxy call. | Preserves selected contracts, nested effects and failure order. Only a successful write owned by this admission advances its expected value. No reference permission or lifetime is extended. | **Recommend**, independently of implementation effort. |
| Trust the original observation or overwrite the nested result with it | The field can claim generic capability but fail on its first specialization, or erase an effect the getter completed. | Neither Rust checked borrowing nor JavaScript assignment gives an old read authority over a later write. | Cheapest check, but breaks storage/IC assumptions about valid contracts. | Publishes an unproved contract or loses committed effects. | Reject. |
| Re-read and re-admit until the object stabilizes | Correct-looking callbacks can run again; a wrong guess duplicates effects or never finishes. | Rust borrowing does not restart user code to repair conflicts. JavaScript getters are observable operations. | Repeated walks, conversions, IC misses and potentially unbounded retries. | Violates retained successful observations and can select a different union member or failure. | Reject. |
| Re-admit only changed compatible data | Avoids some refusals, but a property replacement can silently add constraint checks, specialization or nested member reads. Autocomplete cannot predict those effects. | Static Rust type compatibility is not this proposal's effectful admission. | Requires a second admission graph and further invalidation tracking; proof-only optimizations must still match one specified outcome. | Compatibility alone does not establish materialization, defaults, nested storage or reference liveness. | Reject as the general language rule. A changed observation fails this admission even if the new value is assignable. |
| Lock objects against reentrant admission or mutation | Ordinary callbacks unexpectedly fail at the nested operation rather than at the invalidated outer boundary. | RefCell's explicit runtime borrow is the closest analogy; this proposal has not made structural checks an exclusive borrow. | Locks/ownership tags add hot-path state and reentry cleanup; object tags can affect shapes/ICs. | Changes established reference authority and callback behaviour. | Reject. |
| Roll back the nested boundary or copy the whole object first | Effects disappear or identity/extra properties change. | A Rust value copy or transaction requires its own representation/ownership contract; neither is implicit here. | Undo logs or graph copying have substantial cost and cannot undo arbitrary JavaScript effects. | Violates ordinary identity, width subtyping and completed-effect rules. | Reject. |

## Required behaviour

Retain whether a successful property observation came from the receiver's own ordinary data location and the value it supplied. Before materialization uses that evidence, require the location still to be data holding the same value. Check it again after a potentially effectful materialization and before a write based on the observation. A successful materialization write by this admission advances its own expected value only after the write is verified. Before publishing new structural storage contracts, validate all retained own-data observations again without executing a getter or trap.

A deleted property, data-to-accessor replacement, or different value invalidates this evidence and throws TypeError. A same-value ordinary data definition or an attribute change alone does not invalidate this data-value observation; ordinary permissions still govern any needed write. The separate selected-construction contribution rule remains stricter about replacement of a producing descriptor, including identical-value replacement. These two kinds of evidence have different jobs.

Do not roll back earlier writes or independently completed admissions. Propagate a getter, setter or materialization exception before a later stability check. Publish none of this failed admission's new contracts or receiving field. Earlier fields and effects remain; later fields do not initialize. Unchanged data and the admission's own numeric/callable conversions remain valid. Getter results retain their existing once-only observation semantics and are not reread to manufacture data evidence.

The engine records own-data evidence during structural membership and keeps a local ledger during completion. Checks read internal data storage only. This is a possible implementation, not a requirement for a Map allocation or a particular IC layout.

## Coverage and open work

The corpus covers both union orders, method/function/arrow literals, public/private/static/accessor/readonly/inherited fields, nested tuples/objects, binding/argument/return boundaries, publication and earlier/later state, independent ordinary admission, compatible nested admission, later-instance recovery, once-only reads, numeric and callable materialization, intersection reuse, index signatures, same-value data definitions, freezing, deletion, accessor replacement, setter-induced invalidation, exact exception precedence and imported contexts.

Two inherited-data probes remain open: a sibling getter changes the prototype's supplying value, or replaces the receiver's prototype. An own-location witness cannot prove those lookup paths. The recommendation is the same evidence-validation direction, extended with the actual data holder and intervening lookup dependencies; it must not discover them by an extra effectful property walk. The rejected directions above remain rejected for those cases. Their required TypeError outcomes are preserved as test programs that catch that error and complete normally. Proxy, accessor, buffer/SoA and other representation-specific evidence require their own existing lookup/liveness rules; this increment does not close those audits.

The function-expression decorator, namespace-alias and imported-constructor guard findings remain separate. The broader completion gates remain open.

Sources: [Rust RefCell](https://doc.rust-lang.org/std/cell/struct.RefCell.html), [ECMAScript ordinary property access](https://tc39.es/ecma262/#sec-ordinarygetownproperty), and this proposal's selected-construction, generic-callable crossing, field publication and reference-liveness rules.

## Correcting an obsolete ordering probe

Existing tuple- and sequence-admission tests used `where gate.value`, with a getter that appended to a mutable outer trace. Both the pulled and patched engines reject it before body entry under the already specified compile-time-evaluability rule. These probes cannot establish runtime ordering because its predicate is invalid.

| Direction | Ergonomics | Prior art and limits | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Use pure passing/failing predicates for ordering, retain the impure source as a rejection test | Observable getter/write order and failure timing remain testable; predicates follow the documented rule. | Rust constant evaluation restricts its expression/effect set. Its const contexts are not this proposal's runtime specialization boundary, but shared mutable tracing is likewise not a portable constant-evaluation model. | Keeps predicates deterministic and cacheable; adds no runtime tracing requirement. | A failing pure predicate proves all source reads occurred before specialization failed, with no writes; the passing case proves reads precede writes. | **Recommend.** The original impure source remains covered as a pre-body StaticTypeError. |
| Relax predicate evaluability to preserve the old trace | A tracing convenience changes when arbitrary mutable state may run. | Rust const evaluation does not justify arbitrary shared mutable effects; JavaScript ordinary getters belong to expression evaluation. | Makes cache reuse and specialization evaluation counts observable. | Contradicts the existing fragment and error-phase rules. | Reject. |
| Delete the failing test without replacement | Users lose regression protection for once-only reads and failure timing. | Neither language's effect rules excuse absent behavioural coverage. | No implementation cost, but no useful verification. | Leaves the intended sequencing untested. | Reject. |
| Require host instrumentation to count predicate evaluations | Useful for debugging, but internal traversal/cache counts become the assertion. | Rust and this proposal permit implementation strategies that preserve outcomes without identical internal events. | Can constrain optimization or require interpreter-only hooks. | Not needed to establish the observable read/failure/write order. | Reject as the conformance test; keep temporary instrumentation only for diagnosis. |

Each revised failing-predicate case requires both inherited reads, then TypeError, with no setter writes. The pure successful case requires both reads before both writes. See [Rust constant evaluation](https://doc.rust-lang.org/reference/const_eval.html) and this specification's compile-time-evaluability and generic-where clauses.
