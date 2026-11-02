# Invocation adapters and callable identity

This design complements [eval and stable names](eval-name-stability.md) and the shared [origin-evidence rules](origin-evidence-audit.md).

## Problem and recommendation

Treat `call`, `apply` and `bind` as ordinary property names. Select an adapter summary only after establishing the actual original intrinsic callable. Follow its actual receiver, argument values and operation effects. A replacement property is an ordinary call, including when its receiver is an original reflective intrinsic.

The earlier implementation recognized adapter spellings and rewrote their arguments before effect analysis. That loses effects in this two-Script example:

```javascript
// Script 1
const target = Object.defineProperty;
target.bind = function () { globalThis.f = () => 1; };

// Script 2
function f(): string { return "old"; }
target.bind(null);
const n: number = f();
```

The ordinary replacement runs immediately and changes `f`. Mistaking it for binding the original target can manufacture a string-versus-number Early Error before the Script executes. With result evidence correctly withdrawn, the existing number receiving boundary accepts the actual value. If the replacement instead installs a string-returning function, that boundary still rejects the result at runtime.

The reverse case matters too. A captured original adapter remains original after the corresponding property is replaced. Borrowing or renaming it must not turn it into an unknown implementation solely because its spelling changes. These rules apply to effect evidence, not to a new public callable type or an effect annotation.

## Required conditions

1. Mutable property spelling, compatible signatures and native callability cannot certify adapter identity.
2. Callee lookup, argument evaluation, adapter-internal operations and target invocation retain their separate effect positions and abrupt completions.
3. `bind` captures a target, receiver and prefix; capture does not invoke the target. Prototype and metadata operations during binding can still run user code.
4. `apply` consumes its applicable array-like argument list even when no target parameter uses those values. It is not an iterator operation.
5. Unknown alternatives cannot be erased by known alternatives. An incomplete summary withdraws dependent evidence without weakening an independently established receiving contract.
6. No summary executes user code during checking or adds a runtime lookup, wrapper, dereference, public tag or type-identity rule.

## Question 1: what identifies an adapter?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Rewrite every property spelled `call`, `apply` or `bind` | Familiar completion names appear reliable even after replacement; a wrong guess can reject valid code | Rust call traits are language-established implementations, not mutable JS properties | Cheap syntax match with no identity guard | Fails replacement, getter and renamed-original cases | **Reject** |
| Trust a matching callable signature or native implementation | Makes replacements silently inherit privileges absent from their declarations | Rust `Fn` describes calling capability; it is not an absence-of-global-effects proof | Cheap broad classification, unsound shortcuts | A compatible function or proxy can perform different effects | **Reject** |
| Treat all adapters as opaque forever | Safe calls unnecessarily lose useful static diagnostics; ordinary completion remains unchanged | Rust can retain known implementation information; its closure traits do not require this pessimism | Simple fallback but prevents useful specialization | Safe loss of precision, not the best finished behavior | **Reject** as the general design; retain as fallback |
| Freeze adapter properties or reserve their names | Breaks existing replacement and instrumentation patterns; completion would promise a restriction unlike other function properties | Rust trait methods are not mutable object slots, so the analogy does not justify changing JS properties | Could simplify guards by changing observable mutation behavior | Conflicts with ordinary JavaScript behavior and the proposal's compatibility boundary | **Reject** |
| Associate private summaries with original callable identities | Captures and renaming work; replacements have their own behavior; autocomplete needs no special API | Closest to compiler knowledge of an implementation, while retaining ordinary JS lookup | Existing builtin IDs or private weak metadata; normal property/prototype guards, no public shape change | Distinguishes identity from the mutable route used to obtain it | **Recommend** |

The engine registers the three original Function prototype methods during realm bootstrap, extending the existing reflective-origin registry. Ordinary data lookup carries the actual function value. A getter or exotic operation contributes lookup effects; copying a callable data value alone does not invoke it. Known origins from different initialized realms can share a summary category without conflating their function identities.

A production engine can separate a property/prototype dependency from the final callable-identity check. Its existing shapes and inline caches can establish the lookup route; a builtin identity can select the adapter summary. Invalidation must discard dependent evidence when that route changes. Neither step needs an observable property on the function, a new user-object shape transition or a frozen prototype. A failed runtime guard must continue using the already captured callable and operands in the same execution. This is a design cost analysis, not a measured speedup or a claim that a callable signature carries effect information.

## Question 2: how should borrowed and composed adapters forward values?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Normalize only direct member syntax | Equivalent aliases, computed properties and `call.apply` behave differently in analysis | Rust trait dispatch is based on implementation, not the surface name of a local variable | Small implementation but repeated syntax exceptions | Misses composed operations and actual receivers | **Reject** |
| Treat every composed adapter as an immediate opaque invocation | Safe fallback, but harmless binding can lose unrelated diagnostics | Rust closure construction and invocation are distinct operations | Bounded and simple, poor specialization | Conflates capture with target execution unless separately modeled | **Reject** as the finished design |
| Use callable values, receiver flow and argument-list transformations | Borrowing follows ordinary JS; users need no special spelling and wrong replacement results still meet receiving checks | Rust closure environments capture values for later calls; JS also has mutable properties and dynamic `this` | Reuse finite callable/aggregate graphs; production guards cover assumptions used for specialization | Preserves forwarding, captures and target invocation as separate operations | **Recommend** |

