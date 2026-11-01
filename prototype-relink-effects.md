# Prototype relinking and the time of inherited effects

This is the fourteenth milestone-5 increment of the [eval and stable names plan](eval-name-stability-plan.md), following [property deletion](prototype-deletion-effects.md). The [implementation record](prototype-relink-effects-implementation.md) records the delivered subset, exact bases and validation. Milestones 5 and 6 remain open.

## Recommendation and motivating case

Separate a change to an ordinary object's prototype from the user code that a later operation can reach through that prototype. Apply a relinking summary only to the actual original helper and positively established ordinary targets. Preserve effects at relinking for proxy or unknown targets. Track possible new prototype routes at dependent later operations and preserve values captured before those operations.

```javascript
// Script 1
const target = Object.defineProperty;
const proto = { call() { globalThis.f = () => 1; } };

// Script 2
function f(): string { return "old"; }
Object.setPrototypeOf(target, proto);
const before = f;
target.call(null, {}, "x", {value: 0});
const n: number = f();           // Check the replacement's actual result.
const s: string = before();      // This capture still denotes the old function.
```

An opaque effect attached to every relinking call loses the evidence for `before` too early. Merely declaring the original helper harmless loses the inherited replacement too late. Changing the last receiving type to `number` must retain the established string-versus-number Early Error; removing `before` and retaining the later call through `f` must allow the replacement and its number receiving check. An incompatible replacement result still fails at that receiving boundary.

This is a correction to evidence dependencies, not a new public effect type. The ordinary members shown by autocomplete do not become immutable implementation promises. A callable signature describes admissible calls and results; it does not prove that a property still holds a particular intrinsic or that its object has ordinary internal methods.

## Conditions on any valid solution

1. Establish the actual helper identity. A saved original retains its summary; a same-named replacement, accessor result or proxy wrapper does not acquire it.
2. Establish the target's relevant internal operation for every contributing alternative. A source function with a decorator can be replaced; an ordinary candidate cannot cancel an unknown branch.
3. Keep target/prototype argument evaluation, target `[[SetPrototypeOf]]`, later inherited lookup and later invocation at their respective positions. Do not execute any of them during checking.
4. Retain allocation identity, own-property hiding, possible deletion, intermediate prototype dependencies and the original lookup receiver.
5. Preserve captures, including bound-function prototype capture after bind arguments. Later mutation of the target does not retroactively relink an existing bound function.
6. Account for every applicable consuming protocol, not only calls to properties named `call`, `apply` or `bind`.
7. Preserve completion order, receiving checks, nominal/inherited storage obligations and reference authority/liveness. Unknown facts and analysis exhaustion cannot certify effect-freedom.

## Question 1: should relinking itself invalidate ambient evidence?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Every relinking call is an immediate ambient effect | Harmless setup loses diagnostics, including for a capture made before any inherited access; completion offers no explanation | Rust mutation does not erase unrelated captured function values; its object model has no JS prototype counterpart | Cheap broad invalidation, but discards unrelated specialization | Misplaces possible inherited effects before intervening captures | **Reject** as the finished rule |
| Every original helper call is effect-free | Looks simple until a target proxy runs a trap; a wrong guess can reject valid code | Rust function-item identity identifies a function, not the effect of arbitrary operands | Cheap identity test but missing operand guards | Fails proxy targets and later changed-chain effects | **Reject** |
| Forbid or freeze prototype changes around typed code | Makes familiar JS APIs unexpectedly fail; autocomplete cannot express the hidden restriction | Rust's lack of mutable JS prototypes is not compatibility evidence | Simplifies some guards by changing the language | Violates the intended ordinary-JavaScript boundary and existing mutation machinery | **Reject** |
| Original helper identity plus positive target facts; separate relinking and later use | Ordinary setup preserves diagnostics; proxy effects and inherited calls retain ordinary timing | Rust's distinction between a captured value and mutable storage is useful, with no direct prototype analogy | Reuses builtin identity, internal-method and prototype-dependency guards | Satisfies operation order and capture requirements | **Recommend** |

