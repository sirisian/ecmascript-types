# Eval, ordinary evaluation and nominal ancestry

This document adopts the first implementation increment of the [eval and stable names plan](eval-name-stability-plan.md): clarify annotation-triggered strictness and repair negative nominal receiving judgments made from incomplete ancestry. The remaining declaration-denotation and source-order work is recorded separately below.

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

Conversely, rebinding `V` after `class W extends V {}` does not change the base already captured by `W`. This increment allows the resulting store to reach its existing runtime boundary rather than rejecting it from a missing static base record. It does not yet improve the whole-source origin analysis that discarded that record.

Calls likewise keep their ordinary order: capture the callee, evaluate arguments, then call the captured value with its original receiver. Checking neither runs an unused ordinary initializer nor pre-parses an unexecuted eval source. Explicitly permitted type computations remain governed by their own rules.

Successful admission still acquires the existing nominal prototype obligations. A compatible intermediate prototype link remains allowed, while a mutation that removes the admitted `V` relationship is refused. Reference liveness remains independent: eval can remove an array element, and a later access through the old reference must fail even if an unrelated nominal receiving check succeeded.

## Implementation boundary and remaining work

The engine records unresolved heritage separately from a known end of ancestry on its internal nominal checker records. The receiving check consults that evidence only after ordinary assignability fails. Known contracts and generic arguments survive. Runtime admission, conversion execution, prototype enforcement and reference checks continue through their existing paths.

This is a checking change, not a new property on user instances or constructors. It introduces no instance shape transition or new runtime inline-cache policy. A production engine may optimize a retained boundary using its existing identity and prototype guards, but must preserve the actual operands, effects and failure phase. No performance improvement is claimed by this interpreter experiment.

The recommended later increments remain:

- Retain each evaluated declaration's original type denotation across later Scripts, including publication, shadowing and failure recovery. The current cross-script behavior is not changed here.
- Replace whole-source eval/assignment screening with binding-aware evidence transferred in execution order, including indirect effects and the constructor operand itself.
- Audit other ancestry consumers, overload resolution and optimization paths against that evidence model; complete the broader class, module and reference conformance matrix.

The recommendations are selected in the plan independently of implementation cost. Keeping this first patch focused does not discard those directions or claim their acceptance gates have passed.
