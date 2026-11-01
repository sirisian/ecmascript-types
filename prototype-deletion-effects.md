# Property deletion and exposed prototype effects

This is the thirteenth milestone-5 increment of the [eval and stable names plan](eval-name-stability-plan.md). It follows [invocation-adapter identity](invocation-adapter-origins.md); the [implementation record](prototype-deletion-effects-implementation.md) gives validation and limits. Milestones 5 and 6 remain open.

## Problem and recommendation

A callable identity is not a permanent fact about a property lookup. Deleting an own method can expose another method or getter on the prototype chain without replacing the prototype itself. Track that possible removal and reconsider dependent later lookups. Keep an already captured value intact, and do not invent a getter call at the deletion.

```javascript
// Script 1
const target = Object.defineProperty;
target.call = Function.prototype.call;
Object.setPrototypeOf(target, {
  call() { globalThis.f = () => 1; }
});

// Script 2
function f(): string { return "old"; }
delete target.call;
target.call(null, {}, "x", {value: 0});
const n: number = f();
```

The later call obtains the inherited replacement. Keeping the deleted own adapter as the only candidate manufactures a string-versus-number Early Error. The number receiving boundary should instead check the actual result after the replacement has run. The same gap occurs with a deleted reflective namespace method, an intermediate prototype property, or an indexed property whose removal exposes an inherited getter during `apply` argument-list consumption.

This is a dependency problem, not a change to deletion semantics or a new callable type. The required observations are:

- An ordinary deletion removes an own property; it does not obtain that property's value or invoke an inherited getter.
- A later lookup can acquire a different value and can run user code.
- A deletion cannot affect a callable already captured before it, including an argument-side deletion after callee capture.
- A failed removal cannot reveal a prototype through a positively established non-configurable own property.
- Unknown keys and unavailable prototype evidence cannot certify that a later lookup is effect-free.
- Existing receiving boundaries, typed-member deletion restrictions, prototype obligations and reference liveness continue to apply.

## Question 1: where does the effect belong?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep trusting the original own value after deletion | Completion still shows a familiar method while analysis silently assumes the wrong implementation; a wrong guess rejects valid code | Rust function-item identity identifies a value, not a mutable JS lookup route | No invalidation cost, but an invalid optimization | Fails the replacement example | **Reject** |
| Treat deletion as reading or invoking the revealed property | Errors and side effects appear to move earlier; deleting a getter becomes surprising | Rust dropping a value is not a precedent: JS deletion does not call a property destructor | Adds unnecessary analysis edges; executing a probe would add observable work | Conflates removal with later access and can invent getter execution | **Reject** |
| Treat every deletion as an immediate ambient global effect | Unrelated cleanup loses useful diagnostics even with no subsequent access | Rust mutation does not automatically invalidate every unrelated function value | Cheap broad invalidation; poor specialization and cache reuse | Conservative in some cases, but misplaces an inherited effect before an intervening capture | **Reject** as the general rule |
| Record possible removal and apply its consequence at dependent later lookup | Captured values keep their meaning; later reads follow ordinary JavaScript; autocomplete requires no new API | Rust's value identity versus a mutable storage location is the useful distinction, though Rust has no JS prototype lookup | Allocation/key dependencies can feed existing shape and prototype guards | Keeps removal, lookup and invocation separate while retaining unknown alternatives | **Recommend** |

An exotic deletion can itself run user code; that effect stays at the deletion. Effects from an inherited getter belong to the later lookup. Effects from an inherited callable data value belong to its invocation. The target and key-expression evaluation and key conversion retain their existing positions. None of these distinctions licenses a checking-time getter or trap call.

## Question 2: what should change in the value model?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Erase the property's old candidates everywhere | Earlier aliases or calls can lose valid evidence after later cleanup | Rust copies of function values remain independent of later changes to their source storage | Simple destructive update, but loses history needed by earlier reads | Fails capture and conditional/deferred deletion cases | **Reject** |
| Merge all deleted names across all objects | Same-named properties on independent objects interfere | Rust field identity includes its containing value; a shared spelling is insufficient | Very cheap global set, excessive invalidation | Cannot preserve allocation-specific evidence | **Reject** |
| Execute a complete heap simulation during checking | Appears precise until getters, proxies, loops and arbitrary calls intervene | Rust's static reasoning does not justify executing arbitrary JavaScript | Unbounded analysis and possible reentrancy | Can alter effects and completion timing; cannot solve general execution statically | **Reject** |
| Track allocation, key, contributing removal and dependent access positions | Unrelated keys and objects retain diagnostics; relevant later lookups reflect possible inheritance | Closest to a dependency analysis; Rust does not supply a ready-made prototype-chain model | Finite graph summaries with property/prototype dependencies; stronger abstract-state precision can be added | Preserves earlier values and adds reachable inherited alternatives without claiming removal certainly succeeds | **Recommend** |

