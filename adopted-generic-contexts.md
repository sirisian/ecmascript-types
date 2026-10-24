# Adopted generic contexts preserve binder identity

A fresh method, function expression or arrow with no written type-parameter list can adopt the single generic signature supplied by its position. Its adopted parameters are effective own declarations for body checking, inference and invocation. Naming a function while initializing a binding does not remove that context.

```js
type Adapter<C: type> = {
  map<T: type>(value: T): { captured: C, own: T }
};
function factory<T: type>(seed: T) {
  const adapter: Adapter.<T> = {
    map(value) { return { captured: seed, own: value }; }
  };
  return adapter;
}
const pair = factory.<number>(0).map.<string>("s");
```

The adopted `T` and factory's `T` are distinct declarations. Autocomplete shows Number for `pair.captured` and String for `pair.own`. A wrong String consumer of `captured` is an established mismatch. Renaming the method's parameter to `U`, moving the literal to a return or argument position, or replacing a contextual callable's function expression with an arrow must not retarget the capture.

Retain the declaration identity of an actual type argument before applying an alias or composing a structural signature. Otherwise substituting `C` with an unqualified `T` loses the distinction before the literal receives its context. The adopted parameter scope supplies body references and nested binding contracts; already captured contracts retain their original declarations. Written type-parameter lists retain precedence. Speculative contextual trials must restore their adopted scopes on failure. Do not mutate source syntax into a chosen trial or a concrete activation.

The effective generic contract includes its constraints and defaults in their original context. This semantic requirement does not permit resolving an alias's free parameter in the literal's unrelated environment. No part of adoption initializes a captured value, executes a body to obtain its type, grants reference authority or extends reference liveness. Runtime parameter admission, result admission and reflection remain separate required consumers of the same contract.

## Directions considered

A valid direction must preserve both binders before substitution, use the adopted list throughout body checking and runtime creation, respect written declarations and failed trials, and preserve ordinary effect order and reference rules.

| Direction | Ergonomics and wrong-guess cost | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Adopt a signature only for compatibility, leaving the body in the outer generic scope | A displayed generic parameter means something different inside its implementation; harmless renaming changes values and diagnostics. | Rust closure input inference supplies body types; its nested items have different generic scope rules. Neither supports two contradictory meanings for one effective signature. | Cheap initially; inconsistent contracts obstruct specialization and cache reuse. No shape or IC benefit. | Confuses adopted own binders and captures. | Reject. |
| Reject generic contextual adoption or all overlapping names | Forces explicit boilerplate or renaming and removes already accepted methods. | Rust E0403 restricts duplicate associated-item generic names; E0401 makes nested items independent. Those restrictions do not define this proposal's lexical closures. | Simple syntax checks but no object-layout benefit. | Changes the existing contextual contract and leaves other alias capture losses unresolved. | Reject. |
| Erase the inferred result or waive callable admission | Autocomplete loses established field types; wrong guesses fail later or escape checking. | Rust retains inferred closure types; gradual erasure is not an equivalent rule. | Less checking sacrifices useful optimization facts; no necessary IC change. | Discards established obligations. | Reject. |
| Replace equal names everywhere, or append parameters to shared syntax | Appears to repair simple cases but changes nested own binders or leaks one contextual trial into another. | Semantic binder identities, not textual spelling, distinguish generic parameters; Rust's syntax differences do not justify mutating source identity. | Shared AST mutation complicates incremental checks and activation caches. | Breaks scope, trial rollback or independent activations. | Reject. |
| Preserve actual argument identities and adopt an effective context-owned parameter scope | The body, returned value and editor agree; nested captures survive renaming, placement and separate activations. | Rust's separation of generic identity and closure environment is useful. Rust closures infer their parameters, but are not a direct analogue of these contextually polymorphic function values. | Compiler-owned symbol/type graphs and per-function context suffice. No public fields, wrappers, shape transitions or added property IC keys are required. | Preserves own/free distinction, existing admission and independent liveness. | Choose independently of implementation effort. |

Performance statements describe an implementation architecture, not measurements. Rust references: [E0401](https://doc.rust-lang.org/error_codes/E0401.html), [E0403](https://doc.rust-lang.org/error_codes/E0403.html), and [closure expressions](https://doc.rust-lang.org/reference/expressions/closure-expr.html). The analogy concerns inference and declaration scope; Rust's item and ownership restrictions are not imported here.

## Bounded implementation result and remaining gaps

The current proof-of-concept repair retains argument declaration identity before alias substitution, journals adopted parameter scopes, applies those scopes to nested capture contracts, and gives created function/arrow values the adopted generic list. Named lexical initializers now receive their position's runtime context. Selected cases cover local, return and argument contexts, nested functions/arrows/patterns, inferred and explicit calls, known wrong consumers, separate activations and imported aliases.

The bound/default capture repair is described in [adopted generic records](adopted-generic-records.md). The effective-input repair, including reflection, generic-to-concrete admission and implicit erased invocation, is described in [adopted generic inputs](adopted-generic-inputs.md). The annotation-phase repair is described in [adopted generic annotations](adopted-generic-annotations.md). The lexical-alias repair is described in [adopted generic shadowing](adopted-generic-shadowing.md). Complete-union-context selection remains a required follow-up repair, not alternative semantics or a passing conformance case. Broader generic-context, convergence, failure-state, resource and specification audits remain open.
