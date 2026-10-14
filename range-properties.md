# Static range property contracts

A statically known built-in range contributes the property types below. The endpoint domain includes its full numeric metadata or class identity; it is not narrowed to a primitive base or widened to Number. These are normal-result contracts for public property reads, not permission to replace them with internal-slot reads or to elide a receiving boundary without a stability proof.

## Required outcomes

The same declared property must mean the same thing in dot access, a known computed String key, an optional chain and object destructuring. A finite key union joins all alternatives. A receiver union joins every arm's contribution under the existing member rules. The judgment must not evaluate a getter, order endpoints, inspect range members, specialize an iterator or grant reference authority. Known mistakes in unused bodies remain early errors.

Let `T` be the endpoint type, `B` the intrinsic `Range.Bound` enum, `I` the intrinsic `Range.Interval` enum, and `U` mean `undefined`.

| Static receiver | `start` | `end` | `startBound` | `endBound` | `interval` |
|---|---|---|---|---|---|
| `Range<T,S,E>` and its aliases | T | T | B (S if known) | B (E if known) | I (derived member if S/E known) |
| `RangeFrom<T,S>` | T | U | B (S if known) | U | U |
| `RangeTo<T,E>` | U | T | U | B (E if known) | U |
| `RangeFull<T>` | U | U | U | U | U |
| `RangeBounds<T>` | T or U | T or U | B or U | B or U | I or U |

Every shape contributes `boolean` for `isEmpty` and `isFull`, and `bigint` for a successful `length` read. The latter retains its runtime preconditions: an unbounded, fractional or unsupported range can still throw. A result type is not a proof of normal completion. Empty and descending ranges do not erase their exposed endpoint types.

A known bound ordinal selects an enum-based singleton type, retaining the intrinsic enum identity. The derived interval order is Closed, ClosedOpen, OpenClosed, Open for closed/closed, closed/open, open/closed and open/open. Unknown bound arguments retain the entire enum; the checker does not evaluate a generic argument to force a singleton. An erased `any` endpoint stays `any`, including the union-with-undefined absorption rule. This does not admit a bare generic spelling where the existing intrinsic-argument rules require `T`.

## Endpoint presence and absence

| Direction | Ergonomics / autocomplete / wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Leave all properties unknown/any | Reads work, but autocomplete loses the declared endpoint type and wrong annotations survive until execution or conversion. | Rust's `Range<Idx>` exposes typed endpoint fields. | Cheap checking; downstream guards remain necessary and optimizers lose facts. | Fails the declared property contract and early-error requirement. | **Reject.** |
| Treat every endpoint as T on every shape | Convenient, but absent endpoints can be consumed as actual T values. | Rust has different concrete range shapes; it does not put both fields on every shape. | Uniform static signature, unsound runtime assumption. | Contradicts observable absence. | **Reject.** |
| Use T-or-absence for every concrete shape | Safe but requires unnecessary narrowing even for a guaranteed endpoint. | Rust distinguishes concrete fields from the shape-independent RangeBounds interface. | More downstream nullable branches and less precise IC specialization. | Sound but discards supplied shape information. | **Reject.** |
| Preserve concrete presence and use optional endpoints only on RangeBounds | Autocomplete matches the known shape; a wrong guess fails before body entry. | Matches the distinction between Rust's typed concrete fields and generic bound queries, while keeping this proposal's own API. | Constant-time shape/type-argument lookup; no runtime shape changes or member enumeration. | Preserves domain identity, absence and reference liveness. | **Recommend.** |

For the absence sentinel, retain `undefined`. Keeping the old `null` signature would contradict the existing getters and tests. Returning both null and undefined would advertise a value the implementation never returns. Changing all getters to null would break established range behavior and ordinary JavaScript missing-value handling. Rust uses `Bound::Unbounded`, not either JavaScript sentinel; that is a useful explicitness analogy, not a reason to add a wrapper allocation or new public shape here. The selected sentinel uses existing values and union machinery, needs no runtime conversion, and preserves the existing liveness rules. The stale `null` sketches in `ranges.md` are corrected.

## Enum precision and identity

| Direction | Ergonomics / autocomplete / wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Report number/uint8 ordinals only | Exposes representation but loses the named Bound/Interval contract and enum alternatives. | Rust Bound is an enum rather than an unlabelled integer. Its variants carry endpoints, unlike this proposal's separate fields. | Simple numeric facts; downstream enum checks lose identity. | Conflates representation with the declared result type. | **Reject.** |
| Always report the whole enum | Useful autocomplete; loses a bound already fixed by S/E. | Rust's concrete syntax distinguishes inclusivity, but does not prescribe these value-generic singleton types. | Constant lookup, less precise downstream facts. | Sound but unnecessarily weak when the argument is already known. | **Reject as the only rule.** Retain it for unknown bound arguments. |
| Use the intrinsic enum identity, narrowed to the known member | Precise named alternatives and early errors without guessing generic values. | Rust's enum identity is the relevant analogy; singleton types are proposal-specific. | Existing literal/type records and constant ordinal mapping; no new runtime layout or IC behavior. | Preserves declaration identity and does not create a new enum per lookup. No liveness effect. | **Recommend.** |

