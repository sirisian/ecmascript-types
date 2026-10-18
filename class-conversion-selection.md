# Class conversion selection and structural shapes

A class's field description is used by type relations; it is not a recipe for turning an unrelated object into an instance. When a value does not already belong to the receiving class, select the class's declared conversion on the original value. Do not first read or convert the source's properties as though it were the target's structural shape.

```js
class Point { x: uint8 = 1; }
const saved: any = new Point();
class Source {
  operator Point() { return saved; }
}
const source: any = new Source();
const result: Point = source;
result.x = 9; // saved.x remains 1
```

The source need not have an `x` property. A source getter called `x` is unrelated to conversion and must not run merely because the target has that field. A shape-compatible object with no applicable declared conversion is still a nonmember. Reject it without rewriting its fields. Structural object and interface targets retain their own field conversion and storage admission.

## Selection and copying

Existing membership precedes conversion and retains the ordinary value-class copy rules. Otherwise an applicable converting constructor precedes a source conversion, which precedes an inbound conversion. For a resolved, non-generic class with an ordinary permanent prototype and the already supported simple parameter forms, use the same declaration-context selection proof for a value-class target as for an identity-class target. A representation layout does not justify changing precedence, discarding overload alternatives or matching targets by display name.

Retain the chosen operation and execute it once. Its actual parameter boundary stays checked even when the receiving operation is explicit. An operator's result crosses the [declared result boundary](conversion-result-admission.md), including value copying. A constructor builds its selected result under its construction rules; selection does not add an extra copy merely because it uses a shared planner. Verify selected nominal membership and preserve independent result obligations. A selected failure propagates without fallback. Prior effects remain, while a failed assignment retains its old destination.

These rules do not make value-class parameters pure applicability probes. Nor do they add value classes to the scalar-rest storage domain: selecting one conversion and planning copy-sensitive runs are separate problems. The [constructor applicability extension](constructor-conversion-applicability.md) now covers supported defaulted/optional parameters and constructor overload groups. Generic target selection, effectful parameter admission, static/partial declarations and exotic lookup retain their separate coverage requirements. The general class/shape distinction also applies on these paths; it is not permission to fabricate a class from matching properties.

## Decision record

A valid solution must preserve nominal identity, membership-first behavior, constructor/source/inbound precedence, checked parameters/results, value copies and reference liveness. It must avoid unrelated source property effects and retain structural conversion for actual structural targets.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Convert fields before selecting the declared operation | A conversion advertised for Source unexpectedly requires Point's fields; a wrong guess invokes getters or mutates data before failing | Rust nominal structs do not acquire identity from matching fields | Unnecessary property Gets, writes and shape/IC traffic before dispatch | Violates class identity and observable conversion order | Rejected; this is the reproduced defect |
| Treat a matching shape as a class instance | Autocomplete suggests class methods/private behavior that construction never established | Rust separates nominal struct construction from structural similarity | Requires fabrication or branding, plus validation of private state/layout | Bypasses construction, nominal membership and layout invariants | Rejected; a different type system, not a repair |
| Hoist source operators ahead of every other action | Fixes the example but changes existing-member and constructor behavior | Rust explicit conversion traits do not justify this proposal's changed precedence | Can invoke unnecessary user code and allocate despite existing membership | Breaks membership-first and constructor preference | Rejected |
| Skip class shapes but retain the old value-class dispatch table | Removes accidental field effects, but same-named classes and inbound declaration order can still choose the wrong body | Rust nominal identity and coherence argue against spelling/source-order selection | Cheap lookup, but semantic ambiguity is unresolved | Leaves known identity, applicability and specificity inconsistencies between value and identity classes | Rejected as the complete bounded solution |
| Separate class admission from structural conversion; share the supported nominal plan for value-class targets | Users see the declared target, precedence and copying they already expect; unrelated fields have no effects | Consistent with Rust nominal structs and typed From/TryFrom results; Rust calls these conversions explicitly and does not share this runtime ranking scheme | Target-kind discrimination is stable; retained declaration descriptors avoid speculative Gets/writes and extra source shapes. Result copies retain their actual cost; no benchmark claim | Preserves all required invariants while leaving unsupported proofs explicit | Recommended independently of implementation effort |

