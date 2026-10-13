# Domain-preserving range selection

Intersection, containment and emptiness need ordering, not arithmetic. They must compare stored endpoints without replacing them by Numbers or treating unsupported values as missing bounds. This follows the endpoint-domain distinction in [range-membership.md](range-membership.md).

The implementation audit found that intersecting two byte ranges produced Number endpoints, wide BigInts rounded, and intersecting class ranges could return a full range. Range containment and emptiness used the same lossy or missing-value assumptions. A separate probe confirmed domain/precision loss in scaling and iterator consumers; those arithmetic paths are retained for the next increment, not certified here.

## Requirements

- Preserve selected endpoint values, domains, signed zero, class payloads and existing reference liveness. Selection does not convert or manufacture a reference capability.
- Preserve shape and actual per-endpoint bounds; an omitted endpoint alone is unbounded. An empty range is still a range with its original endpoints.
- Retain a declared union domain even if selection happens to choose narrower values. Full ranges retain meaningful domains without endpoints to recover them from.
- Make result-domain calculation independent of operand order and grouping. Keep uncontextualized `..` neutral.
- Compare numeric mathematical values exactly and use only declared ordering for class endpoints. Do not invoke `valueOf`, `toString`, public endpoint getters or unrelated operators.
- Propagate abrupt comparison completion before publishing a result; preserve the input ranges.
- Require no enumeration or proof over all contained values.

## Result-domain directions

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and disposition |
|---|---|---|---|---|---|
| Rebuild all results as Number ranges | Uniform representation, but autocomplete and generic consumers lose the original endpoint type. Wide values silently change. | Rust `Range<Idx>` retains `Idx`; it offers no precedent for erasing selected values to floats. | Simple arithmetic path, but boxes or converts typed endpoints unnecessarily. | Rounding changes point sets and class endpoints cannot be represented. | **Rejected.** These are measured failures. |
| Preserve only the receiver's domain, converting selected endpoints | Predictable receiver-style signature, but swapping operands can change results or cause conversion failure. | Rust's typed fields cannot silently hold another endpoint type. An explicit conversion would be a distinct operation. | One domain guard; conversion adds checks and may dispatch user code. | Violates commutative domain calculation and changes selected values. | **Rejected.** An intersection is selection, not conversion. |
| Infer solely from selected values | Precise-looking autocomplete, but declared union domains narrow unpredictably and full ranges lose all domain evidence. | Rust's field type is not re-inferred when a value changes. | Up to two value-type reads; full results need a special fallback. | Contradicts declared-domain retention and makes identity depend on which endpoint wins. | **Rejected.** Selected values are insufficient provenance. |
| Require identical domains for every operand | Simple result type, but mutually ordered numeric ranges and erased full-range identities need manual reconstruction. | Closest to a same-`Idx` Rust API, although Rust has no standard interval intersection operation. | Fast identity check; specialization is straightforward. | Sound but unnecessarily restrictive for the existing `RangeBounds<T>` view and cross-domain ordering. | **Rejected.** The type system already represents a union of endpoint domains. |
| Join operand domains, with full-any neutral | Same-domain intersections retain `T`; heterogeneous results honestly expose a union in autocomplete. A wrong assumption is visible at the next typed boundary. | Rust supports explicit endpoint typing and cross-type `contains` comparisons; it does not directly determine this union rule. This proposal has reified union types. | Interned domain join plus at most two endpoint selections. Common same-domain calls specialize to an identity guard; no shape mutation or enumeration. | Preserves endpoint values, declared domains, full-range identity and order/grouping-independent domain calculation. | **Recommended.** Join the domains independently of which endpoints are selected. |

For domains `T` and `U`, intersection's domain is their canonical union. A full range with domain `any` contributes no domain constraint and is neutral; a full range with a specific domain participates normally. Thus two `RangeFull<uint8>` values intersect to `RangeFull<uint8>`, and `(..).intersect(bytes)` retains the byte range's domain. A non-full range with explicit domain `any` is not neutral.

The signature is conceptually `intersect<U: type>(other: RangeBounds<U>): RangeBounds<T | U>`, subject to mutual ordering and the full-any identity rule. The concrete shape and bounds follow the selected endpoints. This is not generic covariance or retagging either input.

