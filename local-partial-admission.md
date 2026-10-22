# Partial registration and interface admission

A partial class extends an already declared class. Its source contract can inform checking and autocomplete, but that information does not execute the partial statement or install its methods. Interface admission must distinguish those two facts.

```js
interface Has<T: type> { get(): T; }
class E implements Has.<E> {}
partial class E { get(): E { return this; } }
```

This nongeneric primary is invalid. Its interface obligation is due when the primary constructor is completed, before the later partial can run. Moving the missing method into the primary makes the declaration valid. Omitting `implements` and installing the partial before a later structural use is another available design; it does not establish the nominal interface claim.

A generic family has a different completion point:

```js
interface Has { get(): string; }
class E<T: type> implements T {}
partial class E { get(): string { return "ok"; } }
const value: Has = new E.<Has>();
```

The partial is registered before this application. Its matching members are evaluated on the in-progress application constructor before interface admission. An application attempted before registration fails; if caught, a later attempt after successful registration can succeed. A failed attempt is not a successful specialization cache entry.

## Timing decision

A valid direction must preserve ordinary statement and computed-key evaluation order, lexical availability, actual declaration identity, admission before completed-contract publication, existing module boundaries, failed-attempt effects, successful cache identity, and reference liveness. Source metadata alone must not be treated as an installed member.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep registration and admission at their existing execution points | Split nongeneric implementations need a complete primary; completion may describe extensions without promising they have executed. A wrong timing assumption fails at admission. | Rust requires a complete trait implementation, but its impl items are not effectful JavaScript registration statements. C# partial types have compilation-wide combination. Neither directly models this timing. | Reuse declaration summaries and per-application verification; no additional object states or property inline-cache modes. Existing extension installation retains its ordinary invalidations. | Preserves all required boundaries, including effects and liveness. | Recommend. Explain the timing distinction explicitly. |
| Hoist or assemble all same-source partials into primary creation | Convenient splitting, but moving a statement changes when its keys and captures are evaluated. Autocomplete cannot reveal those reordered effects. | C# combination is the closest analogy; Rust also treats implementations as items. Neither supplies ECMAScript's dynamic key and lexical-read ordering. | May give a stable initial shape, but requires grouping, dependency analysis and cycle rules. | Violates existing registration and availability rules unless those rules are redesigned. | Reject for this proposal's existing partial semantics. |
| Publish the nominal claim immediately using future source summaries | Completion promises members that an escaping constructor or instance does not yet have. Failure may occur far from the declaration. | Rust does not validate a trait impl merely because another item could later repair it. | Cheap initial admission, but unsound optimized assumptions and cache evidence. | A future declaration is not installed member evidence; failure can leave a published invalid claim. | Reject. |
| Defer nongeneric admission until the end of the declaration group or module | Splitting works, but early uses need new restrictions and errors depend on group boundaries. | C# has a compile-time group; Rust has no analogous incomplete runtime constructor. | Requires incomplete-constructor states, guarded uses and cache/inline-cache handling for transitions. | Needs new rules for constructor escape, static effects, cycles and captured references. | Reject; not merely an alternative implementation of current semantics. |
| Hoist only literal-key, apparently effect-free partials | Similar-looking declarations have different timing; changing a key expression can break admission. | Neither Rust's impls nor C# partial types use this spelling-dependent split. | Two installation paths and dependency analysis; shape benefits only on the special path. | Captured environments and declaration activation still matter even without a computed key. | Reject; syntactic convenience does not establish safe evaluation or liveness. |
| Reject every partial contribution as interface evidence | Simple error story, but rejects already registered methods on generic applications and bases. | Rust allows implementation items separate from the type definition; the analogy supports using actual declarations. | Simple but discards valid compiled contract evidence. | Contradicts the existing admission rule for applicable partials. | Reject. |