The original `Object.setPrototypeOf` returns its target on success; `Reflect.setPrototypeOf` returns a Boolean. Their target requirements and failure behavior differ. Neither summary means that a change succeeds. Ordinary cycle, extensibility and proposal-specific obligations still decide at runtime. A proxy supplied merely as the new prototype does not imply that its traps run while relinking an ordinary target: ordinary cycle traversal stops when it reaches a non-ordinary prototype operation rather than invoking that operation.

## Question 2: how should the analysis represent a changed prototype?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Replace a single global prototype snapshot | Later source text can change the apparent meaning of earlier captures | Rust captured function values do not follow later changes to their old storage | Small state, but incorrect deoptimization dependencies | Loses temporal and allocation distinctions | **Reject** |
| Merge every possible prototype into every use without positions | Independent objects and earlier captures interfere | Rust alias analyses distinguish storage and use sites; they do not supply exact JS semantics | Finite but excessive IC/proof invalidation | Safe only with needless loss of observable diagnostic precision | **Reject** as the intended design |
| Execute a complete heap simulation during checking | Precision fails unpredictably on arbitrary getters, proxies and loops | Rust static checking is not permission to execute arbitrary JS | Unbounded and potentially reentrant | Changes observable behavior; general exact execution cannot be decided statically | **Reject** |
| Allocation-specific possible links, contributing positions and dependent lookup paths | Earlier captures retain meaning; own members still hide inherited ones | Closest to dependency reasoning; Rust has no equivalent mutable prototype chain | Existing shapes, prototype guards and shared dependency metadata are applicable | Retains possible old/new routes without asserting that a change certainly succeeds | **Recommend** |

For an own property positively known to remain present, relinking cannot expose a property of the same name. A possible deletion can remove that hiding; a non-configurable own descriptor excludes that deletion route. An intermediate prototype can itself be relinked. A bound function copies the applicable target prototype when it is created, rather than keeping a live indirection to the target's future prototype. The model must also retain changes to the captured prototype object itself.

Operation order is finer than source-start order. `target.call(argThatRelinksTarget())` first captures the adapter, so that relinking cannot retarget this invocation. In contrast, the original bind's internal prototype operation follows its argument evaluation. A store follows its right-hand side, including a deletion there that exposes a setter, and descriptor conversion follows the helper's arguments. Changes inside those operands can therefore affect the subsequent internal operation. Deferred functions and loops remain conservative when precise occurrence order is unavailable.

The delivered inclusion graph keeps historical candidates. Exact branch, invocation and heap-state contexts can improve it, but they do not change the recommended semantic dependencies. Finite work limits are an implementation fallback, not permission to let arbitrary engine budgets normatively select Early Error versus runtime rejection. Completing prescribed positive static evidence remains part of milestone 5.

## Question 3: which consuming operations must participate?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Only adapter property calls | Fixes the visible example while surprising users with setters or coercion | Rust traits/operators are explicit language protocols; limiting analysis to one member spelling has no analogue | Small implementation, incomplete dependency set | Misses stores, iteration, descriptor fields and other inherited protocols | **Reject** |
| Every later use of the object is an ambient effect | Even `typeof`, truthiness and strict identity lose evidence | Rust likewise distinguishes inspecting identity from invoking a protocol | Cheap and overly broad; prevents independent guards | Conflates non-observing operations with prototype access | **Reject** as the general rule |
| Operation-specific summaries, with a conservative effect at an affected consumer when its protocol is unavailable | Effects follow actual JS operations; a wrong assumption retains the receiving check | Rust's protocol-specific reasoning is useful, but JS getters, proxies and conversion hooks are more dynamic | Keyed read/write and protocol dependencies can feed IC guards; complex paths can use a slow path | Covers relevant consumers without inventing earlier user execution | **Recommend** |

