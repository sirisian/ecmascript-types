# Reference-rest sequence contracts

```js
function update(ref ...xs: [uint8, string], label: string) {
  xs[0] = 2;
  xs[1] = label;
}
let n: uint8 = 1;
let s: string = "before";
update(ref n, ref s, label: "after");
```

The rest collects two original locations, with uint8 at the first position and string at the second. A displayed tuple contract must work in direct, named, stored, generic and dynamic calls. Swapped locations, wrong extents and missing borrows fail before the callee body. A location's invariant contract is its declared storage type where available, not merely the narrower value currently stored there. Explicit `any` remains subject to runtime admission.

Distribution chooses the lexicographically greatest parameter-count list among assignments satisfying the complete contracts. For a reference rest, an array contract checks its element type and stated extent; a tuple checks its positional pattern, including nested rests, with the same greedy giveback rule. Expanding a billion-element extent into a billion host slots is unnecessary: compare the supplied count with the extent. The selected run is checked again at final binding. Earlier defaults and argument effects can invalidate a previously admitted location; no cached partition proves later liveness.

## Alternatives and decision

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Match every location against the tuple's element union | Suggestions show positional types while swapped or shortened arguments may pass, and valid invariant locations may be rejected. | Rust tuples retain distinct positional field types; erasing them is not a close analogy. | Cheap comparison but incorrect partitions; no useful shape or inline-cache benefit. | Loses position correlation and extent, and says nothing about liveness. | Reject. |
| Ban heterogeneous reference rests | Valid tuple and generic-pack callbacks require rewrites. | Rust does not provide this rest syntax, so its restriction cannot be inferred. | Simpler matching without a necessary object-layout benefit. | Conflicts with supported tuple/pack contracts and leaves homogeneous final admission unchecked. | Reject. |
| Convert referents or allocate temporary borrowed slots to fit | A promised alias may silently write a copy. | Rust permits some temporary borrowing and has first-class references; neither is this proposal's model. | Extra allocation, reads and copies; potentially new object access paths. | Violates invariant storage, no-conversion, identity and second-class escape rules. | Reject. |
| Match the complete sequence and enforce it at final binding | Every call spelling preserves the displayed position and extent contract; a wrong guess identifies the incompatible location. | Rust provides a limited analogy for typed tuple positions and references to places; ECMAScript effects and this proposal govern runtime liveness and forwarding. | Share compiler-owned sequence descriptors, specialize fixed cases, memoize by the actual input run, and compare extents without expansion. No new user-object fields or shape/IC transitions are required. | Preserves correlation, arity, location identity, invariant types, effect order and independent final liveness checks. | **Choose independently of effort.** |

## Omitted tuple defaults

A defaulted trailing tuple position makes omission permissible under the tuple's existing length rule. For a reference rest, omission contributes no location: the run length is the number of supplied borrows. Reading that omitted position remains out of range. A supplied location is checked normally and is never replaced with the default. This does not change when the tuple type's compile-time default expression is resolved.

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Require callers to supply defaulted positions for ref rests | The same tuple annotation unexpectedly changes its minimum arity. | Rust tuples have no comparable defaults; Rust provides no justification for changing this rule. | Simple arity check, no shape/IC advantage over omission. | Unnecessarily changes the existing omission contract. | Reject. |
| Materialize defaults as hidden borrowed locations | The run appears longer but writes cannot reach caller storage. | Rust's temporary lifetime extension is not this proposal's second-class borrowing rule. | Allocations and new storage lifetimes, with no necessary optimization benefit. | Invents locations and alias semantics not supplied by the caller. | Reject. |
| Preserve omission and collect only supplied locations | Optional positions may be absent; length and bounds checks explain what can be accessed. | Rust's distinction between tuple field types and reference values is only a limited comparison; it does not specify these defaults. | No temporary storage or added shapes; compare actual run length. | Preserves tuple length admission, original locations and liveness; creates no new reference. | **Choose independently of effort.** |

Primary comparisons: [Rust tuples](https://doc.rust-lang.org/reference/types/tuple.html) associate a type with each position and distinguish tuple arity; [Rust pointer types](https://doc.rust-lang.org/reference/types/pointer.html) make references first-class. This proposal does not adopt Rust's exclusivity, lifetime or temporary-borrow rules.

The implementation keeps run-admission caches within one matching invocation and its fixed admission facts. It does not reuse those facts across parameter initializers or later accesses. [Reference contracts in overload selection](reference-overload-contracts.md) carries the selected positional contract through ranking and dispatch. General forwarding escape coverage, unresolved spread proofs and complete evaluation-budget accounting remain separate audits.
