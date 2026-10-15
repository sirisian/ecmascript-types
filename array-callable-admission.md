# Callable admission in homogeneous arrays

A concrete element contract must retain the selected callable body even when membership already succeeds:

```js
function id<T: type>(value: T): T { return value; }
const functions: [].<(value: string) => string> = [id];
functions[0] === id.<string>; // true
```

This is an in-place boundary on an already compatible container. Retain its length/extent decision, each observed element value, and each selected union contract. Complete specialization after membership, using ordinary element Set only for a changed value. Preserve aliases and custom prototypes. Empty arrays still acquire their element contract. Nested arrays use the same rule, including arrays reached through object properties and the selected tuple assignment. Independent membership and reference/overload viability do not specialize, copy, write or acquire storage.

Existing element or tuple backing contracts remain authoritative. A later view cannot overwrite them. In particular, specializing a generic function in a slot promising a generic function fails when the replacement violates that promise. A broad backing alias may reintroduce a generic function; admission at a concrete view must then materialize again. An unchanged concrete element needs no write even in a frozen array. Ordinary setters, proxies, non-writable storage and exceptions retain their usual behavior.

New homogeneous storage is published after this admission's required writes succeed. Prior ordinary effects survive a later failure, but the receiving binding and this admission's new contracts are not published. Reentrant independent boundaries retain their own effects and contracts. Repeated paths to one location combine their obligations; equal consecutive observations of that location share the write. An array acquisition uses a homogeneous element contract rather than a property contract for each occupied index, so later growth and deletion retain the existing array rules.

Value-class instances retained when an untyped Array acquires owned element storage must be copied as required by the existing store rule. Use the retained observation and ordinary Set; do not reread an accessor or replace it with a data descriptor. Passing an already typed array through another receiving view is not a new store of all its elements. Copying its entire contents at that boundary would silently replace existing places and disagree with elision. Independent elementwise conversion that creates a new array remains a separate path.

## Decision: where to retain the callable

| Direction | Ergonomics, autocomplete, wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Leave compatible generic elements unchanged | Autocomplete promises concrete calls that erasure can lose | Rust selects function items in coercion contexts; it does not keep an unresolved runtime generic here | Cheapest boundary, but leaves later dispatch unresolved | Does not fulfill the concrete body-selection contract | Rejected: missing required behavior |
| Specialize during membership | A mere `is` test can mutate the array and throw from specialization | Rust coercion sites are not independent membership tests | Speculative alternatives create/cache bodies and perform writes | Leaks unselected alternatives and changes reference viability | Rejected: wrong phase |
| Reread and convert elements after membership | Accessors can supply a different function than the one admitted | Rust has no JavaScript accessor analogy | Second element traversal; repeated accessor/IC traffic | Loses selected observations and can repeat effects | Rejected: observable replay |
| Copy or wrap an already compatible array | Aliases diverge; wrapper behavior surprises callers | Rust owned array construction differs from passing a borrowed array | Allocation/copy or wrapper shapes and dispatch | Changes container identity and reference origins | Rejected: wrong storage semantics |
| Retain evidence, specialize, ordinary Set | Concrete element behavior survives erasure; a refused write fails at the boundary | Rust propagates array-literal coercion context and coerces function items to pointers; dynamic alias admission is a weaker analogy | One membership traversal, transient evidence, cached specialization, existing element IC paths | Preserves admitted values, aliases, ordinary writes and liveness rules | **Selected**, independently of implementation effort |

## Decision: storage acquisition and value copies

| Direction | Ergonomics, autocomplete, wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Replace the backing element contract at every view | A widened view silently changes what other aliases can store | Rust does not retag shared mutable storage this way | Repeated element-kind/shape invalidation | Breaks established backing promises | Rejected |
| Give each observed index a new property contract | Existing indices and future indices behave differently despite one homogeneous type | Rust arrays have one element type, not per-occupancy field declarations | Per-index metadata and property IC pressure | Incorrect growth/deletion domain and stale contracts after resizing | Rejected |
| Copy every value-class element at every receiving boundary | Passing an array unexpectedly replaces stored instances | Rust copies owned values when copied, not every element when a reference is passed | O(n) copies even for a compatible view | Elision and existing-place behavior disagree | Rejected |
| Use DefineOwnProperty to force replacement | Accessors disappear or non-writable refusal can be ignored | No Rust analogue for JS descriptors | Can bypass setters but changes shapes/descriptors | Bypasses ordinary observable storage semantics | Rejected |
| Preserve backing; acquire homogeneous storage once; copy retained values at acquisition | One stable storage promise; normal setters and predictable copy positions | Rust distinguishes owned values from borrows; its ownership/lifetime model is not adopted here | One contract per array, copies only at actual acquisition, no wrapper | Preserves existing contracts, effects, aliases and reference checks | **Selected**; ordinary failure publishes no new contract |

## Tuple follow-up

Required fixed positions and a single trailing unbounded homogeneous array rest now use retained admission; see [tuple callable admission](tuple-callable-admission.md). Other tuple elements remain separate implementation work. The current interpreter's first-rest positional stamp disagrees with SequenceAssignment for non-final or multiple rests. A correction must cover later writes, resizing, deletion, reference admission and array methods, not just initial callable selection.

| Direction | Ergonomics, autocomplete, wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Keep the first-rest approximation | A valid tail element can acquire an unrelated write type | Rust fixed tuples have exact positions; no support for this approximation | Cheap indexed checks | Can contradict the admitted assignment | Rejected |
| Permanently freeze one successful assignment | Convenient initial indexing, but legal rest redistribution or resizing can unexpectedly fail | Rust fixed tuples do not model variable heterogeneous rest runs | Cheap indexed checks after admission | Adds a storage restriction the proposal does not currently state; needs a full mutation/liveness contract | Rejected as an implicit fix |
| Give ambiguous positions a union of slot types | Autocomplete is broad; stores can destroy required suffixes | Rust offers no corresponding variadic tuple storage | Cheap local checks | Independent unions do not preserve the sequence language | Rejected |
| Retain the full tuple contract; use selected evidence for admission and validate mutations against the required sequence | Initial body selection follows the declared greedy assignment; mutations preserve the tuple contract | Rust's exact tuple positions are analogous only for the fixed subset | Can need sequence analysis for ambiguous mutations; fixed shapes can retain indexed fast paths | Preserves declared sequences without inventing a frozen partition; must retain reference identity/liveness through effects | **Recommended for the next design/implementation increment** |

This recommendation is independent of cost. The tuple admission audit further requires preserving existing location contracts: a successful repartition alone cannot retype a live invariant reference. It is not yet a complete mutation algorithm: the next task must settle observable candidate conversion, atomicity/effect order and which existing operations change the sequence, with the same five criteria. This increment does not claim to repair tuple storage or combined callable signatures.

Primary prior art: [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html) and [function item types](https://doc.rust-lang.org/reference/types/function-item.html). Rust's fixed array and tuple construction contexts support propagating expected element types; they do not specify runtime getters, alias retagging, proxies, or JavaScript reference liveness.
