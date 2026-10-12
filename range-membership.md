# Range domains, membership and iteration

A range's stored endpoints and its produced iteration values answer different questions. `Range.<T, S, E>` exposes `start: T` and `end: T`; its parameter therefore describes stored endpoints, including excluded endpoints. An iteration consumer instead asks whether every value it would receive fits its element type. These judgments must not be interchangeable.

The previous implementation matched only range shape and bounds: `(0..<5) is Range.<string>` was true. An annotation `const r: Range.<uint8> = 0..<5` also left both endpoints as Number values, and reflection reported `Range<number>`. These are implementation defects, not useful structural compatibility.

## Required properties

1. Reading an endpoint through a typed range returns a value admitted by its declared endpoint domain.
2. Membership neither converts nor stamps its operand, calls ordering/coercion operators, nor enumerates an interval.
3. A concrete specialization retains generic invariance. Merely fitting numerically is not already having a numeric type.
4. Shape and bound arguments remain independent checks. An empty interval still has endpoints and is not a full range.
5. Iterating `0..<256` at `uint8` remains valid; 256 is not produced. No excluded bound is wrapped, clamped or rewritten.
6. Formation evaluates each written endpoint once, in source order. A failed check publishes no range and restores contextual state.
7. A full range has a range type. Its missing endpoints are not `undefined` endpoint values.
8. References keep the existing storage and liveness rules. A membership check neither snapshots class endpoints nor creates a reference capability.

## Directions considered

All directions were assessed against the same five criteria. Implementation effort does not determine the choice.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and disposition |
|---|---|---|---|---|---|
| Keep shape/bounds-only membership | Very permissive, but autocomplete promises endpoints and methods the value cannot supply; a wrong guess fails later. | Unlike Rust's typed `Range<Idx>` fields. JavaScript prototype identity alone cannot describe generic arguments. | Cheap tag checks. | Accepts unrelated endpoint domains; contradicts annotations and reflection. | **Rejected.** The measured `Range<string>` result is a false claim. |
| Define membership by produced-element fit | The byte-loop intuition extends to range annotations, but `.end` can contradict its advertised type and empty ranges pass arbitrary types. | Rust uses its endpoint type; iterator adapters are a separate abstraction. | General ordered types need semantic proofs or enumeration; user comparisons introduce effects and make caches value-dependent. | Fit does not prove a stored endpoint contract or generic identity. Unbounded/noniterable ranges are not empty endpoint records. | **Rejected.** Keep this judgment for iteration consumers only. |
| Preserve element-based `T`, widen endpoint members or add a separate bound representation | Retains old sentinel annotations but gives `.end` a wider type or a wrapper in autocomplete. Every endpoint consumer must handle that distinction. | Rust's range has no such wrapper. .NET's `Range` does separate index positions from collection element types, but is not an ordered generic interval. | A second domain/tag affects layout, arithmetic and IC specialization; compact integer loops could still optimize. | Can be sound if endpoints, comparisons, arithmetic, reflection and metadata are redesigned together. It changes the published `start: T`/`end: T` contract. | **Rejected.** A second abstraction for every interval is unnecessary when the consumer already supplies iteration context. |
| Make every typed range invariant and reject all contextual formation | Endpoint types are truthful, but even `let r: Range<uint8> = 0..<5` requires manual endpoint casts. | Less ergonomic than Rust, whose numeric literals use context. | Simple fixed-domain representation. | Sound but inconsistent with the proposal's literal propagation. | **Rejected.** Fresh literals should use their actual construction context. |
| Give every shape endpoint-wise covariant membership | Truthful reads and useful union views; no stored domain needed. | Immutable covariance has analogies, but Rust's `Range<Idx>` does not establish this proposal's generic variance rule. | Two endpoint checks; no materialization. | Concrete `Range<T>` also exposes operations accepting/returning `T`; blanket covariance changes the existing invariant generic contract. It confuses identity with interface admission. | **Rejected for concrete shapes.** Endpoint admission is appropriate for the shape-independent `RangeBounds<T>` view. |
| Separate invariant endpoint-domain values from iteration admission | An annotation means what endpoint autocomplete says. A byte loop remains natural. A stored byte interval ending at 256 must use a wider endpoint domain or the inclusive `0..=255` spelling. | Closest to Rust's typed endpoint fields and contextual literals; this proposal additionally allows a consumer to convert/check produced numeric values. | Fixed shape/bound/domain guards and at most two endpoint checks; no enumeration, proxy reads or added mutable object shapes. Contextual construction can be scalar-replaced. | Preserves endpoint contracts, generic identity, value order, and reference liveness while retaining produced-element fit separately. | **Recommended.** The compatibility cost is explicit and local to previously misleading annotations. |

