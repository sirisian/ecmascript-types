# Eval, ordinary evaluation and nominal ancestry

Declaration-owned types, source-position value evidence, and runtime admission have separate responsibilities. The rules below preserve ordinary evaluation and reference liveness while allowing bounded static checking.

## Annotation-triggered strictness is already observable

```js
function make() { return 1; }
function f() {
  eval("var make = () => 2");
  const result: any = make();
  return result;
}
f(); // 1
```

The annotation makes the owning function strict. The eval's declaration stays inside the eval, just as it does if `f` instead contains an explicit `"use strict"` directive. Without either trigger, a sloppy direct eval can introduce a caller-local `var make`, and the corresponding unannotated function returns 2. The result is a scope difference, not evidence that checking cached a call or initializer.

Change the eval source to `make = () => 2` and the annotated function returns 2: assignment still reaches the outer mutable binding. This distinction also applies to constructors. An `any` annotation changes strictness, but it is neither a const declaration nor permission to reuse an earlier function or constructor value.

Checking and strictness have different triggers. A type alias alone can make a function checked without making it strict. The design adds no special eval scope mode and does not make all checked code strict.

## Missing ancestry is not incompatibility

```js
class V {}
class W extends V {}
const value: V = new W();
eval("1");
```

This program must run. The implementation previously discarded `W`'s base origin because of the eval anywhere in the source, then treated the absent base record as proof that `W` was unrelated to `V`. Even an unused function containing eval could cause the false Early Error. A later assignment to `V` produced the same problem.

The receiving judgment now distinguishes three outcomes:

| Evidence | Receiving-boundary consequence |
| --- | --- |
| Established compatibility | Apply the existing admission and check-elision rules; compatibility alone does not prove value stability |
| Established incompatibility | Retain the existing Early Error, subject to the existing conversion rules |
| Unavailable ancestry needed to decide this pair | Preserve the contracts and evaluate normally; retain runtime admission |

The third outcome belongs to the checker's evidence. It does not change the Boolean `IsAssignable` relation, public Type Objects, nominal identity, or the meaning of `any`. It cannot select an overload or license omission of a runtime check.

An absent base is not always unknown. A class with no heritage, or a known `extends null`, ends the chain. An unrelated such class still fails before body entry. So do independently known primitive mismatches and incompatible arguments of an already established application of the same generic declaration. Unknown ancestry cannot repair a malformed annotation or erase a class's known own field contract.

Source unions retain each alternative's obligations: a known invalid alternative is not hidden by an unresolved one. A target union can remain possible through an unresolved alternative, while a target intersection still requires every conjunct. Ordinary conversions retain their existing rules. In particular, missing ancestry is not a reason to choose a user conversion ahead of direct nominal admission.

## Runtime execution remains authoritative

```js
let calls = 0;
class V {}
eval("V = class { constructor() { calls++; } }");
class W extends V {}
const value: V = new W(); // runtime TypeError after calls becomes 1
```

Here the actual heritage expression captures the replacement constructor. The same-source annotation still denotes the original declaration's class contract under the existing rules. The ordinary `new W()` runs exactly once; its result does not satisfy that contract. Its effects are not rolled back when the receiving check fails.

Conversely, rebinding `V` after `class W extends V {}` does not change the base already captured by `W`. The resulting store reaches its existing runtime boundary when a static base record is unavailable. An established source-position proof also retains the earlier heritage observation.

Calls likewise keep their ordinary order: capture the callee, evaluate arguments, then call the captured value with its original receiver. Checking neither runs an unused ordinary initializer nor pre-parses an unexecuted eval source. Explicitly permitted type computations remain governed by their own rules.

Successful admission still acquires the existing nominal prototype obligations. A compatible intermediate prototype link remains allowed, while a mutation that removes the admitted `V` relationship is refused. Reference liveness remains independent: eval can remove an array element, and a later access through the old reference must fail even if an unrelated nominal receiving check succeeded.

## One declaration denotation across source texts

```js
// Script 1
class V {}
class W extends V {}
const Original = V;

// Script 2, same realm
V = class {};

// Script 3
const value: V = new W(); // succeeds against the original V
type Saved = V;          // retains the original declaration
```

The source class binding has two roles. A type position denotes the class introduced by that evaluated declaration. An ordinary expression reads the binding's current value. The distinction is the same within one source text and across later Scripts, direct/indirect eval, dynamically constructed functions and module imports.

