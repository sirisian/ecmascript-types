# Opaque values across source boundaries

This design complements [eval and stable names](eval-name-stability.md) and the shared [origin-evidence rules](origin-evidence-audit.md).

## Problem and recommendation

Evaluate these as separate Scripts in the same realm:

```js
// First Script
const box = new Map([["world", globalThis]]);

// Second Script
function f(): string { return "old"; }
box.get("world").f = () => 1;
const n: number = f();
```

The preceding implementation rejected the second Script using `f`'s former string result. An earlier accessor returning `globalThis`, or an earlier `Function("return globalThis")`, reproduced the same problem. Declaring `f` in the consuming Script matters: different placement can already withdraw its result for another reason and conceal this gap.

**Recommendation:** treat unavailable safe value/effect evidence as opaque at the operation using it. Preserve independent contracts and already captured values. Use positive summaries to retain precision where the operation is established; never equate an absent graph edge with absence of effects. This repairs the examples without opening collection internals, running a getter or interpreting generated source during checking.

This is an extension of the effect judgment, not a new public type. An opaque value can carry a capability. An opaque callable can have ambient effects when invoked without itself being the global object. A callable's result contract, its implementation identity and its effects remain three separate questions.

## Requirements

A valid direction must satisfy all of these:

1. Accept the three replacements above and check the actual result at an explicit receiving boundary. Retaining a runtime check does not excuse a false Early Error before execution.
2. Never obtain checking evidence by executing getters, proxy traps, functions, constructors, coercions or dynamic strings. Exceptions and side effects occur only in ordinary evaluation.
3. Distinguish ordinary own-data evidence from accessors, absent properties and exotic objects. A data property containing the JavaScript value `undefined` is still data evidence.
4. Include implicit accesses and ambient effects on reachable mutable global lexicals. The global object need not appear as an argument.
5. Preserve lexical identity, independent local/module/immutable contracts and callee capture order. Reading a callable value is not invoking it.
6. Keep result inference, receiving admission, public type identity, reference authority and reference liveness distinct. No wrapper, extra lookup or replay is permitted.
7. Use finite private compiler facts. Neither hidden-class changes nor new user-object metadata is required. A production implementation must implement the specified observable judgment; arbitrary analysis budgets are not a language-level choice of error phase.

## Decision 1: what does unavailable external evidence mean?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Empty candidate set means no capability/effect | Small refactors across Script boundaries unexpectedly reject valid replacement code; autocomplete can promise the old result | Rust function signatures do not promise purity; ordinary ECMAScript lookup can run a getter | Cheapest, but optimizes using a false premise | Fails the motivating cases | **Reject** |
| Execute or inspect everything needed to recover the value | Appears precise until checking increments a counter or throws | Rust does not execute arbitrary foreign functions to establish their effects; ECMAScript internal slots do not authorize user-code execution during checking | Unbounded execution, host coupling and potentially huge retained heaps | Violates phase, single-evaluation and abrupt-completion rules | **Reject** as the general rule |
| Any external object erases all source contracts | Few false static rejections, but unrelated APIs lose useful diagnostics and completions | Neither Rust storage rules nor ECMAScript lexical identity imply this | Cheap scan; loses specialization even for independent locals | Confuses external reachability with all bindings and contracts | **Reject** |
| Opaque values/effects at their use sites, with positive evidence where available | Supports dynamic interoperation while retaining stable contracts; wrong guesses meet ordinary receiving errors | ECMAScript supplies the direct lookup model. Rust's distinction between function type and effects is useful, but its alias guarantees are stronger | Finite summaries and dependencies; no additional object fields or IC states | Meets the requirements for supported routes; incompleteness must remain explicit | **Recommend** |

Ordinary own data supplies a possible value, not a guarantee that the property will never change. Missing own data is different: an inherited getter, proxy or other unavailable path can execute code and return a capability. This implementation does not walk ordinary prototype chains to recover inherited data. Such a path is conservatively opaque even when a more complete positive summary could be precise.

