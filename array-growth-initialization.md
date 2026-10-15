# Initialization on explicit length growth

This completes the existing promise that growing `[].<uint8>` through `length` supplies zeros. The promise applies to the actual backing type, including a tuple's permitted rest and suffix positions. It is not a change to ordinary untyped arrays.

## Choosing the initialization rule

Required properties: no newly exposed uninitialized typed position; defaults admitted by the actual backing type; no invented default for a non-defaultable type; independent values where copying requires it; preservation of fixed extents, surviving tuple contracts and reference liveness; existing untyped behavior unchanged.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Reject every explicit increase | `length` remains discoverable but its already documented zero-fill example fails; callers must insert values even for numeric storage | Rust exposes explicit resize methods rather than a writable length; that API does not justify withdrawing this proposal's length operation | Cheap guard, no initialization cost | Prevents holes but contradicts the existing proposal | Rejected: unnecessarily removes a promised useful operation |
| Keep holes or implicitly make new positions optional | A typed indexed read can unexpectedly produce `undefined`; autocomplete still advertises the original element type | Ordinary JavaScript arrays allow holes; Rust's initialized `Vec` prefix does not | Hole checks and optional element representations can defeat dense storage and IC assumptions | Conflicts with typed backing, invariant references and the no-holes commitment | Rejected: would require a different type/storage model |
| Fill with raw zero or unconditional `undefined` | Numeric examples look correct but strings, symbols, classes and metadata silently get wrong values | Neither JavaScript's holes nor Rust's initialized values imply a universal default | Fast clearing, but subsequent type checks cannot justify invalid representations | Zero bytes are not a default for every type; shared aggregates also violate value semantics | Rejected: unsound for existing types |
| Use the actual type's default independently at every new position | Matches the documented example and existing default declarations; no-default failure is a precise runtime boundary when the call is dynamic | Rust `resize_with` is the closest initialization analogy; Rust requires an explicit producer and exclusive access, unlike this API | O(growth) writes; primitive/default-free-effect paths can bulk initialize dense storage without changing shapes or IC contracts | Keeps initialized backing, honors metadata/default rules and value copying, and preserves survivor contracts | Selected independently of implementation effort; no new syntax or sentinel |

The implementation must not probe for a default once and then recompute it for the first element. Compute each position exactly once, in index order. `DefaultValueOf` may itself perform the existing parameterization crossing. A host-level absence of a default differs from the actual value `undefined`, which creates a present own element.

## Choosing publication and failure behavior

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Publish length first, then initialize | User callbacks can see an apparently successful resize with missing typed elements | Ordinary JS length growth exposes holes; this is outside Rust's initialized prefix guarantee | Simple length store, but requires partially initialized reads and GC handling | Violates the initialization invariant during observable effects | Rejected |
| Publish one element at a time | Failure leaves a surprising prefix of the requested resize and may already invalidate references | Incremental construction exists elsewhere; Rust's exclusive borrow prevents this particular receiver reentry | Avoids temporary values, but exposes intermediate lengths and repeated backing changes | Can preserve initialized prefixes, but adds unnecessary partial-mutation semantics to a length request | Rejected: greater user-visible failure cost without a required compatibility reason |
| Prepare first, then publish | Default effects see the prior length; failed preparation leaves the requested resize unpublished | Similar to constructing initialized values before exposing storage; not a claim that Rust promises transactional resize | Effectful defaults require temporary rooted values; a real engine can elide staging for proved pure, infallible defaults | Descriptor preflight and revalidation prevent invalid publication; relocation occurs only on success | Selected; independent user effects remain observable |

Preflight the ordinary length descriptor and extensibility after count/extent/survivor checks. Nonextensible typed arrays cannot create the required own positions. Preparing defaults neither runs inherited setters nor performs a redundant conversion. Revalidate descriptor compatibility and extensibility before publication. A descriptor requesting a nonwritable final length is applied after creating the prepared elements.

