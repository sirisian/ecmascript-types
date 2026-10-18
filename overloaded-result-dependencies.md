# Overloaded callable result dependencies

A callable with several signatures retains the local class dependencies of each signature. Introducing an overload, an immutable binding or a forwarding function must not lose those dependencies.

```js
function make() {
  class C {}
  function f(value: string): C { return new C(); }
  function f(value: number): C { return new C(); }
  return f;
}
function forward() { return make(); }
const f = forward();
const first = f("x"), second = f(1); // both use this activation's C
```

The implementation previously rejected the binding of the written-result form. In the inferred-result form, binding a selected call's result could look up `C` in the caller or attempt another conversion after losing the selected invocation's context. These are distinct failures: completion handled only one signature, and the dispatcher discarded the selected call's result dependency.

## Requirements and alternatives

Completion must preserve all signatures, authoritative targets, own generic binders, parameter modes and defaults, receivers, narrowing facts and the ordinary overload-selection rules. Different evaluations of a local class remain distinct. An already checked contribution supplies a dependency independently of the tested callable. No body, getter or initializer may run merely to determine correspondence, and no correspondence grants reference liveness. Signature-list order cannot choose a class activation.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Erase every multi-signature dependency | Adding an overload loses completion precision and lets wrong replacements pass | Rust function types retain parameter and result types; its signatures do not justify this erasure | Cheap but loses useful type and IC guards | Discards established contracts | Rejected |
| Pair signatures by list position or take the first compatible input list | Reordering overloads can change which result class is accepted; equal inputs with different results become dangerous | Rust's scoped binders offer no justification for choosing this proposal's runtime contract by list order | Fast matching with an unsound key | Return-overload and activation ambiguities can select the wrong dependency | Rejected |
| Invoke or inspect the candidate callable to derive the destination | A wrong replacement can redefine its receiving target or execute effects while being checked | Rust function-pointer types describe contracts independently of runtime function identity | Extra calls or public property reads undermine predictable guards | Circular admission; may change failure order and reference state | Rejected |
| Require one source signature and reject all multiple matches | Simple diagnostics, but equivalent retained contributions become unnecessarily unusable | Rust supplies no corresponding runtime overload-set restriction | Avoids agreement checking at the cost of valid reuse | Safe for distinct candidates but too restrictive for equal completed contracts | Rejected |
| Complete corresponding descriptions from retained contracts, require exact contract correspondence, and accept multiple candidates only when their completed contracts agree | Equivalent forwarding retains members; wrong activations and ambiguous dependencies do not silently change meaning | Rust supports declared function shapes and scoped generic parameters. The activation and overload-set rules come from this proposal | Memoized type-graph work; production engines can index stable signature shapes and declaration dependencies. No new user-object properties or shape transitions; existing family guards remain useful | Preserves independent targets, binders, modes, default values and liveness rules; disagreement supplies no identity | Recommended independently of effort |

Choose the final direction. Match each target signature against the retained source descriptions independently of list position. Complete only corresponding unresolved dependencies in its parameter, result, receiver and narrowing descriptions. Preserve the target's modes and other structure, then require exact contract correspondence, including its effective inferred result. Own generic binder declarations must correspond; same spelling or structural alpha equivalence alone does not authorize substituting an unrelated binder's dependency. Several candidates are acceptable only if they establish the same completed contract. An unavailable or conflicting dependency remains unresolved for the ordinary admission rules; it is not `any`.

This is a dependency-completion rule, not a change to function type identity, ordinary subtyping, overload ranking or return-context selection. An already completed target cannot be retargeted. Written erased contracts remain erased. Constraint/default computations keep their declaration environments and existing evaluation timing; this increment does not establish a general solver for unresolved callable constraints or defaults.

Rust's [function pointer types](https://doc.rust-lang.org/reference/types/function-pointer.html) describe parameter and result shapes without requiring a particular runtime function identity. Its [generic-parameter scope](https://doc.rust-lang.org/reference/items/generics.html) provides a limited binder analogy and excludes enclosing parameters from nested items. Neither is a direct model for this proposal's capturing, reified overload sets.

## Selected invocation transport

An implementation's overload or specialization dispatcher is a forwarding mechanism, not a newly declared `any` result contract. Preserve the selected invocation's checked normal-result dependency through that dispatcher and its internal call frame. Preserve abrupt completions and restore selection/type frames on every exit. This applies to ordinary overload resolution and the selected owner/case paths, including explicit selection.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Drop dependency metadata when unwrapping the selected result | Storing the result can fail even though an identical discarded call works | Rust retains the selected function's result type; ECMAScript uses completion records to carry outcomes | Saves a small internal reference but forces missing or incorrect later checks | Loses invocation-owned identity | Rejected |
| Reconstruct it from the dispatcher, caller scope or returned object | A later factory call or a same-named caller class can change admission | Neither Rust binder scope nor ECMAScript lexical scope supports caller-based reconstruction | Repeated lookup and unstable guards | Replaces independent evidence with unrelated context or the tested value | Rejected |
| Preserve the selected call's completion dependency through transparent implementation wrappers | Binding and forwarding agree with the selected member; effects occur once | Rust's retained result contract is the limited type analogy; ECMAScript completion propagation is the control-flow model | An internal pointer transfer; no instance layout change or extra call | Keeps successful evidence and abrupt outcomes with their invocation, without changing reference permissions | Recommended independently of effort |

Choose the final direction. This does not grant general reflective calls or arbitrary user-written wrappers stronger contracts than they declare. The ordinary result-admission rule still applies at each actual boundary.

Reflection and runtime callable compatibility also describe each overload member using that member's inferred publication and captured declaration context. A dispatcher has no single body publication that can replace those member contracts. Reading the signatures does not execute the members or recreate their local classes.

## Scope

The regression work covers written and inferred results, declaration and expression classes, reversed declarations, equal and different activations, recursive forwarding, nested generic arguments, selected cases, effects, erasure and Module transport. Contract-level checks cover reordered, equivalent and conflicting retained descriptions and preservation of parameter permissions. General stored-specialization initialization, unresolved callable constraints/defaults, broader composite transport and the remaining convergence, phase, budget and full-verification gates require their own evidence.
