# Dependent implements admission

Applying a class family must discharge the interface obligations retained by its declaration. `class E<T: type> implements T {}` is an open declaration. Applying it to `string` fails because the target is not an interface. Applying it to an interface requiring `get()` also fails unless the selected class body, its superclass chain or an applicable partial declaration actually declares a compatible member.

A claim is not declaration evidence. Members made available to type lookup by an interface do not become class declarations. In particular, an absent optional member of an inherited interface cannot satisfy a required member of a later target.

## Decision

A valid direction must preserve interface identity, argument admission, selected-case and captured contexts, required-member presence, property capability, operator and call signatures, ordinary reference liveness, failure cleanup and successful application caching. It must not invoke constructors, getters or methods to discover declarations.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Prohibit dependent targets | Disallows useful families and removes their declared member information | Rust requires a known trait path; that is not a reason to remove this proposal's explicit runtime type parameters | Simple implementation | Contradicts retained specialization obligations | Reject |
| Trust the nominal claim or its augmented lookup shape | Completion can promise missing members; mistakes surface at arbitrary uses | Rust requires non-default trait items to be implemented; a claim alone is insufficient | Cheap but provides invalid optimization facts | Circular proof; loses presence and capability checks | Reject |
| Check at first instance or member use | The same completed application can appear valid until used | Rust validates implementations before using their established trait relation; exact timing differs here | Repeated instance checks and extra states on hot paths | Too late for successful application publication and caching | Reject |
| Construct an instance and inspect its values | Validation executes user code and may fail for unavailable constructor inputs | Rust checks declared items; it does not construct sample values to validate an impl | Allocations, getters and shape-dependent reads; unreliable inline-cache facts | Violates effect ordering and cannot establish all instances or reference liveness | Reject |
| Verify instantiated declaration contracts before publication | Completion follows real declarations; wrong applications fail at their admission point | Rust's required trait items are the closest analogy, with this proposal's runtime binding and structural membership as important differences | Compile declaration summaries once; verify per specialization and cache success; no program-object shape changes or new property inline-cache state | Separates member evidence from nominal claims; preserves identity, capture, capability and liveness rules | Recommend independently of effort |

Choose the final direction. The [Rust Reference on trait implementations](https://doc.rust-lang.org/reference/items/implementations.html#trait-implementations) requires all non-default associated items and permits generic implementation parameters. Rust does not allow a type parameter to stand for an arbitrary trait in the same way as this proposal's runtime interface target, so its grammar and checking phase are not a direct model.

TypeScript's [implements-clause guidance](https://www.typescriptlang.org/docs/handbook/2/classes.html#implements-clauses) likewise treats the clause as a check: it does not create an absent optional property or change a method's inferred type. This supports separating declaration evidence from a claim. Its erased type system does not supply a model for this proposal's runtime specialization, nominal interface identities or reference liveness.

## Admission and recursive contracts

Resolve targets and bind their arguments in source order, rejecting established non-interface targets. Then verify member obligations in source order using the completed local target graph. This makes target resolution failures precede member failures. The local graph may support recursive member-type relations, but never proves the existence of a member or its write or call capability. Publish the application and insert a successful cache entry only after all obligations succeed. On failure, restore contexts and discard the incomplete application; this does not undo user-visible effects already performed by ordinary class definition evaluation.

| Direction for recursive member relations | Ergonomics | Prior art | Performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Admit only interfaces already verified to the left | Equivalent interface lists can reject valid mutual recursion when reordered | Rust's trait obligations are not restricted to previously written impl items | Small implementation but creates ordering-dependent workarounds | Incomplete local graph rejects valid recursive contracts | Reject |
| Publish the whole nominal claim before checking members | Recursive relations work, but failed applications can escape as established types | Rust does not publish an invalid implementation | Fast lookup with invalid cache state | Confuses local assumptions with successful publication | Reject |
| Resolve the local target graph, then prove each declaration obligation before publishing | Recursive contracts work and errors have a specified phase order | Rust's obligation reasoning is an analogy, not a runtime publication protocol | Temporary compiler metadata; only success is reusable | Recursive assumptions can relate member types without supplying missing members; reference checks remain independent | Recommend |

Use the selected class body's own captured generic context, and each inherited or partial declaration's own context. Complete nominal dependencies with actual binding identities. A currently verified self type has an actual identity and a temporary target graph even though it has no completed cache record yet. That local evidence must not leak into a separately evaluated activation of the same declaration.

## Member judgments

Check required presence, optionality and write capability separately from member value types. Getter-only and readonly members cannot satisfy writable requirements. Mutable storage remains invariant and readonly storage permits the existing covariance. Index signatures constrain applicable own enumerable fields, including inherited fields; they do not turn prototype methods into fields, and explicit named interface properties keep their own contracts.

A method implementation uses the ordinary function-signature relation, including generic binder identity up to renaming, reference parameters, accepted argument lists and declared or published inferred results. An unannotated result retains the existing gradual callable rules. A writable field containing a function remains storage and is invariant; declaring a method does not make arbitrary function storage covariant.

| Member comparison direction | Ergonomics | Prior art | Performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Exact function type identity for every method | Rejects compatible implementations and treats an absent annotation differently from ordinary callable admission | Rust method signatures must satisfy their trait; its exact type rules are not this proposal's callable relation | Cheap but duplicates an incompatible rule | Ignores published result obligations and existing callable compatibility | Reject |
| Callable covariance for all function-valued properties | Allows writes through an interface that the storage contract cannot accept | Rust's methods and mutable storage are also distinct concepts; it has no direct structural-field analogue here | No useful performance advantage | Violates writable storage invariance and may break reference-bearing signatures | Reject |
| Ordinary callable relation for methods; property rules for storage | Matches method calls and keeps assignment expectations intact | Uses the trait-method analogy only for declared behavior | Reuses callable and property relations; no instance work | Keeps call capabilities, published results, reference parameters and storage invariance | Recommend |

Operators have no ordinary property slot. Verify their actual declarations and signatures, including inherited and applicable partial operators. For callable interfaces, reject construction that is known non-callable; keep genuinely unresolved construction capability at the existing typed boundary. None of these checks executes a constructor or a member body.

## Evaluated member keys

Declaration evidence must use the actual String or Symbol key produced by ordinary class definition evaluation. Record that result once, including for applicable partial member declarations, before final application admission. These are declaration records, not values read from an instance.

| Direction | Ergonomics | Prior art | Performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep symbolic checking keys | A correctly declared symbol member appears missing | Rust's fixed associated-item names do not model computed JavaScript keys | Cheap but loses identity | Distinct symbolic and evaluated Symbols disagree | Reject |
| Re-evaluate key expressions during verification | A method can change name or produce a second side effect | Rust has no comparable computed member-name evaluation | Repeated user code and allocations | Violates single evaluation and mutable-state ordering | Reject |
| Inspect a constructed instance | Validation needs constructor inputs and can invoke accessors | Not how Rust validates trait items | Instance allocation and property-cache work | Violates the declaration-only judgment | Reject |
| Reuse evaluated declaration records | Computed names work with their ordinary effects exactly once | Rust supports the declaration-evidence principle; ECMAScript determines the dynamic key behavior | Store the evaluated key with existing declaration metadata; no instance or inline-cache changes | Preserves Symbol identity, capability and independent liveness | Recommend |

Applicable partial member evaluation occurs while the application constructor is in progress, before final interface admission. It uses the existing specialization context and runs once per attempted application; a failed attempt may retain its ordinary effects, and a successful cache hit does not repeat it.
