# Checking generic expressions

An expression such as `I.<uint8>` applies a generic declaration even when its result is discarded. If `I<T: type extends string>` is the established declaration, the application is invalid before execution, just as an annotation naming `I.<uint8>` is invalid. An unused function or an explicitly unreachable branch does not excuse a statically established invalid application.

This applies to interfaces, aliases and established class constructors. Positional, named and spread arguments use the same parameter assignment, constraints, kinds and defaults as other applications. A supplied but unresolved argument occupies its slot; it is not an omitted argument and cannot select a fallback default. Keep dependent obligations until their determining inputs are available, including through enclosing generic specialization.

The expression's value type and the type it denotes remain distinct. Checking `I.<string>` does not make the expression an instance of `I<string>`. Checking a class application does not construct an instance, materialize its constructor, run a static initializer or install a new object shape.

## Required properties

1. Reject known invalid argument binding at the existing checking boundary.
2. Preserve the applied declaration's lexical context and the caller's argument context.
3. Preserve pending inputs, bypassed defaults and enclosing specialization dependencies.
4. Establish the actual generic origin; a spelling, an erased value or a mutable producer binding alone is insufficient.
5. Preserve ordinary execution, module availability, identity, reference permissions and reference liveness.

## Directions considered

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep expression admission entirely at runtime | Autocomplete can show the right parameters while an obviously wrong argument survives until the expression runs; unused code hides mistakes. | Resembles dynamic application, but loses Rust's use-site bounds and the constraint checking familiar from TypeScript. | Least checking work; still needs runtime specialization and admission. No shape benefit. | Conflicts with established early-error rules and makes equivalent argument lists position-dependent. | Rejected: moving a type to expression position must not suppress a known error. |
| Evaluate every application eagerly | Finds many failures early, but changes when static initializers, getters and calls happen. A wrong autocomplete guess can run user code. | Neither Rust's static generic use nor TypeScript's erased type checking justifies evaluating arbitrary JavaScript expressions. | Can eagerly allocate specializations and alter initialization, cache population and inline-cache histories. | Violates availability, effects and dynamic target identity. It can manufacture local inputs and confuse preparation with reference liveness. | Rejected regardless of implementation effort. |
| Check only directly written, closed applications and discard dependent checks | Improves immediate diagnostics but small refactorings through a generic function change when mistakes appear. | An incomplete analogy to Rust's bounds on generic uses; it omits the relation between an enclosing argument and a dependent use. | Cheap initial check, but duplicates work and loses useful reusable obligations. No shape change. | Misses failures once retained inputs close and can confuse a pending supplied argument with omission. | Rejected: closed direct examples are not the full rule. |
| Check established origins using shared argument rules; retain pending obligations | The same parameter names and restrictions support autocomplete and diagnostics in each position. Wrong guesses fail at the earliest prescribed boundary. | Closest to Rust's use-site bound checking and TypeScript's argument constraints, with the runtime differences below. | Reuses parameter metadata and dependency records. Requires no wrappers, public fields or shape/inline-cache changes; successful runtime specialization keeps its existing cache. | Preserves caller/declaration contexts, argument assignment, default demand, module order and independent reference liveness. Dynamic values retain runtime admission. | **Recommended.** It satisfies all five properties without requiring eager execution. |

## Prior art and analogy limits

[Rust's bounds](https://doc.rust-lang.org/reference/trait-bounds.html#bound.satisfaction) restrict uses of generic items and provide assumptions inside generic bodies. This supports checking a known application's bound even when its result is not consumed. Rust is not a model for executing this proposal's first-class Type Objects or JavaScript getters during checking, and its trait/lifetime system is not a substitute for this proposal's reference-liveness rules.

[TypeScript's generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) use constraints and parameter defaults to guide argument checking. That is a useful tooling analogy. Its types are erased; this proposal retains runtime type identity, initialization, conversion and dynamic admission, so the phase and cache rules cannot simply be copied.

## Boundary and origin rules

Use source declarations and immutable origin information already established by ordinary checking. Parentheses and an eligible immutable alias preserve the original declaration. A class binding that can change, a class decorator that can replace it, an erased parameter, an arbitrary property read or a call result does not acquire that origin merely because its name or runtime value once matched it.

Purely static judgments retain their existing parsing/linking phase. Judgments requiring type computation retain the prescribed pre-body or specialization boundary, evaluability restrictions and budget. The checker does not execute a dependency body on demand, initialize a local binding to obtain an argument, or evaluate an unused default. A failed computation is not an unknown type or a successful cached application.

Runtime application still evaluates its base and performs the required admission. Static checks do not remove accessor/Proxy effects, class initialization, ordinary exceptions or dynamic reference checks. The selected design is independent of the size of the interpreter change.
