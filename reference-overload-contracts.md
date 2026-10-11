# Reference contracts in overload selection

```js
function update(ref ...xs: [uint8, string]): string {
  xs[0] = 2;
  xs[1] = "after";
  return "updated";
}
function update(ref ...xs: [boolean, uint8]): string {
  return "other";
}
let n: uint8 = 1;
let s: string = "before";
update(ref n, ref s); // "updated"; n is 2 and s is "after"
```

Adding another signature must preserve the reference contract of this call. Selection retains the supplied borrow, its available invariant storage type and the current referent type as distinct facts. A reference candidate requires the original location and admits its current value without conversion. A value candidate judges the decayed value under the existing value rules. No candidate gets a temporary location manufactured to make it viable.

The completed argument assignment supplies each argument's receiving leaf contract and owning declared parameter. A heterogeneous reference tuple is not replaced by its element union; a nested rest retains its actual positional assignment. Ranking and specificity use that same assignment. A fixed parameter remains more specific than a rest position when the other existing comparisons tie. Declared extents need not be expanded: only the actual argument sequence needs receiving descriptors.

The overload dispatcher forwards the original contributions to the selected function. It is not a value parameter boundary. The selected function still binds and rechecks its own parameters, and subsequent accesses still check liveness. Defaults and argument effects can invalidate a location after an earlier matching pass. Reflection through `call`, `apply` and `bind` retains its existing value boundaries.

## Preserving the contract

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Rank erased referent values and reconstruct borrows later | A suggested reference overload can win without the caller supplying a borrow; writes may reach invented storage. | Rust distinguishes reference types from referent types. Its first-class references differ from this proposal. | Simple value dispatch, followed by reconstruction and extra storage; no justified shape or IC advantage. | Loses calling convention, invariant storage and original location identity. | Reject. |
| Ban reference overloads or require casts at each call | A working single declaration stops composing when another signature is added. | Rust lacks ordinary free-function overloading; that absence does not constrain this proposal's supported overloads. | Reduces implementation work without a necessary layout benefit. | Contradicts supported reference and callable contracts. | Reject. |
| Retain borrow mode and the actual receiving contract through selection and dispatch | Displayed parameter positions predict selection and caller writes in direct, stored and dynamic calls. Wrong arguments fail before the selected body. | Rust's typed places and tuple positions are useful limited analogies; ECMAScript evaluation and this proposal determine dispatch and liveness. | Share compiler-owned descriptors, specialize fixed cases, compare extents directly, and retain ordinary call ICs. No user-object fields or shape transitions are required. | Preserves invariance, no-conversion, location identity, specificity and independent final liveness checks. | **Choose independently of effort.** |

## Reference versus value ties

An explicit borrow may reach a value parameter by decay. It does not acquire a new preference over an equally ranked reference parameter. For example, an otherwise indistinguishable `f(ref x: uint8)` and `f(x: uint8)` cannot be selected merely by seeing `ref n`. Existing ambiguity rules apply, including declaration checking where ambiguity is proved there. This increment does not redefine when the implementation proves declaration overlap.

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Always prefer a reference parameter for explicit `ref` | The spelling suggests a preference, but adding an overload can silently change an existing value call into a mutating alias call. | Rust distinguishes `T` and reference types but supplies no equivalent free-function overload tie-break. | A cheap mode bit; no shape/IC requirement. | Introduces a ranking rule beyond the existing decay and specificity commitments. | Reject for this design. |
| Reject decay to value candidates during overload selection | Autocomplete requires different call spellings for a single function and a group. | Rust's coercion rules do not specify this proposal's explicit decay boundaries. | Simple filter, no necessary layout benefit. | Makes a value signature cease accepting an otherwise valid argument solely because it is overloaded. | Reject. |
| Keep existing ranks and specificity; reject any remaining ambiguity | Both accepting signatures remain visible and an unresolved choice requires disambiguation instead of an implicit mutation preference. | Rust only supports the distinction between reference and value contracts here; this proposal defines ranking. | Reuses the existing selection descriptors and call sites; no extra shape transitions. | Preserves value decay, reference admission, selected-owner identity and ambiguity semantics. Liveness remains separately enforced. | **Choose independently of effort.** |

Primary comparisons: [Rust pointer types](https://doc.rust-lang.org/reference/types/pointer.html), [Rust tuple types](https://doc.rust-lang.org/reference/types/tuple.html), and [Rust associated functions and methods](https://doc.rust-lang.org/reference/items/associated-items.html). Rust's lifetime, exclusivity and method lookup rules are not adopted here.

Implementation coverage includes both declaration orders, direct/stored/erased calls, fixed references and reference rests, methods and constructors, generic fallback controls, named calls with uniquely selecting names, original-storage writes, reflected value calls and intervening invalidation. [Named overload selection](named-overload-selection.md) extends candidate binding to shared names and preserves the winning groups. Structural-property storage metadata, empty-run forwarding provenance and the broader generic application audit remain open; these examples do not close those gates.