| Spelling after replacement | Denotation or effect |
| --- | --- |
| `: V`, `type V`, `type Saved = V` | Original declaration's type |
| `V`, `new V()`, `class X extends V {}` | Current constructor value and ordinary evaluation |
| `Reflect.isAssignable(V, type V)` | Compare the actual replacement constructor's associated type with the original declaration's type |
| Imported `C` in `: C`, or `ns.V` in a type | Producer's original class declaration, following renames/re-exports |
| Imported `C` or `ns.V` as an ordinary expression | Producer binding's current value |

Only successful class evaluation and binding initialization publish an outer denotation. Decorator or static-initializer failure leaves that outer binding unavailable; a later statement throwing does not undo an earlier successful declaration. A class's inner context remains distinct. Separate local activations, recursion and suspended functions retain their own evaluated identities. Generic references use the original family for arguments, defaults and specialization reuse, including through a module namespace.

Runtime class decorators receive the original created class through `context.type` while the outer binding remains uninitialized. After decoration succeeds, that binding receives the replacement value, if any; its type denotation retains the original declared class contract. This also closes an implementation gap: a failed declaration must not leave a constructor association that could satisfy an escaped forward receiving dependency.

The same evaluation path applies to decorated named and default exports, including an anonymous default class. A partial declaration reopens the established class and does not initialize or republish its original declaration binding.

The publication audit required making the decorator commit point explicit. A valid direction must retain ordinary effects, avoid a usable failed outer declaration, preserve the declared contract and respect reference liveness.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Initialize and publish before decorators finish | Outer reads work during decoration, but a thrown decorator can leave a usable declaration | Rust supplies no runtime-decorator analogy; this diverges from JavaScript's class initialization boundary | Early roots and contracts require invalidation after failure | Violates successful-publication and failure-recovery requirements | Rejected |
| Initialize early, defer type publication, then revoke the binding on failure | The name is temporarily usable and later returns to TDZ; escaped observations become surprising | Neither Rust's declaration identity nor ordinary ECMAScript class evaluation motivates this transaction model | Requires additional state and invalidation; user effects cannot be rolled back | Can distinguish some states, but exposes a provisional outer binding and needs new observable rollback semantics | Rejected |
| Keep the outer binding uninitialized through decoration; initialize once and retain the declaration contract on success | Decorators use their supplied context; a failed declaration remains unavailable | Follows ECMAScript's outer class-binding initialization boundary. Rust supports fixed declaration identity but does not model these runtime callbacks | One successful binding publication; retained constructor is an internal GC root, with no user-instance shape changes | Preserves the contract, failure ordering, activation identities and liveness; effects already performed remain | Recommended independently of implementation effort |

Resolve the nearest ordinary binding first. A nearer `const V = type string` uses that binding's own type value. A nearer uninitialized binding still blocks the outer `V` and keeps the applicable TDZ/type-formation failure. Dynamic lookup likewise cannot bypass a nearer ordinary binding to recover an outer class. This does not introduce independent type and value namespaces or make arbitrary mutable reads valid type computations.

The retained constructor is part of the declaration's reachable contract, including for garbage collection. It is stored in internal binding metadata, with no added user-object property or instance shape transition. Real engines can lower a known type reference to a contract/environment slot; ordinary value loads and their inline caches still track the current value. This is an implementation strategy, not a performance benchmark or a license to omit receiving/prototype/reference checks.

The alternatives below explain the declaration-and-binding-instance model and why per-source snapshots and a second namespace are rejected.

## Implementation boundaries

The engine records unresolved heritage separately from a known end of ancestry on its internal nominal checker records. The receiving check consults that evidence only after ordinary assignability fails. Known contracts and generic arguments survive. Runtime admission, conversion execution, prototype enforcement and reference checks continue through their existing paths.

This is a checking change, not a new property on user instances or constructors. It introduces no instance shape transition or new runtime inline-cache policy. A production engine may optimize a retained boundary using its existing identity and prototype guards, but must preserve the actual operands, effects and failure phase. No performance improvement is claimed by this interpreter experiment.

The [origin-evidence audit](origin-evidence-audit.md) specifies the shared evidence requirements for consumers, deferred effects and receiving boundaries. Historical implementation records retain their original validation results.

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

