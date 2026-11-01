# Legacy prototype setters and ordinary store effects

This is the sixteenth milestone-5 increment of the [eval and stable names plan](eval-name-stability-plan.md), following [prototype construction](prototype-construction-effects.md). Its [implementation record](legacy-prototype-effects-implementation.md) records the delivered subset and validation. Milestones 5 and 6 remain open.

## Recommended language direction

Analyze a property assignment as the operation it actually performs: resolve the applicable store path, retain its receiver, include proposal-specific storage admission, and follow the selected accessor implementation when available. Apply a legacy prototype-setter summary only to the original built-in identity. Separate a successful ordinary relink from effects of later inherited use. Preserve captures between those operations.

The spelling `__proto__` alone proves none of this. It can name an own data property, a replacement setter or an inherited original setter. The original setter can also be saved, borrowed or installed under another name. This extends private effect evidence, without changing public type equality, reference authority/liveness, runtime prototype obligations or the existing JavaScript operation order.

## Motivating cases

Suppose an earlier Script initialized these objects:

```js
const target = {};
const proto = { run() { globalThis.f = () => 1; } };
```

Then consider:

```js
function f(): string { return "old"; }
target.__proto__ = proto;
const before = f;
target.run();
const n: number = f();
```

The ordinary relink does not run `proto.run`. Its later invocation can replace the Script-global binding, so the final read cannot use the former string result to reject the program. Changing the final initializer to `before()` retains the earlier captured function and its incompatible string result. An immediate ambient effect for every legacy assignment loses that evidence too soon; ignoring the changed chain loses the later effect.

Assignment also differs from an explicit call to a previously captured setter:

```js
target.__proto__ = (replaceTheSetter(), proto);
originalSetter.call(target, (replaceTheSetter(), proto));
```

In the assignment, obtaining the target Reference does not capture an accessor function. Setter selection occurs during the subsequent store. In the explicit call, the callee already obtained remains the original setter. The right-hand side and arguments retain their own effects in both forms. Extending the captured-callee rule indiscriminately to property assignment would be incorrect.

A third distinction is storage identity:

```js
Reflect.set(lookupTarget, "world", globalThis, receiver);
receiver.world.f = () => 1;
```

The data-write route places the capability on `receiver`. For an inherited setter, that same receiver is its `this` value. Merging the lookup target and an explicitly supplied receiver can fabricate a relink of the wrong object; omitting receiver flow can miss the later global write.

## Conditions on a valid solution

| Condition | Required consequence |
| --- | --- |
| Actual accessor identity | A same-named replacement receives no original-setter summary; a renamed original can retain it |
| Actual operation position | Right-hand-side changes participate in assignment's setter selection; captured explicit callees are not retargeted |
| Receiver identity | Setter invocation, data definition and typed admission use the applicable receiver rather than a guessed lookup target |
| Positive operation evidence | Ordinary method identity does not erase typed admission, exotic receiver behavior or unavailable storage representation |
| Prototype dependencies | Own-property hiding, deletion, intermediate prototypes and subsequent inherited consumers participate |
| Conjunctive alternatives | One ordinary receiver or ignored primitive prototype cannot cancel an unknown or valid alternative |
| Existing runtime rules | Counts, abrupt completion, receiving errors, nominal/storage obligations and reference liveness remain intact |
| Bounded analysis | Unknown facts, cycles or budget exhaustion never certify absence of an effect |

The following decisions evaluate the available directions against the same five criteria. Implementation effort does not determine the recommendation. Conservative cases in the delivered interpreter are recorded as incomplete precision, not permission for different engines to select observable error phases through arbitrary budgets.

## Decision 1: recognize a name, an operation or an actual setter?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat every `__proto__` write as a relink | Completion suggests a special name even when it is an ordinary data property; a wrong guess changes diagnostics | Contradicts JavaScript descriptor behavior; Rust has no analogous prototype spelling | Cheap classification, invalid dependencies | Misses replacements and renamed originals; invents relinks | Reject |
| Treat every store or original-setter call as an opaque ambient effect | Simple fallback, but harmless setup destroys useful captures | Neither JavaScript allocation/topology changes nor Rust item identity implies arbitrary callbacks | Minimal analysis, unnecessarily broad invalidation | Conflates relinking with later use | Reject as the complete rule |
| Ban or freeze legacy prototype routes in checked code | Breaks familiar APIs and borrowed helper idioms; completion cannot repair compatibility | Rust's lack of prototypes is not a reason to remove JavaScript behavior | Simplifies guards by imposing new runtime restrictions | Changes the language and does not address ordinary replacement setters | Reject |
| Follow the actual store route and selected setter identity | Ordinary properties stay ordinary; saved helpers retain meaning | Matches JavaScript operations; Rust function-item identity is a useful but limited comparison | Reuses accessor identity, shape and prototype dependencies | Covers named, renamed, replaced and borrowed routes without new public types | **Recommend** |

