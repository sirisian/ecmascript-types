# Numeric range iteration

The ordering-only repair in [range-selection.md](range-selection.md) preserves endpoints, but an iterator must also manufacture values between them. The implementation audit found plain Number results from typed ranges, repeated rounded values above 2^53, refusal of BigInt steps, and NaN from an infinite step. This note settles numeric cursor behavior separately from endpoint-domain reflection.

## Requirements

- Preserve the selected item type and the range value; separate cursors must be independent.
- Respect both bounds, including open starts, descending empty ranges, excluded stops and closed carrier maxima.
- Never wrap, silently lose an integer, repeat a rounded value indefinitely, or mistake a present unsupported endpoint for an omitted one.
- Validate the step without coercing objects or consulting public endpoint getters. A step is a positive difference; reverse traversal is a separate operation.
- Specify arithmetic and failure state sufficiently to produce the same observable sequence in another engine.
- Preserve the existing element-fit judgments, metadata boundaries and reference-liveness rules. A numeric result grants no reference capability.

The numerical rules below cover `number`, `bigint`, sized `int`/`uint`, and binary `float16`, `float32`, `float64`, including numeric parameterizations at their existing admission boundary. Other arithmetic domains need their own progression operations. Their ordered endpoints remain valid ranges; ordering alone supplies no numeric iterator implementation.

## Item type

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Always return Number | Common small loops work, but autocomplete promises typed values that reflection and generic consumers never receive. | Unlike Rust's `Range<Idx>::Item = Idx`; Python integers do not justify float erasure. | One representation, but wide integers require lossy conversion. | Changes values and loses the declared item contract. | **Reject.** Measured defect. |
| Infer from the traversal origin on every operation | Easy forward rule; reversing a mixed range can change the item type. | Typed range APIs keep one item type during reverse traversal. | One tag check at acquisition. | Conflicts with stored context and forward/reverse agreement. | **Reject.** |
| Require identical endpoint types | Simple autocomplete but rejects ordinary `0..<array.length` and existing mixed endpoint domains. | Same-type range APIs are only a partial analogy: this proposal already separates endpoint domains and iteration fit. | Cheap identity check. | Too restrictive for existing commitments. | **Reject.** |
| Stored concrete numeric domain, otherwise typed start, typed end, then Number | Stable typed results; a domain error appears at the failing request. The same choice applies to reverse. | Retains the typed-item principle; the fallback is specific to this proposal's contextual typing. | Select once per iterator, specialize on numeric tag/width; no range shape mutation. | Matches typed-endpoint precedence already used by checking and keeps reflection independent. | **Recommend.** A union endpoint domain does not choose a new union arithmetic. |

A concrete numeric parameterization uses its numeric base for progression and its full type at item admission. Where the stored domain is a union or `any`, the typed-endpoint fallback applies. Explicit iteration contexts still check the elements a consumer receives; they do not retag the stored range. This increment does not certify every static adapter carrying an iteration context.

## Step domain and direction

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Require the difference to fit the item type | Apparently simple `step(by: T)`, but a byte range cannot use stride 1000 although it yields only its first byte. | Rust's `step_by` takes a count separate from `Item`; Kotlin also distinguishes stride from endpoints. | Adds a needless narrowing conversion. | Confuses a difference with a yielded item. | **Reject.** |
| Coerce any input or accept signed/nonfinite steps | Familiar coercion hides mistakes; negative steps conflict with descending ranges being empty. | Python permits signed steps; Kotlin spells descending progression separately. Neither decides this API. | User conversion calls and polymorphic ICs on acquisition. | Infinity currently produces NaN; zero or wrong direction may not progress. | **Reject.** |
| Positive finite numeric difference, integral for integer carriers; separate reverse | A wrong step fails immediately; autocomplete can expose a numeric difference independently of the item width. | Rust rejects zero; its count-based skipping is not this API's fractional arithmetic. | Numeric tag/finite/integral checks once; widened exact stride can stay outside the item width. | Preserves the interval direction and supports BigInt and typed steps without coercion. | **Recommend.** Invalid kinds, signs, infinities and NaN throw TypeError at acquisition. |

The numeric `step` overload therefore accepts a numeric difference independently of the item type, including a wider integer difference. Integer carriers also require an integral origin. `reverse()` uses unit descent from the stored upper endpoint, with the upper bound deciding whether the initial index is zero or one. Its stop is the lower endpoint and its bound. An absent lower endpoint permits reverse traversal; an absent or infinite upper origin does not. Ordinary forward iteration still needs a start.