Relevant consumers include inherited reads and calls, stores that can reach setters, array-like indexed reads for `apply`, descriptor-field conversion, iteration, prototype-sensitive enumeration and membership, coercion, `instanceof`, construction and thenable-related operations. These need not all have equally precise summaries: an unavailable protocol retains an effect at its consumption. `typeof`, truthiness and strict equality do not acquire a conversion hook merely because a prototype changed. No summary authorizes duplicate getter acquisition, replay after a throw, or resuming steps that abrupt completion skipped.

## Question 4: what establishes an ordinary target?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Trust a compatible type or source function body | A decorated or imported function appears ordinary but can be a proxy | Rust function-item identity is compiler-established; JS callable type compatibility is weaker | Inexpensive, invalid guard | Confuses a contract or possible source body with actual internal methods | **Reject** |
| Accept any ordinary alternative in the flow | A conditional's unknown branch silently disappears; the wrong guess can produce a false Early Error | Rust joins must account for every possible value, not just the convenient alternative | Cheap may-flow membership test | Turns a possibility into a universal proof | **Reject** |
| Make all targets opaque | Reliable fallback, loses the purpose of separating harmless setup | Rust retains positive facts where established; universal opacity is unnecessary | Minimal analysis and poor proof reuse | Conservative but unnecessarily loses required precision | **Reject** as the finished direction |
| Conjoin positive operation evidence across all contributing expressions | Ordinary local/native targets retain evidence; unknown and decorated replacements retain effects | Rust's positive identity discipline is the useful analogy; JS needs additional dynamic-object facts | Internal metadata checks and bounded flow traversal; no user probe | Keeps unknown alternatives, decorator replacement and exotic operations visible | **Recommend** |

A known primitive target cannot invoke a relinking trap, although the helper may reject it. A source-created ordinary object or undecorated function can supply appropriate internal-method evidence. For initialized external objects, inspect only engine-owned identity and internal-operation facts without invoking getters or traps. A native callable marker does not make the call pure; here the question is solely whether changing that object's prototype can invoke user code. Own-index evidence for a source array also requires a fixed position: after a spread with unknown length, an element's syntactic position does not prove an own property at that numeric index. Its value contributes to the wildcard flow. An unknown return, unsupported expression, unproven target or exhausted proof retains an effect.

## Production-engine and proposal constraints

V8's [property representation overview](https://v8.dev/blog/fast-properties) illustrates why shape and prototype dependencies matter in a real engine. A successful relink can invalidate property and prototype-chain inline caches even when it invokes no JavaScript. That engine maintenance cost is distinct from a semantic ambient mutation of a global function. Keep analysis metadata private; avoid public proof fields or freezing user objects. Guard failures must continue with already evaluated operands, without replaying getters, arguments or relinking. This interpreter supplies conformance evidence, not a production speedup measurement.

The engine's ordinary relinking path still runs nominal-prototype and inherited-storage planning before committing a change. Those obligations can reject an otherwise ordinary operation. The analysis summary neither bypasses them nor asserts that a failed change occurred: possible links are conservative candidates. It grants no reference authority, does not revive an invalidated location, and changes no public type equality or receiving boundary. The regression gate retains nominal/prototype and reference-liveness coverage.

## Scope and follow-up

The delivery covers original Object/Reflect helpers and supported alias, return, borrowed, applied and bound routes; dependent reads, captures and conservative consumer effects; proxy targets; decorators; and unknown contributing expressions. It does not provide a full positive model for arbitrary `__proto__` assignment, `Object.create`, every descriptor transition, native callback consumer, overloaded index operation or merged invocation context. Earlier-source bound internals, packed bound-prefix precision and the remaining static-consumer/optimization audit stay open. Milestone 6 still requires integrated conformance and resolution of the recorded baseline test/build blockers.

Primary references: [OrdinarySetPrototypeOf](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinarysetprototypeof), [Object.setPrototypeOf](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-object.setprototypeof), [Rust function-item types](https://doc.rust-lang.org/reference/types/function-item.html), and the V8 reference above. The Rust analogy supports separating established value identity from mutable access paths; it does not import Rust's object model into JavaScript.