The [Rust Reference](https://doc.rust-lang.org/reference/items/implementations.html) distinguishes inherent and trait implementations and requires all non-default associated items of a trait implementation. Its item model supports complete declaration evidence, not executing a later JavaScript statement early. [C# partial types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/partial-type) split one type across definitions in a project. That is useful prior art for source organization, but not a direct model for this proposal's runtime class extension.

Choose the existing execution boundaries independently of implementation effort. A superclass partial registered before derived class completion can contribute evidence. A later superclass partial cannot repair a failed derived declaration. A top-level published source summary is checking information, and is insufficient on its own in both cases. Computed keys execute only at their ordinary installation point, once per attempted generic application; cache hits do not repeat them. A failed attempt retains ordinary effects already performed. No getter, method or sample constructor is invoked to discover declaration evidence; ordinary later value admission has its own existing behavior.

## Retaining an open target for checking

The checker must retain `implements T` as an obligation in the symbolic family contract. After argument substitution, `E.<Has>` has that declared refinement for checking a destination, argument or result. Its runtime application must still establish that the target is an interface and verify the actual required declarations before successful publication. Retaining the symbolic obligation does not install a member or publish a completed runtime contract.

| Direction | Ergonomics | Prior art | Performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Drop an unresolved target | Direct typed use fails while an `any` alias succeeds, encouraging erasure. Autocomplete loses a declared relationship. | Rust retains generic trait obligations; its fixed trait paths differ from arbitrary runtime interface targets. | Saves little metadata and repeats work at runtime. | Loses a relationship that argument binding can establish. | Reject. |
| Treat every application as implementing every potential target | Hides wrong guesses and promises unrelated interface members. | No comparable blanket validity in Rust. | Cheap but invalid optimization evidence. | Bypasses target identity and admission. | Reject. |
| Retain the symbolic target and substitute actual arguments; keep runtime admission | Direct and erased applications agree when valid, while invalid member sets still fail at application. | Rust's retained generic obligations support the principle, not the exact syntax or timing. | Reuses existing type substitution and nominal hierarchy lookup; no object shape or property-cache changes. | Preserves target identity, actual declaration evidence, failure cleanup and independent liveness. | Recommend. |

The implementation repair retains the open target in the checking record. Runtime completion already replaces that record's targets with actual resolved interfaces and verifies required methods, properties, operators and call capability. An unrelated destination remains invalid. `implements T` alone supplies no required member and cannot turn a non-interface argument into an interface.

## Follow-up: closed targets on generic families

A fresh related witness is still rejected before any module body:

```js
interface Has { get(): string; }
class E<T: type> implements Has {}
partial class E { get(): string { return "ok"; } }
new E.<uint8>();
```

The target is closed, so its declaration/availability checks remain due at family publication. That does not itself complete a class application. The next implementation task is to distinguish those target checks from member evidence supplied by applicable partials, including conditional contributions. This witness is retained as an unresolved implementation observation, not a passing conformance expectation.

| Direction | Ergonomics | Prior art | Performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Reject every primary lacking its own complete closed-target implementation | A harmless change from `implements T` to a fixed interface prevents splitting even when installation precedes use. | Rust's complete impl requirement does not imply all members must occur in the type's primary body. | Simple, but rejects valid available declarations. | Ignores applicable partial evidence at application. | Reject as the general rule. |
| Delay every closed-target check until application | Avoids the false rejection but hides invalid or unavailable targets in unused families. | Rust retains declaration checks independently of monomorphization. | Defers useful reusable checks. | Violates the existing closed-target availability and failure-order commitments. | Reject. |
| Keep closed-target checks; use established partial evidence and retain unresolved application obligations | Completion reflects available contracts; unavailable or incompatible contributions still fail at the required boundary. | Rust's separate well-formedness and generic obligations support the distinction; dynamic installation remains specific to this proposal. | Summaries plus per-application proof; no new instance layout or inline-cache state. | Can preserve target availability, applicability, member evidence, captured identity and liveness together. | Recommend; implementation and conditional coverage remain the next focused task. |

This recommendation is based on semantics, not on how small the repair is. It does not authorize early execution or relax already decidable errors. The nongeneric witness above remains invalid under every application-evidence repair that preserves the selected timing rule.
