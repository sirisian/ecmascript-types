# Completing the origin-evidence and enforcement audit

This closes the implementation decisions in milestones 5 and 6 of [the plan](eval-name-stability-plan.md). The [validation record](milestones-5-6-validation.md) is the authority for completion status, exact bases, test counts and remaining baseline or environment failures. Earlier increment records are historical; they do not each create another required language feature.

## The language direction

Keep the separation already chosen by the plan:

1. A declaration supplies a contract in its declaring environment.
2. A value observation supplies evidence about a particular evaluated value at a particular point.
3. An effect can withdraw dependent evidence without erasing an independently established contract.
4. Runtime admission consumes the actual value or reference already evaluated. It does not repeat lookup, invocation, conversion or dereferencing.

The remaining audit uncovered a systematic issue: an inclusion graph can contain no candidate for an unsupported expression, or contain a known candidate alongside an unsupported contribution. Neither situation proves that an operation cannot enter user code. More individual intrinsic summaries cannot solve that issue by themselves. Positive summaries need a complete-input condition and an explicit unknown fallback.

This is a finite analysis, not an obligation to infer the effects of every possible program. The mandatory transparent cases remain those in the plan. Additional precision must preserve their results, evaluation order, independently enforced contracts and reference rules.

## Decision 1: what closes an effect summary?

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Treat no collected candidate as no effect | Attractive completions until an obscure expression makes a valid program fail before execution | Neither ECMAScript evaluation nor Rust's type discipline supplies this implication | Cheap analysis; incorrect optimization dependencies | Missing information can become a false negative judgment or unsafe elision | **Reject.** The audit reproduced this with coercion, tagged calls, decorators and logical-expression callees. |
| Maintain only a list of known dangerous spellings | Familiar forms work; aliases and new protocols unpredictably change diagnostics | ECMAScript invokes hooks through many syntactic forms; Rust's alias rules are not a JavaScript spelling analysis | Cheap initial screen, recurring maintenance cost and missed effects | A new or indirect effect path silently escapes the screen | **Reject as the deciding rule.** A conservative screen can still supplement a positive proof. |
| Require complete interprocedural inference | Excellent hypothetical completions; compiler-dependent failure boundaries in practice | Rust bounds and borrowing constrain programs differently; they do not provide complete JavaScript effect inference | Unbounded reachability, recursion and dynamic code defeat predictable compilation | Cannot provide a complete portable decision procedure | **Reject**, as Q6 already did. Work required is not the reason for rejection. |
| Execute a getter or callback while checking | An editor could appear precise while running the program unexpectedly | Neither language licenses replay of arbitrary runtime effects for checking | Executes uncontrolled work and changes observable state | Violates evaluation count, abrupt completion and reference lifetime | **Reject.** |
| Add a required public purity/effect annotation | Makes uncertainty explicit, but imposes a new contract and annotation burden on existing APIs | Effect systems are relevant prior art; Rust shared references alone do not imply purity because interior mutability exists | Can improve dependency tracking and IC assumptions if separately soundly enforced | Requires a separate language design and interoperability rule; does not repair existing unannotated paths | **Reject as this repair.** A future independent proposal is possible. |
| Require a summary for every contributing alternative; retain effects otherwise | Stable transparent cases stay useful; uncertain completions remain conditional and wrong guesses fail at ordinary boundaries | ECMAScript's captured operands are the direct model; Rust contributes the distinction between binding immutability and mutation reachable through a value | Bounded compile-time analysis; ordinary tag, identity, descriptor and prototype guards can support fast paths without wrappers or new instance shapes | Unknown is never purity. Missing accessors and empty internal carrier slots remain distinguishable from unknown program values | **Recommend and implement.** |

The Rust comparison is deliberately limited. `UnsafeCell` and the safe abstractions built on it allow mutation through shared references. Thus even Rust's stronger reference discipline is not an analogy for treating a JavaScript immutable binding or readable signature as an effect-free implementation. The proposal does not import Rust's ownership model.