The prototype retains historical may-flow values and adds a possible exposed-prototype route. It does not retract old values or claim to know the exact heap at every point. Known native prototype links and bound-function prototype flows can supply inherited candidates. A matching deletion on an intermediate prototype participates too. If a source aggregate has no established prototype summary, the later affected lookup becomes opaque; absence of a recorded edge is not proof of absence of inheritance.

The occurrence check is against the lookup that contributes the callable value. In a simple sequential region, deleting after that lookup does not retarget the captured callable. Deferred bodies and loops require conservative ordering when stronger evidence is unavailable. Unknown keys may match any key of the same aggregate, not every aggregate in the source.

This is the recommended semantic dependency structure independently of implementation effort. More precise branch, heap and invocation contexts can improve the implementation. Analysis exhaustion must withdraw dependent evidence consistently; a resource budget is not a normative choice of error phase. Completing the plan's positive static-evidence model remains a separate milestone-5 obligation.

## Question 3: which negative facts are sufficient to retain evidence?

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Trust every absent own slot or unchanged prototype pointer | A deletion looks harmless although a getter is newly reachable | A Rust function-item proof says nothing about an absent JS property | Cheap snapshot comparison | A stable prototype pointer does not imply stable lookup | **Reject** |
| Make all post-deletion accesses opaque | Safe fallback, but deleting an own copy of an original adapter needlessly loses evidence | Rust retains an identified value when it is still established; no need for universal opacity | Simple, but discards known ordinary paths | Sound loss of precision, not the best finished behavior | **Reject** as the general design |
| Freeze properties/prototypes to keep proofs valid | Breaks ordinary mutation and instrumentation; exposes analysis requirements to users | Rust's different object model is not a compatibility argument for freezing JS objects | May simplify guards by changing observable behavior | Violates the proposal's ordinary-JavaScript compatibility boundary | **Reject** |
| Require positive, operation-specific facts and invalidate their dependencies | Non-configurable own methods still hide prototypes; a revealed original adapter keeps its summary; unknown paths retain checks | Rust compiler-known identity is a useful analogy only after the actual value is established | Ordinary descriptor/prototype checks, IC dependencies and builtin identity tests | Distinguishes guaranteed hiding, known inherited values and unavailable evidence | **Recommend** |

The prototype uses a positively established native non-configurable descriptor to exclude exposure through that property. It does not infer that every attempted deletion succeeds. If the revealed callable is an established original intrinsic, the existing identity summary applies. Accessors, exotic chains and unknown implementations retain their possible effects. Broader proofs about source-created descriptor attributes and arbitrary relinking remain future precision work.

## Evaluation and engine consequences

Only the final reference is deleted. `delete (object.member)` does not read `member`. In `delete object?.nested.member`, obtaining `nested` can run a getter before deletion of `member`. A non-reference expression such as `delete (0, object.member)` evaluates the member read but does not remove the property. A proven null optional base skips the property operation. Preserve the original receiver along an inherited lookup, and preserve abrupt completion and argument-skipping behavior if a revealed getter throws.

As illustrated by V8's [property and element representations](https://v8.dev/blog/fast-properties), deletion can change an object's shape or element representation in a production engine; it can invalidate an own-property inline-cache entry and expose a prototype-chain dependency. Reuse those existing mechanisms. A stable prototype pointer alone is insufficient because an intermediate own property can disappear. Private proof metadata need not add public fields, freeze objects or alter their observable shape transitions. Guard failure must continue with already evaluated operands in the same execution, not replay the deletion or getter. No production-engine speedup is claimed from this interpreter implementation.

The runtime deletion restrictions on typed fields/elements and required interface members are unchanged. Static effect withdrawal does not grant permission to remove them. Likewise, discovering a callable on a prototype neither grants reference authority nor revives a reference whose location was invalidated. The receiving boundary still rejects a replacement's incompatible result. The broader validation gate retains nominal-prototype and reference/liveness tests alongside the new effect tests.

## Remaining work and references

The [following relinking increment](prototype-relink-effects.md) separates original helper invocation on proven ordinary targets from the later effects of a changed chain, including bound prototype capture.

This increment covers supported direct, parenthesized, computed and optional deletions and original reflective deletion helpers, including borrowed/bound routes. It is not a complete proof for arbitrary prototype relinking, every descriptor transition, callbacks or overloaded-index effects. Packed argument/key precision and the larger static-consumer audit also remain open. Milestone 6 retains its integrated conformance work and recorded baseline test/build blockers.

[ECMAScript OrdinaryDelete and OrdinaryGet](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinarydelete) establish the distinction between own-property removal and later inherited lookup. [Rust function-item types](https://doc.rust-lang.org/reference/types/function-item.html) motivate keeping an established implementation identity separate from the location through which a value was obtained. These comparisons do not import a Rust object model or a new public effect type into this proposal.