The original setter's validation order matters: after checking that its receiver is coercible, it ignores a non-object, non-null prototype before attempting relinking. A valid prototype on an exotic receiver can invoke that receiver's internal operation. The setter returns `undefined`; an assignment expression retains its right-hand value. Merely installing a proxy as the prototype is distinct from later traversing it. Ordinary rejection under existing obligations remains possible.

## Decision 2: when is the accessor selected?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Capture an assignment's setter while evaluating its left-hand Reference | Looks analogous to a callee capture until a right-hand-side replacement changes the runtime behavior | Rust has no matching dynamic descriptor lookup; its function-item model does not justify this analogy | May simplify source ordering but gives the wrong guard position | Excludes changes that precede the actual store | Reject |
| Merge every later setter/prototype change into every use | Earlier captures and unrelated expressions lose evidence | Both captured values and mutable access paths require distinction; Rust supplies no direct prototype rule | Easy joins, excessive invalidation and IC dependency breadth | Too imprecise to prescribe observable diagnostic phase | Reject as the final rule |
| Keep References, captured callees and internal store operations distinct | Matches evaluation order; a wrong assumption can be explained by the actual operation | Fits JavaScript's Reference/call semantics; Rust identity supports only the captured-value part | Guards can be placed at the relevant access without replaying operands | Includes right-hand-side effects and preserves earlier explicit calls | **Recommend** |

Global-property may-writes also retain their operation positions. A wildcard write after an earlier `globalThis` capture cannot flow backward and replace that captured value with its own right-hand side. Later reads and deferred executions still include applicable alternatives. A confirmed missing ordinary inherited data property likewise supplies no unknown callable by itself; accessor and unsupported native-read paths retain their effects. These are precision requirements of the same capture/operation distinction, not new global binding or descriptor semantics.

The same ordering applies to descriptor definitions inside operands, not only direct assignments. An inherited setter must receive the original receiver. A setter's own relink cannot retroactively select a different setter for the very operation that invoked it. Repeated/deferred execution needs conservative dependencies where occurrence-specific ordering is unavailable. No checker step may execute a getter, setter, trap or argument to discover which path wins.

## Decision 3: which object receives a reflective store?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Always use the lookup target | Works for shorthand calls, surprises users of the receiver argument | Contradicts reflective JavaScript; Rust has no corresponding target/receiver API | Small graph, wrong mutation identity | Loses receiver stores and can relink the wrong object | Reject |
| Always join target and receiver, even for a known explicit argument | Appears conservative, but wrong-object effects erase valid captures | Mutable access paths should retain identity; Rust's fixed field model is not equivalent | Unnecessary alias sets and invalidations | Creates avoidable false dependencies | Reject as the intended rule |
| Use the actual explicit receiver; retain alternatives only for unknown arity | Completion and API documentation describe the behavior users get | Matches JavaScript; the Rust analogy stops at storage identity | More precise keyed writes and reusable shape guards | Preserves setter `this`, data placement, proxy definitions and capability flow | **Recommend** |

A data path can consult the receiver's own descriptor and define a property there. An accessor path invokes its setter instead. The lookup target's ordinary internal method therefore does not establish that an exotic receiver is harmless. Supported `Reflect.set` calls with known arity use the actual receiver; uncertain packed argument lists remain conservative. Descriptor definition remains a separate operation that bypasses ordinary setter dispatch.

## Decision 4: does an ordinary internal store imply no user code?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Trust ordinary `[[Set]]` identity alone | Typed data looks simple in completion, but admission may run code unexpectedly | The base JavaScript model omits this proposal's storage extensions; Rust cannot fill that semantic gap | Cheap internal-method check | Misses typed properties/elements and supported backing operations | Reject |
| Infer harmlessness from a type name or result signature | Attractive diagnostics until a converter/getter is involved | Rust item identity is stronger than an arbitrary compatible signature | Tempting but invalid optimization | Confuses result/admission contracts with effect proofs | Reject |
| Make every typed or ordinary store permanently opaque | Reliable fallback, loses ordinary local-store evidence | No universal requirement from either language | Broad invalidation can defeat useful specialization | Safe fallback, not the desired complete model | Reject as the final rule |
| Include admission and the actual storage representation in operation-specific evidence | Effects follow the operation; completion does not promise purity | Requires explicit proposal rules; identity/effect separation is the useful Rust lesson | Native storage metadata, shape and type guards can support specialized paths | Retains conversion/backing effects and existing liveness/authority obligations | **Recommend** |