R9 includes the existing strictness and typed-storage policies when a program opts in. It does not require annotation erasure to preserve behavior.

## Design decisions

Every question below compares the available directions using the same five criteria. “Other” records scope, portability or the reason an alternative loses. Cost of implementing a direction is described where relevant but does not determine the recommendation.

### What should annotation neutrality mean

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Full erasure neutrality | Simple migration story, but it contradicts what sized types and target-directed construction promise | TypeScript's erasable annotations are closer; Rust's typed execution is not an erasure model for JavaScript | Prevents important representation and contextual choices | Fails existing strictness, conversion, copying and overload commitments | Reject for this proposal; it would be a different type system |
| Let checking substitute whatever it resolves | Apparently precise completions can refer to a function that will not actually be called | Neither ECMAScript lookup nor Rust's resolved dispatch licenses executing a different ordinary callee by accident | Easy constant propagation can be unsound under mutation | Violates R1 and R5 | Reject |
| Ban all reuse of checked results | Avoids one suspected bug but repeats work and hides why a contract is stable | Rust and this proposal both retain resolved type information | Repeats contract formation and specialization; can repeat permitted initialization effects | Can violate once-only type/specialization behavior | Reject the blanket ban |
| Preserve ordinary evaluation subject to specified mode and typed semantics | Explains real behavior; the editor must show that an annotation activates strictness | Rust supports typed semantic distinctions; ECMAScript supplies ordinary evaluation order | Retains static contracts, pure folding and legal specialization caches | Meets R1 while preserving the proposal's existing semantics | **Recommend**; ordinary execution follows these mode and type rules |

An annotation-induced overload selection is a specified typed operation. Silently calling a stale ordinary function is not. Test the distinction with explicit-mode controls and effect logs, not only annotation removal.

### Should eval or strictness get a new checked-code rule

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Preserve fully sloppy behavior even in annotated units | Fewer migration effects, but `with`, mapped arguments and injected bindings make static suggestions fragile | ECMAScript permits these behaviors; Rust has no corresponding ambient scope injection | Requires dynamic environment handling and additional contract machinery for mapped aliases | Reopens settled typed-scope and parameter-enforcement issues | Reject as the direction for this design |
| Require an explicit strict context before allowing annotations | A wrong migration guess becomes an actionable error rather than a changed result | Uses ECMAScript strict mode; Rust offers no directive analogy | Similar optimized execution once admitted | Coherent, but changes where currently valid typed source is legal | Reject in favor of the existing syntactic opt-in: directives are redundant in modules and awkward for isolated functions with non-simple parameters |
| Make every checked unit strict, or only give its eval strict-like declaration scope | Fewer categories, but a range, alias or match can unexpectedly change unrelated runtime behavior; a custom eval mode is harder to explain | Full strictness has ECMAScript precedent; a partial strict-eval mode does not. Rust supplies no runtime-eval precedent | Broader lexical certainty, but still no protection against existing-binding writes | Full strictness is possible; partial strictness leaves mapped arguments and other behaviors inconsistent | Reject: the extra semantic reach does not solve class replacement and exceeds the needed opt-in |
| Keep annotation-triggered strictness and ordinary eval semantics | Annotations are a documented mode opt-in; checked-sloppy code remains possible and must get honest dynamic completions | Directly uses ECMAScript's scope rules; Rust comparison is limited to stable lexical resolution | Typed strict scopes can use static slots; only genuinely dynamic scopes need dynamic lookup handling | Preserves existing commitments without treating strict eval as effect-free | **Recommend** |

The implicit mode switch has a real ergonomic cost: even `: any` can change `this`, arguments mapping and eval scope. An editor should show “strict through annotation” at the unit and explain the trigger when a diagnostic or quick fix depends on it. Hiding that cost is not a language-design solution.

