# Global mutations and callable result evidence

October 31, 2026. Fifth implementation increment of the [eval and stable names plan](eval-name-stability-plan.md), continuing milestone 5. This resolves the concrete alias-mediated false Early Error left by the [runtime enforcement audit](runtime-enforcement-audit.md). The [implementation record](global-call-origins-implementation.md) gives patch bases, test results and remaining work. This is a bounded extension of the static contribution screen, not completion of the entire consumer audit.

## The problem and recommendation

```js
function f(): string { return "old"; }
const world = globalThis;
world.f = () => 1;
const n: number = f();
```

The last line must not be rejected using the former declaration's string result. The function actually read is the replacement. Retaining a runtime receiving check fixes an unchecked admission but cannot repair an irrevocable Early Error: the body never executes in that case.

**Recommendation:** withdraw a declaration-result assumption when a binding-aware global mutation footprint affects the read supplying it. Preserve contracts established independently of that assumption. Follow immutable callable copies back to their initializer reads, so a copy after replacement cannot recover the old declaration result and a copy before replacement can retain the captured implementation's evidence. Keep this judgment separate from the stronger proof required to elide runtime admission.

The user-facing distinction is between a declared API contract and evidence about the value currently read. An editor can still show the declaration and signature, but should not present a withdrawn result contribution as an unconditional guarantee. A wrong guess crosses an existing typed receiving boundary where one exists. An untyped discarded call does not acquire a new result check merely because the old declaration had a result annotation.

## Requirements for a valid solution

1. Recognize replacement through aliases, reflection and indirect dynamic source; direct identifier assignments are not the only writes.
2. Identify the affected storage. A global property, a global lexical binding, a module local and a function local are different places even when their names match.
3. Preserve evaluation order: capture a callee before its arguments, and capture a const initializer before later replacement of its source binding.
4. Do not erase an independently established inferred, declared or published contract, or its generic activation dependencies, merely because exact implementation identity is unavailable.
5. Do not execute an initializer, getter, callback, decorator or dynamic source while checking. Do not require parsing eval strings as an effect oracle.
6. Preserve receiving checks, public type equality, prototype obligations, reference authority and reference liveness. No wrapper, object freezing, extra lookup or replay follows from this analysis.
7. Bound the required analysis and state its precision. Runtime optimization may improve separately; an identity guard cannot justify mandatory static rejection.

## Decision 1: how much evidence should a global mutation withdraw?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep only a spelling/direct-property screen | Precise-looking completions reject a working alias example | Rust lexical mutation alone is not an analysis of a JavaScript global object | Cheap inventory, but incomplete facts cannot support negative diagnostics | Leaves the reproduced false Early Error | Reject |
| Make every Script-global callable result unknown | Ordinary factory and overload results lose useful contracts even without a relevant mutation | Rust function pointers distinguish a callable type from knowledge of a particular function; unknown identity does not erase the pointer's type | Simple implementation, lost specialization and more admission work | The previous experiment broke construction sequencing, activation identity, result anchoring and reference diagnostics by deleting independent contracts | Reject as a blanket rule |
| Keep all static results and repair at runtime | Rich autocomplete, but the rejected program cannot reach the repair | Neither Rust compile-time checking nor JavaScript runtime execution can undo a prior rejection | Runtime guards are viable only for executions that occur | Does not solve this problem | Reject |
| Require exact interpretation of all calls, heap aliases and dynamic strings | Attractive apparent precision, unpredictable checking cost and failures | Neither Rust's callable model nor ECMAScript source evaluation supplies this oracle | Unbounded interprocedural/heap reasoning cannot be a mandatory portable analysis | Executing user code during checking changes phases and effects; string inspection alone misses computed source | Reject as a required language rule; optional optimizer precision must preserve semantics |
| Use a finite global-effect footprint at the contributing read; retain independent contracts | Removes the stale promise while retaining useful signatures and unaffected inference | ECMAScript supplies storage and capture semantics; Rust supplies only the contract-versus-function-identity comparison | Compiler-side summaries, no instance fields or wrappers; targeted names avoid whole-global invalidation | Meets the requirements within the specified footprint domain; does not authorize elision | **Recommend**, independently of implementation effort |

This is not a claim that every global function is immutable in the absence of a recognized footprint. The receiving-proof repair already refuses that inference. Converting every static consumer to an explicit positive result-evidence model remains work; a finite withdrawal screen is an incremental correction, not a universal absence-of-effects theorem.

