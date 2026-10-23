# SoA reference use and value copying

A reference into an SoA denotes the original element in the original allocation. Every use checks that location: reading it, writing it, copying its value, and decaying a Reference Value all retain the relocation and removal rules. Removing an element and later filling the same index does not revive the old borrow. Replacing an element's value, or removing another element without relocating storage, preserves a surviving borrow.

An ordinary reference-returning call decays at its call boundary unless it occupies an existing location-consuming context. For an SoA element, decay gathers a detached value, including its nested value fields. Thus `const copy = first(s)` can be changed independently of `s`. `let ref alias = first(s)` keeps the location instead. A member access on the already-decayed result of an ordinary call operates on that result; an explicit reference operand or a reference binding still reaches the original storage. This distinction uses the existing call-boundary rule, not a new access-context syntax.

Copying a value must propagate an abrupt field read as a language completion. It cannot assert that storage-backed fields are always readable. Validate the source before enumerating its fields, and retain failure propagation for the reads themselves. A failed copy does not publish a partially initialized destination. Existing earlier effects remain visible; copying introduces no transaction or rollback rule.

A whole-element write also retains the original borrow while evaluating source fields and conversions. Check it again before each resulting storage write. A source getter may relocate or remove the destination element. Such effects remain, earlier successful column writes remain, and the next invalid write fails. If the getter itself throws, that exception propagates first. Nested views retain the same original location check; keeping an old buffer reachable does not make a removed or relocated element live.

## Choice of checking boundary

The invariants are uniform use-time failure, no stale-location revival, detached value results, preserved evaluation order and effects, and no borrow-counting requirement. These directions were considered independently of implementation effort.

| Direction | Ergonomics, completion and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Catch the host assertion at a copy caller | Some bindings fail cleanly while other reads/writes still succeed; completion gives no warning of the inconsistency | Rust does not turn invalid references into valid values by catching an assertion | Cheap local change, but repeated special cases at consumers | Misses decay, direct reads, whole writes and fieldless access | Rejected: repairs one symptom |
| Refresh the borrow or gather from the current index | A familiar index read, but an old alias silently selects new storage or a replacement element | Rust vector reallocation motivates the opposite restriction | Fast current-storage lookup; loses required history | Violates original-location and removal identity | Rejected: a new borrow must be explicit |
| Copy only at selected annotated boundaries | Autocomplete still exposes the same referent type, yet adding or removing an annotation changes aliasing | Rust distinguishes borrowing from copying, not by an unrelated destination annotation | Fewer checks/copies only by omitting required behaviour | Loses unannotated copy sites and elided checks; nested aliases can escape | Rejected: check sites do not define copy sites |
| Preserve the element view through every ordinary call | A result used as a value unexpectedly aliases the source; completion cannot expose the hidden distinction | Rust distinguishes a returned borrow from a copied aggregate; the proposal already defines where its references decay | Avoids a required gather by retaining an alias | Contradicts the ordinary call-boundary decay rule | Rejected; location-consuming calls already provide the explicit alternative |
| Count borrows and refuse container mutation while any remain | Earlier errors, but unreachable or suspended references can prevent unrelated work | Superficially resembles Rust exclusivity, without Rust's ownership/lifetime analysis | Requires tracking references and synchronizing mutation with their lifetime | Contradicts the established use-time relocation rule | Rejected: changes the proposal's chosen model |
| Shared original-location checks, proper decay, and completion-aware copies | One rule across bindings, calls and imports; no new completion-visible type. Wrong guesses fail at the existing operation | Rust's borrowing/copy distinction and vector invalidation are useful analogies; its static lifetime/exclusive-borrow rules are not adopted | Generation/identity guards can be specialized or hoisted when no effect can invalidate them. Metadata need not alter user object shapes or inline-cache keys | Covers direct and nested reads/writes, decay, copy failures and reentrant effects | Recommended: preserves all invariants |

## Distinguishing removal from replacement

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Check only current length | Regrowth silently changes an alias's referent; completion cannot reveal that history | Rust's rules do not validate a removed borrow merely because an index is filled again | One bounds comparison | Cannot distinguish removed and surviving elements | Rejected |
| Invalidate every borrow on any shrink | Removing an unrelated tail unexpectedly breaks a surviving reference | Rust may conservatively disallow mutation during a borrow, but this proposal already promises surviving locations | One container epoch | Over-invalidates surviving elements | Rejected |
| Retain every removal event forever | Precise externally, with no new syntax | Unlike Rust's static approach, records runtime history | Growing history and range-query costs | Can be correct, but unbounded historical state is unnecessary | Rejected |
| Preserve an element identity until removal | Replacement and unrelated removals behave predictably; explicit reborrowing selects the new element | Captures the distinction between an element and a reused address without importing Rust lifetimes | Lazy per-borrowed-index identity or an equivalent engine representation; constant lookup, no live-reference registry or required shape changes | Removed identities never return; surviving ones persist | Recommended; the proof of concept uses lazy tokens |

## Detached nested values

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Copy the outer record but retain nested column views | A value-looking result still changes the source and can fail after relocation | Unlike copying a Rust `Copy` aggregate's value fields | Cheap, but preserves hidden aliases | Violates detached value semantics | Rejected |
| Refuse nested value fields in SoA | Removes useful existing layouts and completion members | Rust aggregates can contain copyable aggregates | Simplifies copying at the cost of unsupported layouts | Could define a restriction, but contradicts existing nested-column support | Rejected |
| Copy the complete nested value layout | Changes to the copy stay local; reference locations remain an explicit opt-out | Rust `Copy` supplies the close value-copy analogy; no Rust move semantics are added | Engines can copy a known layout in blocks; interpreter recursion is not the production cost model | Detaches nested fields and retains the original source; ordinary reference-class identity remains unchanged | Recommended, regardless of implementation effort |

Sources: [Rust Vec capacity and reallocation](https://doc.rust-lang.org/std/vec/struct.Vec.html#capacity-and-reallocation), [Rust Copy](https://doc.rust-lang.org/std/marker/trait.Copy.html). Rust's safe borrowing prevents many such programs before execution; this proposal deliberately uses garbage-collected reachability and runtime liveness checks. That limits the analogy.

The verification scope covers owned SoA relocation, element removal/regrowth, detached copies, nested views and current out-of-bounds buffer-view failures. The broader lifetime semantics of resizable buffer views are a separate audit boundary.