The engine's ordinary Set path performs proposal admission on the receiver before delegating through ordinary descriptors. Typed-property information is retained in internal storage metadata, not necessarily in the raw property descriptor. Inspecting only that descriptor is insufficient. This delivery retains conservative effects for applicable typed properties/elements/tuples, scalar-rest guards and unavailable ordinary storage representation. Explicit typed source properties and typed definitions contribute later store dependencies. More precise conversion and backing summaries remain work under the recommended direction.

Source-created arrays and actual rest allocations also need their initial prototypes. Otherwise an ordinary new array index appears arbitrarily effectful while an inherited array setter can be missed. This delivery supplies those initial-chain facts for the store model. It does not claim that every iterator, native callback or indexed-storage protocol now has a complete summary.

## Decision 5: how should bound invocation interact with the fallback escape screen?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat every bound call as an unmodeled escape even after propagating its target and arguments | Borrowing a harmless setter unexpectedly destroys captures | Rust closures/bound environments are not JS bind, but their existence does not imply purity or arbitrary invocation | Duplicate analysis and broad invalidation | Loses precision already established by invocation flow | Reject as a universal fallback |
| Treat all bound functions as pure or give them the original target's result contract | Completion suggests a guarantee unavailable for arbitrary targets | Rust signatures likewise do not grant an arbitrary implementation's effect summary | Cheap and unsafe | Misses opaque targets, captured receivers and unknown alternatives | Reject |
| Prove that all contributing callee alternatives have modeled bound invocation flow | Direct and borrowed routes remain consistent; unknown alternatives retain their cost | Separates identity, environment and effects without importing Rust's object model | Reuses a finite graph and avoids duplicate escape handling | Suppresses only the redundant fallback; actual target effects and receiving checks remain | **Recommend** |

The positive test is conjunctive over contributing flows. A possible bound candidate cannot cancel an unsupported or unknown alternative. This is private knowledge that invocation flow was modeled; it is neither a public effect type nor a claim of purity, stable result implementation or check-elision authority. Earlier-source bound internals without an available summary remain opaque.

## Production-engine implications and reference constraints

A production store IC already distinguishes receiver shape, own data, inherited accessor and exotic cases. The proposed evidence can use the same underlying distinctions while retaining separate compile-time provenance and runtime guards. Relinking can invalidate prototype dependencies without invoking JavaScript. Typed admission may need its own guarded fast path or a normal slow path. A runtime guard failure cannot justify an irrevocable Early Error and must not replay operands or callbacks.

The interpreter retains allocation-specific may-values and bounded prototype/store dependencies. Its historical joins are not an exact heap simulation. Source descriptor transitions can still contribute both data and accessor possibilities. Known explicit receivers improve mutation precision; unknown keys, branches, packed arity, cycles and deferred contexts lose it. These tradeoffs are not measurements of production-engine speed.

No summary grants a new reference capability, extends a borrow, revives invalid storage or skips existing liveness checks. Prototype, typed-store and representation facts may explain an effect dependency without authorizing an access. Runtime setters, ordinary definitions, admission, nominal obligations and reference algorithms are unchanged in this increment.

## Scope and follow-up

Acceptance includes original/replaced/renamed/cross-realm setters, ordinary and exotic receivers, primitive and unknown prototype alternatives, unused prototype getters, later calls, intermediate captures, right-hand-side changes, inherited source setters, original receivers and exact counts, throwing setters without replay, incompatible receiving results, data-property and null-chain controls, intermediate deletion, distinct allocations, reflective receiver placement, typed admission, array setters and bound-flow uncertainty.

The full positive static-evidence model remains unfinished. Precise descriptor transitions and null-chain replacement, exact per-invocation/deferred contexts, broader packed/bound arity, source retrieval of arbitrary accessor descriptors, native callbacks, overloaded-index operations and complete storage/conversion summaries remain under milestone 5. The global-object token still has a coarser model than ordinary aggregate storage. Milestone 6 retains integrated conformance and the recorded baseline test/build blockers. The next implementation should close a demonstrated missing consumer or positive-proof condition rather than treating a finite withdrawal screen as complete.

Primary references: [the original legacy setter](https://tc39.es/ecma262/#sec-set-object.prototype.__proto__), [OrdinarySetWithOwnDescriptor](https://tc39.es/ecma262/#sec-ordinarysetwithowndescriptor), [assignment evaluation](https://tc39.es/ecma262/#sec-assignment-operators-runtime-semantics-evaluation), [Rust function-item types](https://doc.rust-lang.org/reference/types/function-item.html), and [V8's property representation overview](https://v8.dev/blog/fast-properties). The production strategy is a design inference, not a claim that V8 implements this proposal. Rust supplies an identity analogy, not JavaScript descriptor or reference-liveness semantics.

The next increment covers [descriptor observation and accessor extraction](descriptor-observation-effects.md), including the existing promised type field and its implementation repair. Its remaining limits are recorded separately from this increment.
