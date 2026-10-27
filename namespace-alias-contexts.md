# Namespace alias contexts

An imported namespace path used as a type annotation retains the exported alias's declaration context. Qualifying the name does not erase the signature that supplies a function literal's adopted generic parameters.

```js
// dep
export type F = <U: type>(value: U) => U;
export type G = <V: type>(value: V) => V;

// main
import * as ns from "dep";
class C {
  f: ns.F = function(value: U): U { return value; };
  g: ns.G = function(value: V): V { return value; };
}
```

Both functions are generic. Their types remain alpha-equivalent, but constructing `g` uses `G`'s `V` declaration even if `F`'s Type Object was interned first. Renaming exports, star re-exports and nested namespace re-exports preserve the exporting declaration. A lexical shadow of `ns` selects its own binding instead.

## Decision

A valid direction must retain contextual binder identity and captured inputs, existing public type equality, source-scope argument/default resolution, ordinary namespace reads and failures, module checking phases, and reference permissions/liveness. It must not execute user code merely to discover a declaration.

| Direction | Ergonomics | Prior art and limits | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Resolve established namespace paths through exports; retain alias context separately from Type Object identity | Autocomplete can show the exported signature and its actual binders. Changing import spelling does not require rewriting the function; invalid bodies get the ordinary checking error. | Rust paths identify declared items and types, including generic applications. Rust paths are statically resolved and have no JavaScript namespace-object TDZ or dynamic property-read effects; ECMAScript supplies those separate obligations. | A linked export can identify a declaration/slot; its existing alias metadata can supply context. No user properties, wrappers or shape transitions are necessary. Optimized namespace ICs can retain the same binding guard. | Preserves declaration identity and alpha-equivalence without bypassing reads, availability or liveness. | **Recommend**, independently of implementation effort. Runtime context is available only from successful alias publication. |
| Require named imports or explicitly repeated function binders for namespace annotations | Namespace autocomplete promises a signature that the literal cannot use; a routine import refactor causes distant body errors. | Rust's type paths do not support rejecting an otherwise resolved item solely for qualification. | Removes a checker path but forces redundant source and misses normal linked-export optimization. | Contradicts the existing contextual signature rule. | Reject. |
| Use the canonical Type Object's signature, or split identity by spelling | First-use order affects construction, or equivalent aliases unexpectedly become unequal. | Rust item identity is not evidence for making these structural function types nominal. | Canonical-only lookup loses information; split identity grows intern tables and guard diversity. | Either loses the selected binder or breaks alpha-equivalence. | Reject both variants. |
| Execute namespace/property reads during checking to discover the alias | Getters, cycles and dependency effects become sensitive to whether a body needs context. A wrong guess can execute code before its expected phase. | Ordinary ECMAScript namespace reads have binding availability checks; Rust's static paths do not authorize early JavaScript evaluation. | Adds potentially effectful calls and prevents straightforward checking reuse. | Can bypass or move TDZ and module failures, publish unavailable inputs, or manufacture provenance from values. | Reject. |
| Infer provenance from property names, object shape or an equal stored Type Object | A lookalike object or Proxy silently acquires another module's binder names. Autocomplete and runtime resolution disagree after shadowing. | Neither Rust declaration paths nor ECMAScript export resolution are shape matching. | Requires unreliable shape/name guards or broad caches; can perturb object shapes if metadata is attached publicly. | Confuses unrelated bindings and activations; cannot establish liveness. | Reject. Ordinary object typing remains governed by its existing rules. |

These performance assessments describe feasible implementation costs, not measurements.

## Sequencing and scope

1. During checking, identify the namespace from established import/immutable-alias evidence and follow ordinary resolved exports. Retain the actual alias declaration and its producing module; do not use a same-spelled consumer declaration or read a property for provenance. Ambiguous, missing or unestablished paths gain no declaration evidence from this rule.
2. Check the supplied signature in its declaration context. Written generic arguments belong to the caller; omitted defaults, constraints and the expanded alias body belong to the alias declaration with its preceding argument bindings. Qualification does not create a new application rule or ordinary binding for an unapplied generic alias.
3. During evaluation, perform the existing qualified reads in their existing order. Only after a successful read from an actual module namespace can its resolved exporting binding supply published alias context. An abrupt read supplies no context. No extra getter read, alias re-evaluation or module-body execution is needed to discover provenance.
4. Preserve ordinary receiving admission and publication. Failed body checking precedes execution where the existing checking phase can decide it. A cyclic runtime read of an unavailable alias retains its ordinary TDZ failure. Later graphs and successful bindings remain independent of failed consumers.

The engine connects established namespace paths to source alias declarations and retains context through the existing binding metadata after normal namespace reads. This is an implementation strategy, not a required object layout. Public alias values, reflection equality and reference-liveness rules are unchanged.

## Coverage and remaining work

The corpus covers direct, renamed, star, nested/deep and locally forwarded namespace exports; named namespace imports and immutable local aliases; fields, bindings, arguments, returns, assignments, objects, tuples, arrows and decorators; binder order, warming, repeated instances, private/imported dependencies, explicit/named/spread/defaulted generic arguments, constraints, alias chains, lexical shadowing and explicit own binders. It includes ordinary-object controls, evaluability refusals, invalid bodies, missing exports, dependency failures, cycles and same-realm recovery.

This repairs the two preserved namespace contextual-signature cases. It does not certify every computed alias, arbitrary stored Type Object, callable union, namespace-property reference location, deferred-import family or generic specialization family. The subsequent [preparation and eval context repair](prepared-alias-contexts.md) recovers external alias contexts for direct eval and refreshes contextual obligations after computed aliases become available. The [computed-result context repair](computed-result-contexts.md) then preserves selected builder contexts before alias publication, closing the named/namespace computed-result pair. A written outer generic parameter in eval, the imported-constructor guard/effect audit and the broader completion gates remain open; no namespace value authorizes executing reads during checking.

Sources: [Rust paths](https://doc.rust-lang.org/reference/paths.html), [Rust generic scopes](https://doc.rust-lang.org/reference/items/generics.html#generic-parameter-scopes), and [ECMAScript module namespace reads](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-module-namespace-exotic-objects-get-p-receiver). The recommendation applies this proposal's existing contextual typing, alias publication and module-order rules; the analogies above do not add new JavaScript semantics.
