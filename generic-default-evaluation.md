# Recursive generic defaults

Defaults supply omitted arguments at their prescribed application. Describing or publishing a generic declaration is not such an application. The same distinction applies to classes, functions, interfaces and aliases. Explicit or inferred bindings bypass their defaults. An annotation in an uncalled function can still require a closed application before program evaluation.

```js
type Unbounded<T: type = Unbounded.<> > = T;
type Explicit = Unbounded.<string>; // string; the default is not needed
type Finite<T: type = Finite.<string>> = T;
type Defaulted = Finite.<>;          // string
// type Required = Unbounded.<>;    // checked evaluation-limit failure
```

A default's presence and a completed description of its result are different facts. A pending default is neither an absent default nor a completed `any`. Re-entering one default's source node does not prove a semantic cycle: source, parameter identity, lexical environment, preceding arguments and captured substitutions all contribute to the query. Required evaluation uses the existing shared resource state, including recursive argument binding, and publishes no result after exhaustion. A finite result still undergoes constraint and consumer checks. Failure restores the enclosing environment and generic bindings; it establishes no initialization or reference-liveness fact.

## Decision

Required properties are demand-driven default evaluation, preservation of finite results, complete context for reusable results, checked limits before native failure, correct failure precedence, and restoration without invented types or permissions. The implementation effort is not a selection criterion.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance | Correctness and reference liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Increase the native stack or evaluation allowance | Small sources still fail differently across tools; completion may hang longer. | Rust's compiler query model does not use native capacity as the meaning of a type. | More duplicate work and stack memory; no object-shape or inline-cache benefit. | Neither bounds the unmetered route nor restores a valid result. | Reject. |
| Catch a native overflow and report a type error | Avoids a process crash, but a finite program may be rejected at an arbitrary host boundary. | Compiler failure containment is useful in Rust too, but is not a language-level query result. | Pays the cost until host failure; no useful layout effect. | Does not implement logical accounting or the required diagnostic cause. No liveness proof follows. | Reject as the semantic repair. |
| Reject every repeated declaration or default source | Easy diagnostic, but rejects finite uses with distinct arguments and unused defaults. | Rust queries distinguish the key and its arguments; Rust-specific cycle errors are not automatically appropriate for reified JavaScript computations. | Small bookkeeping cost, incomplete key; no shape benefit. | Source repetition alone cannot prove a cycle or required evaluation. | Reject. |
| Complete recursive queries with unknown/any | Autocomplete loses precise contracts and bad consumers can pass. | Rust's query results are not interchangeable with an in-progress computation. | Cheap early exit buys unsound reuse, not a production optimization. | Discharges an obligation without evidence and can erase constraints. It grants no valid reference permission. | Reject. |
| Evaluate every default while publishing its declaration | Tooling may appear to have all results, but an unused or explicitly bypassed default can fail a program. | Rust's declaration checking and const restrictions differ from this proposal's reified, application-time default computations. | Performs unnecessary evaluation and may materialize unused specializations; avoidable cache and memory pressure. | Violates the binding ladder and prescribed evaluation point. | Reject. |
| Suspend recursive description without publishing a result; retain each required application and finish it under ordinary metered evaluation | Finite defaults keep precise completion results; unused defaults remain usable and unbounded demanded defaults have checked diagnostics. A wrong consumer is still rejected. | Rust's query keys and in-progress/completed distinction are useful implementation analogies. ECMAScript supplies lexical environments; neither Rust's nested-item restrictions nor its cycle-error policy is imported. | Compiler-owned active markers can prevent redundant description. Completed caches retain all determining inputs. No new properties, user-object shapes or member inline-cache cases are needed. | Preserves context, demand, limits, pending obligations, failure restoration and independent liveness rules. A source-node marker is safe only as a conservative suspension, never as a completed cache key or cycle verdict. | **Recommend, independently of effort.** An iterative dependency/work queue is also valid if it preserves these same outcomes and accounting. |

The prototype uses an active marker for speculative description and the existing application-context records and metered evaluator for required defaults. This does not establish a universal stack-capacity bound for every static traversal or close the broader portable-budget audit. A production implementation can use an explicit work queue to bound host-stack use without changing source outcomes.

Primary comparisons: [Rust compiler query evaluation](https://rustc-dev-guide.rust-lang.org/queries/query-evaluation-model-in-detail.html), [Rust generic parameters](https://doc.rust-lang.org/reference/items/generics.html), and [ECMAScript Environment Records](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-environment-records).

## Captures through generic aliases

A returned signature retains dependencies reached through a generic alias's body, constraints and defaults. The alias's own parameters remain bound by its application; they are not captures of the surrounding function. Follow dependencies by declaration identity through intermediate aliases and helpers. Retaining a source dependency does not execute it or force an unused default.

```js
function identity(t: type): type { return t; }
function outer<T: type>(x: T) {
  type R<U: type = T> = identity(U);
  return function inner<V: type = R.<> >(y: V): V { return y; };
}
const fn = outer.<string>("x");
function use<T: type>() {
  const g = fn.<>;
  return g(1); // type error before evaluation, including when use is uncalled
}
```

The typed parameter makes `outer` participate in result inference. The published signature carries the captured String argument into `R`, completes the required default, and supplies the consumer's String parameter. A caller's `T`, an alias's own `U`, and a different activation of `outer` cannot replace that capture. Replacing `1` with `"ok"` admits the consumer. An explicit or inferred argument for `V` still bypasses its default. Completing a type supplies no initialization or reference-liveness evidence.

The required properties are complete lexical dependencies, separation of local binders and captures, activation-sensitive substitution, application-time demand, completed consumer checking and restoration after failure. None is traded away to reduce implementation work.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance | Correctness and reference liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Stop dependency collection at generic aliases | Extraction into an alias loses the closed signature and lets wrong calls pass. | Rust generic scope is declaration-based; its nested items cannot capture enclosing generics, so that restriction is not a model for these JavaScript closures. | Less checking by discarding an input; no shape or inline-cache benefit. | Misses required defaults and their consumers; supplies no valid liveness proof. | Reject. |
| Read alias bodies alone and resolve free names in the consumer | Alias-local parameters can become false captures; autocomplete changes with caller spelling. | Rust's scoped parameters and ECMAScript lexical environments distinguish these binders. | Small name maps permit incorrect reuse; no layout advantage. | Loses declaration and activation identity. | Reject. |
| Execute alias defaults while collecting captures | Makes even explicitly bypassed or unused defaults fail. | Rust's compiler query model separates inputs from computed results; its const restrictions differ from this proposal. | Extra evaluation and specializations, potentially user calls; unnecessary cache pressure. | Violates demand and initialization ordering. | Reject. |
| Retain declaration-identified transitive captures, then complete required defaults and reconsider their consumers | Alias extraction preserves completion and early diagnostics; distinct activations keep distinct contracts. | Rust query inputs and scoped substitution are useful analogies. ECMAScript environments supply the actual capture semantics. | Reuse compiler binding symbols and dependency summaries; memoize by determining inputs. No added user-object properties, shape transitions or member IC cases. | Preserves local binders, captures, demand, constraints and independent reference liveness. | **Choose independently of effort.** |

The prototype walks whole alias declarations during capture collection, using its existing free-reference analysis to exclude each alias's own parameters. The existing default evaluator and consumer recheck then use those captures. This repair does not change which function results participate in inference or complete the broader callable-compatibility audit.
