# Property-key coercion and stable result evidence

November 1, 2026. Tenth increment of the [eval and stable names plan](eval-name-stability-plan.md), continuing milestone 5 after [descriptor/accessor effects](descriptor-access-effects.md). The [implementation record](property-key-effects-implementation.md) records patch reconciliation, validation and limits.

## Problem and recommendation

```js
// First Script
const key = {
  [Symbol.toPrimitive]() {
    globalThis.f = () => 1;
    return "x";
  }
};

// Second Script
function f(): string { return "old"; }
Object.defineProperty({}, key, { value: 0 });
const n: number = f();
```

The pulled implementation rejects the second Script before execution. Its descriptor summary accounts for descriptor conversion and installed accessors, but not the conversion of `key`. Execution converts the key, replaces `f`, and admits the actual result `1`.

The same distinction applies to computed reads, writes, deletes, updates, property definitions, patterns and applicable reflective calls. A target can be an ordinary fresh object and still have an effectful key. Evaluating a reference for a plain write does not read the property's getter. Its key expression runs while forming the reference; the final key conversion runs at the store, after the right-hand side.

**Recommend treating ordinary property-key coercion as a separate effect at the operation that consumes the key.** A positive primitive proof can establish that this conversion cannot invoke an object hook. An unknown value, an object candidate, or an incomplete proof retains the possible effect. Keep the receiving check for an unavailable result relationship; do not turn the missing proof into a declaration-result incompatibility.

## Conditions a valid solution must meet

| Condition | Consequence |
| --- | --- |
| Key evaluation and conversion are distinct | A key expression's own effects occur even if its resulting value is primitive |
| Empty flow does not mean primitive | Unknown parameters and opaque results retain possible conversion effects |
| All possible contributors matter | One primitive branch cannot cancel an unknown branch |
| The operation determines timing | A simple store converts its key after its RHS; a compound read converts before its RHS; reflective conversion follows argument evaluation |
| Definition names are immediate | A computed method or field name is evaluated during definition, separately from a deferred method body or instance initializer |
| Optional short circuit remains meaningful | A provably skipped conversion contributes no conversion effect |
| Reference and contract rules remain independent | A primitive key supplies no new reference authority, liveness or callable implementation identity |

A primitive-key proof concerns the conversion only. It does not prove that a subsequent Get, Set, proxy operation, overloaded index accessor or invocation is effect-free.

## Question 1: which computed keys should contribute effects?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Ignore conversion and summarize the final property operation only | Ordinary-looking brackets can cause false Early Errors far from the hook | Rust indexing can call `Index`/`IndexMut`; syntax alone does not imply a data read. Rust has no direct equivalent of JS field-name coercion | Cheap, but omits actual calls | Fails the reproduced cases and plain-write key semantics | **Reject** |
| Treat every computed key as arbitrary code | Avoids the counterexamples but loses errors for literal and proven primitive keys | Rust statically distinguishes index types and implementations; ECMAScript primitive conversion also has positive cases | Cheap fallback; discards useful checking and optimization evidence | Conservative, but fails required precision controls as a complete policy | **Reject** as the finished direction |
| Execute or constant-fold arbitrary key objects during checking | Autocomplete and diagnostics depend on speculative user execution and mutable state | Neither Rust trait checking nor ECMAScript conversion licenses replaying ordinary user hooks here | Unbounded work and observable effects | Violates phase, order and single-evaluation requirements | **Reject** |
| Separate coercion effects with positive primitive proofs | Literals and proven primitives retain diagnostics; uncertain code keeps runtime boundaries | ECMAScript supplies the conversion protocol. Rust reinforces that operator syntax can hide calls, while its reference-returning traits have different authority rules | Private dependency facts; no added object fields, wrapper or shape transition | Handles opaque hooks without executing them and preserves independent contracts | **Recommend** |

Autocomplete can continue to show independently declared callable and value contracts. It should not promise that a mutable global still holds an earlier implementation merely because a property operation appears harmless. An uncertain key can move the affected result judgment to its runtime receiving boundary; guessing incorrectly does not silently admit an incompatible value.

A production engine can reuse established primitive facts and existing compiler dependencies. Shape and inline-cache guards may optimize the actual property operation, but a later guard cannot retroactively justify an irrevocable Early Error. This prototype adds analysis metadata, not an extra runtime key lookup or conversion.

## Question 2: what constitutes a primitive proof in a may-flow graph?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Interpret an empty candidate set as primitive | Looks precise until an unknown parameter or omitted return path appears | Rust uses established type information, not the absence of a heap-origin candidate | Minimal work | Confuses unavailable evidence with a guarantee | **Reject** |
| Accept any primitive contributor | Simple aliases work, but `choose ? "x" : unknown` becomes unsound | Rust branch typing must account for alternatives; it does not discard a possible trait implementation because another branch is simple | Cheap existential test | An unknown alternative can still invoke conversion code | **Reject** |
| Accept only syntactic literals | Predictable but needlessly loses aliases and packed literal arguments | Rust uses more than surface syntax; ECMAScript also has stable primitive values and intrinsic results | Cheap and bounded | Safe subset, inadequate as the intended precision policy | **Reject** as the finished direction |
| Require positive evidence across contributing values, with an unknown fallback | Preserves supported aliases and primitive sources; uncertain joins defer instead of guessing | Closest to ordinary type-proof discipline, including Rust, without importing its ownership guarantees | Reverse flow edges and bounded traversal; cycles and exhaustion fall back conservatively | Distinguishes empty carrier slots from unknown expressions and rejects mixed unknown contributors | **Recommend** |