## Operation audit

| Operation family | Evidence or conservative behavior | Evaluation invariant |
| --- | --- | --- |
| Ordinary, optional, bound and adapted calls | Existing callable identities and argument/return flow; incomplete alternatives retain an effect | Capture the callee and receiver before arguments; no second call |
| Logical-expression and named-argument values | Both possible logical results contribute; named syntax forwards its value, and unavailable argument assignment retains an effect | A known alternative cannot hide an unknown one |
| Tagged templates | Follow the actual tag and substitutions; the template array occupies the first argument | Tag lookup and substitutions run once; substitutions are not untagged string coercions |
| Reads, stores and deletion | Existing ordinary descriptor/prototype summaries; unknown bases retain effects; boxing may expose primitive-prototype accessors | Store effects follow RHS evaluation; deletion does not invoke a deleted accessor |
| Global accessors | An ordinary value-position global read/store can invoke a hook even without an explicit call | Type-position names are not ordinary global value reads |
| Coercion and operators | Primitive coercion proofs are separate from operator dispatch: a mixed primitive domain or an object unary operator can still enter user code. Preserve the intrinsic own-operand pair; otherwise retain effects | Do not invoke coercion during checking or replay operands |
| Construction, superclass access and decorators | Retain receiving admission and effects where implementation/lookup proof is missing | Actual constructor, receiver and evaluated class family remain authoritative |
| Enumeration, rest, spread and iteration | Existing positive summaries plus complete-input fallback | Iterator acquisition, next, closing and pattern defaults retain their order |
| Suspension and disposal | Retain possible reentry and hook effects; preserve independently proven delegation/input obligations | Registration and disposal are separate points; resume does not revive a reference |
| Implicit admission at bindings, members, parameters and results | Retain conversion effects and possible installed backing obligations; do not carry untyped-allocation facts through an unproved admission | The admitted result and the effect of admitting its original input are separate |
| Typed conversion, membership and matching | Retain possible conversion/refinement/protocol effects | The selected target and original location remain fixed |

Deferred code and its execution point are separate. An operation inside an uncalled local body does not precede an earlier top-level capture. The finite callable graph projects that operation to known invocation sites, recursively through local callers; keyed global writes use the same scheduling rule, and effects inside a deferred body are compared with captures within that activation. Callee and argument evaluation, including adapter argument access, precedes reads inside the entered body. A later body operation cannot erase its own earlier capture. Distinct calls, recursion and loops retain possible effects on subsequent activations or iterations, including loops in an intermediate caller. An unknown invocation still carries its own effect, including when a callback escapes this graph. Exhaustion stands the proof down. This is a bounded scheduling summary, not complete interprocedural inference or a promise that a callable is pure.

Implicit admission is also an operation. In particular, a typed structural binding can install storage obligations, and a later member write can run a source conversion. Keeping the original literal allocation as proof of untyped backing loses that effect. Admission therefore retains possible conversion effects and withdraws unsupported backing facts. A successfully enforced primitive target establishes its primitive result independently of any input conversion effect; bare intrinsic primitive admissions remain transparent whether they fit, convert numerically or fail. A direct function literal admitted to a simple shallow function type does not run its body; possible source conversions on its prototype paths remain a separate obligation. A fresh dense array of primitive literals can be admitted without invoking code while still acquiring typed backing, provided prototype paths also exclude source conversions. Pattern admission follows its own default expression. Parameters, written block/concise returns and typed property initializers obey the same rule.

Ordinary truthiness does not invoke coercion hooks. The proposal's overloaded `!` is a separate dispatch: its transparent proof inspects raw operator registration across every possible prototype path, including relinking, without executing an operator. Admission to a bare intrinsic receiving type likewise retains a transparent result where all contributing values and prototype paths exclude source conversions; a failing compatibility check alone does not imply user-code execution.

