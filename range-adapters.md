# Numeric range counts and adapters

`length`, `take`, and `drop` operate on the exact unit lattice of the numeric ranges covered by [range-iteration.md](range-iteration.md). They do not traverse an iterator. The audit found lost endpoint types, rounded `uint64` boundaries, and a count of five for a three-member wide integer range. This note replaces the implementation's closed-open normalization and unspecified count representation.

## Required outcomes

- Preserve the source's endpoint domain, unchanged endpoint values (including negative zero), and omitted bounds. A returned range is reusable and grants no reference authority.
- Compute with mathematical integers, without item-width wrap or Number rounding. Closed carrier maxima must not require a successor.
- Specify observable bounds, count types, count admission, and failure order. Autocomplete must not promise an endpoint domain that runtime erases.
- Construct in constant work with respect to the number of lattice members. Arbitrary-width integer arithmetic still costs work proportional to operand size.
- Keep endpoint admission separate from iteration. A range's lattice cardinality does not prove that every intermediate value passes the item type's representation or metadata checks.

## Count representation

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Count in the item type, with wrap or float rounding | A byte range appears to contain zero rather than 256 values; autocomplete confuses items with counts. | Rust uses `usize`, not `Item`, for iterator counts. | Cheap native arithmetic but incorrect overflow. | Loses cardinality. | **Reject.** |
| Always Number, throwing if not exactly representable | Convenient for ordinary JS arithmetic, but large virtual intervals cannot report their count. | Python also bounds `len` despite larger range endpoints; that is a limit, not a requirement here. | Checked native arithmetic fast path; exact intermediate still needed. | Can be sound with an explicit error, but unnecessarily partial. | **Reject.** |
| Always the container index type `uint64`, throwing on overflow | Familiar to array consumers and stable in autocomplete; a full `uint64` interval itself has one more member than it can report. | Rust's `ExactSizeIterator` has a `usize` ceiling; not every iterator implements it. | Checked machine arithmetic; no general boxing fast path required. | Sound but conflates a virtual interval with allocatable storage. | **Reject for range cardinality.** Container counts keep their existing contract. |
| Number for small counts, BigInt for large counts or selected endpoint families | Preserves some existing code but a value or generic instantiation changes arithmetic operators' admissible operands. | Neither Rust's fixed `usize` result nor Python's single integer type justifies this union. | Polymorphic result and arithmetic ICs. | Exactness is possible, but no stable common count type. | **Reject.** |
| Always `bigint` | Stable `length: bigint`; conversion to a container count or Number is explicit and checked at that boundary. Even a full wide integer domain has an exact count. | Rust separates count from item; unlike Rust's machine count, JS already has a portable arbitrary-width integer. | Small-count optimized representations are possible; wide arithmetic and BigInt results have their normal costs, no range shape mutation. | Every finite integral pair has one exact nonnegative result. | **Recommend.** This intentionally changes Number-range `length` from the old implementation's Number result. |

For two finite integral supported numeric endpoints, `length` is `max(0, end - start + 1 - startIsOpen - endIsOpen)`, returned as a BigInt. Mixed supported endpoint representations compare and subtract exactly. Missing, infinite, fractional, or unsupported endpoints throw TypeError. This is lattice cardinality, not the size of a successfully materialized array or the continuous interval. It neither enumerates nor invokes metadata validation. Iteration can still fail on an intermediate unrepresentable or inadmissible item. `isEmpty` retains its ordering-based interval contract.

## Count arguments

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Keep `ToIntegerOrInfinity`, including objects, truncation and Infinity | Looks like ordinary Iterator helpers, but fractional mistakes truncate and object conversion runs before boundary work; BigInt counts fail. | Base JS helpers are the closer precedent; Rust accepts a `usize` count without dynamic coercion. | Conversion callbacks and polymorphic call sites. | Cannot represent the exact range-count contract and has incoherent infinite/BigInt combinations. | **Reject for range-returning adapters.** Iterator helpers keep their own rules. |
| Require only `uint64` or only BigInt | Crisp parameter type; ordinary `take(3)` or another exact numeric count needs an avoidable conversion. | Rust's typed count is a partial analogy; this proposal accepts numeric literals in typed positions. | Simple tag check, then arithmetic. | Correct but needlessly excludes exact available counts. | **Reject.** |
| Accept nonnegative finite integral supported numeric values without coercion | `take(3)`, `take(3n)`, and a typed numeric count agree. Wrong kinds and values fail immediately. | Retains Rust's integral count principle without its machine width; differs deliberately from JS Iterator coercion. | Tag, integral and sign checks once; no callbacks or range shape effects. | Exact count independent of item width and container limits. | **Recommend.** Wrong kinds (including omitted/undefined) throw TypeError; negative, fractional, NaN or infinite numeric counts throw RangeError. Negative zero is zero. |

Validate the receiver, then endpoint suitability, then the count, even for a zero count or empty input. Ordinary argument-expression evaluation still occurs before the call. No public endpoint property or user conversion method participates. Endpoint-first versus count-first are both implementable; endpoint-first is selected because it preserves the existing acquisition order and reports an unavailable numeric operation before its count. Skipping validation on fast exits is rejected: it makes a wrong call succeed depending on unrelated bounds. These choices require only fixed guards and no effects journal.