## Arithmetic

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Repeated native-width addition, including integer wrap | Looks like manual arithmetic but can repeat values, wrap into the interval, or accumulate floating error. | Ordinary integer operators are not a range-iteration contract. | Cheapest loop, but wrong even in common boundary cases. | Violates exact integral stepping and the existing indexed expression. | **Reject.** |
| Widen every result to Number or BigInt | Avoids some overflow but changes observable types and boundary behavior. | Python's unbounded integers suit Python's item type, not a declared byte. | Boxing/type changes weaken specialization. | Breaks the selected item contract. | **Reject.** |
| Round every intermediate to the item width | Familiar typed arithmetic, but a float16 index/stride can lose progress before the intended item calculation. | Native-width arithmetic is a plausible alternative; no cited range API determines fractional stepping here. | Narrow arithmetic may be fast, but conversions are needed for the index. | Does not satisfy the exact integral rule; adds avoidable rounding to fractional progression. | **Reject as the common rule.** |
| Exact integers; fractional binary64 multiply then add, then item-width rounding | Integral values stay exact; fractional values follow the documented indexed expression with one final width conversion. | Python demonstrates exact indexed integer progression; floating policy is an explicit proposal choice. | Integer fast paths may use checked machine arithmetic; wider cases use exact arithmetic. Float16/32 values extend exactly to binary64. | Deterministic without accumulated addition error; integer-to-float items must be exactly representable. | **Recommend.** No fused multiply-add substitution if it changes results. |
| Exact rational evaluation for every fractional expression | Could be more accurate mathematically, but changes existing Number-expression results. | Exact rational sequences are a different numeric contract. | Operand growth and rounding work beyond the ordinary float path. | Conflicts with the already specified binary floating expression's operations. | **Reject for this API.** An explicit rational domain is separate work. |

If the origin and step are integral, evaluate `start + n * step` in the mathematical integers, even for a float item type. Do not impose the carrier's arithmetic wrap rule on the cursor. Otherwise convert the origin and difference exactly to binary64, multiply the binary64 index by the difference, add the origin, and round to the item width. A wider integer operand that cannot enter that fractional calculation exactly throws RangeError at acquisition. Silently rounding it, changing every fractional sequence to rational arithmetic, and allowing an engine-dependent extended format are rejected: they respectively hide input loss, alter the established expression, and lose portability; exact-entry validation has bounded acquisition cost and no shape effects.

| Fractional input direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Round a wide integer difference to binary64 | Accepts more calls but silently changes their stride. | Rust has no fractional range contract authorizing this conversion. | One conversion, no extra guard. | Loses an explicitly supplied difference before iteration. | **Reject.** |
| Use a wider or exact format for those calls | More inputs work, but expression results depend on the operand path or engine precision. | A rational arithmetic API is a distinct contract. | Additional arithmetic modes and specialization. | Changes the specified binary64 expression unless all calls change with it. | **Reject here.** |
| Require exact entry to the fractional path | An unsupported combination fails before a cursor is returned; explicit conversion makes the choice visible. | Consistent with checked numeric boundaries; this floating rule is not inherited from Rust. | One exactness guard at acquisition, no user shape effects. | Preserves the supplied origin and difference in the selected arithmetic. | **Recommend.** Integral progressions still accept arbitrary-width integer differences. |

At index zero retain the stored origin, including negative zero. An open start begins at index one and requires the first item to advance beyond the excluded origin. After acquisition validation, an already-empty interval is exhausted without attempting a rounded candidate; a tiny step cannot turn an empty interval into a progress error. For implicit iteration, present finite endpoints must be integral. Infinite stops retain their sign: a positive infinite forward stop does not bound a finite prefix, while a negative infinite forward stop is already behind the origin.

## Bound checks and failure

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Admit the next value before checking the bound | A closed byte maximum unexpectedly throws on the following request. | Checked successor operations distinguish representability from iteration completion. | Unnecessary conversion/check at exhaustion. | Requires an excluded value to inhabit the item type. | **Reject.** |
| Compare the unrounded floating expression, or require both rounded and unrounded membership | A value equal to an open stop can escape, or a valid rounded closed stop can disappear. | Rust's integer stepping supplies no fractional rounding precedent. | Avoids or adds one comparison but cannot prove membership of the actual result. | Uses a different value from the item received by the consumer. | **Reject.** |
| Clamp, skip unrepresentable values, or silently finish at carrier exhaustion | Avoids errors but changes a requested lattice, truncates an unbounded request, or hides the first lost integer. | Rust's checked and unchecked stepping APIs explicitly distinguish overflow; they do not prescribe this policy. | Cheap clamping; skipping can be unbounded work. | Violates unit/integer progression and hides failure. | **Reject.** |
| Check bounds on the candidate; admit and require strict progress; throw and close on failure | Closed maxima finish normally; an invalid in-range request fails once and a retry is done. | Fused iterators motivate stable exhaustion; the error-then-done policy is specified here. | One endpoint comparison and checked admission per item, often eliminated by loop proofs. | Preserves bounds, exactness and a defined post-failure state. | **Recommend.** |