### What does a class name denote in a type position

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Re-read the current class value at every check or store | One apparent name meaning, but an existing variable's type silently changes; completions can go stale between statements | Unlike Rust's fixed item types; JavaScript variable reads alone do not define a durable contract | Repeated lookup, polymorphic guards and potentially different layouts at one slot | Violates R2 and can violate R6 | Reject |
| Capture the current value separately at each annotation's execution | Each individual binding has a stable type, but identical source annotations can mean different classes within an activation | Runtime contract capture has analogies in dynamic languages; it is not Rust's type-name model | Requires dynamic contract instances and guards for layouts, signatures and specialization | Coherent only with a redesigned type-formation and forward-reference model | Reject as the default: arbitrary mutable reads would conflict with closed type computation and predictable declared signatures |
| Capture whatever is visible at each source-checking boundary, retaining the local/foreign asymmetry | Matches several observations, but splitting one script into two can change a name's type meaning | No useful Rust analogy; this is phase-dependent capture rather than item identity | Cheap local caches, costly and surprising invalidation at source boundaries | Gives stable individual snapshots but lacks one declaration rule across sources | Reject as the language model |
| Introduce independent type and value namespaces | Stable type completion while values remain mutable; same-spelled shadows can surprise users | Rust explicitly separates namespaces; TypeScript classes also contribute type and value entities | Efficient static type resolution | Can be coherent, but changes this proposal's ordinary shadowing and first-class-type lookup story | Reject a general namespace split; do not bypass a nearer ordinary binding to find an outer type |
| Resolve a source class's type through its lexical declaration and evaluated binding instance; resolve ordinary expressions normally | Type positions have stable class/member information; value-position completion reflects replacement uncertainty. The divergence is explicit and explainable | Uses Rust's declaration-identity principle without copying its namespaces or lack of runtime class evaluations; fits existing class-family clauses | Stable contract pointers and family keys; mutable value loads retain ordinary guards | Preserves R2, R5 and R6; extends declaration evidence consistently across source boundaries | **Recommend** |

This is contextual denotation, not a second search namespace. Resolve the nearest lexical binding first. If it is a source class declaration, type position denotes that declaration's class instance; value position reads its current constructor value. A nearer ordinary binding blocks an outer class or alias. That nearer binding is usable as a type only when it independently meets the existing type-expression rules.

A symbolic class reference may be prepared before class evaluation. It must complete to the class associated with that binding instance, without evaluating its constructor or enclosing body early. Completion cannot use the most recently evaluated class with the same source node, and cannot choose the target from the value being admitted.

For a previously evaluated Script-global class declaration, retain its declaration association just as module class metadata retains its origin. A later script must not silently switch `: V` to an unrelated replacement merely because the class's mutable value slot changed. An old `W` remains admissible against the original declaration's `V` when its ordinary receiving obligations succeed. `new V()` still constructs the replacement.

`type V` should use the same denotation as `: V`; bare `V` remains an ordinary value read. The distinction must be documented, including reflection of an actual constructor value. Existing immutable aliases provide clear stable names. For an intentionally mutable constructor choice, give the chosen replacement its own stable declaration or pass the Type Object through an already supported generic/contract formation boundary. This design does not invent a new runtime-dependent annotation syntax or make ambient mutable reads compile-time evaluable.

### Should class bindings become immutable

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Make all class bindings immutable | Very stable constructor suggestions; existing JavaScript replacement code breaks | Rust items are not assignable locals, but JavaScript classes are constructor values | Fewer binding guards; no protection against every object/prototype effect | Violates R9 for ordinary code | Reject |
| Make class bindings immutable when their declaration is checked | One annotation can disable replacement of an otherwise ordinary class. Imported legacy classes still differ | Similar in spirit to Rust item stability, but checkedness has no Rust analogue | Helps some local sites; does not eliminate guards for mutable producers, decorators or dynamic origins | Possible only if every write route respects declaration-owned immutability | Reject: removes useful value behavior while solving only part of the evidence problem |
| Forbid assignment only from checked callers | Looks permissive until untyped callers replace a supposedly fixed class | Neither Rust item identity nor ECMAScript storage invariants support caller-dependent constness | Still requires guards against untyped writes | Does not make the binding immutable or justify proofs | Reject |
| Keep mutable class declarations; use existing immutable class-expression bindings and aliases when stability is intended | Preserves JavaScript replacement; users can give a stable constructor an explicit `const` name | ECMAScript `const` supplies the direct analogy; Rust's fixed type identity informs the separate contract | Stable paths optimize; mutable paths use normal loads and dependency guards | Works with stable declaration denotation and runtime admission without changing reference permission | **Recommend** |

Alias immutability does not force class-binding immutability. A `type` declaration introduces a type-level definition whose value is its denotation. A class declaration also introduces an ordinary replaceable constructor binding. Their roles differ. No outcome should depend on whether the code attempting the write is checked; storage properties belong to the declaration.

