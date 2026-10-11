# Named calls select a candidate and its bindings together

```js
function update(ref ...xs: [uint8], label: string): string {
  xs[0] = 2;
  return "integer";
}
function update(ref ...xs: [string], label: string): string {
  xs[0] = label;
  return "string";
}
let text: string = "before";
update(ref text, label: "after"); // "string"; text becomes "after"
```

Both declarations contain the name `label`. The first name match is not enough to select a signature: each candidate must bind the same supplied contributions against its own parameters, then satisfy their types, calling conventions and complete rest contracts. Declaration order cannot decide the result. The same rule selects among signatures in a declared callable view; the view still supplies names and defaults without replacing the function value being called.

Evaluate argument expressions and spread contributions once, in source order. Retain each contribution's source position, optional name, value or original borrowed location, and available forwarding information. Candidate binding maps these contributions to parameter groups without replaying expressions, running value parameter initializers or converting values to test a candidate. Rank admitted candidates using the proposal's existing rules. Compare the same source contribution across candidates even when its parameter position changes; omitted defaults supply no ranking contribution. Existing repeated-name assignment rules remain: earlier expressions still run, but a replaced contribution does not rank as an argument reaching the callee.

Preserve the selected candidate's groups into its parameter initialization. Flattening them and running positional selection or greedy rest distribution again can pick another body or move values out of a named rest. For example, `f(second: "b")` must retain an empty `first` and a one-element `second` when both parameters are String rests. Keeping those counts does not retain an admission proof: final parameter checks, initializers and later reference accesses still run under their own rules.

## Candidate selection

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep the first signature containing the names | Suggestions show accepted types which may be unreachable; reordering declarations changes valid calls. | C# tests argument correspondence and conversions across candidates. Rust has no comparable ordinary named-argument overload set. | A cheap lookup can cache the wrong target; no useful shape advantage. | Ignores viable alternatives and the selected reference contract. | Reject; remove the former first-match rule for callable views too. |
| Require unique labels across overloads | Existing APIs need arbitrary names or casts even where types clearly distinguish the call. | Neither C# candidate admission nor Rust's different function model justifies this restriction. | Simplifies implementation without a necessary layout benefit. | Conflicts with names being routing information rather than signature identity. | Reject. |
| Retry evaluation and conversion for each candidate | Getters, iterators and user code can run repeatedly, including in losing candidates. | C# evaluates arguments in source order. Rust type-directed checks do not license JavaScript effect replay. | Repeated user calls and allocations can invalidate shapes and ICs. | Violates evaluation order, abrupt completion and reference-liveness rules; conversion cannot manufacture borrowed storage. | Reject. |
| Evaluate once, retain candidate-specific bindings, and rank all viable candidates | Autocomplete's names and types predict the selected body in either declaration order; wrong guesses fail before body entry. | C# is a useful comparison for candidate-specific name/type admission and explicit-argument ranking. Rust provides a limited typed-place comparison, not this dispatch syntax. | Compiler-owned source indices and binding descriptors support cached mappings and direct call ICs without adding user-object fields or changing shapes. | Preserves candidate identity, correlation, effects, ambiguity, invariant storage and independent final liveness. | **Choose independently of effort.** |

## Transporting the selected groups

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Flatten and rerun ordinary positional selection/distribution | Naming a rest can still put values in a different rest or invoke another body. | C# does not use the named order as a second positional call. Rust's function model supplies no reason to reinterpret a selected binding. | Discards reusable work and repeats matching; no required shape benefit. | Loses the selected assignment and can alter defaults or reference receivers. | Reject. |
| Store a single pending binding on the function or globally | Nested and recursive calls may consume another invocation's mapping. | Neither comparison makes argument bindings mutable shared function state. | Shared writes and invalidation inhibit stable call ICs; user-visible fields would alter shapes. | Breaks invocation isolation and abrupt cleanup. | Reject. |
| Carry an invocation-owned binding descriptor to the selected parameter initialization | Named groups, omitted slots and receiver identity remain stable through nested calls. | Rust's parameter binding offers only a limited analogy; this proposal defines multiple rests and runtime borrowing. | Engines can lower a known descriptor directly to registers/stack slots. The interpreter uses argument-list WeakMaps, not user-visible properties. | Retains counts and original values while allowing each final contract and liveness check to run after intervening effects. | **Choose independently of effort.** |

Primary comparisons: [C# named and optional arguments](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments), [Rust functions](https://doc.rust-lang.org/reference/items/functions.html), and [Rust references](https://doc.rust-lang.org/reference/types/pointer.html). This proposal does not adopt C#'s omitted-optional-parameter tie-break or argument-order restrictions, nor Rust's first-class references, exclusivity or lifetime rules.

The implementation's covered paths include direct/stored/erased groups, reordered parameter names, methods, constructors, fixed callable-view defaults, reference rests, ambiguity, once-only effects and selected default invalidation. Variable-only object-spread entry, full callable-view transport across different rest layouts, generic application completeness, structural-property storage metadata, empty-run provenance and broader declaration-overlap/resource gates remain separate audits.
