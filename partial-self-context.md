# Self types in partial class bodies

A bare reference to the name used by a partial class, in a type position inside its body, denotes the actual class application receiving that contribution. It has the same arguments as the receiving application, including value arguments. An alias or renamed import uses the name in the partial declaration.

```js
class E<T: type> {}
const Alias = E;
partial class Alias {
  self(): Alias { return this; }
  accept(value: Alias): Alias { return value; }
}
const value = new E.<uint8>();
value.accept(value);                 // valid
value.accept(new E.<string>());      // type error
```

This rule covers signatures and body annotations, instance and static methods, accessors, operators and functions created in the partial body. A conditional contribution retains its receiving application, including arguments matched by a wildcard or nested capture. Defaults do not replace those arguments. An explicit argument list continues to denote its explicit application.

The rule applies to the name written after `partial class`. It does not turn every alias of the family into a self name. Ordinary nearer bindings, including method type parameters, captures, local aliases and nested classes, shadow that name. Value-position lookup is unchanged: `Alias` as a value still reads its lexical binding. The partial adds no new value binding or instance storage.

## Decision

A solution must preserve actual family and application identity, ordinary shadowing, partial registration order, source-context isolation, failed-attempt cleanup, cached application identity and reference liveness. Checking summaries must remain separate from evaluated constructors.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Require explicit arguments in all partial bodies | Moving a member out of its primary requires annotation edits; completion exposes an avoidable distinction. | Rust provides `Self` in impls; C# partial parts contribute to one type. Neither requires this mismatch. | Simple resolution. | Can be sound but does not preserve the primary body's self shorthand. | Reject. |
| Apply defaults or erase arguments | Short spelling can silently accept the wrong application; completion conceals the lost arguments. | No close supporting analogy. | Cheap but weakens specialization assumptions. | Does not denote the receiving application. Liveness cannot repair a wrong type identity. | Reject. |
| Resolve the lexical constructor again whenever a method runs | Reassignment or a caller's context can change a previously declared signature. | Rust's impl target is fixed, not selected again by a call. | Repeated environment lookup and possible application work. | Loses the registration/application context; wrong family may satisfy or fail a contract. | Reject. |
| Store the most recent target globally against the partial's source node | Simple examples work, but repeated factory calls contaminate older methods and completion summaries. | Rust's `Self` belongs to an impl; source-node reuse across JavaScript activations has no such equivalence. | Small lookup cost, but unsound sharing prevents valid cache assumptions. | Conflates distinct evaluated families and applications. | Reject. |
| Retain the target application in the partial member's declaration context | A member moved between primary and partial keeps its self annotation. Completion shows the receiving type; an argument from another application fails at the ordinary boundary. | Rust's impl `Self` is the closest analogy. C# combines partial declarations, but neither supplies JavaScript's dynamic registration or capture rules. | Resolve once per applicable contribution and retain context on created callables. No new instance shape or property inline-cache state. | Preserves arguments, activation identity, lexical shadowing, failure cleanup and independent reference liveness. | Recommend, independently of implementation effort. |

The [Rust Reference](https://doc.rust-lang.org/reference/paths.html#self-1) defines `Self` in an implementation as the type being implemented. This proposal uses an existing class name rather than a reserved `Self`, so ordinary lexical shadowing remains significant. [C# partial types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/classes#1527-partial-type-declarations) provide a source-splitting analogy, not an execution-order rule.

## Context and failure boundaries

Checking resolves the target through the partial's lexical binding and describes its arguments under the contribution's pattern. Explicit references to primary parameters, such as `E.<T>`, remain open in member checking until the receiving application supplies them; a separately checked method must not try to evaluate that `T` before application. Checking must not publish an evaluated application merely because that source summary exists.

When a contribution is installed, its methods retain the actual evaluated family, ordered arguments and constructor. Functions created later by those methods retain the same self context. Reusing source in another activation, applying the family at another argument list or rebinding the original source name cannot change an earlier method's self type. A method's own generic parameters still have their ordinary scope; their spelling cannot replace the already retained application.

An application attempted before its required partial is registered still fails admission. A later successful registration permits a fresh attempt; failure leaves no successful application cache entry or self context for the next attempt. Computed keys execute at the ordinary installation point and are not re-executed to resolve self. Reference types retain their normal storage and liveness rules; this shorthand grants no extra borrowing permission.

See [generic member completion with partials](closed-partial-admission.md) for the distinction between interface lookup promises and actual declaration evidence.
