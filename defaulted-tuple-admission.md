# Defaulted tuple admission and storage

A tuple without a rest has an index-determined type at each position even when its trailing positions have defaults. Defaults change the admitted length range; they do not make supplied positions gradual or defer callable specialization.

```js
function id<T: type>(x: T): T { return x; }
const raw: any = [id, 1];
const t: [(x: string) => string, number = 7] = raw;
t === raw;                 // true
raw[0] === id.<string>;     // true
raw.pop();                 // permitted: one required position remains
raw.push(8);               // number position; no new callable specialization
```

The same retained evidence applies through an object, array, tuple or selected union path. Pure membership and reference viability remain observational. Acquiring owned storage copies value-class elements and materializes selected callables using retained observations and ordinary changed-value writes. Existing backing stays authoritative. Own-data preflight, sparse initialization, final publication and failure effects follow [typed Array accessor policy](typed-array-accessors.md) and [sparse admission](sparse-storage-admission.md).

## Admission decision

Required properties: retain the selected callable and nested evidence; preserve compatible container identity; read each position only as required by membership; publish storage only after required work; preserve independent effects and existing backing.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Keep generic values whenever any position has a default | Adding an unrelated numeric default changes the function returned through a concrete signature | Rust propagates tuple coercion context per field; it has no defaulted tuple positions | Cheap admission, but loses concrete callable optimization | Drops selected callable obligations | Rejected |
| Specialize during membership or replay reads afterwards | A predicate mutates values, or a getter's second result replaces the value that passed | Rust coercion sites are distinct from arbitrary predicates; JS getter effects have no direct analogue | Speculative specializations or duplicate reads and IC traffic | Leaks effects from rejected alternatives or changes evidence | Rejected |
| Always allocate a new tuple | Aliases silently stop sharing even when every value already fits | Rust tuple construction creates a value; that does not prescribe JS structural alias admission | Extra allocation, copy and GC work | Changes compatible identity and ordinary failure behavior | Rejected |
| Retain evidence and complete admission at supplied positions | The advertised concrete signature behaves consistently with required tuples; nonwritable storage fails at the boundary | Rust's per-field coercion context is relevant; dynamic backing publication is specific to this proposal | Reuses specialization caches and an internable positional descriptor; no wrapper shapes | Preserves evidence, backing invariance and ordinary liveness | **Selected**, independent of implementation effort |

## Length-domain decision

For a no-rest tuple with `r` required positions and `m` total positions, acquired storage permits every integer length from `r` through `m`. At any permitted length, index `i` has its declared position type. A construction that fills all defaults still acquires this range. Omission changes occupancy, not an existing location's type.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Freeze the initial or fully filled length | `pop` and `length` reject lengths the tuple type admits; identical types behave differently depending on construction | Rust tuples have fixed arity, so the analogy cannot justify this proposal's optional suffix becoming fixed | Cheap exact extent guard | Narrows the declared length domain | Rejected |
| Keep only currently occupied positions | Future permitted positions lose their contracts; autocomplete describes more than storage enforces | Neither Rust fixed tuple fields nor TypeScript optional positions imply forgetting declared types | Per-instance shapes and unstable growth transitions | Wrong later stores and references | Rejected |
| Treat optional positions as independent unions with undefined | A missing suffix becomes a present undefined value; numeric reads surprise users | TypeScript optional elements include undefined under strict null checking; this proposal instead has explicit supplied-value membership and construction defaults | Optional representations and extra guards affect dense storage | Confuses omitted positions, holes and supplied undefined; changes declared position types | Rejected |
| Recompute a partition from current values | Store permission depends on neighboring values | Rust keeps field types stable; its borrow model is not dynamic repartition | Whole-sequence scans and changing IC assumptions | Retypes live locations without removal or relocation | Rejected |
| Retain the full positional list and the admitted length interval | Permitted shrinking and growth are predictable; wrong guesses fail at the existing operation boundary | TypeScript optional tuples expose an arity window; Rust supplies the stable-field analogy but not optional arity | One minimum plus maximum width and indexed type table; no element reads for planning | Preserves complete domain and SameType for every surviving location | **Selected** |

Conjoined acquisitions intersect length intervals and every participating positional contract. A finite maximum alone is not an exact length: do not flatten a non-singleton interval to its maximum. A separately required fixed extent may select one length. Required-rest suffixes in a conjunction remain end-aligned; survivor checks still apply. Search judgments include absent but permitted position domains, restricted by the complete conjunction.

Shrinking removes locations, and later growth does not revive their references. A surviving prefix reference remains live unless an independent relocation invalidates it. Deleting an optional indexed property still creates a prohibited hole; optional suffix removal is a length/mutation operation. Moving between distinct position types still fails before element work.

## Construction and later growth decision

