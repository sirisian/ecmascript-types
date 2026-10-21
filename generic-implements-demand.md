# Generic implements demand

A generic class can describe an interface implementation before its parameters have arguments. Publishing that family must not evaluate a target that depends on those unbound parameters. The target remains a specialization obligation; it is not an already resolved interface contract.

```js
// main
import { E } from "dep";
export interface Has<T: type> { get(): T; }
const value = new E.<string>();

// dep
import { Has } from "main";
export class E<T: type> implements Has.<T> {
  get(): T { throw new Error("example method"); }
}
```

The dependency can publish `E` without reading the unavailable `Has`. When `main` later applies `E`, its interface declaration is available and the application resolves `Has.<string>` in `E`'s source context. No module runs early. By contrast, `E<T> implements Has.<string>` has a closed target and retains the required pre-body availability check, even when nobody applies `E`.

## Requirements and decision

The rule must preserve declaration checking that is already decidable, per-target dependency, lexical parameter identity, injected self types, actual interface binding identity, selected-case captures, ordinary module order, failed-result handling and independent reference liveness. An editor should retain the declared generic member signatures without presenting an unbound contract as a completed application.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Evaluate targets with placeholder parameters when publishing the family | Valid unused families fail on placeholder constraints or unavailable imports; completion advertises a declaration that cannot be published | Rust generic implementations describe parameterized obligations, not an application to a placeholder runtime type | Adds useless binding, constraint and cache work at family creation | Reads explicitly deferred inputs; placeholder success cannot establish a concrete interface identity | Reject |
| Read the interface head now and defer only its arguments | A module cycle fails even though no application requested its target | Rust's compile-time trait item availability does not model JavaScript live-binding initialization | Avoids some argument work but retains an unnecessary binding dependency | Splits one explicitly deferred target evaluation and violates its availability boundary | Reject |
| Defer every target on any generic class | Simple rule hides errors in closed targets and makes unrelated parameters affect failure timing | Rust permits generic implementations with closed trait references; that fact supplies no reason to delay every JavaScript judgment | Cheap family creation, with unnecessary work shifted to applications | Violates closed declaration obligations, including mixed clauses | Reject |
| Publish symbolic or shape-only targets as completed contracts | Completion appears precise while runtime nominal membership can conflate unrelated declarations | Rust trait identity is not replaced by a matching shape; structural object membership is a different relation here | Cheap summaries, but unsound identity-based caches and optimization facts | Loses actual interface activations and can discharge obligations falsely; no reference permissions follow | Reject |
| Resolve only on first instance construction or membership test | Merely naming a specialization or reflecting on it can expose an incomplete contract; failure depends on which API runs first | Unlike a generic implementation's established relation to its implementing type in Rust; no close analogy for runtime Type Objects | Adds lazy state and checks on instance or membership paths | Too late for specialization publication and cached type identity | Reject |
| Defer only targets depending on the family's own unbound inputs, then resolve in the selected specialization's captured context | Generic members remain discoverable; unused open targets stay inert; closed targets and eventual applications retain precise failures | Rust's generic implementations are a useful parameterized-obligation analogy. ECMAScript module execution supplies the separate initialization constraint | Dependency classification can be compiler metadata. Existing family/application caches and captured contexts suffice; no program-object shape transition or property inline-cache state is needed | Preserves actual declaration and argument identity, selected-case captures, closed checks, context restoration and independent liveness | Recommend independently of effort |

Choose the final direction. Dependency follows lexical declarations, not matching names. A nested callable's own generic binder does not make the enclosing target depend on the class's same-spelled parameter. An enclosing function's already bound parameter is distinct from a nested class's own unbound parameter. The bare self type in a generic class implicitly includes that class's parameters and is therefore dependent.

