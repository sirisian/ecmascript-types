# Module reference bindings

A module-level `let ref` or `const ref` reads through its location just like a reference binding in a block or function. Exporting the binding preserves that behaviour. Named imports, re-exports and module namespace reads observe the current referent, including changes to the referent and permitted rebinding by the declaring module.

```js
// producer
let first = 1, second = 2;
export let ref current = first;
export function redirect() { ref current = second; }

// consumer
import { current, redirect } from "producer";
const before = current; // 1
redirect();
const after = current;  // 2
```

Each read performs the ordinary reference access, with its accessor/proxy effects, abrupt completion and liveness checks. Exporting does not copy a value, pin storage or revive an invalidated borrow. A binding's uninitialized state and the referent's uninitialized state remain independently observable. Preparing source contracts does not initialize either.

An ordinary value-consuming expression still copies out a value as otherwise specified. Thus `export { current as default }` exposes the live binding, whereas `export default current` evaluates the expression once and exports that result. Export visibility confers no new write permission: an import binding remains immutable, a namespace property cannot be assigned, and an imported reference binding cannot be rebound by its importer. Borrowing an import does not remove those restrictions. A declaring module may expose a function that performs its own permitted write or rebinding.

## Decision

The required invariants are scope-independent reference reads; preservation of import indirection and temporal dead zones; exactly the ordinary referent effects and failures; unchanged write/rebinding authority; and no new allocation or object-layout requirement. All directions below were evaluated independently of implementation effort.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Replace a missing stored value with `undefined`, or guard downstream operators | Autocomplete still advertises the referent type while reads yield an unrelated value; failures move away from the declaration | Neither ECMAScript live imports nor Rust dereference supports this substitution | A cheap guard leaves effects and invalidation unimplemented | Loses the referent, liveness and accessor effects | Rejected: conceals the defect |
| Snapshot on declaration or export | Familiar value copying, but surprising for explicit `ref`; users lose updates and rebinding | Ordinary ECMAScript expression exports copy results; Rust distinguishes copying a value from dereferencing a borrow | Simple stored-value loads, but obtaining a snapshot can invoke user code | Contradicts reference aliasing and can conceal invalidation | Rejected: changes the established meaning of `ref` |
| Prohibit module references or their export | A declaration unexpectedly stops working when moved across scopes; completion would need an otherwise unjustified scope restriction | Rust references can cross module API boundaries subject to its own lifetime rules; ECMAScript module bindings already support live reads | Avoids this read path but forces wrapper functions and their calls | Could define a new restriction, but conflicts with the proposal's general reference-binding rule | Rejected: no semantic benefit justifies the restriction |
| Duplicate reference read-through in the module operation | Correct visible behaviour; the same type and completion expectations as local references | Consistent with live imports and the limited Rust dereference analogy | Can compile to the same checks and loads; no required shape or inline-cache change | Correct only while all liveness, effects and dependency rules stay synchronized | Rejected in favour of shared semantics: duplicate rules invite divergence |
| Resolve imports and preserve module initialization checks, then use the ordinary direct-binding read | Scope-independent behaviour; completion continues to describe the referent, and failed guesses produce the existing boundary errors | Extends ECMAScript's live binding resolution with the proposal's reference semantics; Rust likewise distinguishes a referent access from a copied value | Engines can specialize a known binding kind and inline the common operation; no new per-instance properties, shapes, inline-cache keys or export wrappers | Preserves current referents, alias chains, effects, storage liveness, temporal dead zones and existing authority | Recommended: one semantic read-through rule with module-specific lookup preserved |

The Rust analogy is deliberately limited. Rust's ownership, exclusive mutable borrowing and compile-time lifetime rules are not imported by this decision; in particular, this proposal's retained local locations do not imply that Rust allows returning a reference to a destroyed local. Its relevance is the distinction between accessing a referent and copying its value. ECMAScript's existing module operation is the closer model for live import indirection.

Sources: [ECMAScript Module Environment Records and GetBindingValue](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-module-environment-records-getbindingvalue-n-s), [Rust reference and pointer types](https://doc.rust-lang.org/reference/types/pointer.html).

## Verification boundaries

The conformance cases cover local, returned, destructured and imported references; alias rebinding; named, renamed, re-exported and namespace reads; default export distinctions; getters and proxies; temporal dead zones; typed-array invalidation; SoA field access; and unchanged import/namespace write restrictions. Repeated evaluation after a failed module checks that failure caching does not rerun its body, and that explicit later rebinding can establish a new live location.

This decision does not settle the separate copying of an already stale SoA view or expand inference/narrowing rules. Those boundaries require their own analysis; they are not accepted merely because this module read path is repaired.