For the original `call`, the adapter's receiver is its target; its first argument is the target receiver and the remaining arguments are forwarded in order. For original `apply`, the second argument supplies the array-like list. For original `bind`, the remaining arguments form the captured prefix. Later bound calls prepend that prefix and use the captured receiver. Binding another bound function does not override its earlier bound receiver.

`Function.prototype.call.apply(target, [receiver, a, b])` therefore requires an index shift, not merging all arguments into one slot. The prototype preserves indexed tails where available. Unknown packed bound-prefix arity is currently joined conservatively with later arguments; that is a recorded precision limit, not a language rule permitting lost arguments.

Function values also need property identities. Source callables and newly bound functions can have ordinary own properties, including a replacement `call`, `apply` or `bind`. Source-function adapter lookup follows the applicable function prototype. Newly bound functions inherit the target's actual prototype; substituting the current realm's Function prototype would change later lookup. Unknown prototype behavior retains possible effects.

## Question 3: which operations during binding and applying count?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Assume binding is entirely inert and only inspect used apply parameters | Surprising errors depend on target parameter count and overlook getters | Rust closure construction can evaluate captures, but Rust does not have these JS metadata/list hooks | Cheapest scan, invalid omissions | Misses bind metadata/proxy effects and unused argument-list element getters | **Reject** |
| Invoke getters or inspect the target by running a probe | Gives apparently precise answers at the cost of duplicated effects | Rust compiler reasoning does not justify arbitrary execution of JS getters | Unbounded work, reentrancy and observable extra execution | Changes evaluation count, order and error timing | **Reject** |
| Mark every bind/apply operation effectful | Predictable conservative fallback; loses safe independent-result diagnostics | No close Rust requirement for blanket opacity | Low analysis cost, lost optimization and diagnostics | Sound withdrawal but fails the goal of positive evidence | **Reject** as the general design |
| Model semantic operations using positive operand evidence and an opaque fallback | Getter counts and errors follow existing JavaScript; safe ordinary cases retain precision | Rust capture/call separation is helpful; ECMAScript itself supplies the closer hook-order precedent | Guarded ordinary data access and bounded list summaries; exotic paths keep ordinary slow paths | Includes all required accesses without checking-time execution | **Recommend** |

Binding includes the target prototype operation, its own-length presence/read operations and its name read. It does not invoke the captured target. A throwing metadata getter prevents later binding steps; the runtime evaluator continues to provide that ordering. The effect graph is a conservative summary, not a replacement evaluator.

Applying includes length retrieval and conversion plus every selected indexed read, independently of the target's declared parameters. A known empty list does not read an unrelated getter. Null or undefined means an empty argument list under ordinary ECMAScript semantics; an arbitrary array-like list does not invoke an iterator merely because spread would. Accessor lengths, exotic targets, unknown indices and mutations that invalidate a known length require fallback unless stronger evidence exists.

The prototype uses bounded ordinary-list summaries and an opaque fallback. It accepts direct null/no-list cases; it does not yet prove every equivalent nullish alias. A numerical list bound is an analysis resource limit, not a normative maximum on arguments or a license to choose observable Early Errors. The finished static-evidence model must satisfy the plan's optimization-independence condition regardless of these engineering budgets.

## Order, receiving boundaries and reference liveness

A getter for `target.call` runs before argument evaluation, and throwing there skips the arguments. Argument-side replacement does not retarget the adapter already captured. Internal bind metadata and apply-list accesses happen after argument evaluation and before any applicable target invocation. No operation is replayed on a failed proof or receiving check.

Preserve earlier valid captures while withdrawing evidence at a later affected read. Forwarded reflective operations retain their target, key-conversion, descriptor and setter effects. Defining a property must not be modeled as calling its former setter; review caught and removed that false effect. An identified adapter is not proof that its target is effect-free.

No new shallow callable admission, result contract, nominal prototype obligation or reference permission follows from adapter recognition. A reference in a bound prefix remains subject to the original authority and liveness rules when later used. Binding neither copies its referenced location nor revives it after invalidation. This increment changes static effect evidence; existing runtime conversion and reference enforcement remain in place and in the broader validation gate.

## Remaining work and primary references

The [following deletion increment](prototype-deletion-effects.md) handles possible removal of an own property and the inherited effects of a later dependent lookup, including intermediate prototypes and preserved captures.

The preferred language direction is complete value-based operation reasoning, independent of implementation effort. The current graph is still a finite may-flow approximation. Packed-prefix precision, earlier-source bound-function internals, arbitrary prototype changes, callback consumers, overloaded-index effects and other static-evidence consumers remain work for milestone 5. Milestone 6 still requires integrated conformance and resolution of the recorded baseline failures/build blockers.

[ECMAScript Function prototype methods](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-function.prototype.apply) define adapter behavior. [BoundFunctionCreate](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-boundfunctioncreate) specifies prototype acquisition. [Rust Fn](https://doc.rust-lang.org/std/ops/trait.Fn.html), [FnMut](https://doc.rust-lang.org/std/ops/trait.FnMut.html) and [FnOnce](https://doc.rust-lang.org/std/ops/trait.FnOnce.html) distinguish call capabilities and receiver use. They do not supply a JS property-identity or effect-freedom guarantee; even a Rust shared receiver is not a general prohibition on ambient effects.
