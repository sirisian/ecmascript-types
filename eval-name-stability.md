# Eval, ordinary evaluation and nominal ancestry

This document records the first two implementation increments of the [eval and stable names plan](eval-name-stability-plan.md): clarify annotation-triggered strictness, repair negative nominal receiving judgments made from incomplete ancestry, and retain a source class declaration's type denotation across source texts. The third increment adds [source-position constructor-origin analysis](class-origin-order.md). The fourth implements the receiving-boundary portion of the [runtime enforcement audit](runtime-enforcement-audit.md); broader static-contribution work and the integrated gate remain open.

## Annotation-triggered strictness is already observable

```js
function make() { return 1; }
function f() {
  eval("var make = () => 2");
  const result: any = make();
  return result;
}
f(); // 1
```

The annotation makes the owning function strict. The eval's declaration stays inside the eval, just as it does if `f` instead contains an explicit `"use strict"` directive. Without either trigger, a sloppy direct eval can introduce a caller-local `var make`, and the corresponding unannotated function returns 2. The result is a scope difference, not evidence that checking cached a call or initializer.

Change the eval source to `make = () => 2` and the annotated function returns 2: assignment still reaches the outer mutable binding. This distinction also applies to constructors. An `any` annotation changes strictness, but it is neither a const declaration nor permission to reuse an earlier function or constructor value.

Checking and strictness have different triggers. A type alias alone can make a function checked without making it strict. This increment adds no special eval scope mode and does not make all checked code strict.

## Missing ancestry is not incompatibility

```js
class V {}
class W extends V {}
const value: V = new W();
eval("1");
```

This program must run. The implementation previously discarded `W`'s base origin because of the eval anywhere in the source, then treated the absent base record as proof that `W` was unrelated to `V`. Even an unused function containing eval could cause the false Early Error. A later assignment to `V` produced the same problem.

The receiving judgment now distinguishes three outcomes:

| Evidence | Receiving-boundary consequence |
| --- | --- |
| Established compatibility | Apply the existing admission and check-elision rules; compatibility alone does not prove value stability |
| Established incompatibility | Retain the existing Early Error, subject to the existing conversion rules |
| Unavailable ancestry needed to decide this pair | Preserve the contracts and evaluate normally; retain runtime admission |

The third outcome belongs to the checker's evidence. It does not change the Boolean `IsAssignable` relation, public Type Objects, nominal identity, or the meaning of `any`. It cannot select an overload or license omission of a runtime check.

An absent base is not always unknown. A class with no heritage, or a known `extends null`, ends the chain. An unrelated such class still fails before body entry. So do independently known primitive mismatches and incompatible arguments of an already established application of the same generic declaration. Unknown ancestry cannot repair a malformed annotation or erase a class's known own field contract.

Source unions retain each alternative's obligations: a known invalid alternative is not hidden by an unresolved one. A target union can remain possible through an unresolved alternative, while a target intersection still requires every conjunct. Ordinary conversions retain their existing rules. In particular, missing ancestry is not a reason to choose a user conversion ahead of direct nominal admission.

## Runtime execution remains authoritative

```js
let calls = 0;
class V {}
eval("V = class { constructor() { calls++; } }");
class W extends V {}
const value: V = new W(); // runtime TypeError after calls becomes 1
```

Here the actual heritage expression captures the replacement constructor. The same-source annotation still denotes the original declaration's class contract under the existing rules. The ordinary `new W()` runs exactly once; its result does not satisfy that contract. Its effects are not rolled back when the receiving check fails.

Conversely, rebinding `V` after `class W extends V {}` does not change the base already captured by `W`. The first increment allowed the resulting store to reach its existing runtime boundary rather than rejecting it from a missing static base record. The third now also retains the earlier heritage observation when its source-position proof is established.

Calls likewise keep their ordinary order: capture the callee, evaluate arguments, then call the captured value with its original receiver. Checking neither runs an unused ordinary initializer nor pre-parses an unexecuted eval source. Explicitly permitted type computations remain governed by their own rules.

Successful admission still acquires the existing nominal prototype obligations. A compatible intermediate prototype link remains allowed, while a mutation that removes the admitted `V` relationship is refused. Reference liveness remains independent: eval can remove an array element, and a later access through the old reference must fail even if an unrelated nominal receiving check succeeded.

## One declaration denotation across source texts

```js
// Script 1
class V {}
class W extends V {}
const Original = V;

// Script 2, same realm
V = class {};

// Script 3
const value: V = new W(); // succeeds against the original V
type Saved = V;          // retains the original declaration
```

The source class binding has two roles. A type position denotes the class introduced by that evaluated declaration. An ordinary expression reads the binding's current value. The distinction is the same within one source text and across later Scripts, direct/indirect eval, dynamically constructed functions and module imports.