Making class bindings immutable would reject every ordinary reassignment, including writes from dynamically constructed functions. Restricting only checked writers would leave the binding mutable and would not establish the assumed identity proof.

### What should happen when heritage or constructor evidence is missing

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Treat missing ancestry as no ancestry | Precise-looking but false ancestry errors; inherited members disappear | Rust can reject unknown types in its closed static setting, but this proposal has explicit dynamic admission | Cheap analysis; no legitimate optimization benefit from an incorrect negative | Violates R3 | Reject |
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

### How precise should mutation and eval analysis be

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

### When are runtime guards and checks sufficient

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Use a predicted runtime identity to justify a pre-body error | Most executions may match the editor's guess, but valid executions become impossible | Rust's static proof is not speculation; JIT guards protect executable paths | Cheap prediction cannot recover from code never starting | Violates R4 and R10 | Reject |
| Re-read the name or replay an expression after effects to validate the guess | A getter, receiver or callee can change; failures appear in the wrong order | ECMAScript evaluation order is decisive; Rust supplies no replay permission | Extra calls and reads can destroy IC locality and duplicate effects | Violates R1 and can violate R8 | Reject |
| Retain the actual evaluated operands and perform ordinary admission against the fixed target | Calls and constructions run once; errors occur at the established boundary | ECMAScript operand capture plus this proposal's runtime contracts; Rust has a different static enforcement model | Monomorphic sites can use identity/tag/shape guards; deoptimization handles failed assumptions | Meets the requirements if all independent obligations survive | **Recommend** |
| Deep-wrap every callable and constructor that crosses a typed boundary | Stronger contracts, but wrappers change identity, receiver behavior and stack traces unless separately specified | Contract systems use wrappers; this is not Rust dispatch and not the proposal's shallow callable admission | Allocation, wrapper dispatch and changed IC targets | Changes existing callable and overload semantics | Reject as a remedy for unstable names |

Guards are an implementation strategy for accepted programs. They do not create a source-language proof. The actual selected function retains its own parameter/result checks. A shallow callable contract does not guarantee that a replaced function enforces the former declaration's return type; the receiving boundary cannot be elided on that assumption.

### Should class membership become a permanent brand or remain tied to prototypes

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Make every class a permanent hidden brand independent of its prototype chain | Stable class suggestions, but objects can satisfy the type after losing the behavior users associate with it | Rust nominal identity is stable, but Rust has no mutable JavaScript prototype chain | Potentially cheap brand tests and specialized ICs | Changes the proposal's existing class/prototype membership model | Reject for this problem |
| Freeze all class-related prototypes as soon as names are checked | Easy predictions, but checking an unused annotation can restrict unrelated objects | No close Rust analogy; JavaScript freezing is an explicit object operation | May simplify guards but imposes broad observable effects | Exceeds existing bounded admission obligations and checking phases | Reject |
| Test once and forget, or silently weaken existing contracts after mutation | Simple implementation; completions promise members that later disappear | Neither Rust's fixed types nor durable runtime contracts support silent weakening | Cheap admission, unsafe later elision | Violates R2 and R7 in the supported exact domain | Reject |
| Preserve current exact prototype obligations and independently handle domains they do not cover | Compatible prototype changes remain possible; invalidating changes fail at their defined operation | ECMAScript determines prototype effects; Rust only motivates preserving established guarantees | Internal dependency maintenance on mutations; no public per-instance field is required | Retains the bounded semantics and R7/R8 | **Recommend** |

This design does not claim a general solution for all generic, exotic, composite or union membership. Where existing rules lack durable proof, retain their required runtime checks and prohibit extra elision on that basis. A future expansion of prototype obligations needs its own semantic decision; the immediate name-stability repair must not silently expand or discard them.

