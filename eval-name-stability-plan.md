# Design plan for eval and stable names in checked code

October 31, 2026. Recommended directions accepted for implementation; implementation proceeds in increments.

The first increment implements the receiving-boundary repair and strictness regressions; the second implements declaration type denotation across source texts. The third adds the bounded source-position constructor analysis described in [Class constructor origins in execution order](class-origin-order.md). The original analysis below retains its recorded baseline and observations. The broader consumer/optimization audit and integrated conformance gate remain pending. Validation and patch bases are recorded separately for the [first increment](eval-name-stability-implementation.md), [declaration denotation](class-denotation-implementation.md) and [source-position origins](class-origin-order-implementation.md).

The central defect is a loss of evidence being reported as evidence of incompatibility. The checker can lose the origin of a superclass because an `eval` occurs anywhere in the source, then report that a subclass is not assignable to its base. A stable type contract, the current value of a JavaScript binding, and the evidence that connects a value to that contract need distinct treatment.

The draft's strongest purported counterexample needs correction. Adding an annotation already makes its owning unit strict under this proposal. Examples 10 and 12 therefore do not demonstrate that an initializer or callee was cached. Their results agree with ordinary strict JavaScript. The difference is intentional under the current strictness policy, though it deserves much clearer documentation and editor feedback.

The recommended direction is:

1. Keep the existing annotation-triggered strictness policy. Do not introduce an additional eval-specific scope mode or make all checked code strict.
2. Preserve ordinary runtime lookup, evaluation order, receivers and effects. Reuse checked contracts and permitted type computations, never an ordinary initializer's imagined result.
3. Give a source class declaration a stable type denotation tied to its evaluated declaration instance. Keep its ordinary JavaScript value binding mutable. Apply that distinction consistently across scripts and modules, rather than recovering a declaration's type from whatever constructor currently occupies its binding.
4. Distinguish established compatibility, established incompatibility and unavailable evidence. Only an established violation supports an Early Error. Unknown value compatibility retains the required runtime admission.
5. Replace whole-source eval contamination with evidence attached to lexical bindings, execution points and dependencies. Runtime checks remain the correctness fallback; literal-eval analysis is optional additional precision, not the foundation.
6. Preserve existing storage, prototype and reference-liveness obligations. Neither a class identity nor an optimizer guard supplies reference authority.

The cross-script part of item 3 is a substantive design recommendation: it changes the draft's example 6. It is separated from the immediate false-rejection repair so that its consequences are reviewable. The recommendation does not depend on implementation effort.

## Repository baseline and scope

All three repositories were cloned and pulled with `git pull --ff-only` before analysis. No `AGENTS.md` was found in the checked-out repositories.