## Ordering directions

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and disposition |
|---|---|---|---|---|---|
| Use host Number comparison for all endpoints | Familiar numeric results until a large integer changes or a class becomes unbounded. | Rust compares its actual endpoint type. | Cheap but requires lossy conversion. | Loses precision and confuses unsupported values with absence. | **Rejected.** |
| Invoke general coercive relational operations | Supports objects with conversion methods, but ordinary conversion side effects become part of range operations. | Rust uses explicit comparison traits; implicit string/number coercion is not the analogy. | Polymorphic coercion calls hinder IC specialization. | The proposal's `Ordered` contract is a declared `<`, not `valueOf` or `toString`. | **Rejected.** |
| Enumerate or materialize values to decide selection/subsets | Sometimes intuitive for tiny integer sequences; fails for continuous or unbounded intervals. | Rust distinguishes ranges and iterator adapters. | Unbounded cost and allocations; cannot represent arbitrary ordered intervals. | Not defined for all shapes and ordered types. | **Rejected.** |
| Compare exact numeric values, otherwise dispatch declared `<` | Numeric precision is preserved; class authors see only their chosen ordering operator. | Closest to Rust's comparison-trait approach, with this proposal's total-order requirement instead of Rust's partial-order behavior. | Numeric tag fast paths and bounded comparisons; class calls use normal operator dispatch. Exact wide-integer paths have operand-size cost. | Preserves domains, avoids coercion, and distinguishes absence from a present value. | **Recommended.** An incomparable pair throws; NaN is not contained. |

Comparison returns less, equivalent or greater. Numeric equivalence is exact mathematical equality, including signed zeros. For class endpoints, call the left `<` first; if false, call the right `<` with operands exchanged. If both are false, totality makes them equivalent. This adds no proof that arbitrary user operators obey total-order laws; that remains their existing contract. Abrupt completions propagate immediately.

Intersection selects the greater lower endpoint first, then the lesser upper endpoint. At equivalent endpoints it keeps the left operand's endpoint value and makes the bound open if either input bound is open. This deterministic representative rule does not equate distinct class payloads or signed-zero representations: commutativity and associativity describe the **point set**, while domain joins are also commutative and associative. The alternative of manufacturing a canonical endpoint representative was rejected: it would need a new ordering over payloads or reference identities unrelated to `Ordered`, extra work in real engines, and potentially discard class state. Canonical domain joins do not require canonicalizing values.

Containment first determines whether the inner range is empty; if so it succeeds. Otherwise it checks lower-bound coverage, then upper-bound coverage, stopping at the first failure. Equivalent endpoints account for open/closed bounds. Emptiness compares two present endpoints and is true when descending, or equivalent with either bound open. A one-ended numeric range is also empty when its only bound excludes positive infinity at the start or negative infinity at the end; closed extremes remain nonempty. A missing endpoint does not compare as `undefined` and a class endpoint does not become missing because it has no numeric representation.

## Examples

```js
const a: Range.<uint8> = 0..<10;
const b: Range.<uint8> = 2..<8;
Reflect.typeOf(a.intersect(b)) === (type Range.<uint8>); // true
Reflect.typeOf((..).intersect(a)) === Reflect.typeOf(a); // true
const full: RangeFull.<uint8> = (..);
Reflect.typeOf(full.intersect(full)) === (type RangeFull.<uint8>); // true
(9007199254740992n..=9007199254740992n)
  .contains(9007199254740993n..=9007199254740993n); // false
```

## Sources and verification scope

[Rust Range](https://doc.rust-lang.org/std/ops/struct.Range.html) exposes typed endpoints and permits `contains<U>` with mutual comparison bounds. [Rust RangeFull](https://doc.rust-lang.org/std/ops/struct.RangeFull.html) is a nongeneric endpoint-free type, so it motivates an identity but does not prescribe this proposal's reified full-range domains. [Rust Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.step_by) separates step/take adapters from endpoint selection. Rust has no standard interval-arithmetic/intersection API whose result-domain policy could be imported unchanged.

This increment covers the ordering-only operations and their comparison failures. Scaling, interval arithmetic, take/drop normalization, step/reverse and implicit iterator values require their own arithmetic-domain audit. The interpreter's old Number-based arithmetic paths are not certified by the selection repair.

The later [scaling audit](range-scaling.md) specifies numeric scalar arithmetic and the extremal-infinity emptiness correction. Iteration and count/adapter rules are in [range-iteration.md](range-iteration.md) and [range-adapters.md](range-adapters.md). Binary interval arithmetic remains open.
