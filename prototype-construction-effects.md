# Prototype construction and descriptor effects

This is the fifteenth milestone-5 increment of the [eval and stable names plan](eval-name-stability-plan.md), following [prototype relinking](prototype-relink-effects.md). The [implementation record](prototype-construction-effects-implementation.md) distinguishes the delivered subset from the recommended complete language model. Milestones 5 and 6 remain open.

## Recommendation

Keep ordinary prototype installation, descriptor processing and later prototype consumption as distinct operations with distinct effect positions. Recognize the original `Object.create` by callable identity, give its successful result a fresh allocation identity, and retain the supplied prototype as a dependency of subsequent lookup. Apply the same dependency model to object-literal prototype syntax. Descriptor conversion, including this proposal's `type` field and typed-value admission/defaulting, contributes effects during creation. A stored getter contributes invocation effects when actually consumed.

This extends private implementation evidence; it adds no public type, purity annotation, freezing requirement or implicit reference authority. Unknown evidence withdraws an unsupported result assumption. It does not establish a compatible receiving result, erase an independently established contract or justify skipping liveness checks.

## The observable problem

Assume an earlier Script established `proto` as follows:

```js
const proto = { run() { globalThis.f = () => 1; } };
```

The consuming Script is:

```js
function f(): string { return "old"; }
const target = Object.create(proto);
const before = f;
target.run();
const n: number = f();
```

Creation itself does not call `run`. The call through `target` can replace the Script-global object binding before the last read of `f`, so the former string result cannot justify an Early Error there. Changing the final initializer to `before()` retains the earlier capture and its incompatible string result. Treating all creation as opaque loses that distinction; treating creation and its result as wholly inert misses the later effect. Replacing `Object.create(proto)` with `({__proto__: proto})` should preserve this reasoning.

A second route acts during creation:

```js
// Earlier Script:
let reads = 0;
const candidate = {
  get x() { reads++; globalThis.f = () => 1; return 1; }
};
// Consuming Script:
function f(): string { return "old"; }
Object.create(null, {
  item: { type: type { x: number }, value: candidate }
});
const n: number = f();
```

The type-directed admission can consume `candidate.x`. A descriptor's own or inherited `type` getter is a separate possible effect. Neither is covered by merely reading the standard descriptor field names. Static analysis must not perform either operation to discover its result.

## Conditions a valid direction must meet

| Requirement | Observable consequence |
| --- | --- |
| Actual helper identity | Replacing a namespace property does not retain the original helper's summary; saved originals can retain it |
| Fresh allocation | Distinct creations do not share own slots merely because their keys match |
| Separate operation positions | Creation does not invoke unused prototype members; descriptor processing can run code immediately |
| Complete initial chain | Default, null, explicit object and unknown prototype alternatives remain distinguishable |
| Positive own-presence evidence | A possibly present descriptor entry cannot hide an inherited effect unconditionally |
| Capture stability | Later argument-side writes cannot retarget a factory already read |
| Runtime behavior | Receiver, access count, abrupt completion and typed admission remain governed by the existing algorithms |
| No authority expansion | Prototype reachability supplies no new borrow, alias permission or reference lifetime |
| Bounded implementation | Exhaustion and cycles lose precision, never certify effect-freedom |

The following decisions use the same five criteria. Recommendations are chosen for the language semantics, independently of implementation effort. Conservative fallback describes the current executable subset, not a license for engine resource budgets to select the language's observable error phase.

## Decision 1: what does construction mean for effect evidence?

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat every creation as an arbitrary ambient effect | Ordinary allocation unpredictably changes diagnostic timing; completion cannot explain the lost evidence | Neither ECMAScript allocation nor Rust function identity implies arbitrary invocation | Very cheap analysis; unnecessarily defeats local assumptions and optimization | Avoids some missed effects but merges distinct positions | Reject as the language rule; reserve for genuinely unavailable operation evidence |
| Treat recognized creation and everything reachable from its result as pure | Pleasant completion and fewer checks until a wrong guess rejects valid code | Confuses allocation with later method execution | Cheap but unsound optimization opportunity | Misses descriptors, traps and inherited calls | Reject |
| Require frozen prototypes or ban this syntax in checked code | Extra migration burden; completion suggests a restricted dialect | Rust has no directly comparable mutable prototype chain | May simplify guards but imposes new runtime restrictions | Changes supported JavaScript and does not make descriptor callbacks pure | Reject |
| Separate fresh allocation, descriptor operations and dependent later uses | Errors follow the operation a reader expects; a capture remains useful | Fits ECMAScript operation boundaries; Rust analogy is limited to identity evidence | Supports allocation/shape facts and narrow dependent invalidation; requires effect dependencies | Meets the conditions above | **Recommend** |

