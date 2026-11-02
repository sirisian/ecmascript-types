# Descriptor conversion and later accessor effects

This design complements [eval and stable names](eval-name-stability.md) and the shared [origin-evidence rules](origin-evidence-audit.md).

## Problem and recommendation

```js
// First Script
const box = { x: 0 };
const getter = () => { globalThis.f = () => 1; return 0; };

// Second Script
function f(): string { return "old"; }
Object.defineProperty(box, "x", { get: getter });
box.x;
const n: number = f();
```

The pulled implementation rejects the second Script before it runs. Its effect graph follows descriptor `value` but misses the accessor installed over the formerly ordinary data property. At execution, the read invokes the getter, replaces `f`, and the receiving boundary admits `1`.

There are two separate operations to summarize: converting the descriptor object can run code now; installing its `get` or `set` function permits an invocation at a later property operation. Installation alone does not call the function. A descriptor with an ordinary `value` can still have an effectful `enumerable`, `configurable`, or `writable` getter. Conversely, an absent optional descriptor field is not an unconditional property read.

**Recommend property- and operation-sensitive accessor summaries.** Retain possible getter and setter flows with their installation sites. Apply them to the relevant later read or write, with the correct receiver and argument. Account for all descriptor fields during conversion. Keep known copy exclusions, enumerable-property restrictions, earlier captures, and independently established contracts. No new effect annotation or public type is needed.

## Required conditions

| Requirement | Consequence |
| --- | --- |
| Descriptor conversion and accessor invocation are distinct | Observing `descriptor.get` can invoke a descriptor getter; merely installing the returned function does not invoke it |
| Read and write protocols differ | A plain assignment target does not read its getter; compound/update operations can read and write |
| Names, aliases and semantic positions matter | Installing `x` must not automatically affect a read of `y` or an earlier capture |
| Descriptor fields are optional and inherited | Prove ordinary absence where available; otherwise retain possible lookup effects |
| Copies contain property values | Rest/spread/assign must not copy accessor functions as if they were the property values |
| Existing authority and liveness rules survive | Accessor analysis cannot manufacture a live reference or bypass nominal prototype obligations |
| Checking does not execute the protocol | No getter, setter, proxy trap or initializer is run speculatively; actual execution remains single-pass |

These conditions are constraints, not a weighted score that trades correctness for implementation cost.

## Question 1: where should descriptor effects be attached?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Track only descriptor `value` | Small model, but ordinary accessor programs get false Early Errors | Rust direct struct fields do not justify ignoring JavaScript accessor descriptors | Cheap; fails to model real calls | Misses metadata getters and installed accessors | **Reject** |
| Treat every descriptor installation as an arbitrary call of its accessors | Accepts counterexamples but loses useful errors even when nothing reads the property | Rust field access and explicit method calls remain distinct; ECMAScript defines installation separately | Cheap global invalidation; discards reusable proofs | Conservative for some paths, but fails installation-only and key-specific precision controls | **Reject** as the language direction |
| Execute descriptor/accessor code during checking | Appears maximally precise until getters throw, mutate state or depend on later state | Neither Rust compilation nor ECMAScript descriptor conversion licenses speculative ordinary user-code execution here | Arbitrary work, replay and observable effects | Violates phase, order and evaluation-count requirements | **Reject** |
| Separate conversion effects from later keyed getter/setter invocation | Matches what the author wrote; autocomplete retains explicit contracts while uncertain global results retain runtime checks | ECMAScript supplies the direct operations. Rust distinguishes field places from calls, but autoderef may itself invoke user implementations | Private graph/IR dependencies; no accessor wrapper or added property on user objects | Meets the operation, alias and capture requirements without executing code | **Recommend** |

The selected direction does not infer a stable property implementation from its apparent type or shape. Shape/inline-cache dependencies can accelerate execution in a production engine, but a guard that runs later cannot justify an irrevocable Early Error now. The prototype adds private summaries, not a new hidden-class layout or runtime invalidation protocol.