## Selected rules

A fresh range expression at a concrete range or `RangeBounds<T>` position reads each present endpoint at `T`, using ordinary literal propagation and checked boundary admission. Known failures are early errors, including in an unused function. A runtime-only endpoint is checked when evaluated; an excluded endpoint is checked too. No completed range is published after failure. A range arriving through a binding or a call keeps its domain; a position does not silently rebuild it.

Concrete shapes have invariant endpoint domains, in addition to their shape and bound arguments. Bare shapes and an `any` domain are erased membership queries. `RangeBounds<T>` is the endpoint-reading view: the shape must be a range and every present endpoint must already satisfy `T`. A full range satisfies that view vacuously. Neither view changes the value's concrete specialization.

A context-free expression preserves the domains of its evaluated endpoints. When they differ, its reflected domain is their canonical union, not whichever endpoint was examined first. For example, a plain Number start and a `uint8` end produce `Range<number | uint8>`; the typed endpoint can independently guide the consumer's iteration domain. This does not grant an ordering capability to arbitrary unions or relax the requirement that the endpoints be ordered with one another.

A full literal receives the endpoint domain of its construction context; without one it reports `RangeFull<any>`. Membership in `RangeFull<uint8>` does not specialize an existing uncontextualized full range. The `RangeBounds<uint8>` view needs no endpoint conversion and admits it.

Iteration positions (a loop binding, array/rest spread, or collection seed) supply the type of produced values, not an endpoint construction type. The existing first/last produced-element rule remains. A direct literal can consequently fit `uint8` iteration while its range value has Number endpoints. A concrete `RangeFrom<uint8>` can describe an unbounded interval; consuming its unbounded sequence at a bounded integer type remains invalid. Interval formation does not promise that arbitrary iteration can finish within its endpoint domain.

```js
const raw = 0..<5;
raw is Range.<uint8>; // false
raw is Range.<number>; // true
const bytes: Range.<uint8> = 0..<5;
bytes.end is uint8; // true
const stop: Range.<uint8> = 0..<256; // type error, including the excluded end
for (const b: uint8 of 0..<256) {} // valid: only 0 through 255 are produced
const full: RangeFull.<uint8> = (..);
Reflect.typeOf(full) === (type RangeFull.<uint8>); // true
```

This supersedes the old use of produced-element fit for **stored range annotations**, including empty and floating-point intervals. It does not change metadata's independent atom-admission and declared-field normalization rules. Metadata cannot use an excluded endpoint to bypass its field domain either.

## Prior-art sources and analogy limits

- [Rust `Range`](https://doc.rust-lang.org/std/ops/struct.Range.html) exposes both endpoints as `Idx`; `contains<U>` has mutual comparison bounds. Cross-comparison is not generic type identity.
- [Rust literal expressions](https://doc.rust-lang.org/reference/expressions/literal-expr.html) describe context-driven integer literal inference. This proposal additionally supports untyped Number values and checked numeric boundaries.
- [.NET `Range`](https://learn.microsoft.com/en-us/dotnet/api/system.range?view=net-10.0) represents start/end `Index` positions for slicing. It demonstrates a separate bound abstraction, not a generic ordered element domain.

## Verification boundary

The accompanying implementation and focused tests cover ordinary endpoint admission, concrete and interface membership, reflection, iteration-context separation and failure restoration. They do not certify every generic activation, full operator algebra, or all reference carriers. The broader checking, budget and specification-only audit gates remain open.