### How should declaration evidence cross scripts, modules and generic activations

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Cache by spelling, source node or last evaluated constructor | Simple editor cache, but identical local declarations from different calls become confused | Rust paths identify declarations; runtime JavaScript class activations need an additional distinction | Cheap keys can reuse the wrong layout or specialization | Violates R5 and R6 | Reject |
| Treat an import as an immutable constructor because the importer cannot assign it | Attractive completions hide producer-side replacement | ECMAScript imports are live; Rust imports do not model mutable JavaScript export cells | Missing invalidation can make IC assumptions unsound | Confuses local write permission with producer stability | Reject |
| Infer the destination identity from the incoming value or re-run its producer | Almost anything can be made to match itself; wrong-family errors disappear | Rust infers under declared constraints; reflection of a value does not define its required destination | Extra reads, effects and repeated specialization | Violates R1, R2 and R6 | Reject |
| Retain declaration and environment-instance evidence, publish only at successful initialization, and resolve imports to producers | Captured closures and later sources keep the intended type; unresolved inputs stay visibly unresolved | Extends the declaration-identity principle used by Rust without importing its different nested-generic scope rules | Private slots/weakly held metadata; specialization caches key by family and arguments | Preserves scope, availability, failure recovery and liveness | **Recommend** |

The current specification already requires much of this for class families, immutable bindings and generic captures. Extend it to the class declaration's type denotation across later Scripts. Do not publish future declaration evidence just because checking encountered it. A failed class initialization contributes no completed class; a successful earlier initialization remains real after a later statement throws.

Direct eval receives its actual lexical generic environment. Indirect eval and `Function` use their global source environment, not the caller's generic stack. Escaped closures keep their originating activation, and a suspended activation must recover its own evidence when resumed. A nearer binding, including an uninitialized one, blocks a same-spelled outer fallback.

### How do name-stability proofs interact with references

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Treat stable element/class types as proof that a reference remains live | Autocomplete looks reassuring while a removed or relocated element can be accessed | Rust separates reference validity from the referent's type; its borrow checker is different from this proposal's liveness rules | Skips checks that can be necessary | Violates R8 | Reject |
| Let guard failure reconstruct or refresh a reference from the current name/index | Appears forgiving, but the program accesses a different location after rebinding or removal | No Rust precedent for silently replacing a borrow; ordinary JS names do not identify old array storage | Requires re-resolution and can replay getters or proxies | Changes location identity and revives invalid references | Reject |
| Carry the original location and existing liveness obligations independently of type-origin evidence | A wrong guess fails at the specified access; completions can distinguish type from availability | Rust provides the limited distinction between place/type and borrow validity; runtime enforcement remains this proposal's choice | Existing generation/relocation state and access checks; no new borrow counting | Preserves R8 without changing alias or storage contracts | **Recommend** |

If eval or a callback relocates storage between reference acquisition and parameter binding, the final binding/access checks still apply. If a reference binding is redirected, an alias follows the proposal's existing rebinding rules; a cached old origin cannot override them. Generic arguments are values, not newly borrowable storage. Neither the target contract nor a declaration metadata slot supplies a location for `ref`.

### What should tooling and diagnostics promise

| Direction | Ergonomics and autocomplete | Prior art including Rust | Real-engine performance | Correctness | Other and disposition |
| --- | --- | --- | --- | --- | --- |
| Always display the original class as the current constructor and all inherited members as certain | Convenient until a wrong guess calls another constructor or fails at runtime | Rust can give stronger static promises because its names and types have different rules | No engine advantage | Misrepresents the evidence and encourages unsafe elision assumptions | Reject |
| Erase every affected suggestion or report a hard error whenever proof is missing | Users lose known own members and valid code; workarounds spread `any` | Rust's errors arise from its required static discipline, not a template for this runtime system | Simple editor logic, unnecessary runtime dynamism | Confuses lack of evidence with invalidity | Reject |
| Distinguish declared contract, current-value origin, uncertain inherited information and implicit strictness | Users see which `V` a type position means and why a value call may differ; wrong guesses have explainable costs | Rust supports declaration-linked navigation; TypeScript demonstrates type/value presentations but has different runtime guarantees | Uses existing compiler metadata; optional richer analysis need not affect runtime | Aligns editor claims with declaration denotation, incomplete evidence, and phase-correct diagnostics | **Recommend** |

An early diagnostic should identify the established source and target and the failed rule. A dynamic failure should identify the fixed target contract and the actual failing boundary; use the existing diagnostic schema rather than inventing mandatory new public fields. Optional editor warnings about dynamic origin must not become an undocumented acceptance rule. Display names alone are insufficient when two evaluated classes share a source spelling.


## Sources and limits of the comparisons

Repository links below identify the historical revisions used for these design comparisons. Section identifiers refer to the corresponding `spec.emu` source.

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