Positive summaries are equally necessary: an absent accessor is not an unknown getter, primitive object spread does not enumerate primitive-prototype accessors, an ordinary source class has its own data `prototype` property, and an undecorated empty base class has no user constructor body. Other class reads, replacement decorators and nonempty/derived construction retain effects. Literal conversion to an unshadowed intrinsic numeric type preserves its built-in result; custom or unavailable conversion evidence does not. The independent, guarded local generator-delegation proof retains its first-resumption input obligation despite a later potentially effectful resume.

Builtin type spellings are also value-binding hypotheses. An earlier global property or reflective write can replace a builtin without assigning its identifier. The analysis withdraws affected intrinsic hypotheses and rebuilds their dependent flows, preserving operation order and later-activation uncertainty. Rebuilding is capped at four rounds; exhaustion withdraws remaining hypotheses. It does not execute a candidate conversion or substitute the caller's type environment.

The implementation's work budget bounds propagation. Exhaustion withdraws evidence; it never proves compatibility, purity, an absent capability or a live reference. Historical descriptor joins remain conservative. Exact strong updates, complete deferred-body reachability and exhaustive native callback summaries are precision work, not prerequisites for sound unknown fallback.

### Deferred-body scheduling alternatives

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Treat every body effect as preceding every source read | Adding an unused helper can remove completions and move unrelated errors from checking to execution | Declaring a function executes neither an ECMAScript nor a Rust body | Very cheap, but disables useful stable-capture and IC evidence | Erases required independent contracts and source order | **Reject.** The integrated constructor and stored-callable gates exposed this loss. |
| Ignore a deferred effect unless every possible call is discovered | Useful completions become unsafe after escape, native callbacks or later activations | Rust borrowing constrains escape differently; JavaScript permits these routes | Requires whole-program reachability or assumes an unsound closed world | An unknown call can execute the omitted body | **Reject.** |
| Project known calls, preserve ordering within an activation, and retain effects across repeated activations and unknown calls | An unused helper does not change an earlier capture; a later unknown call still invalidates dependent observations | Function definition versus invocation is the close analogy in both languages; this is not a Rust ownership system | Bounded graph traversal during checking, existing callable/guard dependencies, no new object shape or wrapper | Preserves mandatory transparent order, including unused-body diagnostics; repeated calls and loops remain conservative, and exhaustion withdraws evidence | **Recommend and implement.** |

## Decision 2: which family does a nested contract mean?

The recovery gate found two related defects. A returned generic method's parameter constraints could retain an unresolved nominal description even though the enclosing return boundary knew the class activation. The same dependencies must be forwarded through an admitted result, including a signature's own parameter constraints; they cannot be reconstructed from the candidate value. Separately, anonymous family-pattern matching compared source declarations without their evaluated identities. Repeating one class declaration could therefore either reject a method against its own advertised contract or admit a value from a foreign activation.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Identify a family by source syntax alone | Repeated factory calls look interchangeable despite distinct classes | Rust item identity is useful prior art, but a Rust item is not a freshly evaluated JavaScript class | Cheap syntax-keyed cache | Contradicts evaluated family identity and can admit foreign values | **Reject.** |
| Recover the target from the returned value or candidate callable | Often makes the immediate call pass, but silently changes a written destination | Rust's declared bounds do not derive themselves from the value being admitted | Requires inspection and risks unstable cache/IC assumptions | Lets the candidate redefine its obligation; can conflate activations | **Reject.** |
| Erase unresolved nested constraints | Simpler completions conceal a missing obligation | Neither Rust bounds nor this proposal's shallow callable rule means an absent constraint | Avoids preparation at the cost of weaker checks | Unknown is not `any`; a closed constraint must be completed | **Reject.** |
| Complete retained dependencies in their captured context, including constraints and family heads | Returned APIs retain the family the author declared; a wrong activation fails at its argument boundary | Rust's generic scopes are an analogy for declaration ownership, not for this proposal's runtime class evaluation | Reuses declaration/context handles and per-activation specialization caches; no extra per-instance public field | Preserves binders, completed identities and ordinary callable admission | **Recommend and implement.** |