## Boundary representation

Let `first = start + (startIsOpen ? 1 : 0)`. For a finite end let `last = end - (endIsOpen ? 1 : 0)`. Use exact integers throughout.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Normalize every result to closed-open | Familiar small endpoints; fails when the exclusive successor is outside the domain or metadata. | Rust's `take` returns an iterator, not a normalized range; it supplies no successor guarantee. | Often two fabricated endpoints and admissions. | Cannot represent closed maxima without widening or error. | **Reject.** |
| Use closed-open when possible, closed otherwise | Saves common reflected shapes but makes bounds depend on representability and potentially effectful admission failure. | No cited API specifies this fallback. | Extra paths; speculative metadata checks would need an effects policy. | Can be specified for primitives, but fallback after hooks is not harmless. | **Reject.** |
| Widen the range domain or return an iterator | Avoids endpoint limitations but changes the advertised domain or removes `contains`/`intersect` from the result. | Rust uses iterator adapters; that is not this proposal's existing range-returning API. | New representations or iterator state and less stable specialization. | Abandons domain preservation or interval closure. | **Reject.** |
| Preserve unmodified bounds; take closes at the last selected value, drop opens after the last removed value | Exact reflected bounds, including at carrier maxima. `take(3)` of `0..<10` is `0..=2`; `drop(3)` is `2<..<10`. | Uses Rust's typed-item principle; the endpoint representation is specific to this proposal's four bound forms. | At most one fabricated endpoint and admission; fixed immutable range layout. | No successor needed; domain and integer sequence preserved. | **Recommend.** Fractional membership between adjacent integers follows these written bounds, not the old normalized representation. |

After validation:

1. `take(0)` returns `start..<start`; `drop(0)` copies the original range.
2. A negative infinite end or `first > last` has no lattice members; positive-count adapters copy it unchanged.
3. If a finite lattice has `m` members and `n >= m`, `take(n)` copies the original range and `drop(n)` returns `start..<start`.
4. Otherwise compute `cut = first + n - 1`. `take(n)` retains the original start and its bound, with closed end `cut`. `drop(n)` uses open start `cut`, retaining the original end and its bound, including absence or signed infinity.

The canonical empty result uses the already stored start twice. Using a successor of the last member is rejected for maximum values; searching for another empty anchor needlessly invokes admission. Preserving the original empty lattice and whole-input result avoids fabricating irrelevant endpoints and retains useful bound information. These choices are O(1) in member count and preserve reference-liveness rules; Rust's empty iterator does not expose corresponding endpoint fields.

Every result stores the original endpoint domain, including an inferred union, even if one endpoint is replaced. A new cut uses the original range's numeric item type selection, exactly as iteration does. A cut equal to the stored start reuses that value only when it already has the selected item type, including its zero sign and metadata, without revalidation. Reusing a plain start when only the old end supplied a typed item carrier is rejected: a one-item take would erase that carrier. Always admitting the cut is also rejected because it unnecessarily repeats hooks on an unchanged value. The selected type-and-value reuse test is constant work, preserves autocomplete and metadata contracts, and has no shape effects; Rust has no mixed-endpoint fallback to settle this choice. Otherwise the cut must fit the item carrier exactly, then pass its full numeric parameterization's existing admission. No wrap, rounding, rescaling, or change from computed positive zero to negative zero is allowed. Representability or numeric-identity failures throw RangeError; existing metadata exceptions propagate. Publish the result only after successful admission. Neither failure nor success mutates the source; user hook side effects retain their ordinary semantics.

## Admission and laziness

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Enumerate every selected or skipped value, including metadata checks | Resembles consuming an iterator, but a billion-element `drop` now executes a billion hooks and may never return. | Rust adapters are lazy; advancing may skip efficiently but does not prescribe this range-value operation. | O(n), defeats algebraic range construction and cannot optimize arbitrary callbacks. | Confuses boundary construction with traversal. | **Reject.** |
| Trust or strip metadata on new endpoints | Fast, but the result can claim an endpoint it does not admit. | Rust primitive ranges have no direct user-validation analogue. | Skips hooks and required checks. | Breaks existing type admission. | **Reject.** |
| Validate only manufactured endpoints; leave per-item checks to traversal | Predictable boundary cost; `length` and `drop` are not proofs that skipped items could have been yielded. | Closest to lazy typed adapters, with this proposal's explicit metadata boundary. | Fixed number of admissions, immutable source, no IC invalidation. | Preserves endpoint and iteration contracts separately; borrowed locations still require their existing liveness checks. | **Recommend.** |

## Sources and remaining scope

- [Rust Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html): `take` and `skip` accept `usize` and return adapters, not ranges with observable new bounds.
- [Rust ExactSizeIterator](https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html): exact length is limited by `usize`; not all iterators qualify.
- [Python ranges](https://docs.python.org/3/library/stdtypes.html#ranges): large integer endpoints and constant storage do not guarantee that `len` fits its supported size.

Numeric scaling is now specified in [range-scaling.md](range-scaling.md). Interval arithmetic, custom arithmetic domains, and complete static range-adapter result typing remain separate audits. The `length` result contract is `bigint` also for statically known range-family property reads. This change does not claim closure of the broader checking or inference work packages.
