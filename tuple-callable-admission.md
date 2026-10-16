# Tuple callable admission and storage domains

A required fixed tuple position and a single trailing unbounded homogeneous array rest have types determined by their indices. Their receiving boundaries must retain generic-callable specialization:

```js
function id<T: type>(value: T): T { return value; }
const fixed: [(value: string) => string] = [id];
const tail: [boolean, ...[].<(value: string) => string>] = [true, id];
fixed[0] === id.<string>; // true
tail[1] === id.<string>; // true
```

Retain the selected membership observations and union choices; for a rest, keep only evidence selected by SequenceAssignment. Complete specialization after membership with ordinary Set only for changed values. Preserve container identity, custom prototypes, existing backing storage, setter/proxy behavior and ordinary failure effects. Pure membership and reference/overload viability still neither specialize nor write. Removing a tuple element invalidates its existing references, and later filling the same index does not revive them; retaining a different position does not invalidate that position.

The acquired tuple contract covers its complete index domain, including an empty trailing rest and later growth. It is not a per-occupied-index property map. Value-class elements copy when owned storage is first acquired, using retained observations and ordinary Set. A later view does not copy the elements again. Publish newly acquired storage only after all required writes; independent nested boundaries and earlier ordinary effects survive a later failure.

When several paths acquire stable tuple or homogeneous-array contracts on one previously untyped Array, all obligations apply. Intersect types at each position and over the tail, and intersect their length domains. A fixed length bounds a rest; required prefixes contribute a minimum. Conflicting fixed lengths or a fixed length below a required prefix have no common domain and fail before element materialization. Existing backing contracts, including one established by a setter during this admission, are preserved. No view replaces a generic-function slot's promise with a concrete signature.

## Decision: stable tuple admission

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Keep compatible generic values | A concrete positional signature can lose its selected output through erasure | Rust propagates a tuple's expected type to each component | Cheap boundary, unresolved later dispatch | Fails the receiving callable contract | Rejected |
| Specialize in membership or all candidate slots | A membership query or an unselected candidate can fail or mutate storage | Rust coercion sites are distinct from independent predicates | Speculative body creation and writes | Leaks unsuccessful sequence/union alternatives | Rejected |
| Replay reads or copy/wrap the compatible container | Getter results can change; aliases and identity surprise callers | Rust owned tuple construction is not a JavaScript alias boundary | Extra traversal/allocation or wrapper shapes/ICs | Changes admitted values, origins and observable effects | Rejected |
| Retain selected values, ordinary writes, complete stable domains | Positional signatures survive erasure; write refusal belongs to the boundary | Rust tuple fields have fixed indexed types and propagate coercion context; its borrow model does not specify JS effects | Indexed store checks, internable shape/element contracts and existing specialization cache | Preserves selected evidence, backing promises and ordinary liveness checks | **Selected** for required fixed positions and one trailing unbounded array rest |

## Decision: overlapping acquisition

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| First or last path wins | Renaming/reordering fields changes allowed stores | Rust aliases do not redeclare an existing field's type | Cheap but unstable shape selection | Drops required obligations | Rejected |
| Stamp only occupied indices | Empty tails and future pushes lose the advertised contract | Rust array elements share one domain regardless of occupancy | Per-index metadata and property IC pressure | Wrong growth/deletion/reference domains | Rejected |
| Check every path again on every store | Correct-looking stores can unexpectedly replay unrelated getters | Rust checks known field types without runtime descriptor walks | Repeated checks and poor store specialization | Introduces observations not required by the existing storage judgment | Rejected |
| Intersect complete domains once; preserve established backing | Every admitted promise remains in force; impossible joint arities fail at acquisition | Rust uses stable field types; dynamic overlapping structural acquisition has no close counterpart | One composed positional/tail descriptor, no new wrapper | Stable location contracts and no retyping of an existing place | **Selected**, regardless of implementation effort |

## Why the general rest problem is separate

The prior [array admission note](array-callable-admission.md) recommended retaining a full tuple contract and validating mutations against it. The audit qualifies that recommendation: a successful post-store SequenceAssignment alone is not enough. Repartitioning can change the declared type of an existing index without removing or relocating that location. A live invariant reference would still name it, and the current liveness rules do not invalidate such a reference merely because an inference selected a different slot.