## Decision 2: what storage can an effect reach?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Invalidate every same-spelled binding | Editing a global property unexpectedly changes diagnostics for a module local | Rust and ECMAScript both resolve lexical bindings; spelling is not place identity | Cheap string sets, unnecessary loss of optimization | Conflates distinct storage | Reject |
| Treat every indirect effect as a global property write | Global lexical functions appear protected from indirect eval when they are not | ECMAScript's global environment has object and declarative components; Rust has no equivalent eval rule | Cheap but incomplete dependency model | Misses reachable mutable global lexicals | Reject |
| Separate property footprints from dynamic-source footprints, then filter by binding identity | Named writes affect the relevant global API; lexical and module names remain understandable | ECMAScript is the direct model; Rust's distinct places are a limited analogy | Named sets can be indexed; unknown keys widen only the relevant domain | Preserves the property/lexical distinction and existing local/direct-eval rules | **Recommend** |

The supported categories are:

| Effect | Footprint | Does not by itself replace |
| --- | --- | --- |
| Global alias member assignment, update, delete, pattern or iteration target | Named global property; unknown key means all global-object properties | A same-named lexical or module binding |
| `Reflect.set`, `Reflect.defineProperty`, `Reflect.deleteProperty`, `Object.defineProperty` on a tracked global alias | Literal or unannotated const-string key; otherwise all properties | Global lexical bindings merely because the key is unknown |
| `Object.assign` / `Object.defineProperties` on that alias | Literal source property names; opaque source means all properties | Unrelated lexical bindings |
| Indirect/aliased/optional/bound eval, or invocation/construction of a generated function | Global-object bindings and mutable global lexical bindings | Inaccessible function or module locals solely because their names match |
| Opaque invocation receiving a tracked global object, including a receiver or bound argument | All global-object properties exposed through that argument | A separately held lexical binding solely because of object exposure |

Possible direct eval retains its separate lexical-lookup and sloppy-injection rules. A global `const` cannot be rebound by indirect source; this does not make its referent immutable. Invoked code can have additional effects through its own closures, properties and callbacks. The last table row describes the contribution of the tracked exposure, not an assertion that it exhausts every effect of arbitrary code.

## Decision 3: when does the footprint matter?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Withdraw every matching result throughout the source | A later write destroys evidence for an already captured const | Both ECMAScript values and Rust copied function pointers are distinct from later writes to the source slot | Simple scan but loses useful static information | Too coarse for the accepted capture model | Reject as the universal rule |
| Compare text offsets everywhere | Easy to explain for straight-line code, misleading in loops and deferred bodies | Neither language equates textual order with invocation order | Cheap, but back edges and closures defeat it | Can preserve stale evidence on later execution | Reject |
| Fully solve call scheduling and control flow | Potentially best precision but no bounded contract for analysis | Rust's explicit callable contracts avoid requiring this JavaScript effect oracle | Excessive mandatory analysis; useful only as separately justified optimization | Cannot be the finite language requirement | Reject as mandatory |
| Order simple top-level reads/effects, preserve immutable capture, conservatively widen in loops/deferred regions | Early snapshots retain their result; late copies do not. Some unused closure writes reduce precision | ECMAScript callee/initializer capture is the direct rule; Rust's copy analogy does not imply borrow or lifetime guarantees here | Reusable compiler location/effect summaries, with conservative cyclic regions | Covers repeated/deferred execution without replay or false absence proof | **Recommend** for this increment |

```js
function f(): string { return "old"; }
const before = f;
const world = globalThis;
world.f = () => 1;
const after = f;
const n: number = after();   // replacement result admitted at this boundary
const s: string = before();  // retained old implementation
```

`const bad: number = before()` retains its string-versus-number Early Error. A write in an argument to `f(...)` does not replace the callee already obtained for that invocation. The engine compares effects against the contributing **read**, not the end of the whole call.

Top-level sequential effects use source locations. If either the read or effect is inside a callable, instance field or loop, this increment considers the matching footprint across the source. This deliberately includes an unused closure's write. Branches and exceptions are not fully path-sensitive. Alias origins are may-origins collected from initializers and assignments: rebinding a possible global alias to an ordinary object does not erase its possible-global status. These precision limits are recorded, not silently presented as exact execution analysis.

Constructing `Function("...")` does not execute its body. Likewise, `Reflect.set.bind(...)` captures arguments; invocation performs the write. Binding a tracked global argument produces a possible all-property exposure at later invocation, without claiming an exact key. Ordinary argument evaluation still has its own effects.