The prototype records expression provenance and incoming flow edges. An empty carrier slot can contribute no value; an expression without a positive primitive fact is unknown. A recognized object or capability candidate defeats the proof. Literal/native primitive facts and supported intrinsic results can establish a positive contribution, but every relevant contributing expression must agree.

The String/Symbol shortcut requires the actual intrinsic binding or ordinary primitive data, no conflicting source binding/assignment, and the existing source-effect screen. It is not a rule that the spelling `String` or `Symbol` always denotes an intrinsic. Guards are attached to the selected operation; ancestors enclosing that capture do not pretend to execute before it. Other source children remain part of the screen.

The same source proof admits immutable local Symbol keys naming own data entries of a fresh literal object. It checks the key's declaration identity, rather than its description. The key and object bindings must have no annotation or reference conversion, the object must contain only data definitions, and a matching own entry must exist. Those accesses are inert under the intrinsic hypothesis being checked; source deletion, reflection, escape calls, conflicting writes and other unknown effects still defeat the screen. This preserves existing local-symbol construction judgments without claiming that arbitrary computed reads or prototype lookups are harmless.

This is a prototype proof subset. Broader primitive-contract judgments, exotic paths and other intrinsic summaries still need specification and tests. Resource bounds are not a normative license for engines to choose arbitrary observable Early Errors.

## Question 3: where does the coercion occur?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Place every effect at the key's textual start | Later conversion appears to alter an already evaluated operand or callee | Rust evaluation order is semantic too; textual position is not an execution trace | Cheap offset comparison | Misorders reflective arguments and pattern initializers | **Reject** |
| Move simple-store conversion before its RHS | Breaks existing effect logs, captured RHS values and throwing-hook behavior | Conflicts with ECMAScript's explicit deferred conversion; Rust indexing is not equivalent to this JS protocol | Requires a runtime change without improving the evidence proof | Changes base-language order and abrupt-completion precedence | **Reject** |
| Treat all syntax inside a method or field as deferred | Computed names lose their definition-time behavior and earlier captures become unreliable | Rust associated item names are not arbitrary JS expressions; its method-body analogy does not cover computed names | Simple ancestor classification | Confuses a name with the method body or instance initializer | **Reject** |
| Use the consuming operation and distinguish names, bodies and skipped paths | Matches ordinary execution, wrong-value errors and effect counters | ECMAScript is authoritative. Rust offers a limited distinction between locating/indexing storage and later use | Reuses operation-site summaries, without runtime replay | Preserves RHS-before-pattern/store conversion, computed-name-before-value, compound-read-before-RHS and reflective-call ordering | **Recommend** |

For a computed reference, evaluate its base and key expression first. In simple assignment, evaluate the RHS before converting the final reference key at the store. A nested member read used to obtain the destination base, and a compound or logical assignment's initial read, perform their conversion before the RHS. Effects inside the key expression itself also precede the RHS. A computed property definition converts its name before its value. A destructuring initializer precedes the pattern; a pattern key precedes its own default. Reflective calls evaluate their callee and arguments before the intrinsic converts the key. The `in` operator's ordinary key path participates; a private-name brand check does not perform this conversion.

Computed method and field names execute during definition even when their bodies or instance initializers run later. The supported literal-null optional case skips conversion. This does not claim general reachability analysis or a complete optional-chain abstract interpreter.

An initial test draft assumed that simple assignment converted the final key before its RHS. The current ECMAScript EvaluatePropertyAccessWithExpressionKey algorithm explicitly defers that conversion, and the engine already implements that order. Reject moving conversion earlier: it would change ordinary effects, thrown completions and the value captured by the RHS. The corrected tests distinguish key evaluation, final store conversion and a compound target's read. The analysis also narrows its previous blanket assignment-LHS exclusion so earlier key/base effects are not suppressed.

No change makes a reference live, reuses a location after failure, bypasses a nominal prototype obligation or changes the runtime evaluation algorithm. Counters, ordering logs, throwing hooks and receiving failures verify that ordinary execution remains single-pass.

## Remaining work

The [eleventh increment](reflective-method-origins.md) repairs reflective method identity and opaque invocation fallback. Milestone 5 remains open for broader reflective consumers, primitive-contract precision, arbitrary prototype relinking, callback combinations, overloaded-index effects and other consumers of static evidence. This increment does not prove universal absence of effects through an arbitrary heap. Milestone 6 and the existing test/build blockers remain open; no full extension/test262 run or production-engine benchmark is claimed.

## Primary references

[ECMAScript ToPropertyKey](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-topropertykey), [ToPrimitive](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-toprimitive), [EvaluatePropertyAccessWithExpressionKey](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-evaluate-property-access-with-expression-key) and [Rust Index](https://doc.rust-lang.org/std/ops/trait.Index.html) supply the comparisons. The Rust operation can dispatch through a trait and return a reference; that is a useful warning against treating brackets as inert syntax, not a justification for transplanting Rust borrowing rules into JavaScript coercion.