A shorter array converted directly to the tuple supplies omitted trailing defaults in the existing separate output. A fully supplied compatible input uses retained admission. A supplied hole is observed by Get and must satisfy the position type; it does not select the initializer. Defaults are evaluated and converted when the type is formed, with existing specialization and budget rules.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Fill on membership, and treat holes/undefined as omission | A query mutates length; explicitly supplied undefined unexpectedly requests a default | JS default parameters treat undefined specially, unlike this tuple's length-based omission; Rust has no corresponding tuple feature | Extra writes and transitions during queries | Violates observation-only membership and supplied-hole rules | Rejected |
| Reapply tuple initializer whenever an index is regrown | Convenient-looking reset, but length growth means a different initializer depending on an earlier view | Rust resize takes an explicit producer; it does not replay a field initializer | Must retain default provenance and reconcile differing defaults in overlapping views | Conflicts with established DefaultValueOf of the actual backing type; introduces ambiguous overlapping defaults | Rejected |
| Preserve omission during construction; use actual type defaults for unsupplied growth | `[number, number = 7]` constructs an omitted tail as 7, while regrowth supplies number's 0; the distinction is explicit | Rust construction vs resize initialization is a useful distinction, not a syntax or effect equivalence | Existing dense initialization/staging; no per-position initializer metadata in store ICs | Preserves construction identity, initialized storage, prior effect ordering and liveness | **Selected** |

For example, after admitting `[1, 2]` as `[number, number = 7]`, shrinking to length 1 and regrowing to 2 yields `[1, 0]`. A supplied indexed store uses its supplied value, and skipped new positions use their actual type defaults. Existing nonextensibility, descriptor, required-property and publication checks remain in force.

## Bounded tuple copies

A copy's positional shape follows the positions the operation actually produces. For a bounded no-rest source, `slice` takes the captured window and `toReversed` reverses the occupied positions. These results have exactly those required positions. `with` retains the complete source storage domain after checking its replacement; `toSorted` retains that domain when all occupied position contracts are the same. Omitted source defaults do not create extra copied positions or new construction initializer metadata. The source retains its admitted length interval. A heterogeneous comparator-driven sort retains the existing untyped-result rule.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Drop all positional backing because the source length varies | Copies unexpectedly accept values incompatible with their produced positions | Rust slice copies preserve an element contract; this is only a homogeneous analogy | Cheap creation but loses specialized stores | Regresses the existing guaranteed-position contract | Rejected |
| Transfer the original optional/default metadata unchanged | Reversal or slicing moves/removes fields but leaves defaults attached to the old indices | Rust tuples have no optional suffix metadata to transfer | Cheap descriptor reuse, wrong geometry | A reversed optional suffix can become an unsupported optional prefix; sliced defaults can name absent fields | Rejected |
| Invent transformed optional languages separately for each method | Richer future APIs, but surprising optional-prefix types and inconsistent discovery | TypeScript mapped tuples offer type-level transformations; erased types do not define runtime backing or liveness | More descriptor forms and transformation rules | Could be designed, but needs new contracts beyond currently guaranteed output positions | Rejected for these existing methods; not required to preserve their positional guarantees |
| Retain only concrete output positions for every method | Fields stay checked, but a harmless with copy unexpectedly loses permitted shortening/growth | Rust fixed arity does not justify narrowing this proposal's existing source domain | O(copied positions) descriptor derivation with stable ICs | Sound, but needlessly discards unchanged source contracts | Rejected for with and supported sorting; appropriate for transformed windows/reversal |
| Preserve the source domain when position contracts remain unchanged; derive concrete positions for window/reversal | with and homogeneous sorting preserve the existing usable length range; changed geometry gets precisely its guaranteed positions | Rust stable-field typing is relevant, but supplies no rule for optional JS tuple methods | Reuse immutable source descriptors for with/sort; derive concrete descriptors alongside transformed copying; stable ICs | Retains complete unchanged domains, produces valid transformed shapes, and never alters source references | **Selected**, independent of effort |

## Scope and further work

This completes no-rest storage domains and supplied-position retained admission. It does not add static indexed-access inference for defaulted tuple positions. Default-plus-rest, finite/nested and multiple-rest representation still require their own assignment rules. Recursive omitted-default conversion through a membership-successful outer structural boundary remains a separate audit; the successful direct short-input control is not evidence that all nested omissions are filled. General callable intersections remain open.

Primary prior art checked for this decision: [Rust tuple types](https://doc.rust-lang.org/reference/types/tuple.html), [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html), and [TypeScript 3.0 optional tuple elements](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-0.html#optional-elements-in-tuple-types). Rust tuples have fixed arity; TypeScript's erased optional tuple types do not implement this proposal's runtime storage or reference-liveness rules.