An earlier callable receives a possible invocation effect independently of its stored result annotation. For example, `const get = (): string => "old"` can still supply its enforced string result to a later source. Merely copying that function from an ordinary data property does not run its body. Invoking an unsummarized external callable can reach ambient globals even with no arguments; classifying it only when a global argument is passed misses this case.

## Decision 2: how should native and known intrinsic calls differ?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Trust every native callable | Familiar builtins keep precise errors | Native Rust/FFI implementations are not automatically pure either | Very small classifier | Native methods can return stored capabilities or enter user callbacks/coercions; an aliased `Map.prototype.get` demonstrates the hole | **Reject** |
| Treat every native call as affecting every mutable global, without exceptions | Safe fallback but even `String("f")` and `Symbol("key")` change unrelated error phases | Rust signatures alone lack effects, but known operation semantics still matter | Loses optimization and diagnostics unnecessarily | Broader experiment regressed existing lexical and construction-result controls | **Reject** as the finished design |
| Require a whole builtin/host effect system before handling opaque source boundaries | Potentially precise, but leaves the demonstrated defects in place meanwhile | Rust effect/alias rules are not a ready-made specification for arbitrary JavaScript hosts | High engineering cost is not itself an objection; a complete host inventory still cannot eliminate unknown code | Does not remove the need for an opaque fallback | **Reject** as a prerequisite; retain systematic summaries as follow-up |
| Opaque fallback plus positive, operand-sensitive intrinsic summaries | Precise common cases coexist with safe interoperation; completion displays need not claim that an uncertain implementation is unchanged | ECMAScript intrinsic algorithms establish the facts; Rust offers only the limited signature/effect distinction | Small reusable summaries can fit existing compiler IR and dependency tracking; no wrappers or new shapes | Preserves literal coercion controls while keeping object coercions and unknown native aliases conservative | **Recommend** |

The current narrow additions cover intrinsic `String` and `Symbol` calls with no argument or a primitive literal as the first argument. Those operations cannot invoke an object's coercion hooks. Arguments are still evaluated normally, including unused extra arguments. With an object first argument, coercion can call user code. Bound calls must inspect the bound argument's facts, not infer safety from a later literal passed to the bound function. `apply` remains conservative when its packed argument values are unavailable.

This is positive knowledge of intrinsic identity and operands, not a whitelist by spelling or a promise about every function implemented in native code. Existing intrinsic summaries are not a complete builtin effect system. A larger summary catalog must specify its required precision and include coercion, callbacks, abrupt completion, replacement and inherited lookup cases. More compiler effort can improve internal optimizations, but cannot change mandatory Early Errors arbitrarily.

## Decision 3: where do implicit effects occur?

```js
// box was created in the earlier Script:
// { get field() { globalThis.f = () => 1; return undefined; } }
function f(): string { return "old"; }
const { field: n: number = f() } = box;
```

The getter precedes the default's call. Conversely, in `const {field} = take(f())`, the initializer calls `f` before destructuring starts. Using the pattern's textual position for a synthetic property access reverses that order.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Only explicit member syntax contributes effects | Destructuring/spread refactors change behavior unexpectedly | ECMAScript operations include hidden Gets; Rust patterns do not model arbitrary JavaScript getters | Small scan | Misses destructuring, copies, descriptor conversion and apply containers | **Reject** |
| Put synthetic pattern accesses at their source-text offsets | Easy implementation, hard-to-explain lost capture evidence | Neither ECMAScript evaluation order nor Rust's place evaluation is textual-offset ordering | Cheap | Can place a getter before its RHS initializer | **Reject** |
| Discard order for the whole source | Removes some false assumptions but erases earlier captures and unrelated evidence | Rust immutable captures and ECMAScript captured values both oppose this collapse | Cheap, broad optimization loss | Fails existing capture controls | **Reject** |
| Attach effects to semantic operations and preserve established capture points | Users retain ordinary evaluation order and predictable receiving errors | ECMAScript is the direct authority; Rust is only an analogy for preserving an already obtained value | Can reuse compiler operation/CFG positions; no runtime operation added | Models RHS-before-pattern and lookup-before-default without executing either | **Recommend** |

