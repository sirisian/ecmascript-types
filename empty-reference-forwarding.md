# Forwarding empty reference runs

A reference run remains second-class when its length is zero. Expanding it contributes no value, but still requires a receiving `ref` rest under the callable contract in view.

```js
function values(...ys: [].<uint8>) { return ys.length; }
function refs(ref ...ys: [].<uint8>) { return ys.length; }
function relay(ref ...xs: [].<uint8>) {
  refs(...xs);   // Valid when xs is empty too; supplies zero arguments.
  values(...xs); // Invalid forwarding, including when xs is empty.
}
```

Evaluate written arguments once in source order. Retain the source boundary of every empty reference-run expansion separately from the expanded argument list. Bind the actual arguments with both their ordinary type contracts and their forwarding permissions, preserving names and choosing the lexicographically greatest valid rest-count list. Nonempty forwarded entries must reach reference rests. An empty occurrence before any named contribution must touch a reference rest's selected positional interval: its start, interior or end all qualify, including a zero-length interval. Parameters pinned by names or their supplied suffixes do not consume this positional cursor. After names have begun, the occurrence must belong to the currently open named reference rest. It cannot reopen a closed named rest or pass through a fixed named parameter.

These occurrences add no argument, location, rank or count. Their permissions constrain the valid assignments of the actual values, just as reference and type requirements do. At an overload, forwarding compatibility is part of each candidate's binding and viability before ranking. It gives no ranking advantage among compatible candidates. A failed forwarding check runs no parameter initializer or callee body. Earlier argument effects and abrupt completions retain their ordinary order. Successful distribution does not establish later reference liveness.

A callable view governs the forwarding permission. A function implementing that view may use individual `ref` parameters as already specified. Once the forwarding contract is satisfied, its entries reach that implementation as ordinary reference arguments; the implementation need not spell a rest. A zero-length ref-rest view can likewise be implemented by a zero-parameter function. This does not permit a direct call with a run to an arbitrary fixed-parameter function, or forwarding through an ordinary reflective builtin's value boundary.

## Direction: retain empty occurrences

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Discard empty occurrences | A value-rest signature silently accepts a forwarding form it cannot accept when populated. | Rust has no corresponding second-class run; its first-class references do not justify this exception. | Cheap but removes required contract evidence; no useful shape benefit. | Loses the run's forwarding restriction. | Reject. |
| Insert a sentinel argument or temporary reference | Autocomplete's arity and defaults no longer describe the actual call. | Rust references are values, not synthetic ECMAScript parameters. | Adds allocation and dispatch cases. | Invents an argument/location, affects defaults and `arguments.length`, and risks escape changes. | Reject. |
| Retain invocation-owned occurrence metadata independently of entries | The same displayed reference-rest contract governs zero and nonzero runs. | Rust supports the limited distinction between a typed contract and its supplied values; this proposal supplies its own run model. | Compiler-owned descriptors can be eliminated at proven calls; no user-object fields, shape transitions or member-IC cases. | Preserves zero arity, evaluation order, reference identity and independent liveness checks. | **Choose independently of effort.** |

## Direction: place an empty occurrence

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Accept if any reference rest exists anywhere | A run before an unrelated fixed prefix appears to target a later rest. | Rust has no named/rest routing analogue endorsing this. | Cheap scan; layout unchanged. | Ignores source position and named grouping. | Reject. |
| Check permission only after choosing a distribution by value types | A defaulted value prefix can consume a forwarded reference, rejecting a call whose later reference rest could accept it. An empty occurrence can likewise require skipping an optional prefix. | Rust's reference contracts participate in admission; its different rest model does not justify ignoring this proposal's forwarding requirement. | Saves a little matching state but may report false failures; no shape advantage. | Fails to find the greatest assignment satisfying all required contracts. | Reject; the first implementation attempt exposed this with a defaulted String prefix. |
| Include entry and empty-boundary permissions in distribution | A rest can receive an empty run at either endpoint; valid giveback around defaults remains available and named rest scope remains explicit. | Rust is only a typed-contract comparison; the proposal's own BindArguments and SequenceAssignment determine the valid greedy assignment. | Match compiler-owned offsets alongside values; no extra user reads or shapes. Memoization retains whether the current boundary has already been admitted. | Preserves actual arity and ranks while satisfying every receiving obligation; liveness remains separate. | **Choose independently of effort.** Multiple qualifying zero-width rests require no observable tie-break because none receives a value or gains a rank. |

For example, `g(...xs, "L")` with empty `xs` and parameters `(first: string = "d", ref ...ys: [].<uint8>, last: string = "t")` binds zero arguments to `first`, zero to `ys`, and `"L"` to `last`. Taking `"L"` at `first` would cross the empty occurrence before any eligible rest. With an ordinary empty iterable instead, the occurrence does not exist and `first` takes `"L"`. Neither call gains a synthetic argument.

## Direction: choose the governing contract

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Require the implementation itself to spell a reference rest | A callback accepted under the displayed view fails solely because its implementation uses individual refs. | Rust has no equivalent rest syntax; it supplies no reason to revoke this proposal's callback compatibility. | Simple guard but needless adapters, no shape advantage. | Contradicts the existing callback rule. | Reject. |
| Permit runs at every fixed reference parameter or zero-parameter function | Expands the permitted forwarding forms beyond the declared view, with unclear zero-length destinations. | Rust's first-class references are too different to justify changing the second-class model. | Broader routing and escape cases; no required IC gain. | Discards the requirement for a reference-rest contract. | Reject. |
| Validate the effective view, then preserve individual references to the implementation | Autocomplete and invocation agree; individual-ref and zero-parameter implementations continue to work. | Rust's function contracts are a limited comparison; ECMAScript callable views and runtime checks remain distinct. | A proven view can lower directly, while dynamic candidates keep invocation metadata. No user shape changes. | Preserves callback compatibility, invariant admission, final binding and liveness. | **Choose independently of effort.** |

Primary comparisons: [Rust pointer types](https://doc.rust-lang.org/reference/types/pointer.html) explicitly make references first-class values. [Rust functions](https://doc.rust-lang.org/reference/items/functions.html) describe typed parameters and C variadics, which are not this proposal's second-class reference-rest facility.

## Implementation boundary

The matcher records the next unaccepted empty boundary in its invocation-local memo key. Caching only the slot and argument indices is rejected: equal indices can be reached with that boundary already admitted by a preceding rest or still awaiting one. Ergonomically this would cause order-dependent false failures; Rust supplies no analogy authorizing omission of a determining input; production memoization needs at most the two states at a given boundary (admitted or pending), with no shape/IC effect; correctness requires distinct states without granting liveness; therefore retain this input independently of effort. The test uses exhaustive complete count-vector enumeration as a separate oracle for the memoized search.

The engine retains empty source offsets in an invocation-owned WeakMap and nonempty entry provenance alongside the existing argument items. Positional calls, named calls, constructors and overload dispatch share the check. The ordinary positional path creates entry descriptors only when it encounters a run. A view in a lexical binding or typed property validates forwarding before its implementation receives the locations; reading the property contract performs no additional property lookup. Positional omissions retain zero counts, so forwarding adds no `undefined` to `arguments`.

The focused runtime repair does not certify all static forwarding proofs, all callable-view origins, view-to-implementation transport across different non-final rest layouts, generic adapters, or variable-only object-spread entry. These remain separate contract/provenance audits; no successful mapping removes final liveness checks.