| Repository and responsibility | Branch | Inspected commit |
| --- | --- | --- |
| [ecmascript-types](https://github.com/sirisian/ecmascript-types), language design and rationale | `master` | `5800b8e3373283a0cdb0d65ab40a61f2cbef6433` |
| [proposal-runtime-types](https://github.com/sirisian/proposal-runtime-types), normative `spec.emu` | `main` | `6534d50c8926cafb31a9c9f630821b32d734798b` |
| [engine262-proposal-runtime-types](https://github.com/sirisian/engine262-proposal-runtime-types/tree/proposal-runtime-types), executable experiment | `proposal-runtime-types` | `2568d80348434a68b961a6fc477126a1e35b6209` |

The engine is one commit beyond the draft's `0de3c940`. That intervening change implements immutable alias bindings and their publication to later scripts. It does not introduce the strictness behavior discussed here. The relevant strictness implementation already exists at the draft's engine revision.

The proposal is a runtime type system, not erased annotations: sized numeric families, literal adaptation and explicit conversions; typed bindings and storage; nominal classes and structural contracts; value classes with layout and copying; unions, tuples, interfaces, enums and callable contracts; overloads and operators; first-class Type Objects; declaration-bound generics, specialization and constraints; and references governed by runtime liveness. Its broader design includes metadata, decorators, memory views, SIMD, ranges and other extensions. These commitments matter because an apparently local name-resolution fix can change layout selection, overload admission, specialization identity or a borrowed location's treatment.

The formal specification separates checking applicability, strictness, type computation, declaration identity, runtime admission and check elision. The engine reflects those concerns in `type-system/check.mts`, runtime type resolution, declaration instantiation and the ordinary evaluator. The implementation is evidence about behavior and a source of counterexamples; neither it nor the design prose is assumed infallible.

The original analysis delivered this design plan. Subsequent implementation belongs in the three repositories according to the ownership above. The source draft remains a historical record; this plan supersedes its analysis. The author has since directed implementation using the recommended directions.

## Corrections to the rough draft

### The annotation examples change strictness

The design's [Strict Mode and Checked Code sections][design-modes] and specification clauses `sec-typed-strict-mode` and `sec-checked-code` establish different triggers:

| Property | Trigger and extent | Consequence for eval |
| --- | --- | --- |
| Strict code | A written annotation, including `any`, in a Script's own body or a function-like unit's parameters, result or own body; normal downward inheritance; classes and modules retain their usual strictness | Direct eval cannot add `var` bindings to its caller |
| Checked code | Any proposal syntax in the unit, or inheritance from checked code | Additional static judgments apply; this alone does not prevent sloppy eval declaration leakage |
| Ordinary code | Neither applicable trigger | Existing JavaScript behavior remains available |

Classification applies to the whole owning unit regardless of source order. An annotation in a nested function does not make its parent strict. Class-body annotations do not make an enclosing Script strict. Indirect eval and dynamically constructed functions classify their own source.

Consequently:

```js
function make() { return 1; }
function f() {
  eval('var make = () => 2');
  const r: any = make(); // This annotation makes all of f strict.
  return String(r);
}
f(); // "1"
```

The correct base-language control has an explicit strict directive and no annotation. It also returns `"1"`. Replacing the eval declaration with `eval('make = () => 2')` returns `"2"` with the annotation still present. That latter eval changes an existing outer binding; strictness does not prohibit it.

The engine's ordinary lexical initializer path evaluates the initializer, obtains its value and then enforces the annotation. Identifier evaluation uses `ResolveBinding`, and ordinary calls obtain the actual callee through `GetValue`. The reproduced examples and this source path contradict the draft's suggested initializer-cache explanation. They do not prove every contextual-evaluation path free of bugs.

### Unconditional annotation neutrality is not this proposal's contract

The draft requires annotations to change only acceptance. That would also rule out existing literal adaptation, value copying, typed defaults, contextual construction, overload selection, admission effects and implicit strictness. It describes an erasure property this proposal deliberately does not have.

Use a narrower obligation: **with execution mode and specified typed operations held fixed, checking must not replace ordinary runtime name lookup or invent, omit, duplicate or reorder ordinary effects**. An implementation may cache a permitted type computation or a proven pure constant; it may not assume that `make()` denotes the function a pre-body analysis happened to find.

The draft's direction B is therefore too broad in saying that nothing the checker resolves can be reused at runtime. Reusing a resolved contract is essential. B's useful requirement is ordinary evaluation fidelity, not a ban on all retained evidence.

### Strict eval is not effect-free

Strict eval does not inject declarations into its caller, but it can assign reachable mutable bindings, mutate objects, call arbitrary code or throw. The particular string `'1'` is harmless when the call actually invokes intrinsic eval. Strictness alone does not establish that a superclass binding, constructor, prototype or borrowed container remains unchanged.

Conversely, seeing the spelling `eval` somewhere in an unused nested function is not evidence that an earlier class definition used a different superclass. Scope, reachability and timing matter.

### A runtime check is not always a single disposable test

The specification already has [ordinary nominal prototype obligations][spec-prototypes]. A successful exact receiving boundary for its supported ordinary class domain establishes an obligation that later prototype mutations must respect. It is more than an `instanceof` test that can be forgotten after assignment.

This was observable in additional probes: after admitting `new W()` into `V`, `Reflect.setPrototypeOf(W.prototype, Object.prototype)` returned `false`, while inserting an ordinary intermediate object that still reaches `V.prototype` returned `true`. This is dependency maintenance, not blanket freezing. Pure membership tests do not acquire those obligations, and the clause explicitly has a bounded domain.

### Precision cannot mean deciding every safe program

The proposal intentionally rejects some statically invalid code in unused functions. It also has prescribed checking phases, constrained type computation and resource limits. General reachability and the behavior of arbitrary JavaScript cannot be decided completely.

The useful requirement is that a mandatory rejection follows an applicable specified proof, rather than missing metadata or an optimizer's guess. Safe cases covered by the language's evidence rules must not be rejected because an implementation loses that evidence.

## Reproduced behavior

These outcomes are observations at the engine commit above. They are not the proposed semantics. The probes observed body entry through the host observer without prepending a sentinel or changing directive prologues. Each row used a fresh realm, except the explicitly sequenced multi-script cases.

| Draft example | Current result | Interpretation |
| --- | --- | --- |
| 1. `W extends V`, assignment to `V`, then `eval('1')` | `StaticTypeError`, `RT_ASSIGNABILITY`, before body entry | A harmless later eval destroys useful heritage evidence |
| 2. Reassign `V` by direct eval after defining `W` | Same early rejection | Does not establish that the already created `W` changed its base |
| 3. Visible assignment to `V` after defining `W` | Same early rejection | Whole-source assignment screening also loses earlier evidence |
| 4. The replacement through `Function(...)()` | Runs | The local annotation continues to denote the original declaration's class |
| 5. The replacement through indirect eval | Runs | Same result as 4, despite the differently spelled dynamic operation |
| 6. Define classes, replace `V` in a second script, annotate in a third | Runtime `TypeError` in the third script | Later-source lookup uses the replacement; this exposes the contract-denotation inconsistency |
| 7. Replace `V` through `Function` before evaluating `W extends V` | Runtime `TypeError` at the class boundary | The actual base and the declaration-based target disagree |
| 8. Detach `W.prototype` before the receiving boundary | Runtime `TypeError` at that boundary | Actual prototype membership matters independently of the class names |
| 9. Explicitly strict local version with harmless eval | Early `RT_ASSIGNABILITY` rejection | Strictness does not repair the coarse heritage screen |
| 10. Annotated `make()` after eval's `var make` | `"1"` | Expected implicit-strict behavior |
| 11. Unannotated version | `"2"` | Expected sloppy direct-eval behavior |
| 12. `new V()` under `: any` after eval's `var V` | `"1"` | Expected implicit-strict behavior |
| 13. Unannotated construction | `"2"` | Expected sloppy direct-eval behavior |
| 14. Explicitly strict annotated call | `"1"` | Same scope rule as 10 |
| 15. Outer alias `T`, eval's `var T`, then `let t:T` | `"x"` | The annotation makes the function strict; eval does not shadow the outer alias in the caller |
| 16. Eval stores 300 into `uint8` | Runtime `RangeError` | Typed storage enforcement is already independent of how the write is reached |
| 17. Eval replaces a function, result enters `uint8` | Runtime `TypeError` | The actual replacement executes and its result fails admission |
| 18. Eval assigns to a type alias | Runtime `TypeError`, assignment to constant `T` | Changed since the draft: alias immutability is implemented |

Forty-eight focused probes included these eighteen cases and adjacent controls. Particularly useful additional results were:

- Explicit strict JavaScript, with runtime types disabled, reproduced 10 and 12's `"1"` results.
- Replacing an existing function or class through eval produced the replacement's `"2"` result even under `any` annotations.
- An effectful initializer ran once per actual call: two calls returned `"1,2,2"` for first result, second result and call count. An unused function's initializer left the count at zero.
- A function containing only a type-alias declaration remained checked but sloppy; eval's injected `make` was observed. An injected ordinary `T` likewise shadowed an outer alias in value position in a checked-sloppy function.
- An unused function containing eval, even in an unrelated scope, caused an otherwise valid subclass assignment to be rejected.
- Moving a visible `V` replacement after the successful store still caused early rejection.
- An immutable class-expression alias case with eval completed normally. This result alone does not prove that the checker retained complete static ancestry; runtime admission may account for success.
- A stable `type Original = V` alias preserved the original class across later scripts and a replacement of `V`.
- A same-source `new V()` after replacing `V` failed against the original `: V` contract at runtime.
- A further precision case, `const Replacement = class {}; V = Replacement; const v: Replacement = new V();`, was rejected early although the executed construction would select `Replacement`. The proposed origin analysis must cover the constructor operand as well as heritage.

### The implementation path behind the false rejection

In the inspected engine, `collectMutations` in `type-system/check.mts` sets a whole-source `hasDirectEval` flag. `stableClassOrigin` returns no origin for a same-source identifier when that flag is set; it also screens class declarations using whole-source assigned-name sets. `heritageOriginOf` feeds this result into `instanceTypeOf`, where unavailable origin yields no known base. The final assignability query can then treat that incomplete description as an incompatible nominal type.

That is a more specific diagnosis than “eval is conservative.” Withdrawing an origin can be correct. Converting the withdrawal into a closed negative nominal judgment is the problem. Other operations in the checker already distinguish unknown ancestry from established absence; the repair should make the evidence discipline consistent rather than special-case the sample text.

## A model that separates the relevant identities

| Entity | What identifies it | What can change | What it does not prove |
| --- | --- | --- | --- |
| Lexical binding | Declaration plus environment instance; imports resolve to their producer | The value of a mutable binding | Identity of the value currently stored |
| Class declaration template | Source declaration and its lexical dependencies | Its evaluated instances differ across activations | One runtime class identity across all executions |
| Evaluated class or generic family | The particular class evaluation; specialization also includes ordered arguments | Its permitted mutable object state | A source name still contains that constructor |
| Established type contract | Resolved type and retained declaration/activation dependencies | Completion of an unresolved dependency under its defined rules; never silent retargeting | Current value of an unrelated or replaced name |
| Expression-origin evidence | The binding/value origin and assumptions valid at the expression's execution point | Writes, calls, eval, imports, prototype changes and suspension may invalidate assumptions | An enforceable storage contract by itself |
| Prototype obligation | An admitted object, exact target and relevant ordinary prefix | A valid prefix can be replaced while preserving its target | Arbitrary exotic or generic membership outside the clause's domain |
| Borrowed location | Actual backing storage, location and existing liveness state | Relocation, removal, rebinding and scope rules | Continued validity merely because its element type is unchanged |

A binding and its current value are different even in ordinary JavaScript. Class definitions also have an inner self-name distinct from their outer mutable declaration binding. Making a class body strict does not make that outer binding immutable.

At a call, the callee value and receiver are obtained at the language-defined point before the relevant argument effects. An argument that subsequently changes the binding does not make the call re-read its callee. A guard must validate the captured callee, not perform a second lookup after arguments.

### Requirements for an acceptable direction

The following are validity conditions, not a weighted score that allows a safety failure to be offset by convenience.

| Requirement | Meaning |
| --- | --- |
| R1 Ordinary evaluation fidelity | Preserve the specified runtime lookup, selected receiver, evaluation order, abrupt completions and effect count |
| R2 Stable contracts | Once established, a contract does not change because a source spelling later names another value |
| R3 Honest evidence | Unavailable ancestry or origin is not a proof of incompatibility or compatibility |
| R4 Correct phases | Syntax and established type violations retain their specified pre-body phases; unresolved value admission occurs at its actual boundary |
| R5 Scope and activation identity | Respect nearer bindings, TDZ, imports, each class evaluation, captured parameters and suspension |
| R6 Existing type distinctions | Preserve nominal identity, generic arguments, metadata, value/reference storage, exact `any` family entries, overload and conversion rules |
| R7 Durable admission | Preserve the proposal's required storage and prototype obligations, including when a redundant value test is elided |
| R8 Independent liveness | No identity proof, cache hit or failed guard creates or revives a reference, relocates its target or extends permission |
| R9 Legacy behavior | Units that do not opt in retain JavaScript behavior; typed boundaries still enforce contracts when entered from untyped code |
| R10 Portable judgments | Mandatory static rejection follows language-defined evidence rules, not optimization level or an engine's analysis budget |

R9 includes the existing strictness and typed-storage policies when a program opts in. It is not the draft's stronger annotation-erasure requirement.

## Design decisions

Every question below compares the available directions using the same five criteria. “Other” records scope, portability or the reason an alternative loses. Cost of implementing a direction is described where relevant but does not determine the recommendation.

### Q1 What should annotation neutrality mean

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Full erasure neutrality | Simple migration story, but it contradicts what sized types and target-directed construction promise | TypeScript's erasable annotations are closer; Rust's typed execution is not an erasure model for JavaScript | Prevents important representation and contextual choices | Fails existing strictness, conversion, copying and overload commitments | Reject for this proposal; it would be a different type system |
| Let checking substitute whatever it resolves | Apparently precise completions can refer to a function that will not actually be called | Neither ECMAScript lookup nor Rust's resolved dispatch licenses executing a different ordinary callee by accident | Easy constant propagation can be unsound under mutation | Violates R1 and R5 | Reject |
| Ban all reuse of checked results, as a literal reading of draft B | Avoids one suspected bug but repeats work and hides why a contract is stable | Rust and this proposal both retain resolved type information | Repeats contract formation and specialization; can repeat permitted initialization effects | Can violate once-only type/specialization behavior | Reject the blanket ban |
| Preserve ordinary evaluation subject to specified mode and typed semantics | Explains real behavior; the editor must show that an annotation activates strictness | Rust supports typed semantic distinctions; ECMAScript supplies ordinary evaluation order | Retains static contracts, pure folding and legal specialization caches | Meets R1 while preserving the proposal's existing semantics | **Recommend**; replace the draft's neutrality axiom with this rule |

An annotation-induced overload selection is a specified typed operation. Silently calling a stale ordinary function is not. Test the distinction with explicit-mode controls and effect logs, not only annotation removal.

### Q2 Should eval or strictness get a new checked-code rule

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Preserve fully sloppy behavior even in annotated units | Fewer migration effects, but `with`, mapped arguments and injected bindings make static suggestions fragile | ECMAScript permits these behaviors; Rust has no corresponding ambient scope injection | Requires dynamic environment handling and additional contract machinery for mapped aliases | Reopens settled typed-scope and parameter-enforcement issues | Reject as the direction for this plan |
| Require an explicit strict context before allowing annotations | A wrong migration guess becomes an actionable error rather than a changed result | Uses ECMAScript strict mode; Rust offers no directive analogy | Similar optimized execution once admitted | Coherent, but changes where currently valid typed source is legal | Reject in favor of the existing syntactic opt-in: directives are redundant in modules and awkward for isolated functions with non-simple parameters |
| Make every checked unit strict, or only give its eval strict-like declaration scope | Fewer categories, but a range, alias or match can unexpectedly change unrelated runtime behavior; a custom eval mode is harder to explain | Full strictness has ECMAScript precedent; a partial strict-eval mode does not. Rust supplies no runtime-eval precedent | Broader lexical certainty, but still no protection against existing-binding writes | Full strictness is possible; partial strictness leaves mapped arguments and other behaviors inconsistent | Reject: the extra semantic reach does not solve class replacement and exceeds the needed opt-in |
| Keep annotation-triggered strictness and ordinary eval semantics | Annotations are a documented mode opt-in; checked-sloppy code remains possible and must get honest dynamic completions | Directly uses ECMAScript's scope rules; Rust comparison is limited to stable lexical resolution | Typed strict scopes can use static slots; only genuinely dynamic scopes need dynamic lookup handling | Preserves existing commitments without treating strict eval as effect-free | **Recommend** |

The implicit mode switch has a real ergonomic cost: even `: any` can change `this`, arguments mapping and eval scope. An editor should show “strict through annotation” at the unit and explain the trigger when a diagnostic or quick fix depends on it. Hiding that cost is not a language-design solution.

### Q3 What does a class name denote in a type position

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Re-read the current class value at every check or store | One apparent name meaning, but an existing variable's type silently changes; completions can go stale between statements | Unlike Rust's fixed item types; JavaScript variable reads alone do not define a durable contract | Repeated lookup, polymorphic guards and potentially different layouts at one slot | Violates R2 and can violate R6 | Reject |
| Capture the current value separately at each annotation's execution | Each individual binding has a stable type, but identical source annotations can mean different classes within an activation | Runtime contract capture has analogies in dynamic languages; it is not Rust's type-name model | Requires dynamic contract instances and guards for layouts, signatures and specialization | Coherent only with a redesigned type-formation and forward-reference model | Reject as the default: arbitrary mutable reads would conflict with closed type computation and predictable declared signatures |
| Capture whatever is visible at each source-checking boundary, retaining the local/foreign asymmetry | Matches several observations, but splitting one script into two can change a name's type meaning | No useful Rust analogy; this is phase-dependent capture rather than item identity | Cheap local caches, costly and surprising invalidation at source boundaries | Gives stable individual snapshots but lacks one declaration rule across sources | Reject as the language model |
| Introduce independent type and value namespaces | Stable type completion while values remain mutable; same-spelled shadows can surprise users | Rust explicitly separates namespaces; TypeScript classes also contribute type and value entities | Efficient static type resolution | Can be coherent, but changes this proposal's ordinary shadowing and first-class-type lookup story | Reject a general namespace split; do not bypass a nearer ordinary binding to find an outer type |
| Resolve a source class's type through its lexical declaration and evaluated binding instance; resolve ordinary expressions normally | Type positions have stable class/member information; value-position completion reflects replacement uncertainty. The divergence is explicit and explainable | Uses Rust's declaration-identity principle without copying its namespaces or lack of runtime class evaluations; fits existing class-family clauses | Stable contract pointers and family keys; mutable value loads retain ordinary guards | Preserves R2, R5 and R6; extends declaration evidence consistently across source boundaries | **Recommend** |

This is contextual denotation, not a second search namespace. Resolve the nearest lexical binding first. If it is a source class declaration, type position denotes that declaration's class instance; value position reads its current constructor value. A nearer ordinary binding blocks an outer class or alias. That nearer binding is usable as a type only when it independently meets the existing type-expression rules.

A symbolic class reference may be prepared before class evaluation. It must complete to the class associated with that binding instance, without evaluating its constructor or enclosing body early. Completion cannot use the most recently evaluated class with the same source node, and cannot choose the target from the value being admitted.

For a previously evaluated Script-global class declaration, retain its declaration association just as module class metadata retains its origin. A later script must not silently switch `: V` to an unrelated replacement merely because the class's mutable value slot changed. This changes example 6 to accept the old `W` against the original declaration's `V`, assuming the ordinary receiving obligations succeed. `new V()` still constructs the replacement.

`type V` should use the same denotation as `: V`; bare `V` remains an ordinary value read. The distinction must be documented, including reflection of an actual constructor value. Existing immutable aliases provide clear stable names. For an intentionally mutable constructor choice, give the chosen replacement its own stable declaration or pass the Type Object through an already supported generic/contract formation boundary. This plan does not invent a new runtime-dependent annotation syntax or make ambient mutable reads compile-time evaluable.

### Q4 Should class bindings become immutable

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Make all class bindings immutable | Very stable constructor suggestions; existing JavaScript replacement code breaks | Rust items are not assignable locals, but JavaScript classes are constructor values | Fewer binding guards; no protection against every object/prototype effect | Violates R9 for ordinary code | Reject |
| Make class bindings immutable when their declaration is checked | One annotation can disable replacement of an otherwise ordinary class. Imported legacy classes still differ | Similar in spirit to Rust item stability, but checkedness has no Rust analogue | Helps some local sites; does not eliminate guards for mutable producers, decorators or dynamic origins | Possible only if every write route respects declaration-owned immutability | Reject: removes useful value behavior while solving only part of the evidence problem |
| Forbid assignment only from checked callers | Looks permissive until untyped callers replace a supposedly fixed class | Neither Rust item identity nor ECMAScript storage invariants support caller-dependent constness | Still requires guards against untyped writes | Does not make the binding immutable or justify proofs | Reject |
| Keep mutable class declarations; use existing immutable class-expression bindings and aliases when stability is intended | Preserves JavaScript replacement; users can give a stable constructor an explicit `const` name | ECMAScript `const` supplies the direct analogy; Rust's fixed type identity informs the separate contract | Stable paths optimize; mutable paths use normal loads and dependency guards | Works with Q3 and runtime admission without changing reference permission | **Recommend** |

Alias immutability does not force class-binding immutability. A `type` declaration introduces a type-level definition whose value is its denotation. A class declaration also introduces an ordinary replaceable constructor binding. Their roles differ. No outcome should depend on whether the code attempting the write is checked; storage properties belong to the declaration.

The rough draft also overstates what C plus D would make succeed. If `V` is immutable, examples 2 and 3 cannot run successfully: their writes must fail. A `Function`-mediated write in example 7 must fail at that write too if the declaration is actually immutable, rather than reaching the later class-admission failure. Restricting only checked writers would avoid those failures by leaving the binding mutable, which would defeat D's premise.

### Q5 What should happen when heritage or constructor evidence is missing

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Treat missing ancestry as no ancestry | Precise-looking but false errors such as example 1; inherited members disappear | Rust can reject unknown types in its closed static setting, but this proposal has explicit dynamic admission | Cheap analysis; no legitimate optimization benefit from an incorrect negative | Violates R3 | Reject |
| Assume the written `extends V` always denotes the original `V` | Pleasant completions until a dynamic replacement or class-evaluation effect changes the actual base | Rust's statically fixed relationships do not model a JavaScript heritage expression | Can remove checks unsafely | Violates R1 and R6 | Reject |
| Replace the whole class description with public `any` | Avoids false rejections but removes known own members, identity and useful constraints from autocomplete | Rust does not erase known type facts this way; gradual systems distinguish loss of knowledge from a universal value type | Fewer proof queries, more dynamic checks and weaker specialization | Overly broad and dangerous if it changes exact family matching or admitted contracts | Reject |
| Carry explicit unresolved origin/ancestry evidence; defer only the affected judgment | Known own information survives; uncertain inherited suggestions are marked, and a wrong dynamic guess meets admission | Rust motivates precise identity; the dynamic fallback comes from this proposal's runtime type model | Metadata can be compiler-owned; established paths stay cheap, unknown boundaries retain checks | Meets R3 without weakening nominal identity or reference rules | **Recommend** |

Use a three-outcome proof result at the evidence-consuming layer, not an indiscriminate rewrite of every Boolean type relation:

| Evidence outcome | Mandatory behavior | What cannot be inferred |
| --- | --- | --- |
| Compatible, with dependencies established | Accept the static judgment; apply existing elision rules separately | That prototype/storage obligations or liveness checks are unnecessary |
| Incompatible under the applicable contract and admission rules | Report the specified static diagnostic at its required phase | That an optional later optimizer guard can rescue the rejected program |
| Unknown because required evidence is absent or invalidated | Preserve the target contract and the affected runtime admission | Either compatibility or incompatibility; public `any`; permission to perform extra source reads |

Ordinary subtyping of completed Type Records can remain Boolean. The caller must establish that the relevant descriptions are complete enough for the negative conclusion. An independently known mismatch still rejects even if unrelated metadata is unknown. A missing target type is different: a type expression that is invalid, unavailable or non-evaluable retains its existing formation error. Runtime value checks do not make an invalid annotation valid.

This also preserves existing assignability distinctions. Subtyping a value class does not automatically authorize copying a larger subclass into a smaller inline base slot. An exact `any` entry in a family constraint is not a wildcard. Neither rule is relaxed by unavailable origin evidence.

### Q6 How precise should mutation and eval analysis be

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Keep a whole-source eval/assignment flag as the semantic answer | Unrelated unused functions change errors and completions | Conservative compiler screens exist; Rust's lexical analysis does not justify treating unrelated bindings as one | Small analysis cost, widespread lost optimization | Safe only if it produces unknown evidence, never false incompatibility | Reject as the required precision model; usable as a temporary conservative screen |
| Parse literal eval strings and look for assignments | Helps `eval('1')`, but a helper call, nested eval or replaced eval function defeats a textual scan | No close Rust analogue; constant-source analysis is an optimization technique | Extra parsing and effect analysis; no direct instance-shape benefit | A no-assignment scan is insufficient to prove no effects | Reject as the foundation; permit only sound summaries under established callee identity |
| Ignore eval in strict code | Seems intuitive after learning declarations do not leak | Misreads ECMAScript strict eval; Rust has no analogue | Cheap but unsound | Strict eval can still write outer bindings and mutate reachable storage | Reject |
| Require complete interprocedural reachability and effect inference | Ideal completions in theory; inconsistent implementation behavior in practice | Rust uses a different static ownership/type discipline, not a complete analysis of arbitrary eval | Unbounded analysis is incompatible with predictable compilation | Cannot be a complete, portable decision procedure for JavaScript | Reject as a language requirement |
| Use source-ordered, binding-aware evidence with conservative dynamic invalidation | Earlier facts survive later unrelated writes; unknown effects are visible as uncertainty rather than invented errors | ECMAScript supplies order and scope; Rust supplies a limited place/value and declaration-identity analogy | Reuses compiler CFG, environment slots and dependency tracking; fast paths can be guarded | Preserves R1, R3, R5 and R10 with runtime fallback | **Recommend** |

The required analysis should establish a finite baseline of transparent cases: lexical shadowing, immutable aliases, direct assignments and patterns, source order, selected branches, class evaluation order and known captured writes. Unknown calls are effect boundaries for bindings and state they can reach; no claim of purity follows from being indirect or from lacking the text `eval`.

Do not infer behavior merely from the spelling `eval`. Parentheses can preserve a direct eval reference; comma and optional-call forms do not have the same semantics. A call must actually select the intrinsic for direct-eval rules to apply. A shadowed or replaced function named `eval` is an ordinary call. Literal analysis must account for calls, getters, nested dynamic code, abrupt completion and the actual lexical environment, not only assignment tokens.

An optional optimizer may inspect literal source more deeply. It must not make mandatory rejection depend on that effort, execute the string during checking, or report an inner eval syntax error before evaluation reaches that eval. An unexecuted eval body is not an extra outer source unit to reject eagerly.

### Q7 When are runtime guards and checks sufficient

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Use a predicted runtime identity to justify a pre-body error | Most executions may match the editor's guess, but valid executions become impossible | Rust's static proof is not speculation; JIT guards protect executable paths | Cheap prediction cannot recover from code never starting | Violates R4 and R10 | Reject |
| Re-read the name or replay an expression after effects to validate the guess | A getter, receiver or callee can change; failures appear in the wrong order | ECMAScript evaluation order is decisive; Rust supplies no replay permission | Extra calls and reads can destroy IC locality and duplicate effects | Violates R1 and can violate R8 | Reject |
| Retain the actual evaluated operands and perform ordinary admission against the fixed target | Calls and constructions run once; errors occur at the established boundary | ECMAScript operand capture plus this proposal's runtime contracts; Rust has a different static enforcement model | Monomorphic sites can use identity/tag/shape guards; deoptimization handles failed assumptions | Meets the requirements if all independent obligations survive | **Recommend** |
| Deep-wrap every callable and constructor that crosses a typed boundary | Stronger contracts, but wrappers change identity, receiver behavior and stack traces unless separately specified | Contract systems use wrappers; this is not Rust dispatch and not the proposal's shallow callable admission | Allocation, wrapper dispatch and changed IC targets | Changes existing callable and overload semantics | Reject as a remedy for unstable names |

Guards are an implementation strategy for accepted programs. They do not create a source-language proof. The actual selected function retains its own parameter/result checks. A shallow callable contract does not guarantee that a replaced function enforces the former declaration's return type; the receiving boundary cannot be elided on that assumption.

### Q8 Should class membership become a permanent brand or remain tied to prototypes

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Make every class a permanent hidden brand independent of its prototype chain | Stable class suggestions, but objects can satisfy the type after losing the behavior users associate with it | Rust nominal identity is stable, but Rust has no mutable JavaScript prototype chain | Potentially cheap brand tests and specialized ICs | Changes the proposal's existing class/prototype membership model | Reject for this problem |
| Freeze all class-related prototypes as soon as names are checked | Easy predictions, but checking an unused annotation can restrict unrelated objects | No close Rust analogy; JavaScript freezing is an explicit object operation | May simplify guards but imposes broad observable effects | Exceeds existing bounded admission obligations and checking phases | Reject |
| Test once and forget, or silently weaken existing contracts after mutation | Simple implementation; completions promise members that later disappear | Neither Rust's fixed types nor durable runtime contracts support silent weakening | Cheap admission, unsafe later elision | Violates R2 and R7 in the supported exact domain | Reject |
| Preserve current exact prototype obligations and independently handle domains they do not cover | Compatible prototype changes remain possible; invalidating changes fail at their defined operation | ECMAScript determines prototype effects; Rust only motivates preserving established guarantees | Internal dependency maintenance on mutations; no public per-instance field is required | Retains the bounded semantics and R7/R8 | **Recommend** |

This plan does not claim a general solution for all generic, exotic, composite or union membership. Where existing rules lack durable proof, retain their required runtime checks and prohibit extra elision on that basis. A future expansion of prototype obligations needs its own semantic decision; the immediate name-stability repair must not silently expand or discard them.

### Q9 How should declaration evidence cross scripts, modules and generic activations

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Cache by spelling, source node or last evaluated constructor | Simple editor cache, but identical local declarations from different calls become confused | Rust paths identify declarations; runtime JavaScript class activations need an additional distinction | Cheap keys can reuse the wrong layout or specialization | Violates R5 and R6 | Reject |
| Treat an import as an immutable constructor because the importer cannot assign it | Attractive completions hide producer-side replacement | ECMAScript imports are live; Rust imports do not model mutable JavaScript export cells | Missing invalidation can make IC assumptions unsound | Confuses local write permission with producer stability | Reject |
| Infer the destination identity from the incoming value or re-run its producer | Almost anything can be made to match itself; wrong-family errors disappear | Rust infers under declared constraints; reflection of a value does not define its required destination | Extra reads, effects and repeated specialization | Violates R1, R2 and R6 | Reject |
| Retain declaration and environment-instance evidence, publish only at successful initialization, and resolve imports to producers | Captured closures and later sources keep the intended type; unresolved inputs stay visibly unresolved | Extends the declaration-identity principle used by Rust without importing its different nested-generic scope rules | Private slots/weakly held metadata; specialization caches key by family and arguments | Preserves scope, availability, failure recovery and liveness | **Recommend** |

The current specification already requires much of this for class families, immutable bindings and generic captures. Extend it to the class declaration's type denotation across later Scripts. Do not publish future declaration evidence just because checking encountered it. A failed class initialization contributes no completed class; a successful earlier initialization remains real after a later statement throws.

Direct eval receives its actual lexical generic environment. Indirect eval and `Function` use their global source environment, not the caller's generic stack. Escaped closures keep their originating activation, and a suspended activation must recover its own evidence when resumed. A nearer binding, including an uninitialized one, blocks a same-spelled outer fallback.

### Q10 How do name-stability proofs interact with references

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Treat stable element/class types as proof that a reference remains live | Autocomplete looks reassuring while a removed or relocated element can be accessed | Rust separates reference validity from the referent's type; its borrow checker is different from this proposal's liveness rules | Skips checks that can be necessary | Violates R8 | Reject |
| Let guard failure reconstruct or refresh a reference from the current name/index | Appears forgiving, but the program accesses a different location after rebinding or removal | No Rust precedent for silently replacing a borrow; ordinary JS names do not identify old array storage | Requires re-resolution and can replay getters or proxies | Changes location identity and revives invalid references | Reject |
| Carry the original location and existing liveness obligations independently of type-origin evidence | A wrong guess fails at the specified access; completions can distinguish type from availability | Rust provides the limited distinction between place/type and borrow validity; runtime enforcement remains this proposal's choice | Existing generation/relocation state and access checks; no new borrow counting | Preserves R8 without changing alias or storage contracts | **Recommend** |

If eval or a callback relocates storage between reference acquisition and parameter binding, the final binding/access checks still apply. If a reference binding is redirected, an alias follows the proposal's existing rebinding rules; a cached old origin cannot override them. Generic arguments are values, not newly borrowable storage. Neither the target contract nor a declaration metadata slot supplies a location for `ref`.

### Q11 What should tooling and diagnostics promise

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Always display the original class as the current constructor and all inherited members as certain | Convenient until a wrong guess calls another constructor or fails at runtime | Rust can give stronger static promises because its names and types have different rules | No engine advantage | Misrepresents the evidence and encourages unsafe elision assumptions | Reject |
| Erase every affected suggestion or report a hard error whenever proof is missing | Users lose known own members and valid code; workarounds spread `any` | Rust's errors arise from its required static discipline, not a template for this runtime system | Simple editor logic, unnecessary runtime dynamism | Confuses lack of evidence with invalidity | Reject |
| Distinguish declared contract, current-value origin, uncertain inherited information and implicit strictness | Users see which `V` a type position means and why a value call may differ; wrong guesses have explainable costs | Rust supports declaration-linked navigation; TypeScript demonstrates type/value presentations but has different runtime guarantees | Uses existing compiler metadata; optional richer analysis need not affect runtime | Aligns editor claims with Q3/Q5 and phase-correct diagnostics | **Recommend** |

An early diagnostic should identify the established source and target and the failed rule. A dynamic failure should identify the fixed target contract and the actual failing boundary; use the existing diagnostic schema rather than inventing mandatory new public fields. Optional editor warnings about dynamic origin must not become an undocumented acceptance rule. Display names alone are insufficient when two evaluated classes share a source spelling.

## Recommended semantics in representative programs

These are target outcomes for implementation. They are deliberately distinguished from the observations above.

### Harmless eval does not disprove an existing subclass relationship

```js
class V {}
class W extends V {}
const v: V = new W();
eval('1');
```

Accept. Static evidence may establish the relationship at the earlier heritage point; otherwise the receiving boundary decides it. The trailing eval does not justify an Early Error. Required nominal obligations still attach even if a redundant membership test is optimized away.

### Rebinding a name does not retarget its declared type

```js
class V {}
class W extends V {}
V = class {};
const v: V = new W();
```

Accept against the original declaration's `V`, subject to the ordinary class boundary. `W` captured the original superclass when its heritage was evaluated. Replacing the outer `V` value does not replace either class identity or the annotation's target. The same rule should hold if replacement is performed by strict direct eval, an indirect global operation or a later script.

```js
class V {}
V = class {};
const v: V = new V();
```

The value expression constructs the replacement; the annotation denotes the original declaration's type. Reject statically when the actual replacement origin and incompatibility are established under the specified rules; otherwise throw at runtime admission. Do not call the original constructor to make the annotation pass.

This is the principal ergonomic price of preserving mutable values and stable type declarations. Prefer clearly named stable constructors when a program intends to use both identities. Do not disguise the difference in autocomplete.

### A replacement before heritage affects the created subclass

```js
class V {}
Function('V = class {}')();
class W extends V {}
const v: V = new W();
```

The `extends` expression observes the replacement in this Script-global example. It does not observe a function-local binding through `Function`, and the same text inside a module has different reachability. If the actual inherited class does not satisfy the original target contract, admission fails. Static failure requires established effect/origin evidence; the mere fact that a function call could change something is an unknown result, not a negative proof.

The draft's direction E is incomplete if it analyzes only writes to the base before `extends`. The constructed `W` value can also be replaced, constructors can return different objects, decorators can replace classes, and prototype links can change independently. All dependencies of the particular judgment matter.

### Later source texts use the same declaration rule

```js
// Script 1
class V {}
class W extends V {}

// Script 2, same realm
V = class {};

// Script 3, same realm
const oldInstance: V = new W();
```

Recommend acceptance against Script 1's class declaration. This is a change from the observed runtime error. A stable alias already provides this behavior today:

```js
// Script 1
class V {}
class W extends V {}
type Original = V;
// Later, even after V is replaced:
const oldInstance: Original = new W();
```

The purpose of the proposed extension is consistency of the declaration spelling, not a change to Type Object equality or an ability to look through lexical shadows. An earlier successful class remains its own type even if another value occupies its mutable constructor binding.

### Strictness and declaration injection remain separate from reassignment

```js
function make() { return 1; }
function f() {
  eval('var make = () => 2');
  const n: any = make();
  return n;
}
```

Keep the current result `1`. Change `var make = ...` to assignment `make = ...` and the result becomes `2`. A version using proposal syntax that does not trigger strictness can retain sloppy eval injection. The checker must handle that uncertainty without treating a captured outer declaration as the runtime callee.

### Callee capture survives argument effects

```js
function first() { return 'first'; }
let selected = first;
function change() { selected = () => 'second'; return 0; }
const result: any = selected(change());
```

The ordinary call has already captured `first` before evaluating `change()`. The result is `'first'`. A guard or slow path must not re-read `selected` afterward and call the replacement. The same discipline applies to a method's receiver, `new`, argument spreads and abrupt completion.

### Prototype and reference obligations survive evidence loss

For the supported ordinary exact-class domain, changing a prototype before admission can make admission fail. Changing a prototype after admission must preserve the already acquired target-reaching obligation. A compatible insertion remains legal. Names and declaration records do not replace these rules.

If an array reference is acquired and a subsequent callback removes its element or relocates the backing allocation, the reference remains invalid even if every element still has the same nominal type. Guard failure must not retry acquisition at the same numerical index. Source-order origin evidence and storage liveness have separate validity conditions.

## A specification and implementation shape

### Evidence and contract formation

The specification should describe the following steps without requiring these particular internal record names:

1. Resolve the type-position binding by ordinary lexical/module rules. Preserve its declaration and environment identity. A nearer blocker is final; do not search an outer type namespace.
2. Form the target contract using the applicable declaration, class-family, alias and type-evaluation rules. Retain symbolic dependencies where those rules permit forward completion. Reject genuinely invalid type expressions at their established phase.
3. Obtain source-expression evidence at the expression's execution point. Separate its declared/admitted contract from predictions about a mutable value or an incomplete ancestry chain.
4. Ask whether the evidence establishes compatibility, establishes an applicable type violation, or leaves the judgment unresolved. Preserve useful independent facts.
5. For accepted execution, evaluate ordinary source in its normal order and retain the actual results/locations. Complete available target dependencies from their owners, never from the incoming value.
6. Perform required runtime admission and acquire any independent obligations. Elide a test only under the complete stability rule; do not elide observable obligation acquisition, required conversion or liveness checks merely because nominal types appear compatible.

A single `stable` Boolean is unlikely to describe every needed fact. For example, a declaration identity can be stable while the mutable binding's current constructor is unknown; a callee can be captured while a result check remains necessary; nominal membership can be protected while an array location is dead. Represent the dependencies relevant to each proof rather than promoting one fact into all of them.

The model must also distinguish unfinished evidence from a completed unknown answer. A recursive type's coinductive relation is not permission to fill an unresolved runtime class identity with another activation's constructor.

### Invalidation boundaries

| Operation or boundary | Evidence to reconsider | Evidence that need not be discarded merely for this reason |
| --- | --- | --- |
| Direct assignment, destructuring or iteration write | Current value origin for the exact destination binding | A distinct shadowing binding; already established declaration contracts |
| Strict direct eval | Reachable mutable binding and heap facts; effects through called code | The fact that eval declarations do not leak; established immutable lexical identity |
| Sloppy direct eval | The above plus affected dynamic binding resolution/injection | Lexically protected declarations outside the injection scope |
| Indirect eval or `Function` | Reachable global state and effects through globally reachable code | Inaccessible function-local slots solely because their names match |
| Unknown callback, getter, conversion or constructor | Reachable mutable dependencies and published value-origin assumptions | Independent fixed type contracts and already captured operands |
| Class heritage evaluation | Actual selected base, arguments, prototype and decorator dependencies in the defined order | Previously created unrelated class identities |
| Module import or namespace read | Producer-side value-origin evidence and availability | Retained declaration contract identity, when established |
| Await, yield or escaped re-entry | Mutable facts that can change while control is elsewhere | The originating declaration/activation identity |
| Prototype mutation | Relevant ordinary prefixes and other existing property dependencies | Unrelated links or target-outgoing links not used by an acquired obligation |
| Array relocation/removal or ref rebinding | The appropriate location/liveness and origin facts | An unrelated nominal type's identity |

The global distinction matters: a Script `class V` lives in the declarative part of the global environment. `globalThis.V` is not automatically that binding. A Script-global function/`var` binding may instead be an object property. Indirect dynamic code can nevertheless resolve a global lexical `V` through the realm's global environment. Avoid both “indirect eval cannot reach lexical globals” and “every global property write changes every same-named lexical binding.”

### Real-engine costs and optimization limits

These are design assessments, not benchmarks of a production implementation.

Stable type denotations can be represented as internal constructor/type metadata and environment dependencies. They do not require adding public properties to instances, changing visible object shapes or wrapping every value. A JIT can place a proven class identity in compiled code or a captured environment slot. Each evaluated local class still needs its own identity; sharing syntax does not justify cache reuse.

Mutable constructor and callable values remain ordinary dynamic inputs. A monomorphic site can guard the captured target and use the specialized path; polymorphic or megamorphic sites pay the ordinary corresponding costs. Binding-cell versions or dependency invalidation can withdraw a speculative assumption. A deoptimization must preserve the already evaluated callee, receiver, arguments and effects. These guards do not justify Early Errors.

Prototype dependencies are separate. A changed prototype can invalidate property-access assumptions and requires the proposal's applicable admission-obligation processing. The exact mechanism can share engine dependency infrastructure, but its worst-case mutation cost depends on affected dependents and chain lengths. Do not promise one constant-time check per assignment or zero cost for all typed code. Weak bookkeeping must not retain admitted objects solely because a prototype remains alive.

Source-ordered evidence can improve startup and runtime by avoiding needless global invalidation. It also costs analysis time and metadata. A bounded prescribed proof system keeps mandatory diagnostics portable; a better optimizer may remove more runtime work without changing accepted source or error phase. Proven immutable bindings help, but do not make their referents immutable and do not establish reference liveness.

Rust is an important design influence for stable type identity and explicit storage distinctions. Its borrow checker and lack of JavaScript's runtime eval/prototype replacement mean it does not establish the safety of the proposed guards. V8's documented shape and IC mechanisms support the feasibility discussion, not a claim that this proposal has been benchmarked on V8.

## Implementation plan by repository

### Milestone 1 Correct the problem statement and preserve regressions

**Design:** retain the strictness explanation, revised fidelity requirement and separated questions from this plan. Document checked-sloppy code explicitly.

**Engine:** add focused regression fixtures for the original eighteen examples, explicit strict controls, replacement assignments, initializer counts, unused/unrelated eval and later writes. Record pre-body rejection separately from runtime failure. Preserve observed failures initially as evidence rather than converting them into assertions that the language requires those results.

**Spec:** audit the interaction of `sec-typed-strict-mode`, `sec-checked-code`, `sec-type-errors` and compile-time evaluation. Add an example explaining that `: any` activates strictness but does not authorize pre-executing an ordinary initializer.

Exit condition: the scope issue and the origin-evidence issue cannot be confused by the regression harness. No new eval-scope rule is needed.

### Milestone 2 Repair incomplete negative nominal judgments

**Engine:** make missing heritage/origin explicit at the relevant checker boundary. Audit `stableClassOrigin`, `heritageOriginOf`, `instanceTypeOf`, nominal relation consumers and receiving-check elision. Preserve known own contracts and evaluated identities. Do not fix this by removing the eval screen and assuming every syntactic base is stable.

**Spec:** define when origin/ancestry evidence is sufficient for a negative assignability judgment and when value admission remains dynamic. Keep formation errors distinct. Ensure incomplete metadata cannot contradict the existing class-family rule that unresolved identity is not incompatibility.

**Design:** add passing examples with harmless eval and a truly incompatible control, explaining the phase difference.

Exit condition: example 1 and unused-eval variants run; known unrelated-class failures remain early; changed actual bases still fail at the correct boundary; prototype/storage obligations still apply. This milestone does not by itself adopt Q3's cross-script denotation change.

### Milestone 3 Specify and unify declaration type denotation

Implemented by the second increment; see the [declaration-denotation implementation record](class-denotation-implementation.md) for coverage and validation limits. The original observed cross-script outcome below is historical and is intentionally changed by this increment.

**Design:** adopt the Q3 rule explicitly, with the documented difference between `: V`, `type V`, bare `V` and reflection of an actual constructor. Include the proposed changed outcome for example 6 and ordinary-shadow counterexamples.

**Spec:** extend declaration-instance evidence to later Scripts and follow producer declarations through imports/re-exports. State successful-publication, TDZ, failure recovery and generic-activation rules. Keep aliases immutable and preserve existing type-computation restrictions.

**Engine:** retain the original class association independently of mutable value storage. Reuse the existing class-family and lexical-environment infrastructure where appropriate. Audit local classes, immutable class expressions, named/namespace imports, eval, recursive activations and suspended generic functions. Do not mutate a shared syntax record to cache the last environment's identity.

Exit condition: splitting equivalent declaration-and-rebinding scenarios across scripts no longer changes the type denotation of the same declaration. Value-position lookup remains ordinary JavaScript. A nearer ordinary binding still blocks fallback.

### Milestone 4 Improve source-ordered origin precision

The third increment implements the finite class-value transfer/invalidation pass. [Class constructor origins in execution order](class-origin-order.md) records its supported cases, intentional diagnostic changes and conservative limits. It does not replace every independent whole-source screen in the checker; those consumers remain part of milestone 5.

**Engine:** use binding identity and execution-point facts instead of whole-source name sets for the affected proofs. Transfer effects in heritage and ordinary expression order. Cover the constructed class operand as well as its base, including replacement aliases, default parameter environments, callbacks and captured writes.

**Spec:** state the finite mandatory transfer/invalidation cases. Treat unknown effects conservatively without making precision depend on whether the operation was spelled `eval`, `Function` or a callback. Keep deeper literal-source analysis optional and semantics-preserving.

**Design:** explain what users can expect from stable aliases and when inherited completions are conditional. Include before/after-heritage and before/after-callee-capture examples.

Exit condition: writes after the relevant observation do not erase an earlier valid relation; writes that can affect it withdraw only dependent evidence. The replacement-through-stable-name probe is accepted when its origin is established.

### Milestone 5 Audit runtime enforcement and optimization seams

The fourth increment implements the receiving-boundary portion of this audit. [Runtime enforcement after origin analysis](runtime-enforcement-audit.md) records the positive implementation/result proof, property and tagged-call repairs and rejected blanket contribution change; validation is in its [implementation record](runtime-enforcement-audit-implementation.md). The fifth increment adds [global mutation footprints at callable reads](global-call-origins.md), repairing the concrete global-alias false Early Error and related supported reflective/dynamic-source cases while preserving independent inferred contracts. Its [implementation record](global-call-origins-implementation.md) gives validation and bases. The sixth increment follows [stored and returned global-effect values](global-effect-escapes.md) through allocation/property-sensitive may-flow, with validation in its [implementation record](global-effect-escapes-implementation.md). The seventh increment handles [opaque cross-source values and operations](opaque-global-effects.md), including the three recorded source-boundary counterexamples and implicit-access ordering; its [implementation record](opaque-global-effects-implementation.md) records validation and limits. Milestone 5 remains open for other static consumers, broader implicit protocols and integrated coverage; a finite withdrawal screen is not a complete positive static-evidence model.

**Engine and spec:** review callee capture, contextual initializers, shallow callable admission, `new`, `super`, decorators, actual prototype paths and check elision. Preserve existing exact prototype obligations and reference checks. Inspect indirect/global mutations that source-text-only “no assignment” reasoning can miss. Test getters and argument-side effects to ensure no duplicate lookup or replay.

Exit condition: an untyped or erased path cannot use stale origin evidence to bypass a declared receiving boundary. A failed runtime guard falls back within the same execution, with the original operands and locations. No change to public type equality, reference permissions or conversion ordering is required.

### Milestone 6 Conformance and delivery

Run the focused fixtures first, then adjacent class/generic/module/eval, nominal-prototype, conversion and reference suites. Run larger suites once the integrated plan is implemented; do not repeatedly run the entire extension suite during each small step. Preserve exact commands, counts, timeouts and known baseline failures.

Before each implementation task, pull all affected repositories and record the new baselines. If the baselines change, re-run the relevant baseline probes. Deliver one squashed patch per changed repository relative to its recorded base, with a short validation record and any remaining failures. Keep design changes in `ecmascript-types`, normative text in `proposal-runtime-types`, and implementation/tests in the engine branch. The initial delivery of this plan was a documentation-only patch.

## Regression matrix and acceptance conditions

| Family | Required cases | What must be asserted |
| --- | --- | --- |
| Modes | Plain sloppy, explicit strict, annotation-triggered strict, checked-sloppy, nested owner/sibling, class and module | Actual scope behavior, strict errors and owning-unit classification |
| Eval forms | Direct, parenthesized direct, comma/indirect, optional call, shadowed/replaced eval, literal/nonliteral source, nested eval | Actual call kind, reachable scope, runtime inner-source error timing |
| Mutation order | Before/after heritage, before/after receiving boundary, unused closure, called closure, branch, loop, parameter default, static initializer | Only affected origins invalidated; appropriate static/runtime phase |
| Binding routes | Assignment, pattern, iteration write, ref alias, global object, global lexical, imported mutable producer | Lexical identity rather than spelling; all relevant write routes |
| Class identities | Repeated local evaluation, recursion, immutable expressions, stable aliases, generic families, defaults and specialization reuse | Distinct evaluated families remain distinct; already captured identities remain fixed |
| Sources | One script, later same-realm scripts, named/namespace import, re-export, module cycle, direct eval in escaped closure | Consistent declaration denotation, producer availability, no caller-generic leakage |
| Call and construction | Effectful callee/member getter, argument replacement, spreads, receiver use, constructor-returned object, decorator replacement | One evaluation in correct order; actual captured target; preserved failures |
| Type boundaries | Ordinary classes, reference classes, value classes, unions, `any`, metadata and exact generic family entries | Existing assignability/storage distinctions; unknown evidence never becomes a wildcard |
| Prototype obligations | Mutation before admission, invalid/valid relink after admission, multiple obligations, unrelated links, failed outer admission | Correct acquisition and mutation result; no blanket freezing or early release |
| References | Element removal, relocation, ref loop length pinning, reference rebinding, erased call, getter/proxy-backed location | Original location and independent liveness; no revival, copy or extra dereference |
| Recovery | Failed eval, failed class/alias initialization, successful declaration followed by throw, interleaved activations | No poisoned cache, no publication of failed work, retention of real earlier declarations |
| Optimization independence | Cold/warm execution, guard success/failure, repeated source checks and specialization reuse | Same acceptance, error phase, effects and values |

For each new negative test, assert the intended rule or failure boundary rather than accepting any exception. For effect-sensitive tests, record the order and count, not only the final value. For reference tests, demonstrate that the storage operation actually invalidated the location before asserting the access failure.

The design is ready for adoption when each question's chosen direction is accepted as a language rule and the cross-script type-denotation change is explicit. Implementation is complete only when the listed observations match the adopted target semantics and the adjacent safety invariants survive. These are completion gates, not unaddressed alternatives.

## Validation performed for this analysis

- **48 focused probes** on the recorded engine head, including all eighteen draft cases, explicit strict feature-off controls and adjacent identity/effect/prototype cases. They establish the observations above, not comprehensive conformance.
- **Three focused suites: 60 tests passed.** `enforcement/typed-strict-mode`, `narrowing/checked-code-eval`, and `declarations/alias-across-scripts`.
- **Five adjacent suites: 249 tests passed, 1 failed.** `conversions/nominal-prototype-boundaries`, `generics/checked-class-dependencies`, `modules/runtime-generic-scope`, `modules/eval-generic-parameters`, and `functions/lexical-mutations`.
- The existing failing test is **“foreign family rejection then recovery”** in `checked-class-dependencies.test.mts`. It expected an `A.<uint.<8>>` versus `A.<any>` assignability message but received a TypeError admitting `[Function accepts]` to the generic callable contract. This is a different reported failing boundary, not evidence that foreign-family recovery is correct. It remains a baseline issue to investigate before relying on that recovery gate.
- Dependencies and the pinned test262 harness were installed. The full build stopped in `build:dts` because the TypeScript Go compiler could not resolve `/proc/self/exe` in this environment. The engine bundle was subsequently built with Rollup using a 6144 MiB Node heap; the default heap had exhausted memory. Type-declaration generation therefore remains unverified. The build's generated tracked error-message changes were restored; there are no implementation edits in this delivery.
- No full extension or test262 run was attempted. The targeted results above are the extent of executable validation.

Focused-suite command form:

```sh
node node_modules/vitest/vitest.mjs run <listed-test-files> --maxWorkers=2
```

The probe runner used `Agent`, `ManagedRealm`, `EnsureCompletion`, `TypeDiagnosticOf` and the host `onNodeEvaluation` observer. It passed original source strings directly to `evaluateScriptSkipDebugger`; multi-script probes reused a realm. The disabled controls used an agent without the `runtime-types` feature.

## Disposition of the draft's original directions

| Draft direction | Disposition in this plan |
| --- | --- |
| A Current behavior | Reject the false-negative evidence rule; preserve behavior that already follows the explicit strictness and admission policies |
| B Nothing resolved by checking is reused | Replace with Q1's evaluation-fidelity rule. The alleged initializer-caching defect is not established by examples 10/12; retained type evidence is necessary |
| C Checked eval cannot introduce caller bindings | Do not add this rule. Annotated units already get the standard strict rule; checked-sloppy units deliberately retain their mode |
| D Class bindings immutable in checked code | Reject for Q4's semantic and ergonomic reasons; use existing immutable bindings where desired |
| E Keep proofs unless reassignment can precede `extends` | Retain its source-order insight, but expand the model to all relevant origins, actual captured constructors, prototypes, imports, activation identities and runtime obligations |
| Literal-eval proof | Optional precision only, with actual intrinsic identity and sound effect analysis; never a substitute for unknown evidence and runtime enforcement |
| Accept uncertain cases and check runtime identity | Necessary part of Q5/Q7, but insufficient unless target contracts, effect order, prototype obligations and reference liveness are preserved |

The additional choice is the explicit declaration-contract model of Q3. The most important implementation repair can proceed without conflating it with that semantic extension, but the recommended final language model includes both.

## Sources and limits of the comparisons

Repository links below are pinned to the inspected commits. Section identifiers refer to the corresponding `spec.emu` source.

| Source | Relevance |
| --- | --- |
| [Design README][design-readme] | Feature surface, strictness, checkedness, class/reference distinctions and runtime types |
| [Type Objects][design-types] | First-class denotations, type operator, structural versus nominal identity and type computation |
| [References and borrowing][design-refs] | Storage identity, relocation/removal, rebinding and independent liveness |
| [Checked class dependencies][design-class-deps] and [class expression bindings][design-class-expr] | Declaration/activation identity and completion without reading a replacement value |
| [Runtime generic scope][design-generic-scope] and [eval generic parameters][design-eval-generics] | Lexical parameter evidence, escaped activations, ordinary shadows and global dynamic code |
| [Expression effects][design-effects] | Source-position facts, captured values and no speculative ordinary execution |
| [Formal specification][spec-source] | `sec-typed-strict-mode`, `sec-checked-code`, `sec-type-errors`, `sec-compile-time-evaluability`, `sec-class-family-identity`, `sec-typed-classes`, `sec-elision-stability`, `sec-ordinary-nominal-prototype-obligations` |
| [Engine checker][engine-check] | Whole-source eval/assignment screens, stable class origins and inherited type construction |
| [Runtime declarations][engine-decls], [lexical initializer evaluation][engine-lexical], [call evaluation][engine-call] | Retained family identities and actual runtime evaluation paths |
| [Nominal prototype implementation][engine-prototypes] and [typed strict tests][engine-strict-tests] | Existing obligation enforcement and mode controls |

Primary external comparisons:

- [ECMAScript PerformEval](https://tc39.es/ecma262/multipage/global-object.html#sec-performeval) and [EvalDeclarationInstantiation](https://tc39.es/ecma262/multipage/global-object.html#sec-evaldeclarationinstantiation): direct versus global source environments, strict declaration isolation and ordinary lexical lookup. These explain the strict controls; they do not make eval free of effects.
- [ECMAScript ClassDefinitionEvaluation](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-runtime-semantics-classdefinitionevaluation) and [CreateDynamicFunction](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-createdynamicfunction): heritage is an evaluated expression; dynamically constructed functions receive their defined global environment. No proposed static cache can substitute for those value operations.
- [Rust namespaces](https://doc.rust-lang.org/reference/names/namespaces.html) and [name scopes](https://doc.rust-lang.org/reference/names/scopes.html): declaration ownership is useful prior art, but Rust explicitly separates namespaces. Adopting that entire model would be a new choice here.
- [Rust generic parameters](https://doc.rust-lang.org/reference/items/generics.html): nested function items do not implicitly inherit outer generic parameters. This proposal's capturing JavaScript closures and per-activation class families are materially different.
- [Rust operator expressions](https://doc.rust-lang.org/reference/expressions/operator-expr.html) and [pointer types](https://doc.rust-lang.org/reference/types/pointer.html): useful comparisons for places, values and borrows. Rust's ownership/lifetime enforcement must not be substituted for the proposal's runtime liveness rules, and its detailed expression order is not JavaScript's.
- [TypeScript declaration merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html): classes have type and value roles, while type aliases are type-only there. This is a tooling/namespace comparison, not evidence that erased TypeScript contracts enforce this proposal's runtime guarantees.
- [V8 fast properties](https://v8.dev/blog/fast-properties) and [super property access](https://v8.dev/blog/fast-super): demonstrate shape-based and inline-cache implementation techniques. The proposed use of internal identity/dependency metadata is an engineering inference, not a reported implementation or benchmark.

[design-readme]: https://github.com/sirisian/ecmascript-types/blob/5800b8e3373283a0cdb0d65ab40a61f2cbef6433/README.md
[design-modes]: https://github.com/sirisian/ecmascript-types/blob/5800b8e3373283a0cdb0d65ab40a61f2cbef6433/README.md#strict-mode
[design-types]: https://github.com/sirisian/ecmascript-types/blob/5800b8e3373283a0cdb0d65ab40a61f2cbef6433/typeobjects.md
[design-refs]: https://github.com/sirisian/ecmascript-types/blob/5800b8e3373283a0cdb0d65ab40a61f2cbef6433/references.md
[design-class-deps]: https://github.com/sirisian/ecmascript-types/blob/5800b8e3373283a0cdb0d65ab40a61f2cbef6433/checked-class-dependencies.md
[design-class-expr]: https://github.com/sirisian/ecmascript-types/blob/5800b8e3373283a0cdb0d65ab40a61f2cbef6433/class-expression-bindings.md
[design-generic-scope]: https://github.com/sirisian/ecmascript-types/blob/5800b8e3373283a0cdb0d65ab40a61f2cbef6433/runtime-generic-scope.md
[design-eval-generics]: https://github.com/sirisian/ecmascript-types/blob/5800b8e3373283a0cdb0d65ab40a61f2cbef6433/eval-generic-parameters.md
[design-effects]: https://github.com/sirisian/ecmascript-types/blob/5800b8e3373283a0cdb0d65ab40a61f2cbef6433/expression-effect-sequencing.md
[spec-source]: https://github.com/sirisian/proposal-runtime-types/blob/6534d50c8926cafb31a9c9f630821b32d734798b/spec.emu
[spec-prototypes]: https://github.com/sirisian/proposal-runtime-types/blob/6534d50c8926cafb31a9c9f630821b32d734798b/spec.emu#L4467
[engine-check]: https://github.com/sirisian/engine262-proposal-runtime-types/blob/2568d80348434a68b961a6fc477126a1e35b6209/src/type-system/check.mts
[engine-decls]: https://github.com/sirisian/engine262-proposal-runtime-types/blob/2568d80348434a68b961a6fc477126a1e35b6209/src/runtime-semantics/RuntimeTypesDeclarations.mts
[engine-lexical]: https://github.com/sirisian/engine262-proposal-runtime-types/blob/2568d80348434a68b961a6fc477126a1e35b6209/src/runtime-semantics/LexicalDeclaration.mts
[engine-call]: https://github.com/sirisian/engine262-proposal-runtime-types/blob/2568d80348434a68b961a6fc477126a1e35b6209/src/runtime-semantics/CallExpression.mts
[engine-prototypes]: https://github.com/sirisian/engine262-proposal-runtime-types/blob/2568d80348434a68b961a6fc477126a1e35b6209/src/type-system/nominal-prototype.mts
[engine-strict-tests]: https://github.com/sirisian/engine262-proposal-runtime-types/blob/2568d80348434a68b961a6fc477126a1e35b6209/test/engine262/runtime-types/enforcement/typed-strict-mode.test.mts
