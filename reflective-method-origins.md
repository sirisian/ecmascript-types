# Reflective method identity and effect summaries

This design complements [eval and stable names](eval-name-stability.md) and the shared [origin-evidence rules](origin-evidence-audit.md).

## Problem and recommended rule

```js
// First Script
Object.defineProperty = function () {
  globalThis.f = () => 1;
};

// Second Script
function f(): string { return "old"; }
Object.defineProperty({}, "x", { value: 0 });
const n: number = f();
```

The pulled engine rejects the second Script before execution. Its global-effect analysis assigns the intrinsic `defineProperty` summary from the receiver/property spelling, even though the actual method was replaced. The intrinsic summary says nothing about this replacement's ambient write to `f`.

The converse also matters:

```js
// First Script
const original = Object.defineProperty;
Object.defineProperty = function () { globalThis.f = () => 1; };

// Second Script
function f(): string { return "old"; }
original({}, "x", { value: 0 });
const n: number = f(); // retain the ordinary known-result error
```

A captured original is still the original function. Looking at the current namespace property cannot identify it. The pulled analysis instead treats this earlier-source callable as opaque and loses a useful error.

**Attach a supported effect summary to the established callable value, independently of the property or binding through which it is reached.** Method lookup and invocation are separate effect sites. An actual getter, proxy wrapper, replacement or unrecognized callable retains its applicable possible effects. A captured original keeps its summary, subject to that summary's operand and stability conditions.

This continues milestone 5's distinction between unavailable result evidence and a proven incompatible result. It changes neither public function type equality nor shallow callable admission. All declared receiving boundaries and reference-liveness rules remain in force.

## Conditions for a valid direction

| Condition | Required consequence |
| --- | --- |
| A mutable property can hold a replacement | Its spelling cannot select the original intrinsic summary |
| Captured values survive unrelated storage edits | Replacing a namespace property cannot retarget an existing function value |
| Lookup can itself invoke code | Account for accessor/exotic lookup before evaluating call arguments |
| A native function can invoke arbitrary user hooks | Native implementation alone supplies no effect-free proof |
| Supported summaries have operand conditions | Knowing the helper does not make keys, descriptors, targets or argument containers harmless |
| Function signatures and implementation identity differ | A compatible replacement receives no original-implementation privileges |
| Earlier captures and runtime checks remain authoritative | Withdraw only affected evidence; never replay a lookup or bypass a receiving boundary |

## Question 1: how is a reflective summary selected?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Select by `Object`/`Reflect` and property spelling | Autocomplete appears to promise behavior that a legal replacement need not have; a wrong guess can reject runnable code before entry | Rust module paths resolve items, unlike mutable JS object properties | Cheap string dispatch | Fails the replacement counterexample | **Reject** |
| Select by name, native callability or signature | Familiar function metadata looks reassuring but can describe a different implementation | Rust function pointer signatures admit multiple functions; function-item identity is stronger | Cheap metadata tests | Names and compatible signatures do not establish an effect summary; native helpers can call hooks | **Reject** |
| Treat every reflective callable as opaque | Conservative for replacement effects, but loses diagnostics for original helpers and stable aliases | Rust retains item-specific knowledge where available instead of discarding it at every copy | Simple, but unnecessarily loses checking and optimization evidence | Handles one side of the problem and fails captured-original precision | **Reject** as the complete direction |
| Identify the callable value and apply its supported summary | Copies and renaming behave consistently; completion lists still show public contracts without promising storage immutability | Rust function items demonstrate implementation-specific identity; JS needs private value evidence because its ordinary function types do not encode that identity | Constant-time identity metadata, independent of object shapes; ordinary property ICs remain applicable to lookup | Handles replacement and capture independently, preserving operand conditions | **Recommend** |

The public type of a replacement may be compatible with the original while its effects differ. This rule does not introduce public identity types or make Rust's function-item types part of the proposal. Rust is a close analogy for the distinction between signature and implementation, and a poor analogy for a mutable JavaScript namespace lookup.

Autocomplete should continue to report declared callable contracts. It should not hide legal replacement behavior or imply that an ordinary property is frozen. An unsupported implementation defers an affected result judgment to its receiving check; it does not silently admit the wrong value.