## Decision 2: which initial chain does literal syntax establish?

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Interpret every property named `__proto__` as a prototype setter | Autocomplete spelling appears sufficient, but computed/shorthand/method code gets surprising results | Conflicts with ECMAScript's syntax distinction; no Rust analogue | Simple but wrong classification | Can hide real own properties or invent inherited routes | Reject |
| Inspect or execute a candidate object to discover its chain | A diagnostic may itself invoke a getter or trap | Neither language requires checker execution of arbitrary user operations | Can execute unbounded code and disturb engine state | Violates ordering, access counts and checker non-execution | Reject |
| Carry syntax-specific, conjunctive chain evidence | Matches ordinary JavaScript forms; unknown alternatives cost precision rather than altered runtime behavior | Follows ECMAScript; Rust supplies no prototype rule to transplant | Small private facts, reusable prototype dependencies; ordinary shape/IC guards remain possible | Distinguishes null, default, ignored primitives and explicit/unknown objects | **Recommend** |

Only the noncomputed colon form establishes a literal prototype. Computed names, shorthand and methods remain own properties. A null source excludes the default chain; a non-null primitive ignored by the literal form retains that chain. `Object.create` has its own argument validation and must not inherit the literal form's primitive rule. A proxy installed as prototype is not traversed merely by installing it. The JSON-parsing exception remains governed by the existing runtime rule.

The implementation tracks null/undefined/other primitive facts privately. Every contributing expression needs evidence; one known object alternative cannot cancel an unknown alternative. An exact Boolean literal selector can select its reachable result arm, without becoming a general control-flow evaluator.

## Decision 3: when may a fresh own property hide inherited effects?

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Every candidate descriptor entry proves own presence | Optimistic completion; a wrong guess can select the wrong diagnostic phase | Descriptor enumeration permits callbacks; Rust field construction is not analogous | Cheap own-key set | A getter can remove a later descriptor-table entry | Reject |
| Never retain own-presence evidence | Common factory idioms lose useful diagnostics and inherited-route precision | More pessimistic than either language's positive construction facts | Prevents useful shape and lookup proofs | Safe fallback, insufficient final model | Reject as universal rule |
| Require a positive construction and availability proof | Ordinary fresh tables behave predictably; exotic tables cost precision | Fits ECMAScript's descriptor stages; Rust's fixed fields offer only a limited analogy | Reuses allocation and own-property dependencies; no new runtime shape | Conditional on successful creation and all relevant effects, hiding is justified | **Recommend** |

The delivered proof covers a directly supplied fresh descriptor-table literal containing only data definitions or shorthand, without spreads, methods, accessors or decorators. It cannot escape before the call. Known actual own keys can therefore hide same-key prototype routes after successful creation. Arbitrary tables, aliased tables and callback-bearing tables receive no such blanket guarantee. A descriptor conversion may still fail or run user code; the proof neither predicts success nor skips conversion. Later deletion/relinking retains its existing dependencies.

## Decision 4: which factory was invoked after its namespace changed?

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Use the latest namespace value for every earlier lookup | An argument silently changes the apparent callee; completion cannot convey that fiction | Contradicts ECMAScript evaluation order; Rust function-item identity also survives a copy | Simple mutable slot model | Retargets already obtained values | Reject |
| Join every write into every lookup without position | Sound conservative effects but common saved helpers lose evidence | Rust identity is more precise; JavaScript needs additional mutation tracking | Easy flow joins, excessive invalidation | Can incorrectly erase the distinction relevant to required Early Errors | Reject as the complete rule |
| Retain captured value identity and relevant write positions | Matches reading a callee before arguments; replacement values still behave normally | Rust function items illustrate identity versus signature, not mutable-property stability | Guards/dependencies can target the property read; historical joins stay bounded | Preserves captures, with conservative handling of loops/deferred contexts | **Recommend** |

For example, `Object.create((Object.create = replacement, proto))` uses the original factory read before its argument. The same principle applies if an argument uses `Object.defineProperty` to replace that property. The interpreter now filters native intrinsic-property write contributions by the lookup position. It retains possible earlier writes; it does not pretend to know the exact current heap, and it does not extend identity evidence to a compatible function signature.

## Decision 5: what does a typed descriptor authorize the checker to assume?

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Summarize only standard descriptor fields | Familiar JavaScript examples pass, while proposal-specific code receives misleading errors | Standard ECMAScript lacks this proposal's `type` extension; Rust cannot fill that gap | Cheapest extension | Misses `type` getters and admission/default effects | Reject |
| Infer purity from the declared type or getter signature | Autocomplete advertises a type but cannot promise no side effects | Rust function signatures alone do not identify an arbitrary implementation | Tempting check elision, invalid assumptions | Conflates result contract with effects | Reject |
| Treat every descriptor operation as opaque forever | Reliable fallback but makes ordinary descriptor-heavy code lose precision | No requirement from either prior-art model | Avoids analysis; broad invalidation harms optimization | Safe but discards available positive evidence | Reject as final direction |
| Include all fields and each admission/default operation, requiring positive effect proofs | Diagnostic cost follows actual uncertainty; types remain useful without implying purity | Consistent with identity/effect distinction; extension-specific semantics must be specified here | Specialized primitive/structural paths can use ordinary guards; arbitrary conversions retain effects | Accounts for getters, original receivers and type-directed callbacks | **Recommend** |