## Choosing the response to reentrant changes

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Continue using the stale plan | An inner resize or newly acquired contract is silently overwritten | Rust's ordinary exclusive mutable borrow excludes this situation; JavaScript permits callbacks | Cheapest path, but relies on an invalid assumption across calls and IC invalidation | Can publish defaults under obsolete contracts or destroy inner work | Rejected |
| Roll back callbacks or retry the plan | External effects may repeat, disappear only partly, or loop indefinitely | Neither JS effects nor Rust callbacks provide general rollback | Requires logging arbitrary state; retries can be unbounded | Cannot undo I/O, reference invalidation or externally visible effects | Rejected |
| Reject a conflicting outer plan, retaining independent effects | A single TypeError identifies a stale operation; unrelated element writes remain visible | A dynamic guard supplies the exclusion that Rust normally obtains through borrowing; it is not static borrowing | Recheck length and backing identities after each effectful preparation, plus final descriptor checks; no reference scan | Does not overwrite changed length/backing; reservation changes and unrelated value writes remain valid | Selected; no retry or speculative rollback |

Check for changed length, backing contracts or fixed extent after each successfully prepared value. Missing defaults retain their own TypeError precedence. Attribute/extensibility changes are handled by the final ordinary compatibility check. Capacity-only changes can be retained: publication uses current capacity. Growth within capacity preserves existing references; growth requiring relocation invalidates them. Preparation failure itself changes neither capacity nor generation.

The default producer also retains tuple backing on fresh default aggregates, copies declared class zeros before returning them, and computes each fixed-array element default once. These corrections prevent a nested default from escaping typed storage or sharing mutable value state. They apply to existing default consumers as well as length growth.

## Aggregate default evaluation count

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Compute once and clone the default into every position | Metadata effects occur only once even though each element is independently initialized; reference aggregates may share unless fresh allocation is specified | Rust distinguishes clone-based `resize` from producer-based `resize_with`; neither prescribes this proposal's choice | One effectful crossing, then copies; attractive for large aggregates | Can be defined soundly with complete copy rules, but differs from individual typed default initialization and dynamic nested-array freshness | Rejected for this operation: it needs an explicit repeat/clone API if desired |
| Probe once, then compute each element | A visible callback runs for a value never stored; failures have an extra effect | No requirement in Rust or ordinary JS calls for this redundant probe | Extra conversion/allocation without useful storage | Unnecessary observables and inconsistent callback counts | Rejected |
| Compute once per position, with no preliminary probe | Matches writing one defaulted position at a time, while keeping aggregate publication private | Corresponds to producer-based initialization; JS effects and metadata remain the relevant differences | Pure identical defaults can still be bulk filled; effectful producers run exactly as specified | Preserves fresh nested aggregates and deterministic effect order; each default is already of its type | Selected; fixed arrays and tuples use the same rule |

## Scope of this increment and the next task

The implemented boundary is explicit length growth through assignment and reflected definition/set. Proxy eligibility is unchanged. Fixed extents and suffix survivor checks still precede preparation. The implementation retains the interpreter's existing ordinary-Array size ceiling; it does not claim to implement the full index-type allocation range.

Skipped-index stores and descriptors without supplied values still need an insertion plan. They cannot simply be routed through this default-based resize rule: `unshift` and growing `splice` write high destinations before lower ones, and valid insertion into a function array must work without a function default. The next increment must resolve operation-aware initialization/publication and failure visibility for those writes, including getter/proxy effects. That work is required before the broader no-holes audit is closed. Multiple, finite/nested and defaulted tuple-rest representations remain separate work.

Primary references: [Rust Vec guarantees](https://doc.rust-lang.org/std/vec/struct.Vec.html#guarantees), [resize_with](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.resize_with), [set_len](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.set_len), and [ECMAScript ArraySetLength](https://tc39.es/ecma262/2026/multipage/ordinary-and-exotic-objects-behaviours.html#sec-arraysetlength). Rust is used for initialized-storage and borrowing comparisons, not as authority for JavaScript property/effect ordering.