| Spelling after replacement | Denotation or effect |
| --- | --- |
| `: V`, `type V`, `type Saved = V` | Original declaration's type |
| `V`, `new V()`, `class X extends V {}` | Current constructor value and ordinary evaluation |
| `Reflect.isAssignable(V, type V)` | Compare the actual replacement constructor's associated type with the original declaration's type |
| Imported `C` in `: C`, or `ns.V` in a type | Producer's original class declaration, following renames/re-exports |
| Imported `C` or `ns.V` as an ordinary expression | Producer binding's current value |

Only successful class evaluation and binding initialization publish an outer denotation. Decorator or static-initializer failure leaves that outer binding unavailable; a later statement throwing does not undo an earlier successful declaration. A class's inner context remains distinct. Separate local activations, recursion and suspended functions retain their own evaluated identities. Generic references use the original family for arguments, defaults and specialization reuse, including through a module namespace.

Runtime class decorators receive the original created class through `context.type` while the outer binding remains uninitialized. After decoration succeeds, that binding receives the replacement value, if any; its type denotation retains the original declared class contract. This also closes an implementation gap: a failed declaration must not leave a constructor association that could satisfy an escaped forward receiving dependency.

The same evaluation path applies to decorated named and default exports, including an anonymous default class. A partial declaration reopens the established class and does not initialize or republish its original declaration binding.

The publication audit required making the decorator commit point explicit. A valid direction must retain ordinary effects, avoid a usable failed outer declaration, preserve the declared contract and respect reference liveness.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Initialize and publish before decorators finish | Outer reads work during decoration, but a thrown decorator can leave a usable declaration | Rust supplies no runtime-decorator analogy; this diverges from JavaScript's class initialization boundary | Early roots and contracts require invalidation after failure | Violates successful-publication and failure-recovery requirements | Rejected |
| Initialize early, defer type publication, then revoke the binding on failure | The name is temporarily usable and later returns to TDZ; escaped observations become surprising | Neither Rust's declaration identity nor ordinary ECMAScript class evaluation motivates this transaction model | Requires additional state and invalidation; user effects cannot be rolled back | Can distinguish some states, but exposes a provisional outer binding and needs new observable rollback semantics | Rejected |
| Keep the outer binding uninitialized through decoration; initialize once and retain the declaration contract on success | Decorators use their supplied context; a failed declaration remains unavailable | Follows ECMAScript's outer class-binding initialization boundary. Rust supports fixed declaration identity but does not model these runtime callbacks | One successful binding publication; retained constructor is an internal GC root, with no user-instance shape changes | Preserves the contract, failure ordering, activation identities and liveness; effects already performed remain | Recommended independently of implementation effort |

Resolve the nearest ordinary binding first. A nearer `const V = type string` uses that binding's own type value. A nearer uninitialized binding still blocks the outer `V` and keeps the applicable TDZ/type-formation failure. Dynamic lookup likewise cannot bypass a nearer ordinary binding to recover an outer class. This does not introduce independent type and value namespaces or make arbitrary mutable reads valid type computations.

The retained constructor is part of the declaration's reachable contract, including for garbage collection. It is stored in internal binding metadata, with no added user-object property or instance shape transition. Real engines can lower a known type reference to a contract/environment slot; ordinary value loads and their inline caches still track the current value. This is an implementation strategy, not a performance benchmark or a license to omit receiving/prototype/reference checks.

The plan's Q3 and Q9 matrices give the rejected alternatives and their reasons. This increment adopts their declaration-and-binding-instance model rather than a per-source snapshot or a second namespace.

## Implementation boundary and remaining work

The engine records unresolved heritage separately from a known end of ancestry on its internal nominal checker records. The receiving check consults that evidence only after ordinary assignability fails. Known contracts and generic arguments survive. Runtime admission, conversion execution, prototype enforcement and reference checks continue through their existing paths.

This is a checking change, not a new property on user instances or constructors. It introduces no instance shape transition or new runtime inline-cache policy. A production engine may optimize a retained boundary using its existing identity and prototype guards, but must preserve the actual operands, effects and failure phase. No performance improvement is claimed by this interpreter experiment.

The recommended later increments remain:

- Audit other ancestry consumers, callable/intrinsic origin judgments, overload resolution and optimization paths against the source-position evidence model. Their independent whole-source screens are not automatically relaxed by the third increment.
- Complete the broader class, module and reference conformance matrix and the integrated plan's larger regression gate.

The recommendations are selected in the plan independently of implementation cost. These increments do not discard the remaining directions or claim their acceptance gates have passed. The [first implementation record](eval-name-stability-implementation.md) and [declaration-denotation record](class-denotation-implementation.md) identify their separate patch bases and validation.
