# Contributions follow expression evaluation order

A later evaluated operand sees the effects of earlier operands. This applies when an expression supplies a return, initializer, contextual result or selected member, as well as when it controls a statement.

```js
function make(seed: number) {
  const box = { x: "s" };
  return (box.x = 1) && box.x;
}
const n: number = make(0);
```

The right operand contributes Number. The earlier String initializer cannot reject this program. The equivalent conditional and comma forms use the same source-position facts. This adds no annotation to `box` and does not change assignment's ordinary runtime result.

For `&&` and `||`, obtain the left operand's value contribution in its incoming environment, transfer its effects, then query the right under the applicable truthy or falsy continuation. For `??`, use the captured left value's nullish alternatives to select the fallback environment after the left's effects. A comma result uses the environment after its preceding operands. A conditional queries each arm under its own selecting continuation. Join outgoing effects separately from the values the expression can produce. Preserve the existing result-type selection, contextual adaptation and required checks of written operands; impossible-test and unreachable-operand diagnostics still apply.

Preserve both the source-position description and the identity of an aggregate result. A type query must restore its caller's flow without rereading the result in that earlier state. A later binding of that result still aliases the same aggregate: after `const alias = (box.x = 1, box)`, a write through either name affects a later private description read through the other. Scalar results instead retain the value already obtained. Independent admitted contracts remain authoritative. A query does not execute source code, initialize a binding, or establish reference permission or liveness.

Unknown calls, getters, stores and other effects retain the existing invalidation rules. Only evaluated alternatives transfer effects; required checking of a skipped operand does not execute or transfer its effects onto the skipped path. Queries retain their original declaration, generic, receiver and contextual environments and dependencies. An unfinished input is not a completed unknown answer.

## Decision

A valid direction must preserve order, skipped effects, earlier values, aggregate aliases, declaration evidence, admitted contracts, contextual typing, dependency completion and independent reference liveness.

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Leave operand queries independent | Harmless statement-to-expression refactors expose stale member types and reject valid code. | JavaScript and Rust sequence effects within evaluated expressions; neither licenses stale later reads. | Cheap but wrong metadata; no shape or IC benefit. | Fails source-position consistency. | Reject. |
| Apply all syntactic effects before every operand | Skipped writes pollute results and earlier copied values appear to change. | Both languages skip unselected lazy operands. | A simple scan loses edge information; no runtime layout benefit. | Violates selection and captured-value semantics. | Reject. |
| Erase every effectful contribution | Useful types disappear from autocomplete and known mistakes move to runtime. | Rust retains static types through effects, although its pervasive contracts differ from this proposal. | Less analysis at substantial precision cost; no shape or IC benefit. | Conservative invalidation is necessary for unproved effects, but blanket erasure fails established transparent inference. | Reject as the general rule. |
| Keep only a frozen aggregate result description | Initial reads look right but aliases stop describing each other's writes. | Rust's place/value distinction is relevant; its borrowing prevents many aliases allowed here. | Small snapshots, but stale alias information; no shape benefit. | Confuses an aggregate alias with a copied scalar. | Reject. |
| Execute expressions to discover their results | Checking and autocomplete can invoke getters or ordinary bodies, even for rejected code. | Neither language's runtime order grants a checker permission to execute source. | Unbounded effects obstruct compilation, caching and predictable resource use. | Violates phases, initialization and reference liveness. | Reject. |
| Transfer each evaluated component in its governing environment; retain its result and aggregate identity | Equivalent forms retain useful types; wrong dynamic guesses meet ordinary receiving checks. | JavaScript supplies the actual order and operand-return semantics. Rust provides the limited place/value and lazy-evaluation analogy. | Compiler-owned flow, identity and dependency records can share existing analysis. No user-object fields, wrappers, shape transitions or IC keys are required. | Preserves all listed requirements, with existing conservative invalidation where no transfer is proved. | Choose independently of implementation effort. |

Rust's `&&` and `||` require Boolean operands, and assignment produces unit; JavaScript returns selected operands and assignment values. Rust assignment evaluation order must not be imported into this proposal. The performance assessment above is design reasoning, not a benchmark. Primary comparisons: [Rust operator expressions](https://doc.rust-lang.org/reference/expressions/operator-expr.html) and [ECMAScript logical operators](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-binary-logical-operators).

## Implementation boundary

The proof of concept shares aggregate transfer with operand-flow queries, carries nullish fallback state, and keeps a queried aggregate's description separate from its alias identity. Contextual comma and conditional results follow the same ordering. It does not add a general heap analysis, execute call bodies for effect inference, or promote private descriptions to storage contracts. The bounded tests do not close all expression forms, dependent/recursive completion, portable resource accounting or final regressions.
