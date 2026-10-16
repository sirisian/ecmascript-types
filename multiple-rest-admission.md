# Collected extents in tuples with multiple rests

A finite array rest contributes its exact extent even when dynamic rests occur on both sides. Element membership alone does not establish that collected sequence contract.

```js
type T = [...[].<number>, ...[2].<string>, ...[].<boolean>];
const short: any = [1, "s", true];
const exact: any = [1, "s", "t", true];
const long: any = [1, "s", "t", "u", true];
short is T; // false
exact is T; // true
long is T;  // false
```

These are membership examples. General conversion and writable backing for multiple dynamic rests remain separate work; this increment does not make the old first-rest backing approximation correct.

## Assignment decision

A valid solution must preserve exact finite counts, zero extents, ordered tuple expansion, leftmost-greedy giveback, positional omission, and the existing shared sequence matcher. It must not allocate one type record per finite position or use a successful membership partition as a new storage contract.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Check only each item's rest element type | A displayed extent of two accepts one or three; the wrong guess succeeds silently | Rust `[T; N]` retains N; homogeneous Rust slices do not supply an exact count. TypeScript's rest syntax does not imply finite counts | Cheap but omits a required judgment | Loses the collected contract and can feed an invalid assignment into later admission | Rejected |
| Check the finite count only after the first greedy assignment | A valid same-type sequence can fail because an earlier rest took values needed by the finite rest | Rust has no equivalent competing-rest tuple matcher; TypeScript permits only one unbounded rest in the cited form | Cheap post-check, but retrying requires the search that this direction omits | Rejects valid giveback; tuples and function/reference-rest matching disagree | Rejected |
| Expand N required slots before matching | Results can be correct but a compact type can consume enormous memory before receiving values | Rust fixed arrays retain their length without N distinct type fields | O(N) metadata, poor cache and allocation behavior | No benefit to reference correctness; conflicts with compact finite-domain design | Rejected |
| Apply the collected length constraint inside the existing run predicate | Extent and membership agree, including zero and same-type giveback; completion still describes the written types | Combines Rust's exact-count principle with this proposal's own greedy matcher. TypeScript's erased tuples and single-rest restriction are a limited analogy | Compact count metadata; search state depends on written slots and actual input length, not the finite extent's magnitude | Retains whole-run admission and existing evidence; changes no location contract or liveness rule | **Selected**, independently of implementation effort |

For a finite array rest, the run predicate requires its count to equal the extent. A rejected count continues ordinary greedy giveback; it is not a final verdict about the whole sequence. Tuple spread operands retain ordered expansion. For a defaulted fixed position, a zero count is valid only when the input is already exhausted. A supplied incompatible value cannot skip that position and reach a later rest.

## Length-proof decision

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Derive minimum length only from positions before the first rest | Required finite runs and fixed suffixes disappear from the diagnostic | Rust arrays retain their exact count irrespective of surrounding fields | Cheap but incomplete | Invalid short literals reach execution, including inside uncalled bodies | Rejected |
| Read elements or execute conversions to discover arity | A length error invokes indexed getters and can alter state before an avoidable failure | Rust length checks use container/count facts; JavaScript observable getters make the difference material | Unnecessary loads, calls and IC dependencies | Introduces effects into a count-only proof | Rejected |
| Sum compact declared length ranges independently of positional backing | A finite run contributes its count wherever it occurs; errors point to an impossible length | Rust supplies the count principle; this proposal supplies optional positions and multiple rests | O(written structure) metadata traversal, with no per-count expansion or element reads | Valid negative proof without claiming a unique assignment, new backing, or reference authority | **Selected** |

Fixed required positions contribute one, defaulted positions zero to one, finite rests N to N, dynamic rests zero to unbounded, and nested tuple operands their ordered contents. Unknown extents and recursive active expansion supply no new proof. Supported count arithmetic remains compact; this does not complete full uint64 allocation or opaque-count support.

When these facts prove a runtime length impossible, membership refuses before indexed Gets. Implicit and explicit tuple conversion refuse impossible arity before indexed conversion. At an array literal with an established exact count, including an established fixed spread, the same contradiction is a type error even in an uncalled body. An unknown iterator supplies no exact-count proof. A possible total length does not prove that types can be assigned to every collected run.

## Backing work still required

The observed `[number..., string...]` backing defect remains: a direct boundary stamps the first rest, while nested acquisition can leave the sequence unprotected. Correcting the matcher is a prerequisite, not a storage solution. Do not freeze a selected value partition, flatten the alternatives into independently writable unions, or rewrite a surviving reference's contract using current values. The next design must specify complete correlated storage, ordinary mutation/failure effects, conversion choice and reference authority together. General literal contextual distribution and conversion selection also still use first-rest approximations and require their own repair.

Primary prior art: [Rust array types](https://doc.rust-lang.org/reference/types/array.html), [Rust slices](https://doc.rust-lang.org/reference/types/slice.html), [Rust variance](https://doc.rust-lang.org/reference/subtyping.html), and [TypeScript 4.2 tuple rests](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-2.html). Rust's mutable references are invariant in their referent type, and its slices are homogeneous; neither models this proposal's multiple-rest mutable sequence. TypeScript's erased tuple types do not enforce runtime backing or reference liveness.