The current interpreter also contains a concrete older defect: its first-rest stamp puts all non-rest positions at the front. A member of `[...[].<string>, boolean]` can therefore reject a String store at its first String position. This increment does not extend that stamp to new shapes or claim to repair it. Direct callable admission here covers only required fixed positions and a single trailing unbounded array rest. Defaulted positions, finite/nested rest operands, non-final rests and multiple rests remain follow-up work.

A valid general design must satisfy all of these requirements: preserve the declared sequence language; determine conversion targets before effectful conversion; retain selected observations without replay; preserve ordinary setter/proxy failure effects; maintain established location contracts or explicitly end their lifetimes; keep covariance and invariant references sound; and preserve the existing distinction between mutation and copying methods. Existing rules refuse moves between differently typed positions, so general acceptance based only on the final values would itself change that policy.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Keep the first-rest approximation | Accepted initialization and later writes disagree | Rust's fixed tuple fields do not justify guessed positions | Constant-time indexed checks | Wrong slot types | Rejected |
| Permanently freeze the admission partition | Simple indexing, but a rest's permitted sequences and resizing silently narrow | Rust fixed tuples are not variable heterogeneous runs | Cheap shape/IC checks | Adds a new storage restriction and can reject sequences the declaration admits | Rejected as an implicit rule |
| Union the types that might own each index | Broad autocomplete; independent stores can remove a required suffix | Rust enum storage has a stable enum type, unlike this encoding | Cheap local checks | Does not preserve the tuple language | Rejected |
| Repartition after each write based on current values | Convenient local mutation but hidden type changes; conversions may pick their own destination | Rust prevents incompatible mutation through borrowed places; no equivalent runtime repartition | Whole-sequence membership, getter traffic and changing IC assumptions | Breaks invariant references without relocation/removal; value-only validation can bypass conversion obligations | Rejected |
| Transactionally simulate every mutation and roll back failures | A method looks atomic despite setters having run | Rust exclusive mutation is not rollback of arbitrary JS effects | Copies/logging and difficult deoptimization | Cannot roll back external setter/proxy effects; changes current failure semantics | Rejected |
| Retain the full contract and require each mutation plan to preserve location contracts; make any retyping an explicit relocation/reconstruction operation | Predictable existing references and copying alternatives; general rest mutators need precisely stated admissibility | Rust fixed fields and exclusive mutation support stable place types, but this proposal uses runtime liveness instead of borrow checking | Indexed fast paths for stable domains; guarded sequence plans for complex shapes; no speculative effects | Can preserve sequence and reference invariants together without pretending writes are transactional | **Recommended next direction**; define operation-specific plans before implementation |

This recommendation does not silently add a retyping operation. Ordinary stores must retain the location contracts they are authorized to write. A future reconstruction operation would need its own effect, identity and reference-invalidation rules. The next increment should first define a pure tuple mutation plan (direct store, length change and existing built-in mutators), reject impossible plans before conversion, then apply the selected conversions/writes in ordinary order. Method replacement and generic calls retain their existing identity guards. Default filling is a separate construction question and must not be guessed from current occupancy.

The audited consumers are `value.mts` (Set), `array-objects.mts` (definition and length), `argument-distribution.mts` (reference-location contract), `UnaryExpression.mts` (deletion), `ArrayPrototype.mts` (with/copying shapes), `ArrayPrototypeShared.mts` (search), and `runtime-types.mts` (admission and elementwise conversion). The stable forms keep these existing positional/tail consumers exact; the complex forms need a coordinated replacement.

Primary prior art: [Rust tuple types](https://doc.rust-lang.org/reference/types/tuple.html), [coercion sites](https://doc.rust-lang.org/reference/type-coercions.html), and [reference/pointer types](https://doc.rust-lang.org/reference/types/pointer.html). Rust's heterogeneous tuple has a fixed field count and type per field. These are useful analogies for the implemented subset, not a specification of variable rests, JavaScript getters or this proposal's reference liveness.

The next [mutation-plan increment](tuple-mutation-plans.md) repairs intrinsic movement and length planning for the existing positional/tail backing representation, including failure timing. General-rest representation and default construction remain open.

The [defaulted tuple increment](defaulted-tuple-admission.md) now covers no-rest tuples, retaining every declared positional type and the full admitted length interval. Default/rest combinations remain part of general assignment work.