For an integral progression compare the exact candidate with the stop before carrier admission. For a fractional progression compare the rounded item, because that is the value the consumer would see: rounding onto an open endpoint excludes it, and rounding onto a closed endpoint admits it. Comparing a different, pre-rounding value is rejected because it can both leak an excluded item and discard an admitted one; testing both would add a condition unrelated to membership. After a closed endpoint is returned the cursor is exhausted immediately.

An in-range item that overflows or loses integral precision throws RangeError. A fractional item must be finite and strictly advance; otherwise it also throws RangeError. Exhaustion and abrupt item admission leave the cursor exhausted. Keeping the failed index for repeated errors, advancing past the failed value, and rolling back to a prior item are rejected: each makes retries ambiguous or silently changes the sequence, with no type-system or performance benefit. A fresh iterator remains available from the unchanged range.

Numeric parameterizations retain their existing checked admission and error behavior. Admission may not change the computed numeric value or its zero sign; otherwise the iterator closes with RangeError. Silently dropping metadata would violate the item type, while allowing an implicit metadata cast to rescale a computed item would violate bounds and progression. Both are rejected. Admission's existing hooks may run, but a reentrant request on the same active cursor throws TypeError; after an uncaught abrupt admission the cursor is done. This uses an internal execution flag, with no user-object shape changes. Merely reading a cursor does not establish the liveness of any borrowed storage later used by the program.

| Refinement / reentry direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Drop numeric metadata or trust an unvalidated tag | Autocomplete promises a refinement that yielded items may violate. | Rust's ordinary numeric range types have no direct analogue of these user-defined runtime metadata hooks. | Cheap but skips required admission. | Violates the item domain and existing validation semantics. | **Reject.** |
| Allow admission to move the numeric value | A cast can change a stride or cross the stop after its bound check. | Ordinary conversion is not a progression operation. | Reuses the boundary but loses useful range proofs. | Breaks the computed sequence and membership. | **Reject.** |
| Allow reentry to read or advance incomplete state | A validation hook can consume or duplicate the item currently being checked. | Generator execution guards are the closer analogy; Rust's borrowing prevents many same-cursor aliases but does not directly specify JavaScript callbacks. | Avoids a flag at the cost of reentrant state mutation. | Exposes an uncommitted index or previous value. | **Reject.** |
| Preserve checked admission, verify numeric identity, reject active reentry | Refinements keep their hooks and diagnostics; retries and callback attempts have explicit outcomes. | Uses this proposal's existing admission contract and a generator-like execution guard, without importing Rust ownership rules. | Existing hook cost only for refinements; constant internal state, no range shapes or IC invalidations. | Maintains the item type, progression, error state and independent liveness checks. | **Recommend.** A caught inner reentry error permits the outer admission to finish. |

## Sources and scope

- [Rust Range](https://doc.rust-lang.org/std/ops/struct.Range.html): typed endpoints, typed iteration and reverse support.
- [Rust Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.step_by): a separate positive skip count and iterator adapters; fractional value steps are outside that analogy.
- [Rust Step](https://doc.rust-lang.org/std/iter/trait.Step.html): checked successor/predecessor operations report representability failure. The trait is experimental; its unchecked/default overflow choices are not adopted.
- [Kotlin ranges and progressions](https://kotlinlang.org/docs/ranges.html): progression stride and separately spelled descending traversal.
- [Python ranges](https://docs.python.org/3/library/stdtypes.html#ranges): integer indexed progression with signed steps; this proposal retains separate reverse traversal.

No endpoint enumeration is required to construct a cursor. A production engine can lower ordinary bounded loops to checked machine counters and specialize on an immutable carrier descriptor; the proof-of-concept's internal WeakMap is not a production layout prescription. Reference location validation remains at the existing accesses.

This increment covers implicit numeric iteration, numeric `step`, `reverse`, and the helpers delegating to these cursors. It does not repair `length`, range-returning `take`/`drop`, scaling, interval arithmetic, general custom arithmetic domains, or all static iterator-result adapters. Those remain independent audits.
