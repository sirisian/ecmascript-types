# Heterogeneous rests: conversion selection and correlated storage

Retain the complete ordered sequence contract, choose a conversion assignment before executing its conversions, and give borrowed positions declaration-derived invariant contracts. A successful partition is evidence about the current sequence, not its permanent writable layout.

This is a design decision and implementation handoff. The general implementation and its detailed `spec.emu` algorithms are not complete. In particular, changing the first-rest fallback into a union does not implement this decision. The existing established positional domains, including repeated equal rests, remain valid specializations.

## What the implementation demonstrates

These observations were reproduced on engine262 commit `bc246070b04f13661e877d23b399870a2b2c5cf3`. Use dynamic sources to distinguish runtime behavior from contextual literal checking.

```js
type T = [...[].<number>, ...[].<string>];
const raw: any = [1, "s"];
const t: T = raw;
raw[1] = "t"; // currently throws: the direct boundary retained only number
```

A nested `{ items: T }` boundary instead lets this replacement succeed, but also permits subsequent changes to `["a", 2]`, which is not a member of `T`. A direct boundary permits `push(2)` after `[1, "s"]` and leaves an actual Number after the String. A union boundary `T | null` has the missing-guard failure too. These are observations of invalid *results*. They do not prove that the input Number must be rejected: ordinary String conversion may produce a valid replacement. Inspect runtime types or final membership, not only joined text.

There is also a conversion failure independent of later writes:

```js
type T = [...[].<uint8>, ...[1].<string>, ...[].<boolean>];
const raw: any = [1, 2, 3];
const t = (raw := T);
t is T; // currently false, even though the conversion completed normally
```

The corresponding constructor example uses `[C..., string{1}, boolean...]`, where `C` accepts `any`, and input `[1, "s"]`. It currently calls the constructor twice and returns a nonmember. The required String run is not optional. Conversely, for `[C..., string...]` without a required String run, two `C` constructions can be a legitimate greedy assignment. Constructor counts alone do not establish a bug.

After nested admission, borrowing the second element, popping it, pushing a new String, and writing through the old borrow currently succeeds. That is a separate removal-identity failure caused by missing backing, not a conversion-choice question. Pure `is` intentionally acquires no backing and is a control, not another missing-guard failure.

The companion audit contains 28 executable Script probes, their raw observations and classifications. They are discovery evidence, not passing conformance expectations for these defects.

## Requirements for a valid solution

1. Retain the ordered contract and every finite count, including zero. `[number..., string...]` is not an independently writable array of `number | string`.
2. Permit a valid new partition without making current values define the location types. Replacing the first item of `[1, "s"]` with `"a"` may change the partition to two Strings.
3. Preserve existing acquired backing and identity through direct, nested, union and intersection boundaries. A later view cannot overwrite a narrower existing location contract.
4. Never publish a converted result outside its target domain, or publish a result without the backing required to keep its acquired contracts valid.
5. Reuse the operation's retained source observations. Conversion search must not run constructors, casts or coercion methods whose results will be discarded while trying another partition.
6. Retain selected nested/callable/default evidence and ordinary earlier effects. Do not publish failed speculative admission, undo completed nested boundaries, or treat a retained Object identity as a deep snapshot.
7. Keep reference invariance, allocation generation and removal identity independent. A plan made before user code is not authority to use a stale reference or overwrite changed backing.
8. Check all mutation routes against the same contracts: indexed assignment, reflection, deletion, length changes, intrinsic moves, aliases and references. General growth needs a valid default sequence, not a first-rest default repeated into every gap.
9. Keep counts compact and make optimization costs explicit. Full uint64 realization, opaque extents and active recursive expansion remain separate limitations; they must not become guessed positional proofs.

## Prior-art limits used in the decisions

