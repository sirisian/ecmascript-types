# Contexts of computed type results

A builder's selected type result retains the declaration context that supplies its contextual signature. The public Type Object still uses structural identity, including alpha-equivalence. Two builders returning equal function types can therefore supply different binder spellings to their consumers:

```js
// dep
function makeF(): type { type R = <U:type>(value:U)=>U; return R; }
function makeG(): type { type R = <V:type>(value:V)=>V; return R; }
export type F = makeF();
export type G = makeG();

// main (the namespace-import equivalent has the same behavior)
import {F, G} from "dep";
class C {
  f: F = function(value:U):U { return value; };
  g: G = function(value:V):V { return value; };
}
const c = new C();
if (c.f.<number>(7) !== 7 || c.g.<string>("ok") !== "ok") throw new Error("value");
if (Reflect.typeOf(c.f) !== Reflect.typeOf(c.g)) throw new Error("identity");
```

The [preparation repair](prepared-alias-contexts.md#computed-result-context-before-alias-publication) made the first signature available at the right checking phase, but the second builder's return still lost `V` to the canonical Type Object's `U`. Retaining context only when the exported alias is published is too late. This increment closes that remaining named/namespace pair and the binding-initializer equivalents.

## Decision and rejected alternatives

A valid solution must retain the actually selected declaration and captured inputs, alpha-equivalent public identity, ordinary reads and their failures, return admission, module checking order, resource accounting, and reference permissions/liveness. An unselected, discarded, failed or incomplete computation cannot supply context. Context discovery must not replay user code.

The earlier matrix remains in [the preparation analysis](prepared-alias-contexts.md#computed-result-context-before-alias-publication). The following completes the transport and storage decision rather than dropping the rejected directions.

| Direction | Ergonomics | Prior art including Rust | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Recover context from the canonical Type Object, a name cache, or a search for an equal result | Warm-up order can choose the accepted spelling; autocomplete can suggest a name that fails in the consumer. | Rust resolves generic declarations in scope, not by reverse lookup from equal values. | Cheap canonical lookup loses information; broader searches need inadequate identity guards. | Cannot distinguish selected declarations or activations. | Reject all variants. |
| Give signatures with different binder names different public identities | A harmless rename changes reflection equality. | Rust item identity is not a basis for making these structural function signatures nominal. | More interned identities and type guards; potentially greater IC diversity. | Violates the existing alpha-equivalence commitment. | Reject. |
| Require explicit own binders for every computed alias | A consumer must know how its annotation was produced; moving an alias behind a helper breaks adoption. | Rust has no first-class runtime type-builder/eval equivalent that justifies this exception. | Small implementation cost, redundant source. | Weakens uniform contextual typing. | Reject as the rule; explicit binders remain supported. |
| Preserve context only for a syntactically direct `return Alias` | Adding parentheses, a branch, a helper or `.call` can change which names are accepted. | Rust's lexical resolution does not depend on this JavaScript call spelling; ECMAScript forwarding calls invoke the selected target. | Local recognition is cheap but misses equivalent evaluation paths and optimization opportunities. | Does not preserve the selected result through existing call semantics. | Reject. |
| Carry alias provenance on every ordinary copy, property, array element and argument | Copies would preserve spelling, but users could no longer reason from a Type Object's public equality alone about hidden provenance. | Rust preserves declared types through storage; it does not attach JavaScript alias-binder provenance to first-class runtime Type Objects. The analogy does not settle this boundary. | Requires per-slot flow metadata, store invalidation and propagation through arbitrary containers; public attachment could disturb shapes and ICs. | Conflicts with the existing rule that an unrelated stored value does not acquire alias provenance. Reference and alias authority would need a separate redesign. | Reject for this design, independently of implementation effort. |
| Erase the producing expression's context whenever it stores a value or constructs an instance | `return (slot = R)` or a constructor returning `R` loses names that `return R` supplies, even though the returned Type Object is identical. | ECMAScript distinguishes the assignment expression's result and an explicitly returned object from a later slot read or the allocated instance. Rust has neither this assignment-result value nor this runtime type-builder mechanism. | Slightly less result tracking; no slot metadata is saved by erasing only the independent result. | Conflates the selected result with storage provenance, making forwarding depend on surrounding syntax. | Reject; the produced result and stored slot have separate boundaries. |
| Retain context on the selected successful computation result through value-preserving expressions, returns and calls; keep ordinary storage boundaries | A builder and its forwarding helpers supply the same signature. Autocomplete can use the selected declaration; copying only its public value has the existing ordinary-value meaning. | Rust supports declaration-based scope as a limited analogy. ECMAScript supplies actual completion selection and forwarding semantics. Neither language supplies this proposal's contextual binder adoption rule. | Private result/SSA metadata can survive call inlining and memoization; no public wrapper, object-layout change or new namespace IC guard is required. | Preserves declaration/capture identity separately from equality, with ordinary read, admission, failure and liveness checks intact. | **Recommend and implement** for the coverage below. |

These performance assessments describe feasible engine strategies, not interpreter timing measurements. The recommendation does not depend on which direction is easiest to implement.

## Result and storage boundaries

| Boundary | Context supplied to the selected result |
| --- | --- |
| A type expression | Its resolved Type Record before public interning, retaining binder declarations and captured inputs. |
| A read of an initialized alias, including named imports and actual namespace exports | Its published alias context, only after the ordinary read succeeds. A nearer unrelated binding supplies no outer alias context. |
| Parentheses, a selected conditional or short-circuit operand, the final comma operand | The selected operand's context, when its value is preserved. Discarded operands supply none. |
| Successful return and call, including helpers, specializations, bound calls and `call`/`apply`/`Reflect.apply` | The selected invocation result's context after ordinary return admission. An actual permitted getter call can also produce a result. No getter is executed merely to discover provenance. |
| Simple/destructuring assignment, selected logical-assignment result | The produced value's context after the required store succeeds. The stored slot still gains none. A skipped right operand supplies none. |
| A constructor returning a Type Object, including bound/reflective construction | The explicitly returned result's context. A newly allocated instance cannot inherit context from an ignored or replaced return. |
| A successful overriding `finally` return | Its own context. A throw or failed return admission cannot publish an abandoned return's context. |
| Ordinary variable, data-property, array, argument/parameter or caught-exception storage | No alias provenance. A later read cannot reconstruct it from public Type Object equality. Explicit own binders remain available. |
| Successful computed-type consumption and alias publication | The result's retained context; otherwise the existing Type Object rules. Failed or incomplete preparation publishes none. |

For example, `function forward():type { return makeG(); }` preserves the selected builder result. `function identity(value:type):type { return value; }` does not recover alias provenance from its ordinary parameter. Likewise, `const copy = R; return copy;` returns the ordinary Type Object; `type Copy = R; return Copy;` follows alias publication. Returning `(copy = R)` directly preserves the producing expression's context after the store succeeds; a subsequent `return copy` still reads an ordinary stored value. This distinction already exists between alias bindings and unrelated stored values; this change defines how a computation reaches that boundary.

The engine uses private metadata on evaluation completions. It neither adds a property to Type Objects nor caches a context on a shared function or syntax node. General result-flow helpers also visit ordinary bindings and property descriptors, so context transfer explicitly excludes those destinations. Reflective built-in call frames forward this type-result context without forwarding ordinary admitted-value contracts or reference rights. No additional type interning request is needed just to transfer context.

## Specification and reuse

`EvaluateBuilderCall` retains the selected context along with the successful Type Object, including in the evaluation memo. A cache hit returns both. A reusable result must have the same determining declaration and captured inputs; equality of the public result alone is insufficient. An in-flight placeholder supplies no completed context. Fixpoint completion must complete corresponding context edges consistently while preserving their declaration identities, and failure cannot publish an incomplete result.

`EvaluateToTypeObject` returns a normal completion containing the Type Object (or `empty` on the already specified failure path). A successful completion carries the auxiliary context into the containing type judgment. This makes the formal transport boundary explicit; the language value, evaluability rules, diagnostics and metering remain unchanged. Deferred inputs do not acquire a completed context before their ordinary binding point.

The implementation consumes context before unwrapping the completion. This matters in engine262 because its completion macros can replace a named completion variable with its value. Forwarding a completion and inspecting a public Type Object are different operations even when both eventually produce that same object.

## Coverage and remaining work

The portable `computed-result-contexts.json` corpus has 96 cases. It covers named and namespace imports, binding and class-field consumers, direct/type-expression returns, parentheses, comma/conditional/logical/nullish selection, `finally`, ordinary/arrow/generic helpers, alias chains, reverse/warm order, re-exports, eval consumers, imported builder reads, reflective/bound and optional calls, actual getters, direct/bound/reflective constructors, simple/destructuring/logical assignment results, failed stores, discarded results, ordinary-copy barriers with explicit-binder controls, and interleaved captured number/string contexts. The successful alpha-equivalent cases also check public reflection equality. Invalid consumers still fail before their module body.

Three additional engine tests check failed overridden-return publication and subsequent module recovery, budget exhaustion after a discarded intermediate context and independent recovery, and ordinary assignment, construction and reflective values/throws with runtime types disabled. The focused adjacent suite exercises preparation, namespace/imported alias dependencies, admitted result contracts, logical contribution provenance, tuple construction, evaluability, accounting, exhaustion, early errors, composite stores, assignment construction, logical store completion and reflective construction contracts.

This closes the preserved computed-result cases; it does not certify every recursive-builder memo/fixpoint family or arbitrary runtime storage. The subsequent [generic-parameter evidence repair](eval-generic-parameters.md) closes the written outer parameter in direct-eval annotations and preserves newly exposed runtime scope discrepancies. The imported-constructor guard/effect audit remains open with its exact reproducers and recommendation in [prepared-alias-contexts.md](prepared-alias-contexts.md). Broader identity, inference, phase/recovery, resource and conformance gates remain open.

Sources: [Rust generic parameter scopes](https://doc.rust-lang.org/reference/items/generics.html#generic-parameter-scopes), [Rust assignment expressions](https://doc.rust-lang.org/reference/expressions/operator-expr.html#assignment-expressions), [ECMAScript Function.prototype.call](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-function.prototype.call), [Reflect.apply](https://tc39.es/ecma262/multipage/reflection.html#sec-reflect.apply), [assignment evaluation](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-assignment-operators-runtime-semantics-evaluation), and [ECMAScript function construction](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ecmascript-function-objects-construct-argumentslist-newtarget). The declaration and call analogies support the boundaries above; the contextual adoption rule is this proposal's own design.