## Question 2: how should absence, copying and property changes be modeled?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat all missing fields as harmless | Convenient until inherited accessors, proxies or unknown descriptors are used | Rust fixed-layout fields are a poor analogy for dynamic ECMAScript lookup | Minimal work | An empty analysis result does not establish absence | **Reject** |
| Treat every missing field and every copy as arbitrary code | Loses diagnostics for ordinary `{ value: 1 }` descriptors and unrelated target keys | ECMAScript has narrower positive cases; Rust structural fields likewise do not imply arbitrary calls | Cheap fallback, unnecessary loss of precision | Safe fallback, but fails required positive controls as a complete policy | **Reject** as the finished direction |
| Build a complete path-sensitive mutable heap interpreter | Could recover more precision but creates a difficult observable checking model | Rust borrow checking uses different ownership guarantees; it is not a solution to arbitrary JS heap behavior | Potentially very expensive, with cycles, callbacks and path explosion | Cannot generally decide arbitrary JavaScript effects | **Reject** as the prerequisite or normative oracle |
| Positive ordinary descriptor evidence plus monotone possible flows and conservative fallback | Ordinary data descriptors retain useful errors; genuinely uncertain paths defer to runtime | ECMAScript presence/descriptor operations are authoritative; Rust is useful only for separating places from values and authority | Finite alias graph and bounded propagation; no checking-time traps | Known absence is distinct from unknown; exclusions and possible later enumerability changes participate | **Recommend** |

The prototype recognizes optional-field absence along an inspected ordinary engine-owned prototype chain. It joins supported current-source field values and installed accessors, including dependencies on the inspected descriptor prototype chain, without executing a lookup. This is a snapshot and a bounded source summary, not proof of permanent prototype stability. A broader audit must cover intervening prototype changes, key coercion, replaced reflective methods and arbitrary callbacks.

Object rest and descriptor-map copying retain supported keys rather than flattening all values into one wildcard. `Object.assign` writes those keys to its target, so an unrelated setter is not automatically invoked. Known non-enumerable installed accessors do not supply rest reads; a possible later change to enumerability can make them relevant. String copying preserves its UTF-16 index properties: the character values carry no capabilities, but writing them can invoke a target setter. An empty string contributes no such properties. Unknown keys and metadata remain conservative. Historical hooks accumulate: this increment does not prove that every subsequent replacement or deletion makes an older accessor impossible.

## Question 3: should writes and reflective aliases share read summaries?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Analyze every member expression as a read | Simple implementation, surprising getter effects on `obj.x = value` | Rust place expressions distinguish locating storage from reading a value; ECMAScript distinguishes Reference evaluation and GetValue | Cheap AST traversal | Invents a getter invocation; misses setter-only compound paths | **Reject** |
| Implement each spelling independently | Direct calls work while `call`, `apply`, binding or returned aliases silently diverge | Neither language gives equivalent callable routing new semantics merely for a different spelling | Duplicated summaries and inconsistent maintenance | Loses alias/argument/receiver flow | **Reject** |
| Use operation-aware targets and shared invocation summaries | Plain writes, compound writes and reflective routes agree; a wrong value still reaches its receiving check | ECMAScript Get/Set and Rust place/value distinctions support the separation, with different borrowing rules | Reuses summaries; source parameter/receiver edges remain private | Preserves setter arguments, explicit `Reflect.set` receivers and object-returning reflective aliases | **Recommend** |

`Object.defineProperty` and `Object.defineProperties` return their target; `Reflect.defineProperty` returns a Boolean. The analysis distinguishes those results. Bound reflective calls share argument propagation with direct calls; unavailable key/arity information falls back conservatively. Setter receiver flow includes the explicit `Reflect.set` receiver where provided. This adds no new receiver binding or runtime call.

No descriptor operation bypasses an existing prototype obligation, converts a getter result into a reference, revives a dead borrow, or changes error propagation. Runtime counters and abrupt-completion tests guard against replay, while the broader focused gate retains reference and nominal-prototype coverage.

## Scope and remaining work

This increment repairs the recorded descriptor/accessor cases and preserves the passing prototype controls in its probe set. It does **not** claim a complete prototype-mutation or reflective-intrinsic proof. The [tenth increment](property-key-effects.md) covers the recorded ordinary property-key coercion cases. Remaining work includes arbitrary relinking of descriptor prototype chains, precise removal/replacement of old hooks, more implicit operations and key coercion, replaced builtin methods, callback combinations and other static consumers. The selected direction allows more positive summaries when they can be specified and tested; it does not let an engine's analysis budget choose arbitrary observable rejection behavior.

Milestones 5 and 6 remain open. The earlier deep alias-analysis overflow, baseline test failures and build blockers are recorded separately. No production-engine benchmark or complete extension/test262 run is claimed.

## Primary references

[ECMAScript property descriptors](https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html#sec-topropertydescriptor), [ordinary Get/Set operations](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinarysetwithowndescriptor), and [Rust field expressions](https://doc.rust-lang.org/reference/expressions/field-expr.html) supply the comparisons. The Rust reference includes automatic dereferencing and qualifications on field borrowing; ordinary Rust fields are not a direct model for JavaScript getters or mutable prototypes. The performance discussion is an engineering assessment, not a measurement of a production engine.