Rust [generic implementations](https://doc.rust-lang.org/reference/items/implementations.html#generic-implementations) can use type and const parameters in the implementing type, trait and bounds. This supports reasoning about a parameterized relation; it does not prescribe this proposal's runtime Type Objects, structural interface membership or JavaScript module cycles. ECMAScript [module execution](https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#sec-moduleexecution) retains its ordinary ordering. The proposal's own demand and declaration-identity rules determine the recommendation.

## Resolution and publication

On family publication, classify each target without evaluating its head, arguments or helper computations. Closed targets use ordinary resolution. Dependent targets remain in the declaration for later application; no placeholder or symbolic contract is stored as their successful result.

Checking still retains the declaration's parameterized contracts. Completing an imported class's evaluated identity must not overwrite those contracts with the family's partial runtime list. An actual completed specialization can supply its resolved list; a family cannot certify that an omitted dependent target is absent. Complete interface identities from successful declaration-publication metadata in the class's source context, including renamed imports, without reading current mutable binding values. An ambiguous activation remains unresolved. A real engine can index these declaration dependencies; no program-object shape change is required. Preserving a checked declaration obligation does not claim it has already resolved, execute specialization code during identity completion, or waive eventual admission.

At application, use the selected class body's captured lexical environment, realm, private environment and generic frame, including specialization-case captures. Resolve the actual interface targets with bound arguments and the ordinary generic binder. An injected self argument must retain the current family identity and the in-progress interface graph; rebuilding it from the Parse Node and argument list alone is insufficient. Restore the caller context on success or failure. Cache only a successfully published specialization.

Evaluating `type E.<string>` must obtain the completed application's contract too. Copying the family's unresolved interface list gives a false negative for `Reflect.isAssignable(type E.<string>, Has.<string>)`, and looking up a constructor without its associated record leaves the answer dependent on which spelling ran first.

| Direction for a type-position application | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Copy the family's unresolved contract | A type loses its declared interface relation; completion and reflection disagree | Rust's instantiated trait relation is not an empty relation | Cheap but produces unusable optimization facts | Drops a required specialization obligation | Reject |
| Substitute a symbolic declaration contract | Looks precise until selected cases or captured interfaces differ | Rust substitution retains trait identity; textual replacement is not equivalent | Avoids construction but requires correct identity and case machinery | A symbolic summary cannot establish an actual interface activation or discharge its binding obligations | Reject |
| Add a separate contract-only specialization path | Could offer the same relation without constructing a class value | Rust can separate code generation from trait reasoning, but has no corresponding runtime class static initialization | Potential allocation savings; requires coordinated case selection, recursion, failure and cache accounting | Valid only if observationally equivalent to ordinary specialization, including required initialization and identity | Reject a distinct semantic path; equivalent internal optimization remains permitted |
| Reuse ordinary specialization and its associated completed Type Record | Type and value spellings agree before and after construction; the first evaluated application initializes the specialization once | Rust supplies the consistency analogy, not JavaScript initialization timing | Shares existing family/application caches; no instance shape changes or property inline-cache invalidation are introduced | Preserves selected contracts, evaluated identities, failure handling and independent liveness | Recommend independently of effort |

Use the shared application path. A successfully completed specialization supplies its associated Type Record. An exact recursive application can retain the constructor already under construction. Repeated type and value spellings reuse the result and do not repeat static initialization. This concerns evaluation of an application; it does not authorize source preparation to execute ordinary enclosing bodies or make an unavailable dependency available.

This does not waive eventual target-kind or member checks. A parameter that later denotes a non-interface, or an interface whose members are not implemented, must still be refused at its prescribed boundary. The current implementation audit separately records gaps in that general dependent-target admission path and in direct reflection on specialized class constructors. Passing availability and identity tests does not close those gaps.

The focused implementation covers deferred family publication, injected-self openness, callable-binder shadowing, captured case inputs, actual self identity and completed type-position applications. Tests exercise cycles, renaming, re-exports, namespace access, type/value parameters, defaults, selected cases, distinct lexical activations, mixed clauses, shared static initialization and failed-specialization recovery. [Cyclic imported interface availability](cyclic-interface-availability.md) covers the complementary closed-target rule.
