# Recursive interface bounds

An interface may constrain a type parameter with its own application:

```js
interface I<T: type extends I.<T>> { get(): T; }
class C implements I.<C> { get(): C { return this; } }
type Accepted = I.<C>;
```

This describes a finite recursive contract. Merely declaring `I` must terminate. Applying it still requires the argument to satisfy the bound: `I.<string>` and a class with an incompatible `get` are refused. The same requirement applies through names, spreads, defaults, imports and forwarded Type Objects. An uncalled function's closed invalid annotation remains an error.

## Required properties

Preserve the interface's actual declaration binding and captured context, the parameter identity, preceding bindings and the current candidate. Form the recursive description without treating it as a successful admission. Check the completed class's declared interface graph and member obligations before publishing the class. Distinguish a genuinely repeated structural description from changing arguments and executable type computations. Restore pending state on every completion. No step establishes reference liveness or initializes a source binding early.

## Directions considered

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Reject recursive interface bounds | Removes useful self-typed members from autocomplete; familiar declarations unexpectedly fail | Rejects the useful pattern supported by Rust bounds and Java F-bounds | Cheap syntactic rejection | Contradicts the proposal's existing F-bound rule | Rejected, regardless of implementation cost |
| Recursively validate every nested application immediately | No usable completion for even the declaration; a wrong guess can crash the process | Confuses formation of a recursive type with an infinite proof search | Unbounded stack/work; increasing the stack only postpones failure | Cannot implement the committed finite cases | Rejected |
| Treat any repeated source node as success, or replace it with `any` | Appears convenient but loses precise members and hides mistakes | Rust's solver documentation explicitly distinguishes provisional results from completed proofs | Cheap but produces unsafe reuse across bindings | Conflates different arguments, captures and declarations; can suppress real failures | Rejected |
| Stop every cycle with the evaluation budget | Predictable error instead of a host crash, but rejects ordinary finite self bounds | Resource limits are appropriate for divergent computations, not every recursive contract | Bounded but repeatedly expands a finite graph | Does not satisfy the accepted F-bound semantics | Rejected as the normal rule; retain limits for unresolved computation/growing applications |
| Form symbolic contracts, retain admission obligations and close only the same structural cycle | Autocomplete retains `get(): T`; invalid concrete uses still receive the ordinary diagnostic | Closest to the separation of generic assumptions and use-site satisfaction in Rust; Java's `Enum<E extends Enum<E>>` is a familiar finite pattern | Finite graph construction and local pending comparisons; no per-instance fields, shape changes or inline-cache changes | Preserves identities, argument checks, declared implementation edges and liveness rules | Recommended independently of the work required |

[The Rust Reference](https://doc.rust-lang.org/reference/trait-bounds.html) distinguishes bounds available while checking a generic item from satisfaction when using it. [Rust's compiler guide](https://rustc-dev-guide.rust-lang.org/solve/coinduction.html) distinguishes inductive and coinductive goals and warns about provisional results and cycle caching. This is an analogy for separating obligations, not a claim that Rust accepts every recursive trait proof. [Java's Enum](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Enum.html) supplies a direct F-bound example. Java's erased generics and Rust's trait implementation system do not model this proposal's runtime Type Objects, JavaScript computations or reference-liveness rules.

## Binding and completion

A permitted self bound needs its own candidate to describe the constraint. Resolve the explicit argument in the caller, or the demanded default in the declaration's preceding bindings. Supply that candidate provisionally to the bound's type-argument references, then perform ordinary admission. Provisional availability is not permission for computed self reads, forward reads or self-referential defaults; their existing rejection rules remain.

There are three possible placements: evaluate the bound before the candidate (rejected because the permitted self reference has no binding); make the candidate available to arbitrary self computations (rejected because it changes the read-order rule); or expose it only for the already permitted F-bound and publish it after admission (recommended). The third preserves expected member types, follows the limited Rust/Java analogy, uses a small transient binding frame, and changes neither object layout nor reference liveness. A failed candidate is not replaced with a default.

When describing a bound reaches the same unfinished interface constraint, a recursive edge can close only for the same evaluated interface identity, parameter, candidate and preceding bindings, without traversing executable type computation. The outer obligation remains to be checked. Different arguments continue normal evaluation under the existing budget. An executable computation, including one hidden behind an alias, cannot borrow the structural cycle shortcut. No unfinished result becomes a reusable completed admission.

A class implementing its own F-bound needs its actual interface graph before the bound is judged. Retain the checks on that forming class, resolve all of its written interface targets in the class's environment, then discharge the retained checks. Do not add an undeclared interface merely because a bound requests it. Other class/member constraints and incompatible interface arguments remain independent obligations. A failure abandons this publication and clears its pending state; already performed language effects are not rolled back.

The engine262 implementation uses per-checker description guards and delayed pure comparisons for class descriptions; runtime constraint entries are keyed by evaluated interface identity and ordered bindings. Runtime class-formation checks have stack lifetime. None is a completed-result cache. This increment does not close the broader inference-convergence or cache-accounting audits.