## Decision 4: how should evidence cross source boundaries?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Trust the spelling of an external alias or import | Works for familiar builtin names, fails for prior-source aliases and shadows | Rust imports are not JavaScript live values; ECMAScript lookup decides the binding | Cheap but conflates declaration and current value | Misses replacements or invents builtin identity | Reject |
| Evaluate getters, imports or initializers to discover targets | Checking can run code or fail before the intended evaluation phase | Neither cited model supports speculative arbitrary JavaScript execution as static checking | Executes unbounded user work and disrupts IC/effect assumptions | Breaks ordering and TDZ behavior | Reject |
| Forget all external evidence | Conservative in isolation but misses a safe known global alias unless all calls invalidate everything | Rust's typed module boundary is a different contract; ECMAScript has live environment state | Avoids inspection but sacrifices useful identity information | Insufficient as the chosen precision rule | Reject |
| Use available initialized engine-owned binding/data evidence without invoking user operations; treat unresolved imports conservatively | Recognizes aliases from prior Scripts while preserving local shadows | ECMAScript environments are the direct basis; Rust is only a reminder to separate the imported contract from implementation identity | Private compiler/environment metadata, no user-visible shape or wrapper | A may-effect observation withdraws evidence; it does not prove future stability or bypass TDZ/ref rules | **Recommend** |

The engine follows initialized declarative/import bindings where its existing safe metadata reader permits it. For a realm-global object binding it can inspect an ordinary own data descriptor without invoking an accessor or exotic operation. A recognized exact global-object or builtin value seeds the may-origin analysis. A source import without such evidence conservatively may expose the global object or dynamic-source operations. No producer implementation identity is inferred from import syntax alone.

## Implemented domain and remaining work

The finite source inventory follows simple declarations and assignments, alias chains, parentheses, comma results, conditional alternatives, supported object binding projections and simple array-literal destructuring. It recognizes global-object receivers, reflection object/operation aliases, optional chains, `.call` / literal-array `.apply`, `.bind`, and generated function invocation. A cycle/work bound widens may-origins rather than turning exhausted analysis into an absence-of-effects proof.

At this increment's baseline the helper did not track object/array storage and reloads, returned aliases, callback forwarding, destructuring assignment into alias variables or nonliteral `.apply` argument containers. The subsequent [stored and returned values increment](global-effect-escapes.md) repairs supported instances of those routes with a finite allocation/property-sensitive value-flow graph. It remains attached to the existing callable contribution predicate rather than replacing every use of the older global-property inventory. Arbitrary accessors, hidden collection storage, all intrinsic mutation routes and all cross-source generated callable provenance still require the broader static-consumer/effect-model work. The subsequent record gives precise remaining counterexamples; the stronger runtime receiving proof remains necessary and separate.

The next acceptance gate is explicit: represent unavailable results independently in every affected static consumer, cover these remaining escape routes, and rerun the factory, generic activation, publication and reference-location regressions. Do not solve them by reinstating blanket contract erasure. The conservative deferred-region rule can later gain prescribed precision only with tests distinguishing early errors from runtime admission; interpreter timing alone is not language-design evidence. Milestone 5 and the integrated milestone 6 gate remain open.

## Production-engine implications and prior-art limits

These are private compiler facts. A production implementation can index named effect summaries by binding and reuse lexical/control-flow infrastructure; it need not linearly rescan every effect for every read. The interpreter's bounded recursive inventory is a correctness prototype, not a production performance claim. More unknown results can retain receiving checks and reduce specialization; no field is added to application objects, so this design itself introduces no shape transition. The actual program's global mutations still have their ordinary effects on ICs and dependencies.

[V8's Maps and constant-property dependency discussion](https://v8.dev/docs/hidden-classes) shows why property stability and invalidation matter. Reusing related private guards for runtime elision is an engineering inference, not a benchmark or a claim that V8 implements this proposal. Guards must use captured operands and preserve reference liveness on failure.

[Rust function pointer types](https://doc.rust-lang.org/reference/types/function-pointer.html) are the useful comparison: callable type and exact function identity are different facts. Rust's typed admission is stronger than this proposal's shallow JavaScript admission, and supplies no equivalent of indirect eval, live global-object aliases or these mandatory Early Error rules. No Rust ownership or lifetime guarantee is imported by analogy.

The direct semantic references are ECMAScript [Global Environment Records](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-global-environment-records), [PerformEval](https://tc39.es/ecma262/multipage/global-object.html#sec-performeval), [CreateDynamicFunction](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-createdynamicfunction) and [EvaluateCall](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-evaluatecall). They establish the storage, environment and operand order; the static contribution policy above is this proposal's design choice.
