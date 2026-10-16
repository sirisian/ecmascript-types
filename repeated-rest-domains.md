# Repeated rests with a stable positional domain

Several written rests need not make the type of an occupied position ambiguous. Equal unbounded element types, separated only by required positions of that same type, describe one homogeneous region with a retained minimum count.

```js
type T = [string, ...[].<number>, ...[2].<number>, ...[].<number>, boolean];
const raw: any = ["start", 1, 2, 3, true];
const t: T = raw;
raw[4] = false; // Boolean suffix, independent of the two rest counts
raw[1] = 4;     // Number region
```

Its positional domain is the same as `[string, ...[2].<number>, ...[].<number>, boolean]`. This is an internal domain proof, not a rewrite of the written Type Object or a promise that these two Type Objects have the same identity. The original sequence matcher still determines argument/run assignments where those counts matter.

## Required properties and decision

A valid solution retains the full prefix, suffix, finite minimum, defaults, declared type identity, ordinary conversion/effect order, backing authority, and reference liveness. It derives position types from declarations and admitted lengths, not current element values. It must keep finite counts compact and refuse to infer this domain across a different intervening type.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Keep the first-rest approximation | A displayed Boolean suffix can be treated as Number; adding an equivalent rest changes writes or drops defaults | Rust array/slice element contracts do not disappear when a source is decomposed; no exact multiple-rest analogy | Cheap, but cannot be cached as a correct contract | Loses suffixes, finite floors and nested storage evidence | Rejected |
| Freeze the first successful value partition | Completion suggests stable slots whose contracts change when initial values change | Rust mutable-reference invariance concerns the place type, not current contents | Per-instance partition metadata and avoidable shape/IC variation | Makes current values determine later writable contracts and reference authority | Rejected |
| Use an independent union of the written element types at every index | Suggests that Boolean can be stored anywhere and hides required counts | Rust homogeneous slices offer no precedent for erasing ordered constraints | Simple per-store check, but an invalid domain | Loses prefix/suffix correlation and cannot preserve invariant locations | Rejected |
| Try conversions while searching for a partition | A wrong guess can run user conversion code or repeat observations | Rust coercion selection does not authorize speculative JavaScript effects | Discarded calls, allocations and IC effects | Violates ordinary effects even where the position type was already provable | Rejected |
| Always perform a correlated whole-sequence check, even for this proved family | Can express the right contract, but a local write needlessly depends on unrelated positions | Rust fixed homogeneous storage uses its declared element type and bounds | Unnecessary sequence work per store and additional invalidation dependencies | Could be correct only with a complete effects/liveness account | Not selected for the proved family; still relevant to genuinely heterogeneous rest domains |
| Prove and retain one complete positional domain from equal declared types and counts | Equivalent decompositions have the same checking and completion results; suffixes and minimum lengths stay visible | Rust exact array counts and homogeneous slice contracts are useful principles; its mutable-reference invariance remains a limited analogy | Shared compact run descriptors; ordinary position checks and existing IC guards; no partition fields on user objects | Preserves all position contracts at every admitted length, existing admission evidence and independent liveness | **Selected**, independently of implementation effort |

TypeScript 4.2's tuple-rest documentation allows one unbounded rest and excludes another rest after it. Its erased runtime model therefore does not settle this proposal's multiple-rest storage or conversion rules.

## Domain proof

Expand nested tuple operands in order, retaining finite array extents as counts. When another unbounded run is reached, it can join the preceding unbounded region only if its element type is the same type and every required position between them has that same type. Preserve the intervening count as required positions in the homogeneous region; do not erase it. Repeating this step establishes at most one unbounded region. Zero finite extents contribute nothing. Unresolved counts and active recursive expansion supply no new proof.

An implementation can move the intervening homogeneous required runs to the end of the fixed prefix in its internal domain descriptor. Every affected position still has exactly the same type at every admitted length, and the minimum length is unchanged. A defaulted prefix keeps its original initializer provenance and omission rule. A required element after a default, or a default after an unbounded rest, remains invalid under the existing placement rules.

For this family, the existing complete-domain rules apply to membership, contextual literals, conversion, default construction, nested/union acquisition, overlapping contracts, stores, initialized growth, intrinsic movement, search, subtyping and references. A surviving position cannot change contract during resizing. Removing an element still invalidates its reference. Pure membership neither acquires backing nor constructs omitted defaults. Existing backing remains authoritative through widened views.

The proof does not cross `string` in `[...number, string, ...number]`, combine different rest types merely because one is a subtype of another, or merge distinct declaration/metadata identities. Finite counts never expand into per-position Type Records. A finite count may require actual elements when a value is constructed; compact type metadata does not remove that allocation requirement.

## Remaining general case

`[...[].<number>, ...[].<string>]` has a genuinely value-dependent partition. Its direct first-rest backing and nested missing-backing failures remain open. General conversion selection must retain observations and avoid executing discarded object conversions, and successful output must acquire complete correlated backing. This increment neither changes that recommendation nor claims those paths are repaired. It establishes a complete declaration-based family that the general implementation can continue to specialize.

Primary comparisons: [Rust arrays](https://doc.rust-lang.org/reference/types/array.html), [Rust slices](https://doc.rust-lang.org/reference/types/slice.html), [Rust variance](https://doc.rust-lang.org/reference/subtyping.html), and [TypeScript 4.2 tuple rests](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-2.html).
