# Invocation-specific inferred result dependencies

Two converters created from one local class declaration can retain different evaluated classes. A forwarding function must preserve whichever dependency its executed return contributes:

```javascript
function make(tag: string) {
  class P { tag: string = tag; constructor(v: string) {} }
  return (v: any) => { const p: P = v; return p; };
}
const a = make("a");
const b = make("b");
function choose(flag: boolean) {
  if (flag) return a("x");
  return b("x");
}
```

The class source declaration is shared, but the constructors are not. Likewise, `const local = make("c")` inside a forwarding function supplies a dependency created by that invocation. Reflection cannot execute that initializer or decide which branch will run.

## Requirements and decision

Preserve the published nominal declaration and the actual contributing invocation's class identity. Instantiate a target independently of the returned value. Keep completed targets, explicit annotations, erased contracts, conversion, failure precedence and reference-liveness obligations authoritative. Nested calls and arguments must not overwrite each other's evidence. A normal `finally` preserves the pending return; a replacing return contributes its own dependency, and a throwing `finally` cancels the pending result. Admission remains at the existing boundary.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Erase or reject every invocation-dependent result | Valid factories and branch refactors lose members or require exposed implementation details; wrong guesses fail far from the boundary | Rust does not erase a function's result merely because control flow chooses an expression; its static nominal model differs from dynamic classes | Cheap but forfeits useful published information | Changes existing publication and nominal obligations | Rejected: representation limits do not justify weaker semantics |
| Pick a constructor from one branch, the latest call, or the returned object | Call order and unrelated calls change behavior; autocomplete cannot promise a stable contract | Neither Rust's return location nor JavaScript lexical bindings are a latest-result cache | Tempting small cache; invalid across activations, reentry and Realms, with unsafe IC specialization | Conflates identity or lets the value establish its own admission target | Rejected: insufficient or circular evidence |
| Re-evaluate the source dependency after the body or run producers during reflection | Initializers and getters run again; wrong assumptions duplicate user effects | Rust return evaluation moves one evaluated argument; JavaScript also evaluates the operand once | Repeated effects and environment lookup prevent safe reuse | Can observe a replaced binding, lose a departed block or violate phase and liveness rules | Rejected: discovery must not be another program evaluation |
| Admit eagerly at each `return`, before `finally` | Most simple examples work, but discarded returns can throw and change which cleanup runs | JavaScript completion selection is the closer analogy; Rust has no JavaScript-style `finally` that replaces a return | Avoids retained evidence, but adds checks on discarded contributions | Changes the established admission phase and failure precedence | Rejected: dependency retention must not move the check |
| Retain the checked dependency with the contributing completion and instantiate at the existing receiving boundary | Factories, branches and ordinary forwarding keep their members and nominal checks; wrong returns still fail at their boundary | Fits JavaScript completion selection and an invocation-specific output location; Rust supports the latter intuition, not dynamic nominal classes | Internal temporary metadata or compiler result slots, with no user object shape changes; no global activation cache or IC key based only on source syntax | Preserves selected identity through reentry and `finally`; requires the receiving check again, including liveness | Recommended independently of implementation effort |

The ECMAScript [try-statement semantics](https://tc39.es/ecma262/multipage/ecmascript-language-statements-and-declarations.html#sec-try-statement-runtime-semantics-evaluation) preserve a pending completion when `finally` completes normally, and otherwise select the finalizer's completion. This is the control-flow model the dependency must follow. Rust's [return expression](https://doc.rust-lang.org/reference/expressions/return-expr.html) places the evaluated argument in the current invocation's output location before transferring control. That supports retaining invocation-specific output information, but Rust's move/destruction rules, static nominal identity and lack of JavaScript `finally` are substantial differences. This proposal keeps its existing reference-liveness rules.

## Bounded engine implementation

The engine now retains a successfully checked root nongeneric ClassDeclaration result on an internal completion. A direct returned call contributes that admitted dependency to the new return completion. After the body and finalizers complete, the receiving function may use it to complete an unresolved nominal target only when the source declaration agrees. It cannot replace an already completed constructor or a written return annotation. The ordinary receiving check still runs. Only after it succeeds can that receiving invocation pass its checked target onward.

Separate weak maps distinguish a forwarded candidate from a checked result. Keys are individual completion records, not user values, function objects or shared Parse Nodes. No metadata is added to a returned instance, and reflection never reads these maps. Rewrapping an ordinary return as a call result transfers the checked dependency. An explicit `any` or other non-nominal result does not export a hidden stronger target merely because its operand had one. Throwing paths export no successful result.

This supports local factories, different return-site activations, nested ordinary forwarding, concise arrows, recursion, generic frames, member and imported calls, and foreign Realms. Normal and replacing finalizers follow their selected completion without consulting a last-call slot. Argument calls and reentry cannot overwrite the pending result. The build transform may unwrap a variable passed directly to `Q`; code that needs the completion afterward must explicitly retain it and inspect its completion kind before reading its value.

In a production engine this evidence can live in temporary result slots and flow with optimized control flow. Those slots must keep their constructor dependency reachable until the boundary completes, then release it. The interpreter's weak maps establish no global cache of concrete activations. Complete retention proofs and portable budget accounting remain part of the broader audit.

## Remaining propagation

The conditional-return and plain const-result cases are now repaired by [expression and immutable-binding transport](expression-result-dependencies.md). Selected operand completions retain their dependency; an inferred initializer admits its value before retaining the checked target on its binding. Reads and receiving returns keep their own ordinary checks.

Generic class applications, nominal components of composites, destructured and mutable storage, resumable resolution/return paths and wrappers that discard internal completions still need further coverage. Successful admission through an imported direct call or immutable binding does not imply that static reflection can already close every imported dependency.
