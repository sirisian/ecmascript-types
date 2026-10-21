# Foreign class declaration contexts

An imported class keeps the contracts checked where it was declared. A consumer does not need to import the class's interface dependencies, private aliases or helper classes. An unrelated consumer declaration with the same spelling cannot replace one of those dependencies.

```js
// provider
export interface Has<T: type> { get(): T; }

// dep
import { Has } from "provider";
export class E implements Has.<E> { get(): E { return this; } }

// main
import { E } from "dep";
const e: E = new E();
```

This remains valid if provider imports an unused binding from main, forming a module cycle in which provider and dep finish before main's body. A required read of an unavailable interface still fails at its existing boundary. Knowing a declaration's contract does not initialize its binding.

## Requirements and alternatives

A solution must retain source lexical identity and each actual activation, instance and static contracts, constructor signatures, generic arguments, inherited and partial contributions, module availability and failure order. It must neither execute user code to discover a contract nor turn type knowledge into permission to access a dead reference. Autocomplete should expose the class's declared API without requiring imports of its implementation dependencies.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Require consumers to import every dependency used by the class | Hidden dependency lists become part of routine use; adding a local name can change diagnostics and completion | Rust imports preserve an item's identity; consumers do not rebind its implementation's free names | Cheap name lookup, but duplicate checking and larger dependency sets | Reinterprets lexical references and makes same-spelled unrelated declarations interchangeable; grants no liveness | Reject: moves a checker defect into the API |
| Rebuild the foreign declaration in the consumer and repair identities afterward | Simple cases work, but a wrong name can already have changed a member or produced a false error | Neither Rust item lookup nor ECMAScript lexical binding supports this rebinding | Repeats source walks per consumer; final identity completion cannot repair earlier decisions | Cannot undo an incorrectly resolved contract or a queued obligation; reference liveness remains independent | Reject: late identity repair is insufficient |
| Rebuild the declaration in a fully restored producer checker context | Can preserve the intended API if every frame, publication and obligation is restored correctly | Consistent with Rust's lexical item identity, but JavaScript execution contexts add requirements | Valid in principle, with context restoration and repeated checking costs; caches must include the correct context | Requires isolation of diagnostics, inference, captured inputs and publication; must not execute a module early | Valid alternative, not recommended: unnecessary rechecking when the producer already supplies the descriptions |
| Reuse producer-owned checked descriptions and retain the source context for computations and actual-identity completion | Stable member completion and diagnostics through renames and re-exports; wrong calls still fail | Closest to Rust imports retaining existing items and to ECMAScript imports indirectly referencing source bindings | Compiler-owned declaration summaries and existing identity caches; no user-object shape changes or additional property inline-cache state | Preserves constructors, static and instance members, independent identities, module order and reference liveness; unknown descriptions remain unknown | Recommend independently of effort |
| Read export values, construct a sample, or execute a dependency early | Results depend on side effects and cycles; autocomplete cannot predict a stable declaration API | Rust has no direct counterpart to mutable JavaScript class bindings or module execution | Adds runtime reads, effects and invalidation dependencies | Violates the distinction between declaration contracts and current mutable values, and may violate initialization order | Reject |

Choose producer-owned descriptions. Rechecking a consumer must not replace the producer's declaration evidence, inferred returns, constructor signatures or interface obligations with descriptions formed in the consumer. Each later producer checking pass may refine its own descriptions under the existing inference rules. Actual identity completion creates a consumer-local record using the producing environment; it does not mutate the shared symbolic description.

Rust's [use declarations](https://doc.rust-lang.org/reference/items/use-declarations.html) allow aliases for existing paths; this supports identity-preserving imports, not a claim that Rust traits have this proposal's runtime interface semantics. ECMAScript's [Module Environment Records](https://tc39.es/ecma262/2025/multipage/executable-code-and-execution-contexts.html#sec-module-environment-records) provide indirect access to source bindings. Rust has no equivalent of this proposal's runtime class activations, mutable exported constructors or pre-body type evaluation. Those behaviors remain governed by the proposal and ECMAScript.

The implementation shares the producer's instance, constructor and static descriptions by declaration identity. Specialized interface cases likewise publish their own checked object or callable description, with capture parameters preserved; a consumer must not silently fall back to the primary when a case was selected. Consumers complete available actual identities in the resolved producing module. A foreign interface reached through a declaration also uses its producer's checked description. This complements [imported class identity completion](imported-class-contracts.md) and [imported superclass origins](imported-subclass-heritage.md); it does not establish a stable origin for an expression whose producer can reassign it.

## Inferred boundary normalization

Reifying a symbolic inferred class application can revisit its generic constraints, including when the application was returned by a function or extracted from a pattern. That replay uses the actual declaration's captured context. Complete the inferred destination's identity first; do not choose its declaration from the value being admitted. Interface applications retain the same source-context requirement when normalization reaches them.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Recheck constraints using the boundary's caller environment | A valid specialization fails merely because it was assigned to a const or returned | Rust's declaration-bound generic obligations do not resolve against unrelated caller names | Cheap context access but redundant, incorrect work | Changes the destination contract; cannot grant reference liveness | Reject |
| Remove all normalization and accept the symbolic arguments | Removes this error but can disagree with explicit annotations | Rust has no direct counterpart of the interpreter's compact numeric arguments | Avoids work, but only safe when canonical arguments are already established | Raw numeric descriptions can still require conversion under their declared value domain | Reject as a general rule; reusing already established canonical arguments is a valid optimization |
| Infer the destination's declaration from the admitted value | Usually accepts the apparent result, even when its identity is wrong | Conflicts with Rust's distinction between a generic obligation and the value offered for it | Adds value inspection and shape dependencies | Lets a value select its own admission target and can conflate activations | Reject |
| Complete the written/inferred destination through its source bindings, then normalize in that actual declaration's captured context | Assignment, extraction and return preserve the same advertised specialization; wrong arguments still fail | Preserves Rust-style declaration identity, with additional runtime activation handling | Existing identity completion and compiler-owned captured context; no shape or inline-cache changes | Keeps the target independent of the value, source constraints intact, caller context restored, and liveness independent | Recommend independently of effort |

The interpreter can recover a foreign module context through linked import metadata when a symbolic inferred result names a private source declaration. The search matches the original module syntax identity, visits each environment at most once, and rejects conflicting actual environments. It reads no export values. Local declaration activations still require their own registered family or published binding; finding their containing module is not enough to choose an activation. Engines may retain the equivalent source-context association directly in their checked metadata.