The prototype attaches pattern projections to their enclosing declaration or assignment operation and treats reads inside the pattern conservatively. It preserves the RHS capture distinction but joins uncertain order within the pattern, including computed keys and defaults. It does not yet implement a full event graph for every pattern or iterator protocol. Spread, supported reflective copying/descriptor operations, and apply's length/index accesses also carry operation sites. `apply` obtains its argument-container length even for a zero-parameter target.

## Decision 4: what survives withdrawal?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Erase every type descended from the affected expression | Autocomplete and independent factory contracts disappear after unrelated calls | A Rust callable's type is distinct from its effects; this proposal's shallow admission makes implementation proof a further issue | Easy broad invalidation; loses independent specialization | Repeats the rejected blanket contribution change | **Reject** |
| Keep the former result and add a runtime identity guard | Stable-looking completions, but valid code can already have been rejected | JIT guards in ECMAScript engines are not Rust static proofs | Useful for optimization, not irrevocable checking decisions | A runtime guard cannot undo an Early Error | **Reject** for static acceptance |
| Withdraw only dependent current-value evidence; retain independent contracts and receiving checks | Stable APIs remain visible; uncertain calls cannot falsely promise the former result | Rust helps distinguish type/effect facts; actual ECMAScript lexical identity and this proposal's boundary rules govern | More precise dependency tracking, fewer discarded optimizations, no shape changes | Preserves earlier immutable capture, local/module isolation, explicit receivers and reference rules | **Recommend** |

Mutable global declarative bindings are reachable through opaque ambient code such as indirect eval. Function-local and module-local bindings are not replaced merely by matching names. Immutable bindings cannot be rebound by such code. This says nothing new about mutation of their referents, reference permissions or lifetime. No effect summary certifies a live reference or authorizes skipping a liveness check.

Autocomplete can show an independently declared callable/storage API while treating a result that depends on a replaceable implementation as unavailable evidence. There is no proposed user-facing `opaque` annotation and no LSP implementation in this patch. Guessing the old result can incur a runtime receiving TypeError; the system must not silently admit the wrong result or pre-reject a valid replacement.

## Implementation boundary and next gate

The new tests cover the three recorded cross-source counterexamples, ordinary/native/generated/bound/optional/proxy call routes, inherited and own accessors, ambient lexical effects, implicit projections/copies, coercion, receiving failures, capture order and single evaluation. They do not establish a universal absence-of-effects proof.

The [eighth increment](implicit-protocol-effects.md) implements rest and iterator summaries and repairs duplicate iterator-method acquisition. Continue milestone 5 with a positive-evidence audit of other static consumers, unsummarized builtin operations and callback paths, and descriptor and class/prototype operations. Preserve a regression for each newly discovered route rather than silently classifying an unsupported expression as empty. The separate existing deep `operandParticipates` stack overflow remains unresolved. Milestone 6's integrated/full-suite gate is still open.

## Sources and limits of the analogy

- [ECMAScript ordinary object operations](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinaryget) distinguish own data, prototype lookup and accessors. These are the direct semantic constraints.
- [Rust function pointers](https://doc.rust-lang.org/reference/types/function-pointer.html) and [interior mutability](https://doc.rust-lang.org/reference/interior-mutability.html) help separate callable contracts from effects and storage guarantees. This proposal does not acquire Rust borrowing guarantees from JavaScript `const`, a readonly view or shallow function admission.
- [V8 hidden classes and dependencies](https://v8.dev/docs/hidden-classes) explain why private compiler summaries are preferable to adding fields to ordinary objects. Reusing CFG/dependency infrastructure is an engineering inference, not a measured performance claim for this interpreter or a claim that V8 implements the proposal.