A fixed `any` argument inside a wildcard family pattern remains exact. The existing concrete `A.<any>` erased view retains its ordinary assignability rule. Neither form permits a different evaluated class family.

## Decision 3: how does a computed default read its parameters?

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Substitute Type Records only | `U = T` works while `U = build(T)` encounters an unexpected TDZ | Rust generic scope is more static; it does not justify splitting two readings of the same parameter here | Small bookkeeping cost | Ordinary expressions still read an uninitialized lexical slot | **Reject.** |
| Borrow the current caller's bindings | A same-spelled parameter can unexpectedly change a default | Rust parameters belong to their declaring item, not an unrelated caller | Convenient but invalid cache keys | Breaks lexical ownership, recursion and repeated activation | **Reject.** |
| Evaluate once and cache by source node | Fast repeated completion, wrong after another activation | Rust monomorphization keys include the application; syntax alone is not that key | Small cache, incorrect reuse | Can retain the previous invocation's identity or failure | **Reject.** |
| Bind preceding completed operands in the captured declaration environment for both ordinary and type reads | Computed and direct defaults agree; errors occur when the default is required | Rust motivates declaration-owned parameter scope; runtime Type Objects and TDZ remain ECMAScript-specific | A temporary parameter environment plus existing evaluation budget; reusable completed application keys | Preserves ordering, shadowing and restoration on success/failure | **Recommend and implement.** |

The implementation does not evaluate unused defaults or interpret failed default evaluation as overload mismatch. The existing default/ranking tests exercise both guarantees.

## Decision 4: deep proof traversal

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Rely on host recursion depth | A longer property chain crashes checking rather than producing a language result | A compiler implementation limit is not a Rust or ECMAScript type rule | Low implementation cost, uncontrolled stack use | Host `RangeError` is not the specified failure boundary | **Reject.** |
| Catch stack overflow and assume success | Appears resilient while hiding an incomplete check | Neither language treats compiler failure as evidence | Fast recovery with unsound results | Can bypass admission or suppress a required error | **Reject.** |
| Replace every difficult closed judgment with unknown | Avoids a crash, but makes mandatory diagnostics depend on incidental depth | Both languages distinguish inference from required checking | Bounded but loses required work | Unknown fallback is valid for unavailable optional evidence, not blanket permission to skip closed obligations | **Reject as a general solution.** |
| Traverse reachability iteratively and evaluate deep receiver chains from their base within one query | Long equivalent expressions preserve behavior | Standard compiler worklists are the relevant technique; Rust supplies no special permission to alter the result | Bounded host stack, query-local reuse and no enduring shape/IC change | Preserves the same contribution relation and discards temporary answers before context changes | **Recommend and implement.** The validation includes a 280-member cyclic heap path that formerly exhausted host recursion. A separate experiment combining that depth with hundreds of increasingly nested inferred object types exceeded the environment's memory limit; the worklist change does not claim to bound all structural-type inference costs. |

## Completion boundary

No new public type equality, callable wrapper, conversion priority, class brand or reference permission is introduced. The runtime enforcement proof still requires both the actual stable implementation and its enforced result. A property signature or shallow callable annotation alone supplies neither.

The final validation must distinguish passing acceptance gates, reproduced unrelated baseline failures, time-limited batches and environment-limited build steps. A completed audit is not a claim of proof for the whole interpreter, a production-engine performance measurement, or a complete test262 run.

Primary comparisons: [Rust interior mutability](https://doc.rust-lang.org/reference/interior-mutability.html), [Rust generic parameter scopes](https://doc.rust-lang.org/reference/items/generics.html), and [ECMAScript call evaluation](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-function-calls-runtime-semantics-evaluation). These comparisons motivate the recommendations; the proposal's own normative rules determine its behavior.
