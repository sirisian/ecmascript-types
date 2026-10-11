# Value-sensitive argument spread routing

A spread supplies arguments according to its evaluated value. Moving a plain object literal into a variable must preserve its named contributions:

```js
function f(a: string, b: string) { return a + b; }
const args = { b: "B", a: "A" };
f(...args); // "AB", just like f(...{ b: "B", a: "A" })
```

## Required behavior

Evaluate each operand once in source order. A reference run forwards its locations and occurrence, including when empty. A tuple Composite iterates by kind. For other objects, read `Symbol.iterator` with GetMethod once: a callable method wins over named properties, a nullish hook selects named properties, and a present non-callable hook or thrown lookup fails. Invoke the obtained iterator method without looking it up again. Primitive values retain their ordinary iteration requirements.

For named properties, capture OwnPropertyKeys once. Visit String keys in that order, checking each current own descriptor and then reading its value exactly once if enumerable. Ignore Symbol keys without consulting their descriptors or values. Earlier getters may remove later properties, change their enumerability, or replace their values. New keys absent from the captured list are not visited. Repeated fixed names still assign sequentially, with the last value winning; every earlier getter runs.

Optional calls that short-circuit evaluate no arguments. Otherwise arguments precede the ordinary runtime callability/constructibility check, subject to earlier static checking. Abrupt argument evaluation stops later arguments and parameter defaults. IteratorStepValue keeps the ordinary argument-spread behavior: a failure in next or value does not invent an additional call to return. A completed iterator is not closed again when later name binding fails.

Bind the retained contributions against the callable view or each overload/case candidate. Neither selection nor a losing candidate replays effects. Ordinary empty spreads supply zero arguments; they do not invent placeholders or named-rest scope. Reference forwarding retains its separate permission and live-location checks, including empty occurrence metadata. This applies to calls, construction and super calls.

## Direction decisions

The criteria below are fixed: ergonomics includes autocomplete and the cost of a wrong guess; performance means a real engine, including shapes and inline caches. The recommendation is independent of implementation effort.

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Only object-literal syntax supplies names | Extracting a variable changes meaning; autocomplete cannot explain the same shape behaving differently | Rust has no named/spread call equivalent; ordinary JS iteration is value based | Syntax test is cheap but supplies no useful object-shape guarantee | Contradicts the existing value-based spread rule | Reject |
| Probe during checking or replay during candidate selection | Getters run unexpectedly; a wrong guess changes effects | Neither Rust call operands nor JS spread provide a replay model | Repeated lookups, allocations and invalidations; no IC benefit | Violates once-only source evaluation and can invalidate references | Reject |
| Evaluate source items once; specialize guarded positional cases | Refactoring preserves meaning; completion uses the effective signature's names | JS supplies protocol/evaluation rules; Rust supplies only the limited contract-oriented call analogy | Ordinary no-spread calls keep their fast path; known iterator guards can specialize; invocation metadata adds no user-object fields or shapes | Preserves value-selected routing, names, original references, empty runs and deferred liveness | Choose |

### Iterator selection and failure

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Read once to classify, again to iterate | A getter may return a different method; debugging a wrong guess is expensive | JS GetIteratorFromMethod consumes an already selected method; Rust has no matching hook | Two property IC accesses and effects | Can select one protocol then execute another | Reject |
| Treat any invalid hook as named fallback | Malformed iterables silently use parameter names; autocomplete cannot reveal runtime hook failure | JS GetMethod rejects a non-callable non-nullish property; Rust trait failures are compile-time and not equivalent | More fallback branches and shape-dependent surprises | Suppresses required abrupt completions | Reject |
| One GetMethod, nullish fallback only, tuple-kind exception | Stable once-only effects; a malformed hook reports failure | Follows JS method protocol plus this proposal's Composite rule; Rust is not an iterator-getter analogy | One hook lookup; guardable iterator identity; no new shapes | Preserves precedence, abrupt failure and tuple iteration | Choose |

### Property mutation order

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Freeze all enumerable names before any Get | Deleted later properties may be read through a prototype; surprising wrong guess | Rust has no reflective property-spread analogue; differs from JS CopyDataProperties/EnumerableOwnProperties with values | Requires an extra selected-key list; no shape/IC benefit | Once-only reads possible, but mutation order remains surprising | Reject and record explicitly |
| Re-enumerate after every getter | New properties can unexpectedly join the call; unstable completion suggestions | Neither Rust calls nor JS property copying provide this behavior | Repeated enumeration; possible nontermination under additions | No finite original contribution list | Reject |
| Capture keys once; descriptor and Get interleaved | Matches familiar object-spread mutation behavior; known signature names remain useful | Close to JS property copying, except Symbols do not name parameters; Rust lacks this feature | One enumeration; one descriptor/Get pair per retained String key; normal property ICs | Deterministic order, deletion/enumerability observed, no replay or new reference rights | Choose |

### Intrinsic fast-path boundary still open

Some proof-of-concept intrinsic interceptions consume positional values directly and have not yet been connected to complete named signature metadata. The implementation now refuses named/forwarded contributions at those interceptions instead of silently dropping their metadata. This is containment, not completion of named intrinsic calls or a new restriction on the language design. Positional calls still use the same intrinsic behavior.

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Discard names and consume property order | Wrong names appear to work and may execute unintended code; autocomplete lies | Neither Rust contract calls nor this proposal's named binding supports it | Cheap only by skipping required work | Loses names and forwarding permissions | Reject |
| Permanently ban named intrinsic calls | Inconsistent with declared signatures and user functions | Rust has no named-call analogy; a ban supplies no relevant precedent | Simple but no necessary shape advantage | Conflicts where the intrinsic has an applicable named signature | Reject as final design; fail-closed containment is temporary |
| Attach declaration-identity-based signatures and bind once before intrinsic dispatch | Names in completion agree with selected contracts; wrong names fail before invocation | Rust declaration contracts are a limited analogy; reuse this proposal's BindArguments | Cache compiler-owned signatures; preserve positional fast paths and ICs | Carries names, generic substitutions, references, defaults and phase order consistently | Recommend next, independently of work required |

## Evidence and limits

The engine increment covers variable/literal/property/returned operands, optional/member calls, construction, super, callable views, generic applications and cases, overloads, property and iterator effects, empty/nonempty reference forwarding, and feature-off behavior. Static checking must not enumerate arguments to decide their protocol. Stronger static argument inference, complete intrinsic signatures and general callable adapters remain separate audits.

Primary prior-art references: [Rust call expressions](https://doc.rust-lang.org/reference/expressions/call-expr.html), [ECMAScript CopyDataProperties and iterator operations](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-copydataproperties), and [ECMAScript argument-list evaluation](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-argument-lists-runtime-semantics-argumentlistevaluation). Rust has ordinary positional operands, not this proposal's named object-spread protocol; its analogy is deliberately limited throughout.