This delivery adds the `type` field to shared descriptor conversion analysis and conservatively retains an ambient effect whenever that field may be present. Proven absence adds none. This covers creation, Object/Reflect definition and multiple-property definition. Future precision should prove the actual admission or default operation harmless; it should not whitelist the spelling of a type. A broad fallback is preferable to an invented purity promise, but it is not the final prescribed positive-evidence model.

## Decision 6: how should ordinary inherited reads preserve existing precision?

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Make all inherited reads opaque, even proven ordinary absence | Harmless computed reads lose established diagnostics | Neither ECMAScript ordinary data reads nor Rust field access inherently invokes a callback | Cheap analysis, broad invalidation | Safe fallback but misplaces uncertainty when positive facts exist | Reject as universal rule |
| Whitelist a property name or assume every native getter harmless | Completion suggests a guarantee that replacements can violate | Rust identity concerns the implementation, not a mutable name | Cheap but invalid guards | Misses replacement getters and exotic receivers | Reject |
| Prove the actual lookup, accessor identity and applicable receiver operation | Ordinary reads keep useful evidence; user accessors and proxies retain effects | Consistent with the earlier identity discipline; no direct Rust prototype analogue | Ordinary descriptors and internal-method guards reuse existing metadata | Preserves access counts without executing the getter during checking | **Recommend** |

The delivered inherited-read path follows ordinary prototype data and proven absence without an invented callback. The original legacy prototype getter has a private identity recorded at bootstrap, separate from callable invocation summaries. Its no-user-code read requires an ordinary receiver operation; a replacement getter does not qualify. Unknown keys still consider candidate accessors. Direct invocation with an exotic receiver remains conservative. This is narrower than treating all native functions or the spelling `__proto__` as harmless.

## Implementation and production-engine implications

An allocation identity, initial prototype dependency and own-key facts can coexist with normal object shapes. A production engine can reuse existing property/prototype guards for optimizations whose failure is recoverable. None can retroactively justify an irrevocable Early Error. Keep analysis provenance separate from public type identity and from the runtime representation of references.

Source getters need the receiver of the original access, even when a descriptor field is inherited. Typed descriptor processing must retain its real order and abrupt completion. A static may-flow graph can conservatively connect candidates without executing them, replaying a read or establishing a new receiving boundary. This patch does not change runtime allocation, descriptor conversion, prototype obligations or reference-liveness algorithms.

The interpreter uses bounded propagation and bounded traversal of contributing flows. Native property-write dependencies add bookkeeping proportional to the relevant history, charged against the same work budget. This is an implementation experiment, not a measured production speedup. Shape sharing, inline-cache behavior and invalidation breadth need measurements in a real engine.

## Remaining work and acceptance

Acceptance for this increment includes inert prototype installation, immediate descriptor effects, later inherited calls/getters/setters/coercion, captures before effects, genuine runtime receiving failures, exact access/trap counts, null/default chains, computed `__proto__` controls, replacement/saved/borrowed/applied/bound/cross-realm helpers, distinct allocations, deletion and later relinking. The implementation record gives completed checks and baseline comparisons.

The complete milestone-5 direction remains positive, operation-specific evidence over every relevant consumer. Arbitrary legacy `__proto__` assignment, general descriptor transitions and conversion/default summaries, native callback consumers, overloaded-index effects, broader packed/bound argument precision and merged invocation contexts remain open under that direction. The array/default-prototype protocol audit is not declared complete. Milestone 6 retains integrated conformance, the known test/build blockers and the separately recorded deep-recursion stress failure. These are unfinished implementation and validation items, not undecided alternatives hidden by the matrices above.

## Primary references and analogy limits

- [ECMAScript Object.create and ObjectDefineProperties](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-object.create): allocation and descriptor processing are separate stages.
- [ECMAScript object initializer evaluation](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-object-initializer-runtime-semantics-propertydefinitionevaluation): syntax determines the special prototype form.
- [Rust function item types](https://doc.rust-lang.org/reference/types/function-item.html): an item carries implementation identity; coercion to a function pointer can retain only its signature. This does not make a mutable JavaScript property an immutable item or introduce Rust borrowing rules here.
- [V8 fast properties](https://v8.dev/blog/fast-properties): background on shapes and property representations. The proposed dependency/guard strategy is a design inference, not a claim that V8 implements this proposal.

The following [legacy prototype-setter increment](legacy-prototype-effects.md) extends the store path, including explicit receivers, typed admission and initial array/rest prototypes. The limits above describe this construction delivery; the newer implementation record identifies the next delivered subset.