Rust sources: [nominal structs](https://doc.rust-lang.org/reference/items/structs.html), [`From`](https://doc.rust-lang.org/std/convert/trait.From.html), [`TryFrom`](https://doc.rust-lang.org/std/convert/trait.TryFrom.html). This is a limited analogy: Rust has no corresponding JavaScript prototype mutation or implicit constructor/source/inbound ranking. The performance assessment describes plausible production mechanisms, not interpreter timing evidence.

## Correction to the retained Module observations

The previous report incorrectly classified two `RT_IMPOSSIBLE_TEST` results as checker bugs. Both tested membership already guaranteed by an annotation in a deciding position, for example `if (!(n is uint8))` after `const n: uint8 = ...`. The existing narrowing rule intentionally rejects a checked guard that can never succeed or can never fail. Negating it does not make the test uncertain. The same rule applies in Scripts and Modules.

A value-position query such as `String(n is uint8)` or `String(c instanceof C)` remains allowed. The paired controls now check both value queries and required guard rejections, including imported classes, primitives, explicit `any` and feature-disabled code. The old witnesses are retained as required rejections, not silently removed or counted as repairs.

| Direction considered | Ergonomics | Prior art, including Rust | Performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Exempt Module guards | Assertion idioms work only in one source form; autocomplete/type guarantees no longer imply the same checking result | Rust does not establish a Script/Module distinction for this proposal's rule | Trivial runtime effect | Contradicts the existing source-form-independent checked-code rule | Rejected |
| Permit every statically settled assertion guard | Familiar defensive assertion style, but changes the proposal's deliberate dead-branch diagnostics | Rust's lint/error policy differs; its acceptance of assertion idioms does not determine this proposal's policy | May permit constant folding; no inherent runtime obstacle | Requires a general narrowing-design change, not a fix to these witnesses | Recorded as a possible separate design revision; rejected for this repair because the present rule is explicit and consistent |
| Keep the rule and observe membership in value positions | Preserves the specified diagnostic; test authors can still inspect actual runtime values | Compatible with keeping nominal facts reliable, without importing Rust's diagnostic policy | No new runtime mechanism or object shape | Matches both existing specification and baseline/current behavior | Recommended; correct the report and add controls |

No reference capability is created or revived by either the class conversion selection rule or the observation controls. Conversion parameter/result checks and any actual reference read retain their own permission and liveness requirements.

## Regression repair: logical operand checking

The broader subset exposed a separate, genuine baseline checker failure: `((v instanceof A) && v.x === 1) ? "yes" : "no"` was rejected for `v: A | null`. The existing flow walk narrowed the right operand, but result inference and discarded-expression validation separately inspected it in the unrefined entry state. Inferred and annotated bindings, call arguments, typed returns and the corresponding `||` form had the same problem.

These paths now use the existing predecessor-state computation: the right operand of `&&` is checked under the left operand's true outcome, and the right operand of `||` under its false outcome. Restore the caller's flow state afterwards. Wrong-predecessor reads remain errors; facts do not leak past the join or survive invalidating assignments. This implements the existing tested-expression rule, without changing the settled-guard diagnostic policy above.

| Direction | Ergonomics | Prior art, including Rust | Performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Suppress member errors on logical right operands | Stops the false positive but also hides genuinely unsafe reads | Rust does not treat short-circuit syntax as permission to ignore type errors | Cheap omission | Unsound for the opposite branch and invalidated facts | Rejected |
| Walk both operands early in the current state | Superficially unifies checking, but may change diagnostics and inferred facts by consumer position | Short-circuit evaluation in Rust and ECMAScript does not evaluate both operands unconditionally | Repeats analysis and may over-invalidate facts | Loses conditional predecessor information and source-order effects | Rejected |
| Reuse existing predecessor flow for every affected typing path | The same guarded expression has the same member availability in a return, binding, argument or discarded expression | Consistent with control-dependent checking; Rust's static type/narrowing details are not this proposal's | Additional compile-time flow queries, no new runtime Gets, copies or object shapes; caching/accounting remains a separate gate | Preserves branch-specific facts, invalidation, joins and the independent reference rules | Recommended and implemented |

## Regression repair: comma-result literal adaptation

The expanded narrowing subset also exposed a baseline rejection of `o.v >= (o.v = y, 0)`. The specification already gives a comma expression's final operand the contextual type while preserving earlier effects. The numeric-family check instead required the entire operand to be constant; runtime literal recognition also stopped at the comma. The comparison now recognizes the result producer separately, passes known numeric ordering context to it, and retains ordinary evaluation of every preceding operand. Literal range and equality eligibility use the same result producer. The general constant-expression predicate and narrowing-path recognition remain unchanged.

Controls cover either operand, nested commas, named constants, a constant-expression result, runtime-only operand types, earlier throws, stored Number values, different typed widths, invalid earlier arithmetic, out-of-range ordering literals and unequal out-of-range equality literals. The metadata regression is paired with a no-write case: an intervening write prevents refinement of the changed place, while a direct literal comparison still yields its specified impossible-branch diagnostic. Reference reads and invalidations are still performed in source order; looking through a comma for result eligibility does not prove a reference remains live.

| Direction | Ergonomics | Prior art, including Rust | Performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Require a cast after every effectful comma | A literal's accepted type depends on unrelated preceding work; wrong guesses cause inconsistent errors | Rust blocks can perform statements and then yield a final expression; that analogy does not support discarding the result's typing context | Simple rejection, but unnecessary conversions at call sites | Contradicts the existing final-operand contextual rule | Rejected |
| Classify the entire comma expression as constant | Makes examples pass but silently invites folding away required work or deriving stale facts | Rust block results do not make their preceding statements constant or effect-free | May incorrectly eliminate effects | Conflates result eligibility, constant evaluation and narrowing-path validity | Rejected |
| Forward result eligibility and context, preserving the separate effect analysis | Users retain the expected final value/type; stored values still require an explicit conversion | Rust's sequential block evaluation and final-operand result are a close structural analogy, not an identical coercion or liveness model | Compile-time result-node traversal and existing runtime numeric admission; no new source object shapes or IC keys | Preserves evaluation order, abrupt completion, range checks, numeric provenance and invalidation | Recommended and implemented |

Rust reference: [block evaluation and its final operand](https://doc.rust-lang.org/reference/expressions/block-expr.html). This repair implements the existing contextual-types and narrowing rules; it does not add a new implicit conversion policy.
