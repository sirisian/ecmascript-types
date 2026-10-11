# Structural boundaries preserve property storage contracts

```js
function update(ref ...xs: [uint8]): void { xs[0] = 2; }
let object: { x: uint8 | string } = { x: (1 := uint8) };
const dynamic: any = update;
dynamic(ref object.x); // TypeError: the location has a union storage contract.
```

A property's current value and its storage contract answer different questions. A typed structural boundary must retain the contracts acquired there even when every current value already satisfies its type. The current `uint8` above does not turn the property into `uint8` storage. Ordinary and reflective writes, deletion and reference admission must consult the same backing contract.

An existing backing contract survives later views. A `{ x: uint8 }` object seen as `{ readonly x: uint8 | string }` still has `uint8` storage. Conversely, a narrower readonly view of an existing union-typed property does not narrow that storage. An object type's `readonly` remains a view permission; recording a value contract does not freeze the object or add readonly permission to other aliases.

## Which contract is retained?

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat the current value as the permanent storage type | Suggestions vary with the last value, and a successful narrow borrow hides a wider writable location. | Rust distinguishes a place's type from its contents; TypeScript's erased annotations cannot supply runtime storage evidence. | Cheap current-value test, but unstable specialization assumptions; shapes alone do not establish the permitted value domain. | Violates reference invariance and later stores. | Reject. |
| Replace the backing type at each structural view | Passing an alias to a function can change other aliases' accepted writes. | Rust mutable references are invariant in their target; TypeScript allows readonly aliases to observe mutation but does not enforce this proposal's runtime contracts. | Repeated contract changes invalidate field/store IC assumptions. | Confuses the receiving view with storage ownership and can widen or narrow existing backing silently. | Reject. |
| Copy or wrap every structural crossing | Identity, prototypes and extra properties cease to behave as the proposal specifies; wrong guesses create surprising alias splits. | Neither comparison supplies a matching implicit copy rule for these JavaScript objects. | Allocation and wrapper dispatch; additional shapes and polymorphic ICs. | Breaks existing in-place conversion and identity rules. | Reject. |
| Install missing acquired contracts and preserve existing backing contracts | Declared member types remain useful through dynamic aliases; wider views do not change what a write can store. | Rust's target invariance is a useful limited comparison; JavaScript structural admission and runtime checks remain this proposal's rules. | Engine-owned descriptors or shared typed shapes can support stable store ICs without enumerable user properties. | Preserves identity, conversion rules, backing invariance and independent reference liveness. | **Choose independently of effort.** |

## Acquisition, effects and failed checks

Retain the object identities, property keys and resolved types already traversed by a successful structural admission. A boundary installs missing contracts from that evidence. Nested structural checks carry their own evidence. A failed or abrupt membership branch contributes none; a union retains its successful alternative, an intersection retains its successful member checks, and tuple-rest admission retains only the selected assignment. A pure `is` or `instanceof` query never acquires storage contracts. Neither does overload/reference membership probing.

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Reread members after success to discover their storage types | Getters can run twice and return a different object merely because an annotation is present. | Rust's compile-time place analysis does not authorize repeated JavaScript effects. TypeScript has no comparable runtime walk. | Repeated getters, proxy traps and graph traversal; user effects can invalidate shapes and ICs. | Loses correlation with the checked values and changes effect order. | Reject. |
| Install contracts while trying each membership alternative | A failed check can constrain a still-dynamic object, or a losing union branch can decide its later writes. | Neither Rust target invariance nor TypeScript aliasing justifies publishing failed obligations. | Cheap eager writes but requires rollback and invalidation. | Leaks speculative state into later operations. | Reject. |
| Retain successful admission evidence and publish it at the boundary | The same successful boundary determines both accepted values and acquired contracts; a failed probe adds no contract. | Rust is a limited comparison for separating established obligations from guesses, not a model for these runtime effects. | A compiler may combine check and descriptor installation; a temporary interpreter journal avoids another observable traversal. No user-object fields are needed. | Preserves source effects, selected alternatives, nested identity and independent final liveness checks. | **Choose independently of effort.** |

Check elision must retain required acquisition. Proving current membership alone is insufficient. An engine may emit only a descriptor installation when its proof makes that safe, or omit it when the backing contract is already established. The interpreter uses existing backing metadata to retain no-evaluation elision where acquisition is already established, and retains the boundary otherwise. In particular, returning an already-checked parameter must not read its getter again. The design does not require a repeated membership walk in a production engine.

This does not make in-place conversion transactional. A conversion or nested boundary that actually completes before a later failure retains its ordinary effects. Discarding speculative membership evidence is distinct from undoing getters, setters or successful nested conversions.

Explicit structural conversion uses the same successful-admission evidence. Generic interface conversion resolves its member types with the applied arguments before converting them, consistently with its membership check.

Covered acquisition includes declared members encountered by admission, nested objects, existing enumerable String properties admitted by index signatures, resolved interface members, and selected tuple-rest contributions. Absent optional properties are not materialized. Complete contracts for later-created index-domain properties, optional-property creation/deletion, conflicting overlapping index/intersection contracts, inherited/accessor backing and broader elision proofs remain audit work. These limits do not permit replacing a known backing contract with a current-value guess.

Primary comparisons: [Rust subtyping and variance](https://doc.rust-lang.org/reference/subtyping.html), especially mutable target invariance and the limited scope of Rust subtyping; [TypeScript object types](https://www.typescriptlang.org/docs/handbook/2/objects.html), especially readonly aliasing. This proposal does not adopt Rust exclusivity/lifetimes or TypeScript's erased runtime model.