**R1.** [Rust mutable-reference variance](https://doc.rust-lang.org/reference/subtyping.html) supplies the invariant-target comparison. It does not model this proposal's dynamic aliases, checked structural acquisition or multiple-rest partitions.

**R2.** [Rust slices](https://doc.rust-lang.org/reference/types/slice.html) have one element type. They are useful for the homogeneous specialization, not a precedent for erasing the ordering of heterogeneous runs.

**R3.** [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html) give a declared destination to propagated tuple elements. This supports separating destination selection from execution, but Rust does not supply this proposal's greedy matcher or effectful JavaScript conversion protocol.

**T1.** [TypeScript 4.2 tuple rests](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-2.html) allow a leading rest followed by fixed positions, but prohibit another rest or optional element after it. Its erased types provide neither the general pattern nor runtime backing required here.

**J1.** [ECMAScript array mutation algorithms](https://tc39.es/ecma262/multipage/indexed-collections.html#sec-array.prototype.splice) specify ordered property operations with abrupt completion. They do not promise atomic replacement of an entire array. The proposal's existing typed insertion/publication rules must be preserved when extending them.

The recommendations below are design inferences from this proposal's commitments and those limited comparisons, not behavior attributed to Rust, TypeScript or ECMAScript.

## Decision 1: what survives admission?

| Direction | Ergonomics, completion and wrong-guess cost | Prior art | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Retain only the first rest | Displayed String positions reject Strings; an apparently successful append can invalidate the tuple | R2 justifies homogeneous storage only; T1 does not justify ignoring later runs | Cheap descriptor and stores, but optimized code receives false type facts | Loses ordering, finite runs and reference authority | Rejected; the current defect |
| Freeze the first successful partition as a layout | Initial values silently determine later legal writes; equivalent declarations get different completion behavior | R1 concerns declared places, not a partition guessed from contents | Per-instance partition descriptors and avoidable shape/store-IC variation | Prevents legitimate repartition and makes borrowing depend on initialization history | Rejected |
| Use independent per-index unions alone | Completion shows the alternatives, but writes can create impossible combinations | Neither R2 nor T1 supplies correlated runtime enforcement | Ordinary union checks are cheap; no dependency maintenance | Accepts `["a", 2]` for `[number..., string...]` | Rejected |
| Make all heterogeneous rest tuples immutable, or copy/wrap every crossing | Existing array identity and mutation expectations change; dynamic aliases see a different object or surprising failures | Rust ownership cannot be imported without its separate alias discipline; T1 is erased | Allocations, wrapper dispatch and more polymorphic ICs; freezing still has alias consequences | Can simplify enforcement only by changing already committed identity/mutability semantics | Rejected as the general answer, independently of implementation effort |
| Retain the complete ordered domain, with invariant position contracts and correlated guards | Completion can show the declared alternatives while a conditional write still reports an ordering constraint; valid repartition is possible | R1 constrains borrowing, R2 remains a fast specialization; the correlated guard is this proposal's addition | Shared declaration descriptors and stable IC guards; general checks can cost sequence work, with invalidation-aware summaries as an optimization | Preserves ordering and stable locations together, provided every mutation and nested dependency is covered | **Recommended** |

The retained domain is an ordered pattern of fixed/defaulted positions, finite runs and dynamic runs. Keep the original written-slot/count relationship where `SequenceAssignment` consumers need it. A shared automaton or compact run representation is possible; no eager expansion of a finite count or eager enumeration of all partitions is required. This does not alter Type Object identity.

For a given occupied index and length, a *position projection* describes all types that can occupy that index under declaration-compatible assignments, without inspecting current values. An established positional domain has its existing exact type. The unbounded `[number..., string...]` example has `number | string` at every occupied index. The complete ordered guard remains an additional obligation. A projection is neither evidence that a current value matches a particular branch nor permission to write every projected member in the current surrounding sequence.

Existing narrower storage is retained. New sequence acquisition adds its correlated obligation; it does not replace an existing element/property contract with the projection. Resize still requires surviving positions to keep their invariant contracts, in addition to satisfying the new length and ordered sequence. An unresolved projection is not permission to substitute the current value's type.

## Decision 2: how is a conversion partition chosen?

| Direction | Ergonomics, completion and wrong-guess cost | Prior art | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Assign by counts alone, then convert to the first rest | A count-compatible input can return a value outside its annotation | R3 requires the receiving position's destination, not an arbitrary rest | Cheap search; incorrect destination specialization | Drops the required collected contracts before publication | Rejected |
| Execute conversions while backtracking until something works | A failed guess runs arbitrary code, allocates objects and may mutate unrelated state | R3 supplies no such speculative runtime protocol; J1 does not roll effects back | Repeated calls, allocations, deoptimizations and unbounded wasted work | Cannot undo effects or independently completed nested boundaries | Rejected |
| Eagerly convert to every possible target and cache the results | Several unwanted constructors run even if the first candidate works | Neither R3 nor T1 supplies this behavior | Work scales with all candidates and retains unused results | Caching does not make discarded effects correct | Rejected |
| Globally minimize the number of conversions before selecting | Sometimes preserves more input identities, but a distant change can alter an earlier destination | R3 does not define this ranking; it replaces this proposal's greedy rule | Adds an optimization objective and associated cache state | Can be deterministic but changes existing assignment priority | Rejected as the default; no new ranking rule is justified |
| Retain ordinary complete membership success; otherwise select a greedy assignment from admission/conversion plans, then execute it once | A declared destination explains each conversion; a selected conversion can still throw, and there is no hidden retry | R3 motivates destination-first selection; the greedy order comes from this proposal | Shared applicability facts and memoized assignment; only selected conversions allocate | Keeps counts, destination contracts, observations and effects distinct; references never become converting slots | **Recommended** |

There are three distinct facts: an item is already admitted; an applicable conversion operation can be selected; that operation actually completes with an admitted result. A constructor's declared applicability does not predict whether its body throws or returns an invalid result. Do not call a conversion to discover the third fact during search.

Use the existing whole-sequence membership shortcut when it succeeds, retaining its selected admission and construction evidence. Otherwise determine applicability under the relevant implicit or explicit conversion rules, including required fixed/finite runs, and choose the greatest valid count list in the existing leftmost-greedy order. Retain the selected per-item destinations and any required obligations. Execute in source index order and propagate the first abrupt completion. A selected conversion failure does not trigger a different partition, even if another conversion might have succeeded. This cost of a wrong guess is deliberate and visible; avoiding speculative user effects takes precedence over finding an arbitrary successful execution.

This does not authorize extra membership hooks or merge separate candidate observations. Plan discovery must respect the existing observation/effect discipline and distinguish pure applicability facts from required effectful judgments. General metadata and dynamic constructor applicability need explicit coverage before the plan algorithm is specified as complete.

For `[uint8..., string{1}, boolean...]` and `[1, true]`, the finite String run must receive one item or the assignment fails; the Boolean rest cannot erase it. For `[uint8..., boolean...]` and `[1, true]`, a valid plan assigns the Number to the numeric rest and the Boolean to the Boolean rest. For `[C..., string...]`, where `C` has an applicable `any` constructor, an empty String rest remains possible and greedy conversion may select `C` for all items. Adding a required String run changes that judgment.

The produced sequence must satisfy the selected complete pattern and acquire complete backing before publication. Checking only total length, attaching the first rest, or doing a final membership check without backing is insufficient. That final obligation must consume retained result/admission evidence rather than introduce a second observable source walk.

## Decision 3: what may a reference claim?

| Direction | Ergonomics, completion and wrong-guess cost | Prior art | Real-engine performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Give a borrow the current member's type | A String borrow unexpectedly promises permanent String storage merely because the item is a String today | Contradicts the place/content distinction motivating R1 | Cheap initially, but later repartition invalidates optimized assumptions | Grants narrow authority over a broader location | Rejected |
| Pin a partition while borrowed, or silently retarget/invalidate a borrow when the partition changes | An unrelated alias's write gains a new failure mode that depends on outstanding borrows | Rust exclusivity is not this proposal's alias/liveness model | Borrow registries and partition-dependent IC invalidation | Introduces a new lifetime regime and can hide contract changes as liveness changes | Rejected |
| Prohibit every reference to a heterogeneous rest tuple | Simple diagnostics, but removes valid broad-location references | R1 requires invariant compatibility, not a blanket prohibition | Cheap, with lost expressiveness | Safe but stronger than the required contract and liveness rules | Rejected as the general rule |
| Borrow the declaration-derived invariant location; retain the whole-sequence guard on every write | Completion shows the broad location type; a narrow ref parameter is refused before its body, and a broad reference can still fail an ordering check | R1 is the relevant limited analogy; guard failures remain this proposal's runtime checks | Stable reference/store IC contract plus the correlated guard; ordinary generation/identity checks stay separate | No current-value narrowing or partition-dependent lifetime; removal and relocation still invalidate independently | **Recommended** |

A fresh `[number..., string...]` location is therefore not a `ref number` or `ref string` merely because of its current value. If an already acquired narrower backing contract exists, it still supplies that location's invariant type. Ordinary value reads can narrow under the proposal's existing flow rules; that does not narrow borrowed storage.

A broad reference does not promise every write of its target union succeeds: bounds, readonly state and acquired backing already provide independent runtime constraints. The correlated guard is another required constraint. Replacing a value without removing its slot does not itself create a new element identity. Pop/delete, recreation and relocation retain the existing independent liveness rules.

## Decision 4: effects and mutation publication

| Direction | Ergonomics, completion and wrong-guess cost | Prior art | Real-engine performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Check the whole sequence only after a write or intrinsic completes | The reported failure leaves an invalid object visible through aliases | J1 allows earlier effects but does not authorize breaking an acquired type invariant | Easy post-check; invalid optimized assumptions can escape before it | Too late for observation, reference safety and abrupt intermediate states | Rejected |
| Make every mutation a transaction and roll everything back on failure | Looks atomic until a getter, comparator or conversion changes another object | J1 has ordered abrupt effects; Rust is not a JavaScript rollback model | Copies/journals cannot reverse arbitrary calls and external effects | Changes existing failure semantics and cannot undo completed nested boundaries | Rejected |
| Guard publication, retain existing operation order and revalidate after effects | A failing operation preserves earlier ordinary effects while never publishing a newly invalid typed state | Extends the proposal's existing staged-publication rules while respecting J1; R1 still supplies no lease across calls | Preflight can use owned backing evidence; caches need relevant descriptor/value/length generations | Requires all published intermediate states, nested obligations and final liveness to remain valid | **Recommended** |

For a single store, the guard must choose among destination contracts compatible with the *rest of the sequence* as well as existing storage. It cannot first accept a Number against the broad `number | string` projection and then reject an ordering violation if an ordinary permitted String conversion supplies the valid selected destination. Conversely, it must never publish the unconverted Number after a String run merely because that Number belongs to the projection.

Intrinsic geometry and ordinary write checks are separate. Preserve the existing declaration-based movement checks and insertion staging. A general extension must account for every state it can expose, including abrupt property operations, not merely the final permutation. A preflight before a comparator or conversion cannot bypass subsequent guard and liveness checks. If a proposed extension relies on atomic visibility, it must prove that from the existing publication mechanism; it cannot assume it from the method name.

Nested objects require backed evidence for the roles the guard relies upon. A transient structural membership success or an Object identity alone is not stable evidence across child mutation. Preserve selected descendant obligations and completed child boundaries; establish how retained role evidence survives every allowed child mutation, or maintain and revalidate its dependencies. Do not cache metadata validation indefinitely merely because ordinary data properties have not changed. These are implementation proof obligations, not permission for an incomplete scalar-only guard to claim general coverage.

## Implementation handoff and completion gates

The selected direction is the complete ordered guard plus invariant projections and destination-first conversion. It is selected independently of effort. The following gates delimit implementation work; none is claimed complete by this design note.

| Gate | Concrete completion evidence |
|---|---|
| Compact declaration model | Fixed/defaulted positions, finite/dynamic runs, ordered expansion and count consumers agree; established-domain optimizations remain equivalent; projections do not depend on observed values |
| Conversion selection | The two nonmember-producing witnesses return correctly admitted results; implicit/explicit cases, required finite runs and ambiguous constructors follow the selected rule without discarded conversions or replayed source Gets |
| Owned storage and acquisition | Direct, nested, union and intersection admission install the same complete guard before publication; all ordinary/reflective stores, growth, defaults, deletion and failed acquisition preserve it |
| References and effects | Broad versus narrow borrowing is checked against backing; removed/recreated and relocated elements stay stale; reentrant and nested mutation cannot use stale admission or validation evidence |
| Intrinsics and transport | Movement, insertion, species/copy results, proxies, metadata and child/default transport preserve the whole contract and existing failure precedence |
| Specification and bounded regressions | `spec.emu` determines the selected outcomes and effect order without interpreter-specific fields; focused probes become desired-outcome regression tests as their gates are implemented |

Begin by implementing the compact ordered-domain and selection machinery together with the storage enforcement needed by its first published results. Do not publish a newly successful general conversion behind the old first-rest backing. A small initial specialization is useful only when its proof covers its entire admitted family; mark unsupported general effects and transport explicitly. Existing stable tuple domains must keep their present fast path.
