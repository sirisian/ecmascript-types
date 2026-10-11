# Calling a stored specialization

Storing a specialization must preserve the contract of the specialized function. Its completed signature supplies parameter names, optionality, rest types and reference permissions. A separately declared function or interface type in view still supplies the call site's names and defaults under the ordinary named-argument rule.

```js
function identity(t: type): type { return t; }
function outer<T: type>() {
  function inner<U: type, W: type = identity(U)>(y: W) { return "ok"; }
  const fn = inner.<U: T>;
  return fn(y: 1);
}
outer.<uint8>(); // "ok"
```

The engine262 wrapper had an instantiated signature for reflection but no source formal list. Named binding sometimes found a type on the local binding and worked; routes without that view read the wrapper's absent formals and rejected the name. Whether a local happens to carry that view cannot determine whether this call works.

## Required behavior

- A call with no separately declared callable view reads the specialization's own completed signature. Type arguments and their defaults are already bound; the call does not infer them again.
- Written arguments are evaluated once in source order. Their names then determine positions, and the completed parameter types determine rest distribution and applicable contextual types.
- A skipped ordinary parameter reaches the original function as `undefined`. Its initializer executes at the ordinary call-time point, in the original environment. A default stored in a function/interface type remains call-site data and has its existing precedence. Reading metadata executes neither kind of initializer.
- Interning, receiver selection, where-clause timing and argument admission retain their existing rules. The implementation's forwarding wrapper is not an extra value-consuming boundary for a reference; the original function checks the location or consumes its value. Actual reflective builtins remain value-consuming boundaries.
- Metadata supplies no proof that a reference remains live. The ordinary location checks still apply, including after argument evaluation has effects.

## Directions considered

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / reference liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Require positional calls after storing a specialization | Suggested parameter names cease to work after a harmless assignment; a wrong guess fails after effects. | Rust has no general named-call syntax; that is no precedent for restricting a feature this proposal already defines. | Small interpreter shortcut; no useful shape or inline-cache benefit. | Loses the callable contract and does not resolve reference forwarding. | Reject. |
| Copy source formals and treat the wrapper as the original closure | Names appear repaired while generic types and default environments can disagree. | Rust function items identify an item and its generic arguments; they do not model JavaScript builtin/closure slots. | Duplicates descriptors and can mislead representation-dependent fast paths. | Source syntax alone is not the instantiated contract or the original environment. It establishes no liveness. | Reject the blanket slot copy. |
| Re-evaluate the producer and generic arguments for each call | Repeated defaults/effects make stored identity and completion misleading. | Rust's function-item specialization does not re-execute the source producer. | Extra evaluation and invalidation; effects may change shapes and inline caches. | Breaks creation-time binding and where checks, without proving liveness. | Reject. |
| Read completed specialization metadata and preserve the ordinary binding boundaries | Stored, aliased and direct values expose the same usable names and contracts; explicit views keep their documented behavior. | Rust separates callable identity/instantiation from representation, a limited analogy. This proposal supplies named arguments, runtime environments and reference rules. | Reuse compiler-owned descriptors; no user-visible fields, object shape transitions or extra member-inline-cache cases are required. A production compiler can lower a known call directly. | Preserves completion, source order, defaults, receiver and reference checks without re-inference or new liveness claims. | **Choose independently of implementation effort.** |

Primary comparison: [Rust calls](https://doc.rust-lang.org/reference/expressions/call-expr.html) have positional expression arguments; [Rust function item types](https://doc.rust-lang.org/reference/types/function-item.html) identify the function and generic arguments and can coerce to function pointers. Those facts motivate separating identity and representation, not importing Rust's call syntax or lifetime rules.

## Reference forwarding choices

The wrapper also used builtin entry's ordinary reference decay, so even a valid positional `ref` call reached the original function as a value.

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / reference liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep that extra decay boundary | A displayed ref signature fails only because its specialization was stored. | Rust generic instantiation does not silently turn a reference parameter into a copied value; JavaScript decay rules are specific to this proposal. | Cheap but incorrect; no shape benefit. | Loses the location before the actual parameter check. | Reject. |
| Copy fixed ref-parameter indices to the wrapper | Appears to work for fixed lists but fails when a rest changes later argument positions. | Rust function signatures do not justify a fixed-position encoding of this proposal's variable rests. | Cheap fixed list; requires additional distribution logic for general calls. | A syntactic index is not necessarily an argument position. | Reject as the general solution. |
| Preserve references at every builtin boundary | Makes reflective calls retain locations they currently consume as values. | Rust references are first-class; that is not the proposal's second-class reference model. | Broad change to every builtin and reflective call; no necessary layout benefit. | Violates existing decay/escape semantics. | Reject. |
| Preserve references only at the internal specialization forwarding step | The original ref contract continues to work; explicit reflective calls retain their separate behavior. | The Rust comparison is only preservation of the instantiated signature; this proposal determines decay and liveness. | A private wrapper classification, removable for direct compiled calls; no new user-object shape. | The original callee distributes and checks its parameters, including value decay and liveness. No location is stored or granted a new lifetime. | **Choose independently of effort.** |

## Audit boundary

The implementation repair uses the existing specialization registry, with a narrow single-signature reader, for names and completed parameter types. Only registered specialization wrappers bypass builtin entry's extra reference decay. It does not synthesize source-function slots or expand generic-overload selection.

The audit separately retained failures in variable object spreads without a syntactically named argument, mixed named/reference calls, and non-final reference-rest distribution. They require their own ordinary-call repairs; passing wrapper tests does not close them. Named-reference syntax such as `x: ref value` is not currently in the grammar; a positional borrow followed by a named value is the relevant existing route. Repeated non-rest names follow the current BindArguments sequence of assignments, so observing the last value win is not by itself a conformance failure.