A user class or type named `Range`, `Bound` or `Interval` retains its own identity and members. The property rule recognizes the resolved built-in family, not a source spelling. Do not install new bare global enum names to make the static lookup work.

## Access forms and gradual information

Special-casing dot access alone would make bracket access and destructuring loopholes in the same contract. Maintaining separate tables risks drift and duplicated cache inputs. Evaluating dynamic keys or getters to determine a type would introduce effects and source-position dependencies into checking. All are rejected. Use one property contribution for every statically established String key, joining finite key alternatives without evaluating them. Rust's field access does not offer JavaScript computed String keys, so its analogy stops at using the same declared member contract. Shared static lookup has bounded work, no hidden-class changes, no getter calls and no new reference authority.

Unknown computed keys retain ordinary gradual behavior. Unrelated dynamic properties are not invented or rejected by this table. Existing participation rules still control how facts flow through bindings: a constant range expression already contributes a range fact, while an unannotated function parameter does not acquire a domain merely because the function is called with a range. This repair does not replace that policy with whole-program inference. Generic endpoint types, metadata and user-class declaration identities are retained as given.

## Elision and effects

| Direction | Ergonomics / autocomplete / wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Treat the static property type as sufficient proof to elide receiving checks | Faster reads, but replacing a getter can make a typed binding hold the wrong value silently. | Rust fields do not have JavaScript's replaceable property descriptors/prototype lookup. | Unsound unguarded optimization. | Confuses a declared view with proof of the accessed implementation. | **Reject.** |
| Seal all range instances/prototypes or bypass getters with internal reads | Makes some proofs easier, but changes existing observable descriptors, hooks and exceptions. | Rust's fixed layouts are not a mandate to change JavaScript reflection. | Potentially simpler ICs at a public compatibility cost. | Changes behavior unrelated to typing and hides required effects. | **Reject.** |
| Keep receiving checks unless an independent origin/shape guard proves the getter | Correct early diagnostics and runtime boundaries; public getters still run once. | Rust's stability is supplied by its storage model; JavaScript engines instead use guarded property access. | Interpreter retains checks; a production engine can guard shape/prototype/getter identity and deoptimize. No new layouts required. | Maintains admission and liveness when public properties are replaced. | **Recommend.** |

The getter result is not read during checking and cannot be replaced by an internal endpoint read. A read can throw even when the static result type is known. A stale borrowed location still fails its ordinary liveness checks. Receiving boundaries must preserve their existing admissions, effects and failure order when the implementation of the property has not independently been proved stable. Static facts are useful for early rejection without granting that proof.

## Extracted binding boundaries

The existing inferred-initialization rule also applies to a pattern's extracted binding when the existing participation rules give that binding a contract. Without its initialization boundary, `const {start:x}=r` could store a replacement getter's String while advertising `uint8`, and a later typed alias or captured return could elide its check.

| Direction | Ergonomics / autocomplete / wrong guess | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Drop the extracted type | Destructuring loses autocomplete and early rejection that the equivalent plain binding provides. | Rust destructuring preserves field types. | Less checking information, no layout benefit. | Discards an existing required inferred contract. | **Reject.** |
| Keep an unchecked type and carry instability through every downstream use | Preserves autocomplete but delays failure and requires every alias, capture, import and later Script to retain provenance. | Rust's typed storage needs no comparable unchecked-value provenance. | Additional dependency/cache state and guards at consumers; no shape benefit. | Can prevent a particular elision, but leaves the inferred binding's invariant broken. | **Reject.** |
| Enforce the existing inferred contract at extracted initialization | Equivalent plain and destructured bindings have the same contract; wrong values fail at the binding that receives them. | Rust preserves types through binding patterns; JavaScript needs dynamic admission where a getter or iterator is replaceable. | Uses the ordinary boundary and allows independently proved checks to optimize away; no new object layout or IC form. | Establishes the binding before aliases/captures trust it and leaves reference rules separate. | **Recommend.** |

Read once, select the default if needed, then admit the extracted value before initializing that binding or reading the next member. A failure preserves prior effects and initialized siblings, leaves the failed binding uninitialized, and follows ordinary iterator closing. Generic parameters resolve in the binding's runtime context. Already conforming plain aggregates stay plain. This closes the shared extracted-binding omission instead of adding range-specific provenance to later consumers; ordinary gradual bindings still acquire no inferred contract.

## Scope and next work

This increment establishes the data-property surface. It does not claim complete method or arithmetic-result typing, new nominal-to-structural admission, readonly mutation diagnostics, iterator contexts, or general custom enclosure declarations. Those consumers must build on the same endpoint/presence contract and preserve the existing runtime checks. In particular, a computed range can change shape; do not copy an input's concrete shape into the Static Type of every operation result.

Primary sources: [Rust Range](https://doc.rust-lang.org/std/ops/struct.Range.html), [Rust RangeBounds](https://doc.rust-lang.org/std/ops/trait.RangeBounds.html), and [Rust Bound](https://doc.rust-lang.org/std/ops/enum.Bound.html). The tables state where the analogies stop.
