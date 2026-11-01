# Runtime enforcement after origin analysis

This is the first implementation part of milestone 5 in the [eval and stable names plan](eval-name-stability-plan.md). It closes receiving-boundary bypasses exposed by auditing the remaining elision consumers. The [implementation record](runtime-enforcement-audit-implementation.md) records validation and the unfinished static-contribution work. It does not claim that the complete milestone or integrated conformance gate has passed.

## The observed failure

```js
function f(): uint8 { return 1; }
function g(): uint8 { return f(); }
const world = globalThis;
world.f = () => "bad";
g(); // must throw TypeError at g's declared result boundary
```

On the pulled engine base, `g()` returned the string. There was no lexical assignment to `f`, so an old spelling fallback treated it as unchanged and omitted `g`'s result check. The same bypass occurred through indirect eval, `Function`, a replaced method and parenthesized direct eval in a local scope.

Immutability is not sufficient either:

```js
function bad() { return "bad"; }
const f: () => uint8 = bad;
function g(): uint8 { return f(); }
g(); // must throw TypeError; storing f did not check every future result
```

The existing shallow callable admission can accept the untyped implementation. `const` preserves that implementation; it does not make the advertised result true. Wrapping the callable was not part of that admission and is not introduced here.

A property view has the same limitation. A declared getter can be shadowed with `Object.defineProperty`, and a readonly structural getter can return a different value later through an erased captured dependency. Its previously admitted receiver does not establish that every future property result satisfies the view. Ordinary members and `super` reads must retain their receiving checks without performing another lookup.

## Required invariants

1. A declared receiving boundary remains effective after shallow admission, replacement, dynamic source and property access.
2. Existing independently established inferred contracts, generic activation identities and construction dependencies remain effective. Disabling an optimization must not discard a semantic obligation.
3. The actual callee and receiver are captured before arguments. Property keys, getters, arguments, spreads, invocation and construction run once in their specified order.
4. A failed guard or admission retains prior effects and the original failure boundary. It does not retry overload selection or user code.
5. Prototype obligations, materialization, value copying, conversion effects and reference liveness remain independent. Equal Type Objects do not prove these conditions.
6. Static diagnostics need static evidence. A prospective runtime guard cannot justify an irreversible rejection before execution.

## Decision 1: what authorizes omission of a call-result check?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Trust the advertised callable type, including const and readonly storage | Completions look precise, but a wrong implementation silently leaks an incompatible value | Rust's typed function pointers do not model this proposal's shallow untyped admission | Few checks, but based on a false premise | Fails invariant 1 even without mutation | Reject |
| Require every admitted callable to be wrapped or deeply checked | Makes a callable view a stronger promise, but changes identity, call behavior and when errors arise | Rust supplies typed callable compatibility, not a JavaScript wrapper model | Wrappers add allocation/dispatch concerns and may interfere with IC targets; arbitrary behavior cannot be proved by inspecting a body | Conflicts with the accepted shallow-admission contract | Reject for this repair; a different callable-admission design would need separate adoption |
| Always check every call result against the advertised signature | Predictable failures even when no declared receiving boundary exists | Rust's result guarantee is part of its callable model, not a new check at an arbitrary JavaScript use | Adds admission at all calls, including discarded results; structural checks may execute user code | Creates obligations and effects the proposal does not currently impose | Reject |
| Elide only with evidence of the actual implementation and its enforced result; otherwise retain the existing receiving boundary | Advertised signatures remain useful, but autocomplete must not imply that shallow admission validates behavior. Wrong guesses fail at the existing checked boundary | Rust distinguishes callable type from compile-time knowledge of a particular function; its stronger admission assumptions do not transfer | Isolated declarations and literal functions can have static proof. A production engine can use existing identity/dependency mechanisms for safe guarded paths, without wrapper or instance fields | Satisfies the receiving and effect invariants while preserving shallow admission | Recommend, independently of implementation effort |

The engine's positive proof follows a fresh function literal or an unchanged isolated function declaration, then immutable aliases of that actual value. It matches the signature's provenance to the implementation and requires an enforced declared result. A signature created by a callable storage annotation has different provenance. Decorators on the declaration or its export can defeat implementation identity. Parameter values, arbitrary property results, imports without implementation evidence and unknown global implementations retain checks. Existing restrictions on published inferred returns remain in force.

This is deliberately separate from the class-value transfer map. Knowing a constructor's declaration can establish ancestry; it does not establish an arbitrary callable's return enforcement.

## Decision 2: how should unavailable global identity affect static contributions?

The audit tried applying the stronger runtime identity requirement directly to the shared static-signature predicate. That was too broad: it also removed existing published result contracts and generic-family dependencies. In the focused exploratory run, it caused failures in construction-effect sequencing, callable activation identity, result anchoring and reference-location diagnosis. Those results are evidence against that implementation shortcut, not permission to ignore global replacement.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Preserve the old spelling screen as proof of both static results and runtime elision | Familiar inference, but global aliases and dynamic source invalidate the promise | Rust's lexical mutation analysis is not analysis of a JavaScript global object | Cheap scan; unsound check removal | Reproduces the bypass | Reject |
| Make every Script-global function contribution unknown | Erases useful factory, overload and inferred-binding contracts even in ordinary source with no relevant effect | Rust does not erase a callable's declared type merely because exact function identity is unavailable | Easy to implement; loses specialization and useful typed flows | Conflates an established contract with a speculative implementation fact and loses independent obligations | Reject as a blanket implementation rule |
| Keep all static guesses and rely on a runtime guard to repair any result | Rich completions, but an early rejection cannot be repaired when the program never runs | Rust's compile-time checking also cannot be retroactively repaired by a runtime branch | Runtime guards are feasible; they do not solve compile-time acceptance | Fails invariant 6 | Reject |
| Separate contract preservation, source-position result evidence and runtime elision proof | Retains genuine contracts and marks unavailable results conditional; wrong guesses remain checked where a contract exists | Rust is a useful contract/value comparison, but supplies neither eval analysis nor this proposal's inferred publication rules | Distinct compiler facts can share lexical and CFG infrastructure; no new object shape required | Can satisfy all invariants without weakening activation or liveness rules | Recommend; the runtime-proof separation is implemented here, the broader static-consumer conversion remains unfinished |

