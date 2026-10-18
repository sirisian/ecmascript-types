# Result dependencies through expressions and immutable bindings

A selected value keeps its applicable nominal dependency when a value-preserving expression or an ordinary immutable binding forwards it:

```javascript
function make(tag: string) {
  class P { tag: string = tag; constructor(v: string) {} }
  return (v: any) => { const p: P = v; return p; };
}
const a = make("a");
const b = make("b");
function use(flag: boolean) {
  const result = flag ? a("x") : b("x");
  return result;
}
```

The two activations share a declaration, but not a constructor. The selected invocation supplies the dependency. The returned instance cannot choose its own admission target.

## Requirements and decision

Selection and binding must preserve the published members and the selected activation. Evaluate each reached operand and each ordinary value read once, in ordinary order. Preserve short-circuiting, abrupt completion, TDZ, reference checks and finalizer selection. An inferred initialization and a receiving return each keep their own admission boundary. A written target or erased contract remains authoritative. No dependency may be published on an uninitialized binding after failed admission, or recovered by examining the result object. Shared syntax and reflection remain independent of the last activation.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Reject or erase selected and stored results | Ordinary conditional/const refactoring loses members or fails remotely; autocomplete becomes weaker | Rust infers let bindings and gives an if expression its selected block's value, subject to a common static type | Cheap, but loses useful contracts and optimizations | Discards existing result obligations | Rejected: representation limits do not justify changed semantics |
| Reconstruct from the returned instance or its prototype | Appears convenient, but a wrong value can validate itself | Rust does not infer nominal identity by inspecting an instance at a receiving boundary | Adds object/prototype inspection and weakens stable IC guards | Circular admission; conflates contract evidence with the value being checked | Rejected: a target must be independently established |
| Cache by source, spelling, or most recent selected call | Simple examples pass; reentry, shadowing and repeated factory activations change behavior | Neither Rust lexical bindings nor JavaScript environments identify activations by spelling alone | Small cache is unsound across activations and Realms | Stale targets can replace the actual contribution; reference checks do not repair identity | Rejected: incorrect cache key |
| Re-evaluate the initializer or inspect all branches at the return | A const refactor duplicates side effects; skipped branches can throw | JavaScript selects only one operand; Rust also executes only the selected if block | Repeated calls and environment walks; effects preclude safe reuse | Changes evaluation order, TDZ and failure precedence | Rejected: retention is not another evaluation |
| Carry dependency with selected expression results and retain the checked target on the actual binding instance | Refactors preserve members and nominal checks; errors stay at their actual boundary | Fits JavaScript operand selection and lexical binding identity; Rust supports the selection/inference intuition but has static nominal identity and different borrowing | Internal result/binding slots; no user object shape changes; optimized flow can pass the dependency alongside the value | Selected path only, successful initialization only, ordinary reads/checks retained; explicit targets remain authoritative | Recommended independently of implementation effort |

Rust's [if expressions](https://doc.rust-lang.org/reference/expressions/if-expr.html) yield the selected block's value and require a consistent static type. Its [let statements](https://doc.rust-lang.org/reference/statements.html#let-statements) infer an omitted annotation. Neither models a fresh nominal constructor on every dynamic class activation. ECMAScript's [conditional and short-circuit expressions](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-conditional-operator) supply the closer runtime evaluation model. This proposal retains its own conversion and reference-liveness obligations.

## Bounded implementation

The existing root nongeneric ClassDeclaration dependency is now transported through conditional, logical-or, logical-and, nullish-coalescing and comma expressions. Only the selected operand or final comma operand supplies evidence. A shared value-read helper preserves internal completion metadata without passing a retained completion variable through the compiler's in-place Q unwrapping transform.

A plain const initializer retains its completion through contextual evaluation. Its inferred boundary can instantiate a matching unresolved nominal dependency, then performs ordinary admission. An annotated initializer records its own successfully checked target, including erasure. Only after initialization succeeds does the immutable ordinary binding retain that checked dependency. The binding's recorded inferred type is that invocation's completed target, not a mutation of the shared source publication. Ordinary environment reads export the dependency after the normal TDZ check; indirect Module reads use the actual target binding. Return statements and concise bodies use the completed value read, rather than the earlier reference result.

Internal weak-map keys are completion records and binding records, not user objects, names or Parse Nodes. The immutable-binding restriction means successful later writes cannot leave stale retained metadata: mutable and reference bindings receive none from this path. A failed const assignment does not change the binding's value or retained evidence. A future mutable-storage path must replace or invalidate metadata at the actual committed write, preserve it after a failed write, and retain the storage's own declared contract.

## Remaining scope

This closes the measured conditional-return and plain const-result failures in both selected branches. It does not establish generic class applications, nominal components of composites, destructured result bindings, mutable result storage, reference aliases, async/generator resolution, or every call wrapper. The full extension suite and portable resource/retention audit remain open. Carrying a checked dependency never waives a receiving check or proves reference liveness.
