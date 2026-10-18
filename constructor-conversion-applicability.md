# Constructor applicability before conversion selection

A default or an overload group does not make a constructor applicable to every value. A conversion considers each explicitly declared constructor with exactly one simple value parameter. A defaulted or optional parameter still checks a supplied non-undefined value against its declared type. A supplied `undefined` uses the parameter's existing optional/default binding rule; it is not an omitted argument in the conversion operation itself. Default declarations retain compile-time evaluability and validation requirements. Applicability does not evaluate their initializers.

```js
let calls = "";
class Point {
  x: uint8 = 1;
  constructor(value: boolean = true) { calls += "b"; }
  constructor(value: string) { calls += "t"; }
}
class Source {
  operator Point() { calls += "s"; return new Point(true); }
}
const source: any = new Source();
const point: Point = source; // calls is "sb"
```

Neither constructor accepts the Source instance, so its exact source conversion is selected. A selected constructor that throws is different: propagate that throw, preserve earlier effects and leave a failed destination assignment unchanged. Do not retry a source or inbound operator.

## Selection and retained execution

For the bounded ordinary nominal planner, retain every constructor body's signature in its declaration context. Reflection's record of the first constructor is not the overload set. An unavailable annotated signature is not an unannotated `any` parameter. An unavailable declaration set is not proof that no constructor applies.

First establish parameter admission using the existing pure rules: `any`, primitive membership or intrinsically pure checked conversion, and existing ordinary identity-class membership. A pure failed conversion makes that candidate unavailable and retains its failure. Unknown effectful admission must not be called inapplicable. Value-class parameters, metadata/composite parameters, generic target specialization and exotic evidence remain outside this bounded proof.

Rank the admitted constructor candidates with the ordinary call-ranking rules. Do not replace them with source order or subtype-only ranking. Constructors with more or fewer than one parameter are not converting constructors, even when a multi-parameter constructor can accept a one-argument direct call through defaults. Rest and reference parameters do not enter this simple-value-parameter specialization.

Retain the chosen body and the original argument. Construct the selected class with its normal allocation, fields, private state, `this`, `new.target`, parameter checking and return rules. Do not rerank the group after field initializers or earlier selected conversions have performed effects. If an effect invalidates the selected parameter's actual admission, that boundary fails; a different overload does not replace it. Ordinary direct `new` keeps its existing call resolution.

Existing class membership still wins before constructor selection and preserves value copying. Otherwise constructors precede source and inbound operators. A selected constructor ambiguity fails before instance fields or user conversion bodies run. In ordered rests, retain that ambiguity through complete count assignment; a zero-length run selects nothing. Pure failures must not displace a different viable constructor, later conversion form or complete rest assignment. Preserve final sequence/backing validation and reference liveness. Retention uses weak owner keys; this does not establish complete portable cache accounting.

## Decision record

A valid solution must preserve declaration identity and captures, exact converting-constructor eligibility, call ranking, constructor/source/inbound precedence, selected-failure propagation, copy rules, source-order effects and reference liveness. It must distinguish absent evidence, proved inapplicability, pure failure and selected ambiguity.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Try constructors and catch failures before trying an operator | A wrong guess can run fields, defaults or bodies before the advertised conversion | Rust From/TryFrom select a typed conversion; failure is not a search over speculative constructors | Unnecessary allocations, effects and shape/IC traffic | Breaks failure precedence and observable effects | Rejected |
| Ignore every defaulted or overloaded constructor | Useful constructors disappear from conversion although autocomplete shows their matching parameter | Rust traits do not justify ignoring an established source type; C++ has converting constructors and overload resolution, with different eligibility rules | Cheap but may invoke a lower-priority operation | Breaks constructor preference and the current one-parameter rule | Rejected |
| Use the first viable declaration or subtype-only ranking | Reordering declarations changes behavior, or conversion disagrees with ordinary call ranks | Neither Rust's typed trait selection nor C++ overload resolution supports a declaration-order tie break | Simple dispatch, but semantically unstable | Loses numeric/exact/catchall ranking or ambiguity | Rejected |
| Resolve the full direct-call group again after allocation | Familiar direct-call behavior but can select a multi-parameter constructor or change the winner after field effects | C++ permits a wider converting-constructor domain; importing it would change this proposal. Rust has no corresponding implicit constructor group | Repeated ranking and dependent lookups; can allocate before an already-known error | Violates eligibility and retention of the selected operation | Rejected |
| Prove admission, use shared call ranking, retain and execute the selected body | Matching parameters predict behavior; defaults and overload order no longer hide a source conversion; failures identify the selected operation | Rust supports typed conversion identity and a distinct fallible operation; C++ supports constructor-based conversion/ranking. Neither determines this proposal's runtime precedence or borrow rules | Declaration descriptors avoid speculative Gets, new source shapes and repeated selection; ordinary construction still pays its real allocation/copy cost | Preserves the required invariants and leaves unsupported proofs explicit | Recommended independently of implementation effort |

Primary sources: [Rust From and its TryFrom guidance](https://doc.rust-lang.org/std/convert/trait.From.html), [Rust associated functions](https://doc.rust-lang.org/reference/items/associated-items.html), [C++ converting constructors](https://eel.is/c++draft/class.conv.ctor). These are limited analogies; Rust conversion traits are not this proposal's implicit constructor/source/inbound ranking, and C++ eligibility is not restricted to this proposal's exactly-one-parameter form. The performance discussion describes production mechanisms, not interpreter benchmark results.

## Validation boundaries

Tests cover binding/cast/call/return/field/array/tuple boundaries, defaulted and optional parameters, overload order, exact numeric matches, failed pure conversions, source/inbound precedence, ambiguity before field effects, retained bodies after prototype mutation, private state/new.target, captured parameter types, Modules, Function/eval, foreign Realms and ordered rests. Effectful parameter defaults remain compile-time errors rather than being used to test speculative runtime effects.

A separate baseline witness remains: a closure returned from a factory, with or without generic parameters, can lose the local class used by its receiving annotation and throw `ReferenceError: "P" is not defined`. A corresponding nested call while the factory frame is active succeeds. That closure/captured-type dependency is not repaired by constructor applicability and is retained in the handoff.
