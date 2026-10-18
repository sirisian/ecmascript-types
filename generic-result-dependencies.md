# Generic local-class result dependencies

A factory's local generic class belongs to that factory activation. Returning a closure whose result is `P.<uint8>` must retain both that family and its argument bindings. A selected call can forward the result directly or place it in a named object without losing the independently established admission target. This extends [named object dependencies](composite-result-dependencies.md) and [Symbol-key dependencies](symbol-key-dependencies.md).

```js
function make(tag: string) {
  class P<T: type> {
    tag: string = tag;
    constructor(value: string) {}
  }
  return (value: any) => { const p: P.<uint8> = value; return p; };
}
const a = make("a"), b = make("b");
function choose(first: boolean) { if (first) return a("x"); return b("x"); }
const holder = { result: choose(false) };
holder.result.tag; // "b"
```

## Requirements and directions

Preserve declaration identity, the evaluated family activation and invariant argument bindings. Nested arguments have their own declaration dependencies. Own binders of a returned generic function remain unbound until that function is applied. Completed targets and written erasure stay authoritative. Completion cannot inspect a candidate instance to choose its target, replay a producer, execute a class specialization, initialize a binding, establish reference liveness, or mutate a shared symbolic publication.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Erase generic results to `any` | Adding a generic parameter removes members from autocomplete and postpones mistakes | Rust retains generic result types; TypeScript erasure does not implement this proposal's runtime checks | Cheap, but discards specialization information | Loses an established obligation | Rejected |
| Cache one constructor per declaration or parameter spelling | Independent factories or same-named binders interfere | Rust's static nominal identity does not justify merging JavaScript class evaluations | Compact cache with an unsound key | Confuses activations or binder identities | Rejected |
| Infer the target from the instance or its prototype | A bad replacement can choose its own acceptable type | Rust generic arguments are not chosen by inspecting a stored instance | Adds reads, effects and unstable property guards | Circular admission | Rejected |
| Complete only instance results and discard callable context | A factory can reject the closure before its first call; recursive forwarding can use the caller's family | Rust preserves callable result contracts, but its closure model differs | Saves metadata at the wrong boundary | Loses the result dependency carried inside the callable | Rejected |
| Always use the receiving environment | Straight-line cases work; recursion can select the wrong activation | Rust nested items cannot capture outer generic parameters; that restriction is not this proposal's model | Lexical lookup is cheap but insufficient alone | A selected callee result may belong to another activation | Rejected as the complete rule |
| Preserve independent lexical contexts and selected checked contributions, matching declaration and arguments at each result path | Generic refactoring preserves members and checks; a wrong replacement fails at the existing receiving boundary | Rust supports scoped generic binders and nominal distinctions. JavaScript's per-evaluation class identity requires additional captured context | Internal context/type-graph sharing and family/application guards can use existing IC machinery; no new public property or user-object shape transition | Keeps distinct activations, invariant arguments, authoritative targets, effects and liveness | Recommended independently of implementation effort |

Primary references: Rust [generic parameters](https://doc.rust-lang.org/reference/items/generics.html), Rust [structure types](https://doc.rust-lang.org/reference/types/struct.html), and TypeScript [generics](https://www.typescriptlang.org/docs/handbook/2/generics.html). Rust is an analogy for binder scope and nominality, not a specification of JavaScript's generative class families or effectful admission.

## Completion boundaries

A captured class family completes the closure's publication independently of returned values. At an executing return or inferred initialization, use a matching selected checked contribution first, then complete still-unresolved lexical dependencies in the applicable activation. The receiving contract retains its own arguments; completion of nested nominal arguments must precede their identity comparison. An already completed nominal is not rebound, even when its source declaration is visible in another activation.

A returned callable carries the family context of its corresponding result position. Its own type parameters remain declarations with their existing constraints; returning the function does not apply those parameters. In the bounded implementation, correspondence requires a single signature, matching own binder declarations, parameter types and optional/rest/reference modes. Overload correspondence is a separate coverage task. Ordinary callable compatibility remains required after completion.

A source class name in type position denotes the declaration's evaluated family even if assignment changes its ordinary value binding. The runtime lookup is guarded by lexical declaration resolution and availability of the family's published class type. It must not skip a nearer value alias or replace the existing forward-declaration path while self-referential fields are being defined. The available directions are:

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Follow the mutable value binding in type position | An assignment silently changes an annotation's meaning and makes autocomplete unreliable | Rust distinguishes type/value roles but has no assignable class constructors; TypeScript's static type role is not this dynamic rule | Requires repeated lookup or mutation guards | Contradicts declaration-based type evaluation and destabilizes the target | Rejected |
| Freeze both the class value binding and its type meaning | Existing ordinary assignments start failing | Rust's restrictions do not authorize a new JavaScript assignment restriction | Stable lookup, with new write rejection | Prevents the replacement by changing ordinary value semantics rather than preserving the type obligation | Rejected |
| Retain declaration-based type meaning and ordinary value lookup separately | Annotation members stay predictable while ordinary assignment keeps working | TypeScript's distinction is closer ergonomically, but its erased types supply no runtime enforcement. Rust supports the distinction only as a limited analogy | Stable internal family guards; no additional public property lookup | Matches the proposal's declaration-reading rule without initializing a binding or granting liveness | Recommended independently of implementation effort |

Choose the third direction. Its lookup must preserve a nearer alias and the class's existing self-reference and forward-declaration behavior.

The checker's compact numeric application arguments must be rebound to their declared value-parameter domains before runtime admission. Retain those typed literal records for source classes rather than collapsing them back to host numbers. A numeric literal in a type-parameter position retains its Number domain; a replacement with an equal numeric payload in a different typed domain cannot complete it as that replacement type. This does not change numeric builtin width/count representations or choose arguments from the value being checked.

## Scope and follow-up

The focused corpus covers direct and selected forwarding, named and Symbol-keyed object paths, closures and generic closures, separate factory activations, recursive factories, captured generic/value parameters, nested generic arguments, erasure, reflection and failed initialization. A retained object view continues to reject an incompatible replacement after an ordinary backing write. The plain object remains ordinary mutable storage.

This is a bounded repair within declaration-identity coverage. It does not close the six broader identity, convergence, phase/failure-state, budget/cache, specification-only and final-regression gates. Other composite shapes, overloaded returned callables, general mutable bindings and wrapper/resumable paths remain open. The handoff report retains any next desired-outcome failures separately from the passing corpus.