The new runtime proof does not reuse the static predicate's global spelling fallback. Direct known writes to `globalThis.f` now also withdraw an otherwise unproved declaration-result assumption. Parenthesized eval is recognized by both mutation analyses, and a possible direct eval prevents an outer const from being treated as unshadowable solely because of its storage kind.

There is a concrete remaining static limitation:

```js
function f(): string { return "old"; }
const world = globalThis;
world.f = () => 1;
const x: number = f();
```

The alias-mediated write is not yet represented by the static contribution screen and can cause an inappropriate Early Error from the old result signature. The runtime fixes in this increment do not claim to solve this case. The next static-consumer work must represent the unavailable result evidence without globally deleting independent result contracts. It must cover global aliases, reflective writes, indirect dynamic code, imports, deferred bodies and source boundaries, and preserve the regression suites that exposed the blanket approach's losses. The recommendation is to complete that separation, not to accept the false rejection or weaken the tests.

## Decision 3: how does retained admission interact with properties and guards?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat readonly or a previous membership test as permanent descriptor proof | Surprising when an erased alias or an own shadow changes what lookup returns | Rust borrow restrictions are materially stronger than a readonly JavaScript view | Cheap but invalid shape/descriptor assumption | Does not establish future values or getters | Reject |
| Freeze receivers/prototypes or wrap every property access | Restricts ordinary JavaScript mutation and adds new behavior | Rust ownership is not blanket freezing of a JavaScript object graph | Can disrupt shapes, IC behavior and object identity; imposes work beyond the boundary | Changes accepted mutation and prototype-obligation semantics | Reject |
| On guard failure, re-read and retry with a different target or overload | Getters and arguments may run twice; a different receiver or value may be selected | Neither JavaScript operand capture nor Rust's value reasoning supports replay as an invisible fallback | Requires replay/deoptimization machinery and duplicated effects | Fails invariants 3-5 | Reject |
| Check the already obtained value; allow an independent, non-observable implementation proof | Preserves ordinary lookup and failure order; readonly remains a view permission | ECMAScript's captured operands are the direct model; Rust's place/value distinction is only an analogy | Uses existing receiving admission. Production IC/dependency guards can prove suitable cases without changing layouts | Preserves effects, storage identities, conversion ordering and liveness | Recommend |

An engine may use an actual captured-function identity guard or descriptor/dependency proof where that preserves semantics. Implementation identity includes its captured and generic environment; a shared syntax node is not a runtime activation identity. It must not guard by repeating a getter. On failure, the current execution continues through its required admission with the same operands and locations. Such a guard is an implementation technique, not additional evidence for mandatory static errors. This interpreter increment conservatively retains admission where it lacks static proof; it adds no speculative execution or deoptimization machinery.

## Audit disposition

| Seam | Disposition in this increment |
| --- | --- |
| Calls, parenthesized calls, optional live calls and tags | Shared positive result-enforcement proof; no syntax bypass |
| Shallow const storage and callable parameters | Storage identity is insufficient; required receiving admission remains |
| Ordinary and super property reads | Retain admission without another Get; readonly alone does not establish descriptor identity |
| Local direct eval, including parentheses and sloppy injection | Conservative mutation screen retained and corrected |
| Exported decorators | Declaration and export decorators defeat exact callable implementation proof |
| Callee/receiver capture, argument effects and abrupt completion | Existing evaluator order retained; tested with getter, mutation and throw traces |
| `new`, constructor-returned objects, heritage and inherited contracts | Existing constructor-origin and runtime admission machinery retained; adjacent suites verify these boundaries |
| Prototype obligations, reference location and liveness | Existing machinery retained; no authority comes from the new proof |
| Overload selection and generic result dependencies | Existing contracts preserved; a check fallback does not retry resolution |
| Global static contribution and early-negative evidence | Unfinished, with the concrete counterexample above; milestone 5 remains open |
| Full integrated effect/mode/recovery and extension gate | Milestone 6 remains open |

## Sources and limits

- [ECMAScript EvaluateCall](https://tc39.es/ecma262/#sec-evaluatecall) and [tagged template evaluation](https://tc39.es/ecma262/#sec-tagged-templates-runtime-semantics-evaluation) supply the operand/receiver ordering used here. No new evaluation order is proposed.
- [Rust function pointers](https://doc.rust-lang.org/reference/types/function-pointer.html) demonstrate a callable type whose exact function identity need not be known at compile time. This is a limited comparison: Rust does not thereby model JavaScript eval, shallow admission or this proposal's runtime reference liveness.
- [V8 Maps and constant-property dependencies](https://v8.dev/docs/hidden-classes) describe maps, descriptors, ICs and invalidation of optimized constant-property assumptions. Applying related internal guards here is an engineering inference, not a claim that V8 implements this proposal or a benchmark of this patch.
- The [original plan](eval-name-stability-plan.md#sources-and-limits-of-the-comparisons) supplies the broader declaration, Rust lifetime and prototype comparisons.
