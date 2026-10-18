# Declared conversion results

`operator T()` and `operator T(value: S)` promise a result of `T`. Their selected body does not publish an unchecked value merely because the caller already selected that declaration. The result crosses a checked return boundary in the operator's declaration context, before control returns to the receiving operation.

```js
class Source {
  operator uint8() {
    const dynamic: any = 7;
    return dynamic;
  }
}
const input: any = new Source();
const result: uint8 = input; // carries uint8, not an untyped Number
```

Replacing `7` with dynamic `257` throws `RangeError`. Writing `input := uint8` selects the same declared operator and does not relax its return contract. The body may explicitly write `dynamic := uint8` if wrapping is intended. A wrong dynamic kind or prohibited implicit numeric-family conversion remains `TypeError`.

## Required behavior

- Resolve the target with the declaration's lexical, private, generic type/value, Realm and Script/Module context. An escaped declaration retains these captures; the caller's same-spelled bindings cannot substitute for them.
- In the absence of a written return annotation, the declared target provides the return expression's contextual type. A written annotation is an additional contract; even `: any` cannot erase the target. Preserve its existing contextual and admission behavior, then enforce the target.
- Apply ordinary return-expression evaluation, including `finally`, before dynamic result admission. A normal fallthrough supplies `undefined`; existing static no-return errors still apply before execution.
- For a nominal target, require the selected body to return a member before ordinary result transport. A nonmember must not trigger a constructor or inbound retry. Other targets use their existing checked boundary conversion, including numeric representation, metadata transport and structural storage admission.
- Preserve value-class copying, independently completed obligations and ordinary effects. A failed assignment does not replace its destination. A failure does not undo effects of the body or its `finally`.
- Propagate a selected body or result-admission failure without retrying another outer candidate. A body can catch a nested failure and recover. Non-nominal checked result admission can itself invoke a declared conversion: a finite chain is nested boundary execution, not speculative candidate search.
- Explicit references remain subject to the existing value/reference distinction and liveness checks. This result boundary grants no location permission, revives no invalid reference and extends no lifetime.
- Primitive operator blocks retain their separate raw-result protocol. A class conversion is not a primitive arithmetic operator merely because its target is primitive.

For a metadata target, explicit conversion must consult a matching source declaration for the complete target before decomposing an intrinsic numeric cast into base conversion and metadata construction.

## Decision record

Valid directions must enforce the promised representation, preserve checked numeric failures, retain declaration captures, avoid outer candidate retries, preserve nominal selected-result membership and existing copying/liveness rules. Cost of implementation does not decide the result.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness | Other / disposition |
|---|---|---|---|---|---|
| Trust the body and return its raw value | Autocomplete promises `T` while operations observe another type; a wrong guess can leak into later storage | Rust checks a function's result against its declared return type; trusting dynamic results has no close analogy | Cheapest locally, but callers cannot rely on representation or stable specialized code | Fails numeric carriers, range checks and storage contracts | Rejected: the declaration would not be a reliable contract |
| Stamp the target onto any normal result | Appears convenient until invalid values behave as valid `T` | Rust casts and `From`/`TryFrom` implementations do not make an arbitrary value inhabit the result type by relabeling it | Cheap marking, but invalid payloads defeat optimized representation assumptions | Unsound for wrong kinds, out-of-range numbers, metadata and nominal identity | Rejected: a stamp is evidence after admission, not admission |
| Require membership for every result, with no conversion | Rejects a dynamic in-range Number returned for `uint8`, unlike ordinary checked return boundaries | Rust has restricted return coercions, not blanket exact source-type equality | Cheap guard; extra explicit conversions at callers/bodies | Preserves membership but conflicts with the proposal's checked numeric and structural boundaries | Rejected as a universal rule; retained as the existing nominal precondition |
| Apply unrestricted checked conversion to every result | Consistent-looking returns, but a bad nominal result unexpectedly invokes another constructor/operator | Rust's `From`/`TryFrom` are explicit APIs, not implicit arbitrary conversion chains | Can run additional user code and allocate after the selected body | Violates the selected nominal result rule and observable no-retry behavior | Rejected: cannot turn a nonmember from a selected nominal conversion into a second conversion search |
| Checked result admission with the existing nominal membership precondition | The visible target remains trustworthy; implicit and explicit callers see the same body contract and failures | Closest to Rust's typed function result/coercion-site model, with the explicit caveat that this proposal permits more boundary conversions | Resolve/capture the target per specialization; guards and numeric checks can inline, stable representations support ICs. Metadata and structural effects remain real costs; no claim of interpreter timings as engine evidence | Preserves checked transport, nominal selection, captures, copy rules and reference liveness | Recommended and implemented for the covered result paths |

Rust comparison: [return expressions](https://doc.rust-lang.org/reference/expressions/return-expr.html), [coercion sites](https://doc.rust-lang.org/reference/type-coercions.html), and [`TryFrom`](https://doc.rust-lang.org/std/convert/trait.TryFrom.html). Rust return coercions are restricted and do not implicitly execute arbitrary `From` or `TryFrom` implementations. The analogy supports a typed result contract, not this proposal's entire conversion policy.

## Coverage boundary

The paired corpus covers primitive representation and errors, explicit casts/type calls, generic captures and aliases, contextual overloads, tuples/arrays, metadata, nominal nonmember refusal, inbound value-class copying, effects and finite chains. Host tests cover Modules, `Function`, `eval`, foreign Realm errors and feature-disabled behavior.

General conversion selection is still incomplete. In particular, a source conversion to a value class can currently fail on the target's structural fields before entering the source operator. This is a separate retained witness, not a passing result-copy case. Unbounded recursive conversion limits, general static/partial/exotic selection and the imported-class `instanceof` checker failure remain separate work. These bounded results do not close those gates.