## Question 2: what happens when there is no supported summary?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat an unrecognized namespace member as inert | Ordinary-looking code gets false Early Errors when key coercion, proxies or callbacks change a global | Rust has no rule that an unknown function body is pure; ECMAScript reflective algorithms explicitly perform hooks | Cheap but omits calls | Reproduced by `Reflect.has({}, key)` with an effectful key | **Reject** |
| Execute the method, getter or a generated body while checking | Diagnostics gain side effects and can depend on speculative execution | Neither Rust type checking nor ECMAScript property lookup licenses replaying a user's runtime operation | Unbounded cost and reentrancy | Violates phase, order, throw propagation and single evaluation | **Reject** |
| Require an effect annotation for every native/replaced callable | Forces new public syntax and annotations onto existing APIs and replacements | Rust's callable traits specify invocation interfaces, not a blanket no-global-effects guarantee | Large enforcement and compatibility burden | Does not by itself establish implementation identity or completeness of a user claim | **Reject** for this problem |
| Retain possible effects while extending positive summaries deliberately | Uncertain calls keep runtime boundaries; supported harmless cases retain errors | Consistent with requiring actual proof in Rust and with ECMAScript's specified observable operations | Bounded analysis; engines can add established intrinsic facts without wrapping objects | Prevents missing summaries from becoming false guarantees | **Recommend** |

For example, the actual `Reflect.has` converts its key before the target's property-presence operation. Knowing that the target is a fresh ordinary object does not remove an object's key-conversion hook. This increment routes unrecognized native methods through the existing opaque-call path; it does not claim a complete positive summary for `has`, reflective invocation, prototype operations or every proposal-specific reflection method.

Absence of a summary is a conservative prototype limitation, not a language rule allowing implementations to choose observable Early Errors arbitrarily. The normative positive-evidence model still needs its complete, testable coverage. The implementation must preserve existing independent contracts and supported proofs while filling this gap.

## Question 3: how should identity survive capture, renaming and realms?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Compare with the current namespace property | A saved original loses its behavior when an unrelated property is reassigned | Rust item identity is independent of a later function-pointer variable's contents | Cheap lookup, but repeats a mutable observation | Can misclassify a replacement as original and lose a genuine original | **Reject** |
| Add a writable public tag or privileged function name | Exposes implementation machinery in normal APIs and permits imitation | Rust item types are compiler-established, not self-asserted public string tags | Adds observable property/shape concerns and validation work | A forged tag cannot justify a summary | **Reject** |
| Limit every summary to the current realm and direct spelling | Cross-realm originals and aliases needlessly lose diagnostics | Rust does not have the same realm model; its item analogy does not justify this restriction | Simple but discards already established identity | Safe subset, incomplete handling of captured values | **Reject** as the finished direction |
| Keep private identity metadata associated with original function objects | Capture and renaming retain behavior; wrappers remain distinct values | Closest to implementation-specific compiler knowledge, without exposing Rust-style item types | Private weak metadata or equivalent engine builtin IDs; no public property or shape transition | Records originals before mutation, supports their aliases and foreign-realm values, does not bless proxies | **Recommend** |

The prototype records six function identities during realm initialization: Object's `assign`, `defineProperties` and `defineProperty`, and Reflect's `set`, `defineProperty` and `deleteProperty`. It uses a private weak map; a production engine can use existing builtin-function identities. This metadata selects the already supported summaries. It does not certify that the namespace property still contains an original when later code reads it.

The namespace objects now also participate as ordinary native data sources. Their own data properties can identify actual function values without running code. An accessor or unavailable/exotic read retains lookup effects. An unknown callable data value contributes an invocation effect when called, rather than merely because its property was read. A proxy around an original is a different callable and retains the proxy's behavior.

## Operation order and limits

Capture the method after its ordinary lookup, then evaluate arguments, then invoke the captured callable. A getter effect can change a value read by an argument. An argument-side replacement cannot retarget the callee already captured for that invocation. A throwing getter prevents argument evaluation. These observations require no second lookup, trap call or checking-time invocation.

All aliases still carry their applicable target, key, descriptor and argument-container effects. Identifying `Object.defineProperty` does not suppress a descriptor getter; identifying `Reflect.set` does not suppress key conversion or a target setter. Independent receiving checks reject wrong replacement results, and reference authority, liveness and prototype obligations are unchanged.

The [following invocation-adapter increment](invocation-adapter-origins.md) handles overridden and borrowed `call`/`apply`/`bind` methods, bound captures and adapter-internal operations. Milestone 5 remains open for broader positive intrinsic summaries, cross-source bound-function precision, arbitrary prototype and callback paths, overloaded-index effects and other static consumers. Historical flow joins can conservatively retain both old and new property values. This increment does not provide a complete heap or reachability proof. Milestone 6's integrated conformance gate and recorded baseline test/build blockers remain open.

## Primary references

[ECMAScript reflection algorithms](https://tc39.es/ecma262/multipage/reflection.html), especially `Reflect.has`, `Reflect.apply` and `Reflect.defineProperty`, establish the relevant hook order. [Rust function-item types](https://doc.rust-lang.org/reference/types/function-item.html) distinguish an identified implementation from a function-pointer signature. These comparisons guide the proof boundary; they do not import a new public effect or identity type into this proposal.
